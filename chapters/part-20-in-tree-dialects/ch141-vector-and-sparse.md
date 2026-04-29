# Chapter 141 — Vector and Sparse

*Part XX — In-Tree Dialects*

Two dialects in MLIR address the challenge of efficient computation on structured but irregular data at opposite ends of the density spectrum: the `vector` dialect abstracts SIMD parallelism in a target-independent form, bridging the gap between `linalg`'s structured loops and hardware vector instructions; the `sparse_tensor` dialect enables automatic generation of efficient sparse code from dense algorithmic descriptions, transforming `linalg.generic` operations on annotated sparse tensors into explicit loops over compressed storage formats. Together they cover the majority of performance-critical computational patterns in modern ML and scientific computing workloads.

---

## 141.1 The `vector` Dialect Overview

### 141.1.1 Design Goals

The `vector` dialect ([`mlir/include/mlir/Dialect/Vector/IR/VectorOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Vector/IR/VectorOps.td)) provides a target-independent representation of SIMD computation. Its goals:

1. **Target independence**: express vectorized computations without committing to x86 SSE/AVX, AArch64 NEON/SVE, or RISC-V RVV
2. **Multi-dimensional**: vectors are n-dimensional (`vector<4x4xf32>`) rather than 1-dimensional as in LLVM IR
3. **Masking**: first-class support for predicated execution (`vector<4xi1>` masks)
4. **Transfer semantics**: `vector.transfer_read`/`write` bridge between non-contiguous memory and vectors
5. **Composition**: vector ops compose with each other and with `linalg`, `affine`, and `scf`

The `vector` dialect is the output of `linalg` vectorization and the input to hardware-specific lowering (`--lower-vector-to-llvm`, `--convert-arm-neon-to-llvm`, etc.).

### 141.1.2 VectorType

```mlir
vector<4xf32>           // 1D vector, 4 f32 elements (like __m128)
vector<8xi32>           // 1D vector, 8 i32 elements
vector<4x4xf32>         // 2D vector, 4×4 f32 elements (register tile)
vector<4x4x4xf32>       // 3D vector
vector<[4]xf32>         // scalable vector: 4 × vscale elements (SVE/RVV)
vector<[4]x[4]xf32>     // 2D scalable vector
```

The `[4]` notation denotes a scalable vector: the actual size is `4 × vscale` where `vscale` is determined at runtime from the hardware configuration. This enables writing code that compiles for SVE (any vector length that is a multiple of 128 bits) or RVV (any supported LMUL).

Multi-dimensional vectors are not arrays — they are register-level values that correspond to small register tiles used in high-performance kernels.

---

## 141.2 Core Vector Ops

### 141.2.1 `vector.transfer_read` and `vector.transfer_write`

These are the primary interface between vector operations and memory:

```mlir
// Read a vector from a memref (handles out-of-bounds with padding)
%v = vector.transfer_read %memref[%i, %j], %padding_val
    {in_bounds = [true, true], permutation_map = affine_map<(d0, d1) -> (d0, d1)>}
    : memref<?x?xf32>, vector<4x8xf32>

// Read with masking
%v_masked = vector.transfer_read %memref[%i, %j], %padding_val, %mask
    {in_bounds = [false, false]}
    : memref<?x?xf32>, vector<4x8xf32>

// Write a vector to a memref
vector.transfer_write %v, %memref[%i, %j]
    {in_bounds = [true, true]}
    : vector<4x8xf32>, memref<?x?xf32>

// Transposed read (reads rows as columns)
%vt = vector.transfer_read %memref[%i, %j], %zero
    {permutation_map = affine_map<(d0, d1) -> (d1, d0)>}
    : memref<?x?xf32>, vector<8x4xf32>
```

**`in_bounds`**: if `true` for a dimension, the op asserts the access is within bounds — no masking is needed, enabling simpler codegen. If `false`, the op generates masked loads/stores using the `padding_val` for out-of-bounds positions.

**`permutation_map`**: an affine map from memory dimensions to vector dimensions. Used for transposed loads (register blocking), broadcast reads (repeat a row), and rank-reducing reads (extract one row/column into a 1D vector).

The transfer ops are the cornerstone of `linalg` vectorization: every `affine.load`/`memref.load` in a vectorized linalg op becomes a `vector.transfer_read`.

### 141.2.2 `vector.load` and `vector.store`

The simpler siblings of transfer ops — direct contiguous vector load/store:

```mlir
// Load contiguous vector from memory (must be in-bounds and contiguous)
%v = vector.load %memref[%i] : memref<?xf32>, vector<8xf32>

// Store vector to memory
vector.store %v, %memref[%i] : vector<8xf32>, memref<?xf32>
```

`vector.load`/`store` are for the simple case: contiguous, in-bounds, no transposition. They lower directly to LLVM `load <8 x float>` / `store <8 x float>` instructions without the overhead of masked conditional stores.

### 141.2.3 `vector.gather` and `vector.scatter`

```mlir
// Gather: load elements at addresses base[indices[i]] for each i
%gathered = vector.gather %base[%c0][%indices], %pass_thru, %mask
    : memref<?xf32>, vector<8xi32>, vector<8xf32>, vector<8xi1> into vector<8xf32>

// Scatter: store elements to addresses base[indices[i]] for each i
vector.scatter %base[%c0][%indices], %mask, %values
    : memref<?xf32>, vector<8xi32>, vector<8xi1>, vector<8xf32>
```

Gather/scatter are the sparse-access analogs of load/store. They map to hardware gather instructions (AVX-512 `vgatherdps`, AArch64 `LD1` with vector indices) when available, or to scalar loops otherwise.

### 141.2.4 `vector.contract`

`vector.contract` is the vector analog of `linalg.generic` — a parameterized contraction with indexing maps and reduction:

```mlir
// Matrix multiply via vector.contract (4x4xf32 × 4x8xf32 → 4x8xf32)
%result = vector.contract {
  indexing_maps = [
    affine_map<(m, n, k) -> (m, k)>,   // A[m,k]
    affine_map<(m, n, k) -> (k, n)>,   // B[k,n]
    affine_map<(m, n, k) -> (m, n)>    // C[m,n]
  ],
  iterator_types = ["parallel", "parallel", "reduction"],
  kind = #vector.kind<add>
} %A, %B, %C : vector<4x4xf32>, vector<4x8xf32> into vector<4x8xf32>

// Dot product
%dot = vector.contract {
  indexing_maps = [affine_map<(d0) -> (d0)>, affine_map<(d0) -> (d0)>, affine_map<() -> ()>],
  iterator_types = ["reduction"],
  kind = #vector.kind<add>
} %va, %vb, %acc : vector<8xf32>, vector<8xf32> into f32
```

`kind` specifies the reduction operation: `add` (dot/matmul), `mul` (product contraction), `minsi`/`maxsi`, etc.

`vector.contract` is the output of `linalg.matmul` vectorization. It lowers to:
- `llvm.intr.matrix.multiply` for full LLVM matrix intrinsics
- AVX-512 VNNI for integer dot products
- AArch64 `FMLA` instructions via the NEON/SVE lowering path
- Scalar loops via `--vector-to-scf`

### 141.2.5 `vector.multi_reduction` and `vector.reduction`

```mlir
// Reduce along specified dimensions
%reduced = vector.multi_reduction <add>, %vec, %init, [1, 2]
    : vector<4x4x4xf32> to vector<4xf32>

// Reduce entire vector to scalar
%sum = vector.reduction <add>, %vec : vector<8xf32> into f32
%max = vector.reduction <maxf>, %vec : vector<8xf32> into f32
%min = vector.reduction <minsi>, %vec : vector<8xi32> into i32
```

Reduction kinds: `add`, `mul`, `minui`, `minsi`, `minf`, `maxui`, `maxsi`, `maxf`, `and`, `or`, `xor`.

### 141.2.6 Elementwise and Extraction Ops

```mlir
// Splat (broadcast scalar to vector)
%splat = vector.splat %scalar : vector<8xf32>

// Extract element or sub-vector
%elem = vector.extract %v[%i] : f32 from vector<8xf32>       // dynamic index
%sub  = vector.extract %v[2] : vector<4xf32> from vector<8xf32>  // static slice

// Insert element or sub-vector
%new_v = vector.insert %elem, %v[%i] : f32 into vector<8xf32>
%new_v2 = vector.insert %sub, %v[1] : vector<4xf32> into vector<8xf32>

// Broadcast (repeat lower-rank to higher-rank)
%mat = vector.broadcast %row : vector<4xf32> to vector<4x4xf32>

// Shuffle (permute elements)
%shuffled = vector.shuffle %v1, %v2 [0, 2, 4, 6] : vector<4xf32>, vector<4xf32>

// Outer product (AXPY-style)
%outer = vector.outerproduct %a, %b, %acc
    : vector<4xf32>, vector<4xf32>  into vector<4x4xf32>
```

`vector.outerproduct` computes `C += outer(a, b)` — the outer product accumulation. This is the fundamental operation in many BLAS-style micro-kernels: repeatedly compute outer products of short vectors to form a matrix tile.

### 141.2.7 FMA and Masked Operations

```mlir
// Fused multiply-add (a * b + c)
%fma = vector.fma %a, %b, %c : vector<8xf32>

// Masked store (write only where mask is true)
vector.maskedstore %base[%idx], %mask, %values
    : memref<?xf32>, vector<8xi1>, vector<8xf32>

// Masked load (read where mask is true, use pass_thru where false)
%v = vector.maskedload %base[%idx], %mask, %pass_thru
    : memref<?xf32>, vector<8xi1>, vector<8xf32> into vector<8xf32>

// Compress store (scatter non-masked elements contiguously)
vector.compressstore %base[%idx], %mask, %values
    : memref<?xf32>, vector<8xi1>, vector<8xf32>

// Expand load (gather elements filling only masked positions)
%v = vector.expandload %base[%idx], %mask, %pass_thru
    : memref<?xf32>, vector<8xi1>, vector<8xf32> into vector<8xf32>
```

---

## 141.3 Vector Unrolling and Target Lowering

### 141.3.1 Unrolling to Target Width

Hardware has fixed vector widths. A `vector<16xf32>` operation must be unrolled to two `vector<8xf32>` operations on an AVX2 target:

```bash
# Unroll vectors to match hardware width
mlir-opt \
  --vector-unroll-target-info="target-vector-bitwidth=256" \
  input.mlir
```

The `VectorUnrollOpInterface` enables any op to specify how it should be unrolled. The pass walks all vector ops and splits those wider than the target into multiple narrower ops.

### 141.3.2 `--vector-to-scf`

For targets without native SIMD support:

```bash
# Lower all vector ops to scalar scf loops
mlir-opt --convert-vector-to-scf input.mlir
```

This converts `vector.transfer_read` to a loop over elements with scalar `memref.load`, `vector.reduction` to a loop with `arith` accumulation, etc. Useful for verification, testing, or embedding on scalar cores.

### 141.3.3 `--lower-vector-to-llvm`

The primary hardware lowering path:

```bash
mlir-opt \
  --lower-vector-to-llvm="enable-avx512=true" \
  input.mlir
```

Options:
- `enable-avx512`: use AVX-512 instructions for x86
- `enable-arm-neon`: use NEON intrinsics
- `enable-arm-sve`: use scalable vector intrinsics
- `enable-x86vector`: use x86vector dialect extensions
- `reassociate-fp-reductions`: allow reordering float reductions for vectorization

The lowering maps:
- `vector.transfer_read/write` → `llvm.intr.masked.load/store` (with masking) or direct `llvm.load/store`
- `vector.reduction` → `llvm.intr.experimental.vector.reduce.*`
- `vector.fma` → `llvm.intr.fma`
- `vector.shuffle` → `llvm.shufflevector`

### 141.3.4 Scalable Vectors

Scalable vectors (`vector<[4]xf32>`) target SVE and RVV:

```mlir
// Read with scalable vector (unknown width at compile time)
%vscale = vector.vscale : index
%n = arith.muli %vscale, %c4 : index  // actual width = 4 * vscale

// Scalable load
%sv = vector.transfer_read %memref[%i], %zero
    : memref<?xf32>, vector<[4]xf32>

// Scalable FMA
%result = vector.fma %a, %b, %c : vector<[4]xf32>
```

After `--lower-vector-to-llvm` with SVE enabled, `vector<[4]xf32>` becomes `<vscale x 4 x float>` in LLVM IR, which SVE-enabled backends lower to scalable vector registers.

---

## 141.4 The `sparse_tensor` Dialect

### 141.4.1 Sparse Tensor Encoding

The `sparse_tensor` dialect ([`mlir/include/mlir/Dialect/SparseTensor/IR/SparseTensor.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/SparseTensor/IR/SparseTensor.td)) enables expressing sparse matrix algorithms using the same `linalg.generic` framework used for dense computation, annotated with a sparse encoding attribute.

