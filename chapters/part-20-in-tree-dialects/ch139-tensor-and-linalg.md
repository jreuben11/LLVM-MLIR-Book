# Chapter 139 — Tensor and Linalg

*Part XX — In-Tree Dialects*

The `tensor` and `linalg` dialects form the computational heart of MLIR for array-processing workloads. The `tensor` dialect provides the value-semantic, aliasing-free view of dense arrays that makes high-level transformations safe and composable. The `linalg` dialect builds on it with a systematic framework for expressing structured computation — from simple elementwise maps to complex convolutions — in a form that exposes rich parallelism and transformation opportunities. Understanding these two dialects thoroughly is the prerequisite for working with any machine learning compiler or high-performance array processing pipeline built on MLIR.

---

## 139.1 The `tensor` Dialect

### 139.1.1 Value Semantics and Immutability

The `tensor` dialect ([`mlir/include/mlir/Dialect/Tensor/IR/TensorOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Tensor/IR/TensorOps.td)) provides operations on the MLIR `tensor` type. Unlike `memref`, a tensor value is conceptually immutable — there is no aliasing between tensor values, and "modifying" a tensor always conceptually produces a new tensor.

This design choice enables:
- Safe reordering of computations (no hidden aliasing hazards)
- Clean fusion analysis (two ops sharing a tensor input have no interference)
- Straightforward SSA use-def for data flow

The cost is that the actual implementation (after bufferization) may copy data. The one-shot bufferization framework in `bufferization` dialect is responsible for eliminating unnecessary copies.

### 139.1.2 `tensor.empty` and `tensor.splat`

```mlir
// Uninitialized tensor — will be filled by subsequent ops
%t = tensor.empty() : tensor<4x4xf32>
%t_dyn = tensor.empty(%n, %m) : tensor<?x?xf32>

// Broadcast scalar to all elements
%ones = tensor.splat %c1_f32 : tensor<4x4xf32>
%dyn_fill = tensor.splat %val : tensor<?xf32>
```

`tensor.empty` creates a tensor with undefined element values — semantically "poison" in the Alive2/MLIR sense. It is not a constant zero or uninitialized memory — no element value can be assumed. The canonical pattern is to pass it as the output operand to `linalg.generic` or `linalg.fill`, which overwrites all elements.

`tensor.splat` broadcasts a single scalar to every position, producing a constant tensor without listing all elements. It folds into `DenseElementsAttr` at compile time for static shapes.

### 139.1.3 Element Access

```mlir
// Extract a single element → scalar
%val = tensor.extract %t[%i, %j] : tensor<4x4xf32>

// Insert a single element → new tensor
%new_t = tensor.insert %val into %t[%i, %j] : tensor<4x4xf32>
```

Note that `tensor.insert` does not mutate `%t` — it produces a new tensor `%new_t` with the same elements as `%t` except at position `[%i, %j]`. In SSA form, `%t` remains valid after this op. After bufferization, this becomes an in-place store if `%t` has no other live uses.

### 139.1.4 Slice Operations

```mlir
// Extract a sub-tensor (slice)
%slice = tensor.extract_slice %t[%r, %c][%h, %w][1, 1]
    : tensor<16x16xf32> to tensor<?x?xf32>

// Insert a sub-tensor into a larger tensor
%updated = tensor.insert_slice %patch into %base[%r, %c][%h, %w][1, 1]
    : tensor<?x?xf32> into tensor<16x16xf32>
```

These mirror `memref.subview` in structure. The three brackets are `[offsets]`, `[sizes]`, `[strides]`. Rank reduction works identically: a size-1 dimension at a fixed offset produces a lower-rank result.

`tensor.extract_slice` and `tensor.insert_slice` form a conjugate pair used throughout tiling transformations. The tiling pattern:

```mlir
// Tile a linalg.generic output:
scf.for %tile_row = %c0 to %rows step %tile_h {
  scf.for %tile_col = %c0 to %cols step %tile_w {
    // Extract input tiles
    %a_tile = tensor.extract_slice %A[%tile_row, 0][%tile_h, %k][1, 1]
        : tensor<?x?xf32> to tensor<?x?xf32>
    %c_tile = tensor.extract_slice %C[%tile_row, %tile_col][%tile_h, %tile_w][1, 1]
        : tensor<?x?xf32> to tensor<?x?xf32>
    // Compute on tile
    %result_tile = linalg.matmul ins(%a_tile, %b_tile : tensor<?x?xf32>, tensor<?x?xf32>)
                                  outs(%c_tile : tensor<?x?xf32>) -> tensor<?x?xf32>
    // Insert result tile back
    %C_updated = tensor.insert_slice %result_tile into %C[%tile_row, %tile_col][...][...]
        : tensor<?x?xf32> into tensor<?x?xf32>
  }
}
```

### 139.1.5 Construction Operations

```mlir
// Build tensor from scalar values (small tensors)
%row = tensor.from_elements %a, %b, %c, %d : tensor<4xf32>
%mat = tensor.from_elements %v00, %v01, %v10, %v11 : tensor<2x2xf32>

// Concatenate tensors along a dimension
%joined = tensor.concat dim(0) %top, %bottom
    : (tensor<4x8xf32>, tensor<4x8xf32>) -> tensor<8x8xf32>

// Pad a tensor with constant or computed values
%padded = tensor.pad %input low[1, 1] high[1, 1] {
  ^bb0(%i: index, %j: index):
    %zero = arith.constant 0.0 : f32
    tensor.yield %zero : f32
} : tensor<6x6xf32> to tensor<8x8xf32>
```

`tensor.pad` is particularly useful for convolution implementations that need zero-padded inputs. The body region computes the padding value for each out-of-bounds position.

### 139.1.6 Shape Operations

```mlir
// Get dynamic dimension size
%sz = tensor.dim %t, %c0 : tensor<?x?xf32>

// Reshape: arbitrary shape change (requires same element count)
%reshaped = tensor.reshape %t(%shape) : (tensor<16xf32>, tensor<2xi64>) -> tensor<?x?xf32>

// Collapse dimensions
%flat = tensor.collapse_shape %mat [[0, 1]] : tensor<4x4xf32> into tensor<16xf32>

// Expand dimensions
%mat2 = tensor.expand_shape %flat [[0, 1]] : tensor<16xf32> into tensor<4x4xf32>
```

---

## 139.2 `linalg.generic`

### 139.2.1 The Universal Structured Computation Op

`linalg.generic` ([`mlir/include/mlir/Dialect/Linalg/IR/LinalgOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Linalg/IR/LinalgOps.td)) is the central abstraction in MLIR for expressing structured computation over arrays. It generalizes every standard linear algebra operation — matrix multiply, convolution, reduction, elementwise operations, batch operations — in a single uniform framework.

