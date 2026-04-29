# Chapter 50 — Clang as SYCL, OpenCL, and OpenMP-Offload

*Part VII — Clang as a Multi-Language Compiler*

Where CUDA and HIP bind programmers to a single vendor's runtime, OpenCL, SYCL, and OpenMP offload each aim for hardware portability. Clang serves as the front end for all three: it compiles OpenCL C and C++ for compute runtimes on virtually every vendor's GPU and DSP; it is the compiler infrastructure underlying Intel's DPC++ SYCL implementation; and it generates device-side LLVM IR for OpenMP `#pragma omp target` regions that the `libomptarget` offload runtime dispatches to NVPTX and AMDGPU. This chapter examines the implementation of each path in depth, covering address space semantics, SPIR-V generation, OpenMP device codegen, SYCL kernel extraction, the `clang-offload-bundler`/`clang-offload-packager` fat-binary infrastructure shared across all offload models, and the mechanics of multi-target fat binary builds.

---

## 50.1 OpenCL in Clang

### 50.1.1 Language Mode Selection

Clang activates OpenCL mode via the `-x cl` (OpenCL C) or `-x cl++` (OpenCL C++) driver flags, or by file extensions `.cl` and `.clcpp`. The language standard is selected with `-cl-std=CL1.2`, `-cl-std=CL2.0`, or `-cl-std=CL3.0`. In `LangOptions`, the fields `OpenCL` (bool), `OpenCLVersion` (integer, e.g. 300), and `OpenCLGenericAddressSpace` track the active dialect.

The `CLOptions` struct in `clang/include/clang/Basic/LangOptions.h` consolidates OpenCL-specific flags:

```cpp
struct CLOptions {
  bool NativeHalfType;              // -cl-native-half-type
  bool NativeHalfArgsReturns;       // -cl-native-half-args-return
  bool DoubleSupport;               // cl_khr_fp64 available
  bool FP16Support;                 // cl_khr_fp16 available
  bool IntelRequiredSubgroupSize;   // intel extension
  bool UniformWGSize;               // -cl-uniform-work-group-size
  bool FastRelaxedMath;             // -cl-fast-relaxed-math
  bool UnsafeMath;                  // -cl-unsafe-math-optimizations
};
```

`OpenCLOptions` (in `clang/include/clang/Basic/OpenCLOptions.h`) tracks the set of enabled extensions as a `StringMap<OpenCLOptionInfo>`, populated from the target description and from explicit `-cl-ext=+cl_khr_fp64,-cl_khr_fp16` command-line arguments. Each OpenCL extension (`cl_khr_fp64`, `cl_khr_int64_base_atomics`, `cl_intel_subgroups`, `cl_amd_fp64`, etc.) has an entry; the `#pragma OPENCL EXTENSION cl_khr_fp64 : enable` directive toggles entries at parse time, and Sema checks that only enabled extensions are used.

### 50.1.2 Predefined Macros and OpenCL 3.0 Feature Macros

`clang/lib/Frontend/InitPreprocessor.cpp` defines OpenCL predefined macros in `InitializeOpenCLFeatureTestMacros`:

```cpp
// Version: CL3.0 = 300, CL2.0 = 200, CL1.2 = 120
Builder.defineMacro("__OPENCL_VERSION__", "300");
Builder.defineMacro("CL_VERSION_1_0", "100");
Builder.defineMacro("CL_VERSION_2_0", "200");
Builder.defineMacro("CL_VERSION_3_0", "300");

// CL3.0 optional feature macros
Builder.defineMacro("__opencl_c_program_scope_global_variables", "1");
Builder.defineMacro("__opencl_c_3d_image_writes", "1");
Builder.defineMacro("__opencl_c_int64", "1");           // 64-bit integer support
Builder.defineMacro("__opencl_c_fp64", "1");            // if cl_khr_fp64 present
Builder.defineMacro("__opencl_c_images", "1");          // image read/write
Builder.defineMacro("__opencl_c_atomic_order_acq_rel", "1");
Builder.defineMacro("__opencl_c_atomic_scope_device", "1");
```

OpenCL 3.0 made previously-mandatory CL 2.0 features optional and introduced feature macros to replace the `#ifdef __OPENCL_C_VERSION__ >= 200` pattern. Clang's OpenCL target description (`clang/lib/Basic/Targets/SPIR.cpp`, `AMDGPU.cpp`, `NVPTX.cpp`) lists which optional features each platform supports; this feeds the `OpenCLOptions` map and thereby controls which feature macros are defined.

### 50.1.3 Address Space Model

OpenCL's named address spaces map to LLVM IR address spaces. The mapping is target-defined; the standard assignment for the SPIR/SPIR-V target (verified against actual Clang 22.1.3 SPIR output):

| OpenCL qualifier | LLVM AS (SPIR) | LLVM AS (AMDGPU) | Semantics |
|------------------|---------------|------------------|-----------|
| `__private` / default | 0 | 5 | Per-thread private (register/stack) |
| `__global` | 1 | 1 | Device global memory |
| `__constant` | 2 | 4 | Read-only constant memory (SPIR:2, AMDGPU:4) |
| `__local` | 3 | 3 | Workgroup local (shared) memory |
| `__generic` (CL2.0+) | 4 | 0 (flat) | Unified address space |

Note the difference between SPIR and AMDGPU: constant memory is address space 2 on SPIR targets but address space 4 on AMDGPU. This mapping is performed by `getTargetAddressSpace(LangAS)` in the respective `TargetInfo` subclass. Clang's `LangAS` enum (in `clang/include/clang/Basic/AddressSpaces.h`) uses named constants like `LangAS::opencl_global`, `LangAS::opencl_local`, etc., as the canonical representation; the mapping to numeric LLVM IR address spaces is deferred to the target.

The `OpenCLKernelAttr` attribute is attached to kernel functions; Clang's `SemaOpenCL.cpp` enforces the OpenCL restrictions on kernel signatures: no pointer-to-pointer in kernel args, address space consistency on pointer parameters, and no `__private` pointer kernel arguments:

```cpp
// Valid OpenCL kernel with explicit address spaces
__kernel void vector_add(
    __global float * restrict a,      // SPIR: addrspace(1)
    __global float * restrict b,      // SPIR: addrspace(1)
    __local  float *scratch,          // SPIR: addrspace(3)
    __constant float *coeffs,         // SPIR: addrspace(2)
    int n)
{
    int gid = get_global_id(0);
    int lid = get_local_id(0);
    scratch[lid] = a[gid];
    barrier(CLK_LOCAL_MEM_FENCE);
    a[gid] = scratch[lid] * coeffs[0] + b[gid];
}
```

