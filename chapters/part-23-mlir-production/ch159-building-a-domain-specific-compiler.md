# Chapter 159 — Building a Domain-Specific Compiler

*Part XXIII — MLIR in Production*

MLIR's real power emerges when you build a complete domain-specific compiler: a dialect that captures your problem domain's semantics, a lowering pipeline that transforms those semantics progressively to hardware-efficient code, and the testing and debugging infrastructure to make it maintainable. This chapter walks through the complete process of building an out-of-tree dialect and compiler using MLIR 22 infrastructure. The canonical template is the `standalone` example in `mlir/examples/standalone/`, but a production compiler needs substantially more: proper ODS definitions, multi-stage lowering, integration with the LLVM pipeline, comprehensive FileCheck tests, and a compelling reason to exist. The chapter uses a running example — a "Matrix Algebra" (MatAlg) dialect for a scientific computing DSL — to ground each concept.

---

## 159.1 Why Build a Domain-Specific Compiler?

General-purpose compilers optimize well for general programs. Domain-specific compilers can exploit invariants that a general compiler cannot assume. For a matrix algebra DSL:

- **Static shapes**: all matrices have compile-time-known dimensions; no bounds checks needed
- **Value semantics**: immutable-by-default tensors with in-place optimization as a pass
- **Named operations**: `matvec`, `gemm`, `solve`, `svd` have mathematical identities that justify specialized rewrites
- **Target-specific backends**: route `gemm` to cuBLAS on GPU, BLAS on CPU, or hand-tuned SIMD

A DSL compiler can express and optimize these concerns without polluting a general-purpose IR.

---

## 159.2 Project Structure

Following the `standalone` template with an out-of-tree build:

```
matalg-compiler/
├── CMakeLists.txt
├── include/
│   └── matalg/
│       ├── IR/
│       │   ├── MatAlgDialect.h
│       │   ├── MatAlgOps.h
│       │   ├── MatAlgOps.td          ← ODS definition
│       │   └── MatAlgDialect.td
│       └── Transforms/
│           └── Passes.h
├── lib/
│   ├── IR/
│   │   ├── MatAlgDialect.cpp
│   │   └── MatAlgOps.cpp
│   └── Transforms/
│       ├── LowerMatAlgToLinalg.cpp
│       └── LowerLinalgToLLVM.cpp
├── tools/
│   └── matalg-opt/
│       └── matalg-opt.cpp
└── test/
    ├── lit.cfg.py
    ├── IR/
    │   └── basic.mlir
    └── Transforms/
        ├── lower-to-linalg.mlir
        └── lower-to-llvm.mlir
```

### 159.2.1 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(MatAlgCompiler)

# Find MLIR installation
find_package(MLIR REQUIRED CONFIG
    PATHS /usr/lib/llvm-22/lib/cmake/mlir)

# Include MLIR's CMake modules
include(AddMLIR)
include(TableGen)

add_subdirectory(include)
add_subdirectory(lib)
add_subdirectory(tools)
add_subdirectory(test)
```

---

## 159.3 Defining the Dialect with ODS

ODS (Operation Definition Specification) uses TableGen to generate boilerplate-free C++ dialect and op definitions.

### 159.3.1 Dialect Definition

```tablegen
// include/matalg/IR/MatAlgDialect.td
include "mlir/IR/BuiltinDialect.td"

def MatAlg_Dialect : Dialect {
  let name = "matalg";
  let summary = "Matrix Algebra domain-specific dialect";
  let cppNamespace = "::mlir::matalg";
  
  // Use default type/attribute printing
  let useDefaultTypePrinterParser = 1;
  let useDefaultAttributePrinterParser = 1;
}
```

### 159.3.2 Type Definition

```tablegen
// include/matalg/IR/MatAlgTypes.td
include "mlir/IR/BuiltinTypes.td"

