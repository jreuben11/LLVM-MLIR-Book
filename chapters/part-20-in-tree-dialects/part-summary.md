# Part XX — In-Tree Dialects — Part Summary

*This part provides a systematic tour of MLIR's complete in-tree dialect ecosystem — from the always-loaded builtin primitives through arithmetic, memory, structured computation, control flow, GPU, SPIR-V, hardware vector ISAs, the LLVM terminus, and specialized dialects for parallelism, data layout, and C emission — giving the reader the full vocabulary of MLIR's standard dialect library.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 137 | Core Dialects | `builtin`, `func`, `arith`, `math`, `index`, `cf` foundations |
| 138 | Memory Dialects | `memref` types, operations, one-shot bufferization |
| 139 | Tensor and Linalg | Value-semantic tensors, `linalg.generic`, TOSA dialect |
| 140 | Affine and SCF | Polyhedral loops, structured control flow, Presburger arithmetic |
| 141 | Vector and Sparse | SIMD abstraction, sparse tensor encodings, sparsification |
| 142 | GPU Dialect Family | `gpu`, `nvgpu`, ROCDL, GPU lowering pipelines |
| 143 | SPIR-V Dialect | Vulkan/OpenCL shader IR, capability model, serialization |
| 144 | Hardware Vector Dialects | `arm_neon`, `arm_sve`, `arm_sme`, `x86vector`, `amdgpu`, `rocdl` |
| 145 | LLVM Dialect | MLIR representation of LLVM IR, lowering terminus |
| 146 | Async, OpenMP, OpenACC, DLTI, and EmitC | Concurrency, offload, data layout, C emission, MPI, WasmSSA |

## Part Overview

Part XX forms the practical core of the book's MLIR coverage. Where Part XIX established the framework — the compilation model, type system, pass infrastructure, and dialect authoring mechanisms — Part XX populates that framework with the dialects that real compiler pipelines consume every day. The ten chapters move from universal primitives to domain-specific layers, tracing a coherent compilation stack rather than an alphabetic catalog.

The part opens at the foundation with Chapter 137, covering the six dialects that every MLIR pipeline depends on: `builtin` (always-loaded module and type infrastructure), `func` (function abstraction with inlining and symbol visibility), `arith` (signless integer and IEEE float arithmetic with overflow and fast-math flags), `math` (transcendental functions with three distinct lowering paths), `index` (platform-word-size arithmetic for target-neutral loop bounds), and `cf` (unstructured control flow as the lowering target for all structured loop dialects). Chapter 138 covers the memory layer: `MemRefType` with its strided layout model expressed as affine maps, and the `bufferization` dialect's one-shot bufferization algorithm that converts value-semantic tensor programs to imperative memref programs while minimizing copies through alias analysis.

The computational heart of the part occupies Chapters 139 and 140. Chapter 139 introduces the `tensor` dialect's value-semantic array operations and `linalg.generic`, MLIR's universal structured computation abstraction parameterized by affine indexing maps and iterator types. Named linalg ops, the `TilingInterface`, producer-consumer fusion, `linalg.pack`/`unpack` for cache-optimal layout, and the complete TOSA dialect for portable ML operator semantics are all covered here. Chapter 140 descends to the control flow layer with the `affine` dialect's polyhedral loop representation — enabling static dependence analysis, automatic parallelization, and loop transformations via the in-tree Presburger arithmetic library — and the `scf` dialect's structured but non-polyhedral control flow that serves as the universal intermediate between high-level computations and the `cf` terminus.

Chapters 141 through 144 cover the parallelism and hardware-targeting stack. Chapter 141 treats the `vector` dialect's target-independent SIMD abstraction (including scalable vectors for SVE and RVV) alongside the `sparse_tensor` dialect's automatic sparsification of `linalg.generic` programs via the Bik-Wijshoff algorithm. Chapter 142 covers the `gpu` dialect family — the thread/block/grid execution model, NVGPU Tensor Core and TMA operations, ROCDL for AMD, and complete lowering pipelines to PTX and GCN. Chapter 143 examines the `spirv` dialect, which provides a 1:1 mapping to SPIR-V binary instructions including the capability system, specialization constants, and serialization to binary for Vulkan compute dispatch. Chapter 144 catalogs the hardware vector dialects — `arm_neon`, `arm_sve`, `arm_sme`, `x86vector`, `amdgpu`, and `rocdl` — each providing instruction-set-specific operations that sit below the generic `vector` dialect and above the LLVM backend.

