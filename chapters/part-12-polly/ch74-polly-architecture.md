# Chapter 74 — Polly Architecture

*Part XII — Polly*

Polly is LLVM's polyhedral loop optimizer. It takes an LLVM IR function, extracts the largest affine loop nests (SCoPs) it can find, applies the scheduling and code generation algorithms developed in Part XI, and replaces the original loop code with the optimized version. This chapter covers Polly's architecture: how it detects and represents SCoPs from LLVM IR, how it integrates into the LLVM pass pipeline, the role of ISL inside Polly, and how Polly's design compares to standalone polyhedral tools like Pluto.

## Table of Contents

- [74.1 Overview](#741-overview)
- [74.2 The Polly Pass Pipeline](#742-the-polly-pass-pipeline)
- [74.3 ScopDetection](#743-scopdetection)
  - [74.3.1 Detection Conditions](#7431-detection-conditions)
  - [74.3.2 ScalarEvolution Integration](#7432-scalarevolution-integration)
  - [74.3.3 Non-Affine Extensions](#7433-non-affine-extensions)
- [74.4 ScopInfo: The Polyhedral Representation](#744-scopinfo-the-polyhedral-representation)
  - [74.4.1 The Scop Class](#7441-the-scop-class)
  - [74.4.2 ScopStmt: Statement Representation](#7442-scopstmt-statement-representation)
  - [74.4.3 MemoryAccess: Access Functions](#7443-memoryaccess-access-functions)
  - [74.4.4 Modeling Array Layout](#7444-modeling-array-layout)
- [74.5 Dependence Analysis](#745-dependence-analysis)
  - [74.5.1 Validity vs Proximity Constraints](#7451-validity-vs-proximity-constraints)
- [74.6 Schedule Optimization](#746-schedule-optimization)
  - [74.6.1 Polly-Specific Optimizations](#7461-polly-specific-optimizations)
- [74.7 Code Generation](#747-code-generation)
  - [74.7.1 Value Substitution](#7471-value-substitution)
- [74.8 Comparison to Pluto](#748-comparison-to-pluto)
- [Chapter Summary](#chapter-summary)

---

## 74.1 Overview

Polly is an out-of-tree pass that ships as part of the LLVM monorepo under `polly/`. It is enabled with `LLVM_ENABLE_POLLY=ON` at cmake time and accessed via:

```bash
# Using Polly via clang:
clang -O3 -mllvm -polly input.c -o output

# Using Polly via opt:
opt -load-pass-plugin /path/to/LLVMPolly.so \
    -passes='polly-detect,polly-scops,polly-codegen' \
    -S input.ll

# Check which loops Polly detected:
clang -O3 -mllvm -polly -mllvm -polly-report input.c
```

Polly operates entirely on LLVM IR — it never sees the original C/C++ source. This means it must reconstruct the SCoP structure from the SSA graph, which is harder than working directly on a high-level AST.

## 74.2 The Polly Pass Pipeline

Polly inserts a sequence of passes into the LLVM optimization pipeline, typically after the inliner and scalar optimizations:

```
LLVM IR (after O1/O2 scalar opts)
  │
  ├── [Polly] ScopDetection         — find SCoP regions
  ├── [Polly] ScopInfo              — build polyhedral representation
  ├── [Polly] Dependences           — compute dependence relations
  ├── [Polly] ScheduleOptimizer    — apply Pluto/ISL scheduling
  ├── [Polly] CodeGeneration        — emit new LLVM IR loop nests
  └── [Polly] CleanupPasses         — canonicalize generated code
  │
  └── LLVM IR (optimized loop nest) → continues to backend
```

Each pass is a standard LLVM RegionPass or FunctionPass.

## 74.3 ScopDetection

`ScopDetectionPass` (in `polly/lib/Analysis/ScopDetection.cpp`) scans a function's region tree for maximal **regions** (connected subgraphs of the CFG with single-entry, single-exit) that satisfy the SCoP constraints.

### 74.3.1 Detection Conditions

A region qualifies as a SCoP if:

1. **All branch conditions are affine**: every conditional branch within the region depends only on loop induction variables and symbolic parameters, not on array values.
2. **All memory accesses are affine**: every load and store address is an affine function of surrounding loop IVs and symbolic parameters.
3. **All loops have affine bounds**: loop trip counts must be computable as affine functions of parameters.
4. **No function calls with side effects**: calls to `malloc`, `free`, or functions with unknown memory effects are disqualifying (unless annotated as pure).

```cpp
// In ScopDetection.cpp:
bool ScopDetection::isValidMemoryAccess(MemoryAccess &MA,
                                         DetectionContext &Context) const {
  // Check that the access function is affine in the surrounding IVs
  if (!SE->isSCEVable(MA.getType())) return false;
  const SCEV *AccessFunction = SE->getSCEV(MA.getPointerOperand());
  if (!isAffine(AccessFunction, Context.CurRegion, Context)) return false;
  return true;
}
```

### 74.3.2 ScalarEvolution Integration

The critical enabling technology for SCoP detection is **ScalarEvolution** (`ScalarEvolutionAnalysis`, Chapter 61): LLVM's loop-trip-count and induction-variable analysis pass. ScalarEvolution expresses loop induction variables as `SCEVAddRecExpr` (affine recurrences), which Polly can directly map to the polyhedral loop variable representation.

```
// LLVM IR loop:
%i = phi i64 [ 0, %entry ], [ %i.next, %loop ]
%i.next = add nsw i64 %i, 1
// ScalarEvolution: %i = {0, +, 1}<%loop>   — start 0, step 1

// Array access:
%addr = getelementptr double, ptr %A, i64 %i
// ScalarEvolution: %addr = A + {0, +, 8}<%loop>   — affine in %i
```

Polly queries `SE->getBackedgeTakenCount` for loop bounds and `SE->getSCEV` for access function analysis. Non-affine SCEV expressions (containing `SCEVUnknown`, `SCEVMulExpr` with non-constant factors, or `SCEVUDivExpr`) cause detection to fail.

### 74.3.3 Non-Affine Extensions

Polly's `NonAffineSubRegions` extension allows regions containing non-affine code to be modeled with **non-affine subregions**: the non-affine code is treated as an opaque black box with conservative memory effects:

```c
for (int i = 0; i < N; i++) {
  // Non-affine subregion: call to printf (side effects)
  if (A[i] > threshold)    // data-dependent condition — non-affine
    printf("found: %d\n", A[i]);
  B[i] = A[i] * 2.0;      // can still be optimized
}
```

The non-affine subregion boundary defines where Polly's analysis can and cannot reason. Memory accesses inside the subregion are modeled conservatively (may read/write any memory).

## 74.4 ScopInfo: The Polyhedral Representation

`ScopInfoPass` (in `polly/lib/Analysis/ScopInfo.cpp`) converts the detected SCoP region into a full polyhedral representation: `Scop` objects containing `ScopStmt` objects with `MemoryAccess` objects.

### 74.4.1 The Scop Class

```cpp
class Scop {
  // Context: the set of valid parameter values
  isl_set *Context;            // e.g., { [N, M] : N >= 1, M >= 1 }

  // All statements in this SCoP
  SmallVector<ScopStmt *, 8> Stmts;

  // The ISL context
  isl_ctx *IslCtx;

  // Assumption set: conditions under which the SCoP is valid
  isl_set *Assumptions;

  // Schedule: the current polyhedral schedule (may be modified by ScheduleOptimizer)
  isl_schedule *Schedule;
};
```

### 74.4.2 ScopStmt: Statement Representation

Each `ScopStmt` corresponds to one basic block (or a maximal sequence of non-branching instructions) inside the SCoP:

```cpp
class ScopStmt {
  // Iteration domain: the set of valid iteration vectors
  isl_set *Domain;            // { [i, j] : 0 <= i < N, 0 <= j < M }

  // Memory accesses of this statement
  SmallVector<MemoryAccess *, 8> MemAccs;

  // The original basic block(s) this statement covers
  BasicBlock *BB;

  // Schedule in the current Scop::Schedule
  isl_map *Schedule;          // maps iteration to schedule space
};
```

### 74.4.3 MemoryAccess: Access Functions

Each `MemoryAccess` records one array read or write:

```cpp
class MemoryAccess {
  // Access type: READ or WRITE (MK_Read / MK_Write)
  AccessType AccType;

  // The access function as an ISL map:
  // { StmtDomain -> MemRefSpace }  e.g., { [i,j] -> MemRef_A[i, j] }
  isl_map *AccessRelation;

  // The original LLVM load or store instruction
  Instruction *AccessInstruction;
};
```

Building the access relation requires mapping from LLVM's GEP (GetElementPtr) computation back to an affine expression over the loop IVs — this is done by querying ScalarEvolution and then translating the resulting SCEV into an ISL affine expression.

### 74.4.4 Modeling Array Layout

Polly must understand the multi-dimensional array layout to build correct access functions. For a C array `double A[N][M]`, the access `A[i][j]` compiles to:

```llvm
%addr = getelementptr double, ptr %A, i64 %idx
; where %idx = i * M + j  (row-major layout)
```

Polly delinearizes this flat index `i * M + j` back into the multi-dimensional index `[i, j]` using ScalarEvolution's divisibility analysis. This delinearization is critical: without it, Polly would treat `A[i*M+j]` as a one-dimensional access and miss all 2D structure.

## 74.5 Dependence Analysis

`DependencesAnalysis` (in `polly/lib/Analysis/Dependences.cpp`) computes all RAW, WAR, WAW, and RAR dependences between ScopStmts using ISL's exact dependence computation:

```cpp
// Compute flow dependences (RAW):
Dependences::computeFlow(
    AllWrites,   // union_map of all write access relations
    AllReads,    // union_map of all read access relations
    /*Kills=*/nullptr,
    ScheduleMap, // current schedule to determine ordering
    &FlowDependences,  // output: RAW dependences
    &AntiDependences,  // output: WAR dependences
    &OutputDependences // output: WAW dependences
);
```

ISL's `isl_union_map_compute_flow` performs exact dependence computation: for each read of a memory location, it finds the most recent write to the same location in program order. The result is a set of `(source_stmt, destination_stmt, iteration_pair)` relations.

### 74.5.1 Validity vs Proximity Constraints

Polly distinguishes two categories of dependence constraints for the scheduler:

- **Validity** (`ValidDeps`): dependences that *must* be preserved. The schedule must respect these.
- **Proximity** (`ProximityDeps`): a subset of valid dependences that the scheduler should try to minimize (for data locality). Typically: RAW dependences between consecutive statements.

```cpp
isl_union_map *ValidDeps = FlowDependences;  // must respect
isl_union_map *ProximityDeps = FlowDependences;  // optimize distance
```

The ISL scheduler receives both via `isl_schedule_constraints_set_validity` and `isl_schedule_constraints_set_proximity`.

## 74.6 Schedule Optimization

`ScheduleOptimizerPass` (in `polly/lib/Transform/ScheduleOptimizer.cpp`) applies the ISL scheduler (Pluto-style, Chapter 72) to compute the optimal polyhedral schedule, then applies tiling and vectorization marks:

```cpp
// Apply Polly's default schedule optimization:
isl_schedule *optimizeSchedule(Scop &S) {
  isl_schedule_constraints *SC = buildScheduleConstraints(S);
  isl_schedule *Sched = isl_schedule_constraints_compute_schedule(SC);

  // Apply loop tiling:
  if (PollyTiling) {
    isl_multi_val *TileSizes = computeTileSizes(S);
    Sched = applyTiling(Sched, TileSizes);
  }

  // Mark parallel and vector dimensions:
  Sched = markParallelDimensions(Sched, S);
  Sched = markVectorizableDimensions(Sched, S);
  return Sched;
}
```

### 74.6.1 Polly-Specific Optimizations

Beyond the standard ISL schedule, Polly applies:

- **Loop tiling** (`-polly-tiling`): tile all permutable dimensions with user-specified or auto-tuned sizes.
- **Loop vectorization** (`-polly-vectorizer`): mark the innermost parallel dimension for SIMD vectorization.
- **Loop parallelization** (`-polly-parallel`): mark outer parallel dimensions for OpenMP `parallel for`.
- **Pattern-based matmul detection** (`-polly-opt-isl`): detect matrix multiplication kernels and apply a fixed high-performance schedule (bypassing the generic ISL scheduler).

## 74.7 Code Generation

`CodeGenerationPass` (in `polly/lib/CodeGen/CodeGeneration.cpp`) converts the optimized polyhedral schedule back to LLVM IR using ISL's AST generator:

```cpp
// Code generation in Polly:
void generateCode(Scop &S) {
  // Build ISL AST from schedule:
  isl_ast_build *Build = isl_ast_build_from_context(S.getContext());
  isl_ast_node *Ast = isl_ast_build_node_from_schedule(Build, S.getSchedule());

  // Traverse AST and emit LLVM IR:
  IslNodeBuilder Builder(DT, LI, SE, DL, RI, P);
  Builder.create(Ast);

  // The original SCoP region is replaced by the Builder's output.
}
```

The `IslNodeBuilder` walks the ISL AST recursively:
- `isl_ast_node_for` → create a new basic block for the loop header, latch, and body; create the loop IV via PHI; compute bounds via `isl_ast_expr_to_llvm_value`.
- `isl_ast_node_if` → create a conditional branch; guard the ScopStmt basic blocks.
- `isl_ast_node_user` → copy the original LLVM IR instructions from the ScopStmt's basic block, substituting the new loop IVs for the original (via a value-map).

### 74.7.1 Value Substitution

The key challenge in code generation is mapping from the ISL schedule variables back to the original LLVM IV values. Polly maintains a `ValueMapT` that maps each original LLVM SSA value to its new counterpart after schedule transformation:

```cpp
// For each ScopStmt execution (user node in the ISL AST):
void IslNodeBuilder::createUser(isl_ast_node *User) {
  // Compute the new iteration vector from the current AST loop variables:
  isl_ast_expr *ScheduleExpr = isl_ast_node_user_get_expr(User);

  // Map schedule values back to statement iteration values:
  ValueMapT VMap = computeIteratorMap(ScheduleExpr, Stmt);

  // Copy statement's basic block, substituting mapped values:
  copyBB(Stmt, VMap);
}
```

## 74.8 Comparison to Pluto

Polly and the standalone Pluto tool (Bondhugula et al.) implement the same polyhedral algorithms, but with different integration points:

| Aspect | Polly | Pluto (standalone) |
|--------|-------|-------------------|
| Input | LLVM IR | C source code (via clan) |
| Output | LLVM IR | C source code |
| SCoP detection | ScalarEvolution-based | Clan (C-to-SCoP parser) |
| Access recovery | Delinearization from GEP | Direct from AST |
| Scheduling | ISL (Pluto-style) | PIP + custom LP solver |
| Code generation | ISL AST → LLVM IR | CLooG → C |
| OpenMP support | Yes (LLVM OpenMP) | Yes (OpenMP C pragmas) |
| GPU support | Experimental (Polly-ACC) | Via PPCG integration |

The key advantage of Polly over standalone Pluto is seamless LLVM integration: Polly's output is directly LLVM IR, which the backend then lowers to native machine code with full access to the LLVM backend pipeline (vectorization, register allocation, instruction scheduling). Pluto's C output requires a second compilation step and loses the SSA form that LLVM's optimizer relies on.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **NewPassManager full migration**: Polly's remaining RegionPass infrastructure is being ported to the LLVM New Pass Manager (NPM); tracking issue on llvm-project GitHub and Polly ML threads; expect all legacy `addRequired<>` pass dependencies replaced by explicit analysis requests via `AM.getResult<>`.
- **ISL 0.27 upgrade**: Polly bundles ISL; the upstream ISL project regularly releases fixes to `isl_schedule_constraints_compute_schedule` for edge cases in Feautrier/Pluto scheduling—Polly's vendored copy lags; a periodic sync effort (tracked under `polly/lib/External/isl`) is in progress.
- **Delinearization robustness improvements**: PRs fixing false-negative SCoP detection due to delinearization failures on multi-dimensional arrays with parametric inner dimensions (`isl_pw_aff` cannot represent them); patches under review on Phabricator/GitHub CI.
- **`-polly-use-runtime-alias-checks` stabilization**: Runtime alias checks allow Polly to optimize loops where pointer aliasing cannot be disproven statically; known miscompiles in the presence of restrict-qualified pointers under active fixup (see llvm-dev thread "Polly alias check soundness", March 2026).

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Polly-ACC (GPU offload) revival**: The `polly/lib/CodeGen/PPCGCodeGeneration.cpp` GPU backend has been dormant since ~2020; community interest in reviving it as an alternative/complement to OpenMP target offload; requires integrating with LLVM's new `TargetTransformInfo`-aware offload decision logic and the `OpenMPIRBuilder`.
- **Affine dialect round-trip**: MLIR's `affine` dialect (Chapter 93) represents the same mathematical objects as Polly's ISL-based IR; an RFC has been floated to allow Polly to lower its `Scop` representation to MLIR affine dialect for joint optimization with MLIR passes, then lower back to LLVM IR—enabling MLIR's `affine-loop-fusion` and `affine-parallelize` to complement Polly's Pluto scheduler.
- **Auto-tuning tile size selection**: Polly's default tile sizes (32×32 or parameter-derived) are known to be suboptimal on modern cache hierarchies; integration with an ML-guided tile-size auto-tuner (similar to TVM's AutoScheduler/Ansor) has been proposed; requires adding a profiling feedback loop into Polly's `ScheduleOptimizerPass`.
- **Pattern-based GEMM/CONV detection expansion**: Current pattern matcher in `ScheduleOptimizer.cpp` recognizes only matrix multiplication; plans to extend to convolution (NHWC/NCHW) and batched-GEMM patterns, particularly for workloads emitted by `torch-mlir` or `stablehlo` lowering paths that reach the LLVM backend.

### 5-Year Horizon (Long-Term, by ~2031)

- **Full Polly/MLIR convergence**: Long-term trajectory is for Polly's polyhedral analysis to be expressed natively in MLIR, replacing the ISL C API with MLIR's integer set library (`mlir/include/mlir/Analysis/Presburger/`); this would unify Polly's `isl_set`/`isl_map` world with MLIR's `FlatAffineConstraints` and allow a single polyhedra-aware optimizer shared across the LLVM/MLIR toolchain.
- **Sparse polyhedral scheduling**: Extension of Polly's dense-array SCoP model to handle sparse tensor formats (CSR, COO, ELLPACK); requires integrating sparse iteration space representations (as in TACO/Finch) into the ISL schedule, potentially via `sparse_tensor` dialect lowering meeting Polly's code generator.
- **Verified polyhedral transformations**: Applying Alive2-style semantic equivalence checking (Chapter 165) to verify that Polly's schedule transformations and code generation produce semantically identical IR; requires encoding the polyhedral dependence validity proof into a form checkable by SMT solvers (Z3/Bitwuzla), building on ongoing work in formal verification of loop transformations.

---

## Chapter Summary

- Polly integrates into the LLVM pass pipeline as a sequence of passes: `ScopDetection`, `ScopInfo`, `Dependences`, `ScheduleOptimizer`, and `CodeGeneration`.
- **ScopDetection** uses ScalarEvolution to verify that all loop bounds and array access functions within a region are affine in the surrounding loop IVs and symbolic parameters.
- **ScopInfo** builds the polyhedral representation: `Scop` containing `ScopStmt` objects with `MemoryAccess` records, all expressed as ISL sets and maps; array delinearization recovers multi-dimensional access structure from flat GEP addresses.
- **DependencesAnalysis** computes exact RAW/WAR/WAW dependences using ISL's `isl_union_map_compute_flow`, producing validity and proximity constraints for the scheduler.
- **ScheduleOptimizerPass** applies the ISL Pluto-style scheduler, then tiles permutable dimensions and marks parallel/vectorizable dimensions for OpenMP/SIMD code generation.
- **CodeGenerationPass** uses ISL's `isl_ast_build` to generate an AST from the optimized schedule and walks it to emit LLVM IR, substituting new IV values for original SSA values via a value map.
- Polly's advantage over standalone polyhedral tools (Pluto, PoCC) is direct LLVM IR I/O: the output feeds immediately into the LLVM backend with no intermediate C compilation step.


---

@copyright jreuben11
