# Book Progress Checklist
*Last updated: 2026-04-29 (Part XXVI planned — 4 new chapters to write). Read this first after any context compaction.*

## Status Legend
- `[ ]` Not started
- `[~]` In progress
- `[x]` Written and committed

---

## Current Focus
**Part XXVI — Ecosystem and Frontiers** (4 new chapters, Ch177–Ch180)
- 184/184 original items complete; 4 new chapters planned, 0 written
- Write sequentially: Ch177 → Ch178 → Ch179 → Ch180

---

## Part I — Foundations *(~85 pp, 5 ch)*
- [x] Ch01 — The LLVM Project
- [x] Ch02 — Building LLVM from Source
- [x] Ch03 — The Compilation Pipeline
- [x] Ch04 — The LLVM C++ API
- [x] Ch05 — LLVM as a Library

## Part II — Compiler Theory and Foundations *(~120 pp, 6 ch)* [THEORETICAL]
- [x] Ch06 — Lexical Analysis
- [x] Ch07 — Parsing Theory
- [x] Ch08 — Semantic Analysis Foundations
- [x] Ch09 — Intermediate Representations and SSA Construction
- [x] Ch10 — Dataflow Analysis: The Lattice Framework
- [x] Ch11 — Classical Optimization Theory

## Part III — Type Theory *(~80 pp, 4 ch)* [THEORETICAL]
- [x] Ch12 — Lambda Calculus and Simple Types
- [x] Ch13 — Polymorphism and Type Inference
- [x] Ch14 — Advanced Type Systems
- [x] Ch15 — Type Theory in Practice: From Theory to LLVM and MLIR

## Part IV — LLVM IR *(~225 pp, 12 ch)*
- [x] Ch16 — IR Structure
- [x] Ch17 — The Type System
- [x] Ch18 — Constants, Globals, and Linkage
- [x] Ch19 — Instructions I — Arithmetic and Memory
- [x] Ch20 — Instructions II — Control Flow and Aggregates
- [x] Ch21 — SSA, Dominance, and Loops
- [x] Ch22 — Metadata and Debug Info
- [x] Ch23 — Attributes, Calling Conventions, and the ABI
- [x] Ch24 — Intrinsics
- [x] Ch25 — Inline Assembly
- [x] Ch26 — Exception Handling
- [x] Ch27 — Coroutines and Atomics

## Part V — Clang Internals: Frontend Pipeline *(~210 pp, 11 ch)*
- [x] Ch28 — The Clang Driver
- [x] Ch29 — SourceManager, FileEntry, SourceLocation
- [x] Ch30 — The Diagnostic Engine
- [x] Ch31 — The Lexer and Preprocessor
- [x] Ch32 — The Parser
- [x] Ch33 — Sema I — Names, Lookups, and Conversions
- [x] Ch34 — Sema II — Templates, Concepts, and Constraints
- [x] Ch35 — The Constant Evaluator
- [x] Ch36 — The Clang AST in Depth
- [x] Ch37 — C++ Modules Implementation
- [x] Ch38 — Code Completion and clangd Foundations

## Part VI — Clang Internals: Codegen and ABI *(~120 pp, 6 ch)*
- [x] Ch39 — CodeGenModule and CodeGenFunction
- [x] Ch40 — Lowering Statements and Expressions
- [x] Ch41 — Calls, the ABI Boundary, and Builtins
- [x] Ch42 — C++ ABI Lowering: Itanium
- [x] Ch43 — C++ ABI Lowering: Microsoft
- [x] Ch44 — Coroutine Lowering in Clang

## Part VII — Clang as a Multi-Language Compiler *(~140 pp, 7 ch)*
- [x] Ch45 — The Static Analyzer
- [x] Ch46 — libtooling and AST Matchers
- [x] Ch47 — clangd, clang-tidy, clang-format, clang-refactor
- [x] Ch48 — Clang as a CUDA Compiler
- [x] Ch49 — Clang as a HIP Compiler
- [x] Ch50 — Clang as SYCL, OpenCL, and OpenMP-Offload
- [x] Ch51 — Clang as an HLSL Compiler

