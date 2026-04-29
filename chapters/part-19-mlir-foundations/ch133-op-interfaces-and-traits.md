# Chapter 133 — Op Interfaces and Traits

*Part XIX — MLIR Foundations*

MLIR's extensibility creates a challenge: how can generic compiler passes work with operations they have never seen? A CSE pass needs to know which ops have no side effects. An inliner needs to understand call-like ops. A vectorizer needs to query loop induction variables. The answer is **interfaces** — protocols that operations can implement, enabling generic code to interact with any op through a common API — and **traits** — compile-time properties attached to op types that the compiler can query and verify without dispatching at runtime. Together, interfaces and traits are the mechanism by which MLIR's extensible op set can still be analyzed and transformed in a principled way.

---

## 133.1 Interface Overview

### The Interface Pattern

An interface in MLIR is a virtual dispatch mechanism that does not require inheritance. It consists of:

1. **An interface declaration** (in TableGen or C++): defines virtual method signatures
2. **An implementation**: an op declares which interfaces it implements; ODS generates the dispatch table entry
3. **A lookup mechanism**: `dyn_cast<MyInterface>(op)` returns an interface wrapper if the op implements it

This is similar to Rust traits or Java interfaces, but applied to IR operations rather than C++ objects.

### Why Not Virtual Dispatch on Operation?

MLIR could have added a `virtual` method `isReorderable()` to the base `Operation` class. But this would require every op to link against every interface's implementation, and every new interface would require recompiling all ops. MLIR's approach uses **separate linkage**: an interface and its dispatch table entry are compiled with each op that implements it; passes that use the interface link against the interface declaration only.

---

## 133.2 Defining Interfaces in TableGen

Interfaces are defined in `OpInterface` (or `TypeInterface`, `AttrInterface`) TableGen records:

```tablegen
// mlir/include/mlir/Interfaces/InferTypeOpInterface.td (simplified)
def InferTypeOpInterface : OpInterface<"InferTypeOpInterface"> {
  let description = [{
    Interface for ops that can infer their result types from operand types.
  }];

  let methods = [
    InterfaceMethod<
      /*description=*/"Infer result types from operand types",
      /*retTy=*/"::mlir::LogicalResult",
      /*methodName=*/"inferReturnTypes",
      /*args=*/(ins
        "::mlir::MLIRContext *":$context,
        "::std::optional<::mlir::Location>":$location,
        "::mlir::ValueRange":$operands,
        "::mlir::DictionaryAttr":$attributes,
        "::mlir::OpaqueProperties":$properties,
        "::mlir::RegionRange":$regions,
        "::llvm::SmallVectorImpl<::mlir::Type>&":$inferredReturnTypes
      ),
      /*defaultImpl=*/[{
        return ::mlir::failure();
      }],
      /*isStatic=*/true  // static method on the op class
    >
  ];
}
```

An op implements this interface by declaring it in its trait list:
```tablegen
def My_BinaryOp : My_Op<"binary",
    [DeclareOpInterfaceMethods<InferTypeOpInterface>]> {
  ...
}
```

And providing the implementation in C++:
```cpp
LogicalResult My::BinaryOp::inferReturnTypes(
    MLIRContext *ctx, std::optional<Location> loc,
    ValueRange operands, DictionaryAttr attrs,
    OpaqueProperties props, RegionRange regions,
    SmallVectorImpl<Type> &inferredTypes) {
  inferredTypes.push_back(operands[0].getType());
  return success();
}
```

---

## 133.3 Key Built-in Interfaces

### MemoryEffectOpInterface / SideEffectOpInterface

The most widely used interface. It allows passes to reason about what an op reads, writes, allocates, or frees.

```cpp
// mlir/include/mlir/Interfaces/SideEffectInterfaces.h
class MemoryEffectOpInterface : public OpInterface<...> {
public:
  void getEffects(
    SmallVectorImpl<SideEffects::EffectInstance<MemoryEffects::Effect>> &effects);
  bool hasEffect<EffectT>(Value value = nullptr);
  bool onlyHasEffect<EffectT>(Value value = nullptr);
};
```

