# Chapter 164 — CUDA Tile IR

*Part XXIII — MLIR in Production*

NVIDIA's modern GPU programming model has evolved beyond CUDA threads and warps. Hopper (sm_90) introduced the Warpgroup MMA (WGMMA) — a 256-thread cluster-level matrix instruction that computes a 64×128×16 matmul in one operation — and the Tensor Memory Accelerator (TMA) — a hardware unit that performs bulk async copies between global and shared memory. CuTe (a component of CUTLASS) introduced an algebraic tile layout system that describes how logical matrix elements map to physical addresses. MLIR is catching up to this hardware through the `nvgpu` dialect, the CUDA Tile IR initiative, and upstream extensions. This chapter covers these hardware features, their MLIR representations, and how they compose into high-performance Hopper kernel generation.

---

## 164.1 Hardware Context: Hopper Architecture

NVIDIA's Hopper GPU (H100, H200) introduced three hardware mechanisms that fundamentally change kernel authoring:

### 164.1.1 Warpgroup MMA (WGMMA)

Traditional CUDA tensor core ops (Ampere `wmma` or MMA PTX instructions) operate at the warp level — 32 threads cooperate on a 16×16×16 matmul. WGMMA operates at the warpgroup level — 128 threads (4 warps) cooperate on a single instruction:

```
WGMMA.m64n256k16.F16: 64×256 output tiles, consuming 64×16 and 16×256 input tiles
WGMMA.m64n128k16.F32 (BF16 inputs): 64×128 FP32 outputs from BF16 inputs
```

The instruction throughput is dramatically higher than warp-level MMA because the hardware can pipeline the warpgroup operation across the 4 warps internally.

### 164.1.2 Tensor Memory Accelerator (TMA)

TMA is an on-chip DMA unit that copies rectangular tiles between global memory and shared memory (SMEM) asynchronously. It addresses the bandwidth bottleneck of loading matrix tiles:

```ptx
// PTX TMA async copy
cp.async.bulk.tensor.2d.shared::cluster.global
    [smem_desc, global_ptr, tensor_map, coord_x, coord_y];
mbarrier.arrive.expect_tx.shared::cta.b64 [mbar], expected_bytes;
```

Key features:
- **Strided access**: the tensor map encodes the tensor dimensions, strides, and swizzle pattern; TMA handles non-contiguous layouts natively
- **Async with barrier**: producer uses `mbarrier.arrive`, consumer uses `mbarrier.wait`
- **Multicast**: TMA can broadcast data to multiple CTA clusters simultaneously

### 164.1.3 CTA Clusters

Hopper introduced **CTA clusters** — groups of CTAs that can share a single hardware SM cluster. Cluster-level distributed shared memory (DSMEM) allows one CTA to read another CTA's shared memory directly:

```ptx
// Load from another CTA's shared memory
ld.shared::cluster.b32 %r0, [%ptr_in_remote_cta_smem];
```

---

## 164.2 CuTe: The Layout Algebra

CuTe (part of CUTLASS 3.x) introduces a formal algebra for describing how tensor elements map to hardware resources. It is not a compiler IR — it is a C++ template library — but its concepts are the foundation for NVIDIA's Tile IR effort in MLIR.

### 164.2.1 Layouts

A CuTe `Layout` is a pair `(Shape, Stride)`:

```cpp
// A 4×4 row-major matrix with stride (4, 1)
auto layout = make_layout(make_shape(4, 4), make_stride(4, 1));

// Map logical index (i, j) to physical offset:
// offset = i * stride_0 + j * stride_1 = 4*i + j
int offset = layout(2, 3);  // = 11
```

Layouts compose: you can apply a layout to the output of another layout, creating multi-level tile hierarchies.

### 164.2.2 Tile Algebra

```cpp
// Tiling a 128×128 matrix with 32×32 tiles:
auto matrix = make_layout(make_shape(128, 128), make_stride(128, 1));
auto tile = make_shape(32, 32);

// Logical tile coordinates: (tile_i, tile_j, elem_i, elem_j)
auto tiled = zipped_divide(matrix, tile);
// tiled has shape ((4, 4), (32, 32)) with appropriate strides

// Access tile (1, 2), element (5, 3):
int offset = tiled(make_coord(make_coord(1, 2), make_coord(5, 3)));
```

This algebra generalizes to arbitrary hierarchies (warpgroup-level, warp-level, thread-level tiles) and automatically handles swizzled layouts needed to avoid shared memory bank conflicts.

### 164.2.3 Atom Layouts

Tensor core operations have specific register file layouts. CuTe models these as "atoms":

