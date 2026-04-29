# Chapter 134 — The MLIR C++ API

*Part XIX — MLIR Foundations*

Writing MLIR passes, dialect implementations, and lowering pipelines requires fluency with the C++ API. The API is large and spans several subsystems: context and dialect management, the `OpBuilder` for IR construction, the `PatternRewriter` for IR transformation, the pass manager for pipeline assembly, and the walking/visiting APIs for IR inspection. This chapter is a working reference for the APIs that appear in nearly every MLIR C++ file, organized around the tasks a compiler engineer performs daily.

---

## 134.1 Context and Dialects

### MLIRContext

`MLIRContext` ([`mlir/include/mlir/IR/MLIRContext.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/MLIRContext.h)) is the root object that owns all MLIR state: type uniquing storage, dialect registry, string interner. One context per compilation unit; typically stack-allocated in `main()`.

```cpp
#include "mlir/IR/MLIRContext.h"
#include "mlir/Dialect/Arith/IR/Arith.h"
#include "mlir/Dialect/Func/IR/FuncOps.h"
#include "mlir/Dialect/LLVMIR/LLVMDialect.h"

// Create a context and load needed dialects:
mlir::MLIRContext ctx;
ctx.loadDialect<mlir::arith::ArithDialect,
                mlir::func::FuncDialect,
                mlir::LLVM::LLVMDialect>();

// Alternatively, use a registry:
mlir::DialectRegistry registry;
registry.insert<mlir::arith::ArithDialect>();
registry.insert<mlir::func::FuncDialect>();
mlir::MLIRContext ctx2(registry);
ctx2.loadAllAvailableDialects();

// Thread safety: MLIRContext is NOT thread-safe for concurrent modification.
// Use one context per thread or serialize access.
ctx.disableMultithreading();  // for debugging (disables parallel pass execution)
ctx.enableMultithreading(numThreads);
```

### Accessing the Context

```cpp
// From a type, attribute, or operation:
MLIRContext *ctx = type.getContext();
MLIRContext *ctx = attr.getContext();
MLIRContext *ctx = op->getContext();
MLIRContext *ctx = region.getContext();

// From a builder:
MLIRContext *ctx = builder.getContext();
```

---

## 134.2 Building Operations with OpBuilder

### OpBuilder Basics

`mlir::OpBuilder` ([`mlir/include/mlir/IR/Builders.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/Builders.h)) is the primary interface for creating MLIR operations. It maintains an **insertion point** (a position in the IR) and inserts new ops there.

```cpp
#include "mlir/IR/Builders.h"

// Create a builder anchored to a context (no initial insertion point):
mlir::OpBuilder builder(&ctx);

// Create a builder at the end of an existing block:
mlir::OpBuilder builder(existingBlock, existingBlock->end());

// Create a builder inside a region (at the start of its first block):
mlir::OpBuilder builder(&region, region.begin()->begin());
```

### Setting the Insertion Point

```cpp
// Before/after an operation:
builder.setInsertionPoint(op);       // insert before op
builder.setInsertionPointAfter(op);  // insert after op

// At start/end of a block:
builder.setInsertionPointToStart(block);  // first position
builder.setInsertionPointToEnd(block);    // after last op

// At the end of a region's entry block:
builder.setInsertionPointToEnd(&region.front());

// Save and restore insertion point:
auto savedPoint = builder.saveInsertionPoint();
// ... insertions ...
builder.restoreInsertionPoint(savedPoint);
```

### Creating Operations

```cpp
// Generic creation:
auto *op = builder.create<SomeOp>(
    /*loc=*/loc,
    /*resultTypes=*/TypeRange{resultTy},
    /*operands=*/ValueRange{lhs, rhs},
    /*attributes=*/ArrayRef<NamedAttribute>{});

// Convenience: ODS-generated builder (most common):
auto addOp = builder.create<arith::AddIOp>(loc, lhs, rhs);
// ↑ result type inferred from SameOperandsAndResultType

// With explicit result type:
auto constOp = builder.create<arith::ConstantOp>(
    loc, builder.getI32Type(), builder.getI32IntegerAttr(42));

// Creating block arguments:
Block *newBlock = builder.createBlock(&region);
BlockArgument arg = newBlock->addArgument(builder.getI32Type(), loc);
```

