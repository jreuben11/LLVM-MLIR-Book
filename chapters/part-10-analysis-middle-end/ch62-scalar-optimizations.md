# Chapter 62 — Scalar Optimizations

*Part X — Analysis and the Middle-End*

Scalar optimizations — those that operate on a single function without loop-level restructuring or inter-procedural information — form the backbone of the optimization pipeline. They eliminate dead code, propagate constants, simplify control flow, number equivalent values, and combine redundant operations. This chapter covers the most important scalar passes in LLVM 22.1: DCE/ADCE/BDCE, GVN/NewGVN, SCCP/IPSCCP, InstCombine/InstSimplify, SimplifyCFG, Reassociate, EarlyCSE, CorrelatedValuePropagation, JumpThreading, and TailCallElim.

## 62.1 Dead Code Elimination (DCE, ADCE, BDCE)

### 62.1.1 DCE — Basic Dead Code Elimination

`DCEPass` (in `llvm/Transforms/Scalar/DCE.h`) is the simplest dead-code eliminator. It marks an instruction dead if it has no users and no side effects (not a store, call, branch, or other observable operation), then removes it. It runs in O(n) time over the instruction list.

```bash
opt -passes='dce' -S input.ll
```

The key limitation: DCE cannot remove instructions involved in cycles (e.g., a loop computing a value that is only used by the loop itself, with no exit use). That is ADCE's domain.

### 62.1.2 ADCE — Aggressive DCE

`ADCEPass` (in `llvm/Transforms/Scalar/ADCE.h`) uses the reverse dataflow approach: it starts by assuming every instruction is dead, then marks instructions live if they are "live roots" (stores, calls, terminators) or are used by a live instruction. The reverse propagation continues until no more instructions become live. Anything still unmarked is dead.

ADCE handles loops: a loop that only computes a value used by the loop itself, with no output reaching a live instruction, is entirely eliminated. This subsumes basic loop deletion.

```bash
opt -passes='adce' -S input.ll
```

ADCE may modify the CFG (by removing dead branches), so it invalidates CFG analyses.

### 62.1.3 BDCE — Bit-Tracking DCE

`BDCEPass` (in `llvm/Transforms/Scalar/BDCE.h`) combines `DemandedBitsAnalysis` with dead-code elimination. Rather than asking "is this value used?", it asks "which bits of this value are used?". If no bits are demanded, the instruction is dead. If some bits are demanded but the instruction provides more, the undemanded bits can sometimes be cleared to expose further simplifications.

```bash
opt -passes='bdce' -S input.ll
```

BDCE is effective after InstCombine, where many high bits may have been proven to be zeros.

## 62.2 Global Value Numbering (GVN, NewGVN)

### 62.2.1 GVN

`GVNPass` (in `llvm/Transforms/Scalar/GVN.h`) assigns value numbers to expressions: two expressions with the same value number compute the same value. It uses dominance-based value numbering: if computation A dominates computation B and they are equivalent, B is replaced with A.

GVN performs:
- **Full redundancy elimination (FRE)**: removes a computation that is available along all paths to a use.
- **Partial redundancy elimination (PRE)**: inserts a computation on the path where it is not available, then removes the redundant computation (load PRE).
- **Dead load elimination**: a load whose value is already known (through a prior store or load) is replaced.

```bash
opt -passes='gvn' -S input.ll
```

GVN requires `DominatorTree`, `MemorySSA`, and `AliasAnalysis`. It preserves `DominatorTree` but not `MemorySSA` (since it may insert instructions).

### 62.2.2 NewGVN

`NewGVNPass` is a reimplementation of GVN with a more principled congruence-based value numbering. It handles more equivalences (particularly around memory operations) and is more predictable, but is slightly slower than `GVNPass`. It is currently in the process of replacing GVN in the default pipeline.

```bash
opt -passes='newgvn' -S input.ll
```

## 62.3 SCCP — Sparse Conditional Constant Propagation

`SCCPPass` implements the classic Wegman-Zadeck algorithm (Chapter 10 §lattice). It propagates constant values through the program using a lattice:

```
top (unknown)
    ↓
constant C   (exactly known)
    ↓
bottom (overdefined — not a constant)
```

Operations on lattice values: `constant(3) + constant(4) = constant(7)`, `constant(3) + overdefined = overdefined`. Conditional branches with constant conditions become unconditional, enabling further propagation along the live path.

