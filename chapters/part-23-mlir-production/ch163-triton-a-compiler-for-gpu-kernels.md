# Chapter 163 — Triton: A Compiler for GPU Kernels

*Part XXIII — MLIR in Production*

Triton is an MLIR-based compiler that lets GPU kernels be written in Python-like syntax at the block level, rather than at the thread level of CUDA C. Since its integration into PyTorch 2.0 as the codegen backend for `torch.compile`, Triton has become the de facto kernel authoring tool for ML researchers. XLA uses Triton for GEMM kernels (Chapter 156), FlashAttention is written in Triton, and many custom GPU operators in the ML ecosystem have been reimplemented in Triton. This chapter covers Triton's programming model, its MLIR-based compilation pipeline, the `triton` and `triton_gpu` dialects, the blocked encoding abstraction, autotuning, and the low-level details of the NVPTX and AMDGPU targets.

---

## 163.1 The Triton Programming Model

Triton's key insight is that GPU kernel performance is primarily determined by how data is tiled across the thread hierarchy, not by individual thread operations. Triton exposes **tile-level programming**: a program operates on 1D, 2D, or 3D blocks of data, and Triton's compiler determines the mapping to warps and threads.

### 163.1.1 Kernel Structure

```python
import triton
import triton.language as tl

@triton.jit
def matmul_kernel(
    a_ptr, b_ptr, c_ptr,
    M, N, K,
    stride_am, stride_ak,
    stride_bk, stride_bn,
    stride_cm, stride_cn,
    BLOCK_M: tl.constexpr,
    BLOCK_N: tl.constexpr,
    BLOCK_K: tl.constexpr,
):
    # Program ID: which output tile is this instance computing?
    pid_m = tl.program_id(0)
    pid_n = tl.program_id(1)
    
    # Offset arrays for the M and N tile dimensions
    offs_m = pid_m * BLOCK_M + tl.arange(0, BLOCK_M)
    offs_n = pid_n * BLOCK_N + tl.arange(0, BLOCK_N)
    offs_k = tl.arange(0, BLOCK_K)
    
    # Compute pointers for the current tile
    a_ptrs = a_ptr + (offs_m[:, None] * stride_am + offs_k[None, :] * stride_ak)
    b_ptrs = b_ptr + (offs_k[:, None] * stride_bk + offs_n[None, :] * stride_bn)
    
    # Accumulator: [BLOCK_M, BLOCK_N] block initialized to zero
    acc = tl.zeros((BLOCK_M, BLOCK_N), dtype=tl.float32)
    
    # Iterate over K dimension in BLOCK_K chunks
    for k in range(0, tl.cdiv(K, BLOCK_K)):
        # Load tiles from A and B with boundary masking
        a = tl.load(a_ptrs, mask=(offs_m[:, None] < M) & (offs_k[None, :] < K - k * BLOCK_K))
        b = tl.load(b_ptrs, mask=(offs_k[:, None] < K - k * BLOCK_K) & (offs_n[None, :] < N))
        
        # Matrix multiply-accumulate
        acc = tl.dot(a, b, acc)
        
        # Advance pointers
        a_ptrs += BLOCK_K * stride_ak
        b_ptrs += BLOCK_K * stride_bk
    
    # Store result
    c_ptrs = c_ptr + stride_cm * offs_m[:, None] + stride_cn * offs_n[None, :]
    c_mask = (offs_m[:, None] < M) & (offs_n[None, :] < N)
    tl.store(c_ptrs, acc, mask=c_mask)
```

The key elements:
- `tl.program_id(axis)`: which tile in the grid this thread block is computing (analogous to `blockIdx.x`)
- `tl.arange(0, BLOCK)`: a 1D block of indices `[0, 1, ..., BLOCK-1]`
- `tl.load(ptrs, mask)`: load a block of values with safe OOB masking
- `tl.dot(a, b, acc)`: compute the 2D matrix product `acc += a @ b` using tensor cores
- `tl.constexpr`: compile-time constant (becomes a template parameter in the IR)

### 163.1.2 Launching the Kernel

