# Chapter 72 — Scheduling Algorithms

*Part XI — Polyhedral Theory*

The polyhedral model characterizes all legal loop transformations as schedules in a feasibility polyhedron. The scheduling problem is to find the *best* legal schedule according to a given objective: maximum parallelism, minimum communication, maximum tileability, or best cache behavior. This chapter derives three foundational scheduling algorithms from first principles — Feautrier's multidimensional scheduling, Lim-Lam's affine partitioning, and the Pluto algorithm — and then covers the ISL scheduler, iterative compilation, and the relationship between schedules and concrete loop transformations.

## 72.1 The Scheduling Framework

### 72.1.1 Problem Statement

Given a SCoP with statements S = {S₁, …, Sₘ} and dependence relations D = {dep(Sᵢ → Sⱼ)}, find an **affine schedule**:

```
θ = { θₛ : ℤⁿ → ℤᵈ | S ∈ S }
```

where each θₛ(I) = Θₛ·I + cₛ is affine, such that:

1. **Legality**: all dependences are respected (→ linear constraints on Θₛ, cₛ).
2. **Profitability**: some objective (parallelism, tileability, data locality) is optimized.

The legal schedules form a convex polyhedron in the space of schedule coefficients (Θₛ, cₛ). The scheduling problem is to find a point in this polyhedron that optimizes the objective.

### 72.1.2 The Legality Constraint (Farkas Application)

For each dependence dep(S1 → S2), the schedule must satisfy:

```
∀(I₁, I₂) ∈ dep(S1 → S2): θ_S2(I₂) − θ_S1(I₁) ≥ 0
```

(or ≥ 1 if we require strict temporal separation to avoid the same time step).

By the **Farkas lemma** (Chapter 70), this "∀ points in a polyhedron implies a linear constraint" can be converted to a *finite* system of linear constraints on the schedule coefficients:

```
∀ I ∈ dep(S1 → S2): (Θ_S2·I₂ + c_S2) − (Θ_S1·I₁ + c_S1) ≥ 0

By Farkas: there exists λ ≥ 0 such that:
  (Θ_S2·I₂ + c_S2) − (Θ_S1·I₁ + c_S1) = λᵀ·(A·[I₁;I₂] − b)
                                           (with dependence polyhedron Ax ≥ b)
```

This produces a finite system of linear constraints on λ and on the schedule coefficients Θₛ, cₛ. Solving this system (LP or ILP) yields a valid schedule.

## 72.2 Feautrier's Multidimensional Scheduling

### 72.2.1 Historical Context

Feautrier's algorithm (Feautrier, 1992a, 1992b) was the first *complete* polyhedral scheduling algorithm: it finds a multidimensional affine schedule that is guaranteed to exist for any SCoP (as long as the program terminates). The algorithm proceeds dimension by dimension, finding the lexicographically minimal schedule.

### 72.2.2 The Algorithm

**Phase 1: One-dimensional scheduling.**

Find the lexicographically minimal (earliest possible) schedule dimension θ₁ that satisfies all dependences:

```
minimize lex{ (Θ_S1·I₁, Θ_S2·I₂, …) }
subject to:
  (Farkas constraints from all dependences)
  Θₛ·I ≥ 0 for all I ∈ Dₛ  (non-negative time)
```

This is solved by **Parametric Integer Programming** (PIP, Chapter 70): find the lexicographically minimal integer-valued (Θ, c) satisfying the constraints as a function of the symbolic parameters N.

**Phase 2: Residual dependences.**

Some dependences may not be fully satisfied by θ₁ alone (they are "weakly" satisfied — θ₁ separates some but not all pairs). Project out the satisfied dependences and repeat for θ₂.

**Phase 3: Iterate** until all dependences are strongly satisfied (every dependent pair has a strictly greater schedule time for the later statement).

### 72.2.3 Example: Linear Recurrence

```c
for (int i = 1; i < N; i++)
  S: A[i] = A[i-1] + 1;   // dep: S[i-1] → S[i], distance = 1
```

Dependence relation: dep(S → S) = { (i, i+1) : 0 ≤ i < N-1 }.

