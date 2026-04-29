# Chapter 70 — Foundations: Polyhedra and Integer Programming

*Part XI — Polyhedral Theory*

The polyhedral model — used by Polly, MLIR's Affine dialect, and every production affine loop optimizer — rests on three mathematical pillars: the geometry of convex polyhedra, the arithmetic of integer lattices, and the logic of Presburger arithmetic. A loop nest's iteration space is a polyhedron. Its dependences are constraints between polyhedra. Its transformations are linear maps over those polyhedra. The algorithms that test legality, compute schedules, and generate new loop bounds all reduce to questions in integer linear programming and Presburger arithmetic. This chapter builds that foundation: from the definitions of polyhedra through the simplex method, the Omega test, and the algorithmic core of ISL — the library that implements these operations for Polly and MLIR.

## 70.1 Convex Polyhedra

### 70.1.1 Definitions

A **convex set** S ⊆ ℝⁿ satisfies: for all x, y ∈ S and λ ∈ [0,1], λx + (1−λ)y ∈ S. A **convex polyhedron** is the intersection of finitely many closed half-spaces:

```
P = { x ∈ ℝⁿ | Ax ≤ b }
```

where A ∈ ℤ^{m×n} is a constraint matrix and b ∈ ℤ^m is a right-hand-side vector. This is the **H-representation** (half-space representation). A **polytope** is a bounded polyhedron.

The **V-representation** (vertex representation) expresses a bounded polyhedron as the convex hull of its vertices plus a cone of its rays:

```
P = conv(V) + cone(R) = { Σ λᵢvᵢ + Σ μⱼrⱼ | λᵢ ≥ 0, Σλᵢ = 1, μⱼ ≥ 0 }
```

where V is the finite set of vertices and R is the finite set of extreme rays. Both representations are equivalent by the Minkowski-Weyl theorem: every polyhedron has both an H-representation and a V-representation.

### 70.1.2 Faces, Facets, and Vertices

A **face** of polyhedron P is any set of the form:

```
F = P ∩ { x | cᵀx = δ }   where cᵀx ≤ δ for all x ∈ P
```

Faces form a lattice ordered by inclusion. The faces of dimension 0 are **vertices**, dimension 1 are **edges**, and dimension (n−1) are **facets**. In the H-representation, each facet corresponds to one tight constraint (equality at a face).

For loop analysis, the vertices of an iteration space polyhedron bound the loop ranges, and the facets define where one loop's iteration set meets another's — critical for dependence analysis.

### 70.1.3 The Farkas Lemma

Farkas' lemma (1902) characterizes when a system Ax ≤ b is infeasible. It is foundational for duality in linear programming and for the scheduling algorithms in Chapter 72:

**Lemma (Farkas):** The system Ax ≤ b has no solution if and only if there exists y ≥ 0 with yᵀA = 0 and yᵀb < 0.

Equivalently: cᵀx ≤ δ holds for all x satisfying Ax ≤ b if and only if there exist y ≥ 0 such that yᵀA = cᵀ and yᵀb ≤ δ.

The Farkas lemma is used directly in Feautrier's scheduling algorithm (Chapter 72) to derive scheduling constraints from dependence constraints: "the schedule of the producer must not exceed the schedule of the consumer" is expressed as a Farkas condition on the dependence polyhedron.

## 70.2 Integer Lattices

### 70.2.1 Lattices

A **lattice** Λ ⊆ ℝⁿ is the set of all integer linear combinations of a set of linearly independent basis vectors b₁, …, bₖ:

```
Λ = { c₁b₁ + … + cₖbₖ | c₁, …, cₖ ∈ ℤ }
```

The standard lattice ℤⁿ has the identity matrix as its basis. For polyhedral analysis, the lattice of interest is the set of integer points inside a polyhedron: P ∩ ℤⁿ — the valid integer iteration vectors.

### 70.2.2 Hermite Normal Form

The **Hermite Normal Form (HNF)** of an integer matrix A is a canonical lower triangular form H such that A = UH for some unimodular matrix U (det(U) = ±1, all entries integers). HNF is the integer analogue of row echelon form over the reals.

HNF algorithms run in polynomial time and are used to:
- Test whether a system Ax = b has an integer solution.
- Find a basis for the integer null space of A.
- Canonicalize loop transformation matrices.

```
A = [2 4 ; 1 3]

HNF(A) = H = [1 0 ; 0 2]   (lower triangular)
U = [3 -4 ; -1 2]           (unimodular)
A = U * H
```

### 70.2.3 Smith Normal Form

The **Smith Normal Form (SNF)** diagonalizes an integer matrix: given A ∈ ℤ^{m×n}, there exist unimodular matrices U ∈ ℤ^{m×m} and V ∈ ℤ^{n×n} such that:

```
U A V = D   (diagonal, D[i,i] = dᵢ, where d₁ | d₂ | … | dᵣ)
```

The diagonal entries dᵢ are the **invariant factors** of A. SNF is used to:
- Determine whether a dependence system has integer solutions (the condition is gcd-based on the dᵢ).
- Compute exact dependence vectors in the Omega test.
- Classify transformation matrices by their effect on the lattice.

## 70.3 Linear Programming

### 70.3.1 The Linear Program

A **linear program (LP)** is:

```
minimize   cᵀx
subject to Ax ≤ b,  x ≥ 0
```

LPs always have a rational optimal solution (if bounded and feasible). The set of feasible solutions is a convex polyhedron; the optimum is attained at a vertex.

In polyhedral compilation, LPs arise in:
- **Profitability testing**: "Is there a schedule satisfying the dependence constraints?"
- **Loop bound generation**: "What are the tight bounds on loop variables?"
- **Cost minimization**: The Pluto algorithm minimizes a schedule cost function via LP.

### 70.3.2 The Simplex Method

The simplex method (Dantzig, 1947) solves LPs by traversing vertices of the feasibility polyhedron:

```
1. Start at a feasible vertex (corner of the feasibility region).
2. Move to an adjacent vertex that improves the objective.
3. Stop when no improving adjacent vertex exists (optimum found).
```

Despite exponential worst case, the simplex method is highly efficient in practice and is the algorithm used inside ISL for LP-based operations.

**Polyhedral compilation note:** LP infeasibility detection (no feasible vertex) is used to prove that two statement instances are never simultaneously active — enabling dead code elimination and dependence pruning.

### 70.3.3 Fourier-Motzkin Elimination

**Fourier-Motzkin elimination (FME)** projects out one variable from a system of linear inequalities, producing a (potentially larger) system in the remaining variables:

```
Given:  Ax ≤ b  (containing variable xₙ)
Output: A'x' ≤ b'  (same solutions, xₙ eliminated)

Procedure:
  Partition constraints into:
    xₙ ≤ uᵢ(x')  (upper bounds on xₙ)
    lⱼ(x') ≤ xₙ  (lower bounds on xₙ)
    hₖ(x') ≤ 0   (bounds not involving xₙ)
  New constraints: lⱼ(x') ≤ uᵢ(x') for all i, j
  Plus all hₖ constraints
```

FME is used in polyhedral code generation to compute the symbolic loop bounds that appear in the generated C/LLVM code. It can cause quadratic blowup in the number of constraints, but practical SCoPs have small dimensionality.

## 70.4 Integer Linear Programming

### 70.4.1 The ILP Problem

An **integer linear program (ILP)** adds the constraint that the solution must be integer-valued:

```
minimize   cᵀx
subject to Ax ≤ b,  x ∈ ℤⁿ
```

ILP is NP-hard in general. However, the specific ILPs arising in polyhedral compilation have special structure that admits efficient exact algorithms.

### 70.4.2 Branch and Bound

The standard ILP algorithm is **branch and bound**:

1. Solve the LP relaxation (ignoring integrality).
2. If the solution is integer, stop (optimal).
3. If some xᵢ = α (non-integer), branch:
   - Left branch: add xᵢ ≤ ⌊α⌋
   - Right branch: add xᵢ ≥ ⌈α⌉
4. Recurse on each branch; prune branches where LP lower bound exceeds best integer solution found.

For polyhedral compilation, branch-and-bound is used in the Pluto scheduler (Chapter 72) to find the minimal-cost schedule satisfying dependence constraints.

### 70.4.3 Gomory Cuts

**Gomory cuts** strengthen the LP relaxation by adding constraints that are satisfied by all integer solutions but violated by the current fractional solution:

```
If xⱼ = α (fractional), the Gomory cut is:
  Σᵢ ⌊aᵢⱼ⌋ xᵢ ≤ ⌊b⌋
```

Gomory cuts are derived from a row of the simplex tableau and progressively tighten the LP relaxation toward the integer hull.

## 70.5 Presburger Arithmetic

### 70.5.1 Definition

**Presburger arithmetic** (Presburger, 1929) is the first-order theory of the integers with addition (but not multiplication): its language has integer constants, variables, addition, and the predicates =, <, and ≤.

The key result: Presburger arithmetic is **decidable** — there exists an algorithm that determines truth or falsity of any Presburger sentence. This stands in contrast to Peano arithmetic (with multiplication), which is undecidable (Gödel).

In polyhedral compilation, every question about integer iteration spaces — "are these two iterations dependent?", "does this loop execute at all?", "are these two access functions aliased?" — can be expressed as a Presburger formula over the loop induction variables and symbolic parameters.

### 70.5.2 Syntax

