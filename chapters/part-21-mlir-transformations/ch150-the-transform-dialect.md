# Chapter 150 — The Transform Dialect

*Part XXI — MLIR Transformations*

The conventional MLIR pass pipeline is static: a fixed sequence of passes executed in a fixed order, parameterized only by coarse-grained options like tile sizes. This works for general compilation but breaks down for specialized workloads—deep learning kernels, HPC stencils, cryptographic primitives—where the optimal transformation sequence depends on fine-grained structural properties of the payload IR. The Transform dialect addresses this by making the compiler transformation itself an MLIR program: a sequence of operations that query, match, and mutate a "payload" IR, interpreted at compile time. The result is a metaprogramming system for compilers: expressive, composable, and testable in isolation.

---

## 150.1 The Transform Dialect Concept

### The problem with static pipelines

A static pipeline like `--tile --vectorize --lower` applies the same strategy uniformly. But a matrix multiply operation may need tiles of `[64, 64, 32]` while a depthwise convolution needs `[1, 1, C]`. Encoding this variation in pass options requires exponentially many flags, or worse, special-casing inside pass implementations.

The Transform dialect solves this by separating:

- **What to transform**: expressed using `transform.structured.match` and related query ops that select specific payload ops by name, attribute, or interface.
- **How to transform**: expressed using `transform.structured.tile_using_for`, `transform.structured.vectorize`, etc.—each operation invoking a specific MLIR transformation on the matched payload.
- **Failure handling**: `failures(propagate)` vs `failures(suppress)` control whether transformation errors abort or are silently ignored.

### Two-program architecture

When the Transform dialect is active, the compilation unit contains two programs:

1. **Payload IR**: the program being compiled (e.g., a `func.func` with `linalg.generic` ops).
2. **Transform IR**: an MLIR program that describes how to transform the payload (e.g., a `transform.sequence` op containing tile/vectorize/lower ops).

The transform interpreter executes the Transform IR, using it as a script to mutate the payload IR. The Transform IR itself is pure metadata—it is not compiled to machine code.

### Key source locations

