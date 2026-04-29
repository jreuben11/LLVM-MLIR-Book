# Chapter 12 — Lambda Calculus and Simple Types

*Part III — Type Theory*

Every type system in a modern compiler rests on foundations laid in the 1930s through the 1970s: Church's untyped lambda calculus, Curry and Howard's correspondence between types and proofs, and the syntactic safety theorems that crystallized in Wright and Felleisen's 1994 work. This chapter develops those foundations with the precision an expert audience requires. It is not a gentle introduction — it is the technical substrate for the three chapters that follow.

By the end of this chapter you will understand: the precise formal definition of the untyped lambda calculus and why naive substitution fails; the Church-Rosser confluence theorem and its proof strategy; why the untyped calculus diverges while the simply typed calculus is strongly normalizing; the typing rules of STLC and the proofs of preservation and progress that constitute type soundness; and the Curry-Howard correspondence that identifies programs with proofs. Chapter 13 extends STLC with polymorphism (System F and Hindley-Milner inference). Chapter 14 adds recursive types, dependent types, and refinement types. Chapter 15 connects this theoretical apparatus to the concrete type systems of LLVM IR and MLIR.

All formal material is grounded in the canonical references: Pierce, *Types and Programming Languages* [TAPL], MIT Press, 2002; Harper, *Practical Foundations for Programming Languages* [PFPL], 2nd ed., Cambridge, 2016; and Mitchell, *Foundations for Programming Languages* [FPL], MIT Press, 1996.

**Why this matters for compiler engineers.** The lambda calculus is not merely a theoretical toy. Its structural properties — confluence, normalization, and the Curry-Howard correspondence — underpin real engineering decisions:

- **Confluence** explains why the order of independent optimizations does not matter: if two passes both perform semantics-preserving transformations (analogous to β-steps), the result is the same regardless of order. LLVM's pass manager assumes passes commute when they address orthogonal concerns; this is the engineering application of confluence.

- **Strong normalization for typed terms** is the theorem behind the termination checks in Rust's type system, the totality checker in Agda, and the termination oracle in Coq. When a compiler must guarantee termination (e.g., for hardware synthesis or safety-critical embedded software), it is enforcing a type-theoretic SN property.

- **Preservation and Progress** are the formal specification of "no runtime type errors." Every typed language's correctness argument eventually reduces to these two properties — whether stated explicitly (as in research papers on Rust's type system or Java's JVM specification) or implicitly (as in the informal argument that C++'s type system prevents certain misuses of values).

- **The Curry-Howard correspondence** is the intellectual foundation for proof assistants. When LLVM's correctness is argued via Vellvm (Chapter 169) or a transformation's correctness via Alive2 (Chapter 170), those arguments inhabit the type-theoretic world this chapter describes.

---

## 1. Untyped Lambda Calculus: Syntax and Variables

### The Grammar

The untyped lambda calculus is built from exactly three syntactic forms [TAPL §5.1]:

```
M, N, L  ::=  x            (variable)
           |  λx. M        (abstraction)
           |  M N          (application)
```

The metavariables `M`, `N`, `L` range over terms; `x`, `y`, `z` range over variables drawn from a countably infinite set *Var*. This is the complete syntax. There are no constants, no base types, no built-in operations — everything must be encoded. (When we want base types we add them; the resulting calculus is still called pure once we restrict attention to the λ-fragment.)

Notation conventions: application associates to the left, so `M N L` means `(M N) L`. The body of a λ extends as far right as possible: `λx. M N` means `λx. (M N)`, not `(λx. M) N`. Multiple-argument functions are encoded via currying: `λx. λy. M` is commonly written `λxy. M`.

### Free and Bound Variables

The occurrence of a variable `x` in a term is either *bound* (if it falls within the scope of a binder `λx. _`) or *free* (if no enclosing binder claims it). The sets of free and bound variables are defined inductively [TAPL §5.1, PFPL §1.1]:

```
FV(x)        = {x}
FV(λx. M)   = FV(M) \ {x}
FV(M N)     = FV(M) ∪ FV(N)

BV(x)        = ∅
BV(λx. M)   = BV(M) ∪ {x}
BV(M N)     = BV(M) ∪ BV(N)
```

A term is *closed* (also called a *combinator*) if `FV(M) = ∅`. Closed terms are the programs of the lambda calculus; they can be evaluated without any external environment.

Note that the same variable name can appear in both free and bound positions in the same term. In `(λx. x) x`, the `x` under the binder is bound, while the trailing `x` is free. This is syntactically well-formed but conceptually confusing — the Barendregt convention addresses it.

### The Barendregt Variable Convention

Throughout this chapter (and throughout the lambda-calculus literature) we adopt the **Barendregt convention** [TAPL §5.1, Barendregt 1984]: whenever we write a term, we choose variable names so that all bound variables are distinct from each other and from all free variables. The convention is justified by the α-equivalence of terms (Section 2.1): we can always rename bound variables without changing the meaning of a term, so we choose names to avoid clashes. With this convention, the pathological case `(λx. x) x` would be rewritten with a fresh binder name, say `(λy. y) x`.

The Barendregt convention does not change the theory — it is a notational convenience that eliminates tedious case-splits in proofs about substitution. We will occasionally depart from it explicitly when illustrating the precise problem it solves.

### De Bruijn Indices

An alternative representation that eliminates variable names entirely is **de Bruijn indices** [de Bruijn 1972; TAPL §6]. In the de Bruijn representation, a variable is replaced by a natural number indicating how many enclosing λ-binders separate it from its binding site:

```
λx. λy. x   becomes   λ. λ. 1
λx. λy. y   becomes   λ. λ. 0
λx. λy. x y becomes   λ. λ. 1 0
```

