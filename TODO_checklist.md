# Book Progress Checklist
*Last updated: 2026-04-30 (Part XXVII planned — 4 new theoretical chapters Ch184–Ch187). 191 items complete; 4 new chapters planned, not yet written.*

## Status Legend
- `[ ]` Not started
- `[~]` In progress
- `[x]` Written and committed

---

## Current Focus
**Part XXVII — Mathematical Foundations and Verified Systems** (4 new theoretical chapters, Ch184–Ch187)
- Part XXVI complete (Ch177–Ch183 all written and committed)
- Next: write Ch184–Ch187 sequentially (all are theoretical, no inter-part cross-ref conflicts)

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

## Part XXVII — Mathematical Foundations and Verified Systems *(~80 pp, 4 ch)* [THEORETICAL]
- [ ] Ch184 — Proof Assistant Internals: Lean 4, Coq/Rocq, and Isabelle/HOL
- [ ] Ch185 — Mathematical Logic and Model Theory for Compiler Engineers
- [ ] Ch186 — Verified Hardware: CHERI Capabilities and the seL4 Microkernel
- [ ] Ch187 — Commutative Algebra and Its Applications in Compilation

### Chapter Plans

**Ch177 — rustc: Architecture, MIR, and Codegen Backends** (~20 pp)
1. rustc architecture: driver phases → HIR → THIR → MIR → codegen; the Rust compiler dev guide as reference
2. MIR deep dive: `Place`, `Rvalue`, `Terminator`; MIR as Rust's mid-level SSA-like IR; borrow-checker output on MIR; `mir_borrowck` query
3. Miri — the MIR interpreter: UB detection model; memory model (allocation-based, tag-based Stacked Borrows); extern function stubs; `cargo miri test`; limitations vs sanitizers
4. Polonius — new NLL-successor borrow checker: Datalog-based constraint formulation; three problem variants (location-insensitive, Naive, DataFrog engine); relation to the Chalk trait solver; Polonius v2 and a-mir-formality
5. a-mir-formality — formal mechanised model of Rust's type system: PLT Redex notation; the formality type rules; how it drives Polonius v2 and the next-gen trait solver
6. `rustc_codegen_llvm`: `FunctionCx`; MIR → LLVM IR lowering pass; ADT/enum layout; trait-object vtable emission; `dyn Trait` fat pointers
7. Rust-specific LLVM attributes: `nounwind`, `noalias`, `dereferenceable`, `noundef`; why they matter for optimisation
8. Panics and unwinding: `begin_unwind` path; Itanium EH vs `-C panic=abort`; personality function and landing pads
9. LTO: `-C lto=thin` (ThinLTO cache), `-C lto=fat`; PGO (`-C profile-generate`/`-C profile-use`)
10. Cross-compilation: target triples, `--target`, sysroot layout, `build.rs` target detection
11. Cranelift (`rustc_codegen_cranelift`): CLIF IR; the pipeline (parse → legalization → regalloc2 → emission); why it matters for fast debug builds; Wasmtime integration
12. GCC backend (`rustc_codegen_gcc`): status, motivation, libgccjit API surface
13. rustc's vendored LLVM: version delta strategy, custom patches, upgrade process

**Ch178 — The Rust Compiler Ecosystem** (~15 pp)
1. **LLVM Rust bindings**: `llvm-sys` (raw unsafe C API bindings, version feature flags); `inkwell` (safe, idiomatic `IRBuilder` wrapper; `Module`, `Function`, `BasicBlock`, `Builder`); `iron-kaleidoscope` as worked LLVM-in-Rust tutorial
2. **MLIR Rust bindings**: `melior` (Context, Module, OpBuilder, PassManager, Block API); mapping melior to the MLIR C API; `pliron` — MLIR-inspired extensible IR framework in Rust (trait-based Op/Type/Attribute, dialect registry, SSA builder); comparing melior vs pliron design philosophies
3. Calyxir — hardware-accelerator compiler IR: the Calyx language; lowering to RTL via CIRCT; use in FPGA-targeting ML compilers
4. **Object-file and debug-info crates**: `object` — unified ELF/Mach-O/PE/COFF/Wasm/XCOFF read-write; `gimli` — lazy zero-copy DWARF 5 read/write (used by rustc, `backtrace`, Linux `perf`); `addr2line` — DWARF-based address→file:line symbolication (Rust backtrace crate)
5. **Compiler infrastructure crates**: `ena` — union-find + congruence closure + snapshot/rollback extracted from rustc; used by Chalk trait resolver and Polonius
6. **GPU and ML from Rust**: `rust-cuda` — device-side Rust targeting PTX via `nvptx64-nvidia-cuda` target; Burn — pluggable ML framework (WGPU, CUDA, ROCm backends); CubeCL — GPU kernel DSL embedded in Rust (compile-time kernel specialisation, targets CUDA/Metal/Vulkan/WGPU); how Burn uses CubeCL for its fusion engine

