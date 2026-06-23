# Part III — Type Theory — Part Summary

*This part develops the complete theoretical foundation of type systems — from the untyped lambda calculus through dependent types, linear logic, and refinement types — and maps each construct to its concrete manifestation in LLVM IR, MLIR, Clang, and the Rust/ML compiler ecosystem.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 12 | Lambda Calculus and Simple Types | Untyped λ-calculus, Church-Rosser, STLC soundness, Curry-Howard |
| 13 | Polymorphism and Type Inference | System F, Hindley-Milner, Algorithm W, type classes |
| 14 | Advanced Type Systems | Dependent types, linear/affine types, refinement types, effects |
| 15 | Type Theory in Practice: From Theory to LLVM and MLIR | Theory-to-IR mapping across LLVM, MLIR, Clang, Rust, ML compilers |

## Part Overview

Part III builds the formal type-theoretic foundation that underpins every safety and correctness argument in the rest of the book. It proceeds from the most primitive theoretical machinery — the untyped lambda calculus — to the most expressive modern type systems, and closes by grounding each theoretical construct in specific LLVM ecosystem artifacts.

Chapter 12 establishes the canonical baseline: the untyped lambda calculus and its reduction theory (Church-Rosser confluence, normalization strategies, the diverging Ω combinator), followed by the Simply Typed Lambda Calculus with its two foundational safety theorems (Preservation and Progress via the Wright-Felleisen syntactic approach). Strong normalization via Tait's reducibility candidates, the undefinability of the Y combinator, and the Curry-Howard correspondence — identifying types with propositions and programs with proofs — complete the chapter. Every subsequent chapter in the book implicitly invokes these results when it argues that a type system prevents a class of errors.

Chapter 13 addresses parametric polymorphism, the critical expressiveness gap that STLC cannot fill. System F (the second-order lambda calculus, developed independently by Girard and Reynolds) is presented with full typing rules, Church encodings, and Girard's reducibility-candidate proof of strong normalization. Existential types are shown to encode abstract data types — a connection made explicit through Rust's `impl Trait` and `dyn Trait`. Hindley-Milner is then derived as the decidable, rank-1 fragment of System F for which Algorithm W computes principal types via Robinson's unification. The chapter continues through the value restriction (soundness in the presence of mutable state), bidirectional type checking (the DK algorithm used by GHC, Rust, and TypeScript), row polymorphism (OCaml polymorphic variants), and the dictionary-translation elaboration of type classes — tracing the path from Haskell source all the way to LLVM IR function calls through vtable-like dictionary structs.

Chapter 14 dismantles the barrier between types and values that both STLC and System F maintain. It covers dependent Π and Σ types with the J eliminator, Martin-Löf Type Theory's four judgments, the λ-cube and the Calculus of Constructions as realized in Coq's CIC and Lean 4. Subtyping theory — width, depth, structural, nominal, and the S-Arrow contravariance/covariance law — is developed alongside System F<: and F-bounded polymorphism (Java generics). The chapter's second half treats resource-sensitive type systems: Girard's linear logic, affine and relevant types, Rust's borrow checker as a flow-sensitive affine type system with the NLL analysis, RustBelt's Coq/Iris machine-verified safety proof, uniqueness types and region types (Tofte-Talpin, Cyclone), algebraic effects and Koka's row-polymorphic effect system, refinement types and LiquidHaskell via SMT-discharged predicates, gradual typing and the blame calculus, session types and their Curry-Howard correspondence with linear logic, and finally ATS and F*/Low* as the state of the art in applying dependent and affine types to systems programming with LLVM as the ultimate target.

Chapter 15 is the bridge chapter that makes the preceding theory operational. It maps LLVM IR's type grammar (a flat, non-polymorphic instantiation of STLC with deliberate weaknesses such as signedness-free integers and opaque pointers) to the formal models, explains why every polymorphism strategy — C++ monomorphization, Rust monomorphization vs. `dyn Trait`, Go GC-shape stenciling, Haskell dictionary passing, Java erasure, Swift witness tables — resolves the `∀α. τ` of System F to ground types before reaching LLVM IR. MLIR's richer, theory-aligned type universe is then analyzed: parameterized types (`tensor<4x4xf32>` as a limited dependent-type constructor), dialect-defined open type universes, storage uniquing as structural identity, and type interfaces as ad-hoc polymorphism analogous to type classes. Clang's `QualType`/canonical-type split, Rust's MIR-level borrow checker, the GHC/MLton/OCaml type-erasure gradient, Alive2's refinement relation on LLVM IR programs, and C++'s `constexpr`/`consteval` as a restricted total-functional-programming sublanguage round out the chapter.

