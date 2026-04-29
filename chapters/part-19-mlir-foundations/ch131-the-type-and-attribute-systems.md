# Chapter 131 — The Type and Attribute Systems

*Part XIX — MLIR Foundations*

MLIR's type and attribute systems are the backbone of its extensibility. Where LLVM IR's type system is closed — every possible type is baked into the IR specification — MLIR's is open: any dialect can introduce new types and attributes that are first-class citizens of the IR, verified by the compiler, serialized to bytecode, and queryable through interfaces. This chapter covers the full architecture: how types and attributes are uniqued and stored, how to define custom types and attributes using ODS, how the built-in container types (memref, tensor, vector) work, the Properties system that replaces the attribute dictionary for op-owned data, and the constraint and interface system that ties it all together.

---

## 131.1 Type System Overview

### Type as a Handle

In MLIR, `mlir::Type` is a pointer-sized value type — a handle around a `TypeStorage *`:

```cpp
// mlir/include/mlir/IR/Types.h:
class Type {
  TypeStorage *impl;  // uniqued storage; nullptr only for null types
public:
  bool operator==(Type other) const { return impl == other.impl; }
  // Since types are uniqued, pointer equality = structural equality
  TypeID getTypeID() const;
  Dialect *getDialect() const;
  MLIRContext *getContext() const;
  void print(raw_ostream &os) const;
  bool isa<ConcreteType>() const;
  ConcreteType dyn_cast<ConcreteType>() const;
};
```

The uniquing guarantee means:
- `type1 == type2` is an `O(1)` pointer comparison
- There is exactly one `TypeStorage` object per distinct type; MLIR manages the type storage pool in the `MLIRContext`
- Types are immutable: once created, their parameters cannot change

### The TypeID Dispatch System

MLIR uses `TypeID` (a pointer to a static data member, one per C++ type) for fast runtime type dispatch:

```cpp
// Checking type kind without virtual dispatch:
if (type.getTypeID() == IntegerType::getTypeID())
  // ...

// Preferred API:
if (auto intTy = dyn_cast<IntegerType>(type))
  unsigned width = intTy.getWidth();
```

---

## 131.2 Builtin Types in Depth

### Integer Types

MLIR integers are **signless**: `i32` does not encode whether the value is signed or unsigned. Signedness is a property of the *operation*, not the type:

```mlir
arith.addi %a, %b : i32      // addition; same for signed and unsigned
arith.divsi %a, %b : i32     // signed division
arith.divui %a, %b : i32     // unsigned division
arith.cmpi slt, %a, %b : i32 // signed less-than comparison
arith.cmpi ult, %a, %b : i32 // unsigned less-than comparison
```

This design avoids the proliferation of signed/unsigned type variants that burdens C type systems and simplifies type inference.

Arbitrary widths are supported: `i1` (boolean), `i8`, `i16`, `i32`, `i64`, `i128`, `i256`, `i1024`, and any other positive integer.

### Float Types

```cpp
// Query float semantics:
auto fTy = dyn_cast<FloatType>(type);
const llvm::fltSemantics &sem = fTy.getFloatSemantics();
unsigned width = fTy.getWidth();

// Specific float types:
FloatType f32 = builder.getF32Type();
FloatType bf16 = FloatType::getBF16(&ctx);
FloatType tf32 = FloatType::getTF32(&ctx);  // 19-bit mantissa, for NVIDIA Tensor Cores
```

### MemRef Type

`MemRefType` is the typed memory reference type — the MLIR equivalent of a typed pointer with known shape:

```mlir
memref<4x4xf32>                    // static shape, default row-major layout
memref<?xf64>                      // dynamic rank-1 (size unknown at compile time)
memref<?x?xf32, strided<[?, 1], offset: ?>>  // strided layout
memref<4x4xf32, 1>                 // in address space 1 (e.g., GPU shared memory)
memref<4x4xf32, affine_map<(d0, d1) -> (d1, d0)>>  // column-major layout
```

