# Chapter 167 — Operational Semantics and Program Logics

*Part XXIV — Verified Compilation*

Compiler verification demands a precise mathematical account of what programs mean and what it means for a compiler to be correct. Informal English prose — however carefully written — cannot support machine-checked proofs; a single ambiguous sentence in a language reference can conceal a miscompilation that persists undetected for years. This chapter builds the formal foundations that the rest of Part XXIV relies on: operational semantics as a method for giving programs unambiguous meanings, Hoare logic and separation logic as specification languages for imperative code, concurrent extensions that scale to shared-memory parallelism, bisimulation as the bridge between source and target execution, and SMT solving as the computational engine that makes automated verification tractable. Together these form the theoretical core of CompCert, Vellvm, Alive2, and every other serious effort to verify that an optimizing compiler does what its specification promises.

---

## 167.1 Operational Semantics

Operational semantics defines the meaning of a program by specifying, precisely, how it executes. Unlike denotational semantics (which maps programs to mathematical objects) or axiomatic semantics (which uses logical assertions), operational semantics describes execution as a transition relation on configurations — pairs of a program fragment and a machine state. The resulting definition is executable, close to implementation intuitions, and well-suited to inductive proof techniques.

### Small-Step (Structural Operational Semantics)

Small-step semantics, introduced by Gordon Plotkin in his 1981 Aarhus lecture notes and later systematized as Structural Operational Semantics (SOS), defines the elementary reduction steps a program takes. A configuration is a pair `⟨e, σ⟩` where `e` is an expression (or statement, instruction — depending on the language) and `σ` is a store mapping variables to values. The reduction relation `→` is defined by a set of inference rules of the form:

```
        premise₁  premise₂  ...
    ─────────────────────────────  [Rule-Name]
         ⟨e, σ⟩ → ⟨e', σ'⟩
```

Consider a simple arithmetic language with variables, constants, addition, and assignment. Representative rules include:

```
  σ(x) = v
─────────────  [Var]            ──────────────────────  [Const]
 ⟨x, σ⟩ → ⟨v, σ⟩              ⟨n, σ⟩ → ⟨n, σ⟩

  ⟨e₁, σ⟩ → ⟨e₁', σ'⟩
──────────────────────────────  [Add-L]
 ⟨e₁ + e₂, σ⟩ → ⟨e₁' + e₂, σ'⟩

  n₁ + n₂ = n
───────────────────────────────  [Add]
 ⟨n₁ + n₂, σ⟩ → ⟨n, σ⟩
```

The multi-step relation `→*` is the reflexive-transitive closure of `→`. A configuration `⟨v, σ⟩` where `v` is a value is terminal. Small-step semantics captures the fine-grained structure of execution: evaluation order, interleaving of effects, intermediate states. This granularity is essential for compiler verification, where every transformation must preserve behavior at the level of individual steps.

The SOS framework naturally handles non-termination: a program that diverges simply has an infinite reduction sequence `⟨e₀, σ₀⟩ → ⟨e₁, σ₁⟩ → ⟨e₂, σ₂⟩ → ...` with no terminal configuration reachable. Stuck configurations — configurations that are not terminal but have no applicable rule — represent runtime errors (type errors, null dereference, etc.) that a well-typed program should never reach.

### Big-Step (Natural) Semantics

Big-step semantics, due to Gilles Kahn (1987), collapses entire evaluation sequences into a single judgment `⟨e, σ⟩ ⇓ ⟨v, σ'⟩`, read "expression e in store σ evaluates to value v producing final store σ'". The rules directly relate inputs to outputs:

```
                               ⟨e₁, σ⟩ ⇓ ⟨n₁, σ'⟩    ⟨e₂, σ'⟩ ⇓ ⟨n₂, σ''⟩    n₁ + n₂ = n
─────────────────  [B-Var]    ────────────────────────────────────────────────────────────────  [B-Add]
 ⟨x, σ⟩ ⇓ ⟨σ(x), σ⟩                           ⟨e₁ + e₂, σ⟩ ⇓ ⟨n, σ''⟩
```

Big-step semantics is concise and directly corresponds to a recursive interpreter. Its weakness: divergent programs have no big-step derivation (they simply fail to evaluate), making it harder to distinguish non-termination from runtime errors. For compiler correctness, this is a real problem — a compiler that miscompiles a diverging program (making it terminate with an error, or with a result) may be undetectable in big-step semantics, because both source and target would have no derivation. Xavier Leroy's work on CompCert carefully addresses this by using a combined semantics for the correctness theorem.

