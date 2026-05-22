# Book Progress Checklist
*Last updated: 2026-05-22 (Ch234–Ch238 planned, not yet written). 259 / 264 items complete.*

## Status Legend
- `[ ]` Not started
- `[~]` In progress
- `[x]` Written and committed

---

## Current Focus
**Gap-fill chapters: Ch234–Ch238** (5 new chapters across 3 parts)
- Ch234 (Part XXIX) — KLEE Symbolic Execution; Ch235, Ch237, Ch238 (Part XXVIII) — GHC LLVM backend, TinyGo, Kotlin/Native; Ch236 (Part XV) — Android NDK
- All independent, can be written in parallel
- Recommended order: Ch234+Ch235+Ch237+Ch238+Ch236 all in one pass (no cross-references between them)

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

## Part XV — Targets *(~274 pp, 15 ch)*
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
- [ ] Ch236 — Android NDK: Cross-Compiling C/C++ for Android with LLVM (~12 pages; §236.1 NDK architecture/toolchain layout/LLD default, §236.2 target triples and API levels, §236.3 sysroot structure/stub libraries, §236.4 CMake integration/ANDROID_ABI/ANDROID_PLATFORM, §236.5 Bionic libc differences/x18 register/missing APIs, §236.6 multi-ABI APK layout, §236.7 sanitizers/HWASan/wrap.sh, §236.8 Android LLVM fork/GKI requirement)
- [x] Ch233 — Emscripten: C/C++ to WebAssembly via LLVM (~12 pages; §233.1 architecture/target triple/system libs, §233.2 memory model/ALLOW_MEMORY_GROWTH, §233.3 JS interop EM_JS/EM_ASM/ccall, §233.4 Embind C++↔JS bindings, §233.5 Asyncify stack unwind/rewind, §233.6 ports/SDL2/WebGL/event loop, §233.7 Wasm threads/SharedArrayBuffer/pthreads, §233.8 output pipeline/wasm-opt/DWARF)

## Part XVI — JIT, Sanitizers, and Diagnostic Tools *(~294 pp, 18 ch)*
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
- [x] Ch219 — ORC JIT in Production: clang-repl, LLDB, PostgreSQL, Numba, and Halide (~12 pages; §219.1 ORC recipe, §219.2 clang-repl/Cling, §219.3 LLDB expression JIT, §219.4 PostgreSQL query JIT, §219.5 Numba, §219.6 Halide, §219.7 WAVM, §219.8 taxonomy table)
- [x] Ch220 — Runtime Self-Modification: Source Introspection, Incremental Recompilation, and ORC Hot-Loading (~14 pages; §220.1–§220.9 self-modification loop through reference impls, §220.10 embedded bitcode self-analysis, §220.11 self-propagation to peer instances)
- [x] Ch221 — Speculative Optimization, Inline Caches, and Deoptimization (~12 pages; §221.1 speculative model, §221.2 inline caches, §221.3 guard IR, §221.4 @llvm.deoptimize/deopt bundles, §221.5 stackmap frame reconstruction, §221.6 OSR deep-dive, §221.7 V8 tiering, §221.8 HotSpot tiering, §221.9 ORC ReOptimizeLayer connection)
- [x] Ch222 — Plugin Architecture, Dynamic Loading, and ABI Stability (~12 pages; §222.1 POSIX dlopen, §222.2 ABI stability contract, §222.3 type-safe C++ interfaces, §222.4 plugin discovery, §222.5 Windows/macOS, §222.6 llvm::sys::DynamicLibrary, §222.7 real-world architectures, §222.8 security model, §222.9 ORC interaction)
- [x] Ch228 — Virtual Machine Design: Bytecode Interpreters, GC Strategies, and Object Models (~14 pages; §228.1 stack vs register VM, §228.2 bytecode instruction set design, §228.3 dispatch strategies incl. copy-and-patch, §228.4 object representation/NaN-boxing/hidden classes, §228.5 GC strategies mark-sweep/generational/concurrent, §228.6 GC roots and precise vs conservative scanning, §228.7 method dispatch and PICs, §228.8 frame layout and calling conventions, §228.9 ORC JIT tiered compilation integration, §228.10 reference VM architectures CPython/Lua/YARV/HotSpot)
- [x] Ch232 — LLVM XRay: Low-Overhead Function Tracing (~12 pages; §232.1 design goals vs perf/gprof/valgrind, §232.2 NOP sled instrumentation/xray_patch/xray_instr_map, §232.3 basic/flight-recorder/custom modes, §232.4 custom handlers/flame-graph collector, §232.5 llvm-xray tool suite account/convert/stack/graph, §232.6 LTO integration/instruction-threshold, §232.7 selective tracing attributes/per-function patch, §232.8 production patterns/eBPF comparison)

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

