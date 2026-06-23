---
# Part XXIV — Verified Compilation — Part Summary

*This part develops the mathematical and engineering foundations for proving that compilers preserve program semantics — covering formal semantics, machine-checked proofs in Coq, automated SMT-based translation validation, and the precise formal treatment of LLVM IR's undefined behavior.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 167 | Operational Semantics and Program Logics | Small-step/big-step semantics, Hoare and separation logic, SMT solvers |
| 168 | CompCert | Machine-checked C compiler correctness via Coq simulation proofs |
| 169 | Vellvm and Formalizing LLVM IR | Coq formalization of LLVM IR semantics using interaction trees |
| 170 | Alive2 and Translation Validation | SMT-based per-compilation correctness checking for LLVM passes |
| 171 | The Undef/Poison Story Formally | Formal model of `undef`, `poison`, `freeze`, and LLVM undefined behavior |

## Part Overview

Compilers are among the most trusted components in a software stack, yet they are also among the largest and most complex programs humans have written. The intuition that "the compiler is correct" is tested constantly — by random testing tools like CSmith that find hundreds of silent miscompilations in GCC and LLVM — and yet the informal English prose of a language reference cannot serve as a foundation for machine-checked correctness. Part XXIV addresses this directly: it gives the reader both the mathematical machinery needed to reason about compiler correctness and detailed accounts of the major engineering efforts that have made verified compilation a practical reality.

Chapter 167 lays the theoretical substrate. Small-step operational semantics defines execution as a transition relation over configurations, giving a precise, inductive account of every step a program takes; this granularity is exactly what compilation pass proofs need. Hoare logic and separation logic provide specification languages for imperative programs, with the frame rule enabling compositional reasoning that scales to large codebases. Concurrent separation logic and the Iris framework extend these ideas to shared-memory parallelism, culminating in RustBelt's machine-checked proof that Rust's ownership type system is sound. Forward and backward simulation relations formalize what it means for a compiled program to be a faithful implementation of its source; the composition of simulations across passes is the mechanism by which per-pass proofs combine into a global compiler correctness theorem. SMT solvers — Z3, CVC5, Bitwuzla — close the loop by providing decidable automated decision procedures for the bitvector theories that model LLVM IR integer semantics.

Chapter 168 presents CompCert, the landmark Coq-verified C compiler from INRIA. CompCert compiles a substantial subset of C (Clight) through eight intermediate languages to PowerPC, ARM, x86-32, and AArch64 assembly. Each of the 15+ compilation passes carries an individual forward simulation proof; the `compose_forward_simulations` theorem chains them into a global backward simulation: every behavior of the assembly was already a behavior of the C source. CompCert's abstract block-offset memory model — where pointers are (block, offset) pairs rather than raw integers — provides the invariant needed by all pass proofs, and memory injections track how allocations transform across passes. The Iterated Register Coalescing allocator is verified by a validator rather than a proof of the algorithm, demonstrating that post-hoc checking is sometimes more tractable than pre-verified algorithms. CSmith found zero bugs in CompCert's verified passes while finding hundreds in GCC and LLVM, confirming the practical impact of the approach.

Chapter 169 turns to LLVM itself. The LLVM Language Reference Manual specifies LLVM IR in English prose, leaving `undef`, `poison`, pointer provenance, and concurrent memory ordering systematically ambiguous. Vellvm (University of Pennsylvania) gives LLVM IR a Coq formalization using algebraic data types for the IR AST and interaction trees (coinductive structures parameterized by effect signatures) for the semantics. Memory effects, undefined behavior, and nondeterminism are first-class effects whose handlers can be swapped to give different semantics (concrete, symbolic, or validated). The DVALUE/UVALUE distinction separates fully concrete values from those that may contain `undef` or `poison`; the `PickE` effect models `undef`'s nondeterminism; `UBE` models the triggering of undefined behavior. Verified transformations (DCE, constant propagation) are stated as ITree bisimilarity (`eutt`). Memory-aliasing-dependent optimizations remain an active research frontier.

