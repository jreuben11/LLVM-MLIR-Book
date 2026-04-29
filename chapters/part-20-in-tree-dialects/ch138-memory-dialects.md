# Chapter 138 — Memory Dialects

*Part XX — In-Tree Dialects*

Managing memory explicitly is unavoidable in any production compiler. MLIR addresses this through two closely related dialects: `memref`, which provides a rich typed, strided memory abstraction, and `bufferization`, which provides the bridge from the pure functional tensor world to the imperative memref world. Together they form MLIR's complete answer to the question of how high-level array operations become concrete memory allocations with predictable lifetimes. This chapter covers both dialects in depth — their type systems, operations, layout semantics, and the one-shot bufferization algorithm that minimizes unnecessary data copies.

---

## 138.1 The `memref` Dialect

### 138.1.1 MemRefType: MLIR's Memory Type

`MemRefType` is the primary type for representing explicit memory regions in MLIR ([`mlir/include/mlir/IR/BuiltinTypes.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/BuiltinTypes.h)). It encodes:

1. **Element type** — the type of each element (scalar, vector, complex, etc.)
2. **Shape** — static or dynamic dimensions
3. **Layout map** — an affine map describing how multi-dimensional indices map to a linear address offset
4. **Memory space** — an attribute identifying the address space (GPU global, shared, etc.)

The full type syntax:

```mlir
memref<4x4xf32>                          // static shape, identity layout, default space
memref<?x?xf32>                          // dynamic shape (runtime extents)
memref<4x4xf32, strided<[4, 1], offset: 0>>  // explicit row-major stride
memref<4x4xf32, strided<[?, 1], offset: ?>>  // dynamic stride/offset (from subview)
memref<f32>                              // 0-dimensional (scalar reference)
memref<4xf32, 3>                         // address space 3 (e.g., GPU shared memory)
memref<4xf32, #gpu.address_space<workgroup>>  // named address space
```

`UnrankedMemRefType` is the unranked variant — `memref<*xf32>` — used when rank is not known at compile time. It is rarely used outside of generic runtime interfaces, since MLIR's strength comes from ranked, statically-shaped types.

### 138.1.2 Allocation Operations

```mlir
// Stack allocation (scope-limited, no explicit dealloc needed)
%stack = memref.alloca() : memref<4x4xf32>
%stack_dyn = memref.alloca(%n, %m) : memref<?x?xf32>

// Heap allocation
%heap = memref.alloc() : memref<4x4xf32>
%heap_dyn = memref.alloc(%n, %m) : memref<?x?xf32>
%aligned = memref.alloc() {alignment = 64} : memref<4x4xf32>

// Deallocation (required for heap-allocated memrefs)
memref.dealloc %heap : memref<4x4xf32>
```

`memref.alloca` is valid only in function scope and produces a stack-allocated buffer that is automatically freed on function return. It does not require an explicit `memref.dealloc`. `memref.alloc` heap-allocates and requires explicit deallocation.

Both accept dynamic size operands for `?` dimensions. The operands appear in declaration order:

```mlir
%buf = memref.alloc(%rows, %cols) : memref<?x?xf32>
// rows → first ?, cols → second ?
```

### 138.1.3 Load and Store

```mlir
// Load: read a single element
%val = memref.load %buf[%i, %j] : memref<4x4xf32>

// Store: write a single element
memref.store %val, %buf[%i, %j] : memref<4x4xf32>

// Atomic load/store (for concurrency)
%old = memref.atomic_rmw addf %delta, %buf[%i] : (f32, memref<4xf32>) -> f32
```

Index operands must be of type `index`. For memrefs with dynamic shapes, the runtime will perform bounds checking only in debug builds unless `--mlir-disable-runtime-checks` is set. In release mode, no bounds checking is emitted.

### 138.1.4 `memref.subview` — Slicing Without Copies

`memref.subview` produces a view of a portion of a memref without copying data. It is one of the most powerful ops in the memref dialect:

```mlir
// Source memref
%src = memref.alloc() : memref<16x16xf32>

// Subview: rows 0..4, columns 0..4 (a 4x4 tile)
%tile = memref.subview %src[0, 0][4, 4][1, 1]
    : memref<16x16xf32> to memref<4x4xf32, strided<[16, 1], offset: 0>>

// Dynamic subview
%row_tile = memref.subview %src[%r, %c][%h, %w][1, 1]
    : memref<16x16xf32> to memref<?x?xf32, strided<[16, 1], offset: ?>>
```

The three bracket groups are: `[offsets]`, `[sizes]`, `[strides]`. When a dimension's size is 1 and the offset is the original extent, that dimension is dropped (rank reduction):

```mlir
// Extract a single row (rank reduction: 2D → 1D)
%row = memref.subview %src[%r, 0][1, 16][1, 1]
    : memref<16x16xf32> to memref<16xf32, strided<[1], offset: ?>>
```

The result type's layout map is computed by composing the subview's affine transformation with the source's layout map. No data movement occurs — the resulting memref points into the same underlying allocation with a different base offset and strides.

This is the foundation for tiled computation: tile a large buffer into smaller subviews, then pass each subview independently to a kernel that processes the tile.

### 138.1.5 Shape Manipulation

```mlir
// Cast between compatible types (layout must match)
%cast = memref.cast %typed : memref<4x4xf32> to memref<?x?xf32>

// Reinterpret cast: change shape/strides without checking compatibility
%view = memref.reinterpret_cast %flat to
    offset: [0], sizes: [4, 4], strides: [4, 1]
    : memref<16xf32> to memref<4x4xf32, strided<[4, 1], offset: 0>>

// Collapse shape: merge dimensions
%mat = memref.collapse_shape %tensor3d [[0, 1], [2]]
    : memref<2x4x8xf32> into memref<8x8xf32>

// Expand shape: split dimensions
%tensor3d = memref.expand_shape %mat [[0, 1], [2]]
    : memref<8x8xf32> into memref<2x4x8xf32>

// Reshape (arbitrary shape change, contiguous required)
%new = memref.reshape %old(%shape) : (memref<16xf32>, memref<2xi64>) -> memref<?x?xf32>
```

`memref.collapse_shape` and `memref.expand_shape` require the transformations to be expressible as affine operations — they cannot arbitrarily permute dimensions. `memref.reshape` is more flexible but requires the source to be contiguous.

### 138.1.6 Global and Symbolic Memory

```mlir
// Define a global constant in the module
memref.global "private" constant @weights : memref<256xf32> =
    dense<[1.0, 2.0, ...]> : tensor<256xf32>

// Access a global by symbol
%ptr = memref.get_global @weights : memref<256xf32>

// Hint to the optimizer about minimum alignment
memref.assume_alignment %buf, 64 : memref<?xf32>
```

`memref.global` is analogous to a global variable in C with an optional initializer. The `private` visibility makes it local to the module; `public` makes it externally linkable. `constant` asserts that the memory is read-only and enables constant folding through loads.

`memref.assume_alignment` provides alignment information without enforcing it at runtime. It is used to enable vectorized loads that require aligned memory.

### 138.1.7 Explicit Copying

```mlir
// Full copy (both must have same shape and element type)
memref.copy %src, %dst : memref<4x4xf32> to memref<4x4xf32>
```

`memref.copy` is semantically equivalent to a nested loop over all elements, but implementations can lower it to `memcpy` when layouts are compatible. It is primarily used after bufferization when a buffer needs to be materialized at a function boundary.

---

## 138.2 MemRef Layout Maps

### 138.2.1 The Strided Layout Model

Every `memref` has a layout that maps multi-dimensional indices to a linear byte offset. The canonical form is the strided layout:

```
offset(i₀, i₁, ..., iₙ) = base_offset + i₀·s₀ + i₁·s₁ + ... + iₙ·sₙ
```

This is represented as `strided<[s₀, s₁, ..., sₙ], offset: base_offset>`. Strides and the base offset can be static integers or `?` (dynamic, stored at runtime alongside the base pointer).

The default layout for `memref<N×M×f32>` is row-major: `strided<[M, 1], offset: 0>`. The fastest-varying dimension (last) has stride 1, and each slower dimension's stride is the product of all dimensions below it.

```mlir
// Explicit row-major (same as default)
memref<4x4xf32, strided<[4, 1], offset: 0>>

// Column-major (Fortran order)
memref<4x4xf32, strided<[1, 4], offset: 0>>

// Non-contiguous submatrix (stride-4 columns in a stride-1 base)
memref<4x4xf32, strided<[16, 1], offset: 8>>  // second 4x4 block in an 8x4 buffer
```

### 138.2.2 Layout Maps as Affine Maps

Internally, the strided layout is represented as an `AffineMap`. The strided form is syntactic sugar for:

```
affine_map<(d0, d1) -> (4*d0 + d1)>   // equivalent to strided<[4,1], offset:0>
```

More complex layouts — for blocked/packed formats used in cache-optimized matrix multiplication — can be expressed as more general affine maps:

```mlir
// 4×4 micro-tile packed layout: tile row × tile col → (t_r × 4 + t_c) * 16 + ...
memref<16x16xf32, affine_map<(d0, d1) -> (d0 floordiv 4, d1 floordiv 4,
                                           d0 mod 4, d1 mod 4)>>
```

This is a 4-dimensional linearization of a 2D buffer, encoding a "NCHW" style tiling. The `linalg.pack` / `linalg.unpack` operations automate this kind of layout transformation.

### 138.2.3 Subview Layout Composition

When `memref.subview` is applied, the result's layout map is computed by composing the subview's affine transformation with the source layout. For the common case of row-major source:

```
source:    strided<[N, 1], offset: 0>         // NxM row-major
subview:   offsets=[r, c], sizes=[h, w], strides=[sr, sc]
result:    strided<[N*sr, sc], offset: r*N + c>
```

This composition is performed symbolically at compile time when offsets/strides are static, and generates runtime address computations when they are dynamic.

---

## 138.3 The `bufferization` Dialect

### 138.3.1 The Tensor-to-MemRef Bridge

The `bufferization` dialect ([`mlir/include/mlir/Dialect/Bufferization/IR/BufferizationOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Bufferization/IR/BufferizationOps.td)) provides the explicit conversion ops between the immutable `tensor` world and the mutable `memref` world. These ops appear during and after the bufferization analysis phase.

