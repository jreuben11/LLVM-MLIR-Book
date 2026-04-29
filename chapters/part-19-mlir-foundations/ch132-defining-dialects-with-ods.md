# Chapter 132 — Defining Dialects with ODS

*Part XIX — MLIR Foundations*

ODS — the Operation Definition Specification — is MLIR's TableGen-based system for declaring dialects, types, attributes, and operations declaratively. From an ODS definition, the build system generates C++ class declarations, verifiers, builders, parsers, printers, and folder stubs. Writing these by hand for even a small dialect of a dozen ops is thousands of lines of boilerplate; ODS reduces that to concise, readable TableGen that self-documents the dialect's contract. This chapter covers the complete ODS workflow: dialect registration, operation definitions, type and attribute declarations, trait and constraint annotations, and the assembly format DSL that eliminates custom parsers for the vast majority of ops.

---

## 132.1 ODS (Operation Definition Specification)

### What ODS Is

ODS is a layer on top of LLVM's **TableGen** language ([`mlir/include/mlir/IR/OpBase.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/OpBase.td)). TableGen is a declarative specification language that, through a set of MLIR-specific backends, generates:

- `MyDialect.h.inc` / `MyDialect.cpp.inc`: dialect class and initialization
- `MyOps.h.inc` / `MyOps.cpp.inc`: op class declarations and definitions
- `MyTypes.h.inc` / `MyTypes.cpp.inc`: type class declarations
- `MyAttrs.h.inc` / `MyAttrs.cpp.inc`: attribute class declarations

The `.inc` files are then `#include`-d into hand-written `.h` and `.cpp` files.

### Why ODS Instead of Pure C++

A manually written `arith::AddIOp` in C++ needs:
- A class declaration inheriting `OpState`
- `build()` static methods (multiple overloads)
- `getODSOperandIndexAndLength()` for each operand
- `verify()` with constraint checks
- `print()` and `parse()` methods
- Trait declarations

ODS generates all of this from:

```tablegen
def Arith_AddIOp : Arith_Op<"addi",
    [Pure, Commutative, SameOperandsAndResultType]> {
  let summary = "Integer addition";
  let arguments = (ins AnySignlessIntegerOrIndex:$lhs,
                       AnySignlessIntegerOrIndex:$rhs);
  let results = (outs AnySignlessIntegerOrIndex:$result);
  let assemblyFormat = "$lhs `,` $rhs attr-dict `:` type($result)";
  let hasFolder = 1;
}
```

---

## 132.2 Dialect Definition

### TableGen Dialect Record

```tablegen
// MyDialect.td
include "mlir/IR/DialectBase.td"

def MyDialect : Dialect {
  let name = "my";           // textual name: ops use "my.opname"
  let cppNamespace = "::my"; // C++ namespace: ::my::MyAddOp
  let summary = "My example dialect for demonstration";
  let description = [{
    The MyDialect dialect provides example operations for teaching
    purposes. It demonstrates the ODS workflow.
  }];

  // Extra declarations injected into the generated class:
  let extraClassDeclaration = [{
    void registerTypes();
    void registerAttrs();
  }];

  // Whether to generate default attribute parser/printer:
  let useDefaultAttributePrinterParser = 1;
  let useDefaultTypePrinterParser = 1;
}
```

### Generated C++ Dialect Class

The generated `MyDialect.h.inc` contains:

```cpp
class MyDialect : public ::mlir::Dialect {
  explicit MyDialect(::mlir::MLIRContext *context);
public:
  static constexpr ::llvm::StringLiteral getDialectNamespace() {
    return ::llvm::StringLiteral("my");
  }
  void initialize();
  void registerTypes();
  void registerAttrs();
  // Parsing/printing hooks
};
```

### Registering the Dialect

```cpp
// MyDialect.cpp:
void MyDialect::initialize() {
  addOperations<
    #define GET_OP_LIST
    #include "MyOps.cpp.inc"
  >();
  registerTypes();
  registerAttrs();
}

// Registration function for use by tools:
void mlir::registerMyDialect(DialectRegistry &registry) {
  registry.insert<MyDialect>();
}
```

