# Chapter 149 — The Pass Infrastructure

*Part XXI — MLIR Transformations*

MLIR's pass infrastructure is the organizational backbone of every compiler built on the framework. It manages pass scheduling, nested execution, parallelism, analysis caching, and instrumentation in a uniform way that scales from small research compilers to production systems like IREE and XLA. Unlike LLVM's legacy pass manager, MLIR's design enforces strict isolation between passes—no shared mutable state, no interprocedural assumptions without explicit analysis requests—which enables safe parallel execution and composable pipeline construction. This chapter covers everything from writing a minimal pass to running a deeply nested, instrumented pass pipeline.

---

## 149.1 Pass Types

MLIR provides three pass base classes, each expressing a different constraint on the operations the pass may inspect and modify.

### OperationPass\<OpTy\>

The most common form. A pass parameterized by `OpTy` is scheduled on every op of that type found during the pipeline walk:

```cpp
// Runs on every func::FuncOp in the module:
struct MyFuncPass : mlir::OperationPass<mlir::func::FuncOp> { ... };

// Runs on the top-level builtin::ModuleOp:
struct MyModulePass : mlir::OperationPass<mlir::ModuleOp> { ... };
```

The pass infrastructure guarantees:
- `getOperation()` returns an op of exactly type `OpTy`.
- The pass only sees ops nested inside `OpTy`; it cannot reach sibling or parent ops without explicit analysis queries.
- Parallel execution across multiple ops of the same type is safe if the pass has no shared mutable state (see §149.4).

### InterfacePass\<IfTy\>

Runs on any op implementing the interface `IfTy`. Useful for cross-dialect passes that work on any callable or any symbol table:

```cpp
struct MyCallablePass : mlir::InterfacePass<mlir::CallableOpInterface> {
  void runOnOperation() override {
    mlir::CallableOpInterface callable = getOperation();
    // ...
  }
};
```

### Pass (unparameterized)

The generic base class. `getOperation()` returns `Operation *`. Rarely used directly because it prevents the infrastructure from scheduling the pass safely in parallel.

### Required declarations

Every pass must provide:

| Method | Purpose |
|--------|---------|
| `getName()` | Human-readable pass name; used in diagnostics |
| `getArgument()` | CLI flag (e.g., `"my-func"` → `--my-func`) |
| `getDependentDialects(DialectRegistry &)` | Declare dialects the pass may introduce |
| `runOnOperation()` | The pass logic |

`getDependentDialects` is critical: if a pass creates ops from a dialect not yet loaded, MLIR will crash. Declaring the dialect here causes it to be loaded before the pass runs.

---

## 149.2 Writing a Pass

```cpp
#include "mlir/Pass/Pass.h"
#include "mlir/Dialect/Func/IR/FuncOps.h"
#include "mlir/Dialect/Arith/IR/Arith.h"

namespace {

struct CountArithOpsPass
    : mlir::OperationPass<mlir::func::FuncOp> {
  MLIR_DEFINE_EXPLICIT_INTERNAL_INLINE_TYPE_ID(CountArithOpsPass)

  // CLI argument: --count-arith-ops
  llvm::StringRef getArgument() const override {
    return "count-arith-ops";
  }
  llvm::StringRef getName() const override {
    return "CountArithOpsPass";
  }
  llvm::StringRef getDescription() const override {
    return "Count arithmetic operations in each function";
  }

  // Statistics: accessible via --mlir-pass-statistics
  mlir::Pass::Statistic numArithOps{
      this, "num-arith-ops", "Number of arith ops found"};

  void runOnOperation() override {
    mlir::func::FuncOp func = getOperation();
    func.walk([&](mlir::Operation *op) {
      if (op->getDialect()->getNamespace() == "arith")
        ++numArithOps;
    });
  }
};

} // namespace

// Registration in a plugin or library init function:
std::unique_ptr<mlir::Pass> createCountArithOpsPass() {
  return std::make_unique<CountArithOpsPass>();
}

void registerCountArithOpsPass() {
  mlir::PassRegistration<CountArithOpsPass>();
}
```