The innermost binder is index 0; each additional enclosing binder increments the index. Under de Bruijn indices, α-equivalent terms are literally identical — there is no α-equivalence relation, just syntactic equality. Substitution requires a *shift* operation to prevent index capture, but the resulting formalism is uniform and directly implementable. LLVM and MLIR use name-based representations internally; de Bruijn indices appear in proof assistants (Coq's kernel, Lean 4, Agda) where definitional equality must be computable. We use de Bruijn as background knowledge but work with named variables throughout this chapter, relying on the Barendregt convention for hygiene.

### Church Encodings

Even without built-in data types, the untyped lambda calculus can encode booleans, natural numbers, pairs, and lists — demonstrating its computational universality [TAPL §5.2, §5.3].

**Church booleans:**
```
true  = λt. λf. t
false = λt. λf. f
if    = λb. λt. λe. b t e
not   = λb. b false true
and   = λb. λc. b c false
or    = λb. λc. b true c
```

Here `if M then N else L` is represented as `(λb.λt.λe. b t e) M N L`, which reduces to `M N L`. When `M = true = λt.λf.t`, the result is `N`; when `M = false = λt.λf.f`, the result is `L`. This is selection-by-function-application.

**Church numerals:**
```
c₀ = λf. λx. x                  (zero)
c₁ = λf. λx. f x                (one)
c₂ = λf. λx. f (f x)            (two)
cₙ = λf. λx. f ⁿ x              (n)

succ  = λn. λf. λx. f (n f x)
plus  = λm. λn. λf. λx. m f (n f x)
times = λm. λn. m (plus n) c₀
iszero = λn. n (λx. false) true
```

The numeral `cₙ` represents `n` by taking a function `f` and applying it `n` times to `x`. Addition applies `f` `m+n` times; multiplication composes the `n`-fold application `m` times. The predecessor function exists but is considerably more complex (due to Kleene); it is a beautiful exercise in encoded computation. In the simply typed lambda calculus, Church numerals have type `(τ → τ) → τ → τ` for each `τ` — this polymorphism is central to System F's treatment of arithmetic, covered in Chapter 13.

**Church pairs:**
```
pair  = λf. λs. λb. b f s
fst   = λp. p true
snd   = λp. p false
```

`pair M N` encodes the pair `(M, N)` as a function that selects either component using a boolean. `fst p` selects the first component; `snd p` the second. This encoding demonstrates that products are definable from functions and booleans in the untyped calculus.

These encodings are theoretically significant but practically limited: they work in the untyped calculus and in System F (Chapter 13), but are *not* how real compilers represent data. LLVM IR uses struct types and GEP instructions (Chapter 16); MLIR uses dialect-specific types (Part XX). The encodings matter because they prove that the lambda calculus is Turing-complete — every computable function has a lambda-calculus representation — and because they illuminate what type structure is needed to make various constructs first-class.

---

## 2. Substitution and α-Equivalence

### Capture-Avoiding Substitution

The operation `M[x := N]` — read "M with N substituted for x" — replaces every free occurrence of `x` in `M` with `N`. The definition must be *capture-avoiding*: when substituting `N` into the body of a λ-abstraction whose binder happens to use a free variable of `N`, we must rename the binder before proceeding [TAPL §5.2]:

```
x[x := N]         = N
y[x := N]         = y                         (y ≠ x)
(M₁ M₂)[x := N]  = (M₁[x := N]) (M₂[x := N])
(λy. M)[x := N]  = λy. (M[x := N])            (y ≠ x, y ∉ FV(N))
(λy. M)[x := N]  = λz. (M[y := z][x := N])    (y ≠ x, y ∈ FV(N), z fresh)
```

The fourth clause handles the routine case: the binder `y` is not `x` and does not conflict with `N`'s free variables. The fifth clause is the critical one: if `y` occurs free in `N`, then blindly substituting under `λy. M` would *capture* that free occurrence of `y`, changing its meaning. The fix is to α-rename the binder to a fresh variable `z` before substituting.

**The canonical failure example.** Consider the term `M = λy. x` and the substitution `M[x := λz. yz]`. We are replacing the free `x` in `(λy. x)` with the term `(λz. yz)`, which has `y` free.

If we naively applied the fourth clause (ignoring that `y ∈ FV(λz. yz)`):

```
(λy. x)[x := λz. yz]  ↝ (wrong)  λy. (λz. yz)
```

In this wrong result, the `y` that was free in the substituend `λz. yz` has been captured by the binder `λy`, transforming it from a free occurrence to a bound one. The function `λz. yz` now refers to the parameter of the outer abstraction rather than whatever `y` was in scope externally — a semantically different term.

The capture-avoiding rule renames `y` to a fresh variable `w` before substituting:

```
(λy. x)[x := λz. yz]
  = (λw. x[y := w])[x := λz. yz]   (rename y → w, w fresh)
  = λw. (x[y := w])[x := λz. yz]
  = λw. x[x := λz. wz]             (y renamed to w in substituend too)
  = λw. λz. wz
```

This result correctly represents a function that takes a `w` and returns `λz. wz` — equivalent to what we intended. (Under the Barendregt convention the renaming step is handled automatically by the choice of fresh names; the point of the example is to show what goes wrong if you don't apply the convention.)

### α-Equivalence

Two terms are **α-equivalent** (written M ≡_α N) if they differ only in the names of bound variables. The relation is defined as the smallest congruence satisfying [TAPL §5.1]:

```
λx. M  ≡_α  λy. M[x := y]     (y ∉ FV(M), y ≠ x)
```

plus reflexivity, symmetry, transitivity, and compatibility with application and abstraction. Thus:

```
λx. x       ≡_α  λy. y        ≡_α  λz. z
λx. λy. x y ≡_α  λa. λb. a b
```

We work with terms *up to α-equivalence* throughout: when we write `M = N` we mean α-equivalence unless specified otherwise. The Barendregt convention makes α-equivalence implicit — we simply choose fresh names upfront. Under de Bruijn indices, α-equivalence collapses to syntactic equality.

**α-equivalence as a quotient.** Formally, the set of lambda terms is the quotient of the raw syntax (the free algebra over the grammar of Section 1) by the congruence generated by the α-rule above. Working with this quotient avoids the need to ever consider terms that differ only in bound-variable names. Every equivalence class has representatives satisfying the Barendregt convention; we work with those representatives by convention.

**Scope and shadowing.** The α-rule applies only to *bound* occurrences. If `x` appears both free and bound in a term, α-renaming the binder renames only the bound occurrences:

```
(λx. x) x  ≡_α  (λy. y) x      (only the bound x becomes y; the free x remains)
```

This is not surprising but worth stating: α-equivalence respects the syntactic scoping structure of the term. In implementations, compilers ensure α-equivalence by SSA (Static Single Assignment) form for imperative languages, or by fresh-name generation for functional languages. Chapter 21 (SSA, Dominance, and Loops) shows how LLVM IR's SSA invariant — each value has exactly one definition site — eliminates the practical need for α-renaming in the compiler backend.

### The Substitution Lemma (Untyped)

The following lemma about the interaction of two substitutions is used repeatedly in proofs [TAPL §5.3]:

> **Lemma (Substitution Commutation):** If `x ≠ y` and `x ∉ FV(L)`, then:
> `M[x := N][y := L] ≡_α M[y := L][x := N[y := L]]`

This says we can perform two substitutions in either order provided we also apply the second substitution to the first substituend when needed. The typed version of this lemma (the *Substitution Lemma* for STLC) is stated in Section 7.

**Substitution and complexity.** Naively implemented, substitution has worst-case quadratic complexity in the size of the term (if the substituted variable appears in many places). This is irrelevant for the metatheory but matters enormously for efficient lambda-calculus interpreters. Practical implementations use environments (closures) rather than literal substitution: instead of substituting `N` for `x` everywhere in `M`, they create a closure `(λx.M, ρ)` where `ρ` maps `x` to the unevaluated term `N` (in CBN) or to its evaluated form (in CBV). Evaluation of variables then looks up the environment. This is the standard implementation technique in OCaml, Haskell, and all other compiled functional languages; it transforms the exponential worst case of naive substitution into a constant-time lookup. The machinery of closures and environments is the implementation counterpart of the formal substitution operation — a bridge from the theory of this chapter to the practice of compiler construction.

---

## 3. β-Reduction and Reduction Strategies

### β-Reduction

The computational rule of the lambda calculus is **β-reduction** [TAPL §5.2, PFPL §2.1]:

```
(λx. M) N  →_β  M[x := N]
```

A term of the form `(λx. M) N` — a lambda abstraction applied to an argument — is called a **β-redex** (from "reducible expression"). The result `M[x := N]` is its **contractum**. A term with no β-redexes is in **β-normal form**.

β-reduction is closed under evaluation contexts: if `M →_β M'` then:
- `M N →_β M' N` (reduce the function position)
- `N M →_β N M'` (reduce the argument position)
- `λx. M →_β λx. M'` (reduce under a binder)

The *reflexive-transitive closure* of `→_β` is written `→*_β`: `M →*_β N` means zero or more β-steps from `M` to `N`.

**Example.** The identity function `I = λx. x` applied to itself:

```
(λx. x)(λy. y)
→_β (λy. y)         [M[x := λy.y] = λy.y]
```

A more interesting example, with a function that applies its argument to itself:

```
(λx. x x)(λy. y)
→_β (λy. y)(λy. y)
→_β λy. y
```

### η-Reduction

The companion rule **η-reduction** captures extensionality [TAPL §5.3]:

```
λx. M x  →_η  M     (x ∉ FV(M))
```

If `x` does not appear free in `M`, then `λx. M x` and `M` are *extensionally equal*: they produce the same result for every argument. η-reduction is the formal embodiment of this equivalence. In a pure extensional setting, `→_β` and `→_η` can be combined into `→_βη`. For most theoretical purposes (in particular, for type safety in STLC) we work with β only; η is relevant for categorical models (cartesian closed categories) and for some proof-normalization arguments. PFPL §1.2 discusses the extensional view in detail.

### Reduction Strategies

A given term may contain multiple redexes. A **reduction strategy** specifies which redex to reduce at each step [TAPL §5.2, PFPL §2.2]:

**Full β-reduction** allows any redex to be chosen at any step. It is the most permissive strategy and is used in metatheory (the Church-Rosser theorem is about full β-reduction).

**Normal-order reduction** (leftmost-outermost, often called "call-by-name" in a loose sense) always selects the *leftmost-outermost* redex: the one whose λ is furthest to the left in the term, with no enclosing β-redex. Normal-order reduction will find a β-normal form whenever one exists; it is the strategy used to state the Church-Rosser corollary about normal forms.

**Call-by-name (CBN)** is leftmost-outermost *excluding* redexes under λ-binders. It never reduces inside an abstraction. In PFPL's terminology, it reduces the leftmost-outermost redex among those that are not under a `λ`. Arguments are substituted unevaluated; a function body is not entered until the function is applied. Non-strict: it never evaluates arguments that are not needed. Haskell's theoretical foundation uses a refined version called call-by-need.

**Call-by-value (CBV)** reduces the argument to a value before substituting. A *value* in the pure lambda calculus is an abstraction `λx. M` (in extensions with constants, also constant literals). The strategy is: find the leftmost-innermost redex `(λx. M) V` where `V` is already a value, and reduce it. Equivalently: always reduce the argument before applying the function. CBV is the evaluation model of ML, OCaml, Rust, C, Java, and most mainstream languages.

**Call-by-need (lazy evaluation)** is CBN with *sharing*: when an argument is substituted into a body, all occurrences of `x` in `M` share the same unevaluated argument; once one occurrence forces evaluation, the result is memoized. This is the model underlying Haskell. Lazy evaluation is observationally equivalent to CBN (same results, if termination is disregarded) but more efficient in practice because arguments are evaluated at most once.

**Strategy and termination.** The choice of strategy matters for termination:

- Normal-order always finds a normal form *if one exists*. It is the "most normalizing" strategy.
- CBV may diverge on terms that have a normal form: consider `(λx. I)((λy. y y)(λy. y y))`. CBV evaluates the argument first, which diverges. Normal-order substitutes the unevaluated argument and immediately produces `I = λx.x`.
- CBN may evaluate terms that CBV would diverge on, but it may do redundant work (evaluate the same argument multiple times in different positions).

This sensitivity to strategy is precisely why the untyped lambda calculus is interesting and why the Church-Rosser theorem's uniqueness-of-normal-form result is important: when a normal form exists, it is unique regardless of which strategy found it (or failed to find it, in CBV's case).

**Reduction contexts.** Formally, each strategy can be specified using **evaluation contexts** (also called *reduction contexts*) — term grammars with a hole `[]` indicating the next redex site [TAPL §5.2]. For CBV (where arguments are reduced before application):

```
E  ::=  []              (the hole; current focus of reduction)
     |  E N             (reduce the function position)
     |  V E             (reduce the argument position, once function is a value)
```

A *redex* `(λx.M) V` is contracted only when `V` is a value. In the full reduction relation `E[(λx.M)V] →_β E[M[x:=V]]`, the context `E` specifies the unique next step. This context-based formulation is standard in programming language semantics (Pierce uses it starting TAPL §3; Harper uses it throughout PFPL) because it cleanly separates the *what* (the β-rule) from the *where* (the evaluation context).

---

## 4. Church-Rosser Confluence

### The Diamond Property and Confluence

A reduction relation `→` is **confluent** (satisfies the **Church-Rosser property**) if whenever `M →* N₁` and `M →* N₂`, there exists `P` such that `N₁ →* P` and `N₂ →* P` [TAPL §3.5, §8.3]. In diagram form:

```
       M
      / \
    *↓   ↓*
    N₁   N₂
      \ /
      ↓*↓*
       P
```

The stronger **diamond property** says the same with single-step reduction: if `M → N₁` and `M → N₂` then there exists `P` with `N₁ → P` and `N₂ → P`. β-reduction does not satisfy the diamond property directly (a term can have two overlapping redexes requiring more than one step to reconcile), but it is confluent.

### The Church-Rosser Theorem

> **Theorem (Church-Rosser, 1936):** β-reduction `→_β` is confluent: if `M →*_β N₁` and `M →*_β N₂`, then there exists `P` with `N₁ →*_β P` and `N₂ →*_β P` [TAPL §3.5].

**Proof strategy.** A direct proof by induction on reduction length fails because the diamond property does not hold step-by-step. The standard proof, due to Tait and Martin-Löf, uses an intermediate relation called **parallel reduction** [TAPL Exercise 8.3.10; PFPL §2.3]:

```
M →_p M'   (M parallel-reduces to M')
```

defined by the simultaneous contraction of zero or more non-overlapping redexes. The rules are:

```
────────    (PVar)
x →_p x

  M →_p M'
──────────────    (PAbs)
λx.M →_p λx.M'

 M →_p M'    N →_p N'
──────────────────────    (PApp)
    MN →_p M'N'

M →_p M'    N →_p N'
──────────────────────────────    (PBeta)
(λx.M)N →_p M'[x := N']
```

Note that PBeta contracts a redex while also reducing inside both `M` and `N`; PApp reduces both parts of an application without contracting the outermost redex.

**Key lemma (parallel reduction has the diamond property):**

> If `M →_p N₁` and `M →_p N₂`, then there exists `P` with `N₁ →_p P` and `N₂ →_p P`.

This is proved by induction on the structure of `M`. The cases are systematic; the interesting one is when `M = (λx. M₀) M₁` and one derivation uses PBeta while the other uses PApp. In that case both `N₁` and `N₂` are of the form "something applied to something", and one can complete the diamond by doing the contraction that the other strategy postponed.

**Completing the proof.** Parallel reduction is sandwiched between single-step and multi-step β-reduction:

```
(→_β)  ⊆  (→_p)  ⊆  (→*_β)
```

The left inclusion: any single β-step is a parallel step (using PBeta with identity steps inside). The right inclusion: parallel reduction is a subrelation of multi-step β. The diamond property for `→_p` implies confluence of `→*_p`, which by the inclusions implies confluence of `→*_β`.

**The critical pair.** The key case in the diamond lemma for `→_p` is when `M = (λx.M₀)M₁` and we have two parallel reductions:
- `N₁` arises from PBeta: `N₁ = M₀'[x := M₁']` where `M₀ →_p M₀'` and `M₁ →_p M₁'`.
- `N₂` arises from PApp: `N₂ = (λx.M₀'')M₁''` where `M₀ →_p M₀''` and `M₁ →_p M₁''`.

Note that `N₂` has not yet contracted the redex; `N₁` has. To close the diamond, we need `P` with `N₁ →_p P` and `N₂ →_p P`. Since `M₀ →_p M₀'` and `M₀ →_p M₀''` (both are parallel reductions of `M₀`), by the induction hypothesis (IH on `M₀`) there exists `P₀` with `M₀' →_p P₀` and `M₀'' →_p P₀`. Similarly for `M₁`. Then:
- From `N₁ = M₀'[x:=M₁']`: use PBeta with `M₀' →_p P₀` and `M₁' →_p P₁` to get `N₁ →_p P₀[x:=P₁]`.
- From `N₂ = (λx.M₀'')M₁''`: use PBeta with `M₀'' →_p P₀` and `M₁'' →_p P₁` to get `N₂ →_p P₀[x:=P₁]`.

So `P = P₀[x:=P₁]` closes the diamond. The proof relies on a substitution lemma for parallel reduction: if `M →_p M'` and `N →_p N'` then `M[x:=N] →_p M'[x:=N']`. This holds by induction on `M →_p M'`.

### Consequences of Confluence

**Uniqueness of normal forms.** If `M` has two β-normal forms `N₁` and `N₂`, then by Church-Rosser there exists `P` with `N₁ →*_β P` and `N₂ →*_β P`. But normal forms have no redexes, so `N₁ = P = N₂`. Therefore *every term has at most one β-normal form* [TAPL §3.5].

**Consistency of the calculus.** No two distinct normal forms are β-equal. In particular, `λx.x ≢_β λx.λy.y` — the two cannot be proved equal by any sequence of β-steps. This shows the untyped lambda calculus is not trivially inconsistent.

**Independence of strategy for normal forms.** If a term has a normal form and a particular strategy finds it, the normal form found must be the unique one guaranteed by Church-Rosser. Different strategies may take different paths but, when they terminate, produce identical results.

**βη-confluence.** The βη-reduction relation (closing β and η under the same rules) is also confluent. The proof is analogous, using parallel βη-reduction. This is relevant for the category-theoretic interpretation of the lambda calculus in cartesian closed categories, where βη-equality is definitional equality.

**Standardization theorem.** A stronger result than confluence is the **standardization theorem** [Curry and Feys 1958; TAPL §11.5 notes]: if `M →*_β N`, then `M` can reach `N` via a *standard reduction sequence* — one in which redexes are always reduced from left to right, outermost first. Standardization implies that normal-order reduction reaches every reachable normal form and justifies the claim that CBN is "complete" for normalization. The proof is substantially more involved than Church-Rosser, using a combinatorial argument about positions of redexes.

---

## 5. Normalization: Weak, Strong, and the Ω Divergent Term

### Weak and Strong Normalization

A term `M` is **weakly normalizing (WN)** if there *exists* a reduction sequence from `M` that terminates in a normal form [TAPL §12.1]. Weak normalization says "a normal form is reachable" without guaranteeing that all sequences terminate.

A term `M` is **strongly normalizing (SN)** if *every* reduction sequence from `M` terminates. Strong normalization says no matter what choices are made during evaluation, the term eventually reduces to a normal form — there are no infinite reduction sequences.

SN implies WN trivially. The converse fails, as we will see.

### The Ω Combinator: Divergence in the Untyped Calculus

The canonical diverging term is **Ω** (omega) [TAPL §5.2]:

```
ω  =  λx. x x
Ω  =  ω ω  =  (λx. x x)(λx. x x)
```

Reducing Ω:

```
Ω = (λx. x x)(λx. x x)
  →_β (x x)[x := λx.x x]
  = (λx. x x)(λx. x x)
  = Ω
```

Ω reduces to itself in one step and loops forever. It has no β-normal form. It witnesses that the untyped lambda calculus is not weakly normalizing: Ω is WN-false (no normal form exists). It is also SN-false (every reduction sequence is infinite).

### Normalizable Terms with Non-Terminating Strategies

The untyped lambda calculus contains terms that *are* weakly normalizing but can still produce infinite reduction sequences under the wrong strategy [TAPL §5.2]. Consider:

```
K  =  λx. λy. x        (the "constant" combinator)
Δ  =  KΩ  =  (λx. λy. x) Ω
```

Under normal-order (CBN) strategy:

```
Δ = (λx. λy. x) Ω
  →_β  λy. Ω[x := Ω]   -- wait, the body is x, substituting: λy. x[x := Ω]
  = λy. Ω
```

Wait — `K = λx.λy.x`, so `KΩ = (λx.λy.x)Ω`. The body of the outermost λ is `λy.x`; substituting Ω for x:

```
KΩ = (λx. λy. x) Ω  →_β  (λy. x)[x := Ω]  =  λy. Ω
```

`λy. Ω` is a normal form (it is a λ-abstraction; the `Ω` inside is not a redex because we are in CBN which does not reduce under binders). So CBN terminates.

Under CBV strategy, we must first reduce the argument `Ω` to a value before applying `K`. But `Ω` has no normal form, so CBV diverges on `KΩ` even though a normal form exists. This is the content of the claim above: the untyped lambda calculus is not SN, and CBV does not find normal forms that exist.

The Y combinator further illustrates non-termination:

```
Y = λf. (λx. f(x x))(λx. f(x x))
```

For any `f`:

```
Yf = (λf. (λx. f(xx))(λx. f(xx))) f
   →_β (λx. f(xx))(λx. f(xx))
   →_β f((λx. f(xx))(λx. f(xx)))
   →_β f(Yf)
```

So `Yf →*_β f(Yf)` — the Y combinator computes the *fixed point* of `f`. Every function has a fixed point in the untyped lambda calculus. The reduction sequence `Yf → f(Yf) → f(f(Yf)) → ...` is infinite unless `f` ignores its argument (in which case the fixed point is reached in one step from the appropriate starting point).

**Undefinability of Y in STLC.** The Y combinator cannot be assigned a type in the simply typed lambda calculus. To see why: suppose `Y : (τ → τ) → τ` for some type `τ`. Then the inner term `λx. f(xx)` must have some type; `x` must be given a type `σ` such that `σ = σ → τ` (because `x` is applied to itself: `xx` requires `x : σ → τ` and also `x : σ`, hence `σ = σ → τ`). No finite type `σ` satisfies `σ = σ → τ` in STLC's tree-structured types. This is not a fixable notational problem — STLC has no recursive types. The undefinability of Y in STLC is not a defect but a virtue: it is precisely *why* STLC is strongly normalizing. Adding fixpoints (via a `fix` combinator or recursive types as in Chapter 14) restores the ability to write non-terminating programs.

**The Turing and Curry fixed-point combinators.** The Y combinator is one of infinitely many fixed-point combinators. Another is the Turing fixed-point combinator, due to Alan Turing:

```
Θ = (λx. λf. f (x x f))(λx. λf. f (x x f))
```

`Θ f →*_β f (Θ f)` — it computes the same fixed point as Y but via a different reduction sequence. Turing's version has the property that `Θ f →_β f (Θ f)` in *one* step, whereas `Y f →_β →_β f(Yf)` requires two. Neither is typeable in STLC for the same self-application reason as Y.

**Fixed points in typed systems.** Languages with recursive types can type a fixed-point operator. If we add a recursive type `μα. α → τ` (a type `α` satisfying `α = α → τ` by definition of the μ-type), then the self-application `x x` becomes typeable. With an explicit `fix` operator (as in Mitchell-Plotkin-style recursive types, TAPL §20), we can write:

```
fix : (τ → τ) → τ
fix f = f (fix f)
```

This is a *primitive* constant with a reduction rule `fix f →_β f (fix f)`, not a term in STLC — it is an extension. In real compilers: Haskell's `fix` in `Data.Function` is exactly this; OCaml's `let rec` desugars to a form of fixed-point; LLVM IR represents recursion via direct tail calls or back edges in control flow, bypassing the typing problem entirely by operating below the lambda abstraction layer.

---

## 6. Simply Typed Lambda Calculus: Types and Typing Rules

### Types

The **simply typed lambda calculus (STLC)** equips the lambda calculus with a simple type system. Types are defined by the grammar [TAPL §9.1]:

```
τ, σ  ::=  B        (base type, e.g., Bool, Nat, ...)
         |  τ → σ   (function type)
```

Base types `B` are uninterpreted placeholders in the pure STLC; in extensions they are replaced by concrete types (booleans, naturals, etc.). The function type `τ → σ` associates to the right: `τ₁ → τ₂ → τ₃` means `τ₁ → (τ₂ → τ₃)`.

### Extensions: Products, Sums, and Base Types

The pure STLC defined above is minimal. In practice, type systems extend it with:

**Base types with constants.** Adding `Bool` with constants `true`, `false : Bool` and a conditional `if-then-else`:

```
M  ::=  ...  |  true  |  false  |  if M then M else M
```

with typing rule:

```
Γ ⊢ M : Bool    Γ ⊢ N₁ : τ    Γ ⊢ N₂ : τ
────────────────────────────────────────────     (T-If)
Γ ⊢ if M then N₁ else N₂ : τ
```

and evaluation rules `if true then V₁ else V₂ →_β V₁` and `if false then V₁ else V₂ →_β V₂`. Adding `Nat` with `zero`, `succ`, and a primitive recursor gives the system of TAPL §9.

**Product types** `τ × σ` with pairs `(M, N)`, projections `π₁` and `π₂`:

```
Γ ⊢ M : τ    Γ ⊢ N : σ            Γ ⊢ M : τ × σ           Γ ⊢ M : τ × σ
───────────────────────────      ──────────────────      ──────────────────
Γ ⊢ (M, N) : τ × σ              Γ ⊢ π₁(M) : τ           Γ ⊢ π₂(M) : σ
```

**Sum types** `τ + σ` with injections and case analysis:

```
Γ ⊢ M : τ                       Γ ⊢ N : σ
─────────────────────────     ─────────────────────────
Γ ⊢ inl M : τ + σ             Γ ⊢ inr N : τ + σ

Γ ⊢ M : τ + σ    Γ, x:τ ⊢ N₁ : ρ    Γ, y:σ ⊢ N₂ : ρ
──────────────────────────────────────────────────────────     (T-Case)
Γ ⊢ case M of (inl x ⇒ N₁ | inr y ⇒ N₂) : ρ
```

These extensions are conservative: they add new forms without changing the structure of function types. All the theorems of this chapter (confluence, SN, Preservation, Progress) extend straightforwardly to STLC with products and sums, by adding cases to the proofs. Chapter 14 treats the full hierarchy; we note them here because they complete the Curry-Howard table in Section 8.

### Church-Style vs. Curry-Style

There are two presentations of STLC [TAPL §9.1, Mitchell §5]:

**Church-style (explicitly typed):** Binders carry type annotations: `λx:τ. M`. The type of every term is determined by its syntax alone. Given `Γ` and `M`, if `M` has a type at all, it has exactly one type. This is the presentation we use here — it corresponds directly to static type checking in compilers.

**Curry-style (implicitly typed):** Binders are unannotated: `λx. M`. The same term may be assigned many types (by different derivations). A term like `λx. x` has type `τ → τ` for *every* `τ`. In Curry-style STLC, the typing relation is a *ternary* judgment `Γ ⊢ M : τ` where `(M, τ)` pairs are in a many-to-many relationship. Curry-style is the basis for Hindley-Milner type inference (Chapter 13).

**This chapter uses Church-style.** Typing derivations in Church-style STLC are unique given `Γ` and `M` — a fact we prove below.

### Typing Contexts and the Typing Judgment

A **typing context** (also called a type environment or type assignment) `Γ` is a finite sequence of variable-type bindings:

```
Γ  ::=  ∅  |  Γ, x:τ     (x not already in dom(Γ))
```

We write `dom(Γ)` for the set of variables bound in `Γ`. The **typing judgment** `Γ ⊢ M : τ` ("under context `Γ`, term `M` has type `τ`") is defined by the following inference rules [TAPL §9.1]:

```
──────────────────────     (T-Var)
Γ, x:τ ⊢ x : τ

Γ, x:τ₁ ⊢ M : τ₂
──────────────────────     (T-Abs)
Γ ⊢ λx:τ₁. M : τ₁ → τ₂

Γ ⊢ M : τ₁ → τ₂    Γ ⊢ N : τ₁
────────────────────────────────     (T-App)
       Γ ⊢ M N : τ₂
```

(T-Var) says that a variable `x` has whatever type `Γ` assigns to it. (T-Abs) says that to type an abstraction `λx:τ₁. M`, extend the context with `x:τ₁` and check that `M` has type `τ₂`; the whole abstraction then has type `τ₁ → τ₂`. (T-App) says that if `M` has a function type `τ₁ → τ₂` and `N` has the domain type `τ₁`, then `M N` has the result type `τ₂`. This is *modus ponens* — we return to this in Section 8.

**Example typing derivation.** Let us type the term `λf:Nat→Nat. λx:Nat. f x`:

```
                              ──────────────────────────────────── (T-Var)
                              f:Nat→Nat, x:Nat ⊢ f : Nat→Nat
                                                                          ──────────────────────── (T-Var)
                                                                          f:Nat→Nat, x:Nat ⊢ x : Nat
    ────────────────────────────────────────────────────────────────────────────────────────────────── (T-App)
    f:Nat→Nat, x:Nat ⊢ f x : Nat
    ──────────────────────────────────────────────────────────────── (T-Abs)
    f:Nat→Nat ⊢ λx:Nat. f x : Nat → Nat
    ──────────────────────────────────────────────────────────────────────────── (T-Abs)
    ∅ ⊢ λf:Nat→Nat. λx:Nat. f x : (Nat→Nat) → Nat → Nat
```

The type `(Nat→Nat) → Nat → Nat` is the type of the "higher-order identity on `Nat→Nat`" — a function that takes an endofunction on `Nat` and an input, and applies the endofunction to the input.

**Second worked derivation.** Let us also type function composition `compose = λf:σ→ρ. λg:τ→σ. λx:τ. f(g x)` for fixed types `τ`, `σ`, `ρ`:

```
Derivation of f:σ→ρ, g:τ→σ, x:τ ⊢ f(g x) : ρ:

──────────────────────── (T-Var)          ──────────────────────── (T-Var)    ─────────────────── (T-Var)
f:σ→ρ,g:τ→σ,x:τ ⊢ f:σ→ρ                g:τ→σ,x:τ... ⊢ g:τ→σ               ...x:τ ⊢ x:τ
                                         ──────────────────────────────────────────────────────── (T-App)
                                         f:σ→ρ, g:τ→σ, x:τ ⊢ g x : σ
──────────────────────────────────────────────────────────────────────────────────────────── (T-App)
f:σ→ρ, g:τ→σ, x:τ ⊢ f(g x) : ρ
────────────────────────────────────────────────── (T-Abs × 3)
∅ ⊢ λf:σ→ρ. λg:τ→σ. λx:τ. f(g x) : (σ→ρ) → (τ→σ) → τ → ρ
```

This is the standard type of function composition, corresponding under Curry-Howard to the hypothetical syllogism rule of logic: `(σ⊃ρ) → (τ⊃σ) → (τ⊃ρ)`. The type derivation uniquely determines the type; any other assignment of types to the sub-expressions would be inconsistent with the typing rules.

### Type Uniqueness

> **Theorem (Type Uniqueness):** In Church-style STLC, if `Γ ⊢ M : τ` and `Γ ⊢ M : σ`, then `τ = σ` [TAPL §9.2].

*Proof.* By induction on the typing derivation.

- *Case T-Var:* `M = x`. The context `Γ` contains at most one binding for `x` (by the well-formedness condition that dom bindings are distinct). So `τ = σ = Γ(x)`.

- *Case T-Abs:* `M = λx:τ₁. M'`. The type derived is `τ₁ → τ₂` for some `τ₂` determined uniquely by the premise `Γ, x:τ₁ ⊢ M' : τ₂`. By the induction hypothesis, `τ₂` is unique, so `τ₁ → τ₂` is unique.

- *Case T-App:* `M = M₁ M₂`. The derivation must use T-App, yielding `τ₁ → τ` and `τ₁` from two sub-derivations. The types of `M₁` and `M₂` are unique by the induction hypothesis, determining `τ` uniquely. □

Type uniqueness is a fundamental property of Church-style simply typed calculi. It fails in Curry-style (where polymorphism is implicit) and in systems with subtyping (where a term can have many types related by ≤). Both exceptions are pursued in Chapter 13 and Chapter 14.

### Strong Normalization of STLC

> **Theorem (Strong Normalization for STLC):** Every well-typed term of STLC is strongly normalizing — every reduction sequence terminates [TAPL §12.1, PFPL §47].

This is a profound result: typing *guarantees* termination. The proof uses **Tait's method of reducibility candidates** (also called logical predicates or reducibility predicates) [Tait 1967; Girard 1972]. A naive induction on the term or type fails because the inductive hypothesis is not strong enough. The fix is to define a *semantic* predicate `Red_τ` for each type `τ`:

```
Red_B(M)        iff   M is strongly normalizing   (for base type B)
Red_{τ→σ}(M)   iff   for all N, Red_τ(N) implies Red_σ(M N)
```

`Red_τ(M)` is a stronger statement than "M is SN"; it says M is SN *and* its applications to reducible arguments are reducible. The proof then proceeds in two parts:

1. **Closure under reduction:** If `M →_β M'` and `Red_τ(M)` then `Red_τ(M')`. (Reduction preserves reducibility.)

2. **Adequacy:** If `Γ ⊢ M : τ` and `Red_σ(N)` for each `x:σ` in `Γ`, then `Red_τ(M[x₁:=N₁,…])`. (Well-typed terms are reducible under reducible substitutions.)

The SN conclusion follows because `Red_τ(M)` implies `M` is SN (by induction on `τ`), and adequacy applied to the identity substitution (where each `Ni = xi`, which are vacuously SN variables) gives `Red_τ(M)` for any closed `⊢ M : τ`.

The reducibility method generalizes to more powerful systems: it is the same technique used to prove SN for System F (Chapter 13) and for dependent type theories. The key feature is that `Red_τ` is defined by *recursion on τ*, not on the term — this breaks the circularity that defeats a naive term induction.

The undefinability of Y in STLC (Section 5) is not just a curiosity: it is the *other side* of SN. Any type system that can type Y cannot be SN. Conversely, if a type system is SN, it cannot type Y (or any other fixpoint combinator). The tradeoff between expressiveness and termination is a fundamental tension in type-system design, revisited in every chapter of Part III.

**The reducibility candidates in detail.** For concreteness, the three key properties that `Red_τ` must satisfy — often called the *reducibility conditions* — are [Girard 1972, TAPL §12.1]:

```
(CR1)  Red_τ(M)  implies  M is SN
(CR2)  Red_τ(M) and M →_β M'  implies  Red_τ(M')
(CR3)  M is neutral (not a λ-abstraction) and
       for all M', M →_β M' implies Red_τ(M')
       implies Red_τ(M)
```

A *neutral* term is a variable or an application — anything that is not a λ-abstraction. CR3 is the key: it allows us to prove `Red_τ(x)` for variables (since variables have no reduction steps, the universal over M' is vacuously true). This lets us satisfy the base case of the adequacy lemma without needing variables to be values.

One verifies by induction on `τ` that each `Red_τ` satisfies CR1–CR3. For base types, `Red_B = SN` and the conditions are immediate. For function types, `Red_{τ→σ}(M) = ∀N, Red_τ(N) → Red_σ(MN)`: CR1 follows from CR3 applied to variables; CR2 follows from the definition; CR3 requires an argument about head expansion using SN of `N` [TAPL §12.1 Lemma 12.1.6].

---

## 7. Type Safety: Preservation and Progress

The **syntactic approach to type safety**, due to Wright and Felleisen [1994], establishes that a well-typed program never reaches a *stuck state* — a state that is neither a value nor reducible. Stuck states are the type-theoretic manifestation of "undefined behavior" or "type error at runtime." Type safety consists of two theorems: Preservation (types are maintained by reduction) and Progress (well-typed closed terms can always take a step or are already values) [TAPL §9.3].

### The Substitution Lemma

Both safety theorems depend on the following central lemma [TAPL §9.3]:

> **Lemma (Substitution, Typed):** If `Γ, x:σ ⊢ M : τ` and `Γ ⊢ N : σ`, then `Γ ⊢ M[x := N] : τ`.

In words: substituting a well-typed term for a variable preserves the type of the surrounding term. This lemma is the key technical tool; Preservation reduces to it.

*Proof.* By induction on the typing derivation `Γ, x:σ ⊢ M : τ`.

- *Case T-Var, M = x:* Then `τ = σ` and `M[x := N] = N`. We need `Γ ⊢ N : σ`, which is given.

- *Case T-Var, M = y, y ≠ x:* `M[x := N] = y`. We need `Γ ⊢ y : τ`. Since `y ≠ x` and `Γ, x:σ ⊢ y : τ`, we know `Γ` binds `y:τ` (the binding for `y` is not the `x:σ` we added). So `Γ ⊢ y : τ` by T-Var.

- *Case T-Abs, M = λy:τ₁. M':* By the Barendregt convention, `y ≠ x` and `y ∉ FV(N)`. The premise is `Γ, x:σ, y:τ₁ ⊢ M' : τ₂` with `τ = τ₁ → τ₂`. We need `Γ ⊢ (λy:τ₁. M')[x := N] : τ₁ → τ₂`, which is `Γ ⊢ λy:τ₁. M'[x:=N] : τ₁ → τ₂`. By the induction hypothesis (weakening `Γ ⊢ N : σ` to `Γ, y:τ₁ ⊢ N : σ`): `Γ, y:τ₁ ⊢ M'[x:=N] : τ₂`. Applying T-Abs: `Γ ⊢ λy:τ₁. M'[x:=N] : τ₁ → τ₂`. ✓

- *Case T-App, M = M₁ M₂:* Premises: `Γ, x:σ ⊢ M₁ : τ₁→τ₂` and `Γ, x:σ ⊢ M₂ : τ₁` (with `τ = τ₂`). By the induction hypothesis on each: `Γ ⊢ M₁[x:=N] : τ₁→τ₂` and `Γ ⊢ M₂[x:=N] : τ₁`. By T-App: `Γ ⊢ (M₁ M₂)[x:=N] : τ₂`. ✓ □

### Preservation (Subject Reduction)

> **Theorem (Preservation):** If `Γ ⊢ M : τ` and `M →_β M'`, then `Γ ⊢ M' : τ` [TAPL §9.3].

*Proof.* By induction on the derivation of `M →_β M'`.

The only interesting case is a top-level β-step: `M = (λx:τ₁. M₀) N` and `M' = M₀[x := N]`, with the typing derivation:

```
Γ, x:τ₁ ⊢ M₀ : τ₂          Γ ⊢ N : τ₁
───────────────────────────────────────────────   (T-App ∘ T-Abs)
Γ ⊢ (λx:τ₁. M₀) N : τ₂
```

where `τ = τ₂`. Applying the Substitution Lemma with `Γ, x:τ₁ ⊢ M₀ : τ₂` and `Γ ⊢ N : τ₁`:

```
Γ ⊢ M₀[x := N] : τ₂
```

which is exactly `Γ ⊢ M' : τ`.

The congruence cases (reduction inside application or under a binder) go by the induction hypothesis applied to the appropriate sub-derivation. □

Preservation is also called *subject reduction* because it says the *subject* of the typing judgment (`M`) can be reduced without changing the *predicate* (its type `τ`).

### Canonical Forms Lemma

Progress requires knowing what values of a given type look like. We need:

> **Lemma (Canonical Forms):** If `⊢ v : τ₁ → τ₂` and `v` is a value (i.e., a β-normal form that is not an application), then `v = λx:τ₁. M` for some `x` and `M`.

*Proof.* By inspection of the value grammar (in pure STLC, values are exactly the λ-abstractions) and the typing rules. A closed value of function type cannot be a variable (no context) and cannot be an application (applications are not values in CBV/normal-order; and if it were an application `M₁ M₂`, it would be a redex if `M₁` is a λ). □

In STLC with base type constants (like `true : Bool` or `0 : Nat`), an additional canonical forms lemma covers base types: if `⊢ v : Bool` and `v` is a value, then `v = true` or `v = false`.

### Progress

> **Theorem (Progress):** If `⊢ M : τ` (M is closed and well-typed), then either M is a value, or there exists `M'` with `M →_β M'` [TAPL §9.3].

*Proof.* By induction on the typing derivation of `⊢ M : τ`.

- *Case T-Var:* Impossible — `M = x` implies `x ∈ dom(Γ)`, but `Γ = ∅` for a closed term.

- *Case T-Abs:* `M = λx:τ₁. M₀`. This is a value. Done.

- *Case T-App:* `M = M₁ M₂`, with `⊢ M₁ : τ₁ → τ₂` and `⊢ M₂ : τ₁`.

  By the induction hypothesis on `M₁`: either `M₁` is a value or `M₁ →_β M₁'`.

  - If `M₁ →_β M₁'`: then `M₁ M₂ →_β M₁' M₂` by the congruence rule. Done.
  - If `M₁` is a value: apply the Canonical Forms Lemma. Since `⊢ M₁ : τ₁ → τ₂` and `M₁` is a value, `M₁ = λx:τ₁. M₀`.

    By the induction hypothesis on `M₂`: either `M₂` is a value or `M₂ →_β M₂'`.

    - *In CBV:* if `M₂ →_β M₂'`, step `M₂`. If `M₂` is a value, we have `(λx:τ₁. M₀) V` where `V = M₂` is a value — this is a β-redex. Step by β-reduction. Done.
    - *In normal-order:* once `M₁ = λx:τ₁. M₀`, the term `(λx:τ₁. M₀) M₂` is a redex regardless of `M₂`'s form. Step by β-reduction. Done. □

The **Canonical Forms Lemma** is what makes the App case go through: we need to know that a value of function type is a λ-abstraction, not some other construction that cannot be applied. In extensions of STLC (e.g., with pairs, sums, or recursive types), additional canonical forms clauses are needed for each type constructor.

### Type Soundness

> **Corollary (Type Soundness):** A well-typed closed program does not get stuck. Formally: if `⊢ M : τ` and `M →*_β N`, then either `N` is a value, or there exists `N'` with `N →_β N'`.

*Proof.* By Preservation, `⊢ N : τ` (after any number of steps). By Progress, `N` is either a value or can take a step. □

Combined with Strong Normalization for STLC, we get an even stronger result: a well-typed closed term of STLC *always terminates* with a value of the correct type. In a language that adds recursive types or a `fix` operator, SN is lost but type soundness (the two-theorem version above) is maintained — the program may diverge, but if it produces any output, that output has the correct type.

**The Wright-Felleisen framework.** The syntactic approach presented here — define a small-step operational semantics, identify values, prove Preservation and Progress — has become the standard method for establishing type soundness in programming language theory. It was introduced in Wright and Felleisen, "A Syntactic Approach to Type Soundness," *Information and Computation*, 1994. Before their paper, type soundness was often proved semantically (via denotational semantics), which required building a model. The syntactic approach is purely proof-theoretic, more accessible, and directly applicable to language implementations.

**Worked example: a complete type safety trace.** Let us trace a single step of a well-typed program and verify that both Preservation and Progress apply at each stage. Consider the closed, well-typed term:

```
M = (λf:Bool→Bool. λx:Bool. f x) not true
```

where `not = λb:Bool. if b then false else true : Bool → Bool`.

Step 1: Apply Preservation. We need `⊢ M : Bool`. The derivation:
```
⊢ λf:Bool→Bool. λx:Bool. f x : (Bool→Bool) → Bool → Bool
⊢ not : Bool→Bool
────────────────────────────────────────────────────────── (T-App)
⊢ (λf:Bool→Bool. λx:Bool. f x) not : Bool → Bool
⊢ true : Bool
────────────────────────────────────────────────────────── (T-App)
⊢ M : Bool
```

Step 2: Apply Progress. `M` is a closed well-typed term of type `Bool`. It is not a value (it is an application, not a boolean constant). So Progress guarantees a step exists. In CBV:

- The function position `(λf:Bool→Bool. λx:Bool. f x) not` — the outer function `λf…` is a value and `not` is a value. So we can β-reduce:
```
M →_β (λx:Bool. not x) true
```

Step 3: Apply Preservation again. The new term `M' = (λx:Bool. not x) true` must have type `Bool`. By the Substitution Lemma applied to the β-step: `⊢ (λx:Bool. not x) true : Bool`. ✓

Step 4: Progress again on `M'`. The function `λx:Bool. not x` is a value; `true` is a value. β-reduce:
```
M' →_β not true = (λb:Bool. if b then false else true) true
```

Step 5: Another β-step:
```
→_β if true then false else true
```

Step 6: The conditional rule: `if true then false else true →_β false`.

The chain `M →*_β false : Bool` is a complete run of type-correct evaluation. At every step, the term is well-typed at `Bool` (Preservation), and at every non-value step, a step is possible (Progress). The final result `false` is a value of the correct type `Bool`.

**Semantic type soundness via logical relations.** The alternative, semantic approach proves soundness by constructing a *logical relation* (a family of sets `L_τ` of closed terms that "behave well" at type `τ`) and showing that all well-typed terms belong to `L_τ`. This approach is more powerful for proving properties beyond soundness (e.g., parametricity, which we discuss in Chapter 13), and it is the method used to prove SN above (the reducibility candidates *are* a logical relation). The syntactic and semantic approaches are complementary: the syntactic approach is more direct and modular; the semantic approach is more expressive and scales to languages where typing rules involve complex metatheory.

**Inversion lemmas.** The proofs above rely tacitly on *inversion* of typing derivations: if `Γ ⊢ M N : τ`, then the last rule used must be T-App, giving `Γ ⊢ M : σ→τ` and `Γ ⊢ N : σ` for some `σ`. If `Γ ⊢ λx:τ₁.M : τ`, then the last rule must be T-Abs, so `τ = τ₁→τ₂` for some `τ₂` with `Γ, x:τ₁ ⊢ M : τ₂`. These inversion principles are immediate from the syntax-directedness of the typing rules: in Church-style STLC, each syntactic form of term has exactly one applicable typing rule. Inversion is what lets us extract sub-derivations in the Progress and Preservation proofs without ambiguity.

**Weakening and Strengthening.** Two structural properties of the typing judgment that are used throughout the proofs:

> **Weakening:** If `Γ ⊢ M : τ` and `x ∉ dom(Γ)`, then `Γ, x:σ ⊢ M : τ`.

Typing is monotone in the context: adding new bindings does not invalidate existing ones. Proof: by induction on the derivation.

> **Strengthening:** If `Γ, x:σ ⊢ M : τ` and `x ∉ FV(M)`, then `Γ ⊢ M : τ`.

Conversely, if a variable is not free in `M`, its binding can be removed from the context. Strengthening is used in the T-Abs case of the Substitution Lemma to show that the context extension by the binder variable is valid. Not all type systems have strengthening (systems with linear types or coeffects break it), but it holds for STLC.

---

## 8. The Curry-Howard Correspondence

### The Correspondence

The **Curry-Howard correspondence** (also called the propositions-as-types or proofs-as-programs correspondence) identifies the type theory of STLC with propositional intuitionistic logic [Howard 1980; Curry 1958; TAPL §9.4; PFPL §12]:

| Type Theory (STLC)          | Logic (Intuitionistic Propositional)      |
|-----------------------------|-------------------------------------------|
| Type `τ`                    | Proposition `P`                           |
| Term `M : τ`               | Proof of `P`                              |
| Typing derivation           | Natural deduction proof                   |
| Function type `τ → σ`      | Implication `P ⊃ Q`                      |
| T-Abs rule                 | Implication introduction (⊃-I)           |
| T-App rule                 | Modus ponens / implication elim (⊃-E)    |
| β-reduction                | Proof normalization (cut elimination)     |
| Value                       | Normal proof                              |
| Strong normalization        | Termination of proof normalization        |

This is not an analogy — it is a precise, formal bijection between two independently developed disciplines.

### Intuitionistic vs. Classical Logic

The correspondence is with **intuitionistic** (constructive) propositional logic, not classical logic. Classical logic adds the law of excluded middle `P ∨ ¬P` or double negation elimination `¬¬P → P`. These principles are not provable in intuitionistic logic from the rules we have, and they are not typeable in STLC from the rules we have.

Adding *control operators* — specifically, call-with-current-continuation (call/cc) or delimited continuations — corresponds to adding classical reasoning. Specifically [TAPL §9.4]:
- The type of call/cc is `((τ → σ) → τ) → τ`, which corresponds to Peirce's law `((P → Q) → P) → P` — a classical but not intuitionistically provable principle.
- Double-negation translation maps classical proofs to intuitionistic proofs in continuation-passing style, corresponding to CPS transformation of programs.

This is not a curiosity. It means: classical logic is the type theory of languages with first-class control. Intuitionistic logic is the type theory of pure functional languages (no exceptions, no call/cc, no mutable state that can implement control). Expert readers building type-safe languages should know which logical system their type theory inhabits.

### Propositions as Types in Detail

Let us trace the correspondence precisely.

**Implication:** A term `M : τ → σ` is a function from `τ`-values to `σ`-values. A proof of `P ⊃ Q` is a construction that turns any proof of `P` into a proof of `Q`. The T-Abs rule:

```
Γ, x:τ ⊢ M : σ
───────────────────
Γ ⊢ λx:τ. M : τ → σ
```

corresponds to the natural deduction rule ⊃-Introduction:

```
[P]
⋮
Q
─────
P ⊃ Q
```

"Assume P (x : τ), derive Q (M : σ), conclude P ⊃ Q (λx:τ. M : τ → σ)."

**Modus ponens:** The T-App rule:

```
Γ ⊢ M : τ → σ    Γ ⊢ N : τ
────────────────────────────
Γ ⊢ M N : σ
```

corresponds to ⊃-Elimination:

```
P ⊃ Q    P
──────────
    Q
```

"Given a proof of P ⊃ Q and a proof of P, conclude Q."

**β-reduction as cut elimination.** In natural deduction, a *detour* is a proof that introduces a connective (⊃-I) and immediately eliminates it (⊃-E) — this corresponds to an immediate β-redex:

```
   [x:τ]
    M : σ         N : τ
──────────────── (⊃-I)  ─────
λx:τ. M : τ → σ
────────────────────────────── (⊃-E)
M[x := N] : σ
```

The β-reduction `(λx:τ. M) N →_β M[x := N]` eliminates this detour in one step. Gentzen's Hauptsatz (cut elimination theorem) in proof theory says exactly that every proof can be normalized to eliminate all such detours. Under the correspondence, this *is* β-reduction. Strong normalization for STLC is the computational content of cut elimination for propositional intuitionistic logic.

### Product Types, Sum Types, and Units

The correspondence extends to other connectives [TAPL §11, §14]:

**Product types** `τ × σ` correspond to conjunction `P ∧ Q`:
- Introduction: the pair `(M, N) : τ × σ` corresponds to the proof of `P ∧ Q` from proofs of `P` and `Q`.
- Elimination: projections `π₁(M) : τ` and `π₂(M) : σ` correspond to conjunction elimination.
- The typing rules:

```
Γ ⊢ M : τ    Γ ⊢ N : σ         Γ ⊢ M : τ × σ         Γ ⊢ M : τ × σ
──────────────────────────     ──────────────────     ──────────────────
Γ ⊢ (M, N) : τ × σ            Γ ⊢ π₁(M) : τ         Γ ⊢ π₂(M) : σ
```

**Sum types** `τ + σ` correspond to disjunction `P ∨ Q`:
- Introduction: injections `inl(M) : τ + σ` (from `M : τ`) and `inr(N) : τ + σ` (from `N : σ`) correspond to `∨-I`.
- Elimination: case analysis `case M of inl(x) ⇒ L | inr(y) ⇒ R` corresponds to `∨-E` (proof by cases).

**Unit type** `1` (or `Unit`) corresponds to ⊤ (truth):
- The unique term `() : 1` corresponds to the unique proof of ⊤.
- No elimination rule: `⊤` cannot be used to conclude anything beyond what was already known.

**Empty type** `0` (or `Void`, `Empty`) corresponds to ⊥ (falsehood):
- No terms inhabit `0` (in a type-safe system).
- Elimination: `absurd(M) : τ` for any `τ`, given `M : 0`, corresponds to ex falso (`⊥-E`): from a proof of `⊥`, conclude anything.

The table is exact:

| Type constructor | Logical connective | Proof principle |
|------------------|--------------------|-----------------|
| `τ → σ`         | `P ⊃ Q`            | Implication     |
| `τ × σ`         | `P ∧ Q`            | Conjunction     |
| `τ + σ`         | `P ∨ Q`            | Disjunction     |
| `1` (Unit)       | `⊤`                | Truth           |
| `0` (Empty)      | `⊥`                | Falsehood       |
| `¬τ = τ → 0`   | `¬P = P ⊃ ⊥`      | Negation        |

Note that `¬τ` is *not* a primitive constructor: negation is defined as the function type from `τ` to `0`. This is characteristic of intuitionistic logic: there is no primitive notion of negation that can "unilaterally" refute a proposition.

### Inhabitation Equals Provability

> **Theorem:** A type `τ` is **inhabited** in STLC (there exists a closed term `⊢ M : τ`) if and only if the corresponding proposition is a **tautology of intuitionistic propositional logic** [TAPL §9.4].

This gives an algorithmic decision procedure for both problems: type inhabitation in STLC is decidable (by the decision procedure for intuitionistic propositional logic), and the same algorithm decides provability. The classical tautologies that are not intuitionistic tautologies (excluded middle, Peirce's law, double-negation elimination) correspond to types that are uninhabited in pure STLC.

**Example.** The type `(τ → σ) → (σ → ρ) → τ → ρ` is inhabited by `λf. λg. λx. g(f x)` — this is function composition, corresponding to the hypothetical syllogism `(P ⊃ Q) → (Q ⊃ R) → P ⊃ R`. The type `τ → τ` is inhabited by `λx:τ. x` — the identity, corresponding to `P ⊃ P`. The type `τ → σ` for distinct base types `B₁ → B₂` is *not* inhabited (no closed STLC term has this type when `B₁ ≠ B₂`), corresponding to the non-tautology `P ⊃ Q` for distinct atomic `P`, `Q`.

A related algorithmic question is **type inhabitation search**: given a type `τ`, find a term `M` with `⊢ M : τ`. This is the *synthesis* direction of type checking. For STLC it is decidable (since intuitionistic propositional provability is decidable — PSPACE-complete); in practice, tools like Agda's proof search, Coq's `auto` tactic, and Haskell's `djinn` implement versions of this search. The inhabitation problem is one of the foundational problems of automated theorem proving, and its connection to program synthesis — "find a program with this type" — is a major research theme in programming languages [PFPL §12].

### The Extension to Dependent Types: Preview

The Curry-Howard correspondence extends beyond propositional logic to predicate logic when the type system gains **dependent types** — types that can depend on values. In a dependent type system (Chapter 14):

- A dependent function type `Π(x:A). B(x)` (written `∀x:A. B(x)` in logic) corresponds to *universal quantification*: a function that takes a value `a:A` and produces a proof of `B(a)`.
- A dependent pair type `Σ(x:A). B(x)` (written `∃x:A. B(x)` in logic) corresponds to *existential quantification*: a pair of a value `a:A` and a proof of `B(a)`.

The correspondence at this level is:

| Dependent Type Theory | Predicate Logic |
|-----------------------|-----------------|
| `Π(x:A). B(x)`       | `∀x. P(x)`     |
| `Σ(x:A). B(x)`       | `∃x. P(x)`     |
| `Id_A(a, b)` (identity type) | `a = b` (equality) |

This is the foundation of proof assistants: Coq uses the Calculus of Inductive Constructions (CIC), Lean 4 uses a related dependent type theory, and Agda uses Martin-Löf type theory — all of which are extensions of the simply typed lambda calculus in the Curry-Howard direction. The proofs-as-programs principle allows these systems to extract verified computational content from proofs: a proof of `∀n:Nat. ∃m:Nat. m > n` yields a computable function that, given `n`, produces `m`. Chapter 14 develops dependent type theory in detail; Chapter 167 connects it to verified compilation.

### Why Coq and Lean Use Dependent Types

In Coq and Lean, theorems are types and proofs are programs. A theorem statement `∀(n:ℕ), n + 0 = n` is a dependent function type `Π(n:ℕ). Id_ℕ(n + 0, n)`. A proof of this theorem is a term of that type. The kernel of Coq or Lean is a *type checker* for the underlying dependent type theory: to verify a proof, the kernel checks that a term has the claimed type. This reduces proof verification to *type checking* — a decidable, efficiently implementable operation. The correctness of the proof assistant reduces to: (1) the type checker is correctly implemented, and (2) the type theory is sound (consistent — has no closed proof of `⊥`). The second property is established by the metatheory of the type system, which traces directly to the normalization arguments we have surveyed in this chapter.

LLVM and MLIR do not use dependent types — their type systems are intentionally simple for efficiency and decidable type-checking. But the *intellectual lineage* runs straight from the Curry-Howard correspondence through Coq/Lean to the correctness arguments about LLVM's type system and the formal semantics work in Vellvm (Chapter 169) and Alive2 (Chapter 170).

### β-Reduction as Proof Simplification

One final and elegant consequence of the correspondence: **proof normalization in natural deduction is exactly β-reduction of the corresponding term** [Prawitz 1965; TAPL §9.4]. A β-redex `(λx:τ.M)N` corresponds to a proof that introduces an assumption (`⊃-I`) and immediately discharges it (`⊃-E`) — this is a *detour* in the proof, logically redundant. Eliminating the detour (reducing the redex) produces a *normal proof* (a value). Strong normalization for STLC is, from the logical perspective, the theorem that all natural deduction proofs in intuitionistic propositional logic can be normalized. This was proved by Prawitz in 1965, independently of Tait's 1967 type-theoretic proof. The two proofs are, of course, the same proof — one written in the language of proof theory, one in the language of type theory, related by the Curry-Howard correspondence.

### The Sequent Calculus Side

The Curry-Howard correspondence has a second branch: **sequent calculus** corresponds to explicit substitution and explicit proof-term manipulation [PFPL §12]. In Gentzen's sequent calculus LJ (intuitionistic), the Cut rule:

```
Γ ⊢ A    A, Δ ⊢ C
──────────────────    (Cut)
Γ, Δ ⊢ C
```

corresponds to let-binding or substitution: `let x = M in N`, which has the typing:

```
Γ ⊢ M : τ    Γ, x:τ ⊢ N : σ
────────────────────────────────
Γ ⊢ let x = M in N : σ
```

The **cut-elimination theorem** for LJ (Gentzen's *Hauptsatz*) says that every proof using Cut can be rewritten into a cut-free proof. This corresponds to *reducing all let-bindings to values* (or to performing all substitutions). In terms of the lambda calculus: cut elimination is normalization; the cut-elimination algorithm is the normalizing evaluator.

This gives three levels of the same phenomenon:
1. **Proof theory:** Cut elimination (Gentzen, 1935)
2. **Type theory:** β-reduction and strong normalization (Tait, 1967; Girard, 1972)
3. **Programming:** evaluation of a functional program (Church, 1941; Scott, 1970s)

The correspondence is deep enough that proof-assistant kernel design follows directly from it. Coq's kernel is a normalizer for the Calculus of Inductive Constructions; its termination checking reduces to normalization arguments for the underlying type theory; and the correctness of its definitional equality check reduces to confluence (Church-Rosser) for the dependent-type analogue of β-reduction.

### Negative Types and Control Operators

A complete picture of Curry-Howard includes the correspondence between classical logic and languages with control [TAPL §9.4, Filinski 1989, Griffin 1990]:

| Classical principle | Control operator | Type |
|--------------------|--------------------|------|
| Peirce's law `((P⊃Q)⊃P)⊃P` | `call/cc` | `((τ→σ)→τ)→τ` |
| Double-negation `¬¬P⊃P` | `abort` / `callcc at unit` | `((τ→⊥)→⊥)→τ` |
| Excluded middle `P∨¬P` | `call/cc` on disjunction | `τ + (τ→⊥)` |

Griffin's 1990 paper "A formulae-as-types notion of control" was the first to make this correspondence precise. It means that reasoning about programs with exceptions, non-local exits, or delimited continuations is equivalent to reasoning in classical (or intermediate) logic. The continuation monad, used pervasively in compiler intermediate representations (including LLVM IR's continuation-passing-style transformation), is the computational interpretation of double-negation translation — the standard way of embedding classical logic into intuitionistic logic.

This observation has direct consequences for compiler construction. When Clang lowers C++ exceptions to LLVM IR invoke/landingpad instructions, it is performing a controlled escape from pure functional structure — introducing the control-operator machinery. The type discipline of LLVM IR, which tracks `invoke` and landingpad separately from ordinary calls, can be understood as a type-theoretic constraint ensuring that exception-propagation control flow is structurally bounded. Chapter 26 (Exception Handling) discusses this lowering in detail; the present chapter provides the type-theoretic vocabulary for understanding what the safety guarantees mean.

### The Lambda Calculus as Intermediate Representation

Several mature compilers use the lambda calculus (or close relatives) directly as their intermediate representation:

**Continuation-Passing Style (CPS).** CPS transforms every program so that functions never return — they take an explicit *continuation* argument representing "what to do next." The result is a lambda calculus where all control flow is explicit and there are no implicit stacks. CPS was introduced by Plotkin (1975) and developed by Appel and others for the SML/NJ compiler. Under CPS, every intermediate value has a name; every call is a tail call; control flow is just function application. The CPS representation is strongly typed: the continuation argument has a function type, and typing enforces proper control flow. Appel's book, *Compiling with Continuations* (Cambridge, 1992), remains the definitive reference.

**Administrative Normal Form (ANF).** ANF (also called *A-normal form*, introduced by Flanagan et al. 1993) is a less radical transformation than CPS: it ensures every argument to a function is a value (variable or constant), by introducing let-bindings for intermediate computations. ANF is "lambda calculus with explicit let-binding and no nested applications." It is easier to work with than CPS for many analyses while still exposing the fine-grained evaluation order. GHC (the Glasgow Haskell Compiler) uses a variant of ANF called *STG* (Spineless Tagless G-machine) as its final functional IR before code generation.

**LLVM IR and SSA.** LLVM IR is not literally the lambda calculus, but it is definitionally related: SSA form is isomorphic to a particular restricted form of the lambda calculus where λ-binders correspond to basic-block parameters and φ-nodes correspond to joins [Kelsey 1995; Chakravarty et al. 2003]. The block-parameter form of φ-elimination used in modern LLVM IR (where φ-nodes are replaced by explicit block arguments) makes this correspondence visible. Chapter 21 (SSA, Dominance, and Loops) develops this connection. The point for this chapter is that the lambda calculus is not just a theoretical tool — it is the mathematical structure that SSA IR instantiates, and the metatheory of this chapter (substitution, reduction, typing) applies, with appropriate translations, to LLVM IR analysis.

---

## 9. Chapter Summary

- **The untyped lambda calculus** has exactly three term forms: variables, abstractions, and applications. Free and bound variables are defined inductively; the Barendregt convention requires bound variables to be chosen fresh. De Bruijn indices provide an alternative, name-free representation.

- **Capture-avoiding substitution** `M[x := N]` renames bound variables in `M` that would capture free variables of `N`. Naive substitution produces incorrect semantics by conflating distinct occurrences.

- **α-equivalence** identifies terms that differ only in bound-variable names. β-reduction `(λx.M)N →_β M[x:=N]` is the computational rule. η-reduction `λx.Mx →_η M` (for x ∉ FV(M)) captures extensionality. Reduction strategies — normal-order (CBN), call-by-value, call-by-need — determine which redex is chosen and when; only normal-order is guaranteed to find a normal form when one exists.

- **Church-Rosser confluence** states that β-reduction is confluent: any two reduction sequences from the same term eventually meet. The proof uses parallel reduction to establish the diamond property. The consequence is that β-normal forms are unique: a term has at most one normal form regardless of the reduction path taken.

- **The untyped lambda calculus is neither weakly nor strongly normalizing.** The Ω = (λx.xx)(λx.xx) term diverges. The Y combinator `Y = λf.(λx.f(xx))(λx.f(xx))` encodes general recursion but cannot be typed in STLC because it requires a self-referential type `σ = σ → τ`, which has no finite solution.

- **Simply typed lambda calculus (STLC)** uses Church-style explicitly typed binders. The three typing rules — T-Var, T-Abs, T-App — correspond precisely to variable lookup, implication introduction, and modus ponens. Type uniqueness holds in Church-style STLC. **Every well-typed term is strongly normalizing** (proved by Tait's reducibility candidates); the undefinability of Y in STLC is the other side of this coin.

- **Type safety** consists of two theorems. **Preservation** (subject reduction): if `Γ ⊢ M : τ` and `M →_β M'`, then `Γ ⊢ M' : τ`. The proof hinges on the Substitution Lemma: substituting a well-typed term preserves the type of the context. **Progress**: if `⊢ M : τ` (closed and well-typed), then M is a value or can take a step; the Canonical Forms Lemma is essential for the App case. Together they guarantee that well-typed programs never reach a stuck state.

- **The Curry-Howard correspondence** identifies STLC with intuitionistic propositional logic: types are propositions, terms are proofs, typing derivations are natural deduction proofs, β-reduction is proof normalization (cut elimination). Product types correspond to conjunction, sum types to disjunction, Unit to ⊤, and Empty to ⊥. Classical extensions (Peirce's law) correspond to control operators (call/cc), not pure functions. A type is inhabited if and only if the corresponding proposition is an intuitionistic tautology.

- **Dependent types extend the correspondence** to predicate logic: `Π(x:A).B(x)` is universal quantification and `Σ(x:A).B(x)` is existential. This is the foundation of Coq and Lean, where proof verification reduces to type checking. Chapter 14 develops dependent type theory; Chapter 167 connects it to verified compilation using Vellvm and Alive2.

---

*Cross-references:*
- *Chapter 13 — Polymorphism and Type Inference* extends STLC with System F's universal types and Hindley-Milner type inference, where Curry-style typing and type reconstruction replace the explicit annotations of this chapter.
- *Chapter 14 — Advanced Type Systems* adds recursive types (which enable typing Y), dependent types (which extend Curry-Howard to predicate logic), and refinement types.
- *Chapter 15 — Type Theory in Practice: From Theory to LLVM and MLIR* connects the abstract machinery developed here to the concrete type systems of LLVM IR and MLIR's type and attribute framework.
- *Chapter 167 — Operational Semantics and Program Logics* and *Chapter 169 — Vellvm* show how the Progress/Preservation methodology is applied to formalize the semantics of LLVM IR itself.


---

@copyright jreuben11