### Reduction Strategies

The same expression can reduce in different orders depending on evaluation strategy. Three canonical strategies:

**Call-by-value** (CBV, strict evaluation): arguments are evaluated to values before function application. The rule for application `(λx.e) v` reduces only when the argument is already a value. This is the strategy used by C, Java, ML, and most mainstream languages. LLVM IR uses call-by-value semantics for function calls.

**Call-by-name** (CBN): arguments are substituted unevaluated; they are re-evaluated at each use. This is operationally defined by the rule `(λx.e) e_arg → e[e_arg/x]` without requiring `e_arg` to be a value first. Call-by-name is not commonly used in implementations (it duplicates work) but is important in denotational semantics and as a theoretical reference point.

**Call-by-need** (lazy evaluation): like call-by-name but each argument is evaluated at most once; the result is memoized and shared. Haskell uses call-by-need. The operational semantics requires heaps (sharing graphs) to represent this faithfully.

For compiler verification, the relevant evaluation strategy is that of the source language. A Haskell-to-LLVM compiler must map lazy evaluation to a graph reduction machine; a C-to-LLVM compiler uses strict evaluation and the LLVM calling convention directly.

### Progress and Preservation (Type Safety)

The canonical result connecting type systems and operational semantics is the type safety theorem, usually split into two lemmas:

**Progress**: A well-typed closed expression is either a value or can take a step.
```
  ∀ e, τ.  ⊢ e : τ  →  val(e) ∨ (∃ e'. e → e')
```

**Preservation** (Subject Reduction): If a well-typed expression takes a step, the result is also well-typed with the same type.
```
  ⊢ e : τ  ∧  e → e'  →  ⊢ e' : τ
```

Together, Progress and Preservation establish that a well-typed program never reaches a stuck configuration (neither a value nor reducible). The classic proof method uses induction on typing derivations for Preservation and structural induction on typing derivations for Progress. Wright and Felleisen (1994) formalized this approach as "syntactic type soundness."

In the context of LLVM, type safety is weaker: LLVM IR is explicitly typed but permits undefined behavior. The type system catches certain malformed programs (applying an integer to a function type, etc.) but does not prevent all runtime errors — UB is permitted. Vellvm (Chapter 169) formalizes exactly which operations are defined and which are UB.

### Labeled Transition Systems

A Labeled Transition System (LTS) `(S, A, →)` consists of a set of states S, a set of labels A (representing observable actions), and a labeled transition relation `→ ⊆ S × A × S`. An unlabeled (or τ-labeled) transition represents internal computation. LTS provides an abstract model for concurrent and communicating systems.

Key concepts:
- **Trace**: a sequence of observable labels produced by a sequence of transitions; two systems are trace-equivalent if they produce the same set of traces.
- **Bisimulation**: a stronger equivalence (defined in §167.5) that equates systems not only on their traces but on their interactive behavior.
- **CCS** (Calculus of Communicating Systems, Milner 1980): a process algebra built on LTS; the foundation for concurrent program semantics.

In compiler verification, LTS is used to model concurrent programs: source language semantics defines an LTS, target semantics defines another, and correctness requires a simulation relation between them. The labels represent observable events (I/O, synchronization primitives, final values) that must be preserved.

### Contextual Equivalence

Two expressions are contextually equivalent if no context can distinguish them:

```
  e₁ ≅ e₂  iff  ∀ C[·]. C[e₁] ↓  ⟺  C[e₂] ↓
```

where `C[·]` is a closing context (a term with a hole that, when filled, gives a closed program that can be run), and `↓` means "terminates with a value." Contextual equivalence is the finest observational equivalence: two programs are contextually equivalent if and only if no experiment can distinguish them.

Contextual equivalence is the gold standard for compiler correctness: a compiler is correct if it maps contextually equivalent programs to contextually equivalent machine code. However, proving contextual equivalence directly requires quantifying over all contexts, which is undecidable in general. Logical relations and bisimulations provide tractable proof methods that imply contextual equivalence.

For LLVM's purposes, contextual equivalence must account for UB: if a source program invokes UB, the compiler may produce any behavior. The refinement ordering used by Alive2 (`tgt ⊑ src`, meaning the target's behaviors are a subset of the source's behaviors) is a weakened form of contextual equivalence that accounts for UB monotonically.

---

## 167.2 Hoare Logic

Hoare logic, introduced by C.A.R. Hoare in his 1969 paper "An axiomatic basis for computer programming," provides a formal system for reasoning about the correctness of imperative programs using logical assertions about program states.

