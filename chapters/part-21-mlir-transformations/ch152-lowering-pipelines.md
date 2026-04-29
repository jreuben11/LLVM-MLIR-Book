# Chapter 152 — Lowering Pipelines

*Part XXI — MLIR Transformations*

No single MLIR pass transforms a high-level tensor computation into machine code. Lowering is always a sequence: bufferization converts tensors to buffers; loop generators unfold array operations into iteration; vector lowering maps abstract vector operations to hardware-width SIMD; dialect conversion translates the result to LLVM IR; and finally `mlir-translate` emits the LLVM module for `llc` or `lld`. Understanding how these stages compose, why their ordering matters, and how to debug breakdowns at each stage is the practical knowledge that separates working compilers from textbook examples. This chapter presents complete, runnable pipelines for CPU, GPU, and production ML compilers, then addresses the diagnostic tools for when things go wrong.

---

## 152.1 Pipeline Architecture

### The composition model

MLIR lowering pipelines are compositions of passes assembled in a `PassManager`. Each pass has a single responsibility:

- Some passes are **analysis passes**: they compute information without modifying IR (`--canonicalize` sometimes, `--cse`, analysis caching).
- Some passes are **transformation passes**: they apply rewrites to the IR (`--linalg-vectorize`, `--one-shot-bufferize`).
- Some passes are **conversion passes**: they translate one dialect to another (`--convert-linalg-to-loops`, `--convert-arith-to-llvm`).

A complete pipeline is a sequence of these three types, organized so that:
1. High-level abstractions are simplified before being lowered (canonicalize early and often).
2. Each lowering step targets an intermediate dialect that is fully representable and testable in isolation.
3. The final conversion step produces LLVM IR, after which `mlir-translate` hands off to the LLVM backend.

### Standard intermediate dialects

| Level | Dialects | Purpose |
|-------|----------|---------|
| High-level | `linalg`, `tensor`, `tosa`, `stablehlo` | Algorithm-level; value semantics |
| Loop-level | `scf`, `affine` | Explicit iteration; still structured |
| Low-level | `memref`, `arith`, `math`, `vector` | Buffer semantics; hardware-neutral |
| Target | `llvm`, `nvvm`, `rocdl` | LLVM IR in MLIR form |

---

## 152.2 Linalg-to-LLVM CPU Pipeline

The canonical CPU lowering pipeline for a program expressed in `linalg` + `tensor`:

```bash
#!/bin/bash
MLIR_OPT=/usr/lib/llvm-22/bin/mlir-opt
MLIR_TRANSLATE=/usr/lib/llvm-22/bin/mlir-translate
LLC=/usr/lib/llvm-22/bin/llc

$MLIR_OPT input.mlir \
  \
  # --- Phase 1: High-level cleanup ---
  --canonicalize \
  --cse \
  \
  # --- Phase 2: Bufferization ---
  --one-shot-bufferize="bufferize-function-boundaries=true \
                        function-boundary-type-conversion=infer-layout-map" \
  --buffer-deallocation-pipeline \
  \
  # --- Phase 3: Linalg on buffers → loops ---
  --convert-linalg-to-loops \
  \
  # --- Phase 4: Affine/SCF cleanup and lowering ---
  --lower-affine \
  --convert-scf-to-cf \
  --canonicalize \
  --cse \
  \
  # --- Phase 5: Dialect-to-LLVM conversions ---
  --convert-arith-to-llvm \
  --convert-math-to-llvm \
  --convert-memref-to-llvm \
  --convert-func-to-llvm \
  --convert-cf-to-llvm \
  \
  # --- Phase 6: Cleanup unrealized casts ---
  --reconcile-unrealized-casts \
  -o lowered.mlir

# Translate MLIR LLVM dialect → LLVM IR:
$MLIR_TRANSLATE --mlir-to-llvmir lowered.mlir -o output.ll

# Compile to object file:
$LLC -O3 -filetype=obj output.ll -o output.o
```