`SemaOpenCL::checkOpenCLAddressSpaces()` validates implicit and explicit address space casts. Casting `__global float *` to `__local float *` is an error; casting either to `__generic float *` (in CL2.0+) is permitted.

### 50.1.4 Built-in Functions and the OpenCL Builtins Database

OpenCL built-in functions (`get_global_id`, `barrier`, `atomic_add`, `mad`, `fma`, `clz`, `async_work_group_copy`, etc.) are declared in the OpenCL built-in function database (`clang/lib/Sema/OpenCLBuiltins.td`), which is TableGen-driven. The `.td` file declares each built-in's polymorphic signature using type qualifiers and type classes (`gentype`, `gentypef`, `gentypen`, etc.) that expand to all valid overload combinations. At a call site, `SemaOpenCL::checkBuiltinFunctionCall()` resolves the correct overload by matching argument types against the expanded overload set.

For SPIR-V targets, Clang maps OpenCL built-ins to `spir_func` calling convention calls that the SPIR-V backend recognises as OpenCL extended instruction references:

```llvm
; SPIR-V LLVM IR for get_global_id(0) — verified output:
; clang -x cl -cl-std=CL2.0 --target=spir64-unknown-unknown -emit-llvm -S

define spir_kernel void @vector_add(...) {
  %gid = call spir_func i64 @_Z13get_global_idj(i32 0)
  ; ...
}

declare spir_func i64 @_Z13get_global_idj(i32)
```

The calling convention `spir_func` (CC 75 in LLVM) and `spir_kernel` (CC 76) are OpenCL-specific LLVM calling conventions that instruct the SPIR-V backend to emit the correct OpenCL linkage: `spir_kernel` functions get `OpEntryPoint` with the `Kernel` execution model; `spir_func` calls become regular `OpFunctionCall` instructions.

### 50.1.5 Clang as the OpenCL Front End for Compute Runtimes

Several production compute runtime stacks use Clang as their OpenCL compiler frontend:

| Runtime | Backend path |
|---------|-------------|
| Intel OpenCL (NEO/IGC) | Clang → SPIR-V → Intel Graphics Compiler (IGC) |
| Mesa Rusticl | Clang → SPIR-V → LLVM AMDGPU/LLVMPIPE/VIRGL |
| Mesa Clover (legacy) | Clang → LLVM IR → target backend |
| Qualcomm Adreno | Clang → SPIR-V → Adreno offline compiler |
| AMD ROCm OpenCL | Clang → LLVM IR → AMDGPU backend → code object |
| Apple OpenCL (legacy) | Clang → LLVM IR → Apple GPU backend |

The `opencl_unroll_hint` pragma controls loop unrolling:

```c
#pragma opencl unroll_hint 4
for (int i = 0; i < N; i++) {
    sum += a[i] * b[i];
}
```

This emits `!llvm.loop` metadata with `llvm.loop.unroll.count` = 4, identical to the C/C++ `#pragma unroll` mechanism. The `__attribute__((reqd_work_group_size(64, 1, 1)))` kernel attribute emits `!reqd_work_group_size` metadata that runtime compilers use for occupancy calculations.

---

## 50.2 SPIR and SPIR-V Generation

### 50.2.1 SPIR (LLVM Bitcode Subset)

SPIR 1.2/2.0 is a portable intermediate representation for OpenCL based on a restricted subset of LLVM bitcode. Clang produces SPIR by targeting `spir-unknown-unknown` (32-bit) or `spir64-unknown-unknown` (64-bit):

```bash
clang -x cl -cl-std=CL1.2 \
      -target spir64-unknown-unknown \
      -emit-llvm -S \
      -o kernel.ll kernel.cl
```

SPIR bitcode imposes restrictions on the LLVM IR subset: no target-specific intrinsics, no backend-specific attributes, a defined set of calling conventions, and mandatory metadata blocks (`!opencl.ocl.version`, `!opencl.spir.version`, `!kernel_arg_addr_space`, `!kernel_arg_type`). These metadata blocks allow SPIR consumers to reconstruct the original kernel argument information after bitcode serialisation.

### 50.2.2 SPIR-V Generation via the In-Tree Backend

LLVM 14+ includes an in-tree SPIR-V backend (`llvm/lib/Target/SPIRV/`). It can be targeted directly:

```bash
clang -x cl -cl-std=CL3.0 \
      -target spirv64-unknown-unknown \
      -o kernel.spv kernel.cl
```

The SPIR-V backend (`SPIRVTargetMachine` in `llvm/lib/Target/SPIRV/SPIRVTargetMachine.cpp`, `SPIRVAsmPrinter`) traverses LLVM IR and emits binary SPIR-V. It handles:

- **Type deduplication**: SPIR-V's type system requires unique `OpType*` instructions; the backend builds a type deduplication map before emitting
- **`OpDecorate` annotations**: LLVM IR attributes (`readonly`, `writeonly`, address space attributes) become `OpDecorate` / `OpMemberDecorate` instructions
- **OpenCL extended instruction set**: Math operations map to `OpExtInst` with `GLSL.std.450` or the OpenCL extended instruction set references
- **Kernel argument metadata**: `!kernel_arg_type` and `!kernel_arg_addr_space` metadata from Clang are translated into `OpFunctionParameter` decorations
- **OpenCL 3.0 capability bits**: `__opencl_c_int64` adds the `Int64` capability; `__opencl_c_fp64` adds `Float64`; `__opencl_c_images` adds `ImageBasic`

The SPIR-V module structure emitted by Clang for a minimal OpenCL kernel:

```
OpCapability Kernel
OpCapability Addresses
OpCapability Int64
OpCapability Float64
OpMemoryModel Physical64 OpenCL
OpEntryPoint Kernel %vector_add "vector_add" ...
OpExecutionMode %vector_add ContractionOff
OpDecorate %a Alignment 4
; type section: OpTypeVoid, OpTypePointer, etc.
; constant section
; function definitions
```

### 50.2.3 SPIR-V via the SPIRV-LLVM Translator

