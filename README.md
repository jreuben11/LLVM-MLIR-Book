# LLVM, Clang & MLIR — The Expert Programming Reference

*With Compiler Theory, Type Theory, Polyhedral Theory, and Verified Compilation*

**Targeting LLVM 22.1.x · Clang 22 · MLIR in-tree · OpenXLA · ClangIR · April 2026**

---

A ~3,000-page expert reference for the LLVM compiler ecosystem. 182 chapters and 8 appendices across 26 parts. Beyond the LLVM/MLIR/XLA/Clang implementation, the book covers the theoretical foundations of compilation (Dragon Book / Cooper-Torczon), modern type theory (Pierce's TAPL / Harper's PFPL), the polyhedral model derived from Presburger arithmetic, formal verification of compilers (CompCert, Vellvm, Alive2), and the Rust compiler ecosystem including push-button program verification tools.

Practical chapters are verified against LLVM 22.1.x (`clang --version` → `22.x`). Theoretical chapters are anchored to the canonical literature cited per-chapter. All Rust code targets Edition 2024 (Rust 1.85+).

**License:** [CC BY 4.0](LICENSE) · **Copyright:** © 2026 [jreuben11](https://github.com/jreuben11)

---

## Table of Contents

### Part I — Foundations *(~85 pp)*

| # | Chapter |
|---|---------|
| 1 | [The LLVM Project](chapters/part-01-foundations/ch01-the-llvm-project.md) |
| 2 | [Building LLVM from Source](chapters/part-01-foundations/ch02-building-llvm-from-source.md) |
| 3 | [The Compilation Pipeline](chapters/part-01-foundations/ch03-the-compilation-pipeline.md) |
| 4 | [The LLVM C++ API](chapters/part-01-foundations/ch04-the-llvm-cpp-api.md) |
| 5 | [LLVM as a Library](chapters/part-01-foundations/ch05-llvm-as-a-library.md) |

### Part II — Compiler Theory and Foundations *(~120 pp)* `[THEORETICAL]`

| # | Chapter |
|---|---------|
| 6 | [Lexical Analysis](chapters/part-02-compiler-theory/ch06-lexical-analysis.md) |
| 7 | [Parsing Theory](chapters/part-02-compiler-theory/ch07-parsing-theory.md) |
| 8 | [Semantic Analysis Foundations](chapters/part-02-compiler-theory/ch08-semantic-analysis-foundations.md) |
| 9 | [Intermediate Representations and SSA Construction](chapters/part-02-compiler-theory/ch09-intermediate-representations-and-ssa.md) |
| 10 | [Dataflow Analysis: The Lattice Framework](chapters/part-02-compiler-theory/ch10-dataflow-analysis-lattice-framework.md) |
| 11 | [Classical Optimization Theory](chapters/part-02-compiler-theory/ch11-classical-optimization-theory.md) |

### Part III — Type Theory *(~80 pp)* `[THEORETICAL]`

| # | Chapter |
|---|---------|
| 12 | [Lambda Calculus and Simple Types](chapters/part-03-type-theory/ch12-lambda-calculus-and-simple-types.md) |
| 13 | [Polymorphism and Type Inference](chapters/part-03-type-theory/ch13-polymorphism-and-type-inference.md) |
| 14 | [Advanced Type Systems](chapters/part-03-type-theory/ch14-advanced-type-systems.md) |
| 15 | [Type Theory in Practice: From Theory to LLVM and MLIR](chapters/part-03-type-theory/ch15-type-theory-in-practice.md) |

### Part IV — LLVM IR *(~225 pp)*

| # | Chapter |
|---|---------|
| 16 | [IR Structure](chapters/part-04-llvm-ir/ch16-ir-structure.md) |
| 17 | [The Type System](chapters/part-04-llvm-ir/ch17-the-type-system.md) |
| 18 | [Constants, Globals, and Linkage](chapters/part-04-llvm-ir/ch18-constants-globals-and-linkage.md) |
| 19 | [Instructions I — Arithmetic and Memory](chapters/part-04-llvm-ir/ch19-instructions-arithmetic-and-memory.md) |
| 20 | [Instructions II — Control Flow and Aggregates](chapters/part-04-llvm-ir/ch20-instructions-control-flow-and-aggregates.md) |
| 21 | [SSA, Dominance, and Loops](chapters/part-04-llvm-ir/ch21-ssa-dominance-and-loops.md) |
| 22 | [Metadata and Debug Info](chapters/part-04-llvm-ir/ch22-metadata-and-debug-info.md) |
| 23 | [Attributes, Calling Conventions, and the ABI](chapters/part-04-llvm-ir/ch23-attributes-calling-conventions-abi.md) |
| 24 | [Intrinsics](chapters/part-04-llvm-ir/ch24-intrinsics.md) |
| 25 | [Inline Assembly](chapters/part-04-llvm-ir/ch25-inline-assembly.md) |
| 26 | [Exception Handling](chapters/part-04-llvm-ir/ch26-exception-handling.md) |
| 27 | [Coroutines and Atomics](chapters/part-04-llvm-ir/ch27-coroutines-and-atomics.md) |

### Part V — Clang Internals: Frontend Pipeline *(~210 pp)*

| # | Chapter |
|---|---------|
| 28 | [The Clang Driver](chapters/part-05-clang-frontend/ch28-the-clang-driver.md) |
| 29 | [SourceManager, FileEntry, SourceLocation](chapters/part-05-clang-frontend/ch29-sourcemanager-fileentry-sourcelocation.md) |
| 30 | [The Diagnostic Engine](chapters/part-05-clang-frontend/ch30-the-diagnostic-engine.md) |
| 31 | [The Lexer and Preprocessor](chapters/part-05-clang-frontend/ch31-the-lexer-and-preprocessor.md) |
| 32 | [The Parser](chapters/part-05-clang-frontend/ch32-the-parser.md) |
| 33 | [Sema I — Names, Lookups, and Conversions](chapters/part-05-clang-frontend/ch33-sema-names-lookups-conversions.md) |
| 34 | [Sema II — Templates, Concepts, and Constraints](chapters/part-05-clang-frontend/ch34-sema-templates-concepts-constraints.md) |
| 35 | [The Constant Evaluator](chapters/part-05-clang-frontend/ch35-the-constant-evaluator.md) |
| 36 | [The Clang AST in Depth](chapters/part-05-clang-frontend/ch36-the-clang-ast-in-depth.md) |
| 37 | [C++ Modules Implementation](chapters/part-05-clang-frontend/ch37-cpp-modules-implementation.md) |
| 38 | [Code Completion and clangd Foundations](chapters/part-05-clang-frontend/ch38-code-completion-and-clangd-foundations.md) |

### Part VI — Clang Internals: Codegen and ABI *(~120 pp)*

| # | Chapter |
|---|---------|
| 39 | [CodeGenModule and CodeGenFunction](chapters/part-06-clang-codegen/ch39-codegenmodule-and-codegenfunction.md) |
| 40 | [Lowering Statements and Expressions](chapters/part-06-clang-codegen/ch40-lowering-statements-and-expressions.md) |
| 41 | [Calls, the ABI Boundary, and Builtins](chapters/part-06-clang-codegen/ch41-calls-abi-boundary-and-builtins.md) |
| 42 | [C++ ABI Lowering: Itanium](chapters/part-06-clang-codegen/ch42-cpp-abi-lowering-itanium.md) |
| 43 | [C++ ABI Lowering: Microsoft](chapters/part-06-clang-codegen/ch43-cpp-abi-lowering-microsoft.md) |
| 44 | [Coroutine Lowering in Clang](chapters/part-06-clang-codegen/ch44-coroutine-lowering-in-clang.md) |

### Part VII — Clang as a Multi-Language Compiler *(~140 pp)*

| # | Chapter |
|---|---------|
| 45 | [The Static Analyzer](chapters/part-07-clang-multilang/ch45-the-static-analyzer.md) |
| 46 | [libtooling and AST Matchers](chapters/part-07-clang-multilang/ch46-libtooling-and-ast-matchers.md) |
| 47 | [clangd, clang-tidy, clang-format, clang-refactor](chapters/part-07-clang-multilang/ch47-clangd-clang-tidy-clang-format-clang-refactor.md) |
| 48 | [Clang as a CUDA Compiler](chapters/part-07-clang-multilang/ch48-clang-as-cuda-compiler.md) |
| 49 | [Clang as a HIP Compiler](chapters/part-07-clang-multilang/ch49-clang-as-hip-compiler.md) |
| 50 | [Clang as SYCL, OpenCL, and OpenMP-Offload](chapters/part-07-clang-multilang/ch50-clang-as-sycl-opencl-openmp-offload.md) |
| 51 | [Clang as an HLSL Compiler](chapters/part-07-clang-multilang/ch51-clang-as-hlsl-compiler.md) |

### Part VIII — ClangIR (CIR) *(~50 pp)*

| # | Chapter |
|---|---------|
| 52 | [ClangIR Architecture](chapters/part-08-clangir/ch52-clangir-architecture.md) |
| 53 | [CIR Generation from AST](chapters/part-08-clangir/ch53-cir-generation-from-ast.md) |
| 54 | [CIR Lowering and Analysis](chapters/part-08-clangir/ch54-cir-lowering-and-analysis.md) |

### Part IX — Frontend Authoring (Building Your Own) *(~80 pp)*

| # | Chapter |
|---|---------|
| 55 | [Building a Frontend](chapters/part-09-frontend-authoring/ch55-building-a-frontend.md) |
| 56 | [Lowering AST to IR](chapters/part-09-frontend-authoring/ch56-lowering-ast-to-ir.md) |
| 57 | [Lowering High-Level Constructs](chapters/part-09-frontend-authoring/ch57-lowering-high-level-constructs.md) |
| 58 | [Language Runtime Concerns](chapters/part-09-frontend-authoring/ch58-language-runtime-concerns.md) |

### Part X — Analysis and the Middle-End *(~210 pp)*

| # | Chapter |
|---|---------|
| 59 | [The New Pass Manager](chapters/part-10-analysis-middle-end/ch59-the-new-pass-manager.md) |
| 60 | [Writing a Pass](chapters/part-10-analysis-middle-end/ch60-writing-a-pass.md) |
| 61 | [Foundational Analyses](chapters/part-10-analysis-middle-end/ch61-foundational-analyses.md) |
| 62 | [Scalar Optimizations](chapters/part-10-analysis-middle-end/ch62-scalar-optimizations.md) |
| 63 | [Loop Optimizations](chapters/part-10-analysis-middle-end/ch63-loop-optimizations.md) |
| 64 | [Vectorization Deep Dive](chapters/part-10-analysis-middle-end/ch64-vectorization-deep-dive.md) |
| 65 | [Inter-Procedural Optimizations](chapters/part-10-analysis-middle-end/ch65-inter-procedural-optimizations.md) |
| 66 | [The ML Inliner and ML Regalloc](chapters/part-10-analysis-middle-end/ch66-ml-inliner-and-regalloc.md) |
| 67 | [Profile-Guided Optimization](chapters/part-10-analysis-middle-end/ch67-profile-guided-optimization.md) |
| 68 | [Hardening and Mitigations](chapters/part-10-analysis-middle-end/ch68-hardening-and-mitigations.md) |
| 69 | [Whole-Program Devirtualization](chapters/part-10-analysis-middle-end/ch69-whole-program-devirtualization.md) |

### Part XI — Polyhedral Theory *(~80 pp)* `[THEORETICAL]`

| # | Chapter |
|---|---------|
| 70 | [Foundations: Polyhedra and Integer Programming](chapters/part-11-polyhedral-theory/ch70-foundations-polyhedra-and-integer-programming.md) |
| 71 | [The Polyhedral Model](chapters/part-11-polyhedral-theory/ch71-the-polyhedral-model.md) |
| 72 | [Scheduling Algorithms](chapters/part-11-polyhedral-theory/ch72-scheduling-algorithms.md) |
| 73 | [Code Generation from Polyhedral Schedules](chapters/part-11-polyhedral-theory/ch73-code-generation-from-polyhedral-schedules.md) |

### Part XII — Polly *(~50 pp)*

| # | Chapter |
|---|---------|
| 74 | [Polly Architecture](chapters/part-12-polly/ch74-polly-architecture.md) |
| 75 | [Polly Transformations](chapters/part-12-polly/ch75-polly-transformations.md) |
| 76 | [Polly in Practice](chapters/part-12-polly/ch76-polly-in-practice.md) |

### Part XIII — Link-Time and Whole-Program *(~80 pp)*

| # | Chapter |
|---|---------|
| 77 | [LTO and ThinLTO](chapters/part-13-lto-whole-program/ch77-lto-and-thinlto.md) |
| 78 | [The LLVM Linker (LLD)](chapters/part-13-lto-whole-program/ch78-the-llvm-linker-lld.md) |
| 79 | [Linker Internals: GOT, PLT, TLS](chapters/part-13-lto-whole-program/ch79-linker-internals-got-plt-tls.md) |
| 80 | [llvm-cas and Content-Addressable Builds](chapters/part-13-lto-whole-program/ch80-llvm-cas-and-content-addressable-builds.md) |

### Part XIV — The Backend *(~280 pp)*

| # | Chapter |
|---|---------|
| 81 | [Backend Architecture](chapters/part-14-backend/ch81-backend-architecture.md) |
| 82 | [TableGen Deep Dive](chapters/part-14-backend/ch82-tablegen-deep-dive.md) |
| 83 | [The Target Description](chapters/part-14-backend/ch83-the-target-description.md) |
| 84 | [SelectionDAG: Building and Legalizing](chapters/part-14-backend/ch84-selectiondag-building-and-legalizing.md) |
| 85 | [SelectionDAG: Combining and Selecting](chapters/part-14-backend/ch85-selectiondag-combining-and-selecting.md) |
| 86 | [GlobalISel](chapters/part-14-backend/ch86-globalisel.md) |
| 87 | [Inline Assembly Lowering](chapters/part-14-backend/ch87-inline-assembly-lowering.md) |
| 88 | [The Machine IR](chapters/part-14-backend/ch88-the-machine-ir.md) |
| 89 | [Pre-RegAlloc Passes](chapters/part-14-backend/ch89-pre-regalloc-passes.md) |
| 90 | [Register Allocation](chapters/part-14-backend/ch90-register-allocation.md) |
| 91 | [The Machine Pipeliner](chapters/part-14-backend/ch91-the-machine-pipeliner.md) |
| 92 | [The Machine Outliner](chapters/part-14-backend/ch92-the-machine-outliner.md) |
| 93 | [Post-RegAlloc and Pre-Emit](chapters/part-14-backend/ch93-post-regalloc-and-pre-emit.md) |
| 94 | [The MC Layer and MIR Test Infrastructure](chapters/part-14-backend/ch94-the-mc-layer-and-mir-test-infrastructure.md) |

### Part XV — Targets *(~250 pp)*

| # | Chapter |
|---|---------|
| 95 | [The X86 Backend](chapters/part-15-targets/ch95-the-x86-backend.md) |
| 96 | [The AArch64 Backend](chapters/part-15-targets/ch96-the-aarch64-backend.md) |
| 97 | [The 32-bit ARM Backend](chapters/part-15-targets/ch97-the-32-bit-arm-backend.md) |
| 98 | [The RISC-V Backend Architecture](chapters/part-15-targets/ch98-the-riscv-backend-architecture.md) |
| 99 | [The RISC-V Vector Extension (RVV)](chapters/part-15-targets/ch99-the-riscv-vector-extension-rvv.md) |
| 100 | [RISC-V Bit-Manip, Crypto, and Custom Extensions](chapters/part-15-targets/ch100-riscv-bit-manip-crypto-custom.md) |
| 101 | [PowerPC, SystemZ, MIPS, SPARC, LoongArch](chapters/part-15-targets/ch101-powerpc-systemz-mips-sparc-loongarch.md) |
| 102 | [NVPTX and the CUDA Path](chapters/part-15-targets/ch102-nvptx-and-the-cuda-path.md) |
| 103 | [AMDGPU and the ROCm Path](chapters/part-15-targets/ch103-amdgpu-and-the-rocm-path.md) |
| 104 | [The SPIR-V Backend](chapters/part-15-targets/ch104-the-spirv-backend.md) |
| 105 | [DXIL and DirectX Shader Compilation](chapters/part-15-targets/ch105-dxil-and-directx-shader-compilation.md) |
| 106 | [WebAssembly and BPF](chapters/part-15-targets/ch106-webassembly-and-bpf.md) |
| 107 | [Embedded Targets](chapters/part-15-targets/ch107-embedded-targets.md) |

### Part XVI — JIT, Sanitizers, and Diagnostic Tools *(~210 pp)*

| # | Chapter |
|---|---------|
| 108 | [The ORC JIT](chapters/part-16-jit-sanitizers/ch108-the-orc-jit.md) |
| 109 | [JITLink](chapters/part-16-jit-sanitizers/ch109-jitlink.md) |
| 110 | [User-Space Sanitizers](chapters/part-16-jit-sanitizers/ch110-user-space-sanitizers.md) |
| 111 | [HWASan and MTE](chapters/part-16-jit-sanitizers/ch111-hwasan-and-mte.md) |
| 112 | [Production Allocators: Scudo and GWP-ASan](chapters/part-16-jit-sanitizers/ch112-production-allocators-scudo-gwp-asan.md) |
| 113 | [Kernel Sanitizers](chapters/part-16-jit-sanitizers/ch113-kernel-sanitizers.md) |
| 114 | [LibFuzzer and Coverage-Guided Fuzzing](chapters/part-16-jit-sanitizers/ch114-libfuzzer-and-coverage-guided-fuzzing.md) |
| 115 | [Source-Based Code Coverage](chapters/part-16-jit-sanitizers/ch115-source-based-code-coverage.md) |
| 116 | [LLDB Architecture](chapters/part-16-jit-sanitizers/ch116-lldb-architecture.md) |
| 117 | [DWARF and Debug Info](chapters/part-16-jit-sanitizers/ch117-dwarf-and-debug-info.md) |
| 118 | [BOLT and Post-Link Optimization](chapters/part-16-jit-sanitizers/ch118-bolt-and-post-link-optimization.md) |

### Part XVII — Runtime Libraries *(~110 pp)*

| # | Chapter |
|---|---------|
| 119 | [compiler-rt Builtins](chapters/part-17-runtime-libs/ch119-compiler-rt-builtins.md) |
| 120 | [libunwind](chapters/part-17-runtime-libs/ch120-libunwind.md) |
| 121 | [libc++](chapters/part-17-runtime-libs/ch121-libcxx.md) |
| 122 | [libc++abi](chapters/part-17-runtime-libs/ch122-libcxxabi.md) |
| 123 | [LLVM-libc](chapters/part-17-runtime-libs/ch123-llvm-libc.md) |
| 124 | [OpenMP and Offload Runtimes](chapters/part-17-runtime-libs/ch124-openmp-and-offload-runtimes.md) |

### Part XVIII — Flang *(~80 pp)*

| # | Chapter |
|---|---------|
| 125 | [Flang Architecture and Driver](chapters/part-18-flang/ch125-flang-architecture-and-driver.md) |
| 126 | [The HLFIR and FIR Dialects](chapters/part-18-flang/ch126-the-hlfir-and-fir-dialects.md) |
| 127 | [Flang OpenMP and OpenACC](chapters/part-18-flang/ch127-flang-openmp-and-openacc.md) |
| 128 | [Flang Codegen and Runtime](chapters/part-18-flang/ch128-flang-codegen-and-runtime.md) |

### Part XIX — MLIR Foundations *(~150 pp)*

| # | Chapter |
|---|---------|
| 129 | [MLIR Philosophy](chapters/part-19-mlir-foundations/ch129-mlir-philosophy.md) |
| 130 | [MLIR IR Structure](chapters/part-19-mlir-foundations/ch130-mlir-ir-structure.md) |
| 131 | [The Type and Attribute Systems](chapters/part-19-mlir-foundations/ch131-the-type-and-attribute-systems.md) |
| 132 | [Defining Dialects with ODS](chapters/part-19-mlir-foundations/ch132-defining-dialects-with-ods.md) |
| 133 | [Op Interfaces and Traits](chapters/part-19-mlir-foundations/ch133-op-interfaces-and-traits.md) |
| 134 | [The MLIR C++ API](chapters/part-19-mlir-foundations/ch134-the-mlir-cpp-api.md) |
| 135 | [PDL and PDLL](chapters/part-19-mlir-foundations/ch135-pdl-and-pdll.md) |
| 136 | [MLIR Bytecode and Serialization](chapters/part-19-mlir-foundations/ch136-mlir-bytecode-and-serialization.md) |

### Part XX — In-Tree Dialects *(~190 pp)*

| # | Chapter |
|---|---------|
| 137 | [Core Dialects](chapters/part-20-in-tree-dialects/ch137-core-dialects.md) |
| 138 | [Memory Dialects](chapters/part-20-in-tree-dialects/ch138-memory-dialects.md) |
| 139 | [Tensor and Linalg](chapters/part-20-in-tree-dialects/ch139-tensor-and-linalg.md) |
| 140 | [Affine and SCF](chapters/part-20-in-tree-dialects/ch140-affine-and-scf.md) |
| 141 | [Vector and Sparse](chapters/part-20-in-tree-dialects/ch141-vector-and-sparse.md) |
| 142 | [GPU Dialect Family](chapters/part-20-in-tree-dialects/ch142-gpu-dialect-family.md) |
| 143 | [SPIR-V Dialect](chapters/part-20-in-tree-dialects/ch143-spirv-dialect.md) |
| 144 | [Hardware Vector Dialects](chapters/part-20-in-tree-dialects/ch144-hardware-vector-dialects.md) |
| 145 | [LLVM Dialect](chapters/part-20-in-tree-dialects/ch145-llvm-dialect.md) |
| 146 | [Async, OpenMP, OpenACC, DLTI, EmitC](chapters/part-20-in-tree-dialects/ch146-async-openmp-openacc-dlti-emitc.md) |

### Part XXI — MLIR Transformations *(~110 pp)*

| # | Chapter |
|---|---------|
| 147 | [Pattern Rewriting](chapters/part-21-mlir-transformations/ch147-pattern-rewriting.md) |
| 148 | [Dialect Conversion](chapters/part-21-mlir-transformations/ch148-dialect-conversion.md) |
| 149 | [The Pass Infrastructure](chapters/part-21-mlir-transformations/ch149-the-pass-infrastructure.md) |
| 150 | [The Transform Dialect](chapters/part-21-mlir-transformations/ch150-the-transform-dialect.md) |
| 151 | [Bufferization Deep Dive](chapters/part-21-mlir-transformations/ch151-bufferization-deep-dive.md) |
| 152 | [Lowering Pipelines](chapters/part-21-mlir-transformations/ch152-lowering-pipelines.md) |

### Part XXII — XLA and the OpenXLA Stack *(~120 pp)*

| # | Chapter |
|---|---------|
| 153 | [XLA Architecture](chapters/part-22-xla-openxla/ch153-xla-architecture.md) |
| 154 | [HLO and StableHLO](chapters/part-22-xla-openxla/ch154-hlo-and-stablehlo.md) |
| 155 | [XLA:CPU](chapters/part-22-xla-openxla/ch155-xla-cpu.md) |
| 156 | [XLA:GPU](chapters/part-22-xla-openxla/ch156-xla-gpu.md) |
| 157 | [PJRT — The Plugin Runtime Interface](chapters/part-22-xla-openxla/ch157-pjrt-the-plugin-runtime-interface.md) |
| 158 | [SPMD, GSPMD, and Auto-Sharding](chapters/part-22-xla-openxla/ch158-spmd-gspmd-and-auto-sharding.md) |

### Part XXIII — MLIR in Production *(~160 pp)*

| # | Chapter |
|---|---------|
| 159 | [Building a Domain-Specific Compiler](chapters/part-23-mlir-production/ch159-building-a-domain-specific-compiler.md) |
| 160 | [MLIR Python Bindings](chapters/part-23-mlir-production/ch160-mlir-python-bindings.md) |
| 161 | [torch-mlir, ONNX-MLIR, and JAX/TF Bridges](chapters/part-23-mlir-production/ch161-torch-mlir-onnx-mlir-and-jax-tf-bridges.md) |
| 162 | [IREE — A Deployment Compiler](chapters/part-23-mlir-production/ch162-iree-a-deployment-compiler.md) |
| 163 | [Triton — A Compiler for GPU Kernels](chapters/part-23-mlir-production/ch163-triton-a-compiler-for-gpu-kernels.md) |
| 164 | [CUDA Tile IR](chapters/part-23-mlir-production/ch164-cuda-tile-ir.md) |
| 165 | [GPU Compilation Through MLIR](chapters/part-23-mlir-production/ch165-gpu-compilation-through-mlir.md) |
| 166 | [Mojo, Polygeist, Enzyme-MLIR, and Beyond](chapters/part-23-mlir-production/ch166-mojo-polygeist-enzyme-mlir-and-beyond.md) |

### Part XXIV — Verified Compilation *(~110 pp)* `[THEORETICAL]`

| # | Chapter |
|---|---------|
| 167 | [Operational Semantics and Program Logics](chapters/part-24-verified-compilation/ch167-operational-semantics-and-program-logics.md) |
| 168 | [CompCert](chapters/part-24-verified-compilation/ch168-compcert.md) |
| 169 | [Vellvm and Formalizing LLVM IR](chapters/part-24-verified-compilation/ch169-vellvm-and-formalizing-llvm-ir.md) |
| 170 | [Alive2 and Translation Validation](chapters/part-24-verified-compilation/ch170-alive2-and-translation-validation.md) |
| 171 | [The Undef/Poison Story Formally](chapters/part-24-verified-compilation/ch171-the-undef-poison-story-formally.md) |

### Part XXV — Operations, Bindings, and Contribution *(~100 pp)*

| # | Chapter |
|---|---------|
| 172 | [Testing in LLVM and MLIR](chapters/part-25-operations-contribution/ch172-testing-in-llvm-and-mlir.md) |
| 173 | [Debugging the Compiler](chapters/part-25-operations-contribution/ch173-debugging-the-compiler.md) |
| 174 | [Performance Engineering](chapters/part-25-operations-contribution/ch174-performance-engineering.md) |
| 175 | [Language Bindings](chapters/part-25-operations-contribution/ch175-language-bindings.md) |
| 176 | [Contributing to LLVM](chapters/part-25-operations-contribution/ch176-contributing-to-llvm.md) |

### Part XXVI — Ecosystem and Frontiers *(~94 pp)*

| # | Chapter |
|---|---------|
| 177 | [rustc: Architecture, MIR, and Codegen Backends](chapters/part-26-ecosystem-frontiers/ch177-rustc-architecture-mir-codegen-backends.md) |
| 178 | [The Rust Compiler Ecosystem](chapters/part-26-ecosystem-frontiers/ch178-the-rust-compiler-ecosystem.md) |
| 179 | [LLVM/MLIR for AI: The Full Stack](chapters/part-26-ecosystem-frontiers/ch179-llvm-mlir-for-ai-the-full-stack.md) |
| 180 | [AI-Guided Compilation](chapters/part-26-ecosystem-frontiers/ch180-ai-guided-compilation.md) |
| 181 | [Formal Verification in Practice](chapters/part-26-ecosystem-frontiers/ch181-formal-verification-in-practice.md) |
| 182 | [Language Tooling: Parsers, Lexers, and Syntax Trees](chapters/part-26-ecosystem-frontiers/ch182-language-tooling-parsers-lexers-syntax-trees.md) |

---

### Appendices *(~95 pp)*

| | Appendix |
|-|---------|
| A | [LLVM IR Quick Reference](appendices/appendix-a-llvm-ir-quick-reference.md) |
| B | [MLIR Dialect Quick Reference](appendices/appendix-b-mlir-dialect-quick-reference.md) |
| C | [Command-Line Tools Cheat Sheet](appendices/appendix-c-command-line-tools-cheat-sheet.md) |
| D | [Migration Notes: LLVM 18 → 22](appendices/appendix-d-migration-notes-llvm-18-to-22.md) |
| E | [Glossary](appendices/appendix-e-glossary.md) |
| F | [Object File Format Reference](appendices/appendix-f-object-file-format-reference.md) |
| G | [DWARF Debug Info Reference](appendices/appendix-g-dwarf-debug-info-reference.md) |
| H | [The C++ ABI: Itanium and Microsoft](appendices/appendix-h-the-cpp-abi-itanium-and-microsoft.md) |

---

## Stats

| | |
|-|-|
| Parts | 26 |
| Chapters | 182 |
| Appendices | 8 |
| Total items | 190 |
| Estimated pages | ~3,000 |
| LLVM version | 22.1.x |
| Rust edition | 2024 (1.85+) |
| License | [CC BY 4.0](LICENSE) |

---

@copyright jreuben11
