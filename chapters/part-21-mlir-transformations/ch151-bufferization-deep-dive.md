# Chapter 151 — Bufferization Deep Dive

*Part XXI — MLIR Transformations*

Bufferization is the process of converting tensor semantics—where each operation produces an immutable new tensor value—to buffer semantics—where operations read and write mutable memory. The challenge is not the conversion itself but the minimization of copies: naive bufferization introduces a copy at every tensor-producing operation, defeating the purpose of tensor-level optimizations. MLIR's One-Shot Bufferization performs a global alias analysis across the entire program before inserting any copy, placing copies only where aliasing would produce incorrect results. The result is competitive with hand-tuned buffer-passing code in many cases. This chapter dissects the algorithm, the extension interfaces, and the practical pipeline for turning a tensor-valued program into efficient buffer-passing code.

---

## 151.1 The Tensor-Buffer Problem

### Value semantics vs. buffer semantics

Tensor operations in MLIR are defined with **value semantics**: each SSA value is conceptually a distinct immutable tensor. A `linalg.matmul` with tensor output type produces a new tensor; the input tensors are untouched. This enables clean optimization: CSE, DCE, and loop fusion all work naturally on tensor IR without alias analysis.

Buffer operations (`memref`) have **aliasing semantics**: a `memref.load` may see values written by a previous `memref.store` if they alias. This is necessary for final code generation but makes optimization much harder.

The goal of bufferization is to assign each tensor SSA value to a memory buffer, potentially sharing buffers between values that are provably non-conflicting—i.e., where the value is no longer used when the buffer is overwritten.

### Why not naive bufferization?

Consider:

```mlir
%result = linalg.matmul ins(%A, %B : tensor<?x?xf32>, tensor<?x?xf32>)
                        outs(%C : tensor<?x?xf32>)
          -> tensor<?x?xf32>
```

A naive approach allocates a fresh buffer for `%result`, copies `%C` into it (to satisfy the `outs` semantics), and runs the matmul. But if `%C` is not used after this point, the copy is unnecessary: the matmul can write directly into the buffer backing `%C`.

One-Shot Bufferization detects this pattern through **in-place analysis**: it determines that `%result` can alias `%C`'s buffer because there are no read uses of `%C` after the matmul that would conflict.

### Relevant sources

- [`mlir/include/mlir/Dialect/Bufferization/IR/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Bufferization/IR/)
- [`mlir/include/mlir/Dialect/Bufferization/Transforms/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Bufferization/Transforms/)
- [`mlir/lib/Dialect/Bufferization/Transforms/OneShotAnalysis.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Dialect/Bufferization/Transforms/OneShotAnalysis.cpp)
- [`mlir/lib/Dialect/Bufferization/Transforms/OneShotModuleBufferize.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Dialect/Bufferization/Transforms/OneShotModuleBufferize.cpp)

---

## 151.2 One-Shot Bufferization Overview

### Two-phase algorithm

One-Shot Bufferization runs in two completely separate phases:

**Phase 1 — Analysis**: Walk the IR in a specific order (use-before-def, post-order across operations). For each tensor-producing operation, determine whether its result can be placed in-place in one of its operand buffers. No IR is modified. The analysis produces a `AnalysisState` that records, for each op and each OpOperand, whether the use is in-place.

**Phase 2 — Rewrite**: Walk the IR again, applying the decisions from Phase 1. For each op, call `bufferize(rewriter, ...)` from its `BufferizableOpInterface` implementation. Ops where Phase 1 decided in-place: no copy is inserted. Ops where Phase 1 decided out-of-place: an allocation + copy is inserted before the buffer is passed to the op.

### Running One-Shot Bufferization

```bash
# Basic invocation:
/usr/lib/llvm-22/bin/mlir-opt \
  --one-shot-bufferize \
  input.mlir -o bufferized.mlir

# With function boundary conversion:
/usr/lib/llvm-22/bin/mlir-opt \
  --one-shot-bufferize="bufferize-function-boundaries=true \
                        function-boundary-type-conversion=infer-layout-map" \
  input.mlir -o bufferized.mlir
```

### OneShotBufferizationOptions

```cpp
mlir::bufferization::OneShotBufferizationOptions opts;

// Bufferize function signatures and return types:
opts.bufferizeFunctionBoundaries = true;

// How to convert function argument/result memref types:
// Options: InferLayoutMap, FullyDynamicLayoutMap, IdentityLayoutMap
opts.functionBoundaryTypeConversion =
    mlir::bufferization::LayoutMapOption::InferLayoutMap;

// Allow loop-body allocations to be returned:
opts.allowReturnAllocsFromLoops = false;

// Test mode: do not actually rewrite, just annotate analysis results:
opts.testAnalysisOnly = false;

// Print alias sets for debugging:
opts.dumpAliasSets = false;

// OpFilter: only bufferize ops from specific dialects:
opts.opFilter.allowDialect("linalg");
opts.opFilter.allowDialect("tensor");

mlir::LogicalResult result =
    mlir::bufferization::runOneShotBufferize(moduleOp, opts);
```

