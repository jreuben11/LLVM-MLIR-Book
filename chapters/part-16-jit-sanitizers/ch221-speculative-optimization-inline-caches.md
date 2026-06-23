# Chapter 221 — Speculative Optimization, Inline Caches, and Deoptimization

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

Conservative compilation produces correct code for all possible inputs but leaves performance on the table. Speculative compilation exploits the observation that real programs behave predictably: a call site that has always dispatched to the same method will probably continue to do so, a branch that has always been taken will probably continue to be taken, and a pointer that has never been null will probably remain non-null. Speculative optimization compiles these assumptions into the hot path — a single direct call instead of a virtual dispatch, a branch-free fast path instead of a guarded slow path — and provides a deoptimization escape hatch for the rare case when the assumption fails. This chapter covers the full mechanism: inline caches, guard IR patterns, LLVM's deoptimization intrinsics, stackmap-based frame reconstruction, on-stack replacement, and the tiering systems in V8 and HotSpot. It closes by connecting the theory to ORC's `ReOptimizeLayer`.

---

## Table of Contents

- [221.1 The Speculative Optimization Model](#2211-the-speculative-optimization-model)
  - [Conservative vs Speculative Compilation](#conservative-vs-speculative-compilation)
- [221.2 Inline Caches](#2212-inline-caches)
  - [IC State Machine](#ic-state-machine)
  - [Monomorphic IC in x86 Machine Code](#monomorphic-ic-in-x86-machine-code)
  - [Polymorphic IC: Type-Check Chain](#polymorphic-ic-type-check-chain)
  - [Feedback Vector Structure](#feedback-vector-structure)
- [221.3 Guard IR Patterns](#2213-guard-ir-patterns)
  - [@llvm.assume](#llvmassume)
  - [Deopt Bundles](#deopt-bundles)
  - [@llvm.experimental.guard](#llvmexperimentalguard)
  - [Guard Placement](#guard-placement)
- [221.4 @llvm.deoptimize and Deopt Bundles in Depth](#2214-llvmdeoptimize-and-deopt-bundles-in-depth)
  - [Deopt Bundle Format](#deopt-bundle-format)
  - [RewriteStatepointsForGC](#rewritestatepointsforgc)
- [221.5 Stackmap Frame Reconstruction](#2215-stackmap-frame-reconstruction)
  - [.llvm_stackmaps Section Format (v3)](#llvmstackmaps-section-format-v3)
  - [Location Kinds](#location-kinds)
  - [StackMapParser API](#stackmapparser-api)
  - [Deoptimizer Frame Reconstruction](#deoptimizer-frame-reconstruction)
- [221.6 On-Stack Replacement](#2216-on-stack-replacement)
  - [Forward OSR: Interpreter → JIT Mid-Loop](#forward-osr-interpreter-jit-mid-loop)
  - [Backward OSR: JIT → Interpreter (Deoptimization)](#backward-osr-jit-interpreter-deoptimization)
  - [LLVM's Current Support](#llvms-current-support)
- [221.7 V8: Ignition → Maglev → TurboFan Tiering](#2217-v8-ignition-maglev-turbofan-tiering)
  - [Tier 1: Ignition Bytecode Interpreter](#tier-1-ignition-bytecode-interpreter)
  - [Tier 2: Maglev (Mid-Tier JIT)](#tier-2-maglev-mid-tier-jit)
  - [Tier 3: TurboFan (Optimizing Compiler)](#tier-3-turbofan-optimizing-compiler)
  - [Tiering Policy](#tiering-policy)
  - [Deopt Counters and Bailout Prevention](#deopt-counters-and-bailout-prevention)
- [221.8 JVM HotSpot: C1 → C2 Tiering](#2218-jvm-hotspot-c1-c2-tiering)
  - [Tier 0: Template Interpreter](#tier-0-template-interpreter)
  - [Tier 1: C1 (Client Compiler)](#tier-1-c1-client-compiler)
  - [Tier 2: C2 (Server Compiler)](#tier-2-c2-server-compiler)
  - [Tiering Trigger](#tiering-trigger)
  - [Uncommon Traps (HotSpot's Deoptimization)](#uncommon-traps-hotspots-deoptimization)
  - [nmethod Invalidation](#nmethod-invalidation)
- [221.9 Connection to ORC ReOptimizeLayer](#2219-connection-to-orc-reoptimizelayer)
  - [Architecture](#architecture)
  - [AddProfilerFunc: Instrumenting Tier-1 Code](#addprofilerfunc-instrumenting-tier-1-code)
  - [reoptimizeIfCallFrequent: The Trigger](#reoptimizeifcallfrequent-the-trigger)
  - [ReOptimizeFunc: Tier-2 Recompilation](#reoptimizefunc-tier-2-recompilation)
  - [ResourceTracker Lifecycle](#resourcetracker-lifecycle)
  - [Extending with Speculative Guards](#extending-with-speculative-guards)
- [Chapter Summary](#chapter-summary)

---

## 221.1 The Speculative Optimization Model

Speculative compilation is a three-party contract between the *profiler*, the *compiler*, and the *runtime*:

- **Profiler**: observes runtime behavior and records type profiles, call frequencies, and branch outcomes in a *feedback vector* or *method data object*
- **Compiler**: reads the profile and generates *speculative* code that exploits observed behavior; emits *guards* that check each assumption before the fast path
- **Runtime**: detects when a guard fires (assumption violated), reconstructs the interpreter or unoptimized frame, and continues execution correctly

The compiler's obligation: every guard-protected fast path must have a correct but slower *deoptimization path* (deopt path) that handles the assumption-violating case. The fast path may be arbitrarily wrong as long as the guard fires before any observable state is committed.

```
        ┌──────────────────────────────┐
        │      speculative hot path    │
        │  [guard: type == Integer?]   │
        │        │           │         │
        │     (yes)         (no)       │
        │        │           │         │
        │  [fast path:   [deopt block: │
        │   direct call]  save state,  │
        │               call deopt     │
        │               handler]       │
        └──────────────────────────────┘
```

### Conservative vs Speculative Compilation

| Property | Conservative | Speculative |
|----------|-------------|-------------|
| Correctness | Always correct | Correct under guard |
| Polymorphic call cost | Virtual dispatch (indirect) | Direct call (speculative type) |
| Branch cost | Both sides compiled | Cold path moved to deopt block |
| Null check cost | Explicit check every time | Check at guard only |
| Recompilation required? | No | Yes, on assumption violation |
| Suitable for | AOT compilers | JIT compilers with profile feedback |

---

## 221.2 Inline Caches

An *inline cache* (IC) is a call-site-specific data structure that caches the last-seen dispatch target for a polymorphic call or property access. The cache is "inline" because it lives adjacent to (or inside) the call instruction stream.

### IC State Machine

```
Uninitialized
     │  first call: observe type T₁
     ▼
Monomorphic (T₁ → target₁)
     │  type T₂ ≠ T₁ observed
     ▼
Polymorphic (T₁→t₁, T₂→t₂, ... Tₙ→tₙ, n ≤ 8)
     │  too many types seen (>8)
     ▼
Megamorphic (fallback: runtime dispatch)
     │  type profile stabilizes (one dominant type)
     ▼  (optional: re-specialize to monomorphic)
Monomorphic
```

IC transitions are irreversible in the upward direction (never revert from polymorphic to uninitialized) but can specialize downward with explicit reset.

### Monomorphic IC in x86 Machine Code

The classic IC patch: the call instruction's displacement is patched to point at the observed target.

```
; Before first call (uninitialized):
call [stub_ptr]         ; stub_ptr → IC_miss_handler

; After first call with type Integer:
call [stub_ptr]         ; stub_ptr → integer_add_method
                        ; (IC_miss_handler patched stub_ptr)
```

Under W^X, the call instruction itself is immutable (on an `r-x` page). Only the *pointer cell* on a writable page is patched — the same model as ORC's `RedirectableSymbolManager`. The call instruction calls through the pointer; patching the pointer changes the dispatch target without touching executable pages.

### Polymorphic IC: Type-Check Chain

A polymorphic IC compiles a linear chain of type checks:

```llvm
; Generated LLVM IR for a 2-type polymorphic IC
define i64 @poly_ic(%Obj* %recv, i64 %arg) {
entry:
  %type = load i32, i32* getelementptr(%recv, 0, 0)  ; load type tag
  %is_int = icmp eq i32 %type, 1   ; Integer?
  br i1 %is_int, label %int_fast, label %check_float

int_fast:
  ; Direct call: Integer.add
  %result = call i64 @Integer_add(%Obj* %recv, i64 %arg)
  ret i64 %result

check_float:
  %is_float = icmp eq i32 %type, 2   ; Float?
  br i1 %is_float, label %float_fast, label %generic

float_fast:
  ; Direct call: Float.add
  %result2 = call i64 @Float_add(%Obj* %recv, i64 %arg)
  ret i64 %result2

generic:
  ; Megamorphic fallback
  %fn = call i64* @lookup_method(%Obj* %recv, i32 %method_id)
  %result3 = call i64 %fn(%Obj* %recv, i64 %arg)
  ret i64 %result3
}
```

The chain has O(n) type-check cost in the polymorphic case but O(1) in the monomorphic case (predicted correctly by the branch predictor).

### Feedback Vector Structure

V8 stores per-call-site profile data in a `FeedbackVector` object allocated alongside the function's compiled code. Each IC slot stores:

| Field | Type | Contents |
|-------|------|----------|
| `ic_state` | 2-bit enum | Uninitialized / Monomorphic / Polymorphic / Megamorphic |
| `map_or_maps` | tagged pointer | Monomorphic: single map; Polymorphic: WeakFixedArray of maps |
| `handler_or_smi` | tagged pointer | Compiled handler or call count |

HotSpot's equivalent is the `MethodData` object (`oops/methodData.hpp`), which stores `ReceiverTypeData` and `CallTypeData` entries per bytecode.

---

## 221.3 Guard IR Patterns

A *guard* is an assumption check inserted before the speculative fast path. Guards must be placed before any side-effecting operation that relies on the assumption.

### @llvm.assume

For lightweight hints that the optimizer can use but that have no runtime cost:

```llvm
; Tell the optimizer: %ptr is non-null
call void @llvm.assume(i1 %ptr_nonnull)
%val = load i32, i32* %ptr  ; optimizer elides null check
```

`@llvm.assume` is a no-op at runtime if the assumption is true. If the assumption is false, behavior is undefined — this is *not* a guard (no deopt path). Use only when the property is provably true by invariant, not merely observed.

### Deopt Bundles

A deopt bundle attaches interpreter frame state to a call instruction so that the deoptimizer can reconstruct the interpreter's view of the stack:

```llvm
; Call with a deopt bundle: if the callee deoptimizes,
; the runtime uses the bundle to reconstruct the caller's frame
%result = call i32 @speculative_callee(%T* %recv)
    [ "deopt"(i32 %pc_offset, i32 %frame_size,
              i32* %slot0, i64 %slot1, float %slot2) ]
```

The `"deopt"` operand bundle is consumed by `RewriteStatepointsForGC` during lowering. Each value in the bundle must be live at the call site; the compiler ensures no optimizations eliminate or merge these values before lowering.

### @llvm.experimental.guard

A guard that conditionally deoptimizes based on a condition:

```llvm
; If %condition is false, deoptimize (transfer control to deopt handler)
call void @llvm.experimental.guard(i1 %condition)
    [ "deopt"(i32 %pc, ...) ]
```

This is equivalent to:

```llvm
br i1 %condition, label %fast_path, label %deopt_block
deopt_block:
  call void @llvm.deoptimize() [ "deopt"(...) ]
  unreachable
fast_path:
  ...
```

`@llvm.experimental.guard` is a higher-level API that the lowering infrastructure converts to the explicit branch form. Source: `llvm/include/llvm/IR/Intrinsics.td`, lowered in `llvm/lib/Transforms/Scalar/GuardWidening.cpp`.

### Guard Placement

Guards must be placed *before* any operation that assumes the guarded property:

```llvm
; WRONG: guard after the load — undefined behavior if type != Integer
%val = load i32, i32* @int_field(%recv)   ; assumes Integer
call void @llvm.experimental.guard(i1 %is_integer) [ "deopt"(...) ]

; CORRECT: guard before the fast path
call void @llvm.experimental.guard(i1 %is_integer) [ "deopt"(...) ]
%val = load i32, i32* @int_field(%recv)   ; safe: guard fired first
```

Multiple guards can be *widened* (hoisted and merged) by `GuardWidening.cpp` — if two guards protect the same property along all paths to a common use, they can be merged into one. This reduces guard-check overhead.

---

## 221.4 @llvm.deoptimize and Deopt Bundles in Depth

`@llvm.deoptimize` is the intrinsic that triggers a deoptimization at a specific program point:

```llvm
declare void @llvm.deoptimize(...)
```

Declaration in `llvm/include/llvm/IR/Intrinsics.td`:
```
def int_experimental_deoptimize : Intrinsic<[], [llvm_vararg_ty],
    [IntrNoReturn, IntrCold]>;
```

### Deopt Bundle Format

The deopt bundle attached to `@llvm.deoptimize` (or to `@llvm.experimental.guard`) encodes the interpreter state needed to reconstruct the execution frame:

```llvm
call void @llvm.deoptimize()
    [ "deopt"(
        i32 42,        ; virtual PC offset (bytecode index)
        i32 16,        ; number of live slots
        i32 %vreg0,    ; local variable 0
        i64 %vreg1,    ; local variable 1
        float %vreg2,  ; local variable 2
        ...
      ) ]
unreachable
```

The `i32 42` PC offset identifies which bytecode instruction to restart from. The remaining values are the live interpreter slots at that point — the exact set that the interpreter would have on its stack if it had been executing this bytecode from the start.

### RewriteStatepointsForGC

The same mechanism serves both GC safepoints and deoptimization. `RewriteStatepointsForGC` (`llvm/lib/Transforms/Scalar/RewriteStatepointsForGC.cpp`) transforms:

```llvm
; Before RewriteStatepointsForGC:
%result = call i32 @gc_allocate(i32 %size)
    [ "deopt"(i32 %pc, i32* %ptr0) ]
%val = load i32, i32* %ptr0   ; may be stale after GC

; After RewriteStatepointsForGC:
%sp = call token @llvm.experimental.gc.statepoint.p0f_i32i32f(
    i64 0, i32 0, i32 (i32)* @gc_allocate, i32 1, i32 0,
    i32 %size, i32 0, i32 0, i32* %ptr0)
%result = call i32 @llvm.experimental.gc.result.i32(token %sp)
%ptr0_relocated = call i32* @llvm.experimental.gc.relocate.p0i32(
    token %sp, i32 7, i32 7)   ; %ptr0 after possible GC move
%val = load i32, i32* %ptr0_relocated  ; uses relocated pointer
```

The statepoint sequence:
1. `gc.statepoint` call: GC-safe call site; carries all live GC-managed pointers
2. `gc.result`: extracts the call return value from the statepoint token
3. `gc.relocate`: extracts each live pointer *after* potential GC movement

For deoptimization purposes, the statepoint's attached deopt values encode the interpreter state, and the deopt handler reads them from the stackmap record.

---

## 221.5 Stackmap Frame Reconstruction

When a deoptimization guard fires, the deoptimizer needs to reconstruct the interpreter frame. The `.llvm_stackmaps` section provides a machine-readable map from each safepoint or guard to the location of every live value.

### .llvm_stackmaps Section Format (v3)

```
Header:
  uint8_t  Version = 3
  uint8_t  Reserved[3]
  uint32_t NumFunctions
  uint32_t NumConstants
  uint32_t NumRecords

Function records [NumFunctions]:
  uint64_t FunctionAddress
  uint64_t StackSize
  uint64_t RecordCount

Constants [NumConstants]:
  uint64_t LargeConstant

Stack map records [NumRecords]:
  uint64_t PatchPointID
  uint32_t InstructionOffset  ; from function start
  uint16_t Reserved
  uint16_t NumLocations

  Locations [NumLocations]:
    uint8_t  Kind           ; 1=Register, 2=Direct, 3=Indirect, 4=Constant, 5=ConstIndex
    uint8_t  Size
    uint16_t DwarfRegNum
    int32_t  Offset

  NumLiveOuts
  LiveOuts [NumLiveOuts]:
    uint16_t DwarfRegNum
    uint8_t  Reserved
    uint8_t  Size
```

### Location Kinds

| Kind | Interpretation |
|------|---------------|
| Register (1) | Value is in `DwarfRegNum` |
| Direct (2) | Value is at `[DwarfRegNum + Offset]` (pointer to value) |
| Indirect (3) | Value is at `*[DwarfRegNum + Offset]` (pointer to pointer to value) |
| Constant (4) | Value is the literal `Offset` (small constants) |
| ConstIndex (5) | Value is `Constants[Offset]` (large constants) |

### StackMapParser API

```cpp
#include "llvm/Object/StackMapParser.h"

// Find the .llvm_stackmaps section in the running binary
const uint8_t* stackmapData = getStackMapsSection();
llvm::StackMapParser<llvm::support::native> Parser(stackmapData);

for (auto FR : Parser.functions()) {
    llvm::outs() << "Function at: " << FR.getFunctionAddress() << "\n";
    for (auto R : Parser.records()) {
        llvm::outs() << "  Record ID: " << R.getID()
                     << " at offset " << R.getInstructionOffset() << "\n";
        for (auto Loc : R.locations()) {
            // Loc.getKind(), Loc.getDwarfRegNum(), Loc.getOffset()
        }
    }
}
```

### Deoptimizer Frame Reconstruction

The deopt handler algorithm:

```
1. Signal received (guard fires → call to deopt handler)
2. Walk the native call stack to find the JIT'd frame (using .llvm_stackmaps or DWARF)
3. Locate the stackmap record for the current instruction offset
4. For each live value in the record:
   a. Use Location.Kind to find the value (register, stack slot, or constant)
   b. Read the value from the JIT'd frame
5. Construct the interpreter frame: allocate interpreter stack, populate slots
6. Resume the interpreter at the PC recorded in the deopt bundle
```

The key distinction from DWARF unwinding: DWARF's `.debug_frame` (or `.eh_frame`) describes how to unwind *return addresses* — how to find the caller's frame from the callee's frame. Stackmaps describe how to read *live SSA values* — the actual data the interpreter needs. Both are needed for full deoptimization: DWARF for the call chain, stackmaps for the value map.

---

## 221.6 On-Stack Replacement

*On-stack replacement* (OSR) is a technique for transferring execution between two compiled representations of the same program *while the program is running* — without returning to a common call site. The two directions:

### Forward OSR: Interpreter → JIT Mid-Loop

Scenario: a loop is being interpreted; the loop becomes hot; the JIT compiles the function; the interpreter needs to transfer control to the JIT'd version at the current loop back-edge, not at the function entry.

The JIT'd code must accept the interpreter's live state *at the loop header* as entry values. A dedicated *OSR entry stub* is emitted at the loop header:

```llvm
; OSR entry point for loop iteration i=42
define i64 @hot_loop_osr_entry(i64 %i, i64* %arr, i64 %n) {
osr_entry:
  ; Validate that the entry state is coherent
  ; (no speculative assumptions yet — we just arrived from interpreter)
  br label %loop_header

loop_header:
  %cur_i = phi i64 [%i, %osr_entry], [%next_i, %loop_body]
  ...
}
```

The interpreter, when it detects the hot back-edge counter overflow, calls `hot_loop_osr_entry` with the current values of `%i`, `%arr`, and `%n`. The JIT'd loop runs from there.

The `LoopVersioning` pass (`llvm/lib/Transforms/Transforms/LoopVersioning.cpp`) creates versioned copies of loops with and without speculative assumptions, providing landing pads for OSR entry.

### Backward OSR: JIT → Interpreter (Deoptimization)

The guard fires inside the JIT'd loop:

```
JIT'd loop body:
  [guard: type == Integer?]  → fires on unexpected type
  → call deopt handler
  → reconstruct interpreter frame at current loop header
  → resume interpreter at loop header
```

This is a guard-triggered deopt from within a loop body, not from a function call boundary. The deopt handler must reconstruct the interpreter's state mid-loop, not just at function entry. The deopt bundle on the guard carries the *loop iteration* state: the induction variable, all live scalar values, and any stack-allocated objects.

### LLVM's Current Support

LLVM fully supports backward OSR (deopt side exits from JIT code) via `@llvm.deoptimize` and the stackmap mechanism. Forward OSR (live transfer *into* compiled code mid-loop) requires custom runtime glue:

1. The JIT must emit an OSR entry point at every loop header
2. The interpreter must track per-back-edge invocation counts
3. The interpreter must know how to call the OSR entry point with the correct live values
4. The OSR entry must be ABI-compatible with the interpreter's representation of live values

No in-tree LLVM component provides this glue end-to-end; it is the responsibility of the embedding language runtime. Julia, V8, and HotSpot each implement it differently.

---

## 221.7 V8: Ignition → Maglev → TurboFan Tiering

V8 uses a three-tier compilation pipeline governed by profiling feedback.

### Tier 1: Ignition Bytecode Interpreter

Ignition compiles JavaScript to a register-based bytecode (not LLVM IR). Each bytecode instruction updates the per-call-site `FeedbackVector` slot:

- `CallIC` slots: record observed receiver maps
- `LoadIC`/`StoreIC` slots: record property access shapes
- `BinaryOpIC` slots: record operand types (Smi, HeapNumber, String, Mixed)

The feedback vector is the *profile* that drives tier-2 and tier-3 compilation.

### Tier 2: Maglev (Mid-Tier JIT)

Maglev reads the feedback vector and generates optimized code with speculative type checks (guards). It operates at lower optimization cost than TurboFan:

- **No escape analysis** (all allocations remain allocations)
- **No global value numbering** (local CSE only)
- **Type-checked direct calls** from IC feedback, with deopt on type mismatch
- **Compilation latency**: ~1 ms for a typical function (vs ~10 ms for TurboFan)

Maglev emits `@llvm.deoptimize`-equivalent deopt points (implemented in V8's own IR, not LLVM IR). On deopt, V8 reconstructs the Ignition frame from the Maglev frame using its own stackmap equivalent.

### Tier 3: TurboFan (Optimizing Compiler)

TurboFan applies full speculative optimization:

- **Escape analysis**: heap allocations replaced with stack allocations or scalar replacements
- **Load/store elimination**: redundant loads from known-shape objects eliminated
- **Speculation**: type feedback → speculative type-narrowing → guard + fast path
- **Sea-of-nodes IR**: TurboFan's IR is a sea-of-nodes graph (cf. Click-Cooper 1995), where control and data edges are unified — a concept that influenced LLVM's `SelectionDAG` and `MachineIR`

### Tiering Policy

The V8 tiering engine promotes functions based on invocation count and bytecode size:

```
Invocations   Bytecode size   Action
< 1,000       any             Ignition only
1,000–10,000  < 500 bytes     Maglev
10,000+       < 1000 bytes    TurboFan
any           > 1000 bytes    Maglev (large functions cap at mid-tier)
```

### Deopt Counters and Bailout Prevention

TurboFan tracks how many times each speculation point has deoptimized. After a configurable threshold (default: 10 deopts from the same site), TurboFan marks the speculation as *bailout-prevented* and recompiles without that speculation. This prevents the pathological case of repeatedly compiling, deoptimizing, and recompiling for a chronically polymorphic call site.

---

## 221.8 JVM HotSpot: C1 → C2 Tiering

HotSpot uses a two-tier JIT (C1 and C2) with an interpreter as tier 0.

### Tier 0: Template Interpreter

The template interpreter generates per-bytecode native code snippets at VM startup. It instruments every bytecode with profiling counters in the `MethodData` object:

- `ReceiverTypeData`: records receiver types at `invokevirtual`/`invokeinterface`
- `CallTypeData`: records argument/return types
- `BranchData`: records branch taken/not-taken counts
- `MultiBranchData`: records `tableswitch`/`lookupswitch` case frequencies

### Tier 1: C1 (Client Compiler)

C1 generates optimized native code quickly (< 1 ms). It performs:
- Null check elimination
- Range check elimination
- Inlining (up to ~35 bytecodes)
- Linear-scan register allocation

C1 instruments the output code with *method invocation counters* and *back-edge counters* that drive the tier-2 trigger.

### Tier 2: C2 (Server Compiler)

C2 performs full speculative optimization from the `MethodData` profile:

- **Receiver type speculation**: virtual dispatch replaced by type-guarded direct call
- **Null check elimination**: proven-non-null pointers have checks removed
- **Loop optimization**: vectorization, unrolling, induction variable elimination
- **Escape analysis**: short-lived allocations replaced with scalar components
- **Devirtualization**: abstract method calls resolved to concrete targets

### Tiering Trigger

```
Method invocations:    1,500  → C1 compile (Tier 3: with profiling)
Back-edge count:      10,000  → OSR compile (Tier 3 OSR)
Invocations × calls: 100,000  → C2 compile (Tier 4: full optimization)
```

### Uncommon Traps (HotSpot's Deoptimization)

When a C2 speculation fails at runtime, the JIT-compiled code transfers to an *uncommon trap*:

1. C2 emits a call to `Deoptimization::uncommon_trap()` at each speculation point
2. The uncommon trap handler reads the oop map (HotSpot's stackmap equivalent) to reconstruct the interpreter frame
3. The interpreter resumes at the failing bytecode
4. The `MethodData` object records the deoptimization
5. After `PerMethodRecompilationCutoff` deopts, C2 recompiles the method with the assumption disabled

The HotSpot oop map (`oops/oop.hpp`, `compiler/oopMap.hpp`) serves the same role as `.llvm_stackmaps`: it maps each safepoint to the location of every GC-managed reference (for GC) and every interpreter value (for deoptimization).

### nmethod Invalidation

When a loaded class changes (new subclass added, class redefinition via JVMTI), all compiled methods that speculated on that class's type hierarchy must be invalidated:

```
HotSpot nmethod invalidation:
1. ClassHierarchyAnalysis dependency recorded at compile time
2. Class load → traverse dependency list → mark dependent nmethods as not_entrant
3. Not-entrant nmethods: existing activations complete; no new entries
4. Zombie nmethods: no more activations → GC can reclaim
```

LLVM's equivalent: if an ORC-compiled module relied on a type assumption that is later violated, the ResourceTracker for that module is removed and a new module compiled without the assumption.

---

## 221.9 Connection to ORC ReOptimizeLayer

ORC's `ReOptimizeLayer` (`llvm/include/llvm/ExecutionEngine/Orc/ReOptimizeLayer.h`) implements the speculative tiering concept directly in LLVM.

### Architecture

```
LLJIT
  └── ReOptimizeLayer   ← intercepts addIRModule
        └── IRTransformLayer
              └── ObjectTransformLayer
                    └── ObjectLinkingLayer (JITLink)
```

When a module is added through `ReOptimizeLayer`, the layer:

1. Calls `AddProfilerFunc` to instrument the module (inject call-count instrumentation)
2. Adds the instrumented (tier-1) module
3. Registers a `reoptimizeIfCallFrequent` callback with the `ExecutionSession`

### AddProfilerFunc: Instrumenting Tier-1 Code

```cpp
// User-supplied: instrument the module before tier-1 emission
auto AddProfiler = [](llvm::orc::ReOptimizeLayer& ROL,
                      llvm::orc::ReOptimizeLayer::ReOptMaterializationUnitID MUID,
                      unsigned CurVersion,
                      llvm::Module& M) -> llvm::Error {
    auto& Ctx = M.getContext();
    auto* CounterType = llvm::Type::getInt64Ty(Ctx);
    for (auto& F : M) {
        if (F.isDeclaration()) continue;
        // Insert call-count increment at function entry
        llvm::IRBuilder<> Builder(&*F.getEntryBlock().getFirstInsertionPt());
        auto* Counter = new llvm::GlobalVariable(
            M, CounterType, false, llvm::GlobalValue::InternalLinkage,
            llvm::ConstantInt::get(CounterType, 0),
            F.getName() + "_call_count");
        auto* Load = Builder.CreateLoad(CounterType, Counter);
        auto* Inc = Builder.CreateAdd(Load, llvm::ConstantInt::get(CounterType, 1));
        Builder.CreateStore(Inc, Counter);
    }
    return llvm::Error::success();
};
```

### reoptimizeIfCallFrequent: The Trigger

```cpp
// Built-in callback: triggers reoptimization when call count exceeds threshold
llvm::orc::ReOptimizeLayer::CallCountThreshold = 10;

auto Trigger = llvm::orc::ReOptimizeLayer::reoptimizeIfCallFrequent;
// When call count > CallCountThreshold:
//   ROL.reoptimize(MUID) is called
//   → ReOptimizeFunc is invoked with the original ThreadSafeModule
//   → ReOptimizeFunc recompiles at -O3
//   → redirect() atomically patches callers
```

### ReOptimizeFunc: Tier-2 Recompilation

```cpp
auto ReOptimize = [](llvm::orc::ReOptimizeLayer& ROL,
                     llvm::orc::ReOptimizeLayer::ReOptMaterializationUnitID MUID,
                     unsigned CurVersion,
                     llvm::orc::ThreadSafeModule& TSM) -> llvm::Error {
    return TSM.withModuleDo([&](llvm::Module& M) -> llvm::Error {
        llvm::PassBuilder PB;
        llvm::ModulePassManager MPM;
        llvm::ModuleAnalysisManager MAM;
        PB.registerModuleAnalyses(MAM);
        MPM = PB.buildPerModuleDefaultPipeline(llvm::OptimizationLevel::O3);
        MPM.run(M, MAM);
        return llvm::Error::success();
    });
};
```

### ResourceTracker Lifecycle

The tier-1 `ResourceTracker` (`OldRT`) must remain alive until all threads that may be executing tier-1 code have returned. ORC does not automatically quiesce in-flight calls; the simplest approach is a brief delay:

```cpp
auto OldRT = JD.createResourceTracker();
// ... add tier-1 module under OldRT ...

// After ReOptimize completes and redirect() patches callers:
// Wait for in-flight tier-1 calls to drain
std::this_thread::sleep_for(std::chrono::milliseconds(50));
llvm::cantFail(OldRT->remove());
```

More precise alternatives: epoch counting (ORC `TaskDispatcher` extension), hazard pointers, or RCU (read-copy-update) — the same techniques used in Linux kernel module unloading.

### Extending with Speculative Guards

The full speculative+deopt loop under ORC:

1. **Tier-1**: `AddProfilerFunc` instruments with call counts *and* emits `@llvm.experimental.guard` at each speculation point (e.g., type assumption on a polymorphic call)
2. **Guard fires** → deopt handler increments a deopt counter per guard site
3. **Reoptimization trigger**: either call count threshold or deopt counter reset
4. **ReOptimizeFunc** reads the type profile accumulated since tier-1, recompiles with type-narrowed fast paths, emits fresh guards for the dominant type
5. **redirect()** atomically patches callers
6. **Old tier-1 ResourceTracker** released after drain

This is equivalent to V8's Ignition→TurboFan tiering, implemented over LLVM ORC primitives.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **ORC `ReOptimizeLayer` stabilization**: the layer is currently experimental in LLVM 22; community effort on [discourse.llvm.org](https://discourse.llvm.org) is focused on graduating it to a supported API, including making `reoptimizeIfCallFrequent` production-ready and adding precise quiescing of in-flight tier-1 calls via `TaskDispatcher` epoch counting instead of the current `sleep_for` workaround.
- **`@llvm.experimental.guard` promotion**: `GuardWidening.cpp` and the `@llvm.experimental.guard` intrinsic carry the "experimental" prefix; an LLVM RFC is in progress to formalize the guard IR design, clarify interactions with `MemorySSA`, and rename to `@llvm.guard` once semantics are frozen.
- **Stackmap v4 format**: the current v3 stackmap format lacks per-record flags and per-location type metadata needed for precise deoptimization of SIMD values and vector types; a v4 proposal (tracking issue in LLVM bugzilla) adds a `Flags` field and extended location encoding for `<N x T>` types.
- **Maglev LLVM backend experiments**: members of the V8 and LLVM communities have explored compiling Maglev's mid-tier IR through an LLVM backend rather than V8's own code generator; active prototype patches surfaced on the LLVM dev list in early 2026.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **ORC speculative type-profile integration**: extending `ReOptimizeLayer` to thread per-call-site type profiles (not just invocation counts) through the `AddProfilerFunc` / `ReOptimizeFunc` interface, enabling LLVM-hosted JITs to implement monomorphic-IC → polymorphic-IC re-specialization without bespoke profiling infrastructure.
- **Deoptimization support for scalable vectors (SVE/SME)**: current `.llvm_stackmaps` location encoding is undefined for AArch64 SVE `<vscale x N x T>` types; deoptimizing through a JIT'd SVE loop body requires a new "indirect scalable" location kind and matching runtime support in `StackMapParser`; this is a prerequisite for speculative JIT use on AArch64 HPC targets.
- **Hardware-accelerated ICs via BTI/PAC**: AArch64 Branch Target Identification and Pointer Authentication can be used to protect IC patch targets; ongoing standards work in the AArch64 ABI committee is defining a JIT-safe calling convention that allows patching IC pointer cells under PAC without disabling authentication — directly impacting IC performance under security-hardened kernels.
- **LLVM `SpeculativeJIT` library**: the Julia, Swift, and Zig communities have each implemented speculative-optimization layers atop ORC with partially-overlapping infrastructure; a proposal is forming to consolidate common guard-insertion, deopt-bundle construction, and stackmap-parsing utilities into an in-tree `llvm/lib/ExecutionEngine/SpeculativeJIT` library, analogous to how `LLJIT` consolidated the previous scatter of JIT entry points.

### 5-Year Horizon (Long-Term, by ~2031)

- **Ahead-of-time profile-guided deoptimization (AOTPGD)**: merging offline PGO profile databases with JIT speculative profiles to pre-seed inline-cache states and guard thresholds before the first execution, eliminating the cold-start deoptimization spike; research prototypes exist for HotSpot (Project Leyden) and are expected to influence ORC's `ReOptimizeLayer` trigger design.
- **Formal verification of deoptimization correctness**: extending the Alive2 / Vellvm verification frameworks to reason about deopt-bundle semantics — proving that the interpreter state reconstructed by a stackmap record is observationally equivalent to the interpreter state that would have existed had the JIT'd code never run; this is an open problem referenced in Lopes et al.'s Alive2 papers and a prerequisite for verified JIT correctness.
- **Continuous adaptive recompilation via ML feedback**: replacing threshold-based tiering policies (call-count, back-edge counter) with learned policies that predict deoptimization risk from runtime features (branch history, IC state distribution, allocation rate), drawing on ML-guided compiler work in TensorFlow XLA's cost models and LLVM's `MLInlineAdvisor` infrastructure; expected to appear first in server-side JIT deployments where profile data is abundant.

---

## Chapter Summary

- Speculative compilation exploits observed runtime stability — direct calls, branch-free fast paths, null-check elimination — and requires a correct deoptimization path for every assumption
- Inline caches progress through uninitialized → monomorphic → polymorphic → megamorphic states; under W^X, only the *pointer cell* (on a writable page) is patched, never the call instruction itself
- `@llvm.assume` provides hint-only hints with no runtime overhead; `@llvm.experimental.guard` with a deopt bundle provides a checked guard with a deopt path; `@llvm.deoptimize` triggers the deoptimization
- The deopt bundle format encodes the interpreter state — PC offset and live slot values — needed for frame reconstruction; `RewriteStatepointsForGC` (`llvm/lib/Transforms/Scalar/RewriteStatepointsForGC.cpp`) lowers bundles to the statepoint sequence
- `.llvm_stackmaps` (v3 format) maps each safepoint to the location of every live value; `StackMapParser` provides the API for reading it at runtime; this is orthogonal to DWARF (which maps return-address chains, not live SSA values)
- Forward OSR (interpreter → JIT mid-loop) requires custom runtime glue (OSR entry stub + interpreter back-edge counter); backward OSR (JIT → interpreter) is fully supported via `@llvm.deoptimize` and stackmaps
- V8's Ignition → Maglev → TurboFan pipeline and HotSpot's Interpreter → C1 → C2 pipeline are production implementations of the same speculative tiering concept; deopt counters and nmethod invalidation prevent recompilation thrash
- ORC's `ReOptimizeLayer` implements the concept directly: `AddProfilerFunc` instruments tier-1, `reoptimizeIfCallFrequent` triggers at threshold, `ReOptimizeFunc` recompiles at `-O3`, `redirect()` atomically patches callers

---

@copyright jreuben11