```python
import torch

def matmul(a, b):
    M, K = a.shape
    K, N = b.shape
    c = torch.empty((M, N), device='cuda', dtype=torch.float32)
    
    # Tile sizes (configurable for autotuning)
    BLOCK_M, BLOCK_N, BLOCK_K = 128, 128, 32
    
    # Grid: one block per output tile
    grid = (triton.cdiv(M, BLOCK_M), triton.cdiv(N, BLOCK_N))
    
    matmul_kernel[grid](
        a, b, c,
        M, N, K,
        a.stride(0), a.stride(1),
        b.stride(0), b.stride(1),
        c.stride(0), c.stride(1),
        BLOCK_M=BLOCK_M, BLOCK_N=BLOCK_N, BLOCK_K=BLOCK_K,
    )
    return c
```

---

## 163.2 Triton's MLIR Compilation Pipeline

`@triton.jit` compiles Python source to GPU machine code through an MLIR pipeline:

```
Python AST (the @triton.jit function)
    │ Triton's Python AST frontend (triton/compiler/compiler.py)
    ▼
Triton dialect IR (triton::FuncOp, triton::LoadOp, triton::DotOp, ...)
    │ TritonToTritonGPU pass (add GPU-specific annotations)
    ▼
TritonGPU dialect IR (triton_gpu::*)
    │ Optimization passes (pipeline, prefetch, coalesce)
    ▼
Optimized TritonGPU IR
    │ TritonGPUToLLVM pass
    ▼
LLVM dialect + NVVM/ROCDL intrinsics
    │ mlir-translate --mlir-to-llvmir
    ▼
LLVM IR (nvptx64 or amdgcn)
    │ ptxas / hipcc
    ▼
cubin / hsaco
```

---

## 163.3 The Triton Dialect