## Part XXVI — Ecosystem and Frontiers *(~122 pp, 8 ch)*
- [x] Ch177 — rustc: Architecture, MIR, and Codegen Backends
- [x] Ch178 — The Rust Compiler Ecosystem
- [x] Ch179 — LLVM/MLIR for AI: The Full Stack
- [x] Ch180 — AI-Guided Compilation
- [x] Ch181 — Formal Verification in Practice
- [x] Ch182 — Language Tooling: Parsers, Lexers, and Syntax Trees
- [x] Ch183 — Modern C++ for Compiler Development: C++23, Contracts, and Reflection
- [x] Ch229 — torch.compile: TorchDynamo, AOTAutograd, and TorchInductor (~14 pages; §229.1 PyTorch 2.0 compilation model eager vs compiled, §229.2 TorchDynamo PEP-523 frame hook/proxy tensors/guards, §229.3 FX graph IR node types/passes, §229.4 AOTAutograd joint fwd-bwd tracing/decompositions, §229.5 TorchInductor GPU→Triton/CPU→C++, §229.6 torch.export/ExportedProgram/ExecuTorch, §229.7 guard system recompilation/dynamic shapes, §229.8 backends/stability PyTorch 2.4-2.6)

## Part XXVII — Mathematical Foundations and Verified Systems *(~120 pp, 6 ch)* [THEORETICAL]
- [x] Ch184 — Proof Assistant Internals: Lean 4, Coq/Rocq, and Isabelle/HOL
- [x] Ch185 — Mathematical Logic and Model Theory for Compiler Engineers
- [x] Ch186 — Verified Hardware: CHERI Capabilities and the seL4 Microkernel
- [x] Ch187 — Commutative Algebra and Its Applications in Compilation
- [x] Ch188 — Category Theory for Compiler Engineers
- [x] Ch189 — Denotational Semantics and Domain Theory

