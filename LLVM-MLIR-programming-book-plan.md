# LLVM, Clang & MLIR — The Expert Programming Reference

### *With Compiler Theory, Type Theory, Polyhedral Theory, and Verified Compilation*
### *Targeting LLVM 22.x, MLIR (in-tree), Clang 22, OpenXLA, and ClangIR (April 2026)*

The complete expert reference for the LLVM compiler ecosystem. ~2200 pages across 176 chapters and 8 appendices in 25 parts. covers, in addition to the LLVM/MLIR/XLA/Clang implementation: the theoretical foundations of compilation (Dragon Book / Cooper-Torczon material), modern type theory (Pierce's TAPL / Harper's PFPL material), the derivation of the polyhedral model from Presburger arithmetic and integer programming, and the formal-verification approach to compilation pioneered by CompCert, Vellvm, and Alive2.

**Pin version:** LLVM 22.1.x. Practical chapters verified against the LLVM 22 release branch. Theoretical chapters verified against the canonical literature (cited per-chapter at draft time).

---

## Scope

**In:** Everything in the previous outline, plus four new theoretical Parts: Compiler Theory and Foundations (Part II), Type Theory (Part III), Polyhedral Theory (Part XI, before Polly), and Verified Compilation (Part XXIV, after MLIR in Production).

**Out:** Proof assistant internals beyond kernel level (e.g., the full Mathlib4 development methodology, Lean 4 metaprogramming library in depth); mathematical logic beyond what Ch185 covers (a full graduate logic curriculum); verified hardware beyond CHERI and seL4 (e.g., CHERI-x86, seL4 binary-level proofs, hypervisor verification); commutative algebra beyond Ch187's compiler-relevant applications (full algebraic geometry curriculum). *Note: the four areas previously listed as entirely out-of-scope (proof assistant internals, model theory, verified hardware, commutative algebra) are now covered in Part XXVII.*

**A note on size and density.** ~2200 pages at typical technical-book density. Theoretical chapters are denser per page — the Pluto algorithm derivation, the Hindley-Milner correctness proof, and the CompCert pass-by-pass semantic preservation argument each have higher information density than, say, the X86 backend chapter. Theoretical chapters average ~20 pages each rather than ~12. See the volume-split section.

**A note on verification.** It is CRITICAL that Non-theoretical chapters are verified against LLVM 22 code - **THE CODE MUST BE ACCURATE!!!**. Theoretical chapters cannot be verified against LLVM 22 code — they're verified against the foundational literature. Per-chapter drafting will require checking against:
- **Compiler Theory:** Aho/Lam/Sethi/Ullman (Dragon Book 2/e), Cooper-Torczon (Engineering a Compiler 3/e), Appel (Modern Compiler Implementation), Muchnick (Advanced Compiler Design and Implementation)
- **Type Theory:** Pierce (TAPL, ATTAPL), Harper (PFPL), Mitchell (Foundations for Programming Languages), Reynolds (Theories of Programming Languages)
- **Polyhedral Theory:** Bondhugula's PhD thesis, Feautrier's scheduling papers, Verdoolaege's ISL papers, the Polyhedral Compilation textbook by Grosser et al.
- **Verified Compilation:** Leroy's CompCert papers, Zhao/Zdancewic/Nagarakatte papers on Vellvm, Lopes/Lee/Hur papers on Alive2
- **Mathematical Foundations and Verified Systems (Part XXVII):** Lean 4 reference manual and LCNF compiler docs; Coq/Rocq reference manual; Nipkow/Paulson/Wenzel (Isabelle/HOL: A Proof Assistant for Higher-Order Logic); Church (1936) / Gödel (1931) primary sources; Enderton (A Mathematical Introduction to Logic); Watson/Woodruff/Woodruff (CHERI ISA specification); Watson et al. (CHERI: A Hybrid Capability-System Architecture); Klein et al. (seL4: Formal Verification of an OS Kernel, SOSP 2009); Mattern (The Formal Semantics of seL4, l4v project docs); Tarjan (A Unified Approach to Path Problems, 1981); Cox/Little/O'Shea (Ideals, Varieties, and Algorithms)

---

## Part I — Foundations *(~85 pages, 5 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 1 | The LLVM Project | History; the monorepo; full subproject map; release cadence; the umbrella model; what LLVM is *not*; downstream forks landscape |
| 2 | Building LLVM from Source | CMake; Ninja; `LLVM_ENABLE_PROJECTS` vs `LLVM_ENABLE_RUNTIMES`; bootstrap and 2/3-stage builds; ccache/sccache/distcc; cross-compilation; `llvm-cas`-backed builds |
| 3 | The Compilation Pipeline | The full pipeline diagram; where each tool sits; reading the cc1 invocation; pipeline diagrams for all major flows |
| 4 | The LLVM C++ API | RTTI; `LLVMContext`; ADT library; `Support` utilities; `Error` and `Expected`; `cl::opt`; visitor and iterator patterns |
| 5 | LLVM as a Library | `llvm-config`; CMake's `find_package(LLVM)`; component model; the C API; out-of-tree pass plugin and backend skeleton |

## Part II — Compiler Theory and Foundations *(~120 pages, 6 ch)*  

