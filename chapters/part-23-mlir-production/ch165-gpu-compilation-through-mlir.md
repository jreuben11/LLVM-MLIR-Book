# Chapter 165 — GPU Compilation Through MLIR

*Part XXIII — MLIR in Production*

MLIR provides a complete, modular infrastructure for GPU kernel compilation — from high-level `linalg.generic` operations all the way to PTX or AMD GCN assembly. Unlike CUDA C or Triton, which are framework-specific, MLIR's GPU compilation path is composable: any dialect that can be lowered to `vector.contract` can target any GPU through the same sequence of passes. This chapter traces the complete GPU compilation path, covering tiling, vectorization, mapping to the GPU thread hierarchy, NVVM and ROCDL lowering, async copy patterns, shared memory management, and the practical considerations that determine kernel performance.

---

## 165.1 The End-to-End GPU Compilation Path

The complete pipeline, starting from Linalg:

```
linalg.generic or linalg.matmul (input)
    │
    ▼ Step 1: Tile for L1 cache / warp tiles
scf.for (outer tile loops) + linalg.generic (inner tile)
    │
    ▼ Step 2: Vectorize inner tile
vector.contract + vector.transfer_read/write
    │
    ▼ Step 3: Map to GPU thread hierarchy
gpu.launch → gpu.func with thread/block indexing
    │
    ▼ Step 4: Lower GPU ops to target-specific dialect
nvvm.* (NVIDIA) or rocdl.* (AMD)
    │
    ▼ Step 5: Translate to LLVM IR
mlir-translate --mlir-to-llvmir
    │
    ▼ Step 6: LLVM backend
llc -march=nvptx64 → PTX (NVIDIA) or llc -march=amdgcn → GFX ISA (AMD)
    │
    ▼ Step 7: Assemble to binary
ptxas → cubin (NVIDIA) or hipcc/lld → hsaco (AMD)
    │
    ▼ Embed in host program and launch via CUDA/ROCm runtime
```

---

## 165.2 Step 1: Tiling for the GPU Memory Hierarchy

GPU memory hierarchy:
- **Global memory**: ~80 GB, ~900 GB/s bandwidth, ~400 cycle latency
- **L2 cache**: ~50 MB, shared across all SMs
- **Shared memory (SMEM)**: ~228 KB per SM, ~20 TB/s bandwidth, ~20 cycle latency
- **Registers**: ~256 KB per SM, zero latency, but limited count (typically 255 per thread)

Effective tiling exploits SMEM as a programmable cache.

### 165.2.1 Two-Level Tiling

For a matmul, we tile twice: once for SMEM tiles and once for register tiles:

```mlir
// Input: linalg.matmul [M, K] x [K, N] -> [M, N]
// Tile 1: SMEM tiles (64×64×64)
// Tile 2: Register tiles (8×8×8)

// Step 1a: Tile for SMEM
%tiled_loop, %outer_loops = transform.structured.tile_using_for %matmul
    [64, 64]  // M, N tile sizes; K handled separately
    
// Step 1b: Tile K separately
%k_tiled, %k_loop = transform.structured.tile_using_for %tiled_loop
    [0, 0, 64]  // K tile size (M, N already tiled)
```

After tiling, the inner `linalg.matmul` operates on `[64, 64, 64]` tiles — small enough to fit A and B tiles in SMEM.

### 165.2.2 Packing into Shared Memory

After tiling, operands are explicitly loaded into SMEM:

```mlir
// Before SMEM packing: read directly from global memref
%a_slice = memref.subview %A_global[%i, %k][64, 64][1, 1]
    : memref<M x K x f32> to memref<64 x 64 x f32>

// After SMEM packing:
%a_smem = memref.alloc() : memref<64x64xf32, #gpu.address_space<workgroup>>
linalg.copy ins(%a_slice : memref<64x64xf32>) 
            outs(%a_smem : memref<64x64xf32, #gpu.address_space<workgroup>>)
gpu.barrier  // __syncthreads()
// ... matmul on %a_smem ...
```

The `#gpu.address_space<workgroup>` annotation marks the buffer as SMEM; LLVM's NVPTX backend maps this to `.shared` address space.