Feautrier's phase 1: find θ_S(i) = α·i + β such that:
```
θ_S(i+1) − θ_S(i) ≥ 1   (strict separation, one-step recurrence)
α·(i+1) + β − α·i − β ≥ 1
α ≥ 1
```
Lexicographic minimum: α = 1, β = 0. Schedule: θ_S(i) = i. This is the identity (original loop order), which is optimal for a serial recurrence.

### 72.2.4 Example: Matrix Multiplication

```c
for (i=0; i<N; i++)
  for (j=0; j<N; j++)
    for (k=0; k<N; k++)
      S: C[i][j] += A[i][k] * B[k][j];
```

No loop-carried dependences in the inner loop body (the += is a reduction, but each (i,j) accumulates independently). Feautrier's algorithm finds:

- θ_S(i,j,k) = (i, j, k): the identity schedule.
- The k-loop is serial (reduction), i and j loops are fully parallel.

This is the correct result: all (i,j) iterations are independent; only the k-loop carries the dependence through the accumulation.

### 72.2.5 Feautrier's Properties

| Property | Value |
|----------|-------|
| Completeness | Always finds a schedule if one exists |
| Schedule dimensions | At most 2n+1 for depth-n loop nests |
| Optimality | Lexicographically minimal (greedy per dimension) |
| Complexity | PIP calls per dimension; exponential in worst case |
| Implemented in | Candl, Clan (standalone tools); ISL has a Feautrier-mode scheduler |

## 72.3 Lim-Lam Affine Partitioning

### 72.3.1 Affine Partitioning for Parallelism

Lim and Lam (1997) developed a scheduling framework specifically targeting **maximum parallelism** — finding the minimal number of sequential dimensions and maximizing the dimensionality of the parallel portion.

The key insight: a loop dimension l is **parallel** if all dependences project to zero along that dimension. The problem is to find an affine transformation that rotates the iteration space to maximize the number of zero-projection dimensions.

### 72.3.2 The Lim-Lam Schedule

For a single loop nest with uniform dependences d₁, d₂, …, dₖ, the Lim-Lam schedule finds a partitioning matrix P such that:

```
P · dᵢ = 0   for as many dᵢ as possible  (parallel directions)
P · dᵢ > 0   for remaining dᵢ             (serial directions)
```

This is a null-space computation over the dependence vectors: the parallel directions are those orthogonal to all dependence vectors.

For matrix multiplication: dependence vectors = {(0,0,1)} (k-direction only). Null space: all vectors orthogonal to (0,0,1), i.e., any (a,b,0). The i and j loops are fully parallel; k is the serial dependence dimension.

### 72.3.3 Affine Partitions and Wavefront Parallelism

When no perfectly parallel dimension exists (the dependence cone has full rank), Lim-Lam finds **wavefront** parallelism: a skewed schedule that creates diagonal hyperplanes of simultaneously-executable iterations:

```c
// Original: recurrence dep (i,j) → (i+1,j-1)
for (i = 0; i < N; i++)
  for (j = 0; j < N; j++)
    A[i][j] = f(A[i-1][j+1]);

// Wavefront schedule: θ(i,j) = i+j (wavefront index), then i within wavefront
// Parallel: all (i,j) with i+j = c execute simultaneously
// Schedule: θ_S(i,j) = (i+j, i)
```

The skewed outer dimension `i+j` is sequential (must increase by 1 per step); the inner dimension `i` is parallel (within each wavefront).

## 72.4 The Pluto Algorithm

### 72.4.1 Motivation

Feautrier's algorithm finds the lexicographically earliest schedule but does not optimize for **data locality** or **tileability**. A loop nest that is maximally parallel but has non-local memory access patterns will underperform on modern hardware with cache hierarchies.

The **Pluto algorithm** (Bondhugula et al., PLDI 2008) is a framework for finding schedules that simultaneously maximize:
- **Tileability** (permutability): the number of loop levels that can be legally tiled.
- **Parallelism**: the number of parallel outermost loops.
- **Data locality**: the pattern of memory accesses within tiles.

Pluto became the reference algorithm for polyhedral scheduling in compilers, implemented in Polly, OpenStream, PolyMage, and MLIR.

