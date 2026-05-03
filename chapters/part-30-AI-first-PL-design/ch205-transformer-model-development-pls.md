# Chapter 205 — Transformer Model Development PLs: From Python Frameworks to AI-First Design

*Part XXX — AI-First Programming Language Design*

The coding-agent properties defined in Chapter 203 — formal semantics, dependent types, algebraic effects, homoiconicity — apply to programs written by AI. A unified AI-first PL must also serve as the language in which AI *models themselves* are developed. This is not a separate domain: named tensor dimensions are refinement types, automatic differentiation is a type-level algebraic transform, stochasticity is an algebraic effect, parallelism strategies are type annotations. The same type-theoretic foundation that governs AI-generated programs governs the programs those agents run on. This chapter catalogs ten design considerations specific to transformer model development, surveys the languages and frameworks that implement identifiable subsets of those properties, examines StableHLO/MLIR as the universal compilation target, and reviews the emerging body of LLM-driven kernel generation research. The connection to existing LLVM/MLIR infrastructure is direct: Chapter 179 covers the full ML compilation stack (torch.export → StableHLO → MLIR → hardware), Chapter 163 covers Triton as the current reference kernel DSL, and Chapter 180 covers AI-guided compilation (Ansor, AutoTVM, MLGO) as the precursor to the LLM-driven kernel generation systems surveyed in §205.6.

---

## 205.1 Why Python Frameworks Miss the Target

Transformer model development is dominated by frameworks embedded in Python — a language designed for human data scientists. Each framework has made pragmatic choices that are correct for human authors and wrong for AI authors.

**PyTorch.** Extraordinary ergonomics, degraded by legacy autograd API, imperative `nn.Module` design, and a decade of accumulated cruft. The dynamic computation graph is human-friendly — allowing arbitrary Python control flow — but compiler-hostile. Shape inference requires executing code, not analysing types. Effect tracking requires running hooks, not reading type signatures. An AI generating PyTorch code generates text that a Python interpreter runs; the type system is a dynamic checking afterthought.

**TensorFlow / Swift for TensorFlow.** Swift for TF (2018–2021) had the theoretically correct instincts: a systems language with first-class automatic differentiation (`@differentiable` functions), ownership semantics, and a strong type system (§205.7). It predated coding agents. Without LLMs to eliminate the human learning curve, it could not gain traction against PyTorch's ecosystem.

**JAX.** Closest to a principled design: pure functions, `jit`/`grad`/`vmap`/`pmap` as composable transforms, XLA as the compilation backend. The functional purity is genuine; the syntax is still Python. JAX had to introduce Pallas — a constrained Python kernel DSL — to give users hardware-level control, avoiding the loss of the Python user base. The constraint: Python syntax limits what the type system can express.

**Triton / CuTile.** Python DSLs for GPU kernel authoring. Constrained Python, not new PLs. Tiles and blocks are Python decorators. The language boundary (Python types, not GPU types) forces shapes and memory hierarchies into runtime conventions. Chapter 163 covers Triton's tile-based execution model in detail.

**Burn (Rust).** A deep learning framework, not a new PL — it inherits Rust's type system, which has no native dependent types. Tensor shapes are not checked at compile time. This is a fundamental limitation of the host language, not a Burn implementation choice.

**The fundamental opportunity.** If an AI is the primary author of model code, human ergonomics can be eliminated as a design constraint. A language as unfamiliar as Dex or as verbose as Lean 4 is learnable by an AI. This unlocks a completely different design space: dependent tensor types, first-class AD semantics, algebraic-effect-based stochasticity, and MLIR as a native compilation target rather than a framework backend.

---

## 205.2 Production Serving Demands

An AI-first transformer model PL does not exist in isolation — it compiles to a runtime that must serve inference at scale. The production serving layer defines concrete demands that an ideal PL must satisfy:

