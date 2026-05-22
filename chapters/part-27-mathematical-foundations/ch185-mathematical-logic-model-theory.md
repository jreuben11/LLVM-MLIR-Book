# Chapter 185 — Mathematical Logic and Model Theory for Compiler Engineers

*Part XXVII — Mathematical Foundations and Verified Systems*

Compiler engineers routinely cross into territory that demands mathematical logic fluency. Alive2 encodes LLVM undefined behavior as quantifier-free bit-vector formulas handed to Z3; Dafny translates pre- and post-conditions into first-order formulas over linear arithmetic; Polly operates within Presburger arithmetic, the decidable integer-linear fragment that Gödel's incompleteness theorems explicitly do not reach. Yet the same engineers often lack a unified theoretical map: why is one fragment decidable and another not? What does it mean for Z3's linear-arithmetic solver to be *sound* and *complete*? Why does Isabelle/HOL use a logic in which there is no complete proof procedure, and why does that choice not render it useless? This chapter provides that map. It begins with proof systems — natural deduction, the sequent calculus, and the Curry-Howard bridge — proceeds through the semantics of first-order logic and Gödel's incompleteness theorems, arrives at the decidable fragments that make SMT-based verification practical, and ends with the model-theoretic perspective that unifies the whole landscape.

---

## Table of Contents

