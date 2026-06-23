# Part XXIII — MLIR in Production — Part Summary

*This part surveys the complete ecosystem of production ML compilers, GPU kernel generators, and language systems built on MLIR, showing how the infrastructure from Parts XIX–XXII is applied to real-world deployment problems spanning cloud GPUs to bare-metal microcontrollers.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 159 | Building a Domain-Specific Compiler | ODS dialect authoring, multi-stage lowering pipelines |
| 160 | MLIR Python Bindings | pybind11/C-API IR construction, PassManager, ORC JIT |
| 161 | torch-mlir, ONNX-MLIR, and JAX/TF Bridges | Framework-to-MLIR ingestion; TOSA portable operator set |
| 162 | IREE: A Deployment Compiler | Flow/Stream/HAL dialects, VMFB portable binary format |
| 163 | Triton: A Compiler for GPU Kernels | Block-level GPU programming, TritonGPU dialect encodings |
| 164 | CUDA Tile IR | Hopper WGMMA/TMA, nvgpu dialect, CuTe layout algebra |
| 165 | GPU Compilation Through MLIR | Linalg-to-PTX pipeline, NVVM/ROCDL, vector.contract |
| 166 | Mojo, Polygeist, Enzyme-MLIR, and Beyond | MLIR-powered languages, polyhedral lifting, AD, FHE, EDA |
| 202 | Apache TVM: An ML Operator Compiler | Schedule synthesis, AutoTVM/Meta-Schedule, microTVM |
| 229 | torch.compile: TorchDynamo, AOTAutograd, TorchInductor | PEP 523 graph capture, joint fwd/bwd tracing, Triton codegen |

## Part Overview

Part XXIII opens with the foundational skill every MLIR practitioner needs: building a complete out-of-tree dialect and lowering pipeline from scratch (Chapter 159). Using a running "MatAlg" example, it demonstrates how ODS TableGen definitions produce C++ boilerplate, how `TypeConverter` and `ConversionTarget` govern multi-stage lowering (MatAlg → Linalg → Affine → SCF → LLVM), and how FileCheck tests validate every stage. Chapter 160 then exposes this machinery to Python through MLIR's C-API-backed pybind11 bindings — the same bindings that JAX and IREE use internally — covering `Context`, `Module`, `InsertionPoint`, dialect-specific op builders, `PassManager.parse()`, `ExecutionEngine` ORC JIT, and the NumPy/memref bridge.

The middle chapters address the ML framework ingestion problem. Chapter 161 surveys how four major frameworks — PyTorch (torch-mlir), ONNX (ONNX-MLIR), JAX, and TensorFlow — produce MLIR. torch-mlir lowers `torch.export`-traced programs through the Torch dialect (ATen ops) to StableHLO, Linalg, or TOSA; ONNX-MLIR compiles `.onnx` files to native shared libraries via its own `onnx` and `krnl` dialects; JAX's pipeline is StableHLO-native end-to-end with a 5-year forward-compatibility guarantee; TOSA provides a portable ~60-op inference operator set consumed by embedded targets and IREE. Chapter 162 then shows IREE as the canonical end-to-end MLIR deployment compiler: its `flow` dialect identifies kernel fusion boundaries, `stream` models async GPU execution with typed resource lifetimes, and `hal` abstracts device differences into a portable command-buffer model. The resulting `.vmfb` FlatBuffer encapsulates compiled kernel variants for CPU, CUDA, Vulkan, Metal, and ROCm, enabling single-artifact multi-target deployment.

The GPU focus sharpens through Chapters 163–165. Chapter 163 covers Triton's block-level programming model and its MLIR compilation pipeline (Python AST → Triton dialect → TritonGPU dialect with `blocked`/`mma`/`shared` encodings → LLVM/NVPTX). Chapter 164 descends to Hopper-specific hardware: WGMMA 128-thread cluster MMA instructions, TMA async bulk copy, CuTe layout algebra, and their MLIR representations in the `nvgpu` dialect (`nvgpu.warpgroup.mma`, `nvgpu.tma.async.load`, `mbarrier`). Chapter 165 then presents the generic MLIR GPU compilation path — two-level tiling for SMEM and register tiles, `vector.contract` as the abstract matmul, `ConvertGpuOpsToNVVMOps`/`ConvertGpuOpsToROCDLOps` lowering, `nvgpu.device_async_copy` pipelining, and the full `linalg.matmul → cubin` toolchain. Together these three chapters establish why GPU performance requires matching tile granularity to hardware tiers (blocks → SMEM, threads → registers, warpgroups → tensor cores).