**Ch179 — LLVM/MLIR for AI: The Full Stack** (~15 pp)
1. The AI compilation hierarchy: PyTorch/JAX eager → `torch.export`/`jax.jit` capture → StableHLO/HLO → MLIR dialect tower → target ISA
2. Key dialects: `tosa`, `stablehlo`, `tensor`, `linalg.generic`, `vector`, `memref`, `gpu` — roles and handoff points
3. Quantization: `quant` dialect encoding; INT8/INT4/FP8 lowering paths; GPTQ/GGUF integration points
4. Dynamic shapes: StableHLO dynamism extensions; `tensor.dim`/`shape` dialect; IREE's dynamic shape model
5. Hardware target landscape: CUDA (PTX via NVPTX), ROCm (HSACO via AMDGPU), Apple ANE (CoreML), Hexagon DSP (Qualcomm), Edge TPU
6. Inference deployment: IREE (multi-target, VM-based dispatch); TFLite (flatbuffer format, delegate API); ONNX Runtime (execution provider model); TensorRT (engine caching, layer fusion)
7. Training vs inference: throughput vs latency priorities; precision requirements; rematerialization for activation memory
8. Kernel fusion: FlashAttention 2/3 — tiling, softmax fusion, and causal masking as MLIR case study
9. End-to-end: `torch.export` → torch-mlir → StableHLO → IREE dispatch regions → PTX/HSACO — a traced walkthrough

**Ch180 — AI-Guided Compilation** (~12 pp)
1. MLGO architecture: the `MLModelRunner` abstraction; development mode (gRPC to Python server) vs release mode (compiled TFLite model)
2. The RL-trained inliner: feature vector construction, reward signal (size + perf delta), training loop; cross-ref Ch66
3. ML-based register allocator eviction: `RegAllocEvictionAdvisor`; priority model; how to retrain
4. ML-based cost models for instruction scheduling: learned latency/throughput tables
5. Ansor/AutoTVM: search space sketch-based definition; program sampler; learned cost model; relation to MLIR tiling
6. IREE's ML-based tile size selection: the tuning flow; serialised tuning configurations
7. Triton's autotuner: `@triton.autotune`; exhaustive vs evolutionary search; kernel caching
8. Profile inference with ML: synthesising PGO data from code structure (no instrumented run)
9. LLM-assisted compiler development: TableGen authoring, LLM-guided fuzzing (EvoFuzz, CovRL-Fuzz); LLVM IR as LLM training corpus
10. Neural superoptimization: Bansal-Aiken PLDI approach; learned peephole rule synthesis