## Part XXVIII — Language Ecosystems, Emerging Targets, and Engineering Practice *(~148 pp, 12 ch)*
- [x] Ch190 — CIRCT: Circuit IR Compilers and Tools
- [x] Ch191 — Quantum Compilation: QIR, QUIR, and MLIR Quantum Dialects
- [x] Ch192 — Swift SIL: Ownership, Optimization, and Influence on MLIR
- [x] Ch193 — Julia: Type-Inference-Driven LLVM Specialization
- [x] Ch194 — Zig: Comptime Metaprogramming and LLVM IR Generation
- [x] Ch195 — Safety-Critical Toolchain Qualification: DO-178C, ISO 26262, and Ferrocene
- [x] Ch196 — Cross-Language ABI Interoperability: Binding Generators and UniFFI
- [x] Ch230 — Cranelift: A Lightweight JIT for WebAssembly and Rust (~12 pages; §230.1 design philosophy/speed vs quality tradeoffs, §230.2 CLIF IR SSA/block-parameters/types, §230.3 FunctionBuilder frontend, §230.4 ISLE instruction selection DSL, §230.5 regalloc2 live-range splitting, §230.6 Wasmtime/Winch integration, §230.7 rustc_codegen_cranelift, §230.8 comparison table vs LLVM/QBE/Baseline)
- [x] Ch231 — GraalVM: Native Image, Truffle Interpreters, and Polyglot Runtimes (~14 pages; §231.1 architecture JVM substrate/Graal JIT/Truffle/SubstrateVM, §231.2 Truffle @Specialization state machine, §231.3 partial evaluation/sea-of-nodes/JVMCI, §231.4 Native Image points-to analysis/closed-world/heap snapshot, §231.5 building a Truffle language, §231.6 GraalPy/TruffleRuby/GraalJS production languages, §231.7 polyglot API, §231.8 Sulong LLVM bitcode on Truffle)
- [ ] Ch235 — GHC's LLVM Backend: Haskell to Native via LLVM (~14 pages; §235.1 GHC pipeline Haskell→Core→STG→Cmm→LLVM IR, §235.2 GHC Core/System FC/coercion proofs/Core Lint, §235.3 STG Machine/thunks/closures/info tables/heap allocation, §235.4 Cmm GenCmmDecl/CmmProc/SRT/stack layout, §235.5 LlvmCodeGen/LlvmM monad/LlvmType, §235.6 ghccc CC10/R1-R10/Sp/Hp/alwaysLive registers, §235.7 GC integration/stgTBAA/info-table root discovery, §235.8 -fllvm vs NCG/supported LLVM 13-22/when LLVM wins)
- [ ] Ch237 — TinyGo: Compiling Go for Embedded Systems and WebAssembly via LLVM (~12 pages; §237.1 design/go/ssa frontend/vs gc toolchain, §237.2 pipeline compiler.go→LLVM IR, §237.3 go-llvm wrapper/custom datalayout, §237.4 GC options -gc=none/leaking/conservative/precise, §237.5 goroutines as LLVM coroutines/llvm.coro.*/cooperative scheduling, §237.6 interface lowering typeID+data pairs, §237.7 target JSON files/Cortex-M/RV32/AVR/wasm, §237.8 stdlib subset/go:linkname/export FFI/wasm_exec.js)
- [ ] Ch238 — Kotlin/Native: Compiling Kotlin to Native via LLVM (~12 pages; §238.1 KonanBackend/K2 FIR→KIR→LLVM/KonanTarget enum, §238.2 InteropLowering/ModuleBitcodeOptimization/LTOBitcodeOptimization, §238.3 CodeGenerator.kt/custom LLVM fork/-Xsave-llvm-ir-after, §238.4 ARC/freeze→concurrent GC 1.7.20+, §238.5 safepoints/stack maps/Instruments integration, §238.6 cinterop tool/.def files/kotlinx.cinterop/CPointer, §238.7 ObjC/Swift interop/@ObjCName/@ExportObjCClass, §238.8 KonanTarget enum/framework/static/XCFramework/@CName)

## Part XXIX — Compiler Tooling, Kernel Integration, and Binary Analysis *(~84 pp, 7 ch)*
- [x] Ch197 — Clang Plugin System
- [x] Ch198 — Value Tracking Infrastructure in LLVM
- [x] Ch199 — llvm-mca: Static Performance Analysis
- [x] Ch200 — Linux Kernel Compilation with LLVM/Clang
- [x] Ch201 — Binary Lifting to LLVM IR
- [x] Ch202 — Apache TVM: An ML Operator Compiler
- [ ] Ch234 — KLEE: Symbolic Execution of LLVM IR (~12 pages; §234.1 design goals/scope vs fuzzing/concolic, §234.2 KModule/KFunction/KInstruction shadow layer, §234.3 ExecutionState/pc/stack/addressSpace/ConstraintSet, §234.4 MemoryObject/ObjectState/concreteStore/knownSymbolics, §234.5 Searcher classes DFS/BFS/WeightedRandom/CoveringNew, §234.6 SolverChain/Query/Z3+STP+CVC5 backends/TimingSolver, §234.7 klee_make_symbolic/uclibc/POSIX model/KTest format, §234.8 clang -emit-llvm -g; klee --libc=uclibc --posix-runtime; extensions/seeding/SCKLEE)