The structural invariants that make `linalg.generic` powerful:

1. **No aliasing**: inputs and outputs do not alias
2. **Loop order independence**: computations over parallel dimensions can execute in any order
3. **Single affine index**: each operand element is accessed at most once per loop iteration
4. **Pure body**: the region computes one output element value from corresponding input values

These invariants enable automatic parallelization, vectorization, tiling, and fusion.

### 139.2.2 Anatomy of `linalg.generic`

```mlir
%result = linalg.generic {
  // One affine map per operand (inputs + outputs combined)
  indexing_maps = [
    affine_map<(m, n, k) -> (m, k)>,   // A: read A[m,k]
    affine_map<(m, n, k) -> (k, n)>,   // B: read B[k,n]
    affine_map<(m, n, k) -> (m, n)>    // C: read/write C[m,n]
  ],
  // One iterator type per loop dimension
  iterator_types = ["parallel", "parallel", "reduction"]
} ins(%A, %B : tensor<4x4xf32>, tensor<4x4xf32>)
  outs(%C : tensor<4x4xf32>) {
^bb0(%a: f32, %b: f32, %c: f32):
  // Body: computes one element of C += A * B
  %mul = arith.mulf %a, %b : f32
  %acc = arith.addf %c, %mul : f32
  linalg.yield %acc : f32
} -> tensor<4x4xf32>
```

**Indexing maps**: each affine map maps from the loop iteration space (the "indexing domain") to indices into that operand. The loop iteration space has as many dimensions as there are `iterator_types`. The number of affine maps equals `num_inputs + num_outputs`.