---

## 151.3 BufferizableOpInterface

Every op that participates in bufferization must implement `BufferizableOpInterface`. This interface has methods that the analysis and rewrite phases call.

### Interface declaration (ODS)

```tablegen
// In YourDialect.td:
def YourOp : Op<YourDialect, "your_op",
    [DeclareOpInterfaceMethods<BufferizableOpInterface>]> {
  // ...
}
```

### C++ implementation

```cpp
// In YourOpBufferizableOpInterfaceImpl.cpp:
#include "mlir/Dialect/Bufferization/IR/BufferizableOpInterface.h"

namespace {
struct YourOpBufferizationInterface
    : public mlir::bufferization::BufferizableOpInterface::ExternalModel<
          YourOpBufferizationInterface, YourOp> {

  // Does the op read from the OpOperand at the given index?
  bool bufferizesToMemoryRead(
      mlir::Operation *op, mlir::OpOperand &opOperand,
      const mlir::bufferization::AnalysisState &state) const {
    // Your op reads all operands:
    return true;
  }

  // Does the op write to the OpOperand at the given index (in-place)?
  bool bufferizesToMemoryWrite(
      mlir::Operation *op, mlir::OpOperand &opOperand,
      const mlir::bufferization::AnalysisState &state) const {
    // Your op writes only the output operand (index 2):
    return opOperand.getOperandNumber() == 2;
  }

  // Which results alias the given OpOperand?
  mlir::AliasingValueList getAliasingValues(
      mlir::Operation *op, mlir::OpOperand &opOperand,
      const mlir::bufferization::AnalysisState &state) const {
    // Result 0 aliases operand 2:
    if (opOperand.getOperandNumber() == 2)
      return {{op->getResult(0),
               mlir::bufferization::BufferRelation::Equivalent}};
    return {};
  }

  // What memref type should the result have?
  mlir::FailureOr<mlir::BaseMemRefType> getBufferType(
      mlir::Operation *op, mlir::Value value,
      const mlir::bufferization::BufferizationOptions &options,
      const mlir::bufferization::BufferizationState &state,
      llvm::SmallVector<mlir::Value> &invocationStack) const {
    auto tensorType = mlir::cast<mlir::RankedTensorType>(value.getType());
    return mlir::bufferization::getMemRefTypeWithStaticIdentityLayout(
        tensorType, options);
  }

  // Perform the actual bufferization rewrite:
  mlir::LogicalResult bufferize(
      mlir::Operation *op,
      mlir::RewriterBase &rewriter,
      const mlir::bufferization::BufferizationOptions &options,
      mlir::bufferization::BufferizationState &state) const {
    auto yourOp = mlir::cast<YourOp>(op);
    // Get the bufferized operands (already converted by the time this is called):
    mlir::FailureOr<mlir::Value> inputBuf =
        state.getBuffer(rewriter, yourOp.getInput());
    mlir::FailureOr<mlir::Value> outputBuf =
        state.getBuffer(rewriter, yourOp.getOutput());
    if (mlir::failed(inputBuf) || mlir::failed(outputBuf))
      return mlir::failure();

    // Create the buffer-semantic version of your op:
    rewriter.create<YourBufferOp>(op->getLoc(), *inputBuf, *outputBuf);
    mlir::bufferization::replaceOpWithBufferizedValues(
        rewriter, op, {*outputBuf});
    return mlir::success();
  }
};
} // namespace

// Register the external model:
void mlir::yourDialect::registerBufferizableOpInterfaceExternalModels(
    mlir::DialectRegistry &registry) {
  registry.addExtension(+[](mlir::MLIRContext *ctx,
                             mlir::yourDialect::YourDialect *) {
    YourOp::attachInterface<YourOpBufferizationInterface>(*ctx);
  });
}
```

---

## 151.4 Copy Elision Analysis

### The core question

For each tensor-producing op, the analysis asks: can the result share a buffer with one of the op's tensor operands without causing incorrect behavior?

The answer is yes if and only if there is no use of the aliased operand's buffer that would observe the in-progress write. More precisely:

A result `r` of op `O` with operand `x` can be placed in-place in `x`'s buffer if there is no read use of `x`'s buffer (by any op that may execute after `O` starts writing) before `x`'s buffer is next overwritten.

### Read-after-write analysis