| Serving system | Key demands on the upstream PL |
|---|---|
| **vLLM** (PagedAttention, continuous batching) | Dynamic shape handling — sequence lengths vary per request; the type system must handle runtime shapes without 105× recompilation overhead (see §205.6, DVM) |
| **llama.cpp** | Quantisation-aware types — `Int4`, `Int8`, `BFloat16` must be first-class precision types, not casts; the PL should derive quantised model variants from the reference `Float32` model |
| **TensorRT-LLM** (NVIDIA) | Kernel fusion at the PL level — fused attention + FFN is the performance-critical path; the ideal PL's compiler derives fusion, not the programmer |
| **Triton Inference Server** | StableHLO/MLIR as the handoff IR — the PL's compilation target must be the convergence IR that all serving backends accept |
| **SGLang** | Structured generation integration — the PL should express FSM-constrained generation (Chapter 203 §203.6.3, Outlines) as a first-class type, not a wrapper library |

This table directly motivates four of the ten design considerations: dynamic shape handling (§205.3.10), device and precision as types (§205.3.4), kernel fusion as compiler-derived (§205.3.5), and StableHLO/MLIR as the target IR (§205.3.9).

---

## 205.3 Ten Design Considerations for Transformer Model Development

These extend — rather than replace — the eight AI-first PL properties from Chapter 203 §203.1. The unified AI-first PL satisfies both sets simultaneously.

### 205.3.1 Named, Typed Tensor Dimensions

The largest source of silent bugs in transformer code. `Tensor[Batch, SeqLen, DModel, Float32]` is a compile-time type, not a runtime convention. Named axes eliminate string einsum specs, `dim=` arguments, and documentation-based coding ("Tensor Considered Harmful", Harvard NLP 2019). Shape mismatches become type errors, reported by the compiler with a structured diagnostic that the generating agent can act on.

The formal type rule is in Chapter 204 §204.5 (Tensor-Intro and Matmul). The implementation requires Z3 linear arithmetic for shape constraint discharge. An AI author benefits doubly: it generates shape-correct code by construction, and receives structured type-error messages — not Python runtime traceback — when shapes mismatch.

```pseudo
-- Named tensor type: Batch, Heads, SeqLen, DModel are dimension variables
fn scaled_dot_product_attention(
    q : Tensor[Batch, Heads, SeqLen, DModel, Float32],
    k : Tensor[Batch, Heads, SeqLen, DModel, Float32],
    v : Tensor[Batch, Heads, SeqLen, DModel, Float32]
  ) : Tensor[Batch, Heads, SeqLen, DModel, Float32]
{
  let scale = 1.0 / sqrt(DModel : Float32)
  let scores = matmul(q, transpose(k, -2, -1)) * scale
  -- Compiler verifies: matmul(q[B,H,S,D], kᵀ[B,H,D,S]) : Tensor[B,H,S,S]
  -- Shape error if DModel mismatches between q and k
  let attn = softmax(scores, dim=-1)
  matmul(attn, v)
}
```

### 205.3.2 Automatic Differentiation as a Language-Level Transform

Not a library function (`torch.autograd.grad`, `jax.grad`) but a first-class type-level construct. The formal type:

```pseudo
grad : (A →<Gradient> B) → A → (B → A)
```

This is the VJP (vector-Jacobian product / reverse-mode AD) form: given a point in A, return the function mapping output cotangents to input cotangents. The `{Gradient}` algebraic effect propagates through the call graph — functions without `{Gradient}` in their effect row are statically non-differentiable; composing them with a differentiable function is a type error, not a silent bug.

The POPL 2026 paper "Nominal Semantics for First-class AD" formalises this. The denotational semantics is in Chapter 204 §204.6.2: `⟦ e : (A →<Gradient> B) ⟧ : A → (B × (B → A))`. Dex is the closest existing language to this model (§205.4.2). Stan's compiler-derived gradients (Chapter 203 §203.7.4) prove the approach is production-feasible for probabilistic models.

### 205.3.3 Parallelism Strategies as Type Annotations

