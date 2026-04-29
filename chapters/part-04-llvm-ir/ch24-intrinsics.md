# Chapter 24 — Intrinsics

*Part IV — LLVM IR*

LLVM IR is intentionally small: a few dozen opcodes cover arithmetic, memory access, control flow, and aggregate operations. Yet compilers must express a much richer set of operations — transcendental mathematics, memory bulk-copy, vector reductions, atomics, exception handling, garbage collector hooks, coroutine suspension points, and hundreds of target-specific machine instructions. Intrinsics are the mechanism that bridges this gap. They look like function calls — with `call` or `invoke` opcode, a callee named `@llvm.*`, and an ordinary argument list — but the optimizer and backend treat them as first-class operations with precisely specified semantics. No definition exists in IR; the declaration is the specification.

This chapter is a systematic reference to the entire intrinsic vocabulary of LLVM 22.1.x. It covers the declaration format, attribute conventions, and optimizer implications common to all intrinsics, then works through each major category in depth. All IR examples are verified against LLVM 22.1.3 (`llvm-as`).

---

## 24.1 The Anatomy of an Intrinsic

### 24.1.1 Declaration Format

Every intrinsic appears in IR as a `declare` statement with no body. The function name begins with `llvm.` followed by a dot-separated path that identifies the operation and, for overloaded intrinsics, encodes the type. For example:

```llvm
declare void @llvm.memcpy.p0.p0.i64(
    ptr noalias nocapture writeonly,
    ptr noalias nocapture readonly,
    i64,
    i1 immarg)

declare double @llvm.sqrt.f64(double %val)

declare {i32, i1} @llvm.sadd.with.overflow.i32(i32 %lhs, i32 %rhs)
```

The type suffix in the name (`f64`, `i32`, `p0.p0.i64`) is produced by the name-mangling scheme defined in [`llvm/lib/IR/Function.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/IR/Function.cpp). For non-overloaded intrinsics such as `llvm.trap`, the name is a fixed string with no suffix.

### 24.1.2 Overloading

Many intrinsics are polymorphic over their element type. The overloading mechanism is purely a naming convention: when `Intrinsic::getName(ID, ArrayRef<Type*>, Module*)` is called, it appends type encodings to the base name to produce the full mangled name. The inverse direction — `Intrinsic::lookupIntrinsicID(StringRef)` — maps the mangled name back to the `Intrinsic::ID` enum value.

The `Intrinsic::isOverloaded(ID)` predicate distinguishes overloaded intrinsics from fixed-name ones. For a pass that pattern-matches intrinsic calls by ID, the ID comparison is type-independent: `Function::getIntrinsicID()` returns the same `Intrinsic::sqrt` regardless of whether the argument is `float`, `double`, or `<4 x float>`.

### 24.1.3 The IntrinsicInst Class Hierarchy

From the C++ API perspective, every call to an intrinsic is a `CallInst` whose callee is an intrinsic `Function`. The header [`llvm/include/llvm/IR/IntrinsicInst.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/IntrinsicInst.h) provides typed wrappers that support `isa<>`, `cast<>`, and `dyn_cast<>`:

```
IntrinsicInst : CallInst
├── DbgInfoIntrinsic
│   ├── DbgVariableIntrinsic
│   │   ├── DbgDeclareInst
│   │   ├── DbgValueInst
│   │   └── DbgAssignInst
│   └── DbgLabelInst
├── MemIntrinsicBase<T>
│   ├── MemIntrinsic  (memcpy, memmove, memset)
│   │   ├── MemSetInst
│   │   ├── MemTransferInst
│   │   │   ├── MemCpyInst
│   │   │   └── MemMoveInst
│   ├── AnyMemIntrinsic  (atomic element-unordered variants)
├── BinaryOpIntrinsic
│   ├── WithOverflowInst
│   └── SaturatingInst
├── VAStartInst, VAEndInst, VACopyInst
├── InstrProfInstBase → InstrProfCntrInstBase → InstrProfIncrementInst …
├── GCProjectionInst → GCRelocateInst, GCResultInst
├── AssumeInst
└── ConvergenceControlInst
```

Pattern-matching an intrinsic call in a pass uses `dyn_cast<MemCpyInst>(I)` rather than checking the intrinsic ID directly, which provides typed accessor methods (`getDest()`, `getSource()`, `getLength()`, `isVolatile()`).

### 24.1.4 TableGen Definition Structure