After completing this part, the reader can: state and prove Preservation and Progress for a typed calculus; derive free theorems from polymorphic types; trace the path from a Haskell function through System FC and STG to LLVM IR dictionaries; explain why Rust's `noalias` annotation is the affine typing guarantee expressed as LLVM IR metadata; characterize MLIR's type system as structurally identified parameterized types with verifier-enforced invariants; and situate any new type system feature — gradual types, effects, session types, dependent types — within the formal hierarchy established here.

## Key Concepts Introduced

- **Capture-avoiding substitution and α-equivalence**: the formal definition of substitution that prevents variable capture, and the quotient of lambda terms by renaming of bound variables; the basis for all reduction and typing results.
- **Church-Rosser confluence (parallel reduction proof)**: β-reduction is confluent; normal forms are unique; reduction order does not change the final result — the type-theoretic justification for commuting compiler passes.
- **Strong Normalization (Tait reducibility candidates)**: every well-typed STLC (and System F) term terminates; proved by defining semantic predicates `Red_τ` satisfying CR1–CR3, closed under reduction and verified by induction on types.
- **Type Soundness (Preservation + Progress)**: the Wright-Felleisen syntactic framework proving that well-typed programs never reach a stuck state; the standard method used throughout the book to argue that a type system prevents runtime errors.
- **Curry-Howard correspondence**: the formal bijection between type theory (STLC) and intuitionistic propositional logic; types are propositions, terms are proofs, β-reduction is cut elimination; extended to predicate logic via dependent types.
- **System F and parametric polymorphism (Reynolds parametricity)**: universal types `∀α. τ`; type abstraction and application; the abstraction theorem deriving free theorems from polymorphic types; Church encodings of all inductive types within System F.
- **Existential types as abstract data types**: `∃α. τ` hiding the concrete representation type; definable in System F via universals; the type-theoretic account of Rust's `impl Trait` / `dyn Trait` and Java wildcard types.
- **Hindley-Milner and Algorithm W**: rank-1 let-polymorphism with decidable, complete type inference; Robinson unification with the occurs check; generalization via `Gen(Γ, τ)`; principal types theorem; exponential worst-case from nested let expressions; the value restriction for soundness with mutable state.
- **Bidirectional type checking (DK algorithm)**: splitting the typing judgment into synthesis (`⇒`) and checking (`⇐`) modes; propagating expected types inward; enabling impredicative polymorphism and better error messages; used in GHC, Rust, Scala Dotty, and TypeScript.
- **Dependent Π and Σ types (Martin-Löf Type Theory)**: types that can reference runtime values; length-indexed vectors; the J eliminator for identity types; the λ-cube and the Calculus of Constructions; Coq's CIC with inductive types and the positivity condition; Lean 4's proof-irrelevant Prop and universe polymorphism.
- **Subtyping and System F<:**: width and depth subtyping for records; contravariance in function argument positions; structural vs. nominal subtyping; bounded quantification `∀(α <: T). τ`; F-bounded polymorphism (Java generics); Full F<: undecidability.
- **Linear, affine, and relevant types (Girard linear logic)**: restricting weakening and contraction; the `!` modality for unlimited use; affine types (at most once) as the model for Rust ownership; linear types for resources that must be consumed.
- **Rust's NLL borrow checker and RustBelt**: affine type system over MIR; lifetime variables as sets of program points; constraint generation and least-fixed-point solving; the xor-of-references invariant; RustBelt's Iris/Coq machine-verified safety proof; `noalias` as the IR encoding of exclusive references.
- **Refinement types, Liquid Types, and F*/Low***: base types decorated with SMT-dischargeable predicates; LiquidHaskell's abstract refinements and Z3 backend; F*'s Dijkstra monad effect system; Low* compilation through KreMLin to verified C and LLVM IR; HACL* and EverCrypt as production deployments.
- **Algebraic effects and session types**: Koka's row-polymorphic effect system; algebraic effects and handlers as first-class delimited continuations; session types encoding communication protocols; duality; the Curry-Howard correspondence between session types and linear logic; multiparty session types via global types.
- **Type erasure gradient (theory to LLVM IR)**: the systematic loss of type information from full dependent/refinement types at the source, through affine types in MIR, to nominal/structural types in LLVM IR, to bare bit widths at machine code level; each erasure step is a principled decision about which invariants are still needed downstream.