### Why each step is ordered as it is

**Canonicalize before bufferization**: Simplifying redundant `tensor.cast`, `tensor.dim`, and identity `linalg` operations before bufferization reduces the number of buffers that must be allocated and improves in-place analysis accuracy.

**Buffer deallocation after bufferization, before loop lowering**: The ownership-based deallocation algorithm works at the memref level. After loop lowering, ownership tracking becomes more complex (allocations inside loops). Deallocate while the structure is still clear.

**Affine lowering before SCF-to-CF**: `--lower-affine` converts `affine.for` / `affine.if` to `scf.for` / `scf.if`. `--convert-scf-to-cf` then converts `scf.*` to unstructured control flow (`cf.br`, `cf.cond_br`). They must run in this order.

**All dialect-to-LLVM conversions before `reconcile-unrealized-casts`**: Each conversion pass introduces `unrealized_conversion_cast` ops at type boundaries. Only after all conversions have run can the reconciliation pass determine which casts are trivial.

---

## 152.3 Linalg Vectorization Pipeline

Vectorization occurs between bufferization and loop lowering. Insert these steps after `--buffer-deallocation-pipeline` and before `--convert-linalg-to-loops`:

```bash
$MLIR_OPT bufferized.mlir \
  \
  # --- Normalize named ops to linalg.generic ---
  --linalg-generalize-named-ops \
  \
  # --- Fold unit-extent dimensions (enables vectorization of degenerate dims) ---
  --linalg-fold-unit-extent-dims \
  \
  # --- Vectorize linalg.generic → vector.transfer_read/write + vector.contract ---
  --linalg-vectorize-named-ops \
  --canonicalize \
  \
  # --- Multi-dimensional reduction lowering ---
  --vector-multi-reduction-lowering \
  \
  # --- Lower vector.contract to a sequence of vector.fma / vector.outerproduct ---
  --vector-contraction-lowering="vector-product-lowering" \
  \
  # --- Lower vector.transfer_read/write to vector.load/store with masking ---
  --vector-transfer-lowering \
  \
  # --- Unroll to the native SIMD width (e.g., 256 bits for AVX2) ---
  --vector-unroll="target-vector-bitwidth=256" \
  \
  # --- Map vector ops to LLVM intrinsics ---
  --lower-vector-to-llvm \
  \
  # Continue with standard loop and dialect lowerings...
  --convert-linalg-to-loops \
  --lower-affine \
  --convert-scf-to-cf \
  --convert-arith-to-llvm \
  --convert-math-to-llvm \
  --convert-memref-to-llvm \
  --convert-func-to-llvm \
  --convert-cf-to-llvm \
  --reconcile-unrealized-casts \
  -o vectorized.mlir
```

### Vectorization ordering

The pass ordering for vector lowering is not arbitrary:

1. `--vector-multi-reduction-lowering` must precede `--vector-contraction-lowering` because contractions may contain reductions.
2. `--vector-contraction-lowering` must precede `--vector-transfer-lowering` because transfer ops implement the memory interface for contractions.
3. `--vector-unroll` must precede `--lower-vector-to-llvm` because LLVM has fixed native vector widths; unrolling first avoids generating over-wide vectors that the backend must split.

### Masking and out-of-bounds handling

For non-multiple sizes, vector operations require masking. MLIR handles this at the `vector.transfer_read/write` level with an `in_bounds` attribute:

```mlir
// Masked read (may access out-of-bounds, use padding value):
%v = vector.transfer_read %buf[%i, %j], %pad
    : memref<?x?xf32>, vector<8xf32>

// In-bounds read (caller guarantees no OOB):
%v = vector.transfer_read %buf[%i, %j], %pad {in_bounds = [true]}
    : memref<?x?xf32>, vector<8xf32>
```

The `in_bounds` attribute is set by the vectorizer based on tile size divisibility analysis. Ops with `in_bounds = [true]` lower to direct `llvm.load` + `llvm.vector.insert`; masked ops lower to `llvm.masked.load`.