A sparse tensor is declared by adding a `SparseTensorEncodingAttr` to a regular tensor type:

```mlir
// Define sparse encodings
#CSR = #sparse_tensor.encoding<{
  map = (d0, d1) -> (d0 : dense, d1 : compressed)
}>

#CSC = #sparse_tensor.encoding<{
  map = (d0, d1) -> (d1 : dense, d0 : compressed)
}>

#COO = #sparse_tensor.encoding<{
  map = (d0, d1) -> (d0 : compressed(nonunique), d1 : singleton)
}>

#BSR = #sparse_tensor.encoding<{
  map = (d0, d1) -> (d0 floordiv 2 : dense, d1 floordiv 2 : dense,
                     d0 mod 2 : dense, d1 mod 2 : dense)
  posWidth = 32,
  crdWidth = 32
}>
```

**Dimension levels:**
- `dense`: all positions exist (like a dense array)
- `compressed`: only non-empty positions are stored (like CSR/CSC inner dimension)
- `compressed(nonunique)`: like `compressed` but allows duplicate coordinates (for COO)
- `singleton`: singleton coordinate list (COO-style per element)

### 141.4.2 Sparse Tensor Operations

```mlir
// Convert between formats (dense ↔ sparse, sparse ↔ sparse)
%csr = sparse_tensor.convert %dense : tensor<100x100xf64> to tensor<100x100xf64, #CSR>
%csc = sparse_tensor.convert %csr : tensor<100x100xf64, #CSR> to tensor<100x100xf64, #CSC>
%dense2 = sparse_tensor.convert %csr : tensor<100x100xf64, #CSR> to tensor<100x100xf64>

// Access internal storage arrays
%values = sparse_tensor.values %csr : tensor<100x100xf64, #CSR> to memref<?xf64>
%col_idx = sparse_tensor.coordinates %csr {level = 1 : index}
    : tensor<100x100xf64, #CSR> to memref<?xi64>
%row_ptr = sparse_tensor.positions %csr {level = 0 : index}
    : tensor<100x100xf64, #CSR> to memref<?xi64>

// Iterate over non-zeros
sparse_tensor.foreach in %csr : tensor<100x100xf64, #CSR> do {
  ^bb0(%row: index, %col: index, %val: f64):
    // process each non-zero element at (row, col) with value val
}

// Get number of stored elements
%nnz = sparse_tensor.number_of_entries %csr : tensor<100x100xf64, #CSR>
```