Intrinsics are defined in TableGen source under [`llvm/include/llvm/IR/Intrinsics.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/Intrinsics.td) and per-target files such as `IntrinsicsX86.td`. The `Intrinsic` class takes three arguments: a list of return types, a list of parameter types (using type variables like `llvm_anyfloat_ty` for overloaded positions), and a list of `IntrinsicProperty` records. The `DefaultAttrsIntrinsic` subclass applies `IntrNoFree`, `IntrWillReturn`, and `IntrNoSync` automatically, which is correct for the vast majority of pure computation intrinsics.

The TableGen build generates `IntrinsicEnums.inc` (the `Intrinsic::ID` enum), `IntrinsicImpl.inc` (the `getName`/`getType`/`getAttributes` implementations), and per-target `*GenInstrInfo.inc` tables that map intrinsic IDs to `ISD::INTRINSIC_W_CHAIN` DAG nodes or direct instruction patterns during SelectionDAG lowering.

---

## 24.2 Intrinsic Attributes and Their Optimizer Implications

The attributes on an intrinsic declaration are the mechanism through which the optimizer understands the intrinsic's semantics without knowing anything about its implementation. Attributes on `declare` statements are generated from the `IntrinsicProperty` records in the TableGen definition and are enforced at IR parse time — passing the wrong attributes to a known intrinsic is an IR verification error.

### 24.2.1 Memory Effect Attributes

| Attribute / TableGen property | Meaning |
|-------------------------------|---------|
| `memory(none)` / `IntrNoMem` | No memory access; call may be CSE'd, DCE'd, speculated, hoisted out of loops |
| `memory(read)` / `IntrReadMem` | Read-only; call may not be eliminated but can be sunk below writes |
| `memory(write)` / `IntrWriteMem` | Write-only; dead stores before this call may be removed |
| `memory(argmem: readwrite)` / `IntrArgMemOnly` | Accesses only memory reachable through pointer arguments |
| `memory(inaccessiblemem: readwrite)` / `IntrInaccessibleMemOnly` | Accesses only memory invisible to the IR (e.g., errno, thread-local state) |

`llvm.sqrt.f64` carries `memory(none)` — it has no side effects and its result is a pure function of its argument. The vectorizer can therefore hoist it out of a loop without a version check. By contrast, `llvm.memcpy` carries `memory(argmem: readwrite)`, which prevents the optimizer from moving it across other memory operations on the same pointer.

### 24.2.2 Control-Flow and Termination Attributes

`nounwind` is present on virtually every intrinsic: they do not throw C++ exceptions. `willreturn` (TableGen: `IntrWillReturn`) asserts that the intrinsic always terminates, enabling the optimizer to place code after the call without inserting a landing pad. `nosync` (TableGen: `IntrNoSync`) asserts no synchronization operations, allowing the optimizer to freely reorder the call relative to other unsynchronized accesses.

`speculatable` (TableGen: `IntrSpeculatable`) is set on pure math intrinsics such as `llvm.sqrt` and `llvm.sin`. It means the intrinsic can be called on any path, including those not reached in the original program, without introducing new exceptional behavior. The LICM pass uses this to hoist loop-invariant math calls out of loop bodies.

`nocallback` (TableGen: `IntrNoCallback`) indicates the intrinsic does not invoke callbacks or function pointers. Combined with `nosync`, it enables GVN to reason across calls.

`inaccessiblememonly` is used for intrinsics that touch hidden program state. `llvm.assume` uses `IntrInaccessibleMemOnly` so that the optimizer cannot reorder assumptions relative to each other.

---

## 24.3 Variable Argument Intrinsics

Variadic functions in C require a way to traverse the argument list whose layout is determined by the platform ABI. LLVM IR provides three intrinsics for this purpose:

```llvm
declare void @llvm.va_start(ptr %ap)
declare void @llvm.va_end(ptr %ap)
declare void @llvm.va_copy(ptr %dest, ptr %src)
```

`llvm.va_start` initializes the `va_list` object pointed to by `%ap` so that subsequent `va_arg` IR instructions can extract successive arguments. `llvm.va_end` destroys the `va_list`. `llvm.va_copy` copies the state of one `va_list` to another.

The `va_list` is an opaque structure; its layout is target-defined. On x86-64 System V, `va_list` is a 24-byte struct containing `gp_offset`, `fp_offset`, `overflow_arg_area`, and `reg_save_area` pointers (the AMD64 ABI §3.5.7). On AArch64, it is a 32-byte struct defined by the AAPCS64 §Appendix B.4. The IR `va_arg` instruction fetches a value of a specified type and advances the `va_list`:

```llvm
; Variadic function using va intrinsics
define i32 @sum_ints(i32 %count, ...) {
entry:
  %va = alloca ptr, align 8
  call void @llvm.va_start(ptr %va)
  ; %arg = va_arg ptr %va, i32  ; promoted to i32 per default argument promotions
  call void @llvm.va_end(ptr %va)
  ret i32 0
}
```

The wrapper classes `VAStartInst`, `VAEndInst`, and `VACopyInst` in `IntrinsicInst.h` provide `getArgList()` which returns the pointer to the `va_list` object, enabling passes to identify and reason about variadic argument traversal patterns.

---

## 24.4 Memory Intrinsics

### 24.4.1 memcpy, memmove, and memset

The three bulk-memory intrinsics have overloaded signatures that encode the address space of the pointer operands:

```llvm
; Non-volatile memcpy from address space 0 to address space 0, 64-bit length
declare void @llvm.memcpy.p0.p0.i64(
    ptr noalias nocapture writeonly,
    ptr noalias nocapture readonly,
    i64, i1 immarg)

; Non-volatile memmove (overlapping regions allowed)
declare void @llvm.memmove.p0.p0.i64(
    ptr nocapture writeonly,
    ptr nocapture readonly,
    i64, i1 immarg)

; Non-volatile memset
declare void @llvm.memset.p0.i64(
    ptr nocapture writeonly,
    i8, i64, i1 immarg)
```

The fourth argument is the `volatile` flag (always an immediate `i1`): `false` for ordinary copies, `true` for copies from/to volatile-qualified memory (which the optimizer must not eliminate or reorder). Alignment is expressed through `align` parameter attributes on the pointer arguments rather than as an IR argument; this mirrors the change made in LLVM 10 when the alignment parameter was removed.

`llvm.memcpy` requires that the source and destination do not overlap (the `noalias` attribute reflects this); `llvm.memmove` handles overlap correctly but cannot carry `noalias` on both pointers.

### 24.4.2 Inline Variants

`llvm.memcpy.inline` and `llvm.memset.inline` have identical signatures to their non-inline counterparts but carry an additional semantic guarantee: the backend must lower them to inline code, never to a library call. This is critical for code paths that must be free of external dependencies (kernel code, signal handlers, sanitizer runtime initialization):

```llvm
declare void @llvm.memcpy.inline.p0.p0.i64(
    ptr noalias nocapture writeonly,
    ptr noalias nocapture readonly,
    i64, i1 immarg)
```

The length of an inline variant is typically a small compile-time constant, though the IR does not enforce this.

### 24.4.3 Atomic Element-Unordered Variants

For concurrent data structure manipulation where individual element loads and stores must be atomic but the overall transfer is not synchronized, LLVM provides atomic memory intrinsics:

```llvm
declare void @llvm.memcpy.element.unordered.atomic.p0.p0.i64(
    ptr noalias nocapture writeonly,
    ptr noalias nocapture readonly,
    i64, i32 immarg)    ; element size: 1, 2, 4, or 8

declare void @llvm.memmove.element.unordered.atomic.p0.p0.i64(
    ptr nocapture writeonly, ptr nocapture readonly, i64, i32 immarg)

declare void @llvm.memset.element.unordered.atomic.p0.i64(
    ptr nocapture writeonly, i8, i64, i32 immarg)
```

The fourth argument specifies the element size in bytes (must be 1, 2, 4, or 8). Each element-sized store or load uses `unordered` atomic memory ordering. This enables Java-style array copy semantics where racing reads see either the old or new value but never a torn word. The passes in [`llvm/lib/Transforms/Utils/LowerAtomicIntrinsics.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Utils/LowerAtomicIntrinsics.cpp) handle the lowering.

### 24.4.4 The AnyMemIntrinsic Class

Some analysis and transformation passes need to reason about both the ordinary and atomic variants of memory intrinsics uniformly. `AnyMemIntrinsic` is the common base class in `IntrinsicInst.h` that covers both families. The `MemIntrinsic` subclass covers only the non-atomic variants and provides the `isVolatile()` accessor; `AnyMemIntrinsic` covers all variants and provides only length and alignment queries. The `MemorySSA` analysis uses `AnyMemIntrinsic` to find memory-touching intrinsics when building def-use chains.

---

## 24.5 Floating-Point Math Intrinsics

### 24.5.1 Correctly-Rounded Operations

The floating-point math intrinsics express operations whose semantics is defined by IEEE 754 and the C standard. They are overloaded over any floating-point scalar or vector type:

| Intrinsic | Operation | Notes |
|-----------|-----------|-------|
| `llvm.sqrt` | Square root | IEEE 754 correctly rounded |
| `llvm.fabs` | Absolute value | Always exact |
| `llvm.fma` | Fused multiply-add | Single rounding, IEEE 754 §5.4.1 |
| `llvm.copysign` | Copy sign bit | Always exact |
| `llvm.minnum` | Minimum (NaN-propagating if either operand is NaN) | IEEE 754-2008 minNum |
| `llvm.maxnum` | Maximum (NaN-propagating) | IEEE 754-2008 maxNum |
| `llvm.minimum` | Minimum (NaN-propagating; -0.0 < +0.0) | IEEE 754-2019 minimum |
| `llvm.maximum` | Maximum (NaN-propagating; +0.0 > -0.0) | IEEE 754-2019 maximum |

```llvm
define double @fused_mad(double %a, double %b, double %c) {
entry:
  %r = call double @llvm.fma.f64(double %a, double %b, double %c)
  ret double %r
}
declare double @llvm.fma.f64(double, double, double)
```

`llvm.fma` is particularly important: it guarantees a single rounding operation, unlike `fadd(fmul(a, b), c)` which rounds twice. On targets with a native FMA instruction (x86 AVX2 `vfmadd`, AArch64 `fmadd`, PowerPC `fmadd`), the backend lowers `llvm.fma` directly. On targets without FMA, it expands to a libcall to `fma(3)` from libm. The auto-vectorizer and the `FMACombine` pass use this distinction to decide whether to form FMA patterns.

### 24.5.2 Transcendental Functions