```cpp
// Constructing in C++:
MemRefType::get({4, 4}, builder.getF32Type())     // static
MemRefType::get({ShapedType::kDynamic, 4}, f32Ty) // one dynamic dimension
MemRefType::get({}, f64Ty)                         // scalar memref (0D)

// Querying:
MemRefType mrt = cast<MemRefType>(type);
ArrayRef<int64_t> shape = mrt.getShape();   // {4, 4} or {-1, 4}
Type elemTy = mrt.getElementType();
bool isDynamic = ShapedType::isDynamic(shape[0]);
MemRefLayoutAttrInterface layout = mrt.getLayout();
Attribute memSpace = mrt.getMemorySpace();
```

### Tensor Type

`RankedTensorType` and `UnrankedTensorType` provide value-semantics array types:

```mlir
tensor<4x4xf32>   // ranked, static shape
tensor<?xi64>     // ranked, dynamic shape
tensor<*xi32>     // unranked tensor (rank unknown)
```

The distinction from `memref`:
- `tensor` has **value semantics**: it is immutable; operations produce new tensors
- `memref` has **reference semantics**: it is a mutable buffer; operations load/store into it
- Bufferization (see Chapter 138) converts tensors to memrefs as a lowering step

```cpp
RankedTensorType tensorTy = RankedTensorType::get({4, 4}, f32Ty);
int64_t rank = tensorTy.getRank();                   // 2
ArrayRef<int64_t> shape = tensorTy.getShape();       // {4, 4}
bool isStaticDim = !ShapedType::isDynamic(shape[0]); // true
```

### Vector Type

```mlir
vector<4xf32>       // 1D fixed SIMD vector (4 floats)
vector<4x4xf32>     // 2D vector (matrix-like)
vector<[4]xf32>     // 1D scalable vector (SVE/RISC-V V; size is multiple of 4)
vector<[4x4]xf32>   // 2D scalable vector
```

Vector types have a mandatory static (or scalable-multiply) shape. They are distinct from `tensor` in that they represent register-level SIMD values, not high-level data collections.

---

## 131.3 Defining Custom Types with ODS

Types are defined in TableGen using `TypeDef`:

### Minimal Type Definition

```tablegen
// mlir/include/mlir/Dialect/MyDialect/IR/MyDialect.td

def My_PointType : TypeDef<MyDialect, "Point"> {
  let mnemonic = "point";
  let summary = "A 2D integer point";

  let parameters = (ins
    "int64_t":$x,
    "int64_t":$y
  );

  // Textual syntax: !my.point<x, y>
  let assemblyFormat = "`<` $x `,` $y `>`";
}
```

This generates:

```cpp
class PointType : public Type::TypeBase<PointType, Type, PointTypeStorage> {
public:
  static PointType get(MLIRContext *ctx, int64_t x, int64_t y);
  int64_t getX() const;
  int64_t getY() const;
};
```

### Type with Verification

```tablegen
def My_RangeType : TypeDef<MyDialect, "Range"> {
  let mnemonic = "range";
  let summary = "An integer range [low, high)";

  let parameters = (ins
    "int64_t":$low,
    "int64_t":$high
  );

  let assemblyFormat = "`<` $low `..` $high `>`";

  let genVerifyDecl = 1;  // generates verifyConstructionInvariants
}
```

```cpp
// In MyDialect.cpp:
LogicalResult RangeType::verify(
    function_ref<InFlightDiagnostic()> emitError,
    int64_t low, int64_t high) {
  if (low >= high)
    return emitError() << "range must have low < high, got "
                       << low << " >= " << high;
  return success();
}
```

### Type Parameters

`TypeParameter<"C++ type", "description">` supports any C++ type as a parameter:

```tablegen
def My_BufferType : TypeDef<MyDialect, "Buffer"> {
  let parameters = (ins
    TypeParameter<"mlir::Type", "element type">:$elementType,
    TypeParameter<"unsigned", "capacity">:$capacity,
    OptionalParameter<"mlir::Attribute">:$memorySpace
  );
}
```

`OptionalParameter` generates an optional parameter (may be null/default).

---

## 131.4 Attribute System

### Attribute as a Handle

`mlir::Attribute` mirrors `Type`: a pointer-sized handle around a uniqued `AttributeStorage *`. The same uniquing, equality, and type-dispatch APIs apply.

### Dense Element Attributes

`DenseIntOrFPElementsAttr` stores a dense constant tensor — all elements inline in the attribute storage:

```mlir
// Dense constant tensor:
%w = arith.constant dense<[[1.0, 0.0], [0.0, 1.0]]> : tensor<2x2xf32>

// 1D integer:
%v = arith.constant dense<[1, 2, 3, 4]> : tensor<4xi32>

// Splat (all elements same):
%z = arith.constant dense<0.0> : tensor<10xf32>
```

```cpp
DenseIntOrFPElementsAttr denseAttr = cast<DenseIntOrFPElementsAttr>(constOp.getValue());
ShapedType shapedTy = denseAttr.getType();
// Iterate elements:
for (float val : denseAttr.getValues<float>())
  llvm::outs() << val << "\n";

// Check if splat:
if (denseAttr.isSplat())
  float splatVal = denseAttr.getSplatValue<float>();
```

### DenseResourceElementsAttr

For large weight tensors (e.g., neural network parameters), storing them inline in the text format is impractical. `DenseResourceElementsAttr` stores a reference to an external binary blob:

```mlir
%weights = arith.constant dense_resource<weights_blob> : tensor<1024x1024xf32>
// The actual data is in the bytecode external resource section
```

This enables model weights to be embedded in `.mlirbc` files without making the text representation unreadable.

### AffineMapAttr

```mlir
#map = affine_map<(d0, d1) -> (d1, d0)>

// Use as attribute value:
linalg.generic {indexing_maps = [#map, #map]}
```

### SymbolRefAttr

References another globally-named op (function, global variable, etc.):

```mlir
func.call @my_function(%arg) : (i32) -> ()
// "@my_function" is a FlatSymbolRefAttr
```

### Defining Custom Attributes with ODS

```tablegen
def My_DeviceAttr : AttrDef<MyDialect, "Device"> {
  let mnemonic = "device";
  let parameters = (ins
    "DeviceKind":$kind,
    "unsigned":$ordinal
  );
  let assemblyFormat = "`<` $kind `:` $ordinal `>`";
}
```

Custom attributes can carry any C++ data. They appear in the textual IR with the dialect prefix:
```mlir
// An op with a custom attribute:
my.kernel attributes {my.device = #my.device<GPU:0>} { ... }
```

---

## 131.5 Properties (MLIR 17+)

### Motivation

Before MLIR 17, all op-owned data lived in the **attribute dictionary** — a `DictionaryAttr` that maps string names to attributes. This has two downsides:
1. **Access cost**: looking up `"result_type"` requires a string hash lookup at runtime
2. **Type erasure**: the attribute is stored as `Attribute` (generic), not as the concrete C++ type