- [`mlir/include/mlir/Dialect/Transform/IR/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Transform/IR/)
- [`mlir/lib/Dialect/Transform/IR/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Dialect/Transform/IR/)
- [`mlir/include/mlir/Dialect/Linalg/TransformOps/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Linalg/TransformOps/)

---

## 150.2 Core Transform Operations

### transform.sequence

The top-level container for a sequence of transformations:

```mlir
transform.sequence failures(propagate) {
^bb0(%module: !transform.any_op):
  // %module is a handle to the top-level module.
  // Operations inside this block transform the payload.
  %funcs = transform.structured.match ops{["func.func"]} in %module
      : (!transform.any_op) -> !transform.any_op
  // ... apply transformations to %funcs ...
}
```

`failures(propagate)` means: if any operation inside the sequence returns a definite failure, propagate that failure and stop execution. Alternatives:

| Mode | Behavior |
|------|----------|
| `propagate` | Failure in one op aborts the sequence |
| `suppress` | Failures are silently ignored; execution continues |
| `allow` | Failures are expected; the sequence continues |

### Handle types

Transform ops communicate via **handles**—SSA values that represent references to payload IR operations, values, or parameters:

| Type | Represents |
|------|-----------|
| `!transform.any_op` | A set of payload operations |
| `!transform.op<"dialect.op">` | A set of ops of the named type |
| `!transform.any_value` | A set of payload SSA values |
| `!transform.param<i64>` | A compile-time integer parameter |

Handles are not MLIR values in the payload IR—they are metadata living in the transform IR that the interpreter resolves to actual ops/values in the payload.

### transform.structured.match

The primary query mechanism:

```mlir
// Match all ops with the given name:
%matmuls = transform.structured.match ops{["linalg.matmul"]} in %module
    : (!transform.any_op) -> !transform.any_op

// Match ops with a specific attribute:
%marked = transform.structured.match attributes{tag = "to_tile"} in %module
    : (!transform.any_op) -> !transform.any_op

// Match ops implementing an interface:
%linalgOps = transform.structured.match interface{LinalgOp} in %module
    : (!transform.any_op) -> !transform.any_op
```

### transform.get_parent_op

Navigate up the payload op tree:

```mlir
%parent = transform.get_parent_op %child {op_name = "func.func"}
    : (!transform.any_op) -> !transform.any_op
```

### transform.apply_patterns

Apply a named pattern set to a matched op:

```mlir
transform.apply_patterns to %func {
  transform.apply_patterns.linalg.tiling_canonicalization
  transform.apply_patterns.canonicalization
} : !transform.any_op
```

### transform.apply_registered_pass

Run a named MLIR pass (registered in the pass registry) on a matched op:

```mlir
transform.apply_registered_pass "canonicalize" to %func
    : (!transform.any_op) -> !transform.any_op
```

---

## 150.3 Structured (Linalg) Transformations

The `transform.structured.*` namespace provides high-level transformations for `LinalgOp` instances.

### Tiling

```mlir
// Tile a matmul into three nested scf.for loops:
%tiled_matmul, %loops:3 = transform.structured.tile_using_for %matmul
    tile_sizes [64, 64, 32]
    : (!transform.any_op) -> (!transform.any_op,
                               !transform.any_op,
                               !transform.any_op,
                               !transform.any_op)

// %loops:3 are handles to the three generated scf.for ops.
// %tiled_matmul is a handle to the tiled linalg.matmul inside the loops.
```

Tile size 0 means "do not tile this dimension" (the loop is not generated for that dimension).

### Fusing producers

```mlir
// Fuse the fill op (producer) into the tiled matmul consumer:
%fused, %loops2:3 = transform.structured.fuse_into_containing_op %fill
    into %loops#0
    : (!transform.any_op, !transform.any_op)
      -> (!transform.any_op, !transform.any_op)
```

Producer fusion eliminates the intermediate buffer for `fill` by executing it inside the consumer's tile loop.

### Vectorization

```mlir
// Vectorize all matched linalg ops (uses target vector width):
transform.structured.vectorize %linalg_op
    : !transform.any_op

// Vectorize with explicit vector sizes:
transform.structured.vectorize %linalg_op
    vector_sizes [8, 8]
    : !transform.any_op
```

### Padding

```mlir
// Pad operands to multiples for better vector alignment:
%padded, %pad, %copy_back =
    transform.structured.pad %matmul {
      padding_values = [0.0 : f32, 0.0 : f32, 0.0 : f32],
      padding_dimensions = [0, 1, 2],
      pack_paddings = [1, 1, 0],
      copy_back_op = "linalg.copy"
    } : (!transform.any_op) -> (!transform.any_op,
                                 !transform.any_op,
                                 !transform.any_op)
```

### Decomposition and specialization

```mlir
// Decompose conv2d_nhwc_hwcf into a sequence of simpler linalg ops:
transform.structured.decompose_convolutions %conv
    : !transform.any_op

// Generalize named op to linalg.generic:
transform.structured.generalize %matmul
    : (!transform.any_op) -> !transform.any_op
```

---

## 150.4 Loop Transformations

### Unrolling

```mlir
// Unroll the innermost loop by a factor of 4:
transform.loop.unroll %inner_loop factor = 4
    : !transform.any_op
```

### Loop peeling

```mlir
// Peel the last partial iteration from the loop:
%main_loop, %remainder =
    transform.loop.peel %loop
    : (!transform.any_op) -> (!transform.any_op, !transform.any_op)
```

Peeling creates a main loop (with full tiles) and a remainder loop (with the partial last tile), enabling the main loop to be vectorized without a mask.

### Software pipelining

```mlir
// Pipeline the loop with stage depth 2:
transform.loop.pipeline %loop {
    iteration_interval = 1,
    read_latency = 3,
    scheduling_strategy = "greedy-modulo-scheduling"
} : !transform.any_op
```

### Loop coalescing

```mlir
// Collapse a two-level loop nest into a single loop:
%coalesced = transform.loop.coalesce %outer_loop
    : (!transform.any_op) -> !transform.any_op
```

### Getting enclosing loops

```mlir
// Get the parent scf.for of a linalg op (for subsequent loop transforms):
%loop = transform.loop.get_parent_for %tiled_op {num_loops = 1}
    : (!transform.any_op) -> !transform.any_op
```

---

## 150.5 Named Sequences and Composability

### transform.named_sequence

A named sequence is a reusable named transform block, analogous to a function:

```mlir
// Definition:
transform.named_sequence @tile_and_vectorize(
    %target: !transform.any_op {transform.readonly})
    -> !transform.any_op {
  %tiled, %loops:3 = transform.structured.tile_using_for %target
      tile_sizes [64, 64, 32]
      : (!transform.any_op) -> (!transform.any_op,
                                 !transform.any_op,
                                 !transform.any_op,
                                 !transform.any_op)
  transform.structured.vectorize %tiled : !transform.any_op
  transform.yield %tiled : !transform.any_op
}

// Call site:
transform.sequence failures(propagate) {
^bb0(%module: !transform.any_op):
  %matmuls = transform.structured.match ops{["linalg.matmul"]} in %module
      : (!transform.any_op) -> !transform.any_op
  %result = transform.include @tile_and_vectorize failures(propagate) (%matmuls)
      : (!transform.any_op) -> !transform.any_op
}
```

Named sequences enable reuse across compilation units—multiple kernels can invoke the same transformation recipe.

### transform.foreach_match

Apply different transforms to different ops in a single traversal:

```mlir
transform.foreach_match in %module
    @match_matmul -> @tile_matmul,
    @match_conv   -> @tile_conv
    : (!transform.any_op) -> !transform.any_op
```

Where `@match_matmul` and `@match_conv` are named sequences that return success/failure based on op properties, and `@tile_matmul` / `@tile_conv` are the corresponding transformation sequences.

---

## 150.6 Interactivity and Extensibility

### Adding new transform operations

New transform ops are defined using ODS like any other MLIR op, but they implement `TransformOpInterface`:

```cpp
// C++ implementation of a custom transform op:
mlir::DiagnosedSilenceableFailure MyCustomTransformOp::apply(
    mlir::transform::TransformRewriter &rewriter,
    mlir::transform::TransformResults &results,
    mlir::transform::TransformState &state) {

  // Get the payload ops referenced by the handle:
  auto payloadOps = state.getPayloadOps(getTarget());

  for (mlir::Operation *payloadOp : payloadOps) {
    // Apply the transformation:
    if (mlir::failed(applyMyTransform(rewriter, payloadOp))) {
      return mlir::emitSilenceableFailure(getLoc())
             << "transformation failed for " << *payloadOp;
    }
  }

  // Record the results (new handles for created ops):
  results.set(getResults()[0], newOps);
  return mlir::DiagnosedSilenceableFailure::success();
}
```

`DiagnosedSilenceableFailure` has three states:
- **Success**: transformation succeeded.
- **Silenceable failure**: transformation did not apply but this is not fatal (can be `suppress`-ed).
- **Definite failure**: a hard error (always propagated).

### TransformState

`TransformState` is the interpreter's bookkeeping object:

```cpp
// Get all payload ops referenced by a handle:
llvm::ArrayRef<mlir::Operation *>
    ops = state.getPayloadOps(handle);

// Get all payload values referenced by a handle:
llvm::ArrayRef<mlir::Value>
    vals = state.getPayloadValues(handle);

// Set result handles:
results.set(resultHandle, {newOp1, newOp2});
```

`TransformState` enforces linearity: handles that are consumed cannot be reused. This prevents transformations from referencing payload ops that have been erased by a previous step.

---

## 150.7 The Transform Interpreter

### Running transforms programmatically

```cpp
#include "mlir/Dialect/Transform/Transforms/TransformInterpreterUtils.h"

// payloadOp: the IR to transform (e.g., a ModuleOp)
// transformOp: the top-level transform.sequence op
mlir::transform::TransformOptions options;
options.setExpensiveChecksEnabled(true); // enable invariant checking

mlir::LogicalResult result =
    mlir::transform::applyTransforms(
        payloadOp,         // payload IR root
        transformOp,       // transform.sequence or named_sequence
        {},                // extra handle bindings (usually empty)
        options,
        /*transformInserted=*/false);
```

### Running via mlir-opt

When the transform IR is embedded in the same file as the payload:

```bash
/usr/lib/llvm-22/bin/mlir-opt \
  --transform-interpreter \
  --test-transform-dialect-erase-schedule \
  input.mlir -o output.mlir
```

`--test-transform-dialect-erase-schedule` removes the transform.sequence from the output so it does not pollute the lowered IR.

### Separating transform and payload

The transform IR can live in a separate file:

```bash
/usr/lib/llvm-22/bin/mlir-opt \
  --transform-interpreter="transform-library-path=my_schedule.mlir" \
  payload.mlir -o output.mlir
```

This is the production usage pattern: the payload IR is the model, and the schedule is a separately maintained transformation script that can be tuned without recompiling the compiler.

---

## 150.8 Complete Example: Tiled Matrix Multiplication

```mlir
// payload.mlir
func.func @matmul(%A: tensor<256x256xf32>,
                  %B: tensor<256x256xf32>,
                  %C: tensor<256x256xf32>) -> tensor<256x256xf32> {
  %result = linalg.matmul ins(%A, %B : tensor<256x256xf32>, tensor<256x256xf32>)
                          outs(%C : tensor<256x256xf32>)
            -> tensor<256x256xf32>
  return %result : tensor<256x256xf32>
}

// schedule.mlir — the Transform IR
transform.named_sequence @__transform_main(
    %module: !transform.any_op {transform.readonly}) {

  // 1. Find the matmul op.
  %matmul = transform.structured.match ops{["linalg.matmul"]} in %module
      : (!transform.any_op) -> !transform.any_op

  // 2. Tile into 64x64x32 tiles.
  %tiled, %loops:3 = transform.structured.tile_using_for %matmul
      tile_sizes [64, 64, 32]
      : (!transform.any_op) -> (!transform.any_op,
                                 !transform.any_op,
                                 !transform.any_op,
                                 !transform.any_op)

  // 3. Pad to enable vectorization.
  %padded, %pad, %copy_back =
      transform.structured.pad %tiled {
        padding_values = [0.0 : f32, 0.0 : f32, 0.0 : f32],
        padding_dimensions = [0, 1, 2],
        pack_paddings = [1, 1, 0],
        copy_back_op = "linalg.copy"
      } : (!transform.any_op) -> (!transform.any_op,
                                   !transform.any_op,
                                   !transform.any_op)

  // 4. Vectorize the padded op.
  transform.structured.vectorize %padded : !transform.any_op

  // 5. Canonicalize to clean up.
  transform.apply_patterns to %module {
    transform.apply_patterns.canonicalization
  } : !transform.any_op

  transform.yield
}
```

Run:

```bash
/usr/lib/llvm-22/bin/mlir-opt \
  --transform-interpreter="transform-library-path=schedule.mlir" \
  --test-transform-dialect-erase-schedule \
  payload.mlir | \
/usr/lib/llvm-22/bin/mlir-opt \
  --one-shot-bufferize \
  --convert-linalg-to-loops \
  --lower-affine \
  --convert-scf-to-cf \
  --convert-vector-to-llvm \
  --convert-arith-to-llvm \
  --convert-func-to-llvm \
  --reconcile-unrealized-casts
```

---

## 150.9 Debugging and Verification

### Expensive checks

`--transform-interpreter-enable-expensive-checks` enables additional invariant checks:

- Handles are not used after the payload ops they reference are erased.
- Results are set exactly once per output handle.
- No handle outlives the transform sequence that created it.

These checks are disabled by default because they add O(n) overhead per transform op application.

### Printing transform state

```bash
/usr/lib/llvm-22/bin/mlir-opt \
  --transform-interpreter \
  --mlir-print-ir-after-all \
  input.mlir 2>&1 | less
```

This prints the payload IR state after every transform op application, making it easy to see the effect of each step in the schedule.

### Common errors

| Error | Cause |
|-------|-------|
| "handle is already consumed" | Using a handle after passing it to a consuming op |
| "no payload ops associated with handle" | `match` returned zero results; handle is empty |
| "silenceable failure not handled" | A silenceable failure occurred inside `failures(propagate)` sequence |
| "expected op to implement LinalgOp" | Passing a non-Linalg op to a `transform.structured.*` op |

---

## Chapter Summary

- The **Transform dialect** is a metaprogramming system: transform ops are MLIR operations that query and mutate a payload IR at compile time, enabling compiler strategies to be expressed as programs rather than hard-coded pass sequences.
- The **two-program architecture** separates payload IR (the program being compiled) from transform IR (the compilation strategy); the transform interpreter executes the strategy.
- `transform.sequence` is the top-level container; handle types (`!transform.any_op`, `!transform.any_value`, `!transform.param<T>`) connect transform ops to payload IR elements.
- `transform.structured.*` ops tile, fuse, vectorize, pad, and generalize `LinalgOp` instances; each returns handles to the newly created ops.
- `transform.named_sequence` and `transform.include` provide function-call-like composability; `transform.foreach_match` dispatches to different strategies based on op properties.
- New transform ops implement `TransformOpInterface`; `TransformState` tracks handle-to-payload mappings and enforces linearity.
- The interpreter entry point is `mlir::transform::applyTransforms`; `mlir-opt --transform-interpreter` runs it from the command line.
- `--transform-interpreter-enable-expensive-checks` enables runtime invariant checking at the cost of additional overhead.
