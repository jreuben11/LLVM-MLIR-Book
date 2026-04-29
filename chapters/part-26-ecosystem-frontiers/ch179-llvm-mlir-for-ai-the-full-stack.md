# Chapter 179 — LLVM/MLIR for AI: The Full Stack

*Part XXVI — Ecosystem and Frontiers*

Parts XIX through XXIII each examined a layer of the AI compilation stack in isolation: MLIR foundations, the in-tree dialects that carry tensor computations, the transformation machinery that rewrites them, XLA and StableHLO, and the production compilers — IREE, Triton, torch-mlir, ONNX-MLIR — that target real hardware. This chapter synthesises those pieces. It shows how the layers connect, traces how data-dependent decisions propagate from the framework down to PTX instructions, and covers two topics that demand cross-layer understanding: quantization (types, formats, and the full lowering chain) and dynamic shapes (where the clean static-shape assumption breaks down). It maps the hardware target landscape from CUDA to edge NPUs, examines what makes FlashAttention a canonical case study for kernel fusion in MLIR, and closes with a step-by-step lowering walkthrough of a single linear layer.

---

## 179.1 The AI Compilation Hierarchy

Every AI compilation stack must answer the same question at four distinct levels: what is being compiled, what information is available, and what transformation is legal.

### 179.1.1 The Four Levels

**Level 1 — Framework eager execution.** PyTorch, JAX, and TensorFlow can all execute operations as Python lines run. The compiler sees nothing; each op dispatches to a hand-tuned library kernel (cuBLAS, cuDNN, NCCL). No cross-op fusion, no shape specialisation beyond what the library provides. This is the baseline against which compiled paths are measured.

**Level 2 — Traced/compiled capture.** The framework traces through user code once and captures a computation graph. PyTorch `torch.export.export()` produces an `ExportedProgram` with an ATen-level FX graph and a list of dynamic-shape constraints. JAX `jax.jit` traces through Python with abstract values (`ShapedArray`) and hands an XLA `StableHLO` module to the compiler. TensorFlow `tf.function` captures `GraphDef` or feeds XLA HLO directly. At this level, whole-program information is available: all shapes (or their symbolic relationships), dataflow graph, memory layout hints.

**Level 3 — Compiler middle layer.** The captured graph enters an MLIR dialect tower (or HLO) for optimisation: constant folding, algebraic simplifications, operator fusion, tiling, and vectorisation. StableHLO serves as the portable handoff format at this level. The key constraint: transformations must preserve tensor semantics exactly — no numerical change is permitted modulo deliberate quantisation or precision relaxation.

**Level 4 — Target code generation.** The tile-and-vectorise-and-bufferise result maps to hardware instructions: LLVM IR for CPU, PTX for NVIDIA, HSA for AMD, SPIR-V for Vulkan. Target-specific rewrites (tensor core MMA intrinsics, cache-control prefetch, scatter/gather primitives) happen here.

### 179.1.2 The Handoff Protocols

```
PyTorch nn.Module
    │ torch.export.export(model, args, dynamic_shapes={...})
    ▼
ExportedProgram (FX graph, ATen ops, ShapeConstraints)
    │ torch_mlir.compile(..., output_type="stablehlo")
    ▼
StableHLO MLIR module
    │
    ├─── IREE iree-compile ──────────► .vmfb flatbuffer (multi-target)
    └─── XLA compiler ───────────────► XLA executable

JAX function
    │ jax.jit → trace with AbstractValues
    ▼
JAX jaxpr
    │ mlir.lower_jaxpr_to_module()
    ▼
StableHLO MLIR module (same pipeline as above)

TensorFlow SavedModel / tf.function
    │ tf2xla bridge (mlir-hlo)
    ▼
MHLO → StableHLO canonicalisation
```

### 179.1.3 Architecture: Framework to Target

```
┌────────────────────────────────────────────────────────────┐
│  ML Frameworks: PyTorch · JAX · TensorFlow · ONNX          │
└─────────────────────────────┬──────────────────────────────┘
                              │ capture (torch.export / jax.jit / tf.function)
┌─────────────────────────────▼──────────────────────────────┐
│  Input dialects: Torch dialect · StableHLO · TOSA · ONNX   │
└─────────────────────────────┬──────────────────────────────┘
                              │ legalize / decompose
┌─────────────────────────────▼──────────────────────────────┐
│  Mid-level MLIR: linalg · vector · tensor · scf · affine   │
└─────────────────────────────┬──────────────────────────────┘
                              │ tile · vectorize · bufferize
┌─────────────────────────────▼──────────────────────────────┐
│  Buffer-level: memref · scf · vector · llvm dialect         │
└──────┬──────────────────────┬──────────────────────────────┘
       │                      │
  ┌────▼────┐           ┌─────▼──────┐
  │ LLVM IR │           │ gpu / nvvm │
  │ CPU/ARM │           │ rocdl/spirv│
  └────┬────┘           └─────┬──────┘
  native .so             PTX / CUBIN / HSACO / SPIR-V
```

---

## 179.2 The MLIR Dialect Tower for AI

The path from a named operator (`stablehlo.dot_general`) to machine code passes through four dialect levels, each serving a distinct purpose.

### 179.2.1 Level Roles and the Bufferisation Boundary

| Level | Key Dialects | Primary Role |
|-------|-------------|-------------|
| Input ops | `stablehlo`, `tosa`, Torch | Operator semantics, shape typing |
| Structured loops | `linalg`, `tensor`, `affine`, `scf` | Tiling, fusion, iteration space |
| Register tile | `vector` | SIMD abstraction, hardware mapping |
| Buffers + target | `memref`, `llvm`, `nvvm`, `rocdl` | Memory, ABI, instruction emission |

The **bufferisation boundary** divides the tower in two. Above the line, all operations use the `tensor` type: a functional, aliasing-free view where every op produces a fresh tensor value. This enables safe fusion, CSE, and algebraic rewriting without alias analysis. Below the line, `memref` provides mutable buffer semantics: loads and stores with explicit address computation. One-Shot Bufferization (cross-ref Ch151 — Bufferization Deep Dive) crosses this boundary with a global analysis that minimises copies.

The boundary has a precise location in the pass pipeline. The `--one-shot-bufferize` pass is always applied after all tensor-level fusions (elementwise fusion, `linalg` tiling, `vector` contraction generation) and before any target-specific lowering. Applying it too early (before tiling) prevents the tiler from reasoning about tile shapes; applying it too late means target-specific passes receive tensor semantics they cannot handle. Cross-ref Ch152 — Lowering Pipelines for the canonical pipeline ordering.

### 179.2.2 `linalg.generic` as Universal Abstraction