## Part VIII — ClangIR (CIR) *(~50 pp, 3 ch)*
- [x] Ch52 — ClangIR Architecture
- [x] Ch53 — CIR Generation from AST
- [x] Ch54 — CIR Lowering and Analysis

## Part IX — Frontend Authoring (Building Your Own) *(~80 pp, 4 ch)*
- [x] Ch55 — Building a Frontend
- [x] Ch56 — Lowering AST to IR
- [x] Ch57 — Lowering High-Level Constructs
- [x] Ch58 — Language Runtime Concerns

## Part X — Analysis and the Middle-End *(~210 pp, 11 ch)* ✓ COMPLETE
- [x] Ch59 — The New Pass Manager
- [x] Ch60 — Writing a Pass
- [x] Ch61 — Foundational Analyses
- [x] Ch62 — Scalar Optimizations
- [x] Ch63 — Loop Optimizations
- [x] Ch64 — Vectorization Deep Dive
- [x] Ch65 — Inter-Procedural Optimizations
- [x] Ch66 — The ML Inliner and ML Regalloc
- [x] Ch67 — Profile-Guided Optimization
- [x] Ch68 — Hardening and Mitigations
- [x] Ch69 — Whole-Program Devirtualization

## Part XI — Polyhedral Theory *(~80 pp, 4 ch)* [THEORETICAL] ✓ COMPLETE
- [x] Ch70 — Foundations: Polyhedra and Integer Programming
- [x] Ch71 — The Polyhedral Model
- [x] Ch72 — Scheduling Algorithms
- [x] Ch73 — Code Generation from Polyhedral Schedules

## Part XII — Polly *(~50 pp, 3 ch)* ✓ COMPLETE
- [x] Ch74 — Polly Architecture
- [x] Ch75 — Polly Transformations
- [x] Ch76 — Polly in Practice

## Part XIII — Link-Time and Whole-Program *(~80 pp, 4 ch)* ✓ COMPLETE
- [x] Ch77 — LTO and ThinLTO
- [x] Ch78 — The LLVM Linker (LLD)
- [x] Ch79 — Linker Internals: GOT, PLT, TLS
- [x] Ch80 — llvm-cas and Content-Addressable Builds

## Part XIV — The Backend *(~280 pp, 14 ch)* ✓ COMPLETE
- [x] Ch81 — Backend Architecture
- [x] Ch82 — TableGen Deep Dive
- [x] Ch83 — The Target Description
- [x] Ch84 — SelectionDAG: Building and Legalizing
- [x] Ch85 — SelectionDAG: Combining and Selecting
- [x] Ch86 — GlobalISel
- [x] Ch87 — Inline Assembly Lowering
- [x] Ch88 — The Machine IR
- [x] Ch89 — Pre-RegAlloc Passes
- [x] Ch90 — Register Allocation
- [x] Ch91 — The Machine Pipeliner
- [x] Ch92 — The Machine Outliner
- [x] Ch93 — Post-RegAlloc and Pre-Emit
- [x] Ch94 — The MC Layer and MIR Test Infrastructure

## Part XV — Targets *(~250 pp, 13 ch)*
- [x] Ch95 — The X86 Backend
- [x] Ch96 — The AArch64 Backend
- [x] Ch97 — The 32-bit ARM Backend
- [x] Ch98 — The RISC-V Backend Architecture
- [x] Ch99 — The RISC-V Vector Extension (RVV)
- [x] Ch100 — RISC-V Bit-Manip, Crypto, and Custom Extensions
- [x] Ch101 — PowerPC, SystemZ, MIPS, SPARC, LoongArch
- [x] Ch102 — NVPTX and the CUDA Path
- [x] Ch103 — AMDGPU and the ROCm Path
- [x] Ch104 — The SPIR-V Backend
- [x] Ch105 — DXIL and DirectX Shader Compilation
- [x] Ch106 — WebAssembly and BPF
- [x] Ch107 — Embedded Targets

