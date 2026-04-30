# Chapter 184 — Proof Assistant Internals: Lean 4, Coq/Rocq, and Isabelle/HOL

*Part XXVII — Mathematical Foundations and Verified Systems*

Proof assistants are the last line of defence when testing, static analysis, and model checking reach their limit. CompCert ([Chapter 168](../part-24-verified-compilation/ch168-compcert.md)) proves that every compilation pass preserves semantics. Alive2 ([Chapter 170](../part-24-verified-compilation/ch170-alive2-and-translation-validation.md)) checks that every peephole rewrite is sound under the undef/poison model. The seL4 microkernel carries a machine-checked proof of functional correctness for its C implementation ([Chapter 186](../part-27-mathematical-foundations/ch186-verified-hardware-cheri-capabilities-and-the-sel4-microkernel.md)). All three of these results depend, in the last instance, on the trustworthiness of a proof assistant kernel — a small program that can be independently audited and whose acceptance of a proof certificate constitutes the entire foundation of the verification. This chapter opens the hood on three major systems: Lean 4, Coq/Rocq, and Isabelle/HOL. It examines their kernels, their internal representations, their elaboration and tactic machinery, their compilation pipelines, and the specific choices each system has made that explain why CompCert chose Coq extraction, Vellvm chose Coq, and seL4 chose Isabelle. Throughout, the analysis connects to the type-theoretic foundations of [Chapters 12–15](../part-03-type-theory/) and to the practical verification landscape of [Chapter 181](../part-26-ecosystem-frontiers/ch181-formal-verification-in-practice.md).

---

## 184.1 Lean 4 Architecture