### Hoare Triples and Partial Correctness

A Hoare triple `{P} C {Q}` asserts partial correctness: if the precondition P holds before executing command C, and if C terminates, then the postcondition Q holds after. "Partial" because no claim is made about whether C terminates — only about the behavior if it does.

Formally, in a state model where states are functions from variables to values:
```
  |= {P} C {Q}  iff  ∀ σ, σ'. P(σ) ∧ ⟨C, σ⟩ →* ⟨skip, σ'⟩ → Q(σ')
```

The assertion language P, Q ranges over first-order formulas over program variables. Standard axioms and rules:

**Assignment Axiom** (Hoare's classic "backwards" form):
```
  ───────────────────────  [Assign]
  {P[e/x]} x := e {P}
```
The precondition is the postcondition with all free occurrences of x replaced by e. This runs "backwards" — you start from what you want to be true after the assignment and derive what must be true before.

**Sequence Rule**:
```
  {P} C₁ {R}    {R} C₂ {Q}
  ────────────────────────  [Seq]
       {P} C₁; C₂ {Q}
```

**Conditional Rule**:
```
  {P ∧ b} C₁ {Q}    {P ∧ ¬b} C₂ {Q}
  ────────────────────────────────────  [If]
  {P} if b then C₁ else C₂ {Q}
```

**While Rule**:
```
  {P ∧ b} C {P}
  ─────────────────────────  [While]
  {P} while b do C {P ∧ ¬b}
```
where P is a loop invariant — a property that holds before and after each iteration.

**Consequence Rule**:
```
  P' → P    {P} C {Q}    Q → Q'
  ──────────────────────────────  [Cons]
          {P'} C {Q'}
```
Preconditions can be strengthened; postconditions can be weakened.

### Total Correctness

Total correctness `[P] C [Q]` adds a termination requirement: if P holds initially, C terminates and Q holds. Proving total correctness requires a **loop variant** (also called a termination measure): a natural-number-valued expression V that strictly decreases with each loop iteration (well-foundedness ensures the decrease cannot continue forever):

```
  {P ∧ b ∧ V = n} C {P ∧ V < n}    n is fresh
  ────────────────────────────────────────────────  [While-Total]
  [P ∧ (P ∧ ¬b → Q)] while b do C [Q]
```

For compiler verification, partial correctness is often sufficient: if the source program terminates (which is given by assumption), does the compiled program produce the same result? Total correctness becomes relevant when verifying that compiler transformations preserve or introduce non-termination — for example, that a loop optimization does not convert a non-terminating loop into one that terminates with an incorrect result.

### Weakest Preconditions

The weakest precondition transformer `wp(C, Q)` computes the weakest assertion P such that `{P} C {Q}`. "Weakest" means it is implied by every valid precondition for C and Q. Dijkstra (1976) defined wp as a predicate transformer semantics:

```
  wp(skip, Q)         = Q
  wp(x := e, Q)       = Q[e/x]
  wp(C₁; C₂, Q)      = wp(C₁, wp(C₂, Q))
  wp(if b then C₁ else C₂, Q) = (b → wp(C₁, Q)) ∧ (¬b → wp(C₂, Q))
  wp(while b do C, Q) = (requires loop invariant; not directly computable)
```

For straight-line code (no loops, no procedure calls), wp is effectively computable and produces a logical formula that can be checked by an SMT solver. This is the basis for verification condition generation (VCG) tools used in practice. The non-computability of wp for loops is addressed by requiring the user to supply loop invariants.

### The Frame Rule

The frame rule, one of the most important results in program verification, states:

```
  {P} C {Q}
  ────────────────────  [Frame]   (provided mod(C) ∩ fv(R) = ∅)
  {P * R} C {Q * R}
```

where `*` denotes separating conjunction (defined in §167.3) and `mod(C)` is the set of variables modified by C. Informally: if C is correct with respect to P and Q, and C does not touch the region R, then C is also correct when R is present alongside P and Q.

The frame rule is what makes compositional reasoning possible: you can verify a procedure P against a local specification without mentioning the rest of the heap. When the frame rule is applied to the full program, local proofs compose into a global proof. In standard Hoare logic, the frame rule is sound only when the "R does not mention variables modified by C" condition is enforced. Separation logic (§167.3) provides a cleaner, heap-aware version.

---

## 167.3 Separation Logic

Standard Hoare logic struggles with heap-manipulating programs. The frame rule becomes unsound in the presence of aliasing: if two pointers p and q alias the same memory cell, a command that modifies the cell through p also modifies q — but the frame rule naively assumes the modification is local to p. Separation logic, developed by John Reynolds and Peter O'Hearn (2002), addresses this by making heap ownership explicit in the logic.

### Points-To and Separating Conjunction

The points-to assertion `x ↦ v` asserts that x is a valid heap address containing value v, and that this location is exclusively owned by the current assertion. Exclusive ownership is the key: no other part of the program (no other thread, no other pointer) has access to x's location.

The separating conjunction `P * Q` (pronounced "P and Q separately") asserts that P and Q hold on disjoint portions of the heap:

```
  (h₁ * h₂) ⊨ P * Q  iff  ∃ h₁, h₂. h = h₁ ⊎ h₂  ∧  h₁ ⊨ P  ∧  h₂ ⊨ Q
```

where `⊎` denotes disjoint union of heaps. The empty heap `emp` is the unit of `*`: `P * emp = P`.

Examples:
- `x ↦ 3 * y ↦ 5`: x points to 3, y points to 5, and the locations are distinct.
- `(x ↦ a) * (y ↦ b) * (z ↦ c)`: three distinct heap cells.
- `x ↦ _ * y ↦ _`: two distinct cells; contents irrelevant.

The standard Hoare logic assertion `P ∧ Q` holds when both P and Q are true on the same heap, possibly with aliasing. Separating conjunction `P * Q` additionally guarantees no aliasing between the heap regions described by P and Q.

### Magic Wand

The magic wand (or "separating implication") `P -* Q` holds for a heap h if, for every disjoint heap h' satisfying P, the combined heap `h ⊎ h'` satisfies Q. Formally:

```
  h ⊨ P -* Q  iff  ∀ h'. h ⊥ h' ∧ h' ⊨ P → (h ⊎ h') ⊨ Q