Effect types:
- `MemoryEffects::Read`: reads from memory
- `MemoryEffects::Write`: writes to memory
- `MemoryEffects::Allocate`: allocates memory (malloc, alloca)
- `MemoryEffects::Free`: deallocates memory

```cpp
// Checking whether an op is pure (no effects):
if (auto effectOp = dyn_cast<MemoryEffectOpInterface>(op)) {
  SmallVector<SideEffects::EffectInstance<MemoryEffects::Effect>> effects;
  effectOp.getEffects(effects);
  if (effects.empty())
    // Op is pure → CSE/DCE eligible
}

// Simpler: use the isPure() utility:
if (mlir::isPure(&op))
  // Can hoist, CSE, eliminate
```

An op that declares `Pure` trait automatically returns no effects from this interface.

### CallOpInterface

For call-like ops (`func.call`, `llvm.call`, `fir.call`):

```cpp
class CallOpInterface : public OpInterface<...> {
public:
  // Get the symbol or value being called:
  CallInterfaceCallable getCallableForCallee();

  // Get argument operands (may differ from all operands if non-argument ops exist):
  OperandRange getArgOperands();
  MutableOperandRange getArgOperandsMutable();
};
```

```cpp
// Generic inliner:
if (auto callOp = dyn_cast<CallOpInterface>(op)) {
  auto callee = callOp.getCallableForCallee();
  if (auto symRef = dyn_cast<SymbolRefAttr>(callee)) {
    // Look up the callee function by symbol:
    auto funcOp = SymbolTable::lookupNearestSymbolFrom<func::FuncOp>(
        op, symRef.getLeafReference());
    if (funcOp && shouldInline(funcOp))
      inlineOp(callOp, funcOp);
  }
}
```

### CallableOpInterface

For function-like ops (`func.func`, `llvm.func`, `fir.func`):

```cpp
class CallableOpInterface : public OpInterface<...> {
public:
  Region *getCallableRegion();
  ArrayRef<Type> getArgumentTypes();
  ArrayRef<Type> getResultTypes();
};
```

### RegionBranchOpInterface

For control flow ops that may branch between their own regions (`scf.if`, `scf.for`, `omp.parallel`):

```cpp
class RegionBranchOpInterface : public OpInterface<...> {
public:
  // Given the current region index, get possible successors:
  void getSuccessorRegions(
      RegionBranchPoint point,
      SmallVectorImpl<RegionSuccessor> &regions);

  // Get values passed on entry to each region:
  OperandRange getEntrySuccessorOperands(RegionBranchPoint point);
};
```

Used by the dataflow analysis framework to propagate values across region boundaries.

### LoopLikeOpInterface

For loop ops (`scf.for`, `scf.while`, `affine.for`):

```cpp
class LoopLikeOpInterface : public OpInterface<...> {
public:
  // Induction variable(s):
  std::optional<Value> getSingleInductionVar();
  std::optional<OpFoldResult> getSingleLowerBound();
  std::optional<OpFoldResult> getSingleUpperBound();
  std::optional<OpFoldResult> getSingleStep();

  // Loop-invariant value motion:
  FailureOr<LoopLikeOpInterface> replaceWithAdditionalYields(
      RewriterBase &, ValueRange, bool, NewYieldValuesFn);
};
```

Used by LICM (loop-invariant code motion) and other loop optimization passes.

### InferTypeOpInterface

Ops that can infer their result types from operand types. Enables type inference in generic rewriting contexts.

### DestinationStyleOpInterface

For ops that write into pre-allocated output buffers (the linalg style):

```cpp
class DestinationStyleOpInterface : public OpInterface<...> {
public:
  // Get the input operands (read-only):
  OpOperandVector getInputs();
  // Get the output operands (read-write destinations):
  OpOperandVector getOutputs();
  // Get input/output count:
  int64_t getNumInputs();
  int64_t getNumOutputs();
};
```

