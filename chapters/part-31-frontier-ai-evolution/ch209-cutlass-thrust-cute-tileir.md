# Chapter 209 — CUTLASS, Thrust, CuTe, and TileIR: GPU Parallel Primitives and Layout Algebra

*Part XXXI — Frontier AI Evolution*

GPU programming stacks have stratified into a hierarchy of abstraction layers, and understanding each layer — and how they compose — is essential for building, analysing, or generating high-performance AI kernels programmatically. [Chapter 102 — NVPTX and the CUDA Path](../part-15-targets/ch102-nvptx-and-the-cuda-path.md) covers the base metal: how LLVM emits PTX for NVIDIA hardware. [Chapter 164 — CUDA Tile IR](../part-23-mlir-production/ch164-cuda-tile-ir.md) covers the MLIR-level tile abstractions for Hopper and the `nvgpu` dialect. This chapter fills the layer between hardware PTX and MLIR: Thrust (a GPU analogue of the C++ standard library), CUTLASS (template-metaprogramming and then algebraic GEMM), CuTe (the layout algebra underlying CUTLASS 3.x and 4.x), and TileIR (MLIR's evolving tile-computation IR). The chapter concludes with the epistemological payoff: layout algebra functions as a *type system for kernel-level self-modification* — a foundation for the programmatic generation of type-correct tile strategies explored in [Chapter 211 — Neural Programs as Compiled Artifacts](ch211-neural-programs-as-compiled-artifacts.md) and [Chapter 215 — Evolutionary Architecture Search](ch215-evolutionary-architecture-search.md).

---

## Table of Contents

- [209.1 The GPU Parallel Primitives Stack](#2091-the-gpu-parallel-primitives-stack)
- [209.2 Thrust: GPU Algorithms as a Standard Library](#2092-thrust-gpu-algorithms-as-a-standard-library)
  - [209.2.1 Device and Host Vectors](#20921-device-and-host-vectors)
  - [209.2.2 Core Algorithms](#20922-core-algorithms)
  - [209.2.3 Execution Policies and Backend Switching](#20923-execution-policies-and-backend-switching)
  - [209.2.4 Attention Score Statistics via Thrust](#20924-attention-score-statistics-via-thrust)
  - [209.2.5 When Thrust Suffices, When It Doesn't](#20925-when-thrust-suffices-when-it-doesnt)
- [209.3 CUTLASS: Template-Metaprogramming GEMM](#2093-cutlass-template-metaprogramming-gemm)
  - [209.3.1 CUTLASS 2.x: The Template Signature Problem](#20931-cutlass-2x-the-template-signature-problem)
  - [209.3.2 Why CUTLASS 3.x Exists](#20932-why-cutlass-3x-exists)
- [209.4 CuTe: Layout Algebra for Typed Tensor Operations](#2094-cute-layout-algebra-for-typed-tensor-operations)
  - [209.4.1 Layouts as Compile-Time Types](#20941-layouts-as-compile-time-types)
  - [209.4.2 Layout Composition](#20942-layout-composition)
  - [209.4.3 Layout Complementation and Tiling](#20943-layout-complementation-and-tiling)
  - [209.4.4 Tensors: Typed Tile Views](#20944-tensors-typed-tile-views)
  - [209.4.5 TiledMMA and ThrMMA: Warp-Collective Partitioning](#20945-tiledmma-and-thrmma-warp-collective-partitioning)
  - [209.4.6 TiledCopy: Efficient Tile Loading](#20946-tiledcopy-efficient-tile-loading)
  - [209.4.7 CuTe DSL: Python-Level Layout Algebra (CUTLASS 4.x)](#20947-cute-dsl-python-level-layout-algebra-cutlass-4x)
  - [209.4.8 Connection to PTX Hardware Instructions](#20948-connection-to-ptx-hardware-instructions)
- [209.5 Linear Layouts and the F₂ Foundation](#2095-linear-layouts-and-the-f-foundation)
  - [209.5.1 F₂-Linear Maps as a Unification](#20951-f-linear-maps-as-a-unification)
  - [209.5.2 F₂ Encoding of Concrete Layouts](#20952-f-encoding-of-concrete-layouts)
  - [209.5.3 Interoperability Implications](#20953-interoperability-implications)
  - [209.5.4 Categorical Foundations](#20954-categorical-foundations)
- [209.6 TileIR: Tiled Computation as a First-Class MLIR Dialect](#2096-tileir-tiled-computation-as-a-first-class-mlir-dialect)
  - [209.6.1 Tile IR Op Surface](#20961-tile-ir-op-surface)
  - [209.6.2 Lowering Path: TileIR → GPU Dialect → LLVM](#20962-lowering-path-tileir-gpu-dialect-llvm)
  - [209.6.3 TileIR Verification and Error Reporting](#20963-tileir-verification-and-error-reporting)
  - [209.6.4 Relationship to CuTe](#20964-relationship-to-cute)
  - [209.6.5 Attention via TileIR](#20965-attention-via-tileir)
- [209.7 Layout Algebra as a Type System for Self-Modification](#2097-layout-algebra-as-a-type-system-for-self-modification)
  - [209.7.1 Layout Types as Compile-Time Constraints](#20971-layout-types-as-compile-time-constraints)
  - [209.7.2 Searching Over Layout Spaces](#20972-searching-over-layout-spaces)
  - [209.7.3 Register Pressure as a Layout Constraint](#20973-register-pressure-as-a-layout-constraint)
  - [209.7.4 Connection to Evolutionary Architecture Search](#20974-connection-to-evolutionary-architecture-search)
- [Chapter Summary](#chapter-summary)

---

## 209.1 The GPU Parallel Primitives Stack

The modern NVIDIA GPU programming stack decomposes into five layers:

```
┌─────────────────────────────────────────────────────────────┐
│  TileIR (MLIR dialect)  ─  tile.func / tile.parallel        │
│  CuTe DSL (Python, CUTLASS 4.x)  ─  @cute.kernel            │
├─────────────────────────────────────────────────────────────┤
│  CuTe C++ layout algebra  ─  Layout<Shape,Stride>           │
│  CUTLASS 3.x  ─  TiledMMA, ThrFrag                          │
│  Thrust  ─  thrust::sort, thrust::reduce                    │
├─────────────────────────────────────────────────────────────┤
│  CUDA C++ / CUTLASS 2.x  ─  template-metaprogramming GEMM   │
├─────────────────────────────────────────────────────────────┤
│  CUDA Runtime / PTX  ─  wmma, mma, ldmatrix                 │
├─────────────────────────────────────────────────────────────┤
│  Hardware: SM, tensor cores, shared memory, TMA, WGMMA      │
└─────────────────────────────────────────────────────────────┘
```

Each layer introduces a new abstraction boundary. Thrust hides thread-level parallelism behind STL-style algorithms. CUTLASS 2.x hides warp-level tiling behind C++ template parameters. CuTe replaces those parameters with an algebraic type: a `Layout<Shape, Stride>`. TileIR lifts layout-typed tensors into a proper IR with explicit tile structure that a compiler can reason about and a generative system can emit.

For AI kernel development, the relevant question at each layer is: *what invariants does this layer enforce, and can those invariants be checked at compile time before execution?* A `Layout` mismatch is a compile-time type error in CuTe; a tile shape mismatch in TileIR is an IR verification error before codegen. These enforcement points are what make layout algebra useful for programmatic kernel construction.

The table below summarises typical usage patterns at each layer:

| Layer | Primary Abstraction | Key Operations | When to Use |
|-------|---------------------|----------------|-------------|
| Thrust | `device_vector`, iterator | `reduce`, `transform`, `scan`, `sort` | Data preprocessing, statistics, host↔device movement |
| CUTLASS 2.x | `Gemm<...>` template | GEMM launch, epilogue fusion | Rapid deployment of tuned GEMM on Ampere/Volta |
| CUTLASS 3.x | `TiledMMA`, `TiledCopy` | Layout-typed collective GEMM/attention | Hopper WGMMA kernels; custom epilogues |
| CuTe C++ | `Layout<Shape,Stride>` | `composition`, `tiled_divide`, `partition_A/B/C` | Writing or generating custom tile kernels |
| CuTe DSL (4.x) | Python layout API | `@cute.kernel`, JIT compile | Programmatic kernel generation and autotuning |
| TileIR (MLIR) | `tile.func`, `tile.buffer` | Lowering via `nvgpu`, compiler analysis | Compiler-driven kernel generation, generative AI systems |

The demarcation between "use Thrust" and "write CuTe" is primarily determined by whether the computation requires shared memory reuse across threads. Once a computation requires a thread block to cooperate on a tile — loading it into shared memory, computing on it collectively, and writing back — the overhead of programming with raw CUDA intrinsics makes CuTe's layout algebra the productivity-preserving choice.

---

## 209.2 Thrust: GPU Algorithms as a Standard Library

Thrust ([github.com/NVIDIA/thrust](https://github.com/NVIDIA/thrust), now part of the CCCL — CUDA Core Compute Libraries) is NVIDIA's parallel algorithms library, providing GPU implementations of STL-style algorithms with identical interfaces. Since CUDA 12.3, Thrust ships as part of [nvidia/cccl](https://github.com/NVIDIA/cccl).

### 209.2.1 Device and Host Vectors

The entry point to Thrust is its vector types:

```cpp
#include <thrust/device_vector.h>
#include <thrust/host_vector.h>

// Allocate 1024 floats on host (pageable memory)
thrust::host_vector<float> h_vec(1024, 1.0f);

// Transfer to device — Thrust handles cudaMemcpy internally
thrust::device_vector<float> d_vec = h_vec;

// Modify on device, transfer back
thrust::transform(d_vec.begin(), d_vec.end(), d_vec.begin(),
                  [] __device__ (float x) { return x * 2.0f; });

thrust::host_vector<float> h_result = d_vec;  // implicit D→H copy
```

`thrust::device_vector` allocates device memory via `cudaMalloc` and manages lifetime via RAII. Assignment between host and device vectors triggers `cudaMemcpy`. Raw device pointers are accessible via `thrust::raw_pointer_cast`:

```cpp
float* raw_ptr = thrust::raw_pointer_cast(d_vec.data());
my_cuda_kernel<<<grid, block>>>(raw_ptr, 1024);
```

This interoperability boundary — Thrust containers to raw pointers — is where Thrust code connects to hand-written CUDA kernels.

### 209.2.2 Core Algorithms

**Reduction and transformation:**

```cpp
#include <thrust/reduce.h>
#include <thrust/transform_reduce.h>
#include <thrust/functional.h>

// Simple sum reduction: O(n) work, O(log n) depth
float total = thrust::reduce(d_vec.begin(), d_vec.end(), 0.0f,
                              thrust::plus<float>());

// Transform then reduce: computes sum of x^2 in a single pass
float sum_sq = thrust::transform_reduce(
    d_vec.begin(), d_vec.end(),
    [] __device__ (float x) { return x * x; },  // map
    0.0f,                                         // identity
    thrust::plus<float>());                       // reduce
```

`thrust::transform_reduce` fuses the map and reduce, avoiding a round-trip through global memory. For attention score statistics (softmax numerics, layer norm variance), this is the right primitive.

**Inclusive scan (prefix sum):**

```cpp
#include <thrust/scan.h>

// In-place inclusive prefix sum: d_out[i] = sum(d_in[0..i])
thrust::inclusive_scan(d_in.begin(), d_in.end(), d_out.begin());
```

Prefix scans underpin histogram-based sorting, sparse tensor accumulation, and load balancing in GPU work queues.

**Sorting:**

```cpp
#include <thrust/sort.h>

// Sort by key, reordering values in lockstep
thrust::sort_by_key(d_keys.begin(), d_keys.end(), d_values.begin());
```

**Fused transform over zip views:**

```cpp
#include <thrust/zip_iterator.h>

// Compute elementwise dot products of pairs (a[i], b[i])
auto begin = thrust::make_zip_iterator(thrust::make_tuple(
    d_a.begin(), d_b.begin()));
auto end = thrust::make_zip_iterator(thrust::make_tuple(
    d_a.end(), d_b.end()));

thrust::transform(begin, end, d_dot.begin(),
    [] __device__ (thrust::tuple<float, float> t) {
        return thrust::get<0>(t) * thrust::get<1>(t);
    });
```

### 209.2.3 Execution Policies and Backend Switching

Thrust's execution policy separates algorithm semantics from execution location:

```cpp
#include <thrust/execution_policy.h>
#include <thrust/system/cuda/execution_policy.h>

cudaStream_t stream;
cudaStreamCreate(&stream);

// Async execution on a specific CUDA stream
thrust::reduce(thrust::cuda::par.on(stream),
               d_vec.begin(), d_vec.end(), 0.0f);

// CPU execution via OpenMP (same algorithm, different policy)
thrust::reduce(thrust::omp::par,
               h_vec.begin(), h_vec.end(), 0.0f);

// CPU execution via TBB
thrust::reduce(thrust::tbb::par,
               h_vec.begin(), h_vec.end(), 0.0f);
```

The same algorithm code runs on CUDA, OpenMP, or TBB by swapping the execution policy. This matters for testing and portability: a correctness test running on the CPU with `thrust::omp::par` can validate the same computation that runs on GPU with `thrust::cuda::par`.

### 209.2.4 Attention Score Statistics via Thrust

A concrete AI use case: computing per-head softmax temperature statistics for calibration before deploying a quantised attention kernel. Given `d_scores` (a flattened `[batch × heads × seq_len]` tensor of raw attention logits on device):

```cpp
#include <thrust/device_vector.h>
#include <thrust/transform_reduce.h>
#include <thrust/functional.h>
#include <thrust/extrema.h>
#include <thrust/execution_policy.h>
#include <cmath>

cudaStream_t stream;
cudaStreamCreate(&stream);
auto policy = thrust::cuda::par.on(stream);

int B = 2, H = 32, S = 2048;
int total = B * H * S;

thrust::device_vector<float> d_scores(total);
// ... fill d_scores from the model forward pass ...

// Mean
float mean = thrust::reduce(policy, d_scores.begin(), d_scores.end(), 0.0f)
             / static_cast<float>(total);

// Variance: E[x^2] - mean^2
float mean_sq = thrust::transform_reduce(
    policy, d_scores.begin(), d_scores.end(),
    [mean] __device__ (float x) { float d = x - mean; return d * d; },
    0.0f, thrust::plus<float>()) / static_cast<float>(total);

// Max absolute value (for clipping analysis)
auto max_it = thrust::max_element(policy, d_scores.begin(), d_scores.end());
float max_val = *max_it;

printf("mean=%.4f  var=%.4f  max=%.4f\n", mean, mean_sq, max_val);

// Threshold logits: clip to [-max_clip, max_clip] before softmax
float max_clip = 50.0f;
thrust::transform(policy, d_scores.begin(), d_scores.end(), d_scores.begin(),
    [max_clip] __device__ (float x) {
        return fminf(fmaxf(x, -max_clip), max_clip);
    });
```

The three reductions (mean, variance, max) each launch a separate CUDA kernel. Thrust does not automatically fuse them. For a single-pass fused statistics kernel, a hand-written reduction with `__shfl_xor_sync` warp reductions is needed. Thrust is the right tool here as long as the three-kernel overhead is acceptable — which it is for calibration (one-time, not on the critical path).

### 209.2.5 When Thrust Suffices, When It Doesn't

Thrust excels at:
- Standard reductions (sum, max, argmax) without custom memory access patterns
- Data movement and transformation pipelines where operations are elementwise or reduction-shaped
- Rapid prototyping where kernel launch overhead is acceptable
- Host–device data movement via typed container assignment

Thrust is insufficient for:
- Fused attention (QK^T softmax V requires a non-standard reduction across a non-contiguous memory pattern)
- Any kernel that exploits shared memory tiling or register reuse across iterations
- Warp-level primitives (`__shfl_xor_sync`, warp matrix operations via `wmma`)
- Pipelines requiring async copy (TMA), software pipelining, or double-buffering

The performance cliff between `thrust::transform_reduce` and a hand-fused FlashAttention kernel is 10–20× for attention on long sequences because Thrust cannot express shared-memory-resident intermediate results. When the access pattern has no standard library equivalent, the right layer is CUTLASS or CuTe directly.

---

## 209.3 CUTLASS: Template-Metaprogramming GEMM

CUTLASS ([github.com/NVIDIA/cutlass](https://github.com/NVIDIA/cutlass)) is NVIDIA's GEMM library, providing highly tuned implementations of matrix multiplication and related operations. Its history in two major versions illustrates the transition from template-soup abstraction to algebraic abstraction.

### 209.3.1 CUTLASS 2.x: The Template Signature Problem

CUTLASS 2.x uses nested template specialization to express the entire tile hierarchy as template parameters. The top-level `cutlass::gemm::device::Gemm<>` type:

```cpp
using GemmOp = cutlass::gemm::device::Gemm<
  cutlass::half_t,                               // ElementA
  cutlass::layout::RowMajor,                     // LayoutA
  cutlass::half_t,                               // ElementB
  cutlass::layout::ColumnMajor,                  // LayoutB
  cutlass::half_t,                               // ElementC (output)
  cutlass::layout::RowMajor,                     // LayoutC
  float,                                         // ElementAccumulator
  cutlass::arch::OpClassTensorOp,                // OperatorClass
  cutlass::arch::Sm80,                           // ArchTag (Ampere)
  cutlass::gemm::GemmShape<128, 256, 32>,        // ThreadblockShape
  cutlass::gemm::GemmShape<64, 64, 32>,          // WarpShape
  cutlass::gemm::GemmShape<16, 8, 16>,           // InstructionShape
  cutlass::epilogue::thread::LinearCombination<  // EpilogueOp
      cutlass::half_t, 8, float, float>,
  cutlass::gemm::threadblock::GemmIdentityThreadblockSwizzle<>,
  3,                                             // Stages (pipeline depth)
  8,                                             // AlignmentA
  8                                              // AlignmentB
>;
```

This instantiation is a single C++ type — the full GEMM kernel. Every combinatorial choice (precision, layout, tile shape, pipeline depth) is a separate template parameter. Changing from Ampere to Hopper requires replacing `Sm80` with `Sm90` and adjusting `InstructionShape` to reflect WGMMA, but because the parameter names do not encode their dependencies, the compiler error messages for mismatched shapes are several kilobytes of template-instantiation backtrace.

Running a CUTLASS 2.x GEMM on device:

```cpp
GemmOp gemm_op;
GemmOp::Arguments args{
    {M, N, K},                         // problem size
    {A_ptr, lda}, {B_ptr, ldb},        // input matrices with leading dimensions
    {C_ptr, ldc}, {D_ptr, ldd},        // source and destination
    {alpha, beta}                       // epilogue scalars: D = alpha*A*B + beta*C
};
size_t workspace_bytes = GemmOp::get_workspace_size(args);
void* workspace = /* cudaMalloc(workspace_bytes) */;
gemm_op.initialize(args, workspace);
gemm_op();  // launch kernel
```

The abstraction hierarchy underneath this type:

```
GemmUniversal (device-level, handles argument batching and workspace)
  └─ GemmKernel (kernel-level, coordinates grid launch dimensions)
       └─ ThreadblockMma (K-loop over shared-memory tiles)
            └─ WarpMma (warp-level issue of MMA instructions)
                 └─ MmaPolicy (instruction selection: wmma/mma.sync/sm90 WGMMA)
```

Each layer is a template that accepts the layer below. Instantiating with Hopper (`Sm90`) requires replacing the `MmaPolicy` with one that emits WGMMA instructions and changing `InstructionShape` to `<64, 128, 16>`. But the three integer shapes (`ThreadblockShape`, `WarpShape`, `InstructionShape`) have a correctness invariant that is implicit: `ThreadblockShape` must be a multiple of `WarpShape` along each axis, which must be a multiple of `InstructionShape`. A violation produces several hundred lines of template-instantiation failure from inside `cutlass::gemm::threadblock::MmaBase<...>::check_shape` — unnavigable without intimate CUTLASS knowledge.

### 209.3.2 Why CUTLASS 3.x Exists

CUTLASS 3.x ([github.com/NVIDIA/cutlass, tag v3.x](https://github.com/NVIDIA/cutlass)) replaces the template parameter approach with CuTe's layout algebra. The key insight: *a tile shape is not an integer triple — it is a mapping from logical coordinates to physical memory offsets*. Once that mapping is a first-class compile-time type (`Layout<Shape, Stride>`), the correctness conditions for composing tiles become type-checking conditions.

The CUTLASS 3.x equivalent:

```cpp
using TileShape = Shape<_128, _256, _32>;  // BlockM, BlockN, BlockK

using TiledMma = TiledMMA<
    MMA_Atom<SM90_64x256x16_F32F16F16F32_SS>,  // WGMMA atom
    Layout<Shape<_4, _1, _1>>,                  // warpgroup tiling
    Tile<_64, _256, _16>>;                      // per-MMA tile size

using GmemTiledCopyA = decltype(make_tiled_copy(...));
using GmemTiledCopyB = decltype(make_tiled_copy(...));
```

The tile shapes are `Layout` objects, not integers. The warpgroup tiling is a `Layout` applied to the MMA atom. Composing two `Layout` objects produces another `Layout` — and if the composition is invalid, the compiler reports a type error at the `Layout` level, not deep in the template hierarchy.

CUTLASS 3.x also restructures the abstraction hierarchy into *collective* operations, reflecting the Hopper programming model where multiple warps cooperate on a single tile:

```
CollectiveMainloop (handles K-loop: TMA loads, warpgroup MMA, mbarrier sync)
  └─ TiledMMA (warpgroup-collective MMA tile)
       └─ MMA_Atom (WGMMA instruction)
CollectiveEpilogue (handles tile store: register→smem→global via TMA)
  └─ TiledCopy (thread-collective store)
```

The CUTLASS 3.x `GemmUniversal` is parameterised on `CollectiveMainloop` and `CollectiveEpilogue` rather than on a flat list of integer shapes. This makes it straightforward to swap the mainloop (e.g., change pipeline depth or switch from 2-warpgroup to 1-warpgroup tiling) without touching the epilogue, because the two are separate type parameters rather than interleaved template arguments.

The critical improvement over CUTLASS 2.x is in error locality. When a `TiledMMA<MMA_Atom<SM90_64x256x16_...>, Layout<Shape<_4,_1,_1>>, Tile<_64,_256,_16>>` is ill-formed because the tile shape doesn't divide the atom output, the error says:

```
static_assert failed: "TileShape_MNK must be a multiple of AtomShape_MNK"
                        in cutlass/gemm/collective/sm90_mma_tma_gmma_ss.hpp:87
```

Rather than a 2000-line backtrace. The layout type has enough structure for the library to pinpoint which dimension is mismatched.

---

## 209.4 CuTe: Layout Algebra for Typed Tensor Operations

CuTe is the mathematical core of CUTLASS 3.x and the theoretical foundation of [arXiv 2603.02298](https://arxiv.org/abs/2603.02298). [Chapter 164](../part-23-mlir-production/ch164-cuda-tile-ir.md) introduced the basic `(Shape, Stride)` pair and `zipped_divide`. This section extends into layout composition, complementation, `TiledMMA`/`ThrMMA` partitioning, and the `Tensor<Engine, Layout>` abstraction.

### 209.4.1 Layouts as Compile-Time Types

A `Layout<Shape, Stride>` is a compile-time bijective function from logical coordinates to a flat integer offset. Both `Shape` and `Stride` are *nested integer tuples* — not flat arrays — enabling hierarchical tile descriptions:

```cpp
#include <cute/layout.hpp>
using namespace cute;

// Flat row-major: 128×64, stride (64, 1)
// offset(i, j) = 64*i + j
auto rm_128x64 = make_layout(make_shape(Int<128>{}, Int<64>{}),
                               make_stride(Int<64>{}, Int<1>{}));

// Column-major: same shape, stride (1, 128)
auto cm_128x64 = make_layout(make_shape(Int<128>{}, Int<64>{}),
                               make_stride(Int<1>{}, Int<128>{}));

// Hierarchical (tile-first) shape:
// shape  = ((4, 32), (2, 32)) means 4 tiles of 32 rows, 2 tiles of 32 cols
// stride = ((32*64, 1), (32, 64)) — tiles contiguous in memory
auto tiled = make_layout(
    make_shape(make_shape(Int<4>{}, Int<32>{}),
               make_shape(Int<2>{}, Int<32>{})),
    make_stride(make_stride(Int<2048>{}, Int<1>{}),
                make_stride(Int<32>{}, Int<64>{})));
```

The `Int<N>{}` wrapper makes dimensions compile-time constants. At instantiation, `rm_128x64(2, 3)` evaluates to `64*2 + 3 = 131` as a constexpr expression — no runtime arithmetic.

### 209.4.2 Layout Composition

Layout composition applies one layout to the output of another. If layout `f` maps `(i, j)` to an offset, and layout `g` maps an offset to a hardware address, then `composition(g, f)` maps `(i, j)` to hardware addresses:

```cpp
// f: 2D tile coordinates -> flat offset within one tile
auto tile_f = make_layout(make_shape(Int<16>{}, Int<8>{}),
                           make_stride(Int<8>{}, Int<1>{}));

// g: flat tile index -> strided global offset
// (tile index * 128 to skip to next tile)
auto global_g = make_layout(make_shape(Int<128>{}),
                              make_stride(Int<128>{}));

// compose(g, f): (2D tile coord) -> global offset
auto composed = composition(global_g, tile_f);
// composed(3, 2) = global_g(tile_f(3, 2)) = global_g(3*8 + 2) = 26*128 = 3328
```

Composition is the mechanism by which multi-level tile hierarchies are constructed without explicitly tracking pointer arithmetic. A threadblock-level layout composed with a warp-level layout gives the offset arithmetic for a per-warp register fragment within the threadblock tile.

### 209.4.3 Layout Complementation and Tiling

`complement(layout, size)` produces the "rest" of a layout: the set of offsets in a buffer of `size` elements that are *not* covered by `layout`. This is used to compute the stride of tiles:

```cpp
// A layout covering every other element (stride-2 access)
auto every_other = make_layout(make_shape(Int<8>{}), make_stride(Int<2>{}));

// Complement within 16 elements: the 8 elements NOT covered by every_other
// Result: layout with shape<8>, stride<...> covering offsets {1,3,5,7,9,11,13,15}
auto rest = complement(every_other, Int<16>{});
```

`tiled_divide(layout, tile_shape)` partitions a layout into tiles of `tile_shape`. The result is a layout with two coordinate levels: the tile index and the element-within-tile index:

```cpp
// 128×64 row-major matrix, divided into 32×32 tiles
auto matrix = make_layout(make_shape(Int<128>{}, Int<64>{}),
                            make_stride(Int<64>{}, Int<1>{}));
auto tile_shape = make_shape(Int<32>{}, Int<32>{});

// Result shape: ((4, 2), (32, 32))
//   Tile index: 4 tiles along M, 2 tiles along N
//   Tile interior: 32×32 elements with the original strides
auto tiled = tiled_divide(matrix, tile_shape);

// Access tile (1, 0), element (5, 3):
int offset = tiled(make_coord(make_coord(1, 0), make_coord(5, 3)));
// = matrix(1*32 + 5, 0*32 + 3) = (37)*64 + 3 = 2371
```

Note the distinction from `zipped_divide` (covered in Ch164, which interleaves tile and element coordinates): `tiled_divide` keeps them in separate coordinate levels, which is the form used for warp-level partitioning.

### 209.4.4 Tensors: Typed Tile Views

`Tensor<Engine, Layout>` wraps a data source (a pointer or a register fragment) with a layout into a typed tensor view:

```cpp
#include <cute/tensor.hpp>

float* raw_ptr = /* device pointer to 128×64 matrix */;

// Create a tensor view: logical shape 128×64, row-major
auto A = make_tensor(make_gmem_ptr(raw_ptr),
                     make_layout(make_shape(Int<128>{}, Int<64>{}),
                                 make_stride(Int<64>{}, Int<1>{})));

// Slice a 32×32 subtensor starting at logical coord (32, 0)
auto A_tile = local_tile(A, make_shape(Int<32>{}, Int<32>{}),
                          make_coord(1, 0));
// A_tile is a Tensor<..., Layout<(32,32),(64,1)>> backed by raw_ptr + 32*64

// Register-file fragment (lives in registers, not memory)
auto frag = make_fragment_like(A_tile);  // same shape, register-backed
```

`local_tile` extracts a tile at a given tile coordinate. `make_fragment_like` allocates a register-file fragment with the same shape — used for copy destinations when loading tiles from shared memory into registers.

### 209.4.5 TiledMMA and ThrMMA: Warp-Collective Partitioning

`TiledMMA` describes how a threadblock tiles an MMA instruction across threads. `ThrMMA` is the per-thread view obtained by partitioning the `TiledMMA`:

```cpp
// Define the MMA atom: Ampere 16×8×16 FP16 tensor core
using MmaAtom = MMA_Atom<SM80_16x8x16_F32F16F16F32_TN>;

// Tile the atom: 2 atoms along M, 2 along N, 1 along K
// Each warp computes a 32×16 tile (2×16, 2×8)
using TiledMmaType = TiledMMA<
    MmaAtom,
    Layout<Shape<_2, _2, _1>>,    // atom layout within warp
    Layout<Shape<_1, _2, _1>>>;   // value layout

TiledMmaType tiled_mma;

// Partition the global A/B/C tensors for this thread
// A: [128×32] → per-thread fragment
// Thread index comes from threadIdx.x
auto thr_mma = tiled_mma.get_thread_slice(threadIdx.x);

// Partition: extract the per-thread sub-tensor of A, B, C
auto tAgA = thr_mma.partition_A(sA);  // sA: shared memory tensor of A
auto tBgB = thr_mma.partition_B(sB);  // sB: shared memory tensor of B
auto tCrC = thr_mma.partition_C(rC);  // rC: register accumulator tensor

// Execute MMA: accumulate tAgA * tBgB into tCrC
gemm(tiled_mma, tCrC, tAgA, tBgB, tCrC);
```

`partition_A(tensor)` applies the thread's coordinate slice from the `TiledMMA` to `tensor`, returning the fragment of `tensor` that this thread is responsible for loading. The correctness invariant — that `tAgA`, `tBgB`, and `tCrC` have compatible shapes for `gemm()` — is a compile-time type check. Mismatched tile shapes produce a static assertion, not a silent wrong-answer at runtime.

### 209.4.6 TiledCopy: Efficient Tile Loading

Loading tiles from global memory to shared memory uses `TiledCopy`, which tiles a copy instruction (e.g., `cp.async`) across threads:

```cpp
// Define a vectorised copy atom: 128-bit (8×fp16) cp.async
using CopyAtom = Copy_Atom<SM80_CP_ASYNC_CACHEGLOBAL<cute::uint128_t>, half_t>;

// Tile across 32 threads (one warp), each thread copies 8 fp16 elements
auto tiled_copy = make_tiled_copy(
    CopyAtom{},
    Layout<Shape<_32, _1>>{},   // 32 threads along M
    Layout<Shape<_1, _8>>{});   // each thread copies 8 elements along N

auto thr_copy = tiled_copy.get_thread_slice(threadIdx.x);

// Partition global A tensor and shared A tensor for this thread
auto tAgA = thr_copy.partition_S(gA);  // source (global memory)
auto tAsA = thr_copy.partition_D(sA);  // destination (shared memory)

// Execute copy: issues cp.async instructions
copy(tiled_copy, tAgA, tAsA);
```

The `CopyAtom` specifies the PTX instruction (`cp.async` for GMEM→SMEM, `ldmatrix` for SMEM→registers). `TiledCopy` tiles that instruction across threads using the same layout algebra as `TiledMMA`. The connection to hardware is direct: `SM80_CP_ASYNC_CACHEGLOBAL` maps to the PTX `cp.async.ca.global` instruction; `ldmatrix` atoms map to the PTX `ldmatrix.sync.aligned` instruction used for loading MMA register tiles.

### 209.4.7 CuTe DSL: Python-Level Layout Algebra (CUTLASS 4.x)

CUTLASS 4.x ([github.com/NVIDIA/cutlass, tag v4.x](https://github.com/NVIDIA/cutlass)) extends CuTe into a Python-embedded DSL:

```python
import cute
from cute import Layout, Shape, Stride, Tensor

# Define layout at Python level — maps to C++ Layout<Shape,Stride> at codegen
tile_layout = cute.make_layout(
    cute.make_shape(128, 64),
    cute.make_stride(64, 1))

# Compose layouts
composed = cute.composition(global_layout, tile_layout)

# Define a tiled GEMM kernel
@cute.kernel
def gemm_kernel(A: cute.Tensor, B: cute.Tensor, C: cute.Tensor,
                tiled_mma: cute.TiledMMA):
    # Partition tensors for this thread — same API as C++
    thr_mma = tiled_mma.get_thread_slice(cute.thread_idx().x)
    tArA = thr_mma.partition_A(A)
    tBrB = thr_mma.partition_B(B)
    tCrC = thr_mma.partition_C(C)
    cute.gemm(tiled_mma, tCrC, tArA, tBrB, tCrC)

# JIT-compile to PTX
compiled = cute.compile(gemm_kernel, arch="sm_90a")
```

The Python DSL generates the same C++ template instantiations as hand-written CUTLASS 3.x code, but the layout algebra is accessible at Python runtime — enabling programmatic layout generation, which §209.7 exploits.

### 209.4.8 Connection to PTX Hardware Instructions

The layout algebra has a direct, non-abstract connection to PTX instructions. Three PTX instructions dominate CuTe's generated code:

**`ldmatrix.sync.aligned`**: loads a matrix tile from shared memory into the distributed register layout expected by `mma.sync`. The register layout for a 16×16 FP16 tile distributed across 32 threads is precisely the layout produced by `ThrMMA::partition_A(sA)` — CuTe's `partition_A` generates the address computations that map each thread's `threadIdx.x` to the exact shared memory address to load.

```ptx
// PTX: each of 32 threads contributes 1 row address, result distributes 4 FP16
// registers per thread covering the 16x8 sub-tile owned by this thread
ldmatrix.sync.aligned.x4.m8n8.shared.b16 {r0, r1, r2, r3}, [smem_addr];
```

**`mma.sync.aligned`**: Ampere tensor core instruction. The `MMA_Atom<SM80_16x8x16_F32F16F16F32_TN>` in CuTe corresponds exactly to the PTX `mma.sync.aligned.m16n8k16.row.col.f32.f16.f16.f32` instruction, with register operand layout matching the `ldmatrix` output:

```ptx
mma.sync.aligned.m16n8k16.row.col.f32.f16.f16.f32
    {d0, d1, d2, d3},   // FP32 accumulator (4 regs)
    {a0, a1, a2, a3},   // FP16 A fragment (4 regs, 8 elements)
    {b0, b1},           // FP16 B fragment (2 regs, 4 elements)
    {d0, d1, d2, d3};   // FP32 source accumulator
```

**`wgmma.mma_async`**: Hopper warpgroup-level instruction. The `MMA_Atom<SM90_64x128x16_F32F16F16F32_SS>` in CuTe corresponds to the PTX `wgmma.mma_async.sync.aligned.m64n128k16.f32.f16.f16` instruction, which operates on shared-memory-resident A and B tiles (no register operands for A/B — the warpgroup hardware reads SMEM directly):

```ptx
wgmma.mma_async.sync.aligned.m64n128k16.f32.f16.f16
    d_regs,         // FP32 accumulator registers (distributed across 128 threads)
    [a_smem_desc],  // TMA descriptor for A tile in SMEM
    [b_smem_desc],  // TMA descriptor for B tile in SMEM
    1, 1, 0;        // scale-D, scale-A, trans-B flags
```

The `CopyAtom<SM80_CP_ASYNC_CACHEGLOBAL<uint128_t>, half_t>` in a `TiledCopy` corresponds directly to the `cp.async.ca.global.L2::128B` PTX instruction for 128-bit (8×FP16) cached async global loads. CuTe's layout algebra produces exactly the address expressions and register assignments that map to these PTX instructions — without any intermediate lowering step. The C++ template instantiation *is* the PTX code, mediated by NVCC's backend.

---

## 209.5 Linear Layouts and the F₂ Foundation

Two independent traditions of GPU layout algebra — CuTe's `Layout<Shape, Stride>` and Triton's linear layout system — have been unified in [arXiv 2505.23819](https://arxiv.org/abs/2505.23819) under a common mathematical foundation: *linear maps over F₂*, the field with two elements (the integers mod 2).

### 209.5.1 F₂-Linear Maps as a Unification

A layout maps an integer index (a thread index, a logical coordinate) to a physical address. When the index and address are viewed in binary, many GPU layouts are *linear over F₂*: the physical address bits are linear combinations (XOR combinations) of the input index bits. Row-major, column-major, swizzled layouts, and Triton's distributed tensor layouts are all F₂-linear maps when the dimensions are powers of two.

Formally: a *linear layout* is a function `f: F₂ⁿ → F₂ᵐ` represented by a matrix `M ∈ F₂^{m×n}`. The image `M·x` (matrix-vector product over F₂) gives the physical address for logical index `x`. This is efficient to compute in hardware (XOR of address bits) and efficient to compose (matrix multiplication over F₂).

The unification captures:
- CuTe `Layout<Shape<_M,_N>, Stride<_S0,_S1>>` with power-of-two dimensions → F₂ matrix
- Triton's `LinearLayout` (used internally in the TritonGPU dialect) → F₂ matrix  
- Swizzled shared memory access patterns (128B swizzle) → F₂ XOR permutation matrix

### 209.5.2 F₂ Encoding of Concrete Layouts

As a concrete example, consider 8 threads indexed `t ∈ {0..7}` mapping to offsets in a 64-element FP16 buffer with stride 8 (row-major, 8 elements per row):

```
Thread index (binary): t[2] t[1] t[0]
Offset  (binary):      m[5] m[4] m[3] m[2] m[1] m[0]

Row-major stride-8 layout:
m = 8 * t  →  m[5]=t[2], m[4]=t[1], m[3]=t[0], m[2..0]=0

F₂ matrix M (columns = input bits t[2],t[1],t[0]):
       t[2] t[1] t[0]
m[5] [  1    0    0  ]
m[4] [  0    1    0  ]
m[3] [  0    0    1  ]
m[2] [  0    0    0  ]
m[1] [  0    0    0  ]
m[0] [  0    0    0  ]
```

A toy XOR swizzle that avoids 2-way bank conflicts by XORing bit 3 of the offset with bit 0 of the thread index (`m[3] ^= t[0]`) modifies the F₂ matrix by adding 1 to position (row m[3], col t[0]):

```
Swizzled offset = base_offset XOR (t[0] << 3)
         → adds t[0] to m[3]

Swizzled F₂ matrix:
       t[2] t[1] t[0]
m[5] [  1    0    0  ]
m[4] [  0    1    0  ]
m[3] [  0    0    1  ]  ← t[0] contribution preserved (was already here)
m[2] [  0    0    0  ]
m[1] [  0    0    0  ]
m[0] [  0    0    0  ]

For the 128B swizzle used in shared memory (XOR bit 4 of offset with bit 2 of thread):
m[4] ^= t[2]:
       t[2] t[1] t[0]
m[5] [  1    0    0  ]
m[4] [  1    1    0  ]  ← t[2] added to row m[4]
m[3] [  0    0    1  ]
m[2] [  0    0    0  ]
...
```

The swizzle is literally an off-diagonal perturbation of the F₂ matrix. Its correctness property — that no two threads in a warp access the same shared memory bank — is equivalent to the matrix `M` having full column rank (injective linear map over F₂), checkable by Gaussian elimination in O(n³). NVIDIA's swizzle patterns are designed precisely so that `M` has full rank under standard bank-width assumptions.

### 209.5.3 Interoperability Implications

[arXiv 2511.10374](https://arxiv.org/abs/2511.10374) applies F₂ linear layout theory to *layout abstraction via integer sets*, enabling formal analysis of layout compatibility across systems. Concretely:

- A CuTe layout for a CUTLASS 3.x kernel can be represented as an F₂ matrix
- The same layout used in a Triton kernel (covered in [Chapter 163 — Triton](../part-23-mlir-production/ch163-triton-a-compiler-for-gpu-kernels.md) and [Chapter 208](ch208-gpu-kernel-dsls.md)) can be represented as a different F₂ matrix  
- Checking *compatibility* — whether the two kernels can pass a tile between each other without a reformatting copy — is a question of F₂ matrix equality, which is decidable in O(n³) via Gaussian elimination over F₂

This cross-system analysis matters for heterogeneous pipeline construction: when combining a CUTLASS attention kernel with a Triton MLP kernel in the same GPU pipeline, layout compatibility at the tile boundary determines whether an expensive reformatting copy is needed. A layout mismatch that costs a 10–15% throughput penalty (the cost of a global-memory round-trip for reformatting) is detectable before compilation using F₂ matrix analysis.

### 209.5.4 Categorical Foundations

[arXiv 2601.05972](https://arxiv.org/abs/2601.05972) provides a categorical account of CuTe layout algebra: layouts are morphisms in a category where objects are pairs (index space, data space). Composition of layouts is functor composition. The tiling operation is a natural transformation. The categorical view proves that layout composition is associative and that `complement` and `tiled_divide` are adjoint operations — formal guarantees that the algebra is consistent.

For compiler engineering, the practical payoff is: the same layout algebra can be implemented in multiple languages (C++ templates, Python DSL, MLIR attribute types) with guaranteed semantic equivalence, because the semantics are defined categorically rather than operationally.

---

## 209.6 TileIR: Tiled Computation as a First-Class MLIR Dialect

[Chapter 164](../part-23-mlir-production/ch164-cuda-tile-ir.md) documented NVIDIA's Tile IR initiative as of 2024–2026, noting it was in active development with `nvgpu.warpgroup.mma` and TMA ops as the first concrete steps. TileIR is the proposed MLIR dialect surface that raises tile-level computation to first-class status, giving the compiler (and generative systems) explicit tile structure to analyse and transform.

### 209.6.1 Tile IR Op Surface

The proposed TileIR dialect introduces:

**`tile.func`** — a function with explicit tile metadata: the tile shape, element type, and memory hierarchy annotations are part of the function signature, not inferred from the body.

```mlir
// A tiled matmul: computes C[m, n] += A[m, k] * B[k, n]
// tile shape: [128, 256] output, [128, 32] A tile, [32, 256] B tile
tile.func @gemm_tile(
    %A: !tile.buffer<128x32xf16, global>,
    %B: !tile.buffer<32x256xf16, global>,
    %C: !tile.buffer<128x256xf32, global>)
    -> !tile.buffer<128x256xf32, global>
    attributes { tile.shape = [128, 256, 32] } {

  %C_tile = tile.buffer.alloc [128, 256] : !tile.buffer<128x256xf32, shared>

  tile.parallel %k in [0, K, 32] {
    %A_tile = tile.load %A[0, %k] : !tile.buffer<128x32xf16, global>
                                  -> !tile.buffer<128x32xf16, shared>
    %B_tile = tile.load %B[%k, 0] : !tile.buffer<32x256xf16, global>
                                  -> !tile.buffer<32x256xf16, shared>
    tile.sequential {
      tile.mma %A_tile, %B_tile, %C_tile :
          !tile.buffer<128x32xf16, shared>,
          !tile.buffer<32x256xf16, shared>,
          !tile.buffer<128x256xf32, shared>
    }
  }
  tile.return %C_tile : !tile.buffer<128x256xf32, shared>
}
```

**`tile.buffer`** — a buffer type with explicit memory space annotations (`global`, `shared`, `register`). The shape is part of the type, making tile shape mismatches detectable by the MLIR verifier before codegen.

**`tile.parallel`** — a loop annotation indicating parallel execution (loop iterations run as independent tiles across the threadblock grid or within a warpgroup). Semantically equivalent to a `scf.parallel` but with tile-level granularity constraints that the backend lowering respects.

**`tile.sequential`** — a region where operations execute sequentially within a tile, used to express ordered accumulations (the MMA reduction loop along K).

### 209.6.2 Lowering Path: TileIR → GPU Dialect → LLVM

The TileIR lowering pipeline:

```
tile.func / tile.parallel / tile.mma
    │  tile-to-gpu lowering
    ▼
gpu.func / gpu.launch_func / nvgpu.warpgroup.mma / nvgpu.tma.async.load
    │  gpu-to-nvvm conversion
    ▼
llvm.func (NVVM IR with @llvm.nvvm.* intrinsics)
    │  LLVM NVPTX backend (Chapter 102)
    ▼
PTX assembly (.ptx)
    │  ptxas
    ▼
SASS (hardware instruction set)
```

The `tile.parallel` annotation guides the tile-to-gpu lowering in how to map logical tile indices to `blockIdx`/`threadIdx`. A `tile.parallel` over the output tile's M and N dimensions becomes a `gpu.launch` with grid dimensions derived from `ceil(M / tile_M)` and `ceil(N / tile_N)`. The `tile.mma` op becomes `nvgpu.warpgroup.mma` after selecting the appropriate MMA atom based on the element types and the target architecture attribute.

### 209.6.3 TileIR Verification and Error Reporting

Unlike CUTLASS 2.x's cryptic template instantiation backtraces, TileIR's MLIR verifier checks tile shape invariants as part of standard IR verification:

```bash
# Run tile IR verifier
mlir-opt --verify-each --mlir-print-debuginfo flash_attention.mlir

# A tile shape mismatch produces a clean error:
# error: 'tile.mma' operand #0 (A tile) has shape [128, 32] but
#         MMA atom expects [128, 16] along the K dimension
#         at flash_attention.mlir:23:5
```

Because the shape is part of the `!tile.buffer` type, the verifier can check all type-level invariants (`tile.mma` input and output shapes, `tile.load` source vs destination shape compatibility) without running any code. This is the same correctness guarantee as a statically typed language: shape errors are caught at compile time, not at CUDA kernel launch.

### 209.6.4 Relationship to CuTe

TileIR and CuTe are complementary, not competing:

- **CuTe** is a C++ library implementing layout algebra at the expression level. It describes *how* tiles are addressed but does not provide a program IR — it produces inline C++ that the CUDA compiler compiles.
- **TileIR** is an MLIR dialect providing a program IR with explicit tile structure. It can represent the same tile computations that CuTe expresses, but as IR ops subject to compiler analysis, transformation, and verification.

In practice, TileIR computations use CuTe layouts as attributes: the `tile.mma` op carries a CuTe `Layout` attribute describing how threads partition the tile, and the tile-to-gpu lowering translates that attribute into `TiledMMA` instantiation code. The result is that CuTe's layout algebra becomes the *type annotation system* for TileIR operations.

### 209.6.5 Attention via TileIR

A two-pass FlashAttention outer loop in TileIR:

```mlir
tile.func @flash_attention_fwd(
    %Q: !tile.buffer<Sx64xf16, global>,     // [seq_len, head_dim]
    %K: !tile.buffer<Sx64xf16, global>,
    %V: !tile.buffer<Sx64xf16, global>,
    %O: !tile.buffer<Sx64xf16, global>)
    attributes { tile.shape = [128, 64, 64] } {

  // Outer loop: tiles of Q (parallel across SM grid)
  tile.parallel %q_tile in [0, S, 128] {
    %Q_tile = tile.load %Q[%q_tile, 0] : ... -> !tile.buffer<128x64xf16, shared>
    %m_vec = tile.buffer.alloc [128] : !tile.buffer<128xf32, register>  // row max
    %l_vec = tile.buffer.alloc [128] : !tile.buffer<128xf32, register>  // row sum
    %O_acc = tile.buffer.alloc [128, 64] : !tile.buffer<128x64xf32, register>

    // Inner loop: tiles of K and V (sequential — online softmax update)
    tile.sequential {
      tile.for %kv_tile in [0, S, 128] {
        %K_tile = tile.load %K[%kv_tile, 0] : ...
        %S_tile = tile.mma %Q_tile, %K_tile, zeros :
            !tile.buffer<128x128xf32, register>
        // softmax numerics update %m_vec, %l_vec, %O_acc via tile.elementwise
        %V_tile = tile.load %V[%kv_tile, 0] : ...
        tile.update_online_softmax %S_tile, %V_tile, %m_vec, %l_vec, %O_acc
      }
    }
    tile.store %O_acc, %O[%q_tile, 0] : ...
  }
}
```

The TileIR representation makes explicit which loops are parallel (outer Q tiles, safe to execute across SMs) and which are sequential (inner KV tiles, which must execute in order for the online softmax to be numerically correct). The lowering backend reads these annotations to emit the correct CUDA launch configuration and synchronization primitives.

---

## 209.7 Layout Algebra as a Type System for Self-Modification

The chapters in Part XXXI focus on AI systems that can analyse and modify themselves. For GPU kernel generation, layout algebra provides the *type-level correctness substrate* for such modification: generating a kernel means generating a set of `Layout<Shape, Stride>` objects that satisfy the composition invariants. The type checker — whether the CUTLASS template instantiation or the TileIR verifier — reports which candidate kernels are ill-typed before any hardware execution.

### 209.7.1 Layout Types as Compile-Time Constraints

In CUTLASS 3.x, the type `TiledMMA<MMA_Atom, LayoutTV, TileShape>` is correct only if:

1. The MMA atom's output shape divides `TileShape` evenly along all dimensions.
2. The `LayoutTV` (value layout within the atom) assigns each thread a non-overlapping sub-tensor of the tile.
3. The `TiledCopy` layouts for A and B are compatible with the shared memory swizzle pattern and the `ldmatrix` instruction's register format requirements.

Each of these is a constraint on the `Layout` types involved. A kernel generator that produces `Layout` objects satisfying these constraints produces a type-correct, compilable kernel. A kernel generator that violates them gets a static assertion at C++ template instantiation — analogous to a type error in a typed programming language.

### 209.7.2 Searching Over Layout Spaces

Given a problem size `(M, N, K)` and a target GPU architecture, the valid tile configurations form a *search space* over `Layout` parameters:

- `BlockM` ∈ {64, 128} (threadblock tile along M)
- `BlockN` ∈ {64, 128, 256} (threadblock tile along N)
- `BlockK` ∈ {16, 32, 64} (reduction tile depth)
- Pipeline stages ∈ {2, 3, 4}
- Warp layout ∈ {(2,2), (4,1), (1,4)} (2D arrangement of warps within the block)

Not all combinations are valid: `BlockK × sizeof(element)` must not exceed shared memory bank width requirements; `BlockM × BlockN / warp_count` must be a multiple of the MMA instruction's output tile. Layout algebra expresses these constraints as F₂ matrix rank conditions and divisibility properties — checkable at Python level before any compilation.

The following Python function generates the set of valid CuTe layout configurations for a given problem size and GPU architecture, using the CuTe Python DSL from CUTLASS 4.x:

```python
from itertools import product
import cute
from cute import Layout, Shape, Stride

def enumerate_valid_gemm_layouts(M, N, K, arch="sm_90a", dtype="f16"):
    """
    Enumerate type-correct CuTe layout configurations for a GEMM of shape
    (M, N, K) on the given GPU architecture.
    Returns a list of (tile_config, tiled_mma) tuples, sorted by estimated
    shared memory footprint.
    """
    # Architecture-specific MMA atoms
    mma_atoms = {
        "sm_80":  [("SM80_16x8x16_F32F16F16F32_TN", 16, 8, 16)],
        "sm_90a": [("SM90_64x64x16_F32F16F16F32_SS", 64, 64, 16),
                   ("SM90_64x128x16_F32F16F16F32_SS", 64, 128, 16),
                   ("SM90_64x256x16_F32F16F16F32_SS", 64, 256, 16)],
    }
    atoms = mma_atoms.get(arch, mma_atoms["sm_80"])

    candidate_block_k = [16, 32, 64]
    candidate_stages  = [2, 3, 4]
    smem_limit_bytes  = 228 * 1024  # H100 max shared memory
    dtype_bytes       = 2 if dtype in ("f16", "bf16") else 4

    valid_configs = []

    for (atom_name, atom_m, atom_n, atom_k), block_k, stages in product(
            atoms, candidate_block_k, candidate_stages):

        # block_m and block_n are determined by the atom's output tile
        block_m = atom_m
        block_n = atom_n

        # Constraint 1: problem dimensions must be multiples of tile dimensions
        if M % block_m != 0 or N % block_n != 0 or K % block_k != 0:
            continue

        # Constraint 2: block_k must be a multiple of the atom's K dimension
        if block_k % atom_k != 0:
            continue

        # Constraint 3: shared memory footprint must fit within smem_limit
        smem_A = block_m * block_k * dtype_bytes * stages
        smem_B = block_n * block_k * dtype_bytes * stages
        if smem_A + smem_B > smem_limit_bytes:
            continue

        # Build the CuTe layout objects (Python DSL)
        A_layout = cute.make_layout(
            cute.make_shape(block_m, block_k),
            cute.make_stride(block_k, 1))          # row-major A tile
        B_layout = cute.make_layout(
            cute.make_shape(block_k, block_n),
            cute.make_stride(1, block_k))           # col-major B tile (transposed)

        # Attempt TiledMMA construction — raises if layouts are incompatible
        try:
            tiled_mma = cute.make_tiled_mma(
                atom_name,
                cute.make_layout(cute.make_shape(1, 1, 1)))  # single-atom tiling
            config = {
                "atom": atom_name,
                "block": (block_m, block_n, block_k),
                "stages": stages,
                "smem_bytes": smem_A + smem_B,
                "A_layout": A_layout,
                "B_layout": B_layout,
            }
            valid_configs.append((config, tiled_mma))
        except cute.LayoutError:
            pass  # incompatible layout — skip

    # Sort by smem footprint ascending (prefer lower occupancy cost)
    valid_configs.sort(key=lambda t: t[0]["smem_bytes"])
    return valid_configs

# Example: enumerate valid configs for a 4096×4096×4096 GEMM on H100
configs = enumerate_valid_gemm_layouts(4096, 4096, 4096, arch="sm_90a")
for cfg, _ in configs[:5]:
    print(f"block={cfg['block']}, stages={cfg['stages']}, "
          f"smem={cfg['smem_bytes']//1024}KB")
```

This function is the kernel generator's innermost loop: instead of hard-coding a single `TileShape`, it produces the *complete set of type-correct candidates*, which an autotuner, evolutionary search, or a reinforcement-learning-guided optimizer can evaluate on hardware.

### 209.7.3 Register Pressure as a Layout Constraint

One dimension not captured by `smem_bytes` alone is register pressure. The accumulator tile `tCrC` lives in registers across all threads. For a 64×128 output tile shared among 128 threads (one warpgroup), each thread holds `(64×128) / 128 = 64` accumulator elements. At FP32 (4 bytes each), that is 256 bytes = 64 registers per thread. Adding the A and B register fragments for double-buffering requires another 32–48 registers. Total register use reaches ~100–110 registers per thread on Hopper.

The H100 SM has 65,536 32-bit registers, shared among all resident warps. With 128-thread warpgroups (4 warps) and 110 registers/thread:

```
registers per warpgroup = 128 × 110 = 14,080
max warpgroups per SM   = floor(65536 / 14080) ≈ 4
occupancy               = 4 warpgroups × 128 threads = 512 threads per SM
```

If the pipeline stages are increased (more double-buffering), register pressure increases and occupancy falls. A valid tile layout configuration must balance SMEM, register, and occupancy constraints simultaneously. The `enumerate_valid_gemm_layouts` function above checks SMEM; a production version also estimates register usage from the tile shapes and stages:

```python
def estimate_registers_per_thread(block_m, block_n, block_k, stages, atom_m, atom_n):
    # Accumulator: each thread holds 1/(warpgroup_size) of the output tile
    warpgroup_size = 128  # Hopper warpgroup
    acc_elems = (block_m * block_n) // warpgroup_size
    acc_regs  = acc_elems  # FP32, 1 reg each

    # A and B fragments (one stage + prefetch = 2 stages worth)
    a_elems = (block_m * block_k) // warpgroup_size
    b_elems = (block_k * block_n) // warpgroup_size
    frag_regs = (a_elems + b_elems) * stages // 2  # rough: half loaded at a time

    # Overhead (loop indices, predicates, addresses) ≈ 20 registers
    return acc_regs + frag_regs + 20

def max_warpgroups_per_sm(regs_per_thread, sm_register_file=65536):
    regs_per_warpgroup = regs_per_thread * 128
    return sm_register_file // regs_per_warpgroup
```

### 209.7.4 Connection to Evolutionary Architecture Search

[Chapter 215 — Evolutionary Architecture Search](ch215-evolutionary-architecture-search.md) treats tile configuration as an architectural choice subject to mutation and selection. Layout algebra makes this safe:

- **Mutation** generates a new `(block_m, block_n, block_k, stages)` tuple.
- **Type checking** via `enumerate_valid_gemm_layouts` filters out invalid mutations before compilation.
- **Fitness evaluation** runs the surviving candidates on hardware and records throughput.

The layout type system acts as the *validity predicate* in the evolutionary loop — without it, invalid mutations would cause silent miscompilation or cryptic CUDA errors rather than clean rejection. The F₂ linear layout theory (§209.5) extends this to cross-kernel compatibility: a mutation that changes the output layout of an attention kernel must be checked against the expected input layout of the subsequent MLP kernel, which is an F₂ matrix comparison.

For the self-modification vision of [Chapter 211 — Neural Programs as Compiled Artifacts](ch211-neural-programs-as-compiled-artifacts.md), the layout algebra occupies the role of a *type discipline for kernel-level introspection*: a system that can read a running kernel's tile layout (from SASS analysis or profiler annotations) and propose a layout modification has a formal correctness criterion — the proposed layout must be F₂-compatible with the hardware's warp register layout — that is checkable before the new kernel is compiled and deployed.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **CUTLASS 4.x stable release and CuTe DSL stabilisation**: The CUTLASS 4.x Python DSL (`@cute.kernel`, `cute.compile`) is in active preview as of April 2026; the NVIDIA CUTLASS team has signalled a stable API freeze targeting the CUDA 13.0 release window. Track progress at [github.com/NVIDIA/cutlass/milestone/9](https://github.com/NVIDIA/cutlass) and the CUTLASS 4.x design RFC on the NVIDIA developer forums.
- **TileIR upstreaming into LLVM monorepo**: NVIDIA's TileIR prototype (residing in the `nvgpu` dialect's `warpgroup` ops as of LLVM 22) has an open MLIR RFC proposing a standalone `tile` dialect at [discourse.llvm.org — MLIR tile dialect RFC](https://discourse.llvm.org/). The 6-month target is first-draft dialect ops (`tile.func`, `tile.mma`, `tile.load`) landing upstream with the verifier infrastructure from §209.6.3.
- **F₂ linear layout analysis tools in Triton and CUTLASS**: The unification paper [arXiv 2505.23819] is being translated into a concrete `LinearLayout::isCompatible()` API in the Triton compiler (`lib/Dialect/TritonGPU/IR/LinearLayout.cpp`); CUTLASS intends to adopt F₂ matrix encoding as the canonical layout representation in CUTLASS 5.x, enabling cross-kernel layout compatibility checks at the Python DSL level.
- **Blackwell (SM100) support in CUTLASS and CuTe**: NVIDIA GB200 (Blackwell architecture, SM100) introduces new MMA atoms (`tcgen05.mma`) and a 5th-generation tensor core with 4:2 fine-grained sparsity; CUTLASS 3.7+ and CuTe 4.x are adding `MMA_Atom<SM100_*>` specialisations. Track at [github.com/NVIDIA/cutlass/tree/main/include/cute/atom/mma_traits_sm100.hpp](https://github.com/NVIDIA/cutlass).

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **TileIR as a fully-featured MLIR compilation target**: Once the `tile` dialect is upstream, the 2028 target is a complete lowering infrastructure — `tile.parallel` to multi-SM dispatch, `tile.sequential` to barrier-synchronised warpgroup reduction loops — with support for both NVIDIA (via `nvgpu`) and AMD (via `amdgpu`) backends; AMD's ROCm team is tracking MLIR tile dialect progress for potential RDNA 4 / CDNA 4 adoption.
- **Categorical layout algebra formalised in Lean 4**: The categorical account of CuTe layouts ([arXiv 2601.05972]) is a candidate for mechanised proof in Lean 4, providing a formally verified correctness certificate for `composition`, `complement`, and `tiled_divide` operations; this would enable layout-correctness checking to be part of a certified compiler pass rather than a runtime assertion. Related work: the Alive2 extension to GPU memory semantics ([arXiv 2404.10728]) provides the verification infrastructure.
- **Autotuning over F₂ layout space via ML surrogates**: Using the `enumerate_valid_gemm_layouts` approach from §209.7.2 as the candidate generator, a Bayesian optimisation or learned surrogate model (similar to the Google TPU autotuning work, [arxiv.org/abs/2301.07766]) trained on (layout config, hardware throughput) pairs is expected to achieve within 5% of hand-tuned CUTLASS kernels for arbitrary `(M, N, K)` shapes on Hopper and Blackwell by 2028; NVIDIA's internal CUTLASS autotuner (`tools/profiler`) is the most likely deployment vehicle.
- **Thrust integration with CUDA cooperative groups and `cuda::pipeline`**: Thrust's three-reduction model (§209.2.4) currently emits three separate kernel launches; the CCCL roadmap includes a fused multi-statistic reduction primitive using `cuda::pipeline` double-buffering and `cuda::atomic_ref` for in-kernel accumulation, targeting a single-kernel path for `(mean, variance, max)` in one sweep.

### 5-Year Horizon (Long-Term, by ~2031)

- **CuTe-style layout algebra for non-NVIDIA hardware**: As AMD, Intel (Xe), and Qualcomm AI accelerators all develop tile-level hardware instructions analogous to WGMMA (AMD's WMMA/MFMA, Intel's DPAS), the F₂ linear layout framework is a natural cross-vendor abstraction layer; the 5-year expectation is a vendor-neutral `cute::Layout` standard (potentially under the Khronos or SYCL umbrella) allowing tile-correct kernel code to be retargeted across vendors by swapping `MMA_Atom` specialisations, as discussed in the 2025 LLVM Developer Meeting BOF on cross-vendor GPU lowering.
- **TileIR as the universal ML compiler IR for large-scale models**: As model complexity pushes past trillion-parameter MoE architectures, the kernel fusion and inter-kernel layout compatibility problem (§209.5.3) becomes the dominant compilation bottleneck; TileIR is positioned as the IR where fusion decisions are made — fusing an attention tile, a post-attention layer norm tile, and the first MLP tile into a single GPU kernel without reformatting copies. The 5-year target is an MLIR-based ML compiler pipeline (TileIR → `nvgpu` / `amdgpu` / `xla.gpu`) replacing hand-written XLA fusions for transformers.
- **Formally verified tile kernel generation for safety-critical AI**: The combination of CuTe's categorical layout algebra (§209.5.4, verified in Lean 4) and TileIR's MLIR verifier infrastructure creates a path to formally verified GPU kernel generation: a tile kernel is type-correct by construction (layout types), passes IR verification (TileIR verifier), and its lowering is certified (via a CompCert-style proof of the tile-to-gpu lowering pass). This is a prerequisite for deploying AI inference kernels in automotive (ISO 26262 ASIL-D) and aerospace (DO-178C DAL-A) safety contexts.

---

## Chapter Summary

- **Thrust** is the GPU analogue of the C++ STL: `thrust::device_vector`, `thrust::transform_reduce`, `thrust::inclusive_scan`, and `thrust::sort_by_key` express parallel patterns without explicit thread coordination; execution policies (`thrust::cuda::par.on(stream)`) separate algorithm semantics from backend, enabling the same code to run on CUDA, TBB, or OpenMP. The hard ceiling is fused operations requiring shared memory — where CUTLASS or hand-written kernels are needed.

- **CUTLASS 2.x** expressed the entire GEMM tile hierarchy as template parameters (`ElementA`, `LayoutA`, `ThreadblockShape`, `WarpShape`, `InstructionShape`, pipeline stages), leading to 13+ template parameter lists whose correctness invariants were implicit. CUTLASS 3.x replaces this with CuTe layout algebra, making tile composition relationships explicit as `Layout<Shape, Stride>` types.

- **CuTe's layout algebra** represents tile structures as compile-time bijective functions: `Layout<Shape, Stride>` maps logical coordinates to linear offsets; `composition` chains two layouts; `tiled_divide` partitions into a tile-index plus element coordinate; `complement` computes the uncovered stride set. Mismatched compositions are type errors, caught at template instantiation rather than at runtime.

- **`TiledMMA` / `ThrMMA`** tile an MMA instruction across a threadblock using layout algebra: `thr_mma.partition_A(sA)` extracts the per-thread sub-tensor of a shared memory tile; `cute::gemm(tiled_mma, ...)` executes the tiled matrix multiply. Compatibility among A, B, and C fragment shapes is a compile-time invariant.

- **F₂ linear layout theory** ([arXiv 2505.23819](https://arxiv.org/abs/2505.23819)) unifies CuTe layouts, Triton linear layouts, and SMEM swizzle patterns as F₂ matrices. Cross-system layout compatibility — whether a CuTe kernel and a Triton kernel can share a tile without reformatting — is decidable via F₂ matrix equality. Categorical foundations ([arXiv 2601.05972](https://arxiv.org/abs/2601.05972)) prove the algebra is consistent.

- **TileIR** is the MLIR dialect surface for tiled computation: `tile.func` carries explicit tile shape metadata; `tile.buffer` encodes memory space; `tile.parallel` / `tile.sequential` encode which loops are safe to run concurrently versus which must be serialised (e.g., the online softmax KV loop). The lowering path TileIR → `nvgpu` dialect → NVVM → PTX → SASS is where layout attributes become hardware instructions. TileIR and CuTe are complementary: TileIR provides the IR substrate; CuTe provides the layout algebra used as TileIR type annotations.

- **Layout algebra as a type system for self-modification**: the set of valid `Layout<Shape, Stride>` configurations for a given hardware target and problem size is finitely enumerable, type-checkable before compilation, and expressible as F₂ matrix constraints. Evolutionary kernel search ([Chapter 215](ch215-evolutionary-architecture-search.md)) and neural kernel introspection ([Chapter 211](ch211-neural-programs-as-compiled-artifacts.md)) both benefit from this: layout types act as the validity predicate that filters ill-formed kernel mutations before they reach the compiler.

---

@copyright jreuben11
