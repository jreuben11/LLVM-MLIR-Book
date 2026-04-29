# Chapter 73 — Code Generation from Polyhedral Schedules

*Part XI — Polyhedral Theory*

A polyhedral schedule is an abstract mathematical object: a multidimensional affine map from iteration space to time. To execute a program, this abstract schedule must be materialized into loops, conditionals, and array accesses in a target language (C, LLVM IR, or an MLIR dialect). The code generation problem — given a polyhedral schedule, produce a correct and efficient loop nest — is non-trivial: loop bounds may be complex expressions involving floor and ceiling functions, iteration order may require conditionals to handle boundary cases, and tiles may have irregular shapes near the boundaries of the domain. This chapter covers the CLooG algorithm, ISL's AST generator, loop bound generation via Fourier-Motzkin elimination, tiling strategies, OpenMP code generation, SIMD-aware codegen, and GPU mapping.

## 73.1 The Code Generation Problem

### 73.1.1 From Schedule to Loops

Given a polyhedral schedule θ_S: ℤⁿ → ℤᵈ for each statement S, the code generator must produce a loop nest that:

1. Executes each statement instance (S, I) exactly once (completeness).
2. Executes them in the order determined by θ_S (correctness).
3. Groups computationally related iterations together (efficiency).

The schedule θ partitions the set of all statement instances into time steps — sets of instances sharing the same schedule vector. The code generator must enumerate these time steps in lexicographic order and, within each time step, enumerate the statement instances.

### 73.1.2 Challenges

The code generation problem is challenging because:

- **Non-rectangular domains**: iteration spaces may be triangular, trapezoidal, or more complex polyhedra. The loop bounds must express all constraints, resulting in floor/ceiling expressions.
- **Tiled iteration spaces**: tiling introduces new loop variables (tile coordinates) with bounds that depend on both the tile size and the domain boundary.
- **Multiple statements**: different statements may have different domains, requiring conditional guards (`if (condition)`) inside generated loops.
- **Symbolic parameters**: loop bounds may involve parameters N, M known only at runtime, requiring parametric bound computation.

### 73.1.3 History: CLooG

The **CLooG** (Chunky Loop Generator) tool (Bastoul, 2004) was the first practical polyhedral code generator, widely adopted by GCC-Graphite, Pluto, and early Polly. CLooG uses the **scanning algorithm**: enumerate all integer points in the image of the schedule map by interleaving loop construction with domain scanning.

ISL's AST generator (`isl_ast_build`) is the modern replacement, supporting richer schedule tree representations and better handling of tiling, fusion, and distributed loops.

## 73.2 Loop Bound Generation

### 73.2.1 Projecting Out Variables via Fourier-Motzkin

The innermost loop variable's bounds are derived by **Fourier-Motzkin elimination** (FME, Chapter 70): project the polyhedron onto all dimensions except the innermost loop, collecting lower and upper bounds.

For statement domain D = { [i, j] | 0 ≤ i < N, 0 ≤ j ≤ i }:

```
Bounds on j (given i): 0 ≤ j ≤ i
→ generated code: for (j = 0; j <= i; j++)

Then bounds on i (projecting out j): 0 ≤ i < N
→ generated code: for (i = 0; i < N; i++)
```

This produces the correct triangular loop:

```c
for (int i = 0; i < N; i++)
  for (int j = 0; j <= i; j++)
    S[i][j];
```

FME can produce exponentially many constraints in the worst case, but for typical SCoPs with small loop depth, the result is tractable.

### 73.2.2 Maximum and Minimum Expressions

When FME produces multiple lower bounds or multiple upper bounds, the code generator must take the maximum of lower bounds and minimum of upper bounds:

```
Lower bounds: { j ≥ 0, j ≥ i−N }  →  j_lb = max(0, i−N)
Upper bounds: { j ≤ i, j ≤ N−1 }  →  j_ub = min(i, N−1)
```

Generated code:

```c
for (j = max(0, i - N); j <= min(i, N-1); j++)
```

ISL generates `isl_ast_expr` nodes representing these max/min expressions, which are lowered to `llvm.smax`/`llvm.smin` intrinsics or conditional expressions in the target code.

### 73.2.3 Floor and Ceiling Functions

After tiling, loop bounds involve floor (`⌊x/T⌋`) and ceiling (`⌈x/T⌉`) functions:

```
Tiled iteration with tile size T:
  Outer loop: ii = i / T      →  ii ∈ [0, ⌈N/T⌉)
  Inner loop: i = ii*T + i'   →  i' ∈ [0, min(T, N - ii*T))

Generated:
  for (ii = 0; ii < (N + T - 1) / T; ii++)         // ceil(N/T) using integer division
    for (i = ii*T; i < min(ii*T + T, N); i++)
```

ISL handles floor division in C via the idiom `(x >= 0 ? x/T : (x - T + 1)/T)` to match the floor semantics (round toward −∞) rather than C's truncation (round toward 0).

### 73.2.4 Parametric Bounds

When domain bounds involve symbolic parameters, FME produces parametric bounds that are evaluated at runtime:

```
Domain: { [i, j] : 0 ≤ i < M, i ≤ j < N }
→ j bounds: max(i, 0) ≤ j < N
→ j loop: for (j = i; j < N; j++)   (since i ≥ 0 always)

Loop structure:
  for (i = 0; i < M; i++)
    for (j = i; j < N; j++)   // runtime bound N
      S[i][j];
```

## 73.3 The ISL AST Generator

### 73.3.1 Architecture

ISL's AST generator (`isl_ast_build`) converts a schedule tree into an **AST** (`isl_ast_node`) — a high-level intermediate representation of the generated loop nest. This AST is then lowered to C, LLVM IR, or MLIR by the consumer.

```c
// Generate AST from schedule
isl_ast_build *build = isl_ast_build_alloc(ctx);
isl_ast_node *ast = isl_ast_build_node_from_schedule(build, schedule);

// Pretty-print as C
isl_printer *printer = isl_printer_to_file(ctx, stdout);
printer = isl_printer_set_output_format(printer, ISL_FORMAT_C);
printer = isl_printer_print_ast_node(printer, ast);
isl_printer_free(printer);
```

### 73.3.2 AST Node Types

| Node Type | Meaning |
|-----------|---------|
| `isl_ast_node_for` | A `for` loop with init, cond, inc, body |
| `isl_ast_node_if` | An `if` conditional with optional else |
| `isl_ast_node_block` | A sequence of statements |
| `isl_ast_node_mark` | A marker (for OpenMP pragma insertion) |
| `isl_ast_node_user` | A statement execution (leaf) |

`isl_ast_expr` nodes represent expressions:
- `isl_ast_expr_int`: integer constant
- `isl_ast_expr_id`: loop variable reference
- `isl_ast_expr_op`: arithmetic/comparison operation (add, max, floordiv, …)

### 73.3.3 Schedule Trees to AST

The ISL AST generator processes a schedule tree band-by-band:

1. **Band node**: generate a for-loop for each dimension of the band. The loop variable runs over the projection of the iteration domain onto this dimension.

2. **Sequence node**: generate a block of code with one section per child, adding guards (if-conditions) to distinguish which child's domain is active at each iteration.

3. **Set node**: generate code for unordered statement sets (used when statements at this level are mutually exclusive or can be scheduled in any order).

4. **Mark node**: pass through with an annotation attached to the surrounding loop (used for `parallel` or `vector` pragmas).

### 73.3.4 Generating Statement Bodies

At the leaves of the schedule tree are **user nodes** corresponding to statement executions. The code generator must:

1. Map the loop variables (in schedule space) back to the original iteration variables (in statement space).
2. Substitute the original iteration variables into the access functions to produce array indices.

This is done via the **inverse schedule**: if θ_S(I) = Θ_S·I + c_S, and the AST loop variables are t = θ_S(I), then I = Θ_S⁻¹ · (t − c_S). When Θ_S is not invertible (as with tiled schedules), the inverse is applied dimension-by-dimension using the schedule tree structure.

## 73.4 Tiling Strategies

### 73.4.1 Hyperrectangular Tiling

The simplest tiling: divide each loop dimension independently by a fixed tile size T_l:

```
Tile coordinates: tt_l = i_l / T_l       (outer loops)
Point coordinates: tp_l = i_l mod T_l    (inner loops)
```

Generated loop nest:

```c
for (ti = 0; ti < N/T; ti++)
  for (tj = 0; tj < N/T; tj++)
    for (i = ti*T; i < (ti+1)*T; i++)
      for (j = tj*T; j < (tj+1)*T; j++)
        S[i][j];
```

Hyperrectangular tiling is legal when all tiled dimensions are permutable (non-negative dependence distances in all tiled dimensions). It is the default in Polly and MLIR's affine tiling pass.

### 73.4.2 Parallelogram Tiling

