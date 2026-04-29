# Chapter 146 — Async, OpenMP, OpenACC, DLTI, and EmitC

*Part XX — In-Tree Dialects*

The final chapter of Part XX collects five dialects that each address a distinct and important concern: `async` provides a structured representation of asynchronous concurrent execution; `omp` models OpenMP 5.x directives for portable CPU parallelism; `acc` models OpenACC 3.x for GPU offloading in scientific computing; `dlti` provides machine description data that drives correct data layout and ABI decisions; and `emitc` enables MLIR to target C directly, bypassing LLVM for embedded and microcontroller targets. While these dialects are more specialized than the ones in earlier chapters, each plays an indispensable role in a complete MLIR ecosystem.

---

## 146.1 The `async` Dialect

### 146.1.1 Design and Model

The `async` dialect ([`mlir/include/mlir/Dialect/Async/IR/AsyncOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Async/IR/AsyncOps.td)) provides a structured model for asynchronous execution. Its design is similar to C++20 coroutines or structured concurrency libraries: computations can be dispatched asynchronously, returning a token (a future-like handle), and the caller can await these tokens to synchronize.

The primary types:

| Type | Description |
|---|---|
| `!async.token` | A handle representing completion of an async computation |
| `!async.value<T>` | A handle representing a value that will be produced asynchronously |
| `!async.group` | A collection of tokens (for fan-out/fan-in patterns) |

### 146.1.2 Core Operations

```mlir
// Execute a block of code asynchronously
// Returns a token (completion handle) and async values for results
%token, %result = async.execute [%dep_token] (
    %captured_val as %inner_val : !async.value<f32>
) -> !async.value<f32> {
  // This body runs asynchronously
  %val = async.await %captured_val : !async.value<f32>
  %doubled = arith.mulf %val, %c2_f32 : f32
  async.yield %doubled : f32
}

// Await a token (block until the async computation completes)
async.await %token : !async.token

// Await a value (block and extract the value)
%final_val = async.await %result : !async.value<f32>
```

`async.execute` takes:
- **Dependencies** `[%dep_token, ...]`: tokens that must complete before this execute body starts
- **Captures** `(%val as %inner : !async.value<T>)`: async values captured into the body
- **Body**: the computation to run asynchronously
- **Returns**: a `!async.token` + zero or more `!async.value<T>` results

This models structured concurrency: you can express fork/join patterns without explicit threading:

```mlir
// Fan-out: launch N independent computations
%tok0, %r0 = async.execute { ... ; async.yield %val0 : f32 } -> !async.value<f32>
%tok1, %r1 = async.execute { ... ; async.yield %val1 : f32 } -> !async.value<f32>
%tok2, %r2 = async.execute { ... ; async.yield %val2 : f32 } -> !async.value<f32>

// Fan-in: wait for all
%group = async.create_group 3 : !async.group
async.add_to_group %tok0, %group : !async.token
async.add_to_group %tok1, %group : !async.token
async.add_to_group %tok2, %group : !async.token
async.await_all %group

// Extract results
%v0 = async.await %r0 : !async.value<f32>
%v1 = async.await %r1 : !async.value<f32>
%v2 = async.await %r2 : !async.value<f32>
```

### 146.1.3 Lowering Pipeline

```bash
# Step 1: Convert async.execute to async runtime ops (coroutine-based)
mlir-opt --async-to-async-runtime input.mlir

# Step 2: Add reference counting for async values
mlir-opt --async-runtime-ref-counting input.mlir

# Step 3: Optimize ref-count operations
mlir-opt --async-runtime-ref-counting-opt input.mlir

# Step 4: Convert async runtime ops to LLVM calls
mlir-opt --convert-async-to-llvm input.mlir
```

`--async-to-async-runtime` converts `async.execute` bodies into coroutines using LLVM's coroutine infrastructure. Each `async.await` becomes a coroutine suspension point. The runtime manages the coroutine state.

`--convert-async-to-llvm` maps the async runtime ops to calls into the MLIR async runtime library (`mlir_async_runtime_*` functions), which provides a thread pool and reference-counted async values.

### 146.1.4 Runtime Infrastructure

The async runtime ([`mlir/lib/ExecutionEngine/AsyncRuntime.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/ExecutionEngine/AsyncRuntime.cpp)) provides:

```cpp
// C functions called by lowered async code:
void mlirAsyncRuntimeAddToken(MlirAsyncToken *token);   // mark token ready
void mlirAsyncRuntimeDropRef(void *ptr, int64_t count); // reference counting
MlirAsyncGroup mlirAsyncRuntimeCreateGroup(int64_t size); // fan-in group
void mlirAsyncRuntimeAddTokenToGroup(MlirAsyncToken *token, MlirAsyncGroup *group);
void mlirAsyncRuntimeAwaitAllInGroupAndExecute(MlirAsyncGroup *group, void *fn, void *storage);
```

The runtime uses `std::thread` or a configurable thread pool. Reference counting manages the lifetime of async values, similar to `std::shared_ptr`.

---

## 146.2 The `omp` Dialect

### 146.2.1 OpenMP in MLIR

The `omp` dialect ([`mlir/include/mlir/Dialect/OpenMP/OpenMPOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/OpenMP/OpenMPOps.td)) represents OpenMP 5.1 directives as MLIR operations. It is the primary representation used by Flang (Chapter 127) and is also used by ClangIR for C/C++ OpenMP programs. Each OpenMP directive maps to one or more MLIR operations with region bodies.

### 146.2.2 Parallel and Work Distribution

```mlir
// omp.parallel: fork-join model
omp.parallel num_threads(%n : i32) {
  // This block executes on all threads in the team
  // Implicit barrier at end of parallel region
  omp.terminator
}

// omp.wsloop: worksharing for loop
omp.wsloop {
  omp.loop_nest (%i) : index = (%c0) to (%n) step (%c1) {
    // body: iterations are distributed across threads
    omp.yield
  }
}

// omp.simd: SIMD loop vectorization hint
omp.simd {
  omp.loop_nest (%i) : index = (%c0) to (%n) step (%c1) {
    %elem = memref.load %arr[%i] : memref<?xf32>
    %result = arith.mulf %elem, %scale : f32
    memref.store %result, %out[%i] : memref<?xf32>
    omp.yield
  }
}

// Combined parallel + wsloop
omp.parallel {
  omp.wsloop {
    omp.loop_nest (%i) : index = (%c0) to (%n) step (%c1) {
      omp.yield
    }
  }
  omp.terminator
}
```

### 146.2.3 Synchronization and Data Sharing

```mlir
// Critical section
omp.critical(@section_name) {
  // Only one thread executes at a time
  omp.terminator
}

// Single: only one thread executes the body
omp.single {
  // body
  omp.terminator
}

// Barrier: all threads synchronize
omp.barrier

// Flush: memory fence
omp.flush(%variable : memref<f32>)

// Sections: divide work into sections
omp.sections {
  omp.section {
    // Section 1 work
    omp.terminator
  }
  omp.section {
    // Section 2 work
    omp.terminator
  }
  omp.terminator
}
```

### 146.2.4 Reduction

```mlir
// Declare a custom reduction operator
omp.declare_reduction @add_reduction : f32
    init { %zero = arith.constant 0.0 : f32; omp.yield %zero : f32 }
    combiner { ^bb0(%lhs : f32, %rhs : f32): %s = arith.addf %lhs, %rhs : f32; omp.yield %s : f32 }

// Use reduction in a parallel loop
omp.parallel reduction(@add_reduction -> %partial : f32) {
  omp.wsloop {
    omp.loop_nest (%i) : index = (%c0) to (%n) step (%c1) {
      %elem = memref.load %arr[%i] : memref<?xf32>
      omp.reduction %elem, %partial : f32
      omp.yield
    }
  }
  omp.terminator
}
```

### 146.2.5 Task Model

```mlir
// Task: unit of work that can be deferred
omp.task if(%should_run : i1) final(%is_final : i1) {
  // Task body
  omp.terminator
}

// Taskgroup: group of tasks (implicit barrier on exit)
omp.taskgroup {
  omp.task { ... ; omp.terminator }
  omp.task { ... ; omp.terminator }
  omp.terminator
}

// Taskwait: wait for child tasks
omp.taskwait
```

### 146.2.6 Target Offload

```mlir
// omp.target: offload to accelerator
omp.target device(%device_id : i32)
    map_entries(%host_arr -> %device_arr : memref<?xf32> -> memref<?xf32>)
{
  // Body executes on target device
  omp.terminator
}

// omp.target_data: manage data movement
omp.target_data map_entries(%arr -> %dev_arr : memref<?xf32> -> memref<?xf32>) {
  omp.terminator
}

// omp.teams: create teams of thread groups on target
omp.teams num_teams_lower(%min_teams : i32) num_teams_upper(%max_teams : i32) {
  omp.distribute {
    omp.loop_nest (%i) : index = (%c0) to (%n) step (%c1) {
      omp.yield
    }
  }
  omp.terminator
}
```

### 146.2.7 Lowering

```bash
# Convert OpenMP dialect to LLVM runtime calls (__kmpc_*)
mlir-opt --convert-openmp-to-llvm input.mlir

# Then:
mlir-opt --convert-func-to-llvm --convert-arith-to-llvm input.mlir
```

`--convert-openmp-to-llvm` maps:
- `omp.parallel` → `__kmpc_fork_call(ident, 1, outlined_fn, args...)`
- `omp.wsloop` → `__kmpc_for_static_init_*` + `__kmpc_for_static_fini`
- `omp.critical` → `__kmpc_critical` / `__kmpc_end_critical`
- `omp.barrier` → `__kmpc_barrier`
- `omp.task` → `__kmpc_task_alloc` + `__kmpc_omp_task`
- `omp.reduction` → `__kmpc_reduce` + `__kmpc_end_reduce`

The outlined functions (one per `omp.parallel` body) become separate LLVM functions with the signature expected by the OpenMP runtime.

---

## 146.3 The `acc` Dialect

### 146.3.1 OpenACC in MLIR

The `acc` dialect ([`mlir/include/mlir/Dialect/OpenACC/OpenACC.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/OpenACC/OpenACC.td)) represents OpenACC 3.x directives. OpenACC is a directive-based GPU programming model popular in scientific computing, particularly in Fortran-heavy HPC codes. The `acc` dialect is primarily used by Flang.

