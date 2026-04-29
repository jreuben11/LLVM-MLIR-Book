# Chapter 10 — Dataflow Analysis: The Lattice Framework

*Part II — Compiler Theory and Foundations*

Chapter 9 concluded with a program in Static Single Assignment form: every variable is defined exactly once, and φ-functions at join points gather the reaching definitions from all predecessor blocks. That structural invariant is more than an IR convenience — it is the concrete expression of a deeper mathematical pattern. Compilers routinely need to ask questions such as: *Which variables are live at this point? Which definitions can reach this use? Is this expression available without recomputation? What are the possible values of this register?* Each question asks for information that is a function of every execution path through the program, yet we cannot enumerate paths (there are exponentially many, and loops create infinitely many). The answer is the **lattice-theoretic framework for dataflow analysis**, whose mathematical roots reach back to Alfred Tarski's 1955 fixed-point theorem and whose application to compilers was unified by Gary Kildall in 1973 [Kildall 1973].

The central insight is this: the information we seek at each program point can be represented as an element of a carefully chosen mathematical structure — a **complete lattice** — and the process of computing that information is nothing more than finding the **least fixed point** of a system of monotone equations. Two theorems guarantee that the fixed point exists and can be found by a finite iterative computation: Tarski's theorem gives existence, and Kleene's ascending chain theorem gives computability. The whole framework is then instantiated to each classical analysis by choosing the right lattice and the right transfer functions.

This chapter develops the mathematics from first principles before deriving each classical analysis — liveness, available expressions, reaching definitions, very busy expressions, and constant propagation — as special cases. It then extends the framework to infinite-height lattices via widening and narrowing operators (Cousot–Cousot abstract interpretation [Cousot & Cousot 1977]), and finally treats interprocedural extensions: call strings, *k*-CFA, and summary-based analysis. Chapter 11 shows how classical optimisations — dead code elimination, common subexpression elimination, code motion — consume the results computed here. Chapter 61 surveys how LLVM's analysis infrastructure implements these ideas in production.

---

## 10.1 Mathematical Foundations: Partial Orders and Lattices

### 10.1.1 Partial Orders

A **partial order** is a pair (S, ≤) where S is a set and ≤ ⊆ S × S is a binary relation that satisfies three axioms for all a, b, c ∈ S [Dragon §9.2, Muchnick §7.1]:

1. **Reflexivity**: a ≤ a
2. **Antisymmetry**: a ≤ b ∧ b ≤ a ⟹ a = b
3. **Transitivity**: a ≤ b ∧ b ≤ c ⟹ a ≤ c

The intended reading of a ≤ b is "a is at least as specific (or as informative) as b" — but the exact meaning depends on the analysis. Two elements a, b ∈ S are **comparable** if a ≤ b or b ≤ a; otherwise they are **incomparable**, written a ∥ b.

A partial order becomes a **total order** if every pair is comparable (the integers under ≤ are a total order). Partial orders that interest us typically have incomparable elements — for instance, different sets of variable definitions are neither subsets nor supersets of each other in general.

A **Hasse diagram** is a graphical representation of a finite partial order: elements are drawn as nodes; an edge from node a to node b (b above a) means b covers a, i.e., a ≤ b with no c such that a < c < b. The diagram for {∅, {x}, {y}, {x,y}} ordered by ⊆:

```
     {x,y}
    /     \
  {x}     {y}
    \     /
       ∅
```

### 10.1.2 Lattices

A **lattice** (L, ≤) is a partial order in which every pair of elements a, b ∈ L has:

- A **least upper bound** (join, lub): a ⊔ b, the unique smallest element c with a ≤ c and b ≤ c.
- A **greatest lower bound** (meet, glb): a ⊓ b, the unique largest element c with c ≤ a and c ≤ b.

A lattice is **bounded** if it has a greatest element ⊤ (top) and a least element ⊥ (bottom), satisfying a ≤ ⊤ and ⊥ ≤ a for all a. In a bounded lattice, ⊤ = ⊔L and ⊥ = ⊓L.

A **semilattice** has only one of join or meet. A **meet-semilattice** (S, ⊓) is often the natural structure for "must" analyses (every path must witness the property). A **join-semilattice** (S, ⊔) is natural for "may" analyses (some path may witness the property). The Kildall framework is usually cast as a meet-semilattice with a top element.

### 10.1.3 Complete Lattices

A **complete lattice** (L, ≤) is one in which *every* subset T ⊆ L has both a lub (⊔T) and a glb (⊓T). This is stronger than a lattice (which requires only pairwise lub and glb). Every finite lattice is complete. Tarski's theorem requires completeness; Kleene's theorem requires a finite chain condition, which follows from finite height.

**Key examples of complete lattices:**

**(a) The powerset lattice ℘(S).** For any set S, the collection of all subsets ordered by ⊆ is a complete lattice. Join is ∪, meet is ∩, ⊤ = S, ⊥ = ∅. This is the workhorse lattice for liveness, reaching definitions, available expressions, and very busy expressions.

```
     S = {x,y,z}          ⊤ = {x,y,z}
        / | \
    {x,y} {x,z} {y,z}
    / \   / \   / \
  {x} {y}   {z}
      \  |  /
          ∅               ⊥ = ∅
```

**(b) The flat lattice.** Given a set V of values, build a lattice by adding ⊤ and ⊥ with no ordering among elements of V:

```
         ⊤  (overdefined / unknown)
       / | \
      1  2  3  ...  (concrete values, all incomparable)
       \ | /
         ⊥  (undefined / unreachable)
```

This is the lattice for constant propagation; any two distinct concrete values meet to ⊥ (contradiction / not a constant).

**(c) The product lattice.** If (L₁, ≤₁) and (L₂, ≤₂) are complete lattices, so is L₁ × L₂ with the pointwise order (a₁, a₂) ≤ (b₁, b₂) ⟺ a₁ ≤₁ b₁ ∧ a₂ ≤₂ b₂. This extends to any finite or countable product. The per-program-variable product of flat lattices is the lattice for constant propagation of an entire environment.

**(d) The function lattice.** If (L, ≤) is a complete lattice and S is any set, then the set of all functions f : S → L ordered pointwise is a complete lattice. Transfer functions on environments form instances of this lattice.

### 10.1.3.1 Why Completeness Matters

A lattice that is not complete can fail to have a least upper bound for an infinite ascending chain — a situation that arises in any analysis involving loops. The integers ℤ under ≤ form a totally ordered set that is not a complete lattice: the subset ℕ = {0, 1, 2, ...} has no least upper bound within ℤ. Any analysis whose states are drawn from ℤ would fail to converge. Completeness is non-negotiable for the convergence arguments to go through.

The **two-point lattice** (also called the Boolean lattice) 2 = {⊥, ⊤} is the simplest non-trivial complete lattice. It appears in reachability analyses and branch condition analyses. The lattice 2^n (n independent Boolean flags) models the domain for liveness restricted to n named variables: each flag tracks whether a specific variable is live.

**The sign lattice.** A richer example is the sign abstraction used in sign analysis, tracking whether a variable is negative, zero, positive, or unknown:

```
        ⊤  (unknown sign / may be any)
      / | \
    -   0   +   (definitely negative / zero / positive)
      \ | /
        ⊥  (no value — unreachable)
```

This is a five-element flat-like lattice with the three concrete sign values incomparable. Abstract addition on sign values: (+) + (+) = (+); (+) + (-) = ⊤ (can be any sign); (0) + (x) = (x). Sign analysis is distributive over the gen/kill pattern and terminates in O(n) iterations.

### 10.1.4 Height of a Lattice and Chain Conditions

A **chain** in (L, ≤) is a totally ordered subset c₀ ≤ c₁ ≤ c₂ ≤ .... The **height** of L is the supremum of lengths of all chains. A finite-height lattice satisfies both:

- **Ascending chain condition (ACC)**: every ascending chain c₀ ≤ c₁ ≤ c₂ ≤ ... eventually stabilises (there is no infinite strictly ascending chain).
- **Descending chain condition (DCC)**: every descending chain c₀ ≥ c₁ ≥ c₂ ≥ ... eventually stabilises.

Both conditions hold for any lattice of finite height. The ACC is the key property guaranteeing termination of upward iterative dataflow algorithms (those that start at ⊥ and climb). The DCC guarantees termination of downward algorithms (those that start at ⊤ and descend). For the powerset lattice ℘(V) over a finite set V of n elements, the height is n (the longest chain is ∅ ⊊ {v₁} ⊊ {v₁,v₂} ⊊ ... ⊊ V). This means iterative analyses over ℘(V) terminate in at most n+1 iterations per basic block.

---

## 10.2 Tarski's Fixed-Point Theorem

### 10.2.1 Statement and Proof

Let (L, ≤) be a complete lattice and let f : L → L be **monotone**: a ≤ b ⟹ f(a) ≤ f(b). Tarski's theorem [Tarski 1955] guarantees:

> **Theorem (Tarski 1955).** Every monotone function on a complete lattice has a least fixed point lfp(f) and a greatest fixed point gfp(f). Specifically:
>
>   lfp(f) = ⊓ { x ∈ L | f(x) ≤ x }
>   gfp(f) = ⊔ { x ∈ L | x ≤ f(x) }