### MLIR_DEFINE_EXPLICIT_INTERNAL_INLINE_TYPE_ID

This macro generates a static `TypeID` for the pass class, used by the pass manager for safe downcasting. It must appear in the body of every concrete pass class. Forgetting it causes undefined behavior if two pass classes hash to the same pointer.

### signalPassFailure

If the pass detects an error it cannot recover from, call `signalPassFailure()`. This sets a flag that causes `PassManager::run` to return `failure()` after the current op finishes. It does not throw or longjmp; the pass continues running normally after the call. Accessing `getOperation()` after `signalPassFailure()` is still valid.

---

## 149.3 Pass Manager and Nesting

The `PassManager` is the execution engine for pipelines.

### Flat pipeline

```cpp
mlir::MLIRContext ctx;
mlir::PassManager pm(&ctx);

// Add passes at the module level:
pm.addPass(mlir::createCanonicalizerPass());
pm.addPass(mlir::createCSEPass());
pm.addPass(createCountArithOpsPass());

// Execute the pipeline:
mlir::ModuleOp module = ...; // parsed or built
if (mlir::failed(pm.run(module))) {
  llvm::errs() << "Pass pipeline failed\n";
  return 1;
}
```

### Nested pipeline

Nesting is the mechanism for applying passes to each op of a given type independently. This enables parallel execution and keeps pass logic scoped:

```cpp
// Top-level passes operate on the module:
pm.addPass(mlir::createSymbolDCEPass());

// Nested pass manager for func.func: each func.func is processed independently.
mlir::OpPassManager &funcPM = pm.nest<mlir::func::FuncOp>();
funcPM.addPass(mlir::createCanonicalizerPass());
funcPM.addPass(mlir::createCSEPass());
funcPM.addPass(createCountArithOpsPass());

// Deeper nesting: an affine.for nested inside func.func:
mlir::OpPassManager &loopPM =
    funcPM.nest<mlir::affine::AffineForOp>();
loopPM.addPass(mlir::createAffineLoopInvariantCodeMotionPass());
```

The nesting hierarchy mirrors the MLIR op hierarchy. Nested passes cannot see ops outside their subtree; this isolation is what makes parallel execution safe.

### Pipeline string syntax

The text form of a pipeline, used with `-pass-pipeline` or `mlir-opt`:

```
builtin.module(
  symbol-dce,
  func.func(
    canonicalize,
    cse,
    count-arith-ops
  )
)
```

As a single string for `mlir-opt`:

```bash
/usr/lib/llvm-22/bin/mlir-opt input.mlir \
  --pass-pipeline="builtin.module(symbol-dce,func.func(canonicalize,cse))"
```

Passes with options use `pass-name{option=value}` syntax:

```
func.func(my-tiling-pass{tile-size=64},canonicalize)
```

### Programmatic pipeline construction from string

```cpp
std::string pipelineStr =
    "builtin.module(canonicalize,func.func(cse))";
if (mlir::failed(mlir::parsePassPipeline(pipelineStr, pm))) {
  llvm::errs() << "Failed to parse pipeline\n";
  return 1;
}
```

---

## 149.4 Parallelism

MLIR's pass manager can execute passes on sibling ops of the same type in parallel. This is the most important performance feature for large programs with many `func.func` ops.

### Enabling threading

```cpp
// Default: uses llvm::parallel::DefaultThreadPool
pm.enableMultithreading();

// Custom thread pool (e.g., from IREE's execution environment):
llvm::ThreadPool myPool(llvm::hardware_concurrency(8));
pm.enableMultithreading(myPool);

// Disable for debugging (forces sequential execution):
pm.enableMultithreading(false);
```

From the command line:

```bash
# Disable threading for reproducible debug output:
/usr/lib/llvm-22/bin/mlir-opt --mlir-disable-threading input.mlir
```

