# Chapter 148 — Dialect Conversion

*Part XXI — MLIR Transformations*

Pattern rewriting (Chapter 147) handles within-dialect simplifications where types remain unchanged. Dialect conversion goes further: it simultaneously converts operations and their types, enabling whole-dialect translations such as `linalg` → `loops`, `scf` → `cf`, or `gpu` → `nvvm`. The conversion framework coordinates three concerns—legality (what the output must look like), type mapping (how source types become target types), and operation conversion (how each op is rewritten)—while maintaining IR consistency throughout. Getting these three concerns right is the most nuanced part of writing a lowering pass.

---

## 148.1 Why a Separate Framework

Greedy pattern rewriting cannot express type changes. When you replace a `tensor.extract` with a `memref.load`, the operand type changes from `tensor<4xf32>` to `memref<4xf32>`. If other ops still use the old `tensor<4xf32>` SSA value, the IR becomes invalid mid-transformation.

Dialect conversion solves this by:

1. **Type conversion** — running a `TypeConverter` that maps every source type to a target type.
2. **Materialization** — automatically inserting cast operations wherever a converted value is used by an unconverted consumer, or vice versa.
3. **Legality checking** — after applying all conversion patterns, verifying that the result contains only legal ops. If any illegal ops remain, the conversion is reported as failed.
4. **Atomic semantics** — the full conversion either succeeds (producing a valid, legal IR) or fails (leaving the original IR unchanged).

The framework is implemented in [`mlir/lib/Transforms/DialectConversion.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Transforms/DialectConversion.cpp) and declared in [`mlir/include/mlir/Transforms/DialectConversion.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Transforms/DialectConversion.h).

---

## 148.2 ConversionTarget

`ConversionTarget` declares which ops and dialects are legal in the output IR.

```cpp
mlir::MLIRContext ctx;
mlir::ConversionTarget target(ctx);

// All ops in LLVMDialect are legal (target dialect):
target.addLegalDialect<mlir::LLVM::LLVMDialect>();

// All ops in ArithDialect must be converted (source dialect):
target.addIllegalDialect<mlir::arith::ArithDialect>();

// Specific op is legal regardless of its dialect's status:
target.addLegalOp<mlir::func::FuncOp>();

// Specific op is illegal regardless of its dialect's status:
target.addIllegalOp<mlir::memref::AllocOp>();

// Dynamic legality: the op is legal only if the predicate holds.
target.addDynamicallyLegalOp<mlir::func::FuncOp>(
    [](mlir::func::FuncOp func) -> bool {
      // Legal only if all argument types are already LLVM types.
      return llvm::all_of(func.getArgumentTypes(), [](mlir::Type t) {
        return mlir::isa<mlir::LLVM::LLVMType>(t);
      });
    });

// Dynamic legality for entire dialect:
target.addDynamicallyLegalDialect<mlir::arith::ArithDialect>(
    [](mlir::Operation *op) {
      return llvm::none_of(op->getOperandTypes(), [](mlir::Type t) {
        return mlir::isa<mlir::MemRefType>(t);
      });
    });

// Mark all ops in a dialect as unknown (default).
// Unknown ops are not checked — they pass through unchanged.
target.addLegalDialect<mlir::BuiltinDialect>();
```

### Legality states

| State | Meaning |
|-------|---------|
| Legal | Op is acceptable in the output; never transformed |
| Illegal | Op must be converted; conversion fails if it survives |
| Dynamically legal | Op is legal only if its predicate returns true |
| Unknown (default) | Op is not checked; passes through without error |

The distinction between "unknown" and "legal" matters: the conversion framework only reports errors for ops explicitly marked illegal that remain after all patterns are applied. Unknown ops are silently ignored.

---

## 148.3 TypeConverter

`TypeConverter` maps source types to target types and handles the materialization of values across type boundaries.

```cpp
// The LLVM-specific type converter handles the standard MLIR→LLVM mapping:
mlir::LLVMTypeConverter typeConverter(&ctx);

// Add a custom type conversion rule:
typeConverter.addConversion([](mlir::MemRefType memrefTy)
    -> std::optional<mlir::Type> {
  // Unranked and ranked memrefs both map to opaque pointer in opaque-ptr mode.
  return mlir::LLVM::LLVMPointerType::get(memrefTy.getContext());
});

// Add a passthrough rule (type is already legal):
typeConverter.addConversion([](mlir::IndexType t) -> mlir::Type {
  return mlir::IntegerType::get(t.getContext(), 64);
});

// Conversions are tried in LIFO order; last-added wins for a given type.
```

### Materializations

When a value is converted from type A to type B, but some uses still expect type A, the framework must insert a **materialization cast**. There are three kinds:

```cpp
// Source materialization: used when a converted value (type B) is passed
// to an op that still expects type A. Inserts a cast B→A.
typeConverter.addSourceMaterialization(
    [](mlir::OpBuilder &builder, mlir::Type resultType,
       mlir::ValueRange inputs, mlir::Location loc)
        -> std::optional<mlir::Value> {
  if (inputs.size() != 1) return std::nullopt;
  return builder.create<mlir::UnrealizedConversionCastOp>(
      loc, resultType, inputs).getResult(0);
});

// Target materialization: used when a value of type A is passed
// to an op that now expects type B after conversion. Inserts a cast A→B.
typeConverter.addTargetMaterialization(
    [](mlir::OpBuilder &builder, mlir::Type resultType,
       mlir::ValueRange inputs, mlir::Location loc)
        -> std::optional<mlir::Value> {
  return builder.create<mlir::UnrealizedConversionCastOp>(
      loc, resultType, inputs).getResult(0);
});

// Argument materialization: used for block argument type changes.
// Converts a block argument of the new type back to the old type.
typeConverter.addArgumentMaterialization(
    [](mlir::OpBuilder &builder, mlir::Type resultType,
       mlir::ValueRange inputs, mlir::Location loc)
        -> std::optional<mlir::Value> {
  return builder.create<mlir::UnrealizedConversionCastOp>(
      loc, resultType, inputs).getResult(0);
});
```

The `UnrealizedConversionCastOp` is a placeholder that `--reconcile-unrealized-casts` later removes if both sides have been fully converted. If it remains after reconciliation, the conversion was incomplete.

---

## 148.4 Conversion Patterns

Conversion patterns are like `RewritePattern` but receive type-converted operands via an **adaptor** parameter.

### OpConversionPattern

```cpp
struct ConvertAddIToLLVM
    : mlir::OpConversionPattern<mlir::arith::AddIOp> {
  using OpConversionPattern::OpConversionPattern;

  mlir::LogicalResult matchAndRewrite(
      mlir::arith::AddIOp op,
      OpAdaptor adaptor,          // <-- converted operands
      mlir::ConversionPatternRewriter &rewriter) const override {

    // adaptor.getLhs() and adaptor.getRhs() have the converted types
    // (e.g., llvm.i32 instead of arith.i32-equivalent).
    rewriter.replaceOpWithNewOp<mlir::LLVM::AddOp>(
        op, adaptor.getLhs(), adaptor.getRhs());
    return mlir::success();
  }
};
```

The `OpAdaptor` is auto-generated by ODS for each op. It wraps `adaptor.getOperands()` with named accessors that return converted values. The original, unconverted operands are accessible via `op.getLhs()` directly; use the adaptor for operands destined for the new op.

### ConversionPatternRewriter vs PatternRewriter

`ConversionPatternRewriter` extends `PatternRewriter` with conversion-specific operations:

```cpp
// Convert region entry block argument types:
rewriter.applySignatureConversion(
    &op.getBody(),          // the region
    sigConversion,          // TypeConverter::SignatureConversion
    &typeConverter);

// Convert a function-like op's signature:
mlir::TypeConverter::SignatureConversion sig(numArgs);
sig.addInputs(argIdx, newType);           // convert one arg
sig.addInputs(argIdx, {newType1, newType2}); // one arg → two args
rewriter.applySignatureConversion(&funcOp.getBody(), sig, &typeConverter);
```

### Registering patterns

```cpp
mlir::RewritePatternSet patterns(&ctx);
patterns.add<ConvertAddIToLLVM>(typeConverter, &ctx);
// Or using a populate function (the standard approach for dialects):
mlir::arith::populateArithToLLVMConversionPatterns(typeConverter, patterns);
```

The `populate*` convention is idiomatic MLIR: each conversion pass provides a `populate` free function that adds all the conversion patterns needed for that dialect. Callers compose multiple populate functions to build the full pattern set.

---

## 148.5 Running Conversion

Three functions drive conversion, differing in failure semantics:

```cpp
// Partial conversion: apply patterns; fail only if illegal ops remain
// that could not be converted (and are explicitly marked illegal).
mlir::LogicalResult mlir::applyPartialConversion(
    mlir::Operation *op,
    const mlir::ConversionTarget &target,
    mlir::RewritePatternSet &&patterns,
    mlir::ConversionConfig config = {});

// Full conversion: every op must be converted; fail if any pattern fails.
mlir::LogicalResult mlir::applyFullConversion(
    mlir::Operation *op,
    const mlir::ConversionTarget &target,
    mlir::RewritePatternSet &&patterns,
    mlir::ConversionConfig config = {});

// Analysis conversion: dry run — reports what would happen without modifying IR.
mlir::LogicalResult mlir::applyAnalysisConversion(
    mlir::Operation *op,
    mlir::ConversionTarget &target,
    mlir::RewritePatternSet &&patterns,
    mlir::DenseSet<mlir::Operation *> &convertedOps,
    mlir::ConversionConfig config = {});
```

