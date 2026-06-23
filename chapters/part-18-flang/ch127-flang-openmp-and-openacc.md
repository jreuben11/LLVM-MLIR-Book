# Chapter 127 — Flang OpenMP and OpenACC

*Part XVIII — Flang*

Parallel and accelerator programming are first-class concerns in modern Fortran. The Fortran community adopted OpenMP before most other languages, and OpenACC was developed largely to serve the scientific Fortran community. Flang's support for both standards is built on MLIR's `omp.*` and `acc.*` dialect layers, which give the compiler a clean, well-typed representation of parallel constructs before any runtime-specific lowering occurs. This chapter traces the full path from Fortran source directive to LLVM IR, covering CPU threading, GPU offload, and the semantics of DO CONCURRENT as an emerging parallelism annotation.

---

## Table of Contents

- [127.1 OpenMP in Flang](#1271-openmp-in-flang)
  - [Architecture Overview](#architecture-overview)
  - [Semantic Analysis of OpenMP](#semantic-analysis-of-openmp)
  - [Lowering Entry Point](#lowering-entry-point)
- [127.2 The OpenMP MLIR Dialect](#1272-the-openmp-mlir-dialect)
  - [Parallel and Work-Sharing](#parallel-and-work-sharing)
  - [Synchronization](#synchronization)
  - [Reductions](#reductions)
  - [Atomic Operations](#atomic-operations)
- [127.3 OpenMP GPU Offload via Flang](#1273-openmp-gpu-offload-via-flang)
  - [Target Region Mapping](#target-region-mapping)
  - [Map Clauses](#map-clauses)
  - [OpenMP to LLVM Lowering](#openmp-to-llvm-lowering)
  - [Whole-Device Compilation](#whole-device-compilation)
- [127.4 OpenACC in Flang](#1274-openacc-in-flang)
  - [OpenACC Dialect Overview](#openacc-dialect-overview)
  - [Lowering Entry Point](#lowering-entry-point)
  - [acc Dialect Operations](#acc-dialect-operations)
  - [Data Directives](#data-directives)
  - [Kernels Construct](#kernels-construct)
  - [acc.routine](#accroutine)
  - [OpenACC to LLVM Lowering](#openacc-to-llvm-lowering)
- [127.5 DO CONCURRENT and Parallelism](#1275-do-concurrent-and-parallelism)
  - [Fortran 2008 DO CONCURRENT](#fortran-2008-do-concurrent)
  - [Flang Lowering of DO CONCURRENT](#flang-lowering-of-do-concurrent)
  - [DO CONCURRENT → OpenMP](#do-concurrent-openmp)
  - [DO CONCURRENT → GPU](#do-concurrent-gpu)
  - [LOCALITY Clauses](#locality-clauses)
- [127.6 OpenMP Runtime Integration](#1276-openmp-runtime-integration)
  - [libomp and liboffload](#libomp-and-liboffload)
  - [Thread-Private Variables](#thread-private-variables)
- [127.7 Practical Examples](#1277-practical-examples)
  - [Parallel Reduction](#parallel-reduction)
  - [GPU Target Offload](#gpu-target-offload)
- [MLIR OpenMP GPU Target Offload](#mlir-openmp-gpu-target-offload)
  - [Pre-2024 Design and Its Limitations](#pre-2024-design-and-its-limitations)
  - [2024 Redesign: OpenMP_Clause and TableGen-Driven Clauses](#2024-redesign-openmp_clause-and-tablegen-driven-clauses)
  - [omp.target-to-GPU Lowering Pipeline](#omptarget-to-gpu-lowering-pipeline)
  - [Fortran Pointer and Allocatable Device Mapping](#fortran-pointer-and-allocatable-device-mapping)
  - [MLIR → Flang Integration](#mlir--flang-integration)
  - [Example: Fortran OpenMP Target and Resulting MLIR](#example-fortran-openmp-target-and-resulting-mlir)
- [Chapter Summary](#chapter-summary)

---

## 127.1 OpenMP in Flang

### Architecture Overview

Flang's OpenMP support follows this pipeline:

```
Fortran source with !$omp directives
    │
    ▼  Semantic analysis: check-omp-structure.cpp
    │  (directive legality, clause restrictions, data sharing rules)
    │
    ▼  Lower/OpenMP/OpenMP.cpp: genOpenMPConstruct()
    │  (constructs → omp.* MLIR ops inside the HLFIR/FIR module)
    │
    ▼  OpenMP dialect ops in MLIR IR
    │
    ▼  OpenMPToLLVM conversion pass
    │  (omp.* ops → llvm.* ops + __kmpc_* runtime calls)
    │
    ▼  LLVM IR with libomp calls / GPU kernel launches
```

### Semantic Analysis of OpenMP

Before any lowering, Flang's semantic checker validates OpenMP constructs. The checker ([`flang/lib/Semantics/check-omp-structure.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Semantics/check-omp-structure.cpp)) validates:

- Nesting rules (e.g., `BARRIER` not allowed inside `CRITICAL`)
- Data-sharing defaults and explicit `PRIVATE`/`SHARED`/`FIRSTPRIVATE` clauses
- `REDUCTION` clause type/operator compatibility
- `TARGET` device clause compatibility
- OpenMP 5.1 `INTEROP` and `DISPATCH` construct rules

The checker uses `OmpDirectiveSet` and `OmpClauseSet` bit masks for efficient nesting validation:

```cpp
// from check-omp-structure.cpp:
static const OmpDirectiveSet parallelSet{
    llvm::omp::Directive::OMPD_parallel,
    llvm::omp::Directive::OMPD_parallel_do,
    llvm::omp::Directive::OMPD_parallel_sections, ...};
```

### Lowering Entry Point

The top-level lowering function is `genOpenMPConstruct()` in [`flang/lib/Lower/OpenMP/OpenMP.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Lower/OpenMP/OpenMP.cpp). It dispatches on the `OpenMPConstruct` parse tree variant to individual `gen*()` functions:

```cpp
void genOpenMPConstruct(AbstractConverter &converter,
                        Fortran::lower::pft::Evaluation &eval,
                        const parser::OpenMPConstruct &ompConstruct) {
  std::visit(common::visitors{
    [&](const parser::OpenMPStandaloneConstruct &x) { genOmpStandalone(converter, eval, x); },
    [&](const parser::OpenMPSectionConstruct &x) { genOmpSection(converter, eval, x); },
    [&](const parser::OpenMPLoopConstruct &x) { genOmpLoop(converter, eval, x); },
    [&](const parser::OpenMPBlockConstruct &x) { genOmpBlock(converter, eval, x); },
    [&](const parser::OpenMPAtomicConstruct &x) { genOmpAtomic(converter, eval, x); },
    [&](const parser::OpenMPCriticalConstruct &x) { genOmpCritical(converter, eval, x); },
    ...
  }, ompConstruct.u);
}
```

---

## 127.2 The OpenMP MLIR Dialect

The OpenMP dialect ([`mlir/include/mlir/Dialect/OpenMP/OpenMPOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/OpenMP/OpenMPOps.td)) provides typed MLIR representations for all OpenMP 5.1 constructs.

### Parallel and Work-Sharing

```mlir
// !$omp parallel private(x) shared(y) num_threads(4)
omp.parallel num_threads(%four : i32) private(%x_alloc : !fir.ref<i32> -> %x_priv) {
  // body
  omp.terminator
}

// !$omp do schedule(dynamic, 4)
omp.wsloop schedule(dynamic, chunk_size(%four : i32)) {
  omp.loop_nest (%i) : index = (%lb) to (%ub) step (%step) {
    // loop body
    omp.yield
  }
}

// !$omp sections
omp.sections {
  omp.section {
    // section 1 body
    omp.terminator
  }
  omp.section {
    // section 2 body
    omp.terminator
  }
  omp.terminator
}

// !$omp simd safelen(4)
omp.simd safelen(4 : i64) {
  omp.loop_nest (%i) : index = (%lb) to (%ub) step (%c1) {
    // vectorized body
    omp.yield
  }
}
```

### Synchronization

```mlir
// !$omp barrier
omp.barrier

// !$omp critical(name)
omp.critical(%critical_name) {
  // critical section body
  omp.terminator
}

// !$omp taskwait
omp.taskwait

// !$omp single
omp.single {
  // single-thread body
  omp.terminator
}
```

### Reductions

```mlir
// !$omp parallel reduction(+:s)
omp.parallel reduction(byref @add_f64_reduction %s_ref -> %s_priv : !fir.ref<f64>) {
  // body uses %s_priv; result is reduced at barrier
  omp.terminator
}

// Reduction declaration (generated for each reduction variable+operator):
omp.declare_reduction @add_f64_reduction : f64
  init { ^bb0(%initial : f64): omp.yield %zero : f64 }
  combiner { ^bb0(%lhs : f64, %rhs : f64):
    %sum = arith.addf %lhs, %rhs : f64
    omp.yield %sum : f64 }
```

### Atomic Operations

```mlir
// !$omp atomic update: x = x + delta
omp.atomic.update %x_ref : !fir.ref<f32> {
^bb0(%old : f32):
  %new = arith.addf %old, %delta : f32
  omp.yield(%new : f32)
}

// !$omp atomic capture
omp.atomic.capture {
  omp.atomic.read %captured = %x_ref : !fir.ref<i32>, i32
  omp.atomic.update %x_ref : !fir.ref<i32> {
  ^bb0(%old : i32):
    %new = arith.addi %old, %c1 : i32
    omp.yield(%new : i32)
  }
}
```

---

## 127.3 OpenMP GPU Offload via Flang

### Target Region Mapping

GPU offload uses the `omp.target` op to represent the device-executed region:

```fortran
!$omp target teams distribute parallel do map(to:a,b) map(from:c)
do i = 1, n
  c(i) = a(i) + b(i)
end do
!$omp end target teams distribute parallel do
```

```mlir
omp.target map_entries(%a_map : !fir.box<!fir.array<?xf64>> -> %a_dev,
                        %b_map : !fir.box<!fir.array<?xf64>> -> %b_dev,
                        %c_map : !fir.box<!fir.array<?xf64>> -> %c_dev) {
  omp.teams {
    omp.distribute {
      omp.loop_nest (%i) : index = (%c1) to (%n) step (%c1) {
        // kernel body: compute c_dev[i] = a_dev[i] + b_dev[i]
        omp.yield
      }
      omp.terminator
    }
    omp.terminator
  }
  omp.terminator
}
```

### Map Clauses

The `map_entries` on `omp.target` carry the data movement semantics:

| Map Type | Fortran | Meaning |
|----------|---------|---------|
| `map(to:x)` | Copy host→device at target entry | Input only |
| `map(from:x)` | Copy device→host at target exit | Output only |
| `map(tofrom:x)` | Both directions | In/out |
| `map(alloc:x)` | Allocate on device only | Scratch |

### OpenMP to LLVM Lowering

The `OpenMPToLLVM` conversion pass ([`mlir/lib/Target/LLVMIR/Dialect/OpenMP/OpenMPToLLVMIRTranslation.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Target/LLVMIR/Dialect/OpenMP/OpenMPToLLVMIRTranslation.cpp)) translates:

- `omp.parallel` → `__kmpc_fork_call()` with an outlined parallel function
- `omp.wsloop` → `__kmpc_for_static_init_*()` + `__kmpc_for_static_fini()`
- `omp.barrier` → `__kmpc_barrier()`
- `omp.atomic.*` → LLVM atomic instructions or `__kmpc_atomic_*`
- `omp.target` → GPU kernel extraction (outlined to a separate module for NVPTX/AMDGPU) + `__tgt_target_teams_mapper()` host call

The GPU kernel extraction is handled by the `LowerToLLVM` pipeline which, for `nvptx64-nvidia-cuda` targets, feeds the outlined kernel function to the NVPTX backend (see [Chapter 102](../part-15-targets/ch102-nvptx-and-the-cuda-path.md)).

### Whole-Device Compilation

For full GPU compilation, Flang integrates with the **liboffload** framework (part of the OpenMP runtime rework in LLVM 18+). The device code path:

```bash
flang-new -fopenmp -fopenmp-targets=nvptx64-nvidia-cuda \
          -Xopenmp-target -march=sm_90 \
          -o kernel.o vadd_omp.f90
```

Internally, this generates two LLVM IR modules: one for the host (with `__tgt_*` calls) and one for the device (the outlined kernel), then links them via the offload linker wrapper.

---

## 127.4 OpenACC in Flang

### OpenACC Dialect Overview

OpenACC support uses the MLIR `acc` dialect ([`mlir/include/mlir/Dialect/OpenACC/OpenACCOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/OpenACC/OpenACCOps.td)). The structure mirrors the OpenMP dialect but follows OpenACC 3.x semantics.

### Lowering Entry Point

[`flang/lib/Lower/OpenACC.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Lower/OpenACC.cpp) is analogous to `OpenMP.cpp`. Its entry point is `genOpenACCConstruct()`.

### acc Dialect Operations

```fortran
! !$acc parallel loop
!$acc parallel loop copyin(a,b) copyout(c)
do i = 1, n
  c(i) = a(i) + b(i)
end do
!$acc end parallel loop
```

```mlir
%a_in = acc.copyin varPtr(%a_ref : !fir.ref<!fir.array<?xf64>>)
        -> !fir.ref<!fir.array<?xf64>>
%b_in = acc.copyin varPtr(%b_ref : !fir.ref<!fir.array<?xf64>>)
        -> !fir.ref<!fir.array<?xf64>>
%c_out = acc.create varPtr(%c_ref : !fir.ref<!fir.array<?xf64>>)
         -> !fir.ref<!fir.array<?xf64>>
acc.parallel dataOperands(%a_in, %b_in, %c_out
             : !fir.ref<!fir.array<?xf64>>,
               !fir.ref<!fir.array<?xf64>>,
               !fir.ref<!fir.array<?xf64>>) {
  acc.loop {
    // generated loop body
    acc.yield
  }
  acc.yield
}
acc.copyout accPtr(%c_out : !fir.ref<!fir.array<?xf64>>)
            to varPtr(%c_ref : !fir.ref<!fir.array<?xf64>>)
            : !fir.ref<!fir.array<?xf64>>
```

### Data Directives

```mlir
// !$acc data copyin(a) copyout(b)
acc.data dataOperands(%a_in, %b_out : ...) {
  // structured block
  acc.terminator
}

// !$acc enter data copyin(a)
acc.enter_data dataOperands(%a_in : ...)

// !$acc exit data copyout(b)
acc.exit_data dataOperands(%b_out : ...)

// !$acc update host(c)
acc.update_host accPtr(%c_dev : ...) to varPtr(%c_host : ...)
```

### Kernels Construct

```fortran
!$acc kernels
do i = 1, n
  b(i) = a(i) * 2.0
end do
!$acc end kernels
```

```mlir
acc.kernels dataOperands(...) {
  // loop nest; the ACC runtime/compiler decides parallelism
  acc.terminator
}
```

### acc.routine

For device subroutines (called from GPU kernels):

```fortran
!$acc routine seq
subroutine device_func(x)
  real :: x
  x = sqrt(x)
end subroutine
```

```mlir
// Function gets acc.routine attribute:
func.func @_QPdevice_func(%arg0: !fir.ref<f32>)
    attributes {acc.routine_info = #acc<routine_info seq>} { ... }
```

### OpenACC to LLVM Lowering

The `ConvertOpenACCToLLVM` pass translates `acc.*` ops to OpenACC runtime calls (`acc_parallel_start`, `acc_data_start`, etc.) or to direct NVPTX/AMDGPU backend paths depending on the compilation mode.

---

## 127.5 DO CONCURRENT and Parallelism

### Fortran 2008 DO CONCURRENT

`DO CONCURRENT` is a Fortran 2008 construct that declares that loop iterations are independent and can execute in any order:

```fortran
real :: a(n,n), b(n,n), c(n,n)
do concurrent (i = 1:n, j = 1:n)
  c(i,j) = 0.0
  do k = 1, n
    c(i,j) = c(i,j) + a(i,k) * b(k,j)
  end do
end do
```

Unlike OpenMP `!$omp parallel do`, `DO CONCURRENT` is a language-level annotation — no directive needed.

### Flang Lowering of DO CONCURRENT

Flang supports three parallelism modes for `DO CONCURRENT` via the `-fdo-concurrent-parallel=` driver flag:

| Flag Value | Behavior |
|------------|----------|
| `none` (default) | Lower to sequential `fir.do_loop` |
| `host` | Lower to `omp.parallel + omp.wsloop` |
| `device` | Lower to `omp.target + omp.teams + omp.distribute` |

When `host` is selected, the DO CONCURRENT body is analyzed by `DoLoopHelper` for reductions and local variables, and the appropriate `omp.reduction` clauses are generated automatically.

### DO CONCURRENT → OpenMP

```mlir
// DO CONCURRENT with -fdo-concurrent-parallel=host:
omp.parallel {
  omp.wsloop {
    omp.loop_nest (%i, %j) : index = (%c1, %c1) to (%n, %n) step (%c1, %c1) {
      // loop body
      omp.yield
    }
    omp.terminator
  }
  omp.terminator
}
```

### DO CONCURRENT → GPU

```mlir
// DO CONCURRENT with -fdo-concurrent-parallel=device:
omp.target {
  omp.teams {
    omp.distribute {
      omp.loop_nest (%i) : index = (%c1) to (%n) step (%c1) {
        omp.parallel {
          omp.wsloop {
            omp.loop_nest (%j) : index = (%c1) to (%n) step (%c1) {
              // body
              omp.yield
            }
            omp.terminator
          }
          omp.terminator
        }
        omp.yield
      }
      omp.terminator
    }
    omp.terminator
  }
  omp.terminator
}
```

### LOCALITY Clauses

Fortran 2018 added locality clauses to `DO CONCURRENT`:

```fortran
do concurrent (i = 1:n) local(tmp) shared(a) reduce(+:s)
  tmp = expensive_fn(a(i))
  s = s + tmp
end do
```

The `LOCAL`, `LOCAL_INIT`, `SHARED`, and `REDUCE` clauses are validated during semantic analysis ([`flang/lib/Semantics/check-do-forall.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Semantics/check-do-forall.cpp)) and map to `omp.private`/`omp.reduction` annotations when `DO CONCURRENT` is parallelized.

---

## 127.6 OpenMP Runtime Integration

### libomp and liboffload

Flang links against the same OpenMP runtime as Clang:
- **libomp** (`openmp/runtime/`): the CPU threading runtime; implements `__kmpc_*` functions
- **liboffload** (LLVM 18+, `offload/`): the device offload runtime; implements `__tgt_*` functions

The key `__kmpc_*` entry points generated by `OpenMPToLLVM`:

| MLIR op | LLVM IR call |
|---------|-------------|
| `omp.parallel` (fork) | `__kmpc_fork_call(ident, argc, microtask, ...)` |
| `omp.wsloop` (static) | `__kmpc_for_static_init_4/8`, `__kmpc_for_static_fini` |
| `omp.wsloop` (dynamic) | `__kmpc_dispatch_init_*`, `__kmpc_dispatch_next_*` |
| `omp.barrier` | `__kmpc_barrier` |
| `omp.critical` | `__kmpc_critical`, `__kmpc_end_critical` |
| `omp.taskwait` | `__kmpc_omp_taskwait` |
| `omp.single` | `__kmpc_single`, `__kmpc_end_single` |

### Thread-Private Variables

Fortran `THREADPRIVATE` common blocks and module variables require special handling:

```fortran
integer :: x
!$omp threadprivate(x)
```

Flang emits a call to `__kmpc_threadprivate_cached()` to get the thread-local copy of `x` on each access.

---

## 127.7 Practical Examples

### Parallel Reduction

```fortran
! Parallel dot product:
real :: a(n), b(n), s
s = 0.0
!$omp parallel do reduction(+:s)
do i = 1, n
  s = s + a(i) * b(i)
end do
!$omp end parallel do
```

```bash
flang-new -fopenmp -O2 -o dot.o dot.f90
```

Emitted FIR (before OpenMP lowering):
```mlir
omp.parallel reduction(byref @add_f64_reduction %s_ref -> %s_priv : !fir.ref<f64>) {
  omp.wsloop {
    omp.loop_nest (%i) : index = (%c1) to (%n) step (%c1) {
      %ai = fir.array_fetch %a, %i : ...
      %bi = fir.array_fetch %b, %i : ...
      %prod = arith.mulf %ai, %bi : f64
      %s_old = fir.load %s_priv : !fir.ref<f64>
      %s_new = arith.addf %s_old, %prod : f64
      fir.store %s_new to %s_priv : !fir.ref<f64>
      omp.yield
    }
    omp.terminator
  }
  omp.terminator
}
```

### GPU Target Offload

```fortran
! GPU SAXPY:
!$omp target teams distribute parallel do &
!$omp   map(to:x,y,a) map(from:z)
do i = 1, n
  z(i) = a * x(i) + y(i)
end do
```

```bash
flang-new -fopenmp \
  -fopenmp-targets=nvptx64-nvidia-cuda \
  -Xopenmp-target=nvptx64-nvidia-cuda -march=sm_90 \
  -O2 -o saxpy.o saxpy.f90
```

---

## MLIR OpenMP GPU Target Offload

The `omp.target` operation in the OpenMP dialect represents a region of computation to be executed on an accelerator device. GPU offload via OpenMP has grown substantially since LLVM 16; LLVM 22 delivers a unified, TableGen-driven clause representation that replaces the ad-hoc attribute approach used through LLVM 15.

### Pre-2024 Design and Its Limitations

Before the 2024 redesign, `omp.target` and related ops stored their clause data as ad-hoc MLIR attributes attached to the operation: the map clause was a flat `ArrayAttr` of pairs, the `if` clause was an `Optional<Value>`, and the device clause was another optional value. Each backend — NVPTX, AMDGPU, the SPIR-V path — handled these attributes independently in its own lowering pass. Adding a new OpenMP clause (e.g., `uses_allocators`, `has_device_addr`) required touching every lowering pass separately, with no single source of truth for clause legality.

### 2024 Redesign: OpenMP_Clause and TableGen-Driven Clauses

The redesign ([`mlir/include/mlir/Dialect/OpenMP/OpenMPClauses.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/OpenMP/OpenMPClauses.td)) introduces an `OpenMP_Clause` ODS base class. Each OpenMP clause is a separate ODS-defined entity:

```tablegen
// OpenMPClauses.td (excerpt)
def OpenMP_MapClause : OpenMP_Clause<"map"> {
  let description = "map(type: var) clause for target/target data constructs";
  let arguments = (ins
    Variadic<AnyType>:$map_vars,
    Variadic<AnyType>:$map_types,  // map type per var: to/from/tofrom/alloc/delete
    OptionalAttr<ArrayAttr>:$map_type_modifiers
  );
}

def OpenMP_IfClause : OpenMP_Clause<"if"> {
  let arguments = (ins Optional<I1>:$if_var);
}

def OpenMP_NumTeamsClause : OpenMP_Clause<"num_teams"> {
  let arguments = (ins Optional<Index>:$num_teams_lower,
                       Optional<Index>:$num_teams_upper);
}
```

Ops that support a clause mixin it via ODS `mixin`:

```tablegen
def OpenMP_TargetOp : OpenMP_Op<"target",
    [OpenMP_MapClause, OpenMP_IfClause, OpenMP_DeviceClause,
     OpenMP_NowaitClause, OpenMP_DependClause, ...]> { ... }
```

The benefits are:
- **Uniform lowering**: `ConvertOpenMPToGPUPass` iterates over `map_vars` and `map_types` through the clause interface, regardless of which op carries the clause.
- **Easier new clause addition**: adding a new clause requires only a new `OpenMP_Clause` TableGen record and mixin; no pass-by-pass additions.
- **Verified clause legality**: verifiers are generated from the TableGen spec, catching illegal clause combinations at parse time.

### omp.target-to-GPU Lowering Pipeline

GPU offload proceeds through four stages:

**Stage 1: OpenMP dialect → GPU dialect**

`ConvertOpenMPToGPUPass` ([`mlir/lib/Conversion/OpenMPToGPU/OpenMPToGPU.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Conversion/OpenMPToGPU/OpenMPToGPU.cpp)) lowers `omp.target` + `omp.teams` + `omp.distribute` + `omp.wsloop` into `gpu.launch_func` calls:

```mlir
// Before: OpenMP dialect
omp.target {
  omp.teams num_teams_upper(%c80 : i32) {
    omp.distribute {
      omp.loop_nest (%i) : index = (%c0) to (%n) step (%c1) {
        // kernel body
        omp.yield
      }
      omp.terminator
    }
    omp.terminator
  }
  omp.terminator
}

// After: GPU dialect
gpu.launch_func @kernel_module::@kernel_fn
    blocks in (%c80, %c1, %c1)
    threads in (%c256, %c1, %c1)
    args(%data_ptr : memref<?xf32>, %n : index)
```

**Stage 2: GPU dialect → NVPTX IR or HSACO**

The `gpu.module` containing `gpu.func` ops is lowered to NVPTX IR via `GpuKernelOutliningPass` + `ConvertGpuOpsToNVVMOps`. The NVPTX IR is then compiled to a `.cubin` or `.ptx` file by the NVPTX backend. For AMD targets, the equivalent ROCDL path produces HSACO.

**Stage 3: Host-side runtime calls**

The host side emits calls to the OpenMP offload runtime:

```llvm
; Pseudo-LLVM IR for host side
call void @__tgt_register_lib(%struct.__tgt_bin_desc* @.offload_entries)
call i32 @__tgt_target_teams_mapper(%ident_t* @src_loc,
    i64 %device_id, i8* @kernel_name,
    i32 %num_args, i8** %args_base, i8** %args, i64* %arg_sizes,
    i64* %arg_types, i64 %num_teams, i64 %thread_limit)
```

`__tgt_register_lib` registers the device binary (the compiled `.cubin`) with the offload runtime. `__tgt_target_teams_mapper` performs the actual launch, handling map clause semantics: `to` clauses trigger `omp_target_associate_ptr` before the launch, `from` clauses trigger a copy-back after.

**Stage 4: Linker wrapper**

`clang-linker-wrapper` bundles the host object and device binary, calling `nvlink` (for NVPTX) or `lld` (for AMDGPU) to produce the final linked binary. The device binary is embedded as a section in the host ELF.

### Fortran Pointer and Allocatable Device Mapping

Fortran arrays use a descriptor structure (`fir.box`) containing a base pointer, bounds, and strides. Mapping a Fortran array to the device requires mapping both the descriptor and the underlying data:

```mlir
// Mapping a Fortran assumed-shape array to GPU
omp.target map_entries(
    %arr_desc_ref -> %arr_desc_dev : !fir.ref<!fir.box<!fir.array<?xf64>>>,
    %arr_data_ref -> %arr_data_dev : !fir.ref<!fir.array<?xf64>>
) {
  // kernel uses %arr_data_dev for data access
  omp.terminator
}
```

The `FlattenFortranArrayToOffloadPass` handles nested descriptor flattening: for multi-dimensional assumed-shape arrays, it extracts the `base_addr`, `dim` extents, and `dim` strides from the descriptor and maps each as a separate scalar or pointer, avoiding the need to marshal the full Fortran array descriptor into GPU memory.

Fortran allocatable variables require additional care: `ALLOCATABLE` variables may be null on entry; the pass inserts a runtime check (`associated(arr)`) and conditionally performs the map operation.

### MLIR → Flang Integration

The end-to-end Flang GPU offload compilation path:

```
Fortran source (!$omp target ...)
    │
    ▼  flang/lib/Lower/OpenMP/OpenMP.cpp
    │  genOmpTargetConstruct() → omp.target + map_entries from OpenMP_Clause
    │
    ▼  FIR/HLFIR + OpenMP dialect (in-memory MLIR module)
    │
    ▼  ConvertOpenMPToGPUPass  →  gpu.launch_func + gpu.module
    │
    ▼  GpuKernelOutliningPass  →  outlined gpu.func operations
    │
    ▼  NVPTX/ROCDL lowering + LLVM translation  →  device IR
    │
    ▼  NVPTX backend  →  .ptx / .cubin
    │
    ▼  clang-linker-wrapper  →  final ELF with embedded device binary
```

```bash
# Complete Flang GPU offload compilation
flang-new -fopenmp \
  -fopenmp-targets=nvptx64-nvidia-cuda \
  -Xopenmp-target=nvptx64-nvidia-cuda -march=sm_90a \
  -O2 -o vadd_gpu vadd_omp.f90

# Inspect the generated OpenMP MLIR before GPU lowering:
flang-new -fopenmp \
  -fopenmp-targets=nvptx64-nvidia-cuda \
  -mmlir --mlir-print-ir-after=convert-openmp-to-gpu \
  -O2 vadd_omp.f90 2>gpu_lowered.mlir
```

### Example: Fortran OpenMP Target and Resulting MLIR

```fortran
! vadd_omp.f90
subroutine vadd(a, b, c, n)
  real(8), intent(in)  :: a(n), b(n)
  real(8), intent(out) :: c(n)
  integer, intent(in)  :: n
  integer :: i

  !$omp target teams distribute parallel do &
  !$omp   map(to:a,b) map(from:c)
  do i = 1, n
    c(i) = a(i) + b(i)
  end do
end subroutine
```

The Flang lowering produces (simplified):

```mlir
// After flang/lib/Lower/OpenMP/OpenMP.cpp
func.func @vadd_(%a : !fir.box<!fir.array<?xf64>>,
                  %b : !fir.box<!fir.array<?xf64>>,
                  %c : !fir.box<!fir.array<?xf64>>,
                  %n : !fir.ref<i32>) {
  %a_map = omp.map.info var_ptr(%a : !fir.box<!fir.array<?xf64>>)
      map_clauses(to) capture(ByRef) -> !fir.box<!fir.array<?xf64>>
  %b_map = omp.map.info var_ptr(%b : !fir.box<!fir.array<?xf64>>)
      map_clauses(to) capture(ByRef) -> !fir.box<!fir.array<?xf64>>
  %c_map = omp.map.info var_ptr(%c : !fir.box<!fir.array<?xf64>>)
      map_clauses(from) capture(ByRef) -> !fir.box<!fir.array<?xf64>>

  omp.target map_entries(%a_map -> %a_dev, %b_map -> %b_dev,
                          %c_map -> %c_dev :
                          !fir.box<!fir.array<?xf64>>,
                          !fir.box<!fir.array<?xf64>>,
                          !fir.box<!fir.array<?xf64>>) {
    omp.teams {
      omp.distribute {
        omp.loop_nest (%i) : index = (%c1) to (%n_val) step (%c1) {
          omp.parallel {
            omp.wsloop {
              // inner parallel do
              omp.yield
            }
            omp.terminator
          }
          omp.yield
        }
        omp.terminator
      }
      omp.terminator
    }
    omp.terminator
  }
  return
}
```

After `ConvertOpenMPToGPUPass`, the `omp.target` block is replaced by a `gpu.launch_func` call with the loop body outlined into a `gpu.func` within a `gpu.module`.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **OpenMP 6.0 clause coverage in Flang**: The OpenMP 6.0 specification (ratified November 2025) introduces `omp.assume`, `omp.nothing`, and revised `interop` semantics; Flang patches tracked in the [OpenMP 6.0 Flang RFC on discourse.llvm.org](https://discourse.llvm.org/c/subprojects/flang/) are expected to land in the `main` branch by mid-2026, adding `OpenMP_AssumeClause` and `OpenMP_NothingClause` to `OpenMPClauses.td`.
- **DO CONCURRENT locality clause completeness**: The `-fdo-concurrent-parallel=device` path for `LOCAL_INIT` and `DEFAULT(NONE)` locality clauses remains partially unimplemented as of LLVM 22; several patches (e.g., D157512 follow-ons) targeting `check-do-forall.cpp` and `DoLoopHelper` are in review to complete Fortran 2018 `DO CONCURRENT` compliance.
- **liboffload v2 stabilization**: The `offload/` subtree refactoring (replacing the legacy `openmp/libomptarget/`) continues; LLVM 22 ships a transitional dual-library layout, and the community aims to drop the legacy `libomptarget` path by LLVM 23, consolidating all `__tgt_*` dispatch into the new `liboffload` ABI.
- **OpenACC 3.3 support**: OpenACC 3.3 added `acc.serial` construct semantics and revised `routine` gang/vector/worker directives; the `acc.*` dialect in MLIR requires new `acc.serial` op and updated `acc.routine_info` attribute — active development tracked via the OpenACC MLIR RFC at [https://discourse.llvm.org/t/rfc-openacc-mlir-dialect-updates](https://discourse.llvm.org/t/rfc-openacc-mlir-dialect-updates).

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Flang as primary Fortran GPU compiler for HPC**: As NVIDIA's `nvfortran` and AMD's `amdflang` converge with upstream Flang, the community roadmap targets Flang being the de-facto open-source Fortran GPU compiler for DOE HPC workloads (Frontier, Aurora successors), requiring full OpenMP 5.2 `metadirective` support, `declare variant`, and device-side `allocate` directives in the `omp.*` dialect.
- **Unified offload runtime across Clang and Flang**: The `clang-linker-wrapper` + `liboffload` stack is being unified so that Flang-generated and Clang-generated offload objects can be linked in the same `--offload-arch` build; this requires a common fat-binary format and interoperable `__tgt_offload_entry` descriptors across both frontends.
- **`omp.target` multi-device dispatch (OpenMP 5.1 `device(ancestor:)`)**:  The `ancestor` device selector, allowing target regions to fall back to the host or a parent device, requires new `omp.device_clause` extensions in the MLIR dialect and multi-device scheduling logic in `liboffload`; this is targeted for LLVM 25–26 according to the OpenMP WG roadmap.
- **MLIR `acc.*` to SPIR-V path for Intel GPUs**: Intel's oneAPI deployment of Fortran via SYCL/SPIR-V requires an `acc.*` → `spirv.*` lowering path analogous to the existing NVPTX/ROCDL paths; early prototypes are appearing in Intel's LLVM fork (`intel/llvm`) and are expected to be upstreamed into MLIR by 2027–2028.

### 5-Year Horizon (Long-Term, by ~2031)

- **Fortran 202X `PARALLEL` coarray extensions and OpenMP interoperability**: The ongoing Fortran standardization (WG5/J3 for Fortran 202X) is considering `PARALLEL DO` as a native language construct superseding `DO CONCURRENT`, with direct coarray and `TEAMS`/`CRITICAL` interoperability; Flang would need new parse-tree nodes, semantic checkers, and `omp.*`/`caf.*` dialect interoperability.
- **Whole-program GPU optimization across `omp.target` boundaries**: Current Flang GPU offload outlines each `!$omp target` region independently; future toolchain directions include interprocedural analysis across target regions (e.g., fusing adjacent kernels, hoisting invariant map operations), enabled by `omp.target` being a first-class op in an MLIR module-level analysis pass.
- **Verified OpenMP lowering via Alive2/Vellvm techniques**: Ongoing research (e.g., the OMPVerif and Deductive Verification of Parallel Programs projects) aims to formally verify that `omp.*`-to-`__kmpc_*` lowering preserves sequential consistency for reduction and atomic operations; integration with the LLVM verification infrastructure (see [Chapter 149](../part-24-verified-compilation/ch149-alive2-and-peephole-verification.md)) would enable certified OpenMP lowering for safety-critical HPC applications.

---

## Chapter Summary

- Flang's OpenMP support runs through semantic validation (`check-omp-structure.cpp`), lowering to `omp.*` MLIR dialect ops (`flang/lib/Lower/OpenMP/OpenMP.cpp`), and conversion to `__kmpc_*`/`__tgt_*` runtime calls via `OpenMPToLLVM`
- The `omp.*` dialect provides typed representations for `omp.parallel`, `omp.wsloop`, `omp.simd`, `omp.sections`, `omp.target`, `omp.teams`, `omp.distribute`, `omp.reduction`, and `omp.atomic.*` constructs
- GPU offload via `!$omp target` extracts device kernels into a separate NVPTX/AMDGPU module; host-side code uses `__tgt_target_teams_mapper()` from liboffload
- OpenACC support uses the `acc.*` MLIR dialect; `flang/lib/Lower/OpenACC.cpp` drives the lowering of `!$acc parallel`, `!$acc kernels`, `!$acc data`, and `!$acc routine` constructs
- `DO CONCURRENT` is a Fortran-native parallelism annotation; Flang's `-fdo-concurrent-parallel=host|device` maps it to OpenMP parallel loops or GPU target regions automatically
- Reductions in both OpenMP and DO CONCURRENT are represented as `omp.declare_reduction` operations with init and combiner regions, enabling correct parallel reduction semantics


---

@copyright jreuben11
