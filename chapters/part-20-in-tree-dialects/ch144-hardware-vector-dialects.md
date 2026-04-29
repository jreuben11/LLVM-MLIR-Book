# Chapter 144 — Hardware Vector Dialects

*Part XX — In-Tree Dialects*

The `vector` dialect (Chapter 141) provides a target-independent SIMD abstraction, but real performance on specific hardware requires reaching for instruction-set-specific intrinsics: AArch64 NEON dot products, SVE scalable operations, SME outer products on the ZA array, x86 AVX-512 compress stores, and AMD MFMA matrix operations. MLIR provides dedicated dialects for each major hardware vector ISA — `arm_neon`, `arm_sve`, `arm_sme`, `x86vector`, `amdgpu`, and `rocdl` — that map directly to architecture intrinsics. These dialects sit below the `vector` dialect in the abstraction hierarchy and are the final target-specific step before entering the LLVM dialect. This chapter covers each hardware vector dialect, its key operations, and its lowering path.

---

## 144.1 `arm_neon`

### 144.1.1 Overview

The `arm_neon` dialect ([`mlir/include/mlir/Dialect/ArmNeon/ArmNeon.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/ArmNeon/ArmNeon.td)) provides operations corresponding to AArch64 Advanced SIMD (NEON) intrinsics. NEON vectors are fixed-width 128-bit (16 bytes) or 64-bit (8 bytes) registers operating on elements of various sizes.

### 144.1.2 Dot Product Operations

The most performance-critical NEON ops for ML workloads are dot products:

```mlir
// SDOT: signed 8-bit integer dot product (4 i8 elements per dot)
// Computes: result[i] += dot(a[4i..4i+3], b[4i..4i+3])
// Maps to ARM NEON sdot instruction
%result = arm_neon.2d_sdot %acc, %a, %b
    : vector<2xi32>, vector<8xi8>, vector<8xi8>

// UDOT: unsigned 8-bit dot product
%result_u = arm_neon.2d_udot %acc, %a, %b
    : vector<2xi32>, vector<8xi8>, vector<8xi8>

// BFDOT: BFloat16 dot product (pairs of BF16 → FP32 accumulation)
%result_bf = arm_neon.intr.bfdot %acc, %a, %b
    : vector<2xf32>, vector<4xbf16>, vector<4xbf16>

// BFMMLA: BFloat16 matrix multiply accumulate (2×2 tiles)
// Computes a 2×2 f32 matrix result from 2×4 × 4×2 BF16 matrices
%result_mm = arm_neon.intr.bfmmla %acc, %a, %b
    : vector<4xf32>, vector<8xbf16>, vector<8xbf16>
```

`arm_neon.2d_sdot` maps to AArch64's `sdot v0.2s, v1.8b, v2.8b` — a 4-wide i8 dot product that is fundamental to INT8 quantized inference. For a 128-bit vector `vector<4xi32>`, the 4D version is available via `arm_neon.intr.sdot`.

### 144.1.3 Other NEON Operations

```mlir
// Table lookup (permute/shuffle elements via index table)
%shuffled = arm_neon.intr.tbl1 %table, %indices
    : vector<8xi8>, vector<8xi8> -> vector<8xi8>
%shuffled2 = arm_neon.intr.tbl2 %table0, %table1, %indices
    : vector<8xi8>, vector<8xi8>, vector<8xi8> -> vector<8xi8>

// SMMLA: signed 8-bit matrix multiply accumulate (4×4 i8 → i32)
%acc2 = arm_neon.intr.smmla %acc, %a, %b
    : vector<4xi32>, vector<16xi8>, vector<16xi8>

// FMLAL: f16 multiply add (pairs of f16 → f32)
%res = arm_neon.intr.fmlal %a, %b, %c
    : vector<2xf32>, vector<2xf16>, vector<2xf16>
```

### 144.1.4 Lowering

```bash
# Lower arm_neon dialect to LLVM AArch64 intrinsics
mlir-opt --convert-arm-neon-to-llvm input.mlir
```

The pass replaces `arm_neon.*` ops with corresponding `llvm.intr.aarch64.neon.*` intrinsics, which the AArch64 backend then maps to machine instructions. The LLVM intrinsic for `arm_neon.intr.sdot` is `llvm.aarch64.neon.sdot`.

---

## 144.2 `arm_sve`

### 144.2.1 Scalable Vector Extension

The `arm_sve` dialect ([`mlir/include/mlir/Dialect/ArmSVE/ArmSVEDialect.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/ArmSVE/ArmSVEDialect.td)) provides SVE-specific ops that cannot be expressed cleanly as generic `vector` ops with scalable types.