**Iterator types**: each dimension of the loop space is classified as:
- `parallel`: iterations are independent; the dimension can be parallelized, vectorized, or reordered
- `reduction`: iterations are dependent due to accumulation; must be reduced before producing output

For the matmul above: loops `m` and `n` are parallel (output dimensions), loop `k` is a reduction (the inner product dimension).

**Body region**: receives one value per operand (the element at the current iteration's indices). For inputs, this is the element value. For outputs, this is the current accumulated value. The `linalg.yield` returns the new output value.

### 139.2.3 Iteration Space Semantics

The iteration space is the Cartesian product of all loop dimension ranges. Ranges are determined by the operand types:

- Static shapes: ranges come from the operand's type dimensions
- Dynamic shapes: ranges come from runtime dimension values
- The shape inference verifier ensures consistency across operands

For the matmul example with `tensor<M×K>`, `tensor<K×N>`, `tensor<M×N>`:
- `m` ∈ [0, M), `n` ∈ [0, N), `k` ∈ [0, K)

The verifier checks that applying each affine map to a point in the iteration space produces valid indices for its operand. This static verification catches shape mismatches at compile time.

### 139.2.4 Projections and Complex Index Maps

The power of `linalg.generic` lies in its generality via affine maps. Consider a few examples:

**Transpose** (swap dimensions):
```mlir
%Bt = linalg.generic {
  indexing_maps = [
    affine_map<(d0, d1) -> (d1, d0)>,  // read B[j, i] (transposed)
    affine_map<(d0, d1) -> (d0, d1)>   // write Bt[i, j]
  ],
  iterator_types = ["parallel", "parallel"]
} ins(%B : tensor<4x4xf32>) outs(%init : tensor<4x4xf32>) {
^bb0(%b: f32, %c: f32):
  linalg.yield %b : f32
} -> tensor<4x4xf32>
```

**Broadcast** (repeat one dimension):
```mlir
%result = linalg.generic {
  indexing_maps = [
    affine_map<(d0, d1) -> (d1)>,   // read vec[j] (broadcast over d0)
    affine_map<(d0, d1) -> (d0, d1)>
  ],
  iterator_types = ["parallel", "parallel"]
} ins(%vec : tensor<4xf32>) outs(%init : tensor<4x4xf32>) {
^bb0(%v: f32, %c: f32):
  linalg.yield %v : f32
} -> tensor<4x4xf32>
```

**Batch matmul**:
```mlir
%result = linalg.generic {
  indexing_maps = [
    affine_map<(b, m, n, k) -> (b, m, k)>,
    affine_map<(b, m, n, k) -> (b, k, n)>,
    affine_map<(b, m, n, k) -> (b, m, n)>
  ],
  iterator_types = ["parallel", "parallel", "parallel", "reduction"]
} ins(%A, %B : tensor<?x?x?xf32>, tensor<?x?x?xf32>)
  outs(%C : tensor<?x?x?xf32>) { ... } -> tensor<?x?x?xf32>
```

The batch dimension `b` is parallel and appears identically in all maps — each batch element is independent.

### 139.2.5 `linalg.index`

Inside a `linalg.generic` body, `linalg.index` retrieves the current iteration index along a given dimension:

```mlir
linalg.generic {
  indexing_maps = [affine_map<(d0, d1) -> (d0, d1)>],
  iterator_types = ["parallel", "parallel"]
} outs(%out : tensor<4x4xf32>) {
^bb0(%c: f32):
  %row = linalg.index 0 : index  // current row index
  %col = linalg.index 1 : index  // current column index
  %row_f = arith.index_cast %row : index to f32
  %col_f = arith.index_cast %col : index to f32
  %val = arith.addf %row_f, %col_f : f32
  linalg.yield %val : f32
} -> tensor<4x4xf32>
```

This enables position-dependent computations (constructing identity matrices, positional encoding, etc.) without requiring explicit loop induction variables.

---

## 139.3 Named Linalg Ops

### 139.3.1 Operation Generator

Named linalg ops are generated from YAML definitions in [`mlir/include/mlir/Dialect/Linalg/IR/LinalgNamedOps.yaml`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Linalg/IR/LinalgNamedOps.yaml) using the `mlir-linalg-ods-yaml-gen` tool. Each named op is exactly equivalent to a specific `linalg.generic` — it is pure syntactic sugar that carries a human-readable name, making IR easier to read and enabling op-specific optimizations.

Converting named ops to generic form:
```bash
mlir-opt --linalg-generalize-named-ops input.mlir
```

### 139.3.2 Matrix and Vector Operations

```mlir
// Matrix multiplication: C += A × B
%C_new = linalg.matmul
    ins(%A, %B : tensor<4x4xf32>, tensor<4x4xf32>)
    outs(%C : tensor<4x4xf32>) -> tensor<4x4xf32>

// Batch matrix multiply: C[b] += A[b] × B[b]
%C_new = linalg.batch_matmul
    ins(%A, %B : tensor<2x4x4xf32>, tensor<2x4x4xf32>)
    outs(%C : tensor<2x4x4xf32>) -> tensor<2x4x4xf32>

// Matrix-vector: y += A × x
%y_new = linalg.matvec
    ins(%A, %x : tensor<4x4xf32>, tensor<4xf32>)
    outs(%y : tensor<4xf32>) -> tensor<4xf32>

// Vector-matrix: y += x × A
%y_new = linalg.vecmat
    ins(%x, %A : tensor<4xf32>, tensor<4x4xf32>)
    outs(%y : tensor<4xf32>) -> tensor<4xf32>

// Dot product: acc += dot(x, y)
%result = linalg.dot
    ins(%x, %y : tensor<4xf32>, tensor<4xf32>)
    outs(%acc : tensor<f32>) -> tensor<f32>
```

### 139.3.3 Convolution Operations

Convolution ops follow a systematic naming convention based on data layout:

```mlir
// 2D conv: NHWC input, HWCF filter, NHWF output
%out = linalg.conv_2d_nhwc_hwcf
    ins(%input, %filter : tensor<1x8x8x3xf32>, tensor<3x3x3x32xf32>)
    outs(%output : tensor<1x6x6x32xf32>) -> tensor<1x6x6x32xf32>

// 2D conv: NCHW input, FCHW filter, NFHW output
%out = linalg.conv_2d_nchw_fchw
    ins(%input, %filter : tensor<1x3x8x8xf32>, tensor<32x3x3x3xf32>)
    outs(%output : tensor<1x32x6x6xf32>) -> tensor<1x32x6x6xf32>

// Depthwise 2D conv: NHWC input, HWC filter
%out = linalg.depthwise_conv_2d_nhwc_hwc
    ins(%input, %filter : tensor<1x8x8x3xf32>, tensor<3x3x3xf32>)
    outs(%output : tensor<1x6x6x3xf32>) -> tensor<1x6x6x3xf32>

// Pooling
%out = linalg.pooling_nhwc_max
    ins(%input, %kernel_size : tensor<1x8x8x3xf32>, tensor<3x3xf32>)
    outs(%output : tensor<1x6x6x3xf32>) -> tensor<1x6x6x3xf32>
```

### 139.3.4 Elementwise Operations

```mlir
// Fill: set all elements to a scalar
%filled = linalg.fill ins(%scalar : f32) outs(%t : tensor<4x4xf32>) -> tensor<4x4xf32>

// Map: apply a function elementwise (variadic inputs, single output)
%result = linalg.map { arith.addf }
    ins(%A, %B : tensor<4xf32>, tensor<4xf32>)
    outs(%out : tensor<4xf32>)

// Reduce: reduce along a specified dimension
%red = linalg.reduce { arith.addf }
    ins(%A : tensor<4x4xf32>)
    outs(%init : tensor<4xf32>)
    dimensions = [1]  // reduce over columns

// Transpose: reorder dimensions
%Bt = linalg.transpose
    ins(%B : tensor<4x8xf32>)
    outs(%init : tensor<8x4xf32>)
    permutation = [1, 0]

// Broadcast: broadcast dims of lower-rank tensor to higher-rank
%expanded = linalg.broadcast
    ins(%vec : tensor<4xf32>)
    outs(%init : tensor<4x4xf32>)
    dimensions = [0]  // broadcast along dimension 0
```

`linalg.map` is the element-wise generalization. The block body receives one element from each input and produces one element for the output. The `arith.addf` shorthand expands to the standard body.

---

## 139.4 Linalg Transformations

### 139.4.1 `TilingInterface`

`TilingInterface` ([`mlir/include/mlir/Interfaces/TilingInterface.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Interfaces/TilingInterface.h)) is the core abstraction enabling tiling of any linalg op. It provides:

```cpp
// Returns iteration domains (loop bounds)
SmallVector<Range> getIterationDomain(OpBuilder &b);

// Returns tiled ops + surrounding loops given tile sizes
FailureOr<TilingResult> getTiledImplementation(OpBuilder &b,
    ArrayRef<OpFoldResult> offsets, ArrayRef<OpFoldResult> sizes);

// Returns result slices for the given tile
LogicalResult getResultTilePosition(OpBuilder &b, unsigned resultNumber,
    ArrayRef<OpFoldResult> offsets, ArrayRef<OpFoldResult> sizes,
    SmallVectorImpl<OpFoldResult> &resultOffsets,
    SmallVectorImpl<OpFoldResult> &resultSizes);
```

All named linalg ops and `linalg.generic` implement this interface. The Transform dialect's `transform.structured.tile_using_for` action calls this interface to produce a tiled version.

### 139.4.2 Tiling Example

```bash
# Tile a matmul with 4×4×4 tiles:
mlir-opt --transform-interpreter='
  transform.sequence failures(propagate) {
    ^bb0(%module: !transform.any_op):
      %matmul = transform.structured.match ops{["linalg.matmul"]} in %module
      %tiled, %loops:3 = transform.structured.tile_using_for %matmul
          tile_sizes [4, 4, 4]
  }
' input.mlir
```

After tiling, the `linalg.matmul` is inside three `scf.for` loops with step 4, and operates on `tensor<4x4xf32>` tiles.

### 139.4.3 Producer-Consumer Fusion

`linalg.fuse_into_containing_op` merges a producer linalg op into the loop nest of a consumer:

```bash
mlir-opt --linalg-fuse-into-containing-op input.mlir
```

Fusion conditions:
- The producer's result is used only by the consumer
- The consumer implements `TilingInterface`
- The produced slice matches a tiled input of the consumer

When fusion succeeds, the producer is moved inside the consumer's loop nest, enabling the combined computation to be vectorized or mapped to GPU threads together.

### 139.4.4 `linalg.pack` and `linalg.unpack`

Pack/unpack operations transform data layout for cache-optimal computation:

```mlir
// Pack a matrix for GEMM: tile rows by 4, cols by 4 (KCRS format)
%packed = linalg.pack %A
    inner_dims_pos = [0, 1]
    inner_tiles = [4, 4]
    into %packed_buf : (tensor<16x16xf32>, tensor<4x4x4x4xf32>) -> tensor<4x4x4x4xf32>

// Unpack back to original layout
%restored = linalg.unpack %packed
    inner_dims_pos = [0, 1]
    inner_tiles = [4, 4]
    into %output : (tensor<4x4x4x4xf32>, tensor<16x16xf32>) -> tensor<16x16xf32>
```

The packed layout `tensor<4x4x4x4xf32>` represents a 16×16 matrix as 4×4 tiles of 4×4 elements. This layout is cache-friendly for micro-kernel GEMM implementations that operate on 4×4 register tiles.

### 139.4.5 Vectorization

The `--linalg-vectorize-named-ops` pass converts named linalg ops to vector ops:

```bash
mlir-opt \
  --linalg-vectorize-named-ops \
  --convert-linalg-to-loops \
  --convert-vector-to-llvm \
  input.mlir
```

Alternatively, the Transform dialect provides fine-grained control:

```
transform.structured.vectorize %matmul vector_sizes [4, 4, 4]
```

Vectorization translates the linalg op into:
1. Vector loads (`vector.transfer_read`) for inputs
2. A `vector.contract` for the computation
3. Vector stores (`vector.transfer_write`) for the output

For `linalg.matmul` with vector sizes `[4, 4, 4]`, this produces a `vector<4x4xf32>` outer product computation using `vector.contract`.

### 139.4.6 Lowering Paths

The standard lowering path from linalg:

```
linalg.generic / named ops
     ↓  --convert-linalg-to-loops (or --linalg-to-affine-loops)
scf.for / affine.for loops with tensor.extract/insert or memref.load/store
     ↓  --convert-scf-to-cf
cf.br / cf.cond_br
     ↓  --convert-cf-to-llvm
llvm dialect
```

For GPU targets:

```
linalg.generic
     ↓  tiling + fusion
linalg.generic on tiles inside scf.forall
     ↓  vectorization
vector.contract inside scf.forall
     ↓  --gpu-map-parallel-loops
gpu.launch_func
```

---

## 139.5 Mixed Tensor/MemRef Linalg

`linalg.generic` and all named ops work on both tensors and memrefs. When operating on memrefs, outputs must be passed as `outs` operands (in-place semantics); no return value is produced:

```mlir
// Tensor form: returns new tensor
%result = linalg.matmul
    ins(%A, %B : tensor<4x4xf32>, tensor<4x4xf32>)
    outs(%C : tensor<4x4xf32>) -> tensor<4x4xf32>

// MemRef form: writes in-place, no return
linalg.matmul
    ins(%A, %B : memref<4x4xf32>, memref<4x4xf32>)
    outs(%C : memref<4x4xf32>)
```

Mixing tensor inputs with memref outputs is not allowed — all operands must consistently use one model per `ins`/`outs` group. The bufferization pass converts tensor forms to memref forms.

---

## 139.6 The Structured Computation Invariants

Understanding why `linalg.generic`'s invariants matter:

| Invariant | Why It Matters |
|---|---|
| No input-output aliasing | Safe to reorder and parallelize iterations |
| Parallel dims independent | Auto-vectorization, parallelization, tiling |
| Reduction dims sequential | Correct reduction requires ordering |
| Affine index maps | Index transformations are cost-free (just different loads) |
| Single region body | Body can be analyzed statically, vectorized, or outlined |
| Works on tensors | No aliasing — fusion is always safe |

These invariants are what distinguish `linalg.generic` from a generic `scf.for` loop: the compiler has much more information about the computation structure, enabling transformations that would be unsafe or impossible to prove correct with general loops.

---

## Chapter 139 Summary

- The `tensor` dialect provides value-semantic, aliasing-free dense arrays. Key ops: `tensor.empty` (uninitialized), `tensor.splat` (broadcast scalar), `tensor.extract`/`tensor.insert` (scalar access), `tensor.extract_slice`/`tensor.insert_slice` (sub-tensor slicing), `tensor.pad` (padding with computed values).
- `linalg.generic` is MLIR's universal structured computation op. It is parameterized by indexing maps (how operand indices relate to loop dimensions) and iterator types (`parallel` or `reduction`). The body computes one output element per iteration.
- Named linalg ops (`linalg.matmul`, `linalg.conv_2d_nhwc_hwcf`, `linalg.fill`, `linalg.map`, `linalg.reduce`, `linalg.transpose`, `linalg.broadcast`) are syntactic sugar over `linalg.generic`. `--linalg-generalize-named-ops` converts all to generic form.
- `TilingInterface` enables tiling any linalg op into smaller ops inside loop nests. Tiling is driven by the Transform dialect or direct pass invocation and produces `scf.for` loops with `tensor.extract_slice`/`tensor.insert_slice` around the tiled op.
- `linalg.pack`/`linalg.unpack` transform data layout into blocked/packed formats for cache-optimal micro-kernel computation.
- Vectorization converts linalg ops to `vector.contract` and `vector.transfer_read`/`vector.transfer_write`, enabling hardware SIMD utilization.
- The standard lowering path goes linalg → scf → cf → llvm for CPU targets, and linalg → tiled linalg → vectorized vectors → gpu.launch_func for GPU targets.
- The structural invariants of `linalg.generic` — no aliasing, affine index maps, typed iterator types — are what make it transformable in ways that general loops are not.


---

@copyright jreuben11