```llvm
declare double @llvm.powi.f64.i32(double, i32)  ; x^n, integer exponent
declare double @llvm.pow.f64(double, double)      ; x^y, general power
declare double @llvm.exp.f64(double)
declare double @llvm.exp2.f64(double)
declare double @llvm.log.f64(double)
declare double @llvm.log2.f64(double)
declare double @llvm.log10.f64(double)
declare double @llvm.sin.f64(double)
declare double @llvm.cos.f64(double)
```

These are not required to be correctly-rounded; their accuracy matches the corresponding libm functions. They carry `speculatable` and `memory(none)` so the optimizer can hoist them freely. `llvm.powi` is special: the exponent is an integer, which enables constant folding and special-case lowering (e.g., `powi(x, 2)` → `fmul x, x`).

### 24.5.3 Rounding Modes

The family of rounding intrinsics corresponds to the C99 `<math.h>` rounding functions:

| Intrinsic | C equivalent | Rounding direction |
|-----------|-------------|--------------------|
| `llvm.ceil` | `ceil` | Toward +∞ |
| `llvm.floor` | `floor` | Toward −∞ |
| `llvm.trunc` | `trunc` | Toward zero |
| `llvm.round` | `round` | Nearest, ties away from zero |
| `llvm.roundeven` | `roundeven` | Nearest, ties to even (banker's rounding) |
| `llvm.rint` | `rint` | Current floating-point rounding mode; may raise inexact |
| `llvm.nearbyint` | `nearbyint` | Current rounding mode; does NOT raise inexact |

The distinction between `llvm.rint` and `llvm.nearbyint` affects whether the MXCSR/FPCR inexact exception flag is set. Both are `speculatable`, but `llvm.rint` is `memory(inaccessiblemem: readwrite)` because it observes and may modify floating-point status flags, while `llvm.nearbyint` is `memory(none)`.

### 24.5.4 Fast-Math Flags and Math Intrinsics

Ordinary `fadd`, `fmul`, etc. accept fast-math flags (`reassoc`, `nnan`, `ninf`, `nsz`, `arcp`, `contract`, `afn`, `fast`) as IR-level annotations that relax IEEE 754 semantics. Intrinsic calls follow the same convention: fast-math flags appear between the `call` opcode and the return type:

```llvm
; FMA with nnan+ninf allows the optimizer to exploit algebraic identities
%r = call nnan ninf double @llvm.fma.f64(double %a, double %b, double %c)

; Approximate sqrt — allows rsqrt-based implementation
%r2 = call afn float @llvm.sqrt.f32(float %x)
```

The `afn` (approximate function) flag specifically relaxes the correctness requirement on math intrinsics, enabling the vectorizer to substitute hardware reciprocal-square-root instructions (`rsqrtps` on x86, `frsqrte` on AArch64) for `llvm.sqrt`.

---

## 24.6 Bit-Manipulation Intrinsics

### 24.6.1 Byte and Bit Reversal

```llvm
declare i32 @llvm.bswap.i32(i32)      ; reverse byte order
declare i64 @llvm.bswap.i64(i64)
declare i32 @llvm.bitreverse.i32(i32) ; reverse all bits
```

`llvm.bswap` is lowered to native byte-swap instructions (`bswap` on x86, `rev` on AArch64, `sthbrx/stwbrx` on PowerPC). It is folded by InstCombine when the argument is a constant and by `TargetLowering` when the surrounding shift/or pattern matches the canonical bswap idiom recognized in C code.

### 24.6.2 Population Count and Leading/Trailing Zeros

```llvm
declare i32 @llvm.ctpop.i32(i32)              ; count set bits
declare i32 @llvm.ctlz.i32(i32, i1 immarg)   ; count leading zeros; undef if zero
declare i32 @llvm.cttz.i32(i32, i1 immarg)   ; count trailing zeros; undef if zero
```

The second argument to `ctlz` and `cttz` is a poison-safety flag: `true` means the result is `poison` if the input is zero (enabling more efficient code generation on targets where the native instruction has undefined behavior on zero), `false` means the result is defined (returning the bit-width) even for zero. Clang emits `true` for `__builtin_clz` (which has undefined behavior on zero in C) and `false` for `__builtin_clzg` (C23, defined for zero). On x86, `ctlz true` maps to `BSR` or `LZCNT`; `ctlz false` maps to `LZCNT` or a conditional subtraction.

### 24.6.3 Funnel Shifts

```llvm
declare i32 @llvm.fshl.i32(i32 %hi, i32 %lo, i32 %shift)  ; funnel shift left
declare i32 @llvm.fshr.i32(i32 %hi, i32 %lo, i32 %shift)  ; funnel shift right
```

`fshl(hi, lo, shift)` concatenates `hi:lo` into a 64-bit value, shifts left by `shift % 32` bits, and returns the upper 32 bits. `fshr` returns the lower 32 bits after a right shift. When `hi == lo`, this computes a rotation:

```llvm
; Rotate left by %shift positions
define i32 @rotl(i32 %val, i32 %shift) {
entry:
  %r = call i32 @llvm.fshl.i32(i32 %val, i32 %val, i32 %shift)
  ret i32 %r
}
declare i32 @llvm.fshl.i32(i32, i32, i32)
```

The backend lowers `fshl(x, x, k)` to native rotation instructions: `ROL` on x86, `ROR`/`EXTR` on AArch64. InstCombine recognizes the canonical idiom `(x << k) | (x >> (32-k))` in the input IR and converts it to `llvm.fshl`.

---

## 24.7 Arithmetic with Overflow and Saturation

### 24.7.1 Overflow-Checking Intrinsics

The six overflow-checking intrinsics return a struct containing the result and an overflow flag:

```llvm
declare {i32, i1} @llvm.sadd.with.overflow.i32(i32, i32)
declare {i32, i1} @llvm.uadd.with.overflow.i32(i32, i32)
declare {i32, i1} @llvm.ssub.with.overflow.i32(i32, i32)
declare {i32, i1} @llvm.usub.with.overflow.i32(i32, i32)
declare {i32, i1} @llvm.smul.with.overflow.i32(i32, i32)
declare {i32, i1} @llvm.umul.with.overflow.i32(i32, i32)
```

The `s` prefix indicates signed, `u` unsigned. The struct return type is extracted with `extractvalue`:

```llvm
define i32 @checked_add(i32 %a, i32 %b) {
entry:
  %res = call {i32, i1} @llvm.sadd.with.overflow.i32(i32 %a, i32 %b)
  %val = extractvalue {i32, i1} %res, 0
  %ovf = extractvalue {i32, i1} %res, 1
  br i1 %ovf, label %overflow, label %ok
overflow:
  call void @abort() noreturn
  unreachable
ok:
  ret i32 %val
}
```

On x86-64, `llvm.sadd.with.overflow.i32` lowers to an `add` instruction followed by a `seto` to capture the overflow flag from EFLAGS. On AArch64, it lowers to `adds` (set flags variant) followed by `cset` for the carry or overflow bit. The `WithOverflowInst` wrapper class in `IntrinsicInst.h` provides `getBinaryOp()` (returns the `Instruction::BinaryOps` value), `isSigned()`, and `getLHS()`/`getRHS()` methods.

Clang emits `llvm.sadd.with.overflow` for `__builtin_sadd_overflow` and `__builtin_add_overflow` when the types are integral. Swift uses the overflow intrinsics pervasively for its checked integer arithmetic.

### 24.7.2 Saturating Arithmetic

Saturation clamps the result to the representable range instead of wrapping:

```llvm
declare i32 @llvm.sadd.sat.i32(i32, i32)   ; signed saturating add
declare i32 @llvm.uadd.sat.i32(i32, i32)   ; unsigned saturating add
declare i32 @llvm.ssub.sat.i32(i32, i32)   ; signed saturating subtract
declare i32 @llvm.usub.sat.i32(i32, i32)   ; unsigned saturating subtract
declare i32 @llvm.sshl.sat.i32(i32, i32)   ; signed saturating shift left
declare i32 @llvm.ushl.sat.i32(i32, i32)   ; unsigned saturating shift left
```

These map to `PADDS`/`PSUBSW` on x86 SSE2, `sqadd`/`sqsub` on AArch64 NEON, and the saturating arithmetic instructions on DSP targets. Signal-processing code, image processing pipelines, and any domain with bounded dynamic range use saturation to avoid clipping artifacts from overflow.

---

## 24.8 Vector Reduction Intrinsics

Vector reductions compute a scalar result from all elements of a vector. They are the IR-level abstraction over horizontal operations:

```llvm
; Integer reductions — exact
declare i32 @llvm.vector.reduce.add.v4i32(<4 x i32>)
declare i32 @llvm.vector.reduce.mul.v4i32(<4 x i32>)
declare i32 @llvm.vector.reduce.and.v4i32(<4 x i32>)
declare i32 @llvm.vector.reduce.or.v4i32(<4 x i32>)
declare i32 @llvm.vector.reduce.xor.v4i32(<4 x i32>)
declare i32 @llvm.vector.reduce.smin.v4i32(<4 x i32>)
declare i32 @llvm.vector.reduce.smax.v4i32(<4 x i32>)
declare i32 @llvm.vector.reduce.umin.v4i32(<4 x i32>)
declare i32 @llvm.vector.reduce.umax.v4i32(<4 x i32>)

; Floating-point reductions — take an initial accumulator
declare float @llvm.vector.reduce.fadd.v4f32(float %start, <4 x float> %vec)
declare float @llvm.vector.reduce.fmul.v4f32(float %start, <4 x float> %vec)

; FP min/max — IEEE NaN semantics
declare float @llvm.vector.reduce.fmin.v4f32(<4 x float>)
declare float @llvm.vector.reduce.fmax.v4f32(<4 x float>)
```

The floating-point accumulator form (`fadd` and `fmul`) is necessary for correct sequential semantics: the initial value (typically `0.0` for add, `1.0` for multiply) establishes the identity element, and the reduction is performed left-to-right in element order when no fast-math flags are present. With `reassoc`, the order is unspecified and the backend may use a tree reduction for better latency.

The SLP vectorizer and the loop vectorizer in [`llvm/lib/Transforms/Vectorize/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Vectorize/) emit these intrinsics when converting scalar horizontal reductions to their vector form. On x86, they lower to `phaddw`, `vpaddq`, `vphminposuw`, or the AVX-512 `vreduceps` family. On AArch64 NEON, they map to `addv`, `sminv`, `smaxv`, `faddp`.

---

## 24.9 Masked Vector Intrinsics

Masked intrinsics add a boolean predicate vector that gates memory accesses element-wise. They are the IR representation of the masked operations required by auto-vectorization of loops with conditionals and by explicit SIMD programming models.

### 24.9.1 Masked Load and Store

```llvm
; Load <4 x float> from %ptr, setting masked-off lanes to %passthrough
declare <4 x float> @llvm.masked.load.v4f32.p0(
    ptr %ptr,
    i32 immarg %alignment,
    <4 x i1> %mask,
    <4 x float> %passthrough)

; Store <4 x float> to %ptr, skipping masked-off lanes
declare void @llvm.masked.store.v4f32.p0(
    <4 x float> %data,
    ptr %ptr,
    i32 immarg %alignment,
    <4 x i1> %mask)
```

```llvm
define <4 x float> @masked_load_example(ptr %p, <4 x i1> %mask) {
entry:
  %res = call <4 x float> @llvm.masked.load.v4f32.p0(
      ptr %p, i32 16, <4 x i1> %mask, <4 x float> zeroinitializer)
  ret <4 x float> %res
}
```

The alignment is an immediate integer argument (not an `align` parameter attribute) because it applies to the entire vector access, not to an individual pointer. On x86, masked loads lower to `VMASKMOVPS` (AVX1), `VPMASKMOVD` (AVX2), or `VMOVDQU32` with a mask register (AVX-512). On AArch64 SVE, they lower to `ld1` with a predicate register.

### 24.9.2 Masked Gather and Scatter

Gather and scatter access a vector of distinct addresses:

```llvm
; Gather: load from each pointer in %ptrs vector
declare <4 x float> @llvm.masked.gather.v4f32.v4p0(
    <4 x ptr> %ptrs,
    i32 immarg %alignment,
    <4 x i1> %mask,
    <4 x float> %passthrough)

; Scatter: store each element to its corresponding pointer
declare void @llvm.masked.scatter.v4f32.v4p0(
    <4 x float> %data,
    <4 x ptr> %ptrs,
    i32 immarg %alignment,
    <4 x i1> %mask)
```

These are required for vectorizing indirect array accesses (`a[idx[i]]`). On x86, they lower to `VGATHERDPS`/`VSCATTERDPS` (AVX2/AVX-512). On targets without hardware gather/scatter, the backend expands them into scalar loads/stores in a sequence generated by `TargetLowering::expandMaskedGather`.

### 24.9.3 Expand Load and Compress Store

These intrinsics perform a compaction: `expandload` reads only the active (mask-true) elements sequentially from a packed array into the corresponding vector lanes; `compressstore` writes only the active lanes sequentially:

```llvm
declare <4 x float> @llvm.masked.expandload.v4f32(
    ptr %ptr, <4 x i1> %mask, <4 x float> %passthrough)

declare void @llvm.masked.compressstore.v4f32(
    <4 x float> %data, ptr %ptr, <4 x i1> %mask)
```

These implement the C++ `std::experimental::simd` `where`-clause semantics and map to `VPEXPANDD`/`VPCOMPRESSD` on AVX-512. They are emitted by the loop vectorizer when the access pattern cannot be expressed as a strided vector access.

---

## 24.10 Scalable Vector Intrinsics

Scalable vectors (type prefix `<vscale x N x T>`) require intrinsics that describe operations whose element count is not known at compile time. The canonical source of information about the actual vector length is `llvm.vscale`:

```llvm
declare i32 @llvm.vscale.i32()
declare i64 @llvm.vscale.i64()
```

`llvm.vscale()` returns the number of 128-bit "chunks" in the scalable vector at runtime. For `<vscale x 4 x i32>`, the actual element count at runtime is `vscale * 4`. On AArch64 SVE, `vscale` equals `VL / 128` where `VL` is the SVE vector length in bits; on RISC-V RVV, `vscale` is determined by the `vtype` CSR.

The `vscale_range(min, max)` function attribute (see Chapter 23) bounds `vscale` statically, enabling the optimizer to constant-fold `llvm.vscale()` when min==max.

### 24.10.1 Step Vector and Insert/Extract

```llvm
; Generate <0, 1, 2, 3, ..., vscale*4-1>
declare <vscale x 4 x i32> @llvm.stepvector.nxv4i32()

; Insert a fixed-length vector into a scalable vector at offset %idx
declare <vscale x 4 x i32> @llvm.vector.insert.nxv4i32.v4i32(
    <vscale x 4 x i32> %vec, <4 x i32> %subvec, i64 immarg %idx)

; Extract a fixed-length slice from a scalable vector
declare <4 x i32> @llvm.vector.extract.v4i32.nxv4i32(
    <vscale x 4 x i32> %vec, i64 immarg %idx)
```

`llvm.stepvector` generates the index vector used in address arithmetic for vectorized loops over scalable-length trip counts. `llvm.vector.insert` and `llvm.vector.extract` bridge between fixed-length SIMD operations (which often have richer support in the ISA) and scalable vector types. The loop vectorizer emits these when generating the scalable vector body plus a fixed-width epilogue.

---

## 24.11 Lifetime and Invariant Intrinsics

### 24.11.1 Lifetime Markers

```llvm
declare void @llvm.lifetime.start.p0(i64 immarg %size, ptr nocapture %ptr)
declare void @llvm.lifetime.end.p0(i64 immarg %size, ptr nocapture %ptr)
```

These mark the live range of a stack allocation. The size is the size of the stack object in bytes (as a compile-time constant), and the pointer must refer to a `alloca`. Clang emits lifetime markers for every local variable, C++ temporary, and OpenMP private variable with a clearly defined scope.

The stack coloring pass ([`llvm/lib/CodeGen/StackColoring.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/StackColoring.cpp)) uses lifetime markers to identify non-overlapping stack slots and merge them, reducing total stack frame size. Two allocas can share stack space if their lifetime intervals do not overlap.