Chapter 170 presents Alive2, which takes a complementary approach: rather than verifying the compiler algorithm in advance, it checks every specific compilation by encoding source and target IR as Z3 formulas and querying whether the refinement `refines(tgt, src)` holds. The refinement ordering captures LLVM's UB convention precisely: the target may not introduce UB where the source had none, but may narrow nondeterminism where the source had UB. Alive2's intercept mode — loaded as an `opt` pass that checks every transformation applied by `-O2` — found hundreds of real bugs in InstCombine, GVN, LICM, and SROA when first deployed in 2019. The `alive-tv` command-line tool is now standard practice for LLVM developers submitting new InstCombine rules.

Chapter 171 closes the part with a formal account of the `undef`/`poison`/`freeze` story — one of the most consequential design decisions in LLVM IR history. The original `undef` had "fresh at each use" semantics: each use of an `undef` SSA value could see a different bit pattern. This non-compositionality broke equational reasoning and caused subtle miscompilations. `poison`, introduced as a first-class value in LLVM 10, fixes this: it is a single stable "bad" value that propagates through computations and triggers UB only when used in a control-flow-determining context. The `freeze` instruction converts `undef`/`poison` to a stable concrete value, enabling loop optimizations that require a stable upper bound. Lee et al. (PLDI 2017) provided the formal model; Alive2 encodes it as per-value Z3 poison flags; Vellvm distinguishes `UVALUE_Undef` (fires `PickE`) from `UVALUE_Poison` (deterministic bad value).

## Key Concepts Introduced

- **Small-step operational semantics**: execution defined as a transition relation `⟨e, σ⟩ → ⟨e', σ'⟩`; the basis for inductive proof of compiler correctness over individual computation steps.
- **Separation logic and the frame rule**: extends Hoare logic with explicit heap ownership via separating conjunction `P * Q`; the frame rule enables compositional local proofs that compose globally without aliasing side conditions.
- **Forward simulation**: a relation between source and target states such that every source step is matched by one or more target steps with the same observable label; the per-pass proof obligation in CompCert; compositions give the global theorem.
- **CompCert's abstract block-offset memory model**: memory as a collection of abstract blocks (not a flat byte array); pointers are `(block, offset)` pairs; memory injections characterize how blocks map across compilation passes.
- **Memory injections (`Mem.inject`)**: the invariant relating source and target memory through passes that merge, split, or relocate allocations; proved to be preserved by every `alloc`/`free`/`load`/`store`.
- **Verified validator pattern**: rather than proving a complex algorithm (e.g., Iterated Register Coalescing) correct in Coq, run the algorithm and verify its output with a simpler Coq-proved checker; used in CompCert's register allocator.
- **Interaction trees (`itree E R`)**: coinductive structures with `Ret`, `Tau`, and `Vis` constructors; model effectful programs in a purely functional (Coq) setting; handlers interpret effects by replacing `Vis` nodes with ITree computations.
- **DVALUE / UVALUE distinction in Vellvm**: `DVALUE` for fully concrete values; `UVALUE` for values that may contain `undef`, `poison`, or unevaluated expressions; enables separate reasoning about concrete and nondeterministic execution.
- **`eutt` (equivalence up to tau)**: the ITree bisimilarity relation that abstracts over silent `Tau` steps; the statement of optimization correctness in Vellvm: `eutt (⟦ transform P ⟧) (⟦ P ⟧)`.
- **Translation validation**: instead of proving the compiler algorithm correct, verify that a specific compilation's output refines its input using an automated SMT decision procedure; the approach underlying Alive2.
- **Refinement relation `refines(tgt, src)`**: the target's behaviors must be a subset of the source's; the compiler may not introduce UB where the source had none, but may narrow UB where the source was undefined.
- **`poison` value**: a single stable "bad" value distinct from `undef`; produced by arithmetic with violated `nsw`/`nuw`/`exact`/`inbounds` flags; propagates through operations; triggers UB only when used as a branch condition, memory address, or other control-flow-determining position.
- **`freeze` instruction**: converts `undef`/`poison` to a stable concrete (but arbitrary) value; all uses of the frozen value see the same value; necessary before loop bounds, index variables, and any context where "fresh at each use" nondeterminism is unacceptable.
- **`noundef` attribute**: asserts that a function parameter or return value is not `undef` or `poison`; passing `poison` to a `noundef` parameter is immediately UB at the call site.
- **QF_BV (Quantifier-Free Bitvectors) SMT theory**: the theory of fixed-width integers with machine arithmetic operations; decidable by bit-blasting; the primary SMT theory used by Alive2 to encode LLVM IR integer semantics and verify optimization correctness automatically.