### 146.3.2 Compute Constructs

```mlir
// acc.parallel: parallel execution on accelerator
acc.parallel dataOperands(%A_in : memref<?xf32>, %B_out : memref<?xf32>) {
  // Body runs on GPU
  acc.yield
}

// acc.kernels: compiler-parallelizable region
acc.kernels dataOperands(%input : memref<?xf32>) {
  acc.yield
}

// acc.serial: sequential execution on accelerator (for debugging)
acc.serial dataOperands(%data : memref<?xf32>) {
  acc.yield
}

// acc.loop: parallel loop inside a compute construct
acc.parallel {
  acc.loop gang vector {
    // Distributed across gangs and vectors
    acc.yield
  }
  acc.yield
}
```

### 146.3.3 Data Management

```mlir
// acc.copyin: copy data from host to device
%dev_A = acc.copyin varPtr(%A : memref<?xf32>) -> memref<?xf32>

// acc.copyout: copy data from device to host
%dev_B = acc.copyout varPtr(%B : memref<?xf32>) -> memref<?xf32>

// acc.create: allocate on device (no copy)
%dev_C = acc.create varPtr(%C : memref<?xf32>) -> memref<?xf32>

// acc.delete: free device allocation
acc.delete accPtr(%dev_C : memref<?xf32>)

// acc.present: assert data is already on device
%dev_D = acc.present varPtr(%D : memref<?xf32>) -> memref<?xf32>

// acc.data: structured data region
acc.data
    copyin(%A : memref<?xf32>)
    copyout(%B : memref<?xf32>) {
  // data is available as device copies inside this region
  acc.terminator
}
```