An alternative path uses the Khronos `SPIRV-LLVM-Translator` (`llvm-spirv` tool), which predates the in-tree backend and supports a broader set of SPIR-V extensions including `SPV_INTEL_subgroups`, `SPV_KHR_float_controls`, and `SPV_EXT_shader_atomic_float_add`:

```bash
# Clang → LLVM IR → SPIR-V via external translator
clang -x cl -cl-std=CL2.0 \
      -target spir64-unknown-unknown \
      -emit-llvm -o kernel.bc kernel.cl

llvm-spirv -spirv-ext=+SPV_INTEL_subgroups \
           -o kernel.spv kernel.bc

# Reverse: SPIR-V → LLVM IR (for inspection or re-optimisation)
llvm-spirv -r -o kernel.bc kernel.spv
```

The `spirv-tools` package provides `spirv-val` for validation against the SPIR-V specification, `spirv-dis` for disassembly, and `spirv-opt` for SPIR-V level optimisations (dead code elimination, loop unrolling, scalar replacement). For production use, the translator is often preferred because of its extension coverage; the in-tree backend is the path of least friction in monorepo builds.

### 50.2.4 OpenCL C++ and Template Kernels

OpenCL C++ (standardised as OpenCL C++ 1.0 in the OpenCL 3.0 specification) provides templates, lambdas, and class types. Clang's `-x cl++` mode enables C++ parsing with OpenCL restrictions. Template kernels over address spaces are supported:

```cpp
// OpenCL C++ template over generic type and address space
template <typename T>
__kernel void scale(
    __global T *dst,
    __global const T *src,
    T factor, int n)
{
    int i = get_global_id(0);
    if (i < n) dst[i] = src[i] * factor;
}
```

The restrictions compared to host C++: no exceptions, no dynamic allocation (`new`/`delete`), no virtual functions, no recursion (callee cycles prohibited), no function pointers in kernel parameters, no non-trivial global constructors/destructors. The `clang -cl-std=CLC++2021` flag enables the OpenCL C++ 2021 dialect.

---

## 50.3 OpenMP Target Offloading

### 50.3.1 Driver and Compilation Model

OpenMP target offload compilation follows the same dual-compilation model as HIP. The primary flags:

```bash
# Multi-target: compile for both NVPTX and AMDGPU
clang++ -fopenmp \
        -fopenmp-targets=nvptx64-nvidia-cuda,amdgcn-amd-amdhsa \
        -Xopenmp-target=nvptx64-nvidia-cuda -march=sm_90 \
        -Xopenmp-target=amdgcn-amd-amdhsa   -march=gfx942 \
        dgemm.cpp -o dgemm
```

`-Xopenmp-target=<triple>` passes the following argument only to the device compilation for that specific triple. The driver creates one `OffloadAction` per target triple; each spawns a separate `clang` process for device compilation. The resulting device objects are packaged by `clang-offload-packager` into a fat binary embedded in the host object.

The three-layer runtime stack for OpenMP offload:
1. **`libomp.so`** — the OpenMP host runtime, managing thread pools, task queues, and the `#pragma omp parallel` execution model
2. **`libomptarget.so`** — the offload dispatch runtime, loading plugins, managing device allocation tables, and calling `__tgt_*` entry points
3. **`libomptarget.rtl.*.so`** — device-specific plugins (CUDA, AMDGPU, x86 serial fallback), each implementing the `PluginInterface`

### 50.3.2 AST Nodes for Target Directives

The Clang AST represents OpenMP target directives as `OMPTargetDirective` nodes and derived classes. The inheritance hierarchy:

```
Stmt
└── OMPExecutableDirective
    ├── OMPTargetDirective
    ├── OMPTargetTeamsDirective
    ├── OMPTargetTeamsDistributeDirective
    ├── OMPTargetTeamsDistributeParallelForDirective
    ├── OMPTargetParallelDirective
    ├── OMPTargetParallelForDirective
    └── OMPTaskDirective    (for async task offload via nowait+depend)
```

Each directive node carries a list of `OMPClause*` objects and a `CapturedStmt` child containing the outlined region. Clause types relevant to offload:

- `OMPMapClause` — `map(to:/from:/tofrom:/alloc:)` data movement directives
- `OMPNumTeamsClause` — `num_teams(expr)` — sets the grid dimension
- `OMPThreadLimitClause` — `thread_limit(expr)` — caps threads per team (block size)
- `OMPNowaitClause` — async dispatch (no implicit barrier after the region)
- `OMPDependClause` — `depend(in:/out:/inout:)` for task dependence with `nowait`
- `OMPDeviceClause` — `device(expr)` — selects target device ID

For a complex target directive:

```cpp
#pragma omp target teams distribute parallel for \
        map(to: a[0:N], b[0:N]) map(from: c[0:N]) \
        num_teams(N/256) thread_limit(256) nowait
for (int i = 0; i < N; i++)
    c[i] = a[i] + b[i];
```

The AST node is `OMPTargetTeamsDistributeParallelForDirective` with an `OMPMapClause` (to: a, b), an `OMPMapClause` (from: c), `OMPNumTeamsClause`, `OMPThreadLimitClause`, `OMPNowaitClause`, and a `CapturedStmt` wrapping the for loop.

### 50.3.3 OpenMP Codegen Infrastructure

The OpenMP codegen path is split between three files:

- **`clang/lib/CodeGen/CGOpenMPRuntime.cpp`** — host/device shared codegen: target region outlining, map clause processing, runtime call emission
- **`clang/lib/CodeGen/CGOpenMPRuntimeGPU.cpp`** — GPU-specific overrides for both NVPTX and AMDGPU: workgroup size queries, barrier intrinsics, shared memory allocation
- **`clang/lib/CodeGen/CGStmtOpenMP.cpp`** — per-directive IR emission, reducing each AST directive to codegen method calls

`CGOpenMPRuntime` is an abstract base class with a factory method `createOpenMPRuntime(CodeGenModule &CGM)` that returns a concrete implementation: `CGOpenMPRuntimeGPU` for GPU device compilations, `CGOpenMPRuntime` for host compilations. The key virtual methods governing target region code generation:

```cpp
class CGOpenMPRuntime {
public:
  // Called for the host side of a #pragma omp target: emits
  // __tgt_target_kernel() or fallback to outlined function call
  virtual void emitTargetCall(
      CodeGenFunction &CGF,
      const OMPExecutableDirective &D,
      llvm::Function *OutlinedFn,
      llvm::Value *OutlinedFnID, ...);

  // Emit __tgt_target_data_begin_mapper() for map clauses
  virtual void emitTargetDataCalls(
      CodeGenFunction &CGF,
      const OMPTargetDataDirective &D, ...);

  // Outline the target region body as a standalone function
  virtual llvm::Function *emitParallelOutlinedFunction(
      const OMPExecutableDirective &D,
      const VarDecl *ThreadIDVar,
      OpenMPDirectiveKind InnermostKind, ...);
};
```

### 50.3.4 Target Region Outlining and Naming

The critical codegen step is *outlining*: the body of a `#pragma omp target` region is extracted into a standalone LLVM function. The outlined function name follows the pattern (confirmed against actual Clang 22.1.3 IR output):

```
__omp_offloading_<file_hex>_<func_hex>__<mangled_func_name>_l<line>
```

For example, a target region in `/tmp/test_omp.cpp` inside function `_Z5dgemmPdS_S_i` at line 2 produces:
```
__omp_offloading_fc01_7009fec__Z5dgemmPdS_S_i_l2
```

The file hash (`fc01`) and function hash (`7009fec`) are computed from the source file path and function identity. The name is used by `libomptarget` at runtime to look up the corresponding device function in the fat binary.

The outlined function's parameters consist of all variables captured from the enclosing scope. Map clauses determine whether captures are passed by value (scalars, `firstprivate`) or by device pointer (mapped arrays that require `__tgt_target_data_begin_mapper`). A `__tgt_offload_entry` structure is emitted for each target region in the `.llvm_offload_entries` section (the actual section name in Clang 22's output):

```cpp
// Actual structure from LLVM IR output
struct __tgt_offload_entry {
  i64   reserved1;   // 0
  i16   version;     // 1
  i16   kind;        // 1 = function
  i32   flags;       // 0
  ptr   addr;        // host address (region_id symbol)
  ptr   name;        // pointer to name string
  i64   size;        // 0 for functions
  i64   aux_addr;    // 0
  ptr   aux_ptr;     // null
};
```

### 50.3.5 Map Clauses and the Data Movement API

`#pragma omp target map(...)` clauses control data transfer between host and device. The codegen calls `libomptarget`'s mapper API:

```cpp
// Before target region: map listed arrays to device
void __tgt_target_data_begin_mapper(
    ident_t *loc,           // source location for diagnostics
    int64_t device_id,      // -1 = default device
    int32_t arg_num,        // number of mapped variables
    void **args_base,       // base pointers (original array start)
    void **args,            // actual pointers (subarray start)
    int64_t *arg_sizes,     // sizes in bytes
    int64_t *arg_types,     // OMP_TGT_MAPTYPE_* bitmask flags
    void **arg_mappers,     // custom mapper functions, or null
    ...);

// After target region: copy results back and/or release
void __tgt_target_data_end_mapper(...);
```

The `arg_types` bitmask encodes `OMP_TGT_MAPTYPE_TO` (copy H→D), `OMP_TGT_MAPTYPE_FROM` (copy D→H), `OMP_TGT_MAPTYPE_ALWAYS` (always copy, even if already present), `OMP_TGT_MAPTYPE_DELETE` (free device copy), and `OMP_TGT_MAPTYPE_PTR_AND_OBJ` (pointer target structure).

Clang's map clause lowering in `CGOpenMPRuntime::emitTargetDataCalls()` computes these bitmasks statically where possible. Dynamic array extents (variable-length VLAs or pointer+length pairs) become runtime-computed size arguments passed via `arg_sizes`. Nested data structures with pointer members require the `mapper` mechanism: a user-defined mapper function specifies recursively which pointer fields to follow during mapping.

### 50.3.6 GPU Thread Mapping and Device Codegen

On GPU targets, the `teams distribute parallel for` construct maps to the GPU execution hierarchy:

| OpenMP construct | NVPTX mapping | AMDGPU mapping |
|-----------------|---------------|----------------|
| `teams` | CUDA grid (CTAs) | Workgroups |
| `distribute` | Loop over thread blocks | Loop over workgroups |
| `parallel` | Thread block | Workgroup |
| `for` / `simd` | Loop within a warp | Loop within a wavefront |

`CGOpenMPRuntimeGPU` overrides `emitParallelOutlinedFunction` to emit GPU-specific thread-ID queries. Actual IR from the NVPTX device compilation:

```llvm
; NVPTX: omp_get_thread_num() on device
%tid   = call i32 @llvm.nvvm.read.ptx.sreg.tid.x()
%ntid  = call i32 @llvm.nvvm.read.ptx.sreg.ntid.x()
%ctaid = call i32 @llvm.nvvm.read.ptx.sreg.ctaid.x()

; AMDGPU equivalent
%tid   = call i32 @llvm.amdgcn.workitem.id.x()
%ntid  = call i32 @llvm.amdgcn.workgroup.size.x()
%ctaid = call i32 @llvm.amdgcn.workgroup.id.x()
```

The `nowait` clause on `#pragma omp target nowait` generates `__tgt_target_kernel_nowait` instead of `__tgt_target_kernel`, which submits the kernel asynchronously and returns immediately. The `depend` clause on `#pragma omp task target nowait depend(...)` generates a task with the device kernel as the task body and the dependency tracking handled by `libomp`'s task scheduler.

### 50.3.7 `__builtin_omp_required_simd_align` and SIMD Offload

The `__builtin_omp_required_simd_align(type)` built-in returns the required alignment for SIMD vectorisation of the given type under the current target and OpenMP SIMD settings. For a `#pragma omp simd` loop inside a target region, Clang emits `!llvm.loop` metadata with `llvm.loop.vectorize.enable` and `llvm.loop.vectorize.width` derived from the `simdlen` clause.

```cpp
#pragma omp target teams distribute parallel for simd simdlen(16)
for (int i = 0; i < N; i++)
    c[i] = a[i] + b[i];
```

On NVPTX, `simdlen(16)` generates a vectorized loop within each thread; on AMDGPU, it maps to 16-element VALU operations (e.g., 4× `v_add_f32` operating on `<4 x float>` registers). The backend honours the vectorization metadata from the OpenMP codegen.

---

## 50.4 OpenMP Device-Side Optimisations

