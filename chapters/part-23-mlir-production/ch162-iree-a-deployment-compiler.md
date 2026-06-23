# Chapter 162 — IREE: A Deployment Compiler

*Part XXIII — MLIR in Production*

IREE (Intermediate Representation Execution Environment) is a full-stack ML deployment compiler built entirely on MLIR. Where other ML compilers treat MLIR as an intermediate layer in a larger system, IREE uses MLIR dialects end-to-end — from the input (StableHLO/TOSA/Linalg) all the way to the hardware abstraction layer (HAL) and the final binary artifact (the `.vmfb` FlatBuffer). IREE's design prioritizes portable deployment: a single compilation produces a binary that runs on CPU, CUDA, Vulkan, Metal, or ROCm. This chapter covers IREE's architecture in depth, including its key intermediate dialects (`flow`, `stream`, HAL), its target backends, its Python API, and the performance characteristics that make it competitive with TensorRT for production inference.

---

## Table of Contents

- [162.1 IREE Architecture Overview](#1621-iree-architecture-overview)
- [162.2 The Flow Dialect](#1622-the-flow-dialect)
  - [162.2.1 flow.dispatch.region](#16221-flowdispatchregion)
  - [162.2.2 Dispatch Formation Heuristics](#16222-dispatch-formation-heuristics)
  - [162.2.3 flow.dispatch.tensor Protocol](#16223-flowdispatchtensor-protocol)
- [162.3 The Stream Dialect](#1623-the-stream-dialect)
  - [162.3.1 stream.cmd.execute](#16231-streamcmdexecute)
  - [162.3.2 Resource Types](#16232-resource-types)
  - [162.3.3 Async Dependency Tracking](#16233-async-dependency-tracking)
- [162.4 The HAL Dialect](#1624-the-hal-dialect)
  - [162.4.1 hal.executable](#16241-halexecutable)
  - [162.4.2 hal.command_buffer](#16242-halcommandbuffer)
- [162.5 Target Backends](#1625-target-backends)
  - [162.5.1 llvm-cpu Backend](#16251-llvm-cpu-backend)
  - [162.5.2 cuda Backend](#16252-cuda-backend)
  - [162.5.3 vulkan-spirv Backend](#16253-vulkan-spirv-backend)
- [162.6 The VMFB Format](#1626-the-vmfb-format)
- [162.7 Python API](#1627-python-api)
  - [162.7.1 Compilation](#16271-compilation)
  - [162.7.2 Runtime Execution](#16272-runtime-execution)
  - [162.7.3 Compiler from MLIR Python Bindings](#16273-compiler-from-mlir-python-bindings)
- [162.8 Performance Engineering in IREE](#1628-performance-engineering-in-iree)
  - [162.8.1 Tuning Dispatch Tiling](#16281-tuning-dispatch-tiling)
  - [162.8.2 Weight Bundling](#16282-weight-bundling)
  - [162.8.3 Benchmarking](#16283-benchmarking)
- [162.9 Comparing IREE with Alternatives](#1629-comparing-iree-with-alternatives)
- [Chapter Summary](#chapter-summary)

---

## 162.1 IREE Architecture Overview

IREE's compilation pipeline transforms ML computations through a sequence of MLIR dialects, each representing a different level of the execution model:

```
Input Dialects
├── stablehlo.*       (from JAX/TF/torch-mlir)
├── tosa.*            (from torch-mlir/TFLite)
└── linalg.*          (direct from framework bridges)
    │
    ▼ iree-import-public (legalization + decomposition)
    
flow dialect
├── flow.dispatch.region   (compute regions → kernels)
├── flow.dispatch.tensor   (tensor protocol for dispatch args)
└── flow.channel.*         (collective communication)
    │
    ▼ flow → stream lowering

stream dialect
├── stream.cmd.execute     (async command buffer recording)
├── stream.cmd.dispatch    (kernel dispatch command)
├── stream.resource.*      (abstract buffer resources)
└── stream.timepoint.*     (async dependency tracking)
    │
    ▼ stream → hal lowering

hal dialect (Hardware Abstraction Layer)
├── hal.executable         (compiled kernel object)
├── hal.command_buffer.*   (GPU command buffer ops)
├── hal.buffer.*           (device buffer management)
└── hal.device.*           (device queries)
    │
    ▼ hal → target backend

Target Backends
├── llvm-cpu   → LLVM IR → native .so
├── cuda       → PTX → cubin (embedded in flatbuffer)
├── vulkan-spirv → SPIR-V → Vulkan shader
├── metal-spirv → MSL → Metal shader
└── rocm       → HSA → AMD binary
    │
    ▼
.vmfb (VM FlatBuffer) — portable deployment artifact
```

---

## 162.2 The Flow Dialect

The `flow` dialect represents the logical computation graph with explicit dispatch regions. Its primary purpose is to identify which portions of the computation can be fused into a single GPU/CPU kernel.

### 162.2.1 flow.dispatch.region

The central concept is the `flow.dispatch.region` — a region of MLIR ops that will be compiled into a single kernel:

```mlir
// Before flow dispatch formation: unfused linalg ops
func.func @forward(%input: tensor<4x8xf32>, %weight: tensor<8x4xf32>)
    -> tensor<4x4xf32> {
  %empty = tensor.empty() : tensor<4x4xf32>
  %matmul = linalg.matmul ins(%input, %weight : tensor<4x8xf32>, tensor<8x4xf32>)
                           outs(%empty : tensor<4x4xf32>) -> tensor<4x4xf32>
  %relu = linalg.generic {
    indexing_maps = [affine_map<(d0,d1) -> (d0,d1)>,
                     affine_map<(d0,d1) -> (d0,d1)>],
    iterator_types = ["parallel", "parallel"]}
    ins(%matmul : tensor<4x4xf32>) outs(%empty2 : tensor<4x4xf32>) {
    ^bb0(%in: f32, %out: f32):
      %relu_val = arith.maximumf %in, %zero : f32
      linalg.yield %relu_val : f32
  } -> tensor<4x4xf32>
  return %relu : tensor<4x4xf32>
}

// After flow dispatch formation: fused into one dispatch region
func.func @forward(%input: tensor<4x8xf32>, %weight: tensor<8x4xf32>)
    -> tensor<4x4xf32> {
  %result = flow.dispatch.region -> tensor<4x4xf32> {
    // Matmul + relu fused into one kernel
    %empty = tensor.empty() : tensor<4x4xf32>
    %matmul = linalg.matmul ...
    %relu = linalg.generic { ... }
    flow.return %relu : tensor<4x4xf32>
  }
  return %result : tensor<4x4xf32>
}
```

### 162.2.2 Dispatch Formation Heuristics

IREE's `FormDispatchRegionsPass` groups ops into dispatch regions based on:
1. **Producer-consumer fusion**: a producer is fused with its consumer if the consumer is element-wise
2. **Reduction boundary**: reductions create a boundary (cannot fuse past a reduce)
3. **Memory pressure**: regions that exceed available shared memory are split

### 162.2.3 flow.dispatch.tensor Protocol

`flow.dispatch.tensor` ops implement the protocol for passing tensors into and out of dispatch regions:

```mlir
%arg = flow.dispatch.tensor.load %input, offsets=[0], sizes=[4,8], strides=[1,1]
    : !flow.dispatch.tensor<readonly:tensor<4x8xf32>> -> tensor<4x8xf32>

flow.dispatch.tensor.store %result, %output, offsets=[0], sizes=[4,4], strides=[1,1]
    : tensor<4x4xf32> -> !flow.dispatch.tensor<writeonly:tensor<4x4xf32>>
```

These ops are lowered to buffer accesses in the target backend.

---

## 162.3 The Stream Dialect

The `stream` dialect models async GPU execution. It replaces MLIR's `async` dialect with a lower-level abstraction closer to GPU command buffers.

### 162.3.1 stream.cmd.execute

`stream.cmd.execute` records a sequence of GPU commands that execute asynchronously:

```mlir
// Async GPU execution
%result_resource, %timepoint = stream.cmd.execute
    with(%input_resource as %input: !stream.resource<transient>{4x8xf32},
         %output_resource as %output: !stream.resource<transient>{4x4xf32}) {
  
  // Record a kernel dispatch command
  stream.cmd.dispatch @matmul_relu_kernel[%workgroup_x, %workgroup_y, 1](%input, %output)
      : (!stream.resource<transient>, !stream.resource<transient>)
  
} => !stream.timepoint

// Await completion before reading results
%result = stream.timepoint.await %timepoint
    => !stream.resource<transient>
```

### 162.3.2 Resource Types

Stream resources are abstract buffer handles with an access qualifier:
- `!stream.resource<constant>`: immutable (weights)
- `!stream.resource<transient>`: short-lived (activations, intermediates)
- `!stream.resource<variable>`: persistent mutable state (optimizer state)
- `!stream.resource<external>`: provided by the caller (model inputs/outputs)

The access qualifier drives lifetime analysis and memory pool assignment.

### 162.3.3 Async Dependency Tracking

`stream.timepoint` values represent points in time that can be awaited:

```mlir
// Chain two async dispatches
%tp1 = stream.cmd.execute { ... } => !stream.timepoint
%tp2 = stream.cmd.execute with(...) after %tp1 { ... } => !stream.timepoint
stream.timepoint.await %tp2 => !stream.resource<transient>
```

This structure allows IREE to build a dependency graph and schedule GPU work with maximum parallelism.

---

## 162.4 The HAL Dialect

HAL (Hardware Abstraction Layer) is IREE's portable device interface. It exposes the minimal set of operations needed to manage device memory, compile kernels, and launch work.

### 162.4.1 hal.executable

`hal.executable` is a container for compiled GPU kernel objects. Each target backend compiles the executable differently:

```mlir
hal.executable @matmul_relu_kernel {
  hal.executable.variant @cuda target("cuda") {
    hal.executable.condition(%device: !hal.device) -> i1 {
      // Check if this variant's requirements are met
      %ok = hal.device.query<"cuda.compute_capability_major">(%device) : i1
      hal.return %ok : i1
    }
    builtin.module {
      func.func @dispatch(%arg0: !hal.interface.binding.subspan<...>, ...) {
        // GPU kernel code in NVVM dialect
        ...
      }
    }
  }
  hal.executable.variant @llvm-cpu target("llvm-cpu") {
    // CPU fallback variant
    ...
  }
}
```

A single `hal.executable` can contain multiple variants for different targets. At runtime, the IREE VM selects the variant that matches the available hardware.

### 162.4.2 hal.command_buffer

```mlir
// Record GPU commands
%cmd_buf = hal.command_buffer.create device(%device)
    mode("OneShot") : !hal.command_buffer

hal.command_buffer.begin<%cmd_buf : !hal.command_buffer>

// Fill a buffer
hal.command_buffer.fill_buffer<%cmd_buf : !hal.command_buffer>
    target(%buffer : !hal.buffer)[offset = 0, length = 1024]
    pattern(0 : i32)

// Dispatch a kernel
hal.command_buffer.dispatch<%cmd_buf : !hal.command_buffer>
    target(%executable : !hal.executable[0])
    workgroups([4, 2, 1])

hal.command_buffer.end<%cmd_buf : !hal.command_buffer>
hal.command_buffer.submit<%cmd_buf : !hal.command_buffer>
    wait(%wait_fence : !hal.fence) signal(%signal_fence : !hal.fence)
```

---

## 162.5 Target Backends

### 162.5.1 llvm-cpu Backend

The LLVM CPU backend compiles dispatch regions to native code via LLVM IR:

```
flow.dispatch.region (linalg ops)
    │ tiling + vectorization (transform dialect)
    ▼
vector.contract, scf.for (tiled + vectorized)
    │ lower-to-llvm pipeline
    ▼
llvm dialect
    │ mlir-translate --mlir-to-llvmir
    ▼
LLVM IR → native .so (x86-64 / AArch64)
```

IREE's CPU backend uses MLIR's transform dialect to tile and vectorize linalg ops before lowering:

```mlir
// IREE's tile + vectorize strategy for CPU matmul
transform.sequence failures(propagate) {
  ^bb0(%module: !transform.any_op):
    %matmul = transform.structured.match ops{["linalg.matmul"]} in %module
    
    // Tile for L1 cache (64x64x256 tiles)
    %tiled, %loops = transform.structured.tile_using_for %matmul [64, 64, 256]
    
    // Vectorize the inner tile
    transform.structured.vectorize %tiled
    
    // Unroll inner loop for register reuse
    transform.loop.unroll %loops#2 factor 4
}
```

### 162.5.2 cuda Backend

The CUDA backend compiles to PTX via LLVM's NVPTX backend:

```
linalg.generic (dispatch region)
    │ iree-linalg-ext extensions (tiling, im2col)
    ▼
gpu.launch (explicit thread/block layout)
    │ gpu → nvvm lowering
    ▼
nvvm dialect (NVPTX intrinsics)
    │ mlir-translate --mlir-to-llvmir
    ▼
LLVM IR (nvptx64) → PTX → ptxas → cubin
    │ embedded in .vmfb
```

For matrix operations, IREE uses warp-level MMA operations via `nvgpu.warpgroup.mma`.

### 162.5.3 vulkan-spirv Backend

```
linalg.generic
    │ spirv-specific tiling
    ▼
gpu.launch + vector ops
    │ convert-gpu-to-spirv
    ▼
spirv dialect (Vulkan ops)
    │ serialize to SPIR-V binary
    ▼
Vulkan compute shader (embedded in .vmfb)
```

IREE's Vulkan backend is notable for supporting WebGPU: the same SPIR-V shaders run on WebGPU-capable browsers via wgpu.

---

## 162.6 The VMFB Format

`.vmfb` (VM FlatBuffer) is IREE's portable deployment format. A single `.vmfb` file contains:

- **FlatBuffer metadata**: module structure, function signatures, buffer layouts
- **Compiled executables**: cubin/SPIR-V/native code for each target variant
- **VM bytecode**: instructions for the IREE VM (memory management, dispatch scheduling)
- **Constant data**: model weights (optionally as read-only mapped files)

The IREE VM (Virtual Machine) is a tiny interpreter that drives execution: it manages the command buffer building, timepoint tracking, and variant selection. The VM overhead is negligible compared to kernel execution time.

```python
# Inspect a .vmfb file
import iree.runtime as ireert

config = ireert.Config("cuda")
with open("model.vmfb", "rb") as f:
    vmfb_data = f.read()

vm_module = ireert.VmModule.copy_buffer(config.vm_instance, vmfb_data)
ctx = ireert.SystemContext(config=config)
ctx.add_vm_module(vm_module)

# List available functions
print(ctx.modules.module.keys())
```

---

## 162.7 Python API

### 162.7.1 Compilation

```python
import iree.compiler as ireec

# From StableHLO text
mlir_text = """
func.func @matmul(%a: tensor<4x8xf32>, %b: tensor<8x4xf32>) -> tensor<4x4xf32> {
  %empty = tensor.empty() : tensor<4x4xf32>
  %result = linalg.matmul ins(%a, %b : tensor<4x8xf32>, tensor<8x4xf32>)
                          outs(%empty : tensor<4x4xf32>) -> tensor<4x4xf32>
  return %result : tensor<4x4xf32>
}
"""

# Compile to CUDA
flatbuffer = ireec.compile_str(
    mlir_text,
    input_type="linalg",
    target_backends=["cuda"],
    extra_args=["--iree-cuda-target-chip=sm_89"])

# Compile to multiple targets (CPU fallback)
flatbuffer_multi = ireec.compile_str(
    mlir_text,
    input_type="linalg",
    target_backends=["cuda", "llvm-cpu"])

# From file
with open("model.mlir") as f:
    flatbuffer = ireec.compile_str(f.read(),
        input_type="stablehlo",
        target_backends=["cuda"])
```

### 162.7.2 Runtime Execution

```python
import iree.runtime as ireert
import numpy as np

# Configure for CUDA
config = ireert.Config("cuda")
ctx = ireert.SystemContext(config=config)

# Load compiled model
vm_module = ireert.VmModule.copy_buffer(config.vm_instance, flatbuffer)
ctx.add_vm_module(vm_module)

# Get callable function
matmul_fn = ctx.modules.module["matmul"]

# Prepare inputs
a = np.random.randn(4, 8).astype(np.float32)
b = np.random.randn(8, 4).astype(np.float32)

# Execute
result = matmul_fn(a, b)
print(result.to_host())  # NumPy array, shape (4, 4)

# Async execution
future = matmul_fn.async_call(a, b)
result = future.get()
```

### 162.7.3 Compiler from MLIR Python Bindings

```python
from iree.compiler import ir as iree_ir
from iree.compiler.dialects import flow as flow_d
from iree.compiler import compile_module

import mlir.ir as mlir_ir

with mlir_ir.Context() as ctx:
    flow_d.register_dialect(ctx)
    module = mlir_ir.Module.parse(mlir_text)
    
    # Use IREE's compiler API directly
    flatbuffer = compile_module(module, target_backends=["cuda"])
```

---

## 162.8 Performance Engineering in IREE

### 162.8.1 Tuning Dispatch Tiling

IREE's performance relies on tile sizes for its GEMM dispatches. These are configurable via compilation flags or MLIR transform scripts:

```bash
iree-compile \
    --iree-llvmcpu-enable-pad-matmul-to-block-dimensions \
    --iree-llvmcpu-vector-width=512 \  # AVX-512
    --iree-cuda-llvm-target-arch=sm_89 \
    --iree-flow-enable-fuse-padding-into-consumer-ops \
    model.mlir -o model.vmfb
```

### 162.8.2 Weight Bundling

IREE supports pre-compiled weight bundling for faster startup:

```python
# Compile with weights embedded (slower first load, faster startup)
flatbuffer = ireec.compile_str(
    mlir_text,
    target_backends=["llvm-cpu"],
    extra_args=["--iree-hal-executable-constant-mode=device-store"])
```

### 162.8.3 Benchmarking

IREE ships with `iree-benchmark-module` for systematic benchmarking:

```bash
iree-benchmark-module \
    --module=model.vmfb \
    --device=cuda \
    --function=forward \
    --input=32x784xf32 \
    --benchmark_format=json
```

---

## 162.9 Comparing IREE with Alternatives

| Feature | IREE | TensorRT | XLA | ONNX Runtime |
|---------|------|----------|-----|--------------|
| Multi-target | CPU/GPU/Vulkan/Metal | NVIDIA only | CPU/GPU/TPU | CPU/GPU |
| Open source | Yes (Apache 2) | No | Yes (Apache 2) | Yes (MIT) |
| MLIR-based | Fully | No | Partially | No |
| Portable binary | `.vmfb` | `.plan` | N/A | `.ort` |
| Quantization | int8, fp8 | int8, fp8 | int8 | int8 |
| Mobile/embedded | Yes | No | Limited | Yes |
| Web (WebGPU) | Yes (via Vulkan) | No | No | Limited |

IREE's primary advantages are portability and the MLIR foundation. TensorRT outperforms IREE on large NVIDIA-specific workloads (larger GEMM tiles, cuDNN fusion). IREE is competitive for int8 inference on mobile and embedded targets.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **IREE 3.x HAL abstraction unification**: ongoing work to merge the `hal.fence` and `hal.timepoint` APIs into a single semaphore-based synchronization primitive, tracked in the IREE GitHub milestone for HAL v2 cleanup; reduces VM overhead for multi-queue scheduling on CUDA and Vulkan.
- **WebGPU / Dawn backend stabilization**: IREE's `webgpu-spirv` target (replacing the wgpu-based path) is under active review; the Dawn C API bindings in `iree/hal/drivers/webgpu/` are being hardened for production browser deployment via Emscripten, tracking [openxla/iree#18342](https://github.com/openxla/iree/issues/18342).
- **`iree-turbine` integration for PyTorch 2.x export**: the `iree-turbine` frontend (successor to `torch-iree`) is landing direct `torch.export`→StableHLO→IREE compilation without an intermediate ONNX step; expected in the `iree-turbine` 3.x release aligned with PyTorch 2.6.
- **Codegen tuning database (TuningSpec)**: IREE is formalizing a per-target tile-size database (`iree-codegen-tuning-spec`) stored as MLIR transform scripts alongside `.vmfb` artifacts, enabling offline autotuning results to be reused without recompilation, as described in the RFC at discourse.llvm.org/t/iree-tuning-spec-rfc.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Unified MLIR dispatch IR across OpenXLA projects**: the OpenXLA working group is converging IREE's `flow`/`stream` dialects with StableHLO's partitioning IR so that IREE can serve as the single deployment backend for both JAX (XLA) and PyTorch (torch-mlir) without dialect translation overhead; requires stabilizing the `sdy` (Shardy) mesh-sharding dialect integration.
- **IREE for LLM attention kernels — FlashAttention via transform dialect**: integrating `iree-linalg-ext.attention` with codegen strategies matching FlashAttention-3's warp-specialization pattern on H100; involves extending the `gpu.subgroup_mma` abstraction to express TMA (Tensor Memory Accelerator) loads in the NVGPU dialect within IREE's pipeline.
- **ROCm/HIP backend parity with CUDA**: the `rocm` HAL driver is scheduled to reach feature parity with the `cuda` driver (including ROCDL dialect support for CDNA3/RDNA4 MFMAs and stream-ordered memory allocation) tracked in the OpenXLA roadmap for AMD partnership deliverables.
- **Ahead-of-time (AOT) weight sharding for distributed inference**: IREE is developing a multi-device collective lowering path where `flow.channel` ops lower to NCCL/RCCL calls embedded directly in the `.vmfb`, enabling single-file deployment of tensor-parallel LLM inference across device meshes without a separate orchestration layer.

### 5-Year Horizon (Long-Term, by ~2031)

- **IREE as a standards-track W3C WebNN backend**: as WebNN (Web Neural Network API) matures toward CR status, IREE's WebGPU backend is a leading candidate to serve as the reference implementation, necessitating formal conformance testing infrastructure and a stable ABI between IREE's runtime VM and browser JavaScript engines.
- **Verified compilation for safety-critical ML deployment**: convergence of IREE's lowering pipeline with Coq/Lean-based dialect semantics (building on Vellvm and Alive2 methodology) to produce end-to-end correctness proofs for quantized int8 matmul dispatch chains, targeting ISO 26262 ASIL-D certification workflows for automotive ML SoCs.
- **Hardware-retargetable codegen via open ISA descriptors**: replacing IREE's hand-coded target backends with a declarative ISA description layer (extending LLVM's `MLIR-TableGen` backend infrastructure) so that new accelerator targets (e.g., RISC-V Vector 1.0, Arm Scalable Matrix Extension 2) can be added by providing a TableGen `.td` file rather than a full new HAL driver.

---

## Chapter Summary

- IREE is an end-to-end MLIR-based deployment compiler; all passes and IR are MLIR dialects from input to binary.
- The `flow` dialect identifies dispatch regions (ops to fuse into one kernel) via `flow.dispatch.region`; `flow.dispatch.tensor` ops handle tensor protocol.
- The `stream` dialect models async GPU execution via `stream.cmd.execute` / `stream.timepoint`; resource types (`constant`, `transient`, `variable`) drive memory pool assignment.
- The `hal` dialect provides hardware-neutral buffer management and command buffer recording; `hal.executable` supports multi-variant binaries for different target backends.
- Target backends: `llvm-cpu` (native), `cuda` (PTX), `vulkan-spirv` (portable GPU/Web), `metal-spirv` (Apple), `rocm`.
- `.vmfb` FlatBuffer is IREE's portable binary format; it embeds compiled kernels, VM bytecode, and constant weights.
- `iree.compiler.compile_str()` and `iree.runtime.SystemContext` are the primary Python APIs.
- IREE is competitive with TensorRT for int8 inference and outperforms it for portable/embedded/web targets; its MLIR foundation enables rapid addition of new hardware backends.


---

@copyright jreuben11