Central to linalg's fusion, tiling, and bufferization.

### VectorTransferOpInterface

For vector memory transfer ops (`vector.transfer_read`, `vector.transfer_write`):

```cpp
class VectorTransferOpInterface : public OpInterface<...> {
public:
  Value getSource();
  Value getVector();
  AffineMap getPermutationMap();
  ArrayRef<bool> getInBounds();
  Value getMask();
};
```

### ShapeHelperOpInterface and InferShapedTypeOpInterface

For ops that can infer the shape of their outputs:

```cpp
class InferShapedTypeOpInterface : public OpInterface<...> {
public:
  LogicalResult inferReturnTypeComponents(
      MLIRContext *ctx,
      std::optional<Location> loc,
      ValueShapeRange operands,
      DictionaryAttr attrs,
      OpaqueProperties props,
      RegionRange regions,
      SmallVectorImpl<ShapedTypeComponents> &components);
};
```

Used by shape propagation passes and the shape dialect.

---

## 133.4 Using Interfaces in Passes

### Pattern: Generic CSE

```cpp
// A CSE pass using MemoryEffectOpInterface:
void eliminateCommonSubexpressions(Block &block) {
  DenseMap<Operation *, Operation *> opEquivalence;

  for (Operation &op : llvm::make_early_inc_range(block)) {
    // Only CSE pure ops:
    if (!mlir::isPure(&op))
      continue;

    // Check if equivalent op already exists:
    if (auto *existing = opEquivalence.lookup(&op)) {
      op.replaceAllUsesWith(existing->getResults());
      op.erase();
    } else {
      opEquivalence.insert({&op, &op});
    }
  }
}
```

### Pattern: Walk + Interface Dispatch

```cpp
module.walk([](Operation *op) {
  // Query call interface:
  if (auto callOp = dyn_cast<CallOpInterface>(op)) {
    llvm::outs() << "Call to: " << callOp.getCallableForCallee() << "\n";
    return;
  }

  // Query loop interface:
  if (auto loopOp = dyn_cast<LoopLikeOpInterface>(op)) {
    if (auto iv = loopOp.getSingleInductionVar())
      llvm::outs() << "Loop with IV: " << *iv << "\n";
    return;
  }

  // Query memory effects:
  if (mlir::isMemoryEffectFree(op))
    llvm::outs() << op->getName() << " is pure\n";
});
```

### Pattern: Type Converter with Interface

```cpp
// During dialect conversion, use ShapedType to handle memref/tensor uniformly:
converter.addConversion([](ShapedType shaped) -> Type {
  if (shaped.hasStaticShape())
    return shaped;
  // Convert dynamic dims to fully dynamic:
  SmallVector<int64_t> shape(shaped.getRank(), ShapedType::kDynamic);
  if (auto tensorTy = dyn_cast<RankedTensorType>(shaped))
    return RankedTensorType::get(shape, tensorTy.getElementType());
  if (auto memrefTy = dyn_cast<MemRefType>(shaped))
    return MemRefType::get(shape, memrefTy.getElementType());
  return nullptr;  // unhandled
});
```

---

## 133.5 Traits in Depth

Traits are **zero-cost compile-time properties** attached to op types. They appear as template parameters in `Op<dialect, name, [traits...]>`.

### How Traits Work Internally

Traits are C++ mixin classes. `Op<..., [Pure, Commutative]>` generates a class that inherits from both `Pure::Impl<ConcreteOp>` and `Commutative::Impl<ConcreteOp>`. Each mixin:
- Adds static member functions or types
- Optionally adds a `verifyTrait()` method (called during op verification)
- Optionally adds runtime behavior

### Important Traits Reference

**Memory/Side-Effect Traits**:

| Trait | Meaning |
|-------|---------|
| `Pure` | Equivalent to `NoMemoryEffect`; op has no observable side effects |
| `HasRecursiveMemoryEffects` | Op's effects = union of contained ops' effects |
| `RecursivelySpeculatable` | Contained ops may be executed speculatively |

**Type Relationship Traits**:

| Trait | Meaning |
|-------|---------|
| `SameOperandsAndResultType` | All operands and all results have the same type |
| `SameOperandsAndResultElementType` | Same element type (for shaped types) |
| `SameTypeOperands` | All operands have the same type |
| `AllTypesMatch<["a", "b"]>` | Named operands/results must share a type |
| `FirstAttrDerivedResultType` | Result type derived from first operand's type |

**Structural Traits**:

| Trait | Meaning |
|-------|---------|
| `Terminator` | Must be last op in block; has no results |
| `ReturnLike` | Like a terminator but returns values (for regions) |
| `IsolatedFromAbove` | Body regions cannot capture outer SSA values |
| `NoRegionArguments` | Regions of this op take no block arguments |
| `SingleBlock` | Each region contains exactly one block |
| `SingleBlockImplicitTerminator<T>` | Single block with implicit T terminator |
| `NoTerminator` | Regions don't need explicit terminators |

**Variadic Operand Traits**:

| Trait | Meaning |
|-------|---------|
| `AttrSizedOperandSegments` | Variadic operand segment sizes stored in `operand_segment_sizes` attr |
| `AttrSizedResultSegments` | Variadic result segment sizes stored in `result_segment_sizes` attr |

**Symbol Traits**:

| Trait | Meaning |
|-------|---------|
| `Symbol` | Op defines a named symbol (e.g., function, global) |
| `SymbolTable` | Op contains a symbol table (e.g., module) |

**Arithmetic Traits**:

| Trait | Meaning |
|-------|---------|
| `Commutative` | Result is independent of operand order |
| `Idempotent` | `f(f(x)) = f(x)` |
| `Involution` | `f(f(x)) = x` |

### AttrSizedOperandSegments Example

For ops with multiple variadic operand groups:

```tablegen
def My_OpWithVariadic : My_Op<"variadic_op",
    [AttrSizedOperandSegments]> {
  let arguments = (ins
    Variadic<I32>:$inputs,
    Variadic<F32>:$weights,
    I32:$bias
  );
  // The generated `operand_segment_sizes` DenseI32ArrayAttr records
  // [inputs.size(), weights.size(), 1] at op creation time
}
```

Without `AttrSizedOperandSegments`, MLIR cannot determine where `inputs` ends and `weights` begins in the operand list.

---

## 133.6 Implementing Custom Interfaces

### Step 1: Declare the Interface in TableGen

```tablegen
// MyInterfaces.td
def ComputeOp_Interface : OpInterface<"ComputeOpInterface"> {
  let description = "Interface for compute ops that support tiling";

  let methods = [
    InterfaceMethod<
      "Get the number of parallel loops",
      "int64_t", "getNumParallelLoops",
      (ins), // no extra args
      /*defaultImpl=*/[{
        return 0;
      }]
    >,
    InterfaceMethod<
      "Get the number of reduction loops",
      "int64_t", "getNumReductionLoops"
    >,
    InterfaceMethod<
      "Tile the op by the given tile sizes; returns the tiled op",
      "::mlir::FailureOr<::mlir::Operation *>",
      "tile",
      (ins "::mlir::OpBuilder &":$builder,
           "::llvm::ArrayRef<int64_t>":$tileSizes)
    >
  ];
}
```

### Step 2: Implement on an Op

```tablegen
def My_MatmulOp : My_Op<"matmul",
    [DeclareOpInterfaceMethods<ComputeOp_Interface,
      ["getNumParallelLoops", "getNumReductionLoops", "tile"]>]> {
  ...
}
```

```cpp
// MyOps.cpp:
int64_t My::MatmulOp::getNumParallelLoops() { return 2; }  // i, j
int64_t My::MatmulOp::getNumReductionLoops() { return 1; } // k

FailureOr<Operation *> My::MatmulOp::tile(
    OpBuilder &builder, ArrayRef<int64_t> tileSizes) {
  // Return a tiled version of this matmul
  // ...
}
```

### Step 3: Use the Interface in a Pass

