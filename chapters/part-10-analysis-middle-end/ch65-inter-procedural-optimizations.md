# Chapter 65 — Inter-Procedural Optimizations

*Part X — Analysis and the Middle-End*

Scalar and loop optimizations operate within a single function. Inter-procedural (IP) optimizations see across function boundaries, enabling transformations that no single-function pass can perform: inlining callee bodies, eliminating unused arguments, inferring function attributes, merging identical functions, and globally removing dead definitions. This chapter covers the key IPO passes in LLVM 22.1: the inliner, GlobalOpt, ArgPromotion, FunctionAttrs inference, DeadArgElim, MergeFunctions, PartialInlining, OpenMPOpt, and the Attributor framework.

## 65.1 The Inliner

### 65.1.1 Design

`InlinerPass` (a CGSCC pass, in `llvm/Transforms/IPO/Inliner.h`) inlines call sites where the benefit of inlining (removing call overhead, enabling further optimization) outweighs the cost (code size increase). It runs in bottom-up call graph order (callees before callers), so by the time a callee is inlined into its callers, the callee has already been optimized.

The inliner is driven by the **inline advisor**, which computes a threshold for each call site:

```cpp
class InlineAdvisor {
public:
  virtual std::unique_ptr<InlineAdvice>
  getAdvice(CallBase &CB, FunctionAnalysisManager &FAM) = 0;
};
```

Three advisor implementations:
- `DefaultInlineAdvisor`: threshold-based; see `InlineCost.h`.
- `MLInlineAdvisor`: ML-model-based (Chapter 66).
- `ReplayInlineAdvisor`: replays a previous inlining decision log.

### 65.1.2 Inline Cost