## Existing Chapter Expansions (planned, not yet written)
- [x] Ch79 expansion — linker relaxation in depth (RISC-V, AArch64, x86 GOTPCRELX, TLS chains)
- [x] Ch80 expansion — reproducible builds and toolchain supply-chain security
- [x] Ch106 expansion — WASI/WasmGC (WASI preview 2, wasm32-wasip1/p2, WasmGC typed references)
- [x] Ch110 expansion — DataFlowSanitizer (DFSan): taint tracking, shadow memory, dfsan_label API
- [x] Ch133 expansion — MLIR external models (registerExternalModels, retroactive interface attachment)
- [x] Ch140 expansion — MLIR Presburger arithmetic library (IntegerPolyhedron, Omega test)
- [x] Ch173 expansion — mlir-lsp-server (dialect registration for LSP, mlir-vscode, mlir-query)
- [x] Ch58 expansion — patchpoint→code-mutation full cycle (stackmap v3, dual-map W^X pattern, guard IR emission for deopt)
- [x] Ch108 expansion — self-modifying code: ReOptimizeLayer/RedirectableSymbolManager tiered JIT (LLVM 22); OSR/guard/deopt cycle; ELF W^X + JITLink dual-mapping; homoiconic language examples (Forth, SBCL, Julia @generated); WASM limitations; cross-ref Ch207
- [x] Ch200 expansion — Linux kernel live patching (klp_func/klp_object/klp_patch/klp_enable_patch); text_poke_bp() int3 protocol; poking_mm dual-mapping; ftrace/-mfentry NOP-sled redirection; userspace uprobes

## Part XXX — AI-First Programming Language Design *(~60 pp, 5 ch)*
- [x] Ch203 — AI-First PL Principles and Landscape
- [x] Ch204 — Formal Language Specification for AI-First PLs
- [x] Ch205 — Transformer Model Development PLs
- [x] Ch206 — Multi-Agent PLs, AI-First SDLC, Security, and Paradigm Failures
- [x] Ch207 — Reflective Code, Open Problems, and Build Roadmap

## Part XXXI — Frontier AI Evolution *(~192 pp, 16 ch)*
- [x] Ch208 — GPU Kernel DSLs: Triton, Helion, and Gluon
- [x] Ch209 — CUTLASS, Thrust, CuTe, and TileIR: GPU Parallel Primitives and Layout Algebra
- [x] Ch210 — The JAX Ecosystem: A Functional Neural Compilation Stack
- [x] Ch211 — Neural Programs as Compiled Artifacts: The Self-Aware Execution Stack (bridging)
- [x] Ch212 — Weights as a Programming Substrate
- [x] Ch213 — Mechanistic Interpretability Infrastructure
- [x] Ch214 — Gradient-Based Self-Modification: Model Editing, Meta-Learning, and Test-Time Adaptation
- [x] Ch215 — Evolutionary Architecture Search
- [x] Ch216 — Formal Self-Improvement Theory
- [x] Ch217 — Self-Reflective Inference and Architecture Introspection
- [x] Ch218 — Self-Improvement Fitness Functions and Capability Assessment
- [x] Ch223 — Verification-Guided Pass Selection: LLM-VeriOpt and the Alive2 Reward Loop (~12 pp; §223.1 verification-in-loop architecture, §223.2 Alive2 as reward signal, §223.3 LLM-VeriOpt system, §223.4 pass ordering MDP, §223.5 AlphaVerus tree search, §223.6 LLVM CI integration, §223.7 verified self-modification, §223.8 constraints/open problems)
- [x] Ch224 — Imitation Learning for Compiler Heuristics: BC-Max and Behavioral Cloning (~12 pp; §224.1 why RL is hard, §224.2 inlining as heuristic decision, §224.3 BC-Max framework, §224.4 feature engineering/MLGO, §224.5 register allocator eviction, §224.6 offline vs online deployment, §224.7 RL vs IL comparison, §224.8 generalizing to other heuristics)
- [x] Ch225 — Knowledge-Infused Evolutionary Search: Pass Synergy Graphs and ECCO (~12 pp; §225.1 beyond blind operators, §225.2 pass behavioral vectors, §225.3 pass synergy graphs, §225.4 knowledge-guided genetic operators, §225.5 ECCO causal reasoning, §225.6 hybrid framework, §225.7 ORC ReOptimizeLayer integration)
- [x] Ch226 — Hierarchical RL for Register Allocation and Code Optimization (~12 pp; §226.1 register allocation as MDP, §226.2 RL4ReAL decomposition, §226.3 hierarchical agent architecture, §226.4 Pearl generalized RL, §226.5 GNN-based IR representation, §226.6 training infrastructure, §226.7 comparison with classical approaches)
- [x] Ch227 — LLM-Guided Polyhedral Optimization: LOOPer and Agentic Auto-Scheduling (~12 pp; §227.1 polyhedral parameter selection problem, §227.2 LOOPer deep learning cost models, §227.3 agentic auto-scheduling, §227.4 Polly integration, §227.5 MLIR affine dialect integration, §227.6 comparison with Ansor/Halide, §227.7 limitations)

