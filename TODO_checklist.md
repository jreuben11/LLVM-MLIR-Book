# Book Progress Checklist
*Last updated: 2026-04-30 (Parts XXVII–XXVIII complete; Part XXIX in progress — Ch197–Ch199 written; Ch200–Ch202 + 7 existing chapter expansions remaining). 207 items complete; 3 new chapters planned, not yet written.*

## Status Legend
- `[ ]` Not started
- `[~]` In progress
- `[x]` Written and committed

---

## Current Focus
**Part XXIX — Compiler Tooling, Kernel Integration, and Binary Analysis** (6 new chapters, Ch197–Ch202)
- Parts XXVII–XXVIII complete (Ch184–Ch196 all written and committed)
- Part XXIX: 6 practical chapters (~12 pp each), all independent — can fully parallelize
- Recommended order: Ch197–Ch202 can be written in parallel (no inter-chapter cross-references)
- Also: 7 existing chapters have planned expansions (Ch79, Ch80, Ch106, Ch110, Ch133, Ch140, Ch173)

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

## Part XXVI — Ecosystem and Frontiers *(~108 pp, 7 ch)*
- [x] Ch177 — rustc: Architecture, MIR, and Codegen Backends
- [x] Ch178 — The Rust Compiler Ecosystem
- [x] Ch179 — LLVM/MLIR for AI: The Full Stack
- [x] Ch180 — AI-Guided Compilation
- [x] Ch181 — Formal Verification in Practice
- [x] Ch182 — Language Tooling: Parsers, Lexers, and Syntax Trees
- [x] Ch183 — Modern C++ for Compiler Development: C++23, Contracts, and Reflection

## Part XXVII — Mathematical Foundations and Verified Systems *(~120 pp, 6 ch)* [THEORETICAL]
- [x] Ch184 — Proof Assistant Internals: Lean 4, Coq/Rocq, and Isabelle/HOL
- [x] Ch185 — Mathematical Logic and Model Theory for Compiler Engineers
- [x] Ch186 — Verified Hardware: CHERI Capabilities and the seL4 Microkernel
- [x] Ch187 — Commutative Algebra and Its Applications in Compilation
- [x] Ch188 — Category Theory for Compiler Engineers
- [x] Ch189 — Denotational Semantics and Domain Theory

## Part XXVIII — Language Ecosystems, Emerging Targets, and Engineering Practice *(~86 pp, 7 ch)*
- [x] Ch190 — CIRCT: Circuit IR Compilers and Tools
- [x] Ch191 — Quantum Compilation: QIR, QUIR, and MLIR Quantum Dialects
- [x] Ch192 — Swift SIL: Ownership, Optimization, and Influence on MLIR
- [x] Ch193 — Julia: Type-Inference-Driven LLVM Specialization
- [x] Ch194 — Zig: Comptime Metaprogramming and LLVM IR Generation
- [x] Ch195 — Safety-Critical Toolchain Qualification: DO-178C, ISO 26262, and Ferrocene
- [x] Ch196 — Cross-Language ABI Interoperability: Binding Generators and UniFFI

## Part XXIX — Compiler Tooling, Kernel Integration, and Binary Analysis *(~72 pp, 6 ch)*
- [x] Ch197 — Clang Plugin System
- [x] Ch198 — Value Tracking Infrastructure in LLVM
- [x] Ch199 — llvm-mca: Static Performance Analysis
- [ ] Ch200 — Linux Kernel Compilation with LLVM/Clang
- [ ] Ch201 — Binary Lifting to LLVM IR
- [ ] Ch202 — Apache TVM: An ML Operator Compiler

## Existing Chapter Expansions (planned, not yet written)
- [ ] Ch79 expansion — linker relaxation in depth (RISC-V, AArch64, x86 GOTPCRELX, TLS chains)
- [ ] Ch80 expansion — reproducible builds and toolchain supply-chain security
- [ ] Ch106 expansion — WASI/WasmGC (WASI preview 2, wasm32-wasip1/p2, WasmGC typed references)
- [ ] Ch110 expansion — DataFlowSanitizer (DFSan): taint tracking, shadow memory, dfsan_label API
- [ ] Ch133 expansion — MLIR external models (registerExternalModels, retroactive interface attachment)
- [ ] Ch140 expansion — MLIR Presburger arithmetic library (IntegerPolyhedron, Omega test)
- [ ] Ch173 expansion — mlir-lsp-server (dialect registration for LSP, mlir-vscode, mlir-query)

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
- Total chapters: 202 + 8 appendices = 210 items (+ 7 existing chapter expansions)
- Completed: 207 / 210 (Ch200–Ch202 not yet written; 7 expansions not yet written)
- Estimated pages written: ~2,900 (original) + ~108 (Part XXVI) + ~120 (Part XXVII) + ~86 (Part XXVIII) = ~3,214 total
- Planned: +~72 (Part XXIX, 6 ch) + ~35 (7 chapter expansions, ~5 pp each) = ~3,321 total when complete
