# Chapter 137 — Core Dialects

*Part XX — In-Tree Dialects*

The MLIR dialect ecosystem contains dozens of in-tree dialects, but a handful form the indispensable foundation that nearly every compilation pipeline touches. The `builtin`, `func`, `arith`, `math`, `index`, and `cf` dialects together cover the full spectrum from module-level structure to integer arithmetic to low-level control flow. Understanding these six dialects in depth — their operation semantics, type systems, interfaces, and lowering paths — is prerequisite knowledge for working with any MLIR pipeline. This chapter provides that grounding, tracing each dialect from its source definitions through to its lowering targets.

---

## 137.1 The `builtin` Dialect

### 137.1.1 Always-Loaded Status

The `builtin` dialect occupies a unique position in MLIR: it is always implicitly loaded and cannot be unloaded. It provides the root IR constructs that make MLIR an IR at all — without it, there is no module, no type system, and no attribute infrastructure. Its source lives in [`mlir/include/mlir/IR/BuiltinDialect.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/BuiltinDialect.h) and [`mlir/lib/IR/BuiltinDialect.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/IR/BuiltinDialect.cpp).

The namespace for `builtin` ops is intentionally minimal. The original MLIR design placed `ModuleOp` and `FuncOp` both in `builtin`, but `FuncOp` was migrated to the `func` dialect as the system matured, leaving `builtin` holding only the most fundamental structural ops.

### 137.1.2 `builtin.module`

`ModuleOp` is the root container of every MLIR program. It holds a single region with a single block, which in turn contains a flat sequence of operations:

```mlir
builtin.module {
  func.func @foo(%x: i32) -> i32 {
    func.return %x : i32
  }
}
```

Key constraints: the module's body block has no block arguments, and no control flow may enter it from outside. The module's symbol table is exposed through the `SymbolTable` trait — ops inside can be looked up by `@name` symbol reference.

`ModuleOp` implements `OpAsmDialectInterface` to allow bare `module` without the `builtin.` prefix, which is why you virtually never see `builtin.module` in practice, even though that is the canonical form.

The `module` op carries optional `sym_name` (for nested modules), `sym_visibility`, and `dlti.dl_spec` (for data layout) attributes. Nested modules are used in GPU compilation, where a `builtin.module` labeled `#gpu.container_module` contains nested `gpu.module` ops.

### 137.1.3 Builtin Types

All the fundamental MLIR types are defined in the `builtin` dialect, though they are used universally. These include:

| Type class | Examples | Source |
|---|---|---|
| Integer | `i1`, `i8`, `i32`, `i64`, `i256` | `IntegerType` |
| Float | `f16`, `bf16`, `f32`, `f64`, `f80`, `f128`, `tf32` | `FloatType` |
| Index | `index` | `IndexType` |
| Complex | `complex<f32>` | `ComplexType` |
| Tuple | `tuple<i32, f64>` | `TupleType` |
| Vector | `vector<4xf32>`, `vector<[4]xf32>` | `VectorType` |
| Tensor | `tensor<4x4xf32>`, `tensor<?xf32>` | `RankedTensorType`, `UnrankedTensorType` |
| MemRef | `memref<4x4xf32>`, `memref<?xf32>` | `MemRefType`, `UnrankedMemRefType` |
| Function | `(i32, f32) -> i64` | `FunctionType` |
| None | `none` | `NoneType` |
| Opaque | `!dialect.type` | `OpaqueType` |

Integer types in MLIR are signless — `i32` does not encode sign. Signedness is carried by the operation (`arith.divsi` vs `arith.divui`), not the type. This differs from LLVM IR's identical approach and from languages that encode sign in their type systems.

The `index` type represents the platform's native word size and is used for all memory indexing. It is lowered to `i64` on 64-bit targets and `i32` on 32-bit targets.

### 137.1.4 Builtin Attributes

Standard attributes defined in `builtin`:

```mlir
// Integer attributes
%c = arith.constant 42 : i32          // IntegerAttr
%f = arith.constant 3.14 : f32        // FloatAttr

// Array attributes
#attr = [1, 2, 3]                      // ArrayAttr
#dense = dense<[[1.0, 2.0], [3.0, 4.0]]> : tensor<2x2xf32>  // DenseElementsAttr
#sparse = sparse<[[0,0]], [1.0]>       // SparseElementsAttr

// Type attribute
#type = f32                            // TypeAttr

// String attributes
#str = "hello"                         // StringAttr

// Symbol reference
#ref = @func_name                      // FlatSymbolRefAttr
#nested = @module::@func               // SymbolRefAttr (nested)

// Dictionary attribute
#dict = {key = "value", n = 42 : i64} // DictionaryAttr
```

`DenseElementsAttr` is particularly important — it is the in-memory representation for constant tensors and is heavily used in quantization, constant folding, and weight storage for ML models.

### 137.1.5 `UnrealizedConversionCastOp`

This op warrants special attention because it appears in every non-trivial lowering pipeline. When a dialect conversion is in progress and some types have been converted while others have not yet, the conversion framework inserts `unrealized_conversion_cast` ops to bridge the type gap:

```mlir
// After partial lowering of func args from tensor to memref:
%0 = builtin.unrealized_conversion_cast %tensor : tensor<4xf32>
     to memref<4xf32>
```

The op takes one or more values of arbitrary types and produces one or more values of different arbitrary types. It carries no semantic meaning — it is purely a placeholder that must be removed before the IR is valid. The `--reconcile-unrealized-casts` pass handles this: it either folds cast chains (if a cast from A→B is followed by B→A, both cancel), or raises an error if any unrealized casts remain.

Understanding `unrealized_conversion_cast` is essential when debugging partial conversion failures. If your lowering pipeline fails with "unrealized conversion cast not removed," the typical cause is either a missing pattern for some op, or a type conversion roundtrip that does not form an identity.

---

## 137.2 The `func` Dialect

### 137.2.1 Core Operations

The `func` dialect ([`mlir/include/mlir/Dialect/Func/IR/FuncOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Func/IR/FuncOps.td)) provides the standard function abstraction:

```mlir
// Function definition
func.func @compute(%x: f32, %y: f32) -> f32 {
  %sum = arith.addf %x, %y : f32
  func.return %sum : f32
}

// Direct call
%result = func.call @compute(%a, %b) : (f32, f32) -> f32

// Indirect call via function-type value
%fn = arith.constant @compute : (f32, f32) -> f32
%result2 = func.call_indirect %fn(%a, %b) : (f32, f32) -> f32
```

`func.func` has an attached region (the function body) consisting of basic blocks. The first block receives the function arguments. `func.return` terminates the function, returning zero or more values whose types must match the function result types.

`func.call_indirect` accepts a value of function type as its first operand, enabling higher-order programming. However, it is less optimizable than `func.call` since the callee is not statically known.

### 137.2.2 `FunctionOpInterface`

`func.func` implements `FunctionOpInterface` ([`mlir/include/mlir/Interfaces/FunctionInterfaces.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Interfaces/FunctionInterfaces.h)), which provides a uniform API for function-like operations across dialects. Any op implementing this interface:

- Has a body region with block arguments matching function argument types
- Has a `FunctionType` attribute
- Can carry argument and result attributes (`arg_attrs`, `res_attrs`)
- Participates in the inliner and cloner infrastructure

The `CallableOpInterface` companion interface is implemented by any op that can be inlined — `func.func` implements it by providing the region to inline. The inliner infrastructure calls `mlir::inlineRegion()`, which replaces a `func.call` with the callee's body, substituting block arguments for call operands.

### 137.2.3 Symbol Visibility and Linkage

Functions carry a `sym_visibility` attribute controlling their linkage:

| Visibility | Meaning |
|---|---|
| `public` | Externally visible; may be referenced from other modules |
| `private` | Only visible within the current module; equivalent to `static` in C |
| `nested` | Only visible within the enclosing symbol scope (nested modules) |