## How This Part Fits the Book

Part II (Compiler Theory, Chapters 6–11) provides the operational framework — lexing, parsing, AST construction, intermediate representations, control-flow and data-flow analysis, SSA form — that this part's type systems are implemented over. Part III is logically prior to Part IV (LLVM IR, Chapters 16–22) but depends on Part II for the concrete notion of intermediate representations; together Parts II and III supply the theoretical and formal vocabulary that Chapters 16–22 apply when describing LLVM IR's type grammar, SSA invariants (whose isomorphism with the lambda calculus is established in Chapter 12 §8), and the instruction-level semantics verified by Alive2 (Chapter 170). Parts V and VI (Clang frontend and code generation) rely heavily on Chapter 15's analysis of `QualType`, template-dependent types, and the monomorphization imperative; Part XIX and beyond (MLIR) build directly on Chapter 15's account of MLIR's extensible, theory-aligned type universe.

## Cross-Part Dependencies

The following chapters in other parts draw directly on material from Part III:

- Ch 16 (Part IV — LLVM IR Structure) — LLVM IR's type grammar is the ground instantiation of STLC described in Ch 15; named vs. literal struct nominality comes from Ch 14 §3.
- Ch 17 (Part IV — LLVM IR Type System) — Full treatment of LLVM IR types; theoretical foundation is Ch 12 (STLC) and Ch 15 §1.
- Ch 21 (Part IV — SSA, Dominance, and Loops) — SSA form's isomorphism with the lambda calculus (block parameters = λ-binders, φ-nodes = joins) established formally in Ch 12 §8.
- Ch 34 (Part V — Clang Frontend, Template Instantiation) — C++ templates as a monomorphization of System F instantiation (Ch 13 §2); C++20 Concepts as enforcement of type-constraint early detection (Ch 15 §4).
- Ch 36 (Part V — The Clang AST in Depth) — QualType and canonical types analyzed in Ch 15 §4.
- Ch 39 (Part VI — Clang Code Generation) — The descent from sugar to canonical types before IR generation, described in Ch 15 §4.
- Ch 47 (Part IV — clang-tidy and Tooling) — Static analysis at the AST level where sugar types are preserved; motivation from Ch 15 §4.
- Ch 77 (Part XIII — LTO) — Canonical-type-based name mangling for cross-TU template deduplication, explained in Ch 15 §4.
- Ch 127 (Part XVIII — Flang) — Fortran's type system; comparison with STLC base types and array typing from Ch 12 §6 and Ch 14 §1.
- Ch 132 (Part XIX — MLIR Foundations) — MLIR's type universe; parameterized types as restricted dependent types; storage uniquing; type interfaces — all analyzed in Ch 15 §3.
- Ch 154 (Part XXI — MLIR Transformations) — Shape inference and verifier-enforced invariants build on the MLIR type theory of Ch 15 §3.
- Ch 160 (Part XXII — XLA/OpenXLA) — Shaped tensor types and lowering; the theory behind MLIR's `tensor<NxMxf32>` is in Ch 15 §3 and Ch 14 §1.
- Ch 167 (Part XXIV — Verified Compilation, Operational Semantics) — Preservation and Progress framework from Ch 12 §7 applied to LLVM IR semantics.
- Ch 169 (Part XXIV — Vellvm) — Coq formalization of LLVM IR uses STLC-style typing judgment from Ch 12; RustBelt Iris techniques from Ch 14 §7.
- Ch 170 (Part XXIV — Alive2) — Refinement relation on LLVM IR programs; refinement as subtyping from Ch 14 §7; poison/undef in type lattice from Ch 15 §7.
- Ch 173 (Part XXV — Contributing to LLVM) — Understanding type-system design decisions requires the full theory of Ch 12–15 when proposing changes to LLVM IR's type grammar or MLIR's dialect-defined types.
- Ch 188 (Part XXVIII — Language Ecosystems, Rust) — Rust's borrow checker as affine type system; NLL; RustBelt; MIR — Ch 14 §7 and Ch 15 §5.
- Ch 190 (Part XXVIII — Language Ecosystems, Haskell/GHC) — HM inference, System FC coercions, dictionary passing, GHC Core to LLVM IR — Ch 13 and Ch 15 §6.

## Navigation

- ← Part II — Compiler Theory
- → Part IV — LLVM IR

---

*@copyright jreuben11*
