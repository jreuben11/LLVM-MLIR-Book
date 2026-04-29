# Chapter 129 — MLIR Philosophy

*Part XIX — MLIR Foundations*

By 2019, the compiler infrastructure landscape had splintered. TensorFlow had XLA HLO. PyTorch had TorchScript. Intel had nGraph. ONNX provided an interchange format but not a compilation infrastructure. Every ML framework was building the same pieces — graph IR, pattern rewriting, lowering to CPU/GPU primitives — from scratch and in isolation. LLVM IR, the natural terminus for all these systems, was too low-level to retain the high-level semantic information that enables whole-program optimizations like kernel fusion and memory layout selection. MLIR — Multi-Level Intermediate Representation — was designed to end this fragmentation by providing shared infrastructure that operates at any level of abstraction. This chapter explains the design principles that make MLIR different from every prior compiler IR.

---

## 129.1 The Problem MLIR Solves

### Fragmentation

Before MLIR, a TensorFlow model had to traverse a remarkable number of IR translations:

```
TensorFlow GraphDef
    → XLA HLO (TF2XLA)
    → XLA LLO (scheduling, tiling)
    → LLVM IR (cpu/gpu/tpu backends)
    → machine code
```

Each arrow represented a translation between completely separate systems with different type systems, different optimization passes, different debugging infrastructure, and different extension points. TVM, Glow, and MXNet each had analogous but incompatible pipelines. None of these systems could share optimizations with each other, with Clang, or with GCC.

### The Abstraction Gap

LLVM IR excels at machine-level optimization but cannot represent a matrix multiplication as a single operation — it is already exploded into nested loops, loads, stores, and scalar arithmetic. By the time a neural network operation reaches LLVM IR, the compiler has lost the information that `%result = compute_matmul(...)` is a GEMM. The vectorizer must rediscover this from loop structure; the GPU code generator must re-identify tiling opportunities. Every transformation that could have been done at the graph level must now be approximated at the loop level.

### MLIR's Approach

MLIR solves both problems simultaneously:

1. **Shared infrastructure**: one IR format, one verifier framework, one pass manager, one pattern rewriting engine, one textual format, one bytecode — shared by all users
2. **Open type/operation system**: users add their own types, attributes, and operations (called *dialects*) without forking the compiler
3. **Multi-level by design**: the same program can contain `tosa.conv2d` (graph-level), `linalg.conv_2d_nhwc_hwcf` (named-structure level), `affine.for` (polyhedral level), and `llvm.call` (machine level) simultaneously; lowering proceeds incrementally

---

## 129.2 The Dialect System

The dialect is MLIR's unit of extensibility. A **dialect** is a namespace-bounded bundle of:

- **Types**: new data types (e.g., `!fir.box<T>`, `!spirv.sampled_image<T>`)
- **Attributes**: compile-time constants (e.g., `#linalg.indexing_maps<...>`)
- **Operations**: the instructions of the dialect (e.g., `linalg.matmul`, `gpu.func`)
- **Interfaces**: protocols that operations implement (e.g., `MemoryEffectOpInterface`)
- **Passes**: transformations scoped to the dialect

Dialects register into an `MLIRContext` and can freely intermix in the same MLIR module. A Flang-produced module contains FIR ops, HLFIR ops, OpenMP ops, and standard `arith`/`func`/`llvm` dialect ops all at once.

### Dialect Comparison with LLVM IR

| Property | LLVM IR | MLIR |
|----------|---------|------|
| Operations | Fixed opcode set (~70 instructions) | Extensible (30+ dialects, 1000+ ops in-tree) |
| Types | Fixed (i1..i64, f32/f64, ptr, struct, array, ...) | Extensible (each dialect adds types) |
| Metadata | Loosely-typed name→value map | Strongly-typed attributes with verification |
| Nesting | Flat function → basic blocks | Recursive: op → region → block → op |
| Multiple levels | No (only machine-adjacent) | Yes (graph → loops → vectors → machine) |
| Extension | Fork the project | Register a dialect |

