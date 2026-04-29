# Chapter 63 — Loop Optimizations

*Part X — Analysis and the Middle-End*

Loops dominate program execution time in numeric and systems code. The LLVM loop optimization pipeline transforms loop structure to improve instruction-level parallelism, cache behavior, and vectorization opportunities. This chapter covers the key loop passes: LICM, LoopRotate, LoopUnroll, IndVarSimplify, LoopUnswitch, LoopIdiom, LoopDeletion, LoopFusion, LoopDistribute, LoopInterchange, LoopFlatten, and LoopVersioning.

## 63.1 Loop Preprocessing: Simplify Form

Before most loop optimization passes run, loops must be in *simplify form*:
- A unique **preheader** (single predecessor of the loop header from outside).
- A single **latch** (predecessor of the loop header from inside).
- A single **exit** in LCSSA (loop-closed SSA) form (values defined inside the loop and used outside are passed through a phi in the exit block).

The `LoopSimplifyPass` and `LCSSAPass` ensure these invariants. Most loop passes assert that their input is in simplify form.

```bash
opt -passes='loop-simplify,lcssa' -S input.ll
```

## 63.2 LICM — Loop-Invariant Code Motion

`LICMPass` (in `llvm/Transforms/Scalar/LICM.h`) hoists loop-invariant computations to the preheader (or sinks them to the exit) so they execute once rather than on every iteration.

**Hoist condition**: an instruction is loop-invariant if all its operands are loop-invariant (defined outside the loop or themselves hoistable) and it has no side effects that preclude movement (stores cannot be hoisted freely; loads can be hoisted if they cannot alias a store in the loop).

```
; Before LICM:
preheader:
  br %loop.header

loop.header:
  %i = phi i64 [0, %preheader], [%i.next, %loop.body]
  %n_squared = mul i64 %n, %n  ; loop-invariant!
  ...
  br %loop.body

; After LICM:
preheader:
  %n_squared = mul i64 %n, %n  ; hoisted
  br %loop.header
```

LICM uses MemorySSA to prove loads are safe to hoist: a load is hoistable if no store in the loop aliases the loaded pointer (checked via `MemorySSA::getWalker()->getClobberingMemoryAccess`).

```bash
opt -passes='function(loop-mssa(licm))' -S input.ll
```

The `loop-mssa` wrapper builds MemorySSA for the loop passes that need it.

## 63.3 LoopRotate

`LoopRotatePass` transforms a loop from header-tested form (condition at the top) to latch-tested form (condition at the bottom), exposing the loop body to more optimization:

```
; Before rotation (while loop):
header:
  br %cond, %body, %exit
body:
  ... loop work ...
  br %header

; After rotation (do-while loop):
preheader:
  br %cond.init, %body, %exit    ; initial check
body:
  ... loop work ...
  br %cond, %body, %exit          ; condition at latch
```

The rotated form:
1. Enables **LICM** to run more aggressively (the exit condition is in the latch, not the header, so more code can be moved to the preheader).
2. Enables **loop vectorization** (vectorizers prefer the latch-tested form).
3. Reduces the number of branches executed per iteration by one.

```bash
opt -passes='function(loop-rotate)' -S input.ll
```

## 63.4 IndVarSimplify

`IndVarSimplifyPass` canonicalizes induction variables (IVs) in loops. It:
1. Replaces `SCEVAddRecExpr`-typed IVs with a single canonical IV (an integer counting from 0).
2. Eliminates redundant IVs (two IVs with the same step can share a base).
3. Folds IV-dependent expressions: if `%j = 2 * %i + 1` and `%i` is the canonical IV, `%j` can be replaced with `2*%canonical + 1`, potentially eliminating `%j`'s definition.
4. Sets the loop's back-edge taken count as loop metadata for downstream passes.

IndVarSimplify uses `ScalarEvolution` to analyze IV expressions and replaces them with simpler canonical forms.

```bash
opt -passes='function(indvars)' -S input.ll
```

## 63.5 LoopUnroll

`LoopUnrollPass` replicates the loop body N times, reducing the iteration count by N and exposing N times as much code for instruction-level parallelism and other optimizations.