**Ch181 — Formal Verification in Practice** (~20 pp)
1. Landscape overview: the four quadrants (push-button vs interactive, code vs spec)
2. **Dafny** (MIT/Microsoft): spec language (pre/post/invariant), ghost variables, `decreases` for termination; Z3 discharge; generics and traits in Dafny; limitations; LLM-assisted Dafny (96% benchmark pass rate, April 2026)
3. **Verus** — Dafny-style for Rust: `requires`/`ensures`/`invariant` macros; the `verus!` proc-macro; `Tracked<T>` and `Ghost<T>`; affine types meeting verification; AWS case studies; current maturity
4. **F*** (INRIA/Microsoft): dependent types + effect system + SMT discharge; extraction to OCaml/C/Wasm; HACL* — verified crypto in Firefox NSS, Linux WireGuard; the trusted base; Why3 as F*'s alternative backend target
5. **Why3** — meta-verification platform: WhyML language; targeting Z3, CVC5, Alt-Ergo, Coq; WP calculus; use in CompCert verification cross-ref
6. **Bounded model checking**: CBMC for C (unwinding assertions, pointer analysis); Kani for Rust (AWS, harness model, `kani::assume`/`kani::assert`; used on s2n-tls and Firecracker); comparison with sanitizers
7. **Temporal model checkers**: TLA+/TLC (exhaustive state enumeration) + Apalache (symbolic, SMT-backed); AWS DynamoDB/S3 case studies; SPIN for LTL/CSP concurrency; nuXmv for symbolic CTL/LTL
8. **SV-COMP 2026** (TACAS, Turin): categories and winners; CPAchecker, UAutomizer, Symbiotic; what the competition measures and doesn't
9. **SMT/SAT engines**: Z3 (MIT) — API basics, tactics, quantifier handling; CVC5 (BSD) — arithmetic and strings; Bitwuzla — bitvector/floating-point; Kissat (SAT, CDCL); how verifiers compose these
10. **Neural network verification**: α,β-CROWN (multi-year VNN-COMP winner, GPU-accelerated branch-and-bound); Marabou (NYU, simplex-based); use cases (safety-critical ML, adversarial robustness)
11. **Refinement types for Rust**: Flux — liquid-type checker as rustc plugin; `#[flux::sig]` annotations; qualifier maps; `RefinedBy`; current limitations
12. **AI-assisted proof**: Lean Copilot (LLM suggestions inside Lean 4 tactic mode); LeanDojo (retrieval-augmented proof search, training infrastructure); DeepSeek-Prover-V2, Kimina-Prover, Goedel-Prover — open-weight models trained via RL against the Lean kernel; 2026 vericoding benchmarks: Dafny 96%, Verus 44%, Lean 27%

**Ch182 — Language Tooling: Parsers, Lexers, and Syntax Trees** (~12 pp)
1. The tooling-vs-compilation axis: when error-tolerance, incrementality, and speed matter more than full correctness
2. **Logos** (Rust lexer generator): proc-macro DFA construction; zero-copy token types; `#[regex]` and `#[token]` attributes; performance characteristics
3. **PEG parsers**: Pest (`.pest` grammar files, built-in error reporting, `pairs()` API); `peg` (proc-macro inline `peg::parser!`); `pom` (PEG via Rust operator overloading, no macros); PEG semantics and packrat memoization tradeoffs
4. **Parser combinators**: Winnow (actively-maintained nom successor; streaming/partial parsing; built-in error context and span tracking); `combine` (Parsec-style, zero-copy, streaming); `chumsky` (rich error recovery via `Recovery`, built-in Pratt expression parser, migrated to Codeberg)
5. **LR/LL parser generators**: LALRPOP (LR(1), `.lalrpop` grammar files, the LALRPOP book); Parol (LL(k)+LALR(1), dedicated book, `parol-ls` VS Code language server); Grmtools (YACC-style LR, lexxer+parser suite, Rust-native)
6. **Lossless syntax trees**: Rowan (green/red tree architecture; immutable green tree + mutable red tree with parent pointers; used by rust-analyzer; preserves all whitespace and trivia; comparison with ANTLR4 parse trees)
7. **ANTLR4**: LL(*) adaptive parsing; `.g4` grammar syntax; visitor vs listener patterns; the ANTLR4 C++ runtime (`ANTLRInputStream`, `CommonTokenStream`); connecting to LLVM IR generation; error recovery strategies
8. **TreeSitter**: incremental GLR parsing; `.js` grammar authoring; the C API (`ts_parser_parse`, `ts_node_*`); Rust/Python bindings; `ts_query` pattern matching; why TreeSitter is unsuitable for full compilation; nvim-treesitter and clangd coexistence
9. **Decision guide**: choosing among hand-written recursive descent, ANTLR4, TreeSitter, PEG, LR generators, and combinator libraries by use-case (production compiler, IDE tooling, DSL, one-off tooling, incrementality requirement)

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
- Total chapters: 183 + 8 appendices = 191 items
- Completed: 191 / 191 (Ch183 written 2026-04-30 — all chapters complete)
- Estimated pages written: ~2,900 (original) + ~108 (Part XXVI actual) = ~3,008 total