```bash
opt -passes='sccp' -S input.ll
```

SCCP is applied before inlining (to reduce call argument variability) and after inlining (to propagate constants from inlined bodies).

### 62.3.1 IPSCCP

`IPSCCPPass` is the inter-procedural version: it propagates constants across function boundaries, treating non-public functions as candidates for specialization. If a function is always called with the same constant argument, IPSCCP propagates that constant into the function.

```bash
opt -passes='ipsccp' -S input.ll
```

IPSCCP requires the entire module and can interact with `FunctionAttrs` inference to mark functions `readonly`, `readnone`, etc.

## 62.4 InstCombine and InstSimplify

### 62.4.1 InstCombine

`InstCombinePass` (in `llvm/Transforms/InstCombine/`) is the most complex scalar pass. It applies thousands of algebraic identity rewrites to instruction sequences, always replacing the matched pattern with a simpler or fewer-instruction equivalent. Key properties:

- Every rewrite reduces the instruction count or lowers the instruction cost.
- Rewrites are bidirectional: the canonical form may be more or less complex than the source.
- Combines across multiple instructions: `(a << 1) + a → a * 3 → a * 3` (or `imul` on x86).

InstCombine is driven by `InstructionCombining.cpp` and a large set of visitor methods in `InstCombine{Casts,Calls,Compare,LoadStoreAlloca,Shifts,Vectors}.cpp`.

Example rewrites:
```
; Algebraic:
%1 = sub i32 %x, %x    → 0
%2 = or i32 %x, 0      → %x
%3 = mul i32 %x, -1    → neg %x → sub i32 0, %x

; Strength reduction:
%4 = udiv i32 %x, 4   → lshr i32 %x, 2

; Cast simplification:
%5 = sext i8 (trunc i32 %x to i8) to i32 → sextint if %x is in [-128, 127]

; Compare folding:
%cmp = icmp slt i32 %x, 0
%sel = select i1 %cmp, i32 -1, i32 0   → ashr i32 %x, 31
```

```bash
opt -passes='instcombine' -S input.ll
opt -passes='instcombine<max-iterations=3>' -S input.ll
```

### 62.4.2 InstSimplify

`InstSimplifyPass` is a lighter-weight, side-effect-free version of InstCombine. It only performs rewrites that can be applied without scanning other instructions:
- Constant folding
- Identity operations (`x + 0 → x`, `x & -1 → x`)
- Obvious impossibilities (`1 == 2 → false`)

InstSimplify does not modify the CFG and has no iteration limit. It is used as a subroutine within other passes (e.g., GVN uses `simplifyInstruction` to constant-fold expressions it encounters).

## 62.5 SimplifyCFG

`SimplifyCFGPass` cleans up the control-flow graph:

- **Empty block elimination**: merges a block that has only a `br` to its successor.
- **Phi simplification**: `phi [%x, pred1], [%x, pred2] → %x`.
- **Conditional branch with constant condition**: `br i1 true, %then, %else → br %then`.
- **Common-successor tail merging**: if two branches end in the same tail (identical instructions), merge them.
- **Switch-to-branch**: converts `switch` with few cases to a chain of `icmp`/`br`.
- **Unreachable block removal**.

```bash
opt -passes='simplifycfg' -S input.ll
```

SimplifyCFG runs frequently in the pipeline (before inlining, after inlining, after loop transformations) to keep the CFG in canonical form.

## 62.6 Reassociate

`ReassociatePass` reorders commutative and associative operations to expose common subexpressions:

```
; Source:
; (a + b) + (a + c)  →  (a + a) + (b + c)  →  mul(2,a) + (b+c)

; Before Reassociate:
%t1 = add i32 %a, %b
%t2 = add i32 %a, %c
%t3 = add i32 %t1, %t2

; After Reassociate:
%aa = mul i32 %a, 2     ; or add %a, %a
%bc = add i32 %b, %c
%t3 = add i32 %aa, %bc
```

Reassociate assigns a "rank" to each value (based on loop depth and instruction distance from entry) and reorders operands to pair high-rank values together. This enables GVN to find common subexpressions that were hidden by different association trees.

```bash
opt -passes='reassociate' -S input.ll
```

## 62.7 EarlyCSE

`EarlyCSEPass` performs common subexpression elimination using a simple hash-based dominance-order traversal (depth-first over the dominator tree). It handles:
- Pure expression CSE (same operation + same operands)
- Load CSE (a load from a location previously loaded, with no intervening stores)
- Store-to-load forwarding (a load immediately after a store to the same location)

