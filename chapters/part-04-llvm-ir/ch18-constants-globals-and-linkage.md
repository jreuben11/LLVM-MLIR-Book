# Chapter 18 — Constants, Globals, and Linkage

*Part IV — LLVM IR*

Constants and globals are the fixed scaffolding of an LLVM module: the named storage, the addresses the linker resolves, and the compile-time values that survive into the final binary unchanged. Getting them right is not merely a matter of syntax. Each linkage type carries an ABI contract with the linker; each thread-local storage model implies a specific sequence of register-relative loads; each comdat selection kind corresponds to a distinct linker behavior that differs between ELF and COFF. This chapter works through every level of that hierarchy — from the `APInt`-backed integer constant to the multi-file linkage interplay that makes C++ inline functions and vtables work correctly across translation units.

We begin with the constant hierarchy in memory (`ConstantExpr`, `ConstantData`, and the special-case singletons), move to global variables and their properties, cover aliases and indirect functions, explain the four ELF TLS models, catalogue every linkage type, and finish with comdats and the Windows DLL visibility attributes. Throughout, all IR examples are verified against LLVM 22.

---

## 1. Constant Expressions and ConstantData

### The `Value` Hierarchy for Constants

In the LLVM C++ API every object that can appear as an operand derives from `llvm::Value`. Constants occupy a subtree:

```
Value
└── Constant
    ├── ConstantData           (leaf, no operands referencing other Values)
    │   ├── ConstantInt
    │   ├── ConstantFP
    │   ├── ConstantAggregateZero
    │   ├── ConstantDataArray
    │   ├── ConstantDataVector
    │   ├── ConstantPointerNull
    │   └── UndefValue / PoisonValue
    ├── ConstantAggregate      (operands are other Constants)
    │   ├── ConstantArray
    │   ├── ConstantStruct
    │   └── ConstantVector
    ├── ConstantExpr           (constant-folded expression, no parent BB)
    └── GlobalValue
        ├── GlobalVariable
        ├── GlobalAlias
        ├── GlobalIFunc
        └── Function
```

The key distinction between `ConstantData` and `ConstantExpr` is that `ConstantData` leaves carry their value entirely within themselves — they do not reference any other `Value`. `ConstantExpr` nodes, by contrast, are expression trees built over other constants (including globals), and therein lies their complexity.

### `ConstantExpr`: Constant-Folded Expressions

A `ConstantExpr` is an expression whose result is known at compile time, expressed as an LLVM opcode applied to constant operands. Unlike an instruction, it has no parent basic block and no associated `IRBuilder` insertion point. It appears wherever a `Constant*` is required — most commonly as the initializer of a global variable.

The historical motivation was to allow the IR to carry non-trivial constant initializers (e.g., the address of a global plus an offset) without introducing a synthetic function or constructor. The practical consequence was that many optimizer passes had to handle both the instruction and the constant-expression form of the same operation, leading to duplicated logic and pass ordering hazards: a pass that rewrites `add` instructions cannot easily rewrite a `ConstantExpr add` embedded inside a global initializer or function attribute, because constant expressions are folded eagerly during construction and may be shared across multiple use sites.

**LLVM 22 surviving ConstantExpr opcodes.** The LLVM project has been systematically removing constant expression opcodes over multiple release cycles. As of LLVM 22, the surviving opcodes that `llvm-as` still accepts are:

| Opcode | Typical use |
|--------|-------------|
| `getelementptr` | Pointer arithmetic into a global; the most common CE |
| `bitcast` | Reinterpreting a pointer or aggregate constant |
| `inttoptr` | Embedding a literal address (e.g., MMIO base address in embedded code) |
| `ptrtoint` | Encoding a global's address as an integer constant |
| `addrspacecast` | Promoting/demoting a global to a different address space |
| `add` | Adding integer constants (rarely useful; constant folding handles it) |
| `sub` | Subtracting integer constants |
| `xor` | Bitwise XOR of integer constants |
| `trunc` | Truncating an integer constant to a narrower type |

Opcodes that have been **removed** and now cause a parse error include: `mul`, `shl`, `lshr`, `ashr`, `and`, `or`, `sdiv`, `udiv`, `srem`, `urem`, `sext`, `zext`, `fptrunc`, `fpext`, `fneg`, `fadd`, `fsub`, `fmul`, `fdiv`, `icmp`, `fcmp`, `select`, `extractelement`, `insertelement`, `shufflevector`. The `llvm-as` error message is unambiguous: `"mul constexprs are no longer supported"`.

**Practical consequence for pass authors.** If your pass pattern-matches a `ConstantExpr` operand and attempts to fold it into something simpler, you must subsequently call `ConstantFoldInstruction()` or `llvm::InstSimplifyPass` rather than reconstructing a new `ConstantExpr` with a removed opcode. The recommended migration path is: if you need a constant-folded result in a function body, emit the equivalent instruction in the function entry block; the `instcombine` and `early-cse` passes will fold it at their first opportunity.

**The `ConstantFold` / `InstSimplify` split.** The ConstantExpr deprecation sharpens a fundamental architectural boundary in LLVM's folding infrastructure. `ConstantFold*` functions (in [`llvm/lib/Analysis/ConstantFolding.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Analysis/ConstantFolding.cpp)) operate exclusively on `Constant*` operands and produce a `Constant*` result; they require no basic block context and no data-flow information. `InstSimplify` (in [`llvm/lib/Analysis/InstructionSimplify.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Analysis/InstructionSimplify.cpp)) operates on `Value*` operands, which may include non-constant SSA values, and can exploit data-flow facts (range information, known bits, implied conditions). By moving constant-expression uses toward the instruction domain, LLVM consolidates all simplification logic in `InstSimplify`, eliminating the duplication where a pass previously had to handle both `icmp (ConstantExpr ...)` and `icmp (%reg, ...)` separately. The `ConstantFold` layer is now reserved for true compile-time constants: global initializers and attributes that structurally require a `Constant*`.