### 146.3.4 Routine Annotations

```mlir
// acc.routine: annotate a function for device use
acc.routine @compute_kernel() seq
// or: gang, worker, vector, seq

// Usage: call from OpenACC region
func.func @compute_kernel(%val: f32) -> f32 attributes {acc.routine_seq} {
  // ...
}
```

### 146.3.5 Device Memory Declarations

```mlir
// acc.declare_device_resident: allocate static data on device
acc.declare_device_resident varPtr(%static_buf : memref<1024xf32>)

// acc.declare_link: device memory accessible from multiple program units
acc.declare_link varPtr(%shared_data : memref<?xf32>)
```

### 146.3.6 Lowering

```bash
# Convert OpenACC to SCF loops (for CPU fallback)
mlir-opt --convert-openacc-to-scf input.mlir

# Convert OpenACC to LLVM runtime calls
mlir-opt --convert-openacc-to-llvm input.mlir
```

`--convert-openacc-to-llvm` maps acc constructs to calls in the OpenACC runtime (typically provided by the GPU vendor: `acc_init`, `acc_malloc`, `acc_copyin`, etc.).

---

## 146.4 The DLTI Dialect

### 146.4.1 Purpose

The DLTI dialect ([`mlir/include/mlir/Dialect/DLTI/DLTI.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/DLTI/DLTI.td)) provides a structured representation of data layout and target system specifications. Its purpose is to answer questions like "how many bytes does `i32` take on this target?" and "what is the cache line size?" — information needed for correct and efficient memory layout decisions.

This is the MLIR-native alternative to LLVM's `DataLayout` string. While LLVM uses a compact string like `"e-m:e-i64:64-f80:128-n8:16:32:64-S128"`, DLTI provides structured attributes that are human-readable and extensible.

### 146.4.2 Data Layout Attributes

```mlir
// Target system specification: machine-level layout info
#dlti.target_system_spec<
  "CPU": #dlti.target_device_spec<
    "endianness" = "little",
    "pointer_bit_width" = 64 : i32,
    "pointer_alignment" = 8 : i32,
    "native_integer_widths" = [8, 16, 32, 64] : vector<4xi32>
  >
>

// On a module:
builtin.module attributes {
  dlti.dl_spec = #dlti.dl_spec<
    #dlti.dl_entry<i8, dense<[8, 8]> : vector<2xi64>>,   // size=8 bits, abi-align=8 bits
    #dlti.dl_entry<i32, dense<[32, 32]> : vector<2xi64>>,
    #dlti.dl_entry<i64, dense<[64, 64]> : vector<2xi64>>,
    #dlti.dl_entry<f32, dense<[32, 32]> : vector<2xi64>>,
    #dlti.dl_entry<!llvm.ptr, dense<[64, 64, 64, 32]> : vector<4xi64>>
    // [size_bits, abi_align_bits, pref_align_bits, idx_bits_for_offset]
  >
} {
  ...
}
```

The `dl_spec` attribute on a module provides per-type data layout entries. Each entry maps a type to a vector describing its size and alignment. This is what drives correct struct layout, array stride computation, and ABI-compatible calling conventions.

### 146.4.3 GPU Target Device Spec

```mlir
// GPU device specification
#dlti.target_system_spec<
  "GPU": #dlti.target_device_spec<
    #dlti.dl_entry<"max_work_group_size", 1024 : i64>,
    #dlti.dl_entry<"preferred_warp_size", 32 : i64>,
    #dlti.dl_entry<"num_compute_units", 80 : i64>,
    #dlti.dl_entry<"max_shared_mem_per_block", 49152 : i64>,  // 48 KB
    #dlti.dl_entry<"target_triple", "nvptx64-nvidia-cuda">
  >
>
```

This enables GPU-specific optimizations to query device properties (shared memory size for tiling decisions, warp size for shuffle-based reductions) directly from the module attributes without hardcoding values.

### 146.4.4 `DataLayout` API

The C++ `DataLayout` class queries data layout information:

```cpp
#include "mlir/Interfaces/DataLayoutInterfaces.h"

// Query type sizes from data layout
DataLayout dataLayout(moduleOp);

uint64_t i32_bytes = dataLayout.getTypeSize(i32Type);       // 4
uint64_t i32_bits  = dataLayout.getTypeSizeInBits(i32Type); // 32
uint64_t i32_abi   = dataLayout.getTypeABIAlignment(i32Type); // 4

// For pointer
uint64_t ptr_bits = dataLayout.getTypeIndexBitwidth(ptrType); // 64 on x86-64
```

The `DataLayoutOpInterface` allows ops to provide custom data layout information — memref lowering uses this to compute struct sizes for the memref descriptor, and the `index` type lowering uses it to determine whether `index` is `i32` or `i64`.

---

## 146.5 The `emitc` Dialect

### 146.5.1 C Code Generation

The `emitc` dialect ([`mlir/include/mlir/Dialect/EmitC/IR/EmitC.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/EmitC/IR/EmitC.td)) enables MLIR programs to target C as their output language rather than LLVM IR. This is essential for:

- **Microcontrollers and embedded systems**: many MCU targets lack LLVM backends but have C compilers (e.g., Keil MDK, IAR, RISC-V GCC for custom cores)
- **Safety-critical systems**: C output can be reviewed, audited, and compiled with certified compilers
- **Testing and verification**: C output is human-readable, making it easy to verify correctness of the compilation
- **Android NDK targets**: MLIR → C → NDK build system integration

### 146.5.2 Basic Operations

```mlir
// emitc.func: C function definition
emitc.func @add(%a: f32, %b: f32) -> f32
    attributes {specifiers = ["static"]} {
  %sum = emitc.add %a, %b : (f32, f32) -> f32
  emitc.return %sum : f32
}

// Arithmetic (these become C operators)
%add = emitc.add %a, %b : (f32, f32) -> f32        // a + b
%sub = emitc.sub %a, %b : (f32, f32) -> f32        // a - b
%mul = emitc.mul %a, %b : (f32, f32) -> f32        // a * b
%div = emitc.div %a, %b : (f32, f32) -> f32        // a / b
%rem = emitc.rem %a, %b : (i32, i32) -> i32        // a % b

// Bitwise
%and = emitc.bitwise_and %a, %b : (i32, i32) -> i32  // a & b
%or  = emitc.bitwise_or  %a, %b : (i32, i32) -> i32  // a | b
%xor = emitc.bitwise_xor %a, %b : (i32, i32) -> i32  // a ^ b
%shl = emitc.shift_left  %a, %b : (i32, i32) -> i32  // a << b
%shr = emitc.shift_right %a, %b : (i32, i32) -> i32  // a >> b

// Comparison
%lt = emitc.cmp lt, %a, %b : (f32, f32) -> i1   // a < b
%eq = emitc.cmp eq, %a, %b : (i32, i32) -> i1   // a == b
```

### 146.5.3 Control Flow and Expressions

```mlir
// emitc.if: C if-else
emitc.if %cond {
  // then body
} else {
  // else body
}

// emitc.for: C for loop
emitc.for %i : i32 = %c0 to %n step %c1 {
  // loop body
}

// emitc.while: C while loop
emitc.while %cond : i1 {
  // body
}

// emitc.expression: inline C expression (arbitrary C expression as an op)
%result = emitc.expression : i32 {
  %tmp = emitc.add %a, %b : (i32, i32) -> i32
  emitc.yield %tmp : i32
}
```

