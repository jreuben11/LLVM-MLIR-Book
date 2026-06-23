# Chapter 156 — XLA:GPU

*Part XXII — XLA and the OpenXLA Stack*

XLA's GPU backend is where the majority of production ML training runs. It targets NVIDIA GPUs through NVPTX and AMD GPUs through AMDGPU, both via LLVM. The backend's central challenge is the same one that has driven GPU compiler research for two decades: transforming a high-level computation graph into GPU kernels that saturate memory bandwidth and arithmetic throughput. XLA's solution combines an aggressive fusion model, a Triton-based GEMM path for high-performance matrix operations, profile-guided algorithm selection, and a stream-based asynchronous execution model. This chapter covers the GPU codegen pipeline in detail, including the fusion analysis, the Triton integration, collective communication support, and the `GpuExecutable` runtime.

---

## Table of Contents

- [156.1 GPU Backend Architecture](#1561-gpu-backend-architecture)
- [156.2 GPU-Specific HLO Passes](#1562-gpu-specific-hlo-passes)
  - [156.2.1 GpuLayoutAssignment](#15621-gpulayoutassignment)
  - [156.2.2 TritonGemmRewriter](#15622-tritongemmrewriter)
  - [156.2.3 GPU Fusion Analysis](#15623-gpu-fusion-analysis)
- [156.3 GPU Kernel Emission](#1563-gpu-kernel-emission)
  - [156.3.1 GpuIrEmitter](#15631-gpuiremitter)
  - [156.3.2 Reduction Kernels](#15632-reduction-kernels)
  - [156.3.3 Row Reduction vs Column Reduction](#15633-row-reduction-vs-column-reduction)
- [156.4 Triton Integration](#1564-triton-integration)
  - [156.4.1 TritonFusionEmitter](#15641-tritonfusionemitter)
  - [156.4.2 Tiling Parameters](#15642-tiling-parameters)
  - [156.4.3 Sparse Dot (Flash Attention)](#15643-sparse-dot-flash-attention)
- [156.5 Auto-Tuning](#1565-auto-tuning)
  - [156.5.1 GEMM Auto-Tuning](#15651-gemm-auto-tuning)
  - [156.5.2 cuBLAS Algorithm Selection](#15652-cublas-algorithm-selection)
  - [156.5.3 cuDNN Convolution Selection](#15653-cudnn-convolution-selection)
- [156.6 Memory Management: StreamExecutor](#1566-memory-management-streamexecutor)
- [156.7 Collective Operations and NCCL](#1567-collective-operations-and-nccl)
  - [156.7.1 AllReduce Thunk](#15671-allreduce-thunk)
  - [156.7.2 Communication Scheduling](#15672-communication-scheduling)
- [156.8 GpuExecutable and the Runtime](#1568-gpuexecutable-and-the-runtime)
  - [156.8.1 Execution Flow](#15681-execution-flow)
  - [156.8.2 Kernel Cache](#15682-kernel-cache)
- [156.9 ROCm Support](#1569-rocm-support)
- [156.10 Performance Considerations](#15610-performance-considerations)
  - [156.10.1 Occupancy](#156101-occupancy)
  - [156.10.2 Memory Coalescing](#156102-memory-coalescing)
  - [156.10.3 Shared Memory Tiling](#156103-shared-memory-tiling)
  - [156.10.4 Bank Conflicts](#156104-bank-conflicts)
- [Chapter Summary](#chapter-summary)

---

## 156.1 GPU Backend Architecture

The GPU backend lives in [`xla/backends/gpu/`](https://github.com/openxla/xla/tree/main/xla/backends/gpu) with the core compilation infrastructure in `xla/service/gpu/`. The pipeline:

```
Optimized HloModule (from common passes)
    │
    ▼
GPU-specific HLO passes
    ├── GpuLayoutAssignment       — column-major for cuBLAS compat
    ├── TritonGemmRewriter        — kDot → kCustomCall[triton_gemm]
    ├── GpuFusionRewriter         — annotate fusion kinds
    ├── InstructionFusion         — element-wise → loop fusion
    ├── FusionMerger              — merge adjacent fusions
    ├── GemmAlgorithmPicker       — auto-tune cuBLAS GEMM algorithms
    └── ConvAlgorithmPicker       — auto-tune cuDNN conv algorithms
    │
    ▼
Thunk emitter (HloModule → thunk sequence)
    ├── KernelThunk → GpuIrEmitter → LLVM IR
    ├── GemmThunk → cuBLAS call
    ├── TritonFusionThunk → Triton compiler → PTX
    ├── AllReduceThunk → NCCL call
    └── ConvolutionThunk → cuDNN call
    │
    ▼
LLVM NVPTX/AMDGPU backend → PTX/LLVMIR → ptxas/hipcc → cubin/hsaco
    │
    ▼
GpuExecutable (cubin/hsaco + thunk sequence + buffer assignment)
```

---

## 156.2 GPU-Specific HLO Passes

### 156.2.1 GpuLayoutAssignment

The GPU backend prefers column-major (Fortran order) layouts for `kDot` operations to match cuBLAS conventions. `GpuLayoutAssignment` propagates `{0,1}` (column-major) constraints for matrix inputs and outputs of dot products, then inserts `kCopy` (transpose) instructions where layout conflicts arise:

```cpp
// Simplified from xla/service/gpu/gpu_layout_assignment.cc
Status GpuLayoutAssignment::AddBackendConstraintsToDotOperands(
    HloInstruction* dot, LayoutConstraints* constraints) {
  // cuBLAS expects column-major for the rhs (weight matrix)
  constraints->SetOperandLayout(*dot, 1, LayoutUtil::MakeLayout({0, 1}));
  // Output layout determined by cuBLAS output format
  constraints->SetResultLayout(*dot, LayoutUtil::MakeLayout({0, 1}));
}
```

### 156.2.2 TritonGemmRewriter

`TritonGemmRewriter` converts eligible `kDot` instructions to `kCustomCall` with target `"__triton_gemm"`:

```cpp
// Input: standard kDot
%dot = dot(%lhs, %rhs),
    lhs_contracting={1}, rhs_contracting={0}

// Output: custom call to Triton GEMM
%dot = custom-call(%lhs, %rhs),
    custom_call_target="__triton_gemm",
    backend_config={"block_m": 128, "block_n": 128, "block_k": 32,
                    "split_k": 1, "num_stages": 3, "num_warps": 4}
```

The backend config encodes Triton's tiling parameters. These are determined by the auto-tuner (Section 156.5).

### 156.2.3 GPU Fusion Analysis

XLA GPU recognizes several distinct fusion kinds, each mapped to a different codegen path:

| Fusion Kind | Description | Codegen |
|-------------|-------------|---------|
| `kLoop` | Element-wise, same shape | GpuIrEmitter loop kernel |
| `kInput` | Reduction with producer fused in | Reduction kernel |
| `kOutput` | Output can be computed element-wise from operand | GpuIrEmitter |
| `kTriton` | Triton-based GEMM fusion | Triton compiler |
| `kCustom` | User-registered CustomCall | User-provided kernel |

`GpuFusionRewriter` and `FusionMerger` annotate and merge fusions. The key heuristic: merge two adjacent loop fusions if the merged kernel fits in L1/L2 cache and the merge reduces global memory traffic.

---

## 156.3 GPU Kernel Emission

### 156.3.1 GpuIrEmitter

`GpuIrEmitter` emits LLVM IR for loop and reduction fusion kernels. Unlike the CPU emitter, it must map work to CUDA's thread hierarchy (grid, block, thread):

```cpp
// For a loop fusion with shape [N]:
// Launch: grid_dim=(N/256, 1, 1), block_dim=(256, 1, 1)
// Each thread handles one element

// LLVM IR structure emitted:
define void @fusion_kernel(ptr %out, ptr %in0, ptr %in1, i64 %n) {
entry:
  ; tid = blockIdx.x * blockDim.x + threadIdx.x
  %tid = ... ; NVVM intrinsics
  %in_bounds = icmp ult i64 %tid, %n
  br i1 %in_bounds, label %body, label %exit
body:
  ; Load inputs, compute, store output
  %a = load float, ptr getelementptr(float, %in0, i64 %tid)
  %b = load float, ptr getelementptr(float, %in1, i64 %tid)
  %result = fadd float %a, %b
  store float %result, ptr getelementptr(float, %out, i64 %tid)
  br label %exit
exit:
  ret void
}
```

The LLVM IR uses NVVM intrinsics (`@llvm.nvvm.read.ptx.sreg.tid.x`, `@llvm.nvvm.read.ptx.sreg.ctaid.x`, etc.) that the NVPTX backend converts to PTX register reads.

### 156.3.2 Reduction Kernels

Reductions require inter-thread communication via shared memory. XLA emits two-phase reduction kernels:

**Phase 1** (per-block partial reduction):
```
Each thread computes a partial sum over a strided range of the input.
Partial sums are stored in shared memory.
__syncthreads() barrier.
```

**Phase 2** (warp-level reduction):
```
Use warp shuffle instructions for the final 32-element reduction:
__shfl_down_sync(0xffffffff, val, offset)
```

For large reductions, XLA uses a two-kernel strategy:
- Kernel 1: N blocks each reduce a chunk → write N partial results
- Kernel 2: One block reduces the N partial results

### 156.3.3 Row Reduction vs Column Reduction

XLA distinguishes row reductions (reduce along last dimension, common in softmax/layer norm) from column reductions (reduce along batch dimension):

- **Row reduction**: use a transposed read pattern where each block handles one row; warp-level reduction along the row dimension
- **Column reduction**: use a blocked pattern where each thread handles multiple rows; reduction is along the minor dimension

---

## 156.4 Triton Integration

Chapter 163 covers Triton in depth; this section covers XLA's specific integration.

### 156.4.1 TritonFusionEmitter

When `GpuExecutable` encounters a `kCustomCall["__triton_gemm"]` thunk, it invokes the Triton compiler:

```cpp
// From xla/service/gpu/triton_fusion_analysis.cc
StatusOr<std::string> TritonFusionEmitter::EmitKernel(
    const HloFusionInstruction& fusion) {
  // Extract tiling parameters from backend config
  TritonGemmConfig config = GetTritonConfig(fusion);

  // Generate Triton IR (as MLIR using Triton dialect)
  mlir::MLIRContext ctx;
  auto module = GenerateTritonGemmModule(fusion, config, &ctx);

  // Compile Triton IR → PTX via Triton's pipeline
  return CompileTritonToPtx(module, gpu_version_);
}
```

The generated Triton IR uses the `triton` and `triton_gpu` MLIR dialects. The Triton compiler then applies its own optimization pipeline (blocked encoding, pipeline prefetching, MMA lowering) before emitting PTX.

### 156.4.2 Tiling Parameters

The key Triton GEMM tiling parameters:

| Parameter | Typical Values | Effect |
|-----------|----------------|--------|
| `BLOCK_M` | 64, 128 | M-dimension tile size per thread block |
| `BLOCK_N` | 64, 128 | N-dimension tile size per thread block |
| `BLOCK_K` | 32, 64 | K-dimension tile size (inner loop) |
| `SPLIT_K` | 1, 2, 4 | K-splitting for small-batch GEMMs |
| `NUM_STAGES` | 2, 3, 4 | Software pipeline depth |
| `NUM_WARPS` | 4, 8 | Warps per thread block |

For an `[M=256, K=1024] x [K=1024, N=512]` GEMM on H100:
- `BLOCK_M=128, BLOCK_N=128, BLOCK_K=64` → each block computes a 128×128 output tile
- `NUM_STAGES=3` → triple-buffered data prefetch
- `NUM_WARPS=8` → 256 CUDA threads per block

### 156.4.3 Sparse Dot (Flash Attention)

XLA also uses Triton for attention kernels. The `FlashAttentionRewriter` replaces the naive attention computation (Q@K.T + softmax + @V) with a `kCustomCall["__triton_flash_attention"]` that fuses the entire attention into a tiled kernel with O(N) memory complexity.

---

## 156.5 Auto-Tuning

XLA GPU implements an auto-tuner that benchmarks multiple kernel configurations to find the fastest one.

### 156.5.1 GEMM Auto-Tuning

For Triton GEMM, the auto-tuner runs through a set of candidate tiling configurations:

```cpp
// From xla/service/gpu/gemm_algorithm_picker.cc
absl::StatusOr<TritonGemmConfig> PickBestTritonConfig(
    const HloInstruction& dot,
    se::StreamExecutor* stream_executor) {
  std::vector<TritonGemmConfig> candidates = GenerateCandidateConfigs(dot);
  
  TritonGemmConfig best;
  int64_t best_time = INT64_MAX;
  for (const auto& config : candidates) {
    // Compile and benchmark this config
    auto kernel = CompileTritonGemm(dot, config);
    int64_t time = BenchmarkKernel(kernel, stream_executor);
    if (time < best_time) {
      best_time = time;
      best = config;
    }
  }
  return best;
}
```

Auto-tuning results are cached in a `PersistentAutotuneCacheProto` keyed by the HLO instruction fingerprint. Subsequent compilations of the same shape use the cached result.

### 156.5.2 cuBLAS Algorithm Selection

For GEMMs that route to cuBLAS (large batch, special data types), `GemmAlgorithmPicker` benchmarks all cuBLAS algorithms and caches the winner:

```cpp
// cuBLAS algorithm IDs are integers 0..N
for (int algo_id = 0; algo_id < num_algorithms; ++algo_id) {
  cublasGemmAlgo_t algo = CUBLAS_GEMM_ALGO0 + algo_id;
  float time = BenchmarkCublasGemm(algo, m, n, k, dtype);
  ...
}
```

### 156.5.3 cuDNN Convolution Selection

`ConvAlgorithmPicker` does the same for convolutions, benchmarking cuDNN's available algorithms (IMPLICIT_GEMM, FFT, WINOGRAD, etc.) and selecting the fastest for the given filter/input shape.

---

## 156.6 Memory Management: StreamExecutor

XLA uses StreamExecutor (part of the XLA repository, originally TensorFlow's device abstraction) to manage GPU memory and command submission:

```cpp
// Allocate device memory
se::StreamExecutor* executor = ...;
se::DeviceMemory<float> buf = executor->AllocateArray<float>(4096);

// Copy host → device
stream->ThenMemcpy(&buf, host_data, 4096 * sizeof(float));

// Launch kernel
stream->ThenLaunch(thread_dim, block_dim, kernel_fn, arg0, arg1, ...);

// Wait for completion
stream->BlockHostUntilDone();
```

`StreamExecutor` abstracts over CUDA and ROCm, providing a uniform interface. The GPU backend uses streams to overlap computation and memory transfers.

---

## 156.7 Collective Operations and NCCL

Multi-GPU training requires collective communication operations (AllReduce, AllGather, etc.). XLA uses NCCL (NVIDIA Collective Communications Library) for CUDA and RCCL for ROCm.

### 156.7.1 AllReduce Thunk

`AllReduceThunk` wraps an NCCL call:

```cpp
class AllReduceThunk : public Thunk {
  absl::Status ExecuteOnStream(const ExecuteParams& params) override {
    // Gather all participating device buffers
    std::vector<DeviceBufferPair> device_buffers = GetBuffers(params);
    
    // Issue NCCL AllReduce
    return RunNcclAllReduce(
        /*op=*/reduction_kind_,     // SUM, PROD, MIN, MAX
        device_buffers,
        params.nccl_params.comm,
        params.stream);
  }
};
```

### 156.7.2 Communication Scheduling

For pipeline-parallel training, XLA supports overlapping compute with communication. The `AsyncAllReduce` transformation converts synchronous `kAllReduce` to an async pair `kAllReduceStart` / `kAllReduceDone`, allowing the next compute kernel to launch before the AllReduce finishes:

```
// Before:
%grad = all-reduce(%partial_grad), replica_groups={{0,1,2,3}}

// After async transformation:
%token = all-reduce-start(%partial_grad), replica_groups={{0,1,2,3}}
%next_layer_grad = ... (compute runs concurrently with AllReduce)
%grad = all-reduce-done(%token)
```

---

## 156.8 GpuExecutable and the Runtime

`GpuExecutable` (in [`xla/service/gpu/gpu_executable.cc`](https://github.com/openxla/xla/blob/main/xla/service/gpu/gpu_executable.cc)) stores:

```cpp
class GpuExecutable : public Executable {
  // GPU binaries: CUDA cubin or ROCm hsaco
  std::vector<uint8_t> binary_;

  // Thunk sequence — the execution plan
  ThunkSequence thunk_sequence_;

  // Loaded GPU kernels (cuFunction handles)
  absl::flat_hash_map<std::string, se::KernelBase*> kernel_cache_;

  // Buffer assignment for all HLO values
  std::unique_ptr<BufferAssignment> buffer_assignment_;

  absl::Status ExecuteThunks(
      const ServiceExecutableRunOptions* run_options,
      const BufferAllocations& buffer_allocations,
      bool block_host_until_done);
};
```

### 156.8.1 Execution Flow

1. XLA loads the cubin/hsaco binary into the CUDA/ROCm driver at `GpuExecutable::Create()` time
2. At execution time, `ExecuteThunks()` dispatches each thunk in order
3. Thunks share the GPU stream; CUDA stream semantics guarantee ordering
4. The host blocks (or uses a callback) to detect completion

### 156.8.2 Kernel Cache

GPU kernels are expensive to load. `GpuExecutable` maintains a `kernel_cache_` that maps kernel names to `se::KernelBase*` objects, avoiding redundant `cuModuleGetFunction` calls on repeated executions.

---

## 156.9 ROCm Support

XLA targets AMD GPUs through LLVM's AMDGPU backend. The ROCm path mirrors the CUDA path:

- `nvptx64` → `amdgcn-amd-amdhsa` target triple
- NVVM intrinsics → ROCDL intrinsics (`@llvm.amdgcn.workitem.id.x`, etc.)
- cuBLAS → rocBLAS; cuDNN → MIOpen; NCCL → RCCL
- PTX → ROCm assembly (GFX ISA)
- `ptxas` → `hipcc` / LLVM/AMDGPU backend

The `IrEmitter` has a dual-path where GPU-specific intrinsics are selected by the target:

```cpp
llvm::Value* IrEmitter::GetThreadId() {
  if (target_ == GpuTarget::kCuda) {
    return b_.CreateCall(
        llvm::Intrinsic::getDeclaration(module_, llvm::Intrinsic::nvvm_read_ptx_sreg_tid_x));
  } else {  // ROCm
    return b_.CreateCall(
        llvm::Intrinsic::getDeclaration(module_, llvm::Intrinsic::amdgcn_workitem_id_x));
  }
}
```

---

## 156.10 Performance Considerations

### 156.10.1 Occupancy

GPU occupancy — the fraction of maximum concurrent threads — is a primary performance metric. Low occupancy indicates wasted GPU resources. XLA tunes thread block sizes to maximize occupancy given shared memory and register constraints. The `GpuHloSchedule` pass reorders HLO instructions to maximize GPU utilization by avoiding idle periods between kernels.

### 156.10.2 Memory Coalescing

Global memory access patterns must be coalesced for maximum bandwidth. XLA's loop fusion emits kernels where thread `tid` accesses `array[tid]`, ensuring the 32 threads in a warp access 32 consecutive elements — fully coalesced. For transpose-like patterns, XLA uses shared memory as a transposition buffer.

### 156.10.3 Shared Memory Tiling

For reduction kernels, XLA allocates shared memory via NVVM's `@shared` address space:

```llvm
; LLVM IR for shared memory allocation
@shared_reduce_buf = addrspace(3) global [256 x float] undef
```

The NVPTX backend converts this to:
```ptx
.shared .f32 shared_reduce_buf[256];
```

### 156.10.4 Bank Conflicts

Shared memory is divided into 32 banks. Concurrent accesses to the same bank are serialized. XLA avoids bank conflicts in transposition kernels by padding the shared memory array:

```cpp
// Pad to avoid bank conflicts: [32+1] instead of [32]
int shared_mem_size = (TILE_DIM + 1) * TILE_DIM * sizeof(float);
```

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **Triton 3.x GEMM backend consolidation**: OpenXLA's migration from the legacy `xla/service/gpu/triton_fusion_emitter.cc` path to the unified `codegen/transforms/` MLIR pipeline is underway; expect the split-K and sparse-dot Flash Attention paths to land in the consolidated backend by mid-2026, tracked in [openxla/xla#20xxx series PRs](https://github.com/openxla/xla/issues).
- **FP8 (E4M3/E5M2) GEMM via cuBLAS LT and Triton on H100/Hopper**: XLA's `GemmAlgorithmPicker` is being extended to handle cuBLAS LT FP8 (GEMM using `cublasLtMatmul` with `CUBLAS_COMPUTE_32F_FAST_TF32`) and the Triton WMMA/MMA lowering for `tt.dot` on FP8 tensors; this enables 2× throughput over BF16 on Ada/Hopper.
- **StreamExecutor removal (GPU plugin API)**: The OpenXLA community is actively replacing `StreamExecutor` with a stable plugin ABI (`xla/stream_executor/gpu/gpu_kernel.h`); framework maintainers are tracking the migration in [openxla/xla#17090](https://github.com/openxla/xla/issues/17090) with target completion in 2026 H1.
- **AMD ROCm 7.x / GFX12 (RDNA 4) support**: LLVM's AMDGPU backend added GFX1200 target support in LLVM 19; XLA's ROCm path needs corresponding `ROCDL` intrinsic updates for new wave64/wave32 MAI instructions and the new CDNA4 memory model; patches expected in XLA 0.5.x alongside ROCm 7.0.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **XLA:GPU → MLIR codegen end-to-end**: The OpenXLA roadmap calls for replacing `GpuIrEmitter` (direct LLVM IR emission) entirely with an MLIR-based pipeline using `gpu`, `nvgpu`, and `vector` dialects, lowering through `nvvm` to PTX — eliminating the hand-rolled NVVM intrinsic injection in `GpuIrEmitter` and aligning with MLIR upstream's `nvgpu::WarpMmaOp` and `gpu::SubgroupMmaOp` for tensor-core targeting.
- **Persistent kernel and CUDA Graph integration**: XLA's thunk-sequenced execution model requires CUDAGraph integration for latency-sensitive inference; the `GpuExecutable` runtime will be extended with graph capture (`cudaStreamBeginCapture`) and replay for static-shape workloads, removing repeated driver API overhead on each execution.
- **Distributed collective scheduling with topology awareness**: The current `AllReduceThunk` dispatches NCCL operations at fixed schedule points; OpenXLA is developing a collective scheduler that models NVLink/PCIe bandwidth topology (via `DeviceAssignment` extensions) to overlap fine-grained ring-allreduce steps with dependent compute; this is related to [google/jax#18455](https://github.com/google/jax/issues/18455) and DeepMind's Pathways communication model.
- **Unified XLA:GPU auto-tuner with learned cost models**: The current benchmarking auto-tuner (Section 156.5) runs expensive kernel compilations at compile time; a learned cost model (ML-based tile-size predictor trained over cubin profiling data) is under development within Google's internal TensorFlow/JAX stack and is expected to be open-sourced in XLA's `autotuning/` namespace, replacing exhaustive benchmarking with model-guided search.

### 5-Year Horizon (Long-Term, by ~2031)

- **Grace-Hopper NVLink-C2C unified memory model**: NVIDIA's Grace-Hopper (NVL72/GB200) exposes CPU+GPU shared memory via NVLink-C2C; XLA's buffer assignment and StreamExecutor memory model will need to be redesigned to exploit unified virtual addressing without explicit `H2D`/`D2H` `cudaMemcpy` boundaries, fundamentally changing how `GpuExecutable` allocates and addresses computation buffers.
- **Post-PTX IR targeting: NVVM IR as stable ABI**: NVIDIA has been signalling that PTX will be supplemented by a stable NVVM IR ABI (`nvvm.annotations` + LLVM IR modules) for AOT compilation across GPU generations; XLA's NVPTX backend may shift to emitting versioned NVVM IR modules rather than PTX text, reducing `ptxas` JIT overhead at driver level.
- **Cross-accelerator heterogeneous compilation (GPU + TPU + spatial)**: OpenXLA's long-term architecture envisions a single HLO module being partitioned across GPU, TPU, and spatial accelerators (e.g., Groq, Cerebras) using a unified `DeviceMesh` abstraction and per-device backends registered via a plugin API; this requires the `GpuExecutable` runtime to participate in heterogeneous thunk scheduling coordinated by the XLA runtime client.

---

## Chapter Summary

- XLA GPU targets NVIDIA (NVPTX) and AMD (AMDGPU) via LLVM; the pipeline runs GPU-specific passes before code emission.
- `GpuLayoutAssignment` sets column-major layouts for cuBLAS compatibility; `TritonGemmRewriter` converts dot products to Triton-based custom calls.
- Fusion kinds: `kLoop` (element-wise), `kInput` (reduction), `kTriton` (GEMM), `kCustom`; each maps to a different codegen path.
- `GpuIrEmitter` emits NVVM-intrinsic LLVM IR for loop/reduction kernels; the NVPTX backend converts this to PTX.
- Triton integration handles high-performance GEMM with configurable tiling (BLOCK_M/N/K, NUM_STAGES, NUM_WARPS); auto-tuning selects the best config.
- Auto-tuning is cached in a persistent proto; cuBLAS and cuDNN algorithm selection uses the same benchmarking approach.
- `AllReduceThunk` wraps NCCL calls; async AllReduce (`kAllReduceStart`/`kAllReduceDone`) enables compute-communication overlap.
- `GpuExecutable` stores the cubin/hsaco binary and a thunk sequence; the kernel cache avoids repeated driver API calls.
- ROCm mirrors the CUDA path using ROCDL intrinsics, rocBLAS, MIOpen, and RCCL.


---

@copyright jreuben11