The final two chapters complete the lowering stack. Chapter 145 covers the `llvm` dialect, the terminus of all MLIR lowering pipelines that target LLVM-based code generation, explaining how LLVM IR concepts (opaque pointers, GEP, phi nodes as block arguments, calling conventions, metadata) are represented in MLIR and how `LLVMTypeConverter` maps memref descriptors, how `--reconcile-unrealized-casts` closes the conversion, and how `mlir-translate --mlir-to-llvmir` produces the final `llvm::Module`. Chapter 146 collects five important specialized dialects: `async` for structured concurrency with coroutine-based lowering, `omp` for OpenMP 5.1 fork-join and offload semantics, `acc` for OpenACC GPU offload in scientific codes, `dlti` for structured machine description driving correct ABI and layout decisions, and `emitc` for MLIR-to-C code generation targeting embedded and MCU environments — plus the MPI dialect for distributed collective communication and the experimental WasmSSA dialect for MLIR-native WebAssembly targeting.

After completing this part, a reader can navigate the complete standard MLIR dialect library, understand the purpose and position of each dialect in a lowering pipeline, write passes that operate on any of the ten major dialects, construct end-to-end pipelines from `linalg.generic` through to PTX or C, and reason about dialect interactions — bufferization, tiling, vectorization, GPU mapping, and LLVM finalization — as a coherent compilation strategy rather than a collection of independent transformations.

## Key Concepts Introduced

- **`builtin` dialect and always-loaded status** — The dialect providing `ModuleOp`, all built-in types (`IntegerType`, `FloatType`, `TensorType`, `MemRefType`, `VectorType`), standard attributes including `DenseElementsAttr`, and `UnrealizedConversionCastOp` as the type-conversion bridge during partial lowerings.
- **Signless integer types and operation-carried signedness** — MLIR's `i32` carries no sign; signedness is encoded in the operation (`arith.divsi` vs `arith.divui`), a design that eliminates sign-proliferation in type systems.
- **`MemRefType` strided layout model** — The representation of memory regions as element type + shape + affine-map-based strided layout + address space, enabling `memref.subview` to produce aliased views without data copies by composing affine transformations.
- **One-shot bufferization (OSB)** — The two-phase algorithm (alias analysis then buffer assignment) that converts value-semantic tensor programs to memref programs while minimizing copies, driven by `BufferizableOpInterface` implementations on each op.
- **`linalg.generic` structural invariants** — The four invariants (no input-output aliasing, parallel dimensions independent, reduction dimensions sequential, affine index maps) that make structured computation uniformly tileble, vectorizable, parallelizable, and fusable.
- **`TilingInterface` and the tiling abstraction** — The interface that any structured op can implement to expose iteration domains and tiled implementations, enabling the Transform dialect to produce tiled `scf.for` or `scf.forall` nests around any linalg op.
- **Presburger arithmetic library** — MLIR's in-tree pure-C++ implementation of the Omega test for decidable integer linear arithmetic, used by the `affine` dialect for exact dependence analysis, loop fusion legality, and loop transformation correctness.
- **`scf.forall` and the tiled parallelism model** — The structured parallel loop with `shared_outs` and `tensor.parallel_insert_slice` that expresses non-overlapping tiled parallel output, serving as the high-level mapping target for GPU thread/block hierarchies.
- **Sparse tensor encoding attribute** — The `SparseTensorEncodingAttr` mechanism that annotates standard tensor types with compressed format specifications (CSR, CSC, COO, BSR) so that the same `linalg.generic` body compiles to both dense and sparse code.
- **Sparsification (Bik-Wijshoff algorithm)** — The `--sparsification` pass transformation that converts `linalg.generic` ops over sparsely-encoded tensors into explicit loops over `positions`/`coordinates`/`values` arrays, automatically skipping structural zeros.
- **GPU address space model** — The `#gpu.address_space<global>`, `#gpu.address_space<workgroup>`, and `#gpu.address_space<private>` annotations on `memref` types that drive correct instruction selection (global memory loads vs. shared memory loads) without runtime overhead.
- **SPIR-V capability system and `TargetEnvAttr`** — The mechanism by which the `spirv` dialect verifies that all operations used in a module have their required capabilities declared, and that target devices support those capabilities before serialization.
- **`LLVMTypeConverter` and memref descriptors** — The conversion that maps each `memref<...>` to an LLVM struct containing allocated pointer, aligned pointer, offset, and per-dimension size and stride arrays, enabling dynamic-shape memrefs to travel without global state.
- **`emitc.lvalue` and mutable C targets** — The type that models a C addressable location, enabling `emitc` to break from MLIR's otherwise purely-SSA discipline and generate readable mutable C code for embedded and MCU targets.
- **DLTI `DataLayout` API** — The structured alternative to LLVM's DataLayout string, providing per-type size and alignment entries on `builtin.module` via `#dlti.dl_spec`, queried at compile time to drive `index`-to-integer selection and struct layout computation.

