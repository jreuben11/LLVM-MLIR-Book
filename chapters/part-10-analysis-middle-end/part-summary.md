# Part X — Analysis & Middle-End — Part Summary

*This part explains how LLVM's pass manager, foundational analyses, scalar/loop/vector optimizations, inter-procedural passes, machine-learning advisors, profile-guided feedback, security hardening transforms, and link-time devirtualization compose into a complete middle-end optimization pipeline.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 59 | The New Pass Manager | Pass/analysis infrastructure, `PassBuilder`, pipeline composition |
| 60 | Writing a Pass | Implementing function, loop, module, and CGSCC passes |
| 61 | Foundational Analyses | DominatorTree, LoopInfo, AliasAnalysis, MemorySSA, ScalarEvolution |
| 62 | Scalar Optimizations | DCE, GVN, SCCP, InstCombine, SimplifyCFG, ConstraintElimination |
| 63 | Loop Optimizations | LICM, LoopRotate, IndVarSimplify, Unroll, Fusion, Distribution |
| 64 | Vectorization Deep Dive | Loop Vectorizer, VPlan, SLP Vectorizer, SVE/RVV, cost modeling |
| 65 | Inter-Procedural Optimizations | Inliner, GlobalOpt, ArgPromotion, FunctionAttrs, Attributor |
| 66 | The ML Inliner and ML Register Allocation | RL-trained inline advisor, ML eviction policy |
| 67 | Profile-Guided Optimization | Instrumentation PGO, AutoFDO, CSSPGO, MemProf, Profi |
| 68 | Hardening and Mitigations | CFI, SafeStack, SLH, PAuth, CET, FORTIFY, `counted_by` |
| 69 | Whole-Program Devirtualization | WPD strategies, ThinLTO summary, CFI/WPD interplay |

## Part Overview

Part X begins by establishing the infrastructure that makes optimization possible. Chapter 59 introduces the new pass manager (NPM), explaining how passes and analyses are plain C++ types using concept-based polymorphism, how `AnalysisManager` caches and invalidates results through the `PreservedAnalyses` protocol, and how `PassBuilder` assembles the standard `-O3` pipeline from its five major phases. Chapter 60 puts this infrastructure into practice, walking through four complete pass implementations — function, loop, module, and CGSCC — including out-of-tree plugin loading and debugging with `opt-bisect` and `-verify-each`. These two chapters provide the foundation every subsequent chapter assumes: a reader who cannot write and register a pass cannot meaningfully extend the optimization pipeline.

Chapter 61 catalogs the foundational analyses that all optimization passes consume: the Lengauer-Tarjan dominator tree, natural-loop structure via `LoopInfo`, alias analysis chains (`BasicAA`, `TypeBasedAA`, `MemorySSA`), `ScalarEvolution`'s SCEV representation of induction variables, `DemandedBits`, `LazyValueInfo`, and the `BranchProbabilityInfo`/`BlockFrequencyInfo` pair. With the analysis layer established, Chapters 62 and 63 cover the optimization passes that consume these analyses. Chapter 62 treats scalar optimizations — dead-code elimination at three granularities (DCE/ADCE/BDCE), global value numbering (GVN and NewGVN), sparse conditional constant propagation (SCCP/IPSCCP), InstCombine's thousands of algebraic rewrites, SimplifyCFG's CFG canonicalization, and the linear-arithmetic constraint-elimination pass for redundant bounds-check removal. Chapter 63 covers loop-structure optimizations: LICM, LoopRotate, IndVarSimplify, LoopUnroll (including unroll-and-jam), LoopUnswitch, idiom recognition, deletion, fusion, distribution, interchange, flatten, and versioning. Each pass is shown in the context of what it requires (simplify form, MemorySSA, ScalarEvolution) and what it enables downstream.

