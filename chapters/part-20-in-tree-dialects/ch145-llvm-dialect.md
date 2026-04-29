# Chapter 145 — LLVM Dialect

*Part XX — In-Tree Dialects*

The `llvm` dialect is the terminus of every MLIR lowering pipeline that targets LLVM-based code generation. It represents LLVM IR as MLIR operations — every LLVM instruction, type, attribute, and metadata construct has a corresponding MLIR representation in this dialect. This two-world approach means the full MLIR infrastructure can be applied at the LLVM-IR level: pattern rewriting, the pass manager, dialect conversion, and region-based analyses all work on LLVM-level code before the final `mlir-translate --mlir-to-llvmir` step. This chapter covers the `llvm` dialect's type system, operation inventory, calling convention support, and the conversion pipeline that bridges MLIR's high-level dialects to it.

---

## 145.1 Purpose and Design

### 145.1.1 Why Represent LLVM IR in MLIR

The `llvm` dialect ([`mlir/include/mlir/Dialect/LLVMIR/LLVMDialect.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/LLVMIR/LLVMDialect.td)) exists for two complementary reasons:

**Downward compatibility**: Every other MLIR dialect that wants to generate native code must eventually lower to something that can be translated to `llvm::Module`. The `llvm` dialect is that common target — all dialects lower to it, and then a single `mlir-translate` call produces LLVM IR.

**MLIR infrastructure on LLVM-level code**: Code that is already at LLVM level can benefit from MLIR's pattern rewriting, regions, and structured transformations. The ClangIR work (Part VIII) and certain GPU lowerings generate LLVM-level MLIR directly.

The principle of operation is straightforward: every LLVM instruction corresponds to one `llvm.*` op, every LLVM type to an MLIR type in the `llvm` namespace, and the translation to/from `llvm::Module` is a thin mechanical transform ([`mlir/lib/Target/LLVMIR/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Target/LLVMIR/)).

### 145.1.2 MLIR vs LLVM IR Structural Differences

| Concept | LLVM IR | MLIR `llvm` dialect |
|---|---|---|
| Phi nodes | `%x = phi i32 [%a, %bb1], [%b, %bb2]` | Block arguments with matching preds |
| Regions | Not applicable | Single-region functions |
| Types | Typed pointers (legacy), opaque pointers | `!llvm.ptr` (opaque only, LLVM 17+) |
| Constants | `i32 42` inline | `llvm.mlir.constant 42 : i32` |
| Global vars | `@g = global i32 0` | `llvm.mlir.global @g : i32` |
| Metadata | `!0 = !{i32 1}` | Attributes on ops |
| Attributes | `nounwind nosync` on function | `llvm.func_attrs = {nounwind, nosync}` |

The most important structural difference is phi nodes vs block arguments: MLIR represents data flow through phi-like block arguments, while LLVM IR uses explicit phi nodes. The `mlir-translate` tool converts between the two forms automatically.

---

## 145.2 LLVM Dialect Types

### 145.2.1 Pointer Type

```mlir
!llvm.ptr            // opaque pointer (default, equivalent to LLVM ptr)
!llvm.ptr<2>         // pointer in address space 2
!llvm.ptr<3>         // pointer in address space 3 (CUDA shared memory)
```

Since LLVM 17, typed pointers (`!llvm.ptr<i32>`) are deprecated. The `llvm` dialect follows LLVM's lead — all pointers are opaque. The element type must be tracked separately (via `llvm.getelementptr`'s `rawConstantIndices` and `elem_type` attribute).

### 145.2.2 Aggregate Types

```mlir
// Fixed-size array
!llvm.array<4 x i32>           // [4 x i32]
!llvm.array<4 x !llvm.array<4 x f32>>  // [4 x [4 x float]]

// Struct (identified or literal)
!llvm.struct<(i32, f64)>       // anonymous struct: {i32, double}
!llvm.struct<"Point", (f32, f32)>  // identified struct: %Point = {float, float}
!llvm.struct<"Node", (i32, !llvm.ptr)>  // recursive struct (ptr to self)
!llvm.struct<packed, (i8, i32)>    // packed struct (no padding)

// Function type (for function pointers)
!llvm.func<i32 (i64, f32)>         // int(long, float) function type
!llvm.func<void (i32) vararg>      // variadic function
```