The final three chapters broaden the lens. Chapter 166 surveys MLIR-powered systems outside the core ML compiler space: Mojo maps Python-compatible syntax to MLIR vector and GPU dialects; Polygeist lifts C/C++ affine loop nests for polyhedral transformation; Enzyme-MLIR performs forward and reverse automatic differentiation on any MLIR dialect via `enzyme.autodiff`; HEIR compiles FHE arithmetic circuits with noise-budget optimization; CIRCT applies MLIR infrastructure to hardware design; and VAST provides C AST-level dialects for security analysis. Chapter 202 covers Apache TVM, which takes an orthogonal approach to MLIR-based compilers through explicit schedule synthesis: loop nest primitives (`split`, `reorder`, `tensorize`, `cache_read/write`), AutoTVM template-guided search, Ansor template-free sketch generation, Meta-Schedule's pluggable search pipeline, BYOC pattern-matched subgraph offload, and the Relax unified IR (TVM Unity) that supersedes the Relay/TIR split. microTVM extends this to bare-metal MCUs with a static-C AOT executor and no heap allocator. Chapter 229 closes the part with `torch.compile`: TorchDynamo's PEP 523 frame hook intercepts CPython bytecode to build FX graphs without model changes; AOTAutograd jointly traces forward and backward passes enabling cross-kernel optimization; TorchInductor lowers FX graphs to Triton kernels (GPU) or vectorized C++ (CPU); and `torch.export`/AOTInductor provide serializable deployment artifacts with stable C ABI.

## Key Concepts Introduced

- **ODS (Operation Definition Specification)**: TableGen DSL that generates C++ dialect code, including op verifiers, assembly format parsers/printers, and Python binding stubs, from declarative `.td` files.
- **TypeConverter + ConversionTarget**: MLIR framework for specifying which types and ops are legal at each lowering stage and how to map types across dialect boundaries.
- **MLIR Python C API layer**: the stable ABI boundary between pybind11 Python bindings and MLIR C++ internals; all Python IR construction goes through this layer to remain ABI-stable.
- **Torch dialect / vtensor type**: MLIR representation of PyTorch ATen operations with optional symbolic shape information; the output of torch-mlir's ingestion of `torch.export` programs.
- **TOSA (Tensor Operator Set Architecture)**: a portable ~60-op ML inference operator set with formal precision specifications; serves as a hardware-neutral intermediary between framework dialects and Linalg/LLVM backends.
- **IREE flow dialect**: identifies dispatch regions — groups of ops to be fused into a single GPU or CPU kernel — through `flow.dispatch.region` and `FormDispatchRegionsPass` heuristics based on producer-consumer topology.
- **IREE stream dialect**: models async GPU execution with `stream.cmd.execute` command blocks, typed resource lifetimes (`constant`, `transient`, `variable`, `external`), and `stream.timepoint` for dependency tracking.
- **VMFB (VM FlatBuffer)**: IREE's portable deployment artifact embedding compiled kernel variants, VM bytecode, and constant weights for multiple target backends in a single file.
- **TritonGPU dialect encodings**: `blocked`, `mma`, and `shared` encoding attributes that describe how tensor elements are distributed across threads, warps, and warpgroups; `triton_gpu.convert_layout` bridges incompatible encodings.
- **Software pipelining (Triton)**: `TritonGPUPipelinePass` overlaps K-loop compute with async prefetch to hide global memory latency; `num_stages` controls pipeline depth.
- **WGMMA (Warpgroup MMA)**: Hopper's 128-thread cluster-level matrix instruction computing 64×128×16 matmuls; represented in MLIR as `nvgpu.warpgroup.mma` operating on `!nvgpu.warpgroup.accumulator` register tiles.
- **TMA (Tensor Memory Accelerator)**: Hopper's hardware DMA unit for async bulk copies between global and shared memory; represented in MLIR as `nvgpu.tma.descriptor.create` + `nvgpu.tma.async.load` + `mbarrier` synchronization.
- **vector.contract**: MLIR's abstract matmul op representing a contracted tensor product; lowers to `nvvm.mma.*` (Ampere), `nvvm.wgmma.*` (Hopper), or `rocdl.wmma.*` (AMD) depending on target and vector shapes.
- **TVM schedule primitives**: `split`, `fuse`, `reorder`, `compute_at`, `vectorize`, `unroll`, `parallel`, `tensorize`, `cache_read/write` — composable loop nest transformations that preserve mathematical semantics while mapping computation to hardware.
- **Relax / TVM Unity**: TVM's unified IR where `R.call_tir` bridges graph-level `relax.Function` directly to loop-level `tir.PrimFunc` within the same `IRModule`, eliminating the Relay/TIR IR boundary and enabling dynamic shape support through `StructInfo`.
- **BYOC (Bring Your Own Codegen)**: TVM's pattern-matching + subgraph-offload mechanism routing matched operator subgraphs to external codegens (NPUs, DSPs, vendor libraries) while the TVM runtime handles dispatch.
- **Enzyme-MLIR `enzyme.autodiff` op**: applies forward or reverse automatic differentiation to arbitrary MLIR programs at the dialect level, before lowering to LLVM IR; works on any dialect with proper memory effects.
- **TorchDynamo PEP 523 frame hook**: intercepts CPython bytecode frame evaluation to symbolically execute tensor operations on proxy tensors and accumulate them into an FX graph, without requiring model source changes.
- **AOTAutograd**: jointly traces forward and backward passes before execution using `__torch_dispatch__`, producing static FX graphs for both that enable cross-kernel fusion, dead code elimination, and optimal min-cut activation checkpointing.