Chapter 64 goes deep on vectorization. The loop vectorizer's four-phase architecture — legality, cost modeling via `TargetTransformInfo`, code generation, and VPlan — is covered in detail, followed by scalable vectorization for ARM SVE and RISC-V RVV (including EVL intrinsics and tail folding), reductions, first-order recurrences, and the SLP vectorizer's bottom-up tree construction. The experimental Sandbox Vectorizer built on SandboxIR's transaction-semantic wrapper is introduced as a safe research vehicle. Chapter 65 shifts to inter-procedural scope: the `InlinerPass`, `GlobalOpt`, `ArgPromotion`, `PostOrderFunctionAttrsPass`, `DeadArgumentElimination`, `MergeFunctions`, `PartialInlining`, `OpenMPOpt`, and the `Attributor` fixpoint framework. Chapter 66 describes LLVM's two production ML components — the reinforcement-learning-trained `MLInlineAdvisor` and the ML eviction policy for register allocation — covering feature extraction, TFLite AOT compilation, and the offline training loop.

Chapter 67 covers profile-guided optimization end to end: instrumentation-based PGO's two-stage build, edge profiling and value profiling, AutoFDO/SampleProfile, pseudo-probes for stable sample-to-IR mapping, context-sensitive PGO (CSSPGO), `llvm-profdata`, ThinLTO+PGO integration, and BOLT post-compilation reordering. Two deeper topics are treated: MemProf (context-sensitive heap allocation profiling enabling cold-allocator routing) and Profi (min-cost max-flow profile inference for incomplete instrumentation). Chapter 68 covers security hardening: CFI via `@llvm.type.test`, SafeStack, ShadowCallStack, Speculative Load Hardening, stack protectors, stack-clash protection, PAuth/BTI (AArch64), CET/IBT (x86), FORTIFY_SOURCE, hot/cold splitting, `-Wunsafe-buffer-usage` and Safe Buffers, the `counted_by` attribute for flexible array members, and the new `llvm.protected.field.ptr` intrinsic for use-after-free protection. Chapter 69 closes the part with whole-program devirtualization — the five WPD strategies (direct call, branch funnel, uniform return value, unique return value, virtual constant propagation), the shared `@llvm.type.checked.load` infrastructure it shares with CFI, ThinLTO summary-based export/import, and visibility requirements.

## Key Concepts Introduced

- **New Pass Manager (NPM)**: concept-based polymorphism for passes and analyses; `PassManager<IRUnitT>` type-erases `run(IRUnit&, AnalysisManager&)` via `PassModelT`; four IR-unit levels: `Module`, `Function`, `Loop`, CGSCC.
- **`PreservedAnalyses`**: the protocol by which a transform pass declares which cached analysis results survive its transformation; drives incremental invalidation in `AnalysisManager`.
- **`PassBuilder` and textual pipelines**: `buildPerModuleDefaultPipeline(OptimizationLevel)` constructs the standard five-phase O3 pipeline; `parsePassPipeline` parses the `default<O3>`, `function(pass1,pass2)`, `loop(licm)` textual format used by `opt -passes=`.
- **`AnalysisInfoMixin` / `AnalysisKey`**: the mechanism by which each analysis type acquires a unique identity (a static `AnalysisKey Key` whose address is the ID); enables O(1) cache lookup.
- **ScalarEvolution (SCEV)**: symbolic representation of loop-related expressions as `SCEVAddRecExpr {start, +, step}` and compound expressions; basis for induction-variable canonicalization, vectorization trip-count computation, and loop unrolling.
- **MemorySSA**: SSA-form memory access chains (`MemoryDef`, `MemoryUse`, `MemoryPhi`); `getWalker()->getClobberingMemoryAccess()` provides the nearest aliasing store for any load; used by LICM, DSE, and GVN.
- **VPlan**: a recipe-graph IR for the vectorized loop that decouples legality analysis from code generation, enabling vectorization factor selection by instantiating the same plan at multiple widths.
- **SandboxIR**: transaction-semantic wrapper over LLVM IR with `Tracker::save()` / `revert()` / `accept()`; used by the experimental Sandbox Vectorizer for speculative, rollback-capable IR mutations.
- **`InlineAdvisor` / `MLInlineAdvisor`**: the abstraction separating inlining policy from mechanism; the ML advisor uses a TFLite AOT-compiled model trained with reinforcement learning on large corpora to make inline/skip decisions.
- **Profile-Guided Optimization (PGO)**: `!prof` branch weight metadata from `llvm-profdata` feeds `BranchProbabilityAnalysis` and `BlockFrequencyAnalysis`; value profiling (`!vpid`) enables indirect-call specialization; CSSPGO maintains per-call-stack context for finer inlining granularity.
- **MemProf**: per-allocation-site heap profiling recording lifetime, access frequency, and call-stack context; `!memprof` IR metadata enables context-aware cold-allocator routing and struct field reordering.
- **Profi**: min-cost max-flow formulation over the CFG for inferring missing or inconsistent edge counts from partial instrumentation profiles; output feeds downstream `BranchProbabilityAnalysis`.
- **Control-Flow Integrity (CFI)**: `@llvm.type.test` and `@llvm.type.checked.load` intrinsics lowered by `LowerTypeTestsPass` to vtable-region range checks; shared with WPD; KCFI provides a per-module variant without LTO.
- **`counted_by` attribute**: `__attribute__((counted_by(field)))` annotates flexible array members with their element-count field, enabling precise `__builtin_dynamic_object_size()` values and activating FORTIFY_SOURCE and UBSan array-bounds checking for FAM accesses.
- **Whole-Program Devirtualization (WPD)**: five strategies (single-impl direct call, branch funnel, uniform return value, virtual constant, virtual constant propagation) applied at LTO/ThinLTO time using `@llvm.type.checked.load` intrinsics and `!vcall_visibility` metadata; summary-based two-phase export/import for ThinLTO.