### 146.5.4 C Operators and Casts

```mlir
// emitc.apply: apply C prefix operator
%ptr = emitc.apply "&"(%var) : (!emitc.lvalue<i32>) -> !emitc.ptr<i32>  // &var
%val = emitc.apply "*"(%ptr) : (!emitc.ptr<i32>) -> i32                  // *ptr

// emitc.cast: C-style cast
%ui32 = emitc.cast %i32val : i32 to ui32

// emitc.subscript: array subscript
%elem = emitc.subscript %arr[%idx] : (!emitc.ptr<f32>, i32) -> !emitc.lvalue<f32>

// emitc.member: struct member access
%field = emitc.member "x" of %point : (!emitc.ptr<!emitc.opaque<"Point">>) -> !emitc.lvalue<f32>
%field2 = emitc.member_of_ptr "y" of %ptr : (!emitc.ptr<!emitc.opaque<"Point">>) -> !emitc.lvalue<f32>
```

### 146.5.5 Opaque Calls and Verbatim

```mlir
// Call an external C function by name (opaque — no type checking of callee)
%result = emitc.call_opaque "my_c_function"(%arg0, %arg1)
    : (i32, f32) -> f32

// Call a standard library function
emitc.call_opaque "printf"(%fmt, %val) : (!emitc.ptr<ui8>, f32) -> ()

// Verbatim: emit arbitrary C text
emitc.verbatim "#include <stdlib.h>"
emitc.verbatim "typedef struct { float x, y; } Point;"
emitc.verbatim "static_assert(sizeof(int) == 4, \"int must be 4 bytes\");"
```

`emitc.call_opaque` is the primary mechanism for calling external C functions or C macros from MLIR-generated code. The callee is a string and is emitted verbatim in the C output.