Data parallelism, tensor parallelism, pipeline parallelism, and sequence parallelism are currently runtime wrappers: `DistributedDataParallel`, `DeviceMesh`, `pmap`. In an AI-first PL these are type annotations or effect qualifiers, with the compiler deriving communication patterns (all-reduce, all-gather, scatter) and verifying consistency at compile time:

```pseudo
-- Data parallel: replicate model, partition batch
fn train_step[Devices: D]
    (model: Model, batch: Tensor[Batch, SeqLen, Float32])
  : <DataParallel[D]> Loss
{
  -- Compiler derives: split batch across D devices, reduce gradients
}

-- Tensor parallel: partition weight matrices across devices
fn linear[Devices: D]
    (x    : Tensor[Batch, In, Float32],
     w    : Shard[D, Tensor[In, Out, Float32]])
  : <TensorParallel[D]> Tensor[Batch, Out, Float32]
{
  -- Compiler derives: all-gather on output
}
```

No existing production PL implements parallelism as a verified type annotation. This is a completely open design space.

### 205.3.4 Device and Precision as Types

```pseudo
type DeviceTensor[Dev: Device, Prec: Precision, Dims..., Base] =
  Tensor[Dims..., Base] at Dev with Prec
```

`Tensor[CUDA:0, BFloat16, Batch, SeqLen, DModel]` — device mismatches and precision mismatches are type errors, not runtime crashes. Mixed-precision rules are typed coercions with explicit promotion schedules. Memory tier (HBM / SRAM / registers) as a type for kernel-level programming enables the compiler to statically verify that no tensor crosses a device boundary implicitly.

Mojo partially implements this via `SIMD[DType.float32, 8]` as a first-class type. No system implements device placement as a full type parameter.

### 205.3.5 Kernel Fusion as a Compiler-Derived Property

FlashAttention-style I/O-aware tiled attention (Dao et al.) is currently hand-coded or expressed manually in Triton. In an AI-first PL, the high-level operator composition `softmax(QKᵀ/√d) · V` admits fusion automatically via compiler analysis — the programmer writes the specification; the compiler schedules the execution. Halide demonstrated this separation for image pipelines (schedule separate from algorithm); no equivalent exists for transformer kernels at production scale. This is an open design space.

XLA (the TensorFlow/JAX backend) performs implicit fusion via HLO operation fusion passes, but the fusion decisions are not user-controllable and the fusion semantics are not expressed in the source language's type system.

### 205.3.6 Stochasticity as an Algebraic Effect

Dropout, weight initialisation, stochastic depth, and speculative decoding sampling all involve randomness. Currently this threads through code as implicit global state (PyTorch's RNG) or an explicit key (JAX's `PRNGKey`). In an AI-first PL, stochasticity is an algebraic effect tracked by the type system:

```pseudo
-- Dropout: stochastic, rate p, but type-tracked
fn dropout(x: Tensor[B,D, Float32], rate: Float) : <Stochastic> Tensor[B,D, Float32]
  x.map(fn(v) -> if sample(Bernoulli(1.0 - rate)) then v / (1.0 - rate) else 0.0)

-- Training: stochastic
-- Inference: deterministic (swap the handler)
fn inference(model: Model, x: Input) : Output
  with Deterministic(seed=42) { model.forward(x) }
```

Reproducibility becomes a compiler guarantee: a function without `{Stochastic}` in its effect row is deterministic by construction — not by convention. This is PPL Paradigm 2 (Chapter 203 §203.7.2) applied to model training. The handler-swappable model means that switching from stochastic training to deterministic inference is a handler change, not a code change.

### 205.3.7 Recomputation (Activation Checkpointing) as an Annotation

Currently `torch.utils.checkpoint.checkpoint(fn, *args)` — a manual, error-prone wrapper. In an AI-first PL, activation checkpointing is a type annotation or memory-tier declaration:

```pseudo
-- @recompute instructs the compiler to recompute activations during backward
@recompute
fn transformer_block(x: Tensor[B,S,D, Float32]) : Tensor[B,S,D, Float32]
  -- compiler inserts checkpointing: stores only input, recomputes activations
```

