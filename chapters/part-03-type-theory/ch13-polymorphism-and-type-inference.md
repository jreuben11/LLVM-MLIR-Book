# Chapter 13 — Polymorphism and Type Inference

*Part III — Type Theory*

The Simply Typed Lambda Calculus developed in Chapter 12 is safe and strongly normalizing, but it suffers a crippling expressivity problem: every function must be typed for exactly one argument type. The identity function must be written once as `id_int : Int→Int` and again as `id_bool : Bool→Bool`, each a distinct term with a distinct type. Real programming demands that a single identity function work over all types uniformly. This chapter develops the theoretical machinery that makes that possible.

We cover parametric polymorphism and the free theorems it entails (Reynolds's parametricity), System F (the second-order lambda calculus that gives parametric polymorphism its denotation), existential types and their role as the type-theoretic account of abstract data types, and the Hindley-Milner type system — the fragment of System F that admits a decidable, complete type-inference algorithm. Algorithm W is derived in full with a worked trace. The chapter then examines the value restriction, principal types, bidirectional type checking, row polymorphism, type classes, and the implementation continuum from Haskell dictionaries through Rust monomorphization to C++ template instantiation, closing with the connection to LLVM IR.

Chapter 12 covered STLC and the Curry-Howard correspondence. Chapter 14 extends the story to dependent types, linear types, and refinement types. Chapter 15 connects this theoretical apparatus to the concrete type systems of LLVM IR and MLIR.

**Why this matters for compiler engineers.** Every production language with generics or templates must solve the problems this chapter analyzes formally. C++ template instantiation, Rust's monomorphization, Haskell's dictionary passing, and Java's type erasure are all engineering answers to the same theoretical question: how do you implement a type `∀α. τ`? Algorithm W is the core of every ML-family type checker; understanding its cost model (near-linear in practice, exponential in the worst case) is necessary to understand why certain Haskell and OCaml programs are slow to compile. The value restriction is why `let r = ref []` behaves differently in OCaml versus Haskell. Bidirectional type checking underpins GHC's extended type system, Rust's type checker, and Scala's Dotty. These are not academic curiosities — they are load-bearing parts of the compilers this book analyzes.

---

## 1. Parametric Polymorphism and Free Theorems

### The Monomorphism Problem in STLC

In STLC, types are assigned structurally: every term has a single, fixed type. The identity function written as `λx. x` is not typeable in a vacuum — we must choose a type for `x`. We can derive `λx:Int. x : Int→Int` or `λx:Bool. x : Bool→Bool`, but no single typing covers both. In a language with *n* base types, a programmer must write *n* copies of the identity. Polymorphism is the mechanism that collapses these infinitely many copies into one.

**Parametric polymorphism** [TAPL §23.1] means that a single term `id` has type `∀α. α→α`: for any type `τ`, `id[τ]` has type `τ→τ`. The key word is *uniformly* — the implementation does not inspect the type. It is the same machine code whether `α` is instantiated to `Int`, `Bool`, or `List Int`. This uniformity is what enables **free theorems**.

### Reynolds's Parametricity

Reynolds's **abstraction theorem** [Reynolds 1983; Wadler 1989] states: a well-typed polymorphic function must behave consistently with every type abstraction it is given. More precisely, a function of type `∀α. τ` treats `α` as an abstract type — it cannot branch on the runtime structure of an `α` value.

The formal statement uses **logical relations**. For each type `τ`, define a relation `⟦τ⟧ ⊆ Val × Val` interpreting what it means for two values to be related at type `τ`. For a type variable `α`, the relation is a parameter — any relation `R ⊆ A × B` may be chosen. The abstraction theorem then states:

> **Abstraction Theorem (Reynolds 1983).** If `⊢ M : τ` and `η` is a relational interpretation of type variables, then `(M, M) ∈ ⟦τ⟧_η`.

In other words, every well-typed term is related to itself under every relational interpretation. This is the fixed point that forces uniformity.

**Deriving the free theorem for `∀α. α→α`.** Let `f : ∀α. α→α` be any closed term. Pick two sets `A` and `B` and any relation `R ⊆ A × B`. Interpreting `α` with `R`, the abstraction theorem requires:

```
(f_A, f_B) ∈ ⟦α→α⟧_R  =  { (g, h) | ∀(a,b)∈R. (g a, h b) ∈ R }
```

That is: for all `(a, b) ∈ R`, we have `(f_A a, f_B b) ∈ R`. Now specialize: take `A = B = X` and `R = { (x, f_X x) | x ∈ X }` (the graph of `f_X`). The condition becomes: for all `x`, `(f_X x, f_X (f_X x)) ∈ R`, meaning `f_X (f_X x) = f_X x` — i.e., `f_X` is idempotent. Furthermore, taking `R = Id_X = { (x,x) | x ∈ X }`, the condition says `f_X x = f_X x` for all `x`, which is trivially true, but taking `R` as any single-element relation `{(a, b)}` forces `f_A a = a` and `f_B b = b`. Combining these: **the only possible behavior of `f : ∀α. α→α` is the identity**.

**The `map` naturality square.** For `map : ∀α β. (α→β) → [α] → [β]`, the abstraction theorem, instantiated with types `A`, `B`, `C`, `D` and functions `h : A→C` and `g : B→D`, produces the naturality condition:

```
(map g) ∘ (map h)  =  map (g ∘ h)    [when h : A→B, g : B→C]
```

This is the functor composition law for `map` — the central axiom of the `Functor` type class — derived entirely from the type, without inspecting the implementation [Wadler 1989]. The naturality square commutes:

```
[A] ─── map h ──→ [B]
 │                  │
map(g∘h)          map g
 │                  │
 ↓                  ↓
[C] ──────────── [C]
```

The general pattern is: given a function's polymorphic type, the parametricity theorem produces an equation that every implementation must satisfy. Compiler writers exploit this: a transformation that changes the representation of `α` is provably safe if the function is truly parametric. This is the formal justification behind whole-program type-erasure optimizations.

### Universal Types vs. Bounded Quantification

In `∀α. τ`, the variable `α` ranges over *all* types without restriction. This is **unbounded universal quantification**, the core of System F (Section 2 below). The identity function `id : ∀α. α→α` works for absolutely any type — there is no constraint on `α` whatsoever. This is what makes free theorems possible: because `α` is completely unconstrained, the only operations available on a value of type `α` are those that work for any type, which forces the implementation to behave uniformly.

**Bounded quantification** `∀α<:τ'. τ` restricts the range of `α` to subtypes of `τ'`. A function `∀α<:Animal. α→α` works for any type that is a subtype of `Animal`, but may use methods specific to `Animal`. This is the foundation of **System F<:** [TAPL §26], the calculus underlying Java generics, Scala's variance annotations, and C++20 concepts. Bounded quantification partially restores the ability to use interface methods on the type variable, at the cost of breaking the full parametricity theorem (free theorems hold only modulo the bound).

Chapter 14 develops System F<: in depth, along with its interaction with subtype polymorphism, recursive types, and dependent types. The present chapter confines itself to the unbounded case (System F and HM), where parametricity holds in its strongest form.

---

## 2. System F: The Second-Order Lambda Calculus

System F, introduced independently by Girard [1972] and Reynolds [1974], extends STLC with type-level abstraction [TAPL §23].

### Syntax

```
Types    τ  ::=  α                   (type variable)
              |  τ₁ → τ₂             (function type)
              |  ∀α. τ               (universal type)

Terms    M  ::=  x                   (variable)
              |  λx:τ. M             (abstraction)
              |  M N                 (application)
              |  Λα. M               (type abstraction)
              |  M [τ]               (type application)
```

The new constructs are the **type abstraction** `Λα. M` and **type application** `M [τ]`. Type variables `α, β, γ` range over types; term variables `x, y, z` range over terms as before.

### Typing Rules

The typing context Γ now contains both term bindings `x:τ` and type variable declarations `α`. The standard STLC rules carry over; two rules are new [TAPL §23.3]:

```
          Γ, α ⊢ M : τ
T-TAbs  ─────────────────────────
          Γ ⊢ Λα. M : ∀α. τ


          Γ ⊢ M : ∀α. τ
T-TApp  ──────────────────────────
          Γ ⊢ M [τ'] : τ[α ↦ τ']
```

**T-TAbs** adds `α` as an abstract type variable to the context and checks the body; the resulting type closes over `α`. **T-TApp** instantiates a universally quantified type by substituting `τ'` for `α` throughout `τ`.

The reduction rules add:

```
(Λα. M) [τ]  →β  M [α ↦ τ]        (type beta-reduction)
```

### The Identity in System F

The polymorphic identity is:

```
id  =  Λα. λx:α. x
```

Type checking: starting from empty context, T-TAbs with `α` extends the context to `{α}`. Inside, T-Abs derives `λx:α. x : α→α` (since `x:α` and `x` has type `α`). So by T-TAbs, `id : ∀α. α→α`. Application: `id [Int] 42` first type-applies to get `id [Int] : Int→Int`, then applies to `42 : Int` to yield `42 : Int`.

### Church Encodings in System F

System F is computationally universal; every data type can be encoded via its eliminator (the Church encoding) [TAPL §23.4].

**Church booleans:**
```
Bool  =  ∀α. α → α → α

true   =  Λα. λt:α. λf:α. t
false  =  Λα. λt:α. λf:α. f
if     =  λb:Bool. Λα. λt:α. λf:α. b [α] t f
```

**Church naturals.** The representation of `n` is the *n*-fold iteration function:

```
Nat   =  ∀α. (α→α) → α → α

zero  =  Λα. λs:α→α. λz:α. z
one   =  Λα. λs:α→α. λz:α. s z
succ  =  λn:Nat. Λα. λs:α→α. λz:α. s (n [α] s z)
add   =  λm:Nat. λn:Nat. Λα. λs:α→α. λz:α. m [α] s (n [α] s z)
```

To check: `add one one` reduces to `Λα. λs. λz. s (s z)`, which is `two`. The type `∀α. (α→α)→α→α` says: given any type `α`, a step function `s:α→α`, and a base `z:α`, produce an element of `α` by applying `s` exactly `n` times to `z`. This is the Peano induction principle as a type.

**Church pairs:**
```
Pair τ₁ τ₂  =  ∀α. (τ₁→τ₂→α) → α

pair  =  λx:τ₁. λy:τ₂. Λα. λf:τ₁→τ₂→α. f x y
fst   =  λp:Pair τ₁ τ₂. p [τ₁] (λx:τ₁. λy:τ₂. x)
snd   =  λp:Pair τ₁ τ₂. p [τ₂] (λx:τ₁. λy:τ₂. y)
```

The power of these encodings is that they require no base types or built-in elimination forms — System F is its own standard library.

### Strong Normalization

**Theorem (Girard 1972):** Every well-typed term of System F is strongly normalizing — all reduction sequences terminate.

The proof uses **reducibility candidates** (also called *logical predicates* or *saturated sets*) [TAPL §12.1; Girard 1972]. A **reducibility candidate** at type `τ` is a set `RED(τ) ⊆ Terms` satisfying three conditions (collectively called the CR conditions):

```
CR1:  if M ∈ RED(τ), then M ∈ SN  (all elements are strongly normalizing)
CR2:  if M ∈ RED(τ) and M →β M', then M' ∈ RED(τ)  (closure under reduction)
CR3:  if M is neutral (not a redex) and
          ∀M'. (M →β M') ⟹ M' ∈ RED(τ),
      then M ∈ RED(τ)  (closure under reverse reduction from neutral terms)
```

The candidates are defined by induction on the structure of types:

```
RED(B)        =  SN  (for base types B, all SN terms qualify)

RED(τ₁→τ₂)   =  { M | ∀N ∈ RED(τ₁). M N ∈ RED(τ₂) }

RED(∀α. τ)   =  { M | ∀ candidate P satisfying CR1–CR3.
                       M [τ'] ∈ RED(τ[α ↦ τ']/P) for all τ' }
```

The definition of `RED(∀α. τ)` quantifies over all *candidates* P (not just all types τ'), which is the impredicative step: the candidates form a collection indexed not just by types but by semantic predicates satisfying CR1–CR3. This is where the proof demands care — an informal argument that "quantify over all types" would be circular because `∀α. τ` itself is a type.

Three lemmas drive the proof:

1. **All candidates satisfy CR1–CR3.** Verified by structural induction on the type, checking CR1–CR3 for each case. For `RED(τ₁→τ₂)`, SN of applications follows from the SN of the function and argument.

2. **Substitution lemma.** If `M ∈ RED(τ)` and `σ` is a type substitution, then `M ∈ RED(σ(τ))` (properly stated via the candidate interpretation under `σ`).

3. **Fundamental lemma.** If `Γ ⊢ M : τ`, then for every substitution `γ` mapping each `x:σ ∈ Γ` to a term in `RED(σ)`, we have `γ(M) ∈ RED(τ)`.

Strong normalization is immediate: take the identity substitution; by the fundamental lemma, every closed well-typed term is in `RED(τ)` for its type; by CR1, it is strongly normalizing.

The proof's difficulty lies entirely in the `RED(∀α. τ)` case of the fundamental lemma: when `M = Λα. N` is a type abstraction, one must show `(Λα. N)[τ'] ∈ RED(τ[α↦τ']/P)` for all candidates `P`. This reduces to showing `N[α↦τ'] ∈ RED(τ[α↦τ']/P)` by the type beta rule, which holds by induction — but only if the induction hypothesis is stated over candidates rather than types. This is why the proof appeared in Girard's doctoral thesis [Girard 1972] as a 50-page argument and required significant subsequent work (Tait, Girard, Gallier) to formalize cleanly.

### Impredicativity and the Rank Hierarchy

System F is **impredicative**: in `∀α. τ`, the variable `α` ranges over all types, including types of the form `∀β. τ'`. In particular, instantiating `α` in `∀α. τ` with `∀β. τ'` is well-typed. This is the source of System F's expressive power (it can encode all of Peano arithmetic and beyond), but it also makes type *checking* undecidable in the presence of type inference [Wells 1999].

The **rank** of a type measures how deeply universally quantified variables appear on the left side of arrows:

```
rank-0:  no quantifiers
rank-1:  ∀α₁...∀αn. τ   where τ contains no ∀  (prenex / Hindley-Milner)
rank-2:  quantifiers may appear one level into arrow arguments
rank-k:  ∀ appears at argument depth ≤ k-1
```

Type inference is decidable for rank-1 (Hindley-Milner, Section 4), decidable but expensive for rank-2, and undecidable for rank ≥ 3 [Kfoury and Tiuryn 1992]. Full System F (rank-ω) requires explicit type annotations for inference, which is why Haskell's `RankNTypes` extension requires annotations at rank ≥ 2.

---

## 3. Existential Types and Abstract Data Types

### Syntax and Typing Rules

**Existential types** [TAPL §24] are the dual of universal types. Where `∀α. τ` abstracts over a type that the *caller* chooses, `∃α. τ` abstracts over a type that the *implementor* chooses and the caller cannot inspect.

```
Types    τ  ::=  ...  |  ∃α. τ        (existential type)

Terms    M  ::=  ...
              |  pack <τ', v> as ∃α. τ    (introduction)
              |  unpack <α, x> = e in e'  (elimination)
```

The typing rules are:

```
            Γ ⊢ v : τ [α ↦ τ']
T-Pack  ──────────────────────────────────────
            Γ ⊢ pack <τ', v> as ∃α.τ : ∃α. τ


            Γ ⊢ e : ∃α. τ      Γ, α, x:τ ⊢ e' : τ'    α ∉ FTV(τ')
T-Unpack  ──────────────────────────────────────────────────────────
            Γ ⊢ unpack <α, x> = e in e' : τ'
```

The side condition `α ∉ FTV(τ')` ensures the abstract type `α` cannot escape its scope.

### Existential Types as Abstract Data Types

The canonical use of existential types is encoding **abstract data types** (ADTs) [Mitchell and Plotkin 1988; TAPL §24.2]. A counter ADT with operations `create : Counter`, `inc : Counter→Counter`, and `get : Counter→Int` can be typed as:

```
Counter  =  ∃α. { create: α,  inc: α→α,  get: α→Int }
```

An implementation packs a concrete representation (say `Int`) with its operations:

```
intCounter = pack <Int, { create = 0,
                          inc    = λn:Int. n+1,
                          get    = λn:Int. n }>
             as ∃α. { create:α,  inc:α→α,  get:α→Int }
```

A client unpacks the existential and uses only the operations:

```
unpack <Repr, ops> = intCounter in
  ops.get (ops.inc (ops.inc ops.create))   -- = 2
```

The client code cannot mention `Int`; it only knows `Repr` is *some* type. Swapping `intCounter` for a different implementation (say, one backed by `String`) leaves the client code valid.

### The Equivalence with Universal Types

In System F, existentials are definable via universals [TAPL §24.3]:

```
∃α. τ  ≅  ∀β. (∀α. τ→β) → β
```

The **pack** and **unpack** operations are encoded as:

```
pack <τ', v> as ∃α.τ  =  Λβ. λf: (∀α.τ→β). f [τ'] v

unpack <α, x> = e in e'  =  e [τ'] (Λα. λx:τ. e')
```

The proof of soundness: to construct a value of type `∀β. (∀α.τ→β)→β`, we provide a function that, given any type `β` and a consumer `f : ∀α. τ→β`, applies `f` to the witness `τ'` and value `v`. The consumer must work for any `α`, so it cannot inspect the representation — the abstraction barrier is preserved.

We can verify the round-trip identity (pack-then-unpack reduces to the original) with a beta-step calculation. Let `e = pack <τ', v> as ∃α.τ` and `e' = body`. Then:

```
unpack <α, x> = e in e'
  =  e [τ''] (Λα. λx:τ. e')            (by the encoding of unpack)
  =  (Λβ. λf:(∀α.τ→β). f [τ'] v) [τ''] (Λα. λx:τ. e')
  →β  (λf:(∀α.τ→τ''). f [τ'] v) (Λα. λx:τ. e')   (type beta: β↦τ'')
  →β  (Λα. λx:τ. e') [τ'] v             (term beta: f↦Λα.λx.e')
  →β  (λx:τ[α↦τ']. e'[α↦τ']) v         (type beta: α↦τ')
  →β  e'[α↦τ'][x↦v]                    (term beta: x↦v)
```

This is precisely the operational semantics of `unpack`: substitute the witness type `τ'` for `α` and the packed value `v` for `x` in the body `e'`. The encoding is therefore not merely a representation trick — it correctly captures the elimination semantics of existential types within System F's universal quantification.

### Connection to Rust and Modern Languages

Rust's `impl Trait` return type is an existential: `fn make_counter() -> impl Counter` says "I return *some* type satisfying `Counter`, but I won't tell you which." The concrete type is chosen by the implementor and cannot be inspected by the caller. This is precisely the pack/T-Pack rule: the function is the implementor, and the concrete type (`α = ConcreteCounterImpl`) is packed into the existential. The caller, using T-Unpack, can only access the operations declared by the `Counter` trait — the abstract interface.

Rust's `dyn Trait` is the existential paired with a vtable for runtime dispatch: `Box<dyn Counter>` is `∃α. (Counter_vtable, α)` where the vtable is carried at runtime rather than erased. In terms of the ADT encoding: the existential `∃α. { ops: Counter_vtable, data: α }` is not immediately unpacked at the call site but is stored as a fat pointer. Every method call goes through the vtable. The vtable plays the role of the `ops` field in the `intCounter` example — except that rather than being a record of closures (as in the formal encoding), it is a static array of function pointers organized by the Rust ABI.

Swift's **opaque return types** (`some Protocol`) and **existential types** (`any Protocol`, introduced in Swift 5.7) make the same theoretical distinction explicit at the syntax level: `some Protocol` is an existential whose type is fixed at definition time (the compiler knows the concrete type), while `any Protocol` is an existential whose type may vary at runtime (requiring an "existential container" — Rust's fat pointer equivalent). The Swift team's rationale for introducing distinct syntax mirrors the theory: the two concepts have different type-checking and runtime costs despite both being "existentials" in informal parlance.

---

## 4. The Hindley-Milner Type System

### Motivation: Decidable Inference for Let-Polymorphism

Full System F is expressive but its type inference is undecidable [Wells 1999]. The Hindley-Milner (HM) type system identifies a subset — rank-1 (prenex) polymorphism with a specific restriction on generalization — for which inference is decidable and complete [Damas-Milner 1982; Hindley 1969].

### Type Schemes and the Let Rule

HM distinguishes **monotypes** (types without top-level ∀) from **type schemes** (polytypes):

```
Monotypes   τ  ::=  α  |  τ₁→τ₂  |  B       (B = base type)
Type schemes σ  ::=  τ  |  ∀α. σ
```

A type scheme `∀α̅. τ` (where `α̅ = α₁...αn`) is a monotype prefixed with zero or more universal quantifiers. The generalization function is:

```
Gen(Γ, τ)  =  ∀(FTV(τ) \ FTV(Γ)). τ
```

That is, `Gen` closes over exactly those type variables free in `τ` that are not free in any assumption in `Γ`. Variables free in `Γ` represent constraints imposed by the enclosing context and must not be generalized away — doing so would allow a variable whose type has been constrained by surrounding code to be reused at an incompatible type.

The HM typing rules are those of STLC extended with:

```
                 Γ ⊢ e₁ : τ₁      σ = Gen(Γ, τ₁)      Γ, x:σ ⊢ e₂ : τ₂
Let  ────────────────────────────────────────────────────────────────────
                 Γ ⊢ let x = e₁ in e₂ : τ₂
```

The **instantiation** rule (going from a scheme to a monotype) is:

```
                 x : ∀α̅. τ ∈ Γ       τ' = τ[α̅ ↦ τ̅']    (τ̅' fresh)
Inst  ─────────────────────────────────────────────────
                 Γ ⊢ x : τ'
```

Each use of a let-bound variable instantiates its scheme with fresh type variables, allowing different uses at different types.

### Declarative vs. Syntax-Directed Presentation

The **declarative** HM system has Gen and Inst as separate inference rules that may be applied at any point in a derivation. This gives a clean specification — a term is typeable if any derivation tree, with Gen and Inst applied anywhere, assigns it a type — but it is not directly implementable, because applying Gen/Inst at arbitrary points requires search.

The **syntax-directed** (algorithmic) presentation fixes when Gen and Inst are applied: Inst only at variable lookup, Gen only at let-bindings. Algorithm W (Section 5) implements this syntax-directed system. The key soundness and completeness results state that the declarative and syntax-directed systems type exactly the same terms, and that Algorithm W produces the principal type with respect to the declarative system.

### Why Let-Polymorphism Makes Inference Decidable

The restriction is that **only let-bound variables** receive polymorphic types; lambda-bound variables are monomorphic. This is the critical distinction between HM and full System F. In `λx. e`, `x` has a single monotype throughout `e`. In `let x = e₁ in e₂`, `x` has a polytype and can be used at different types in `e₂`.

Why does this restriction restore decidability? The core reason is that unification — the computational engine of type inference — works on **first-order** terms (type expressions without ∀). In HM, whenever the type checker must unify two types during lambda elaboration, both types are guaranteed to be monotypes. Unification of first-order terms is decidable (Robinson 1965). In full System F, unifying types can require matching across ∀ binders — a second-order unification problem — which is undecidable. By confining ∀ to let-introduced schemes and preventing it from appearing inside monotypes (no rank-2 quantifiers), HM keeps unification firmly first-order.

This is also why extending ML with rank-2 types (functions that accept polymorphic arguments) requires type annotations: the moment a lambda argument is allowed to have a type scheme `∀α. τ`, the unification needed to type-check application is no longer first-order.

---

## 5. Algorithm W: Unification and Type Inference

### Robinson's Unification

Unification is the core computational primitive of Algorithm W [Robinson 1965]. A **substitution** `S` is a finite map from type variables to types; `S(τ)` denotes the result of applying `S` to `τ` (replacing each `α` with `S(α)` simultaneously, with idempotency). Two types `τ₁` and `τ₂` are **unifiable** if there exists a substitution `S` (a **unifier**) such that `S(τ₁) = S(τ₂)`. The **most general unifier** (mgu) is a unifier `U` such that every other unifier `S` can be written as `S = S' ∘ U` for some `S'`.

The unification algorithm [TAPL §22.2]:

```
unify(τ₁, τ₂):
  if τ₁ = τ₂:                          return id (identity substitution)
  if τ₁ = α and α ∉ FTV(τ₂):          return [α ↦ τ₂]
  if τ₂ = α and α ∉ FTV(τ₁):          return [α ↦ τ₁]
  if τ₁ = α and α ∈ FTV(τ₂):          fail (occurs check violation)
  if τ₁ = (τ₁ˡ→τ₁ʳ) and τ₂ = (τ₂ˡ→τ₂ʳ):
    S₁ = unify(τ₁ˡ, τ₂ˡ)
    S₂ = unify(S₁(τ₁ʳ), S₁(τ₂ʳ))
    return S₂ ∘ S₁
  otherwise: fail (type error)
```

The **occurs check** (`α ∉ FTV(τ)` before binding `α ↦ τ`) prevents cyclic substitutions that would produce infinite types. Standard ML 97 mandates the occurs check. Several implementations (notably many Prolog interpreters and early ML implementations) omit it for performance; the consequence is that unification can produce infinite type structures, leading to non-terminating elaboration.

**Theorem (Robinson 1965):** If two types are unifiable, the algorithm returns their mgu; otherwise it reports failure. The mgu is unique up to renaming of type variables.

### Algorithm W

The Damas-Milner Algorithm W [Damas-Milner 1982] infers principal types for HM. It takes an environment `Γ` and a term `e` and returns a pair `(S, τ)` where `S` is a substitution on the free type variables in `Γ` and `τ` is the inferred type.

**W(Γ, x):**
```
  σ = Γ(x)                              -- look up x's scheme
  τ = inst(σ)                           -- fresh copies of bound vars
  return ([], τ)                        -- no new constraints on Γ
```
`inst(∀α̅. τ)` replaces each `αᵢ` with a fresh type variable `βᵢ`.

**W(Γ, λx. e):**
```
  β = fresh type variable
  (S, τ) = W(Γ[x:β], e)               -- infer body with x:β
  return (S, S(β)→τ)
```

**W(Γ, e₁ e₂):**
```
  (S₁, τ₁) = W(Γ, e₁)
  (S₂, τ₂) = W(S₁(Γ), e₂)
  β = fresh type variable
  S₃ = unify(S₂(τ₁), τ₂→β)           -- τ₁ must be a function type
  return (S₃ ∘ S₂ ∘ S₁, S₃(β))
```

**W(Γ, let x = e₁ in e₂):**
```
  (S₁, τ₁) = W(Γ, e₁)
  Γ' = S₁(Γ)
  σ = Gen(Γ', τ₁)                      -- generalize: ∀(FTV(τ₁)\FTV(Γ')). τ₁
  (S₂, τ₂) = W(Γ'[x:σ], e₂)
  return (S₂ ∘ S₁, τ₂)
```

### Worked Example: `let f = λx. x in (f 1, f true)`

We trace Algorithm W on this term with empty initial context `Γ = {}`.

**Step 1:** Apply the Let rule. We must first infer the type of `e₁ = λx. x`.

**Step 1a:** W({}, λx. x):
- Introduce fresh variable `α₁` for `x`.
- W({x:α₁}, x) = ([], α₁).
- Return `([], α₁→α₁)`. The substitution is the identity.

**Step 1b:** Generalize. `Γ' = S₁({}) = {}`. `FTV(α₁→α₁) = {α₁}`. `FTV({}) = {}`. So `σ = ∀α₁. α₁→α₁`.

**Step 2:** Infer `e₂ = (f 1, f true)` under `Γ'' = {f : ∀α₁. α₁→α₁}`.

A pair `(e₁, e₂)` desugars to `pair e₁ e₂` (or equivalently we handle it structurally). We need to infer types for `f 1` and `f true` and pair them.

**Step 2a:** Infer `f 1`:
- W(Γ'', f): instantiate `∀α₁. α₁→α₁` with fresh variable `α₂`. Return `([], α₂→α₂)`.
- W(Γ'', 1): return `([], Int)`.
- Unify `α₂→α₂` with `Int→β₁` (fresh `β₁`): S = `[α₂↦Int, β₁↦Int]`. Return `(S, Int)`.
- After applying S, type of `f 1` is `Int`.

**Step 2b:** Infer `f true`:
- W(S(Γ''), f): instantiate `∀α₁. α₁→α₁` with fresh variable `α₃` (independent of `α₂`!). Return `([], α₃→α₃)`.
- W(S(Γ''), true): return `([], Bool)`.
- Unify `α₃→α₃` with `Bool→β₂` (fresh `β₂`): S' = `[α₃↦Bool, β₂↦Bool]`. Return `(S', Bool)`.
- Type of `f true` is `Bool`.

**Step 2c:** Pair the results. The pair `(f 1, f true)` is handled as application of a pair constructor `pair : ∀α β. α→β→α×β`, which is instantiated to `pair [Int][Bool] : Int→Bool→Int×Bool`. The overall substitution at this point is `S_final = S' ∘ S = [α₃↦Bool, β₂↦Bool] ∘ [α₂↦Int, β₁↦Int]`. Since these substitutions act on disjoint variables, their composition is simply `[α₂↦Int, β₁↦Int, α₃↦Bool, β₂↦Bool]`.

**Final type:** `let f = λx.x in (f 1, f true) : Int × Bool`.

The crucial substitution algebra: at Step 2a, after applying S₃ = `[α₂↦Int, β₁↦Int]`, the context `S₃(Γ'')` contains `f : ∀α₁. α₁→α₁` unchanged — because `α₁` is universally quantified in `f`'s scheme and does not appear free in `Γ''`, so applying S₃ to `Γ''` leaves `f`'s scheme intact. This is the invariant that makes let-polymorphism compositional: substitutions on fresh variables introduced during the elaboration of one subterm cannot affect the schemes of let-bound variables.

The generalization in Step 1b — giving `f` the scheme `∀α₁. α₁→α₁` rather than the monotype `α₁→α₁` — is what permits the two independent instantiations. Without let-polymorphism, both uses of `f` would share the single type variable `α₁`, and unification of `α₁ = Int` (from step 2a) and `α₁ = Bool` (from step 2b) would fail with a type error at the composition `S' ∘ S`, since both substitutions would conflict on `α₁`.

### Correctness: The Principal Types Theorem

**Theorem (Damas-Milner 1982) [TAPL §22.6]:** If `Γ ⊢ e : τ` is derivable, then Algorithm W terminates and returns `(S, τ')` where `τ'` is the principal type of `e` with respect to `Γ`. Every other valid type for `e` in `Γ` is an instance of `τ'`.

The proof of correctness has two components:

**Soundness:** By induction on the structure of `e`. Each case shows that if W returns `(S, τ)`, then `S(Γ) ⊢ e : τ` is derivable in the declarative HM system. The key steps are: (a) for variables, the inst operation correctly derives an instance of the scheme; (b) for lambda, the fresh type variable for the bound argument gives a monotype which is consistent with the inference of the body; (c) for application, the unification step produces a substitution such that the function's argument type matches the argument's type; (d) for let, the generalization Gen is sound because only free variables not constrained by `Γ` are quantified.

**Completeness:** If `Γ ⊢ e : τ` is any valid typing derivation, then there exists a substitution `S'` such that `S'(τ_W) = τ`, where `τ_W` is the type returned by W. Equivalently, W's result is at least as general as any valid typing. The proof proceeds by showing that at each unification step, the mgu is a factor of any unifier that would be needed for any valid derivation. The existence of an mgu is guaranteed by Robinson's theorem, and the principality of the mgu propagates through the recursion.

Together they mean W can be trusted as the complete type-inference algorithm for HM: it neither rejects valid programs (completeness) nor accepts invalid ones (soundness).

### Complexity

Algorithm W is nearly linear in practice (O(n α(n)) where α is the inverse Ackermann function from union-find, achieved via the path-compressed union-find structure for the substitution). However, the worst-case complexity is exponential. The canonical pathological example is the deeply nested let:

```
let x₁ = λa. a in
let x₂ = λa. x₁ (x₁ a) in
let x₃ = λa. x₂ (x₂ a) in
  ...
let xₙ = λa. xₙ₋₁ (xₙ₋₁ a) in xₙ
```

To see why the types grow exponentially, trace the first few levels. At level 1:

```
x₁ : ∀α. α→α     (scheme: the identity)
```

At level 2, inferring the type of `λa. x₁ (x₁ a)`: both applications of `x₁` are instantiated independently. The first application `x₁ a` gets type `α₂→α₂` for fresh `α₂`, and unifying with the argument `a:α₀` gives `α₀ = α₂`. The outer `x₁ (x₁ a)` gets another fresh instantiation. After all unifications, the body has a monotype, and `x₂`'s scheme is `∀α. α→α` — same size. But the *internal substitution* being composed is already of size proportional to the number of unifications performed.

The explosion occurs at level `k`: when inferring `λa. xₖ₋₁ (xₖ₋₁ a)`, each of the two calls to `xₖ₋₁` is instantiated with a *fresh copy* of its type scheme, which at level `k-1` has 2^(k-2) free variables before generalization. The unifier produced at level `k` has size 2^(k-1). The composed substitution `Sₖ ∘ ... ∘ S₁` is exponential in `k`.

This is not a flaw in Algorithm W — it is a fundamental property of the HM type system, because the types *are* exponentially large when fully unfolded. Haskell's notorious slow compilation of certain deeply nested polymorphic code traces directly to this. The practical mitigation is sharing (representing types as graphs rather than trees), which makes equal subterms physically identical and reduces the apparent blow-up, but does not eliminate it for types that genuinely grow.

---

## 6. Let-Polymorphism and the Value Restriction

### The Mutable Reference Counterexample

HM's generalization rule is unsound in the presence of mutable references. The classic counterexample [Wright 1995]:

```sml
let r = ref []
```

Without the value restriction, the expression `ref []` is typed as follows: `[]` has type `∀α. α list` (empty list, any element type), so `ref []` has type `∀α. α list ref`. Algorithm W, without restriction, would generalize `r` to the scheme `∀α. α list ref`. The unsoundness is exposed in four steps:

```
Step 1:  r : ∀α. α list ref
         (* instantiate at int:  *) r₁ = r : int list ref
         (* instantiate at bool: *) r₂ = r : bool list ref

Step 2:  r₁ := [1]
         (* stores integer 1 in the cell pointed to by r *)

Step 3:  val x = hd (!r₂)
         (* r₂ is the SAME cell as r₁ (same runtime location!) *)
         (* but typed as bool list ref: reads a bool from an int cell *)

Step 4:  x : bool    (* type says bool, runtime value is an int word *)
         (* any subsequent use of x as a bool is undefined behavior *)
```

The key fact is that `r₁` and `r₂` are the same runtime location: the let-binding assigns one reference cell, but the polymorphic scheme makes the type checker believe it is two distinct, differently-typed cells. The type system's model of the heap is unsound — it permits two incompatible views of a single mutable cell. The resulting program reads a machine word containing an integer and interprets it as a boolean pointer or tagged value, producing a type confusion that on a concrete SML or OCaml runtime would manifest as a segmentation fault or garbage data in subsequent computations.

The deeper issue is that **effects and let-polymorphism do not commute**: generalization assumes the let-bound expression is "pure" (its type is independent of when and how many times it is evaluated). A `ref []` call is not pure — it allocates a new cell with a specific runtime address. Each instantiation of its type scheme corresponds to a different use of that same cell, and the two uses are not independent.

### The SML '97 Value Restriction

The **value restriction** [Wright 1995] states: generalization is safe only when the expression being bound is a **syntactic value** — a lambda abstraction, a constant, or a constructed value containing only values (not general expressions). The critical observation is that values do not perform side effects; they cannot allocate mutable state. Hence, generalizing a value's type is safe.

Formally, the Let rule in SML '97 generalizes only when `e₁` is a value:

```
          e₁ is a syntactic value     Γ ⊢ e₁ : σ     Γ, x:σ ⊢ e₂ : τ
LetVal  ──────────────────────────────────────────────────────────────
          Γ ⊢ let x = e₁ in e₂ : τ   (with Gen applied to e₁'s type)
```

If `e₁` is not a syntactic value, the Let rule still applies but without generalization: `σ = τ₁` (the monotype), and no `∀` is introduced.

Applied to the counterexample: `ref []` is not a syntactic value (it is a function application), so `r` receives only the monotype `α₀ list ref` for a fixed `α₀`. The first store `r := [1]` forces `α₀ = int`, making `r : int list ref` throughout. Attempting `r := [true]` is a type error, as required.

### Weak Type Variables and OCaml's Relaxed Value Restriction

When the SML '97 value restriction applies and generalization is withheld, the variable receives a **weak type variable**, written `'_a` (or `'_weak1` in OCaml's notation). A weak type variable is a monotype placeholder that will be resolved the first time the variable is used at a concrete type:

```ocaml
# let r = ref [];;
val r : '_weak1 list ref = {contents = []}

# r := [1];;
- : unit = ()

# r;;
val r : int list ref = {contents = [1]}
```

The first assignment `r := [1]` resolves `'_weak1` to `int`, permanently. Attempting `r := [true]` afterward is a type error. Weak type variables are not polymorphic — they are deferred monotypes. The OCaml toplevel's distinct display of `'_weak1` versus `'a` is the user-visible signal that generalization has been suppressed.

**OCaml's relaxed value restriction** [Garrigue 2004] is more permissive than SML '97: it permits generalization of type variables that appear only in *covariant positions* (output positions, not under arrows on the left). The reason is that covariant type variables cannot be used as the type of a stored value — only as the type of a value returned. Since no mutation can be performed through a covariant type variable, generalizing it is safe even for non-values.

```ocaml
(* Empty list: [] : 'a list — covariant 'a *)
let empty = [];;              (* Fully generalized: 'a list — safe! *)

(* Partial application with covariant result *)
let mk_list x = [x];;        (* 'a -> 'a list — fully polymorphic *)
let f = mk_list;;             (* 'a -> 'a list — OK (covariant output) *)
```

In practice, OCaml users encounter the value restriction when using partial application:

```ocaml
let f = List.map (fun x -> x + 1)   (* not a value! eta-expand: *)
let f lst = List.map (fun x -> x+1) lst   (* OK: eta-expanded lambda *)
```

The standard workaround — eta-expansion — converts a partial application (non-value) to a lambda abstraction (value), restoring the right to generalize. The OCaml documentation's canonical advice is: "if you see `'_weak1`, add an explicit argument."

The historical predecessor to the value restriction was Tofte's **imperative type discipline** (used in early ML of Edinburgh), which tracked "dangerous" type variables in the context through a region-based effect annotation. The value restriction replaced it in SML '97 as a simpler rule with the same soundness guarantee and fewer false negatives in practice, at the cost of some expressiveness.

---

## 7. Principal Types

### Definition

A type `σ` is a **principal type** for `e` under context `Γ` if:
1. `Γ ⊢ e : σ` is derivable, and
2. For every `τ` with `Γ ⊢ e : τ`, there exists a substitution `S` such that `S(σ) = τ`.

In other words, `σ` is a generic instance of every other type for `e`, so it is the *most general* type. A term has a principal type iff all its types can be obtained by instantiating a single scheme.

### The Principal Types Theorem for HM

**Theorem [Damas-Milner 1982; Hindley 1969]:** Every typeable term in HM has a principal type, and Algorithm W computes it.

This is a strong completeness result: the type inferencer never loses information. Compare with System F, where no analogue holds — a closed term in System F may have multiple types none of which subsumes the others (because of impredicativity).

### Contrast: Languages Without Principal Types

C++ template functions do not have principal types in the HM sense. A template `template<typename T> T id(T x) { return x; }` is resolved by instantiation, not generalization. The "type" of the template is a compile-time polymorphic entity, not a single first-class type. When overloading is involved, a function can be typed by multiple non-comparable signatures. Type-directed overloading resolution in C++ (and similarly in Java via generics with wildcards) violates the principal types property: there is no single most-general type from which all valid types derive.

**Why this matters practically.** In a language with principal types, the type inferencer can be run once, producing the most general type, and every later use is a specialization of that type — no re-inference is needed at call sites. Error messages can point precisely to the location where the inferred type became too specific. In languages without principal types, the type of a function depends on its call sites (as in C++ concept checking and Java bounded wildcards), making error attribution more difficult and whole-program reasoning necessary.

Type classes in Haskell (Section 10) maintain principal types because the class constraint `Eq a => a→a→Bool` is itself a principal type — every valid instance type for the function is an instance of this constrained type. The constraint solver determines which instance to use, but the type remains principal.

### The Relation Between Principal Types and Free Theorems

The principal types property and free theorems are connected: if a function has a principal type that is a polymorphic type `∀α. τ`, then Reynolds's parametricity theorem applies to it — the free theorem derived from `∀α. τ` holds for every implementation of that principal type. If a function does *not* have a principal type (e.g., an overloaded C++ function), there is no single polymorphic type from which to derive a free theorem, and different instantiations may behave in entirely different ways.

This is the deep reason why free theorems hold for parametric polymorphism but not for ad-hoc polymorphism: parametricity relies on the existence of a *single* most-general type whose universal quantifier ranges over all types uniformly. Type-class constrained types `∀a. Eq a ⇒ τ` occupy an intermediate position: free theorems hold for the parametric part (the unconstrained type variables), but not for the constrained part (the `Eq a` constraint permits different behavior for different instances of `Eq`).

---

## 8. Bidirectional Type Checking

### The Two Modes

**Bidirectional type checking** [Pierce and Turner 2000; Dunfield and Krishnaswami 2013] splits the single `Γ ⊢ e : τ` judgment into two:

```
Γ ⊢ e ⇒ τ    (inference / synthesis: e synthesizes type τ)
Γ ⊢ e ⇐ τ    (checking: e checks against type τ)
```

In the *inference* mode, the type is an output — computed from the structure of `e`. In the *checking* mode, the type is an input — the expected type is propagated into `e`. This bidirectionality eliminates the need for unification in many common cases and enables handling of impredicative polymorphism that pure inference cannot handle.

### Bidirectional Rules for STLC

The core rules [TAPL §25.6; Dunfield-Krishnaswami 2013]:

```
                  x:τ ∈ Γ
Var⇒  ──────────────────────────
                  Γ ⊢ x ⇒ τ


                  Γ, x:τ₁ ⊢ e ⇐ τ₂
Abs⇐  ──────────────────────────────────
                  Γ ⊢ λx. e ⇐ τ₁→τ₂


                  Γ ⊢ e₁ ⇒ τ₁→τ₂     Γ ⊢ e₂ ⇐ τ₁
App⇒  ──────────────────────────────────────────────
                  Γ ⊢ e₁ e₂ ⇒ τ₂


                  Γ ⊢ e ⇒ τ'     τ' = τ   (or τ' <: τ in subsystem)
Sub   ──────────────────────────────────────────────────────
                  Γ ⊢ e ⇐ τ


                  Γ ⊢ e ⇐ τ
Ann⇒  ──────────────────────────
                  Γ ⊢ (e : τ) ⇒ τ
```

The **Abs⇐** rule is key: it avoids creating a fresh type variable for `x` (as inference would require) by reading `τ₁` from the expected type `τ₁→τ₂`. The **Sub** rule switches from checking mode to inference mode: if `e` synthesizes `τ'` and the inferred type equals (or is a subtype of) the expected `τ`, the check succeeds. **Ann⇒** handles explicit annotations: `(e : τ)` synthesizes `τ` by checking `e` against `τ`.

For System F, the type abstraction and application rules are:

```
                  Γ, α ⊢ e ⇐ τ
TAbs⇐  ──────────────────────────────
                  Γ ⊢ Λα. e ⇐ ∀α. τ


                  Γ ⊢ e ⇒ ∀α. τ
TApp⇒  ──────────────────────────────
                  Γ ⊢ e [τ'] ⇒ τ[α↦τ']
```

### Practical Advantages

Bidirectional type checking provides several engineering benefits over pure inference:

1. **Better error messages.** When checking mode is entered, the expected type is known; a mismatch can report exactly what type was expected and what was found.
2. **Impredicative polymorphism.** Pure HM inference cannot handle `∀` on the left of arrows without extending unification; bidirectional checking can handle these cases by propagating the known type inward.
3. **Reduced annotation burden.** Propagating expected types into subexpressions means fewer explicit annotations are required. Lambda arguments annotated at the call site rather than the definition site.

### The DK Algorithm

Dunfield and Krishnaswami's 2013 algorithm (the **DK algorithm**) [Dunfield-Krishnaswami 2013] extends bidirectional checking to full System F with a complete algorithmic treatment. The key innovation is an **ordered context** Γ that interleaves type variable declarations, term variable bindings, and *existential type variable declarations* (written `α̂`), together with solved constraints on those existentials.

The context has three kinds of entries:

```
Γ  ::=  ·                    (empty)
      | Γ, α                  (universal type variable)
      | Γ, x:σ                (term variable)
      | Γ, α̂                  (existential type variable — unsolved)
      | Γ, α̂ = τ              (existential type variable — solved to τ)
      | Γ, ▶α̂                 (scope marker for α̂)
```

Existential variables `α̂` are fresh metavariables introduced during inference of lambda arguments and function applications. They play the role that "fresh type variables" play in Algorithm W, but they are tracked in the ordered context rather than in a substitution. Solving `α̂ = τ` in the context is the DK analogue of producing `[α̂ ↦ τ]` in Algorithm W.

The ordering of the context is significant: an existential variable `α̂` can only be solved to a type `τ` if `FTV(τ)` names only variables that appear *to the left* of `α̂` in the context. This prevents circular dependencies (the analogue of the occurs check) and ensures that solutions respect scope.

The central judgment is:

```
Γ ⊢ e ⇒ A ⊣ Δ      (inference: e infers type A, input context Γ, output context Δ)
Γ ⊢ e ⇐ A ⊣ Δ      (checking: e checks against A, input Γ, output Δ)
```

The output context `Δ` is `Γ` with existentials solved; it records the constraints learned during type checking. For example, checking `λx. x` against `α̂→β̂` (where `α̂` and `β̂` are existentials) checks `x:α̂ ⊢ x ⇐ β̂`, which unifies `β̂ = α̂` in the output context: `Δ = Γ, β̂ = α̂`. This is cleaner than Algorithm W's substitution threading, because the context serves as the accumulator.

The **completeness** of DK requires sufficient type annotations: for each type abstraction `Λα. M`, the user must annotate the type argument (or the expected type must be known from context). This is the same restriction as `RankNTypes` in GHC: a rank-2 or higher function requires annotation at the call site.

GHC's type checker (GHC 8.0+) uses a variant of bidirectional checking extended with *quantified type variables* and *impredicative instantiation* (ImpredicativeTypes extension). Scala's Dotty compiler (the reference implementation of Scala 3) uses a bidirectional algorithm based on local type inference, called **Colored Local Type Inference** (Odersky et al. 2001) extended to handle Scala 3's path-dependent types. Rust's type checker is fundamentally bidirectional, propagating expected types through pattern matching and closures: the type of a closure `|x| x + 1` is not inferred from `x`'s definition but from how the closure is used — if it is passed to a function expecting `Fn(i32) -> i32`, the argument `x` is given type `i32` by downward propagation. TypeScript's inference is similarly bidirectional, explaining why `const f = (x) => x + 1` infers the argument as `number` when used in a numeric context but `any` in isolation.

---

## 9. Row Polymorphism

### The Structural Record Problem

Standard parametric polymorphism handles lists, trees, and functions uniformly, but it struggles with records. Consider a function that reads the `name` field of a record:

```
getName : { name: String } → String
getName r = r.name
```

In nominal type systems, this function can only be called with records that have exactly the type `{ name: String }`. Adding a field — `{ name: String, age: Int }` — produces a distinct type, and `getName` cannot be called on it without explicit upcast or duplicate code. Structural subtyping (Section 14's topic) solves part of the problem, but it interacts poorly with mutation and cannot express *record extension* generically.

### Row Variables

**Row polymorphism** [Wand 1987; Rémy 1989] introduces **row variables** `ρ` that range over sets of fields. A record type is written:

```
τ  ::=  { l₁:τ₁, ..., lₙ:τₙ | ρ }
```

where `ρ` is a row variable representing "the rest of the fields." The field types `τᵢ` are concrete; `ρ` is an open tail. A function that accesses only the `name` field can be typed:

```
getName : ∀ρ. { name: String | ρ } → String
```

This function is polymorphic in `ρ`: it works on any record that has *at least* a `name: String` field, regardless of what other fields are present. A closed record type is `{ name: String | ∅ }` where `∅` is the empty row.

Row unification works similarly to ordinary unification but must account for field labels. Rémy's formulation [Rémy 1989] uses **presence/absence labels**: each field `l` in a row is either present (with a type) or absent. Row variables range over assignments of presence/absence to field names. The constraint that field `l` is absent from row `ρ` is written with the **lacks predicate** `ρ \ l`, read "`ρ` lacks label `l`".

The formal row unification rule for the extension type is:

```
unify({ l:τ | ρ₁ }, { l:τ' | ρ₂ }):
  S₁ = unify(τ, τ')
  fresh ρ₃  (a new row variable)
  S₂ = [ρ₁ ↦ { S₁(ρ₂) }, ρ₂ ↦ ρ₃]   -- only if ρ₁, ρ₂ ≠ ρ₃
  return S₂ ∘ S₁
```

Unifying `{ name:String | ρ₁ }` with `{ name:String, age:Int | ρ₂ }` proceeds by matching the `name` fields (unify `String` with `String`, trivially), then binding `ρ₁ = { age:Int | ρ₂ }`. Ordering of fields is irrelevant (rows are sets, not sequences), so `{ age:Int, name:String }` and `{ name:String, age:Int }` unify without constraint.

The **lacks predicate** is essential for record extension. The type of the extension operation for label `l` must require that the input row does not already have `l`:

```
extend_l : ∀α. ∀ρ\l. { ρ } → α → { l:α | ρ }
```

The notation `∀ρ\l` quantifies over row variables that lack the label `l`, preventing the operation from adding a duplicate field. Without this restriction, a record `{ l:Int, ... }` extended with `l:Bool` would produce a row with two `l` fields — which is ill-formed (standard row systems prohibit duplicate labels).

**An OCaml polymorphic variant example.** Row polymorphism for variants is dual to record rows: a variant type is an open sum rather than an open product. In OCaml:

```ocaml
let describe = function
  | `Circle r    -> Printf.sprintf "circle of radius %d" r
  | `Square s    -> Printf.sprintf "square of side %d" s

(* Inferred type: *)
(* describe : [< `Circle of int | `Square of int ] -> string *)
```

The type `[< `Circle of int | `Square of int ]` is a row type: the `[<` notation means "at most these variants" (the row is closed from above — a presence predicate). Calling `describe` on a value of type `[< `Circle of int | `Square of int | `Triangle of int ]` is rejected (the extra case is not handled). Adding a handler for `Triangle`:

```ocaml
let describe2 = function
  | `Triangle t  -> Printf.sprintf "triangle of side %d" t
  | other        -> describe other   (* delegate the rest *)

(* Inferred type: *)
(* describe2 : [< `Circle of int | `Square of int | `Triangle of int ] -> string *)
```

Here `other` has a row type polymorphic in the remaining cases, and the composition is type-safe without any explicit union type declaration.

### Implementations

OCaml's **polymorphic variants** use row polymorphism for variant types: a function that handles some cases of a variant is polymorphic in the remaining cases (`[> `A | `B ]` means "at least `A` and `B`"). Elm's **extensible records** (in earlier versions) used row polymorphism to allow functions to access specific fields without being restricted to a particular record type. PureScript supports row polymorphism as a first-class feature, making it possible to write type-safe `merge`, `project`, and `restrict` record operations polymorphically. TypeScript's `keyof` and mapped types provide a more limited form of record polymorphism at the type level.

---

## 10. Type Classes and Dictionary Translation

### The Wadler-Blott Approach

Type classes [Wadler-Blott 1989] provide **ad-hoc polymorphism** within a principled type-inference framework. A type class defines a set of operations parameterized by a type:

```haskell
class Eq a where
  (==) :: a -> a -> Bool
  (/=) :: a -> a -> Bool
  x /= y = not (x == y)    -- default implementation