The optimizer treats a `lifetime.start` as a write of `undef` to the pointed-to memory (from the perspective of the abstract machine, the memory may contain garbage before the lifetime begins). This enables GVN to invalidate stale values and MemorySSA to correctly model the allocation.

```llvm
define void @lifetime_example() {
entry:
  %x = alloca i32, align 4
  call void @llvm.lifetime.start.p0(i64 4, ptr %x)
  store i32 42, ptr %x, align 4
  call void @llvm.lifetime.end.p0(i64 4, ptr %x)
  ret void
}
```

### 24.11.2 Invariant Intrinsics

```llvm
declare {ptr, i1} @llvm.invariant.start.p0(i64 immarg %size, ptr nocapture %ptr)
declare void @llvm.invariant.end.p0(ptr %handle, i64 immarg %size, ptr nocapture %ptr)
```

`llvm.invariant.start` asserts that the memory at `%ptr` will not be modified for the duration marked by the corresponding `invariant.end`. The returned token is the handle passed to `invariant.end`. GVN and LICM use these annotations to hoist loads out of loops and to CSE loads across potential stores that are guarded by the invariant.

Clang emits `invariant.start`/`invariant.end` around `const`-qualified objects whose address has not been taken in a way that allows mutation. C++ `constexpr` global variables are a prime example.