```

The magic wand is used to express "consume a resource and produce another." For example, `(x ↦ _) -* P` means "if you give me an exclusive pointer to x, I can produce P." This is useful in ownership-passing protocols and in formalizing procedure specifications that consume or produce heap resources.

### Inductive Data Structures

Separation logic naturally handles recursive data structures via inductive definitions. A singly-linked list starting at pointer x with values n₁, ..., nₖ:

```
  list(null, [])     ≜  emp
  list(x, v::vs)    ≜  ∃ y. x ↦ (v, y) * list(y, vs)
```

The separating conjunction ensures that all cells in the list are distinct (no cycles in a valid list). A binary tree rooted at x:

```
  tree(null)         ≜  emp
  tree(x)           ≜  ∃ v, l, r. x ↦ (v, l, r) * tree(l) * tree(r)
```

These inductive predicates form the basis for verifying algorithms that traverse and modify recursive heap structures. Proof rules for folding and unfolding inductive definitions allow reasoning about loops that process list elements: on each iteration, one step of the list predicate is unfolded.

### Soundness of the Frame Rule in Separation Logic

The frame rule in separation logic is:

```
  {P} C {Q}
  ─────────────  [Frame-SL]   (provided fv(R) ∩ mod(C) = ∅)
  {P * R} C {Q * R}
```

This is sound even for heap-manipulating programs that write through pointers, provided C does not allocate or free memory in the R region. The soundness proof uses the disjoint heap interpretation: if C operates on a heap satisfying P (with R on a disjoint sub-heap), C's steps can only affect P's portion; the R portion is untouched; hence Q * R holds after C.

The key insight: separation logic shifts the burden of proving non-interference from the client (who would need to argue about the absence of aliasing) to the resource ownership structure (enforced by the type of separating conjunction). This is why separation logic scales to large programs in ways classical Hoare logic cannot.

### Bi-Abduction and Automated Verification

Bi-abduction (Calcagno et al., 2011) is an inference procedure for separation logic that enables automatic verification without user-supplied invariants. Given a pre-state P and a procedure call, bi-abduction simultaneously infers:
- The **frame** R: the part of the heap not touched by the procedure.
- The **missing resource** A: heap cells that must be allocated for the call to be valid.

Formally, given P and postcondition Q, find A and R such that `P * A ⊢ Q * R`. This underpins Facebook's Infer tool, which scales separation logic verification to millions of lines of code.

In compiler verification, bi-abduction is less directly relevant — the focus is on full functional correctness rather than resource-safety — but the compositional proof structure it enables (local proofs compose) directly informs how CompCert and Vellvm organize their correctness arguments.

---

## 167.4 Concurrent Separation Logic

Concurrency adds sharing and interference: two threads may access the same memory cell simultaneously, leading to data races and subtle interleaving-dependent behaviors. Standard separation logic assumes sequential execution. Concurrent Separation Logic (CSL), introduced by O'Hearn (2007) building on work by Reynolds and Brookes, extends separation logic to concurrent programs while preserving compositional reasoning.

### The Parallel Composition Rule

The core rule of CSL for parallel composition:

```
  {P₁} C₁ {Q₁}    {P₂} C₂ {Q₂}
  ─────────────────────────────────  [Par]   (provided fv(P₁, Q₁) ∩ mod(C₂) = ∅ and vice versa)
  {P₁ * P₂} C₁ ∥ C₂ {Q₁ * Q₂}