### Safety requirements

A pass is safe for parallel execution if and only if:

1. Its class has no non-`const` data members mutated in `runOnOperation()`.
2. Any external state it reads (e.g., a lookup table) is read-only after construction.
3. It accesses `MLIRContext` only through the thread-safe API (e.g., `getContext()`, type creation via `IntegerType::get(ctx, ...)` — these are all locked internally).
4. It does not call `llvm::errs()` or other unsynchronized I/O directly (use the diagnostic engine instead).

Pass options (§149.6) are set once before execution and are read-only during `runOnOperation()`, so they are safe.

### ThreadLocalCache

For per-pass heavy-weight objects that are expensive to construct but should not be shared across threads, use `ThreadLocalCache`:

```cpp
struct MyPass : mlir::OperationPass<mlir::func::FuncOp> {
  mlir::ThreadLocalCache<SomeHeavyObject> cache;

  void runOnOperation() override {
    SomeHeavyObject &obj = *cache;  // per-thread instance
    // use obj...
  }
};
```

`ThreadLocalCache` allocates one instance per thread and destroys them when the pass is destroyed.

---

## 149.5 Pass Statistics and Instrumentation

### Pass statistics

```cpp
struct MyPass : mlir::OperationPass<mlir::func::FuncOp> {
  // Declare a statistic:
  mlir::Pass::Statistic numFolded{
      this, "num-folded", "Number of ops folded"};
  mlir::Pass::Statistic numErased{
      this, "num-erased", "Number of ops erased"};

  void runOnOperation() override {
    // Increment:
    ++numFolded;
    numErased += 3;
  }
};
```

Print statistics after the pipeline runs:

```bash
/usr/lib/llvm-22/bin/mlir-opt --canonicalize \
  --mlir-pass-statistics input.mlir
```

Output:

```
===-------------------------------------------------------------------------===
                         ... Pass statistics report ...
===-------------------------------------------------------------------------===
'canonicalize' pass statistics:
  num-folded                   - Number of ops folded: 42
```

Statistics from parallel execution are aggregated atomically across threads.

### PassInstrumentation

`PassInstrumentation` is a hook interface for observing pass execution lifecycle events:

```cpp
struct TimingInstrumentation : mlir::PassInstrumentation {
  void runBeforePass(mlir::Pass *pass, mlir::Operation *op) override {
    // Record start time for this (pass, op) pair.
  }
  void runAfterPass(mlir::Pass *pass, mlir::Operation *op) override {
    // Record end time; emit timing.
  }
  void runAfterPassFailed(mlir::Pass *pass,
                          mlir::Operation *op) override {
    llvm::errs() << "Pass " << pass->getName() << " failed on "
                 << op->getName() << "\n";
  }
};

pm.addInstrumentation(std::make_unique<TimingInstrumentation>());
```

### IR printing instrumentation

```cpp
// Print IR before and after every pass:
pm.enableIRPrinting(
    /*shouldPrintBeforePass=*/[](mlir::Pass *, mlir::Operation *) { return true; },
    /*shouldPrintAfterPass=*/[](mlir::Pass *, mlir::Operation *) { return true; },
    /*printModuleScope=*/true,
    /*printAfterOnlyOnChange=*/false,
    /*printAfterOnlyOnFailure=*/false,
    llvm::errs(),
    mlir::OpPrintingFlags{});
```

Command-line equivalents:

```bash
# Print IR after every pass:
/usr/lib/llvm-22/bin/mlir-opt --mlir-print-ir-after-all input.mlir

# Print IR only when a pass modifies it:
/usr/lib/llvm-22/bin/mlir-opt --mlir-print-ir-after-change input.mlir

# Print the entire module (not just the nested op being processed):
/usr/lib/llvm-22/bin/mlir-opt --mlir-print-ir-module-scope \
  --mlir-print-ir-after-all input.mlir

# Print timing for each pass:
/usr/lib/llvm-22/bin/mlir-opt --canonicalize --mlir-timing input.mlir
```