```cpp
module.walk([&](Operation *op) {
  if (auto computeOp = dyn_cast<ComputeOpInterface>(op)) {
    int64_t numPar = computeOp.getNumParallelLoops();
    int64_t numRed = computeOp.getNumReductionLoops();

    if (shouldTile(numPar, numRed)) {
      auto tiled = computeOp.tile(rewriter, getTileSizes(op));
      if (succeeded(tiled))
        rewriter.replaceOp(op, {*tiled});
    }
  }
});
```

---

## 133.7 External Interface Model

Sometimes an op is defined in one dialect but needs to implement an interface from another dialect that it cannot depend on. MLIR supports **external interface models**:

```cpp
// In the pass that needs the interface:
struct ExternalMemEffectsModel
    : public MemoryEffectOpInterface::ExternalModel<
          ExternalMemEffectsModel, my::ThirdPartyOp> {
  void getEffects(Operation *op,
    SmallVectorImpl<SideEffects::EffectInstance<MemoryEffects::Effect>> &effects) const {
    // Provide effects on behalf of ThirdPartyOp
    effects.emplace_back(MemoryEffects::Read::get(),
                         cast<my::ThirdPartyOp>(op).getInput(),
                         SideEffects::DefaultResource::get());
  }
};

// Register the model when the dialect is loaded:
void myPass::initialize(MLIRContext *ctx) {
  ctx->getOrLoadDialect<my::MyDialect>()
     ->addInterfaces<ExternalMemEffectsModel>();
}
```

External models are used extensively in MLIR for cross-dialect contracts: the bufferization pass uses `BufferizableOpInterface` external models to support ops from dialects that predate bufferization.

---

## 133.8 Trait Verification

Traits can add verifier logic that runs during `mlir::verify()`:

```cpp
// From OpDefinition.h — the Pure trait's verifyTrait:
template <typename ConcreteOp>
struct Pure : public TraitBase<ConcreteOp, Pure> {
  static LogicalResult verifyTrait(Operation *op) {
    // Pure ops must not have variadic successors:
    if (op->hasTrait<mlir::OpTrait::IsTerminator>())
      return op->emitError("pure ops cannot be terminators");
    return success();
  }
};
```

When `mlir::verify(op)` runs, it calls `verifyTrait()` on every trait attached to the op. This layered verification — op-specific `verify()` + per-trait `verifyTrait()` — catches many bugs early in the compilation pipeline.

---

## Chapter Summary

- Interfaces are MLIR's protocol mechanism: an interface declares virtual method signatures; ops implement them by listing `DeclareOpInterfaceMethods<I>` in their ODS traits; passes dispatch via `dyn_cast<Interface>(op)`
- Key built-in interfaces: `MemoryEffectOpInterface` (side effects), `CallOpInterface`/`CallableOpInterface` (call/function semantics), `RegionBranchOpInterface` (multi-region control flow), `LoopLikeOpInterface` (induction variables and bounds), `InferTypeOpInterface` (result type inference), `DestinationStyleOpInterface` (linalg-style output operands)
- Traits are compile-time properties encoded as C++ mixin classes: `Pure` (no side effects), `Commutative`, `SameOperandsAndResultType`, `IsolatedFromAbove`, `Terminator`, `AttrSizedOperandSegments`, and many others; each trait may add verification logic via `verifyTrait()`
- The `isPure()` utility and `MemoryEffectOpInterface::getEffects()` are the primary tools for effect-based reasoning in passes; combining them with `walk()` enables generic CSE, DCE, and code motion
- Custom interfaces are defined in TableGen using `OpInterface` + `InterfaceMethod` records; default implementations reduce the amount of boilerplate each implementing op must provide
- External interface models allow a third-party op to implement an interface defined in a different dialect without creating a dependency; used for cross-dialect contracts in bufferization, effect modeling, and shape inference
- Trait verification is composed: MLIR's verifier calls both the op's own `verify()` method and `verifyTrait()` on every attached trait, providing layered correctness checks


---

@copyright jreuben11
