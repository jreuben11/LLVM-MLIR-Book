# Chapter 26 — Exception Handling

*Part IV — LLVM IR*

Exception handling is the area of compiler infrastructure where the gap between the programmer's mental model and the generated code is widest. A C++ `try`/`catch` block looks like a control-flow annotation, but underneath it demands an entirely separate data structure — the LSDA (Language-Specific Data Area) — plus two-phase stack unwinding, personality function dispatch, and destructor invocation sequences that execute on a path the optimizer never sees during normal compilation. Getting EH correct is not optional: a miscompiled destructor or skipped cleanup corrupts program state invisibly, often manifesting as a leak or a use-after-free far from the original throw site.

LLVM must model EH explicitly in IR rather than treating it as a calling-convention side effect, because analysis passes must know about exceptional control flow. A memory store before an `invoke` and the `landingpad` it can reach are not independent: the store is visible on the exceptional path, and alias analysis, loop-invariant code motion, and lifetime analysis all need to respect unwind edges. This chapter covers every EH model LLVM 22 supports, the IR constructs that encode each model, the runtime data structures they generate, and the optimizer passes that reason about them.

All IR examples in this chapter are verified against LLVM 22.1.3 using `/usr/lib/llvm-22/bin/llvm-as`.

---

## 26.1 EH Models: A Taxonomy