### 145.2.3 Other Types

```mlir
// MLIR vector types become LLVM vector types
vector<4xf32>          // <4 x float>
vector<[4]xf32>        // <vscale x 4 x float> (scalable)

// MLIR integer/float types map directly
i32 → i32, f32 → float, f16 → half, bf16 → bfloat

// Void (for void function returns)
// There is no explicit void type; funcs returning nothing use 0 results

// Token (for coroutine/async tokens)
!llvm.token
```

---

## 145.3 Key LLVM Ops

### 145.3.1 Memory Operations

```mlir
// Alloca (stack allocation)
%ptr = llvm.alloca %size x i32 : (i64) -> !llvm.ptr
%ptr_aligned = llvm.alloca %c1 x f32 {alignment = 16 : i64} : (i64) -> !llvm.ptr

// Load and store
%val = llvm.load %ptr : !llvm.ptr -> i32
%val_aligned = llvm.load %ptr {alignment = 16 : i64} : !llvm.ptr -> f32
%val_volatile = llvm.load %ptr {volatile_ = true} : !llvm.ptr -> i32
llvm.store %val, %ptr : i32, !llvm.ptr
llvm.store volatile %val, %ptr {alignment = 8 : i64} : f64, !llvm.ptr

// GEP (GetElementPointer) — compute element address
%elem_ptr = llvm.getelementptr %base_ptr[%idx] : (!llvm.ptr, i64) -> !llvm.ptr, i32
%struct_field = llvm.getelementptr %base_ptr[%c0, 1]  // struct field 1
    : (!llvm.ptr, i32, i32) -> !llvm.ptr, !llvm.struct<(i32, f64)>
```

GEP is the most complex op. The `elem_type` trailing type parameter tells GEP the pointee type (since pointers are opaque). Array GEPs take one index; struct GEPs take two indices (outer struct navigation + field index, where field index must be a constant).

### 145.3.2 Pointer Arithmetic and Casts

```mlir
// Bitcast: reinterpret pointer (opaque ptr only for pointer-to-pointer)
%new_ptr = llvm.bitcast %ptr : !llvm.ptr to !llvm.ptr<2>  // change address space

// Integer ↔ pointer conversion
%int = llvm.ptrtoint %ptr : !llvm.ptr to i64
%ptr2 = llvm.inttoptr %int : i64 to !llvm.ptr

// Address space cast
%new_ptr2 = llvm.addrspacecast %shared_ptr : !llvm.ptr<3> to !llvm.ptr
```

### 145.3.3 Arithmetic and Logic

```mlir
// Integer
%add = llvm.add %a, %b : i32
%add_nsw = llvm.add %a, %b overflow<nsw> : i32
%sub = llvm.sub %a, %b : i32
%mul = llvm.mul %a, %b : i32
%sdiv = llvm.sdiv %a, %b : i32
%udiv = llvm.udiv %a, %b : i32
%srem = llvm.srem %a, %b : i32
%urem = llvm.urem %a, %b : i32
%shl  = llvm.shl  %a, %b : i32
%ashr = llvm.ashr %a, %b : i32  // arithmetic shift right
%lshr = llvm.lshr %a, %b : i32  // logical shift right
%and  = llvm.and  %a, %b : i32
%or   = llvm.or   %a, %b : i32
%xor  = llvm.xor  %a, %b : i32

// Float
%fadd = llvm.fadd %a, %b : f32
%fsub = llvm.fsub %a, %b : f32
%fmul = llvm.fmul %a, %b : f32
%fdiv = llvm.fdiv %a, %b : f32
%frem = llvm.frem %a, %b : f32
%fneg = llvm.fneg %a : f32

// Comparison
%icmp_eq  = llvm.icmp "eq"  %a, %b : i32
%icmp_slt = llvm.icmp "slt" %a, %b : i32
%fcmp_olt = llvm.fcmp "olt" %a, %b : f32   // ordered less than

// Select (ternary)
%result = llvm.select %cond, %a, %b : i1, i32
```