// MatrixType: statically-shaped 2D float matrix
def MatAlg_MatrixType : TypeDef<MatAlg_Dialect, "Matrix"> {
  let mnemonic = "matrix";
  let summary = "Statically-shaped 2D float matrix";
  let parameters = (ins
    "int64_t":$rows,
    "int64_t":$cols,
    "FloatType":$elementType
  );
  let assemblyFormat = "`<` $rows `x` $cols `x` $elementType `>`";
  
  let builders = [
    TypeBuilder<(ins "int64_t":$rows, "int64_t":$cols), [{
      return Base::get($_ctxt, rows, cols, Float32Type::get($_ctxt));
    }]>
  ];
}
```

This generates `MatrixType` with `getRows()`, `getCols()`, `getElementType()` accessors, and a parser that handles `!matalg.matrix<4x8xf32>`.

### 159.3.3 Op Definition

```tablegen
// include/matalg/IR/MatAlgOps.td
include "mlir/IR/OpBase.td"
include "mlir/Interfaces/SideEffectInterfaces.td"
include "mlir/Interfaces/InferTypeOpInterface.td"

class MatAlg_Op<string mnemonic, list<Trait> traits = []>
    : Op<MatAlg_Dialect, mnemonic, traits>;

// Matrix multiply: C = A * B
def MatAlg_GemmOp : MatAlg_Op<"gemm",
    [Pure, InferTypeOpAdaptor]> {
  let summary = "General matrix multiplication: C = alpha * A * B + beta * C_in";
  let arguments = (ins
    MatAlg_MatrixType:$a,
    MatAlg_MatrixType:$b,
    Optional<MatAlg_MatrixType>:$c_in,
    F64Attr:$alpha,
    F64Attr:$beta
  );
  let results = (outs MatAlg_MatrixType:$result);
  
  let assemblyFormat = [{
    $a `,` $b (`,` $c_in^)? `alpha` `=` $alpha `beta` `=` $beta
    attr-dict `:` type($a) `,` type($b) `->` type($result)
  }];
  
  // Custom verifier
  let hasVerifier = 1;
  
  // Canonicalization patterns
  let hasCanonicalizer = 1;
}

// Element-wise ops
def MatAlg_AddOp : MatAlg_Op<"add", [Pure, SameOperandsAndResultType]> {
  let arguments = (ins MatAlg_MatrixType:$lhs, MatAlg_MatrixType:$rhs);
  let results = (outs MatAlg_MatrixType:$result);
  let assemblyFormat = "$lhs `,` $rhs attr-dict `:` type($result)";
}

// Transpose
def MatAlg_TransposeOp : MatAlg_Op<"transpose", [Pure]> {
  let arguments = (ins MatAlg_MatrixType:$input);
  let results = (outs MatAlg_MatrixType:$result);
  let hasVerifier = 1;
  let assemblyFormat = "$input attr-dict `:` type($input) `->` type($result)";
}
```

### 159.3.4 CMake TableGen Integration

```cmake
# include/matalg/IR/CMakeLists.txt
mlir_tablegen(MatAlgOps.h.inc -gen-op-decls)
mlir_tablegen(MatAlgOps.cpp.inc -gen-op-defs)
mlir_tablegen(MatAlgDialect.h.inc -gen-dialect-decls)
mlir_tablegen(MatAlgDialect.cpp.inc -gen-dialect-defs)
mlir_tablegen(MatAlgTypes.h.inc -gen-typedef-decls)
mlir_tablegen(MatAlgTypes.cpp.inc -gen-typedef-defs)

add_public_tablegen_target(MatAlgOpsIncGen)
add_dependencies(mlir-headers MatAlgOpsIncGen)
```

---

## 159.4 Implementing the Dialect in C++

### 159.4.1 Dialect Registration

```cpp
// lib/IR/MatAlgDialect.cpp
#include "matalg/IR/MatAlgDialect.h"
#include "matalg/IR/MatAlgOps.h"

// Include generated .inc files
#include "matalg/IR/MatAlgDialect.cpp.inc"

using namespace mlir;
using namespace mlir::matalg;

