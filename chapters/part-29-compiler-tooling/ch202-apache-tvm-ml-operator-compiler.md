# Ch202 — Apache TVM: An ML Operator Compiler

*Part XXIX — Compiler Tooling, Kernel Integration, and Binary Analysis*

Apache TVM occupies a singular position in the ML compiler landscape: it is the only production-grade system that spans from cloud GPUs to bare-metal microcontrollers, does so without relying on vendor libraries for every hot kernel, and provides a rigorous autotuning story at the operator level. Where XLA ([Ch153 — XLA Architecture](../part-22-xla-openxla/ch153-xla-architecture.md)) optimizes at the whole-program HLO level with strong ties to JAX and TensorFlow, and IREE ([Ch162 — IREE: A Deployment Compiler](../part-23-mlir-production/ch162-iree-a-deployment-compiler.md)) brings MLIR-native VM dispatch to edge hardware, TVM specializes in *schedule-driven kernel generation* — expressing how a mathematical tensor computation should be mapped onto hardware loops, caches, SIMD lanes, and tensor cores. This chapter walks through TVM's three-level IR hierarchy (Relay, TIR, Tensor Expressions), the schedule primitives that are TVM's core contribution, the two generations of automated search (AutoTVM and Meta-Schedule), the BYOC escape hatch for external accelerators, the Relax unified IR that supersedes the Relay/TIR divide, and the microTVM path to bare-metal deployment. Readers who have completed Parts XXII–XXIII will already understand why domain-specific lowering pipelines exist; this chapter shows how TVM's design choices differ from — and in many places complement — the MLIR ecosystem.

---

## 1. TVM's Place in the ML Compiler Ecosystem

The ML compiler space converged on three architectural philosophies between 2018 and 2024, each optimizing for a different primary tension.

**XLA** ([Ch153](../part-22-xla-openxla/ch153-xla-architecture.md)) centralizes optimization at the HLO graph level. Its strengths are whole-program buffer assignment, aggressive operation fusion controlled by the compiler, and first-class integration with cuBLAS/cuDNN for the performance-critical kernels that hardware vendors have already hand-tuned. The cost is target narrowness: XLA's GPU backend emits PTX/SASS and is deeply coupled to NVIDIA and Google TPU hardware. Porting to a new target requires implementing a substantial HLO backend.

**IREE** ([Ch162](../part-23-mlir-production/ch162-iree-a-deployment-compiler.md)) is an MLIR-native compiler with a hardware abstraction layer (HAL) that abstracts dispatch over Vulkan, Metal, CUDA, and CPU. Its VM model provides portability; its dispatch region model manages kernel granularity at compile time. IREE is deployment-first: it minimizes runtime footprint and binary size. Its tuning story historically depended on MLIR's own transformation infrastructure rather than a dedicated search framework.

**Triton** ([Ch163 — Triton: A Compiler for GPU Kernels](../part-23-mlir-production/ch163-triton-gpu-kernels.md)) operates at the tile abstraction level: the programmer specifies blocked matrix operations and memory loads in Python, and the compiler emits optimized PTX. It excels on attention and matmul kernels but targets a narrower class of computations and a narrower set of hardware (primarily NVIDIA and AMD GPUs).

**TVM** takes the orthogonal approach of *operator-level schedule synthesis*. Its user-visible APIs let compiler engineers express — or let the autotuner discover — exactly how loop iterations should be split, reordered, parallelized, and mapped to hardware. This makes TVM uniquely effective for targets that lack vendor library coverage: RISC-V DSPs, Arm Cortex-M cores, custom NPUs. The TVM runtime imposes minimal overhead and can operate without an OS, heap allocator, or dynamic dispatch — enabling microcontroller deployment that is simply out of scope for XLA or IREE.

The comparative difference in design philosophy is best captured in a table:

| Property | XLA | IREE | TVM | Triton |
|---|---|---|---|---|
| IR abstraction level | HLO (op-graph) | Linalg/Flow/HAL (MLIR) | Relay → TIR (loop) | Triton (tile) |
| Primary target | GPU/TPU | GPU/CPU/FPGA | GPU/CPU/MCU | GPU |
| Autotuning | Limited (GEMM configs) | IREE-TurbineFlow | AutoTVM/MetaSchedule | None (manual tiles) |
| Bare-metal MCU | No | Limited | Yes (microTVM) | No |
| MLIR-native | Partial (StableHLO) | Yes | Partial (Relax uses MLIR) | Yes |
| Framework integration | JAX, TF first-class | ONNX/IREE runtime | TFLite, ONNX, PyTorch | PyTorch (torch.compile) |
| Vendor library offload | Heavy (cuBLAS/cuDNN) | Some | BYOC/optional | cuBLAS for fallback |

TVM was originally published at OSDI 2018 (*TVM: An Automated End-to-End Optimizing Compiler for Deep Learning*, Chen et al.) and has since become an Apache top-level project. Its architecture has evolved substantially since that paper: the original TE/Schedule API has been joined by TVMScript/TIR, then AutoTVM, then Ansor, then Meta-Schedule, and finally Relax — each iteration broadening target coverage or reducing the manual effort required of compiler engineers.

TVM's source repository is hosted at [`https://github.com/apache/tvm`](https://github.com/apache/tvm). The primary reference for this chapter is the TVM Unity codebase as of the `v0.16` release line; interface spellings may vary slightly across older versions.

---

## 2. The TVM IR Hierarchy

TVM organizes compilation through three progressively lower levels of abstraction. Each level is responsible for a distinct class of transformation.

### 2.1 Relay — Typed Functional Graph IR

Relay is TVM's graph-level IR. It is a typed, first-order functional language: every `relay.Function` has a return type annotation, and every tensor carries shape and dtype information in its type. The type system supports tensor types `TensorType(shape, dtype)`, tuple types `TupleType([t1, t2, ...])`, and function types. Shape polymorphism is supported via symbolic shape variables.

A simple convolutional network fragment in Relay:

```python
import tvm
import tvm.relay as relay

# Define symbolic input shapes
x = relay.var("x", shape=(1, 3, 224, 224), dtype="float32")
w = relay.var("w", shape=(64, 3, 3, 3), dtype="float32")
b = relay.var("b", shape=(64,), dtype="float32")

# Graph construction
y = relay.nn.conv2d(x, w, kernel_size=(3, 3), padding=(1, 1), channels=64)
y = relay.nn.bias_add(y, b)
y = relay.nn.relu(y)

# Wrap in a module
mod = tvm.IRModule({"main": relay.Function([x, w, b], y)})
print(mod)
```

The `relay.nn` namespace mirrors the ONNX/TFLite operator set. Relay also carries `relay.Let`, `relay.If`, `relay.Tuple`, and `relay.TupleGetItem` for control flow and data-flow routing.