### 145.3.4 Control Flow

```mlir
// Unconditional branch
llvm.br ^target_block

// Conditional branch
llvm.cond_br %cond, ^true_block, ^false_block

// Switch
llvm.switch %val : i32, [
  default: ^default_bb,
  42: ^case42_bb(%extra : i64),
  100: ^case100_bb
]

// Indirect branch
llvm.indirectbr %addr : !llvm.ptr, [^bb1, ^bb2, ^bb3]

// Return
llvm.return
llvm.return %val : i32

// Unreachable
llvm.unreachable
```

### 145.3.5 Function Calls

```mlir
// Direct call
%result = llvm.call @my_func(%arg0, %arg1) : (i32, f64) -> i32

// Direct call with calling convention
%result2 = llvm.call @my_func(%arg0) fastcc : (i32) -> i32

// Indirect call (via function pointer)
%fn_ptr = llvm.mlir.addressof @my_func : !llvm.ptr
%result3 = llvm.call %fn_ptr(%arg0) : !llvm.func<i32 (i32)>, (i32) -> i32

// Call with attributes
%res = llvm.call @malloc(%size) {no_unwind = true, will_return = true}
    : (i64) -> !llvm.ptr

// Tail call
%tail = llvm.call tail @fast_func(%x) : (f32) -> f32
```

### 145.3.6 Type Conversion Ops

```mlir
// Integer conversion
%ext_s = llvm.sext %i8val : i8 to i32    // sign-extend
%ext_z = llvm.zext %i8val : i8 to i32    // zero-extend
%trunc = llvm.trunc %i32val : i32 to i8

// Float conversion
%fpext   = llvm.fpext %f32val : f32 to f64
%fptrunc = llvm.fptrunc %f64val : f64 to f32

// Integer ↔ float
%sitofp = llvm.sitofp %ival : i32 to f32
%uitofp = llvm.uitofp %ival : i32 to f32
%fptosi = llvm.fptosi %fval : f32 to i32
%fptoui = llvm.fptoui %fval : f32 to i32

// Bitcast (type punning)
%bits = llvm.bitcast %fval : f32 to i32
```

### 145.3.7 Aggregates and Vectors

```mlir
// Extract and insert struct/array elements
%field = llvm.extractvalue %struct[0] : !llvm.struct<(i32, f64)>
%new_struct = llvm.insertvalue %field, %struct[0] : !llvm.struct<(i32, f64)>

// Vector extract/insert
%elem = llvm.extractelement %vec[%idx] : vector<4xf32>
%new_vec = llvm.insertelement %elem, %vec[%idx] : vector<4xf32>

// Vector shuffle
%shuffled = llvm.shufflevector %v1, %v2 [0, 2, 4, 6] : vector<4xf32>
```

### 145.3.8 Intrinsics

```mlir
// Memory intrinsics
llvm.intr.memcpy %dst, %src, %len : !llvm.ptr, !llvm.ptr, i64
llvm.intr.memmove %dst, %src, %len : !llvm.ptr, !llvm.ptr, i64
llvm.intr.memset %dst, %val, %len : !llvm.ptr, i8, i64

// Math intrinsics
%sqrt = llvm.intr.sqrt %x : f32
%fabs = llvm.intr.fabs %x : f32
%ceil = llvm.intr.ceil %x : f32
%floor = llvm.intr.floor %x : f32
%round = llvm.intr.round %x : f32
%fma  = llvm.intr.fma %a, %b, %c : f32
%pow  = llvm.intr.pow %x, %y : f32
%exp  = llvm.intr.exp %x : f32
%exp2 = llvm.intr.exp2 %x : f32
%log  = llvm.intr.log %x : f32
%log2 = llvm.intr.log2 %x : f32
%maxnum = llvm.intr.maxnum %a, %b : f32
%minnum = llvm.intr.minnum %a, %b : f32

// Overflow-checked arithmetic
%result, %overflow = llvm.intr.sadd.with.overflow %a, %b : i32
%result2, %of2 = llvm.intr.smul.with.overflow %a, %b : i32

// Lifetime markers (optimizer hints)
llvm.intr.lifetime.start 8, %ptr : !llvm.ptr
llvm.intr.lifetime.end 8, %ptr : !llvm.ptr

// Stack operations (for dynamic alloca size checking)
%sp = llvm.intr.stacksave : !llvm.ptr
llvm.intr.stackrestore %sp : !llvm.ptr
```

