# Chapter 64 — Vectorization Deep Dive

*Part X — Analysis and the Middle-End*

Vectorization transforms scalar loops and instruction groups into SIMD operations, exploiting data-level parallelism on hardware that supports SSE, AVX-512, NEON, SVE, or RISC-V Vector. LLVM provides two vectorization passes — the Loop Vectorizer and the SLP (Superword-Level Parallelism) Vectorizer — each targeting different program structures. This chapter covers the loop vectorizer's four-phase design, VPlan's role, the SLP vectorizer's bottom-up tree construction, cost modeling, scalable vectorization, and predication for loop tails.

## Table of Contents

- [64.1 The Loop Vectorizer](#641-the-loop-vectorizer)
  - [64.1.1 Four-Phase Architecture](#6411-four-phase-architecture)
  - [64.1.2 Loop Vectorizer Hints](#6412-loop-vectorizer-hints)
- [64.2 VPlan: The Vectorization Plan](#642-vplan-the-vectorization-plan)
  - [64.2.1 VPlan Structure](#6421-vplan-structure)
  - [64.2.2 VPlan and VF Selection](#6422-vplan-and-vf-selection)
- [64.3 Scalable Vectorization (SVE, RVV)](#643-scalable-vectorization-sve-rvv)
  - [64.3.1 EVL (Explicit Vector Length) Intrinsics](#6431-evl-explicit-vector-length-intrinsics)
- [64.4 Predication and Tail Folding](#644-predication-and-tail-folding)
  - [64.4.1 Scalar Remainder Loop](#6441-scalar-remainder-loop)
  - [64.4.2 Tail Folding (Predicated Vectorization)](#6442-tail-folding-predicated-vectorization)
- [64.5 Reductions and First-Order Recurrences](#645-reductions-and-first-order-recurrences)
- [64.6 SLP Vectorizer](#646-slp-vectorizer)
  - [64.6.1 Bottom-Up Tree Construction](#6461-bottom-up-tree-construction)
  - [64.6.2 Differences from Loop Vectorization](#6462-differences-from-loop-vectorization)
- [64.7 TargetTransformInfo: The Cost Model](#647-targettransforminfo-the-cost-model)
- [Chapter Summary](#chapter-summary)

---

## 64.1 The Loop Vectorizer

`LoopVectorizePass` (in `llvm/Transforms/Vectorize/LoopVectorize.h`) widens innermost loops by a vectorization factor (VF), transforming scalar iterations into vector iterations. A loop with VF=4 runs four iterations of the scalar loop per vector iteration.

### 64.1.1 Four-Phase Architecture

**Phase 1: Legality checking** (`LoopVectorizationLegality`)

Determines whether the loop *can* be vectorized. A loop is legal if:
- All memory accesses have known stride (no indirect indexing that prevents analysis).
- All loop-carried dependences have a distance ≥ VF (so iterations don't race).
- All instructions have a vector equivalent (no inline assembly, no unvectorizable intrinsics).
- The loop is in simplify form with a computable trip count.

```cpp
// Legality check within vectorizer internals
LoopVectorizationLegality LEL(TheLoop, PSE, DT, TTI, TLI, AA, F, &LAI, &LVL);
if (!LEL.canVectorize(EnableVPlanNativePath)) return false;
```

**Phase 2: Cost modeling** (`LoopVectorizationCostModel`)

For each candidate VF (powers of 2: 2, 4, 8, 16, …), computes the vector loop's expected cost using `TargetTransformInfo` (TTI):

```cpp
// Get instruction cost for vectorized add
InstructionCost vecCost = TTI.getArithmeticInstrCost(
    Instruction::Add,
    VectorType::get(Type::getInt32Ty(Ctx), VF),
    TTI::TCK_RecipThroughput);
```

The cost model computes the vector body's throughput in terms of cycles-per-iteration, scales by VF (since one vector iteration replaces VF scalar iterations), and selects the VF that minimizes cycles per element. If no VF is profitable (the overhead of vectorization exceeds the benefit), the loop is not vectorized.

**Phase 3: Code generation** (`InnerLoopVectorizer`)

Generates the vectorized loop. Key steps:
1. Widen scalars to vectors: `%x → <4 x i32>`.
2. Widen memory accesses: `load i32` → `load <4 x i32>` (strided loads become `llvm.masked.gather`).
3. Widen phi nodes: the loop IV becomes `<4 x i32> {0,1,2,3}` + `4*i`.
4. Handle reductions: `%sum = %sum + %elem` becomes `%vsum = add <4 x float> %vsum, %velems`, followed by a vector reduction `llvm.vector.reduce.fadd`.
5. Generate a scalar remainder loop for the trip count mod VF iterations that don't fill a complete vector.

**Phase 4: VPlan generation** (described in §64.2)

### 64.1.2 Loop Vectorizer Hints

Developers can provide vectorization hints via loop metadata:

```llvm
; Force vectorization with VF=8
br ..., !llvm.loop !0
!0 = !{!0, !1}
!1 = !{!"llvm.loop.vectorize.width", i32 8}

; Disable vectorization
!2 = !{!"llvm.loop.vectorize.enable", i1 false}

; Hint: assume no aliasing (enables more aggressive vectorization)
!3 = !{!"llvm.loop.parallel_accesses", !4}
```

From Clang source, the `#pragma clang loop vectorize(enable)` annotation generates these metadata hints.

## 64.2 VPlan: The Vectorization Plan

VPlan (Vectorization Plan) is a graph-based IR for representing the vectorized loop before code generation. It was introduced to replace the ad-hoc "widened" IR that the vectorizer generated directly, enabling more flexible VF selection and transformation.

### 64.2.1 VPlan Structure

A `VPlan` is a DAG of `VPBasicBlock` nodes containing `VPRecipeBase` objects. Each recipe corresponds to one or more IR instructions in the vectorized output:

- `VPWidenRecipe`: widen a scalar instruction to operate on vectors.
- `VPWidenMemoryInstructionRecipe`: widen a load/store.
- `VPReductionRecipe`: a reduction over a vector.
- `VPBlendRecipe`: a vector blend (for predicated lanes).
- `VPBranchOnMaskRecipe`: conditional execution based on a mask.
- `VPCanonicalIVPHIRecipe`: the canonical loop IV in vector form.

### 64.2.2 VPlan and VF Selection

By constructing VPlan early (before code generation), the vectorizer can instantiate the plan for multiple candidate VFs without re-running legality and cost analysis. The cheapest plan is selected and generated:

```
// Pseudocode: VPlan-based VF selection
VPlan *VP = buildVPlan(Loop, VF_candidates);
auto [bestVF, bestIC] = selectVF(VP, TTI);
generateCode(VP, bestVF);
```

The VPlan-native path (enabled with `-vplan-native-path`) uses VPlan for all vectorization, including outer-loop vectorization (a longer-term goal).

## 64.3 Scalable Vectorization (SVE, RVV)

Traditional vectorization uses fixed-width SIMD (128-bit SSE, 256-bit AVX2). Scalable vectorization targets architectures with variable-width SIMD: ARM SVE (Scalable Vector Extension) and RISC-V Vector (RVV), where the vector length (VL) is determined at runtime.

LLVM represents scalable vectors as `<vscale x N x T>`: a vector whose actual width is `vscale * N * sizeof(T)` bits, where `vscale` is a runtime-determined multiplier:

```llvm
; scalable vector of i32 (actual width: vscale*4 ints)
%sv = load <vscale x 4 x i32>, ptr %p
%v1 = add  <vscale x 4 x i32> %sv, %sv
```

The loop vectorizer generates scalable-vector code when `-scalable-vectorization=preferred` or when TTI indicates the target prefers scalable vectors:

```bash
opt -passes='loop-vectorize' -scalable-vectorization=preferred -S input.ll
```

### 64.3.1 EVL (Explicit Vector Length) Intrinsics

For RVV, LLVM uses EVL (Explicit Vector Length) intrinsics to express predicated operations where the tail lanes are not computed:

```llvm
; Process %vl elements (last iteration has fewer than vscale*4)
%res = call <vscale x 4 x i32> @llvm.vp.add.nxv4i32(
    <vscale x 4 x i32> %a,
    <vscale x 4 x i32> %b,
    <vscale x 4 x i32 i1> %mask,  ; active lanes mask
    i32 %vl)                        ; active element count
```

EVL intrinsics allow the loop to process a complete LMUL (length multiplier) group on every iteration except possibly the last, eliminating the need for a scalar remainder loop.

## 64.4 Predication and Tail Folding

When the loop trip count is not known to be a multiple of the VF at compile time, the vectorizer must handle the remaining iterations (the "tail"). There are two strategies:

### 64.4.1 Scalar Remainder Loop

The classic approach: vectorize all complete vector iterations, then run a scalar loop for the remaining 0 to VF-1 iterations. Code size is doubled (two loop versions).

### 64.4.2 Tail Folding (Predicated Vectorization)

Tail folding runs the same vectorized loop for all iterations, using a mask to disable inactive lanes on the last (possibly partial) iteration:

```llvm
; Last iteration: only %vl lanes are active
%remaining = sub i64 %N, %i   ; remaining elements
%vl = umin i64 %remaining, %VF ; actual vector width for this iteration

; Build mask: first %vl lanes are 1
%mask = call <4 x i1> @llvm.get.active.lane.mask.v4i1(i64 %i, i64 %N)

; Masked load: only load active lanes
%v = call <4 x i32> @llvm.masked.load.v4i32.p0(
    ptr %p,
    i32 4,          ; alignment
    <4 x i1> %mask,
    <4 x i32> undef)

; Masked store
call void @llvm.masked.store.v4i32.p0(
    <4 x i32> %result,
    ptr %p,
    i32 4,
    <4 x i1> %mask)
```

Tail folding reduces code size (one loop version) at the cost of mask computation overhead. On hardware with efficient predication (SVE, RVV, AVX-512 mask registers), this is often a net win.

```bash
opt -passes='loop-vectorize' -prefer-predicate-over-epilog=1 -S input.ll
```

## 64.5 Reductions and First-Order Recurrences

Reductions accumulate a loop result into a scalar:

```
; Loop: sum += A[i]
; Vectorized: %vsum = fadd <4 x float> %vsum, %vA
; After loop: %sum = call float @llvm.vector.reduce.fadd(float 0.0, <4 x float> %vsum)
```

LLVM supports reductions over `add`, `mul`, `and`, `or`, `xor`, `fmul`, `fadd`, `smin`, `smax`, `umin`, `umax`, and `fmin`/`fmax`. The `llvm.vector.reduce.*` intrinsics implement the reduction tree over the final vector accumulator.

**First-order recurrences** (non-reduction loop-carried dependences of the form `x_i = f(x_{i-1})`) are vectorized using a rotate-and-blend pattern:

```
; Source: a[i] = a[i-1] * factor
; (element i depends on element i-1 — this is a first-order recurrence)

; Vectorized: compute [a[i-4], a[i-3], a[i-2], a[i-1]] = prev_vector
;             compute [a[i], a[i+1], a[i+2], a[i+3]] = current_vector
; Blend: insert a[i-1] from prev_vector into current_vector
```

## 64.6 SLP Vectorizer

The SLP (Superword-Level Parallelism) Vectorizer (`SLPVectorizerPass` in `llvm/Transforms/Vectorize/SLPVectorizer.h`) packs independent scalar operations that appear *outside* loops into SIMD operations. Where the loop vectorizer iterates over loop iterations, SLP exploits independent instruction-level parallelism within a basic block.

### 64.6.1 Bottom-Up Tree Construction

SLP starts from a "seed" — typically a store or a reduction operation — and builds a tree upward by finding independent isomorphic operations that can be packed:

```
; Before SLP:
%a0 = load i32, ptr %p0     ; p0 = base + 0
%a1 = load i32, ptr %p1     ; p1 = base + 4
%a2 = load i32, ptr %p2     ; p2 = base + 8
%a3 = load i32, ptr %p3     ; p3 = base + 12
%b0 = mul i32 %a0, %k
%b1 = mul i32 %a1, %k
%b2 = mul i32 %a2, %k
%b3 = mul i32 %a3, %k
store i32 %b0, ptr %q0
store i32 %b1, ptr %q1
store i32 %b2, ptr %q2
store i32 %b3, ptr %q3

; After SLP (VF=4):
%va = load <4 x i32>, ptr %p0
%vk = insertelement <4 x i32> undef, i32 %k, i32 0
%vk = shufflevector %vk, undef, <0,0,0,0>
%vb = mul <4 x i32> %va, %vk
store <4 x i32> %vb, ptr %q0
```

SLP uses `TargetTransformInfo` to cost each candidate vectorization; if the vector tree is cheaper than the scalar tree, it is emitted.

### 64.6.2 Differences from Loop Vectorization

| | Loop Vectorizer | SLP Vectorizer |
|--|-----------------|----------------|
| Target | Loops | Basic blocks (without loops) |
| Parallelism | Across loop iterations | Across independent operations |
| Input | SCF loops | Isomorphic instruction trees |
| Width selection | VF based on trip count | Pack size based on instruction count |
| Reductions | Yes (first-class) | Yes (horizontal reductions) |

SLP and loop vectorization are complementary: `SLPVectorizerPass` runs *after* `LoopVectorizePass` and can pack operations in the vectorized loop's pre/post-amble, as well as non-loop code.

## 64.7 TargetTransformInfo: The Cost Model

`TargetTransformInfo` (TTI, from `llvm/Analysis/TargetTransformInfo.h`) provides the cost model that both vectorizers query. It is the interface between target-independent optimization and target-specific knowledge.

Key TTI queries for vectorization:

```cpp
// How many registers of type T are available?
unsigned numRegs = TTI.getNumberOfRegisters(/*vector=*/true);

// Can we efficiently use vectors of this width?
TargetTransformInfo::PopcntSupportKind support =
    TTI.getPopcntSupport(128); // 128-bit vector

// Cost to vectorize a load at this stride
InstructionCost cost = TTI.getMemoryOpCost(
    Instruction::Load,
    VectorType::get(i32, 8),
    Align(4),
    /*AddressSpace=*/0);

// Is it profitable to unroll this loop?
TTI.getUnrollPreferences(TheLoop, SE, /*UP=*/...);

// Cost for gather/scatter
InstructionCost gatherCost = TTI.getGatherScatterOpCost(
    Instruction::Load, VectorType::get(i32, 4), ptr, /*variableMask=*/false, ...);
```

Each target implements `TargetTransformInfo::Concept` (using the same concept-based polymorphism as the pass manager), providing target-specific costs that guide the vectorizer's VF selection.

## 64.8 The Sandbox Vectorizer

The Loop Vectorizer and SLP Vectorizer described in §64.1–§64.6 are mature, production-quality passes. Their complexity, however, makes it difficult to prototype new vectorization algorithms: a one-line heuristic change in VPlan can introduce regressions across hundreds of test cases, discouraging experimentation. The **Sandbox Vectorizer** (introduced in LLVM 19, present as an experimental pass in LLVM 22) addresses this by providing an isolated IR sandbox where vectorization transformations are tried speculatively before being committed.

### Motivation

VPlan (§64.2) represents the vectorized loop as a recipe graph, but its tight coupling to the main IR and to the Loop Vectorizer's cost model makes it hard to test new ideas in isolation. Extending the SLP Vectorizer requires understanding its interleaved legality/cost/codegen logic. A mechanism is needed to:

1. Apply a speculative sequence of IR mutations.
2. Check whether the result is profitable (or even legal).
3. Either **commit** the mutations to the real IR or **discard** them completely — without writing a manual undo trail.

### Architecture: SandboxIR

The Sandbox Vectorizer is built on top of **SandboxIR** (in [`llvm/include/llvm/SandboxIR/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/SandboxIR/) and `llvm/lib/SandboxIR/`), a thin wrapper layer over standard LLVM IR that adds **transaction semantics**:

- `sandboxir::Context`: owns a mapping from `llvm::Value *` to `sandboxir::Value *`. All SandboxIR objects are thin wrappers; the underlying LLVM IR objects are authoritative.
- `sandboxir::Function`, `sandboxir::BasicBlock`, `sandboxir::Instruction`: mirror the standard IR hierarchy but route all mutation operations (instruction insertion, deletion, operand replacement) through a **change log**.
- `sandboxir::Tracker`: records every mutation applied in the sandbox. Calling `Tracker::revert()` replays the change log in reverse, restoring the LLVM IR to its pre-sandbox state. Calling `Tracker::accept()` clears the log, making the mutations permanent.

```cpp
// Pseudocode illustrating SandboxIR transaction semantics
sandboxir::Context Ctx(LLVMCtx);
sandboxir::Function *F = Ctx.createFunction(LLVMFunc);

auto &Tracker = Ctx.getTracker();
Tracker.save();  // Begin transaction

// Try a vectorization transformation on SandboxIR values
tryVectorize(F, Ctx);

if (isProfitable(F, TTI)) {
  Tracker.accept();  // Commit: mutations are now in the real LLVM IR
} else {
  Tracker.revert();  // Rollback: LLVM IR is restored exactly
}
```

Key SandboxIR classes:

| Class | Role |
|-------|------|
| `sandboxir::Context` | Mapping + Tracker; root of the sandbox |
| `sandboxir::Function` | Wraps `llvm::Function` with mutation tracking |
| `sandboxir::BasicBlock` | Wraps `llvm::BasicBlock`; instruction iteration |
| `sandboxir::Instruction` | Wraps `llvm::Instruction`; tracked insert/erase/replace |
| `sandboxir::Tracker` | Records changes; supports `save()`, `revert()`, `accept()` |

### Relationship to Loop Vectorizer and SLP Vectorizer

The Sandbox Vectorizer is a **separate experimental pass**, not a replacement for either the Loop Vectorizer or SLP Vectorizer in LLVM 22:

- It does not share VPlan's recipe graph.
- It does not use the Loop Vectorizer's cost model or legality checks.
- It is intended as a **research vehicle** for prototyping new bottom-up or top-down vectorization strategies.

The long-term intent is that successful algorithms proven in the sandbox can be promoted to production passes. The sandbox lowers the risk of prototype code escaping into the main pipeline.

### How to Enable

The Sandbox Vectorizer is registered under the name `sandbox-vectorizer`:

```bash
# Run the experimental Sandbox Vectorizer:
opt -passes='sandbox-vectorizer' -S input.ll

# Combine with standard passes (sandbox-vectorizer is additive):
opt -passes='function(sandbox-vectorizer,instcombine,simplifycfg)' -S input.ll
```

The pass is **not** included in the default `-O2`/`-O3` pipeline in LLVM 22. It must be explicitly requested via `-passes=` or a custom `PassBuilder` extension.

### Use Cases

- **Rapid heuristic prototyping:** Test a new pack-selection heuristic for SLP-style vectorization without touching the production SLP Vectorizer.
- **Research experimentation:** Implement a new loop vectorization algorithm (e.g., polyhedral-informed vectorization) and measure its effect on a benchmark suite without risking regressions in the default pipeline.
- **Teaching and tooling:** The SandboxIR layer provides a clean API for writing IR transformations that can be rolled back, useful outside of vectorization (e.g., for try-and-revert loop transformations).

### Current Limitations

- **Subset of IR supported.** SandboxIR does not yet wrap all LLVM IR constructs; complex instructions (e.g., `callbr`, certain intrinsics) may fall back to treating the instruction as opaque.
- **No VPlan integration.** The sandbox operates on plain IR; it cannot reuse VPlan's vectorization cost model or predication machinery.
- **No integration with the default pipeline.** The pass is opt-in; it must be explicitly inserted via `-passes=` or a programmatic `PassBuilder` extension.
- **Performance regression risk on untested inputs.** As an experimental pass, the Sandbox Vectorizer may produce suboptimal or incorrect code for patterns not yet covered by its test suite.

For production vectorization in LLVM 22, the Loop Vectorizer (§64.1) and SLP Vectorizer (§64.6) remain the authoritative implementations. See [Chapter 64 §64.2 — VPlan](#642-vplan-the-vectorization-plan) for the production vectorization plan IR and [§64.6 — SLP Vectorizer](#646-slp-vectorizer) for the production superword-level parallelism pass.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **VPlan native path promotion to default**: The VPlan-native vectorization path (`-vplan-native-path`) is on track to replace the legacy code-generation path in LLVM 22/23; active RFC and patch series on discourse.llvm.org (see "RFC: Enabling VPlan-based vectorization by default") aim to land this in the LLVM 23 release cycle, enabling outer-loop vectorization as a first-class feature.
- **EVL tail-loop folding for RVV**: Upstream patches (D157054 and successors) extending EVL-based tail folding to cover reductions and first-order recurrences on RISC-V Vector — currently EVL tail folding handles simple loads/stores but not all reduction patterns supported by the scalar remainder path.
- **Sandbox Vectorizer region support**: Active development to extend SandboxIR's `Tracker` to cover `callbr` and complex intrinsics (currently treated as opaque), tracked in the llvm-project GitHub issue tracker under the "SandboxIR coverage" milestone, targeting LLVM 23.
- **SLP vectorizer scalable-vector support**: Work in progress (D144448 follow-ons) to enable SLP packing of `<vscale x N x T>` values for SVE/RVV targets, currently blocked by the lack of a cost model for variable-width pack operations in TTI.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Outer-loop vectorization via VPlan**: Once the VPlan native path is the default, outer-loop vectorization (loop nests whose inner loop is vectorized across the outer-loop dimension) becomes achievable; the VPlan recipe graph already models the control-flow structure needed, and RFC discussions on discourse.llvm.org outline the remaining legality and cost-model extensions required.
- **Polyhedral-informed VF selection**: Integration of polyhedral dependence information (from Polly or a lightweight polyhedral analysis pass) into `LoopVectorizationLegality` to handle loop nests with parametric dependence distances that the current distance-based legality check conservatively rejects.
- **Unified cost model for mixed fixed/scalable vectorization**: `TargetTransformInfo` currently treats fixed-width and scalable-vector costs separately; a unified cost API supporting `<vscale x N x T>` pack-cost queries would allow the Loop Vectorizer and SLP Vectorizer to compare fixed-VF and scalable-VF candidates in a single cost sweep, enabling better VF selection on SVE2 and RVV 1.0 targets.
- **Auto-vectorization of histogram and scatter-accumulate patterns**: These patterns (common in HPC and ML workloads) require conflict-detection idioms (`llvm.experimental.vector.histogram.*` intrinsics, RFC posted mid-2025) not yet supported by the Loop Vectorizer's legality checker; mid-term goal is full legality and cost-model support, enabling AVX-512 VPCONFLICT and SVE2 HISTCNT code generation.

### 5-Year Horizon (Long-Term, by ~2031)

- **Machine-learning–guided VF and unroll-factor selection**: Replacing or augmenting TTI's static throughput model with a learned cost model (following the approach of MLGO for inlining, extended to vectorization) that predicts actual hardware performance from IR features, adapting VF selection to micro-architectural details not captured by the reciprocal-throughput abstraction.
- **Cross-loop vectorization and supernode SLP**: Extending SLP beyond basic-block boundaries to pack operations across adjacent loop iterations or across function calls in a polyhedral supernode, requiring integration with the inter-procedural dependence analysis infrastructure being developed in the LLVM middle-end.
- **Auto-vectorization for emerging SIMD ISAs**: Full vectorization support for Arm SME (Scalable Matrix Extension) streaming-mode SVE and for RISC-V Zvfbfmin/Zvfhmin half-precision vector extensions, including cost models reflecting the distinct throughput characteristics of matrix-tile and half-precision arithmetic units expected in 2028–2031 hardware generations.

---

## Chapter Summary

- The loop vectorizer has four phases: legality checking (`LoopVectorizationLegality`), cost modeling (`LoopVectorizationCostModel`), code generation (`InnerLoopVectorizer`), and VPlan generation.
- `VPlan` is a graph IR for the vectorized loop, enabling VF selection by instantiating the same plan at multiple widths without re-running legality.
- Scalable vectorization targets ARM SVE and RISC-V RVV via `<vscale x N x T>` types and EVL (explicit vector length) intrinsics.
- Tail folding uses `@llvm.masked.load`/`@llvm.masked.store` to predicate the last vector iteration, eliminating the scalar remainder loop.
- Reductions use `@llvm.vector.reduce.*` intrinsics after the loop; first-order recurrences use a rotate-and-blend pattern.
- The SLP vectorizer packs isomorphic scalar operations within basic blocks into SIMD operations using a bottom-up tree construction from stores/reductions.
- `TargetTransformInfo` provides the cost model queried by both vectorizers; each backend implements it to express hardware throughput and register constraints.
- The Sandbox Vectorizer (`sandbox-vectorizer`) is an experimental pass built on **SandboxIR** — a transaction-semantic wrapper over LLVM IR that supports speculative mutations with `Tracker::revert()` rollback; it is not part of the default pipeline in LLVM 22 but provides a low-risk environment for prototyping new vectorization algorithms.


---

@copyright jreuben11
