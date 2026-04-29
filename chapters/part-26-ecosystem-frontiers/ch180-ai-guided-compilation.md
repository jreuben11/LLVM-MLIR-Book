# Chapter 180 — AI-Guided Compilation

*Part XXVI — Ecosystem and Frontiers*

Machine learning is no longer at the periphery of compiler development. From reinforcement-learned inlining policies already deployed in LLVM mainline, to neural superoptimizers that propose verified peephole rewrites, to LLM-assisted TableGen authoring, the boundary between compiler engineering and machine learning has dissolved. This chapter maps the full landscape — examining deployed production systems, research-grade techniques with active upstreaming efforts, and speculative horizons that will define the compiler work of the next decade. The common thread running through every technique described here is that the compiler's decision problem is the ML problem: given features extracted from program structure, predict the action that leads to the best outcome. Readers should be comfortable with C++ and have passing familiarity with how neural networks are trained; deep ML expertise is not required.

## 180.1 MLGO: ML-Guided Optimization in LLVM

[Chapter 66 — The ML Inliner and ML Register Allocation](../part-10-analysis-middle-end/ch66-ml-inliner-and-regalloc.md) introduced MLGO's two deployed components — the `MLInlineAdvisor` and the ML eviction advisor — covering their feature vectors, TFLite runtime modes, and published performance results. This section focuses on the infrastructure layer that both components share: the `MLModelRunner` abstraction, the mechanics of switching between training and deployment, and the precise C++ API that advisor authors use when integrating a new ML policy.

### 180.1.1 The `MLModelRunner` Abstraction

The relationship between an advisor and its model follows a clean separation of concerns:

```
┌─────────────────────────────┐
│  MLInlineAdvisor             │  ← advisor business logic
│  MLEvictAdvisor              │
└────────────┬────────────────┘
             │ owns
             ▼
┌─────────────────────────────┐
│  MLModelRunner (base class)  │  ← evaluate() / getTensor<T>()
├─────────────────────────────┤
│  ReleaseModeModelRunner      │  ← AOT C++ class, direct call
│  DevelopmentModeModelRunner  │  ← gRPC pipe to Python server
│  NoOpModelRunner             │  ← fixed default, for testing
└─────────────────────────────┘
```

Both the inliner and the eviction advisor delegate model evaluation through a single base class defined in [`llvm/include/llvm/Analysis/MLModelRunner.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Analysis/MLModelRunner.h). The public contract is minimal:

```cpp
class MLModelRunner {
public:
  enum class Kind { Unknown, Release, Development, NoOp };

  virtual ~MLModelRunner() = default;

  template <typename T>
  T evaluate() {
    return *reinterpret_cast<T *>(evaluateUntyped());
  }

  template <typename T, typename FeatureT>
  T *getTensor(FeatureT Feature) {
    return reinterpret_cast<T *>(
        getTensorUntyped(static_cast<size_t>(Feature)));
  }

  Kind getKind() const { return Kind_; }

  static std::unique_ptr<MLModelRunner>
  create(LLVMContext &Ctx, Kind K, StringRef ModelPath,
         int64_t InteractiveChannelBaseFD = -1);

protected:
  MLModelRunner(LLVMContext &Ctx, Kind K) : Ctx(Ctx), Kind_(K) {}
  virtual void *evaluateUntyped() = 0;
  virtual void *getTensorUntyped(size_t Index) = 0;

  LLVMContext &Ctx;

private:
  Kind Kind_;
};
```

The two type-safe wrappers — `getTensor<T>(Feature)` to write input features and `evaluate<T>()` to read the model output — are the entire interface an advisor sees. All tensor lifecycle management, memory layout, and model execution are hidden behind `evaluateUntyped()`.

The static factory `MLModelRunner::create()` dispatches on `Kind`:

- `Kind::Release` constructs a `ReleaseModeModelRunner`. This runner wraps the AOT-compiled model, which was emitted as a C++ class during the LLVM build via `tfcompile`. Calling `evaluate()` becomes a direct non-virtual C++ function call with no allocation.
- `Kind::Development` constructs a `DevelopmentModeModelRunner` (also called `InteractiveModelRunner` in some LLVM versions). This runner opens a bidirectional pipe or gRPC channel to an out-of-process Python training server. The compiler becomes a feature-generation engine: it emits tensors over the channel, waits for the server to send back a decision, applies it, and reports the reward.

The `NoOp` kind produces a runner that always returns a fixed default value — used for test coverage and for `-enable-ml-inliner=default` which exercises the advisor code path without ML involvement.

The reason for the template wrapper rather than a plain virtual method returning `void *` is type safety at the feature-population site. Advisor code like:

```cpp
*Runner->getTensor<int64_t>(InlineFeatures::callee_basic_block_count)
    = static_cast<int64_t>(CalleeBBCount);
```

produces a compiler error at the call site if `InlineFeatures::callee_basic_block_count` does not correspond to an `int64_t` tensor slot, catching feature/type mismatches during development rather than at model-evaluation time. The underlying `getTensorUntyped(size_t Index)` is what the concrete runner subclass implements; it returns a pointer into the pre-allocated tensor buffer managed by the runner's input arena.

### 180.1.2 Switching Modes: CMake and the `create()` Factory

The `LLVM_HAVE_TF_AOT` preprocessor macro gates `ReleaseModeModelRunner`. When built with `-DLLVM_ENABLE_TENSORFLOW_AOT=ON`, `LLVM_HAVE_TF_AOT` is defined and the generated AOT header (e.g., `InlinerSizeModel.h`) is compiled in. Without it, `create(Kind::Release, ...)` falls back to the heuristic path.

`LLVM_HAVE_TF_API` gates `DevelopmentModeModelRunner`. Building with `-DLLVM_ENABLE_TENSORFLOW_API=ON` links against `libtensorflow.so` and enables the interactive channel. This is strictly a development-time dependency; release builds never carry it.

```bash
# Development build for training data collection
cmake -DLLVM_ENABLE_TENSORFLOW_API=ON \
      -DTF_PATH=/opt/tensorflow \
      -DLLVM_ENABLE_PROJECTS="clang" \
      ../llvm-project/llvm

# Release build with AOT-compiled model
cmake -DLLVM_ENABLE_TENSORFLOW_AOT=ON \
      -DTENSORFLOW_AOT_PATH=/opt/tf-aot \
      -DLLVM_ENABLE_PROJECTS="clang" \
      ../llvm-project/llvm
```

To activate the ML inliner at compile time:

```bash
clang -O2 -mllvm -enable-ml-inliner=release     input.cpp  # release model
clang -O2 -mllvm -enable-ml-inliner=development  input.cpp  # gRPC to Python server
clang -O2 -mllvm -enable-ml-inliner=default       input.cpp  # no model, heuristic
```

### 180.1.3 The RL-Trained Inliner: Features and Training Loop

The `MLInlineAdvisor` is implemented in [`llvm/lib/Analysis/MLInlineAdvisor.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Analysis/MLInlineAdvisor.cpp). Its feature set is declared in `InlineFeatures.def` via an X-macro and includes roughly thirty per-call-site values: caller and callee instruction counts, basic block counts, call graph node depth, call site frequency relative to function entry, the estimated inline cost, whether the callee is used elsewhere (speculability), and several derived statistics about the callee's argument types and uses. For each candidate call site, the advisor calls `getTensor<int64_t>(InlineFeatures::callee_basic_block_count)` and so on to populate the input tensor, then calls `evaluate<bool>()` to obtain the decision.