```

A function using equality has a *constrained type*:

```haskell
member :: Eq a => a -> [a] -> Bool
```

The constraint `Eq a` is a predicate on the type `a`: the function requires that `a` belongs to the `Eq` class. The full type is `∀a. Eq a ⇒ a → [a] → Bool`.

This is more expressive than parametric polymorphism alone: a truly parametric `member` (without constraints) cannot compare elements, because parametricity forbids inspecting values. The `Eq` constraint permits element-specific comparison while still allowing a single implementation at the source level.

### Dictionary Translation

The elaboration of type classes into System F (or the Core IR in GHC) proceeds by **dictionary translation** [Wadler-Blott 1989; TAPL §25.7]. Each class instance becomes a **dictionary** — a record of function values:

```
-- The Eq a constraint becomes an implicit dict parameter:
member :: ∀a. EqDict a -> a -> [a] -> Bool
member dict x []     = False
member dict x (y:ys) = (dict.eq x y) || member dict x ys
```

where `EqDict a = { eq :: a→a→Bool, neq :: a→a→Bool }`.

At each call site, the compiler resolves which instance is needed and passes the appropriate dictionary:

```haskell
member 3 [1,2,3]
-- elaborates to:
member intEqDict 3 [1,2,3]
-- where intEqDict :: EqDict Int = { eq = intEq, neq = intNeq }
```

Dictionary construction is done entirely at compile time for monomorphic call sites; for polymorphic call sites (where the type is a variable), the dictionary is passed as a runtime argument (a function pointer record, in C++ terms).

### End-to-End Elaboration: From Haskell Source to LLVM IR

The full path from a constrained Haskell function to LLVM IR illustrates the entire pipeline concretely. Consider `elem :: Eq a => a -> [a] -> Bool`:

**Source Haskell:**
```haskell
class Eq a where
  (==) :: a -> a -> Bool