```cpp
// MMA atom for Hopper WGMMA 64×128×16 in BF16
using MmaAtom = MMA_Atom<SM90_64x128x16_F32BF16BF16F32_SS>;
// Shape of register file tile owned by one warpgroup: [64, 128] output
```

---

## 164.3 The nvgpu Dialect in MLIR

The `nvgpu` dialect in MLIR ([mlir/include/mlir/Dialect/NVGPU/](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/NVGPU/)) provides MLIR-level ops for Hopper hardware features.

### 164.3.1 TMA Descriptor Creation

```mlir
// Create a TMA tensor descriptor
%tma_desc = nvgpu.tma.descriptor.create
    %global_ptr[%dim0, %dim1],
    box_shape = [32, 64],  // tile size
    element_type = f16,
    swizzle = "swizzle_128B"
    : memref<128x256xf16>, index, index
        -> !nvgpu.tensormap.descriptor<tensor = memref<32x64xf16>>
```

The descriptor encodes the tensor's dimensions, strides, and the swizzle pattern needed for conflict-free SMEM access.

### 164.3.2 TMA Async Load

```mlir
// Allocate shared memory
%smem = memref.alloc() : memref<32x64xf16, #gpu.address_space<workgroup>>

// Create barrier for synchronization
%mbar = nvgpu.mbarrier.create -> !nvgpu.mbarrier.type<1>

// Issue async TMA load
nvgpu.tma.async.load %tma_desc[%coord_i, %coord_j], %mbar
    into %smem[0, 0] : !nvgpu.tensormap.descriptor<tensor=memref<32x64xf16>>,
                        !nvgpu.mbarrier.type<1>,
                        memref<32x64xf16, #gpu.address_space<workgroup>>

// Wait for TMA completion
nvgpu.mbarrier.wait %mbar : !nvgpu.mbarrier.type<1>
```

### 164.3.3 Warpgroup MMA

```mlir
// Perform WGMMA: D += A * B
// A is in shared memory, B is in registers or shared memory
%d_out = nvgpu.warpgroup.mma
    %a_smem, %b_smem, %d_in,
    transposeA = false, transposeB = true
    : memref<64x16xbf16, #gpu.address_space<workgroup>>,
      memref<128x16xbf16, #gpu.address_space<workgroup>>,
      !nvgpu.warpgroup.accumulator<fragmented = vector<64x128xf32>>
    -> !nvgpu.warpgroup.accumulator<fragmented = vector<64x128xf32>>
```

The `!nvgpu.warpgroup.accumulator` type tracks the fragmented register layout that 128 threads collectively hold for the 64×128 accumulator tile.

### 164.3.4 Warpgroup Accumulator → MemRef

After the WGMMA loop, the accumulator is stored back to memory:

```mlir
nvgpu.warpgroup.mma.store %d_out, %output_smem
    : !nvgpu.warpgroup.accumulator<fragmented = vector<64x128xf32>>,
      memref<64x128xf32, #gpu.address_space<workgroup>>
```

---

## 164.4 The Tile IR Initiative

NVIDIA, together with the MLIR community, is developing a **Tile IR** — a higher-level MLIR abstraction for tiled computations that maps cleanly to hardware tile units like WGMMA and TMA. The initiative is in active development (2024–2026) in the LLVM monorepo.

### 164.4.1 Design Goals

The Tile IR aims to:

1. **Abstract over hardware generations**: express a matmul at tile granularity without hard-coding Hopper-specific parameters; lower to WGMMA on Hopper, to MMA on Ampere, to Triton on AMD.
2. **Express tiling hierarchy**: describe the relationship between a 64×128 warpgroup tile, 16×128 warp tiles, and 8×8 thread elements without manually constructing CuTe layouts.
3. **Enable compiler-driven layout selection**: instead of the programmer choosing swizzle patterns, the compiler selects the layout that avoids bank conflicts given the access pattern.

### 164.4.2 Current State: nvgpu.tma + nvgpu.warpgroup

The current `nvgpu` ops are the first step. A matmul kernel using these ops:

```mlir
gpu.func @wgmma_matmul(%A_global: memref<4096x4096xbf16>,
                        %B_global: memref<4096x4096xbf16>,
                        %C_global: memref<4096x4096xf32>) {

  // Create TMA descriptors
  %A_desc = nvgpu.tma.descriptor.create %A_global[...] : ...
  %B_desc = nvgpu.tma.descriptor.create %B_global[...] : ...

  // Shared memory tiles
  %A_smem = memref.alloc() : memref<64x32xbf16, #gpu.address_space<workgroup>>
  %B_smem = memref.alloc() : memref<128x32xbf16, #gpu.address_space<workgroup>>

  // Accumulator register tile (distributed across 128 threads)
  %C_acc = nvgpu.warpgroup.mma.init_accumulator
      : !nvgpu.warpgroup.accumulator<fragmented = vector<64x128xf32>>

  // Tile K loop
  %final_acc = scf.for %k = %c0 to %K step %cBLOCK_K
      iter_args(%acc = %C_acc)
      -> !nvgpu.warpgroup.accumulator<fragmented = vector<64x128xf32>> {
    
    // Async load tiles via TMA
    nvgpu.tma.async.load %A_desc[%m_coord, %k], %mbar
        into %A_smem[0, 0] : ...
    nvgpu.tma.async.load %B_desc[%n_coord, %k], %mbar
        into %B_smem[0, 0] : ...
    nvgpu.mbarrier.wait %mbar : ...
    
    // WGMMA
    %new_acc = nvgpu.warpgroup.mma %A_smem, %B_smem, %acc,
        transposeA = false, transposeB = true : ...
    
    scf.yield %new_acc : ...
  }
  
  // Store accumulator
  nvgpu.warpgroup.mma.store %final_acc, %C_smem : ...
  // ... async copy back to global ...
}
```

### 164.4.3 linalg + Transform Dialect Path

An alternative to hand-written `nvgpu` IR is using linalg.matmul + the Transform dialect to auto-generate Hopper-optimized code:

```mlir
// Transform sequence for Hopper matmul
transform.sequence failures(propagate) {
^bb0(%module: !transform.any_op):
  %matmul = transform.structured.match ops{["linalg.matmul"]} in %module

  // Tile for warpgroup: [64, 128, 32] tiles
  %tiled, %loops = transform.structured.tile_using_for %matmul [64, 128, 32]
  
  // Map to GPU thread hierarchy
  %gpu_mapped = transform.gpu.map_nested_forall_to_threads %tiled
      block_dims = [128, 1, 1]  // 4 warps = 128 threads
  
  // Lower to nvgpu.warpgroup.mma
  transform.nvgpu.rewrite_matmul_as_mma_sync %gpu_mapped
}
```

This transform sequence is the emerging pattern for Hopper codegen in MLIR.

---

## 164.5 Memory Hierarchy for Hopper Kernels

Efficient Hopper kernels exploit the full memory hierarchy:

```
L2 cache (50 MB, shared across all SMs)
    │ TMA bulk async copy
    ▼
Shared Memory (SMEM) per SM (228 KB on H100)
    │ WGMMA reads from SMEM
    ▼
Register File per warpgroup (256 KB per SM total)
    │ distributed accumulator
    ▼
L2 → global memory (TMA async copy back)
```

### 164.5.1 SMEM Pipeline Depth

A key performance knob is SMEM pipeline depth — how many K-tiles of A and B fit simultaneously in SMEM:

```
SMEM budget: 228 KB
A tile [64, 32] × BF16 = 4 KB
B tile [128, 32] × BF16 = 8 KB
Pipeline depth 4: 4 × (4 + 8) = 48 KB for AB tiles
Remaining: 180 KB available for C or other uses
```

Deeper pipelines hide TMA latency better but consume more SMEM and reduce the number of concurrent CTAs per SM (occupancy).

### 164.5.2 mbarrier Synchronization

Hopper's `mbarrier` is the synchronization primitive for TMA:

```mlir
// Initialize a per-warpgroup mbarrier with 128-thread arrive count
%mbar = nvgpu.mbarrier.create 128 : !nvgpu.mbarrier.type<128>

// Producer (TMA unit) arrives automatically when load completes
nvgpu.tma.async.load ... %mbar ...

// Consumer (WGMMA) waits for all arrivals
nvgpu.mbarrier.wait %mbar phase %phase : !nvgpu.mbarrier.type<128>
// %phase alternates 0/1 for double-buffering
```

---

## 164.6 Putting It Together: A Simple Hopper GEMM in MLIR

Combining all the pieces, a simplified Hopper GEMM (excluding full pipelining for clarity):