## Part XVI — JIT, Sanitizers, and Diagnostic Tools *(~210 pp, 11 ch)* ✓ COMPLETE
- [x] Ch108 — The ORC JIT
- [x] Ch109 — JITLink
- [x] Ch110 — User-Space Sanitizers
- [x] Ch111 — HWASan and MTE
- [x] Ch112 — Production Allocators: Scudo and GWP-ASan
- [x] Ch113 — Kernel Sanitizers
- [x] Ch114 — LibFuzzer and Coverage-Guided Fuzzing
- [x] Ch115 — Source-Based Code Coverage
- [x] Ch116 — LLDB Architecture
- [x] Ch117 — DWARF and Debug Info
- [x] Ch118 — BOLT and Post-Link Optimization

## Part XVII — Runtime Libraries *(~110 pp, 6 ch)* ✓ COMPLETE
- [x] Ch119 — compiler-rt Builtins
- [x] Ch120 — libunwind
- [x] Ch121 — libc++
- [x] Ch122 — libc++abi
- [x] Ch123 — LLVM-libc
- [x] Ch124 — OpenMP and Offload Runtimes

## Part XVIII — Flang *(~80 pp, 4 ch)*
- [x] Ch125 — Flang Architecture and Driver
- [x] Ch126 — The HLFIR and FIR Dialects
- [x] Ch127 — Flang OpenMP and OpenACC
- [x] Ch128 — Flang Codegen and Runtime

## Part XIX — MLIR Foundations *(~150 pp, 8 ch)*
- [x] Ch129 — MLIR Philosophy
- [x] Ch130 — MLIR IR Structure
- [x] Ch131 — The Type and Attribute Systems
- [x] Ch132 — Defining Dialects with ODS
- [x] Ch133 — Op Interfaces and Traits
- [x] Ch134 — The MLIR C++ API
- [x] Ch135 — PDL and PDLL
- [x] Ch136 — MLIR Bytecode and Serialization

## Part XX — In-Tree Dialects *(~190 pp, 10 ch)* ✓ COMPLETE
- [x] Ch137 — Core Dialects
- [x] Ch138 — Memory Dialects
- [x] Ch139 — Tensor and Linalg
- [x] Ch140 — Affine and SCF
- [x] Ch141 — Vector and Sparse
- [x] Ch142 — GPU Dialect Family
- [x] Ch143 — SPIR-V Dialect
- [x] Ch144 — Hardware Vector Dialects
- [x] Ch145 — LLVM Dialect
- [x] Ch146 — Async, OpenMP, OpenACC, DLTI, EmitC

## Part XXI — MLIR Transformations *(~110 pp, 6 ch)*
- [x] Ch147 — Pattern Rewriting
- [x] Ch148 — Dialect Conversion
- [x] Ch149 — The Pass Infrastructure
- [x] Ch150 — The Transform Dialect
- [x] Ch151 — Bufferization Deep Dive
- [x] Ch152 — Lowering Pipelines

## Part XXII — XLA and the OpenXLA Stack *(~120 pp, 6 ch)*
- [x] Ch153 — XLA Architecture
- [x] Ch154 — HLO and StableHLO
- [x] Ch155 — XLA:CPU
- [x] Ch156 — XLA:GPU
- [x] Ch157 — PJRT — The Plugin Runtime Interface
- [x] Ch158 — SPMD, GSPMD, and Auto-Sharding