`emitc.verbatim` emits raw C strings, useful for:
- Include directives
- Type declarations
- Static asserts
- Compiler pragmas (`#pragma once`, `__attribute__((...))`

### 146.5.6 EmitC Types

```mlir
!emitc.opaque<"Point">          // opaque C type by name (struct, typedef, etc.)
!emitc.ptr<f32>                 // C pointer: float*
!emitc.lvalue<i32>              // C lvalue (addressable location)
!emitc.array<4 x f32>           // C array: float[4]
```

`!emitc.lvalue<T>` is an important type — it represents a C lvalue (something that can appear on the left side of an assignment). Variables and array elements are lvalues; temporary expression results are not. The distinction is important for generating valid C code.

### 146.5.7 Variable Assignment

```mlir
// Declare a mutable variable (emitc.variable)
%var = emitc.variable {} : !emitc.lvalue<i32>

// Assign to a variable
emitc.assign %new_val : i32 to %var : !emitc.lvalue<i32>

// Load from a variable
%val = emitc.load %var : !emitc.lvalue<i32> -> i32
```

This models C's `int x = 0; x = x + 1;` pattern. Unlike the rest of MLIR (which is purely SSA), `emitc` allows mutable variables through `emitc.lvalue`. This is necessary because C code must be readable, and overly complex SSA-phi patterns cannot always be expressed as clean C.

### 146.5.8 Lowering and Translation

```bash
# Translate MLIR (emitc dialect) to C source file
mlir-translate --mlir-to-c input.mlir -o output.c

# Then compile with any C compiler:
gcc -O2 output.c -o output
arm-none-eabi-gcc -mcpu=cortex-m4 output.c -o firmware.elf
```

The `--mlir-to-c` translator ([`mlir/lib/Target/Cpp/CppEmitter.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Target/Cpp/CppEmitter.cpp)) walks the module and emits formatted C code. It handles:
- Function prototypes before definitions
- Variable declarations with correct C types
- Operator precedence and parenthesization
- Struct member access chains
- Verbatim text insertions

### 146.5.9 Lowering Higher-Level Dialects to EmitC

Several passes lower higher-level dialects to `emitc`:

```bash
# Lower arith ops to emitc operators
mlir-opt --convert-arith-to-emitc input.mlir

# Lower SCF control flow to emitc if/for/while
mlir-opt --convert-scf-to-emitc input.mlir

# Lower func dialect to emitc functions
mlir-opt --convert-func-to-emitc input.mlir

# Lower memref to emitc pointer operations
mlir-opt --convert-memref-to-emitc input.mlir
```

A complete pipeline for MCU code generation:

```bash
mlir-opt \
  --convert-linalg-to-loops \
  --lower-affine \
  --convert-scf-to-emitc \
  --convert-arith-to-emitc \
  --convert-func-to-emitc \
  --convert-memref-to-emitc \
  input.mlir | \
mlir-translate --mlir-to-c -o firmware_compute.c

arm-none-eabi-gcc -O2 -mcpu=cortex-m4 -mfpu=fpv4-sp-d16 firmware_compute.c -o firmware.elf
```

---

## 146.6 Dialect Interplay

These five dialects often appear together in real pipelines:

**async + gpu**: MLIR GPU programs use `async` tokens to overlap CPU work with GPU computation — `gpu.alloc async`, `gpu.memcpy async`, and `gpu.launch_func async` all produce `!async.token` values.

**omp + dlti**: OpenMP target regions use DLTI to determine pointer sizes for correct runtime call argument marshaling between host and device.

**emitc + dlti**: When generating C code for specific targets, DLTI provides the `sizeof`/`alignof` information used to generate correct struct layouts in the C output.

**acc + async**: OpenACC async directives can be represented in `acc` using async clause and lowered through async tokens before final lowering.

---

## Chapter 146 Summary

- The `async` dialect provides structured asynchronous execution: `async.execute` dispatches work asynchronously, returning `!async.token` and `!async.value<T>`. `async.await` synchronizes. The fan-out/fan-in pattern uses `!async.group`. Lowering pipeline: `--async-to-async-runtime` (coroutinizes), `--async-runtime-ref-counting`, `--convert-async-to-llvm` (maps to thread pool runtime).
- The `omp` dialect models OpenMP 5.1: `omp.parallel` (fork-join), `omp.wsloop` (worksharing), `omp.simd` (vectorization hint), `omp.critical`/`omp.barrier`/`omp.sections` (synchronization), `omp.task`/`omp.taskgroup` (task model), `omp.target`/`omp.teams` (offload). `--convert-openmp-to-llvm` maps to `__kmpc_*` runtime calls.
- The `acc` dialect models OpenACC 3.x: `acc.parallel`/`acc.kernels`/`acc.serial` (compute constructs), `acc.copyin`/`acc.copyout`/`acc.create`/`acc.present` (data management), `acc.data` (structured data regions), `acc.routine` (device function annotation). Lowered by `--convert-openacc-to-llvm` or `--convert-openacc-to-scf`.
- The `dlti` dialect provides `#dlti.dl_spec` (type size/alignment entries on modules), `#dlti.target_system_spec` (machine description), and `#dlti.target_device_spec` (GPU properties). The C++ `DataLayout` API queries this information to drive correct lowering decisions for pointer widths, struct layout, and ABI alignment.
- The `emitc` dialect enables MLIR → C code generation for MCU and embedded targets. Core ops: `emitc.func`, `emitc.call_opaque` (external C function calls), `emitc.verbatim` (raw C injection), `emitc.apply` (`&`/`*` operators), `emitc.expression`, `emitc.if`/`emitc.for`. `!emitc.lvalue<T>` models C addressable locations. Translation via `mlir-translate --mlir-to-c`.


---

@copyright jreuben11
