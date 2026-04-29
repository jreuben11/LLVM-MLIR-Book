# Chapter 108 — The ORC JIT

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

LLVM's third-generation JIT framework, ORC (On-Request Compilation), represents the state of the art in compiler-integrated execution engines. Where its predecessors required whole-module compilation and single-threaded execution, ORC was designed from first principles for concurrency, laziness, and composability. It powers Julia's numeric computing runtime, LLDB's expression evaluator, Swift's REPL, and the Kaleidoscope tutorial JIT. Understanding ORC means understanding how a modern language runtime can compile and execute code at the speed demanded by interactive and server-side workloads, while remaining safe across hundreds of concurrent compilation threads. This chapter walks from the conceptual history of LLVM JIT designs through the full machinery of ORC v2: sessions, dylibs, layers, lazy compilation, multi-process execution, and the debugging integration that makes JIT-compiled code first-class in GDB and perf.

---

## 108.1 JIT Design Generations

### 108.1.1 Generation 1: ExecutionEngine and MCJIT

The original LLVM JIT, `ExecutionEngine`, predates the modern pass manager and was never designed for concurrent use. It exposed a simple interface: hand it an `llvm::Module`, call `getPointerToFunction`, and receive a callable function pointer. The implementation used `RuntimeDyld` to relocate ELF/MachO object files in-process, and the entire pipeline was protected by a single global lock.

`MCJIT` (Machine Code JIT) replaced the even older "JIT" infrastructure in LLVM 3.x. MCJIT used the MC (Machine Code) layer to produce proper object files rather than ad-hoc code emission. This gave accurate debug information and better ABI compatibility. However, MCJIT still committed to compiling entire modules at once, held a global lock during compilation, and had no mechanism for incremental or lazy compilation. Once a module was added, its symbols could not be removed without destroying the entire engine.

The resource management problem proved fatal for large JIT users. A JIT that cannot remove code cannot serve a language with hot-code reload, garbage collection of compiled methods, or long-running processes that accumulate hundreds of gigabytes of compilation artifacts.

### 108.1.2 Generation 2: ORC v1

ORC v1 introduced the **layer model**: a JIT was composed of stacked layers, each transforming IR or object files before passing them down. An `IRCompileLayer` compiled modules; an `ObjectLinkingLayer` linked them. Lazy compilation became possible through the `CompileOnDemandLayer`. This was a significant conceptual advance.

However, ORC v1 was not designed for fine-grained concurrency. Sharing state across layers required careful external synchronization, and the API exposed several thread-safety pitfalls. The multi-process execution model that became critical for sandboxed JITs was not supported. Symbol lookup required holding a global session lock, serializing all concurrent compilations.

### 108.1.3 Generation 3: ORC v2 (Current)

ORC v2, introduced in LLVM 9 and stabilized through LLVM 13, redesigned the internal state model around **sessions**, **JIT dynamic libraries**, and **materialization units**. Every mutable piece of state lives behind the `ExecutionSession`, which manages a work queue and ensures that symbol definition and lookup operations are serialized appropriately — while compilation itself can proceed on any thread in a user-supplied thread pool.

Key design goals achieved by ORC v2:

- **Concurrency**: Multiple materializers can execute simultaneously on a thread pool. The session coordinates symbol lookup results without a global compile lock — only symbol-state transitions are serialized, not compilation.
- **Laziness**: Functions can be represented as stubs whose bodies are compiled only when first called. The compiler path is triggered transparently on the first invocation via platform-specific trampolines.
- **Composability**: Layers are independent objects that can be mixed, subclassed, and replaced. A user can write a custom `IRTransformLayer` that applies interprocedural optimizations on the JIT path.
- **Multi-process capability**: The `ExecutorProcessControl` abstraction separates the compiling process from the executing process, enabling sandboxed JIT, cross-architecture compilation, and debugger expression evaluation in a separate address space.
- **Resource tracking**: `ResourceTracker` allows removal of compiled code — critical for hot-reload and GC-based language runtimes.

The relevant source tree:

```
llvm/lib/ExecutionEngine/Orc/          # Core ORC v2 implementation
llvm/include/llvm/ExecutionEngine/Orc/ # Public API headers
compiler-rt/lib/orc/                   # ORC runtime (trampolines, platform support)
llvm/tools/llvm-jitlink/               # JITLink testing harness
llvm/examples/Kaleidoscope/Chapter9/   # Tutorial JIT reference implementation
llvm/unittests/ExecutionEngine/Orc/    # ORC unit tests
```

---

## 108.2 Core Abstractions

### 108.2.1 ExecutionSession

