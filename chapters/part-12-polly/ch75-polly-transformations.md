# Chapter 75 — Polly Transformations

*Part XII — Polly*

Polly's value lies in the transformations it applies after the SCoP is detected and represented. Loop tiling for cache reuse, loop fusion for data locality, loop parallelization for multi-core, and vectorization for SIMD — these transformations are all expressed as modifications to the polyhedral schedule. This chapter covers how Polly maps the theoretical schedule transformations of Part XI to concrete LLVM IR changes, how tile sizes are selected, loop fusion in practice, the JSON schedule format for manual schedule inspection and injection, and the pattern-based matrix multiplication optimization.

## 75.1 Tiling in Polly

### 75.1.1 Default Tiling

Polly's default tiling (`-polly-tiling`, enabled by default at `-O3`) tiles all **permutable band dimensions** — those schedule dimensions where all dependences have non-negative direction:

```bash
clang -O3 -mllvm -polly -mllvm -polly-tiling \
      -mllvm -polly-tile-sizes=32,32,32 input.c
```

The ISL schedule band is split into a tile and point band:

```
Before tiling:
  Band { i, j, k } (all permutable)

After tiling (T=32):
  Band { ii, jj, kk }  (tile-coordinate outer loops)
  Band { i', j', k' }  (point-coordinate inner loops)
  where ii = i/32, i' = i mod 32  (similarly for j, k)
```

Generated LLVM IR reflects this two-level loop nest:

```c
// Generated from Polly for matmul (simplified):
for (ii = 0; ii < N/32; ii++)
  for (jj = 0; jj < N/32; jj++)
    for (kk = 0; kk < N/32; kk++)
      for (i = ii*32; i < min(ii*32+32, N); i++)
        for (j = jj*32; j < min(jj*32+32, N); j++)
          for (k = kk*32; k < min(kk*32+32, N); k++)
            C[i][j] += A[i][k] * B[k][j];
```

### 75.1.2 Tile Size Selection

The default tile size (32 in each dimension) is a conservative heuristic. Optimal tile sizes depend on:
- **L1 cache size**: the tile should fit in L1 cache. For three arrays of doubles, the constraint is `3 * T² * 8 ≤ L1_size`.
- **SIMD width**: the innermost dimension should be a multiple of the SIMD width (4 for AVX2 doubles, 8 for AVX-512 floats).
- **TLB entries**: very large tiles may exceed TLB capacity.

Polly accepts tile sizes as a comma-separated list:

```bash
# Tile sizes for 2D loop: 32 in i, 64 in j
clang -mllvm -polly -mllvm -polly-tile-sizes=32,64 input.c

# Per-loop-nest tile sizes (colon-separated groups):
clang -mllvm -polly -mllvm -polly-tile-sizes=32,32:16,16 input.c
```

### 75.1.3 Auto-Tiling via ISL

For automatic tile size selection, Polly integrates with ISL's parametric scheduling mode: tile sizes become parameters of the schedule, and the schedule optimizer minimizes dependence distances parametrically. This produces a schedule function of the tile sizes (T₁, T₂, …) that can be evaluated for different tile size values.

```bash
# Use ISL's auto-tiling heuristics:
clang -mllvm -polly -mllvm -polly-opt-isl -mllvm -polly-tiling input.c
```

The ISL scheduler's tile size heuristic balances the number of conflict misses (proportional to `1/T`) against the parallelism granularity (proportional to `T`).

## 75.2 Loop Fusion and Distribution in Polly

### 75.2.1 Fusion via Schedule

Loop fusion is implemented by assigning the same schedule row to corresponding iterations of two consecutive loops:

```c
// Before: two separate loops
for (int i = 0; i < N; i++) A[i] = B[i] + 1;
for (int i = 0; i < N; i++) C[i] = A[i] * 2;

// After fusion (same schedule for both statements):
for (int i = 0; i < N; i++) {
  A[i] = B[i] + 1;
  C[i] = A[i] * 2;  // uses A[i] written above: flow dep S1→S2 at same i
}
```