## Part XXIII — MLIR in Production *(~160 pp, 8 ch)*
- [x] Ch159 — Building a Domain-Specific Compiler
- [x] Ch160 — MLIR Python Bindings
- [x] Ch161 — torch-mlir, ONNX-MLIR, and JAX/TF Bridges
- [x] Ch162 — IREE — A Deployment Compiler
- [x] Ch163 — Triton — A Compiler for GPU Kernels
- [x] Ch164 — CUDA Tile IR
- [x] Ch165 — GPU Compilation Through MLIR
- [x] Ch166 — Mojo, Polygeist, Enzyme-MLIR, and Beyond

## Part XXIV — Verified Compilation *(~110 pp, 5 ch)* [THEORETICAL] ✓ COMPLETE
- [x] Ch167 — Operational Semantics and Program Logics
- [x] Ch168 — CompCert
- [x] Ch169 — Vellvm and Formalizing LLVM IR
- [x] Ch170 — Alive2 and Translation Validation
- [x] Ch171 — The Undef/Poison Story Formally

## Part XXV — Operations, Bindings, and Contribution *(~100 pp, 5 ch)* ✓ COMPLETE
- [x] Ch172 — Testing in LLVM and MLIR
- [x] Ch173 — Debugging the Compiler
- [x] Ch174 — Performance Engineering
- [x] Ch175 — Language Bindings
- [x] Ch176 — Contributing to LLVM

## Part XXVI — Ecosystem and Frontiers *(~54 pp, 4 ch)*
- [ ] Ch177 — rustc and the Rust Codegen Backend
- [ ] Ch178 — LLVM/MLIR for AI: The Full Stack
- [ ] Ch179 — AI-Guided Compilation
- [ ] Ch180 — ANTLR4, TreeSitter, and Language Tooling with LLVM

### Chapter Plans

**Ch177 — rustc and the Rust Codegen Backend** (~15 pp)
1. rustc architecture: driver → HIR → THIR → MIR → codegen
2. MIR: Rust's mid-level SSA-like IR; `Place`/`Rvalue`/`Terminator`; borrow-check output
3. `rustc_codegen_llvm`: `FunctionCx`, MIR→LLVM IR lowering; ADT/enum layout; trait objects (`dyn Trait` vtable emission)
4. Rust-specific LLVM attributes: `nounwind`, `noalias`, `dereferenceable`, `noundef`, `align`
5. Panics and unwinding: `begin_unwind` → `llvm.eh.sjlj`/Itanium EH; `-C panic=abort`
6. The Cranelift alternative backend (`rustc_codegen_cranelift`): motivation, IR model, maturity
7. The GCC backend (`rustc_codegen_gcc`): status and rationale
8. LTO: `-C lto=thin`, `-C lto=fat`; ThinLTO with rustc's ThinLTO cache
9. PGO in Rust: `-C profile-generate`/`-C profile-use`
10. Cross-compilation: target triples, `--target`, sysroot, `build.rs` implications
11. rustc's vendored LLVM: why, version delta, custom patches

**Ch178 — LLVM/MLIR for AI: The Full Stack** (~15 pp)
1. The AI compiler hierarchy: PyTorch/JAX eager → `torch.export`/`jax.jit` capture → StableHLO → MLIR dialects → target
2. Key dialects in the AI stack: `tosa`, `stablehlo`, `tensor`, `linalg.generic`, `vector`, `memref`, `gpu`
3. Quantization: `quant` dialect encoding; INT8/INT4/FP8 lowering; GPTQ and GGUF pipelines
4. Dynamic shapes: StableHLO dynamism extensions; `tensor.dim` / `shape` dialect; IREE's dynamic shape model
5. Target landscape: CUDA (PTX via NVPTX), ROCm (HSACO via AMDGPU), NPUs (Apple ANE via CoreML, Hexagon DSP via Qualcomm toolchain, Edge TPU)
6. Inference deployment: IREE (multi-target, VM-based), TFLite (flatbuffer format), ONNX Runtime (EP model), TensorRT (engine caching)
7. Training vs inference: different optimization priorities (throughput vs latency), different precision requirements
8. Kernel fusion: FlashAttention 2/3 as a case study in tiling and softmax fusion through MLIR
9. The full end-to-end path: PyTorch model → `torch.export` → torch-mlir → StableHLO → IREE dispatch → PTX/HSACO