| # | Chapter | Topics |
|---|---------|--------|
| 6 | Lexical Analysis | Regular languages and regular expressions as a recognition formalism; Thompson's NFA construction; subset construction (NFA → DFA); Hopcroft and Brzozowski DFA minimization; lexer generators (`lex`, `flex`, `re2c`, `ragel`); the case for hand-written lexers (Clang, GCC); incremental and lazy lexing for IDEs |
| 7 | Parsing Theory | Context-free grammars; ambiguity and disambiguation; top-down: predictive parsing, LL(k), LL(*), recursive descent; bottom-up: LR(0), SLR(1), LALR(1), LR(1), GLR; Earley parsing; CYK; operator-precedence parsing; Pratt parsing; PEG and packrat; parser combinators; error recovery (panic mode, phrase-level, error productions); incremental parsing for IDEs; the Clang choice of hand-written recursive descent |
| 8 | Semantic Analysis Foundations | Symbol tables (chained, hashed, persistent); scoping models (lexical, dynamic, hierarchical); attribute grammars (synthesised vs inherited); syntax-directed translation schemes; the type-checking judgment as a tree walk; name resolution algorithms; the visitor pattern as compiled attribute grammar |
| 9 | Intermediate Representations and SSA Construction | Why an IR; AST vs three-address code vs CPS vs ANF; the SSA invariant restated formally; the Cytron-Ferrante-Rosen-Wegman-Zadeck dominance-frontier algorithm with derivation; pruned vs minimal vs semi-pruned SSA; out-of-SSA conversion (the lost-copy and swap problems; Briggs-Cooper-Harvey-Reeves and Sreedhar's algorithm); CPS↔SSA equivalence |
| 10 | Dataflow Analysis: The Lattice Framework | Partial orders, lattices, complete lattices; ascending and descending chain conditions; Tarski's fixed-point theorem; Kleene iteration and termination; monotone vs distributive frameworks; transfer functions; the MOP–MFP relationship and when they coincide; widening/narrowing for abstract interpretation; classical analyses derived: liveness, available expressions, reaching definitions, very-busy expressions, constant propagation; interprocedural extensions: call strings, k-CFA, m-CFA; context-sensitivity and the precision-cost tradeoff |
| 11 | Classical Optimization Theory | Local vs global vs interprocedural; common subexpression elimination via value numbering; loop-invariant code motion; strength reduction and induction-variable elimination; partial redundancy elimination (Morel-Renvoise, lazy code motion); register allocation theory: graph coloring (Chaitin), Chaitin-Briggs, iterated register coalescing (George-Appel), linear scan (Poletto-Sarkar), PBQP, SSA-based RA; instruction scheduling: list scheduling, trace scheduling, superblock and hyperblock scheduling, software pipelining (modulo scheduling); garbage collection theory: mark-sweep, copying (Cheney), generational, incremental, concurrent, region-based; mapping back to LLVM's actual implementations |

## Part III — Type Theory *(~80 pages, 4 ch)*  

| # | Chapter | Topics |
|---|---------|--------|
| 12 | Lambda Calculus and Simple Types | Untyped λ-calculus: syntax, α-equivalence, β-reduction; Church-Rosser confluence; weak vs strong normalization; simply typed λ-calculus (STLC); typing judgments and derivation trees; type safety theorems: progress and preservation; the Curry-Howard correspondence (STLC ↔ propositional intuitionistic logic); the connection between proofs and programs |
| 13 | Polymorphism and Type Inference | Parametric polymorphism; System F (second-order λ-calculus); existentials and abstract data types; the rank hierarchy; Hindley-Milner type system; Algorithm W with derivation; Damas-Milner; let-polymorphism and value restriction; principal types; bidirectional type checking; row polymorphism; type classes (Wadler-Blott) and dictionary translation; ad-hoc vs parametric polymorphism contrasted |
| 14 | Advanced Type Systems | Dependent types: Π and Σ types; Martin-Löf type theory; the Calculus of Constructions and Coq's CIC; Lean 4's dependent type theory; subtyping: width, depth, structural; bounded quantification (System F<:); linear logic and linear types; affine types and Rust's borrow checker formally; uniqueness types; region types and Tofte-Talpin; effect systems (Marino-Millstein, Koka); refinement types and Liquid types; gradual typing (Siek-Taha); session types; dependent types for systems programming (ATS, F*) |
| 15 | Type Theory in Practice: From Theory to LLVM and MLIR | LLVM IR's type system as a (mostly) nominal-structural hybrid without polymorphism; why generics monomorphize before reaching LLVM; MLIR types and the more theory-aligned design (parameterised types, type interfaces); how Clang implements C/C++ types (canonical types, sugar, dependent types in templates); Rust's borrow checker as a substructural type system over MIR; how ML compilers (HM-typed) elaborate to LLVM; refinement-style verification on LLVM IR via Alive2; the C++ `constexpr`/`consteval` calculus and its relationship to total functional programming |

## Part IV — LLVM IR *(~225 pages, 12 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 16 | IR Structure | Modules, functions, basic blocks, instructions; textual `.ll` vs bitcode `.bc`; printing/parsing; `llvm-dis`/`llvm-as`; module-level inline asm; module flags |
| 17 | The Type System | Primitives; arbitrary-width integers; FP types incl. `bfloat`/`fp128`/`x86_fp80`/`ppc_fp128`; vectors fixed and scalable; arrays and structs; opaque pointers (`ptr`); target extension types; `DataLayout` |
| 18 | Constants, Globals, and Linkage | `ConstantExpr`; `ConstantData`; globals; thread-locals (all four TLS models); aliases and ifuncs; full linkage taxonomy; `dso_local`; `dllimport`/`dllexport`; comdats |
| 19 | Instructions I — Arithmetic and Memory | `nsw`/`nuw`/`exact`/`disjoint`; FP fast-math flags; `getelementptr` index-calculation rules; load/store with align/volatile/atomic; `alloca`; `freeze`; `ptrtoint`/`inttoptr` and provenance |
| 20 | Instructions II — Control Flow and Aggregates | `br`/`switch`/`indirectbr`/`callbr`; `call`/`invoke`; `phi`; `select`; aggregate ops; vector ops; EH instructions (`landingpad`, `cleanuppad`, `catchpad`, `catchswitch`); `unreachable`/`ret`/`resume` |
| 21 | SSA, Dominance, and Loops | Phi placement; dominator/post-dominator trees; loops, headers, latches, exits; natural loops; LoopInfo; LoopNest; cycle terminology; iterated dominance frontier (cross-ref Ch 9) |
| 22 | Metadata and Debug Info | `!metadata`; `MD_tbaa`/`MD_range`/`MD_loop`/`MD_prof`/`MD_unpredictable`/`MD_alias_scope`; DWARF metadata family; the new debug-record format (RemoveDIs); pseudo-probe metadata |
| 23 | Attributes, Calling Conventions, and the ABI | Full attribute reference; attribute groups; calling conventions catalogue; the ABI lowering boundary; sret/byval/byref nuances |
| 24 | Intrinsics | Target-independent intrinsics; vector predication (`llvm.vp.*`); constrained FP and the FP environment; per-target intrinsic catalogues |
| 25 | Inline Assembly | Constraint syntax; register classes; `volatile`/`inteldialect`; memory clobbers; multiple alternatives; how Clang lowers `__asm__`; MS-style inline asm |
| 26 | Exception Handling | Itanium model; SEH (Windows); Wasm EH; `llvm.eh.*`; `__cxa_throw` to unwinder path; cross-language EH; setjmp/longjmp EH |
| 27 | Coroutines and Atomics | `llvm.coro.*` lowering; the four coro passes; switched-resume vs returned-continuation; atomic orderings; `cmpxchg` weak/strong; `atomicrmw`; sync scopes; the C/C++ memory model lowering |

## Part V — Clang Internals: Frontend Pipeline *(~210 pages, 11 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 28 | The Clang Driver | Driver vs `-cc1`; `ToolChain`; argument translation; multi-stage jobs; offload bundling at the driver level |
| 29 | SourceManager, FileEntry, SourceLocation | Compressed `SourceLocation`; FileID and offset encoding; macro-expansion locations; spelling vs expansion; line tables; `FileManager` and VFS |
| 30 | The Diagnostic Engine | `DiagnosticsEngine`; severity mapping; group hierarchies; `-W` plumbing; `clang-tblgen` for diagnostics; FixIt hints; SARIF; serialized diagnostics |
| 31 | The Lexer and Preprocessor | Token model; lexer state machine; macro expansion algorithm (object-like vs function-like, hygiene, rescanning); `__has_*` builtins; `#pragma`; conditional compilation; header search; the preprocessor as a library |
| 32 | The Parser | Recursive-descent organization; ambiguity resolution; error recovery; tentative parsing; Parser ↔ Sema action interface; specific challenges (most-vexing-parse, template angle brackets, requires-clauses); cross-ref Ch 7 for theoretical background |
| 33 | Sema I — Names, Lookups, and Conversions | Name lookup (unqualified, qualified, two-phase, ADL); overload resolution; implicit conversion sequences; reference binding; access control; ambiguity and visibility |
| 34 | Sema II — Templates, Concepts, and Constraints | Template parameter deduction; partial ordering; SFINAE; the `InstantiatingTemplate` stack; class/function/variable/alias template instantiation; variadic templates; concepts and requires-clauses; constraint normalisation and subsumption (cross-ref Ch 13) |
| 35 | The Constant Evaluator | `ExprConstant.cpp`; the `Evaluate` method family; integer constant expressions vs general constant expressions; `constexpr`/`consteval`/`constinit`; pointer/reference constant evaluation; `bit_cast`; immediate-invocation contexts |
| 36 | The Clang AST in Depth | `ASTContext`; `Decl` hierarchy; `Stmt`/`Expr` hierarchies; `Type` hierarchy (canonical types, sugar, dependent); `DeclContext` and lookup; `ASTReader`/`ASTWriter` |
| 37 | C++ Modules Implementation | Clang Modules (`-fmodules`); C++20 Modules; BMI files; module maps; the prebuilt-module-cache; module compilation database |
| 38 | Code Completion and clangd Foundations | `CodeCompleteConsumer`; semantic vs lexical completion; incremental AST building; index-while-building; background indexing; dynamic vs static index |

## Part VI — Clang Internals: Codegen and ABI *(~120 pages, 6 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 39 | CodeGenModule and CodeGenFunction | Module-level state; lazy emission; deferred decls; emission of globals; the function emission flow; debug-info and EH state per function |
| 40 | Lowering Statements and Expressions | `CGStmt`; `CGExpr` for lvalues and rvalues; aggregate vs scalar emission; complex-number emission; `CGExprConstant`; `CGExprAgg` |
| 41 | Calls, the ABI Boundary, and Builtins | `CGCall`; `ABIInfo` and per-target `TargetCodeGenInfo`; argument classification (SysV, Win64, AAPCS, AAPCS64, RISC-V); split aggregates; sret/byval/byref legalization; the builtin lowering tables |
| 42 | C++ ABI Lowering: Itanium | `CGCXXABI`/`ItaniumCXXABI`; `VTableBuilder`; construction vtables and VTT for virtual inheritance; thunks; thread-safe statics; RTTI emission; the Itanium name mangler |
| 43 | C++ ABI Lowering: Microsoft | `MicrosoftCXXABI`; vftables and vbtables; MS this-pointer adjustment; MS RTTI; MS thread-safe statics; the Microsoft name mangler; SEH-driven cleanup |
| 44 | Coroutine Lowering in Clang | `CoroutineStmt`; lowering to `llvm.coro.*`; promise-type integration; symmetric transfer |

## Part VII — Clang as a Multi-Language Compiler *(~140 pages, 7 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 45 | The Static Analyzer | Symbolic-execution engine; ExplodedGraph; checkers; path-sensitive vs flow-sensitive; cross-translation-unit analysis; writing custom checkers |
| 46 | libtooling and AST Matchers | `libtooling`; `CompilationDatabase`; `RecursiveASTVisitor` vs `DynTypedVisitor`; the AST-matchers DSL; rewriter-based source transformations |
| 47 | clangd, clang-tidy, clang-format, clang-refactor | clangd architecture and LSP feature mapping; check authoring; the `clang-format` engine; `clang-include-cleaner`; `clang-doc` |
| 48 | Clang as a CUDA Compiler | Single-source compilation; host vs device passes; offload bundler; PTX emission via NVPTX; libdevice |
| 49 | Clang as a HIP Compiler | HIP language vs CUDA; AMDGPU code object format; ROCm runtime integration; rocm-device-libs |
| 50 | Clang as SYCL, OpenCL, and OpenMP-Offload | SYCL 2020 single-source; the integration header; OpenCL frontend; OpenMP target offload |
| 51 | Clang as an HLSL Compiler | HLSL frontend; resource bindings; DXIL backend; SPIR-V output for Vulkan; root signatures; comparison with DXC |

## Part VIII — ClangIR (CIR) *(~50 pages, 3 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 52 | ClangIR Architecture | The CIR MLIR dialect; CIR's pipeline position; `-fclangir`/`-emit-cir`; upstreaming status |
| 53 | CIR Generation from AST | `CIRGenModule`/`CIRGenFunction`; AST-to-CIR translation; CIR types and ops; back-references to AST |
| 54 | CIR Lowering and Analysis | Direct-to-LLVM and through-MLIR-dialects lowering; the lifetime checker; idiom recognition |

## Part IX — Frontend Authoring (Building Your Own) *(~80 pages, 4 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 55 | Building a Frontend | Lexer/parser; AST design; the Kaleidoscope reference walkthrough; building a small expression language |
| 56 | Lowering AST to IR | `IRBuilder<>`; emitting expressions; visitor patterns; emission of control flow with phi; alloca-then-mem2reg; debug info alongside codegen |
| 57 | Lowering High-Level Constructs | Aggregates; arrays and bounds; vtables; closures; tagged unions; coroutines; string literals; RTTI emission |
| 58 | Language Runtime Concerns | GC strategies and statepoints; TLS lowering models; atomics and the C/C++ memory model; stack maps; signal-safety |

## Part X — Analysis and the Middle-End *(~210 pages, 11 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 59 | The New Pass Manager | `PassManager`, `AnalysisManager`; analysis vs transform passes; pass instrumentation; pipeline parsing; `default<O3>` |
| 60 | Writing a Pass | Out-of-tree pass plugin; `FunctionPass`/`LoopPass`/`ModulePass`/`CGSCCPass`; preserving analyses; pass registration |
| 61 | Foundational Analyses | DominatorTree; PostDominatorTree; LoopInfo; AliasAnalysis variants; MemorySSA; ScalarEvolution; DemandedBits; LazyValueInfo; BranchProbabilityInfo; BlockFrequencyInfo (cross-ref Ch 10) |
| 62 | Scalar Optimizations | DCE/ADCE/BDCE; GVN/NewGVN; SCCP/IPSCCP; InstCombine and InstSimplify; SimplifyCFG; Reassociate; EarlyCSE; CorrelatedValuePropagation; JumpThreading; TailCallElim |
| 63 | Loop Optimizations | LICM; LoopRotate; LoopUnroll; IndVarSimplify; LoopUnswitch; LoopIdiom; LoopDeletion; LoopFusion; LoopDistribute; LoopInterchange; LoopFlatten; LoopVersioning |
| 64 | Vectorization Deep Dive | The Loop Vectorizer; VPlan in detail; the SLP Vectorizer; cost models; scalable vectorization; predication and tail folding |
| 65 | Inter-Procedural Optimizations | Inliner; GlobalOpt; ArgPromotion; FunctionAttrs inference; DeadArgElim; MergeFunctions; PartialInlining; OpenMPOpt; AttributorPass |
| 66 | The ML Inliner and ML Regalloc | The RL-trained inliner; offline training pipeline; ML eviction policy in regalloc; the TFLite runtime dependency |
| 67 | Profile-Guided Optimization | Instrumentation-based PGO; SampleProfile/AutoFDO; pseudo-probes; CSSPGO; `llvm-profdata`; ThinLTO + PGO; PGO + BOLT |
| 68 | Hardening and Mitigations | CFI variants; SafeStack; ShadowCallStack; Speculative Load Hardening; Stack Clash; Stack Protector; PAuth/BTI; Intel CET/IBT; FORTIFY_SOURCE; Hot/Cold splitting |
| 69 | Whole-Program Devirtualization | The WPD pass; type-test/type-checked-load; LowerTypeTests; CFI/WPD shared infrastructure; integration with LTO/ThinLTO |

## Part XI — Polyhedral Theory *(~80 pages, 4 ch)*  

| # | Chapter | Topics |
|---|---------|--------|
| 70 | Foundations: Polyhedra and Integer Programming | Convex polyhedra; H-representation and V-representation; faces, facets, vertices; lattices and integer points; Hermite and Smith normal forms; linear programming and the simplex method; integer linear programming and branch-and-bound; Presburger arithmetic; quantifier elimination; the Omega test; ISL's algorithmic foundations |
| 71 | The Polyhedral Model | Iteration spaces as polyhedra; access functions and image computation; the Static Control Part (SCoP) and its restrictions; dependence relations (flow, anti, output, input); dependence polyhedra; exact dependence vs may-dependence; uniformization and parametric dependences; the affine schedule space |
| 72 | Scheduling Algorithms | Feautrier's multidimensional scheduling; Lim-Lam scheduling for affine partitioning; the Pluto algorithm: tiling-aware scheduling, the cost function derivation, permutability and tilability; affine transformations realised as schedules: skewing, fusion, fission, distribution, interchange, reversal; iterative compilation and autotuning; recent extensions (PoCC, isl scheduler) |
| 73 | Code Generation from Polyhedral Schedules | The CLooG algorithm; isl's AST generator; loop bound generation; loop fusion and distribution legality; tiling: hyperrectangular, parallelogram, time-skewed, diamond; generation of OpenMP parallel pragmas; SIMD-aware codegen; mapping to GPU code (the Polly-ACC and PPCG approaches); unroll-and-jam at the polyhedral level |

## Part XII — Polly *(~50 pages, 3 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 74 | Polly Architecture | ScopDetection and ScopInfo; the SCoP representation in Polly; integration with the LLVM pass pipeline; ISL inside Polly; comparison to Pluto's reference implementation |
| 75 | Polly Transformations | How Polly maps the theory of Part XI to LLVM IR; tile-size selection; loop fusion in practice; the JSON schedule format and exporting/re-importing schedules |
| 76 | Polly in Practice | When Polly helps and when it doesn't; auto-tuning; OpenMP parallel-loop generation; the GPU codegen path (experimental); current maintenance status |

## Part XIII — Link-Time and Whole-Program *(~80 pages, 4 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 77 | LTO and ThinLTO | Monolithic LTO; ThinLTO summary index; cross-module inlining/type propagation; LLD/gold plugin integration; Distributed ThinLTO |
| 78 | The LLVM Linker (LLD) | LLD architecture; ELF/COFF/MachO/Wasm drivers; symbol resolution; relocations and relaxations; linker scripts; partial linking; ICF |
| 79 | Linker Internals: GOT, PLT, TLS | The Global Offset Table; the Procedure Linkage Table; lazy binding; TLS dispatch (TLSDESC, IE, LE, GD, LD); IFUNC resolution; eh_frame layout; `.ctors`/`.dtors` and `.init_array` |
| 80 | llvm-cas and Content-Addressable Builds | The CAS object model; integration with Clang for module reuse; CASFS for hermetic builds; the CAS Plugin API |

## Part XIV — The Backend *(~280 pages, 14 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 81 | Backend Architecture | The codegen pipeline; LLT vs MVT vs EVT; SDAG vs GISel coexistence |
| 82 | TableGen Deep Dive | `.td` syntax; multiclass, foreach, defm, defvar, !-operators; record graph; backends (`-gen-instr-info`, `-gen-asm-matcher`, `-gen-dag-isel`, `-gen-global-isel`, `-gen-register-info`, `-gen-callingconv`, `-gen-subtarget`) |
| 83 | The Target Description | `TargetMachine`, `Subtarget`, `TargetLowering`; register classes; instruction info; calling convention description; predicates and features; `SchedMachineModel`, itineraries |
| 84 | SelectionDAG: Building and Legalizing | SelectionDAGBuilder; ISD opcodes; type and operation legalization; vector splitting and widening |
| 85 | SelectionDAG: Combining and Selecting | DAGCombiner; target DAG combines; pattern-based instruction selection; custom selection in C++; isel debugging |
| 86 | GlobalISel | IRTranslator → Legalizer → RegBankSelect → InstructionSelect; LLT type system; the GlobalISel combiner DSL; falling back to SDAG |
| 87 | Inline Assembly Lowering | Backend-side constraint handling; tied operands; clobbers and `INLINEASM`; integrated assembler interaction (cross-ref Ch 25) |
| 88 | The Machine IR | `MachineFunction`, `MachineBasicBlock`, `MachineInstr`, `MachineOperand`; properties; MIR text format; `PseudoSourceValue` and `MachineMemOperand` |
| 89 | Pre-RegAlloc Passes | TwoAddressInstructionPass; PHI elimination; live interval analysis; coalescing; pre-RA scheduling (machine combiner, `MachineScheduler`) |
| 90 | Register Allocation | The greedy allocator deep dive; live range splitting; spill weights; eviction; reg-bank-aware allocation; PBQP, basic, fast allocators; spill placement (cross-ref Ch 11) |
| 91 | The Machine Pipeliner | Software pipelining; modulo scheduling; Swing modulo scheduling; VLIW targets; current target uptake (Hexagon, PowerPC) |
| 92 | The Machine Outliner | The MachineOutliner pass; suffix-tree-based candidate detection; cost models; per-target hooks; size-vs-perf tradeoffs |
| 93 | Post-RegAlloc and Pre-Emit | Prologue/epilogue insertion; stack frame layout; branch folding; tail duplication; if-conversion; post-RA scheduling; bundling; MachineFunctionSplit; CFGuard |
| 94 | The MC Layer and MIR Test Infrastructure | `MCStreamer`, `MCAssembler`, `MCObjectWriter`; `MCInst` vs `MachineInstr`; assembly printing; integrated assembler; relocations and fixups; `MCDisassembler`; `.mir` format |

## Part XV — Targets *(~250 pages, 13 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 95 | The X86 Backend | GPR/XMM/YMM/ZMM/k registers; calling conventions; SSE/AVX/AVX-512; APX (EGPR, NDD); CET; complex addressing; `__attribute__((target))` and multiversioning; MS x64 ABI specifics |
| 96 | The AArch64 Backend | AAPCS64; NEON; SVE/SVE2; SME and streaming mode; PAuth; BTI; LSE atomics; Apple-specific variants; outlining |
| 97 | The 32-bit ARM Backend | A32/T32 ISA mix; Thumb interworking; AAPCS; VFP/NEON; Cortex-M; `arm-none-eabi` |
| 98 | The RISC-V Backend Architecture | Base ISA + scalar extensions; calling conventions (LP64, ILP32 variants); relaxation; subtarget features; profiles (RVA22, RVA23) |
| 99 | The RISC-V Vector Extension (RVV) | The V extension in detail; vtype/vl/vlenb; LMUL and SEW; tail-/mask-agnostic policies; tuple types; LLVM's scalable-vector type modelling; the VLS optimizer; Loop Vectorizer integration |
| 100 | RISC-V Bit-Manip, Crypto, and Custom Extensions | Zb*, Zk*, Zc*, Zfh/Zfa, Zicfilp/Zicfiss; vendor extensions (XTHead, XSiFive); how to add a custom extension upstream |
| 101 | PowerPC, SystemZ, MIPS, SPARC, LoongArch | Server-class targets compared |
| 102 | NVPTX and the CUDA Path | PTX as a target; address spaces; texture/surface; libdevice; `nvvm` intrinsics; tensor-core MMA intrinsics |
| 103 | AMDGPU and the ROCm Path | GCN/RDNA/CDNA wave model; SGPR/VGPR; LDS; SI scheduling; AMDGPU code object v5; rocm-device-libs; HSA queue ABI |
| 104 | The SPIR-V Backend | SPIR-V as a target; logical vs physical addressing; capability and extension model; entry-point handling; comparison with the Khronos translator |
| 105 | DXIL and DirectX Shader Compilation | The DXIL backend; resource binding lowering; root signature; comparison with DXC; signing and validation |
| 106 | WebAssembly and BPF | Wasm: SSA-on-stack mismatch, multi-value, SIMD128, reference types, GC proposal, exceptions, threads. eBPF: verifier-driven constraints, BTF, CO-RE relocations |
| 107 | Embedded Targets | AVR; MSP430; Hexagon (DSP, VLIW); M68k; Xtensa; Lanai; CSky; bare-metal toolchain considerations |

## Part XVI — JIT, Sanitizers, and Diagnostic Tools *(~210 pages, 11 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 108 | The ORC JIT | Layered architecture; JITDylib; lazy and speculative compilation; remote JIT; LLJIT helper |
| 109 | JITLink | The `LinkGraph`; section/atom-based linking; per-format passes; debugging JIT'd code |
| 110 | User-Space Sanitizers | ASan, TSan, MSan, UBSan, LeakSanitizer; sanitizer common runtime |
| 111 | HWASan and MTE | Hardware-Address Sanitizer; Memory Tagging Extension; KMTE; deployment in production |
| 112 | Production Allocators: Scudo and GWP-ASan | Scudo's hardened-allocator architecture; GWP-ASan sample-based detection; integration with Bionic and Fuchsia |
| 113 | Kernel Sanitizers | KASAN; KMSAN; KCSAN; KFENCE; kernel-side runtime requirements |
| 114 | LibFuzzer and Coverage-Guided Fuzzing | `-fsanitize=fuzzer`; SanitizerCoverage; the libFuzzer driver loop; structured fuzzing; OSS-Fuzz |
| 115 | Source-Based Code Coverage | `-fprofile-instr-generate -fcoverage-mapping`; the coverage mapping format; branch and MC/DC coverage; HTML and lcov export |
| 116 | LLDB Architecture | Process plugins; expression evaluator; DWARF + accelerators; Python scripting bridge; remote debugging; LLDB-DAP |
| 117 | DWARF and Debug Info | DWARF 5; split DWARF; DWP files; CodeView (Windows); how Clang emits debug info; `dsymutil` |
| 118 | BOLT and Post-Link Optimization | BOLT pipeline; perf-based profile collection; basic-block reordering; ICF; PGO + BOLT stacking |

## Part XVII — Runtime Libraries *(~110 pages, 6 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 119 | compiler-rt Builtins | Polyfills; soft-float; integer division; 128-bit ops; profile and coverage runtimes |
| 120 | libunwind | The unwinding ABI; `_Unwind_*` API; DWARF CFI vs SEH; libunwind on bare metal |
| 121 | libc++ | Architecture; ABI versioning; the inline-namespace trick; modules support; hardening modes; PSTL backends |
| 122 | libc++abi | The Itanium C++ ABI implementation; type info; thread-safe statics; demangling; `__dynamic_cast` |
| 123 | LLVM-libc | Goals and scope; the public-only-headers approach; per-function selection; testing infrastructure |
| 124 | OpenMP and Offload Runtimes | libomp; task scheduling; affinity; `omptarget` and liboffload; the `_OPENMP` macro version matrix |

## Part XVIII — Flang *(~80 pages, 4 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 125 | Flang Architecture and Driver | Flang vs Classic Flang; parser; semantic analysis; symbol tables; the driver (`flang-new`); module file format |
| 126 | The HLFIR and FIR Dialects | HLFIR semantics; FIR closer-to-LLVM; lowering between them; codegen down to LLVM IR |
| 127 | Flang OpenMP and OpenACC | OpenMP lowering via the OpenMP MLIR dialect; OpenACC pipeline; GPU offload through Flang |
| 128 | Flang Codegen and Runtime | The Fortran runtime; array intrinsics; I/O runtime; descriptor-based dispatch; do-concurrent lowering |

## Part XIX — MLIR Foundations *(~150 pages, 8 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 129 | MLIR Philosophy | Why "multi-level"; dialects as first-class units; progressive lowering; comparison with LLVM IR, Swift SIL, Rust MIR, GIMPLE |
| 130 | MLIR IR Structure | Operations; results, operands, attributes, regions, successors; SSA values; symbol tables; nested regions and isolation |
| 131 | The Type and Attribute Systems | Built-in types; defining custom types/attributes; storage uniquing; type/attribute interfaces (cross-ref Part III) |
| 132 | Defining Dialects with ODS | TableGen for MLIR; `Op<>`; traits and interfaces; verifiers; assembly-format DSL; constraints |
| 133 | Op Interfaces and Traits | `OpInterface`; common interfaces; SymbolOpInterface |
| 134 | The MLIR C++ API | `OpBuilder`, `RewriterBase`; `walk`; PatternRewriter API; locations and diagnostics |
| 135 | PDL and PDLL | Pattern Description Language; PDLL frontend; declarative rewrite specification; PDL vs DRR vs C++ patterns |
| 136 | MLIR Bytecode and Serialization | The bytecode format; round-tripping; versioning; cross-version compatibility |

## Part XX — In-Tree Dialects *(~190 pages, 10 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 137 | Core Dialects | `builtin`, `func`, `arith`, `math`, `complex`, `cf`, `scf`, `ub`; canonicalizations and folders |
| 138 | Memory Dialects | `memref` in detail; `bufferization`; one-shot bufferize; the deallocation pass |
| 139 | Tensor and Linalg | `tensor` ops; `linalg` named ops vs `linalg.generic`; indexing maps; iterator types; tiling and fusion; destination-passing style |
| 140 | Affine and SCF | The polyhedral subset (cross-ref Part XI); `affine.for`/`affine.if`; affine maps and sets; loop transformations |
| 141 | Vector and Sparse | `vector` dialect; `sparse_tensor` encoding and codegen |
| 142 | GPU Dialect Family | `gpu`, `nvgpu`, `nvvm`, `amdgpu`, `rocdl`; the host-device split |
| 143 | SPIR-V Dialect | Modeling; capability/extension model; entry-point ops; lowering from `gpu`/`vector`/`memref`; serialization |
| 144 | Hardware Vector Dialects | `amx`, `arm_sve`, `arm_sme`, `x86vector` |
| 145 | LLVM Dialect | The bottom of the MLIR tower; mapping MLIR types to LLVM types; translation to LLVM IR |
| 146 | Async, OpenMP, OpenACC, DLTI, EmitC | `async`; OpenMP dialect; OpenACC dialect; DLTI; EmitC |

## Part XXI — MLIR Transformations *(~110 pages, 6 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 147 | Pattern Rewriting | `RewritePattern`, `OpRewritePattern`; `applyPatternsAndFoldGreedily`; canonicalization; folder vs canonicalizer vs rewriter |
| 148 | Dialect Conversion | `ConversionTarget`, `TypeConverter`, `ConversionPattern`; partial vs full conversion; materialization; signature conversion |
| 149 | The Pass Infrastructure | Pass classes; nesting; pipeline grammar; pass instrumentation; multithreaded pass execution |
| 150 | The Transform Dialect | Transform IR as a script; named sequences; pattern interpreters |
| 151 | Bufferization Deep Dive | One-shot bufferize algorithm; `BufferizableOpInterface`; in-place vs out-of-place; the buffer deallocation pipeline |
| 152 | Lowering Pipelines | `linalg → tensor/scf → vector → memref → llvm`; conversion to LLVM IR via `mlir-translate` |

## Part XXII — XLA and the OpenXLA Stack *(~120 pages, 6 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 153 | XLA Architecture | OpenXLA; XLA's place between frameworks and hardware; the HLO compilation pipeline; relationship to MLIR; buffer assignment |
| 154 | HLO and StableHLO | HLO operation set; layouts and the layout-assignment pass; StableHLO portability layer; backward-compatibility; bytecode serialization; quantization; dynamism |
| 155 | XLA:CPU | LLVM IR emission; runtime calls (Eigen, oneDNN); XLA Runtime and RuntimeIR; AOT compilation |
| 156 | XLA:GPU | Native (LLVM/PTX) emitters vs Triton-based emitters; the priority-fusion cost model; CUDA graph extraction; cuBLAS/cuDNN/NCCL integration |
| 157 | PJRT — The Plugin Runtime Interface | The PJRT C API; framework-agnostic device plugins (CUDA, ROCm, TPU, Metal, Intel GPU); execution model |
| 158 | SPMD, GSPMD, and Auto-Sharding | Sharding annotations; the GSPMD partitioner; auto-sharding inference; collective generation; PyTorch-XLA SPMD |

## Part XXIII — MLIR in Production *(~160 pages, 8 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 159 | Building a Domain-Specific Compiler | A complete worked example; per-layer testing strategy |
| 160 | MLIR Python Bindings | `mlir-python-bindings`; `Context`, `Module`, `OpView`; building IR from Python |
| 161 | torch-mlir, ONNX-MLIR, and JAX/TF Bridges | torch-mlir; ONNX-MLIR; PyTorch and JAX hand-off through StableHLO |
| 162 | IREE — A Deployment Compiler | Architecture; the dispatch region model; backend targets; the VM and HAL |
| 163 | Triton — A Compiler for GPU Kernels | Triton's MLIR-based stack; the Triton dialect; tile-level abstractions; PyTorch integration |
| 164 | CUDA Tile IR | NVIDIA's MLIR-based dialect for tile-based tensor-core programming; type system for tiles; transformation passes |
| 165 | GPU Compilation Through MLIR | Kernel outlining; serialization to PTX/HSACO/SPIR-V; the host-device split |
| 166 | Mojo, Polygeist, Enzyme-MLIR, and Beyond | Mojo's dialect stack; Polygeist's C/C++ → MLIR; Enzyme-MLIR for autodiff; CIRCT for hardware |

## Part XXIV — Verified Compilation *(~110 pages, 5 ch)*  

| # | Chapter | Topics |
|---|---------|--------|
| 167 | Operational Semantics and Program Logics | Small-step vs big-step semantics; structural operational semantics; reduction-context semantics; Hoare logic with derivation; weakest preconditions; separation logic; concurrent separation logic; Iris; the relationship to the type-safety theorems of Part III |
| 168 | CompCert | The CompCert C subset and its rationale; the pass architecture in detail (Clight → C#minor → Cminor → CminorSel → RTL → LTL → LTLin → Linear → Mach → Asm); semantic preservation as the formal correctness statement; the Coq formalization workflow; the trusted base (parser, assembler, OS); empirical results (Csmith findings); industrial adoption (Airbus, MTU); CompCert's limitations and what it doesn't verify |
| 169 | Vellvm and Formalizing LLVM IR | The Vellvm project; the LLVM IR semantic model formally (small-step + interaction trees); memory models in Vellvm; the undef and poison semantics formally; verified passes (mem2reg as a flagship example); Vellvm vs CompCert: middle-end vs full-pipeline approach; the K-LLVM and Velldoc adjacent work |
| 170 | Alive2 and Translation Validation | Translation validation as a complement to full verification; Alive (peephole verification) → Alive2 (general); Z3-based verification of LLVM optimizations; the IR refinement relation formally; how Alive2 has driven LLVM-IR semantic clarification; the undef/poison/freeze refinement story; integration with LLVM's continuous integration; identified soundness bugs |
| 171 | The Undef/Poison Story Formally | The history: undef → poison → freeze → noundef; the semantic justification; why undef is being phased out; the IR refinement relation; the "twin allocation" model and why it was abandoned; the current status; remaining issues; what `noundef`, `noundef-on-result`, and frozen-poison semantics imply for backends; cross-references to Ch 19 (instruction semantics) and Ch 170 (verification) |

## Part XXV — Operations, Bindings, and Contribution *(~100 pages, 5 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 172 | Testing in LLVM and MLIR | `lit`; `FileCheck`; `update_test_checks.py`; `gtest`; `ninja check-all`; ABI tests; differential testing (Csmith, yarpgen) |
| 173 | Debugging the Compiler | `--debug-only`; `LLVM_DEBUG`; `-print-after`/`-before`/`-changed`; bisecting passes; reducing test cases (`llvm-reduce`, `bugpoint`, `mlir-reduce`); `llvm-symbolizer`; debug-info validation |
| 174 | Performance Engineering | Compile-time profiling; `-ftime-trace`; pass timing reports; quadratic-pass detection; the buildbot fleet |
| 175 | Language Bindings | The C API; the MLIR C API; OCaml; Python (llvmlite, mlir-python-bindings); Rust (Inkwell, llvm-sys); Go; Haskell |
| 176 | Contributing to LLVM | The GitHub workflow; commit-message conventions; pre-merge CI; the RFC process; release branch policy; security policy |

## Part XXVI — Ecosystem and Frontiers *(~108 pages, 7 ch)*

| # | Chapter | Topics |
|---|---------|--------|
| 177 | rustc: Architecture, MIR, and Codegen Backends | rustc architecture: driver → HIR → THIR → MIR → codegen; the Rust compiler dev guide as reference; MIR: `Place`/`Rvalue`/`Terminator`; MIR as Rust's mid-level SSA-like IR; borrow-checker output on MIR; **Miri** — the MIR interpreter for undefined-behaviour detection; execution model, extern function stubs, `cargo miri test`; **Polonius** — the new NLL-successor borrow checker; Datalog-based constraint formulation; the three problem variants (location-insensitive, Naive, DataFrog); comparison with Polonius v2 (a-mir-formality-based); **a-mir-formality** — the formal mechanised model of Rust's type system in PLT Redex; the formality type rules; how it drives Polonius v2 and trait solver work; `rustc_codegen_llvm`: `FunctionCx`; MIR → LLVM IR lowering; ADT/enum layout; trait-object vtable emission; Rust-specific LLVM attributes (`nounwind`, `noalias`, `dereferenceable`, `noundef`); panics and unwinding (`-C panic=abort` vs Itanium EH); LTO (`-C lto=thin/fat`); PGO; cross-compilation and target triples; rustc's vendored LLVM fork and version delta; **Cranelift** — the alternative codegen backend used by Wasmtime and `rustc_codegen_cranelift`; Cranelift IR (CLIF); the compiler pipeline (parsing → legalization → regalloc2 → emission); why Cranelift for fast debug builds; **GCC backend** (`rustc_codegen_gcc`): status and motivation |
| 178 | The Rust Compiler Ecosystem | **LLVM/MLIR Rust bindings**: `llvm-sys` — raw unsafe bindings to the LLVM C API; `inkwell` — safe, idiomatic LLVM IR builder wrapping `llvm-sys`; `iron-kaleidoscope` as a worked tutorial; `melior` — ergonomic MLIR bindings for Rust (Context, Module, OpBuilder, PassManager); `pliron` — MLIR-inspired extensible IR framework written in Rust (trait-based ops, dialect registry, SSA builder); Calyxir — compiler IR for hardware accelerators; **Object-file and debug-info crates**: `object` — unified read/write for ELF, Mach-O, PE/COFF, Wasm, XCOFF; `gimli` — lazy zero-copy DWARF read/write (used by rustc, `backtrace`, `perf`); `addr2line` — address→file:line symbolication via DWARF; **Compiler-infrastructure crates**: `ena` — union-find + congruence closure + snapshot/rollback extracted from rustc, used by Chalk and Polonius; **GPU and ML**: `rust-cuda` — device-side Rust targeting PTX via `nvptx64-nvidia-cuda`; Burn — ML framework with pluggable backends; CubeCL — GPU kernel DSL embedded in Rust (compile-time kernel specialisation, targets CUDA/Metal/Vulkan) |
| 179 | LLVM/MLIR for AI: The Full Stack | The AI compilation hierarchy: PyTorch/JAX eager → `torch.export`/`jax.jit` capture → StableHLO/HLO → MLIR dialects → target ISA; key dialects: `tosa`, `stablehlo`, `tensor`, `linalg.generic`, `vector`, `memref`, `gpu`; quantization: `quant` dialect, INT8/INT4/FP8 lowering; dynamic shapes and StableHLO dynamism extensions; hardware targets: CUDA (PTX via NVPTX), ROCm (HSACO via AMDGPU), NPUs (Apple ANE via CoreML, Hexagon DSP, Edge TPU); inference deployment stack: IREE, TFLite, ONNX Runtime, TensorRT; training vs inference compilation tradeoffs; FlashAttention 2/3 as kernel-fusion case study; end-to-end path: `torch.export` → torch-mlir → StableHLO → IREE dispatch → PTX/HSACO; synthesis across Parts XIX–XXIII |
| 180 | AI-Guided Compilation | MLGO: the `MLModelRunner` abstraction; the RL-trained inliner (feature vector, reward, training loop — cross-ref Ch66); ML-based register allocator eviction advisor; TFLite runtime embedded in LLVM; offline training and online serving model; ML-based cost models for instruction scheduling; Ansor/AutoTVM: search space definition, sketch-based sampler, cost model learning; IREE's ML-based tile size selection; Triton's autotuner (`@triton.autotune`); profile inference with ML; LLM-assisted compiler development: TableGen authoring, LLM-guided fuzzing (EvoFuzz, CovRL-Fuzz); LLVM IR as LLM training corpus; neural superoptimization (Bansal-Aiken); learned middle-end optimization policies |
| 181 | Formal Verification in Practice | **Push-button verifiers**: Dafny (MIT/Microsoft) — pre/post/invariants with Z3 discharge; ergonomics and limitations; LLM-assisted Dafny (96% benchmark pass rate 2026); Verus — Dafny-style verification for Rust, `requires`/`ensures`/`invariant` macros, the `verus!` proc-macro, ghost variables, `Tracked<T>`; F* (INRIA/Microsoft) — dependent types + SMT, the effect system, extraction to OCaml/C/Wasm; HACL* as the existence proof (verified crypto in Firefox NSS, Linux WireGuard); Why3 — meta-platform targeting multiple solvers; **Model checkers**: CBMC for C (bounded model checking); Kani for Rust (AWS, used on s2n-tls and Firecracker); TLA+/TLC/Apalache for protocol specs (AWS DynamoDB, S3); SPIN for concurrency; nuXmv for symbolic model checking; SV-COMP 2026 results (TACAS); **SMT/SAT engines**: Z3 (MIT), CVC5 (BSD), Bitwuzla (bitvector-heavy), Kissat (SAT); **Neural network verification**: α,β-CROWN (multi-year VNN-COMP winner, GPU-accelerated); Marabou (NYU); **Refinement types for Rust**: Flux — liquid-type checker as a Rust compiler plugin, qualifier maps, `#[flux::sig]`; **AI-assisted proof**: Lean Copilot (LLM tactics in Lean 4); LeanDojo (retrieval + training infrastructure); DeepSeek-Prover-V2, Kimina-Prover, Goedel-Prover (open-weight models trained against the Lean kernel via RL); 2026 vericoding benchmarks: Dafny 96%, Verus 44%, Lean 27% |
| 182 | Language Tooling: Parsers, Lexers, and Syntax Trees | **The tooling-vs-compilation axis**: when error-tolerance and incrementality matter more than correctness; **Lexer generators**: Logos (proc-macro, DFA-based, zero-copy, used widely in Rust); **PEG parsers**: Pest (`.pest` grammar files, PEG semantics, built-in error reporting); `peg` (proc-macro inline `peg::parser!` macro); `pom` (PEG combinators via operator overloading, no macros); **Parser combinators**: Winnow (actively maintained nom successor, streaming/partial parsing, built-in error context); `combine` (Parsec-style, zero-copy, streaming); `chumsky` (rich error recovery, built-in Pratt expression parser); **LR/LL parser generators**: LALRPOP (LR(1), `.lalrpop` grammar files, comprehensive book); Parol (LL(k)+LALR(1), dedicated book, `parol-ls` VS Code language server); Grmtools (YACC-style LR, Rust-native, lexxer + parser); **Lossless syntax trees**: Rowan (green/red tree architecture, used by rust-analyzer, preserves all whitespace and trivia); **ANTLR4**: LL(*) grammar syntax, visitor/listener patterns, the ANTLR4 C++ runtime; connecting an ANTLR4 parse tree to LLVM IR generation; **TreeSitter**: incremental GLR parsing; `.js` grammar authoring; the C API; Rust/Python bindings; use cases (syntax highlighting, structural navigation, `ts_query` pattern matching); why TreeSitter is unsuitable for full compilation; nvim-treesitter and LSP coexistence; **Comparison matrix**: hand-written recursive descent (Clang), ANTLR4, TreeSitter, PEG, LR, parser combinators — decision guide by use case |
| 183 | Modern C++ for Compiler Development: C++23, Contracts, and Reflection | **LLVM's C++ baseline and coding standards**: the LLVM coding standards; banned features (exceptions, RTTI/`dynamic_cast`); the monorepo minimum C++ standard (C++17, moving toward C++20); LLVM's own vocabulary types (`isa<>`, `dyn_cast<>`, `cast<>`, `SmallVector`, `StringRef`, `ArrayRef`, `DenseMap`, `LLVM_DEBUG`); the RFC process for standard upgrades; **C++20 in LLVM today**: concepts replacing SFINAE in ADT and type traits; `std::span` vs `ArrayRef<T>` in new APIs; `[[likely]]`/`[[unlikely]]` on hot paths; `constinit`/`consteval`; `std::bit_cast` replacing `memcpy` punning; the LLVM RFC process for C++20 adoption; **C++23 for compiler infrastructure**: `std::expected<T,E>` — design comparison with `llvm::Expected<T>`/`llvm::Error`, migration path; `std::mdspan` — multi-dimensional non-owning array views directly relevant to MLIR MemRef buffer descriptors and linalg tiling; `std::flat_map`/`std::flat_set` — sorted-vector associative containers matching `SmallDenseMap` performance profile; **deducing `this`** (P0847) — explicit object parameter; eliminating CRTP in `OpBase<>`, `PassBase<>`, `PatternBase<>`; concrete MLIR before/after examples; `std::ranges` views composition for IR iteration chains; **C++26 Contracts (P2900)**: `pre`, `post`, `contract_assert` syntax; violation semantics (`ignore`, `observe`, `enforce`, `quick-enforce`) and build modes; applying contracts to LLVM pass APIs — preconditions on `run(Function &, FunctionAnalysisManager &)` that IR is verified, postconditions that preserved analyses are consistent; contracts as complement to (not replacement for) the IR verifier and `AssertingVH`; contracts on `mlir::PatternRewriter` methods; current Clang implementation status (`-fcontracts`, experimental in Clang 22); the WG21 design history and P2900 trajectory; comparison with Dafny/Verus pre/post contracts (cross-ref Ch181); **C++26 Static Reflection (P2996)**: `^T` reflection of types, functions, members; `std::meta::info` as a compile-time handle; `[:r:]` splicers; `std::meta::members_of`, `std::meta::bases_of`, `std::meta::type_of`, `std::meta::name_of`; `std::meta::define_class` for compile-time class synthesis; applications for LLVM/MLIR: auto-generating TableGen descriptor structs from annotated C++ classes, replacing X-macro dialect/op registration tables, static generation of `getOperationName()` and `verify()` dispatch; comparison with Rust proc-macros (cross-ref Ch177/178); current status: EDG reference implementation, Clang experimental P2996 branch; expected WG21 trajectory toward C++26 IS; **C++26 pattern matching (P2688/P1371 status)**: `inspect` expressions; wildcard `_`, binding, type-test `<T>`, guard `if` patterns; structured binding patterns; applications to LLVM — replacing `isa<>`/`dyn_cast<>` chains in instruction visitors, MLIR Op dispatch, type hierarchy traversal; expected C++29 fallback if not C++26; comparison with Rust `match` and MLIR `PatternRewriter`; **The C++/Rust boundary**: why LLVM itself will not switch to Rust; what contracts + reflection would concretely improve in the C++ codebase; the `unsafe` analogy in C++ (undefined behaviour contracts); co-evolution of the two ecosystems through Rust's LLVM bindings (cross-ref Ch177, Ch178) |

## Part XXVII — Mathematical Foundations and Verified Systems *(~80 pages, 4 ch)* [THEORETICAL]

| # | Chapter | Topics |
|---|---------|--------|
| 184 | Proof Assistant Internals: Lean 4, Coq/Rocq, and Isabelle/HOL | **Lean 4 architecture**: the kernel as the sole trusted base (`Lean.Kernel.check` — does NOT perform unification; that is the elaborator's job); the core dependent type theory (CIC + universe polymorphism + proof irrelevance + quotient types); `Lean.Expr` AST; universe levels (`Level.succ`, `Level.max`, `Level.imax`); the elaboration pipeline: surface syntax → `Syntax` → `Lean.Elab.Term.TermElabM` → `Expr`; `Lean.Meta.M` monad for unification (`isDefEq`) and reduction (`whnf`); the tactic framework: `TacticM`, `omega`, `simp`, `decide`, `native_decide`, `aesop`; the compilation pipeline: `Lean.Compiler.LCNF` (λ-compiler normal form — an ANF-based IR using let-expressions) → IR with explicit reference-counting → C code emission; the experimental LLVM backend (PR #1837, WIP — generates LLVM IR directly, currently limited by inability to inline `lean_object` runtime intrinsics); how Lean 4 bootstraps; Lake build system; `lean4checker` and `Lean4Lean` (a verified Lean 4 typechecker written in Lean, arxiv 2403.14064); LeanDojo and proof search interacting with the kernel (cross-ref Ch181); **Coq/Rocq architecture**: the Calculus of Inductive Constructions (pCIC); the kernel: `check_inductive`, `infer`; the guard checker for fixpoints (structural recursion + size-based termination); `Prop`/`Set`/`Type` universe hierarchy; `SProp` for strict propositions; `Ltac` and `Ltac2` metaprogramming; `SSReflect`; the extraction mechanism: `Extraction Language OCaml|Haskell|Scheme`; how CompCert (cross-ref Ch168) uses extraction to produce a verified OCaml compiler; `native_compute` compiled reduction engine; `Equations` for dependent pattern matching; **Isabelle/HOL architecture**: the metalogic Pure (intuitionistic higher-order logic with schematic variables); HOL built on Pure; the `Thm.t` type as the only route to theorems (LCF approach); Isar structured proof language; `Sledgehammer` — calls external ATPs (Z3, CVC5, Vampire, E, SPASS) and reconstructs Isabelle proofs; how seL4 (cross-ref Ch186) uses Isabelle; Nitpick model finder; AFP (Archive of Formal Proofs); code generation to SML/OCaml/Haskell/Scala; **Comparison**: the three trusted bases; extracted/compiled code entering the LLVM pipeline; connection to Part III type theory (Ch12–15); connection to Ch170 (Alive2) and Ch181 (LLM provers); why CompCert chose Coq extraction, Vellvm chose Coq, and seL4 chose Isabelle |
| 185 | Mathematical Logic and Model Theory for Compiler Engineers | **Proof systems**: natural deduction (ND) for propositional and FOL; the sequent calculus LK; Gentzen's cut-elimination theorem and its computational content (normalisation = reduction — cross-ref Curry-Howard, Ch12); Hilbert-style vs ND vs sequent calculus compared; **First-order logic**: syntax (terms, function symbols, atomic formulas, connectives, quantifiers); Tarski semantics (structures/interpretations, variable assignments, satisfaction); soundness; Henkin's completeness proof; Gödel's compactness theorem; the upward and downward Löwenheim-Skolem theorems; **Incompleteness**: Gödel's first incompleteness theorem (diagonal lemma); the second incompleteness theorem; Tarski's undefinability of truth; what this means for formal verification in practice — why undecidability of Peano Arithmetic is not an obstacle (the theories compilers actually need — Presburger arithmetic, linear arithmetic over rationals — ARE decidable; cross-ref Part XI); **Decidable fragments and the SMT connection**: Church's theorem (full FOL is undecidable); the decidability spectrum: propositional logic (NP-complete, DPLL); quantifier-free linear arithmetic over integers (Presburger, decidable in 2EXPTIME, Omega test — cross-ref Ch70); quantifier-free linear arithmetic over reals (Fourier-Motzkin, LP); equality with uninterpreted functions (EUF, decidable via congruence closure); bit-vector arithmetic (BV, decidable); the DPLL(T) architecture for SMT: CDCL propositional solver + theory solvers for each decidable theory; Nelson-Oppen theory combination; the theories built into Z3 (T_LIA, T_NIA, T_BV, T_UF, T_Arrays, T_FP, T_RE) and how Alive2 (cross-ref Ch170) and Dafny (cross-ref Ch181) encode their VCs as FOL formulas in these theories; **Model theory**: structures and their theories; elementary equivalence and elementary embeddings; the compactness theorem (model-theoretic proof via ultraproducts); quantifier elimination (the model-theoretic perspective — vs the algorithmic Omega test); the model theory of algebraically closed fields and Hilbert's Nullstellensatz (preview of Ch187); **Higher-order logic vs FOL**: the expressivity gap; why HOL is undecidable; why Isabelle uses HOL and Coq/Lean use dependent type theory (cross-ref Ch184); the HOL family (HOL4, HOL Light, Isabelle/HOL); **Connections throughout the book**: Hoare logic (Ch167) as a proof system for programs; separation logic model theory; Z3's theory stack; lattice semantics of abstract interpretation (Ch10) as a model-theoretic structure; the Curry-Howard-Lambek correspondence bridging proof theory, type theory, and category theory |
| 186 | Verified Hardware: CHERI Capabilities and the seL4 Microkernel | **CHERI architecture and capabilities**: capabilities as 128-bit fat pointers — base, length (bounds), permissions bitmask, sealed bit, object-type field, and the out-of-band hardware tag bit (stored outside addressable memory, never forgeable); the CHERI monotonicity guarantee (bounds can only be narrowed, not widened in software); capability-based addressing vs address-space isolation vs MPU; **CHERI ISA variants**: CHERI-RISC-V (the primary research platform, integrated into RISC-V formal Sail spec — see below); Morello (Arm AArch64+CHERI, Neoverse N1 silicon, production hardware used in CheriBSD and Android research); CherIoT (Microsoft, RISC-V32 for IoT bare-metal, combines CHERI with seL4-style capabilities); CHERI-MIPS (original prototype); **CHERI LLVM** (`github.com/CTSRD-CHERI/llvm-project`): address space 200 as the CHERI capability address space (capabilities are non-integral — 64-bit address + 64-bit metadata, external state unlike AMDGPU buffer descriptors); `IntrinsicsCHERI.td` with `int_cheri_cap_bounds_set`, `int_cheri_cap_bounds_set_exact`; CHERI-aware `memcpy`/`memmove` intrinsics (preserve capability metadata); the CheriABI/PCuABI (pure-capability ABI): all memory access and syscall boundaries use capabilities, constraining both userspace and kernel; the Morello AArch64 backend and CHERI-RISC-V backend; `__capability` qualifier and `__intcap_t`; `__builtin_cheri_*` builtins; sub-object bounds enforcement; how CHERI changes the LLVM ABI layer (cross-ref Ch41, Ch96, Ch98); Cornucopia: the CHERI-based temporal memory safety extension using a revocation GC; comparison with ASan/HWASan (cross-ref Ch110, Ch111); **The Sail ISA specification language**: Sail as a typed DSL for ISA specs (bitvector-typed, effect-tracked); the official RISC-V Sail model (used for compliance test suite generation); Sail-to-C extraction (reference simulators); Sail-to-Isabelle extraction (for formal proofs); the CHERI-RISC-V Sail model; connection to LLVM RISC-V backend testing (Ch98–100): Sail model as ground truth; **seL4 microkernel verification**: L4 architecture: IPC as central mechanism; capability-based access control (software object capabilities — unrelated to CHERI hardware capabilities); kernel objects: TCBs, CNodes, VSpaces, Endpoints, Notification objects, Reply objects, Untyped memory; **The seL4 three-level refinement stack**: (1) abstract specification — the system interface as a functional state machine; (2) executable specification — a Haskell-prototype-style functional model; (3) C implementation (~10,000 lines C); two Isabelle/HOL refinement proofs: abstract→executable and executable→C; what the proofs guarantee: functional correctness (C matches spec), integrity (unprivileged code cannot modify kernel state), confidentiality (information-flow policy); what the proofs do NOT cover: hardware defects, timing side-channels, compiler correctness (the C compiler is in the trusted base — CompCert/Ch168 could close this gap); **AutoCorres and the C parser**: Michael Norrish's C parser translates seL4's C into Isabelle `Simpl` language; AutoCorres abstracts from byte-level heap to type-safe monadic form and generates Isabelle proof of the translation; the l4v project (~200,000 lines of Isabelle proof); **Multicore seL4 (WIP)**: the extension to multiprocessor seL4 with spinlock-based concurrency using rely/guarantee reasoning; 2024 status: a formal proof of the CLH lock in seL4 was published; full multicore seL4 proof is on hold pending funding; **CAmkES and deployments**: component architecture metalanguage; capDL (capability description language); DARPA HACMS; F-16 retrofit; seL4 on RISC-V; **seL4 and Rust**: `sel4-sys` crate; Ferrocene (safety-critical Rust, ISO 26262/IEC 61508); **CHERI + seL4 combined**: CherIoT combining hardware capabilities with seL4-style software capabilities on RISC-V32 as complementary orthogonal mechanisms; **Connecting to the book**: the seL4 trusted base gap closed by CompCert (Ch168); CHERI as hardware complement to ASan/HWASan (Ch110-111); Sail as formal ISA ground truth for LLVM backend correctness (Ch98-100) |
| 187 | Commutative Algebra and Its Applications in Compilation | **Ring theory foundations**: commutative rings and ideals; prime and maximal ideals; quotient rings; polynomial rings `k[x₁,…,xₙ]`; the Hilbert basis theorem (every ideal in `k[x₁,…,xₙ]` is finitely generated — Noetherian rings); Hilbert's Nullstellensatz (weak form: the radical ideal correspondence `I(V(J)) = √J`; strong form: varieties and ideals are in bijection for algebraically closed fields); connection to model theory (Ch185 Nullstellensatz as model completeness); **Gröbner bases**: monomial orderings (lex, grlex, grevlex); the multivariate division algorithm; S-polynomials; Buchberger's algorithm; reduced and minimal Gröbner bases; applications to compilation: ideal membership testing (constraint satisfaction); ideal intersection and elimination (projection of polynomial varieties — analogous to Fourier-Motzkin for linear arithmetic, cross-ref Ch70, Ch185); how SMT solvers use Gröbner bases in the theory of nonlinear arithmetic (T_NIA in Z3, cross-ref Ch185); **Algebraic geometry and polyhedral compilation**: toric varieties — varieties defined by monomial maps corresponding to fans and cones; the orbit-cone correspondence; the normal fan of a polytope; the Ehrhart polynomial `ehr(P, t) = |tP ∩ ℤⁿ|` (a polynomial in the dilation parameter `t`) and the Hilbert series of the associated semigroup ring; how Ehrhart theory provides a purely algebraic view of the lattice-point counting that underlies loop iteration counts in polyhedral compilation (cross-ref Ch70–73); connection to the generating-function approach to counting: the Barvinok algorithm as a rational function computation; **Abstract interpretation as algebra**: Galois connections as adjoint functors (cross-ref Ch10); numerical abstract domains as geometric/algebraic objects: intervals (order-1 projection), octagons (unit ball of ℓ∞ norm on the difference-constraint domain), polyhedra (intersection of halfspaces); the widening operator as a semidecision procedure for convergence; abstract domains as completions of the concrete powerset lattice; **Semirings in program analysis — the algebraic path problem**: Tarjan's unified treatment (1981): a dataflow analysis over a graph is an instance of the algebraic path problem `A* = (I - A)⁻¹` where `A` is the transfer-function matrix over a closed semiring; the Boolean semiring (reachability), integer/rational semiring (counting), (min,+) tropical semiring (shortest path / critical path / WCET), (max,+) max-plus semiring (latest firing in timed Petri nets — the well-established connection between tropical algebra and scheduling in discrete-event systems, not instruction-scheduling cost models); MLIR's analysis infrastructure (Ch149) as a closed semiring fixed-point engine; Mohri (2002) for weighted automata as the tropical semiring in action; **Effect systems and graded structures**: Koka's row-polymorphic effects as free commutative monoids; effect algebras (ordered abelian groups) as the semantics of Frank/Effekt; graded monads as effect algebras; connection to Ch14 (effect systems section); **Module theory connections**: free modules over a ring; the Lambek correspondence (the Curry-Howard-Lambek triangle: propositions = types = objects in a Cartesian closed category — cross-ref Ch12 and Ch185); linear logic propositions as modules over a quantale; connection to Rust's affine types and the substructural type-theory of Ch14 and Ch177; **Cox/Little/O'Shea** (Ideals, Varieties, and Algorithms) and **Ziegler** (Lectures on Polytopes) as primary references |

## Appendices *(~95 pages, 8 appendices)*

| # | Appendix | Contents |
|---|----------|----------|
| A | LLVM IR Quick Reference | Every instruction; full attribute reference; full intrinsic reference; metadata reference; the textual grammar |
| B | MLIR Dialect Quick Reference | Every in-tree dialect; one-page summary cards; bytecode versioning notes |
| C | Command-Line Tools Cheat Sheet | All ~40 binaries from `clang` to `llvm-bolt` |
| D | Migration Notes: LLVM 18 → 22 | Opaque pointers; RemoveDIs; pass manager API churn; GlobalISel API changes; MLIR API churn; deprecated and removed APIs |
| E | Glossary | ~250 terms |
| F | Object File Format Reference | ELF, Mach-O, COFF/PE, WebAssembly |
| G | DWARF Debug Info Reference | DWARF 5 sections; abbreviation tables; tag and attribute reference; CFI; `.debug_names`; split DWARF |
| H | The C++ ABI: Itanium and Microsoft | Cross-reference for Ch 42 and 43 |

---

## Estimated Page Distribution

| Part | Title | Chapters | Pages |
|------|-------|----------|-------|
| I | Foundations | 5 | 85 |
| II | **Compiler Theory and Foundations** | 6 | 120 |
| III | **Type Theory** | 4 | 80 |
| IV | LLVM IR | 12 | 225 |
| V | Clang Internals: Frontend Pipeline | 11 | 210 |
| VI | Clang Internals: Codegen and ABI | 6 | 120 |
| VII | Clang as a Multi-Language Compiler | 7 | 140 |
| VIII | ClangIR (CIR) | 3 | 50 |
| IX | Frontend Authoring (Building Your Own) | 4 | 80 |
| X | Analysis and the Middle-End | 11 | 210 |
| XI | **Polyhedral Theory** | 4 | 80 |
| XII | Polly | 3 | 50 |
| XIII | Link-Time and Whole-Program | 4 | 80 |
| XIV | The Backend | 14 | 280 |
| XV | Targets | 13 | 250 |
| XVI | JIT, Sanitizers, and Diagnostic Tools | 11 | 210 |
| XVII | Runtime Libraries | 6 | 110 |
| XVIII | Flang | 4 | 80 |
| XIX | MLIR Foundations | 8 | 150 |
| XX | In-Tree Dialects | 10 | 190 |
| XXI | MLIR Transformations | 6 | 110 |
| XXII | XLA and the OpenXLA Stack | 6 | 120 |
| XXIII | MLIR in Production | 8 | 160 |
| XXIV | **Verified Compilation** | 5 | 110 |
| XXV | Operations, Bindings, Contribution | 5 | 100 |
| XXVI | Ecosystem and Frontiers | 7 | 108 |
| XXVII | **Mathematical Foundations and Verified Systems** | 4 | 80 |
| | Appendices A–H | 8 | 95 |
| | **Total** | **187 chapters + 8 appendices** | **~2383 pages** |

---

## Suggested Volume Splits

A 2200-page single volume is a doorstop. Natural fascicle splits:

**Four-volume split (recommended at this size):**
- **Vol 1 — Foundations and Theory** (Parts I–III, IX, XI): ~445 pages — the conceptual base. Ships as "Compiler Theory, Type Theory, and the Polyhedral Model" — readable on its own without LLVM as a prerequisite.
- **Vol 2 — LLVM and Clang** (Parts IV–X, XII–XVIII): ~1665 pages... too big. Subsplit:
  - **Vol 2a — LLVM IR, Optimization, and Codegen** (IV, X, XII–XVI): ~1115 pages
  - **Vol 2b — Clang and Runtimes** (V–VIII, XVII–XVIII): ~870 pages
- **Vol 3 — MLIR and ML Compilation** (Parts XIX–XXIII): ~730 pages — the ML compiler stack
- **Vol 4 — Verified Compilation and Engineering Practice** (XXIV–XXVI + Appendices): ~399 pages

So really **six volumes** at this size:
- Vol 1: Theory (445 pp) — Parts I–III, IX, XI
- Vol 2: LLVM Core (1115 pp) — Parts IV, X, XII–XVI
- Vol 3: Clang and Runtimes (870 pp) — Parts V–VIII, XVII–XVIII
- Vol 4: MLIR and ML Compilation (730 pp) — Parts XIX–XXIII
- Vol 5: Verified Compilation + Operations + Ecosystem (413 pp) — Parts XXIV–XXVI
- Vol 6: Mathematical Foundations and Verified Systems + Appendices (175 pp) — Part XXVII + A–H

Total across volumes: ~3748 pages of *target* material — consolidated single set is ~2383 pages.

 
## Writing Guidance
- use a team of agents and /loop - keep context threshold in mind on planning how many chapters to do at once. 
- keep a TODO_checklist.md doc of what is done and what is next, so that continuation after compaction is smooth
- plan out each chapter before writing
- add reference links inline as needed
- review for accuracy of code after each chapter is done. then git commit.
- before compaction approaching compaction threshold, suggest improvements to content and anything that may be missing. keep a running list of suggested improvements in the TODO_checklist.md
---

---

---

*Outline pinned to LLVM 22.1.x and corresponding MLIR / OpenXLA snapshot as of April 2026. Theoretical chapters anchored to the canonical literature cited at the head. ClangIR, CUDA Tile IR, Triton, Mojo, ML Inliner/Regalloc, and the Verified Compilation chapters track moving research and engineering frontiers. Part XXVI (Ecosystem and Frontiers, 7 chapters) added April 2026: rustc internals (MIR/Miri/Polonius/a-mir-formality/Cranelift), Rust compiler ecosystem (inkwell/llvm-sys/melior/gimli/Burn/CubeCL), LLVM/MLIR for AI, AI-guided compilation, formal verification in practice (Dafny/Verus/F*/HACL*/Kani/Flux/LLM provers), language tooling (Logos/Pest/LALRPOP/Rowan/Chumsky/Winnow/ANTLR4/TreeSitter), and modern C++ for compiler development (C++23 std::expected/mdspan/deducing-this, C++26 Contracts/P2900, C++26 Static Reflection/P2996, pattern matching). Part XXVII (Mathematical Foundations and Verified Systems, 4 theoretical chapters) added April 2026: proof assistant internals (Lean 4 LCNF pipeline/Coq extraction/Isabelle LCF), mathematical logic and model theory (FOL completeness/incompleteness/decidable fragments/DPLL(T)/SMT), verified hardware (CHERI LLVM with address-space-200 capabilities/Morello/CherIoT/seL4 three-level refinement/Sail ISA spec), commutative algebra for compilation (Gröbner bases/toric varieties/Ehrhart theory/algebraic path problem/tropical semiring). Moves four previously out-of-scope areas in-scope.*


---

@copyright jreuben11
