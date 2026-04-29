# Chapter 61 — Foundational Analyses

*Part X — Analysis and the Middle-End*

Optimization passes do not operate blind. Before transforming IR they query a set of foundational analyses: the dominator tree, loop structure, alias relationships, memory access SSA, scalar evolution of induction variables, demanded bits, branch probabilities, and block frequencies. These analyses are the pre-computable facts that make transformations correct and profitable. This chapter covers each analysis — what it computes, how to request it, and the key API surface — with reference to Chapter 10's theoretical lattice foundations where applicable.

## 61.1 Dominator Tree and Post-Dominator Tree

### 61.1.1 DominatorTree

`DominatorTree` (declared in
[`llvm/include/llvm/IR/Dominators.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/Dominators.h)) computes which basic blocks dominate which other basic blocks. Block `A` dominates block `B` if every path from the function entry to `B` passes through `A`. The tree is the Lengauer-Tarjan dominator tree, computed in O(n α(n)) time.

**Request from a pass:**
```cpp
auto &DT = AM.getResult<DominatorTreeAnalysis>(F);
```

**Key API:**
```cpp
// Does A dominate B?
bool dominated = DT.dominates(blockA, blockB);

// Does instruction A dominate instruction B?
// (Takes into account position within a basic block)
bool idominated = DT.dominates(instrA, instrB);

// Immediate dominator of a block
DomTreeNodeBase<BasicBlock> *node = DT.getNode(BB);
DomTreeNodeBase<BasicBlock> *idom = node->getIDom();
BasicBlock *idomBB = idom ? idom->getBlock() : nullptr;

// Does value V dominate use U?
bool vdom = DT.dominates(V, &use);

// Is a block reachable from the entry?
bool reachable = DT.isReachableFromEntry(BB);
```

The dominator tree is the foundation for SSA construction, LICM (a value hoisted out of a loop must dominate the loop header), and GVN (a value replaces uses that are dominated by the definition).

### 61.1.2 PostDominatorTree

The post-dominator tree (`PostDominatorTreeAnalysis`) reverses the edge direction: `B` post-dominates `A` if every path from `A` to any exit passes through `B`. Post-dominance is used for:
- Control dependence analysis (for PDG construction)
- Sinking optimizations (move instructions down to post-dominated uses)
- Dead-code detection (instructions not post-dominated by any exit are unreachable)

```cpp
auto &PDT = AM.getResult<PostDominatorTreeAnalysis>(F);
bool pdominates = PDT.dominates(BB_B, BB_A); // B post-dominates A
```

## 61.2 LoopInfo

`LoopInfo` (from `LoopAnalysis`, declared in
[`llvm/include/llvm/Analysis/LoopInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Analysis/LoopInfo.h)) identifies the natural loop structure of a function. It requires `DominatorTree` (used to identify back edges, which define loops).

```cpp
auto &LI = AM.getResult<LoopAnalysis>(F);
```

**Iterating loops:**
```cpp
// Top-level loops (outermost)
for (Loop *L : LI) {
  errs() << "Loop header: " << L->getHeader()->getName() << "\n";

  // Subloops (nested)
  for (Loop *SubL : *L)
    errs() << "  Sub-loop: " << SubL->getHeader()->getName() << "\n";
}

// All loops in a function, regardless of depth
for (Loop *L : LI.getLoopsInPreorder())
  processLoop(L);
```

**Loop properties:**
```cpp
Loop *L = /* from LI */;

// Does BB belong to this loop?
bool inLoop = L->contains(BB);

// Is L in simplify form? (preheader, single-exit, single-latch)
bool simplified = L->isLoopSimplifyForm();

// The loop's preheader (unique predecessor of header outside loop)
BasicBlock *PH = L->getLoopPreheader();

// The loop latch (predecessor of header inside loop)
BasicBlock *latch = L->getLoopLatch();

// Is the loop rotated? (header is also the check block)
bool rotated = L->isRotatedForm();

// Has this loop invariant operands?
bool inv = L->hasLoopInvariantOperands(someInstr);

// Get all exit blocks (blocks outside loop that are successors of loop blocks)
SmallVector<BasicBlock*, 4> exits;
L->getExitBlocks(exits);
```

The depth of the loop nest is `L->getLoopDepth()`. Top-level loops have depth 1.

## 61.3 Alias Analysis

### 61.3.1 AliasResult and AAResults