The compiler optimises the memory/compute trade-off across the whole model, not per-layer. An AI generating a large model can annotate entire subgraphs with `@recompute` and receive a compiler-verified memory budget guarantee.

### 205.3.8 Einsum and Index Notation as First-Class Syntax

`torch.einsum("bhsd,bhnd->bhsn", q, k)` is a string-embedded DSL with no type checking. In an AI-first PL, Einstein index notation is first-class syntax, type-checked against named dimension types:

```pseudo
-- First-class einsum: compiler verifies index names against tensor types
let scores = einsum[b,h,s,d; b,h,n,d -> b,h,s,n](q, k)
-- Type error if q[b,h,s,d] has DModel ≠ k[b,h,n,d]'s last dimension
```

Dex's "higher-order dependently-typed Einstein summation" (§205.4.2) is the existence proof. The index names *are* the dimension variables from the tensor type — no separate einsum string required.

### 205.3.9 Hardware Portability via MLIR

CUDA, ROCm, TPU XLA, Intel Gaudi — current code is CUDA-first with vendor-specific shims. An AI-first PL targets MLIR as its compilation backend (StableHLO / Linalg dialects), gaining hardware portability for free while allowing hardware-specific annotations when needed:

```pseudo
-- Default: portable via MLIR → StableHLO → hardware
fn attention(...) : Tensor[...]

-- With hardware-specific hint
@target(cuda, wmma)     -- NVIDIA tensor cores
@target(tpu, dot_general) -- TPU systolic array
fn fused_attention(...) : Tensor[...]
```

This is where Mojo's architecture is strongest (§205.4.1). Chapter 179 covers the full StableHLO → MLIR → hardware stack in detail. §205.5 covers StableHLO as the convergence IR.

### 205.3.10 LLM-Legible Compact Kernel DSL

An LLM writing kernels in a compact, high-level DSL with Speed-of-Light (SOL) guidance — first-principles performance bounds derived from hardware specs — outperforms an LLM writing raw CUDA by a significant margin (arXiv 2603.29010, January 2026). The DSL is designed for LLM in-context reasoning over performance trade-offs, not for human readability. The key insight: the LLM needs structured feedback ("this kernel is at 73% of memory bandwidth peak") not just a performance number, and a compact DSL makes patch edits local and semantics-preserving.

---

## 205.4 Language Survey

### 205.4.1 Mojo (Modular)

**Status:** Production / open-sourcing in progress (compiler expected Apache 2.0, end 2026); Apache 2.0 kernel library (450,000+ lines) released 2025.

The most production-ready attempt to solve the two-language problem in AI. Mojo is positioned as "Python syntax sugar for MLIR" (Jeremy Howard). Key properties:

- **Python syntax compatibility** — existing PyTorch/NumPy code runs unchanged; new Mojo code adds ownership semantics, SIMD types, and `@parameter` compile-time specialisation
- **MLIR as native IR** — direct access to the MLIR compilation pipeline; hardware-portable by construction; Mojo is effectively a high-level syntax for composing MLIR passes (Chapter 179)
- **SIMD/tensor types as first-class** — `SIMD[DType.float32, 8]`, not NumPy arrays; width is a compile-time parameter
- **Ownership + borrow checker** — Rust-style memory safety without GC overhead
- **MAX Engine** — production LLM inference/training runtime; 4× speedup on FLUX.2 image generation (March 2026); cross-compatible NVIDIA/AMD/CPU

**Gaps vs AI-first ideal:** No dependent tensor types (dimension names are not types — they are runtime values), no first-class AD at the language level (AD is a framework library, not a language primitive), no parallelism-as-types. Still Python-shaped: the syntax inherits Python conventions, which are not the most AI-legible (§205.1).

