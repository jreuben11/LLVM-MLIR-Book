# Part XII — Polly — Part Summary

*This part covers Polly, LLVM's polyhedral loop optimizer — how it detects affine loop nests in LLVM IR, applies cache tiling, fusion, parallelization, and vectorization via ISL, and how it relates to the MLIR Affine dialect that is superseding it.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 74 | Polly Architecture | SCoP detection, polyhedral IR, ISL integration, code generation pipeline |
| 75 | Polly Transformations | Tiling, fusion, parallelization, vectorization, matmul pattern optimizer, JSCOP format |
| 76 | Polly in Practice | Applicability, diagnostics, GPU codegen, maintenance status, MLIR comparison |

## Part Overview

Part XII applies the theoretical machinery of Part XI — Polyhedral Theory — to a concrete, production-grade implementation inside the LLVM toolchain. Polly operates entirely on LLVM IR: it extracts Static Control Parts (SCoPs) from the SSA graph using ScalarEvolution to verify that loop bounds and array access functions are affine, converts them to ISL sets and maps, applies the Pluto-style ISL scheduler, and regenerates LLVM IR from the resulting schedule tree. The three chapters move in a natural arc from architecture, through transformation mechanisms, to practical usage and ecosystem positioning.

Chapter 74 establishes the internal pipeline: ScopDetection identifies maximal SCoP regions by verifying affine bounds and access functions through ScalarEvolution; ScopInfo constructs the polyhedral representation as `Scop`, `ScopStmt`, and `MemoryAccess` objects expressed as ISL sets and maps, with delinearization recovering multi-dimensional array structure from flat GEP addresses; DependencesAnalysis computes exact RAW/WAR/WAW dependences using `isl_union_map_compute_flow`; ScheduleOptimizerPass applies the ISL Pluto-style scheduler and marks tiled, parallel, and vectorizable dimensions; and CodeGenerationPass walks the ISL AST to emit new LLVM IR, substituting new induction variable values for original SSA values via a value map. The chapter also positions Polly against standalone polyhedral tools such as Pluto, highlighting that Polly's IR-to-IR nature eliminates a second compilation step and preserves SSA form for the LLVM backend.

Chapter 75 details how each class of polyhedral transformation materializes in practice. Loop tiling splits permutable band dimensions into tile-coordinate and point-coordinate loops; the default 32-element tile size is a conservative heuristic replaceable by user-specified sizes or ISL auto-tuning. Loop fusion is controlled by coincidence constraints in the ISL scheduler, with `MaxFuse` mode aggressively prioritizing data reuse. Parallelization wraps coincident dimensions in OpenMP `parallel for` with reduction clause detection for scalar reductions. Vectorization either delegates to LLVM's Loop Vectorizer via loop metadata or directly emits vector IR in `-polly-vectorizer=polly` mode. A pattern-based recognizer detects the three-nested matrix multiplication kernel and applies a fixed cache-tiling, register-tiling, and unroll-and-jam schedule that bypasses the generic ISL scheduler. The JSON `.jscop` schedule format provides a round-trip interface enabling external ML schedulers, Tiramisu, or manual schedule injection to drive Polly's code generation without modifying its C++ source.

Chapter 76 grounds the abstractions in empirical reality: where Polly succeeds (high arithmetic intensity kernels, regular stencils, nested parallelism hidden from scalar passes), where it fails (indirect accesses, data-dependent conditions, impure function calls, short loops), and how to read diagnostic output via `-polly-report`, `-polly-report-scop-restrictions`, and `-Rpass-missed=polly-detect`. The chapter documents the experimental Polly-ACC GPU offloading path, which generates CUDA kernels from polyhedral schedules but is hampered by naive data-transfer policies. Critically, it assesses Polly's maintenance status as of LLVM 22.1 — bug-fix mode with active development migrating to the MLIR Affine dialect — and provides a detailed comparison of the two approaches, emphasizing that MLIR represents polyhedral structure explicitly in the IR rather than recovering it retroactively via ScalarEvolution. For new work, the MLIR Affine dialect is the recommended path.

## Key Concepts Introduced