Lean 4 is a dependently-typed functional programming language and proof assistant developed by Leonardo de Moura and Sebastian Ullrich at Microsoft Research and later Amazon Web Services. Unlike its predecessor Lean 3, which separated the programming and proving aspects through an awkward two-namespace design, Lean 4 is a unified system: every mathematical proof is a program, and every program can serve as a computational substrate for proof automation. It is written in Lean 4 itself after bootstrapping — the first substantial language in this class where the entire implementation, including the build system and standard library, is verified or at least type-checked by the system itself. The source lives at [https://github.com/leanprover/lean4](https://github.com/leanprover/lean4); the core of what follows refers to files under `src/Lean/`.

### 184.1.1 The Kernel: The Sole Trusted Base

The Lean 4 kernel, implemented in [`src/Lean/Kernel.lean`](https://github.com/leanprover/lean4/blob/main/src/Lean/Kernel.lean) and its supporting C++ files under `src/kernel/`, is the only component that must be trusted to accept any Lean proof as valid. The central entry point is `Lean.Kernel.check`, which takes a declaration and an environment and decides whether the declaration is type-correct according to the core type theory. Critically, **the kernel does not perform unification**. Unification — deciding whether two expressions can be made equal by instantiating metavariables — is the job of the elaborator. By the time a term reaches the kernel, every metavariable must already be resolved; the kernel merely type-checks a closed, fully-elaborated term. This design decision is what keeps the trusted computing base (TCB) small: the kernel implements a decidable type-checking algorithm, not a semi-decidable unification search.

The kernel's responsibilities are exactly:
- **Definition well-typedness**: the type annotation of a definition is itself a well-typed type, and the body has that type under the empty local context.
- **Inductive type well-typedness**: the constructors satisfy the **strict positivity** condition (no occurrence of the type being defined to the left of an arrow in a constructor argument) and respect the universe level constraints implied by the constructor types.
- **Definitional equality**: five reduction rules determine when two terms are considered equal. β (apply a λ-abstraction to its argument), δ (unfold a transparent definition to its body), ι (reduce a recursor applied to a constructor — the computation rule of induction), ζ (substitute a `let`-bound variable), and η (two functions are equal if equal on all arguments). These five rules are decidable because they all terminate and the combination is confluent (Church-Rosser).

```lean4
-- Example: the kernel sees this fully elaborated term, not the source
-- Kernel.check is called by the elaborator via Environment.addDecl
#check @Nat.add_comm   -- type: ∀ (n m : Nat), n + m = m + n
-- The kernel verifies the proof term; no metavariables remain at this point
-- The proof term itself is an application of Nat.rec with appropriate arguments
```

The kernel is deliberately conservative: it implements a straightforward structural recursion over `Expr` with universe level checking at each `sort` node. There is no caching of intermediate type-checking results in the trusted path (though the elaborator caches `isDefEq` results in its monad state, and the kernel does maintain a definitional equality cache for performance). The kernel's C++ implementation is approximately 6,000 lines — small enough to read in a day and audit independently. The decision to implement the kernel in C++ rather than Lean itself reflects a bootstrapping necessity: the kernel must be trusted before Lean can check anything.

The **`Environment`** type, central to both the kernel and the elaborator, maps `Name` values to `Declaration` values — either `axiom`, `opaque`, `def`, `theorem`, or `inductive`. The kernel's trusted inference rules can only add new declarations to the environment if they pass the `Lean.Kernel.check` gate. This unidirectional flow — elaborator proposes, kernel decides — is the architectural guarantee of soundness.

### 184.1.2 Core Dependent Type Theory

Lean 4's type theory is a variant of the Calculus of Inductive Constructions (CIC) enriched with four features beyond standard CIC:

**Universe polymorphism**: declarations are parameterized by universe level variables. A type `List.{u} (α : Type u)` lives at `Type u` for any universe level `u`. This avoids the need to duplicate definitions at every universe level — without polymorphism, `List Nat`, `List (List Nat)`, and `List (Type 0)` would each require a separately-defined `List` type at different universes. The elaborator infers universe level arguments automatically; the programmer writes `List α` and the system inserts the level.

**Proof irrelevance**: any two proofs of the same proposition are definitionally equal. If `h₁ h₂ : P` where `P : Prop`, then `h₁ ≡ h₂` in the kernel. This is essential for practical proof automation: it prevents proof terms from bloating data structures or affecting computational behaviour. Without proof irrelevance, a natural number that carries a proof of being prime would have a different identity depending on which of the many possible primality proofs was attached. With proof irrelevance, all such numbers are definitionally equal (their underlying values being equal).

**Quotient types**: `Quot r` quotients a type `α` by a relation `r : α → α → Prop`. The kernel rules for `Quot.mk`, `Quot.lift`, and `Quot.ind` together implement a first-class extensional quotient. The kernel rule for `Quot.sound` — given a proof `h : r a b`, conclude `Quot.mk r a = Quot.mk r b` — is what gives quotient types their power. This mechanism underpins integers (quotient of `Nat × Nat` by `(a,b) ~ (c,d) ↔ a+d=b+c`), rationals, finsets, and `Multiset`.

**Large elimination**: inductives in `Type u` support **large elimination** — recursion that computes types, not just values. For example, `Bool.rec (motive := fun _ => Type) Nat String` yields `Nat` or `String` depending on a `Bool` value. Inductives in `Prop` generally do not support large elimination (only into `Prop`), which is what prevents classical logic from breaking computational content. The exception is `Subsingleton` — propositions with at most one proof can support some restricted forms of elimination.

The universe hierarchy is: `Prop : Type 0 : Type 1 : Type 2 : ...`. `Sort u` unifies both: `Sort 0 = Prop`, `Sort (u+1) = Type u`. The rule `Prop : Type 0` — propositions inhabit the lowest type universe — is what makes Lean's logic **predicative** with respect to types but **impredicative** with respect to propositions.

### 184.1.3 The `Lean.Expr` AST

Every Lean 4 term is represented internally as a value of type `Lean.Expr`, defined in [`src/Lean/Expr.lean`](https://github.com/leanprover/lean4/blob/main/src/Lean/Expr.lean). The twelve constructors partition terms into: variable forms (bvar, fvar, mvar), type classifiers (sort), constants (const), structural term constructors (app, lam, forallE, letE), data (lit), annotations (mdata), and projections (proj):

```lean4
inductive Expr where
  | bvar    : Nat → Expr
            -- bound variable: de Bruijn index 0 = innermost binder
  | fvar    : FVarId → Expr
            -- free variable: unique ID into the local context
  | mvar    : MVarId → Expr
            -- metavariable: an unresolved hole in the term
  | sort    : Level → Expr
            -- Sort u: Prop = Sort Level.zero, Type u = Sort (Level.succ u)
  | const   : Name → List Level → Expr
            -- global constant with its universe level arguments
  | app     : Expr → Expr → Expr
            -- f a: curried application (n-ary app is a left-nested chain)
  | lam     : Name → Expr → Expr → BinderInfo → Expr
            -- λ (x : τ) => body; BinderInfo is implicit/explicit/instImplicit
  | forallE : Name → Expr → Expr → BinderInfo → Expr
            -- Π (x : τ), body, i.e., ∀ x : τ, P x
  | letE    : Name → Expr → Expr → Expr → Bool → Expr
            -- let x : τ := val; body; Bool = nonDep flag
  | lit     : Literal → Expr
            -- Literal.natVal n or Literal.strVal s
  | mdata   : MData → Expr → Expr
            -- metadata wrapper (source positions, tactic hints, display info)
  | proj    : Name → Nat → Expr → Expr
            -- structure field projection: proj `Point 0 e = e.x
```

The representation uses **de Bruijn indices** for bound variables (`bvar`): the innermost binder is index 0, the next enclosing binder is index 1, and so on. The term `λ x y, x` in named form becomes `lam _ _ (lam _ _ (bvar 1))` — `bvar 1` refers up two binders to `x`. This representation makes α-equivalence syntactic equality and avoids the need for capture-avoiding substitution on named terms. Any term that is closed (no free `bvar` occurrences) is automatically α-equivalent to any of its α-variants.

Free variables (`fvar`) are identified by a `FVarId` — an opaque unique identifier allocated by the elaborator when it introduces a local hypothesis or let-binding into the `LocalContext`. The `LocalContext` is a finite map from `FVarId` to `LocalDecl`, which carries the variable's name, type, and optional value (for `let` bindings). Metavariables (`mvar`) represent goals in tactic proofs or unresolved implicit arguments; they live in the `MetavarContext`, which maps `MVarId` to `MetavarDecl` carrying the metavariable's expected type, local context, and current assignment (if any).

Every `Expr` node carries a cached **hash** and a set of flags — `hasMVar`, `hasFVar`, `hasLooseBVar`, `hasExprMVar`, `hasLevelMVar` — computed once at construction time. These flags allow the elaborator and unifier to quickly skip subtrees that cannot contain metavariables (for occurs-checking), or to quickly determine whether a term is closed (for kernel submission).

### 184.1.4 Universe Levels

Universe levels are represented by `Lean.Level`, defined alongside `Expr`:

```lean4
inductive Level where
  | zero  : Level
            -- Level 0, the bottom (Sort 0 = Prop)
  | succ  : Level → Level
            -- one step up: Sort (succ u) = Type u
  | max   : Level → Level → Level
            -- maximum of two levels: used for Σ-types, products
  | imax  : Level → Level → Level
            -- impredicative max: imax u 0 = 0, otherwise max u v
  | param : Name → Level
            -- universe parameter (free universe variable in a poly. def.)
  | mvar  : LMVarId → Level
            -- universe level metavariable (unresolved level hole)
```

The `imax` constructor captures the impredicative rule for `Prop`: a `Π`-type `(x : α) → β x` lives in `Sort (imax u v)` where `u = level(α)` and `v = level(β x)`. The rule is: if `v = 0` (the codomain is a `Prop`), the entire `Π`-type is in `Prop` regardless of `u` — this is impredicativity. If `v ≠ 0`, `imax u v = max u v`. Without `imax`, it would be impossible to correctly compute the universe of `Prop`-valued `forall` types.

Level normalization — reducing `max (succ zero) zero` to `succ zero`, or `imax u (succ v)` to `max u (succ v)` — is performed by `Level.normalize` before universe constraint solving. Universe constraint solving determines whether a set of `≤` and `<` constraints on level parameter variables (collected during elaboration of universe-polymorphic definitions) has a solution. The constraint solver is implemented in [`src/Lean/Elab/LevelMVarToParam.lean`](https://github.com/leanprover/lean4/blob/main/src/Lean/Elab/LevelMVarToParam.lean) via a constraint propagation algorithm over level expressions.

### 184.1.5 Elaboration Pipeline

The gap between what the programmer writes and what the kernel sees is bridged by the **elaborator**. The full pipeline is:

```
Source text
    → Lean.Parser (Pratt parser)
    → Lean.Syntax (concrete syntax tree)
    → Macro expansion (MacroM: Syntax → Syntax)
    → Command elaboration (CommandElabM)
    → Term elaboration (TermElabM: Syntax → Expr)
    → Typeclass synthesis (SynthInstance)
    → Unification (Meta.isDefEq)
    → Environment.addDecl
    → Lean.Kernel.check
```

**Parsing** produces a `Lean.Syntax` value — a rose tree of `Syntax.node`, `Syntax.atom`, and `Syntax.ident` nodes, tagged with source positions (`SourceInfo`). Lean 4 uses an extensible **Pratt parser** (operator precedence parsing) defined in [`src/Lean/Parser/`](https://github.com/leanprover/lean4/tree/main/src/Lean/Parser): users can register new syntax via `syntax` declarations, and the parser rebuilds its precedence table incrementally.

**Macro expansion** transforms `Syntax → Syntax` by repeatedly applying `@[macro]`-tagged transformations until a fixpoint. The `MacroM` monad can only produce `Syntax` (not inspect the `Environment`), which prevents macros from creating circular dependencies or breaking the trust invariant. Lean 4's `do` notation, `match` expressions, `where` clauses, and most surface syntax are macro-expanded before the term elaborator sees them.

**Term elaboration** runs in [`Lean.Elab.Term.TermElabM`](https://github.com/leanprover/lean4/blob/main/src/Lean/Elab/Term.lean), a reader/state/exception monad stacked over `IO`. The state carries:
- `MetavarContext`: all unresolved metavariables and their assignments.
- `LocalContext`: local hypotheses in scope (introduced by `intro`, `obtain`, `fix`).
- `Environment`: all previously checked declarations.
- `InfoTree`: the tree of elaboration information used by the language server for hover info and go-to-definition.

Elaboration is **bidirectional**: the elaborator threads an `expectedType? : Option Expr` through every recursive call. When `expectedType?` is `some τ`, the elaborator can insert coercions, fill in implicit arguments by unification against `τ`, and guide typeclass synthesis. When it is `none` (inference mode), the elaborator must determine the type from the structure of the term.

**Typeclass synthesis** (`SynthInstance`) searches the `Environment` for `instance` declarations whose types unify with the required typeclass application. It uses a tabling-based search (similar to Prolog's SLD resolution) with cycle detection, implemented in [`src/Lean/Elab/SynthesizeConst.lean`](https://github.com/leanprover/lean4/blob/main/src/Lean/Elab/SynthesizeConst.lean). Failed typeclass synthesis results in `synthPendingDefault` insertion, which either inserts a default value or reports an error.

### 184.1.6 The `Lean.Meta.M` Monad: Unification and Reduction

`Lean.Meta.M` is the monad for operations that require metavariable manipulation and reduction. It extends `TermElabM`'s core state with a `ReducibilityCache` (recording whether definitions have been unfolded during a particular reduction sequence) and a configuration record `Meta.Config` that controls reduction behaviour (e.g., whether opaque definitions may be unfolded, whether η-reduction is applied).

**`whnf` (weak head normal form)**, implemented in [`src/Lean/Meta/WHNF.lean`](https://github.com/leanprover/lean4/blob/main/src/Lean/Meta/WHNF.lean), reduces an `Expr` to the point where its head is neither a reducible constant, a `bvar`, nor a β-redex. The reduction strategy is:
1. If the expression is a `const` with a transparent definition, δ-reduce (unfold).
2. If the expression is `app (lam ...)`, β-reduce.
3. If the expression is a recursor application where the major premise is a constructor application, ι-reduce.
4. If the expression is a `letE`, ζ-reduce (substitute).
5. If none of the above apply, the expression is in weak head normal form.

`whnf` is **lazy**: it stops at the outermost reducible position. For definitional equality checking, only the head needs to be in normal form; full normalization (which is undecidable in general for CIC due to non-terminating unfolding chains through impredicative quantification) is avoided.

**`isDefEq`**, implemented in [`src/Lean/Meta/DefEq.lean`](https://github.com/leanprover/lean4/blob/main/src/Lean/Meta/DefEq.lean), decides whether two expressions are definitionally equal. The algorithm proceeds:
1. **Syntactic check**: if the two expressions are pointer-equal or hash-equal, return `true` immediately.
2. **Metavariable assignment**: if either side is an assigned metavariable, substitute its assignment and recurse.
3. **whnf both sides**: reduce to weak head normal form.
4. **Head comparison**: if both heads are the same constructor/constant, compare arguments pairwise (up to arity). If heads differ, return `false` (unless a further reduction applies).
5. **Proof irrelevance**: if both expressions have type `P : Prop`, return `true` unconditionally.
6. **Quotient rules**: apply `Quot.lift` and `Quot.sound` reduction rules.
7. **Metavariable instantiation**: if one side is an unassigned metavariable, attempt **pattern unification** (determine if the metavariable can be assigned to make both sides equal). If pattern unification fails, record a **postponed constraint** for later resolution.

```lean4
-- Meta.M monad usage: isDefEq and whnf
open Lean Meta in
def checkDefEq (e₁ e₂ : Expr) : MetaM Bool := do
  let w₁ ← whnf e₁     -- reduce e₁ to weak head normal form
  let w₂ ← whnf e₂     -- reduce e₂ to weak head normal form
  isDefEq w₁ w₂         -- check definitional equality

-- Example: isDefEq (Nat.succ 2) 3
-- whnf (Nat.succ 2) = Nat.succ (Nat.succ (Nat.succ 0)) after ι-reduction
-- isDefEq compares constructors and recurses on arguments
```

### 184.1.7 Tactic Framework

Lean 4's tactic framework is built on `Lean.Elab.Tactic.TacticM`, which extends `TermElabM` with a **tactic state**: a list of `MVarId` values representing the current proof goals, each backed by a metavariable in the `MetavarContext`. The current goal's type and local context are retrieved from the `MetavarContext`. A tactic either:
- **Closes a goal** by assigning its metavariable to a proof term of the appropriate type.
- **Splits a goal** by replacing one `MVarId` with multiple new `MVarId` values (each of a subgoal type).
- **Fails** by throwing a `TacticException`, which the tactic combinator `first | t₁ | t₂` can catch for backtracking.

Key built-in tactics and their implementations:

| Tactic | Implementation file | Mechanism |
|--------|---------------------|----|
| `intro h` | `Lean.Elab.Tactic.Intro` | Introduces a `forallE` binder; creates an `fvar` in the local context |
| `apply e` | `Lean.Elab.Tactic.Basic` | Unifies goal type with `e`'s type conclusion; new goals for premises |
| `exact e` | `Lean.Elab.Tactic.Basic` | Elaborates `e` against the goal type; assigns the metavariable |
| `simp` | `Lean.Elab.Tactic.Simp` | Term-rewriting engine with congruence closure; produces `Eq.mpr` proof terms |
| `omega` | `Lean.Elab.Tactic.Omega` | Presburger arithmetic decision procedure (Cooper's algorithm) |
| `decide` | `Lean.Elab.Tactic.Decide` | Kernel-evaluated `Decidable` instance; trusted (kernel evaluates) |
| `native_decide` | `Lean.Elab.Tactic.Decide` | Native-compiled `Decidable`; fast but adds to TCB (native code not kernel-checked) |
| `aesop` | external `Aesop` package | Configurable best-first proof search over `@[aesop]`-tagged rules |
| `norm_num` | `Lean.Elab.Tactic.NormNum` | Normalised numeric arithmetic over `Nat`, `Int`, `Rat`, `Real` |
| `ring` | `Mathlib.Tactic.Ring` | Commutative ring equations via polynomial normalization |

**`simp`** deserves detailed examination because it is the most frequently-used tactic in Lean 4's Mathlib4. It maintains a `SimpTheorems` structure — a discrimination tree (a first-order trie indexed by function head symbol) over all `@[simp]`-tagged lemmas. For each subterm, simp extracts the head symbol, looks up applicable lemma schemas, attempts unification with the left-hand side, and if successful, replaces the subterm with the right-hand side instantiation. The correctness argument is: each rewrite step produces an `Eq.mpr` application (rewriting the goal type using a proved equality), which is a valid kernel term. The kernel checks the entire resulting proof term; `simp` itself is not in the trusted base.

**`omega`** implements Cooper's decision procedure for Presburger arithmetic (quantifier-free linear integer and natural-number arithmetic). It translates the goal to a conjunction of linear inequalities over integers, runs the Omega test or a variant, and if a proof is found, constructs an explicit `Nat.le_antisymm` / `Int.eq_of_lt_of_lt` proof term. The tactic can handle goals involving `Nat.succ`, `+`, `*` (by a literal), `%`, `/` (by a literal), `min`, `max`.

### 184.1.8 Compilation Pipeline: LCNF to C and LLVM

Lean 4 is a programming language as well as a proof assistant, and terms in types other than `Prop` are compiled to native code via a multi-stage pipeline:

```
Lean.Expr (in Environment)
    → Lean.Compiler.LCNF.toDecl       -- A-normalize; erase Prop arguments
    → LCNF optimization passes         -- inlining, DCE, specialization, float-out
    → RC insertion                     -- lean_inc/lean_dec on lean_object*
    → Lean.Compiler.IR.EmitC           -- emit C with lean.h includes
    → system cc (GCC or Clang)         -- native binary / .so
    → (experimental) LLVM IR emitter   -- direct LLVM IR generation
```

**LCNF** (λ-Compiler Normal Form), defined in [`src/Lean/Compiler/LCNF/`](https://github.com/leanprover/lean4/tree/main/src/Lean/Compiler/LCNF), is an **administrative normal form** (ANF) IR. In ANF, every intermediate result is bound to a name via a `let`-binding; all arguments to function applications are atomic (variables or literals). This property simplifies reference-counting insertion because every heap allocation site is an explicit named `let` binding.

```lean4
-- Source: List.map f (x :: xs)
-- After ANF (LCNF) transformation, conceptually:
let r₁ : β := f x              -- the mapped element (explicit call)
let r₂ : List β := List.map f xs  -- the recursive result (explicit call)
let r₃ : List β := r₁ :: r₂   -- construct result cell
r₃
-- RC annotation inserts lean_inc r₁ if r₁ escapes into r₃ and is still live
```

The **LCNF optimization passes** include:
- **Inlining**: small definitions (≤ 32 LCNF nodes by default) are inlined at call sites; this enables further optimizations.
- **Dead code elimination (DCE)**: bindings whose results are never used (or whose types are `Prop`) are removed.
- **Specialization**: polymorphic functions applied to specific type arguments generate monomorphic specializations. `Array.map (f : Nat → Nat)` is specialized to avoid the polymorphic dispatch overhead.
- **`join` point optimization**: tail-recursive branches that share a common continuation are hoisted to a `join` point (similar to LLVM's `musttail` optimization).

After LCNF optimization, the **reference-counting insertion pass** analyzes each binding's use-count and inserts `lean_inc`/`lean_dec` calls. Lean 4 uses a **functional but in-place** (FBIP) strategy: if the reference count of a boxed object is 1 at the point of destructuring (pattern matching on a constructor), the object can be reused in-place rather than allocated anew. This is the same technique used in Koka and described in the paper "Counting Immutable Beans" (Ullrich & de Moura, 2020).

The **C emission pass** (`src/Lean/Compiler/IR/EmitC.lean`) translates LCNF with RC annotations to a `.c` file that `#include`s `<lean/lean.h>`. The emitted C uses the Lean runtime API: `lean_ctor_get` (read constructor field), `lean_ctor_set` (write constructor field in-place when RC=1), `lean_apply_n` (apply a closure to `n` arguments), `lean_alloc_ctor` (allocate a new constructor node). This C file is compiled by the system's `cc` to produce a `.o` file linked into the final binary or shared library.

The **experimental LLVM backend** (proposed in [PR #1837](https://github.com/leanprover/lean4/pull/1837)) bypasses C emission and generates LLVM IR directly from LCNF. As of April 2026, the backend handles the standard LCNF constructs (arithmetic, constructor operations, function calls) and achieves performance comparable to the C backend on numeric benchmarks. The primary technical obstacle is the `lean_object*` tagged-pointer representation: the Lean runtime functions (`lean_ctor_get`, `lean_is_scalar`, `lean_unbox_uint64`, etc.) are defined in `lean/runtime/object.h` as inline C functions or macros. When generating LLVM IR directly, these definitions must either be inlined explicitly in the emitter or provided as LLVM bitcode that is linked and merged with LTO. The PR includes the runtime as LLVM bitcode and uses `llvm::Linker::linkModules` at compile time; the approach works but increases per-file compile times. The inability to inline `lean_object` intrinsics without LTO means the backend cannot yet deliver the speculative optimizations (tag-check elimination, allocation elision) that would make it significantly faster than the C path.

### 184.1.9 Bootstrapping and Lake

Lean 4 bootstraps in two stages:

- **Stage 0**: a pre-built C snapshot of the Lean 4 compiler, committed under `stage0/`. This binary is used to compile Lean 4 from source during the first stage build.
- **Stage 1**: the Lean 4 compiler compiled from source by stage 0. Stage 1 is then used to compile Lean 4 a second time (`lean4 stage2`) to verify that the bootstrap is self-consistent — the stage 1 and stage 2 binaries must produce identical outputs.

```bash
# Bootstrap build
cmake -S . -B build -GNinja -DSTAGE=1 -DCMAKE_BUILD_TYPE=Release
ninja -C build lean
# Stage 2 consistency check (optional, ~30 min)
cmake -S . -B build2 -GNinja -DSTAGE=2
ninja -C build2 lean
diff build/bin/lean build2/bin/lean  # must be identical
```

The two-stage bootstrap is **not** a full self-hosting proof. Stage 0's C code is not verified by Lean; it is a trusted artefact produced by a previous developer snapshot. Full trust in the compiler requires the Lean4Lean work described in the next section.

**Lake** is Lean 4's build system and package manager, implemented in Lean 4 itself under [`src/lake/`](https://github.com/leanprover/lean4/tree/main/src/lake). A `lakefile.lean` declares packages, targets, library roots, and external dependencies in plain Lean syntax:

```lean4
import Lake
open Lake DSL

package mylib where
  srcDir := "src"
  precompileModules := true     -- cache .olean files across builds

lean_lib MyLib where
  globs := #[.andSubmodules `MyLib]

require mathlib from git
  "https://github.com/leanprover-community/mathlib4" @ "v4.12.0"
```

Lake resolves dependencies via git, downloads pre-built `.olean` caches from a content-addressable store keyed by Lean version and package hash, and invokes Lean's elaborator in parallel across independent files. The `elan` version manager (analogous to `rustup`) manages Lean toolchain installations.

### 184.1.10 Lean4Lean and Proof of the Kernel

`lean4checker` is a standalone Lean 4 binary that re-checks a compiled `.olean` file against the kernel. It is used to verify that a library's proofs are kernel-accepted without trusting the elaborator's incremental caching — a faster version of deleting all `.olean` files and rerunning from scratch.

More ambitious is **Lean4Lean** ([arXiv 2403.14064](https://arxiv.org/abs/2403.14064), Mario Carneiro, 2024), a verified reimplementation of the Lean 4 typechecking algorithm **written in Lean 4 itself**. Lean4Lean implements a Lean 4 kernel — the `check` function, definitional equality, `whnf`, universe constraint solving — in Lean 4, and proves the following **soundness theorem**: if the Lean4Lean checker accepts a declaration, then the formal type theory (specified as an inductive relation in Lean 4) validates the declaration. The project then runs this verified checker on actual Mathlib4 `.olean` exports and reports that they pass.

Lean4Lean embodies the **de Bruijn criterion** at the meta-level: the proof of soundness is itself checked by the Lean 4 kernel, so trust reduces entirely to the C++ kernel (~6,000 lines). For the first time, there is a machine-checked argument that the Lean 4 elaborator and kernel are mutually consistent — not just empirically, but as a mathematical theorem within the same system being verified.

### 184.1.10b Mathlib4: Scale as a Correctness Argument

**Mathlib4** ([https://github.com/leanprover-community/mathlib4](https://github.com/leanprover-community/mathlib4)) is the primary Lean 4 mathematical library, maintained by a community of ~500 contributors and containing approximately 1,400,000 lines of Lean 4 as of April 2026. It covers undergraduate-through-graduate mathematics: algebra (rings, fields, modules, categories), number theory (primes, L-functions, class groups), analysis (real and complex analysis, measure theory, Fourier analysis), topology (metric spaces, manifolds), and combinatorics. The formalized **Fermat–Catalan conjecture cases**, the **Polynomial Freiman-Ruzsa theorem** (Gowers, Green, Manners, Tao, 2023 — formalized within months of the original proof), and a substantial fraction of the Putnam competition problems are in Mathlib4.

The Mathlib4 scale serves as a **stress test** for Lean 4's kernel, elaborator, and `simp` infrastructure: if there were a soundness bug in `Lean.Expr` handling, `isDefEq`, or universe level normalization, it would almost certainly have been found in 1.4 million lines of formalized mathematics. This empirical argument is not a substitute for the formal soundness proof (Lean4Lean), but it provides strong practical confidence. Similarly, Lean 4's performance characteristics — `simp` speed, `omega` effectiveness, typeclass synthesis overhead — are continuously profiled against Mathlib4, making it the de facto performance benchmark for the system.

### 184.1.11 LeanDojo and Proof Search

**LeanDojo** ([https://leandojo.org](https://leandojo.org)) is a framework for machine learning–guided proof search in Lean 4. It instruments the Lean 4 elaborator to extract the tactic state (local hypotheses, goal type, the set of relevant Mathlib4 lemmas computed by a premise retriever) at each proof step, assembles a large dataset of (tactic state, tactic, resulting states) triples, and trains transformer models (ReProver and its successors) to predict the next tactic. During proof search at inference time, LeanDojo's client submits tactic predictions via the Lean 4 REPL API; the **kernel evaluates each prediction** and returns success or failure. The kernel remains the sole arbiter of correctness regardless of what the neural model suggests — a hallucinat proof step fails cleanly at the kernel gate.

This architecture connects to [Chapter 181](../part-26-ecosystem-frontiers/ch181-formal-verification-in-practice.md)'s discussion of LLM-aided verification: the LLM does not need to be correct; it only needs to be a useful generator of plausible tactic candidates. The kernel filters out all incorrect ones.

---

## 184.2 Coq/Rocq Architecture

Coq — rebranded **Rocq** in 2024 as described in §184.2.7 — is the oldest of the three systems treated here, originating from Thierry Coquand and Gérard Huet's work on the Calculus of Constructions at INRIA in the mid-1980s, with inductive types added by Christine Paulin-Mohring in the 1990s. The resulting type theory, pCIC, has been a workhorse of verified compilation since the original CompCert work of 2006. Its source lives at [https://github.com/coq/coq](https://github.com/coq/coq), implemented in OCaml.

### 184.2.1 The Calculus of Inductive Constructions

Coq's type theory is **pCIC** (the predicative Calculus of Inductive Constructions), an extension of CIC with:

- **Cumulative universe hierarchy**: `Prop : Type(0) : Type(1) : ...` with **cumulativity**: if `τ : Type(i)` then `τ : Type(i+1)`. This means `Type(i)` is a subtype of `Type(i+1)` — a type can be used where a larger type is expected.
- **`Prop` as impredicative**: `∀ (x : T), P x` is in `Prop` even if `T : Type(i)` for large `i`. This impredicativity enables classical reasoning about sets of arbitrary size, which is essential for mathematical formalizations (e.g., quantifying over all sets of natural numbers).
- **`SProp` (strict propositions)**, introduced in Coq 8.10: a more restricted universe where proof-irrelevance is definitional (not just propositional). Terms in `SProp` are erased completely during extraction, with zero runtime cost. `SProp` is distinct from `Prop` in that even `Prop`-valued functions with `SProp` arguments are eliminated.
- **`Set`**: historically `Set = Type(0)` in Coq. The `Set Impredicative Set` flag (rarely used, and not in standard libraries) makes `Set` behave like `Prop` for quantification.
- **Inductive types**: defined by constructor schemas. The **positivity checker** enforces that recursive occurrences of the defined type may only appear to the right of all arrows in constructor argument types — the strict positivity condition that prevents Girard's paradox.

The rule distinction between `Prop` and `Set`/`Type` matters for extraction: `Prop` content is erased (proof terms are replaced by `()`); `Set` and `Type` content is retained and extracted to executable OCaml/Haskell code.

### 184.2.2 The Kernel

Coq's kernel is implemented across three OCaml files:

- [`kernel/typeops.ml`](https://github.com/coq/coq/blob/master/kernel/typeops.ml): type inference (`infer`), type checking (`check`), and declaration consistency (`check_constant_declaration`).
- [`kernel/reduction.ml`](https://github.com/coq/coq/blob/master/kernel/reduction.ml): definitional equality via reduction to weak head normal form (`whd_betaiotazeta`, `whd_betadeltaiota`).
- [`kernel/inductive.ml`](https://github.com/coq/coq/blob/master/kernel/inductive.ml): checking inductive type declarations, their constructors' types, and recursor elimination rules.

The kernel is approximately 10,000 lines of OCaml — the intentional result of years of refactoring to keep it auditable. The main entry points:

```ocaml
(* kernel/typeops.ml *)
val infer    : env -> constr -> unsafe_judgment
  (* infer the type of a term; returns term and its type *)
val check    : env -> constr -> types -> unit
  (* check that a term has a given type; raises TypeError on failure *)

(* kernel/inductive.ml *)
val check_inductive : env -> mutual_inductive_entry -> mutual_inductive_body
  (* validate a new inductive type declaration *)
```

The kernel's definitional equality uses five reduction rules — β, δ, ι, ζ, η — identical in name and semantics to Lean 4's five rules, though the reduction strategies differ. Coq's kernel distinguishes between **transparent** and **opaque** constants: `Transparent` definitions are δ-unfolded; `Opaque` ones (declared with `Qed` rather than `Defined`) are not unfolded by the kernel's definitional equality check. This `Opaque`/`Transparent` distinction is a performance engineering choice: large proofs whose internal structure should never be exposed can be made opaque to prevent runaway δ-reduction during type checking.

### 184.2.3 Guard Checker for Fixpoints

Coq's **fixpoint guard checker**, implemented in [`kernel/guard.ml`](https://github.com/coq/coq/blob/master/kernel/guard.ml) (approximately 1,200 lines), verifies that recursive definitions terminate. This is essential: in a system where proofs are programs, a non-terminating `fix` could loop forever during type checking, causing the kernel to hang — or worse, with `False.elim` as the infinite loop, could silently accept an inconsistent theorem.

The **structural recursion** check is the primary mechanism: the checker tracks which variable is the "principal argument" (the decreasing one) through each recursive call. In each arm of the `match`, the bound variables introduced by the pattern are recorded as **strict subterms** of the matched value. The recursive call's principal argument must be a strict subterm. For example:

```coq
Fixpoint factorial (n : nat) : nat :=
  match n with
  | O    => 1
  | S n' => S n' * factorial n'  (* n' : strict subterm of n *)
  end.
(* Guard checker: factorial is applied to n', which is a subterm of S n' *)
```

For more complex patterns — mutually recursive functions, recursion over complex inductive types — the guard checker performs a more elaborate analysis tracking subterm orderings through constructor projections. The `Program Fixpoint` mechanism (via the `Equations` plugin) handles well-founded recursion over arbitrary orderings by reducing to structural recursion over a well-foundedness proof.

The guard checker is part of the kernel's trusted base because any unsound fixpoint could be used to produce a proof of `False` by non-termination during type-checking (the "looping" exploit, which could cause the kernel's inference algorithm to diverge rather than return a type).

### 184.2.4 Metaprogramming: Ltac, Ltac2, and SSReflect

Coq's tactic language has evolved through two generations plus a major library tradition:

**Ltac** is the original tactic metalanguage, introduced in Coq 8.0. It is a dynamically-typed scripting language with backtracking (`first [t₁ | t₂]`), pattern matching on proof contexts (`match goal with | [ H : _ ∧ _ |- _ ] => destruct H end`), and open-term construction. Ltac is not in the kernel's trusted base; every tactic eventually produces an explicit `constr` (a Coq term) that the kernel checks. Ltac's dynamic typing means that a tactic script that looks correct can fail with an inscrutable error when the goal's structure does not match the script's assumptions — a common source of fragility in large Coq developments.

**Ltac2**, introduced in Coq 8.11 and stabilised in Coq 8.17, is a statically-typed tactic language. It has:
- An ML-like type system with `int`, `string`, `constr`, `tactic`, `unit`, `list`, `option`, and user-defined algebraic types.
- First-class tactics: a value of type `tactic` is a Lean 4-style function from tactic state to tactic state.
- Typed pattern matching on `constr` values using the `match! goal` notation.
- A typed FFI to OCaml via `external` declarations, allowing performance-critical parts of a tactic to be written in OCaml while maintaining a typed interface.

```coq
(* Ltac2 example: a typed destructor tactic *)
Ltac2 destruct_and () :=
  match! goal with
  | [ h : _ /\ _ |- _ ] =>
    let (h1, h2) := destruct h in
    exact (pair h1 h2)
  end.
```

**SSReflect** (Small-Scale Reflection), originally developed by Georges Gonthier and Assia Mahboubi for the formalization of the Four-Colour Theorem (2005–2012), is now part of the [Mathematical Components](https://math-comp.github.io) library distributed with Coq. SSReflect introduces a radically different proof style: instead of manipulating the proof context with tactics like `destruct` and `induction`, the developer uses boolean decision procedures directly as proof steps via the **reflect** predicate:

```coq
(* SSReflect-style: reflect connects Prop to bool *)
Lemma leq_reflP n : reflect (n <= n) (n <=? n).
Proof. apply iff_reflect. exact (Nat.leb_refl n). Qed.

(* Using reflect for case analysis *)
From mathcomp Require Import ssreflect ssrnat.
Lemma addn_comm (n m : nat) : n + m = m + n.
Proof. by elim: n => [//| n IHn]; rewrite addSn IHn addnS. Qed.
```

The `by` at the start of a proof branch means "close this goal by the following sequence of tactics or fail." SSReflect's `//` means "try `trivial` or `reflexivity`"; `//=` adds simplification. The discipline of always applying a single finishing tactic makes SSReflect proofs more robust to library changes than goal-oriented Ltac scripts.

### 184.2.5 The Extraction Mechanism

Coq's **extraction mechanism**, implemented in [`plugins/extraction/`](https://github.com/coq/coq/tree/master/plugins/extraction) (approximately 6,000 lines of OCaml), translates verified Coq programs to executable code. The three target languages are OCaml (the most mature, used by CompCert), Haskell, and Scheme. The extraction algorithm:

1. **Type erasure**: terms in `Prop` and `SProp` are erased to a special `Obj.magic` unit value in OCaml. A function `∀ (h : P), f h` where `P : Prop` becomes `fun _ -> f ()` in OCaml.
2. **Universe erasure**: universe polymorphism is erased; `Type u` becomes OCaml's generic `'a`.
3. **Coinductive handling**: Coq coinductive types are extracted to OCaml `Lazy.t` streams; `cofix` body is wrapped in `lazy`.
4. **Module extraction**: Coq modules map to OCaml modules; functors map to OCaml functors.

```coq
(* CompCert: extracting the verified compiler *)
Require Import Compiler.
Extraction Language OCaml.
Extraction "compcert.ml" transf_c_program.
(* Output: compcert.ml contains:
   val transf_c_program :
     Csyntax.program -> Asm.program Errors.res
   implemented entirely in terms of pattern-matching on inductive types *)
```

Extraction is not in the kernel's trusted base, but it is in the **trusted base of extracted programs**. The Coq community has addressed this with the **verified extraction** project: a Coq formalization of the extraction algorithm's correctness, partially completed as of 2026, aiming to prove that extraction preserves the denotational semantics of extracted terms.

**CompCert** ([Chapter 168](../part-24-verified-compilation/ch168-compcert.md)) uses extraction as the bridge from Coq proof to executable compiler. The entire CompCert pipeline (`SimplExpr.v`, `Csharpminor.v`, ..., `Asmgen.v`) consists of Coq functions extracted to OCaml and compiled by `ocamlopt` to produce the `ccomp` binary. The trusted base is: Coq kernel + extraction plugin + OCaml compiler — dramatically smaller than trusting GCC or LLVM's optimizer.

### 184.2.6 `native_compute` and the `Equations` Plugin

**`native_compute`**, introduced in Coq 8.4, compiles a Coq term to native OCaml bytecode, evaluates it, and returns a kernel-checked proof that the original term reduces to the result. The crucial invariant: `native_compute` only claims `a = b`; the kernel checks the proof (a `refl` of the normal form). Therefore `native_compute` is not in the kernel's trusted base — it is a **performance-critical but non-trusted** optimization, analogous to Lean's `native_decide`.

```coq
(* native_compute for large normalizations *)
Lemma mersenne_prime : Nat.prime (2^31 - 1).
Proof. native_compute. reflexivity. Qed.
(* Without native_compute, this would time out using vm_compute *)
```

`vm_compute` (the bytecode VM, available since Coq 8.1) is similar but uses Coq's abstract machine bytecode rather than native OCaml. `vm_compute` is slightly slower than `native_compute` but requires no OCaml compilation step and is available in all Coq deployments.

**`Equations`** (Sozeau, since 2010) is a Coq plugin providing dependent pattern matching with automatically-generated induction principles. Standard `match` in Coq is limited: it cannot easily express structural recursion over nested types, complex overlapping patterns, or dependent eliminations where the motive depends on which branch was taken. `Equations` addresses this by:
- Translating complex match+fix combinations into sequences of auxiliary lemmas.
- Generating a `funelim` tactic for reasoning about functions defined with `Equations` by their definitional equations.
- Supporting well-founded recursion via explicit orderings.

```coq
From Equations Require Import Equations.
Equations zip {A B : Type} (l₁ : list A) (l₂ : list B) : list (A * B) :=
  zip [] _         := [];
  zip _ []         := [];
  zip (x :: xs) (y :: ys) := (x, y) :: zip xs ys.
```

### 184.2.7 The Coq/Rocq Renaming

The renaming from **Coq** to **Rocq** was announced at CoqPL 2024 (co-located with POPL 2024) and formalized by the Coq steering committee in mid-2024. The motivation was to remove the French-language double entendre while preserving the pronunciation. The new spelling "Rocq" echoes the French chess term *rocque* (the rook / castle move), fitting a system named for formal reasoning.

As of April 2026, the transition is partially complete: the binary remains `coqc`, the GitHub repository is `coq/coq`, distribution packages are labeled `coq`, and most existing literature refers to "Coq." The Rocq Prover website ([https://rocq-prover.org](https://rocq-prover.org)) is the official new home. Both names are acceptable in technical writing; this chapter uses "Coq/Rocq" to reflect the transition period.

---

## 184.3 Isabelle/HOL Architecture

Isabelle is a generic proof assistant developed by Lawrence Paulson at Cambridge and Tobias Nipkow at TU München, with the Isar proof language and many core tactics contributed by Makarius Wenzel. It is implemented in Standard ML (using the Poly/ML compiler) and distributed via [https://isabelle.in.tum.de](https://isabelle.in.tum.de). Isabelle's architecture is fundamentally different from Lean 4 and Coq: it is a **generic logical framework** (the metalogic Pure) on which specific object logics (HOL, ZF, CTT) are built as libraries. In practice, nearly all users work with the HOL instantiation.

### 184.3.1 The Metalogic Pure

Isabelle's foundation is the **Pure metalogic**: a fragment of intuitionistic higher-order logic containing exactly:

- **Types**: simple types (function types `α ⇒ β`, type variables `'a`). Pure has **no dependent types** — it is a simply-typed metalogic.
- **The metatype `prop`**: the type of propositions in the metalogic.
- **Universal quantification** `⋀x :: α. P x` (written `!!x :: α. P x` in ASCII): quantify over all values of type `α`.
- **Implication** `A ⟹ B` (written `A ==> B`): the standard → of intuitionistic logic.
- **Equality** `a ≡ b`: definitional equality in Pure (used by the kernel for definitional steps).
- **Schematic variables** `?x`, `?P`: unified during proof steps. Schematic variables give theorems their generality: the axiom schema `?A ⟹ ?A` applies to any proposition `?A`.

Pure has no induction, no data types, no classical logic. These are all added by object logics built on top. This stratification means that the soundness of any object logic (including HOL) reduces to the soundness of Pure's four inference rules plus the object logic's axioms.

**HOL** is built on Pure via [`src/HOL/HOL.thy`](https://isabelle.in.tum.de/repos/isabelle/file/Isabelle2024/src/HOL/HOL.thy). The HOL axioms are:
- `ext`: function extensionality (`∀x. f x = g x ⟹ f = g`).
- `refl`: equality reflexivity.
- `subst`: substitution in equality.
- `select`: Hilbert's epsilon (`∃x. P x ⟹ P (SOME x. P x)`) — the axiom of choice.
- `excluded_middle`: `P ∨ ¬P` — classical logic.

The inclusion of `excluded_middle` and `select` distinguishes Isabelle/HOL from Lean 4 and Coq (which are constructive by default). Classical logic makes it easier to reason about C code and hardware, where the law of excluded middle is assumed throughout.

### 184.3.2 The `Thm.t` Type: LCF Architecture

The **LCF approach** (Logic for Computable Functions, Edinburgh 1970s, Milner) is Isabelle's architectural invariant. In Isabelle's Poly/ML implementation, the type `thm` (in [`src/Pure/thm.ML`](https://isabelle.in.tum.de/repos/isabelle/file/Isabelle2024/src/Pure/thm.ML)) is **abstract**: its internal constructor (a record of the proposition, the hypothesis set, and a sort-constraint record) is not visible outside the kernel module. The only way to produce a value of type `thm` is through the **primitive inference rules** exported by the `Thm` structure:

```sml
(* Isabelle kernel interface — simplified from src/Pure/thm.ML *)
signature THM =
sig
  type thm  (* abstract *)

  (* Primitive introduction rules — the only routes to thm values *)
  val assume      : cterm -> thm
    (* assume A: produces the theorem A ⊢ A *)
  val implies_intr: cterm -> thm -> thm
    (* given H ⊢ B, produce ⊢ A ⟹ B by discharging hypothesis A = H *)
  val implies_elim: thm -> thm -> thm
    (* ⊢ A ⟹ B and ⊢ A give ⊢ B (modus ponens) *)
  val forall_intr : cvar -> thm -> thm
    (* ⊢ P x with x fresh gives ⊢ !!x. P x *)
  val forall_elim : cterm -> thm -> thm
    (* ⊢ !!x. P x and term t give ⊢ P t (universal instantiation) *)
  val reflexive   : cterm -> thm
    (* produce ⊢ t ≡ t *)
  val symmetric   : thm -> thm
    (* ⊢ t ≡ u gives ⊢ u ≡ t *)
  val transitive  : thm -> thm -> thm
    (* ⊢ t ≡ u and ⊢ u ≡ v give ⊢ t ≡ v *)
  val beta_conversion: cterm -> thm
    (* ⊢ (λx. t) u ≡ t[u/x]  (beta reduction as a theorem) *)
  val eta_conversion : cterm -> thm
    (* ⊢ (λx. f x) ≡ f  (eta reduction as a theorem) *)
end
```

Any tactic, proof method, or external solver that produces a `thm` value must go through these primitives. This is the **LCF invariant**: it is structurally impossible for a bug in the tactic language, the Sledgehammer integration, or any user-defined proof method to produce an incorrect `thm` value without passing through kernel code. The type system of ML enforces this invariant syntactically — it is not a runtime check.

The Isabelle kernel is implemented in approximately 6,000 lines of Poly/ML spanning `src/Pure/thm.ML`, `src/Pure/term.ML`, `src/Pure/type.ML`, and `src/Pure/sign.ML`. This is small enough to audit; it has been independently reviewed multiple times and no soundness bugs have been found in the core since the 1990s (the `subtype_def` soundness issue found in 2015 was in HOL's definitional mechanisms, not Pure's kernel).

### 184.3.3 Isar: Structured Proof Language

**Isar** (Intelligible semi-automated reasoning), designed by Makarius Wenzel, is Isabelle's structured proof language. Unlike Ltac or Lean's tactic combinators (which are imperative state transformers on a goal list), Isar proofs are **declarative**: each step produces a named theorem that can be cited in subsequent steps, and the proof is readable as a mathematical argument.

```isabelle
theorem sqrt_2_irrational: "(sqrt 2) ∉ ℚ"
proof
  assume "sqrt 2 ∈ ℚ"
  then obtain p q :: int
    where "q ≠ 0" and "sqrt 2 = p / q" and "gcd (nat (abs p)) (nat (abs q)) = 1"
    by (auto simp: Rats_def)
  hence "2 * q^2 = p^2"
    by (auto simp: field_simps power2_eq_square)
  hence "2 dvd p"
    by (metis even_iff_2_dvd odd_sq)
  then obtain k where "p = 2 * k" by auto
  with ‹2 * q^2 = p^2› have "2 dvd q"
    by (auto simp: power2_eq_square)
  with ‹gcd (nat (abs p)) (nat (abs q)) = 1› show False
    by (auto simp: gcd_eq_1_imp_not_dvd)
qed
```

**Isar keywords**:
- `proof` / `qed`: opens/closes a proof block.
- `have` `name: "statement" by method`: proves an intermediate lemma.
- `show "conclusion" by method`: proves the main goal or final step.
- `obtain x where "P x" by ...`: introduces an existential witness.
- `fix x :: type`: universally quantifies over a fresh variable.
- `assume "A"`: introduces hypothesis `A` into the local context.
- `then` / `hence` / `thus`: forward-chaining from the immediately preceding statement.
- `with a b c`: brings named facts `a`, `b`, `c` into scope as premises.

The key design decision is that each `have` / `show` step is a complete proof obligation checked by the kernel before the next step is processed. If the `by method` fails, the error is at the specific step, not after a long chain of unverified reasoning. This contrasts with Ltac, where a tactic script can proceed for many steps before a later unification failure reveals that an earlier step was incorrect.

### 184.3.4 Sledgehammer: ATP Integration

**Sledgehammer** is Isabelle's automation mechanism for discharging subgoals by calling external Automated Theorem Provers (ATPs) in parallel. Given a current subgoal and the available facts, Sledgehammer performs:

1. **Relevance filtering**: uses the MaSh (Machine learning for Sledgehammer) module, implemented in `src/HOL/Tools/Sledgehammer/sledgehammer_mash.ML`, to select the ~512 most relevant facts from the Isabelle/HOL library. MaSh uses a k-nearest-neighbour retrieval over a dependency graph of previously-proved lemmas.

2. **Translation to TPTP**: translates the goal and selected facts to TPTP (Thousands of Problems for Theorem Provers) first-order format. HOL goals are approximated as first-order formulas; some expressivity is lost (higher-order functions, polymorphism) but the approximation is often sufficient.

3. **Parallel ATP calls**: simultaneously invokes Z3 (as an SMT oracle), CVC5, Vampire, E, and SPASS. Each ATP is given a 30-second timeout. The first ATP to succeed returns a proof certificate.

4. **Certificate reconstruction**: the certificate (a resolution proof, a DRUP certificate, or an SMT2 model) is translated back to Isabelle proof obligations and replayed through `metis` or `smt` tactics. `metis` uses a specialized higher-order resolution procedure; `smt` uses Z3 directly via the `z3` binding. Both ultimately produce sequences of primitive `thm.ML` calls.

```isabelle
-- After Sledgehammer finds a proof for "x + y = y + x":
by (metis add.commute)
-- or for more complex goals:
by (smt (verit, best) int_add_commute abs_triangle_ineq)
```

The design guarantee is: Sledgehammer calls arbitrary external solvers but **the solvers are untrusted**. A buggy or malicious ATP can only produce a certificate that fails reconstruction. The reconstruction through `metis`/`smt` → `thm.ML` primitives is the sole trust path. This is why Sledgehammer can safely call external tools without expanding the TCB.

### 184.3.5 Nitpick: Model Finding and Counterexample Search

**Nitpick** is Isabelle's counterexample finder, implemented in `src/HOL/Tools/Nitpick/`. Given a conjecture, Nitpick attempts to find a finite model that falsifies it. The pipeline:

1. **Translation to relational form**: the conjecture is translated to a relational formula over a finite domain (e.g., `{0,1,2}` for natural numbers).
2. **Kodkod**: the relational formula is passed to [Kodkod](https://github.com/emina/kodkod), a SAT-based bounded model finder. Kodkod encodes the relational formula as a SAT instance and calls MiniSat or Glucose.
3. **Rendering**: if a model is found, Nitpick renders it as Isabelle terms and presents a human-readable counterexample.

```isabelle
-- False conjecture: Nitpick finds a counterexample
lemma "∀ (f :: nat → nat) (x :: nat). f (f x) = x"
  nitpick
  (*  Nitpick found a counterexample for card nat = 3:
      Free variable: f = [0 := 0, 1 := 0, 2 := 2]
      x = 1                                          *)
  oops  -- abandon the false conjecture
```

Nitpick is in the **refutation layer**: found counterexamples are checked by evaluating the falsifying assignment in Isabelle's term evaluator, not by the kernel. If Nitpick cannot find a counterexample within its search bounds, the conjecture is plausibly true and worth attempting to prove — this is the primary workflow use: try Nitpick first to catch errors, then try Sledgehammer or an Isar proof.

### 184.3.6 The seL4 Verification: l4v and AutoCorres

**seL4** ([Klein et al., SOSP 2009](https://dl.acm.org/doi/10.1145/1629575.1629596), "seL4: Formal Verification of an OS Kernel") is a formally verified L4-family microkernel. Its correctness proof, the **l4v** (l4 verification) project ([https://github.com/seL4/l4v](https://github.com/seL4/l4v)), consists of approximately 200,000 lines of Isabelle/HOL proof and establishes three properties:

1. **Functional correctness**: every execution of the seL4 C code that terminates matches the behaviour specified by the abstract Isabelle/HOL specification.
2. **Integrity**: no userspace thread can modify kernel data structures it does not have permission to modify.
3. **Confidentiality**: information cannot flow from high-confidentiality threads to low-confidentiality threads in violation of the seL4 capability model.

The proof is structured as a two-level **refinement stack**:

```
Abstract specification
    (Isabelle/HOL: pure functional state machine over abstract kernel objects)
         ↓ refinement proof A (abstract → executable)
Executable specification
    (Isabelle/HOL: Haskell-prototype-style imperative model with explicit state monad)
         ↓ refinement proof B (executable → C implementation)
C implementation (~10,000 lines C)
    ↓ C parser (Norrish) → Simpl language (Isabelle/HOL)
    ↓ AutoCorres → monadic abstraction (with soundness proof)
```

**AutoCorres** ([https://github.com/seL4/l4v/tree/master/tools/autocorres](https://github.com/seL4/l4v/tree/master/tools/autocorres)) is a tool by Michael Norrish that translates C code (via the Isabelle C parser's Simpl embedding) to a monadic, type-safe Isabelle/HOL representation. The C parser translates seL4's C into `Simpl` — a shallow embedding of C's control flow in Isabelle/HOL where every statement is a Hoare triple. AutoCorres then abstracts this low-level representation: instead of a byte-level heap (a `word64` array indexed by address), the monadic abstraction exposes a typed heap where `ptr deref` operations return typed values. AutoCorres generates an Isabelle/HOL proof that the monadic abstraction **refines** the byte-level Simpl model.

Without AutoCorres, every seL4 C proof step would require manual reasoning about memory layout, pointer arithmetic, and byte-level representation. With AutoCorres, the seL4 C proof works with clean functional specifications.

The l4v proof's **trusted base** includes: Isabelle/HOL's Pure metalogic, the HOL axioms (including classical logic and the axiom of choice), the C parser (which translates C to Simpl — if the parser is wrong, the refinement proof does not apply to the actual C code), and AutoCorres's soundness argument. Notably absent from the trusted base: the C compiler (seL4 uses GCC, not CompCert). This means the proof covers the C source but not the compiled binary — a gap that CompCert-style verification could close.

### 184.3.6b Definitional Mechanisms and Consistency

Isabelle/HOL adds to Pure's four inference rules a set of **definitional extension mechanisms** that expand the theory safely. Each definitional command produces new axioms, but the axioms introduced are provably conservative: they do not add any theorems that were not already provable in the base theory (the conservativity argument is checked as part of the mechanism's implementation).

The definitional mechanisms in `src/HOL/Tools/`:

**`typedef`** introduces a new type as a non-empty subset of an existing type. For example, `typedef 'a fin_set = {s :: 'a set | finite s}` introduces `'a fin_set` as a copy of the finite subsets of `'a`, with axioms `Abs_fin_set :: 'a set ⇒ 'a fin_set` and `Rep_fin_set :: 'a fin_set ⇒ 'a set` satisfying the expected isomorphism laws. The proof obligation is to show the set `{s | finite s}` is non-empty (which is trivially satisfied by `{}`). The `typedef` mechanism is in HOL's definitional layer, not Pure's kernel — it is trusted by the community but has been a source of past soundness discussions.

**`definition`** introduces a new constant via an equation `c ≡ t`, where `t` is a closed term. The `≡` equation is added as a definitional axiom. Since `≡` is Pure's definitional equality, the new constant is transparently unfolded by the kernel's beta-eta conversion machinery.

**`fun` and `primrec`** introduce recursive function definitions. `primrec` generates equations for each constructor of the datatype; `fun` allows pattern-matching over multiple arguments with a well-foundedness proof. Both translate to a `definition` of a constant plus a set of `simp` lemmas (the computation equations).

**`inductive`** introduces inductive predicates via the least fixed-point construction. The implementation uses the Knaster-Tarski fixed-point theorem (proved in HOL) to produce an inductive definition whose introduction and elimination rules are automatically derived.

These mechanisms collectively constitute the "definitional layer" that sits between Pure's primitive rules and the mathematician-facing library. Their soundness is argued by conservativity, not by direct kernel verification — another contrast with Lean 4's kernel, which directly handles `inductive` type declarations.

### 184.3.7 AFP and Code Generation

The **Archive of Formal Proofs** ([https://www.isa-afp.org](https://www.isa-afp.org)) is a peer-reviewed collection of Isabelle/HOL formalizations with over 800 entries as of April 2026. Entries span: number theory (prime gaps, Bertrand's postulate), cryptography (RSA correctness, AES), graph algorithms (Dijkstra, Kruskal), automata theory (DFA minimization, Büchi automata), programming language semantics (IMP, CakeML, VeriCon), and architecture models. Every AFP entry is checked on every Isabelle release, providing a continuous regression test of both the formalizations and the Isabelle system.

Isabelle's **code generation** mechanism (`src/HOL/Tools/Codegen/`) translates Isabelle/HOL function definitions to executable code via:

```isabelle
(* Export a verified function to OCaml *)
export_code my_sort in OCaml module_name "Sorting" file "sorting.ml"

(* Or multiple targets simultaneously *)
export_code quicksort in
  SML module_name "Sort"
  OCaml module_name "Sort"
  Haskell module_name "Sort"
  Scala module_name "Sort"
```

The code generator uses **code refinements** to replace abstract datatype implementations with efficient concrete ones. For example, `Nat` (Peano natural numbers) is refined to machine integers; `List.sort` is refined to an in-place merge sort. Each refinement is an equation `code_unfold foo_abstract = foo_concrete` accompanied by an Isabelle/HOL proof of their equality. The kernel checks these equality proofs; the code generator itself is not in the trusted base.

---

## 184.4 Comparison and Connections

### 184.4.1 The de Bruijn Criterion

The **de Bruijn criterion** (N.G. de Bruijn, Automath project, Eindhoven, 1967–1977) states: a formal system is trustworthy if its **proof certificates** can be independently checked by a **small, auditable verifier**. The criterion demands not just that proofs are machine-checked, but that the checker itself is small enough for a human expert to read and verify manually. The checker is the **entire** trusted computing base for logical correctness; everything above it — tactic engines, ATP integrations, proof search, extraction mechanisms — may contain bugs without compromising the logical validity of accepted theorems.

De Bruijn applied this criterion to the Automath language by designing it so that proof checking required no backtracking or search — just a forward pass over a term. Modern proof assistants implement the criterion differently but the principle is the same: Lean 4's C++ kernel (~6,000 lines), Coq's OCaml kernel (~10,000 lines), and Isabelle's ML kernel (~6,000 lines) are all small enough for a careful audit, and the architecture ensures that no theorem can be accepted without passing through kernel code.

### 184.4.2 The Three Kernels Compared

| Property | Lean 4 | Coq/Rocq | Isabelle/HOL |
|----------|--------|----------|--------------|
| **Foundation** | CIC + quotient types | pCIC (`Prop`/`Set`/`Type`/`SProp`) | Pure metalogic + HOL axioms |
| **Kernel size** | ~6,000 lines C++ | ~10,000 lines OCaml | ~6,000 lines SML (Poly/ML) |
| **Kernel language** | C++ (kernel), Lean 4 (elaborator) | OCaml throughout | SML (Poly/ML) |
| **Universe hierarchy** | Polymorphic: `param`, `succ`, `max`, `imax`, `zero`, `mvar` | Cumulative: `Type(0)`, `Type(1)`, ... | None (simple types only) |
| **Definitional equality** | β, δ, ι, ζ, η + proof irrel. + quot rules | β, δ, ι, ζ, η + `Opaque`/`Transparent` | β, η only (kernel); δ via `thm.ML` lemma |
| **Induction** | Structural recursion via kernel recursor; `WellFounded` in library | Structural + guard checker (in kernel) | HOL axioms: primitive recursion, `wfrec` |
| **Termination** | Library `WellFounded` + `Equations` | Guard checker (kernel) + `Equations` | `termination` proof (library) |
| **Classical logic** | Not by default; `Classical.em` axiom available | `Classical` axiom available but not default | Yes, `excluded_middle` is an axiom of HOL |
| **Axiom of choice** | Not default; `Classical.choice` available | Available via `Classical.choice` | Yes, `Hilbert_Choice` is an axiom |
| **Extraction target** | C (stable), LLVM IR (experimental, PR #1837) | OCaml, Haskell, Scheme | SML, OCaml, Haskell, Scala |
| **Primary tactic automation** | `omega`, `simp`, `aesop`, `decide`, `norm_num` | `omega`, `ring`, `tauto`, `auto`, `eauto` | `auto`, `blast`, `fastforce`, Sledgehammer |
| **External ATP** | LeanDojo (proof search, kernel validates each step) | `coq-hammer` (Sledgehammer-like, WIP) | Sledgehammer (Z3, CVC5, Vampire, E, SPASS) |
| **Counterexample search** | `#eval` + QuickCheck-style (partial) | `QuickChick` plugin | Nitpick (Kodkod/SAT) |
| **Verified meta-kernel** | Lean4Lean (arXiv 2403.14064) | Partial work on verified extraction | None as of 2026 |
| **Primary large use** | Mathlib4, LLM proof search | CompCert, Vellvm, mathematical proofs | seL4 (l4v), Archive of Formal Proofs |

### 184.4.3 Why Each Project Made Its Choice

**CompCert chose Coq** for three interlocking reasons. First, Coq's extraction to OCaml was the only mature mechanism in 2003–2006 for producing an executable compiler from a verified specification. Isabelle's code generation was less mature; Lean 4 did not exist. Second, Coq's pCIC, with `Prop` as impredicative, supported the program-logic reasoning needed to state and prove the simulation theorems in `backend/Smallstep.v` — particularly the coinductive reasoning about program divergence. Third, Coq's dependent types allowed CompCert to write verified passes as Coq functions with machine-checked types, ensuring that every pass returned a semantically consistent program.

**Vellvm chose Coq** because it builds directly on CompCert's infrastructure. Vellvm reuses CompCert's memory model (`Memory.v`), its `Errors.res` monad for partial computation, and its simulation proof framework. Sharing these components required staying in the Coq ecosystem; porting CompCert's memory model to Lean 4 or Isabelle would have been a substantial independent effort.

**seL4 chose Isabelle** for reasons that reflect the nature of C verification. First, Isabelle/HOL includes classical logic and the axiom of choice as axioms, which match the reasoning style of C programmers and hardware designers — C code implicitly assumes that `if (p != NULL)` and `if (p == NULL)` are exhaustive. Second, Isabelle's `Simpl` language and C parser infrastructure existed before seL4 began its verification effort, providing a proven path from C source to Isabelle proof obligations. Third, Sledgehammer's ATP integration dramatically reduces the cost of routine lemmas — the l4v team has stated that Sledgehammer saved years of manual proof work on the 200,000-line proof corpus.

### 184.4.3b The HOL Family and Minimal Kernels

Isabelle/HOL belongs to the **HOL family** of proof assistants, all sharing Gordon's original LCF-style HOL logic. The family members and their kernel sizes illustrate that the de Bruijn criterion is achievable at various scales:

| System | Kernel size | Language | Notable properties |
|--------|-------------|----------|--------------------|
| **HOL4** | ~3,000 lines SML | SML (PolyML/Moscow ML) | Oldest active HOL derivative; used for ARM and RISC-V ISA verification; `opentheory` exchange format |
| **HOL Light** | ~600 lines OCaml | OCaml | Extremely minimal; Harrison's Flyspeck (Kepler conjecture formalization) used HOL Light; kernel can be understood in an afternoon |
| **Isabelle/HOL** | ~6,000 lines SML | Poly/ML | Generic framework; `Isar` proof language; Sledgehammer; largest automation ecosystem |
| **ProofPower** | ~5,000 lines SML | SML | Z/CSP specification language support; used in UK defense verification |

**HOL Light** deserves special mention as the most extreme application of the de Bruijn criterion: its kernel is approximately 600 lines of OCaml, making it arguably the simplest complete proof assistant in active use. Thomas Hales used HOL Light (supplemented by `flyspeck` tactics calling Mathematica and custom C++ for interval arithmetic) to complete the Flyspeck project — the formal proof of the Kepler conjecture (every packing of equal spheres has density ≤ π/√18) — spanning 20,000 lemmas and taking approximately 12 person-years. The fact that this was done with a 600-line kernel is the most vivid demonstration that logical complexity can be built atop a minimal trusted base.

**HOL4** has been used for the first fully machine-checked proof of the ARMv8-A ISA semantics (the official Arm architecture specification in HOL4 covers approximately 1,000 instruction encodings) and for the RISC-V formal model used in the RISC-V Foundation compliance test suite. These uses directly connect to LLVM backend verification: the HOL4 ARMv8 model is the ground-truth specification against which LLVM AArch64 code generation correctness could theoretically be checked.

### 184.4.4 Extracted and Compiled Code in the LLVM Pipeline

All three proof assistants can produce OCaml, Haskell, SML, or Scala code that is subsequently compiled by a conventional compiler. If the downstream compiler is LLVM-based — GHC with the LLVM backend, OCaml with the LLVM output plugin, the Scala native backend, or Rust — the formal verification guarantee terminates at the source level. The compiled binary is in the trusted base of the deployment.

This matters concretely. HACL* (the verified cryptographic library from F*) produces verified C code compiled by Clang. The Alive2 work ([Chapter 170](../part-24-verified-compilation/ch170-alive2-and-translation-validation.md)) has found LLVM misoptimization bugs that could in principle corrupt HACL*-verified code in its compiled form. The chain of trust from the F* or Coq proof to the executing binary includes the LLVM optimizer — every Alive2 misoptimization fix strengthens this chain.

The **Lean 4 LLVM backend** (PR #1837, §184.1.8) makes this connection direct in the future: Lean programs compiled to LLVM IR could become self-verifying substrates, but the current approach means both the Lean compiler's LCNF-to-IR translation and the system C compiler (when taking the C path) are in the trusted base of any Lean-generated binary. Closing this gap requires either CompCert-style verification of Lean's code generation or a verified Lean 4 LLVM backend — a research project that does not exist as of April 2026.

### 184.4.5 Connection to Type Theory (Chapters 12–15)

The three systems are concrete instantiations of the abstract type theory of [Chapters 12–15](../part-03-type-theory/):

[Chapter 12](../part-03-type-theory/ch12-lambda-calculus-and-simple-types.md) introduces the untyped λ-calculus and simple type theory. Lean 4's `Lean.Expr` AST is the implementation of the λ-cube: `forallE` is the dependent product (Π-type), `lam` is λ-abstraction, `app` is application, and the de Bruijn index representation for `bvar` is the standard nameless representation from Barendregt's *The Lambda Calculus: Its Syntax and Semantics*.

[Chapter 13](../part-03-type-theory/ch13-polymorphism-and-type-inference.md) covers polymorphism and type inference. Lean 4's `mvar` nodes and the `isDefEq` algorithm implement higher-order pattern unification — a decidable subset of higher-order unification (full higher-order unification is undecidable, as shown by Huet 1973). Universe polymorphism (`Level.param`) extends System F's type-level polymorphism to the universe level.

[Chapter 14](../part-03-type-theory/ch14-advanced-type-systems.md) discusses advanced type systems including substructural types and effect types. Coq's `SProp` is a form of irrelevant/erased types; Lean 4's proof irrelevance rule makes all `Prop` inhabitants definitionally equal. Quotient types (`Quot r`) in Lean 4 are the type-theoretic equivalent of the quotient construction in set theory — types modulo an equivalence relation.

[Chapter 15](../part-03-type-theory/ch15-type-theory-in-practice.md) connects theory to practice. The practical tactics — `simp`, `omega`, `Sledgehammer` — are the operational faces of the decidability results from Chapters 12–15. `omega` implements Cooper's decision procedure for Presburger arithmetic (which is a decidable fragment of arithmetic, as discussed in Chapter 185 on mathematical logic). `simp` implements conditional term rewriting, sound by construction via the `Eq.mpr` proof term mechanism.

### 184.4.5b Proof Terms vs. Proof Certificates: Practical Performance Implications

A fundamental architectural difference between the three systems affects practical verification performance: whether the system retains explicit **proof terms** (Lean 4, Coq) or works with **proof certificates** that are checked but not stored (Isabelle in certain modes).

In Lean 4 and Coq, every accepted proof produces a fully explicit proof term — a lambda-calculus term whose type is the proved proposition. These proof terms are stored in compiled `.olean` or `.vo` files. For large developments like Mathlib4 (Lean 4) or the Mathematical Components library (Coq), these proof term files can be hundreds of megabytes. The benefit is that proof checking is a deterministic, single-pass type-check of the stored term; the cost is storage and memory.

Isabelle uses a different approach: by default, Isabelle does not store proof terms. The `thm` value carries only the statement and the hypothesis set; the derivation that produced it is not recorded. This means that Isabelle's `.thy` files are much smaller than comparable Lean `.olean` or Coq `.vo` files — but checking a `.thy` file requires re-running all the tactics, which may be slow if they involve Sledgehammer calls. Isabelle does have an optional **Proofterm** mechanism that records explicit proof terms in its metalogic, but it is rarely used in production because it incurs significant overhead.

The practical consequence for compiler verification projects:
- CompCert's Coq proof (~100,000 lines) produces large `.vo` files but can be re-checked from scratch in a few hours on a modern server.
- The l4v seL4 proof (~200,000 lines of Isabelle) takes 40–60 hours to re-check from scratch on a modern machine because Sledgehammer calls are not cached; the team uses a CI build that only re-checks modified theories.
- Lean 4's Mathlib4 (~1,000,000 lines) uses a content-addressable `.olean` cache aggressively; a typical developer's change requires checking only the transitively-affected modules.

The trend is toward **proof certificate storage and independent checking**: Lean4Lean (§184.1.10) demonstrates that Lean 4's proof terms can be re-checked by an independently-written kernel; a similar "Coq checker" (`coqchk`) exists for Coq and re-checks `.vo` files without the elaborator.

### 184.4.6 Connection to Alive2 (Chapter 170) and LLM Provers (Chapter 181)

Alive2 ([Chapter 170](../part-24-verified-compilation/ch170-alive2-and-translation-validation.md)) uses Z3 as an SMT oracle to verify LLVM peephole rewrites. Its verification conditions are encoded directly in first-order logic with bitvector arithmetic — not in any of the three proof assistants discussed here. The relationship is complementary: Alive2's Z3-based checks are push-button (no proof terms, no kernel) but scope-limited (only quantifier-free bitvector formulas about peephole rewrites); Lean 4, Coq, and Isabelle handle unbounded induction, semantic definitions, and program-logic reasoning. Bringing Alive2's peephole-rewrite results into Lean 4 or Coq would require formalizing LLVM IR semantics (as Vellvm does) and proving each rewrite rule as an Isabelle/HOL or Coq theorem — a project that would significantly strengthen the formal basis of LLVM's correctness argument.

LLM-based proof search ([Chapter 181](../part-26-ecosystem-frontiers/ch181-formal-verification-in-practice.md)) currently operates primarily in Lean 4 via LeanDojo (§184.1.11). The LLM's role is to propose tactic steps; the kernel's role is to accept or reject them. No LLM output ever enters the trusted base. The practical effect is dramatic: LeanDojo's ReProver model (and its 2025–2026 successors) can close roughly 50–65% of Mathlib4 proof goals that cannot be closed by `aesop` or `omega` alone. The remaining goals require either more powerful automation (e.g., `native_decide` on large finite computations) or human mathematical insight.

For Coq, the `coq-hammer` tactic provides Sledgehammer-like functionality via the `hammer` tactic, calling ATPs and reconstructing proofs through Coq's `crush` / `cvc4` / `z3` tactics. It is less integrated than Isabelle's Sledgehammer and requires more manual guidance, but the underlying LCF principle is the same: external ATP outputs are not trusted; only the reconstruction is in the trusted path.

---

## 184.5 Chapter Summary

- **Lean 4's kernel** (`Lean.Kernel.check`, ~6,000 lines C++) is the sole trusted base. It implements CIC with universe polymorphism, proof irrelevance, and quotient types. It checks β, δ, ι, ζ, η definitional equality but does not perform unification — that is the elaborator's responsibility.
- **`Lean.Expr`** uses twelve constructors (`bvar`, `fvar`, `mvar`, `sort`, `const`, `app`, `lam`, `forallE`, `letE`, `lit`, `mdata`, `proj`) with de Bruijn indices for bound variables (`bvar`). Every node caches a hash and occurrence flags for efficient checking.
- **`Level`** uses six constructors (`zero`, `succ`, `max`, `imax`, `param`, `mvar`). The `imax` constructor implements `Prop`'s impredicativity: `imax u 0 = 0` (codomain in `Prop` forces the `∀` into `Prop`).
- The **elaboration pipeline** runs: source → `Lean.Syntax` → macro expansion → `TermElabM` → typeclass synthesis + `isDefEq` unification → `Expr` → `Kernel.check`. The pipeline is bidirectional; unification and reduction (`whnf`) live in `Lean.Meta.M`, entirely outside the kernel.
- **LCNF** is Lean 4's ANF-based compiler IR targeting C (stable) and LLVM IR (experimental, PR #1837). The LCNF-to-LLVM path requires LTO to inline `lean_object*` runtime intrinsics effectively.
- **Lean4Lean** (arXiv 2403.14064, Carneiro) reimplements and proves sound the Lean 4 typechecking algorithm within Lean 4 itself — the strongest available embodiment of the de Bruijn criterion for Lean.
- **Coq/Rocq's kernel** (~10,000 lines OCaml in `kernel/typeops.ml`, `kernel/reduction.ml`, `kernel/inductive.ml`) implements pCIC with `Prop`/`Set`/`Type`/`SProp`. The guard checker for fixpoints (structural recursion) is part of the kernel's trusted base; unsound fixpoints could produce looping type-checks or proofs of `False`.
- **Coq's extraction** (OCaml, Haskell, Scheme) is the bridge to executable code. CompCert and Vellvm use extraction to produce verified binaries. Extraction is not in the kernel but is in the trusted base of any extracted program; a verified extraction plugin is under development.
- **`native_compute`** and Lean's **`native_decide`** compile terms to native code for fast reduction, producing kernel-checked equality proofs. They are in the performance layer, not the trusted kernel.
- **Isabelle/HOL's kernel** (~6,000 lines SML) enforces the **LCF invariant**: `thm` is an abstract ML type; the only route to theorem values is through primitive rules in `src/Pure/thm.ML`. Tactics, Sledgehammer, ATP certificates, and code generation all ultimately terminate at kernel primitive calls — the architecture makes it structurally impossible for external tools to introduce unsound theorems.
- **Isar** provides structured proofs that read as mathematical arguments, with each intermediate step kernel-checked before proceeding. **Sledgehammer** calls Z3, CVC5, Vampire, E, and SPASS in parallel and reconstructs proofs through `metis`/`smt` → kernel primitives; external ATPs are untrusted.
- **seL4 chose Isabelle** for classical logic matching C reasoning patterns, `Simpl` C embedding, and Sledgehammer automation. The l4v proof (~200,000 lines) covers functional correctness, integrity, and confidentiality but excludes compiler correctness — a gap CompCert could close.
- The **de Bruijn criterion** — a trusted kernel small enough to audit independently — is satisfied by all three systems. Kernels are 6,000–10,000 lines; everything above (tactics, automation, extraction, code generation) is untrusted performance infrastructure.
- **Extracted and compiled code** enters the LLVM pipeline and inherits LLVM's correctness assumptions. Alive2's misoptimization detection ([Chapter 170](../part-24-verified-compilation/ch170-alive2-and-translation-validation.md)) directly strengthens the practical validity of formally verified programs compiled by LLVM-based toolchains.

---

*@copyright jreuben11*
