# Chapter 126 — The HLFIR and FIR Dialects

*Part XVIII — Flang*

Fortran's data model is fundamentally richer than C's at the IR level. Array sections, assumed-shape descriptors, allocatable variables, WHERE/FORALL constructs, and a zoo of elemental intrinsics all need to be represented faithfully before they can be lowered to machine code. Flang addresses this with a two-level MLIR dialect stack: **HLFIR** (High-Level FIR) retains Fortran semantic concepts — array operations, character lengths, intrinsics — and **FIR** (Fortran IR) provides the lower-level view with explicit memory descriptors, runtime calls, and loop nests. This chapter dissects both dialects, their types, their key operations, and the lowering pipeline that bridges them to the LLVM backend.

---

## 126.1 The Fortran IR Hierarchy

Fortran requires representation at multiple levels of abstraction:

```
Fortran source
    │
    ▼  HLFIR — high-level Fortran semantics preserved
    │         (array expressions, intrinsics, character lengths)
    │
    ▼  FIR — explicit boxing, runtime calls, loop nests
    │        (fir.box<T>, fir.array_load, fir.do_loop)
    │
    ▼  LLVM dialect — LLVM types and intrinsics
    │                 (llvm.ptr, llvm.call, llvm.struct)
    │
    ▼  LLVM IR — standard LLVM representation
    │
    ▼  LLVM backend → target machine code
```

This multi-level approach follows MLIR's core philosophy of progressive lowering ([Chapter 129](../part-19-mlir-foundations/ch129-mlir-philosophy.md)). By keeping high-level information in HLFIR longer, the compiler can perform optimizations (array expression fusion, intrinsic specialization) that would be impossible after explicit loop generation.

The dialect sources:
- FIR: [`flang/include/flang/Optimizer/Dialect/FIROps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/include/flang/Optimizer/Dialect/FIROps.td)
- HLFIR: [`flang/include/flang/Optimizer/HLFIR/HLFIROps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/include/flang/Optimizer/HLFIR/HLFIROps.td)

---

## 126.2 FIR Types

FIR extends MLIR's type system with Fortran-specific constructs. All FIR types are prefixed `fir.` in textual MLIR.

### Reference and Pointer Types

```
fir.ref<T>     -- Fortran INTENT variable, local variable (like C T*)
fir.ptr<T>     -- Fortran POINTER attribute variable (pointer semantics)
fir.heap<T>    -- Fortran ALLOCATABLE attribute variable (heap storage)
```

The distinction matters: `fir.ptr<T>` and `fir.heap<T>` imply possible aliasing patterns distinct from `fir.ref<T>`, which allows stronger alias analysis.

### Array Types

```
fir.array<10x20xi32>   -- rank-2 static array of 32-bit integers
fir.array<?xf64>       -- rank-1 array with unknown (dynamic) extent
fir.array<*:f32>       -- assumed-rank array (Fortran 2018)
```

### The Box Type

`fir.box<T>` is the Fortran **array descriptor** — the most important FIR type. It represents a Fortran assumed-shape, assumed-size, allocatable, or pointer array:

```
fir.box<fir.array<?xf64>>      -- assumed-shape rank-1 REAL(8) array
fir.box<fir.array<?x?xf32>>    -- assumed-shape rank-2 REAL(4) array
fir.box<!fir.heap<fir.array<?xi32>>>  -- allocatable INTEGER array
fir.box<!fir.ptr<f32>>         -- Fortran scalar POINTER
fir.class<T>                   -- polymorphic CLASS(*) descriptor
```

At runtime, a `fir.box` lowers to a `CFI_cdesc_t`-compatible struct (the C interoperability descriptor from Fortran 2018 §18.5):

```c
struct CFI_cdesc_t {
  void *base_addr;
  size_t elem_len;
  int version;
  CFI_rank_t rank;
  CFI_attribute_t attribute;
  CFI_type_t type;
  CFI_dim_t dim[rank];   // per-dimension: lower_bound, extent, stride
};
```

### Character Types

