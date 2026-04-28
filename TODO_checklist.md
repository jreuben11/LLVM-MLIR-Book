# Book Progress Checklist
*Last updated: 2026-04-28 (Part VI complete — Ch39–Ch44 committed). Read this first after any context compaction.*

## Status Legend
- `[ ]` Not started
- `[~]` In progress
- `[x]` Written and committed

---

## Current Focus
**Parts I–VI COMPLETE** ✓ (49/184 chapters done)
**Next up:** Part VII — Clang as a Multi-Language Compiler (Ch45–Ch51, 7 chapters)
**Batch:** Ch45–Ch51 in parallel

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
- [ ] Ch45 — The Static Analyzer
- [ ] Ch46 — libtooling and AST Matchers
- [ ] Ch47 — clangd, clang-tidy, clang-format, clang-refactor
- [ ] Ch48 — Clang as a CUDA Compiler
- [ ] Ch49 — Clang as a HIP Compiler
- [ ] Ch50 — Clang as SYCL, OpenCL, and OpenMP-Offload
- [ ] Ch51 — Clang as an HLSL Compiler

## Part VIII — ClangIR (CIR) *(~50 pp, 3 ch)*
- [ ] Ch52 — ClangIR Architecture
- [ ] Ch53 — CIR Generation from AST
- [ ] Ch54 — CIR Lowering and Analysis

## Part IX — Frontend Authoring (Building Your Own) *(~80 pp, 4 ch)*
- [ ] Ch55 — Building a Frontend
- [ ] Ch56 — Lowering AST to IR
- [ ] Ch57 — Lowering High-Level Constructs
- [ ] Ch58 — Language Runtime Concerns

## Part X — Analysis and the Middle-End *(~210 pp, 11 ch)*
- [ ] Ch59 — The New Pass Manager
- [ ] Ch60 — Writing a Pass
- [ ] Ch61 — Foundational Analyses
- [ ] Ch62 — Scalar Optimizations
- [ ] Ch63 — Loop Optimizations
- [ ] Ch64 — Vectorization Deep Dive
- [ ] Ch65 — Inter-Procedural Optimizations
- [ ] Ch66 — The ML Inliner and ML Regalloc
- [ ] Ch67 — Profile-Guided Optimization
- [ ] Ch68 — Hardening and Mitigations
- [ ] Ch69 — Whole-Program Devirtualization

## Part XI — Polyhedral Theory *(~80 pp, 4 ch)* [THEORETICAL]
- [ ] Ch70 — Foundations: Polyhedra and Integer Programming
- [ ] Ch71 — The Polyhedral Model
- [ ] Ch72 — Scheduling Algorithms
- [ ] Ch73 — Code Generation from Polyhedral Schedules

## Part XII — Polly *(~50 pp, 3 ch)*
- [ ] Ch74 — Polly Architecture
- [ ] Ch75 — Polly Transformations
- [ ] Ch76 — Polly in Practice

## Part XIII — Link-Time and Whole-Program *(~80 pp, 4 ch)*
- [ ] Ch77 — LTO and ThinLTO
- [ ] Ch78 — The LLVM Linker (LLD)
- [ ] Ch79 — Linker Internals: GOT, PLT, TLS
- [ ] Ch80 — llvm-cas and Content-Addressable Builds

## Part XIV — The Backend *(~280 pp, 14 ch)*
- [ ] Ch81 — Backend Architecture
- [ ] Ch82 — TableGen Deep Dive
- [ ] Ch83 — The Target Description
- [ ] Ch84 — SelectionDAG: Building and Legalizing
- [ ] Ch85 — SelectionDAG: Combining and Selecting
- [ ] Ch86 — GlobalISel
- [ ] Ch87 — Inline Assembly Lowering
- [ ] Ch88 — The Machine IR
- [ ] Ch89 — Pre-RegAlloc Passes
- [ ] Ch90 — Register Allocation
- [ ] Ch91 — The Machine Pipeliner
- [ ] Ch92 — The Machine Outliner
- [ ] Ch93 — Post-RegAlloc and Pre-Emit
- [ ] Ch94 — The MC Layer and MIR Test Infrastructure

## Part XV — Targets *(~250 pp, 13 ch)*
- [ ] Ch95 — The X86 Backend
- [ ] Ch96 — The AArch64 Backend
- [ ] Ch97 — The 32-bit ARM Backend
- [ ] Ch98 — The RISC-V Backend Architecture
- [ ] Ch99 — The RISC-V Vector Extension (RVV)
- [ ] Ch100 — RISC-V Bit-Manip, Crypto, and Custom Extensions
- [ ] Ch101 — PowerPC, SystemZ, MIPS, SPARC, LoongArch
- [ ] Ch102 — NVPTX and the CUDA Path
- [ ] Ch103 — AMDGPU and the ROCm Path
- [ ] Ch104 — The SPIR-V Backend
- [ ] Ch105 — DXIL and DirectX Shader Compilation
- [ ] Ch106 — WebAssembly and BPF
- [ ] Ch107 — Embedded Targets