---

## 145.4 Globals and Constants

### 145.4.1 Global Variables

```mlir
// External global (declared, not defined)
llvm.mlir.global external @input : f32

// Internal global with initializer
llvm.mlir.global internal @counter(0 : i32) : i32

// Constant global (read-only)
llvm.mlir.global private constant @str("hello\00")
    {addr_space = 0 : i32} : !llvm.array<6 x i8>

// Global with alignment
llvm.mlir.global internal @aligned_data(0.0 : f32)
    {alignment = 64 : i64} : f32

// Global with linkonce_odr (weak linkage for C++ template instantiations)
llvm.mlir.global linkonce_odr @template_instance(42 : i32) : i32
    {dso_local}
```

### 145.4.2 Constants and Addresses

```mlir
// Numeric constant (needed because LLVM has typed constants)
%c42 = llvm.mlir.constant(42 : i32) : i32
%c3_14 = llvm.mlir.constant(3.14 : f32) : f32
%null = llvm.mlir.null : !llvm.ptr        // null pointer
%undef = llvm.mlir.undef : i32            // LLVM undef
%poison = llvm.mlir.poison : i32          // LLVM poison

// Get address of global
%addr = llvm.mlir.addressof @counter : !llvm.ptr

// Dense vector/array constant
%vec_const = llvm.mlir.constant(dense<[1.0, 2.0, 3.0, 4.0]> : vector<4xf32>) : vector<4xf32>
```

### 145.4.3 Atomic Operations

```mlir
// Atomic read-modify-write
%old = llvm.atomicrmw add %ptr, %val acquire : !llvm.ptr, i32

// Available operations: add, sub, and, nand, or, xor, max, min, umax, umin, xchg, fadd, fsub
%old_max = llvm.atomicrmw umax %ptr, %val seq_cst : !llvm.ptr, i32

// Compare and swap
%result:2 = llvm.cmpxchg %ptr, %expected, %desired acq_rel monotonic
    : !llvm.ptr, i32

// Fence
llvm.fence acquire
llvm.fence seq_cst synchscope<"singlethread">
```

---

## 145.5 Function Definitions and Attributes

### 145.5.1 `llvm.func`

```mlir
// Basic function
llvm.func @add(%x: i32, %y: i32) -> i32 {
  %sum = llvm.add %x, %y : i32
  llvm.return %sum : i32
}

// Function with attributes
llvm.func @hot_path(%x: f32) -> f32
    attributes {passthrough = ["nounwind", "nosync", ["target-features", "+avx2"]]} {
  // ...
}

// Internal linkage
llvm.func internal @helper(%n: i64) -> !llvm.ptr {
  %buf = llvm.alloca %n x i8 : (i64) -> !llvm.ptr
  llvm.return %buf : !llvm.ptr
}

// Variadic function
llvm.func @printf(%fmt: !llvm.ptr, ...) -> i32

// External declaration
llvm.func external @malloc(i64) -> !llvm.ptr attributes {no_unwind}
```

### 145.5.2 Calling Conventions

```mlir
// Standard calling conventions
llvm.func @foo(%x: f32) fastcc -> f32 { ... }

// CConv as attribute
llvm.func @bar(%x: f32) cc<aarch64_sve_vector_pcs> -> vector<[4]xf32> { ... }

// Available CConvs:
// ccc, fastcc, coldcc, cc<10..> (numbered), webkit_jscc, swiftcc,
// aarch64_vector_pcs, aarch64_sve_vector_pcs, amdgpu_kernel, ptx_kernel, ...
```

### 145.5.3 Operand Bundles and Invoke

