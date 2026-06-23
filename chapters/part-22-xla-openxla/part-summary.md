---
# Part XXII — XLA / OpenXLA — Part Summary

*This part covers XLA's full compilation stack — from HLO IR and StableHLO through CPU/GPU code generation, the PJRT device abstraction, and distributed SPMD partitioning — equipping the reader to understand, extend, and target the production ML compiler infrastructure used by JAX, TensorFlow, and PyTorch.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 153 | XLA Architecture | OpenXLA project structure, HLO IR hierarchy, compilation pipeline |
| 154 | HLO and StableHLO | HLO opcode set, StableHLO versioned dialect, VHLO, CHLO |
| 155 | XLA:CPU | IrEmitter, OneDNN/Eigen integration, thunk execution model |
| 156 | XLA:GPU | GPU fusion pipeline, Triton GEMM, auto-tuning, NCCL collectives |
| 157 | PJRT: The Plugin Runtime Interface | PJRT C API, plugin loading, IFRT distributed runtime |
| 158 | SPMD, GSPMD, and Auto-Sharding | HloSharding annotations, SpmdPartitioner, ILP auto-sharding |

## Part Overview

XLA (Accelerated Linear Algebra) predates MLIR by roughly three years, yet the two systems have grown together into a complementary production stack. This part traces the complete path a computation takes from an ML framework to machine code on CPU or GPU, making the design choices at each layer explicit.

The part opens with XLA's architectural foundation (Chapter 153): the `HloModule` → `HloComputation` → `HloInstruction` hierarchy, the pass pipeline that transforms that IR through algebraic simplification, layout assignment, and fusion, and the emerging MLIR integration via StableHLO. Chapter 154 deepens the treatment of the IR itself, cataloguing all roughly 130 HLO opcodes across element-wise arithmetic, linear algebra, shape manipulation, reductions, control flow, and collective communication. It then introduces StableHLO — a versioned MLIR dialect that serves as the stable serialization contract between frameworks and XLA — and its versioning substrate VHLO, which provides five-year forward/backward compatibility for serialized models. CHLO (Complex HLO) sits above StableHLO and provides composite ops that decompose to StableHLO primitives.

With the IR understood, Chapters 155 and 156 examine the two main backend targets. The CPU backend (Chapter 155) uses an `IrEmitter` visitor to emit LLVM IR directly from HLO, selecting between OneDNN (for AVX-512/AMX-accelerated GEMMs), Eigen (portable multi-threaded fallback), or scalar loops based on matrix dimensions and hardware features. A thunk execution model — where each compiled unit is a `CpuThunk` invoked from a sequence — separates compiled kernels from buffer management. The GPU backend (Chapter 156) introduces a more complex pipeline: GPU-specific HLO passes transform dot products into Triton-based custom calls (`TritonGemmRewriter`), categorize fusions into `kLoop`/`kInput`/`kTriton`/`kCustom` kinds, and benchmark configurations through an auto-tuner whose results are persisted in a proto cache. Kernel emission uses NVVM intrinsics via `GpuIrEmitter`; NCCL collectives handle multi-GPU communication, and an asynchronous AllReduce mechanism overlaps communication with compute.

Chapter 157 steps back to the compilation interface: PJRT (Pretty Just a Runtime) is a stable C ABI — a struct of function pointers — that decouples frameworks from backends. Any hardware vendor can ship a shared library implementing `GetPjrtApi()`; JAX, TensorFlow, and PyTorch all program against this interface rather than against XLA's internal C++ API. IFRT (Interfaces for Runtime) extends PJRT with a distributed Array abstraction that enables logical sharded tensors spanning multiple hosts. The final chapter (158) addresses multi-device training: `HloSharding` annotations encode how tensors are tiled or replicated across device meshes; the `SpmdPartitioner` pass materializes those annotations into partitioned ops and collective communication; GSPMD generalizes this to named multi-axis meshes (data, model, expert axes simultaneously); and the `AutoSharding` pass formulates optimal sharding selection as an Integer Linear Program solved by OR-Tools, minimizing communication plus memory cost subject to correctness constraints.

A reader who completes this part will be able to read and modify HLO dumps, write a `HloPassInterface` pass, author a StableHLO-consuming transformation, interpret XLA's GPU fusion and auto-tuning decisions, implement a minimal PJRT plugin, and reason about SPMD sharding strategies and the ILP formulation that selects among them.

## Key Concepts Introduced