For non-orthogonal dependence structures, **parallelogram tiling** uses a sheared tile shape:

```c
// Stencil with cross-iteration dependences:
// dep: S[t,i] → S[t+1,i], S[t,i] → S[t+1,i+1]
// Parallelogram tile: width W in i, depth D in t, skewed by the stencil footprint

// Outer loops (tile coordinates):
for (tp = ...; tp < ...; tp++)       // diagonal wavefront coordinate
  for (ti = ...; ti < ...; ti++)    // tile-within-wavefront coordinate
    // Inner loops (point within tile):
    for (t = tp*D; ...)
      for (i = ti*W + t; ...)        // i bound skewed by t (parallelogram shape)
```

Parallelogram tiles are regions of the iteration space bounded by skewed hyperplanes. They are harder to generate (the bounds involve both tile and time coordinates) but enable more data reuse for stencil computations.

### 73.4.3 Diamond Tiling

**Diamond tiling** (Bandishti et al., 2012) is a tiling strategy for stencil computations that enables **concurrent start**: after a warmup phase, all tiles on the same wavefront can execute in parallel and each tile can start as soon as its dependencies are met, without global synchronization.

```
Diamond tile shape: rhombus in (t, i) space.
  Tile (k, m) covers: { (t, i) | k*Dt ≤ t + i ≤ (k+1)*Dt, m*Di ≤ t ≤ (m+1)*Di }
  where Dt and Di are the temporal and spatial tile sizes.

Generated:
  for (k = ...) {          // wavefront (sequential)
    #pragma omp parallel for
    for (m = ...) {         // concurrent tiles on wavefront
      for (t = ...; i = ...) // within-tile computation
        S[t][i];
    }
  }
```

Diamond tiling is implemented in Polly via the `polly-tiling` pass when the stencil dependences are detected.

### 73.4.4 Time-Skewed Tiling

**Time-skewed tiling** (also called **cache-oblivious tiling**) skews the time dimension to allow tiles to start before the prior time-step tile finishes:

```
Skewed tile coordinate:
  t' = t,  i' = i + c*t   (skewed)
Tile in (t', i') space: rectangular (hyperrectangular in skewed coordinates)
```

The skewing makes the dependence vectors uniform in the skewed coordinates, enabling tiling. The Polly and MLIR polyhedral frameworks both support skewing-then-tiling as a composition of schedule transformations.

## 73.5 OpenMP Code Generation

### 73.5.1 Parallel Loops from Schedules

When a schedule dimension is marked **parallel** (all dependences in that dimension have zero projection — equivalently, coincidence constraints are satisfied), the ISL AST generator annotates the corresponding `for` loop with a `mark` node for the `omp_parallel` pragma:

```c
// ISL marks the outermost parallel loop:
#pragma omp parallel for
for (ii = 0; ii < N/T; ii++) {
  for (jj = 0; jj < N/T; jj++) {
    for (i = ii*T; i < (ii+1)*T; i++)
      for (j = jj*T; j < (jj+1)*T; j++)
        S[i][j];
  }
}
```

The parallelism annotation is carried in the schedule band's **permutability** and **coincidence** flags. Polly uses `isl_schedule_node_band_member_get_coincident` to determine which dimensions to mark parallel.

### 73.5.2 Reduction Parallelism

When a loop carries a **reduction dependence** (multiple iterations all update the same memory location), the loop is not strictly parallel but can be parallelized with OpenMP reduction clauses:

```c
double sum = 0.0;
#pragma omp parallel for reduction(+: sum)
for (int i = 0; i < N; i++)
  sum += A[i];   // reduction dependence: all iterations write sum
```

Polly detects reduction dependences through a special analysis and generates OpenMP reduction clauses for the accumulated variables.

### 73.5.3 Barrier Elimination

After tiling, inter-tile dependences determine which tile boundaries require synchronization. When two adjacent tiles on the same wavefront have no dependences between them, the barrier between them can be eliminated:

```c
// With barrier:
#pragma omp parallel for
for (tile = 0; tile < nTiles; tile++)
  compute_tile(tile);
#pragma omp barrier      // may be eliminable if tiles are independent
#pragma omp parallel for
for (tile = 0; tile < nTiles; tile++)
  compute_tile_next_step(tile);
```

ISL's schedule tree represents the barrier structure explicitly via **sequence** vs **set** nodes: set nodes indicate unordered execution (implicit barrier-free), sequence nodes indicate ordered execution (barrier required between children).

## 73.6 SIMD-Aware Code Generation