### 50.4.1 The OpenMPOpt LLVM Pass

The `OpenMPOpt` pass (in `llvm/lib/Transforms/IPO/OpenMPOpt.cpp`) operates on device LLVM IR after target region outlining and performs several inter-procedural optimisations:

**Dead region elimination**: target regions whose condition is provably false (e.g., `#pragma omp target if(0)`) are replaced with the CPU fallback code. The `__kmpc_kernel_prepare_parallel()` / `__kmpc_kernel_parallel()` calls that implement the GPU state machine are elided when only one parallel region exists.

**Runtime call deduplication**: multiple `__kmpc_global_thread_num()` calls in the same function are merged into one. Multiple `__kmpc_init_common()` calls are hoisted to a single call per function.

**Attribute inference**: `nosync`, `nofree`, and `willreturn` attributes are inferred for outlined device functions that do not call synchronising operations, enabling further optimisations (LICM, dead store elimination) in downstream passes.

**State machine unfolding**: the generic GPU execution model uses a state machine loop where each participating thread picks up work items. When the parallel regions inside a teams region can be statically enumerated, the state machine is unfolded into direct function calls, eliminating the branch overhead.

Enabling optimization remarks:

```bash
clang++ -fopenmp -fopenmp-targets=nvptx64-nvidia-cuda \
        -Rpass=openmp-opt \
        -Rpass-missed=openmp-opt \
        -Rpass-analysis=openmp-opt \
        dgemm.cpp 2>&1 | grep remark
```

Example remarks from `OpenMPOpt`:

```
dgemm.cpp:15:3: remark: Removing parallel region with 0 iterations [-Rpass=openmp-opt]
dgemm.cpp:42:1: remark: Specialized parallel for single-threaded team [-Rpass=openmp-opt]
dgemm.cpp:58:1: remark: Rewriting device state machine to direct call [-Rpass=openmp-opt]
```

### 50.4.2 Unified Shared Memory and Device Allocation

The `omp_target_alloc()` / `omp_target_free()` functions allocate memory directly in device memory without the map/unmap overhead:

```cpp
// Allocate on device 0, bypassing map machinery
float *d_buf = (float*)omp_target_alloc(N * sizeof(float), 0);
#pragma omp target is_device_ptr(d_buf) teams distribute parallel for
for (int i = 0; i < N; i++)
    d_buf[i] *= 2.0f;
omp_target_free(d_buf, 0);
```

For systems supporting `unified_shared_memory` (APUs, NVLink systems with fine-grain UVA), the `requires` directive relaxes the need for explicit map clauses:

```cpp
#pragma omp requires unified_shared_memory

float *data = (float*)malloc(N * sizeof(float));
#pragma omp target teams distribute parallel for
for (int i = 0; i < N; i++)
    data[i] *= 2.0f;   // No map() needed: system memory accessible on device
```

### 50.4.3 Declare Target and Diagnostics

The `#pragma omp declare target` construct marks device-accessible variables and functions. Clang diagnoses several common misuse patterns:

- **`threadprivate` on device**: `threadprivate` variables have no meaning inside target regions (no persistent thread state on GPU). Clang emits `error: 'threadprivate' is not supported in target region`.
- **`declare target` on recursive functions**: deep recursion is unsupported on most GPU targets (limited stack). Clang emits a warning when a `declare target` function is recursive.
- **Implicit mapping of zero-length arrays**: `map(a[0:0])` silently maps nothing; Clang emits a warning `warning: zero-length array section in map clause`.
- **Stack usage analysis**: for offloaded regions, Clang tracks the maximum stack usage through `CGOpenMPRuntimeGPU::computeNestedParallelLevel()` and emits a warning if it exceeds the GPU private memory limit.

---

## 50.5 SYCL in Clang

### 50.5.1 SYCL Implementation Architecture

SYCL (pronounced "sickle") is a Khronos standard for single-source heterogeneous programming using standard ISO C++. Intel's DPC++ (Data Parallel C++) is the primary implementation, maintained in the `intel/llvm` fork of the LLVM monorepo. As of LLVM 22, upstream Clang contains the essential SYCL attribute infrastructure (`SemaSYCL.cpp`, `SYCLAttr`, `SYCLKernelAttr`); the Intel fork carries additional integration with the SYCL runtime and the `sycl::queue::submit` machinery.

The compilation command for Intel DPC++:

```bash
# Compile for OpenCL/SPIR-V (works on Intel GPU, CPU, FPGA)
clang++ -fsycl -fsycl-targets=spir64 -O2 -o vecadd vecadd.cpp

# For NVIDIA GPU (requires Codeplay plugin or oneAPI for CUDA)
clang++ -fsycl -fsycl-targets=nvptx64-nvidia-cuda-sycldevice \
        -Xsycl-target-backend '--cuda-gpu-arch=sm_90' \
        -O2 -o vecadd vecadd.cpp

# For AMD GPU
clang++ -fsycl -fsycl-targets=amdgcn-amd-amdhsa-sycldevice \
        -Xsycl-target-backend '--offload-arch=gfx942' \
        -O2 -o vecadd vecadd.cpp
```

`-fsycl` enables SYCL mode and `-fsycl-targets` is analogous to `-fopenmp-targets`: it selects the target triples for device compilation. The SYCL host compilation is a normal C++ compilation with SYCL headers.

### 50.5.2 SYCL Attributes and LangAS Extension

Clang's SYCL front end (in `clang/lib/Sema/SemaSYCL.cpp`) processes several SYCL-specific attributes:

- `[[sycl::kernel]]` — marks a lambda or functor type as a SYCL kernel (automatically applied to `parallel_for` callables)
- `[[sycl::device_has(aspect::fp64)]]` — requires the device to have a specific capability
- `[[intel::reqd_work_group_size(64)]]` — specifies required workgroup size; maps to `reqd_work_group_size` in LLVM IR metadata
- `[[intel::num_simd_work_items(8)]]` — SIMD vectorisation hint for Intel GPUs
- `[[sycl::work_group_size_hint(256)]]` — hint (not requirement) for workgroup size

The SYCL address spaces in `LangAS` enum (from `clang/include/clang/Basic/AddressSpaces.h`) are:

```cpp
sycl_global,          // → AS 1 (device global memory)
sycl_global_device,   // → AS 1 (device-only global)
sycl_global_host,     // → AS 1 (host-accessible global)
sycl_local,           // → AS 3 (workgroup local memory)
sycl_private,         // → AS 0 (private/register)
```