```mlir
// Convert a tensor value to a memref (read-only view)
%mem = bufferization.to_memref %tensor : tensor<4x4xf32> to memref<4x4xf32>

// Declare that a memref will be treated as a tensor (immutable view)
%tens = bufferization.to_tensor %mem restrict writable
    : memref<4x4xf32> to tensor<4x4xf32>

// Allocate a tensor (will become a buffer during bufferization)
%empty = bufferization.alloc_tensor(%n, %m) : tensor<?x?xf32>

// Copy a tensor value into a specific destination buffer
bufferization.materialize_in_destination %source in writable %dest
    : tensor<4x4xf32> to memref<4x4xf32>
```

### 138.3.2 `bufferization.to_memref`

This op converts a tensor value into a memref with the same element type and shape. The semantics depend on whether the tensor was produced by an operation that already allocated a buffer:

- If the tensor was produced by `bufferization.alloc_tensor`, the memref directly views that buffer — zero copy.
- If the tensor was produced by another op (e.g., `linalg.generic`), the bufferization analysis determines whether an in-place buffer can be reused or whether a copy is needed.

The `restrict` attribute asserts that the resulting memref does not alias with any other live memref, enabling more aggressive optimization.

### 138.3.3 `bufferization.alloc_tensor`

`alloc_tensor` creates a tensor that conceptually needs a buffer — it is the escape hatch for situations where a new allocation is explicitly required:

```mlir
// Statically shaped allocation
%buf = bufferization.alloc_tensor() : tensor<16x16xf32>

// Dynamically shaped allocation
%buf_dyn = bufferization.alloc_tensor(%n, %m) : tensor<?x?xf32>

// Allocation that copies an existing tensor (avoids modifying the original)
%copy = bufferization.alloc_tensor() copy(%existing) : tensor<4x4xf32>
```

This op tells the bufferization framework "here is a point where a new allocation must occur." After one-shot bufferization, it becomes a `memref.alloc`.

---

## 138.4 One-Shot Bufferization

### 138.4.1 Overview

One-shot bufferization (OSB) is MLIR's primary algorithm for transforming tensor-based programs into equivalent memref-based programs while minimizing unnecessary copies ([`mlir/lib/Dialect/Bufferization/Transforms/OneShotAnalysis.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Dialect/Bufferization/Transforms/OneShotAnalysis.cpp)).

The fundamental challenge: tensor ops have value semantics — every "modification" conceptually creates a new value. But hardware has pointer semantics — modifications happen in-place. The question is: when can a tensor "update" (like `tensor.insert_slice`) be implemented as an in-place buffer write, and when must it copy the buffer first to preserve the original value?

OSB answers this by performing alias analysis to determine which tensor values share the same underlying buffer at any given program point. If the original tensor value is dead after the "update" op, no copy is needed — the update can be in-place. Otherwise, a copy is inserted.

### 138.4.2 Algorithm

