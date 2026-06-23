# Part XXI — MLIR Transformations — Part Summary

*This part explains how MLIR programs are systematically transformed from high-level abstractions to low-level code, covering the full mechanical stack from individual rewrite rules through pass pipelines and the metaprogramming model that makes transformation schedules first-class programs.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 147 | Pattern Rewriting | Greedy rewrite driver, `OpRewritePattern`, canonicalization, matchers |
| 148 | Dialect Conversion | Type-changing rewrites, `ConversionTarget`, `TypeConverter`, materialization |
| 149 | The Pass Infrastructure | Pass scheduling, nesting, parallelism, analysis caching |
| 150 | The Transform Dialect | Compiler strategies as first-class MLIR programs |
| 151 | Bufferization Deep Dive | One-Shot Bufferization, copy-elision analysis, `BufferizableOpInterface` |
| 152 | Lowering Pipelines | End-to-end CPU/GPU/IREE pipelines, pass ordering, debugging |

## Part Overview

Part XXI addresses the central challenge of any multi-level compiler built on MLIR: how to move a program from the high-level dialect it enters the compiler in down to machine-executable form, while preserving correctness and maximizing performance. The six chapters are ordered from the finest-grained mechanism to the coarsest-grained composition. Chapter 147 begins at the atomic unit of transformation: a `RewritePattern` that matches one IR subgraph and replaces it with an equivalent one. The greedy driver applies patterns to fixpoint, and this mechanism underlies canonicalization, constant folding, and most simplification passes throughout the MLIR tree. Chapter 148 extends the same model to cross-type rewrites via the dialect conversion framework, which coordinates legality checking, type mapping, and materialization of cast operations to keep the IR consistent during multi-dialect translations such as `linalg` to `loops` or `arith` to `llvm`.

Chapter 149 steps up to the organizational level: the pass infrastructure that schedules, isolates, and parallelizes passes over the op hierarchy. It establishes the `OperationPass<T>` nesting model, the analysis manager's cache-invalidation discipline, and the instrumentation hooks used by the timing, statistics, and crash-reproduction facilities. Chapter 150 then introduces the Transform dialect, MLIR's answer to the static-pipeline limitation: rather than hard-coding tile sizes and transformation sequences in C++ passes, the Transform dialect lets a compiler author write a second MLIR program — the schedule — that queries and mutates the payload IR at compile time. Named sequences, `transform.structured.*` tiling/fusion/vectorization ops, and `transform.foreach_match` dispatch give the schedule the expressiveness of a domain-specific language for compiler strategies.

Chapters 151 and 152 address the two most practically consequential concerns in ML and HPC compilation. Chapter 151 dissects One-Shot Bufferization — the algorithm that converts tensor-semantic programs to buffer-semantic programs with provably minimal copies. Its two-phase design (global alias analysis before any rewrite) and the `BufferizableOpInterface` extension mechanism are the bridge between the value-semantic world where optimizations are clean and the aliasing world where code runs. Chapter 152 ties everything together by presenting complete, runnable lowering pipelines for CPU, GPU, and the IREE production deployment compiler, explaining the ordering constraints between bufferization, vectorization, loop lowering, and dialect-to-LLVM conversions, along with the diagnostic tools for when any stage fails.

A reader who completes this part can write custom `RewritePattern` and `OpConversionPattern` classes, construct `ConversionTarget`/`TypeConverter` pairs for a new lowering, build and instrument a nested `PassManager`, implement `BufferizableOpInterface` for a custom dialect, author a Transform dialect schedule that tiles and vectorizes Linalg ops without recompiling the compiler, and assemble or debug a complete tensor-to-LLVM pipeline.

## Key Concepts Introduced