- [185.1 Proof Systems](#1851-proof-systems)
  - [Natural Deduction](#natural-deduction)
  - [The Sequent Calculus LK](#the-sequent-calculus-lk)
  - [Gentzen's Cut-Elimination Theorem (Hauptsatz)](#gentzens-cut-elimination-theorem-hauptsatz)
  - [Hilbert-Style Systems vs Natural Deduction vs Sequent Calculus](#hilbert-style-systems-vs-natural-deduction-vs-sequent-calculus)
  - [Resolution and Robinson Unification](#resolution-and-robinson-unification)
- [185.2 First-Order Logic: Syntax and Semantics](#1852-first-order-logic-syntax-and-semantics)
  - [Syntax](#syntax)
  - [Tarski Semantics](#tarski-semantics)
  - [Soundness and Completeness](#soundness-and-completeness)
  - [Henkin's Completeness Proof](#henkins-completeness-proof)
  - [Compactness and Löwenheim-Skolem](#compactness-and-lwenheim-skolem)
- [185.3 Incompleteness](#1853-incompleteness)
  - [Gödel's First Incompleteness Theorem](#gdels-first-incompleteness-theorem)
  - [The Second Incompleteness Theorem](#the-second-incompleteness-theorem)
  - [Tarski's Undefinability of Truth](#tarskis-undefinability-of-truth)
  - [What Incompleteness Means for Compiler Verification](#what-incompleteness-means-for-compiler-verification)
- [185.4 Decidable Fragments and the SMT Connection](#1854-decidable-fragments-and-the-smt-connection)
  - [Church's Theorem and the Decidability Spectrum](#churchs-theorem-and-the-decidability-spectrum)
  - [The DPLL(T) Architecture](#the-dpllt-architecture)
  - [Nelson-Oppen Theory Combination](#nelson-oppen-theory-combination)
  - [Congruence Closure: The EUF Decision Procedure](#congruence-closure-the-euf-decision-procedure)
  - [Z3's Theory Stack](#z3s-theory-stack)
  - [SMT-LIB 2 Concrete Examples](#smt-lib-2-concrete-examples)
- [185.5 Model Theory](#1855-model-theory)
  - [Structures and Their Theories](#structures-and-their-theories)
  - [Elementary Equivalence and Embeddings](#elementary-equivalence-and-embeddings)
  - [Model-Theoretic Proof of Compactness via Ultraproducts](#model-theoretic-proof-of-compactness-via-ultraproducts)
  - [Quantifier Elimination](#quantifier-elimination)
  - [Algebraically Closed Fields and Hilbert's Nullstellensatz](#algebraically-closed-fields-and-hilberts-nullstellensatz)
- [185.6 Higher-Order Logic vs. First-Order Logic](#1856-higher-order-logic-vs-first-order-logic)
  - [The Expressivity Gap](#the-expressivity-gap)
  - [Why Isabelle Uses HOL](#why-isabelle-uses-hol)
  - [Why Isabelle Uses HOL and Coq/Lean Use Dependent Types](#why-isabelle-uses-hol-and-coqlean-use-dependent-types)
  - [The HOL Family](#the-hol-family)
- [185.7 Connections Across the Compiler Ecosystem](#1857-connections-across-the-compiler-ecosystem)
  - [Hoare Logic as a Proof System for Programs](#hoare-logic-as-a-proof-system-for-programs)
  - [Abstract Interpretation as a Model-Theoretic Structure](#abstract-interpretation-as-a-model-theoretic-structure)
  - [Z3 Enabling Alive2, Dafny, and Verus](#z3-enabling-alive2-dafny-and-verus)
  - [The Curry-Howard-Lambek Correspondence](#the-curry-howard-lambek-correspondence)
- [185.8 Summary](#1858-summary)
- [References](#references)

---

## 185.1 Proof Systems

A proof system is a purely syntactic device: given a set of axioms and inference rules, it generates theorems without referring to meaning. The interplay between syntax (provability, `⊢`) and semantics (truth, `⊨`) is the central tension of logic — resolved for first-order logic by Gödel's completeness theorem, and shown to be fundamentally unresolvable for arithmetic by his incompleteness theorems.

### Natural Deduction

Natural deduction (ND), introduced by Gentzen in 1935 and elaborated by Prawitz, writes proofs as trees of inference rules. Each connective has an *introduction* rule (how to prove it) and an *elimination* rule (how to use it). For propositional logic:

```
  [A]
   ⋮
   B
─────────  [→-I]        ─────────  [→-E] (modus ponens)
  A → B                 A → B   A
                        ─────────────
                              B

  A   B                  A ∧ B                A ∧ B
────────  [∧-I]         ──────  [∧-E₁]       ──────  [∧-E₂]
  A ∧ B                   A                    B

    A                     B
──────────  [∨-I₁]     ──────────  [∨-I₂]
  A ∨ B                  A ∨ B

  A ∨ B   [A]⋮C   [B]⋮C
  ─────────────────────  [∨-E]
             C

  [¬A]
    ⋮
    ⊥
──────────  [¬-I]        ─────────────  [¬-E]
   ¬A                    A   ¬A
                         ─────────
                             ⊥
```

For first-order logic, universal quantification adds:

```
  A[x/a]   (a fresh)                 ∀x. A    t a term
──────────────────────  [∀-I]        ─────────────────  [∀-E]
       ∀x. A                             A[x/t]

                                     ∃x. A    [A[x/a]]⋮C    (a fresh)
  A[x/t]                             ─────────────────────────────────  [∃-E]
──────────  [∃-I]                                    C
  ∃x. A
```

The bracketed `[A]` notation marks a *discharged assumption* — a hypothesis that is local to a subproof and is consumed by the conclusion. Natural deduction directly models mathematical proof structure; the `[→-I]` rule captures the mathematician's move of assuming `A`, deriving `B`, then concluding `A → B`.

To derive `A → A` in natural deduction:

```
  [A]¹
──────  (assumption)
  A
────────────────  [→-I, discharging ¹]
    A → A
```

### The Sequent Calculus LK

Gentzen's other invention, the sequent calculus LK (1935), writes judgments as *sequents* `Γ ⊢ Δ`, where `Γ` (the antecedent) and `Δ` (the succedent) are multisets of formulas. The intuitive reading: if all formulas in `Γ` hold, then at least one formula in `Δ` holds. LK has *left* rules (operating on the antecedent) and *right* rules (operating on the succedent) for each connective:

```
  Γ, A ⊢ Δ                            Γ ⊢ A, Δ    Γ ⊢ B, Δ
──────────────────  [¬-L]             ──────────────────────  [∧-R]
  Γ, ¬A ⊢ Δ                              Γ ⊢ A ∧ B, Δ

  Γ ⊢ A, Δ                            Γ, A ⊢ Δ    Γ, B ⊢ Δ
──────────────────  [¬-R]             ──────────────────────  [∨-L]
  Γ ⊢ ¬A, Δ                              Γ, A ∨ B ⊢ Δ
```

The **cut rule** plays a special role:

```
  Γ ⊢ A, Δ    Γ', A ⊢ Δ'
───────────────────────────  [Cut]
      Γ, Γ' ⊢ Δ, Δ'
```

Cut encodes the use of a lemma: prove `A`, then use `A` as a hypothesis. Every real mathematical proof uses cut — but Gentzen proved that it is *eliminable*.

### Gentzen's Cut-Elimination Theorem (Hauptsatz)

**Theorem (Hauptsatz, Gentzen 1935):** Every LK proof with cut can be transformed into a cut-free proof of the same sequent. [Gentzen, "Untersuchungen über das logische Schließen," *Mathematische Zeitschrift* 39, 1935.]

The proof proceeds by a double induction on the *cut rank* (the complexity of the cut formula) and the *proof height*. The elimination procedure replaces each cut with a (typically larger) derivation that avoids it.

**Computational content.** Cut-elimination is the proof-theoretic counterpart of *normalization* in the lambda calculus. Under the Curry-Howard correspondence, a proof is a program and cut is function application. Eliminating a cut of the form `(λx.e) e'` corresponds to β-reduction: substituting `e'` for `x` in `e`. This is the same normalization story told in [Chapter 12 — Lambda Calculus and Simple Types](../part-03-type-theory/ch12-lambda-calculus-and-simple-types.md): strong normalization for simply-typed λ-calculus is exactly Gentzen's cut-elimination for intuitionistic propositional logic.

For LK (classical logic), cut-elimination produces proofs that are not in normal form in the Curry-Howard sense — classical logic corresponds to control operators (`call/cc`), not simply-typed programs. This is the logical reason why classical logic and pure functional programming do not align cleanly.

### Hilbert-Style Systems vs Natural Deduction vs Sequent Calculus

The three main styles of proof system express the same theorems but differ substantially in mechanizability and theoretical tractability:

| System | Proof Objects | Key Virtue | Weakness |
|--------|--------------|------------|----------|
| Hilbert-style | Linear derivations from axioms | Minimal primitives (1–2 rules) | Proofs are enormous; hard for humans |
| Natural Deduction | Trees; discharging hypotheses | Matches mathematical practice; clean Curry-Howard | Structural rules require care |
| Sequent Calculus LK | Sequent trees; symmetric | Cut-elimination theorem; subformula property | Proof search requires managing multisets |

A Hilbert system for classical propositional logic uses just *modus ponens* and axiom schemas like:
- `A → (B → A)`
- `(A → (B → C)) → ((A → B) → (A → C))`
- `(¬A → ¬B) → (B → A)`

The proof of `A → A` in the Hilbert system takes three steps:

```
(1)  A → ((A → A) → A)               [Axiom 1: P → (Q → P) with P=A, Q=(A→A)]
(2)  (A → ((A → A) → A)) → ((A → (A → A)) → (A → A))
                                      [Axiom 2: (P→(Q→R))→((P→Q)→(P→R)) with R=A]
(3)  (A → (A → A)) → (A → A)         [MP on (1),(2)]
(4)  A → (A → A)                      [Axiom 1: P → (Q → P) with Q=A]
(5)  A → A                             [MP on (4),(3)]
```

Natural deduction accomplishes the same in two steps (as shown above). For *automated* theorem proving, sequent calculus is preferable because the subformula property (every formula in a cut-free proof is a subformula of the conclusion) makes proof search finite for propositional logic.

### Resolution and Robinson Unification

Resolution (Robinson 1965) is the basis for automated theorem proving in first-order logic and for Prolog. A *clause* is a disjunction of literals `L₁ ∨ L₂ ∨ ... ∨ Lₙ`. The *resolution rule* derives a new clause from two clauses containing complementary literals `L` and `¬L`:

```
  A₁ ∨ ... ∨ L ∨ ... ∨ Aₘ    B₁ ∨ ... ∨ ¬L ∨ ... ∨ Bₙ
  ──────────────────────────────────────────────────────────  [Res]
                A₁ ∨ ... ∨ B₁ ∨ ...
```

For first-order logic, `L` and `¬L` must be *unifiable* — Robinson's **unification algorithm** finds the most general unifier (MGU) σ such that `σ(L) = σ(L')`. Unification is the key to Prolog's execution model: a Prolog query succeeds if the head of a clause unifies with the goal.

```prolog
% Robinson unification in Prolog (demonstration)
?- f(X, g(Y)) = f(a, g(b)).
X = a, Y = b.   % MGU: {X ↦ a, Y ↦ b}

?- f(X, X) = f(g(Y), Y).
false.           % No unifier: would require X = g(X)
```

Resolution is *refutation-complete*: if a set of clauses is unsatisfiable, resolution can derive the empty clause (a contradiction). This completeness result (Robinson 1965) underlies both early theorem provers (RESOLUTION, Otter) and the SLD-resolution at the heart of Prolog.

---

## 185.2 First-Order Logic: Syntax and Semantics

### Syntax

A first-order language `L` consists of:
- **Non-logical symbols**: a set of *function symbols* `f, g, h, ...` each with an arity, and *predicate symbols* (relation symbols) `P, Q, R, ...` each with an arity. Constants are 0-ary function symbols.
- **Logical symbols**: connectives `∧, ∨, ¬, →, ↔`, quantifiers `∀, ∃`, variables `x, y, z, ...`, and equality `=`.

*Terms* are built inductively: every variable is a term; if `f` is an `n`-ary function symbol and `t₁, ..., tₙ` are terms, then `f(t₁, ..., tₙ)` is a term.

*Atomic formulas*: if `P` is an `n`-ary predicate and `t₁, ..., tₙ` are terms, then `P(t₁, ..., tₙ)` is an atomic formula. `t₁ = t₂` is atomic (equality).

*Formulas* are built from atomic formulas using connectives and quantifiers. A formula is *closed* (a *sentence*) if it has no free variables.

For example, the language of arithmetic `L_PA` has function symbols `{0, S, +, ×}` (0-ary constant `0`, unary successor `S`, binary `+` and `×`), and predicate symbol `{=}`. The sentence `∀x. ¬(S(x) = 0)` (no successor is zero) is a first-order sentence of `L_PA`.

### Tarski Semantics

A *structure* (interpretation) `M` for a language `L` consists of:
- A non-empty set `D` called the *domain* (or *universe*)
- For each `n`-ary function symbol `f`: a function `f^M : Dⁿ → D`
- For each `n`-ary predicate symbol `P`: a relation `P^M ⊆ Dⁿ`

A *variable assignment* `α : Var → D` maps variables to domain elements.

The *satisfaction relation* `M, α ⊨ φ` (M satisfies φ under α) is defined inductively:

```
M, α ⊨ P(t₁,...,tₙ)   iff  ⟨⟦t₁⟧^{M,α}, ..., ⟦tₙ⟧^{M,α}⟩ ∈ P^M
M, α ⊨ ¬φ             iff  M, α ⊭ φ
M, α ⊨ φ ∧ ψ          iff  M, α ⊨ φ  and  M, α ⊨ ψ
M, α ⊨ φ ∨ ψ          iff  M, α ⊨ φ  or   M, α ⊨ ψ
M, α ⊨ ∀x. φ          iff  M, α[x↦d] ⊨ φ  for all  d ∈ D
M, α ⊨ ∃x. φ          iff  M, α[x↦d] ⊨ φ  for some d ∈ D
```

where `⟦t⟧^{M,α}` is the interpretation of term `t` under `M` and `α`. A sentence `φ` (no free variables) holds in `M`, written `M ⊨ φ`, if `M, α ⊨ φ` for any (equivalently, some) `α`.

This semantics, due to Alfred Tarski (1936), is the standard meaning-conferring device for mathematical logic. It makes the notion of truth precise: `M ⊨ φ` is an entirely mathematical statement about a structure, not a philosophical notion of "truth."

### Soundness and Completeness

For a proof system `⊢` and satisfaction `⊨`:

- **Soundness**: If `Γ ⊢ φ`, then `Γ ⊨ φ`. Every provable consequence is semantically valid. This is verified by induction on proof structure — each rule preserves validity.
- **Completeness**: If `Γ ⊨ φ`, then `Γ ⊢ φ`. Every valid consequence is provable. This is non-trivial and was Gödel's great result.

Soundness and completeness together mean that proof-theoretic derivation and model-theoretic validity *coincide*. For a compiler engineer using an SMT solver: soundness means Z3 never reports `unsat` for a satisfiable formula; completeness means Z3 never reports `sat` for an unsatisfiable formula over the supported theories.

### Henkin's Completeness Proof

Gödel's completeness theorem (1930) states: every consistent first-order theory has a model. Gödel's original proof (1930 dissertation, published 1930) used a syntactic normalization argument. The modern proof, due to Henkin (1949), constructs a model *directly from the syntax* — the "canonical model" or "term model" construction. Henkin's approach is cleaner, more generalizable (it works for uncountable languages), and directly motivates the design of automated theorem provers.

**Stage 1 — Henkin extension.** Let `L` be a countable language and `T` a consistent `L`-theory. Introduce a new constant symbol `c_φ` for each formula `∃x.φ(x)` (countably many new constants, producing `L' ⊇ L`). Add the *Henkin axioms*:

```
H_φ  =  ∃x.φ(x) → φ(c_φ)    for each formula ∃x.φ(x)
```

**Claim:** `T ∪ {H_φ : φ}` is consistent in `L'`. Proof: any finite inconsistency would use finitely many `H_φ`, but the Henkin constants are fresh — a derivation of `⊥` from `T ∪ {H_{φ₁},...,H_{φₙ}}` could be turned into a derivation from `T` alone by substituting fresh variables for the Henkin constants, contradicting consistency of `T`.

**Stage 2 — Maximal consistent extension (Lindenbaum's lemma).** Given a consistent theory `T'` in `L'`, enumerate all `L'`-sentences `φ₀, φ₁, φ₂, ...`. Construct a chain:
```
T₀ = T'
T_{n+1} = T_n ∪ {φ_n}    if consistent
         = T_n            otherwise
```
Let `T* = ⋃ T_n`. Then `T*` is:
- **Consistent**: any finite subset is in some `T_n`, hence consistent.
- **Complete**: for every sentence `φ`, either `φ ∈ T*` or `¬φ ∈ T*` (one of them was added).
- **Has witnesses**: for every `∃x.φ(x) ∈ T*`, the Henkin constant `c_φ` satisfies `φ(c_φ) ∈ T*` (since `H_φ ∈ T'` and `∃x.φ(x) ∈ T*` implies `φ(c_φ) ∈ T*`).

**Stage 3 — The term model.** Define the *canonical structure* `M_{T*}`:
- Domain: ground terms of `L'` modulo `=` (the provable equality in `T*`), written `[t]`
- Interpretation: `f^M([t₁],...,[tₙ]) = [f(t₁,...,tₙ)]`; `P^M([t₁],...,[tₙ]) ↔ P(t₁,...,tₙ) ∈ T*`

**The satisfaction lemma** (proved by induction on formula complexity): `M_{T*} ⊨ φ ↔ φ ∈ T*`. For atomic formulas, by definition. For `¬φ`: `M ⊭ φ ↔ φ ∉ T* ↔ ¬φ ∈ T*` (by completeness). For `∃x.φ(x)`: if `∃x.φ(x) ∈ T*`, then `φ(c_φ) ∈ T*` (by witnesses), so `M ⊨ φ([c_φ])`, so `M ⊨ ∃x.φ(x)`. Conversely, if `M ⊨ ∃x.φ(x)`, there is a term `t` with `M ⊨ φ([t])`, so `φ(t) ∈ T*`, so `∃x.φ(x) ∈ T*`.

Since `T ⊆ T*`, `M_{T*} ⊨ T`. The theorem is proved.

**Why this proof matters for compiler verification.** The Henkin construction shows that consistency (no proof of `⊥`) is sufficient for satisfiability (existence of a model). In SMT terms: when Z3 reports `sat` for a QF_LIA formula and returns a model, it is explicitly constructing a finite version of the Henkin term model — an assignment to the declared constants that satisfies all the asserted constraints. The model is the *witness* that the constraints are consistent.

The Henkin proof is canonical: every completeness proof for FOL is essentially a Henkin construction in some form. [Enderton, *A Mathematical Introduction to Logic*, 2e, Academic Press 2001, Chapter 2.]

### Compactness and Löwenheim-Skolem

**Compactness Theorem:** A set of sentences `Σ` has a model iff every finite subset of `Σ` has a model.

The proof follows immediately from completeness: if every finite subset is satisfiable (consistent), the whole set is consistent (since a proof uses only finitely many premises), hence has a model by completeness.

**Application 1 — Non-standard arithmetic:** Consider `T = PA ∪ { c > n̄ : n ∈ ℕ }` where `c` is a fresh constant and `n̄` is the numeral for `n`. Every finite subset of `T` is satisfiable (interpret `c` as a sufficiently large natural number). By compactness, `T` has a model — a structure satisfying all of PA, plus an element `c` that is larger than every standard natural number. This is a *non-standard model of arithmetic*. Such models exist and are elementarily equivalent to `ℕ`, yet contain "infinite" numbers not present in the standard model.

**Application 2 — Four-color theorem as arithmetic:** The statement "every planar graph is four-colorable" can be expressed as an infinite conjunction of sentences of the form "planar graph G (presented by its adjacency list) is four-colorable." The compactness theorem implies: if every *finite* planar graph is four-colorable, then there exists a four-coloring of every planar graph (possibly infinite). This is not the four-color theorem itself, but compactness is a key tool in the model-theoretic analysis of combinatorial properties.

**Application 3 — Consistency of axiom systems extending PA:** If an axiom system is consistent, so is any finite extension of it (by compactness applied to the set of axioms). This is the model-theoretic reflection of the finiteness of proofs.

The *downward Löwenheim-Skolem theorem*: if a first-order theory has a model, it has a model of cardinality at most `max(|L|, ℵ₀)` where `|L|` is the number of non-logical symbols. In particular, every consistent countable theory has a countable model.

The *upward Löwenheim-Skolem theorem* (Tarski-Vaught): if a theory has an infinite model, it has models of every infinite cardinality.

Together, these theorems establish the *Löwenheim-Skolem paradox* (not a true paradox, but a philosophical tension): ZFC set theory, which proves the existence of uncountable sets (like `ℝ`), has a countable model by the downward LS theorem. In that countable model, the set playing the role of "ℝ" is countable — but the model cannot find an internal bijection between it and ℕ (since no such bijection *exists in the model*). "Uncountability" is a model-relative notion.

For compiler engineers, the practical content of these theorems is: **first-order logic is too weak to pin down the structure you intend.** When you write an axiom system for, say, machine-word arithmetic, there will be non-standard models satisfying all your axioms but not modeling the intended computation. This is why bit-vector arithmetic (which works modulo 2^32) is modeled as a finite structure — finite structures have no non-standard models, and the theory of a finite structure is decidable (by exhaustive evaluation).

---

## 185.3 Incompleteness

### Gödel's First Incompleteness Theorem

**Theorem (Gödel, 1931):** Any consistent, recursively axiomatizable theory `T` that extends Robinson Arithmetic (Q) is *incomplete*: there exists a sentence `G_T` such that neither `T ⊢ G_T` nor `T ⊢ ¬G_T`. [Gödel, "Über formal unentscheidbare Sätze der Principia Mathematica und verwandter Systeme I," *Monatshefte für Mathematik und Physik* 38, 1931.]

The proof has two components:

The proof has three conceptual components:

**Gödel numbering.** Every symbol, term, formula, and finite sequence of formulas is assigned a unique natural number (its *Gödel number*) via a primitive recursive encoding. For instance, one standard encoding assigns: `¬` ↦ 5, `∧` ↦ 7, `∀` ↦ 11, variable `v_i` ↦ `⟨13, i⟩` (pair encoding). A proof — a finite sequence of formulas — is then a natural number. The key insight is that syntactic operations on proofs (check whether a step follows by modus ponens; check whether the conclusion is a given formula) are all *primitive recursive* functions on natural numbers. The theory PA is strong enough to reason about primitive recursive functions.

**Diagonal Lemma (Fixed-Point Lemma):** For any formula `φ(x)` with one free variable, there exists a sentence `D` such that `T ⊢ D ↔ φ(⌈D⌉)`, where `⌈D⌉` is the Gödel number (numeral representation) of `D`.

*Proof sketch:* Define the *substitution function* `sub(m, n)` = Gödel number of the result of substituting the numeral `n̄` for the free variable in the formula with code `m`. This is primitive recursive. For any formula `φ(x)`, let `ψ(y) ≡ φ(sub(y, y))`, and let `d = ⌈ψ⌉`. Then `D ≡ ψ(d̄) ≡ φ(sub(d, d))`, and `sub(d, d) = ⌈D⌉`. So `T ⊢ D ↔ φ(⌈D⌉)`.

The diagonal lemma is the arithmetic self-reference mechanism. It encodes exactly the same diagonal construction as Cantor's proof that `|ℝ| > |ℕ|`, Russell's paradox, and the Turing halting problem — all are instances of the same structural pattern.

**Provability Predicate:** Define `Proof_T(m, n)` as the primitive recursive predicate "m is the Gödel number of a proof in T of the formula with code n." Then `Prov_T(n) ≡ ∃m. Proof_T(m, n)` is a Σ₁ formula (one existential quantifier over a decidable predicate) asserting "the formula with code n is provable in T." Such a formula exists because proof-checking is computable.

Applying the diagonal lemma to `¬Prov_T(x)`: there is a sentence `G_T` such that `T ⊢ G_T ↔ ¬Prov_T(⌈G_T⌉)`. In prose, `G_T` says "I am not provable in T."

- If `T ⊢ G_T`: then (in the standard model) `Prov_T(⌈G_T⌉)` is true arithmetically — there is a proof — but `G_T` asserts the negation. Assuming T is sound with respect to ℕ, this is a contradiction. (The Gödel sentence is *true* in ℕ but not provable in T.)
- If `T ⊢ ¬G_T`: then T proves there exists a proof of `G_T`, which makes T ω-inconsistent: T proves `∃m. Proof_T(m, ⌈G_T⌉)` but for each specific numeral k̄, T also proves `¬Proof_T(k̄, ⌈G_T⌉)` (assuming T actually has no such proof).

Under the **Rosser variant** (1936), the assumption of ω-consistency is weakened to simple consistency: Rosser strengthens the Gödel sentence to "for every proof of me, there is a shorter disproof of me," and proves this sentence undecidable under consistency alone.

### The Second Incompleteness Theorem

**Theorem (Gödel 1931):** If `T` is consistent and extends PA (Peano Arithmetic), then `T ⊬ Con(T)`, where `Con(T) ≡ ¬Prov_T(⌈0 = 1⌉)` is the arithmetic sentence encoding "T is consistent" (T does not prove the falsehood `0 = 1`).

The argument proceeds in two stages. First, one formalizes within T the reasoning from the first theorem: "if T is consistent, then `G_T` is not provable in T," written `T ⊢ Con(T) → ¬Prov_T(⌈G_T⌉)`. But `G_T ↔ ¬Prov_T(⌈G_T⌉)` is provable in T (by the diagonal lemma), so `T ⊢ Con(T) → G_T`. If T could prove `Con(T)`, it would prove `G_T`, contradicting the first theorem (T would then be inconsistent). Hence `T ⊬ Con(T)`.

The second incompleteness theorem has a dramatic practical implication: no sufficiently powerful formal system can serve as its own proof of consistency. ZFC set theory cannot prove its own consistency; to prove ZFC is consistent, one needs a stronger system (e.g., ZFC + large cardinal axioms). This is why proof assistants like Lean 4 and Coq rely on a *trusted kernel* rather than a self-certifying proof: the kernel is the external anchor of consistency that the system cannot provide internally.

For compilers, the most direct consequence is that a verified compiler implemented and verified *within* a system S (like CompCert in Coq) inherits whatever assumptions S makes. The correctness proof of CompCert does not prove Coq is consistent — it proves "if Coq is consistent, then this compiler is correct." The chain terminates at trust in the Coq kernel.

### Tarski's Undefinability of Truth

**Theorem (Tarski 1936):** The truth predicate for a language `L` is not definable in `L` itself (within arithmetic).

That is, there is no arithmetic formula `True(n)` such that for all sentences `φ`: `N ⊨ True(⌈φ⌉) ↔ φ`. If such a predicate existed, the diagonal lemma would yield a sentence equivalent to its own falsity — a contradiction.

Tarski's result implies that a meta-language must always be strictly stronger than the object language to define truth for the object language. This layers upward: arithmetic truth requires a second-order meta-language, or a set-theoretic framework (as in ZFC).

### What Incompleteness Means for Compiler Verification

These theorems are foundational but are often misunderstood as obstacles to formal verification. They are not. The critical point:

**Gödel's theorems apply to theories strong enough to encode arithmetic (PA and beyond). Compiler verification tools use decidable theories far weaker than PA.**

Specifically:
- Alive2 uses quantifier-free bitvector arithmetic (QF_BV) — decidable, not subject to incompleteness.
- Polly uses Presburger arithmetic (quantifier-free linear arithmetic over integers) — decidable in doubly-exponential time [Presburger 1929; Fischer-Rabin 1974].
- Dafny verification conditions live in quantifier-free linear arithmetic over integers and reals.
- The uninterpreted-function theories underlying equational reasoning about IR (EUF) are decidable.

Incompleteness bites only when one asks: "does this *arbitrary* first-order property of integers hold?" That question is undecidable. But compiler correctness questions are more specific: "does this bit-vector transformation preserve semantics?" or "does this loop body satisfy these linear constraints?" These questions live in decidable islands.

The stratification by decidability is the fundamental engineering insight:

```
Full PA (all of first-order integer arithmetic)
├── Undecidable (Gödel 1931, Church 1936)
├── Non-linear integer arithmetic
│   └── Undecidable (Hilbert's 10th, Matiyasevich 1970)
├── Presburger arithmetic (linear integer arithmetic, ∀/∃)
│   └── Decidable, 2EXPTIME (Fischer-Rabin 1974)
│   └── QF fragment: NP-complete (Omega test, Papadimitriou 1981)
│       └── ← POLLY, LOOP ANALYSIS LIVE HERE
├── QF_BV (fixed-width bit-vector arithmetic)
│   └── Decidable, NP-complete (via bitblasting to SAT)
│       └── ← ALIVE2 LIVES HERE
├── QF_LRA (linear real arithmetic)
│   └── Decidable, PTIME (LP, simplex)
│       └── ← RANGE ANALYSIS, FLOATING-POINT BOUNDS
└── EUF (equality with uninterpreted functions)
    └── Decidable, PTIME (congruence closure)
        └── ← CSE VALIDITY, PEEPHOLE EQUALITY REASONING
```

The design principle for SMT-based verification tools is to stay within decidable fragments. When a tool must handle the boundary (e.g., Dafny's heap reasoning requires quantifiers), it uses heuristics (E-matching, pattern-based instantiation) that are incomplete but practically effective for the specific VC shapes that verification condition generators produce.

---

## 185.4 Decidable Fragments and the SMT Connection

### Church's Theorem and the Decidability Spectrum

**Church's Theorem (1936):** The validity problem for first-order logic is undecidable (Σ₁-complete, RE-hard).

The decision problem "given φ, does every structure satisfy φ?" cannot be solved by any algorithm. This follows from the undecidability of the halting problem via a straightforward encoding.

The decidability spectrum of practically relevant fragments:

| Theory | Decision Complexity | Algorithm | Compiler Use |
|--------|--------------------|-----------|-------------|
| Propositional logic (SAT) | NP-complete | DPLL / CDCL | Bitblasting BV, SAT sweeping |
| EUF (equality + uninterpreted functions) | PTIME | Congruence closure | Peephole equalities, CSE |
| QF_LRA (linear real arithmetic) | PTIME (LP) | Fourier-Motzkin, simplex | Polyhedral bounds, range analysis |
| QF_LIA (linear integer arithmetic) | NP (Omega test) | Omega, Presburger | Loop bounds, array subscripts, Polly |
| QF_BV (bit-vectors) | NP (bitblast to SAT) | Bitblast + CDCL | Alive2, overflow checks |
| QF_UF + LIA (UFLÍA) | NP (via reduction) | DPLL(T) | Dafny VCs, Verus |
| QF_NRA (nonlinear reals) | Decidable (Tarski) | CAD (doubly exp.) | Floating-point analysis |
| FOL over integers (PA) | Undecidable | None | Not used directly |
| Nonlinear integer arithmetic | Undecidable | None | Hilbert's 10th problem |

The boundary of undecidability on the integer side is sharp: quantifier-free linear integer arithmetic (Presburger) is decidable; adding multiplication of two variables immediately reaches Hilbert's 10th problem, which Matiyasevich proved undecidable in 1970 by showing every recursively enumerable set is Diophantine.

**Presburger arithmetic** is the first-order theory of `(ℤ, +, <, 0, 1)`. The Omega test [Pugh 1992] is the standard practical algorithm; it underlies Polly's dependence analysis (cross-referenced in [Chapter 70 — Foundations: Polyhedra and Integer Programming](../part-11-polyhedral-theory/ch70-foundations-polyhedra-and-integer-programming.md)).

**DPLL (Davis-Putnam-Logemann-Loveland):** Propositional satisfiability via unit propagation + backtracking:
1. Unit propagation: if a clause has a single unassigned literal, assign it true.
2. Pure literal elimination: if a variable appears with only one polarity, set it accordingly.
3. Decision: pick an unassigned variable, try true/false.
4. Backtrack on conflict.

CDCL (Conflict-Driven Clause Learning) extends DPLL with clause learning from conflicts, dramatically improving performance on industrial instances. When CDCL reaches a conflict (an assignment that falsifies a clause), it performs *conflict analysis* by tracing the implication graph: which decisions and propagations led to the conflict? The analysis produces a *conflict clause* (a new clause implied by the formula but not already present) that prunes the search space globally. CDCL solvers (Glucose, MiniSAT, CaDiCaL, Kissat) are the reason SAT solving became industrial-scale in the 2000s — clause learning gives the solver a form of memory that prevents it from re-exploring the same contradictory subspaces.

The decision procedure for **Fourier-Motzkin elimination** (QF_LRA) eliminates a variable `x` from a conjunction of linear inequalities by: (a) collecting upper bounds `x ≤ u_i` and lower bounds `l_j ≤ x`, then (b) replacing them with all pairwise `l_j ≤ u_i`. Repeat until no variables remain; the result is satisfiable iff the final system has no contradiction. For `n` variables this is doubly exponential in the worst case, but for the small systems generated by compiler VCs (typically 5-20 variables), it is fast in practice.

### The DPLL(T) Architecture

Modern SMT solvers are built on the DPLL(T) architecture: a CDCL propositional solver orchestrates one or more *theory solvers* (T-solvers) for decidable fragments. The interface between the propositional layer and theory layers is the key abstraction:

```
┌──────────────────────────────────────────────────────────────┐
│                    DPLL(T) Core                              │
│                                                              │
│  ┌─────────────────────┐       theory literals              │
│  │  CDCL Propositional │  ──────────────────────►  T-Solver │
│  │      Solver         │  ◄──────────────────────  (LIA,BV, │
│  │  (Boolean skeleton) │       T-lemmas /           EUF...) │
│  │  + watched literals │       T-conflicts                  │
│  └─────────────────────┘                                     │
│         ↑                                                    │
│   boolean assignment                                         │
└──────────────────────────────────────────────────────────────┘
```

The propositional layer treats each theory atom (e.g., `x + y ≤ 5`, `f(a) = b`) as a Boolean variable. When the CDCL solver assigns a consistent Boolean assignment, it passes the corresponding theory literals to the T-solver. The T-solver either:
- Returns `sat` (consistent with theory), and propositional solving continues.
- Returns `unsat` with a *conflict clause* (the minimal subset of assigned literals that is theory-inconsistent), which the CDCL solver learns and backtracks on.

This architecture, formalized by Nieuwenhuis, Oliveras, and Tinelli [JACM 2006], separates the search (propositional) from the reasoning (theory), allowing each T-solver to be specialized for its fragment.

### Nelson-Oppen Theory Combination

Many queries involve multiple theories simultaneously: a formula may mix linear arithmetic with uninterpreted functions and arrays. The *Nelson-Oppen combination procedure* [Nelson-Oppen, TOPLAS 1979] handles combinations of theories under two conditions:

1. **Disjoint signatures**: the non-logical symbols of `T₁` and `T₂` do not overlap (except for `=`).
2. **Stably infinite theories**: every satisfiable quantifier-free formula has a model with an infinite domain (ensuring equality reasoning over shared variables is sound).

The procedure:
1. Purify the formula: split into a `T₁`-part and a `T₂`-part sharing only variables.
2. **Equality sharing**: each theory generates all equalities between shared variables that it can derive. Pass these equalities to the other theory.
3. Iterate until a fixed point. The combined formula is unsatisfiable iff one theory reaches `⊥`.

For example, combining `T_LIA` (linear integer arithmetic) and `T_UF` (equality with uninterpreted functions):
- `T_LIA` may derive `x = y` from `x - y = 0`.
- Pass `x = y` to `T_UF`, which may then derive `f(x) = f(y)` by congruence.

The Nelson-Oppen procedure can be run in *deterministic* mode (where one theory is designated to enumerate equalities) or *non-deterministic* mode (where theories may case-split). The non-deterministic version handles theories like the theory of arrays (which may need to case-split on equality of array indices). Efficiency is a concern: a naive implementation runs in quadratic time (all pairs of shared terms), but sophisticated implementations using union-find data structures run in near-linear time.

**Practical limits of Nelson-Oppen:** The disjoint-signatures condition is a real constraint. Arithmetic combined with arrays requires `T_Arrays` and `T_LIA` to share index variables — they must cooperate on equalities like `i = j`. Floating-point arithmetic (`T_FP`) is handled separately from integer arithmetic; mixing them requires either an explicit cast or a hybrid theory. Z3's handling of these combinations is engineered case by case, with Nelson-Oppen as the underlying theoretical guarantee.

### Congruence Closure: The EUF Decision Procedure

The theory of **equality with uninterpreted functions** (EUF) is decided by *congruence closure* — a classic algorithm dating to Downey, Sethi, and Tarjan (1980) and refined by Shostak (1984), Nelson-Oppen (1980), and Nieuwenhuis-Oliveras (2005).

Given a set of ground equalities and disequalities involving function terms:
```
E = {t₁ = t₂, t₃ = t₄, ...,  u₁ ≠ v₁, ...}
```

The congruence closure algorithm builds equivalence classes of terms using a union-find structure:
1. Initialize: each term is its own class.
2. For each equality `t₁ = t₂`: union the classes of `t₁` and `t₂`.
3. Propagate congruences: if `f(a₁,...,aₙ)` and `f(b₁,...,bₙ)` are two terms and `aᵢ ≡ bᵢ` for all `i` (same class), then union `f(a₁,...,aₙ)` and `f(b₁,...,bₙ)`.
4. Repeat until no new merges occur (fixed point).
5. Check: is any disequality `u ≠ v` falsified (u and v in the same class)?

The result is `unsat` iff some disequality is falsified. Otherwise, the equivalence classes form a *model* of all the equalities (interpret each function symbol as mapping tuples of representatives to representatives).

This algorithm has near-linear time complexity using inverse-Ackermann union-find (O(n α(n)) where α is the inverse Ackermann function, effectively constant). In the context of DPLL(T), the congruence closure T-solver is called once per CDCL assignment; the incremental interface (undo merges on backtrack) makes it efficient in practice.

**Application to compiler optimization:** Common subexpression elimination (CSE) in LLVM corresponds exactly to congruence closure on the IR. Two instructions computing `add i32 %x, %y` at different program points produce the same value iff `%x` and `%y` are equal. LLVM's GVN (Global Value Numbering) pass computes congruence classes of IR values — it is, algorithmically, a congruence closure over the SSA value graph. The EUF decision procedure is the proof-theoretic justification that this optimization is semantics-preserving.

### Z3's Theory Stack

Z3 [de Moura and Bjørner, TACAS 2008; [Z3 source](https://github.com/Z3Prover/z3)] implements DPLL(T) with the following theory stack:

| Theory | Name | Core Algorithm |
|--------|------|---------------|
| Linear integer arithmetic | T_LIA | Simplex + branch-and-bound |
| Nonlinear integer arithmetic | T_NIA | Gröbner bases + CAD |
| Linear real arithmetic | T_LRA | Dual simplex |
| Bit-vectors | T_BV | Bitblast to SAT |
| Uninterpreted functions | T_UF | Congruence closure (Shostak) |
| Arrays | T_Arrays | McCarthy axioms + Ackermann reduction |
| Floating-point | T_FP | IEEE 754 via bitblasting |
| Regular expressions | T_RE | Automata intersection |

Alive2's verification queries (see [Chapter 170 — Alive2 and Translation Validation](../part-24-verified-compilation/ch170-alive2-and-translation-validation.md)) predominantly use QF_BV. Dafny's verification conditions (see [Chapter 181 — Formal Verification in Practice](../part-26-ecosystem-frontiers/ch181-formal-verification-in-practice.md)) primarily use QF_LIA and QF_UF, with quantifiers added for heap reasoning, which Z3 handles via pattern-based instantiation (E-matching).

### SMT-LIB 2 Concrete Examples

SMT-LIB 2 is the standardized input format for SMT solvers ([smtlib.cs.uiowa.edu](https://smtlib.cs.uiowa.edu/)). The following examples illustrate the three main use patterns for compiler verification.

**Example 1: Linear Integer Arithmetic (LIA) — Loop Bound Analysis**

A compiler might generate this query to verify a loop bound is non-negative after strength reduction:

```smt2
; Verify: if n >= 1 and i = 0 and step = 2, then n - i > 0
(set-logic QF_LIA)
(declare-const n Int)
(declare-const i Int)
(declare-const step Int)
(assert (>= n 1))
(assert (= i 0))
(assert (= step 2))
; Check satisfiability of the negation (n - i <= 0)
; to verify n - i > 0 holds
(assert (<= (- n i) 0))
(check-sat)
; Expected: unsat (meaning n - i > 0 always holds under constraints)
(get-model)
```

The solver returns `unsat`, confirming the loop bound check. Polly generates queries of this form for dependence analysis using Presburger constraints.

**Example 2: Bit-Vector Arithmetic — Overflow Check (Alive2 Style)**

Alive2 verifies peephole rewrites by encoding "does the source have a defined result that the target fails to compute (or vice versa)?" as a QF_BV query. Here: verifying `x * 2` → `x << 1` preserves semantics for 32-bit no-wrap signed arithmetic:

```smt2
; Verify: x * 2 == x << 1 (32-bit signed, assuming no overflow)
(set-logic QF_BV)
(declare-const x (_ BitVec 32))
; nsw condition: x * 2 does not signed-overflow
; i.e., (x >= 0 → x*2 >= 0) and (x < 0 → x*2 < 0)
; Equivalently: x's sign bit equals (x<<1)'s sign bit
(define-fun lhs () (_ BitVec 32)
  (bvmul x (_ bv2 32)))
(define-fun rhs () (_ BitVec 32)
  (bvshl x (_ bv1 32)))
; Assert they differ (look for a counterexample)
(assert (not (= lhs rhs)))
(check-sat)
; Expected: unsat — they are always equal at the bit level
```

The solver returns `unsat`, confirming the rewrite is universally sound at the bit level. A more realistic Alive2 query would additionally constrain on `nsw` flags using the sign-extension overflow test:

```smt2
(set-logic QF_BV)
(declare-const x (_ BitVec 32))
; nsw overflow check: signedOverflow(x * 2) iff msb changes
(define-fun no_signed_overflow ((v (_ BitVec 32))) Bool
  (= ((_ extract 31 31) v)
     ((_ extract 31 31) (bvmul v (_ bv2 32)))))
(assert (no_signed_overflow x))
; Under no-overflow, x*2 = x<<1
(assert (not (= (bvmul x (_ bv2 32))
               (bvshl x (_ bv1 32)))))
(check-sat)
; Expected: unsat
```

**Example 3: Equality with Uninterpreted Functions (EUF) — Function Equivalence**

Compilers use EUF to reason about common subexpression elimination and function inlining without knowing the concrete implementation of called functions:

```smt2
; Show that CSE is valid: if f(a) = f(b) follows from a = b,
; then replacing a second call f(a) with the cached value f(b) is sound
; (given a = b holds at the call site)
(set-logic QF_UF)
(declare-sort Val 0)
(declare-fun f (Val) Val)
(declare-const a Val)
(declare-const b Val)
; Hypothesis: a = b
(assert (= a b))
; Query: does f(a) = f(b) hold? (it must, by congruence)
; Try to find a model where f(a) ≠ f(b) despite a = b
(assert (not (= (f a) (f b))))
(check-sat)
; Expected: unsat — congruence closure derives f(a) = f(b) from a = b
```

The `unsat` result expresses the *congruence axiom*: functions respect equality of arguments. Congruence closure algorithms [Downey-Sethi-Tarjan 1980; Nieuwenhuis-Oliveras 2005] compute this in near-linear time, making EUF reasoning efficient in practice.

---

## 185.5 Model Theory

### Structures and Their Theories

For a structure `M`, its *theory* is the set of all sentences it satisfies:

```
Th(M) = { φ : M ⊨ φ,  φ a sentence of L }
```

`Th(ℕ)` — the complete theory of the standard natural numbers — is undecidable (by Gödel): no recursive axiom system can enumerate all of its theorems. `Th(ℝ, +, ×, <)` — the theory of real closed fields — is decidable (by Tarski's quantifier elimination). This contrast epitomizes the decidability landscape.

A theory `T` is *complete* if for every sentence `φ`, either `T ⊢ φ` or `T ⊢ ¬φ`. Completeness has both a proof-theoretic meaning (syntactic derivability settles every question) and a model-theoretic one: `T` is complete iff it is *categorical in some power* — all its models of a fixed cardinality are isomorphic (by Vaught's test). For example, the theory of dense linear orders without endpoints (DLO) is ω-categorical (all countable models are isomorphic to `(ℚ, <)`), hence complete.

Examples of complete theories relevant to the compiler engineer:
- **DLO**: `(ℚ, <)` — complete, decidable, QE. Models order-comparisons in loop bounds.
- **RCF**: `(ℝ, +, ×, <, 0, 1)` — complete, decidable, QE. Models floating-point range analysis over the reals.
- **ACF₀**: `(ℂ, +, ×, 0, 1)` — complete, decidable. Models polynomial algebra; Hilbert's Nullstellensatz is a theorem of ACF₀.
- **Presburger**: `(ℤ, +, <, 0, 1)` — complete, decidable, QE. Models loop subscripts, array bounds.

By contrast, **PA** (Peano Arithmetic) is *not* complete: `G_PA` is neither provable nor disprovable. And **RCF** is complete while **PA** is not — the difference is that multiplication of two variables in `ℤ` breaks decidability, while multiplication in `ℝ` (combined with the order and the intermediate value property) remains decidable via Tarski's algebraic technique.

### Elementary Equivalence and Embeddings

Two structures `M` and `N` are **elementarily equivalent** (written `M ≡ N`) if they satisfy exactly the same first-order sentences: `Th(M) = Th(N)`. Elementary equivalence is strictly weaker than isomorphism (`M ≅ N`): the non-standard models of PA are elementarily equivalent to `ℕ` but not isomorphic to it.

The tools for proving elementary equivalence without finding an explicit isomorphism are **Ehrenfeucht-Fraïssé games** (1954/1965). Two players — Duplicator and Spoiler — play a game of `n` rounds on structures `M` and `N`:
- Each round: Spoiler picks an element from one structure; Duplicator must pick a corresponding element from the other.
- After `n` rounds, let `a₁,...,aₙ ∈ M` and `b₁,...,bₙ ∈ N` be the chosen elements.
- Duplicator wins if the map `aᵢ ↦ bᵢ` is a partial isomorphism (preserves all relations and function symbols in the language).

**Theorem:** `M ≡ₙ N` (M and N satisfy the same sentences of quantifier rank ≤ n) iff Duplicator has a winning strategy for the n-round game on M and N. In particular, `M ≡ N` iff Duplicator wins all finite games.

This game-theoretic characterization is extremely useful: to show two structures are elementarily equivalent, exhibit a winning strategy. To show they differ, exhibit a sentence of appropriate quantifier depth that one satisfies and the other does not.

**Example relevant to compiler analysis:** The structure `(ℤ, +, <)` (integers with addition and order) is elementarily equivalent to `(ℚ, +, <)` (rationals). A winning strategy for Duplicator exists: for any finite set of integers the game may have selected, there is a corresponding finite set of rationals with the same order type. The Ehrenfeucht-Fraïssé game makes this precise and allows the conclusion without constructing an isomorphism (the two structures are not isomorphic, since ℤ is discrete and ℚ is dense).

An **elementary embedding** `f : M → N` is an injective map preserving all first-order satisfaction:

```
M, α ⊨ φ  ↔  N, f∘α ⊨ φ   for all formulas φ and all assignments α
```

An **elementary substructure** `M ≼ N` means `M` is a substructure and the inclusion map is an elementary embedding. The *Tarski-Vaught test* characterizes this: `M ≼ N` iff for every formula `φ(x, ȳ)` and every `b̄ ∈ M`, if `N ⊨ ∃x. φ(x, b̄)` then there exists `a ∈ M` with `N ⊨ φ(a, b̄)`.

The downward Löwenheim-Skolem theorem constructs elementary substructures of cardinality ≤ |L| + ℵ₀ using the Tarski-Vaught test: start with any countable subset of N's domain; close under the Skolem functions (witness-choosing functions for each existential formula); the result is an elementary substructure.

**Model-theoretic relevance to SMT:** Theory solvers for linear arithmetic maintain an implicit elementary extension of the "standard" model of integers/reals — the solver state represents an element of the theory's model. The soundness of the theory solver is exactly the statement that its conclusion is a valid sentence of `Th(ℤ, +, <)` or `Th(ℝ, +, ×, <)`. When Z3 reports `unsat` for a QF_LIA query, it is reporting that the formula has no model in *any* structure satisfying the LIA axioms — not just the standard integers. By completeness of the theory (QE + decidability), this is equivalent to no standard-integer model existing.

### Model-Theoretic Proof of Compactness via Ultraproducts

The compactness theorem has an elegant model-theoretic proof using ultraproducts that avoids syntactic arguments. An *ultrafilter* `U` on an index set `I` is a maximal filter: a collection of subsets of `I` closed under supersets and finite intersections, containing exactly one of `A` or `I\A` for every `A ⊆ I`. Non-principal ultrafilters (which exist by Zorn's lemma) contain no finite sets.

Let `{M_i}_{i ∈ I}` be a family of structures and `U` a non-principal ultrafilter on `I`. The **ultraproduct** `M = ∏_U M_i` has domain `∏_i D_i / ~_U`, where two sequences `(a_i)` and `(b_i)` are equivalent iff `{i : a_i = b_i} ∈ U`. Functions and relations are lifted coordinate-wise: `f^M([(a_i)]) = [(f^{M_i}(a_i))]`, and `P^M([(a_i)]) ↔ {i : P^{M_i}(a_i)} ∈ U`.

**Łoś's Theorem (Fundamental Theorem of Ultraproducts):** For any first-order sentence `φ`:
```
∏_U M_i ⊨ φ   ↔   {i : M_i ⊨ φ} ∈ U
```
The proof is by induction on the complexity of `φ`. For atomic formulas, the equivalence holds by definition. For Boolean connectives, the ultrafilter's maximality handles `¬` (complement in U) and `∧` (intersection in U). For `∃x.φ(x)`: if `{i : M_i ⊨ ∃x.φ(x)} ∈ U`, then for each such `i` pick a witness `a_i`; the element `[(a_i)]` witnesses `∃x.φ(x)` in the ultraproduct.

**Compactness from ultraproducts:** Given `Σ` such that every finite subset `F ⊆ Σ` is satisfiable, let `I` = the collection of all finite subsets of `Σ`. For each `σ ∈ Σ`, let `A_σ = {F ∈ I : σ ∈ F}`. The family `{A_σ}` has the finite intersection property (`A_{σ₁} ∩ ... ∩ A_{σₙ} ⊇ A_{{σ₁,...,σₙ}} ≠ ∅`), so it extends to an ultrafilter `U` on `I`. For each `F ∈ I`, pick a model `M_F ⊨ F` (possible by hypothesis). The ultraproduct `∏_U M_F` satisfies every `σ ∈ Σ` by Łoś's theorem, since `{F : M_F ⊨ σ} ⊇ A_σ ∈ U`.

Ultraproducts are the model-theoretic analog of limit constructions — they encode a "typical" behavior at infinity. Non-standard analysis (Robinson 1966) exploits exactly this: the ultraproduct of countably many copies of `ℝ` indexed by `ℕ` yields a non-Archimedean field *R containing infinitesimals `ε` with `0 < ε < 1/n` for all standard `n ∈ ℕ`. The transfer principle (a consequence of Łoś) says that *R satisfies exactly the same first-order sentences as `ℝ` — so all standard real analysis transfers to non-standard analysis, making Leibniz's infinitesimals rigorous.

### Quantifier Elimination

A theory `T` admits **quantifier elimination** (QE) if for every formula `φ(x₁,...,xₙ)` there is a quantifier-free formula `ψ(x₁,...,xₙ)` such that `T ⊢ ∀x̄. (φ ↔ ψ)`. QE implies decidability (for theories with decidable quantifier-free fragment) and gives a model-theoretic characterization of definable sets: the definable subsets of Dⁿ are exactly the sets defined by quantifier-free formulas, which are finite Boolean combinations of atomic formulas.

**The method for DLO (dense linear orders without endpoints):** To eliminate `∃x. φ(x)` where `φ` is a conjunction of literals involving `x`:
- Collect lower bounds: `l₁ < x, ..., lₖ < x`
- Collect upper bounds: `x < u₁, ..., x < uₘ`
- Replace `∃x. (l₁ < x ∧ ... ∧ x < u₁ ∧ ...)` with `l₁ < u₁ ∧ l₁ < u₂ ∧ ... ∧ lₖ < uₘ` (all pairs of lower/upper bounds are consistent). This procedure terminates because each step reduces quantifier rank.

**The Omega test for Presburger arithmetic** [Pugh 1992] is the decision procedure derived from QE for `(ℤ, +, <, 0, 1)`. Given `∃x. φ(x)` where `φ` is a conjunction of linear constraints over integers:
1. Collect lower bounds `aᵢ ≤ x` and upper bounds `x ≤ bⱼ`.
2. If all `aᵢ ≤ bⱼ` are satisfiable over ℤ (accounting for the discrete gap), the formula is satisfiable.
3. The *dark shadow* condition: `aᵢ ≤ bⱼ - (coefficient-1)` for all pairs. This is a sufficient condition.
4. The *gray shadow* handles borderline cases with divisibility constraints.

The Omega test is the backbone of Polly's dependence analysis and of ISCC (the Integer Set Calculator). The model-theoretic fact that `(ℤ, +, <, 0, 1)` admits QE is exactly why the Omega test terminates and is complete.

**Real closed fields (RCF):** Tarski (1951) proved QE for `(ℝ, +, ×, <, 0, 1)` via the method of *cylindrical algebraic decomposition* (CAD). The key step for `∃x. p₁(x) ≥ 0 ∧ ... ∧ pₙ(x) ≥ 0` (polynomial constraints) is: compute the roots of all pᵢ's and their discriminants, then reason about sign assignments between consecutive roots. Because the number of real roots is bounded by the degree, this process terminates.

QE for RCF implies:
- **Decidability of Euclidean geometry** (Tarski 1951): every first-order statement about points, lines, and distances in the real plane is decidable.
- **Decidability of polynomial optimization over ℝ**: the question "does `∃x₁,...,xₙ. p(x₁,...,xₙ) ≤ c`?" for polynomials `p` and rational `c` is decidable, relevant to compiler analyses that bound polynomial expressions.

**Algebraically closed fields (ACF):** The theory of `(ℂ, +, ×, 0, 1)` admits QE (Chevalley's theorem: the image of a polynomial map is constructible). QE for ACF means the definable sets are exactly the constructible sets (finite Boolean combinations of algebraic varieties). This is the model-theoretic expression of Hilbert's Nullstellensatz, discussed next.

### Algebraically Closed Fields and Hilbert's Nullstellensatz

The model theory of **algebraically closed fields** (ACF) is particularly elegant. The theory `ACF` has two complete extensions: `ACF₀` (characteristic 0, theory of `ℂ`) and `ACF_p` for each prime `p`.

**Nullstellensatz (Hilbert):** For polynomials `f₁,...,fₘ, g ∈ k[x₁,...,xₙ]` over an algebraically closed field `k`, the following are equivalent:
- `g` vanishes on every common zero of `f₁,...,fₘ`
- `gᴺ ∈ ⟨f₁,...,fₘ⟩` for some `N ≥ 1`

The model-theoretic proof uses the *transfer principle* (a consequence of QE for ACF): a statement holds in one algebraically closed field iff it holds in all algebraically closed fields of the same characteristic. This connects the geometry of zeros (varieties) to ideal membership in polynomial rings — a connection developed further in [Chapter 187 — Commutative Algebra for Compilation](./ch187-commutative-algebra-for-compilation.md) in the context of Gröbner bases and toric varieties.

---

## 185.6 Higher-Order Logic vs. First-Order Logic

### The Expressivity Gap

Second-order logic allows quantification over predicates and functions, not just individuals. This radically increases expressiveness: the second-order sentence

```
∀P. [P(0) ∧ ∀n. P(n) → P(S(n))] → ∀n. P(n)
```

*categorically* characterizes the natural numbers — it pins down `ℕ` up to isomorphism. No set of first-order sentences can do this (by Löwenheim-Skolem: there are non-standard models). The first-order Peano axioms have non-standard models with "infinite" elements satisfying all the first-order axioms of PA; the second-order induction axiom rules them out.

Similarly, second-order Peano Arithmetic (PA₂) is *semantically complete* for standard arithmetic: every true sentence of arithmetic follows semantically from PA₂. But this completeness is not effective: there is no sound and complete *recursive* proof system for second-order logic. The set of second-order validities is not recursively enumerable (it is Π₁₁-complete — far beyond the arithmetic hierarchy).

**Higher-order logic (HOL)** generalizes beyond second order: functions and predicates can take functions as arguments, and types are stratified by order. Church's Simple Theory of Types (STT, 1940) provides the standard formulation: types are built from a base type `o` (booleans) and `ι` (individuals) using the function-type constructor `→`. Functions from type `α` to type `β` have type `(α → β)`. Quantifiers range over each type.

HOL is undecidable and has no complete proof system (in the sense of a recursive set of axioms that derive all valid sentences). This follows from the fact that full second-order logic — a subset of HOL — already lacks complete proof procedures. Concretely: the satisfiability problem for HOL is not recursively enumerable; no algorithm can list all HOL satisfiable formulas.

**Why this is not an obstacle for proof assistants:** The insight of the LCF paradigm (Edinburgh, 1972) is that undecidability of validity does not prevent *verification*. The proof assistant does not need to find proofs automatically; it only needs to *check* that a proof presented by the user is correct. Checking a single proof step (applying one inference rule) is decidable. The undecidability of validity means: there is no algorithm that, given a formula, decides whether it is provable. But given a formula *and* a proof, checking the proof takes polynomial time.

The practical workflow in Isabelle/HOL: the user writes an Isar proof script; the system checks each step. For harder steps, `sledgehammer` calls an external ATP (which works on a first-order approximation of the goal), and the ATP's proof is reconstructed and verified within Isabelle's HOL kernel. The ATP's completeness (for FOL) is used locally, within a HOL system that is globally undecidable. This is modular logic-engineering at its finest.

### Why Isabelle Uses HOL

Despite undecidability, HOL is the logic of choice for Isabelle/HOL because:

1. **Expressiveness**: mathematical concepts — functions, sets, sequences, inductive definitions — all fit naturally in HOL without the encoding overhead of FOL.
2. **The LCF architecture**: in the LCF approach (Milner, Edinburgh, 1972), the logic is implemented such that the only way to create a `Thm.t` (a theorem object) is to apply inference rules. Undecidability does not matter: the user provides proofs; the system just checks them.
3. **Automation**: `Sledgehammer` calls external FOL ATPs (Z3, CVC5, Vampire, E) on the FOL fragment of the goal and reconstructs Isabelle proofs from the ATP output. FOL completeness is used *modularly* within a HOL system.

By contrast, Coq and Lean 4 use *dependent type theory* (the Calculus of Inductive Constructions), which is a higher-order proof-relevant type theory — not FOL or HOL per se. See [Chapter 184 — Proof Assistant Internals: Lean 4, Coq/Rocq, and Isabelle/HOL](./ch184-proof-assistant-internals.md) for the architectural details.

### Why Isabelle Uses HOL and Coq/Lean Use Dependent Types

The choice between HOL and dependent type theory is a fundamental architectural decision:

- **Isabelle/HOL**: uses HOL (Church's STT), which is a *classical* logic with a relatively shallow type hierarchy. Classical reasoning (proof by contradiction, law of excluded middle) is built in. The meta-logic Pure is an intuitionistic higher-order logic, but HOL itself is classical. This makes it easy to import classical mathematics (e.g., Zorn's lemma, Axiom of Choice, non-constructive existence proofs) without philosophical overhead. The price: the logic is not proof-relevant; proofs are not first-class mathematical objects. This matters for applications like homotopy type theory.

- **Coq/Rocq and Lean 4**: use *dependent type theory* (the Calculus of Inductive Constructions, CIC). Types can depend on terms, enabling propositions-as-types with full proof-relevance. The type `a = b` is inhabited iff `a` and `b` are definitionally equal; different inhabitants represent different proofs of the same equality. This richer structure enables the HoTT program and makes the type theory itself a foundation for mathematics. The price: classical logic requires an explicit axiom (`Classical.em` in Lean 4, `Axiom classical : forall P : Prop, P \/ ~P` in Coq), and proof irrelevance is a separate axiom. See [Chapter 184 — Proof Assistant Internals: Lean 4, Coq/Rocq, and Isabelle/HOL](./ch184-proof-assistant-internals.md).

For **compiler verification**: CompCert uses Coq (constructive proofs enable computational extraction to a running compiler); seL4 uses Isabelle/HOL (classical reasoning simplifies the OS kernel proof, which requires extensive case analyses); Alive2 uses neither — it uses Z3 directly (SMT reasoning about bitvectors does not require a proof assistant).

### The HOL Family

The HOL family derives from Mike Gordon's HOL88 (1988), which formalized HOL as a sequent calculus implemented in ML using the LCF kernel idea. Gordon's system was itself inspired by Robin Milner's LCF system (Edinburgh, 1972), which introduced the idea of implementing a logic in a strongly-typed programming language (ML) so that the type system enforces proof correctness.

| System | Kernel | Proof Language | Key Use | Lines of Trust |
|--------|--------|----------------|---------|----------------|
| HOL4 | SML; ~3k LOC | `hol_tactics` | Hardware verification, CakeML | ~3,000 |
| HOL Light | OCaml; ~400 LOC | ML functions | Flyspeck (Kepler conjecture proof) | ~400 |
| Isabelle/HOL | SML/ML; Pure metalogic | Isar structured proofs | seL4, CryptoProofs, CompCert ports | ~5,000 |

**HOL Light kernel** (Harrison 1996, [source](https://github.com/jrh13/hol-light)): The entire trusted base is approximately 400 lines of OCaml implementing exactly 10 primitive inference rules and 2 definition mechanisms. The 10 rules are:

| Rule | Type | Description |
|------|------|-------------|
| `REFL` | `term → thm` | `⊢ t = t` |
| `TRANS` | `thm → thm → thm` | `⊢ s=t`, `⊢ t=u` → `⊢ s=u` |
| `MK_COMB` | `thm → thm → thm` | `⊢ f=g`, `⊢ x=y` → `⊢ f x = g y` |
| `ABS` | `term → thm → thm` | `⊢ t=u` → `⊢ (λx.t) = (λx.u)` |
| `BETA` | `term → thm` | `⊢ (λx.t) x = t` |
| `ASSUME` | `term → thm` | `A ⊢ A` |
| `EQ_MP` | `thm → thm → thm` | `⊢ p=q`, `⊢ p` → `⊢ q` |
| `DEDUCT_ANTISYM` | `thm → thm → thm` | `A ⊢ p`, `B ⊢ q` → `A−{q} ∪ B−{p} ⊢ p=q` |
| `INST` | `(term × term) list → thm → thm` | Instantiate term variables |
| `INST_TYPE` | `(type × type) list → thm → thm` | Instantiate type variables |

The 2 definition mechanisms are: (1) constant definition `c = t` (where `c` is fresh and `t` is a closed term), and (2) type definition by a bijection to a non-empty subset of an existing type.

Every HOL Light theorem is a `Sequent.thm` OCaml abstract type, constructible *only* through these rules and definitions. This is the LCF paradigm: the ML type system enforces proof correctness. Any theorem `⊢ φ` in scope was provably derived from axioms using only these rules; there is no back-door. John Harrison used HOL Light to formally verify a large fragment of the proof of the Kepler conjecture (the Flyspeck project, 2014 — 300 person-years of informal proof formalized in HOL Light and Isabelle), demonstrating that even large-scale mathematical verification is feasible with this minimal kernel.

---

## 185.7 Connections Across the Compiler Ecosystem

### Hoare Logic as a Proof System for Programs

Hoare logic (Floyd 1967; Hoare 1969) can be understood as a proof system layered on top of a programming language. The judgment `{P} C {Q}` (P is the precondition, C the command, Q the postcondition) is valid iff every execution of C that starts in a state satisfying P terminates in a state satisfying Q.

The rules — assignment axiom, sequential composition, conditional, while loop with invariant — form an inference system that is sound (valid triples are provable relative to the program's axiomatic semantics) and relatively complete (given an oracle for the underlying theory) by Cook's theorem (1978).

Separation logic [Reynolds 2002; O'Hearn-Reynolds-Yang 2001] extends Hoare logic with a spatial conjunction `P * Q`: the heap can be split into two *disjoint* parts satisfying `P` and `Q` respectively. A *heap* is a partial function `h : Addr ⇀ Val` from addresses to values. The separating conjunction:

```
h ⊨ P * Q   iff   ∃ h₁, h₂.  h = h₁ ⊎ h₂  ∧  h₁ ⊨ P  ∧  h₂ ⊨ Q
```

where `⊎` is the disjoint union of partial functions (defined only when the domains are disjoint). The *emp* assertion means the heap is empty: `h ⊨ emp ↔ h = ∅`.

The *frame rule*:

```
  {P} C {Q}    (C does not modify free(R))
───────────────────────────────────────────  [Frame]
        {P * R} C {Q * R}
```

licenses modular reasoning: a proof about C in a small heap extends to any larger heap, provided C does not touch the extension `R`. This is the model-theoretic soundness condition: the satisfaction of `P * R` by a heap `h = h₁ ⊎ h₂` (with `h₁ ⊨ P`, `h₂ ⊨ R`) decomposes into independent subproblems.

The model-theoretic structure underlying separation logic is a *commutative partial monoid* `(Heap, ⊎, emp)`. The separating conjunction `*` is the monoidal product. The logic of separation is the logic of *heap resources* as monoidal elements — a logic of ownership and sharing. Concurrent separation logic [O'Hearn 2007] extends this with *permissions* and *thread-local* heaps, enabling proofs of concurrent programs. Iris [Jung et al. 2016] is the modern framework that unifies concurrent separation logic with higher-order logic and step-indexing, providing a complete proof system for concurrent higher-order programs verified in Coq.

For compiler verification, separation logic is the specification language for memory safety of programs targeting LLVM IR: a correct compilation of a separation-logic-verified source program must preserve the *ownership* structure in the generated IR. Rustc's borrow checker enforces exactly this ownership discipline statically, making Rust programs separation-logic-correct by construction. See [Chapter 167 — Operational Semantics and Program Logics](../part-24-verified-compilation/ch167-operational-semantics-and-program-logics.md) for the full formal treatment.

### Abstract Interpretation as a Model-Theoretic Structure

Abstract interpretation [Cousot-Cousot 1977] can be viewed through the model-theoretic lens: an abstract domain is a *structure* that models the program's concrete semantics approximately. The Galois connection `(α, γ)` between concrete and abstract lattices is exactly a model-theoretic *approximation morphism*: the abstract domain is a structure homomorphically related to the concrete domain.

Concretely: the concrete domain for a variable `x : int32` is `℘(ℤ)` — sets of possible values, ordered by subset. An abstract domain like the interval domain replaces each element of `℘(ℤ)` with an interval `[l, u]` or `⊤` or `⊥`. The abstraction map `α : ℘(ℤ) → Intervals` sends a set `S` to the smallest interval containing it; the concretization `γ : Intervals → ℘(ℤ)` sends `[l, u]` to `{n ∈ ℤ : l ≤ n ≤ u}`. The Galois connection condition `α(S) ⊑ I ↔ S ⊆ γ(I)` is precisely a *model-theoretic adjunction*: `α ⊣ γ`.

The widening operator `∇` that forces convergence of the abstract fixpoint iteration corresponds, in model-theoretic terms, to a coarsening of the approximation relation — trading precision for decidability. The concrete fixed point `⊔_{n≥0} fⁿ(⊥)` (the *least Herbrand model* of the logic program encoding the analysis, or the *collecting semantics*) is approximated by the abstract fixed point.

This model-theoretic framing has a direct logical interpretation. The collecting semantics is the *canonical model* of the analysis viewed as a Horn clause program: every reachable program state is "true" in this model, and the fixpoint computation enumerates it. The abstract interpretation computes an *overapproximation* of this model in the abstract structure. Termination is guaranteed not by the logic itself (the full collecting semantics is undecidable for most properties by Rice's theorem) but by the Noetherian property of the abstract lattice under `∇`.

This perspective connects [Chapter 10 — Dataflow Analysis and the Lattice Framework](../part-02-compiler-theory/ch10-dataflow-analysis-lattice-framework.md) to the model theory of temporal logic: LLVM's IR is a transition system, a dataflow analysis is a safety property, and abstract interpretation is approximate model checking. The concrete lattice is a first-order structure; the abstract lattice is an approximate homomorphic image. Full model checking of LTL over finite-state systems is decidable (PSPACE-complete); abstract interpretation approximates this to make it tractable for infinite-state programs. The LLVM-specific dataflow analyses built on this foundation are catalogued in [Chapter 61 — Foundational Analyses](../part-10-analysis-middle-end/ch61-foundational-analyses.md).

### Z3 Enabling Alive2, Dafny, and Verus

The three most important compiler-adjacent verification tools all reduce to Z3 queries:

- **Alive2**: for each peephole rewrite, encodes the source and target in QF_BV and checks `source-semantics ⊨ target-semantics` (UB propagation included). A counterexample from Z3 is a concrete bitvector assignment that witnesses the unsoundness.
- **Dafny**: the verification condition generator emits QF_LIA + EUF (with limited quantifiers for heap axioms) and passes to Boogie, which invokes Z3.
- **Verus**: like Dafny but operating on Rust's MIR; the ghost state and `Tracked<T>` types are erased before Z3 query generation.

CBMC (C Bounded Model Checker) targets C programs directly: it unrolls loops to a bounded depth, encodes the resulting acyclic program as a circuit, and emits QF_BV + QF_AUFLIA queries (bit-vectors for machine arithmetic, arrays for memory). CBMC's encoding closely mirrors hardware model checking, and it shares Z3 (or Bitwuzla for bitvector-heavy benchmarks) as its back-end solver.

The SMT-LIB interface is the lingua franca: each tool produces `.smt2` queries that Z3 processes. The DPLL(T) architecture means that as long as the VCs stay within supported theories (BV, LIA, UF, Arrays), the solver is sound and complete. Queries that wander into quantified nonlinear arithmetic may fail to terminate or return `unknown`.

### The Curry-Howard-Lambek Correspondence

The three-way correspondence introduced in [Chapter 12 — Lambda Calculus and Simple Types](../part-03-type-theory/ch12-lambda-calculus-and-simple-types.md) has a model-theoretic completion:

| Logic | Type Theory | Category Theory |
|-------|-------------|----------------|
| Propositions | Types | Objects |
| Proofs | Programs (terms) | Morphisms |
| Proof normalization | Beta reduction | Equality of morphisms |
| Conjunction `A ∧ B` | Product type `A × B` | Product object |
| Implication `A → B` | Function type `A → B` | Exponential object `Bᴬ` |
| Truth `⊤` | Unit type `1` | Terminal object |
| Falsity `⊥` | Empty type `0` | Initial object |
| Disjunction `A ∨ B` | Sum type `A + B` | Coproduct object |
| Cut elimination | Normalization (β-reduction) | Coherence |
| Hypothetical judgment `A ⊢ B` | Typed term `x:A ⊢ e:B` | Morphism `f : A → B` |

The rows read identically: a proof of `A → B` is a term of type `A → B` is a morphism from `A` to `B`. This is not an analogy but an *identity*: the three descriptions are different syntactic presentations of the same mathematical structure.

The category-theoretic perspective (developed in [Chapter 188 — Category Theory for Compiler Engineers](./ch188-category-theory-for-compiler-engineers.md)) reveals why all three columns are *the same structure*: a **Cartesian closed category** (CCC) is simultaneously a proof system for intuitionistic propositional logic, a denotational model for the simply-typed lambda calculus, and a category of computations. The simply-typed λ-calculus is the *internal language* of any CCC; a CCC is a *model* of the simply-typed λ-calculus; an intuitionistic propositional proof system *is* a CCC presentation.

From the model theory side: the *Lindenbaum-Tarski algebra* of a propositional logic is a Boolean algebra (for classical logic) or a Heyting algebra (for intuitionistic logic). The algebra is the model-theoretic quotient of the proof system by provable equivalence: two formulas `φ` and `ψ` are equivalent in the algebra iff `⊢ φ ↔ ψ`. This algebraic model is a finitary analog of a Kripke model (for intuitionistic logic) or a Stone space (for classical logic).

**Practical implication for compiler verification:** The DPLL(T) architecture can be read as a Curry-Howard instance. The theory solver for EUF computes a *congruence closure* — a Heyting algebra quotient of terms by the entailed equalities. The conflict clause returned by the T-solver when unsatisfiable is a logical *proof* of the negation of the theory literal conjunction. The CDCL solver learns this proof as a clause and uses it to prune the search space. In this sense, every Z3 call that returns `unsat` produces an implicit proof in a fragment of first-order logic; the proof is the resolution refutation witnessed by the conflict clauses.

---

## 185.8 Summary

The landscape of mathematical logic, as seen from the compiler-engineering vantage, divides cleanly into three tiers:

- **The undecidable core**: full first-order logic (Church), Peano Arithmetic (Gödel), and nonlinear integer arithmetic (Matiyasevich). These set the theoretical limits but are not obstacles to practical verification — compiler VCs never require reasoning at this level.

- **The decidable archipelago**: propositional logic (NP), EUF (PTIME), Presburger/linear arithmetic over ℤ and ℝ (NP to PTIME), and bit-vector arithmetic (NP via bitblasting). These are the working theories of SMT-based compiler verification. The DPLL(T) architecture and Nelson-Oppen combination make multi-theory queries tractable.

- **The meta-logical infrastructure**: completeness (Gödel, Henkin), cut-elimination (Gentzen), quantifier elimination (Tarski, Presburger), and Curry-Howard link the syntactic (provability) and semantic (truth) perspectives. These theorems justify the correctness of the tools we build on.

Key takeaways:

- Natural deduction structures proofs as trees with local hypothesis discharge; the sequent calculus structures them as sequents with the provably-eliminable cut rule; Hilbert systems minimize primitives at the cost of proof size; resolution enables automated refutation.
- Tarski semantics grounds first-order logic in structures `M = (D, I)` with the satisfaction relation `M ⊨ φ`; Gödel's completeness theorem ensures `⊢` and `⊨` coincide for FOL.
- Gödel's incompleteness theorems apply to theories extending Peano Arithmetic; they are irrelevant to the decidable fragments (Presburger arithmetic, QF_BV, EUF) that compiler verification tools actually use.
- The decidability spectrum has a sharp boundary: linear arithmetic (Presburger, ℤ; Tarski, ℝ) is decidable; nonlinear integer arithmetic is not. Bit-vector arithmetic is decidable by reduction to SAT.
- DPLL(T) combines a CDCL propositional solver with pluggable T-solvers via a conflict-clause interface; Nelson-Oppen combines disjoint stably-infinite theories by equality sharing.
- SMT-LIB 2 is the standardized query language; Z3's theory stack (T_LIA, T_BV, T_UF, T_Arrays, T_FP) directly supports the VCs generated by Alive2, Dafny, and Verus.
- Model theory's key contributions: elementary equivalence and embeddings, quantifier elimination (RCF decidability, Presburger QE), and the ultraproduct proof of compactness.
- HOL is undecidable but verification-practical via the LCF kernel design and Sledgehammer; Coq/Lean use dependent type theory (CIC) rather than HOL.
- The Curry-Howard-Lambek correspondence unifies proof theory, type theory, and category theory: propositions = types = objects; proofs = programs = morphisms; cut elimination = beta reduction = coherence.

---

## References

- Enderton, H.B. *A Mathematical Introduction to Logic*, 2e. Academic Press, 2001. The standard graduate text for FOL syntax, semantics, completeness, and incompleteness.
- van Dalen, D. *Logic and Structure*, 5e. Springer, 2013. Proof systems, natural deduction, model theory.
- Chang, C.C. and Keisler, H.J. *Model Theory*, 3e. Dover, 1990. The definitive reference for ultraproducts, elementary embeddings, and quantifier elimination.
- Gödel, K. "Über formal unentscheidbare Sätze der Principia Mathematica und verwandter Systeme I." *Monatshefte für Mathematik und Physik* 38:173–198, 1931.
- Gentzen, G. "Untersuchungen über das logische Schließen." *Mathematische Zeitschrift* 39:176–210, 405–431, 1935.
- Robinson, J.A. "A Machine-Oriented Logic Based on the Resolution Principle." *JACM* 12(1):23–41, 1965.
- Tarski, A. *A Decision Method for Elementary Algebra and Geometry*, 2e. Rand Corporation, 1951. QE for real closed fields.
- de Moura, L. and Bjørner, N. "Z3: An Efficient SMT Solver." *TACAS 2008*, LNCS 4963. [[doi](https://doi.org/10.1007/978-3-540-78800-3_24)]
- Nieuwenhuis, R., Oliveras, A., and Tinelli, C. "Solving SAT and SAT Modulo Theories: From an Abstract Davis-Putnam-Logemann-Loveland Procedure to DPLL(T)." *JACM* 53(6):937–977, 2006.
- Nelson, G. and Oppen, D.C. "Simplification by Cooperating Decision Procedures." *TOPLAS* 1(2):245–257, 1979.
- Pugh, W. "The Omega Test: A Fast and Practical Integer Programming Algorithm for Dependence Analysis." *SC 1991*.
- Harrison, J. "HOL Light: A Tutorial Introduction." *FMCAD 1996*. [[HOL Light source](https://github.com/jrh13/hol-light)]
- SMT-LIB standard: [smtlib.cs.uiowa.edu](https://smtlib.cs.uiowa.edu/)

---

*@copyright jreuben11*
