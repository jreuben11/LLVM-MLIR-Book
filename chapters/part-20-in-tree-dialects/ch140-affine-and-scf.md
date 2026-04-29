# Chapter 140 — Affine and SCF

*Part XX — In-Tree Dialects*

Between the high-level array abstractions of `linalg` and `tensor` and the low-level control flow of `cf` sit two dialects that represent loops and conditionals in structured forms with different degrees of expressiveness: `affine` provides polyhedral-grade loop and memory access representation with static analysis capabilities drawn from the polyhedron model, while `scf` (Structured Control Flow) provides general structured loops and conditionals that can express arbitrary dynamic control without committing to polyhedral constraints. Both dialects are essential stages in the MLIR compilation pipeline, with `affine` enabling powerful automatic optimization and `scf` serving as the universal intermediate between high-level structured dialects and low-level `cf`. This chapter covers both from first principles, including their analysis capabilities, transformation passes, and lowering paths.

---

## 140.1 The `affine` Dialect

### 140.1.1 Polyhedral Representation in MLIR

The `affine` dialect ([`mlir/include/mlir/Dialect/Affine/IR/AffineOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Affine/IR/AffineOps.td)) represents loop nests and memory accesses as affine expressions over integer variables — the core idea from the polyhedral model. An affine expression is a linear combination of symbolic integers with integer coefficients:

```
affine_expr = c₀ + c₁·v₁ + c₂·v₂ + ... + cₙ·vₙ
```

where `v₁...vₙ` are loop induction variables or symbolic constants. Affine constraints (systems of linear inequalities) define iteration domains, and affine maps define memory access patterns.

The restriction to affine computations over loop bounds and array indices is precisely what enables static dependence analysis, loop transformations, and automatic parallelization — things that are impossible with general (non-affine) loop bodies.

### 140.1.2 `affine.for`

```mlir
// Static bounds
affine.for %i = 0 to 16 step 4 {
  affine.for %j = 0 to 16 {
    // body
  }
}

// Affine expression bounds
affine.for %i = 0 to affine_map<()[s0] -> (s0)>()[%n] {
  // upper bound is symbolic %n
}

// Min/max bounds (for tiling remainder handling)
affine.for %i = 0 to min affine_map<(d0)[s0] -> (d0 + 4, s0)>(%base)[%n] {
  // runs min(base+4, n) iterations
}

// Loop-carried values
%sum = affine.for %i = 0 to %n iter_args(%acc = %init) -> f32 {
  %elem = affine.load %arr[%i] : memref<?xf32>
  %new = arith.addf %acc, %elem : f32
  affine.yield %new : f32
}
```

The `step` defaults to 1 if omitted. Loop bounds must be affine expressions in outer loop induction variables and symbolic values (module-level constants or function arguments).

The `iter_args` mechanism (matching `scf.for`'s design) allows loop-carried values without mutable registers, maintaining SSA form.

### 140.1.3 `affine.load` and `affine.store`

```mlir
// Access with affine index expressions
%val = affine.load %A[%i * 4 + %j, %k] : memref<16x16xf32>
affine.store %val, %B[%j, %i] : memref<16x16xf32>

// With symbolic dimensions
%val = affine.load %A[affine_map<(d0, d1) -> (d0 + d1)>(%i, %j)]
    : memref<?xf32>

// Explicit affine map form
#map = affine_map<(d0, d1)[s0] -> (d0 * s0 + d1)>
%val = affine.load %A[#map(%i, %j)[%stride]] : memref<?xf32>
```

The index expressions in `affine.load`/`affine.store` must be affine — this is what enables dependence analysis. An access `A[i*4 + j]` has a known pattern; an access `A[f(i)]` where `f` is a non-affine function does not.

The affine constraint means: indices can only be linear combinations of outer loop induction variables and symbolic integer parameters (function arguments, affine.for-derived values), with integer coefficients and integer constant terms.

### 140.1.4 `affine.if`

```mlir
// Conditional based on affine constraint set
affine.if affine_set<(d0)[s0] : (d0 >= 0, s0 - d0 - 1 >= 0)>(%i)[%n] {
  // body: executes when 0 ≤ i ≤ n-1
}

// With else region
%result = affine.if affine_set<(d0) : (d0 - 5 >= 0)>(%i) -> f32 {
  %a = arith.constant 1.0 : f32
  affine.yield %a : f32
} else {
  %b = arith.constant 0.0 : f32
  affine.yield %b : f32
}
```

The condition is an `IntegerSet` — a conjunction of affine inequalities. This corresponds to polyhedral "guards" that restrict iteration domains. They enable peeling, skewing, and partial unrolling while maintaining analyzability.

### 140.1.5 `affine.apply`, `affine.min`, `affine.max`

```mlir
// Evaluate an affine expression → index value
%base = affine.apply affine_map<(d0) -> (d0 * 4)>(%i)

// Minimum of multiple affine expressions (for tiling bounds)
%tile_size = affine.min affine_map<(d0)[s0] -> (d0 + 4, s0)>(%base)[%n]
// = min(base + 4, n) — handles the last tile potentially being smaller

// Maximum
%max = affine.max affine_map<(d0, d1) -> (d0, d1)>(%a, %b)
```

`affine.min`/`affine.max` are critical for tiled loop bounds. When tiling a loop of length `N` by `T`, the last tile may have fewer than `T` elements. `affine.min` expresses this: the tile size is `min(T, N - tile_base)`.

### 140.1.6 `affine.parallel`

```mlir
// Parallel loop nest — all iterations are independent
affine.parallel (%i, %j) = (0, 0) to (16, 16) step (4, 4) {
  // This body can execute in any order or simultaneously
}

// Parallel with reduction
%sum = affine.parallel (%i) = (0) to (64) reduce ("addf") -> f32 {
  %elem = affine.load %arr[%i] : memref<64xf32>
  affine.yield %elem : f32
}
```

`affine.parallel` differs from `affine.for` in that it makes the independence of iterations explicit in the IR. It is the output of `--affine-parallelize` and the input for OpenMP or GPU thread mapping.

---

## 140.2 Affine Analysis

### 140.2.1 Dependence Analysis

The `affine` dialect's primary value-add over `scf` is enabling precise dependence analysis. The analysis framework ([`mlir/include/mlir/Analysis/AffineAnalysis.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Analysis/AffineAnalysis.h)) implements:

```cpp
// Test whether two accesses to the same memref may conflict
DependenceResult checkMemrefAccessDependence(
    const MemRefAccess &srcAccess,
    const MemRefAccess &dstAccess,
    unsigned loopDepth,
    FlatAffineValueConstraints *dependenceConstraints,
    SmallVectorImpl<DependenceComponent> *dependenceComponents,
    bool allowRAR = false);
```

The algorithm:
1. For each pair of `affine.load`/`affine.store` accessing the same memref
2. Compute the access relation as a system of linear constraints
3. Apply the Omega test (or Farkas lemma for simpler cases) to check satisfiability
4. If the system is satisfiable, a dependence exists

A dependence at loop depth `d` means: there exist iterations `(i₀, ..., iₐ)` and `(j₀, ..., jₐ)` of the `d`-deep loop nest, in lexicographic order, such that both access the same memory location, with at least one being a write.

This analysis powers all affine transformations: you can permute loops only if no dependences are created; you can parallelize a loop only if no loop-carried dependences exist along that loop's dimension.

### 140.2.2 Built-in Transformation Passes

```bash
# Loop tiling (tile sizes specified per loop)
mlir-opt --affine-loop-tile="tile-size=4" input.mlir

# Loop unrolling
mlir-opt --affine-loop-unroll="unroll-factor=4" input.mlir
mlir-opt --affine-loop-unroll="unroll-full" input.mlir

# Loop unroll-and-jam (unroll outer, fuse inner)
mlir-opt --affine-loop-unroll-jam="unroll-jam-factor=4" input.mlir

# Loop vectorization (innermost affine loops to vector dialect)
mlir-opt --affine-vectorize="virtual-vector-size=8" input.mlir

# Loop fusion (fuse adjacent loops with compatible bounds)
mlir-opt --affine-loop-fusion input.mlir

# Automatic parallelization
mlir-opt --affine-parallelize input.mlir

# Pipeline inner loops (software pipelining)
mlir-opt --affine-pipeline-data-transfer input.mlir

# Scalar replacement (eliminate redundant loads/stores)
mlir-opt --affine-scalrep input.mlir
```

### 140.2.3 `AffineLoopBandNestResult`

Loop bands are contiguous sequences of perfectly nested `affine.for` loops. The `getLoopBandNestResult` utility identifies bands and their dependences:

```cpp
// Get all loop bands in a function
SmallVector<SmallVector<AffineForOp>> bands;
getLoopBandNestResults(funcOp, bands);

// For each band, compute the dependence graph
for (auto &band : bands) {
  FlatAffineValueConstraints constraints;
  for (unsigned i = 0; i < band.size(); ++i) {
    // Analyze dependences at depth i
  }
}
```

This powers `--affine-loop-fusion`: fuse loops only when no dependences prevent it, and only when the fusion improves locality (producer's output is consumed by the fused consumer without intermediate storage).

### 140.2.4 Lowering Affine

```bash
# Complete affine lowering
mlir-opt \
  --lower-affine \    # affine.for/if/load/store → scf + memref
  input.mlir
```

`--lower-affine` (equivalent to `--convert-affine-to-scf` + `--convert-affine-to-memref`) converts:
- `affine.for` → `scf.for`
- `affine.if` → `scf.if`  
- `affine.parallel` → `scf.parallel`
- `affine.load`/`affine.store` → `memref.load`/`memref.store` with computed index expressions
- `affine.apply` → arithmetic on `index` type
- `affine.min`/`affine.max` → `arith.minsi`/`arith.maxsi` chains

After lowering, the affine index expressions in loads/stores become explicit `arith` computations on `index` values.

---

## 140.3 The `scf` Dialect (Structured Control Flow)

### 140.3.1 Design Philosophy

The `scf` dialect ([`mlir/include/mlir/Dialect/SCF/IR/SCFOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/SCF/IR/SCFOps.td)) provides structured control flow at a higher abstraction level than `cf` but lower than `affine`: loops can have non-affine bounds and strides, and conditionals can branch on arbitrary `i1` values. The defining feature of `scf` is its region-based structure — loops and conditionals use nested regions rather than explicit branch targets, which:

1. Makes the control flow immediately visible from the op structure
2. Enables region-based transformations (region inlining, fusion)
3. Supports loop-carried values via `iter_args` and `scf.yield`
4. Preserves domination structure for SSA values

### 140.3.2 `scf.for`

```mlir
// Basic counted loop
scf.for %i = %lb to %ub step %step {
  // body
  // implicit scf.yield at end
}

// Loop with loop-carried values (SSA accumulators)
%final_sum, %final_prod = scf.for %i = %c0 to %n step %c1
    iter_args(%sum = %init_sum, %prod = %init_prod) -> (f32, f32) {
  %elem = memref.load %arr[%i] : memref<?xf32>
  %new_sum  = arith.addf %sum, %elem : f32
  %new_prod = arith.mulf %prod, %elem : f32
  scf.yield %new_sum, %new_prod : f32, f32
}
// %final_sum and %final_prod have the values from the last iteration
```

The `iter_args` mechanism is central to SSA-clean loop representation. Without it, mutable state would need to be modeled with memrefs. With it, the loop is purely functional: each iteration receives values and produces new values for the next iteration. After lowering to `cf`, these become block arguments and back-edge values.

Loop bounds (`%lb`, `%ub`, `%step`) must be of `index` type and can be arbitrary `index` values — not restricted to affine expressions.

### 140.3.3 `scf.while`

`scf.while` models do-while and while loops with arbitrary condition:

```mlir
// While loop: iterate while condition is true
%result = scf.while (%arg = %init) : (f32) -> f32 {
  // "before" region: compute condition and pass values to "after"
  %cond = arith.cmpf olt, %arg, %threshold : f32
  scf.condition(%cond) %arg : f32
} do {
  ^bb0(%val: f32):
  // "after" region: loop body
  %next = arith.mulf %val, %factor : f32
  scf.yield %next : f32
}
```

The two-region structure separates the condition check ("before") from the body ("after"). The `scf.condition` op passes values to the "after" region when the condition is true, and terminates the loop when false.

For do-while semantics, execute the body once before checking the condition by computing the body in the "before" region and putting the check at the `scf.condition` boundary.

### 140.3.4 `scf.if`

```mlir
// Simple conditional (no results)
scf.if %cond {
  // then branch
}

// Conditional with else (produces values)
%r1, %r2 = scf.if %cond -> (f32, i32) {
  %a = arith.constant 1.0 : f32
  %b = arith.constant 42 : i32
  scf.yield %a, %b : f32, i32
} else {
  %c = arith.constant 0.0 : f32
  %d = arith.constant 0 : i32
  scf.yield %c, %d : f32, i32
}
```

When `scf.if` produces results, both then and else branches must yield values of the matching types. This is MLIR's equivalent of a ternary expression for multiple values. After lowering to `cf`:

```mlir
// Equivalent cf form:
cf.cond_br %cond, ^then, ^else
^then:
  // ...
  cf.br ^merge(%a : f32, %b : i32)
^else:
  // ...
  cf.br ^merge(%c : f32, %d : i32)
^merge(%r1: f32, %r2: i32):
  // %r1 and %r2 are the phi-joined results
```

### 140.3.5 `scf.parallel` and `scf.reduce`

```mlir
// Parallel loop (iterations are independent)
scf.parallel (%i, %j) = (%c0, %c0) to (%n, %m) step (%c1, %c1) {
  // body executes for all (i, j) pairs in any order
  scf.reduce
}

// Parallel with reduction
%sum = scf.parallel (%i) = (%c0) to (%n) step (%c1)
    init(%zero) : f32 {
  %elem = memref.load %arr[%i] : memref<?xf32>
  scf.reduce(%elem : f32) {
    ^bb0(%lhs: f32, %rhs: f32):
      %add = arith.addf %lhs, %rhs : f32
      scf.reduce.return %add : f32
  }
} : f32
```

`scf.parallel` asserts that all iterations are independent and can execute concurrently. The `scf.reduce` block defines how partial results are combined — this is needed for architectures where each thread computes a partial sum that must be accumulated.

### 140.3.6 `scf.forall`

`scf.forall` is the modern replacement for `scf.foreach_thread` (which is deprecated):

```mlir
// Multi-dimensional parallel loop with shared output
%result = scf.forall (%i, %j) in (4, 4) shared_outs(%out = %init) -> tensor<4x4xf32> {
  // Compute tile [i, j]
  %tile = some_computation(%i, %j) : tensor<1x1xf32>

  // Insert tile into shared output (atomically)
  scf.forall.in_parallel {
    tensor.parallel_insert_slice %tile into %out[%i, %j][1, 1][1, 1]
        : tensor<1x1xf32> into tensor<4x4xf32>
  }
}
```

`scf.forall` is the high-level expression of tiled parallelism. It is the output of tiling a `linalg` op with `scf.forall` as the containing loop type. The `shared_outs` mechanism allows multiple threads to contribute to non-overlapping tiles of a shared output tensor. The `scf.forall.in_parallel` region marks the section where the partial results are combined.

`scf.forall` maps naturally to GPU thread blocks:
```bash
# Map scf.forall to GPU thread/block hierarchy
mlir-opt --gpu-map-parallel-loops input.mlir
```

### 140.3.7 `scf.index_switch`

```mlir
%result = scf.index_switch %idx : index -> f32
case 0 {
  %v = arith.constant 1.0 : f32
  scf.yield %v : f32
}
case 1 {
  %v = arith.constant 2.0 : f32
  scf.yield %v : f32
}
default {
  %v = arith.constant 0.0 : f32
  scf.yield %v : f32
}
```

This provides value-based switching on `index` type, which is more restricted than `cf.switch` (which takes arbitrary integers) but produces cleaner, region-based IR.

---

## 140.4 SCF Passes and Transformations

### 140.4.1 Loop Transformations on SCF

```bash
# Unroll scf.for loops
mlir-opt --scf-for-loop-unroll-and-jam input.mlir
mlir-opt --scf-for-to-while input.mlir  # convert for to while

# Outline the body of a parallel loop into a function
mlir-opt --scf-parallel-loop-fusion input.mlir

# Convert parallel loops to GPU launch
mlir-opt --convert-parallel-loops-to-gpu input.mlir
```

### 140.4.2 `--convert-scf-to-cf`

The primary lowering pass:

```bash
mlir-opt --convert-scf-to-cf input.mlir
```

Translations performed:

| `scf` op | `cf` equivalent |
|---|---|
| `scf.for` | loop header block + body block + exit block, with `cf.cond_br` and `cf.br` |
| `scf.while` | before-block + after-block + exit-block |
| `scf.if` | then-block + (else-block) + merge-block |
| `scf.parallel` | nested `scf.for` (parallel semantics lost, must parallelize earlier) |
| `scf.yield` | `cf.br` to the successor block |
| `scf.reduce` | reduction in the merge block |

After `--convert-scf-to-cf`, all control flow is expressed as basic blocks and branches. The structured regions are gone, replaced by explicit block arguments.

### 140.4.3 Loop-Carried Value Pattern in Detail

Consider the transformation of `scf.for` with `iter_args`:

**Before `--convert-scf-to-cf`:**
```mlir
%result = scf.for %i = %c0 to %n step %c1
    iter_args(%acc = %init) -> f32 {
  %elem = memref.load %arr[%i] : memref<?xf32>
  %new = arith.addf %acc, %elem : f32
  scf.yield %new : f32
}
```

**After `--convert-scf-to-cf`:**
```mlir
cf.br ^header(%c0 : index, %init : f32)

^header(%iv: index, %acc: f32):
  %cond = index.cmp slt(%iv, %n)
  cf.cond_br %cond, ^body(%iv, %acc), ^exit(%acc)

^body(%iv2: index, %acc2: f32):
  %elem = memref.load %arr[%iv2] : memref<?xf32>
  %new_acc = arith.addf %acc2, %elem : f32
  %next_iv = index.add %iv2, %c1
  cf.br ^header(%next_iv : index, %new_acc : f32)

^exit(%result: f32):
  // %result holds the final value
```

The `iter_args` pair (`%acc` = `%init`) becomes the initial block argument on the first `cf.br` to `^header`. Each back-edge carries the updated values. The exit block receives the final values.

---

## 140.5 Affine vs SCF Decision Guide

Choosing between `affine` and `scf` depends on what analysis/transformations are needed:

| Feature | `affine` | `scf` |
|---|---|---|
| Loop bounds | Must be affine expressions | Arbitrary `index` values |
| Memory access | Must be affine index maps | Arbitrary index computations |
| Dependence analysis | Static, exact (polyhedral) | Not available |
| Auto-parallelization | Yes (`--affine-parallelize`) | No (manual only) |
| Loop tiling | Built-in (`--affine-loop-tile`) | Via Transform dialect |
| Loop fusion | Built-in (`--affine-loop-fusion`) | Manual |
| Loop-carried values | Yes (`iter_args`) | Yes (`iter_args`) |
| GPU mapping | Via `affine.parallel` | Via `scf.forall` |
| Conditional | `affine.if` (affine constraints) | `scf.if` (arbitrary `i1`) |
| Lowering | `--lower-affine` → `scf` | `--convert-scf-to-cf` → `cf` |

In practice: use `affine` when the loop nest comes from a polyhedral source (Polly, Pluto, affine scheduling) or when you need MLIR's affine analysis passes. Use `scf` when loop bounds are dynamic or when the computation does not fit affine constraints. `linalg` ops tile into `scf.for` or `scf.forall` by default; `affine.for` is typically produced by `--convert-linalg-to-affine-loops`.

---

## Chapter 140 Summary

- The `affine` dialect represents loop nests and memory accesses as polyhedral (affine) expressions, enabling static dependence analysis and powerful loop transformations. `affine.for` requires affine loop bounds; `affine.load`/`affine.store` require affine index maps.
- `affine.if` provides conditionals based on integer linear constraints (conjunction of affine inequalities), corresponding to polyhedral guards. `affine.apply`, `affine.min`, `affine.max` compute affine expressions to produce `index` values.
- `affine.parallel` marks perfectly nested parallel loops with optional reductions, enabling OpenMP or GPU mapping.
- The affine analysis framework provides Farkas/Omega-based dependence analysis for pairs of `affine.load`/`affine.store` accesses, powering passes: `--affine-loop-tile`, `--affine-loop-fusion`, `--affine-parallelize`, `--affine-vectorize`, `--affine-loop-unroll`.
- `--lower-affine` converts the affine dialect to `scf` + `memref`, preserving program semantics while losing analyzability.
- The `scf` dialect provides structured loops (`scf.for`, `scf.while`) and conditionals (`scf.if`) at a higher level than `cf` but without polyhedral restrictions. Loop-carried values via `iter_args` maintain SSA form across loop iterations.
- `scf.parallel` asserts iteration independence for explicit parallelism; `scf.forall` is the modern form supporting shared outputs via `tensor.parallel_insert_slice`, mapping naturally to GPU thread/block hierarchies.
- `--convert-scf-to-cf` lowers structured SCF to basic blocks and branches, converting `iter_args` to block arguments and `scf.yield` to `cf.br` instructions. This is the mandatory step before `--convert-cf-to-llvm`.


---

@copyright jreuben11