---

## 165.3 Step 2: Vectorization

After tiling, `linalg.generic` ops on small tiles are vectorized to `vector.*` ops:

```mlir
// Before vectorize:
linalg.generic {
  indexing_maps = [...], iterator_types = ["parallel", "parallel", "reduction"]}
  ins(%a_tile : memref<8x8xf32>, %b_tile : memref<8x8xf32>)
  outs(%c_tile : memref<8x8xf32>) {
    ^bb(%a: f32, %b: f32, %c: f32):
      %mul = arith.mulf %a, %b : f32
      %add = arith.addf %c, %mul : f32
      linalg.yield %add : f32
  }

// After vectorize (--linalg-vectorize):
%a_vec = vector.transfer_read %a_tile[%c0, %c0], %zero : memref<8x8xf32> -> vector<8x8xf32>
%b_vec = vector.transfer_read %b_tile[%c0, %c0], %zero : memref<8x8xf32> -> vector<8x8xf32>
%c_vec = vector.transfer_read %c_tile[%c0, %c0], %zero : memref<8x8xf32> -> vector<8x8xf32>

// vector.contract: the 2D matrix multiply
%result = vector.contract {
  indexing_maps = [affine_map<(i,j,k) -> (i,k)>,
                   affine_map<(i,j,k) -> (k,j)>,
                   affine_map<(i,j,k) -> (i,j)>],
  iterator_types = ["parallel", "parallel", "reduction"]}
  %a_vec, %b_vec, %c_vec : vector<8x8xf32>, vector<8x8xf32> into vector<8x8xf32>

vector.transfer_write %result, %c_tile[%c0, %c0] : vector<8x8xf32>, memref<8x8xf32>
```

`vector.contract` is the key abstraction — it represents a contracted tensor product that lowers to tensor core instructions or FMA chains depending on the target.

---

## 165.4 Step 3: Mapping to the GPU Thread Hierarchy

The `gpu` dialect pass suite maps tiled, vectorized IR to explicit GPU grid/block/thread indexing.

### 165.4.1 gpu.launch

The outer tile loops (M, N) map to GPU blocks:

```mlir
// Map outer M and N loops to blockIdx.x, blockIdx.y
gpu.launch
    blocks(%bx, %by, %bz) in (%gbx = %grid_x, %gby = %grid_y, %gbz = %one)
    threads(%tx, %ty, %tz) in (%tbx = %block_x, %tby = %block_y, %tbz = %one) {
  
  // %bx = blockIdx.x, %tx = threadIdx.x
  %m_start = arith.muli %bx, %c64 : index  // this block handles M tile at m_start
  %n_start = arith.muli %by, %c64 : index
  
  // Inner tile loop (over K) is sequential within the block
  // Inner register loop (over 8×8 tiles) maps to threads
  ...
  
  gpu.terminator
}
```

### 165.4.2 Thread Indexing for Vectorized Ops

Within a block, threads divide the register-level tile:

```mlir
// 64 threads each handle an 8×8 sub-tile
// Thread (tx, ty) handles the sub-tile at [tx*8, ty*8]
%thread_m = arith.addi %m_start, arith.muli(%tx, %c8)
%thread_n = arith.addi %n_start, arith.muli(%ty, %c8)

// Each thread loads its 8×8 slice of A and B from SMEM
%a_frag = vector.transfer_read %a_smem[%thread_m_local, %k_start]
    : memref<64x64xf32, #gpu.address_space<workgroup>> -> vector<8x8xf32>
```

### 165.4.3 gpu.barrier (Synchronization)

SMEM writes must be followed by a barrier before SMEM reads:

```mlir
// All threads copy global → SMEM
linalg.copy ins(%a_global_tile) outs(%a_smem)
// Barrier: all threads must complete the copy
gpu.barrier
// Now safe to read SMEM
%a_frag = vector.transfer_read %a_smem[...] -> vector<8x8xf32>
```

`gpu.barrier` lowers to `nvvm.barrier0` (PTX `bar.sync 0`) or `rocdl.barrier` (AMD `s_barrier`).

---

## 165.5 Step 4: Lowering to NVVM and ROCDL