```mlir
// Invoke (call with exception handling)
%result = llvm.invoke @may_throw(%arg) to ^normal_bb unwind ^landing_pad_bb
    : (i32) -> i32

// Landing pad (exception handler entry)
^landing_pad_bb:
  %exception = llvm.landingpad catch @MyException
      : !llvm.struct<(ptr, i32)>
  // handle exception
```

---

## 145.6 Metadata and Debug Info

### 145.6.1 LLVM Metadata in MLIR

LLVM metadata (TBAA, debug info, loop hints) is represented as attributes on ops:

```mlir
// Loop vectorization hint
llvm.br ^loop_header {llvm.loop = #llvm.loop<vectorize = {enable = true, width = 8}>, ...}

// TBAA (type-based alias analysis)
%val = llvm.load %ptr {tbaa = [#llvm.tbaa_root<"Simple C/C++ TBAA">, ...]}
    : !llvm.ptr -> f32

// Debug location
llvm.store %val, %ptr {dbg_loc = #llvm.di_location<line = 42, column = 3, scope = ...>}
    : f32, !llvm.ptr
```

### 145.6.2 Access Groups and Alias.scope

```mlir
// Mark loop as vectorizable by providing alias analysis info
%val = llvm.load %ptr {access_groups = [#llvm.access_group<id = 0>]}
    : !llvm.ptr -> f32

llvm.store %val, %out_ptr {alias_scope = [#llvm.alias_scope<id = 0>],
    noalias = [#llvm.alias_scope<id = 1>]}
    : f32, !llvm.ptr
```

---

## 145.7 Conversion to and from LLVM Dialect

### 145.7.1 Standard Conversion Passes

The conversion to LLVM dialect is handled by a collection of passes, each covering a source dialect:

```bash
# Convert individual dialects to LLVM
mlir-opt --convert-arith-to-llvm       input.mlir  # arith → llvm
mlir-opt --convert-func-to-llvm        input.mlir  # func → llvm.func
mlir-opt --convert-cf-to-llvm          input.mlir  # cf → llvm.br
mlir-opt --convert-memref-to-llvm      input.mlir  # memref → llvm.alloca/gep
mlir-opt --convert-index-to-llvm       input.mlir  # index → i32 or i64 arithmetic
mlir-opt --convert-math-to-llvm        input.mlir  # math → llvm intrinsics
mlir-opt --convert-complex-to-llvm     input.mlir  # complex → llvm struct
```

A typical complete lowering pipeline:

```bash
mlir-opt \
  --one-shot-bufferize \
  --buffer-deallocation-pipeline \
  --convert-linalg-to-loops \
  --lower-affine \
  --convert-scf-to-cf \
  --convert-arith-to-llvm \
  --convert-cf-to-llvm \
  --convert-func-to-llvm \
  --convert-memref-to-llvm \
  --convert-index-to-llvm \
  --reconcile-unrealized-casts \
  input.mlir
```

### 145.7.2 `LLVMTypeConverter`

The `LLVMTypeConverter` ([`mlir/include/mlir/Conversion/LLVMCommon/TypeConverter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Conversion/LLVMCommon/TypeConverter.h)) handles all MLIR→LLVM type conversions:

```cpp
LLVMTypeConverter typeConverter(&mlirContext);

// Key conversions:
// i32 → i32 (unchanged)
// f32 → f32 (unchanged)
// index → i64 (or i32 on 32-bit)
// memref<4xf32> → { ptr, ptr, i64, i64, i64 }
//   = {allocated_ptr, aligned_ptr, offset, sizes[1], strides[1]}
// memref<?x?xf32> → { ptr, ptr, i64, i64[2], i64[2] } (2D dynamic)
// func<(i32) -> f32> → !llvm.func<f32 (i32)>
```

The `memref` descriptor struct is a critical conversion: every `memref` becomes a struct containing pointers and shape/stride arrays. This is what makes dynamic-shape memrefs work without global state — the descriptor travels alongside the data pointer.

### 145.7.3 `--reconcile-unrealized-casts`

After all conversion passes, any remaining `builtin.unrealized_conversion_cast` ops must be removed:

```bash
mlir-opt --reconcile-unrealized-casts input.mlir
```

This pass:
1. Cancels cast chains: `A → B → A` becomes identity
2. Fails with an error if any non-canceling casts remain

Remaining casts indicate missing conversion patterns. The error message identifies the specific types involved, guiding you to the missing pass or pattern.

### 145.7.4 `mlir-translate --mlir-to-llvmir`

The final step — translating the LLVM dialect to an `llvm::Module`:

```bash
# Produce LLVM IR text
mlir-translate --mlir-to-llvmir input.mlir -o output.ll

# Then use LLVM tools:
opt -O3 output.ll | llc -filetype=obj -o output.o
```

The translator ([`mlir/lib/Target/LLVMIR/ModuleTranslation.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Target/LLVMIR/ModuleTranslation.cpp)):
1. Maps each `llvm.mlir.global` → `llvm::GlobalVariable`
2. Maps each `llvm.func` → `llvm::Function`
3. Maps each `llvm.*` op → the corresponding `llvm::Instruction`
4. Converts MLIR block arguments → LLVM phi nodes
5. Converts MLIR attributes → LLVM metadata

