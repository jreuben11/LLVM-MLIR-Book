# Chapter 135 — PDL and PDLL

*Part XIX — MLIR Foundations*

Pattern rewriting is the core transformation mechanism in MLIR, and the C++ `RewritePattern` API—while expressive—demands substantial boilerplate for even simple structural rewrites. MLIR provides two declarative alternatives: the PDL (Pattern Description Language) dialect, which encodes patterns as first-class MLIR IR, and PDLL (Pattern Description Language Level), a purpose-built standalone language with a dedicated compiler. Together they eliminate the mechanical code required for structural matching, constraint checking, and replacement, while remaining composable with hand-written C++ patterns. This chapter covers both systems in depth: the PDL dialect's IR representation, ODS-embedded `Pat<>` patterns in TableGen, the PDLL language and its compilation pipeline, the PDL bytecode interpreter for runtime-loaded patterns, and practical guidance on choosing between declarative and imperative approaches.

---

## 135.1 The Pattern Rewriting Problem

### The C++ RewritePattern Baseline

Every MLIR transformation ultimately reduces to a set of `RewritePattern` applications. The C++ API, declared in [`mlir/include/mlir/IR/PatternMatch.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/PatternMatch.h), provides the `RewritePattern` base class from which users derive. A typical pattern that folds `arith.addi %x, 0` to `%x` requires:

```cpp
#include "mlir/IR/PatternMatch.h"
#include "mlir/Dialect/Arith/IR/Arith.h"

struct FoldAddZero : public mlir::OpRewritePattern<mlir::arith::AddIOp> {
  using OpRewritePattern::OpRewritePattern;

  mlir::LogicalResult
  matchAndRewrite(mlir::arith::AddIOp op,
                  mlir::PatternRewriter &rewriter) const override {
    // Check that the RHS is a constant zero.
    mlir::IntegerAttr cst;
    if (!mlir::matchPattern(op.getRhs(), mlir::m_Constant(&cst)))
      return mlir::failure();
    if (!cst.getValue().isZero())
      return mlir::failure();
    rewriter.replaceOp(op, op.getLhs());
    return mlir::success();
  }
};
```

For patterns of this complexity — single-op structural match with one attribute predicate — the class definition, `matchAndRewrite` boilerplate, and explicit match failure paths total 20–30 lines of code that carry little semantic information. Scaling to tens of canonicalization patterns in a dialect produces files dominated by mechanical structure rather than intent.

### What Declarative Systems Provide

The declarative systems offered by MLIR — ODS `Pat<>` / `Pattern<>` in TableGen, the PDL dialect, and PDLL — address this by letting the engineer express *what* to match and *how* to replace, without the surrounding C++ scaffolding:

- **Structural matching**: operand trees expressed as DAGs or nested op constructors
- **Constraint predicates**: typed attribute tests, value constraints, and native C++ escapes for complex logic
- **Replacement specification**: new op constructors or passthrough value references
- **Benefit annotation**: priority for ordering pattern application

The trade-off is power. Declarative patterns cannot walk the IR beyond the matched root, perform multi-step analysis, accumulate state across multiple ops, or use arbitrary C++ data structures during matching. When a pattern exceeds those limits, a C++ `RewritePattern` is the right tool. Most canonicalization and peephole patterns, however, fall squarely within declarative expressiveness.

### Two Declarative Systems

MLIR currently offers two declarative pattern systems:

| System | File type | Processor | Output |
|--------|-----------|-----------|--------|
| ODS `Pat<>` | `.td` (TableGen) | `mlir-tblgen` | C++ pattern classes |
| PDLL | `.pdll` | `mlir-pdll` | C++ classes *or* PDL dialect IR |

Both systems lower ultimately to either C++ `RewritePattern` subclasses (for static linking) or PDL bytecode (for dynamic/plugin scenarios). The PDL dialect is the shared intermediate representation between PDLL and the bytecode interpreter.

---

## 135.2 The PDL Dialect

### PDL as MLIR IR

The PDL dialect ([`mlir/include/mlir/Dialect/PDL/IR/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/PDL/IR/)) represents patterns as first-class MLIR operations. This unusual design — IR that describes how to rewrite other IR — enables the full MLIR infrastructure (passes, printing, serialization, bytecode) to apply to patterns themselves. The key types defined in [`PDLTypes.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/PDL/IR/PDLTypes.h) are:

| PDL type | Represents |
|----------|------------|
| `!pdl.operation` | An MLIR operation being matched or created |
| `!pdl.value` | An MLIR `Value` (SSA value) |
| `!pdl.type` | An MLIR `Type` |
| `!pdl.attribute` | An MLIR `Attribute` |
| `!pdl.range<T>` | A range of values, types, or operations |

### Core PDL Operations

The dialect's operations, defined in [`PDLOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/PDL/IR/PDLOps.td), form two groups: *match* operations that describe what to capture, and *rewrite* operations nested inside a `pdl.rewrite` block that describe replacement.

