# Chapter 175 — Language Bindings

*Part XXV — Operations, Bindings, and Contribution*

LLVM is a C++ library, but the languages that use it as a compilation target or JIT backend span the full breadth of the programming language ecosystem: Python, Rust, Go, Julia, Swift, Java, .NET. Each of these requires a bridge between its own memory management model, object system, and calling convention and LLVM's C++ API. LLVM's stable C API (`llvm-c/`) provides the foundation for most of these bridges; language-specific wrappers add type safety, idiomatic style, and convenience. This chapter examines each major binding layer: the C API, llvmlite (Python), inkwell (Rust), the Go bindings, MLIR's Python bindings, and Julia's direct LLVM integration. The emphasis is on practical, working code that demonstrates IR construction, compilation, and execution across these languages.

---

## 175.1 The LLVM C API

### Why a Stable C API

LLVM's C++ API is not ABI-stable: adding a virtual function to a class, changing a method signature, or reordering enum values can break binary compatibility. Language bindings built directly against the C++ API must be rebuilt for every LLVM version. The C API (`include/llvm-c/Core.h`) solves this by providing an opaque-handle interface with a stable ABI:

- All LLVM types are represented as opaque pointers (`LLVMModuleRef`, `LLVMValueRef`, etc.).
- All functions use C linkage and C types.
- The API evolves carefully: new functions are added; existing functions are deprecated before removal.

The C API is not complete — it covers the core IR building, JIT execution, and target selection, but not every LLVM analysis or optimization. For missing functionality, C extensions can be written in C++ and exposed via the C API pattern.

### Core Types

```c
#include "llvm-c/Core.h"
#include "llvm-c/ExecutionEngine.h"
#include "llvm-c/Target.h"

typedef struct LLVMOpaqueContext     *LLVMContextRef;
typedef struct LLVMOpaqueModule      *LLVMModuleRef;
typedef struct LLVMOpaqueType        *LLVMTypeRef;
typedef struct LLVMOpaqueValue       *LLVMValueRef;
typedef struct LLVMOpaqueBasicBlock  *LLVMBasicBlockRef;
typedef struct LLVMOpaqueBuilder     *LLVMBuilderRef;
```

Each opaque pointer wraps the corresponding LLVM C++ type: `LLVMModuleRef` wraps `llvm::Module*`, `LLVMValueRef` wraps `llvm::Value*`, etc. Type information is preserved at the C++ level but hidden from C callers.

### Building a Module in C

A complete example: a function that returns the sum of two 32-bit integers.

```c
#include <llvm-c/Core.h>
#include <llvm-c/ExecutionEngine.h>
#include <llvm-c/Target.h>
#include <llvm-c/Analysis.h>
#include <stdio.h>
#include <stdlib.h>

int main() {
    // Initialize target (for JIT)
    LLVMInitializeNativeTarget();
    LLVMInitializeNativeAsmPrinter();
    LLVMLinkInMCJIT();

    // Create context, module, builder
    LLVMContextRef ctx = LLVMContextCreate();
    LLVMModuleRef  mod = LLVMModuleCreateWithNameInContext("my_module", ctx);
    LLVMBuilderRef b   = LLVMCreateBuilderInContext(ctx);

    // Define function type: i32 (i32, i32)
    LLVMTypeRef i32      = LLVMInt32TypeInContext(ctx);
    LLVMTypeRef params[] = {i32, i32};
    LLVMTypeRef fn_ty    = LLVMFunctionType(i32, params, 2, /*vararg=*/0);

    // Create function
    LLVMValueRef fn = LLVMAddFunction(mod, "add", fn_ty);
    LLVMSetFunctionCallConv(fn, LLVMCCallConv);

    // Create entry basic block
    LLVMBasicBlockRef entry = LLVMAppendBasicBlockInContext(ctx, fn, "entry");
    LLVMPositionBuilderAtEnd(b, entry);

    // Build: ret (arg0 + arg1)
    LLVMValueRef a = LLVMGetParam(fn, 0);
    LLVMValueRef b_ = LLVMGetParam(fn, 1);
    LLVMValueRef sum = LLVMBuildAdd(b, a, b_, "sum");
    LLVMBuildRet(b, sum);

    // Verify the module
    char *err = NULL;
    if (LLVMVerifyModule(mod, LLVMAbortProcessAction, &err)) {
        fprintf(stderr, "Verification error: %s\n", err);
        LLVMDisposeMessage(err);
        return 1;
    }

    // Print IR
    char *ir = LLVMPrintModuleToString(mod);
    printf("%s\n", ir);
    LLVMDisposeMessage(ir);

    // JIT execute
    LLVMExecutionEngineRef engine;
    if (LLVMCreateExecutionEngineForModule(&engine, mod, &err)) {
        fprintf(stderr, "JIT error: %s\n", err);
        LLVMDisposeMessage(err);
        return 1;
    }

    // Call the function
    LLVMValueRef args[] = {
        LLVMConstInt(i32, 21, /*sign-extend=*/0),
        LLVMConstInt(i32, 21, 0)
    };
    LLVMValueRef result = LLVMRunFunction(engine, fn, 2, args);
    printf("add(21, 21) = %lld\n", LLVMGenericValueToInt(result, 0));

    // Cleanup
    LLVMDisposeGenericValue(result);
    LLVMDisposeExecutionEngine(engine);  // also disposes mod
    LLVMDisposeBuilder(b);
    LLVMContextDispose(ctx);
    return 0;
}
```