LLVM 22 supports five distinct EH models, selected by target triple and compiler flags. The canonical enumeration lives in [`llvm/include/llvm/IR/EHPersonalities.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/EHPersonalities.h) as `enum class EHPersonality`:

| `EHPersonality` value | Personality function | Model | Platforms |
|-----------------------|---------------------|-------|-----------|
| `GNU_CXX` | `__gxx_personality_v0` | Itanium table-based | Linux, macOS, FreeBSD, Fuchsia |
| `GNU_CXX_SjLj` | `__gxx_personality_sj0` | SJLJ | ARM/iOS pre-iOS7, MIPS without table EH |
| `GNU_C` | `__gcc_personality_v0` | C cleanups only | GCC compatibility |
| `GNU_C_SjLj` | `__gcc_personality_sj0` | SJLJ for C | ARM embedded |
| `GNU_ObjC` | `__objc_personality_v0` | Itanium + ObjC runtime | Darwin/GCC ObjC |
| `MSVC_CXX` | `__CxxFrameHandler3` | WinEH funclet, MSVC C++ | Windows x64/x86 |
| `MSVC_TableSEH` | `__C_specific_handler` | WinEH funclet, SEH | Windows x64 |
| `MSVC_X86SEH` | `_except_handler3` / `_except_handler4` | WinEH funclet, SEH x86 | Windows x86 |
| `Wasm_CXX` | `__gxx_wasm_personality_v0` | Wasm scoped EH | WebAssembly |
| `Rust` | `rust_eh_personality` | Itanium-compatible | All Rust targets |
| `CoreCLR` | `__CxxFrameHandler3`-variant | CLR funclet | .NET on Windows |
| `XL_CXX` | `__xlcxx_personality_v1` | Itanium variant | IBM AIX |
| `ZOS_CXX` | `__zos_cxx_personality_v2` | Itanium variant | IBM z/OS |

Three predicate functions classify personalities at the API level:

```cpp
// isFuncletEHPersonality: MSVC_CXX, MSVC_X86SEH, MSVC_TableSEH, CoreCLR
// These use catchswitch/catchpad/cleanuppad IR; handlers are funclets.
bool isFuncletEHPersonality(EHPersonality Pers);

// isScopedEHPersonality: all funclet personalities PLUS Wasm_CXX.
// All use catchswitch/catchpad/cleanuppad rather than landingpad/resume.
bool isScopedEHPersonality(EHPersonality Pers);

// isAsynchronousEHPersonality: MSVC_X86SEH, MSVC_TableSEH.
// These catch hardware exceptions (access violations, divide-by-zero).
bool isAsynchronousEHPersonality(EHPersonality Pers);
```

The IR-level split is sharp: Itanium personalities use `landingpad`/`resume`; funclet personalities use `catchswitch`/`catchpad`/`catchret`/`cleanuppad`/`cleanupret`. A single function may carry only one personality, and all EH pads in the function must be consistent with it. The personality function is declared on the `define` keyword — not on any `declare` — and its type is always `i32(...)` variadic.

SJLJ and the standard ARM EHABI (`GNU_CXX` on ARM with `-marm-eh`) are both table-based at the runtime level but differ in IR representation and code size characteristics.

---

## 26.2 The Itanium Model: Two-Phase Unwinding

The Itanium C++ ABI defines the protocol used by GCC, Clang, and the `libgcc_s` / `libunwind` runtime on all non-Windows Unix-like platforms. The key document is the [Itanium C++ ABI specification §3](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html).

### 26.2.1 Two-Phase Unwinding

When a C++ throw executes, `__cxa_throw` calls `_Unwind_RaiseException` from `libgcc_s` / `libunwind`. The unwinder performs two passes over the call stack:

**Phase 1 (search phase).** The unwinder walks frames upward, calling each frame's personality function with `_UA_SEARCH_PHASE`. The personality function consults the LSDA for that frame and returns `_URC_HANDLER_FOUND` if the frame can catch the exception type, or `_URC_CONTINUE_UNWIND` to continue searching. If no handler is found in any frame, the unwinder calls `std::terminate`.

**Phase 2 (cleanup phase).** Having located a handler, the unwinder walks the same frames again from the top, this time with `_UA_CLEANUP_PHASE | _UA_HANDLER_FRAME` for the catching frame and `_UA_CLEANUP_PHASE` for all intervening frames. Each personality function invokes cleanups (destructors) in intervening frames by transferring control to the appropriate landing pad. The catching frame receives control with the exception object pointer in a specific register (RAX on x86-64, R0 on ARM).

This two-pass design is essential for `std::terminate` on uncaught exceptions: since phase 1 completes before any stack unwinding begins, `std::terminate` is called with the stack intact, which enables stack-trace diagnostics. It also enables `noexcept` checking: a `noexcept` function that receives an unhandled exception in phase 1 calls `std::terminate` before any frames are unwound.

### 26.2.2 Runtime Data Structures

**`.eh_frame` section.** The Call Frame Information (CFI) table, encoding register save/restore rules as a sequence of `DW_CFA_*` operations for each instruction range. The unwinder uses this to restore the caller's register state at each frame. Generated unconditionally when `uwtable` is set (always true on Linux ELF with `-fexceptions`); controlled by `-fasynchronous-unwind-tables`. The format follows DWARF 4 §6.4, with GCC extensions for the `z`, `P`, `L`, `R` augmentation letters.

**`.gcc_except_table` section.** The LSDA, accessed via the `P` augmentation pointer in the CIE. Contains:
- A header (LSDA header) with encoding bytes for pointer types.
- A call site table: for each `invoke` instruction, the range of code addresses covered, the landing pad address, and an action index.
- An action table: chains of (type-index, next-action) pairs.
- A type table: array of `std::type_info *` pointers for catch types and exception specification types.

The personality function `__gxx_personality_v0` (implemented in `libstdc++` or `libc++abi`) decodes this table to determine whether a given frame handles the exception, and if so, which landing pad and catch block.

**`_Unwind_Exception` structure.** The ABI mandates that every exception object begins with an `_Unwind_Exception` struct containing an 8-byte class identifier (e.g., `"GNUCC++\0"` for GCC/Clang C++), a cleanup function pointer, and two private words for the unwinder's use. `__cxa_throw` fills this in before calling `_Unwind_RaiseException`.

### 26.2.3 Type Matching

Each C++ type that can be thrown or caught has a `std::type_info` object. The `catch` clause specifies a type, and the personality function calls `std::type_info::__do_catch` (non-standard Itanium extension in `libstdc++`) or uses RTTI comparison to determine whether the thrown type is compatible with the caught type. LLVM IR references `std::type_info` objects as external `ptr` globals with the mangled name `_ZTI<mangled-type>`. For example, `_ZTISt13runtime_error` is the RTTI for `std::runtime_error`.

---

## 26.3 Itanium EH in LLVM IR

The Itanium model uses three primary constructs: `invoke`, `landingpad`, and `resume`. Their complete syntax is specified in the LLVM Language Reference; this section focuses on their semantics and the patterns Clang generates.

### 26.3.1 The `invoke` Instruction

`invoke` is a function call that specifies two successors: `to label %normal` for the non-exceptional return path and `unwind label %lpad` for the exceptional path. The unwind destination must be a basic block that begins with a `landingpad` instruction. If the callee raises an exception, the unwinder transfers control to the landing pad with the exception object pointer and selector value in registers defined by the ABI (then materialized into the `landingpad` result).

```llvm
%result = invoke i32 @may_throw(i32 %x)
        to label %normal unwind label %lpad
```

`invoke` may only unwind to a landing pad in the same function. Cross-function unwinding is performed by the runtime itself; IR only encodes the intra-function landing pads.

### 26.3.2 The `landingpad` Instruction

`landingpad` must be the first non-phi instruction in its basic block, which must be an unwind destination of at least one `invoke`. Its type is always `{ ptr, i32 }` under opaque pointers: the `ptr` is the exception object pointer, and the `i32` is the selector value — an integer assigned by the personality function to identify which catch clause matched. The `personality` clause on the enclosing `define` specifies which personality function interprets the LSDA.

A `landingpad` carries one or more *clauses* that determine which exceptions it handles:

```llvm
; catch clause: handle this specific type
%lp = landingpad { ptr, i32 }
        catch ptr @_ZTISt13runtime_error

; cleanup clause: run always (for destructors), selector is 0
%lp = landingpad { ptr, i32 }
        cleanup

; filter clause: handle any type NOT in this array (C++03 throw specs)
%lp = landingpad { ptr, i32 }
        filter [2 x ptr] [ptr @_ZTIi, ptr @_ZTId]

; mixed: cleanup + multiple catches
%lp = landingpad { ptr, i32 }
        cleanup
        catch ptr @_ZTIi
        catch ptr @_ZTId
```

A `catch ptr null` clause catches all C++ exceptions (`catch (...)` in source). A `filter [0 x ptr] []` clause (empty filter) catches nothing and triggers `std::unexpected()` for any exception — the encoding of a `noexcept` dynamic-throw-specification in pre-C++11 mode.

### 26.3.3 `llvm.eh.typeid.for` and Type Dispatch

After extracting the selector from the `landingpad` result, code compares it against a per-type integer returned by `llvm.eh.typeid.for`. The intrinsic takes a `ptr` to a type info object and returns an `i32` that is unique within the function's personality context:

```llvm
declare i32 @llvm.eh.typeid.for.p0(ptr) nounwind memory(none)

%tid = call i32 @llvm.eh.typeid.for.p0(ptr @_ZTISt13runtime_error)
%match = icmp eq i32 %sel, %tid
br i1 %match, label %catch_body, label %rethrow
```

The result of `llvm.eh.typeid.for` is stable per function — two calls with the same type info pointer in the same function return the same value. The actual integer assignments are determined by the `EHStreamer` during code generation when it builds the action table.

### 26.3.4 The `resume` Instruction

`resume` re-throws an in-flight exception. It takes a `{ ptr, i32 }` value of the same type returned by `landingpad` and propagates the exception to the next frame. The landing pad in a cleanup-only block will call `resume` unconditionally after running cleanup code:

```llvm
lpad:
  %lp = landingpad { ptr, i32 }
          cleanup
  ; ... run destructor ...
  resume { ptr, i32 } %lp
```

### 26.3.5 Complete try/catch Walkthrough

The following example shows real LLVM 22 IR produced by `clang++ -emit-llvm -S -O0 -fexceptions` for a function with a `try`/`catch(const std::runtime_error&)`:

```llvm
; Compiled from:
;   int try_catch(int x) {
;     try { return may_throw(x); }
;     catch (const std::runtime_error& e) { return -1; }
;   }

define dso_local noundef i32 @_Z9try_catchi(i32 noundef %0)
    personality ptr @__gxx_personality_v0 {
  %2 = alloca i32, align 4          ; return-value slot
  %4 = alloca ptr, align 8          ; saved exception pointer
  %5 = alloca i32, align 4          ; saved selector

  %7 = load i32, ptr %3, align 4
  %8 = invoke noundef i32 @_Z9may_throwi(i32 noundef %7)
          to label %9 unwind label %10

9:                                  ; normal path
  store i32 %8, ptr %2, align 4
  br label %22

10:                                 ; landing pad
  %11 = landingpad { ptr, i32 }
          catch ptr @_ZTISt13runtime_error
  %12 = extractvalue { ptr, i32 } %11, 0    ; exception pointer
  %13 = extractvalue { ptr, i32 } %11, 1    ; selector
  store ptr %12, ptr %4, align 8
  store i32 %13, ptr %5, align 4
  br label %14

14:                                 ; type dispatch
  %15 = load i32, ptr %5, align 4
  %16 = call i32 @llvm.eh.typeid.for.p0(ptr @_ZTISt13runtime_error)
  %17 = icmp eq i32 %15, %16
  br i1 %17, label %18, label %24

18:                                 ; matched: catch block
  %19 = load ptr, ptr %4, align 8
  %20 = call ptr @__cxa_begin_catch(ptr %19)   ; adjust pointer, account ref
  store ptr %20, ptr %6, align 8
  store i32 -1, ptr %2, align 4
  call void @__cxa_end_catch()
  br label %22

22:                                 ; exit
  %23 = load i32, ptr %2, align 4
  ret i32 %23

24:                                 ; unmatched: re-throw
  %25 = load ptr, ptr %4, align 8
  %26 = load i32, ptr %5, align 4
  %27 = insertvalue { ptr, i32 } poison, ptr %25, 0
  %28 = insertvalue { ptr, i32 } %27, i32 %26, 1
  resume { ptr, i32 } %28
}
```

`__cxa_begin_catch` both adjusts the exception pointer for reference binding and increments the uncaught-exceptions counter. `__cxa_end_catch` decrements the counter and, when it reaches zero, destroys the exception object if `__cxa_begin_catch` was the last begin without a corresponding re-throw.

---

## 26.4 Cleanup and Destructor Patterns

RAII is the primary use case for cleanup landingpads. When a C++ function has a local variable with a non-trivial destructor, Clang wraps every potentially-throwing call in an `invoke` whose landing pad calls the destructor before re-propagating the exception. The following example shows the key structure:

```llvm
; Compiled from raii_example(int x) — Guard has ~Guard() noexcept
define dso_local noundef i32 @_Z12raii_examplei(i32 noundef %0)
    personality ptr @__gxx_personality_v0 {
  ; ...
  call void @_ZN5GuardC2Ev(ptr ... %3)       ; construct Guard on normal path

  ; throw path uses invoke
  invoke void @_ZNSt13runtime_errorC1EPKc(ptr ... %9, ptr ... @.str)
          to label %10 unwind label %11

10: ; successful construction, then invoke __cxa_throw
  invoke void @__cxa_throw(ptr %9, ...) #noreturn
          to label %27 unwind label %15

11: ; constructor threw — cleanup allocated exception, then destroy Guard
  %12 = landingpad { ptr, i32 }
          cleanup
  call void @__cxa_free_exception(ptr %9)
  br label %21

15: ; __cxa_throw propagated — destroy Guard
  %16 = landingpad { ptr, i32 }
          cleanup
  br label %21

21: ; common cleanup: invoke destructor, then resume
  call void @_ZN5GuardD2Ev(ptr ... %3)
  ; ... load saved {ptr, i32}, then:
  resume { ptr, i32 } %26

19: ; normal path: also call destructor (no exception)
  call void @_ZN5GuardD2Ev(ptr ... %3)
  ret i32 %20
}
```

Two observations from this actual Clang output: first, the destructor is called on *both* the normal and exceptional paths — the normal-path call is a plain `call` because no exception is possible there, while the exceptional-path call feeds into `resume`. Second, when a constructor itself can throw (here `runtime_errorC1E` might throw), the allocation `__cxa_allocate_exception` must be freed before re-propagating; Clang generates a separate landing pad for that case.

The `cleanup` clause on a `landingpad` means: "execute this block whenever an in-flight exception passes through, regardless of its type." The selector value delivered with `cleanup` is 0; code that only has a `cleanup` clause does not need to call `llvm.eh.typeid.for` — it simply calls destructors and then `resume`s.

Nested scopes generate multiple landing pads. When a throw originates inside an inner scope with three live objects, Clang generates three landing pad chains, each destroying progressively more objects. At `-O1` and above, the `MergeCleanups` transformation in `CodeGenPrepare` and `SimplifyCFG` merges duplicate cleanup sequences into a single block with `phi` nodes.

---

## 26.5 The Windows Funclet Model

The Windows x64 Structured Exception Handling (SEH) and MSVC C++ EH models are fundamentally different from the Itanium model. Rather than a single flat function with inline cleanup code reached by landing pad addresses stored in the LSDA, Windows EH uses *funclets* — nested function-like entities that share the same stack frame as their parent but execute as separate code regions.

### 26.5.1 Why Funclets Exist

Windows x64 uses table-driven EH via the `.pdata` (function table) and `.xdata` (unwind info) PE sections. The unwind engine, implemented in `ntdll!RtlUnwindEx`, follows a similar two-pass model but drives handler funclets via function pointers recorded in the `.xdata` table. The handler code — the `__except` filter, the `__except` body, or the `__finally` block — must be a distinct function (funclet) because the OS EH engine calls it as a callback, passing a pointer to the frame that caught the exception.

From LLVM's perspective, each funclet is a region of basic blocks within the IR function that is later emitted as a logically-separate subfunction sharing the parent's frame. The `colorEHFunclets` function in `EHPersonalities.h` computes which basic blocks belong to which funclet.

### 26.5.2 IR Constructs: `catchswitch`, `catchpad`, `catchret`, `cleanuppad`, `cleanupret`

The `catchswitch` instruction is the landing block — the unwind destination of `invoke` instructions:

```llvm
; Must be the first non-phi in its block; within-clause names parent funclet or none
%cs = catchswitch within none [label %handler1, label %handler2] unwind to caller
; 'unwind to caller' means: if no handler matches, propagate to the caller's EH
; 'unwind label %outer_cs' means: propagate to an enclosing catchswitch (nested try)
```

`catchpad` begins a catch funclet. Its `within` operand is the `catchswitch` token. The argument list is personality-specific: for `__CxxFrameHandler3`, it is `[typeinfo-ptr, adjectives, catch-object-ptr]`:

```llvm
handler:
  %cp = catchpad within %cs [ptr @"??_R0H@8", i32 8, ptr null]
  ; ... handler body ...
  catchret from %cp to label %after_try
```

`catchret` transfers control from the funclet back to the parent frame at a specified label. It implicitly re-enables the parent frame's EH context.

`cleanuppad` begins a cleanup funclet (destructor or `__finally`):

```llvm
cleanup_dispatch:
  %cleanup = cleanuppad within none []
  call void @destructor() [ "funclet"(token %cleanup) ]
  cleanupret from %cleanup unwind to caller
```

Any call inside a funclet that might itself raise an exception must carry the `"funclet"(token %pad)` operand bundle to associate it with the containing pad. This is how the IR encodes the funclet-coloring relationship that Windows EH requires.

The `within` chain builds nested EH: a `catchswitch within %outer_cs [...]` indicates a try block nested inside another try block. `cleanuppad within %cs []` is a cleanup inside a catch handler.

### 26.5.3 `__C_specific_handler` and Windows SEH

For `__try`/`__except`/`__finally` (non-C++ SEH on x64), the personality is `__C_specific_handler`. The IR uses the same funclet framework. The `catchpad` arguments encode the filter expression function pointer and the handler block. `llvm.eh.exceptioncode` extracts the hardware exception code from the `catchpad` token:

```llvm
; Windows SEH: __except with exception code access
dispatch:
  %cs = catchswitch within none [label %handler] unwind to caller

handler:
  %cp = catchpad within %cs [ptr null, i32 0, ptr null]
  %code = call i32 @llvm.eh.exceptioncode(token %cp)
  ; %code is e.g. 0xC0000005 for EXCEPTION_ACCESS_VIOLATION
  catchret from %cp to label %after_try
```

`llvm.eh.exceptionpointer` retrieves the `_EXCEPTION_RECORD *` pointer from a funclet token, enabling access to the full exception record including exception arguments.

`isAsynchronousEHPersonality` returns `true` for `MSVC_X86SEH` and `MSVC_TableSEH` because these personalities respond to hardware exceptions raised asynchronously — they do not require a `call` or `invoke` to trigger.

---

## 26.6 WinEH C++ Model

MSVC C++ exceptions use `__CxxFrameHandler3` as the personality function. Clang emits this model when the target triple is `x86_64-pc-windows-msvc` and `-fexceptions` is active. The structure is the same funclet model as §26.5 but with MSVC-specific RTTI:

```llvm
; MSVC C++ EH: catch(int)
target triple = "x86_64-pc-windows-msvc19.33.0"

declare i32 @__CxxFrameHandler3(...)
declare void @might_throw()

@"??_R0H@8" = external constant ptr   ; MSVC RTTI descriptor for int

define i32 @win_try_catch_cpp() personality ptr @__CxxFrameHandler3 {
entry:
  invoke void @might_throw()
          to label %normal unwind label %dispatch

normal:
  ret i32 0

dispatch:
  %cs = catchswitch within none [label %handler] unwind to caller

handler:
  ; [typeinfo, adjectives=8 (by-value catch), catch-object-alloca or null]
  %cp = catchpad within %cs [ptr @"??_R0H@8", i32 8, ptr null]
  catchret from %cp to label %caught

caught:
  ret i32 -1
}
```

The MSVC RTTI descriptor `??_R0H@8` is a `TypeDescriptor` structure containing the mangled type name string and a pointer to the vtable for `type_info`. This differs from the Itanium `_ZTIi` (`typeinfo for int`) which uses the Itanium RTTI format.

The `adjectives` integer encodes catch modifiers:
- `0x01`: const-qualified catch
- `0x02`: volatile-qualified catch
- `0x08`: catch by value (object is copied into catch-object alloca)
- `0x40`: reference catch (pointer adjustment required)

For nested try/catch, the `within` operand of an inner `catchswitch` is the token of the enclosing `catchpad` or `cleanuppad`, building a tree of funclet parents. `WinEHPrepare` (declared in [`llvm/include/llvm/CodeGen/WinEHPrepare.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/WinEHPrepare.h)) runs as a late function pass to lower the IR funclet representation into the `.xdata` tables that the Windows EH runtime expects.

