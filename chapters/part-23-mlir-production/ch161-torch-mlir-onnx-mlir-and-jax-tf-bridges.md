# Chapter 161 — torch-mlir, ONNX-MLIR, and JAX/TF Bridges

*Part XXIII — MLIR in Production*

The practical value of MLIR for machine learning emerges most clearly in the bridges that connect popular ML frameworks to MLIR-based compilation infrastructure. torch-mlir converts PyTorch programs to MLIR; ONNX-MLIR compiles ONNX model graphs to native binaries; JAX's MLIR lowering path is the canonical example of a framework built on MLIR from the ground up; and TensorFlow's bridge converts SavedModel graphs to MLIR for analysis and optimization. TOSA (Tensor Operator Set Architecture) serves as a portable operator-level intermediary. This chapter examines each bridge in depth, covering their architectures, lowering strategies, and practical usage.

---

## 161.1 torch-mlir: PyTorch → MLIR

torch-mlir ([github.com/llvm/torch-mlir](https://github.com/llvm/torch-mlir)) is the official MLIR-based compilation path for PyTorch models. It converts `torch.export`-traced models to MLIR dialects suitable for downstream compilers (IREE, XLA, ONNX-MLIR, or direct LLVM).

### 161.1.1 Architecture

```
PyTorch nn.Module
    │ torch.export.export() or torch.jit.trace()
    ▼
ExportedProgram (FX graph with ATen ops)
    │ torch_mlir.compile()
    ▼
Torch dialect IR (torch.aten.* ops)
    │
    ├──► Lowering to StableHLO (for XLA/IREE)
    ├──► Lowering to Linalg-on-Tensors (for CPU/IREE)
    └──► Lowering to TMTensor (for sparse/special ops)
    │
    ▼
Target dialect (stablehlo / linalg / tosa)
```

### 161.1.2 The Torch Dialect

The Torch dialect provides MLIR ops corresponding to PyTorch's ATen (A Tensor Library) ops:

```mlir
// Torch dialect IR for a simple linear layer
func.func @forward(%input: !torch.vtensor<[batch,784],f32>,
                   %weight: !torch.vtensor<[128,784],f32>,
                   %bias: !torch.vtensor<[128],f32>)
    -> !torch.vtensor<[batch,128],f32> {
  
  // aten::linear(input, weight, bias)
  %result = torch.aten.linear %input, %weight, %bias
      : !torch.vtensor<[batch,784],f32>,
        !torch.vtensor<[128,784],f32>,
        !torch.vtensor<[128],f32>
      -> !torch.vtensor<[batch,128],f32>
  
  return %result : !torch.vtensor<[batch,128],f32>
}
```

The `!torch.vtensor` type carries optional static shape information (`[batch,784]` where `batch` is symbolic) and element type. The value tensor (`vtensor`) is immutable; there is also `!torch.tensor` for mutable tensors.

The Torch dialect op set mirrors PyTorch's ATen IR directly — `torch.aten.mm`, `torch.aten.add.Tensor`, `torch.aten.relu`, `torch.aten.softmax.int`, etc.

### 161.1.3 Python API

```python
import torch
import torch_mlir
import torch.nn as nn

class SimpleModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.linear = nn.Linear(784, 128)
        self.relu = nn.ReLU()
    
    def forward(self, x):
        return self.relu(self.linear(x))

model = SimpleModel().eval()
example_input = torch.randn(4, 784)

# Compile to StableHLO (for XLA/IREE)
mlir_module = torch_mlir.compile(
    model,
    example_input,
    output_type=torch_mlir.OutputType.STABLEHLO)

# Compile to Linalg-on-Tensors (for CPU optimization)
mlir_module = torch_mlir.compile(
    model,
    example_input,
    output_type=torch_mlir.OutputType.LINALG_ON_TENSORS)

# Compile to TOSA (for embedded deployment)
mlir_module = torch_mlir.compile(
    model,
    example_input,
    output_type=torch_mlir.OutputType.TOSA)

# Print the resulting MLIR
print(mlir_module.operation.get_asm(large_elements_limit=4))
```

### 161.1.4 Lowering Torch to StableHLO

The `TorchToStableHlo` conversion pass suite converts Torch ops to StableHLO:

```mlir
// Input: Torch dialect
%result = torch.aten.mm %a, %b
    : !torch.vtensor<[4,8],f32>, !torch.vtensor<[8,16],f32>
    -> !torch.vtensor<[4,16],f32>

// Output: StableHLO
%result = stablehlo.dot_general %a, %b,
    batching_dims = [] x [],
    contracting_dims = [1] x [0]
    : (tensor<4x8xf32>, tensor<8x16xf32>) -> tensor<4x16xf32>
```

Complex ops like `aten.conv2d` require multi-step lowering through intermediate patterns.

### 161.1.5 Symbolic Shape Handling

PyTorch models often have dynamic batch dimensions. torch-mlir preserves symbolic shapes through the compilation:

```python
# Export with dynamic batch dimension
batch = torch.export.Dim("batch", min=1, max=128)
exported = torch.export.export(
    model,
    (example_input,),
    dynamic_shapes={"x": {0: batch}})

mlir_module = torch_mlir.compile(
    exported,
    output_type=torch_mlir.OutputType.STABLEHLO,
    enable_ir_printing=False)
# Result has tensor<?x784xf32> where ? is dynamic
```

---

## 161.2 ONNX-MLIR: ONNX → Native Code

ONNX-MLIR ([github.com/onnx/onnx-mlir](https://github.com/onnx/onnx-mlir)) compiles ONNX model files to native shared libraries or executables. It targets production inference deployment where ONNX Runtime's interpreter overhead is unacceptable.

### 161.2.1 Architecture

```
ONNX model (.onnx file)
    │ onnx-mlir importer
    ▼
onnx dialect IR (onnx.Add, onnx.Conv, onnx.Gemm, ...)
    │ ONNXToKrnl pass
    ▼
krnl dialect (loop tiling, vectorization annotations)
    │ KrnlToAffine pass
    ▼
affine + memref + vector dialect
    │ standard lowering pipeline
    ▼
LLVM IR → native binary (.so or executable)
```

### 161.2.2 The ONNX Dialect

The `onnx` dialect in ONNX-MLIR mirrors the ONNX operator set precisely:

```mlir
// ONNX dialect IR
func.func @main_graph(%data: tensor<1x3x224x224xf32>,
                       %weight: tensor<64x3x7x7xf32>,
                       %bias: tensor<64xf32>) -> tensor<1x64x112x112xf32> {
  %result = onnx.Conv(%data, %weight, %bias) {
    auto_pad = "NOTSET",
    dilations = [1, 1],
    group = 1 : i64,
    kernel_shape = [7, 7],
    pads = [3, 3, 3, 3],
    strides = [2, 2]
  } : (tensor<1x3x224x224xf32>, tensor<64x3x7x7xf32>, tensor<64xf32>)
      -> tensor<1x64x112x112xf32>
  return %result : tensor<1x64x112x112xf32>
}
```

### 161.2.3 The Krnl Dialect

The `krnl` dialect is ONNX-MLIR's tiling and optimization IR — a layer between ONNX semantics and Affine loops:

```mlir
// Krnl dialect: explicit loop nest with tiling annotations
krnl.iterate (%i, %j) with (%ii in 0 to 64, %jj in 0 to 112) {
  krnl.define_loops 2
  %tile_i, %tile_j = krnl.block %i, %j by 8, 16
  // Vectorization hint: inner dimension is vectorizable
  krnl.vectorize %tile_j, 8
  // ... loop body with memory accesses ...
}
```

Krnl ops are then lowered to Affine loops, which LLVM's vectorizer handles.

### 161.2.4 Compilation and Usage

```bash
# Compile ONNX to shared library
onnx-mlir --EmitLib resnet50.onnx -o resnet50.so

# Compile to object file (for static linking)
onnx-mlir --EmitObj resnet50.onnx -o resnet50.o

# Run inference from Python
from PyRuntime import OMExecutionSession
import numpy as np

session = OMExecutionSession("resnet50.so")
input_data = np.random.randn(1, 3, 224, 224).astype(np.float32)
output = session.run([input_data])
```

### 161.2.5 ONNX-MLIR Optimization Passes

```bash
# Enable specific optimizations
onnx-mlir \
    --onnx-op-transform-iteration-count=3 \  # algebraic simplification iterations
    --enable-gemm-optimization \             # GEMM layout optimization
    --mcpu=znver4 \                          # target AMD Zen4
    resnet50.onnx -o resnet50.so
```

---

## 161.3 JAX/MLIR Bridge

JAX is the most tightly integrated ML framework with MLIR. Since JAX 0.4, the entire compilation path from Python to hardware goes through MLIR's StableHLO dialect.

### 161.3.1 JAX's MLIR Lowering Stack

```python
import jax
import jax.numpy as jnp

# 1. Python function
def transformer_layer(params, x):
    return jax.nn.gelu(x @ params['w'] + params['b'])

# 2. Trace with abstract shapes (jaxpr)
f_lowered = jax.jit(transformer_layer).lower(params, x)

# 3. Access the StableHLO MLIR module
stablehlo_text = f_lowered.as_text()
stablehlo_bytecode = f_lowered.compiler_ir(dialect='stablehlo')
hlo_text = f_lowered.compiler_ir(dialect='hlo')

# 4. Inspect the MLIR
print(stablehlo_text)
```

### 161.3.2 jax.export: Model Serialization

`jax.export` (stable API since JAX 0.4.7) provides a versioned model serialization mechanism based on StableHLO:

```python
import jax
import jax.numpy as jnp

def my_model(x, w):
    return jnp.maximum(x @ w, 0)

# Export with abstract shapes
x_abstract = jax.ShapeDtypeStruct((4, 8), jnp.float32)
w_abstract = jax.ShapeDtypeStruct((8, 4), jnp.float32)

exported = jax.export.export(my_model)(x_abstract, w_abstract)

# Serialize to bytes (StableHLO bytecode)
serialized = exported.serialize()

# Load and execute on any device
reloaded = jax.export.deserialize(serialized)
result = reloaded.call(x, w)  # executes on current platform
```

The serialized format is a StableHLO MLIR bytecode wrapped in a Flatbuffer. The 5-year compatibility guarantee of StableHLO means serialized models remain loadable across JAX versions.

### 161.3.3 Accessing MLIR from JAX for Custom Lowerings

JAX allows registering custom MLIR lowering rules for primitive operations:

```python
import jax.interpreters.mlir as mlir
from mlir.dialects import stablehlo

def my_op_lowering(ctx, x, y, *, config):
    """Custom MLIR lowering for my_op primitive."""
    aval_out = ctx.avals_out[0]
    result_type = mlir.aval_to_ir_type(aval_out)
    
    # Build StableHLO ops
    return stablehlo.AddOp(x, y).results

mlir.register_lowering(my_op_primitive, my_op_lowering)
```

---

## 161.4 TensorFlow → MLIR

TensorFlow's MLIR bridge converts TF graphs to MLIR for analysis, optimization, and compilation.

### 161.4.1 TF SavedModel to MLIR

```python
import tensorflow as tf

# Load a SavedModel
model = tf.saved_model.load("/path/to/saved_model")

# Convert to MLIR (experimental API)
mlir_module = tf.mlir.experimental.convert_saved_model(
    "/path/to/saved_model",
    exported_names=["serving_default"])

print(mlir_module)
# Outputs tf_executor dialect IR
```

### 161.4.2 TF Graph to MLIR

For individual functions:

```python
import tensorflow as tf

@tf.function
def my_fn(x, w):
    return tf.nn.relu(tf.matmul(x, w))

# Get concrete function
cf = my_fn.get_concrete_function(
    tf.TensorSpec([4, 8], tf.float32),
    tf.TensorSpec([8, 4], tf.float32))

# Convert to MLIR
mlir_text = tf.mlir.experimental.convert_function(cf)
print(mlir_text)
```

### 161.4.3 TF Dialect

TF ops appear in MLIR as `tf.*` operations:

```mlir
func.func @serving_default(%x: tensor<4x8xf32>, %w: tensor<8x4xf32>)
    -> tensor<4x4xf32> {
  %matmul = "tf.MatMul"(%x, %w) {
    transpose_a = false, transpose_b = false
  } : (tensor<4x8xf32>, tensor<8x4xf32>) -> tensor<4x4xf32>
  %relu = "tf.Relu"(%matmul) : (tensor<4x4xf32>) -> tensor<4x4xf32>
  return %relu : tensor<4x4xf32>
}
```

The lowering path: `tf.*` → `tf_executor.*` → Functional TF → `mhlo.*` / `stablehlo.*` → XLA.

---

## 161.5 TOSA: Tensor Operator Set Architecture

TOSA ([Tensor Operator Set Architecture](https://www.mlplatform.org/TOSA/)) is a portable ML operator set designed as a middle ground between high-level framework ops and low-level Linalg primitives.

### 161.5.1 TOSA Design Goals

- **Portability**: a small set (~60 ops) sufficient for inference workloads
- **Precision spec**: each op has a formal integer/float precision specification
- **Hardware-neutral**: no target-specific assumptions; backend must implement the precision spec
- **Quantization-aware**: native support for int8/int16 quantized inference

### 161.5.2 TOSA Op Set Sample

```mlir
// TOSA IR for a simple ConvReLU
func.func @conv_relu(%input: tensor<1x28x28x1xf32>,
                     %weight: tensor<32x3x3x1xf32>,
                     %bias: tensor<32xf32>) -> tensor<1x28x28x32xf32> {
  
  %conv = tosa.conv2d %input, %weight, %bias {
    dilation = array<i64: 1, 1>,
    pad = array<i64: 1, 1, 1, 1>,
    stride = array<i64: 1, 1>
  } : (tensor<1x28x28x1xf32>, tensor<32x3x3x1xf32>, tensor<32xf32>)
      -> tensor<1x28x28x32xf32>
  
  // TOSA clamp implements ReLU
  %relu = tosa.clamp %conv {
    max_fp = 3.4028235E+38 : f32,
    min_fp = 0.0 : f32
  } : (tensor<1x28x28x32xf32>) -> tensor<1x28x28x32xf32>
  
  return %relu : tensor<1x28x28x32xf32>
}
```

Key TOSA ops:

| Category | Ops |
|----------|-----|
| Tensor ops | `tosa.matmul`, `tosa.conv2d`, `tosa.depthwise_conv2d` |
| Elementwise | `tosa.add`, `tosa.mul`, `tosa.relu`, `tosa.clamp`, `tosa.exp` |
| Reduction | `tosa.reduce_sum`, `tosa.reduce_max`, `tosa.reduce_mean` |
| Shape | `tosa.reshape`, `tosa.transpose`, `tosa.slice`, `tosa.pad` |
| Data type | `tosa.cast`, `tosa.rescale` (for quantization) |
| Control | `tosa.cond_if`, `tosa.while_loop` |

### 161.5.3 Lowering TOSA

TOSA ops lower to Linalg or directly to loops:

```bash
mlir-opt --tosa-to-linalg \
         --tosa-to-tensor \
         --tosa-to-arith \
         --convert-linalg-to-loops \
         my_model.mlir
```

The MLIR in-tree `tosa-to-linalg` pass converts each TOSA op to the corresponding Linalg named op or `linalg.generic`.

### 161.5.4 Framework TOSA Outputs

- torch-mlir: `OutputType.TOSA`
- XCoreTools (XMOS): uses TOSA as the primary deployment format
- Ethos-U (ARM): Arm's ML accelerator compiler consumes TOSA
- IREE: accepts TOSA as an input dialect

---

## 161.6 Comparing the Framework Bridges

| Bridge | Input | Output Dialects | Use Case |
|--------|-------|-----------------|----------|
| torch-mlir | `torch.export` / `torch.jit.trace` | StableHLO, Linalg, TOSA | PyTorch → IREE/XLA |
| ONNX-MLIR | `.onnx` file | LLVM IR (native binary) | Inference deployment |
| JAX export | JAX Python function | StableHLO bytecode | Portable model serialization |
| TF SavedModel | TF SavedModel directory | `tf.*` → StableHLO | TF → XLA |
| TOSA | (output of above) | Linalg → LLVM | Portable inference |

---

## 161.7 End-to-End Example: PyTorch → IREE

Combining torch-mlir and IREE for a complete deployment pipeline:

```python
import torch
import torch_mlir
import iree.compiler as ireec
import iree.runtime as ireert
import numpy as np

# 1. PyTorch model
class MLP(torch.nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = torch.nn.Linear(784, 256)
        self.fc2 = torch.nn.Linear(256, 10)
    
    def forward(self, x):
        return self.fc2(torch.relu(self.fc1(x)))

model = MLP().eval()
dummy_input = torch.randn(32, 784)

# 2. Export to torch.export
exported = torch.export.export(model, (dummy_input,))

# 3. Convert to StableHLO via torch-mlir
stablehlo_module = torch_mlir.compile(
    exported,
    output_type=torch_mlir.OutputType.STABLEHLO,
    enable_ir_printing=False)

# 4. Compile to IREE CUDA target
flatbuffer = ireec.compile_str(
    str(stablehlo_module),
    input_type="stablehlo",
    target_backends=["cuda"])

# 5. Run inference with IREE runtime
config = ireert.Config("cuda")
vm_module = ireert.VmModule.copy_buffer(
    config.vm_instance, flatbuffer)
ctx = ireert.SystemContext(config=config)
ctx.add_vm_module(vm_module)

# 6. Execute
runner = ctx.modules.module["forward"]
input_data = np.random.randn(32, 784).astype(np.float32)
result = runner(input_data)
print(result.to_host().shape)  # (32, 10)
```

---

## Chapter Summary

- torch-mlir converts PyTorch's `torch.export` programs to the Torch dialect (ATen ops), then lowers to StableHLO, Linalg, or TOSA; `torch_mlir.compile()` is the primary API.
- ONNX-MLIR compiles `.onnx` files to native shared libraries via: `onnx` dialect → `krnl` dialect → Affine → LLVM IR; `PyRuntime.OMExecutionSession` loads the result.
- JAX's compilation path is StableHLO-native; `jax.export.export()` serializes to versioned StableHLO bytecode with a 5-year compatibility guarantee; `jax.jit(f).lower().as_text()` exposes the MLIR.
- TF's bridge (`tf.mlir.experimental.convert_saved_model`) converts TF graphs to `tf.*` dialect, then lowers through `mhlo`/StableHLO to XLA.
- TOSA provides a portable ~60-op inference operator set; it serves as an intermediate for embedded and hardware-accelerated targets; ARM Ethos-U and IREE consume TOSA natively.
- The canonical end-to-end path for new ML deployment: PyTorch → torch-mlir → StableHLO → IREE → device binary.
