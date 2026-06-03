# Chapter 45 — The Static Analyzer

*Part VII — Clang as a Multi-Language Compiler*

Clang's static analyzer transforms the compiler from a code translator into a reasoning engine. Where ordinary compilation operates on individual statements in isolation, the analyzer threads symbolic state through every feasible path in the control-flow graph, tracking constraints on values across loop iterations, function calls, and heap allocations simultaneously. The result is a class of bug detection — null dereferences along specific call chains, heap use-after-free on particular error paths, uninitialized reads under precise preconditions — that no purely syntactic lint tool can approach. This chapter dissects every layer of the analyzer infrastructure, from the frontend action that initiates analysis down to the Z3 SMT solver that refutes infeasible paths, with enough implementation detail to build a production-quality custom checker.

## Table of Contents

- [45.1 Architecture Overview](#451-architecture-overview)
  - [45.1.1 Invocation and the Frontend Pipeline](#4511-invocation-and-the-frontend-pipeline)
  - [45.1.2 `AnalysisManager` and the Engine Hierarchy](#4512-analysismanager-and-the-engine-hierarchy)
  - [45.1.3 Directory Layout](#4513-directory-layout)
  - [45.1.4 Analysis Ordering and the Call Graph](#4514-analysis-ordering-and-the-call-graph)
  - [45.1.5 `CheckerManager` Dispatch](#4515-checkermanager-dispatch)
- [45.2 Symbolic Execution](#452-symbolic-execution)
  - [45.2.1 `ProgramState` — The Immutable State Object](#4521-programstate-the-immutable-state-object)
  - [45.2.2 `SymbolManager` and Symbolic Values](#4522-symbolmanager-and-symbolic-values)
  - [45.2.3 The `SVal` Hierarchy](#4523-the-sval-hierarchy)
  - [45.2.4 `ConstraintManager` — Feasibility Checking](#4524-constraintmanager-feasibility-checking)
  - [45.2.5 `SValBuilder` — Constructing Symbolic Values](#4525-svalbuilder-constructing-symbolic-values)
- [45.3 The Memory Model](#453-the-memory-model)
  - [45.3.1 `MemRegion` Hierarchy](#4531-memregion-hierarchy)
  - [45.3.2 `RegionStoreManager` — Bindings and Lookups](#4532-regionstoremanager-bindings-and-lookups)
  - [45.3.3 Escape and Invalidation](#4533-escape-and-invalidation)
  - [45.3.4 Typed Access and Reinterpretation](#4534-typed-access-and-reinterpretation)
- [45.4 The ExplodedGraph and Path Explosion](#454-the-explodedgraph-and-path-explosion)
  - [45.4.1 `ExplodedNode` = (ProgramPoint, ProgramState)](#4541-explodednode-programpoint-programstate)
  - [45.4.2 `CoreEngine::ExecuteWorkList()`](#4542-coreengineexecuteworklist)
  - [45.4.3 Graph Pruning and Caching](#4543-graph-pruning-and-caching)
- [45.5 `CheckerManager` and the Checker API](#455-checkermanager-and-the-checker-api)
  - [45.5.1 CRTP Registration](#4551-crtp-registration)
  - [45.5.2 Check Families](#4552-check-families)
  - [45.5.3 `CheckerContext` API](#4553-checkercontext-api)
- [45.6 Built-in Checkers](#456-built-in-checkers)
  - [45.6.1 `core.NullDereference`](#4561-corenulldereference)
  - [45.6.2 `core.uninitialized.*`](#4562-coreuninitialized)
  - [45.6.3 `cplusplus.NewDelete`](#4563-cplusplusnewdelete)
  - [45.6.4 `core.DivideZero`](#4564-coredividezero)
  - [45.6.5 `deadcode.DeadStores`](#4565-deadcodedeadstores)
  - [45.6.6 `security.insecureAPI.*`](#4566-securityinsecureapi)
  - [45.6.7 `alpha.cplusplus.STLAlgorithmModeling`](#4567-alphacplusplusstlalgorithmmodeling)
  - [45.6.8 `alpha.webkit.UncountedLambdaCapturesChecker`](#4568-alphawebkituncountedlambdacaptureschecker)
- [45.7 Writing a Custom Checker](#457-writing-a-custom-checker)
  - [45.7.1 Step-by-Step Implementation](#4571-step-by-step-implementation)
  - [45.7.2 Registration via `Checkers.td`](#4572-registration-via-checkerstd)
  - [45.7.3 `REGISTER_MAP_WITH_PROGRAMSTATE` Internals](#4573-registermapwithprogramstate-internals)
- [45.8 `CallEvent` and Inter-Procedural Analysis](#458-callevent-and-inter-procedural-analysis)
  - [45.8.1 `CallEvent` Hierarchy](#4581-callevent-hierarchy)
  - [45.8.2 Inlining Mechanism](#4582-inlining-mechanism)
  - [45.8.3 Objective-C Retain Count Checking](#4583-objective-c-retain-count-checking)
- [45.9 Bug Reports and Path Diagnostics](#459-bug-reports-and-path-diagnostics)
  - [45.9.1 `PathSensitiveBugReport` and `BugType`](#4591-pathsensitivebugreport-and-bugtype)
  - [45.9.2 `BugReporterVisitor` — Adding Path Notes](#4592-bugreportervisitor-adding-path-notes)
  - [45.9.2b Path Reconstruction — `BugPathGetter`](#4592b-path-reconstruction-bugpathgetter)
  - [45.9.3 Diagnostic Output Formats](#4593-diagnostic-output-formats)
- [45.10 Taint Analysis](#4510-taint-analysis)
  - [45.10.1 `taint::` API](#45101-taint-api)
  - [45.10.2 Sources, Propagators, and Sinks](#45102-sources-propagators-and-sinks)
  - [45.10.3 Custom Taint Configuration via YAML](#45103-custom-taint-configuration-via-yaml)
  - [45.10.4 Taint and Partial Regions](#45104-taint-and-partial-regions)
- [45.11 Z3 Constraint Solver Integration](#4511-z3-constraint-solver-integration)
  - [45.11.1 `Z3ConstraintManager`](#45111-z3constraintmanager)
  - [45.11.2 Refutation Mode vs. Full Solving](#45112-refutation-mode-vs-full-solving)
  - [45.11.3 When Z3 Catches What Range Solving Misses](#45113-when-z3-catches-what-range-solving-misses)
- [45.12 Performance and Scalability](#4512-performance-and-scalability)
  - [45.12.1 `FunctionSummariesTy` and Result Caching](#45121-functionsummariesty-and-result-caching)
  - [45.12.2 Key Tuning Options](#45122-key-tuning-options)
  - [45.12.3 Worklist Strategies and Their Trade-offs](#45123-worklist-strategies-and-their-trade-offs)
  - [45.12.4 `clang --analyze` vs. `clang-tidy`](#45124-clang-analyze-vs-clang-tidy)
  - [45.12.5 Scaling to Large Codebases](#45125-scaling-to-large-codebases)
- [45.13 Clang Lifetime Safety Analysis](#4513-clang-lifetime-safety-analysis)
  - [45.13.1 History and Production Graduation](#45131-history-and-production-graduation)
  - [45.13.2 Five Production Checkers](#45132-five-production-checkers)
  - [45.13.3 `[[clang::lifetimebound]]` Mechanics](#45133-clanglifetimebound-mechanics)
  - [45.13.4 Enabling Lifetime Warnings](#45134-enabling-lifetime-warnings)
  - [45.13.5 Comparison with Rust's Borrow Checker](#45135-comparison-with-rusts-borrow-checker)
- [45.14 Clang FlowSensitive Dataflow Analysis Framework](#4514-clang-flowsensitive-dataflow-analysis-framework)
  - [45.14.1 Architectural Distinction from the Path-Sensitive Engine](#45141-architectural-distinction-from-the-path-sensitive-engine)
  - [45.14.2 Core Abstractions](#45142-core-abstractions)
  - [45.14.3 Lattice Requirements and Widening](#45143-lattice-requirements-and-widening)
  - [45.14.4 SAT Solver Integration](#45144-sat-solver-integration)
  - [45.14.5 Built-in Analysis: `UncheckedOptionalAccessModel`](#45145-built-in-analysis-uncheckedoptionalaccessmodel)
  - [45.14.6 Writing a Custom FlowSensitive Analysis](#45146-writing-a-custom-flowsensitive-analysis)
  - [45.14.7 When to Choose FlowSensitive vs. Path-Sensitive](#45147-when-to-choose-flowsensitive-vs-path-sensitive)
- [Chapter 45 Summary](#chapter-45-summary)
- [Cross-References](#cross-references)
- [Reference Links](#reference-links)

---

## 45.1 Architecture Overview

### 45.1.1 Invocation and the Frontend Pipeline

The analyzer activates via two equivalent spellings:

```bash
# High-level driver
clang --analyze -Xanalyzer -analyzer-checker=core,cplusplus foo.cpp

# cc1 direct invocation
clang -cc1 -analyze -analyzer-checker=core.NullDereference foo.cpp
```

The driver translates `--analyze` into `-cc1 -analyze` and passes `-analyzer-checker` flags through. Inside the compilation pipeline, `ParseAnalyzerArgs()` in
[`clang/lib/Frontend/CompilerInvocation.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Frontend/CompilerInvocation.cpp)
populates `AnalyzerOptions`, a ref-counted object carrying every tunable parameter.

The analyzer runs as a `FrontendAction`. `CreateAnalysisConsumer()` in
[`clang/lib/StaticAnalyzer/Frontend/AnalysisConsumer.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/StaticAnalyzer/Frontend/AnalysisConsumer.cpp)
returns a concrete `AnalysisASTConsumer` (the public interface in `AnalysisConsumer.h`) that wraps the full analysis infrastructure. When `HandleTranslationUnit()` fires, the consumer iterates every `Decl` with a body, orders them by call graph, and launches `AnalysisConsumer::HandleCode()` on each.

### 45.1.2 `AnalysisManager` and the Engine Hierarchy

```
AnalysisConsumer
  └─ AnalysisManager          (owns configuration, AnaCtxMgr, PathConsumers)
       └─ ExprEngine          (drives symbolic execution, owns CoreEngine)
            └─ CoreEngine     (CFG traversal + ExplodedGraph construction)
                 └─ WorkList  (DFS, BFS, or UnexploredFirst priority queue)
```

`AnalysisManager` (declared in
[`clang/include/clang/StaticAnalyzer/Core/PathSensitive/AnalysisManager.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/AnalysisManager.h))
holds the `ASTContext`, `Preprocessor`, the set of `PathDiagnosticConsumer`s (HTML/Plist/SARIF output), and factory pointers for the `StoreManager` and `ConstraintManager`. It inherits `BugReporterData` so that `BugReporter` can reach the preprocessor and path consumers without a circular dependency.

`ExprEngine` in
[`clang/lib/StaticAnalyzer/Core/ExprEngine.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/StaticAnalyzer/Core/ExprEngine.cpp)
implements `SubEngine`, the abstract interface `CoreEngine` calls to evaluate individual statements. When `CoreEngine` dequeues a worklist unit, it delegates to `ExprEngine::processCFGElement()`, which dispatches to statement-specific `visit*()` methods — `VisitCallExpr`, `VisitBinaryOperator`, `VisitUnaryOperator`, and so on.

### 45.1.3 Directory Layout

```
clang/lib/StaticAnalyzer/
├── Core/           ExprEngine, CoreEngine, ProgramState, MemRegion, SVals,
│                   SymbolManager, BugReporter, ConstraintManager, Store,
│                   RegionStore, SValBuilder, BasicValueFactory, BlockCounter
├── Checkers/       ~100 built-in checkers + GenericTaintChecker
│                   RetainCountChecker/, MPI/, WebKit/
└── Frontend/       AnalysisConsumer, FrontendActions, CheckerRegistry
```

The public headers mirror this under `clang/include/clang/StaticAnalyzer/{Core,Checkers,Frontend}/`. The `Core/PathSensitive/` subtree is the most important: it contains the type definitions for `ProgramState`, `SVal`, `MemRegion`, `ExplodedGraph`, `CoreEngine`, `CheckerContext`, `CallEvent`, and `ConstraintManager` — essentially the complete symbolic execution substrate.

### 45.1.4 Analysis Ordering and the Call Graph

Before analyzing any function, `AnalysisConsumer` builds a `CallGraph` over the translation unit using Clang's `CallGraphBuilder`. Functions are analyzed in **bottom-up** order: callees before callers. This ordering ensures that when a caller is analyzed, summaries for its callees may already be available in `FunctionSummariesTy`, reducing the need to inline the same callee repeatedly from multiple call sites.

The ordering is approximate: recursion (including mutual recursion) is broken by choosing an arbitrary entry point, which means recursive functions are analyzed without complete summaries of their recursive calls. The analyzer handles this by invalidating state at recursive call sites rather than attempting to find a fixed point, a pragmatic compromise that keeps analysis time bounded.

Top-level entry points are configurable. With `-analyzer-config ipa=inlining` (default), the analyzer inlines aggressively. With `-analyzer-config ipa=none`, each function is treated as an opaque black box — fast but imprecise. The `basic-inlining` mode inlines only small functions; `dynamic-bifurcate` additionally forks paths at virtual dispatch.

### 45.1.5 `CheckerManager` Dispatch

`CheckerManager` (
[`clang/include/clang/StaticAnalyzer/Core/CheckerManager.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/CheckerManager.h))
maintains typed callback lists, one per check family. When `ExprEngine` reaches a `CallExpr`, it calls:

```cpp
Mgr.runCheckersForPreStmt(Dst, Src, CE, *this);   // before evaluation
Mgr.runCheckersForPostStmt(Dst, Src, CE, *this);  // after evaluation
```

Each call iterates the registered `CheckerFn` objects and invokes them with the current `ExplodedNodeSet` and `CheckerContext`. Checkers that produce new nodes append them to the output set; checkers that only inspect state return without modification.

The `runCheckers*` family uses a **cross-product** strategy: if the input `ExplodedNodeSet` contains N nodes (because prior checkers forked state), each checker is applied to every input node, potentially multiplying the output set. The framework manages this automatically; checkers need not iterate over predecessor nodes themselves.

`CheckerManager` also handles **checker dependencies**. If checker B declares a dependency on checker A (via `Mgr.registerDependentChecker<B, A>()`), A is guaranteed to be initialized and registered before B regardless of command-line ordering. This is used by, for example, `cplusplus.NewDeleteLeaks` depending on `cplusplus.NewDelete` to share the allocation state map.

---

## 45.2 Symbolic Execution

### 45.2.1 `ProgramState` — The Immutable State Object

`ProgramState` (
[`clang/include/clang/StaticAnalyzer/Core/PathSensitive/ProgramState.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/ProgramState.h))
is the workhorse of symbolic execution. It is **immutable** — every operation returns a new `ProgramStateRef` (an `IntrusiveRefCntPtr<const ProgramState>`). The class stores three fields:

| Field | Type | Purpose |
|-------|------|---------|
| `Env` | `Environment` | Maps `Stmt *` → `SVal` for expression values |
| `store` | `Store` (opaque ptr) | Maps memory locations → `SVal` |
| `GDM` | `GenericDataMap` | `ImmutableMap<void*,void*>` for checker-private state |

Because states are interned in a `FoldingSet`, two states with identical content share the same heap allocation, giving copy-on-write semantics for free.

**`Environment`** tracks rvalue bindings: after evaluating `x + 3`, the `Environment` maps the `BinaryOperator*` AST node to the symbolic sum. `EnvironmentManager::getSVal()` retrieves existing bindings; `bindExpr()` produces a new `Environment` with an added mapping.

**`Store`** is the memory model, described in §45.3.

**`GenericDataMap`** is the extensibility hook. Checkers define private state types and store them under a unique static `void*` key obtained from a `ProgramStateTrait<T>` specialization.

### 45.2.2 `SymbolManager` and Symbolic Values

When the analyzer cannot determine a concrete value — a function parameter, a return value from an uninlined call, a heap allocation result — it creates a **symbol**. `SymbolManager` (
[`clang/include/clang/StaticAnalyzer/Core/PathSensitive/SymbolManager.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/SymbolManager.h))
interns `SymExpr` objects in a `FoldingSet`. The principal symbol types are:

| Class | Meaning |
|-------|---------|
| `SymbolRegionValue` | Initial value of a typed memory region |
| `SymbolConjured` | Result of an opaque expression (call return, cast) |
| `SymbolDerived` | Value derived from a parent symbol (field access) |
| `SymbolExtent` | Byte extent of a heap region |
| `SymbolMetadata` | Checker-attached metadata symbol |
| `SymIntExpr`, `IntSymExpr`, `SymSymExpr` | Arithmetic over symbols |

### 45.2.3 The `SVal` Hierarchy

`SVal` (
[`clang/include/clang/StaticAnalyzer/Core/PathSensitive/SVals.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/SVals.h))
is a discriminated-union value type, 16 bytes on 64-bit hosts, encoding both kind and payload as tagged pointers:

```
SVal
├── UndefinedVal          — uninitialized / unreachable
├── UnknownVal            — over-approximate (conservatively lost)
└── DefinedOrUnknownSVal
     └── DefinedSVal
          ├── NonLoc      — not a memory address
          │    ├── nonloc::SymbolVal      — wraps a SymExpr
          │    ├── nonloc::ConcreteInt    — compile-time integer (APSInt)
          │    ├── nonloc::LazyCompoundVal — deferred aggregate copy
          │    ├── nonloc::CompoundVal    — explicit aggregate initializer
          │    └── nonloc::PointerToMember
          └── Loc         — memory address
               ├── loc::MemRegionVal     — pointer to a MemRegion
               ├── loc::GotoLabel        — address of a label
               └── loc::ConcreteInt      — integer cast to pointer
```

`SVal::castAs<T>()` performs an asserting downcast; `SVal::getAs<T>()` returns `std::optional<T>`. The kind bits live in the low bits of the `SValKind` enum generated from `SVals.def`.

`UndefinedVal` represents reads from uninitialized storage; using it triggers `core.uninitialized.*` reports. `UnknownVal` is the safe over-approximation: when the analyzer cannot represent a value precisely (e.g., after a call that escapes all arguments), it binds `UnknownVal` rather than fabricating constraints.

### 45.2.4 `ConstraintManager` — Feasibility Checking

After every branch, `ExprEngine` calls `ConstraintManager::assume(State, Cond, true/false)` to add the branch condition to the state. `assume()` returns `nullptr` if the condition is infeasible — pruning that path from the graph — or a new state with the constraint recorded.

The default `RangeConstraintManager` (`clang/lib/StaticAnalyzer/Core/RangeConstraintManager.cpp`) represents each symbol's constraint as a sorted `RangeSet` — a set of disjoint integer ranges. Intersecting ranges during `assume()` is O(n log n) on the number of disjoint ranges, which is typically small for practical programs. The `ConstraintManager::assumeDual()` method calls `assume()` for both the true and false branches simultaneously; if one branch is infeasible, the other is adopted unconditionally without forking the path.

The optional `Z3ConstraintManager` encodes constraints as SMT formulae and calls the Z3 solver; see §45.11.

`ConstraintManager` also supports `ConstraintManager::canReasonAbout(SVal)`, which returns false for symbol types the manager cannot model precisely (e.g., floating-point symbols in the range-based manager). `ExprEngine` uses this to decide whether to bifurcate a branch or just take the conservative over-approximation (keeping both branches without a constraint).

### 45.2.5 `SValBuilder` — Constructing Symbolic Values

`SValBuilder` (
[`clang/include/clang/StaticAnalyzer/Core/PathSensitive/SValBuilder.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/SValBuilder.h))
is the factory for new `SVal` objects. `ExprEngine` calls it during expression evaluation:

```cpp
SVal LHS = State->getSVal(E->getLHS(), LCtx);
SVal RHS = State->getSVal(E->getRHS(), LCtx);
SVal Result = SVB.evalBinOp(State, BO_Add, LHS, RHS, ResultTy);
```

`evalBinOp()` performs constant folding when both operands are `ConcreteInt`, and builds a `SymSymExpr` when at least one is symbolic. `SValBuilder::evalCast()` handles pointer arithmetic, zero extension, sign extension, and truncation symbolically. The goal is to keep values as concrete as possible — simplifying `(3 + 4)` to `7` immediately — while degrading gracefully to symbolic expressions when concretization is impossible.

---

## 45.3 The Memory Model

### 45.3.1 `MemRegion` Hierarchy

`MemRegion` (
[`clang/include/clang/StaticAnalyzer/Core/PathSensitive/MemRegion.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/MemRegion.h))
models memory as a tree of named regions. Every pointer's `SVal` is a `loc::MemRegionVal` wrapping a `MemRegion*`. The full hierarchy:

```
MemRegion
├── MemSpaceRegion               (abstract top-level spaces)
│    ├── GlobalSystemSpaceRegion  (extern C system globals: errno, etc.)
│    ├── GlobalInternalSpaceRegion (static-linkage globals)
│    ├── GlobalImmutableSpaceRegion (const globals)
│    ├── StackLocalsSpaceRegion   (per-frame locals)
│    ├── StackArgumentsSpaceRegion (per-frame parameters)
│    └── HeapSpaceRegion          (malloc'd memory)
├── SymbolicRegion               (region with unknown base: *p for opaque p)
├── AllocaRegion                 (alloca() allocation)
├── StringRegion                 (string literal)
├── ObjCStringRegion
├── VarRegion                    (named variable)
│    ├── NonParamVarRegion
│    └── ParamVarRegion
├── FieldRegion                  (struct/class member)
├── ElementRegion                (array element, with symbolic index)
├── CXXBaseObjectRegion          (base class sub-object)
├── CXXDerivedObjectRegion
└── CXXTempObjectRegion          (C++ temporary)
```

Every region carries a parent region and a `MemSpaceRegion` root, enabling the analyzer to reason about aliasing: two `FieldRegion`s sharing the same `VarRegion` parent are provably distinct from each other and from any `HeapSpaceRegion` region.

### 45.3.2 `RegionStoreManager` — Bindings and Lookups

`RegionStoreManager` (the default `StoreManager` implementation in
[`clang/lib/StaticAnalyzer/Core/RegionStore.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/StaticAnalyzer/Core/RegionStore.cpp))
implements the `Store` as a persistent tree of `BindingKey → SVal` pairs, where `BindingKey` is `(MemRegion*, offset, Direct|Default)`. The `Direct` binding overrides the `Default` binding, enabling field-sensitive treatment of aggregates without copying the entire aggregate on every write.

```cpp
// Writing: Store = Mgr.Bind(Store, Loc, Val)
// Reading: SVal  = Mgr.getBinding(Store, Loc, T)
ProgramStateRef newState = State->bindLoc(loc, val, LCtx);
SVal val = State->getSVal(loc, type);
```

**`LazyCompoundVal`** defers aggregate copies. When a struct is passed by value, instead of immediately copying every field binding, the analyzer creates a `nonloc::LazyCompoundVal` that records `(Store, CXXTempObjectRegion*)`. Individual field reads resolve through the lazy binding on demand, coalescing when the region is eventually bound into a live store.

### 45.3.3 Escape and Invalidation

When a pointer escapes — passed to an uninlined function, stored into a global, cast to `void*` — `RegionStoreManager::invalidateRegions()` replaces all affected bindings with fresh `SymbolConjured` values. This prevents stale bindings from being misread as constraints on the escaped memory.

Invalidation respects the `RegionAndSymbolInvalidationTraits` structure: some regions are marked `IK_NoBindings` (they cannot be invalidated, e.g. const globals) or `IK_RegionValueSymbol` (invalidate the region's binding but preserve the symbol). Checkers can mark regions non-invalidatable via `checkRegionChanges()`, allowing them to prevent the engine from over-approximating across calls that have known effects on specific regions.

### 45.3.4 Typed Access and Reinterpretation

`RegionStoreManager` tracks bindings at the bit-offset level within a region. When code accesses memory at a type different from the binding type (e.g., reading a `uint32_t` field as two `uint16_t`s via a union), the store manager attempts to **reinterpret** the binding using `SValBuilder::dispatchCast()`. If the reinterpretation produces an ill-typed access that cannot be modeled, the result degrades to `UnknownVal`, preventing incorrect inferences while not silencing all analysis on the path.

---

## 45.4 The ExplodedGraph and Path Explosion

### 45.4.1 `ExplodedNode` = (ProgramPoint, ProgramState)

`ExplodedNode` (
[`clang/include/clang/StaticAnalyzer/Core/PathSensitive/ExplodedGraph.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/ExplodedGraph.h))
pairs a `ProgramPoint` (where in the CFG we are) with a `ProgramStateRef` (what is known about values at that point). It is the node type of the directed acyclic `ExplodedGraph`.

`ProgramPoint` types encountered during traversal:

| Type | Meaning |
|------|---------|
| `BlockEntrance` | First element of a CFG basic block |
| `PreStmt<T>` | Immediately before evaluating statement T |
| `PostStmt<T>` | Immediately after evaluating statement T |
| `BlockEdge` | Control transfer between two blocks |
| `CallEnter` | Entry into an inlined callee |
| `CallExitBegin` | Before callee return processing |
| `CallExitEnd` | After return value bound in caller |
| `LoopEntrance` | Entering a loop (for widening) |
| `EpsilonPoint` | Checker-generated transition (no CFG step) |

### 45.4.2 `CoreEngine::ExecuteWorkList()`

`CoreEngine` (
[`clang/include/clang/StaticAnalyzer/Core/PathSensitive/CoreEngine.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/CoreEngine.h))
drives the main analysis loop:

```cpp
bool CoreEngine::ExecuteWorkList(const LocationContext *L,
                                 unsigned MaxSteps,
                                 ProgramStateRef InitState) {
  // seed the initial node
  // loop:
  while (!WList->hasWork()) {
    WorkListUnit U = WList->dequeue();
    dispatchWorkItem(U);  // → ExprEngine::processCFGElement()
    if (G.num_nodes() > MaxSteps) break;
  }
}
```

`dispatchWorkItem()` delegates to `ExprEngine`, which evaluates the CFG element, produces successor `ExplodedNode`s via `NodeBuilder`, and enqueues them. The worklist is pluggable: `WorkList::makeDFS()` for depth-first (the default, favoring complete path exploration), `WorkList::makeBFS()`, or `WorkList::makeUnexploredFirst()` which prioritizes blocks not yet visited along any path (reducing redundant work on long functions).

### 45.4.3 Graph Pruning and Caching

`ExplodedGraph::addNode()` interns nodes in a `FoldingSet<ExplodedNode>` keyed on `(ProgramPoint, ProgramState)`. If an identical node already exists, the new edges are attached to the existing node — this is the **graph folding** that prevents exponential blowup on diamond-shaped CFGs. The `-analyzer-max-nodes` flag (default 200,000) caps the total node count; when exceeded, analysis of the current top-level function is abandoned and marked `RetryExhausted` in `FunctionSummariesTy`.

**Loop widening** (`LoopWidening.cpp`) detects when a loop head has been visited more than `analyzer-max-loop` times (default 4) and invalidates all non-const heap/global bindings, approximating the loop's effect without unrolling indefinitely.

**Graph trimming** is a post-analysis step: `ExplodedGraph::trim()` removes all nodes that are not ancestors of any error node. This drastically reduces the size of the graph passed to `BugReporter` for path reconstruction, making diagnostic output generation tractable even when the full graph contains hundreds of thousands of nodes. Trimming also removes nodes that are exclusively in unexplored branches, which reduces memory pressure when analyzing large functions.

The `BlockCounter` records how many times each `CFGBlock` has been visited along the current path (not globally across paths). `CoreEngine` passes the `BlockCounter` with each `WorkListUnit` so that the loop-widening decision can be made on a per-path basis rather than globally, enabling one path through a loop to be explored more deeply than another when the path conditions differ.

---

## 45.5 `CheckerManager` and the Checker API

### 45.5.1 CRTP Registration

Every checker inherits from `Checker<check::Event1, check::Event2, ...>`, a variadic CRTP base that registers callbacks during checker initialization:

```cpp
#include "clang/StaticAnalyzer/Core/Checker.h"
#include "clang/StaticAnalyzer/Core/PathSensitive/CheckerContext.h"

class MyChecker : public Checker<check::PreStmt<CallExpr>,
                                  check::DeadSymbols> {
public:
  void checkPreStmt(const CallExpr *CE, CheckerContext &C) const;
  void checkDeadSymbols(SymbolReaper &Reaper, CheckerContext &C) const;
};
```

During `CheckerManager` initialization, the variadic base calls `check::PreStmt<CallExpr>::_register(this, Mgr)`, which invokes `Mgr._registerForPreStmt(...)` with a type-erased `CheckerFn` wrapping `_checkStmt<MyChecker>`. The dispatch table is built once at startup.

### 45.5.2 Check Families

| Check class | Trigger |
|------------|---------|
| `check::PreStmt<T>` | Before evaluating AST statement of type `T` |
| `check::PostStmt<T>` | After evaluating AST statement of type `T` |
| `check::PreCall` | Before any call is evaluated |
| `check::PostCall` | After any call is evaluated |
| `check::Location` | On every memory read/write (dereference) |
| `check::Bind` | When a value is bound to a memory location |
| `check::DeadSymbols` | When symbols go out of scope (reaper pass) |
| `check::EndFunction` | At function exit |
| `check::BeginFunction` | At function entry |
| `check::BranchCondition` | Before evaluating a branch condition |
| `check::EndAnalysis` | After all paths through a function |
| `check::ASTDecl<T>` | On AST traversal (non-path-sensitive) |
| `check::ASTCodeBody` | On the entire body (non-path-sensitive) |

### 45.5.3 `CheckerContext` API

`CheckerContext` (
[`clang/include/clang/StaticAnalyzer/Core/PathSensitive/CheckerContext.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/CheckerContext.h))
is the window through which a checker interacts with the engine during a callback:

```cpp
// State access
ProgramStateRef State = C.getState();
const LocationContext *LCtx = C.getLocationContext();

// Transition generation: non-fatal error node
ExplodedNode *ErrNode = C.generateNonFatalErrorNode();

// Transition generation: sink (terminates this path)
ExplodedNode *Sink = C.generateSink(State, C.getPredecessor());

// Normal transition with modified state
C.addTransition(NewState);

// Emit a bug report
auto R = std::make_unique<PathSensitiveBugReport>(BT, Msg, ErrNode);
C.emitReport(std::move(R));
```

`generateNonFatalErrorNode()` creates a new `ExplodedNode` marked as an error locus but leaves the path open — other checkers can still fire on successors. `generateSink()` marks the node as a graph sink, terminating the path (used for guaranteed-fatal conditions like `assert(false)` or memory exhaustion).

---

## 45.6 Built-in Checkers

### 45.6.1 `core.NullDereference`

Implemented in
[`clang/lib/StaticAnalyzer/Checkers/DereferenceChecker.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/StaticAnalyzer/Checkers/DereferenceChecker.cpp),
this checker hooks `check::Location`. On every memory read/write, it retrieves the base address `SVal` and calls `State->assume(MemVal.castAs<DefinedOrUnknownSVal>(), false)`. If the null-hypothesis state is non-null (i.e., null is feasible), it generates an error node. The checker also hooks `check::Bind` for writes through potential nulls and `check::PreStmt<MemberExpr>` for arrow dereferences, giving three separate code paths that funnel into a common `reportBug()` helper.

### 45.6.2 `core.uninitialized.*`

`UndefBranchChecker` hooks `check::BranchCondition`. If the condition `SVal` is `UndefinedVal`, it reports a branch on uninitialized data. `UndefResultChecker` fires on `check::PostStmt<BinaryOperator>` when the result is `UndefinedVal`. `CallAndMessageChecker` fires `check::PreCall` when any argument is `UndefinedVal` or when the callee expression itself is undefined.

### 45.6.3 `cplusplus.NewDelete`

The `MallocChecker` family (
[`clang/lib/StaticAnalyzer/Checkers/MallocChecker.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/StaticAnalyzer/Checkers/MallocChecker.cpp))
handles both `malloc`/`free` and `new`/`delete` through a unified allocation-state machine. It uses a checker-private `ProgramStateTrait`:

```cpp
REGISTER_MAP_WITH_PROGRAMSTATE(RegionState, SymbolRef, RefState)
```

where `RefState` encodes `{Allocated, Released, Escaped, Relinquished}`. On `check::PostCall` of `malloc`/`operator new`, the returned symbol is mapped to `Allocated`. On `check::PreCall` of `free`/`operator delete`, the checker looks up the symbol:
- If `Released`: double-free error node.
- If `Allocated`: transition to `Released`.
- If not found: unknown deallocation (reported by `cplusplus.NewDeleteLeaks`).

`check::DeadSymbols` fires when the allocator's return symbol dies without reaching `Released`, signaling a memory leak.

### 45.6.4 `core.DivideZero`

`DivZeroChecker` hooks `check::PreStmt<BinaryOperator>` for `/` and `%` operations. It calls `State->assume(Denominator, false)` — if zero is feasible, it reports the bug and produces both the zero-denominator state (error) and the non-zero state (normal continuation), forking the path.

### 45.6.5 `deadcode.DeadStores`

An AST-level checker (using `check::ASTCodeBody`) that builds a `LiveVariables` analysis over the function's CFG, then walks assignments to detect stored values that are never subsequently read. Because it uses Clang's intra-procedural `LiveVariables` rather than symbolic execution, it is faster but less precise than path-sensitive checkers.

### 45.6.6 `security.insecureAPI.*`

A family of `check::PreCall` checkers that fire on calls to `gets`, `strcpy`, `sprintf`, `rand`, `mktemp`, `alloca`, and other functions with well-known security problems. The pattern is simple: match by `CallDescription` (function name + argument count), then always report. No state is needed.

`CallDescription` (
[`clang/include/clang/StaticAnalyzer/Core/PathSensitive/CallDescription.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/CallDescription.h))
matches calls by fully-qualified name (optionally with namespace components) and argument count. A `CallDescriptionMap<T>` maps multiple call descriptions to checker-specific data in O(1) amortized time using a hash of the function name, making it the preferred pattern for checkers that handle many different function names.

### 45.6.7 `alpha.cplusplus.STLAlgorithmModeling`

A modeling checker (not a bug-finding checker) that uses `eval::Call` to provide accurate symbolic summaries of standard library algorithms. When `std::find()` is called, for example, the checker returns either a symbolic iterator (constrained to be in `[begin, end)`) on success or `end` on failure, forking the path. This gives downstream checkers — particularly iterator-validity checkers — accurate iterator states without inlining the full algorithm implementation.

The `eval::Call` mechanism lets a checker **replace** the default `ExprEngine` call evaluation entirely. Registering for `eval::Call` on a particular `CallDescription` causes the checker's `evalCall()` method to be invoked instead of the normal inlining / external-function modeling logic. This is the mechanism used for all standard-library modeling.

### 45.6.8 `alpha.webkit.UncountedLambdaCapturesChecker`

WebKit-specific checkers in `clang/lib/StaticAnalyzer/Checkers/WebKit/` demonstrate production-quality checker families targeting a specific codebase. They enforce WebKit's reference-counting rules: raw pointers to ref-counted objects must not be captured by lambdas that can outlive their enclosing context. These checkers combine `check::PostStmt<LambdaExpr>` with AST inspection of captured variables' types, illustrating how path-sensitive and AST-level reasoning can be combined in a single checker.

---

## 45.7 Writing a Custom Checker

### 45.7.1 Step-by-Step Implementation

Consider a checker that enforces: every `FILE*` returned by `fopen()` must be closed via `fclose()` before the pointer leaves scope. The implementation pattern:

```cpp
// clang/lib/StaticAnalyzer/Checkers/FopenChecker.cpp
#include "clang/StaticAnalyzer/Core/BugReporter/BugType.h"
#include "clang/StaticAnalyzer/Core/Checker.h"
#include "clang/StaticAnalyzer/Core/PathSensitive/CallEvent.h"
#include "clang/StaticAnalyzer/Core/PathSensitive/CheckerContext.h"

using namespace clang;
using namespace ento;

namespace {

// Private state: maps SymbolRef (FILE* symbol) → bool (open)
REGISTER_MAP_WITH_PROGRAMSTATE(OpenFiles, SymbolRef, bool)

class FopenChecker
    : public Checker<check::PostCall, check::PreCall, check::DeadSymbols> {

  mutable std::unique_ptr<BugType> LeakBT, DoubleCloseBT;

  void initBugTypes() const {
    if (!LeakBT)
      LeakBT.reset(new BugType(this, "FILE* leak", "Unix Stream API"));
    if (!DoubleCloseBT)
      DoubleCloseBT.reset(new BugType(this, "Double fclose", "Unix Stream API"));
  }

public:
  void checkPostCall(const CallEvent &Call, CheckerContext &C) const {
    // Detect fopen() return
    if (!Call.isGlobalCFunction("fopen") || Call.getNumArgs() != 2)
      return;
    SVal RetVal = Call.getReturnValue();
    auto FileSym = RetVal.getAsSymbol();
    if (!FileSym) return;
    ProgramStateRef State = C.getState();
    // Non-null return → record as open
    ProgramStateRef NonNull, Null;
    std::tie(NonNull, Null) = State->assume(
        RetVal.castAs<DefinedOrUnknownSVal>());
    if (NonNull) {
      NonNull = NonNull->set<OpenFiles>(FileSym, true);
      C.addTransition(NonNull);
    }
    if (Null)
      C.addTransition(Null); // NULL return: no tracking needed
  }

  void checkPreCall(const CallEvent &Call, CheckerContext &C) const {
    if (!Call.isGlobalCFunction("fclose") || Call.getNumArgs() != 1)
      return;
    SVal Arg = Call.getArgSVal(0);
    SymbolRef Sym = Arg.getAsSymbol();
    if (!Sym) return;
    ProgramStateRef State = C.getState();
    const bool *IsOpen = State->get<OpenFiles>(Sym);
    if (IsOpen && !*IsOpen) {
      // Already closed
      initBugTypes();
      ExplodedNode *N = C.generateNonFatalErrorNode();
      if (!N) return;
      auto R = std::make_unique<PathSensitiveBugReport>(
          *DoubleCloseBT, "Double fclose()", N);
      R->addRange(Call.getSourceRange());
      C.emitReport(std::move(R));
      return;
    }
    // Mark as closed
    State = State->set<OpenFiles>(Sym, false);
    C.addTransition(State);
  }

  void checkDeadSymbols(SymbolReaper &Reaper, CheckerContext &C) const {
    ProgramStateRef State = C.getState();
    for (auto [Sym, IsOpen] : State->get<OpenFiles>()) {
      if (Reaper.isDead(Sym) && IsOpen) {
        initBugTypes();
        ExplodedNode *N = C.generateNonFatalErrorNode();
        if (!N) { State = State->remove<OpenFiles>(Sym); continue; }
        auto R = std::make_unique<PathSensitiveBugReport>(
            *LeakBT, "Opened FILE* never closed", N);
        C.emitReport(std::move(R));
      }
      if (Reaper.isDead(Sym))
        State = State->remove<OpenFiles>(Sym);
    }
    C.addTransition(State);
  }
};
} // namespace

void ento::registerFopenChecker(CheckerManager &Mgr) {
  Mgr.registerChecker<FopenChecker>();
}

bool ento::shouldRegisterFopenChecker(const CheckerManager &) {
  return true;
}
```

### 45.7.2 Registration via `Checkers.td`

Checkers are declared in
[`clang/include/clang/StaticAnalyzer/Checkers/Checkers.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Checkers/Checkers.td):

```tablegen
let ParentPackage = Unix in {
def FopenChecker : Checker<"Fopen">,
  HelpText<"Check for unclosed FILE* handles">,
  Documentation<HasDocumentation>;
} // end "unix"
```

TableGen generates `Checkers.inc` which is included by `BuiltinCheckerRegistration.h`. The pair `registerFopenChecker` / `shouldRegisterFopenChecker` must match the `def` name exactly.

For out-of-tree checkers (plugins), registration bypasses `Checkers.td`. The checker is registered via `AnalysisASTConsumer::AddCheckerRegistrationFn()`:

```cpp
// Plugin entry point
extern "C" void clang_registerCheckers(CheckerRegistry &Registry) {
  Registry.addChecker<FopenChecker>(
      "unix.Fopen", "Check for unclosed FILE* handles", "");
}
```

Load with `-load /path/to/mychecker.so -analyzer-checker=unix.Fopen`. This mechanism enables organizations to ship proprietary checkers without modifying the Clang tree.

### 45.7.3 `REGISTER_MAP_WITH_PROGRAMSTATE` Internals

The macro expands to a specialization of `ProgramStateTrait<OpenFiles>`:

```cpp
// Conceptual expansion:
class OpenFiles {};
using OpenFilesTy = llvm::ImmutableMap<SymbolRef, bool>;
template<> struct ProgramStateTrait<OpenFiles>
    : public ProgramStatePartialTrait<OpenFilesTy> {
  static void *GDMIndex() { static int Index; return &Index; }
};
```

The unique `&Index` serves as the `GenericDataMap` key. `State->get<OpenFiles>(Sym)` calls `ProgramStateTrait<OpenFiles>::MakeData(State->GDM.lookup(&Index))` and then performs a lookup in the resulting `ImmutableMap`. All of this is O(log n) on the persistent tree.

The three registration macros cover the common collection shapes:

| Macro | Underlying type | Methods |
|-------|----------------|---------|
| `REGISTER_MAP_WITH_PROGRAMSTATE(Name, K, V)` | `llvm::ImmutableMap<K,V>` | `get<Name>(K)`, `set<Name>(K,V)`, `remove<Name>(K)` |
| `REGISTER_SET_WITH_PROGRAMSTATE(Name, T)` | `llvm::ImmutableSet<T>` | `contains<Name>(T)`, `add<Name>(T)`, `remove<Name>(T)` |
| `REGISTER_LIST_WITH_PROGRAMSTATE(Name, T)` | `llvm::ImmutableList<T>` | `get<Name>()`, `add<Name>(T)` |
| `REGISTER_TRAIT_WITH_PROGRAMSTATE(Name, T)` | Any type with `ProgramStatePartialTrait` | Scalar: `get<Name>()`, `set<Name>(V)` |

For scalar values (e.g., a single `bool` or an enum), `REGISTER_TRAIT_WITH_PROGRAMSTATE` is appropriate. For maps with many entries per state, `REGISTER_MAP_WITH_PROGRAMSTATE` is correct and provides structural sharing across path forks.

---

## 45.8 `CallEvent` and Inter-Procedural Analysis

### 45.8.1 `CallEvent` Hierarchy

`CallEvent` (
[`clang/include/clang/StaticAnalyzer/Core/PathSensitive/CallEvent.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/CallEvent.h))
is an abstract base representing a call site in the symbolic execution context. The concrete kinds are identified by `CallEventKind`:

| Kind constant | Class | Description |
|--------------|-------|-------------|
| `CE_Function` | `SimpleFunctionCall` | Free function, static member |
| `CE_CXXMember` | `CXXMemberCall` | Non-static member function |
| `CE_CXXMemberOperator` | `CXXMemberOperatorCall` | Overloaded operator via member |
| `CE_CXXStaticOperator` | `CXXStaticOperatorCall` | Static operator (C++23) |
| `CE_CXXConstructor` | `CXXConstructorCall` | Constructor |
| `CE_CXXInheritedConstructor` | `CXXInheritedConstructorCall` | Delegating inherited ctor |
| `CE_CXXDestructor` | `CXXDestructorCall` | Destructor |
| `CE_CXXAllocator` | `CXXAllocatorCall` | `operator new` |
| `CE_CXXDeallocator` | `CXXDeallocatorCall` | `operator delete` |
| `CE_Block` | `BlockCall` | Clang block literal |
| `CE_ObjCMessage` | `ObjCMethodCall` | Objective-C message send |

Key `CallEvent` methods used in checkers:

```cpp
SVal getArgSVal(unsigned Idx) const;   // symbolic value of argument Idx
SVal getReturnValue() const;           // symbolic return value
const Decl *getDecl() const;           // callee declaration (may be null)
bool isGlobalCFunction(StringRef Name) const;
const Expr *getOriginExpr() const;     // the call expression in the AST
SourceRange getSourceRange() const;
```

### 45.8.2 Inlining Mechanism

When `ExprEngine` encounters a `CallEvent` and the callee has an available body, `ExprEngine::inlineCall()` pushes a `CallEnter` `ProgramPoint` onto the worklist. A new `StackFrameContext` is allocated via `AnalysisDeclContextManager`, the arguments are bound into the callee's `StackArgumentsSpaceRegion`, and the engine enters the callee's CFG. When execution reaches the callee's exit block, a `CallExitEnd` node transfers the return value back to the caller's `Environment` and pops the stack frame.

The inlining mode is controlled by `AnalysisInliningMode`:
- `All`: inline every function with a body.
- `NoRedundancy`: skip re-analysis of a callee if the same function was already analyzed with a compatible state (the default).

`AnalyzerOptions::getMaxInlinableSize()` (default 100 statements) prevents inlining functions so large that the resulting path explosion would blow the node budget.

When a callee cannot be inlined — because it has no body, exceeds the size limit, or is a virtual dispatch that cannot be resolved — `ExprEngine` applies **conservative modeling**: all pointer arguments are treated as potentially modified (invalidated), the return value becomes a fresh `SymbolConjured`, and any global state accessible through pointer arguments is marked unknown. This is the `evalCall` default behavior in `ExprEngine::defaultEvalCall()`. Checkers can register `eval::Call` to provide more precise models for specific functions, overriding the conservative default entirely.

### 45.8.3 Objective-C Retain Count Checking

`RetainCountChecker` (
[`clang/lib/StaticAnalyzer/Checkers/RetainCountChecker/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/StaticAnalyzer/Checkers/RetainCountChecker/))
uses `RetainSummaryManager` to model ARC/MRC reference counting rules without inlining every `retain`/`release`. For each `ObjCMethodCall` or `CFunctionCall`, `RetainSummaryManager::getSummary()` returns a `RetainSummary` describing parameter and return value effects (`RetainEffect::IncRef`, `DecRef`, `MayEscape`, etc.). The checker applies the summary to the symbolic reference-count state, reporting over-releases and leaks on dead symbols.

---

## 45.9 Bug Reports and Path Diagnostics

### 45.9.1 `PathSensitiveBugReport` and `BugType`

`PathSensitiveBugReport` (
[`clang/include/clang/StaticAnalyzer/Core/BugReporter/BugReporter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/BugReporter/BugReporter.h))
wraps a `BugType`, a description string, and the `ExplodedNode` at which the bug was detected (the **error node**). `BugType` carries a checker-pointer, a short name, and a category string used by output consumers.

Optional metadata on a report:
```cpp
R->addRange(CE->getSourceRange());           // highlight a source range
R->addNote("Value returned here", NoteLoc);  // inline note
R->markInteresting(Sym);                     // flag a symbol for path notes
R->markInteresting(Region);                  // flag a memory region
```

`BugReporter::emitReport()` takes ownership of the report, deduplicates it against already-emitted reports with the same `(BugType, Location)` pair, then runs path reconstruction.

### 45.9.2 `BugReporterVisitor` — Adding Path Notes

Path notes ("null returned here", "memory allocated here") are injected by `BugReporterVisitor` subclasses registered on the report. The visitor's `VisitNode()` method is called for every node on the error path; it can return a `PathDiagnosticPieceRef` (typically a `PathDiagnosticEventPiece`) to be spliced into the path diagnostic.

Important built-in visitors include:
- `NilReceiverBRVisitor`: annotates Obj-C nil message sends
- `UndefOrNullArgVisitor`: explains where a null/undef value was introduced
- `ConditionBRVisitor`: labels branch conditions that constrained a path
- `CXXSelfAssignmentBRVisitor`: annotates self-assignment detection
- `TrackConstraintBRVisitor`: follows a constrained symbol backwards to its origin
- `Z3CrosscheckVisitor`: fires when Z3 refutes a report the range solver emitted

### 45.9.2b Path Reconstruction — `BugPathGetter`

Path reconstruction traverses the `ExplodedGraph` backwards from the error node to the analysis root, computing the **shortest** feasible path. This is non-trivial because the `ExplodedGraph` is a DAG: multiple predecessor paths may reach the error node through different program paths. The `BugReporter` uses `PathPruner` to remove path segments that don't contribute to the bug (e.g., function calls that are fully modeled), then applies each registered `BugReporterVisitor` to every node on the pruned path to inject contextual notes.

The `ConditionBRVisitor` inspects `BlockEdge` nodes: if the branch condition constrained a value that eventually caused the bug, it adds a note like `"Assuming 'ptr' is null"` at the branch point. The `TrackConstraintBRVisitor` follows a specific symbol backwards through the graph to find where its constraint was introduced, adding `"Symbol assigned here"` notes. Together, these visitors make the difference between a report that shows only the crash site and one that walks the reader through the causal chain.

### 45.9.3 Diagnostic Output Formats

`PathDiagnosticConsumer` implementations (declared in
[`clang/include/clang/StaticAnalyzer/Core/PathDiagnosticConsumers.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathDiagnosticConsumers.h))
consume the reconstructed `PathDiagnostic` objects:

| Flag | Consumer | Description |
|------|----------|-------------|
| `-analyzer-output=html` | `HTMLDiagnostics` | Self-contained HTML with path arrows |
| `-analyzer-output=plist` | `PlistDiagnostics` | Apple Xcode integration format |
| `-analyzer-output=sarif` | `SarifDiagnostics` | OASIS SARIF 2.1 for CI tooling |
| `-analyzer-output=text` | `TextPathDiagnostics` | Plain-text for terminals |

`scan-build` (the wrapper script) invokes `clang --analyze` with HTML output, then collates results into a browsable report directory. `clang-tidy` can consume analyzer reports through its `clang-analyzer-*` check family, which re-uses the same checker infrastructure but integrates with clang-tidy's fix-it and suppression machinery.

---

## 45.10 Taint Analysis

### 45.10.1 `taint::` API

`GenericTaintChecker` (
[`clang/lib/StaticAnalyzer/Checkers/GenericTaintChecker.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/StaticAnalyzer/Checkers/GenericTaintChecker.cpp))
implements data-flow taint propagation using the public API in
[`clang/include/clang/StaticAnalyzer/Checkers/Taint.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Checkers/Taint.h):

```cpp
// Mark a value as tainted
ProgramStateRef addTaint(ProgramStateRef State, SVal V,
                         TaintTagType Kind = TaintTagGeneric);
// Query taint
bool isTainted(ProgramStateRef State, SVal V,
               TaintTagType Kind = TaintTagGeneric);
// Remove taint (after sanitization)
ProgramStateRef removeTaint(ProgramStateRef State, SVal V);
// Partial taint: sub-region of a compound symbol
ProgramStateRef addPartialTaint(ProgramStateRef State, SymbolRef ParentSym,
                                const SubRegion *R, TaintTagType Kind);
```

`TaintTagType` is a `typedef unsigned`, enabling multiple independent taint domains (e.g., SQL injection vs. command injection) without interference.

### 45.10.2 Sources, Propagators, and Sinks

**Sources** introduce taint on `check::PostCall`. The default list includes:
- `scanf`, `fscanf`, `sscanf` — taint all pointer arguments written through
- `getenv` — taint return value
- `read`, `pread`, `recv`, `recvfrom` — taint buffer argument
- `fgets`, `fgetc`, `getchar` — taint return / buffer

**Propagators** spread taint through operations. The checker hooks `check::PostCall` for string functions (`strcpy`, `strcat`, `sprintf`, `memcpy`) and marks the destination tainted if any source argument is tainted. Arithmetic propagation is implicit: `SValBuilder` tracks symbolic expressions, and `isTainted` follows the `SymExpr` tree to its leaves.

**Sinks** check for taint on `check::PreCall`. The default sinks are:
- `system`, `popen`, `execl`, `execv*` — command injection
- `dlopen` — library injection
- Format string arguments of `printf`-family where the format is tainted

### 45.10.3 Custom Taint Configuration via YAML

`GenericTaintChecker` reads an optional YAML configuration specifying additional propagation rules:

```yaml
# analyzer-taint-config.yaml
Propagations:
  - Name: my_copy
    DstArgs: [0]      # argument index to taint
    SrcArgs: [1]      # if this argument is tainted
Sinks:
  - Name: execute_query
    Args: [0]         # argument 0 is a sink
Sources:
  - Name: get_user_input
    RetVal: true      # return value is tainted
```

Pass via `-analyzer-config alpha.security.taint.TaintPropagation:Config=path/to/config.yaml`. This mechanism lets security teams audit application-specific trust boundaries without modifying checker source code.

### 45.10.4 Taint and Partial Regions

When user input populates only part of a structure (e.g., a field read from a network packet), `addPartialTaint()` records a `(ParentSym, SubRegion)` pair. Subsequent reads from the sub-region propagate the taint; reads from untainted sub-regions do not. This avoids false positives from overly broad whole-struct taint marking.

Taint propagation interacts with the constraint manager: if a tainted value is constrained to a safe range (e.g., bounds-checked before use as an array index), the sanitization should be modeled explicitly using `removeTaint()`. Without explicit removal, the taint state persists through constraints, which is conservative but can produce false positives when the program implements correct input validation. The YAML configuration's `Filters` stanza allows naming sanitization functions whose output arguments are automatically de-tainted.

---

## 45.11 Z3 Constraint Solver Integration

### 45.11.1 `Z3ConstraintManager`

The default `RangeConstraintManager` represents constraints as sets of integer ranges per symbol. It is fast but incomplete: it may fail to recognize that `x * 2 == 5` is infeasible for integer `x`. The Z3-backed `SMTConstraintManager` (
[`clang/lib/StaticAnalyzer/Core/Z3ConstraintManager.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/StaticAnalyzer/Core/Z3ConstraintManager.cpp))
translates the accumulated `SymExpr` constraints into Z3 assertions and calls `Z3_check_assumptions()`.

Enable with:
```bash
clang --analyze -Xanalyzer -analyzer-constraints=z3 foo.cpp
```

Z3 is a dependency that must be present at build time (`LLVM_ENABLE_Z3_SOLVER=ON` in CMake). The `SMTConv.h` header (`clang/include/clang/StaticAnalyzer/Core/PathSensitive/SMTConv.h`) provides the symbol-to-SMT conversion logic, factored out of the solver implementation.

### 45.11.2 Refutation Mode vs. Full Solving

Two distinct use modes exist:

**Full solving** (`-analyzer-constraints=z3`): Z3 replaces the range solver entirely. Every `assume()` call goes through Z3. This is precise but expensive — 10–100× slower on real code because Z3 must be invoked on every path fork.

**Refutation / cross-check mode** (`-analyzer-config crosscheck-with-z3=true`): The range solver runs normally and generates reports. Before emitting each report, `Z3CrosscheckVisitor` reconstructs the path constraints and queries Z3 for satisfiability. If Z3 finds the path infeasible, the report is suppressed as a false positive. This eliminates the false positives caused by range solver imprecision while keeping analysis time manageable.

### 45.11.3 When Z3 Catches What Range Solving Misses

Consider:
```c
int f(int x) {
  if (x * x == -1) {  // impossible for any integer
    int *p = 0;
    return *p;         // would be false positive without Z3
  }
  return x;
}
```

The range solver cannot represent `x*x == -1` as a range constraint; it treats the branch as feasible and reports a null dereference. Z3 correctly identifies `x*x == -1` as UNSAT over integers and suppresses the report.

Conversely, Z3's non-linear arithmetic solver can time out on complex expressions, causing reports to be retained (conservative). The `analyzer-z3-timeout` option (default 2000ms) limits per-query time.

---

## 45.12 Performance and Scalability

### 45.12.1 `FunctionSummariesTy` and Result Caching

`FunctionSummariesTy` (
[`clang/include/clang/StaticAnalyzer/Core/PathSensitive/FunctionSummary.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/FunctionSummary.h))
records per-function analysis results: the set of `ExplodedNode` sinks, whether analysis was exhausted or aborted, and the number of blocks analyzed. When `ExprEngine` considers inlining a callee, it checks whether the callee's summary is already available and whether the call-site state is "covered" by a previously analyzed call — the **NoRedundancy** heuristic. If covered, the callee is not re-analyzed; its previously computed effects are applied directly. This is the primary scalability mechanism for large translation units.

### 45.12.2 Key Tuning Options

| Option | Default | Effect |
|--------|---------|--------|
| `-analyzer-max-nodes` | 200000 | Abort analysis after N nodes |
| `-analyzer-max-loop` | 4 | Unroll loops at most N times |
| `ipa=dynamic-bifurcate` | — | Inline + model virtual calls |
| `ipa-always-inline-size` | 3 | Always inline bodies ≤ N stmts |
| `max-inlinable-size` | 100 | Skip inlining bodies > N stmts |
| `max-times-inline-large` | 32 | Limit re-inlining of large functions |
| `-analyzer-display-progress` | off | Print each analyzed function |
| `crosscheck-with-z3` | false | Enable Z3 false-positive suppression |

### 45.12.3 Worklist Strategies and Their Trade-offs

The choice of worklist algorithm has a significant impact on both analysis quality and resource consumption:

**DFS (depth-first search)** explores each path to its completion before backtracking. This produces the best diagnostic quality for individual paths — the analyzer reaches function exits frequently, enabling accurate dead-symbol detection and leak reporting. However, DFS can exhaust the node budget on a single long path while leaving large portions of the CFG unexplored.

**BFS (breadth-first search)** explores all paths to a given depth before going deeper. It provides better coverage of shallow bugs across many paths but often runs out of nodes before reaching deep function exits, degrading leak detection.

**UnexploredFirst (the recommended default for scan-build)** uses a priority queue that favors CFG blocks not yet reached by any path. This heuristic provides better CFG coverage than DFS while maintaining path depth comparable to BFS. It is selected via `-analyzer-config exploration_strategy=unexplored_first`.

**DFSBi (DFS with backtracking to unexplored)** is an experimental hybrid that runs DFS but restarts from the most recently unexplored block when a path is exhausted. It is enabled with `unexplored_first_queue`.

### 45.12.4 `clang --analyze` vs. `clang-tidy`

`clang --analyze` runs the full path-sensitive engine on every translation unit, producing high-precision reports but consuming 5–50× the time of compilation. `clang-tidy` with `clang-analyzer-*` checks reuses the same checker code but runs inside the tidy framework, enabling incremental re-analysis and integration with compile-command databases. For CI pipelines, `scan-build` remains the standard wrapper — it intercepts build commands, runs the analyzer in parallel, and aggregates HTML reports. For IDE integration, `clangd` exposes a subset of checkers (excluding the most expensive ones) through background analysis.

### 45.12.5 Scaling to Large Codebases

At translation-unit scale, the analyzer is self-contained. Across translation units, `CodeChecker` (an open-source analyzer management tool) stores results in a PostgreSQL database, deduplicates reports across TUs, and tracks report lifetime across commits. The `CTU` (Cross-Translation-Unit) analysis mode in Clang proper (`-analyzer-config ipa=basic-inlining -analyzer-config experimental-enable-naive-ctu-analysis=true`) imports external function bodies from a pre-built AST dump index, enabling limited cross-TU inlining without full LTO.

---

## 45.13 Clang Lifetime Safety Analysis

### 45.13.1 History and Production Graduation

Lifetime safety analysis in Clang traces to Herb Sutter's 2018 "Lifetime Safety: Preventing Common Dangling" proposal, which outlined a static analysis approach for detecting dangling references in C++ through a type-and-effects system layered on the existing type system. An experimental implementation landed in Clang in 2019 under the `-Wlifetime` umbrella, initially covering only the simplest dangling-reference patterns. Between 2020 and 2025, the implementation expanded to cover moved-from objects, non-trivially-destructed temporaries, and struct fields holding references to locals. In Clang 22, five checker sub-categories have graduated to production status, enabled by default when building with `-std=c++17` or later and `-Wall`.

### 45.13.2 Five Production Checkers

**UseAfterReturn** detects references or pointers to local variables that are returned from the enclosing function or captured in a lambda that outlives the function's stack frame:

```cpp
int& bad() {
  int x = 42;
  return x;   // warning: address of local variable 'x' returned
              // [-Wreturn-local-addr]
}

auto bad_capture() {
  int x = 0;
  return [&x]() { return x; };  // warning: reference to local variable 'x'
                                 // will escape [-Wdangling]
}
```

The diagnostic is surfaced via `-Wreturn-local-addr` (long-standing) and, for the lambda capture case, the newer `-Wdangling-capture` added in Clang 18.

**UseAfterMove** detects access to moved-from objects before reinitialization. Clang's flow-sensitive analysis (§45.14) tracks moved-from state through the CFG. When a variable is the operand of `std::move()` and then accessed on a reachable path without an intervening assignment, the checker fires. The `bugprone-use-after-move` clang-tidy check (§47.3.4) provides an AST-level approximation of the same property; the lifetime checker operates at the CFG level and is therefore more precise.

**NonTriviallyDestructedTemporaries** catches temporaries with non-trivial destructors bound to const references:

```cpp
struct Buffer {
  Buffer() { data = new char[64]; }
  ~Buffer() { delete[] data; }
  char* data;
};

void bad() {
  const char* p;
  {
    const Buffer& b = Buffer();  // temporary destroyed at end of full-expr
    p = b.data;
  }
  use(p);  // warning: 'p' references destroyed temporary [-Wdangling]
}
```

The C++ lifetime-extension rules (binding a temporary to a `const T&` extends its lifetime to the reference's scope) are modeled; only cases outside those rules are flagged.

**FieldReferenceToOutOfScopeLocal** detects struct or class fields declared as references (`T&`) that are initialized with local variables whose lifetime is shorter than the enclosing object's:

```cpp
struct View {
  int& ref;
  View(int& r) : ref(r) {}
};

View make_view() {
  int x = 10;
  return View{x};  // warning: member 'ref' binds to local variable 'x'
                   // [-Wdangling-field]
}
```

**LifetimeboundViolation** enforces `[[clang::lifetimebound]]` annotations (§45.13.3). When a function annotated `[[clang::lifetimebound]]` on a parameter returns a value, and the caller binds the return value to a reference, Clang verifies that the annotated argument outlives the reference.

### 45.13.3 `[[clang::lifetimebound]]` Mechanics

The `[[clang::lifetimebound]]` attribute, declared in
[`clang/include/clang/Basic/AttrDocs.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/AttrDocs.td),
marks a function parameter (or the implicit `this` parameter for member functions) as a lifetime dependency for the return value:

```cpp
// Return value's lifetime is bounded by the lifetime of 'c'
const char* string_data(const std::string& s [[clang::lifetimebound]]);

// Chaining: view's lifetime bounded by the vector
std::span<int> first_half(std::vector<int>& v [[clang::lifetimebound]]);
```

Standard library types in LLVM's libc++ carry `[[clang::lifetimebound]]` annotations on `string_view`, `span`, `optional::value()`, and similar APIs. When a call site binds the result to a local reference and the bound argument is a temporary, Clang emits a diagnostic:

```cpp
std::string_view sv = std::string("hello");
// warning: object whose reference is captured will be destroyed at the
// end of the full-expression [-Wdangling]
```

Fix-it hints suggest adding `auto` storage to give the temporary a name, or using a `std::string` directly. Clang 22 can automatically suggest `[[clang::lifetimebound]]` annotations for user-defined functions via `-Wsuggest-attribute=lifetimebound` (experimental).

### 45.13.4 Enabling Lifetime Warnings

The warnings are organized into two flag groups:

| Flag | Coverage |
|------|----------|
| `-Wdangling` | UseAfterReturn, NonTriviallyDestructedTemporaries, FieldReferenceToOutOfScopeLocal, LifetimeboundViolation |
| `-Wdangling-capture` | Lambda capture of local variables that escape |
| `-Wdangling-field` | FieldReferenceToOutOfScopeLocal for struct/class fields |
| `-Wreturn-local-addr` | UseAfterReturn (pointer/reference returns) |
| `-Wlifetime` | Experimental umbrella (may be incomplete in Clang 22) |

All are included in `-Wall` except `-Wlifetime`. The `cppcoreguidelines-pro-lifetime` clang-tidy checks (part of the `cppcoreguidelines` module) implement a subset of Herb Sutter's original lifetime profile using AST matchers, operating at a coarser granularity than the CFG-based checkers but integrable into CI pipelines via clang-tidy's YAML suppression mechanism.

### 45.13.5 Comparison with Rust's Borrow Checker

Clang's lifetime analysis is deliberately pragmatic: it detects the common-case violations that appear frequently in production C++ without requiring programmers to annotate every function with lifetime parameters. The trade-off versus Rust's borrow checker is substantial:

- **Completeness**: Rust's borrow checker guarantees no dangling references if the program compiles. Clang's warnings are heuristic — false negatives exist, and some patterns that are provably safe are flagged as suspicious.
- **Annotation burden**: Rust requires explicit lifetime parameters (`'a`) on all reference-returning functions. Clang's `[[clang::lifetimebound]]` is opt-in; unannotated functions receive no checking.
- **Ownership tracking**: Rust tracks exclusive (`&mut`) vs. shared (`&`) borrows, preventing aliased mutation. Clang's analysis does not model ownership or aliasing between live references.
- **Integration**: Clang's checker operates on existing C++ code without modification, enabling incremental adoption; Rust's model requires lifetime annotations at every function boundary.

For new code written in modern C++ idioms (`std::span`, `std::string_view`, structured bindings), `-Wdangling` catches the majority of practical lifetime errors. For legacy code with pointer-heavy APIs, false negatives are common.

---

## 45.14 Clang FlowSensitive Dataflow Analysis Framework

### 45.14.1 Architectural Distinction from the Path-Sensitive Engine

The classical static analyzer (§45.1–§45.12) builds an `ExplodedGraph` by forking state at every branch, tracking each feasible path through the function independently. This **path-sensitive** approach achieves high precision — it knows exactly which constraints hold on a given path — but at exponential worst-case cost. Analysis time can grow as 2^N for functions with N independent conditionals, requiring the node-count cutoff (§45.12.2) that abandons analysis on large functions.

The **FlowSensitive** (FS) framework, introduced in 2021 and stabilized in Clang 22, takes a different approach rooted in classical dataflow analysis. Rather than exploring individual paths, it computes a single lattice value per CFG block by joining (merging) information from all predecessor blocks. This is the same foundation as `LiveVariables`, `DominatorTree`, and the SSA construction in the backend — but generalized to user-defined lattices with arbitrary transfer functions. The cost is polynomial: O(N × H) where N is the number of CFG blocks and H is the lattice height.

Source location:
[`clang/include/clang/Analysis/FlowSensitive/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Analysis/FlowSensitive/)

### 45.14.2 Core Abstractions

**`DataflowAnalysisContext`** (
[`clang/include/clang/Analysis/FlowSensitive/DataflowAnalysisContext.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Analysis/FlowSensitive/DataflowAnalysisContext.h))
owns the infrastructure shared across analyses on a single function: the `WatchedLiteralsSolver` SAT instance, an `Arena` for allocating `Value` and `StorageLocation` objects, and the environment model for variables in scope. It is created once per function analysis and reused across multiple `DataflowAnalysis` instances on the same CFG.

**`DataflowAnalysis<LatticeT>`** (
[`clang/include/clang/Analysis/FlowSensitive/DataflowAnalysis.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Analysis/FlowSensitive/DataflowAnalysis.h))
is the CRTP base for user analyses. The type parameter `LatticeT` must satisfy the `JoinSemilattice` concept: it must define `LatticeJoinEffect join(const LatticeT& other)` that computes the least-upper-bound of two lattice elements. The CRTP subclass overrides:

```cpp
// Transfer function: update lattice element for one CFG element
void transfer(const CFGElement &Elt,
              LatticeT &State,
              Environment &Env);
```

**`Environment`** (
[`clang/include/clang/Analysis/FlowSensitive/DataflowEnvironment.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Analysis/FlowSensitive/DataflowEnvironment.h))
is the FS framework's memory model. It maps `StorageLocation` objects (analogous to `MemRegion` in the path-sensitive engine) to `Value` objects (analogous to `SVal`). Unlike `ProgramState`, `Environment` is mutable during transfer function execution — the immutability is managed at the block level by the fixpoint iterator, which copies the environment before calling `transfer` for each element.

**`TransferOptions`** controls optional framework behaviors: `ContextSensitiveOptions` enables limited inter-procedural analysis by inlining small callees' transfer functions.

### 45.14.3 Lattice Requirements and Widening

Every `LatticeT` must implement `join()`, which computes the merge of two states at CFG join points (e.g., after an `if`/`else`). For finite-height lattices (boolean flags, small enums, sets of bounded size), the fixpoint iteration converges automatically: since `join()` can only raise elements in the lattice order, and the lattice is finite, the algorithm terminates in at most H iterations per block, where H is the lattice height.

For infinite-height lattices (e.g., sets of integers, recursive data structures), the framework provides a **widening** hook:

```cpp
// Called after join() when a block has been visited more than once
// and the lattice element has grown. Must over-approximate to ensure
// termination.
LatticeJoinEffect widenAndJoin(const LatticeT& Previous, LatticeT& Current);
```

`widenAndJoin` is the standard widening operator from abstract interpretation: it must return an over-approximation that is larger than both inputs, chosen such that the sequence of widened values converges in a bounded number of iterations. For integer-range lattices, a common widening strategy replaces a growing bound with `+∞`.

### 45.14.4 SAT Solver Integration

A key architectural decision in the FS framework is the integration of a propositional SAT solver for reasoning about boolean implications. The `WatchedLiteralsSolver` (
[`clang/include/clang/Analysis/FlowSensitive/WatchedLiteralsSolver.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Analysis/FlowSensitive/WatchedLiteralsSolver.h))
implements the two-watched-literal DPLL algorithm. It is added in 2022 to address a precision problem: without SAT solving, joining states at merge points loses all information about correlations between boolean variables in different paths.

The `Environment` tracks boolean `Value` objects with an `AtomicBoolValue` leaf type. The `DataflowAnalysisContext` maintains a conjunction of `Formula` constraints that relate these atoms. Before each transfer step, the solver checks which atoms are provably true or false given the accumulated constraints, enabling the transfer function to use `Env.proves(BoolValue)` to query proven facts rather than being forced to assume unknown values at join points.

Noreturn destructor support (added 2023): When a CFG block ends with a destructor call on a type whose destructor is marked `[[noreturn]]` (e.g., `std::terminate`-calling destructors), the framework marks the block's post-state as unreachable. The fixpoint iterator skips unreachable blocks at join points, preventing them from contaminating the join result with their bottom-of-lattice states. This is implemented via `Environment::markUnreachable()` and checked in `DataflowAnalysis::runAnalysis()` before calling `join()`.

### 45.14.5 Built-in Analysis: `UncheckedOptionalAccessModel`

The canonical production use of the FS framework is `UncheckedOptionalAccessModel` (
[`clang/include/clang/Analysis/FlowSensitive/Models/UncheckedOptionalAccessModel.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Analysis/FlowSensitive/Models/UncheckedOptionalAccessModel.h)),
which detects calls to `optional::value()` (and similar `operator*()` on `optional`) when the optional is known to be empty on the current path.

The lattice element is a map from `StorageLocation*` to `OptionalModel::State` (one of `{Unknown, NonEmpty, Empty}`). The transfer function updates this map on assignments, returns from `make_optional`, and `has_value()` branch conditions. The SAT solver propagates the branch condition's effect through the `Environment`: after an `if (opt.has_value())` true branch, `opt` is constrained to `NonEmpty` without requiring the check to be a direct predecessor in the CFG.

Usage via clang-tidy:
```bash
clang-tidy --checks='clang-analyzer-optin.cplusplus.UninitializedObject' \
  # or, for optional:
clang-tidy --checks='bugprone-unchecked-optional-access' \
  -p build/ src/foo.cpp
```

The clang-tidy check `bugprone-unchecked-optional-access` wraps `UncheckedOptionalAccessModel` and handles both `std::optional`, `absl::optional`, and `llvm::Optional` through a configurable type-name list.

### 45.14.6 Writing a Custom FlowSensitive Analysis

A null-check analysis that tracks which pointer-typed variables are known non-null illustrates the full API:

```cpp
#include "clang/Analysis/FlowSensitive/DataflowAnalysis.h"
#include "clang/Analysis/FlowSensitive/DataflowEnvironment.h"

using namespace clang;
using namespace dataflow;

// Lattice: per-variable nullable state
struct NullLattice {
  enum class State { Unknown, NonNull, Null };
  llvm::DenseMap<const VarDecl*, State> VarStates;

  LatticeJoinEffect join(const NullLattice& Other) {
    bool Changed = false;
    for (auto& [VD, S] : Other.VarStates) {
      auto [It, Inserted] = VarStates.try_emplace(VD, S);
      if (!Inserted && It->second != S) {
        It->second = State::Unknown;  // merge: unknown if paths disagree
        Changed = true;
      }
    }
    return Changed ? LatticeJoinEffect::Changed : LatticeJoinEffect::Unchanged;
  }
};

class NullCheckAnalysis
    : public DataflowAnalysis<NullCheckAnalysis, NullLattice> {
public:
  explicit NullCheckAnalysis(ASTContext &Ctx)
      : DataflowAnalysis<NullCheckAnalysis, NullLattice>(Ctx) {}

  static NullLattice initialElement() { return NullLattice{}; }

  void transfer(const CFGElement &Elt, NullLattice &State,
                Environment &Env) {
    if (auto S = Elt.getAs<CFGStmt>()) {
      const Stmt *St = S->getStmt();

      // Track null assignments: ptr = nullptr;
      if (const auto *BO = dyn_cast<BinaryOperator>(St)) {
        if (BO->getOpcode() == BO_Assign) {
          if (const auto *DRE =
                  dyn_cast<DeclRefExpr>(BO->getLHS()->IgnoreParenImpCasts()))
            if (const auto *VD = dyn_cast<VarDecl>(DRE->getDecl())) {
              if (BO->getRHS()->IgnoreParenImpCasts()->isNullPointerConstant(
                      getASTContext(), Expr::NPC_ValueDependentIsNull)
                      != Expr::NPCK_NotNull)
                State.VarStates[VD] = NullLattice::State::Null;
              else
                State.VarStates[VD] = NullLattice::State::NonNull;
            }
        }
      }
    }
  }
};

// Running the analysis:
void runNullCheck(const FunctionDecl *FD, ASTContext &Ctx) {
  auto *CFG = Ctx.getAnalysis<CFGStmtMap>(); // simplified
  DataflowAnalysisContext DACtx(std::make_unique<WatchedLiteralsSolver>());
  auto Result = runDataflowAnalysis<NullCheckAnalysis>(
      *CFG, NullCheckAnalysis(Ctx), Environment(DACtx, *FD));
  if (auto Err = Result.takeError()) {
    llvm::errs() << "Analysis failed: " << Err << "\n";
    return;
  }
  // Inspect per-block lattice results...
}
```

`runDataflowAnalysis` returns `llvm::Expected<std::vector<std::optional<DataflowAnalysisState<LatticeT>>>>` — a vector indexed by CFG block ID, where each element holds the post-state at that block (or `std::nullopt` for unreachable blocks).

### 45.14.7 When to Choose FlowSensitive vs. Path-Sensitive

| Property | Path-Sensitive (`ExplodedGraph`) | FlowSensitive (`DataflowAnalysis`) |
|----------|----------------------------------|-------------------------------------|
| Cost | Exponential worst-case | Polynomial (O(N × H)) |
| Precision | High (tracks per-path constraints) | Moderate (joins at merge points) |
| Loops | Widening via `LoopWidening.cpp` | Widening via `widenAndJoin()` |
| Memory model | Full `MemRegion` hierarchy | `StorageLocation` (coarser) |
| Inter-procedural | Inlining via `CallEnter`/`CallExitEnd` | Limited context-sensitivity option |
| SAT integration | Z3 (optional) | `WatchedLiteralsSolver` (always) |
| Best for | Use-after-free, double-free, taint | Optional access, null-safety, type-state |

Choose FlowSensitive when: (a) the property is a per-variable type-state (e.g., initialized/uninitialized, locked/unlocked, null/non-null) that must hold on all paths; (b) scalability to large functions is required; or (c) the checker must integrate with clangd's background analysis, which restricts per-file analysis time.

Choose path-sensitive when: (a) the property requires tracking a precise sequence of events along a specific path (use-after-free in one error path, correct in another); (b) the bug is only present on a subset of paths and merging states would eliminate the signal; or (c) the checker needs the full `MemRegion`/`SVal` machinery for heap modeling.

---

## Chapter 45 Summary

- The analyzer is a path-sensitive symbolic execution engine built as a Clang `FrontendAction`; `AnalysisConsumer` creates an `AnalysisManager` → `ExprEngine` → `CoreEngine` pipeline for each analyzed function.
- `ProgramState` is an immutable triple of `(Environment, Store, GenericDataMap)`; all state transitions return new `ProgramStateRef` values, with structural sharing via `FoldingSet` interning.
- `SVal` is a discriminated union of `UndefinedVal`, `UnknownVal`, `NonLoc` (symbolic integers), and `Loc` (memory addresses); `SymbolManager` interns `SymExpr` trees representing relationships between unknown values.
- `MemRegion` models memory as a typed-region tree with spaces for stack, heap, and globals; `RegionStoreManager` maintains a persistent region→SVal map with `LazyCompoundVal` deferring aggregate copies.
- The `ExplodedGraph` pairs `ProgramPoint`s with `ProgramState`s; `CoreEngine` drives a pluggable `WorkList` and prunes by graph folding; loop widening prevents infinite unrolling.
- Checkers register CRTP callbacks (`check::PreStmt`, `check::PostCall`, `check::DeadSymbols`, etc.) via `CheckerManager`; `CheckerContext` exposes `generateNonFatalErrorNode()`, `generateSink()`, and `emitReport()`.
- Checker-private state uses `REGISTER_MAP_WITH_PROGRAMSTATE` / `REGISTER_TRAIT_WITH_PROGRAMSTATE`, storing typed data in the `ProgramState::GenericDataMap` under a static key.
- `CallEvent` abstracts over C calls, C++ member calls, constructors, destructors, allocators, blocks, and ObjC messages; inlining pushes `CallEnter` nodes with new `StackFrameContext`s.
- `BugReporterVisitor` adds path notes; `HTMLDiagnostics`, `PlistDiagnostics`, and `SarifDiagnostics` render the reconstructed `PathDiagnostic`.
- Taint analysis uses `taint::addTaint()`/`isTainted()` with configurable YAML source/propagator/sink rules; `TaintTagType` supports multiple independent taint domains.
- Z3 integration operates either as a full constraint solver (`-analyzer-constraints=z3`) or as a post-hoc refutation filter (`crosscheck-with-z3=true`), eliminating false positives from range-solver imprecision at the cost of solver latency.
- Scalability levers include `FunctionSummariesTy` caching, the NoRedundancy inlining heuristic, node-count limits, and loop widening; CTU analysis extends the reach across translation-unit boundaries.
- Clang 22's lifetime safety analysis provides five production checkers — UseAfterReturn, UseAfterMove, NonTriviallyDestructedTemporaries, FieldReferenceToOutOfScopeLocal, and LifetimeboundViolation — enabled under `-Wdangling` and `-Wreturn-local-addr`; `[[clang::lifetimebound]]` annotations propagate lifetime constraints to callers without full borrow-checking overhead.
- The FlowSensitive framework (`clang/Analysis/FlowSensitive/`) provides a polynomial-cost alternative to the path-sensitive engine, using `DataflowAnalysis<LatticeT>` with a join-semilattice, `WatchedLiteralsSolver` for boolean constraint propagation, and `UncheckedOptionalAccessModel` as its canonical production analysis; widening via `widenAndJoin()` ensures termination on loops with infinite-height lattices.

---

## Cross-References

- [Chapter 32 — The Parser and Sema](../part-05-clang-frontend/ch32-parser-sema.md) — Clang AST nodes consumed by `check::ASTDecl` and `check::ASTCodeBody` checkers
- [Chapter 36 — The Clang AST in Depth](../part-05-clang-frontend/ch36-clang-ast-in-depth.md) — `Stmt`, `Expr`, `Decl` hierarchies traversed by the path-sensitive engine
- [Chapter 46 — libtooling and AST Matchers](ch46-libtooling-ast-matchers.md) — Building standalone tools that embed the analyzer as a library
- [Chapter 47 — clangd, clang-tidy, clang-format, clang-refactor](ch47-clangd-clang-tidy-format-refactor.md) — `clang-tidy` consuming `clang-analyzer-*` checks; `scan-build` orchestration

## Reference Links

- [`clang/include/clang/StaticAnalyzer/Core/PathSensitive/ProgramState.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/ProgramState.h)
- [`clang/include/clang/StaticAnalyzer/Core/PathSensitive/SVals.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/SVals.h)
- [`clang/include/clang/StaticAnalyzer/Core/PathSensitive/MemRegion.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/MemRegion.h)
- [`clang/include/clang/StaticAnalyzer/Core/PathSensitive/ExplodedGraph.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/ExplodedGraph.h)
- [`clang/include/clang/StaticAnalyzer/Core/PathSensitive/CoreEngine.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/CoreEngine.h)
- [`clang/include/clang/StaticAnalyzer/Core/Checker.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/Checker.h)
- [`clang/include/clang/StaticAnalyzer/Core/CheckerManager.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/CheckerManager.h)
- [`clang/include/clang/StaticAnalyzer/Core/PathSensitive/CheckerContext.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/CheckerContext.h)
- [`clang/include/clang/StaticAnalyzer/Core/PathSensitive/CallEvent.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/CallEvent.h)
- [`clang/include/clang/StaticAnalyzer/Core/BugReporter/BugReporter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/BugReporter/BugReporter.h)
- [`clang/include/clang/StaticAnalyzer/Core/BugReporter/BugReporterVisitors.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/BugReporter/BugReporterVisitors.h)
- [`clang/include/clang/StaticAnalyzer/Checkers/Taint.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Checkers/Taint.h)
- [`clang/include/clang/StaticAnalyzer/Core/PathSensitive/ProgramStateTrait.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/ProgramStateTrait.h)
- [`clang/include/clang/StaticAnalyzer/Core/PathSensitive/FunctionSummary.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/StaticAnalyzer/Core/PathSensitive/FunctionSummary.h)
- [Reps, Horwitz, Sagiv — "Precise interprocedural dataflow analysis via graph reachability" (1995)](http://portal.acm.org/citation.cfm?id=199462) — theoretical foundation for ExplodedGraph
- [Kremenek, Engler — "Z3: An Efficient SMT Solver" applied to Clang SA cross-checking](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/StaticAnalyzer/Core/Z3ConstraintManager.cpp)
- [`clang/include/clang/Analysis/FlowSensitive/DataflowAnalysis.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Analysis/FlowSensitive/DataflowAnalysis.h)
- [`clang/include/clang/Analysis/FlowSensitive/DataflowAnalysisContext.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Analysis/FlowSensitive/DataflowAnalysisContext.h)
- [`clang/include/clang/Analysis/FlowSensitive/DataflowEnvironment.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Analysis/FlowSensitive/DataflowEnvironment.h)
- [`clang/include/clang/Analysis/FlowSensitive/WatchedLiteralsSolver.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Analysis/FlowSensitive/WatchedLiteralsSolver.h)
- [`clang/include/clang/Analysis/FlowSensitive/Models/UncheckedOptionalAccessModel.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Analysis/FlowSensitive/Models/UncheckedOptionalAccessModel.h)
- [Herb Sutter — "Lifetime Safety: Preventing Common Dangling" (2018 ISO C++ proposal P1179)](https://herbsutter.com/2018/09/20/lifetime-profile-v1-0-posted/)
- [`clang/include/clang/Basic/AttrDocs.td` — `lifetimebound` entry](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/AttrDocs.td)


---

@copyright jreuben11