void MatAlgDialect::initialize() {
  addOperations<
#define GET_OP_LIST
#include "matalg/IR/MatAlgOps.cpp.inc"
  >();
  addTypes<
#define GET_TYPEDEF_LIST
#include "matalg/IR/MatAlgTypes.cpp.inc"
  >();
}
```

### 159.4.2 Op Verifier

```cpp
// lib/IR/MatAlgOps.cpp
LogicalResult GemmOp::verify() {
  auto aType = cast<MatrixType>(getA().getType());
  auto bType = cast<MatrixType>(getB().getType());
  auto resultType = cast<MatrixType>(getResult().getType());
  
  // Contracting dimension check
  if (aType.getCols() != bType.getRows())
    return emitOpError("inner dimensions must match: ")
        << aType << " * " << bType;
  
  // Result shape check
  if (resultType.getRows() != aType.getRows() ||
      resultType.getCols() != bType.getCols())
    return emitOpError("result shape mismatch");
    
  // Optional C_in shape check
  if (auto c = getCIn()) {
    auto cType = cast<MatrixType>(c.getType());
    if (cType != resultType)
      return emitOpError("C_in must match result type");
  }
  
  return success();
}
```

### 159.4.3 Canonicalization

```cpp
// Canonicalize: transpose(transpose(A)) → A
struct TransposeOfTranspose : public OpRewritePattern<TransposeOp> {
  using OpRewritePattern::OpRewritePattern;
  
  LogicalResult matchAndRewrite(TransposeOp op,
                                PatternRewriter& rewriter) const override {
    auto inputOp = op.getInput().getDefiningOp<TransposeOp>();
    if (!inputOp)
      return failure();
    
    rewriter.replaceOp(op, inputOp.getInput());
    return success();
  }
};

void TransposeOp::getCanonicalizationPatterns(
    RewritePatternSet& patterns, MLIRContext* ctx) {
  patterns.add<TransposeOfTranspose>(ctx);
}
```

---

## 159.5 Writing the Lowering Pipeline

The lowering strategy: MatAlg → Linalg → Affine/SCF → LLVM IR.

### 159.5.1 Pass Definition with ODS

```tablegen
// include/matalg/Transforms/Passes.td
def LowerMatAlgToLinalg : Pass<"lower-matalg-to-linalg", "func::FuncOp"> {
  let summary = "Lower MatAlg ops to Linalg named ops";
  let constructor = "mlir::matalg::createLowerMatAlgToLinalgPass()";
  let dependentDialects = [
    "mlir::linalg::LinalgDialect",
    "mlir::tensor::TensorDialect",
    "mlir::arith::ArithDialect"
  ];
}
```

### 159.5.2 MatAlg → Linalg Lowering

```cpp
// lib/Transforms/LowerMatAlgToLinalg.cpp
struct GemmOpLowering : public OpConversionPattern<matalg::GemmOp> {
  using OpConversionPattern::OpConversionPattern;
  