---

## 152.4 GPU Kernel Pipeline (CUDA/NVPTX)

GPU compilation requires a separate pipeline for the device (kernel) code. The host/device split happens at the `gpu.launch` boundary.

### GPU module extraction and lowering

```bash
# Step 1: Run GPU-specific transformations on the MLIR level.
$MLIR_OPT input_with_gpu.mlir \
  --gpu-kernel-outlining \        # Extract gpu.launch body → gpu.func
  --canonicalize \
  \
  # Step 2: Lower GPU ops to NVVM dialect (CUDA-specific).
  --gpu-to-nvvm \                 # gpu.thread_id → nvvm.read.ptx.sreg.tid.x
  --convert-nvgpu-to-nvvm \       # nvgpu.mma.sync → nvvm.mma.sync
  \
  # Step 3: Standard arith/math/cf/func lowerings.
  --convert-arith-to-llvm \
  --convert-math-to-llvm="enable-approximate-log-exp-functions=false" \
  --convert-func-to-llvm \
  --convert-cf-to-llvm \
  --convert-memref-to-llvm \
  --reconcile-unrealized-casts \
  -o gpu_lowered.mlir

# Step 4: Translate to LLVM IR.
$MLIR_TRANSLATE --mlir-to-llvmir gpu_lowered.mlir -o gpu.ll

# Step 5: Compile to PTX.
$LLC -march=nvptx64 -mcpu=sm_90 -mattr=+ptx80 \
     -O3 -filetype=asm gpu.ll -o kernel.ptx
```

### NVGPU intrinsics (Tensor Core)

For Tensor Core MMA operations (WMMA and newer wgmma):

```mlir
// Using nvgpu dialect (before NVVM lowering):
%d = nvgpu.mma.sync (%a[%i, %k], %b[%k, %j], %c[%i, %j])
    {mmaShape = [16, 8, 8], tf32Enabled = false}
    : (memref<16x8xf16, 3>, memref<8x8xf16, 3>, memref<16x8xf32, 3>)
    -> memref<16x8xf32, 3>
```

After `--convert-nvgpu-to-nvvm`, this becomes an `nvvm.mma.sync` with explicit register-level operands that LLVM translates to PTX `wmma.mma.sync` instructions.

### Host-side pipeline

The host side (CPU code managing kernel launches) uses a different pipeline:

```bash
$MLIR_OPT host.mlir \
  --gpu-module-to-binary="format=fatbin" \   # embed PTX/CUBIN in host module
  --convert-gpu-launch-to-llvm \              # gpu.launch_func → cuLaunchKernel
  --convert-arith-to-llvm \
  --convert-func-to-llvm \
  --reconcile-unrealized-casts \
  -o host_lowered.mlir
$MLIR_TRANSLATE --mlir-to-llvmir host_lowered.mlir -o host.ll
```

---

## 152.5 The Affine Polyhedral Pipeline

For programs in the `affine` dialect with known loop bounds, MLIR's polyhedral passes provide loop-level optimizations before lowering:

```bash
$MLIR_OPT affine_program.mlir \
  \
  # --- Polyhedral-level optimizations (operate on affine maps) ---
  --affine-loop-tile="tile-size=32" \       # Tile all loops by 32
  --affine-loop-unroll="unroll-full" \      # Fully unroll inner loop
  --affine-loop-fusion \                     # Fuse adjacent loops sharing array accesses
  --affine-loop-invariant-code-motion \     # Hoist loop-invariant ops
  --affine-parallelize \                    # Mark parallel loops → scf.parallel
  --canonicalize \
  \
  # --- Lower affine to scf ---
  --lower-affine \
  \
  # --- Parallelize scf.parallel → async regions or OpenMP threads ---
  --convert-scf-to-openmp \                 # (if OpenMP target)
  \
  --convert-scf-to-cf \
  --convert-arith-to-llvm \
  --convert-func-to-llvm \
  --convert-cf-to-llvm \
  --convert-memref-to-llvm \
  --convert-openmp-to-llvm \                # (if OpenMP)
  --reconcile-unrealized-casts \
  -o optimized.mlir
```