### 50.5.3 Kernel Extraction and the Kernel Integration Header

A SYCL kernel submission using USM (Unified Shared Memory):

```cpp
#include <sycl/sycl.hpp>

void vector_add(sycl::queue &q, float *a, float *b, float *c, int n) {
    // USM: pointers are directly accessible on device
    q.submit([&](sycl::handler &h) {
        h.parallel_for<class VecAdd>(
            sycl::range<1>(n),
            [=](sycl::id<1> idx) {
                c[idx] = a[idx] + b[idx];
            }
        );
    });
    q.wait();
}
```

During device compilation (`-fsycl`), Clang processes the `parallel_for` call. The lambda `[=](sycl::id<1> idx) { ... }` has its body outlined into a separate function with `spir_kernel` calling convention. The captured state (`a`, `b`, `c`, `n`) becomes the kernel argument list. The outlined function is decorated with `SYCLKernelAttr`, causing it to receive the name-mangled kernel entry point in the SPIR-V output.

The *Kernel Integration Header* (KIH) is emitted as a side effect of device compilation. It contains:

```cpp
// Auto-generated Kernel Integration Header (kernel_integration.h)
namespace sycl {
namespace detail {
// Kernel name → argument type mapping for queue::submit
template <> struct KernelInfo<class VecAdd> {
  static constexpr const char *getName() { return "_ZTSZZ10vector_addRN4sycl3_V15queueEPfS3_S3_iENKUlNS0_2idILi1EEEE_clES5_E6VecAdd"; }
  static constexpr unsigned getNumParams() { return 4; }
  // Parameter descriptors: pointer kind, access qualifier, address space
  static constexpr detail::kernel_param_desc_t getParamDesc(unsigned i) { ... }
};
} // namespace detail
} // namespace sycl
```

This KIH is `#include`d in the host compilation, allowing `sycl::queue::submit` to construct the correct kernel invocation at compile time without requiring dynamic reflection.

### 50.5.4 Specialisation Constants

SYCL 2020 specialisation constants allow compile-time constants whose values are fixed at runtime (before kernel compilation or JIT):

```cpp
constexpr sycl::specialization_id<float> alpha_id(1.0f);

q.submit([&](sycl::handler &h) {
    float alpha_val = 2.5f;
    h.set_specialization_constant<alpha_id>(alpha_val);
    h.parallel_for<class Scale>(sycl::range<1>(N), [=](sycl::id<1> i) {
        float alpha = h.get_specialization_constant<alpha_id>();
        c[i] = alpha * a[i];
    });
});
```

Clang represents specialisation constants as LLVM IR `@llvm.invariant.start` calls on a global that is replaced by the runtime before kernel dispatch. In SPIR-V, they map to `OpSpecConstant` instructions. The Intel runtime replaces the SPIR-V spec constants with actual values before submitting the kernel to the JIT/AOT compiler.

### 50.5.5 AdaptiveCpp / OpenSYCL

AdaptiveCpp (formerly hipSYCL/OpenSYCL) is an alternative SYCL implementation that uses Clang as its backend without requiring the Intel fork. Its compilation model:

1. Parse SYCL source with standard Clang + AdaptiveCpp's Clang plugin (`libacpp-clang.so`)
2. The plugin uses `ASTConsumer` hooks to identify SYCL kernel lambdas at AST time
3. A pass-through codegen produces device LLVM IR for each lambda, targeting NVPTX, AMDGPU, or CPU (via OpenMP)
4. The AdaptiveCpp runtime dispatches to HIP, CUDA, or OpenCL at runtime based on device discovery

AdaptiveCpp supports OpenMP offload as a "CPU SYCL backend": `omp.accelerated` target selects CPU-side threading via `#pragma omp parallel for` for the SYCL device operations. This enables SYCL programs to run on multicore CPUs without any GPU hardware.

---

## 50.6 `clang-offload-packager` and `clang-offload-bundler`

### 50.6.1 clang-offload-bundler

`clang-offload-bundler` (source: `clang/tools/clang-offload-bundler/ClangOffloadBundler.cpp`) is the original tool for creating and unbundling heterogeneous fat objects. It creates a `__CLANG_OFFLOAD_BUNDLE__` section in the host ELF object containing all device images, identified by target triple strings:

```
__CLANG_OFFLOAD_BUNDLE____START__  host-x86_64-unknown-linux-gnu
  [host ELF bytes]
__CLANG_OFFLOAD_BUNDLE____START__  openmp-nvptx64-nvidia-cuda-sm_90
  [NVPTX PTX + cubin bytes]
__CLANG_OFFLOAD_BUNDLE____START__  openmp-amdgcn-amd-amdhsa-gfx942
  [AMDGPU code object bytes]
```

Bundling multiple device targets:

```bash
clang-offload-bundler --type=o \
  --targets=host-x86_64-unknown-linux-gnu,\
            openmp-nvptx64-nvidia-cuda-sm_90,\
            openmp-amdgcn-amd-amdhsa-gfx942 \
  --input=host.o --input=device_sm90.o --input=device_gfx942.o \
  --output=fat.o

# Unbundling
clang-offload-bundler --unbundle --type=o \
  --targets=openmp-nvptx64-nvidia-cuda-sm_90 \
  --input=fat.o --output=device_sm90.o
```

### 50.6.2 clang-offload-packager (LLVM 15+)

`clang-offload-packager` is the newer driver-integrated tool for embedding device code. It creates a binary descriptor that is embedded as a `.llvm.offloading` ELF section in the host object by the linker:

```bash
# Internal driver usage (not typically invoked manually)
clang-offload-packager \
  --image=file=device_sm90.o,triple=nvptx64-nvidia-cuda,arch=sm_90,kind=openmp \
  --image=file=device_gfx942.o,triple=amdgcn-amd-amdhsa,arch=gfx942,kind=openmp \
  -o packaged.bin
```

The packager format is more compact than the bundler format: it uses a binary table-of-contents followed by optionally LZ4-compressed device images. It is the default format in LLVM 16+ when `-fopenmp-new-driver` is active (which it is by default in Clang 22).

### 50.6.3 clang-offload-wrapper

`clang-offload-wrapper` generates the C++ source for the registration constructor — the function that calls `__tgt_register_lib` at program startup to register device binaries with `libomptarget`:

```cpp
// Generated by clang-offload-wrapper (simplified)
static __tgt_device_image DeviceImages[] = {
  { /* NVPTX sm_90 image start/end pointers */ },
  { /* AMDGPU gfx942 image start/end pointers */ }
};

static __tgt_bin_desc BinDesc = {
    2 /* NumDevImages */,
    DeviceImages,
    HostEntries,
    HostEntries + NumEntries
};

__attribute__((constructor)) static void __tgt_register(void) {
    __tgt_register_lib(&BinDesc);
}
__attribute__((destructor)) static void __tgt_unregister(void) {
    __tgt_unregister_lib(&BinDesc);
}
```

`__tgt_register_lib` iterates the device image table, queries each loaded plugin (via `PluginInterface::load_binary()`), and builds a symbol table mapping host-side `__omp_offloading_*` entry point names to device function handles.

### 50.6.4 offload-arch Tool

The `offload-arch` utility queries the system for installed GPUs and returns their canonical architecture identifiers:

```bash
$ offload-arch
gfx942         # AMD Instinct MI300X
sm_90a         # NVIDIA H100 (sm_90a is the compute capability)
```

This enables `--offload-arch=native` in the driver, which expands to all detected GPU architectures. A build system can use `offload-arch -v` to get verbose output including architecture capability flags.

### 50.6.5 libomptarget Plugin Architecture

`libomptarget.so` (source: `openmp/libomptarget/`) is the central OpenMP offload runtime dispatcher. It loads target-specific plugins at startup from directories in `LIBRARY_PATH`:

| Plugin DSO | Target | Launch mechanism |
|------------|--------|-----------------|
| `libomptarget.rtl.cuda.so` | NVPTX (CUDA) | `cuLaunchKernel` |
| `libomptarget.rtl.amdgpu.so` | AMDGPU (ROCm) | `hsa_queue_add_write_index` + AQL dispatch packet |
| `libomptarget.rtl.level0.so` | Intel GPU (Level Zero) | `zeCommandListAppendLaunchKernel` |
| `libomptarget.rtl.x86_64.so` | CPU serial fallback | Direct function call |

Each plugin implements `GenericPluginTy` (the modern interface in LLVM 22) with methods:

```cpp
// From openmp/libomptarget/plugins-nextgen/common/include/PluginInterface.h
class GenericPluginTy {
public:
  virtual Error init() = 0;
  virtual Error deinit() = 0;
  virtual Expected<void *> dataAlloc(int64_t Size, void *HstPtr) = 0;
  virtual Error dataDelete(void *TgtPtr) = 0;
  virtual Error dataSubmit(void *TgtPtr, const void *HstPtr, int64_t Size,
                           AsyncInfoTy &AsyncInfo) = 0;
  virtual Error dataRetrieve(void *HstPtr, const void *TgtPtr, int64_t Size,
                             AsyncInfoTy &AsyncInfo) = 0;
  virtual Error launchKernel(void *TgtEntryPtr, void **TgtArgs,
                             ptrdiff_t *TgtOffsets,
                             KernelArgsTy &KernelArgs,
                             AsyncInfoTy &AsyncInfo) = 0;
};
```

The `AsyncInfoTy` parameter carries a stream/queue handle for asynchronous operations; `libomptarget` accumulates operations in the async handle and flushes them at synchronisation points.

---

## 50.7 Compilation Database and Multi-Target Builds

### 50.7.1 Multi-Target Fat Binaries

A single compilation can produce code for multiple GPU architectures simultaneously:

```bash
clang++ -fopenmp \
  -fopenmp-targets=nvptx64-nvidia-cuda,nvptx64-nvidia-cuda,amdgcn-amd-amdhsa \
  -Xopenmp-target=nvptx64-nvidia-cuda -march=sm_80 \
  -Xopenmp-target=nvptx64-nvidia-cuda -march=sm_90 \
  -Xopenmp-target=amdgcn-amd-amdhsa   -march=gfx942 \
  dgemm.cpp -o dgemm
```

The fat binary embeds three device images. `libomptarget` at runtime calls `PluginInterface::get_capabilities()` for the active device and selects the device image whose `arch` field best matches. For NVIDIA GPUs, sm_90 code runs on an sm_90a (H100) device; sm_80 code runs on an A100. The selection follows CUDA's "best matching" rule: the highest SM version that is ≤ the device's compute capability.

### 50.7.2 --offload-arch=native

When `--offload-arch=native` is specified, the driver calls `offload-arch` (or an equivalent internal GPU probe) to enumerate installed GPUs:

```bash
# Auto-detect and compile for all installed GPUs
clang++ -fopenmp -fopenmp-targets=nvptx64-nvidia-cuda,amdgcn-amd-amdhsa \
        --offload-arch=native dgemm.cpp -o dgemm
```

This is particularly useful in CI environments where the GPU model is not fixed at source time. The `offload-arch` tool reads device properties from `/sys/class/drm/renderD*/device/uevent` (AMD) and the CUDA driver API (NVIDIA).

### 50.7.3 CMake Integration and Device Bitcode Libraries

For OpenMP offload in CMake, the standard `FindOpenMP` module provides `OpenMP_CXX_FLAGS` but does not handle multi-target offload:

```cmake
# OpenMP offload CMake pattern
set(OMP_OFFLOAD_FLAGS
    "-fopenmp -fopenmp-targets=nvptx64-nvidia-cuda,amdgcn-amd-amdhsa"
    "-Xopenmp-target=nvptx64-nvidia-cuda -march=${NVPTX_ARCH}"
    "-Xopenmp-target=amdgcn-amd-amdhsa -march=${AMDGPU_ARCH}")

target_compile_options(dgemm PRIVATE ${OMP_OFFLOAD_FLAGS})
target_link_options(dgemm PRIVATE ${OMP_OFFLOAD_FLAGS})

# Compile a device bitcode library (no host code)
add_library(device_math OBJECT math_kernels.cpp)
target_compile_options(device_math PRIVATE
    -fopenmp -fopenmp-targets=nvptx64-nvidia-cuda
    -Xopenmp-target -march=sm_90
    --offload-device-only)
```

`--offload-device-only` compiles only the device side, producing a device bitcode library that can be linked into other device compilations.

### 50.7.4 -fno-offload-uniformity-analysis

