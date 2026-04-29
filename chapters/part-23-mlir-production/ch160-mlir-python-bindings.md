# Chapter 160 — MLIR Python Bindings

*Part XXIII — MLIR in Production*

MLIR's Python bindings provide a first-class interface for constructing, transforming, and executing MLIR programs from Python. They are not a thin wrapper — they expose the full MLIR IR model (contexts, modules, operations, blocks, regions, values, types, attributes) and the pass manager through a Pythonic API generated from MLIR's C bindings via pybind11. Multiple ML frameworks (JAX, IREE, torch-mlir) use these bindings as their primary mechanism for generating MLIR programs. This chapter covers the complete Python API: IR construction, type and attribute systems, the pass manager interface, dialect-specific generated bindings, execution through the ORC JIT, and the NumPy bridge for feeding data into MLIR programs.

---

## 160.1 Architecture of the Python Bindings

The binding stack:

```
Python user code
    │ import mlir.ir
    ▼
MLIR Python package (mlir/python/)
    │ pybind11 bindings
    ▼
MLIR C API (mlir-c/IR.h, Diagnostics.h, etc.)
    │ extern "C" wrappers
    ▼
MLIR C++ core library (mlir/lib/IR/)
```

The C API is the stability boundary. Python bindings never call C++ MLIR functions directly — they always go through the C API. This allows the Python bindings to remain ABI-stable across MLIR changes.

The Python package lives in `mlir/python/` in the LLVM monorepo. When installed (via `pip install mlir-core` or built from source), it exposes:

```python
import mlir.ir          # Core IR: Context, Module, Operation, Block, etc.
import mlir.passmanager  # PassManager
import mlir.dialects.*   # Dialect-specific bindings (auto-generated)
import mlir.execution_engine  # ORC JIT runner
import mlir.runtime      # ctypes-based ABI helpers
```

---

## 160.2 Core IR Objects

### 160.2.1 Context

`Context` is the root MLIR object. All IR construction requires a context:

```python
from mlir.ir import Context

# Contexts are not thread-safe; use one per thread
with Context() as ctx:
    # Load dialects needed for your IR
    ctx.allow_unregistered_dialects = False
    
    # Access the context explicitly
    current_ctx = Context.current  # within a 'with' block
```

Dialects must be registered before their ops can appear in the IR:

```python
from mlir.dialects import arith, func, linalg
# Loading the module automatically registers dialects
```

### 160.2.2 Module

`Module` is the top-level container:

```python
from mlir.ir import Context, Module, Location

with Context() as ctx, Location.unknown():
    # Create an empty module
    module = Module.create()
    
    # Parse from text
    module = Module.parse("""
    func.func @add(%a: i32, %b: i32) -> i32 {
      %result = arith.addi %a, %b : i32
      return %result : i32
    }
    """)
    
    # Access the module body (a Block containing top-level ops)
    body = module.body  # type: Block
    
    # Print to string
    print(module)
```

### 160.2.3 Operations

Every MLIR operation is represented by `Operation` (or a dialect-specific subclass):

```python
from mlir.ir import Operation, InsertionPoint

# Iterate operations in the module body
for op in module.body.operations:
    print(f"op: {op.name}, results: {len(op.results)}")
    
# Access an operation's regions, blocks, operands, results, attributes
op = module.body.operations[0]
region = op.regions[0]    # type: Region
block = region.blocks[0]  # type: Block
args = block.arguments    # type: BlockArgumentList
```

Operations are created via dialect-specific factory functions or the generic API:

```python
# Generic operation creation
from mlir.ir import Operation, StringAttr, IntegerType, IntegerAttr

i32 = IntegerType.get_signless(32)
result = Operation.create(
    "arith.addi",
    results=[i32],
    operands=[lhs, rhs],
    attributes={},
    regions=0)
```

### 160.2.4 Types

MLIR types are immutable objects cached per-context:

```python
from mlir.ir import (IntegerType, FloatType, RankedTensorType,
                      MemRefType, FunctionType, IndexType)

# Integer types
i1 = IntegerType.get_signless(1)
i32 = IntegerType.get_signless(32)
si64 = IntegerType.get_signed(64)
ui8 = IntegerType.get_unsigned(8)

# Float types
f16 = FloatType.get_f16()
bf16 = FloatType.get_bf16()
f32 = FloatType.get_f32()
f64 = FloatType.get_f64()

# Tensor types
tensor_f32 = RankedTensorType.get([4, 8], f32)
dynamic_tensor = RankedTensorType.get([-1, 8], f32)  # -1 = dynamic

# MemRef types
memref_f32 = MemRefType.get([4, 8], f32)  # row-major default
memref_strided = MemRefType.get([4, 8], f32,
    layout=mlir.ir.StridedLayoutAttr.get(strides=[8, 1], offset=0))

# Function types
fn_type = FunctionType.get(domain=[i32, i32], range=[i32])

# Index type (for loop bounds, etc.)
idx = IndexType.get()
```