- **SCoP (Static Control Part)**: A maximal affine region of a function — all loop bounds, branch conditions, and memory access functions are affine in the loop induction variables and symbolic parameters. SCoPs are the units Polly analyzes and transforms.
- **ScopDetection**: The Polly pass that scans the CFG region tree and verifies SCoP conditions using ScalarEvolution analysis; produces the set of regions eligible for polyhedral optimization.
- **ScopInfo / `Scop` class**: The polyhedral representation of a detected SCoP, holding an ISL context set, an assumption set, a schedule tree, and a collection of `ScopStmt` objects with `MemoryAccess` records expressed as ISL sets and maps.
- **`MemoryAccess` access relation**: An ISL map from the statement's iteration domain to the memory reference space (`{ [i,j] -> MemRef_A[i, j] }`), recovered from GEP instructions via ScalarEvolution and delinearization.
- **Delinearization**: The process of recovering multi-dimensional array structure from flat linear GEP address computations (e.g., decomposing `i * M + j` back into `[i, j]`), essential for building correct 2D or higher-dimensional access relations.
- **Validity and proximity dependence constraints**: The two categories of ISL constraints passed to the scheduler — validity constraints specify dependences that the schedule must respect; proximity constraints specify dependences whose distance the scheduler should minimize for data locality.
- **ScheduleOptimizerPass**: The Polly pass that applies the ISL Pluto-style scheduler to compute an optimized schedule tree, then adds tiling band splits and marks parallel and vectorizable dimensions before code generation.
- **Loop tiling in Polly**: Splitting permutable schedule band dimensions into tile-coordinate outer loops and point-coordinate inner loops; controlled by `-polly-tile-sizes` and ISL auto-tiling; the primary mechanism for improving cache reuse on matrix and stencil kernels.
- **MaxFuse scheduler**: Polly's aggressive loop fusion mode (`-polly-fuse=max`) that modifies the ISL coincidence constraints to maximize the number of statement pairs executing at the same time step, prioritizing data reuse over parallelism.
- **Pattern-based matmul optimizer**: A special-case recognizer in `ScheduleOptimizer.cpp` that detects the three-nested matrix multiplication access pattern (`A[i][k]`, `B[k][j]`, `C[i][j]`) and applies a fixed high-performance schedule (cache tiling, register tiling, vectorization, unroll-and-jam) bypassing the generic ISL scheduler.
- **JSCOP format**: The JSON interchange format for Polly's polyhedral representation (domain, schedule, access functions), exportable via `-polly-export-jscop` and injectable via `-polly-import-jscop`, enabling external schedule sources (ML models, Tiramisu, manual edits) to drive Polly's code generation.
- **Runtime alias checks**: Guards inserted before the polyhedral region when pointer aliasing cannot be disproved statically; if the check fails at runtime, execution falls back to the original non-optimized loop code.
- **CodeGenerationPass / IslNodeBuilder**: The Polly component that walks the ISL AST produced by `isl_ast_build` and emits LLVM IR basic blocks, creating new loop headers and substituting new IV values for original SSA values via a value map.
- **Polly-ACC**: The experimental CUDA/OpenCL GPU offloading path in Polly, which generates a two-level parallel schedule mapped to CUDA grid/block dimensions plus data transfer code; limited by naive whole-array transfers and not production-ready.
- **Non-affine subregions**: An extension of Polly's SCoP model that treats blocks of non-affine code (impure function calls, data-dependent conditions) as opaque black boxes with conservative memory effects, allowing the surrounding affine code to still be optimized.

## How This Part Fits the Book

Part XII builds directly on Part XI — Polyhedral Theory (Chapters 69–73), which established the mathematical foundations: integer polyhedra, the Presburger arithmetic fragment, ISL's set and map algebra, Feautrier scheduling, the Pluto algorithm, and ISL's AST generation. Every mechanism described in Part XII (dependence computation, schedule optimization, tiling, code generation) is the concrete LLVM implementation of the abstractions developed in Part XI. Part XII also draws on Part X — Analysis and the Middle-End (especially Chapter 61 on ScalarEvolution and alias analysis), which provides the LLVM analysis infrastructure Polly queries during SCoP detection and dependence analysis. Looking forward, Part XIII — LTO and Whole-Program Analysis builds on the pass pipeline concepts introduced here, and Part XIX and Part XX — MLIR Foundations and In-Tree Dialects, particularly Chapter 93 (Affine dialect) and Chapter 140 (MLIR Affine Dialect deep dive), represent the modern successor to Polly's approach that Chapter 76 positions explicitly.

## Cross-Part Dependencies

- **Ch 61 (Part X — Analysis, Middle-End)** — ScalarEvolution and AliasAnalysis are the foundational LLVM analyses Polly's ScopDetection and DependencesAnalysis query; understanding Ch 61 is prerequisite for Ch 74's ScoP detection mechanics.
- **Ch 64 (Part X — Analysis, Middle-End)** — The Loop Vectorizer integrates with Polly's vectorization path: Polly sets `llvm.loop.vectorize.enable` metadata and relies on Ch 64's pass to perform actual SIMD code generation.
- **Ch 69–73 (Part XI — Polyhedral Theory)** — The ISL scheduling, Pluto algorithm, Feautrier multi-dimensional scheduling, and ISL AST generation that Ch 74 and Ch 75 implement in C++; Part XII is the applied counterpart of Part XI.
- **Ch 93 (Part XX — In-Tree Dialects)** — MLIR's Affine dialect, which represents polyhedral structure explicitly rather than recovering it from LLVM IR; Ch 76 compares Polly and MLIR Affine directly and positions MLIR as the preferred path for new work.
- **Ch 140 (Part XX — In-Tree Dialects)** — Deep coverage of the MLIR Affine dialect transformation passes (`affine-loop-tile`, `affine-loop-fusion`, `affine-parallelize`); enriched by the contrast with Polly's approach established in Ch 76.
- **Ch 165 (Part XXIV — Verified Compilation)** — Alive2-style semantic equivalence checking; the Ch 74 roadmap section cites applying Alive2 to verify Polly's schedule transformations and code generation for soundness.
- **Ch 48–49 (Part XVI — JIT and Sanitizers / GPU targets)** — CUDA/HIP explicit GPU programming, noted in Ch 76 as the production alternative to Polly-ACC's experimental offloading path.

## Navigation

- ← Part XI — Polyhedral Theory
- → Part XIII — LTO & Whole-Program Analysis

---

*@copyright jreuben11*