**Match-side operations:**

- `pdl.pattern`: top-level container with a `benefit` attribute and an optional symbolic name
- `pdl.operation`: matches an operation by name, with optional named attributes and operands
- `pdl.value`: matches a `Value`, optionally constrained by type
- `pdl.type`: matches a `Type`, optionally with a concrete type binding
- `pdl.attribute`: matches an `Attribute`, optionally with a concrete value
- `pdl.result N of %op`: extracts the N-th result of a matched operation
- `pdl.operands of %op`: extracts all operands

**Rewrite-side operations (inside `pdl.rewrite`):**

- `pdl.replace %op with (%val : !pdl.value)`: replace op results with existing values
- `pdl.replace %op with %new_op`: replace with a new operation
- `pdl.erase %op`: erase the matched operation
- `pdl.create_operation`: create a new operation
- `pdl.apply_native_rewrite`: call into registered C++ rewrite functions

### A Complete PDL Pattern

```mlir
// PDL textual form: fold arith.addi(%x, constant(0)) → %x
pdl.pattern @FoldAddZero : benefit(1) {
  // Match types and attributes
  %i32 = pdl.type : i32
  %zero_attr = pdl.attribute : i32 = 0 : i32

  // Match arith.constant {value = 0 : i32} → %const_result
  %const_op = pdl.operation "arith.constant"
              {"value" = %zero_attr} -> (%i32 : !pdl.type)
  %const_result = pdl.result 0 of %const_op

  // Match any i32 value %lhs
  %lhs = pdl.value : i32

  // Match arith.addi(%lhs, %const_result)
  %add_op = pdl.operation "arith.addi"
            (%lhs, %const_result : !pdl.value, !pdl.value)
            -> (%i32 : !pdl.type)

  // Rewrite: replace the add with the LHS directly
  pdl.rewrite %add_op {
    pdl.replace %add_op with (%lhs : !pdl.value)
  }
}
```

The `pdl.pattern` operation acts as the pattern boundary. Everything before `pdl.rewrite` is the match specification; the `pdl.rewrite` block is the replacement. Because this is valid MLIR IR, it can be stored in `.mlir` files, parsed at runtime, serialized to bytecode, and processed by the PDL-to-PDLInterp lowering pass before execution.

### Constraints in PDL

PDL supports native constraints through `pdl.apply_native_constraint`:

```mlir
pdl.pattern @ConstrainedFold : benefit(1) {
  %type = pdl.type : i32
  %attr = pdl.attribute : i32
  %const_op = pdl.operation "arith.constant" {"value" = %attr} -> (%type : !pdl.type)
  %const_result = pdl.result 0 of %const_op
  %lhs = pdl.value : i32
  %add_op = pdl.operation "arith.addi"(%lhs, %const_result : !pdl.value, !pdl.value)
            -> (%type : !pdl.type)

  // Call a registered C++ constraint function named "isZeroAttr"
  pdl.apply_native_constraint "isZeroAttr"(%attr : !pdl.attribute)

  pdl.rewrite %add_op {
    pdl.replace %add_op with (%lhs : !pdl.value)
  }
}
```