### 160.2.5 Attributes

```python
from mlir.ir import (IntegerAttr, FloatAttr, StringAttr,
                      ArrayAttr, DictionaryAttr, DenseElementsAttr)
import numpy as np

# Scalar attributes
i32_attr = IntegerAttr.get(i32, 42)
f32_attr = FloatAttr.get(f32, 3.14)
str_attr = StringAttr.get("hello")

# Dense constant: initialize from NumPy
data = np.array([[1.0, 2.0], [3.0, 4.0]], dtype=np.float32)
dense_attr = DenseElementsAttr.get(data)

# Array attribute
arr_attr = ArrayAttr.get([i32_attr, f32_attr])

# Dictionary attribute
dict_attr = DictionaryAttr.get({"key": str_attr, "value": i32_attr})
```

---

## 160.3 Building IR with InsertionPoint

The `InsertionPoint` context manager controls where new operations are inserted:

```python
from mlir.ir import Context, Module, Location, InsertionPoint
from mlir.dialects import func, arith

with Context() as ctx, Location.unknown():
    module = Module.create()
    
    with InsertionPoint(module.body):
        # Create function signature
        i32 = mlir.ir.IntegerType.get_signless(32)
        fn_type = mlir.ir.FunctionType.get([i32, i32], [i32])
        
        f = func.FuncOp("my_add", fn_type)
        
        # Enter function body
        entry_block = f.add_entry_block()
        with InsertionPoint(entry_block):
            a, b = entry_block.arguments
            
            # Create arith.addi op
            result = arith.AddIOp(a, b)
            
            # Create return op
            func.ReturnOp([result])
```

### 160.3.1 Location Tracking

MLIR uses `Location` objects to track source provenance:

```python
from mlir.ir import Location

# Unknown location (no source info)
loc = Location.unknown()

# File/line/column location  
loc = Location.file("my_model.py", line=42, col=8)

# Named location (for op identity)
loc = Location.name("my_gemm_kernel")

# Fused location (combine multiple sources)
loc = Location.fuse([loc1, loc2])

# Use location when creating ops:
with Location.file("my_dsl.py", 42, 0):
    op = arith.AddIOp(a, b)  # op gets this location automatically
```

---

## 160.4 Dialect-Specific Python Bindings

Dialect-specific bindings provide typed op constructors and attribute helpers. They are auto-generated from TableGen definitions using `mlir-python-bindings` tooling.

### 160.4.1 Arith Dialect

```python
from mlir.dialects import arith
from mlir.ir import IntegerType, IntegerAttr

i32 = IntegerType.get_signless(32)

# Constants
zero = arith.ConstantOp(i32, IntegerAttr.get(i32, 0))
one = arith.ConstantOp(i32, IntegerAttr.get(i32, 1))

# Arithmetic
sum_ = arith.AddIOp(zero, one)
product = arith.MulIOp(sum_, one)
quotient = arith.DivSIOp(product, one)  # signed divide

# Comparisons
cmp = arith.CmpIOp(arith.CmpIPredicate.slt, sum_, product)  # signed less-than

# Conversions
f32 = mlir.ir.FloatType.get_f32()
float_val = arith.SIToFPOp(f32, sum_)
truncated = arith.TruncIOp(IntegerType.get_signless(8), sum_)
```

### 160.4.2 Func Dialect

```python
from mlir.dialects import func

# Function with internal linkage
fn = func.FuncOp("internal_helper",
                  mlir.ir.FunctionType.get([i32], [i32]),
                  visibility="private")

# Call
result_types = [i32]
call = func.CallOp(result_types, mlir.ir.FlatSymbolRefAttr.get("internal_helper"),
                   [arg])

# Return (void)
func.ReturnOp([])
```

### 160.4.3 Linalg Dialect

```python
from mlir.dialects import linalg, tensor
from mlir.ir import RankedTensorType

f32 = mlir.ir.FloatType.get_f32()
mat_type = RankedTensorType.get([4, 8], f32)
vec_type = RankedTensorType.get([8], f32)
result_type = RankedTensorType.get([4], f32)

# Create empty result tensor
init = tensor.EmptyOp([4], f32)

# Batch matrix-vector: linalg.matvec
result = linalg.MatvecOp([result_type], [matrix, vector], [init])

# Generic op (custom iteration domain)
indexing_maps = [
    mlir.ir.AffineMap.get_permutation([0, 1]),  # (i,j) -> mat[i,j]
    mlir.ir.AffineMap.get_identity(1),           # (j) -> vec[j]
    mlir.ir.AffineMap.get_identity(1),           # (i) -> out[i]
]
result = linalg.GenericOp(
    result_types=[result_type],
    inputs=[matrix, vector],
    outputs=[init],
    indexing_maps=indexing_maps,
    iterator_types=["parallel", "reduction"])
# Fill in region body...
```