  LogicalResult matchAndRewrite(
      matalg::GemmOp op, OpAdaptor adaptor,
      ConversionPatternRewriter& rewriter) const override {
    auto loc = op.getLoc();
    auto aType = cast<matalg::MatrixType>(op.getA().getType());
    auto bType = cast<matalg::MatrixType>(op.getB().getType());
    
    // Convert matrix type to tensor type
    auto tensorType = RankedTensorType::get(
        {aType.getRows(), bType.getCols()},
        aType.getElementType());
    
    // Create zero-initialized result tensor
    Value init = rewriter.create<tensor::EmptyOp>(
        loc, ArrayRef<int64_t>{aType.getRows(), bType.getCols()},
        aType.getElementType());
    
    // Apply alpha scaling to A (if alpha != 1.0)
    Value a = adaptor.getA();
    if (op.getAlpha() != 1.0) {
      Value alpha = rewriter.create<arith::ConstantOp>(
          loc, rewriter.getF32FloatAttr(op.getAlpha()));
      // Use linalg.map to scale
      a = emitScalarMap(rewriter, loc, a, [&](Value elem) {
        return rewriter.create<arith::MulFOp>(loc, elem, alpha);
      });
    }
    
    // Lower to linalg.matmul
    Value result = rewriter.create<linalg::MatmulOp>(
        loc, TypeRange{tensorType},
        ValueRange{a, adaptor.getB()},
        ValueRange{init}).getResult(0);
    
    // Add beta * C_in if present
    if (auto c = adaptor.getCIn()) {
      // linalg.map: result = result + beta * c
      result = emitAddScaled(rewriter, loc, result, c, op.getBeta());
    }
    
    rewriter.replaceOp(op, result);
    return success();
  }
};
```

### 159.5.3 Type Converter

The `TypeConverter` maps MatAlg types to their Linalg equivalents:

```cpp
class MatAlgTypeConverter : public TypeConverter {
 public:
  MatAlgTypeConverter() {
    addConversion([](Type t) { return t; });  // passthrough for non-MatAlg types
    addConversion([](matalg::MatrixType mt) -> Type {
      return RankedTensorType::get(
          {mt.getRows(), mt.getCols()}, mt.getElementType());
    });
  }
};
```

### 159.5.4 Registering the Conversion Pass

```cpp
void createLowerMatAlgToLinalgPass(PassManager& pm) {
  MatAlgTypeConverter typeConverter;
  ConversionTarget target(*pm.getContext());
  
  target.addIllegalDialect<matalg::MatAlgDialect>();
  target.addLegalDialect<linalg::LinalgDialect>();
  target.addLegalDialect<tensor::TensorDialect>();
  target.addLegalDialect<arith::ArithDialect>();
  target.addLegalDialect<func::FuncDialect>();
  
  RewritePatternSet patterns(pm.getContext());
  patterns.add<GemmOpLowering, AddOpLowering, TransposeOpLowering>(
      typeConverter, pm.getContext());
  
  pm.addPass(std::make_unique<LowerMatAlgToLinalgPass>(
      std::move(typeConverter), std::move(target), std::move(patterns)));
}
```

---

## 159.6 The matalg-opt Driver

```cpp
// tools/matalg-opt/matalg-opt.cpp
#include "mlir/Tools/mlir-opt/MlirOptMain.h"
#include "mlir/InitAllDialects.h"
#include "mlir/InitAllPasses.h"
#include "matalg/IR/MatAlgDialect.h"
#include "matalg/Transforms/Passes.h"

int main(int argc, char** argv) {
  mlir::DialectRegistry registry;
  
  // Register all built-in MLIR dialects
  mlir::registerAllDialects(registry);
  
  // Register our dialect
  registry.insert<mlir::matalg::MatAlgDialect>();
  
  // Register all built-in passes
  mlir::registerAllPasses();
  
  // Register our passes
  mlir::matalg::registerMatAlgPasses();
  
  return mlir::MlirOptMain(argc, argv, "MatAlg optimizer driver\n", registry)
             .succeeded() ? 0 : 1;
}
```

Build and use:

```bash
# Build
cmake -B build -DMLIR_DIR=/usr/lib/llvm-22/lib/cmake/mlir .
cmake --build build --target matalg-opt

# Run passes
./build/bin/matalg-opt \
    --lower-matalg-to-linalg \
    --convert-linalg-to-affine-loops \
    --affine-loop-tile="tile-size=64" \
    --convert-affine-to-scf \
    --convert-scf-to-cf \
    --convert-cf-to-llvm \
    --convert-func-to-llvm \
    --reconcile-unrealized-casts \
    input.mlir
```

---

## 159.7 Testing with FileCheck

### 159.7.1 Unit Test Structure

```mlir
// test/Transforms/lower-to-linalg.mlir
// RUN: matalg-opt --lower-matalg-to-linalg %s | FileCheck %s

func.func @test_gemm(%a: !matalg.matrix<4x8xf32>,
                     %b: !matalg.matrix<8x16xf32>) 
    -> !matalg.matrix<4x16xf32> {
  %result = matalg.gemm %a, %b alpha=1.0 beta=0.0
      : !matalg.matrix<4x8xf32>, !matalg.matrix<8x16xf32>
      -> !matalg.matrix<4x16xf32>
  return %result : !matalg.matrix<4x16xf32>
}