The `triton` dialect ([triton/include/triton/Dialect/Triton/IR/](https://github.com/triton-lang/triton/tree/main/include/triton/Dialect/Triton/IR)) is the high-level representation:

### 163.3.1 Pointer Types and Tensor Operations

Triton uses **pointer tensors** — tensors where each element is a device memory address:

```mlir
// Triton dialect IR (simplified)
tt.func @matmul_kernel(%a_ptr: !tt.ptr<f32>, %b_ptr: !tt.ptr<f32>,
                        %M: i32, %N: i32, %K: i32, ...) {
  %pid_m = tt.get_program_id x : i32
  %pid_n = tt.get_program_id y : i32
  
  // Compute pointer offsets as tensor of pointers
  %offs_m = tt.make_range {start = 0 : i32, end = 128 : i32} : tensor<128xi32>
  %offs_k = tt.make_range {start = 0 : i32, end = 32 : i32} : tensor<32xi32>
  
  // Broadcast and add to get 2D pointer array
  %a_ptrs = tt.addptr %a_base, %offsets : !tt.ptr<f32>, tensor<128x32xi64>
  
  // Load a block of values
  %a_block = tt.load %a_ptrs : tensor<128x32x!tt.ptr<f32>> -> tensor<128x32xf32>
  
  // Matrix multiply-accumulate using tensor cores
  %acc = tt.dot %a_block, %b_block, %acc_prev
      : tensor<128x32xf32>, tensor<32x128xf32>, tensor<128x128xf32>
  
  // Store result
  tt.store %c_ptrs, %acc : tensor<128x128x!tt.ptr<f32>>, tensor<128x128xf32>
}
```

Key triton dialect ops:

| Op | Semantics |
|----|-----------|
| `tt.get_program_id` | Grid coordinate (like `blockIdx`) |
| `tt.make_range` | Create range tensor `[start, end)` |
| `tt.splat` | Broadcast scalar to tensor |
| `tt.addptr` | Add integer offset to pointer tensor |
| `tt.load` | Block load from pointer tensor |
| `tt.store` | Block store to pointer tensor |
| `tt.dot` | Matrix multiply-accumulate (uses MMA) |
| `tt.atomic_add` | Atomic add to pointer tensor |
| `tt.reduce` | Reduction across a dimension |
| `tt.scan` | Prefix scan across a dimension |
| `tt.trans` | 2D matrix transpose |

---

## 163.4 The TritonGPU Dialect

After `TritonToTritonGPU`, each value has an **encoding** that describes how the tensor is distributed across threads:

### 163.4.1 Blocked Encoding

The most common encoding for element-wise ops:

```mlir
// Blocked encoding: how one tensor's elements are distributed
// sizePerThread=[4, 1] threadsPerWarp=[8, 4] warpsPerCTA=[4, 1] CTAsPerCGA=[1, 1]
#blocked = #triton_gpu.blocked<{
  sizePerThread = [4, 1],       // each thread owns 4 elements in dim0, 1 in dim1
  threadsPerWarp = [8, 4],      // 8 threads span dim0, 4 span dim1 (32 threads/warp)
  warpsPerCTA = [4, 1],         // 4 warps in dim0, 1 in dim1 (4 warps/block)
  CTAsPerCGA = [1, 1]           // 1 CTA in the cluster
}>

// A 128x32 tensor with blocked encoding:
// - Each warp handles 32x32 elements (8*4 threads * 4*1 elements = 32 in dim0, 4*1*... = 4... not shown)
// - Total: 4 warps * 32 threads * [4,1] = [512, 32] covered → matches 128x32 with 4x1 warps
%a : tensor<128x32xf32, #blocked>
```

### 163.4.2 MMA Encoding

For tensor core operations, values have MMA encoding that matches the warp-level matrix layout:

```mlir
#mma = #triton_gpu.nvidia_mma<{
  versionMajor = 3,     // Hopper WGMMA
  versionMinor = 0,
  warpsPerCTA = [4, 1],
  instrShape = [16, 256, 16]  // 16x256 output, 16 K per instruction
}>
%acc : tensor<128x128xf32, #mma>
```

The MMA encoding aligns with NVIDIA's warp-level matrix layout so that `triton_gpu.warp_reduce` and result stores don't require extra shuffles.

### 163.4.3 Shared Memory Encoding

Tiles loaded into shared memory use a swizzled encoding to avoid bank conflicts:

```mlir
#shared = #triton_gpu.shared<{
  vec = 8,           // vector load width (float4 = 4 elements, but using 8 here)
  perPhase = 1,      // swizzle phase
  maxPhase = 8,      // swizzle period
  order = [1, 0]     // column-major in shared memory
}>
%a_shared : tensor<128x32xf16, #shared>
```

### 163.4.4 Layout Conversion

When operands have incompatible encodings (e.g., a blocked-encoded tensor fed into a MMA op), `triton_gpu.convert_layout` inserts an encoding conversion:

```mlir
%a_blocked : tensor<128x32xf16, #blocked>

// Convert blocked → shared (for MMA input)
%a_shared = triton_gpu.convert_layout %a_blocked
    : tensor<128x32xf16, #blocked> -> tensor<128x32xf16, #shared>

// Load from shared into MMA register layout
%a_mma = triton_gpu.local_load %a_shared_alloc
    : !tt.memdesc<128x32xf16, #shared, #triton_gpu.smem> -> tensor<128x32xf16, #dot_op_a>
```

---

## 163.5 Optimization Passes

### 163.5.1 Software Pipelining

`TritonGPUPipelinePass` transforms the K-loop into a software pipeline with `num_stages` stages:

```
// Before pipeline:
for k in range(K / BLOCK_K):
  a = load(a_ptrs)
  b = load(b_ptrs)
  barrier()
  acc = dot(a, b, acc)

// After pipeline (num_stages=3):
// Prefetch stages 0,1 before loop
a_0 = async_load(a_ptrs + 0)
b_0 = async_load(b_ptrs + 0)
a_1 = async_load(a_ptrs + 1)
b_1 = async_load(b_ptrs + 1)
async_wait(num_stages-2)  // wait for stage 0

for k in range(K / BLOCK_K):
  // Issue prefetch for k+2
  a_next = async_load(a_ptrs + k+2)
  b_next = async_load(b_ptrs + k+2)
  
  async_wait(num_stages-2)   // wait for k
  acc = dot(a_k, b_k, acc)   // compute with k data
```

This hides global memory latency by overlapping compute with data prefetching.

### 163.5.2 Memory Coalescing

`CoalescePass` rearranges how threads access global memory to ensure 128-byte coalesced reads. If the initial pointer computation results in uncoalesced access, the pass rewrites the pointer arithmetic.

### 163.5.3 Shared Memory Prefetching

For Ampere (sm_80+), `PrefetchPass` converts synchronous `shared_load` → `cp.async` (GPU hardware async copy):

```mlir
// Before: synchronous load to shared memory
triton_gpu.insert_slice_async %a_ptrs, %a_shared_alloc[%k_coord]
    : tensor<BLOCK_M x BLOCK_K x !tt.ptr<f16>> -> ...

// After: cp.async instruction
nvgpu.device_async_copy(%a_gptr, %a_smem_slice) {bypassL1} : ...
nvgpu.device_async_create_group
// ... later ...
nvgpu.device_async_wait {num = 0 : i32}
```

---

## 163.6 Autotuning

Triton's autotuner (`triton.autotune`) benchmarks a set of configurations and caches the best:

```python
@triton.autotune(
    configs=[
        triton.Config({'BLOCK_M': 128, 'BLOCK_N': 256, 'BLOCK_K': 64,
                       'SPLIT_K': 1, 'num_stages': 3, 'num_warps': 8},
                      num_stages=3, num_warps=8),
        triton.Config({'BLOCK_M': 64, 'BLOCK_N': 256, 'BLOCK_K': 32,
                       'SPLIT_K': 1, 'num_stages': 4, 'num_warps': 4},
                      num_stages=4, num_warps=4),
        triton.Config({'BLOCK_M': 128, 'BLOCK_N': 128, 'BLOCK_K': 32,
                       'SPLIT_K': 1, 'num_stages': 4, 'num_warps': 4},
                      num_stages=4, num_warps=4),
        # ... more configs ...
    ],
    key=['M', 'N', 'K'],  # cache key: re-tune when these change
    warmup=25, rep=100)   # warmup and timing reps
@triton.jit
def matmul_kernel(a_ptr, b_ptr, c_ptr, M, N, K, ...):
    ...
```

### 163.6.1 Autotuning Internals

The autotuner:
1. Compiles each config (each is a separate MLIR compile with different `constexpr` values)
2. Runs each compiled kernel `warmup` times to warm the GPU pipeline
3. Measures execution time over `rep` iterations
4. Caches the winner in `~/.triton/cache/` keyed by kernel hash + config key
5. On subsequent calls with the same key, uses the cached winner

### 163.6.2 SPLIT_K Optimization

For small-batch GEMMs where M and N are small but K is large, `SPLIT_K > 1` splits the K dimension across multiple thread blocks:

```
// SPLIT_K=4: each block computes K/4 iterations, then atomic-adds to output
%partial_acc = accumulate K/SPLIT_K iterations
tt.atomic_add %c_ptr, %partial_acc
```

This increases GPU utilization when the output matrix is too small to fill the GPU with the standard grid.

---

## 163.7 FlashAttention in Triton

FlashAttention ([Dao et al., 2022]) is the canonical example of a complex algorithm written in Triton:

```python
@triton.jit
def flash_attention_kernel(
    q_ptr, k_ptr, v_ptr, out_ptr,
    seq_len, head_dim, scale,
    BLOCK_Q: tl.constexpr,
    BLOCK_KV: tl.constexpr,
):
    # Block coordinates
    query_block = tl.program_id(0)
    head = tl.program_id(1)
    
    # Initialize running max and sum for numerically stable softmax
    m_i = tl.full((BLOCK_Q,), float('-inf'), dtype=tl.float32)
    l_i = tl.zeros((BLOCK_Q,), dtype=tl.float32)
    acc = tl.zeros((BLOCK_Q, head_dim), dtype=tl.float32)
    
    # Load Q block [BLOCK_Q, head_dim]
    q = tl.load(q_ptrs)
    
    # Iterate over K,V blocks
    for kv_block in range(tl.cdiv(seq_len, BLOCK_KV)):
        k = tl.load(k_ptrs)
        v = tl.load(v_ptrs)
        
        # Attention scores: [BLOCK_Q, BLOCK_KV] = Q @ K^T * scale
        s = tl.dot(q, tl.trans(k)) * scale
        
        # Online softmax update
        m_ij = tl.max(s, axis=1)
        m_new = tl.maximum(m_i, m_ij)
        alpha = tl.exp(m_i - m_new)
        beta = tl.exp(m_ij - m_new)
        
        # Update running sum and accumulator
        l_i = alpha * l_i + beta * tl.sum(tl.exp(s - m_ij[:, None]), axis=1)
        acc = alpha[:, None] * acc + tl.dot(tl.exp(s - m_ij[:, None]), v)
        m_i = m_new
        
        k_ptrs += BLOCK_KV * head_dim
        v_ptrs += BLOCK_KV * head_dim
    
    # Normalize
    acc = acc / l_i[:, None]
    tl.store(out_ptrs, acc)
```

FlashAttention's O(N) memory complexity (compared to O(N²) for naive attention) comes from the online softmax computation that avoids materializing the full N×N attention matrix. Triton's shared memory and pipelining make this kernel highly competitive with cuDNN's hand-optimized attention.

---

## 163.8 XLA Integration

Chapter 156 covers XLA's Triton integration. The key technical detail is how XLA drives Triton's MLIR compiler:

```cpp
// xla/service/gpu/triton_fusion_analysis.cc
StatusOr<std::string> TritonFusionEmitter::EmitKernel(
    const HloFusionInstruction& fusion,
    GpuVersion gpu_version) {
  
  // 1. Build Triton dialect IR from HLO
  mlir::MLIRContext ctx;
  LoadTritonDialects(ctx);
  
  auto module = GenerateTritonModule(fusion, config_, &ctx);
  
  // 2. Run Triton's compilation pipeline
  mlir::PassManager pm(&ctx);
  pm.addPass(triton::createConvertTritonToTritonGPUPass(
      config_.num_warps, kThreadsPerWarp, config_.num_ctas, gpu_version.cc));
  pm.addPass(triton::gpu::createTritonGPUPipelinePass(config_.num_stages));
  pm.addPass(triton::gpu::createTritonGPUPrefetchPass());
  pm.addPass(triton::createConvertTritonGPUToLLVMPass(gpu_version.cc));
  pm.addPass(mlir::createConvertNVVMToLLVMPass());
  pm.run(*module);
  
  // 3. Translate to LLVM IR and compile to PTX
  auto llvmModule = mlir::translateModuleToLLVMIR(*module, llvmCtx);
  return CompileToPtx(std::move(llvmModule), gpu_version);
}
```

---

## 163.9 PyTorch 2.0 Torch.compile Integration

Triton is the default codegen backend for `torch.compile` (PyTorch 2.0+). When `torch.compile` encounters a pointwise or reduction operation, Inductor (PyTorch's ML compiler) generates Triton kernels:

```python
import torch

@torch.compile
def fused_bias_gelu(x, bias):
    x = x + bias
    return x * 0.5 * (1.0 + torch.erf(x / 1.4142135623730951))

# First call: generates and caches Triton kernel
y = fused_bias_gelu(torch.randn(1024, 4096, device='cuda'),
                    torch.randn(4096, device='cuda'))

# Inspect generated Triton code
torch._inductor.config.debug = True
# Writes generated .py files to /tmp/torchinductor_*/
```

Inductor generates Triton kernels with `@triton.jit` decorators for:
- Pointwise operations (element-wise)
- Reductions (sum, max, mean)
- Persistent reductions (for small dimensions)
- Template-based GEMM (wraps Triton's matmul templates)

---

## Chapter Summary

- Triton's programming model is block-level: programs operate on tiles, not individual threads; `tl.program_id()` identifies which tile a block computes.
- `tl.load`/`tl.store` operate on pointer tensors with masking; `tl.dot` uses tensor cores; `tl.reduce` and `tl.scan` handle reductions and prefix scans.
- The compilation pipeline: Python AST → Triton dialect → TritonGPU dialect (with encodings) → LLVM dialect → PTX/HSACO.
- TritonGPU encodings (`blocked`, `mma`, `shared`) specify how tensor elements are distributed across warps; `triton_gpu.convert_layout` bridges incompatible encodings.
- Software pipelining (`TritonGPUPipelinePass`) overlaps K-loop iterations with prefetching; `num_stages` controls pipeline depth.
- `triton.autotune` benchmarks a set of `(BLOCK_M, BLOCK_N, BLOCK_K, num_stages, num_warps)` configs and caches the winner in `~/.triton/cache/`.
- FlashAttention's O(N) attention kernel is the canonical demonstration of Triton's expressive power; SPLIT_K optimizes small-batch GEMMs.
- PyTorch 2.0's `torch.compile` uses Triton as the default GPU codegen backend through the Inductor compiler.