The analysis walks the def-use chain in reverse post-order. For each potential in-place placement, it checks whether `wouldCreateReadAfterWriteHazard(buffer, op)` returns true. If it does, a copy must be inserted.

The check traverses all uses of the operand's buffer:
1. If a use is a read that **dominates** the op: it occurs before the write—safe.
2. If a use is a read that **post-dominates** the op: it occurs after the write is complete—safe.
3. If a use is a read that may be **concurrent** or in a **different iteration** of a loop: unsafe—must copy.

### Alias sets

The analysis maintains a union-find data structure (`AliasInfo`) tracking which tensors may alias each other. When an in-place decision is made (result `r` aliases operand `x`), `r` and `x` are merged into the same alias set. This propagates: if `r` is then passed in-place to another op, the alias set grows.

The alias sets are conservatively propagated through:

- `tensor.extract_slice` / `tensor.insert_slice`: the extracted slice aliases the original tensor.
- `scf.for` iter_args: the iteration argument may alias its initializer.
- `func.return`: return values may alias function arguments.

### `AnalysisState::isInPlace`

After the analysis phase, `AnalysisState::isInPlace(opOperand)` returns true if the operand was determined to be safe for in-place operation. The rewrite phase queries this to decide whether to insert a copy.

---

## 151.5 Function Boundary Bufferization

Bufferizing across function boundaries is the hardest part of One-Shot Bufferization because it requires making consistent decisions about which function arguments alias which return values.

### The challenge

A function that takes a tensor argument and returns a modified tensor can be called from multiple sites. If we decide at one call site that the result aliases the first argument, we must enforce the same decision at all other call sites. This requires a module-level analysis pass (`--one-shot-bufferize="bufferize-function-boundaries=true"`), not just a per-function pass.

### Layout map options

The `functionBoundaryTypeConversion` option controls how tensor argument types are converted to memref argument types:

| Option | Effect |
|--------|--------|
| `InferLayoutMap` | Infer the tightest memref layout that matches the tensor access pattern. May produce non-trivial strides. |
| `FullyDynamicLayoutMap` | Use `memref<?x?xf32, strided<[?, ?], offset: ?>>` — maximally general, minimizes recompilation |
| `IdentityLayoutMap` | Use `memref<?x?xf32>` — contiguous row-major; simplest but may require additional copies |

For most use cases, `InferLayoutMap` produces the best performance. `FullyDynamicLayoutMap` is useful for shared library interfaces.

### Function argument handling

```mlir
// Input (tensor semantics):
func.func @add(%A: tensor<?xf32>, %B: tensor<?xf32>)
    -> tensor<?xf32> {
  %C = linalg.add ins(%A, %B : tensor<?xf32>, tensor<?xf32>)
                  outs(%A : tensor<?xf32>) -> tensor<?xf32>
  return %C : tensor<?xf32>
}

// After bufferization with bufferize-function-boundaries:
func.func @add(%A: memref<?xf32>, %B: memref<?xf32>,
               %result: memref<?xf32>) {
  linalg.add ins(%A, %B : memref<?xf32>, memref<?xf32>)
             outs(%result : memref<?xf32>)
}
```

Tensor return values become additional `memref` arguments (out-params). The function type changes; all call sites must also be updated, which the module-level analysis handles.

---

## 151.6 Buffer Deallocation

Bufferization allocates buffers (`memref.alloc`) but does not insert `memref.dealloc` calls. A separate pipeline handles deallocation.

### The buffer deallocation pipeline

```bash
/usr/lib/llvm-22/bin/mlir-opt \
  --one-shot-bufferize="bufferize-function-boundaries=true" \
  --buffer-deallocation-pipeline \
  input.mlir -o output.mlir
```

The `--buffer-deallocation-pipeline` is a convenience alias for:

```
ownership-based-buffer-deallocation,
buffer-deallocation-simplification,
bufferization-lower-deallocations,
canonicalize,
cse
```

### Ownership-based buffer deallocation

The algorithm assigns ownership to each buffer:

- A buffer created by `memref.alloc` is initially owned by the current block.
- When ownership is transferred (e.g., stored in a data structure or returned from a function), the current block relinquishes ownership.
- Each buffer is deallocated exactly once: in the block that holds ownership at the point the buffer exits scope.

```mlir
// After deallocation insertion:
func.func @example() -> memref<10xf32> {
  %buf = memref.alloc() : memref<10xf32>
  // ... use %buf ...
  return %buf : memref<10xf32>  // ownership transferred to caller
  // No dealloc here: caller is responsible.
}
```

### BufferDeallocationOpInterface

Ops with custom region semantics (e.g., `scf.if`, `scf.for`) implement `BufferDeallocationOpInterface` to specify how ownership flows through their regions:

