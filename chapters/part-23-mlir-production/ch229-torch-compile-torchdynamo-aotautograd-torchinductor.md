# Chapter 229 — torch.compile: TorchDynamo, AOTAutograd, and TorchInductor

*Part XXIII — MLIR and ML Compilation Frameworks*

PyTorch 2.0 introduced `torch.compile`, a compiler stack that accelerates PyTorch programs without requiring users to change their models. Where earlier approaches required explicit graph capture (TorchScript, `torch.fx` transforms) or migration to a separate compiler (torch-mlir, XLA), `torch.compile` intercepts Python execution directly, extracts computational graphs invisibly, and emits optimized GPU kernels or vectorized CPU code. Understanding how this works requires following the path from a Python frame hook through three distinct compilation layers — TorchDynamo, AOTAutograd, and TorchInductor — each solving a different part of the problem.

---

## Table of Contents

- [229.1 The PyTorch 2.0 Compilation Model](#2291-the-pytorch-20-compilation-model)
  - [Eager vs Compiled Mode](#eager-vs-compiled-mode)
  - [What Compilation Gains](#what-compilation-gains)
  - [Compilation Pipeline Overview](#compilation-pipeline-overview)
  - [Relationship to Other PyTorch Compilation Approaches](#relationship-to-other-pytorch-compilation-approaches)
- [229.2 TorchDynamo: Frame-Level Graph Capture](#2292-torchdynamo-frame-level-graph-capture)
  - [CPython Frame Evaluation Hook (PEP 523)](#cpython-frame-evaluation-hook-pep-523)
  - [Proxy Tensors and Symbolic Execution](#proxy-tensors-and-symbolic-execution)
  - [Graph Breaks](#graph-breaks)
  - [Guard Types](#guard-types)
  - [Debugging Graph Breaks](#debugging-graph-breaks)
- [229.3 FX Graph Intermediate Representation](#2293-fx-graph-intermediate-representation)
  - [Graph Structure](#graph-structure)
  - [Node Types](#node-types)
  - [FX Passes](#fx-passes)
- [229.4 AOTAutograd: Joint Forward/Backward Tracing](#2294-aotautograd-joint-forwardbackward-tracing)
  - [The Problem AOTAutograd Solves](#the-problem-aotautograd-solves)
  - [How It Works](#how-it-works)
  - [Why AOTAutograd Enables Better Optimization](#why-aotautograd-enables-better-optimization)
- [229.5 TorchInductor: FX Graph to Native Code](#2295-torchinductor-fx-graph-to-native-code)
  - [Architecture](#architecture)
  - [GPU Path: FX → Triton](#gpu-path-fx-triton)
  - [CPU Path: FX → C++](#cpu-path-fx-c)
  - [AOTInductor: Standalone Inference Binaries](#aotinductor-standalone-inference-binaries)
- [229.6 torch.export and ExportedProgram](#2296-torchexport-and-exportedprogram)
  - [Strict vs Non-Strict Export](#strict-vs-non-strict-export)
  - [ExportedProgram Structure](#exportedprogram-structure)
  - [Serialization and Deployment](#serialization-and-deployment)
- [229.7 Guard System and Recompilation](#2297-guard-system-and-recompilation)
  - [Guard Cache Architecture](#guard-cache-architecture)
  - [Recompilation Limits](#recompilation-limits)
  - [Dynamic Shapes](#dynamic-shapes)
- [229.8 Backends and Ecosystem](#2298-backends-and-ecosystem)
  - [Backend Interface](#backend-interface)
  - [Built-in Backends](#built-in-backends)
  - [Stability (PyTorch 2.4–2.6)](#stability-pytorch-2426)
- [Chapter Summary](#chapter-summary)

---

## 229.1 The PyTorch 2.0 Compilation Model

### Eager vs Compiled Mode

PyTorch's default execution mode is **eager**: each PyTorch operation executes immediately as Python calls it. Eager mode is easy to debug and fully Pythonic but incurs overhead from Python interpreter dispatch, operator-by-operator kernel launches, and missed fusion opportunities across operator boundaries.

`torch.compile` adds a compiled mode that captures a computational graph from the Python code, then hands it to a backend for optimization:

```python
import torch

def model(x, y):
    return torch.nn.functional.relu(x @ y + x)

# Eager: each op dispatches immediately
out_eager = model(x, y)

# Compiled: graph captured and optimized
compiled_model = torch.compile(model)
out_compiled = compiled_model(x, y)   # first call: compile + run
out_compiled = compiled_model(x, y)   # subsequent: run cached compiled artifact
```

### What Compilation Gains

- **Kernel fusion**: `relu(matmul(x, y) + x)` can be fused into a single GPU kernel, eliminating intermediate tensor allocations and multiple kernel launches
- **Elimination of Python overhead**: the compiled path executes native code without re-entering the Python interpreter
- **Layout and memory optimization**: inductor can choose optimal tensor layouts and buffer reuse strategies
- **Vertical and horizontal fusion**: pointwise ops after matmul (vertical); consecutive pointwise ops (horizontal)

### Compilation Pipeline Overview

```
Python function
    ↓  [TorchDynamo — PEP 523 frame hook]
FX Graph (ATen ops + symbolic shapes)
    ↓  [AOTAutograd — joint forward/backward trace]
Forward + Backward FX Graphs (functional, decomposed)
    ↓  [TorchInductor — backend lowering]
Triton kernels (GPU) / C++ templates (CPU)
    ↓  [Triton / LLVM backend]
PTX / SASS / x86 native code
```

### Relationship to Other PyTorch Compilation Approaches

`torch.compile` is distinct from:
- **torch-mlir**: offline AOT conversion to MLIR dialects (Torch → StableHLO/Linalg/TOSA); no Python at inference; cross-platform deployment; covered in [Chapter 161](../part-23-mlir-production/ch161-torch-mlir-onnx-jax-bridges.md)
- **TorchScript**: static-graph tracing/scripting; stable but limited Python subset; being supplanted by `torch.export`
- **XLA/JAX compilation**: functional transforms + XLA HLO; requires JAX semantics; covered in [Chapter 210](../part-31-frontier-ai-evolution/ch210-jax-ecosystem.md)

---

## 229.2 TorchDynamo: Frame-Level Graph Capture

### CPython Frame Evaluation Hook (PEP 523)

Python 3.6 introduced PEP 523 (`_PyInterpreterState_SetEvalFrameFunc`), which allows replacing the CPython frame evaluation function. TorchDynamo installs a custom evaluator that intercepts function frames containing tensor operations:

```c
// CPython internal: PEP 523 hook point
typedef PyObject* (*_PyFrameEvalFunction)(
    PyThreadState *tstate, PyFrameObject *frame, int throwflag);

_PyInterpreterState_SetEvalFrameFunc(
    PyInterpreterState_Get(), dynamo_eval_frame);
```

Source: `torch/_dynamo/eval_frame.py` (Python) and `torch/csrc/dynamo/eval_frame.cpp` (C extension).

When a `torch.compile`-decorated function is called, Dynamo's evaluator:
1. Inspects the CPython bytecode of the frame
2. Symbolically executes operations on **proxy tensors** (fake tensors that record operations without materializing data)
3. Accumulates operations into an FX graph
4. Compiles the FX graph and caches the result
5. Executes the compiled artifact on the real tensors

### Proxy Tensors and Symbolic Execution

Proxy tensors are stand-ins for real tensors during tracing:

```python
# During tracing, operations on proxy tensors record into the graph
proxy_x = ProxyTensor(shape=(batch, seq, hidden), dtype=torch.float32)
proxy_y = ProxyTensor(shape=(hidden, hidden), dtype=torch.float32)

# This call doesn't run the matmul; it records the operation
out = proxy_x @ proxy_y   # records: out = matmul(proxy_x, proxy_y)
```

### Graph Breaks

TorchDynamo cannot capture arbitrary Python. When it encounters unsupported constructs, it **breaks the graph** and restarts a fresh compilation context:

| Construct | Effect |
|-----------|--------|
| `print(tensor)` | Graph break (data-dependent Python side effect) |
| `if tensor.item() > 0:` | Graph break (data-dependent control flow) |
| `for i in range(n):` (static `n`) | Unrolled into graph |
| `len(tensor)` (dynamic shape) | Guard + possible break |
| Calls into `torch.nn.Module` | Inlined if purely PyTorch |
| Calls into unsupported Python | Graph break |

Multiple graph breaks produce multiple compiled regions stitched together with Python execution in between. `torch.compile(fullgraph=True)` raises an error on any graph break, which is useful for diagnosing performance issues.

### Guard Types

When Dynamo compiles a graph, it records **guards** — assumptions that must hold for the compiled graph to be reused:

| Guard type | Example | Trigger for recompile |
|------------|---------|----------------------|
| Shape guard | `x.shape == (32, 128, 512)` | Input shape changes |
| Dtype guard | `x.dtype == torch.float16` | Input dtype changes |
| Value guard | `dropout_p == 0.1` | Python scalar changes |
| Object identity guard | `model.weight is W` | Weight tensor replaced |
| Device guard | `x.device == cuda:0` | Device changes |

Guards are checked before each invocation. On failure, Dynamo retraces and recompiles.

### Debugging Graph Breaks

```python
# Explain why a function produces graph breaks
explanation = torch._dynamo.explain(model)(x, y)
print(explanation.graphs)        # list of captured FX graphs
print(explanation.break_reasons) # why each break occurred

# Force recompilation of all cached graphs
torch._dynamo.reset()

# Enable verbose tracing
import os; os.environ["TORCH_COMPILE_DEBUG"] = "1"
```

Dynamic shapes (avoiding shape guards): `torch.compile(dynamic=True)` uses symbolic integers (`torch.SymInt`) so the graph generalizes across sizes without recompilation.

---

## 229.3 FX Graph Intermediate Representation

### Graph Structure

`torch.fx.Graph` is a Python-level SSA-like representation of a PyTorch computation. Each node produces one output value:

```python
import torch.fx as fx

# FX graph for: lambda x: torch.relu(x + 1)
graph = fx.Graph()
x    = graph.placeholder('x')                          # input
one  = graph.get_attr('one')                           # constant
add  = graph.call_function(torch.add, args=(x, one))   # x + 1
relu = graph.call_function(torch.relu, args=(add,))    # relu(...)
out  = graph.output(relu)                              # return value

gm = fx.GraphModule(root_module, graph)
```

### Node Types

| Node kind | Description | Example |
|-----------|-------------|---------|
| `placeholder` | Function input | `x: Tensor` |
| `call_function` | Global function call | `torch.add(x, y)` |
| `call_method` | Method on a value | `x.reshape(...)` |
| `call_module` | Call an `nn.Module` | `self.linear(x)` |
| `get_attr` | Module attribute access | `self.weight` |
| `output` | Return value | `return x` |

The graph is a pure functional DAG: no explicit loops or branches (both are unrolled or converted to `torch.cond`/`torch.vmap` at trace time). Symbolic shapes propagate through the graph without materializing data.

Source: `torch/fx/graph.py`, `torch/fx/node.py`.

### FX Passes

After capture, FX passes normalize the graph before backend lowering:

- **Decomposition**: `torch.ops.aten.linear` → `matmul` + `add` (complex ops split into primitives)
- **Shape inference**: propagate concrete shapes or symbolic shape expressions through the graph
- **Dead code elimination**: remove unused nodes
- **Fusion passes**: identify fusable operator sequences

---

## 229.4 AOTAutograd: Joint Forward/Backward Tracing

### The Problem AOTAutograd Solves

PyTorch's standard autograd builds a dynamic backward graph at runtime: each forward op registers a backward function, and `.backward()` traverses the chain. This is flexible but prevents cross-kernel optimization of the backward pass.

**AOTAutograd** traces both forward and backward passes *before* execution, producing a static computational graph of the entire forward+backward computation:

```python
from torch._functorch.aot_autograd import aot_function

# Joint forward+backward trace
def fwd_bwd_graph(x, y):
    out = model(x, y)
    out.sum().backward()  # backward is traced, not executed

compiled = aot_function(fwd_bwd_graph, fw_compiler=inductor_compile)
```

The name "ahead-of-time" refers to AOT *with respect to autograd graph construction*, not ahead of module execution — the compilation happens on first call.

### How It Works

AOTAutograd uses `__torch_dispatch__` to intercept tensor operations during forward execution:

1. **Forward trace**: run the forward function on fake tensors; record each ATen operation
2. **Decompose**: replace compound ops with primitives from `functorch.decompositions` (enables better backward differentiation and fusion)
3. **Differentiate**: apply functional autograd differentiation rules to produce backward graph nodes
4. **Partition**: split joint graph into forward subgraph (produces activations for backward) and backward subgraph (consumes saved activations)
5. **Rematerialization**: `min_cut_rematerialization_partition` computes optimal activation checkpointing — which activations to save vs recompute in the backward pass, minimizing memory

Source: `torch/_functorch/aot_autograd.py`.

### Why AOTAutograd Enables Better Optimization

Without AOTAutograd, the backward pass is a dynamic chain of closure calls — opaque to any compiler. With AOTAutograd, the backward is a static FX graph, enabling:
- Fusion of backward kernels with forward kernels
- Dead node elimination in the backward pass
- Optimal activation recomputation via min-cut analysis
- Persistent buffers across forward/backward for fused optimizers

---

## 229.5 TorchInductor: FX Graph to Native Code

### Architecture

TorchInductor is the default backend for `torch.compile`. It receives the decomposed FX graph from AOTAutograd and generates Triton kernels (GPU) or C++ templates (CPU).

Source: `torch/_inductor/`.

### GPU Path: FX → Triton

```
FX Graph (ATen ops)
    ↓  [pointwise/reduction/matmul classification]
LoopIR (loop nests over tensor indices)
    ↓  [fusion decisions, tile sizing, vectorization]
Triton IR (tl.load / tl.store / tl.dot / tl.reduce)
    ↓  [Triton compiler: Triton → LLVM → PTX → SASS]
GPU kernel binary
```

Inductor classifies each op into one of:
- **Pointwise**: elementwise ops (add, mul, relu); trivially fusable into a single kernel
- **Reduction**: sum, max, mean; require synchronization across dimensions
- **Matmul/conv**: dense linear algebra; dispatched to cuBLAS/cuDNN or custom Triton kernels
- **Scatter/gather**: irregular memory access; limited fusion

**Vertical fusion**: a pointwise op following a matmul is fused into the matmul's output tile, avoiding a round-trip to global memory.

**Horizontal fusion**: consecutive pointwise ops reading the same input are merged into a single kernel pass.

**Tile sizing**: the `_LoopBodyVectorizer` selects tile sizes based on register file pressure and shared memory constraints. `torch.compile(mode="max-autotune")` benchmarks candidate tile configurations and selects the fastest.

```python
# Generated Triton kernel (simplified)
@triton.jit
def fused_relu_bias_kernel(x_ptr, bias_ptr, out_ptr, n_elements, BLOCK: tl.constexpr):
    pid = tl.program_id(0)
    offs = pid * BLOCK + tl.arange(0, BLOCK)
    mask = offs < n_elements
    x    = tl.load(x_ptr + offs, mask=mask)
    bias = tl.load(bias_ptr + offs, mask=mask)
    out  = tl.maximum(x + bias, 0.0)   # relu(x + bias) fused
    tl.store(out_ptr + offs, out, mask=mask)
```

### CPU Path: FX → C++

For CPU targets, Inductor generates C++ template code and compiles it with the system compiler:

```cpp
// Generated C++ (simplified)
extern "C" void kernel(float* x, float* bias, float* out, int64_t n) {
    #pragma omp parallel for simd
    for (int64_t i = 0; i < n; ++i)
        out[i] = std::max(x[i] + bias[i], 0.0f);
}
```

The generated code is passed through LLVM's `LoopVectorizer` pass for auto-vectorization. Memory layout optimization (e.g., NCHW vs NHWC choice for conv) is resolved at this stage.

### AOTInductor: Standalone Inference Binaries

`torch._inductor.aot_compile` produces a standalone `.so` shared library with a stable C API:

```python
# Compile to standalone .so
ep = torch.export.export(model, (example_x,))
path = torch._inductor.aot_compile(ep)  # returns path to .so

# Deploy without Python runtime
# C API: torch_inductor_run(handle, inputs, outputs)
```

The AOTInductor C API is stable as of PyTorch 2.4, enabling deployment to mobile, server, and embedded targets without a Python interpreter.

---

## 229.6 torch.export and ExportedProgram

### Strict vs Non-Strict Export

`torch.export.export` produces a serializable, deployable representation:

```python
from torch.export import export

# Strict mode: TorchDynamo traces (bytecode analysis)
ep_strict = export(model, (x,), strict=True)

# Non-strict mode: traces actual code paths via fake tensors
ep_nonstrictexport = export(model, (x,), strict=False)  # recommended default
```

Both modes produce identical `ExportedProgram` objects for well-behaved models. Non-strict is preferred as it handles more Python patterns correctly.

### ExportedProgram Structure

```python
ep = export(model, (x,))

ep.graph           # FX graph with torch._export.* ops
ep.graph_signature # input/output/parameter/buffer specs
ep.state_dict      # model weights
ep.range_constraints  # symbolic shape constraints
```

### Serialization and Deployment

```python
# Save to ZIP archive (FX bytecode + tensors + metadata)
torch.export.save(ep, "model.pt2")
ep_loaded = torch.export.load("model.pt2")

# Deploy to ExecuTorch (mobile/embedded)
from executorch.exir import to_edge
edge_program = to_edge(ep)
# Further lowering to target-specific delegate (XNNPACK, CoreML, etc.)
```

The `.pt2` format (PyTorch 2 export format) is a ZIP file containing serialized FX graph bytecode, parameter tensors, and metadata (input shapes, quantization info).

---

## 229.7 Guard System and Recompilation

### Guard Cache Architecture

Each compiled artifact is stored in a guard cache keyed by (function identity, guard set). Before executing a cached artifact, Dynamo evaluates all guards:

```python
# Internal guard check (simplified)
def check_guards(cached_artifact, inputs):
    for guard in cached_artifact.guards:
        if not guard.check(inputs):
            return False   # cache miss → recompile
    return True            # cache hit → run artifact
```

### Recompilation Limits

Excessive recompilation is expensive and usually indicates a design problem. Dynamo limits recompilation per function (default: 8 compilations). After the limit is exceeded, Dynamo falls back to eager mode for that function and emits a warning.

```python
# Diagnose excessive recompilation
torch._dynamo.config.verbose = True  # log each recompile reason
```

### Dynamic Shapes

With `torch.compile(dynamic=True)`, Dynamo uses **symbolic integers** (`torch.SymInt`) for dimension values. The compiler emits guards on relationships between dimensions (e.g., `seq_len <= 2048`) rather than exact values, allowing the compiled artifact to handle variable-length inputs:

```python
compiled = torch.compile(model, dynamic=True)
compiled(torch.randn(8, 128, 512))   # compiled with symbolic batch/seq dims
compiled(torch.randn(16, 256, 512))  # reuses compiled artifact (guards still hold)
```

---

## 229.8 Backends and Ecosystem

### Backend Interface

```python
# Custom backend registration
import torch._dynamo as dynamo

@dynamo.register_backend
def my_backend(gm: torch.fx.GraphModule, example_inputs):
    # gm: captured FX graph
    # example_inputs: sample tensors for shape inference
    # return: callable that executes the optimized graph
    optimized = my_compiler.compile(gm)
    return optimized

compiled = torch.compile(model, backend="my_backend")
```

### Built-in Backends

| Backend | Use case |
|---------|----------|
| `"inductor"` | Default; Triton (GPU) / C++ (CPU) |
| `"cudagraphs"` | CUDA graph capture; zero overhead for fixed shapes |
| `"onnxrt"` | ONNX Runtime; cross-platform deployment |
| `"tvm"` | Apache TVM; ML hardware targets |
| `"eager"` | Debugging; runs FX graph in eager mode |
| `"aot_eager"` | AOTAutograd + eager; isolates Inductor issues |

### Stability (PyTorch 2.4–2.6)

| Use case | Status |
|----------|--------|
| Inference (static shapes) | Production-ready; 1.5–2× typical speedup |
| Inference (dynamic shapes) | Beta; symbolic shape compilation |
| Training | Beta; graph breaks on some distributed patterns |
| AOTInductor C API | Stable; ABI stability guaranteed |
| `torch.export` | Stable for single-device models |
| Distributed (FSDP2 + compile) | Alpha; `torch.compile(model)` wrapping FSDP |

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **Stable symbolic shape compilation**: The `torch.compile(dynamic=True)` path is graduating from Beta to Stable; remaining blockers around `torch.export` + symbolic shapes in FSDP2 distributed training are tracked in PyTorch RFC #126132 and `torch/_dynamo/symbolic_shapes.py` refactors targeting PyTorch 2.7.
- **AOTInductor ABI freeze and C++ API expansion**: PyTorch 2.7 is expected to stabilize the `torch_inductor_run_with_state` C++ API, enabling multi-model weight sharing in AOTInductor-compiled `.so` binaries; tracked in `torch/_inductor/package.py` and the `aotinductor-api` label on pytorch/pytorch.
- **Inductor FP8 training kernel generation**: TorchInductor is gaining direct Triton-level FP8 gemm emission (replacing the current cuBLASLt dispatch shim) to support BF16/FP8 mixed-precision training pipelines without Inductor graph breaks; prototype patches landed in `torch/_inductor/kernel/mm.py` in early 2026.
- **`torch.compile` + `torch.distributed.pipelining`**: The pipeline-parallelism integration — where each pipeline stage is a separately compiled `torch.export`-exported subgraph — is advancing toward Beta; the `PipelineStage`/`compile`-compatible API is being stabilized in `torch/distributed/pipelining/`.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Convergence of `torch.compile` and torch-mlir lowering paths**: Efforts are underway (OpenXLA/StableHLO working group, PyTorch + IREE integration) to let TorchInductor optionally emit StableHLO rather than Triton, unifying the JIT and AOT MLIR paths and enabling deployment to TPUs and custom accelerators without two separate compilation stacks.
- **Persistent compilation cache (PGO-driven Inductor)**: TorchInductor's `max_autotune` cache currently resets per process. A disk-persistent, profile-guided cache (`torch._inductor.codecache` redesign, discussed at PyTorch 2025 Developer Day) would allow multi-run tuning data to accumulate, converging on optimal tile sizes after several training runs — similar to LLVM's PGO database.
- **First-class higher-order op support in TorchDynamo**: `torch.vmap`, `torch.cond`, `torch.while_loop`, and nested `torch.compile` calls are partially supported; full support for arbitrary higher-order ops (avoiding graph breaks on `functorch`-style transforms) is a known correctness gap, tracked in the `higher-order-ops` milestone on pytorch/pytorch.
- **TorchInductor for AMD ROCm parity with CUDA**: The Triton-ROCm backend (`triton-lang/triton` ROCm fork) is being upstreamed; Inductor's autotune heuristics currently assume CUDA-specific register file / shared memory assumptions; a hardware-agnostic tuning model is needed for production ROCm deployments.

### 5-Year Horizon (Long-Term, by ~2031)

- **Whole-program AOT compilation of transformer training loops**: The current TorchDynamo model captures per-frame graphs and stitches them via Python. A longer-term direction — analogous to JAX's full-program XLA compilation — would export an entire training iteration (forward + backward + optimizer update + data pipeline) as a single static graph, enabling cross-iteration operator scheduling and memory planning across thousands of operators.
- **Formal semantics for the ATen operator set and guard language**: The guard system (shape guards, value guards, object identity guards) currently relies on runtime Python evaluation with no formal specification; a typed, formally verified guard language (building on the `torch.fx.experimental.symbolic_shapes` infrastructure) would enable static verification that guard sets are complete and non-redundant, reducing incorrect fallbacks to eager mode.
- **AI-driven kernel generation replacing hand-authored Triton templates**: Research prototypes (e.g., AlphaCode-derived kernel synthesis, Meta's KernelBench evaluations) suggest that LLM-assisted or RL-trained kernel generators could supersede hand-written Triton templates for the long tail of operator shapes not well served by Inductor's current heuristics, with Inductor acting as a verification harness rather than a primary code generator.

---

## Chapter Summary

- `torch.compile` uses TorchDynamo's PEP 523 frame hook to intercept Python execution and capture FX graphs without requiring model changes; graph breaks divide functions into separately compiled regions
- TorchDynamo records guards (shape, dtype, value, object identity) with each compiled artifact; guard failure triggers recompilation; `dynamic=True` uses symbolic integers to generalize across shapes
- FX Graph is a functional DAG of ATen operations (`torch.fx.Graph`/`GraphModule`/`Node`); node kinds: placeholder, call_function, call_method, get_attr, output; source in `torch/fx/`
- AOTAutograd traces forward and backward passes jointly using `__torch_dispatch__`, enabling cross-kernel fusion of the backward pass and optimal activation checkpointing via min-cut rematerialization; source in `torch/_functorch/`
- TorchInductor emits Triton kernels (GPU) with vertical and horizontal fusion, and C++ templates (CPU) with LLVM vectorization; `max_autotune` mode benchmarks tile configurations; source in `torch/_inductor/`
- `torch.export` / `ExportedProgram` provides a serializable, deployable graph (`strict` vs `non-strict` modes); `.pt2` ZIP format; ExecuTorch deployment path for mobile/embedded inference without a Python runtime
- The AOTInductor C API (`torch._inductor.aot_compile`) produces a standalone `.so` with a stable C interface, decoupling inference from the Python runtime entirely
- Compared to torch-mlir (offline MLIR conversion) and XLA (functional program transforms), `torch.compile` is a JIT path that stays within the Python interpreter ecosystem while still enabling kernel fusion and layout optimization

---

@copyright jreuben11