**Creating constant expressions via the C++ API.**

```cpp
// getelementptr into a global array
// @arr = global [8 x i32] zeroinitializer
GlobalVariable *GV = ...; // points to @arr
Type *I32 = Type::getInt32Ty(Ctx);
Type *I64 = Type::getInt64Ty(Ctx);
// ptr getelementptr (i32, ptr @arr, i64 3)
Constant *GEP = ConstantExpr::getGetElementPtr(
    I32, GV,
    ArrayRef<Constant *>{ConstantInt::get(I64, 3)},
    /*InBounds=*/true);

// bitcast (ptr @arr to ptr)  — identity in opaque-pointer world,
// but still accepted for legacy reasons
Constant *BC = ConstantExpr::getBitCast(GV, PointerType::getUnqual(Ctx));
```

In verified IR:

```llvm
@arr = global [8 x i32] zeroinitializer

; GEP constant expression: pointer to element 3 of @arr
@elem3_ptr = global ptr getelementptr (i32, ptr @arr, i64 3)

; ptrtoint: encode @arr's address as a 64-bit integer
@arr_addr = global i64 ptrtoint (ptr @arr to i64)

; inttoptr: hard-coded MMIO address (embedded systems)
@uart_base = global ptr inttoptr (i64 1073741824 to ptr)
```

### `ConstantInt` and `APInt`

`ConstantInt` represents integer constants of any width. Internally it wraps an `APInt` — the arbitrary-precision integer class defined in [`llvm/include/llvm/ADT/APInt.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ADT/APInt.h). `APInt` stores its value in a small-buffer-optimized word array (one `uint64_t` for widths ≤ 64, heap-allocated otherwise) and carries explicit bit-width semantics: two's complement arithmetic wraps at the declared width, and `APInt` provides both signed and unsigned comparison, division, and shift operations.

Common construction patterns:

```cpp
LLVMContext &Ctx = M.getContext();
Type *I32 = Type::getInt32Ty(Ctx);
Type *I1  = Type::getInt1Ty(Ctx);

// Decimal constant
Constant *C42 = ConstantInt::get(I32, 42);

// From APInt directly (e.g., a 128-bit constant)
APInt BigVal(128, {0xDEADBEEFCAFEBABEULL, 0x0102030405060708ULL}, /*isSigned=*/false);
Constant *C128 = ConstantInt::get(Ctx, BigVal);

// Boolean shorthands
Constant *True  = ConstantInt::getTrue(Ctx);   // i1 true
Constant *False = ConstantInt::getFalse(Ctx);  // i1 false
```

In IR, integer constants are written with their type prefix:

```llvm
@answer = global i32 42
@negone = global i64 -1
@bool_t = global i1 true
@wide    = global i128 340282366920938463463374607431768211455  ; 2^128 - 1
```

### `ConstantFP` and `APFloat`

`ConstantFP` wraps an `APFloat` ([`llvm/include/llvm/ADT/APFloat.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ADT/APFloat.h)), which models IEEE 754 semantics for `half`, `bfloat16`, `float`, `double`, `fp80` (x87 extended precision), and `fp128`. `APFloat` tracks the semantics object (`fltSemantics`) so that operations like rounding and comparison behave correctly for each format.

```cpp
Constant *Pi  = ConstantFP::get(Type::getDoubleTy(Ctx), 3.14159265358979323846);
Constant *Inf = ConstantFP::getInfinity(Type::getFloatTy(Ctx));
Constant *Nan = ConstantFP::getNaN(Type::getDoubleTy(Ctx));
```

In IR, LLVM uses hexadecimal floating-point notation to avoid decimal rounding errors:

```llvm
@pi_double = global double 0x400921FB54442D18
@pi_float  = global float  0x400921FB60000000
@inf_f32   = global float  0x7FF0000000000000
@qnan_f64  = global double 0x7FF8000000000000
```

The `double 3.14159` form is also accepted by `llvm-as` but is discouraged in hand-written IR because the decimal-to-binary conversion may not round-trip.

### Aggregate Constants

**`ConstantAggregateZero`** is the representation of the `zeroinitializer` keyword. It applies to any aggregate type — arrays, structs, vectors — and signals that all bytes of the storage are zero. The linker and object file writer place it in the BSS segment, costing no file space. Construction: `ConstantAggregateZero::get(AggType)`.

```llvm
@zero_arr    = global [1024 x i8]   zeroinitializer
@zero_struct = global {i32, double} zeroinitializer
@zero_vec    = global <8 x i32>     zeroinitializer
```

**`ConstantDataArray`** represents arrays whose element type is a primitive integer or floating-point type (no pointer or aggregate elements). It is backed by a compact raw data buffer rather than a vector of `Constant*` pointers, making it significantly cheaper in memory for large constant tables and string literals. `ConstantDataArray::getString(Ctx, "Hello, World!", /*AddNull=*/true)` produces a `[14 x i8]` constant with the null terminator included.

```llvm
@hello = private constant [14 x i8] c"Hello, World!\00"
@lut   = constant [8 x i32] [i32 0, i32 1, i32 1, i32 2,
                               i32 1, i32 2, i32 2, i32 3]
```

**`ConstantDataVector`** is the vector analogue — a compact representation for `<N x i8>`, `<N x i32>`, `<N x float>`, etc.:

```llvm
@simd_mask = constant <8 x i32>   <i32 1, i32 0, i32 1, i32 0,
                                    i32 1, i32 0, i32 1, i32 0>
@bias_vec  = constant <4 x float> <float 0x3FC999999A000000,
                                    float 0x3FC999999A000000,
                                    float 0x3FC999999A000000,
                                    float 0x3FC999999A000000>
```

**`ConstantArray`**, **`ConstantStruct`**, and **`ConstantVector`** handle heterogeneous aggregates whose elements cannot all be packed into a `ConstantDataArray` (because they involve pointer elements or other non-primitive constants):

```llvm
; Struct with pointer member — must use ConstantStruct, not ConstantDataArray
@dispatch_entry = constant {i32, ptr} {i32 7, ptr @some_function}

; Mixed-type nested struct
@header = constant {i32, [4 x i8], ptr} {
    i32 0xDEADBEEF,
    [4 x i8] c"\01\02\03\04",
    ptr null
}
```

---

## 2. Special Constants: `undef`, `poison`, and `zeroinitializer`

### `UndefValue`

`undef` is a constant that can take *any* bit pattern. It is used to represent uninitialized registers and memory locations, and to tell the optimizer that the exact value does not matter — enabling transformations like eliminating a redundant load into an unused variable. In the C++ API: `UndefValue::get(T)`.

```llvm
; Uninitialized i32 — any 32-bit value
@unset = global i32 undef

; Undef in a function: the optimizer may assume any value
define i32 @undef_user() {
  %x = add i32 undef, 5   ; result is undef (propagates)
  ret i32 %x
}
```

`undef` has a subtle semantic: *each use may independently realize a different value*. Two reads of the same `undef` register are permitted to return different results. This is weaker than C's undefined behavior — it does not allow the compiler to assume an inconsistent universe — but it does allow reordering and widening of loads without violating the IR semantics.

### `PoisonValue` and the `freeze` Instruction

`poison` is a stronger variant introduced to replace `undef` for cases where the language actually has undefined behavior. The difference:

- `undef`: any bit pattern; the runtime can "choose" a value per use, but no UB is implied.
- `poison`: if a side-effecting instruction (e.g., `store`, `ret`, a branch condition) receives a `poison` operand, the behavior is undefined — the program is ill-formed, and the optimizer is permitted to assume it never happens.

The rationale is that `undef` was too weak to justify certain aggressive transformations (e.g., speculating an arithmetic operation past a branch required only that the result is `undef` if the branch is not taken, but the optimizer could not claim UB), while `poison` cleanly models the C/C++ contract "this value is never used in a way that would be observable."

Poison propagates through arithmetic: `add i32 %x, poison` is `poison`. The `freeze` instruction converts a `poison` or `undef` value into an arbitrary but *fixed* value — each `freeze` produces a value that is consistent across all uses:

```llvm
define i32 @freeze_example(i32 %x) {
entry:
  ; Suppose %x might be poison (e.g., from a nsw overflow upstream)
  %frozen = freeze i32 %x
  ; %frozen is now a normal i32: consistent across uses, no UB
  %doubled = mul i32 %frozen, 2
  ret i32 %doubled
}
```

The `freeze` instruction is essential for loop-idiom recognition and LICM passes that must hoist potentially poisoning expressions: after freezing, the hoisted computation is safe to execute speculatively.

In practice, `PoisonValue` is the correct model for:
- Results of `add nsw` when signed overflow occurs
- Results of `udiv` / `urem` by zero
- Out-of-bounds GEP when `inbounds` is asserted
- Load from a dead pointer in an unreachable block that gets speculated

The LLVM project is actively migrating existing `undef` uses to `poison` where the semantics allow it. New code should use `PoisonValue::get(T)` rather than `UndefValue::get(T)` for undefined-behavior modeling.

---

## 3. Global Variables: Declarations, Definitions, and Properties

### Syntax and Anatomy

A global variable declaration or definition in LLVM IR follows this grammar (simplified):

```
<name> = [linkage] [preemption] [visibility] [DLLStorage]
         [thread_local[(model)]] [unnamed_addr | local_unnamed_addr]
         [addrspace(N)] [global | constant] <type> [<initializer>]
         [, section "<name>"] [, comdat[(<name>)]]
         [, align <N>]
```

The three required keywords are `global` (mutable) or `constant` (read-only) and the element type. The name begins with `@`. All attributes to the left of `global`/`constant` are prefixed; `section`, `comdat`, and `align` are comma-separated suffixes.

```llvm
; Mutable global with default (external) linkage
@counter = global i32 0, align 4

; Private read-only string literal (local_unnamed_addr = address not significant)
@msg = private local_unnamed_addr constant [14 x i8] c"Hello, World!\00"

; Weakly defined mutable global (can be overridden)
@config_flags = weak global i32 0, align 4

; External declaration — no initializer, resolved by the linker
@errno_location = external global i32

; Global in a named section with explicit alignment
@perf_counter = global i64 0, section ".perf_data", align 64
```

### `isConstant()` and the `constant` Keyword

The `constant` keyword places the global in a read-only section (e.g., `.rodata` on Linux ELF, `__TEXT,__const` on Mach-O). Any store through a pointer to a `constant` global is undefined behavior. This is distinct from `const` in C: a C++ `const int x = 42` at file scope produces a `constant global` only when its address is not taken in a way that could alias a mutable pointer; the compiler is responsible for choosing correctly.

The optimizer exploits `isConstant()` to bypass alias analysis: a load from a `constant` global can be hoisted freely, speculated past stores, and CSE'd across call sites that might theoretically write to arbitrary memory.

### `hasInitializer()` vs. External Declarations