`linalg.generic` ([`mlir/include/mlir/Dialect/Linalg/IR/LinalgOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Linalg/IR/LinalgOps.td), cross-ref Ch139 — Tensor and Linalg) is the key intermediate form. Every structured computation — matmul, convolution, element-wise map, reduction — can be expressed as a `linalg.generic` with indexing maps and iterator types:

```mlir
// linalg.generic for matrix multiply: C[i,j] += A[i,k] * B[k,j]
#map_A = affine_map<(i, j, k) -> (i, k)>
#map_B = affine_map<(i, j, k) -> (k, j)>
#map_C = affine_map<(i, j, k) -> (i, j)>

%C_out = linalg.generic {
    indexing_maps = [#map_A, #map_B, #map_C],
    iterator_types = ["parallel", "parallel", "reduction"]
  }
  ins(%A, %B : tensor<128x64xf32>, tensor<64x128xf32>)
  outs(%C : tensor<128x128xf32>) {
  ^bb0(%a: f32, %b: f32, %c: f32):
    %mul = arith.mulf %a, %b : f32
    %add = arith.addf %c, %mul : f32
    linalg.yield %add : f32
} -> tensor<128x128xf32>
```

The three `iterator_types` tell the tiling infrastructure which dimensions can be distributed (`"parallel"`) and which must be reduced before the result is valid (`"reduction"`). This is what enables `transform.structured.tile_using_forall` to partition the `i` and `j` loops across threads while keeping the `k` reduction inside each tile.

The named ops `linalg.matmul`, `linalg.conv_2d_nhwc_hwcf`, and `linalg.depthwise_conv_2d_nhwc_hwc` are sugar over `linalg.generic` with pre-set indexing maps. Lowering from named to generic is a single pattern rewrite; the reverse (named op recognition from generic) is used by the Linalg vectorisation pass to select the MMA shape for `vector.contract`.

### 179.2.3 `vector` as the SIMD Bridge

After tiling produces loops with small, fixed-size bodies, the `linalg` vectorisation pass (exposed via `transform.structured.vectorize_children_and_apply_patterns` in the Transform dialect, cross-ref Ch150 — The Transform Dialect) replaces the inner loop body with `vector` ops. The resulting IR uses:

- `vector.transfer_read` / `vector.transfer_write` ([`mlir/include/mlir/Dialect/Vector/IR/VectorOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Vector/IR/VectorOps.td)) — boundary-safe load/store from `memref` or `tensor` into `vector<...>` register tiles (cross-ref Ch141 — Vector and Sparse)
- `vector.contract` — the generalised multiply-accumulate that maps to either `dot`, `fma`, or a tensor-core MMA depending on the lowering target
- `vector.outerproduct` — GEMM microkernel building block that lowers to a sequence of VFMADD on AVX-512 or FMA on Arm NEON

```mlir
// vector.contract for a 4x4 matmul tile
// C[m,n] += A[m,k] * B[k,n]
%result = vector.contract
    {indexing_maps = [affine_map<(m,n,k) -> (m,k)>,
                      affine_map<(m,n,k) -> (k,n)>,
                      affine_map<(m,n,k) -> (m,n)>],
     iterator_types = ["parallel", "parallel", "reduction"],
     kind = #vector.kind<add>}
    %va, %vb, %vc
    : vector<4x4xf32>, vector<4x4xf32> into vector<4x4xf32>
```

`--convert-vector-to-llvm` lowers `vector.contract` to LLVM `llvm.intr.fmuladd` chains for CPU. The GPU path instead routes `vector.contract` through `nvgpu.mma.sync` for Ampere tensor cores or `nvgpu.warpgroup_mma` for Hopper WGMMA.

---

## 179.3 Quantization: Dialects, Formats, and Lowering

Quantisation is the practice of representing weights and activations at lower precision — typically INT8 or INT4 — to reduce memory bandwidth and arithmetic cost. The MLIR ecosystem has three distinct entry points for quantised computation.

### 179.3.1 The `quant` Dialect

MLIR's `quant` dialect ([`mlir/include/mlir/Dialect/Quant/QuantOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Quant/QuantOps.td)) provides a type system for quantised values. The core type is:

```mlir
!quant.uniform<i8:f32, 0.0392156862:128>
// storage type : expressed type, scale : zero_point
```

- **Storage type** (`i8`): the bit-width at rest in memory
- **Expressed type** (`f32`): the floating-point domain the integer approximates
- **Scale and zero_point**: the affine mapping `real = (stored - zero_point) * scale`

For symmetric quantisation (zero_point = 0) the formula simplifies to `real = stored * scale`. Symmetric INT8 maps to the range `[-127, 127]`, which is the most common format for weight matrices. Per-channel quantisation uses a `!quant.uniform<i8<-127:127>:f32:0, {0.00392, 0.00784, ...}>` type that carries one scale per output channel (the `:0` denotes the quantised axis). Per-channel is strictly more accurate than per-tensor because it captures per-row variation in weight magnitudes.

The two core conversion ops are:

```mlir
// quantise: f32 → !quant.uniform<i8:f32, ...>
%q = quant.qcast %fp : f32 to !quant.uniform<i8:f32, 0.00392:0>

// dequantise: !quant.uniform<i8:f32, ...> → f32
%fp = quant.dcast %q : !quant.uniform<i8:f32, 0.00392:0> to f32
```

A `quant.scast` (storage cast) also exists for reinterpreting the storage type without affine rescaling, useful for packing and unpacking raw quantised data.

In practice, `quant.dcast` before a matmul is the pattern for **weight-only quantisation**: the stored INT8 weight is dequantised to FP16/BF16 at kernel entry, then the matmul runs in floating point. True **INT8 compute** (both operands quantised) requires lowering to quantised matmul kernels, which carry the output in INT32 and rescale at the end.

### 179.3.2 Precision Formats

| Format | Range | Notes |
|--------|-------|-------|
| INT8 symmetric | `[-127, 127]` | Industry standard; cuBLAS `cublasGemmEx`, CPU VNNI |
| INT8 asymmetric | `[-128, 127]` | Asymmetric zero point adds bias correction term |
| INT4 (W4A16) | 16 levels/value | Weights in INT4, activations in FP16; GPTQ target |
| FP8 E4M3 | ~`[-448, 448]` | H100 `fp8_e4m3fn` type; narrower range than INT8 |
| FP8 E5M2 | ~`[-57344, 57344]` | Higher dynamic range; used for gradient tensors |
| BF16 | exponent = FP32 | Standard training/inference accumulation format |

FP8 training (Transformer Engine, used in H100/H200 training) stores weights in FP8 E4M3 and uses a per-tensor scale factor derived from the "amax history" — a rolling window of the absolute maximum value seen in that tensor over the last N steps. The MLIR element types `f8E4M3FN` and `f8E5M2` are defined in [`mlir/include/mlir/IR/BuiltinTypes.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/BuiltinTypes.td).

### 179.3.3 GPTQ and Post-Training Weight Quantisation

GPTQ (Frantar et al., NeurIPS 2022) quantises large language model weight matrices to INT4 or INT3 after training using second-order Hessian information. The core insight is that the Hessian of the reconstruction loss with respect to each weight column can be approximated cheaply from the calibration inputs, and the optimal quantisation error can be propagated across columns to compensate for quantised values.

**The algorithm per weight matrix W:**
1. Compute the layer input Hessian H = 2 * X^T * X (where X is the calibration input matrix, batch × in_features)
2. For each column group g (typically 128 consecutive columns):
   a. Cholesky-decompose H_g^{-1} to get the inverse Hessian
   b. For each weight w_{i,g} in the group, round to the nearest quantised value; propagate the rounding error to the remaining columns in the group via the inverse Hessian row

The per-group scale and zero are stored as FP16 values in a separate matrix of shape `[out_features, ceil(in_features/group_size)]`.

The MLIR lowering is:
1. Load the packed INT4 tile from global memory (two INT4 values per INT8 byte)
2. Unpack via `arith.andi` + shift to extract individual nibbles
3. Dequantise to FP16: `real = (stored - zero) * scale` using the group scale/zero
4. Accumulate into FP16/BF16 via `vector.contract`

```mlir
// Pseudocode for GPTQ dequant + matmul dispatch
// packed_w: tensor<?x?xi32> (8 INT4 values packed per INT32)
// scales:   tensor<?x?xf16> (per group, shape [N, K/group_size])
// zeros:    tensor<?x?xf16> (per group, same shape as scales)

// Unpack INT4 from INT32 word (group of 8 values)
%nibble_lo = arith.andi %packed_w, %mask_0f : i32   // bits [3:0]
%nibble_hi = arith.shrui %packed_w, %c4 : i32        // bits [7:4]
// ... repeat for all 8 nibbles

// Dequantise: real = (nibble - zero) * scale
%centered = arith.subi %nibble_lo, %zero_i : i32
%centered_f16 = arith.sitofp %centered : i32 to f16
%dequant = arith.mulf %centered_f16, %scale_f16 : f16
```

There is no standard `quant` dialect representation for GPTQ's two-level (group scale + nibble) format; most implementations lower directly to LLVM intrinsics or Triton kernels (cross-ref Ch163 — Triton). ExLlamaV2 and AutoAWQ each use bespoke CUDA kernels; IREE's VMFB format can carry the packed weights and dispatch a CPU or Vulkan dequant-then-matmul path.

### 179.3.4 GGUF: File Format for Quantised LLMs

GGUF (llama.cpp's successor to GGML) is a self-describing flatbuffer format for quantised models:

- **Header**: magic `GGUF`, version, tensor count, metadata key-value pair count
- **Metadata KV store**: model family, context length, tokenizer vocab, architecture hyperparameters (head count, layer count, rope theta, etc.)
- **Tensor info array**: name string, shape (n_dims × uint64), `ggml_type` enum, byte offset within data blob
- **Data blob**: raw quantised weights, each tensor aligned to 32 bytes

The quantisation types range from full FP16 (`F16`) to aggressively quantised formats:

| Type | Effective bits/weight | Method |
|------|----------------------|--------|
| `Q8_0` | 8.5 | Per-32-weight block, f16 scale stored with block |
| `Q5_K_M` | 5.5 | K-quant; 6-bit quantised sub-scales within super-block |
| `Q4_K_M` | 4.5 | K-quant with super-block scales and per-block min values |
| `Q3_K_S` | 3.4 | K-quant, small variant |
| `Q2_K` | 2.6 | Very aggressive; 4-bit scales for 16-weight blocks |

The `_K` types use a two-level quantisation: a 256-weight super-block has an outer FP16 scale and FP16 min; within it, 8 sub-blocks of 32 weights each have a 6-bit quantised subscale stored compactly. Loading `Q4_K_M` therefore requires decoding two levels of scale before the weight values. The `_M` (medium) and `_S` (small) suffixes denote quality selections: in `Q4_K_M`, attention key/query layers use `Q5_K` while feed-forward weights use `Q4_K`, matching the observed sensitivity to quantisation error.

MLIR-based loading maps the data blob into memory and generates `linalg.generic` dequantise + matmul sequences for each layer. The [llama.cpp](https://github.com/ggml-org/llama.cpp) project provides the reference C implementation; IREE and Turbine (the torch-mlir successor project) are building MLIR-native dequantise paths.

### 179.3.5 The StableHLO Quantization Path

StableHLO carries quantisation via `stablehlo.uniform_quantize` and `stablehlo.uniform_dequantize` (defined in [`stablehlo/dialect/StablehloOps.td`](https://github.com/openxla/stablehlo/blob/main/stablehlo/dialect/StablehloOps.td)):

```mlir
// Quantise activations before dispatch
%q_input = stablehlo.uniform_quantize %fp_input
    : (tensor<1x784xf32>) -> tensor<1x784x!quant.uniform<i8:f32, 0.0156:0>>

// Quantised matrix multiply
%result = stablehlo.dot_general
    %q_input, %q_weight,
    contracting_dims = [1] x [0]
    : (tensor<1x784x!quant.uniform<i8:f32, 0.0156:0>>,
       tensor<784x128x!quant.uniform<i8:f32, 0.00784:0>>)
    -> tensor<1x128x!quant.uniform<i32:f32, 6.1e-5:0>>

// Dequantise output
%fp_result = stablehlo.uniform_dequantize %result
    : (tensor<1x128x!quant.uniform<i32:f32, 6.1e-5:0>>) -> tensor<1x128xf32>
```

This path is used by the TFLite quantiser and by IREE's quantised CPU path. IREE's compiler lowers the quantised `stablehlo.dot_general` to a `linalg.quantized_matmul` named op, which then tiles and vectorises to `vector.contract` on `i8` vectors, mapping to VNNI on x86 (`_mm256_dpbusds_epi32`) or `sdot` on Arm Cortex-A76+.

The TFLite INT8 calibration workflow (post-training quantisation) uses the `stablehlo_quantizer` pass pipeline: it annotates `stablehlo` ops with `quant.stats` attributes computed from a representative dataset, then folds those stats into `!quant.uniform` types and inserts `qcast`/`dcast` ops at the quantisation boundaries.

---

## 179.4 Dynamic Shapes

Static shapes are the compiler's friend: tile sizes, loop bounds, vector widths all become compile-time constants, enabling the full optimisation stack. But NLP models, object detection, and many production scenarios require handling variable-length sequences or batches.

### 179.4.1 The Cost of Static Shapes

Padding to a maximum length wastes bandwidth proportional to `(max_len - actual_len) / max_len`. For sequence models with high length variance, this can be 50–80% waste. XLA's approach is a trace cache (XLA compilation cache keyed by exact shape tuple): the first time a new shape appears, it triggers recompilation; subsequent hits are free. The cost is warm-up latency on first request — 500 ms to several seconds for large models — and a cache that can grow unbounded if shapes are truly unbounded.

Recompilation on every new shape also prevents ahead-of-time (AOT) deployment: a mobile app cannot trigger JIT compilation in the field. Static shape models compiled ahead of time (via IREE or TFLite) avoid this but must be padded or shape-specialised at model preparation time.

### 179.4.2 StableHLO Dynamic Shape Extensions

StableHLO represents dynamic dimensions as `?` in tensor types, with dedicated ops for runtime shape manipulation:

```mlir
// Dynamic shape: batch dimension unknown at compile time
%result : tensor<?x128xf32>

// Query a runtime dimension
%batch_size = stablehlo.get_dimension_size %input, dim = 0
    : (tensor<?x784xf32>) -> tensor<i32>

// Dynamic broadcast (shape vector computed at runtime)
%shape_vec = stablehlo.concatenate %b_size, %c128
    : (tensor<1xi32>, tensor<1xi32>) -> tensor<2xi32>
%bias_bcast = stablehlo.dynamic_broadcast_in_dim %bias, %shape_vec
    broadcast_dimensions = [1]
    : (tensor<128xf32>, tensor<2xi32>) -> tensor<?x128xf32>

// Reshape with a runtime shape
%reshaped = stablehlo.dynamic_reshape %input, %shape_tensor
    : (tensor<?x784xf32>, tensor<2xi32>) -> tensor<?x?xf32>
```

XLA's dynamic shape engine propagates shape computations through the HLO graph as companion `i32` tensors. At dispatch time, a shape analysis kernel runs on the host CPU to compute the concrete shapes, which are then used to configure the device kernels. Cross-ref Ch154 — HLO and StableHLO for the static-shape baseline.

### 179.4.3 The `shape` Dialect

MLIR's `shape` dialect ([`mlir/include/mlir/Dialect/Shape/IR/ShapeOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Shape/IR/ShapeOps.td)) provides an SSA representation for shape computations that can be analysed and folded:

```mlir
// Shape inference through broadcast
%shape_a = shape.shape_of %tensor_a : tensor<?x?xf32> -> !shape.shape
%shape_b = shape.shape_of %tensor_b : tensor<?xf32>   -> !shape.shape
%bcast   = shape.broadcast %shape_a, %shape_b          -> !shape.shape

// Assuming block: ops inside are guarded by the shape constraint
%ok = shape.cstr_broadcastable %shape_a, %shape_b -> !shape.witness
shape.assuming %ok {
  %result = linalg.map { arith.addf }
      ins(%a, %b : tensor<?x?xf32>, tensor<?xf32>)
      outs(%out : tensor<?x?xf32>)
}
```

Ops implementing `InferShapedTypeOpInterface` can fold shape inference at compile time when partial static information is available — for example, a batch dimension that is symbolic `?` but output channels that are static `128`. This hybrid is how torch-mlir represents models exported with `torch.export.Dim("batch")` constraints.

### 179.4.4 `tensor.dim` and Dynamic Tensor Operations

The `tensor` dialect carries dynamic sizes with `?` in the type and uses `tensor.dim` for runtime queries:

```mlir
// Reshape where only the second dim is dynamic
%reshaped = tensor.reshape %t(%shape) : (tensor<512xf32>, tensor<2xindex>) -> tensor<16x?xf32>

// Expand static + dynamic dims
%expanded = tensor.expand_shape %t [[0, 1]] : tensor<?xf32> into tensor<1x?xf32>

// Query at runtime — returns index type
%n = tensor.dim %t, %c0 : tensor<?x128xf32>
```

