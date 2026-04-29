# Chapter 59 — The New Pass Manager

*Part X — Analysis and the Middle-End*

LLVM's pass infrastructure evolved through two generations. The legacy pass manager (LPM) used virtual inheritance, explicit pass IDs, and a global pass registry that made composing passes complex and limited scheduling flexibility. The new pass manager (NPM), enabled by default since LLVM 13, replaces it with a value-semantics design: passes and analyses are plain C++ types with `run()` methods, and the manager owns them by value, enabling aggressive inlining and eliminating virtual dispatch on the hot path. This chapter explains the NPM's design, the `AnalysisManager` invalidation protocol, pipeline textual specification, instrumentation, and the `default<O3>` pipeline composition.

## 59.1 Design Philosophy

The NPM is built on one key insight from Sean Parent's "Inheritance Is The Base Class of Evil" talk: polymorphism through type erasure rather than inheritance. A pass is anything with a `run(IRUnit&, AnalysisManager&)` method. The `PassManager` type-erases it via a template:

```cpp
// From llvm/include/llvm/IR/PassManager.h
template <typename IRUnitT, typename... ExtraArgTs>
class PassManager {
  // Type-erased list of passes
  std::vector<std::unique_ptr<PassConceptT>> passes;

public:
  template <typename PassT>
  void addPass(PassT &&Pass) {
    passes.emplace_back(new PassModelT<PassT>(std::move(Pass)));
  }

  PreservedAnalyses run(IRUnitT &IR, AnalysisManager<IRUnitT> &AM,
                        ExtraArgTs... ExtraArgs);
};
```

`PassConceptT` is a pure-virtual base providing `run()` and `name()`. `PassModelT<PassT>` is the concrete template instantiation that holds an instance of `PassT` by value. The result is that passes are stack-or-heap-allocated concrete types — no `new` per pass instance at the use site, no virtual dispatch on data accesses.

### 59.1.1 IR Units

The NPM is parametric over the IR unit it operates on:

| IR unit | Pass manager type | Analysis manager type |
|---------|------------------|-----------------------|
| `Module` | `ModulePassManager` | `ModuleAnalysisManager` |
| `Function` | `FunctionPassManager` | `FunctionAnalysisManager` |
| `Loop` | `LoopPassManager` | `LoopAnalysisManager` |
| CGSCC (call-graph SCC) | `CGSCCPassManager` | `CGSCCAnalysisManager` |
| `MachineFunction` | `MachineFunctionPassManager` | `MachineFunctionAnalysisManager` |

The modularity allows composing: `ModulePassManager` can contain a `ModuleToFunctionPassAdaptor` that wraps a `FunctionPassManager` and runs it over each function in the module.

## 59.2 Analysis Passes

### 59.2.1 The `AnalysisInfoMixin`

An analysis pass inherits `AnalysisInfoMixin<DerivedT>` and provides:

1. A static `AnalysisKey Key` member (the unique ID).
2. A `Result` type (what the analysis computes and caches).
3. A `run(IRUnit &, AnalysisManager &)` method returning `Result`.

```cpp
class DominatorTreeAnalysis : public AnalysisInfoMixin<DominatorTreeAnalysis> {
  friend AnalysisInfoMixin<DominatorTreeAnalysis>;
  static AnalysisKey Key;  // defined in .cpp; address is the unique ID

public:
  using Result = DominatorTree;
  DominatorTree run(Function &F, FunctionAnalysisManager &AM);
};
```

The analysis result is cached by the `FunctionAnalysisManager`. The first call to `AM.getResult<DominatorTreeAnalysis>(F)` computes the result; subsequent calls return the cached copy.

### 59.2.2 Requesting Analyses

From within a pass:

```cpp
PreservedAnalyses MyPass::run(Function &F, FunctionAnalysisManager &AM) {
  auto &DT  = AM.getResult<DominatorTreeAnalysis>(F);
  auto &LI  = AM.getResult<LoopAnalysis>(F);
  auto &SE  = AM.getResult<ScalarEvolutionAnalysis>(F);
  // … use DT, LI, SE …
  return PreservedAnalyses::all(); // nothing invalidated
}
```

`getResult<T>` returns a `T::Result&` for the given IR unit. If the result is not cached, `T::run` is called. If `T` requires another analysis, it recursively calls `getResult` on the nested manager.

### 59.2.3 `PreservedAnalyses`

A transform pass returns `PreservedAnalyses` to declare which cached results remain valid after its transformation:

```cpp
// Nothing preserved — all caches invalidated
return PreservedAnalyses::none();

// All preserved — pass is a pure analysis (no IR modification)
return PreservedAnalyses::all();

// Specific analyses preserved
PreservedAnalyses PA;
PA.preserve<DominatorTreeAnalysis>();
PA.preserve<LoopAnalysis>();
return PA;

// Preserve all analyses in a set
PA.preserveSet<CFGAnalyses>(); // DT, PDT, LI, and other CFG-dependent analyses
```

The `AnalysisManager` checks the returned `PreservedAnalyses` after each pass and invalidates cached results not in the preserved set. This is the NPM's replacement for the legacy "add pass dependency" mechanism.

### 59.2.4 Analysis Invalidation

When an analysis is invalidated, its cached `Result` is destroyed. The `Result` type can implement `invalidate()` to release resources eagerly:

```cpp
struct DominatorTree {
  // …
  bool invalidate(Function &F, const PreservedAnalyses &PA,
                  FunctionAnalysisManager::Invalidator &Inv) {
    // Invalidate if CFG analyses are not preserved
    auto PAC = PA.getChecker<DominatorTreeAnalysis>();
    return !PAC.preserved() && !PAC.preservedSet<CFGAnalyses>();
  }
};
```

If `invalidate()` returns `true`, the result is discarded. Passing analyses to `Invalidator::invalidate<AnalysisT>()` within the method allows cascading invalidation.

## 59.3 Transform Passes

A transform pass also uses `PassInfoMixin` (not `AnalysisInfoMixin`) and has no `Key`. It returns `PreservedAnalyses`:

```cpp
class InstCombinePass : public PassInfoMixin<InstCombinePass> {
  unsigned MaxIterations;
public:
  InstCombinePass(unsigned MaxIterations = 1) : MaxIterations(MaxIterations) {}
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM);
  static bool isRequired() { return false; }
};
```

`isRequired()` returning `false` means the pass may be skipped if the function has the `optnone` attribute. Passes like `AlwaysInlinerPass` override this to `true`.

The `run()` method applies the transformation and returns the set of preserved analyses. Typically, function-level transforms preserve `DominatorTreeAnalysis` if they do not modify the CFG:

```cpp
PreservedAnalyses InstCombinePass::run(Function &F, FunctionAnalysisManager &AM) {
  bool changed = runInstCombine(F);
  if (!changed) return PreservedAnalyses::all();

  PreservedAnalyses PA;
  PA.preserveSet<CFGAnalyses>(); // CFG structure unchanged
  return PA;
}
```

## 59.4 Pass Adaptors

The NPM uses adaptors to bridge IR unit levels:

```cpp
// Run a FunctionPassManager over each Function in a Module
ModulePassManager MPM;
FunctionPassManager FPM;
FPM.addPass(InstCombinePass());
FPM.addPass(SimplifyCFGPass());
MPM.addPass(createModuleToFunctionPassAdaptor(std::move(FPM)));
```

Available adaptors:

| Adaptor | Purpose |
|---------|---------|
| `ModuleToFunctionPassAdaptor` | Run FPM over each Function in Module |
| `FunctionToLoopPassAdaptor` | Run LPM over each Loop in Function |
| `ModuleToPostOrderCGSCCPassAdaptor` | Run CGSCC passes over bottom-up call graph |
| `CGSCCToFunctionPassAdaptor` | Run FPM over each Function in a CGSCC |

The adaptors handle the mismatch between IR unit levels — e.g., `ModuleToFunctionPassAdaptor` iterates over functions, invokes the FPM, and merges the `PreservedAnalyses` from each invocation.

## 59.5 `PassBuilder` and Pipeline Construction