### Builder Block/Region Creation

```cpp
// Create a block inside a region:
Block *block = builder.createBlock(
    &region,
    region.end(),           // insert at end
    {builder.getI32Type()}, // block argument types
    {loc});                  // block argument locations

// Create a new block after an existing one:
Block *newBlock = builder.createBlock(
    existingBlock->getParent(),  // same region
    std::next(existingBlock->getIterator()),
    {}, {});
```

---

## 134.3 Module and Function Creation

### Building a Module from Scratch

```cpp
#include "mlir/IR/BuiltinOps.h"
#include "mlir/Dialect/Func/IR/FuncOps.h"

mlir::MLIRContext ctx;
ctx.loadDialect<mlir::func::FuncDialect, mlir::arith::ArithDialect>();

// Create the top-level module:
mlir::Location loc = mlir::UnknownLoc::get(&ctx);
auto module = mlir::ModuleOp::create(loc);

// Builder positioned at module body:
mlir::OpBuilder builder(module.getBodyRegion());

// Create a function:
auto funcType = builder.getFunctionType(
    {builder.getI32Type(), builder.getI32Type()},  // param types
    {builder.getI32Type()});                         // return type

auto funcOp = builder.create<mlir::func::FuncOp>(loc, "add", funcType);

// Create the entry block (adds block args matching func params):
mlir::Block *entryBB = funcOp.addEntryBlock();
builder.setInsertionPointToEnd(entryBB);

// Use the block arguments (= function parameters):
mlir::Value a = entryBB->getArgument(0);  // i32
mlir::Value b = entryBB->getArgument(1);  // i32

// Build the body:
auto sum = builder.create<mlir::arith::AddIOp>(loc, a, b);
builder.create<mlir::func::ReturnOp>(loc, mlir::ValueRange{sum.getResult()});
```

Printing the module:
```cpp
module.print(llvm::outs());
```

Output:
```mlir
module {
  func.func @add(%arg0: i32, %arg1: i32) -> i32 {
    %0 = arith.addi %arg0, %arg1 : i32
    return %0 : i32
  }
}
```

---

## 134.4 Type and Attribute Construction

### Common Type Builders

```cpp
// Scalar types:
mlir::Type i1  = builder.getI1Type();
mlir::Type i8  = builder.getI8Type();
mlir::Type i16 = builder.getI16Type();
mlir::Type i32 = builder.getI32Type();
mlir::Type i64 = builder.getI64Type();
mlir::Type idx = builder.getIndexType();
mlir::Type f16 = builder.getF16Type();
mlir::Type bf16 = builder.getBF16Type();
mlir::Type f32 = builder.getF32Type();
mlir::Type f64 = builder.getF64Type();

// Arbitrary integer:
mlir::Type i128 = mlir::IntegerType::get(&ctx, 128);

// Parameterized types:
mlir::Type t = mlir::RankedTensorType::get({4, 4}, builder.getF32Type());
mlir::Type m = mlir::MemRefType::get({-1, -1}, builder.getF64Type());
mlir::Type v = mlir::VectorType::get({8}, builder.getF32Type());

// Function type:
mlir::FunctionType ft = builder.getFunctionType({i32, f64}, {i32});
```

### Common Attribute Builders

