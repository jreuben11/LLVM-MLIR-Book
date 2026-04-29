# Chapter 19 — Instructions I — Arithmetic and Memory

*Part IV — LLVM IR*

The LLVM IR instruction set is not a thin wrapper around machine instructions — it is a semantic contract between the front-end and the optimizer. Every instruction carries not just an opcode and operands but a set of optional flags that assert mathematical facts about the computation. When those facts hold, the optimizer is free to transform the instruction in ways that would be unsound in general; when they do not hold, the result is *poison*, a value that propagates silently until it is observed, at which point behavior is undefined.

This chapter is a systematic treatment of the instructions most directly involved in arithmetic and memory access. It covers the wrap and exactness flags (`nsw`, `nuw`, `exact`, `disjoint`) that govern integer arithmetic; the fast-math flags that relax IEEE 754 semantics for floating-point; the `getelementptr` instruction whose deceptively simple job is pure pointer arithmetic; `load`, `store`, and their `volatile` and atomic variants; `alloca` and the `mem2reg` transformation that converts stack slots into SSA values; the `freeze` instruction that domesticates poison; and the `ptrtoint`/`inttoptr` pair whose use has deep consequences for alias analysis and pointer provenance. Control-flow instructions (`br`, `switch`, `phi`) and aggregate operations (`extractvalue`, `insertvalue`) are reserved for Chapter 20.

Throughout the chapter, code examples are verified against LLVM 22.1.3 (`clang --version` → `Ubuntu clang version 22.1.3`). All IR uses opaque `ptr` — typed pointers were removed in LLVM 15.

---

## 19.1 Wrap Flags: `nsw`, `nuw`, `exact`, and `disjoint`

### 19.1.1 The Poison Semantics Contract

Before examining individual flags, it is worth stating the underlying contract clearly: each flag is a *claim* that the front-end makes about a computed value. If the claim is correct, the flag enables the optimizer to apply transformations that would otherwise be unsound. If the claim is wrong — if the flag is present but the asserted property does not hold at runtime — the result of the instruction is *poison*. Poison is not an integer value. It is a symbolic marker that propagates through any computation that uses it, eventually triggering undefined behavior if it is used in an observable way (as a branch condition, as the address of a memory operation, or as a return value). The optimizer may assume poison never reaches an observable use; it is entitled to exploit that assumption to produce tighter code. The front-end bears full responsibility for the correctness of every flag it emits.

This design is covered in depth in [Chapter 171 — The Undef/Poison Story Formally](../part-24-verified-compilation/ch171-the-undef-poison-story-formally.md). For this chapter, treat it as the background axiom: *flags buy optimization power; incorrect flags buy undefined behavior*.

### 19.1.2 `nsw` — No Signed Wrap

The `nsw` flag is available on `add`, `sub`, `mul`, and `shl`. It asserts that the mathematical result, when interpreted as a signed integer of the operand width, does not overflow.

```llvm
; Clang emits these from 'int a + int b', 'int a - int b', etc.
%sum  = add  nsw i32 %a, %b
%diff = sub  nsw i32 %a, %b
%prod = mul  nsw i32 %a, %b
%shl  = shl  nsw i32 %a, %shift
```

*Why it exists.* In C and C++, signed integer overflow is undefined behavior (C11 §6.5 ¶5, C++23 [expr.arith]). Because Clang can assume the program never triggers signed overflow, it emits `nsw` on every signed arithmetic operation it generates. This gives the optimizer two key powers:

1. **Strength reduction.** `mul nsw i32 %x, 2` can be lowered to `shl nsw i32 %x, 1`. Without `nsw`, a multiply that wraps and a left shift that wraps are not equivalent for signed types at the boundary values.

2. **Range propagation.** If the optimizer knows `%a` is in `[0, 100]` and `add nsw` holds, it can infer `%sum` is in `[b_lo, b_hi + 100]` without worrying about the wrapping edge case. Without `nsw`, any two-value range must be conservatively widened to the full i32 range whenever the upper bounds might add to overflow.

The relationship to C's UB is direct: every signed integer operation in well-formed C maps to an IR operation with `nsw`. An optimizer pass (e.g., IndVarSimplify) may add `nsw` to loop induction variables when it can prove through range analysis that the increment cannot overflow. Conversely, an instrumented build (ASan integer overflow, UBSan) strips `nsw` and inserts explicit overflow checks before the arithmetic, trading correctness guarantees for safety checks.

### 19.1.3 `nuw` — No Unsigned Wrap

The `nuw` flag mirrors `nsw` for unsigned arithmetic. It is valid on `add`, `sub`, `mul`, and `shl`, and asserts that no unsigned overflow (i.e., no wraparound modulo 2^N) occurs.

```llvm
%ptr_off = add nuw i64 %base_offset, %stride
%count   = mul nuw i32 %rows, %cols
```

Unsigned wrap is defined in C (it produces the mathematical result modulo 2^N), so the C front-end does not emit `nuw` routinely the way it emits `nsw`. However, Clang does emit `nuw` when it can statically prove no wrap occurs — most commonly in pointer arithmetic. A pointer offset computed as `(char *)p + n` where `n` is a non-negative value known to fit within the object generates `add nuw` for the index computation, because a valid in-bounds pointer offset cannot wrap an unsigned 64-bit integer on any real hardware. The `nuw` flag then permits the alias analysis to exclude impossible cross-page aliasing scenarios.

