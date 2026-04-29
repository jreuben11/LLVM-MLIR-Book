# Chapter 142 — GPU Dialect Family

*Part XX — In-Tree Dialects*

GPU programming in MLIR is expressed through a family of dialects that together cover the full spectrum from target-independent abstract GPU code to hardware-specific PTX and GCN assembly. The `gpu` dialect provides the execution model shared by CUDA and OpenCL — threads, blocks, grids, shared memory, barriers — in a target-neutral form. The `nvgpu` dialect extends this with NVIDIA-specific features like Tensor Core operations and asynchronous copies. ROCDL provides AMD-specific intrinsics. The GPU lowering infrastructure ties them together, enabling a single MLIR program to target multiple GPU vendors. This chapter traces the full picture from `gpu.launch` to machine code, covering each layer and its abstractions.

---

## 142.1 The `gpu` Dialect

### 142.1.1 Execution Model

The `gpu` dialect ([`mlir/include/mlir/Dialect/GPU/IR/GPUOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/GPU/IR/GPUOps.td)) models the CUDA/OpenCL execution model:

- **Grid**: collection of thread blocks
- **Block** (work group): collection of threads executing together, with shared memory
- **Thread** (work item): the fundamental execution unit; has private registers and stack
- **Warp** (CUDA) / **Wave** (AMD): 32-64 threads executing in lockstep (SIMT)

The three levels of parallelism correspond to:
```
grid   = (gridDimX, gridDimY, gridDimZ)    # number of blocks
block  = (blockDimX, blockDimY, blockDimZ) # threads per block
```

A kernel function runs for each thread; each thread knows its position via:

```mlir
%bx = gpu.block_id  x  // block index within grid (x dimension)
%by = gpu.block_id  y
%tx = gpu.thread_id x  // thread index within block (x dimension)
%ty = gpu.thread_id y
%bdx = gpu.block_dim x  // blockDim.x (threads per block in x)
%gdx = gpu.grid_dim x   // gridDim.x (blocks per grid in x)
```

### 142.1.2 `gpu.module` and `gpu.func`

```mlir
// GPU module: contains kernel code, compiled separately from host
gpu.module @kernels {
  // Kernel entry point (called from host via gpu.launch_func)
  gpu.func @elementwise_add(%A: memref<?xf32, #gpu.address_space<global>>,
                             %B: memref<?xf32, #gpu.address_space<global>>,
                             %C: memref<?xf32, #gpu.address_space<global>>,
                             %n: index)
      kernel attributes {gpu.known_block_size = array<i32: 256, 1, 1>} {
    %tid_x = gpu.thread_id x
    %bid_x = gpu.block_id  x
    %bdim_x = gpu.block_dim x
    %gid = arith.addi %tid_x, arith.muli %bid_x, %bdim_x : index
    %cond = arith.cmpi slt, %gid, %n : index
    scf.if %cond {
      %a = memref.load %A[%gid] : memref<?xf32, #gpu.address_space<global>>
      %b = memref.load %B[%gid] : memref<?xf32, #gpu.address_space<global>>
      %sum = arith.addf %a, %b : f32
      memref.store %sum, %C[%gid] : memref<?xf32, #gpu.address_space<global>>
    }
    gpu.return
  }
}
```

`gpu.func` is analogous to `func.func` but for GPU execution. The `kernel` keyword marks it as a kernel entry point (callable from host). Non-kernel `gpu.func` are device functions callable from kernels.

The `attributes {gpu.known_block_size}` is an optional hint enabling the compiler to specialize the code assuming a fixed block size.

### 142.1.3 `gpu.launch`

`gpu.launch` is the host-side op that launches a GPU kernel inline:

```mlir
// Launch kernel with grid = (n/256, 1, 1), block = (256, 1, 1)
%gx = arith.ceildivsi %n, %c256 : index
gpu.launch blocks(%bx, %by, %bz) in (%gbx = %gx, %gby = %c1, %gbz = %c1)
           threads(%tx, %ty, %tz) in (%tbx = %c256, %tby = %c1, %tbz = %c1) {
  // kernel body (executed on GPU)
  %tid = arith.addi %tx, arith.muli %bx, %c256 : index
  %cond = arith.cmpi slt, %tid, %n : index
  scf.if %cond {
    %a = memref.load %A[%tid] : memref<?xf32, #gpu.address_space<global>>
    %b = memref.load %B[%tid] : memref<?xf32, #gpu.address_space<global>>
    %sum = arith.addf %a, %b : f32
    memref.store %sum, %C[%tid] : memref<?xf32, #gpu.address_space<global>>
  }
  gpu.terminator
}
```

The variables `%bx, %by, %bz` are bound to the block indices at runtime; `%tx, %ty, %tz` to the thread indices. The grid/block dimension operands (`%gbx`, `%tbx`, etc.) specify the launch configuration.

### 142.1.4 `gpu.launch_func`

`gpu.launch_func` launches a pre-compiled kernel function by symbol reference — the runtime analog of `func.call`:

```mlir
gpu.launch_func @kernels::@elementwise_add
    blocks in (%gx, %c1, %c1)
    threads in (%c256, %c1, %c1)
    args(%A : memref<?xf32>, %B : memref<?xf32>, %C : memref<?xf32>, %n : index)
```

This is what remains after `--gpu-kernel-outlining` converts a `gpu.launch` into a separate `gpu.module` + `gpu.func` pair with a corresponding `gpu.launch_func` on the host side.

### 142.1.5 Synchronization

```mlir
// Synchronize all threads in a block (barrier)
gpu.barrier

// Warp-level shuffle operations
%result = gpu.shuffle xor %val, %offset, %width : f32
%result2 = gpu.shuffle up   %val, %delta, %width : f32
%result3 = gpu.shuffle down %val, %delta, %width : f32
%result4 = gpu.shuffle idx  %val, %src_lane, %width : f32
```

`gpu.barrier` corresponds to `__syncthreads()` in CUDA. All four shuffle variants correspond to `__shfl_xor_sync`, `__shfl_up_sync`, `__shfl_down_sync`, and `__shfl_idx_sync`.

Warp reduction pattern:
```mlir
// Tree reduction using shuffles (reduce 32 values to 1)
%v0 = gpu.shuffle xor %initial, %c16, %c32 : f32
%v1 = arith.addf %v0, %initial : f32
%v2 = gpu.shuffle xor %v1, %c8, %c32 : f32
%v3 = arith.addf %v2, %v1 : f32
// ... continue for 4, 2, 1
```

---

## 142.2 GPU Memory Model

### 142.2.1 Address Spaces

GPU memory has a hierarchy with distinct address spaces:

| Address space | CUDA equivalent | Description |
|---|---|---|
| `#gpu.address_space<global>` | global memory | Device DRAM, accessible by all threads, high latency |
| `#gpu.address_space<workgroup>` | shared memory | Per-block SRAM, ~32KB, fast, ~20 cycle latency |
| `#gpu.address_space<private>` | local memory (register spill) | Per-thread stack/registers |

```mlir
// Allocate shared memory (workgroup address space)
%shared = memref.alloca() : memref<256xf32, #gpu.address_space<workgroup>>

// Load from global, store to shared
%gval = memref.load %global_buf[%tid] : memref<?xf32, #gpu.address_space<global>>
memref.store %gval, %shared[%tid_local] : memref<256xf32, #gpu.address_space<workgroup>>
gpu.barrier  // ensure all threads have written shared memory

// Load from shared
%sval = memref.load %shared[%lane] : memref<256xf32, #gpu.address_space<workgroup>>
```

The address space attribute on `memref` types enables the compiler to generate correct memory instructions (global vs. shared memory reads) without runtime overhead.

### 142.2.2 Device Memory Operations

```mlir
// Allocate device memory (returns a global address-space memref)
%dev_mem, %token = gpu.alloc async [%wait_token] (%size) : memref<?xf32>

// Copy host → device
%copy_tok = gpu.memcpy async [%alloc_tok] %dev_mem, %host_mem
    : memref<?xf32>, memref<?xf32>

// Copy device → host
%copy_back = gpu.memcpy async [%compute_tok] %host_out, %dev_result
    : memref<?xf32>, memref<?xf32>

// Wait for all async operations
gpu.wait [%copy_back]

// Free device memory
gpu.dealloc async [%done_tok] %dev_mem : memref<?xf32>

// Fill device memory with a value
gpu.memset async [%tok] %dev_mem, %fill_val : memref<?xf32>, f32
```

The `async` attribute and token system mirrors `async` dialect patterns: each GPU op returns a token, and subsequent ops can depend on previous tokens, creating an explicit data-flow graph of async operations.

---

## 142.3 NVGPU Dialect

### 142.3.1 Tensor Core Operations

The `nvgpu` dialect ([`mlir/include/mlir/Dialect/NVGPU/IR/NVGPUDialect.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/NVGPU/IR/NVGPUDialect.td)) extends `gpu` with NVIDIA-specific Tensor Core and hardware features.

```mlir
// Warp Matrix Multiply-Accumulate (WMMA / mma.sync)
// Each warp (32 threads) cooperates to compute a 16×16 matmul
%d_frag = nvgpu.mma.sync(%a_frag, %b_frag, %c_frag)
    {mmaShape = [16, 8, 16], tf32Enabled = false}
    : (vector<4x2xf16>, vector<2x2xf16>, vector<2x2xf32>) -> vector<2x2xf32>

// Load matrix fragments from shared memory
%a_frag = nvgpu.ldmatrix %shared_mem[%row, %col]
    {numTiles = 4 : i32, transpose = false}
    : memref<16x16xf16, #gpu.address_space<workgroup>> -> vector<4x2xf16>
```

`nvgpu.mma.sync` maps to PTX `mma.sync` instructions that execute across 32 threads in a warp, each thread holding a fragment of the matrices. The `mmaShape` specifies the tile size (M×N×K); supported shapes depend on the GPU architecture.

### 142.3.2 Asynchronous Copies (Ampere)

```mlir
// cp.async: copy from global to shared asynchronously (no register staging)
%group = nvgpu.device_async_create_group
%tok = nvgpu.device_async_copy
    %dst[%dst_idx], %src[%src_idx], 16
    {bypassL1 = false} : memref<...> to memref<..., #gpu.address_space<workgroup>>

// Commit and wait
nvgpu.device_async_commit_group
nvgpu.device_async_wait {numGroups = 0 : i32}
```

`cp.async` on Ampere hardware can copy data from global memory to shared memory without going through registers, improving memory bandwidth utilization. The `bypassL1` flag corresponds to the `.ca`/`.cg` cache policy.

### 142.3.3 Hopper Features (WGMMA and TMA)

Hopper architecture (sm_90) introduces two major features:

**Warpgroup MMA (WGMMA)**: 128-thread warpgroup matrix operations that use the Tensor Memory Accelerator:

```mlir
// Initialize WGMMA accumulator descriptor
%acc = nvgpu.warpgroup.mma.init.accumulator
    {matrixType = vector<64x8xf32>} : !nvgpu.warpgroup.accumulator<fragmented = vector<64x8xf32>>

// Execute WGMMA
%result_acc = nvgpu.warpgroup.mma %descA, %descB, %acc
    {transposeA = false, transposeB = true}
    : !nvgpu.warpgroup.descriptor<tensor = memref<64x16xf16, #gpu.address_space<workgroup>>>,
      !nvgpu.warpgroup.descriptor<tensor = memref<8x16xf16, #gpu.address_space<workgroup>>>,
      !nvgpu.warpgroup.accumulator<fragmented = vector<64x8xf32>>
      -> !nvgpu.warpgroup.accumulator<fragmented = vector<64x8xf32>>
```

**Tensor Memory Accelerator (TMA)**: hardware unit for bulk async copies with striding and transposition:

```mlir
// Create TMA descriptor (describes global memory layout)
%tma_desc = nvgpu.tma.create.descriptor %global_mem {
  tileType = memref<128x64xf16, #gpu.address_space<workgroup>>
} : memref<?x?xf16> -> !nvgpu.tma.descriptor<tensor = memref<128x64xf16, #gpu.address_space<workgroup>>>

// TMA load (bulk copy from global to shared)
%barrier = nvgpu.mbarrier.create : !nvgpu.mbarrier.group<memorySpace = #gpu.address_space<workgroup>>
nvgpu.tma.async.load %tma_desc[%coord0, %coord1], %barrier, %shared_mem
    {operandSegmentSizes = array<i32: 1, 2, 1, 1>}
    : ...
```

TMA transfers are initiated from a single thread and do not require each thread to issue individual loads, dramatically reducing instruction overhead for large copies.

---

## 142.4 GPU Lowering Pipeline

### 142.4.1 Kernel Outlining

The first step in any GPU pipeline is to separate host and device code:

```bash
# Extract gpu.launch bodies into separate gpu.func + gpu.module
mlir-opt --gpu-kernel-outlining input.mlir
```

This converts inline `gpu.launch` bodies into named `gpu.func` functions inside a `gpu.module`, and replaces the launch body with `gpu.launch_func`. The host module and GPU module are now separate compilation units.

### 142.4.2 NVIDIA Path (NVPTX)

```bash
# Step 1: Lower gpu dialect to NVVM (NVIDIA Virtual Machine) intrinsics
mlir-opt --gpu-to-nvvm input.mlir

# Step 2: Lower NVVM dialect to LLVM + NVPTX intrinsics
mlir-opt --nvvm-to-llvm input.mlir

# Step 3: Translate to PTX via LLVM's NVPTX backend
mlir-translate --mlir-to-nvvmir --target-chip=sm_80 input.mlir | \
  llc -march=nvptx64 -mcpu=sm_80 -o kernel.ptx
```

Key passes in `--gpu-to-nvvm`:
- `gpu.thread_id x` → `nvvm.read.ptx.sreg.tid.x`
- `gpu.block_id x` → `nvvm.read.ptx.sreg.ctaid.x`
- `gpu.barrier` → `nvvm.barrier0`
- `gpu.shuffle xor` → `nvvm.shfl.sync.bfly`
- Shared memory allocation → `nvvm.ptr.shared.to.gen`

### 142.4.3 AMD Path (ROCDL/AMDGPU)

```bash
# Lower gpu dialect to ROCDL (ROCm Device Library) intrinsics
mlir-opt --gpu-to-rocdl input.mlir

# Lower ROCDL + AMDGPU to LLVM
mlir-opt --rocdl-to-llvm input.mlir

# Translate to AMDGPU bitcode
mlir-translate --mlir-to-rocdlir input.mlir
```

The ROCDL lowering maps:
- `gpu.thread_id x` → `rocdl.workitem.id.x`
- `gpu.block_id x` → `rocdl.workgroup.id.x`
- `gpu.barrier` → `rocdl.barrier`
- `gpu.shuffle` → `rocdl.ds_bpermute`

### 142.4.4 Host-Side Lowering

After device code is compiled, the host side needs to be lowered too:

```bash
# Lower gpu.launch_func + gpu.alloc/memcpy to CUDA/HIP runtime API calls
mlir-opt --gpu-to-llvm \
  "cubin-chip=sm_80 cubin-features=+ptx75 cubin-format=isa" \
  input.mlir
```

`--gpu-to-llvm` converts:
- `gpu.alloc` → `cudaMalloc` / `hipMalloc`
- `gpu.dealloc` → `cudaFree` / `hipFree`
- `gpu.memcpy` → `cudaMemcpy` / `hipMemcpy`
- `gpu.launch_func` → `cudaLaunchKernel` / `hipLaunchKernel` (with binary embedded in the module as a global string)

The compiled GPU binary (PTX or GCN) is embedded in the module as a string constant and loaded at runtime via `cuModuleLoadData` / `hipModuleLoadData`.

### 142.4.5 Complete End-to-End Pipeline

```bash
# NVIDIA end-to-end GPU compilation
mlir-opt \
  --gpu-kernel-outlining \
  --lower-affine \
  --convert-scf-to-cf \
  --convert-arith-to-llvm \
  --gpu-to-nvvm \
  --nvvm-to-llvm \
  --reconcile-unrealized-casts \
  input.mlir > device.mlir

mlir-translate --mlir-to-nvvmir device.mlir > device.ll
llc -march=nvptx64 -mcpu=sm_80 -O3 device.ll -o kernel.ptx
ptxas -arch sm_80 kernel.ptx -o kernel.cubin

# Then on host side with the cubin embedded:
mlir-opt \
  --gpu-to-llvm="gpu-binary-annotation=gpu.binary" \
  host.mlir | \
mlir-translate --mlir-to-llvmir -o host.ll
```

### 142.4.6 `GpuToLLVMConversionPass`

For a complete end-to-end conversion including binary embedding:

```cpp
// In C++ pipeline construction:
pm.addPass(mlir::createGpuToLLVMConversionPass(
    GpuToLLVMConversionPassOptions{
        .hostBarrier = false,
        .gpuBinaryAnnotation = "gpu.binary"
    }
));
```

This pass:
1. Lowers `gpu.launch_func` to CUDA/HIP runtime calls
2. Loads the pre-compiled binary from the module annotation
3. Sets up argument marshaling for kernel arguments
4. Handles async tokens by mapping to CUDA streams

---

## 142.5 GPU Address Spaces and Type Conversion

### 142.5.1 Address Space in Lowering

When GPU code is lowered to LLVM, address spaces must be preserved:

```mlir
// MLIR
memref<256xf32, #gpu.address_space<workgroup>>
// ↓ after --gpu-to-nvvm
!llvm.ptr<f32, 3>    // address space 3 = NVPTX shared memory
// ↓ after lowering on host
!llvm.ptr<f32>       // address space 0 = generic
```

CUDA address space numbers: 0 = generic, 1 = global, 3 = shared, 4 = constant, 5 = local.

### 142.5.2 The `nvvm` Dialect

The `nvvm` dialect ([`mlir/include/mlir/Dialect/LLVMIR/NVVMDialect.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/LLVMIR/NVVMDialect.td)) sits between `gpu` and the LLVM dialect. It provides NVPTX-specific intrinsics not available in plain LLVM dialect:

```mlir
// PTX special registers
%tid   = nvvm.read.ptx.sreg.tid.x   : i32
%ntid  = nvvm.read.ptx.sreg.ntid.x  : i32
%ctaid = nvvm.read.ptx.sreg.ctaid.x : i32
%nctaid = nvvm.read.ptx.sreg.nctaid.x : i32
%clock = nvvm.read.ptx.sreg.clock64  : i64  // performance counter

// Barrier
nvvm.barrier0  // __syncthreads()

// Vote instructions
%any = nvvm.vote.any %pred : i1 -> i32
%all = nvvm.vote.all %pred : i1 -> i32

// Texture fetch
%t = nvvm.tex.2d.v4f32.f32 %tex, %sampler, %x, %y : ...
```

---

## Chapter 142 Summary

- The `gpu` dialect models the CUDA/OpenCL execution model: threads, blocks, grids, shared memory, and synchronization. `gpu.block_id`, `gpu.thread_id`, `gpu.block_dim`, `gpu.grid_dim` provide hardware indices.
- `gpu.module` contains kernel code; `gpu.func` with the `kernel` attribute is a kernel entry point. `gpu.launch` inlines kernel body on host; `gpu.launch_func` calls a precompiled kernel by symbol reference.
- `gpu.barrier` provides thread block synchronization; `gpu.shuffle` provides warp-level data sharing across four variants (xor/up/down/idx).
- GPU memory hierarchy: `#gpu.address_space<global>` (device DRAM), `#gpu.address_space<workgroup>` (shared memory, per-block SRAM), `#gpu.address_space<private>` (registers/stack). Address space annotations drive correct instruction selection.
- The `nvgpu` dialect extends `gpu` with NVIDIA-specific ops: `nvgpu.mma.sync` for Tensor Core warp-level matmul, `nvgpu.ldmatrix` for fragment loads, `nvgpu.device_async_copy` for Ampere `cp.async`, and Hopper WGMMA/TMA operations.
- GPU lowering pipeline: `--gpu-kernel-outlining` separates host/device, `--gpu-to-nvvm` lowers to NVVM dialect for NVIDIA targets, `--gpu-to-rocdl` for AMD targets. `--gpu-to-llvm` on the host side replaces `gpu.launch_func` with CUDA/HIP runtime API calls.
- The complete NVIDIA pipeline produces PTX via the LLVM NVPTX backend, then embeds compiled cubins in the host module for runtime loading.


---

@copyright jreuben11