The inline cost for a call site is computed by `getInlineCost` (in `llvm/Analysis/InlineCost.h`), which estimates:
- **Benefit**: call overhead removed, constants propagated, dead code exposed.
- **Cost**: instructions added after inlining (the callee's non-eliminated instructions).

The threshold is controlled by `-inlineprep-threshold` and `TargetTransformInfo::getInliningThresholdMultiplier()`. Calls that are "always inline" (`[[clang::always_inline]]`, `__forceinline`) override the threshold.

```bash
opt -passes='cgscc(inline)' -S input.ll
opt -passes='cgscc(inline<threshold=200>)' -S input.ll  # larger threshold
```

### 65.1.3 Inlining Mechanics

When a call is inlined:
1. The callee's body is cloned into the caller (using `llvm::InlineFunction`).
2. Arguments are substituted for parameters.
3. Return values are connected to the call site result.
4. The call instruction is erased.

```cpp
// Programmatic inlining (from a pass)
InlineFunctionInfo IFI;
InlineResult result = InlineFunction(*CB, IFI);
if (result.isSuccess()) {
  // CB is now erased; the callee body is inlined
}
```

## 65.2 GlobalOpt

`GlobalOptPass` (in `llvm/Transforms/IPO/GlobalOpt.h`) performs whole-module optimizations on global variables:

- **Constant propagation**: a global that is only stored once (and read many times) is converted to a constant. If the store is in a `__attribute__((constructor))` function and the value is a compile-time constant, the global is folded.
- **Internal linkage promotion**: a global reachable only within the module gets `private` linkage, enabling further dead-stripping.
- **Struct decomposition**: a global struct where each field is accessed independently is split into multiple globals, enabling per-field dead-code elimination.
- **Malloc/free elision**: a global allocated in a constructor and freed in a destructor that has no external uses is converted to a static alloca or eliminated.

```bash
opt -passes='globalopt' -S input.ll
```

GlobalOpt requires the entire module and runs at the module pass level.

## 65.3 ArgPromotion

`ArgumentPromotionPass` (in `llvm/Transforms/IPO/ArgumentPromotion.h`) promotes pointer arguments to value arguments when the callee only reads specific fields from the pointed-to struct:

```cpp
// Before: callee reads fields 0 and 1 from the struct
void f(struct S *p) { use(p->x, p->y); }

// After ArgPromotion: pass fields by value (no pointer indirection)
void f(int x, int y) { use(x, y); }
```

ArgPromotion requires that the pointer argument:
- Is never stored (the callee never writes through it).
- Is never passed to another function.
- Has a bounded number of fields accessed.

The optimization reduces pointer indirection in hot call paths and enables further scalar optimization on the now-value arguments.

```bash
opt -passes='cgscc(argpromotion)' -S input.ll
```

## 65.4 FunctionAttrs Inference

`PostOrderFunctionAttrsPass` (in `llvm/Transforms/IPO/FunctionAttrs.h`) infers function attributes bottom-up:

| Attribute | Meaning | Inferred when |
|-----------|---------|---------------|
| `readnone` | No memory reads or writes | No loads, stores, or calls with memory effects |
| `readonly` | Reads but does not write memory | Loads but no stores in callee or callees |
| `writeonly` | Writes but does not read memory | Stores but no loads |
| `norecurse` | Function does not recurse | No cycle in call graph including callee SCCs |
| `nosync` | No synchronization with other threads | No `fence`, `cmpxchg`, or synchronized calls |
| `nofree` | Does not call `free` | No calls to `free`/`delete`/`_ZdlPv` |
| `willreturn` | Always returns (no infinite loop) | Trip-count bounded loops and no blocking calls |

These attributes are consumed by alias analysis (a `readnone` function cannot alias memory with surrounding loads), the inliner (a `readonly` function can be moved out of a loop if LICM applies), and code generation (a `nofree`/`nosync` function is a better candidate for stack allocation of its frame).

```bash
opt -passes='cgscc(function-attrs)' -S input.ll
```

`InferFunctionAttrsPass` runs before the main CGSCC optimization pass and infers known-library-function attributes (e.g., `malloc` is `noalias`, `strlen` is `readonly`).

## 65.5 DeadArgumentElimination

`DeadArgumentEliminationPass` removes function arguments that are always passed the same value or whose value is never used:

```cpp
// Before: %y is passed but never read
int f(int x, int y) { return x * 2; }

// After: y is eliminated
int f(int x) { return x * 2; }
// All callers' y arguments are also removed
```

For non-public (internal linkage) functions, DAE also removes return values that are always ignored by all callers.

```bash
opt -passes='deadargelim' -S input.ll
```

DAE must see all callers to determine argument liveness; it therefore requires non-external linkage.

## 65.6 MergeFunctions

`MergeFunctionsPass` (in `llvm/Transforms/IPO/MergeFunctions.h`) finds functions with identical IR and merges them into a single definition. This reduces code size when the same pattern appears in multiple template instantiations or generated functions.

```cpp
// Two template instantiations that produce identical code after optimization:
int add_int(int a, int b)  { return a + b; }
long add_long(long a, long b) { return a + b; }
// If their IR (after type erasure) is identical, they can be merged.
```

Merging replaces one function with a call to the other (thunk style), or with a bitcast when their types are equivalent.

```bash
opt -passes='mergefunc' -S input.ll
```

MergeFunctions is particularly useful for C++ codebases with heavy template use.

## 65.7 PartialInlining

`PartialInlinerPass` inlines only the "hot" prefix of a function — the part before the first early return — leaving the cold remainder in the callee:

```cpp
// Callee: hot path is input validation, cold path is the actual work
int compute(int *p, int n) {
  if (!p || n <= 0) return -1;  // hot early return (often taken)
  // ... cold heavy computation ...
  return result;
}

// After partial inlining:
// At the call site, inline only the guard:
if (!p || n <= 0)
  call_result = -1;          // inlined guard
else
  call_result = compute(p, n); // non-inlined remainder
```

Partial inlining reduces function call overhead for the common fast path without paying the full code size cost of complete inlining.

```bash
opt -passes='partial-inliner' -S input.ll
```

## 65.8 OpenMPOpt

`OpenMPOptPass` (in `llvm/Transforms/IPO/OpenMPOpt.h`) performs OpenMP-specific interprocedural optimizations:

- **Parallel region elimination**: a parallel region that executes with one thread (singleton team) can be replaced by the sequential region body.
- **State machine deduplication**: multiple OpenMP outlined functions with identical bodies are merged.
- **Barrier elimination**: provably unnecessary barriers between regions with no intervening stores are removed.
- **`__kmpc_*` call simplification**: OpenMP runtime calls whose arguments are statically known are folded.

```bash
opt -passes='openmpopt' -S input.ll
```

OpenMPOpt runs at the module level and requires the call graph of the entire OpenMP program.

## 65.9 The Attributor Framework

The `AttributorPass` (in `llvm/Transforms/IPO/Attributor.h`) is a general inter-procedural analysis and transformation framework that goes beyond what individual passes like FunctionAttrs can do. It models program facts as **abstract attributes** and propagates them through the call graph in a fixpoint iteration:

### 65.9.1 Abstract Attributes

Each attribute is an `AbstractAttribute` subclass implementing:
- `initialize`: compute an initial value from the IR.
- `updateImpl`: refine the value using results from dependencies.
- `manifest`: apply the inferred value to the IR.

Built-in abstract attributes include:
- `AANoReturn`: function does not return.
- `AAPotentialValues`: a set of potential constant values for a parameter.
- `AAPointerInfo`: what memory locations a pointer may point to.
- `AAValueConstantRange`: the integer range of a value.
- `AAMustProgress`: a function always makes progress (no infinite loop without side effects).

### 65.9.2 Fixpoint Iteration

The Attributor iterates all abstract attributes to a fixpoint using a worklist algorithm:

```
initialize all AAs
while (worklist not empty):
  AA = worklist.pop()
  changed = AA.update()
  if changed:
    worklist.add(AA.dependents)
for AA in all_AAs:
  if AA.isAtFixpoint():
    AA.manifest()  // apply to IR
```

Each update either keeps the value the same or moves it toward `bottom` (more conservative). The iteration terminates because the lattice is finite.

### 65.9.3 Example: Inlining with Attributor Guidance

The Attributor can compute that a function argument is always a specific constant, enabling the inliner to skip calls and propagate the constant:

```cpp
// All callers pass n=5
void process(int n) { for (int i = 0; i < n; ++i) work(); }
// Attributor detects n is always 5, propagates AAPotentialValues
// Downstream: loop unroll with trip count 5
```

```bash
opt -passes='attributor' -S input.ll
opt -passes='cgscc(attributor-cgscc)' -S input.ll  # CGSCC variant
```

---

## Chapter Summary

- `InlinerPass` is a CGSCC pass that inlines calls where the inline cost (estimated via `TargetTransformInfo`) is below a threshold; driven by an `InlineAdvisor` (default, ML, or replay).
- `GlobalOptPass` converts single-write globals to constants, promotes to private linkage, and decomposes struct globals into fields.
- `ArgumentPromotionPass` replaces pointer arguments with value arguments when the callee only reads specific fields, reducing pointer indirection.
- `PostOrderFunctionAttrsPass` infers `readnone`, `readonly`, `norecurse`, `nosync`, `nofree`, and `willreturn` attributes bottom-up through the call graph.
- `DeadArgumentEliminationPass` removes unused function arguments and ignored return values in internal functions.
- `MergeFunctionsPass` deduplicates functions with identical IR, reducing code size in template-heavy C++ codebases.
- `PartialInlinerPass` inlines only the hot prefix of a function, reducing call overhead for common fast paths without full code-size cost.
- `OpenMPOptPass` eliminates provably unnecessary OpenMP parallel regions, barriers, and runtime calls using call graph analysis.
- The `AttributorPass` provides a general inter-procedural fixpoint analysis framework with pluggable abstract attributes (value ranges, pointer targets, return status) that feed into the inliner and other passes.


---

@copyright jreuben11