### 141.4.3 Sparse Compilation with `--sparsification`

The key insight of MLIR's sparse tensor support: write the algorithm once in `linalg.generic` form, and `--sparsification` automatically generates efficient loops that skip zeros:

```mlir
// Sparse matrix-vector multiplication (SpMV): y = A × x
// A is sparse CSR, x and y are dense
func.func @spmv(%A: tensor<100x100xf64, #CSR>,
                %x: tensor<100xf64>,
                %y_init: tensor<100xf64>) -> tensor<100xf64> {
  %result = linalg.generic {
    indexing_maps = [
      affine_map<(d0, d1) -> (d0, d1)>,   // A[row, col]
      affine_map<(d0, d1) -> (d1)>,         // x[col]
      affine_map<(d0, d1) -> (d0)>          // y[row]
    ],
    iterator_types = ["parallel", "reduction"]
  } ins(%A, %x : tensor<100x100xf64, #CSR>, tensor<100xf64>)
    outs(%y_init : tensor<100xf64>) {
  ^bb0(%a_val: f64, %x_val: f64, %y_val: f64):
    %prod = arith.mulf %a_val, %x_val : f64
    %sum  = arith.addf %y_val, %prod : f64
    linalg.yield %sum : f64
  } -> tensor<100xf64>
  func.return %result : tensor<100xf64>
}
```