This maps naturally to LLVM linkage: `public` → `external`, `private` → `internal`. The `--symbol-dce` pass removes all functions with `private` visibility that are not referenced.

Functions may also carry argument attributes:

```mlir
func.func @blas_gemm(
    %A: memref<4x4xf32> {bufferization.writable = false},
    %B: memref<4x4xf32> {bufferization.writable = false},
    %C: memref<4x4xf32> {bufferization.writable = true}
) {
  // ...
}
```

These per-argument attributes are stored in `arg_attrs` as an `ArrayAttr` of `DictionaryAttr`. The bufferization framework reads `bufferization.writable` to determine aliasing constraints.

### 137.2.4 Inlining Pipeline

The `--inline` pass runs the MLIR inliner. It works in conjunction with `InlinerInterface`, which dialects can implement to customize how their ops behave during inlining. The default behavior handles `func.call` + `func.func` pairs:

```bash
mlir-opt --inline input.mlir
```

The inliner is cost-model-driven; by default it inlines functions with at most a configurable threshold of ops. It handles:

- Cloning the callee body
- Remapping block arguments → call operands
- Replacing `func.return` → the call result values
- Updating the call graph

Post-inlining cleanup typically runs `--canonicalize` to fold constants and `--symbol-dce` to remove now-dead private functions.

---

## 137.3 The `arith` Dialect

### 137.3.1 Integer Arithmetic