### 73.6.1 Vectorization at the Schedule Level

SIMD vectorization is the hardware version of inner-loop parallelism: multiple scalar iterations execute simultaneously in SIMD registers. At the polyhedral level, a loop dimension suitable for vectorization must:

1. Have zero dependence distance (fully parallel).
2. Have a uniform access pattern in the vectorized dimension (unit-stride accesses for efficient gather/scatter avoidance).

Polly marks the innermost parallel loop as **vectorizable** and generates LLVM IR vector intrinsics for the body:

```c
// Polly-vectorized loop (width 4):
// Original: for (j = 0; j < N; j++) B[j] = A[j] * 2.0;
// Vectorized:
for (j = 0; j < N-3; j += 4) {
  %v = load <4 x double>, ptr (A + j)   // vector load
  %r = fmul <4 x double> %v, <2.0, 2.0, 2.0, 2.0>
  store <4 x double> %r, ptr (B + j)    // vector store
}
// Scalar remainder:
for (; j < N; j++) B[j] = A[j] * 2.0;
```

### 73.6.2 MLIR Vector Dialect Integration

In MLIR (Part XX), polyhedral code generation targets the **vector dialect** for SIMD rather than LLVM intrinsics directly:

```mlir
// Polyhedral → MLIR affine → vector dialect
affine.for %i = 0 to %N step 4 {
  %va = vector.load %A[%i] : memref<?xf64>, vector<4xf64>
  %r = arith.mulf %va, %cst : vector<4xf64>
  vector.store %r, %B[%i] : memref<?xf64>, vector<4xf64>
}
```

The vector dialect provides hardware-portable SIMD operations that are later lowered to LLVM IR vectors or target-specific intrinsics by the MLIR lowering pipeline.

## 73.7 GPU Code Generation: Polly-ACC and PPCG

### 73.7.1 The GPU Mapping Problem

GPU execution requires mapping the iteration space to a hierarchy of CUDA/OpenCL thread blocks and threads:

```
Thread blocks (gridDim): coarse-grain parallelism — maps to outer parallel loops.
Threads (blockDim): fine-grain parallelism — maps to inner parallel loops.
```

A polyhedral schedule with two levels of parallelism (two coincident outer dimensions) naturally maps to this hierarchy.

### 73.7.2 PPCG (Polyhedral Parallel Code Generator)

**PPCG** (Verdoolaege et al., 2013) is a standalone polyhedral tool for GPU code generation:

1. Extract the SCoP using ISL.
2. Compute a schedule maximizing outer parallelism (two levels for grid/block).
3. Map the outermost two parallel dimensions to CUDA grid dimensions.
4. Map the next two parallel dimensions to CUDA block dimensions.
5. Generate CUDA C code with ISL's AST generator, inserting `__global__` kernels and data transfer calls.

```c
// Original:
for (int i = 0; i < N; i++)
  for (int j = 0; j < N; j++)
    C[i][j] = 0.0;
    for (int k = 0; k < N; k++)
      C[i][j] += A[i][k] * B[k][j];

// PPCG-generated (simplified):
__global__ void matmul_kernel(float *A, float *B, float *C, int N) {
  int i = blockIdx.x * blockDim.x + threadIdx.x;
  int j = blockIdx.y * blockDim.y + threadIdx.y;
  if (i < N && j < N) {
    float sum = 0.0;
    for (int k = 0; k < N; k++)
      sum += A[i*N + k] * B[k*N + j];
    C[i*N + j] = sum;
  }
}

// Host code:
dim3 grid((N + 31) / 32, (N + 31) / 32);
dim3 block(32, 32);
matmul_kernel<<<grid, block>>>(A_gpu, B_gpu, C_gpu, N);
```

### 73.7.3 Shared Memory Tiling for GPU

For matrix multiplication, the naive GPU mapping has high global memory traffic (A and B rows/columns are loaded multiple times). PPCG and Polly-ACC insert **shared memory tiling**: load a tile of A and a tile of B into shared memory (on-chip), perform the computation, and advance to the next tile:

```
Polyhedral representation:
  Original: C[i][j] += A[i][k] * B[k][j]  for all (i,j,k)
  Tiled: tile in (i,j) for thread block, tile in k for shared-memory reuse

Schedule (after tiling):
  θ(i,j,k) = (i/Tb, j/Tb, k/Tk, i%Tb, j%Tb, k%Tk)
  where Tb = thread block tile, Tk = k-tile (shared memory chunk)
```

