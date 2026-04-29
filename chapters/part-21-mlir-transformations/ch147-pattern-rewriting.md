# Chapter 147 — Pattern Rewriting

*Part XXI — MLIR Transformations*

Pattern rewriting is the primary mechanism through which MLIR transforms programs. Rather than writing monolithic traversal passes that hand-roll every IR mutation, MLIR encodes individual simplification rules as `RewritePattern` objects and delegates driver logic to a general-purpose engine that applies them to fixpoint. This separation of concerns—what to rewrite versus how to schedule rewrites—enables composable, independently testable transformation units that combine safely without coordination. Understanding pattern rewriting in depth is prerequisite to productive work on any MLIR-based compiler: canonicalization, dialect lowering, vectorization, and loop transformations all build on the same foundation.

---

## 147.1 The Rewriting Model

MLIR's transformation model treats the IR as a directed acyclic graph of operations connected by SSA def-use edges. A transformation identifies a subgraph matching some pattern and replaces it with a semantically equivalent subgraph. Repeating this process until no further patterns apply reaches a **fixpoint** representing the fully simplified or lowered program.

The key types are:

- [`RewritePattern`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/PatternMatch.h) — abstract base class; a pattern declares its root operation type, a match function, and a rewrite action.
- [`PatternRewriter`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/PatternMatch.h) — a subclass of `OpBuilder` that the rewrite action uses to mutate the IR. The rewriter records all mutations so the driver can roll back a failed match without corrupting the IR.
- [`RewritePatternSet`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/PatternMatch.h) — a collection of patterns passed to a driver.
- The **greedy driver** (`applyPatternsGreedily`) — iterates until fixpoint, maintaining a worklist of operations that may still benefit from patterns.

### Why pattern rewriting rather than explicit traversals?

Hand-written traversals couple the what (simplification logic) to the how (IR walk order, worklist management, incremental update). Patterns separate these concerns:

1. **Composability** — patterns from different passes can be merged into a single driver invocation, eliminating redundant traversals.
2. **Order independence** — each pattern makes a locally sound decision; the driver handles global scheduling.
3. **Incremental correctness** — the driver re-adds only ops that were created or modified, avoiding redundant work.
4. **Testability** — each `RewritePattern` is a standalone class, unit-testable in isolation via `applyPatternsGreedily` on a small op tree.

### Pattern benefit

Every `RewritePattern` carries a `PatternBenefit` — a non-negative integer expressing how profitable the pattern is relative to others that match the same root op. The greedy driver orders candidates by descending benefit, trying the highest-benefit pattern first. In most cases the default benefit (1) suffices; set benefit 0 for last-resort patterns and large positive values for patterns that are always profitable.

---

## 147.2 Writing a RewritePattern

The canonical form is `OpRewritePattern<OpType>`, which binds the root operation type at compile time and provides a typed `matchAndRewrite` override.

```cpp
#include "mlir/IR/PatternMatch.h"
#include "mlir/Dialect/Arith/IR/Arith.h"

struct FoldAddZero : mlir::OpRewritePattern<mlir::arith::AddIOp> {
  using OpRewritePattern::OpRewritePattern;

  mlir::LogicalResult matchAndRewrite(
      mlir::arith::AddIOp op,
      mlir::PatternRewriter &rewriter) const override {

    mlir::Value lhs = op.getLhs();
    mlir::Value rhs = op.getRhs();

    // Check if rhs is a constant integer zero.
    mlir::IntegerAttr cst;
    if (mlir::matchPattern(rhs, mlir::m_Constant(&cst)) &&
        cst.getValue().isZero()) {
      rewriter.replaceOp(op, lhs);
      return mlir::success();
    }
    // Symmetric: check lhs.
    if (mlir::matchPattern(lhs, mlir::m_Constant(&cst)) &&
        cst.getValue().isZero()) {
      rewriter.replaceOp(op, rhs);
      return mlir::success();
    }
    return mlir::failure();
  }
};
```