## How This Part Fits the Book

Part X builds directly on Part IV (Chapters 35–49, LLVM IR) for the intermediate representation that all passes consume, and on Part II (compiler theory) for the lattice-based dataflow algorithms that underlie SCCP, GVN, and the Attributor. The foundational analyses of Chapter 61 also depend on the SSA construction theory of Part IV. Parts XII–XIV (Polly, LTO, and the Backend) consume the output of Part X: Polly (Part XII) receives loops already in simplify form with ScalarEvolution annotations, the LTO pipeline (Part XIII, Chapter 77) runs the CGSCC inliner, WPD, and ThinLTO passes described here, and the backend (Part XIV) receives function-level IR that has been fully scalar and vectorization optimized. The JIT and sanitizer chapters (Part XVI) also depend directly on this part: the sanitizer instrumentation passes (ASan, TSan, UBSan) interact with the hardening passes of Chapter 68, and the ORC JIT in Chapter 103 uses the NPM pass pipeline architecture from Chapter 59 to compile JIT-loaded modules.

## Cross-Part Dependencies

| Chapter | Part | Reason |
|---------|------|--------|
| Ch 35–41 (Part IV) — LLVM IR | IR Values, Types, Instructions, SSA | All passes in Part X operate on and consume LLVM IR; AnalysisManager caches per-IR-unit results |
| Ch 9 (Part II) — Dataflow Analysis | Lattice theory, Worklist algorithms | SCCP (Ch 62), the Attributor (Ch 65), and Profi (Ch 67) implement dataflow fixpoints over lattices |
| Ch 10 (Part II) — SSA Form | SSA construction, dominance | DominatorTree (Ch 61), MemorySSA (Ch 61), GVN (Ch 62) all exploit SSA's use-def chains |
| Ch 72 (Part XII) — Polly | Polyhedral loop transformation | Polly consumes ScalarEvolution (Ch 61) and loop-simplify form (Ch 63); LoopInterchange (Ch 63) duplicates some Polly functionality |
| Ch 77 (Part XIII) — ThinLTO | Link-time optimization | WPD ThinLTO summary export/import (Ch 69); PGO+ThinLTO cross-module inlining (Ch 67) |
| Ch 80 (Part XIV) — Instruction Selection | SelectionDAG, MachineFunction | Receives fully optimized IR from Part X; ML eviction advisor (Ch 66) runs inside the backend register allocator |
| Ch 96 (Part XVI) — Sanitizers | ASan, UBSan, TSan | Instrumentation passes interact with FORTIFY_SOURCE, `counted_by`, and `BoundsCheckingPass` (Ch 68) |
| Ch 103 (Part XVI) — ORC JIT | JIT compilation pipeline | Uses NPM `PassBuilder` (Ch 59) to build optimization pipelines for JIT-compiled modules |
| Ch 118 (Part XVI) — BOLT | Binary optimization tool | BOLT post-link profile-guided reordering complements compiler PGO (Ch 67 §67.7) |

## Navigation

- ← Part IX — Frontend Authoring
- → Part XI — Polyhedral Theory

---

*@copyright jreuben11*