## How This Part Fits the Book

Part XIX (MLIR Foundations, Chapters 128–136) established the MLIR compilation framework — the MLIRContext, op definition via ODS/TableGen, the type system, dialect registration, the pass manager, pattern rewriting, and dialect conversion mechanics. Part XX builds directly on that foundation, populating it with the specific dialects that real pipelines consume; every op shown in Part XX is defined using the ODS mechanisms from Chapter 130, every lowering pipeline uses the conversion infrastructure from Chapters 133–135, and every transformation pass runs under the framework from Chapter 132. Parts XXI (MLIR Transformations, Chapters 147–155) and XXII (XLA/OpenXLA, Chapters 156–161) depend heavily on this part: Part XXI's Transform dialect, canonicalization, and fusion passes all operate on the `linalg`, `tensor`, `scf`, and `vector` dialects introduced here, while Part XXII's StableHLO and XLA dialects sit above the `linalg`/`vector`/`gpu` stack as higher-level entry points that lower through the dialects in this part.

## Cross-Part Dependencies

- **Ch128 (Part XIX) — MLIR Core IR** — `builtin.module`, `builtin` types, and the distinction between SSA values and block arguments established here underlie every dialect in this part.
- **Ch130 (Part XIX) — ODS and Dialect Definition** — Every op in Chapters 137–146 is defined via ODS/TableGen; understanding the definition mechanism is prerequisite to understanding verifier constraints and generated API.
- **Ch133–135 (Part XIX) — Dialect Conversion, Pattern Rewriting, Pass Infrastructure** — The `--convert-arith-to-llvm`, `--one-shot-bufferize`, and all other conversion passes in this part use the infrastructure established there.
- **Ch70 (Part XI) — Polyhedral Foundations** — The Presburger arithmetic and Omega test used in Chapter 140's `affine` dialect dependence analysis have their mathematical foundation in Part XI.
- **Ch74 (Part XII) — Polly** — Polly uses ISL for polyhedral scheduling over the same affine loop nests that the `affine` dialect (Chapter 140) represents; the comparison between ISL and MLIR's Presburger library in Chapter 140 directly cross-references Polly.
- **Ch147–150 (Part XXI) — MLIR Transformations** — Transform dialect tiling and fusion operations (`transform.structured.tile_using_for`, `transform.structured.fuse`) operate on `linalg.generic` and `tensor` ops from Chapter 139; the vectorization pass from Part XXI targets `vector.contract` introduced in Chapter 141.
- **Ch156–161 (Part XXII) — XLA/OpenXLA** — StableHLO and XLA both lower to the `linalg`, `tensor`, `arith`, and `gpu` dialects covered in this part; Chapter 161's IREE backend uses `linalg`, `vector`, and GPU dialects as its core compilation substrate.
- **Ch162 (Part XXIII) — IREE** — IREE's flow → stream → HAL lowering chain takes TOSA (Ch139) and StableHLO as input and produces GPU dialect (Ch142) and `llvm` dialect (Ch145) as output; the entire IREE compilation stack is an application of the dialects in this part.
- **Ch127 (Part XVIII) — Flang** — Flang uses the `omp` and `acc` dialects (Chapter 146) as its representation for OpenMP and OpenACC directives in Fortran programs, then lowers through the standard `scf` → `cf` → `llvm` chain.
- **Ch106 (Part XV) — WebAssembly Target** — The WasmSSA dialect (Chapter 146) provides an MLIR-native alternative to the LLVM WebAssembly backend described in Chapter 106, with structured control flow regions replacing the CFG-to-stack-machine restructuring.
- **Ch93 (Part XIV) — Backend Architecture** — The `llvm` dialect (Chapter 145) and its translation to `llvm::Module` is the bridge between the MLIR backend discussed in Part XIV and the LLVM backend pipelines described there.
- **Ch180 (Part XXVI) — ML Compiler Frontends** — torch-mlir and ONNX-MLIR both lower to TOSA (Ch139) and then to linalg, making the `linalg.generic` and `tensor` dialects the common target for ML framework frontends.

## Navigation

- ← Part XIX — MLIR Foundations
- → Part XXI — MLIR Transformations

---

*@copyright jreuben11*