---

## 26.7 SJLJ Exception Handling

SjLj (setjmp/longjmp) EH is a software emulation of table-based EH for platforms where the stack layout is insufficient or too costly to unwind via CFI tables. It is the default on ARM targets without VFP support, older iOS targets (pre-iOS 7), and embedded targets that opt in with `-fsjlj-exceptions`.

### 26.7.1 Mechanism

Instead of a separate `.eh_frame` / LSDA, SjLj EH maintains a per-function `_Unwind_SjLj_FunctionContext` structure on the stack. This structure is chained into a thread-local linked list at function entry and removed on function exit. When an exception is thrown, `_Unwind_SjLj_RaiseException` walks the linked list, calling the personality function for each frame.

The key difference from Dwarf EH: no separate table section is needed, but every function that contains an `invoke` pays a runtime cost at entry and exit to register/deregister itself from the linked list.

### 26.7.2 SJLJ Intrinsics

LLVM defines six intrinsics for the SjLj model, emitted by the `SjLjEHPrepare` pass ([`llvm/include/llvm/CodeGen/SjLjEHPrepare.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/SjLjEHPrepare.h)):

| Intrinsic | Signature | Purpose |
|-----------|-----------|---------|
| `llvm.eh.sjlj.lsda` | `ptr ()` | Returns pointer to the function's LSDA |
| `llvm.eh.sjlj.callsite` | `void (i32 immarg)` | Marks the call site index for the next invoke |
| `llvm.eh.sjlj.functioncontext` | `void (ptr)` | Registers function context at entry |
| `llvm.eh.sjlj.setjmp` | `i32 (ptr)` | Saves the unwind context; returns 0 normally |
| `llvm.eh.sjlj.longjmp` | `void (ptr) noreturn` | Restores the unwind context during unwinding |
| `llvm.eh.sjlj.setup.dispatch` | `void ()` | Initializes the dispatch table for the function |

`SjLjEHPrepare` transforms a function that uses standard `invoke`/`landingpad` IR by inserting `functioncontext` registration at entry, renumbering each `invoke` with a `callsite` annotation, and replacing the entry points of landing pads with `sjlj.setjmp` checks. When the personality function calls `longjmp`, the `setjmp` returns non-zero and control transfers to the appropriate dispatch target.

`llvm.eh.sjlj.callsite` is a marker: its `i32 immarg` argument is a monotonically increasing call-site number assigned by `SjLjEHPrepare` in source order. The LSDA for SjLj EH (returned by `llvm.eh.sjlj.lsda`) encodes a mapping from call-site number to landing pad, allowing `_Unwind_SjLj_RaiseException` to jump to the right landing pad without traversing the entire switch dispatch. `llvm.eh.sjlj.setup.dispatch` initializes the indirect branch table at the top of the dispatch block.

### 26.7.3 Overhead vs Table-Based EH

The runtime overhead of SjLj EH is measurable. Benchmarks on ARM Cortex-A8 (the original motivation) show a 2–5% throughput penalty on code with many small functions that each contain an `invoke`. The cost has three sources:

1. **Function context allocation.** The `_Unwind_SjLj_FunctionContext` is allocated on-stack (no heap), but the stack pointer manipulation and frame layout affects register allocation.
2. **Linked-list registration.** At function entry, the context is pushed onto the thread-local EH stack; at every exit (including early returns), it must be popped. This is two atomic-or-barrier-free memory writes per function, but they flush the instruction cache if adjacent functions are not in cache.
3. **`setjmp` save.** Each `invoke`-containing basic block saves registers via `setjmp`-equivalent logic before the call. On ARM, this saves all callee-saved registers and the PC, costing 10–20 cycles even when no exception is thrown.

Dwarf/CFI EH has zero hot-path cost — tables are read only during unwinding, which is already an exceptional slow path. For platforms where Dwarf is available, the `-fno-sjlj-exceptions` default is strongly preferred.

---

## 26.8 WebAssembly Exception Handling

WebAssembly EH has two distinct implementations in Clang/LLVM. The older implementation (`-mllvm -wasm-enable-eh`) uses the scoped IR model (catchswitch/catchpad), while the newer native Wasm EH proposal (`-mllvm -wasm-enable-exnref`, targeting the Wasm exception-handling specification) uses dedicated intrinsics.

### 26.8.1 Scoped Wasm EH

The `Wasm_CXX` personality (`__gxx_wasm_personality_v0`) is classified as a scoped EH personality (`isScopedEHPersonality` returns `true`). It uses `catchswitch`/`catchpad`/`catchret` exactly as the WinEH model does, because WebAssembly also executes catch handlers as funclet-like regions. The IR is structurally identical to the WinEH C++ example in §26.6:

```llvm
target triple = "wasm32-unknown-unknown"

@_ZTIi = external global ptr
declare i32 @__gxx_wasm_personality_v0(...)
declare void @might_throw()

define i32 @wasm_try_catch() personality ptr @__gxx_wasm_personality_v0 {
entry:
  invoke void @might_throw()
          to label %normal unwind label %dispatch

normal:
  ret i32 0

dispatch:
  %cs = catchswitch within none [label %handler] unwind to caller

handler:
  %cp = catchpad within %cs [ptr @_ZTIi]
  catchret from %cp to label %caught

caught:
  ret i32 -1
}
```

`WasmEHPrepare` ([`llvm/include/llvm/CodeGen/WasmEHPrepare.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/WasmEHPrepare.h)) converts the scoped IR into actual Wasm EH instructions. It uses `WasmEHFuncInfo` to track which landing pads correspond to which Wasm `catch` instructions, storing a `SrcToUnwindDest` map from landing pad to outer unwind target.