The key insight is that `?` dimensions are **opaque at compile time but ranked**: the compiler knows the rank (number of dimensions) but not the sizes. Tile size selection must be conservative (e.g., use a tile size that divides any value in the valid range) or use the Transform dialect's tile-using-for-all which inserts runtime bounds checks for the final partial tile.

### 179.4.5 IREE's Dynamic Dispatch

IREE handles dynamic shapes by separating **shape computation** from **dispatch** at the `flow` dialect level (cross-ref Ch162 — IREE). A `flow.dispatch.workgroups` op carries the workgroup count as SSA values computed from `tensor.dim` queries:

```mlir
%n = tensor.dim %input, %c0 : tensor<?x784xf32>     // runtime batch size
%wg_y = arith.ceildivui %n, %c64 : index             // ceil(N/64) workgroups
%result = flow.dispatch.workgroups[%wg_y, %c2, %c1]
    (%input : tensor<?x784xf32>) -> tensor<?x128xf32> {
  ^bb0(%in : !flow.dispatch.tensor<readonly:tensor<?x784xf32>>,
       %out : !flow.dispatch.tensor<writeonly:tensor<?x128xf32>>):
    // inside dispatch: linalg on ranked dynamic tensors
    %in_t  = flow.dispatch.tensor.load %in  : tensor<?x784xf32>
    %out_t = flow.dispatch.tensor.load %out : tensor<?x128xf32>
    %mm = linalg.matmul ins(%in_t, %W : tensor<?x784xf32>, tensor<784x128xf32>)
                        outs(%out_t : tensor<?x128xf32>) -> tensor<?x128xf32>
    flow.dispatch.tensor.store %mm, %out : tensor<?x128xf32>
    flow.return
  }
```

The HAL layer computes the dispatch grid at runtime from `%wg_y`, so no recompilation is needed for different batch sizes.

### 179.4.6 torch.export with Dynamic Shapes

PyTorch's `torch.export` API accepts explicit dynamic shape annotations:

```python
import torch
from torch.export import export, Dim

batch = Dim("batch", min=1, max=512)
seq   = Dim("seq",   min=1, max=2048)

exported = export(
    model,
    args=(input_ids, attention_mask),
    dynamic_shapes={"input_ids":      {0: batch, 1: seq},
                    "attention_mask": {0: batch, 1: seq}}
)
```

The resulting `ExportedProgram` embeds the symbolic shape constraints as `torch.sym_int` values in the FX graph. torch-mlir converts these to `!torch.vtensor<[?, ?], si64>` types with symbolic dim names. The downstream MLIR compiler can partially specialise on static dimensions (vocabulary size, model hidden width) while leaving batch and sequence truly dynamic.

---

## 179.5 Hardware Target Landscape

### 179.5.1 CUDA / NVPTX

The MLIR-to-CUDA path proceeds through the `gpu`, `nvgpu`, and `nvvm` dialects (cross-ref Ch142 — GPU Dialect Family, Ch165 — GPU Compilation through MLIR):

```
gpu.func  →  nvgpu / nvvm dialect  →  nvptx64-nvidia-cuda LLVM IR
          →  PTX text assembly (via LLVM NVPTX backend)
          →  CUBIN via ptxas (JIT or ahead-of-time)
```

The critical junction is `vector.contract` → Tensor Core. The pass `--convert-vector-to-gpu` detects `vector.contract` ops with shapes matching Tensor Core tile sizes and emits `nvgpu.mma.sync` for Ampere or `nvgpu.warpgroup_mma` for Hopper. Key shapes (all FP16→FP32):

| Architecture | PTX instruction | Tile shape |
|---|---|---|
| Volta (V100) | `wmma.mma.sync` | 16×16×16 |
| Ampere (A100) | `mma.sync.aligned.m16n8k16` | 16×8×16 |
| Hopper (H100) | `wgmma.mma_async.sync.aligned` | 64×…×16 (warpgroup) |

The `nvgpu` dialect ([`mlir/include/mlir/Dialect/NVGPU/IR/NVGPUOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/NVGPU/IR/NVGPUOps.td)) exposes:

- `nvgpu.mma.sync` — Ampere warp-level MMA
- `nvgpu.warpgroup_mma` — Hopper warpgroup-level WGMMA (dispatches the persistent async mma machinery)
- `nvgpu.tma_create_descriptor` / `nvgpu.tma_async_load` — Hopper TMA (Tensor Memory Accelerator): hardware DMA that loads a multi-dimensional tile from global memory to shared memory, handling striding, swizzling, and OOB masking in hardware

WGMMA requires that the input tiles reside in shared memory with a specific swizzle pattern (`128-byte swizzle` or `64-byte swizzle`) and are 128-byte aligned. The `--nvgpu-rewrite-to-warpgroup-mma` pass inserts the shared-memory staging and swizzle computation before the WGMMA.

When is cuDNN better than a compiled MLIR kernel? cuDNN implements hand-tuned implicit GEMM convolution with winograd transforms, multi-head attention via the cuDNN Frontend Graph API (CUDNN_FRONTEND_GRAPH), and recurrent cells. For standard operators in a production Transformer inference serving path, cuDNN frequently wins because it selects over hundreds of pre-compiled algorithm variants at runtime (timing cache). Compiled MLIR kernels win for custom operators and novel architectures where cuDNN has no matching algorithm.

### 179.5.2 ROCm / AMDGPU

The AMD path mirrors NVIDIA but uses the `rocdl` dialect and the AMDGPU LLVM backend:

```
gpu.func  →  rocdl dialect  →  amdgcn-amd-amdhsa LLVM IR
          →  HSACO (HSA Code Object) via lld
          →  loaded by hipModuleLoad / HIP driver
```

AMD Matrix Fused Multiply-Add (`MFMA`) instructions are the equivalent of NVIDIA's MMA. The `amdgpu` dialect ([`mlir/include/mlir/Dialect/AMDGPU/IR/AMDGPUDialect.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/AMDGPU/IR/AMDGPUDialect.td)) provides:

```mlir
// MFMA for FP32 accumulation from FP16 operands: 16×16×16 tile
%result = amdgpu.mfma %a, %b, %c
    { m = 16 : i32, n = 16 : i32, k = 16 : i32,
      blocks = 1 : i32, abid = 0 : i32, cbsz = 0 : i32 }
    : vector<4xf16>, vector<4xf16>, vector<4xf32>

// WMMA (RDNA3/gfx11): wave32 16×16 FP16 matmul
%result_wmma = amdgpu.wmma %a, %b, %c
    : vector<16xf16>, vector<16xf16>, vector<8xf32>
```

The `vector.contract` → MFMA mapping is handled by the `--convert-vector-to-amdgpu` pass, which inspects the contraction shape, operand types, and target GPU architecture (gfx908 = MI100, gfx90a = MI210/MI250, gfx940 = MI300X) to select the appropriate `amdgpu.mfma` variant. The MI300X exposes 1152 GB/s HBM3 bandwidth and supports `MFMA_F8F6F4` for FP8 matrix multiply.

IREE supports ROCm as a first-class target with the `rocm` HAL backend; the compilation path is nearly identical to CUDA, diverging at `--convert-gpu-to-rocdl` instead of `--convert-gpu-to-nvvm`.

### 179.5.3 Apple ANE (Neural Engine)

Apple's Neural Engine is not directly programmable through an open compiler stack. The only supported path is:

```
PyTorch / JAX  →  torch.export / tracing
    │  coremltools.convert()
    ▼
CoreML .mlpackage (MLProgram protobuf + weights directory)
    │  on-device Neural Engine compiler (private, runs at app-install time)
    ▼
ANE binary (Apple proprietary, architecture-specific)
```

CoreML's operator set is the effective ANE ISA. Ops outside the set (e.g., custom activations, certain attention patterns lacking CoreML support) fall back to the GPU (via Metal) or CPU (via Accelerate). The `coremltools` Python library handles model conversion and exposes an `MLProgram` IR format, but this IR is not publicly documented at the instruction level. There is no public MLIR dialect targeting ANE; the compilation from MLProgram to ANE binary happens inside the OS.

### 179.5.4 Qualcomm Hexagon DSP and QNN

