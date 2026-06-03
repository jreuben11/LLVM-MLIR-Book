# Chapter 132 — Defining Dialects with ODS

*Part XIX — MLIR Foundations*

ODS — the Operation Definition Specification — is MLIR's TableGen-based system for declaring dialects, types, attributes, and operations declaratively. From an ODS definition, the build system generates C++ class declarations, verifiers, builders, parsers, printers, and folder stubs. Writing these by hand for even a small dialect of a dozen ops is thousands of lines of boilerplate; ODS reduces that to concise, readable TableGen that self-documents the dialect's contract. This chapter covers the complete ODS workflow: dialect registration, operation definitions, type and attribute declarations, trait and constraint annotations, and the assembly format DSL that eliminates custom parsers for the vast majority of ops.

---

## Table of Contents

- [132.1 ODS (Operation Definition Specification)](#1321-ods-operation-definition-specification)
  - [What ODS Is](#what-ods-is)
  - [Why ODS Instead of Pure C++](#why-ods-instead-of-pure-c)
- [132.2 Dialect Definition](#1322-dialect-definition)
  - [TableGen Dialect Record](#tablegen-dialect-record)
  - [Generated C++ Dialect Class](#generated-c-dialect-class)
  - [Registering the Dialect](#registering-the-dialect)
- [132.3 Operation Definition](#1323-operation-definition)
  - [Minimal Op](#minimal-op)
  - [Arguments and Results](#arguments-and-results)
  - [Generated Accessors](#generated-accessors)
- [132.4 Traits and Constraints](#1324-traits-and-constraints)
  - [Common Traits](#common-traits)
  - [Defining a Verifier](#defining-a-verifier)
  - [Defining a Canonicalizer (Folder)](#defining-a-canonicalizer-folder)
- [132.5 Assembly Format DSL](#1325-assembly-format-dsl)
  - [Basic Tokens](#basic-tokens)
  - [Examples](#examples)
  - [Optional Groups](#optional-groups)
  - [Type Inference](#type-inference)
  - [Custom Assembly Format](#custom-assembly-format)
- [132.6 Type and Attribute ODS Definitions](#1326-type-and-attribute-ods-definitions)
  - [Type Constraints in ODS](#type-constraints-in-ods)
  - [Attribute Constraints in ODS](#attribute-constraints-in-ods)
- [132.7 Building and Using ODS-Defined Ops](#1327-building-and-using-ods-defined-ops)
  - [CMake Integration](#cmake-integration)
  - [Including Generated Code](#including-generated-code)
  - [Using ODS-Defined Ops](#using-ods-defined-ops)
  - [Dialect Registration in Context](#dialect-registration-in-context)
- [132.8 End-to-End: A Complete Toy Dialect](#1328-end-to-end-a-complete-toy-dialect)
- [Runtime Dialect Extensibility: IRDL and DynamicDialect](#runtime-dialect-extensibility-irdl-and-dynamicdialect)
  - [The IRDL Dialect](#the-irdl-dialect)
  - [Loading an IRDL File at Runtime](#loading-an-irdl-file-at-runtime)
  - [DynamicDialect: C++ API for Runtime Dialect Creation](#dynamicdialect-c-api-for-runtime-dialect-creation)
  - [Use Cases](#use-cases)
  - [Comparison: ODS vs. IRDL](#comparison-ods-vs-irdl)
  - [Example: A Complete IRDL-Defined Dialect](#example-a-complete-irdl-defined-dialect)
- [Chapter Summary](#chapter-summary)

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

## Runtime Dialect Extensibility: IRDL and DynamicDialect

ODS is the standard approach for defining MLIR dialects: write TableGen, run `mlir-tblgen`, include the generated `.inc` files, recompile. This static, compile-time workflow is optimal for production dialects where type safety, full verifier coverage, and maximum performance matter. However, it is poorly suited to two important scenarios: research prototyping (where a dialect's op set changes daily) and DSL embedding (where a user-supplied dialect specification must be loaded at runtime without any C++ compilation).

IRDL (the IR Definition Language) and the `DynamicDialect` C++ API address both scenarios.

### The IRDL Dialect

IRDL ([`mlir/include/mlir/Dialect/IRDL/IR/IRDL.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/IRDL/IR/IRDL.td), introduced 2022) is a meta-dialect: an MLIR dialect whose operations define other dialects. An IRDL file is itself a valid `.mlir` file that can be loaded by `mlir-opt` to register the described dialect at runtime.

#### Core Operations

```mlir
// Declare a dialect named "toy"
irdl.dialect @toy {

  // Declare a type "tensor" within the dialect
  irdl.type @tensor {
    %0 = irdl.any_of [i32, f32, f64]    // element type constraint
    irdl.parameters(%0)
  }

  // Declare an operation "add"
  irdl.operation @add {
    %elem_type = irdl.any_of [i32, f32]
    %tensor_ty = irdl.base @toy::@tensor  // must be a toy.tensor
    irdl.operands(%tensor_ty, %tensor_ty) // two operands
    irdl.results(%tensor_ty)             // one result
  }

  // Declare an operation "constant"
  irdl.operation @constant {
    %0 = irdl.any                        // unconstrained type
    irdl.results(%0)
  }
}
```

The constraint operations are:

| Operation | Meaning |
|-----------|---------|
| `irdl.any` | Accepts any MLIR type |
| `irdl.is` | Accepts exactly one specific type (e.g., `irdl.is i32`) |
| `irdl.base` | Accepts any parametric instance of a named type (e.g., any `toy.tensor`) |
| `irdl.any_of` | Accepts any type from a fixed list (union of types) |
| `irdl.all_of` | Accepts a type that satisfies all listed constraints (intersection) |
| `irdl.parameters` | Declares type parameters and their constraints |
| `irdl.operands` / `irdl.results` | Constrains operand/result types for an op |

#### Loading an IRDL File at Runtime

```bash
# Load a dialect definition from an IRDL file and verify an MLIR file against it
mlir-opt --irdl-file=toy_dialect.irdl input.mlir

# Or combine with other passes:
mlir-opt --irdl-file=my_dialect.irdl \
         --canonicalize \
         --mlir-verify-each \
         input.mlir
```

When `--irdl-file` is specified, `mlir-opt` loads the IRDL file, registers the described dialect with the `MLIRContext`, and then processes `input.mlir` as if the dialect had been statically compiled in. Op names like `toy.add` are recognized, operand/result type constraints are verified, and the generic op printing/parsing infrastructure handles textual I/O.

### DynamicDialect: C++ API for Runtime Dialect Creation

The `DynamicDialect` API ([`mlir/include/mlir/IR/ExtensibleDialect.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/ExtensibleDialect.h)) provides a C++ interface for creating dialects entirely at runtime, without IRDL files:

```cpp
#include "mlir/IR/ExtensibleDialect.h"
#include "mlir/IR/MLIRContext.h"

MLIRContext ctx;

// Create a dynamic dialect
auto *dynDialect = ctx.getOrLoadDynamicDialect(
    "runtime_dialect",
    [](DynamicDialect *dialect) {
      // Register a dynamic type
      auto tensorTy = DynamicType::get(dialect, "tensor",
          /*verifier=*/[](function_ref<InFlightDiagnostic()> emitError,
                          ArrayRef<Attribute> params) -> LogicalResult {
            if (params.size() != 1 || !isa<TypeAttr>(params[0]))
              return emitError() << "expected one type parameter";
            return success();
          });
      dialect->registerDynamicType(std::move(tensorTy));

      // Register a dynamic op
      auto addOp = DynamicOpDefinition::get(
          "add", dialect,
          /*verifier=*/[](Operation *op) -> LogicalResult {
            if (op->getNumOperands() != 2 || op->getNumResults() != 1)
              return op->emitOpError("expected 2 operands and 1 result");
            return success();
          },
          /*foldHook=*/nullptr);
      dialect->registerDynamicOp(std::move(addOp));
    });
```

Dynamic ops and types created via this API participate in the full MLIR infrastructure: they can be printed, parsed, verified, and processed by generic passes.

### Use Cases

**Research and prototyping**: when exploring a new IR abstraction, IRDL lets you define and iterate on an op set without C++ compilation cycles. A new op takes seconds to add to the `.irdl` file; the verifier and parser are immediately available.

**DSL embedding**: a host MLIR program can load a user-supplied dialect at runtime. For example, a kernel-fusion framework might accept a user-defined "schedule dialect" that describes fusion strategies; the framework loads the IRDL file and processes it without recompilation.

**Plugin-based dialect extensibility**: tools that support third-party dialect plugins can use `DynamicDialect` to register plugin-provided dialects. The plugin supplies a shared library that calls `DynamicDialect::get()` on load; the host tool needs no knowledge of the dialect at build time.

### Comparison: ODS vs. IRDL

| Property | ODS (static) | IRDL (dynamic) |
|----------|-------------|----------------|
| **Type safety** | Full C++ type checking at compile time | Constraint-based at parse/verify time |
| **Performance** | Maximal; direct C++ method dispatch | Slight overhead; dynamic dispatch through verifier lambdas |
| **Validation timing** | Compile time + parse time | Parse time and verify time only |
| **New clause addition** | Requires C++ recompilation | Edit `.irdl` file, re-run `mlir-opt` |
| **Appropriate for** | Production dialects, in-tree dialects | Research, prototyping, DSLs, plugins |
| **Tooling integration** | Full: LSP, query, pdll | Partial: generic print/parse only |

### Example: A Complete IRDL-Defined Dialect

```mlir
// file: toy.irdl
// A minimal "toy" dialect with one type and one binary op.

irdl.dialect @toy {

  // toy.value<elem_type>: a typed value container
  irdl.type @value {
    %elem = irdl.any_of [i32, i64, f32, f64]
    irdl.parameters(%elem)
  }

  // toy.add: element-wise addition of two toy.value containers
  irdl.operation @add {
    %elem = irdl.any_of [i32, i64, f32, f64]
    %val_ty = irdl.base @toy::@value
    irdl.operands(%val_ty, %val_ty)
    irdl.results(%val_ty)
  }
}
```

```mlir
// file: prog.mlir — uses the toy dialect defined above
func.func @compute(%a : !toy.value<f32>, %b : !toy.value<f32>) -> !toy.value<f32> {
  %c = toy.add %a, %b : !toy.value<f32>
  return %c : !toy.value<f32>
}
```

```bash
# Load the dialect at runtime and verify the program:
mlir-opt --irdl-file=toy.irdl --mlir-verify-each prog.mlir
# Output: prog.mlir verified successfully; no errors.
```

The `toy.add` op is recognized and its operand/result type constraints are verified against the `irdl.any_of [i32, i64, f32, f64]` constraint. No C++ code was compiled.

---

## Chapter Summary

- ODS is a TableGen-based system that generates C++ op classes, verifiers, builders, parsers, and printers from concise declarative descriptions; it eliminates boilerplate and makes dialect contracts self-documenting
- A dialect definition specifies namespace, C++ namespace, and initialization hooks; the generated class is registered with `MLIRContext` via `DialectRegistry::insert<D>()` and `initialize()`
- Op definitions use `arguments` (operands + attributes), `results`, `traits`, `hasVerifier`, `hasFolder`, and `hasCanonicalize` to specify behavior; ODS generates named accessor methods for all declared operands, attributes, and results
- Common traits — `Pure`, `Commutative`, `SameOperandsAndResultType`, `IsolatedFromAbove`, `Terminator`, `AttrSizedOperandSegments` — encode op semantics and enable generic analyses and transformations without inspecting op internals
- The `assemblyFormat` DSL handles textual syntax for most ops using tokens for operands (`$name`), types (`type($name)`), literals (`` `token` ``), optional groups (`(... ^)?`), and attribute dictionaries (`attr-dict`)
- Types and attributes are defined with `TypeDef` and `AttrDef`, specifying parameters with typed `TypeParameter<"T", "doc">` entries, optional `assemblyFormat`, and optional `genVerifyDecl` for invariant checking
- The build system generates `.inc` files via `mlir_tablegen`; these are `#include`-d from hand-written `.h`/`.cpp` files using `GET_OP_CLASSES`/`GET_OP_LIST` guards


---

@copyright jreuben11