Polly detects fusion opportunities via the `coincidence` dependence constraints: when the RAW dependence from S1 to S2 has zero distance (same iteration of both statements), fusing them eliminates the reuse distance for array A.

### 75.2.2 Fusion Legality

Fusion is legal if it does not violate any dependences. The critical check: no dependence flows from S2 to S1 at a later iteration (which would require S1 to wait for a future S2, creating a cycle):

```c
// ILLEGAL fusion: anti-dependence S2→S1
for (int i = 0; i < N; i++) A[i+1] = B[i] + 1;  // S1 writes A[i+1]
for (int i = 0; i < N; i++) C[i] = A[i] * 2;     // S2 reads A[i]
// S2[i] reads A[i], S1[i] writes A[i+1]: anti-dep with distance -1
// After fusion: S1[i] writes A[i+1], S2[i] reads A[i] = A[i+1] read by next iter
// → LEGAL at same i, but S2[i+1] reads A[i+1] written by S1[i]: OK
// Actually this IS legal; the detailed dependence analysis determines this
```

Polly uses ISL's exact dependence analysis to verify fusion legality, avoiding the conservative over-approximations of simpler dependence tests.

### 75.2.3 The MaxFuse Scheduler

Polly's `MaxFuse` scheduler mode (`-polly-fuse=max`) aggressively fuses loops that share memory accesses, prioritizing data reuse over parallelism:

```bash
clang -mllvm -polly -mllvm -polly-scheduler=isl \
      -mllvm -polly-fuse=max input.c
```

MaxFuse modifies the ISL scheduler's coincidence constraints to prefer fusing statement iterations: it marks as "coincident" all pairs of statement iterations that can legally execute at the same time step. The ISL scheduler then minimizes the number of distinct time steps, effectively maximizing fusion.

## 75.3 Parallelization

### 75.3.1 OpenMP Generation

When a schedule band dimension is marked **parallel** (all dependences have zero component in that dimension), Polly wraps the corresponding loop in OpenMP pragmas:

```bash
clang -O3 -mllvm -polly -mllvm -polly-parallel input.c \
      -fopenmp
```

Polly's `IslNodeBuilder` detects `mark` nodes in the ISL AST (emitted when a band dimension is coincident) and wraps the loop in an OpenMP region:

```cpp
// In IslNodeBuilder::createMark:
if (MarkName == "SIMD")
  // Mark inner loop for vectorization
  setVectorizeHint(LoopInst, true);
else if (MarkName == "OMP_PARALLEL")
  // Emit OpenMP parallel for:
  emitOmpParallelFor(LoopInst, ReducVars);
```

The generated LLVM IR uses the OpenMP runtime calls (`__kmpc_fork_call`, `__kmpc_for_static_init`, etc.) that are then lowered to native thread parallelism by the OpenMP runtime.

### 75.3.2 Reduction Parallelization

For loops with reduction dependences (S[i] reads and writes the same variable), Polly detects the reduction pattern and emits OpenMP reduction:

```c
// Original (reduction):
double sum = 0.0;
for (int i = 0; i < N; i++)
  sum += A[i];

// Polly output (with -polly-parallel):
#pragma omp parallel for reduction(+:sum)
for (int i = 0; i < N; i++)
  sum += A[i];
```

Polly's reduction detection scans each `MemoryAccess` for the pattern: a variable that is both read and written in the same statement, with a commutative and associative operator (add, mul, min, max, bitwise and/or/xor).

## 75.4 Vectorization in Polly

### 75.4.1 Inner-Loop Vectorization

Polly marks the innermost fully-parallel schedule dimension for **LLVM vectorization**. Rather than emitting explicit SIMD intrinsics, Polly sets LLVM loop metadata:

```llvm
; Generated by Polly with vectorization hints:
br label %for.body, !llvm.loop !0
!0 = !{!0, !1}
!1 = !{!"llvm.loop.vectorize.enable", i1 true}
```

This passes control to LLVM's Loop Vectorizer (Chapter 64), which handles the actual SIMD code generation. Polly ensures the loop is in the correct form (no loop-carried dependences, known trip count expression) for the loop vectorizer to succeed.

