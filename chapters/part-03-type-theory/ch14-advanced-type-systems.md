# Chapter 14 — Advanced Type Systems

*Part III — Type Theory*

The preceding two chapters built the formal scaffolding for simple types (Chapter 12) and polymorphism with inference (Chapter 13). Both systems share a fundamental limitation: types and values inhabit entirely separate universes, and types can neither depend on values nor express fine-grained invariants about the data they classify. A vector is a vector regardless of its length; an integer is an integer regardless of its sign. This chapter dismantles that limitation systematically.

We begin with **dependent types** — the extension that allows types to be indexed by values — and the type theories that build on them: Martin-Löf Type Theory, the Calculus of Constructions, and the Calculus of Inductive Constructions as realized in Coq and Lean 4. We then study **subtyping** in its width, depth, structural, and nominal flavors, extend System F with bounded quantification, and arrive at the resource-sensitive type systems — **linear, affine, and relevant types** — that underpin Rust's ownership model. The latter half surveys **uniqueness types**, **region types**, **effect systems**, **refinement types**, **gradual typing**, **session types**, and the application of dependent types to systems programming via ATS and F*/Low*.

Each topic is presented at a level appropriate for compiler engineers who need to reason precisely about what guarantees a type system actually provides, and what it costs to check or infer those guarantees.

**Why this matters for compiler engineers.** MLIR's type system allows dialect-defined types that carry semantic invariants (shaped tensors, ranked vs. unranked buffers, verified protocol states). The LLVM IR type system is deliberately weak — erased and flat — which is exactly the right tradeoff once dependent-type guarantees have been exploited at the source level and need not be re-checked at every pass. Understanding the full spectrum from MLTT to Liquid Haskell to Rust's borrow checker makes it possible to reason about which invariants survive compilation, which are erased, and which must be re-established at the IR level through metadata and attributes. Chapter 15 connects this theoretical apparatus to the concrete LLVM and MLIR type systems directly.

---

## 1. Dependent Types: Π and Σ Types

### The Motivation

Consider the type of a safe array-access function. In STLC or System F (Chapter 12, Chapter 13), the best we can do is `get : Array T → Nat → T`, which cannot rule out out-of-bounds access statically. In a **dependent type system**, we can write:

```
get : Π(n : Nat). Array(T, n) → Fin(n) → T
```

Here `Array(T, n)` is the type of arrays of `T` with exactly `n` elements, and `Fin(n)` is the type of natural numbers strictly less than `n`. The type of `get` depends on the value `n`. A call `get 5 arr i` is well-typed only when `arr : Array(T, 5)` and `i : Fin(5)` — the compiler can reject out-of-bounds accesses **at type-checking time**, with no runtime overhead in the fast path.

The canonical example is the **length-indexed vector**:

```
Vec : (A : Type) → Nat → Type
Vec A 0      = Nil
Vec A (S n)  = Cons A (Vec A n)
```

The type `Vec A n` has type constructor `Vec` applied to the value `n : Nat`. The type of `append` becomes:

```
append : Π(A:Type). Π(m n : Nat). Vec A m → Vec A n → Vec A (m+n)
```

The return type `Vec A (m+n)` depends on the values `m` and `n`. This is a static proof that append produces a vector whose length is the sum of its inputs' lengths — caught at compile time, erased at runtime.

### Π Types (Dependent Function Types)

[TAPL §29.1; PFPL §36]

The **Π type** (dependent product type, dependent function type) is written `Π(x:A).B(x)`. A term `f` of type `Π(x:A).B(x)` takes an argument `a : A` and returns a result of type `B(a)` — the return type may vary with the argument value.

The formation, introduction, and elimination rules are:

```
Formation:
  Γ ⊢ A : Type    Γ, x:A ⊢ B(x) : Type
  ─────────────────────────────────────────
         Γ ⊢ Π(x:A).B(x) : Type

Introduction (Λ-abstraction):
  Γ, x:A ⊢ b(x) : B(x)
  ──────────────────────────────
  Γ ⊢ λ(x:A).b(x) : Π(x:A).B(x)

Elimination (application):
  Γ ⊢ f : Π(x:A).B(x)    Γ ⊢ a : A
  ──────────────────────────────────
        Γ ⊢ f a : B(a)
```

The computation rule (β-reduction) is:

```
(λ(x:A).b(x)) a  ≡  b[a/x]   (definitional equality)
```

When `B` does not depend on `x`, `Π(x:A).B` reduces to the ordinary function type `A→B` of STLC. Π types strictly generalize non-dependent function types.

**Example.** The `replicate` function builds a vector of `n` copies of a value:

```
replicate : Π(A : Type). Π(n : Nat). A → Vec A n
replicate A 0     a = Nil
replicate A (S k) a = Cons a (replicate A k a)
```

The return type `Vec A n` depends on the value `n`, and the two clauses reduce to definitionally equal normal forms at each concrete `n`.

### Σ Types (Dependent Pair / Dependent Sum)

[TAPL §29.2; PFPL §37]

The **Σ type** (dependent sum type, dependent pair type) is written `Σ(x:A).B(x)`. A term of type `Σ(x:A).B(x)` is a pair `(a, b)` where `a : A` and `b : B(a)`. The second component's type depends on the first component's value.

```
Formation:
  Γ ⊢ A : Type    Γ, x:A ⊢ B(x) : Type
  ─────────────────────────────────────────
         Γ ⊢ Σ(x:A).B(x) : Type

Introduction (pairing):
  Γ ⊢ a : A    Γ ⊢ b : B(a)
  ─────────────────────────────────────
  Γ ⊢ (a, b) : Σ(x:A).B(x)

Elimination (projections):
  Γ ⊢ p : Σ(x:A).B(x)          Γ ⊢ p : Σ(x:A).B(x)
  ────────────────────────      ─────────────────────────
  Γ ⊢ π₁(p) : A                 Γ ⊢ π₂(p) : B(π₁(p))
```

The computation rules are `π₁(a, b) ≡ a` and `π₂(a, b) ≡ b`.

When `B` does not depend on `x`, `Σ(x:A).B` reduces to the ordinary product type `A×B`. Σ types extend products just as Π types extend function types.

**As existentials.** The Σ type `Σ(x:A).B(x)` is the type-theoretic encoding of `∃x:A. B(x)`. A term of this type provides a witness `a` and a proof that `B(a)` holds. This is the constructive reading of the existential quantifier, which we will revisit under the BHK interpretation in Section 2.

**Example.** The type of bounded integers can be expressed as:

```
BoundedInt(lo hi : Int) = Σ(n : Int). (lo ≤ n ∧ n < hi)
```

A value of this type is a pair of an integer and a proof that it lies in the stated range. The proof is erased at runtime; the static check occurs at elaboration time.

### Identity Types and the J Eliminator

[PFPL §38; Martin-Löf 1984]

For any type `A` and terms `a b : A`, the **identity type** (equality type, path type) `Id_A(a, b)` is the type of proofs that `a` and `b` are (propositionally) equal. Its single constructor is:

```
refl_a : Id_A(a, a)
```

The identity type is *inhabited* exactly when there exists a proof of equality. It is not a boolean — it is a type, and the proof is a first-class term.

The **J eliminator** (the induction principle for identity types) is:

```
J : Π(A:Type). Π(C : Π(x y:A). Id_A(x,y) → Type).
    (Π(x:A). C x x (refl_x)) →
    Π(x y : A). Π(p : Id_A(x,y)). C x y p
```

Informally: to prove a property `C` about any equality proof `p : Id_A(x,y)`, it suffices to prove it for the reflexivity proof `refl_x` at each `x:A`. This is the dependent version of the substitution principle: equals can be substituted for equals, but constructively.

### Universes and Girard's Paradox

[PFPL §40; Hurkens 1995]

In a dependent type system, types are terms, so we need a type of types. The standard solution is a **hierarchy of universes**:

```
Type₀ : Type₁ : Type₂ : ...
```

where `Typeₙ : Typeₙ₊₁`. Each universe is closed under Π, Σ, and identity formation. The hierarchy is **predicative**: when forming `Π(x:A).B`, if `A : Typeₙ` and `B : Typeₙ` then `Π(x:A).B : Typeₙ`.

The temptation is to collapse the hierarchy by declaring `Type : Type` (an **impredicative** universe). This is inconsistent: Girard's paradox [Girard 1972; Hurkens 1995] shows that `Type : Type` allows encoding of the Burali-Forti paradox from set theory, deriving `False`. The Coq system uses a separate `Prop` sort that is impredicative (and proof-irrelevant) while keeping `Set` and `Type` predicative.

---

## 2. Martin-Löf Type Theory

[Martin-Löf 1984; PFPL §§36–44; Nordström, Petersson, Smith 1990]

**Intuitionistic Type Theory** (ITT), developed by Per Martin-Löf beginning in 1972, is the foundational framework from which most modern dependent type systems descend. It simultaneously serves as a foundation for constructive mathematics, a programming language, and a proof assistant core.

### The Four Judgments

MLTT has four basic forms of judgment:

| Judgment | Reading |
|----------|---------|
| `A : Type` | `A` is a type |
| `a : A` | `a` is a term of type `A` |
| `A = B : Type` | `A` and `B` are definitionally equal types |
| `a = b : A` | `a` and `b` are definitionally equal terms of type `A` |