---

## 160.5 The Pass Manager

The Python `PassManager` wraps MLIR's C++ `PassManager`:

```python
from mlir.passmanager import PassManager

# Create a pass manager for the module level
pm = PassManager.parse("builtin.module(canonicalize,cse)")

# Run on a module
pm.run(module.operation)

# More complex pipeline
pm = PassManager.parse("""
builtin.module(
  func.func(
    canonicalize,
    cse,
    loop-invariant-code-motion
  ),
  inline
)
""")
pm.run(module.operation)
```

### 160.5.1 Pass Configuration

```python
# Passes with options
pm = PassManager.parse(
    "builtin.module("
    "  func.func("
    "    affine-loop-tile{tile-size=32 loop-depth=2},"
    "    affine-vectorize{virtual-vector-size=8 8}"
    "  )"
    ")")

# Enable pass statistics
pm.enable_verifier(True)
pm.enable_ir_printing()  # print IR before/after each pass

pm.run(module.operation)
```

### 160.5.2 Custom Python Passes

MLIR 22 supports writing passes in Python via the `PassRegistration` API:

```python
import mlir._mlir_libs._mlir.passmanager as _pm

@mlir.ir.register_operation("my_dialect.my_op")
class MyOp(mlir.ir.OpView):
    # ... op wrapper class ...
    pass

# Python pass that counts ops
class CountOpsPass:
    argument = "--count-ops"
    description = "Count operations by name"
    
    def run(self, op: mlir.ir.Operation):
        counts = {}
        def walk(o):
            counts[o.name] = counts.get(o.name, 0) + 1
            for region in o.regions:
                for block in region.blocks:
                    for inner_op in block.operations:
                        walk(inner_op)
        walk(op)
        print(counts)
```

---

## 160.6 MLIR Execution Engine

The `mlir.execution_engine.ExecutionEngine` wraps MLIR's ORC-JIT-based execution:

```python
from mlir.execution_engine import ExecutionEngine
from mlir.ir import Module
import ctypes

# Build and lower a module to LLVM dialect
module = Module.parse("""
llvm.func @add(%a: i32, %b: i32) -> i32 {
  %result = llvm.add %a, %b : i32
  llvm.return %result : i32
}
""")

# JIT-compile
engine = ExecutionEngine(module, opt_level=2, shared_libs=[])

# Invoke the function
result = ctypes.c_int32(0)
a = ctypes.c_int32(3)
b = ctypes.c_int32(4)

# Arguments are passed as ctypes pointer-to-value; result is first
engine.invoke("add", ctypes.byref(result), ctypes.byref(a), ctypes.byref(b))
print(result.value)  # 7
```

### 160.6.1 Calling Conventions

The ORC JIT uses MLIR's lowered-function ABI: all arguments and the result are passed as `void*` pointers. For structured types (memrefs), use the `mlir.runtime` ABI helpers:

```python
from mlir.runtime import get_ranked_memref_descriptor, make_nd_memref_descriptor
import numpy as np

# Create a memref descriptor from a NumPy array
array = np.ones((4, 8), dtype=np.float32)
descriptor = get_ranked_memref_descriptor(array)

# The descriptor has: allocated_ptr, aligned_ptr, offset, sizes[], strides[]
engine.invoke("my_kernel", ctypes.byref(descriptor))
```

---

## 160.7 NumPy/MLIR Bridge

### 160.7.1 numpy → DenseElementsAttr

```python
import numpy as np
from mlir.ir import DenseElementsAttr, RankedTensorType, FloatType

data = np.array([[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]], dtype=np.float32)
f32 = FloatType.get_f32()
tensor_type = RankedTensorType.get(list(data.shape), f32)
dense_attr = DenseElementsAttr.get(data)

# Create a constant op from the dense attribute
from mlir.dialects import arith
const = arith.ConstantOp(tensor_type, dense_attr)
```

### 160.7.2 memref ↔ numpy round-trip

```python
from mlir.runtime import (get_ranked_memref_descriptor,
                           ranked_memref_to_numpy,
                           unranked_memref_to_numpy)

# numpy → ranked memref descriptor (zero-copy view)
arr = np.array([1.0, 2.0, 3.0, 4.0], dtype=np.float32)
memref_desc = get_ranked_memref_descriptor(arr)

# Execute MLIR kernel that writes to the memref
engine.invoke("fill_kernel", ctypes.byref(memref_desc))

# Read back: descriptor → numpy (zero-copy if layout is contiguous)
result = ranked_memref_to_numpy(memref_desc, np.float32, (4,))
```

---

## 160.8 Generating Dialect Bindings

Out-of-tree dialects can generate Python bindings from ODS:

```cmake
# In your dialect's CMakeLists.txt
mlir_tablegen(MatAlgOpsExt.cpp.inc -gen-python-op-bindings
              -bind-dialect=matalg)
declare_mlir_python_extension(MatAlgPythonExtension
    MODULE_NAME _matalg
    ADD_TO_PARENT MLIRPythonSources.Dialects
    SOURCES MatAlgExtension.cpp
    EMBED_CAPI_LINK_LIBS MLIRCAPIMatAlg)
```

The generated bindings expose typed Python classes:

```python
from mlir.dialects import matalg

# Typed constructor with Python-level type checking
gemm = matalg.GemmOp(result_type=matrix_type, a=a_val, b=b_val,
                     alpha=1.0, beta=0.0)
```

---

## 160.9 Real-World Usage Patterns

### 160.9.1 JAX's Use of MLIR Python Bindings

JAX uses the MLIR Python bindings as its primary IR construction mechanism. The `jax._src.interpreters.mlir` module builds StableHLO IR from jaxpr using the Python bindings:

```python
# Simplified from jax/_src/interpreters/mlir.py
def jaxpr_to_stablehlo(jaxpr, args):
    ctx = mlir.ir.Context()
    module = mlir.ir.Module.create()
    
    with mlir.ir.InsertionPoint(module.body):
        fn = func.FuncOp("main", infer_fn_type(jaxpr))
        
        with mlir.ir.InsertionPoint(fn.add_entry_block()):
            # Lower each jaxpr equation to StableHLO ops
            for eqn in jaxpr.eqns:
                lower_equation(eqn, fn.entry_block.arguments)
            
            func.ReturnOp(outvals)
    
    return module
```

### 160.9.2 IREE's Use

IREE uses Python bindings for its `iree.compiler.ir` module, which builds `flow` and `stream` dialect IR programmatically.

### 160.9.3 Interactive Development

The Python bindings are invaluable for interactive exploration:

```python
# IPython / Jupyter
import mlir.ir as ir
from mlir.dialects import arith, func

ctx = ir.Context()
ctx.allow_unregistered_dialects = True

with ctx, ir.Location.unknown():
    m = ir.Module.create()
    # ... build IR interactively ...
    print(m)  # display the MLIR text

# Use the pass manager interactively
from mlir.passmanager import PassManager
pm = PassManager.parse("builtin.module(canonicalize)")
pm.run(m.operation)
print(m)
```

---

## 160.10 Performance Considerations

### 160.10.1 Batching IR Construction

Python overhead per operation is significant (~1-10 µs). For large models (millions of ops), batch IR construction using string parsing:

```python
# Slow: Python loop over 10,000 ops
for i in range(10000):
    arith.AddIOp(vals[i], vals[i+1])

# Fast: construct in text, parse once
mlir_text = "\n".join(
    f"%v{i} = arith.addi %v{i-1}, %v{i-1} : i32"
    for i in range(1, 10001)
)
module = Module.parse(f"func.func @f(%v0: i32) {{ {mlir_text}\n return }}")
```

### 160.10.2 Memory Management

MLIR contexts own all IR objects. When a context is destroyed, all associated IR is freed. Python garbage collection does not drive IR memory — hold a reference to the context:

```python
# Bad: context might be GC'd before module
def make_module():
    ctx = Context()
    return Module.parse("...", context=ctx)  # ctx captured

# Good: explicit ownership
ctx = Context()
module = Module.parse("...", context=ctx)
# Both ctx and module must stay alive
```

---

## Chapter Summary

- MLIR Python bindings are pybind11 wrappers over the MLIR C API; they provide a stable, language-idiomatic interface to the full MLIR IR model.
- Core objects: `Context` (root, not thread-safe), `Module` (top-level), `Operation` (generic op), `Block`, `Region`, `Value`, `Type`, `Attribute` — all follow the MLIR C++ ownership model.
- Types and attributes are immutable, context-cached objects; `RankedTensorType.get([4, 8], f32)` creates or retrieves the cached type.
- `InsertionPoint` context managers control where `create()` calls insert operations; `Location` tracks source provenance.
- Dialect-specific bindings (`mlir.dialects.arith`, `linalg`, `func`, etc.) provide typed op constructors and are auto-generated from ODS.
- `PassManager.parse(pipeline_string)` builds a pass pipeline from text; `pm.run(module.operation)` executes it.
- `ExecutionEngine` provides ORC-JIT execution of LLVM-dialect modules; arguments use ctypes descriptors; `mlir.runtime` helpers handle ranked memref ABIs.
- NumPy integration: `DenseElementsAttr.get(ndarray)` lifts data into MLIR; `get_ranked_memref_descriptor` passes arrays to JIT-compiled code.
- For large-scale IR construction (millions of ops), string parsing is faster than Python-level construction loops.


---

@copyright jreuben11