### 24.11.3 Invariant Group Intrinsics

C++ strict aliasing rules and virtual dispatch introduce a subtlety: after `placement new`, an object's dynamic type changes, so the vptr (virtual function table pointer) stored at the same address is no longer invariant. The invariant group intrinsics manage this:

```llvm
declare ptr @llvm.launder.invariant.group.p0(ptr %ptr)
declare ptr @llvm.strip.invariant.group.p0(ptr %ptr)
```

`llvm.launder.invariant.group` takes a pointer and returns it with a "fresh" provenance that tells the optimizer the pointed-to memory may have had its type changed (e.g., after placement new). `llvm.strip.invariant.group` removes all invariant group tracking from a pointer, used when passing a pointer through an opaque interface. Both intrinsics return the pointer unchanged at runtime; they exist solely to constrain alias analysis.

---

## 24.12 Pointer Authentication and Security Intrinsics

### 24.12.1 Pointer Authentication (PAC)

AArch64 Pointer Authentication (ARMv8.3-A) cryptographically signs pointers stored in memory to detect corruption by code-injection attacks. LLVM 22 exposes PAC through:

```llvm
declare i64 @llvm.ptrauth.sign(i64 %value, i32 immarg %key, i64 %discriminator)
declare i64 @llvm.ptrauth.auth(i64 %value, i32 immarg %key, i64 %discriminator)
declare i64 @llvm.ptrauth.strip(i64 %value, i32 immarg %key)
declare i64 @llvm.ptrauth.blend(i64 %addr_discriminator, i64 %integer_discriminator)
declare i1  @llvm.ptrauth.test(i64 %value, i32 immarg %key, i64 %discriminator)
```

The `key` selects one of the four PAC keys: `IA` (0), `IB` (1), `DA` (2), `DB` (3). The `discriminator` blends a context value into the signature, preventing a pointer signed in one context from being used in another. `llvm.ptrauth.blend` combines an address discriminator and an integer discriminator according to the AArch64 PAC blending formula.

Clang's `-fptrauth-returns` and `-fptrauth-calls` flags cause Clang to emit `llvm.ptrauth.sign` and `llvm.ptrauth.auth` around function pointers and return addresses. Swift mandates PAC for all function pointers as part of its runtime security model. The intrinsics lower to `PACIB`, `AUTIB`, and `XPACD` machine instructions on AArch64 hardware. The TableGen definitions appear in [`llvm/include/llvm/IR/Intrinsics.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/Intrinsics.td) starting at the `int_ptrauth_sign` definition.

### 24.12.2 Stack Save and Restore

```llvm
declare ptr @llvm.stacksave()
declare void @llvm.stackrestore(ptr %saved_sp)
```

These capture and restore the stack pointer, enabling variable-length arrays (VLAs) and `alloca` within loops to release stack memory before the function returns. Clang emits `llvm.stacksave` before a VLA declaration and `llvm.stackrestore` at each exit point of the scope containing the VLA. The backend maps both to direct stack-pointer manipulations: `mov rsp, %rax` / `mov %rax, rsp` on x86-64.

### 24.12.3 Object Size Query

```llvm
declare i64 @llvm.objectsize.i64.p0(
    ptr %ptr,
    i1 immarg %min,
    i1 immarg %nullknown,
    i1 immarg %dynamic)
```

`llvm.objectsize` queries the size of the object pointed to by `%ptr` as known at compile time. The optimizer resolves it to a constant when the allocation site is statically visible (e.g., `alloca [16 x i8]` → `llvm.objectsize` returns 16). The `min` flag selects whether to return the minimum or maximum object size when the bounds are not exactly known. The `nullknown` flag controls whether null pointers count as zero-sized. When the size cannot be determined, `min=false` returns `i64 -1` (maximum representable value) and `min=true` returns 0.

Clang emits `llvm.objectsize` for `__builtin_object_size` and for buffer overflow checks in the `_FORTIFY_SOURCE` hardening mechanism. The `ObjectSizeOffsetVisitor` in [`llvm/lib/Analysis/MemoryBuiltins.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Analysis/MemoryBuiltins.cpp) performs the static analysis.

---

## 24.13 Exception Handling Intrinsics

LLVM supports three exception handling models, each with its own intrinsics. The full treatment is in Chapter 26; this section documents the IR-level intrinsics.

### 24.13.1 Type ID Intrinsic

```llvm
declare i32 @llvm.eh.typeid.for.p0(ptr %type_info)
```

In Itanium-style C++ exception handling, each `catch` clause is identified by a pointer to the `std::type_info` object for the caught type. `llvm.eh.typeid.for` returns the integer type ID that the personality function uses to match against the `typeinfo` entries in the landing pad. This intrinsic is emitted in every `catch` block and used in the dispatch chain:

```llvm
; In landing pad
%lpad = landingpad {ptr, i32} catch ptr @_ZTIi   ; catch (int&)
  %exn = extractvalue {ptr, i32} %lpad, 0
  %sel = extractvalue {ptr, i32} %lpad, 1
  %tid = call i32 @llvm.eh.typeid.for.p0(ptr @_ZTIi)
  %match = icmp eq i32 %sel, %tid
  br i1 %match, label %catch.int, label %resume
```

### 24.13.2 Setjmp/Longjmp EH

For targets that use `setjmp`/`longjmp`-based exception handling (historically MIPS, some bare-metal targets), LLVM provides:

```llvm
declare i32 @llvm.eh.sjlj.setjmp(ptr %jbuf)
declare void @llvm.eh.sjlj.longjmp(ptr %jbuf)
declare void @llvm.eh.sjlj.lsda()
declare void @llvm.eh.sjlj.callsite(i32 immarg)
declare void @llvm.eh.sjlj.functioncontext(ptr)
```

The SjLj EH pass in [`llvm/lib/CodeGen/SjLjEHPrepare.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/SjLjEHPrepare.cpp) transforms Itanium-style EH IR into the setjmp/longjmp form when the target's EH model requires it.