### 75.4.2 Polly Vectorizer (Polly-LLVM)

For SCoPs with constant trip counts, Polly can directly emit vectorized LLVM IR (vector loads, vector arithmetic, vector stores) without relying on the Loop Vectorizer. This is controlled by `-polly-vectorizer=polly`:

```bash
clang -O3 -mllvm -polly -mllvm -polly-vectorizer=polly input.c
```

This mode emits `load <N x double>`, `fadd <N x double>`, `store <N x double>` instructions directly, bypassing the Loop Vectorizer pass.

## 75.5 Pattern-Based Matrix Multiplication

### 75.5.1 Detection

Polly includes a special-case optimization for matrix multiplication kernels (`-polly-opt-isl`). When Polly detects a three-nested loop with:
- Three array accesses with access functions `A[i][k]`, `B[k][j]`, `C[i][j]`
- A `+=` accumulation pattern
- No other side-effecting accesses

it applies a fixed high-performance schedule that overrides the generic ISL scheduler:

```cpp
// In ScheduleOptimizer.cpp:
bool ScheduleOptimizer::isProfitableSchedule(Scop &S, isl_schedule *NewSched) {
  if (PMBasedOpts && containsMatMul(S)) {
    *NewSched = getMatMulSchedule(S);  // apply matmul-specific schedule
    return true;
  }
  return isIslScheduleGoodEnough(S, NewSched);
}
```

### 75.5.2 The Matmul Schedule

The matmul-specific schedule applies:
1. Loop interchange to put the k-loop (reduction) innermost.
2. Tiling in (i, j) for cache reuse of C.
3. Tiling in k for register reuse of the A and B tiles.
4. Vectorization of the innermost j dimension.
5. Unroll-and-jam of the i and j register tiles.

This produces a schedule similar to what BLAS implementations use:

```c
// Polly-optimized matmul (structure, simplified):
for (ii = 0; ii < M; ii += MC)                    // cache tile i
  for (kk = 0; kk < K; kk += KC)                  // cache tile k
    pack_A_panel(A + ii*K + kk, Ap, MC, KC);       // pack A tile
    for (jj = 0; jj < N; jj += NC)                // cache tile j
      pack_B_panel(B + kk*N + jj, Bp, KC, NC);    // pack B tile
      for (i = ii; i < ii+MC; i += MR)            // register tile i
        for (j = jj; j < jj+NC; j += NR)          // register tile j
          ukernel(Ap, Bp, C+i*N+j, MR, NR, KC);  // micro-kernel
```

The micro-kernel is optionally replaced by a call to `cblas_dgemm` or an inline LLVM IR assembly sequence for the target architecture's optimal register blocking.

## 75.6 The JSON Schedule Format

### 75.6.1 Schedule Inspection and Export

Polly can export its computed schedule as a JSON file for inspection and manual modification:

```bash
# Export the schedule to JSON:
clang -O3 -mllvm -polly -mllvm -polly-export-jscop input.c
# Produces: input_function_name__region__entry__exit.jscop

# Import a manually modified schedule:
clang -O3 -mllvm -polly -mllvm -polly-import-jscop=./modified.jscop input.c
```

The `.jscop` file is a JSON representation of the SCoP structure and schedule:

```json
{
  "context": "[N, M] -> { : N >= 1 and M >= 1 }",
  "statements": [
    {
      "name": "Stmt_loop_body",
      "domain": "[N, M] -> { Stmt_loop_body[i, j] : 0 <= i < N and 0 <= j < M }",
      "schedule": "[N, M] -> { Stmt_loop_body[i, j] -> [i, j] }",
      "accesses": [
        {
          "kind": "read",
          "relation": "[N, M] -> { Stmt_loop_body[i, j] -> MemRef_A[i, j] }"
        },
        {
          "kind": "write",
          "relation": "[N, M] -> { Stmt_loop_body[i, j] -> MemRef_B[i, j] }"
        }
      ]
    }
  ]
}
```

### 75.6.2 Manual Schedule Injection