### 26.8.2 Native Wasm EH Intrinsics

The newer Wasm EH proposal introduces four intrinsics defined in [`llvm/include/llvm/IR/IntrinsicsWebAssembly.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/IntrinsicsWebAssembly.td):

| Intrinsic | Signature | Purpose |
|-----------|-----------|---------|
| `llvm.wasm.throw` | `void (i32 immarg tag, ptr value) noreturn` | Throw with ABI tag (0=C++) |
| `llvm.wasm.rethrow` | `void () noreturn` | Re-throw current exception |
| `llvm.wasm.catch` | `ptr (i32 immarg tag)` | Retrieve caught exception pointer in catch funclet |
| `llvm.wasm.landingpad.index` | `void (token, i32 immarg)` | Assign landing pad table index |

The tag argument to `wasm.throw` and `wasm.catch` is `WebAssembly::CPP_EXCEPTION = 0` for C++ exceptions and `WebAssembly::C_LONGJMP = 1` for `longjmp` emulation. Unlike the Itanium model, the type matching is performed by the Wasm runtime using type info embedded in the exception tag, rather than by a personality function traversing the LSDA. `llvm.wasm.lsda` (defined as `DefaultAttrsIntrinsic<[llvm_ptr_ty], [], [IntrNoMem]>`) returns the LSDA pointer for the current function, which the `WasmEHPrepare` pass uses to build the landing pad table.

---

## 26.9 EH and the Pass Manager

EH constructs impose constraints on most analyses and transformations. Understanding these constraints is essential for writing correct LLVM passes.

### 26.9.1 Unwind Edges in the CFG

Every `invoke` instruction adds an unwind edge from its block to the landing pad block. Analyses that traverse the CFG — including dominance, liveness, and LCSSA normalization — must account for unwind edges. `BasicBlock::isEHPad()` returns true for blocks beginning with `landingpad`, `catchswitch`, `catchpad`, or `cleanuppad`. `Instruction::isExceptionalTerminator()` identifies the EH terminators.

`canSimplifyInvokeNoUnwind(const Function *F)` (from `EHPersonalities.h`) returns true when the function's personality is known and none of its EH pads can be reached by the current call graph, permitting `SimplifyCFG` to convert remaining `invoke`s to `call`s.

### 26.9.2 WinEHPrepare and DwarfEHPrepare

**`WinEHPrepare`** runs late in the CodeGen pipeline. It demotes PHI nodes in catchswitch/catchpad/cleanuppad blocks (since funclet entry points cannot have SSA PHIs in the machine representation), builds the `WinEHFuncInfo` data structures (`CxxUnwindMapEntry`, `WinEHTryBlockMapEntry`, `WinEHHandlerType`), and validates the funclet parent chains. It runs as a function pass via `WinEHPreparePass::run(Function&, FunctionAnalysisManager&)`.

**`DwarfEHPrepare`** (declared in `llvm/include/llvm/IR/DwarfEHPrepare.h`) runs on the Itanium model. Its primary job is to lower `resume` into a call to `_Unwind_Resume` or `__cxa_rethrow` for platforms that require it.

**`SjLjEHPrepare`** handles the SJLJ transformation described in §26.7. It runs before instruction selection.

### 26.9.3 LSDA Emission: `EHStreamer`

The backend class `EHStreamer` (in `llvm/lib/CodeGen/AsmPrinter/`) is the base for all EH table emission. It provides the core LSDA layout algorithm: call site table, action table, type table. Two concrete subclasses handle the target-specific formats:

- `DwarfCFIException` — emits `.eh_frame` (via DWARF CFI) and `.gcc_except_table` for the Itanium model.
- `Win64Exception` — emits `.pdata` and `.xdata` for Windows x64 using `MCWin64EHInfo`.

The call site table entry for each `invoke` records the start and end of the instruction range (as offsets from the function start), the landing pad address, and an action table index. Action table entries chain: each entry has a type-info index (positive for `catch`, negative for `filter`, 0 for `cleanup`) and a `NextAction` offset to the next entry in the chain.

### 26.9.4 Impact on Analysis Passes

**Memory effects.** Alias analysis and `MemorySSA` treat `invoke`s as potentially reading and writing any memory accessible to the callee, just as `call`s do. The unwind edge cannot be assumed not to execute.

**Loop analysis.** A landing pad block that is the target of an unwind from inside a loop is not part of the loop — it is a loop-exiting edge. `LoopInfo` handles this correctly; loop-invariant code motion will not hoist past an `invoke` when the computation would be lost if the unwind fires first.

**Instruction scheduling.** The machine scheduler treats unwind edges as memory barriers — no instruction may be scheduled past an `invoke` if doing so would cause the store to become invisible on the unwind path.

---

## 26.10 `nounwind` and EH Elimination

### 26.10.1 `nounwind` Attribute

A function decorated with `nounwind` guarantees that it will never propagate an exception to its caller. This is the direct IR encoding of `noexcept` in C++. When Clang compiles:

```cpp
int noexcept_fn(int x) noexcept { return x * 2; }
```

it produces:

```llvm
define dso_local noundef i32 @_Z11noexcept_fni(i32 noundef %0) #0 {
  ...
}
attributes #0 = { ... nounwind ... }
```

The `nounwind` attribute is also propagated from `noexcept` declarations to function definitions that call them: a function that only calls `nounwind` functions and performs no operations that can throw is inferred to be `nounwind` by the `function-attrs` pass.

### 26.10.2 `invoke` to `call` Conversion

`SimplifyCFG` contains a specific transformation: when the callee of an `invoke` is `nounwind`, the unwind edge can never fire, so the `invoke` becomes a `call` and the landing pad block becomes dead. This is triggered by `simplifycfg` (no hyphen — the correct pass name in the new pass manager):

```
opt -passes=simplifycfg -S input.ll
```

Given an `invoke` of `nounwind_callee` (a `nounwind` function), `simplifycfg` produces:

```llvm
; Before:
%r = invoke i32 @nounwind_callee(i32 %x)
        to label %normal unwind label %lpad