**Key contracts:**

- Return `success()` if and only if the IR was mutated. The driver uses this to decide whether to re-enqueue affected operations.
- Never mutate the IR and then return `failure()`. The `PatternRewriter` tracks mutations and will assert if you do so inconsistently.
- After `rewriter.replaceOp(op, newValues)`, `op` is erased. Do not access it afterwards.
- If you call `rewriter.create<...>(...)` but then decide to abort, call `rewriter.eraseOp` on the newly created op before returning `failure()`.

### PatternRewriter mutation API

| Method | Effect |
|--------|--------|
| `replaceOp(op, newVals)` | Replace all uses of `op`'s results with `newVals`, then erase `op` |
| `replaceAllUsesWith(oldVal, newVal)` | Replace uses only; does not erase `op` |
| `eraseOp(op)` | Erase op (must have no remaining uses) |
| `setInsertionPointBefore(op)` | Move the insertion point |
| `create<OpTy>(loc, args...)` | Create a new op at the current insertion point |
| `mergeBlocks(src, dst, ...)` | Merge `src` into `dst`, replacing block arguments |
| `inlineRegionBefore(region, block)` | Move a region to before `block` |
| `splitBlock(block, iterator)` | Split a block at `iterator` |
| `replaceOpWithNewOp<T>(op, args...)` | Shorthand: create T, replace op with it, erase op |

### Registering patterns

```cpp
mlir::RewritePatternSet patterns(&ctx);
patterns.add<FoldAddZero>(&ctx);
// Or with explicit benefit:
patterns.add<FoldAddZero, /*benefit=*/2>(&ctx);
// Multiple patterns:
mlir::arith::AddIOp::getCanonicalizationPatterns(patterns, &ctx);
```

---

## 147.3 The Greedy Driver