---

## 149.6 Pass Options

Passes declare options using the `Option` and `ListOption` helpers, which integrate with LLVM's `cl::opt` infrastructure:

```cpp
struct TilingPass : mlir::OperationPass<mlir::func::FuncOp> {
  MLIR_DEFINE_EXPLICIT_INTERNAL_INLINE_TYPE_ID(TilingPass)

  // Single integer option with default:
  mlir::Pass::Option<int64_t> tileSize{
      *this, "tile-size",
      llvm::cl::desc("Tile factor for the outermost loop"),
      llvm::cl::init(32)};

  // Boolean flag:
  mlir::Pass::Option<bool> enableVectorization{
      *this, "vectorize",
      llvm::cl::desc("Enable vectorization after tiling"),
      llvm::cl::init(false)};

  // List option:
  mlir::Pass::ListOption<int64_t> tileSizes{
      *this, "tile-sizes",
      llvm::cl::desc("Tile sizes per loop dimension")};

  void runOnOperation() override {
    int64_t ts = tileSize.getValue();
    bool doVec = enableVectorization.getValue();
    llvm::ArrayRef<int64_t> sizes = tileSizes;
    // ...
  }
};
```

Pass options are parsed from the pipeline string:

```bash
/usr/lib/llvm-22/bin/mlir-opt \
  --pass-pipeline="func.func(tiling-pass{tile-size=64,vectorize=true})" \
  input.mlir
```

Or from C++:

```cpp
TilingPass pass;
pass.tileSize = 64;
pass.enableVectorization = true;
pm.addPass(std::unique_ptr<mlir::Pass>(&pass));
```

### CloneOptions

When the pass manager clones passes for parallel execution (each thread needs its own pass instance), it calls the `clonePass()` virtual method. The default implementation uses the `PassRegistration` mechanism to construct a fresh copy. If a pass has complex initialization (e.g., pre-built lookup tables derived from options), override `clonePass()` to copy that state.

---

## 149.7 Analysis Infrastructure

MLIR separates analyses from transformations. Analyses compute information about the IR; transformations use that information to modify the IR. The `AnalysisManager` caches analyses and invalidates them when the IR changes.

### Requesting an analysis

```cpp
void runOnOperation() override {
  mlir::func::FuncOp func = getOperation();

  // Request dominance information (computed on first request, cached thereafter):
  mlir::DominanceInfo &domInfo = getAnalysis<mlir::DominanceInfo>();

  // Use it:
  func.walk([&](mlir::Operation *op) {
    if (domInfo.dominates(someOp, op)) { /* ... */ }
  });
}
```

### Preserving analyses

After a pass runs, the analysis manager invalidates all analyses by default. If a pass does not modify the IR in ways that affect an analysis, it can declare the analysis preserved:

```cpp
void runOnOperation() override {
  // ... read-only traversal ...
  markAllAnalysesPreserved();
}

void runOnOperation() override {
  // ... modify ops but do not change CFG ...
  markAnalysesPreserved<mlir::DominanceInfo>();
}
```

This avoids recomputing analyses unnecessarily across passes in a nested pipeline.

### Built-in analyses

| Analysis | Header | Description |
|----------|--------|-------------|
| `DominanceInfo` | `mlir/IR/Dominance.h` | Dominator tree for a region |
| `PostDominanceInfo` | `mlir/IR/Dominance.h` | Post-dominator tree |
| `CallGraph` | `mlir/Analysis/CallGraph.h` | Inter-procedural call graph |
| `DataFlowSolver` | `mlir/Analysis/DataFlow/...` | General dataflow framework |
| `SparseConstantPropagation` | `mlir/Analysis/DataFlow/...` | Sparse conditional constant propagation |
| `DeadCodeAnalysis` | `mlir/Analysis/DataFlow/...` | Liveness / dead code detection |
| `LoopAnalysis` | `mlir/Dialect/Affine/Analysis/...` | Affine loop properties |