[`ExecutionSession`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/Orc/Core.h#L1502) is the root object of every ORC JIT instance. It owns:

- The `SymbolStringPool` — an interned string table for all symbol names in this JIT session
- The set of `JITDylib`s (JIT-managed dynamic libraries) comprising the symbol namespace
- The `ExecutorProcessControl` (EPC) — the interface to the execution environment
- A `dispatch` function that schedules materializer work onto a user-supplied thread pool
- An `ErrorReporter` function called when asynchronous errors occur during materialization

The session enforces that symbol state transitions (defining, materializing, resolving, emitting) happen without races. Internally it uses a mutex-protected work queue that processes `SessionState` callbacks in a FIFO order. External code interacts via callbacks or blocking `lookup` calls.

```cpp
#include "llvm/ExecutionEngine/Orc/Core.h"
#include "llvm/ExecutionEngine/Orc/ExecutorProcessControl.h"
#include "llvm/ExecutionEngine/Orc/IndirectionUtils.h"

using namespace llvm;
using namespace llvm::orc;

// Create an ExecutionSession with in-process control:
auto SSP = std::make_shared<SymbolStringPool>();
auto EPC = SelfExecutorProcessControl::Create(SSP);
if (!EPC) { handleAllErrors(EPC.takeError(), [](const ErrorInfoBase &E) {
  errs() << "EPC creation failed: " << E.message() << "\n"; }); }

auto ES = std::make_unique<ExecutionSession>(std::move(*EPC));

// Set up a thread pool dispatcher:
auto TP = std::make_unique<DynamicThreadPoolTaskDispatcher>();
ES->setDispatchTask([&TP](std::unique_ptr<Task> T) {
  TP->dispatch(std::move(T));
});

// Set a custom error reporter:
ES->setErrorReporter([](Error Err) {
  logAllUnhandledErrors(std::move(Err), errs(), "JIT Error: ");
});
```

### 108.2.2 JITDylib

A `JITDylib` is ORC's abstraction of a dynamic library: a namespace of symbols with a defined lookup order relative to other `JITDylib`s. Every `ExecutionSession` starts with at least one — typically called "main". Dylibs are created with `createBareJITDylib` (no platform support) or via a `Platform` object that initializes the dylib with platform-specific metadata.

JITDylibs form a **search order**: when a symbol is looked up, the session traverses the dylib list and returns the first match. This models the `DT_NEEDED` dependencies of ELF or the load order of `dyld`. A JIT for Python might have one JITDylib per module, with the interpreter's builtins accessible via a shared "builtins" dylib that all module dylibs depend on.

```cpp
// Create and link two JITDylibs:
JITDylib &MainJD = ES->createBareJITDylib("main");
JITDylib &BuiltinsJD = ES->createBareJITDylib("builtins");

// main can see builtins' exported symbols:
MainJD.addToLinkOrder(BuiltinsJD,
                      JITDylibLookupFlags::MatchExportedSymbolsOnly);

// Symbols defined in main are visible:
cantFail(MainJD.define(
  absoluteSymbols({{ES->intern("myFunc"),
                    {ExecutorAddr::fromPtr(&myFunc),
                     JITSymbolFlags::Exported}}})));
```

### 108.2.3 ResourceTracker

[`ResourceTracker`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/Orc/Core.h#L920) is ORC's answer to the memory-leak problem of earlier JITs. Each `MaterializationUnit` is associated with a tracker, and `RT->remove()` destroys all memory, stubs, and registrations associated with the units under that tracker. The removal is asynchronous — it enqueues work to deallocate the associated executor memory and unregister debug info.

```cpp
// Create a resource tracker for a "version" of some compiled code:
auto RT = MainJD.createResourceTracker();

// Associate this module with RT:
IRLayer->add(RT, ThreadSafeModule(std::move(M), std::move(Ctx)));

// Later, to hot-reload by replacing with a new version:
auto NewRT = MainJD.createResourceTracker();
IRLayer->add(NewRT, ThreadSafeModule(std::move(NewM), std::move(NewCtx)));

// Remove the old version (frees memory, unregisters debug info):
cantFail(RT->remove());
// OldRT is now invalid; all its symbols are removed from MainJD
```

### 108.2.4 MaterializationUnit and MaterializationResponsibility

A [`MaterializationUnit`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/Orc/Core.h#L790) represents deferred work: "here is a set of symbols I can provide; call `materialize()` when you need them." It declares the symbols it will define (via `getInterface()` returning a `SymbolFlagsMap`) and either a `materialize()` function or a `discard()` hint for symbols that turn out not to be needed.

The classic implementation is `IRMaterializationUnit`, which holds a `ThreadSafeModule` and compiles it on demand. Custom materializers can fetch source from a network, decompress a cached object, or synthesize code on the fly.

[`MaterializationResponsibility`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/Orc/Core.h#L618) (MR) is the session's "ticket": when ORC decides to materialize a unit (because a lookup needs one of its symbols), it calls `materialize(std::unique_ptr<MaterializationResponsibility> MR)`. The materializer must eventually call either:
- `MR.notifyResolved(SymbolMap)` followed by `MR.notifyEmitted()` — success path
- `MR.failMaterialization()` — error path; all dependents fail with an error

```cpp
class MyMaterializationUnit : public MaterializationUnit {
public:
  MyMaterializationUnit(ExecutionSession &ES, StringRef FuncName)
      : MaterializationUnit(buildInterface(ES, FuncName)),
        FuncName(FuncName) {}

  StringRef getName() const override { return "MyMU"; }

  void materialize(std::unique_ptr<MaterializationResponsibility> MR) override {
    // Compile on this (or another) thread:
    auto [M, Ctx] = compileFunctionFromSource(FuncName);
    
    // Hand the module to the IR compile layer, passing along the MR:
    if (auto Err = IRLayer.add(MR->getTargetJITDylib(),
                               ThreadSafeModule(std::move(M), std::move(Ctx)),
                               MR->getResourceTracker())) {
      MR->failMaterialization();
    }
  }

private:
  std::string FuncName;
  IRCompileLayer &IRLayer;
  
  static SymbolFlagsMap buildInterface(ExecutionSession &ES, StringRef Name) {
    SymbolFlagsMap Syms;
    Syms[ES.intern(Name)] = JITSymbolFlags::Exported;
    return Syms;
  }
};
```

### 108.2.5 SymbolStringPool

All symbol names in ORC are interned via [`SymbolStringPool`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/Orc/SymbolStringPool.h). The `SymbolStringPtr` is a reference-counted smart pointer to an interned string. Interning allows O(1) symbol comparison and avoids repeated string hashing during lookup — a critical optimization when an ORC session may manage millions of symbols (e.g., Julia's JIT with thousands of compiled methods each having multiple symbols).

The pool is shared across the entire `ExecutionSession` and is safe to access from multiple threads. `ES->intern("printf")` returns the canonical `SymbolStringPtr` for "printf".

---

## 108.3 The Layer Model

Layers are the compositional building blocks of an ORC JIT. Each layer implements a specific transformation and delegates to the layer below it. The typical production stack is:

```
  IRTransformLayer    (optimization passes: O2, inlining, loop vectorization)
        |
  IRCompileLayer      (LLVM Module → object file bytes via TargetMachine)
        |
  ObjectLinkingLayer  (object file → executable memory via JITLink)
        |
  Executor memory     (live symbols callable by JIT'd and native code)
```

### 108.3.1 ObjectLinkingLayer

[`ObjectLinkingLayer`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/ObjectLinkingLayer.cpp) is the bottom layer — it receives object files (as `MemoryBuffer`s) and uses JITLink (Chapter 109) to link and allocate them in executable memory. It handles:
- Creating a `LinkGraph` from the object bytes via format-specific JITLink parsers
- Running the JITLink pipeline (GOT/PLT synthesis, relocation fixup, memory allocation)
- Registering debug information, `.eh_frame` unwinding data, and platform-specific metadata
- Notifying ORC that emitted symbols are now live via `MaterializationResponsibility::notifyEmitted`

The older `RTDyldObjectLinkingLayer` used RuntimeDyld instead of JITLink. For new code, `ObjectLinkingLayer` is always the correct choice.

### 108.3.2 IRCompileLayer

[`IRCompileLayer`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/IRCompileLayer.cpp) receives a `ThreadSafeModule`, invokes the `IRCompiler` to produce an object file, then passes the object file down to the object layer. It implements `emit()` which:

1. Locks the `ThreadSafeModule`'s context mutex
2. Calls the compiler's `operator()` to produce an in-memory object file
3. Passes the object file's `MemoryBuffer` to the `ObjectLinkingLayer::emit` method

`ConcurrentIRCompiler` wraps a `JITTargetMachineBuilder` and creates a fresh `TargetMachine` per compilation. This is critical because `TargetMachine` is not thread-safe and must not be shared across concurrent compilations. The `JITTargetMachineBuilder` itself is thread-safe — it caches only immutable configuration.

### 108.3.3 IRTransformLayer

[`IRTransformLayer`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/IRTransformLayer.cpp) sits above the compile layer and applies user-supplied transformations to each `ThreadSafeModule` before compilation. The transformation function takes `(ThreadSafeModule, MaterializationResponsibility&)` and returns `Expected<ThreadSafeModule>`:

```cpp
// Build an optimization pipeline in an IRTransformLayer:
auto OptimizeModule = [](ThreadSafeModule TSM,
                         MaterializationResponsibility &R)
    -> Expected<ThreadSafeModule> {
  TSM.withModuleDo([](Module &M) {
    PassBuilder PB;
    LoopAnalysisManager LAM;
    FunctionAnalysisManager FAM;
    CGSCCAnalysisManager CGAM;
    ModuleAnalysisManager MAM;
    
    PB.registerModuleAnalyses(MAM);
    PB.registerCGSCCAnalyses(CGAM);
    PB.registerFunctionAnalyses(FAM);
    PB.registerLoopAnalyses(LAM);
    PB.crossRegisterProxies(LAM, FAM, CGAM, MAM);
    
    // Use per-module O2 pipeline (inlining + vectorization):
    ModulePassManager MPM =
        PB.buildPerModuleDefaultPipeline(OptimizationLevel::O2);
    MPM.run(M, MAM);
  });
  return std::move(TSM);
};

auto TransformLayer = std::make_unique<IRTransformLayer>(
    *ES, *CompileLayer, std::move(OptimizeModule));
```

### 108.3.4 CompileOnDemandLayer

[`CompileOnDemandLayer`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/CompileOnDemandLayer.cpp) enables per-function lazy compilation. Instead of compiling an entire module when any symbol is needed, COD partitions the module into individual functions, replaces each function body with a call-through stub, and defers compilation until the stub is first invoked. This is covered in depth in Section 108.4.

### 108.3.5 IRSpeculationLayer

[`IRSpeculationLayer`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/Speculation.cpp) works with COD to pre-compile functions likely to be called soon. A `Speculator` object tracks call frequency: when function A is called, the speculation layer records this and pre-schedules compilation of A's callees that have not yet been compiled. This combines the memory efficiency of lazy compilation (only pay for what you call) with reduced latency for warm code paths.

---

## 108.4 Lazy Compilation and COD

### 108.4.1 The Three-Level Indirection

Lazy compilation via `CompileOnDemandLayer` involves a three-level indirection scheme that is transparent to callers:

```
Before first call:
  Caller code → CALL to stub address
  Stub → LazyCallThroughManager → resolver → compile → patch

After first call (patched):
  Caller code → CALL to stub address
  Stub → direct branch to compiled function address
```

The key insight: once the function is compiled, the stub is "patched" to jump directly to the compiled function. Subsequent calls pay only the cost of one additional indirect branch through the stub — acceptable for most workloads.

### 108.4.2 LazyCallThroughManager

[`LazyCallThroughManager`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/LazyReexports.cpp) manages a pool of trampolines. A trampoline is a small piece of platform-specific machine code (in `compiler-rt/lib/orc/`) that:

1. Saves all caller-saved registers according to the platform ABI (a full register dump)
2. Calls the resolver function with the trampoline's own identity as argument
3. The resolver finds the corresponding logical function and triggers compilation if not yet done
4. After compilation, the resolver updates the stub to point directly at the function
5. The trampoline uses the resolved address to tail-call into the compiled function
6. The register save/restore overhead is paid only on the first call

Trampoline assembly for x86_64:

```asm
; compiler-rt/lib/orc/OrcX86_64.S (simplified):
__orc_rt_reenter:
  ; Save all integer and SSE registers:
  pushq  %rax
  pushq  %rbx
  ...
  pushq  %r15
  subq   $128, %rsp     ; red zone + XMM save area
  movaps %xmm0, (%rsp)
  ...
  
  ; Call the resolver with the trampoline address:
  movq   trampoline_id(%rip), %rdi
  callq  __orc_rt_resolve   ; returns compiled function address
  
  ; Restore registers and tail-call into compiled function:
  movaps (%rsp), %xmm0
  ...
  addq   $128, %rsp
  popq   %r15
  ...
  popq   %rax
  jmpq   *%rax           ; jump to compiled function
```

AArch64 trampolines use `compiler-rt/lib/orc/OrcAarch64.S` with similar save/restore logic.

### 108.4.3 IndirectStubsManager

[`IndirectStubsManager`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/Orc/IndirectionUtils.h) manages the stubs. Each stub is a pair of memory locations:

- **Pointer location** (writable data page): stores the current target address (initially the trampoline, later the compiled function)
- **Stub code** (executable code page): contains `jmpq *(%rip + offset)` on x86_64 or `ldr x16, [pc+8]; br x16` on AArch64 — an indirect jump through the pointer location

When the resolver patches a stub, it writes the compiled function's address to the writable pointer location. The executable stub code does not change. This design respects W^X on Apple Silicon and AArch64 systems that prohibit simultaneously writable and executable pages.

```
Stub layout (x86_64):
  Executable page:          Data page (writable):
  [jmpq *[rip+offset]] -->  [pointer: initially trampoline address]
                             (patched to function address after compilation)
```

### 108.4.4 COD Partition Strategy

When `CompileOnDemandLayer::emit` receives an IR module, it:

1. Extracts each function's signature into a "stub module" — a module containing only external declarations
2. Wraps each original function body in its own `IRMaterializationUnit`
3. Creates stubs for each function, pointing to trampolines
4. Registers the stubs in the JITDylib, so that `ES->lookup("myFunc")` returns the stub address

Adding a 1000-function module creates 1000 stubs but compiles nothing. The compilation of each function is deferred until the first call through that function's stub.

### 108.4.5 Thread Safety of Concurrent Stub Calls

If two threads simultaneously call an uncompiled stub:
1. Both threads hit the trampoline and call the resolver
2. The resolver uses the session's symbol state machine to ensure only one thread triggers materialization
3. The second thread blocks in `ES->lookup` (waiting for the symbol to enter the `Ready` state)
4. After the first thread's compilation completes, `notifyResolved` + `notifyEmitted` are called
5. The second thread's lookup returns the live address; the stub is patched; both threads execute the compiled function

The session's `SymbolState` machine has states: `Linking → Resolved → Emitted → Ready`. Lookups for a symbol in `Linking` or `Resolved` state block until `Ready`.

---

## 108.5 Concurrent Compilation

### 108.5.1 ThreadSafeContext and ThreadSafeModule

LLVM's `LLVMContext` is explicitly not thread-safe. A module's IR — its types, constants, metadata — cannot be accessed from two threads simultaneously. ORC wraps these with two types:

[`ThreadSafeContext`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/Orc/ThreadSafeModule.h#L30): a `shared_ptr` to an `LLVMContext` paired with a `std::mutex`. Any code that needs to access the context must acquire the lock. Multiple `ThreadSafeModule`s can share one `ThreadSafeContext` (if they share a context), or each can own its own.

[`ThreadSafeModule`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/Orc/ThreadSafeModule.h#L72): a `unique_ptr` to an `llvm::Module` paired with a `ThreadSafeContext`. The `withModuleDo(callable)` method acquires the context lock and calls the callable with a `Module&`. This is the only safe way to access the module's IR in a concurrent ORC setup.

```cpp
// Best practice: one context per module (fully independent compilation):
auto Ctx = std::make_unique<LLVMContext>();
auto M = std::make_unique<Module>("jit_module_42", *Ctx);
// ... populate M with IR ...

ThreadSafeModule TSM(std::move(M), std::move(Ctx));

// Correct: use withModuleDo to access the module:
TSM.withModuleDo([](Module &M) {
  // Context lock is held here
  for (Function &F : M)
    F.setCallingConv(CallingConv::Fast);
});

// Move TSM into the compile layer:
cantFail(IRLayer->add(MainJD, std::move(TSM)));
// TSM is now owned by the layer; don't use it after this
```

### 108.5.2 ConcurrentIRCompiler

[`ConcurrentIRCompiler`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/Orc/CompileUtils.h#L85) wraps a `JITTargetMachineBuilder` and creates a fresh `TargetMachine` for each compilation call. Since each `TargetMachine` is independent, `ConcurrentIRCompiler` can safely be called from multiple threads simultaneously:

```cpp
auto JTMB = JITTargetMachineBuilder::detectHost();
if (!JTMB) exitOnError(JTMB.takeError());

// Set optimization level:
JTMB->setCodeGenOptLevel(CodeGenOptLevel::Aggressive);

// Optional: enable specific CPU features:
JTMB->setCPU(sys::getHostCPUName().str());
SubtargetFeatures Features;
sys::getHostCPUFeatures(FeatureMap);
for (auto &F : FeatureMap)
  Features.AddFeature(F.first(), F.second);
JTMB->addFeatures(Features.getString());

auto Compiler = std::make_unique<ConcurrentIRCompiler>(std::move(*JTMB));
auto IRLayer = std::make_unique<IRCompileLayer>(
    *ES, *ObjLayer, std::move(Compiler));
```

### 108.5.3 Task Dispatch and Thread Pools

`ExecutionSession::setDispatchTask` accepts a `DispatchTaskFunction` that schedules materializer work:

```cpp
// Using LLVM's thread pool:
auto TP = std::make_unique<DefaultThreadPool>(
    optimal_concurrency());

ES->setDispatchTask([&TP](std::unique_ptr<Task> T) {
  TP->async([T = std::move(T)]() mutable { T->run(); });
});
```

The `Task` abstraction (introduced in LLVM 14) encapsulates a `MaterializationUnit::materialize` call plus its `MaterializationResponsibility`. Tasks can also represent symbol-table update work. The dispatcher must be able to handle arbitrary numbers of concurrent tasks — if it blocks waiting for other tasks, deadlocks can occur.

### 108.5.4 Deadlock Avoidance

The primary deadlock scenario: a materializer calls `ES->lookup()` while the session is trying to schedule that materializer's dependencies. ORC prevents this through:

1. **Dependency declaration**: `MaterializationResponsibility::addDependenciesForAll` lets a materializer declare which symbols it needs before it runs. The session can then schedule dependencies without blocking.

2. **Async lookup**: use `ES->lookup(SearchOrder, Symbols, callback)` — the async variant — to avoid blocking the materializer's thread. The session calls the callback (on any thread) when symbols resolve.

3. **Documentation rule**: materializers must not call `ES->lookup` (synchronous, blocking form) from within their `materialize()` function. This rule is enforced in debug builds via a "materialization lock" check.

---

## 108.6 Symbol Lookup and Resolution

### 108.6.1 Search Order and Lookup Flags

[`ExecutionSession::lookup`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/Orc/Core.h#L1600) takes a `JITDylibSearchOrder` — a vector of `(JITDylib*, JITDylibLookupFlags)` pairs. Two flag values control the search:

- `MatchExportedSymbolsOnly`: only find symbols with `JITSymbolFlags::Exported` set — the default and most common
- `MatchAllSymbols`: also matches non-exported (hidden/local) symbols; used for debugger queries

```cpp
// Synchronous lookup (blocks until all symbols are ready):
auto Syms = ES->lookup(
    {{&MainJD, JITDylibLookupFlags::MatchExportedSymbolsOnly},
     {&ProcessJD, JITDylibLookupFlags::MatchExportedSymbolsOnly}},
    {ES->intern("printf"), ES->intern("malloc")});
if (!Syms) exitOnError(Syms.takeError());
auto PrintfAddr = (*Syms)[ES->intern("printf")].getAddress();

// Async lookup (non-blocking; callback called from any thread):
ES->lookup(
    LookupKind::Static,
    {{&MainJD, JITDylibLookupFlags::MatchExportedSymbolsOnly}},
    SymbolLookupSet({ES->intern("myFunc")}),
    SymbolState::Ready,
    [](Expected<SymbolMap> Result) {
      if (!Result) { logAllUnhandledErrors(Result.takeError(), errs()); return; }
      auto Addr = (*Result)[/*key*/].getAddress();
      // use Addr...
    },
    NoDependenciesToRegister);
```

### 108.6.2 DynamicLibrarySearchGenerator

[`DynamicLibrarySearchGenerator`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/DynamicLibrarySearchGenerator.cpp) automatically provides process symbols (and symbols from explicitly loaded shared libraries) to a JITDylib. It uses `sys::DynamicLibrary::SearchForAddressOfSymbol`, which calls `dlsym(RTLD_DEFAULT, name)` on POSIX systems.

```cpp
// Add all process symbols to MainJD:
auto DL = cantFail((*JTMB).getDefaultDataLayoutForTarget());
if (auto DLSG = DynamicLibrarySearchGenerator::GetForCurrentProcess(
                    DL.getGlobalPrefix())) {
  MainJD.addGenerator(std::move(*DLSG));
}

// Add a specific shared library:
if (auto DLSG = DynamicLibrarySearchGenerator::Load(
                    "/usr/lib/libm.so.6",
                    DL.getGlobalPrefix())) {
  MainJD.addGenerator(std::move(*DLSG));
}
```

The `GlobalPrefix` is `'_'` on Darwin (MachO) and `'\0'` (empty) on Linux ELF — accounting for the C naming convention difference between the two platforms.

### 108.6.3 StaticLibraryDefinitionGenerator

[`StaticLibraryDefinitionGenerator`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/StaticLibraryDefinitionGenerator.cpp) reads an archive (`.a` file) and adds members on demand. It uses LLVM's `Archive` reader to find which archive members define which symbols, then materializes only those members that are needed:

```cpp
auto ArchiveBuf = MemoryBuffer::getFile("mylib.a");
auto Gen = StaticLibraryDefinitionGenerator::Load(
    ObjLayer, ArchiveBuf->get());
MainJD.addGenerator(std::move(*Gen));
// Only the archive members containing symbols that are actually looked up
// will be linked and executed — lazy library loading.
```

### 108.6.4 AbsoluteSymbols

For pre-defined addresses (runtime helpers, callback functions, global state pointers), `absoluteSymbols` inserts them directly without creating any `MaterializationUnit`:

```cpp
SymbolMap RuntimeSymbols;
RuntimeSymbols[ES->intern("__my_gc_allocate")] =
    {ExecutorAddr::fromPtr(&GC_malloc),
     JITSymbolFlags::Exported | JITSymbolFlags::Callable};
RuntimeSymbols[ES->intern("__my_gc_root")] =
    {ExecutorAddr::fromPtr(&gc_root_list),
     JITSymbolFlags::Exported};

cantFail(MainJD.define(absoluteSymbols(std::move(RuntimeSymbols))));
```

### 108.6.5 Weak Symbols and Reexports

ORC handles weak symbols by priority: a strong definition in a dylib searched before the weak definition wins. `reexports` creates new symbol bindings in a JITDylib that point to symbols in another dylib — implementing symbol aliasing and versioning without copying the actual code:

```cpp
// Expose "fast_malloc" from MathJD as "malloc" in MainJD:
cantFail(MainJD.define(
    reexports(MathJD, {{ES->intern("malloc"), ES->intern("fast_malloc"),
                        JITSymbolFlags::Exported}})));
```

---

## 108.7 ExecutorProcessControl — Multi-Process JIT

### 108.7.1 Motivation and Architecture

`ExecutorProcessControl` (EPC) separates the **compiler side** from the **executor side**. This enables:

- **LLDB expression evaluator**: LLDB compiles expressions in the debugger's process; code executes in the debuggee's address space via `ptrace`
- **Sandboxed JIT**: a JIT server compiles untrusted code; execution happens in an isolated process that cannot access the compiler's memory
- **Cross-architecture JIT**: compiler runs on x86_64 host; executor runs on AArch64 (via QEMU or real hardware)

```
 ┌─────────────────────────────────────┐     ┌──────────────────────────────┐
 │  COMPILER PROCESS                   │     │  EXECUTOR PROCESS            │
 │                                     │     │                              │
 │  ExecutionSession                   │     │  llvm-jitlink-executor       │
 │  ├─ JITDylib "main"                 │     │  ├─ OrcRPCTargetProcessCtrl  │
 │  ├─ IRTransformLayer                │ IPC │  ├─ InProcessMemoryManager   │
 │  ├─ IRCompileLayer                  │<--->│  │  (in executor address     │
 │  └─ ObjectLinkingLayer              │     │  │   space)                  │
 │                                     │     │  └─ Trampolines for lazy     │
 │  SimpleRemoteEPC → sends:           │     │     compilation              │
 │    memory alloc/write/exec requests │     │                              │
 └─────────────────────────────────────┘     └──────────────────────────────┘
```

### 108.7.2 SelfExecutorProcessControl

[`SelfExecutorProcessControl`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/TargetProcess/TargetExecutionUtils.cpp) is the default in-process EPC. All EPC operations (memory allocation, symbol lookup, trampoline invocation) call directly into functions in the same process. This has zero IPC overhead and is used by all standard in-process JIT configurations including `LLJIT`.

### 108.7.3 SimpleRemoteEPC

[`SimpleRemoteEPC`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/SimpleRemoteEPC.cpp) communicates with an executor process over a bidirectional channel (file descriptor pair, socket, or any `SimpleRemoteEPCTransport` implementation). The ORC RPC protocol is:

| Message | Direction | Purpose |
|---------|-----------|---------|
| `MemoryManager::allocate` | Compiler → Executor | Reserve pages in executor |
| `MemoryManager::finalize` | Compiler → Executor | Write bytes, set permissions |
| `MemoryManager::deallocate` | Compiler → Executor | Free pages |
| `EHFrame::registerFrames` | Compiler → Executor | Register unwinding data |
| `JIT::dispatch` | Executor → Compiler | Notify trampoline hit → compile |
| `JIT::resolveSymbols` | Compiler → Executor | Patch stubs post-compile |

The executor binary `llvm-jitlink-executor` (`llvm/tools/llvm-jitlink-executor/`) implements the executor side, reading commands from `stdin`/`stdout`. It handles the `mmap`/`mprotect` calls and trampoline registrations in its own address space.

### 108.7.4 Cross-Architecture JIT Example

```bash
# Host: x86_64. Target: AArch64. QEMU user-mode emulation.
# Start the executor in QEMU:
qemu-aarch64 /path/to/llvm-jitlink-executor --listen-fd=3 &

# Compiler side: connect to the executor via SimpleRemoteEPC:
# (In C++: SimpleRemoteEPC::Create with the pipe fds)
# Then build an ORC stack with JITTargetMachineBuilder("aarch64-linux-gnu")
# Compiled code runs in the QEMU-emulated AArch64 process
```

---

## 108.8 LLJIT and Production Users

### 108.8.1 LLJIT — Batteries-Included ORC

[`LLJIT`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/Orc/LLJIT.h) pre-assembles the standard ORC layer stack, detects the host target, initializes platform support (C++ static initializers, EH frame registration), and provides a high-level `add`/`lookup` API:

```cpp
#include "llvm/ExecutionEngine/Orc/LLJIT.h"
using namespace llvm;
using namespace llvm::orc;

ExitOnError ExitOnErr;

// Build an LLJIT with 4 compile threads and O2 optimization:
auto JIT = ExitOnErr(LLJITBuilder()
    .setNumCompileThreads(4)
    .create());

// Add process symbols:
auto &JD = JIT->getMainJITDylib();
JD.addGenerator(ExitOnErr(
    DynamicLibrarySearchGenerator::GetForCurrentProcess(
        JIT->getDataLayout().getGlobalPrefix())));

// Add a module (will be compiled by one of 4 threads on demand):
auto Ctx = std::make_unique<LLVMContext>();
auto M = buildMyModule(*Ctx);  // returns unique_ptr<Module>
ExitOnErr(JIT->addIRModule(
    ThreadSafeModule(std::move(M), std::move(Ctx))));

// Look up and call a function:
auto FnSym = ExitOnErr(JIT->lookup("computeAnswer"));
auto *Fn = FnSym.toPtr<int(int)>();
int result = Fn(42);   // compiles on first call if lazy=true
```

`LLJITBuilder` options:

| Method | Effect |
|--------|--------|
| `setNumCompileThreads(N)` | Use N threads for concurrent IR compilation |
| `setJITTargetMachineBuilder(JTMB)` | Use a custom target machine configuration |
| `setPlatformSetUp(fn)` | Override platform initialization (C++ init, EH) |
| `setObjectLinkingLayerCreator(fn)` | Custom ObjectLinkingLayer (e.g., with plugins) |
| `setCompileFunctionCreator(fn)` | Custom compiler (e.g., external compilation service) |

### 108.8.2 Kaleidoscope JIT Tutorial

The Kaleidoscope JIT (`llvm/examples/Kaleidoscope/BuildingAJIT/`) demonstrates ORC incrementally across five chapters of the tutorial. Chapter 1 wraps `LLJIT`; subsequent chapters add:
- Chapter 2: lazy compilation via `CompileOnDemandLayer`
- Chapter 3: remote execution via `SimpleRemoteEPC`
- Chapter 4: custom passes in `IRTransformLayer` with optimizer
- Chapter 5: JIT symbols visible to each other across modules

Reading the Kaleidoscope JIT alongside this chapter is the recommended learning path.

### 108.8.3 LLDB Expression Evaluator

LLDB's expression evaluator (`lldb/source/Expression/LLVMUserExpression.cpp`, `IRExecutionUnit.cpp`) is the most demanding production ORC user. The workflow:

1. User types `expr a + b` in LLDB
2. LLDB creates a Clang ASTContext wrapping the debuggee's type information
3. Clang parses and type-checks the expression against the debuggee's types
4. CodeGen lowers the Clang AST to LLVM IR
5. The IR module is added to an ORC JIT configured with a custom EPC that writes to the debuggee's memory via `ptrace`/`process_vm_writev`
6. ORC compiles, links against the debuggee's symbols, and calls the function
7. The result is read back from the debuggee's memory

The LLDB EPC maps compiler-side virtual addresses to executor (debuggee) addresses, handles symbol resolution against the debuggee's dyld/rtld state, and registers debug info so LLDB can report source info for the expression itself.

### 108.8.4 Julia

Julia's JIT (`src/jitlayers.cpp` in the Julia source tree) uses ORC v2 heavily:

- `JuliaOJIT`: wraps `ObjectLinkingLayer` + `IRCompileLayer` + custom optimization pass
- Uses `CompileOnDemandLayer` semantics (Julia implements its own COD-like mechanism via method instances)
- `ResourceTracker`-based code GC: when a method is redefined (`eval`), the old method's compiled code is removed via `RT->remove()`
- Custom `IRTransformLayer` for Julia-specific lowering (GC roots, bounds check removal, alias annotations)
- `DynamicLibrarySearchGenerator` for `ccall`/`@ccall` to call C libraries

---

## 108.9 Debugging JIT-Compiled Code

### 108.9.1 GDB JIT Interface

GDB's JIT debug interface defines a protocol via a pair of symbols in the JIT'd process:

- `__jit_debug_descriptor`: a global `struct jit_descriptor` containing a linked list of `jit_code_entry` nodes, each pointing to an object file in memory
- `__jit_debug_register_code`: a global function that GDB places a breakpoint on

When the JIT adds code, it:
1. Allocates a `jit_code_entry` and writes a pointer to a copy of the linked object file
2. Sets `__jit_debug_descriptor.action_flag = JIT_REGISTER`
3. Calls `__jit_debug_register_code()`
4. GDB's breakpoint fires; GDB reads the new `jit_code_entry` and loads its debug info

The JITLink plugin that implements this is [`DebugObjectManagerPlugin`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/DebugObjectManagerPlugin.cpp):

```cpp
// Register the GDB plugin with ObjectLinkingLayer:
ObjLayer.addPlugin(std::make_unique<DebugObjectManagerPlugin>(
    *ES,
    ExitOnErr(GDBJITDebugInfoRegistrationPlugin::Create(*ES, ProcessJD,
                                                         *EPC)),
    /*AutoRegisterCode=*/true,
    /*EmitDebugObjects=*/true));
```

### 108.9.2 Perf Integration

`PerfJITEventListener` emits `jit_code_load` records to `/tmp/jit-<PID>.dump` in the format expected by `perf inject --jit`. The records include:
- Function name (from the `Symbol` in the `LinkGraph`)
- Load address and code size
- Optional ELF object file bytes for source annotation with DWARF info

```bash
# Profile a JIT program:
perf record -g -k 1 ./my_jit_program
perf inject --jit -i perf.data -o perf.jit.data
perf report -i perf.jit.data
# JIT-compiled frames appear by name in the profile
```

The `ObjectLinkingLayer` integration:

```cpp
ObjLayer.registerJITEventListener(
    *JITEventListener::createPerfJITEventListener());
// Or explicitly via plugin:
ObjLayer.addPlugin(std::make_unique<PerfSupportPlugin>(
    *EPC, MainJD, /*EmitDebugInfo=*/true, /*RegisterPerf=*/true));
```

### 108.9.3 llvm-jitlink as Debugging Tool

`llvm-jitlink` is the essential JITLink/ORC debugging tool:

```bash
# JIT-link and execute an ELF object:
llvm-jitlink -entry=main foo.o bar.o -dlopen /usr/lib/libm.so.6

# Dump the LinkGraph (pre/post passes):
llvm-jitlink -show-graphs -noexec foo.o

# Verify relocations without executing:
llvm-jitlink -verify -noexec foo.o

# Out-of-process execution:
llvm-jitlink foo.o --oop-executor=./llvm-jitlink-executor --oop-executor-connect=launchDeferred

# Show all defined symbols:
llvm-jitlink -show-defined-symbols foo.o bar.o

# Benchmark compile+link time:
time llvm-jitlink -noexec foo.o
```

### 108.9.4 ORC Logging and Debugging

ORC provides verbose logging via LLVM's debug infrastructure:

```bash
# Enable all ORC debug output (very verbose):
./my_jit 2>&1 | grep "^ORC:"

# Programmatically enable debug logging:
llvm::setCurrentDebugType("orc");

# Key debug types:
#   orc          — session-level events
#   orc-speculate — speculation events
#   orc-lazy     — lazy compilation events
#   jitlink      — JITLink pipeline events
```

---

## Chapter 108 Summary

- ORC v2 is LLVM's third-generation JIT, designed for concurrency, laziness, and multi-process execution; it supersedes both the original `ExecutionEngine` and MCJIT by replacing whole-module/single-threaded compilation with a session-based, concurrent, resource-trackable architecture.
- `ExecutionSession` is the root object owning all JIT state; `JITDylib` provides symbol namespaces with configurable search order; `ResourceTracker` enables code removal for hot-reload and GC-based runtimes.
- The layer model (`IRTransformLayer` → `IRCompileLayer` → `ObjectLinkingLayer`) composes transformations as independent objects; each layer receives a `MaterializationUnit` and delegates transformed results downward.
- `CompileOnDemandLayer` implements lazy per-function compilation via platform-specific trampolines in `compiler-rt/lib/orc/`, `LazyCallThroughManager`, and `IndirectStubsManager`; stubs use separate writable (pointer) and executable (code) pages to respect W^X.
- `ThreadSafeModule`/`ThreadSafeContext` ensure LLVM Context is accessed under a mutex; `ConcurrentIRCompiler` creates a fresh `TargetMachine` per compilation thread; the thread pool dispatcher enables true concurrent compilation.
- `ExecutorProcessControl` (EPC) separates compiler from executor: `SelfExecutorProcessControl` for in-process JIT, `SimpleRemoteEPC` + `llvm-jitlink-executor` for out-of-process/cross-architecture JIT.
- `LLJIT` is the high-level convenience API with `LLJITBuilder`; `DynamicLibrarySearchGenerator` and `StaticLibraryDefinitionGenerator` provide process and archive symbols on demand.
- GDB integration uses `DebugObjectManagerPlugin` + `__jit_debug_descriptor`/`__jit_debug_register_code`; perf integration uses `PerfSupportPlugin` with `/tmp/jit-<PID>.dump`; `llvm-jitlink` is the standalone debugging/testing tool.
- Production users include LLDB (custom EPC via ptrace), Julia (ResourceTracker-based code GC), and Swift REPL; all have moved or are moving to ORC v2 from older JIT infrastructure.


---

@copyright jreuben11
