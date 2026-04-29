# Chapter 21 — SSA, Dominance, and Loops

*Part IV — LLVM IR*

Chapter 9 derived the theory: dominator trees, dominance frontiers, the Cytron IDF algorithm, and the formal SSA invariant. This chapter is its engineering complement. Every concept from Chapter 9 has a corresponding C++ class in LLVM's analysis infrastructure, and understanding the mapping between the two is essential for writing any non-trivial pass. This chapter covers how Clang generates SSA from C source, how `mem2reg` and SROA promote memory-based variables to register-like SSA values, how the `DominatorTree` and `PostDominatorTree` are queried in a pass, and how `LoopInfo` exposes the natural loop structure that almost every optimization pass consumes. It also covers `LoopNest` for perfect loop nests, and the `CycleInfo` generalization that handles irreducible CFGs.

Prerequisites: [Chapter 9 — Intermediate Representations and SSA Construction](../part-02-compiler-theory/ch09-intermediate-representations-and-ssa.md), [Chapter 16 — IR Structure](../part-04-llvm-ir/ch16-ir-structure.md). Familiarity with LLVM's new pass manager is assumed; see [Chapter 59 — The New Pass Manager](../part-10-analysis-middle-end/ch59-new-pass-manager.md) for a refresher.

---

## 21.1 SSA in LLVM IR: How Clang Generates It

### 21.1.1 The Alloca-First Strategy

Clang does not construct SSA during code generation. Instead, it applies a two-phase strategy that separates correct code generation from efficient code generation:

**Phase 1 — alloca emission.** For every local scalar variable in a C/C++ function, `CodeGenFunction` emits a single `alloca` instruction in the entry block. All reads of the variable are lowered to `load` from the alloca; all writes are lowered to `store`. The result is semantically valid LLVM IR but not in SSA form for those variables.

**Phase 2 — mem2reg promotion.** The `mem2reg` pass (run early in every optimization pipeline) identifies promotable allocas and converts them to proper SSA φ-functions using the Cytron algorithm. After `mem2reg`, the function is in standard SSA form.

To see the two phases, compile the following C fragment:

```c
// Source: loop_example.c
int sum_array(int *a, int n) {
    int sum = 0;
    for (int i = 0; i < n; i++)
        sum += a[i];
    return sum;
}
```

```bash
# Phase 1: alloca-based IR (no SSA for sum and i)
clang-22 -O0 -emit-llvm -S loop_example.c -o loop_O0.ll
```

The `-O0` output contains four allocas at the entry block — one for `a`, one for `n`, one for `sum`, and one for `i` — and every access to those variables goes through `load`/`store` pairs. The key structural fact is that the header block (`%7` in the raw IR) loads `%6` (the alloca for `i`) and `%4` (the alloca for `n`) anew on every iteration. There are no φ-nodes anywhere.

```bash
# Phase 2: SSA form after mem2reg (via -O1)
clang-22 -O1 -fno-vectorize -fno-unroll-loops -emit-llvm -S loop_example.c -o loop_O1.ll
```

After `mem2reg`, the alloca for `sum` and the alloca for `i` have been eliminated and replaced with φ-nodes at the loop header. The parameters `a` and `n` were already SSA values (function arguments), so their allocas were also trivially eliminated.

### 21.1.2 Why Alloca-First is Cleaner

Section 9.10.2 of Chapter 9 explains the separation of concerns in detail. The short version: generating SSA directly during AST lowering would require the code generator to track every control-flow join point as it lowers statements, inserting incomplete φ-nodes or running the IDF algorithm incrementally. That complexity is avoided entirely by letting `mem2reg` handle SSA construction as a pure IR transformation after code generation is complete.

The alloca-first strategy also means the IR at `-O0` is always valid: it uses only memory operations, which the verifier can check trivially. There is no risk of the code generator emitting a malformed SSA value or violating a dominance constraint.

---

## 21.2 The `mem2reg` and `sroa` Passes

### 21.2.1 `mem2reg`: Promote Memory to Register

`mem2reg` is the pass that converts alloca-based IR to SSA form. It is invoked by the pass name `mem2reg` in the new pass manager and corresponds to `PromoteMemoryToRegisterPass` in the source tree (`llvm/lib/Transforms/Utils/Mem2Reg.cpp`). The underlying algorithm is in `PromoteMemToReg.cpp`.

**Promotability.** An alloca is promotable if and only if every use of the alloca pointer is a `load`, a `store`, or a lifetime marker (`llvm.lifetime.start` / `llvm.lifetime.end`). The public predicate is:

```cpp
#include "llvm/Transforms/Utils/PromoteMemToReg.h"

bool isAllocaPromotable(const AllocaInst *AI);
```

An alloca is *not* promotable if:
- Its address is passed to a function (the address is taken and escapes).
- It is cast to a different pointer type via `bitcast` and then used (absent in post-LLVM-15 opaque-ptr IR, but historically relevant).
- There is a `volatile` or `atomic` load or store through it.
- The element type is a struct or array (those are handled by SROA, not `mem2reg`).

