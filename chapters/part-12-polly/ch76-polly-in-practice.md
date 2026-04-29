# Chapter 76 — Polly in Practice

*Part XII — Polly*

Understanding when Polly helps, when it does not, and how to diagnose failures is essential for using it effectively. Polly's power comes from its strict affine restriction: the same property that enables exact analysis and sound transformation also limits which programs it can transform. This chapter covers what makes a loop SCoP-eligible, the most common reasons SCoP detection fails, how to read Polly's diagnostic output, the experimental GPU codegen path, maintenance status, and a comparison with MLIR's Affine dialect — the modern successor to Polly's design philosophy.

## 76.1 When Polly Helps

Polly provides the most benefit for programs matching the following profile:

### 76.1.1 High Arithmetic Intensity Loops

Loops where the ratio of arithmetic operations to memory accesses is high — matrix multiplication, stencil computations, convolutions — benefit most from Polly's cache tiling. The working set fits in L1/L2 cache after tiling, dramatically reducing memory bandwidth pressure.

```c
// HIGH benefit: O(N³) arithmetic, O(N²) data → high arithmetic intensity
for (i = 0; i < N; i++)
  for (j = 0; j < N; j++)
    for (k = 0; k < N; k++)
      C[i][j] += A[i][k] * B[k][j];
```

A 32×32 tile of doubles uses `32 × 32 × 8 = 8 KB` — fits comfortably in L1 cache (typically 32-64 KB). Without tiling, A's rows are loaded N times (once per j iteration), wasting memory bandwidth.

### 76.1.2 Stencil Computations

Regular stencils (Jacobi, Gauss-Seidel, heat equation, wave equation) have structured dependences that Polly can detect and schedule optimally. Polly's diamond tiling and wavefront parallelism are specifically designed for this pattern.

```c
// HIGH benefit: Jacobi 2D stencil — structured dependences
for (int t = 0; t < T; t++)
  for (int i = 1; i < N-1; i++)
    for (int j = 1; j < N-1; j++)
      A_new[i][j] = 0.25 * (A[i-1][j] + A[i+1][j] + A[i][j-1] + A[i][j+1]);
```

Polly detects the dependence pattern (S[t,i,j] → S[t+1,i,j] and neighbors), applies a time-skewed tile, and parallelizes the wavefront dimension.

### 76.1.3 Loop Nests with Parallelism Hidden by Non-Affine Intervening Code

When a loop nest has parallel inner structure that the Loop Vectorizer cannot see (because the outer loop is not stripped before vectorization), Polly's schedule optimization can expose it:

```c
// The j-loop is always parallel, but the scalar optimizer may not see it:
for (int i = 0; i < N; i++) {
  preprocess(A[i]);               // function call (may prevent vectorization)
  for (int j = 0; j < M; j++)
    B[i][j] = A[i] * C[j];       // j-loop is trivially parallel
}
```

If `preprocess` is annotated as pure (or if Polly models it as a non-affine subregion), Polly can parallelize the j-loop even though the scalar optimizer cannot.

## 76.2 When Polly Does Not Help

### 76.2.1 Non-Affine Loop Bounds or Accesses

The most common reason Polly fails to extract a SCoP:

```c
// FAILS: data-dependent loop bound
for (int i = 0; i < A[0]; i++) {   // A[0] is runtime data, not a parameter
  B[i] = C[i] * 2.0;
}

// FAILS: indirect array access
for (int i = 0; i < N; i++) {
  int idx = index_array[i];         // indirect: not an affine function of i
  B[idx] = A[i] * 2.0;
}
```

Indirect accesses (`A[B[i]]`) cannot be expressed as affine maps; any loop containing them is not a SCoP.

### 76.2.2 Function Calls with Unknown Effects

Any function call inside a potential SCoP that is not proven pure (no memory side effects) causes Polly to conservatively model it as potentially reading and writing all memory:

```c
// FAILS to be SCoP: unknown function call
for (int i = 0; i < N; i++) {
  printf("value: %f\n", A[i]);   // printf has IO side effects
  B[i] = A[i] * 2.0;
}
```

Workaround: separate the non-pure calls from the computational loops, or annotate mathematical functions with `__attribute__((pure))`.

### 76.2.3 Short or Simple Loops

Polly's overhead (SCoP detection, ISL scheduling, code generation) is non-trivial. For short loops (trip count < 64) or loops that are already trivially vectorized by LLVM's Loop Vectorizer, Polly provides no benefit and may even slow compilation significantly.

```bash
# Limit Polly to loops with at least 1000 iterations:
clang -mllvm -polly -mllvm -polly-scops-trip-count-threshold=1000 input.c
```

### 76.2.4 Aliasing Memory Accesses

When the compiler cannot prove that arrays do not alias, Polly inserts runtime alias checks. These checks add overhead and may prevent Polly from applying aggressive transformations:

```c
// Without __restrict__: Polly must assume A and B may alias
void add(double *A, double *B, int N) {
  for (int i = 0; i < N; i++) A[i] += B[i];
}
```

At runtime, if the alias check fails, Polly falls back to the original loop — the check overhead is paid for nothing. Add `__restrict__` to eliminate alias assumptions.

## 76.3 Diagnostic Output

### 76.3.1 SCoP Report

The `-polly-report` flag causes Polly to print a summary of detected SCoPs:

```bash
clang -O3 -mllvm -polly -mllvm -polly-report input.c
```

Output:
```
Polly optimized the following regions:
  matmul (lines 12-15): 3 loop(s), 1 statement(s)
  jacobi (lines 45-52): 3 loop(s), 1 statement(s)
```

### 76.3.2 Detection Failures

The `-polly-report-scop-restrictions` flag explains why a region was not a SCoP:

```bash
clang -O3 -mllvm -polly -mllvm -polly-report-scop-restrictions input.c
```

Output:
```
Failed to detect SCoP at function foo, loop at line 23:
  NonAffineAccess: access A[B[i]] is not affine (indirect indexing)
Failed to detect SCoP at function bar, region 35-48:
  NonAffineSubRegionCall: call to printf() at line 37 has unknown memory effects
```

### 76.3.3 Optimization Remarks

Polly emits optimization remarks via LLVM's remark infrastructure:

```bash
clang -O3 -mllvm -polly -Rpass=polly-detect \
      -Rpass-missed=polly-detect input.c
```

Output:
```
input.c:12:3: remark: SCoP detected in function 'matmul', loop depth 3 [-Rpass=polly-detect]
input.c:23:3: remark: loop at line 23 rejected: NonAffineAccess [-Rpass-missed=polly-detect]
```

These remarks integrate with `-fsave-optimization-record` to produce YAML files for offline analysis with tools like `opt-viewer.py`.

### 76.3.4 ISL AST Visualization

To inspect the generated ISL AST before LLVM IR emission:

```bash
opt -load LLVMPolly.so -polly-detect -polly-scops -polly-dependences \
    -polly-schedule -polly-print-isl-ast -S input.ll
```

This prints the ISL AST (C-like representation) that will be lowered to LLVM IR, allowing verification that the schedule is as expected.

## 76.4 Performance Measurement

### 76.4.1 With vs. Without Polly

A systematic comparison requires isolating Polly's contribution from the rest of LLVM's optimizations:

```bash
# Without Polly:
clang -O3 input.c -o prog_no_polly
time ./prog_no_polly

# With Polly, same optimization level:
clang -O3 -mllvm -polly input.c -o prog_polly
time ./prog_polly

# Polly with explicit options:
clang -O3 -mllvm -polly \
      -mllvm -polly-parallel \
      -mllvm -polly-vectorizer=stripmine \
      -mllvm -polly-tile-sizes=64,64,64 \
      input.c -o prog_polly_tuned -fopenmp
time ./prog_polly_tuned
```

### 76.4.2 PolyBench Benchmarks

**PolyBench** (Pouchet et al.) is the standard benchmark suite for polyhedral compilers, containing 30 kernels across 5 categories:

| Category | Examples |
|----------|---------|
| Linear algebra | `gemm`, `syr2k`, `trmm`, `atax`, `mvt`, `symm` |
| Data mining | `covariance`, `correlation` |
| Stencils | `jacobi-1d`, `jacobi-2d`, `seidel-2d`, `heat-3d` |
| Medley | `reg_detect`, `nussinov`, `floyd-warshall` |
| Linear solve | `lu`, `cholesky`, `trisolv` |

Polly typically achieves 2–10× speedup on the linear algebra and stencil kernels compared to `-O3` without Polly, due to effective cache tiling and parallelization. The data mining kernels with their triangular iteration spaces show more variable results.

```bash
# Run PolyBench with Polly:
git clone https://github.com/MatthiasKlauser/polybench-c
cd polybench-c
for kernel in linear-algebra/blas/gemm stencils/jacobi-2d; do
  clang -O3 -mllvm -polly \
        -I utilities utilities/polybench.c $kernel/$kernel.c \
        -DPOLYBENCH_TIME -o ${kernel##*/}_polly
  clang -O3 \
        -I utilities utilities/polybench.c $kernel/$kernel.c \
        -DPOLYBENCH_TIME -o ${kernel##*/}_base
  echo -n "$kernel base: "; ./${kernel##*/}_base
  echo -n "$kernel polly: "; ./${kernel##*/}_polly
done
```

## 76.5 Experimental GPU Codegen: Polly-ACC

### 76.5.1 Overview

Polly-ACC is Polly's experimental GPU offloading path, generating CUDA/OpenCL code from polyhedral schedules:

```bash
# Experimental: requires LLVM built with CUDA support
clang -O3 -mllvm -polly -mllvm -polly-acc input.c \
      --cuda-gpu-arch=sm_80 -o prog_gpu
```