### 24.13.3 WebAssembly Exception Handling

WebAssembly has a native exception proposal with its own throw/catch model:

```llvm
declare void @llvm.wasm.throw(i32 immarg %tag, ptr %exn_ref)
declare ptr @llvm.wasm.get.exception(token)
declare i32 @llvm.wasm.get.ehselector(token)
```

---

## 24.14 Debugging and Profiling Intrinsics

### 24.14.1 Debug Variable Intrinsics

The debug variable intrinsics attach source-level variable information to SSA values. As of LLVM 16, the preferred representation uses `llvm.dbg.value` and `llvm.dbg.assign`; the older `llvm.dbg.declare` persists for alloca-backed variables:

```llvm
; Declare: associates a source variable with an alloca
declare void @llvm.dbg.declare(
    metadata %alloca,       ; MDValue wrapping the alloca
    metadata %variable,     ; DILocalVariable node
    metadata %expression)   ; DIExpression (offset/deref chain)

; Value: associates a source variable with an SSA value
declare void @llvm.dbg.value(
    metadata %value,        ; MDValue wrapping the SSA value
    metadata %variable,
    metadata %expression)

; Assign: combines value + memory assignment (RemoveDIs model)
declare void @llvm.dbg.assign(
    metadata %value,
    metadata %variable,
    metadata %value_expr,
    metadata %address,      ; alloca being written
    metadata %addr_expr,
    metadata %di_location)

; Label: marks a source-level label location
declare void @llvm.dbg.label(metadata %label)   ; DILabel
```

`llvm.dbg.assign` is the cornerstone of the RemoveDIs infrastructure (LLVM 17+), which moves debug variable information from intrinsics into `DbgRecord` metadata attached directly to instructions. The `DbgAssignInst` wrapper class provides `getValue()`, `getAddress()`, `getExpression()`, and `getAddressExpression()` accessors. See Chapter 22 for the complete debug metadata reference.

### 24.14.2 Profiling Intrinsics

Instrumentation-based PGO uses three intrinsics:

```llvm
; Increment counter for basic block profiling
declare void @llvm.instrprof.increment(
    ptr %name,          ; mangled function name as constant string
    i64 immarg %hash,   ; module-level hash for ABI stability
    i32 immarg %num_counters,
    i32 immarg %index)

; Increment by a custom step (for branch weights)
declare void @llvm.instrprof.increment.step(
    ptr %name, i64 immarg %hash, i32 immarg %num_counters,
    i32 immarg %index, i64 %step)

; Value profiling (indirect call targets, memory intrinsics sizes)
declare void @llvm.instrprof.value.profile(
    ptr %name, i64 immarg %hash, i64 %value,
    i32 immarg %value_kind, i32 immarg %index)

; MC/DC coverage bitmap update
declare void @llvm.instrprof.mcdc.tvbitmap.update(
    ptr %name, i64 immarg %hash, i32 %bitmap_idx, ptr %mcdc_ptr)
```

Clang's `-fprofile-instrument=clang` mode inserts these intrinsics. The `InstrProfiling` pass in [`llvm/lib/Transforms/Instrumentation/InstrProfiling.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Instrumentation/InstrProfiling.cpp) lowers them to actual counter loads/stores against the profiling data section (`.llvm_prof_cnts`).

### 24.14.3 Pseudo-Probe Intrinsics

Pseudo-probes decouple profiling granularity from basic block boundaries, enabling sample-based PGO to track inline contexts precisely:

```llvm
declare void @llvm.pseudoprobe(
    i64 immarg %guid,     ; function GUID
    i64 immarg %index,    ; probe index within function
    i32 immarg %type,     ; 0=block, 1=indirect-call
    i64 immarg %attr)     ; reserved
```

The `PseudoProbeInserter` pass distributes pseudo-probes throughout the function at a granularity that survives inlining. When a probe is inlined into a caller, the probe carries the full inline context (via `!DILocation` with an `inlined_at` chain), enabling SampleFDO to attribute samples to the correct inline instance. The `PseudoProbeInst` wrapper class provides `getGuid()` and `getIndex()`. See [`llvm/lib/Transforms/IPO/SampleProfile.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/IPO/SampleProfile.cpp) for the SampleFDO pipeline.

---

## 24.15 Coroutine Intrinsics

LLVM coroutines are implemented entirely through intrinsics in IR. The full treatment is in Chapter 27; here the key intrinsics are catalogued for reference.

```llvm
; Coroutine identifier — establishes the coroutine frame protocol
declare token @llvm.coro.id(
    i32 %align,        ; alignment of coroutine frame
    ptr %promise,      ; promise object pointer or null
    ptr %func,         ; the function pointer to the coroutine itself
    ptr %properties)   ; null or pointer to a coroutine properties vector

; Test whether allocation is needed
declare i1 @llvm.coro.alloc(token %id)

; Mark the start of the coroutine body; returns the frame pointer
declare ptr @llvm.coro.begin(token %id, ptr %frame_ptr)

; Suspend point — returns 0 (normal suspend), 1 (final suspend), or -1 (destroy)
declare i8 @llvm.coro.suspend(token %save_token, i1 %final)

; Save the suspension point (pairs with coro.suspend)
declare token @llvm.coro.save(ptr %handle)

; Resumption and destruction entry points
declare void @llvm.coro.resume(ptr %handle)
declare void @llvm.coro.destroy(ptr %handle)

; Free the coroutine frame
declare ptr @llvm.coro.free(token %id, ptr %frame_ptr)

; Mark the coroutine end point
declare i1 @llvm.coro.end(ptr %handle, i1 %unwind, token %ret_token)
```

The `CoroSplit` pass in [`llvm/lib/Transforms/Coroutines/CoroSplit.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Coroutines/CoroSplit.cpp) transforms the single coroutine function containing these intrinsics into three separate functions: the ramp function (initial call), the resume function, and the destroy function. The split is driven entirely by the suspend points marked by `llvm.coro.suspend`.

---

## 24.16 Target-Specific Intrinsics

### 24.16.1 Naming Convention and Organization

Target-specific intrinsics follow the naming convention `llvm.<target>.*`. They are declared in per-target TableGen files:

| Target | File | Approximate count (LLVM 22) |
|--------|------|-----------------------------|
| x86 | `IntrinsicsX86.td` | ~2700 definitions |
| AArch64 | `IntrinsicsAArch64.td` | ~866 definitions |
| RISC-V | `IntrinsicsRISCV.td` | ~30 (+extensions) |
| AMDGPU | `IntrinsicsAMDGPU.td` | ~700 definitions |
| NVPTX | `IntrinsicsNVVM.td` | ~800 definitions |
| ARM (32-bit) | `IntrinsicsARM.td` | ~400 definitions |
| PowerPC | `IntrinsicsPowerPC.td` | ~600 definitions |