## Part XVI — JIT, Sanitizers, and Diagnostic Tools *(~210 pp, 11 ch)*
- [ ] Ch108 — The ORC JIT
- [ ] Ch109 — JITLink
- [ ] Ch110 — User-Space Sanitizers
- [ ] Ch111 — HWASan and MTE
- [ ] Ch112 — Production Allocators: Scudo and GWP-ASan
- [ ] Ch113 — Kernel Sanitizers
- [ ] Ch114 — LibFuzzer and Coverage-Guided Fuzzing
- [ ] Ch115 — Source-Based Code Coverage
- [ ] Ch116 — LLDB Architecture
- [ ] Ch117 — DWARF and Debug Info
- [ ] Ch118 — BOLT and Post-Link Optimization

## Part XVII — Runtime Libraries *(~110 pp, 6 ch)*
- [ ] Ch119 — compiler-rt Builtins
- [ ] Ch120 — libunwind
- [ ] Ch121 — libc++
- [ ] Ch122 — libc++abi
- [ ] Ch123 — LLVM-libc
- [ ] Ch124 — OpenMP and Offload Runtimes

## Part XVIII — Flang *(~80 pp, 4 ch)*
- [ ] Ch125 — Flang Architecture and Driver
- [ ] Ch126 — The HLFIR and FIR Dialects
- [ ] Ch127 — Flang OpenMP and OpenACC
- [ ] Ch128 — Flang Codegen and Runtime

## Part XIX — MLIR Foundations *(~150 pp, 8 ch)*
- [ ] Ch129 — MLIR Philosophy
- [ ] Ch130 — MLIR IR Structure
- [ ] Ch131 — The Type and Attribute Systems
- [ ] Ch132 — Defining Dialects with ODS
- [ ] Ch133 — Op Interfaces and Traits
- [ ] Ch134 — The MLIR C++ API
- [ ] Ch135 — PDL and PDLL
- [ ] Ch136 — MLIR Bytecode and Serialization

## Part XX — In-Tree Dialects *(~190 pp, 10 ch)*
- [ ] Ch137 — Core Dialects
- [ ] Ch138 — Memory Dialects
- [ ] Ch139 — Tensor and Linalg
- [ ] Ch140 — Affine and SCF
- [ ] Ch141 — Vector and Sparse
- [ ] Ch142 — GPU Dialect Family
- [ ] Ch143 — SPIR-V Dialect
- [ ] Ch144 — Hardware Vector Dialects
- [ ] Ch145 — LLVM Dialect
- [ ] Ch146 — Async, OpenMP, OpenACC, DLTI, EmitC

## Part XXI — MLIR Transformations *(~110 pp, 6 ch)*
- [ ] Ch147 — Pattern Rewriting
- [ ] Ch148 — Dialect Conversion
- [ ] Ch149 — The Pass Infrastructure
- [ ] Ch150 — The Transform Dialect
- [ ] Ch151 — Bufferization Deep Dive
- [ ] Ch152 — Lowering Pipelines

## Part XXII — XLA and the OpenXLA Stack *(~120 pp, 6 ch)*
- [ ] Ch153 — XLA Architecture
- [ ] Ch154 — HLO and StableHLO
- [ ] Ch155 — XLA:CPU
- [ ] Ch156 — XLA:GPU
- [ ] Ch157 — PJRT — The Plugin Runtime Interface
- [ ] Ch158 — SPMD, GSPMD, and Auto-Sharding

## Part XXIII — MLIR in Production *(~160 pp, 8 ch)*
- [ ] Ch159 — Building a Domain-Specific Compiler
- [ ] Ch160 — MLIR Python Bindings
- [ ] Ch161 — torch-mlir, ONNX-MLIR, and JAX/TF Bridges
- [ ] Ch162 — IREE — A Deployment Compiler
- [ ] Ch163 — Triton — A Compiler for GPU Kernels
- [ ] Ch164 — CUDA Tile IR
- [ ] Ch165 — GPU Compilation Through MLIR
- [ ] Ch166 — Mojo, Polygeist, Enzyme-MLIR, and Beyond

## Part XXIV — Verified Compilation *(~110 pp, 5 ch)* [THEORETICAL]
- [ ] Ch167 — Operational Semantics and Program Logics
- [ ] Ch168 — CompCert
- [ ] Ch169 — Vellvm and Formalizing LLVM IR
- [ ] Ch170 — Alive2 and Translation Validation
- [ ] Ch171 — The Undef/Poison Story Formally

## Part XXV — Operations, Bindings, and Contribution *(~100 pp, 5 ch)*
- [ ] Ch172 — Testing in LLVM and MLIR
- [ ] Ch173 — Debugging the Compiler
- [ ] Ch174 — Performance Engineering
- [ ] Ch175 — Language Bindings
- [ ] Ch176 — Contributing to LLVM

## Appendices
- [ ] Appendix A — LLVM IR Quick Reference
- [ ] Appendix B — MLIR Dialect Quick Reference
- [ ] Appendix C — Command-Line Tools Cheat Sheet
- [ ] Appendix D — Migration Notes: LLVM 18 → 22
- [ ] Appendix E — Glossary
- [ ] Appendix F — Object File Format Reference
- [ ] Appendix G — DWARF Debug Info Reference
- [ ] Appendix H — The C++ ABI: Itanium and Microsoft

---

## Suggested Improvements (running list)
*Add items here before compaction; review after resuming*

---

## Stats
- Total chapters: 176 + 8 appendices = 184 items
- Completed: 49 / 184
- Estimated pages written: ~801 / ~2195