The `arith` dialect ([`mlir/include/mlir/Dialect/Arith/IR/ArithOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Arith/IR/ArithOps.td)) provides standard integer and floating-point arithmetic. All operations work on scalar types and element-wise on vectors and tensors.

Integer arithmetic:

```mlir
%a = arith.addi %x, %y : i32         // addition
%b = arith.subi %x, %y : i32         // subtraction
%c = arith.muli %x, %y : i32         // multiplication
%d = arith.divsi %x, %y : i32        // signed division
%e = arith.divui %x, %y : i32        // unsigned division
%f = arith.remsi %x, %y : i32        // signed remainder
%g = arith.remui %x, %y : i32        // unsigned remainder
%h = arith.maxsi %x, %y : i32        // signed max
%i = arith.minsi %x, %y : i32        // signed min
%j = arith.maxui %x, %y : i32        // unsigned max
%k = arith.minui %x, %y : i32        // unsigned min
```

Bitwise operations:

```mlir
%and = arith.andi %x, %y : i32
%or  = arith.ori  %x, %y : i32
%xor = arith.xori %x, %y : i32
%shl = arith.shli %x, %y : i32       // shift left
%srs = arith.shrsi %x, %y : i32      // arithmetic shift right
%sru = arith.shrui %x, %y : i32      // logical shift right
```

### 137.3.2 Overflow Flags

Starting with LLVM 16 (and fully in 22), `arith.addi`, `arith.subi`, and `arith.muli` support overflow flags:

```mlir
%sum = arith.addi %x, %y overflow<nsw> : i32
%prod = arith.muli %x, %y overflow<nsw, nuw> : i32
```

The flags are:
- `nsw` (No Signed Wrap): result does not overflow in signed arithmetic
- `nuw` (No Unsigned Wrap): result does not overflow in unsigned arithmetic

These map directly to LLVM IR's `nsw`/`nuw` flags and enable the same strength-reduction optimizations (e.g., induction variable analysis). If the overflow occurs despite the flag, behavior is undefined — this is the MLIR counterpart of LLVM's poison semantics.

### 137.3.3 Floating-Point Arithmetic

Float operations follow IEEE 754:

```mlir
%a = arith.addf %x, %y : f32
%b = arith.subf %x, %y : f32
%c = arith.mulf %x, %y : f32
%d = arith.divf %x, %y : f32
%e = arith.remf %x, %y : f32         // floating-point remainder
%f = arith.negf %x : f32             // negate
%g = arith.absf %x : f32             // absolute value
%h = arith.maximumf %x, %y : f32     // IEEE maximum (propagates NaN)
%i = arith.minimumf %x, %y : f32     // IEEE minimum (propagates NaN)
%j = arith.maxnumf %x, %y : f32      // returns non-NaN if one input is NaN
%k = arith.minnumf %x, %y : f32      // ditto
```

Fast-math flags are supported via `FastMathFlagsAttr`:

```mlir
%r = arith.addf %x, %y fastmath<reassoc,nnan> : f32
```

Available flags: `nnan`, `ninf`, `nsz`, `arcp`, `contract`, `afn`, `reassoc`, `fast` (all of the above). These map directly to LLVM IR fast-math flags.

### 137.3.4 Comparison Operations

Integer comparison with 6 predicates:

```mlir
%eq  = arith.cmpi eq,  %x, %y : i32  // equal
%ne  = arith.cmpi ne,  %x, %y : i32  // not equal
%slt = arith.cmpi slt, %x, %y : i32  // signed less than
%sle = arith.cmpi sle, %x, %y : i32  // signed less or equal
%sgt = arith.cmpi sgt, %x, %y : i32  // signed greater than
%sge = arith.cmpi sge, %x, %y : i32  // signed greater or equal
%ult = arith.cmpi ult, %x, %y : i32  // unsigned less than
// ... ule, ugt, uge
```

Result is always `i1`. For vectors, result is `vector<Nxi1>`.

Float comparison with 15 predicates (6 ordered + 6 unordered + `true`/`false`/`ord`/`uno`):

```mlir
%lt   = arith.cmpf olt, %x, %y : f32   // ordered less than (false if NaN)
%lt_u = arith.cmpf ult, %x, %y : f32   // unordered less than (true if NaN)
%nan  = arith.cmpf uno, %x, %y : f32   // true if either is NaN
%ord  = arith.cmpf ord, %x, %y : f32   // true if neither is NaN
```

### 137.3.5 Type Casts

The cast operations form a complete set for converting between numeric types:

```mlir
// Integer widening
%ext_s = arith.extsi %i8val : i8 to i32    // sign-extend
%ext_u = arith.extui %i8val : i8 to i32    // zero-extend

// Integer narrowing
%trunc = arith.trunci %i32val : i32 to i8  // truncate (low bits)

// Float to integer
%ftoi_s = arith.fptosi %fval : f32 to i32  // float → signed int
%ftoi_u = arith.fptoui %fval : f32 to i32  // float → unsigned int

// Integer to float
%itof_s = arith.sitofp %ival : i32 to f32  // signed int → float
%itof_u = arith.uitofp %ival : i32 to f32  // unsigned int → float

// Float precision change
%ext_f = arith.extf  %f32val : f32 to f64  // extend float
%trunc_f = arith.truncf %f64val : f64 to f32  // truncate float

// Bitcast (reinterpret bits, no conversion)
%bits = arith.bitcast %fval : f32 to i32
```

The `arith.bitcast` corresponds to LLVM's `bitcast` — it reinterprets the bit pattern without conversion. This is commonly used for bit manipulation on floats and for implementing low-level numeric operations.

### 137.3.6 Constants and Vectorization

```mlir
%c42    = arith.constant 42 : i32
%c3_14  = arith.constant 3.14 : f32
%ctrue  = arith.constant true              // i1 value 1
%cfalse = arith.constant false             // i1 value 0
```

All `arith` operations work element-wise on vector types automatically:

```mlir
%va = arith.constant dense<[1.0, 2.0, 3.0, 4.0]> : vector<4xf32>
%vb = arith.constant dense<[5.0, 6.0, 7.0, 8.0]> : vector<4xf32>
%vc = arith.addf %va, %vb : vector<4xf32>  // [6.0, 8.0, 10.0, 12.0]
```

When tensors carry `arith` operations, the intended semantics is element-wise too, though the actual iteration requires going through `linalg.map` or vectorization passes.

---

## 137.4 The `math` Dialect

### 137.4.1 Transcendental Functions

The `math` dialect ([`mlir/include/mlir/Dialect/Math/IR/MathOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Math/IR/MathOps.td)) provides transcendental and special mathematical functions. All operations support scalar floats, vectors of floats, and tensors of floats.