```

If C₁ operates on a disjoint portion of the heap from C₂, they can run in parallel without interference. The separating conjunction in the pre- and post-condition enforces ownership disjointness. This is the formal reason why data-race-free programs are safe: each thread owns its resources exclusively.

### Ownership Transfer

Threads need to communicate — a producer writes data, then passes it to a consumer. CSL models this via **resource invariants**: shared heap regions protected by locks carry an invariant. When a thread acquires a lock, it gains access to the invariant's heap; when it releases the lock, it must restore the invariant.

```
  {lock(l, I) * P} acquire(l) {lock(l, I) * I * P}
  {lock(l, I) * I * P} release(l) {lock(l, I) * P}
```

The resource invariant I specifies the heap owned by the lock. A thread that acquires the lock gains ownership of I's heap, does something useful with it, then releases the lock (restoring I). This is how ownership passes between threads: through lock acquisition and release, not through aliased pointers.

### Fractional Permissions

For read-sharing (multiple readers, single writer), CSL introduces fractional permissions. A permission of 1 (or "full") grants exclusive write access; a permission of π (0 < π < 1) grants read-only access. Two fractional permissions can be combined: `π₁ + π₂ = 1` means the two fragments together constitute full ownership.

```
  x ↦^π v   (x points to v with permission π)
  
  x ↦^1 v  ⟺  x ↦ v      (full permission = standard points-to)
  x ↦^(1/2) v * x ↦^(1/2) v  ⟹  x ↦^1 v  (two halves compose)
```

Fractional permissions model reader-writer locks precisely: the writer holds the full permission; readers each hold a fractional permission. This prevents the writer from operating while any reader is active. The formalism, due to Boyland (2003), is now standard in higher-order concurrent logics.

### Iris: A Modern Concurrent Separation Logic

Iris (Krebbers et al., 2017; Jung et al., 2018) is a higher-order concurrent separation logic implemented and machine-checked in Coq. Its innovations over classical CSL:

- **Step-indexing**: handles semantic circularities (e.g., self-referential invariants) by indexing the logic with a natural number representing the number of future steps.
- **Ghost state**: allows threads to maintain logical state (not physical heap) that tracks the progress of concurrent protocols.
- **Invariant mechanism**: `⊤ ▷ I` — the invariant I holds at all future steps; any thread may open I for one step to read or write its contents.
- **Ownership of invariants**: invariants are resources that can be passed between threads; the logic is fully compositional.
- **Higher-order**: propositions can quantify over propositions; programs can manipulate their own specifications.

Iris is the logic underlying RustBelt (see below) and several verified concurrent data structure libraries. It is implemented as a Coq library (`coq-iris`) and provides tactics for reasoning about programs written in a toy concurrent language (HeapLang).

### RustBelt: Formal Semantics of Rust Ownership

RustBelt (Jung et al., POPL 2018) gives the first mechanized formal semantics to Rust's ownership and borrowing type system, using Iris as the underlying logic. The central theorem:

> If a Rust program type-checks (under the standard type system), it is safe — no data races, no use-after-free, no dangling pointers.

RustBelt works not by verifying the Rust type checker (which is large and complex) but by showing that every typing rule is semantically sound: each rule corresponds to a valid proof step in Iris. The semantic model assigns to each type a "type interpretation" — an Iris proposition describing the ownership protocol for values of that type.

The significance for compiler verification: the Rust compiler (rustc) uses LLVM as its backend. A formally verified Rust type system (RustBelt), combined with a formally verified LLVM backend (in the spirit of Vellvm), would give an end-to-end guarantee from Rust types to machine code. This is an active research direction but has not yet been achieved in full generality.

---

## 167.5 Bisimulation and Simulation

Simulation relations are the standard tool for proving that two transition systems — source language semantics and target language semantics — produce equivalent behaviors. They provide a "coinductive" proof principle: rather than reasoning about all finite execution prefixes, a simulation relation is established at the beginning and preserved throughout execution.

### Strong Bisimulation

A binary relation `R ⊆ S × S` on states of an LTS is a **strong bisimulation** if, for all `(s₁, s₂) ∈ R`:
1. For every `s₁ →^a s₁'`, there exists `s₂ →^a s₂'` such that `(s₁', s₂') ∈ R`.
2. For every `s₂ →^a s₂'`, there exists `s₁ →^a s₁'` such that `(s₁', s₂') ∈ R`.

Two states are **bisimilar** (`s₁ ~ s₂`) if there exists a bisimulation relation containing `(s₁, s₂)`. Bisimilarity is the coarsest bisimulation — it is itself a bisimulation. Milner showed that bisimilarity and observational congruence coincide for CCS, making bisimulation the right notion of behavioral equivalence for concurrent systems.

Bisimulation is strictly finer than trace equivalence: two systems may produce identical traces while differing in their interactive (branching) behavior. For compiler verification of sequential programs, trace equivalence (matching observable outputs) is often sufficient — but for concurrent programs, bisimulation (or weaker forms of it) is needed.

### Weak Bisimulation

In many settings, internal computation steps (labeled τ for "silent") should be abstracted over. **Weak bisimulation** allows matching a single step with zero or more τ-steps followed by a visible step:

```
  s₁ →^τ* →^a →^τ* s₁'   matched by   s₂ →^τ* →^a →^τ* s₂'