### 72.4.2 Tileability as a Constraint

A schedule dimension l is **tileable** (enables hyperplane tiling) if:

```
∀ dependences dep(S1 → S2): θ_S2(I₂)_l − θ_S1(I₁)_l ≥ 0
```

This is the **permutability** or **non-negative dependence distance** condition: every dependence has a non-negative l-th component in the new schedule. Such a dimension can be tiled because within a tile, the dependences point outward (from lower tile coordinates to higher ones).

Maximizing the number of tileable dimensions is equivalent to finding the maximal subset of schedule dimensions satisfying the non-negativity constraint simultaneously for all dependences.

### 72.4.3 The Pluto Cost Function

Pluto's objective: find a schedule that minimizes a linear cost function capturing dependence distances. Specifically, for each dependence dep(S1[I₁] → S2[I₂]) and schedule dimension l, the **dependence distance** is:

```
u_dep_l = θ_S2(I₂)_l − θ_S1(I₁)_l ∈ ℤ
```

Pluto minimizes the sum of dependence distances:

```
minimize Σ_dep Σ_l u_dep_l
```

subject to:
- Legality: u_dep_l ≥ 0 for all dep, l (permutability)
- At least one u_dep_l ≥ 1 for each dependence dep (dependence is "carried" by some dimension)
- Schedule coefficients are bounded: |Θₛ[k]| ≤ B (for some bound B, typically 3-5)

### 72.4.4 Derivation of Pluto's LP

For each schedule dimension, Pluto constructs a linear program over the schedule coefficient vectors (uₛ)_S ∈ S:

```
For statement Sᵢ: θ_S(I) = uᵢᵀ · I + w_i   (row of schedule matrix)
```

The legality constraint for dependence dep(S1 → S2) at dimension l:

```
∀(I₁, I₂) ∈ dep: (u_S2 · I₂ + w_S2) − (u_S1 · I₁ + w_S1) ≥ 0
```

By Farkas' lemma, this becomes:

```
∃λ ≥ 0: u_S2 · I₂ − u_S1 · I₁ + (w_S2 − w_S1) 
       = λᵀ · (A · [I₁; I₂] − b) + u_dep
```

where [A, b] defines the dependence polyhedron and u_dep ≥ 0 is the dependence distance variable. This introduces new non-negative variables λ (Farkas multipliers) and u_dep.

The resulting LP is:

```
minimize  Σ_dep u_dep
subject to:
  λ ≥ 0                    (Farkas positivity)
  u_dep ≥ 0                (non-negative distance)
  Farkas equations satisfied (finite # constraints)
  u_dep ≥ 1 for at least one dep (carrying constraint)
```

This is solved by an ILP solver (Pluto uses PIP or GLPK). The solution gives one row of the schedule matrix for all statements simultaneously.

### 72.4.5 The Pluto++ Extensions

The original Pluto LP has limitations: it enforces the same u_dep for all instances of a dependence (uniform distance). Real dependences may have variable distances. Several extensions address this:

- **Pluto+**: handles non-uniform dependences by parameterizing u_dep.
- **MaxFuse**: modifies the cost function to prefer loop fusion over distribution.
- **PoCC (Polyhedral Compiler Collection)**: a framework implementing multiple Pluto variants.
- **ISL scheduler**: re-implements Pluto's approach using ISL's exact Presburger arithmetic, avoiding the bounding heuristic.

### 72.4.6 Example: Matrix-Vector Multiplication

```c
for (int i = 0; i < M; i++)
  for (int j = 0; j < N; j++)
    y[i] += A[i][j] * x[j];

// Dependences: none between iterations (pure reduction, data-independent)
// Pluto schedule: θ(i,j) = (i, j) — identity; all iterations are parallel
// After tiling: θ(i,j) = (i/T, j/T, i%T, j%T)
// With tiling: outer (ii,jj) loops parallel, inner (i',j') loops serial
```

### 72.4.7 Example: Jacobi 2D Stencil