**Partial conversion** is the most common. Use it when the IR contains a mix of source and target dialect ops. The conversion applies patterns to all illegal ops; legal and unknown ops pass through untouched. Failure is reported only if an illegal op has no applicable pattern.

**Full conversion** is stricter: it fails if any op—even one marked "unknown"—cannot be converted. Use it for final lowering steps where the output must be entirely in the target dialect.

**Analysis conversion** is useful for testing and for the Transform dialect's `transform.apply_conversion_patterns` op. It runs the full matching and type conversion logic but writes no mutations to the IR.

---

## 148.6 Block Argument Conversion

Converting function arguments and loop induction variables requires converting block argument types—a special challenge because SSA values, not operations, carry the types.

### Signature conversion for functions

```cpp
struct ConvertFuncOpSignature
    : mlir::OpConversionPattern<mlir::func::FuncOp> {
  using OpConversionPattern::OpConversionPattern;

  mlir::LogicalResult matchAndRewrite(
      mlir::func::FuncOp funcOp,
      OpAdaptor adaptor,
      mlir::ConversionPatternRewriter &rewriter) const override {

    mlir::TypeConverter::SignatureConversion sigConv(
        funcOp.getNumArguments());

    for (auto [idx, argTy] :
         llvm::enumerate(funcOp.getArgumentTypes())) {
      mlir::Type newTy = typeConverter->convertType(argTy);
      if (!newTy) return mlir::failure();
      sigConv.addInputs(idx, newTy);
    }

    // Convert the function type itself:
    mlir::TypeConverter::SignatureConversion resultConv =
        sigConv; // copy; we may need to convert result types too
    // (omitted for brevity)

    rewriter.applySignatureConversion(&funcOp.getBody(),
                                      sigConv, typeConverter);
    return mlir::success();
  }
};
```

### One-to-many argument expansion

A source argument of type `memref<?xf32>` might become two LLVM values: a pointer (`!llvm.ptr`) and a size (`i64`). The signature conversion handles this:

```cpp
sigConv.addInputs(argIdx, {llvm::ptr_type, i64_type});
```

The framework inserts appropriate materializations so existing users of the original argument see either the combination (via a struct cast) or the individual components.

---

## 148.7 Materialization and Casts

When a value `v` has type `A` and the conversion creates a new value `v'` with type `B`, existing users that have not yet been converted still reference `v` with type `A`. The framework automatically inserts `unrealized_conversion_cast(A ← B)` to bridge the gap:

```mlir
// Before conversion:
%0 = arith.addi %a, %b : i32
%1 = my_op %0 : i32   // uses %0

// Mid-conversion (arith.addi converted, my_op not yet):
%0_new = llvm.add %a_new, %b_new : i32
%0_cast = builtin.unrealized_conversion_cast %0_new : i32 to i32
// (no-op cast in this case, but types can differ)
%1 = my_op %0_cast : i32  // still uses original type
```

After all conversion patterns run, `--reconcile-unrealized-casts` eliminates casts where source and target types match or where the cast chain can be proven to be an identity:

```bash
/usr/lib/llvm-22/bin/mlir-opt --reconcile-unrealized-casts input.mlir
```

If `unrealized_conversion_cast` ops remain after reconciliation, it indicates incomplete conversion: some type was converted on one side of a use-def edge but the consumer was never converted. This is a bug.

### ConversionConfig

```cpp
mlir::ConversionConfig config;
// Allow ops to be built in the materialization callback before the
// conversion transaction is committed:
config.buildMaterializations = true;
// Listener for observing the conversion:
config.listener = &myListener;
```

---

## 148.8 One-Shot Conversion Strategy

The idiomatic approach for a complete dialect lowering pass is:

```cpp
void runOnOperation() override {
  mlir::MLIRContext *ctx = &getContext();
  mlir::ConversionTarget target(*ctx);
  mlir::LLVMTypeConverter typeConverter(ctx);
  mlir::RewritePatternSet patterns(ctx);

  // Declare legality:
  target.addLegalDialect<mlir::LLVM::LLVMDialect>();
  target.addLegalDialect<mlir::func::FuncDialect>();  // func.func is handled specially
  target.addIllegalDialect<mlir::arith::ArithDialect>();

  // Populate all patterns needed:
  mlir::arith::populateArithToLLVMConversionPatterns(typeConverter, patterns);
  mlir::populateFuncToLLVMConversionPatterns(typeConverter, patterns);
  mlir::populateFinalizeMemRefToLLVMConversionPatterns(typeConverter, patterns);

  if (mlir::failed(mlir::applyFullConversion(
          getOperation(), target, std::move(patterns)))) {
    signalPassFailure();
  }
}
```

This is the structure of most `--convert-*-to-llvm` passes in the MLIR tree. The key properties:

- **Atomic**: either the entire module converts or the pass reports failure.
- **Composable**: multiple `populate*` calls each contribute patterns; they do not interfere because each handles distinct source ops.
- **Extensible**: add custom `populate*` calls for project-specific ops.

### Multi-step lowering

For complex dialects, a single conversion pass from source to LLVM is infeasible. The standard approach is a sequence of partial conversions, each lowering one level:

```
linalg.generic → scf.for (linalg-to-loops)
scf.for → cf.br/cf.cond_br (scf-to-cf)
cf.br + arith.* → llvm.* (convert-arith-to-llvm + convert-cf-to-llvm)
```

Each intermediate dialect serves as a stable ABI between passes, enabling independent development and testing of each lowering step.

---

## 148.9 Worked Example: arith → LLVM

Below is a minimal but complete conversion pass to illustrate the full workflow.

```cpp
#include "mlir/Transforms/DialectConversion.h"
#include "mlir/Dialect/Arith/IR/Arith.h"
#include "mlir/Dialect/LLVMIR/LLVMDialect.h"
#include "mlir/Conversion/LLVMCommon/TypeConverter.h"

namespace {

// Pattern: arith.constant → llvm.mlir.constant
struct ConvertArithConstant
    : mlir::OpConversionPattern<mlir::arith::ConstantOp> {
  using OpConversionPattern::OpConversionPattern;

  mlir::LogicalResult matchAndRewrite(
      mlir::arith::ConstantOp op, OpAdaptor,
      mlir::ConversionPatternRewriter &rewriter) const override {
    auto intAttr = mlir::dyn_cast<mlir::IntegerAttr>(op.getValue());
    if (!intAttr) return mlir::failure();
    mlir::Type newTy = typeConverter->convertType(op.getType());
    if (!newTy) return mlir::failure();
    rewriter.replaceOpWithNewOp<mlir::LLVM::ConstantOp>(op, newTy,
                                                         intAttr);
    return mlir::success();
  }
};

// Pattern: arith.addi → llvm.add
struct ConvertArithAddI
    : mlir::OpConversionPattern<mlir::arith::AddIOp> {
  using OpConversionPattern::OpConversionPattern;

  mlir::LogicalResult matchAndRewrite(
      mlir::arith::AddIOp op, OpAdaptor adaptor,
      mlir::ConversionPatternRewriter &rewriter) const override {
    rewriter.replaceOpWithNewOp<mlir::LLVM::AddOp>(
        op, adaptor.getLhs(), adaptor.getRhs());
    return mlir::success();
  }
};

struct ConvertArithToLLVMPass
    : mlir::OperationPass<mlir::ModuleOp> {
  MLIR_DEFINE_EXPLICIT_INTERNAL_INLINE_TYPE_ID(ConvertArithToLLVMPass)

  void runOnOperation() override {
    mlir::MLIRContext *ctx = &getContext();
    mlir::LLVMTypeConverter tc(ctx);
    mlir::RewritePatternSet patterns(ctx);
    patterns.add<ConvertArithConstant, ConvertArithAddI>(tc, ctx);

    mlir::ConversionTarget target(*ctx);
    target.addLegalDialect<mlir::LLVM::LLVMDialect>();
    target.addIllegalDialect<mlir::arith::ArithDialect>();

    if (mlir::failed(mlir::applyPartialConversion(
            getOperation(), target, std::move(patterns))))
      signalPassFailure();
  }
};

} // namespace
```

This pass structure—`TypeConverter` + `RewritePatternSet` + `ConversionTarget` + `applyPartialConversion`—is replicated throughout the MLIR source tree. The production arith→LLVM patterns are in [`mlir/lib/Conversion/ArithToLLVM/ArithToLLVM.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Conversion/ArithToLLVM/ArithToLLVM.cpp).

---

## Chapter Summary

- **Dialect conversion** extends pattern rewriting to handle simultaneous type and op changes, which simple `RewritePattern` cannot express.
- `ConversionTarget` declares which ops/dialects are legal, illegal, or dynamically legal in the output IR.
- `TypeConverter` maps source types to target types and specifies materialization callbacks for cross-type value uses.
- `OpConversionPattern<T>` receives type-converted operands via an `OpAdaptor`; use the adaptor, not the original op's operand accessors, when building replacement ops.
- `applyPartialConversion` is the standard driver for lowering passes; `applyFullConversion` is used when the output must be entirely in the target dialect.
- Block argument conversion uses `SignatureConversion` and `applySignatureConversion` to handle function and region argument type changes.
- `UnrealizedConversionCastOp` bridges type mismatches during multi-step conversion; `--reconcile-unrealized-casts` eliminates them after all conversions complete.
- The idiomatic pattern is: one `populate*` function per source dialect + one `ConversionTarget` + one `applyPartialConversion` call per pass.


---

@copyright jreuben11