```cpp
// Integer attributes:
mlir::Attribute i32_42 = builder.getI32IntegerAttr(42);
mlir::Attribute i64_n  = builder.getI64IntegerAttr(n);
mlir::Attribute idxAttr = builder.getIndexAttr(10);

// Float attributes:
mlir::Attribute pi = builder.getF64FloatAttr(3.14159);
mlir::Attribute f0 = builder.getZeroAttr(builder.getF32Type());

// String:
mlir::Attribute str = builder.getStringAttr("hello");

// Array of integers:
mlir::Attribute arr = builder.getDenseI32ArrayAttr({1, 2, 3, 4});
mlir::Attribute arr64 = builder.getDenseI64ArrayAttr({100LL, 200LL});

// Dense elements (constant tensor):
mlir::DenseElementsAttr weights = mlir::DenseElementsAttr::get(
    mlir::RankedTensorType::get({2, 2}, builder.getF32Type()),
    {1.0f, 0.0f, 0.0f, 1.0f});  // identity matrix

// Unit attribute (flag presence):
mlir::Attribute unit = mlir::UnitAttr::get(&ctx);

// Boolean:
mlir::Attribute yes = builder.getBoolAttr(true);

// NamedAttribute (for setting op attributes):
mlir::NamedAttribute named = builder.getNamedAttr("my_key", i32_42);
```

### Setting Attributes on an Op

```cpp
// After creation:
op->setAttr("threshold", builder.getF64FloatAttr(0.5));
op->setAttr("sym_name", builder.getStringAttr("my_func"));

// During creation (via attribute list):
auto op = builder.create<SomeOp>(loc,
    /*result types*/...,
    /*operands*/...,
    ArrayRef<NamedAttribute>{
        builder.getNamedAttr("step", builder.getI32IntegerAttr(4))});
```

---

## 134.5 Pattern Rewriting API

### RewritePatternSet and PatternRewriter

The pattern rewriting infrastructure ([`mlir/include/mlir/IR/PatternMatch.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/PatternMatch.h)) is how MLIR implements `matchAndRewrite` patterns — the fundamental transformation primitive.

```cpp
#include "mlir/IR/PatternMatch.h"
#include "mlir/Transforms/GreedyPatternRewriteDriver.h"

// A simple constant-folding pattern:
struct FoldAddZero : public mlir::OpRewritePattern<mlir::arith::AddIOp> {
  using OpRewritePattern::OpRewritePattern;

  mlir::LogicalResult matchAndRewrite(
      mlir::arith::AddIOp op,
      mlir::PatternRewriter &rewriter) const override {

    // Match: one operand must be a zero constant
    auto rhsConst = op.getRhs().getDefiningOp<mlir::arith::ConstantOp>();
    if (!rhsConst) return mlir::failure();

    auto intAttr = dyn_cast<mlir::IntegerAttr>(rhsConst.getValue());
    if (!intAttr || !intAttr.getValue().isZero())
      return mlir::failure();

    // Rewrite: replace the addi with just the lhs
    rewriter.replaceOp(op, op.getLhs());
    return mlir::success();
  }
};
```

### Registering and Applying Patterns

```cpp
// Collect patterns:
mlir::RewritePatternSet patterns(&ctx);
patterns.add<FoldAddZero>(&ctx);

// Add patterns from a dialect's canonicalization:
mlir::arith::AddIOp::getCanonicalizationPatterns(patterns, &ctx);

// Apply greedily (repeated until fixed point):
mlir::GreedyRewriteConfig config;
config.maxIterations = 10;  // limit iterations
(void)mlir::applyPatternsGreedily(module, std::move(patterns), config);
```

### PatternRewriter Operations

```cpp
// Inside matchAndRewrite():
// Replace op with another op (uses in the IR are updated):
rewriter.replaceOp(op, newOp->getResults());

// Replace op with a list of values:
rewriter.replaceOp(op, ValueRange{newVal1, newVal2});

// Replace uses of a specific result:
rewriter.replaceAllUsesWith(op.getResult(), newVal);

// Erase an op (must have no uses):
rewriter.eraseOp(op);

// Create a new op:
auto newAdd = rewriter.create<arith::AddIOp>(loc, a, b);

// Merge two blocks (the successor's predecessors are moved to the target):
rewriter.mergeBlocks(successor, target, successor->getArguments());

// In-place modification (when structure doesn't change):
rewriter.startOpModification(op);
op->setAttr("step", rewriter.getI32IntegerAttr(newStep));
rewriter.finalizeOpModification(op);

// Clone an op:
Operation *cloned = rewriter.clone(*op);
Operation *clonedWithMapping = rewriter.cloneWithoutRegions(*op);
```

### ConversionPattern and Dialect Conversion

For lowering passes that convert between dialects, use `ConversionPattern` + `ConversionPatternRewriter`:

```cpp
struct LowerMyAddOp : public mlir::ConversionPattern {
  LowerMyAddOp(mlir::TypeConverter &converter, mlir::MLIRContext *ctx)
      : ConversionPattern(converter, my::AddOp::getOperationName(), 1, ctx) {}

  mlir::LogicalResult matchAndRewrite(
      mlir::Operation *op,
      mlir::ArrayRef<mlir::Value> operands,
      mlir::ConversionPatternRewriter &rewriter) const override {
    auto addOp = cast<my::AddOp>(op);
    // operands are already type-converted versions of the original operands
    auto llvmAdd = rewriter.create<mlir::LLVM::AddOp>(
        op->getLoc(), operands[0], operands[1]);
    rewriter.replaceOp(op, llvmAdd->getResults());
    return mlir::success();
  }
};
```

---

## 134.6 Pass Infrastructure API

### PassManager

`mlir::PassManager` ([`mlir/include/mlir/Pass/PassManager.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Pass/PassManager.h)) orchestrates pass execution:

```cpp
#include "mlir/Pass/PassManager.h"
#include "mlir/Transforms/Passes.h"

mlir::PassManager pm(&ctx);

// Add passes at module level:
pm.addPass(mlir::createCanonicalizerPass());
pm.addPass(mlir::createCSEPass());
pm.addPass(mlir::createSymbolDCEPass());

// Nest passes under a specific op type:
mlir::OpPassManager &funcPM = pm.nest<mlir::func::FuncOp>();
funcPM.addPass(mlir::createSROAPass());
funcPM.addPass(mlir::createLoopInvariantCodeMotionPass());

// Nested nesting:
auto &innerPM = funcPM.nest<mlir::scf::ForOp>();
innerPM.addPass(createSomeLoopPass());

// Run the pipeline:
if (mlir::failed(pm.run(module))) {
  llvm::errs() << "Pass pipeline failed\n";
  return 1;
}
```

### Writing a Pass

```cpp
#include "mlir/Pass/Pass.h"

// ModuleOp-level pass:
struct MyPass : public mlir::PassWrapper<MyPass, mlir::OperationPass<mlir::ModuleOp>> {
  MLIR_DEFINE_EXPLICIT_INTERNAL_INLINE_TYPE_ID(MyPass);

  void runOnOperation() override {
    mlir::ModuleOp module = getOperation();

    // Walk the module:
    module.walk([&](mlir::arith::AddIOp addOp) {
      // ... analysis or transformation
    });

    // Signal failure:
    if (foundError)
      signalPassFailure();
  }

  // Pass statistics:
  mlir::Statistic numFolds{this, "num-folds", "Number of folds performed"};
};

// FuncOp-level pass (runs per-function in parallel by default):
struct MyFuncPass : public mlir::PassWrapper<MyFuncPass,
                           mlir::OperationPass<mlir::func::FuncOp>> {
  void runOnOperation() override {
    auto func = getOperation();
    // ...
  }
};

// Registration:
std::unique_ptr<mlir::Pass> createMyPass() {
  return std::make_unique<MyPass>();
}
```

### Pass Options