EarlyCSE runs early in the pipeline before GVN because it is much faster (O(n) vs GVN's O(n²) worst case) and catches obvious redundancies:

```bash
opt -passes='early-cse' -S input.ll
opt -passes='early-cse<memssa>' -S input.ll   # with MemorySSA for load/store
```

The `memssa` variant uses MemorySSA for more precise load/store tracking.

## 62.8 CorrelatedValuePropagation

`CorrelatedValuePropagationPass` uses `LazyValueInfo` to propagate value range information through conditional branches. When a branch checks `%x > 0`, on the true-side `%x` is known to be positive; CVP uses this to:

- Simplify comparisons: `%x > -1` on the true-side → `true`.
- Remove `overflow` checks: `add nsw %x, 1` where `%x < INT_MAX` is proved.
- Replace `select` with direct values: `select (x > 0), x, 0` where x is always positive.

```bash
opt -passes='correlated-propagation' -S input.ll
```

CVP works best on code with explicit conditional checks (bounds checking, null checks) that can be proven redundant on one side of the branch.

## 62.9 JumpThreading

`JumpThreadingPass` duplicates code to "thread" paths through a basic block that would otherwise merge unnecessarily. If a block branches based on a condition that is known on only some incoming paths, JumpThreading duplicates the block and specializes it for the paths where the condition is known:

```
; Before:
BB1: br %cond, %join, %other
BB2: br %join
join:
  %cond2 = ... based on %cond ...
  br %cond2, %target1, %target2

; After: BB1 branches directly to %target1 or %target2
;        BB2 branches to %join (as before)
```

This exposes new constant-folding and loop optimization opportunities by specializing code paths.

```bash
opt -passes='jump-threading' -S input.ll
```

JumpThreading is O(n²) in the worst case and has iteration limits. It uses `LazyValueInfo` to determine which conditions are known on incoming edges.

## 62.10 TailCallElim

`TailCallEliminationPass` converts tail calls (calls immediately followed by `ret`) to `musttail` calls, which the backend lowers to direct jumps:

```
; Before (tail call):
define i64 @factorial(i64 %n, i64 %acc) {
  ...
  %result = call i64 @factorial(i64 %sub, i64 %mul) ; tail call
  ret i64 %result
}

; After tail-call annotation:
  %result = tail call i64 @factorial(i64 %sub, i64 %mul)
  ret i64 %result
```

The `tail` marker tells the backend that the callee may reuse the caller's stack frame. For direct recursion, `musttail` (a stronger guarantee) enforces this reuse, converting recursion to iteration in the backend.

TailCallElim also converts mutual tail recursion to `musttail` when it can prove the call is in tail position.

```bash
opt -passes='tailcallelim' -S input.ll
```

---

## Chapter Summary

- `DCEPass` removes instructions with no users and no side effects; `ADCEPass` uses reverse dataflow to remove dead computations including whole loops; `BDCEPass` uses demanded-bit analysis to find partially-dead instructions.
- `GVNPass` and `NewGVNPass` assign value numbers to remove fully- and partially-redundant computations; GVN also performs load PRE and dead load elimination.
- `SCCPPass` implements the Wegman-Zadeck lattice algorithm for sparse conditional constant propagation; `IPSCCPPass` extends it inter-procedurally.
- `InstCombinePass` applies thousands of algebraic rewrites (strength reduction, cast simplification, compare folding) to reduce instruction count; `InstSimplifyPass` is the side-effect-free lightweight subset.
- `SimplifyCFGPass` eliminates empty blocks, constant-condition branches, and tail-merging opportunities, keeping the CFG in canonical form.
- `ReassociatePass` reorders commutative/associative operations to group common subexpressions, enabling downstream GVN to find them.
- `EarlyCSEPass` performs fast hash-based CSE in dominator order, including load CSE and store-to-load forwarding.
- `CorrelatedValuePropagationPass` uses `LazyValueInfo` to propagate value ranges through conditional branches, simplifying redundant comparisons.
- `JumpThreadingPass` duplicates blocks to thread known-condition paths, specializing code for specific path predicates.
- `TailCallEliminationPass` marks tail calls with `tail`/`musttail`, enabling the backend to convert tail recursion to iteration.


---

@copyright jreuben11
