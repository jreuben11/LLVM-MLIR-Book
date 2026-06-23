# Chapter 23 — Attributes, Calling Conventions, and the ABI

*Part IV — LLVM IR*

Every function declaration in LLVM IR carries two kinds of contract: one with the optimizer, expressed through *attributes*, and one with the hardware, expressed through *calling conventions*. Together these form the ABI boundary — the precise specification of how values flow into and out of a function, which registers hold them, which stack locations back them up, and what the callee is permitted to clobber. Getting this contract right is what makes it possible for a Clang-compiled C++ library to be called from a Rust crate compiled with a different optimization pipeline, on the same target, with no runtime adapter.

This chapter is a systematic treatment of all three layers. It begins with a full reference to function, parameter, and call-site attributes; explains attribute groups and how Clang uses them to compress IR size; catalogues every calling convention that LLVM 22 recognizes; then examines the ABI lowering boundary where IR-level abstractions are converted to actual machine instructions during instruction selection. The chapter closes with a careful examination of the `sret`, `byval`, and `byref` attributes that govern struct passing — the area where IR conventions and machine ABI conventions diverge most visibly.

All IR examples are verified against LLVM 22.1.3. Cross-references to later chapters are noted where deeper treatment is available.

---

## Table of Contents

- [23.1 Function Attributes: Comprehensive Reference](#231-function-attributes-comprehensive-reference)
  - [23.1.1 Memory-Effect Attributes](#2311-memory-effect-attributes)
  - [23.1.2 Control-Flow Attributes](#2312-control-flow-attributes)
  - [23.1.3 Inlining and Optimization Hint Attributes](#2313-inlining-and-optimization-hint-attributes)
  - [23.1.4 Prologue and Stack Attributes](#2314-prologue-and-stack-attributes)
  - [23.1.5 Target Feature and Alignment Attributes](#2315-target-feature-and-alignment-attributes)
  - [23.1.6 Sanitizer Attributes](#2316-sanitizer-attributes)
  - [23.1.7 A Complete Function Definition](#2317-a-complete-function-definition)
- [23.2 Parameter and Return Attributes](#232-parameter-and-return-attributes)
  - [23.2.1 Nullability and Definedness](#2321-nullability-and-definedness)
  - [23.2.2 Aliasing Attributes](#2322-aliasing-attributes)
  - [23.2.3 Struct-Passing Attributes](#2323-struct-passing-attributes)
  - [23.2.4 Register and Extension Attributes](#2324-register-and-extension-attributes)
  - [23.2.5 Swift-Specific Attributes](#2325-swift-specific-attributes)
- [23.3 Call Site Attributes](#233-call-site-attributes)
- [23.4 Attribute Groups](#234-attribute-groups)
  - [23.4.1 Syntax](#2341-syntax)
  - [23.4.2 Grouping Policy in Clang](#2342-grouping-policy-in-clang)
  - [23.4.3 Attribute Groups vs. Function Attributes in the C++ API](#2343-attribute-groups-vs-function-attributes-in-the-c-api)
- [23.5 Calling Conventions: The Complete Catalogue](#235-calling-conventions-the-complete-catalogue)
  - [23.5.1 General-Purpose Conventions](#2351-general-purpose-conventions)
  - [23.5.2 WebKit and JavaScript Conventions](#2352-webkit-and-javascript-conventions)
  - [23.5.3 Apple Swift Conventions](#2353-apple-swift-conventions)
  - [23.5.4 x86 ABI Variants](#2354-x86-abi-variants)
  - [23.5.5 Windows x64](#2355-windows-x64)
  - [23.5.6 GPU and Compute Conventions](#2356-gpu-and-compute-conventions)
  - [23.5.7 Choosing a Calling Convention](#2357-choosing-a-calling-convention)
- [23.6 The ABI Lowering Boundary](#236-the-abi-lowering-boundary)
  - [23.6.1 What "ABI" Means in the LLVM IR Context](#2361-what-abi-means-in-the-llvm-ir-context)
  - [23.6.2 The ABI Lowering Pipeline](#2362-the-abi-lowering-pipeline)
  - [23.6.3 `TargetLowering` and `CCAssignFn`](#2363-targetlowering-and-ccassignfn)
  - [23.6.4 Pre-ABI and Post-ABI Passes](#2364-pre-abi-and-post-abi-passes)
  - [23.6.5 Clang's Role: Generating the ABI-Correct IR](#2365-clangs-role-generating-the-abi-correct-ir)
- [23.7 `sret`, `byval`, and `byref`: The Struct-Passing Nuances](#237-sret-byval-and-byref-the-struct-passing-nuances)
  - [23.7.1 `sret(T)` — Struct Return](#2371-srett-struct-return)
  - [23.7.2 `byval(T)` — Pass by Value (Copy Semantics)](#2372-byvalt-pass-by-value-copy-semantics)
  - [23.7.3 `byref(T)` — Pass by Reference (No Copy)](#2373-byreft-pass-by-reference-no-copy)
  - [23.7.4 `byval` in IR vs. `byval` in the C ABI: The Name Collision](#2374-byval-in-ir-vs-byval-in-the-c-abi-the-name-collision)
  - [23.7.5 `inalloca(T)` — In-Place Allocation for Non-Trivial Destructors](#2375-inallocat-in-place-allocation-for-non-trivial-destructors)
- [23.8 Practical Examples](#238-practical-examples)
  - [23.8.0 The `noundef` Attribute and Optimizer Implications](#2380-the-noundef-attribute-and-optimizer-implications)
  - [23.8.1 Verifying Attribute Emission from Clang](#2381-verifying-attribute-emission-from-clang)
  - [23.8.2 Reading Attribute Effects in `opt` Pipelines](#2382-reading-attribute-effects-in-opt-pipelines)
  - [23.8.3 Calling Convention in Optimized IR](#2383-calling-convention-in-optimized-ir)
  - [23.8.4 Inspecting Calling Convention Lowering](#2384-inspecting-calling-convention-lowering)
  - [23.8.5 Cross-Language ABI Compatibility](#2385-cross-language-abi-compatibility)
- [23.9 Chapter Summary](#239-chapter-summary)

---

## 23.1 Function Attributes: Comprehensive Reference

Function attributes appear between the `define`/`declare` keyword and the function body (or after the function signature). They express facts about the function that the optimizer can rely on, and constraints on what the function is permitted to do. An incorrect attribute is not a runtime error: it is undefined behavior that the optimizer is entitled to exploit.

### 23.1.1 Memory-Effect Attributes

The most consequential optimizer attributes describe a function's effect on memory. Prior to LLVM 16, this was expressed through the legacy trio `readnone`, `readonly`, and `writeonly`. LLVM 16 replaced all three with a single structured attribute `memory(...)` that describes effects per memory location category. As of LLVM 22, the legacy forms are still accepted at parse time but are immediately converted to `memory(...)` internally.

The grammar is:

```
memory(<effect>)
memory(<location> : <effect>, ...)
```

where `<effect>` is one of `none`, `read`, `write`, `readwrite`, and `<location>` is one of `argmem`, `inaccessiblemem`, `other` (covering everything not in `argmem` or `inaccessiblemem`).

| Attribute | Optimizer inference |
|-----------|---------------------|
| `memory(none)` | The function neither reads nor writes any memory visible outside its own stack frame. Equivalent to `readnone`. Calls may be freely CSE'd, hoisted, or sunk. |
| `memory(read)` | The function reads but does not write any accessible memory. Equivalent to `readonly`. Calls may not be CSE'd (reads may have different values), but call elimination is legal if the result is unused. |
| `memory(write)` | The function writes but does not read through any pointer accessible from outside. Equivalent to `writeonly`. |
| `memory(readwrite)` | Full aliasing; no inference. The default when no `memory` attribute is present. |
| `memory(argmem: read)` | Reads only through its pointer arguments; does not touch other global/heap memory. |
| `memory(argmem: readwrite, inaccessiblemem: read)` | Reads/writes argument memory, reads from memory invisible to the IR (e.g., `errno`). |

```llvm
; Pure function — may be CSE'd across basic blocks
declare i32 @strlen(ptr nocapture nonnull) memory(argmem: read) nounwind

; Writes only to its argument pointer, touches no other memory
declare void @memset_zero(ptr nocapture writeonly, i64) memory(argmem: write) nounwind
```

The `memory` attribute is defined in [`llvm/include/llvm/IR/Attributes.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/Attributes.td) and its interpretation is implemented in [`llvm/lib/Analysis/MemoryEffects.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Analysis/MemoryEffects.cpp).

### 23.1.2 Control-Flow Attributes

| Attribute | Meaning |
|-----------|---------|
| `nounwind` | The function never unwinds the stack via a C++ exception or `longjmp`. The backend may omit landing pad entries from the unwind table. Clang emits this on every function declared `noexcept` and on many C library functions. |
| `noreturn` | The function does not return to its call site (e.g., `exit`, `abort`, `__cxa_throw`). The code after a `call` to a `noreturn` function is dead; the optimizer may eliminate it. |
| `willreturn` | The function *will* return if it terminates normally. This rules out infinite loops and allows the optimizer to assume the function returns (enabling code motion across the call). Combined with `memory(none)`, a function is effectively a pure computation. |
| `nosync` | The function does not contain any synchronization operations (atomics, fences, or calls to synchronizing functions). Allows the optimizer to move the call across synchronization barriers. |
| `convergent` | The function may not be moved, duplicated, or speculated relative to control-flow divergence points. Critical for GPU shaders: a barrier intrinsic (`llvm.amdgcn.s.barrier`) must be executed by all threads in a wavefront that execute it — the optimizer must not hoist it out of a divergent branch. See [Chapter 93 — GPU Code Generation](../part-15-targets/ch93-amdgpu-codegen.md). |
| `speculatable` | The function is safe to call speculatively (i.e., in a path that would not be reached in the original control flow), provided its arguments are not `poison`. It may not trap, even for edge-case inputs. Used for pure math functions. |

### 23.1.3 Inlining and Optimization Hint Attributes

| Attribute | Meaning |
|-----------|---------|
| `noinline` | The inliner must not inline this function, even when heuristics favor it. Used by debuggers, profilers, `__attribute__((noinline))`, and sanitizers. |
| `alwaysinline` | The inliner must inline this function at every call site. The optimizer will warn and respect the attribute even across module boundaries (with LTO). Conflicts with `noinline`; `alwaysinline` wins. |
| `inlinehint` | A hint (not a mandate) that inlining is beneficial. Clang emits this for `inline` functions in C++. |
| `optnone` | Optimize this function at `-O0` regardless of the module-level optimization level. Used by `__attribute__((optnone))` and by the debugger when stepping into a specific function. |
| `optsize` | Optimize for size over speed (`-Os`). The pass pipeline applies size-favoring heuristics. |
| `minsize` | Optimize aggressively for code size (`-Oz`). Supersedes `optsize`; will select shorter instruction sequences even at the cost of speed. |
| `cold` | Marks a function (or call site) as rarely executed. The basic block layout passes will move code into cold sections; the inliner reduces willingness to inline into cold callers. |
| `hot` | Marks a function as frequently executed. Supersedes the heuristic cost model; the inliner will be more aggressive. |

### 23.1.4 Prologue and Stack Attributes

| Attribute | Meaning |
|-----------|---------|
| `naked` | The function has no compiler-generated prologue or epilogue. Used for low-level interrupt handlers written in inline assembly. The programmer is responsible for callee-saved register save/restore. |
| `uwtable` | The function must have an entry in the `.eh_frame` / `.pdata` unwind table, even if `nounwind` is also set. Required when the runtime needs to unwind through the function (e.g., profilers, sanitizers, `backtrace(3)`). |
| `uwtable(sync)` | Generates a synchronous (Dwarf-style `.eh_frame`) unwind table. |
| `uwtable(async)` | Generates an asynchronous (fully accurate at every instruction) unwind table. Required when profilers use hardware performance counter NMI-based stack sampling. LLVM generates the larger `async` form when the target or platform requires it. |
| `"frame-pointer"="none"` | The backend may omit the frame pointer entirely (default at `-O1+`). |
| `"frame-pointer"="non-leaf"` | Retain the frame pointer in functions that call other functions. |
| `"frame-pointer"="all"` | Always retain the frame pointer (useful for debugging). Equivalent to `-fno-omit-frame-pointer`. |

The `frame-pointer` string attribute (not to be confused with the function `align` attribute) is emitted by Clang as a string key-value pair and is consumed by the `MachineFunctionPass` infrastructure during backend lowering, not by the IR-level optimizer.

### 23.1.5 Target Feature and Alignment Attributes

| Attribute | Meaning |
|-----------|---------|
| `"target-cpu"="znver4"` | Override the target CPU for this function only. Clang emits this from `__attribute__((target("cpu=znver4")))`. The backend selects instruction scheduling and ISA features accordingly. |
| `"target-features"="+avx512f,+avx512bw"` | Override the set of enabled/disabled ISA features for this function. Enables function multiversioning: one module can contain both a baseline and an AVX-512 path, selected at runtime by IFUNC. |
| `align N` | The machine code for the function must be aligned to at least N bytes. Common values: 4 (ARM Thumb interworking), 16 (branch target alignment on x86). |
| `vscale_range(min, max)` | Bounds on the SVE (AArch64) or RVV (RISC-V) `vscale` value at runtime. `vscale_range(1, 16)` asserts that `vscale` is at least 1 and at most 16. This allows the optimizer to generate scalar fallback code only for the stated range, and to compute vector loop trip counts statically. See [Chapter 94 — AArch64 SVE Lowering](../part-15-targets/ch94-aarch64-sve.md). |

### 23.1.6 Sanitizer Attributes

Clang's sanitizers instrument functions by inserting runtime checks. The following attributes mark a function as being compiled under a given sanitizer, so that other passes can skip redundant instrumentation or adjust their behavior:

| Attribute | Sanitizer |
|-----------|-----------|
| `sanitize_address` | AddressSanitizer (ASan) — detects heap/stack/global buffer overflows |
| `sanitize_memory` | MemorySanitizer (MSan) — detects use of uninitialized memory |
| `sanitize_thread` | ThreadSanitizer (TSan) — detects data races |
| `sanitize_hwaddress` | Hardware-assisted ASan (HWAsan) — uses top-byte tagging |
| `sanitize_realtime` | RealtimeSanitizer (RTSan) — detects non-realtime-safe calls |

### 23.1.7 A Complete Function Definition

```llvm
; A function with a representative attribute set as Clang would emit it
define noundef nonnull ptr @create_object(i64 noundef %size)
    #0 !dbg !5 {
entry:
  %call = tail call noalias ptr @malloc(i64 %size) #1
  ret ptr %call
}

; Attribute group #0 consolidates all function-level attributes
attributes #0 = {
  nounwind
  willreturn
  memory(inaccessiblemem: readwrite)
  "frame-pointer"="all"
  "no-trapping-math"="true"
  "target-cpu"="x86-64"
  "target-features"="+cmov,+cx8,+fxsr,+mmx,+sse,+sse2,+x87"
}

; Attribute group #1 for the malloc call site
attributes #1 = { nounwind allocsize(0) }
```

---

## 23.2 Parameter and Return Attributes

Parameter attributes appear in the function signature, after the type of each parameter. Return attributes appear after the `define`/`declare` keyword, before the return type. They annotate the *value* being passed or returned, not the function as a whole.

### 23.2.1 Nullability and Definedness

| Attribute | Placement | Meaning |
|-----------|-----------|---------|
| `nonnull` | Return, parameter (ptr) | The pointer is not null. The optimizer may eliminate null checks on this value. Incorrect use causes UB if null is actually passed. |
| `noundef` | Return, parameter | The value is not `undef` or `poison`. LLVM 16+ uses this extensively: without `noundef`, a parameter value could be poison, and the optimizer must account for that. With `noundef`, optimizations that assume the value is defined (e.g., GVN, SCCP) apply more aggressively. |
| `dereferenceable(n)` | Return, parameter (ptr) | The pointer refers to at least *n* bytes of accessible memory. The optimizer may load from the first *n* bytes without a null check. |
| `dereferenceable_or_null(n)` | Return, parameter (ptr) | Either null, or dereferenceable for *n* bytes. Weaker than `dereferenceable`; null checks are still needed. |

### 23.2.2 Aliasing Attributes

| Attribute | Placement | Meaning |
|-----------|-----------|---------|
| `noalias` | Return, parameter (ptr) | The pointed-to memory is not aliased by any other pointer accessible in the function. On a return value, it marks freshly allocated memory (like `malloc`). On a parameter, it corresponds to C99 `restrict`. |
| `nocapture` | Parameter (ptr) | The callee does not store the pointer value in memory that outlives the call (no global store, no heap allocation containing the pointer). This allows alias analysis to constrain the pointer's lifetime. |
| `readonly` | Parameter (ptr) | The callee only reads through this pointer; does not write. Superseded by `memory(argmem: read)` at the function level, but may be used on individual parameters for finer granularity. |
| `writeonly` | Parameter (ptr) | The callee only writes through this pointer. |

### 23.2.3 Struct-Passing Attributes

These are discussed in detail in Section 23.7; a brief reference here:

| Attribute | Placement | Meaning |
|-----------|-----------|---------|
| `byval(T)` | Parameter (ptr) | Pass by value: the caller makes a copy of type `T` at the stack; the callee receives a pointer to the copy. The `T` is required and records the copied type. An optional `align N` may follow. |
| `byref(T)` | Parameter (ptr) | Pass by reference with callee-side semantics: the pointer addresses a `T` but the callee may not take ownership (no copy). Used for Swift `inout`. |
| `sret(T)` | Parameter (ptr) | The pointer is a *struct return* slot: the callee must write a value of type `T` to this location and returns void at the IR level. Caller allocates. |
| `inalloca(T)` | Parameter (ptr) | Like `byval`, but the stack allocation is performed by the caller using a separate `alloca`; used when passing C++ objects with non-trivial constructors on x86. |

### 23.2.4 Register and Extension Attributes

| Attribute | Placement | Meaning |
|-----------|-----------|---------|
| `zeroext` | Return, parameter (integer) | Zero-extend the integer to the natural register width before passing/returning. Required by AAPCS and SystemV AMD64 ABI for sub-word integers. |
| `signext` | Return, parameter (integer) | Sign-extend the integer to the natural register width. |
| `inreg` | Return, parameter | Pass or return this value in registers (as opposed to on the stack). The exact register assignment is determined by the calling convention. On x86-32, marks the value for `ecx`/`edx` assignment under `fastcall`. |
| `align(n)` | Parameter (ptr) | The pointer is aligned to at least *n* bytes. The optimizer may generate aligned load/store instructions through this pointer. |

### 23.2.5 Swift-Specific Attributes

| Attribute | Meaning |
|-----------|---------|
| `swiftself` | This parameter carries the Swift `self` reference. The backend assigns it to a dedicated register (e.g., `r13` on x86-64, `x20` on AArch64) that is preserved across function calls. |
| `swiftasync` | This parameter is the Swift async context pointer, passed in a caller-designated register. |
| `swifterror` | This parameter is a Swift error return pointer, passed in a dedicated register (`r12` on x86-64, `x21` on AArch64). The callee writes `null` on success or an error object pointer on failure. |

---

## 23.3 Call Site Attributes

Any attribute that may appear on a function parameter can also appear at a specific call site, attached to the argument at that call. Call site attributes *override or supplement* the callee's declared attributes for that call only. This matters in several scenarios:

1. **Indirect calls.** When calling through a function pointer, there is no callee declaration to carry attributes. The call site must supply any attributes that matter for this call.

2. **Cross-function optimization.** An inliner may add `nonnull` at the call site after proving that a specific argument cannot be null at this site, even if the general declaration does not carry `nonnull`.

3. **`musttail`.** The `musttail` marker at a call site tells the optimizer that this call *must* be turned into a tail call. The backend will error if the transformation is impossible given the calling convention. Used with `tailcc` and `swifttailcc`.

4. **`notail`.** The opposite: this call must *not* be converted to a tail call (e.g., the address of a local variable is passed and must remain valid after the call in the callee's frame).

5. **`tail`.** A hint that the call *may* be converted to a tail call if convenient, but is not required.

```llvm
; musttail example — the backend MUST emit a tail call here
define tailcc void @dispatch(ptr %fn, i64 %arg) {
entry:
  musttail call tailcc void %fn(i64 noundef %arg)
  ret void
}
```

---

## 23.4 Attribute Groups

Every non-trivial function in Clang-generated IR carries dozens of attributes. If each attribute were spelled out inline on every `define`, the IR would become unreadably wide and the text serialization would grow substantially. LLVM's solution is *attribute groups*.

### 23.4.1 Syntax

An attribute group assigns a numeric label to a set of attributes:

```llvm
attributes #0 = { nounwind willreturn memory(none) "no-trapping-math"="true" }
attributes #1 = { cold noinline }
attributes #2 = { alwaysinline nounwind "target-features"="+avx512f" }
```

Functions reference the group by number:

```llvm
define void @hot_path(i32 %x) #0 { ... }
define void @error_handler() #1 { ... }
define void @vec_kernel(ptr %p) #2 { ... }
```

### 23.4.2 Grouping Policy in Clang

Clang assigns attributes to groups in [`clang/lib/CodeGen/CodeGenModule.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CodeGenModule.cpp) via `addDefaultFunctionDefinitionAttributes`. The grouping strategy is:

- All functions compiled at the same optimization level with the same target CPU and feature set share a single attribute group for the common attributes.
- Functions with `__attribute__((noinline))`, `__attribute__((cold))`, etc., get a separate group.
- Sanitizer-instrumented functions get an additional group for the sanitizer attributes.

This means a typical Clang-compiled translation unit with hundreds of functions may have as few as three or four attribute groups, regardless of the number of functions. The reduction in text size is significant: the IR for a large C++ project like Clang itself would grow by tens of megabytes if every attribute were inlined at every definition site.

### 23.4.3 Attribute Groups vs. Function Attributes in the C++ API

When constructing IR programmatically via the C++ API, attribute groups are an optimization applied during text serialization. In memory, every function carries an `AttributeList` object (see [`llvm/include/llvm/IR/Attributes.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/Attributes.h)) that holds per-function, per-return, and per-parameter attributes as three separate `AttributeSet` objects. The text serializer groups functions sharing the same `AttributeList` under the same `#N` label. The reader reconstructs the `AttributeList` from the group definition and attaches it directly to each referencing function.

---

## 23.5 Calling Conventions: The Complete Catalogue

A calling convention specifies:
- Which registers hold the first N arguments
- Where overflow arguments go on the stack (and in what order)
- Which registers the callee must save and restore
- How the return value is delivered
- Stack alignment requirements at the call site

LLVM identifies conventions by a numeric enum (`llvm::CallingConv::ID`) defined in [`llvm/include/llvm/IR/CallingConv.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/CallingConv.h). The IR textual form uses mnemonics.

### 23.5.1 General-Purpose Conventions

| Mnemonic | Numeric ID | Description |
|----------|-----------|-------------|
| `ccc` | 0 | **C calling convention.** The default; follows the platform's standard C ABI (SysV AMD64, AAPCS64, AAPCS32, MSVC x64, etc.). Every `extern "C"` function uses this. The backend's `TargetLowering` maps it to the appropriate set of argument registers and stack layout. |
| `fastcc` | 8 | **Fast calling convention.** Arguments and return values are passed in as many registers as possible; there is no ABI compatibility guarantee. LLVM uses `fastcc` for internal/static functions when it can prove no external caller exists. Clang emits `fastcc` for `__attribute__((fastcall))` only on x86-32; otherwise the optimizer promotes functions to `fastcc` during optimization. |
| `coldcc` | 9 | **Cold calling convention.** Optimized for infrequent invocations. The register allocator minimizes register pressure *at the call site*, spilling caller-saved registers less aggressively than normal. The callee may execute more prologue/epilogue save/restore instructions, but the caller code — which executes every time through the hot path — is leaner. |
| `tailcc` | 18 | **Tail calling convention.** Guarantees that tail calls within the function can be compiled as actual tail calls (no stack growth). Compatible with `musttail`. All argument registers used by the calling convention must be a subset of those used by the callee. |
| `ghccc` | 10 | **Glasgow Haskell Compiler calling convention.** All arguments in registers; no stack allocation by the runtime convention (the GHC runtime manages its own stack). Very large numbers of registers may be used; the convention is effectively undefined for a function with more arguments than there are registers. |
| `preserve_none` | 21 | **Preserve-none calling convention.** The callee may clobber all general-purpose and floating-point registers — the caller must save every live register before the call. This is the opposite of callee-saves conventions; it is designed for runtime stubs (GC write barriers, safepoint polls) that are called so frequently that pushing the save/restore cost entirely onto the callee would bloat hot-path code. Introduced in LLVM 17; identified as `CallingConv::PreserveNone` in [`llvm/include/llvm/IR/CallingConv.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/CallingConv.h). |
| `graalcc` | 107 | **GraalVM calling convention.** Reserves two additional registers beyond the platform C ABI for GraalVM's thread-local state (typically the Java thread pointer and heap base). Used when compiling native image GraalVM binaries with LLVM backends. Identified as `CallingConv::GRAAL = 107` in `CallingConv.h`. |

### 23.5.2 WebKit and JavaScript Conventions

| Mnemonic | Numeric ID | Description |
|----------|-----------|-------------|
| `webkit_jscc` | 12 | **WebKit JavaScript calling convention.** The first argument is passed in a designated register; subsequent arguments are on the stack. Designed to allow the JIT compiler to avoid marshalling overhead at polymorphic call sites. |
| `anyregcc` | 13 | **Any-register calling convention.** Used exclusively by statepoints (Chapter 86). The register allocator may assign arguments to *any* register, and the statepoint records the actual register allocation in the stackmap section. There is no fixed ABI; the caller and callee must be compiled together. |

### 23.5.3 Apple Swift Conventions

| Mnemonic | Numeric ID | Description |
|----------|-----------|-------------|
| `swiftcc` | 16 | **Swift calling convention.** Uses `swiftself` (dedicated `self` register), `swifterror` (dedicated error register), and a larger set of integer/floating-point argument registers than `ccc`. Clang emits this for all non-`@objc` Swift functions. |
| `swifttailcc` | 20 | **Swift tail calling convention.** A variant of `swiftcc` that guarantees tail-call lowering. Used by the Swift async machinery when a continuation is a direct tail call. |

### 23.5.4 x86 ABI Variants

| Mnemonic | Numeric ID | Description |
|----------|-----------|-------------|
| `x86_stdcallcc` | 64 | **x86 stdcall.** The callee cleans the stack (PASCAL calling convention). Used historically for Win32 API functions. Arguments pushed right-to-left; callee pops with `ret N`. |
| `x86_fastcallcc` | 65 | **x86 fastcall.** First two integer arguments in `ecx`/`edx`; callee cleans remaining stack arguments. Used by Microsoft's `__fastcall`. |
| `x86_thiscallcc` | 70 | **x86 thiscall.** `this` pointer in `ecx`; remaining arguments on the stack; callee cleans the stack. Used for non-virtual member function calls in MSVC. |
| `x86_vectorcallcc` | 80 | **x86/x64 vectorcall.** Extended fastcall that passes the first six floating-point/vector arguments in XMM/YMM registers, with homogeneous vector aggregates (HVAs) passed vectorially. Used by MSVC's `__vectorcall`. |
| `x86_regcallcc` | 92 | **x86 register calling convention (Intel ICC).** Passes up to sixteen integer/pointer arguments in registers and up to eight floating-point arguments in XMM registers. Designed to minimize register pressure when calling vectorized library functions. |

### 23.5.5 Windows x64

| Mnemonic | Numeric ID | Description |
|----------|-----------|-------------|
| `win64cc` | 79 | **Windows x64 ABI.** Semantically identical to `ccc` on `x86_64-pc-windows-msvc` targets; the mnemonic exists for explicit platform documentation in cross-compilation scenarios. The x64 Windows ABI uses `rcx`, `rdx`, `r8`, `r9` for the first four integer arguments (not the SysV six), requires 32 bytes of shadow space, and uses `xmm0`–`xmm3` for the first four floating-point arguments. |

### 23.5.6 GPU and Compute Conventions

| Mnemonic | Description |
|----------|-------------|
| `amdgpu_kernel` | AMDGPU compute kernel. Arguments are passed via a pointer to a kernel descriptor in SGPRs. The kernel is launched from the host and may not be called directly. |
| `amdgpu_vs` | AMDGPU vertex shader. Hardware-defined argument passing from the geometry pipeline. |
| `amdgpu_ps` | AMDGPU pixel/fragment shader. Arguments include interpolated vertex attributes and hardware pixel coordinates. |
| `amdgpu_cs` | AMDGPU compute shader. Similar to `amdgpu_kernel` but invoked from the GPU command queue via `DISPATCH_INDIRECT`. |
| `amdgpu_gs` | AMDGPU geometry shader. |
| `amdgpu_hs` | AMDGPU hull/tessellation control shader. |
| `amdgpu_ls` | AMDGPU local (pre-geometry) shader. |
| `amdgpu_es` | AMDGPU export (pre-tessellation) shader. |
| `ptx_kernel` | NVPTX kernel — callable from the CUDA host API. Corresponds to a `.entry` directive in PTX assembly. |
| `ptx_device` | NVPTX device function — callable only from other device functions or kernels. Corresponds to a `.func` in PTX. |
| `spir_kernel` | SPIR-V kernel (OpenCL compute kernel or Vulkan compute shader). Arguments passed via a pointer-to-arguments structure in OpenCL 2.x; via push constants in Vulkan. |
| `spir_func` | SPIR-V function — callable only within a SPIR-V module, not from the host. |

### 23.5.7 Choosing a Calling Convention

The rule of thumb:

- **Externally visible functions** must use `ccc` (or a platform-specific ABI variant). Changing the calling convention of a function that is exported from a shared library or called from a different compiler breaks binary compatibility.
- **Internal functions** (static, not exported) may use `fastcc`. The optimizer (`AttributorPass`, `IPSCCPPass`) promotes eligible functions to `fastcc` automatically when LTO is enabled.
- **Tail-recursive trampolines** should use `tailcc` to guarantee the transformation.
- **GPU kernels** must use the appropriate `amdgpu_*` or `ptx_*` convention; using `ccc` would generate incorrect argument loading sequences.

---

## 23.6 The ABI Lowering Boundary

### 23.6.1 What "ABI" Means in the LLVM IR Context

The term *ABI* in LLVM has two distinct scopes. At the IR level, the ABI is the calling convention as specified by the `call`/`invoke` instruction together with the parameter attributes. At the machine level, the ABI is the set of hardware-specific rules that govern register assignment, stack layout, and callee-saved register lists.

LLVM IR is a *machine-independent* representation. A function with `byval`, `sret`, or `signext` attributes is not a description of machine behavior; it is an annotation that the *backend* must map to machine behavior during instruction selection. The critical insight is that **most LLVM middle-end passes operate entirely on IR-level calling conventions** and never see register assignments. The ABI lowering happens once, late in the pipeline, during instruction selection.

### 23.6.2 The ABI Lowering Pipeline

```
Clang AST
   │
   ▼  (CodeGen: CGCall.cpp)
LLVM IR function with parameter attributes
   │
   ▼  (SROAPass, AllocaPromotion, InstCombine, ...)
Optimized LLVM IR (still using byval/sret/signext)
   │
   ▼  (SelectionDAGISel or GlobalISel)
Machine IR with virtual registers
   │
   ▼  (Register Allocator)
Machine IR with physical registers
   │
   ▼  (PrologEpilogInserter)
Machine IR with stack frame laid out
   │
   ▼  (AsmPrinter)
Target assembly / object code
```

The key boundary is between *Optimized LLVM IR* and *Machine IR*. Everything above the line is target-independent (subject to the calling convention mnemonic and attribute semantics). Everything below the line is target-specific.

### 23.6.3 `TargetLowering` and `CCAssignFn`

The mapping from IR calling conventions to register assignments is implemented by the `TargetLowering` interface, specifically the methods `LowerCall`, `LowerReturn`, and `LowerFormalArguments`. Each target backend provides a concrete implementation (e.g., [`llvm/lib/Target/X86/X86ISelLowering.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/X86/X86ISelLowering.cpp)).

The calling convention tables are described in TableGen `.td` files and compiled into C++ functions of type `CCAssignFn`:

```cpp
typedef bool CCAssignFn(unsigned ValNo, MVT ValVT, MVT LocVT,
                        CCValAssign::LocInfo LocInfo,
                        ISD::ArgFlagsTy ArgFlags,
                        CCState &State);
```

For example, the AMD64 SysV `ccc` description is in [`llvm/lib/Target/X86/X86CallingConv.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/X86/X86CallingConv.td). The TableGen-generated `CC_X86_64_C` function iterates over argument values and assigns each to a register or stack slot, consuming the ordered register sequence (`rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9` for integers; `xmm0`–`xmm7` for floating-point) and falling back to the stack when the register budget is exhausted.

This is explored in depth in [Chapter 84 — SelectionDAG: Instruction Selection](../part-14-backend/ch84-selectiondag.md) and [Chapter 95 — x86 ABI Specifics](../part-15-targets/ch95-x86-abi.md).

### 23.6.4 Pre-ABI and Post-ABI Passes

A subtle consequence of this architecture is that passes that analyze or transform function calls must be aware of whether they are running before or after ABI lowering.

**Pre-ABI passes** (everything in the LLVM middle-end optimizer — `mem2reg`, `instcombine`, `gvn`, `licm`, `inliner`, etc.) see the IR-level representation. A `byval` argument appears as a pointer parameter with a `byval` attribute; the pass must not assume anything about whether that pointer will ultimately be in a register or on the stack.

**Post-ABI passes** (register allocator, prologue/epilogue insertion, machine scheduling) see physical registers and stack offsets. They know that `rdi` holds argument 0, that `rsp+8` holds the saved return address, and so on.

The `MachineFunctionAnalysis` pass group operates in the post-ABI world. Passes that straddle this boundary — notably the interprocedural passes that run during LTO (`WholeProgramDevirt`, `GlobalDCE`) — must be careful to operate only on IR-level attributes, never on assumed physical register assignments.

### 23.6.5 Clang's Role: Generating the ABI-Correct IR

Clang's role is to *produce* IR that correctly describes the intended ABI. The code in [`clang/lib/CodeGen/CGCall.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGCall.cpp) is the most complex file in Clang's code generator — it handles:

- Selecting the correct `byval`/`sret`/`signext` attributes for each argument type on each target
- Splitting or merging aggregate types when the ABI requires it (e.g., two-register struct passing on AMD64)
- Inserting explicit `alloca` + store sequences for callers of `byval` parameters
- Handling `__attribute__((ms_abi))`, `__attribute__((sysv_abi))`, `__attribute__((pcs(...)))`, and similar cross-ABI annotations

This is covered in depth in [Chapter 41 — Calls, the ABI Boundary, and Builtins](../part-06-clang-codegen/ch41-calls-abi-boundary-builtins.md).

---

## 23.7 `sret`, `byval`, and `byref`: The Struct-Passing Nuances

When a C function returns a large struct or accepts a large struct argument, the hardware calling convention almost never allows passing the entire struct in registers. The three attributes `sret`, `byval`, and `byref` are LLVM IR's mechanism for describing the three patterns that actually occur.

### 23.7.1 `sret(T)` — Struct Return

The C declaration:

```c
struct Point { int x; int y; };
struct Point make_point(int x, int y);
```

On x86-64 SysV ABI, a two-integer struct fits in `rax:rdx` and is returned in registers — no `sret` is needed. But on many other targets, or for larger structs, the struct does not fit in the return registers. The ABI standard (both SysV and MSVC) handles this by having the *caller* allocate space and pass a pointer to that space as a hidden first argument; the callee writes the return value there and returns `void` at the machine level.

In LLVM IR, this hidden pointer is modeled with the `sret(T)` attribute:

```llvm
; struct Point { int x, y, z; }  — 12 bytes, returned by pointer
%struct.Point = type { i32, i32, i32 }

; Caller perspective:
define void @caller() {
entry:
  ; Caller allocates the return-value slot
  %retval = alloca %struct.Point, align 4

  ; The sret argument is the hidden first argument
  call void @make_point(ptr sret(%struct.Point) align 4 %retval,
                        i32 1, i32 2, i32 3)

  ; Now load from the allocated slot
  %x = load i32, ptr %retval, align 4
  ret void
}

; Callee perspective:
define void @make_point(ptr sret(%struct.Point) align 4 %result,
                        i32 %x, i32 %y, i32 %z) {
entry:
  ; Write directly into the caller-allocated slot
  %xp = getelementptr inbounds %struct.Point, ptr %result, i32 0, i32 0
  store i32 %x, ptr %xp, align 4
  %yp = getelementptr inbounds %struct.Point, ptr %result, i32 0, i32 1
  store i32 %y, ptr %yp, align 4
  %zp = getelementptr inbounds %struct.Point, ptr %result, i32 0, i32 2
  store i32 %z, ptr %zp, align 4
  ret void
}
```

Key constraints on `sret`:
- The `sret` parameter must be the first parameter.
- The function's IR return type must be `void`.
- The callee is required to write into the `sret` slot before returning; the caller may not read from the slot before the call returns.
- The pointer must remain valid through the duration of the call (lifetime semantics are those of the caller's frame).

The optimizer recognizes the `sret` pattern and can promote it: if the `sret` slot is an `alloca` that is only used by this call and a subsequent load, `SROA` can eliminate the alloca and reconstruct the struct from individual field values.

### 23.7.2 `byval(T)` — Pass by Value (Copy Semantics)

When a large struct is passed by value in C, the semantics are that the callee receives its own private copy. In LLVM IR, the caller passes a *pointer* to the struct, and the `byval(T)` attribute instructs the backend to copy the struct to the stack before making the call:

```llvm
%struct.Data = type { i64, i64, i64, i64 }  ; 32 bytes

; Caller holds a pointer to an existing struct.Data
; The byval attribute causes the backend to copy it to the stack
define void @caller(ptr %data_ptr) {
entry:
  call void @process_data(ptr byval(%struct.Data) align 8 %data_ptr)
  ret void
}

; Callee receives a pointer to its own copy — it may freely modify it
define void @process_data(ptr byval(%struct.Data) align 8 %data) {
entry:
  ; %data is a pointer to a fresh copy; modifying it doesn't affect the caller
  store i64 42, ptr %data, align 8
  ret void
}
```

The copy semantics of `byval` have important implications:

1. **Who performs the copy.** At the IR level, `byval` declares the intent; the backend performs the actual copy (typically by emitting a `memcpy` or inline byte-by-byte copy in the call sequence). The caller's original data is not modified.

2. **The T in `byval(T)`.** The type `T` records the type of the value being copied. It is required — `byval` without a type is a parse error in LLVM 22. The alignment is specified separately via `align N`.

3. **Relationship to the C abstract machine.** In C, passing a struct by value means the callee gets a copy. Clang maps this to `byval` on x86-32 where the struct is too large for registers. On x86-64 SysV, the ABI rules for small structs (up to 16 bytes, depending on fields) may instead split the struct into register-sized pieces and pass each piece in a separate integer or floating-point register, *without* using `byval` at the IR level. The IR-level `byval` attribute does not mean the same thing as C-level pass-by-value; it means a specific calling convention pattern.

4. **Optimization.** If the `byval` argument is provably not modified by the callee (which requires the callee to have `memory(argmem: read)` or similar), `byval` can be optimized away: the backend skips the copy and passes the original pointer directly, reinterpreting the attribute as `byref`. This optimization is performed by the `byval-to-register` pass when applicable.

### 23.7.3 `byref(T)` — Pass by Reference (No Copy)

`byref(T)` is a newer attribute introduced to model Swift's `inout` parameters. Like `byval`, it describes a parameter that is a pointer to a `T`. Unlike `byval`, no copy is made:

```llvm
; Swift inout: no copy; callee may read and write the original
define void @swift_mutate(ptr byref(%struct.S) align 8 %s) {
entry:
  ; %s is the original data — writes are visible to the caller
  store i32 99, ptr %s, align 8
  ret void
}
```

The distinction from a plain `ptr` parameter is that `byref` informs the optimizer about the parameter passing protocol:
- The callee will not escape the pointer (similar to `nocapture`)
- The memory was allocated by the caller (lifetime constraints)
- The type `T` records the intended ABI layout even when opaque pointers obscure the type at the IR level

### 23.7.4 `byval` in IR vs. `byval` in the C ABI: The Name Collision

A persistent source of confusion is that the LLVM IR `byval` attribute and the concept of "pass by value" in a C ABI are related but not identical.

Consider a C function that takes a `struct S` by value on different targets:

| Target | C: `f(struct S s)` | LLVM IR representation |
|--------|---------------------|------------------------|
| x86-32 (struct > 4 bytes) | Copy to stack, callee cleans | `ptr byval(%struct.S)` |
| x86-64 SysV (struct ≤ 16 bytes) | Split into register halves | Separate `i64` or `double` parameters, no `byval` |
| x86-64 SysV (struct > 16 bytes) | Pointer to caller copy | `ptr byval(%struct.S)` |
| AArch64 AAPCS64 (HFA ≤ 4 floats) | Passed in float registers | `float, float, ...` parameters, no `byval` |
| AArch64 AAPCS64 (non-HFA, > 16 bytes) | Pointer to caller copy on stack | `ptr byval(%struct.S)` |
| WASM (struct any size) | Lowered to pointer + memcpy by Clang | `ptr byval(%struct.S)` |

The IR `byval` attribute specifically models the pattern where the caller makes a copy and passes a pointer to that copy; the copy-making is the backend's responsibility at lowering time. For targets where the ABI instead decomposes a struct into its fields and passes them in separate registers, Clang generates separate IR parameters rather than using `byval`. The choice of which IR representation to use is made in [`clang/lib/CodeGen/TargetInfo.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/TargetInfo.cpp) in the target-specific `ABIInfo` subclasses.

This distinction matters when writing a pass that needs to reason about the ABI cost of a call: the presence or absence of `byval` is not a reliable indicator of whether the argument is passed on the stack, because the answer is target-dependent and may be determined entirely by how Clang chose to represent the argument.

### 23.7.5 `inalloca(T)` — In-Place Allocation for Non-Trivial Destructors

A fourth struct-passing attribute, `inalloca(T)`, exists to handle the x86-32 requirement that C++ objects with non-trivial constructors or destructors be constructed in place on the argument stack, not in a temporary. The `inalloca` attribute marks a pointer that was obtained from an `alloca` that was deliberately placed in the outgoing argument area of the stack frame. This ensures the object's constructor and destructor see the same memory location that the callee will receive as its parameter. `inalloca` is primarily a Clang / x86-32 implementation concern; most new code targets x86-64 where the MSVC ABI handles non-trivial structs via `byval` with explicit lifetime.

---

## 23.8 Practical Examples

### 23.8.0 The `noundef` Attribute and Optimizer Implications

The `noundef` attribute deserves special attention because its introduction in LLVM 15–16 materially changed how the optimizer handles function boundaries. Before `noundef`, any argument or return value at a call boundary was legally `undef` or `poison` from the optimizer's perspective. This meant that analyses like SCCP (Sparse Conditional Constant Propagation) and GVN had to conservatively assume that a value returning from a call might be poison, even when the source language guaranteed it was not.

With `noundef`, a function declaration asserts: *this value will never be undef or poison at this boundary*. The optimizer is then allowed to propagate the value as a defined quantity. Consider:

```llvm
; Without noundef — optimizer cannot assume the return value is defined
declare i32 @get_count()

; With noundef — optimizer can CSE and propagate freely
declare noundef i32 @get_count_safe()
```

If `@get_count_safe` is called twice in the same basic block with no intervening stores, the optimizer can replace the second call result with the first — but only if `noundef` is present on the return. Without it, even a `memory(none)` function could theoretically return poison, and the optimizer must preserve the second call in case the first result was poison (since using a poison value twice is not necessarily equivalent to using it once in contexts involving control flow).

Clang emits `noundef` on return values and parameters for all C/C++ functions that carry a well-defined value under the language standard. Notably, it is *not* emitted for `_Bool` parameters when the function is called with an indeterminate value (an area where Clang was historically imprecise), but modern Clang 22 is aggressive about emitting `noundef` wherever the language guarantees a defined value.

### 23.8.1 Verifying Attribute Emission from Clang

To see how Clang translates C function declarations to LLVM IR attributes:

```bash
# Produce human-readable IR with attribute groups
clang -O2 -emit-llvm -S -o - test.c | grep -A5 "define\|attributes"
```

```c
// test.c
#include <stdlib.h>
__attribute__((noinline)) __attribute__((cold))
void error_path(const char *msg) {
    fprintf(stderr, "%s\n", msg);
    exit(1);
}

int compute(int x, int y) __attribute__((pure));
```

```llvm
; Resulting IR (schematic; actual output depends on target triple)
define cold void @error_path(ptr noundef %msg) #0 {
  ...
}

; compute is declared, not defined here, but if defined:
; the 'pure' attribute maps to memory(read) in LLVM 22
declare noundef i32 @compute(i32 noundef %x, i32 noundef %y) #1

attributes #0 = { cold noinline nounwind uwtable "frame-pointer"="all" }
attributes #1 = { nounwind memory(read) }
```

### 23.8.2 Reading Attribute Effects in `opt` Pipelines

The `FunctionAttrs` pass infers attributes that Clang did not emit:

```bash
# Run function attribute inference on IR
/usr/lib/llvm-22/bin/opt -passes=function-attrs -S input.ll -o inferred.ll
```

Typical inferences: `nosync`, `nounwind`, `memory(none)` or `memory(read)` for leaf functions that provably do not escape their arguments or perform IO.

### 23.8.3 Calling Convention in Optimized IR

```llvm
; Internal recursive function promoted to fastcc by the optimizer
define internal fastcc i64 @fib(i64 %n) #0 {
entry:
  %cmp = icmp ult i64 %n, 2
  br i1 %cmp, label %base, label %recurse

base:
  ret i64 %n

recurse:
  %n1 = add nuw nsw i64 %n, -1
  %r1 = tail call fastcc i64 @fib(i64 %n1)
  %n2 = add nuw nsw i64 %n, -2
  %r2 = tail call fastcc i64 @fib(i64 %n2)
  %sum = add nuw i64 %r1, %r2
  ret i64 %sum
}

attributes #0 = { nounwind memory(none) }
```

The `tail call fastcc` makes this eligible for tail call optimization when the recursion is in tail position (the single-step case here is not, but it illustrates the form).

### 23.8.4 Inspecting Calling Convention Lowering

The SelectionDAG dumper shows how IR calling conventions map to physical registers. Given a simple function:

```llvm
; input.ll
define i32 @add(i32 %a, i32 %b) {
  %sum = add i32 %a, %b
  ret i32 %sum
}
```

Running through the SelectionDAG with debug output:

```bash
/usr/lib/llvm-22/bin/llc -mtriple=x86_64-linux-gnu \
    -debug-only=isel input.ll -o /dev/null 2>&1 | head -50
```

The output shows the formal argument lowering: `%a` is assigned to `SDValue(CopyFromReg, 0)` referencing physical register `$edi`, `%b` to `$esi`, and the return value is placed in `$eax`. The `CC_X86_64_C` TableGen-generated function is what makes these assignments; see [`llvm/lib/Target/X86/X86CallingConv.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/X86/X86CallingConv.td) for the human-readable source.

For AArch64 under the same test, `%a` would map to `$w0`, `%b` to `$w1`, and the return to `$w0`, following the AAPCS64 convention defined in [`llvm/lib/Target/AArch64/AArch64CallingConv.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AArch64/AArch64CallingConv.td).

### 23.8.5 Cross-Language ABI Compatibility

One of the most common ABI portability issues arises when mixing C, C++, and Rust in a single binary. All three compile to LLVM IR and ultimately share the same `ccc` calling convention for `extern "C"` or `#[no_mangle]` functions. However, the IR attributes that Clang and rustc emit may differ:

- Rustc emits `noundef` aggressively (Rust's type system guarantees no uninitialized values for most types).
- Clang may not emit `noundef` on a parameter if the C declaration is imprecise about the range of valid values.
- Both compilers agree on `nounwind` for functions that cannot throw.

The key rule: **attribute disagreement at a call boundary is safe as long as the stricter set is respected at runtime**. If the callee is declared `nonnull` in the IR but the caller passes null, the behavior is undefined regardless of which language compiled each side. The attribute is a promise, not a check.

For C++ exception interoperability with Rust (`extern "C++"` linkage), the calling convention remains `ccc` but the unwinding mechanism requires that both sides agree on the unwind format. LLVM 22 uses Itanium EH on Linux/macOS and SEH on Windows; Rust targets the same formats, making interop tractable without special glue.

---

## The Unified `memory(...)` Attribute: C++ API and Migration

Section 23.1.1 introduced the `memory(...)` attribute syntax for the IR text form. This section covers the underlying C++ API, the full location×access matrix, and the migration path from the pre-LLVM-16 scattered memory attributes.

### Background: The Legacy Scattered Attributes

Before LLVM 16, memory effects were expressed by a collection of independent function attributes:

| Legacy attribute | Meaning |
|-----------------|---------|
| `readnone` | No memory access whatsoever |
| `readonly` | Only reads; no writes to any memory |
| `writeonly` | Only writes; no reads |
| `argmemonly` | Accesses only memory pointed to by arguments |
| `inaccessiblememonly` | Accesses only memory inaccessible to the caller |
| `inaccessiblemem_or_argmemonly` | Union of the two above |

These attributes were Boolean flags — either all-or-nothing per access kind — and could not express, for example, "reads argument memory but writes only inaccessible memory." They were also mutually exclusive in confusing ways (`argmemonly` plus `readonly` was valid; `argmemonly` plus `writeonly` was also valid; their interaction was a documented special case).

LLVM 16 replaced all of them with `memory(location: access, ...)` and removed the legacy forms from the IR syntax and the `AttributeList` API. The old attribute names are now rejected by `llvm-as`.

### The Location × Access Matrix

The `memory(...)` attribute encodes a 3×2 matrix: three *location kinds* crossed with two *access kinds* (read and write), independently per location.

**Location kinds** (defined in `llvm/include/llvm/Support/ModRef.h` as `IRMemLocation`):

| Location | `IRMemLocation` enumerator | What it covers |
|----------|--------------------------|----------------|
| `argmem` | `IRMemLocation::ArgMem` | Memory reachable through pointer arguments to the function |
| `inaccessiblemem` | `IRMemLocation::InaccessibleMem` | Memory that the caller cannot observe (e.g., a private allocator's internal slab, `errno`) |
| `other` | `IRMemLocation::Other` | All other globally-reachable memory not accessed via arguments |

**Access kinds** (`ModRefInfo`): `read` (Ref), `write` (Mod), `readwrite` (ModRef), `none` (NoModRef).

The full attribute syntax covers all combinations:

```llvm
; No memory access at all
declare i32 @pure_compute(i32 %x) memory(none)

; Reads any memory, writes nothing (replaces readonly)
declare i32 @read_global(ptr %p) memory(read)

; Only reads argument memory (e.g., strlen)
declare i64 @strlen_like(ptr %s) memory(argmem: read)

; Reads globals, writes only errno/internal state (inaccessible)
declare i32 @libc_func(i32 %x) memory(other: read, inaccessiblemem: write)

; Reads and writes argument memory, doesn't touch globals
declare void @sort_in_place(ptr %arr, i64 %n) memory(argmem: readwrite)

; Writes inaccessible memory (custom allocator tracking),
; reads argument memory (the size argument is a value, not a pointer,
; but imagining a cookie pointer):
declare ptr @allocate(i64 %sz) memory(argmem: read, inaccessiblemem: write)
```

The shorthand `memory(read)` means "read from any location" (all three location kinds), i.e., `memory(argmem: read, inaccessiblemem: read, other: read)`. Similarly, `memory(write)` means write to all locations.

### The `MemoryEffects` C++ API

The `memory(...)` attribute is represented in the C++ API by `llvm::MemoryEffects`, defined in [`llvm/include/llvm/Support/ModRef.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/ModRef.h). It is an alias for `MemoryEffectsBase<IRMemLocation>`.

```cpp
#include "llvm/Support/ModRef.h"
using namespace llvm;

// --- Query ---
Function *F = /* ... */;
MemoryEffects ME = F->getMemoryEffects();

// Does the function read from argument memory?
bool readsArgs = isModOrRefSet(ME.getModRef(IRMemLocation::ArgMem));

// Does the function access any memory at all?
bool noMem = ME.doesNotAccessMemory();   // equivalent to memory(none)

// Does the function only read?
bool readOnly = ME.onlyReadsMemory();    // no writes to any location

// Does the function only access argument pointers?
bool argOnly = ME.onlyAccessesArgPointees();

// --- Construction ---
// memory(none)
MemoryEffects none = MemoryEffects::none();

// memory(read) — all locations read
MemoryEffects readAll = MemoryEffects::readOnly();

// memory(argmem: read)
MemoryEffects argRead = MemoryEffects(IRMemLocation::ArgMem, ModRefInfo::Ref);

// memory(argmem: readwrite, inaccessiblemem: write)
MemoryEffects custom = MemoryEffects::none();
custom = custom.getWithModRef(IRMemLocation::ArgMem,    ModRefInfo::ModRef);
custom = custom.getWithModRef(IRMemLocation::InaccessibleMem, ModRefInfo::Mod);

// Apply to function
F->setMemoryEffects(custom);
```

For querying the inferred memory effects of a call instruction (combining the declared attribute with alias analysis results), use `AAResults::getMemoryEffects(const CallBase *)`:

```cpp
#include "llvm/Analysis/AliasAnalysis.h"

// In a pass that requires AAResults:
AAResults &AA = getAnalysis<AAResultsWrapperPass>().getAAResults();
MemoryEffects callME = AA.getMemoryEffects(&CI);  // CI is a CallInst
```

### `getFnMemoryEffects` for Indirect Calls

For an indirect call (where no `Function*` is available), the alias analysis infrastructure provides `getFnMemoryEffects`:

```cpp
// For a direct call to a known function:
MemoryEffects ME = F->getMemoryEffects();

// For an indirect call via a function pointer: use AAResults
MemoryEffects ME = AA.getMemoryEffects(cast<CallBase>(&I));
```

### Migration Guide: Legacy Attributes → `memory(...)`

| Old IR attribute | New `memory(...)` equivalent |
|-----------------|------------------------------|
| `readnone` | `memory(none)` |
| `readonly` | `memory(read)` |
| `writeonly` | `memory(write)` |
| `argmemonly` | `memory(argmem: readwrite)` |
| `argmemonly readonly` | `memory(argmem: read)` |
| `argmemonly writeonly` | `memory(argmem: write)` |
| `inaccessiblememonly` | `memory(inaccessiblemem: readwrite)` |
| `inaccessiblememonly readonly` | `memory(inaccessiblemem: read)` |
| `inaccessiblemem_or_argmemonly` | `memory(argmem: readwrite, inaccessiblemem: readwrite)` |
| (default — no memory attribute) | `memory(readwrite)` (or omit; same semantics) |

In passes that previously pattern-matched `F->hasFnAttribute(Attribute::ReadNone)`, the replacement is:

```cpp
// Old:
if (F->hasFnAttribute(Attribute::ReadNone)) { ... }

// New:
if (F->getMemoryEffects().doesNotAccessMemory()) { ... }
// Or for read-only:
if (F->getMemoryEffects().onlyReadsMemory()) { ... }
```

The `AttributorPass` automatically upgrades any legacy attributes present in old bitcode files when they are parsed by LLVM 22.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **`preserve_none` broader adoption and expansion**: The `preserve_none` calling convention (CallingConv::PreserveNone, ID 21), introduced in LLVM 17 for GC write barriers and safepoint stubs, is being extended to cover additional runtime call patterns. Active Phabricator reviews target `preserve_none` support on AArch64 (currently only fully supported on x86-64) and discussion on discourse.llvm.org proposes a companion `preserve_most_registers` variant for profiling hooks.
- **RISC-V calling convention stabilization post-ratification**: The RISC-V vector extension (RVV) calling convention for `vscale_range`-annotated functions is being finalized to match the ratified RISC-V EABI specification. The `RISCVCallingConv.td` changes tracking vector register usage (V0–V31) are expected to land in LLVM 22.x point releases, aligning with the GCC 15 implementation.
- **`noundef` completeness for `_Bool` and bit-fields**: Clang 22 still conservatively omits `noundef` on `_Bool` parameters in certain indirect-call scenarios and on bit-field-containing struct parameters. An active RFC (discourse.llvm.org/t/noundef-bool-parameters) proposes extending Clang's `CGCall.cpp` analysis to cover these cases, enabling more aggressive GVN and SCCP across language boundaries.
- **GlobalISel `CCAssignFn` integration**: The GlobalISel path uses a parallel `CallLowering` abstraction that partially duplicates the TableGen `CCAssignFn` tables used by SelectionDAG. An ongoing effort to unify these under a single TableGen-generated description (tracked in LLVM GitHub issue #63279) targets complete unification before LLVM 21 reaches end-of-life, reducing the per-target ABI maintenance burden.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **ABI stability guarantees for `fastcc`-promoted functions under LTO**: Current LTO pipelines can promote internal functions to `fastcc`, but the promotion is not persistently recorded in a way that allows incremental LTO rebuilds to reuse the convention across module cache invalidations. A proposed `fastcc`-sticky bitcode annotation (discussed as part of the ThinLTO roadmap) would allow `fastcc` promotion decisions to survive module recompilation, reducing link-time work for large codebases such as Chrome and Firefox.
- **C23 `_BitInt` ABI standardization across targets**: The C23 arbitrary-precision integer type `_BitInt(N)` is supported in Clang 22 but its ABI (how `byval`/`zeroext`/`signext` apply, how it interacts with struct layout and calling conventions) is not yet standardized across all targets. The AAPCS working group and the x86-64 ABI maintainers (H.J. Lu) have open proposals for `_BitInt` ABI rules; resolution is expected to require coordinated changes to `TargetInfo.cpp`, `AArch64ABIInfo`, and `X86_64ABIInfo`.
- **Capability-aware calling conventions for CHERI/Morello**: The CHERI project (Cambridge) and Arm Morello platform require capability (fat pointer) arguments to be passed in capability registers, not general-purpose registers. A `chericc` calling convention or an extension to `ccc` via capability-aware parameter attributes is under design; a reference implementation exists in the CheriBSD fork of LLVM and is expected to be proposed for upstream inclusion once the Morello hardware ecosystem matures.
- **Swift async calling convention evolution for structured concurrency**: The `swiftasync` attribute and `swifttailcc` convention are central to Swift's async/await ABI. As Swift structured concurrency matures and the Swift runtime evolves its continuation-passing protocol, corresponding IR-level changes to the async context parameter (`swiftasync`) are anticipated, including potential introduction of a `swiftasynccc` distinct from `swifttailcc` to better express continuation ownership semantics.

### 5-Year Horizon (Long-Term, by ~2031)

- **Machine-readable ABI descriptors replacing `CCAssignFn` TableGen tables**: The current `CCAssignFn` mechanism requires per-target hand-written TableGen, which is error-prone and difficult to audit against ISA specifications. A long-term proposal (articulated in the 2024 LLVM Developers' Meeting talk "Towards a Unified ABI Description Language") envisions a formal, machine-readable ABI specification language (similar to ACLE or the RISC-V calling convention specification in RISC-V International's documentation) that generates both `CCAssignFn` tables and verification test cases, enabling automatic conformance checking against hardware reference implementations.
- **Unified IR-level ABI model across LLVM/MLIR boundary**: MLIR's `func.call` operations have no native concept of `byval`, `sret`, or `signext`; these are encoded as dialect attributes or lost during MLIR-to-LLVM lowering. A unified ABI attribute model that survives the MLIR→LLVM IR lowering pass without information loss would enable MLIR-native ABI analysis passes (alias analysis, memory effect modeling) to operate at the same fidelity as LLVM IR-level passes. This is discussed in the MLIR roadmap under "first-class ABI representation."
- **Post-quantum cryptography calling conventions**: As NIST PQC algorithms (CRYSTALS-Kyber, CRYSTALS-Dilithium, now standardized as ML-KEM and ML-DSA) enter hardware acceleration, vendors (ARM, Intel, RISC-V) are proposing ISA extensions with dedicated polynomial multiplication registers. These registers will require new calling convention rules — which argument values live in polynomial registers, what their callee-save obligations are — analogous to how AVX-512 registers required `x86_vectorcallcc` extensions. LLVM will need new `target-features` attributes and potentially new calling convention variants to expose these ABI surfaces to compilers targeting PQC hardware.

---

## 23.9 Chapter Summary

- **Function attributes** encode the optimizer's contract with a function: memory effects (`memory(none/read/write/readwrite)` replacing the legacy `readnone`/`readonly`/`writeonly` as of LLVM 16), control-flow properties (`nounwind`, `noreturn`, `willreturn`, `nosync`), inlining policy (`noinline`, `alwaysinline`, `optnone`), code generation hints (`cold`, `hot`, `naked`), and target-specific overrides (`target-cpu`, `target-features`, `vscale_range`).

- **Parameter and return attributes** annotate individual values: `nonnull` and `noundef` enable more aggressive optimization by ruling out null and poison; `noalias` and `nocapture` constrain pointer aliasing and lifetime; `zeroext`/`signext` encode ABI-required integer extension; `byval`, `byref`, and `sret` express the struct-passing patterns.

- **Call site attributes** apply the same parameter annotations at a specific call, enabling inlining and indirect-call optimization to attach stronger facts than the general declaration carries.

- **Attribute groups** (`#N` syntax) allow Clang to consolidate the attributes of many functions sharing the same properties into a single definition, significantly reducing IR text size without changing semantics.

- **Calling conventions** range from the platform-default `ccc` and optimization-oriented `fastcc`/`coldcc`/`tailcc`, through language-specific conventions (`swiftcc`, `ghccc`, `webkit_jscc`), x86 ABI variants (`stdcall`, `fastcall`, `thiscall`, `vectorcall`, `regcall`), and GPU conventions (`amdgpu_kernel`, `ptx_kernel`, `spir_kernel`). The convention selection determines register assignment and stack layout at the machine level.

- **ABI lowering** is a late, target-specific transformation: the LLVM middle-end optimizer operates on IR-level abstractions (attributes, calling convention mnemonics) without knowledge of register assignments. The backend's `TargetLowering::LowerCall` and the TableGen-generated `CCAssignFn` tables perform the actual mapping to hardware registers and stack slots during instruction selection.

- **`sret(T)`** models large struct returns: the caller allocates, the callee writes, and the IR-level return type is `void`. **`byval(T)`** models stack-copy argument passing: the backend copies the struct before the call; the callee's copy is independent. **`byref(T)`** models reference passing without copy, used for Swift `inout`. The IR-level `byval` attribute does not always correspond to C-level pass-by-value; for many struct sizes and targets, Clang decomposes the struct into register-sized IR parameters instead.

- **The producer/consumer split** is fundamental: Clang (via `CGCall.cpp` and the target `ABIInfo` subclasses) produces the correct IR representation for each target's ABI; the backend (via `TargetLowering` and `CCAssignFn`) consumes that representation and generates hardware-correct call sequences. Neither layer can be correct without the other.


---

@copyright jreuben11