SVE's key feature is predicated (masked) execution: every SVE instruction can take a predicate register that enables/disables each lane. This enables auto-vectorization of loops with irregular trip counts without handling remainder iterations separately.

### 144.2.2 SVE Dot Products and Matrix Ops

```mlir
// SDOT/UDOT for scalable vectors
%result = arm_sve.intr.sdot %acc, %a, %b
    : vector<[4]xi32>, vector<[16]xi8>, vector<[16]xi8>

%result_u = arm_sve.intr.udot %acc, %a, %b
    : vector<[4]xi32>, vector<[16]xi8>, vector<[16]xi8>

// SMMLA / UMMLA: 8-bit matrix multiply-accumulate for SVE2
%acc2 = arm_sve.intr.smmla %acc, %a, %b
    : vector<[4]xi32>, vector<[16]xi8>, vector<[16]xi8>

// BFDOT: BFloat16 dot product (scalable)
%res_bf = arm_sve.intr.bfdot %acc, %a, %b
    : vector<[4]xf32>, vector<[8]xbf16>, vector<[8]xbf16>
```

### 144.2.3 Predicate Operations

```mlir
// Convert from svbool (the all-element predicate) to a specific predicate
%pred_f32 = arm_sve.convert_from_svbool %all_true
    : vector<[16]xi1> -> vector<[4]xi1>  // [16]xi1 = one bit per byte; [4]xi1 = one bit per 4 bytes (f32)

// Convert to svbool
%all_true = arm_sve.convert_to_svbool %pred_f32 : vector<[4]xi1> -> vector<[16]xi1>

// Create predicate from count (whilelt: true for lanes < n)
%pred = arm_sve.intr.whilelt %start, %end : i64 -> vector<[4]xi1>

// Predicated load/store
%vals = arm_sve.intr.ld1 %base_ptr, %pred : vector<[4]xi1>, f32 -> vector<[4]xf32>
arm_sve.intr.st1 %vals, %pred, %base_ptr : vector<[4]xf32>, vector<[4]xi1>, f32
```

### 144.2.4 Structured Zips for ACLE Patterns

```mlir
// Interleave elements from two vectors (ZIP1/ZIP2)
%zip_result = arm_sve.zip.x2 %a, %b : vector<[4]xf32>
    -> (vector<[4]xf32>, vector<[4]xf32>)

// Deinterleave
%uzp_result = arm_sve.uzp.x2 %a, %b : vector<[4]xf32>
    -> (vector<[4]xf32>, vector<[4]xf32>)
```

These correspond to SVE2's ZIP and UZP instructions used in AoS↔SoA data layout conversion.

### 144.2.5 Lowering

```bash
mlir-opt --convert-arm-sve-to-llvm input.mlir
```

SVE ops lower to LLVM scalable vector intrinsics (`llvm.aarch64.sve.*`). The scalable vector type `vector<[4]xf32>` becomes `<vscale x 4 x float>` in LLVM IR. The SVE backend then generates predicated vector instructions.

---

## 144.3 `arm_sme`

### 144.3.1 Scalable Matrix Extension