The constraint function `"isZeroAttr"` must be registered in the `PDLPatternModule` before pattern application (see Section 135.6).

---

## 135.3 ODS-Based PDL Patterns — The `Pat<>` Class

### PatternBase.td and the `Pat<>` Class

Before PDLL existed, the primary declarative mechanism was the `Pat<>` class in TableGen, defined in [`mlir/include/mlir/IR/PatternBase.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/PatternBase.td). This mechanism is deeply integrated with ODS — op definitions generate the DAG node names used as source and result patterns:

```tablegen
// In PatternBase.td:
class Pat<dag pattern, dag result, list<dag> preds = [],
          list<dag> supplemental_results = [],
          dag benefitAdded = (addBenefit 0)>
  : Pattern<pattern, [result], preds, supplemental_results, benefitAdded>;
```

The source `pattern` DAG specifies what to match; the result `result` DAG specifies what to create. Each node in the DAG is either an op name (from ODS), a bound variable (`$name`), a constraint expression, or a `NativeCodeCall<>`.

### Source and Result DAG Syntax

```tablegen
include "mlir/IR/PatternBase.td"
include "mlir/Dialect/Arith/IR/ArithOps.td"

// Match arith.addi(lhs, arith.constant {value = cst : i32})
// Replace with lhs when cst is zero
def FoldAddZero : Pat<
  (Arith_AddIOp $lhs, (Arith_ConstantOp I32Attr:$cst)),
  (replaceWithValue $lhs),
  [(IsZeroI32Attr $cst)]   // constraints list
>;
```

Source DAG nodes use the ODS op name (the TableGen `def` name). Operands either bind to names (`$lhs`), apply typed attribute constraints (`I32Attr:$cst`), or match nested ops (the inner `(Arith_ConstantOp ...)`). The result DAG uses either new op constructors or the built-in directives:

| Directive | Meaning |
|-----------|---------|
| `replaceWithValue $x` | Replace all results with the value bound to `$x` |
| `returnType $_self` | Infer result type from matched operand |
| `attr-dict` | Pass through the source op's attribute dictionary |

### NativeCodeCall

`NativeCodeCall<"expr">` is the escape hatch for computations that cannot be expressed as pure DAG transformations. The expression is C++ code with special placeholders:

- `$N` refers to the N-th argument of the `NativeCodeCall` invocation
- `$_builder` refers to the `PatternRewriter &`
- `$_self` refers to the defining op of the operand in a leaf position

```tablegen
// Compute negation as: 0 - x, creating the zero constant inline
def NegateI32 : Pat<
  (My_NegI32Op $x),
  (Arith_SubIOp
    (Arith_ConstantOp
      (NativeCodeCall<"$_builder.getIntegerAttr($0.getType(), 0)"> $x)),
    $x)
>;

// Multi-result NativeCodeCall: specify returns=2
class SplitI64 : NativeCodeCall<"splitI64($0, &$_builder)", 2>;
```

`NativeCodeCallVoid<"expr">` is a zero-return variant used in `supplemental_results` for side effects (e.g., copying attributes).

### Constraint Predicates

Predicates in the `preds` list are expressed as DAG applications of `Constraint<Pred<...>>` subclasses or `CPred<"...">` inline predicates:

```tablegen
// Define a named constraint
def IsZeroI32Attr : Constraint<CPred<
  "$_self.cast<IntegerAttr>().getValue().isZero()"
>, "must be zero">;

// Use in Pat:
def FoldMulByOne : Pat<
  (Arith_MulIOp $lhs, (Arith_ConstantOp I32Attr:$cst)),
  (replaceWithValue $lhs),
  [(IsOneI32Attr $cst)]
>;
```

Multi-entity constraints (spanning multiple bound names) use the `Pattern<>` class directly and list constraints as a `list<dag>`:

```tablegen
// Verify two operands have the same type
def SameOperandType : Constraint<
  CPred<"$0.getType() == $1.getType()">, "same type"
>;

def : Pattern<(My_AddOp $a, $b),
              [(My_FusedOp $a, $b)],
              [(SameOperandType $a, $b)]>;
```

### Generated Output

`mlir-tblgen -gen-rewriters` processes `.td` files containing `Pat<>` definitions and emits a C++ file with:

```cpp
// Generated by mlir-tblgen. Do not edit.
static void populateWithGenerated(::mlir::RewritePatternSet &patterns) {
  patterns.add<FoldAddZero>(patterns.getContext());
  patterns.add<FoldMulByOne>(patterns.getContext());
  // ...
}
```

Each `Pat<>` becomes a `RewritePattern` subclass implementing `matchAndRewrite`. The generated code is included via an `#include "MyDialectPatterns.inc"` line in the pass source file.

---

## 135.4 PDLL — The Standalone Pattern Language

### PDLL Overview

PDLL (`.pdll` files) is a purpose-built language for expressing MLIR patterns, processed by [`mlir-pdll`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/tools/mlir-pdll/). It provides:

- A cleaner syntax than TableGen DAGs, closer to the pattern's logical structure
- A richer type system that directly mirrors MLIR's type hierarchy
- Better error messages and IDE support (`mlir-pdll-lsp-server`)
- Support for named rewrite functions, composable constraints, and multi-pattern files

A PDLL file contains three kinds of declarations: `Constraint`, `Rewrite`, and `Pattern`, each of which may be named and composed.

### Basic PDLL Syntax

```pdll
// Include ODS-generated PDLL declarations for the arith dialect
#include "mlir/Dialect/Arith/IR/ArithOps.pdll"

// Declare a named constraint that checks an Attr is an i32 zero
Constraint IsZero(attr: Attr) [{
  return success(
    llvm::isa<IntegerAttr>(attr) &&
    llvm::cast<IntegerAttr>(attr).getValue().isZero()
  );
}];

// Pattern: fold arith.addi(%lhs, arith.constant(0)) → %lhs
Pattern FoldAddZero with benefit(1) {
  let cstAttr : Attr;
  let lhs     : Value;

  // Match a constant with an i32 zero attribute
  let cstOp = op<arith.constant> {value = cstAttr : I32Attr};
  let cstVal = cstOp.0;   // first result of cstOp

  // Match addi with lhs and the constant result
  let root = op<arith.addi>(lhs, cstVal);

  // Apply the constraint
  IsZero(cstAttr);

  // Replace the root with lhs
  rewrite root with {
    replace root with lhs;
  };
}
```

### PDLL Type System

PDLL has a static type system mirroring MLIR's runtime types:

| PDLL type | MLIR equivalent |
|-----------|----------------|
| `Value` | `mlir::Value` |
| `Op` | `mlir::Operation *` |
| `Attr` | `mlir::Attribute` |
| `Type` | `mlir::Type` |
| `ValueRange` | `mlir::ValueRange` |
| `TypeRange` | `mlir::TypeRange` |

ODS integration (`#include "...Ops.pdll"`) imports dialect op declarations, enabling the type checker to validate operand counts and attribute types at PDLL compile time.

### Named Rewrite Functions

Complex replacements can be encapsulated in named `Rewrite` declarations, which accept values/ops and return values/ops:

```pdll
// A named rewrite: given an i64 value, produce its lo/hi i32 halves
Rewrite SplitI64(val: Value) -> (Value, Value) [{
  auto i32Ty = IntegerType::get(val.getContext(), 32);
  auto loc   = val.getLoc();
  auto lo = rewriter.create<arith::TruncIOp>(loc, i32Ty, val);
  auto hi = rewriter.create<arith::ShRUIOp>(loc, i32Ty, val,
              rewriter.create<arith::ConstantOp>(loc,
                rewriter.getIntegerAttr(i32Ty, 32)));
  return {lo, hi};
}];

Pattern SplitI64Pattern {
  let src : Value;
  let root = op<my_dialect.i64op>(src);
  rewrite root with {
    let (lo, hi) = SplitI64(src);
    replace root with (lo, hi);
  };
}
```

### Erase, Replace, and Insert

The rewrite block supports three fundamental operations:

```pdll
Pattern EraseUnusedOp {
  let root = op<my_dialect.side_effect_free>(/* any args */);
  rewrite root with { erase root; };
}

Pattern InsertBeforeOp {
  let val : Value;
  let root = op<my_dialect.target>(val);
  rewrite root with {
    // Create a new op and insert it; root is untouched
    let newOp = op<my_dialect.log>(val);
    replace root with root;  // no-op replacement: keep root, log side-effects
  };
}
```

---

## 135.5 The PDLL Compilation Pipeline

### mlir-pdll: The PDLL Compiler

[`mlir-pdll`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/tools/mlir-pdll/) reads `.pdll` files and emits either C++ source or PDL dialect IR. The compilation stages are:

1. **Lexing/parsing** — [`mlir/include/mlir/Tools/PDLL/Parser/Parser.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Tools/PDLL/Parser/Parser.h): tokenize and build the PDLL AST
2. **Type checking** — validate variable types, constraint signatures, ODS constraints
3. **Code generation** — [`mlir/include/mlir/Tools/PDLL/CodeGen/CPPGen.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Tools/PDLL/CodeGen/CPPGen.h) for C++ output, [`MLIRGen.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Tools/PDLL/CodeGen/MLIRGen.h) for PDL IR output

```bash
# Emit C++ patterns (--output-format=cpp is default when -o ends in .h/.cpp)
mlir-pdll MyPatterns.pdll -o MyPatterns.h.inc

# Emit PDL dialect IR (for use with the bytecode interpreter)
mlir-pdll MyPatterns.pdll --output-format=mlir -o MyPatterns.pdl.mlir

# Include ODS context from build directory
mlir-pdll MyPatterns.pdll \
  -I /path/to/build/tools/mlir/include \
  -I /usr/lib/llvm-22/include \
  -o MyPatterns.h.inc
```

### Generated C++ Header

When emitting C++ (`--output-format=cpp`), `mlir-pdll` produces a header with:

1. One class per `Pattern` declaration, inheriting `mlir::RewritePattern`
2. A `populateWithGenerated(RewritePatternSet &)` function

```cpp
// MyPatterns.h.inc — generated by mlir-pdll
namespace {
struct FoldAddZero : public ::mlir::RewritePattern {
  FoldAddZero(::mlir::MLIRContext *context)
      : ::mlir::RewritePattern("arith.addi", 1, context,
                               {"arith.constant"}) {}
  ::mlir::LogicalResult
  matchAndRewrite(::mlir::Operation *op,
                  ::mlir::PatternRewriter &rewriter) const override;
};
} // namespace

static void populateWithGenerated(::mlir::RewritePatternSet &patterns) {
  patterns.add<FoldAddZero>(patterns.getContext());
}
```

The implementation of `matchAndRewrite` is emitted in an adjacent `.cpp.inc` file that expands the PDLL pattern logic.

### CMake Integration

CMake integration uses the `mlir_pdll_target()` function or equivalent custom commands. For in-tree dialects:

```cmake
# CMakeLists.txt for a dialect with PDLL patterns
mlir_pdll(
  MyDialectPatterns.pdll          # source
  MyDialectPatterns.h.inc         # generated header
  EXTRA_INCLUDES
    ${MLIR_INCLUDE_DIRS}
    ${CMAKE_CURRENT_SOURCE_DIR}/../include
)

add_mlir_dialect_library(MyDialect
  MyDialectPatterns.cpp
  DEPENDS
    MLIRMyDialectIncGen    # ensures .h.inc is available
)
```

### Integration with a Pass

The generated `populateWithGenerated` is called alongside any hand-written C++ patterns during pass initialization:

```cpp
#include "MyDialectPatterns.h.inc"

struct MyCanonicalizePass
    : public PassWrapper<MyCanonicalizePass, OperationPass<func::FuncOp>> {
  void runOnOperation() override {
    mlir::RewritePatternSet patterns(&getContext());
    // PDLL-generated patterns
    populateWithGenerated(patterns);
    // Hand-written C++ patterns for complex cases
    patterns.add<ComplexFoldPattern>(&getContext());

    if (failed(applyPatternsGreedily(getOperation(), std::move(patterns))))
      signalPassFailure();
  }
};
```

---

## 135.6 The PDL Bytecode Interpreter

### Architecture

When `mlir-pdll` emits PDL dialect IR rather than C++, the resulting `.mlir` file contains `pdl.pattern` operations. Before execution, these must be lowered to the `pdl_interp` dialect by the `PDLToPDLInterp` conversion pass ([`mlir/include/mlir/Conversion/PDLToPDLInterp/PDLToPDLInterp.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Conversion/PDLToPDLInterp/PDLToPDLInterp.h)). The `pdl_interp` dialect represents patterns as a bytecode-like decision tree of matching and rewriting instructions that the interpreter executes against live IR.