Alias analysis answers the question: can two memory references refer to the same location? The `AAResults` class (declared in
[`llvm/include/llvm/Analysis/AliasAnalysis.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Analysis/AliasAnalysis.h)) is the aggregate of all alias analysis implementations:

```cpp
auto &AA = AM.getResult<AAManager>(F);
```

**Alias query:**
```cpp
// Query whether two memory locations alias
llvm::AliasResult result = AA.alias(
    MemoryLocation::get(loadInstr),   // location 1
    MemoryLocation::get(storeInstr)); // location 2

switch (result) {
  case AliasResult::NoAlias:        /* definitely do not alias */  break;
  case AliasResult::MayAlias:       /* may or may not alias */     break;
  case AliasResult::PartialAlias:   /* overlap but not identical */ break;
  case AliasResult::MustAlias:      /* definitely alias */         break;
}
```

**Mod/Ref query:**
```cpp
// Does calling F modify or reference memory location Loc?
llvm::ModRefInfo mri = AA.getModRefInfo(callInstr, MemoryLocation::get(loadInstr));

if (isModSet(mri))  /* callInstr may write to Loc */ ;
if (isRefSet(mri))  /* callInstr may read from Loc */ ;
if (mri == ModRefInfo::NoModRef) /* no effect on Loc */ ;
```

### 61.3.2 Alias Analysis Variants

LLVM chains multiple AA implementations. The default `AAManager` pipeline (built by `PassBuilder::buildDefaultAAPipeline`) includes:

| Analysis | What it knows |
|----------|--------------|
| `BasicAA` | Pointer comparison, GEP arithmetic, alloca disjointness, constant globals |
| `TypeBasedAA` (TBAA) | C/C++ strict-aliasing rules via `!tbaa` metadata |
| `ScopedNoAliasAA` | `!alias.scope` / `!noalias` metadata |
| `GlobalsAA` | Globals that never escape cannot alias arguments |
| `SCEVAA` | ScalarEvolution-derived pointer arithmetic relationships |
| `CFLSteensAA` | Steensgaard-style points-to analysis (fast, imprecise) |
| `CFLAndersAA` | Andersen-style points-to analysis (slower, more precise) |

The chain stops at the first result that is not `MayAlias`. Frontends that emit `!tbaa` metadata (like Clang) benefit significantly from `TypeBasedAA`.

### 61.3.3 Emitting TBAA Metadata

For frontends implementing strict C aliasing rules:

```cpp
llvm::MDBuilder MDB(ctx);

// Create TBAA root
auto *root = MDB.createTBAARoot("Cal TBAA");

// Create type nodes
auto *intType  = MDB.createTBAAScalarTypeNode("int",  root);
auto *charType = MDB.createTBAAScalarTypeNode("char", root);
auto *ptrType  = MDB.createTBAAScalarTypeNode("ptr",  root);

// Attach to a load instruction
auto *tag = MDB.createTBAAStructTagNode(intType, intType, 0);
load->setMetadata(LLVMContext::MD_tbaa, tag);
```

With TBAA: a load from an `int*` and a load from a `double*` are `NoAlias` (they cannot be the same C object under the C aliasing rules), enabling load forwarding and dead-store elimination across such pairs.

## 61.4 MemorySSA

MemorySSA (declared in
[`llvm/include/llvm/Analysis/MemorySSA.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Analysis/MemorySSA.h)) provides an SSA-form view of memory operations: loads, stores, and memory side effects of calls. It answers "what is the nearest definition that might affect this memory read?" without requiring full alias analysis of every pair.

```cpp
auto &MSSA = AM.getResult<MemorySSAAnalysis>(F).getMSSA();
```

**MemorySSA concepts:**

- **MemoryDef**: a `store` or call that *may write* memory. Has a use-def chain.
- **MemoryUse**: a `load` or call that *reads* memory. Has one defining access.
- **MemoryPhi**: a phi node for memory, at join points in the CFG.

**Querying the clobbering definition:**
```cpp
MemoryUse *MU = cast<MemoryUse>(MSSA.getMemoryAccess(loadInst));
MemoryAccess *clobber = MSSA.getWalker()->getClobberingMemoryAccess(MU);

if (MSSA.isLiveOnEntryDef(clobber)) {
  // This load reads a value that was defined before the function entry
  // (i.e., from a function argument or global)
}
```

**Walking the def-use chain:**
```cpp
// Iterate all memory defs that reach a given load
MemoryAccess *MA = MSSA.getMemoryAccess(load);
for (auto *U : MA->users())
  errs() << "Used by: " << *U << "\n";
```