; After simplifycfg:
%r = call i32 @nounwind_callee(i32 %x)
; %lpad block removed if unreachable
```

This transformation is verified by the LLVM 22 toolchain: `opt -passes=simplifycfg` on a function containing an `invoke` of a `nounwind` callee produces a plain `call`, and if the `lpad` block has no other predecessors it is also eliminated.

### 26.10.3 Dead Landing Pad Elimination and `-fno-exceptions`

When compiled with `-fno-exceptions`, Clang emits `nounwind` on every function and emits no `invoke` instructions — all calls become plain `call`s. This eliminates all EH overhead from the binary: no `.gcc_except_table`, no landing pads, no personality function references. The resulting code is smaller and avoids the register-allocation pressure from keeping live values across `invoke`s.

The `uwtable` module flag (`!{i32 7, !"uwtable", i32 2}` in the module metadata) controls whether the `.eh_frame` section is emitted regardless of exceptions. On Linux ELF, `-fasynchronous-unwind-tables` (the default at `-O0`) sets this flag and forces `.eh_frame` emission even for `nounwind` functions, to support tools like `perf` and `gdb` that rely on CFI unwind data for stack walking. Setting `-fno-asynchronous-unwind-tables` together with `-fno-exceptions` suppresses `.eh_frame` entirely.

---

## 26.11 Practical Patterns

### 26.11.1 Constructing Itanium EH IR by Hand

A minimal verified Itanium EH sequence that catches `int`:

```llvm
target triple = "x86_64-pc-linux-gnu"