### Affine loop fusion

`--affine-loop-fusion` merges adjacent loops that access the same arrays. The analysis uses exact polyhedral dependence testing (via ISL, integrated in MLIR's affine analysis infrastructure) to verify legality. The default fusion strategy is `greedy`; `--affine-loop-fusion="fusion-maximal"` enables more aggressive fusion at the cost of increased register pressure.

---

## 152.6 The IREE Pipeline Overview

IREE (Intermediate Representation Execution Environment) is the most complete MLIR-based deployment compiler. Understanding its pipeline structure illuminates the multi-stage approach required for production ML compilation.

### Input dialects

IREE accepts:
- **StableHLO** (`--iree-import-stablehlo`): from JAX/XLA.
- **TOSA** (`--iree-import-tosa`): from TFLite, ONNX-MLIR.
- **torch-mlir** (`--iree-import-torch`): from PyTorch via torch-mlir.
- **Linalg on tensors**: directly, for custom compilers.

### IREE pipeline stages

```
Input (StableHLO/TOSA/Torch)
  ↓ [iree-flow passes]
  Flow dispatch regions (work-item partitioning)
  ↓ [iree-stream passes]
  Stream commands (async execution graph, HAL memory model)
  ↓ [iree-hal passes]
  HAL executables (per-backend kernel dispatch)
  ↓ [iree-vm passes]
  VM bytecode (host-side orchestration runtime)
  ↓
Serialized .vmfb flatbuffer (portable deployment artifact)
```

Each stage is a set of MLIR passes. The `flow` layer fuses linalg ops into dispatch regions. The `stream` layer analyzes data flow and generates an asynchronous command buffer. The `hal` layer compiles each dispatch region to the target backend (Vulkan SPIR-V, CUDA PTX, LLVM CPU).

### CPU backend (IREE-codegen)

The IREE CPU backend uses the same linalg vectorization techniques described in §152.3, but with a more sophisticated tiling strategy controlled by configuration attributes attached to dispatch regions:

```mlir
// IREE sets this attribute on the function before tiling:
#config = #iree_codegen.lowering_config<tile_sizes = [[64, 64], [8, 8], [0, 0]]>

func.func @matmul(%A: tensor<256x256xf32>, ...) 
    attributes {lowering_config = #config} {
```

The attribute drives the linalg tiling pass to apply two levels of tiling (L2 cache tiles, then register tiles) before vectorization.

---

## 152.7 Canonicalization Between Passes

Canonicalization is not a luxury—it is a correctness and performance necessity between phases.

### Why canonicalize between phases

After tiling, the IR contains `affine.min` ops bounding trip counts, `tensor.dim` ops, and dimension arguments that are often statically equal. If these are not folded before vectorization, the vectorizer cannot determine that loops are full-tile (it sees variable bounds instead of constants) and generates masked instead of unmasked vector loads.

After each major pass, run:

```bash
--canonicalize --cse
```

`--cse` (Common Subexpression Elimination) is especially important: tiling often creates multiple `tensor.dim` ops for the same tensor, and CSE eliminates the duplicates that would otherwise cause redundant load/store pairs.

### Loop invariant code motion

```bash
--loop-invariant-code-motion  # for scf.for loops
--affine-loop-invariant-code-motion  # for affine.for loops
```

These passes hoist computations that do not depend on the loop induction variable. This includes address computations, constants derived from function arguments, and expensive operations like `sqrtf` called with loop-invariant operands.

### Folding order example

```mlir
// After tiling with tile_sizes [64]:
%trip = affine.min affine_map<(d0)[s0] -> (64, s0 - d0)>(%i)[%N]
%result = vector.transfer_read %buf[%i], %pad {in_bounds = [false]}
    : memref<?xf32>, vector<64xf32>

// After canonicalize (if N is a multiple of 64, affine.min → 64):
%result = vector.transfer_read %buf[%i], %pad {in_bounds = [true]}
    : memref<?xf32>, vector<64xf32>
```

The `in_bounds = [true]` version generates a direct aligned load; `[false]` generates a masked predicated load. The difference can be 2-3x in throughput on modern hardware.

---

## 152.8 Debugging a Lowering Pipeline

### IR printing flags

```bash
# Print IR after every pass (very verbose):
$MLIR_OPT --mlir-print-ir-after-all input.mlir

# Print IR only when a pass actually modifies the IR:
$MLIR_OPT --mlir-print-ir-after-change input.mlir

# Print the enclosing module (not just the func.func being processed):
$MLIR_OPT --mlir-print-ir-module-scope --mlir-print-ir-after-all input.mlir

# Print IR before the failing pass (on failure):
$MLIR_OPT --mlir-pass-failure-dump-pass-pipeline input.mlir

# Time each pass:
$MLIR_OPT --mlir-timing --mlir-timing-display=tree input.mlir
```

### mlir-reduce: delta debugging

When a pipeline fails on a large MLIR file, `mlir-reduce` automates delta debugging to find the minimal reproducer:

```bash
# Create an interestingness script:
cat > interesting.sh << 'EOF'
#!/bin/bash
/usr/lib/llvm-22/bin/mlir-opt "$1" \
  --one-shot-bufferize --convert-linalg-to-loops 2>&1 | \
  grep -q "error"
EOF
chmod +x interesting.sh

# Run mlir-reduce:
/usr/lib/llvm-22/bin/mlir-reduce \
  --test=interesting.sh \
  original.mlir \
  -o reduced.mlir
```

`mlir-reduce` applies a series of reduction strategies (removing ops, simplifying types, removing attributes) until the smallest IR that still triggers the error is found.

### Isolating a failing pass

To check whether a specific pass is responsible:

```bash
# Bisect: run only up to the suspicious pass:
$MLIR_OPT \
  --one-shot-bufferize \
  --buffer-deallocation-pipeline \
  input.mlir | \
$MLIR_OPT \
  --convert-linalg-to-loops 2>&1  # does this fail?
```

Splitting the pipeline at a stable intermediate dialect makes it possible to save the intermediate `.mlir` and experiment with the remainder independently.

### Verifying IR between passes

Add `--mlir-verify-each` to run the MLIR verifier after every pass:

```bash
$MLIR_OPT --mlir-verify-each --canonicalize --one-shot-bufferize input.mlir
```

If a pass introduces an invalid IR state (e.g., a type mismatch), the verifier catches it immediately after that pass rather than producing a cryptic error several passes later.

### Common pipeline errors and fixes

| Error | Likely cause | Fix |
|-------|-------------|-----|
| `unrealized_conversion_cast` remains | Missing conversion pattern for some op | Add the appropriate `populate*` patterns or convert the op earlier |
| `failed to legalize operation 'dialect.op'` | Op not covered by any conversion pattern | Add a pattern or mark the op legal in the ConversionTarget |
| `failed to bufferize op` | Op does not implement `BufferizableOpInterface` | Register external model or pre-lower the op |
| `use of value defined as non-constant` | affine.apply / affine.for uses non-SSA-dominating value | Reorder ops or use `scf.for` instead |
| Verifier error: "operand type mismatch" | `--reconcile-unrealized-casts` not run after conversion | Add `--reconcile-unrealized-casts` at end of pipeline |
| All zeros in output | Buffer not initialized before use | Check deallocation pipeline; ensure `linalg.fill` is not removed |

---

## 152.9 Building a Pipeline in C++

For tools that construct the pipeline programmatically (rather than using CLI flags):

```cpp
#include "mlir/Pass/PassManager.h"
#include "mlir/Transforms/Passes.h"
#include "mlir/Dialect/Bufferization/Transforms/Passes.h"
#include "mlir/Dialect/Linalg/Passes.h"
#include "mlir/Conversion/Passes.h"

mlir::MLIRContext ctx;
ctx.loadDialect<mlir::func::FuncDialect,
                mlir::arith::ArithDialect,
                mlir::linalg::LinalgDialect,
                mlir::tensor::TensorDialect,
                mlir::memref::MemRefDialect,
                mlir::LLVM::LLVMDialect>();

mlir::PassManager pm(&ctx);
pm.enableMultithreading();

// Phase 1: cleanup
pm.addPass(mlir::createCanonicalizerPass());
pm.addPass(mlir::createCSEPass());

// Phase 2: bufferization
mlir::bufferization::OneShotBufferizationOptions bufOpts;
bufOpts.bufferizeFunctionBoundaries = true;
pm.addPass(mlir::bufferization::createOneShotBufferizePass(bufOpts));
mlir::bufferization::buildBufferDeallocationPipeline(pm,
    mlir::bufferization::BufferDeallocationPipelineOptions{});

// Phase 3: linalg → loops
pm.nest<mlir::func::FuncOp>().addPass(
    mlir::createConvertLinalgToLoopsPass());

// Phase 4: affine/scf cleanup
pm.addPass(mlir::createLowerAffinePass());
pm.addPass(mlir::createConvertSCFToCFPass());
pm.addPass(mlir::createCanonicalizerPass());

// Phase 5: dialect-to-LLVM
pm.addPass(mlir::createConvertArithToLLVMPass());
pm.addPass(mlir::createConvertMathToLLVMPass());
mlir::ConvertMemRefToLLVMPassOptions memrefOpts;
pm.addPass(mlir::createConvertMemRefToLLVMPass(memrefOpts));
mlir::ConvertFuncToLLVMPassOptions funcOpts;
pm.addPass(mlir::createConvertFuncToLLVMPass(funcOpts));
pm.addPass(mlir::createConvertControlFlowToLLVMPass());
pm.addPass(mlir::createReconcileUnrealizedCastsPass());

// Run:
if (mlir::failed(pm.run(module))) {
  llvm::errs() << "Pipeline failed\n";
  return 1;
}
```

---

## Chapter Summary

- **MLIR lowering is a composition** of canonicalization, bufferization, loop generation, vectorization, and dialect-to-LLVM conversion passes; no single pass performs the full transformation.
- The **standard CPU pipeline** is: canonicalize → one-shot-bufferize → buffer-deallocation-pipeline → convert-linalg-to-loops → lower-affine → convert-scf-to-cf → dialect-to-LLVM conversions → reconcile-unrealized-casts.
- **Vectorization** inserts between bufferization and loop lowering: generalize → vectorize → multi-reduction-lowering → contraction-lowering → transfer-lowering → unroll → lower-to-llvm.
- **GPU pipelines** use `--gpu-to-nvvm` / `--convert-nvgpu-to-nvvm` before the standard dialect-to-LLVM conversions; `mlir-translate --mlir-to-llvmir` then feeds `llc -march=nvptx64`.
- **Canonicalize + CSE between phases** is not optional: without it, symbolic bounds remain unfolded, preventing vectorization from generating `in_bounds = [true]` loads.
- The **IREE pipeline** demonstrates industrial-scale multi-stage lowering: StableHLO/TOSA → flow dispatches → stream commands → HAL executables → backend kernels, each stage expressed as MLIR passes.
- **Debugging tools**: `--mlir-print-ir-after-change`, `--mlir-verify-each`, `--mlir-pass-failure-dump-pass-pipeline`, and `mlir-reduce` for delta debugging to minimal reproducers.
- **C++ pipeline construction** uses `PassManager::addPass`, `pm.nest<T>()`, and typed pass creation functions from the `create*Pass` family; `--mlir-timing-display=tree` profiles pass execution overhead.