elem :: Eq a => a -> [a] -> Bool
elem x []     = False
elem x (y:ys) = x == y || elem x ys
```

**Step 1: Constraint generation.** During type checking, the constraint solver determines that the body uses `(==) :: a → a → Bool`, which requires an `Eq a` instance. The constraint is collected into the type as `∀a. (Eq a) ⇒ a → [a] → Bool`.

**Step 2: Dictionary elaboration to GHC Core.** Each class `Eq a` becomes a record type, and each constraint becomes an explicit dictionary argument. The elaborated Core (approximately):

```
-- Dictionary type (generated for class Eq)
data EqDict a = EqDict { eq :: a -> a -> Bool }

-- Elaborated function
elem :: forall a. EqDict a -> a -> [a] -> Bool
elem @a d x ys = case ys of
  []     -> False
  (y:ys) -> (eq @a d x y) || elem @a d x ys
```

The `@a` notation denotes an explicit type application (System F style); `d` is the dictionary. Every call to `(==)` becomes `eq @a d`, a field access into the dictionary followed by a function call.

**Step 3: Monomorphic call site — specialization.** At the call site `elem 3 [1,2,3]`:

```
elem @Int intEqDict 3 [1,2,3]
-- where intEqDict = EqDict { eq = (==) @Int }
-- and   (==) @Int = \x y -> x == y  (primitive integer comparison)
```

GHC's **SPECIALISE pragma** (or automatic specialization at `-O2`) generates a monomorphic copy `elem_Int :: Int -> [Int] -> Bool` with the dictionary substituted and the indirect call inlined.

**Step 4: LLVM IR (after specialization).** The specialized version with the dictionary inlined:

```llvm
; elem_Int :: Int -> [Int] -> Bool
define i1 @elem_Int(i64 %x, %List_Int* %ys) {
entry:
  %null = icmp eq %List_Int* %ys, null
  br i1 %null, label %ret_false, label %cons_case

cons_case:
  %head_ptr = getelementptr %List_Int, %List_Int* %ys, i32 0, i32 0
  %head = load i64, i64* %head_ptr
  %eq_result = icmp eq i64 %x, %head   ; inlined integer ==
  br i1 %eq_result, label %ret_true, label %recurse

recurse:
  %tail_ptr = getelementptr %List_Int, %List_Int* %ys, i32 0, i32 1
  %tail = load %List_Int*, %List_Int** %tail_ptr
  %rec = call i1 @elem_Int(i64 %x, %List_Int* %tail)
  ret i1 %rec
  ...
}
```

The dictionary has been completely eliminated — the indirect call `eq d x y` has become `icmp eq i64 %x, %head`. LLVM sees a monomorphic function with no polymorphism, no indirect calls, and no dictionaries. The entire type-class machinery is a compile-time artifact.

**Contrast — polymorphic call site.** If `elem` is called at a polymorphic type (e.g., inside another generic function), the dictionary pointer is passed as a runtime argument:

```llvm
define i1 @elem_poly(%EqDict* %dict, i8* %x, %List* %ys) {
  ...
  %eq_fn_ptr = getelementptr %EqDict, %EqDict* %dict, i32 0, i32 0
  %eq_fn = load i1(i8*, i8*)*, i1(i8*, i8*)** %eq_fn_ptr
  %eq_result = call i1 %eq_fn(i8* %x, i8* %head)
  ...
}
```

The indirect call through `%eq_fn` is the dictionary dispatch. LLVM's indirect-call devirtualization (`-O3` with LTO) can sometimes eliminate this when the dictionary is provably a static constant, but in general the indirect call remains.

### GHC's Core and LLVM

In GHC's compilation pipeline, Haskell source is elaborated to **Core** (a typed System F-like IR) where all dictionaries are explicit. From Core, GHC generates **STG** (Spineless Tagless G-machine IR), then **Cmm** (a C-like IR), and finally LLVM IR or native code. By the time LLVM IR is produced, dictionaries are ordinary structs being passed by pointer, and dictionary method calls are loads from those structs followed by indirect calls. The LLVM optimizer can devirtualize these calls (convert indirect to direct calls) when the dictionary is statically known — a process analogous to C++ vtable devirtualization.

### The Elaboration Algorithm: Constraint Generation and Solving

The full elaboration of type classes involves a two-phase algorithm [Wadler-Blott 1989; Jones 1993]:

**Phase 1: Constraint generation.** During type inference (Algorithm W), every use of an overloaded identifier generates a constraint. Inferring `elem x ys` generates the constraint `Eq a` (where `a` is the element type). Constraints are accumulated alongside the type derivation; the result of type-checking an expression is a pair `(τ, C)` where `τ` is the type and `C` is the set of unsolved constraints.

**Phase 2: Constraint solving / dictionary elaboration.** After inference, the constraint solver attempts to discharge each constraint `Eq τ` by finding an instance in scope. The solver proceeds by:

1. **Instance lookup:** For `Eq Int`, find the instance `instance Eq Int where ...` and produce `intEqDict :: EqDict Int`.
2. **Instance expansion:** For a derived constraint like `Eq (a, b)` given `Eq a` and `Eq b`, produce `pairEqDict :: EqDict a → EqDict b → EqDict (a, b)` (the instance for pairs requires sub-dictionaries).
3. **Propagation:** If a constraint `Eq a` cannot be discharged because `a` is a free type variable (the function is itself polymorphic over `a`), the constraint is *propagated* to the function's type as `Eq a ⇒ τ`. Callers of this function must then provide an `Eq a` dictionary.

The elaborated term replaces each constrained identifier with an explicit dictionary access, and each function with a propagated constraint receives an additional dictionary parameter.

**Superclasses and dictionary inheritance.** When a class has a superclass (`class Eq a => Ord a`), the `OrdDict` contains a field pointing to the `EqDict`. Looking up `(==)` on an `Ord a` constraint requires extracting the `EqDict` from the `OrdDict` first — a field projection operation. This is analogous to inheritance in object-oriented languages (where a subclass vtable may contain a pointer to its superclass vtable), and it is precisely how GHC's Core represents the superclass relationship.

### Multi-Parameter Type Classes and Functional Dependencies

Haskell's standard type classes have one type parameter. Multi-parameter type classes (`class Coerce a b where coerce :: a → b`) introduce type relationships. Without constraints, multi-parameter classes lead to ambiguity: the compiler cannot determine which instance to use when multiple instances for a type pair exist. **Functional dependencies** [Jones 2000] declare that some parameters of a multi-parameter class are determined by others (`class Coll c e | c -> e where ...`), restoring principal types by imposing a determinacy condition on the class's parameters: if `c` is known, `e` is uniquely determined. **Type families** are an alternative: a type-level function that maps types to types (`type family Elem c :: *`), eliminating the ambiguity differently and integrating naturally with GHC's constraint solver.

### Coherence

Haskell mandates **coherence**: for any given type, there can be at most one instance of any type class. If multiple instances existed (as they can in Scala via implicits), dictionary passing would be ambiguous — the compiled code would depend on which instance was in scope at the call site, violating referential transparency. Coherence is enforced by the **orphan instance** restriction (an instance may only be defined in the module defining either the class or the type) and the requirement that instances not overlap. Scala's **given**/**implicit** system relaxes coherence, which gives more flexibility but can produce compilation results that depend on the import order — a property that violates the functional purity Haskell relies on for its optimizer.

---

## 11. Ad-Hoc vs. Parametric Polymorphism: The Implementation Continuum

### The Fundamental Distinction

Two forms of polymorphism exist and differ in a precise sense [Strachey 1967; Reynolds 1983]:

**Parametric polymorphism** (System F / HM): a single implementation handles all types uniformly. The implementation does not inspect the type argument. Free theorems hold. Implementation strategy: **type erasure** — type arguments are irrelevant at runtime.

**Ad-hoc polymorphism** (type classes, overloading): different implementations are selected for different types. The implementation *is* type-dependent. Free theorems do not hold (the function `f : Num a => a -> a` need not be the identity — it could be `(+1)` for `Int` and `(*2)` for `Float`). Implementation strategy: dictionary passing or monomorphization.

### Haskell: Dictionary Passing to LLVM

Haskell compiles polymorphic code via dictionary passing (Section 10). At the LLVM IR level, a Haskell function `f :: Eq a => a -> [a] -> Bool` becomes an LLVM function taking an extra `i8**` argument (the dictionary pointer). The dictionary itself is a static constant or heap-allocated record of function pointers. GHC's LLVM backend specializes dictionaries when the type is monomorphic at the call site (via the **specialise** pragma or automatic specialization), producing direct calls that LLVM can inline.

```llvm
; Polymorphic Haskell function after elaboration
define i1 @member(%EqDict* %dict, %a* %x, %List* %xs) {
  ; load the eq function pointer from the dictionary
  %eq_fn_ptr = getelementptr %EqDict, %EqDict* %dict, i32 0, i32 0
  %eq_fn = load i64(%a*, %a*)*, i64(%a*, %a*)** %eq_fn_ptr
  ; indirect call
  %result = call i1 %eq_fn(%a* %x, ...)
  ...
}
```

### C++ Templates: Monomorphization at Instantiation

C++ templates are monomorphized at compile time: `template<typename T> T id(T x)` produces a separate function `id<int>`, `id<float>`, `id<std::string>`, etc., each compiled independently to LLVM IR [TAPL §24.1, informally]. The C++ frontend (Clang's Sema and AST) performs template instantiation before LLVM IR generation; by the time LLVM IR is produced, all type parameters have been erased and replaced with concrete types. This produces code that LLVM can optimize freely for each instantiation — but it also produces code bloat (each instantiation is a separate function).

Clang's template instantiation is performed in `clang/lib/Sema/SemaTemplateInstantiate.cpp`. The generated LLVM IR has no polymorphism — every function is monomorphic, and the LLVM optimizer has full type information.

### Rust: Monomorphization Plus Trait Objects

Rust uses monomorphization for generic functions bounded by traits: `fn id<T>(x: T) -> T` compiles to a separate function for each concrete `T` used. The compiler performs monomorphization in the MIR-to-LLVM lowering stage (`rustc_codegen_llvm`). Each monomorphized instance produces a separate LLVM function.

Rust additionally supports **trait objects** (`dyn Trait`), which are existential types (Section 3) with a vtable. A `Box<dyn Animal>` is a fat pointer: `(data_ptr, vtable_ptr)`. The vtable is a static struct of function pointers, analogous to a Haskell dictionary. Method calls through a trait object are indirect calls through the vtable — the same pattern as Haskell dictionary dispatch, but with a different representation (the dictionary is stored alongside the data pointer rather than passed as a separate argument).

### OCaml: Uniform Representation and Type Erasure

OCaml occupies a middle position between Haskell's dictionary passing and Rust's monomorphization: it uses **uniform representation**, where every value (including polymorphic ones) is represented as a single machine word — either an immediate (integers, booleans tagged in the low bit) or a pointer to a heap block. A `'a list` and an `int list` have exactly the same runtime representation; the element type `'a` is erased. Generic functions like `List.map` work over this uniform representation without any type-specific dispatch.

The advantage is zero runtime overhead for polymorphism (no dictionary, no indirect call, no monomorphization bloat) for functions that are genuinely parametric. The disadvantage is that unboxed primitive types require boxing: an `int list` stores boxed integers (pointers to heap integers with a tag), rather than unboxed `int64` values. OCaml's `Array.get` on a `float array` is a special case: OCaml detects monomorphic `float array` at the point of construction and uses unboxed storage — a form of partial monomorphization for the most performance-critical case.

OCaml's compilation to LLVM (via the `llvm` backend in recent OCaml or via external projects) produces LLVM IR where all polymorphic values are `i64` (the uniform word). Generic functions are single LLVM functions that operate on `i64*` pointers. This is the cleanest form of type erasure at the LLVM IR level: the polymorphism is genuinely absent, replaced by a uniform data model.

### Java Generics: Type Erasure

Java generics use **type erasure**: `List<T>` is compiled to `List<Object>` at the bytecode level, with casts inserted at use sites. This means a `List<Integer>` and a `List<String>` are the same JVM class at runtime — a design choice made for backward compatibility (Java 1.4 `List` and Java 5 `List<T>` use the same `.class` file). Type erasure eliminates the code bloat of monomorphization but prevents certain optimizations (no specialization for primitive types — hence `List<int>` is not legal; you need `List<Integer>` with boxing overhead). Java does not produce LLVM IR (it targets the JVM), but this represents the far end of the erasure spectrum.

### Why LLVM IR Has No Polymorphism

The table in the previous section makes visible a fundamental architectural fact: LLVM IR is positioned *below* every polymorphism strategy. It receives monomorphic code regardless of the strategy the source language uses. This is by design: LLVM's job is to optimize and lower concrete machine code, not to reason about source-level type abstractions. The type-theoretic content of this chapter — parametricity, type schemes, Algorithm W, type classes — is entirely the responsibility of the language frontend.

This creates a sharp division of labor. The frontend (Clang, GHC, rustc, ocamlopt) is responsible for:
- Type inference and checking (Algorithm W or bidirectional variants)
- Polymorphism resolution (instantiation, dictionary construction, specialization)
- Type erasure and representation decisions

LLVM is responsible for:
- Optimizing the resulting monomorphic code (inlining, constant propagation, vectorization)
- Lowering to machine code

The consequence is that LLVM cannot perform polymorphism-aware optimizations such as "this function is the identity because its type is `∀α. α→α`" — it sees only the monomorphic instances. Whole-program reasoning at the type-theoretic level requires operating at the frontend IR (GHC Core, MIR, Clang's AST), before type erasure. Chapter 15 examines how this division manifests in LLVM IR's concrete type system and where MLIR's richer type model changes the boundary.

### The Implementation Continuum

| Language | Strategy | Code per type | Runtime overhead | Free theorems |
|----------|----------|---------------|------------------|---------------|
| Haskell | Dictionary passing (+ specialization) | One | Indirect call | Yes (for parametric) |
| OCaml | Type erasure (uniform representation) | One | None (boxed) | Yes |
| C++ templates | Monomorphization | Per instantiation | None (direct calls) | No (overloading) |
| Rust generics | Monomorphization | Per instantiation | None (direct calls) | Yes (trait bounds) |
| Rust `dyn Trait` | Fat pointer + vtable | One | Indirect call | Constrained |
| Java generics | Type erasure + boxing | One | Boxing / cast | No (wildcards) |

LLVM IR occupies the bottom of this stack: it always receives monomorphic code. Whether that code was produced by C++ template instantiation, Rust monomorphization, GHC specialization, or OCaml's uniform boxing is invisible to LLVM. The type-polymorphism of the source language is fully resolved before LLVM IR is generated. This is why LLVM IR has no notion of parametric polymorphism in its type system — the type-theoretic work of this chapter is done by the language frontends, and LLVM inherits only the concrete result.

---

## 12. Chapter Summary

- **Parametric polymorphism** lets a single term have type `∀α. τ`, working uniformly over all instantiations of `α`. Reynolds's parametricity theorem derives **free theorems** — equations that every implementation of a polymorphic type must satisfy — from the type alone, without examining the code.

- **System F** (the second-order lambda calculus) gives parametric polymorphism a denotational foundation via type abstraction (`Λα. M`) and type application (`M [τ]`). It encodes booleans, naturals, pairs, and all inductive types as Church encodings. Every well-typed System F term is strongly normalizing (Girard's proof via reducibility candidates). Full System F is impredicative, and type inference is undecidable for rank ≥ 3.

- **Existential types** `∃α. τ` model abstract data types: the representation type is hidden from the client. Existentials are definable in System F as `∀β. (∀α.τ→β)→β`. Rust's `impl Trait` and `dyn Trait` are the engineering manifestations of this theory.

- **Hindley-Milner** restricts to rank-1 (prenex) polymorphism with let-generalization, achieving decidable and complete type inference. Lambda-bound variables are monomorphic; only let-bound variables receive type schemes.

- **Algorithm W** computes principal types via unification (Robinson's algorithm with the occurs check) and generalization. A worked trace of `let f = λx.x in (f 1, f true)` shows how let-polymorphism permits two independent instantiations of `f`. The algorithm is near-linear in practice but exponential in the worst case on deeply nested let expressions.

- **The value restriction** is required for soundness in the presence of mutable references. Generalization is safe only for syntactic values (lambdas and constants). OCaml's relaxed value restriction permits more programs via variance analysis. The standard workaround is eta-expansion.

- **Principal types**: every HM-typeable term has a most-general type (principal type) that subsumes all other valid types. Algorithm W computes it. Languages with ad-hoc overloading (C++, Java wildcards) lack this property.

- **Bidirectional type checking** splits inference into synthesis (`⇒`) and checking (`⇐`) modes. Propagating expected types inward eliminates unification in many places, enables impredicative polymorphism, and produces better error messages. GHC, Rust, Scala Dotty, and TypeScript all use bidirectional algorithms. The DK algorithm (Dunfield-Krishnaswami 2013) is the theoretically complete treatment.

- **Row polymorphism** generalizes record types with row variables `ρ`, allowing functions to be polymorphic over the "rest" of a record's fields. OCaml polymorphic variants, Elm extensible records, and PureScript use this to give type-safe record operations.

- **Type classes** provide ad-hoc polymorphism with a principled elaboration: each class becomes a dictionary (record of functions), each instance provides a dictionary value, and each constrained call site receives the appropriate dictionary as an implicit argument. GHC's Core IR makes dictionaries explicit; by LLVM IR, they are ordinary struct pointer arguments. Coherence — at most one instance per type per class — is required for referential transparency.

- **Implementation continuum**: Haskell uses dictionary passing (with specialization to direct calls where possible); C++ and Rust use monomorphization (separate LLVM IR per instantiation); Java erases to `Object`. LLVM IR always receives fully monomorphic code — type polymorphism is a frontend responsibility, fully resolved before the LLVM optimizer sees the code.

---

*Further reading:* Pierce, *Types and Programming Languages* [TAPL], Chapters 22–25 (HM, System F, existentials, bounded quantification); Harper, *Practical Foundations for Programming Languages* [PFPL], Chapters 16–18; Damas and Milner, "Principal Type-Schemes for Functional Programs," POPL 1982; Wadler, "Theorems for Free!" FPCA 1989; Dunfield and Krishnaswami, "Complete and Easy Bidirectional Typechecking for Higher-Rank Polymorphism," ICFP 2013; Wright, "Simple Imperative Polymorphism," LISP and Symbolic Computation 1995.


---

@copyright jreuben11