Compile:
```bash
clang -o c_api_demo c_api_demo.c \
  $(llvm-config --cflags --ldflags --libs core executionengine mcjit native) \
  -lstdc++
```

### Key C API Functions

| Category | C API Function | C++ Equivalent |
|----------|---------------|----------------|
| Context | `LLVMContextCreate()` | `new LLVMContext()` |
| Module | `LLVMModuleCreateWithNameInContext()` | `new Module(name, ctx)` |
| Types | `LLVMInt32TypeInContext()` | `Type::getInt32Ty(ctx)` |
| Values | `LLVMConstInt(ty, val, sext)` | `ConstantInt::get(ty, val)` |
| Build | `LLVMBuildAdd(b, lhs, rhs, name)` | `Builder.CreateAdd(lhs, rhs, name)` |
| Analysis | `LLVMVerifyModule(mod, action, err)` | `verifyModule(mod, &err)` |

The complete C API reference: [`include/llvm-c/`](https://github.com/llvm/llvm-project/tree/llvmorg-22.1.0/llvm/include/llvm-c)

---

## 175.2 Python Bindings (llvmlite)

### What llvmlite Is

`llvmlite` is a Python binding for LLVM maintained by the Numba project (Anaconda). It uses Python's `ctypes`/`cffi` to call LLVM's C API plus a small set of C extension functions. It does **not** use `pybind11` or Cython — the entire binding is through C ABI calls, which makes it version-agnostic within a range.

```bash
pip install llvmlite
# Or from conda:
conda install llvmlite
```

llvmlite has two layers:
- **`llvmlite.ir`**: a pure-Python IR builder (no LLVM calls; generates LLVM IR text).
- **`llvmlite.binding`**: calls into LLVM for JIT execution, target selection, and optimization.

### Building IR with llvmlite.ir

```python
from llvmlite import ir

# Create module
module = ir.Module(name="my_module")

# Define function type: i32(i32, i32)
i32 = ir.IntType(32)
fn_type = ir.FunctionType(i32, [i32, i32])

# Create function
fn = ir.Function(module, fn_type, name="add")
a, b = fn.args
a.name = "a"
b.name = "b"

# Create basic block and builder
block = fn.append_basic_block(name="entry")
builder = ir.IRBuilder(block)

# Build instructions
total = builder.add(a, b, name="sum")
builder.ret(total)

print(str(module))
```

Output:
```llvm
; ModuleID = "my_module"
target triple = "unknown-unknown-unknown"
target datalayout = ""

define i32 @"add"(i32 %"a", i32 %"b")
{
entry:
  %"sum" = add i32 %"a", %"b"
  ret i32 %"sum"
}
```

### JIT Execution with llvmlite.binding

```python
import llvmlite.binding as llvm
import ctypes

# Initialize LLVM
llvm.initialize()
llvm.initialize_native_target()
llvm.initialize_native_asmprinter()

# Parse the IR (from the ir module or from a string)
ir_str = str(module)  # from the builder above
llvm_module = llvm.parse_assembly(ir_str)
llvm_module.verify()

# Create target machine
target = llvm.Target.from_default_triple()
target_machine = target.create_target_machine()

# Create JIT execution engine
engine = llvm.create_mcjit_compiler(llvm_module, target_machine)
engine.finalize_object()
engine.run_static_constructors()

# Get the function pointer
fn_ptr = engine.get_function_address("add")

# Create a ctypes function prototype and call it
add_fn = ctypes.CFUNCTYPE(ctypes.c_int32, ctypes.c_int32, ctypes.c_int32)(fn_ptr)
result = add_fn(21, 21)
print(f"add(21, 21) = {result}")  # Output: add(21, 21) = 42
```

### Optimization

```python
# Optimize the module
from llvmlite.binding import PassManagerBuilder, ModulePassManager

pmb = PassManagerBuilder()
pmb.opt_level = 3
pmb.loop_vectorize = True

pm = ModulePassManager()
pmb.populate(pm)
pm.run(llvm_module)
print(str(llvm_module))  # Optimized IR
```

### Users of llvmlite

- **Numba**: JIT compiler for Python that compiles NumPy-using functions to LLVM IR. Achieves near-C performance for numerical loops.
- **Pyston**: a JIT-optimized CPython variant (now merged back to CPython as JIT hints).
- **mygrad**: a numpy-compatible autodiff library that uses llvmlite for fused kernel JIT.

---

## 175.3 Rust Bindings (inkwell)

### inkwell: Safe Rust Bindings

`inkwell` wraps LLVM's C API with Rust's ownership and lifetime system, providing memory-safe, ergonomic Rust bindings.

```toml
# Cargo.toml
[dependencies]
inkwell = { version = "0.4", features = ["llvm17-0"] }
```

```rust
use inkwell::context::Context;
use inkwell::OptimizationLevel;

fn main() {
    let context = Context::create();
    let module = context.create_module("my_module");
    let builder = context.create_builder();

    // Define function: i32 add(i32 a, i32 b)
    let i32_type = context.i32_type();
    let fn_type = i32_type.fn_type(&[i32_type.into(), i32_type.into()], false);
    let function = module.add_function("add", fn_type, None);

    // Create basic block
    let entry = context.append_basic_block(function, "entry");
    builder.position_at_end(entry);

    // Get parameters
    let a = function.get_nth_param(0).unwrap().into_int_value();
    let b = function.get_nth_param(1).unwrap().into_int_value();

    // Build: return a + b
    let sum = builder.build_int_add(a, b, "sum").unwrap();
    builder.build_return(Some(&sum)).unwrap();

    // Verify
    assert!(module.verify().is_ok());
    println!("{}", module.print_to_string().to_string());

    // JIT execute
    let engine = module.create_jit_execution_engine(OptimizationLevel::None).unwrap();
    let add_fn = unsafe {
        engine.get_function::<unsafe extern "C" fn(i32, i32) -> i32>("add").unwrap()
    };
    
    let result = unsafe { add_fn.call(21, 21) };
    println!("add(21, 21) = {}", result);  // 42
}
```

### Safety Model

inkwell uses Rust's `PhantomData` lifetimes to ensure that objects derived from a `Context` cannot outlive the context:

```rust
// Context owns all LLVM objects; they carry a 'ctx lifetime
pub struct Module<'ctx> { ... }
pub struct Builder<'ctx> { ... }
pub struct IntValue<'ctx> { ... }

// This would not compile: 'ctx would outlive the context
// let module;
// {
//     let context = Context::create();
//     module = context.create_module("test");  // ERROR: context dropped before module
// }
```

This eliminates a class of use-after-free bugs common in C API usage.

### Users of inkwell

- **Neon**: a GPU shader compiler written in Rust that uses inkwell for code generation.
- **Koto**: a scripting language with an inkwell JIT backend.
- Various research compilers in academia using Rust as the implementation language.

---

## 175.4 Go Bindings (tinygo)

### go-llvm: cgo Wrapping the C API

The official LLVM Go bindings (`llvm.org/llvm/bindings/go/llvm`) were abandoned after LLVM 15. Active Go development targets `tinygo`, which uses `go-llvm` (a cgo-based wrapper):

```go
package main

// #cgo LDFLAGS: -lLLVM
// #include <llvm-c/Core.h>
import "C"
import "unsafe"

func main() {
    ctx := C.LLVMContextCreate()
    mod := C.LLVMModuleCreateWithNameInContext(C.CString("my_module"), ctx)
    builder := C.LLVMCreateBuilderInContext(ctx)

    i32 := C.LLVMInt32TypeInContext(ctx)
    params := []C.LLVMTypeRef{i32, i32}
    fnTy := C.LLVMFunctionType(i32, &params[0], 2, 0)
    fn := C.LLVMAddFunction(mod, C.CString("add"), fnTy)

    entry := C.LLVMAppendBasicBlockInContext(ctx, fn, C.CString("entry"))
    C.LLVMPositionBuilderAtEnd(builder, entry)

    a := C.LLVMGetParam(fn, 0)
    b := C.LLVMGetParam(fn, 1)
    sum := C.LLVMBuildAdd(builder, a, b, C.CString("sum"))
    C.LLVMBuildRet(builder, sum)

    // Print IR
    irStr := C.LLVMPrintModuleToString(mod)
    defer C.LLVMDisposeMessage(irStr)
    println(C.GoString(irStr))

    _ = unsafe.Pointer(nil)  // quiet unused import
    C.LLVMDisposeBuilder(builder)
    C.LLVMDisposeModule(mod)
    C.LLVMContextDispose(ctx)
}
```

tinygo wraps this pattern in a higher-level API and uses it to compile Go programs to embedded and WebAssembly targets. The LLVM backend in tinygo is the reason tinygo can target ARM Cortex-M, RISC-V, and WebAssembly — none of which are supported by the standard Go gc compiler.

---

## 175.5 MLIR Python Bindings

### Installation

```bash
# From PyPI (for released versions)
pip install mlir-python-bindings

# Or build from source (for LLVM 22):
cmake ... -DMLIR_ENABLE_BINDINGS_PYTHON=ON \
           -DMLIR_BINDINGS_PYTHON_LOCK_VERSION=OFF
ninja MLIRPythonModules
# The bindings appear in: build/python_packages/mlir_core/
```

### Core API: mlir.ir

```python
import mlir.ir as ir
import mlir.dialects.func as func_dialect
import mlir.dialects.arith as arith

# Create context and module
with ir.Context() as ctx, ir.Location.unknown():
    # Register dialects
    func_dialect.register_dialect(ctx)
    arith.register_dialect(ctx)

    # Create a module
    module = ir.Module.create()
    
    with ir.InsertionPoint(module.body):
        # Create a function
        i32 = ir.IntegerType.get_signless(32)
        fn_type = ir.FunctionType.get([i32, i32], [i32])
        
        @func_dialect.FuncOp.from_py_func(i32, i32)
        def add(a, b):
            result = arith.AddIOp(a, b)
            return result.result
    
    print(module)
```

### Working with Dialects

```python
import mlir.dialects.arith as arith
import mlir.dialects.memref as memref
import mlir.dialects.linalg as linalg

# Build a linalg.generic operation
with ir.Context() as ctx, ir.Location.unknown():
    # Register required dialects
    ctx.allow_unregistered_dialects = True
    
    module = ir.Module.create()
    f32 = ir.F32Type.get()
    
    with ir.InsertionPoint(module.body):
        # Create function with memref arguments
        memref_type = ir.MemRefType.get([4, 4], f32)
        fn_type = ir.FunctionType.get([memref_type, memref_type], [])
        
        fn = func_dialect.FuncOp("elementwise_add", fn_type)
        entry = fn.add_entry_block()
        
        with ir.InsertionPoint(entry):
            # Access arguments
            a = entry.arguments[0]
            b = entry.arguments[1]
            # ... build linalg operations ...
            func_dialect.ReturnOp([])
```

### Pass Manager in Python

```python
from mlir.passmanager import PassManager

# Run optimization passes
with ir.Context() as ctx:
    module = ir.Module.parse("""
        func.func @test(%x: i32) -> i32 {
            %c1 = arith.constant 1 : i32
            %r = arith.addi %x, %c1 : i32
            return %r : i32
        }
    """, ctx)
    
    # Build and run pass pipeline
    pm = PassManager.parse("builtin.module(func.func(canonicalize))", ctx)
    pm.run(module.operation)
    print(module)
```

### Users of MLIR Python Bindings

**IREE**: [openxla/iree](https://github.com/openxla/iree) exposes its compiler pipeline through MLIR Python bindings, allowing Python-based tooling to drive compilation to GPU/CPU targets.

**JAX + jaxlib**: JAX's JIT compiles Python functions through XLA, which uses StableHLO (an MLIR dialect). The `jaxlib` package includes MLIR Python bindings for dialect manipulation.

**torch-mlir**: PyTorch's [torch-mlir](https://github.com/llvm/torch-mlir) project exposes a Python API for lowering PyTorch models through torch dialect → linalg → LLVM. Users interact through:

```python
import torch_mlir
import torch

class MyModel(torch.nn.Module):
    def forward(self, x):
        return torch.relu(x) + x

model = MyModel()
example_input = torch.ones(3, 4)
mlir_module = torch_mlir.compile(model, example_input, 
                                   output_type="linalg-on-tensors")
print(mlir_module)
```

---

## 175.6 Julia

### Julia's Relationship with LLVM

Julia does not use the LLVM C API or any public binding layer. Instead, it embeds LLVM as a C++ library directly into the Julia runtime (`libjulia`). Julia maintains its own fork of LLVM (`julia/llvm-project` on GitHub) that includes patches for Julia-specific needs.

### JIT Architecture

Julia's JIT uses LLVM's ORC JIT v2:

```
Julia code
  │ Julia frontend (parser, macro expansion)
  ↓
Julia AST
  │ Type inference (Julia's type system)
  ↓
Typed IR (Julia IR)
  │ Code generation (codegen.cpp)
  ↓
LLVM IR
  │ LLVM optimization pipeline
  ↓
Machine code (via ORC JIT)
```

Julia's `@code_llvm` and `@code_native` macros expose the intermediate stages:

```julia
julia> function add(a::Int32, b::Int32)
           a + b
       end

julia> @code_llvm add(Int32(1), Int32(2))
;  @ REPL[1]:2 within `add`
define i32 @julia_add_...(...) {
top:
  %3 = add i32 %1, %0
  ret i32 %3
}

julia> @code_native add(Int32(1), Int32(2))
        .section __TEXT,__text
_julia_add_...:
        leal    (%rdi,%rsi), %eax
        retq
```

This introspective capability — the ability to see the LLVM IR for any Julia function — makes Julia uniquely transparent for performance analysis.

### GPU Compilation in Julia

Julia's GPU ecosystem compiles Julia code directly to GPU targets via LLVM:

- **CUDA.jl** (NVIDIA): uses the NVPTX backend; dispatches GPU kernels written in Julia.
- **AMDGPU.jl** (AMD): uses the AMDGPU backend.
- **oneAPI.jl** (Intel): uses the SPIR-V backend.
- **Metal.jl** (Apple): uses Metal shaders compiled via LLVM.

The mechanism: Julia's type inference runs the GPU function, generating typed Julia IR; the GPU codegen layer then generates LLVM IR targeting the GPU backend; LLVM compiles to PTX/AMDGCN/SPIR-V and the runtime loads the result.

```julia
using CUDA

function add_gpu(a, b, c)
    i = (blockIdx().x - 1) * blockDim().x + threadIdx().x
    c[i] = a[i] + b[i]
    return nothing
end

a = CUDA.ones(1024)
b = CUDA.ones(1024)
c = CUDA.zeros(1024)
@cuda threads=256 blocks=4 add_gpu(a, b, c)
```

This `@cuda` macro captures the function, runs Julia's type inference with GPU-context type information, generates LLVM IR targeting NVPTX, compiles it via LLVM, and loads the resulting PTX into the CUDA runtime — all at runtime.

### ORC JIT Transition

Julia is transitioning from the custom JIT stack it built around MCJIT to ORC v2 (Chapter 108). The primary motivation: ORC v2 supports concurrent compilation (multiple threads compiling different methods simultaneously), which is essential for Julia's multi-threaded workloads where many methods may need JIT compilation simultaneously.

---

## Chapter Summary

- The **LLVM C API** (`llvm-c/Core.h`) provides a stable, ABI-stable interface using opaque handles; it is the foundation for all non-C++ language bindings.
- **llvmlite** (Python) has two layers: `llvmlite.ir` for IR construction (pure Python, generates text IR) and `llvmlite.binding` for JIT execution (calls LLVM via ctypes). Used by Numba for high-performance scientific computing.
- **inkwell** (Rust) wraps the C API with Rust lifetimes; the `'ctx` lifetime ensures LLVM objects cannot outlive their `Context`. Used by language compilers implemented in Rust.
- **tinygo** uses cgo-based LLVM C API bindings to compile Go to embedded and WebAssembly targets; the official `go/llvm` bindings were abandoned after LLVM 15.
- **MLIR Python bindings** provide `mlir.ir`, `mlir.dialects.*`, `mlir.passmanager`, and `mlir.execution_engine`; used by torch-mlir, IREE, and JAX's linalg-on-tensors pipeline.
- **Julia** embeds LLVM as a C++ library (not via bindings); uses its own LLVM fork for stability; exposes `@code_llvm` and `@code_native` for transparent inspection of generated code.
- Julia's GPU stack (CUDA.jl, AMDGPU.jl) uses LLVM to compile Julia functions directly to GPU targets at runtime; transitioning to ORC v2 for concurrent JIT compilation.