**Proof of lfp.** Let P = { x ∈ L | f(x) ≤ x } be the set of *post-fixed points* (elements above f's image). Let m = ⊓P. We must show (i) m ∈ P and (ii) m is a fixed point.

*(i) m ∈ P:* For every x ∈ P, m ≤ x, so by monotonicity f(m) ≤ f(x) ≤ x. Since this holds for all x ∈ P, f(m) is a lower bound of P, so f(m) ≤ ⊓P = m. Hence f(m) ≤ m, i.e., m ∈ P.

*(ii) m is a fixed point:* From f(m) ≤ m, applying monotonicity gives f(f(m)) ≤ f(m), so f(m) ∈ P. Then m = ⊓P ≤ f(m). Combined with f(m) ≤ m, antisymmetry gives f(m) = m. □

*(Least fixed point):* If f(a) = a for any a, then a ∈ P, so m = ⊓P ≤ a. □

### 10.2.2 Proof of the Greatest Fixed Point

By a symmetric argument, let Q = { x ∈ L | x ≤ f(x) } be the set of *pre-fixed points*. Let M = ⊔Q.

*(M ∈ Q):* For every x ∈ Q, x ≤ M, so x ≤ f(x) ≤ f(M) by monotonicity. Thus f(M) is an upper bound of Q, so M = ⊔Q ≤ f(M), meaning M ∈ Q.

*(M is a fixed point):* M ≤ f(M). Applying monotonicity: f(M) ≤ f(f(M)), so f(M) ∈ Q. Then f(M) ≤ ⊔Q = M. Antisymmetry gives f(M) = M. □

*(Greatest fixed point):* If f(a) = a, then a ∈ Q, so a ≤ M. □

**Which fixed point is relevant?** For safety analyses — those where being wrong means accepting something unsound — we want the *most conservative* (most restricting) solution. For a forward may-analysis (liveness, reaching definitions), the MFP is the least fixed point starting from ⊥; it under-approximates the concrete set of live variables and is therefore sound for transformations that *remove* dead code. For a forward must-analysis (available expressions), we want the *greatest* conservative answer starting from ⊤ and descending: only expressions certain to be available on all paths. In the Kildall framework convention, both are described uniformly as the MFP of a combined function over the product lattice, with the direction (ascending from ⊥ or descending from ⊤) chosen to match the analysis.

### 10.2.3 Consequence for Dataflow

A dataflow system is a system of equations of the form:

```
  OUT[BB] = f_BB(IN[BB])
  IN[BB]  = ⊔_{P ∈ preds(BB)} OUT[P]   (forward analysis)
```

This system defines a combined function F : L^n → L^n (where n is the number of basic blocks) by applying each equation component-wise. If each f_BB is monotone, then F is monotone on the product lattice L^n. By Tarski, F has a least fixed point, and that least fixed point is the solution we want. The iterative algorithm computes an ascending chain ⊥ ≤ F(⊥) ≤ F²(⊥) ≤ ... that converges to lfp(F) when L has the ACC.

**The product lattice L^n.** For n basic blocks, each carrying a dataflow value from lattice (L, ≤), the product lattice is (L^n, ≤^n) where (v₁, ..., vₙ) ≤^n (w₁, ..., wₙ) iff vᵢ ≤ wᵢ for all i. The combined transfer function F(v₁, ..., vₙ) = (g₁(v), ..., gₙ(v)) where gᵢ(v) = f_BBᵢ(⊔_{P∈preds(BBᵢ)} v_P) is monotone component-wise because each f_BBᵢ is monotone. The height of L^n is n × h(L), bounding the total number of F-applications before convergence.

**Why start from ⊥?** Initialising from ⊥ (the bottom of L^n) and ascending gives the *least* fixed point — the minimum amount of information about the program consistent with the dataflow equations. For may-analyses (liveness, reaching definitions), ⊥ = ∅ means "nothing is live / no definition reaches." As the iteration proceeds, facts are added only when the equations demand it, keeping the result as precise as possible. For must-analyses (available expressions), we typically initialise from ⊤ and descend, so the iteration removes expressions as paths are discovered that invalidate them — giving the *most permissive* sound result.

---

## 10.3 Kleene Iteration and the Worklist Algorithm

### 10.3.1 Kleene's Ascending Chain Theorem

**Theorem (Kleene).** Let (L, ≤) be a complete lattice satisfying the ACC, and let f : L → L be monotone. The ascending Kleene sequence:

```
  a₀ = ⊥,  aₙ₊₁ = f(aₙ)
```

converges to lfp(f) in finitely many steps.

**Proof.** Each step either strictly increases (aₙ < aₙ₊₁) or stabilises (aₙ = aₙ₊₁). By the ACC, there can be at most h strictly increasing steps, where h is the height of L. Once aₙ = aₙ₊₁, aₙ is a fixed point. Since the sequence is ascending and every element is ≤ lfp(f) (because lfp(f) is an upper bound of the entire ascending chain, which is clearly below any post-fixed point), the limit is lfp(f). □

For the product lattice L^n over n basic blocks, the height is n × h(L), so the maximum number of iterations before convergence is n × h(L).

### 10.3.2 The Round-Robin Iterative Algorithm

The naïve instantiation processes all basic blocks in each round:

```
procedure DataflowIterate(CFG, direction):
    -- Initialize
    for each BB in CFG:
        IN[BB]  := ⊥  (or ⊤ for must-analyses initialising with ⊤)
        OUT[BB] := ⊥

    -- If forward, process in reverse postorder (RPO) for faster convergence
    -- If backward, process in postorder
    changed := true
    while changed:
        changed := false
        for each BB in traversal_order(CFG, direction):
            if direction == FORWARD:
                new_in  := meet over {OUT[P] | P ∈ preds(BB)}
                new_out := f_BB(new_in)
                if new_out ≠ OUT[BB]:
                    OUT[BB] := new_out; changed := true
                IN[BB] := new_in
            else:  -- BACKWARD
                new_out := meet over {IN[S] | S ∈ succs(BB)}
                new_in  := f_BB(new_out)
                if new_in ≠ IN[BB]:
                    IN[BB] := new_in; changed := true
                OUT[BB] := new_out
```

For forward analyses, initialise IN[entry] to the program's initial dataflow value (e.g., the set of all parameters that are defined on entry) and IN[BB] to ⊤ for all other blocks (under the must/intersection convention) or ⊥ for may/union analyses.

### 10.3.3 The Worklist Algorithm

The round-robin algorithm re-processes blocks whose inputs have not changed — wasteful. The **worklist algorithm** [Kildall 1973] maintains a set (or priority queue) of blocks that need reprocessing:

```
procedure DataflowWorklist(CFG, direction):
    -- Initialize
    for each BB in CFG:
        IN[BB] := ⊥;  OUT[BB] := ⊥

    worklist := all BBs (or just entry for forward, exit for backward)

    while worklist ≠ ∅:
        BB := remove_some_element(worklist)

        if direction == FORWARD:
            new_in  := meet over {OUT[P] | P ∈ preds(BB)}
            new_out := f_BB(new_in)
            if new_out ≠ OUT[BB]:
                OUT[BB] := new_out
                IN[BB]  := new_in
                for each S ∈ succs(BB):
                    add S to worklist
        else:
            new_out := meet over {IN[S] | S ∈ succs(BB)}
            new_in  := f_BB(new_out)
            if new_in ≠ IN[BB]:
                IN[BB]  := new_in
                OUT[BB] := new_out
                for each P ∈ preds(BB):
                    add P to worklist
```

**Termination.** A block is added to the worklist only when its OUT (or IN) value changes. Since the value can only monotonically increase (in the join-based direction) or decrease (in the meet direction) and the lattice has finite height, each block's value can change at most h(L) times. With n blocks the total number of worklist insertions is bounded by n × h(L) × max_degree, where max_degree is the maximum CFG in-degree.

**Convergence speed in practice.** For reducible CFGs (those arising from structured programs), Rosen [Rosen 1980] showed that two passes suffice for many analyses: one forward pass computes acyclic path information; one or two additional passes propagate information around back-edges (loop headers). In practice, most production analyses converge in 2–4 passes for typical programs [EaC §9.2].

**Ordering within the worklist.** Processing blocks in reverse postorder (RPO) for forward analyses minimises the number of passes over the CFG for reducible graphs. RPO is a topological ordering of the acyclic portion of the CFG extended to handle back-edges; in RPO, every predecessor of a block is processed before the block itself (except for loop-header predecessors that are the block's own back-edge targets).

### 10.3.4 Bit-Vector Implementations

For gen/kill analyses over finite sets, the lattice elements are bit-vectors where bit i represents whether the i-th fact (variable, definition, or expression) is in the set. Operations become hardware-accelerated bitwise instructions:

- Set union (join for may-analyses): `result |= in`
- Set intersection (meet for must-analyses): `result &= in`
- Set difference (kill): `result &= ~kill_mask`
- Set union (gen): `result |= gen_mask`

A single liveness update step for one basic block reduces to:

```c
// LiveIN[BB] = Use[BB] | (LiveOUT[BB] & ~Def[BB])
for (int w = 0; w < words; w++) {
    liveIN[w] = use[w] | (liveOUT[w] & ~def[w]);
}
```

On a 64-bit machine with 64-bit words, processing 64 variables per instruction makes bit-vector analyses extremely fast in practice. LLVM's `BitVector` and `SparseBitVector` classes in `llvm/include/llvm/ADT/` implement this abstraction. For programs with hundreds of variables, `SparseBitVector` avoids allocating full words for sparse live sets.

**Complexity analysis.** For a program with V variables and n basic blocks with E CFG edges, the liveness worklist algorithm runs in O(n × V/64) time per worklist processing step (each step touches V/64 words per block) and O(E × V/64) total, since each block's LiveIN can change at most V times before stabilising.

### 10.3.5 SSA and Dataflow

When the program is in SSA form (Chapter 9), the single-definition invariant drastically simplifies many dataflow analyses:

- **Reaching definitions on SSA**: trivial — every use U of variable v is connected to the unique definition D of v via the SSA def-use chain. No fixed-point iteration is needed; the "reaching definition" of every use is structurally explicit.
- **Liveness on SSA**: the live range of variable v is exactly the set of program points on any path from D (the definition of v) to any use of v. Computing live ranges reduces to a single backward walk from each use to its definition — still linear, but with no fixed-point iteration needed for simple SSA liveness.
- **Available expressions on SSA**: with SSA, the notion of "availability" is replaced by the SSA value: if a computation's result is an SSA value `%r`, it is "available" at any dominated point that can reach a use of `%r`. LLVM's GVN (Global Value Numbering, Chapter 62) exploits this: it assigns a value number to each SSA value and propagates numbers through the dominator tree, detecting redundant computations without an explicit availability fixed-point.

The deep reason for SSA's advantage is that it makes the dataflow information explicit in the program's structure, eliminating the need for iterative re-derivation. Dataflow frameworks remain necessary for analyses that SSA does not capture — notably pointer aliasing, calling-convention effects, and any analysis that crosses φ-functions in a non-trivial way.

---

## 10.4 Monotone vs. Distributive Frameworks and MOP vs. MFP

### 10.4.1 Monotone Frameworks

The **monotone framework** [Kildall 1973, Kam & Ullman 1976] consists of:

- A complete lattice (L, ≤) with finite height.
- A set of transfer functions f_BB : L → L, one per basic block, each monotone.
- A meet operator ⊓ for combining values at join points.

The iterative algorithm for a monotone framework is guaranteed to terminate and produce the **maximum fixed point (MFP)** solution.

### 10.4.2 Distributive Frameworks

A framework is **distributive** if every transfer function satisfies:

```
  f_BB(x ⊓ y) = f_BB(x) ⊓ f_BB(y)   for all x, y ∈ L
```

(Using ⊓ for meet; equivalently with ⊔ for join.) Distributivity implies monotonicity (but not vice versa). The gen/kill analyses — liveness, reaching definitions, available expressions, very busy expressions — are distributive because their transfer functions have the form f(x) = gen ∪ (x \ kill), which distributes over ∪ and ∩.

Constant propagation is **not** distributive: see Section 10.6.5.

### 10.4.3 The MOP Solution and Why MFP ≤ MOP

The **meet-over-all-paths (MOP)** solution is the ideal result of a dataflow analysis. For a forward analysis at block BB, let paths(entry, BB) be the (possibly infinite) set of all paths from entry to BB in the CFG. The dataflow value along path π = (BB₀, BB₁, ..., BBₖ) is the composition of transfer functions:

```
  df(π) = f_BBₖ(... f_BB₁(f_BB₀(⊤)) ...)
```

The MOP value at BB is:

```
  MOP[BB] = ⊓ { df(π) | π ∈ paths(entry, BB) }
```

Direct MOP computation is impractical because the set of paths is generally exponential (for DAGs) or infinite (for loops). The iterative algorithm computes MFP instead.

**Theorem: MFP ≤ MOP (in general).** The MFP value is always a lower bound (less specific or more conservative) than the ideal MOP value in a monotone framework.

**Proof sketch.** At the fixed point, IN[BB] = ⊓_{P∈preds(BB)} OUT[P] and OUT[BB] = f_BB(IN[BB]). One can show by induction on path length that for every finite path π from entry to BB, df(π) ≥ MFP[BB]: the iterative merging at each join point can only lose information compared to tracking each path separately [Dragon §9.3].

**Theorem: MFP = MOP for distributive frameworks.** [Kam & Ullman 1976]

**Proof.** By induction on path length k. Base case (k = 0, single block): trivial. Inductive step: assume MFP[BB] = MOP[BB] for all blocks reachable in ≤ k steps. For a block BB' with predecessors P₁, ..., Pₘ each reachable in ≤ k steps:

```
MFP[BB'] = f_BB'(⊓ᵢ MFP[Pᵢ])
          = f_BB'(⊓ᵢ MOP[Pᵢ])               (by induction hypothesis)
          = f_BB'(⊓ᵢ ⊓{df(π) | π ends at Pᵢ})
          = ⊓ᵢ f_BB'(⊓{df(π) | π ends at Pᵢ})  (f monotone, applied once)
```

For the last step to equal MOP[BB'], we need f_BB'(⊓ᵢ Xᵢ) = ⊓ᵢ f_BB'(Xᵢ) — that is, distributivity of f_BB'. Under the distributivity assumption, f_BB'(x ⊓ y) = f_BB'(x) ⊓ f_BB'(y), which generalises to finite meets by induction. Applying this:

```
= ⊓ᵢ ⊓{f_BB'(df(π)) | π ends at Pᵢ}
= ⊓{df(π') | π' ends at BB'}   (where π' = π extended by BB')
= MOP[BB']   □
```

The practical consequence: for gen/kill analyses (all distributive), the iterative worklist algorithm computes the ideal answer. For constant propagation (non-distributive), it may be imprecise — though still sound (conservative).

### 10.4.4 A Taxonomy of Classical Analyses

It is instructive to classify the classical analyses systematically:

| Analysis | Direction | May/Must | Lattice order | Join op | Meet op | ⊤ | ⊥ | Distributive? |
|---|---|---|---|---|---|---|---|---|
| Liveness | Backward | May | ⊆ | ∪ | ∩ | Vars | ∅ | Yes |
| Reaching Defs | Forward | May | ⊆ | ∪ | ∩ | Defs | ∅ | Yes |
| Available Exprs | Forward | Must | ⊆ | ∩ | ∪ | Exprs | ∅ | Yes |
| Very Busy Exprs | Backward | Must | ⊆ | ∩ | ∪ | Exprs | ∅ | Yes |
| Const. Prop. | Forward | — | flat | ⊓_flat | ⊔_flat | undef | NAC | No |
| Sign Analysis | Forward | — | flat(sign) | ⊓ | ⊔ | ⊤ | ⊥ | Yes |
| Interval Analysis | Forward | — | [l,u] order | ∪_interval | ∩_interval | [−∞,+∞] | ⊥ | No |

The "may" vs. "must" designation determines the direction of the monotone progression: may-analyses start optimistically (∅) and accumulate facts; must-analyses start pessimistically (full set) and prune. Both are instances of the same mathematical framework with different lattice orientations [EaC §9.1].

**Forward vs. backward duality.** Every forward analysis can be converted to a backward analysis by reversing the CFG (swapping predecessors and successors, swapping entry and exit). This duality is important for the theory (it shows forward and backward analyses are equally expressive) but rarely exploited in practice since the natural direction of each analysis is determined by the dependency structure of the facts.

### 10.4.5 Composing Analyses: The Cartesian Product Construction

If two analyses use lattices (L₁, ≤₁) and (L₂, ≤₂), their simultaneous computation can be expressed as a single analysis over the product lattice L₁ × L₂. The combined transfer function is:

```
  (f₁, f₂)(x₁, x₂) = (f₁(x₁), f₂(x₂))
```

This is monotone and distributive if each component is. The worklist algorithm for the product analysis is identical to running the two analyses in lockstep: when a block is processed, both components are updated. This is more efficient than running them independently because the worklist overhead is shared.

LLVM frequently exploits this: the `DemandedBits` analysis, `ValueTracking`, and `InstructionSimplify` all compute related but complementary information in a single pass over the IR, effectively implementing a product analysis without explicitly constructing a product lattice object.

---

## 10.5 Transfer Functions: Gen/Kill and Beyond

### 10.5.1 The Gen/Kill Form

Many analyses use transfer functions of the form:

```
  f_BB(x) = gen(BB) ∪ (x \ kill(BB))
```

where gen(BB) is the set of facts *generated* within BB, and kill(BB) is the set of facts *killed* (invalidated) by BB. This form is always distributive:

```
f_BB(x ∪ y) = gen ∪ ((x ∪ y) \ kill)
            = gen ∪ (x \ kill) ∪ (y \ kill)
            = f_BB(x) ∪ f_BB(y)   □
```

Gen and kill sets can be computed by a single linear scan over the instructions in BB. For a basic block BB = [i₁; i₂; ...; iₙ], the combined gen and kill sets are computed by scanning instructions in order:

```
procedure ComputeGenKill(BB):
    gen  := ∅
    kill := ∅
    for each instruction i in BB (in program order):
        gen  := (gen \ kill_i) ∪ gen_i    -- instructions before i already generated facts
        kill := kill ∪ kill_i
    return (gen, kill)
```

This sequential composition ensures that later instructions in the block correctly shadow earlier ones.

### 10.5.2 Forward vs. Backward Analyses

- **Forward analysis**: information flows from predecessors to successors. IN[BB] depends on OUT[preds(BB)]; transfer function maps IN to OUT. Appropriate when "what happened before?" determines usefulness of current computation.
- **Backward analysis**: information flows from successors to predecessors. OUT[BB] depends on IN[succs(BB)]; transfer function maps OUT to IN. Appropriate when "what will happen after?" determines usefulness of current computation.

Liveness and very busy expressions are backward; available expressions and reaching definitions are forward. The direction determines which blocks are "entry" (for initialisation) and which are "exit" (for termination), and whether the worklist processes successors or predecessors.

### 10.5.3 Computing Gen/Kill for Common Instruction Types

For concreteness, here is how gen and kill sets are computed for each instruction category in a three-address IR (such as LLVM IR):

**Liveness gen/kill (per basic block):**

| Instruction | Effect |
|---|---|
| `v = op(a, b)` | gen: {a, b} (if not already killed in BB); kill: {v} |
| `store a, *p` | gen: {a, p}; kill: ∅ (indirect write; conservative for memory) |
| `v = load *p` | gen: {p}; kill: {v} |
| `call f(a₁,...,aₙ)` | gen: {a₁,...,aₙ}; kill: {caller-saves, return value} |
| `branch cond, L1, L2` | gen: {cond}; kill: ∅ |

**Reaching definitions gen/kill (per basic block):**

Each definition `d: v = ...` generates a new reaching definition and kills all other definitions of v visible from earlier blocks.

| Instruction | Effect |
|---|---|
| `d: v = op(...)` | gen: {d}; kill: all definitions of v except d |
| `call f(...)` returning v | gen: {d_ret}; kill: all previous defs of v; also kills defs of any memory location f may modify |
| φ-function at block join | creates a new definition; kills nothing (SSA invariant: unique defs) |

**Available expressions gen/kill (per basic block):**

An expression `a op b` is generated (made available) when it is computed and all subsequent modifications to its operands within the block are absent; it is killed when any operand is modified.

| Instruction | Effect |
|---|---|
| `v = a + b` | gen: {a+b} (if a and b not written later in BB); kill: all expressions using v |
| `v = load *p` | kill: all expressions involving memory (conservatively), or precisely tracked with alias analysis |
| `store a, *p` | kill: all expressions involving any location p may alias |

The aliasing challenge for available expressions is why modern compilers supplement the basic gen/kill framework with alias analysis [Chapter 61] to compute more precise kill sets, particularly for expressions involving memory operands.

---

## 10.6 Classical Analyses

### 10.6.1 Liveness Analysis

**Purpose.** A variable v is **live** at program point p if there exists *some* path from p to a use of v that passes through no intervening re-definition of v. Liveness is a **may analysis** (existence of one path suffices) and a **backward analysis** (whether v is live at p depends on future uses) [Dragon §9.5, EaC §8.6].

**Lattice.** ℘(Vars), the powerset of all program variables, ordered by ⊆. Join is ∪, meet is ∩. ⊥ = ∅ (no variable is known to be live — this is the most precise/optimistic state). ⊤ = Vars (all variables are assumed live — this is the most conservative initial guess for the backward iteration). In a backward may-analysis, we initialise OUT[exit] = ∅ (no variables live beyond the program's exit) and all other OUT[BB] = ∅, then propagate upward.

```
Hasse diagram (for Vars = {a, b, c}):

         {a,b,c}      ← ⊤  (all live: conservative)
        / | | \ 
    {a,b}{a,c}{b,c}
    / \  |  / \
  {a}  {b} {c}
     \  |  /
        ∅            ← ⊥  (nothing known live: most optimistic init)
```

**Transfer function (backward).**

```
  gen(BB)  = Use(BB)   -- variables read in BB before any definition in BB
  kill(BB) = Def(BB)   -- variables written (defined) in BB

  LiveIN[BB]  = Use(BB) ∪ (LiveOUT[BB] \ Def(BB))
  LiveOUT[BB] = ∪_{S ∈ succs(BB)} LiveIN[S]
```

The worklist algorithm propagates from successors back to predecessors. A block BB is re-added to the worklist whenever any LiveIN[succ] changes.

**Worked Example.** Consider a small loop CFG with four blocks:

```
  entry:
    a = 1
    b = 2
    goto loop_header

  loop_header:                   ← B1
    if a < 10 goto loop_body else exit

  loop_body:                     ← B2
    c = a + b
    a = a + 1
    goto loop_header

  exit:                          ← B3
    use(c)
```

Variables: {a, b, c}. Compute Use and Def for each block:

| Block    | Use        | Def   |
|----------|------------|-------|
| entry    | ∅          | {a,b} |
| B1       | {a}        | ∅     |
| B2       | {a,b}      | {c,a} |
| B3       | {c}        | ∅     |

*Initialise:* LiveOUT[B] = ∅ for all B; worklist = {B3, B2, B1, entry}.

*Iteration 1 (process B3):*
  LiveIN[B3] = Use(B3) ∪ (LiveOUT[B3] \ Def(B3)) = {c} ∪ (∅ \ ∅) = {c}
  LiveOUT[B1] did not change yet.

*Process B1 (successor B2 and B3):*
  LiveOUT[B1] = LiveIN[B2] ∪ LiveIN[B3] = ∅ ∪ {c} = {c}
  LiveIN[B1]  = {a} ∪ ({c} \ ∅) = {a, c}
  Predecessors of B1 (entry and B2) added to worklist.

*Process B2 (successor B1):*
  LiveOUT[B2] = LiveIN[B1] = {a, c}
  LiveIN[B2]  = {a, b} ∪ ({a, c} \ {c, a}) = {a, b} ∪ ∅ = {a, b}
  B1 re-added (its predecessor B2's LiveIN changed).

*Process B1 again:*
  LiveOUT[B1] = LiveIN[B2] ∪ LiveIN[B3] = {a, b} ∪ {c} = {a, b, c}
  LiveIN[B1]  = {a} ∪ ({a, b, c} \ ∅) = {a, b, c}
  Predecessor entry and B2 re-added.

*Process B2 again:*
  LiveOUT[B2] = LiveIN[B1] = {a, b, c}
  LiveIN[B2]  = {a, b} ∪ ({a, b, c} \ {c, a}) = {a, b} ∪ {b} = {a, b}
  No change to LiveIN[B2] → no propagation.

*Process entry:*
  LiveOUT[entry] = LiveIN[B1] = {a, b, c}
  LiveIN[entry]  = ∅ ∪ ({a, b, c} \ {a, b}) = {c}

*Fixed point reached.* Final liveness:

| Block  | LiveIN    | LiveOUT   |
|--------|-----------|-----------|
| entry  | {c}       | {a,b,c}   |
| B1     | {a,b,c}   | {a,b,c}   |
| B2     | {a,b}     | {a,b,c}   |
| B3     | {c}       | ∅         |

Interpretation: At the start of B2, only `a` and `b` are needed (not `c`, since B2 overwrites `c` before any use); `c` first becomes live after B2 computes it and B3 uses it. Register allocation would need two live registers simultaneously at the top of B1 (all three), but in the body of B2, `c` need not occupy a register until after the addition.

### 10.6.2 Available Expressions

**Purpose.** An expression e is **available** at program point p if every path from entry to p evaluates e and does not subsequently modify any operand of e. Available expressions is a **must analysis** (every path must evaluate e) and a **forward analysis**. It is used for common subexpression elimination (CSE): if e is available at a use of e, the recomputation is redundant [Dragon §9.4, EaC §8.5].

**Lattice.** ℘(Exprs) ordered by ⊇ — here, more available expressions is *more* information (we know more expressions are safe to reuse). Alternatively framed with ⊆: a larger set (more expressions known available) corresponds to a higher lattice element. Under the ⊆ ordering, meet is ∩ (the conservative operation: an expression is available only if every predecessor says it is). ⊤ = Exprs (the optimistic initialisation: assume everything is available). ⊥ = ∅ (no expression is available: the conservative initialisation for entry). For the entry block, AvailIN[entry] = ∅.

```
Lattice (for Exprs = {a+b, c*d}):

     {a+b, c*d}    ← ⊤ (most optimistic: all available)
       /       \
   {a+b}       {c*d}
       \       /
           ∅       ← ⊥ (nothing available: entry initialisation)
```

**Transfer function (forward).**

```
  gen(BB)  = {e | e computed in BB, no operand of e modified after the computation}
  kill(BB) = {e | some operand of e is written in BB}

  AvailOUT[BB] = gen(BB) ∪ (AvailIN[BB] \ kill(BB))
  AvailIN[BB]  = ∩_{P ∈ preds(BB)} AvailOUT[P]
```

Initialise AvailOUT[BB] = Exprs (⊤) for all BB except entry; AvailOUT[entry] = gen(entry). The intersection at join points is the must-meet: an expression is available only if it is available on every incoming path. The iterative algorithm descends from ⊤ toward ⊥, converging to the MFP. Since this is a distributive framework, MFP = MOP.

### 10.6.3 Reaching Definitions

**Purpose.** A definition d (an assignment to a variable at a specific program point) **reaches** program point p if there is some path from d to p along which d is not killed (no other definition of the same variable appears on that path). Reaching definitions is a **may analysis** (any path suffices) and a **forward analysis**. It forms the foundation for use-def chains and constant propagation [Dragon §9.2].

**Lattice.** ℘(Defs) ordered by ⊆. Join is ∪ (any reaching definition counts). ⊥ = ∅ (no definitions reach — correct for entry). ⊤ = all definitions.

**Transfer function.**

```
  gen(BB)  = {d | d is a definition in BB and d is not killed by a later
                   definition of the same variable within BB}
  kill(BB) = {d | d is a definition of variable v outside BB, and BB
                   contains a definition of v}

  ReachOUT[BB] = gen(BB) ∪ (ReachIN[BB] \ kill(BB))
  ReachIN[BB]  = ∪_{P ∈ preds(BB)} ReachOUT[P]
```

**Worked example (reaching definitions).** Consider:

```
  entry:
    d1: x = 5
    d2: y = 3
    goto B1

  B1:                         ← loop header
    if y > 0 goto B2 else exit

  B2:
    d3: x = x + y             ← uses x (reaching d1 or d3) and y (reaching d2 or d4)
    d4: y = y - 1
    goto B1

  exit:
    use(x, y)
```

Definitions: d1 (x=5), d2 (y=3), d3 (x=x+y), d4 (y=y-1).

| Block | gen | kill |
|---|---|---|
| entry | {d1,d2} | {d3,d4} (d1 kills d3; d2 kills d4) |
| B1 | ∅ | ∅ |
| B2 | {d3,d4} | {d1,d2} (d3 kills d1; d4 kills d2) |
| exit | ∅ | ∅ |

*Initial:* ReachOUT[all] = ∅.

After iteration 1: ReachOUT[entry] = {d1,d2}; ReachIN[B1] = {d1,d2}; ReachOUT[B1] = {d1,d2}; ReachIN[B2] = {d1,d2}; ReachOUT[B2] = {d3,d4} ∪ ({d1,d2} \ {d1,d2}) = {d3,d4}.

After iteration 2 (back-edge from B2 to B1): ReachIN[B1] = ReachOUT[entry] ∪ ReachOUT[B2] = {d1,d2} ∪ {d3,d4} = {d1,d2,d3,d4}. ReachOUT[B2] = {d3,d4} ∪ ({d1,d2,d3,d4} \ {d1,d2}) = {d3,d4}. No further change.

*Fixed point:* At B2, x can be defined by d1 (initial value 5) or d3 (loop iteration), and y can be defined by d2 (initial 3) or d4 (decremented). This matches intuition. The reaching definitions analysis is the substrate for use-def chain construction and, ultimately, for SSA φ-function placement (Chapter 9).

In SSA form, every definition reaches exactly the uses connected to it by the SSA def-use chain. Reaching definitions on SSA therefore degenerates to a trivial structural property — one of the key benefits of SSA. Out of SSA, the explicit computation via the worklist algorithm is needed.

### 10.6.4 Very Busy Expressions

**Purpose.** An expression e is **very busy** at program point p if on *every* path from p to exit, e is evaluated before any operand of e is modified. Very busy expressions is a **must analysis** and a **backward analysis**. The classical use is **code hoisting** (earliest placement): if e is very busy at the exit of block BB, we can hoist the evaluation of e to the entry of BB without changing program semantics, reducing total evaluations [Muchnick §13.1].

**Lattice.** ℘(Exprs) ordered by ⊆. Meet is ∩ (must appear on every path). ⊤ = Exprs (optimistic: assume all expressions very busy — correct initialisation for backward pass). ⊥ = ∅.

**Transfer function (backward).**

```
  gen(BB)  = {e | e used in BB before any operand of e is modified in BB}
  kill(BB) = {e | some operand of e is modified in BB before e is used in BB}

  VBusy_IN[BB]  = gen(BB) ∪ (VBusy_OUT[BB] \ kill(BB))
  VBusy_OUT[BB] = ∩_{S ∈ succs(BB)} VBusy_IN[S]
```

Initialise VBusy_IN[BB] = Exprs for all non-exit blocks; VBusy_IN[exit] = ∅.

**Example.** Consider:

```
  B1:
    if (...) goto B2 else goto B3

  B2:
    z = x + y
    use(z)
    goto B4

  B3:
    z = x + y
    use(z)
    goto B4

  B4:
    exit
```

The expression `x + y` appears in both branches B2 and B3, so it is very busy at B1 (computed on every path from B1 to exit, with no modification of x or y). Very busy expression analysis reveals this: VBusy_OUT[B1] = VBusy_IN[B2] ∩ VBusy_IN[B3] = {x+y} ∩ {x+y} = {x+y}. The compiler can hoist the computation of `x + y` before the branch in B1.

**Connection to partial redundancy elimination (PRE).** Very busy expressions are closely related to PRE (Chapter 11), which generalises code hoisting and CSE into a unified transformation. Knoop, Rüthing, and Steffen's lazy code motion [Knoop et al. 1992] computes optimal placement of expression evaluations using four combined dataflow passes: earliest possible placement (based on very busy expressions), latest possible placement, and delay/isolatedness conditions.

### 10.6.5 Constant Propagation

**Purpose.** Determine, at each program point, which variables hold compile-time constant values. This enables constant folding, dead branch elimination, and simplification of dependent computations [Dragon §9.4.5, EaC §9.3].

**The lattice.** For each variable v, define a three-level flat lattice:

```
        ⊤   ("undefined" / not yet reached — optimistic start)
      / | \
     1  2  3  ...   (specific constant values, all incomparable)
      \ | /
        ⊥   ("overdefined" / not a constant — two different values seen)
```

The meet operation: c₁ ⊓ c₂ = c₁ if c₁ = c₂; ⊥ if c₁ ≠ c₂ (two different constants); and any value ⊓ ⊤ = that value; any value ⊓ ⊥ = ⊥.

The program-wide lattice is the product: for n variables, L = ∏ᵢ Lᵢ where each Lᵢ is the three-level flat lattice for variable vᵢ.

**Transfer function.** For an instruction `v = e`:
- If e evaluates to a constant c under the current assignment (all operands are constants), set v ← c.
- If any operand of e is ⊥ (not a constant), set v ← ⊥.
- If any operand is ⊤ (not yet analysed), keep v at ⊤ (wait for more information).

**Non-distributivity.** Constant propagation is **not** a distributive framework. Consider:

```
  -- Path 1: a = 2, b = 3
  -- Path 2: a = 3, b = 2
  -- Join point: compute x = a + b
```

Along each individual path, x = 5 (a constant). The MOP value for x is 5. But the per-variable lattice at the join point computes a ⊓ = ⊥ (overdefined, since 2 ≠ 3) and b ⊓ = ⊥ — so the transfer function gives x = ⊥. We have MFP[x] = ⊥ ≠ 5 = MOP[x]. The iterative algorithm is sound (conservative) but not precise.

Formally: f_{join}(x ⊔ y) = 5, but f_{join}(x) ⊔ f_{join}(y) = 5 ⊔ 5 = 5 — wait, the join of two copies of 5 is 5. The failure of distributivity occurs at the *meet* of the per-variable abstractions: f_{BB}({a=2, b=3} ⊓ {a=3, b=2}) = f_{BB}({a=⊥, b=⊥}) = {x=⊥}, while f_{BB}({a=2, b=3}) ⊓ f_{BB}({a=3, b=2}) = {x=5} ⊓ {x=5} = {x=5}. Hence f({a=⊥,b=⊥}) = ⊥ ≠ 5 = f({a=2,b=3}) ⊓ f({a=3,b=2}).

**Sparse Conditional Constant Propagation (SCCP).** Wegman and Zadeck [Wegman & Zadeck 1991] showed that exploiting SSA form and conditional branch information simultaneously yields a more precise algorithm. SCCP propagates constants along *executable* edges of the SSA graph:

1. Initialise all SSA values to ⊤ (unknown). Mark all CFG edges as *not yet executable*. Place the entry block on the CFG worklist.
2. **CFG worklist processing**: When block BB becomes executable (its incoming edge is marked executable), evaluate its φ-functions and instructions. For each φ-function `v = φ(v₁:P₁, ..., vₙ:Pₙ)`, only consider operands vᵢ from executable edges Pᵢ→BB; treat operands from non-executable predecessors as ⊤.
3. **SSA use-def worklist**: When the lattice value of an SSA value v changes, re-evaluate all instructions that use v.
4. **Conditional branches**: If the condition evaluates to a constant, mark only the taken target edge as executable; the non-taken edge remains non-executable.

The key insight: if block B is unreachable, all φ-function inputs from B contribute ⊤ to their targets. This allows a constant value that flows into a φ-function from the only *reachable* predecessor to propagate unchanged — precisely the case that the naïve per-block MFP computation misses.

**SCCP example.** Consider:

```llvm
  entry:
    br i1 true, label %then, label %else

  then:
    %x = add i32 1, 0          ; x = 1
    br label %merge

  else:                         ; unreachable
    %y = add i32 2, 0          ; y = 2
    br label %merge

  merge:
    %v = phi i32 [ %x, %then ], [ %y, %else ]
    ; naïve MFP: v = 1 ⊓ 2 = ⊥ (non-constant)
    ; SCCP: else is unreachable, so φ ignores %y → v = 1
```

SCCP propagates `true` in entry, marks only `%then` as executable, and the φ-function in `%merge` receives only `%x = 1` from the live edge, giving `%v = 1` (a constant). The naïve iterative algorithm would compute `1 ⊓ 2 = ⊥`. This achieves a complexity of O(E + V) where E is the number of SSA edges and V is the number of SSA values [Wegman & Zadeck 1991]. LLVM implements SCCP in `llvm/lib/Transforms/Scalar/SCCP.cpp` and the interprocedural variant in `llvm/lib/Transforms/IPO/SCCP.cpp`.

---

## 10.7 Abstract Interpretation: Widening, Narrowing, and Galois Connections

### 10.7.1 The Problem of Infinite-Height Lattices

Interval analysis — determining that a variable `i` satisfies `0 ≤ i ≤ 99` at each point in a loop — requires the interval domain ℤ_∞ × ℤ_∞ = [−∞, +∞] × [−∞, +∞], which has infinite height. The ascending chain:

```
  [0,0] ⊑ [0,1] ⊑ [0,2] ⊑ [0,3] ⊑ ...
```

does not terminate. Kleene iteration diverges. We need a mechanism to force convergence while preserving soundness.

### 10.7.2 Abstract Interpretation and Galois Connections

Cousot and Cousot [Cousot & Cousot 1977] placed dataflow analysis on a rigorous semantic foundation by introducing **abstract interpretation**. The key concepts are:

**Concrete domain (C, ≤_C)**: the lattice of "exact" program states (e.g., sets of concrete memory configurations, or sets of integers a variable might hold).

**Abstract domain (A, ≤_A)**: a simpler lattice that approximates the concrete domain (e.g., the interval lattice).

**Galois connection**: a pair of monotone functions α : C → A (abstraction) and γ : A → C (concretisation) such that for all c ∈ C and a ∈ A:

```
  α(c) ≤_A a   ⟺   c ≤_C γ(a)
```

This adjunction condition means: "the abstract value a is a sound over-approximation of the concrete value c if and only if c is contained in the concretisation of a." Equivalently, γ ∘ α ≥ id_C (abstraction loses information) and α ∘ γ ≤ id_A (concretisation followed by re-abstraction cannot be less abstract than we started).

**Soundness condition for abstract transfer functions.** An abstract transfer function f# : A → A is a *sound* approximation of the concrete function f : C → C if:

```
  α(f(γ(a))) ≤_A f#(a)   for all a ∈ A
```

or equivalently, γ(f#(a)) ⊇_C f(γ(a)) — the concretisation of the abstract result contains all concrete results. This ensures that any concrete behaviour is captured by the abstract analysis.

**The interval domain.** The interval lattice represents a variable's value as [l, u] where l ∈ ℤ∪{−∞} and u ∈ ℤ∪{+∞} with l ≤ u. The lattice ordering is:

```
  [l₁, u₁] ≤ [l₂, u₂]  ⟺  l₂ ≤ l₁  ∧  u₁ ≤ u₂
```

(A narrower interval is more precise, so it is lower in the lattice — less information lost.) Join: [l₁,u₁] ⊔ [l₂,u₂] = [min(l₁,l₂), max(u₁,u₂)]. Meet: [l₁,u₁] ⊓ [l₂,u₂] = [max(l₁,l₂), min(u₁,u₂)].

Abstract arithmetic:
- [l₁,u₁] + [l₂,u₂] = [l₁+l₂, u₁+u₂]
- [l₁,u₁] × [l₂,u₂] = [min(l₁u₁,l₁u₂,l₂u₁,u₁u₂), max(l₁u₁,l₁u₂,l₂u₁,u₁u₂)]
- Division requires special handling for intervals crossing zero.

### 10.7.3 Widening

The **widening operator** ∇ : A × A → A is an acceleration operator that overapproximates the join while forcing termination [Cousot & Cousot 1977]:

1. **Upper bound**: a ∇ b ≥_A a ⊔ b (widening is at least as large as join).
2. **Termination**: every ascending chain a₀ ∇ a₁ ∇ a₂ ∇ ... eventually stabilises.

**Interval widening.** The standard interval widening checks, for each bound, whether it has changed direction and, if so, jumps to ±∞:

```
  [l₁, u₁] ∇ [l₂, u₂]  =
    [ (l₂ < l₁) ? −∞ : l₁,
      (u₂ > u₁) ? +∞ : u₁ ]
```

**Worked example.** Consider the loop:

```c
  i = 0;
  while (i < N) {
      // loop body: i = i + 1
  }
```

Starting with the abstract value i = [0, 0]:

- After 0 iterations: i = [0, 0]
- After 1 unrolling: join with [1, 1] → [0, 1]
- After 2 unrollings: join with [2, 2] → [0, 2]
- ... this diverges.

Apply widening:
- Iteration 1: state = [0, 0]
- Iteration 2: join with back-edge [1, 1]: [0, 0] ∇ [0, 1]. l₂ = 0 = l₁ (no change), u₂ = 1 > u₁ = 0 → u = +∞. Result: [0, +∞]
- Iteration 3: join with back-edge [1, +∞]: [0, +∞] ∇ [0, +∞]. No change. **Fixed point reached: i ∈ [0, +∞].**

This is sound (every actual value of i during execution is indeed ≥ 0) but imprecise (we lost the upper bound N−1).

### 10.7.4 Narrowing

After widening, the fixed point a_ω = ⊥ ∇ f(⊥) ∇ f²(⊥) ∇ ... is sound but coarse. The **narrowing operator** △ : A × A → A refines this overapproximation:

1. **Soundness**: a △ b ≤_A a (narrowing only decreases, never introduces new concrete states into γ(a)).
2. **Termination**: every descending chain a₀ △ a₁ △ a₂ △ ... eventually stabilises (DCC on the abstract domain).

The narrowing iteration starts from a_ω and applies:

```
  a_{k+1} = a_k △ f(a_k)
```

converging to a **post-fixed point** a* ≤ a_ω. The result a* is still sound (every concrete state satisfies γ(a*)) and is more precise than a_ω.

**Interval narrowing.** The standard interval narrowing replaces ±∞ bounds with concrete values where available, but does not move non-infinite bounds:

```
  [l₁, u₁] △ [l₂, u₂]  =
    [ (l₁ = −∞) ? l₂ : l₁,
      (u₁ = +∞) ? u₂ : u₁ ]
```

**Continuing the example.** After widening we have i ∈ [0, +∞]. The loop guard `i < N` implies i ∈ [0, N−1] inside the loop. Narrowing with the guard constraint:

- [0, +∞] △ [0, N−1] = [0, N−1] (since +∞ is replaced by N−1).

The narrowed result i ∈ [0, N−1] is the precise invariant. In production, the narrowing pass is typically applied only inside the loop, using the guard condition as additional information via the abstract transfer function for conditionals.

### 10.7.5 The Interval Domain in Detail

The interval lattice's abstract arithmetic requires careful handling of edge cases:

**Abstract multiplication.** Because intervals can straddle zero, multiplication requires computing all four products of the endpoints:

```
  [l₁,u₁] × [l₂,u₂]  = [min(l₁u₂, l₁u₁, u₁l₂, u₁u₂),
                          max(l₁u₂, l₁u₁, u₁l₂, u₁u₂)]
```

For example: [-2, 3] × [-1, 4] = [min(8,-8,-3,-3,12), max(8,-8,-3,12)] = [-8, 12].

**Abstract division.** Division by an interval containing zero requires splitting: [l₁,u₁] / [l₂,u₂] where 0 ∈ [l₂,u₂] is handled conservatively as [−∞,+∞], or precisely by splitting [l₂,u₂] into [l₂,−1] ∪ [1,u₂] and computing separately.

**Conditional constraints (guard analysis).** After a conditional test `x < c`, the interval for x on the taken branch is refined: if x ∈ [l, u], then after `x < c` the interval becomes [l, min(u, c−1)]. This is the abstract narrowing of a conditional guard [Cousot & Cousot 1977]. LLVM's `LazyValueInfo` computes exactly these refined intervals at branch targets using a form of forward propagation through the dominator tree combined with backward guard analysis.

**The zone and octagon domains.** The interval domain is sometimes called a *unary* abstract domain because it tracks each variable independently. For relational information (e.g., `x − y ≤ 5`, `x + y ≤ 10`), richer domains are used:
- The **difference-bound matrices (DBM) / zones domain**: tracks constraints of the form x − y ≤ c.
- The **octagon domain** [Miné 2006]: tracks constraints of the form ±x ± y ≤ c. Efficient algorithms (O(n³)) exist for both, making them practical for small variable counts.

These richer domains are used in specialised static analyses (e.g., timing analysis, security analyses) but are generally not incorporated into production compiler optimisers due to their O(n²) space and O(n³) time requirements.

### 10.7.6 Connection to LLVM

LLVM implements several forms of abstract interpretation in production:

- **LazyValueInfo** (`llvm/lib/Analysis/LazyValueInfo.cpp`): an on-demand interval/constant range analysis using `ConstantRange` objects (Chapter 61). It computes intersections of known value ranges at conditionals, powered by a form of narrowing at branch targets.
- **ScalarEvolution** (Chapter 61): analyses induction variables using a symbolic representation based on chains of recurrences, which is a form of abstract interpretation over a symbolic polynomial expression domain.
- **ConstantRange** in `llvm/include/llvm/IR/ConstantRange.h`: LLVM's concrete interval domain, supporting the arithmetic operations and widening/narrowing operations needed by LazyValueInfo and the SCCP pass.
- **KnownBits** (`llvm/include/llvm/Support/KnownBits.h`): a bitwise abstract domain that tracks which bits of an integer value are statically known to be 0 or 1. This is a product of 2n two-point lattices (one per bit). The transfer functions for each operation (AND, OR, XOR, shift, add) preserve the known-bit information; LLVM's `ValueTracking` uses this to eliminate branches and simplify expressions at O(depth) cost per value.
- **MemorySSA** (`llvm/lib/Analysis/MemorySSA.cpp`): applies SSA form to memory operations, creating `MemoryDef`, `MemoryUse`, and `MemoryPhi` nodes. This is effectively an abstract interpretation of memory state using a flat lattice of memory versions, enabling dead store elimination and load-forwarding without explicit available-expressions iteration.

---

## 10.8 Interprocedural Extensions

### 10.8.1 The Interprocedural Challenge

Intraprocedural analyses treat function calls conservatively: a call to an unknown function may read and write any global variable or any memory reachable through any pointer parameter. This **whole-function conservative assumption** is sound but imprecise. For a call-intensive program (e.g., C++ with heavy use of small methods), intraprocedural analysis loses almost all precision at call boundaries. Interprocedural analysis attempts to propagate dataflow facts across call boundaries [Muchnick §19, EaC §9.6].

The central difficulty is **context sensitivity**: the same function may be called from many different call sites, with different dataflow facts at each call. A context-insensitive analysis uses a single summary for the function, applying it regardless of the call site — often too imprecise. A fully context-sensitive analysis tracks a separate state for every possible call sequence — precise but potentially intractable.

### 10.8.2 Context-Insensitive Analysis (k=0 CFA)

The simplest approach collapses all call sites: compute a single fixed-point over the entire call graph, treating each function as a single large basic block with a known summary [Muchnick §19.1]. For reaching definitions, this means that a definition anywhere in any callee "reaches" every call site. This is safe but extremely imprecise for large programs.

### 10.8.3 Call Strings and k-CFA

**Call strings** [Sharir & Pnueli 1981, Shivers 1991] distinguish call contexts by the sequence of active call sites. A **k-call-string** is the sequence of the k most recent call sites on the call stack. The **k-CFA** (k-th order control-flow analysis) maintains separate dataflow states for each k-call-string [Shivers 1991]:

- **k=0 (context-insensitive)**: one state per function; ignores call context.
- **k=1 (1-CFA)**: separate state per (function, calling_site) pair; distinguishes calls from different direct callers.
- **k=2**: separate state per (function, caller, caller_of_caller); further distinguishes indirect call chains.

**Precision vs. cost.** The number of distinct call strings grows as O(call_sites^k), making large k intractable. For first-order languages (no function pointers or virtual dispatch), 1-CFA is typically sufficient to recover most precision losses. For higher-order languages (function pointers, closures, virtual calls), Shivers showed [Shivers 1991] that even 0-CFA is undecidable in the limit; k-CFA for k ≥ 1 is EXPTIME-complete for higher-order programs [Van Horn & Mairson 2010].

**m-CFA.** A variant uses allocation-site abstraction instead of call-string abstraction: heap-allocated objects are identified by their allocation site (the call to `malloc` or `new`), giving a bounded representation of the heap. This is particularly relevant for pointer analysis (Andersen's points-to analysis [Andersen 1994]).

### 10.8.4 Summary-Based Analysis (Functional Approach)

The **summary-based (functional) approach** [Sharir & Pnueli 1981, Reps et al. 1995] computes for each function a **transfer function summary** that maps the dataflow state at the function's entry to the state at its exit — then *composes* summaries at call sites rather than re-analysing the callee.

```
  summary(f) : L → L   where L is the dataflow lattice

  At call site (BB contains call to f):
    OUT[BB] = summary(f)(IN[BB])
```

The summary is computed by analysing the function body with a symbolic/parameterised entry state. For monotone frameworks with finite-height lattices, the summary can be pre-computed once for each function (bottom-up in the call graph) and applied at every call site (top-down). This reduces the analysis complexity from O(program_size × context_size) to O(∑_f (function_size × summary_domain_size)).

**IPSCCP in LLVM.** LLVM's interprocedural SCCP pass (`llvm/lib/Transforms/IPO/SCCP.cpp`) implements a summary-based interprocedural constant propagation. It propagates constant argument values into callees and constant return values back to callers, using the call graph as a worklist. When a constant argument is discovered, the callee is re-analysed with the concrete argument value, which may unlock further constant propagation within the callee (Chapter 65).

### 10.8.5 The IFDS Framework: Interprocedural Finite Distributive Subset Problems

Reps, Horwitz, and Sagiv [Reps et al. 1995] formalised a class of interprocedural analyses called IFDS (**I**nterprocedural **F**inite **D**istributive **S**ubset) problems. An analysis is an IFDS problem if:

1. The domain D is a finite set of facts (instances of a powerset lattice).
2. Transfer functions are **distributive** (gen/kill form: they distribute over the ∪ operator).
3. The analysis must account for call and return edges in addition to ordinary CFG edges.

IFDS reduces to a **graph reachability problem** on an exploded supergraph G# = (N#, E#):

- Nodes of G# are pairs (n, d) where n is a program statement and d ∈ D ∪ {0̃} (0̃ is a special "zero" fact representing the empty set element).
- Edges encode how facts flow through each statement: for each transfer function f with f(S) = gen ∪ (S \ kill), add an edge (n, d) → (n', d) for d ∉ kill (the fact passes through), and (n, 0̃) → (n', d') for d' ∈ gen (the fact is generated).
- At call nodes, add edges modelling parameter binding; at return nodes, add edges modelling return value and callee side effects on globals.

The set of facts reachable from (entry, 0̃) in G# is exactly the MOP solution of the IFDS problem. Using efficient reachability algorithms (BFS or DFS with memoisation), IFDS runs in O(E# × |D|³) time, polynomial in the size of the program and the domain.

**Example IFDS problems:** reaching definitions, liveness, available expressions, possibly-uninitialised variables, taint analysis (tracking which variables are derived from untrusted input). The last two are particularly important in security-oriented static analysis tools (Clang's static analyser implements taint analysis using a variant of IFDS).

**IDE (Interprocedural Distributive Environments).** The IDE framework [Sagiv et al. 1996] generalises IFDS to environments: each fact d has an associated lattice value (not just a presence/absence bit). IDE reduces to **weighted graph reachability**, allowing analyses like linear constant propagation and copy propagation to be solved interprocedurally in polynomial time.

### 10.8.6 Demand-Driven and Compositional Analysis

Rather than computing facts for the entire program, **demand-driven analysis** [Horwitz et al. 1995] computes only the facts needed to answer a specific query. The computation proceeds backwards from the query point, traversing only the relevant slices of the program dependence graph. This is significantly faster for sparse queries (e.g., "is this specific use reached by a tainted definition?") than whole-program IFDS.

**Program dependence graphs (PDGs).** Ferrante, Ottenstein, and Warren [Ferrante et al. 1987] introduced the PDG as a graph combining control dependence and data dependence edges. Dataflow analysis on the PDG is equivalent to analysis on the CFG but can be done more efficiently for sparse problems. LLVM uses a related structure, the **Memory Dependence Analysis** (Chapter 61), which computes def-use and use-def chains for memory operations on demand.

**LLVM's inliner as implicit context sensitivity.** Rather than implementing full k-CFA, LLVM uses **inlining** as its primary mechanism for achieving context-sensitive analysis. By inlining a callee at a call site, LLVM creates a separate copy of the callee's code in the caller's context — effectively achieving k=1 or deeper context sensitivity at the cost of code growth. The inliner's heuristics (cost model, threshold) govern this tradeoff explicitly (Chapter 65). Post-inlining, every analysis (SCCP, GVN, loop analysis) operates on the expanded inline code without needing to understand call context.

**The attribute inference model.** LLVM's function attribute inference pass (`llvm/lib/Transforms/IPO/FunctionAttrs.cpp`) is a simplified form of interprocedural dataflow over the call graph. For each function, it infers attributes such as `readonly`, `readnone`, `noalias` on return values, and `nocapture` on pointer parameters. These attributes are propagated up the call graph (callees that don't write memory enable their callers to be marked `readonly`) and cached as function attributes, serving as a compact summary that obviates the need to re-analyse the callee at each call site.

---

## 10.9 Connection to LLVM's Analysis Infrastructure

### 10.9.1 The New Pass Manager and Analysis Results

LLVM's New Pass Manager (NPM, Chapter 59) manages the lifecycle of analysis results as first-class objects. An analysis pass (e.g., `DominatorTreeAnalysis`, `LivenessAnalysis`) implements the `AnalysisInfoMixin<>` interface and returns an immutable result object. Transformation passes request analysis results through `AnalysisManager::getResult<AnalysisT>(Function&)`, and the manager caches and invalidates results automatically.

The key types:

```cpp
// Analysis pass: computes and returns the result
struct LiveVariablesAnalysis
    : AnalysisInfoMixin<LiveVariablesAnalysis> {
  using Result = LiveVariables;
  Result run(Function &F, FunctionAnalysisManager &FAM);
};

// How a transform requests liveness:
auto &LV = FAM.getResult<LiveVariablesAnalysis>(F);
```

This corresponds exactly to the lattice-theoretic model: `LiveVariables` is the fixed-point solution of the liveness equations, computed on demand and cached until the CFG is modified.

### 10.9.2 Key Analysis Implementations

**LiveVariables** (`llvm/lib/CodeGen/LiveVariables.cpp`, also `llvm/include/llvm/CodeGen/LiveVariables.h`): computes exact liveness at the MachineIR level using a backward worklist pass. It is the direct instantiation of the backward liveness analysis from Section 10.6.1.

**ReachingDefAnalysis** (`llvm/lib/CodeGen/ReachingDefAnalysis.cpp`): computes reaching definitions at the MachineIR level for register values.

**MemorySSA** (`llvm/lib/Analysis/MemorySSA.cpp`, Chapter 61): applies SSA form to memory operations, building MemoryDefs and MemoryUses connected by MemoryPhi nodes. The resulting structure allows available expressions analysis and dead store elimination to operate on memory without needing explicit kill sets.

**LazyValueInfo** (`llvm/lib/Analysis/LazyValueInfo.cpp`): implements an on-demand interval/constant-range analysis. Rather than computing fixpoint for the entire program, it answers specific queries ("what are the possible values of v at program point p?") using backward traversal through the dominator tree — a demand-driven variant of the interval analysis from Section 10.7.

**ScalarEvolution** (`llvm/lib/Analysis/ScalarEvolution.cpp`, Chapter 61): computes **chain-of-recurrences** (CoR) summaries for induction variables. A CoR {start, +, step} represents a variable taking values start, start+step, start+2×step, .... ScalarEvolution composes these over nested loops, answering questions like "what is the maximum trip count of this loop?" This is abstract interpretation over a symbolic polynomial domain, with widening implicit in the representation.

**IPSCCP** (`llvm/lib/Transforms/IPO/SCCP.cpp`, Chapter 65): implements interprocedural SCCP as described in Section 10.8.4. It processes the call graph bottom-up to discover constant return values and top-down to propagate constant arguments, converging to a fixed-point over the product of per-function flat constant lattices.

### 10.9.3 The AnalysisManager as a Lattice Fixed-Point Cache

The NPM's `AnalysisManager` maintains a mapping from (AnalysisKey, IR unit) to analysis results. When a transformation modifies the IR, it reports *preserved analyses* via an `AnalysisManager::Invalidator`. Preserved analyses are those whose fixed-point results are still valid despite the modification (i.e., the modification did not change the values they compute). Non-preserved analyses are evicted from the cache.

This cache-invalidation protocol mirrors the theoretical notion of a **monotone reduction**: a transformation that only removes information (e.g., dead code elimination removes a definition) preserves dataflow results that depend on an intersection/must semantics, because fewer definitions can only tighten or preserve the must-meet. Transformations that add information (e.g., inlining adds new definitions) generally invalidate most dataflow caches.

```cpp
// Example: preserving analyses in a transformation pass
PreservedAnalyses MyTransformPass::run(Function &F,
                                        FunctionAnalysisManager &FAM) {
    // ... perform transformation ...
    PreservedAnalyses PA;
    PA.preserve<DominatorTreeAnalysis>();     // CFG not changed
    PA.preserve<LoopAnalysis>();             // loop structure unchanged
    // LiveVariables, ReachingDefs invalidated (instructions removed)
    return PA;
}
```

### 10.9.4 Implementing a Custom Dataflow Analysis in LLVM

A custom dataflow analysis in LLVM follows the structure of the lattice framework. As a concrete example, consider implementing a simple "must-be-null" analysis that tracks which pointer variables are definitely null at each program point:

```cpp
// Lattice: ℘(Pointers) ordered by ⊆ (more pointers = more known null)
// Direction: forward, must-analysis (intersection at join points)
// Transfer: a pointer p is null after "p = null" instruction;
//           p is non-null after "p = malloc(...)" or dereferencing p

struct MustBeNull {
    SmallPtrSet<const Value *, 8> NullPtrs;

    // Meet: intersection (must be null on all paths)
    MustBeNull meet(const MustBeNull &Other) const {
        MustBeNull Result;
        for (const Value *V : NullPtrs)
            if (Other.NullPtrs.count(V))
                Result.NullPtrs.insert(V);
        return Result;
    }

    // Transfer function for a single instruction
    MustBeNull transfer(const Instruction &I) const {
        MustBeNull Result = *this;
        if (auto *SI = dyn_cast<StoreInst>(&I)) {
            // store null → pointer is null
            if (isa<ConstantPointerNull>(SI->getValueOperand()))
                Result.NullPtrs.insert(SI->getPointerOperand());
        } else if (auto *CI = dyn_cast<CallInst>(&I)) {
            // allocation call → result is non-null (optimistically)
            if (CI->getType()->isPointerTy())
                Result.NullPtrs.erase(CI);
        }
        return Result;
    }
};
```

The worklist algorithm then iterates this transfer function over the CFG's basic blocks, using meet at predecessors. The New Pass Manager wraps this in an `AnalysisInfoMixin<>` subclass with a `run(Function &, FunctionAnalysisManager &)` method.

### 10.9.5 Performance Characteristics of LLVM's Dataflow Analyses

Production experience with LLVM reveals that the theoretical worst-case bounds are rarely reached:

- **LiveVariables** for typical functions converges in 1–3 backward passes. The SSA property ensures that most variables have short live ranges (the use is dominated by the definition), so the backward propagation rarely propagates far.
- **SCCP** typically runs in time proportional to the number of SSA edges because the constant-propagation lattice is only three levels deep; once a value reaches ⊥ it never changes again, so each SSA edge is visited at most twice.
- **ScalarEvolution** uses lazy evaluation (chains of recurrences are constructed on demand) and memoisation (each SCEV expression is computed once and cached), making it effectively a single-pass over the loop nest structure rather than an iterative fixed-point.
- **LazyValueInfo** performs demand-driven backward analysis: it starts from the query point and traverses the dominator tree upward, using cached results for previously queried blocks. For conditional value ranges (the most common query pattern), it terminates in O(dom_depth) steps rather than O(n) for the whole function.

These performance characteristics reflect a general principle: the tighter the lattice and the more localised the information (short live ranges, shallow loop nests, few global values), the faster the analyses converge. SSA form's structural guarantee that definitions dominate uses is, at bottom, a guarantee that the dataflow equations for reaching definitions have a trivial fixed point — which is the deepest reason SSA accelerates the entire analysis infrastructure built on top of it.

---

## 10.10 Correctness, Precision, and the Soundness/Completeness Tradeoff

### 10.10.1 Soundness of Dataflow Analyses

A dataflow analysis is **sound** if every fact it asserts is a valid over-approximation of the corresponding concrete program property. Formally, if the analysis says "variable v is live at point p," then in every possible execution of the program, v *may* indeed be live at p (no concrete execution is excluded). Unsound optimisations — those that remove code that the analysis incorrectly declared dead — introduce miscompilation bugs. LLVM's philosophy is that analysis passes must be sound; transformations may introduce deliberate unsoundness only when explicitly approved (e.g., `nsw`/`nuw` flags mark arithmetic that may overflow — the programmer asserts no overflow, enabling optimisations that would otherwise be unsound).

**What soundness means in practice.** For liveness analysis, soundness means we may compute *more* live variables than are truly live — a conservative over-approximation. A variable incorrectly marked live wastes a register but produces correct code. A variable incorrectly marked dead (unsound) would cause its register to be reused, producing wrong values.

For available expressions, soundness means we may compute *fewer* available expressions than are truly available — the conservative under-approximation. An expression missed as available loses a CSE opportunity but produces correct code. An expression incorrectly marked available would replace a computation with a stale value, producing wrong results.

### 10.10.2 Completeness

A dataflow analysis is **complete** if it never misses a fact that is concrete. Completeness is generally unachievable for non-trivial properties of programs (Rice's theorem: any non-trivial semantic property of programs is undecidable). All practical analyses are incomplete — they may fail to find some live variables, available expressions, or constant values that are actually present.

The art of analysis design is the tradeoff: maximally conservative (sound but imprecise, missing many optimisation opportunities) versus maximally optimistic (precise but risking unsoundness). The MOP solution represents the ideal precise-and-sound point; the MFP solution is always at least as conservative (sound and at most as precise). For distributive frameworks, MFP = MOP.

### 10.10.3 Verification of Analysis Implementations

A practical concern: how do we verify that an analysis implementation is sound? LLVM uses several techniques:

1. **Algebraic testing**: test that the lattice operations satisfy the lattice axioms (idempotency, commutativity, associativity of ⊓ and ⊔; absorption laws).
2. **Soundness testing**: generate random programs, run the analysis, apply the optimisation enabled by the analysis, and check that the optimised program has the same behaviour as the original (using test harnesses or fuzzing).
3. **Formal verification**: tools like Alive2 (Chapter 170) can verify that specific optimisations are sound by checking that the transformation preserves program semantics for all inputs. Alive2 encodes the soundness condition as an SMT formula and checks it with an SMT solver.
4. **Differential testing**: run two analyses (e.g., with and without an approximation) and check that the more precise one never claims fewer facts than the coarser one — a monotonicity sanity check.

## 10.11 Chapter Summary

- **Partial orders and complete lattices** provide the mathematical substrate for dataflow analysis. The powerset lattice ℘(Vars), the flat lattice for constants, and the interval lattice are the three principal instances in compiler analyses.

- **Tarski's Fixed-Point Theorem** guarantees that every monotone function on a complete lattice has a least fixed point (lfp) and a greatest fixed point (gfp), which are computed as meets of post-fixed points and joins of pre-fixed points respectively.

- **Kleene's ascending chain theorem** shows that, under the ascending chain condition (finite-height lattice), the iterative sequence ⊥, f(⊥), f²(⊥), ... converges to lfp(f) in finitely many steps — giving the foundational justification for worklist algorithms.

- **Monotone frameworks** (Kildall 1973) guarantee termination and compute the MFP solution. **Distributive frameworks** (gen/kill analyses) additionally satisfy MFP = MOP, meaning the iterative algorithm computes the ideal answer. Non-distributive analyses (constant propagation) may yield MFP < MOP.

- **Gen/kill transfer functions** — f(x) = gen ∪ (x \ kill) — are always distributive, covering liveness (backward/may), available expressions (forward/must), reaching definitions (forward/may), and very busy expressions (backward/must).

- **Constant propagation** uses a three-level flat lattice (⊤, constants, ⊥). It is non-distributive but can be made significantly more precise by SCCP, which exploits SSA form and conditional branch information simultaneously.

- **Widening and narrowing** (Cousot & Cousot 1977) handle infinite-height lattices such as the interval domain. Widening forces convergence by jumping to ±∞ when a bound is seen to grow; narrowing subsequently recovers precision by using concrete constraints.

- **Abstract interpretation** provides a rigorous semantic foundation through Galois connections (α, γ) between concrete and abstract domains. Soundness requires γ(f#(a)) ⊇ f(γ(a)): the abstract result over-approximates the concrete result.

- **Interprocedural extensions** — call strings, k-CFA, and summary-based analysis — trade precision for scalability. LLVM implements inlining as de facto context sensitivity and IPSCCP for interprocedural constant propagation; the IFDS framework handles distributive analyses efficiently via supergraph reachability.

- **LLVM's analysis infrastructure** (LiveVariables, MemorySSA, LazyValueInfo, ScalarEvolution, IPSCCP) is a direct realisation of the lattice framework, managed by the New Pass Manager's AnalysisManager which caches fixed-point results and invalidates them when the IR changes.

---

## References

- [Kildall 1973] G. A. Kildall, "A unified approach to global program optimization," *POPL 1973*, pp. 194–206.
- [Dragon §9] A. Aho, M. Lam, R. Sethi, J. Ullman, *Compilers: Principles, Techniques, and Tools*, 2nd ed., Addison-Wesley, 2007, Chapter 9.
- [EaC §8, §9] K. Cooper, L. Torczon, *Engineering a Compiler*, 3rd ed., Morgan Kaufmann, 2023, Chapters 8–9.
- [Cousot & Cousot 1977] P. Cousot, R. Cousot, "Abstract interpretation: A unified lattice model for static analysis of programs by construction or approximation of fixpoints," *POPL 1977*, pp. 238–252.
- [Muchnick §7–8] S. Muchnick, *Advanced Compiler Design and Implementation*, Morgan Kaufmann, 1997, Chapters 7–8.
- [Tarski 1955] A. Tarski, "A lattice-theoretical fixpoint theorem and its applications," *Pacific Journal of Mathematics* 5(2), 1955.
- [Kam & Ullman 1976] J. Kam, J. Ullman, "Global data flow analysis and iterative algorithms," *JACM* 23(1), 1976.
- [Sharir & Pnueli 1981] M. Sharir, A. Pnueli, "Two approaches to interprocedural data flow analysis," *Program Flow Analysis: Theory and Applications*, 1981.
- [Shivers 1991] O. Shivers, *Control-Flow Analysis of Higher-Order Languages*, PhD thesis, Carnegie Mellon, 1991.
- [Reps et al. 1995] T. Reps, S. Horwitz, M. Sagiv, "Precise interprocedural dataflow analysis via graph reachability," *POPL 1995*.
- [Wegman & Zadeck 1991] M. Wegman, F. Zadeck, "Constant propagation with conditional branches," *TOPLAS* 13(2), 1991.
- [Andersen 1994] L. Andersen, *Program Analysis and Specialization for the C Programming Language*, PhD thesis, DIKU, 1994.
- [Rosen 1980] B. K. Rosen, "Linear cost is sometimes quadratic," *POPL 1980*.
- [Van Horn & Mairson 2010] D. Van Horn, H. Mairson, "Deciding k-CFA is complete for EXPTIME," *ICFP 2010*.
- [Sagiv et al. 1996] M. Sagiv, T. Reps, S. Horwitz, "Precise interprocedural dataflow analysis with applications to constant propagation," *TCS* 167(1–2), 1996.
- [Ferrante et al. 1987] J. Ferrante, K. Ottenstein, J. Warren, "The program dependence graph and its use in optimization," *TOPLAS* 9(3), 1987.
- [Knoop et al. 1992] J. Knoop, O. Rüthing, B. Steffen, "Lazy code motion," *PLDI 1992*, pp. 224–234.
- [Miné 2006] A. Miné, "The octagon abstract domain," *HOSC* 19(1), 2006.


---

@copyright jreuben11
