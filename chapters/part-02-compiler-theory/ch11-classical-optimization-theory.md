# Chapter 11 — Classical Optimization Theory

*Part II — Compiler Theory and Foundations*

Chapter 10 established the lattice framework for dataflow analysis: how to propagate facts about program state through a control-flow graph to fixed point, and why the monotone-framework guarantees convergence. With that machinery in hand, this chapter turns from *analysis* to *transformation* — the algorithms that use dataflow facts to make programs run faster. Classical optimization theory is a deep and beautiful field; the algorithms presented here were developed over four decades and remain the theoretical bedrock of every production compiler, including LLVM.

The chapter is organized as follows. Section 11.1 situates optimizations along the local–global–interprocedural axis and explains why pipeline order matters. Sections 11.2 through 11.5 cover the scalar value-transformation optimizations: common subexpression elimination via value numbering, loop-invariant code motion, strength reduction, and partial redundancy elimination. Section 11.6 develops register allocation theory in depth — graph coloring (Chaitin, Chaitin-Briggs, George-Appel), linear scan, and PBQP. Section 11.7 covers instruction scheduling. Section 11.8 surveys garbage collection algorithms. Section 11.9 maps every algorithm to its LLVM implementation. A summary closes the chapter.

Throughout, forward references connect to the LLVM practice chapters: Chapter 62 (LLVM's scalar optimization pipeline), Chapter 63 (loop transformations), Chapter 89 (the backend lowering pipeline), Chapter 90 (register allocation in practice), and Chapter 91 (software pipelining and the MachinePipeliner).

---

## 11.1 Local vs. Global vs. Interprocedural: The Optimization Scope

### 11.1.1 Local optimizations

A **local optimization** operates within a single basic block — a maximal straight-line sequence of instructions with no branches in or out (except at the entry and exit). Because control flow cannot divert computation within a basic block, the optimizer has perfect knowledge of which instructions execute and in what order. This simplicity makes local optimizations both the fastest and the simplest category.

Classic local optimizations include:

- **Constant folding**: replace `3 + 4` with `7` at compile time. This subsumes arithmetic simplifications, boolean reductions, and compile-time evaluation of standard library functions on constant arguments. LLVM's `ConstantFoldInstruction` performs this during IR construction.
- **Algebraic simplification**: exploit identities such as `x * 1 → x`, `x + 0 → x`, `x - x → 0`, `x & ~x → 0`, `x / x → 1` (with appropriate care for floating-point semantics and zero divisors).
- **Constant propagation**: if an earlier instruction in the block assigns `x = 5`, substitute `5` for every subsequent use of `x` within the block.
- **Dead code elimination**: if an instruction's result is never used within the block and it has no side effects, delete it.
- **Peephole optimization**: match small instruction sequences and replace them with more efficient equivalents, e.g., replacing `SHIFT LEFT 3` with `MULTIPLY IMMEDIATE 8` on an architecture that makes the latter cheaper, or eliminating a `STORE` immediately followed by a `LOAD` from the same address.

Local optimizations require no dataflow information: they are single-pass (or at most a few-pass) algorithms that scan instruction sequences linearly. Their worst-case complexity is O(n) per block, and they compose cleanly [Dragon §8.5].

### 11.1.2 Global optimizations

A **global optimization** operates across basic blocks within a single function. Control flow makes global optimizations harder: a value computed on one path may or may not reach a given use site, and the optimizer must use dataflow analysis (Chapter 10) to reason about what holds at each program point.

Most of the optimizations studied in this chapter — CSE/GVN, LICM, strength reduction, PRE — are global in scope. They require the CFG, dominator tree, and one or more dataflow lattices (reaching definitions, available expressions, liveness). The theoretical complexity is typically O(n × d), where n is the number of instructions and d is the depth of the dominator tree (often small in practice) [Cooper/Torczon §8.1].

### 11.1.3 Interprocedural optimizations

An **interprocedural optimization** (IPO) crosses function call boundaries. The simplest form is **inlining**: replace a call site with a copy of the callee's body, after which all intra-procedural optimizations apply to the merged function. More sophisticated IPOs include:

- **Interprocedural constant propagation** (IPCP): propagate constants through call sites, specialized to the actual argument values.
- **Dead argument elimination**: remove parameters that are never read; remove return values that are always discarded.
- **Devirtualization**: replace virtual dispatch with a direct call when the receiver type is statically known.
- **Whole-program alias analysis**: build a points-to graph spanning all functions (Andersen's analysis, Steensgaard's analysis) [Muchnick §12.6].

IPO is expensive: the analysis must process the entire call graph, and its precision is bounded by the precision of the underlying alias and points-to analyses. LLVM's IPO passes (in `lib/Transforms/IPO/`) operate at link time via the LTO (link-time optimization) pipeline (Chapter 13).

**Inlining cost models**: the inliner is one of the highest-leverage IPO transformations. LLVM's inliner estimates the **benefit** of inlining as the number of instructions eliminated (constants propagated into the callee, branch conditions resolved, CSE across the call boundary) and the **cost** as the code-size growth. The inliner uses a threshold (configurable via `-inline-threshold`, default 225 for `-O2`) and an **inline cost estimator** that simulates simplifications to estimate net code growth [Cooper/Torczon §15.2].

After inlining, virtually every other optimization benefits: constant propagation across the call boundary, devirtualization (the dynamic dispatch becomes a direct call once the callee is inlined and the receiver type is visible), and dead code elimination (branches on the callee's return value, which is now a known constant). This is why the inliner runs early in the LLVM pipeline and may run multiple times (the main inliner pass, the always-inliner, and a late inliner for remaining small callees).

### 11.1.4 Why pipeline order matters

Individual optimizations are not independent: each creates opportunities for others. Constant folding simplifies branches, exposing unreachable blocks to dead code elimination. Dead code elimination removes assignments, making more values dead and enabling further folding. LICM hoists expressions out of loops, exposing those expressions to inter-loop CSE. Inlining expands call sites, enabling IPCP, which in turn enables branch folding and more inlining.

The optimization pipeline must therefore be ordered thoughtfully. LLVM's standard `-O2` pipeline runs simplified passes (SROA, early CSE) first to clean up the IR before running expensive analyses (alias analysis, loop analyses); it then runs the heavier loop passes (LoopRotate, LICM, IndVarSimplify, LoopUnroll) and finishes with global cleanups (GVN-PRE, ADCE, InstCombine). Some passes must iterate to fixed point: LICM, for instance, runs until no more expressions can be hoisted. The driver applies each pass until convergence, bounded by a pass limit [Cooper/Torczon §9.1].

### 11.1.5 The fixed-point iteration approach

Some pairs of optimizations require iteration to mutual convergence. The classic example is the **constant propagation + dead code elimination** combination: constant propagation may simplify a conditional branch, making one successor unreachable; dead code elimination then removes the unreachable block; this may expose more constants (the branch condition had a live use on the now-deleted path), enabling further constant propagation. The canonical combined algorithm is **Sparse Conditional Constant Propagation** (SCCP) [Wegman-Zadeck 1991], which interleaves the two analyses on an SSA-form program:

```
Algorithm: SCCP
Initialize: all SSA values as "undetermined" (⊥ in the two-point lattice);
            entry block is "executable"; all other blocks "not executable"
Worklists: FlowWorklist (blocks to execute), SSAWorklist (values whose
           lattice value changed)

repeat:
    if FlowWorklist non-empty:
        B ← pop from FlowWorklist
        for each φ-function v = φ(a₁, ..., aₙ) in B:
            new_val ← meet over {lattice(aᵢ) | pred edge i is executable}
            if new_val ≠ lattice(v): lattice(v) ← new_val; add v to SSAWorklist
        for each non-φ instruction I in B:
            evaluate I symbolically on lattice values of operands
            if result changed: update lattice(result); add result to SSAWorklist
        mark all successors of B executable (or only the taken branch for constants)
    if SSAWorklist non-empty:
        v ← pop from SSAWorklist
        for each use u of v: re-evaluate u
until both worklists empty
```

SCCP achieves the precision of executing every program path symbolically, in O(n) time (each SSA value is inserted into the worklist at most twice — once when it goes from ⊥ to a constant, once when it goes from a constant to ⊤). It is strictly more powerful than the sequential application of the two passes: SCCP discovers constants that only become apparent after unreachable code is eliminated, which in turn requires knowing the constants first [Muchnick §12.8].

The general principle is that **phase ordering** is a fundamental challenge in compiler design. The optimal ordering is program-dependent; the standard approach is to use a fixed pipeline with a few strategic iteration points rather than true global convergence, which would be too expensive.

---

## 11.2 Common Subexpression Elimination: Value Numbering and GVN

### 11.2.1 Local value numbering

**Local value numbering** (LVN) [Dragon §8.4] assigns an abstract **value number** to every expression in a basic block such that two expressions receive the same number if and only if they are guaranteed to compute the same value. The algorithm maintains a hash table mapping (operator, VN(operand1), VN(operand2)) tuples to value numbers.

```
Algorithm: Local Value Numbering
Input: Basic block B = [i₁, i₂, ..., iₙ]
Output: Value number map VN; replaces redundant computations

Initialize VN(c) = c for each constant c
Initialize VN(vᵢ) = fresh number for each live-in variable vᵢ

for each instruction iₖ: vₖ ← op(aₖ, bₖ) in B:
    key ← (op, VN(aₖ), VN(bₖ))         // canonicalize commutative ops
    if key ∈ table:
        VN(vₖ) ← table[key]             // redundant: reuse
        replace iₖ with copy vₖ ← table.varOf[key]
    else:
        VN(vₖ) ← fresh()
        table[key] ← VN(vₖ);  table.varOf[key] ← vₖ
    if iₖ is assignment vₖ ← c (constant):
        VN(vₖ) ← c                       // constant folding
```

The key subtlety is **kills**: if instruction `i` assigns to variable `x`, and a later instruction `j` also assigns to `x`, then `x` after `j` is a different value from `x` after `i`. LVN handles this by assigning a new value number to `x` at the second assignment, logically renaming the second version.

LVN runs in O(n) time per block (with a hash table). It discovers all local redundancies, but cannot see across block boundaries.

**Worked LVN example**: consider the following basic block, where letters represent variables:

```
1:  a = x + y
2:  b = x + y     // same as 1: a and b get same VN
3:  c = a * z
4:  x = b + z     // x reassigned: fresh VN for x
5:  d = x + y     // x now has a new VN; a+y would match differently
6:  e = a + z     // same as (VN(a), *, VN(z)) — same as c; c and e get same VN
```

After LVN: instruction 2 is replaced by `b = a` (copy); instruction 6 is replaced by `e = c` (copy). The copies can then be propagated away. Note that instruction 5 uses the *new* x from line 4, so it is **not** redundant with instruction 1.

### 11.2.2 Global value numbering in SSA form

**Global value numbering** (GVN) [Muchnick §12.3] extends LVN to entire functions. The key insight — due to Rosen, Wegman, and Zadeck [RWZ 1988] — is that **SSA form** (Chapter 9) makes the extension almost trivial. In SSA, each variable has exactly one static definition, so two uses of the same SSA value trivially have the same value number. GVN must only determine whether two distinct SSA *definitions* produce the same value.

The GVN algorithm traverses the dominator tree in pre-order (so a dominator is always processed before its dominatees). At each block, it hashes definitions into the value-number table; when a definition matches an earlier entry, the new definition is replaced by a copy of the earlier one. The table is maintained as a scoped structure: entries added while processing a block are removed when the block's scope is exited (maintaining the dominator-tree invariant).

**φ-functions** are the critical complication. A φ-function `v₃ = φ(v₁, v₂)` defines a new value that is either `v₁` or `v₂` depending on which predecessor was taken. Two φ-functions receive the same value number if and only if their operands have the same value numbers (after canonicalization of operand order for commutative φ's) [Cooper/Torczon §10.3]. This congruence-based view of value numbering is called **hash-consing of expressions**.

### 11.2.3 SuperGVN and LLVM's NewGVN

LLVM implements GVN in two passes:

- **`GVN`** (in `lib/Transforms/Scalar/GVN.cpp`): an RPO-based algorithm that also handles **load-PRE** (hoisting loads speculatively to pre-headers when the value is not yet available on all paths). It uses LLVM's `MemorySSA` (Chapter 10) to determine when loads from the same address yield the same value.
- **`NewGVN`** (in `lib/Transforms/Scalar/NewGVN.cpp`): a more principled reimplementation by Novillo and Briggs, based on the VEQ (value-expression-equivalence) framework. NewGVN maintains a set of congruence classes, iteratively merging classes until a fixed point, then replaces all members of each class with its "leader." It handles φ-functions correctly in the face of loops by using the RPO-based worklist to ensure dependencies are processed before uses.

For memory operations (load-store CSE), GVN relies on alias analysis to prove that two loads from address `p` at program points `A` and `B` are not separated by any write to an aliased address. The MemorySSA framework provides this in O(1) per query via the `MemoryAccess` dominator tree structure.

**The RPO-based GVN algorithm in detail**: Rosen, Wegman, and Zadeck's original algorithm [RWZ 1988] processes the CFG in reverse postorder (RPO). RPO guarantees that a block's dominators are always processed before the block itself (for reducible CFGs). The algorithm maintains a scoped hash table: entering a block opens a new scope; exiting closes it. Any expression hashed in a block is visible to all descendants in the dominator tree but not to other branches. This perfectly captures the SSA invariant: a definition in block B dominates all uses in the subtree below B.

For irreducible CFGs (containing unstructured control flow), RPO is not sufficient. LLVM handles this by detecting irreducible SCCs and conservatively assigning fresh value numbers to back-edge φ-functions, then iterating to convergence — an uncommon but correct fallback.

The combination of GVN with **dead code elimination** and **simplification** (canonicalization of expressions using algebraic identities) is LLVM's `InstCombine` pass: it simultaneously performs constant folding, algebraic simplification, and local CSE within each instruction, running on every instruction in the function in one RPO pass, repeated until convergence.

---

## 11.3 Loop-Invariant Code Motion

### 11.3.1 Loop-invariant expressions

An expression `E = op(a, b)` is **loop-invariant** with respect to loop L if, for every iteration of L, `E` computes the same value. Formally, `E` is loop-invariant if:

1. Every operand of `E` is a constant, or
2. Every operand of `E` has all its reaching definitions outside L, or
3. Every operand of `E` is itself a loop-invariant expression in L.

This definition is recursive and is computed as a fixed point over the loop body [Dragon §8.7].

### 11.3.2 Code hoisting

**Code hoisting** (also called **loop-invariant code motion**, LICM) moves a loop-invariant computation from inside the loop to the **loop preheader** — a basic block that (a) is not part of the loop, (b) has the loop header as its sole successor, and (c) is dominated by all loop-entry paths. Every loop can be given a unique preheader by inserting a synthetic block if necessary.

After hoisting, the expression is computed once before the loop begins, and the loop body uses the precomputed value. For a loop executing N iterations, this saves N−1 executions of the expression.

**Correctness conditions** for safe hoisting [Cooper/Torczon §10.5]:

1. **Dominance of all loop exits**: the instruction to be hoisted must dominate every exit point of the loop. This ensures it will execute if the loop executes at all. An expression that occurs only on one path through the loop body — and that path might not be taken — cannot be safely hoisted (the computation might introduce a fault, e.g., division by zero, or have an observable side effect).
2. **No aliasing write inside the loop**: there must be no assignment inside the loop that could modify the expression's operands (for memory references, no aliasing store). This is determined by alias analysis.
3. **No exception**: the expression must not fault. A load from a pointer that might be null cannot be hoisted, because hoisting could introduce a fault on iterations that would otherwise have taken a branch avoiding the load.

When conditions 2 and 3 cannot be established statically, **speculative hoisting** can still be valid if a guard is inserted: the optimizer adds a conditional check before the loop (e.g., a null check before a load), and the load is hoisted under the guard.

### 11.3.3 The LICM algorithm

```
Algorithm: LICM
Input: Loop nest L (innermost first); CFG with dominator tree

for each loop L in post-order (innermost first):
    compute loop-invariant expressions for L
    for each invariant instruction I (in dominator-tree order):
        if I dominates all loop exits
           and I has no aliasing stores in L (per MemorySSA)
           and I has no exception side-effects:
            hoist I to preheader(L)
        elif can_insert_guard(I):
            insert guard; hoist I under guard
```

Processing in innermost-to-outermost order enables propagation: once an expression is hoisted out of an inner loop, it may become loop-invariant with respect to the outer loop and be hoisted again.

### 11.3.4 LLVM's LICM pass

LLVM's LICM pass lives at `llvm/lib/Transforms/Scalar/LICM.cpp`. It uses **MemorySSA** (built on top of SSA, Chapter 10) to reason about aliasing: each load or store is a `MemoryAccess` node in the MemorySSA DAG; a load can be proved loop-invariant if its `MemoryAccess` is dominated by a `MemDef` or `LiveOnEntry` that lies outside the loop. This avoids overly conservative alias analysis by tracking the memory-def chain precisely [Cooper/Torczon §10.6].

LICM also sinks stores: a store that is loop-invariant and always executed on loop exit can be moved to just after the loop, reducing stores within the loop body.

**Interaction with alias analysis**: the precision of LICM depends critically on alias analysis. Consider:

```c
for (int i = 0; i < N; i++) {
    float t = a[0] * scale;   // is this loop-invariant?
    b[i] = t * c[i];
}
```

The expression `a[0] * scale` is loop-invariant *only if* the store `b[i] = ...` does not alias `a[0]`. If `b` and `a` point into the same array (aliasing), then writing `b[i]` might change `a[0]` on the next iteration, making the expression not loop-invariant. LLVM's `BasicAliasAnalysis` can determine that if `b` and `a` are declared with `restrict` (no-alias annotation in LLVM IR: `noalias` attribute), then the store cannot alias `a[0]`, and the hoisting is valid. Without `restrict`, the conservative assumption (may-alias) blocks the hoist. This is one reason that `restrict`-annotated C code compiles significantly faster than unqualified pointer code.

The **MemorySSA** representation tracks memory effects in a precise chain: every load and store is a `MemoryAccess`; a `MemoryPhi` is inserted at join points in the CFG analogously to an SSA φ-function. LICM queries this chain: a load `L` in the loop body is loop-invariant if there is no `MemoryDef` reachable from `L` via a loop-back edge that may alias `L`'s address. The MemorySSA query is O(1) amortized per check due to the dominator structure [Novillo 2017].

---

## 11.4 Strength Reduction and Induction Variables

### 11.4.1 Induction variables and families

A variable `i` is a **basic induction variable** in loop L if, on every iteration, `i` is incremented (or decremented) by a constant. The canonical loop index `for (i = 0; i < N; i++)` defines `i` as a basic IV with initial value 0, step 1, and bound N.

A variable `j` is a **derived induction variable** in the same IV family if `j = f(i)` where f is a linear function: `j = c₁ * i + c₂`. For example, if `j = 4 * i + 8` inside the loop, then `j` is derived from `i` with coefficient 4 and offset 8. On each iteration, `j` increases by `4 * step(i)` = 4 [Dragon §8.11].

The **IV family** of basic IV `i` is the set of all derived IVs that are linear functions of `i`, plus `i` itself.

### 11.4.2 Strength reduction

**Strength reduction** replaces expensive derived-IV computations with cheaper ones. The classic example: if `j = 4 * i` and `i` increments by 1 each iteration, introduce a new variable `j' = 4 * i_init` before the loop and add `j' += 4` at the end of the loop body. The multiplication inside the loop is replaced by an addition, which is typically 1–5 cycles faster on most ISAs.

```
Before:
    i = 0
    while (i < N):
        use(a[4 * i])    // multiply per iteration
        i += 1

After strength reduction:
    i = 0; j = 0       // j = 4 * i
    while (i < N):
        use(a[j])       // j replaces 4*i
        i += 1; j += 4  // step j by 4
```

More generally, for a derived IV `j = c * i + d`:
- **Initialization**: before the loop, `j_new = c * init(i) + d`.
- **Update**: at the IV increment site, `j_new += c * step(i)`.
- **Replacement**: replace all uses of `c * i + d` with `j_new`.

### 11.4.3 Induction variable simplification

Once all derived IVs have been expressed as independent recurrences, the **linear function test replacement** (LFTR) can replace loop exit conditions involving derived IVs with conditions on the basic IV. For example, `j < 4 * N` (where `j = 4 * i`) can be replaced by `i < N`, eliminating the multiply in the exit test.

After strength reduction and exit-test replacement, many derived IVs may be used only to compute each other; they become dead and can be eliminated by dead code elimination, leaving only the basic IV [Muchnick §14.1].

### 11.4.4 ScalarEvolution in LLVM

LLVM does not implement strength reduction as a standalone pass. Instead, it uses the **ScalarEvolution** (SCEV) analysis (`llvm/lib/Analysis/ScalarEvolution.cpp`) to represent all loop-variant quantities symbolically, and then performs strength reduction and IV simplification as part of **IndVarSimplify** and **LoopStrengthReduce**.

The SCEV representation uses *additive recurrences*: `{start, +, step}<loop>` denotes a value that starts at `start` and increases by `step` on each iteration of `<loop>`. A multiplication inside a loop becomes a `{start, +, step}` recurrence. For example:

```
for i in 0..N:
    x = 5 * i + 3
```

SCEV represents `x` as `{3, +, 5}<loop>`, meaning x = 3, 8, 13, 18, …. Multiply-and-accumulate patterns become `{a, +, b, +, c}<loop>` (quadratic), and so on. SCEV can compute **closed-form expressions** for such recurrences (e.g., `{0, +, 1}<L> = (n*(n+1))/2` for the triangular sum), enabling the replacement of loops with scalar computations in some cases.

IndVarSimplify uses SCEV to canonicalize loop induction variables: it replaces all derived IVs with their SCEV-expressed forms and eliminates redundant IVs. LoopStrengthReduce then replaces multiply-based recurrences with addition-based ones. The connection between SCEV and the polyhedral model (Chapter 71) is deep: SCEV computes the same affine recurrences that the polyhedral model requires for loop transformations.

**Worked SCEV example**: consider address computation in a two-dimensional array traversal:

```c
float A[M][N];
for (int i = 0; i < M; i++)
    for (int j = 0; j < N; j++)
        A[i][j] += 1.0f;
```

The address of `A[i][j]` is `base + (i * N + j) * sizeof(float)`. After flattening:
- Outer IV: `{0, +, N}<outer_loop>` (increments by N per outer iteration)
- Inner IV: `{0, +, 1}<inner_loop>` (increments by 1 per inner iteration)
- Combined: `{base, +, 4}<inner_loop>` with outer start `{base, +, 4*N}<outer_loop>`

LoopStrengthReduce recognizes this and introduces a single pointer `p = base` that increments by `4` (sizeof float) each inner iteration and is reset to `base + i * 4 * N` at each outer iteration. The inner multiply `i * N` is completely eliminated from the inner loop, replaced by an addition-only recurrence.

**The trip count problem**: SCEV can also compute the trip count of a loop — the number of times the loop body executes — as a closed-form expression in terms of initial values. For `for (i = 0; i < N; i++)`, the trip count is `N`. For more complex exit conditions, SCEV uses symbolic arithmetic to derive the trip count, which is then used by LoopUnroll to decide whether to fully unroll a loop (if the trip count is a compile-time constant) or partially unroll it.

---

## 11.5 Partial Redundancy Elimination and Lazy Code Motion

### 11.5.1 Partial vs. full redundancy

An expression `E` is **fully redundant** at program point `p` if `E` is computed on every path reaching `p` and none of `E`'s operands are redefined along any such path. Full redundancy is handled by CSE/GVN (Section 11.2).

An expression `E` is **partially redundant** at `p` if it is redundant on *some* but not all paths. For example:

```
       /— [B1: compute a+b] —\
start —                       — [B3: use a+b]
       \— [B2: nothing] ——/
```

At block B3, `a+b` is redundant if control came via B1 (already computed), but not via B2 (not computed). **Partial redundancy elimination** (PRE) inserts a computation of `a+b` at the end of B2, making `a+b` fully redundant at B3, then eliminates the redundant computation in B3.

### 11.5.2 Morel-Renvoise PRE

The classic PRE algorithm of Morel and Renvoise [Morel-Renvoise 1979] solves the problem via two dataflow analyses:

- **PPIN** and **PPOUT** (partially anticipable): indicates whether the expression will be computed along some path from this point; determines where it is safe to insert a copy.
- **EXPIN** and **EXPOUT** (availability): indicates whether the expression is already available (computed and not killed).

The algorithm inserts computations at block exits to "complete" paths where the expression is absent, turning a partial redundancy into a full redundancy. It then runs CSE to eliminate the now-fully-redundant expression [Dragon §9.5].

The Morel-Renvoise formulation, while correct, is complex and not easy to implement efficiently. It was superseded by the cleaner **Lazy Code Motion** framework.

### 11.5.3 Lazy Code Motion

**Lazy Code Motion** (LCM) [Knoop, Rüthing, Steffen 1992] reformulates PRE using four backward and forward dataflow equations:

1. **Anticipability** (backward): An expression is *anticipable* at a point if it will be computed on every path to program exit without redefining its operands.
2. **Availability** (forward): An expression is *available* at a point if it has been computed on every path from program entry without redefining its operands.
3. **Earliest**: The earliest point at which a computation can be inserted is determined by the conjunction of anticipability and non-availability.
4. **Postponability** (forward) + **Latestness**: An optimal placement delays insertion as late as possible to minimize the lifetimes of the introduced temporaries — this is the "lazy" in LCM. The latest insertion point is the last block on each path before the expression is used, consistent with anticipability.
5. **Used expressions** (backward): Tracks which expressions are actually needed at each point, to avoid inserting computations of dead expressions.

The LCM algorithm:
```
Compute Anticipable(B) [backward]
Compute Available(B) [forward]
Compute Earliest(B) = Anticipable(B) ∧ ¬Available(B)
Compute Postponable(B) [forward, using Earliest]
Compute Latest(B) [intersection of Postponable and usage constraints]
Compute Used(B) [backward]

Insert computation of E at the exit of block B if E ∈ Latest(B) ∧ Used(B)
Delete computation of E at entry of block B if E ∈ Used(B) ∧ E ∉ Latest(B)
```

LCM is **the theoretical unification of LICM and CSE**: if an expression is loop-invariant, LCM computes that its "earliest" insertion point is the loop preheader (it can be anticipated backward through the loop header); the placed computation in the preheader makes it available throughout the loop body, and CSE eliminates the in-loop copies. LCM thus subsumes LICM as a special case. Similarly, full CSE is the case where the expression is available on all paths, so no insertion is needed and only deletion occurs [Cooper/Torczon §10.3].

### 11.5.4 GVN-PRE in LLVM

LLVM's `GVN` pass incorporates a form of PRE. When GVN finds that a load or expression is available on some but not all paths, it inserts a speculative computation on the unavailable paths (in the predecessor blocks), provided a safety check (aliasing, exception safety) passes. The combined GVN-PRE is more powerful than pure LCM in practice because value numbering equates expressions that LCM's syntactic approach would treat as distinct — e.g., `a+b` and `b+a` get the same value number and CSE fires, whereas a syntactic dataflow analysis would require separate treatment.

**Code Motion Safety for PRE**: inserting a computation on a path where it did not previously appear can change program semantics if the computation has side effects or can fault. LLVM's GVN-PRE applies PRE only to "safe" expressions:

- Pure arithmetic (no memory access, no exceptions): always safe to insert.
- Loads: safe if the address is provably valid on the inserted path. For load-PRE in a loop, LLVM inserts the load in the preheader only if it can prove the address does not change in the loop (using alias analysis) and is dereferenceable at the preheader (a non-null, inbounds pointer).
- Calls: never safe unless marked `readonly` and `nounwind`.

The LCM framework handles this by requiring that the inserted expression be **anticipable**: it will be computed on every path forward from the insertion point without the operands being killed first. If anticipability cannot be established — e.g., the expression is only sometimes computed after the insertion point — the insertion would introduce a wasted computation on paths that never needed it, potentially increasing register pressure. LLVM conservatively restricts insertion to cases where both safety and anticipability can be established [Cooper/Torczon §10.3].

---

## 11.6 Register Allocation: Graph Coloring Theory

Register allocation is the problem of assigning an unbounded set of virtual registers (the SSA values or temporaries produced by instruction selection) to a finite set of physical machine registers. When the number of simultaneously live values exceeds the number of physical registers, some values must be **spilled** to memory — their value is stored to the stack before use and reloaded afterward. The goal is to minimize spills (because memory accesses are orders of magnitude slower than register accesses) and to maximize **coalescing** (merging a copy `v₁ = v₂` into a single register, eliminating the copy).

Register allocation occurs after **out-of-SSA** destruction: the φ-functions introduced during SSA construction (Chapter 9) must be replaced by explicit copies on predecessor edges. Each φ operand `vᵢ` on the incoming edge from predecessor Bᵢ becomes a copy `φ_result = vᵢ` placed at the end of Bᵢ. The resulting parallel copies are then coalesced during register allocation. The out-of-SSA step can introduce a significant number of copies; the quality of coalescing determines how many survive to the final code.

**Pre-coloring**: some virtual registers are pre-colored — constrained to specific physical registers. Function arguments are typically passed in specific registers per the ABI (e.g., the first integer argument in `rdi` on x86-64 System V). Return values are placed in specific registers (`rax`). Call-clobbered (caller-saved) registers must be treated as live across any call site. These constraints are modeled as pre-colored nodes in the interference graph: a pre-colored node already has its color assigned and cannot be changed; other nodes that interfere with it cannot use the same color.

### 11.6.1 The interference graph

Two virtual registers `u` and `v` **interfere** if there exists a program point where both are simultaneously live. Formally, `u` and `v` interfere if `u` is live at the definition of `v`, or vice versa. Liveness analysis (Chapter 10) provides the live-out sets for each basic block.

The **interference graph** (IG) has:
- **Nodes**: one per virtual register.
- **Edges**: an undirected edge `(u, v)` if `u` and `v` interfere.
- **Affinity edges** (dashed): one for each copy `u = v`, indicating a preference (not a constraint) to assign `u` and `v` the same register.

Register allocation reduces to **graph k-coloring**: assign one of k colors (physical registers) to each node such that no two adjacent nodes share a color. Two interfering virtual registers must not share a physical register; a register allocation is valid iff the coloring is proper [Chaitin 1982, Dragon §8.8].

### 11.6.2 Graph coloring is NP-complete

k-colorability is NP-complete for k ≥ 3. Chaitin [1982] showed that optimal register allocation (minimizing spills) is NP-complete for this reason. In practice, heuristic algorithms produce near-optimal results for typical programs.

The key theoretical observation that makes practical graph coloring feasible is **Kempe's lemma** [1879]: a node `n` with fewer than k neighbors can always be colored — regardless of how its neighbors are colored, at least one color remains available for `n`. This enables the **simplification** phase: remove nodes of degree < k from the graph (push them onto a stack); the residual graph may become more colorable. After simplification, attempt to color the remaining nodes; if any node cannot be colored, it must be spilled.

### 11.6.3 Chaitin's algorithm

Chaitin's original register allocator [Chaitin et al. 1981] proceeds in five phases:

```
Algorithm: Chaitin Graph Coloring Register Allocation
Input: Interference graph G; k = number of physical registers

Phase 1 — Build: construct G from liveness analysis
Phase 2 — Coalesce: merge copy-related nodes (if merging doesn't create new edges)
Phase 3 — Freeze: select a low-degree copy-related node; mark it uncoalesceable
Phase 4 — Simplify: while ∃ node n with degree < k:
                        remove n from G; push n onto stack
           if G is non-empty:
               spill(select_spill_candidate(G))
               goto Phase 4
Phase 5 — Select: while stack non-empty:
               pop n; restore n to G
               if ∃ color c not used by neighbors of n:
                   assign color c to n
               else:
                   n is an actual spill; insert load/store code
```

**Spill cost heuristic**: select the spill candidate with the lowest cost, estimated as:

```
cost(n) = (freq_def(n) + freq_use(n)) / degree(n)
```

where `freq_def` and `freq_use` weight definitions and uses by their loop depth (a use inside a loop of depth d contributes weight 10^d). Belady's optimal offline algorithm (evict the value not needed for the longest time) gives a lower bound on spills but is not applicable here because the interference graph is not ordered by time.

**Problem with Chaitin's algorithm**: coalescing is performed optimistically before simplification. If coalescing merges two nodes `u` and `v` into `uv`, the degree of `uv` in G is `|neighbors(u) ∪ neighbors(v)|`, which may be ≥ k, making `uv` hard to simplify. Chaitin's original algorithm therefore sometimes performs coalescing that makes spilling necessary.

### 11.6.4 Chaitin-Briggs improvements

**Briggs, Cooper, and Torczon** [Briggs-Cooper-Torczon 1994] introduced two key improvements:

**1. Optimistic coloring (Briggs)**: Do not mark a node as "must spill" during simplification just because it has degree ≥ k. Instead, push it onto the stack anyway (it is a potential spill). During the Select phase, attempt to color it. If a color is available (because some neighbors received the same color from each other), the node is colored successfully — an **optimistic spill** that does not actually spill. Chaitin's pessimistic approach would have inserted unnecessary spill code.

**2. Conservative coalescing (Briggs's test)**: Coalesce nodes `u` and `v` only if the merged node `uv` has **fewer than k high-degree neighbors** (where a neighbor is "high-degree" if it has degree ≥ k in the current graph). If the merged node would have fewer than k high-degree neighbors, it can always be colored by Kempe's lemma, so coalescing is safe. Briggs's test is less aggressive than Chaitin's (which always coalesces non-interfering copies) but avoids the situations that Chaitin's coalescing could make uncolorable.

Together, these improvements deliver significantly fewer spills (typically 10–40% fewer on standard benchmarks) with no overhead in compilation time.

### 11.6.5 Iterated Register Coalescing (George-Appel)

**George and Appel** [George-Appel 1996] unified simplification and coalescing into a single iterative algorithm, **Iterated Register Coalescing** (IRC), which is now the standard reference algorithm for graph-coloring register allocation. IRC uses five mutually-exclusive node worklists and processes them until all nodes are colored or spilled.

**Data structures**:
- `simplifyWorklist`: low-degree, non-copy-related nodes (candidates for simplification).
- `freezeWorklist`: low-degree, copy-related nodes.
- `spillWorklist`: high-degree nodes.
- `coalescedMoves`, `constrainedMoves`, `frozenMoves`, `worklistMoves`, `activeMoves`: move classification sets.
- `selectStack`: nodes pushed during simplification.
- `coloredNodes`, `spilledNodes`, `coalescedNodes`.

**Move classification**: a copy `(u, v)` is in one of:
- `coalescedMoves`: already coalesced.
- `constrainedMoves`: `u` and `v` interfere — cannot coalesce.
- `frozenMoves`: no longer a candidate for coalescing (frozen).
- `worklistMoves` / `activeMoves`: not yet processed / currently being considered.

George's test for coalescing `u` into `v`: for every neighbor `t` of `u`, either `t` interferes with `v` already, or `t` has degree < k. If this holds, coalescing is safe (it cannot increase the degree of `v` above what it already handles) [George-Appel 1996].

```
Algorithm: Iterated Register Coalescing (IRC)

procedure Main():
    Build()         // build interference graph; classify moves
    MakeWorklist()  // partition nodes into simplify/freeze/spill worklists
    repeat:
        if simplifyWorklist ≠ ∅: Simplify()
        elif worklistMoves ≠ ∅:  Coalesce()
        elif freezeWorklist ≠ ∅: Freeze()
        elif spillWorklist ≠ ∅:  SelectSpill()
    until all worklists empty
    AssignColors()
    if spilledNodes ≠ ∅:
        RewriteProgram()    // insert load/store for actual spills
        Main()              // iterate: new spill temporaries may introduce new interference

procedure Simplify():
    n ← pick from simplifyWorklist
    push n onto selectStack
    for each adjacent m: DecrementDegree(m)

procedure Coalesce():
    (u,v) ← pick from worklistMoves
    (u, v) ← (GetAlias(u), GetAlias(v))
    if u == v: move to coalescedMoves; AddWorklist(u)
    elif Precolored(v) or Interferes(u,v): move to constrainedMoves; AddWorklist(u); AddWorklist(v)
    elif OK(u,v) [George's test] or Conservative(u,v) [Briggs's test]:
        move to coalescedMoves; Combine(u,v); AddWorklist(u)
    else: move to activeMoves

procedure AssignColors():
    while selectStack ≠ ∅:
        n ← pop from selectStack
        okColors ← {0..k-1}
        for each neighbor w of n:
            if w ∈ coloredNodes ∪ precolored:
                okColors -= {color[GetAlias(w)]}
        if okColors = ∅: add n to spilledNodes
        else: color[n] ← any c ∈ okColors
```

IRC is proven to terminate: each iteration either reduces the number of uncolored nodes (simplification) or reduces the number of moves (coalescing/freezing/spilling). The algorithm delivers both coalescing and coloring, runs in O(n log n) per iteration, and typically requires at most 2–3 iterations [George-Appel 1996].

**Proof of correctness sketch**: The IRC algorithm maintains three invariants throughout:
1. If node `n` is on the `simplifyWorklist`, it has fewer than k significant neighbors — by Kempe's lemma, it can always be colored.
2. If a move `(u, v)` is coalesced (merged), the resulting node is "safe" by either George's or Briggs's test — it can always be colored.
3. A node pushed onto `selectStack` is eventually colored in AssignColors; if no color is available, it goes to `spilledNodes`. Spilled nodes have spill code inserted and the entire process repeats with the new (smaller) interference graph.

Termination follows because each invocation of Main() has strictly fewer virtual registers (some are spilled and replaced by smaller sub-range temporaries); colorability is preserved by the conservative coalescing tests; and the number of iterations is bounded by the number of initial virtual registers.

### 11.6.6 Worked example: interference graph and coloring

Consider the following three-address code (k = 3 registers: r1, r2, r3):

```
1: a = load(p)       // a live: {a}
2: b = load(q)       // b live: {a, b}
3: c = a + b         // c live: {a, b, c}
4: d = a * 2         // d live: {b, c, d}   (a dead after)
5: e = c - d         // e live: {b, e}       (c, d dead)
6: store(b, e)       //                      (b, e dead)
```

Interference edges (pairs simultaneously live):
- (a,b): live together at points 2–3
- (a,c): live together at point 3
- (b,c): live together at points 3–4
- (b,d): live together at point 4
- (c,d): live together at point 4
- (b,e): live together at point 5

Interference graph:
```
    a ——— b ——— d
    |   / |
    c     e
    |
   (a-c edge)
```

Degrees: a:2, b:4, c:2, d:2, e:1. With k=3: nodes b and d are initially high-degree; all others are low. After removing e (degree 1), then d (degree becomes 1 after e removed), the simplification succeeds. Assign colors greedily during Select:
- a → r1
- b → r2 (interferes with a:r1)
- c → r3 (interferes with a:r1, b:r2 → r3 available)
- d → r1 (interferes with b:r2, c:r3 → r1 available)
- e → r1 (interferes with b:r2 → r1 available)

Result: no spills, all values fit in 3 registers.

---

## 11.7 Linear Scan and PBQP Register Allocation

### 11.7.1 Linear scan register allocation

Graph coloring is O(n²) in the worst case (building the interference graph requires comparing all live-range pairs). For JIT compilers and AOT compilers with tight compilation budgets, this is unacceptable. **Linear scan register allocation** [Poletto-Sarkar 1999] achieves O(n log n) at the cost of some allocation quality.

Linear scan operates on **live intervals** rather than an interference graph. A live interval `[start, end]` for virtual register `v` is the range of instruction indices during which `v` is live. (In SSA form, this is the range from the defining instruction to the last use.)

```
Algorithm: Linear Scan Register Allocation (Poletto-Sarkar)
Input: set of live intervals sorted by start point; k registers

active ← {}    // set of currently live intervals, sorted by end point
free_regs ← {r1, ..., rk}

for each interval i in order of increasing start point:
    ExpireOldIntervals(i)      // free registers of intervals that ended
    if |active| = k:
        SpillAtInterval(i)     // spill the interval ending latest
    else:
        r ← allocate_free_register()
        assign(i, r)
        add i to active (sorted by end point)

procedure ExpireOldIntervals(i):
    for each j in active sorted by end point:
        if endpoint(j) ≥ startpoint(i): return
        remove j from active; free_regs += {register(j)}

procedure SpillAtInterval(i):
    spill ← interval in active with largest endpoint
    if endpoint(spill) > endpoint(i):
        assign(i, register(spill))
        assign(spill, memory location)
        remove spill from active; add i to active
    else:
        assign(i, memory location)   // spill i itself
```

The algorithm is correct: every allocated register is free at the interval's start (by ExpireOldIntervals) and is returned to the free pool at the interval's end. The greedy "spill the longest-lasting interval" heuristic is not optimal in general (optimal linear scan is NP-hard) but performs well in practice.

**Extensions for SSA-based linear scan**:
- In SSA form, live ranges may have **holes** (a value defined at point A, used at point C, but not live in B between them if it goes dead and is redefined). Standard linear scan cannot represent holes; extensions use "live range splitting" to handle this.
- φ-function resolution: after allocating registers, φ-functions must be resolved by inserting copies (or swaps) on predecessor edges. This is the parallel-copy problem and is solved by the out-of-SSA transformation.

**Comparison: linear scan vs. graph coloring**:

| Property | Graph Coloring (IRC) | Linear Scan (Poletto-Sarkar) |
|---|---|---|
| Asymptotic time | O(n log n) per iter | O(n log n) |
| Constant factor | High (multiple passes, interference graph) | Low (one pass per interval) |
| Coalescing quality | Excellent (IRC) | None (no coalescing) |
| Spill quality | Near-optimal | 10–30% more spills |
| Memory | O(n²) interference graph | O(n) intervals |
| JIT suitability | Slow for JIT | Standard for JIT |
| AOT suitability | Standard | Only for fast-compile (-O0) |
| Hole handling | Natural (live ranges per def) | Requires extensions |
| Irregular reg files | Poor (needs PBQP) | Poor (needs extensions) |

Linear scan is used in LLVM's `RegAllocFast` (the `-O0` allocator) and was the basis for JVM HotSpot's C1 compiler and V8's Crankshaft allocator.

**Live interval splitting**: the key insight behind LLVM's RegAllocGreedy (which is neither pure graph-coloring nor pure linear scan, but a hybrid) is **live range splitting**. Instead of deciding "spill or not" for an entire virtual register, the allocator can split its live range: assign part of the range to one physical register, spill for a middle portion (e.g., across a call that clobbers all registers), and reassign to another physical register for the remaining use. This is more flexible than classical graph coloring and produces fewer loads/stores than spilling the entire range. Live range splitting is implemented in LLVM via the `SplitKit` infrastructure, which uses the `LiveRangeEdit` interface to perform splits and recomputes interference after each split.

### 11.7.2 PBQP register allocation

For architectures with **irregular register files** — x86's partial registers (AL/AX/EAX/RAX), ARM's Neon vs. integer registers, special-purpose registers like x87 FP stack — graph coloring and linear scan perform poorly because the "k colors" assumption breaks down. Different instructions require different register classes, and the constraint structure is irregular.

**Partitioned Boolean Quadratic Programming** (PBQP) [Scholz-Eckstein 2002] formulates register allocation as an optimization problem over discrete variables with pairwise costs:

- Each virtual register `v` is a variable `xᵥ` with domain = {r1, r2, ..., rk, spill} (the physical registers assignable to `v`, plus a spill option).
- A **node cost vector** `c(v)[rᵢ]` = cost of assigning `v` to `rᵢ` (0 for most assignments, ∞ for disallowed assignments, positive for avoided assignments).
- An **edge cost matrix** `c(u,v)[rᵢ][rⱼ]` = cost of assigning `u=rᵢ` and `v=rⱼ` simultaneously. Interference: `∞` if `rᵢ = rⱼ`. Affinity (copy): low cost if `rᵢ = rⱼ` (reward for coalescing).

The PBQP problem is to assign each `xᵥ` a value minimizing the sum of node and edge costs. Solving PBQP exactly is NP-hard, but approximation via iterative local reduction (PBQP reduction rules: remove degree-0 nodes, simplify degree-1 nodes, and apply a heuristic for higher-degree nodes) is fast in practice and delivers optimal results for many practical instances.

LLVM includes a PBQP-based register allocator (`RegAllocPBQP`) targeted primarily at architectures with register-class constraints.

**Register allocation for SSA vs. non-SSA form**: the two main approaches in modern compilers are:

1. **Destroy SSA first, then allocate**: the traditional pipeline. Out-of-SSA introduces copies at φ-function join points; the allocator coalesces them. LLVM's RegAllocGreedy and RegAllocFast follow this approach: they operate on `MachineFunction` after `PHIElimination` has destroyed φ-functions.

2. **Allocate in SSA form, then destroy**: the research approach [Hack et al. 2006]. SSA-based allocation uses the SSA interference structure (two SSA values interfere iff one is live at the definition of the other, which is easy to check by dominance). After allocation, φ-functions are resolved by a parallel-copy insertion pass. This approach avoids the blow-up of copies introduced by naive out-of-SSA destruction.

The theoretical quality of SSA-based allocation is comparable to graph-coloring on the same program; it runs in linear time for programs with treewidth-bounded interference graphs (which most practical programs have). LLVM's research branch explored SSA-based RA; the production compiler still uses the destroy-then-allocate approach for engineering simplicity.

### 11.7.3 SSA-based register allocation

Recent research and compiler implementations (including several JIT compilers) perform register allocation **directly on SSA form**, without first destroying SSA. The key insight [Hack et al. 2006] is that SSA itself encodes a structured interference graph: two SSA values interfere iff one is live at the definition of the other, and the dominator-tree structure of SSA makes this easy to compute incrementally. φ-functions are resolved after allocation by a parallel-copy insertion pass (the "SSA deconstruction" or "out-of-SSA" phase, Chapter 9). SSA-based RA can achieve graph-coloring quality at linear-scan speed for many programs.

---

## 11.8 Instruction Scheduling

### 11.8.1 The scheduling problem

A modern processor can execute multiple instructions simultaneously (superscalar, out-of-order, VLIW). To exploit this, the compiler must order instructions to expose **instruction-level parallelism** (ILP): arrange independent instructions adjacent to each other so the hardware can issue them in the same cycle. Additionally, the scheduler must avoid **hazards**: structural (two instructions contend for the same functional unit), data (a producer–consumer dependency with a pipeline latency), and control (branch misprediction penalties).

The scheduling problem: given a basic block (or region) of instructions, produce a reordering that minimizes execution time, subject to data-dependence constraints and resource constraints.

### 11.8.2 The dependence DAG

The **dependence DAG** has one node per instruction and directed edges for each dependence:

- **RAW** (read-after-write, true dependence): instruction `j` reads a register written by `i`. Edge `i → j` with weight = latency of `i`.
- **WAW** (write-after-write, output dependence): `j` writes the same register as `i`. Edge `i → j` (to preserve the final value).
- **WAR** (write-after-read, anti-dependence): `j` writes a register read by `i`. Edge `i → j`.
- **Memory dependences**: if `i` writes to memory and `j` reads (or writes) potentially aliased memory, add a memory dependence edge. Alias analysis (Chapter 10) can eliminate conservative edges.

The **critical path** of the DAG is the longest weighted path from any source to any sink, where edge weights are the producer latencies. The critical path length is a lower bound on the schedule length (no schedule can complete in fewer cycles).

**Dependence DAG example**: suppose a superscalar machine with one multiply unit (3-cycle latency) and two ALU units (1-cycle latency):

```
I1: r1 = load [p]     // 4-cycle latency (memory)
I2: r2 = r1 * 5       // 3-cycle multiply; RAW on I1
I3: r3 = r2 + r4      // 1-cycle; RAW on I2
I4: r5 = load [q]     // 4-cycle; independent of I1..I3
I5: r6 = r5 + r3      // 1-cycle; RAW on I4 and I3
```

Edges: I1→I2 (weight 4), I2→I3 (weight 3), I3→I5 (weight 1), I4→I5 (weight 4).
Critical path: I1→I2→I3→I5 = 4+3+1 = 8 cycles.

An optimal schedule:
```
Cycle 0: issue I1, issue I4    (both independent)
Cycle 4: issue I2              (I1 result available)
Cycle 4: (I4 result ready at cycle 4)
Cycle 7: issue I3              (I2 result available)
Cycle 8: issue I5              (I3 result at cycle 8, I4 result at cycle 4 — already ready)
```

Total schedule length: 9 cycles (8 + 1 for I5). Without scheduling I4 early (if issued at cycle 8), I5 would have to wait until cycle 12 — 3 extra cycles of pipeline bubble.

### 11.8.3 List scheduling

**List scheduling** [Dragon §10.3] is the standard greedy instruction scheduling algorithm:

```
Algorithm: List Scheduling
Input: Dependence DAG D; ready_queue (priority queue of instructions)

Initialize ready_queue with all source nodes of D (no predecessors)
schedule ← []

while ready_queue ≠ ∅:
    i ← dequeue highest-priority instruction from ready_queue
    append i to schedule
    for each successor j of i in D:
        decrement pending-predecessor count of j
        if count = 0: enqueue j into ready_queue with priority P(j)
```

**Priority functions**:
- **Critical path remaining** (most common): priority = length of the longest path from `i` to any sink. Prioritizes instructions on the critical path.
- **Register pressure heuristic**: prefer instructions that free registers (whose output will be used soon but whose inputs can be freed).
- **Latency**: prefer longer-latency operations to start them early (useful for filling pipeline bubbles).

List scheduling is O(n log n) (dominated by the priority queue operations). It is not optimal in general (optimal scheduling is NP-complete), but it is within a small constant factor of optimal for most programs [Cooper/Torczon §11.3].

LLVM's `SelectionDAG` scheduler and `MachineScheduler` both implement list scheduling with target-specific priority functions.

### 11.8.4 Trace scheduling

Fischer's **trace scheduling** [Fisher 1981] was designed for VLIW (Very Long Instruction Word) architectures that require explicit parallel operation. The idea: identify a **trace** — a linear sequence of basic blocks (following the most likely path through the CFG) and schedule them as a single large block, ignoring branches. If the "off-trace" control flow is taken, **compensation code** is inserted on those paths to correct for instructions that were moved across branch points.

Trace scheduling achieves high ILP but the compensation code can be significant. **Superblock scheduling** (a restricted trace without backward branches) and **hyperblock scheduling** (with predicated execution to convert if-then-else into straight-line code) are practical variants that limit the compensation code complexity [Hwu et al. 1993].

**Register pressure during scheduling**: an important secondary objective is to keep the number of simultaneously live values (the register pressure) below the number of physical registers. Scheduling for minimal execution time and scheduling for minimal register pressure are in tension: the schedule that maximizes ILP (by reordering instructions to execute in parallel) often increases register pressure (more values are simultaneously in flight). The **register-pressure-aware list scheduling** heuristic [Touati et al. 2005] switches between "ILP-first" and "register-pressure-first" priority functions based on a register pressure estimate. LLVM's `MachineScheduler` supports this via the `RegPressure` scheduling unit, which monitors pressure during scheduling and prefers register-freeing instructions when pressure exceeds a threshold.

**Pre-RA vs. post-RA scheduling**: LLVM performs instruction scheduling at two points in the pipeline:
1. **Pre-register-allocation scheduling** (`SelectionDAGISel` and `MachineScheduler`): schedules virtual-register code to expose ILP and reduce register pressure, before register allocation.
2. **Post-register-allocation scheduling** (`PostRAScheduler`): schedules physical-register code to eliminate pipeline bubbles and fill delay slots, after the register file is known. Post-RA scheduling is restricted (it cannot create new live ranges or spills), but it can fill instruction slots left by register allocation (e.g., after a spill reload, schedule an independent instruction to hide the load latency).

### 11.8.5 Software pipelining and modulo scheduling

**Software pipelining** overlaps iterations of a loop to fill functional unit pipelines. Rather than scheduling one iteration at a time, the compiler finds a **periodic schedule** (a kernel) that, when repeated, overlaps successive iterations. The key metric is the **Initiation Interval** (II): the number of cycles between the start of successive iterations.

The lower bound on II is:
```
II ≥ max(ResourceBound, RecurrenceBound)

ResourceBound = max over all resources r: ⌈uses(r) / capacity(r)⌉
RecurrenceBound = max over all cycles C in the DDG: ⌈weight(C) / length(C)⌉
```

**Modulo Scheduling** [Rau 1994] finds a schedule with a given II and assigns each instruction to a slot `s mod II`. **Swing Modulo Scheduling** (SMS) [Llosa et al. 1996] improves on basic modulo scheduling by choosing the scheduling order to minimize recurrence delays. LLVM's `MachinePipeliner` (`lib/CodeGen/MachinePipeliner.cpp`) implements SMS and is used for Hexagon, RISC-V, and other targets that benefit from software pipelining (Chapter 91).

**Modulo scheduling worked example**: consider a loop with three operations, each with 1-cycle latency, and a machine with 2 functional units (can issue 2 instructions per cycle):

```
A: load r1, [p]      // latency 3 cycles (memory)
B: mul r2, r1, r3    // latency 2 cycles; depends on A (RAW)
C: store [q], r2     // latency 1 cycle; depends on B (RAW)
```

ResourceMII = ceil(3 ops / 2 units) = 2.  
RecurrenceMII = 0 (no loop-carried dependences).  
MII = max(2, 0) = 2.

A valid kernel schedule at II=2:
```
Cycle 0 mod 2: issue A (iteration n), issue C (iteration n-2)
Cycle 1 mod 2: issue B (iteration n)
```

This overlaps three iterations: while iteration n's A is loading, iteration n-1's B is multiplying, and iteration n-2's C is storing. The steady-state throughput is 3 operations per 2 cycles instead of 3 operations per (3+2+1)=6 cycles — a 3× speedup [Muchnick §17.2].

---

## 11.9 Garbage Collection Theory

Garbage collection (GC) is the automatic reclamation of memory that is no longer reachable from any live reference. From the compiler's perspective, GC requires cooperation: the compiler must generate **stack maps** (safepoints) that identify all live GC references at each GC-safe point, and it must insert **write barriers** for incremental/concurrent collectors. This section surveys the major GC algorithms; Chapter 58 covers LLVM's statepoint mechanism in detail.

### 11.9.1 Mark-Sweep

**Mark-Sweep** [McCarthy 1960] is the original GC algorithm:

1. **Mark phase**: starting from GC roots (stack variables, global variables, registers), traverse all reachable objects via reference edges (BFS or DFS), marking each reachable object.
2. **Sweep phase**: scan the entire heap linearly; any object not marked is garbage and is added to the free list.

**Complexity**: O(heap_size) per collection. **Pause time**: the entire collection is a stop-the-world pause. **Fragmentation**: freed objects are scattered through the heap; allocator must manage a free list, leading to fragmentation and slow allocation.

### 11.9.2 Copying Collection (Cheney)

**Cheney's copying collector** [Cheney 1970] divides the heap into **from-space** and **to-space**:

1. Traverse from GC roots. When a live object `o` is encountered in from-space, copy it to to-space at the next available address (`scan` pointer advances). Write a **forwarding pointer** in `o`'s old location in from-space pointing to its new location.
2. Process the to-space BFS queue: for each reference field in a to-space object, if the referenced object is in from-space, copy it to to-space (or follow its forwarding pointer if already copied).
3. When `scan` catches up to `free` (the BFS queue is empty), collection is complete. Swap from-space and to-space.

**Advantages**: O(live_size) complexity — faster than mark-sweep when garbage is plentiful; automatic compaction (no fragmentation); allocation is a simple bump pointer. **Disadvantages**: 2× memory overhead; cache effects from BFS traversal (depth-first scan, as in the Jonkers semispace collector, improves locality).

### 11.9.3 Generational GC

The **generational hypothesis** [Ungar 1984]: most dynamically allocated objects die young. Generational collectors exploit this by partitioning the heap into generations:

- **Young generation (nursery)**: newly allocated objects. **Minor GC** collects only the young generation — fast because it is small.
- **Old generation (tenured space)**: objects that survive one or more minor GCs are promoted to the old generation. **Major GC** (full collection) collects the entire heap — infrequent.

The complication: old-to-young pointers. An old object may reference a young object; without tracking these cross-generation references, a young-generation GC might incorrectly collect a young object that is reachable from the old generation. **Write barriers** intercept every pointer write: when an old object stores a reference to a young object, the old object is added to the **remembered set**. Minor GC treats the remembered set as additional roots.

Generational GC is the dominant strategy in production managed runtimes: Java's G1, ZGC, Shenandoah; .NET's garbage collector; Python's cyclic GC. LLVM's statepoint mechanism (Chapter 58) supports generational GC by generating precise stack maps at safepoints [Latte et al. 2014].

### 11.9.4 Incremental and concurrent GC

**Incremental GC** interleaves GC work with application (mutator) execution, reducing individual pause times. **Concurrent GC** runs the GC in a separate thread alongside the mutator.

The key technique for both is **tri-color marking** [Dijkstra et al. 1978]:
- **White**: not yet visited — may be garbage.
- **Gray**: visited but children not yet fully processed.
- **Black**: visited and all children processed — definitely reachable.

**Invariant**: no black object may point directly to a white object (tri-color invariant). If this invariant is maintained, the black objects form a complete "reachability frontier" and the white objects are provably unreachable.

**Dijkstra write barrier**: when the mutator writes a reference from any object to a white object `w`, shade `w` gray (add to the gray worklist). This prevents the mutator from "hiding" live objects from the concurrent GC.

Go's runtime uses a concurrent tri-color mark-and-sweep GC with a Dijkstra write barrier. Java's ZGC uses a load barrier (the mutator shades objects on load, not store), enabling even more concurrent operation.

### 11.9.5 Region-based memory management

**Region-based memory** (arenas) allocates objects in named **regions** (also called arenas or zones). All objects in a region are freed atomically when the region is deallocated — no per-object tracing is required. Allocation is a bump pointer within the region: O(1) and cache-friendly. This is not garbage collection in the traditional sense, but a form of deterministic reclamation.

The Tofte-Talpin **region type system** [Tofte-Talpin 1997] provides a type-theoretic basis: a region is a type-level construct, and the type system enforces that no object in a freed region is accessible. This is the theoretical foundation for Rust's ownership and borrow-checker model (Chapter 14): Rust's lifetime system is, informally, a region type system where lifetimes correspond to region names.

In practice, many compiler and runtime components use arenas extensively: LLVM's `BumpPtrAllocator`, Clang's AST nodes (allocated in a per-TU arena), and database query processors. Arena allocation eliminates GC overhead in subsystems where object lifetimes are naturally structured.

**Comparing GC strategies**: the following table summarizes the key tradeoffs:

| Algorithm | Pause | Throughput | Fragmentation | Memory Overhead | Complexity |
|---|---|---|---|---|---|
| Mark-Sweep | Stop-world, O(heap) | High | Yes | Low | Low |
| Copying (Cheney) | Stop-world, O(live) | High | No | 2× heap | Low |
| Generational | Short minor pauses | Very high | Some (old gen) | Moderate | Medium |
| Incremental | Short incremental | Medium | Yes | Low | High |
| Concurrent (tri-color) | Near-zero | Medium | Yes | Low–medium | Very high |
| Region-based | None | Very high | No | Low | Low (deterministic) |

The "throughput" column reflects amortized cost per object allocated: copying and generational collectors have the highest throughput because bump-pointer allocation and short minor GCs are extremely fast. Concurrent collectors trade throughput for pause reduction [Jones-Hosking-Moss 2011].

### 11.9.6 Compiler support for GC

The compiler must provide the GC with:

1. **Stack maps (safepoints)**: at each call site (or any GC-safe point), a description of which stack slots and registers contain live GC references. The GC uses these to find all roots.
2. **Write barrier insertion**: the compiler inserts the barrier code (a conditional branch into the GC write-barrier slow path) at every pointer store.
3. **Reference type distinctions**: strong references (counted by GC), weak references (not counted, nulled on GC), and interior pointers (pointers into the middle of an object, requiring the GC to find the object base).

LLVM supports GC via **statepoints** (`llvm.experimental.gc.statepoint` intrinsic) and **stack maps** (`llvm.experimental.stackmap`). A function that participates in GC annotates its function with `gc "statepoint-example"` (or a custom GC strategy). The statepoint intrinsic wraps a call site, marking which values must be considered GC roots across the call. The `RewriteStatepointsForGC` pass inserts the necessary relocations. See Chapter 58 for the full treatment.

---

## 11.10 LLVM Implementations

This section maps each algorithm in this chapter to its LLVM implementation, providing file paths and pass names as of LLVM 18. Forward references to the practice chapters are noted.

### 11.10.1 GVN and NewGVN

- **`GVN`**: `llvm/lib/Transforms/Scalar/GVN.cpp`. RPO-based, includes load-PRE. Registered as `gvn` in the new pass manager.
- **`NewGVN`**: `llvm/lib/Transforms/Scalar/NewGVN.cpp`. Congruence-class-based VEQ framework; handles φ-functions correctly in loops. Registered as `newgvn`.
- **`EarlyCSE`**: `llvm/lib/Transforms/Scalar/EarlyCSE.cpp`. Simplified LVN-style CSE for early in the pipeline (before full alias analysis is available). Uses a scoped hash table over the dominator tree.
- See Chapter 62 for how these passes compose in the optimization pipeline.

### 11.10.2 LICM

- **`LICM`**: `llvm/lib/Transforms/Scalar/LICM.cpp`. Uses `LoopInfo`, `DominatorTree`, and `MemorySSA` for precise alias checking. Runs as part of the loop optimization pipeline.
- Key interaction: `MemorySSA` (`llvm/lib/Analysis/MemorySSA.cpp`) tracks memory definitions and uses in SSA form; LICM queries it to prove that a load's definition site is loop-invariant.
- See Chapter 63 for the full loop optimization pipeline.

### 11.10.3 IndVarSimplify and ScalarEvolution

- **`ScalarEvolution`**: `llvm/lib/Analysis/ScalarEvolution.cpp`. The SCEV analysis; provides `{start, +, step}` recurrence representations for all loop-variant values.
- **`IndVarSimplify`**: `llvm/lib/Transforms/Scalar/IndVarSimplify.cpp`. Uses SCEV to canonicalize induction variables, eliminate redundant IVs, and replace loop exit conditions.
- **`LoopStrengthReduce`**: `llvm/lib/Transforms/Scalar/LoopStrengthReduce.cpp`. Uses SCEV to identify and reduce strength of multiply-based recurrences; minimizes the number of induction variables using a cost model.
- See Chapter 63.

### 11.10.4 The Greedy Register Allocator

LLVM's default register allocator is **RegAllocGreedy** (`llvm/lib/CodeGen/RegAllocGreedy.cpp`), which implements a variant of Chaitin-Briggs graph coloring with live-range splitting. Key design:

- **Live range splitting**: rather than spilling an entire virtual register, split its live range into sub-ranges. Allocate some sub-ranges to registers and spill others. This significantly reduces spill code.
- **Eviction**: when a virtual register `v` cannot be allocated to any color, consider evicting a currently-allocated register if `v` has a lower spill cost; the evicted register is pushed back onto the worklist.
- **Interference graph**: built lazily via `LiveIntervals` (`llvm/lib/CodeGen/LiveIntervals.cpp`).
- **Coalescing**: `RegisterCoalescer` (`llvm/lib/CodeGen/RegisterCoalescer.cpp`) performs copy coalescing, implementing both Briggs's and George's tests.

See Chapter 90 for a full treatment of LLVM's register allocation pipeline.

### 11.10.5 Linear Scan (RegAllocFast)

**`RegAllocFast`** (`llvm/lib/CodeGen/RegAllocFast.cpp`) is LLVM's fast allocator, used at `-O0`. It implements a simplified linear-scan strategy: process instructions in order; maintain a register-to-variable mapping; spill on overflow. No interference graph, no coalescing. Compilation speed is O(n). See Chapter 90.

### 11.10.6 MachinePipeliner (SMS)

**`MachinePipeliner`** (`llvm/lib/CodeGen/MachinePipeliner.cpp`) implements Swing Modulo Scheduling. It is enabled for Hexagon (`-mcpu=hexagonv60` and later) and RISC-V. The pipeliner:

1. Computes the dependence DAG for the loop kernel.
2. Computes MII (Minimum Initiation Interval) = max(ResourceMII, RecurrenceMII).
3. Iteratively tries II = MII, MII+1, ... until a valid schedule is found.
4. Generates prologue and epilogue code to fill and drain the pipeline.

See Chapter 91 for a detailed walkthrough.

### 11.10.7 GC statepoints

**`RewriteStatepointsForGC`** (`llvm/lib/Transforms/Scalar/RewriteStatepointsForGC.cpp`) transforms calls into statepoint intrinsics, inserting `gc.relocate` uses for all GC-managed values that are live across call sites. The result is used by the backend to generate LLVM stack maps in the object file's `.llvm_stackmaps` section. See Chapter 58 for the statepoint ABI.

A statepoint-wrapped call looks like:

```llvm
; Before RewriteStatepointsForGC:
%result = call i64 @some_fn(i64* %ptr)

; After (conceptually):
%statepoint_token = call token (i64 addrspace(1)* %ptr)
                         @llvm.experimental.gc.statepoint.p0f_i64f(
                             i64 0, i32 0, i64()* @some_fn, i32 0, i32 0,
                             i64 addrspace(1)* %ptr)
%ptr.relocated = call i64 addrspace(1)* @llvm.experimental.gc.relocate.p1i64(
                             token %statepoint_token, i32 0, i32 0)
%result = call i64 @llvm.experimental.gc.result.i64(token %statepoint_token)
```

The `gc.relocate` intrinsic informs subsequent IR passes and the backend that `%ptr` may have moved (been relocated by the GC) across the call, and all future uses should use `%ptr.relocated` instead. The backend emits a **stack map record** that describes the exact location (register or stack slot) of each `gc.relocate` operand, so the GC can find and update all live references.

### 11.10.8 The LLVM optimization pipeline overview

The `-O2` pass pipeline in LLVM 18 (as produced by `opt -O2 -print-pipeline-passes`) progresses roughly as:

1. **Simplification**: AlwaysInliner, SROA (scalar replacement of aggregates), EarlyCSE, SimplifyCFG.
2. **Inlining**: the main inliner pass, which runs until the call graph stabilizes.
3. **Loop canonicalization**: LoopSimplify (natural loop structure, preheaders), LCSSA (loop-closed SSA form).
4. **Loop optimization**: LoopRotate, LICM, SimpleLoopUnswitch, IndVarSimplify, LoopStrengthReduce, LoopUnroll, LoopVectorize.
5. **Scalar cleanup**: GVN (with load-PRE), InstCombine, ADCE (aggressive dead code elimination), CorrelatedValuePropagation, MemCpyOpt.
6. **Late cleanup**: DCE, SimplifyCFG, InstCombine.

Each stage feeds the next: SROA breaks up aggregate types into individual scalars (which GVN and CSE can then optimize); loop canonicalization creates preheaders for LICM; IndVarSimplify creates canonical IVs that LoopVectorize can recognize and vectorize. The pipeline embodies decades of accumulated compiler engineering knowledge about which optimizations enable which others [Cooper/Torczon §9.2].

---

## 11.11 Putting It All Together: A Worked Compilation Pipeline

To solidify the connections between the algorithms in this chapter, consider the following C function being compiled with aggressive optimization:

```c
float dot_product(const float * restrict a, const float * restrict b, int n) {
    float sum = 0.0f;
    for (int i = 0; i < n; i++)
        sum += a[i] * b[i];
    return sum;
}
```

The optimization pipeline proceeds as follows:

**1. Frontend and SSA construction** (Chapter 9): Clang emits LLVM IR in SSA form. The loop index `i` and accumulator `sum` are φ-nodes at the loop header. The `restrict` keyword generates `noalias` metadata on `a` and `b`.

**2. SROA**: the local variable `sum` is immediately promoted from alloca to an SSA register (it was trivially a scalar).

**3. Early CSE / InstCombine**: algebraic simplifications; no effect here (already clean).

**4. LoopSimplify**: ensures the loop has a preheader (it does) and a unique backedge. LCSSA inserts φ-nodes at loop exits for `sum`.

**5. LICM**: checks if any expression inside the loop is invariant. The loads `a[i]` and `b[i]` have addresses `{base_a, +, 4}<loop>` and `{base_b, +, 4}<loop>` — they vary per iteration, so they cannot be hoisted. The `noalias` metadata ensures no stores alias `a` or `b`, confirming the loads are safe.

**6. IndVarSimplify / LoopStrengthReduce**: SCEV represents the addresses as `{a, +, 4}<loop>` and `{b, +, 4}<loop>`. LoopStrengthReduce introduces pointer induction variables `p_a = a` and `p_b = b`, each incrementing by 4 per iteration, replacing the `i * 4` computation.

**7. LoopVectorize**: the loop is a reduction (accumulating into `sum`). With `noalias` on `a` and `b`, the vectorizer emits SIMD load and multiply-add intrinsics (e.g., 4× or 8× float vectors on x86 AVX, 16× on AArch64 SVE). The reduction is handled with a `fadd` reduction over the vector at the end.

**8. GVN + InstCombine**: cleans up the scalar remainder loop (the "epilog" for non-multiple-of-vector-width iterations).

**9. Instruction selection** (Chapter 84): SelectionDAG maps the vectorized IR to target-specific MachineInstructions. The float multiply-add becomes `vfmadd` on x86 (if FMA is available) or a `fmul` + `fadd` pair.

**10. Register allocation** (Chapter 90): RegAllocGreedy assigns the pointer IVs and accumulator to registers. On x86-64 with AVX, the YMM registers (ymm0–ymm15) are available; on AArch64, the 32 V-registers. With only a few live values in the loop, no spills occur.

**11. Instruction scheduling** (Chapter 89): the post-RA scheduler reorders instructions within the loop kernel to hide load latency (start loading the next iteration's values while computing the current ones).

The result is a tight vectorized loop with ~0 overhead per iteration beyond the mathematical operations, approaching peak floating-point throughput.

---

## 11.12 Chapter Summary

This chapter has surveyed the theoretical foundations of classical compiler optimization, from simple local transformations to the sophisticated graph algorithms that underpin modern register allocation and instruction scheduling.

Key points:

- **Optimization scope** determines algorithmic complexity: local optimizations are O(n) per block; global optimizations require dataflow analysis (O(n × d) per pass, iterated); interprocedural optimizations require call-graph analysis and are the most expensive. The **SCCP** algorithm shows how combining constant propagation and dead code elimination in a single fixed-point iteration discovers more constants than sequential application.

- **Value numbering** — both local (LVN) and global (GVN) — identifies computationally equivalent expressions and eliminates redundant recomputations. SSA form is the key enabler of GVN: the unique-definition invariant makes value identity structurally explicit. LLVM's NewGVN uses congruence classes for maximum precision; GVN incorporates load-PRE.

- **Loop-invariant code motion** (LICM) hoists expressions from loop bodies to preheaders. Correctness requires dominance of all loop exits, absence of aliasing writes, and exception safety. LLVM's MemorySSA (`lib/Analysis/MemorySSA.cpp`) enables O(1) alias checks per load during LICM. Speculative hoisting with guards handles cases where exception safety cannot be statically established.

- **Strength reduction** and **ScalarEvolution** replace expensive multiplications inside loops with additions, leveraging the SCEV framework's `{start, +, step}` recurrence representation. The SCEV trip-count computation enables loop unrolling decisions; the recurrence representation connects to the polyhedral model (Chapter 71) for advanced loop transformations.

- **Partial redundancy elimination** generalizes both CSE and LICM. The Lazy Code Motion formulation (Knoop-Rüthing-Steffen 1992) is the optimal placement algorithm, minimizing both redundancy and temporary lifetimes. LCM formally proves that LICM and CSE are special cases of the same framework. LLVM's GVN-PRE extends LCM with value numbering for superior cross-syntactic optimization.

- **Graph-coloring register allocation** (Chaitin 1982 → Chaitin-Briggs 1994 → George-Appel IRC 1996) is NP-complete in theory but near-optimal in practice via heuristic simplification (Kempe's lemma), optimistic coloring (avoids unnecessary spills), and conservative coalescing (Briggs's test, George's test). Iterated register coalescing (IRC) is the standard reference algorithm, terminating in O(n log n) per iteration with excellent coalescing quality.

- **Linear scan register allocation** [Poletto-Sarkar 1999] trades allocation quality (10–30% more spills) for dramatically faster compilation (O(n log n), no n² interference graph construction), making it the standard JIT allocator. PBQP captures irregular register file constraints as a quadratic optimization problem. SSA-based allocation is a research direction achieving graph-coloring quality at linear-scan speed.

- **List scheduling** greedily schedules instructions in critical-path order to minimize execution time. Register-pressure-aware scheduling balances ILP against register pressure. Software pipelining (modulo scheduling / SMS) overlaps loop iterations by finding a periodic kernel with minimum initiation interval II = max(ResourceMII, RecurrenceMII); LLVM's MachinePipeliner implements SMS for Hexagon and RISC-V.

- **Garbage collection** algorithms span a tradeoff space of pause time, throughput, fragmentation, and memory overhead: mark-sweep (simple, stop-world), copying (compact, 2× memory), generational (short minor pauses via generational hypothesis), and concurrent tri-color marking (near-zero pauses via write barriers). Region-based memory eliminates GC overhead for structured lifetimes. Compilers support GC via stack maps, safepoints, and write barrier insertion; LLVM uses the statepoint/relocate intrinsic pair and the `.llvm_stackmaps` ELF section.

- **LLVM implementations** of all these algorithms are in `lib/Transforms/Scalar/` (GVN, LICM, IndVarSimplify, PRE), `lib/Analysis/` (ScalarEvolution, MemorySSA), and `lib/CodeGen/` (RegAllocGreedy, RegAllocFast, MachinePipeliner, RegisterCoalescer). The `-O2` pipeline orchestrates all of these into a carefully ordered sequence that progressively simplifies, analyzes, and transforms the IR toward optimal machine code.

---

*End of Chapter 11. Chapter 12 begins Part III — Type Theory, opening with lambda calculus and simple types: the formal foundations that underlie every type system in a compiler frontend, from C's structural types to Haskell's System F.*


---

@copyright jreuben11