```
fir.char<1, 10>    -- CHARACTER(LEN=10, KIND=1) — length-10 ASCII string
fir.char<4, ?>     -- CHARACTER(LEN=*, KIND=4) — assumed-length UCS-4 string
fir.boxchar<1>     -- assumed-length character (descriptor: ptr + length)
```

### Shape and Slice Types

```
fir.shape<rank>        -- describes extents (used with fir.array_load)
fir.shapeshift<rank>   -- describes lower bounds + extents
fir.slice<rank>        -- array section descriptor (triplet subscripts)
```

### Type Table Summary

| FIR Type | Fortran Concept |
|----------|----------------|
| `fir.ref<T>` | Ordinary variable address |
| `fir.ptr<T>` | POINTER variable |
| `fir.heap<T>` | ALLOCATABLE variable |
| `fir.array<NxM:T>` | Static array |
| `fir.array<?xT>` | Dynamic-extent array |
| `fir.box<T>` | Array descriptor (assumed-shape, allocatable, pointer) |
| `fir.class<T>` | Polymorphic descriptor |
| `fir.char<K,L>` | CHARACTER variable |
| `fir.boxchar<K>` | Assumed-length CHARACTER |
| `fir.shape<R>` | Array shape (extents) |
| `fir.shapeshift<R>` | Array shape with lower bounds |
| `fir.slice<R>` | Array section triplet |

---

## 126.3 Key FIR Operations

### Memory Operations

```mlir
// Allocate a local variable:
%x = fir.alloca i32 {uniq_name = "_QFfooEx"}

// Load and store:
%val = fir.load %ref : (!fir.ref<f64>) -> f64
fir.store %val to %ref : !fir.ref<f64>

// Allocate heap storage (ALLOCATE statement):
%heap = fir.allocmem !fir.array<10xi32>
fir.freemem %heap : !fir.heap<!fir.array<10xi32>>
```

### Array Operations

```mlir
// Array load: elevate a reference to an array value for value-semantics operations
%shape = fir.shape %c10 : (index) -> !fir.shape<1>
%arr = fir.array_load %ref(%shape) : (!fir.ref<!fir.array<10xi32>>,
                                       !fir.shape<1>) -> !fir.array<10xi32>

// Array fetch: extract an element from the array value
%elem = fir.array_fetch %arr, %i : (!fir.array<10xi32>, index) -> i32

// Array update: produce a new array value with one element changed
%arr2 = fir.array_update %arr, %new_val, %i
        : (!fir.array<10xi32>, i32, index) -> !fir.array<10xi32>

// Array merge store: write the final array value back to memory
fir.array_merge_store %arr, %arr2 to %ref
        : !fir.array<10xi32>, !fir.array<10xi32>, !fir.ref<!fir.array<10xi32>>
```

The `fir.array_load` / `fir.array_fetch` / `fir.array_update` / `fir.array_merge_store` pattern is the core of Fortran array expression lowering. It ensures that the right-hand side is fully evaluated before any element of the left-hand side is modified — the Fortran array assignment semantics.

### Box Operations

```mlir
// Create a box from a reference and shape:
%box = fir.embox %ref(%shape) : (!fir.ref<!fir.array<10xf64>>,
                                  !fir.shape<1>) -> !fir.box<!fir.array<10xf64>>

// Re-box (change shape or slice view):
%sliced = fir.rebox %box(%new_shape)[%slice]
  : (!fir.box<!fir.array<10xf64>>, !fir.shape<1>, !fir.slice<1>)
    -> !fir.box<!fir.array<5xf64>>

// Unbox: extract fields from a box
%base = fir.box_addr %box : (!fir.box<!fir.array<10xf64>>) -> !fir.ref<!fir.array<10xf64>>
%rank = fir.box_rank %box : (!fir.box<!fir.array<10xf64>>) -> i32
%elem_len = fir.box_elesize %box : (!fir.box<!fir.array<10xf64>>) -> index
```

### Control Flow Operations