- **`RewritePattern` / `OpRewritePattern<T>`**: The atomic unit of IR transformation; implements `matchAndRewrite`, returns `success()` only when the IR was mutated, and is driven by the greedy engine to fixpoint.
- **Greedy rewrite driver (`applyPatternsGreedily`)**: The fixpoint engine that applies a `FrozenRewritePatternSet` to an op tree, maintaining a worklist of affected operations; controlled by `GreedyRewriteConfig` for iteration limits, traversal order, and strictness.
- **`PatternBenefit`**: An integer priority attached to every `RewritePattern`; the driver tries higher-benefit patterns first when multiple patterns match the same root op.
- **`ConversionTarget`**: Declares which ops and dialects are legal, illegal, or dynamically legal in the output IR; drives `applyPartialConversion` / `applyFullConversion` to determine what must be converted.
- **`TypeConverter`**: Maps source types to target types and registers source, target, and argument materialization callbacks that insert `unrealized_conversion_cast` ops at type boundaries during multi-step lowering.
- **`OpConversionPattern<T>` / `OpAdaptor`**: The conversion-specific pattern form; receives pre-converted operands via the auto-generated adaptor, enabling patterns to build replacement ops using already-converted types.
- **`OperationPass<T>` nesting model**: Passes are parameterized by op type; `PassManager::nest<T>()` creates sub-managers that run on each op of type `T` independently, enabling safe parallel execution and scoped isolation.
- **Analysis manager and `getAnalysis<T>()`**: A caching layer that constructs analyses on demand and invalidates them after IR-modifying passes; `markAnalysesPreserved<T>()` prevents unnecessary recomputation.
- **Transform dialect two-program architecture**: The payload IR (the program being compiled) and the transform IR (the compilation strategy) are distinct MLIR programs; the transform interpreter executes the strategy against the payload without the strategy ever becoming machine code.
- **`transform.structured.*` ops**: High-level transformation ops — `tile_using_for`, `fuse_into_containing_op`, `vectorize`, `pad`, `decompose_convolutions` — that drive Linalg transformations from within the Transform dialect, returning handles to newly created ops for subsequent steps.
- **`TransformState` and handle linearity**: The interpreter's bookkeeping object that maps `!transform.any_op` / `!transform.any_value` / `!transform.param<T>` handles to payload IR elements; enforces that handles referencing erased ops cannot be reused.
- **One-Shot Bufferization two-phase algorithm**: Phase 1 (analysis) walks the IR and decides for each tensor-producing op whether its result can be placed in-place in an operand buffer; Phase 2 (rewrite) applies those decisions, inserting copies only where the analysis found aliasing conflicts.
- **`BufferizableOpInterface`**: The extension point for bufferization; methods `bufferizesToMemoryRead`, `bufferizesToMemoryWrite`, `getAliasingValues`, `getBufferType`, and `bufferize` together tell the algorithm how each op interacts with memory.
- **Ownership-based buffer deallocation**: A separate post-bufferization pass pipeline that assigns ownership to each `memref.alloc` and inserts exactly one `memref.dealloc` per allocation in the owning block, using `BufferDeallocationOpInterface` for custom region semantics.
- **`reconcile-unrealized-casts`**: The final cleanup pass that removes `unrealized_conversion_cast` chains left by dialect conversion after all conversion patterns have run; surviving casts indicate incomplete conversion.

## How This Part Fits the Book

Part XXI builds directly on Part XIX (MLIR Foundations, Chapters 130–137), which establishes the IR model, op interfaces, ODS, and the type/attribute system, and on Part XX (In-Tree Dialects, Chapters 138–146), which provides the dialect vocabulary — `linalg`, `tensor`, `memref`, `scf`, `affine`, `vector` — that the transformation passes in this part consume and produce. The material here is in turn the prerequisite foundation for Part XXII (XLA/OpenXLA, Chapters 153–161), which relies on the bufferization, lowering pipelines, and Transform dialect infrastructure to understand how StableHLO and MHLO programs are compiled for TPU and GPU targets, and for Part XXIII (MLIR in Production, Chapters 162–170), which covers the IREE compiler, custom backend development, and deployment artifacts that all depend on the lowering pipeline architecture presented in Chapter 152.

## Cross-Part Dependencies

- Ch 138–146 (Part XX — In-Tree Dialects): Pattern rewriting and dialect conversion directly transform ops from `linalg`, `tensor`, `scf`, `affine`, `vector`, `memref`, and `arith`; understanding these dialects' semantics is required to write correct patterns and bufferization interfaces.
- Ch 153–161 (Part XXII — XLA/OpenXLA): StableHLO and HLO lowering pipelines build on `applyPartialConversion`, `TypeConverter`, and One-Shot Bufferization; the MHLO-to-Linalg lowering is a concrete application of `OpConversionPattern`.
- Ch 162–170 (Part XXIII — MLIR in Production): IREE's pipeline stages — flow, stream, HAL — are assembled using the `PassManager` nesting model and `populate*` pattern functions from Chapter 149 and Chapter 152; Transform dialect schedules (Ch 150) are IREE's primary vehicle for kernel-specific optimization.
- Ch 171–179 (Part XXIV — Verified Compilation): Alive2/Vellvm-style semantic equivalence checking is the formal underpinning for verifying that `RewritePattern` and bufferization decisions are correct; the research roadmap sections in Chapters 147, 148, and 151 each reference this connection.
- Ch 180–188 (Part XXV — Operations and Contribution): Writing a new in-tree pass or dialect lowering requires the full workflow from Chapters 147–149: pattern sets, conversion targets, and `getDependentDialects` registration.
- Ch 196 (Part XXVI — Ecosystem Frontiers): The LLVM IR as interchange section assumes familiarity with the `--convert-*-to-llvm` passes and `mlir-translate --mlir-to-llvmir` pipeline presented in Chapter 152.

## Navigation

- ← Part XX — In-Tree Dialects
- → Part XXII — XLA / OpenXLA

---

*@copyright jreuben11*