| Op | Description | C equivalent |
|---|---|---|
| `math.sqrt` | Square root | `sqrt` |
| `math.rsqrt` | Reciprocal square root | `1/sqrt` |
| `math.cbrt` | Cube root | `cbrt` |
| `math.exp` | e^x | `exp` |
| `math.exp2` | 2^x | `exp2` |
| `math.expm1` | e^x - 1 | `expm1` |
| `math.log` | Natural logarithm | `log` |
| `math.log2` | Base-2 logarithm | `log2` |
| `math.log10` | Base-10 logarithm | `log10` |
| `math.log1p` | log(1+x) | `log1p` |
| `math.sin` | Sine | `sin` |
| `math.cos` | Cosine | `cos` |
| `math.tan` | Tangent | `tan` |
| `math.asin` | Arc sine | `asin` |
| `math.acos` | Arc cosine | `acos` |
| `math.atan` | Arc tangent (1 arg) | `atan` |
| `math.atan2` | Arc tangent (2 args) | `atan2` |
| `math.sinh` | Hyperbolic sine | `sinh` |
| `math.cosh` | Hyperbolic cosine | `cosh` |
| `math.tanh` | Hyperbolic tangent | `tanh` |
| `math.erf` | Error function | `erf` |
| `math.erfc` | Complementary error function | `erfc` |
| `math.pow` | x^y (float) | `pow` |
| `math.ipowi` | x^y (integer exponent) | — |
| `math.fpowi` | x^y (float base, int exp) | — |
| `math.fma` | Fused multiply-add: a*b+c | `fma` |
| `math.floor` | Floor | `floor` |
| `math.ceil` | Ceiling | `ceil` |
| `math.round` | Round to nearest | `round` |
| `math.roundeven` | Round to nearest even | `roundeven` |
| `math.trunc` | Truncate toward zero | `trunc` |

### 137.4.2 Lowering Paths

The `math` dialect has three primary lowering paths, each with different tradeoffs:

**To libm** (`--convert-math-to-libm`): Emits calls to the C standard math library. Accurate to IEEE 754, but requires libm linkage and has function call overhead. Suitable for host-side execution.

**To LLVM intrinsics** (`--convert-math-to-llvm`): Maps to LLVM's `llvm.sqrt`, `llvm.sin`, `llvm.cos`, `llvm.exp`, `llvm.log`, etc. intrinsics. The backend then lowers these to hardware instructions or libm calls depending on the target. Preferred for performance-critical host code.

**To polynomial approximations** (via `--convert-math-to-approximations`): Expands transcendentals into polynomial approximations using `arith` and `vector` ops. Much faster on GPU and SIMD targets where libm calls are impossible. Accuracy is typically within 1–2 ULP.

```bash
# Full lowering pipeline for GPU math:
mlir-opt --convert-math-to-approximations \
         --convert-arith-to-llvm \
         --convert-func-to-llvm \
         input.mlir
```

### 137.4.3 Vector Math

All `math` ops apply element-wise to vectors:

```mlir
%vals = arith.constant dense<[0.0, 0.5, 1.0, 1.5]> : vector<4xf32>
%sins = math.sin %vals : vector<4xf32>
%coss = math.cos %vals : vector<4xf32>
```

When targeting GPU, the polynomial approximation lowering is crucial — GPU kernels cannot call libm and LLVM intrinsics lower to expensive device library calls. The approximation path generates pure arithmetic ops that vectorize efficiently.

---

## 137.5 The `index` Dialect

### 137.5.1 Purpose