The primary container is `PDLPatternModule`, defined (conditionally compiled) in [`mlir/include/mlir/IR/PDLPatternMatch.h.inc`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/PDLPatternMatch.h.inc):

```cpp
class PDLPatternModule {
public:
  PDLPatternModule() = default;
  PDLPatternModule(OwningOpRef<ModuleOp> module);

  // Register a constraint function callable from PDL patterns
  void registerConstraintFunction(StringRef name, PDLConstraintFunction fn);

  // Register a rewrite function callable from pdl.apply_native_rewrite
  void registerRewriteFunction(StringRef name, PDLRewriteFunction fn);

  ModuleOp getModule();
  MLIRContext *getContext();
};
```

### Loading and Applying PDL Patterns at Runtime

This workflow enables plugin-loaded patterns — an external `.mlirbc` or `.mlir` file containing `pdl.pattern` operations can be parsed and applied without recompiling the host tool:

```cpp
#include "mlir/IR/PatternMatch.h"
#include "mlir/Conversion/PDLToPDLInterp/PDLToPDLInterp.h"

// 1. Parse the PDL module from a file
mlir::MLIRContext ctx;
ctx.loadDialect<mlir::pdl::PDLDialect,
                mlir::pdl_interp::PDLInterpDialect,
                mlir::arith::ArithDialect>();

auto pdlModuleRef = mlir::parseSourceFile<mlir::ModuleOp>(
    "FoldPatterns.pdl.mlir", &ctx);

// 2. Wrap in a PDLPatternModule and register any native constraints
mlir::PDLPatternModule pdlPatterns(std::move(pdlModuleRef));
pdlPatterns.registerConstraintFunction(
    "isZeroAttr",
    [](mlir::PatternRewriter &, mlir::PDLResultList &,
       llvm::ArrayRef<mlir::PDLValue> args) -> mlir::LogicalResult {
      auto attr = args[0].cast<mlir::Attribute>();
      auto intAttr = llvm::dyn_cast<mlir::IntegerAttr>(attr);
      return mlir::success(intAttr && intAttr.getValue().isZero());
    });

// 3. Add to a RewritePatternSet alongside C++ patterns
mlir::RewritePatternSet patterns(&ctx);
patterns.add(std::move(pdlPatterns));
patterns.add<MyHandWrittenPattern>(&ctx);

// 4. Apply
if (failed(mlir::applyPatternsGreedily(funcOp, std::move(patterns))))
  return signalPassFailure();
```