```

Weak bisimulation is the standard equivalence for concurrent systems where internal computation order does not matter. For compiler verification, τ-steps represent compiler-introduced reductions (e.g., inlining an administrative reduction); weak bisimulation allows the target to take more internal steps than the source.

### Forward Simulation

In compiler verification, the typical setup is that the source language L₁ and target language L₂ have different state spaces; we need a relation between them, not a bisimulation on a shared state space.

A **forward simulation** from L₁ to L₂ is a relation `R ⊆ States₁ × States₂` such that:

1. **Initial states**: if `s₁ ∈ Init₁`, then there exists `s₂ ∈ Init₂` with `R(s₁, s₂)`.
2. **Step simulation**: if `R(s₁, s₂)` and `s₁ →₁ s₁'`, then there exists `s₂'` with `s₂ →₂* s₂'` and `R(s₁', s₂')`.
3. **Final state simulation**: if `R(s₁, s₂)` and `s₁ ∈ Final₁`, then `s₂ ∈ Final₂` with matching final values.

A forward simulation guarantees: every behavior of the source is a behavior of the target (in the sense of observable traces). This is the direction needed for compiler correctness: a source-program behavior (the source specification) is preserved by compilation.

CompCert (Chapter 168) is built entirely from forward simulations. Each compilation pass `L_i → L_{i+1}` has a Coq-verified forward simulation proof; the passes compose by transitivity of simulations, giving a global forward simulation from Clight to assembly.

### Backward Simulation

A **backward simulation** is the converse: for every target step, there exists a corresponding source step. Backward simulations guarantee that no target behavior is "invented" — every target behavior was already present in the source.

When a compilation pass is deterministic in the target (each target configuration has at most one successor), forward simulation suffices. When the target is more nondeterministic than the source (e.g., a register allocator may choose among several valid colorings), backward simulation may be needed. In practice, CompCert avoids this by making each pass's output deterministic.

A combination — showing both a forward and backward simulation — yields a bisimulation, guaranteeing full behavioral equivalence. This is rarely achievable for full compilers (due to irreversible transformations like inlining) but can be shown for specific passes.

### Compiler Correctness as Simulation

The standard formulation of compiler correctness:

> The compiler C is correct if for every source program P, if the compiled program `C(P)` is defined, then `exec(C(P)) ⊑ exec(P)` — the behaviors of the compiled program are a subset of the behaviors of the source program.