The large x86 count reflects the depth of Intel's intrinsic API: each AVX-512 variant (128/256/512-bit, masked, unmasked, rounding-control) generates a separate intrinsic. For example, three widths of `vpermi2d` produce three distinct `llvm.x86.avx512.vpermi2var.d.*` intrinsics.

### 24.16.2 x86 Intrinsics

```llvm
; SSE reciprocal square root (approximate)
declare <4 x float> @llvm.x86.sse.rsqrt.ss(<4 x float>)

; Flush cache line (CLFLUSH)
declare void @llvm.x86.sse2.clflush(ptr)

; AES encryption round
declare <2 x i64> @llvm.x86.aesni.aesenc(<2 x i64>, <2 x i64>)

; AVX-512 saturating pack
declare <16 x i16> @llvm.x86.avx512.packssdw.512(<16 x i32>, <16 x i32>)
```

The Clang header `<immintrin.h>` maps Intel's C intrinsic API (`_mm_rsqrt_ss`, `_mm_clflush`, etc.) to these LLVM intrinsics via `__builtin_ia32_*` builtins. The x86 backend's `X86ISelLowering.cpp` contains the lowering table mapping each intrinsic to its machine instruction.

### 24.16.3 AArch64 Intrinsics

```llvm
; SVE predicated load (first-faulting)
declare <vscale x 4 x i32> @llvm.aarch64.sve.ldff1.nxv4i32(
    <vscale x 4 x i1> %pred, ptr %base)

; SVE predicated add
declare <vscale x 4 x i32> @llvm.aarch64.sve.add.nxv4i32(
    <vscale x 4 x i1> %pred,
    <vscale x 4 x i32> %op1,
    <vscale x 4 x i32> %op2)

; NEON polynomial multiply
declare <8 x i8> @llvm.aarch64.neon.pmul.v8i8(<8 x i8>, <8 x i8>)

; Pointer authentication (AUTIA)
declare i64 @llvm.ptrauth.auth(i64 %val, i32 immarg %key, i64 %discriminator)
```

The AArch64 SVE intrinsics follow the ACLE (AArch64 C Language Extensions) specification. Clang maps ACLE built-in functions to these intrinsics, and ARM's Compiler SDK provides the mapping table.

### 24.16.4 RISC-V Vector Intrinsics

```llvm
; RISC-V V extension: vadd.vv (vector-vector add)
declare <vscale x 4 x i32> @llvm.riscv.vadd.vv.nxv4i32(
    <vscale x 4 x i32> %passthru,
    <vscale x 4 x i32> %op1,
    <vscale x 4 x i32> %op2,
    i64 %vl)    ; vector length (runtime)

; RISC-V vector load
declare <vscale x 4 x i32> @llvm.riscv.vle32.v.nxv4i32(
    <vscale x 4 x i32> %passthru,
    ptr %base,
    i64 %vl)
```

RVV intrinsics pass the vector length `vl` as an explicit argument because RISC-V's `vsetvli` instruction is programmer-managed. The RISC-V backend's `RISCVISelLowering.cpp` maps these to machine instructions and inserts `vsetvli` as needed.

### 24.16.5 The IntrinsicInfo and getTgtMemIntrinsic Machinery

When a target-specific intrinsic touches memory, the backend needs to know its aliasing properties for the instruction scheduler and MemorySSA. The `TargetLowering::getTgtMemIntrinsic` virtual method fills an `IntrinsicInfo` struct describing the operation:

```cpp
// From llvm/include/llvm/CodeGen/TargetLowering.h
struct IntrinsicInfo {
  unsigned     opc;                    // target opcode
  EVT          memVT;                  // memory value type
  PointerUnion<const Value *, const PseudoSourceValue *> ptrVal;
  std::optional<unsigned> fallbackAddressSpace;
  int          offset = 0;
  uint64_t     size = 0;
  MaybeAlign   align = Align(1);
  MachineMemOperand::Flags flags = MachineMemOperand::MONone;
  AtomicOrdering order = AtomicOrdering::NotAtomic;
};

virtual bool getTgtMemIntrinsic(IntrinsicInfo &Info,
                                 const CallBase &I,
                                 MachineFunction &MF,
                                 unsigned Intrinsic) const;
```

Each target backend overrides this method and populates `Info` for its memory-touching intrinsics. The `SelectionDAGBuilder` calls `getTgtMemIntrinsic` during instruction selection to create a `MemIntrinsicSDNode` with the correct `MachineMemOperand`, which is then used by the scheduler to preserve memory ordering.

---

## 24.17 Miscellaneous Intrinsics

### 24.17.1 Assume

```llvm
declare void @llvm.assume(i1 %cond)
```