```c
for (int t = 0; t < T; t++)
  for (int i = 1; i < N-1; i++)
    for (int j = 1; j < N-1; j++)
      B[i][j] = 0.25 * (A[i-1][j] + A[i+1][j] + A[i][j-1] + A[i][j+1]);
// Then A = B;

// Dependences (ignoring copy for clarity):
// S[t,i,j] → S[t+1,i,j]    (flow in t)
// S[t,i,j] → S[t+1,i-1,j]  (flow in i)
// S[t,i,j] → S[t+1,i+1,j]  (anti in i)
// ... (the full stencil dependence pattern)

// Pluto finds: θ(t,i,j) = (i+j+t, t, i)  (time-skewed tiling enables parallelism)
// → wavefront parallelism along the i+j+t hyperplane
```

The Pluto schedule for 2D stencils typically produces the diamond-tiling pattern: tiles that are skewed in the time dimension, enabling parallel execution of tiles on the same wavefront.

## 72.5 Affine Transformations as Schedules

### 72.5.1 Catalogue of Transformations

Every classical loop transformation is a special case of an affine schedule:

| Transformation | Schedule Θ | Effect |
|---------------|-----------|--------|
| Identity | Θ = I (identity) | Original loop order |
| Reversal (loop l) | Θ: row l negated | Reverse loop l |
| Interchange (l₁, l₂) | Θ: swap rows l₁, l₂ | Exchange loops |
| Skewing | Θ: Θ[l₂,l₁] = c (off-diagonal) | j → j + c·i |
| Tiling | Θ: insert ⌊xᵢ/T⌋ and xᵢ mod T dimensions | Block iteration |
| Strip-mining | Θ: add ⌊xᵢ/S⌋ dimension | Split loop into strips |
| Fusion | Same θ row for two statements | Merge loops |
| Distribution | Different θ rows for statements in fused loop | Split loop |
| Parallelization | Mark outer dimension as parallel | `#pragma omp parallel for` |

### 72.5.2 Skewing for Tileability

When a loop nest is not directly tileable (dependences have negative components in some dimension), skewing can make it tileable:

```
Original dependences: d₁ = (1, -1, 0)   (time i+1 depends on time i, but j-1 on j)
→ Not tileable: d₁ has a negative j-component.

Skewing schedule: Θ = [[1, 0, 0], [1, 1, 0], [0, 0, 1]]
                          i' = i, j' = i + j, k' = k

Under Θ: d₁' = Θ·d₁ = (1, 0, 0)   → non-negative!
Now tileable in both dimensions.
```

This is the key insight behind **diamond tiling** of stencils: skewing makes the time dimension tileable, enabling spatial-temporal tiling that reuses cache contents across time steps.

## 72.6 The ISL Scheduler

### 72.6.1 Architecture

ISL's scheduler (`isl_schedule_constraints_compute_schedule`) implements a Pluto-style algorithm using ISL's exact Presburger arithmetic:

```c
// ISL scheduling API
isl_schedule_constraints *sc = isl_schedule_constraints_on_domain(domain);
sc = isl_schedule_constraints_set_validity(sc, validity_deps);   // must-respect
sc = isl_schedule_constraints_set_coincidence(sc, coincidence_deps);  // for parallelism
sc = isl_schedule_constraints_set_proximity(sc, proximity_deps); // for locality
isl_schedule *schedule = isl_schedule_constraints_compute_schedule(sc);
```

Three types of constraints:
- **Validity**: dependences that *must* be respected (legality).
- **Coincidence**: dependences that should be *zero-distance* if possible (parallelism: make iterations with zero dependence execute at the same time).
- **Proximity**: dependences that should have *small distance* (locality: keep producer and consumer close in time).

### 72.6.2 Band Trees

ISL's scheduler produces **schedule trees** rather than flat schedules. A schedule tree is a tree of **band nodes**, each representing a group of dimensions that can be independently optimized:

```
Band {i, j} with schedule (i+j, i):
  Sequence:
    Leaf S1
    Leaf S2
```

Band nodes carry the tileable/coincident flags per dimension. The **isl_band_tile** operation tiles a band node:

```c
isl_multi_val *tile_sizes = isl_multi_val_read_from_str(ctx, "{ [32, 32] }");
isl_schedule_node *tiled = isl_schedule_node_band_tile(band_node, tile_sizes);
```