```cpp
// scf.if: ownership of yielded buffers flows to the if op's result.
// scf.for: iter_args that are buffers carry ownership through iterations.
```

### `-buffer-results-to-out-params`

After deallocation insertion, function results that are `memref` may cause issues at link boundaries (the caller doesn't know the allocation size). The `--buffer-results-to-out-params` pass converts:

```mlir
// Before:
func.func @alloc_and_return() -> memref<10xf32> { ... }

// After:
func.func @alloc_and_return(%result: memref<10xf32>) { ... }
```

The caller pre-allocates the buffer and passes it as an argument.

---

## 151.7 Debugging Bufferization

### Test mode

`--test-one-shot-bufferize` runs only the analysis phase and annotates ops with the analysis decisions:

```bash
/usr/lib/llvm-22/bin/mlir-opt \
  --test-one-shot-bufferize \
  input.mlir
```

Output annotations:

```mlir
// "bufferization.access" shows the analysis result per OpOperand:
%result = linalg.matmul ins(%A {bufferization.access = "read"},
                             %B {bufferization.access = "read"})
                        outs(%C {bufferization.access = "read-write"})
    -> tensor<?x?xf32>
// "bufferization.inplace" shows in-place decisions:
// #bufferization.in_place = "true" means no copy is needed.
```

### Alias set dumping

```bash
/usr/lib/llvm-22/bin/mlir-opt \
  --one-shot-bufferize="dump-alias-sets=true" \
  input.mlir 2>&1 | grep "AliasSet"
```

This prints the alias sets before the rewrite phase, showing which tensor SSA values are grouped together.

### Common issues

**"failed to bufferize op"**: The op does not implement `BufferizableOpInterface`. Fix: implement the interface, or add an external model registration, or pre-process the op with canonicalization to remove it.

**"could not find a buffer for"**: The analysis could not resolve which buffer backs a tensor value. Common cause: tensor values flowing through ops that are not bufferizable (treated as black boxes).

**"copy inserted at"**: A copy was inserted because in-place analysis failed. Use `--dump-alias-sets=true` to see why the alias check failed.

**Excessive copies in loops**: If an `scf.for` iter_arg is not recognized as aliasing its initializer, every iteration will copy. Ensure the iter_arg's op implements `getAliasingValues` returning `BufferRelation::Equivalent` for the corresponding operand/result pair.

### Visualizing the analysis

For complex programs, it helps to run the test mode, then inspect which operands are marked `"read-write"` (potentially in-place) vs `"read"` (not written, so definitely safe to alias):

```bash
/usr/lib/llvm-22/bin/mlir-opt \
  --test-one-shot-bufferize \
  --mlir-print-ir-after-all \
  input.mlir 2>&1 | grep -A5 "bufferization.access"
```

---

## 151.8 Partial Bufferization and OpFilter

One-Shot Bufferization can be run in partial mode, converting only a subset of ops:

```cpp
mlir::bufferization::OneShotBufferizationOptions opts;
// Only bufferize ops from the linalg dialect:
opts.opFilter.allowDialect("linalg");
// Also allow specific ops:
opts.opFilter.allowOp<mlir::tensor::InsertSliceOp>();
// Deny specific ops (prevent them from being bufferized):
opts.opFilter.denyOp<mlir::func::ReturnOp>();
```

Partial bufferization is useful when:
- The program contains custom dialect ops that are not yet bufferizable.
- Bufferization should be deferred for some regions (e.g., GPU device code handled separately from host code).

---

## Chapter Summary

- **One-Shot Bufferization** converts tensor-semantic IR to memref-semantic IR with minimal copies, using a two-phase (analysis + rewrite) algorithm.
- The core optimization is **in-place analysis**: determine which tensor results can share a buffer with an operand, eliminating the allocation and copy that naive conversion would insert.
- **`BufferizableOpInterface`** is the extension point: every op participating in bufferization implements `bufferizesToMemoryRead`, `bufferizesToMemoryWrite`, `getAliasingValues`, `getBufferType`, and `bufferize`.
- **Function boundary bufferization** (`bufferize-function-boundaries=true`) handles cross-function tensor passing; `functionBoundaryTypeConversion` controls the memref layout used.
- **Buffer deallocation** is a separate pass pipeline (`--buffer-deallocation-pipeline`) using ownership tracking to insert `memref.dealloc` exactly once per allocation.
- **`--test-one-shot-bufferize`** annotates the IR with analysis results (`bufferization.access`, `bufferization.inplace`) without rewriting—invaluable for debugging unexpected copies.
- **`OpFilter`** enables partial bufferization: only specified ops/dialects are converted, deferring others to a later pass or a specialized handling.


---

@copyright jreuben11