### PDL-to-PDLInterp Lowering

The `PDLToPDLInterp` pass transforms `pdl.pattern` operations into an efficient decision tree. Multiple patterns are merged into a single `pdl_interp.func` that selects the first matching pattern:

```bash
# Inspect the PDLInterp lowering:
mlir-opt --convert-pdl-to-pdl-interp FoldPatterns.pdl.mlir \
  | mlir-opt --mlir-print-ir-after-all 2>&1 | head -80
```

The `pdl_interp` operations include `pdl_interp.check_operation_name`, `pdl_interp.check_attribute`, `pdl_interp.check_operand_count`, and `pdl_interp.apply_constraint`, which form a trie-like matching structure that shares prefix checks across patterns.

---

## 135.7 Constraints and Native Code Blocks

### Constraint Declaration Forms

PDLL constraints come in two forms: purely declarative (expressed in PDLL itself) and native (backed by C++ in `[{ ... }]` blocks):

```pdll
// Pure PDLL constraint: compose existing constraints
Constraint IsSignedI32Attr(attr: Attr) :- IsI32Attr(attr), IsSignedAttr(attr);

// Native constraint with C++ body
Constraint IsMultipleOf8(attr: Attr) [{
  auto intAttr = llvm::dyn_cast<IntegerAttr>(attr);
  if (!intAttr) return failure();
  return success((intAttr.getValue().getZExtValue() % 8) == 0);
}];

// Constraint on an Op
Constraint HasOneUse(op: Op) [{
  return success(op->hasOneUse());
}];

// Constraint on a Value (matches Value objects)
Constraint IsBlockArg(val: Value) [{
  return success(llvm::isa<BlockArgument>(val));
}];
```