### 165.5.1 NVVM Dialect

The `nvvm` dialect wraps PTX intrinsics as MLIR ops:

```mlir
// Thread ID intrinsics
%tid_x = nvvm.read.ptx.sreg.tid.x : i32
%tid_y = nvvm.read.ptx.sreg.tid.y : i32
%bid_x = nvvm.read.ptx.sreg.ctaid.x : i32

// Block dimension
%ntid_x = nvvm.read.ptx.sreg.ntid.x : i32

// Barrier
nvvm.barrier0

// Shared memory fence
nvvm.membar.cta

// Warp shuffle (for warp-level reductions)
%shuffled = nvvm.shfl.sync.bfly %mask, %val, %offset, %width
    : (i32, f32, i32, i32) -> !llvm.struct<(f32, i1)>
```

The conversion pass `ConvertGpuOpsToNVVMOps`:
- `gpu.thread_id x` → `nvvm.read.ptx.sreg.tid.x`
- `gpu.block_id x` → `nvvm.read.ptx.sreg.ctaid.x`
- `gpu.barrier` → `nvvm.barrier0`
- `gpu.shuffle xor` → `nvvm.shfl.sync.bfly`

### 165.5.2 ROCDL Dialect

The `rocdl` dialect provides AMD equivalents:

```mlir
// AMD thread ID
%tid_x = rocdl.workitem.id.x : i32
%bid_x = rocdl.workgroup.id.x : i32

// AMD barrier (LDS synchronization)
rocdl.barrier

// AMD warp shuffle (via DS_SWIZZLE_B32)
%shuffled = rocdl.ds_swizzle %val, %offset : (f32, i32) -> f32
```

The `ConvertGpuOpsToROCDLOps` conversion is symmetric with the NVVM version.

### 165.5.3 Lowering vector.contract

`vector.contract` is lowered to target-specific MMA instructions:

**For NVPTX with WMMA** (Volta/Turing):
```
vector<16x16xf32> contract → nvvm.wmma.mma.sync.*
```

**For NVPTX with MMA** (Ampere):
```
vector<8x4xf16> contracts (8 ops) → nvvm.mma.sync.aligned.m16n8k16.*
```

**For ROCDL**:
```
vector<16x16xf32> contract → rocdl.wmma.*
```

The pass `ConvertVectorToGPU` handles this lowering; it uses target information to select the right instruction shape.

---

## 165.6 Async Copy Patterns

### 165.6.1 Ampere cp.async

On Ampere (sm_80+), the `nvgpu.device_async_copy` op implements hardware async copy:

```mlir
// Issue async copy: global → shared memory
nvgpu.device_async_copy %src_ptr, %dst_ptr, 16
    : memref<?xf32>, memref<64xf32, #gpu.address_space<workgroup>>

// Commit the copy group
nvgpu.device_async_create_group

// Wait for the committed group to complete
nvgpu.device_async_wait {num = 0 : i32}

// Now safe to read from %dst_ptr
```

This lowers to:
```ptx
cp.async.cg.shared.global [%dst], [%src], 16;
cp.async.commit_group;
cp.async.wait_group 0;
```

### 165.6.2 Pipelining Pattern

Combining async copy with compute to hide latency:

```mlir
// Prefetch K stage 0
nvgpu.device_async_copy %A_global[0], %A_smem[0, :], 64 : ...
nvgpu.device_async_create_group

// Loop over K
scf.for %k = %c1 to %K step %cBK {
  // Issue prefetch for K+1
  nvgpu.device_async_copy %A_global[%next_k], %A_smem[%next_buf, :], 64 : ...
  nvgpu.device_async_create_group
  
  // Wait for K (allow K+1 to be in-flight)
  nvgpu.device_async_wait {num = 1 : i32}
  
  // Compute with K data
  %c_frag = vector.contract ... %a_frag, %b_frag, %acc : ...
  
  // Advance pointers
}
```

---

## 165.7 Warp-Level Reductions

Reductions (softmax denominator, layer norm variance) require communication within a warp.

### 165.7.1 gpu.shuffle