After `--sparsification`, this generates:
- An outer loop over rows (dense dimension)
- An inner loop over the CSR `positions`/`coordinates` arrays (only non-zeros)
- No multiplication by zero — structural zeros are skipped

```bash
mlir-opt \
  --sparsification \
  --sparse-reinterpret-map \
  --sparsification-and-bufferization \
  --sparse-tensor-codegen \
  --sparse-storage-specifier-to-llvm \
  --convert-vector-to-llvm \
  --convert-func-to-llvm \
  input.mlir
```

### 141.4.4 Sparsification Algorithm

The `--sparsification` pass ([`mlir/lib/Dialect/SparseTensor/Transforms/Sparsification.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Dialect/SparseTensor/Transforms/Sparsification.cpp)) implements the Bik-Wijshoff algorithm for generating sparse loops:

1. **Level ordering**: determine which loops can be merged into sparse traversals
2. **Access pattern analysis**: for each tensor operand, identify which loops correspond to which storage levels
3. **Loop emission**: generate a nested loop structure where:
   - Dense levels become `scf.for` loops with direct index computation
   - Compressed levels become `scf.for` loops over the `positions`/`coordinates` arrays
   - Singleton levels become direct coordinate lookups
4. **Intersection**: when multiple sparse tensors share a loop dimension, generate intersection logic

For a sparse × dense matmul, the result is:
```mlir
// Generated code (conceptual, after sparsification)
scf.for %row = %c0 to %n_rows step %c1 {
  %row_start = memref.load %rowptr[%row]
  %row_end   = memref.load %rowptr[%row + 1]
  scf.for %nz = %row_start to %row_end step %c1 {
    %col = memref.load %colidx[%nz]
    %a   = memref.load %values[%nz]
    %x_v = memref.load %x[%col]
    %mul = arith.mulf %a, %x_v
    // accumulate into y[%row]
  }
}
```

### 141.4.5 Format Conversion and Storage

```mlir
// Allocate a new sparse tensor (will be filled)
%empty = bufferization.alloc_tensor() : tensor<100x100xf64, #CSR>

// Coordinate-level access to sorted sparse tensor
%t = sparse_tensor.sort_coo %size, %col_idx, %row_idx, %vals
    {nx = 1 : index, ny = 1 : index}
    : index, memref<?xi64>, memref<?xi64>, memref<?xf64>

// Concatenate sparse tensors (useful for assembling block matrices)
%cat = sparse_tensor.concatenate %t1, %t2 {dimension = 0 : index}
    : tensor<50x100xf64, #CSR>, tensor<50x100xf64, #CSR> to tensor<100x100xf64, #CSR>
```

### 141.4.6 Lowering Pipeline Summary

The sparse tensor lowering pipeline has multiple stages:

| Pass | Purpose |
|---|---|
| `--sparsification` | Transform `linalg.generic` with sparse operands to explicit index/value loops |
| `--sparse-reinterpret-map` | Normalize dimension ordering after sparsification |
| `--sparsification-and-bufferization` | Bufferize sparse tensors to explicit storage memrefs |
| `--sparse-tensor-codegen` | Generate CSR/COO/... storage struct code |
| `--sparse-storage-specifier-to-llvm` | Lower storage specifiers to LLVM struct types |
| `--convert-sparse-tensor-to-loops` | Final loop generation for remaining sparse ops |

The key design choice is that sparse encoding is an attribute on the tensor type, not a separate IR construct. This means:
- Dense algorithms can be trivially extended to sparse by annotating operands
- The same transformation infrastructure (fusion, tiling) works on sparse ops
- Mixed sparse/dense computations (SpMM, SpGEMM) are handled uniformly

---

## Chapter 141 Summary

- The `vector` dialect provides target-independent SIMD abstraction with multi-dimensional vector types (`vector<4x4xf32>`) and scalable vectors (`vector<[4]xf32>`) for SVE/RVV.
- `vector.transfer_read`/`vector.transfer_write` are the primary memory interfaces, handling out-of-bounds masking, transposition via `permutation_map`, and rank reduction. `in_bounds` attributes enable simpler codegen for guaranteed in-bounds accesses.
- `vector.contract` is the parameterized vector contraction op (matmul, dot product, etc.), driven by affine indexing maps — the vector-level analog of `linalg.generic`.
- Elementwise ops: `vector.splat`, `vector.broadcast`, `vector.shuffle`, `vector.extract`/`insert`, `vector.outerproduct`. Reduction ops: `vector.reduction`, `vector.multi_reduction`.
- Masking ops (`vector.maskedload`/`store`, `vector.compressstore`, `vector.expandload`) enable predicated SIMD execution for irregular boundaries.
- Lowering paths: `--vector-unroll-target-info` splits to target width, `--convert-vector-to-scf` lowers to scalar loops, `--lower-vector-to-llvm` produces LLVM vector intrinsics.
- The `sparse_tensor` dialect represents compressed sparse formats (CSR, CSC, COO, BSR, etc.) via `SparseTensorEncodingAttr` annotations on tensor types. The same `linalg.generic` body describes both dense and sparse algorithms.
- `--sparsification` is the core pass implementing the Bik-Wijshoff algorithm: it transforms `linalg.generic` with sparse operands into explicit loops over the compressed storage arrays, skipping structural zeros.
- The sparse lowering pipeline (`--sparsification`, `--sparse-tensor-codegen`, `--sparse-storage-specifier-to-llvm`) produces LLVM-ready code that accesses `positions`, `coordinates`, and `values` arrays directly.


---

@copyright jreuben11