// CHECK-LABEL: func.func @test_gemm
// CHECK:       %[[INIT:.*]] = tensor.empty() : tensor<4x16xf32>
// CHECK:       %[[RESULT:.*]] = linalg.matmul
// CHECK-SAME:    ins(%{{.*}}, %{{.*}} : tensor<4x8xf32>, tensor<8x16xf32>)
// CHECK-SAME:    outs(%[[INIT]] : tensor<4x16xf32>)
// CHECK:       return %[[RESULT]]
```

### 159.7.2 Negative Tests

```mlir
// test/IR/verify.mlir
// RUN: matalg-opt %s --verify-diagnostics

func.func @bad_gemm(%a: !matalg.matrix<4x8xf32>,
                    %b: !matalg.matrix<16x4xf32>) {
  // expected-error @+1 {{inner dimensions must match}}
  %result = matalg.gemm %a, %b alpha=1.0 beta=0.0
      : !matalg.matrix<4x8xf32>, !matalg.matrix<16x4xf32>
      -> !matalg.matrix<4x4xf32>
  return
}
```

### 159.7.3 Integration Tests

Full pipeline tests that verify the final LLVM IR:

```mlir
// test/Transforms/full-pipeline.mlir
// RUN: matalg-opt --lower-matalg-full-pipeline %s | mlir-translate --mlir-to-llvmir | FileCheck %s

func.func @matmul_4x4(%a: !matalg.matrix<4x4xf32>,
                       %b: !matalg.matrix<4x4xf32>)
    -> !matalg.matrix<4x4xf32> {
  %result = matalg.gemm %a, %b alpha=1.0 beta=0.0
      : !matalg.matrix<4x4xf32>, !matalg.matrix<4x4xf32>
      -> !matalg.matrix<4x4xf32>
  return %result : !matalg.matrix<4x4xf32>
}

// CHECK: define <{{.*}}> @matmul_4x4
// CHECK: fmul
// CHECK: fadd
```

---

## 159.8 Debugging Infrastructure

### 159.8.1 IR Printing Hooks

```bash
# Print IR after every pass
matalg-opt --mlir-print-ir-after-all input.mlir > /dev/null

# Print IR only after specific pass
matalg-opt --mlir-print-ir-after=lower-matalg-to-linalg input.mlir

# Print IR in generic form (full attribute detail)
matalg-opt --mlir-print-generic-form --lower-matalg-to-linalg input.mlir
```

### 159.8.2 Crash Reproducers

When a pass crashes on real input, `mlir-reduce` minimizes the input to the smallest reproducer:

```bash
# Check that the pass crashes on this input:
matalg-opt --lower-matalg-to-linalg large_model.mlir 2>&1 | grep "Segmentation fault"

# Generate reproducer
matalg-opt --lower-matalg-to-linalg \
    --mlir-print-op-on-diagnostic large_model.mlir 2> crash.mlir