```
; Original (trip count 8):
for i in 0..8:
  a[i] += 1;

; After unroll by 4:
for i in 0..8 step 4:
  a[i]   += 1;
  a[i+1] += 1;
  a[i+2] += 1;
  a[i+3] += 1;
```

The unroll factor is determined by the backend's `TargetTransformInfo::getUnrollPreferences()`, which considers register pressure, instruction throughput, and the loop's trip count.

**Full unroll**: if the trip count is a compile-time constant and the unrolled body fits in the unroll threshold, the loop is fully unrolled (no branch remains).

```bash
opt -passes='function(loop-unroll<count=4>)' -S input.ll
opt -passes='function(loop-unroll-and-jam<count=4>)' -S input.ll
```

**Unroll-and-jam**: for nested loops, `LoopUnrollAndJamPass` unrolls the outer loop and fuses (jams) the resulting copies of the inner loop into a single loop, improving data reuse:

```
; Before unroll-and-jam (outer unroll by 2):
for i in 0..N:
  for j in 0..M:
    C[i][j] += A[i][k] * B[k][j];

; After:
for i in 0..N step 2:
  for j in 0..M:
    C[i][j]   += A[i][k]   * B[k][j];
    C[i+1][j] += A[i+1][k] * B[k][j];
```

## 63.6 LoopUnswitch

`SimpleLoopUnswitchPass` moves loop-invariant conditional branches *outside* the loop:

```
; Before:
loop:
  if (invariant_cond)   ; checked on every iteration
    body_A;
  else
    body_B;

; After:
if (invariant_cond)
  loop: body_A;         ; version A of the loop
else
  loop: body_B;         ; version B of the loop
```

Each unswitched version of the loop has the branch removed, exposing the loop body to more aggressive optimization. The code size doubles (one loop per branch arm), so unswitching is controlled by a size threshold.

```bash
opt -passes='function(simple-loop-unswitch)' -S input.ll
opt -passes='function(simple-loop-unswitch<nontrivial>)' -S input.ll
```

The `nontrivial` flag enables unswitching of branches not in the loop header.

## 63.7 LoopIdiomRecognize

`LoopIdiomRecognizePass` identifies loops that implement well-known patterns and replaces them with library calls:

- **Memset**: a loop that stores a constant to every element of a contiguous array → `@llvm.memset`.
- **Memcpy**: a loop that copies array A to B element by element → `@llvm.memcpy`.
- **Memmove**: similar to memcpy, with possible overlap → `@llvm.memmove`.
- **String length**: a loop scanning for a null byte → `@llvm.strlen`.
- **Population count**: a loop counting set bits → `@llvm.ctpop`.

```bash
opt -passes='function(loop-idiom)' -S input.ll
```

These replacements enable the backend to use highly optimized hardware instructions (`REP STOSQ`, vector memset, etc.) instead of scalar loop iterations.

## 63.8 LoopDeletion

`LoopDeletionPass` removes loops whose bodies have no observable effects (no stores, no calls with side effects, no memory writes that escape):

```
; Useless loop — body computes a value only used inside the loop
loop:
  %sum = phi i32 [0, %preheader], [%sum.next, %loop]
  %sum.next = add i32 %sum, 1
  br %cond, %loop, %exit
exit:
  ; %sum is not used after the loop
  ret void
```

ADCE removes individual dead instructions; LoopDeletion removes dead loops as a unit when the trip count is known or the loop is provably non-terminating with no side effects.

```bash
opt -passes='function(loop-deletion)' -S input.ll
```

## 63.9 LoopFusion

`LoopFusePass` (in `llvm/Transforms/Scalar/LoopFuse.h`) merges two adjacent loops with the same trip count and loop bounds into a single loop, improving data reuse and register usage:

```
; Before:
for i in 0..N: A[i] = f(i);
for i in 0..N: B[i] = g(A[i]);

; After fusion (if safe):
for i in 0..N:
  A[i] = f(i);
  B[i] = g(A[i]);
```

Fusion is legal only if the loops have no dependences that prevent reordering — a loop that reads an element written by a prior loop at the same index is fusible; one that reads a *later* element would create a cycle.