### 145.7.5 Import from LLVM IR

Going the other direction — importing an existing `llvm::Module` into MLIR:

```bash
# Import LLVM IR text into LLVM dialect
mlir-translate --import-llvm output.ll -o mlir_output.mlir
```

Or programmatically:

```cpp
#include "mlir/Target/LLVMIR/Import.h"

llvm::LLVMContext llvmCtx;
auto llvmModule = llvm::parseAssemblyFile("input.ll", llvmCtx);
auto mlirModule = mlir::translateLLVMIRToModule(std::move(llvmModule), &mlirCtx);
```

This enables applying MLIR transformations to legacy LLVM IR, then translating back — useful for experimenting with MLIR passes on existing compiled code.

---

## Chapter 145 Summary

- The `llvm` dialect represents LLVM IR as MLIR operations, enabling MLIR infrastructure (passes, pattern rewriting, analysis) to be applied at LLVM-IR level. Every LLVM instruction is one op; every LLVM type is an MLIR type; the translation to `llvm::Module` is mechanical.
- Types: `!llvm.ptr` (opaque pointer, LLVM 17+ style), `!llvm.array<N x T>`, `!llvm.struct<(T...)>` (named or anonymous), `!llvm.func<ret(args)>`. MLIR vector types (`vector<4xf32>`) map directly to LLVM vectors.
- Memory ops: `llvm.alloca`, `llvm.load`, `llvm.store`, `llvm.getelementptr` (with explicit `elem_type`), `llvm.ptrtoint`/`llvm.inttoptr`, `llvm.bitcast`, `llvm.addrspacecast`.
- Control flow: `llvm.br`, `llvm.cond_br`, `llvm.switch`, `llvm.return`, `llvm.unreachable`, `llvm.invoke`/`llvm.landingpad` for exceptions.
- Global infrastructure: `llvm.mlir.global` for global variables, `llvm.mlir.constant` for typed constants, `llvm.mlir.addressof` for taking addresses of globals, `llvm.mlir.null`/`undef`/`poison`.
- Calling conventions via `CConvAttr` (`fastcc`, `coldcc`, `aarch64_vector_pcs`, `ptx_kernel`, etc.). Atomic ops via `llvm.atomicrmw` and `llvm.cmpxchg`. Intrinsics via `llvm.intr.*` namespace.
- `LLVMTypeConverter` converts `memref` to a descriptor struct containing allocated/aligned pointers, offset, and shape/stride arrays. `--reconcile-unrealized-casts` removes leftover cast ops after partial conversion.
- The lowering pipeline: `--convert-arith-to-llvm`, `--convert-cf-to-llvm`, `--convert-func-to-llvm`, `--convert-memref-to-llvm`, `--convert-index-to-llvm`, then `--reconcile-unrealized-casts`, then `mlir-translate --mlir-to-llvmir`.
- `mlir-translate --import-llvm` enables the reverse: importing `llvm::Module` → LLVM dialect MLIR, allowing MLIR transformations on legacy LLVM IR.


---

@copyright jreuben11
