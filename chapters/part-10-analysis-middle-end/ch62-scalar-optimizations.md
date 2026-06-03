# Chapter 62 — Scalar Optimizations

*Part X — Analysis and the Middle-End*

Scalar optimizations — those that operate on a single function without loop-level restructuring or inter-procedural information — form the backbone of the optimization pipeline. They eliminate dead code, propagate constants, simplify control flow, number equivalent values, and combine redundant operations. This chapter covers the most important scalar passes in LLVM 22.1: DCE/ADCE/BDCE, GVN/NewGVN, SCCP/IPSCCP, InstCombine/InstSimplify, SimplifyCFG, Reassociate, EarlyCSE, CorrelatedValuePropagation, JumpThreading, and TailCallElim.

## Table of Contents

- [62.1 Dead Code Elimination (DCE, ADCE, BDCE)](#621-dead-code-elimination-dce-adce-bdce)
  - [62.1.1 DCE — Basic Dead Code Elimination](#6211-dce-basic-dead-code-elimination)
  - [62.1.2 ADCE — Aggressive DCE](#6212-adce-aggressive-dce)
  - [62.1.3 BDCE — Bit-Tracking DCE](#6213-bdce-bit-tracking-dce)
- [62.2 Global Value Numbering (GVN, NewGVN)](#622-global-value-numbering-gvn-newgvn)
  - [62.2.1 GVN](#6221-gvn)
  - [62.2.2 NewGVN](#6222-newgvn)
- [62.3 SCCP — Sparse Conditional Constant Propagation](#623-sccp-sparse-conditional-constant-propagation)
  - [62.3.1 IPSCCP](#6231-ipsccp)
- [62.4 InstCombine and InstSimplify](#624-instcombine-and-instsimplify)
  - [62.4.1 InstCombine](#6241-instcombine)
  - [62.4.2 InstSimplify](#6242-instsimplify)
- [62.5 SimplifyCFG](#625-simplifycfg)
- [62.6 Reassociate](#626-reassociate)
- [62.7 EarlyCSE](#627-earlycse)
- [62.8 CorrelatedValuePropagation](#628-correlatedvaluepropagation)
- [62.9 JumpThreading](#629-jumpthreading)
- [62.10 TailCallElim](#6210-tailcallelim)
- [Chapter Summary](#chapter-summary)

---

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

## 62.11 Constraint Elimination (ConstraintElimPass)

Array bounds checks, null pointer guards, and integer range tests inserted by the front end are often redundant after inlining and loop unrolling: the same check appears multiple times, or a prior branch has already established that the condition holds. `ConstraintElimPass` (implemented in [`llvm/lib/Transforms/Scalar/ConstraintElimination.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Scalar/ConstraintElimination.cpp)) eliminates these redundant integer comparison instructions using a linear arithmetic constraint graph derived from the CFG.

### Problem: Redundant Bounds Checks

Consider a loop that has been unrolled four times. Each unrolled body contains an `i < n` check before the array access. After unrolling, the path condition along the unrolled body already implies successive iterations satisfy `i < n`, `i+1 < n`, `i+2 < n`, `i+3 < n`. Without ConstraintElim, all four checks persist as live comparisons.

Similarly, after inlining a function that accepts a `span<T>`, the caller's check `ptr != nullptr` combined with the inlined assertion `size > 0` may together imply further array accesses within bounds — but only if the pass can reason about linear relationships between values.

### Algorithm

ConstraintElim builds a **linear arithmetic constraint graph** from the conditional branches in the CFG:

1. **Edge annotation:** Each `br i1 %cond, %true_bb, %false_bb` annotates the true-edge with the constraint implied by `%cond` and the false-edge with its negation. For example, `icmp slt i32 %i, %n` annotates the true successor with `i < n` and the false successor with `i >= n`.

2. **System of inequalities:** Constraints are represented as rows in a difference constraint system. Variables are SSA values; each constraint is `a*x + b*y ≤ c` for constants `a`, `b`, `c`. The pass restricts itself to **linear integer constraints** — additions, subtractions, and multiplications by constants.

3. **Forward propagation:** As the pass walks the dominator tree (in dominator order), it accumulates the set of constraints active on the current path. When it encounters a comparison instruction, it queries the constraint system: if the comparison result is already implied by the active constraints, the comparison is replaced with a constant `true` or `false`. Dead branches and their successors are then cleaned up by SimplifyCFG.

4. **Integer arithmetic reasoning:** The system handles derived facts. `i+1 < n` (from unrolled iteration) implies `i < n` (the check for the prior access). `i >= 0` combined with `i < n` implies `0 <= i && i < n`, sufficient to eliminate a bounds check on `a[i]`.

### Example

```c
// Source: after unrolling by 4, the compiler generates:
// check: i+0 < n, access a[i]
// check: i+1 < n, access a[i+1]
// check: i+2 < n, access a[i+2]
// check: i+3 < n, access a[i+3]
```

```llvm
; Before ConstraintElim (simplified):
  %c0 = icmp slt i32 %i,   %n    ; loop invariant: always true here
  %c1 = icmp slt i32 %i1,  %n    ; i+1 < n — implied by i < n-3 at loop header
  %c2 = icmp slt i32 %i2,  %n
  %c3 = icmp slt i32 %i3,  %n
  br i1 %c0, %safe0, %trap
  ; ... load a[i] ...
  br i1 %c1, %safe1, %trap
  ; ... load a[i+1] ...

; After ConstraintElim:
  ; c1, c2, c3 replaced by `true` — only c0 check remains (or it too is removed
  ; if the loop trip count proves i+3 < n at entry).
  %c0 = icmp slt i32 %i, %n
  br i1 %c0, %safe0, %trap
  ; ... load a[i], a[i+1], a[i+2], a[i+3] without redundant checks ...
```

### Pipeline Position and Enabling

ConstraintElimPass is enabled by default at `-O2` and above, having been added to LLVM's default pipeline around LLVM 16. It runs as a function pass after InstCombine and CVP, when value range information is richest:

```bash
# Run explicitly:
opt -passes='constraint-elimination' -S input.ll

# Observe ConstraintElimPass in the O2 pipeline:
opt -passes='default<O2>' -print-pipeline-passes -S /dev/null 2>&1 | grep -i constraint
```

### Interaction with Sanitizers

When code is compiled with `-fsanitize=address` (ASan) or `-fsanitize=undefined` (UBSan), those sanitizers insert their own instrumented checks into the IR *after* the front-end's bounds checks. ConstraintElimPass is sanitizer-aware: it identifies sanitizer-inserted check sequences (recognizable by their `__ubsan_handle_*` or `__asan_report_*` call patterns) and does **not** eliminate them. The pass treats sanitizer checks as having side effects, just as it treats stores and function calls — they are never proven redundant.

### Debugging

```bash
# Show constraints added and removed during the pass:
opt -passes='constraint-elimination' -debug-only=constraint-elimination -S input.ll 2>&1
```

The debug output prints each constraint added to the system when entering a basic block and each comparison that is proven redundant and removed.

### Limitations

- **Linear integer constraints only.** The pass handles additions, subtractions, and multiplications by compile-time constants. Non-linear expressions (`i * j < n`, `i * i < n`) are not handled; the constraint is conservatively treated as `overdefined`.
- **No floating-point.** Floating-point comparisons are not represented in the constraint system.
- **No inter-procedural reasoning.** The pass is a function-level transform; it does not use inter-procedural value range information (that is IPSCCP's domain).
- **Scalability limit.** The difference constraint system is bounded in size; on functions with very large CFGs the pass may conservatively skip blocks to avoid quadratic blowup.

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
- `ConstraintElimPass` builds a linear arithmetic constraint graph from CFG branch conditions and eliminates comparison instructions that are provably redundant under the path condition; enabled by default at `-O2` and above; limited to linear integer constraints; sanitizer-instrumented checks are never eliminated.


---

@copyright jreuben11