The JSON format enables research workflows where a polyhedral schedule computed by an external tool (e.g., a machine-learning model, a custom variant of Pluto, or manually crafted) is injected into Polly's code generation pipeline:

```bash
# 1. Generate the initial .jscop:
clang -O3 -mllvm -polly -mllvm -polly-export-jscop matmul.c

# 2. Modify the schedule in the .jscop file:
# e.g., change: "schedule": "{ S[i,j,k] -> [i, j, k] }"
# to:           "schedule": "{ S[i,j,k] -> [i/32, j/32, k, i%32, j%32] }"

# 3. Compile with the modified schedule:
clang -O3 -mllvm -polly -mllvm -polly-import-jscop=matmul.jscop matmul.c
```

This enables A/B testing of different schedules without modifying Polly's C++ source.

### 75.6.3 Access Function Modification

The `.jscop` format also allows modifying access functions — useful for inserting **array packing** or **data layout transformations**:

```json
// Modified access: pack A into a row-major buffer (changing access pattern)
{
  "kind": "write",
  "relation": "[N, M] -> { Stmt_pack[i, j] -> MemRef_A_packed[i * M + j] }"
}
```

By changing the access relation in the `.jscop`, Polly generates code that reads from the repacked array instead of the original, without the compiler needing to understand the packing intent.

## 75.7 Alias Analysis Integration

Polly requires alias-free access analysis: if the compiler cannot prove that two array accesses do not alias, the dependence analysis is forced to assume a dependence exists. Polly uses several strategies to discharge aliases:

### 75.7.1 LLVM AA Integration

Polly queries LLVM's alias analysis chain (`AAResults`, Chapter 61) to check each pair of memory accesses:

```cpp
// Check if two accesses may alias:
AliasResult AR = AA.alias(MA1.getOriginalBaseAddr(),
                           MA2.getOriginalBaseAddr());
if (AR == AliasResult::NoAlias) {
  // No alias — can compute exact dependences
} else {
  // May alias — add conservative assumption to Scop::Assumptions
}
```

When aliasing cannot be disproved, Polly adds a **runtime alias check** (an assumption that is checked at runtime before entering the polyhedral region). If the check fails, Polly falls back to the original (non-optimized) loop code.

### 75.7.2 The `__restrict__` Annotation

C99's `__restrict__` qualifier tells the compiler that a pointer is the unique accessor to its target memory:

```c
void compute(double * __restrict__ A, double * __restrict__ B, int N) {
  for (int i = 0; i < N; i++) B[i] = A[i] * 2.0;
}
```

With `__restrict__`, LLVM's alias analysis marks A and B as non-aliasing, and Polly can compute exact dependences without runtime checks.

---

## Chapter Summary

- **Tiling** in Polly splits permutable band dimensions into tile-coordinate outer loops and point-coordinate inner loops; tile sizes are user-specified (32 by default) or computed via ISL's auto-tiling heuristics.
- **Loop fusion** is implemented by assigning the same schedule row to corresponding iterations of consecutive loop statements; Polly's `MaxFuse` scheduler mode prioritizes fusion via coincidence constraints.
- **Parallelization** wraps coincident (zero-dependence-distance) schedule dimensions in OpenMP `parallel for` pragmas; reduction patterns are detected and emitted with OpenMP reduction clauses.
- **Vectorization** marks the innermost parallel dimension via LLVM loop metadata (`llvm.loop.vectorize.enable`) to delegate to the Loop Vectorizer, or directly emits vector LLVM IR in `-polly-vectorizer=polly` mode.
- **Pattern-based matmul optimization** detects the three-nested matrix multiplication kernel and applies a fixed high-performance schedule (cache tiling + register tiling + vectorization + unroll-and-jam) that bypasses the generic ISL scheduler.
- **The JSON schedule format** (`.jscop`) exports the full polyhedral representation (domain, schedule, access functions) for external inspection and manual modification; `-polly-import-jscop` allows injecting custom schedules.
- **Alias analysis** integration uses LLVM's `AAResults` to prove array non-aliasing; when aliasing cannot be disproved, Polly inserts runtime alias checks that fall back to the original code if aliasing is detected.


---

@copyright jreuben11