- `hasInitializer()` returns `true` if the global has an initializer (i.e., it is a *definition*). Definitions provide storage.
- A global without an initializer is a *declaration*, equivalent to `extern T x;` in C. It must be resolved by the linker. In IR, a declaration is written without a `= <initializer>` suffix.

```llvm
; Definition: provides storage
@g_defined = global i32 42

; Declaration: no storage in this module, resolved externally
@g_declared = external global i32
```

### `unnamed_addr` and `local_unnamed_addr`

Both attributes indicate that the address of the global is not significant to the program's semantics:

- `unnamed_addr`: the global may be merged with another constant of identical content, even if they have different addresses. Safe for pure data constants (string literals, lookup tables) that are never compared by address.
- `local_unnamed_addr`: weaker — the address is not significant *within the current module*, but may be significant to external code. Enables intra-module merging only.

Clang emits `private unnamed_addr constant` for most string literals.

### Address Space, Section, and Alignment

- `addrspace(N)` places the global in address space N. Address space 0 is the default; GPUs use higher-numbered address spaces for shared memory, constant memory, etc.
- `section ".name"` overrides the default ELF section; used for linker scripts, hardware-specific placement, and profiling data.
- `align N` forces the object to a specific alignment in bytes (must be a power of two). Misaligned accesses on strict-alignment architectures (e.g., RISC-V without the unaligned extension) are undefined behavior at the IR level.

### Visibility: `default`, `hidden`, and `protected`