A **Presburger formula** is built from:
- **Atomic formulas**: linear equalities and inequalities over ℤ variables (e.g., 2i + j ≤ N)
- **Boolean connectives**: ∧ (and), ∨ (or), ¬ (not)
- **Quantifiers**: ∃i, ∀i (where i ranges over ℤ)
- **Divisibility constraints**: k | t (k divides t) — definable but often added for convenience

Examples:
```
∃i ∈ ℤ: 0 ≤ i ≤ N ∧ A[i] = A[j]   (there exists an aliasing iteration)
∀i ∈ [0,N): 2|i → B[i/2] ≥ 0       (even-indexed elements are non-negative)
```

### 70.5.3 Quantifier Elimination

The key operation on Presburger formulas is **quantifier elimination (QE)**: given a formula with existentially quantified variables, produce an equivalent quantifier-free formula.

For Presburger arithmetic, QE over ∃ is computed by **Cooper's algorithm** (1972):

```
To eliminate ∃x from φ(x, y):
1. Bring φ to DNF (disjunctions of conjunctions).
2. For each conjunction:
   a. Collect lower bounds: xₗ ≤ x  →  x = xₗ + k for k ≥ 0
   b. Collect upper bounds: x ≤ xᵤ
   c. Collect divisibility constraints: d | (ax + t)
3. Compute the "least common multiple" δ of all divisibility moduli.
4. Replace ∃x by: ∃x' ∈ [0, δ): all upper bounds satisfied
   → finite case split over x' ∈ {0, …, δ-1}
5. Substitute and simplify.
```

QE allows the dependence analysis question "do iterations iₛ and iₜ access the same memory location?" to be answered without enumerating all possible (iₛ, iₜ) pairs.

### 70.5.4 The Omega Test

The **Omega test** (Pugh, 1992) is a practical exact integer satisfiability test for Presburger formulas that appears widely in dependence analysis tools:

**Problem**: Given a conjunction of linear constraints over integers, determine satisfiability.

**Algorithm**:
1. Apply **dark shadow** (necessary condition, easy to test): multiply out all pairs of lower/upper bounds and add floor-based slack variables.
2. If the dark shadow is infeasible, the system is infeasible.
3. If the dark shadow is feasible and all equality/divisibility conditions are met, the system is feasible.
4. Otherwise, apply **gray shadow** (exact test): enumerate the gray zone using finite case splits.

The Omega test runs in polynomial time for fixed number of variables (common in loop nests with small loop depth) and is exact — unlike heuristic dependence tests (GCD test, Banerjee inequalities) that may report false dependencies.

```
Example: test if i = 2j + 1 has a solution for 0 ≤ i ≤ 10, 0 ≤ j ≤ 4:
  Dark shadow: i ≡ 1 (mod 2), 0 ≤ i ≤ 10 → 1, 3, 5, 7, 9 ∈ range
  j = (i-1)/2: j ∈ {0, 1, 2, 3, 4} — all valid.
  Result: feasible (e.g., i=1, j=0).
```

## 70.6 ISL: The Integer Set Library

### 70.6.1 Overview

**ISL** (Integer Set Library, Verdoolaege, 2010) is the algorithmic backbone of Polly, MLIR's Affine dialect, and several standalone polyhedral tools. It implements:

- Sets and relations over integer points defined by Presburger formulas
- Exact operations: union, intersection, difference, complement, projection
- Quantifier elimination (Omega-style and Barvinok-style)
- Parametric integer programming (solving ILPs with symbolic parameters)
- Schedule computation (the isl scheduler implements Pluto-style optimization)
- AST generation from polyhedral schedules (the isl_ast_build interface)

ISL is a C library with Python and MLIR bindings. It uses a representation based on **quasi-polynomials** and **Barvinok's counting algorithm** for exact cardinality computation.

### 70.6.2 ISL Data Types

| Type | Represents |
|------|-----------|
| `isl_set` | A set of integer points satisfying a Presburger formula |
| `isl_map` | A relation (set of pairs) between integer points |
| `isl_union_set` | A disjoint union of sets (different domains) |
| `isl_union_map` | A disjoint union of maps |
| `isl_space` | Dimension and naming for spaces |
| `isl_aff` | An affine function on integer points |
| `isl_pw_aff` | A piecewise affine function |
| `isl_schedule` | A polyhedral schedule tree |
| `isl_ast_node` | A node in the generated AST |

### 70.6.3 ISL Set Operations