---

## 132.3 Operation Definition

### Minimal Op

```tablegen
// MyOps.td
include "MyDialect.td"
include "mlir/IR/OpBase.td"

class My_Op<string mnemonic, list<Trait> traits = []>
    : Op<MyDialect, mnemonic, traits>;

def My_AddOp : My_Op<"add", [Pure, Commutative, SameOperandsAndResultType]> {
  let summary = "Integer addition with arbitrary-width integers";
  let description = [{
    Computes the sum of two integers of the same type. The operation
    has no overflow semantics (wraps on overflow like C).
  }];

  let arguments = (ins
    AnySignlessInteger:$lhs,
    AnySignlessInteger:$rhs
  );
  let results = (outs
    AnySignlessInteger:$result
  );

  let assemblyFormat = "$lhs `,` $rhs attr-dict `:` type($result)";
  let hasFolder = 1;
}
```

### Arguments and Results

The `arguments` list specifies operands and attributes:

```tablegen
let arguments = (ins
  // Operands:
  I32:$count,                         // single i32 operand
  F32:$value,
  AnyFloat:$scale,                    // any float type
  Optional<I32>:$maybe_offset,        // optional operand (may be absent)
  Variadic<F32>:$values,              // variadic: zero or more f32 operands
  RankedTensorOf<[F32, F64]>:$data,  // tensor of f32 or f64

  // Attributes (compile-time constants):
  I32Attr:$step,                      // i32 attribute
  F64Attr:$threshold,
  StrAttr:$name,
  OptionalAttr<I32Attr>:$opt_count,   // optional attribute
  DefaultValuedAttr<I32Attr, "1">:$stride,  // attribute with default value
  BoolAttr:$flag,
  IndexAttr:$size,
  TypeAttrOf<I32>:$element_type       // attribute that is a type
);
```

The `results` list specifies output values:
```tablegen
let results = (outs
  I32:$quotient,
  I32:$remainder,
  Optional<I32>:$optional_result
);
```

### Generated Accessors

From the above, ODS generates:
```cpp
// Operand accessors:
Value getCount() { return getOperand(0); }
Value getValue() { return getOperand(1); }
Optional<Value> getMaybeOffset() { ... }
ValueRange getValues() { ... }  // variadic

// Attribute accessors:
int64_t getStep() { return getProperties().step.getInt(); }
double getThreshold() { return getProperties().threshold.getValueAsDouble(); }
StringRef getName() { return getProperties().name.getValue(); }
Optional<int64_t> getOptCount() { ... }
int64_t getStride() { return getProperties().stride.getInt(); }  // default=1

// Result accessors:
OpResult getQuotient() { return getResult(0); }
OpResult getRemainder() { return getResult(1); }
```

---

## 132.4 Traits and Constraints

### Common Traits

Traits are attached as template arguments in the `Op<..., traits>` specification. Key built-in traits:

```tablegen
// Purity and effects:
Pure               // NoMemoryEffect: no side effects → DCE/CSE eligible
NoTerminator       // op's regions don't need explicit terminators
Terminator         // must be the last op in a block

// Type relationships:
SameOperandsAndResultType    // all operands and results have same type
SameOperandsAndResultElementType  // for shaped types: same element type
AllTypesMatch<["lhs", "rhs"]>    // named operands must have same type
FirstAttrDerivedResultType       // result type = first operand's type

// Structural:
IsolatedFromAbove    // body regions cannot capture outer SSA values
AttrSizedOperandSegments // variadic operands have sizes in an attr
AttrSizedResultSegments  // variadic results have sizes in an attr
HasRecursiveMemoryEffects // effects of contained ops propagate

// Value semantics:
Commutative    // operand order irrelevant for folding
Idempotent     // f(f(x)) = f(x)
Involution     // f(f(x)) = x

// RegionBranch:
RecursivelySpeculatable  // op's regions are speculatively executable

// LLVM dialect specific:
LLVM_OneResultOp   // op produces exactly one result
```

### Defining a Verifier

For constraints too complex to express declaratively:

```tablegen
def My_MatmulOp : My_Op<"matmul", [Pure]> {
  let arguments = (ins
    RankedTensorOf<[AnyFloat]>:$lhs,
    RankedTensorOf<[AnyFloat]>:$rhs
  );
  let results = (outs RankedTensorOf<[AnyFloat]>:$result);
  let hasVerifier = 1;
}
```

```cpp
// In MyOps.cpp:
LogicalResult My::MatmulOp::verify() {
  auto lhsTy = cast<RankedTensorType>(getLhs().getType());
  auto rhsTy = cast<RankedTensorType>(getRhs().getType());
  auto resTy = cast<RankedTensorType>(getResult().getType());

  if (lhsTy.getRank() != 2 || rhsTy.getRank() != 2)
    return emitOpError("expected rank-2 input tensors");

  if (lhsTy.getDimSize(1) != rhsTy.getDimSize(0))
    return emitOpError("inner dimensions must match: ")
           << lhsTy.getDimSize(1) << " vs " << rhsTy.getDimSize(0);

  if (resTy.getDimSize(0) != lhsTy.getDimSize(0) ||
      resTy.getDimSize(1) != rhsTy.getDimSize(1))
    return emitOpError("result shape mismatch");

  return success();
}
```

### Defining a Canonicalizer (Folder)

```tablegen
def My_MulOp : My_Op<"mul", [Pure, Commutative, SameOperandsAndResultType]> {
  let arguments = (ins AnySignlessInteger:$lhs, AnySignlessInteger:$rhs);
  let results = (outs AnySignlessInteger:$result);
  let hasFolder = 1;
  let hasCanonicalize = 1;
}
```

```cpp
// Folder: constant-fold when both operands are constants
OpFoldResult My::MulOp::fold(FoldAdaptor adaptor) {
  // adaptor provides attribute representations of operands
  auto lhsAttr = dyn_cast_or_null<IntegerAttr>(adaptor.getLhs());
  auto rhsAttr = dyn_cast_or_null<IntegerAttr>(adaptor.getRhs());
  if (lhsAttr && rhsAttr)
    return IntegerAttr::get(getType(),
                             lhsAttr.getValue() * rhsAttr.getValue());

  // Multiply by 1 → identity
  if (rhsAttr && rhsAttr.getValue().isOne())
    return getLhs();

  return {};  // no fold
}

// Canonicalizer: rewrite to simpler form
void My::MulOp::getCanonicalizationPatterns(RewritePatternSet &results,
                                             MLIRContext *ctx) {
  results.add<FoldMulByZero>(ctx);
}
```

---

## 132.5 Assembly Format DSL

The `assemblyFormat` string in ODS is a mini-DSL that specifies the textual representation of an op. This eliminates the need for hand-written `parse()` and `print()` methods in the vast majority of cases.

### Basic Tokens

| Token | Meaning |
|-------|---------|
| `$name` | Print/parse the operand or attribute named `name` |
| `type($name)` | Print/parse the type of operand `name` |
| `` `literal` `` | Print/parse the literal string |
| `attr-dict` | Print/parse the remaining attributes as `{...}` |
| `attr-dict-with-keyword` | Same but requires the `{` to be preceded by `attributes` |
| `functional-type($ins, $outs)` | Print/parse function-type syntax for ins/outs |
| `qualified($name)` | Print type with full dialect qualification |

### Examples

```tablegen
// arith.addi %a, %b : i32
let assemblyFormat = "$lhs `,` $rhs attr-dict `:` type($result)";

// func.call @sym(%args) : (i32, f64) -> i32
let assemblyFormat = "$callee `(` $operands `)` attr-dict `:` functional-type($operands, results)";

// memref.load %ptr[%i, %j] : memref<10x10xf32>
let assemblyFormat = "$memref `[` $indices `]` attr-dict `:` type($memref)";

// With optional operand:
// my.select %cond, %true_val [, %false_val] : i32
let assemblyFormat = "$condition `,` $trueValue (`,` $falseValue^)? attr-dict `:` type($result)";
```

### Optional Groups

The `(... ^)?` syntax prints the group only when the `^`-marked element is present:

```tablegen
// For an op with an optional attribute:
let assemblyFormat = "$value (`where` $cond^)? attr-dict `:` type($value)";
// Prints: "my.op %val where %cond : i32" when cond is present
// Prints: "my.op %val : i32" when cond is absent
```

### Type Inference

When operand and result types are derivable, ODS can omit them from the format:

```tablegen
// SameOperandsAndResultType → only print type once:
let assemblyFormat = "$lhs `,` $rhs attr-dict `:` type($result)";
// Since all three have the same type, printing type($result) is sufficient;
// ODS parses the type and applies it to lhs, rhs, and result.
```

### Custom Assembly Format

For ops where the DSL is insufficient:

```tablegen
def My_ComplexOp : My_Op<"complex"> {
  let hasCustomAssemblyFormat = 1;
}
```

```cpp
// MyOps.cpp:
ParseResult My::ComplexOp::parse(OpAsmParser &parser, OperationState &result) {
  // Custom parsing logic
  OpAsmParser::UnresolvedOperand lhs, rhs;
  Type ty;
  if (parser.parseOperand(lhs) || parser.parseComma() ||
      parser.parseOperand(rhs) || parser.parseColonType(ty))
    return failure();
  // Resolve operand types:
  if (parser.resolveOperand(lhs, ty, result.operands) ||
      parser.resolveOperand(rhs, ty, result.operands))
    return failure();
  result.addTypes({ty});
  return success();
}

void My::ComplexOp::print(OpAsmPrinter &p) {
  p << " " << getLhs() << ", " << getRhs();
  p.printOptionalAttrDict((*this)->getAttrs());
  p << " : " << getResult().getType();
}
```

---

## 132.6 Type and Attribute ODS Definitions

### Type Constraints in ODS

```tablegen
// Type definition:
def My_ComplexTensorType : TypeDef<MyDialect, "ComplexTensor"> {
  let mnemonic = "complex_tensor";
  let parameters = (ins
    "int64_t":$rows,
    "int64_t":$cols,
    TypeParameter<"mlir::FloatType", "element float type">:$elementType
  );
  let assemblyFormat = "`<` $rows `x` $cols `x` $elementType `>`";
  let genVerifyDecl = 1;
}
```

```cpp
// Verify: element type must be f32 or f64:
LogicalResult My::ComplexTensorType::verify(
    function_ref<InFlightDiagnostic()> emitError,
    int64_t rows, int64_t cols, FloatType elemTy) {
  if (!elemTy.isa<Float32Type, Float64Type>())
    return emitError() << "element type must be f32 or f64";
  return success();
}
```

### Attribute Constraints in ODS

```tablegen
def My_ScheduleAttr : AttrDef<MyDialect, "Schedule"> {
  let mnemonic = "schedule";
  let summary = "Compilation schedule hint";
  let parameters = (ins
    "ScheduleKind":$kind,         // C++ enum
    "unsigned":$priority,
    OptionalParameter<"mlir::StringAttr">:$label
  );
  let assemblyFormat = "`<` $kind `,` $priority (`,` $label^)? `>`";
}
```

---

## 132.7 Building and Using ODS-Defined Ops

### CMake Integration

```cmake
# CMakeLists.txt for a dialect:
add_mlir_dialect(MyOps my)  # generates MyOps.h.inc, MyOps.cpp.inc

# Or manually:
mlir_tablegen(MyOps.h.inc   -gen-op-decls)
mlir_tablegen(MyOps.cpp.inc -gen-op-defs)
mlir_tablegen(MyTypes.h.inc  -gen-typedef-decls)
mlir_tablegen(MyTypes.cpp.inc -gen-typedef-defs)
mlir_tablegen(MyAttrs.h.inc  -gen-attrdef-decls)
mlir_tablegen(MyAttrs.cpp.inc -gen-attrdef-defs)
add_public_tablegen_target(MyOpsIncGen)
```

### Including Generated Code

```cpp
// MyOps.h:
#include "mlir/IR/OpDefinition.h"

namespace my {
  // Forward declarations:
  class MyDialect;
}

#define GET_OP_CLASSES
#include "MyOps.h.inc"  // includes generated op declarations

// MyOps.cpp:
#define GET_OP_LIST
#include "MyOps.cpp.inc"  // includes generated op definitions
```