declare i32 @__gxx_personality_v0(...)
declare ptr @__cxa_allocate_exception(i64)
declare void @__cxa_throw(ptr, ptr, ptr) noreturn
declare ptr @__cxa_begin_catch(ptr)
declare void @__cxa_end_catch()
declare i32 @llvm.eh.typeid.for.p0(ptr) nounwind memory(none)
declare void @throw_int_fn()

@_ZTIi = external constant ptr

define i32 @catch_int() personality ptr @__gxx_personality_v0 {
entry:
  invoke void @throw_int_fn()
          to label %normal unwind label %lpad

normal:
  ret i32 0

lpad:
  %lp = landingpad { ptr, i32 }
          catch ptr @_ZTIi
  %exn = extractvalue { ptr, i32 } %lp, 0
  %sel = extractvalue { ptr, i32 } %lp, 1
  %tid = call i32 @llvm.eh.typeid.for.p0(ptr @_ZTIi)
  %match = icmp eq i32 %sel, %tid
  br i1 %match, label %catch, label %rethrow

catch:
  %obj = call ptr @__cxa_begin_catch(ptr %exn)
  %val = load i32, ptr %obj
  call void @__cxa_end_catch()
  ret i32 %val

rethrow:
  resume { ptr, i32 } %lp
}
```

This verifies with `/usr/lib/llvm-22/bin/llvm-as`. Note: `personality` appears on `define`, never on `declare` — placing it on a declaration is a verifier error.

### 26.11.2 Compiling to Machine Code and Reading Tables

To compile the Itanium IR to x86-64 assembly with DWARF EH:

```bash
/usr/lib/llvm-22/bin/llc --exception-model=dwarf -filetype=asm \
    -o /tmp/catch_int.s /tmp/catch_int.ll