### Rewrite Return Types

Named `Rewrite` declarations specify return types from the PDLL type set:

```pdll
// Returns a single Value
Rewrite BuildZeroOf(val: Value) -> Value [{
  auto ty = val.getType();
  return rewriter.create<arith::ConstantOp>(
      val.getLoc(), rewriter.getZeroAttr(ty));
}];

// Returns an Op (for use as a new op reference)
Rewrite CloneWithNewAttr(op: Op, attr: Attr) -> Op [{
  auto *newOp = rewriter.clone(*op);
  newOp->setAttr("extra", attr);
  return newOp;
}];
```

### Multi-Pattern Files

A single `.pdll` file may contain any number of `Pattern`, `Constraint`, and `Rewrite` declarations. All patterns in the file are registered together when `populateWithGenerated` is called:

```pdll
#include "mlir/Dialect/Arith/IR/ArithOps.pdll"

Constraint IsZero(attr: Attr) [{ ... }];
Constraint IsOne(attr: Attr)  [{ ... }];

Pattern FoldAddZero  with benefit(1) { /* ... */ }
Pattern FoldMulOne   with benefit(1) { /* ... */ }
Pattern FoldSubSelf  with benefit(2) { /* ... */ }
```

`mlir-pdll` processes all declarations together, allowing cross-references between them. The benefit values are propagated to the generated `RewritePattern` constructors and used by the pattern driver for ordering.