```mlir
// Structured do loop (FIR's loop representation before SCF lowering):
%result = fir.do_loop %i = %lb to %ub step %step {
  %elem = fir.array_fetch %arr, %i : (!fir.array<10xi32>, index) -> i32
  %sum = arith.addi %acc, %elem : i32
  fir.result %sum : i32
} : (index, index, index) -> i32

// Conditional:
fir.if %cond {
  // then-region
} else {
  // else-region
}
```

### Call and Dispatch

```mlir
// Direct call:
fir.call @_QPcompute(%arg0, %arg1) : (!fir.ref<f32>, !fir.ref<f32>) -> ()

// Dynamic dispatch for polymorphic (CLASS) objects:
fir.dispatch "binding_name"(%obj : !fir.class<T>) (%arg0) : (!fir.ref<f32>) -> ()
```

### A Complete FIR Example

```fortran
! Source:
subroutine vadd(a, b, c, n)
  integer, intent(in)  :: n
  real,    intent(in)  :: a(n), b(n)
  real,    intent(out) :: c(n)
  c = a + b
end subroutine
```

```mlir
func.func @_QPvadd(%arg0: !fir.ref<!fir.array<?xf32>>,  ! a
                   %arg1: !fir.ref<!fir.array<?xf32>>,  ! b
                   %arg2: !fir.ref<!fir.array<?xf32>>,  ! c
                   %arg3: !fir.ref<i32>) {              ! n
  %n = fir.load %arg3 : (!fir.ref<i32>) -> i32
  %n_idx = fir.convert %n : (i32) -> index
  %shape = fir.shape %n_idx : (index) -> !fir.shape<1>
  %a = fir.array_load %arg0(%shape) : (!fir.ref<!fir.array<?xf32>>,
                                        !fir.shape<1>) -> !fir.array<?xf32>
  %b = fir.array_load %arg1(%shape) : (!fir.ref<!fir.array<?xf32>>,
                                        !fir.shape<1>) -> !fir.array<?xf32>
  %c_init = fir.array_load %arg2(%shape) : (!fir.ref<!fir.array<?xf32>>,
                                             !fir.shape<1>) -> !fir.array<?xf32>
  %c_final = fir.do_loop %i = %c0 to %n_idx_m1 step %c1
      iter_args(%c_curr = %c_init) -> (!fir.array<?xf32>) {
    %ai = fir.array_fetch %a, %i : (!fir.array<?xf32>, index) -> f32
    %bi = fir.array_fetch %b, %i : (!fir.array<?xf32>, index) -> f32
    %sum = arith.addf %ai, %bi : f32
    %c_next = fir.array_update %c_curr, %sum, %i
              : (!fir.array<?xf32>, f32, index) -> !fir.array<?xf32>
    fir.result %c_next : !fir.array<?xf32>
  }
  fir.array_merge_store %c_init, %c_final to %arg2
        : !fir.array<?xf32>, !fir.array<?xf32>, !fir.ref<!fir.array<?xf32>>
  return
}
```

---

## 126.4 HLFIR (High-Level FIR)

HLFIR was introduced in LLVM 16 to address a specific pain point: the lowering of **array expressions** and **intrinsic operations** directly to FIR loops was too complex and prevented high-level optimizations. HLFIR adds a thin layer above FIR that retains Fortran's data-parallel semantics.

### HLFIR Types

HLFIR reuses FIR types but introduces one new notion: the **expression type** `hlfir.expr<T>`. An `hlfir.expr` is an immutable array value (similar to a `tensor` in the core MLIR dialects) — it represents the result of an array expression before it is materialized into memory:

```
hlfir.expr<10xf32>    -- rank-1 static-shape array expression result
hlfir.expr<?xf64>     -- dynamic-shape array expression result
hlfir.expr<!fir.char<1,?>>  -- character array expression
```

### HLFIR Operations

**Variable declaration** is the canonical starting point:

```mlir
// hlfir.declare: creates a named Fortran variable from a reference
%x, %x_ref = hlfir.declare %alloc {uniq_name = "_QFfooEx"}
    : (!fir.ref<f32>) -> (!fir.ref<f32>, !fir.ref<f32>)

// For arrays with shape:
%a, %a_ref = hlfir.declare %arg0(%shape) {uniq_name = "_QFfooEa",
    fortran_attrs = #fir.var_attrs<intent_in>}
    : (!fir.ref<!fir.array<10xf32>>, !fir.shape<1>)
      -> (!fir.box<!fir.array<10xf32>>, !fir.ref<!fir.array<10xf32>>)
```

The two results are the canonical form (a box for assumed-shape, or directly the ref for explicit-shape) and the raw reference.

**Array assignment**:

```mlir
// Simple elemental assignment:
hlfir.assign %expr to %a : hlfir.expr<10xf32>, !fir.box<!fir.array<10xf32>>
```

**Elemental operations**:

```mlir
// Represent an elemental array operation (vectorized kernel):
%result = hlfir.elemental %shape unordered : (!fir.shape<1>) -> hlfir.expr<10xf32> {
^bb0(%i: index):
  %ai = hlfir.apply %a, %i : (!fir.box<!fir.array<10xf32>>, index) -> f32
  %bi = hlfir.apply %b, %i : (!fir.box<!fir.array<10xf32>>, index) -> f32
  %sum = arith.addf %ai, %bi : f32
  hlfir.yield_element %sum : f32
}
```

**Intrinsic array operations** have dedicated HLFIR ops that preserve their high-level meaning:

```mlir
// MATMUL intrinsic:
%c = hlfir.matmul %a, %b : (!fir.box<!fir.array<?x?xf64>>,
                             !fir.box<!fir.array<?x?xf64>>)
                            -> hlfir.expr<?x?xf64>

// SUM reduction:
%s = hlfir.sum %a dim %d : (!fir.box<!fir.array<?xf64>>, i32) -> f64

// MAXVAL reduction:
%m = hlfir.maxval %a : (!fir.box<!fir.array<?xf64>>) -> f64

// TRANSPOSE:
%t = hlfir.transpose %a : (!fir.box<!fir.array<?x?xf32>>)
                          -> hlfir.expr<?x?xf32>

// COUNT:
%n = hlfir.count %mask : (!fir.box<!fir.array<?xl1>>) -> i64

// DOT_PRODUCT:
%dp = hlfir.dot_product %a, %b : (!fir.box<!fir.array<?xf64>>,
                                   !fir.box<!fir.array<?xf64>>) -> f64
```

**Memory management for expressions**:

```mlir
// Destroy a temporary expression (hlfir.expr cleanup):
hlfir.destroy %expr : hlfir.expr<10xf32>
```

---

## 126.5 The HLFIR → FIR Lowering Pipeline

The full lowering pipeline is a sequence of MLIR passes. Each pass transforms the IR one step closer to LLVM IR:

### Pass Sequence

```
HLFIR (output of Lower::Bridge)
    │
    ▼  SimplifyHLFIRIntrinsics
    │  -- inline simple intrinsics (TRANSPOSE for small static arrays, etc.)
    │
    ▼  LowerHLFIROrderedAssignments
    │  -- handle WHERE/FORALL/masked array assignments
    │
    ▼  LowerHLFIRIntrinsics
    │  -- lower hlfir.matmul, hlfir.sum, etc. to runtime calls or loop nests
    │
    ▼  OptimizedBufferization
    │  -- hlfir.elemental → in-place operations where safe
    │
    ▼  BufferizeHLFIR
    │  -- materialize hlfir.expr<T> into fir.box / fir.ref memory buffers
    │
    ▼  ConvertHLFIRtoFIR
    │  -- remaining HLFIR ops → FIR ops
    │
    ▼  FIR (all HLFIR ops gone)
    │
    ▼  CanonicalizeFIR
    │
    ▼  ArrayValueCopy
    │  -- fir.array_load/fetch/update/merge_store → explicit fir.do_loop
    │
    ▼  CharacterConversion
    │  -- fir.char/fir.boxchar → i8 buffers + length passing
    │
    ▼  MemRefDataFlowOpt (FIR's scalar store-forwarding)
    │
    ▼  InlineElementals (simple elemental transformations)
    │
    ▼  TargetRewrite
    │  -- ABI-specific adjustments (struct passing, complex number layout)
    │
    ▼  BoxedProcedurePass
    │  -- function pointers in boxes → table entries
    │
    ▼  ConvertFIRToLLVM
    │  -- fir.* types → LLVM struct/ptr types; fir.* ops → llvm.* ops
    │
    ▼  ConvertFIRMemOpsToLLVM
    │
    ▼  LLVM dialect (standard MLIR LLVM dialect ops)
    │
    ▼  ConvertMLIRToLLVMIR → LLVM IR Module
```

