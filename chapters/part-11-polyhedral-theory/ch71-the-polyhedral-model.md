# Chapter 71 — The Polyhedral Model

*Part XI — Polyhedral Theory*

The polyhedral model is a mathematical framework for reasoning about programs whose control flow and memory accesses are expressible as affine functions of loop induction variables and symbolic constants. Within this model, a loop nest becomes a polyhedron (the iteration space), each array access becomes an affine map from iteration space to memory space, and the dependences between statements become integer relations between polyhedra. Transformations — loop tiling, fusion, interchange, parallelization — are legal changes to the iteration order that preserve all dependence relations. This chapter develops the polyhedral model rigorously: the Static Control Part (SCoP) abstraction, iteration spaces, access functions, dependence polyhedra, and the affine schedule space that encompasses all valid loop transformations.

## 71.1 The Static Control Part (SCoP)

### 71.1.1 Definition

A **Static Control Part (SCoP)** (Girbal et al., 2006; Grosser et al., 2012) is a maximal program region whose control flow and memory accesses can be represented exactly as affine functions of the surrounding loop induction variables and symbolic parameters.

Formally, a SCoP consists of:
- A set of **loop induction variables** I = (i₁, i₂, …, iₙ) defining the iteration space.
- A set of **symbolic parameters** N = (N₁, N₂, …, Nₖ) — compile-time-unknown but execution-time-constant values (loop bounds, matrix dimensions).
- A set of **statements** S = {S₁, S₂, …, Sₘ}, each with:
  - A **domain** Dₛ ⊆ ℤⁿ — the set of integer (i₁, …, iₙ) for which Sₛ executes.
  - A set of **access functions** Aₛ,ᵣ: ℤⁿ → ℤᵐ — maps from iteration to array index.
  - A **schedule** θₛ: ℤⁿ → ℤᵈ — the temporal order in which Sₛ's iterations execute.

### 71.1.2 The Affine Restriction

The defining restriction: every domain constraint, every access function component, and every schedule component must be an **affine function** of (I, N):

```
f(I, N) = c₀ + c₁i₁ + … + cₙiₙ + d₁N₁ + … + dₖNₖ
```

where all cᵢ and dⱼ are integer constants known at compile time.

This restriction excludes:
- **Non-linear loop bounds**: `for (i = 0; i < i*i; i++)` — not affine.
- **Data-dependent control flow**: `if (A[i] > 0) ...` — depends on runtime data.
- **Indirect array accesses**: `A[B[i]]` — not an affine function of i.
- **Function calls with side effects** (unless proven pure).

### 71.1.3 SCoP Examples

A matrix multiplication SCoP:

```c
// SCoP: loops are affine, accesses are affine, no data-dependent control
for (int i = 0; i < N; i++)       // i ∈ [0, N)
  for (int j = 0; j < N; j++)     // j ∈ [0, N)
    for (int k = 0; k < N; k++)   // k ∈ [0, N)
      C[i][j] += A[i][k] * B[k][j];

// Statements:
// S1: C[i][j] += A[i][k] * B[k][j]  (the only statement)

// Domain: D_S1 = { [i,j,k] : 0 ≤ i < N, 0 ≤ j < N, 0 ≤ k < N }
// Access functions:
//   Read  A[i][k]:  [i,j,k] → [i, k]       (affine)
//   Read  B[k][j]:  [i,j,k] → [k, j]       (affine)
//   R/W   C[i][j]:  [i,j,k] → [i, j]       (affine)
```

A triangular loop SCoP with non-rectangular domain:

```c
for (int i = 0; i < N; i++)
  for (int j = 0; j < i; j++)   // j bound depends on i — still affine!
    A[i][j] = A[j][i];

// Domain: D = { [i,j] : 0 ≤ i < N, 0 ≤ j < i }
// This is a non-rectangular polyhedron, but still a valid SCoP.
```

A non-SCoP (data-dependent control):