The optimizer also uses `nuw` for loop canonicalization: if a loop counter starts at zero and is incremented by a positive constant, induction variable simplification can tag the `add` with `nuw`, which then feeds into loop exit condition analysis and allows the backend to emit tighter loop structures.

### 19.1.4 `exact` — Exact Division and Exact Shift

The `exact` flag applies to `sdiv`, `udiv`, `lshr`, and `ashr`. It asserts that the division (or shift) is exact — that dividing the dividend by the divisor leaves no remainder.

```llvm
; Asserts x is divisible by 4 — enables x / 4 → x >> 2
%q = sdiv exact i32 %x, 4

; Asserts x has no bits below position 2 — enables lshr exact → mul strength reduction
%s = lshr exact i32 %x, 2

; Arithmetic right shift with no lost bits
%t = ashr exact i32 %x, 1
```

*Optimization unlock: `sdiv exact` → `ashr`.* An arithmetic right shift is faster than a signed division on almost every target, but `x >> 2` and `x / 4` are not equivalent for negative odd values (right shift rounds toward negative infinity; division rounds toward zero). With `sdiv exact`, the optimizer knows the quotient is exact, so both operations produce the same result, and it replaces the division with the shift.

*Optimization unlock: `lshr exact` → pointer multiply reconstruction.* A value computed by `lshr exact i32 %x, k` can be multiplied by `2^k` to recover `%x` losslessly. This is the inverse of a left shift. The optimizer can use this, for example, to rebuild a byte count from a word count when it knows the conversion is exact.

*When Clang emits `exact`.* Clang does not routinely emit `exact` on user-visible divisions because C does not guarantee them. However, some internal transformations (e.g., the induction variable substitution in LoopStrengthReduce) add `exact` when they can prove divisibility holds for all loop iterations.

### 19.1.5 `disjoint` — Non-Overlapping Bits on `or`

The `disjoint` flag on the `or` instruction, introduced in LLVM 16, asserts that the two operands have no bits in common: for any bit position, at most one operand has a `1` bit at that position. Formally, `(a & b) == 0`.

```llvm
; Combine high nibble and low nibble — guaranteed disjoint
%hi       = and i32 %x, -16         ; 0xFFFFFFF0
%lo       = and i32 %x, 15          ; 0x0000000F
%combined = or disjoint i32 %hi, %lo
```

*Why it matters.* When bits are disjoint, `or` is equivalent to `add` (no carry can propagate between bits that are not both set). This equivalence is used in two directions:

1. **`or disjoint` → `add` lowering.** On targets where `add` is cheaper than `or` in a particular context (rare but possible with SIMD and FMA chaining), the optimizer can substitute. More commonly, it allows expression reassociation that would not be legal for a general `or`.

2. **Constant folding and known-bits analysis.** If the optimizer knows `%hi` has only bits 4–31 set and `%lo` has only bits 0–3 set, it can immediately prove `disjoint` holds and propagate range information through the result without bit-level case splits.

*Clang emission.* Clang 16+ emits `or disjoint` when constructing tagged pointers, bit-field packing, and color component combining where the compiler can prove the fields do not overlap.

---

## 19.2 Floating-Point Fast-Math Flags

### 19.2.1 IEEE 754 and Its Costs

IEEE 754 guarantees that floating-point operations produce the correctly rounded result, that NaN propagates predictably, that the sign of zero is preserved, and that operations happen in exactly the order written. These guarantees are expensive: they prevent algebraic transformations such as reassociation (`(a + b) + c` ≠ `a + (b + c)` in general), folding additions of zero (`x + 0.0` ≠ `x` when `x` is `-0.0`), and reciprocal reuse (`x / y` and `x / z` cannot share `1/y` unless `y == z`). The fast-math flags selectively waive those guarantees.

### 19.2.2 The Seven Flags

Each fast-math flag is placed between the opcode and the type of the instruction. Multiple flags are space-separated. All verified against LLVM 22.1.3.

**`nnan` — No NaN Operands or Results**

```llvm
; Operands and result are guaranteed not to be NaN
%r = fadd nnan float %x, 1.0
```

With `nnan`, the optimizer may replace `%x == %x` with `true` (because NaN is the only value not equal to itself), replace `%x != %x` with `false`, and eliminate NaN checks on the result. Clang emits `nnan` when `-fno-honor-nans` or `-ffinite-math-only` is active.

**`ninf` — No Infinity**

```llvm
%r = fmul ninf float %x, %y
```

Infinities cannot appear as operands or results. This enables range-based simplifications: the optimizer can assume intermediate values are representable as finite numbers, which allows it to fold comparisons against `±∞` to constant false/true and to use compact range representations in value tracking.

**`nsz` — Negative Zero Treated as Positive Zero**

```llvm
; With nsz, x + 0.0 → x (because -0.0 + 0.0 = +0.0, not the same as -0.0)
%r = fadd nsz float %x, 0.0
```

Without `nsz`, `fadd float %x, 0.0` cannot be eliminated because if `%x` is `-0.0`, the IEEE result is `+0.0`, not `-0.0`. With `nsz`, the optimizer treats `+0.0` and `-0.0` as interchangeable, permitting the identity fold.

**`arcp` — Allow Reciprocal**

```llvm
; x / y → x * (1.0 / y) with arcp
%r = fdiv arcp float %x, %y
```