Visibility controls whether and how a symbol appears in the dynamic symbol table of the object file. It is orthogonal to linkage (which governs the linker's merging rules) and directly determines whether `dso_local` is sound.

**`default`** (the default when omitted): the symbol is exported into the dynamic symbol table as a preemptable global symbol. Another DSO placed earlier in the `LD_PRELOAD` order can provide a definition that overrides this one — the linker must go through the PLT for every call to a `default`-visibility function, because it cannot know at compile time which definition will win. This is the semantics demanded by the POSIX `dlopen`/`LD_PRELOAD` interposition model.

**`hidden`**: the symbol is not placed in the `.dynsym` dynamic symbol table at all. It is invisible to the dynamic linker; no other DSO can take its address or call it directly. Because preemption is impossible, the compiler can emit a direct PC-relative call (no PLT) and a direct PC-relative load (no GOT) and assert `dso_local`. Clang emits `hidden` for `__attribute__((visibility("hidden")))` and for all symbols when compiling with `-fvisibility=hidden`.

**`protected`**: a compromise — the symbol appears in `.dynsym` so that other DSOs can take its *address* (e.g., store it in a callback table), but the definition in this DSO is not preemptable. The local definition always wins for calls within the DSO. Rarer than `hidden`; used when symbol interposition by address is needed but call interposition is undesirable. `dso_local` is sound with `protected` visibility.

```llvm
@default_sym   = global i32 0                    ; default visibility
@hidden_sym    = hidden global i32 0             ; not in .dynsym
@protected_sym = protected global i32 0          ; in .dynsym, not preemptable
```

The ELF visibility model is defined in the System V ABI. LLVM maps `default` → `STV_DEFAULT`, `hidden` → `STV_HIDDEN`, `protected` → `STV_PROTECTED` in the ELF symbol table.

---

## 4. Aliases and Indirect Functions

### `GlobalAlias`

A `GlobalAlias` defines an alternative name for an existing global or a constant expression derived from it:

```llvm
@original = global i32 0

; Simple alias: @alias refers to the same storage as @original
@alias = alias i32, ptr @original

; Alias to a function
define i32 @foo_impl(i32 %x) {
  ret i32 %x
}
@foo = alias i32 (i32), ptr @foo_impl
```

The syntax is `@name = [linkage] alias <value-type>, ptr @aliasee`. The value type is the pointee type of the alias — for a function alias it is the function type, for a data alias it is the element type. In the opaque-pointer model the `ptr` keyword always precedes the aliasee.

**Aliasee constraints.** The aliasee must be a `GlobalValue` or a `ConstantExpr` built from one. Common patterns include aliasing to a GEP into a global array (to expose a named pointer into the middle of a table) and aliasing with a `bitcast` for legacy type-punning. Alias chains — alias of an alias — are resolved transitively: each alias ultimately points to a `GlobalValue` definition.

**Linkage on aliases.** An alias has its own linkage, which governs the symbol's visibility in the object file. `weak` aliases are commonly used to provide default implementations of hook functions that user code can override:

```llvm
; Weak alias: user code can define @malloc_hook to override
define void @default_malloc_hook(ptr %p, i64 %sz) {
  ret void
}
@malloc_hook = weak alias void (ptr, i64), ptr @default_malloc_hook
```

### `GlobalIFunc`

An indirect function (`ifunc`) is an ELF-only feature that defers the selection of a function's actual address to a *resolver* function invoked at load time (before `main`). The runtime linker calls the resolver once and patches the GOT entry with the returned address; all subsequent calls go directly to the selected implementation without further indirection.

The canonical use case is CPU dispatch — choosing between an AVX-512 implementation and an SSE2 fallback at program start based on `cpuid`:

```llvm
; Implementations
define i32 @dot_product_avx512(<16 x float> %a, <16 x float> %b) {
  ; ... AVX-512 implementation ...
  ret i32 0
}
define i32 @dot_product_sse2(<4 x float> %a, <4 x float> %b) {
  ; ... SSE2 fallback ...
  ret i32 0
}

; Resolver: called once at load time, returns a function pointer
define ptr @resolve_dot_product() {
entry:
  ; In real code this would call __builtin_cpu_supports or cpuid
  ; Here simplified for illustration
  ret ptr @dot_product_sse2
}

; ifunc declaration: callers see @dot_product, get the resolved address
@dot_product = ifunc i32 (...), ptr @resolve_dot_product
```

The resolver must be a *definition* within the same module (LLVM verifies this). It must return `ptr` and take no arguments (regardless of the ifunc's own signature). The ifunc type is the function type that callers will use — the resolver type is always `ptr ()`.

`ifunc` is **not supported** on Mach-O or PE/COFF targets. The portable alternative on Apple platforms is `__attribute__((target_clones(...)))` which Clang lowers to a regular dispatch wrapper.

---

## 5. Thread-Local Storage: The Four ELF TLS Models

Thread-local variables (`__thread` in C, `thread_local` in C++) are allocated once per thread. On ELF targets, the ABI specifies four TLS access models that trade generality for performance. Choosing the wrong model either causes a crash (if the model is too restrictive) or wastes cycles (if the model is more conservative than necessary).

### The ELF TLS ABI Background

Each ELF module (executable or shared library) has a *TLS block* — a template copied for each new thread. The thread pointer (`%fs` on x86-64, `%tpidr_el0` on AArch64) points to a thread-control block (TCB); the TLS block for the main executable is at a known negative offset from the thread pointer. For dynamic shared libraries loaded after program start, the TLS block address must be looked up through `__tls_get_addr`.

### General Dynamic (GD) — Default

The most conservative model. Both the *module* (which DSO the variable lives in) and the *variable's offset within that module's TLS block* are unknown at compile time. The compiler emits two relocations — `R_X86_64_TLSGD` — and two calls to `__tls_get_addr`. This is required for:
- Variables in shared libraries that may be loaded via `dlopen` after program start.
- Any variable whose definition is not known at compile time.

```llvm
@errno_tls = thread_local global i32 0     ; GD (default — no qualifier needed)
```

Overhead: two `__tls_get_addr` calls per access sequence. In high-frequency code, GD TLS can dominate runtime.

### Local Dynamic (LD)

The module is unknown (TLS block must be looked up), but the variable's offset within the TLS block is *known at link time* (the variable is defined in the same module). One call to `__tls_get_addr` obtains the module's TLS base; subsequent accesses use constant offsets from that base.

LD is appropriate when a single function accesses multiple thread-local variables from the same DSO: the compiler can share one `__tls_get_addr` call and reach all variables by offset arithmetic. GCC and Clang automatically upgrade multiple GD accesses in a single function to LD during codegen if they are provably from the same module.

```llvm
@tls_ld = thread_local(localdynamic) global i32 0
```

### Initial Executable (IE)

The variable's module is the main executable or a shared library that is *guaranteed to be loaded at program start* (i.e., not via `dlopen`). Under IE, the variable's offset from the thread pointer is stored in the GOT and loaded once with `R_X86_64_GOTTPOFF`. There is no `__tls_get_addr` call — just a GOT load followed by a `%fs`-relative access.

IE is the most common model for shared libraries that are direct dependencies of the main executable (`NEEDED` entries in the dynamic section). The linker can relax GD relocations to IE during link time when it determines the variable's module is loaded initially.

```llvm
@tls_ie = thread_local(initialexec) global i32 0
```

### Local Executable (LE) — Fastest

Both the module and the variable's offset are known at link time: the variable is in the main executable itself, and the thread pointer offset is a link-time constant. The compiler emits a single instruction: `movl %fs:offset, %eax` (or the AArch64 equivalent using `%tpidr_el0`). No GOT, no function call.

LE is appropriate for `thread_local` variables in the main executable when compiling without `-fpic`/`-fPIC`, or when the programmer can assert the variable is never in a DSO.

```llvm
@tls_le = thread_local(localexec) global i32 0
```

### TLS Model Selection Summary

| Model (IR syntax) | Module known? | Offset known? | `__tls_get_addr` calls | GOT access | Cost |
|-------------------|--------------|---------------|------------------------|------------|------|
| `thread_local` — GD (default; no qualifier accepted) | No | No | 2 | Yes | Highest |
| `thread_local(localdynamic)` — LD | No | Yes | 1 (shared) | Yes | Medium-high |
| `thread_local(initialexec)` — IE | Yes | At link time | 0 | Yes (once) | Low |
| `thread_local(localexec)` — LE | Yes (main exe) | Yes | 0 | No | Lowest |

**When to choose each model.** Prefer LE for thread-local variables in non-PIC executables. Use IE for thread-locals in shared libraries that are always directly linked (not `dlopen`'d). Use LD when a function body accesses several thread-locals from the same DSO. Fall back to GD only for variables in DSOs that might be loaded dynamically.

**Platform support.** TLS models are ELF-specific. On Mach-O (Darwin), `thread_local` is implemented via a per-thread dictionary accessed through a stub in `__TEXT,__thread_starts`; LLVM maps all TLS models to a single Mach-O mechanism. On Windows, `__declspec(thread)` maps to the `.tls` section and is accessed through `TlsGetValue`/`TEB.ThreadLocalStoragePointer`; LLVM's PE/COFF backend ignores the ELF model qualifiers and emits a single Windows-compatible access sequence.

---

## 6. The Full Linkage Taxonomy

Linkage governs how the linker resolves multiple definitions of the same symbol and which symbols become visible in the object file. Every global value (variable, function, alias, ifunc) has a linkage attribute. If omitted, the default is `external`.

### Linkage Types Reference

| Linkage | Symbol in object file? | Multiple definitions? | Discardable? | C/C++ analogue |
|---------|------------------------|----------------------|--------------|----------------|
| `private` | No | N/A | N/A | Anonymous (no external name) |
| `internal` | Local symbol | N/A | N/A | C `static` at file scope |
| `available_externally` | No (not emitted) | Treated as `external` at CG | — | Inlined cross-module definition (ThinLTO) |
| `linkonce` | Weak global | Yes (one copy kept) | Yes, if unused | C++ inline function (non-ODR) |
| `linkonce_odr` | Weak global | Yes, must be identical | Yes, if unused | C++ inline function, template instantiation |
| `weak` | Weak global | Yes (one copy kept) | No | Override-able default implementation |
| `weak_odr` | Weak global | Yes, must be identical | No | Override-able, ODR-guaranteed |
| `common` | Common symbol | Yes (merged, zero-init) | No | C tentative definition |
| `appending` | — | Concatenated | No | `@llvm.used`, `@llvm.global_ctors` |
| `extern_weak` | Undefined weak | — | Resolves to null if absent | `__attribute__((weak))` declaration |
| `external` | Global / external ref | One definition required | No | Default C/C++ global |

### Details and ABI Implications

**`private`** emits no symbol at all. The global is accessible only within the current object file, and only by its `Value*` identity in the IR. Clang emits `private` for string literals, lifetime markers, and other synthetic data that should never appear in the symbol table. There is no `STB_LOCAL` entry in the `.symtab` — the address is simply patched by the assembler.

**`internal`** emits a local symbol (`STB_LOCAL` in ELF). This is the LLVM analogue of C `static`: the symbol is visible to debuggers and profilers as a named entity, but the linker will not resolve it from other objects. Clang emits `internal` for `static` functions and `static` global variables.

**`available_externally`** marks a definition that is present in this IR module *for inlining purposes only*. The definition will **never** be emitted into the object file; it is as-if `external` for code generation. The primary consumer is ThinLTO's cross-module inlining: the importer synthesizes `available_externally` copies of imported functions so that IPO passes can inline them without requiring a subsequent link-time merge. This linkage cannot appear on declarations (it requires an initializer or function body).

**`linkonce`** and **`linkonce_odr`** are used for definitions that may appear in multiple translation units and should be deduplicated by the linker. The difference is the ODR guarantee: `linkonce` tells the linker "keep any one copy, they may differ"; `linkonce_odr` asserts "all copies are identical under the C++ One Definition Rule, so the linker may freely merge or discard them." `linkonce_odr` is the correct linkage for C++ inline functions, `constexpr` functions, template instantiations, and vtables. A `linkonce`/`linkonce_odr` symbol may be discarded from an object file if no reference to it exists in the same file — the linker is not required to pull it in. The ODR equivalence assertion is also load-bearing for LTO devirtualization (Chapter 69): because all `linkonce_odr` copies of a virtual function are identical, the LTO pass can treat any vtable slot as a known callee and replace a virtual dispatch with a direct call or even an inlined copy.

**`weak`** and **`weak_odr`** are like their `linkonce` counterparts but are *not* discardable: even if no reference exists in the current file, the symbol is emitted. `weak` is appropriate for override-able hook functions (e.g., a library provides a default `__malloc_hook`; user code can override it with a strong symbol). `weak_odr` combines the non-discardable guarantee with the ODR equivalence assertion.

**`common`** models C's *tentative definitions* — a file-scope variable declaration without an explicit initializer, like `int errno;`. Multiple translation units may each have a tentative definition; the linker merges them into a single zero-initialized symbol. In ELF this corresponds to a `STT_OBJECT` symbol in the `SHN_COMMON` pseudo-section; the linker allocates space in BSS. `common` globals must be zero-initialized; they cannot have an explicit initializer.

```llvm
; C tentative definition: int global_counter;
@global_counter = common global i32 0, align 4
```

**`appending`** is a special linkage for array-typed globals that the linker concatenates rather than merges. It is restricted to a fixed set of magic globals known to both LLVM and the linker:

- `@llvm.used`: an array of `ptr` values that must be retained by the optimizer (equivalent to `__attribute__((used))`).
- `@llvm.compiler.used`: like `@llvm.used` but the optimizer may eliminate the reference; the symbol is still kept by the compiler but not necessarily by the linker.
- `@llvm.global_ctors`: constructors to run before `main`; array of `{i32, ptr, ptr}` (priority, function, associated data).
- `@llvm.global_dtors`: destructors to run at program exit.

```llvm
; Keep @perf_handler from being optimized away
@llvm.used = appending global [1 x ptr] [ptr @perf_handler]

; Register a module constructor at priority 65535
@llvm.global_ctors = appending global
    [1 x {i32, ptr, ptr}]
    [{i32, ptr, ptr} {i32 65535, ptr @module_init, ptr null}]
```

**`extern_weak`** declares a symbol that *may not be defined*: if no definition is provided at link time, the symbol resolves to `null` (address zero) rather than causing a link error. This enables optional library features: code can check `if (&optional_func != nullptr)` before calling. In ELF this corresponds to `STB_WEAK` with an undefined binding. Note that on ELF, `extern_weak` functions must be called through an indirect call; the direct-call optimization is unsafe because the target address is null when unresolved.

```llvm
; Optional hook: resolves to null if not provided
@optional_perf_hook = extern_weak global void (i64)

; Caller checks before use
define void @maybe_record(i64 %ns) {
  %hook = load ptr, ptr @optional_perf_hook
  %is_null = icmp eq ptr %hook, null
  br i1 %is_null, label %skip, label %call
call:
  call void %hook(i64 %ns)
  br label %skip
skip:
  ret void
}
```

**`external`** is the default: a single definition must exist across all translation units, and the linker errors if more than one definition is found (modulo weak/common merging rules). External functions and globals emit a `STB_GLOBAL` symbol; external declarations emit an undefined reference resolved at link time.

---

## 7. `dso_local`, `dllimport`, and `dllexport`

### `dso_local`

The `dso_local` attribute asserts that the symbol will be *resolved within the current DSO* — the object file or shared library being compiled. When the linker agrees (because the symbol is not in the dynamic symbol table or is protected), it can generate a direct call or PC-relative data reference instead of going through the PLT (Procedure Linkage Table) or GOT (Global Offset Table).

The performance implication is significant for hot functions: a PLT stub adds an indirect branch and a GOT load on the critical path, while a `dso_local` direct call is a single `call rel32` instruction. On AArch64 and RISC-V, it also enables the compiler to generate PC-relative loads for global variables instead of materializing a full GOT-relative address.

Clang emits `dso_local` on:
- All symbols when compiling with `-fno-pic` (non-position-independent executables).
- Symbols with `__attribute__((visibility("hidden")))` or `__attribute__((visibility("protected")))` — these cannot be preempted by DSO interposition.
- Symbols with `__attribute__((visibility("default")))` when `-fno-semantic-interposition` is passed.

```llvm
; Hidden symbol: cannot be preempted, dso_local is sound
@hidden_var = dso_local hidden global i32 0

; Default visibility but dso_local asserted (needs -fno-semantic-interposition)
define dso_local i32 @hot_function(i32 %x) {
  ret i32 %x
}
```

**Semantic interposition.** By default, a `default` visibility symbol in a shared library *can* be preempted: another DSO loaded earlier in the `LD_PRELOAD` order can substitute a different definition. `-fno-semantic-interposition` tells the compiler that the user will not exercise this preemption for non-exported symbols, making `dso_local` safe. This is the default in GCC 9+ and can be enabled in Clang with the flag.

### `dllimport` and `dllexport`

On Windows PE/COFF targets, the symbol visibility mechanism uses the DLL import/export table rather than ELF `.dynsym`. Two DLL storage attributes model this:

**`dllexport`** marks a symbol as exported from a DLL: it appears in the `.edata` (export directory) section and is accessible to other modules that load the DLL. Clang emits `dllexport` for `__declspec(dllexport)` and for `__attribute__((visibility("default")))` on Windows targets.

**`dllimport`** marks a symbol as imported from another DLL. The compiler must access it through the Import Address Table (IAT) — a `ptr`-sized slot in the `.idata` section that the Windows loader patches with the actual address at load time. This means:
- Function calls go through an indirect call: `call [__imp_func_name]`.
- Variable accesses go through a double-dereference: `@__imp_var` is a pointer to the actual storage.

```llvm
; Accessing a variable declared __declspec(dllimport) in C
@__imp_errno = external dllimport global ptr

; Calling a DLL-imported function
declare dllimport i32 @MessageBoxA(ptr, ptr, ptr, i32)
```

**Interaction with dso_local.** `dllimport` and `dso_local` are mutually exclusive: an imported symbol is by definition not local to the current DSO. LLVM's verifier rejects `dso_local dllimport` combinations. `dllexport` may coexist with `dso_local` on the exporting side.

---

## 8. Comdats

### What Comdats Are

A *comdat* (common data) is a linker feature for grouping sections that must be kept or discarded together. When the linker encounters multiple object files each containing a comdat group with the same name, it applies a *selection rule* to decide which copy to keep. This is essential for C++ because many constructs must appear in every translation unit that uses them — inline functions, template instantiations, vtables, type-info objects, string literals referenced by vtable RTTI — yet the final binary must contain exactly one copy.

In LLVM IR, comdats are declared at the module level with a selection kind, then referenced by globals and functions:

```llvm
; Module-level comdat declarations
$my_func = comdat any
$my_data = comdat exactmatch

; Function in a comdat group
define linkonce_odr void @my_func() comdat($my_func) {
  ret void
}

; Variable associated with the same group
@my_vtable = linkonce_odr constant [2 x ptr] [ptr null, ptr @my_func],
             comdat($my_func)
```

When the comdat name matches the global's own name, the parenthesized name may be omitted (`comdat` without parentheses). This is the common case for functions.

### Comdat Selection Kinds

| Kind | Behavior | ELF mapping | COFF mapping |
|------|----------|-------------|--------------|
| `any` | Keep any one copy; others are discarded | `GRP_COMDAT` (WEAK, any) | `IMAGE_COMDAT_SELECT_ANY` |
| `exactmatch` | All copies must be bitwise identical; error otherwise | GNU `STB_GLOBAL` unique | `IMAGE_COMDAT_SELECT_EXACT_MATCH` |
| `largest` | Keep the largest (by section size) | Not directly supported | `IMAGE_COMDAT_SELECT_LARGEST` |
| `nodeduplicate` | All copies are retained (no deduplication) | Separate sections | `IMAGE_COMDAT_SELECT_NODUPLICATES` |
| `samesize` | All copies must have the same size | Not directly supported | `IMAGE_COMDAT_SELECT_SAME_SIZE` |

The `any` kind corresponds to the normal C++ `linkonce_odr` use case: the linker picks one copy and discards the rest, trusting the ODR guarantee that they are equivalent. On ELF with GNU binutils or LLD, this maps to a `SHF_GROUP` section group with `GRP_COMDAT` flag; on COFF it maps to `IMAGE_COMDAT_SELECT_ANY`.

`exactmatch` is rarely used in practice but is the theoretically correct kind when the ODR equivalence should be enforced by the linker rather than trusted. Some security hardening toolchains use it to detect ODR violations at link time.

`nodeduplicate` forces all copies to be kept — useful for sections that must exist once per object file (e.g., profiling stubs that count per-file events). It defeats the space savings of comdat but preserves per-file granularity.

### C++ Use Cases in Detail

**Inline functions and their associated data.** A C++ inline function defined in a header is instantiated in every translation unit that includes the header. Each TU emits a `linkonce_odr` function with a same-named comdat. The linker keeps one copy and discards the rest. Crucially, *all globals that belong to the same comdat group are kept or discarded together*: if the vtable for a class is in the same comdat as the key virtual function, they will always appear together, preventing a vtable without its associated function body.

```llvm
; C++ class Foo: vtable, typeinfo, and destructor in one comdat group
$_ZN3FooD1Ev = comdat any    ; comdat named after the destructor

define linkonce_odr void @_ZN3FooD1Ev(ptr %this) comdat($\_ZN3FooD1Ev) {
  ret void
}
@_ZTV3Foo = linkonce_odr constant [3 x ptr]
    [ptr null, ptr @_ZTI3Foo, ptr @_ZN3FooD1Ev],
    comdat($_ZN3FooD1Ev)
@_ZTI3Foo = linkonce_odr constant {ptr, ptr}
    {ptr @_ZTVN10__cxxabiv117__class_type_infoE, ptr @_ZTS3Foo},
    comdat($_ZN3FooD1Ev)
```

**Template instantiations.** Every explicit or implicit template instantiation that is not in a designated TU uses `linkonce_odr` with a comdat. The mangled name of the instantiation serves as the comdat name, ensuring that all copies of `std::vector<int>::push_back` are merged into one definition at link time.

**The `__attribute__((weak))` / `weak_odr` combination without comdat.** Note that `weak_odr` without a comdat is weaker than `linkonce_odr` with a comdat: without comdat, the linker treats associated data independently, potentially keeping the vtable from one TU and the function body from a different TU. Correct C++ ABI implementations always pair `linkonce_odr` globals with comdats on COFF targets and often on ELF targets.

---

## 9. Chapter Summary

- **`ConstantExpr`** is a constant-folded expression tree with no basic block parent. In LLVM 22 the surviving opcodes are `getelementptr`, `bitcast`, `inttoptr`, `ptrtoint`, `addrspacecast`, `add`, `sub`, `xor`, and `trunc`; arithmetic, logical, comparison, and vector opcodes have been removed. New code should use instructions in initializer functions rather than removed constant expression opcodes.

- **`ConstantData`** leaves (`ConstantInt`, `ConstantFP`, `ConstantDataArray`, `ConstantDataVector`) carry their value internally with no references to other LLVM values; `ConstantAggregate` types (`ConstantArray`, `ConstantStruct`, `ConstantVector`) hold vectors of `Constant*` operands. Use `ConstantDataArray::getString()` for string literals.

- **`undef`** can take any bit pattern per-use and does not imply undefined behavior. **`poison`** propagates through arithmetic and triggers undefined behavior if it reaches a side-effecting instruction; the `freeze` instruction stops propagation by producing an arbitrary but fixed value. Prefer `poison` over `undef` for new UB-modeling.

- **`GlobalVariable`** carries linkage, visibility, initializer, thread-local model, `unnamed_addr`, section, comdat, and alignment attributes. The `constant` keyword enables read-only section placement and alias-analysis shortcuts. External declarations have no initializer; `hasInitializer()` distinguishes definitions from declarations.

- **`GlobalAlias`** provides an alternative name for a global or a constant expression over one. **`GlobalIFunc`** defers function address resolution to a resolver called at load time; it is ELF-only and used for CPU dispatch.

- The four ELF TLS models — General Dynamic, Local Dynamic, Initial Executable, Local Executable — trade generality for performance. LE requires one register-relative instruction and no GOT access; GD requires two `__tls_get_addr` calls. The linker can relax GD to IE or LE at link time for statically linked variables.

- The full linkage taxonomy spans `private` (no symbol), `internal` (local symbol, C `static`), `available_externally` (for IPO, not emitted), `linkonce`/`linkonce_odr` (discardable, deduplicated), `weak`/`weak_odr` (non-discardable, override-able), `common` (tentative definitions, zero-init), `appending` (linker-concatenated arrays), `extern_weak` (null-resolving optional symbol), and `external` (default). The `_odr` variants carry an ODR equivalence assertion that enables safe linker merging and LTO devirtualization.

- **`dso_local`** avoids PLT/GOT indirection by asserting the symbol resolves within the current DSO; enabled by hidden/protected visibility or `-fno-semantic-interposition`. **`dllimport`/`dllexport`** are the Windows IAT/export-directory counterparts; `dllimport` mandates indirect access through `__imp_` stubs.

- **Comdats** group related sections so the linker keeps or discards them atomically. The `any` selection kind maps to C++ `linkonce_odr` semantics (keep one copy). Correct C++ ABI implementations pair `linkonce_odr` vtables, type-info objects, and inline function bodies into a single comdat group so they are always kept or discarded together.

---

*Cross-references: [Chapter 16 — IR Structure](ch16-ir-structure.md) covers module layout and the `Value`/`User`/`Use` graph; [Chapter 17 — The Type System](ch17-the-type-system.md) covers the type hierarchy this chapter assumes; [Chapter 77 — LTO and ThinLTO](../../part-13-lto-whole-program/ch77-lto-and-thinlto.md) explores how `available_externally` and `linkonce_odr` interact during cross-module optimization; [Chapter 79 — Linker Internals: GOT, PLT, TLS](../../part-13-lto-whole-program/ch79-linker-internals-got-plt-tls.md) digs into the ELF relocation mechanics that underpin the TLS models and `dso_local` optimization.*