The `index` dialect ([`mlir/include/mlir/Dialect/Index/IR/IndexOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Index/IR/IndexOps.td)) provides arithmetic on the `index` type — MLIR's platform-independent word-size type. Its core purpose is to keep loop bounds, array dimensions, and memory offsets in a target-neutral form until lowering.

Using `arith.addi %x, %y : i64` hardcodes 64-bit semantics. Using `index.add %x, %y : index` is correct for both 32-bit and 64-bit targets.

### 137.5.2 Operations

```mlir
%a = index.add  %x, %y                // addition
%b = index.sub  %x, %y                // subtraction
%c = index.mul  %x, %y                // multiplication
%d = index.divs %x, %y                // signed division
%e = index.divu %x, %y                // unsigned division
%f = index.rems %x, %y                // signed remainder
%g = index.remu %x, %y                // unsigned remainder
%h = index.ceildivs %x, %y            // ceiling signed division
%i = index.ceildivu %x, %y            // ceiling unsigned division
%j = index.floordivs %x, %y           // floor signed division
%k = index.maxs %x, %y                // signed max
%l = index.maxu %x, %y                // unsigned max
%m = index.mins %x, %y                // signed min
%n = index.minu %x, %y                // unsigned min
%o = index.shl  %x, %y                // shift left
%p = index.shrs %x, %y                // arithmetic shift right
%q = index.shru %x, %y                // logical shift right
%r = index.and  %x, %y                // bitwise and
%s = index.or   %x, %y                // bitwise or
%t = index.xor  %x, %y                // bitwise xor
```

Comparison returns `i1`:

```mlir
%eq = index.cmp eq(%x, %y)
%lt = index.cmp slt(%x, %y)
%lt_u = index.cmp ult(%x, %y)
```

### 137.5.3 Type Conversion

```mlir
%idx = index.casts %i32val : i32 to index   // signed cast from integer
%idx2 = index.castu %u32val : i32 to index  // unsigned cast
%ival = index.casts %idx : index to i64     // cast to integer
```

The `casts`/`castu` distinction matters for 32-bit targets: if `index` is 32 bits and you are casting from `i64`, the upper bits will be truncated regardless. On 64-bit targets, both are equivalent to zero-extend or sign-extend.

### 137.5.4 Special Ops

```mlir
%sz = index.sizeof index           // returns the size of index in bytes (as index)
%b  = index.bool_to_index %pred    // convert i1 to index (0 or 1)
%c  = index.constant 42            // index constant (simpler than arith.constant)
```

`index.sizeof` is evaluated at compile time and produces a constant. It is useful for computing offsets in data structures that mix `index` with other types.

### 137.5.5 Lowering

```bash
mlir-opt --convert-index-to-llvm input.mlir
```

`--convert-index-to-llvm` replaces `index` with `i32` or `i64` depending on the target data layout, and replaces `index.*` ops with corresponding `arith.*` ops on the chosen integer type. The data layout information comes from either the module's `dlti.dl_spec` or the target triple.

---

## 137.6 The `cf` Dialect (Control Flow)

### 137.6.1 Purpose and Position

The `cf` dialect ([`mlir/include/mlir/Dialect/ControlFlow/IR/ControlFlowOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/ControlFlow/IR/ControlFlowOps.td)) provides low-level, unstructured control flow — the direct MLIR analog of LLVM IR's `br`, `condbr`, and `switch` instructions. It sits at the bottom of the control flow hierarchy: `affine` and `scf` both lower into `cf`, which then lowers to `llvm`.

The key difference between `cf` and higher-level dialects is the treatment of block arguments. While `scf.for` and `scf.if` use region-based structure with implicit dominance, `cf` ops explicitly pass values through block arguments in branch instructions, mimicking LLVM's phi nodes in block-argument form.

### 137.6.2 Branch Operations

```mlir
// Unconditional branch, passing values to destination block args
cf.br ^target(%val1 : i32, %val2 : f32)

// Conditional branch
cf.cond_br %cond, ^true_dest(%a : i32), ^false_dest(%b : i32)

// Switch on an integer value
cf.switch %idx : i32, [
  default: ^default_bb,
  42: ^case42(%extra : i64),
  100: ^case100
]
```

Block arguments in `cf` are MLIR's equivalent of LLVM phi nodes. When `cf.br ^target(%val : i32)` is executed, `%val` is delivered to `^target`'s first block argument. Multiple predecessors delivering different values to the same block argument implement the phi-join.

### 137.6.3 `cf.assert`

