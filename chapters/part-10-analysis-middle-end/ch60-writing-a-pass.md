# Chapter 60 — Writing a Pass

*Part X — Analysis and the Middle-End*

Understanding the pass manager architecture (Chapter 59) is a prerequisite, but writing a pass is the hands-on complement. This chapter builds four passes from scratch — a `FunctionPass`, a `LoopPass`, a `ModulePass`, and a `CGSCCPass` — showing the full lifecycle from implementation through registration, plugin loading, and analysis preservation. The code in this chapter compiles against the LLVM 22.1.x headers installed at `/usr/lib/llvm-22`.

## Table of Contents

- [60.1 A Function Pass: Counting Instructions](#601-a-function-pass-counting-instructions)
  - [60.1.1 Pass Implementation](#6011-pass-implementation)
  - [60.1.2 Registering the Pass](#6012-registering-the-pass)
- [60.2 A Function Pass that Uses Analyses](#602-a-function-pass-that-uses-analyses)
- [60.3 A Loop Pass](#603-a-loop-pass)
  - [60.3.1 Implementation](#6031-implementation)
  - [60.3.2 Inserting into a Pipeline](#6032-inserting-into-a-pipeline)
- [60.4 A Module Pass: Dead Global Removal](#604-a-module-pass-dead-global-removal)
- [60.5 A CGSCC Pass](#605-a-cgscc-pass)
  - [60.5.1 Adding a CGSCC Pass to a Pipeline](#6051-adding-a-cgscc-pass-to-a-pipeline)
- [60.6 Out-of-Tree Pass Plugins](#606-out-of-tree-pass-plugins)
  - [60.6.1 CMake Build](#6061-cmake-build)
  - [60.6.2 Loading the Plugin](#6062-loading-the-plugin)
- [60.7 Preserving Analyses: Detailed Rules](#607-preserving-analyses-detailed-rules)
  - [60.7.1 What to Preserve](#6071-what-to-preserve)
  - [60.7.2 Analysis Updaters](#6072-analysis-updaters)
- [60.8 Debugging a Pass](#608-debugging-a-pass)
  - [60.8.1 Print-Before/After](#6081-print-beforeafter)
  - [60.8.2 Opt-Bisect](#6082-opt-bisect)
  - [60.8.3 `verifyFunction` After Each Pass](#6083-verifyfunction-after-each-pass)
- [Chapter Summary](#chapter-summary)

---

## 60.1 A Function Pass: Counting Instructions

The simplest useful pass iterates over a function's instructions. This example counts by opcode and prints a histogram.

### 60.1.1 Pass Implementation

```cpp
// InstructionCount.h
#include "llvm/IR/PassManager.h"
#include "llvm/IR/Instructions.h"

namespace llvm {

class InstructionCountPass : public PassInfoMixin<InstructionCountPass> {
public:
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM);
  static bool isRequired() { return false; }
};

} // namespace llvm
```

```cpp
// InstructionCount.cpp
#include "InstructionCount.h"
#include "llvm/IR/Function.h"
#include "llvm/Support/raw_ostream.h"
#include <map>

using namespace llvm;

PreservedAnalyses InstructionCountPass::run(Function &F,
                                             FunctionAnalysisManager &AM) {
  std::map<unsigned, unsigned> counts;
  for (BasicBlock &BB : F)
    for (Instruction &I : BB)
      ++counts[I.getOpcode()];

  outs() << "=== " << F.getName() << " ===\n";
  for (auto &[opcode, count] : counts)
    outs() << "  " << Instruction::getOpcodeName(opcode)
           << ": " << count << "\n";

  // This pass only reads IR — all analyses preserved
  return PreservedAnalyses::all();
}
```

Key points:
- `PassInfoMixin<InstructionCountPass>` provides `name()` and `printPipeline()`.
- `run()` accepts a `Function&` and a `FunctionAnalysisManager&`. The function is not modified, so `PreservedAnalyses::all()` is correct.
- `isRequired()` returning `false` allows the pass to be skipped on `optnone` functions.

### 60.1.2 Registering the Pass

Registration makes the pass available by name in `opt -passes=` and in `PassBuilder::parsePassPipeline`:

```cpp
// In the pass plugin registration (see §60.5)
PB.registerPipelineParsingCallback(
    [](StringRef Name, FunctionPassManager &FPM,
       ArrayRef<PassBuilder::PipelineElement> InnerPipeline) -> bool {
      if (Name == "instr-count") {
        FPM.addPass(InstructionCountPass());
        return true;
      }
      return false;
    });
```

Usage:
```bash
opt -load-pass-plugin=./InstructionCount.so \
    -passes='instr-count' -S input.ll
```

## 60.2 A Function Pass that Uses Analyses

A more realistic pass uses dominator trees and loop info. This example identifies loop-invariant loads that could be hoisted (a simplified preview of LICM):

```cpp
class LoopInvariantLoadFinderPass
    : public PassInfoMixin<LoopInvariantLoadFinderPass> {
public:
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM) {
    auto &LI  = AM.getResult<LoopAnalysis>(F);
    auto &DT  = AM.getResult<DominatorTreeAnalysis>(F);
    auto &AA  = AM.getResult<AAManager>(F);

    for (Loop *L : LI) {
      for (BasicBlock *BB : L->getBlocks()) {
        for (Instruction &I : *BB) {
          if (auto *LI_inst = dyn_cast<LoadInst>(&I)) {
            if (L->hasLoopInvariantOperands(LI_inst) &&
                AA.isNoAlias(LI_inst->getPointerOperand(),
                             /* some store */ ...))
              outs() << "Hoistable: " << *LI_inst << "\n";
          }
        }
      }
    }
    return PreservedAnalyses::all(); // read-only
  }
};
```

The call `AM.getResult<LoopAnalysis>(F)` requests `LoopAnalysis`, which internally requests `DominatorTreeAnalysis` (since `LoopInfo` depends on `DominatorTree`). This chain is resolved automatically by the `FunctionAnalysisManager`.

## 60.3 A Loop Pass

Loop passes operate on individual `Loop` objects (and optionally their sub-loops). They are wrapped by `FunctionToLoopPassAdaptor` when inserted into a `FunctionPassManager`.

### 60.3.1 Implementation

```cpp
// SimpleTilingHintPass.h — marks trivially tileable loops with metadata
#include "llvm/Transforms/Scalar/LoopPassManager.h"

class SimpleTilingHintPass : public PassInfoMixin<SimpleTilingHintPass> {
public:
  PreservedAnalyses run(Loop &L, LoopAnalysisManager &AM,
                        LoopStandardAnalysisResults &AR, LPMUpdater &U);
};
```

The loop-pass `run()` signature differs from function/module passes: it receives `LoopStandardAnalysisResults` (a bundle of pre-computed analyses: `DominatorTree`, `LoopInfo`, `ScalarEvolution`, `AssumptionCache`, `TargetLibraryInfo`, `TargetTransformInfo`, `BlockFrequencyInfo`, `MemorySSA`) and `LPMUpdater` (for notifying the pass manager of structural changes to the loop nest).

```cpp
#include "llvm/IR/Metadata.h"
#include "llvm/Analysis/LoopInfo.h"
#include "llvm/Analysis/ScalarEvolution.h"

PreservedAnalyses SimpleTilingHintPass::run(Loop &L,
                                             LoopAnalysisManager &AM,
                                             LoopStandardAnalysisResults &AR,
                                             LPMUpdater &U) {
  // Check if the loop has a simple induction variable with known bounds
  ScalarEvolution &SE = AR.SE;
  if (!L.isLoopSimplifyForm()) return PreservedAnalyses::all();

  const SCEV *BTC = SE.getBackedgeTakenCount(&L);
  if (isa<SCEVCouldNotCompute>(BTC)) return PreservedAnalyses::all();

  // Attach metadata hint for a downstream tiling pass
  LLVMContext &Ctx = L.getHeader()->getContext();
  SmallVector<Metadata*, 4> MDs = {
    MDString::get(Ctx, "tiling.hint"),
    ConstantAsMetadata::get(ConstantInt::get(
        Type::getInt32Ty(Ctx), 16)) // tile size suggestion
  };
  auto *MD = MDNode::get(Ctx, MDs);
  L.getLoopID(); // ensure loop ID is set
  // Add to loop's metadata
  L.setLoopID(MDNode::concatenate(L.getLoopID(), MD));

  PreservedAnalyses PA;
  PA.preserveSet<CFGAnalyses>();  // no CFG change
  return PA;
}
```

### 60.3.2 Inserting into a Pipeline

```cpp
FunctionPassManager FPM;
// Build a LoopPassManager
LoopPassManager LPM;
LPM.addPass(SimpleTilingHintPass());
// Wrap it for function-level execution
FPM.addPass(createFunctionToLoopPassAdaptor(std::move(LPM),
                                             /*UseMemorySSA=*/false,
                                             /*UseBlockFrequencyInfo=*/false));
```

The adaptor iterates over loops in the function in LCSSA form, runs the loop pass on each, and propagates `PreservedAnalyses` to the function level.

## 60.4 A Module Pass: Dead Global Removal

Module passes see the entire module at once and can remove globals, create or merge functions, or perform whole-program transformations.

```cpp
class DeadGlobalRemoverPass : public PassInfoMixin<DeadGlobalRemoverPass> {
public:
  PreservedAnalyses run(Module &M, ModuleAnalysisManager &MAM) {
    SmallVector<GlobalVariable*, 8> dead;
    for (GlobalVariable &GV : M.globals()) {
      if (GV.isDeclaration()) continue;
      if (GV.use_empty() && !GV.hasExternalLinkage())
        dead.push_back(&GV);
    }
    for (GlobalVariable *GV : dead)
      GV->eraseFromParent();

    if (dead.empty()) return PreservedAnalyses::all();

    PreservedAnalyses PA;
    // Removing globals invalidates module-level AA
    // but function-level analyses for surviving functions are unchanged
    PA.preserveSet<AllAnalysesOn<Function>>();
    return PA;
  }
};
```

`AllAnalysesOn<Function>` is a set that preserves all function-level cached analyses — if the pass removes some globals but does not touch function bodies, function-level caches remain valid.

## 60.5 A CGSCC Pass

Call-graph SCC passes run on strongly connected components of the call graph in bottom-up order. They are ideal for inter-procedural analyses that need to see all callers or callees of a function.

```cpp
class RecursiveCallReporterPass
    : public PassInfoMixin<RecursiveCallReporterPass> {
public:
  PreservedAnalyses run(LazyCallGraph::SCC &C,
                        CGSCCAnalysisManager &AM,
                        LazyCallGraph &CG,
                        CGSCCUpdateResult &UR) {
    // An SCC with more than one node contains a cycle — i.e., mutual recursion
    if (C.size() > 1) {
      outs() << "Mutually recursive functions: ";
      for (auto &Node : C)
        outs() << Node.getFunction().getName() << " ";
      outs() << "\n";
    }
    // Single-node SCC: check for self-recursion
    if (C.size() == 1) {
      Function &F = C.begin()->getFunction();
      for (BasicBlock &BB : F)
        for (Instruction &I : BB)
          if (auto *Call = dyn_cast<CallInst>(&I))
            if (Call->getCalledFunction() == &F)
              outs() << F.getName() << " is self-recursive\n";
    }
    return PreservedAnalyses::all();
  }
};
```

CGSCC passes receive a `LazyCallGraph::SCC&` (the SCC being processed), a `CGSCCAnalysisManager&`, the `LazyCallGraph&` (for navigating the full graph), and a `CGSCCUpdateResult&` (for signaling mutations to the call graph — e.g., when inlining creates or removes edges).

### 60.5.1 Adding a CGSCC Pass to a Pipeline

```cpp
ModulePassManager MPM;
CGSCCPassManager CGSCC_PM;
CGSCC_PM.addPass(RecursiveCallReporterPass());
MPM.addPass(createModuleToPostOrderCGSCCPassAdaptor(std::move(CGSCC_PM)));
```

`createModuleToPostOrderCGSCCPassAdaptor` traverses the call graph's SCCs in post-order (bottom-up: callees before callers) and applies the CGSCC pass to each.

## 60.6 Out-of-Tree Pass Plugins

Out-of-tree passes are compiled as shared libraries and loaded into `opt` with `-load-pass-plugin`. The plugin must export a C-linkage entry point:

```cpp
// MyPlugin.cpp
#include "llvm/Plugins/PassPlugin.h"
#include "InstructionCount.h"
#include "SimpleTilingHintPass.h"

using namespace llvm;

extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
  return {
    LLVM_PLUGIN_API_VERSION,
    "MyPlugin",
    "v0.1",
    [](PassBuilder &PB) {
      // Register function passes
      PB.registerPipelineParsingCallback(
          [](StringRef Name, FunctionPassManager &FPM,
             ArrayRef<PassBuilder::PipelineElement>) -> bool {
            if (Name == "instr-count") {
              FPM.addPass(InstructionCountPass());
              return true;
            }
            return false;
          });

      // Register loop passes
      PB.registerPipelineParsingCallback(
          [](StringRef Name, LoopPassManager &LPM,
             ArrayRef<PassBuilder::PipelineElement>) -> bool {
            if (Name == "tiling-hint") {
              LPM.addPass(SimpleTilingHintPass());
              return true;
            }
            return false;
          });

      // Hook into scalar optimizer pipeline after vectorization
      PB.registerVectorizerStartEPCallback(
          [](FunctionPassManager &FPM, OptimizationLevel Level) {
            FPM.addPass(InstructionCountPass());
          });
    }
  };
}
```

The `PassPluginLibraryInfo` struct (declared in
[`llvm/include/llvm/Plugins/PassPlugin.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Plugins/PassPlugin.h)) carries:
- `APIVersion`: must match `LLVM_PLUGIN_API_VERSION`.
- `PluginName`, `PluginVersion`: human-readable identifiers.
- `RegisterPassBuilderCallbacks`: the hook that installs passes into the `PassBuilder`.

### 60.6.1 CMake Build

```cmake
add_library(MyPlugin SHARED MyPlugin.cpp InstructionCount.cpp SimpleTilingHintPass.cpp)
target_link_libraries(MyPlugin PRIVATE LLVMCore LLVMPasses LLVMAnalysis)
set_target_properties(MyPlugin PROPERTIES POSITION_INDEPENDENT_CODE ON)
```

The shared library must NOT link with LLVM as a static library if `opt` is itself dynamically linked against LLVM — they would have two copies of LLVM's global state, breaking the `AnalysisKey` singleton addresses. Use `target_link_libraries` with `PRIVATE` and mark LLVM as an interface dependency (not linked):

```cmake
# Alternative: leave LLVM symbols unresolved (resolved at load time from opt)
target_link_options(MyPlugin PRIVATE "-Wl,--unresolved-symbols=ignore-in-shared-libs")
```

### 60.6.2 Loading the Plugin

```bash
# Load and run a specific pass
opt -load-pass-plugin=./MyPlugin.so \
    -passes='instr-count' -S -o /dev/null input.ll

# Load and inject into the default pipeline
opt -load-pass-plugin=./MyPlugin.so \
    -passes='default<O2>' -S input.ll

# List registered passes (including plugin passes)
opt -load-pass-plugin=./MyPlugin.so -print-passes 2>&1 | grep instr-count
```

`PassPlugin::Load` (called by the driver internally) opens the shared library with `dlopen`, resolves `llvmGetPassPluginInfo`, checks the API version, and calls `RegisterPassBuilderCallbacks` to wire the passes in.

## 60.7 Preserving Analyses: Detailed Rules

Getting `PreservedAnalyses` right is important for correctness (avoiding stale cached data) and performance (avoiding unnecessary recomputation).

### 60.7.1 What to Preserve

| Pass modifies | Do NOT preserve |
|---------------|-----------------|
| CFG (adds/removes edges, adds/removes BBs) | `CFGAnalyses` (DT, PDT, LI, DomTree-dependent) |
| Call instructions | `CallGraphAnalysis` |
| Memory accesses | `MemorySSAAnalysis`, `AliasAnalysis` |
| Value definitions | `ScalarEvolutionAnalysis`, `DemandedBitsAnalysis` |
| IR instructions | `BranchProbabilityAnalysis`, `BlockFrequencyAnalysis` |
| Nothing | `PreservedAnalyses::all()` |

```cpp
// Pattern: CFG-preserving transform
PreservedAnalyses PA;
PA.preserveSet<CFGAnalyses>();
return PA;

// Pattern: pure value replacement (no CFG change, no memory effect)
PreservedAnalyses PA;
PA.preserveSet<CFGAnalyses>();
PA.preserve<MemorySSAAnalysis>();  // if memory effects unchanged
return PA;
```

### 60.7.2 Analysis Updaters

Some passes that modify the CFG can update existing analysis results incrementally, avoiding full recomputation. The `DominatorTree::insertEdge`/`deleteEdge` API and the `LoopInfo` updater allow a pass to keep analyses live after CFG changes:

```cpp
// From within a pass that splits an edge
auto &DT = AM.getResult<DominatorTreeAnalysis>(F);
auto &LI = AM.getResult<LoopAnalysis>(F);

// Split edge from PredBB to SucBB at NewBB
SplitEdge(PredBB, SucBB, &DT, &LI);
// SplitEdge updates DT and LI incrementally

// Since DT and LI were updated in place, they're still valid
PreservedAnalyses PA;
PA.preserve<DominatorTreeAnalysis>();
PA.preserve<LoopAnalysis>();
PA.preserveSet<CFGAnalyses>(); // LI + DT preserved through updater
return PA;
```

When a dominator tree or loop info is updated through the updater API, the analysis result in the cache is still valid — the preserved set must reflect this to avoid discarding and recomputing.

## 60.8 Debugging a Pass

### 60.8.1 Print-Before/After

```bash
# Print IR before and after each function pass
opt -passes='instr-count,instcombine' \
    -print-before=instcombine -print-after=instcombine \
    -S input.ll 2>&1 | head -40

# Or print all
opt -passes='instcombine' -print-before-all -print-after-all \
    -S input.ll 2>&1
```

### 60.8.2 Opt-Bisect

When a wrong-code bug is suspected in one pass in a large pipeline, `opt-bisect-limit` binary-searches which pass introduces the problem:

```bash
# First, find the total number of passes
opt -passes='default<O3>' -opt-bisect-limit=-1 -S input.ll

# Then binary search: if N passes produces wrong output
opt -passes='default<O3>' -opt-bisect-limit=N -S input.ll | compare_output
```

### 60.8.3 `verifyFunction` After Each Pass

During development, add verification:

```cpp
// In your pass after making changes
if (llvm::verifyFunction(F, &llvm::errs())) {
  llvm::report_fatal_error("Pass introduced invalid IR");
}
```

Or enable globally:
```bash
opt -passes='instr-count' -verify-each -S input.ll
```

`-verify-each` inserts a `VerifierPass` between every pass in the pipeline, catching the exact pass that breaks IR invariants.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **Pass plugin ABI stabilisation**: The LLVM community has ongoing RFC discussion ([discourse.llvm.org](https://discourse.llvm.org)) around versioning `LLVM_PLUGIN_API_VERSION` more rigorously so that out-of-tree plugins survive minor LLVM releases without recompilation; watch for an in-tree compatibility layer targeting the 22.x → 23.x transition.
- **`PassInstrumentation` hooks expansion**: Patches in review add pre/post-pass JSON telemetry callbacks to `PassInstrumentationCallbacks`, enabling external tools (IDEs, CI pipelines) to record per-pass timing and IR size deltas without modifying the plugin itself.
- **`LoopNestPass` interface stabilisation**: The `LoopNestPass` concept (passes that receive an entire loop nest rather than individual `Loop` objects) is being hardened in LLVM 23; passes implementing polyhedral-style tiling will migrate from `LoopPass` to `LoopNestPass` once the `LPMUpdater` contract for nest restructuring is finalised.
- **`CGSCCUpdateResult` devirtualisation**: An active refactor removes the virtual dispatch inside `CGSCCUpdateResult::RefinedFunctions` propagation; pass authors who call `UR.revisitCurrentSCC()` will see changed semantics when the slab-allocated SCC list is replaced with a flat `SmallVector`.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Typed analysis keys and compile-time pass graph verification**: Proposals on LLVM discourse aim to replace the `AnalysisKey` singleton-address trick with a `constexpr`-tagged type system, enabling the compiler itself to detect analysis dependency cycles and missing preservation declarations at build time rather than at run time.
- **Sandboxed pass execution for fuzzing**: Building on the `SandboxIR` infrastructure (introduced in LLVM 19), a dedicated `SandboxPassManager` is planned that checkpoints IR state before each pass, enabling rollback and differential IR mutation testing; pass authors will be able to fuzz-test their `PreservedAnalyses` declarations automatically.
- **`PassBuilder` pipeline serialisation / deserialisation**: An RFC proposes a machine-readable pipeline description format (extending the current text pipeline syntax) that captures not just pass names but per-pass configuration options and extension-point hooks, enabling reproducible pipeline snapshots for release engineering and A/B optimisation studies.
- **`MemorySSA`-aware `LPMUpdater` for loop transformations**: Current loop passes that modify memory accesses must manually invalidate `MemorySSAAnalysis`; a proposed `MemorySSAUpdater`-integrated `LPMUpdater` API would allow loop passes to perform incremental MemSSA updates identically to how `SplitEdge` handles `DominatorTree`, reducing redundant recomputation in pipelines containing multiple loop transformation passes.

### 5-Year Horizon (Long-Term, by ~2031)

- **Declarative pass specification language**: Research prototypes (e.g., work from the Compilers for Machine Learning workshop) explore specifying pass behaviour — including analysis requirements, IR pre/postconditions, and transformation invariants — in a Datalog-like or SMT-constraint DSL, with the LLVM pass infrastructure automatically generating C++ skeleton code and verifying correctness via Alive2-style bounded model checking.
- **Heterogeneous pass scheduling across LLVM and MLIR**: As ClangIR and the LLVM/MLIR convergence matures, a unified pass scheduling layer that can interleave LLVM `PassManager` passes with MLIR `OpPassManager` passes on the same IR unit is expected; pass authors will write a single `run()` method against a common `IRUnit` concept, with the scheduler selecting the appropriate representation at each stage.
- **AI-guided pass ordering and configuration**: Reinforcement-learning-based pass sequence optimisers (successors of ML-driven inlining heuristics already in LLVM) are projected to offer a `PassBuilder::setMLPipelineAdvisor()` API; individual pass authors will annotate their passes with `mlir::PassEffect` metadata to allow the adviser to reason about interaction costs without needing to recompile the pass.

---

## Chapter Summary

- Function passes inherit `PassInfoMixin<T>` and implement `run(Function&, FunctionAnalysisManager&) -> PreservedAnalyses`.
- Loop passes have signature `run(Loop&, LoopAnalysisManager&, LoopStandardAnalysisResults&, LPMUpdater&)` and are wrapped by `createFunctionToLoopPassAdaptor`.
- Module passes operate on the whole `Module`; CGSCC passes receive a `LazyCallGraph::SCC` in bottom-up order.
- Out-of-tree plugins export `llvmGetPassPluginInfo()` returning a `PassPluginLibraryInfo` struct; loaded with `opt -load-pass-plugin=./plugin.so`.
- Analysis requests via `AM.getResult<AnalysisT>(IR)` are cached; returning `PreservedAnalyses::all()` from a read-only pass avoids invalidation.
- CFG-modifying passes should `preserveSet<CFGAnalyses>()` only if they also update `DominatorTree`/`LoopInfo` incrementally; otherwise omit those from the preserved set.
- `opt -verify-each` and `opt-bisect-limit` are the primary tools for debugging wrong-code bugs introduced by a pass.


---

@copyright jreuben11