The OSB algorithm runs in two phases:

**Phase 1 — Analysis**: For each tensor op with a "write" operand (e.g., `linalg.generic` output, `tensor.insert_slice` destination), determine whether the operand's buffer can be used in-place. The key query is: "Does any other live use of this tensor value exist downstream?"

The analysis computes:
- `OpOperand` equivalence sets (which operands share buffers)
- Read/write conflict detection using SSA use-def chains
- Conservative alias analysis for ops that do not implement `BufferizableOpInterface`

**Phase 2 — Bufferization**: Lower all tensor ops to memref ops, inserting `memref.alloc` where new allocations are needed and `memref.copy` where conflicts require preserving original values.

### 138.4.3 `OneShotBufferizationOptions`

Key configuration options:

```cpp
OneShotBufferizationOptions opts;

// Allow allocations in function return values (instead of forcing callers to own)
opts.allowReturnAllocs = true;

// Convert all function arguments/results (crossing function boundaries)
opts.functionBoundaryTypeConversion =
    BufferizationOptions::LayoutMapOption::IdentityLayoutMap;

// Copy behavior: never copy (fail if conflict), always copy, or auto
opts.copyBeforeWrite = false;

// Enable op-by-op bufferization analysis
opts.bufferizeFunctionBoundaries = true;
```

The `functionBoundaryTypeConversion` option controls how function arguments and results are converted. `IdentityLayoutMap` produces fully static layouts; `FullyDynamicLayoutMap` produces `memref<?x?x...xf32>` with dynamic layout for maximum generality at the cost of some optimization opportunity.

### 138.4.4 Running the Pipeline

```bash
# Complete one-shot bufferization pipeline
mlir-opt \
  --one-shot-bufferize="bufferize-function-boundaries=1 allow-return-allocs=1" \
  --buffer-deallocation-pipeline \
  --buffer-loop-hoisting \
  --convert-bufferization-to-memref \
  input.mlir
```

The `--buffer-deallocation-pipeline` pass inserts `memref.dealloc` ops at the appropriate control flow points. MLIR's buffer deallocation analysis tracks buffer ownership through control flow — including branches and loops — using a liveness analysis to determine the last use of each buffer.

`--buffer-loop-hoisting` moves `memref.alloc` ops out of loop bodies when the allocation's size is loop-invariant, avoiding repeated allocation/deallocation overhead.

### 138.4.5 `BufferizableOpInterface`