```cpp
struct MyTilingPass : public mlir::PassWrapper<MyTilingPass,
                           mlir::OperationPass<mlir::func::FuncOp>> {
  // Declare options (parsed from command line or pm string):
  mlir::Pass::Option<int64_t> tileSize{*this, "tile-size",
      llvm::cl::desc("Tile size for loop tiling"), llvm::cl::init(32)};
  mlir::Pass::ListOption<int64_t> tileSizes{*this, "tile-sizes",
      llvm::cl::desc("Per-dimension tile sizes"), llvm::cl::ZeroOrMore};

  void runOnOperation() override {
    int64_t ts = tileSize.getValue();
    // ...
  }
};
```

Using pass options from the command line:
```bash
mlir-opt --my-tiling="tile-size=64" input.mlir
```

### Pipeline Specification Strings

Passes can be specified as strings and parsed:

```cpp
// Equivalent to adding passes programmatically:
std::string pipeline = "canonicalize,cse,func.func(sroa,loop-invariant-code-motion)";
if (mlir::failed(mlir::parsePassPipeline(pipeline, pm))) {
  // Error parsing pipeline
}
```

From the command line:
```bash
mlir-opt --pass-pipeline="builtin.module(canonicalize,cse,func.func(sroa))" input.mlir
```

---

## 134.7 Walking and Visitors

### Walk API

The `walk()` method traverses all nested ops:

```cpp
// Default: post-order (children before parent), returns void:
module.walk([](mlir::Operation *op) {
  llvm::outs() << op->getName() << "\n";
});

// Walk with specific op type:
module.walk([](mlir::arith::AddIOp addOp) {
  llvm::outs() << "addi at " << addOp.getLoc() << "\n";
});

// Pre-order walk (parent before children):
module.walk<mlir::WalkOrder::PreOrder>([](mlir::func::FuncOp func) {
  llvm::outs() << "Entering function: " << func.getName() << "\n";
});

// Walk with interrupt (returns WalkResult):
mlir::WalkResult result = module.walk([](mlir::arith::DivSIOp divOp) {
  if (divOp.getRhs().getDefiningOp<mlir::arith::ConstantOp>())
    return mlir::WalkResult::advance();  // continue
  // Found a non-constant divisor — stop
  return mlir::WalkResult::interrupt();
});
bool foundNonConstDiv = result.wasInterrupted();
```

### Dominator and Use-Def Traversal

```cpp
// Iterate all uses of a value:
for (mlir::OpOperand &use : value.getUses()) {
  mlir::Operation *user = use.getOwner();
  unsigned operandIdx = use.getOperandNumber();
}

// Check if value has any uses:
bool unused = value.use_empty();
bool usedOnce = value.hasOneUse();

// Replace all uses:
value.replaceAllUsesWith(newValue);
value.replaceAllUsesExcept(newValue, except_op);

// Get defining op:
if (auto *defOp = value.getDefiningOp()) {
  // value is an OpResult
  auto addOp = dyn_cast<arith::AddIOp>(defOp);
}
// value might be a block argument:
if (auto blockArg = dyn_cast<mlir::BlockArgument>(value)) {
  unsigned argIdx = blockArg.getArgNumber();
  Block *block = blockArg.getOwner();
}
```

---

## 134.8 The mlir-opt Tool

`mlir-opt` is the workhorse testing tool for MLIR development:

### Basic Usage

```bash
# Verify and round-trip a textual MLIR file:
mlir-opt input.mlir

# Apply a single pass:
mlir-opt --canonicalize input.mlir

# Apply a pipeline:
mlir-opt \
  --canonicalize \
  --cse \
  --convert-linalg-to-loops \
  --convert-scf-to-cf \
  --convert-cf-to-llvm \
  --convert-arith-to-llvm \
  --convert-func-to-llvm \
  --reconcile-unrealized-casts \
  input.mlir -o output.mlir

# Print IR after each pass:
mlir-opt --mlir-print-ir-after-all input.mlir 2>&1 | less

# Print IR before and after a specific pass:
mlir-opt --mlir-print-ir-before=canonicalize \
         --mlir-print-ir-after=canonicalize input.mlir

# Print module in generic form (without syntactic sugar):
mlir-opt --mlir-print-op-generic input.mlir

# Time each pass:
mlir-opt --mlir-timing input.mlir
```