# Minimize the reproducer
mlir-reduce --test-fn="matalg-opt --lower-matalg-to-linalg" crash.mlir
```

### 159.8.3 Verifier Integration

MLIR's verifier (`mlir::verify()`) checks invariants after every pass when `--mlir-verify-diagnostics` or `MLIR_DEBUG_VERIFY_PASSES=1` is set. Add custom verification in `verifyInvariants()`:

```cpp
// In the pass implementation:
struct LowerMatAlgToLinalgPass : ... {
  void runOnOperation() override {
    // ... run conversion ...
    
    // Verify no MatAlg ops remain
    auto op = getOperation();
    op.walk([&](matalg::GemmOp gemm) {
      gemm.emitError("GemmOp was not lowered");
      signalPassFailure();
    });
  }
};
```

---

## 159.9 Integrating the Full LLVM Pipeline

To produce actual executables, the MatAlg compiler needs to lower all the way to LLVM IR and invoke `llc`:

```cpp
// A complete compilation function
mlir::LogicalResult compileMatAlg(
    mlir::MLIRContext* ctx,
    llvm::StringRef input_path,
    llvm::StringRef output_path) {
  // Parse the input
  auto module = mlir::parseSourceFile<mlir::ModuleOp>(input_path, ctx);
  if (!module) return mlir::failure();
  
  // Build the full pass pipeline
  mlir::PassManager pm(ctx);
  pm.addPass(mlir::matalg::createLowerMatAlgToLinalgPass());
  pm.addNestedPass<mlir::func::FuncOp>(
      mlir::createLinalgGeneralizationPass());
  pm.addNestedPass<mlir::func::FuncOp>(
      mlir::createConvertLinalgToAffineLoopsPass());
  pm.addNestedPass<mlir::func::FuncOp>(
      mlir::affine::createAffineLoopNormalizePass());
  pm.addNestedPass<mlir::func::FuncOp>(
      mlir::affine::createLoopTilingPass(/*tileSize=*/64));
  pm.addNestedPass<mlir::func::FuncOp>(
      mlir::affine::createAffineVectorizePass({8}));
  pm.addPass(mlir::createConvertAffineToSCFPass());
  pm.addPass(mlir::createConvertSCFToCFPass());
  pm.addPass(mlir::createConvertFuncToLLVMPass());
  pm.addPass(mlir::createConvertArithToLLVMPass());
  pm.addPass(mlir::createConvertCFToLLVMPass());
  pm.addPass(mlir::createReconcileUnrealizedCastsPass());
  
  if (mlir::failed(pm.run(*module)))
    return mlir::failure();
  
  // Translate to LLVM IR and emit object file
  llvm::LLVMContext llvmCtx;
  auto llvmModule = mlir::translateModuleToLLVMIR(*module, llvmCtx);
  
  // Invoke LLVM backend
  llvm::InitializeNativeTarget();
  llvm::InitializeNativeTargetAsmPrinter();
  // ... target machine creation and object emission ...
  
  return mlir::success();
}
```

---

## 159.10 Dialect Design Principles

Lessons learned from building production DSL compilers with MLIR:

1. **Keep ops high-level**: resist the temptation to add micro-ops. `matalg.gemm` is better than `matalg.dot_product_with_accumulate`. High-level ops carry more information for optimization.

2. **Use interfaces liberally**: implement `InferTypeOpInterface`, `MemoryEffects`, `LoopLikeOpInterface` so your ops participate in MLIR's generic passes (CSE, DCE, loop analysis).

3. **Separate concerns into multiple lowering stages**: MatAlg → Linalg → Affine → SCF → LLVM is better than one mega-lowering. Each stage can be independently tested.

4. **FileCheck tests for every pass**: test the output of every lowering. Regressions in lowering quality are otherwise invisible.

5. **Make the dialect roundtrippable**: every op must have an assembly format that parses and prints bijectively. Use `--mlir-print-ir-after-all` and verify the output parses back.

6. **Version your bytecode early**: if the dialect will serialize models, add a version attribute from day one. Migrating an unversioned format is painful.

---

## Chapter Summary

- An out-of-tree MLIR dialect requires: ODS TableGen definitions, dialect registration, op verifiers, CMake integration with `mlir_tablegen`, and a driver binary.
- ODS generates C++ headers, source includes, and parser/printer code; `assemblyFormat` strings handle common patterns without hand-written parsers.
- The lowering pipeline proceeds in stages: domain dialect → Linalg (named ops) → Affine (explicit loops) → SCF → CF → LLVM dialect → `mlir-translate` → LLVM IR → `llc`.
- `TypeConverter` maps domain types to lowering-target types; `ConversionTarget` specifies which ops are legal at each stage.
- FileCheck tests verify every lowering stage; `--mlir-print-ir-after-all` and `mlir-reduce` are the primary debugging tools.
- The `matalg-opt` driver uses `MlirOptMain` to get standard option parsing, pass pipeline registration, and diagnostic infrastructure for free.
- Dialect design principles: keep ops high-level, implement interfaces, use multiple lowering stages, and add versioning from the start.
