# LLVM, Clang & MLIR — The Expert Programming Reference

*With Compiler Theory, Type Theory, Polyhedral Theory, and Verified Compilation*

**Targeting LLVM 22.1.x · Clang 22 · MLIR in-tree · OpenXLA · ClangIR · April 2026**

---

## About This Book

~3,050 pages of expert-level material spanning 239 chapters and 10 appendices across 31 parts. The book is organised around **five structural arcs**:

1. **LLVM and Clang implementation** (Parts I, IV–XVIII, XXV) — the compiler infrastructure itself: LLVM IR (Part IV), the analysis and middle-end pass pipeline (Part X), LTO/ThinLTO and LLD (Part XIII), SelectionDAG, GlobalISel, the backend, and all target ports including NVPTX/AMDGPU/RISC-V (Parts XIV–XV), JIT (ORC/JITLink), sanitizers, BOLT, and diagnostic tools (Part XVI), runtime libraries — compiler-rt, libunwind, libc++, OpenMP (Part XVII); Clang's full internals: driver, SourceManager, preprocessor, parser, Sema, modules, AST, CodeGen, and C++ ABI (Parts V–VI); Clang tooling and heterogeneous compilation — CUDA, HIP, SYCL, OpenCL, HLSL, the static analyzer, libtooling, and clangd (Part VII); ClangIR (CIR) — the Clang-level MLIR dialect (Part VIII); building a custom compiler frontend with LLVM (Part IX); Flang — the LLVM Fortran compiler (Part XVIII); and tooling, language bindings, and contributing to LLVM (Part XXV).
2. **Compiler theory** (Part II, ~120 pp) — lexical analysis, parsing theory, SSA construction, the lattice-based dataflow framework, and classical optimization theory. Anchored to Aho/Lam/Sethi/Ullman (*Dragon Book* 2e), Cooper & Torczon (*Engineering a Compiler* 3e), Appel (*Modern Compiler Implementation*), and Muchnick.
3. **Type theory** (Part III, ~80 pp) — lambda calculus, System F, Hindley-Milner and Algorithm W, dependent types (Martin-Löf / CoC), linear and affine types, refinement types, and a capstone chapter mapping theory to LLVM and MLIR. Anchored to Pierce (*TAPL*, *ATTAPL*), Harper (*PFPL*), Mitchell, and Reynolds.
4. **Polyhedral model and MLIR** (Parts XI–XII, XIX–XXIII) — polyhedral theory from Presburger arithmetic through the Pluto scheduling algorithm and CLooG code generation (Part XI, ~80 pp, anchored to Bondhugula's PhD thesis, Feautrier's papers, Verdoolaege's ISL papers, and Grosser et al.); Polly (Part XII); and the entire MLIR stack: foundations, in-tree dialects, transformations, XLA/OpenXLA, and production deployment including Triton, IREE, torch-mlir, and Mojo (Parts XIX–XXIII, ~730 pp total).
5. **Verified compilation and ecosystem frontiers** (Parts XXIV–XXXI) — formal semantics, CompCert, Vellvm, Alive2, and the undef/poison story (Part XXIV, ~110 pp, anchored to Leroy's CompCert papers, Zhao/Zdancewic/Nagarakatte on Vellvm, and Lopes/Lee/Hur on Alive2); adjacent ecosystems — Rust, AI toolchains, formal verification, and language tooling (Part XXVI); mathematical foundations for compiler engineers including proof assistants, category theory, and denotational semantics (Part XXVII); language ecosystems — CIRCT, quantum compilation, Swift SIL, Julia, Zig, safety-critical toolchains (Part XXVIII); compiler tooling and binary analysis (Part XXIX); AI-first programming language design (Part XXX); and AI-native compilation and frontier systems — GPU kernel DSLs, JAX ecosystem, weight-space geometry, mechanistic interpretability, evolutionary architecture search, and formal self-improvement theory (Part XXXI).

### Estimated Page Distribution

| Volume | Parts | Pages |
|--------|-------|-------|
| Vol 1 — Foundations and Theory | I, II, III, IX, XI | ~445 pp |
| Vol 2 — LLVM Core | IV, X, XII–XVI | ~1,211 pp |
| Vol 3 — Clang and Runtimes | V–VIII, XVII–XVIII | ~870 pp |
| Vol 4 — MLIR and ML Compilation | XIX–XXIII | ~730 pp |
| Vol 5 — Verified Compilation + Operations + Ecosystem + Appendices | XXIV–XXVI + A–J | ~445 pp |
| Vol 6 — Mathematical Foundations + Language Ecosystems + Tooling + AI-First PLs + Frontier AI | XXVII–XXXI | ~709 pp |
| **Total** | **31 parts + 10 appendices** | **~4,412 pp target** |

| | |
|-|-|
| Parts | 31 |
| Chapters | 239 |
| Appendices | 10 |
| Total items | 249 |
| Estimated pages | ~3,050 |
| LLVM version | 22.1.x |
| Rust edition | 2024 (1.85+) |
| License | [CC BY 4.0](LICENSE) |

---

## Target Audience

Expert audience. Assumed: C++ proficiency, familiarity with compilers at the level of having used Clang/GCC and read assembly output. Not assumed: prior LLVM API experience, type theory, or formal methods. The theoretical parts (II, III, XI, XXIV, XXVII) build from first principles while keeping an eye on how the theory maps to concrete LLVM and MLIR code.

---

## Verification Methodology

**Practical chapters** (Parts I, IV–X, XII–XVIII, XIX–XXIII, XXV–XXIX, XXXI): all code verified against the respective toolchain versions. LLVM code verified against LLVM 22.1.x (`clang --version` → `22.x`; `llvm-config --version` → `22.x`). Source file cross-references use paths valid in the LLVM 22 monorepo. Code is emitted and checked with `clang -emit-llvm -S` or `mlir-opt` as appropriate. Rust code (Part XXVI) targets Edition 2024 (Rust 1.85+, `edition = "2024"` in every `Cargo.toml`). Python/JAX code (Parts XXX–XXXI) verified against JAX 0.4.x, PyTorch 2.x, and TransformerLens.

**Theoretical chapters** (Parts II, III, XI, XXIV, XXVII): verified against the canonical literature cited per-chapter (Dragon Book, TAPL/PFPL, Bondhugula thesis, CompCert/Vellvm/Alive2 papers, Kolmogorov/Hutter/Schmidhuber for Part XXXI §216). No LLVM toolchain verification is applicable or attempted.

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

### Part II — Compiler Theory *(~120 pp)* `[THEORETICAL]`

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

### Part VII — Clang Tooling and Heterogeneous Compilation *(~140 pp)*

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

### Part IX — Building a Custom Compiler Frontend *(~80 pp)*

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

### Part XVI — JIT, Sanitizers, and Diagnostic Tools *(~282 pp)*

#### JIT and Dynamic Runtimes

| # | Chapter |
|---|---------|
| 108 | [The ORC JIT](chapters/part-16-jit-sanitizers/ch108-the-orc-jit.md) |
| 109 | [JITLink](chapters/part-16-jit-sanitizers/ch109-jitlink.md) |
| 219 | [ORC JIT in Production: clang-repl, LLDB, PostgreSQL, Numba, and Halide](chapters/part-16-jit-sanitizers/ch219-orc-jit-in-production.md) |
| 220 | [Runtime Self-Modification: Source Introspection, Incremental Recompilation, and ORC Hot-Loading](chapters/part-16-jit-sanitizers/ch220-runtime-self-modification.md) |
| 221 | [Speculative Optimization, Inline Caches, and Deoptimization](chapters/part-16-jit-sanitizers/ch221-speculative-optimization-inline-caches.md) |
| 222 | [Plugin Architecture, Dynamic Loading, and ABI Stability](chapters/part-16-jit-sanitizers/ch222-plugin-architecture-dynamic-loading.md) |
| 228 | [Virtual Machine Design: Bytecode Interpreters, GC Strategies, and Object Models](chapters/part-16-jit-sanitizers/ch228-virtual-machine-design.md) |

#### Sanitizers and Diagnostic Tools

| # | Chapter |
|---|---------|
| 110 | [User-Space Sanitizers](chapters/part-16-jit-sanitizers/ch110-user-space-sanitizers.md) |
| 111 | [HWASan and MTE](chapters/part-16-jit-sanitizers/ch111-hwasan-and-mte.md) |
| 112 | [Production Allocators: Scudo and GWP-ASan](chapters/part-16-jit-sanitizers/ch112-production-allocators-scudo-gwp-asan.md) |
| 113 | [Kernel Sanitizers](chapters/part-16-jit-sanitizers/ch113-kernel-sanitizers.md) |
| 114 | [LibFuzzer and Coverage-Guided Fuzzing](chapters/part-16-jit-sanitizers/ch114-libfuzzer-and-coverage-guided-fuzzing.md) |
| 115 | [Source-Based Code Coverage](chapters/part-16-jit-sanitizers/ch115-source-based-code-coverage.md) |
| 116 | [LLDB Architecture](chapters/part-16-jit-sanitizers/ch116-lldb-architecture.md) |
| 117 | [DWARF and Debug Info](chapters/part-16-jit-sanitizers/ch117-dwarf-and-debug-info.md) |
| 118 | [BOLT and Post-Link Optimization](chapters/part-16-jit-sanitizers/ch118-bolt-and-post-link-optimization.md) |
| 232 | [LLVM XRay: Low-Overhead Function Tracing](chapters/part-16-jit-sanitizers/ch232-llvm-xray-function-tracing.md) |

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

### Part XXIII — MLIR and ML Compilation Frameworks *(~208 pp)*

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
| 202 | [Apache TVM: ML Operator Compiler](chapters/part-23-mlir-production/ch202-apache-tvm-ml-operator-compiler.md) |
| 229 | [torch.compile: TorchDynamo, AOTAutograd, and TorchInductor](chapters/part-23-mlir-production/ch229-torch-compile-torchdynamo-aotautograd-torchinductor.md) |

### Part XXIV — Verified Compilation *(~110 pp)* `[THEORETICAL]`

| # | Chapter |
|---|---------|
| 167 | [Operational Semantics and Program Logics](chapters/part-24-verified-compilation/ch167-operational-semantics-and-program-logics.md) |
| 168 | [CompCert](chapters/part-24-verified-compilation/ch168-compcert.md) |
| 169 | [Vellvm and Formalizing LLVM IR](chapters/part-24-verified-compilation/ch169-vellvm-and-formalizing-llvm-ir.md) |
| 170 | [Alive2 and Translation Validation](chapters/part-24-verified-compilation/ch170-alive2-and-translation-validation.md) |
| 171 | [The Undef/Poison Story Formally](chapters/part-24-verified-compilation/ch171-the-undef-poison-story-formally.md) |

### Part XXV — Tooling, Bindings, and Contributing *(~100 pp)*

| # | Chapter |
|---|---------|
| 172 | [Testing in LLVM and MLIR](chapters/part-25-operations-contribution/ch172-testing-in-llvm-and-mlir.md) |
| 173 | [Debugging the Compiler](chapters/part-25-operations-contribution/ch173-debugging-the-compiler.md) |
| 174 | [Performance Engineering](chapters/part-25-operations-contribution/ch174-performance-engineering.md) |
| 175 | [Language Bindings](chapters/part-25-operations-contribution/ch175-language-bindings.md) |
| 176 | [Contributing to LLVM](chapters/part-25-operations-contribution/ch176-contributing-to-llvm.md) |

### Part XXVI — Adjacent Ecosystems *(~108 pp)*

| # | Chapter |
|---|---------|
| 177 | [rustc: Architecture, MIR, and Codegen Backends](chapters/part-26-ecosystem-frontiers/ch177-rustc-architecture-mir-codegen-backends.md) |
| 178 | [The Rust Compiler Ecosystem](chapters/part-26-ecosystem-frontiers/ch178-the-rust-compiler-ecosystem.md) |
| 179 | [LLVM/MLIR for AI: The Full Stack](chapters/part-26-ecosystem-frontiers/ch179-llvm-mlir-for-ai-the-full-stack.md) |
| 180 | [AI-Guided Compilation](chapters/part-26-ecosystem-frontiers/ch180-ai-guided-compilation.md) |
| 181 | [Formal Verification in Practice](chapters/part-26-ecosystem-frontiers/ch181-formal-verification-in-practice.md) |
| 182 | [Language Tooling: Parsers, Lexers, and Syntax Trees](chapters/part-26-ecosystem-frontiers/ch182-language-tooling-parsers-lexers-syntax-trees.md) |
| 183 | [Modern C++ for Compiler Development: C++23, Contracts, and Reflection](chapters/part-26-ecosystem-frontiers/ch183-modern-cpp-for-compiler-development.md) |

### Part XXVII — Mathematical Foundations for Compiler Engineers *(~120 pp)* `[THEORETICAL]`

| # | Chapter |
|---|---------|
| 184 | [Proof Assistant Internals: Lean 4, Coq/Rocq, Isabelle/HOL, and Agda](chapters/part-27-mathematical-foundations/ch184-proof-assistant-internals-lean4-coq-isabelle.md) |
| 185 | [Mathematical Logic and Model Theory for Compiler Engineers](chapters/part-27-mathematical-foundations/ch185-mathematical-logic-model-theory.md) |
| 186 | [Verified Hardware: CHERI Capabilities and the seL4 Microkernel](chapters/part-27-mathematical-foundations/ch186-verified-hardware-cheri-sel4.md) |
| 187 | [Commutative Algebra and Its Applications in Compilation](chapters/part-27-mathematical-foundations/ch187-commutative-algebra-compilation.md) |
| 188 | [Category Theory for Compiler Engineers](chapters/part-27-mathematical-foundations/ch188-category-theory-compiler-engineers.md) |
| 189 | [Denotational Semantics and Domain Theory](chapters/part-27-mathematical-foundations/ch189-denotational-semantics-domain-theory.md) |

### Part XXVIII — Language Ecosystems and Engineering Practice *(~180 pp)*

#### Languages Using LLVM

| # | Chapter |
|---|---------|
| 192 | [Swift SIL: Ownership, Optimization, and Influence on MLIR](chapters/part-28-language-ecosystems/ch192-swift-sil-ownership-mlir.md) |
| 193 | [Julia: Type-Inference-Driven LLVM Specialization](chapters/part-28-language-ecosystems/ch193-julia-llvm-specialization.md) |
| 194 | [Zig: Comptime Metaprogramming and LLVM IR Generation](chapters/part-28-language-ecosystems/ch194-zig-comptime-llvm.md) |
| 235 | [GHC's LLVM Backend: Haskell to Native via LLVM](chapters/part-28-language-ecosystems/ch235-ghc-llvm-backend.md) |
| 237 | [TinyGo: Compiling Go for Embedded Systems and WebAssembly via LLVM](chapters/part-28-language-ecosystems/ch237-tinygo-go-embedded-llvm.md) |
| 238 | [Kotlin/Native: Compiling Kotlin to Native Code via LLVM](chapters/part-28-language-ecosystems/ch238-kotlin-native-llvm.md) |
| 239 | [ISPC: An SPMD Compiler for CPU SIMD via LLVM](chapters/part-28-language-ecosystems/ch239-ispc-spmd-compiler.md) |

#### Specialized Compilers and Engineering Practice

| # | Chapter |
|---|---------|
| 190 | [CIRCT: Circuit IR Compilers and Tools](chapters/part-28-language-ecosystems/ch190-circt-hardware-compiler.md) |
| 191 | [Quantum Compilation: QIR, QUIR, and MLIR Quantum Dialects](chapters/part-28-language-ecosystems/ch191-quantum-compilation-qir-quir.md) |
| 195 | [Safety-Critical Toolchain Qualification: DO-178C, ISO 26262, and Ferrocene](chapters/part-28-language-ecosystems/ch195-safety-critical-qualification.md) |
| 196 | [Cross-Language ABI Interoperability: Binding Generators and UniFFI](chapters/part-28-language-ecosystems/ch196-cross-language-abi.md) |
| 230 | [Cranelift: A Lightweight JIT for WebAssembly and Rust](chapters/part-28-language-ecosystems/ch230-cranelift-lightweight-jit.md) |
| 231 | [GraalVM: Native Image, Truffle Interpreters, and Polyglot Runtimes](chapters/part-28-language-ecosystems/ch231-graalvm-native-image-truffle.md) |
| 233 | [Emscripten: C/C++ to WebAssembly via LLVM](chapters/part-28-language-ecosystems/ch233-emscripten-c-cpp-to-webassembly.md) |
| 236 | [Android NDK: Cross-Compiling C/C++ for Android with LLVM](chapters/part-28-language-ecosystems/ch236-android-ndk-cross-compiling.md) |

### Part XXIX — Compiler Tooling, Kernel Integration, and Binary Analysis *(~72 pp)*

| # | Chapter |
|---|---------|
| 197 | [Clang Plugin System](chapters/part-29-compiler-tooling/ch197-clang-plugin-system.md) |
| 198 | [Value Tracking Infrastructure](chapters/part-29-compiler-tooling/ch198-value-tracking-infrastructure.md) |
| 199 | [llvm-mca: Static Performance Analysis](chapters/part-29-compiler-tooling/ch199-llvm-mca-static-performance-analysis.md) |
| 200 | [Linux Kernel Compilation with LLVM/Clang](chapters/part-29-compiler-tooling/ch200-linux-kernel-compilation-llvm-clang.md) |
| 201 | [Binary Lifting to LLVM IR](chapters/part-29-compiler-tooling/ch201-binary-lifting-to-llvm-ir.md) |
| 234 | [KLEE: Symbolic Execution of LLVM IR](chapters/part-29-compiler-tooling/ch234-klee-symbolic-execution.md) |

### Part XXX — AI-First Programming Language Design *(~60 pp)*

| # | Chapter |
|---|---------|
| 203 | [AI-First PL Principles and Landscape](chapters/part-30-AI-first-PL-design/ch203-ai-first-pl-principles.md) |
| 204 | [Formal Language Specification for AI-First PLs](chapters/part-30-AI-first-PL-design/ch204-formal-language-specification.md) |
| 205 | [Transformer Model Development PLs](chapters/part-30-AI-first-PL-design/ch205-transformer-model-development-pls.md) |
| 206 | [Multi-Agent PLs, AI-First SDLC, Security, and Paradigm Failures](chapters/part-30-AI-first-PL-design/ch206-multi-agent-pls-sdlc-security.md) |
| 207 | [Reflective Code, Open Problems, and Build Roadmap](chapters/part-30-AI-first-PL-design/ch207-reflective-code-open-problems-roadmap.md) |

### Part XXXI — AI-Native Compilation and Frontier Systems *(~192 pp)*

| # | Chapter |
|---|---------|
| 208 | [GPU Kernel DSLs: Triton, Helion, and Gluon](chapters/part-31-frontier-ai-evolution/ch208-gpu-kernel-dsls-triton-helion-gluon.md) |
| 209 | [CUTLASS, Thrust, CuTe, and TileIR: GPU Parallel Primitives and Layout Algebra](chapters/part-31-frontier-ai-evolution/ch209-cutlass-thrust-cute-tileir.md) |
| 210 | [The JAX Ecosystem: A Functional Neural Compilation Stack](chapters/part-31-frontier-ai-evolution/ch210-jax-ecosystem-functional-neural-compilation-stack.md) |
| 211 | [Neural Programs as Compiled Artifacts: The Self-Aware Execution Stack](chapters/part-31-frontier-ai-evolution/ch211-neural-programs-compiled-artifacts.md) |
| 212 | [Weights as a Programming Substrate](chapters/part-31-frontier-ai-evolution/ch212-weights-as-programming-substrate.md) |
| 213 | [Mechanistic Interpretability Infrastructure](chapters/part-31-frontier-ai-evolution/ch213-mechanistic-interpretability.md) |
| 214 | [Gradient-Based Self-Modification: Model Editing, Meta-Learning, and Test-Time Adaptation](chapters/part-31-frontier-ai-evolution/ch214-gradient-based-self-modification.md) |
| 215 | [Evolutionary Architecture Search](chapters/part-31-frontier-ai-evolution/ch215-evolutionary-architecture-search.md) |
| 216 | [Formal Self-Improvement Theory](chapters/part-31-frontier-ai-evolution/ch216-formal-self-improvement-theory.md) |
| 217 | [Self-Reflective Inference and Architecture Introspection](chapters/part-31-frontier-ai-evolution/ch217-self-reflective-inference.md) |
| 218 | [Self-Improvement Fitness Functions and Capability Assessment](chapters/part-31-frontier-ai-evolution/ch218-self-improvement-fitness-functions.md) |
| 223 | [Verification-Guided Pass Selection: LLM-VeriOpt and the Alive2 Reward Loop](chapters/part-31-frontier-ai-evolution/ch223-verification-guided-pass-selection.md) |
| 224 | [Imitation Learning for Compiler Heuristics: BC-Max and Behavioral Cloning](chapters/part-31-frontier-ai-evolution/ch224-imitation-learning-compiler-heuristics.md) |
| 225 | [Knowledge-Infused Evolutionary Search: Pass Synergy Graphs and ECCO](chapters/part-31-frontier-ai-evolution/ch225-knowledge-infused-evolutionary-search.md) |
| 226 | [Hierarchical Reinforcement Learning for Register Allocation and Code Optimization](chapters/part-31-frontier-ai-evolution/ch226-hierarchical-rl-register-allocation.md) |
| 227 | [LLM-Guided Polyhedral Optimization: LOOPer and Agentic Auto-Scheduling](chapters/part-31-frontier-ai-evolution/ch227-llm-guided-polyhedral-optimization.md) |

---

### Appendices *(~115 pp)*

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
| I | [LLVM and GCC: A Structural Comparison](appendices/appendix-i-llvm-and-gcc-structural-comparison.md) |
| J | [Grammar Notation Reference: BNF, EBNF, PEG, and Modern Alternatives](appendices/appendix-j-grammar-notation-reference.md) |

---

**License:** [CC BY 4.0](LICENSE) · **Copyright:** © 2026 [jreuben11](https://github.com/jreuben11)
