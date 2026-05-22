# Chapter 227 — LLM-Guided Polyhedral Optimization: LOOPer and Agentic Auto-Scheduling

*Part XXXI — Frontier AI Evolution*

Polyhedral compilation (Chapters 131–132) is the most principled framework for loop optimization: it represents loop nests as integer polyhedra, proves legality of transformations using dependence analysis, and generates new loop structures via affine scheduling. The problem is not correctness — the polyhedral model guarantees that only legal transformations are applied — but performance. Which tiling factor should be used? Which loops should be fused? What is the optimal interchange order? Classical cost models based on dependence distances and memory footprint estimates fail to predict actual performance on modern out-of-order processors with complex prefetchers, deep cache hierarchies, and non-uniform NUMA topology. Machine learning replaces these analytical cost models with data-driven ones that can capture the complex hardware behavior that analytical models miss. This chapter covers LOOPer's deep learning cost model, the agentic auto-scheduling loop with empirical feedback, integration with Polly and MLIR's affine dialect, and the limitations that remain unsolved.

---

## Table of Contents

- [227.1 The Polyhedral Parameter Selection Problem](#2271-the-polyhedral-parameter-selection-problem)
  - [What Polyhedral Compilation Gives You](#what-polyhedral-compilation-gives-you)
  - [The Parameter Space](#the-parameter-space)
  - [Why Analytical Cost Models Fail](#why-analytical-cost-models-fail)
- [227.2 LOOPer: Deep Learning Cost Models for Affine Transformations](#2272-looper-deep-learning-cost-models-for-affine-transformations)
  - [Architecture](#architecture)
  - [Feature Vector Construction](#feature-vector-construction)
  - [Training Data Collection](#training-data-collection)
  - [Using LOOPer for Schedule Selection](#using-looper-for-schedule-selection)
  - [Empirical Results](#empirical-results)
- [227.3 Agentic Auto-Scheduling with Empirical Feedback](#2273-agentic-auto-scheduling-with-empirical-feedback)
  - [The Agentic Loop](#the-agentic-loop)
  - [LLM Prompting Strategy](#llm-prompting-strategy)
  - [Convergence Behavior](#convergence-behavior)
- [227.4 Integration with Polly](#2274-integration-with-polly)
  - [Polly's Schedule Representation](#pollys-schedule-representation)
  - [Setting the Schedule](#setting-the-schedule)
  - [Fallback Chain](#fallback-chain)
- [227.5 Integration with MLIR Affine Dialect](#2275-integration-with-mlir-affine-dialect)
  - [Affine Loop Transformation Passes](#affine-loop-transformation-passes)
  - [Advantages Over Polly for Agentic Optimization](#advantages-over-polly-for-agentic-optimization)
- [227.6 Comparison with Ansor/AutoTVM and Halide Autoscheduler](#2276-comparison-with-ansorautotvm-and-halide-autoscheduler)
  - [Ansor (TVM, arXiv 2006.06762)](#ansor-tvm-arxiv-200606762)
  - [Halide Autoscheduler (Li et al. 2021)](#halide-autoscheduler-li-et-al-2021)
  - [Comparison Table](#comparison-table)
- [227.7 Limitations and Open Problems](#2277-limitations-and-open-problems)
  - [Evaluation Cost](#evaluation-cost)
  - [Redundant Proposals](#redundant-proposals)
  - [Hardware Generalization](#hardware-generalization)
  - [Legality at Scale](#legality-at-scale)
  - [The Last-Mile Problem](#the-last-mile-problem)
- [Chapter Summary](#chapter-summary)

---

## 227.1 The Polyhedral Parameter Selection Problem

### What Polyhedral Compilation Gives You

A loop nest like matrix multiplication:

```c
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++)
        for (int k = 0; k < N; k++)
            C[i][j] += A[i][k] * B[k][j];
```

is represented in the polyhedral model as a set of integer points in a 3-dimensional space `(i, j, k)` with domain constraints `0 ≤ i,j,k < N`. The iteration order is a *schedule* — a function from iteration points to execution timestamps.

The polyhedral model permits any schedule that satisfies dependence constraints (the *legality condition*). The set of legal schedules is infinite. Among all legal schedules, we want the one that minimizes runtime.

### The Parameter Space

Even after fixing the *structure* of a transformation (tile the outermost two loops, then vectorize the innermost), the *parameters* are free:

- Tile size for loop `i`: `{1, 2, 4, 8, 16, 32, 64, 128}`
- Tile size for loop `j`: same
- Vector width for loop `k`: `{1, 4, 8, 16}` (SSE, AVX, AVX-512)
- Fusion: fuse loops A and B, or keep separate
- Interchange: which loop is outermost after tiling

For a 3-loop nest with 2 tiled loops, the parameter space has `8 × 8 × 4 × 2 × 6 = 3,072` configurations. For real programs (5–7 nested loops, multiple candidate fusion groups), the space is exponential.

### Why Analytical Cost Models Fail

Pluto's cost model is based on:
1. Dependence distance minimization (minimize carried dependences)
2. Parallelism maximization (maximize outermost loop trip count)
3. Simple memory footprint estimates (tile to fit in L1/L2)

These heuristics work well on textbook loop nests but fail when:
- The L1/L2 size varies across execution (cache contention with other threads)
- Prefetchers hide memory latency in ways the model does not account for
- The CPU's out-of-order window interleaves multiple iterations in ways that change the effective latency
- NUMA topology makes memory placement the dominant factor, not cache size

---

## 227.2 LOOPer: Deep Learning Cost Models for Affine Transformations

LOOPer (arXiv 2403.11522) replaces Pluto's analytical cost model with a learned regression model that predicts the runtime of a transformed loop nest.

### Architecture

```
Input:
  - Polyhedral representation of the loop nest (trip counts, array accesses)
  - Proposed schedule (tiling factors, loop ordering)
  → Feature vector (handcrafted, 48 dimensions)

Model:
  3-layer MLP, 512 units per layer, ReLU activation
  Output: predicted log(runtime) ∈ ℝ (regression)

Training:
  Dataset: 50,000 (loop nest, schedule, measured runtime) triples
           collected by exhaustive sampling on target hardware
  Loss: mean squared error on log(runtime)
```

### Feature Vector Construction

LOOPer's 48-dimensional feature vector captures polyhedral properties:

| Feature group | Dimensions | Description |
|--------------|-----------|-------------|
| Loop structure | 6 | Trip counts (normalized), nesting depth |
| Memory access patterns | 12 | Array access strides, reuse distances (A, B, C) |
| Schedule parameters | 10 | Tiling factors, loop permutation vector |
| Parallelism metrics | 6 | Innermost/outermost parallel degree |
| Dependence metrics | 8 | Carried dependence count per loop level |
| Target hardware | 6 | L1/L2/L3 cache size, SIMD width (target-specific) |

Features are extracted from Polly's polyhedral representation using the isl library.

### Training Data Collection

The training dataset is collected by:
1. Sampling loop nests from benchmark suites (Polybench, SPEC, NumPy)
2. Enumerating a subset of the legal schedule space for each nest
3. Compiling and running each (nest, schedule) pair on the target hardware
4. Recording measured wallclock time

This is expensive (50,000 × ~10 ms = ~8 minutes per target hardware configuration) but is a one-time offline cost.

### Using LOOPer for Schedule Selection

At optimization time:

```python
def looper_select_schedule(loop_nest, candidate_schedules):
    features = [extract_features(loop_nest, sched) for sched in candidate_schedules]
    predicted_runtimes = looper_model.predict(features)
    best_idx = np.argmin(predicted_runtimes)
    return candidate_schedules[best_idx]
```

LOOPer evaluates all candidate schedules in ~1 ms (batch MLP inference) and returns the predicted best. No compilation or execution is required at selection time.

### Empirical Results

On Polybench v4.2, Polly baseline vs LOOPer on x86-64 (Intel Core i9-12900K):

| Benchmark | Polly speedup vs `-O3` | LOOPer speedup vs `-O3` |
|-----------|----------------------|------------------------|
| gemm | 1.8× | 2.4× |
| syrk | 1.5× | 2.1× |
| conv-2d | 2.1× | 2.6× |
| heat-3d | 1.4× | 1.9× |
| ludcmp | 1.2× | 1.6× |
| **mean** | **1.6×** | **2.1×** |

LOOPer achieves ~30% additional speedup over Polly's default cost model at the cost of hardware-specific training data.

---

## 227.3 Agentic Auto-Scheduling with Empirical Feedback

LOOPer's limitation: it predicts from a pre-enumerated candidate set. It cannot *explore* outside that set — if the optimal schedule is not in the candidates, LOOPer cannot find it.

Agentic auto-scheduling (arXiv 2511.00592) uses an LLM as an *explorer*: the agent proposes schedules based on reasoning about the loop structure, compiles and measures them, and iterates based on the measurements.

### The Agentic Loop

```
Initialize:
  context = polyhedral representation of loop nest (as text)
  measurements = []
  best_schedule = None, best_runtime = ∞

Iteration (repeat 5–20 times):
  1. Prompt the LLM:
     "Loop nest: {context}
      Previous measurements: {measurements}
      Suggest a schedule (tiling factors and loop ordering) to try next.
      Polyhedral legality is guaranteed by the compiler; focus on performance."

  2. Parse LLM response → extract schedule parameters

  3. Validate legality: isl_check_legality(schedule)
     • Illegal → feedback to LLM: "Schedule is illegal, reason: {violation}"
     • Legal → continue

  4. Compile: polly --tile-sizes={schedule} loop_nest.ll -o /tmp/optimized.o
     clang /tmp/optimized.o -o /tmp/benchmark

  5. Measure: wallclock = run(/tmp/benchmark, repeats=5, min=True)

  6. Update measurements: measurements.append((schedule, wallclock))
     Update best if improved

Output: best_schedule
```

### LLM Prompting Strategy

The LLM receives the loop nest as a textual MLIR `affine.for` nest plus previous measurement results:

```
System: You are an expert in polyhedral loop optimization. Your goal is to
        find tile sizes and loop orderings that minimize runtime on a modern
        x86-64 processor with AVX-512 and 32MB L3 cache.

User: Loop nest (MLIR affine form):
  affine.for %i = 0 to 1024 {
    affine.for %j = 0 to 1024 {
      affine.for %k = 0 to 1024 {
        %a = affine.load %A[%i, %k] : memref<1024x1024xf32>
        %b = affine.load %B[%k, %j] : memref<1024x1024xf32>
        %c = affine.load %C[%i, %j] : memref<1024x1024xf32>
        %sum = arith.addf %c, arith.mulf %a, %b
        affine.store %sum, %C[%i, %j] : memref<1024x1024xf32>
      }
    }
  }

  Previous measurements:
  - tile_i=32, tile_j=32, order=ijk: 0.82s
  - tile_i=64, tile_j=64, order=ijk: 0.71s
  - tile_i=128, tile_j=64, order=ijk: 0.68s

  Suggest next schedule:
```

The LLM proposes a schedule based on its knowledge of cache blocking and the previous measurements. The polyhedral model validates legality; actual measurement provides the ground truth.

### Convergence Behavior

On 128 Polybench benchmarks:

| Metric | Value |
|--------|-------|
| Mean iterations to converge | 8.3 |
| Time per iteration (compile + run) | ~2 seconds |
| Total time per benchmark | ~17 seconds |
| Mean speedup vs Polly (LOOPer) | 28% additional |
| Mean speedup vs `-O3` | 2.7× (vs LOOPer's 2.1×) |

The additional 28% over LOOPer comes from exploring schedules outside LOOPer's pre-enumerated candidate set — the LLM discovers novel configurations that the uniform sampling in LOOPer's training set did not cover.

---

## 227.4 Integration with Polly

Polly is LLVM's polyhedral optimizer (Chapters 131–132). Integrating LOOPer or the agentic system replaces Polly's internal cost model.

### Polly's Schedule Representation

Polly represents schedules as isl schedule trees (`isl_schedule`). The agentic system produces a schedule as tiling parameters; translation to isl:

```cpp
// Convert agentic schedule parameters to isl schedule tree
isl_schedule* buildSchedule(isl_ctx* ctx, 
                             const ScheduleParams& params,
                             const Scop& S) {
    // Start from Pluto's legal schedule
    isl_schedule* base = computePlutoSchedule(S);
    
    // Apply tiling
    isl_schedule* tiled = applyTiling(base, params.tile_i, params.tile_j, ctx);
    
    // Apply loop interchange
    isl_schedule* reordered = applyPermutation(tiled, params.loop_order, ctx);
    
    return reordered;
}
```

### Setting the Schedule

Polly's `Scop::setSchedule()` accepts an externally computed `isl_schedule`:

```cpp
// After the agentic system selects a schedule:
Scop& S = ...; // from Polly's ScopInfo pass
isl_schedule* agenticSchedule = buildSchedule(S.getIslCtx(), bestParams, S);

// Validate legality before setting
if (!S.isValid(agenticSchedule)) {
    llvm::errs() << "Agentic schedule failed legality check\n";
    // Fallback to Polly's default
} else {
    S.setSchedule(agenticSchedule);
}
```

Source: `polly/include/polly/ScopInfo.h`.

### Fallback Chain

```
1. Try agentic schedule (best performance, 17s overhead)
2. If no improvement over Pluto or compile timeout: try LOOPer schedule (1 ms overhead)
3. If no improvement: use Polly's default Pluto schedule
4. If Polly disabled: pass through to LLVM's loop optimizer
```

The fallback chain ensures that the agentic optimization never makes performance worse than Polly's baseline.

---

## 227.5 Integration with MLIR Affine Dialect

MLIR's `affine` dialect (Chapter 130) provides a direct representation of polyhedral loop nests at the IR level. Unlike Polly (which operates on LLVM IR and must infer polyhedral structure via SCEV analysis), MLIR's affine operations carry the polyhedral structure as explicit IR.

### Affine Loop Transformation Passes

MLIR's transformation passes accept tile sizes as command-line parameters:

```bash
# Tile the outermost two loops with tile_i=32, tile_j=64
mlir-opt --affine-loop-tile="tile-sizes=32,64" matmul.mlir

# Loop interchange: move k loop to outermost position
mlir-opt --affine-loop-interchange matmul.mlir

# Parallelize the outermost loop
mlir-opt --affine-parallelize matmul.mlir
```

The agentic system drives these passes by setting the `tile-sizes` parameter:

```python
def apply_mlir_schedule(mlir_file, schedule_params):
    tile_str = ",".join(str(t) for t in schedule_params.tile_sizes)
    cmd = [
        "mlir-opt",
        f"--affine-loop-tile=tile-sizes={tile_str}",
        "--affine-loop-fusion",
        "--affine-parallelize",
        mlir_file, "-o", "/tmp/scheduled.mlir"
    ]
    subprocess.run(cmd, check=True)
    # Then lower to LLVM IR for measurement
    subprocess.run(["mlir-translate", "--mlir-to-llvmir",
                    "/tmp/scheduled.mlir", "-o", "/tmp/scheduled.ll"], check=True)
```

### Advantages Over Polly for Agentic Optimization

1. **Explicit structure**: the affine nest structure is directly visible as IR, making it easier to extract features for the LLM prompt without SCEV analysis

2. **Modular passes**: each transformation is a separate pass that can be selectively applied; the agentic system can mix and match

3. **Dialect-aware reasoning**: the LLM can reason about `affine.for` directly, whereas LLVM IR loop representations require indirect reasoning through induction variable patterns

4. **Integration with downstream compilation**: `linalg.generic` operations lowered from affine loops can be further transformed by vectorization passes in the vector dialect

---

## 227.6 Comparison with Ansor/AutoTVM and Halide Autoscheduler

### Ansor (TVM, arXiv 2006.06762)

Ansor is TVM's auto-scheduler for tensor computations. It uses a sketch-based search:
1. Generate a set of *sketches* (structural templates: split, reorder, compute-at)
2. For each sketch, use an ML cost model (XGBoost) to select tile sizes
3. Evaluate top candidates empirically

Ansor vs LOOPer/agentic:
- **IR abstraction**: Ansor operates on TVM's `te.compute` (tensor expression) IR, not LLVM IR; the two systems target different compilers
- **Search strategy**: Ansor separates structure (sketch) from parameters (tile sizes); agentic search handles both jointly
- **Cost model**: Ansor uses XGBoost on hardware-independent features; LOOPer uses a deep MLP on polyhedral features

### Halide Autoscheduler (Li et al. 2021)

Halide's autoscheduler uses beam search guided by an ML cost model:
- **Algorithm/schedule separation**: Halide's fundamental design principle
- **Cost model**: a deep neural network trained on (algorithm, schedule, runtime) triples
- **Search**: beam search over the schedule DAG

Halide autoscheduler vs LOOPer/agentic:
- Halide's algorithm/schedule separation is cleaner than Polly's post-hoc analysis
- Halide's cost model is conceptually identical to LOOPer (deep ML on structured features)
- Agentic approach provides broader exploration at higher evaluation cost

### Comparison Table

| System | IR | Search | Cost model | Empirical feedback | Portability |
|--------|----|---------|-----------|--------------------|-------------|
| Pluto | isl | Analytical | Dependence distance | No | High |
| Ansor | TVM te.compute | Sketch+ML | XGBoost | Yes (top-k) | TVM targets |
| Halide autoscheduler | Halide func | Beam search | Deep MLP | Yes (beam) | Halide targets |
| LOOPer | LLVM IR / Polly | Enumeration | Deep MLP | No | LLVM targets |
| Agentic | MLIR affine | LLM+empirical | None (empirical) | Yes (each iteration) | LLVM/MLIR targets |

---

## 227.7 Limitations and Open Problems

### Evaluation Cost

The agentic system requires compiling and running the benchmark 5–20 times to converge. For a benchmark that takes 10 seconds to run, one agentic search takes 50–200 seconds. This is acceptable for ahead-of-time optimization of hot computational kernels (where optimization is a one-time cost), but not for interactive compilation.

Mitigation: use LOOPer as a fast first pass; invoke the agentic system only when LOOPer's improvement is < 10% (indicating the search space is complex enough to warrant deeper exploration).

### Redundant Proposals

LLMs frequently propose schedules they have already proposed (or very similar ones). Few-shot memory — including all previous measurements in the context — mitigates this but does not eliminate it. Vector database retrieval of similar past measurements could further reduce redundancy.

### Hardware Generalization

LOOPer's deep MLP is trained on one specific hardware configuration (CPU model, cache sizes). A model trained on an Intel Core i9 may perform poorly on an AMD EPYC server (different cache hierarchy) or on an ARM Neoverse server (different SIMD capabilities). The agentic system avoids this problem (empirical measurement on the actual target), but LOOPer requires retraining per target.

### Legality at Scale

For large loop nests (7+ nested loops with complex dependences), verifying the legality of all proposed schedules via isl can take seconds per schedule. The agentic loop's empirical measurement step already includes compilation; the legality check adds < 1% overhead. But for static tools that cannot afford empirical measurement, legality verification is a bottleneck.

### The Last-Mile Problem

Neither LOOPer nor the agentic system reaches hand-tuned library performance for large matrix multiplications. On 1024×1024 float GEMM:

| Implementation | GFLOPs |
|----------------|--------|
| `-O3` only | 18 |
| Polly + Pluto | 32 |
| LOOPer | 42 |
| Agentic (20 iters) | 54 |
| OpenBLAS (hand-tuned) | 89 |
| Intel MKL (hand-tuned) | 93 |

The gap between agentic (54 GFLOPs) and MKL (93 GFLOPs) comes from:
1. Micro-kernel design (hand-vectorized inner loops using AVX-512 intrinsics)
2. Multi-level tiling (L1/L2/L3 each with separate tile sizes)
3. Cache-oblivious algorithms not expressible as affine schedules
4. NUMA-aware data placement

Closing this gap requires going beyond the affine schedule parameter selection that polyhedral optimization handles, into the micro-kernel territory that currently requires hand-written assembly.

---

## Chapter Summary

- Polyhedral compilation guarantees transformation correctness; the performance bottleneck is parameter selection (tile sizes, loop ordering, fusion decisions) — the exponential search space that analytical cost models cannot navigate
- LOOPer replaces Pluto's analytical cost model with a deep MLP trained on (loop nest, schedule, measured runtime) triples; achieves 2.1× speedup vs `-O3` on Polybench, 30% better than Polly's default
- Agentic auto-scheduling uses an LLM as an explorer: the agent proposes schedules, the polyhedral model validates legality, empirical measurement provides ground-truth feedback; 8 iterations to converge, 2.7× speedup vs `-O3`
- Integration with Polly: the agentic/LOOPer schedule is passed to `Scop::setSchedule(isl_schedule*)` after legality validation; a fallback chain ensures no regression below Polly's baseline
- MLIR affine dialect integration: `--affine-loop-tile=tile-sizes=...` and related passes accept parameters directly; LLM reasoning over `affine.for` IR is more natural than over LLVM IR loop patterns
- Comparison: LOOPer is conceptually equivalent to Halide's autoscheduler (deep ML cost model on structured features); agentic search is distinct in using LLM reasoning plus empirical measurement without a pre-trained cost model
- Remaining gaps: evaluation cost (50–200 seconds per benchmark), hardware-specific model training, and a 40% performance gap vs hand-tuned BLAS libraries that require micro-kernel design beyond the affine schedule parameter space

---

@copyright jreuben11