```mlir
// Warp-level sum reduction using shuffle
%val = arith.constant 1.0 : f32
%active_mask = arith.constant -1 : i32  // all lanes active

// Tree reduction: add lanes offset apart
%v1 = gpu.shuffle xor %val, 16, 32 : (f32, i32, i32) -> (f32, i1)
%s1 = arith.addf %val, %v1 : f32
%v2 = gpu.shuffle xor %s1, 8, 32 : (f32, i32, i32) -> (f32, i1)
%s2 = arith.addf %s1, %v2 : f32
// ... continue for offsets 4, 2, 1 ...
// Lane 0 holds the warp sum
```

`gpu.shuffle xor %val, %offset, %width` lowers to:
- NVPTX: `shfl.sync.bfly.b32`
- ROCDL: `ds_swizzle_b32`

### 165.7.2 vector.multi_reduction

At a higher level, `vector.multi_reduction` handles reductions across vector dimensions:

```mlir
%sum = vector.multi_reduction <add>, %values, %acc [1]
    : vector<32xf32> to f32

// Lowers to warp shuffle tree
```

---

## 165.8 Step 5-6: LLVM IR Translation and PTX Emission

```bash
# Translate MLIR → LLVM IR
mlir-translate --mlir-to-llvmir gpu_module.mlir -o gpu_module.ll

# Compile to PTX
llc -march=nvptx64 -mcpu=sm_90 \
    -mattr=+ptx80 \
    -float-abi=hard \
    gpu_module.ll -o gpu_module.ptx

# Assemble PTX → cubin
ptxas -arch=sm_90a \
      --verbose \
      gpu_module.ptx -o gpu_module.cubin
```

For AMD:
```bash
# Translate to LLVM IR (amdgcn target)
mlir-translate --mlir-to-llvmir amd_gpu_module.mlir -o amd_module.ll

# Compile to GFX ISA
llc -march=amdgcn -mcpu=gfx942 \
    -mattr=+wavefrontsize64 \
    amd_module.ll -o amd_module.s

# Assemble to HSACO
clang -x assembler-with-cpp \
      -target amdgcn-amd-amdhsa \
      -mcpu=gfx942 \
      amd_module.s -o amd_module.hsaco
```

### 165.8.1 PTX Feature Requirements

The `mcpu` and `mattr` flags control which PTX features are used:

| Feature | sm_75 | sm_80 | sm_89 | sm_90 |
|---------|-------|-------|-------|-------|
| TF32 | no | yes | yes | yes |
| cp.async | no | yes | yes | yes |
| MMA BF16 | no | yes | yes | yes |
| WGMMA | no | no | no | yes |
| TMA | no | no | no | yes |
| fp8 | no | no | yes | yes |

---

## 165.9 Occupancy and Register Pressure

### 165.9.1 Occupancy Definition

Occupancy = (active warps on SM) / (maximum warps per SM). H100 supports 64 warps per SM. Higher occupancy enables the GPU to hide latency by switching to ready warps.

### 165.9.2 Register Pressure

Each thread has a limited register budget. LLVM's NVPTX backend will spill to local memory (slow!) if register pressure exceeds the hardware limit (~255 regs per thread).

Monitor register usage:
```bash
ptxas -v --gpu-name sm_90 kernel.ptx 2>&1 | grep "registers"
# Output: ptxas info: Function properties for kernel:
#         256 bytes stack frame, 0 bytes spill stores, 0 bytes spill loads
#         Used 128 registers, 16384+0 bytes smem, 128 bytes cmem[0]
```

To reduce register pressure in MLIR:
```mlir
// Prefer smaller vector widths for register-constrained kernels
vector.transfer_read [...] -> vector<4xf32>  // vs. vector<8xf32>
```

### 165.9.3 Shared Memory vs. Registers Trade-off

Using larger tiles increases register pressure but reduces global memory traffic. The optimal tile size depends on the operation's arithmetic intensity:

- GEMM: high arithmetic intensity → large tiles benefit from SMEM
- Element-wise ops: low arithmetic intensity → small tiles, more occupancy
- Softmax: medium intensity → moderate SMEM tiling

---

## 165.10 A Complete Example: linalg.matmul → cubin