`PassBuilder` (declared in
[`llvm/include/llvm/Passes/PassBuilder.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Passes/PassBuilder.h)) is the factory for standard LLVM pass pipelines. It ties together the four analysis managers and provides registration methods:

```cpp
llvm::TargetMachine *TM = /* from target selection */;
llvm::PipelineTuningOptions PTO;
llvm::PassBuilder PB(TM, PTO);

// Register all standard analyses with their respective managers
llvm::LoopAnalysisManager     LAM;
llvm::FunctionAnalysisManager FAM;
llvm::CGSCCAnalysisManager    CGAM;
llvm::ModuleAnalysisManager   MAM;

PB.registerModuleAnalyses(MAM);
PB.registerCGSCCAnalyses(CGAM);
PB.registerFunctionAnalyses(FAM);
PB.registerLoopAnalyses(LAM);
PB.crossRegisterProxies(LAM, FAM, CGAM, MAM);
```

`crossRegisterProxies` installs proxy analyses that allow a manager at one IR-unit level to query analyses at an adjacent level. For example, `FunctionAnalysisManager` gets a `ModuleAnalysisManagerFunctionProxy` that delegates to `MAM` for module-level analyses.

### 59.5.1 Building Standard Pipelines

```cpp
// -O3 pipeline (equivalent to clang -O3)
llvm::ModulePassManager MPM =
  PB.buildPerModuleDefaultPipeline(llvm::OptimizationLevel::O3);

// -O2 pipeline
MPM = PB.buildPerModuleDefaultPipeline(llvm::OptimizationLevel::O2);

// -O0 (minimal passes — no optimization)
MPM = PB.buildO0DefaultPipeline(llvm::OptimizationLevel::O0,
                                  /*LTOPreLink=*/false);

// ThinLTO pre-link pipeline
MPM = PB.buildThinLTOPreLinkDefaultPipeline(llvm::OptimizationLevel::O2);

// ThinLTO post-link (link-time) pipeline
MPM = PB.buildThinLTODefaultPipeline(llvm::OptimizationLevel::O2,
                                       /*ImportSummary=*/nullptr);

// Running the pipeline
MPM.run(*Mod, MAM);
```

`OptimizationLevel` is a struct carrying separate optimization and size levels:

```cpp
namespace llvm {
class OptimizationLevel {
public:
  static const OptimizationLevel O0; // No optimization
  static const OptimizationLevel O1; // Optimize without size increase
  static const OptimizationLevel O2; // Moderate optimization
  static const OptimizationLevel O3; // Aggressive optimization
  static const OptimizationLevel Os; // Optimize for code size
  static const OptimizationLevel Oz; // Minimize code size aggressively
};
}
```

### 59.5.2 Textual Pipeline Specification

`PassBuilder::parsePassPipeline` parses a textual pipeline description into a `ModulePassManager`:

```cpp
llvm::ModulePassManager MPM;
if (auto Err = PB.parsePassPipeline(MPM, "default<O3>")) {
  llvm::handleAllErrors(std::move(Err), [](const llvm::StringError &E) {
    llvm::errs() << "Pipeline parse error: " << E.message() << "\n";
  });
}
```

The pipeline text format:
```
default<O3>
function<eager-inv>(instcombine,simplifycfg)
module(inline,function(mem2reg,gvn))
cgscc(inline)
loop(licm<allowspeculation>)
```

Pipeline elements are:
- A pass name (looks up the pass by registered name)
- A pass name with angle-bracket options: `instcombine<max-iterations=2>`
- A nested pipeline: `function(pass1,pass2)` runs `pass1` then `pass2` over each function
- `default<O2>`: expands to the full O2 standard pipeline

The `opt` tool uses this format with `-passes=`:
```bash
opt -passes='function(mem2reg,instcombine,simplifycfg)' -S input.ll -o output.ll
opt -passes='default<O3>' -S input.ll -o output.ll
```

## 59.6 Pass Instrumentation

The `PassInstrumentationCallbacks` system allows hooking into the pass execution lifecycle. It is used for:
- `print-after-all` / `print-before-all`: dump IR before/after each pass
- `-time-passes`: timing each pass
- `opt-bisect`: binary-search which pass introduces a bug
- Custom debugging hooks

```cpp
llvm::PassInstrumentationCallbacks PIC;

// Hook called before each pass
PIC.registerBeforeNonSkippedPassCallback(
    [](StringRef PassName, llvm::Any IR) {
      llvm::outs() << "Before pass: " << PassName << "\n";
    });

// Hook called after each pass (with PreservedAnalyses)
PIC.registerAfterPassCallback(
    [](StringRef PassName, llvm::Any IR, const PreservedAnalyses &PA) {
      llvm::outs() << "After pass: " << PassName << "\n";
    });

// Hook to decide whether to skip a pass (opt-bisect)
PIC.registerShouldRunOptionalPassCallback(
    [](StringRef PassName, llvm::Any IR) -> bool {
      // Return false to skip this pass
      return true;
    });

// Register PIC with PassBuilder
llvm::PassBuilder PB(TM, PTO, std::nullopt, &PIC);
```

`StandardInstrumentations` (in
[`llvm/include/llvm/Passes/StandardInstrumentations.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Passes/StandardInstrumentations.h)) registers all the built-in instrumentation callbacks:

```cpp
llvm::StandardInstrumentations SI(Mod->getContext(), /*DebugLogging=*/true);
SI.registerCallbacks(PIC, &MAM);
```

The `opt` tool installs `StandardInstrumentations` by default, enabling `-print-before-all`, `-print-after-all`, `-time-passes`, and similar flags.

## 59.7 The Default<O3> Pipeline Internals

The `default<O3>` pipeline represents the production optimization configuration. Its high-level structure (from `PassBuilder::buildPerModuleDefaultPipeline`):

```
ModulePassManager {
  // Phase 1: Pre-inlining canonicalization
  ModuleToFunctionPassAdaptor {
    FunctionPassManager {
      EntryExitInstrumenter
      SimplifyCFGPass
      SROAPass
      EarlyCSEPass
      // ...
    }
  }
  
  // Phase 2: Inlining (CGSCC passes)
  ModuleToPostOrderCGSCCPassAdaptor {
    CGSCCPassManager {
      InlinerPass  // threshold-based inliner
      PostOrderFunctionAttrsPass
      // ...
    }
  }
  
  // Phase 3: Post-inlining scalar optimization
  ModuleToFunctionPassAdaptor {
    FunctionPassManager {
      SROAPass
      EarlyCSEPass<memssa>
      CorrelatedValuePropagationPass
      InstCombinePass
      AggressiveInstCombinePass
      LibCallsShrinkWrapPass
      // ...
    }
  }
  
  // Phase 4: Loop optimization
  ModuleToFunctionPassAdaptor {
    FunctionPassManager {
      FunctionToLoopPassAdaptor {
        LoopPassManager {
          LoopRotatePass
          LICMPass
          LoopFullUnrollPass
          // ...
        }
      }
      InstCombinePass
      GVNPass
      // ...
    }
  }

  // Phase 5: Vectorization
  ModuleToFunctionPassAdaptor {
    FunctionPassManager {
      LoopVectorizePass
      SLPVectorizerPass
      // ...
    }
  }

  // Phase 6: Global cleanup
  GlobalDCEPass
  DeadArgumentEliminationPass
  // ...
}
```

The exact composition evolves between LLVM releases. To see the current O3 pipeline for a given target, run:

```bash
opt -passes='default<O3>' -print-pipeline-passes -S /dev/null 2>&1
```

Or from within code:

```cpp
MPM.printPipeline(llvm::outs(), [&](StringRef cls) -> StringRef {
  return PB.getPassNameForClassName(cls).value_or(cls);
});
```

## 59.8 The Legacy Pass Manager (LPM) Compatibility

The legacy pass manager is retained for backend passes (SelectionDAG, register allocation) that have not yet been ported. The `opt` tool supports both:

```bash
# New PM (default)
opt -passes='instcombine' -S input.ll

# Legacy PM (deprecated, for backend use)
opt -enable-new-pm=0 -instcombine -S input.ll
```

LLVM provides `createModuleToFunctionPassAdaptor(createLegacyPMPass<LegacyPassT>())` as a bridge, but new passes should always be written for the NPM.

---

## Chapter Summary

- The new pass manager (NPM) uses concept-based polymorphism: passes are value types with `run(IRUnit&, AnalysisManager&)` methods; no virtual base class required.
- IR units are `Module`, `Function`, `Loop`, and CGSCC; each has its own `PassManager<U>` and `AnalysisManager<U>`.
- Analysis passes inherit `AnalysisInfoMixin<T>` and provide a unique `AnalysisKey Key`, a `Result` type, and a `run()` method. Results are cached and invalidated via `PreservedAnalyses`.
- Transform passes inherit `PassInfoMixin<T>` and return `PreservedAnalyses` from `run()`, declaring which cached analysis results survive the transformation.
- `PassBuilder` constructs standard pipelines (`buildPerModuleDefaultPipeline`) and parses textual pipeline descriptions (`parsePassPipeline`).
- The textual format `default<O3>`, `function(instcombine,simplifycfg)`, and `loop(licm)` is used by `opt -passes=` and by `PassBuilder::parsePassPipeline` in library use.
- `PassInstrumentationCallbacks` + `StandardInstrumentations` provide print-before/after, timing, and bisection hooks.
- The default O3 pipeline has five major phases: pre-inlining canonicalization, CGSCC inlining, post-inlining scalar opts, loop opts, and vectorization.