### Key Pass Details

**`SimplifyHLFIRIntrinsics`** ([`flang/lib/Optimizer/HLFIR/Transforms/SimplifyHLFIRIntrinsics.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Optimizer/HLFIR/Transforms/SimplifyHLFIRIntrinsics.cpp)):
- Recognizes patterns like `hlfir.transpose` of a rank-2 array with compile-time-known extents → inline loop
- Recognizes `hlfir.sum` over a constant array → constant folded result

**`LowerHLFIROrderedAssignments`** ([`flang/lib/Optimizer/HLFIR/Transforms/LowerHLFIROrderedAssignments.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Optimizer/HLFIR/Transforms/LowerHLFIROrderedAssignments.cpp)):
- Handles Fortran's WHERE/FORALL constructs which have specific sequencing semantics
- Must respect the two-pass semantics of WHERE (evaluate mask first, then apply assignments)

**`BufferizeHLFIR`**:
- Transforms `hlfir.expr<T>` values to concrete FIR buffers
- Allocates temporary storage and inserts `hlfir.destroy` → `fir.freemem` when the expression goes out of scope

**`ArrayValueCopy`** ([`flang/lib/Optimizer/Transforms/ArrayValueCopy.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Optimizer/Transforms/ArrayValueCopy.cpp)):
- The core array semantics pass: determines when array assignments require a temporary copy (alias analysis)
- If `a = a(n:1:-1)` (reversed self-assignment), a temporary is required; if `b = a + c`, no self-dependence → no copy needed

---

## 126.6 Box Lowering to LLVM Structs

`fir.box<T>` maps to an LLVM struct that is ABI-compatible with `CFI_cdesc_t`:

```mlir
// FIR type:
!fir.box<!fir.array<?x?xf64>>

// → LLVM struct type (rank=2 example):
!llvm.struct<(ptr, i64, i32, i8, i8, i8, i8,
              array<2 x array<3 x i64>>)>
//            ^ base_addr                  ^ dim[2] = {lower, extent, stride}×2
//                  ^ elem_len
//                        ^ version
//                             ^ rank
//                                ^ attribute  ^ type
//                                       ^ reserved
```

The `ConvertFIRToLLVM` pass ([`flang/lib/Optimizer/CodeGen/CodeGen.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Optimizer/CodeGen/CodeGen.cpp)) performs this mapping and generates `llvm.gep`, `llvm.insertvalue`, `llvm.extractvalue` sequences to construct and access box fields.

---

## 126.7 Fortran Runtime Integration from FIR

FIR frequently emits calls to the Fortran runtime library (`flang-rt`). The most common categories:

### I/O Runtime Calls

```mlir
// Generated for: WRITE(*,*) x
%unit = fir.call @_FortranAioBeginExternalListOutput(%c-1_i32, %filename, %lineno)
        : (i32, !llvm.ptr, i32) -> !llvm.ptr
%ok = fir.call @_FortranAioOutputReal64(%unit, %x)
        : (!llvm.ptr, f64) -> i1
fir.call @_FortranAioEndIo(%unit) : (!llvm.ptr) -> i32
```

### Intrinsic Array Runtime Calls

```mlir
// Generated for: MATMUL(a, b) when not inlined:
fir.call @_FortranAMatmul(%result_box, %a_box, %b_box, %src_file, %line)
         : (!fir.box<none>, !fir.box<none>, !fir.box<none>, !llvm.ptr, i32) -> ()

// Generated for: RESHAPE(a, shape):
fir.call @_FortranAReshape(%result_box, %source_box, %shape_box,
                            %pad_box, %order_box)
         : (!fir.box<none>, ...) -> ()
```