```bash
opt -passes='function(loop-fusion)' -S input.ll
```

LoopFusion uses `DependenceAnalysis` to check dependences and `ScalarEvolution` to verify matching trip counts.

## 63.10 LoopDistribute

`LoopDistributePass` splits a loop into multiple loops when it contains independent groups of operations, enabling vectorization of the independent groups:

```
; Original (mixes independent and dependent operations):
for i:
  A[i] = B[i] + C[i];   ; vectorizable
  D[i] = D[i-1] + E[i]; ; loop-carried dependence, not vectorizable

; After distribution:
for i: A[i] = B[i] + C[i];   ; vectorizable
for i: D[i] = D[i-1] + E[i]; ; not vectorizable, but no longer blocking A
```

Distribution increases loop count but enables vectorization of the independent part, often leading to a net performance gain.

```bash
opt -passes='function(loop-distribute)' -S input.ll
```

## 63.11 LoopInterchange

`LoopInterchangePass` swaps nested loop order to improve cache access patterns. A loop nest `for i: for j:` that accesses `A[j][i]` (column-major in a row-major array) has poor spatial locality. Interchanging to `for j: for i:` makes accesses to `A[j][i]` sequential.

```bash
opt -passes='function(loop-interchange)' -S input.ll
```

Interchange legality requires that no data dependence prevents it (checked via `DependenceAnalysis`).

## 63.12 LoopFlatten

`LoopFlattenPass` fuses a perfectly nested loop nest into a single loop when the bounds are independent:

```
; Before (nest with j running 0..N for each i):
for i in 0..M:
  for j in 0..N:
    A[i*N + j] = 0;

; After:
for ij in 0..M*N:
  A[ij] = 0;
```

The flattened form is easier to vectorize (single-level loop vectorizer handles it) and reduces loop overhead.

```bash
opt -passes='function(loop-flatten)' -S input.ll
```

## 63.13 LoopVersioning and LoopVersioningLICM

`LoopVersioningLICMPass` creates two versions of a loop: one that assumes no aliasing (and is optimized more aggressively) and one that is conservative, with a runtime alias check selecting between them:

```
; Runtime check: if A and B don't alias:
if (A + len <= B || B + len <= A)
  <optimized loop with LICM-hoisted loads>
else
  <safe fallback loop>
```

This is valuable when the alias relationship between two pointer arguments is not statically determinable but is almost always non-aliasing at runtime.

```bash
opt -passes='function(loop-versioning-licm)' -S input.ll
```

---

## Chapter Summary

- Loops must be in simplify form (unique preheader, single latch, LCSSA) before most loop passes; `LoopSimplifyPass` and `LCSSAPass` ensure this.
- `LICMPass` hoists loop-invariant code to the preheader, using MemorySSA to safely hoist loads; stores require aliasing proof to hoist.
- `LoopRotatePass` converts header-tested loops to latch-tested form, enabling more aggressive LICM and vectorization.
- `IndVarSimplifyPass` canonicalizes induction variables using ScalarEvolution, replacing multiple IVs with a single canonical IV.
- `LoopUnrollPass` replicates the body N times; `LoopUnrollAndJamPass` unrolls an outer loop and fuses inner loop copies for better cache reuse.
- `SimpleLoopUnswitchPass` moves loop-invariant conditions outside the loop, creating two specialized loop versions.
- `LoopIdiomRecognizePass` replaces recognized patterns (memset/memcpy/strlen/ctpop loops) with intrinsic calls.
- `LoopDeletionPass` removes loops with no observable effects when the loop body is proven dead.
- `LoopFusePass` merges adjacent loops with matching trip counts when dependences allow, improving data reuse.
- `LoopDistributePass` splits independent operations into separate loops to enable partial vectorization.
- `LoopInterchangePass` swaps nested loop order to improve spatial locality; `LoopFlattenPass` collapses perfectly nested loops into a single loop.
- `LoopVersioningLICMPass` creates aliasing-optimized and conservative loop versions selected by a runtime check.


---

@copyright jreuben11