The uniformity (divergence) analysis run on GPU device code can sometimes prevent optimisations by conservatively marking values as divergent. The flag `-fno-offload-uniformity-analysis` disables Clang-level uniformity analysis, relying instead on the backend's `AMDGPUDivergenceAnalysis` or NVPTX's `DivergenceAnalysis`:

```bash
clang++ -fopenmp -fopenmp-targets=amdgcn-amd-amdhsa \
        -Xopenmp-target=amdgcn-amd-amdhsa -march=gfx942 \
        -fno-offload-uniformity-analysis \
        dgemm.cpp
```

This can improve codegen quality for loops where the trip count is uniform but the analysis cannot prove it statically. The tradeoff is that the Clang-level analysis that feeds SGPR/VGPR hints to the backend is weakened; genuinely divergent values may be promoted to SGPR paths, potentially generating incorrect `v_readfirstlane_b32` insertions.

---

## 50.8 Cross-Model Interoperability

### 50.8.1 HIP + OpenMP Coexistence

A single translation unit can combine HIP kernels and OpenMP target regions. Clang handles this by producing both HIP-style `__hipRegisterFatBinary` registration and OpenMP-style `__tgt_register_lib` registration from the same fat object:

```cpp
// HIP kernel
__global__ void hip_kernel(float *a, int n) { ... }

// OpenMP target region in same TU
void omp_offload(float *a, int n) {
    #pragma omp target teams distribute parallel for map(tofrom:a[0:n])
    for (int i = 0; i < n; i++) a[i] *= 2.0f;
}
```

Compilation:

```bash
clang++ -x hip -fopenmp -fopenmp-targets=amdgcn-amd-amdhsa \
        --offload-arch=gfx942 mixed.cpp -o mixed
```

### 50.8.2 SYCL Interoperability with Underlying Native APIs

SYCL 2020 provides backend interoperability through `sycl::get_native<>`:

```cpp
#include <sycl/sycl.hpp>
#include <sycl/ext/oneapi/backend/hip.hpp>

sycl::queue q;
sycl::buffer<float, 1> buf(N);

// Extract native HIP stream from SYCL queue
hipStream_t hip_stream = 
    sycl::get_native<sycl::backend::ext_oneapi_hip>(q);

// Call HIP API directly on the same stream
hipLaunchKernelGGL(my_hip_kernel, grid, block, 0, hip_stream, ...);

// Synchronise: SYCL operations submitted after this point
// will wait for the HIP kernel via the shared stream
q.submit([&](sycl::handler &h) { /* ... */ });
```

This interoperability is implemented in the SYCL runtime's backend abstraction layer; the SYCL queue wraps a native stream handle, and `get_native<>` simply exposes that handle.

---

## 50.9 Summary

- Clang compiles OpenCL C/C++ with `-x cl`. Address spaces are enforced through `SemaOpenCL.cpp`'s `checkOpenCLAddressSpaces()`. For SPIR targets, `__constant` maps to addrspace(2), `__global` to addrspace(1), `__local` to addrspace(3) — verified against actual Clang 22.1.3 output.
- SPIR-V is generated either through the in-tree SPIR-V backend (`-target spirv64-unknown-unknown`) or via the external `SPIRV-LLVM-Translator`. The `spir_kernel` (CC 76) and `spir_func` (CC 75) calling conventions carry OpenCL semantics to the emitter.
- OpenMP target offload uses `CGOpenMPRuntime` / `CGOpenMPRuntimeGPU` to outline target regions into `__omp_offloading_<file>_<func>_l<line>` device functions. Map clauses lower to `__tgt_target_data_begin_mapper` calls with `OMP_TGT_MAPTYPE_*` bitmask arguments. `nowait` enables async dispatch via `AsyncInfoTy`. The `OpenMPOpt` pass performs state machine unfolding and dead region elimination.
- SYCL in Clang (Intel DPC++ architecture) extracts `parallel_for` lambda bodies into `spir_kernel` entry points using `SYCLKernelAttr` / `SYCLAttr`. The Kernel Integration Header bridges device-extracted kernel metadata back to the host compilation for type-safe queue submission.
- `clang-offload-bundler` and `clang-offload-packager` embed multiple device images in fat objects. `clang-offload-wrapper` generates `__tgt_register_lib` constructors. `libomptarget`'s plugin architecture (`GenericPluginTy`) selects CUDA, AMDGPU, Level Zero, or CPU plugins at runtime.
- `--offload-arch=native` auto-detects installed GPUs. Multiple architectures can be targeted in a single compilation; `libomptarget` selects the best-matching device image at runtime using compute capability comparison.

---

*Cross-references:*
- [Chapter 48 — Clang as a CUDA Compiler](../part-07-clang-multilang/ch48-clang-as-cuda-compiler.md) — NVPTX offload path and dual-compilation model foundational concepts
- [Chapter 49 — Clang as a HIP Compiler](../part-07-clang-multilang/ch49-clang-as-hip-compiler.md) — AMDGPU backend details and fat-binary bundling shared with OpenMP-AMDGPU
- [Chapter 104 — The SPIR-V Backend](../../part-15-targets/ch104-spirv-backend.md) — in-tree SPIR-V code generation for OpenCL and SYCL

*Reference links:*
- [CGOpenMPRuntime.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGOpenMPRuntime.cpp)
- [CGOpenMPRuntimeGPU.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGOpenMPRuntimeGPU.cpp)
- [SemaOpenCL.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaOpenCL.cpp)
- [SemaSYCL.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaSYCL.cpp)
- [OpenMPOpt.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/IPO/OpenMPOpt.cpp)
- [libomptarget PluginInterface](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/openmp/libomptarget/plugins-nextgen/common/include/PluginInterface.h)
- [SPIRV backend](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/SPIRV/)
- [clang-offload-bundler](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/tools/clang-offload-bundler/ClangOffloadBundler.cpp)
- [AddressSpaces.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/AddressSpaces.h)
- [OpenCL C 3.0 Specification](https://www.khronos.org/registry/OpenCL/specs/3.0-unified/pdf/OpenCL_C.pdf)
- [SYCL 2020 Specification](https://registry.khronos.org/SYCL/specs/sycl-2020/pdf/sycl-2020.pdf)
- [OpenMP 5.2 Target Offloading](https://www.openmp.org/spec-html/5.2/openmpsu116.html)