```mlir
// Assume: M=4096, N=4096, K=4096, BLOCK_M=64, BLOCK_N=128, BLOCK_K=32
// One warpgroup (128 threads) computes a 64×128 output tile

gpu.func @hopper_gemm(%A: memref<4096x4096xbf16>,
                       %B: memref<4096x4096xbf16>,
                       %C: memref<4096x4096xf32>)
    attributes {gpu.known_block_size = array<i32: 128, 1, 1>} {
  
  %c0 = arith.constant 0 : index
  %cK = arith.constant 4096 : index
  %cBK = arith.constant 32 : index
  
  // Create TMA descriptors from tensor maps (set up by host)
  %A_desc = nvgpu.tma.descriptor.create %A[...] : ...
  %B_desc = nvgpu.tma.descriptor.create %B[...] : ...
  
  // Allocate SMEM for double-buffered tiles (2 * (A_tile + B_tile))
  %A_smem = memref.alloc() : memref<2x64x32xbf16, #gpu.address_space<workgroup>>
  %B_smem = memref.alloc() : memref<2x128x32xbf16, #gpu.address_space<workgroup>>
  
  // Initialize accumulator
  %init_acc = nvgpu.warpgroup.mma.init_accumulator
      : !nvgpu.warpgroup.accumulator<fragmented = vector<64x128xf32>>
  
  // K loop (simplified: no double-buffering shown)
  %final = scf.for %k = %c0 to %cK step %cBK iter_args(%acc = %init_acc)
      -> !nvgpu.warpgroup.accumulator<fragmented = vector<64x128xf32>> {
    
    %buf = arith.remui %k, %c2 : index  // ping-pong buffer index
    %mbar = nvgpu.mbarrier.create : !nvgpu.mbarrier.type<1>
    
    nvgpu.tma.async.load %A_desc[%bid_m, %k], %mbar
        into %A_smem[%buf, 0, 0] : ...
    nvgpu.tma.async.load %B_desc[%bid_n, %k], %mbar
        into %B_smem[%buf, 0, 0] : ...
    nvgpu.mbarrier.wait %mbar : !nvgpu.mbarrier.type<1>
    
    %A_slice = memref.subview %A_smem[%buf, 0, 0][64, 32][1, 1] : ...
    %B_slice = memref.subview %B_smem[%buf, 0, 0][128, 32][1, 1] : ...
    
    %new_acc = nvgpu.warpgroup.mma %A_slice, %B_slice, %acc,
        transposeB = true : ...
    scf.yield %new_acc : ...
  }
  
  // Store accumulator to global memory
  %C_smem = memref.alloc() : memref<64x128xf32, #gpu.address_space<workgroup>>
  nvgpu.warpgroup.mma.store %final, %C_smem : ...
  
  // Async copy SMEM → global
  %bid_m = gpu.block_id  x : index
  %bid_n = gpu.block_id  y : index
  // ... store C_smem to C[bid_m*64:(bid_m+1)*64, bid_n*128:(bid_n+1)*128] ...
}
```

---

## 164.7 Performance Expectations

On an H100 SXM5:
- Peak BF16 tensor core throughput: 989 TFLOPS (dense) / 1,979 TFLOPS (sparse)
- TMA bandwidth to shared memory: ~12 TB/s aggregate (vs. ~7 TB/s for Ampere cp.async)
- WGMMA utilization in well-optimized kernels: 85–95% of peak
- cuBLAS achieves ~970 TFLOPS for large square GEMMs — near-peak utilization

Achieving near-cuBLAS performance with MLIR-generated code requires:
1. TMA double-buffering (2 stages of A and B in SMEM)
2. WGMMA issue rate matching TMA latency (hiding ~30 cycle TMA latency with ~3 pipeline stages)
3. Correct swizzle patterns (128B swizzle for BF16 tiles)
4. Warpgroup scheduling alignment (WGMMA instructions separated by `wgmma.wait_group`)

---

## Chapter Summary

- Hopper introduces three hardware features: WGMMA (128-thread 64×128×16 matmul), TMA (async bulk copy with tensor maps), and CTA clusters (inter-CTA SMEM sharing).
- CuTe's layout algebra describes multi-level tile hierarchies as `(Shape, Stride)` pairs; it underpins NVIDIA's Tile IR vision for MLIR.
- The `nvgpu` dialect provides MLIR ops for TMA (`nvgpu.tma.descriptor.create`, `nvgpu.tma.async.load`) and WGMMA (`nvgpu.warpgroup.mma`, `nvgpu.warpgroup.mma.init_accumulator`, `nvgpu.warpgroup.mma.store`).
- `mbarrier` synchronizes TMA producers with WGMMA consumers; phase alternation enables double-buffering.
- The Transform dialect path (`transform.nvgpu.rewrite_matmul_as_mma_sync`) is the emerging high-level approach to auto-generate Hopper-optimized GEMM code.
- SMEM pipeline depth is the key tuning knob: more stages hide TMA latency but reduce occupancy; 3–4 stages is typical for 64×128 tiles on H100.
- Near-peak WGMMA utilization (85–95%) requires correctly coordinated TMA + WGMMA pipelining and swizzled SMEM layouts.


---

@copyright jreuben11