Schedule trees are the representation consumed by `isl_ast_build` (Chapter 73) for code generation.

### 72.6.3 Options and Heuristics

The ISL scheduler exposes control over the optimization:

```c
isl_options_set_schedule_algorithm(ctx, ISL_SCHEDULE_ALGORITHM_FEAUTRIER);
// or
isl_options_set_schedule_algorithm(ctx, ISL_SCHEDULE_ALGORITHM_ISL);  // Pluto-style

isl_options_set_schedule_maximize_band_depth(ctx, 1);   // prefer deeper bands
isl_options_set_schedule_outer_coincidence(ctx, 1);     // prefer outer parallelism
```

The default ISL algorithm is Pluto-style (minimize dependence distances, maximize band depth and coincidence). The Feautrier mode finds the lexicographically earliest schedule.

## 72.7 Iterative Compilation and Autotuning

### 72.7.1 Limitations of Static Scheduling

Static scheduling algorithms like Pluto produce good schedules for the abstract cost model (minimize dependence distances, maximize permutability) but cannot account for:
- Cache size and line sizes (hardware-specific).
- TLB pressure (very large tile sizes may cause TLB misses).
- Vectorization constraints (SIMD width must divide tile sizes).
- Actual runtime behavior on specific inputs.

### 72.7.2 Iterative Compilation

**Iterative compilation** (Kisuki et al., 2000) explores the schedule space empirically:

```
1. Generate multiple schedule candidates (varying tile sizes, loop order, fusion choices).
2. Compile each candidate.
3. Execute on representative inputs and measure performance.
4. Select the best-performing schedule.
```

The PoCC (Polyhedral Compiler Collection) framework implements iterative compilation on top of the polyhedral model. The search space is parametric: tile sizes (T₁, T₂, …) are treated as variables, and performance is measured as a function of (T₁, T₂).

### 72.7.3 Autotuning with Polyhedral Models

Modern autotuning frameworks (e.g., **OpenTuner**, **Halide's autoscheduler**, **TVM's AutoTVM**) use the polyhedral model to define the *space* of valid transformations and a search strategy to navigate it:

- **Grid search**: evaluate all tile size combinations in a grid.
- **Bayesian optimization**: model performance as a Gaussian process, select promising configurations.
- **Genetic algorithms**: evolve schedule parameters across generations.
- **Reinforcement learning**: train a policy to select schedules (the connection to ML compilers discussed in Chapter 66).

The polyhedral model ensures that all candidates evaluated are *legal* (dependences preserved), so autotuning can focus entirely on performance rather than correctness.

---

## Chapter Summary

- The **scheduling problem** is to find an affine schedule θₛ: ℤⁿ → ℤᵈ for each statement that satisfies all dependence legality constraints (derived via the Farkas lemma) while optimizing a given objective.
- **Feautrier's algorithm** (1992) finds the lexicographically minimal multidimensional affine schedule using Parametric Integer Programming; it is complete (always finds a schedule if one exists) but does not optimize for locality.
- **Lim-Lam affine partitioning** (1997) maximizes the dimensionality of the parallel portion by finding directions orthogonal to all dependence vectors; for non-trivially-parallel nests it finds wavefront schedules via skewing.
- **Pluto** (Bondhugula et al., PLDI 2008) minimizes dependence distances subject to the Farkas legality constraints, maximizing tileability and parallelism simultaneously; it solves an ILP per schedule dimension using PIP.
- Every classical loop transformation — interchange, reversal, skewing, tiling, strip-mining, fusion, distribution — is a special case of an affine schedule change; their legality is verified by checking the schedule against the dependence polyhedra.
- The **ISL scheduler** reimplements Pluto-style optimization using exact Presburger arithmetic, producing **schedule trees** (band nodes with tileable/coincident annotations) as output; these are consumed by `isl_ast_build` for code generation.
- **Iterative compilation** and autotuning search the schedule parameter space empirically; the polyhedral model provides the *feasibility guarantee* (all candidates are legal) while the search optimizes actual runtime performance.


---

@copyright jreuben11
