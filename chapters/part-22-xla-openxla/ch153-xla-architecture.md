# Chapter 153 — XLA Architecture

*Part XXII — XLA and the OpenXLA Stack*

XLA (Accelerated Linear Algebra) is Google's domain-specific compiler for machine learning models. Originally developed as an internal project to speed up TensorFlow, XLA has evolved into a production-grade compilation infrastructure that powers training and inference across CPU, GPU, and Google's custom TPU hardware. The open-source fork, OpenXLA, is hosted at [github.com/openxla/xla](https://github.com/openxla/xla) and serves as the shared compilation backbone for JAX, TensorFlow, and increasingly for PyTorch via the torch-mlir bridge. This chapter examines XLA's architecture: how computations are represented, how they flow through an optimization pipeline, and how the system integrates with the broader MLIR ecosystem.

---

## 153.1 The OpenXLA Project

XLA predates MLIR by several years. It was designed as a purpose-built compiler for linear algebra graphs, with explicit knowledge of shapes at compile time and an aggressive fusion model that eliminates memory bandwidth bottlenecks. The OpenXLA project was announced in 2022 as a vendor-neutral home for XLA development, with contributions from Google, NVIDIA, AMD, Intel, and Apple.

The repository structure of [openxla/xla](https://github.com/openxla/xla) reflects XLA's dual identity as both a standalone compiler and an MLIR-integrated system:

```
xla/
├── xla/                   # Core XLA library
│   ├── hlo/               # HLO IR and passes
│   ├── client/            # Client API (LocalClient, XlaBuilder)
│   ├── service/           # Compilation service, buffer assignment
│   ├── backends/          # CPU and GPU backend implementations
│   ├── pjrt/              # Plugin Runtime Interface
│   └── mlir_hlo/          # MLIR dialects: MHLO, CHLO, StableHLO bridge
├── stablehlo/             # Vendored StableHLO dialect (openxla/stablehlo)
└── third_party/           # LLVM, Abseil, etc.
```

### 153.1.1 Frontend Integration Points

Multiple ML frameworks feed into XLA through different entry points:

- **JAX** (primary user): `jax.jit`-decorated Python functions are traced with abstract values to produce a `jaxpr` (JAX's internal IR), which is then lowered to StableHLO and handed to XLA's PJRT compilation interface.
- **TensorFlow**: the TF-XLA bridge converts TF graphs to HLO computations via `ConvertToHloModule`.
- **PyTorch**: `torch.export` → torch-mlir → StableHLO → XLA via PJRT.

The key insight is that all frameworks converge on **HLO** (or its stable MLIR counterpart, StableHLO) as the compiler interface. Anything above that level is framework business; everything below is XLA's domain.

```
JAX ──────────────────────────┐
TensorFlow ────────────────── ├──► StableHLO / HLO ──► XLA Optimizer ──► Backend
PyTorch (via torch-mlir) ─────┘
```

---

## 153.2 HLO: The Core Intermediate Representation

XLA's primary IR is HLO — a graph of named operations with static shapes. Unlike LLVM IR, HLO is a **functional, dataflow graph** rather than a control-flow graph of instructions. Each node in the graph is an `HloInstruction` that consumes typed tensors (operands) and produces a single typed tensor (output).

### 153.2.1 IR Hierarchy

```
HloModule
└── HloComputation (entry)
    ├── HloInstruction: %p0 = parameter(0), shape=[batch,784]{1,0}:f32
    ├── HloInstruction: %p1 = parameter(1), shape=[784,128]{1,0}:f32
    ├── HloInstruction: %dot = dot(%p0, %p1), lhs=[1], rhs=[0]
    └── HloInstruction: %relu = maximum(%dot, broadcast(0))
HloComputation (called: "body")
    └── ...
```

The three-level hierarchy is central to XLA:

- **`HloModule`**: The top-level container. Holds the entry computation plus any called computations (loop bodies, conditional branches, map/reduce lambdas). Corresponds to one compiled program. Defined in [`xla/hlo/ir/hlo_module.h`](https://github.com/openxla/xla/blob/main/xla/hlo/ir/hlo_module.h).

- **`HloComputation`**: A named function with a single root instruction. The root instruction's output is the computation's return value. Multiple computations can be embedded in a module — for example, `while` loops carry a separate body computation and a condition computation.

- **`HloInstruction`**: A single operation node. Every instruction has an opcode (`HloOpcode`), a `Shape` (type + layout), a list of operand `HloInstruction*` pointers, and optional attached computations (for `reduce`, `map`, `while`, etc.).

### 153.2.2 Shapes and Primitive Types

XLA's `Shape` class encodes both the element type and the multi-dimensional layout:

```cpp
// Simplified from xla/shape.h
class Shape {
  PrimitiveType element_type_;  // F32, S32, BF16, PRED, TUPLE, TOKEN, ...
  std::vector<int64_t> dimensions_;
  std::vector<int64_t> layout_;  // minor-to-major dimension ordering
  bool is_dynamic_dimension_[];  // whether each dim is dynamic
};
```

`PrimitiveType` covers the full ML numeric zoo: `F16`, `BF16`, `F32`, `F64`, `S8`, `S16`, `S32`, `S64`, `U8`, `U16`, `U32`, `U64`, `C64`, `C128`, `PRED`. The `layout` field specifies the minor-to-major ordering of dimensions — `{1,0}` means row-major (last dimension varies fastest, which is C order), while `{0,1}` means column-major (Fortran order). XLA internally prefers column-major for matrix operations to align with cuBLAS and BLAS conventions.

Tuple shapes aggregate multiple shapes into a product type, enabling multiple-output operations.

---

## 153.3 The XlaBuilder Client API

User code rarely constructs `HloInstruction` objects directly. Instead, the `XlaBuilder` API provides a C++ builder interface that accumulates operations and produces an `XlaComputation`:

```cpp
#include "xla/client/xla_builder.h"

xla::XlaBuilder b("my_computation");
auto x = xla::Parameter(&b, 0, xla::ShapeUtil::MakeShape(xla::F32, {4, 4}), "x");
auto y = xla::Parameter(&b, 1, xla::ShapeUtil::MakeShape(xla::F32, {4, 4}), "y");
auto dot = xla::Dot(x, y);
auto relu = xla::Max(dot, xla::ZerosLike(dot));

xla::XlaComputation computation = b.Build(relu).value();
```

The builder records operations lazily in a `HloModuleProto` (a protobuf), which is then deserialized into an in-memory `HloModule` for compilation. This protobuf representation is the serialization format used across process boundaries and for debugging.

JAX bypasses `XlaBuilder` entirely — it emits StableHLO MLIR directly and hands the serialized bytecode to PJRT for compilation.

---

## 153.4 The Compilation Pipeline

XLA's compilation pipeline is organized as a sequence of `HloPassInterface` passes collected into `HloPassPipeline` objects. The high-level flow is:

```
HloModule (input)
    │
    ▼
HloVerifier ─────────────────── validate structure
    │
    ▼
HloPassPipeline("simplification")
    ├── AlgebraicSimplifier ────── identity elim, broadcast folding
    ├── HloConstantFolding ──────── constant propagation
    ├── ReshapeMover ───────────── push reshapes outward
    └── HloDCE ──────────────────── dead code elimination
    │
    ▼
HloPassPipeline("layout_assignment")
    └── LayoutAssignment ────────── assign row/col-major per op
    │
    ▼
HloPassPipeline("fusion")
    ├── InstructionFusion ───────── element-wise → loop fusion
    └── FusionMerger ────────────── merge adjacent fusions
    │
    ▼
HloPassPipeline("spmd_partitioner")   [if multi-device]
    └── SpmdPartitioner ─────────── insert collectives per HloSharding
    │
    ▼
Backend-specific lowering
    ├── CPU: IrEmitter → LLVM IR → LLVM backend
    └── GPU: NvptxIrEmitter / TritonFusion → PTX → ptxas
    │
    ▼
Executable (XlaExecutable / GpuExecutable)
```

### 153.4.1 AlgebraicSimplifier

`AlgebraicSimplifier` is XLA's equivalent of LLVM's instcombine. It implements rewrites such as:

- `transpose(transpose(x, perm), inverse(perm)) → x`
- `broadcast(scalar) + broadcast(scalar) → broadcast(scalar + scalar)`
- `dot(reshape(x), y) → dot(x, reshape_if_needed(y))` when shapes permit
- `slice(pad(x)) → pad(slice(x))` when the slice avoids the padding region

These rewrites run to a fixed point and dramatically simplify the graph before layout assignment.

### 153.4.2 Layout Assignment

Choosing row-major vs column-major layout per operand has major performance implications. `LayoutAssignment` propagates layout constraints through the graph: backends declare their preferred layouts for specific operations (e.g., cuBLAS wants column-major for `dot`), and the pass inserts `copy` instructions where layouts must be converted.

### 153.4.3 Fusion

XLA's fusion model is one of its most significant contributions to ML performance. The `InstructionFusion` pass identifies sequences of element-wise operations and fuses them into a single **`kFusion` HLO instruction** that contains a sub-computation. The fused computation is then emitted as a single GPU kernel, eliminating intermediate memory allocations.

Fusion categories:

| Category | Description |
|----------|-------------|
| `kLoop` | Element-wise map over matching shapes |
| `kInput` | Reduction fusion: producer is fused into the reduction |
| `kOutput` | Elements of the output are computed by a fused producer |
| `kCustom` | Manually specified (e.g., Triton GEMM) |

---

## 153.5 Buffer Assignment

After optimization, XLA must assign physical memory buffers to logical HLO values. `BufferAssignment` performs liveness analysis, computes buffer sizes, and identifies safe **aliasing** opportunities — cases where an output buffer can reuse an input buffer without a copy.

The `LogicalBuffer` → `BufferAllocation` mapping is exposed in the `BufferAssignmentProto`, which XLA's profiling tools use to report peak memory usage and identify copies.

Key concepts:

- **Live range**: the interval between an instruction's execution and its last use. Buffers with non-overlapping live ranges can share the same allocation.
- **In-place operations**: ops like `DynamicUpdateSlice` are annotated as in-place — they update a slice of a buffer without copying the whole tensor.
- **Parameter aliasing**: input parameters can alias output buffers when the calling convention permits (JAX's `donate_argnums`).

```cpp
// From xla/service/buffer_assignment.h
class BufferAssignment {
  // For each HloValue, which BufferAllocation holds it?
  absl::flat_hash_map<const HloValue*, BufferAllocation::Slice> allocation_map_;
  
  // All allocations, sorted by size for best-fit packing
  std::vector<BufferAllocation> allocations_;
};
```

---

## 153.6 Ahead-of-Time and JIT Compilation

XLA supports two compilation modes:

### 153.6.1 JIT via LocalClient

The `LocalClient` API compiles and executes on the local machine:

```cpp
xla::LocalClient* client = xla::ClientLibrary::GetOrCreateLocalClient().value();

// Compile
xla::ExecutableBuildOptions opts;
auto exec = client->Compile(computation, {shape0, shape1}, opts).value();

// Execute
std::vector<xla::ScopedShapedBuffer> args = ...;
auto result = exec[0]->Run(args, xla::ExecutableRunOptions()).value();
```

### 153.6.2 AOT via AotCompilationResult

XLA can serialize compiled executables for ahead-of-time deployment:

```cpp
// Compile to AOT binary
std::vector<std::unique_ptr<xla::AotCompilationResult>> aot_results;
auto compiler = xla::Compiler::GetForPlatform(platform).value();
aot_results = compiler->CompileAheadOfTime({module}, opts).value();

// Serialize to bytes for embedding in a binary
std::string serialized = aot_results[0]->SerializeAsString().value();
```

The AOT path is used by TensorFlow Serving and by JAX's `jax.export` + `jax.load` round-trip workflow.

---

## 153.7 XLA and MLIR Integration

XLA predates MLIR but has been progressively integrating MLIR infrastructure. The integration strategy:

### 153.7.1 StableHLO as the Front Door

StableHLO (Chapter 154) provides a stable, versioned MLIR dialect for XLA's input. Clients (JAX, TF, torch-mlir) emit StableHLO MLIR, which XLA then converts to its internal HLO representation via `ConvertStableHloToHlo`:

```cpp
// xla/mlir_hlo/stablehlo/transforms/stablehlo_legalize_to_hlo.cc
mlir::LogicalResult ConvertStableHloToHlo(
    mlir::ModuleOp mlir_module,
    xla::HloModuleProto* hlo_module_proto);
```

### 153.7.2 MHLO (Deprecated)

MHLO (Meta HLO) was XLA's first MLIR dialect. It provided MLIR representations of XLA ops but lacked a versioning guarantee. MHLO is now deprecated in favor of StableHLO. The codebase still contains MHLO for backward compatibility but new code should target StableHLO.

### 153.7.3 The Linalg / MLIR Codegen Path

For some backends, XLA is experimenting with routing through MLIR's Linalg dialect rather than its legacy `IrEmitter`. The `XlaGpuToLinalg` transformation converts fused HLO computations to `linalg.generic` operations, enabling MLIR's vectorization and tiling passes. This path is not yet the default but is the direction for future GPU codegen.

---

## 153.8 HloModuleConfig and Compilation Options

`HloModuleConfig` carries per-compilation settings that affect optimization decisions:

```cpp
struct HloModuleConfig {
  int64_t seed;                    // for RNG reproducibility
  bool debug_options;              // enable extra verification
  ShardingConfig sharding_config;  // SPMD sharding constraints
  FusionConfigCollection fusion_config;  // per-op fusion hints
  std::optional<int> device_memory_size;  // override available memory
  bool use_spmd_partitioning;
  int num_partitions;              // number of SPMD partitions
  int replica_count;               // data-parallel replicas
};
```

The `debug_options` field (a `DebugOptions` proto) enables a large number of diagnostic flags: `xla_dump_hlo_as_text`, `xla_dump_to` (directory), `xla_enable_hlo_passes_only` (whitelist of passes to run), `xla_disable_hlo_passes` (passes to skip).

---

## 153.9 Profiling and Debugging XLA Programs

XLA integrates with multiple profiling tools:

### 153.9.1 HLO Dumps

Setting `XLA_FLAGS=--xla_dump_to=/tmp/xla_dump` causes XLA to write the HLO module before and after each pass. The text format is human-readable:

```
HloModule my_module

ENTRY %main (p0: f32[4,4], p1: f32[4,4]) -> f32[4,4] {
  %p0 = f32[4,4]{1,0} parameter(0)
  %p1 = f32[4,4]{1,0} parameter(1)
  %dot.0 = f32[4,4]{1,0} dot(%p0, %p1),
      lhs_contracting_dims={1}, rhs_contracting_dims={0}
  ROOT %relu = f32[4,4]{1,0} maximum(%dot.0, %broadcast.0)
}
```

### 153.9.2 XProf and TensorBoard Integration

XLA instruments kernel launches with CUPTI/ROCTRACER events, which are collected by XProf (TensorFlow's profiler) and displayed in TensorBoard's trace viewer. Each kernel shows its HLO fusion name, shape, and flop count.

### 153.9.3 Cost Analysis

`HloCostAnalysis` estimates compute and memory costs per instruction:

```cpp
HloCostAnalysis analysis;
entry_computation->Accept(&analysis);
int64_t flops = analysis.flop_count(*dot_instruction);
int64_t bytes = analysis.bytes_accessed(*dot_instruction);
```

This cost model drives fusion heuristics and auto-sharding's ILP formulation (Chapter 158).

---

## 153.10 XLA vs MLIR: Complementary Roles

A common question is: "If MLIR exists, why does XLA not simply use MLIR as its IR?" The answer is historical and pragmatic:

1. **XLA was built before MLIR** (2016 vs 2019). Rewriting the entire pass pipeline in MLIR terms would be a multi-year effort.
2. **HLO has specialized semantics** that MLIR's generic framework doesn't capture out of the box — particularly shape inference, layout assignment, and the fusion model.
3. **MLIR is now used where it provides clear value** — primarily at the input layer (StableHLO) and in new codegen experiments (Linalg GPU path). The integration is growing.

The practical picture: MLIR is the compile-time representation for the frontend contract (StableHLO), XLA HLO handles high-level ML optimization, and LLVM IR handles target-specific lowering.

---

## Chapter Summary

- XLA is Google's ML compilation framework; OpenXLA is the open-source project hosted at github.com/openxla/xla.
- The IR hierarchy: `HloModule` → `HloComputation` → `HloInstruction`; shapes carry both type and layout (minor-to-major ordering).
- Key optimization passes: `AlgebraicSimplifier`, `LayoutAssignment`, `InstructionFusion`, `BufferAssignment`.
- XLA's fusion model eliminates intermediate memory allocations by merging element-wise ops into single kernels.
- `BufferAssignment` performs liveness analysis and identifies aliasing opportunities to minimize peak memory.
- `LocalClient` provides JIT execution; `AotCompilationResult` supports ahead-of-time compilation.
- MLIR integration is through StableHLO at the input level; MHLO is deprecated; a Linalg codegen path is emerging for GPU.
- `HloModuleConfig` and `DebugOptions` control optimization and diagnostic behavior; `XLA_FLAGS` environment variable is the common diagnostic entry point.


---

@copyright jreuben11