`llvm.assume` informs the optimizer that `%cond` is true at the program point where the intrinsic appears. It is not a runtime check; it is an optimizer hint that generates no code. When `%cond` is false at runtime, the behavior is undefined. Clang emits `llvm.assume` for `__builtin_assume`, `__assume` (MSVC), `assert()` in `-DNDEBUG` mode, and C++ `[[assume(expr)]]` (C++23). The `AssumptionCache` in [`llvm/lib/Analysis/AssumptionCache.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Analysis/AssumptionCache.cpp) collects all `llvm.assume` calls in a function and makes them available to passes through a per-function cache.

### 24.17.2 Trap and Debug Trap

```llvm
declare void @llvm.trap()         ; unconditional abort; noreturn + cold
declare void @llvm.debugtrap()    ; breakpoint; may be removed in non-debug builds
```

`llvm.trap` generates an illegal instruction or an explicit trap instruction (`UD2` on x86, `BRK #0` on AArch64) intended to terminate the program with a signal. It carries `IntrNoReturn` and `IntrCold` properties. `llvm.debugtrap` generates a software breakpoint (`INT3` on x86, `BKPT` on ARM); in optimized builds without a debugger attached, it may be lowered to a no-op or a trap.

### 24.17.3 sideeffect and convergent

```llvm
declare void @llvm.sideeffect()
```

`llvm.sideeffect` asserts that the program point has an observable side effect, preventing the optimizer from reordering or eliminating surrounding code. It is used in `asm volatile("")` (as a memory barrier) and by sanitizers that need to prevent the optimizer from eliminating instrumented code. The intrinsic has `IntrInaccessibleMemOnly`, making it a write to the hidden state model.

### 24.17.4 Freeze

`freeze` is not strictly an intrinsic — it is an instruction opcode — but it is closely related to the `llvm.assume` and `llvm.sideeffect` infrastructure. `freeze %x` returns `%x` if it is not `poison` or `undef`, and returns an arbitrary (but fixed) value otherwise. It is inserted by InstCombine when the optimizer cannot prove that a value is well-defined but a downstream instruction requires it.

---

## 24.18 Working with Intrinsics Programmatically

### 24.18.1 Querying Intrinsic ID

```cpp
// From a CallInst, check if it's a specific intrinsic
if (auto *II = dyn_cast<IntrinsicInst>(&I)) {
  if (II->getIntrinsicID() == Intrinsic::memcpy) {
    auto *MCI = cast<MemCpyInst>(II);
    Value *Dest = MCI->getDest();
    Value *Src  = MCI->getSource();
    Value *Len  = MCI->getLength();
  }
}

// From a Function, get the intrinsic ID
Intrinsic::ID IID = F.getIntrinsicID();
bool IsTarget = Intrinsic::isTargetIntrinsic(IID);
```

### 24.18.2 Creating Intrinsic Calls

```cpp
// Build a call to llvm.sqrt.f64
Type *DoubleTy = Type::getDoubleTy(Context);
Function *SqrtFn = Intrinsic::getOrInsertDeclaration(
    M, Intrinsic::sqrt, {DoubleTy});
CallInst *CI = Builder.CreateCall(SqrtFn, {ArgVal});

// Build a call to llvm.memcpy.p0.p0.i64
Function *MemcpyFn = Intrinsic::getOrInsertDeclaration(
    M, Intrinsic::memcpy,
    {Builder.getPtrTy(), Builder.getPtrTy(), Builder.getInt64Ty()});
Builder.CreateCall(MemcpyFn,
    {DstPtr, SrcPtr, Builder.getInt64(Size),
     Builder.getInt1(false) /*non-volatile*/});
```

The function `Intrinsic::getOrInsertDeclaration` (formerly `Intrinsic::getDeclaration` prior to LLVM 20) looks up the intrinsic declaration in the module by name, creating it if absent. The attribute list for the declaration is generated from the TableGen properties, so passes do not need to manually specify attributes when creating intrinsic calls.

### 24.18.3 Matching Intrinsics in Patterns

The `PatternMatch.h` infrastructure supports intrinsic matching:

```cpp
// Match any call to llvm.sqrt
using namespace PatternMatch;
Value *SqrtArg;
if (match(V, m_Intrinsic<Intrinsic::sqrt>(m_Value(SqrtArg)))) {
  // V is a call to llvm.sqrt; SqrtArg is the argument
}

// Match llvm.fma(a, b, c)
Value *A, *B, *C;
if (match(V, m_Intrinsic<Intrinsic::fma>(
              m_Value(A), m_Value(B), m_Value(C)))) {
  // ...
}
```

---

## 24.19 Chapter Summary

- Intrinsics are `declare`-only `llvm.*`-named pseudo-functions that allow IR to express operations — memory bulk transfers, math, bit manipulation, overflow detection, vector reductions, masked operations, debug information, profiling, exceptions, coroutines, and target-specific instructions — beyond the core opcode set.
- The `IntrinsicInst` class hierarchy in `IntrinsicInst.h` provides typed wrappers (`MemCpyInst`, `WithOverflowInst`, `DbgValueInst`, etc.) that support `isa<>`/`cast<>` and typed accessor methods.
- Intrinsic attributes — `memory(none)`, `speculatable`, `willreturn`, `nosync`, `nocallback` — are generated from TableGen `IntrinsicProperty` records and precisely constrain what the optimizer may assume about the intrinsic's effect on memory, control flow, and synchronization.
- Memory intrinsics (`llvm.memcpy`, `llvm.memmove`, `llvm.memset`) encode alignment via parameter attributes; volatile behavior via an `i1 immarg` flag; and have element-unordered atomic variants for Java-style array copy semantics.
- Math intrinsics carry `speculatable` and `memory(none)`, enabling hoisting out of loops. Fast-math flags (`afn`, `nnan`, `reassoc`) on intrinsic calls relax IEEE 754 constraints, enabling hardware approximations and tree reductions.
- Overflow-checking intrinsics return `{i<N>, i1}` struct types and lower to flag-setting arithmetic. Saturating intrinsics clamp rather than wrap, mapping to native DSP and SIMD saturation instructions.
- Vector reduction intrinsics abstract horizontal operations; floating-point reductions take an explicit accumulator for sequential semantics in the absence of `reassoc`.
- Masked intrinsics (`llvm.masked.load`, `llvm.masked.store`, `llvm.masked.gather`, `llvm.masked.scatter`, `llvm.masked.expandload`, `llvm.masked.compressstore`) are the canonical vectorizer output for conditionally-executed vector memory accesses.
- Scalable vector intrinsics (`llvm.vscale`, `llvm.stepvector`, `llvm.vector.insert`, `llvm.vector.extract`) bridge between fixed-length operations and SVE/RVV scalable vector types.
- Lifetime and invariant intrinsics convey liveness and immutability to the stack coloring pass and alias analyses. Invariant group intrinsics handle C++ placement-new type changes.
- Debug intrinsics (`llvm.dbg.declare`, `llvm.dbg.value`, `llvm.dbg.assign`, `llvm.dbg.label`) carry source-level variable location information; the RemoveDIs infrastructure (LLVM 17+) migrates toward `DbgRecord` metadata, but the intrinsic form is generated and consumed in LLVM 22 for compatibility.
- Target-specific intrinsics follow the `llvm.<target>.*` convention and are defined in per-target TableGen files; the backend's `getTgtMemIntrinsic` virtual method informs the instruction scheduler of their memory effects.

---

*Cross-references:*
- [Chapter 19 — Instructions I: Arithmetic and Memory](ch19-instructions-arithmetic-and-memory.md) — core opcode set that intrinsics extend
- [Chapter 20 — Instructions II: Control Flow and Aggregates](ch20-instructions-control-flow-and-aggregates.md) — `extractvalue` for overflow struct results; `va_arg` instruction
- [Chapter 22 — Metadata and Debug Info](ch22-metadata-and-debug-info.md) — DILocalVariable, DIExpression, and the metadata nodes consumed by debug intrinsics
- [Chapter 23 — Attributes, Calling Conventions, and the ABI](ch23-attributes-calling-conventions-abi.md) — `memory(...)`, `speculatable`, `willreturn` attribute semantics
- [Chapter 26 — Exception Handling](ch26-exception-handling.md) — landing pads, personality functions, and the EH intrinsics in context
- [Chapter 27 — Coroutines and Atomics](ch27-coroutines-and-atomics.md) — full treatment of `llvm.coro.*` intrinsics and atomic memory ordering
- [Chapter 59 — The New Pass Manager](../part-10-analysis-middle-end/ch59-pass-manager.md) — `AssumptionCache` and `MemorySSA` passes that consume intrinsic annotations
- [Chapter 64 — Vectorization](../part-10-analysis-middle-end/ch64-vectorization.md) — emission of masked, reduction, and scalable vector intrinsics by the SLP and loop vectorizers

*Reference links:*
- [`llvm/include/llvm/IR/Intrinsics.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/Intrinsics.td) — master intrinsic TableGen definitions
- [`llvm/include/llvm/IR/IntrinsicInst.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/IntrinsicInst.h) — C++ wrapper class hierarchy
- [`llvm/include/llvm/IR/Intrinsics.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/Intrinsics.h) — `Intrinsic::ID`, `getName`, `getType`, `getOrInsertDeclaration` API
- [`llvm/lib/Analysis/MemoryBuiltins.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Analysis/MemoryBuiltins.cpp) — `llvm.objectsize` analysis and `ObjectSizeOffsetVisitor`
- [`llvm/lib/CodeGen/StackColoring.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/StackColoring.cpp) — stack slot merging driven by lifetime markers
- [`llvm/lib/Transforms/Coroutines/CoroSplit.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Coroutines/CoroSplit.cpp) — coroutine splitting at `llvm.coro.suspend` points
- [`llvm/lib/Transforms/Instrumentation/InstrProfiling.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Instrumentation/InstrProfiling.cpp) — lowering of `llvm.instrprof.*` intrinsics
- [LLVM Language Reference Manual — Intrinsic Functions](https://llvm.org/docs/LangRef.html#intrinsic-functions) — canonical authoritative specification


---

@copyright jreuben11