### Allocatable Operations

```mlir
// Generated for: ALLOCATE(a(n)):
fir.call @_FortranAAllocatableAllocate(%box, %hasStat, %errMsg, %file, %line)
         : (!fir.ref<!fir.box<none>>, i1, !fir.box<none>, !llvm.ptr, i32) -> i32

// Generated for: DEALLOCATE(a):
fir.call @_FortranAAllocatableDeallocate(%box, %hasStat, %errMsg, %file, %line)
         : (!fir.ref<!fir.box<none>>, i1, !fir.box<none>, !llvm.ptr, i32) -> i32
```

---

## 126.8 Inspecting FIR and HLFIR in Practice

### Emitting HLFIR

```bash
# Emit HLFIR from a Fortran source file using bbc:
bbc --emit-hlfir -o - vadd.f90

# Using flang-new driver:
flang-new -fc1 -emit-hlfir vadd.f90 -o vadd.hlfir
```

### Emitting FIR

```bash
# Emit FIR (after all HLFIR lowering):
bbc --emit-fir -o - vadd.f90

# Or use flang-new:
flang-new -fc1 -emit-fir vadd.f90 -o vadd.fir
```

### Running Individual Passes

```bash
# Run just the HLFIR bufferization pass:
mlir-opt --bufferize-hlfir vadd.hlfir

# Run the full FIR-to-LLVM-IR conversion:
mlir-opt \
  --lower-hlfir-intrinsics \
  --bufferize-hlfir \
  --convert-hlfir-to-fir \
  --array-value-copy \
  --fir-to-llvm-ir \
  vadd.hlfir -o vadd.ll
```

### FileCheck Test Pattern

```fortran
! RUN: bbc --emit-hlfir %s -o - | FileCheck %s
subroutine sum_arr(a, n, result)
  integer, intent(in)  :: n
  real,    intent(in)  :: a(n)
  real,    intent(out) :: result
  result = sum(a)
end subroutine
! CHECK: hlfir.declare
! CHECK: hlfir.sum
! CHECK: hlfir.assign
```

---

## Chapter Summary

- FIR (Fortran IR) is an MLIR dialect providing Fortran-specific types and operations: `fir.ref<T>`, `fir.box<T>` (array descriptor), `fir.array<NxT>`, `fir.char<K,L>`, `fir.heap<T>`, `fir.ptr<T>`
- `fir.box` is the central abstraction for Fortran's assumed-shape, allocatable, and pointer arrays; it lowers to a `CFI_cdesc_t`-compatible LLVM struct containing base pointer, element size, rank, and per-dimension lower bound/extent/stride
- The FIR array value model (`fir.array_load` / `fir.array_fetch` / `fir.array_update` / `fir.array_merge_store`) preserves Fortran's copy-semantics for array assignments, with `ArrayValueCopy` determining when actual temporaries are needed
- HLFIR sits above FIR and retains high-level concepts: `hlfir.declare`, `hlfir.assign`, `hlfir.elemental`, `hlfir.expr<T>`, and dedicated intrinsic ops (`hlfir.matmul`, `hlfir.sum`, `hlfir.transpose`, etc.)
- The lowering pipeline runs: `SimplifyHLFIRIntrinsics` → `LowerHLFIROrderedAssignments` → `LowerHLFIRIntrinsics` → `BufferizeHLFIR` → `ConvertHLFIRtoFIR` → FIR passes → `ConvertFIRToLLVM` → LLVM dialect → LLVM IR
- FIR calls the `flang-rt` runtime library for I/O (`_FortranAio*`), intrinsics (`_FortranAMatmul`), and allocatable management (`_FortranAAllocatableAllocate`)
- `bbc` and `flang-new -fc1 -emit-hlfir/-emit-fir` are the tools for inspecting intermediate representations


---

@copyright jreuben11