### In-Tree Dialects

LLVM 22 ships 35+ dialects in-tree:

**Foundational**:
- `builtin`: module, function type, integer/float types
- `func`: function definition, call, return
- `arith`: integer/float arithmetic (type-agnostic)
- `math`: mathematical functions (sqrt, exp, etc.)
- `cf`: unstructured control flow (br, cond_br)

**Memory and Data**:
- `memref`: typed memory references, alloc/dealloc
- `tensor`: value-semantics tensors
- `bufferization`: tensor→memref conversion infrastructure

**Structured Computations**:
- `linalg`: named and generic structured tensor/memref operations
- `affine`: polyhedral loops and memory access patterns
- `scf`: structured control flow (for, if, while)
- `vector`: multi-dimensional SIMD vectors
- `sparse_tensor`: sparse tensor algebra

**Hardware Targets**:
- `gpu`: GPU functions, launches, barriers
- `nvgpu`: NVIDIA-specific GPU operations
- `amdgpu`: AMD-specific GPU operations
- `spirv`: Vulkan/OpenCL SPIR-V
- `llvm`: LLVM IR ops and types
- `x86vector`: x86 vector intrinsics
- `arm_neon`, `arm_sve`: ARM vector extensions

**Compiler-Specific**:
- `omp`: OpenMP parallel constructs
- `acc`: OpenACC accelerator constructs
- `fir`, `hlfir`: Flang's Fortran IR
- `cir`: ClangIR (see [Chapter 52](../part-08-clangir/ch52-clangir-architecture.md))

**ML/XLA**:
- `tosa`: TOSA (Tensor Operator Set Architecture) ML ops
- `stablehlo`: StableHLO (XLA-compatible ML ops)
- `quant`: quantization operations and types

---

## 129.3 Progressive Lowering

The central MLIR usage pattern is **progressive lowering**: starting from a high-level dialect, each transformation pass replaces some ops with equivalent ops from a lower-level dialect, until the entire program is expressed in a target dialect (usually `llvm`).

### A Concrete Lowering Path

```
tosa.conv2d (graph level: semantic op)
    │  tosa-to-linalg pass
    ▼
linalg.conv_2d_nhwc_hwcf (named structure)
    │  linalg-generalize-named-ops pass
    ▼
linalg.generic (generic structured op with indexing maps)
    │  linalg-tile-and-fuse
    ▼
scf.parallel + linalg.generic (tiled; ready for vectorization)
    │  convert-linalg-to-loops
    ▼
scf.for + memref.load + memref.store + arith.mulf/addf (explicit loops)
    │  convert-scf-to-cf
    ▼
cf.br + memref.load + memref.store + arith.* (CFG, unstructured)
    │  convert-memref-to-llvm + convert-arith-to-llvm + convert-cf-to-llvm
    ▼
llvm.* (LLVM dialect; direct translation to LLVM IR)
    │  ConvertMLIRToLLVMIR
    ▼
LLVM IR Module
```

Each step is a separate, composable, testable transformation pass. Any step can be replaced (e.g., substitute a different tiling strategy) without affecting the others.

### Comparison with LLVM IR Lowering

In LLVM, `clang` performs the entire C→LLVM IR translation in a single monolithic pass over the AST. There is no intermediate level at which the compiler retains structured loop information — it is all gone once the IR is emitted. MLIR's progressive lowering lets analysis and transformation happen at each level before information is discarded.

---

## 129.4 Key Design Decisions

### Ops Are Extensible, But Verification Is Mandatory

Every MLIR operation is verified before being processed. Verification checks:
- Operand/result type constraints
- Attribute type/value constraints
- Structural constraints (e.g., terminators must be last in a block)
- Interface-specific invariants