- **HloModule / HloComputation / HloInstruction**: XLA's three-level IR hierarchy; a functional dataflow graph with static shapes where each node is a typed operation consuming and producing tensors.
- **HloOpcode**: The enumeration of ~130 distinct XLA operations spanning arithmetic, linear algebra, shape manipulation, reductions, control flow, and collective communication.
- **Shape and layout (minor-to-major)**: XLA's `Shape` class encodes element type, dimensions, and memory layout as a minor-to-major dimension ordering; column-major (`{0,1}`) is preferred for GPU, row-major (`{1,0}`) for CPU.
- **StableHLO**: A versioned MLIR dialect providing a 5-year serialization compatibility guarantee for ML models; the primary input contract between frameworks (JAX, TF, torch-mlir) and XLA.
- **VHLO (Versioned HLO)**: The underlying versioning substrate for StableHLO; each op version is a distinct VHLO op with a version stamp, enabling forward-compatible bytecode serialization.
- **CHLO (Complex HLO)**: An MLIR dialect above StableHLO containing composite ops (e.g., `chlo.erf`, `chlo.lgamma`) that decompose to StableHLO sequences via `ChloLegalizeToStablehlo`.
- **InstructionFusion / fusion kinds**: XLA's fusion pass merges element-wise ops into single GPU kernels; fusion categories `kLoop`, `kInput`, `kOutput`, `kTriton`, and `kCustom` each map to different codegen paths.
- **BufferAssignment**: Liveness analysis and best-fit buffer packing that maps logical HLO values to physical memory allocations; enables output aliasing via `donate_argnums`/`DynamicUpdateSlice` in-place ops.
- **Thunk execution model**: `CpuThunk` and `GpuThunk` subclasses (KernelThunk, OneDNNGemmThunk, AllReduceThunk, WhileThunk, etc.) form a sequence that separates compiled kernel code from runtime buffer management and dispatch.
- **Triton GEMM integration**: `TritonGemmRewriter` converts eligible `kDot` ops to `kCustomCall["__triton_gemm"]`; the Triton compiler emits PTX from MLIR with configurable tiling parameters (BLOCK_M/N/K, NUM_STAGES, NUM_WARPS); auto-tuning selects the optimal configuration.
- **PJRT (Plugin Runtime Interface)**: A stable C ABI (struct of function pointers) defining `PjRtClient`, `PjRtDevice`, `PjRtBuffer`, and `PjRtLoadedExecutable`; backends ship as `GetPjrtApi()`-exporting shared libraries, decoupling frameworks from hardware implementations.
- **IFRT (Interfaces for Runtime)**: A distributed runtime abstraction built on PJRT that adds a sharded `Array` type — logical tensors whose shards reside on different devices across multiple hosts — enabling multi-host model-parallel execution.
- **HloSharding**: XLA's tensor distribution annotation encoding `Replicated`, `Tile` (split along mesh dimensions), `PartialTile` (tiled + replicated), or `Manual` strategies via a tile assignment array mapping logical shards to physical devices.
- **SpmdPartitioner / GSPMD**: The `SpmdPartitioner` HLO pass transforms annotated single-device programs into partitioned multi-device programs by splitting ops over local shards and inserting collective communication; GSPMD extends this to named multi-axis device meshes supporting data, model, and expert parallelism simultaneously.
- **AutoSharding (ILP)**: An `HloModulePass` that formulates sharding selection as an Integer Linear Program — binary variables per instruction per strategy, communication and memory cost objectives, consistency constraints — solved by OR-Tools to find optimal tensor distribution without manual annotation.

## How This Part Fits the Book

Parts XIX–XXI (MLIR Foundations, In-Tree Dialects, MLIR Transformations — Chapters 126–152) established the MLIR type system, operation semantics, dialect conversion framework, and pass infrastructure that StableHLO and XLA's MLIR integration paths depend on; readers should particularly understand `tensor` types, `linalg.generic`, and the `ConversionPattern` mechanism before working through Chapter 154's StableHLO material. Part XXIII (MLIR in Production — Chapters 159–166) builds directly on this part by examining how XLA's abstractions are used in real deployment scenarios, including the Triton dialect (Chapter 163 is referenced from Chapter 156), IREE, and model serving; PJRT's plugin model (Chapter 157) is the foundation on which those production systems register hardware targets.

## Cross-Part Dependencies

- **Ch 126–128 (Part XIX — MLIR Foundations)**: The `MLIRContext`, `OpBuilder`, `DialectConversion`, and MLIR bytecode format underlie StableHLO's C++ API and VHLO serialization described in Ch 154.
- **Ch 138–142 (Part XX — In-Tree Dialects)**: The `linalg.generic`, `tensor`, and `scf` dialects are the target of XLA's experimental `XlaGpuToLinalg` codegen path described in Ch 153 and Ch 156; `bufferization` concepts inform Ch 155's arena allocation model.
- **Ch 148–152 (Part XXI — MLIR Transformations)**: The `transform` dialect, tiling, and vectorization passes discussed there are the upstream basis for the `CpuFusionPass` and Triton tile scheduling explained in Ch 155–156.
- **Ch 163 (Part XXIII — Triton)**: The Triton MLIR dialect and compilation pipeline referenced in Ch 156's Triton GEMM section are covered in depth in Part XXIII.
- **Ch 166 (Part XXIII — IREE)**: IREE's HAL and plugin model closely parallel PJRT (Ch 157); both are covered as complementary production MLIR runtimes in Part XXIII.
- **Ch 172–174 (Part XXIV — Verified Compilation)**: The formal semantics of HLO rewrites mentioned in Ch 154's roadmap (Vellvm-style Lean 4 proofs for AlgebraicSimplifier) and the Alive2-inspired verification of InstructionFusion transforms connect to Part XXIV's treatment of verified compilation.
- **Ch 209–212 (Part XXXI — Frontier AI Evolution)**: The distributed training scaling challenges motivating GSPMD (Ch 158) and the auto-sharding ILP are revisited in Part XXXI's treatment of compiler infrastructure for frontier model training.

## Navigation

- ← Part XXI — MLIR Transformations
- → Part XXIII — MLIR in Production

---

*@copyright jreuben11*