**Ch179 — AI-Guided Compilation** (~12 pp)
1. MLGO: the ML-guided optimization framework in LLVM; the `MLModelRunner` abstraction
2. The RL-trained inliner: feature vector, reward function (binary size + perf), training loop (cross-ref Ch66)
3. ML-based register allocator eviction: the `RegAllocEvictionAdvisor`; priority model; deployment
4. The TFLite runtime embedded in LLVM: AOT-compiled models, the LLVM build flag, offline vs development mode
5. ML-based cost models: instruction scheduling cost functions; IREE's ML-based tile size selection
6. Ansor/AutoTVM: search space definition, sketch-based program sampler, cost model learning
7. Triton's autotuner: `@triton.autotune` mechanism; exhaustive vs evolutionary search
8. Profile inference with ML: synthesizing PGO profiles from code structure without instrumented runs
9. LLM-assisted compiler development: TableGen authoring with LLMs; pass synthesis; LLM-guided fuzzing (EvoFuzz, CovRL-Fuzz)
10. LLVM IR as training corpus: the `llvm-ir` dataset; training code LLMs on compiler IRs
11. Neural superoptimization: Bansal-Aiken PLDI approach; learned peephole rules

**Ch180 — ANTLR4, TreeSitter, and Language Tooling with LLVM** (~12 pp)
1. Tooling vs compilation: why editor tooling needs different properties (error tolerance, incrementality, speed) than a compiler
2. ANTLR4 fundamentals: LL(*) parsing; `.g4` grammar syntax; lexer rules vs parser rules; semantic predicates
3. The ANTLR4 C++ runtime: `ANTLRInputStream`, `CommonTokenStream`, visitor vs listener patterns
4. ANTLR4 → LLVM pipeline: walking the parse tree to build an AST; emitting LLVM IR from the AST; a worked example (a simple expression language)
5. Error recovery in ANTLR4: sync-and-return, single-token insertion/deletion, default error strategy
6. TreeSitter fundamentals: incremental GLR parsing; `.js` grammar authoring; the C API (`ts_parser_parse`, `ts_node_*`)
7. TreeSitter language bindings: Rust (`tree-sitter` crate), Python (`tree_sitter` package), Node.js
8. What TreeSitter is for: syntax highlighting, structural navigation, lightweight pattern matching (`ts_query`); what it is NOT for (ambiguous grammars, full semantic analysis)
9. The nvim-treesitter ecosystem; integrating with clangd over LSP for a two-tier toolchain
10. Comparison matrix: hand-written (Clang/GCC), ANTLR4, TreeSitter, PEG (pest, nom, peg.js), parser combinators (`combine`, `winnow`)
11. The two-tier frontend pattern: TreeSitter for IDE tooling + ANTLR4/hand-written recursive descent for compilation

## Appendices
- [x] Appendix A — LLVM IR Quick Reference
- [x] Appendix B — MLIR Dialect Quick Reference
- [x] Appendix C — Command-Line Tools Cheat Sheet
- [x] Appendix D — Migration Notes: LLVM 18 → 22
- [x] Appendix E — Glossary
- [x] Appendix F — Object File Format Reference
- [x] Appendix G — DWARF Debug Info Reference
- [x] Appendix H — The C++ ABI: Itanium and Microsoft

---

## Suggested Improvements (running list)
*Add items here before compaction; review after resuming*

---

## Stats
- Total chapters: 180 + 8 appendices = 188 items
- Completed: 184 / 188 (original complete; 4 new Part XXVI chapters pending)
- Estimated pages written: ~2,900 (original) + ~54 (Part XXVI target) = ~2,954 total target