**Properties** ([`mlir/include/mlir/IR/PropertiesBase.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/PropertiesBase.h)) are op-specific storage separate from the attribute dictionary:

```cpp
// Properties are a struct inlined into the Operation storage:
struct FuncOp::Properties {
  StringAttr sym_name;
  TypeAttr function_type;
  StringAttr sym_visibility;
  UnitAttr is_declaration;
  DictionaryAttr arg_attrs;
  DictionaryAttr res_attrs;
};
```

Access is a direct struct member load — no hash lookup:
```cpp
FuncOp funcOp = ...;
StringAttr name = funcOp.getProperties().sym_name;  // O(1)
```

### ODS Properties Declaration

```tablegen
def MyOp : MyDialect_Op<"compute"> {
  let arguments = (ins I32Attr:$step, I64Attr:$count);
  // With properties (MLIR 17+):
  // The 'let hasProperties = 1;' triggers property storage generation
}
```

ODS generates `getProperties()`, `setStep()`, `getStep()`, etc.

### Serialization of Properties

Properties are serialized to bytecode and round-trip through the textual format. An op with properties prints them in the attribute dictionary position but stores them more efficiently:

```mlir
my.compute {step = 4 : i32, count = 100 : i64}
// ↑ Same textual appearance; internal storage is struct, not DictionaryAttr
```

---

## 131.6 Type Constraints and Interfaces

### ShapedType Interface

`ShapedType` is an interface implemented by `MemRefType`, `RankedTensorType`, `UnrankedTensorType`, and `VectorType`. It provides a common API for shaped containers:

```cpp
ShapedType shaped = cast<ShapedType>(type);
bool isRanked = shaped.hasRank();
int64_t rank = shaped.getRank();
ArrayRef<int64_t> shape = shaped.getShape();
Type elementType = shaped.getElementType();
int64_t numElements = shaped.getNumElements();  // product of static dims
bool hasStaticShape = shaped.hasStaticShape();

// Dimension queries:
bool isDynamic = ShapedType::isDynamic(shape[0]);
int64_t kDynamic = ShapedType::kDynamic;  // = -1
```

### SubElementTypeInterface

For types that contain other types (composite types), `SubElementTypeInterface` provides a protocol for walking and replacing contained types:

```cpp
// Example: a tuple type contains element types
if (auto subElemIface = dyn_cast<SubElementTypeInterface>(type)) {
  subElemIface.walkSubTypes([](Type inner) {
    // Called for each type nested within `type`
  });
}
```

Used by the type converter infrastructure to recursively convert all nested types during dialect conversion.

### ODS Type Constraints

In ODS operation definitions, operand and result types are constrained using `TypeConstraint` values:

```tablegen
// Common constraints:
AnyType          -- any type
I32              -- exactly i32
F32              -- exactly f32
AnyFloat         -- any float type (f16/bf16/f32/f64/f80/f128)
AnyInteger       -- any integer type (any width)
Index            -- index type
AnySignlessInteger       -- any signless integer
SignlessIntegerLike       -- i1..iN
I32OrI64         -- exactly i32 or i64
AnyOf<[I32, F32]> -- i32 or f32
RankedTensorOf<[F32]>    -- tensor<...xf32>
MemRefOf<[AnyFloat]>     -- memref<...xfN>
VectorOfAnyRankOf<[I32]> -- vector<...xi32>
TensorOrMemref           -- tensor or memref of anything
AnyRankedTensor          -- any ranked tensor
AnyUnrankedTensor        -- any unranked tensor
AnyShapedType            -- any shaped type
AnyNonEmptyTuple         -- tuple with at least one element
```

### TypesMatchWith Constraint

For ops where one type must be related to another:

```tablegen
def My_CastOp : My_Op<"cast"> {
  let arguments = (ins AnyType:$input);
  let results = (outs AnyType:$output);
  // input and output must have the same element type (for shaped types):
  let hasVerifier = 1;
}
```

ODS also supports `TypesMatchWith<"description", "$operand", "$result", "getElementType($_self)">` to express element-type matching without writing a custom verifier.

---

## 131.7 Type Converter

The `TypeConverter` class ([`mlir/include/mlir/Transforms/DialectConversion.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Transforms/DialectConversion.h)) is used by dialect conversion passes to specify how types transform between dialects:

```cpp
TypeConverter converter;

// Convert i32 → i64 (widening):
converter.addConversion([](IntegerType ty) -> Type {
  if (ty.getWidth() < 64)
    return IntegerType::get(ty.getContext(), 64);
  return ty;  // already 64-bit; no conversion
});

// Convert memref<T> → !llvm.ptr when T converts to llvm type:
converter.addConversion([&](MemRefType mrt) -> Type {
  Type elemLlvm = converter.convertType(mrt.getElementType());
  return LLVM::LLVMPointerType::get(mrt.getContext());
});

// Fallback: pass-through for types not matched:
converter.addConversion([](Type ty) { return ty; });
```

`TypeConverter` is central to MLIR's dialect conversion infrastructure and is used in every lowering pass.

---

## 131.8 Practical Type System Usage

### Checking and Casting Types

```cpp
Type ty = value.getType();

// Check and cast:
if (ty.isa<IntegerType>())              // check
  auto intTy = ty.cast<IntegerType>();  // cast (asserts on failure)

if (auto intTy = ty.dyn_cast<IntegerType>())  // check + cast
  unsigned width = intTy.getWidth();

// Nested type inspection:
if (auto tensorTy = ty.dyn_cast<RankedTensorType>()) {
  if (auto floatElem = tensorTy.getElementType().dyn_cast<FloatType>()) {
    unsigned floatWidth = floatElem.getWidth();
  }
}
```

### Building Types

```cpp
MLIRContext &ctx = ...;
OpBuilder builder(&ctx);

// Scalar types:
Type i32 = builder.getI32Type();
Type f64 = builder.getF64Type();
Type idx = builder.getIndexType();

// Container types:
Type tensor4x4f32 = RankedTensorType::get({4, 4}, builder.getF32Type());
Type dynMemref = MemRefType::get({ShapedType::kDynamic}, f64);
Type vec4f32 = VectorType::get({4}, builder.getF32Type());

// Function type:
FunctionType funcTy = builder.getFunctionType({i32, f64}, {i32});
```

### Building Attributes

```cpp
// Integer attribute:
Attribute int42 = builder.getI32IntegerAttr(42);

// Float attribute:
Attribute pi = builder.getF64FloatAttr(3.14159);

// String attribute:
Attribute str = builder.getStringAttr("hello");

// Array attribute:
Attribute arr = builder.getArrayAttr({int42, pi});

// Dense elements attribute (constant tensor):
DenseIntOrFPElementsAttr weights = DenseIntOrFPElementsAttr::get(
    RankedTensorType::get({2, 2}, builder.getF32Type()),
    {1.0f, 0.0f, 0.0f, 1.0f});
```

---

## Chapter Summary

- MLIR types and attributes are both uniqued, immutable, pointer-sized handles; equality is an `O(1)` pointer comparison; all are uniqued per `MLIRContext`
- Builtin integer types are signless (`i32` with signed/unsigned ops); arbitrary widths are supported; float types span `f16`, `bf16`, `f32`, `f64`, `f80`, `f128`, `tf32`
- `MemRefType` is a typed memory reference with optional shape, layout (affine map), and memory space; `RankedTensorType` uses value semantics; `VectorType` represents SIMD register values; the `ShapedType` interface unifies all three
- Custom types are defined in TableGen using `TypeDef` with typed parameters, ODS-generated `get()`/accessor methods, optional `assemblyFormat` for textual syntax, and optional `verify()` for invariant checking
- `DenseIntOrFPElementsAttr` stores dense constant tensor data inline; `DenseResourceElementsAttr` externalizes large blobs for efficient bytecode; `AffineMapAttr` and `SymbolRefAttr` are the other key structural attributes
- The **Properties** system (MLIR 17+) replaces attribute dictionary access for op-owned data with a direct struct layout, eliminating hash lookups and type erasure for frequently accessed op data
- `TypeConstraint` values in ODS (e.g., `AnyFloat`, `RankedTensorOf<[F32]>`) encode operand/result type requirements; `ShapedType` and `SubElementTypeInterface` provide interfaces for generic shaped-type operations; `TypeConverter` drives type transformations between dialects