```c
for (int i = 0; i < N; i++)
  if (A[i] > 0)           // condition depends on A[i] — not affine
    B[i] = A[i] * 2;
// NOT a SCoP: the condition A[i] > 0 is not an affine function of i.
```

## 71.2 Iteration Spaces as Polyhedra

### 71.2.1 The Iteration Space

For a single-statement loop nest with induction variables I = (i₁, …, iₙ), the **iteration space** is:

```
D = { I ∈ ℤⁿ | A·I + B·N ≥ 0 }
```

where A and B are integer matrices encoding the loop bounds. Each constraint corresponds to one bound on one induction variable.

For the matrix multiplication example:
```
D_S1 = { [i,j,k] : i ≥ 0, i < N, j ≥ 0, j < N, k ≥ 0, k < N }
     = { [i,j,k] : 
         [1 0 0][i]   [0]        [0 0 0][i]   [0]
         [0 1 0][j] ≥ [0]  and   [0 0 0][j] + [N][N] > [0] }
         [0 0 1][k]   [0]        [0 0 0][k]   [0]    [0]
```

This is a 3-dimensional box polytope with 6 faces.

### 71.2.2 Parametric Polytopes

When loop bounds involve symbolic parameters N, the domain is a **parametric polytope**: a family of polyhedra, one for each value of the parameter vector N. For N = 4, D_S1 is a 4×4×4 cube; for N = 100, it is a 100×100×100 cube.

ISL represents parametric polytopes natively: `[N] -> { [i,j,k] : 0 <= i,j,k < N }`.

### 71.2.3 Multiple Statements

When the SCoP has multiple statements, each has its own iteration domain. The **union of iteration spaces** is the set of all (statement, iteration) pairs:

```
IS = ∪ₛ { (s, I) | I ∈ Dₛ }
```

In ISL, this is an `isl_union_set` where each piece is tagged with the statement name.

## 71.3 Access Functions

### 71.3.1 Definition

An **access function** for array reference A[f₁(I), f₂(I), …, fₘ(I)] is the affine map:

```
α: ℤⁿ → ℤᵐ,  α(I) = M·I + c
```

where M ∈ ℤ^{m×n} is the **access matrix** and c ∈ ℤᵐ is the **offset vector**.

For `A[i][j]` in a loop with variables (i, j, k): M = [[1,0,0],[0,1,0]], c = [0,0].
For `A[i+1][j-1]`: M = [[1,0,0],[0,1,0]], c = [1,-1].
For `A[2i+3][j]`: M = [[2,0,0],[0,1,0]], c = [3,0].

### 71.3.2 The Access Relation

The **access relation** for a statement S and array reference r is:

```
Rₛ,ᵣ = { (I, α(I)) | I ∈ Dₛ } ⊆ ℤⁿ × ℤᵐ
```

This is a many-to-one map from iterations to memory locations: if the map is not injective, multiple iterations access the same memory location (potential dependence).

In ISL: `{ S[i,j] -> A[i, j] : 0 <= i,j < N }` for `A[i][j]`.

### 71.3.3 Image Computation

The **image** of a set S under relation R is the set of all memory locations accessed by iterations in S:

```
image(S, Rₛ,ᵣ) = { α(I) | I ∈ S ∩ Dₛ }
```

Image computation is used to determine which array elements are written by a statement (for alias analysis) and which are read (for reuse analysis). In ISL: `isl_map_range`.

The **preimage** (or inverse image) asks which iterations access a given memory location:

```
preimage(L, Rₛ,ᵣ) = { I ∈ Dₛ | α(I) ∈ L }
```

Preimage computation is used to find all iterations that write to a given memory location before a given read — the core of dependence analysis.

## 71.4 Dependence Analysis

### 71.4.1 Types of Dependences

Given two statements S1 (writing) and S2 (reading or writing) and their iteration domains D1, D2:

| Dependence | Condition |
|-----------|-----------|
| **Flow (RAW)** | S1 writes then S2 reads the same location: write(S1) ∩ read(S2) ≠ ∅ |
| **Anti (WAR)** | S1 reads then S2 writes the same location: read(S1) ∩ write(S2) ≠ ∅ |
| **Output (WAW)** | S1 writes then S2 writes the same location: write(S1) ∩ write(S2) ≠ ∅ |
| **Input (RAR)** | S1 reads and S2 reads the same location (not a true dependence, but for reuse analysis) |

All dependences must be **carried** by the lexicographic order of iterations: if iteration I₁ of S1 must execute before iteration I₂ of S2 (because S1 writes then S2 reads), then I₁ ≺_lex I₂ in the original loop nest.

### 71.4.2 Dependence Polyhedra

A **dependence polyhedron** for a flow dependence from S1 at I₁ to S2 at I₂ is the set of pairs (I₁, I₂) such that S1's write at I₁ is followed (in lexicographic order) by S2's read of the same location:

```
dep(S1 → S2) = { (I₁, I₂) ∈ D1 × D2 | 
                  α_write(S1, I₁) = α_read(S2, I₂)  AND
                  (I₁, I₂) ≺_lex (I₂, I₁) is ordered by original schedule }
```

Concretely, for two statements in a loop nest with shared loop variable i:

```c
for (int i = 0; i < N; i++) {
    S1: A[i] = A[i-1] + 1;  // write A[i], read A[i-1]
    S2: B[i] = A[i] * 2;    // read A[i]
}

// Flow dependence S1[i] → S2[i]: both access A[i], S1 writes before S2 reads
dep(S1 → S2) = { (i, i) : 0 ≤ i < N }   (self-dependence at same i)

// Flow dependence S1[i] → S1[i+1]: S1 writes A[i], next iteration reads A[(i+1)-1]=A[i]
dep(S1 → S1) = { (i, i+1) : 0 ≤ i < N-1 }  (loop-carried dependence)
```

### 71.4.3 Exact vs. Approximate Dependences