**Relay type inference** is performed by the type checker in [`src/relay/analysis/type_infer.cc`](https://github.com/apache/tvm/blob/main/src/relay/analysis/type_infer.cc). The type of every expression is inferred from the tensor shapes declared on `relay.var` nodes and propagated through operator type relations. Operators register their type relation as a C++ function in the operator registry — for example, `nn.conv2d` registers `Conv2DRel` which computes the output shape from input shapes, kernel size, stride, and padding. This means Relay has full shape knowledge at every node, enabling memory allocation planning and fusion legality checks.

**Quantization in Relay** is expressed through `relay.qnn` operators: `relay.qnn.conv2d`, `relay.qnn.dense`, `relay.qnn.quantize`, `relay.qnn.dequantize`. Quantization-aware passes convert FP32 graphs to QNN, then `relay.transform.Legalize()` lowers QNN operators to integer arithmetic on targets that support it. On Cortex-M with CMSIS-NN, a further BYOC partition routes QNN ops to CMSIS-NN fixed-function kernels.

**Whole-graph optimization passes** run on the Relay `IRModule` through the pass manager:

```python
from tvm.relay import transform

seq = tvm.transform.Sequential([
    transform.FoldConstant(),             # constant propagation + folding
    transform.EliminateCommonSubexpr(),   # CSE
    transform.FuseOps(fuse_opt_level=2),  # operator fusion
    transform.SimplifyInference(),        # fold BatchNorm into Conv weight/bias
    transform.ToMixedPrecision("float16"), # FP32 → FP16 graph rewrite
])
mod = seq(mod)
```

`FuseOps` uses a dataflow analysis to find chains of elementwise/broadcast operations following a heavy operator (conv, matmul) and collapses them into a single fused kernel. The fusion optimizer is configurable: `fuse_opt_level=2` allows producer→consumer fusion across a depth of two. `SimplifyInference` is particularly valuable for deployment: it pre-computes the scale/bias from BatchNorm running statistics and folds them into the preceding convolution weights, eliminating the BatchNorm op entirely.

**Frontend importers** translate external model formats into Relay. The Python-level frontends in [`python/tvm/relay/frontend/`](https://github.com/apache/tvm/tree/main/python/tvm/relay/frontend) cover:

| Frontend | Source format | Entry point |
|---|---|---|
| `from_onnx` | ONNX protobuf | `relay.frontend.from_onnx(model)` |
| `from_tflite` | TFLite flatbuffer | `relay.frontend.from_tflite(buf)` |
| `from_pytorch` | TorchScript | `relay.frontend.from_pytorch(scripted, input_infos)` |
| `from_tensorflow` | TF SavedModel / GraphDef | `relay.frontend.from_tensorflow(graph_def)` |
| `from_mxnet` | MXNet symbol + params | `relay.frontend.from_mxnet(symbol, params)` |

The Relay IR source lives in [`include/tvm/relay/`](https://github.com/apache/tvm/tree/main/include/tvm/relay) and [`src/relay/`](https://github.com/apache/tvm/tree/main/src/relay).

### 2.2 TIR / TVMScript — Loop-Level Imperative IR

Below Relay, each operator is represented as a `tir.PrimFunc` — a loop-level imperative function over `Buffer` objects. TVMScript provides a Python DSL that maps directly to TIR nodes, making the IR human-readable and editable.

A matrix multiplication in TVMScript:

```python
import tvm
from tvm.script import tir as T

@T.prim_func
def matmul(
    A: T.Buffer((1024, 1024), "float32"),
    B: T.Buffer((1024, 1024), "float32"),
    C: T.Buffer((1024, 1024), "float32"),
):
    for i, j, k in T.grid(1024, 1024, 1024):
        with T.block("C"):
            vi, vj, vk = T.axis.remap("SSR", [i, j, k])
            T.reads(A[vi, vk], B[vk, vj])
            T.writes(C[vi, vj])
            with T.init():
                C[vi, vj] = T.float32(0)
            C[vi, vj] = C[vi, vj] + A[vi, vk] * B[vk, vj]
```

The `T.block` abstraction is the central concept in TIR. Each block:
- Declares its *iteration variables* with axis kinds: `S` (spatial) or `R` (reduction).
- Declares explicit `T.reads` and `T.writes` regions, enabling dependency analysis without alias reasoning.
- Has an optional `T.init` body executed when the reduction initializer is needed (first iteration of a reduction).

The axis kind encoding `"SSR"` in `T.axis.remap` means: `vi` and `vj` are spatial (can be freely parallelized), `vk` is a reduction axis (requires serial accumulation or atomic operations).

TIR lowering passes transform `PrimFunc` nodes from high-level loop nests to hardware-specific instructions. The pass pipeline is applied through `tvm.lower`:

```python
# Inspect TIR after lowering (before codegen)
with tvm.transform.PassContext(opt_level=3):
    lowered = tvm.lower(s, [A, B, C], simple_mode=True)
print(lowered)
```

Key TIR passes include:

- `VectorizeLoop` — replaces vectorized loops with `tir.BufferLoad`/`Store` over `<N x dtype>` vector types.
- `UnrollLoop` — fully unrolls loops marked with `unroll_explicit` or `unroll_implicit`.
- `StorageFlatten` — converts multi-dimensional buffer accesses to flat linear indices, tracking strides.
- `InjectDoubleBuffer` — inserts asynchronous prefetch idioms for GPU targets with memory-compute overlap.
- `LowerThreadAllreduce` — lowers `tir.comm_reducer` reduction trees to warp-level shuffle instructions (`__shfl_xor_sync` on CUDA).
- `LowerWarpMemory` — maps `"warp"` storage scope allocations to CUDA `__shared__` warp-tiled accesses.
- `InjectPTXAsyncCopy` — on Hopper (sm_90+), emits `cp.async.bulk` (TMA) instructions for tensor memory access.

Source: [`src/tir/transforms/`](https://github.com/apache/tvm/tree/main/src/tir/transforms).

### 2.3 Tensor Expression (TE) — The Legacy Scheduling API

Before TVMScript, TVM used a compute/schedule API called Tensor Expression (TE). TE is still widely used in operator libraries and AutoTVM templates:

```python
from tvm import te

# Declare placeholders
A = te.placeholder((1024, 1024), name="A", dtype="float32")
B = te.placeholder((1024, 1024), name="B", dtype="float32")

# Declare computation symbolically
k = te.reduce_axis((0, 1024), name="k")
C = te.compute(
    (1024, 1024),
    lambda i, j: te.sum(A[i, k] * B[k, j], axis=k),
    name="C",
)

# Create schedule
s = te.create_schedule([C.op])
```

The `te.compute` call creates a `ComputeOp` node that records the index expression as a lambda over symbolic indices. `te.sum` introduces a reduction. `te.create_schedule` allocates a `Schedule` object that holds per-stage scheduling decisions.

TE lowers to TIR via `tvm.lower(s, [A, B, C])`. The resulting TIR `PrimFunc` is what the downstream codegen consumes.

The TE/Schedule API is considered stable but in maintenance mode. New operator implementations in the TVM standard library are expected to use TVMScript directly. The operator library in [`python/tvm/topi/`](https://github.com/apache/tvm/tree/main/python/tvm/topi) (TVM Operator Inventory) provides reference TE implementations and AutoTVM templates for common operators (conv2d in NCHW/NHWC/NCHWc layouts, batch matmul, pooling, depthwise conv, softmax) across all supported targets.

---

## 3. Schedule Primitives

Schedule primitives are the core of TVM's value proposition. They transform loop nests from naive reference implementations into hardware-optimized code without changing mathematical semantics. All primitives operate on `Schedule` objects (TE) or `Schedule` + `BlockRV`/`LoopRV` handles (TIR/Meta-Schedule).

### 3.1 Primitive Catalogue

**`split(axis, factor)`** — Tile an axis into an outer and inner loop:
```python
i, j = s[C].op.axis
i_outer, i_inner = s[C].split(i, factor=32)
j_outer, j_inner = s[C].split(j, factor=32)
```

**`fuse(ax1, ax2)`** — Merge two consecutive loops into one. Useful before `parallel` or `bind` to expose sufficient parallelism:
```python
fused = s[C].fuse(i_outer, j_outer)
```

**`reorder(ax1, ax2, ...)`** — Permute the loop order. Changing `i, j, k` to `i, k, j` can improve cache locality by making the inner loop stride-1 on B:
```python
s[C].reorder(i_outer, j_outer, k_outer, i_inner, j_inner, k_inner)
```

**`compute_at(stage, axis)`** — Move a producer stage's computation inside a consumer loop. This is the key primitive for fusion: it eliminates intermediate materialization:
```python
# Compute bias_add inside the tiling loop of relu
s[bias].compute_at(s[relu], i_inner)
```

**`vectorize(axis)`** — Emit SIMD instructions for the innermost loop. The axis length must be a static constant matching the target's vector width:
```python
s[C].vectorize(j_inner)  # emit 256-bit AVX2 for j_inner=8 floats
```

**`unroll(axis)`** — Generate unrolled loop body. Reduces branch overhead; exposes instruction-level parallelism to the backend:
```python
s[C].unroll(k_inner)
```

**`parallel(axis)`** — Emit OpenMP `#pragma omp parallel for` or thread-level parallelism:
```python
s[C].parallel(i_outer)
```

**`tensorize(axis, intrin)`** — Replace a sub-loop nest with a hardware intrinsic call. The `TensorIntrin` describes the compute semantics and codegen behavior. Used for AVX-512 VNNI, Arm Neon SDOT, and GPU `wmma`/`mma` tensor core instructions:
```python
from tvm.contrib import cblas
s[C].tensorize(j_inner, intrin_avx2_8x1)
```

**`cache_read(stage, scope, readers)`** / **`cache_write(stage, scope)`** — Insert explicit cache buffers in a named memory scope (`"local"`, `"shared"`, `"global"` on GPU; `"local.L1"`, `"local.L2"` on CPU):
```python
AA = s.cache_read(A, "shared", [C])
```

### 3.2 Complete AVX2 Matmul Schedule

The following schedule drives a 1024×1024 FP32 matmul to near-peak throughput on x86 with AVX2 (8-wide 256-bit FMA):

```python
from tvm import te
import tvm

A = te.placeholder((1024, 1024), name="A")
B = te.placeholder((1024, 1024), name="B")
k = te.reduce_axis((0, 1024), name="k")
C = te.compute(
    (1024, 1024),
    lambda i, j: te.sum(A[i, k] * B[k, j], axis=k),
    name="C",
)

s = te.create_schedule(C.op)
[i, j]  = s[C].op.axis
[k]     = s[C].op.reduce_axis

# Tile: 64x8 output tiles, 4-deep k strip
i_outer, i_inner = s[C].split(i, factor=64)
j_outer, j_inner = s[C].split(j, factor=8)
k_outer, k_inner = s[C].split(k, factor=4)

# Reorder to expose the vectorizable inner j loop
s[C].reorder(i_outer, j_outer, k_outer, k_inner, i_inner, j_inner)

# Parallelize outer i tiles across CPU cores
s[C].parallel(i_outer)

# Unroll k_inner for better instruction scheduling
s[C].unroll(k_inner)

# Vectorize inner j (8 float32 = 256-bit AVX2)
s[C].vectorize(j_inner)

target = tvm.target.Target("llvm -mcpu=core-avx2")
func   = tvm.build(s, [A, B, C], target=target)
```

The generated LLVM IR will contain `<8 x float>` vector types and fused multiply-add intrinsics when lowered through LLVM's x86 backend. On a Skylake core with 2× FMA units, 8 floats wide, 4 deep unroll, the inner loop approaches the theoretical peak of 32 FLOPs/cycle.

### 3.3 GPU Schedule: Shared Memory Tiling

On CUDA targets, schedule primitives map directly to GPU thread hierarchy concepts. The following schedule tile-maps a matmul onto CUDA thread blocks and warps:

```python
from tvm import te
import tvm

A = te.placeholder((1024, 1024), name="A")
B = te.placeholder((1024, 1024), name="B")
k = te.reduce_axis((0, 1024), name="k")
C = te.compute(
    (1024, 1024),
    lambda i, j: te.sum(A[i, k] * B[k, j], axis=k),
    name="C",
)

s = te.create_schedule(C.op)

# Cache operands in shared memory
AA = s.cache_read(A, "shared", [C])
BB = s.cache_read(B, "shared", [C])
CC = s.cache_write(C, "local")   # accumulate in register

# Tile for 128×128 thread blocks, 8×8 per thread
block_y, thread_y = 128, 8
block_x, thread_x = 128, 8

i, j    = s[C].op.axis
by, ty  = s[C].split(i, factor=block_y)
bx, tx  = s[C].split(j, factor=block_x)

# Bind outer tiles to CUDA block and thread indices
s[C].bind(by, te.thread_axis("blockIdx.y"))
s[C].bind(bx, te.thread_axis("blockIdx.x"))
s[C].bind(ty, te.thread_axis("threadIdx.y"))
s[C].bind(tx, te.thread_axis("threadIdx.x"))

# Cooperatively load tiles into shared memory
s[AA].compute_at(s[CC], k)
s[BB].compute_at(s[CC], k)

target = tvm.target.cuda(arch="sm_80")
func   = tvm.build(s, [A, B, C], target=target)
```

The `bind` primitive associates a loop variable with a CUDA thread axis. The TIR lowering pass `LowerThreadBind` translates these bindings to `threadIdx.x/y/z` and `blockIdx.x/y/z` accesses in the emitted PTX. The shared memory tiles `AA` and `BB` receive the `__shared__` storage class; cooperative loading across threads is implicit through the `compute_at` placement.

---

## 4. AutoTVM — Template-Guided Search

Writing schedules by hand is expert work. AutoTVM automates the search by parameterizing schedule templates and measuring candidate configurations on real hardware.

### 4.1 Writing a Tunable Template

A template is an ordinary Python function decorated with `@autotvm.template`. Inside, `cfg` is a `ConfigSpace` object used to declare tunable knobs:

```python
import tvm
from tvm import te, autotvm

@autotvm.template("conv2d_nchw")
def conv2d_nchw(N, C, H, W, K, R, S, stride, padding, dtype):
    # Compute graph (same as manual)
    data    = te.placeholder((N, C, H, W), dtype=dtype, name="data")
    weight  = te.placeholder((K, C, R, S), dtype=dtype, name="weight")
    rc      = te.reduce_axis((0, C), name="rc")
    ry      = te.reduce_axis((0, R), name="ry")
    rx      = te.reduce_axis((0, S), name="rx")
    OH = (H + 2*padding - R) // stride + 1
    OW = (W + 2*padding - S) // stride + 1
    conv    = te.compute(
        (N, K, OH, OW),
        lambda n, k, h, w: te.sum(
            data[n, rc, h*stride+ry, w*stride+rx] * weight[k, rc, ry, rx],
            axis=[rc, ry, rx],
        ),
        name="conv",
    )

    s   = te.create_schedule(conv.op)
    cfg = autotvm.get_config()

    # Declare tunable splits
    n, k, h, w = s[conv].op.axis
    cfg.define_split("tile_k", k, num_outputs=3)
    cfg.define_split("tile_w", w, num_outputs=2)
    cfg.define_knob("unroll_kw", [True, False])

    # Apply the configuration
    ko, km, ki = cfg["tile_k"].apply(s, conv, k)
    wo, wi     = cfg["tile_w"].apply(s, conv, w)
    if cfg["unroll_kw"].val:
        s[conv].unroll(ki)
    s[conv].vectorize(wi)
    s[conv].parallel(n)

    return s, [data, weight, conv]
```

`cfg.define_split("tile_k", k, num_outputs=3)` generates all factorizations of `K` into three factors as the search space. `cfg.define_knob` creates a categorical choice. The total search space is the Cartesian product of all defined knobs.

### 4.2 Running the Tuner

```python
from tvm import autotvm

# Create the task (N=1, C=64, H=56, W=56, K=128, R=3, S=3, stride=1, pad=1)
task   = autotvm.task.create(
    "conv2d_nchw",
    args=(1, 64, 56, 56, 128, 3, 3, 1, 1, "float32"),
    target=tvm.target.cuda(),
)

# XGBoost-based cost model tuner
tuner  = autotvm.tuner.XGBTuner(task, loss_type="rank")
log_file = "conv2d_tuning.log"

tuner.tune(
    n_trial=1000,
    measure_option=autotvm.measure_option(
        builder=autotvm.LocalBuilder(),
        runner=autotvm.LocalRunner(number=5, timeout=10),
    ),
    callbacks=[
        autotvm.callback.log_to_file(log_file),
        autotvm.callback.progress_bar(1000),
    ],
)
```

The XGBoost cost model is trained incrementally: each measured configuration provides a `(feature_vector, latency)` sample. Features encode loop structure (trip counts, axis kinds, memory access strides) and hardware utilization estimates. After ~200 trials the model typically predicts relative performance rankings accurately enough to guide the evolutionary search efficiently.

**Transfer learning**: When moving to similar hardware (e.g., a different GPU of the same microarchitecture), previously collected logs can warm-start the model:

```python
tuner.load_history("previous_gpu.log")  # warm-start from prior measurements
```

**Deploying the best config**:

```python
with autotvm.apply_history_best(log_file):
    with tvm.transform.PassContext(opt_level=3):
        lib = relay.build(mod, target=tvm.target.cuda(), params=params)
```

`apply_history_best` installs a global dispatch context that intercepts `autotvm.template` invocations and replaces the config with the best recorded result.

### 4.3 Supported Tuners

TVM ships several tuning algorithms in [`python/tvm/autotvm/tuner/`](https://github.com/apache/tvm/tree/main/python/tvm/autotvm/tuner):

| Tuner | Strategy | Best for |
|---|---|---|
| `XGBTuner` | XGBoost surrogate + evolutionary | Large search spaces (>1M configs) |
| `GATuner` | Genetic algorithm | Moderate spaces |
| `GridSearchTuner` | Exhaustive | Tiny spaces (<1000 configs) |
| `RandomTuner` | Random sampling | Baseline / sanity check |

### 4.4 RPC Measurement on Remote Devices

AutoTVM supports measuring on remote hardware through TVM's RPC infrastructure. This is essential when tuning for an Arm SoC or an Android phone from a development workstation:

```bash
# On the target device (e.g., an Arm phone or Raspberry Pi):
python3 -m tvm.exec.rpc_server --host 0.0.0.0 --port 9090

# On the development machine:
```

```python
from tvm import rpc

tracker = rpc.connect_tracker("0.0.0.0", 9090)
remote  = tracker.request("arm_v8a", session_timeout=60)

measure_option = autotvm.measure_option(
    builder=autotvm.LocalBuilder(build_func="default"),
    runner=autotvm.RPCRunner(
        key="arm_v8a",
        host="0.0.0.0",
        port=9090,
        number=5,
        repeat=3,
    ),
)
```

The `RPCRunner` cross-compiles the candidate schedule on the development machine, uploads the `.so` artifact to the target via TVM's RPC protocol, executes it, and returns measured latencies. The XGBoost model learns from these remote measurements exactly as it would from local ones.

---

## 5. Ansor and Meta-Schedule — Template-Free Search

AutoTVM requires domain experts to write schedule templates. The second generation of TVM's search infrastructure eliminates this requirement.

### 5.1 Ansor — Automatic Sketch Generation

Ansor (introduced in TVM 0.8, paper: *Ansor: Generating Diverse Programs for Tensor Computations*, OSDI 2020) decouples program structure from parameter values:

1. **Sketch generation**: Ansor applies a set of structural rules to a computation (e.g., "tile the spatial dimensions", "fuse point-wise ops into the consumer") and generates a set of *sketches* — schedule templates without specific tile sizes.
2. **Evolutionary search**: Random tile size samples are drawn for each sketch; the XGBoost cost model ranks them; top candidates are mutated (change one tile factor) and re-ranked.
3. **Measurement**: The top-k candidates per round are compiled and actually measured; results update the cost model.

```python
import tvm
from tvm import relay
from tvm.auto_scheduler import RelayIntegrationContext

target = tvm.target.cuda(arch="sm_90")
log_file = "ansor_resnet50.json"

# Extract tuning tasks from a Relay module
tasks, task_weights = tvm.auto_scheduler.extract_tasks(mod, params, target)

tuner = tvm.auto_scheduler.TaskScheduler(tasks, task_weights)
tune_option = tvm.auto_scheduler.TuningOptions(
    num_measure_trials=20000,
    runner=tvm.auto_scheduler.LocalRunner(repeat=3, timeout=20),
    measure_callbacks=[tvm.auto_scheduler.RecordToFile(log_file)],
)
tuner.tune(tune_option)
```

The `TaskScheduler` allocates the measurement budget across tasks proportionally to their weight (execution time share in the full model). This joint scheduling avoids spending 90% of the budget on a tiny operator.

### 5.2 Meta-Schedule — Modular Search Infrastructure

Meta-Schedule (`tvm.meta_schedule`, introduced in TVM 0.9) is a ground-up redesign that makes every component of the search pipeline pluggable:

- **Search space rules**: Python classes implementing `generate_design_space` over TIR `Schedule` objects.
- **Postprocessors**: Validate that a generated schedule is legal (e.g., shared memory does not exceed device limits).
- **Cost models**: `XGBModel`, `RandomModel`, or a custom neural predictor.
- **Runners**: Local, RPC, or custom measure backends.
- **Database**: `JSONDatabase` for persisting tuning records.

Tuning a single TIR function:

```python
from tvm import meta_schedule as ms

# Tune a TVMScript PrimFunc directly
database = ms.tune_tir(
    fn=matmul,          # T.prim_func defined in Section 2.2
    target=tvm.target.Target("llvm -num-cores=8"),
    max_trials_global=512,
    num_trials_per_iter=64,
    work_dir="./tune_tmp",
)

# Apply best schedule and build
sch = ms.tir_integration.compile_tir(database, matmul, target)
```

Tuning a full Relay model via Meta-Schedule:

```python
with ms.Profiler() as profiler:
    lib = ms.relay_integration.compile_relay(
        database=database,
        mod=mod,
        target=target,
        params=params,
    )

print(profiler.table())
```

Meta-Schedule integrates with BYOC (Section 6) through `PartitionedModule` — the search ignores BYOC-partitioned subgraphs and focuses on the TVM-compiled remainder.

### 5.3 Custom Search Space Rules in Meta-Schedule

The pluggable search space architecture lets compiler engineers inject domain knowledge without writing full schedule templates. A custom rule is a Python class:

```python
from tvm.meta_schedule import SearchSpaceRule
from tvm import tir

class AlwaysTileBy16(SearchSpaceRule):
    """Force all spatial axes to be tiled with factor 16."""
    def generate_design_space(self, func: tir.PrimFunc):
        sch = tir.Schedule(func)
        for block in sch.get_child_blocks(sch.get_root_block()):
            loops = sch.get_loops(block)
            for loop in loops:
                outer, inner = sch.split(loop, factors=[None, 16])
                sch.vectorize(inner)
        return [sch]

database = ms.tune_tir(
    fn=matmul,
    target=tvm.target.Target("llvm -num-cores=8"),
    max_trials_global=256,
    space=ms.space_generator.PostOrderApply(
        sch_rules=[AlwaysTileBy16()],
        postprocs=[ms.postproc.DisallowDynamicLoop()],
    ),
    work_dir="./tune_custom",
)
```

This extensibility is the key architectural difference from Ansor: where Ansor has a fixed sketch generation algorithm, Meta-Schedule allows per-target customization at every stage of the search pipeline.

---

## 6. BYOC — Bring Your Own Codegen

Many deployments include specialized accelerators — NPUs, DSPs, FPGAs — for which TVM cannot generate code natively. BYOC is TVM's mechanism to offload subgraphs to external codegens while keeping the TVM runtime as the orchestration layer.

### 6.1 Pattern Matching in Relay

BYOC's first step is identifying which subgraph to offload. The `relay.dataflow_pattern` API provides a combinator language for matching Relay expression trees:

```python
from tvm.relay.dataflow_pattern import (
    is_op, wildcard, is_constant, is_tuple_get_item
)

# Match conv2d → bias_add → relu
conv_pattern = (
    is_op("nn.conv2d")(wildcard(), is_constant())
    .optional(lambda x: is_op("nn.bias_add")(x, is_constant()))
    .optional(is_op("nn.relu"))
)
```

`is_op` checks the operator name; `wildcard()` matches any expression; `is_constant()` matches literal weights. The `.optional()` combinator allows the bias and relu to be absent.

### 6.2 Registering an External Codegen

```python
from tvm.relay.backend import compile_engine

@tvm.register_func("relay.ext.my_npu")
def my_npu_compiler(subgraph: tvm.IRModule) -> tvm.runtime.Module:
    """
    Receives a partitioned Relay subgraph.
    Returns a TVM runtime Module wrapping the NPU binary.
    """
    # Serialize the subgraph to NPU assembly (hypothetical)
    npu_asm = my_npu_sdk.compile(subgraph)
    # Package as a TVM C source module
    return tvm.runtime.load_static_library(npu_asm, ["my_npu_kernel"])

# Annotate and partition
from tvm.relay.op.contrib.register import register_pattern_table

@register_pattern_table("my_npu")
def pattern_table():
    return [("my_npu.conv_relu", conv_pattern)]

# Apply annotation pass
mod = relay.transform.MergeComposite(pattern_table())(mod)
mod = relay.transform.AnnotateTarget("my_npu")(mod)
mod = relay.transform.PartitionGraph()(mod)

# Build — TVM will call relay.ext.my_npu for annotated subgraphs
lib = relay.build(mod, target=tvm.target.cuda(), params=params)
```

The compilation flow is: `relay.build` runs fusion passes → the `PartitionGraph` pass wraps annotated subregions in `relay.Function` nodes tagged with `Compiler = "my_npu"` → the TVM build process calls the registered `relay.ext.my_npu` function for each partition → the returned `runtime.Module` is linked alongside the TVM-generated CUDA module → the runtime dispatches to the NPU binary when the annotated function is invoked.

### 6.3 Runtime Integration

BYOC modules are linked into the final `tvm.runtime.Module` and discovered through the module `GetFunction` interface. At runtime, the TVM executor invokes the external module's function handle with the same `PackedFunc` calling convention used for native TVM kernels — no special glue is needed.

### 6.4 Production BYOC Backends

TVM ships several production-quality BYOC backends in [`src/relay/backend/contrib/`](https://github.com/apache/tvm/tree/main/src/relay/backend/contrib):

| Backend | Target | Mechanism |
|---|---|---|
| `cutlass` | NVIDIA GPU (CUTLASS GEMM) | C++ template generation via CUTLASS emitter |
| `cublas` | NVIDIA GPU | cuBLAS GEMM/TRSM dispatch |
| `cudnn` | NVIDIA GPU | cuDNN conv/pool/norm dispatch |
| `arm_compute_lib` | Arm CPU/Mali GPU | Pattern match → ACL C++ calls |
| `ethos_n` | Arm Ethos-N NPU | Hardware-vendor SDK integration |
| `cmsisnn` | Arm Cortex-M | CMSIS-NN int8 kernel dispatch |
| `tensorrt` | NVIDIA GPU | TensorRT engine compilation and caching |
| `vitis_ai` | Xilinx DPU | Vitis AI compiler integration |

The CUTLASS BYOC backend is particularly instructive: rather than calling a pre-compiled library, it emits specialized CUTLASS C++ source code (with `cutlass::gemm::device::Gemm<>` template instantiations) for each matched GEMM subgraph, compiles that source with `nvcc`, and packages the resulting `.cubin` as a TVM `CUDAModule`. This achieves CUTLASS-level performance while still routing through the TVM runtime and enabling mixed TVM/CUTLASS execution within a single model.

---

## 7. Relax — The Unified IR (TVM Unity)

TVM's Relay/TIR split is an architectural seam: Relay knows about the whole graph but cannot express loop-level optimizations; TIR knows about loops but cannot express cross-operator data flow. *Relax* (`tvm.relax`), the centerpiece of the TVM Unity project (available since TVM 0.14), eliminates this split by providing a single IR that accommodates both graph-level and loop-level operations.

### 7.1 Relax IR Structure

A Relax function combines Relay-style graph structure with direct `PrimFunc` calls:

```python
import tvm
from tvm.script import relax as R, tir as T

@tvm.script.ir_module
class MyModule:
    @T.prim_func
    def matmul_kernel(
        A: T.Buffer((1024, 1024), "float32"),
        B: T.Buffer((1024, 1024), "float32"),
        C: T.Buffer((1024, 1024), "float32"),
    ):
        for i, j, k in T.grid(1024, 1024, 1024):
            with T.block("C"):
                vi, vj, vk = T.axis.remap("SSR", [i, j, k])
                with T.init():
                    C[vi, vj] = T.float32(0)
                C[vi, vj] += A[vi, vk] * B[vk, vj]

    @R.function
    def main(
        x: R.Tensor((1024, 1024), "float32"),
        y: R.Tensor((1024, 1024), "float32"),
    ) -> R.Tensor((1024, 1024), "float32"):
        # Relax call to a TIR PrimFunc — no IR boundary crossing
        z = R.call_tir(
            MyModule.matmul_kernel,
            (x, y),
            out_sinfo=R.TensorStructInfo((1024, 1024), "float32"),
        )
        return z
```

`R.call_tir` is the linchpin: it bridges the graph-level `relax.Function` to a loop-level `tir.PrimFunc` within the same `IRModule`. This eliminates the type-erasure overhead that occurred when Relay `ExternFunc` called into TIR.

### 7.2 Relax Transformations

The Relax pass infrastructure mirrors MLIR's pattern rewriting. Key passes:

- `relax.transform.FuseOps()` — graph-level operator fusion with shape awareness
- `relax.transform.LegalizeOps()` — lower `relax.nn.conv2d` → `call_tir(te_conv2d_prim_func)`
- `relax.transform.MetaScheduleApplyDatabase(database)` — inject tuned schedules from a `MetaSchedule` database
- `relax.transform.ToNVVM()` / `relax.transform.ToROCm()` — lower to GPU targets

### 7.3 Building and Running

```python
target  = tvm.target.Target("cuda -arch=sm_90")
ex      = relax.build(MyModule, target)
vm      = relax.VirtualMachine(ex, tvm.cuda(0))

x_nd    = tvm.nd.array(x_np, tvm.cuda(0))
y_nd    = tvm.nd.array(y_np, tvm.cuda(0))
result  = vm["main"](x_nd, y_nd)
```

The Relax VM is a register-based virtual machine that interprets the high-level graph structure while dispatching to compiled `PrimFunc` kernels for compute-intensive operations. Unlike the Relay graph executor, the Relax VM supports dynamic shapes through shape-function propagation.

### 7.4 Relax vs. Relay: Architectural Differences

The differences between Relay and Relax are not merely syntactic; they reflect a fundamental restructuring of the compiler's type system and lowering strategy:

| Aspect | Relay | Relax |
|---|---|---|
| IR boundary | Graph (Relay) and loops (TIR) are separate IRs | Single `IRModule` containing both `R.function` and `T.prim_func` |
| Shape handling | Static shapes only (or `Any` for dynamic) | First-class `StructInfo` with symbolic shape expressions |
| Operator lowering | `ExternFunc` crossing an IR boundary | `R.call_tir` inlined into the same function body |
| Pass infrastructure | `relay::transform::Pass` (AST visitor) | `relax::transform::Pass` (pattern rewriting via `ExprMutator`) |
| Quantization | `relay.qnn` operators then legalize | `relax.qnn` or direct int8 TIR lowering |
| Dynamic control flow | `relay.If`, Relay VM with ADTs | `relax.If`, shape-aware Relax VM |

The `StructInfo` type in Relax is more expressive than Relay's type: `R.TensorStructInfo((batch, seq_len, 512), "float16")` carries symbolic shape variables that are propagated through the function body and inferred by the `InferStructInfo` pass. This enables Relax to represent LLM inference with dynamic sequence lengths in a single compiled artifact.

The Relax source lives in [`include/tvm/relax/`](https://github.com/apache/tvm/tree/main/include/tvm/relax) and [`src/relax/`](https://github.com/apache/tvm/tree/main/src/relax).

---

## 8. Compilation Targets

TVM's target system abstracts hardware-specific details through the `tvm.target.Target` class. Targets carry a JSON-encoded attribute set that downstream passes query.

### 8.1 CUDA / GPU Targets

```python
# NVIDIA H100 (Hopper, sm_90)
target = tvm.target.cuda(arch="sm_90")

# NVIDIA A100 with explicit memory sizes
target = tvm.target.Target(
    "cuda -arch=sm_80 -max_shared_memory_per_block=166912"
)

# AMD RDNA3 via ROCm
target = tvm.target.Target("rocm -mcpu=gfx1100")
```

For CUDA targets, TVM generates PTX via its own codegen in [`src/target/llvm/`](https://github.com/apache/tvm/tree/main/src/target/llvm), which routes through LLVM's NVPTX backend. The `arch` attribute controls which PTX ISA version is emitted and which tensor core intrinsics are available.

Tensor core operations for sm_80 (A100) use `wmma` intrinsics:

```python
# tensorize with wmma fragment accumulation
from tvm.contrib import nvcc
wmma_intrin = tvm.tir.TensorIntrin.get("wmma_load_16x16x16_f16_a")
s[C_frag].tensorize(tx, wmma_intrin)
```

For sm_90 (H100), TVM emits `mma.sync.aligned.m16n8k16` PTX instructions through the `mma` tensor intrinsic family, achieving near-roofline throughput on FP16 and BF16 GEMMs.

### 8.2 Arm CPU Targets

```python
# Arm Cortex-A55 (mobile, in-order, Neon SIMD)
target = tvm.target.arm_cpu("cortex-a55")

# Arm Cortex-A78 with SVE (Scalable Vector Extension)
target = tvm.target.arm_cpu("cortex-a78", mattr="+sve")

# Apple M2 (Firestorm P-cores + ASIMD + AMX)
target = tvm.target.Target("llvm -mtriple=arm64-apple-macos -mcpu=apple-m2")
```

TVM's Arm backend uses LLVM's AArch64 target. Schedule primitives that call `vectorize` will emit Neon `float32x4_t` operations; `tensorize` with the `Neon SDOT` intrinsic emits `sdot` instructions for int8 inference.

### 8.3 microTVM AOT Target

```python
# Bare-metal C with AOT executor (no dynamic dispatch)
target = tvm.target.Target(
    "c --runtime=c --executor=aot --link-params=1 --unpacked-api=1"
)
```

The `--executor=aot` flag switches from the VM/graph-executor to the Ahead-of-Time (AOT) executor, which generates a single C function `tvmgen_default_run` with statically allocated workspace. `--link-params=1` embeds model weights into the generated binary. `--unpacked-api=1` uses a simpler C ABI without the `TVMValue`/`DLTensor` boxing that the standard `PackedFunc` convention requires.

### 8.4 OpenCL and Vulkan Targets

TVM also supports OpenCL and Vulkan compute for embedded GPUs and FPGAs:

```python
# Arm Mali GPU (OpenCL ES 3.1)
target = tvm.target.Target("opencl -device=mali -model=mali-g77")

# Vulkan compute (cross-platform)
target = tvm.target.Target("vulkan -from_device=0")
```

OpenCL targets use a string-based kernel codegen in [`src/target/source/codegen_opencl.cc`](https://github.com/apache/tvm/blob/main/src/target/source/codegen_opencl.cc). The schedule primitives `bind(te.thread_axis("blockIdx.x"))` translate to OpenCL `get_group_id(0)`, and `bind(te.thread_axis("threadIdx.x"))` translates to `get_local_id(0)`. Vulkan targets generate SPIR-V through LLVM's SPIR-V backend.

---

## 9. microTVM — Bare-Metal Deployment

microTVM extends TVM to deeply embedded hardware where no OS, heap allocator, or dynamic library loader is available. It is the only ML compiler in this book with this capability.

### 9.1 Constraints and Design Choices

Bare-metal deployment imposes hard constraints:
- **No heap**: all buffers are statically allocated at compile time.
- **No POSIX**: no `malloc`, no threads, no file I/O.
- **No dynamic dispatch**: function pointers must resolve at link time.
- **Code size**: flash budget on Cortex-M4 is often 256 KB–1 MB.

The AOT executor meets these constraints by generating a C source file for each model. The generated code is a DAG of calls to kernel functions, all using pre-allocated workspace arrays.

### 9.2 Compilation from the Command Line

```bash
# Compile a TFLite model for Cortex-M55 (Ethos-M55 NPU optional)
tvmc micro compile \
    --target "c --runtime=c --executor=aot --link-params=1" \
    --target-c-mcpu cortex-m55 \
    --output mobilenet_aot/ \
    mobilenet_v1_int8.tflite

# Flash to a Zephyr RTOS board
tvmc micro run \
    --platform zephyr \
    --board nucleo_f746zg \
    mobilenet_aot/ \
    --inputs input_data.npz
```

The `tvmc micro` toolchain wraps:
1. `relay.build` / `relax.build` with the AOT target.
2. `tvm.micro.generate_project` to produce a Zephyr or Arduino project tree.
3. Platform-specific flash/run commands via the `tvm.micro.transport` layer.

### 9.3 Python API for AOT Build

```python
import tvm
import tvm.micro as micro
from tvm import relay

# Load TFLite model
with open("mobilenet_v1_int8.tflite", "rb") as f:
    tflite_buf = f.read()

mod, params = relay.frontend.from_tflite(tflite_buf)

# AOT compilation target
target = tvm.target.Target(
    "c --runtime=c --executor=aot --link-params=1 --unpacked-api=1",
    host="c --runtime=c --executor=aot --link-params=1 --unpacked-api=1",
)

with tvm.transform.PassContext(opt_level=3, config={"tir.disable_vectorize": True}):
    lib = relay.build(mod, target=target, params=params)

# Generate Zephyr project
template_project = micro.get_microtvm_template_projects("zephyr")
project = micro.generate_project(
    template_project,
    lib,
    "zephyr_project/",
    {"zephyr_board": "nucleo_f746zg", "west_cmd": "west"},
)
project.build()
project.flash()
```

`tir.disable_vectorize=True` is required for bare-metal C targets because the C backend cannot emit SIMD intrinsics — vectorization must be suppressed or handled by the platform's CMSIS-NN integration.

### 9.4 CMSIS-NN Integration

For Cortex-M cores with the M-Profile Vector Extension (MVE, also marketed as Helium), TVM integrates ARM's CMSIS-NN library. Schedule `tensorize` calls replace inner loops with CMSIS-NN kernel functions:

```python
from tvm.relay.op.contrib import cmsisnn

# Partition conv2d/depthwise_conv2d/FC/softmax to CMSIS-NN
mod = cmsisnn.partition_for_cmsisnn(mod, params)
```

This is a BYOC usage where the "external codegen" is the CMSIS-NN C library rather than a custom accelerator.

### 9.5 The AOT Executor C Output

The AOT executor generates a C function with a fixed signature that the embedded application calls directly:

```c
/* Generated by TVM AOT executor */
/* Model: mobilenet_v1_int8, Target: c -runtime=c --executor=aot */

/* Statically allocated workspace (2KB scratch) */
static int8_t workspace[2048];

/* Statically allocated input/output buffers */
static int8_t input_data[1 * 3 * 224 * 224];
static int8_t output_data[1 * 1000];

/* Entry point — no malloc, no libc, no dynamic dispatch */
int32_t tvmgen_default___tvm_main__(
    void* input_data_ptr,
    void* output_data_ptr,
    void* workspace_ptr
) {
    tvmgen_default_fused_nn_conv2d_add_nn_relu(
        input_data_ptr, model_weights_conv0, workspace_ptr, workspace_ptr + 512);
    /* ... remaining ops ... */
    return 0;
}
```

The generated file includes only `<stdint.h>` and operator-specific kernel C files. The complete generated output for a ResNet-8 model on Cortex-M4 fits in under 60 KB of flash.

---

## 10. The TVM Runtime

After compilation, the TVM runtime provides a thin, portable execution layer.

### 10.1 Module and NDArray

```python
import tvm
import tvm.runtime
import numpy as np

# Load a pre-built module (from relay.build or relax.build)
lib = tvm.runtime.load_module("compiled_model.so")

# Create device handle
dev = tvm.cuda(0)      # CUDA device 0
# dev = tvm.cpu(0)     # CPU
# dev = tvm.opencl()   # OpenCL

# Wrap a numpy array as a TVM NDArray on the device
x_np   = np.random.randn(1, 3, 224, 224).astype("float32")
x_tvm  = tvm.nd.array(x_np, dev)

# Run via the graph executor
from tvm.contrib import graph_executor
gmod = graph_executor.GraphModule(lib["default"](dev))
gmod.set_input("data", x_tvm)
gmod.run()
output = gmod.get_output(0).numpy()
```

`tvm.nd.array` performs a host-to-device copy (or creates a zero-copy device buffer if the data already resides on device). The `NDArray` object carries a `DLTensor` descriptor — the same tensor descriptor used in the DLPack standard, enabling zero-copy interop with PyTorch, JAX, and NumPy.

### 10.2 PackedFunc — The Universal Calling Convention

Every TVM kernel, operator implementation, and runtime utility is exposed as a `PackedFunc`. This is TVM's equivalent to MLIR's `FuncOp` + `LLVM::CallIntr` convention: a single C ABI that threads through the entire stack.

```cpp
// C++ definition (include/tvm/runtime/packed_func.h)
using PackedFunc = std::function<void(TVMArgs args, TVMRetValue* ret)>;
```

`TVMArgs` is a type-erased argument list carrying `TVMValue` union elements tagged with `DLDataType`. The type erasure allows Python and C++ callers to invoke the same kernel handle without generating separate wrapper code.

From Python, any registered `PackedFunc` is callable as an ordinary Python function:

```python
add_func = tvm.get_global_func("tvm.runtime.ndarray.add")
result   = add_func(a_nd, b_nd)
```

From C++:

```cpp
#include <tvm/runtime/registry.h>
auto f = tvm::runtime::Registry::Get("my_kernel");
(*f)(a, b, c);  // TVMArgs deduction from argument types
```

### 10.3 Executor Variants

TVM provides three executor backends, selected at build time:

| Executor | Description | Use case |
|---|---|---|
| `GraphExecutor` | JSON graph + compiled .so | Cloud inference, production CPU/GPU |
| `RelaxVM` | Register-based VM with dynamic shapes | Dynamic batch, research |
| `AotExecutor` | Static C function, no allocator | microTVM, bare-metal |
| `VirtualMachineExecutor` (legacy Relay VM) | Stack-based, supports ADTs | Conditional models |

### 10.4 Profiling and Debugging

The TVM runtime exposes timing utilities for operator-level profiling:

```python
from tvm.contrib import graph_executor
import tvm

# Build with profiling enabled
with tvm.transform.PassContext(opt_level=3):
    lib = relay.build(mod, target=target, params=params)

dev  = tvm.cuda(0)
gmod = graph_executor.create(lib.get_graph_json(), lib, dev)
gmod.set_input("data", x_tvm)

# Warm up
for _ in range(5):
    gmod.run()

# Collect per-operator timing (in microseconds)
ftimer = gmod.module.time_evaluator("run", dev, number=10, repeat=3)
prof   = ftimer()
print(f"Mean: {prof.mean * 1e3:.3f} ms, Std: {prof.std * 1e3:.3f} ms")

# Per-layer breakdown
debug_executor = graph_executor.create(
    lib.get_graph_json(), lib, dev, dump_root="/tmp/tvmdbg"
)
debug_executor.run()
# /tmp/tvmdbg/ contains per-operator timing JSON
```

For GPU profiling at the kernel level, TVM can emit NVTX range markers that integrate with Nsight Systems:

```python
with tvm.transform.PassContext(
    opt_level=3,
    config={"relay.backend.use_auto_scheduler": True,
            "cuda.inject_nvtx": True}
):
    lib = relay.build(mod, target=tvm.target.cuda(), params=params)
```

The per-operator `time_evaluator` is implemented in [`src/runtime/graph_executor/debug/`](https://github.com/apache/tvm/tree/main/src/runtime/graph_executor/debug) and uses CUDA events (or CPU `clock_gettime`) to bracket each operator execution.

---

## 11. Comparison: IREE vs XLA vs TVM vs Triton

The following table consolidates the design-level distinctions discussed across [Ch153](../part-22-xla-openxla/ch153-xla-architecture.md), [Ch162](../part-23-mlir-production/ch162-iree-a-deployment-compiler.md), [Ch163](../part-23-mlir-production/ch163-triton-gpu-kernels.md), and this chapter. These comparisons reflect production release behavior as of 2025–2026; all four projects are actively developed and the gaps are narrowing.

| Dimension | XLA | IREE | TVM | Triton |
|---|---|---|---|---|
| **IR abstraction level** | HLO / StableHLO (op-graph, buffers) | Linalg → Flow → HAL (MLIR stack) | Relay → TIR (loop level) / Relax (unified) | Triton IR (tile level, Python DSL) |
| **Autotuning** | Limited: GemmFusion config, cuBLAS algo select | IREE Turbine / codegen parameter search | AutoTVM (templates), Ansor, Meta-Schedule | None (manual tile sizes in Python) |
| **Target breadth** | NVIDIA GPU, Google TPU, x86/Arm CPU | Vulkan, Metal, CUDA, CPU (via LLVM) | NVIDIA/AMD GPU, x86, Arm, RISC-V, custom NPU | NVIDIA GPU primary; AMD experimental |
| **Bare-metal MCU** | No | No (HAL requires OS services) | Yes (microTVM, Zephyr/Arduino) | No |
| **MLIR-native** | Partial (StableHLO ↔ HLO bridge) | Yes (native MLIR dialects) | Partial (Relax uses MLIR concepts; TIR is independent) | Yes (Triton IR → MLIR dialects) |
| **Framework integration** | JAX (first-class), TF, PyTorch/XLA | ONNX, TFLite, torch-mlir | PyTorch, TFLite, ONNX, MXNet | PyTorch (torch.compile backend) |
| **Custom hardware (BYOC/HAL)** | Limited (HloCustomCall) | HAL device drivers | BYOC: pattern match + codegen callback | No |
| **Runtime overhead** | Moderate (XLA runtime, buffer management) | Low (HAL thin layer) | Low (C runtime, NDArray) | Minimal (kernel library only) |
| **Operator library dependency** | Heavy (cuBLAS, cuDNN) | Minimal (some IREE kernels) | Optional (BYOC to cuBLAS if desired) | Generates own kernel from scratch |
| **Dynamic shapes** | Limited (XLA shape inference) | Yes (dynamic shapes in HAL) | Yes (Relax VM) | Limited |

The table reveals TVM's unique position: it is the only system that can target everything from H100 GPUs to a 256 KB Cortex-M4 flash, does so without mandatory vendor library dependencies, and provides an industrial-strength automated search framework. Its trade-off is compile-time cost — autotuning 1000+ operators in a full model requires hours of measurement — and a historically complex API surface that the Relax/Unity project is actively simplifying.

### 11.1 When to Choose TVM

**Choose TVM when:**
- Targeting hardware without vendor kernel libraries (new NPUs, RISC-V processors, custom ASICs with BYOC).
- Deploying to bare-metal MCUs where XLA and IREE cannot operate.
- The target operator mix is unusual enough that hand-tuned vendor kernels are unavailable (sparse attention variants, custom activations, non-standard data layouts).
- Autotuning time is acceptable and the hardware is stable enough that a tuning log can be reused across many deployments.
- The team needs fine-grained control over memory layout and loop structure (e.g., NCHWc layout for AVX-512 VNNI).

**Choose XLA when:**
- Running JAX or TensorFlow on NVIDIA GPUs or Google TPUs, where XLA's integration is first-class and vendor library performance is acceptable.
- Whole-program buffer assignment and cross-operator fusion at the HLO level are needed.

**Choose IREE when:**
- MLIR-native infrastructure is a hard requirement (reusing existing MLIR passes, integrating with torch-mlir, leveraging StableHLO).
- Deployment targets include Vulkan-capable embedded GPUs (Qualcomm Adreno, Arm Mali) where IREE's Vulkan backend provides better ecosystem support than TVM's.

**Choose Triton when:**
- Writing custom attention kernels, sparse GEMM, or other tile-structured GPU computations in Python is the primary workflow.
- The team is NVIDIA GPU-first and willing to invest in per-kernel optimization rather than automated whole-model tuning.

For an integrative view of where TVM fits in the broader AI compiler ecosystem, including its relationship to MLIR infrastructure, see [Ch179 — LLVM/MLIR for AI: Full-Stack Compilation](../part-26-ecosystem-frontiers/ch179-llvm-mlir-for-ai-full-stack.md).

---

## 12. TVM in Practice: LLM Compilation with MLC-LLM

Machine Learning Compilation for LLMs (MLC-LLM) is the most prominent production application of TVM's Relax stack as of 2025–2026. MLC-LLM compiles quantized LLMs (Llama, Mistral, Phi, Qwen families) to native executables on iOS, Android, WebGPU, CUDA, and Metal — a target breadth that no other single ML compiler system covers.

The compilation pipeline uses Relax throughout:

```bash
# Compile Llama-3-8B to CUDA with 4-bit GPTQ quantization
mlc_llm compile \
    --model HF://meta-llama/Meta-Llama-3-8B-Instruct \
    --quantization q4f16_1 \
    --target cuda \
    --output llama3_8b_q4f16_cuda/
```

Internally, `mlc_llm compile` invokes:
1. `relax.frontend.nn.to_relax` — converts the PyTorch model definition to a Relax `IRModule`.
2. `relax.transform.FuseTransposeMatmul` — fuses weight transposes into GEMM scheduling.
3. `relax.transform.LiftTransformParams` — lifts quantized weight dequantization into a separate offline phase.
4. `ms.relay_integration.compile_relay` — applies Meta-Schedule tuning or a pre-built tuning database.
5. `relax.build` with the target-specific backend.

The resulting artifact is a `.so` plus a JSON configuration, loadable via the MLC runtime:

```python
from mlc_llm import LLMEngine

engine = LLMEngine("llama3_8b_q4f16_cuda/")
output = engine.chat("Explain TVM's schedule primitives in one paragraph.")
```

MLC-LLM demonstrates that TVM's schedule-synthesis approach scales from 10 KB MCU models to 70B+ parameter LLMs, validating the generality of the TIR/Relax design.

---

## Chapter Summary

- **TVM's design center** is operator-level schedule synthesis: expressing how loop iterations are tiled, reordered, vectorized, and mapped to hardware intrinsics. This distinguishes it from XLA (graph-level, library-heavy) and IREE (MLIR-native VM dispatch).

- **The IR hierarchy** has three levels: Relay (typed functional graph IR for whole-model optimizations), TIR / TVMScript (imperative loop-level IR with `T.block` spatial/reduction axis abstraction), and Tensor Expression (legacy compute/schedule DSL, still used in AutoTVM templates).

- **Schedule primitives** (`split`, `fuse`, `reorder`, `compute_at`, `vectorize`, `unroll`, `parallel`, `tensorize`, `cache_read/write`) are composable transformations on loop nests that preserve mathematical semantics while exploiting hardware.

- **AutoTVM** parameterizes hand-written schedule templates with tunable knobs and searches the parameter space using an XGBoost cost model trained on measured hardware latencies. Transfer learning across similar hardware reduces measurement cost.

- **Ansor** (auto_scheduler) eliminates the need for hand-written templates by automatically generating schedule sketches and running evolutionary search. **Meta-Schedule** makes every component of this pipeline pluggable for production use.

- **BYOC** (Bring Your Own Codegen) partitions a Relay/Relax graph via pattern matching and routes matched subgraphs to external codegens (NPUs, DSPs, vendor libraries). The external codegen returns a `tvm.runtime.Module`; the TVM runtime handles dispatch.

- **Relax / TVM Unity** supersedes the Relay/TIR split with a single `IRModule` where `R.call_tir` bridges graph-level functions directly to loop-level `PrimFunc` kernels. The Relax VM supports dynamic shapes and integrates with Meta-Schedule for tuning.

- **Target support** is expressed through `tvm.target.Target` attribute strings; TVM natively targets CUDA (via LLVM NVPTX), Arm (via LLVM AArch64), x86 (via LLVM x86), and bare-metal C via `--executor=aot`.

- **microTVM** enables deployment on bare-metal microcontrollers (Cortex-M, RISC-V) by generating a static C AOT executor with no heap allocation, no dynamic dispatch, and optional Zephyr/Arduino project scaffolding.

- **The TVM runtime** is built on `PackedFunc` — a type-erased universal calling convention that threads through C++, Python, and generated kernels alike. NDArray wraps `DLTensor`, enabling zero-copy interop with PyTorch and JAX.