---

## 135.8 PDL vs C++ RewritePattern — Decision Guide

### When to Use PDL/PDLL

The declarative systems shine for patterns that:

- Match a **single root operation** with a fixed structural neighborhood (bounded depth)
- Apply only **attribute-level predicates** or simple value equality tests
- Implement **canonicalization rules** — folding constants, removing identity ops, strength reduction
- Are numerous enough that boilerplate would dominate the file
- Need to be **loaded dynamically** (bytecode path) without recompiling the host tool

A practical heuristic: if the pattern's `matchAndRewrite` body would be fewer than 25 lines and involves no IR walking beyond the matched op's operands, it belongs in PDLL.

### When to Write C++ RewritePattern

C++ patterns are the right choice when:

- The pattern must **walk the IR** — e.g., checking all users of a value, visiting containing regions, or following def-use chains beyond immediate operands
- **Complex control flow** in matching — loops over operand lists, dynamic dispatch based on op types
- **Accumulating state** during matching — e.g., collecting all ops in a sequence for a combined replacement
- **Maximum performance is required** — the PDL bytecode interpreter adds overhead per pattern application (roughly 2–5× slower than compiled C++ for tight loops); C++ patterns avoid interpreter dispatch entirely
- The pattern needs to create **new regions** or perform **structural cloning** that requires access to `rewriter.cloneRegionBefore` and related API

### Combining Both Systems

A pass can freely mix PDL-generated and C++ patterns in the same `RewritePatternSet`. This is the recommended approach for dialects with a mix of simple canonicalizations and complex transformations:

```cpp
void populateMyDialectPatterns(mlir::RewritePatternSet &patterns) {
  // PDLL-generated: simple folds (majority of patterns)
  populateWithGenerated(patterns);
  // C++: complex structural patterns
  patterns.add<FuseConsecutiveTranspose,
               SimplifyBroadcastToSplatConst>(patterns.getContext());
}
```

The pattern driver (`applyPatternsGreedily` or `applyOpPatternsAndFold`) applies all patterns uniformly, ordering by benefit value.

---

## 135.9 Integration with mlir-opt and Passes

### Canonicalization Pattern Registration

Dialect canonicalization patterns are the canonical use case for PDLL. An ODS-defined op specifies its canonicalizer source file, and `mlir-tblgen` or `mlir-pdll` generates the `.inc` file:

```tablegen
// In ArithOps.td — tells mlir-tblgen to look for patterns in ArithPatterns.td
def Arith_AddIOp : ... {
  let hasCanonicalizer = 1;
}
```

The generated patterns are included in the canonicalization pass:

```cpp
// In ArithCanonicalization.cpp:
#include "ArithPatterns.h.inc"

void mlir::arith::AddIOp::getCanonicalizationPatterns(
    RewritePatternSet &results, MLIRContext *context) {
  populateWithGenerated(results);
}
```

### Debugging PDL Pattern Matching

When a declarative pattern does not fire as expected, the `--debug` flag traces the pattern driver's decisions:

```bash
# Trace all pattern matching attempts on a single function
mlir-opt --debug-only=rewriter mymodule.mlir --canonicalize 2>&1 | grep "PDL\|FoldAdd"

# Inspect PDL IR emitted by mlir-pdll
mlir-pdll MyPatterns.pdll --output-format=mlir | mlir-opt --mlir-print-op-generic
```

For native constraint failures, adding a temporary `llvm::errs()` call in the constraint's `[{ ... }]` body gives immediate feedback.

### The PDLL LSP Server

`mlir-pdll-lsp-server` ([`mlir/include/mlir/Tools/mlir-pdll-lsp-server/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Tools/mlir-pdll-lsp-server/)) provides IDE integration for `.pdll` files: hover documentation for op names, jump-to-definition for patterns and constraints, and inline diagnostics. Configure it in VS Code as a custom language server with the `vscode-mlir` extension or in Neovim via `nvim-lspconfig`.

---

## Chapter Summary

- **PDL (Pattern Description Language)** is an MLIR dialect that represents patterns as MLIR IR (`pdl.pattern`, `pdl.operation`, `pdl.value`, `pdl.rewrite`), enabling patterns to be processed, serialized, and interpreted like any other MLIR module.

- **ODS `Pat<>` / `Pattern<>`** in TableGen provides a DAG-based declarative pattern syntax integrated with op definitions; `NativeCodeCall<>` provides a C++ escape hatch; `mlir-tblgen -gen-rewriters` emits `populateWithGenerated` and `RewritePattern` subclasses.

- **PDLL** (`.pdll` files) is a purpose-built language with a richer type system, better error messages, and support for named `Constraint` and `Rewrite` declarations; `mlir-pdll` compiles to C++ headers or PDL dialect IR.

- **The PDL bytecode interpreter** enables dynamic pattern loading: `PDLPatternModule` wraps a PDL-containing module, native constraints are registered at runtime, and patterns are executed via the `pdl_interp` decision tree without recompiling the host tool.

- **Decision heuristic**: use PDLL/PDL for single-op structural folds and attribute predicates; use C++ `RewritePattern` when the match requires IR walking, complex control flow, or maximum performance.

- **Mixing both systems** in one `RewritePatternSet` is idiomatic and recommended: PDLL handles the majority of canonicalization cases; C++ handles the complex structural transformations.

- Key source locations: [`mlir/Dialect/PDL/IR/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/PDL/IR/), [`mlir/Tools/PDLL/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Tools/PDLL/), [`mlir/IR/PatternBase.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/PatternBase.td), [`mlir/IR/PDLPatternMatch.h.inc`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/PDLPatternMatch.h.inc).