[`mlir/lib/Transforms/Utils/GreedyPatternRewriteDriver.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Transforms/Utils/GreedyPatternRewriteDriver.cpp) implements the fixpoint engine.

### Entry point

```cpp
mlir::LogicalResult mlir::applyPatternsGreedily(
    mlir::Operation *op,
    const mlir::FrozenRewritePatternSet &patterns,
    mlir::GreedyRewriteConfig config = {});
```

`FrozenRewritePatternSet` is a compiled, indexed form of `RewritePatternSet`. The indexing maps each root operation name to the set of patterns that target it, sorted by descending benefit, enabling O(1) pattern lookup per op.

### GreedyRewriteConfig

```cpp
struct GreedyRewriteConfig {
  // Maximum number of fixpoint iterations before giving up.
  unsigned maxIterations = 10;
  // Maximum number of pattern rewrites per iteration.
  unsigned maxNumRewrites = GreedyRewriteConfig::kNoLimit;
  // Use top-down traversal instead of bottom-up (default false).
  bool useTopDownTraversal = false;
  // Limit which ops may be rewritten.
  GreedyRewriteStrictness strictMode = GreedyRewriteStrictness::AnyOp;
  // Enable expensive correctness checks (debug builds).
  bool enableExpensiveChecks = false;
};
```

**Bottom-up vs top-down:** Bottom-up traversal processes leaves first, which is natural for strength reduction and constant folding (inner constants fold before outer ops can use them). Top-down is useful when parent ops constrain the rewriting of children, as in some region-collapsing patterns.

### Worklist semantics

1. Initially, all ops in the region are added to the worklist (in post-order for bottom-up mode).
2. The driver pops an op, finds matching patterns (sorted by benefit), and tries them in order.
3. On the first `success()` return, all newly created ops and all ops whose operands changed are added back to the worklist.
4. The replaced op (if erased) is removed from the worklist.
5. This continues until the worklist is empty or `maxIterations` is reached.

The driver does not re-add ops that were not affected by a rewrite step, preventing O(n²) behavior on programs with many ops.

### Strict mode

`GreedyRewriteStrictness::ExistingAndNewOps` restricts rewrites to ops that were already in the IR when the driver was invoked, plus ops newly created by the driver itself. This prevents the driver from "escaping" into surrounding IR that the caller did not intend to transform.

`GreedyRewriteStrictness::ExistingOps` further restricts to pre-existing ops only—no newly created ops will be re-processed.

---

## 147.4 Canonicalization

Canonicalization is MLIR's built-in simplification mechanism. Every op class may declare canonical-form patterns by overriding the static method:

```cpp
void MyOp::getCanonicalizationPatterns(
    mlir::RewritePatternSet &results,
    mlir::MLIRContext *context);
```

These patterns are gathered by the `--canonicalize` pass and applied greedily to the entire module. Canonical patterns typically cover:

- Algebraic identities (`a + 0 = a`, `a * 1 = a`, `a - a = 0`)
- Strength reductions (`a * 2 = a + a` or `a << 1`)
- Shape inference (`tensor.cast` elision when shapes already match)
- Attribute canonicalization (normalizing attribute values to a unique form)

### The `fold` method

For simpler cases—constant folding and trivial simplifications that do not require creating new ops—ops override `fold()` rather than `matchAndRewrite`:

```cpp
// Returns: success + OpFoldResult if folded; failure otherwise.
mlir::OpFoldResult mlir::arith::AddIOp::fold(FoldAdaptor adaptor) {
  // adaptor.getLhs()/getRhs() are Attribute if operands are constants.
  if (auto lhsAttr = adaptor.getLhs().dyn_cast_or_null<IntegerAttr>())
    if (auto rhsAttr = adaptor.getRhs().dyn_cast_or_null<IntegerAttr>())
      return IntegerAttr::get(getType(),
                              lhsAttr.getValue() + rhsAttr.getValue());
  return {};
}
```

`OpFoldResult` is a discriminated union:
- An `Attribute` — the op folds to a constant (the rewriter creates a `constant` op).
- A `Value` — the op simplifies to an already-existing value (the rewriter replaces uses).
- Empty — not foldable.

The `fold()` path is invoked by the canonicalizer, by `OpBuilder::createOrFold`, and by the constant folder in the greedy driver. It is faster than `matchAndRewrite` because it does not go through the full pattern match infrastructure.

### Running canonicalization

```bash
# Run canonicalize on a .mlir file:
/usr/lib/llvm-22/bin/mlir-opt --canonicalize input.mlir -o output.mlir

# With debug output:
/usr/lib/llvm-22/bin/mlir-opt --canonicalize \
  --mlir-print-ir-after-change input.mlir
```

---

## 147.5 Pattern Matching Utilities

[`mlir/include/mlir/IR/Matchers.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/Matchers.h) provides a composable matcher library for use in pattern conditions.

### Scalar matchers

```cpp
using namespace mlir;
using namespace mlir::matchers;

Value val = ...;
IntegerAttr intAttr;
FloatAttr floatAttr;
APInt apint;
APFloat apfloat;

// Match any constant integer, capture the attribute:
matchPattern(val, m_Constant(&intAttr));

// Match integer zero:
matchPattern(val, m_Zero());

// Match integer one:
matchPattern(val, m_One());

// Match all-ones integer (-1 / ~0):
matchPattern(val, m_AllOnes());

// Match any constant, capture APInt:
matchPattern(val, m_ConstantInt(&apint));

// Match a positive integer constant:
matchPattern(val, m_Positive());

// Match specific value (pointer equality):
matchPattern(val, m_Val(otherVal));

// Match defining op type:
matchPattern(val, m_Op<arith::NegFOp>());
```

### Recursive matchers

Matchers compose recursively. To match `a - (b + c)`:

```cpp
Value a, bPlusC;
if (matchPattern(op.getResult(),
       m_Op<arith::SubIOp>(m_Val(a),
           m_Op<arith::AddIOp>(m_Any(), m_Any())))) {
  // matched
}
```

`m_Any()` matches any value without binding. `m_AnyOf(m1, m2)` tries `m1` first, then `m2`.

### Tensor/vector matchers

```cpp
// Match a splat constant (all elements equal):
DenseElementsAttr splatAttr;
matchPattern(val, m_Constant(&splatAttr));
if (splatAttr.isSplat()) { ... }
```

---

## 147.6 Region Inlining

Inlining is a special pattern: it replaces a call op with the callee's body. MLIR's inliner is in [`mlir/lib/Transforms/Inliner.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Transforms/Inliner.cpp).

### InlinerInterface

Each dialect that owns call or callable ops must implement `DialectInlinerInterface`:

```cpp
struct MyDialectInlinerInterface : DialectInlinerInterface {
  using DialectInlinerInterface::DialectInlinerInterface;

  // Can we inline a call into the given callable?
  bool isLegalToInline(Operation *call, Operation *callable,
                       bool wouldBeCloned) const override {
    return true; // allow inlining by default
  }

  // Can we inline a region into the given op?
  bool isLegalToInline(Region *dest, Region *src, bool wouldBeCloned,
                       IRMapping &valueMapping) const override {
    return true;
  }

  // Handle any type mismatch at the inline boundary.
  Operation *materializeCallConversion(OpBuilder &builder, Value input,
                                       Type resultType,
                                       Location loc) const override {
    return builder.create<UnrealizedConversionCastOp>(
        loc, resultType, input);
  }
};
```

The inliner uses `CallOpInterface` to get the callee region and `CallableOpInterface` to validate the callable.

### Running the inliner

```bash
/usr/lib/llvm-22/bin/mlir-opt --inline input.mlir -o inlined.mlir
```

The `--inline` pass uses a greedy, bottom-up call graph traversal. Recursion is detected and inlining is blocked for recursive call edges. The current cost model is simple (no call site weighting), but the interface is extensible.

### Manual region inlining via PatternRewriter

```cpp
// Inside a rewrite pattern for some "call-like" op:
Block *callBlock = rewriter.getInsertionBlock();
Block *afterCall = rewriter.splitBlock(callBlock, op->getIterator());
// Clone callee blocks into the caller:
rewriter.inlineRegionBefore(calleeRegion, *callBlock->getParent(),
                            afterCall->getIterator());
// Wire up arguments:
Block *calleeEntry = &calleeRegion.front(); // now in caller
rewriter.mergeBlocks(calleeEntry, callBlock, adaptor.getOperands());
```

---

## 147.7 Debugging Patterns

Understanding why patterns did or did not fire is essential for debugging complex transformation pipelines.

### Debug flags

```bash
# Print every pattern match attempt and its result:
/usr/lib/llvm-22/bin/mlir-opt --canonicalize \
  --debug-only=greedy-rewriter input.mlir

# Print IR after every successful pattern application:
/usr/lib/llvm-22/bin/mlir-opt --canonicalize \
  --mlir-print-ir-after-each-pattern input.mlir

# Print IR whenever a pass modifies it:
/usr/lib/llvm-22/bin/mlir-opt --canonicalize \
  --mlir-print-ir-after-change input.mlir
```

### RewriterListener

For programmatic observation, install a `RewriterListener` on the `PatternRewriter`:

```cpp
struct DebugListener : mlir::RewriterBase::Listener {
  void notifyOperationReplaced(mlir::Operation *op,
                               mlir::Operation *newOp) override {
    llvm::dbgs() << "Replaced: " << *op << " → " << *newOp << "\n";
  }
  void notifyOperationErased(mlir::Operation *op) override {
    llvm::dbgs() << "Erased: " << *op << "\n";
  }
};

DebugListener listener;
mlir::GreedyRewriteConfig config;
config.listener = &listener;
mlir::applyPatternsGreedily(op, patterns, config);
```

### Common mistakes

| Symptom | Likely cause |
|---------|-------------|
| Pattern fires but IR is wrong | Returning `success()` from the wrong branch |
| Infinite loop (maxIterations reached) | Pattern creates an op it also matches |
| Pattern never fires | Root op type mismatch; check `OpRewritePattern<WrongOp>` |
| Use-after-free crash | Accessing `op` after `replaceOp` |
| "Pattern returned failure after modifying IR" | Called `rewriter.create` before deciding to fail; must undo or restructure |

### Pattern benefit pitfalls

If two patterns match the same op and both have equal benefit, the driver applies them in registration order. This is usually acceptable, but if patterns are mutually exclusive (one makes the other inapplicable), registration order affects which path is taken first. For op classes with many canonicalization patterns, consider assigning explicit benefits to express the intended priority ordering.

---

## 147.8 Advanced Topics

### Notifying the driver of external IR changes

If a rewrite modifies ops outside the pattern's root (e.g., modifying a producer of the root op), those ops must be re-added to the worklist explicitly:

```cpp
// From inside matchAndRewrite:
rewriter.modifyOpInPlace(producerOp, [&]() {
  producerOp->setAttr("updated", rewriter.getUnitAttr());
});
// producerOp is now on the worklist automatically.
```

`modifyOpInPlace` wraps the mutation in a listener notification so the driver knows to re-evaluate `producerOp`.

### Pattern application to specific ops

To apply patterns to a single op rather than recursively to all ops in a region:

```cpp
mlir::LogicalResult mlir::applyOpPatternsGreedily(
    mlir::Operation *op,
    const mlir::FrozenRewritePatternSet &patterns,
    mlir::GreedyRewriteConfig config,
    bool *changed = nullptr,
    bool *allErased = nullptr);
```

This is useful in the Transform dialect (Chapter 150) where a transform op selects a specific payload op and applies a named set of patterns to it.

### Combining patterns with analyses

Patterns can query analyses computed by the pass infrastructure:

```cpp
struct MyOptPattern : OpRewritePattern<MyOp> {
  DominanceInfo *domInfo;
  MyOptPattern(MLIRContext *ctx, DominanceInfo *di)
      : OpRewritePattern(ctx), domInfo(di) {}

  LogicalResult matchAndRewrite(MyOp op,
                                PatternRewriter &rewriter) const override {
    if (domInfo->dominates(op.getSomeOp(), op)) {
      // ...
    }
    return failure();
  }
};
```

Pass analysis objects must be computed before calling `applyPatternsGreedily` and must remain valid for the duration of the driver run.

---

## Chapter Summary

- **Pattern rewriting** separates the "what to transform" (patterns) from "how to schedule transforms" (greedy driver), enabling composable, order-independent transformations.
- `OpRewritePattern<T>` is the primary pattern base class; `matchAndRewrite` must return `success()` only after mutating the IR.
- The **greedy driver** (`applyPatternsGreedily`) maintains a worklist and iterates to fixpoint; `GreedyRewriteConfig` controls depth, iteration count, and strictness.
- `PatternBenefit` controls inter-pattern priority; higher benefit patterns are tried first when multiple patterns match the same op.
- **Canonicalization** (`getCanonicalizationPatterns` + `fold()`) provides per-op simplification rules; the `--canonicalize` pass applies all of them together.
- The **matcher DSL** (`m_Constant`, `m_Zero`, `m_Op<T>`, etc.) enables concise, composable pattern conditions.
- **Region inlining** uses `DialectInlinerInterface` plus `CallOpInterface`/`CallableOpInterface`; the `--inline` pass drives it.
- **Debugging**: `--debug-only=greedy-rewriter`, `--mlir-print-ir-after-each-pattern`, and `RewriterListener` are the primary tools.


---

@copyright jreuben11