## How This Part Fits the Book

Part XXIV builds on the foundational LLVM IR semantics developed across Parts IV (Chapters 17–29, LLVM IR instruction set and type system), X (Chapters 57–68, analysis and middle-end optimization passes), and the ClangIR formalization sketched in Part VIII (Chapter 50). The simulation proof techniques and memory model introduced here also require the compilation pass architecture explained in Part V (Clang frontend) and Part XIV (backend passes). Parts XXV and beyond — particularly the operations and contribution guidelines of Part XXV (Chapters 172–176) and the frontier AI/PL topics of Parts XXX–XXXI — build on this part by assuming that readers can critically evaluate whether a new optimization pass is semantics-preserving, can read Alive2 bug reports, and understand when `freeze` is required; the verified compilation perspective also informs the correctness-by-construction approach advocated in Part XXIV's extended research roadmaps.

## Cross-Part Dependencies

- **Ch 10 (Part I) — LLVM IR overview**: Ch 167's bitvector semantics section directly extends the informal IR introduction; the QF_BV encoding in Ch 170 makes Ch 10's integer arithmetic description machine-checkable.
- **Ch 17–29 (Part IV) — LLVM IR instructions**: Chs 169–171 provide the formal semantics of every instruction type introduced in Part IV; `undef`/`poison` in Ch 171 is the formal account of behavior hinted at throughout Part IV.
- **Ch 50 (Part VIII) — ClangIR**: Ch 169's research roadmap explicitly targets a ClangIR-to-LLVM-IR translation validator as an Alive2-style extension; Ch 170's roadmap section discusses this connection.
- **Ch 57–68 (Part X) — Analysis and middle-end passes**: InstCombine (Ch 57-adjacent), GVN (Ch 59), LICM (Ch 62), and SROA (Ch 64) are the primary targets of Alive2 bug discovery described in Ch 170; understanding Ch 170 is prerequisite for writing new passes in Part X correctly.
- **Ch 70–73 (Part XI) — Polyhedral theory**: Ch 167's QF_LIA section references polyhedral analysis as an application of linear integer arithmetic SMT theories.
- **Ch 114 (Part XVI-adjacent) — KLEE and symbolic execution**: Ch 167's bounded model checking section references KLEE; the symbolic interpreter in Ch 169 is a related technique for the same problem class.
- **Ch 155–166 (Part XXIII) — MLIR in Production**: Ch 170 explicitly discusses an "Alive2 for MLIR" dialect lowering verifier as a mid-term research direction; the formal semantics approach of this part applies directly to MLIR dialect correctness.
- **Ch 172–176 (Part XXV) — Operations and Contribution**: the Alive2 workflow for verifying new InstCombine rules (`alive-tv` before PR submission) is the practical companion to the contribution guidelines in Part XXV; Ch 170's bug report format is directly relevant to filing LLVM bug reports.

## Navigation

- ← Part XXIII — MLIR in Production
- → Part XXV — Operations & Contribution

---

*@copyright jreuben11*