**Exact dependence analysis** computes the precise set of dependence pairs using Presburger arithmetic (ISL's `isl_union_map_compute_flow`). This can be expensive.

**Approximate dependence analysis** over-approximates by testing only whether the dependence polyhedron is satisfiable (not computing the exact set):
- **GCD test**: checks if the GCD of access function coefficients divides the constant difference. Fast, may report false positives.
- **Banerjee inequalities**: linear bounds on the dependence distance. Fast, may report false positives.
- **Omega test**: exact integer satisfiability test (Chapter 70). Exact, polynomial for fixed loop depth.

Polly uses ISL for exact dependence analysis within SCoPs, falling back to conservative over-approximation for non-affine accesses.

### 71.4.4 Dependence Distance Vectors

For a loop-carried dependence from S1[I₁] to S2[I₂], the **dependence distance vector** is:

```
d = I₂ − I₁ ∈ ℤⁿ
```

If d is the same for all pairs (constant distance), the dependence is **uniform**. Non-uniform dependences have d varying over the dependence polyhedron.

A dependence is **lexicographically positive** if d ≻_lex 0 (the direction of time). A transformation is legal if and only if all dependences remain lexicographically positive under the new iteration order.

```
// Uniform dependence: d = (0, 1) for the S1[i][j] → S2[i][j+1] dependence
for (int i = 0; i < N; i++)
  for (int j = 0; j < N-1; j++)
    S1: A[i][j+1] += A[i][j];  // dependence distance d = (0, 1) — always
```

## 71.5 The Affine Schedule Space

### 71.5.1 Schedules

A **schedule** for a SCoP assigns a **logical execution time** to each statement instance (I, S). In the polyhedral model, schedules are affine maps:

```
θₛ: ℤⁿ → ℤᵈ,  θₛ(I) = Θₛ·I + cₛ
```

where d is the **schedule dimension** (the number of independent time components, each representing a level of the loop nest hierarchy).

For a one-dimensional schedule (one-level loop):
```
θ_S1(i) = 2i    (execute S1 at logical time 2i)
θ_S2(i) = 2i+1  (execute S2 at logical time 2i+1, after S1)
```

For a two-dimensional schedule (matrix mult. with tiling):
```
θ_S(i,j,k) = (i/T, j/T, k, i mod T, j mod T)
            = (⌊i/T⌋, ⌊j/T⌋, k, i%T, j%T)   (tile coordinates, then point coordinates)
```

### 71.5.2 Schedule Legality

A schedule is **legal** if it respects all dependences: for every flow dependence (I₁, S1) → (I₂, S2), the schedule assigns an earlier time to I₁ than to I₂:

```
Legality condition:
  ∀(I₁, I₂) ∈ dep(S1 → S2): θ_S1(I₁) ≺_lex θ_S2(I₂)
```

Equivalently, for each dependence relation R_{S1→S2}, the composed schedule difference must be lexicographically positive:

```
(θ_S2 ∘ π₂ − θ_S1 ∘ π₁)(dep) ≥_lex 0
```

where π₁ and π₂ project onto the source and destination iterations respectively.

The Farkas lemma (Chapter 70) converts this into a system of linear constraints on the schedule coefficients Θₛ: the schedule must satisfy all these constraints simultaneously, and any valid solution is a legal schedule.

### 71.5.3 The Affine Schedule Space

All legal schedules form the **affine schedule space**: the feasibility region of the linear system derived from all dependence constraints via the Farkas lemma. This space is a convex polyhedron in the space of schedule coefficients.

The polyhedral scheduling problem is: find a schedule in this feasibility region that optimizes a given cost function (typically: maximize parallelism, minimize data movement, enable tiling). This is the subject of Chapter 72.

### 71.5.4 Multidimensional Schedules and Statement Order

A full schedule for a SCoP is a sequence of schedule dimensions:

```
θ = (θ₁, θ₂, …, θ_d)
```

where each θₗ is a (possibly statement-specific) affine map. Iteration (I, S) is executed at logical time (θ₁_S(I), θ₂_S(I), …, θ_d_S(I)) in lexicographic order.

The original loop nest corresponds to the **identity schedule**: θₛ(i₁, …, iₙ) = (i₁, …, iₙ). Any other schedule that satisfies the legality condition is a valid reordering.

## 71.6 Transformations as Schedule Changes

### 71.6.1 Loop Interchange

**Loop interchange** swaps loops i and j:

```c
// Original:                         // After interchange:
for (i = 0; i < N; i++)             for (j = 0; j < N; j++)
  for (j = 0; j < N; j++)             for (i = 0; i < N; i++)
    A[i][j] = B[j][i];                  A[i][j] = B[j][i];
```

As a schedule change: θ_S(i, j) = (i, j) → θ'_S(i, j) = (j, i).

Legal if and only if there is no loop-carried dependence with distance d such that the interchange would reverse the direction: no (d_i, d_j) in the dependence cone with d_i > 0 and d_j < 0 simultaneously.

### 71.6.2 Loop Tiling

**Loop tiling** (blocking) partitions the iteration space into rectangular tiles of size T:

```c
// Original (2D iteration space):
for (i = 0; i < N; i++)
  for (j = 0; j < N; j++)
    C[i][j] += A[i][k] * B[k][j];

// After tiling (tile size T):
for (ii = 0; ii < N; ii += T)       // tile coordinates
  for (jj = 0; jj < N; jj += T)
    for (i = ii; i < min(ii+T, N); i++)   // point coordinates
      for (j = jj; j < min(jj+T, N); j++)
        C[i][j] += A[i][k] * B[k][j];
```

As a schedule: θ'_S(i, j) = (⌊i/T⌋, ⌊j/T⌋, i mod T, j mod T).

Tiling is legal if the loop nest is **tileable**: all dependences have non-negative projections onto the tiled dimensions (the distance cone lies in the positive orthant). Tileable loops are called **permutable**.

### 71.6.3 Loop Skewing

**Loop skewing** adds a multiple of an outer loop variable to an inner loop variable:

```c
// Original:                          // After skewing (j ← j + i):
for (i = 0; i < N; i++)              for (i = 0; i < N; i++)
  for (j = 0; j < N; j++)              for (j = i; j < N+i; j++)
    S[i][j];                              S[i][j-i];
```

Skewing transforms non-rectangular dependence vectors into rectangular ones, enabling tiling of originally non-tileable loops. Schedule: θ'_S(i, j) = (i, i + j).

### 71.6.4 Loop Fusion and Distribution

**Loop fusion** merges two consecutive loops with the same iteration space into one:

```c
// Before:                            // After fusion:
for (i = 0; i < N; i++) S1[i];       for (i = 0; i < N; i++) { S1[i]; S2[i]; }
for (i = 0; i < N; i++) S2[i];
```

In schedule terms: assign the same schedule to S1 and S2's i-th iterations.

**Fusion is legal** if it does not introduce new dependences: no flow dependence S2[i] → S1[j] with j > i (which would require S1 at j to execute after S2 at i, violating the fused order).

**Loop distribution (fission)** is the inverse: split one loop into two with the same domain. Always legal as long as dependences remain satisfied.

## 71.7 Parallelism and Tileability

### 71.7.1 Parallelism Conditions

A loop dimension is **parallel** if removing it from the schedule does not violate any dependence — all dependences have zero component in that dimension:

```
Dimension l is parallel for statement S if:
  ∀(I₁, I₂) ∈ dep(S → S): (θ_S(I₂) − θ_S(I₁))_l = 0
```

Equivalently: no dependence has a non-zero l-th component.

The parallelism analysis reduces to a Presburger formula: "for all pairs satisfying the dependence relation, the l-th schedule component difference is zero."

### 71.7.2 The Permutability Condition

A set of loops is **permutable** (the group can be freely reordered without violating dependences) if the dependence distance cone lies in the positive orthant for all dimensions:

```
Permutability: ∀ dep (I₁ → I₂): ∀l ∈ 1..d: θ_S(I₂)_l − θ_S(I₁)_l ≥ 0
```

Permutable loops can be legally tiled, enabling cache-friendly access patterns and parallelization.

The Pluto algorithm (Chapter 72) finds schedules that maximize the number of permutable dimensions — making as many loops as possible legally tileable and parallel.

---

## Chapter Summary

- A **SCoP** (Static Control Part) is a maximal program region where loop bounds and array accesses are affine functions of induction variables and symbolic parameters; it is the input domain of the polyhedral model.
- The **iteration space** of each statement is a parametric polyhedron D ⊆ ℤⁿ; the union of all statement iteration spaces is representable as an `isl_union_set`.
- **Access functions** α: ℤⁿ → ℤᵐ are affine maps from iteration to memory index; the access relation is the set {(I, α(I)) | I ∈ D}.
- **Dependences** are integer relations between iteration spaces: a flow dependence (RAW) requires that for all (I₁, I₂) in the dependence relation, I₁ executes before I₂ in the original schedule.
- **Dependence polyhedra** represent the complete set of dependent iteration pairs; exact analysis uses ISL's `isl_union_map_compute_flow` via the Omega test; approximate analysis uses the GCD test or Banerjee inequalities.
- **Schedules** θ_S: ℤⁿ → ℤᵈ assign logical execution times to statement instances; a schedule is legal if all dependences remain lexicographically positive after the schedule change.
- The **affine schedule space** — the feasibility region of the Farkas-lemma-derived linear system on schedule coefficients — contains all legal schedules; optimizing over it (for parallelism, locality, tileability) is the polyhedral scheduling problem addressed in Chapter 72.
- Loop transformations (interchange, tiling, skewing, fusion, distribution) are all representable as schedule changes; their legality reduces to checking whether the new schedule remains in the affine schedule space.


---

@copyright jreuben11