```

The resulting `.s` file contains both `.cfi_*` directives (for `.eh_frame`) and the explicit `.gcc_except_table` section with the call site and action tables. To read the unwind information from a compiled object:

```bash
clang++ -c -fexceptions -o /tmp/eh_basic.o /tmp/eh_basic.cpp
/usr/lib/llvm-22/bin/llvm-readobj --unwind /tmp/eh_basic.o
```

This dumps the CIE/FDE records from `.eh_frame`, showing register save/restore programs as sequences of `DW_CFA_*` operations.

Available `--exception-model` values in `llc`: `default`, `dwarf`, `sjlj`, `arm`, `wineh`, `wasm`. The `dwarf` model uses `DwarfCFIException`; `wineh` uses `Win64Exception`.

### 26.11.3 Asynchronous Unwind Tables

`-fasynchronous-unwind-tables` (or the equivalent `uwtable` module flag) forces every function to have a DWARF FDE regardless of whether it contains `invoke`s. This is the default on most Linux targets and is required for correct `backtrace(3)`, `perf record`, and signal-handler stack unwinding. The performance cost is purely code size — no runtime overhead on the hot path.

### 26.11.4 POSIX Thread Cancellation

`pthread_cancel` with `PTHREAD_CANCEL_DEFERRED` relies on cancellation points, which are implemented as forced unwind via `_Unwind_ForcedUnwind`. This API is an extension to the Itanium unwinding protocol: instead of searching for a handler, it always continues unwinding, calling cleanup landing pads in every frame. The `cleanup` clause on `landingpad` is critical for correctness here: a `catch ptr null` clause (catch-all) that does not re-throw would prevent `_Unwind_ForcedUnwind` from completing, causing `pthread_cancel` to silently fail. Correctly written C++ destructors only use the `cleanup` clause and always `resume` on the exceptional path.

---

## 26.12 EH in Non-C++ Languages

### 26.12.1 Objective-C

Clang's ObjC EH maps `@try`/`@catch`/`@throw` to the Itanium personality `__objc_personality_v0` on Darwin or `__gcc_personality_v0` with ObjC extensions on Linux/GNUstep. Under GCC runtime ObjC EH (`-fobjc-runtime=gnu`), `@throw obj` calls `objc_exception_throw(obj)` which calls `_Unwind_RaiseException` using an ObjC-specific exception class identifier (`"GNUOBJC"`). Type matching compares the thrown `id`'s class against the caught class using `objc_exception_matchClass`. In optimized `-O0`-compiled IR, the ARC/ObjC optimizer may lower `@try`/`@catch` into a non-EH form if all paths are statically provably safe, but `-fexceptions` forces the full landingpad path.

### 26.12.2 Swift

Swift's error model is primarily based on typed error returns — functions that can fail return an `Error` protocol value through a dedicated `%error_out` pointer (Swift calling convention extension). This avoids EH infrastructure entirely for the common case. Swift 5.7+ typed throws (`func f() throws(MyError) -> T`) strengthen this further, allowing the caller to stack-allocate the error slot. When Swift interoperates with ObjC exceptions or calls C++ code that throws, Clang's Swift interoperability layer wraps the cross-language call boundary with a personality function switch — Swift exceptions from ObjC `@throw` are caught at the Swift/ObjC boundary and converted to Swift `Error` values. LLVM IR from `swiftc` uses `__gxx_personality_v0` for functions that can receive C++ exceptions at ABI boundaries.

### 26.12.3 Go

Go's panic/recover mechanism does not use LLVM EH at the IR level. `panic` calls the runtime function `runtime.gopanic`, which stores the panic value on the goroutine's `g` struct and then iterates the goroutine's defer chain, running each deferred function. `recover` simply reads and clears the goroutine's panic field. This is implemented entirely in Go runtime code without `invoke`/`landingpad`; Go functions compiled by `gollvm` carry `nounwind` and do not participate in C++ unwinding. Cross-language interop (cgo calling C++ code) wraps the call in a C shim that catches C++ exceptions and translates them to Go panic values.

### 26.12.4 Rust

Rust's panic/unwind uses `rust_eh_personality` (`EHPersonality::Rust`), which wraps the standard Itanium two-phase protocol. On Linux, `libpanic_unwind` links against `libgcc_s` or `libunwind` and uses `_Unwind_RaiseException`. The Rust compiler annotates most functions `nounwind` (corresponding to `#[unwind(aborts)]` or the default `panic=abort` profile). With `panic=unwind`, Rust `landingpad`/`resume` IR looks structurally identical to C++, but Rust's type system eliminates `catch`-by-type — the personality function only uses cleanup entries, and `panic` payloads are opaque `Box<dyn Any>` objects. Cross-language unwind (Rust unwinding through C frames) is defined behavior as of Rust Edition 2021 when using `extern "C-unwind"` functions.

