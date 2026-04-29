# Chapter 154 — HLO and StableHLO

*Part XXII — XLA and the OpenXLA Stack*

HLO (High Level Operations) is XLA's internal IR — a strongly typed, functional dataflow graph with static shapes. StableHLO is its stable, versioned MLIR counterpart: a dialect that freezes HLO semantics behind a compatibility guarantee so that serialized ML models remain loadable across compiler versions. Together, they form the lingua franca between ML frameworks and the XLA compiler stack. This chapter covers HLO's instruction set and text format in depth, then examines StableHLO's role as a cross-framework IR, its versioning mechanism, and the dialect conversion chain that bridges the two.

---

## 154.1 HLO Instruction Opcodes

Every `HloInstruction` carries an `HloOpcode` that determines its semantics. The opcode space spans roughly 130 distinct operations; the most important fall into five categories:

### 154.1.1 Element-wise Arithmetic

These operations apply a scalar function to every element of one or more tensors of the same shape:

| Opcode | Semantics |
|--------|-----------|
| `kAdd`, `kSubtract`, `kMultiply`, `kDivide` | Standard arithmetic |
| `kNegate`, `kAbs`, `kSign` | Unary arithmetic |
| `kExp`, `kLog`, `kSqrt`, `kRsqrt`, `kCbrt` | Transcendentals |
| `kSin`, `kCos`, `kTan`, `kAtan2` | Trigonometric |
| `kMaximum`, `kMinimum` | Element-wise max/min |
| `kPower` | Element-wise pow |
| `kClamp` | Three-operand clamp |
| `kCompare` | Comparison → PRED tensor |
| `kSelect` | PRED-conditional select |
| `kAnd`, `kOr`, `kXor`, `kNot`, `kShiftLeft`, etc. | Bitwise |

### 154.1.2 Linear Algebra

| Opcode | Semantics |
|--------|-----------|
| `kDot` | Batched dot product; `DotDimensionNumbers` specifies contracting and batch dims |
| `kConvolution` | N-D convolution; `ConvolutionDimensionNumbers` encodes layout |
| `kCholesky` | Cholesky factorization |
| `kTriangularSolve` | Triangular linear system solve |

`kDot` is the most performance-critical op. Its `DotDimensionNumbers` proto carries:
- `lhs_contracting_dims`, `rhs_contracting_dims`: which dimensions to contract
- `lhs_batch_dims`, `rhs_batch_dims`: batch dimensions

This generalizes matrix multiply to arbitrary tensor contraction.

### 154.1.3 Shape Manipulation

| Opcode | Semantics |
|--------|-----------|
| `kReshape` | Reinterpret shape without data movement (if layout allows) |
| `kTranspose` | Permute dimensions |
| `kSlice` | Static slice; start/limit/stride per dim |
| `kDynamicSlice` | Slice with dynamic start offsets |
| `kDynamicUpdateSlice` | In-place update of a slice |
| `kConcatenate` | Concatenate along a dimension |
| `kPad` | Pad with a fill value; supports interior padding |
| `kBroadcast` | Expand scalar or lower-rank tensor to a target shape |
| `kReverse` | Reverse along specified dimensions |
| `kGather` | Generalized gather; `GatherDimensionNumbers` specifies index mapping |
| `kScatter` | Generalized scatter-update |

### 154.1.4 Reductions and Scans

```
kReduce: applies a sub-computation to collapse one or more dimensions
kReduceWindow: sliding-window reduction (pooling)
kSelectAndScatter: backward pass for ReduceWindow
kScan: prefix scan (inclusive or exclusive)
```

`kReduce` takes an operand tensor, an initial value, and a sub-computation (a `HloComputation` computing `f(accumulator, element)`). It reduces over a set of specified dimensions.

### 154.1.5 Control Flow

| Opcode | Semantics |
|--------|-----------|
| `kWhile` | Loop: body and condition are sub-computations |
| `kConditional` | Multi-branch: branch computations selected by integer index |
| `kCall` | Static call to a named sub-computation |
| `kMap` | Apply a sub-computation element-wise |

`kWhile` is the only loop construct in HLO. Its condition computation takes the loop state tuple and returns a `PRED` scalar; its body computation transforms the state.

### 154.1.6 Collective Communication

These ops are essential for multi-device training (Chapter 158):

| Opcode | Semantics |
|--------|-----------|
| `kAllReduce` | Sum/max/product across replica groups |
| `kAllGather` | Gather shards across replicas along a dimension |
| `kReduceScatter` | All-reduce then scatter shards |
| `kAllToAll` | Pairwise exchange of tensor slices |
| `kCollectiveBroadcast` | Broadcast from one replica to all |