Definitional equality (the last two judgments) is the equational theory built into the type checker — it holds when both sides reduce to the same normal form under the computation rules. It is distinct from *propositional* equality `Id_A(a,b)`, which requires an explicit proof term.

### The Basic Types

MLTT contains the following type formers:

- **Unit (1)**: a single constructor `tt : 1`; the type of trivial proofs
- **Empty (0)**: no constructors; the type of false; `absurd : 0 → A` for any `A`
- **Bool (2)**: constructors `true, false : 2`; elimination by case analysis
- **Nat**: constructors `O : Nat` and `S : Nat → Nat`; elimination by the recursor `Nat-rec`
- **Product A×B**: constructors `pair : A → B → A×B`; projections `fst`, `snd`
- **Coproduct A+B**: constructors `inl : A → A+B`, `inr : B → A+B`; elimination by case analysis
- **Π(x:A).B(x)**: dependent function types (Section 1)
- **Σ(x:A).B(x)**: dependent pair types (Section 1)
- **Id_A(a,b)**: identity types (Section 1)

Each type former has four rules: **formation** (when is the type well-formed), **introduction** (the constructors), **elimination** (how to use a value of this type), and **computation** (definitional reduction of eliminators applied to constructors).

### Definitional vs. Propositional Equality

A critical distinction in MLTT is between two notions of equality:

- **Definitional equality** (`a = b : A` as a judgment): holds when both sides are convertible — they reduce to the same normal form under the β, η, and ι (iota, for inductives) reduction rules. The type checker decides this automatically. No proof term is needed; the type checker accepts it or rejects it.

- **Propositional equality** (`Id_A(a,b)` as a type): requires an explicit proof term. You can have `Id_Nat(n+0, n)` as a propositional equality, proved by induction on `n`, even though `n+0` and `n` are not definitionally equal when `+` is defined by recursion on the first argument.

The two are connected: definitional equality implies propositional equality (by reflexivity), but the converse — the **reflection rule** — is not admitted in intensional MLTT. Admitting reflection (propositional ⟹ definitional) gives **extensional** MLTT, which makes type checking undecidable [Hofmann 1995].

Homotopy Type Theory (HoTT) takes a different path: it retains intensional MLTT but interprets identity proofs as *paths* in a space, leading to the **univalence axiom** (Voevodsky) and an equivalence between mathematical equivalence and type isomorphism.

**The univalence axiom** [Voevodsky; HoTT Book 2013] states:

```
ua : (A ≃ B) ≃ Id_Type(A, B)
```

An equivalence `A ≃ B` (a pair of functions `f : A → B` and `g : B → A` with `g ∘ f = id` and `f ∘ g = id`) is itself equivalent to a proof that `A` and `B` are the same type. This makes **isomorphic structures identical** in the type theory — a long-sought goal for mathematical foundations (where one wishes to treat isomorphic groups, rings, or spaces as interchangeable).

From the perspective of a compiler engineer, univalence has a striking consequence: if two types are isomorphic, you can freely substitute one for the other in any context, and this substitution is **proof-relevant** (the specific isomorphism chosen matters, not just its existence). This is related to the notion of **type-directed program transformation** in compilers: a shape-preserving transformation of tensor types in MLIR's affine dialect (Chapter 15) can be understood as applying univalence to commute a type equivalence through a program.

### Propositions as Types: The BHK Interpretation

[Brouwer-Heyting-Kolmogorov; Martin-Löf 1984; PFPL §25]

The cornerstone of MLTT is the identification of **propositions with types** and **proofs with programs** — the Curry-Howard correspondence extended to full constructive mathematics.

Under the **BHK interpretation**:

| Logical connective | Type | Proof = Program |
|-------------------|------|----------------|
| `A ∧ B` | `A × B` | a pair `(a, b)` where `a` proves `A` and `b` proves `B` |
| `A ∨ B` | `A + B` | either `inl a` (proof of `A`) or `inr b` (proof of `B`) |
| `A → B` | `A → B` | a function that converts any proof of `A` to a proof of `B` |
| `¬A` | `A → 0` | a function showing `A` leads to contradiction |
| `∀x:A. P(x)` | `Π(x:A). P(x)` | a function returning a proof of `P(x)` for each `x` |
| `∃x:A. P(x)` | `Σ(x:A). P(x)` | a witness `a` together with a proof of `P(a)` |
| `⊤` | `1` | the trivial proof `tt` |
| `⊥` | `0` | no proof exists (empty type) |

This is not merely an analogy — in MLTT, propositions *are* types and proofs *are* terms. A proof of `∀n:Nat. n+0=n` is literally a function of type `Π(n:Nat). Id_Nat(n+0, n)`, computed by induction on `n`. Programs and proofs are unified in a single framework.

The computational interpretation is crucial: proofs are not inert certificates but executable programs. Running a proof of `∃n:Nat. Prime(n) ∧ n > 100` extracts the prime number `n` and the primality proof — the proof *is* the computation.

---

## 3. The Calculus of Constructions and Coq's CIC

### Coquand and Huet's Calculus of Constructions

[Coquand and Huet 1988; Barendregt 1992]

The **Calculus of Constructions** (CoC) was introduced by Thierry Coquand and Gérard Huet in 1988. It unifies types and terms in a single syntactic category and extends both System F (Chapter 13) and MLTT's Π types into a single framework.

In CoC, the single sort `*` contains all propositions and types, and `□` is the sort of `*`. The type theory has:

- **Sorts**: `* : □`
- **Π types**: `Π(x:A).B` for any `A : s₁` and `B : s₂` where `(s₁, s₂) ∈ { (*, *), (□, *), (□, □), (*, □) }`
- Terms are classified by their types; types are themselves terms; large elimination is possible

### The λ-Cube

[Barendregt 1991; TAPL §29.4]

The **λ-cube** organizes type theories along three independent axes, each allowing a new form of dependency:

```
            λω ─────── λPω  (= CoC)
           /│          /│
          / │         / │
        λ2  │       λP2 │
         │  λω̲ ────│── λPω̲
         │ /        │ /
         │/         │/
         λ→ ─────── λP
```

The three axes are:

| Axis | Feature | Example system |
|------|---------|---------------|
| **→** | Terms depending on terms | λ→ = STLC |
| **2** | Types depending on types (type abstraction) | λ2 = System F |
| **P** | Types depending on terms (dependent types) | λP = LF |
| **ω** | Types depending on types (type operators) | λω = Fω |

The **top corner** of the cube — all three axes simultaneously — is the **Calculus of Constructions** λC (also written λPω or CoC), encompassing STLC, System F, LF, and Fω as subsystems.

Moving along each axis:
- `λ→ → λ2`: add ∀α.τ (System F)
- `λ→ → λP`: add Π(x:A).B dependent on values (LF)
- `λ→ → λω`: add type operators (type-level functions `λα::K.τ`)
- All three → CoC

### Coq's Calculus of Inductive Constructions

[Paulin-Mohring 1993; Bertot and Castéran 2004]

CoC alone is powerful but lacks **inductive types**: there is no primitive `Nat`, `List`, or `Tree` — only Church-encoding approximations, which are expressively incomplete. Christine Paulin-Mohring added **inductive definitions** to produce the **Calculus of Inductive Constructions** (CIC), the foundation of Coq.

In Coq's CIC, an inductive type is declared by specifying its **type** and **constructors**:

```coq
Inductive nat : Type :=
  | O : nat
  | S : nat → nat.

Inductive list (A : Type) : Type :=
  | nil  : list A
  | cons : A → list A → list A.
```

The **elimination principle** (recursion/induction scheme) is automatically generated:

```coq
nat_rec : ∀ (P : nat → Type),
          P O →
          (∀ n, P n → P (S n)) →
          ∀ n, P n
```

**The positivity condition.** Not every inductive declaration is allowed. The constructors must be **strictly positive** in the type being defined: the recursive type may only appear in the *codomain* of constructor argument types, not in the *domain* of a negative position. This rules out:

```coq
(* REJECTED — non-positive *)
Inductive Bad : Type :=
  | mk : (Bad → nat) → Bad.
```

If negative occurrences were allowed, one could encode a fixed-point combinator and break termination and consistency [Coquand 1994].

**Termination via structural recursion.** In Coq, recursive functions defined with `Fixpoint` must terminate. Coq's guardedness checker requires that every recursive call is on a *structurally smaller* argument (a subterm of the constructor-decomposed input). This is decidable but conservative; for more complex termination arguments, well-founded recursion via an accessibility relation is used.

### Lean 4's Dependent Type Theory

[de Moura and Ullrich 2021; Lean 4 documentation]

Lean 4, developed at Amazon and Microsoft Research, uses a type theory closely related to CIC with several notable differences:

- **Prop vs. Type**: Lean separates `Prop` (a proof-irrelevant sort) from `Type u` (a universe-polymorphic sort indexed by a level `u`)
- **Proof irrelevance**: Two proofs of the same proposition are definitionally equal. This means `Prop` types can be freely discarded at runtime.
- **Universe polymorphism**: Definitions can be quantified over universe levels: `∀ (u : Level), Sort u`
- **Quotient types**: First-class quotient types `Quotient r` for a setoid relation `r`, with elimination requiring proof that the function respects the relation
- **`#check` mechanism**: The `#check` command elaborates a term and reports its type, serving as the primary interactive tool during proof development