The complete training loop looks like this:

```
Phase 1 — Data Collection (development mode)
  ┌─────────────────────────────────────────────────────────┐
  │  clang -O2 -enable-ml-inliner=development               │
  │       ↓                                                  │
  │  For each call site:                                     │
  │    1. Fill input tensor (InlineFeatures)                 │
  │    2. Send tensor → gRPC channel → Python server         │
  │    3. Python server returns decision (bool)              │
  │    4. Apply decision; record (features, decision) pair   │
  │    5. After compilation: measure binary size delta       │
  │    6. Write (features, decision, reward) to trace file   │
  └─────────────────────────────────────────────────────────┘

Phase 2 — Policy Training (Python, ml-compiler-opt)
  ┌─────────────────────────────────────────────────────────┐
  │  Read trace files into replay buffer                    │
  │  PPO gradient step: maximise E[reward | features]       │
  │  Export updated weights → inliner_model.tflite          │
  └─────────────────────────────────────────────────────────┘

Phase 3 — Model Integration
  ┌─────────────────────────────────────────────────────────┐
  │  tfcompile inliner_model.tflite → InlinerModel.h (C++)  │
  │  Rebuild LLVM with LLVM_ENABLE_TENSORFLOW_AOT=ON        │
  │  ReleaseModeModelRunner wraps InlinerModel::Run()       │
  └─────────────────────────────────────────────────────────┘
```

