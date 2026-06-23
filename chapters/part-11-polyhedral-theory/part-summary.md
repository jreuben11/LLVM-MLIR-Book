# Part XI — Polyhedral Theory — Part Summary

*This part establishes the complete mathematical foundation of the polyhedral model — from convex geometry and integer arithmetic through the SCoP abstraction, scheduling algorithms, and concrete loop code generation — equipping readers to understand and extend Polly, MLIR's Affine dialect, and every production affine loop optimizer.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 70 | Foundations: Polyhedra and Integer Programming | ISL, Presburger arithmetic, Omega test, ILP |
| 71 | The Polyhedral Model | SCoPs, iteration spaces, dependences, schedule legality |
| 72 | Scheduling Algorithms | Feautrier, Lim-Lam, Pluto, ISL scheduler |
| 73 | Code Generation from Polyhedral Schedules | CLooG/ISL AST, tiling, OpenMP, SIMD, GPU |

## Part Overview

Part XI builds the full intellectual stack required to understand, implement, and extend polyhedral loop optimizers. The arc is deliberate: Chapter 70 lays the pure mathematical foundation, Chapter 71 applies it to programs, Chapter 72 solves the optimization problem over the resulting spaces, and Chapter 73 materializes the abstract solution back into executable code. Together they form a closed loop from algebraic theory to machine instruction.