Division is typically five to ten times slower than multiplication on modern hardware. With `arcp`, the compiler may compute a reciprocal and multiply, trading a possible half-ulp accuracy difference for a substantial throughput improvement. When dividing multiple values by the same denominator (e.g., normalizing a vector), `arcp` allows the reciprocal to be computed once and reused.

**`contract` — Allow Fused Multiply-Add**

```llvm
; May fuse into a single FMA instruction
%prod = fmul contract float %a, %b
%sum  = fadd contract float %prod, %c
```

The `contract` flag permits the optimizer to merge adjacent multiply and add operations into a fused multiply-add (FMA), which avoids the intermediate rounding step and is computed as `a*b + c` in a single operation to full precision. This typically improves both accuracy and throughput. On x86 with AVX2, the backend emits `vfmadd213ss`; on AArch64, `fmadd`. The `contract` flag is narrower than `fast`: it does not permit reassociation or reciprocal substitution.

**`afn` — Approximate Functions**

```llvm
; sin/cos/sqrt may use hardware approximations or lookup tables
%r = call afn float @llvm.sin.f32(float %x)
```

With `afn`, the optimizer may substitute calls to transcendental functions with faster approximations that have lower precision. On x86, `rsqrtss` computes an approximate reciprocal square root with about 11-12 bits of mantissa precision; with `afn`, the compiler may use it instead of the full-precision `sqrtss` followed by a reciprocal.

**`reassoc` — Reassociation of Floating-Point Operations**

```llvm
%t = fadd reassoc float %a, %b
%r = fadd reassoc float %t, %c
```

IEEE 754 requires operations to be performed in the written order. `reassoc` permits the optimizer to reorder additions and multiplications freely, which is essential for auto-vectorization (the loop vectorizer sums partial results from multiple SIMD lanes) and for loop reductions. Note that `reassoc` alone does not imply `nnan`, `ninf`, or `nsz`; those must be listed separately if desired.

**`fast` — All Flags Combined**

```llvm
; Equivalent to nnan ninf nsz arcp contract afn reassoc
%r = fdiv fast float %x, %y
%s = fadd fast float %a, %b
```

The `fast` flag is syntactic shorthand for the conjunction of all seven individual flags. Clang emits `fast` when the function has `-ffast-math` (or its component flags) active.

### 19.2.3 How Clang Emits Fast-Math Flags

Clang translates command-line math options to fast-math flags at the instruction level:

| Clang flag | IR flags enabled |
|---|---|
| `-ffast-math` | All seven (`fast`) |
| `-fno-honor-nans` | `nnan` |
| `-fno-honor-infinities` | `ninf` |
| `-fno-signed-zeros` | `nsz` |
| `-freciprocal-math` | `arcp` |
| `-fno-trapping-math` | (permits `afn`; not a single IR flag) |
| `-ffp-contract=fast` | `contract` |
| `-ffinite-math-only` | `nnan` + `ninf` |
| `-funsafe-math-optimizations` | `nnan` + `ninf` + `nsz` + `arcp` + `reassoc` |

A per-function pragma or attribute can override the translation unit default:

```c
__attribute__((optimize("fast-math")))
float vector_dot(float *a, float *b, int n) { /* ... */ }
```

Clang applies the `fast` flag to every FP instruction in that function regardless of the command-line flags on the translation unit. This is the standard mechanism used in performance-critical libraries that need fast-math in isolated hot loops without poisoning the entire program.

Observed in the wild — Clang 22 output with `-ffast-math`:

```llvm
define dso_local nofpclass(nan inf) float @fast_div(
    float noundef nofpclass(nan inf) %0,
    float noundef nofpclass(nan inf) %1) {
  %3 = fdiv fast float %0, %1
  ret float %3
}
```

Note the `nofpclass(nan inf)` parameter and return attributes: these are function-level attestations that complement the per-instruction flags, allowing interprocedural analyses to exploit the same assumptions without requiring dataflow.

---

## 19.3 `getelementptr`: Address Computation Without Dereferencing

### 19.3.1 Purpose and Formal Semantics

`getelementptr` (GEP) computes a pointer to a sub-element of an aggregate type or advances a pointer by a typed stride. It does *not* read or write memory. It is pure arithmetic on addresses.

Formally, GEP computes:

```
result = base_address + offset_in_bytes
```

where `offset_in_bytes` is derived by walking the type tree according to the index sequence:

- Each index multiplied by the byte size of the type it traverses contributes to the offset.
- Struct indices select a field; the contribution is the byte offset of that field within the struct (including padding bytes).
- Array indices select an element; the contribution is `index * sizeof(element_type)`.

The type path is fully determined by the GEP's element type argument and the number of indices; the runtime values of the indices affect only the numeric offset, not the type of the result, which is always `ptr` in modern LLVM.

### 19.3.2 Syntax and Index Rules

```
%result = getelementptr [inbounds] ElemType, ptr %base, idx0, idx1, ...
```

*Element type.* `ElemType` is the type that `%base` nominally points to. It is used solely to determine how to scale indices. It does not mean `%base` has been declared as a pointer to `ElemType`; in opaque-pointer IR, all pointers are `ptr` and the element type is just metadata for the index calculation.

*First index.* The first index (`idx0`) advances `%base` as if it points to the beginning of an array of `ElemType`. `getelementptr i32, ptr %p, i64 1` advances `%p` by `sizeof(i32)` = 4 bytes, producing a pointer to the next `i32` in memory. This index may be any integer type and may be negative.