The k-dimension tiling maps directly to the shared memory fetch loop:

```c
__global__ void matmul_tiled(float *A, float *B, float *C, int N) {
  __shared__ float As[32][32], Bs[32][32];  // shared memory tiles
  int bx = blockIdx.x, by = blockIdx.y;
  int tx = threadIdx.x, ty = threadIdx.y;
  float sum = 0.0;
  for (int k_tile = 0; k_tile < N; k_tile += 32) {
    // Load tile from global to shared:
    As[ty][tx] = A[(by*32 + ty)*N + k_tile + tx];
    Bs[ty][tx] = B[(k_tile + ty)*N + bx*32 + tx];
    __syncthreads();
    // Compute within tile:
    for (int kk = 0; kk < 32; kk++)
      sum += As[ty][kk] * Bs[kk][tx];
    __syncthreads();
  }
  C[(by*32 + ty)*N + bx*32 + tx] = sum;
}
```

This shared memory pattern emerges naturally from the polyhedral tiling when the k-tile maps to the shared memory level.

## 73.8 Unroll-and-Jam at the Polyhedral Level

### 73.8.1 Definition

**Unroll-and-jam** (also called **unroll-and-fuse**) unrolls an outer loop by factor U and fuses the resulting U copies of the inner loop:

```c
// Original:                         // After unroll-and-jam (U=2):
for (i = 0; i < N; i += 2) {        for (i = 0; i < N-1; i += 2) {
  for (j = 0; j < M; j++) {           for (j = 0; j < M; j++) {
    A[i][j] += B[i][j];                 A[i][j] += B[i][j];     // original i
  }                                     A[i+1][j] += B[i+1][j]; // unrolled i+1
}                                     }
                                     }
```

Unroll-and-jam increases ILP (instruction-level parallelism) by exposing independent operations from different unrolled iterations in the same inner loop body — enabling the processor to execute them in parallel.

### 73.8.2 Polyhedral Formulation

In the polyhedral model, unroll-and-jam is a two-step transformation:

1. **Strip-mine** the outer loop by factor U: replace loop i with ii = i/U (outer) and i' = i%U (inner).
2. **Interchange** the strip-mined inner dimension (i') past the innermost original loop (j).

Resulting schedule (with U=2):

```
θ_S(i, j) = (i/2, j, i mod 2)  →  unroll-and-jam with U=2
```

The code generator then unrolls the innermost i' loop (which has fixed trip count U) statically, producing U copies of the j-loop body.

### 73.8.3 Legality

Unroll-and-jam is legal if and only if no dependence is violated by bringing i%U to the innermost position — specifically, no i-direction dependence exists with a component that would be reversed by the interchange. For element-wise (embarrassingly parallel) operations, it is always legal.

---

## Chapter Summary

- **Loop bound generation** uses Fourier-Motzkin elimination to project the schedule image onto each loop variable dimension; the result is a sequence of max/min expressions (potentially involving floor/ceiling for tiled loops) that form the loop bounds in the generated code.
- **ISL's AST generator** (`isl_ast_build`) converts a polyhedral schedule tree into a structured `isl_ast_node` tree (for/if/block/mark/user nodes), which is then lowered to C, LLVM IR, or MLIR by the consumer.
- **Multiple statements** with different domains require conditional guards (if-conditions) in the generated loops; ISL generates these guards automatically by checking which domain each loop-variable assignment satisfies.
- **Hyperrectangular tiling** divides each permutable dimension by a constant tile size, generating outer (tile-coordinate) and inner (point-coordinate) loops; **parallelogram** and **diamond tiling** use skewed tile shapes for stencil computations with non-orthogonal dependences.
- **OpenMP code generation** maps parallel schedule dimensions to `#pragma omp parallel for` pragmas; ISL schedule tree **set nodes** represent unordered (barrier-free) children, **sequence nodes** represent ordered (barrier-required) children.
- **SIMD vectorization** marks the innermost fully-parallel dimension for vector code generation; in MLIR, this targets the vector dialect for hardware-portable SIMD before lowering to LLVM IR vectors.
- **GPU code generation** (PPCG, Polly-ACC) maps the two outermost parallel dimensions to CUDA grid and block dimensions, with shared memory tiling derived from the k-dimension tile in the polyhedral schedule.
- **Unroll-and-jam** is a composition of strip-mining and interchange in the polyhedral model; it exposes ILP within the inner loop body by statically unrolling the strip-mined outer loop.


---

@copyright jreuben11