### FileCheck Testing with mlir-opt

```mlir
// RUN: mlir-opt %s --canonicalize | FileCheck %s

func.func @fold_add_zero(%x: i32) -> i32 {
  %zero = arith.constant 0 : i32
  %result = arith.addi %x, %zero : i32
  return %result : i32
}
// CHECK-LABEL: func @fold_add_zero
// CHECK-NEXT:  return %arg0
// CHECK-NOT:   arith.constant
// CHECK-NOT:   arith.addi
```

### Translating to LLVM IR

```bash
# Full lowering to LLVM IR:
mlir-opt \
  --convert-arith-to-llvm \
  --convert-func-to-llvm \
  --reconcile-unrealized-casts \
  input.mlir | \
mlir-translate --mlir-to-llvmir - -o output.ll

# Compile to object:
llc -filetype=obj output.ll -o output.o
```

---

## 134.9 Diagnostics

```cpp
// Emit an error on an op (InFlightDiagnostic):
op->emitError("unexpected type: ") << op->getResultTypes()[0];

// Emit a warning:
op->emitWarning("this pattern is deprecated");

// Emit a note (attached to a prior diagnostic):
op->emitNote("see also: ") << relatedOp->getLoc();

// Emit a remark (informational):
op->emitRemark("inlining this call site");

// During pass:
auto &diag = op->emitError("invalid combination of flags");
diag.attachNote(relatedOp->getLoc()) << "conflicting flag here";
return signalPassFailure();

// Emit from a Location directly:
mlir::emitError(loc, "failed to infer type");

// Custom diagnostic handler:
ctx.getDiagEngine().registerHandler([](mlir::Diagnostic &diag) {
  llvm::errs() << "DIAG: " << diag << "\n";
  return mlir::success();
});
```

---

## Chapter Summary

- `MLIRContext` owns all MLIR state; dialects are loaded via `ctx.loadDialect<D>()` or `DialectRegistry`; each context has a unique type storage pool
- `OpBuilder` maintains an insertion point and creates ops via `builder.create<OpType>(loc, ...)`;  insertion point methods (`setInsertionPoint`, `setInsertionPointToEnd`) control where new ops land
- `ModuleOp::create()` starts a module; `builder.create<func::FuncOp>()` creates functions; `func.addEntryBlock()` adds the function body and returns the block with arguments matching function parameters
- Type construction uses `builder.getI32Type()`, `RankedTensorType::get(shape, elemTy)`, `MemRefType::get(shape, elemTy)`; attribute construction uses `builder.getI32IntegerAttr(n)`, `DenseElementsAttr::get(type, data)`, `getDenseI64ArrayAttr()`
- `OpRewritePattern<OpTy>` + `matchAndRewrite()` is the fundamental transformation pattern; `PatternRewriter` provides `replaceOp`, `eraseOp`, `create`, `mergeBlocks`; `applyPatternsGreedily()` drives convergence
- `PassManager` assembles pipelines with `addPass()` and `nest<OpTy>()`; `pass.run(module)` executes; `PassWrapper` is the base for custom passes; `Pass::Option<T>` adds command-line-configurable parameters
- `walk()` traverses nested ops (pre/post-order, with optional early exit via `WalkResult`); value use-def traversal uses `value.getUses()`, `value.getDefiningOp()`, `value.replaceAllUsesWith()`
- `mlir-opt` is the canonical testing and development tool; combined with FileCheck, it enables precise IR transformation tests; `mlir-translate --mlir-to-llvmir` produces LLVM IR for final compilation


---

@copyright jreuben11
