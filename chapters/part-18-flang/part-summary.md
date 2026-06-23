# Part XVIII — Flang — Part Summary

*This part covers LLVM's production Fortran compiler — from its driver and front-end architecture through the HLFIR/FIR dialect stack, OpenMP/OpenACC parallel offload, and final codegen to the flang-rt runtime library — equipping readers to understand, use, and extend the full Flang compilation pipeline for high-performance scientific computing.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 125 | Flang Architecture and Driver | Driver framework, parser, semantic analysis, lowering bridge |
| 126 | The HLFIR and FIR Dialects | FIR/HLFIR type system, ops, lowering pipeline to LLVM dialect |
| 127 | Flang OpenMP and OpenACC | omp.*/acc.* dialect lowering, GPU offload, DO CONCURRENT |
| 128 | Flang Codegen and Runtime | FIR-to-LLVM conversion, flang-rt I/O and intrinsics, IEEE FP |

## Part Overview

Fortran remains the dominant language for dense numerical computation — weather forecast models, nuclear physics codes, and the linear-algebra kernels underlying modern machine learning all depend on it. Yet for most of LLVM's history, the ecosystem lacked a production-quality Fortran front end. Part XVIII documents Flang, LLVM's in-tree Fortran compiler, which emerged from a 2017 clean-room rewrite (the f18 project, donated by NVIDIA to the LLVM Foundation in 2019) and reached production status — passing SPEC CPU 2017 Fortran benchmarks and compiling the ECMWF IFS operational weather model — by LLVM 22. The part traces the compiler from source file to object code, covering every stage at expert depth.

Chapter 125 establishes the architectural foundation. Flang reuses the Clang Driver framework for toolchain integration and adds a Fortran-specific front end consisting of three major components: a hand-written Prescanner that normalizes fixed-form and free-form source into a flat `CookedSource` with full provenance tracking; a recursive-descent Parser that produces a strongly-typed `ParseTree` of `std::tuple`/`std::variant`-based node types with no parser generator; and a semantic analysis engine that builds nested `Scope`/`Symbol` tables, resolves `USE` module imports, evaluates constant expressions into typed `Evaluate::Expr<T>` trees, and validates constraints through checker passes including `ResolveNames` (~8,000 lines), `CheckDeclarations`, and `CheckExpressions`. The `Lower::Bridge` class then converts the decorated parse tree into an `mlir::ModuleOp` holding HLFIR dialect operations.

Chapter 126 dissects the two-level MLIR dialect stack that sits between the Fortran front end and the LLVM backend. FIR (Fortran IR) provides types and operations faithful to Fortran's data model — `fir.box<T>` for array descriptors, `fir.heap<T>` and `fir.ptr<T>` for allocatables and pointers, `fir.array_load`/`fir.array_merge_store` for copy-safe array value semantics, and `fir.do_loop` for structured iteration. HLFIR (High-Level FIR), introduced in LLVM 16, sits above FIR and retains Fortran's data-parallel abstractions — `hlfir.elemental` for array expressions, `hlfir.matmul`/`hlfir.sum`/`hlfir.transpose` for intrinsics, and `hlfir.expr<T>` as an immutable array-value type. The lowering pipeline descends through `SimplifyHLFIRIntrinsics`, `LowerHLFIROrderedAssignments`, `BufferizeHLFIR`, `ConvertHLFIRtoFIR`, `ArrayValueCopy`, `TargetRewrite`, and `ConvertFIRToLLVM` before reaching the LLVM dialect.

Chapter 127 covers Flang's parallel programming support. OpenMP constructs (`!$omp parallel`, `!$omp do`, `!$omp target`, reductions, atomics) are validated by `check-omp-structure.cpp` and lowered to typed `omp.*` MLIR dialect operations by `flang/lib/Lower/OpenMP/OpenMP.cpp`, then converted to `__kmpc_*` and `__tgt_*` runtime calls by `OpenMPToLLVM`. GPU offload uses the 2024 TableGen-driven `OpenMP_Clause` redesign to represent map clauses uniformly, extracts device kernels via `ConvertOpenMPToGPUPass` → `gpu.launch_func`, and ultimately produces NVPTX or AMDGPU device binaries bundled by `clang-linker-wrapper`. OpenACC support mirrors this architecture using the `acc.*` dialect. Fortran's native `DO CONCURRENT` construct maps automatically to sequential, OpenMP, or GPU target regions via the `-fdo-concurrent-parallel=` driver flag.