Chapter 70 establishes the three pillars that underpin everything downstream. Convex polyhedra — sets of integer points satisfying systems of linear inequalities — are the representation for iteration spaces. Integer lattice theory (Hermite Normal Form, Smith Normal Form) provides the algebraic tools for testing integer solvability and canonicalizing transformation matrices. Presburger arithmetic — first-order integer arithmetic without multiplication — is the logical language in which every compiler question about loop nests is expressed: "are these two array accesses dependent?", "does this loop execute at all?", "what is the loop trip count?" The chapter covers all three main decision procedures (Cooper's quantifier elimination, the Omega test, Parametric Integer Programming) and grounds them in ISL, the C library that implements these operations for Polly and MLIR's Affine dialect.

Chapter 71 translates the mathematics into a programming model. A Static Control Part (SCoP) is a program region where every loop bound and array index is an affine function of the surrounding induction variables and symbolic parameters. Within a SCoP, each statement's iteration space is a parametric polytope, each array reference is an affine access relation, and the correctness constraints between statements are dependence polyhedra. The chapter defines all four dependence types (RAW, WAR, WAW, RAR), shows how dependence pairs are computed using ISL's `isl_union_map_compute_flow`, and derives the affine schedule space — the convex polyhedron in schedule-coefficient space that contains precisely the legal schedules. Every classical loop transformation (interchange, tiling, skewing, fusion, distribution) is revealed to be a change of coordinates within this space.

Chapter 72 addresses the central optimization problem: find the *best* point in the affine schedule space. Three foundational algorithms are derived from first principles. Feautrier's algorithm (1992) finds the lexicographically minimal multidimensional affine schedule using Parametric Integer Programming, proceeding dimension by dimension and is provably complete. Lim-Lam affine partitioning (1997) targets maximum parallelism by finding the null space of the dependence matrix, recovering wavefront schedules when perfect parallelism is impossible. The Pluto algorithm (Bondhugula et al., PLDI 2008) is the modern standard: it applies the Farkas lemma to convert dependence legality into linear constraints on schedule coefficients, then minimizes dependence distances via ILP, simultaneously maximizing tileability and parallelism. ISL's scheduler (`isl_schedule_constraints_compute_schedule`) re-implements Pluto-style optimization using exact Presburger arithmetic and produces schedule trees — band nodes annotated with tileable/coincident flags — as output.

Chapter 73 closes the loop by generating executable code from the abstract polyhedral schedule. ISL's AST generator (`isl_ast_build`) converts schedule trees into structured AST nodes (for/if/block/mark/user). Loop bounds are derived by Fourier-Motzkin elimination applied to each dimension's projection; the result is max/min expressions that may involve floor and ceiling functions for tiled loops. The chapter covers all major tiling shapes (hyperrectangular, parallelogram, diamond, time-skewed), the ISL schedule-tree mechanism that distinguishes barrier-free set nodes from barrier-required sequence nodes for OpenMP code generation, SIMD vectorization mapping via MLIR's vector dialect, and GPU code generation via PPCG's two-level parallelism hierarchy mapping to CUDA grid and block dimensions. Unroll-and-jam is shown to be a composition of strip-mining and interchange in the polyhedral model.

## Key Concepts Introduced

- **Convex polyhedron (H-representation)**: the set P = {x ∈ ℝⁿ | Ax ≤ b}; the representation of iteration spaces, dependence domains, and schedule feasibility regions throughout the polyhedral model.
- **Farkas' lemma**: a duality result characterizing when linear constraints imply other linear constraints; the algebraic engine used by both Feautrier's algorithm and Pluto to convert "∀ iterations in the dependence polyhedron, the schedule difference is non-negative" into a *finite* system of linear constraints on schedule coefficients.
- **Presburger arithmetic**: first-order integer arithmetic with addition but not multiplication; decidable (in contrast to Peano arithmetic), and the exact logical language used to express every compiler question about affine loop nests.
- **Omega test**: Pugh's 1992 exact integer satisfiability procedure for Presburger formulas; polynomial-time for fixed loop depth, used inside ISL for dependence satisfiability queries.
- **ISL (Integer Set Library)**: Verdoolaege's C library implementing Presburger-formula sets and relations, parametric integer programming, Pluto-style scheduling, and AST generation; the algorithmic backbone of Polly and MLIR's Affine dialect.
- **Static Control Part (SCoP)**: a maximal program region where all loop bounds and array indices are affine functions of induction variables and symbolic parameters; the input domain of the polyhedral model and the unit of analysis for Polly and MLIR's Affine dialect.
- **Dependence polyhedron**: the set of all iteration pairs (I₁, I₂) from two statements such that I₁'s write to a memory location precedes I₂'s access to the same location; the constraint that every legal schedule must respect.
- **Affine schedule space**: the convex polyhedron in schedule-coefficient space whose interior contains precisely the legal affine schedules; found by applying the Farkas lemma to all dependence constraints.
- **Feautrier's algorithm**: the first complete polyhedral scheduling algorithm; finds the lexicographically minimal multidimensional affine schedule via Parametric Integer Programming, dimension by dimension.
- **Pluto algorithm**: Bondhugula et al.'s PLDI 2008 scheduler; minimizes dependence distances (maximizing tileability and parallelism) via an ILP per schedule dimension, using Farkas-derived legality constraints; implemented in ISL's default scheduler mode.
- **Permutability (tileability)**: the condition that all dependences have non-negative projection onto a set of loop dimensions; a permutable band of loops can be legally tiled for cache and parallelism optimization.
- **Band tree (schedule tree)**: ISL's hierarchical schedule representation; a tree of band nodes (groups of jointly-optimized schedule dimensions) with per-dimension tileable/coincident annotations, consumed by `isl_ast_build` for code generation.
- **Fourier-Motzkin elimination**: the variable-projection algorithm used in code generation to derive loop bounds from the schedule image; produces max/min expressions that become the loop increment bounds in generated C, LLVM IR, or MLIR.
- **Diamond tiling**: a tiling strategy for stencil computations that uses rhombus-shaped tiles in (time, space) dimensions, enabling concurrent start across wavefronts and efficient reuse of spatial data across time steps.
- **PPCG (Polyhedral Parallel Code Generator)**: Verdoolaege's standalone tool for GPU code generation from polyhedral schedules; maps the two outermost parallel schedule dimensions to CUDA grid and block dimensions, inserting shared memory tiling for data reuse.

## How This Part Fits the Book

Part XI is positioned after Part X (Analysis and Middle-End, Chapters 55–69), which introduces LLVM's scalar analysis infrastructure — ScalarEvolution, LoopInfo, MemorySSA, alias analysis — that Polly and MLIR use to detect and verify SCoP boundaries. Parts XII and XIII (Polly, Chapters 74–80, and LTO/Whole-Program Optimization, Chapters 81–88) build directly on this part: Chapter 74 uses the SCoP abstraction from Chapter 71, Chapters 75–78 apply the scheduling algorithms of Chapter 72, and Chapter 79 uses the code generation infrastructure of Chapter 73. Part XIX (MLIR Foundations, Chapters 120–128) and Part XX (In-Tree Dialects, Chapters 129–143) draw heavily on Chapters 70–71 for the Affine dialect's iteration space representation and the `mlir::presburger` analysis library.

## Cross-Part Dependencies

- **Ch 74 (Part XII — Polly)**: SCoP detection and representation in Polly directly implement Ch 71's SCoP definition; `ScopInfo.cpp` encodes iteration spaces as ISL sets and dependences as ISL maps.
- **Ch 75–78 (Part XII — Polly)**: Polly's scheduling and optimization passes implement Ch 72's Pluto/ISL scheduling; Polly's codegen implements Ch 73's ISL AST generator interface.
- **Ch 129–131 (Part XX — MLIR In-Tree Dialects, Affine dialect)**: MLIR's `affine.for`, `affine.if`, and `affine.load/store` operations operationalize the SCoP abstraction from Ch 71; the `mlir/lib/Analysis/Presburger/` library implements a subset of Ch 70's ISL functionality in-tree.
- **Ch 107 (Part XXIV — Verified Compilation)**: CompCert/Vellvm-style verified compilation requires machine-checked proofs that polyhedral schedules satisfy Farkas constraints; the theoretical foundations for such proofs rest directly on Ch 70's Presburger arithmetic decidability results.
- **Ch 66 (Part X — ML-guided Optimizations)**: ML-guided autotuning of tile sizes and schedule choices (autoschedulers, TVM MetaSchedule) navigates the schedule space defined in Ch 72; the polyhedral model provides the legal-schedule oracle that guarantees autotuner candidates are correct.
- **Ch 144–150 (Part XXI — MLIR Transformations)**: The MLIR Transform dialect's loop transformation operations (`transform.loop.tile`, `transform.loop.fuse`) are formal encodings of the schedule-space operations defined in Chs 71–72.
- **Ch 155–162 (Part XXII — XLA/OpenXLA)**: XLA's `op_cost_model` and `affine_analysis` components use polyhedral iteration space representations for fusion and tiling decisions in the HLO-to-GPU compilation pipeline; the scheduling concepts of Ch 72 underpin XLA's buffer assignment and kernel fusion strategies.

## Navigation

- ← Part X — Analysis & Middle-End
- → Part XII — Polly

---

*@copyright jreuben11*