The `hexagon` dialect (downstream of LLVM, maintained by Qualcomm) targets Hexagon's HVX (Hexagon Vector eXtensions): 1024-bit SIMD vectors operating on packed 8/16/32-bit integers. IREE's Hexagon HAL backend compiles via:

```
linalg.generic  →  linalg tiling + vectorisation
    │  vector → HVX lowering (Hexagon-specific vector patterns)
    │  LLVM Hexagon backend (llvm/lib/Target/Hexagon/)
    ▼
Hexagon ELF shared library embedded in .vmfb + HAL dispatch
```

Key Hexagon properties: HVX_V69 (Snapdragon 8 Gen 3) supports 32×32-bit multiply-accumulate in a single cycle across the 1024-bit vector. The `V6_vmpybus_acc` intrinsic performs unsigned×signed i8 multiply-accumulate into i16 accumulators, mapping to INT8 inference for convolutional layers.

The Qualcomm Neural Network SDK (QNN) is Qualcomm's deployment runtime, consuming ONNX or TFLite as input. For the dedicated AI Engine (cDSP + HTA architecture), IREE and QNN are complementary: IREE handles Hexagon CPU path; QNN handles the dedicated HTA (Hexagon Tensor Accelerator) hardware.

### 179.5.5 Edge TPU and Embedded NPUs

Google's Edge TPU requires TFLite flatbuffer input through `edgetpu_compiler`, which produces a `.tflite` file with custom op metadata that the Edge TPU runtime dispatches to the accelerator. Constraints: fully INT8 quantised, static shapes, limited op support (Conv2D/DepthwiseConv2D/MatMul/Add/ReLU). Unsupported ops fall back to the Cortex-A CPU.