*Subsequent indices.* Each subsequent index descends one level into the aggregate type. For an array `[N x T]`, the next index selects an element of type `T`. For a struct `{T0, T1, T2}`, the next index must be an `i32` constant (struct layout is fixed at compile time and cannot vary dynamically).

*The first index does not "enter" the type.* This is a common source of confusion. To reach the first element of a struct without any array-level offset, the first index is `i32 0`, which means "advance by 0 * sizeof(struct) bytes"; the second index then selects the field.

### 19.3.3 Worked Examples

**Example 1: Struct field access.**

```llvm
; struct { i32 x; float y; } — access field y (index 1)
%field_ptr = getelementptr inbounds {i32, float}, ptr %s, i32 0, i32 1
; Byte offset: 0 * sizeof({i32,float}) + offsetof(field 1) = 4 bytes
; result ptr points to the float field
```

The `i32 0` first index means "stay at the zero-th element of an implicit array of structs" — it does not advance the pointer. The `i32 1` second index selects field 1 (the `float`), advancing by `offsetof({i32,float}, 1) = 4` bytes. Clang generates exactly this pattern for `s->y`.

**Example 2: Multi-dimensional array element.**

```llvm
; int arr[10]; — access arr[5]
%elem_ptr = getelementptr inbounds [10 x i32], ptr %arr, i64 0, i64 5
; Byte offset: 0 * sizeof([10 x i32]) + 5 * sizeof(i32) = 20 bytes
```

Again the first index is zero (no array-of-arrays level), and the second index selects element 5, contributing `5 * 4 = 20` bytes.

**Example 3: Pointer arithmetic.**

```llvm
; int *p; — compute &p[3]
%next = getelementptr inbounds i32, ptr %p, i64 3
; Byte offset: 3 * sizeof(i32) = 12 bytes
```

This is the one-index form. The element type `i32` is the stride. This is how Clang lowers `p + 3` or `&p[3]`.

**Example 4: Nested struct — field of a field.**

```llvm
; struct Outer { int a; struct { float x; double y; } inner; };
; Access inner.y: first advance to outer[0], then into field 1 (inner), then into field 1 (y)
%y_ptr = getelementptr inbounds {i32, {float, double}}, ptr %obj, i32 0, i32 1, i32 1
; Byte offset: 0 + offsetof(field 1) + offsetof(double in {float, double})
;            = 4 + 8 = 12 bytes (with natural alignment padding: 4 bytes i32, 4 bytes padding, 4 bytes float, 4 bytes padding, 8 bytes double)
```

The actual byte offsets depend on the target's data layout; GEP handles them automatically from the `datalayout` string.

### 19.3.4 The `inbounds` Flag