Lean 4's kernel is significantly simpler than Coq's, and the elaborator is implemented in Lean 4 itself (unlike Coq's OCaml implementation), enabling metaprogramming with the same language used for proofs.

**Coq's Prop vs. Set.** The separation between `Prop` (proof-irrelevant, impredicative) and `Type` (proof-relevant, predicative) has concrete consequences for program extraction:

- Code extracted from a `Prop` proof can be **erased**: since any two proofs of the same proposition are definitionally equal, the specific proof term carries no computational content
- Code extracted from a `Set` or `Type` computation is **retained**: the computational content matters

This matches the compile-time/run-time distinction. When ATS and F* erase proof terms (Section 13), they are essentially extracting from the `Prop` fragment. The `Set`/`Type` fragment gives executable programs; the `Prop` fragment gives certificates that are erased.

**Cubical type theory** [Cohen et al. 2016; Angiuli et al. 2017] is a more recent development that realizes HoTT in a type theory with computational content: the univalence axiom becomes a **theorem** (not an axiom) with a computational reduction rule. Cubical Agda and Cubical Lean implementations demonstrate that this is not merely theoretical — it enables computation with types as geometric objects, enabling proof search by normalization.

---

## 4. Subtyping: Width, Depth, Structural, and Nominal

[TAPL §§15–17; Reynolds 1974]

### The Subtype Relation

A type `S` is a **subtype** of `T`, written `S <: T`, if a value of type `S` can be used wherever a value of type `T` is expected — the Liskov Substitution Principle (LSP) at the type level. Subtyping extends the type system with an **implicit coercion** rule:

```
S-Sub (Subsumption):
  Γ ⊢ e : S    S <: T
  ──────────────────────
       Γ ⊢ e : T
```

Subtyping is a **preorder**: it is reflexive (`T <: T`) and transitive (`S <: U` and `U <: T` implies `S <: T`).

### Width Subtyping for Records

**Width subtyping** [TAPL §15.3] says a record type with *more* fields is a subtype of a record type with *fewer* fields:

```
S-RcdWidth:
  ────────────────────────────────────────────────────────
  {l₁:T₁, ..., lₙ:Tₙ, lₙ₊₁:Tₙ₊₁, ..., lₘ:Tₘ} <: {l₁:T₁, ..., lₙ:Tₙ}
```

Intuition: if a function expects a record with fields `{x:Int}` and I pass a record with fields `{x:Int, y:Int}`, the function can still access `x`; the extra field `y` is ignored. A record with more fields "provides everything the supertype requires, and more."

This matches object-oriented inheritance at the record level: a subclass that adds new fields is still usable as its superclass.

### Depth Subtyping for Records

**Depth subtyping** [TAPL §15.4] allows covariant subtyping within record fields:

```
S-RcdDepth:
  T₁ <: S₁   ...   Tₙ <: Sₙ
  ────────────────────────────────────────────
  {l₁:T₁, ..., lₙ:Tₙ} <: {l₁:S₁, ..., lₙ:Sₙ}
```

This is **unsound with mutable fields**. Suppose `Cat <: Animal` and we have:

```
r : {pet : Cat}    r <: {pet : Animal}    (by depth subtyping)
```

Through the `{pet : Animal}` view, we can write `r.pet := some_dog` (where `Dog <: Animal`). But `r` still has type `{pet : Cat}`, so reading `r.pet` and treating it as `Cat` is now type-unsafe. This is the standard **mutable-field depth subtyping unsoundness**.

Languages avoid this by restricting depth subtyping to immutable fields (OCaml object fields are immutable by default for this reason) or using **variance annotations** (Java arrays are covariant and famously unsound; Scala uses `+T/-T` annotations).

### Structural vs. Nominal Subtyping

The two fundamental flavors of subtyping disagree on what determines whether `S <: T`:

**Structural subtyping**: `S <: T` is determined entirely by the *structure* of the types. If `S` has all the methods/fields of `T` with compatible types, then `S <: T`, regardless of whether `S` was declared as a subtype of `T`.

- Go interfaces: a type implicitly satisfies an interface if it has all the required methods
- TypeScript: structural throughout; `{x:number, y:number}` satisfies `{x:number}` without declaration

**Nominal subtyping**: `S <: T` only if it was *explicitly declared* (via `extends`, `implements`, or similar). Structure is irrelevant; the programmer must declare the relationship.

- Java: `class Cat extends Animal` — `Cat <: Animal` only because of this declaration
- C++: public inheritance declares subtyping; unrelated classes with the same methods are not subtypes

The tradeoffs are significant:
- Structural subtyping enables **duck typing at the type level** — code that was written before an interface was declared can still satisfy it retroactively
- Nominal subtyping enables **intentional design** — two types with the same structure but different semantics (e.g., `Celsius` and `Fahrenheit`, both `Float`) are not subtypes

### The Liskov Substitution Principle

Barbara Liskov's original formulation [Liskov and Wing 1994] goes beyond structural subtyping:

> **LSP**: If `φ(x)` is a property provable about objects `x` of type `T`, then `φ(y)` should be true for objects `y` of type `S` where `S <: T`.

This is a *behavioral* requirement, not just a structural one. A `Square` that overrides `Rectangle.setWidth` to also set the height (to remain square) violates the LSP even if `Square <: Rectangle` structurally. Behavioral subtyping is not mechanically checkable by type systems alone; it requires reasoning about specifications (pre/postconditions), which is what refinement types and dependent types add.

### Subtyping for Function Types

[TAPL §15.2]

The subtyping rule for function types is the central result of subtyping theory. We derive it from the substitutability principle.

Suppose `f : S₁ → S₂` and we want to use `f` where a `T₁ → T₂` is expected. The caller will supply an argument of type `T₁` and expect a result of type `T₂`. For this to be safe:

1. **Argument (contravariance)**: the caller passes a `T₁`; `f` expects an `S₁`. For `f` to accept the argument, we need `T₁ <: S₁` — the required type must be a subtype of the function's domain.

2. **Result (covariance)**: `f` returns an `S₂`; the caller expects a `T₂`. For the result to be usable, we need `S₂ <: T₂` — the returned type must be a subtype of the expected type.

Combining:

```
S-Arrow:
  T₁ <: S₁    S₂ <: T₂
  ────────────────────────
  (S₁ → S₂) <: (T₁ → T₂)
```

The function type is **contravariant in the argument and covariant in the result**. This is the standard result, matching Java's method overriding rules (covariant return types since Java 5; argument types remain invariant in Java but are contravariant in theory), Kotlin's declaration-site variance, and Scala's `Function[-A, +B]`.

---

## 5. Bounded Quantification: System F<:

[TAPL §§26–28; Cardelli and Wegner 1985]

### Extending System F with Subtyping

**System F<:** combines the parametric polymorphism of System F (Chapter 13) with subtyping. Type variables are given **upper bounds**: instead of `∀α.τ` (any type), we write `∀(α <: T).τ` (any subtype of `T`).

The formation rule for bounded quantification:

```
  Γ, α <: T ⊢ τ : Type
  ─────────────────────────────
  Γ ⊢ ∀(α <: T). τ : Type
```

The type application rule (T-TApp) becomes:

```
T-TApp:
  Γ ⊢ M : ∀(α <: T).τ    Γ ⊢ S <: T
  ──────────────────────────────────────
       Γ ⊢ M[S] : τ[S/α]
```

Instantiation `M[S]` is valid only when `S <: T` — the supplied type must satisfy the bound.

### Kernel F<: vs. Full F<:

The difference concerns the **type abstraction rule** (T-TAbs). There are two variants:

**Kernel F<:** requires that the bound of the variable introduced by abstraction is a *ground* type (not involving the bound variable):

```
Kernel T-TAbs:
  Γ, α <: T ⊢ M : τ    T does not mention α
  ────────────────────────────────────────────
  Γ ⊢ ΛαT.M : ∀(α <: T).τ
```

**Full F<:** allows the bound to be any well-formed type in the extended context, enabling F-bounded polymorphism:

```
Full T-TAbs:
  Γ, α <: T ⊢ M : τ
  ──────────────────────────────
  Γ ⊢ Λ(α<:T).M : ∀(α <: T).τ
```

The crucial difference: Kernel F<: has a **decidable subtype checking algorithm** (the kernel algorithm runs in time cubic in the depth of the bound). Full F<: is **undecidable** [Pierce and Steffen 1994] — subtype checking in Full F<: is equivalent to the halting problem.

### F-Bounded Polymorphism and Java Generics

**F-bounded polymorphism** [Canning et al. 1989] is the special case where the bound mentions the variable itself:

```
∀(α <: Comparable<α>). α → α → Bool
```

This is exactly Java's `<T extends Comparable<T>>`:

```java
<T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}
```

Java **wildcard types** add use-site variance:
- `? extends T`: an unknown subtype of `T` (covariant use; read-only)
- `? super T`: an unknown supertype of `T` (contravariant use; write-only)

These correspond to **existential types** in the type-theoretic encoding: `List<? extends T>` is `∃α <: T. List<α>`. The PECS (Producer Extends, Consumer Super) rule follows directly from the S-Arrow covariance/contravariance analysis.

The undecidability of Full F<: has practical implications: Java's type checker sidesteps it through restrictions on wildcard nesting and a fixed-point algorithm for F-bounded constraints, trading completeness for termination [Kennedy and Pierce 2007].

---

## 6. Linear Logic and Linear Types

[Girard 1987; TAPL §§3.4, 15; ATTAPL Ch. 1]

### Girard's Linear Logic

Jean-Yves Girard's **linear logic** (1987) is a **resource-sensitive** logic where propositions represent resources that must be used **exactly once**. Classical and intuitionistic logics have the structural rules of **weakening** (an unused assumption is harmless) and **contraction** (an assumption can be used multiple times). Linear logic restricts these:

| Structural rule | Classical/Intuitionistic | Linear |
|----------------|------------------------|--------|
| Exchange | Allowed | Allowed |
| Weakening | Allowed | Restricted |
| Contraction | Allowed | Restricted |

The **!** ("of course" or "bang") modality re-introduces unlimited use: `!A` is a resource that can be used arbitrarily many times. Intuitionistic logic is embedded by translating `A` as `!A`.

### Linear Types in Programming Languages

A **linear type** is a type whose values must be used **exactly once** in any term. A linearly typed variable:
- Cannot be discarded (no weakening)
- Cannot be duplicated (no contraction)
- Must appear exactly once in the term body

The typing contexts are **multisets** rather than sets: a variable occurrence consumes it from the context, and a variable not present cannot be used.

```
Linear T-App (multiplicative conjunction):
  Γ ⊢ f : A ⊸ B    Δ ⊢ a : A    Γ ∩ Δ = ∅
  ──────────────────────────────────────────
            Γ, Δ ⊢ f a : B
```

The disjoint-context condition `Γ ∩ Δ = ∅` prevents sharing resources between the function and its argument.

### Affine and Relevant Types

Two important weakenings of linear types:

**Affine types**: used **at most once**. Weakening is allowed (you can drop an affine value without consuming it), but contraction is not (you cannot duplicate it). An affine variable may appear zero or one times.

**Relevant types**: used **at least once**. Contraction is allowed (you can duplicate), but weakening is not (you must use it at least once).

The resource interpretation connects naturally to programming:
- Linear: must use exactly once — file handles that must be closed, channels that must be communicated on
- Affine: must use at most once — Rust's owned values (can be dropped without using)
- Relevant: must use at least once — values where ignoring is a bug (certain error codes)

### The Linear λ-Calculus

The **linear lambda calculus** (ILL, Intuitionistic Linear Logic) is the term calculus associated with intuitionistic linear logic [Benton 1995; Wadler 1993]. The grammar extends the standard λ-calculus with a multiplicative conjunction (`⊗`), a linear function arrow (`⊸`), and the bang modality:

```
Types:  A, B ::= 1 | A ⊗ B | A ⊸ B | !A | A & B | A ⊕ B | 0
Terms:  M, N ::= x | λ(x:A).M | M N | (M, N) | let (x,y) = M in N
              | !M | let !x = M in N | ...
```

The crucial rule for the bang modality:

```
!-Introduction:
  !Γ ⊢ M : A
  ─────────────
  !Γ ⊢ !M : !A

(where !Γ means every variable in Γ has a ! type)
```

This says: to promote a term to a !-type, every free variable it uses must itself be !-typed (i.e., already available for unlimited use). The bang modality is thus the gateway that allows linear code to call code written in the intuitionistic fragment.

**The !-Weakening and !-Contraction rules:**

```
!-Weakening:
  Γ ⊢ M : B    x : !A ∉ FV(M)
  ─────────────────────────────
     Γ, x : !A ⊢ M : B

!-Contraction:
  Γ, x:!A, y:!A ⊢ M : B
  ─────────────────────────────────
  Γ, z:!A ⊢ M[z/x][z/y] : B
```

Weakening and contraction are allowed *only* for !-typed variables. This is the formal mechanism by which Rust's `Copy` types — which have unlimited use — correspond to variables of type `!A` in the linear type theory.

**Certified resource management.** Languages like Idris 2 adopt a quantitative type theory where each variable is annotated with a **quantity** — 0 (erased, compile-time only), 1 (linear), or ω (unrestricted). This gives a single unified framework for dependent types, linear types, and erasure, making it possible to write both proofs (quantity 0) and efficient programs (quantity ω) in the same language with the same syntax.

---

## 7. Affine Types and Rust's Borrow Checker

[Jung et al. RustBelt 2018; Pearce 2021; Matsakis and Klock 2014]

### Rust's Ownership as an Affine Type System

Rust's type system is, at its core, an **affine type system**: each value has exactly one owner, and **moving** a value transfers ownership, rendering the source variable inaccessible (affine, not strictly linear, because values can be `drop`ped without consuming them):

```rust
let s1 = String::from("hello");
let s2 = s1;           // move: s1 is no longer valid
// println!("{}", s1); // ERROR: value used after move
```

In type-theoretic terms, `String` has an affine type: the binding `s1` introduces an affine variable, `let s2 = s1` consumes it (one use), and any subsequent use of `s1` is a type error.

Types that implement `Copy` are non-affine (they can be duplicated freely). `Drop` types without `Copy` are affine. This is exactly the Curry-Howard view: `Copy` corresponds to the **contraction** structural rule being available for that type.

### Borrowing as Temporary Aliasing

Rust adds a layer beyond simple affine types: **borrowing** allows temporary, scoped aliasing without consuming ownership.

- `&T` (**shared reference**, "borrow"): allows reading; multiple shared borrows may coexist; the owned value cannot be mutated while any shared borrow is live
- `&mut T` (**exclusive reference**, "mutable borrow"): allows reading and writing; there may be at most one exclusive borrow at any time; no other borrows may coexist

The key invariant is the **aliasing XOR mutability** (AXM) principle: at any point, either one value has arbitrarily many shared readers, or exactly one exclusive writer, but never both.

### The NLL Borrow Checker as a Flow-Sensitive Type System

Rust's borrow checker operates on the **Mid-Level Intermediate Representation** (MIR), a CFG-based representation similar in spirit to SSA. The **Non-Lexical Lifetimes** (NLL) analysis [Matsakis 2018] computes the **liveness** of borrows as a region of program points — the minimal set of points at which a borrow must be live to satisfy all uses.

Formally, a lifetime `'a` is a set of program points `Locs ⊆ ProgramPoints`. The NLL constraints are:

1. **Outlives constraints**: `'a: 'b` ("`'a` outlives `'b`") means `Locs('b) ⊆ Locs('a)`
2. **Liveness constraints**: `'a` must be live at point `p` if any path from `p` uses the value through a reference with lifetime `'a`

The borrow checker solves these constraints by a dataflow fixpoint, computing the minimal lifetimes. A **borrow error** is a constraint that cannot be satisfied: e.g., a shared borrow `&'a T` and an exclusive borrow `&'b mut T` with overlapping live ranges `'a ∩ 'b ≠ ∅`.

**Two-phase borrows** [RFC 2025] is an extension that allows a mutable borrow to be used in two phases: a "reservation" phase (during which the borrow is live but not yet exclusive) and an "activation" phase (during which it becomes exclusive). This admits common patterns like `vec.push(vec.len())` that were previously rejected.

### RustBelt: The Formal Safety Proof

[Jung et al. 2018, "RustBelt: Securing the Foundations of the Rust Programming Language"]

Ralf Jung, Jacques-Henri Jourdan, Robbert Krebbers, and Derek Dreyer formalized Rust's type system using **Iris**, a concurrent separation logic framework built on top of Coq. Their key theorem:

> **RustBelt Safety Theorem.** Every well-typed Rust program (in the λRust fragment) is **safe** with respect to the operational semantics: it does not exhibit undefined behavior (no data races, no use-after-free, no type confusion).

The proof technique is **semantic typing**: they define a semantic interpretation of each Rust type as an Iris invariant and show that well-typed programs satisfy the invariant. This is more powerful than syntactic type safety (progress + preservation) because it handles unsafe code: Rust's standard library uses `unsafe` internally, and RustBelt shows that the public safe API maintains the invariants despite the unsafe internals.

### Connection to Session Types

A `&mut T` borrow in Rust can be viewed as a two-state **session type**: the borrower receives the `&mut T` (entering a "borrow" session), performs reads and writes, and releases the borrow (exiting the session). The owner regains access only after the borrow session ends. This is exactly the pattern of a two-state channel in session type theory (Section 12), with the lifetime providing the protocol structure.

---

## 8. Uniqueness Types and Region Types

### Uniqueness Types

[Barendsen and Smetsers 1993; Clean language reference]

**Uniqueness types** (developed in the Clean language) provide a complementary guarantee to linear types: a value with a **unique type** is guaranteed to have **exactly one reference** at the point of use. This enables **destructive update** — the runtime can safely reuse the memory of a uniquely referenced value.

The attribute is written `*T` in Clean. The uniqueness type inference system tracks which values have unique references and which are shared. A function `update : *Array a → Int → a → *Array a` can update the array in place because the unique type guarantees no other reference exists.

The key inference rule:

```
Share:
  u : *T    (u used in two subterms)
  ─────────────────────────────────────
  ERROR: unique value used more than once
```

Uniqueness types differ from linearity in direction: **linear types** track *consumption* (a linear value must be used), while **uniqueness types** track *aliasing* (a unique value has one reference). The two are dual in a precise sense [Wadler 1990].

### Region Types

[Tofte and Talpin 1994, 1997; ATTAPL Ch. 3]

**Region-based memory management** (Tofte and Talpin) assigns every allocated value to a named **region**. A region is a pool of memory; all values in a region are deallocated simultaneously when the region is exited. The **region type system** tracks which region each value lives in, preventing dangling references.

The type of a value is extended with a **region annotation** `τ at ρ`: a value of type `τ` located in region `ρ`. The types of functions carry **region polymorphism**:

```
map : ∀ρ₁ ρ₂. (α at ρ₁ → β at ρ₂) → List(α) at ρ₁ → List(β) at ρ₂
```

The key constraint: a value of type `τ at ρ` cannot outlive region `ρ`. This is enforced by **effect annotations** on function types: a function that allocates in region `ρ` has effect `{alloc ρ}`.

The **region inference algorithm** [Tofte and Talpin 1997] infers region annotations and effects automatically. For the Standard ML of New Jersey region memory manager (MLKit), this allowed significant reductions in GC pressure.

**Limitations**: The Tofte-Talpin system has well-known limitations — it cannot handle certain recursive data structures that span region boundaries, and the inference can produce region annotations that are too coarse (all of a large data structure assigned to one region that lives longer than necessary). Subsequent work (Aiken et al., Cyclone) addressed these.

**Rust's lifetime inference** is spiritually related to region types: a lifetime `'a` in Rust is essentially a region, and the borrow checker's liveness analysis is a variant of region inference. The key difference is that Rust's regions are not allocated on a stack in source-code order — they are computed by the NLL dataflow analysis.

### Cyclone and Safe C with Regions

**Cyclone** [Jim et al. 2002] was an experimental safe dialect of C that added region-based memory management to C's type system. Key contributions:

- **Region variables** `ρ` annotate pointer types: `int *ρ` is a pointer to an `int` in region `ρ`
- **Dangling pointer prevention**: dereferencing `*ρ p` is only valid while region `ρ` is live; the type system enforces this
- **Dynamic regions**: a `region_t<ρ>` capability token controls region creation and destruction; all pointers into `ρ` become invalid when the region is deallocated
- **Unique pointers**: `unique T *` for linear ownership without regions

The Cyclone region checker influenced the design of Rust's lifetime system. The key insight carried forward: lifetimes are not source-code scopes but **regions of program flow** during which a reference is valid. The NLL borrow checker (Section 7) computes these regions precisely, while Cyclone's checker was more conservative (it required explicit region annotations and admitted fewer patterns).

**Arena allocation as a linear region.** A common systems pattern is arena (bump) allocation: all objects for a computation are allocated in a contiguous block and freed together when done. Dependent type systems can model this with a linear region type:

```
arena_alloc : Π(ρ:Region). Π(T:Type). Arena(ρ) ⊸ (T at ρ, Arena(ρ))
```

The linear `Arena(ρ)` thread ensures the arena is used sequentially and freed exactly once. This pattern appears in F*'s `HS.ST` (heap-and-stack) effect and in ATS's linear proof terms for memory ownership.

---

## 9. Effect Systems: Koka and Algebraic Effects

[Gifford and Lucassen 1986; Marino and Millstein 2009; Leijen 2014]

### The Motivation

A **pure** type system like STLC or System F says nothing about whether a function reads global state, raises an exception, or performs I/O. An **effect system** extends the type of a function with an **effect annotation** describing what it may do:

```
f : Int →{div, exn} Int
```

This says `f` is a function from `Int` to `Int` that may diverge (`div`) or raise an exception (`exn`).

Effect systems enable:
- **Static safety**: an expression in a region that must be exception-free can be type-checked to verify it does not have the `exn` effect
- **Optimization**: a function with the empty effect `{}` (pure) can be memoized, reordered, or duplicated freely
- **Capability passing**: the effect annotation describes what the function is *allowed* to do

### Marino and Millstein's Generic Effect System

[Marino and Millstein 2009]

Marino and Millstein's **generic type-and-effect system** parametrizes over an arbitrary **effect lattice** `(E, ≤)`. Effects are combined by the join operation `⊔`. The typing judgment is extended to `Γ ⊢ e : τ ! ε` where `ε : E` is the effect.

The key rules:

```
T-App-Effect:
  Γ ⊢ f : S →{ε₁} T ! ε₂    Γ ⊢ e : S ! ε₃
  ──────────────────────────────────────────────
       Γ ⊢ f e : T ! (ε₁ ⊔ ε₂ ⊔ ε₃)

T-Sub-Effect:
  Γ ⊢ e : τ ! ε    ε ≤ ε'
  ─────────────────────────
      Γ ⊢ e : τ ! ε'
```

Effect **polymorphism** allows a function to be polymorphic in its effects:

```
map : ∀ε. (α →{ε} β) → List α →{ε} List β
```

The mapped function's effect is propagated to the list map.

### Koka: Row Polymorphism for Effects

[Leijen 2014; Leijen 2017]

**Koka** (Daan Leijen, Microsoft Research) is a research language that treats effects as **row types** — ordered lists of effect labels, subject to **row polymorphism**.

The effect `<div, exn | e>` is a row containing `div` and `exn` plus any other effects in the row variable `e`. A pure function has type `a → b` (syntactic sugar for `a → <> b`, the empty effect row).

The **effect inference** algorithm infers effect rows automatically using unification extended with row polymorphism. Koka's key property: **effect handlers** are first-class:

```koka
effect raise
  ctl raise(msg: string): a

fun safe-div(x: int, y: int): <raise> int
  if y == 0 then raise("division by zero") else x / y

fun example(): int
  with handler
    ctl raise(msg) -> 0   // handle raise by returning 0
  safe-div(10, 0)
```

The `with handler` block installs a handler for the `raise` effect; within the block, `raise` is handled by the continuation returning `0`. Effect handlers generalize exceptions, iterators, coroutines, and async/await in a uniform framework.

### Algebraic Effects and Handlers

[Plotkin and Power 2001; Plotkin and Pretnar 2009; Bauer and Pretnar 2015]

**Algebraic effects** decompose computational effects into **operations** (the primitives that *perform* effects) and **handlers** (the interpretations that *respond* to effects). This is the semantic foundation of Koka's design.

An algebraic effect `E` with operations `op₁, ..., opₙ` gives rise to a **free monad** `Free E`. An effect handler is a **fold** over this free monad, providing an algebra for each operation.

The `perform`/`handle` primitives (as in OCaml 5.x effects):

```ocaml
(* OCaml 5.x *)
effect Yield : int -> unit

let generator () =
  perform (Yield 1);
  perform (Yield 2);
  perform (Yield 3)

let collect () =
  match_with generator () {
    val_case = (fun () -> []);
    exn_case = (fun e -> raise e);
    eff_case = (fun (Yield v) k -> v :: continue k ())
  }
```

The `continue k ()` resumes the suspended computation at the `Yield` point with value `()`. This subsumes coroutines, generators, and cooperative threading.

**Relationship to continuations**: an effect handler captures a **delimited continuation** at the effect boundary. The handler's `k` is the continuation of the `perform` up to the enclosing `handle`. Algebraic effects are thus a structured form of first-class delimited continuations.

### Effect Inference vs. Annotation

- **Koka**: infers effects automatically; no annotations required except at module boundaries
- **F* / Dijkstra Monad**: requires full effect annotations; effect types are verified against a predicate transformer semantics (Hoare triples generalized)
- **OCaml 5.x effects**: untyped at present (as of OCaml 5.x series); a typed extension is being designed

### The Monad Connection

Haskell's **monads** are an alternative encoding of effects that does not require extending the type system: the effect is tracked in the return type `IO a`, `State s a`, `Either e a`, etc. The monadic approach is **composable** through monad transformers (e.g., `StateT s (ExceptT e IO) a` stacks state, exceptions, and I/O), but transformer stacks are notoriously opaque and have non-trivial performance overhead.

**Algebraic effects vs. monads**: the key expressivity difference is in the **handler scope**. A monadic computation commits to a specific monad instance at the point of use; an algebraic effect handler can intercept effects from a deeply nested call and resume the computation mid-way (non-local control). Specifically:

1. **Effect handlers are multishot**: the continuation `k` can be invoked zero, one, or many times by a handler. A monadic `>>=` binds exactly once.
2. **Effect polymorphism is natural**: a function `∀ε. (α →{ε} β) → List α →{ε} List β` works for any effect row `ε`. With monad transformers, this requires quantifying over the transformer stack.
3. **Performance**: algebraic effect implementations (Koka's evidence-passing, OCaml's fibers) achieve near-zero overhead for the common case of a single handler, competitive with direct-style code.

The mathematical relationship: every algebraic effect theory `E` gives rise to a monad `Free E` (the free monad). Every monad that is the free monad of some algebraic theory is captured by algebraic effects. Monads that are not free (e.g., continuations, the IO monad under linear constraints) require richer frameworks like **graded monads** or **parameterized monads** to capture within effect systems.

---

## 10. Refinement Types and Liquid Types

[Freeman and Pfenning 1991; Rondon, Kawaguchi, Jhala 2008]

### Refinement Types

A **refinement type** is a base type decorated with a **logical predicate** that constrains the values it classifies:

```
{x : Int | 0 ≤ x ∧ x < n}
```

This is "the type of integers `x` such that `0 ≤ x` and `x < n`." The predicate may refer to variables in scope (like `n`), making refinement types a form of dependent type where the dependency is restricted to first-order logic predicates.

**Subtyping via implication**: `{x:T|P(x)} <: {x:T|Q(x)}` when `∀x. P(x) ⟹ Q(x)` is valid in the underlying logic. This reduces subtype checking to **validity checking** in a decidable theory (e.g., linear arithmetic over integers — Presburger arithmetic).

```
S-Refine:
  Γ ⊢ ∀x. P(x) ⟹ Q(x)    (discharged to SMT solver)
  ──────────────────────────────────────────────────
  Γ ⊢ {x:T|P(x)} <: {x:T|Q(x)}
```

The function type `x:T → {y:S | Q(x,y)}` is a dependent refinement: the postcondition `Q` may mention the argument `x`.

### Liquid Types

[Rondon, Kawaguchi, Jhala 2008]

**Liquid types** (Logically Qualified Data Types) restrict the predicates to a **decidable sublanguage** called *liquid predicates*: conjunctions of **qualifiers** drawn from a fixed set of templates. The restriction enables **automatic inference** via abstract interpretation (a form of predicate abstraction).

The liquid fixpoint algorithm:

1. Assign each program variable a **refinement variable** `κ` (a set of qualifier templates)
2. Propagate constraints through the program's typing derivation, generating subtyping constraints between refinement variables
3. Solve the constraints by iterating **Houdini-style** elimination: start with all qualifiers, remove those that violate any constraint, repeat until fixpoint

The result is the **weakest safe refinement** for each variable — the most permissive type assignment consistent with the program's safety.

**Example.** For a vector access function:

```haskell
-- LiquidHaskell annotation
{-@ get :: n:Nat -> {v:[a] | len v = n} -> {i:Int | 0 <= i && i < n} -> a @-}
get n xs i = xs !! i
```

The annotation says: `get` takes a natural number `n`, a list of length `n`, and an index `i` with `0 ≤ i < n`, and returns an element. LiquidHaskell checks that every call site satisfies this contract and that the body respects it.

### LiquidHaskell

[Vazou et al. 2014; Jhala and Vazou 2020]

LiquidHaskell implements refinement types for Haskell as a **GHC plugin**. The system:

1. Takes Haskell source with refinement annotations (in comments `{-@ ... @-}`)
2. Elaborates the Core IR using the annotations
3. Generates **verification conditions** (VCs) in quantifier-free linear arithmetic + uninterpreted functions
4. Discharges VCs to an SMT solver (Z3)

LiquidHaskell can verify:
- **Vector bounds**: `Data.Vector.unsafeIndex` is safe at the call site
- **Termination**: a termination metric (a decreasing measure) is required for all recursive functions
- **Abstract refinements**: parametric polymorphism over predicates: `∀p. {v:a|p v} → {v:a|p v}` is a type-safe identity

**Abstract refinements** [Vazou et al. 2013] are a key innovation: refinement predicates can be abstracted over, enabling higher-order specifications:

```haskell
{-@ type OList a = [a]<{\x v -> x <= v}> @-}
-- OList a is a list whose elements are non-decreasing

{-@ insertSort :: Ord a => [a] -> OList a @-}
insertSort :: Ord a => [a] -> [a]
insertSort = foldr insert []
```

The refinement `{\x v -> x <= v}` is an abstract refinement parameter — it constrains adjacent list elements. LiquidHaskell instantiates such abstract refinements at use sites and passes them to Z3 as uninterpreted function constraints.

**The interaction with Haskell's laziness**: LiquidHaskell must handle the fact that Haskell is lazy and refinements are eager predicates. A value `x : {v:Int | v > 0}` might be a thunk that, when forced, throws an exception rather than producing a positive integer. LiquidHaskell addresses this by requiring that refinements only constrain values that are **total** (provably terminate), using the termination checker to ensure this.

### F* and Low*

[Swamy et al. 2016; Protzenko et al. 2017]

**F\*** (F-star) is a dependently-typed, effectful programming language designed for **program verification**. Its type system combines:
- Dependent types (Π, Σ)
- Refinement types `{x:t | φ}` with SMT-discharged predicates
- Effect system based on the **Dijkstra monad** framework: effects are specified by predicate transformers (WP calculus), making specifications Hoare-triple style
- **Monotonic state**: a ghost state that can only be extended, enabling incremental proofs of stateful programs

**Low\*** is a subset of F* that compiles to C via the **KreMLin** tool. Low* restricts to a C-compatible memory model (stack and heap, no GC) and produces verified C code.

Applications:
- **HACL\***: a verified cryptographic library implementing Chacha20, Poly1305, BLAKE2, Curve25519, and others
- **EverCrypt**: a provider-agnostic cryptographic API with verified functional correctness and memory safety
- **miTLS**: a verified implementation of TLS 1.2 and 1.3

The guarantee: the F* types are erased to C types, but the C code produced by KreMLin is provably safe by the F* verification — the C code has no undefined behavior for the specified input conditions.

---

## 11. Gradual Typing

[Siek and Taha 2006; Garcia et al. 2016]

### The Dynamic Type

**Gradual typing** [Siek and Taha 2006] integrates static and dynamic typing in a single language by introducing the **dynamic type** `?` (also written `Dyn` or `unknown`). A term of type `?` has its type checked at runtime rather than compile time.

The key idea: statically-typed code and dynamically-typed code can **interoperate** through **implicit casts**. The cast `⟨τ ← ?⟩` checks at runtime that a `?`-typed value actually has type `τ`; if it does not, a dynamic type error is raised.

### The Consistency Relation

The fundamental relation of gradual typing is **consistency** `τ ~ σ`, defined inductively:

```
? ~ τ         for all τ
τ ~ ?         for all τ
τ ~ τ         (reflexivity)
τ₁~σ₁, τ₂~σ₂   →   τ₁→τ₂ ~ σ₁→σ₂
```

Consistency is *not* transitive: `Int ~ ?` and `? ~ Bool` but `Int ≁ Bool`. This distinguishes consistency from subtyping.

The gradual type system replaces the standard typing rule

```
T-App:  Γ ⊢ f : τ→σ    Γ ⊢ e : τ
```

with:

```
GT-App:  Γ ⊢ f : τ→σ    Γ ⊢ e : τ'    τ' ~ τ
```

If `τ' = ?`, the cast `⟨τ ← ?⟩` is inserted around `e` and checked at runtime.

### The Gradual Guarantee

The **gradual guarantee** [Siek et al. 2015] states two properties:

1. **Static gradual guarantee**: if `e` is well-typed without any `?`, replacing any type annotation with `?` preserves well-typedness
2. **Dynamic gradual guarantee**: if `e : τ` with some `?` annotations and we refine those annotations to more precise types, the program's behavior does not change (no new errors are introduced by static typing)

The gradual guarantee is the key correctness criterion: gradual typing should not break programs that were working, and adding type annotations should not introduce failures. TypeScript's `any` type satisfies the static guarantee but violates the dynamic guarantee (TypeScript is **optionally typed**, not gradually typed in the formal sense — it performs no runtime checks).

### The Blame Calculus

[Wadler and Findler 2009]

The **blame calculus** is the operational semantics framework for gradually typed languages. When a cast fails at runtime, the blame calculus assigns **blame** to one of two parties:

- **Positive blame** (`⊢p`): the term that produced the value is at fault — it claimed the value had a type it did not have
- **Negative blame** (`⊢n`): the context that consumed the value is at fault — it imposed a type constraint the value could not satisfy

The principal theorem:

> **Blame Safety Theorem** [Wadler and Findler 2009]. In the blame calculus, blame is always assigned to the **untyped** side of a typed/untyped boundary.

This means that a **fully annotated** program (no `?` types) never raises a blame error from the typed side — all failures are traceable to untyped code that violated the contract. This is the formal justification for the guarantees that Typed Racket's contract system provides.

The cast `⟨τ ← σ⟩^p M` (read: "cast `M` from `σ` to `τ`, with positive blame label `p`") is defined operationally:

```
⟨τ ← τ⟩^p v  →  v                                   (identity cast)
⟨? ← τ⟩^p v  →  v                                   (demotion to ?)
⟨τ ← ?⟩^p v  →  check v : τ or raise blame(+p)      (promotion from ?)
⟨S₁→S₂ ← T₁→T₂⟩^p f  →  λx. ⟨S₂←T₂⟩^p (f ⟨T₁←S₁⟩^p̄ x)
```

For the function cast, note the **polarity inversion**: `⟨T₁ ← S₁⟩^p̄` uses the *negated* blame label `p̄` for the argument — because the callee's domain check is a constraint imposed by the caller, blame for a domain failure lies with the *caller* (negative position), not the callee.

### Practical Gradual Type Systems

- **TypeScript** (`any`): static type checking with `any` as an escape hatch; no runtime casts; optional typing rather than gradual typing in the formal sense — TypeScript erases types entirely at runtime
- **Python type hints + mypy/pyright**: annotations are optional; mypy checks annotated code statically; no runtime enforcement unless explicitly added (via `isinstance` or `beartype`)
- **Typed Racket**: full gradual typing with **contracts** that enforce types at the boundary between typed and untyped modules; the most faithful implementation of the Siek-Taha model; blame is tracked and reported with module-level precision
- **Reticulated Python** [Vitousek et al. 2014]: gradually typed Python with runtime type tags; demonstrates the performance cost of pervasive runtime checking (the **gradual guarantee penalty**) — in the worst case, inserting `?` everywhere makes a program 10× slower due to pervasive tag checks

**The performance problem.** The fundamental tension in gradual typing: full enforcement of the gradual guarantee requires inserting casts at *every* typed/untyped boundary. For higher-order functions, a cast on a function type wraps it in a thunk that checks every call and return. This is known as the **space- and time-safety cost of gradual typing**. Research directions include:

- **Transient gradual typing** [Vitousek et al. 2017]: check only at the immediate consumption point (no higher-order wrapping); faster but does not track blame through higher-order functions
- **Concrete semantics** [Greenman and Felleisen 2018]: a taxonomy of gradual type system "flavors" ranging from purely nominal (no runtime checks) to fully enforced, with formal characterization of what each guarantees

---

## 12. Session Types

[Honda 1993; Honda, Vasconcelos, Kubo 1998; Gay and Hole 2005]

### Session Types Model Communication Protocols

A **session type** describes the sequence of communications that occur on a channel. The basic constructors are:

- `!T.S`: **send** a value of type `T`, then continue with session type `S`
- `?T.S`: **receive** a value of type `T`, then continue with session type `S`
- `⊕{l₁:S₁, ..., lₙ:Sₙ}`: **internal choice** — select one of the labeled alternatives
- `&{l₁:S₁, ..., lₙ:Sₙ}`: **external choice** — offer all labeled alternatives; the peer selects
- `end`: the session is complete; the channel must be closed

**Example.** A simple calculator server protocol:

```
Server : &{ Add: ?Int.?Int.!Int.end,
            Mul: ?Int.?Int.!Int.end }

Client : ⊕{ Add: !Int.!Int.?Int.end,
             Mul: !Int.!Int.?Int.end }
```

The server offers a choice; the client makes a selection. Each branch: receive two integers, send one integer, close.

### Duality

The **dual** of a session type describes the other end of the channel:

```
dual(!T.S)  = ?T.dual(S)
dual(?T.S)  = !T.dual(S)
dual(⊕{lᵢ:Sᵢ}) = &{lᵢ:dual(Sᵢ)}
dual(&{lᵢ:Sᵢ}) = ⊕{lᵢ:dual(Sᵢ)}
dual(end)   = end
```

A channel connecting two processes is well-typed if and only if one end has a session type `S` and the other has `dual(S)`. This is the type-level enforcement of protocol compatibility.

### Linear Session Types

Session types are naturally **linear**: each channel endpoint must be used **exactly once per step** — you cannot use a channel twice at the same communication point (which would confuse the protocol) or discard it (which would leave the peer waiting forever).

The combination of session types and linear types gives:
- **Protocol adherence**: the sequence of communications matches the declared session type
- **Deadlock freedom** (in some restricted calculi): if all channels are used linearly, the communication graph is acyclic and deadlock is ruled out

### Dependent Session Types

**Dependent session types** allow the session type to depend on communicated values:

```
!n:Nat. ?(Vec Int n). end
```

"Send a natural number `n`, then receive a vector of integers of length `n`, then close." The type of what is received depends on the value that was sent — a full dependent type in the protocol.

### Session Types and the π-Calculus

Session types are deeply connected to the **π-calculus** [Milner, Parrow, Walker 1992], which is the process calculus for modeling mobile communicating systems. In the π-calculus, processes communicate by passing channel names along channels — enabling dynamic network topologies. Session types were introduced by Honda [1993] to type-check π-calculus processes:

- A **channel** `c` has a session type `S` on one endpoint and `dual(S)` on the other
- A **process** using `c` at session type `!T.S` must **send** a value of type `T` and then continue with `c` at type `S`
- **Session type safety** (subject reduction for π-calculus): well-typed processes do not get stuck; communications are type-consistent

The **binary session types** of Honda (1993) handle two-party protocols. **Multiparty session types** [Honda, Yoshida, Carbone 2008] generalize to *n*-party protocols using **global types** that describe the entire communication from a bird's-eye view, projected down to local types for each participant.

A global type for a three-party protocol:

```
G = A → B : Int . B → C : String . C → A : Bool . end
```

"A sends an Int to B; B sends a String to C; C sends a Bool to A; done." Projecting onto each participant gives the local session type for that party — and the projection algorithm checks consistency of the global specification.

### The Curry-Howard Correspondence for Session Types

[Caires and Pfenning 2010; Wadler 2012]

A profound connection exists between session types and **linear logic**: the Curry-Howard correspondence extends to the π-calculus and session types.

| Linear Logic | Session Types |
|-------------|--------------|
| `A ⊗ B` | `!A.S` (send `A`, continue with `S`) |
| `A ⅋ B` | `?A.S` (receive `A`, continue with `S`) |
| `A ⊕ B` | `⊕{l₁:S₁, l₂:S₂}` (internal choice) |
| `A & B` | `&{l₁:S₁, l₂:S₂}` (external choice) |
| `!A` | server: unlimited use |
| `1` | `end` (empty session) |

Under this correspondence, a **proof of a linear logic sequent** `A₁, ..., Aₙ ⊢ B` corresponds to a **process** that provides service `B` given resources `A₁, ..., Aₙ`. Session-typed processes are proofs; communication steps are proof reductions. **Deadlock freedom** follows from the cut-elimination theorem for linear logic.

### Implementation

- **GV (Good Variation) calculus** [Lindley and Morris 2015]: a session-typed lambda calculus with proof-theoretic foundations in classical linear logic; formally proven deadlock-free
- **Links** language [Cooper et al. 2006]: a web programming language with session types for safe client-server communication; rows for effects
- **Rust session types** via the `session-types` crate: session types encoded using Rust's type system; the channel type is `Chan<E, S>` where `S` is the session type; linearity enforced by Rust's affine type system and ownership
- **Ferrite** [Chen and Balzer 2022]: a Rust library implementing shared session types via Iris-style logical relations, enabling safe concurrent services beyond simple two-party protocols

---

## 13. Dependent Types for Systems Programming: ATS and F*/Low*

### The Gap Between Theory and Systems

Dependent and refinement type systems are most useful when they can be applied to **systems programming** — code that manipulates memory directly, interfaces with hardware, and has hard safety requirements. The challenge is that dependent type systems are designed for functional settings; adapting them to imperative, memory-managing systems code requires substantial engineering.

### ATS: Applied Type System

[Xi 2007; Xi and Pfenning 1999; ats-lang.org]

**ATS** (Applied Type System) is a programming language with C-like performance and dependent types for static verification of memory safety and resource usage. ATS uses:

- **Dependent types** for index verification: array bounds, pointer arithmetic
- **Linear types** for resource management: each proof of ownership is linear; it must be consumed exactly once (by a deallocation or transfer)
- **Proof terms alongside programs**: the language distinguishes `proof` code (erased at runtime) from `program` code (executed); proofs are linear and cannot escape into programs

The canonical example is safe array access:

```ats
fun array_get {n:int}{i:int | 0 <= i; i < n}
    (arr: array(a, n), i: int(i)): a = array_get_elt(arr, i)
```

The signature reads: given a proof that `0 ≤ i` and `i < n` (encoded in the index refinement `{i:int | 0 <= i; i < n}`), `array_get` accesses element `i` of an array of size `n` safely.

ATS programs are compiled to C and then to LLVM IR. The dependent type guarantees are **erased at the C level** but the C code they produce is sound — the compiler generates no bounds checks in the output because the static verification has already certified safety.

**Linear proof terms.** ATS's linear proofs model resources:

```ats
// array_v is a linear proof of array ownership
prval pf = array_v_split(arr_v, i)  // split array proof at index i
// pf is consumed; produces two sub-proofs
val x = !p  // dereference; requires ownership proof
```

The `prval` declarations introduce linear proof variables; attempting to use them twice is a type error.

### F* with Low*: Verified C via KreMLin

[Protzenko et al. 2017; Bhargavan et al. 2017]

The **Low\*** programming model in F* targets **stack and heap memory** explicitly, using a memory model that matches C's. The key types:

- `Buffer.t a`: a heap-allocated buffer of type `a`, analogous to `a*` in C
- `Stack (a:Type) (pre:mem → Type0) (post:mem → a → mem → Type0)`: a computation on stack with Hoare-triple specification
- `HST.ST a pre post`: heap-state computations with precondition `pre` and postcondition `post`

**KreMLin extraction** translates Low* F* programs to C by:
1. Erasing all F* types (refinements, proofs, ghost computations) to C types
2. Translating `match` to `switch`, `let/in` to variable declarations, recursive functions to loops where possible
3. Generating `#include <stdint.h>` types for fixed-width integers

The safety guarantee flows from the F* type derivation: if F* accepts the program, the generated C code satisfies the specification for all inputs satisfying the precondition.

**HACL\*** (High-Assurance Cryptographic Library): implements Chacha20-Poly1305, SHA2, BLAKE2, HMAC, Curve25519, P-256, Ed25519, and other algorithms in Low*. Every function is verified for:
- **Functional correctness**: the computed value equals the mathematical specification
- **Memory safety**: no buffer overflows, use-after-free, or double frees
- **Secret independence** (for some algorithms): execution does not branch on secret data (constant-time properties)

**EverCrypt** wraps HACL* in an agile API with **multiplexing**: at startup, EverCrypt selects the best implementation (hardware AES-NI, AVX2 BLAKE2, etc.) using a verified dispatch table. The dispatch is verified to preserve the functional specification.

### The Compilation Path to LLVM

Both ATS and Low* ultimately compile through C to LLVM IR:

```
ATS source      →  C code (by atscc2c)  →  LLVM IR (by clang)
F*/Low* source  →  C code (by KreMLin)  →  LLVM IR (by clang)
```

At the LLVM IR level, the dependent type guarantees are completely erased. LLVM IR's type system (Chapter 15, Chapter 17) is flat and unrefined — there are no array-length invariants in `[N x i32]` beyond the literal count `N`. The memory safety that ATS and F* provide statically is not re-checked at the IR level; LLVM's optimizers assume the C code is well-formed and optimize accordingly (including removing `__builtin_expect`-annotated branches that the verifier proved unreachable).

This **stratification** is a deliberate design: expensive dependent-type checking happens once at the source level; the compiler backend receives provably safe code and may optimize aggressively without redundant runtime checks. Chapter 15 discusses how MLIR's type system re-exposes some of these invariants (e.g., shaped tensor types) in the IR itself, allowing shape-aware optimization passes that could not safely operate on flat LLVM IR.

### The Type-Erasure Spectrum

The progression from source to binary can be viewed as a **series of type erasure steps**, each erasing a layer of type information in exchange for simpler code:

| Level | Type information retained | What is erased |
|-------|--------------------------|----------------|
| F*/ATS source | Full dependent + refinement types | Nothing yet |
| Generated C | C types (`int`, `uint8_t *`, struct fields) | Proofs, refinement predicates, size indices |
| LLVM IR | First-class types: `i32`, `ptr`, struct layouts | C type qualifiers, restrict, VLAs |
| Machine code | Register widths, calling convention | All types; only bits remain |

At each erasure step, invariants that were enforced by the richer type system must either be maintained by the generated code structurally (e.g., not generating out-of-bounds accesses) or checked by a weaker mechanism (e.g., LLVM's `!range` metadata, AddressSanitizer at test time).

The central insight connecting this chapter to the rest of the book: **type systems are erasure-invariant in what they *ensure* but erased-specific in what they *express***. A well-typed ATS program is safe at all levels; but the safety guarantee is expressed in the ATS type system, not in LLVM's. To re-express and exploit invariants below the C level, MLIR's richer type system (shaped memrefs, ranked tensors, protocol-typed channels) is the current state of the art — it brings some of what this chapter formalizes down into the compiler IR, enabling shape-aware code generation, kernel fusion, and verified lowering at the mid-level.

---

## 14. Chapter Summary

This chapter surveyed the principal advanced type systems that compiler engineers encounter when designing compilers for modern languages or analyzing the guarantees that source-level type systems provide.

**Key takeaways:**

- **Dependent types** (Π and Σ types) allow types to depend on values, enabling static verification of invariants like vector lengths, array bounds, and protocol states. The return type of `replicate : Π(n:Nat). A → Vec(A,n)` carries a proof obligation that the compiler checks once.

- **Martin-Löf Type Theory** unifies programming and mathematics through the propositions-as-types, proofs-as-programs correspondence. The four judgments (type formation, term formation, definitional equality for types and terms) and the BHK interpretation ground all of constructive mathematics computationally.

- **The λ-cube** organizes type theories along three axes of dependency. The Calculus of Constructions (CoC) sits at the top corner; Coq's CIC adds inductive types and the positivity condition; Lean 4 adds proof irrelevance, universe polymorphism, and quotient types.

- **Subtyping** comes in width (more fields is a subtype), depth (covariant within fields — unsound for mutable fields), structural (Go, TypeScript), and nominal (Java, C++) flavors. The S-Arrow rule — contravariant in the argument, covariant in the result — is derived from substitutability and is the correct rule for function types.

- **System F<:**, bounded quantification, extends System F with bounded type variables. Full F<: is undecidable; Java generics implement a decidable approximation. Wildcard types correspond to existential types.

- **Linear, affine, and relevant types** restrict the structural rules of weakening and contraction, enabling resource-precise type checking. Girard's linear logic is the logical foundation; affine types model ownership; linear types model resources that must be consumed.

- **Rust's borrow checker** is formally an affine type system extended with region-typed shared and exclusive references. The NLL analysis computes minimal lifetimes by dataflow. RustBelt (Jung et al. 2018) proves safety for a realistic Rust fragment using Iris, a concurrent separation logic.

- **Uniqueness types** (Clean) guarantee single-reference aliasing, enabling safe destructive update. **Region types** (Tofte-Talpin) assign allocations to named pools with static lifetime bounds; Rust's lifetimes are a flow-sensitive variant.

- **Effect systems** (Koka, F*) extend function types with effect annotations describing allowed side effects. Algebraic effects and handlers (OCaml 5.x, Eff, Frank) generalize exceptions, coroutines, and async/await uniformly through first-class continuations.

- **Refinement types** add SMT-dischargeable predicates to base types. **Liquid types** restrict to a decidable predicate sublanguage and enable automatic inference via abstract interpretation. LiquidHaskell, F*/Low*, and HACL* demonstrate that safety-critical systems code can be fully verified at scale.

- **Gradual typing** integrates static and dynamic typing via the consistency relation and the dynamic type `?`. The gradual guarantee ensures that adding type annotations cannot break correct programs. TypeScript, Python type hints, and Typed Racket occupy different points on the spectrum.

- **Session types** model communication protocols as types. Duality ensures that the two ends of a channel are always compatible. Linear session types rule out protocol violations and (in restricted calculi) deadlock.

- **ATS and F*/Low*** apply dependent and affine types to systems programming, compiling through C to LLVM IR. The static guarantees are erased at the IR level; the generated C is provably safe, and LLVM can optimize without redundant runtime checks.

---

### References

- **[TAPL]** Pierce, B. C. *Types and Programming Languages*. MIT Press, 2002. §§15–17 (Subtyping), §§23–24 (Universal Types), §§26–28 (Bounded Quantification), §29 (Dependent Types)
- **[ATTAPL]** Pierce, B. C., ed. *Advanced Topics in Types and Programming Languages*. MIT Press, 2005. Ch. 1 (Linear Types), Ch. 3 (Region Types)
- **[PFPL]** Harper, R. *Practical Foundations for Programming Languages*, 2nd ed. Cambridge, 2016. §§36–44 (Dependent Types), §§25, 48 (Curry-Howard)
- **[Martin-Löf 1984]** Martin-Löf, P. *Intuitionistic Type Theory*. Bibliopolis, 1984
- **[Reynolds 1974]** Reynolds, J. C. "Towards a Theory of Type Structure." *LNCS* 19, 1974
- **[Coquand and Huet 1988]** Coquand, T. and Huet, G. "The Calculus of Constructions." *Information and Computation* 76(2–3), 1988
- **[Barendregt 1991]** Barendregt, H. P. "Introduction to Generalized Type Systems." *Journal of Functional Programming* 1(2), 1991
- **[Girard 1987]** Girard, J.-Y. "Linear Logic." *Theoretical Computer Science* 50(1), 1987
- **[Jung et al. RustBelt 2018]** Jung, R., Jourdan, J.-H., Krebbers, R., Dreyer, D. "RustBelt: Securing the Foundations of the Rust Programming Language." *POPL* 2018
- **[Matsakis 2018]** Matsakis, N. D. "Non-Lexical Lifetimes." *Rust RFC 2094*, 2018
- **[Tofte and Talpin 1994]** Tofte, M. and Talpin, J.-P. "Implementation of the Typed Call-by-Value λ-Calculus using a Stack of Regions." *POPL* 1994
- **[Siek and Taha 2006]** Siek, J. G. and Taha, W. "Gradual Typing for Functional Languages." *Scheme Workshop* 2006
- **[Honda 1993]** Honda, K. "Types for Dyadic Interaction." *CONCUR* 1993
- **[Rondon et al. 2008]** Rondon, P. M., Kawaguchi, M., Jhala, R. "Liquid Types." *PLDI* 2008
- **[Protzenko et al. 2017]** Protzenko, J. et al. "Verified Low-Level Programming Embedded in F*." *ICFP* 2017
- **[Xi 2007]** Xi, H. "Applied Type System (Extended Abstract)." *TYPES* 2003, published 2007
- **[Pierce and Steffen 1994]** Pierce, B. C. and Steffen, B. "Higher-Order Subtyping." *ICALP* 1994
- **[Leijen 2014]** Leijen, D. "Koka: Programming with Row Polymorphic Effect Types." *MSFP* 2014
- **[Marino and Millstein 2009]** Marino, D. and Millstein, T. "A Generic Type-and-Effect System." *TLDI* 2009