Chapter 128 completes the pipeline with codegen and the runtime library. `ConvertFIRToLLVM` maps every FIR type to its LLVM counterpart — `fir.box` to the `CFI_cdesc_t`-compatible LLVM struct, character types to `i8` arrays — and every FIR op to its LLVM equivalent, with `TargetRewrite` handling ABI adjustments (hidden character length arguments, complex-number struct splitting). The `flang-rt` runtime library provides the services that generated code cannot inline: a rich formatted and list-directed I/O subsystem with a cookie-based state machine, intrinsic array operations (`_FortranAMatmul`, `_FortranASumReal8`, `_FortranAReshape`), allocatable lifetime management, polymorphic type descriptors for `CLASS` dispatch, and IEEE floating-point exception control via `feenableexcept`/`fegetexceptflag`.

## Key Concepts Introduced

- **`flang-new` driver**: Fortran compiler front end built on the Clang Driver framework; invokes `-fc1`, manages preprocessing, compilation, and links via `clang`.
- **Prescanner and CookedSource**: normalizes fixed-form and free-form Fortran into a contiguous character stream with a provenance map, handling `INCLUDE`, macros, and line continuations before the recursive-descent parser runs.
- **ParseTree**: strongly-typed, struct-per-production syntax tree built by Flang's hand-written recursive-descent parser; supports an `Unparse()` round-trip for correctness testing.
- **`Scope`/`Symbol` tables**: nested scoping model (global → module → subprogram → block) populated by `ResolveNames`; each `Symbol` carries a `SymbolDetails` variant with attributes (`ALLOCATABLE`, `INTENT`, `POINTER`, etc.).
- **`Evaluate::Expr<T>`**: compile-time typed expression tree; folds `PARAMETER` constants, array bounds, and `DATA` initializers at semantic analysis time.
- **`Lower::Bridge` / `AbstractConverter`**: the interface between Flang's semantic representation and MLIR code generation; drives the translation from `ParseTree` + decorated semantics into an `mlir::ModuleOp` containing HLFIR ops.
- **FIR (Fortran IR)**: MLIR dialect providing Fortran-specific types (`fir.ref<T>`, `fir.box<T>`, `fir.heap<T>`, `fir.ptr<T>`, `fir.array<NxT>`, `fir.char<K,L>`) and operations (`fir.alloca`, `fir.array_load`, `fir.array_merge_store`, `fir.do_loop`, `fir.embox`).
- **`fir.box<T>`**: the Fortran array descriptor type; lowers to a `CFI_cdesc_t`-compatible LLVM struct containing base pointer, element size, rank, and per-dimension lower bound/extent/stride.
- **HLFIR (High-Level FIR)**: dialect layer above FIR that retains Fortran's data-parallel semantics through `hlfir.elemental`, `hlfir.expr<T>`, `hlfir.assign`, and intrinsic-specific ops (`hlfir.matmul`, `hlfir.sum`, `hlfir.transpose`), enabling high-level optimizations before explicit loop generation.
- **`ArrayValueCopy`**: FIR pass that performs alias analysis on `fir.array_load`/`fir.array_merge_store` chains to determine when Fortran array assignment semantics require an intermediate temporary copy.
- **`omp.*` dialect**: MLIR representation of OpenMP 5.1 constructs (`omp.parallel`, `omp.wsloop`, `omp.simd`, `omp.target`, `omp.teams`, `omp.distribute`, `omp.declare_reduction`, `omp.atomic.*`); lowers to `__kmpc_*`/`__tgt_*` runtime calls.
- **TableGen-driven `OpenMP_Clause`**: 2024 redesign that defines each OpenMP clause as a separate ODS entity mixed into ops, replacing ad-hoc attribute representation and enabling uniform lowering across backends.
- **`acc.*` dialect**: MLIR representation of OpenACC 3.x constructs (`acc.parallel`, `acc.kernels`, `acc.loop`, `acc.data`, `acc.enter_data`, `acc.copyin`, `acc.copyout`, `acc.routine`).
- **DO CONCURRENT parallelism**: Fortran 2008 language-level iteration independence annotation; Flang maps it to sequential FIR loops, OpenMP parallel work-sharing, or GPU target regions via the `-fdo-concurrent-parallel=none|host|device` flag.
- **`flang-rt` runtime library**: provides `_FortranAio*` I/O entry points, `_FortranAMatmul`/reduction/reshape intrinsic array routines, `_FortranAAllocatableAllocate`/`Deallocate`, polymorphic `DerivedType`/`Binding` type info, and IEEE FP exception control; the `Descriptor` class mirrors `CFI_cdesc_t`.
- **`bbc` tool**: Flang's `mlir-opt` equivalent; drives the front end to HLFIR or FIR and is the primary vehicle for lit FileCheck tests in `flang/test/Lower/` and `flang/test/HLFIR/`.

