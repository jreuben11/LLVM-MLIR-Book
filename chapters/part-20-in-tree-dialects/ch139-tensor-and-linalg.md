# Chapter 139 — Tensor and Linalg

*Part XX — In-Tree Dialects*

The `tensor` and `linalg` dialects form the computational heart of MLIR for array-processing workloads. The `tensor` dialect provides the value-semantic, aliasing-free view of dense arrays that makes high-level transformations safe and composable. The `linalg` dialect builds on it with a systematic framework for expressing structured computation — from simple elementwise maps to complex convolutions — in a form that exposes rich parallelism and transformation opportunities. Understanding these two dialects thoroughly is the prerequisite for working with any machine learning compiler or high-performance array processing pipeline built on MLIR.

---

## Table of Contents

- [139.1 The `tensor` Dialect](#1391-the-tensor-dialect)
  - [139.1.1 Value Semantics and Immutability](#13911-value-semantics-and-immutability)
  - [139.1.2 `tensor.empty` and `tensor.splat`](#13912-tensorempty-and-tensorsplat)
  - [139.1.3 Element Access](#13913-element-access)
  - [139.1.4 Slice Operations](#13914-slice-operations)
  - [139.1.5 Construction Operations](#13915-construction-operations)
  - [139.1.6 Shape Operations](#13916-shape-operations)
- [139.2 `linalg.generic`](#1392-linalggeneric)
  - [139.2.1 The Universal Structured Computation Op](#13921-the-universal-structured-computation-op)
  - [139.2.2 Anatomy of `linalg.generic`](#13922-anatomy-of-linalggeneric)
  - [139.2.3 Iteration Space Semantics](#13923-iteration-space-semantics)
  - [139.2.4 Projections and Complex Index Maps](#13924-projections-and-complex-index-maps)
  - [139.2.5 `linalg.index`](#13925-linalgindex)
- [139.3 Named Linalg Ops](#1393-named-linalg-ops)
  - [139.3.1 Operation Generator](#13931-operation-generator)
  - [139.3.2 Matrix and Vector Operations](#13932-matrix-and-vector-operations)
  - [139.3.3 Convolution Operations](#13933-convolution-operations)
  - [139.3.4 Elementwise Operations](#13934-elementwise-operations)
- [139.4 Linalg Transformations](#1394-linalg-transformations)
  - [139.4.1 `TilingInterface`](#13941-tilinginterface)
  - [139.4.2 Tiling Example](#13942-tiling-example)
  - [139.4.3 Producer-Consumer Fusion](#13943-producer-consumer-fusion)
  - [139.4.4 `linalg.pack` and `linalg.unpack`](#13944-linalgpack-and-linalgunpack)
  - [139.4.5 Vectorization](#13945-vectorization)
  - [139.4.6 Lowering Paths](#13946-lowering-paths)
- [139.5 Mixed Tensor/MemRef Linalg](#1395-mixed-tensormemref-linalg)
- [139.6 The Structured Computation Invariants](#1396-the-structured-computation-invariants)
- [139.7 The TOSA Dialect](#1397-the-tosa-dialect)
  - [139.7.1 Design Goals and Position in the ML Stack](#13971-design-goals-and-position-in-the-ml-stack)
  - [139.7.2 Type System](#13972-type-system)
  - [139.7.3 Operator Taxonomy](#13973-operator-taxonomy)
  - [139.7.4 The tosa-to-linalg Lowering](#13974-the-tosa-to-linalg-lowering)
  - [139.7.5 Validation and Conformance](#13975-validation-and-conformance)
  - [139.7.6 Production Usage](#13976-production-usage)
- [Chapter 139 Summary](#chapter-139-summary)

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

## 139.7 The TOSA Dialect

TOSA (Tensor Operator Set Architecture) is an MLIR dialect that defines a portable, verifiable set of ML operators positioned one level above `linalg` in the abstraction hierarchy. Where `linalg.generic` is a flexible computation framework, TOSA defines concrete semantics for specific ML operations — convolutions, activations, reductions — in a form that both hardware compilers and portability-focused toolchains can target. Its primary lowering path is `--tosa-to-linalg`, making it a natural companion to the material in §139.2–139.5.

Sources: [`mlir/include/mlir/Dialect/Tosa/IR/TosaOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Tosa/IR/TosaOps.td), [`mlir/lib/Dialect/Tosa/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Dialect/Tosa/), and the [TOSA Specification 1.0](https://www.mlplatform.org/tosa/tosa_spec.html).

### 139.7.1 Design Goals and Position in the ML Stack

TOSA was developed by Arm and contributed to MLIR in 2021. Its design goals distinguish it from both `linalg` and higher-level frameworks:

**Portability**: TOSA's ~90 operators have fully specified semantics — including edge cases for overflow, saturation, and out-of-bounds access — so a program in TOSA has identical observable behaviour on any compliant implementation. This is stronger than linalg's guarantee (which specifies structure but defers some semantics to the body region).

**Quantization awareness**: Every TOSA operator natively handles quantized integer tensors alongside floating-point. Integer arithmetic with specified shift-and-scale quantization parameters is a first-class feature, not an afterthought.

**Verifiability**: The `tosa-validate` pass checks that an MLIR module conforms to the TOSA specification (operator constraints, type constraints, shape rules). This enables downstream tools to rely on TOSA as a stable, checked contract.

Position in a typical ML lowering stack:

```
PyTorch / TensorFlow / ONNX
        ↓
  StableHLO / Torch dialect
        ↓
      TOSA          ← portable, verifiable ML ops
        ↓
  linalg.generic   ← structured loops
        ↓
  vector + scf
        ↓
    LLVM dialect
```

TOSA serves as the portability boundary: frontends (TFLite, ONNX-MLIR, torch-mlir) lower to TOSA; backends (IREE, hardware NPU compilers) lower from TOSA to linalg or directly to hardware IR.

### 139.7.2 Type System

TOSA operates exclusively on ranked tensors. The element types are:

| TOSA element type | MLIR type | Notes |
|-------------------|-----------|-------|
| `float` | `f32` | 32-bit float |
| `fp16` | `f16` | 16-bit float |
| `bf16` | `bf16` | Brain float |
| `int32` | `i32` | Signed 32-bit |
| `int16` | `i16` | Signed 16-bit |
| `int8` | `i8` | Signed 8-bit (common for quantization) |
| `int4` | `i4` | Signed 4-bit |
| `bool` | `i1` | Comparison output |

TOSA uses `tensor<shape x element_type>` directly — no custom type wrapper. Quantization information is carried in operator attributes rather than in the type, unlike `quant` dialect types.

**Quantized operations** carry explicit `input_zp` (zero-point) and `weight_zp` attributes:

```mlir
// Quantized 8-bit convolution
%output = tosa.conv2d %input, %weight, %bias {
    pad = array<i64: 0, 0, 0, 0>,
    stride = array<i64: 1, 1>,
    dilation = array<i64: 1, 1>,
    input_zp = -128 : i32,
    weight_zp = 0 : i32
} : (tensor<1x8x8x1xi8>, tensor<16x3x3x1xi8>, tensor<16xi32>)
  -> tensor<1x6x6x16xi32>
```

The output is `i32` — TOSA mandates 32-bit accumulation for 8-bit convolution to prevent overflow.

### 139.7.3 Operator Taxonomy

TOSA's ~90 operators are organised into functional categories:

**Tensor arithmetic** — elementwise binary and unary ops:
```mlir
%add  = tosa.add %a, %b : (tensor<4xf32>, tensor<4xf32>) -> tensor<4xf32>
%mul  = tosa.mul %a, %b {shift = 0 : i8} : (...) -> tensor<4xf32>
%sub  = tosa.sub %a, %b : ...
%abs  = tosa.abs %a : (tensor<4xf32>) -> tensor<4xf32>
%neg  = tosa.negate %a {input1_zp = 0 : i32, output_zp = 0 : i32} : ...
%recip = tosa.reciprocal %a : ...
```

The `shift` attribute on `tosa.mul` is for integer fixed-point multiplication: the product is right-shifted by `shift` bits before storing.

**Activation functions**:
```mlir
%relu  = tosa.clamp %x {min_fp = 0.0 : f32, max_fp = 3.4e38 : f32,
                         min_int = 0 : i64, max_int = 2147483647 : i64}
                   : (tensor<4xf32>) -> tensor<4xf32>
%gelu  = tosa.sigmoid %x : (tensor<4xf32>) -> tensor<4xf32>
%tanh  = tosa.tanh    %x : ...
%log   = tosa.log     %x : ...
%exp   = tosa.exp     %x : ...
```

`tosa.clamp` implements ReLU (clamp to [0, +∞]), ReLU6 (clamp to [0, 6]), and general clamp in one op.

**Convolution and pooling**:
```mlir
// 2D convolution: NHWC input, OHWI weight (TOSA canonical layout)
%out = tosa.conv2d %input, %weight, %bias {
    pad = array<i64: 1, 1, 1, 1>,
    stride = array<i64: 1, 1>,
    dilation = array<i64: 1, 1>
} : (tensor<1x8x8x3xf32>, tensor<16x3x3x3xf32>, tensor<16xf32>)
  -> tensor<1x8x8x16xf32>

// Depthwise convolution: NHWC input, HWCM weight
%dw  = tosa.depthwise_conv2d %input, %weight, %bias { ... }
     : (tensor<1x8x8x4xf32>, tensor<3x3x4x1xf32>, tensor<4xf32>)
    -> tensor<1x8x8x4xf32>

// Max pooling
%pool = tosa.max_pool2d %input {
    kernel = array<i64: 2, 2>,
    stride = array<i64: 2, 2>,
    pad    = array<i64: 0, 0, 0, 0>
} : (tensor<1x8x8x4xf32>) -> tensor<1x4x4x4xf32>
```

TOSA mandates NHWC (batch, height, width, channel) layout for activations and OHWI (output-channel, height, width, input-channel) for convolution weights. This differs from PyTorch's default NCHW; framework frontends insert `tosa.transpose` to normalise layout.

**Matrix multiplication**:
```mlir
// Batch matmul: [N, H, K] × [N, K, W] → [N, H, W]
%out = tosa.matmul %lhs, %rhs
     : (tensor<2x4x8xf32>, tensor<2x8x4xf32>) -> tensor<2x4x4xf32>
```

**Reduction**:
```mlir
%sum  = tosa.reduce_sum  %x {axis = 1 : i32} : (tensor<4x8xf32>) -> tensor<4x1xf32>
%max  = tosa.reduce_max  %x {axis = 2 : i32} : ...
%min  = tosa.reduce_min  %x {axis = 0 : i32} : ...
%mean = tosa.reduce_mean %x {axis = 1 : i32} : ...  // not in TOSA 1.0; use reduce_sum + reciprocal
```

Note: TOSA 1.0 has no `reduce_mean` — it is expressed as `reduce_sum` followed by `mul` with a reciprocal of the dimension size.

**Shape manipulation**:
```mlir
%r = tosa.reshape %x {new_shape = array<i64: 2, 4>}
   : (tensor<8xf32>) -> tensor<2x4xf32>

%t = tosa.transpose %x {perms = array<i32: 0, 2, 1>}
   : (tensor<2x3x4xf32>) -> tensor<2x4x3xf32>

%s = tosa.slice %x {start = array<i64: 0, 1>, size = array<i64: 2, 3>}
   : (tensor<4x5xf32>) -> tensor<2x3xf32>

%p = tosa.pad %x {padding = array<i64: 1, 1, 0, 2>}
            (tensor<4x5xf32>) -> tensor<6x7xf32>
```

**Scatter / gather**:
```mlir
// Gather: index into dimension 0 of %values
%out = tosa.gather %values, %indices
     : (tensor<1x16x4xf32>, tensor<1x8xi32>) -> tensor<1x8x4xf32>

// Scatter: write %src into %values at %indices positions
%out = tosa.scatter %values_in, %indices, %src
     : (tensor<1x16x4xf32>, tensor<1x8xi32>, tensor<1x8x4xf32>)
    -> tensor<1x16x4xf32>
```

**Data-format / cast**:
```mlir
%cast = tosa.cast %x : (tensor<4xi8>) -> tensor<4xf32>   // int→float conversion
%rs   = tosa.rescale %x {
    input_zp = -128 : i32, output_zp = 0 : i32,
    multiplier = array<i32: 1073741824>,
    shift      = array<i8: 1>,
    scale32 = true, double_round = true, per_channel = false
} : (tensor<4xi8>) -> tensor<4xi8>
```

`tosa.rescale` is the quantization re-scaling operator: it takes an integer tensor, subtracts the input zero-point, multiplies by a fixed-point scale (`multiplier >> shift`), adds the output zero-point, and saturates/rounds. This single op encodes the entire integer requantization step that sits between quantized layers.

### 139.7.4 The tosa-to-linalg Lowering

The primary lowering pass is `--tosa-to-linalg` (elementwise and reductions) plus `--tosa-to-linalg-named` (convolutions and matmul). Together they produce `linalg` IR ready for tiling and vectorization:

```bash
mlir-opt input.mlir \
    --tosa-to-linalg-named \
    --tosa-to-linalg \
    --linalg-bufferize \
    --convert-linalg-to-loops \
    --convert-cf-to-llvm \
    --convert-arith-to-llvm \
    --finalize-memref-to-llvm \
    --reconcile-unrealized-casts \
    -o output.mlir
```

**Elementwise ops** (`tosa.add`, `tosa.mul`, etc.) lower to `linalg.map`:

```mlir
// tosa.add lowers to:
%out = linalg.map { arith.addf }
       ins(%a, %b : tensor<4xf32>, tensor<4xf32>)
       outs(%empty : tensor<4xf32>)
```

**`tosa.matmul`** lowers to `linalg.batch_matmul`:

```mlir
%out = linalg.batch_matmul
       ins(%lhs, %rhs : tensor<2x4x8xf32>, tensor<2x8x4xf32>)
       outs(%init    : tensor<2x4x4xf32>) -> tensor<2x4x4xf32>
```

**`tosa.conv2d`** lowers to `linalg.conv_2d_nhwc_ohwi`:

```mlir
%out = linalg.conv_2d_nhwc_ohwi
       { dilations = dense<1> : tensor<2xi64>,
         strides   = dense<1> : tensor<2xi64> }
       ins(%input, %weight : tensor<1x8x8x3xf32>, tensor<16x3x3x3xf32>)
       outs(%init          : tensor<1x8x8x16xf32>) -> tensor<1x8x8x16xf32>
```

**`tosa.reduce_sum`** lowers to `linalg.reduce`:

```mlir
%out = linalg.reduce { arith.addf }
       ins(%x    : tensor<4x8xf32>)
       outs(%init : tensor<4x1xf32>)
       dimensions = [1]
```

After `tosa-to-linalg`, the resulting linalg IR is subject to the full tiling, vectorization, and bufferization machinery described in §139.4 and §139.5.

### 139.7.5 Validation and Conformance

The `--tosa-validate` pass checks that a module satisfies TOSA specification constraints — shape rules, type constraints, attribute ranges — and can be run in strict mode to catch spec violations:

```bash
mlir-opt --tosa-validate="profile=bi level=warning" input.mlir
```

TOSA defines three profiles that form a conformance hierarchy:

| Profile | Abbreviation | Intended target |
|---------|-------------|-----------------|
| Base Inference | `bi` | Inference-only hardware (NPUs) |
| Main Inference | `mi` | Full inference with all float types |
| Main Training | `mt` | Training (adds gradient ops) |

The `tosa-check-shapes` pass performs shape inference for dynamically-shaped TOSA programs, propagating static shapes where possible before lowering.

### 139.7.6 Production Usage

**IREE**: TOSA is one of IREE's primary input dialects (`--iree-import-tosa`). TFLite flatbuffers are converted to TOSA by `iree-import-tflite`, which then passes through IREE's flow → stream → HAL lowering chain (covered in [Chapter 162 — IREE](../part-23-mlir-production/ch162-iree-deployment-compiler.md)).

**torch-mlir**: The `--torch-backend-to-tosa-backend-pipeline` lowers PyTorch `torch.aten.*` ops to TOSA for deployment targets that accept TOSA as their interface.

**onnx-mlir**: The `--convert-onnx-to-tosa` pass maps ONNX operators to their TOSA equivalents. ONNX operators without a direct TOSA equivalent are decomposed into sequences of TOSA primitives.

**Arm Ethos NPU SDK**: Uses TOSA as its public compilation interface — the Vela compiler accepts TOSA as input and schedules computation onto the NPU's DMA and MAC array. This is the primary industrial motivation for TOSA's quantization-first design.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **`linalg.generic` iterator-type extensions**: The MLIR community is actively discussing adding `window` and `scan` iterator types to `linalg.generic` to natively express prefix-scan patterns and sliding-window operations without resorting to index-arithmetic workarounds — tracked on [discourse.llvm.org RFC: Extended Iterator Types for Linalg](https://discourse.llvm.org/c/mlir/).
- **TOSA 1.0 `reduce_mean` operator**: The TOSA specification working group (Arm, Qualcomm, Intel) has an open patch to add `tosa.reduce_mean` as a first-class op in TOSA 1.1, eliminating the current workaround of `reduce_sum` + `mul` with a reciprocal scalar.
- **Structured named-op expansion via Python DSL**: The `mlir-linalg-ods-yaml-gen` workflow is being replaced by a Python-based structured op generation DSL to ease authoring of new convolution variants (e.g., `conv_3d_ndhwc_dhwcf` for video models). In-progress on the LLVM monorepo as of early 2026.
- **One-shot bufferization composability with `linalg.pack`**: Several patches in review aim to make one-shot bufferization correctly analyse aliasing through `linalg.pack`/`linalg.unpack` pairs, enabling in-place packing without intermediate copies on CPU targets.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Dataflow graph partitioning at linalg level**: Research into automatic operator fusion heuristics beyond greedy producer-consumer fusion — incorporating cost models for memory bandwidth, cache footprint, and register pressure — is underway in IREE's codegen research group. The goal is linalg-level partitioning that drives GPU thread block and warp assignment without manual `TilingInterface` annotations.
- **TOSA 2.0 training profile and gradient ops**: The TOSA specification roadmap includes a `Main Training` (MT2) profile adding `tosa.conv2d_backprop_input`, `tosa.conv2d_backprop_filter`, and gradient ops for all activation functions, enabling TOSA to serve as a deployment target for training frameworks such as IREE-PT.
- **Structured sparsity in `linalg.generic`**: Building on the `sparse_tensor` dialect (Chapter 141), ongoing work (notably from MIT/MIT-IBM Watson) targets first-class 2:4 structured sparsity encoding in `linalg.generic` indexing maps, allowing the compiler to emit cuSPARSELt-compatible sparse GEMM patterns directly from linalg.
- **`tensor` dialect dynamic shape completeness**: Completing the `ShapeReification` interface for all tensor ops so that dynamic shape propagation through `tensor.pad`, `tensor.concat`, and `tensor.reshape` is fully inferrable at compile time, removing the need for runtime `tensor.dim` queries in fully-static subgraphs.

### 5-Year Horizon (Long-Term, by ~2031)

- **Declarative transformation scheduling**: A long-term goal articulated at the 2025 MLIR Open Design Meeting is a declarative, provably-correct transformation scheduler for `linalg.generic` — analogous to Halide's scheduling language but integrated with the MLIR Transform dialect and capable of generating optimal tile/vectorize/parallelize schedules for arbitrary hardware cost models.
- **TOSA as a hardware ABI**: As heterogeneous SoCs (smartphone NPUs, edge inference chips) proliferate, TOSA is positioned to become a binary-compatible ABI layer — analogous to SPIR-V for GPUs — where compiled TOSA modules execute on any conformant accelerator without recompilation. Standardization through MLCommons is the expected path.
- **Fused `linalg`-to-systolic-array lowering**: Direct lowering from `linalg.matmul` and `linalg.conv_2d_*` to systolic-array dataflow IR (analogous to Google TPU HLO → XLA → LLO) is a research goal for in-tree MLIR, enabling MLIR to serve as the universal compilation stack for custom ML ASICs without vendor-proprietary intermediate representations.

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
- **TOSA** (Tensor Operator Set Architecture) is a portable, verifiable ML operator dialect positioned above linalg. Its ~90 ops span elementwise arithmetic, activations, convolution, pooling, matmul, reduction, shape manipulation, and scatter/gather, all with fully specified semantics including quantization. `--tosa-to-linalg` and `--tosa-to-linalg-named` lower TOSA to linalg for the standard optimization pipeline. TOSA is used by IREE, torch-mlir, onnx-mlir, and the Arm Ethos NPU SDK as the portability boundary between ML frameworks and hardware backends.


---

@copyright jreuben11