Polly-ACC generates:
1. A polyhedral schedule with two outer parallel dimensions (for CUDA grid and block).
2. Data transfer code (`cudaMemcpy` for host-to-device and device-to-host).
3. A CUDA kernel via LLVM's PTX backend.

### 76.5.2 Limitations

Polly-ACC is marked experimental because:
- **Data transfer overhead**: Polly does not perform sophisticated data transfer minimization (it transfers entire arrays before and after each kernel), limiting its benefit for kernels with small arithmetic intensity.
- **Memory management**: dynamic allocation inside the offloaded region is not supported.
- **Multi-GPU**: no multi-device support.

For production GPU offloading, OpenMP `target offload` directives or CUDA/HIP explicit programming (Chapter 48–49) are more reliable than Polly-ACC's experimental path.

## 76.6 Polly Maintenance Status

As of LLVM 22.1, Polly's maintenance status reflects the shift in the compiler community toward MLIR for high-performance loop optimization:

- **Active but declining**: Polly remains in the LLVM monorepo and compiles correctly, but active development has slowed. The last major algorithmic addition was the matmul pattern optimizer.
- **Bug fixes only**: most commits to `polly/` are bug fixes, build system updates, and LLVM API compatibility changes rather than new features.
- **MLIR preferred**: for new work requiring polyhedral optimization, the MLIR Affine dialect (Chapter 140) is the preferred approach. It has a growing community, first-class support for machine learning workloads, and better integration with hardware-specific backends.

```bash
# Check Polly works in your LLVM build:
llvm-config --has-polly     # may not exist; use:
clang -mllvm -polly -mllvm -polly-report /dev/null -x c /dev/null 2>&1 | head -1
```

## 76.7 Polly vs. MLIR Affine Dialect

The MLIR **Affine dialect** (Part XX) is the modern successor to Polly's design philosophy. The comparison:

| Aspect | Polly | MLIR Affine Dialect |
|--------|-------|---------------------|
| Input | LLVM IR (post-lowering) | MLIR at source-level IR |
| SCoP detection | ScalarEvolution analysis | Explicit affine annotations |
| Schedule representation | ISL schedule tree | ISL schedule tree + MLIR transforms |
| Code generation | ISL AST → LLVM IR | Affine → SCF → LLVM dialects |
| Extensibility | C++ passes only | MLIR pass infrastructure |
| ML workload support | Limited | First-class (linalg, tensor) |
| GPU targets | Experimental CUDA | GPU dialect + SPIR-V + NVVM |

The key architectural difference: Polly recovers polyhedral structure from LLVM IR by reverse-engineering access patterns via ScalarEvolution. MLIR's Affine dialect represents polyhedral structure *explicitly* in the IR from the start — `affine.for` loops carry their bounds as affine maps, and `affine.load`/`affine.store` carry their access functions explicitly. This makes analysis exact without requiring backward recovery.

MLIR's approach allows domain-specific dialects (e.g., `linalg.matmul`) to carry semantic information that can drive specialized code generation (e.g., mapping directly to BLAS calls or GPU tensor cores) without going through a generic polyhedral analysis.

### 76.7.1 Migration Path

For new projects, prefer the MLIR Affine dialect workflow:

```python
# MLIR workflow (Python bindings):
import mlir.dialects.affine as affine
import mlir.dialects.linalg as linalg

# Express matrix multiplication at linalg level:
linalg.matmul(A, B, C)  # → polyhedral schedule → GPU kernel
```

For existing C/C++ codebases where adding MLIR is impractical, Polly remains the only option for automatic polyhedral optimization with LLVM.

---

## Chapter Summary

- Polly provides the most benefit for **high arithmetic intensity** loops (matrix multiplication, stencils) where cache tiling reduces memory bandwidth pressure by an order of magnitude; it provides little or no benefit for short loops, loops with indirect accesses, or loops with function calls with unknown memory effects.
- **SCoP detection failures** are diagnosed with `-polly-report`, `-polly-report-scop-restrictions`, and `-Rpass-missed=polly-detect`; common causes are non-affine accesses (indirect indexing), data-dependent conditions, and impure function calls.
- **PolyBench** is the standard benchmark for evaluating polyhedral compilers; Polly typically achieves 2–10× speedup on linear algebra and stencil kernels vs. `-O3`.
- **Polly-ACC** is an experimental CUDA offloading path that generates GPU kernels from polyhedral schedules; it is limited by naive data transfer and is not production-ready.
- **Maintenance status**: Polly is in maintenance mode as of LLVM 22.1; the MLIR Affine dialect is the preferred platform for new polyhedral optimization work.
- **MLIR Affine dialect** supersedes Polly's design: it represents polyhedral structure explicitly in the IR rather than recovering it via ScalarEvolution, enabling exact analysis, first-class ML workload support, and integration with GPU dialects.


---

@copyright jreuben11