The reward signal during training is binary size delta plus a weighted performance delta collected from microbenchmarks. The training loop, hosted at [github.com/google/ml-compiler-opt](https://github.com/google/ml-compiler-opt), repeats phases 1–3 until the model converges on a stable policy.

The `InlineAdvice` subclass `MLInlineAdvice` records whether the decision was accepted and — on `recordUnsuccessfulInliningImpl` — annotates the module with metadata so the training harness can compute a per-call-site outcome. This is the feedback mechanism: even a "don't inline" decision that was wrong gets recorded, because the training server can detect that a later inlining step had to compensate.

The policy network itself is small — a two-layer MLP with around 64 hidden units. Small model size is deliberate: each inlining decision during compilation must evaluate in microseconds. The AOT path compiles the MLP's matrix multiplications to unrolled SIMD instructions via XLA; the resulting `InlinerModel::Run()` function executes in under 1 µs on modern hardware.

### 180.1.4 ML-Based Register Allocator Eviction

The interface lives in [`llvm/include/llvm/CodeGen/RegAllocEvictionAdvisor.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/RegAllocEvictionAdvisor.h). The greedy allocator calls:

```cpp
virtual bool shouldEvict(const LiveInterval &A,
                         bool IsHint,
                         const LiveInterval &JoinVInt,
                         bool BreaksHint) const;
```

The ML implementation is in [`llvm/lib/CodeGen/MLRegallocEvictAdvisor.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/MLRegallocEvictAdvisor.cpp) (the interface and default policy are in the sibling file `RegAllocEvictionAdvisor.cpp`). At each potential eviction, the advisor fills a feature tensor covering both the current candidate evictee and the new interval seeking a register: live range weight (use frequency × type spill cost), cascade count (how many prior evictions led to this interval), number of available physical registers for each party, and whether the assignment would break a copy-hint from coalescing. The model scores each eviction candidate; the greedy allocator selects the highest-scoring one.

Activation follows the same CMake path and the same `-mllvm -regalloc-enable-advisor=release` flag described in Ch66. The training data consists of (feature, eviction choice, spill cost delta) tuples; the reward is post-allocation total spill cost.

One structural difference from the inliner is that the eviction problem is a ranking problem rather than a binary decision: given a set of candidate eviction targets (all live intervals currently interfering with the new interval), the model assigns a score to each and the greedy allocator evicts the one with the highest score. The feature tensor therefore has a fixed maximum number of candidate slots — if fewer candidates are available, unused slots are zeroed — and the model output is a per-slot float32 score vector. The interface exposes this via `getBestEvicteeForVirtReg`, which the greedy allocator calls and then acts on the returned index.

---

## 180.2 ML-Based Cost Models for Scheduling and Tiling

### 180.2.1 The Limits of Static Latency Tables

LLVM's `TargetSchedModel`, populated by `SchedMachineModel` records in TableGen, expresses instruction latencies and throughputs as compile-time constants. These constants are derived from processor documentation, `llvm-mca` measurements, and hand-tuning. They are averages: a `vmulps` on Skylake has a documented throughput of 0.5 cycles, but the actual throughput in context depends on port pressure, micro-op fusion, and L1 cache state. Static tables cannot capture this context-dependence.

`llvm-mca` closes some of the gap by simulating an out-of-order core's dispatch and retirement queues at the block level, producing per-instruction throughput estimates that reflect contention:

```bash
# Analyse a hot loop body
/usr/lib/llvm-22/bin/llvm-mca -mcpu=skylake -iterations=1000 \
    -bottleneck-analysis hot_loop.s
```

The output provides `Resource pressure per iteration` broken down by execution port. Feeding thousands of these measurements into a model produces learned latency tables that are more accurate than the static version for the target microarchitecture.

### 180.2.2 Ithemal: LSTM Over Instruction Sequences

Ithemal (Mendis et al., ICML 2019) trains an LSTM-based model over Intel instruction encoding to predict per-block throughput. The input is a sequence of instructions, each encoded as a tuple of (opcode, source operand types, destination operand type); the model outputs a scalar throughput in cycles. Training data comes from `uops.info`, a crowd-sourced database of measured instruction latencies and port usage for x86 processors.

The key insight is that the model learns implicit data-hazard and structural-hazard patterns without being given a processor model. On Haswell and Skylake, Ithemal achieves roughly 10% mean absolute error on block throughput — comparable to `llvm-mca` for the same target, but without a hand-written `SchedMachineModel`.

Integrating a learned throughput model into LLVM's machine scheduler requires subclassing `TargetSchedModel` and overriding `computeOperandLatency` and `getResourceFactor`, routing queries to the inference runtime. No such integration has been upstreamed into LLVM 22; it remains a research integration point.

A concrete integration path would look like the following: for a specific microarchitecture target (say, a custom RISC-V accelerator without a vendor-supplied `SchedMachineModel`), compile a representative set of hot loops with different instruction orderings, measure actual throughput via hardware performance counters, and train a gradient-boosted tree (GBT) to map (opcode sequence, register class sequence, preceding instruction context window) to observed throughput. The GBT is serialised to a platform-independent format and loaded at compiler startup. The custom `TargetSchedModel::computeOperandLatency` queries the GBT for pairs of instructions that have data dependencies; the returned latency overrides the table default. For compiler backends targeting AI accelerators — where vendor-published latency specs are often preliminary or incorrect — this approach provides a measurably more accurate scheduler cost model without waiting for the vendor to update their LLVM TableGen files.

### 180.2.3 Ansor and TVM Meta-Schedule

TVM's `meta_schedule` (formerly Ansor) frames tensor program optimization as a search over the space of valid loop transformations. The search space is a `Schedule` object — an ordered sequence of tiling, vectorization, unrolling, and fusion decisions applied to a `TIR` (Tensor IR) program.

The cost model predicts normalized throughput (ops per second divided by theoretical peak) for each candidate schedule without running it on hardware. An XGBoost ensemble, trained on measured throughput data from previous search runs on the target device, maps feature representations of the schedule (tile sizes, memory access patterns, parallelism degree) to predicted throughput. The evolutionary search samples schedules, scores them with the cost model, measures the top candidates on hardware, and updates the model with the new measurements:

```python
import tvm
from tvm import meta_schedule as ms

mod = tvm.IRModule.from_expr(matmul_workload)
target = tvm.target.Target("llvm -mcpu=skylake-avx512")

with ms.Profiler() as profiler:
    database = ms.tune_tir(
        mod=mod,
        target=target,
        max_trials_global=2000,
        num_trials_per_iter=64,
        work_dir="./tune_results",
    )

sch = ms.tir.Schedule(mod)
database.get_top_k(database.all_tuning_records(sch.mod)[0], top_k=1)
```

TVM's MLIR backend emits the winning schedule as a sequence of `linalg` operations with tiling attributes that map directly to `linalg.tile_using_forall`. The tile sizes chosen by meta-schedule can thus seed MLIR's tiling infrastructure for downstream lowering.

The search strategy deserves closer attention. Each candidate schedule is represented as a "sketch" — a partial program with free parameters (tile sizes, split factors, unroll depths) — and the evolutionary algorithm fills the free parameters by combining candidates from the population. The cost model evaluates thousands of filled-in candidates per second by scoring their feature vectors; only the top-scoring candidates are actually compiled and measured on hardware. This creates an inner loop (cost-model scoring, milliseconds per candidate) and an outer loop (hardware measurement, seconds per candidate). The cost model is updated after each hardware measurement batch with the new (feature, measured-throughput) pairs, so it improves as tuning progresses.

For CPU targets using the `llvm` backend, the winning schedule's `TIR` is lowered through TVM's LLVM code generator, producing LLVM IR that is then optimised and compiled with Clang. The tile sizes and vectorisation decisions made by meta-schedule thus interact with LLVM's own loop vectoriser and unroller: if meta-schedule tiles a loop to a width that matches a SIMD register, LLVM's vectoriser recognises the inner loop as vectorisable without needing auto-vectorisation to discover the tiling.

### 180.2.4 IREE's External Tuning Spec

IREE's compilation pipeline for GPU and CPU targets supports externalising tile-size and workgroup-size decisions as a `tuning.mlir` file containing `transform.named_sequence` ops. The file is passed at compile time and overrides the default heuristic:

```bash
iree-compile \
    --iree-hal-target-backends=llvm-cpu \
    --iree-codegen-tuning-spec-path=tuning.mlir \
    matmul.mlir \
    -o matmul.vmfb
```

The tuning workflow is a measurement loop: compile with a candidate spec, benchmark with `iree-benchmark-module`, record throughput, update the spec, and repeat. Because the spec is plain MLIR, it is version-controlled and human-readable. The `transform.named_sequence` ops apply named transformations — `transform.iree.tile_using_forall`, `transform.structured.vectorize` — whose parameters are the tile sizes being tuned. This design decouples the search strategy (Python script, Bayesian optimiser, or manual engineering) from the transformation logic.

A minimal tuning spec that overrides tile sizes for a matmul looks like:

```mlir
module @tuning_spec {
  transform.named_sequence @__kernel_config(
      %variant_op: !transform.any_op {transform.consumed}) {
    %matmul = transform.structured.match ops{["linalg.matmul"]}
                in %variant_op : (!transform.any_op) -> !transform.any_op
    transform.iree.tile_using_forall %matmul
        tile_sizes [64, 64, 32]
        mapping [#iree_codegen.workgroup_mapping<y>,
                 #iree_codegen.workgroup_mapping<x>]
      : (!transform.any_op) -> (!transform.any_op, !transform.any_op)
    transform.iree.apply_patterns to %variant_op {
      transform.apply_patterns.iree.fold_fill_into_pad
      transform.apply_patterns.linalg.tiling_canonicalization
    } : !transform.any_op
    transform.yield
  }
}
```

Each iteration of the tuning loop generates a new spec with different `tile_sizes` values, recompiles, benchmarks, and retains the configuration that maximised throughput. The MLIR serialisation format ensures that the winning configuration is a first-class artifact that can be checked into source control alongside the model and deployed without regenerating it.

---

## 180.3 Triton's Autotuner

Triton kernels are JIT-compiled for a specific GPU, and their performance depends critically on tile sizes, pipeline depth (number of software-pipelined stages), and warp count. Triton's autotuner explores a user-specified configuration space and caches the best result per input shape.

### 180.3.1 The `@triton.autotune` Decorator

```python
import triton
import triton.language as tl
import torch

@triton.autotune(
    configs=[
        triton.Config({"BLOCK_M": 128, "BLOCK_N": 256, "BLOCK_K": 64},
                      num_warps=8, num_stages=4),
        triton.Config({"BLOCK_M": 64,  "BLOCK_N": 256, "BLOCK_K": 32},
                      num_warps=4, num_stages=4),
        triton.Config({"BLOCK_M": 128, "BLOCK_N": 128, "BLOCK_K": 32},
                      num_warps=4, num_stages=4),
        triton.Config({"BLOCK_M": 128, "BLOCK_N": 64,  "BLOCK_K": 32},
                      num_warps=4, num_stages=4),
        triton.Config({"BLOCK_M": 64,  "BLOCK_N": 128, "BLOCK_K": 64},
                      num_warps=4, num_stages=4),
    ],
    key=["M", "N", "K"],
)
@triton.jit
def matmul_kernel(
    A, B, C,
    M, N, K,
    stride_am, stride_ak,
    stride_bk, stride_bn,
    stride_cm, stride_cn,
    BLOCK_M: tl.constexpr,
    BLOCK_N: tl.constexpr,
    BLOCK_K: tl.constexpr,
):
    pid_m = tl.program_id(0)
    pid_n = tl.program_id(1)
    offs_m = pid_m * BLOCK_M + tl.arange(0, BLOCK_M)
    offs_n = pid_n * BLOCK_N + tl.arange(0, BLOCK_N)
    offs_k = tl.arange(0, BLOCK_K)
    acc = tl.zeros((BLOCK_M, BLOCK_N), dtype=tl.float32)
    for k in range(0, K, BLOCK_K):
        a = tl.load(A + offs_m[:, None] * stride_am
                      + (offs_k + k)[None, :] * stride_ak)
        b = tl.load(B + (offs_k + k)[:, None] * stride_bk
                      + offs_n[None, :] * stride_bn)
        acc += tl.dot(a, b)
    tl.store(C + offs_m[:, None] * stride_cm + offs_n[None, :] * stride_cn, acc)
```

The `key` parameter determines the cache key: two calls with the same `(M, N, K)` reuse the previously measured best configuration. A call with a new shape triggers a fresh benchmark round across all listed `triton.Config` instances.

### 180.3.2 The Benchmark Loop and Config Space

For each candidate config, Triton JIT-compiles the kernel with the config's `constexpr` values substituted, runs it for a fixed number of warmup and measurement iterations, and records the median execution time. The config with the lowest median time wins and is stored in the persistent cache.

The cache lives under `~/.triton/cache/` by default; the `TRITON_CACHE_DIR` environment variable overrides this. Each cache entry is keyed by a hash of the kernel source, the target GPU's capability string, and the shape key values. Pre-populating the cache for CI by copying cache directories from a reference machine eliminates autotuning latency in production builds.

When the config space is large (hundreds of tile-size combinations), exhaustive search is slow. Triton supports evolutionary search by implementing the optional `prune_configs_by` hook:

```python
def prune_configs(configs, named_args, **kwargs):
    # Discard configs where a tile dimension exceeds the problem size.
    M, N, K = named_args["M"], named_args["N"], named_args["K"]
    return [
        cfg for cfg in configs
        if cfg.kwargs["BLOCK_M"] <= M
        and cfg.kwargs["BLOCK_N"] <= N
        and cfg.kwargs["BLOCK_K"] <= K
    ]

@triton.autotune(configs=[...], key=["M", "N", "K"],
                 prune_configs_by={"early_config_prune": prune_configs})
@triton.jit
def matmul_kernel(...):
    ...
```

This prune hook runs before benchmarking and removes configurations that are structurally invalid for the current problem shape, reducing the search space without needing an ML cost model.

### 180.3.3 Triton's JIT Pipeline Internals

Understanding why autotune is effective requires understanding how Triton compiles a kernel. The `@triton.jit` decorator does not compile the kernel immediately; it wraps the Python function in a `JITFunction` object. The first call with a specific argument signature triggers compilation through the following pipeline:

```
Python function (AST)
       │
       ▼ triton.compiler.ast_to_ttir
TTIR (Triton IR — high-level tile ops)
       │
       ▼ triton.compiler.ttir_to_ttgir
TTGIR (Triton GPU IR — explicit warp layout)
       │
       ▼ triton.backends.nvidia.ttgir_to_llir  (or AMD equivalent)
LLVM IR with NVPTX/AMDGPU intrinsics
       │
       ▼ LLVM PTX/HSAIL backend
PTX / GFX assembly
       │
       ▼ CUDA driver / ROCm runtime
Executable GPU binary (cubin / HSACo)
```

Each `triton.Config` changes the tile dimensions (`BLOCK_M`, `BLOCK_N`, `BLOCK_K`) and the software pipeline depth (`num_stages`), which affects both the TTGIR layout decisions and the number of shared-memory double-buffering stages inserted during TTIR-to-TTGIR lowering. A config with `num_stages=4` inserts four copies of the prefetch loop, consuming four times the shared memory per threadblock compared to `num_stages=1`. If the config exceeds the GPU's shared memory limit, `triton.Config.pre_hook` can detect this before compilation via a call to `torch.cuda.get_device_properties`.

Because the JIT path is deterministic for a fixed config, the autotuner's per-config compilation cost is incurred once per (kernel, config, shape, GPU capability) tuple. Subsequent invocations with the same key retrieve the pre-compiled binary from the in-process cache, and the kernel is launched directly.

---

## 180.4 Profile Inference with ML

### 180.4.1 The PGO Gap

Profile-guided optimisation requires an instrumented binary, a representative workload, and a feedback loop. For many real-world scenarios — cold code paths that never execute during profiling, privacy-sensitive production workloads where attaching a profiler is prohibited, or open-source libraries compiled without access to their consumers' workloads — measured profiles are unavailable. When profile data is absent, LLVM's `BranchProbabilityInfo` falls back to static heuristics: loop back edges are taken with 88% probability, call to `exit` terminates with 100%, and so on.

AutoFDO partially closes this gap by converting hardware performance counter samples (Intel LBR, AMD IBS) into approximate branch probabilities without recompiling the binary. The `create_llvm_prof` tool reads a `perf.data` file and produces an `afdo` profile that can be passed to `clang -fprofile-sample-use=prof.afdo`. But hardware-counter profiles still require privileged access to production machines and still miss branches that are cold in the sampled window. More fundamentally, `perf` LBR sampling captures the last 32 conditional branches in the CPU's branch history buffer — branches that were never taken in any sampled execution window are invisible to AutoFDO, which means that error-handling code, rare fallback paths, and test branches for large input corner cases receive no profile data even when AutoFDO is in use.

### 180.4.2 ML-Based Profile Inference

ML-based profile inference attempts to predict per-branch probabilities and per-block frequencies from the static structure of the IR alone: loop nesting depth, loop trip count bounds, whether a branch guards a null pointer check, whether a block contains a call to a function whose name suggests it handles errors, and graph-level features of the CFG extracted via a GNN.

`BranchProbabilityInfo` in [`llvm/lib/Analysis/BranchProbabilityInfo.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Analysis/BranchProbabilityInfo.cpp) is the consumer: it populates an `EdgeProbability` map that downstream passes (loop rotation, code layout, inliner hot-path detection) query. An ML-inferred profile populates the same map with predicted probabilities rather than measured ones, making the inference transparent to all downstream consumers.

Google's "Learning to Fuse" line of work (Shi et al., 2021) demonstrates a GNN trained over the LLVM CFG that predicts whether a kernel fusion will improve end-to-end throughput — a related problem of predicting program behaviour from IR structure. Profile synthesis via GNN over the CFG is analogous: nodes are basic blocks (annotated with instruction mix features), edges are branches (annotated with loop membership and dominance depth), and the output is a per-edge probability distribution that sums to one at each branch instruction.

The predicted profile feeds directly into `BlockFrequencyInfo`, which LLVM uses to annotate blocks with integer frequency weights. Those weights influence inlining hotness thresholds, basic block placement (cold blocks are sunk to the end of the function to improve icache utilisation), and the loop vectoriser's profitability estimate.

### 180.4.3 Practical Status

No ML-based profile inference backend is present in LLVM 22 mainline. The research foundation is solid, and the consumer infrastructure (`BranchProbabilityInfo`, `BlockFrequencyInfo`) is already abstracted enough to accept externally generated profiles. Integration would require a loadable pass that queries an ML runtime and populates `BranchProbabilityInfo` before any profile-sensitive optimisation. The LLVM new pass manager's explicit analysis invalidation makes this straightforward to plumb.

The main technical challenge is not the inference infrastructure but the training data. A ground-truth profile for training requires both an IR-level snapshot (the function structure before optimisation) and a measured branch probability for every branch — which still requires instrumentation or hardware profiling. The ML model learns to extrapolate from the subset of functions that have measured profiles to the majority that do not, treating the measured profiles as a supervision signal and the IR structure as the input feature space. This self-supervised training regime has been demonstrated on function-level prediction; scaling it to the full codebase distribution is an open research problem.

The output format is critical for integration. `BranchProbabilityInfo` uses a 32-bit fixed-point weight per edge, normalised so that the weights on all outgoing edges from a block sum to a known total (defined by `BranchProbabilityInfo::getBranchWeightSum`). An ML-inferred probability of 0.93 for a branch must be converted to `(BW_taken, BW_not_taken)` integer weights that round-trip through the analysis without disturbing the invariant. The conversion is straightforward: multiply the float probability by `1 << 20` (the standard scale factor used for PGO weights) and round; this matches the format that `-fprofile-generate` produces.

---

## 180.5 LLM-Assisted Compiler Development

### 180.5.1 TableGen Authoring

TableGen is LLVM's domain-specific language for describing instruction semantics, scheduling models, and register files. Writing a new backend requires thousands of lines of TableGen: every instruction needs a DAG pattern for selection (`def ADD : I<..., [(set GPR:$dst, (add GPR:$src1, GPR:$src2))]>`), a scheduling class entry (`WriteIALU`), and register class membership declarations. For a target with several hundred instructions, this is weeks of repetitive work with a high error rate.

LLMs lower the barrier significantly. Given a description of a new instruction in natural language — "a three-operand fused multiply-add that takes two float32 source registers and one destination register, has 4-cycle latency, executes on port 0 or port 1" — an LLM generates a plausible TableGen record including the `FPFormat`, `SchedReadWrite` annotation, and `Pat` match. The developer reviews the output against the processor manual and corrects hallucinated constraint names (a common error: LLMs invent plausible-sounding register class names that do not exist in the target's TableGen files).

The verification step is non-negotiable: `llvm-tblgen --print-records` will reject malformed records or undefined references immediately, providing a fast feedback loop. Iterating with an LLM over `llvm-tblgen` error output converges quickly on a syntactically and structurally correct record, even when the LLM's first attempt is wrong.

A typical generated TableGen record for an FMA-style instruction on a hypothetical target might look like:

```tablegen
def FMADD_S : RVInstR4<0b00, FPR32:$rd, FPR32:$rs1, FPR32:$rs2, FPR32:$rs3,
                       (outs FPR32:$rd),
                       (ins FPR32:$rs1, FPR32:$rs2, FPR32:$rs3, frmarg:$frm),
                       "fmadd.s", "$rd, $rs1, $rs2, $rs3, $frm",
                       [(set FPR32:$rd,
                         (fma FPR32:$rs1, FPR32:$rs2, FPR32:$rs3))]> {
  let Predicates = [HasStdExtF];
  let SchedRW = [WriteFMA32, ReadFMA32, ReadFMA32, ReadFMA32];
}
```

The LLM's output for this record is usually correct in structure (the four `ReadFMA32` entries in `SchedRW` matching the three source registers plus the implicit rounding-mode read), but it frequently invents scheduling class names (`WriteFMA32`) that must be checked against the target's existing `SchedWriteRes` definitions. The developer's role shifts from writing records from scratch to reviewing and correcting a draft — a substantially faster workflow for large instruction sets.

The pattern holds for other TableGen use cases: generating `InstAlias` records for assembler pseudo-instructions, adding `Scheduling` annotations to existing instruction classes when porting a scheduler model to a new CPU variant, and writing `DiagnosticInfo` subtypes for new Clang diagnostics.

### 180.5.2 LLM-Guided Fuzzing

Coverage-guided fuzzing (LibFuzzer, AFL++) generates test inputs by mutating a seed corpus at the byte level, guided by which mutations reach new coverage. For MLIR dialects and LLVM pass pipelines, random byte-level mutations almost never produce valid IR — the mutation hits a type-check or a verifier assertion before reaching the interesting code.

LLM-guided fuzzing attacks this problem by replacing the byte-level mutator with an LLM that understands the grammar of the target language. Given a seed MLIR module and a coverage signal indicating which dialect operations have not been exercised, the LLM generates a syntactically valid mutation — inserting a new `linalg.generic` with different indexing maps, adding an `affine.if` inside a `scf.for`, or swapping integer and index types in a way that respects SSA dominance.

EvoFuzz applies evolutionary operators (crossover between two valid IR fragments, mutation via LLM edit) to MLIR dialects, reporting dialect-specific crashes that random byte-level mutation misses. CovRL-Fuzz replaces the fixed mutation strategy with an RL policy trained to maximise coverage: the policy observes the current coverage bitmap and selects from a set of LLM-generated candidate mutations, learning over time which mutation operators tend to reach new code regions.

The practical result is that LLM-guided fuzzers consistently find crashes in dialect lowering pipelines and in canonicalisation passes that depend on structural properties of the IR (e.g., a pattern that fires only when a `linalg.generic` has exactly one output with a specific indexing map). These crashes are structurally meaningful and reproducible, unlike crashes found by byte-level mutation which are often non-minimal and hard to reduce.

An important practical point: LLM-guided fuzzing does not eliminate the need for coverage-guided feedback. The LLM provides semantically valid mutations; coverage-guidance (via SanitizerCoverage or LLVM's built-in coverage instrumentation) directs those mutations toward unexplored execution paths. Combining the two — LLM as the mutation oracle, coverage bitmap as the reward signal — gives a fuzzer that is both structurally aware and converges toward unexplored code. The CovRL-Fuzz RL policy learns which LLM-generated mutation templates tend to produce coverage gains on the current codebase, accumulating a mutation strategy that is tuned to the specific IR patterns prevalent in the target dialect.

### 180.5.3 LLVM IR as Training Data

The `llvm-ir` dataset on HuggingFace contains millions of LLVM IR files harvested from open-source projects compiled with `clang -emit-llvm`. Code LLMs trained on this dataset learn instruction sequence patterns, calling conventions, and the structure of optimised versus unoptimised IR. The dataset's breadth — spanning C, C++, Rust, Swift, and Fortran frontends — means the trained model sees the variety of IR idioms each language frontend emits, including Clang's specific patterns for vtable dispatch, Rust's zero-cost abstractions lowered to IR, and Flang's array-section notation. Applications that have demonstrated feasibility in research include:

- **IR-level code completion**: given the beginning of a function in LLVM IR, a model predicts likely continuations — useful for filling in boilerplate sequences like GEP chains or phi node arrangements.
- **Pass selection prediction**: given a function's IR features, predict which optimisation passes would yield the largest size or performance improvement. This is the core problem of the `CompilerGym` environment.
- **Optimisation hint generation**: annotate IR with `!prof` metadata or `llvm.loop` hints predicted to be beneficial based on loop structure features extracted from the IR.

The `CompilerGym` environment (Cummins et al., 2021) wraps the LLVM pass pipeline as an OpenAI Gym environment. An RL agent observes a feature representation of the current IR state, selects an optimisation pass to apply from an action space of roughly 120+ transformations, and receives a reward proportional to the code size reduction. This framing lets researchers apply standard deep RL algorithms (DQN, PPO, A3C) to the pass ordering problem without modifying LLVM itself:

```python
import compiler_gym
import gym

env = gym.make("llvm-ic-v0")
env.reset(benchmark="cbench-v1/dijkstra")

for _ in range(50):
    action = env.action_space.sample()
    observation, reward, done, info = env.step(action)
    print(f"Action: {env.action_space.flags[action]:30s}  "
          f"IR size: {observation['IrInstructionCount']:6d}  "
          f"Reward: {reward:+.4f}")
    if done:
        break

env.close()
```

The `LlvmEnv` class within CompilerGym handles lowering the benchmark to LLVM IR, managing the pass manager, and computing observations. The observation space can be configured to return raw IR instruction counts, Programl graph features, or autophase hand-crafted features.

The `llvm-ic-v0` environment targets **instruction count** (IC) reduction as the reward: the difference in IR instruction count before and after the action, normalised by the initial count. This is a proxy for code size and compilability; it is not the same as binary performance. The `llvm-autophase-ic-v0` variant uses the Autophase 56-dimensional hand-crafted feature vector as the observation, which trains faster than raw IR features because the feature vector is pre-engineered to capture the most decision-relevant IR properties.

A complete RL training loop over CompilerGym uses standard `stable-baselines3` or `rllib` agents:

```python
from stable_baselines3 import PPO
import compiler_gym

def make_env():
    env = compiler_gym.make("llvm-autophase-ic-v0",
                             reward_space="IrInstructionCountOz")
    env.reset(benchmark="cbench-v1/dijkstra")
    return env

model = PPO("MlpPolicy", make_env(), verbose=1,
            n_steps=2048, batch_size=256, n_epochs=10)
model.learn(total_timesteps=500_000)
model.save("ppo_llvm_pass_ordering")
```

The trained policy maps Autophase feature vectors to pass selections. At inference time, the policy runs greedily until the reward stops improving, producing a per-function pass sequence that can be applied via `opt` with a custom pass pipeline string.

---

## 180.6 Neural Superoptimization and Learned Peephole Rules

### 180.6.1 Classical Superoptimization

Superoptimization (Massalin 1987) searches exhaustively for the shortest instruction sequence equivalent to a given sequence, using formal equivalence checking to verify candidates. The search space is exponential; practical superoptimizers bound the sequence length and restrict the instruction set. Bansal and Aiken (ASPLOS 2006) introduced a stochastic superoptimizer that samples random programs and checks equivalence via SMT solving — widening the search to include programs with loops and memory operations.

LLVM's `InstCombine` pass implements a large library of hand-written peephole rewrites: algebraic simplifications, strength reductions, and canonicalisations. Adding a new rewrite requires identifying the pattern, proving it correct, and expressing it in LLVM's `PatternMatch` API. Each step is labour-intensive, and the space of beneficial patterns grows with every new ISA extension.

The pattern space for 3-instruction sequences over a vocabulary of 30 common IR opcodes with two operand types is already in the tens of millions. Enumerating and verifying this space by hand is infeasible; Alive2 can verify a specific candidate in milliseconds, but generating candidates requires either exhaustive search or a smarter proposal mechanism.

### 180.6.2 Neural Superoptimization

Phothilimthana et al. (ICLR 2020) train a sequence-to-sequence model on pairs of (slow instruction sequence, fast equivalent sequence). The training corpus is generated by running a classical superoptimizer over millions of short code snippets and filtering pairs where the fast sequence is strictly shorter or has fewer dependent operations. At inference time, the model proposes a candidate replacement for a given input sequence; an SMT solver (Z3, or Alive2 for LLVM IR semantics) verifies that the proposal is semantically equivalent before it is applied.

The verification step is load-bearing: without it, the neural model's proposals cannot be trusted. Alive2 (Lopes et al., 2021), LLVM's IR-level equivalence checker, provides exactly the needed backend. Given two LLVM IR snippets, Alive2 encodes both as first-order formulas and queries Z3 for a counter-example. If none exists within the solver's timeout, the transformation is declared correct. The integration of Alive2 into LLVM's CI means that proposed peephole rules can be verified automatically without human proof review.

Alive2 can be invoked from the command line against any pair of IR functions:

```bash
/usr/lib/llvm-22/bin/alive2 \
    --smt-to=5000 \
    before.ll after.ll
```

A "transformation is correct" result means no counter-example was found within the 5-second Z3 timeout. For peephole proposals that are semantically wrong — even subtly so, due to integer overflow or poison semantics — Alive2 produces a concrete counter-example: specific bit-width inputs for which the before and after functions return different results. This counter-example mechanism is what makes the neural superoptimizer pipeline trustworthy: proposal generation has no correctness constraint, but every proposal passes through an oracle that either certifies it or refutes it with evidence.

### 180.6.3 Feeding Learned Rules Back to InstCombine

The practical pipeline for neural peephole rule discovery is:

1. Extract `(before, after)` pairs from LLVM's existing `InstCombine` patterns as a bootstrap corpus.
2. Train a seq2seq model (transformer or LSTM) to generalise from the corpus.
3. For each new LLVM IR function in a target codebase, apply the model to each eligible window of 3–8 instructions and collect proposals.
4. Run Alive2 on each proposal. Discard unverified ones.
5. Cluster verified proposals by abstract pattern (using symbolic abstraction over variable names).
6. For clusters with more than a threshold frequency, generate a `PatternMatch`-based C++ rule and submit it for inclusion in `InstCombine`.

Step 6 has not been automated end-to-end for mainline LLVM as of version 22. Alive2's automated patch suggestion feature (triggered via `llvm/utils/alive2/`) can generate the C++ `PatternMatch` template given a verified pair, but the code review and upstreaming step remains manual. The research prototype demonstrates that the model discovers non-trivial patterns: multi-instruction sequences involving `icmp`, `select`, and bitwise operations that `InstCombine`'s authors had not catalogued.

A verified pair from the research corpus — converting a two-instruction sequence into a semantically equivalent one-instruction sequence — might look like:

```llvm
; Before: two instructions
%t = or i32 %x, %y
%r = and i32 %t, %z

; After: equivalent (when x & z == 0 is provable from surrounding context)
%r = and i32 %y, %z
```

The `PatternMatch` rule generated for this would be:
```cpp
// InstCombine: (x | y) & z → y & z when (x & z) is always zero
if (match(I, m_And(m_Or(m_Value(X), m_Value(Y)), m_Value(Z))) &&
    MaskedValueIsZero(X, cast<IntegerType>(I->getType())->getMask()
                        & *CI->getValue(), DL))
  return BinaryOperator::CreateAnd(Y, Z);
```

The structural parallel between the IR pattern and the `PatternMatch` API call is tight enough that it can be generated mechanically from a verified (before, after) pair — which is what Alive2's patch suggestion mode does.

The longer-term vision is a continuous pipeline: as LLVM's CI compiles the project test suite, the neural superoptimizer runs asynchronously on hot functions, collects verified proposals, clusters them by abstract pattern, and files patches against `InstCombine.cpp` for human review. The human's role becomes reviewing verified proofs rather than discovering patterns.

---

## 180.7 The Future: Learned Optimization Policies

### 180.7.1 End-to-End Pass Ordering via RL

The LLVM pass pipeline at `-O3` is a fixed sequence of roughly 60–80 passes chosen by the compiler engineers to work well on average. For any specific program, a different ordering might do significantly better. The `CompilerGym` experiments described in §180.5.3 demonstrate that RL agents can learn pass sequences that outperform `-O3` on individual benchmarks from `cBench` and `MiBench`.

POSET-RL (Kulkarni et al.) extends this to hierarchical pass ordering: a high-level policy selects which optimisation phase to enter (scalar, loop, interprocedural), and a low-level policy selects individual passes within that phase. Hierarchical decomposition reduces the effective action space and improves sample efficiency. On selected benchmarks, POSET-RL produces binaries 5–10% smaller than `-O3`, though with high variance across the benchmark suite.

The fundamental limits of end-to-end learned policies are:

- **Generalisation**: policies trained on one codebase (say, cBench) perform poorly on out-of-distribution code (say, a cryptographic library). The IR feature space is enormous, and training coverage is thin.
- **Training cost**: collecting enough (IR, pass sequence, binary quality) tuples to train a reliable policy requires millions of compilation runs. At a minute per compilation, this is years of CPU time without significant parallelism.
- **Correctness**: an RL agent that applies a sequence of passes with no correctness constraint can trigger compiler bugs, producing miscompiled output. Every applied pass must be semantically sound, which is guaranteed when drawing from LLVM's pass registry but not guaranteed when the agent learns to chain passes in orders the pass developers never tested.

### 180.7.2 The Realistic Near-Term: ML Over Validated Configurations

The practically deployable version of learned pass ordering sidesteps the generalisation problem by restricting the agent's action space to a small validated set of pass orderings. The ML component does not freely compose passes; instead, it selects among a handful of pre-validated pipeline variants (e.g., "standard -O3", "-O3 + aggressive loop unrolling", "-O3 + size-optimised loop handling") based on per-function IR features. This is a classification problem, not a combinatorial search, and it generalises well.

MLGO's inline advisor and eviction advisor follow exactly this pattern: they replace a single binary decision with a learned policy, not an unconstrained sequence of transformations. The lesson for future ML integration in compilers is to identify individual decision points — places where the compiler commits to one of a small finite set of choices — and learn policies for those points rather than attempting to learn the entire optimisation strategy end-to-end.

Formal verification keeps this safe. Each choice in the validated set is proven correct by construction (it is a specific invocation of LLVM's existing, tested pass pipeline). The ML layer only selects among safe options; it cannot introduce miscompilation. As Alive2 and other IR-level verification tools mature, the set of validated choices can grow: a superoptimizer-generated peephole rule that Alive2 has verified can be added to the action space without a human proof review.

### 180.7.3 Convergence of ML and Formal Methods

The trajectory of AI-guided compilation is toward a system where ML proposes and formal methods verify. Neural superoptimizers propose peephole rewrites; Alive2 verifies them. RL agents select pass orderings; equivalence checkers confirm that the selected sequence preserves program semantics. LLMs generate TableGen records; `llvm-tblgen` rejects syntactic errors immediately; a semantic validator (future work) confirms that the instruction encoding matches the hardware manual.

The verification layer has a natural cost structure. SMT solving scales polynomially for short sequences (3–8 instructions) and hits exponential blow-up for longer ones. This is not a coincidence: peephole optimisation is exactly the domain where formal verification scales, and the neural superoptimizer's proposal generator is calibrated to produce short-sequence replacements. For longer-range optimisations — loop transformations, inlining, interprocedural specialisation — semantic equivalence checking is undecidable in general, and the verification falls back to testing (running both the original and transformed program on representative inputs) rather than proving.

The realistic picture for a 2030-era production compiler is a tiered verification architecture:

| Transformation class | Verification method | Cost per check |
|---|---|---|
| Peephole (≤8 instr) | Alive2 / SMT | milliseconds |
| Local loop transformations | Polyhedral dependence analysis | microseconds |
| Pass pipeline variant | Pre-validated set membership | zero (offline) |
| Inlining decision | RL policy over learned features | microseconds |
| Novel loop schedule | Testing on representative inputs | seconds |

The ML layer operates at all levels: proposing peepholes (neural superoptimizer), selecting pipeline variants (MLGO-style advisors), and eventually scheduling novel loop transformations (meta-schedule). The verification layer certifies each class at its natural cost point. The key insight is that correctness does not require verification at compilation time for all classes — pre-validation at training time, combined with restricting the deployment-time action space to the pre-validated set, achieves the same guarantee without adding per-compilation latency.

This division of labour exploits the complementary strengths of each approach: ML scales to enormous pattern spaces that exhaustive search cannot reach, while formal methods provide the correctness guarantees that a deployed compiler must have. Compilers that fully integrate both layers will be able to learn from every compiled program, continuously improving their optimisation policies, while providing the same semantic guarantees that today's manually written compilers provide.

---

## Chapter Summary

- The `MLModelRunner` abstraction unifies MLGO's ML components under a two-mode API: `ReleaseModeModelRunner` executes an AOT-compiled model as a direct C++ call; `DevelopmentModeModelRunner` opens a gRPC channel to a Python training server. The mode is selected at CMake time via `LLVM_HAVE_TF_AOT` / `LLVM_HAVE_TF_API` and at invocation time via `-enable-ml-inliner=release|development`.
- The `MLInlineAdvisor` (in `llvm/lib/Analysis/MLInlineAdvisor.cpp`) and the ML eviction advisor (in `llvm/lib/CodeGen/MLRegallocEvictAdvisor.cpp`) share this infrastructure; both are trained with RL over large corpora and deployed as AOT-compiled models with no added inference overhead relative to the heuristics they replace.
- ML cost models for scheduling (Ithemal, uops.info-trained) predict block throughput from instruction sequences more accurately than static latency tables, but no integration is yet upstream in LLVM 22.
- TVM's `meta_schedule` uses an XGBoost cost model trained on measured hardware data to guide tile-size search; IREE's tuning spec externalises the result as `transform.named_sequence` MLIR ops that override default tile sizes at compile time.
- Triton's `@triton.autotune` decorator exhaustively benchmarks a user-specified configuration space per input shape; the `prune_configs_by` hook prunes invalid configs before benchmarking; results are cached under `~/.triton/cache/` (override with `TRITON_CACHE_DIR`).
- ML-based profile inference predicts per-branch probabilities from static IR structure using GNN models, feeding the same `BranchProbabilityInfo` / `BlockFrequencyInfo` consumer infrastructure that measured PGO profiles populate; no backend is yet mainline in LLVM 22.
- LLM-assisted TableGen authoring accelerates target backend development; `llvm-tblgen` provides immediate syntactic feedback; LLM-guided fuzzing (EvoFuzz, CovRL-Fuzz) discovers dialect-specific crashes that byte-level mutation misses.
- Neural superoptimization pairs a seq2seq model proposing fast instruction sequences with Alive2 SMT verification, enabling automated discovery of `InstCombine`-style peephole rules; the research pipeline is demonstrated but not yet automated end-to-end for mainline LLVM.
- `CompilerGym` wraps the LLVM pass pipeline as an OpenAI Gym environment, allowing RL agents to learn pass orderings that outperform `-O3` on individual benchmarks; POSET-RL's hierarchical decomposition improves sample efficiency.
- The near-term practical deployment model for learned policies restricts ML to selecting among a small set of pre-validated pipeline variants rather than freely composing passes; this eliminates miscompilation risk while capturing the majority of the ML benefit.
- The long-term trajectory converges ML proposal with formal verification: neural models generate candidates, equivalence checkers (Alive2, SMT solvers) certify correctness, and only verified transformations enter the compiler's action space.