Here, "behaviors" include termination with a value, divergence, and undefined behavior. The refinement `⊑` direction: the target may be more deterministic (UB is resolved by the compiler's choice) but must not introduce new observable behaviors that the source did not permit.

This is formalized as a forward simulation: source states are related to target states by the simulation invariant; each source step is matched by one or more target steps maintaining the invariant. The global correctness theorem follows from the simulation and the definition of behavioral equivalence.

---

## 167.6 SMT Solvers in Compiler Verification

Automated theorem proving reduces the burden of interactive proof, enabling verification tools that can check thousands of transformation instances without human guidance. Satisfiability Modulo Theories (SMT) solvers are the engine behind tools like Alive2 and many translation validators.

### SAT and SMT

**SAT** (Boolean Satisfiability) asks: given a propositional formula, does there exist an assignment of truth values to variables that makes the formula true? SAT is NP-complete, but modern DPLL(T)-based solvers (MiniSat, CaDiCaL, Kissat) routinely handle formulas with millions of variables in practice.

**SMT** (Satisfiability Modulo Theories) extends SAT with background theories: instead of bare propositional variables, SMT formulas may contain integer arithmetic, bitvectors, arrays, uninterpreted functions, etc. The DPLL(T) architecture integrates a SAT solver with theory solvers:
- SAT solver decides propositional structure.
- Theory solver checks consistency of the theory-level literals.
- Conflicts propagated between layers via theory lemmas.

### Core Theories

Key theories for compiler verification:

**QF_BV** (Quantifier-Free Bitvectors): fixed-width integers with machine arithmetic. Operations: `bvadd`, `bvmul`, `bvsub`, `bvand`, `bvor`, `bvxor`, `bvshl`, `bvlshr`, `bvashr`, `bvult`, `bvslt`, etc. LLVM IR integer operations are directly modeled in QF_BV — `add i32 %a, %b` becomes `bvadd(a, b)` where a and b are 32-bit bitvector variables. This is the theory Alive2 uses most heavily (Chapter 170).

**QF_LIA** (Quantifier-Free Linear Integer Arithmetic): linear arithmetic over integers. Used for loop bounds reasoning, polyhedral analysis (Chapter 70–73). Handled by Presburger arithmetic decision procedures (Omega test, Barvinok algorithm).

**QF_AX** (Arrays with Extensionality): arrays as total functions from indices to values, with select (read) and store (write) operations. The McCarthy array axioms: `select(store(a, i, v), j) = if i = j then v else select(a, j)`. Used to model memory arrays in SMT-based verification.

**UF** (Uninterpreted Functions): constants and function symbols with no fixed interpretation; useful for abstracting away details (e.g., modeling an unanalyzed function call as an uninterpreted symbol with constraints on its return value).

### Primary SMT Solvers

**Z3** (de Moura and Bjørner, 2008; Microsoft Research): the most widely used SMT solver in program verification. Supports all major theories; has a rich C/C++ API; is the solver used by Alive2. Z3's bitvector solver (based on bit-blasting and incremental SAT) is particularly strong.

**CVC5** (Barrett et al.): the successor to CVC4; strong on quantified formulas and strings; used in Isabelle/HOL and some CompCert-adjacent tools.

**Bitwuzla** (Preiner et al.): specialized in QF_BV and QF_FP (floating-point); often faster than Z3 on pure bitvector problems. Increasingly used for LLVM IR verification.

### LLVM IR Bitvector Semantics

LLVM IR integer operations have precise bitvector semantics when UB is excluded:
- `add i32 %a, %b` → `bvadd₃₂(a, b)` (wraps on overflow unless `nsw`/`nuw` flags set)
- `add nsw i32 %a, %b` → result is `bvadd₃₂(a, b)` if no signed overflow; else poison
- `icmp slt i32 %a, %b` → `bvslt₃₂(a, b)`
- `shl i32 %a, %b` → `bvshl₃₂(a, b)` (shift amount 0 ≤ b < 32, else poison)
- `udiv i32 %a, %b` → `bvudiv₃₂(a, b)` (b ≠ 0, else poison)

This clean mapping makes SMT-based verification of LLVM IR transformations practical: an optimization peephole `add i32 %x, 0 → %x` is verified by checking that `bvadd₃₂(x, 0) = x` for all 32-bit bitvectors x — a trivially valid formula discharged instantly by any SMT solver.

Floating-point operations use the QF_FP theory (IEEE 754), which is more expensive but handles rounding modes, NaN, and infinity correctly.

### Bounded Model Checking

**Bounded Model Checking** (BMC) applies SMT to verify properties of programs with loops by unrolling the loop a fixed number of times K and checking whether any error occurs within K iterations. Given a program P and a property φ:

1. Generate the unrolled program `P^K` (K copies of the loop body).
2. Encode `P^K ∧ ¬φ` as an SMT formula.
3. If SAT: found a counterexample with ≤ K iterations.
4. If UNSAT: φ holds for all executions with ≤ K iterations.

BMC is incomplete (does not prove φ holds for all iterations), but it is highly effective for finding bugs. KLEE (Chapter 114-adjacent) uses symbolic execution — a related technique — to explore all paths up to a given depth.

For compiler verification, BMC is used in tools like CBMC (C Bounded Model Checker) and in automated optimization validators that check that a specific transformation is correct for inputs bounded in bitwidth.

### Alive2 and Z3

Alive2 (Chapter 170) uses Z3 to check the refinement condition for LLVM IR transformations. For each transformation `src → tgt`:

1. **Encode** both src and tgt as Z3 formulas over bitvector variables (one variable per LLVM SSA value).
2. **Assert** the negation of the refinement: `∃ input. defined(src, input) ∧ (tgt(input) ≠ src(input) ∨ undefined(tgt, input))`.
3. **Query** Z3: if SAT, the counterexample is a concrete input that witnesses the incorrectness; if UNSAT, the transformation is valid.

The memory model (heap operations, pointer aliasing) is encoded using QF_AX and uninterpreted functions, with additional axioms capturing LLVM's memory semantics. The full encoding is several thousand lines of C++ code; its correctness is validated empirically against known-good and known-bad transformations.

### Decision Procedures and Undecidability

Key decidability results:
- **QF_BV** with fixed bitwidths: decidable (reduces to SAT by bit-blasting); `2^n`-hard in the worst case for n-bit integers.
- **QF_LIA**: decidable (Presburger arithmetic); non-elementary worst case, but practical.
- **QF_NIA** (nonlinear integer arithmetic): undecidable in general; partial decision procedures exist.
- **First-order arithmetic** (Peano arithmetic): undecidable (Gödel incompleteness).

For LLVM IR, the relevant fragment is primarily QF_BV (for integer operations) plus memory model extensions. The decidability of QF_BV means that Alive2's verification queries are always decidable in principle, though exponential in the bitwidth. In practice, 64-bit queries are handled by Z3's incremental SAT solver and internal simplifications, not by raw bit-blasting.

---

## Chapter Summary

- **Small-step semantics** defines execution as a transition relation `⟨e, σ⟩ → ⟨e', σ'⟩`; it captures evaluation order and intermediate states, making it the preferred basis for compiler correctness proofs.
- **Big-step semantics** relates inputs directly to outputs via `⟨e, σ⟩ ⇓ ⟨v, σ'⟩`; simpler for terminating programs but cannot distinguish divergence from runtime errors.
- **Progress and Preservation** together constitute syntactic type soundness: well-typed programs do not get stuck.
- **Hoare logic** provides a specification language for imperative programs via triples `{P} C {Q}`; the frame rule enables compositional reasoning.
- **Separation logic** extends Hoare logic with explicit heap ownership via separating conjunction `P * Q`; the frame rule becomes unconditionally sound for programs that respect ownership.
- **Concurrent Separation Logic** handles shared-memory parallelism; fractional permissions enable read-sharing; Iris is the state-of-the-art machine-checked CSL.
- **RustBelt** applies Iris to give a formal semantics to Rust's ownership types, proving the type system sound.
- **Forward simulations** are the standard proof obligation for each compilation pass; they compose transitively to give global compiler correctness.
- **SMT solvers** (Z3, CVC5, Bitwuzla) provide automated decision procedures for QF_BV and related theories; they are the computational engine of Alive2 and other translation validators.
- **Bounded model checking** uses SMT to find bugs in programs by checking finite unrollings of loops; it is incomplete but highly effective for bug-finding.

### References

- Plotkin, G. (1981). *A Structural Approach to Operational Semantics*. Aarhus University technical report DAIMI FN-19.
- Hoare, C.A.R. (1969). "An axiomatic basis for computer programming." *CACM* 12(10):576–580.
- Reynolds, J.C. (2002). "Separation logic: A logic for shared mutable data structures." *LICS 2002*.
- O'Hearn, P.W. (2007). "Resources, Concurrency, and Local Reasoning." *Theoretical Computer Science* 375(1–3):271–307.
- Jung, R. et al. (2018). "Iris from the ground up." *JFP* 28:e20. (Iris framework)
- Jung, R. et al. (2017). "RustBelt: Securing the Foundations of the Rust Programming Language." *POPL 2018*.
- de Moura, L. and Bjørner, N. (2008). "Z3: An Efficient SMT Solver." *TACAS 2008*.
- Leroy, X. (2009). "Formal verification of a realistic compiler." *CACM* 52(7):107–115.
- Lopes, N.P. et al. (2015). "Provably Correct Peephole Optimizations with Alive." *PLDI 2015*.
- Milner, R. (1980). *A Calculus of Communicating Systems*. Springer LNCS 92.


---

@copyright jreuben11