Dialects opt into OSB by implementing `BufferizableOpInterface` ([`mlir/include/mlir/Dialect/Bufferization/IR/BufferizableOpInterface.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Bufferization/IR/BufferizableOpInterface.h)):

```cpp
// Key methods to implement:
SmallVector<OpOperand *> getAliasingOpOperands(Operation *op, Value value,
                                                AnalysisState &state);
bool isWritable(Operation *op, Value value, AnalysisState &state);
LogicalResult bufferize(Operation *op, RewriterBase &rewriter,
                         const BufferizationState &state);
```

`getAliasingOpOperands` tells the analysis which input operands "alias" with which output operands — i.e., which outputs can potentially share a buffer with which inputs. For `linalg.generic`, inputs alias with nothing; the output aliasing with itself is the key decision the analysis makes.

`isWritable` answers "can this op's output buffer be written to without copying?" For most linalg ops, the answer is yes if the output operand was not read before the op.

`bufferize` performs the actual rewrite: given a populated `BufferizationState` (which maps each tensor value to its buffer), rewrite the op to use memrefs.

### 138.4.6 Ownership-Based Buffer Deallocation

After OSB produces memref code, the `BufferDeallocationOpInterface` framework inserts `memref.dealloc` ops. The algorithm:

1. Compute liveness of each buffer (first def to last use) across CFG edges
2. Identify "ownership" — which basic block/function owns each buffer (responsible for deallocation)
3. Insert `memref.dealloc` at the end of the owning scope, handling:
   - Conditional dealloc after branches with different ownership
   - Loop-exit deallocs for buffers allocated inside loops
   - Return-value ownership transfer (callee allocates, caller deallocates)

This is the MLIR counterpart of RAII or reference counting, implemented via SSA analysis rather than runtime overhead.

### 138.4.7 Debugging Bufferization

OSB failures typically manifest as:

1. **"failed to bufferize op"** — the op does not implement `BufferizableOpInterface` and is not in the fallback list. Solution: generalize the op or add a custom interface implementation.

2. **"value not available as a buffer"** — the bufferization state does not have a mapping for a tensor value. Often caused by ops that create new tensor values without being bufferizable.

3. **Unexpected copies** — OSB inserted a copy where you expected in-place. Use `--mlir-print-op-generic` to inspect the IR and look for `memref.copy` after the pass. The `--debug` flag on `--one-shot-bufferize` prints the aliasing decisions.

```bash
# Debug mode: print analysis decisions
mlir-opt --one-shot-bufferize="bufferize-function-boundaries=1" \
         --mlir-disable-threading \
         --debug-only=one-shot-bufferization \
         input.mlir 2>&1 | head -200
```

---

## 138.5 Complete Bufferization Example

To tie together the concepts, here is a complete example of a tensor addition being bufferized:

**Before bufferization (tensor world):**

```mlir
func.func @add(%A: tensor<4x4xf32>, %B: tensor<4x4xf32>) -> tensor<4x4xf32> {
  %init = tensor.empty() : tensor<4x4xf32>
  %result = linalg.generic {
    indexing_maps = [
      affine_map<(d0, d1) -> (d0, d1)>,
      affine_map<(d0, d1) -> (d0, d1)>,
      affine_map<(d0, d1) -> (d0, d1)>
    ],
    iterator_types = ["parallel", "parallel"]
  } ins(%A, %B : tensor<4x4xf32>, tensor<4x4xf32>)
    outs(%init : tensor<4x4xf32>) {
  ^bb0(%a: f32, %b: f32, %c: f32):
    %sum = arith.addf %a, %b : f32
    linalg.yield %sum : f32
  } -> tensor<4x4xf32>
  func.return %result : tensor<4x4xf32>
}
```

**After one-shot bufferization:**

```mlir
func.func @add(%A: memref<4x4xf32>, %B: memref<4x4xf32>,
               %result: memref<4x4xf32>) {
  linalg.generic {
    indexing_maps = [
      affine_map<(d0, d1) -> (d0, d1)>,
      affine_map<(d0, d1) -> (d0, d1)>,
      affine_map<(d0, d1) -> (d0, d1)>
    ],
    iterator_types = ["parallel", "parallel"]
  } ins(%A, %B : memref<4x4xf32>, memref<4x4xf32>)
    outs(%result : memref<4x4xf32>) {
  ^bb0(%a: f32, %b: f32, %c: f32):
    %sum = arith.addf %a, %b : f32
    linalg.yield %sum : f32
  }
  return
}
```

Notice that:
- The output tensor `%init` became the `%result` output argument — no allocation inside the function.
- The function signature changed: the output tensor becomes an output memref argument (with `functionBoundaryTypeConversion = IdentityLayoutMap`).
- `linalg.generic` works directly on memrefs in the same structure as the tensor version.
- No `memref.alloc` or `memref.dealloc` were needed because the result buffer is owned by the caller.

---

## Chapter 138 Summary

- `MemRefType` encodes element type, shape, strided layout (as an affine map), and memory space. Dynamic shapes use `?` for runtime-determined dimensions.
- Core memref ops: `memref.alloc`/`memref.alloca`/`memref.dealloc` for allocation; `memref.load`/`memref.store` for element access; `memref.subview` for zero-copy slicing with layout map recomputation.
- `memref.subview` is the foundation for tiled computation — it produces a view into a tile of a larger buffer with updated strides and offset, enabling cache-efficient loop nests without data copies.
- MemRef layout maps are affine maps. Strided layouts (`strided<[s0, s1], offset: o>`) cover row-major, column-major, and arbitrary strided access. General affine maps express blocked/packed formats for cache-optimized GEMM.
- The `bufferization` dialect provides `to_memref`, `to_tensor`, `alloc_tensor`, and `materialize_in_destination` — explicit bridges between tensor and memref worlds.
- One-shot bufferization (OSB) analyzes tensor programs for aliasing conflicts and converts them to equivalent memref programs with minimal copies. It is driven by `BufferizableOpInterface` implementations on each op.
- `OneShotBufferizationOptions` controls function boundary handling, return allocation policies, and copy insertion behavior. The post-bufferization pipeline (`--buffer-deallocation-pipeline`, `--buffer-loop-hoisting`) inserts deallocations and hoists invariant allocations.
- The ownership-based deallocation model inserts `memref.dealloc` via SSA liveness analysis, handling branches, loops, and cross-function ownership transfer without runtime reference counting.