### Defining a custom analysis

```cpp
struct MyReachabilityAnalysis {
  // Constructor called by the analysis manager:
  explicit MyReachabilityAnalysis(mlir::Operation *op) {
    // compute reachability for all blocks in op's regions
    compute(op);
  }

  bool isReachable(mlir::Block *from, mlir::Block *to) const { ... }

private:
  void compute(mlir::Operation *op) { ... }
  llvm::DenseSet<std::pair<mlir::Block *, mlir::Block *>> reachable;
};

// In a pass:
auto &ra = getAnalysis<MyReachabilityAnalysis>();
```

The analysis manager constructs `MyReachabilityAnalysis` on the first request and caches it. It is destroyed when invalidated (i.e., when the pass manager determines the analysis is no longer valid due to IR changes).

---

## 149.8 Pass Pipelines and Registration

### Registering a pipeline

Beyond individual passes, groups of passes composing a logical unit can be registered as a named pipeline:

```cpp
void registerMyOptPipeline() {
  mlir::PassPipelineRegistration<>(
      "my-opt",
      "My optimization pipeline: canonicalize + CSE + tiling",
      [](mlir::OpPassManager &pm) {
        pm.addPass(mlir::createCanonicalizerPass());
        pm.addPass(mlir::createCSEPass());
        pm.nest<mlir::func::FuncOp>().addPass(createTilingPass());
      });
}
```

After registration:

```bash
/usr/lib/llvm-22/bin/mlir-opt --my-opt input.mlir
```

### Dynamic pass loading

MLIR supports pass plugins: shared libraries that register passes when loaded:

```bash
/usr/lib/llvm-22/bin/mlir-opt \
  --load-dialect-plugin=libMyDialectPlugin.so \
  --my-custom-pass input.mlir
```

The plugin's `mlirGetDialectPluginInfo()` function is called at load time to register dialects and passes.

---

## 149.9 Crash Reproduction

When a pass crashes, the pass manager can emit a crash reproducer: a minimal `.mlir` + pipeline string that reproduces the failure:

```bash
/usr/lib/llvm-22/bin/mlir-opt \
  --mlir-pass-pipeline-crash-reproducer=/tmp/reproducer \
  --canonicalize --cse input.mlir
```

The reproducer file contains the module state just before the crashing pass and the pipeline that was running. It can be fed directly to `mlir-opt` to reproduce the crash. For non-crash failures, `--mlir-pass-failure-dump-pass-pipeline` prints the partial pipeline up to the failing pass.

---

## Chapter Summary

- MLIR passes are parameterized by op type (`OperationPass<T>`) or interface (`InterfacePass<I>`); this constraint enables safe parallel scheduling.
- `getDependentDialects()` must declare all dialects the pass introduces; forgetting it causes crashes when the dialect is not yet loaded.
- `PassManager::nest<T>()` creates a nested sub-manager that runs on each op of type `T`; nesting isolates passes and enables parallel execution.
- Pipeline strings (`"builtin.module(canonicalize,func.func(cse))"`) are the textual representation; parsed with `parsePassPipeline` or passed to `--pass-pipeline`.
- Parallel execution is enabled by `enableMultithreading()`; passes must be stateless (no mutable class members accessed in `runOnOperation()`).
- `Pass::Statistic` provides thread-safe counters; `--mlir-pass-statistics` prints them.
- `PassInstrumentation` hooks observe pass start/finish; `enableIRPrinting` and `--mlir-timing` are built on this mechanism.
- `Option<T>` and `ListOption<T>` expose per-pass CLI options parsed from the pipeline string.
- `getAnalysis<T>()` returns a cached analysis; `markAnalysesPreserved<T>()` prevents unnecessary recomputation.
- `--mlir-pass-pipeline-crash-reproducer` emits minimal reproducers for debugging pass crashes.
