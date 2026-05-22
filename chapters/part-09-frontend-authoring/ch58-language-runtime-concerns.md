# Chapter 58 — Language Runtime Concerns

*Part IX — Frontend Authoring (Building Your Own)*

A language frontend does not end at IR emission. The runtime environment — memory management, thread-local storage, atomic operations, signal safety, and machine-level stack metadata — must be designed and wired into the IR explicitly. LLVM provides hooks for all of these: GC roots and statepoints for garbage collection, TLS lowering models for thread-local variables, the C11/C++11 memory model for atomics, stack maps for runtime inspection, and signal-safety constraints that affect code placement. This chapter covers each concern from the frontend author's perspective.

## Table of Contents

- [58.1 Garbage Collection Strategies](#581-garbage-collection-strategies)
  - [58.1.1 Registering a GC Strategy](#5811-registering-a-gc-strategy)
  - [58.1.2 The gc.root Model](#5812-the-gcroot-model)
  - [58.1.3 The Statepoint Model (Recommended)](#5813-the-statepoint-model-recommended)
  - [58.1.4 Stack Maps](#5814-stack-maps)
- [58.2 Thread-Local Storage Lowering](#582-thread-local-storage-lowering)
  - [58.2.1 TLS Models](#5821-tls-models)
  - [58.2.2 Emitting Thread-Local Globals](#5822-emitting-thread-local-globals)
  - [58.2.3 TLS Access Intrinsics](#5823-tls-access-intrinsics)
- [58.3 Atomics and the C/C++ Memory Model](#583-atomics-and-the-cc-memory-model)
  - [58.3.1 Atomic Orderings](#5831-atomic-orderings)
  - [58.3.2 Atomic Load and Store](#5832-atomic-load-and-store)
  - [58.3.3 Atomic RMW Operations](#5833-atomic-rmw-operations)
  - [58.3.4 Compare-and-Exchange](#5834-compare-and-exchange)
  - [58.3.5 Fences](#5835-fences)
  - [58.3.6 Volatile Accesses](#5836-volatile-accesses)
- [58.4 Stack Maps and Patchpoints](#584-stack-maps-and-patchpoints)
  - [58.4.1 The Complete Code-Mutation Workflow](#5841-the-complete-code-mutation-workflow)
  - [58.4.2 Guard IR Patterns](#5842-guard-ir-patterns)
- [58.5 Signal Safety](#585-signal-safety)
  - [58.5.1 Async-Signal-Safe Code in IR](#5851-async-signal-safe-code-in-ir)
  - [58.5.2 `setjmp`/`longjmp`](#5852-setjmplongjmp)
- [58.6 Putting It Together: Runtime Design Checklist](#586-putting-it-together-runtime-design-checklist)
- [Chapter Summary](#chapter-summary)

---

## 58.1 Garbage Collection Strategies

LLVM supports two main GC strategies: the **gc.root** model (legacy) and the **statepoint** model (recommended). The strategy is selected per-function via the `gc` attribute and the registered `GCStrategy` subclass.

### 58.1.1 Registering a GC Strategy

```cpp
// In a separate GC plugin or linked into the compiler
class MyGCStrategy : public llvm::GCStrategy {
public:
  MyGCStrategy() {
    UseStatepoints = true;          // use statepoint model
    NeededSafePoints = 0;           // no automatic insertion
  }
};
LLVM_INITIALIZE_GC_STRATEGY(MyGCStrategy, "mygc")
```

Apply the strategy to each GC-managed function:

```cpp
fn->setGC("mygc");
```

### 58.1.2 The gc.root Model

The legacy `gc.root` model registers each GC pointer as a root in the function frame:

```llvm
; declare i8* @llvm.gcroot(i8** %ptrloc, i8* %metadata)

define i64 @f() gc "shadow-stack" {
entry:
  %p.root = alloca ptr         ; GC root slot
  call void @llvm.gcroot(ptr %p.root, ptr null)
  ; … allocate and use %p through %p.root …
}
```

The `shadow-stack` GC strategy (a builtin) emits a linked list of root frames in the shadow stack. This model is simple but imposes per-alloca overhead; it is suitable for small-scale implementations.

### 58.1.3 The Statepoint Model (Recommended)

The statepoint model places GC safe points at calls: every call that can trigger GC is replaced by `gc.statepoint`, which explicitly lists all live GC pointers. After the call, `gc.relocate` loads the (possibly-moved) pointer values:

```llvm
; Before statepoint lowering:
%ptr = call ptr @allocate()

; After statepoint lowering:
%statepoint_token = call token (i64, i32, ptr, i32, i32, ...)
  @llvm.experimental.gc.statepoint.p0(
    i64 0,                ; statepoint ID
    i32 0,                ; num patch bytes
    ptr @allocate,        ; callee
    i32 0, i32 0,         ; num call args, flags
    ... %live_ptr1 ...)   ; GC-managed live values

; Relocate after GC may have moved objects
%ptr.relocated = call ptr @llvm.experimental.gc.relocate(
    token %statepoint_token,
    i32 0, i32 0)         ; base index, derived index
```

The frontend emits `gc.statepoint` at every call that may trigger GC, listing all live GC-managed pointers. The `RewriteStatepointsForGC` pass (in `llvm/lib/Transforms/Scalar/RewriteStatepointsForGC.cpp`) can insert these automatically given an initial set of GC roots, reducing frontend burden.

**Frontend workflow with statepoints:**

1. Mark all GC-allocated pointers with a distinct type (e.g., pointers in a named address space).
2. At each call that may trigger GC, emit a `gc.statepoint` listing all live GC pointers.
3. After the call, emit `gc.relocate` for each GC pointer that is still live.
4. Run `RewriteStatepointsForGC` as a post-IR-emission pass to handle tracking.

```cpp
// Emit a statepoint-wrapped call
llvm::Value *IREmitter::emitGCCall(llvm::Function *callee,
                                    ArrayRef<llvm::Value*> args,
                                    ArrayRef<llvm::Value*> liveGCPtrs) {
  // Build the statepoint intrinsic call
  auto *statepointDecl =
    llvm::Intrinsic::getOrInsertDeclaration(mod.get(),
      llvm::Intrinsic::experimental_gc_statepoint,
      {callee->getType()});

  llvm::SmallVector<llvm::Value*> spArgs = {
    builder.getInt64(0),    // ID
    builder.getInt32(0),    // patch bytes
    callee,
    builder.getInt32(args.size()),
    builder.getInt32(0),    // flags
  };
  spArgs.append(args);
  // Append live GC pointers
  spArgs.push_back(builder.getInt32(0)); // num transition args
  spArgs.push_back(builder.getInt32(0)); // num deopt args
  spArgs.append(liveGCPtrs);

  auto *token = builder.CreateCall(statepointDecl, spArgs, "statepoint");

  // Emit gc.result to extract the return value
  auto *resultDecl =
    llvm::Intrinsic::getOrInsertDeclaration(mod.get(),
      llvm::Intrinsic::experimental_gc_result,
      {callee->getReturnType()});
  return builder.CreateCall(resultDecl, {token}, "gc.result");
}
```

### 58.1.4 Stack Maps

Stack maps are binary sections emitted by LLVM that describe, for each safe point, where live GC roots reside (register or stack offset). They are consumed by the GC runtime to identify and update roots during collection.

Stack maps are generated automatically when `gc.statepoint` intrinsics are present. The resulting `.llvm_stackmaps` section uses the StackMap format documented in `llvm/docs/StackMaps.rst`. The GC runtime reads this section at startup or on-demand to build a table of safe points.

## 58.2 Thread-Local Storage Lowering

### 58.2.1 TLS Models

Thread-local variables have four addressing models on ELF platforms, each with different performance/flexibility trade-offs:

| Model | Flag | Use case |
|-------|------|---------|
| General Dynamic | `-ftls-model=global-dynamic` | Position-independent code in shared libraries; most flexible, slowest |
| Local Dynamic | `-ftls-model=local-dynamic` | Module-local TLS in shared libraries; one call per module |
| Initial Exec | `-ftls-model=initial-exec` | Known-at-load TLS in executables; fast, limited to exec |
| Local Exec | `-ftls-model=local-exec` | Executable-internal TLS; fastest, no dynamic linking |

### 58.2.2 Emitting Thread-Local Globals

```cpp
// Thread-local global variable
auto *gv = new llvm::GlobalVariable(*mod,
             builder.getInt64Ty(), /*isConstant=*/false,
             llvm::GlobalValue::ExternalLinkage,
             builder.getInt64(0),
             "tls_counter");

// Set TLS model
gv->setThreadLocalMode(llvm::GlobalValue::GeneralDynamicTLSModel);
// or InitialExecTLSModel, LocalExecTLSModel, LocalDynamicTLSModel
```

### 58.2.3 TLS Access Intrinsics

LLVM lowers TLS accesses to platform-specific sequences. On x86-64 Linux (General Dynamic):

```llvm
; __tls_get_addr protocol
%tls_addr = call ptr @__tls_get_addr(ptr @tls_counter@tlsgd)
%val      = load i64, ptr %tls_addr
```

The frontend need not emit these sequences manually: declaring the global as `thread_local` and using it normally causes the backend to generate the correct TLS access protocol for the target. The frontend only needs to set `setThreadLocalMode` correctly.

For languages that implement their own fibers or coroutines with custom stacks, the TLS model must be `LocalExec` or `InitialExec` to avoid per-access overhead. Custom green-thread runtimes often use a single TLS variable pointing to the current fiber's state block:

```cpp
auto *fiberPtr = new llvm::GlobalVariable(*mod,
                   builder.getPtrTy(), false,
                   llvm::GlobalValue::ExternalLinkage,
                   llvm::ConstantPointerNull::get(builder.getPtrTy()),
                   "__current_fiber");
fiberPtr->setThreadLocalMode(llvm::GlobalValue::InitialExecTLSModel);
```

## 58.3 Atomics and the C/C++ Memory Model

### 58.3.1 Atomic Orderings

LLVM's memory model maps directly to C11/C++11 ordering:

| C++ ordering | LLVM `AtomicOrdering` |
|-------------|----------------------|
| `relaxed` | `AtomicOrdering::Monotonic` |
| `consume` | `AtomicOrdering::Acquire` (promoted) |
| `acquire` | `AtomicOrdering::Acquire` |
| `release` | `AtomicOrdering::Release` |
| `acq_rel` | `AtomicOrdering::AcquireRelease` |
| `seq_cst` | `AtomicOrdering::SequentiallyConsistent` |

### 58.3.2 Atomic Load and Store

```cpp
using AO = llvm::AtomicOrdering;

// Atomic load: int x = atomic_load(&shared, memory_order_acquire)
llvm::LoadInst *ld = builder.CreateLoad(builder.getInt32Ty(), sharedPtr, "ld");
ld->setAtomic(AO::Acquire);
ld->setAlignment(llvm::Align(4));

// Atomic store: atomic_store(&shared, val, memory_order_release)
llvm::StoreInst *st = builder.CreateStore(val, sharedPtr);
st->setAtomic(AO::Release);
st->setAlignment(llvm::Align(4));
```

### 58.3.3 Atomic RMW Operations

Read-modify-write operations use `CreateAtomicRMW`:

```cpp
// fetch_add: old = atomic_fetch_add(&counter, 1, acq_rel)
llvm::Value *old = builder.CreateAtomicRMW(
    llvm::AtomicRMWInst::Add,
    counterPtr,
    builder.getInt32(1),
    llvm::MaybeAlign(4),
    AO::AcquireRelease);
```

Available RMW operations: `Add`, `Sub`, `And`, `Nand`, `Or`, `Xor`, `Max`, `Min`, `UMax`, `UMin`, `FAdd`, `FSub`, `FMax`, `FMin`, `Xchg`.

### 58.3.4 Compare-and-Exchange

```cpp
// bool success = atomic_compare_exchange_strong(
//     &ptr, &expected, desired, success_ord, fail_ord)
auto [oldVal, success] = builder.CreateAtomicCmpXchg(
    ptr, expected, desired,
    llvm::MaybeAlign(8),
    AO::AcquireRelease,   // success ordering
    AO::Acquire);         // failure ordering
// oldVal: the value that was in *ptr before the CAS
// success: i1 — true if exchange succeeded
```

`CreateAtomicCmpXchg` returns a `{ T, i1 }` struct (or an `llvm::Value` of struct type); use `CreateExtractValue` to access the components:

```cpp
llvm::Value *result = builder.CreateAtomicCmpXchg(ptr, expected, desired,
    llvm::MaybeAlign(8), AO::AcquireRelease, AO::Acquire);
llvm::Value *oldVal  = builder.CreateExtractValue(result, {0}, "old");
llvm::Value *success = builder.CreateExtractValue(result, {1}, "success");
```

### 58.3.5 Fences

Standalone fences impose ordering without a memory access:

```cpp
// seq_cst fence (e.g., after a store to ensure global visibility)
builder.CreateFence(AO::SequentiallyConsistent,
                     llvm::SyncScope::System);

// Acquire fence (after a relaxed load)
builder.CreateFence(AO::Acquire, llvm::SyncScope::System);
```

`SyncScope::System` (the default) applies to all threads on the system. `SyncScope::SingleThread` applies only to the current thread (useful for signal-safety fences).

### 58.3.6 Volatile Accesses

For MMIO (memory-mapped I/O) or other non-atomic volatile accesses:

```cpp
llvm::LoadInst *ld = builder.CreateLoad(ty, mmioPtr, "mmio");
ld->setVolatile(true);
// volatile prevents reordering relative to other volatile accesses
// but does NOT provide multi-thread synchronization
```

Volatile is about observability (the compiler must not elide or reorder), not atomicity.

## 58.4 Stack Maps and Patchpoints

Stack maps enable a runtime to inspect or modify a running program's stack. They are used for:
- GC root enumeration (see §58.1.3)
- JIT deoptimization — inspecting the LLVM-compiled frame and reinterpreting it at a different tier
- Profiling — sampling the live values at a function call

The `@llvm.experimental.stackmap` and `@llvm.experimental.patchpoint` intrinsics generate stack map records:

```llvm
; Record a stack map at this point with live values %a, %b
call void (i64, i32, ...) @llvm.experimental.stackmap(
    i64 42,      ; stack map ID (used by runtime to look up entry)
    i32 0,       ; number of shadow bytes
    i64 %a,      ; live value 1
    i64 %b)      ; live value 2
```

```llvm
; Patchpoint: reserve N bytes for runtime patching (e.g., JIT inline caches)
%result = call i64 (i64, i32, ptr, i32, ...) @llvm.experimental.patchpoint.i64(
    i64 43,          ; patchpoint ID
    i32 13,          ; num patch bytes (NOP sled)
    ptr @target_fn,  ; initial callee
    i32 1,           ; num call args
    i64 %arg)        ; call argument
```

The backend fills the `.llvm_stackmaps` section. At runtime, the runtime locates the section, parses the StackMap header, and for each record finds the PC offset and live-value locations (register number or stack offset).

Use case — JIT deoptimization:

```cpp
// Emit a patchpoint at a JIT-compiled call site
// The runtime can later overwrite the NOP sled with a conditional deopt branch
void IREmitter::emitDeoptPoint(uint64_t id, llvm::Value *liveVal) {
  auto *sp = llvm::Intrinsic::getOrInsertDeclaration(mod.get(),
    llvm::Intrinsic::experimental_stackmap);
  builder.CreateCall(sp, {builder.getInt64(id), builder.getInt32(0), liveVal});
}
```

### 58.4.1 The Complete Code-Mutation Workflow

The patchpoint intrinsic reserves a NOP sled in compiled code. At runtime the JIT — or any code-patching runtime — locates the sled, overwrites it with a real instruction, and resumes execution. The complete cycle has five steps.

**Step 1 — Emit.** At IR construction time, emit a patchpoint for every call site that may require future patching:

```cpp
// Emit a patchpoint wrapping @fastpath with a 13-byte NOP sled.
// 13 bytes accommodates a REX.W CALL r/m64 (3 bytes) or a JZ rel32 (6 bytes)
// plus padding NOPs on x86_64.
llvm::Function *ppDecl = llvm::Intrinsic::getOrInsertDeclaration(
    mod.get(),
    llvm::Intrinsic::experimental_patchpoint_i64,
    {builder.getInt64Ty()});

llvm::Value *ppArgs[] = {
    builder.getInt64(patchpointID),  // unique ID → keyed in .llvm_stackmaps
    builder.getInt32(13),            // NOP-sled size in bytes
    fastpathFn,                      // initial callee
    builder.getInt32(1),             // number of call arguments
    arg,                             // the actual argument
    livePtr,                         // live value recorded for deopt reconstruction
};
llvm::Value *result = builder.CreateCall(ppDecl, ppArgs, "pp_result");
```

After compilation the backend emits a `call` to `fastpathFn` at the patchpoint address followed by `(13 - CALL_SIZE)` bytes of `nop` padding. One record keyed by `patchpointID` is appended to `.llvm_stackmaps`.

**Step 2 — Locate the sled.** At runtime, parse `.llvm_stackmaps` using `StackMapParser` to find the patchpoint's code address:

```cpp
#include "llvm/Object/StackMapParser.h"

// smData: raw bytes of the .llvm_stackmaps section obtained from the JIT
// (e.g., via an ObjectLinkingLayer plugin that captures section contents).
llvm::ArrayRef<uint8_t> smData = getStackMapSection();
llvm::StackMapParser<llvm::support::little> SMP(smData);
// SMP.getVersion() == 3  (the only format LLVM 22 emits)

uint64_t sledPC = 0;
for (auto &Rec : SMP.records()) {
    if (Rec.getID() == patchpointID) {
        // getInstructionOffset() is the PC of the patchpoint relative to
        // the start of the function; add the function's load address.
        sledPC = funcLoadAddr + Rec.getInstructionOffset();
        break;
    }
}
```

Each record also carries a list of live-value locations. Each location is either a register number (type `Register`) or a frame-pointer-relative offset (type `Direct` / `Indirect`) — the raw materials for frame reconstruction.

**Step 3 — Establish a writable mapping.** ELF `.text` carries program-header flags `PF_R | PF_X` (no `PF_W`). Modern OS kernels and security policies enforce W^X: a page cannot be simultaneously writable and executable. The standard solution is **dual-mapping** — map the same physical memory at two virtual addresses: one `r-x` (for execution) and one `rw-` (for patching). Writes through the `rw-` alias appear immediately in the `r-x` view because both virtual addresses reference the same physical page frames.

On Linux using `memfd_create`:

```cpp
int memFd = memfd_create("jit_patch", MFD_CLOEXEC);
ftruncate(memFd, pageSize);

// Executable view — what the CPU fetches instructions from:
void *rxView = mmap(nullptr, pageSize, PROT_READ | PROT_EXEC,
                    MAP_SHARED, memFd, 0);

// Writable alias at a distinct virtual address:
void *rwView = mmap(nullptr, pageSize, PROT_READ | PROT_WRITE,
                    MAP_SHARED, memFd, 0);
close(memFd);  // mappings remain after fd is closed
```

On Apple Silicon, use `MAP_JIT` and bracket writes with `pthread_jit_write_protect_np`:

```cpp
void *mem = mmap(nullptr, pageSize,
                 PROT_READ | PROT_WRITE | PROT_EXEC,
                 MAP_PRIVATE | MAP_ANONYMOUS | MAP_JIT, -1, 0);
pthread_jit_write_protect_np(false);
// ... write machine code or patch pointer cells ...
pthread_jit_write_protect_np(true);
```

`JITLink`'s `InProcessMemoryManager` handles these platform differences automatically when allocating JIT memory.

**Step 4 — Overwrite the sled.** Compute the offset of the sled within the mapped page and write the new instruction sequence through `rwView`:

```cpp
uintptr_t offsetInPage = sledPC % pageSize;
uint8_t *rwSled = static_cast<uint8_t *>(rwView) + offsetInPage;

// On x86_64: write a JZ rel32 (6 bytes) — branch to deopt handler if guard fails.
rwSled[0] = 0x0F;
rwSled[1] = 0x84;                                 // JZ near opcode
int32_t rel = static_cast<int32_t>(
    deoptHandlerAddr - (sledPC + 6));             // PC-relative displacement
memcpy(rwSled + 2, &rel, 4);
for (int i = 6; i < 13; ++i) rwSled[i] = 0x90;  // NOP padding

// Sequential memory fence — ensure write is visible before any CPU fetches the page.
std::atomic_thread_fence(std::memory_order_seq_cst);
```

**Step 5 — Resume.** Execution continues normally through `rxView`. On the next call, the formerly-NOP sled contains the conditional branch. If the guard condition holds, fall-through continues to the fast path with zero added cost. If the guard fails, the deopt block fires, the deopt handler reads the live-value locations from the `.llvm_stackmaps` record for `patchpointID`, reconstructs the interpreter's variable table, and resumes execution in the interpreter.

### 58.4.2 Guard IR Patterns

Guards are conditional branches that enforce speculative assumptions. When a guard fails the program deoptimizes — it falls back to a slower, more general execution tier. The following patterns cover the three most common guard forms.

**Type guard** — assume an object's type tag matches a concrete class:

```llvm
; Assume %obj is an instance of Foo (runtime tag == 0x42).
%tag = load i32, ptr %obj, align 4
%ok  = icmp eq i32 %tag, 42
br i1 %ok, label %fast_path, label %deopt_block

fast_path:
  ; Optimized code that exploits Foo-specific field layout.
  %field = getelementptr inbounds %Foo, ptr %obj, i32 0, i32 1
  %val   = load i64, ptr %field, align 8
  ret i64 %val

deopt_block:
  ; Record all values the interpreter needs to reconstruct state.
  call void (i64, i32, ...) @llvm.experimental.stackmap(
      i64 1001,   ; deopt-site ID — matched against .llvm_stackmaps at runtime
      i32 0,
      ptr %obj,   ; live: the object
      i64 %a,     ; live: other values
      i64 %b)
  call void @__deopt_handler(i64 1001)
  unreachable    ; tells LLVM this path never returns; enables fast-path cleanup
```

The `unreachable` terminator is mandatory: without it LLVM must keep register save/restore code on the fast path in case the deopt block returns.

**Null guard** — check a pointer before dereferencing:

```llvm
%null = icmp eq ptr %ptr, null
br i1 %null, label %deopt_block, label %continue
continue:
  %val = load i64, ptr %ptr, align 8
```

**Bounds guard** — verify an array index before a GEP:

```llvm
%len   = load i64, ptr %arr_len_ptr, align 8
%inbnd = icmp ult i64 %idx, %len
br i1 %inbnd, label %continue, label %deopt_block
```

**Connecting guards to patchpoints.** A guard that is never observed to fail can be patched away entirely: emit the guard condition check as a patchpoint; once the JIT determines the guard always passes, overwrite the patchpoint's sled with an unconditional fall-through (NOP or `jmp` past the branch). When an assumption finally fails, the deopt runtime restores the conditional branch before re-executing through the interpreter. This is the pattern used in V8 Turbofan (type-feedback-driven guard elision), JVM HotSpot C2 (uncommon-trap nodes), and YJIT for Ruby (side-exit guards).

## 58.5 Signal Safety

Signal handlers execute asynchronously relative to the main thread. Code in a signal handler must be async-signal-safe: it may not call `malloc`, `free`, or any function that acquires a lock. This constrains what IR the frontend may emit inside signal handler bodies.

### 58.5.1 Async-Signal-Safe Code in IR

A function called from a signal handler must:
- Not call any function that is not async-signal-safe (no `printf`, `malloc`, etc.).
- Use `SingleThread` sync scope for fences and atomic accesses, since multi-thread synchronization within a signal handler requires `sys_futex` or similar, not C atomics.
- Not use `alloca` in a way that races with the interrupted thread's stack pointer.

LLVM has no static enforcement of async-signal-safety. The frontend must track which functions are signal-handler bodies and restrict emission accordingly, or analyze the call graph statically.

### 58.5.2 `setjmp`/`longjmp`

`setjmp`/`longjmp` interact with stack unwinding. LLVM represents `setjmp` as an intrinsic:

```llvm
; %buf = alloca [5 x i64]  ; jmp_buf
; %r = call i32 @llvm.eh.sjlj.setjmp(ptr %buf)
```

The `sjlj` (setjmp/longjmp) exception handling model uses these intrinsics. Most modern targets prefer DWARF EH, but embedded targets without stack unwinding support use `sjlj`. The frontend selects the model via `-fexceptions -fsjlj-exceptions`.

## 58.6 Putting It Together: Runtime Design Checklist

When designing a new language runtime that targets LLVM:

| Concern | Decision | LLVM mechanism |
|---------|----------|---------------|
| Memory management | GC or manual? | `gc.statepoint` / no annotation |
| GC roots | How to enumerate live ptrs? | StackMap section |
| Thread locals | Which TLS model? | `setThreadLocalMode` |
| Atomic model | C11 or weaker? | `AtomicOrdering` enum |
| Exception handling | DWARF, SJLJ, or none? | `setjmp`/`invoke`/`landingpad` |
| Signal safety | Which fns are handler-safe? | Frontend invariant |
| Stack inspection | Profiling or deopt? | `stackmap`/`patchpoint` |
| Coroutines | Stackful or stackless? | `@llvm.coro.*` / manual lowering |

The choices are interrelated: a GC-managed language that also uses coroutines must ensure that GC safe points are correctly emitted at coroutine suspend points, and that the GC can scan coroutine stacks. The statepoint model supports this by treating coroutine resume calls as ordinary `gc.statepoint` calls with live GC roots in the coroutine frame.

---

## Chapter Summary

- LLVM provides two GC models: the legacy `gc.root` model and the recommended `gc.statepoint` model; prefer statepoints for new implementations.
- `gc.statepoint` wraps every GC-safe call, listing live GC pointers; `gc.relocate` loads potentially-moved pointer values after the call.
- Stack maps (`.llvm_stackmaps` ELF section) enumerate live GC roots at each safe point; consumed by the GC runtime at collection time.
- TLS globals are emitted with `setThreadLocalMode`; General Dynamic is safest for shared libraries, Local Exec fastest for executables.
- Atomic loads/stores use `setAtomic` on `LoadInst`/`StoreInst`; RMW uses `CreateAtomicRMW`; CAS uses `CreateAtomicCmpXchg`.
- `SyncScope::System` applies to all threads; `SyncScope::SingleThread` applies to the current thread (appropriate for signal handlers).
- Stackmap and patchpoint intrinsics (`@llvm.experimental.stackmap`, `@llvm.experimental.patchpoint`) support JIT deoptimization and profiling; the `.llvm_stackmaps` section (version-3 format, parsed by `StackMapParser`) records patchpoint IDs, instruction offsets, and per-live-value register/stack locations.
- The complete code-mutation workflow: emit patchpoint → compile → locate sled via `StackMapParser::records()` → establish a writable alias via `memfd_create`+dual-`mmap` (Linux) or `MAP_JIT`+`pthread_jit_write_protect_np` (Apple Silicon) → overwrite the sled through the writable alias → fence → resume; W^X is never violated because the executable view is never made writable.
- Guard IR patterns: type guards, null guards, and bounds guards emit conditional branches to `deopt_block` basic blocks; `unreachable` after the deopt call enables fast-path code-size reduction; guards can be patched away via patchpoints once observed to be always-passing (the V8 / HotSpot pattern).
- Signal handler bodies must restrict themselves to async-signal-safe operations; LLVM provides no static enforcement, so the frontend must maintain this invariant.


---

@copyright jreuben11