`inbounds` asserts that every step of the pointer walk stays within the bounds of the allocated object (or one-past-the-end, following C's rule for pointer comparisons). If `inbounds` is present and the address computation would go out of bounds, the result is poison.

The practical consequences:

1. **Alias analysis.** With `inbounds`, LLVM's pointer alias analysis (AA) can reason that a GEP result aliases only within the same allocation as the base pointer. Without `inbounds`, the pointer arithmetic might wrap around the address space or cross allocation boundaries, forcing AA to be conservative.

2. **Optimization.** The loop strength reduction and induction variable analysis passes prefer `inbounds` GEPs because they can establish that successive loop iterations access contiguous memory within a single allocation.

3. **What Clang emits.** Clang emits `inbounds` for all array subscript expressions and member access expressions. It does not emit `inbounds` for explicit pointer casts or pointer arithmetic on `void *` — those explicitly opt out of bounds guarantees.

The `inbounds` guarantee is stronger than it might appear: if `%base` itself is a GEP result with `inbounds`, the whole chain of GEP calculations is subject to the invariant, which means that the "base" allocation for alias analysis purposes is the original allocation, not just the most recent step.

### 19.3.5 GEP Does Not Dereference

This bears repeating because beginners frequently misread GEP as a load: **GEP never touches memory**. It computes an address. To read the value at that address, a `load` must follow. To write to it, a `store` must follow.

```llvm
; This does NOT read the float field — it computes its address
%field_ptr = getelementptr inbounds {i32, float}, ptr %s, i32 0, i32 1

; This reads the float value
%field_val = load float, ptr %field_ptr, align 4
```

Confusing GEP with a load is the single most common LLVM IR mistake. A GEP with a dead result is pure dead code (it has no observable side effects), while a `load` with a dead result is not removable in general (loads may have observable side effects via `volatile`, and they participate in the memory model).

---

## 19.4 `load` and `store`: Alignment, Volatile, and Atomics

### 19.4.1 Basic Syntax

```llvm
; Load: read a value of type T from address %p
%val = load T, ptr %p, align N

; Store: write %val to address %p
store T %val, ptr %p, align N
```

The `align N` specifier asserts that `%p` is aligned to at least `N` bytes (N must be a power of two). This is not a request — it is a *claim* the front-end makes to the optimizer. If the claim is wrong and `%p` is not N-byte aligned, the behavior is undefined. The optimizer may use the alignment to select wider SIMD loads, reorder loads across SIMD lane boundaries, or vectorize loops; a misaligned pointer invalidates all of those inferences.

If `align` is omitted, the default alignment is 1 byte (fully unaligned), which is safe but disables most alignment-based optimizations. Clang always emits an explicit `align` based on the type's ABI alignment.

### 19.4.2 Alignment and Undefined Behavior

Loading a value of type `i64` from a 4-byte-aligned address is undefined behavior in LLVM IR. On x86, the hardware tolerates it (with a performance penalty), but the optimizer may have already used the `align 8` claim to vectorize or reorder memory accesses, producing incorrect code. A common real-world bug is a C `struct` with a `__attribute__((packed))` annotation that forces misaligned fields, combined with a Clang code path that does not propagate the reduced alignment through pointer arithmetic — the field accesses inherit the struct's alignment rather than the field's actual alignment, potentially emitting `align 4` loads for `double` fields that are only 4-byte aligned.

### 19.4.3 `volatile` Loads and Stores

The `volatile` keyword prevents the optimizer from removing, duplicating, reordering, or coalescing the load or store.

```llvm
; Read a memory-mapped I/O register — must not be eliminated even if result unused
%status = load volatile i32, ptr %mmio_reg, align 4

; Write to a control register — must not be merged with adjacent writes
store volatile i32 %control, ptr %mmio_reg, align 4
```

**What `volatile` does not guarantee.** Despite what C programmers sometimes believe, `volatile` in LLVM IR does *not* imply any memory ordering. Two `volatile` accesses to different addresses may be reordered relative to each other. `volatile` is solely about preserving the existence and order of accesses *to the same address*, not about multi-threaded visibility. For synchronization, use atomic operations (§19.5) or memory barriers.

**Canonical uses.**

- Memory-mapped I/O: side-effecting register accesses that must occur exactly as written.
- Signal handler visibility: a `volatile sig_atomic_t` flag read in a signal handler context.
- `setjmp`/`longjmp` interactions: a variable modified between `setjmp` and `longjmp` that the compiler must not keep only in a register.

Clang maps C's `volatile` qualifier directly to LLVM's `volatile` flag on loads and stores. Note that C++ `volatile` is deprecated in some contexts in C++20 and its semantics do not change LLVM's IR-level behavior.

### 19.4.4 Load and Store Type Consistency

In opaque-pointer IR, there are no typed pointers; all pointers are `ptr`. The type annotation on the `load` and `store` instructions serves as an *assertion* about what type the caller believes is at that address, not a guarantee enforced by the type system. It is legal (and sometimes useful) to store an `i32` and load a `float` from the same address — this is the LLVM equivalent of a C `union`. However, the type reinterpretation is entirely defined by the LLVM memory model (no special float-int transmutation rules; it is just a bit pattern). Clang does not emit mismatched type loads/stores for normal C++ programs; they appear in hand-written IR or output from specific lowering passes.

---

## 19.5 Atomic `load` and `store`: A Preview

Full coverage of the LLVM memory model, fences, RMW operations, and their mapping to C++23 and Linux kernel memory semantics appears in [Chapter 27 — Coroutines and Atomics](../part-04-llvm-ir/ch27-coroutines-and-atomics.md). This section covers the syntax and the ordering vocabulary.

### 19.5.1 Syntax

```llvm
; Atomic store — seq_cst ordering
store atomic i32 %val, ptr %p seq_cst, align 4

; Atomic load — acquire ordering
%v = load atomic i32, ptr %p acquire, align 4
```

The `atomic` keyword follows `load`/`store`. The ordering specifier comes after the address (for stores) or after the type (for loads). The `align` must be at least the natural alignment of the type; for atomic operations it must also satisfy the platform's atomicity guarantee.

### 19.5.2 Ordering Levels

| Ordering | Meaning |
|---|---|
| `unordered` | Weakest: prevents torn reads/writes. No synchronization. Valid only for atomic ops. |
| `monotonic` | No reordering, but no synchronization with other threads. Maps to C++ `memory_order_relaxed`. |
| `acquire` | Load side of a synchronization pair. No memory accesses after this load may be reordered before it. |
| `release` | Store side of a synchronization pair. No memory accesses before this store may be reordered after it. |
| `acq_rel` | Both acquire and release. Used for read-modify-write operations. |
| `seq_cst` | Strongest: sequentially consistent total order across all `seq_cst` operations. |

`unordered` has no C++ equivalent and is primarily used for Java's non-volatile memory accesses and for safe tearing (read/write without a data race in the language-defined sense). `monotonic` maps to `std::memory_order_relaxed`. The acquire/release pair establishes the canonical synchronizes-with relationship: a store `release` to address `%p` followed (in time) by a load `acquire` from the same address creates a happens-before edge.

Atomic load/store require the `align` to match the operand's size for the operation to be genuinely atomic on most platforms: an atomic `i32` requires `align 4`, and an atomic `i64` requires `align 8`. Misalignment of an atomic operation is undefined behavior regardless of whether the hardware tolerates unaligned accesses.

---

## 19.6 `alloca`: Stack Allocation and the `mem2reg` Pattern

### 19.6.1 Syntax and Semantics

```llvm
; Allocate space for one i32 on the stack
%x = alloca i32, align 4

; Allocate space for N i32s (variable-length array)
%arr = alloca i32, i32 %n, align 4
```

`alloca` returns a `ptr` to freshly allocated stack memory. The allocation is live for the entire lifetime of the function (or until `llvm.lifetime.end` marks it dead). There is no corresponding `free`; the stack frame is reclaimed when the function returns.

*Entry block convention.* In practice, `alloca` instructions are valid anywhere in the function but are meaningful only in the entry block when the allocation count is constant. Clang places all scalar `alloca` instructions in the entry block (even for variables declared in inner scopes) to ensure that the stack frame is fixed-size and can be computed before the first instruction executes. Placing an `alloca` inside a loop with a constant size still compiles, but it may trigger a runtime check in debug builds and will confuse stack size analysis. A `alloca` with a variable size argument (`alloca i32, i32 %n`) implements variable-length arrays (C99 VLAs) and resizes the stack frame at runtime via `llvm.stacksave`/`llvm.stackrestore`.

### 19.6.2 The Clang Alloca-then-`mem2reg` Pattern

Clang does not construct SSA form directly. Instead, it follows the classical approach of:

1. Emitting every mutable local variable as an `alloca`.
2. Emitting every assignment as a `store` to the alloca.
3. Emitting every use as a `load` from the alloca.
4. Relying on the `mem2reg` pass (part of the `-O1` pass pipeline as `sroa,mem2reg`) to promote `alloca`/`load`/`store` triples to SSA φ-functions.

This is both simpler to implement in the front-end and provably correct: the resulting IR before `mem2reg` is valid LLVM IR, and `mem2reg` is a pure optimization (it does not change the semantics of well-typed programs).

*Before `mem2reg`:*

```llvm
define i32 @example() {
entry:
  %x = alloca i32, align 4
  store i32 42, ptr %x, align 4
  %val = load i32, ptr %x, align 4
  ret i32 %val
}
```

*After `mem2reg`:*

```llvm
define i32 @example() {
entry:
  ret i32 42
}
```

The `alloca` is entirely eliminated; the constant `42` is propagated directly to the `ret`. For a variable assigned in different branches:

```llvm
; C: int x; if (cond) x = 1; else x = 2; return x;
; Before mem2reg:
entry:
  %x = alloca i32, align 4
  br i1 %cond, label %then, label %else

then:
  store i32 1, ptr %x, align 4
  br label %merge

else:
  store i32 2, ptr %x, align 4
  br label %merge

merge:
  %val = load i32, ptr %x, align 4
  ret i32 %val

; After mem2reg:
entry:
  br i1 %cond, label %then, label %else

then:
  br label %merge

else:
  br label %merge

merge:
  %val = phi i32 [ 1, %then ], [ 2, %else ]
  ret i32 %val
```

`mem2reg` implements the classic dominance-frontier algorithm (Cytron et al., 1991) to insert φ-functions at exactly the join points where the reaching definition is ambiguous. [Chapter 9 — Intermediate Representations and SSA Construction](../part-02-compiler-theory/ch09-ssa-construction.md) covers the theory; [Chapter 21 — SSA, Dominance, and Loops](../part-04-llvm-ir/ch21-ssa-dominance-and-loops.md) covers LLVM's implementation.

### 19.6.3 Lifetime Markers

`llvm.lifetime.start` and `llvm.lifetime.end` mark the semantic live range of an `alloca`:

```llvm
declare void @llvm.lifetime.start.p0(ptr captures(none))
declare void @llvm.lifetime.end.p0(ptr captures(none))

define i32 @with_lifetime() {
entry:
  %x = alloca i32, align 4
  call void @llvm.lifetime.start.p0(ptr %x)
  store i32 99, ptr %x, align 4
  %v = load i32, ptr %x, align 4
  call void @llvm.lifetime.end.p0(ptr %x)
  ret i32 %v
}
```

*Stack slot reuse.* If two `alloca` slots in the same function have non-overlapping lifetimes as marked by these intrinsics, the stack allocator may assign them the same offset in the frame, reducing total stack usage. This is especially important in functions with many small local variables in distinct scopes.

*AddressSanitizer.* ASan uses lifetime markers to detect use-after-scope bugs: when `llvm.lifetime.end` is called on an `alloca`, ASan poisons the corresponding shadow memory. Any subsequent access before the next `llvm.lifetime.start` triggers an ASan report. Without lifetime markers, ASan cannot distinguish a stack variable that has gone out of scope (a use-after-scope error) from one that is still live.

Clang emits lifetime markers for local variables in C++ when optimizations are enabled (`-O1` or higher). At `-O0`, markers are omitted because the alloca pattern is preserved for debugger visibility.

---

## 19.7 `freeze`: Making Poison Safe

### 19.7.1 The Problem Freeze Solves

Poison propagates: any computation that depends on a poison value produces poison. The optimizer relies on this propagation to justify transformations. But sometimes the optimizer wants to *speculate* — to move a computation to a place where its inputs might be poison, on the assumption that the result will not be observed if the inputs are poison. This works only if the speculated computation does not introduce new undefined behavior merely by being executed.

Consider hoisting a load above a branch:

```llvm
; Original: load only if cond is true
br i1 %cond, label %load_block, label %exit

load_block:
  %v = load i32, ptr %p, align 4
  ...
```

If the optimizer wants to hoist `%v = load i32, ptr %p` above the branch (to expose it to out-of-order execution), it must ensure that loading from `%p` when `cond` is false does not cause a fault. If `%p` might be null when `cond` is false, a speculative load would segfault. But if the optimizer can determine `%p` is non-null and within the address space, it can hoist the load, producing a value that might be garbage (if the access was never "supposed" to happen), but not a segfault. The resulting value is semantically poison — it should not be used if `cond` is false — but its production is safe.

### 19.7.2 Syntax and Semantics

```llvm
%y = freeze T %x
```

`freeze` is the escape hatch from poison. Its semantics:

- If `%x` is neither `undef` nor poison, `%y = %x` exactly.
- If `%x` is `undef` or poison, `%y` is an arbitrary but fixed value of type `T`. The value is nondeterministic — different executions may produce different values — but within a single execution, each evaluation of the same `freeze` instruction produces the same value.

The critical property is that the result of `freeze` is *never* poison or undef. It is always a concrete (if arbitrary) value. This makes it safe to pass to any instruction that would have undefined behavior on poison.

```llvm
; Example: freeze an nsw add before using it as a branch condition
%sum        = add nsw i32 %x, %y        ; poison if overflow occurs
%safe_sum   = freeze i32 %sum           ; converts poison to an arbitrary i32
%is_pos     = icmp sgt i32 %safe_sum, 0 ; safe: no more poison
br i1 %is_pos, label %positive, label %other
```

Without `freeze`, using `%sum` as a branch condition when it is poison is undefined behavior. With `freeze`, the branch is taken in an unspecified direction but without UB.

### 19.7.3 `freeze`, `undef`, and the Transition from `undef` to `poison`

LLVM historically had two distinct "non-values": `undef` (an arbitrary fixed value within one basic block) and, later, `poison` (which propagates and causes UB if observed). The current direction is to phase out `undef` entirely in favor of `poison` + `freeze`: anywhere that `undef` was used to mean "I don't care about this value," the idiomatic replacement is `freeze poison` or, more practically, `freeze` applied to a poisoned result.

The `freeze` instruction was added in LLVM 11. Since then, passes increasingly emit `freeze` to prevent speculative hoisting from introducing poison paths where none existed in the original program. The full formal treatment, including the Vellvm formalization and the interaction with the C abstract machine, appears in [Chapter 171 — The Undef/Poison Story Formally](../part-24-verified-compilation/ch171-the-undef-poison-story-formally.md).

---

## 19.8 `ptrtoint`/`inttoptr` and Pointer Provenance

### 19.8.1 The Two Conversion Instructions

```llvm
; Convert a pointer to its numeric address
%addr = ptrtoint ptr %p to i64

; Convert an integer to a pointer
%q = inttoptr i64 %n to ptr
```

Both conversions are value-preserving in the numeric sense: the integer produced by `ptrtoint` is the actual byte address of `%p` at runtime, and the pointer produced by `inttoptr` has the same numeric address as `%n`. But numeric equality of addresses is not the same as semantic equivalence of pointers, and this distinction is the source of the most subtle bugs in LLVM's memory model.

### 19.8.2 Pointer Provenance

LLVM's memory model associates each pointer with a *provenance* — a record of which allocation the pointer is derived from. Two pointers may have the same numeric address but different provenance; the alias analysis is permitted to conclude they do not alias. This is not a theoretical nicety: it is the foundation of several important optimizations.

Consider:

```c
int a, b;
int *p = &a;
int *q = (int *)((uintptr_t)p + 0); // same address, but via inttoptr
*p = 1;
*q = 2;
return a;
```

The numeric address of `q` is identical to `p`. However, `q` was produced via `inttoptr` and has lost its provenance. LLVM's type-based alias analysis (TBAA) and basic alias analysis can observe that `q` has no provenance, making it a "wild" pointer that could alias anything. The alias analysis must be conservative: it cannot prove `p` and `q` do not alias, so it must assume they do, preventing the `*p = 1` store from being sunk past `*q = 2`.

In practice, however, some alias analysis implementations in LLVM have historically been *too aggressive* — treating `inttoptr` as non-aliasing with provenance-tracked pointers — which led to real miscompilation bugs in programs that deliberately manipulated pointers as integers (common in lock-free data structures, in the Linux kernel, and in garbage collector implementations).

### 19.8.3 The PNVI-ae-udi Model

The working C/C++ memory model for pointer provenance is the *PNVI-ae-udi* model (Pointer, Not Value, Implicit-Exposed, Arithmetic, Under-Defined Integer), proposed in ISO WG14 paper N2676 and the related N2577 (Memarian et al., 2019). Its key rules relevant to LLVM:

1. **Provenance is per-allocation.** Every `malloc`, stack allocation, or static allocation creates a new provenance.
2. **Pointer arithmetic preserves provenance.** `p + n` has the same provenance as `p`.
3. **Round-trip through integer.** `inttoptr(ptrtoint(p))` under PNVI-ae has the provenance of `p` — the "ae" means "address-exposed," so once a pointer's address is taken as an integer, that allocation's provenance is exposed, and any integer that matches the address is treated as potentially carrying that provenance.

LLVM does not yet fully implement PNVI-ae-udi. The current alias analysis (`BasicAA`, TBAA) uses simpler approximations. There is ongoing work (see the llvm-dev thread "Rethinking LLVM's string/pointer model" and the LLVM RFC on provenance-based alias analysis) to align LLVM's memory model with PNVI-ae-udi.

### 19.8.4 Why `inttoptr` Is Dangerous for Alias Analysis

The fundamental problem:

```llvm
; %p has provenance of allocation A
store i32 1, ptr %p, align 4      ; store to allocation A

; Produce a wild pointer via inttoptr
%addr = ptrtoint ptr %p to i64
%q = inttoptr i64 %addr to ptr    ; %q has no tracked provenance

store i32 2, ptr %q, align 4      ; store to... unknown allocation

; Can we assume the first store is still visible?
%v = load i32, ptr %p, align 4    ; Is this 1 or 2?
```

Without the `inttoptr`, alias analysis knows `%p` and `%q` are the same pointer (or can prove they do not alias). With `inttoptr`, `%q` has no provenance, and basic alias analysis must conservatively assume it aliases `%p`. The result: the store to `%q` is a potential clobber of `%p`, and the load cannot be forwarded from the first store value `1`.

For this reason, Clang avoids emitting `inttoptr` whenever possible. Pointer casts in C (`(T *)p`) are handled by changing the GEP element type, not by round-tripping through an integer. The only common cases where `inttoptr` appears legitimately are:

1. Inline assembly with integer outputs that are used as pointers.
2. Explicit `uintptr_t` manipulation in user code (e.g., tagging low bits of a pointer).
3. Constant expressions: `(void *)0x100000` appears as `inttoptr i64 1048576 to ptr` in LLVM IR for absolute addresses in embedded firmware.

In all of these cases, the programmer is explicitly opting out of provenance tracking, and the alias analysis conservatism is expected and acceptable.

---

## 19.9 Chapter Summary

- **`nsw`/`nuw`** on integer arithmetic assert no signed or unsigned overflow respectively; if the assertion is violated, the result is poison. Clang emits `nsw` for every signed C integer operation, leveraging C's signed overflow UB. `nuw` appears in pointer arithmetic and unsigned loop counters.

- **`exact`** on `sdiv`, `udiv`, `lshr`, and `ashr` asserts no remainder; it enables `sdiv exact` → `ashr` strength reduction. `lshr exact` and `ashr exact` assert that the shifted-out bits are all zero.

- **`disjoint`** on `or` (LLVM 16+) asserts no bits in common between operands, enabling `or disjoint` → `add` substitution and tighter known-bits analysis. Clang emits it for bit-field packing and tag word construction.

- **Fast-math flags** (`nnan`, `ninf`, `nsz`, `arcp`, `contract`, `afn`, `reassoc`, or the combined `fast`) selectively relax IEEE 754 semantics. Each flag buys a specific class of optimization; `fast` is the union. Clang maps `-ffast-math` and its sub-flags directly to per-instruction IR annotations.

- **`getelementptr`** computes addresses by walking a type tree; it never touches memory. The first index strides through the base element type; subsequent indices descend into aggregates. Struct indices must be `i32` constants. The `inbounds` flag asserts the result stays within the source allocation and is required for alias analysis to reason precisely about GEP chains.

- **`load`/`store`** with `align N` assert pointer alignment; misalignment is UB regardless of hardware tolerance. The `volatile` keyword prevents removal and reordering of a specific access but provides no multi-threaded ordering guarantee. Atomic loads and stores layer ordering semantics (`unordered` through `seq_cst`) on top of alignment-safe accesses; full treatment is in Chapter 27.

- **`alloca`** allocates stack memory and returns a `ptr`; Clang places all scalar allocas in the entry block and relies on `mem2reg` to promote them to SSA φ-functions. Lifetime markers (`llvm.lifetime.start.p0` / `llvm.lifetime.end.p0`) enable stack slot reuse and ASan use-after-scope detection.

- **`freeze`** converts poison or undef to an arbitrary but fixed concrete value, enabling the optimizer to speculate safely. It is the mechanism by which LLVM avoids introducing UB when hoisting or replicating computations. `undef` is being replaced by `poison` + `freeze` throughout the IR.

- **`ptrtoint`/`inttoptr`** perform numeric address conversions but destroy pointer provenance. LLVM's alias analysis tracks provenance to prove non-aliasing; `inttoptr` produces provenance-free "wild" pointers that force conservative alias assumptions. Clang minimizes `inttoptr` emission; it appears mainly for inline assembly and explicit `uintptr_t` manipulation.

---

*Cross-references:* [Chapter 9 — Intermediate Representations and SSA Construction](../part-02-compiler-theory/ch09-ssa-construction.md) · [Chapter 17 — The Type System](../part-04-llvm-ir/ch17-the-type-system.md) · [Chapter 20 — Instructions II — Control Flow and Aggregates](../part-04-llvm-ir/ch20-instructions-control-flow-and-aggregates.md) · [Chapter 21 — SSA, Dominance, and Loops](../part-04-llvm-ir/ch21-ssa-dominance-and-loops.md) · [Chapter 27 — Coroutines and Atomics](../part-04-llvm-ir/ch27-coroutines-and-atomics.md) · [Chapter 171 — The Undef/Poison Story Formally](../part-24-verified-compilation/ch171-the-undef-poison-story-formally.md)

*Reference links:* [LangRef — Binary Operations](https://llvm.org/docs/LangRef.html#binary-operations) · [LangRef — Memory Access and Addressing Operations](https://llvm.org/docs/LangRef.html#memory-access-and-addressing-operations) · [LangRef — freeze instruction](https://llvm.org/docs/LangRef.html#freeze-instruction) · [LangRef — getelementptr instruction](https://llvm.org/docs/LangRef.html#getelementptr-instruction) · [Memarian et al., "Exploring C Semantics and Pointer Provenance" (POPL 2019)](https://www.cl.cam.ac.uk/~pes20/cerberus/cerberus-popl2019.pdf) · [LLVM Poison Semantics RFC](https://llvm.org/docs/PoisonSemantics.html)


---

@copyright jreuben11