### 154.1.7 Custom Operations

`kCustomCall` provides an escape hatch for operations XLA doesn't natively understand:

```
%result = custom-call(%arg0, %arg1),
    custom_call_target="MyCudaKernel",
    api_version=API_VERSION_TYPED_FFI,
    backend_config="{\"kernel_config\": 42}"
```

XLA backends register custom call handlers by target name. The FFI version allows typed argument passing instead of raw pointers.

---

## 154.2 HLO Text Format

XLA's text format is essential for debugging. The full grammar is defined in [`xla/hlo/parser/hlo_parser.cc`](https://github.com/openxla/xla/blob/main/xla/hlo/parser/hlo_parser.cc). A representative module:

```
HloModule softmax_module, entry_computation_layout={(f32[16,1024]{1,0})->f32[16,1024]{1,0}}

ENTRY %main (input: f32[16,1024]) -> f32[16,1024] {
  %input = f32[16,1024]{1,0} parameter(0)
  
  // Compute max for numerical stability
  %reduce_max = f32[16]{0} reduce(%input, %init_max),
      dimensions={1},
      to_apply=%max_computation
  
  // Broadcast max back to original shape
  %bcast_max = f32[16,1024]{1,0} broadcast(%reduce_max), dimensions={0}
  
  // Subtract max (stable softmax)
  %shifted = f32[16,1024]{1,0} subtract(%input, %bcast_max)
  
  // Exponentiate
  %exp = f32[16,1024]{1,0} exponential(%shifted)
  
  // Sum of exp
  %reduce_sum = f32[16]{0} reduce(%exp, %init_zero),
      dimensions={1},
      to_apply=%add_computation
  
  // Broadcast sum
  %bcast_sum = f32[16,1024]{1,0} broadcast(%reduce_sum), dimensions={0}
  
  // Divide
  ROOT %result = f32[16,1024]{1,0} divide(%exp, %bcast_sum)
}

%max_computation (a: f32[], b: f32[]) -> f32[] {
  %a = f32[] parameter(0)
  %b = f32[] parameter(1)
  ROOT %max = f32[] maximum(%a, %b)
}

%add_computation (a: f32[], b: f32[]) -> f32[] {
  %a = f32[] parameter(0)
  %b = f32[] parameter(1)
  ROOT %add = f32[] add(%a, %b)
}
```

Key syntax elements:
- `%name = shape instruction_name(operands), attributes`
- `ROOT` marks the computation's return value
- `{1,0}` after the shape is the minor-to-major layout (column-major)
- `dimensions={1}` on reduce means collapse dimension 1

---

## 154.3 Shape System in Depth

HLO shapes are first-class objects that carry far more information than simple type + rank:

```protobuf
// From xla/shape.proto
message ShapeProto {
  PrimitiveType element_type = 2;
  repeated int64 dimensions = 3;
  repeated DimensionEntry layout = 7;     // minor-to-major ordering
  repeated bool is_dynamic_dimension = 9; // bounded dynamic dims
}
```

### 154.3.1 Dynamic Shapes

XLA supports **bounded dynamic shapes** — tensors where a dimension has a static upper bound but a dynamic runtime size. The `is_dynamic_dimension` field marks which dims are dynamic. XLA pads these dimensions to the bound and uses a "dynamic size" scalar to track the actual size.

This is distinct from MLIR's dynamic dimension model (the `?` in `tensor<?xf32>`): XLA's model keeps everything statically sized in memory, only tracking the logical size.

### 154.3.2 Tuple Shapes

`TUPLE` shapes aggregate multiple sub-shapes:

```
%result = (f32[4], s32[8]) tuple(%float_array, %int_array)
%first  = f32[4] get-tuple-element(%result), index=0
```

Tuple shapes are used for returning multiple values from computations and for structuring the loop state in `kWhile`.

---

## 154.4 StableHLO: A Versioned MLIR Dialect