```mlir
cf.assert %condition, "Assertion message: invariant violated"
```

`cf.assert` is a runtime assertion. It is lowered to a conditional branch to a trap/abort sequence. The pass `--convert-cf-to-llvm` emits the standard pattern:

```llvm
br i1 %cond, label %ok, label %fail
fail:
  call void @llvm.trap()
  unreachable
ok:
  ...
```

In release builds, assertions are typically removed by optimization passes or by setting `NDEBUG`-equivalent flags.

### 137.6.4 Lowering Chain

The standard lowering path for control flow:

```
affine.for / affine.if
    ↓  --convert-affine-to-scf
scf.for / scf.if / scf.while
    ↓  --convert-scf-to-cf
cf.br / cf.cond_br
    ↓  --convert-cf-to-llvm
llvm.br / llvm.cond_br
```

```bash
mlir-opt \
  --convert-affine-to-scf \
  --convert-scf-to-cf \
  --convert-cf-to-llvm \
  input.mlir
```

The `cf` dialect is the convergence point for all structured control flow before entering LLVM dialect territory. It is deliberately minimal — no loops, no conditionals with yields, no structured regions — because all of that structure has already been linearized by the time we reach `cf`.

### 137.6.5 `cf` in Practice: The Block Argument Pattern

Consider an accumulation loop after `--convert-scf-to-cf`:

```mlir
// Original scf.for loop-carried sum
^entry:
  cf.br ^loop_header(%init : f32, %c0 : index)

^loop_header(%acc: f32, %i: index):
  %cond = index.cmp slt(%i, %n)
  cf.cond_br %cond, ^loop_body(%acc, %i), ^loop_exit(%acc)

^loop_body(%acc2: f32, %i2: index):
  %elem = memref.load %arr[%i2] : memref<?xf32>
  %new_acc = arith.addf %acc2, %elem : f32
  %next_i = index.add %i2, %c1
  cf.br ^loop_header(%new_acc : f32, %next_i : index)

^loop_exit(%final: f32):
  func.return %final : f32
```

This pattern — header block with accumulator arguments, body block that updates and branches back, exit block that receives the final value — is the canonical representation of a counted loop in `cf`. Each back-edge carries the updated loop-carried values, replacing what would be phi nodes in traditional SSA form.

---

## Chapter 137 Summary

- The `builtin` dialect is always loaded and provides `ModuleOp`, all built-in types (`IntegerType`, `FloatType`, `VectorType`, `TensorType`, `MemRefType`), and standard attributes including `DenseElementsAttr`. `UnrealizedConversionCastOp` is the bridge op emitted during partial type conversion, removed by `--reconcile-unrealized-casts`.
- The `func` dialect provides `func.func`, `func.call`, `func.call_indirect`, and `func.return`. Functions implement `FunctionOpInterface`, enabling generic inlining via `--inline`. Symbol visibility (`public`/`private`/`nested`) controls linkage.
- The `arith` dialect covers integer and float arithmetic with signless integer types. Operations carry `overflow<nsw,nuw>` flags for integers and `fastmath<...>` flags for floats. All operations work element-wise on vectors and tensors. The cast operations (`extsi`, `extui`, `trunci`, `sitofp`, etc.) form a complete numeric conversion set.
- The `math` dialect provides transcendentals (`sin`, `cos`, `exp`, `log`, `tanh`, `erf`, `pow`, etc.). Lowering paths include `--convert-math-to-libm` (accurate, requires libm), `--convert-math-to-llvm` (uses LLVM intrinsics), and polynomial approximations for GPU.
- The `index` dialect represents platform-word-size arithmetic on the `index` type, keeping loop bounds and array sizes target-neutral until `--convert-index-to-llvm` selects `i32` or `i64` based on data layout.
- The `cf` dialect provides low-level unstructured control flow (`cf.br`, `cf.cond_br`, `cf.switch`, `cf.assert`). Block arguments carry values across branches in place of phi nodes. It is the lowering target for both `affine` and `scf`, and the source for `--convert-cf-to-llvm`.