MemorySSA is used by LICM (to determine whether a load is loop-invariant considering all possible stores in the loop), by DSE (dead store elimination), and by the NewGVN pass.

## 61.5 ScalarEvolution

ScalarEvolution (SE) analyzes induction variables and loop-related expressions. It represents values in loops as symbolic expressions over the loop trip count and initial values:

```cpp
auto &SE = AM.getResult<ScalarEvolutionAnalysis>(F);
```

**SCEV expression types:**

| SCEV type | Meaning |
|-----------|---------|
| `SCEVConstant` | A compile-time constant |
| `SCEVUnknown` | An arbitrary value SE cannot analyze further |
| `SCEVAddExpr` | Sum of SCEVs |
| `SCEVMulExpr` | Product of SCEVs |
| `SCEVAddRecExpr` | `{start, +, stride}L` — an add-recurrence in loop `L` |
| `SCEVUDivExpr` | Unsigned division |
| `SCEVZeroExtendExpr`/`SCEVSignExtendExpr` | Extension |

**Getting a SCEV for a value:**
```cpp
const SCEV *S = SE.getSCEV(value);

if (auto *AddRec = dyn_cast<SCEVAddRecExpr>(S)) {
  // S is a loop induction variable: {start, +, step}
  const SCEV *start = AddRec->getStart();
  const SCEV *step  = AddRec->getStepRecurrence(SE);
  const Loop *loop  = AddRec->getLoop();
}

// Get the trip count (number of times the loop executes)
const SCEV *backEdgeTaken = SE.getBackedgeTakenCount(loop);
if (!isa<SCEVCouldNotCompute>(backEdgeTaken)) {
  // Loop has a computable trip count
  auto *tripCount = SE.getTripCountFromExitCount(backEdgeTaken, …);
}
```

**SE-based arithmetic:**
```cpp
// Add two SCEVs: (s1 + s2)
const SCEV *sum = SE.getAddExpr(s1, s2);

// Scale: k * s
const SCEV *scaled = SE.getMulExpr(SE.getConstant(APInt(64, k)), s);

// Is s provably positive?
bool positive = SE.isKnownPositive(s);

// Is s1 < s2 at all loop iterations?
bool dominates = SE.isKnownPredicate(ICmpInst::ICMP_SLT, s1, s2);
```

ScalarEvolution is essential for loop vectorization (knowing the trip count), loop unrolling (knowing how many iterations), and IndVarSimplify (canonicalizing the induction variable).

## 61.6 DemandedBits

`DemandedBitsAnalysis` computes which bits of an integer value are used by downstream instructions. If bits 8–31 of an `i32` are never read, the operation producing them can be simplified.

```cpp
auto &DB = AM.getResult<DemandedBitsAnalysis>(F);
```

**Query:**
```cpp
// Which bits of instruction I are demanded by its users?
APInt demanded = DB.getDemandedBits(I);
// If demanded.isZero(), I is dead (no bits are consumed)
// If demanded == APInt::getAllOnes(I->getType()->getIntegerBitWidth()),
// all bits are needed
```

DemandedBits is used by BDCE (bit-tracking DCE) to remove instructions whose output is entirely undemanded, and by InstCombine to mask off unneeded high bits in operations.

## 61.7 LazyValueInfo

`LazyValueInfo` (LVI) computes known value ranges and constants at specific program points:

```cpp
auto &LVI = AM.getResult<LazyValueAnalysis>(F);
```

**Query:**
```cpp
// Is value V known to be a specific constant at the end of BB?
Constant *C = LVI.getConstant(V, BB);
if (C) errs() << V->getName() << " = " << *C << " at BB exit\n";

// Get the value's range at a use site
ConstantRange CR = LVI.getConstantRange(V, BB, /*UndefAllowed=*/false);
// CR.getSingleElement() is non-null if it's a known constant

// Check a predicate on two values
LazyValueInfo::Tristate result =
    LVI.getPredicateAt(CmpInst::ICMP_SGT, V, ConstantInt::get(ty, 0), insn);
switch (result) {
  case LazyValueInfo::True:    // definitely > 0
  case LazyValueInfo::False:   // definitely not > 0
  case LazyValueInfo::Unknown: // unknown
}
```

LVI is used by CorrelatedValuePropagation (propagating range information through conditional branches) and JumpThreading.

## 61.8 BranchProbabilityInfo

`BranchProbabilityInfo` (BPI) assigns probabilities to branch edges, derived from profile data (`!prof` metadata), heuristics (loop back-edges are usually taken), or a combination:

```cpp
auto &BPI = AM.getResult<BranchProbabilityAnalysis>(F);
```