The generalised pattern for embedded NPU targets: nearly all require a **specific frontend format** (TFLite, ONNX, QNN, CoreML) as the hardware-facing API. The MLIR-based compiler's job is to compile *to* that format, then the vendor runtime handles final hardware dispatch. MLIR's `tosa` dialect ([`mlir/include/mlir/Dialect/Tosa/IR/TosaOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/Tosa/IR/TosaOps.td)) is increasingly used as the portable intermediate: TOSA's ~90 ops have well-defined semantics that the major NPU SDKs can interpret, and MLIR's `--tosa-to-tflite` pass bridges to the TFLite flatbuffer schema.

---

## 179.6 Inference Deployment Stack

The choice of deployment runtime determines what optimisations, targets, and deployment constraints apply. The four major options are not interchangeable.

### 179.6.1 IREE

IREE (cross-ref Ch162 — IREE) is the most complete open-source MLIR-native deployment compiler. Its key advantages: single compilation produces multi-target binaries in a `.vmfb` FlatBuffer; the HAL abstraction adds only ~5% overhead over hand-written kernels in most benchmarks; the VM bytecode allows deployment without a host compiler. IREE ships an `iree-run-module` binary and a C API for embedding in applications.

IREE's CPU codegen uses LLVM's `linalg`-to-LLVM pipeline with micro-kernel dispatch: hot inner loops (matmul, convolution) are dispatched to architecture-tuned micro-kernels (via the `ukernels` library, analogous to BLIS's microkernels) while the tiling and packing is done in generic MLIR-generated code. The Vulkan backend uses SPIR-V generated from the `gpu` + `spirv` dialect pipeline.

### 179.6.2 TFLite

TFLite uses a FlatBuffer schema ([`tensorflow/lite/schema/schema.fbs`](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/lite/schema/schema.fbs)) for the model graph and a hand-written C++ interpreter for execution. The delegate API (`TfLiteDelegate`) allows routing ops to accelerated backends:

- **GPU delegate**: OpenGL ES 3.1 Compute / Vulkan compute shaders for Android; Metal for iOS
- **NNAPI delegate**: Android's Neural Networks API (routes to vendor NPU/DSP)
- **Hexagon delegate**: Qualcomm DSP via QNN
- **Coral Edge TPU delegate**: INT8 ops to the Edge TPU chip

TFLite Micro removes dynamic memory allocation entirely, enabling deployment on MCUs with 256 KB RAM. The model must be pre-quantised to INT8 and use only supported ops. TFLite's tensor arena allocator is a statically computed bump allocator over a pre-allocated byte array — no `malloc` or `free` at inference time. Choose TFLite when: the target is Android/iOS with vendor NPU; the model is standard (MobileNet, EfficientDet, MediaPipe); low-resource MCU deployment is required.

### 179.6.3 ONNX Runtime

ONNX Runtime's execution provider (EP) model partitions the ONNX graph into sub-graphs, each dispatched to the fastest available EP:

| EP | Accelerator | Notes |
|----|-------------|-------|
| CUDA EP | NVIDIA GPU | cuBLAS, cuDNN, cuSPARSE |
| TensorRT EP | NVIDIA TRT engine | INT8 calibration, engine cache |
| DirectML EP | Windows GPU (DX12) | Via D3D12 compute |
| QNN EP | Qualcomm AI Engine | QNN SDK, INT8 quantised |
| CoreML EP | Apple ANE + GPU | Via coremltools |
| OpenVINO EP | Intel CPU/GPU/VPU | Intel-specific fusions |
| XNNPACK EP | ARM/x86 CPU | Mobile-optimised FP32/FP16 |

The EP partitioning works by querying each EP's `GetCapability()` method: the EP returns a set of `NodeIndex` groups it can handle; the runtime assigns each group to the highest-priority capable EP. Sub-graphs that no accelerator EP claims are handled by the CPU EP (using Eigen/OpenBLAS).

The C API (`OrtEnv`, `OrtSession`, `OrtValue`) is the stable ABI. The C++ API is header-only and adds convenience wrappers around the C types. Choose ONNX Runtime when: the model is already in ONNX; cross-platform deployment with vendor EP flexibility is needed; Windows DirectML is the primary target.

### 179.6.4 TensorRT

TensorRT is NVIDIA's production inference library. Its engine-build phase fuses layers, calibrates INT8, and serialises to a platform-specific engine file. Key capabilities unavailable in open MLIR stacks:

- **Algorithm selection**: TRT benchmarks hundreds of cuDNN algorithm variants (IMPLICIT_GEMM, PRECOMP_GEMM, WINOGRAD, FFT, etc.) at engine-build time, selecting the fastest for the exact batch size and input shape on the specific GPU. The result is stored in the timing cache. This is why TRT often outperforms hand-tuned Triton kernels: the search space covers operations that Triton cannot express (Winograd convolution, multi-head attention with Flash variants).
- **INT8 PTQ calibration**: `IInt8Calibrator` interface runs a representative dataset and collects per-activation histograms, computing optimal per-tensor scale via entropy or percentile methods.
- **2:4 structured sparsity**: NVIDIA's sparse tensor core on Ampere/Hopper supports 2:4 structured weight sparsity (exactly 2 zeros per 4 elements), achieving 2× throughput for sparse GEMM. TRT's `--sparsity` flag prunes and re-fine-tunes the model.
- **Plugins**: custom CUDA kernels registered via `IPluginV3` with typed FP16/INT8/BF16 I/O that TRT treats as first-class fused nodes.

Choose TRT when: target is exclusively NVIDIA GPU; maximum throughput is the goal; commercial deployment licensing is acceptable.

### 179.6.5 Decision Matrix

| Requirement | Best Choice |
|-------------|------------|
| NVIDIA, max throughput | TensorRT |
| Cross-platform (CPU + GPU + Vulkan) | IREE |
| Android/iOS mobile NPU | TFLite |
| ONNX model, cross-vendor EP | ONNX Runtime |
| Custom MLIR dialect / research | IREE or direct mlir-opt pipeline |
| Windows DirectML | ONNX Runtime (DirectML EP) |
| MCU / bare-metal | TFLite Micro |
| AMD GPU deployment | IREE (ROCm) or ROCm MIOpen directly |

---

## 179.7 Training vs Inference Compilation

Training and inference have fundamentally different compilation goals, and the MLIR-based stack handles them differently.

### 179.7.1 Inference: Latency and Throughput

Inference compilation can assume fixed weights (no gradient state), known batch size (or bounded range), and numerical freedom to quantise. The entire chapter up to this point has focused on inference. The optimisation objectives — minimise memory bandwidth, maximise arithmetic intensity, minimise dispatch overhead — are all downstream of the assumption that the computation graph is fixed at deploy time.

### 179.7.2 Training: Gradient Correctness and Memory

Training introduces two compilation constraints with no inference analogue.

**Gradient correctness.** Every operator must have a defined adjoint. Numerically unstable forms of the forward pass that are acceptable for inference (e.g., `logsumexp` computed as `log(sum(exp(...)))`) must be replaced by numerically stable gradient-friendly forms. The cross-entropy loss gradient, for example, requires the softmax output before reduction, not the scalar loss, which affects fusion opportunities.

**Memory efficiency via rematerialisation.** A standard GPT-3-scale model (175B parameters) in FP16 requires ~350 GB for parameters. The activation memory for a single forward pass at training batch size can exceed this by 2×–4×. The activations must be stored for use in the backward pass; but holding all of them simultaneously is infeasible.

**Activation checkpointing** (rematerialisation) solves this by freeing selected activation tensors after the forward pass and recomputing them during the backward pass when they are needed for the gradient. The tradeoff is ~33% extra compute in exchange for a square-root reduction in peak memory.

JAX exposes rematerialisation via `jax.checkpoint` with a policy argument:

```python
from jax.checkpoint import checkpoint, checkpoint_policies

# Only save values that are "dots_with_no_batch_dims" (matrix multiplies)
# Recompute everything else: elementwise, layer norms, activations
@functools.partial(checkpoint, policy=checkpoint_policies.dots_saveable)
def transformer_block(x, params):
    ...
```

The policy types include:
- `everything_saveable`: no rematerialisation (baseline memory)
- `nothing_saveable`: recompute everything (minimal memory, maximum compute)
- `dots_saveable`: save only matmul outputs; recompute activations and norms
- `checkpoint_dots_with_no_batch_dims`: save only non-batched matmuls

At the XLA HLO level, `jax.checkpoint` introduces a `custom_jvp_call_jaxpr` boundary that tells the XLA scheduler to free the enclosed activations and insert a recomputation node in the backward pass. This is implemented in XLA's HLO scheduling pass, not in the MLIR layer. Current MLIR-based training stacks implement rematerialisation inside XLA rather than exposing it as a general MLIR transformation.

### 179.7.3 Mixed Precision and BF16

Mixed-precision training stores weights in FP32 (for stable gradient updates via the Adam/AdaGrad optimizer's running moments) but computes forward and backward passes in BF16. The master weight copy in FP32 is updated at the end of each step from the BF16 gradients scaled back up.

The StableHLO representation uses `tensor<...xbf16>` operands with `arith.truncf` / `arith.extf` conversions at the precision boundary. XLA:GPU inserts these casts during the `--xla-gpu-enable-bf16-gemm` pass (cross-ref Ch156 — XLA:GPU), which also fuses the cast into the GEMM epilogue to avoid materialising a full FP32 output tensor.

FP8 training (H100 Transformer Engine) requires tracking the per-tensor amax (absolute maximum) across steps to set the per-tensor scale factor. The MLIR element type `f8E4M3FN` (E4M3, finite-only, NaN-representation-free) is used for weights and forward activations; `f8E5M2FNUZ` is used for gradients where the wider dynamic range matters. The companion scale tensor is a `tensor<f32>` updated every step from the amax history.

### 179.7.4 Distributed Training: SPMD Sharding

SPMD-sharded training distributes the computation across a device mesh. The StableHLO collective ops (cross-ref Ch158 — SPMD/GSPMD and Auto-Sharding) are the MLIR-level representation:

```mlir
// AllReduce gradient across data-parallel axis
%grad_sum = stablehlo.all_reduce %local_grad
    replica_groups = [[0, 1, 2, 3], [4, 5, 6, 7]]
    channel_handle = #stablehlo.channel_handle<handle = 1, type = 1>
    : (tensor<128x128xf32>) -> tensor<128x128xf32>

// AllGather for Megatron-style tensor parallelism: reassemble shards
%full_weight = stablehlo.all_gather %shard,
    all_gather_dim = 1
    replica_groups = [[0, 2], [1, 3]]  // column-parallel groups
    : (tensor<128x64xf32>) -> tensor<128x128xf32>
```

The GSPMD partitioner (XLA's sharding propagation pass) takes per-tensor sharding annotations (`@mesh<"data"=4, "model"=2>`) and the single-device program, and produces the collective-augmented multi-device program automatically. Each device runs the same compiled XLA executable on its shard.

---

## 179.8 Kernel Fusion: FlashAttention as Case Study

No single kernel better illustrates the need for ML-aware compiler fusion than attention. It is also the benchmark against which MLIR-based fusion claims must be measured.

### 179.8.1 The Fusion Problem in Attention

Naive multi-head self-attention computes `softmax(QK^T / sqrt(d_k)) * V`. The `QK^T` matrix has shape `[batch, heads, seq, seq]`. For seq=4096 with 32 heads in FP16, this is `4096 × 4096 × 32 × 2 bytes = 1 GB` per batch element — larger than L2 cache on any current GPU. The attention scores must be written to HBM, then read back for softmax, then written again, then read again for the value projection. The kernel is severely memory-bandwidth bound.

The compute intensity of naive attention is:
- FLOPs: O(seq² × d_k) per head
- Memory accesses: O(seq² + seq × d_v) per head for storing/loading the score matrix

For seq=4096, d_k=128: FLOPs ≈ 4.3 GFLOP/head, memory ≈ 134 MB/head (for scores alone). Arithmetic intensity ≈ 32 FLOP/byte — well below the ridge point on an A100 (arithmetic intensity for peak efficiency ≈ 208 FLOP/byte for FP16). The kernel cannot saturate the tensor cores.

### 179.8.2 FlashAttention 2 Algorithm

FlashAttention (Dao et al., ICLR 2022) and FlashAttention-2 (NeurIPS 2023) eliminate the O(seq²) materialisation via a tiled computation with online softmax normalisation.

**Setup.** Partition Q into tiles Q_i (size B_r × d_k) and K, V into tiles K_j, V_j (size B_c × d_k), where B_r and B_c are chosen so that two tiles of K/V and the output accumulator O fit in SRAM simultaneously.

**Online softmax recurrence.** For each query tile Q_i, iterate over all key tiles K_j:

1. Compute the raw score tile: S_{ij} = Q_i K_j^T / sqrt(d_k), shape B_r × B_c
2. Compute the row-wise maximum of S_{ij}: m_new = max(m_prev, rowmax(S_{ij}))
3. Compute the softmax numerator with shifted exponents: P_{ij} = exp(S_{ij} - m_new)
4. Update the running denominator: l_new = exp(m_prev - m_new) * l_prev + rowsum(P_{ij})
5. Correct the accumulated output: O_new = diag(exp(m_prev - m_new)) * O_prev + P_{ij} V_j

After all K/V tiles are processed: O_final = O / l (rescale by final denominator).

This recurrence is numerically equivalent to the standard softmax because the `exp(m_prev - m_new)` correction terms cancel out the shifting across all tiles. The critical property: the recurrence can be maintained with O(B_r) scalars (the running max m and sum l), so no O(seq²) tensor is ever written to global memory.

FlashAttention-2 improves the parallelism partitioning to assign Q tiles to thread blocks (rather than K/V tiles in FA1), reducing inter-block communication and achieving better GPU occupancy. It also handles causal masking by skipping K/V tiles that are entirely in the future (below the causal diagonal).

### 179.8.3 FlashAttention 3 on Hopper

FlashAttention 3 (Shah et al., 2024) introduces two Hopper-specific optimisations that overlap the two main bottlenecks: HBM loads and WGMMA compute.

**Producer-consumer warpgroup pipeline.** Hopper's Thread Block Clusters allow explicit producer-consumer separation across warpgroups within a block:

```
Producer warpgroup 0:
  └── nvgpu.tma_async_load: DMA tile K_{j+1} from HBM → smem (ping buffer)
  └── nvgpu.tma_async_load: DMA tile V_{j+1} from HBM → smem (pong buffer)

Consumer warpgroup 1:
  └── nvgpu.warpgroup_mma: compute S_{ij} = Q_i K_j^T (from smem, prev buffer)
  └── softmax recurrence on S_{ij} (register-resident)
  └── nvgpu.warpgroup_mma: compute O_{ij} += P_{ij} V_j (from smem, prev buffer)
```

The `nvgpu.warpgroup_mma` op dispatches a Hopper `wgmma.mma_async` instruction, which runs the 64×256×16 MMA asynchronously while the TMA DMA runs in parallel. This achieves near-complete overlap of compute and memory, pushing throughput to ~1200 TFLOP/s FP16 on H100 SXM5 — compared to ~650 TFLOP/s for FlashAttention-2.

**FP8 mixed precision (FA3 variant).** The QK^T matmul uses FP8 E4M3 (higher throughput), while the PV matmul uses FP16 (maintaining output quality). Per-tile scale factors correct the FP8 quantisation error.

### 179.8.4 The MLIR Lowering Path for Fused Attention

A tiled MLIR implementation of FlashAttention would use the Transform dialect to drive the fusion:

```mlir
// Phase 1: tile Q along seq dimension for output tiles
transform.sequence failures(propagate) {
^bb0(%module: !transform.any_op):
  // Match the outer QK^T matmul
  %qk_matmul = transform.structured.match ops{["linalg.batch_matmul"]}
      in %module : (!transform.any_op) -> !transform.any_op
  // Tile seq dimension with B_r = 128 (fits in SRAM with K, V tiles)
  %tiled, %loops = transform.structured.tile_using_forall %qk_matmul
      tile_sizes [1, 128, 0, 0]       // [batch, seq_q, seq_k, d_k]
      mapping = [#gpu.block<x>, #gpu.block<y>]
      : (!transform.any_op) -> (!transform.any_op, !transform.any_op)

  // Fuse the softmax reduce + exp elementwise into the tiled loop
  transform.structured.fuse_into_containing_op %softmax_reduce into %loops#0
      : (!transform.any_op, !transform.any_op) -> !transform.any_op

  // Vectorize the inner kernel
  transform.structured.vectorize_children_and_apply_patterns %tiled
      : !transform.any_op
}
```

After tiling and vectorisation, the inner kernel contains `vector.contract` ops for QK^T and PV multiplications, scalar `math.exp` / `arith.maxf` / `arith.addf` ops for the online softmax recurrence, and `vector.transfer_read/write` for shared memory staging.

The Triton path (cross-ref Ch163 — Triton) reaches equivalent fusion by writing the tiled loop directly in the Triton programming model, which compiles through the `triton_gpu` dialect to the same `nvvm` / PTX level. Triton's `tl.dot` corresponds to `vector.contract`; `tl.load` with block pointers corresponds to `vector.transfer_read` from shared memory.

### 179.8.5 What MLIR Enables vs What It Does Not (Yet)

As of LLVM 22, the fully automatic path — input: `stablehlo.dot_general` + `stablehlo.reduce` for softmax; output: FlashAttention-quality PTX — does not exist in the open-source stack. Two gaps remain:

1. **Fusion heuristics.** The `linalg` elementwise fusion pass (`--linalg-fuse-elementwise-ops`) fuses producer-consumer pairs that share a loop dimension. But it does not recognise the softmax reduction pattern as fusible with the adjacent matmul, because softmax reduces across the inner dimension that the matmul produces — a reduction-into-producer pattern that requires the online normalisation transformation before fusion becomes safe. This transform is not currently in the `transform` dialect library.

2. **Shared memory scheduling.** Generating the WGMMA + TMA producer-consumer pipeline requires target-aware passes that expose Hopper's async programming model. This is implemented in XLA:GPU's `--xla-gpu-enable-async-all-reduce` and `--xla-gpu-use-runtime-fusion` passes, and partially in IREE's CUDA codegen via `iree-codegen-llvm-gpu-lower-to-llvm`, but not as general-purpose MLIR passes that any compiler can reuse.

The practical production paths are: XLA:GPU (built-in FlashAttention-equivalent fusion for the attention pattern, triggered by `AttentionRewriter`), cuDNN Graph API (`CUDNN_FRONTEND_GRAPH` with Flash attention node), and custom Triton kernels (the path used by PyTorch's `F.scaled_dot_product_attention` with `torch.backends.cuda.sdp_kernel()`).

---

## 179.9 End-to-End Traced Walkthrough: Linear Layer to PTX

This section traces `y = xW^T + b` — a single linear layer — from PyTorch Python to PTX assembly, with abbreviated MLIR at the critical stages.

### Step 1 — `torch.export` → `ExportedProgram`

```python
import torch
from torch.export import export

class Linear(torch.nn.Module):
    def __init__(self):
        super().__init__()
        self.fc = torch.nn.Linear(784, 128)

    def forward(self, x):
        return self.fc(x)

model = Linear().eval()
exported = export(model, (torch.randn(1, 784),))
# exported.graph contains:
#   %x      = placeholder[target=x]
#   %weight = get_attr[target=fc.weight]   (tensor<128x784xf32>)
#   %bias   = get_attr[target=fc.bias]     (tensor<128xf32>)
#   %linear = aten.linear.default(%x, %weight, %bias)
#   return (%linear,)
```

The `ExportedProgram` captures the FX graph with `aten.linear` as a single node. Weight and bias tensors are static constants lifted to graph inputs. The input batch dimension can be made dynamic with `Dim("batch")`.

### Step 2 — torch-mlir → `stablehlo.dot_general` + `stablehlo.add`

torch-mlir's `TorchToStableHLO` pass (cross-ref Ch161 — torch-mlir, ONNX-MLIR) converts `torch.aten.linear` to a transposed `dot_general` plus broadcast-add:

```mlir
// After TorchToStableHLO conversion
func.func @forward(%x: tensor<1x784xf32>,
                   %weight: tensor<128x784xf32>,
                   %bias: tensor<128xf32>) -> tensor<1x128xf32> {
  // aten::linear(x, W, b) = x @ W^T + b
  // contracting over dim 1 of x and dim 1 of W (the transposed dim)
  %dot = stablehlo.dot_general %x, %weight,
      contracting_dims = [1] x [1]
      : (tensor<1x784xf32>, tensor<128x784xf32>) -> tensor<1x128xf32>
  %bias_bcast = stablehlo.broadcast_in_dim %bias, dims = [1]
      : (tensor<128xf32>) -> tensor<1x128xf32>
  %result = stablehlo.add %dot, %bias_bcast
      : tensor<1x128xf32>
  return %result : tensor<1x128xf32>
}
```

Cross-ref Ch154 — HLO and StableHLO for `dot_general` contracting-dimension semantics.

### Step 3 — StableHLO → `linalg.matmul` + `linalg.map`

The `--stablehlo-legalize-to-linalg` pass (cross-ref Ch152 — Lowering Pipelines) converts named HLO operators to structured `linalg` ops:

```mlir
// After stablehlo → linalg conversion
func.func @forward(%x: tensor<1x784xf32>,
                   %weight: tensor<128x784xf32>,
                   %bias: tensor<128xf32>) -> tensor<1x128xf32> {
  %empty  = tensor.empty() : tensor<1x128xf32>
  %zero_f = arith.constant 0.0 : f32
  %init   = linalg.fill ins(%zero_f : f32)
                        outs(%empty : tensor<1x128xf32>) -> tensor<1x128xf32>

  // dot_general(contracting=[1]x[1]) maps to linalg.matmul with transposed rhs
  %matmul = linalg.matmul
      ins(%x, %weight : tensor<1x784xf32>, tensor<128x784xf32>)
      outs(%init : tensor<1x128xf32>) -> tensor<1x128xf32>

  // bias add: broadcast %bias along batch dimension, then elementwise add
  %empty2    = tensor.empty() : tensor<1x128xf32>
  %bias_2d   = linalg.broadcast ins(%bias : tensor<128xf32>)
                                 outs(%empty2 : tensor<1x128xf32>)
                                 dimensions = [0]
  %result    = linalg.map { arith.addf }
               ins(%matmul, %bias_2d : tensor<1x128xf32>, tensor<1x128xf32>)
               outs(%empty2 : tensor<1x128xf32>)
  return %result : tensor<1x128xf32>
}
```

### Step 4 — Tiling via Transform Dialect

The Transform dialect drives tiling for GPU dispatch:

```mlir
transform.sequence failures(propagate) {
^bb0(%func: !transform.any_op):
  %matmul = transform.structured.match ops{["linalg.matmul"]} in %func
      : (!transform.any_op) -> !transform.any_op
  // Tile for GPU: 64×64 output tiles mapped to thread blocks
  %tiled, %loops:3 = transform.structured.tile_using_forall %matmul
      tile_sizes [64, 64, 32]
      mapping = [#gpu.block<y>, #gpu.block<x>]
      : (!transform.any_op) -> (!transform.any_op, !transform.any_op)
  // Promote A and B tiles to shared memory
  transform.structured.promote %tiled { operands_to_promote = [0, 1] }
      : !transform.any_op
}
```

This produces nested `scf.forall` loops with `tensor.extract_slice` / `tensor.insert_slice` operations carving 64×64×32 sub-tensors, and `bufferization.alloc_tensor` for shared memory staging.

### Step 5 — Vectorisation → `vector.contract`

After `transform.structured.vectorize_children_and_apply_patterns`:

```mlir
// Vectorized inner kernel: 16×16 register tile per warp
%va = vector.transfer_read %a_smem[%i16, %k16], %c0_f32
    {in_bounds = [true, true]} : memref<64x32xf32, #gpu.address_space<workgroup>>,
                                  vector<16x16xf32>
%vb = vector.transfer_read %b_smem[%k16, %j16], %c0_f32
    {in_bounds = [true, true]} : memref<32x64xf32, #gpu.address_space<workgroup>>,
                                  vector<16x16xf32>
%vc = vector.transfer_read %c_reg[%i16, %j16], %c0_f32
    {in_bounds = [true, true]} : memref<64x64xf32, #gpu.address_space<private>>,
                                  vector<16x16xf32>

%vc_new = vector.contract
    {indexing_maps = [affine_map<(m,n,k)->(m,k)>,
                      affine_map<(m,n,k)->(k,n)>,
                      affine_map<(m,n,k)->(m,n)>],
     iterator_types = ["parallel", "parallel", "reduction"],
     kind = #vector.kind<add>}
    %va, %vb, %vc : vector<16x16xf32>, vector<16x16xf32> into vector<16x16xf32>

vector.transfer_write %vc_new, %c_reg[%i16, %j16]
    {in_bounds = [true, true]} : vector<16x16xf32>,
                                  memref<64x64xf32, #gpu.address_space<private>>
```

Cross-ref Ch141 — Vector and Sparse for `vector.transfer_read/write` and `vector.contract` semantics.

### Step 6 — One-Shot Bufferization → `memref`

```bash
mlir-opt --one-shot-bufferize="bufferize-function-boundaries allow-return-allocs" ...
```

`tensor.extract_slice` → `memref.subview`; `tensor.insert_slice` → elided (in-place where possible) or `memref.copy`. `linalg.fill` → `memref.fill`. The `vector.transfer_read/write` ops now operate on `memref<...>` with address-space attributes (`#gpu.address_space<workgroup>` for shared memory, `#gpu.address_space<private>` for registers). Cross-ref Ch151 — Bufferization Deep Dive.

### Step 7 — `nvvm` Dialect → PTX

The final lowering pipeline applies:

```bash
mlir-opt \
  --convert-gpu-to-nvvm \
  --convert-nvvm-to-llvm \
  --nvvm-attach-target="chip=sm_80,fast=true,ftz=true" \
  --serialize-to-cubin \
  ...
```

Key transformations:
- `gpu.block_id x/y/z` → `nvvm.read.ptx.sreg.ctaid.x` → PTX `%ctaid.x`
- `gpu.thread_id x` → `nvvm.read.ptx.sreg.tid.x` → PTX `%tid.x`
- `gpu.barrier` → `nvvm.bar.warp.sync` → PTX `bar.warp.sync 0xffffffff`
- `vector.contract` (16×16×16, f32) → `nvgpu.mma.sync` → PTX:

```
// PTX for mma.sync.aligned.m16n8k8 (Ampere, FP32 accumulation)
mma.sync.aligned.m16n8k8.row.col.f32.tf32.tf32.f32
    {%acc0, %acc1, %acc2, %acc3},
    {%a0, %a1, %a2, %a3},
    {%b0, %b1},
    {%acc0, %acc1, %acc2, %acc3};
```

The PTX is compiled by `ptxas` to a CUBIN embedded in the IREE `.vmfb` FlatBuffer or an ONNX Runtime CUDA EP engine cache. At runtime, `cuModuleLoadData` maps the CUBIN into the device context; `cuLaunchKernel` dispatches execution with the grid and block dimensions computed from the tiling parameters.

---

## Chapter Summary

- **The four-level hierarchy** — eager, traced capture, middle-layer MLIR, target codegen — determines what information is available at each stage; the handoff formats are `ExportedProgram` (PyTorch), JAX jaxpr, and StableHLO; all converge on the same MLIR dialect tower.

- **The dialect tower** proceeds `stablehlo`/`tosa` → `linalg.generic` → `vector.contract` → `memref` → `nvvm`/`rocdl`/LLVM IR; the bufferisation boundary between `tensor` and `memref` is the key structural divide that must be respected by all passes.

- **Quantisation** has three entry points: the `quant` dialect type system for static type annotations, `stablehlo.uniform_quantize/dequantize` for XLA/IREE pipelines, and direct intrinsic lowering for GPTQ and GGUF; formats range from INT8 symmetric to FP8 E4M3/E5M2 with distinct lowering paths for each.

- **GPTQ** quantises LLM weights using Hessian-corrected group quantisation; the INT4 GGUF `Q4_K_M` format stores two-level group scales alongside packed 4-bit weights; both require custom dequantise kernels not expressible through the standard `quant` dialect.

- **Dynamic shapes** are represented via `?` dimensions, `tensor.dim`, and the `shape` dialect; IREE handles them via runtime-computed `flow.dispatch.workgroups`; `torch.export` carries symbolic `Dim` constraints from framework to compiler.

- **Target landscape**: CUDA/ROCm have deep open-source MLIR paths through `nvgpu`/`amdgpu` dialects with `nvgpu.mma.sync`/`nvgpu.warpgroup_mma` and `amdgpu.mfma`; ANE and most embedded NPUs require vendor frontend formats as the hardware-facing layer.

- **Deployment runtime selection** follows a clear decision tree: TensorRT for NVIDIA-only maximum throughput; IREE for cross-platform portability; TFLite for mobile NPU/MCU; ONNX Runtime for vendor EP flexibility.

- **Training compilation** differs from inference in gradient correctness requirements, activation checkpointing (JAX `jax.checkpoint` policies: `dots_saveable`, `nothing_saveable`), and distributed collective communication via `stablehlo.all_reduce`/`all_gather`.

- **FlashAttention** demonstrates that high-performance fused kernels still require framework-specific fusion passes (XLA `AttentionRewriter`), cuDNN Graph API, or manually written Triton kernels; the fully automatic MLIR path does not yet implement the online softmax normalisation transform or the WGMMA + TMA producer-consumer pipeline as general reusable passes.

- **The linear-layer walkthrough** makes the seven transformation stages concrete: `aten.linear` → `stablehlo.dot_general` → `linalg.matmul` → tiled `scf.forall` with shared-memory promotion → `vector.contract` → bufferised `memref` → `nvgpu.mma.sync` → PTX `mma.sync.aligned`.
