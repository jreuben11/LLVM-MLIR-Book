# Chapter 127 — Flang OpenMP and OpenACC

*Part XVIII — Flang*

Parallel and accelerator programming are first-class concerns in modern Fortran. The Fortran community adopted OpenMP before most other languages, and OpenACC was developed largely to serve the scientific Fortran community. Flang's support for both standards is built on MLIR's `omp.*` and `acc.*` dialect layers, which give the compiler a clean, well-typed representation of parallel constructs before any runtime-specific lowering occurs. This chapter traces the full path from Fortran source directive to LLVM IR, covering CPU threading, GPU offload, and the semantics of DO CONCURRENT as an emerging parallelism annotation.

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

## Chapter Summary

- Flang's OpenMP support runs through semantic validation (`check-omp-structure.cpp`), lowering to `omp.*` MLIR dialect ops (`flang/lib/Lower/OpenMP/OpenMP.cpp`), and conversion to `__kmpc_*`/`__tgt_*` runtime calls via `OpenMPToLLVM`
- The `omp.*` dialect provides typed representations for `omp.parallel`, `omp.wsloop`, `omp.simd`, `omp.sections`, `omp.target`, `omp.teams`, `omp.distribute`, `omp.reduction`, and `omp.atomic.*` constructs
- GPU offload via `!$omp target` extracts device kernels into a separate NVPTX/AMDGPU module; host-side code uses `__tgt_target_teams_mapper()` from liboffload
- OpenACC support uses the `acc.*` MLIR dialect; `flang/lib/Lower/OpenACC.cpp` drives the lowering of `!$acc parallel`, `!$acc kernels`, `!$acc data`, and `!$acc routine` constructs
- `DO CONCURRENT` is a Fortran-native parallelism annotation; Flang's `-fdo-concurrent-parallel=host|device` maps it to OpenMP parallel loops or GPU target regions automatically
- Reductions in both OpenMP and DO CONCURRENT are represented as `omp.declare_reduction` operations with init and combiner regions, enabling correct parallel reduction semantics


---

@copyright jreuben11