## How This Part Fits the Book

Part XVIII builds directly on the LLVM IR and MLIR foundations established in Parts IV (LLVM IR), XIX (MLIR Foundations), and XX (In-Tree Dialects) — Flang's FIR and HLFIR dialects are MLIR dialects defined via TableGen ODS, and the entire lowering pipeline from HLFIR to object code runs through the same MLIR pass manager and `ConvertMLIRToLLVMIR` translation infrastructure described in Parts XIX–XXI (Chapters 129–145). The OpenMP and OpenACC material in Chapter 127 connects to the LLVM runtime library architecture of Part XVII (Chapters 119–124), specifically libomp and liboffload. Parts XX and XXI (in-tree MLIR dialects and transformations, Chapters 132–145) expand on the `omp.*`, `acc.*`, and `gpu.*` dialects used here. Part XXII (XLA/OpenXLA, Chapters 146–149) and Part XXIII (MLIR in production, Chapters 150–155) show how the same dialect-based progressive lowering philosophy scales to ML compilers.

## Cross-Part Dependencies

- Ch 129–131 (Part XIX — MLIR Foundations): MLIR's dialect system, ODS, and pass manager are prerequisites for understanding FIR/HLFIR as MLIR dialects; Ch 126–128 reference Ch 129 explicitly.
- Ch 132–145 (Part XX–XXI — In-Tree Dialects and Transformations): the `omp.*`, `acc.*`, `gpu.*`, `llvm.*`, and `memref.*` dialects used by Flang's lowering pipeline are documented there; `OpenMPToLLVM` and `ConvertGpuOpsToNVVMOps` are MLIR infrastructure passes.
- Ch 102 (Part XV — Targets: NVPTX and the CUDA path): GPU offload in Ch 127 routes through the NVPTX backend and `clang-linker-wrapper` described in Ch 102.
- Ch 119–124 (Part XVII — Runtime Libraries): libomp (`__kmpc_*`), liboffload (`__tgt_*`), and the OpenACC runtime called from Flang GPU kernels are covered there.
- Ch 113–116 (Part XVI — JIT and Sanitizers): Flang's `-fbounds-check` runtime checks use the same `Terminator::Crash()` path that intersects with sanitizer-instrumented builds; LLVM's loop vectorizer applied to FIR-lowered loops connects to the JIT optimization pipeline.
- Ch 149–155 (Parts XXIV–XXIII — Verified Compilation and MLIR in Production): the roadmap items in Chs 126–128 for formal verification of FIR array-copy semantics and flang-rt I/O correctness via Lean 4/Coq connect directly to Ch 149 (Alive2) and Ch 150 (CompCert/Vellvm).
- Ch 160–165 (Part XXVI — Ecosystem Frontiers): Flang's role as the open-source Fortran compiler for HPC workloads (DOE Frontier/Aurora) is contextualised by the ecosystem coverage there.

## Navigation

- ← Part XVII — Runtime Libraries
- → Part XIX — MLIR Foundations

---

*@copyright jreuben11*