**Query:**
```cpp
// Probability that BB branches to Succ (expressed as BranchProbability)
BranchProbability prob = BPI.getEdgeProbability(BB, Succ);

// Check if this is a "likely" branch
bool taken = BPI.isEdgeHot(BB, Succ);

// Probability as a ratio
uint32_t n = prob.getNumerator();
uint32_t d = prob.getDenominator(); // always a power of 2

errs() << "P(BB->Succ) = " << n << "/" << d << "\n";
```

BPI is used by the vectorizer (weighing cost against expected trip count), the loop unroller (hot inner loops), and hot/cold splitting.

## 61.9 BlockFrequencyInfo

`BlockFrequencyInfo` assigns a relative execution frequency to each basic block, computed from `BranchProbabilityInfo` via forward propagation from the function entry (which has frequency 1.0):

```cpp
auto &BFI = AM.getResult<BlockFrequencyAnalysis>(F);
```

**Query:**
```cpp
// Block frequency (relative units, not absolute counts)
BlockFrequency freq = BFI.getBlockFreq(BB);
uint64_t raw = freq.getFrequency(); // raw count, scale is internal

// Entry block frequency (reference for relative comparisons)
BlockFrequency entry = BFI.getEntryFreq();

// Is this block "hot" (above a threshold)?
bool hot = BFI.getBlockProfileCount(BB).value_or(0) > threshold;

// Print a function's block frequencies (for debugging)
BFI.print(errs());
```

Block frequency information is consumed by:
- `InlinerPass`: avoiding inlining into cold call sites.
- Hot/cold splitting: moving cold basic blocks into separate functions for better icache.
- `LoopUnrollPass`: aggressive unrolling of frequently executed loops.

With PGO data (`-fprofile-use`), BFI carries actual execution counts; without PGO it uses heuristic-based estimates.

## 61.10 Analysis Interdependencies

The analyses form a dependency graph that the `AnalysisManager` resolves on demand:

```
BlockFrequencyInfo
  └─ BranchProbabilityInfo
       └─ LoopInfo
            └─ DominatorTree

MemorySSA
  └─ AliasAnalysis (AAManager)
       ├─ BasicAA
       │    └─ (none)
       ├─ TypeBasedAA (metadata only)
       └─ SCEVAA
            └─ ScalarEvolution
                 └─ LoopInfo, AssumptionCache, DominatorTree

LazyValueInfo
  └─ AssumptionCache, DominatorTree
```

When a pass requests `BlockFrequencyAnalysis`, the `FunctionAnalysisManager` recursively computes `BranchProbabilityAnalysis`, which triggers `LoopAnalysis`, which triggers `DominatorTreeAnalysis`. This chain runs once and all results are cached.

---

## Chapter Summary

- `DominatorTreeAnalysis` computes the classic Lengauer-Tarjan dominator tree; `dominates(A, B)` tests whether `A` dominates `B`; `isReachableFromEntry(BB)` identifies unreachable blocks.
- `PostDominatorTreeAnalysis` computes the reverse-direction dominator tree, used for control-dependence and sinking.
- `LoopAnalysis` identifies natural loops via back-edges; loops are organized in a depth-first loop nest with `getLoopPreheader()`, `getLoopLatch()`, and `getExitBlocks()`.
- `AAManager` chains multiple alias analysis implementations; `alias(Loc1, Loc2)` returns `NoAlias`, `MayAlias`, `PartialAlias`, or `MustAlias`; `getModRefInfo` classifies memory effects of calls.
- `TypeBasedAA` exploits `!tbaa` metadata to apply C/C++ strict-aliasing rules; frontends should emit this metadata for each typed load/store.
- `MemorySSA` provides SSA-form memory access chains; `getWalker()->getClobberingMemoryAccess(use)` finds the nearest store that may affect a given load.
- `ScalarEvolutionAnalysis` represents loop expressions as SCEVs; `SCEVAddRecExpr {start, +, step}` is the canonical induction variable; `getBackedgeTakenCount(L)` gives the trip count.
- `DemandedBitsAnalysis` identifies which bits of a value are consumed; zero demanded bits means the instruction is dead.
- `LazyValueInfo` provides per-point value range and predicate queries; `getPredicateAt` checks a comparison's Tristate value at a specific instruction.
- `BranchProbabilityInfo` and `BlockFrequencyInfo` assign probabilities and relative frequencies to edges and blocks, derived from `!prof` metadata or heuristics.


---

@copyright jreuben11