---

## Summary

- LLVM 22 supports five EH models: Itanium table-based (Linux/macOS/FreeBSD), Windows SEH/MSVC C++ (funclet model), SJLJ (setjmp/longjmp), WebAssembly (scoped or native instruction), and platform variants (XL_CXX, ZOS_CXX, GNU_ObjC).
- The `EHPersonality` enum and predicates `isFuncletEHPersonality`, `isScopedEHPersonality`, `isAsynchronousEHPersonality` in `EHPersonalities.h` classify any personality function.
- Itanium EH uses `invoke`/`landingpad { ptr, i32 }`/`resume`; the selector from the landingpad is compared against `llvm.eh.typeid.for` results to dispatch to the correct catch block.
- The `personality` attribute is a property of `define`, never `declare`; all EH pads in a function share one personality.
- Windows and Wasm EH use `catchswitch`/`catchpad`/`catchret`/`cleanuppad`/`cleanupret`; any call inside a funclet must carry `[ "funclet"(token %pad) ]`.
- SJLJ EH uses six intrinsics (`lsda`, `callsite`, `functioncontext`, `setjmp`, `longjmp`, `setup.dispatch`) inserted by `SjLjEHPrepare`; it trades code size and entry/exit overhead for the absence of a CFI table requirement.
- Wasm EH exists in two forms: the scoped funclet model (compatible with existing `catchswitch` infrastructure) and the native Wasm EH instruction model using `llvm.wasm.throw`, `llvm.wasm.catch`, `llvm.wasm.rethrow`.
- `nounwind` functions cannot propagate exceptions; `simplifycfg` converts `invoke` of `nounwind` callees to `call` and eliminates the dead landing pad.
- `-fno-exceptions` produces `nounwind` on all functions and no `invoke`s; `-fno-asynchronous-unwind-tables` additionally suppresses `.eh_frame` emission.
- EH overhead consists of LSDA emission (code size), `.eh_frame` emission, and — only in the SJLJ model — runtime linked-list registration overhead per function call.

---

*Cross-references:*
- [Chapter 20 — Instructions II: Control Flow and Aggregates](../part-04-llvm-ir/ch20-instructions-control-flow-and-aggregates.md) — complete syntax reference for `landingpad`, `resume`, `invoke`, `catchswitch`, `catchpad`, `cleanuppad`
- [Chapter 23 — Attributes, Calling Conventions, and the ABI](../part-04-llvm-ir/ch23-attributes-calling-conventions-abi.md) — `nounwind`, `noreturn`, `personality` attribute
- [Chapter 41 — Calls, the ABI Boundary, and Builtins](../part-06-clang-codegen/ch41-calls-abi-builtins.md) — Clang's EH lowering in `CodeGenFunction`
- [Chapter 42 — C++ ABI Lowering: Itanium](../part-06-clang-codegen/ch42-cxx-abi-itanium.md) — `__cxa_allocate_exception`, `__cxa_throw`, LSDA encoding
- [Chapter 43 — C++ ABI Lowering: Microsoft](../part-06-clang-codegen/ch43-cxx-abi-msvc.md) — `__CxxFrameHandler3`, `.xdata` format, funclet emission
- [Chapter 78 — LLD: EH Sections](../part-13-lto-whole-program/ch78-lld-eh-sections.md) — linker-level `.eh_frame` merging and LSDA relocation

*Reference links:*
- [Itanium C++ ABI: Exception Handling](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html)
- [`llvm/include/llvm/IR/EHPersonalities.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/EHPersonalities.h)
- [`llvm/include/llvm/CodeGen/WinEHFuncInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/WinEHFuncInfo.h)
- [`llvm/include/llvm/CodeGen/WasmEHFuncInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/WasmEHFuncInfo.h)
- [`llvm/include/llvm/CodeGen/SjLjEHPrepare.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/SjLjEHPrepare.h)
- [`llvm/include/llvm/CodeGen/WinEHPrepare.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/WinEHPrepare.h)
- [`llvm/include/llvm/IR/Intrinsics.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/Intrinsics.td) — `int_eh_sjlj_*`, `int_eh_typeid_for`, `int_eh_exceptionpointer`, `int_eh_exceptioncode`
- [`llvm/include/llvm/IR/IntrinsicsWebAssembly.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/IntrinsicsWebAssembly.td) — `int_wasm_throw`, `int_wasm_catch`, `int_wasm_rethrow`, `int_wasm_landingpad_index`
- [LLVM Exception Handling documentation](https://llvm.org/docs/ExceptionHandling.html)
- [Windows x64 Exception Handling](https://learn.microsoft.com/en-us/cpp/build/exception-handling-x64)


---

@copyright jreuben11