*References: [modular.com/mojo](https://www.modular.com/mojo) · [Mojo Manual](https://docs.modular.com/mojo/manual/) · [MAX Engine](https://docs.modular.com/max/changelog/)*

### 205.4.2 Dex (Google Research)

**Status:** Research / experimental — "expect monstrous bugs and razor-sharp edges."

The language closest to the AI-first transformer PL ideal on type-theoretic grounds. Dex is explicitly described as "higher-order dependently-typed Einstein summation." Core properties:

- **Typed indices** — array dimensions are types; index operations are verified at compile time; shape mismatches are type errors, not runtime exceptions
- **First-class differentiation** — `grad` is a language primitive with a sound denotational semantics (Chapter 204 §204.6.2 formalises this for the unified PL)
- **Purely functional** — no mutation, no hidden state; all effects are explicit
- **User-directed parallelism** — parallel maps expressed as higher-order functions over typed index sets
- **Algebraic data types + type classes** — full ML-family type system

Practically, Dex is research-grade and slow. The Google team has not pursued production. But it is the clearest existence proof of what the type theory for an AI-first tensor PL looks like — design consideration §205.3.1 (named tensor dimensions) and §205.3.8 (first-class einsum) are directly Dex.

*References: [Dex Tutorial](https://google-research.github.io/dex-lang/examples/tutorial.html) · [arXiv: array programming with typed indices](https://openreview.net/pdf?id=rJxd7vsWPS)*

### 205.4.3 Futhark (DIKU, University of Copenhagen)

**Status:** Active research, mature for its niche (PLDI 2017 origin, active 2025 publications).

Futhark compiles purely functional data-parallel programs to efficient CUDA or OpenCL. Uniqueness types enforce safe in-place array updates without sacrificing referential transparency (a dependent-type-adjacent mechanism). Supports automatic differentiation.

Recent 2025 work: WebGPU backend optimisation, convolution optimisation, multi-precision integer arithmetic. Not ML-specific, but the type system and compilation model are directly applicable to transformer kernels. More academically rigorous than Mojo; less production-ready. The uniqueness type system is an alternative to the `T^1` linear types from Chapter 204 §204.5 (Linear-Use) for safe in-place update.

*References: [futhark-lang.org](https://futhark-lang.org/) · [PLDI 2017 paper](https://futhark-lang.org/publications/pldi17.pdf)*

### 205.4.4 Exo 2 (MIT CSAIL, ASPLOS 2025)

**Status:** Active research, ASPLOS 2025 publication.

Exo inverts the traditional compiler model: instead of a fixed optimisation pipeline, the performance engineer (or AI agent) writes *scheduling transformations* as library functions applied to object code. The key 2025 innovation is **Cursors** — stable references to points in object code that survive transformations, enabling encapsulation of schedules in libraries.

Results: 100× reduction in total schedule code; performance competitive with or better than MKL, OpenBLAS, and BLIS on BLAS kernels underlying transformer GEMM operations. The exocompilation model is directly relevant to design consideration §205.3.5 (kernel fusion as compiler-derived): an AI agent writes scheduling transformations in the Exo language; the compiler applies them to the object code.

The separation of *specification* (what the kernel computes) from *schedule* (how it executes) is exactly the goal of design consideration §205.3.5. Exo proves this separation is implementable for production BLAS kernels.

*References: [exo-lang.dev](https://exo-lang.dev/) · [ASPLOS 2025 paper](https://techxplore.com/news/2025-03-exo-language-high-code.html) · [PLDI 2022 paper](https://dl.acm.org/doi/10.1145/3519939.3523446)*

### 205.4.5 Clef

**Status:** Early development (2025); compiler frontend operational for a growing subset.

Clef describes itself as "a concurrent systems language for the AI and quantum eras." Most relevant properties:

- **Dimensional type safety** — dimension constraints flow through the Program Semantic Graph (PSG) and persist through MLIR generation, erased only at final lowering. Unlike .NET phantom types, they inform code generation decisions.
- **Memory safety as coeffect algebra** — memory safety properties tracked via coeffects (the dual of effects), verified at compile time
- **MLIR backend** — targets CPU, GPU, NPU, FPGA via MLIR dialects
- **Proof-carrying** — supports formal property verification

Too early to use in production, but the dimensional type safety design is the most explicit articulation of what shape-safe transformer code could look like at the PL level — design consideration §205.3.1 directly instantiated in a new language design.

*References: [clef-lang.com](https://clef-lang.com/) · [Dimensional Type Safety](https://clef-lang.com/docs/design/types/dimensional-type-safety/)*

### 205.4.6 Burn (Rust)

**Status:** Production-viable for inference; training support maturing.

Burn is a tensor library and deep learning framework leveraging Rust's ownership model to apply optimisations (kernel fusion, memory reuse) normally available only in static-graph frameworks, without sacrificing dynamic flexibility. Key properties: ONNX import, multi-backend (CUDA, ROCm, CPU, WebAssembly), multi-GPU training (v0.8.1+), no GC for predictable latency.

**Fundamental limitation:** Rust's type system has no native dependent types. Tensor shapes are runtime values, not compile-time types. This is a PL limitation, not a Burn implementation choice — design consideration §205.3.1 cannot be satisfied within Rust without a language extension. Chapter 178 §178.7 covers Burn in the Rust compiler ecosystem context.

*References: [burn.dev](https://burn.dev/) · [GitHub](https://github.com/tracel-ai/burn)*

---

## 205.5 StableHLO and MLIR as the Convergence IR

StableHLO is the shared intermediate representation that PyTorch, JAX, and TensorFlow all compile to, backed by Google, Meta, AMD, NVIDIA, Apple, Intel, and others. ~100 operations with full type inference and verifiers. A 5-year backward compatibility guarantee (from 2023) makes it the stable target IR that production hardware backends can commit to.

MLIR's Linalg dialect is the lowest-common-denominator abstraction for ML operations: named operands, indexing maps, iterator types (parallel/reduction/window). The compilation path from an AI-first PL is:

```
Named tensor types → tensor_named dialect
{Gradient} effect  → effects dialect (VJP insertion)
{Stochastic} effect → effects dialect (PRNGKey threading)
Parallelism types  → distribution dialect (collective insertion)
                  ↓
              Linalg dialect
                  ↓
             StableHLO
                  ↓
  hardware (NVPTX, ROCm, TPU, CPU vectorised)
```

Clef, Mojo, and Dex all use MLIR backends. The unified AI-first PL must target MLIR to gain hardware portability as a compiler output rather than a runtime add-on. Chapter 179 covers the `torch.export → StableHLO → MLIR → target ISA` path in detail. Chapter 205's design consideration §205.3.9 is the PL-level view of the same path.

The `tensor_named` dialect needed for design consideration §205.3.1 (named tensor dimensions) does not exist in MLIR in-tree as of 2026 — it is an open implementation task, addressed in the Chapter 207 §207.3 Phase 1 deliverable.

---

## 205.6 LLM-Driven Kernel Generation

A parallel research track sidesteps PL design: use LLMs to generate and iteratively refine transformer kernels, guided by compact DSLs and performance oracles. Three clusters have emerged.

### 205.6.1 Agentic Generate-Test-Refine Loops

**AutoKernel** (arXiv 2603.21331): Autonomous agent loop that profiles a PyTorch model, ranks bottlenecks via Amdahl's law, and iteratively refines kernels through hundreds of automated experiments. A 5-stage validation harness (smoke tests, shape sweeps, numerical stability, determinism, edge cases) gates each candidate. Results on H100: 5.29× over PyTorch eager on RMSNorm, 2.82× on softmax; 9 transformer kernel types covered.

**KernelSmith** (arXiv 2603.28342): Pairs an evolutionary agent with evolution-oriented training — converts optimisation trajectories into step-centric supervision, training models as "strong local improvers" rather than one-shot generators. SOTA on KernelBench: 3.70× average speedup; 14.59× on the DeepSeek Engram module. PRs merged into SGLang and LMDeploy production systems.

**CuTeGen** (arXiv 2604.01489): Targets CuTe (NVIDIA's CUDA Template abstraction) rather than raw CUDA or Triton. Key insight: withhold profiling metrics initially (staged optimisation) to force structural improvements before low-level tuning. Localized patch edits rather than full rewrites. Results: competitive with cuBLAS on GEMM specialisations.

**arXiv 2603.29010** (January 2026): Compact DSL + Speed-of-Light (SOL) guidance — first-principles performance bounds steer LLM kernel search without measurement. The DSL is designed for LLM in-context reasoning over performance trade-offs, not human readability. This is design consideration §205.3.10 instantiated in a research system.

### 205.6.2 Compiler-LLM Cooperation Across Abstraction Levels

**ACCLAIM** (arXiv 2604.04238): Integrates LLM agents at source code, LLVM IR, and assembly simultaneously. A guiding agent (Claude 3.7 Sonnet) orchestrates compiler passes and level-specific LLM agents. Result: 1.25× geometric mean speedup over `clang -O3` on 100 programs; top 1% of programs: up to 1384× speedup. The multi-level principle applies directly to transformer PL compilation stacks — an AI-first ML PL compiler could apply LLM reasoning at MLIR dialect boundaries.

### 205.6.3 Dynamic Shape Handling

**DVM: Dynamic Virtual Machine** (arXiv 2603.24239): Addresses the problem that compile-time approaches cannot solve — dynamic tensor shapes and control flow. Current compilers take 105× longer to compile than to execute dynamic models. DVM encodes programs as bytecode on the CPU, decoded to tile-level virtual instructions for direct NPU execution. Results: up to 11.77× speedup vs TorchInductor, 5 orders of magnitude faster compilation, evaluated on BERT, Qwen, Llama.

DVM is relevant to design consideration §205.3.1: static named tensor dimensions require shapes known at compile time; DVM shows what a runtime fallback must look like for genuinely dynamic shapes. The ideal PL handles static shapes via dependent types and falls back to a DVM-like bytecode for dynamic shapes.

**Emerging synthesis.** The kernel generation community is converging on a PL design conclusion empirically: the ideal target for LLM kernel authorship is a mid-level structured DSL (CuTe, Triton, or a purpose-designed compact language) with four properties: formal enough that correctness is checkable without running; structured enough that patch edits are local and semantics-preserving; hardware-exposed enough to reason about tiling, fusion, and memory hierarchy; performance-oracle-connected (SOL bounds) so the LLM receives structured feedback without measurement. This is a PL design problem the kernel community is solving from the bottom up.

---

## 205.7 Design Space Assessment: What the Unified PL Still Lacks

Combining the ten design considerations with the language survey reveals specific gaps — properties that no current system delivers at production quality:

| Property | Closest Existing | Gap |
|---|---|---|
| Named tensor dimensions as compile-time types | Dex, Clef (partial) | No production system; `tensor_named` MLIR dialect not in-tree |
| First-class AD with sound type theory | Dex | Research-only; not in any production ML framework |
| Parallelism strategy as type annotations | None | Completely open; collective insertion is runtime-only |
| Device + precision as first-class types | Mojo (partial SIMD types) | No full `Tensor[Device, Precision, ...]` type parameter |
| Kernel fusion as compiler-derived | XLA (implicit), Halide | Not for transformers at the source language level |
| Einsum as first-class typed syntax | Dex | Research-only; production code uses string-embedded einsum |
| MLIR backend for hardware portability | Mojo, Clef | Not combined with dependent tensor types |
| LLM-legible compact kernel DSL | AutoKernel, KernelSmith, CuTeGen, arXiv 2603.29010 | Converging on CuTe/Triton-level; no unified design |
| Compiler-LLM cooperation at MLIR boundaries | ACCLAIM (LLVM IR level) | Not yet applied to MLIR dialect boundaries |
| Dynamic shape handling without 105× compile overhead | DVM (NPU bytecode VM) | NPU-specific; CUDA/MLIR equivalent open |

Note: `{Stochastic}` as an algebraic effect is not a gap — it is a shared property from the unified PL design already established in Chapter 203 §203.7 and formalised in Chapter 204 §204.5. The ideal synthesis: **Dex's type theory + Mojo's MLIR backend + Exo 2's schedulability + first-class parallelism effects + the coding-agent PL foundation from Chapters 203–204**. No single system achieves this combination. Chapter 207 §207.3 provides the build roadmap for assembling it.

---

## 205.8 The Swift for TensorFlow Retrospective

Swift for TensorFlow (2018–2021) had the theoretically correct instincts: a systems language with Python interop, first-class differentiable programming, ownership semantics, and a strong type system. It failed for identifiable reasons:

1. **No LLM coding agents.** The steep learning curve had no mitigation; human users refused to leave PyTorch.
2. **Apple captured Swift.** The Swift Evolution process prioritised iOS/macOS use cases over ML research needs — language changes required for AD conflicted with Swift's general-purpose design goals.
3. **JAX proved the pragmatic alternative worked.** Pythonic functional transforms (`jit`, `grad`, `vmap`) were good enough for researchers; the type system gap was tolerable.

**The lesson for 2026.** With coding agents, the human ergonomics constraint is largely gone. A language as unfamiliar as Dex or as formal as Lean 4 is learnable by AI. Swift for TF failed because it required humans to switch; an AI-authored PL for the transformer tier does not. The question is whether the ecosystem investment is justified — for the kernel layer, where Mojo and Exo 2 are pushing the frontier, the answer is yes. For the model architecture layer, Dex's type theory is the right model, but no production implementation exists.

**The Phase 0 gate (Chapter 207 §207.3).** Swift for TF skipped the soundness argument for its AD type system and accumulated type-system inconsistencies that could not be fixed retroactively. Chapter 204 §204.6.3 defines the three hardest type interactions; Phase 0 of the build roadmap requires proving them correct before production implementation begins. This gate is the specific lesson from the Swift for TF failure applied to the unified AI-first PL.

---

## Chapter 205 Summary

- Transformer model development requires ten additional design considerations beyond the general AI-first PL properties: named typed tensor dimensions, AD as a language-level transform, parallelism strategies as type annotations, device and precision as types, compiler-derived kernel fusion, stochasticity as an algebraic effect, recomputation annotations, first-class einsum syntax, MLIR as the target IR, and LLM-legible compact kernel DSLs.
- Python frameworks (PyTorch, JAX, Triton) make pragmatic choices correct for human authors — dynamic graphs, Python syntax, runtime shape checking — that are wrong for AI authors; the human ergonomics constraint is eliminated when the agent is an LLM.
- Production serving systems (vLLM, llama.cpp, TensorRT-LLM, SGLang) impose concrete demands that directly motivate the ten design considerations.
- Mojo is the most production-ready (MLIR backend, MAX Engine, 450k lines of open-source kernels) but lacks dependent tensor types and first-class AD.
- Dex is the most theoretically correct (typed indices, first-class AD, purely functional) but is research-grade and not production-ready.
- Futhark (uniqueness types, auto-AD, CUDA/OpenCL) and Exo 2 (schedulable kernel authorship with Cursors) address complementary aspects of the design space.
- Clef (dimensional type safety through MLIR) and Burn (Rust-native, no dependent types) represent early-stage and production-adjacent points on the spectrum.
- StableHLO/MLIR is the universal convergence IR; the `tensor_named` dialect for named dimension preservation is an open implementation task.
- LLM-driven kernel generation (AutoKernel, KernelSmith, CuTeGen, SOL-guided DSLs, ACCLAIM) is converging empirically on the same PL design conclusion: a mid-level structured DSL with formal correctness guarantees and SOL performance feedback.
- Swift for TensorFlow's failure teaches: skip the type-system soundness proof at your peril — the Phase 0 gate (Chapter 207 §207.3) is the specific lesson applied.
- The open design space: Dex's type theory + Mojo's MLIR backend + Exo 2's schedulability + first-class parallelism effects, unified with the coding-agent PL foundation from Chapters 203–204 — not yet achieved in any single system.