The `arm_sme` dialect ([`mlir/include/mlir/Dialect/ArmSME/IR/ArmSME.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/ArmSME/IR/ArmSME.td)) represents operations targeting ARM's Scalable Matrix Extension — a separate processing unit introduced with Cortex-X3 that can operate on matrix tiles stored in a special register called ZA (the accumulator array).

SME provides a hardware matrix accumulator that is independent of the SVE vector registers, enabling sustained matrix multiply performance by overlapping matrix loads with outer-product accumulation.

### 144.3.2 ZA Array Management

```mlir
// Enable/disable ZA register access (required around SME operations)
arm_sme.za_enable
arm_sme.za_disable

// Query the streaming vector length (SVL — size of ZA matrix tiles)
%svl_bytes = arm_sme.streaming_vl sz_b : index  // bytes per SME row
%svl_elems = arm_sme.streaming_vl sz_s : index  // f32 elements per SME row

// Get the ZA tile ID for indexing
%tile_id = arm_sme.get_tile_id : i32
```

`arm_sme.za_enable` / `arm_sme.za_disable` correspond to the `SMSTART ZA` / `SMSTOP ZA` instructions and must bracket any code using ZA register operations.

### 144.3.3 Tile Load and Store

```mlir
// Load a matrix tile from memory into ZA (row-by-row)
%tile = arm_sme.tile_load %base[%row_base, %col_base]
    layout<row> : memref<?x?xf32>, vector<[4]x[4]xf32>

// Store a ZA tile back to memory
arm_sme.tile_store %tile, %base[%row_base, %col_base]
    layout<row> : vector<[4]x[4]xf32>, memref<?x?xf32>

// Streaming mode load
%tile2 = arm_sme.tile_load %svmem[%i, %j]
    layout<column> : memref<?x?xf32, #arm_sme.za_storage>, vector<[4]x[4]xf32>
```

The `layout<row>` / `layout<column>` attribute controls the data layout in memory. ZA stores tiles in row-major or column-major order, enabling efficient BLAS-level matrix multiply without explicit transposition.

### 144.3.4 Outer Product (MOP)

```mlir
// SME outer product: tile[i][j] += a[i] * b[j]
// FP32 outer product (fmopa)
%result = arm_sme.outerproduct %a, %b acc(%acc)
    : vector<[4]xf32>, vector<[4]xf32>

// FP64 outer product (fmopa)
%result_d = arm_sme.outerproduct %a_d, %b_d acc(%acc_d)
    : vector<[2]xf64>, vector<[2]xf64>

// BF16 outer product (bfmopa — BF16 inputs, FP32 accumulation)
%result_bf = arm_sme.outerproduct %a_bf, %b_bf acc(%acc_f32)
    {kind = #arm_sme.outer_product_kind<add>}
    : vector<[8]xbf16>, vector<[8]xbf16>

// Predicated outer product (mask out certain rows/cols)
%pred_result = arm_sme.outerproduct %a, %b, %row_pred, %col_pred acc(%acc)
    : vector<[4]xf32>, vector<[4]xf32>
```

`arm_sme.outerproduct` maps to SME's `FMOPA` (floating outer product and accumulate) instruction, which computes the outer product of two scalable vectors and accumulates the result into a ZA tile. This is the inner-most operation in SME-optimized GEMM.

### 144.3.5 Lowering

```bash
mlir-opt --convert-arm-sme-to-llvm input.mlir
```

SME ops lower through LLVM AArch64 intrinsics (`llvm.aarch64.sme.*`). The LLVM AArch64 backend generates streaming-mode SVE and SME instructions. The lowering also handles calling convention details: streaming mode must be entered (`SMSTART SM`) before SME code and exited (`SMSTOP SM`) on return.

---

## 144.4 `x86vector`

### 144.4.1 Overview

The `x86vector` dialect ([`mlir/include/mlir/Dialect/X86Vector/X86Vector.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/X86Vector/X86Vector.td)) provides x86-specific vector instructions not covered by the generic `vector` dialect — primarily AVX, AVX2, and AVX-512 specific operations.

### 144.4.2 AVX Dot Products

```mlir
// AVX dpps: dot product of two float vectors with selector mask
// Computes partial dot products based on 8-bit selector
%dot = x86vector.avx.intr.dot.f32 %a, %b, %selector
    : vector<8xf32>, vector<8xf32>, i8

// AVX dppd: double precision dot product
%dot_d = x86vector.avx.intr.dot.f64 %a, %b, %selector
    : vector<4xf64>, vector<4xf64>, i8
```

### 144.4.3 AVX-512 Compress and Expand

```mlir
// AVX-512 compress store: pack non-masked elements to memory
x86vector.avx512.mask.compress %values, %mask, %base, %idx
    : vector<16xf32>, vector<16xi1>, memref<?xf32>, index

// AVX-512 expand load: load into masked positions
%result = x86vector.avx512.mask.expand %base, %idx, %pass_thru, %mask
    : memref<?xf32>, index, vector<16xf32>, vector<16xi1> -> vector<16xf32>
```

These map to AVX-512's `VPCOMPRESSPS`/`VPEXPANDPS` — essential for sparse workloads and conditional filtering.

### 144.4.4 VP2INTERSECT (Set Intersection)

```mlir
// VP2INTERSECT: compute the intersection of two integer sets
// Returns masks indicating which elements of each input are in the intersection
%mask0, %mask1 = x86vector.avx512.vp2intersect %a, %b
    : vector<16xi32>, vector<16xi32>
    -> (vector<16xi1>, vector<16xi1>)
```

`VP2INTERSECT` is a Tiger Lake / Ice Lake instruction that finds which elements of two sorted integer vectors share values. Used in database workloads (join operations) and sparse linear algebra (intersection of index arrays).

### 144.4.5 BF16 Conversion

```mlir
// VCVTNEPS2BF16: convert f32 to bf16 (round to nearest even, truncate)
%bf16_128 = x86vector.avx512.intr.vcvtneps2bf16.128 %f32_128
    : vector<4xf32> -> vector<8xbf16>

%bf16_256 = x86vector.avx512.intr.vcvtneps2bf16.256 %f32_256
    : vector<8xf32> -> vector<8xbf16>

%bf16_512 = x86vector.avx512.intr.vcvtneps2bf16.512 %f32_512
    : vector<16xf32> -> vector<16xbf16>

// DPBF16PS: BF16 dot product with FP32 accumulation (AMX / AVX512-BF16)
%acc = x86vector.avx512.bf16.intr.dpbf16ps.256 %a, %b, %c
    : vector<8xbf16>, vector<8xbf16>, vector<4xf32> -> vector<4xf32>
```

### 144.4.6 Lowering

```bash
mlir-opt --convert-x86vector-to-llvm input.mlir
```

x86vector ops lower to LLVM x86 intrinsics (`llvm.x86.avx512.*`, `llvm.x86.avx.*`). The LLVM x86 backend then selects specific machine instructions.

---

## 144.5 `amdgpu`

### 144.5.1 AMD GPU Matrix Operations

The `amdgpu` dialect ([`mlir/include/mlir/Dialect/AMDGPU/IR/AMDGPUDialect.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/AMDGPU/IR/AMDGPUDialect.td)) provides AMD GPU-specific ops including the MFMA (Matrix Fused Multiply-Add) instruction family that powers AMD's matrix compute acceleration on GCN and CDNA architectures.

### 144.5.2 MFMA Matrix Operations

```mlir
// MFMA FP32 32×32×1: 64-wide wave, each thread holds a fragment
%c = amdgpu.mfma %a, %b, %c_init {
  m = 32 : i32, n = 32 : i32, k = 1 : i32, blocks = 1 : i32,
  abid = 0 : i32, cbsz = 0 : i32
} : vector<2xf32>, vector<2xf32>, vector<16xf32> -> vector<16xf32>

// MFMA FP16 16×16×16
%c2 = amdgpu.mfma %a2, %b2, %c_init2 {
  m = 16 : i32, n = 16 : i32, k = 16 : i32, blocks = 1 : i32,
  abid = 0 : i32, cbsz = 0 : i32
} : vector<4xf16>, vector<4xf16>, vector<4xf32> -> vector<4xf32>

// WMMA (RDNA3+): Wave Matrix Multiply-Accumulate
%c3 = amdgpu.wmma %a3, %b3, %c_init3 {subwordOffset = 0 : i32, unsignedA = false, unsignedB = false, clamp = false}
    : vector<16xf16>, vector<16xf16>, vector<8xf32> -> vector<8xf32>
```

MFMA instructions execute across a wavefront (64 threads on GCN, 32 on RDNA): each thread holds a fragment of A, B, and C. The entire wavefront collaborates to compute a matrix multiply tile. The `m×n×k` parameters specify the tile size; `blocks` allows multiple accumulators per wave.

### 144.5.3 Buffer Operations

```mlir
// Raw buffer load (AMDGPU-specific buffered loads with bounds checking)
%val = amdgpu.raw_buffer_load {boundsCheck = true, indexOffset = 0 : i32}
    %rsrc, %offset, %stride, %voffset : !llvm.ptr, i32, i32, i32 -> f32

// Raw buffer store
amdgpu.raw_buffer_store {boundsCheck = true}
    %val, %rsrc, %offset, %stride, %voffset : f32, !llvm.ptr, i32, i32, i32

// LDS (local data store) barrier
amdgpu.lds_barrier
```

Buffer operations use AMDGPU's buffer descriptor resource (`rsrc`) which encodes base address, size, and format in a 4-DWORD descriptor. They enable hardware-assisted out-of-bounds checking without explicit compares.

### 144.5.4 Wave Operations

```mlir
// Warp reduction (DPP-based butterfly reduction across a wave)
%reduced = amdgpu.dpp "row_xmask:0" %val {row_mask = 0xf, bank_mask = 0xf, bound_ctrl = true}
    : vector<2xf32>

// Read lane from another wave position
%cross_lane = amdgpu.readlane %val, %lane_id : f32

// Lane permutation
%permuted = amdgpu.permlane16 %val, %vdst_in, %sel0, %sel1
    {fi = false, bc = false} : f32
```

### 144.5.5 Lowering

```bash
mlir-opt --convert-amdgpu-to-rocdl input.mlir
```

AMDGPU ops lower to ROCDL intrinsics (`rocdl.mfma.*`, `rocdl.raw.buffer.*`), which in turn lower to the AMDGPU LLVM backend via `--convert-rocdl-to-llvm`.

---

## 144.6 `rocdl`

### 144.6.1 ROCDL as the AMD Hardware Dialect

The `rocdl` dialect ([`mlir/include/mlir/Dialect/LLVMIR/ROCDLDialect.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/LLVMIR/ROCDLDialect.td)) sits at the same level as `nvvm` for AMD: it provides AMD GPU intrinsics below the `amdgpu` dialect, serving as the penultimate step before the LLVM AMDGPU backend.

### 144.6.2 Thread and Block Indices

```mlir
// Work item IDs (thread indices within a workgroup)
%tid_x = rocdl.workitem.id.x : i32
%tid_y = rocdl.workitem.id.y : i32
%tid_z = rocdl.workitem.id.z : i32

// Workgroup IDs (block indices within grid)
%bid_x = rocdl.workgroup.id.x : i32
%bid_y = rocdl.workgroup.id.y : i32
%bid_z = rocdl.workgroup.id.z : i32

// Workgroup dimensions
%bdim_x = rocdl.workgroup.dim.x : i32
%bdim_y = rocdl.workgroup.dim.y : i32

// Flat work item ID within grid
%flat_id = rocdl.workitem.flat.id : i32
```

### 144.6.3 Memory and Synchronization

```mlir
// LDS (local data store = AMD shared memory) barrier
rocdl.barrier

// S_BARRIER: scalar unit barrier for inter-workgroup sync
rocdl.s.barrier

// LDS permute (cross-lane data exchange using LDS)
%permuted = rocdl.ds_bpermute %lane_offset, %val : i32, f32

// Raw buffer operations (see amdgpu dialect for higher-level forms)
%val = rocdl.raw.buffer.load %rsrc, %offset, %soffset, %glc_slc
    : !llvm.ptr, i32, i32, i32 -> f32
rocdl.raw.buffer.store %val, %rsrc, %offset, %soffset, %glc_slc
    : f32, !llvm.ptr, i32, i32, i32
```

### 144.6.4 MFMA Intrinsics

```mlir
// Direct MFMA intrinsic (lower-level than amdgpu.mfma)
%c = rocdl.mfma.f32.32x32x1f32 %a, %b, %c_init, %cbsz, %abid, %blgp
    : f32, f32, vector<32xf32>, i32, i32, i32 -> vector<32xf32>

%c2 = rocdl.mfma.f32.16x16x16f16 %a2, %b2, %c_init2, %cbsz, %abid, %blgp
    : vector<4xf16>, vector<4xf16>, vector<4xf32>, i32, i32, i32 -> vector<4xf32>

// WMMA (RDNA3)
%c3 = rocdl.wmma.f32.16x16x16.f16 %a3, %b3, %c_init3, %opsel
    : vector<16xf16>, vector<16xf16>, vector<8xf32>, i1 -> vector<8xf32>
```

### 144.6.5 Lowering

```bash
mlir-opt --convert-rocdl-to-llvm input.mlir
```

ROCDL ops lower to `llvm.intr.amdgcn.*` intrinsics. The LLVM AMDGPU backend then generates GFX ISA machine code. The complete AMD GPU pipeline:

```bash
mlir-opt \
  --gpu-to-rocdl="chipset=gfx90a" \
  --convert-amdgpu-to-rocdl \
  --convert-rocdl-to-llvm \
  --reconcile-unrealized-casts \
  input.mlir | \
mlir-translate --mlir-to-rocdlir | \
llc -march=amdgcn -mcpu=gfx90a --filetype=obj -o kernel.o
```

---

## 144.7 Hardware Dialect Comparison

| Dialect | Architecture | Key Operations | Primary Use Case |
|---|---|---|---|
| `arm_neon` | AArch64 NEON | `sdot`, `udot`, `bfmmla`, `tbl` | INT8/BF16 ML inference on mobile/server |
| `arm_sve` | AArch64 SVE/SVE2 | scalable dot, predicate ops, zip | Variable-length SIMD on Neoverse/Cortex-X |
| `arm_sme` | AArch64 SME | `outerproduct`, tile load/store | Matrix GEMM on Cortex-X3+, Grace |
| `x86vector` | x86 AVX/AVX-512 | compress, expand, vp2intersect, bf16 cvt | High-end x86 server (Sapphire Rapids+) |
| `amdgpu` | AMD GCN/CDNA/RDNA | MFMA, WMMA, buffer ops | AMD GPU matrix compute (ROCm) |
| `rocdl` | AMD GCN/CDNA/RDNA | workitem IDs, barrier, MFMA intrinsics | Low-level AMD GPU code |

---

## Chapter 144 Summary

- The `arm_neon` dialect provides fixed-width NEON intrinsics for AArch64: `arm_neon.2d_sdot`/`udot` for i8 dot products (ML INT8 inference), `arm_neon.intr.bfmmla` for BF16 2×2 matrix multiply, `arm_neon.intr.tbl1/2` for table lookup. Lowered by `--convert-arm-neon-to-llvm`.
- The `arm_sve` dialect provides scalable vector ops for SVE: scalable dot products, `whilelt` predicate generation, predicated `ld1`/`st1`, and `zip`/`uzp` data reorganization. Scalable vectors use `vector<[4]xf32>` notation; lowered by `--convert-arm-sve-to-llvm`.
- The `arm_sme` dialect targets the SME accumulator array (ZA): `arm_sme.outerproduct` maps to `FMOPA`, `arm_sme.tile_load`/`store` access ZA rows, `arm_sme.za_enable`/`za_disable` bracket SME usage. SME enables persistent matrix accumulation independent of SVE vector registers.
- The `x86vector` dialect provides AVX/AVX-512 specific ops: `avx.intr.dot.f32` for partial dot products, `avx512.mask.compress`/`expand` for gather/scatter, `avx512.vp2intersect` for set intersection, `avx512.intr.vcvtneps2bf16.*` for f32→bf16 conversion. Lowered by `--convert-x86vector-to-llvm`.
- The `amdgpu` dialect provides `amdgpu.mfma` for AMD's MFMA matrix instruction family (GCN/CDNA) and `amdgpu.wmma` for RDNA3 wave matrix multiply. Buffer operations (`raw_buffer_load/store`) provide hardware-checked global memory access. Lowered via `--convert-amdgpu-to-rocdl`.
- The `rocdl` dialect is AMD's hardware intrinsic layer (equivalent to `nvvm` for NVIDIA): `rocdl.workitem.id.x` provides thread IDs, `rocdl.barrier` provides workgroup sync, and `rocdl.mfma.*` provides direct MFMA intrinsics. Lowered by `--convert-rocdl-to-llvm`.