```c
// ISL C API example: compute iteration space intersection
isl_ctx *ctx = isl_ctx_alloc();

// Create the set { [i, j] : 0 <= i < N, 0 <= j < i }
isl_set *S = isl_set_read_from_str(ctx,
    "[N] -> { [i, j] : 0 <= i < N and 0 <= j < i }");

// Project out j: ∃j such that the constraint holds
isl_set *proj = isl_set_project_out(isl_set_copy(S),
    isl_dim_set, 1, 1);  // project out dimension 1 (j)
// Result: [N] -> { [i] : 1 <= i < N }

// Check if a specific point is in the set
isl_point *p = isl_point_zero(isl_set_get_space(S));
p = isl_point_set_coordinate_val(p, isl_dim_set, 0,
    isl_val_int_from_si(ctx, 3));  // i = 3
p = isl_point_set_coordinate_val(p, isl_dim_set, 1,
    isl_val_int_from_si(ctx, 1));  // j = 1
isl_bool in_set = isl_set_contains(S, p);  // true: 0 <= 1 < 3 < N

isl_set_free(S);
isl_point_free(p);
isl_set_free(proj);
isl_ctx_free(ctx);
```

### 70.6.4 ISL Maps for Dependences

A dependence from statement S1 at iteration (i₁, j₁) to statement S2 at iteration (i₂, j₂) is an `isl_map`:

```c
// Dependence map: { S1[i1,j1] -> S2[i2,j2] : i1 = i2 and j2 = j1 + 1 }
isl_map *dep = isl_map_read_from_str(ctx,
    "{ S1[i, j] -> S2[i, jj] : jj = j + 1 }");

// Domain: source iterations of the dependence
isl_set *domain = isl_map_domain(isl_map_copy(dep));
// Range: destination iterations
isl_set *range = isl_map_range(dep);
```

### 70.6.5 Parametric Integer Programming

ISL implements **Parametric Integer Programming (PIP)** (Feautrier, 1988): given a system of constraints involving both integer variables x and symbolic parameters p, find the lexicographic minimum integer solution for x as a function of p.

PIP is the algorithmic core of Feautrier's scheduling algorithm (Chapter 72): the schedule variables are the unknowns, the dependence constraints are the constraints, and the scheduling parameters (statement counts) determine the solution.

```
PIP: minimize lex{ (θ₁(p), θ₂(p)) } subject to dependence constraints D(θ, p) ≥ 0
```

ISL solves PIP using the **dual simplex method** over the integers, producing piecewise quasi-affine functions as solutions.

## 70.7 Complexity and Practical Considerations

### 70.7.1 Theoretical Complexity

| Problem | Complexity |
|---------|-----------|
| Linear programming (LP) | Polynomial (ellipsoid/interior point); simplex: polynomial in practice |
| Integer linear programming (ILP) | NP-hard in general |
| Presburger arithmetic decision | 2EXPTIME-complete (double exponential) |
| Quantifier elimination (Cooper's) | Double exponential in quantifier alternation |
| Omega test (fixed #variables) | Polynomial |

Despite the pessimistic theoretical complexity, polyhedral tools are practical because loop nests in real programs have:
- Small dimensionality (2–6 loops typically)
- Small number of statements per loop
- Few symbolic parameters (loop bounds, tile sizes)

### 70.7.2 Practical Bounds

ISL uses heuristics to avoid exponential blowup in practice:
- **Pairwise dependence computation**: ISL tracks dependences per statement pair, not globally.
- **Coalescing**: `isl_set_coalesce` merges adjacent pieces, keeping representations compact.
- **Approximate analysis**: ISL provides `isl_set_simple_hull` (convex approximation) when exact computation is too expensive.
- **Bounded quantifier alternation**: ISL's schedule computation limits quantifier alternation to maintain tractability.

---

## Chapter Summary

- A **convex polyhedron** P = {x | Ax ≤ b} is the H-representation; the V-representation via vertices and rays is equivalent (Minkowski-Weyl). Faces, facets, and vertices are the geometric vocabulary of iteration space analysis.
- **Farkas' lemma** characterizes when linear constraints imply other linear constraints; it is the foundation of Feautrier's scheduling algorithm.
- **Integer lattices** are integer linear combinations of basis vectors. **Hermite Normal Form** canonicalizes integer matrices for solvability testing; **Smith Normal Form** diagonalizes them for dependence structure analysis.
- **Linear programming** finds an optimal solution in a polyhedron; the simplex method traverses vertices. **Fourier-Motzkin elimination** projects out variables, producing loop bound expressions.
- **Integer linear programming** is NP-hard in general but tractable for the small-dimensional loop nests that appear in practice; branch-and-bound and Gomory cuts are the standard methods.
- **Presburger arithmetic** — first-order integer arithmetic without multiplication — is decidable. **Cooper's quantifier elimination** and the **Omega test** are the two main practical algorithms; both appear inside ISL.
- **ISL** implements sets and relations over integer points as Presburger formulas, providing exact union/intersection/projection/QE operations plus parametric integer programming and polyhedral schedule computation. It is the algorithmic core of Polly and MLIR's Affine dialect.


---

@copyright jreuben11