This contrasts with LLVM IR, where metadata and some intrinsics are loosely checked. MLIR's strong verification catches bugs early and makes dialects self-documenting.

### No Implicit Captures

MLIR strongly discourages "implicit captures" — ops inside a region should not silently use values from an outer region (except for explicitly designed cases). The `IsolatedFromAbove` trait enforces this. This makes each region independently compilable and parallelizable.

### Value Semantics for Data

The `tensor` dialect uses **value semantics**: a `tensor<4x4xf32>` is an immutable value, not a mutable memory buffer. This enables compiler transformations without alias analysis — tensors can always be freely copied, CSE'd, and eliminated because there is no aliasing. Materialization to actual memory buffers is handled explicitly by the bufferization passes.

### Regions Replace Flat CFGs

MLIR's `Region` construct — operations that contain lists of blocks — enables structured control flow at the IR level. `scf.for` is an op with a region; the loop body is textually and semantically inside the op. This is more composable than LLVM IR's flat basic block model: passes can walk the region tree, transform the loop body in isolation, and reason about nested scopes without global CFG analysis.

---

## 129.5 Key Papers and History

### Origin

MLIR was conceived by **Chris Lattner** (creator of LLVM and Swift) while at Google, working on TensorFlow's XLA compiler. The core problem was clear: XLA had a good compiler infrastructure but it was tightly coupled to XLA's own types and ops; other frameworks could not reuse it. The solution was a general infrastructure.

Co-designers include:
- **Jacques Pienaar**: TensorFlow dialect, shape inference
- **Nicolas Vasilache**: linalg dialect, polyhedral/affine transformation framework
- **Mehdi Amini**: MLIR bytecode, tooling, release engineering
- **River Riddle**: ODS, PDLL, many core IR features
- **Alex Zinenko**: affine dialect, polyhedral analysis

### Timeline

| Year | Milestone |
|------|-----------|
| 2018 | Internal Google prototype ("MLIF") |
| 2019 | MLIR open-sourced; donated to LLVM Foundation |
| 2020 | Merged into `llvm-project` monorepo |
| 2020 | Published: "MLIR: Scaling Compiler Infrastructure for Domain Specific Computation" (CGO 2021) |
| 2021 | linalg, tensor, bufferization stabilized |
| 2022 | MLIR bytecode format introduced (LLVM 15) |
| 2023 | Python bindings stabilized; MLIR in production at Google, Meta, AMD, NVIDIA |
| 2024 | Properties system replaces attr dict for op-owned data |
| 2025 | In-tree StableHLO; IREE 2.0; torch-mlir mainlined |

### The CGO 2021 Paper

The canonical reference is:

> Chris Lattner et al., "MLIR: Scaling Compiler Infrastructure for Domain Specific Computation," *Proceedings of CGO 2021*. [https://arxiv.org/abs/2002.11054](https://arxiv.org/abs/2002.11054)

The paper introduces the key concepts: operations, types, attributes, dialects, regions, the verifier, and the pass infrastructure. It includes case studies on TensorFlow, MLIR's use in Swift, and the affine analysis framework.

---

## 129.6 The MLIR Ecosystem in 2026

### In-Tree Toolchain

The MLIR toolchain (available at `/usr/lib/llvm-22/bin/`) includes:

| Tool | Purpose |
|------|---------|
| `mlir-opt` | Apply passes to `.mlir` files; the primary test tool |
| `mlir-translate` | Convert between MLIR and other formats (LLVM IR, SPIR-V, etc.) |
| `mlir-reduce` | Delta-debugging: reduce an MLIR file to a minimal reproducer |
| `mlir-lsp-server` | Language server for `.mlir` files (syntax, diagnostics, hover) |
| `mlir-cpu-runner` | JIT-compile and execute MLIR programs |

### Out-of-Tree Projects

The MLIR ecosystem outside the LLVM monorepo has grown dramatically:

- **IREE** (Intermediate Representation Execution Environment): end-to-end ML deployment via MLIR; targets CPU, CUDA, Vulkan, Metal (Chapter 162)
- **torch-mlir**: bridge from PyTorch's TorchScript/FX to MLIR dialects
- **Triton**: NVIDIA's GPU kernel compiler, built on MLIR's `triton.*` dialect
- **Enzyme**: automatic differentiation framework for MLIR/LLVM IR (Chapter 162)
- **CIRCT**: circuit IR built on MLIR for hardware design (RTL synthesis)
- **Flang**: Fortran compiler (Part XVIII)
- **RISE/LIFT**: functional array language targeting MLIR

### Relationship to XLA / StableHLO

XLA HLO is Google's ML compiler graph IR. **StableHLO** is a dialect that implements the XLA HLO op semantics in MLIR, providing a stable, versioned, serializable representation. It is maintained in-tree as of LLVM 22 and serves as the "bytecode for ML models" — the interchange format between ML frameworks (JAX, TF, PyTorch via torch-mlir) and compiler backends (IREE, XLA). See [Chapter 158](../part-22-xla-openxla/ch158-xla-and-openxla.md) for details.

---

## 129.7 Getting Started

### Running mlir-opt

```bash
# Verify and print a textual MLIR file:
/usr/lib/llvm-22/bin/mlir-opt input.mlir

# Apply canonicalization:
mlir-opt --canonicalize input.mlir

# Apply CSE then canonicalize:
mlir-opt --cse --canonicalize input.mlir

# Lower from linalg to LLVM:
mlir-opt \
  --convert-linalg-to-loops \
  --convert-scf-to-cf \
  --convert-cf-to-llvm \
  --convert-arith-to-llvm \
  --convert-func-to-llvm \
  --reconcile-unrealized-casts \
  input.mlir -o output.mlir

# Translate MLIR with LLVM dialect to LLVM IR:
mlir-translate --mlir-to-llvmir output.mlir -o output.ll
```

### A Minimal MLIR Program

```mlir
// hello.mlir: a function that adds two integers
module {
  func.func @add(%a: i32, %b: i32) -> i32 {
    %sum = arith.addi %a, %b : i32
    return %sum : i32
  }
}
```

```bash
# Verify the module is well-formed:
mlir-opt hello.mlir

# Lower to LLVM IR:
mlir-opt \
  --convert-arith-to-llvm \
  --convert-func-to-llvm \
  hello.mlir | \
mlir-translate --mlir-to-llvmir - | \
llc -filetype=obj -o hello.o
```

---

## Chapter Summary

- MLIR was created to end compiler infrastructure fragmentation in the ML ecosystem by providing a shared, extensible IR that operates at any level of abstraction
- The core extensibility mechanism is the **dialect**: a namespace-bounded bundle of types, attributes, operations, interfaces, and passes; dialects can intermix freely in a single module
- LLVM IR's closed type/opcode system prevents representing high-level semantics; MLIR's open system allows `tosa.conv2d`, `linalg.matmul`, `affine.for`, and `llvm.call` to coexist in a single module during progressive lowering
- Progressive lowering is the canonical usage pattern: each pass replaces ops from a higher-level dialect with ops from a lower-level dialect, preserving and transforming semantic information incrementally
- Key design decisions — mandatory verification, no implicit captures, value semantics for tensors, structured regions — distinguish MLIR from prior IRs and enable the composability that makes it practical
- MLIR originated at Google in 2018, was open-sourced in 2019, and merged into the LLVM monorepo in 2020; the CGO 2021 paper "MLIR: Scaling Compiler Infrastructure for Domain Specific Computation" is the canonical reference
- The ecosystem spans Flang (Fortran), IREE (ML deployment), torch-mlir (PyTorch), Triton (GPU kernels), CIRCT (hardware design), and StableHLO (ML model interchange)


---

@copyright jreuben11