```bash
# Step 1: Tile and vectorize
mlir-opt \
    --linalg-tile-and-fuse-tensor-ops="tile-sizes=128,128,32" \
    --linalg-vectorize \
    matmul.mlir > tiled_vectorized.mlir

# Step 2: Map to GPU (using gpu-map-parallel-loops)
mlir-opt \
    --gpu-map-parallel-loops \
    --convert-linalg-to-gpu \
    tiled_vectorized.mlir > gpu_mapped.mlir

# Step 3: Lower GPU ops to NVVM
mlir-opt \
    --convert-gpu-to-nvvm \
    --convert-vector-to-gpu \
    --nvvm-attach-target="chip=sm_90" \
    gpu_mapped.mlir > nvvm.mlir

# Step 4: Lower to LLVM dialect
mlir-opt \
    --convert-nvvm-to-llvm \
    --convert-arith-to-llvm \
    --convert-index-to-llvm \
    --reconcile-unrealized-casts \
    nvvm.mlir > llvm_dialect.mlir

# Step 5: Translate to LLVM IR
mlir-translate --mlir-to-llvmir llvm_dialect.mlir > kernel.ll

# Step 6: Compile to PTX
llc -march=nvptx64 -mcpu=sm_90 kernel.ll > kernel.ptx

# Step 7: Assemble to cubin
ptxas -arch=sm_90a kernel.ptx -o kernel.cubin

echo "Compilation complete: kernel.cubin"
```

---

## 165.11 Embedding in a Host Program

```cpp
// Host code to load and launch the MLIR-compiled cubin
#include <cuda_runtime.h>
#include <stdio.h>

int main() {
  // Load the cubin
  CUmodule module;
  cuModuleLoad(&module, "kernel.cubin");
  
  CUfunction kernel;
  cuModuleGetFunction(&kernel, module, "gpu_matmul");
  
  // Allocate and copy data
  float *d_A, *d_B, *d_C;
  size_t size = 4096 * 4096 * sizeof(float);
  cudaMalloc(&d_A, size);
  cudaMalloc(&d_B, size);
  cudaMalloc(&d_C, size);
  // ... copy data ...
  
  // Launch
  int M = 4096, N = 4096, K = 4096;
  void* args[] = {&d_A, &d_B, &d_C, &M, &N, &K};
  
  // Grid: (M/128, N/128) blocks, 256 threads/block (2 warpgroups)
  cuLaunchKernel(kernel,
      M/128, N/128, 1,   // grid dimensions
      256, 1, 1,          // block dimensions
      228 * 1024,         // shared memory bytes
      nullptr,            // stream
      args, nullptr);
  
  cudaDeviceSynchronize();
  // ... check results ...
}
```

---

## Chapter Summary

- MLIR's GPU compilation path: linalg → tiled linalg → vectorized vector.contract → gpu.launch + thread indexing → nvvm/rocdl → LLVM IR → PTX/GFX → cubin/hsaco.
- Two-level tiling: outer tiles map to GPU blocks (SMEM), inner tiles map to threads (registers); `#gpu.address_space<workgroup>` marks SMEM allocations.
- `vector.contract` is the key abstract matmul op; it lowers to `nvvm.mma.*` (Ampere MMA) or `nvvm.wmma.*` (Volta) depending on target and vector shape.
- `ConvertGpuOpsToNVVMOps` maps `gpu.thread_id`, `gpu.block_id`, `gpu.barrier`, `gpu.shuffle` to NVVM intrinsics; `ConvertGpuOpsToROCDLOps` handles AMD.
- Async copy (`nvgpu.device_async_copy` + `nvgpu.device_async_create_group` + `nvgpu.device_async_wait`) hides global memory latency on Ampere+.
- Warp reductions use `gpu.shuffle xor` in a tree pattern; `vector.multi_reduction` provides a higher-level abstraction.
- Register pressure monitoring via `ptxas -v`; reducing vector widths and using SMEM for intermediate data are primary mitigations.
- The compiled cubin is loaded and launched via standard CUDA driver API (`cuModuleLoad`, `cuModuleGetFunction`, `cuLaunchKernel`).