### Using ODS-Defined Ops

```cpp
// Creating an op:
OpBuilder builder(&ctx);
builder.setInsertionPointToEnd(block);
Location loc = builder.getUnknownLoc();

auto addOp = builder.create<my::AddOp>(loc, lhs, rhs);
// lhs, rhs are Values; result type is inferred from SameOperandsAndResultType

Value result = addOp.getResult();
Value lhsVal = addOp.getLhs();    // named accessor from ODS
Value rhsVal = addOp.getRhs();
```

### Dialect Registration in Context

```cpp
// Register all needed dialects:
MLIRContext ctx;
ctx.loadDialect<my::MyDialect,
                mlir::arith::ArithDialect,
                mlir::func::FuncDialect>();

// Or using registry:
DialectRegistry registry;
registry.insert<my::MyDialect>();
MLIRContext ctx(registry);
ctx.loadAllAvailableDialects();
```

---

## 132.8 End-to-End: A Complete Toy Dialect

To demonstrate the full ODS workflow, here is a self-contained toy `calc` dialect with two ops:

```tablegen
// Calc.td
include "mlir/IR/OpBase.td"
include "mlir/IR/BuiltinAttributes.td"

def Calc_Dialect : Dialect {
  let name = "calc";
  let cppNamespace = "::calc";
}

class Calc_Op<string mnemonic, list<Trait> traits = []>
    : Op<Calc_Dialect, mnemonic, traits>;

def Calc_ConstantOp : Calc_Op<"constant", [Pure, ConstantLike]> {
  let summary = "Integer constant";
  let arguments = (ins I64Attr:$value);
  let results = (outs I64:$result);
  let assemblyFormat = "$value attr-dict";
  let hasFolder = 1;
}

def Calc_AddOp : Calc_Op<"add", [Pure, Commutative, SameOperandsAndResultType]> {
  let summary = "Integer addition";
  let arguments = (ins I64:$lhs, I64:$rhs);
  let results = (outs I64:$result);
  let assemblyFormat = "$lhs `,` $rhs attr-dict `:` type($result)";
  let hasFolder = 1;
}
```

A program in this dialect:
```mlir
func.func @compute(%x: i64) -> i64 {
  %c10 = calc.constant 10
  %y = calc.add %x, %c10 : i64
  return %y : i64
}
```

The folder for `Calc_ConstantOp` returns `value` attr as `OpFoldResult`; the folder for `Calc_AddOp` folds when both inputs are `Calc_ConstantOp` results.

---

## Chapter Summary

- ODS is a TableGen-based system that generates C++ op classes, verifiers, builders, parsers, and printers from concise declarative descriptions; it eliminates boilerplate and makes dialect contracts self-documenting
- A dialect definition specifies namespace, C++ namespace, and initialization hooks; the generated class is registered with `MLIRContext` via `DialectRegistry::insert<D>()` and `initialize()`
- Op definitions use `arguments` (operands + attributes), `results`, `traits`, `hasVerifier`, `hasFolder`, and `hasCanonicalize` to specify behavior; ODS generates named accessor methods for all declared operands, attributes, and results
- Common traits — `Pure`, `Commutative`, `SameOperandsAndResultType`, `IsolatedFromAbove`, `Terminator`, `AttrSizedOperandSegments` — encode op semantics and enable generic analyses and transformations without inspecting op internals
- The `assemblyFormat` DSL handles textual syntax for most ops using tokens for operands (`$name`), types (`type($name)`), literals (`` `token` ``), optional groups (`(... ^)?`), and attribute dictionaries (`attr-dict`)
- Types and attributes are defined with `TypeDef` and `AttrDef`, specifying parameters with typed `TypeParameter<"T", "doc">` entries, optional `assemblyFormat`, and optional `genVerifyDecl` for invariant checking
- The build system generates `.inc` files via `mlir_tablegen`; these are `#include`-d from hand-written `.h`/`.cpp` files using `GET_OP_CLASSES`/`GET_OP_LIST` guards