StableHLO is a separately versioned MLIR dialect that provides a stable serialization target for ML models. It lives at [github.com/openxla/stablehlo](https://github.com/openxla/stablehlo) and is vendored into the OpenXLA repository.

### 154.4.1 Design Rationale

Before StableHLO, ML frameworks had no stable way to serialize model computations. MHLO (XLA's first MLIR dialect) changed its ops freely. TFLite's FlatBuffer format was TF-specific. ONNX required a separate ecosystem. StableHLO fills this gap with:

1. **Stability guarantee**: StableHLO 1.0+ promises 5-year forward/backward compatibility. A model serialized today can be loaded by XLA versions released up to 5 years later.
2. **MLIR-native**: full first-class MLIR op, including verifiers, canonicalizers, and type inference.
3. **Spec-driven**: every op has a formal specification at [github.com/openxla/stablehlo/tree/main/docs/spec.md](https://github.com/openxla/stablehlo/blob/main/docs/spec.md).

### 154.4.2 StableHLO Op Set

StableHLO's ops correspond closely to XLA HLO ops but are expressed as MLIR operations on `tensor` types:

```mlir
// MLIR module with StableHLO ops
func.func @softmax(%input: tensor<16x1024xf32>) -> tensor<16x1024xf32> {
  // Compute max for numerical stability
  %init_max = stablehlo.constant dense<-3.40282347E+38> : tensor<f32>
  %reduce_max = stablehlo.reduce(%input init: %init_max)
      applies stablehlo.maximum
      across dimensions = [1]
    : (tensor<16x1024xf32>, tensor<f32>) -> tensor<16xf32>

  // Broadcast back
  %bcast_max = stablehlo.broadcast_in_dim %reduce_max,
      dims = [0] : (tensor<16xf32>) -> tensor<16x1024xf32>

  // Subtract, exp, sum, divide
  %shifted = stablehlo.subtract %input, %bcast_max
      : (tensor<16x1024xf32>, tensor<16x1024xf32>) -> tensor<16x1024xf32>
  %exp = stablehlo.exponential %shifted : (tensor<16x1024xf32>) -> tensor<16x1024xf32>
  
  %init_zero = stablehlo.constant dense<0.0> : tensor<f32>
  %reduce_sum = stablehlo.reduce(%exp init: %init_zero)
      applies stablehlo.add
      across dimensions = [1]
    : (tensor<16x1024xf32>, tensor<f32>) -> tensor<16xf32>
  
  %bcast_sum = stablehlo.broadcast_in_dim %reduce_sum,
      dims = [0] : (tensor<16xf32>) -> tensor<16x1024xf32>
  
  %result = stablehlo.divide %exp, %bcast_sum
      : (tensor<16x1024xf32>, tensor<16x1024xf32>) -> tensor<16x1024xf32>
  return %result : tensor<16x1024xf32>
}
```

Key StableHLO ops and their HLO counterparts:

| StableHLO Op | HLO Opcode |
|--------------|------------|
| `stablehlo.add`, `stablehlo.multiply`, etc. | `kAdd`, `kMultiply` |
| `stablehlo.dot_general` | `kDot` (with DotDimensionNumbers) |
| `stablehlo.convolution` | `kConvolution` |
| `stablehlo.reduce` | `kReduce` |
| `stablehlo.scatter` | `kScatter` |
| `stablehlo.gather` | `kGather` |
| `stablehlo.while` | `kWhile` |
| `stablehlo.if` | `kConditional` |
| `stablehlo.custom_call` | `kCustomCall` |
| `stablehlo.all_reduce` | `kAllReduce` |
| `stablehlo.all_gather` | `kAllGather` |
| `stablehlo.sort` | `kSort` |
| `stablehlo.fft` | `kFft` |

### 154.4.3 dot_general: The Universal Contraction Op

`stablehlo.dot_general` is the most expressive linear algebra op. Unlike BLAS-style `gemm`, it handles arbitrary batch and contracting dimensions:

```mlir
// Batched matrix multiply: [batch, m, k] x [batch, k, n] -> [batch, m, n]
%result = stablehlo.dot_general %lhs, %rhs,
    batching_dims = [0] x [0],
    contracting_dims = [2] x [1]
    : (tensor<8x4x16xf32>, tensor<8x16x32xf32>) -> tensor<8x4x32xf32>
```

The `batching_dims` and `contracting_dims` pairs specify which dimensions are batch and which are contracted, generalizing to arbitrary tensor products.

---

## 154.5 VHLO: The Versioning Substrate

VHLO (Versioned HLO) is the underlying mechanism that enables StableHLO's compatibility guarantee. It lives in `stablehlo/dialect/VhloDialect.td`.

### 154.5.1 How Versioning Works

Each StableHLO op version is expressed as a distinct VHLO op with a version stamp:

```
vhlo.add_v1  ──► (introduced in StableHLO 0.9.0)
vhlo.add_v2  ──► (attribute change in StableHLO 1.2.0)
```

When XLA serializes a StableHLO module to bytecode, it:
1. Converts StableHLO → VHLO at the current version
2. Serializes VHLO to MLIR bytecode with the version tag embedded

When a future XLA version loads the bytecode:
1. Deserializes VHLO at the stored version
2. Applies upgrade passes (`VhloToVersionConverter`) to reach the current version
3. Converts VHLO → current StableHLO

This forward-compatibility model means older models always work, even after StableHLO ops evolve.

### 154.5.2 Bytecode Format

StableHLO bytecode is MLIR's standard bytecode format (`.mlirbc`) with a `#stablehlo<version "1.3.0">` attribute. The JAX export format encodes this bytecode in a `FunctionDef` proto for TF SavedModel compatibility.

---

## 154.6 CHLO: Complex HLO Decompositions

CHLO (Complex/Client HLO) is an MLIR dialect that sits above StableHLO. It contains "complex" ops — operations that are mathematically well-defined but decompose into multiple StableHLO primitives:

```mlir
// CHLO op: complex normalization
%result = chlo.broadcast_add %lhs, %rhs
    : (tensor<4xf32>, tensor<f32>) -> tensor<4xf32>

// Decomposes to:
%bcast = stablehlo.broadcast_in_dim %rhs, dims = []
    : (tensor<f32>) -> tensor<4xf32>
%result = stablehlo.add %lhs, %bcast
    : (tensor<4xf32>, tensor<4xf32>) -> tensor<4xf32>
```

CHLO also includes composite ops like `chlo.erf`, `chlo.lgamma`, `chlo.polygamma` that decompose to StableHLO sequences. The decomposition pass `ChloLegalizeToStablehlo` runs these transformations.

---

## 154.7 The Dialect Conversion Chain

The full conversion pipeline from framework to XLA backend:

```
JAX (Python)
    │ jax.jit tracing
    ▼
jaxpr (JAX's functional IR)
    │ jax._src.interpreters.mlir.jaxpr_to_stablehlo
    ▼
StableHLO MLIR module
    │ PJRT compile API (serialized bytecode)
    ▼
XLA: ConvertStableHloToHlo
    │ stablehlo/transforms/stablehlo_legalize_to_hlo.cc
    ▼
XLA HloModule (C++ objects)
    │ HloPassPipeline (algebraic simplification, fusion, etc.)
    ▼
Optimized HloModule
    │ Backend-specific lowering
    ▼
CPU: LLVM IR → native code
GPU: PTX → cubin
```

For CHLO-using frameworks (some TF operations), the chain has an additional step:

```
CHLO → ChloLegalizeToStablehlo → StableHLO → ...
```

### 154.7.1 StableHLO → HLO Conversion

The conversion from StableHLO ops to HLO instructions is handled by `LegalizeStableHloToHlo` in [`xla/mlir_hlo/stablehlo/transforms/stablehlo_legalize_to_hlo.cc`](https://github.com/openxla/xla/blob/main/xla/mlir_hlo/stablehlo/transforms/stablehlo_legalize_to_hlo.cc). It uses MLIR's dialect conversion framework with a `TypeConverter` that maps `tensor<4xf32>` to `xla::Shape` and pattern rewrites for each op:

```cpp
struct DotGeneralToHlo : public OpConversionPattern<stablehlo::DotGeneralOp> {
  LogicalResult matchAndRewrite(stablehlo::DotGeneralOp op,
                                 OpAdaptor adaptor,
                                 ConversionPatternRewriter& rewriter) const override {
    // Convert stablehlo::DotGeneralOp -> xla::HloInstruction::CreateDot
    xla::DotDimensionNumbers dims = Convert(op.getDotDimensionNumbers());
    ...
  }
};
```

### 154.7.2 HLO → StableHLO Conversion

The reverse path — converting HLO back to StableHLO for export — is `LegalizeHloToStableHlo` in `xla/mlir_hlo/tools/mlir_interpreter/mlir_interpreter.cc`. This enables round-tripping and is used by model export workflows.

---

## 154.8 Type System Correspondence

| HLO PrimitiveType | MLIR Type | StableHLO Type |
|-------------------|-----------|----------------|
| `F16` | `f16` | `tensor<...xf16>` |
| `BF16` | `bf16` | `tensor<...xbf16>` |
| `F32` | `f32` | `tensor<...xf32>` |
| `F64` | `f64` | `tensor<...xf64>` |
| `S8`, `S16`, `S32`, `S64` | `i8`, `i16`, `i32`, `i64` | `tensor<...xi8>` etc. |
| `U8`, `U16`, `U32`, `U64` | `ui8` etc. (or `i8` w/ signless) | `tensor<...xui8>` etc. |
| `PRED` | `i1` | `tensor<...xi1>` |
| `C64`, `C128` | `complex<f32>`, `complex<f64>` | `tensor<...xcomplex<f32>>` |
| `TUPLE` | struct types | `stablehlo.tuple` |
| `TOKEN` | `!stablehlo.token` | `!stablehlo.token` |

The `TOKEN` type is used for ordering operations that have side effects — similar to MLIR's `mlir::async::TokenType`.

---

## 154.9 Working with StableHLO Programmatically

### 154.9.1 C++ API

```cpp
#include "stablehlo/dialect/StablehloOps.h"
#include "mlir/IR/Builders.h"

mlir::MLIRContext ctx;
ctx.loadDialect<mlir::stablehlo::StablehloDialect>();

mlir::OpBuilder builder(&ctx);
auto loc = builder.getUnknownLoc();
auto f32 = builder.getF32Type();
auto tensor_type = mlir::RankedTensorType::get({4, 4}, f32);

// Create an add op
auto add = builder.create<mlir::stablehlo::AddOp>(loc, tensor_type, lhs, rhs);
```

### 154.9.2 Python API via JAX

JAX exposes StableHLO text format through `jax.export`:

```python
import jax
import jax.numpy as jnp

def my_model(x, w):
    return jnp.maximum(x @ w, 0)

# Get StableHLO text
x = jnp.ones((4, 8))
w = jnp.ones((8, 4))
lowered = jax.jit(my_model).lower(x, w)
stablehlo_text = lowered.as_text()
print(stablehlo_text)
# Outputs: func.func @main(%arg0: tensor<4x8xf32>, %arg1: tensor<8x4xf32>) -> ...
```

### 154.9.3 stablehlo-opt Tool

The `stablehlo-opt` binary (analogous to `mlir-opt`) processes StableHLO files:

```bash
# Canonicalize and dump
stablehlo-opt --canonicalize my_model.mlir

# Check versioned bytecode round-trip
stablehlo-opt --serialize='target=current' my_model.mlir | \
    stablehlo-opt --deserialize --check-version-compatibility
```

---

## 154.10 HLO Optimization Techniques

### 154.10.1 Algebraic Simplification Examples

The `AlgebraicSimplifier` handles hundreds of rewrites. Notable examples:

```
// Eliminate redundant transposes
transpose(transpose(x, {1,0}), {1,0}) → x

// Eliminate identity broadcasts
broadcast(x, {}) where x is already the target shape → x

// Constant folding
add(constant(3), constant(4)) → constant(7)

// Strength reduction
multiply(x, constant(2)) → add(x, x)   [for integers]

// Reshape of constant
reshape(constant([1,2,3,4]), [2,2]) → constant([[1,2],[3,4]])
```

### 154.10.2 Fusion for the Softmax Example

The softmax graph shown earlier fuses as follows:

1. `subtract + exp` → loop-fusion kernel (element-wise chain)
2. `divide` is fused with the exp kernel (if shapes match)
3. `reduce` and `broadcast` cannot be fused due to shape change — they form separate kernels

Post-fusion, softmax compiles to 3 kernels: one for reduce-max, one for exp+subtract+divide, one for reduce-sum. A further hand-tuned `kCustomCall` to cuDNN's softmax implementation replaces all three.

---

## Chapter Summary

- HLO's opcode space spans element-wise arithmetic, linear algebra, shape manipulation, reductions, control flow, and collective communication.
- `kDot`'s `DotDimensionNumbers` generalizes matrix multiply to arbitrary tensor contraction; `kReduce` with sub-computations handles all reduction variants.
- HLO shapes carry element type, dimensions, and minor-to-major layout; dynamic dimensions use bounded shapes.
- StableHLO is a versioned MLIR dialect providing a 5-year compatibility guarantee for serialized models; VHLO is the underlying versioning substrate.
- `stablehlo.dot_general` maps cleanly to `kDot`; the full op set covers 80+ ops with formal specifications.
- The conversion chain: JAX/TF → StableHLO → (CHLO decomposition) → HLO → optimized HLO → backend.
- `stablehlo-opt` is the tool for processing StableHLO files; JAX's `lowered.as_text()` exposes StableHLO for inspection.
- `AlgebraicSimplifier` and `InstructionFusion` are the two highest-impact optimization passes; together they eliminate redundant computation and memory traffic.