## Appendices
- [x] Appendix A — LLVM IR Quick Reference
- [x] Appendix B — MLIR Dialect Quick Reference
- [x] Appendix C — Command-Line Tools Cheat Sheet
- [x] Appendix D — Migration Notes: LLVM 18 → 22
- [x] Appendix E — Glossary
- [x] Appendix F — Object File Format Reference
- [x] Appendix G — DWARF Debug Info Reference
- [x] Appendix H — The C++ ABI: Itanium and Microsoft
- [x] Appendix I — LLVM and GCC: A Structural Comparison (~25–30 pages; §I.1 design philosophy/licensing, §I.2 IR comparison GENERIC/GIMPLE/RTL vs LLVM IR/MachineIR, §I.3 pass infrastructure, §I.4 backend/target description TableGen vs .md files, §I.5 frontend/diagnostics, §I.6 toolchain ecosystem table, §I.7 optimization comparison, §I.8 language/target support matrix, §I.9 ABI interoperability)
- [x] Appendix J — Grammar Notation Reference: BNF, EBNF, PEG, and Modern Alternatives (~15 pages; §J.1 historical background BNF→EBNF→ISO 14977, §J.2 BNF constructs, §J.3 EBNF/W3C variants, §J.4 notation variants in practice comparison table, §J.5 PEG ordered choice/lookahead/packrat, §J.6 Ohm, §J.7 ANTLR4 .g4, §J.8 TreeSitter grammar.js, §J.9 railroad diagrams, §J.10 Earley/Marpa, §J.11 selection guide decision table, §J.12 converting between notations)

---

## Suggested Improvements (running list)
*Add items here before compaction; review after resuming*

---

## Stats
- Total chapters: 239 + 10 appendices = 249 items (+ 10 existing chapter expansions complete) = 259 tracked items
- Completed: 254 / 259 (ch229, ch230, ch231, ch232, ch233 not yet written)
- Estimated pages written: ~2,900 (original) + ~108 (Part XXVI) + ~120 (Part XXVII) + ~86 (Part XXVIII) + ~72 (Part XXIX) + ~35 (7 ch expansions) + ~60 (Part XXX) + ~132 (Part XXXI Ch208–Ch218) + ~108 (Part XVI Ch219–Ch222) = ~3,621 total written; ~3,681 when ch223–ch227 complete
- Part XXX complete: Ch203–Ch207 all written (2026-05-03)
- 2026-05-16: added 3 self-modifying-code expansion items (Ch58, Ch108, Ch200) covering OSR, ReOptimizeLayer, kernel live patching, W^X dual-mapping, homoiconic examples
- 2026-05-16: added Part XXXI — Frontier AI Evolution (Ch208–Ch218, 11 chapters, ~132 pp) to plan and TODO
- 2026-05-16: Ch218 written — Self-Improvement Fitness Functions and Capability Assessment (oracle problem, linear probing, PRMs, self-eval loop, LiveBench, SAE interpretation-as-evaluation, regression testing, fitness landscape topology)