## How This Part Fits the Book

Part XXIII builds directly on the MLIR dialect and transformation infrastructure of Parts XIX–XXI (which introduce `affine`, `linalg`, `vector`, `scf`, `gpu`, `spirv`, `nvvm`, and the Transform dialect) and on the XLA/OpenXLA architecture of Part XXII (Chapters 153–158, which establish StableHLO as the inter-framework interchange and cover XLA's use of Triton). The frameworks and compilers in this part are consumers of all that infrastructure, demonstrating how production systems are assembled from the dialect building blocks. Part XXIV (Verified Compilation, starting at Chapter 193) then addresses the formal correctness question raised implicitly here — how to prove that the multi-stage lowerings in Chapters 159, 162, and 165 preserve semantics — building on the real-world complexity exposed in this part.

## Cross-Part Dependencies

- **Ch127 (Part XIX) — MLIR Core Infrastructure**: Provides `MLIRContext`, `DialectRegistry`, `PassManager`, and `OpBuilder` that every chapter in this part relies on for dialect authoring (Ch159), Python bindings (Ch160), and lowering pipelines (Ch162, Ch165).
- **Ch134 (Part XX) — Linalg Dialect**: Ch159 lowers MatAlg → `linalg.matmul`; Ch161 uses `tosa-to-linalg`; Ch162's flow dialect fuses linalg ops into dispatch regions; Ch165 starts from `linalg.matmul` and ends at cubin.
- **Ch136 (Part XX) — Vector Dialect**: Ch165 depends on `vector.contract`, `vector.transfer_read/write`, `vector.multi_reduction`; Ch163's TritonGPU encodings parallel the vector type system.
- **Ch143 (Part XXI) — Transform Dialect**: Ch162 uses transform sequences for CPU tiling; Ch164 uses `transform.nvgpu.rewrite_matmul_as_mma_sync`; Ch165 uses `transform.structured.tile_using_for`.
- **Ch153 (Part XXII) — XLA Architecture**: Establishes StableHLO as the input to IREE (Ch162) and the target of torch-mlir and JAX bridges (Ch161); XLA's Triton integration from Ch156 is extended in Ch163 section 163.8.
- **Ch193 (Part XXIV) — Alive2 and Vellvm**: The multi-stage lowering correctness problem introduced in Chapters 159, 162, and 165 motivates the formal verification machinery of Part XXIV; Ch164's roadmap explicitly references Vellvm for GPU semantics verification.
- **Ch179 (Part XXVI) — LLVM/MLIR for AI Full-Stack**: Ch202 cross-references this chapter for the integrative view of TVM within the broader AI compiler landscape.
- **Ch210 (Part XXXI) — JAX Ecosystem**: Ch229 cross-references this chapter; JAX's compilation path through StableHLO (detailed in Ch161, Ch162) and its relationship to `torch.compile` (Ch229) are complementary treatments of the same problem space.

## Navigation

- ← Part XXII — XLA / OpenXLA
- → Part XXIV — Verified Compilation

---

*@copyright jreuben11*