**The algorithm** (summarized from Chapter 9's §9.5 treatment):

1. Collect all promotable allocas in the function.
2. For each alloca acting as variable `v`, compute `Defs(v)` — the set of blocks containing a `store` to the alloca.
3. Run the Cytron IDF algorithm to determine where φ-nodes must be inserted: compute `IDF(Defs(v))` using a worklist that propagates through the dominance frontier.
4. Insert φ-instructions at each block in `IDF(Defs(v))`.
5. Rename: walk the dominator tree in DFS order, maintaining a per-variable stack of current definitions. Replace each `load` with the top-of-stack value; each `store` pushes a new definition. Fill incoming values of φ-nodes as they are encountered.
6. Delete all promoted allocas, loads, and stores.

The `PromoteMemToReg` free function is the programmatic API:

```cpp
#include "llvm/Transforms/Utils/PromoteMemToReg.h"

// Promote a specific list of allocas in one shot.
// DT must be valid; AC (AssumptionCache) is optional but improves quality.
void PromoteMemToReg(ArrayRef<AllocaInst *> Allocas,
                     DominatorTree &DT,
                     AssumptionCache *AC = nullptr);
```

**When to run it.** `mem2reg` is placed early in every optimization pipeline — before alias analysis, before GVN, before LICM. This is because virtually all scalar optimizations assume SSA form, and most analyses produce inaccurate results on alloca-based IR. The standard `O1` pipeline runs `mem2reg` as one of the first passes:

```bash
opt --passes="mem2reg,instcombine,simplifycfg,..." -S input.ll
```

### 21.2.2 `sroa`: Scalar Replacement of Aggregates

SROA (`SROAPass`, pass name `sroa`) extends `mem2reg` to handle aggregate types — structs and fixed-size arrays. Where `mem2reg` requires the alloca to hold a scalar type, SROA handles cases like:

```c
struct Point { int x, y; };

int foo(int a, int b) {
    struct Point p;
    p.x = a;
    p.y = b;
    return p.x + p.y;
}
```

Clang emits a single `alloca` for `p`. SROA *slices* it: it detects that `p.x` and `p.y` are always accessed independently via distinct constant GEP offsets, and splits the aggregate alloca into two independent `i32` allocas. These scalar allocas are then trivially promotable by `mem2reg`.

SROA handles partial use — when only some fields of a struct are loaded or stored. It assigns each slice to a scalar alloca that covers exactly the bytes used in that slice. Slices that straddle a GEP boundary or overlap with a non-promotable use are left as memory (a "cannot promote" path).

SROA is also run early in the pipeline, typically before or alongside `mem2reg`:

```bash
opt --passes="sroa,mem2reg,instcombine,..." -S input.ll
```

The SROA pass provides two mode variants:

```cpp
#include "llvm/Transforms/Scalar/SROA.h"

// PreserveCFG mode: does not split critical edges or add blocks.
// ModifyCFG mode: may modify the CFG to enable more promotions.
SROAPass(SROAOptions::PreserveCFG)
SROAPass(SROAOptions::ModifyCFG)
```

The standard pipeline uses `ModifyCFG`; `PreserveCFG` is available for use in contexts where CFG modifications are not yet safe (e.g., early in a function-level pass sequence that has not yet run `simplifycfg`).

---

## 21.3 DominatorTree and PostDominatorTree

### 21.3.1 The `DominatorTree` Class

The `DominatorTree` (`llvm/IR/Dominators.h`) implements the Cooper–Harvey–Kennedy algorithm and exposes the dominator tree over a function's CFG. The tree is computed by `DominatorTreeAnalysis` in the new pass manager:

```cpp
#include "llvm/IR/Dominators.h"
#include "llvm/Analysis/Dominators.h"

PreservedAnalyses MyPass::run(Function &F, FunctionAnalysisManager &AM) {
  DominatorTree &DT = AM.getResult<DominatorTreeAnalysis>(F);
  // use DT...
  return PreservedAnalyses::all();
}
```

The tree node type is `DomTreeNode` (a typedef for `DomTreeNodeBase<BasicBlock>`). Key queries:

```cpp
// Does block A dominate block B?
bool dom = DT.dominates(A, B);

// Does A strictly dominate B (A != B and A dom B)?
bool sdom = DT.properlyDominates(A, B);

// Immediate dominator of B (nullptr for the entry block)
DomTreeNode *BNode = DT.getNode(B);
DomTreeNode *IdomNode = BNode->getIDom();
BasicBlock *Idom = IdomNode ? IdomNode->getBlock() : nullptr;

// Nearest common dominator of A and B (the LCA in the dominator tree)
BasicBlock *NCD = DT.findNearestCommonDominator(A, B);

// Does instruction DefI dominate use instruction UseI?
// This requires that DefI and UseI be in the same function.
bool idom = DT.dominates(DefI, UseI);
```

The `dominates(Instruction*, Instruction*)` overload handles the within-block case correctly: an instruction dominates a later instruction in the same block by position, and it dominates instructions in dominated blocks by the basic block dominance relation. φ-nodes are handled specially: a `phi` in block B is dominated by `DefI` if `DefI` dominates the predecessor block from which the φ incoming value flows, not the block B itself.

The tree is rooted at the entry block's node. Iterating the tree:

```cpp
// Walk the dominator tree in DFS preorder
for (DomTreeNode *Node : depth_first(DT.getRootNode())) {
  BasicBlock *BB = Node->getBlock();
  unsigned Depth = Node->getLevel();
  // ...
}
```

### 21.3.2 `PostDominatorTree`

The post-dominator tree (`llvm/Analysis/PostDominators.h`) is the dual structure computed on the reversed CFG. Block A post-dominates block B if every path from B to any function exit passes through A. It is computed by `PostDominatorTreeAnalysis`:

```cpp
#include "llvm/Analysis/PostDominators.h"

PostDominatorTree &PDT = AM.getResult<PostDominatorTreeAnalysis>(F);

// Does A post-dominate B?
bool pdom = PDT.dominates(A, B);

// Immediate post-dominator of B
DomTreeNode *BNode = PDT.getNode(B);
DomTreeNode *IPdomNode = BNode->getIDom();
```

The post-dominator tree has a virtual exit node that serves as the root (the dual of the entry node in the dominator tree). All blocks that return from the function have the virtual exit as an immediate post-dominator; unreachable blocks may be excluded.

### 21.3.3 Control Dependence

Block B is *control-dependent* on block A if A has two successors, one of which can reach B without going through A, and one of which cannot reach B without going through A (equivalently, A's outcome determines whether B executes). Control dependence is computed from the post-dominator tree:

A is a control-dependent predecessor of B if there is an edge `(A, C)` in the CFG such that:
- B post-dominates C, and
- B does not strictly post-dominate A.

Control dependence is the backbone of the Program Dependence Graph (PDG) and is used in program slicing, parallelism analysis, and predication. LLVM does not ship a standalone control dependence class, but the computation is straightforward given `PostDominatorTree`.

### 21.3.4 Invalidation and `PreservedAnalyses`

When a pass modifies the CFG, it must communicate the invalidation to the pass manager. The mechanism is `PreservedAnalyses`:

```cpp
PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM) {
  // ... modify CFG ...

  // Inform the pass manager that DominatorTree and LoopInfo are no longer valid
  PreservedAnalyses PA;
  PA.preserve<DominatorTreeAnalysis>();  // still valid (if you maintained it)
  // OR:
  PA.abandon<DominatorTreeAnalysis>();   // explicitly invalidated
  return PA;
}
```

If a pass returns `PreservedAnalyses::all()`, it asserts that it made no observable changes; the pass manager assumes all analyses remain valid. If a pass returns `PreservedAnalyses::none()`, it asserts that no analysis is preserved; every subsequent analysis will be recomputed from scratch.

For passes that perform incremental CFG modifications (inserting or deleting edges), LLVM provides `DominatorTree::insertEdge(From, To)` and `DominatorTree::deleteEdge(From, To)` to update the tree in O(depth) time rather than paying O(|V| + |E|) for a full recomputation. See [Chapter 61 — Foundational Analyses](../part-10-analysis-middle-end/ch61-foundational-analyses.md) for the full incremental update protocol.

---

## 21.4 Natural Loops: `LoopInfo`, Headers, Latches, Exits

### 21.4.1 LLVM's Definition of a Natural Loop

LLVM uses the classical definition of a *natural loop*: a set of basic blocks with a single entry point (the *header*) and at least one *back-edge* — an edge from a block inside the loop (the *latch*) back to the header. The body consists of all blocks from which the latch can be reached by a path that does not pass through the header.

Equivalently, an edge `(B, H)` is a back-edge if and only if `H` dominates `B`. The natural loop of this back-edge is the set of all blocks reachable from `H` in the CFG when the back-edge `(B, H)` is followed, plus all blocks from which `B` is reachable without passing through `H`.

This definition requires reducible control flow: every cycle in the CFG must have a single entry (the header). Irreducible CFGs (multiple entries into a strongly connected component) do not have natural loops — they have *cycles*, covered in §21.8.

### 21.4.2 Querying `LoopInfo`

`LoopInfo` (`llvm/Analysis/LoopInfo.h`) is the analysis result that encodes the loop nest forest for a function. It is computed by `LoopAnalysis`:

```cpp
#include "llvm/Analysis/LoopInfo.h"

PreservedAnalyses MyLoopPass::run(Function &F, FunctionAnalysisManager &AM) {
  LoopInfo &LI = AM.getResult<LoopAnalysis>(F);
  DominatorTree &DT = AM.getResult<DominatorTreeAnalysis>(F);

  // Iterate over top-level loops (outermost loops not contained in any other)
  for (Loop *L : LI) {
    BasicBlock *Header    = L->getHeader();
    BasicBlock *Preheader = L->getLoopPreheader();  // null if not simplified
    BasicBlock *Latch     = L->getLoopLatch();       // null if multiple latches

    // Iterate nested loops
    for (Loop *SubL : L->getSubLoops()) {
      // ...
    }
  }
  return PreservedAnalyses::all();
}
```

`LoopInfo` presents loops as a *forest*: the top-level nodes are the outermost loops; each loop has a list of sub-loops (`getSubLoops()`). The depth-first traversal of the forest visits all loops in the function; LLVM provides `LI.getLoopsInPreorder()` for a pre-order vector of all loops, and `LI.getLoopsInPostorder()` (via `LoopInfoBase`) for post-order.

Key `Loop` API methods:

```cpp
// Loop structure
BasicBlock *Header    = L->getHeader();
BasicBlock *Preheader = L->getLoopPreheader();   // nullptr if not simplified
BasicBlock *Latch     = L->getLoopLatch();        // nullptr if multiple latches

SmallVector<BasicBlock *, 4> Latches;
L->getLoopLatches(Latches);  // all latches when there are multiple

// Exit topology
BasicBlock *ExitBB = L->getExitBlock();          // nullptr if multiple exits
SmallVector<BasicBlock *, 4> ExitBlocks;
L->getExitBlocks(ExitBlocks);                    // all blocks outside with pred inside

BasicBlock *ExitingBB = L->getExitingBlock();    // nullptr if multiple exiting blocks
SmallVector<BasicBlock *, 4> ExitingBlocks;
L->getExitingBlocks(ExitingBlocks);              // all blocks inside with succ outside

// Loop-invariance test
Value *V = /* some value */;
bool Invariant = L->isLoopInvariant(V);

// Depth in the loop nest (1 = outermost)
unsigned Depth = L->getLoopDepth();

// All blocks in the loop (including sub-loop blocks)
ArrayRef<BasicBlock *> Blocks = L->getBlocks();

// Look up which loop (if any) a given block belongs to
Loop *OwningLoop = LI.getLoopFor(SomeBB);
```

### 21.4.3 Checking Dominance in a Loop Pass

A common pattern is to check whether a definition dominates all of its loop-body uses before hoisting it out of the loop (a simplified form of LICM):

```cpp
PreservedAnalyses MyLoopPass::run(Function &F, FunctionAnalysisManager &AM) {
  auto &LI = AM.getResult<LoopAnalysis>(F);
  auto &DT = AM.getResult<DominatorTreeAnalysis>(F);

  for (Loop *L : LI) {
    BasicBlock *Header = L->getHeader();
    BasicBlock *Preheader = L->getLoopPreheader();
    if (!Preheader)
      continue;  // loop is not in simplified form; skip

    for (BasicBlock *BB : L->getBlocks()) {
      for (Instruction &I : *BB) {
        // Check that all operands dominate the loop header
        // (necessary but not sufficient for loop-invariance)
        bool AllOpsDomHeader = true;
        for (Value *Op : I.operands()) {
          if (auto *OpInst = dyn_cast<Instruction>(Op)) {
            if (!DT.dominates(OpInst->getParent(), Header)) {
              AllOpsDomHeader = false;
              break;
            }
          }
        }
        if (AllOpsDomHeader && L->isLoopInvariant(&I)) {
          // Safe to hoist I to Preheader
          I.moveBefore(Preheader->getTerminator());
        }
      }
    }
  }
  return PreservedAnalyses::all();
}
```

The real LICM pass (`llvm/lib/Transforms/Scalar/LICM.cpp`) is substantially more complex — it uses alias analysis, handles stores, and interacts with `MemorySSA` — but the structure above illustrates the canonical pattern of combining `LoopInfo`, `DominatorTree`, and per-instruction invariance checks.

---

## 21.5 Loop Terminology in LLVM

The loop structure terminology in LLVM is precise and consistent across all optimization passes. This section defines each term and shows how it maps to the `Loop` API.

### 21.5.1 Header

The **header** is the unique basic block that serves as the entry point to the loop. It is the only block in the loop with a predecessor *outside* the loop. By definition, the header dominates all other blocks in the loop.

```cpp
BasicBlock *Header = L->getHeader();
```

In the CFG, the header has two classes of predecessors:
- Predecessors outside the loop (via which the loop is entered on the first iteration).
- The latch (or latches), which deliver the back-edge.

### 21.5.2 Latch

The **latch** is the basic block that contains the back-edge to the header. After `LoopSimplify`, a normalized loop has exactly one latch.

```cpp
BasicBlock *Latch = L->getLoopLatch();  // null if multiple latches
```

In a while-loop lowering, the latch typically contains the increment of the induction variable and the unconditional branch back to the header. In a *rotated* loop (§21.5.5), the latch contains the loop exit condition check.

### 21.5.3 Exiting Block and Exit Block

These are dual concepts:

- An **exiting block** is a block *inside* the loop that has at least one successor *outside* the loop. It is where the loop can leave.
- An **exit block** is a block *outside* the loop that has at least one predecessor *inside* the loop. It is where execution continues after the loop.

```cpp
// Single-exit queries (return null if there are multiple)
BasicBlock *ExitingBB = L->getExitingBlock();
BasicBlock *ExitBB    = L->getExitBlock();

// Multi-exit queries
SmallVector<BasicBlock *, 4> ExitingBBs, ExitBBs;
L->getExitingBlocks(ExitingBBs);
L->getExitBlocks(ExitBBs);
```

After `LoopSimplify`, each exit block has exactly one predecessor inside the loop (a *dedicated exit block*), which enables analyses like scalar evolution to reason about trip counts precisely.

### 21.5.4 Preheader

The **preheader** is a basic block that:
- Dominates the header.
- Has the header as its only successor.
- Is the only predecessor of the header from outside the loop.

The preheader is not part of the loop itself — it is the block that falls into the loop on the first iteration. Its existence enables critical loop optimizations: LICM hoists loop-invariant instructions to the preheader; loop peeling inserts code before the preheader; loop invariant code motion targets the preheader for hoist destinations.

```cpp
BasicBlock *Preheader = L->getLoopPreheader();  // null if loop is not simplified
```

If the loop has not been simplified by `LoopSimplify`, the preheader may not exist (multiple outside predecessors of the header). Always check for null before using the preheader.

### 21.5.5 Rotated Loop

A **rotated loop** is a loop whose condition check has been moved from the header to the latch. After rotation, the header contains only the loop body (no conditional branch), and the latch contains the combination of induction variable increment and the exit condition check.

Original while-loop structure (before rotation):
```
preheader → header (contains: check i < n) → body → latch → back-edge to header
                                            → exit
```

After loop rotation:
```
preheader → header-becomes-body (no check) → latch (contains: i++; check i < n) → back-edge
                                                                                  → exit
```

Loop rotation (`loop-rotate` in the new PM) enables LICM and vectorization because the latch's single conditional branch is the canonical form that those passes expect. It also enables the compiler to emit a *do-while* loop in assembly, which is one branch shorter than a while-loop equivalent.

The `LoopInfo` queries after rotation:
- `getHeader()` — still the first block entered from the preheader.
- `getLoopLatch()` — the block containing the conditional back-edge.
- `getExitingBlock()` — the latch (it has both the back-edge and the exit edge).

---

## 21.6 `LoopSimplify`: Normalizing Loops for Optimization

### 21.6.1 What `LoopSimplify` Does

The `LoopSimplifyPass` (pass name `loop-simplify`) transforms a natural loop into a *simplified* (or *canonical*) form that virtually all loop optimization passes require. It establishes three structural properties:

1. **Preheader insertion.** If the loop header has multiple predecessors from outside the loop, `LoopSimplify` inserts a new preheader block that merges them. After this, `L->getLoopPreheader()` is guaranteed non-null.

2. **Single latch.** If the loop has multiple back-edges (multiple latches), `LoopSimplify` merges them into a single latch block. After this, `L->getLoopLatch()` is guaranteed non-null.

3. **Dedicated exit blocks.** If an exit block has predecessors from both inside and outside the loop, `LoopSimplify` inserts a new dedicated exit block for the loop. This ensures that φ-nodes in exit blocks have all their loop-coming values from a single predecessor inside the loop, which simplifies trip count analysis.

```bash
# Apply loop simplification to a function
opt --passes="loop-simplify" -S input.ll -o simplified.ll
```

### 21.6.2 Why Simplification Is Required

LICM, loop vectorization, loop unrolling, and induction variable simplification (`indvars`) all begin with a precondition check: `assert(L->getLoopPreheader() && L->getLoopLatch())`. If the loop is not simplified, these passes skip it entirely. The standard optimization pipeline therefore runs `loop-simplify` before any loop transformation:

```bash
opt --passes="loop-simplify,indvars,loop-rotate,licm,loop-vectorize" -S input.ll
```

In practice, `LoopSimplify` is so foundational that LLVM's `PassBuilder` inserts it automatically before any loop optimization group.

### 21.6.3 The Annotated Loop CFG

Here is the canonical annotated loop IR that results from compiling and optimizing the `process` function with loop-simplify and rotation applied:

```llvm
; Source: void process(int *a, int n) { for (int i = 0; i < n; i++) a[i] *= 2; }
; Compiled: clang-22 -O1 -fno-vectorize -fno-unroll-loops -emit-llvm -S
;
; BB %2 is the entry block (implicit preheader role: checks n > 0)
; BB %4 is a bridge block from the entry check to the loop header
; BB %7 is the loop header AND latch (rotated: body + back-edge condition)
; BB %6 is the exit block

define dso_local void @process(ptr noundef captures(none) %0, i32 noundef %1) {
entry:                                                ; [preheader / guard]
  %3 = icmp sgt i32 %1, 0          ; guard: skip loop entirely if n <= 0
  br i1 %3, label %loop.preheader, label %exit

loop.preheader:                                       ; [preheader]
  %trip = zext nneg i32 %1 to i64  ; trip count = (i64)n
  br label %loop.header

loop.header:                                          ; [header = latch, rotated]
  %i = phi i64 [ 0, %loop.preheader ], [ %i.next, %loop.header ]
  %gep = getelementptr inbounds nuw i32, ptr %0, i64 %i
  %val = load i32, ptr %gep, align 4
  %doubled = shl nsw i32 %val, 1
  store i32 %doubled, ptr %gep, align 4
  %i.next = add nuw nsw i64 %i, 1
  %done = icmp eq i64 %i.next, %trip
  br i1 %done, label %exit, label %loop.header  ; back-edge to self = latch

exit:                                                 ; [exit block]
  ret void
}
```

In this rotated form:
- The preheader guard (`entry`) prevents entering the loop when `n <= 0`.
- `loop.preheader` is the true preheader — it dominates `loop.header` and has only `loop.header` as its successor.
- `loop.header` is simultaneously the header (single entry from outside the loop) and the latch (the back-edge `br` returns to itself). This is the rotated structure: body and condition check are combined.
- `exit` is the exit block.

The `%i` phi node has two incoming values: `0` from the preheader (first iteration) and `%i.next` from the latch (subsequent iterations). `%i` is the canonical induction variable for ScalarEvolution.

---

## 21.7 `LoopNest`: Perfect Loop Nests

### 21.7.1 The `LoopNest` Class

`LoopNest` (`llvm/Analysis/LoopNestAnalysis.h`) captures the structure of a loop nest rooted at a given loop. It is a loop-level analysis (run by `LoopNestAnalysis`, which is registered with the `LoopAnalysisManager`):

```cpp
#include "llvm/Analysis/LoopNestAnalysis.h"

// Inside a loop pass run with a LoopAnalysisManager:
PreservedAnalyses MyNestPass::run(Loop &L, LoopAnalysisManager &LAM,
                                  LoopStandardAnalysisResults &AR,
                                  LPMUpdater &U) {
  LoopNest &LN = LAM.getResult<LoopNestAnalysis>(L, AR);

  Loop &Outermost = LN.getOutermostLoop();
  ArrayRef<Loop *> AllLoops = LN.getLoops();  // outermost-first, all levels

  // Perfect loops require ScalarEvolution
  ScalarEvolution &SE = AR.SE;
  SmallVector<LoopVectorTy, 4> PerfectNests = LN.getPerfectLoops(SE);

  return PreservedAnalyses::all();
}
```

`LN.getLoops()` returns all loops in the nest from outermost to innermost. `LN.getOutermostLoop()` returns the root of the nest.

### 21.7.2 Perfect Loop Nests

A loop nest is *perfect* if every level of the nest (except the innermost) contains *only* the nested loop — no instructions appear in the body of an outer loop other than the loop induction variable management and the single inner loop. In LLVM's formalization (used by `getPerfectLoops`), two adjacent loops `L_outer` and `L_inner` are *perfectly nested* if:

1. `L_inner` is the only sub-loop of `L_outer`.
2. The header of `L_outer` (excluding the branch to `L_inner`) contains no computation other than the outer induction variable φ-node.
3. The latch of `L_outer` (excluding the back-edge) contains no computation other than the increment.

`getPerfectLoops(SE)` returns a vector of loop-level groups: each group is a maximal sequence of perfectly nested loops from some outer level down to the innermost:

```cpp
SmallVector<LoopVectorTy, 4> PerfGroups = LN.getPerfectLoops(SE);
for (const LoopVectorTy &Group : PerfGroups) {
  // Group[0] is the outermost, Group.back() is the innermost perfectly nested loop
  unsigned Depth = Group.size();
}
```

### 21.7.3 Use in Vectorization and Tiling

The loop vectorizer (`LoopVectorizePass`) operates on the innermost loop of a nest. For loop tiling (used in cache-optimization transformations and in the polyhedral model, discussed in [Chapter 71 — The Polyhedral Model](../part-11-polyhedral-theory/ch71-the-polyhedral-model.md)), a perfect nest is required: the transformation reorders loop iterations across levels, which is only valid when no outer-level computation would be displaced.

Polly, LLVM's polyhedral optimizer ([Chapter 74 — Polly Architecture](../part-12-polly/ch74-polly-architecture.md)), requires a perfect nest and models each loop as a dimension of a polyhedron. `LoopNest::getPerfectLoops` is the API that vectorization legality checkers and Polly's interface use to identify eligible nests.

---

## 21.8 Cycle Terminology and Irreducible CFGs

### 21.8.1 Reducibility

A CFG is *reducible* if every cycle in the CFG is a natural loop — equivalently, if every strongly connected component of the CFG has a unique entry node (a dominator of all nodes in the component from outside the component). Reducible CFGs are produced by all structured programming constructs: `for`, `while`, `do-while`, `if-else`, `break`, `continue`. They are the overwhelming majority of real-world CFGs.

An *irreducible* CFG has at least one cycle with multiple entries. This arises from:
- `goto` statements that jump into the middle of a loop.
- Hand-written assembly or bytecode with unstructured jumps.
- Some coroutine or state-machine lowering patterns.
- Computed `goto` (GCC extension) used in interpreter dispatch loops.

LLVM's loop optimizations — LICM, vectorization, unrolling, indvars — all require reducible CFGs. When a function has irreducible control flow, these passes simply skip the affected regions.

### 21.8.2 `CycleInfo` and `CycleAnalysis`

LLVM 15 introduced `CycleInfo` as a generalization of `LoopInfo` that handles irreducible CFGs. A *cycle* is a maximal strongly connected subgraph. Every natural loop is a cycle, but a cycle with multiple entries is not a natural loop.

The key types are in `llvm/IR/CycleInfo.h`:

```cpp
using CycleInfo = GenericCycleInfo<SSAContext>;
using Cycle     = CycleInfo::CycleT;
```

The `CycleAnalysis` pass (`llvm/Analysis/CycleAnalysis.h`) computes `CycleInfo` for a function under the new pass manager:

```cpp
#include "llvm/Analysis/CycleAnalysis.h"
#include "llvm/IR/CycleInfo.h"

PreservedAnalyses MyCyclePass::run(Function &F, FunctionAnalysisManager &AM) {
  CycleInfo &CI = AM.getResult<CycleAnalysis>(F);

  // Iterate over top-level cycles
  for (Cycle *C : CI.toplevel_cycles()) {
    BasicBlock *Entry = C->getHeader();  // entry block of the cycle
    bool HasUniquePred = /* check if reducible */ true;
    for (BasicBlock *Pred : predecessors(Entry)) {
      if (!C->contains(Pred)) { /* outside predecessor */ }
    }
  }
  return PreservedAnalyses::all();
}
```

A cycle is reducible (a natural loop) if and only if it has exactly one entry block — the header — such that no block inside the cycle has a predecessor outside the cycle other than via the header. You can check this by comparing `C->getNumEntries()` to `1` (on the `GenericCycle` template, this is expressed by checking whether the entry set has size one).

### 21.8.3 Interaction with Existing Passes

`CycleInfo` is newer and less widely consumed than `LoopInfo`. Most existing LLVM passes use `LoopInfo` and simply do not optimize over irreducible regions. The intended role for `CycleInfo` is:

1. Passes that need to reason about *all* cycles (not just natural loops), e.g., divergence analysis for GPU targets.
2. Structural normalization passes that want to detect and potentially convert irreducible cycles into reducible form before handing off to standard loop optimizations.

The `GenericCycleInfo<SSAContext>` specialization is also instantiated for `MachineFunction`-level analysis (as `MachineCycleInfo`), allowing backend passes to use the same infrastructure.

---

## 21.9 Iterated Dominance Frontier: Infrastructure Details

### 21.9.1 `IDFCalculator`

`mem2reg` does not call the `DominanceFrontier` analysis directly. Instead, it uses the `IDFCalculator` template from `llvm/Analysis/IteratedDominanceFrontier.h`, which wraps the generic `IDFCalculatorBase` from `llvm/Support/GenericIteratedDominanceFrontier.h`:

```cpp
// From llvm/Analysis/IteratedDominanceFrontier.h:
template <bool IsPostDom>
class IDFCalculator final : public IDFCalculatorBase<BasicBlock, IsPostDom> { ... };

using ForwardIDFCalculator = IDFCalculator<false>;
using ReverseIDFCalculator = IDFCalculator<true>;
```

`mem2reg` instantiates `ForwardIDFCalculator` with the function's `DominatorTree`, sets the definition blocks, and calls `calculate()` to obtain the set of φ-insertion points:

```cpp
ForwardIDFCalculator IDF(DT);
IDF.setDefiningBlocks(DefBlocks);       // blocks with store-to-alloca
IDF.setLiveInBlocks(LiveInBlocks);      // pruning: only where var is live
SmallVector<BasicBlock *, 32> PHIBlocks;
IDF.calculate(PHIBlocks);               // output: IDF(DefBlocks)
```

The `setLiveInBlocks` call implements *pruned SSA*: φ-nodes are inserted only where the variable is actually live, reducing the number of trivial φ-nodes. Chapter 9 §9.7.2 covers the pruning theory; this is the implementation.

### 21.9.2 `DominanceFrontier` Analysis

The `DominanceFrontierAnalysis` (new PM) computes and caches the full dominance frontier for every block in the function. It is available from `llvm/Analysis/DominanceFrontier.h`:

```cpp
#include "llvm/Analysis/DominanceFrontier.h"

DominanceFrontier &DF = AM.getResult<DominanceFrontierAnalysis>(F);

// DF frontier for a specific block
const DominanceFrontier::DomSetType &Frontier = DF.find(BB)->second;
for (BasicBlock *DFBlock : Frontier) {
  // DFBlock is in DF(BB)
}
```

`DominanceFrontier` is rarely used directly in production passes — it is an O(|V|²) worst-case structure and most passes prefer the on-demand `IDFCalculator` approach. Its main value is for debugging, for educational implementations of SSA construction, and for passes that need the full DF table for all blocks simultaneously.

### 21.9.3 SSA Verification

LLVM's IR verifier (`llvm/IR/Verifier.h`) checks the SSA dominance property as part of its soundness checks: every use of an SSA value must be dominated by its definition. The verifier calls `DominatorTree` internally and reports a diagnostic for any violation:

```bash
# Run the verifier explicitly on an IR file
opt --passes="verify" -S input.ll
```

A typical verifier error for a dominance violation:
```
Instruction does not dominate all uses!
  %1 = add i32 %a, %b
  %3 = add i32 %1, %c
```

The verifier runs implicitly at every stage in debug builds (`LLVM_ENABLE_ASSERTIONS=ON`). In release builds, passes are expected to maintain the invariant and the verifier is only run explicitly when requested.

---

## 21.10 Chapter Summary

- **Clang uses the alloca-first strategy**: every local variable starts as an `alloca`; `mem2reg` runs early in the optimization pipeline and promotes allocas to φ-nodes using the Cytron IDF algorithm. This cleanly separates correct-code generation from efficient SSA construction.

- **`isAllocaPromotable`** is the predicate for `mem2reg` promotability: the alloca must be used only by `load`, `store`, and lifetime-marker instructions. SROA extends this to aggregates by slicing structs and arrays into independently promotable scalar allocas before `mem2reg` runs.

- **`DominatorTree`** is computed by `DominatorTreeAnalysis`; `DT.dominates(A, B)`, `DT.properlyDominates(A, B)`, and `DT.findNearestCommonDominator(A, B)` are the key queries. The `PostDominatorTree` is the dual structure used for control-dependence analysis. Both are invalidated whenever the CFG changes; passes communicate this via `PreservedAnalyses`.

- **`LoopInfo`** encodes the natural loop nest forest. `L->getHeader()`, `L->getLoopPreheader()`, `L->getLoopLatch()`, `L->getExitingBlock()`, and `L->getExitBlock()` are the structural queries. `L->isLoopInvariant(V)` tests invariance. All loop optimization passes require the loop to be in simplified form (preheader + single latch + dedicated exits).

- **`LoopSimplify`** (pass name `loop-simplify`) normalizes the loop by inserting a preheader, merging multiple latches, and adding dedicated exit blocks. It is an absolute prerequisite for LICM, vectorization, unrolling, and induction variable simplification.

- **Loop rotation** (`loop-rotate`) moves the exit condition from the header to the latch, enabling do-while code generation. After rotation, the latch and exiting block are the same block. LICM and vectorization expect rotated loops.

- **`LoopNest`** captures the structure of a loop nest; `getPerfectLoops(SE)` identifies maximal groups of perfectly-nested loops (no outer-level computation except induction management). Perfect nests are required for the polyhedral model and for loop tiling.

- **`CycleInfo`** (LLVM 15+) generalizes `LoopInfo` to irreducible CFGs using a strongly connected component decomposition. Every natural loop is a cycle; a cycle with multiple entries is irreducible. Standard LLVM loop optimizations require reducible CFGs; `CycleInfo` is consumed by GPU divergence analysis and structural normalization passes that operate on arbitrary CFGs.

---

*Cross-references:* [Chapter 9 — Intermediate Representations and SSA Construction](../part-02-compiler-theory/ch09-intermediate-representations-and-ssa.md) · [Chapter 16 — IR Structure](../part-04-llvm-ir/ch16-ir-structure.md) · [Chapter 59 — The New Pass Manager](../part-10-analysis-middle-end/ch59-new-pass-manager.md) · [Chapter 60 — Writing a Pass](../part-10-analysis-middle-end/ch60-writing-a-pass.md) · [Chapter 61 — Foundational Analyses](../part-10-analysis-middle-end/ch61-foundational-analyses.md) · [Chapter 63 — Loop Optimizations](../part-10-analysis-middle-end/ch63-loop-optimizations.md) · [Chapter 71 — The Polyhedral Model](../part-11-polyhedral-theory/ch71-the-polyhedral-model.md) · [Chapter 74 — Polly Architecture](../part-12-polly/ch74-polly-architecture.md)

*Reference links:* [`llvm/IR/Dominators.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/Dominators.h) · [`llvm/Analysis/LoopInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Analysis/LoopInfo.h) · [`llvm/Analysis/LoopNestAnalysis.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Analysis/LoopNestAnalysis.h) · [`llvm/Analysis/CycleAnalysis.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Analysis/CycleAnalysis.h) · [`llvm/Transforms/Utils/PromoteMemToReg.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Transforms/Utils/PromoteMemToReg.h) · [`llvm/Analysis/IteratedDominanceFrontier.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Analysis/IteratedDominanceFrontier.h) · [LLVM Language Reference — phi instruction](https://llvm.org/docs/LangRef.html#phi-instruction) · [Cooper, Harvey, Kennedy — "A Simple, Fast Dominance Algorithm" (2001)](https://www.cs.rice.edu/~keith/EMBED/dom.pdf)


---

@copyright jreuben11
