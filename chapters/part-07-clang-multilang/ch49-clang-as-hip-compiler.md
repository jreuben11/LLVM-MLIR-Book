# Chapter 49 — Clang as a HIP Compiler

*Part VII — Clang as a Multi-Language Compiler*

AMD's Heterogeneous-compute Interface for Portability (HIP) is the primary programming model for GPU compute on AMD hardware, and since LLVM 8 Clang has been its canonical compiler front end. This chapter examines every layer of that toolchain in depth: the language extension model shared with CUDA, the ROCm software stack, the AMDGPU LLVM backend and its SGPR/VGPR register architecture, fat-binary packaging, separate compilation with relocatable device code, the HSA runtime and AQL dispatch protocol, and the ecosystem of profiling and debugging tools. Readers who have worked through Chapter 48 (Clang as a CUDA compiler) will recognise the structural parallels; this chapter highlights where the two paths diverge and why those divergences matter for portability work.

---

## 49.1 HIP Overview and the ROCm Stack

### 49.1.1 Design Goals and CUDA Compatibility

HIP's central promise is source-level compatibility with CUDA C++: most CUDA source files become valid HIP source by a mechanical transformation. The runtime API names map one-to-one (`cudaMalloc` → `hipMalloc`, `cudaMemcpy` → `hipMemcpy`, `<<<...>>>` kernel launch → `hipLaunchKernelGGL`), thread/block/grid built-ins acquire a `hip` prefix (`threadIdx.x` → `hipThreadIdx_x`), and vector types and intrinsic headers follow the same pattern. The transformation is not merely lexical: HIP's runtime semantics match CUDA's at the API contract level, so code that uses streams, events, pinned memory, and asynchronous copies ports with zero algorithmic changes.

The canonical HIP compilation command targeting AMD GCN9:

```bash
clang++ -x hip --offload-arch=gfx942 -O2 -o matmul matmul.hip
```

For NVIDIA targets (HIP-on-CUDA), the same source compiles with:

```bash
clang++ -x hip --offload-arch=sm_90 -O2 --cuda-path=/usr/local/cuda -o matmul matmul.hip
```

`-x hip` tells the Clang driver to treat the input as HIP source regardless of file extension; `.hip` files are recognised automatically when the extension is present. The `--offload-arch` flag accepts a GCN target identifier (`gfx906`, `gfx908`, `gfx90a`, `gfx942`, `gfx1100`, `gfx1201`) or an NVIDIA SM version.

### 49.1.2 HIPification: CUDA to HIP Migration

The `hipify-perl` and `hipify-clang` tools perform mechanical CUDA→HIP source translation. `hipify-clang` is the higher-fidelity tool: it parses the CUDA AST via Clang's libtooling infrastructure and rewrites it using the `Rewriter` class, preserving formatting and comments. `hipify-perl` is a regex-based approximation useful for quick passes over large codebases where full parsing is too expensive. Both tools output statistics on translated APIs, identifying calls that require manual attention (typically because HIP lacks a direct equivalent or the semantics differ slightly).

The `hip/hip_runtime.h` header provides CUDA→HIP mapping macros for host code compiled with `HIP_PLATFORM=amd`. Under that platform definition, `cuda_runtime.h`-style includes resolve to HIP equivalents, and the `cudaError_t` type is aliased to `hipError_t`. When `HIP_PLATFORM=nvidia`, HIP degrades to a thin wrapper that includes the real CUDA headers unchanged, enabling a single codebase to build and run on both hardware families.

### 49.1.3 The ROCm Software Stack

The full ROCm stack, bottom to top:

| Layer | Component | Role |
|-------|-----------|------|
| Hardware | AMD Instinct MI300X, MI250, Radeon | GCN/CDNA/RDNA microarchitecture |
| Kernel driver | `amdgpu` (in-tree Linux) | PCIe device management, memory mapping, KFD |
| KFD | `kfd` (Kernel Fusion Driver) | User-mode process management, queue allocation |
| HSA runtime | `libhsa-runtime64.so` | Signal/queue/memory HSA abstractions |
| ROCm runtime | `libamdhip64.so` | HIP runtime API implementation |
| Device libs | `rocm-device-libs` (bitcode) | `ocml`, `oclc`, `ockl` math/builtin libraries |
| Compiler | Clang + AMDGPU backend | HIP/OpenCL/OpenMP to ISA |
| Math libraries | rocBLAS, rocSPARSE, rocFFT | Accelerated compute primitives |

The `libhsa-runtime64.so` HSA (Heterogeneous System Architecture) runtime is the substrate on which the HIP runtime is built. It exposes a low-level queue/signal API defined by the HSA Foundation specification. An HSA queue is a ring buffer of AQL (Architected Queuing Language) packets residing in pinned memory; hardware reads these packets directly without kernel involvement. The `hipLaunchKernelGGL` function constructs and submits an `hsa_kernel_dispatch_packet_t` to a hardware queue, setting the grid dimensions, workgroup dimensions, the kernel object address, and the kernarg segment pointer.

### 49.1.4 Compilation Pipeline Overview

The driver invocation `clang++ -x hip --offload-arch=gfx942 foo.hip` produces a compilation DAG that can be inspected with `-ccc-print-bindings`:

```bash
clang++ -x hip --offload-arch=gfx942 -ccc-print-bindings foo.hip 2>&1
# "clang", inputs: ["foo.hip"], output: (nothing)   [host-x86_64-unknown-linux-gnu]
# "clang", inputs: ["foo.hip"], output: (nothing)   [device-amdgcn-amd-amdhsa:gfx942]
# "clang-offload-bundler", inputs: [...], output: "foo.o"
```

Each `clang` node in the binding tree is an `OffloadAction` that carries both a host and one or more device compilations. The `ToolChain::constructActions` method in the Clang driver (see `clang/lib/Driver/Driver.cpp`) builds these DAG nodes; for HIP, it calls `HIPAMDToolChain::constructActions`, which sets up the separate host and device compilation passes and schedules them.

---

## 49.2 HIP Language Support in Clang

### 49.2.1 Shared CUDA/HIP Sema Infrastructure

Clang's semantic analysis for both CUDA and HIP lives in `clang/lib/Sema/SemaCUDA.cpp`. The file is not CUDA-specific despite its name; it handles the `__host__`, `__device__`, `__global__`, and `__host__ __device__` function qualifiers for both languages. The determination of which qualifier applies at any given call site is made through `SemaCUDA::IdentifyTarget()`, which returns a value of the `CUDAFunctionTarget` enum:

```cpp
enum CUDAFunctionTarget {
  CFT_Device,
  CFT_Global,
  CFT_Host,
  CFT_HostDevice,
  CFT_InvalidTarget
};
```

For HIP, the same enum drives identical semantics: `__global__` functions are kernel entry points callable only from host code, `__device__` functions are callable only from device code, and `__host__ __device__` functions may be called from either context. The diagnostic machinery that detects cross-context call violations — emitting `error: reference to __device__ function from __host__ function` — is reused without modification. The `HIPAttr` attribute class (generated from `clang/include/clang/Basic/Attr.td`) marks functions as HIP-annotated, and `HIPKernelCallAttr` records that a call expression is a kernel launch requiring grid/block argument synthesis.

The key divergence point between CUDA and HIP in Sema is `SemaCUDA::isValidDevice()`: for CUDA, a device is an NVPTX target; for HIP, it is an AMDGPU target. The `LangOptions::HIP` bit (distinct from `LangOptions::CUDA`) controls which branch of conditionals in `SemaCUDA.cpp` fires when the two languages require different handling — primarily in template resolution rules and the handling of `__managed__` vs `__hip_managed__` attributes.

### 49.2.2 Predefined Macros and Feature Detection

When Clang processes HIP source, `InitPreprocessor` defines the following macros (see `clang/lib/Frontend/InitPreprocessor.cpp`):

```cpp
// Defined for all HIP compilation (both host and device passes)
#define __HIP__ 1
// Defined only on the device compilation pass
#define __HIP_DEVICE_COMPILE__ 1
// AMD GPU target identification
#define __AMDGCN__ 1
#define __AMD__ 1
// Math precision guarantees
#define __HIP_ARCH_HAS_DOUBLES__ 1
#define __HIP_ARCH_HAS_WARP_SHUFFLE__ 1
#define __HIP_ARCH_HAS_GLOBAL_INT64_ATOMICS__ 1
```

`__HIP_DEVICE_COMPILE__` is the primary guard for device-only code paths:

```cpp
#ifdef __HIP_DEVICE_COMPILE__
__device__ float fast_sqrt(float x) {
    return __builtin_amdgcn_sqrt_f32(x);
}
#else
float fast_sqrt(float x) { return sqrtf(x); }
#endif
```

`clang/lib/Basic/Targets/AMDGPU.cpp` adds target-specific macros corresponding to the `--offload-arch` value. For `gfx942` (AMD Instinct MI300), Clang defines `__gfx942__`, `__GFX9__`, `__HIP_ARCH_HAS_SPARSE_MATRICES__`, and `__HIP_ARCH_HAS_FP8__`. The `AMDGPUTargetInfo::getTargetDefines()` method populates these macros from the GCN ISA version table.

### 49.2.3 Atomic and Synchronisation Builtins

HIP exposes atomic operations through both the standard C++ `<atomic>` header (lowered through Clang's standard `__atomic_*` codegen) and AMD-specific builtins for operations not in the C++ standard. The AMD builtins correspond directly to specific AMDGPU ISA instructions and follow the naming convention `__builtin_amdgcn_*`:

```cpp
// Floating-point atomic add to global memory (GFX908+)
// Lowers to: global_atomic_add_f32
__device__ float fatomic_add(float *addr, float val) {
    return __builtin_amdgcn_global_atomic_fadd_f32(addr, val);
}

// Warp-level ballot (64-wide wavefront on GFX9)
// Lowers to: s_ballot_b64
__device__ unsigned long long warp_ballot(int pred) {
    return __builtin_amdgcn_ballot_w64((bool)pred);
}

// Lane-to-lane permute within a wavefront
// Lowers to: ds_bpermute_b32 (byte-addressed lane index * 4)
__device__ int ds_permute(int val, int lane) {
    return __builtin_amdgcn_ds_bpermute(lane * 4, val);
}

// Quad-level data exchange (butterfly pattern)
// Lowers to: ds_swizzle_b32 with XOR mask
__device__ int quad_swap(int val, int mask) {
    return __builtin_amdgcn_ds_swizzle(val, 0x80000000 | mask);
}
```

`__builtin_amdgcn_s_barrier()` implements `__syncthreads()` — it lowers to `llvm.amdgcn.s.barrier`, which emits an `s_barrier` instruction synchronising all threads in a workgroup. For fine-grained memory fences, `__builtin_amdgcn_fence()` takes a scope constant (workgroup/agent/system) and an atomic ordering (`__ATOMIC_SEQ_CST`, `__ATOMIC_ACQUIRE`, etc.) matching the HSA memory model:

```cpp
// HSA system-scope sequential consistency fence
// Equivalent to: __atomic_thread_fence(__ATOMIC_SEQ_CST)
__device__ void system_fence() {
    __builtin_amdgcn_fence(__ATOMIC_SEQ_CST, "");        // system scope
    // or:
    __builtin_amdgcn_fence(__ATOMIC_ACQUIRE, "workgroup"); // workgroup scope
}
```

The fence scope strings are `""` (system), `"workgroup"`, `"agent"`, and `"wavefront"`. These map to the HSA memory scope hierarchy: wavefront → workgroup → agent (device) → system.

A comprehensive table of commonly used AMDGPU intrinsics:

| Builtin / Intrinsic | LLVM Intrinsic | ISA instruction | Purpose |
|---------------------|----------------|-----------------|---------|
| `__builtin_amdgcn_workitem_id_x()` | `llvm.amdgcn.workitem.id.x` | — (from SGPR) | Thread ID X |
| `__builtin_amdgcn_workgroup_id_x()` | `llvm.amdgcn.workgroup.id.x` | — (from SGPR) | Block ID X |
| `__builtin_amdgcn_s_barrier()` | `llvm.amdgcn.s.barrier` | `s_barrier` | Workgroup sync |
| `__builtin_amdgcn_ds_bpermute()` | `llvm.amdgcn.ds.bpermute` | `ds_bpermute_b32` | Lane permute |
| `__builtin_amdgcn_ds_swizzle()` | `llvm.amdgcn.ds.swizzle` | `ds_swizzle_b32` | Quad/butterfly |
| `__builtin_amdgcn_ballot_w64()` | `llvm.amdgcn.ballot.i64` | `s_ballot_b64` | 64-wide ballot |
| `__builtin_amdgcn_readlane()` | `llvm.amdgcn.readlane` | `v_readlane_b32` | Read specific lane |
| `__builtin_amdgcn_writelane()` | `llvm.amdgcn.writelane` | `v_writelane_b32` | Write specific lane |
| `__builtin_amdgcn_global_atomic_fadd_f32()` | `llvm.amdgcn.global.atomic.fadd` | `global_atomic_add_f32` | FP32 global atomic |
| `__builtin_amdgcn_sqrt_f32()` | `llvm.amdgcn.sqrt.f32` | `v_sqrt_f32` | Native sqrt |

### 49.2.4 Vector Types and Device Libraries

`hip/hip_vector_types.h` defines packed SIMD types: `float2`, `float4`, `int2`, `int4`, `ulong2`, and so on — matching CUDA's vector type header in both naming and structure layout. Unlike CUDA, where vector types map to NVPTX's `.v2` and `.v4` types, on AMDGPU they map to LLVM IR vector types `<2 x float>`, `<4 x float>`, which the AMDGPU backend lowers to consecutive VGPR pairs. The backend implements `float4` arithmetic by emitting four parallel VALU instructions operating on four-lane VGPR groups.

The `rocm-device-libs` package ships a set of LLVM bitcode libraries linked into every device compilation:

| Library | Purpose | Key functions |
|---------|---------|--------------|
| `ocml.bc` | Open Compute Math Library | `sin`, `cos`, `exp`, `log`, `pow`, `sqrt` — all transcendentals |
| `oclc_isa_version_942.bc` | ISA-version selection constants | `__oclc_ISA_version` = 9420 |
| `ockl.bc` | Open Compute Kernel Library | `printf`, global atomics, image load/store |
| `hip.bc` | HIP device runtime | `hipDeviceSynchronize` (no-op on device), `assert` |
| `irif.bc` | IR interface functions | Wrappers that call LLVM intrinsics by name |

Clang's HIP toolchain links these bitcode files via `llvm-link` during device compilation before optimisation and code generation, analogous to CUDA's `libdevice.bc` linking step. The `--rocm-path` argument specifies the ROCm installation root; Clang searches `$ROCM_PATH/amdgcn/bitcode/` for the `.bc` files, selecting the ISA-version-specific `oclc` library to match the `--offload-arch` target.

---

## 49.3 AMDGPU IR and the AMDGPU Backend

### 49.3.1 Target Triple and Data Layout

HIP device code compiles to the triple `amdgcn-amd-amdhsa`. The `amdhsa` environment component selects the HSA ABI, which differs from the `amdpal` environment used by game shader compilation and `amdmesa` used by Mesa. The actual data layout string for GFX942 (verified with Clang 22.1.3):

```
e-m:e-p:64:64-p1:64:64-p2:32:32-p3:32:32-p4:64:64-p5:32:32-p6:32:32
-p7:160:256:256:32-p8:128:128:128:48-p9:192:256:256:32
-i64:64-v16:16-v24:32-v32:32-v48:64-v96:128-v192:256-v256:256
-v512:512-v1024:1024-v2048:2048-n32:64-S32-A5-G1-ni:7:8:9
```

The `A5` component specifies that alloca addresses are in address space 5 (private/scratch). The `G1` component specifies that global variable declarations go to address space 1 (global memory). The `ni:7:8:9` component declares that address spaces 7, 8, and 9 (fat pointers and buffer resources) use non-integral pointer semantics, preventing the optimizer from performing invalid pointer arithmetic on them.

### 49.3.2 AMDGPU Address Spaces

Address space handling is critical to correct HIP codegen. The mapping from HIP/OpenCL source to LLVM IR address spaces, verified against `llvm/include/llvm/Support/AMDGPUAddrSpace.h`:

| Source qualifier | LLVM AS | `AMDGPUAS` enum constant | Hardware resource |
|-----------------|---------|--------------------------|------------------|
| (none) / flat | 0 | `FLAT_ADDRESS` | Flat addressing — unified VA over global + private |
| `__global` / global heap | 1 | `GLOBAL_ADDRESS` | VMEM (video memory), accessed via scalar/vector memory instructions |
| GDS region | 2 | `REGION_ADDRESS` | Global Data Share — rare, cross-CU |
| `__local` / `__shared__` | 3 | `LOCAL_ADDRESS` | LDS (Local Data Share) — on-chip shared memory per CU |
| `__constant` | 4 | `CONSTANT_ADDRESS` | Constant memory — cached read-only VMEM |
| `__private` | 5 | `PRIVATE_ADDRESS` | Private memory — scratch (VGPR overflow stack) |
| const 32-bit | 6 | `CONSTANT_ADDRESS_32BIT` | 32-bit constant buffer segment |
| Fat pointer | 7 | `BUFFER_FAT_POINTER` | 160-bit {resource descriptor, offset} |
| Buffer resource | 8 | `BUFFER_RESOURCE` | 128-bit SGPR buffer resource descriptor |

Flat addressing (AS 0) enables uniform pointer handling at the cost of an extra instruction: flat load/store instructions examine the high bits of the 64-bit virtual address to route to the correct memory subsystem (global, private, or LDS). This is the default for HIP code compiled with fat pointers; the AMDGPU backend may specialise flat accesses into typed accesses when the address space can be inferred statically.

Clang inserts explicit `addrspacecast` instructions when HIP source mixes pointer types. The `SemaOpenCL` and `SemaCUDA` shared addrspace checking logic (extended for HIP in `clang/lib/Sema/SemaType.cpp`) diagnoses mismatched address space casts at compile time.

### 49.3.3 SGPR vs VGPR: Divergence-Driven Register Assignment

The defining microarchitectural feature of AMDGPU is the split between *scalar* and *vector* register files:

- **VGPR (Vector General-Purpose Register)**: 64 copies (one per wavefront lane) in parallel, 32 bits each. An instruction operating on a VGPR operates on all 64 lanes simultaneously. A divergent value — one that differs between lanes — must live in a VGPR.
- **SGPR (Scalar General-Purpose Register)**: One copy shared across all 64 lanes. An instruction operating on an SGPR computes a single result. A *uniform* value — identical across all 64 lanes — can live in an SGPR, saving 63 registers.

The `AMDGPUDivergenceAnalysis` pass (see `llvm/lib/Target/AMDGPU/AMDGPUDivergenceAnalysis.cpp`) computes which IR values are uniform vs. divergent. Values derived from `llvm.amdgcn.workitem.id.x()` (thread ID) are always divergent; values derived only from uniform sources (kernel arguments, constants, `llvm.amdgcn.workgroup.id.x()`) are uniform. The result feeds into register bank selection: divergent values go to VGPR, uniform values go to SGPR.

Incorrect divergence analysis — typically caused by indirect function calls, function pointers, or complex control flow — is a frequent source of ISA-level miscompilation. The `SIFoldOperands` pass attempts to fold SGPR values into instruction operand slots to reduce VGPR pressure. When register pressure exceeds hardware limits (256 VGPRs per CU on GFX9), the AMDGPU backend spills VGPRs to scratch (private address space 5), which is expensive (off-chip memory).

### 49.3.4 Key AMDGPU Intrinsics in LLVM IR

The `llvm.amdgcn.*` intrinsic family provides access to AMD-specific hardware operations that have no portable LLVM equivalent. Actual IR emitted by Clang 22.1.3 for an AMDGCN target:

```llvm
; Thread ID (equivalent to threadIdx.x) — uniform within a workitem lane
%tid.x = call i32 @llvm.amdgcn.workitem.id.x()
%tid.y = call i32 @llvm.amdgcn.workitem.id.y()
%tid.z = call i32 @llvm.amdgcn.workitem.id.z()

; Workgroup (block) ID
%bid.x = call i32 @llvm.amdgcn.workgroup.id.x()

; Workgroup barrier (__syncthreads())
call void @llvm.amdgcn.s.barrier()

; Lane-to-lane permute within a wavefront (via LDS)
; arg0 = byte-addressed lane index (lane * 4), arg1 = src value
%shuffled = call i32 @llvm.amdgcn.ds.bpermute(i32 %lane_x4, i32 %src)

; Read value from a specific lane unconditionally
%val = call i32 @llvm.amdgcn.readlane(i32 %vgpr_val, i32 %lane_id)

; 64-lane ballot (all lanes predicate)
%mask64 = call i64 @llvm.amdgcn.ballot.i64(i1 %predicate)

; GFX908+ FP32 global atomic add (returns old value)
%old = call float @llvm.amdgcn.global.atomic.fadd.f32(
    ptr addrspace(1) %addr, float %addend)

; Special register reads
%exec = call i64 @llvm.amdgcn.read.register.i64(metadata !"exec")
%flat_scratch_lo = call i32 @llvm.amdgcn.read.register.i32(metadata !"flat_scratch_lo")
```

The `EXEC` mask register is a 64-bit bitmask controlling which lanes are active in the current wavefront. It is modified by divergent branches; the `llvm.amdgcn.read.register` intrinsic allows reading it for wavefront-level predicates.

### 49.3.5 HSA Kernel Metadata

Compiled HIP kernels carry machine metadata in the `.rodata` ELF section, formatted as YAML in an ELF note of type `NT_AMD_HSA_CODE_OBJECT_VERSION` / `NT_AMD_AMDGPU_METADATA`. The `AMDGPUTargetAsmStreamer` emits this metadata during assembly using the MSGPACK binary format (for CO v3/v4). For a kernel with 24-byte kernarg and default resource usage:

```yaml
amdhsa.version:
  - 1
  - 2
amdhsa.kernels:
  - .name:            _Z9vectorAddPfS_S_i
    .symbol:          _Z9vectorAddPfS_S_i.kd
    .kernarg_segment_size: 24
    .group_segment_fixed_size: 0
    .private_segment_fixed_size: 0
    .max_flat_workgroup_size: 1024
    .wavefront_size:  64
    .sgpr_count:      24
    .vgpr_count:      8
    .num_spilled_sgprs: 0
    .num_spilled_vgprs: 0
    .args:
      - .name:    a
        .type_name: float*
        .size:    8
        .offset:  0
        .value_kind: global_buffer
        .address_qualifier: global
      - .name:    n
        .type_name: int
        .size:    4
        .offset:  16
        .value_kind: by_value
```

The HSA runtime reads this metadata to set up kernel launch parameters before submitting an AQL dispatch packet. The `.sgpr_count` and `.vgpr_count` fields determine occupancy: too high a count reduces the number of wavefronts that can reside on a Compute Unit simultaneously, limiting latency hiding.

### 49.3.6 AMDGPUTargetMachine and the Backend Pass Pipeline

`AMDGPUTargetMachine` (declared in `llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.h`) is the base class; `GCNTargetMachine` inherits from it for GCN/CDNA targets. The backend pass pipeline includes AMD-specific passes not found in other targets:

| Pass | Purpose |
|------|---------|
| `AMDGPUDivergenceAnalysis` | Classify IR values as uniform/divergent |
| `AMDGPUDAGToDAGISel` | SelectionDAG instruction selection; handles VGPR/SGPR register class assignment |
| `SIFoldOperands` | Fold SGPR inline constant operands into VALU/SMEM instructions |
| `SILoadStoreOptimizer` | Merge adjacent memory operations into 128-bit (4-dword) loads/stores |
| `SIShrinkInstructions` | Convert 64-bit VOP3 encoding to 32-bit VOP1/VOP2 where immediates fit |
| `SIInsertSkips` | Insert `s_cbranch_execz` (skip if EXEC=0) around VALU sequences in divergent regions |
| `GCNNSAReassign` | Non-sequential addressing reassignment for image instructions |
| `SIPostRAScheduler` | Post-register-allocation VALU/SMEM/VMEM scheduling for latency hiding |
| `GCNCreateVOPD` | Fuse VALU pairs into dual-issue VOPD (GFX11+ RDNA3) |

SGPR (scalar GPR) vs VGPR (vector GPR) assignment is the defining challenge of the AMDGPU backend. The `GCNRegBankCombiner` resolves register bank conflicts: when a VALU instruction requires a SGPR operand that analysis cannot prove uniform, it inserts `v_readfirstlane_b32` instructions to extract lane-0 of a VGPR into an SGPR. This insertion is correct but expensive; avoiding it requires writing kernels with clear divergence structure.

---

## 49.4 Fat Binary and Device Linking for HIP

### 49.4.1 The Compilation Model

HIP uses the same dual-compilation model as CUDA: each translation unit is compiled twice — once for the host (targeting the host CPU triple, typically `x86_64-unknown-linux-gnu`) and once for each device architecture. The Clang driver coordinates this through `OffloadAction` nodes in the compilation DAG.

The full pipeline for `clang++ -x hip --offload-arch=gfx942 foo.hip`:

```
foo.hip
  ├─ [host pass]   clang -x hip -D__HIP_HOST__         → foo.host.bc → foo.host.o
  └─ [device pass] clang -x hip --target amdgcn-amd-amdhsa -mcpu=gfx942
                     → foo.device.bc
                     → llvm-link (+ rocm-device-libs bitcode)
                     → opt   (device optimisations: inlining + DCE of host code)
                     → llc   → foo.gfx942.o  (ELF with AMDGPU ISA + HSA metadata)
                     → lld   → foo.gfx942.hsaco (HSA code object)
clang-offload-bundler foo.host.o foo.gfx942.hsaco → foo.o (fat object)
```

The host compilation defines `__HIP_HOST__` and compiles the `__global__` kernel stubs as external symbols that call `hipLaunchKernelGGL` at runtime. The device compilation defines `__HIP_DEVICE_COMPILE__` and discards all `__host__` functions.

### 49.4.2 clang-offload-bundler

`clang-offload-bundler` (source: `clang/tools/clang-offload-bundler/ClangOffloadBundler.cpp`) creates a fat object containing host code and one or more device blobs. The bundle format uses a magic header followed by a table of contents embedded as a `.llvm_offloading` ELF section. Each entry is identified by a target string:

```
__CLANG_OFFLOAD_BUNDLE____START__  host-x86_64-unknown-linux-gnu
  [host ELF content]
__CLANG_OFFLOAD_BUNDLE____START__  hipv4-amdgcn-amd-amdhsa--gfx942
  [device HSA code object]
```

The `hipv4-` prefix identifies HIP code object format version 4. The double dash before `gfx942` is the triple-separator for the environment component. Unbundling a device image:

```bash
clang-offload-bundler --unbundle \
  --type=o \
  --input=foo.o \
  --targets=hipv4-amdgcn-amd-amdhsa--gfx942 \
  --output=foo.gfx942.hsaco
```

For multi-architecture fat binaries targeting both gfx942 and gfx90a:

```bash
clang++ -x hip --offload-arch=gfx942 --offload-arch=gfx90a \
        -O2 -o matmul matmul.hip
```

The bundler receives both device objects and packs them into a single host `.o` with two device entries. At runtime, `hipGetDeviceProperties` queries the GPU architecture and `__hipRegisterFatBinary` selects the matching device image.

### 49.4.3 Device Linker and LLD

When multiple translation units are compiled separately, their device code objects must be linked. The HIP toolchain invokes `lld` with the AMDGPU target:

```bash
ld.lld --target=amdgcn-amd-amdhsa \
    -plugin-opt=mcpu=gfx942 \
    -shared foo.gfx942.o bar.gfx942.o \
    -o combined.gfx942.hsaco
```

LLD understands AMD code object (CO) v3/v4 format: kernel descriptor placement at offsets specified by the ABI, note sections containing MSGPACK HSA metadata, and the `.dynsym` table required by the HSA runtime's `hsa_code_object_get_symbol` for kernel lookup. The code object is a standard ELF shared object, though AMD extensions to the ELF note sections carry the HSA-specific metadata.

### 49.4.4 Relocatable Device Code (-fgpu-rdc)

By default, HIP uses monolithic device compilation: each source file produces a fully linked device image with no undefined external device symbols. This precludes cross-TU device-function calls at link time. The `-fgpu-rdc` flag (relocatable device code) enables separate compilation:

```bash
# Compile with RDC: device code not linked until final step
clang++ -x hip --offload-arch=gfx942 -fgpu-rdc -c foo.hip -o foo.o
clang++ -x hip --offload-arch=gfx942 -fgpu-rdc -c bar.hip -o bar.o
# Final link: host object linking + device-side linking
clang++ --offload-arch=gfx942 -fgpu-rdc foo.o bar.o \
        -o app -L/opt/rocm/lib -lamdhip64
```

With `-fgpu-rdc`, device relocations are preserved as LLVM IR bitcode inside the fat object (rather than as a compiled ISA blob). The final link step:
1. Extracts all device LLVM IR modules from the fat objects
2. Links them together with `llvm-link`, resolving cross-TU device symbols
3. Runs device-side LTO: inlining, dead code elimination, whole-program VGPR/SGPR analysis
4. Produces a single device code object via `llc` + `lld`

The LTO pass enables cross-TU inlining of small device helper functions, which is critical for GPU performance: function call overhead on GPU (the `s_call`/`s_ret` pair) is significant because return addresses consume SGPRs.

### 49.4.5 Runtime Registration

The host code generated by Clang registers kernel entry points with the HIP runtime at program startup via constructors. For each kernel, Clang generates a host stub that is the target of `hipLaunchKernelGGL` calls, plus a registration constructor:

```cpp
// Synthesized by Clang's HIP host codegen (simplified)
static const char __hip_gpu_bin[] = {  /* fat binary bytes */ };

__attribute__((constructor(101))) static void __hip_register_globals(void) {
    void **handle = __hipRegisterFatBinary((void*)__hip_gpu_bin);
    // Register each kernel: maps stub pointer → mangled device name
    __hipRegisterFunction(
        handle,
        (void*)_Z9vectorAddPfS_S_i_host_stub,  // host stub address
        (char*)"_Z9vectorAddPfS_S_i",           // host name
        "_Z9vectorAddPfS_S_i",                  // device name
        -1, nullptr, nullptr, nullptr, nullptr, nullptr);
    __hipRegisterFatBinaryEnd(handle);
}
```

`__hipRegisterFatBinary` calls the HIP runtime, which calls `hsa_code_object_deserialize` to load the embedded device ELF and then `hsa_executable_load_code_object` to link it against the HSA runtime. `__hipRegisterFunction` inserts an entry into the function table mapping host stub addresses to HSA kernel objects.

### 49.4.6 hipcc vs Raw Clang

`hipcc` is a Perl/Python wrapper around Clang that sets default paths for the ROCm installation:

```bash
# hipcc is approximately equivalent to:
clang++ -x hip                                    \
  --offload-arch=$(rocm_agent_enumerator)          \
  -I/opt/rocm/include -I/opt/rocm/hip/include     \
  -L/opt/rocm/lib -lamdhip64                      \
  --rocm-path=/opt/rocm                           \
  "$@"
```

The `--rocm-path` argument tells Clang where to find the `rocm-device-libs` bitcode files and the ROCm header tree. The `rocm_agent_enumerator` tool queries the system for installed AMD GPUs and returns their GCN architecture identifiers, so `--offload-arch=$(rocm_agent_enumerator)` dynamically compiles for all present hardware. Raw Clang invocation is preferable in reproducible build systems that pin the ROCm version.

---

## 49.5 ROCm and HIP Ecosystem

### 49.5.1 Compute Libraries

AMD ships a suite of HIP-accelerated libraries that parallel the CUDA ecosystem:

| ROCm library | CUDA equivalent | Domain |
|-------------|-----------------|--------|
| `rocBLAS` | cuBLAS | Dense linear algebra (GEMM, TRSM, SYMM, etc.) |
| `rocSPARSE` | cuSPARSE | Sparse linear algebra (SpMM, SpMV, etc.) |
| `rocFFT` | cuFFT | Fast Fourier transform (R2C, C2C) |
| `rocRAND` | cuRAND | Pseudo/quasi-random number generation |
| `rocSOLVER` | cuSOLVER | Dense matrix factorisation (LU, QR, SVD) |
| `MIOpen` | cuDNN | Deep learning primitives (conv, BN, RNN) |
| `hipBLAS` | cuBLAS | Cross-platform portability wrapper |
| `hipSPARSE` | cuSPARSE | Cross-platform portability wrapper |
| `hipCUB` | CUB | Device-level primitives (sort, reduce, scan) |
| `rocThrust` | Thrust | High-level parallel algorithms |

The `hip*` prefixed libraries are cross-platform wrappers: on `HIP_PLATFORM=amd` they call the underlying `roc*` implementations; on `HIP_PLATFORM=nvidia` they call the corresponding `cu*` implementations. This makes them the recommended choice for portable HPC code that targets both AMD and NVIDIA hardware.

### 49.5.2 HIP Graphs and Stream Capture

HIP 5.0+ supports the CUDA Graph API through `hipGraph`:

```cpp
hipGraph_t graph;
hipGraphExec_t exec;
hipStream_t stream;

hipStreamCreate(&stream);
// Begin capturing all API calls into a graph
hipStreamBeginCapture(stream, hipStreamCaptureModeGlobal);
hipblasGemmEx(blas_handle, HIPBLAS_OP_N, HIPBLAS_OP_N,
              m, n, k, &alpha, A, HIPBLAS_R_16F, m,
              B, HIPBLAS_R_16F, k, &beta,
              C, HIPBLAS_R_32F, m, HIPBLAS_R_32F,
              HIPBLAS_GEMM_DEFAULT);
hipStreamEndCapture(stream, &graph);

// Instantiate: produces an optimised execution plan
hipGraphInstantiate(&exec, graph, nullptr, nullptr, 0);

// Launch 1000 times without re-recording overhead
for (int i = 0; i < 1000; ++i)
    hipGraphLaunch(exec, stream);

hipStreamSynchronize(stream);
hipGraphExecDestroy(exec);
hipGraphDestroy(graph);
```

`hipStreamCapture` intercepts HIP runtime API calls during capture and builds a DAG of operation nodes. `hipGraphInstantiate` converts this to an HSA-optimised execution plan: kernels that can overlap are identified, memory copies are fused where possible, and a single AQL command chain replaces the sequence of individual kernel dispatches. Repeated launches avoid the overhead of device function lookup and kernarg setup that occurs on each individual `hipLaunchKernelGGL` call.

### 49.5.3 Managed Memory and Unified Virtual Addressing

HIP managed memory mirrors CUDA unified memory:

```cpp
float *data;
// Allocated in device-accessible address space, migrates on access
hipMallocManaged(&data, N * sizeof(float));

// Or with the managed attribute on global variables:
__hip_managed__ float managed_array[1024];
```

On APUs (integrated CPU+GPU sharing physical DRAM), managed memory is true zero-copy with no migration. On discrete GPUs, the `amdgpu` driver's SVM (Shared Virtual Memory) subsystem handles page migration via HSA signals and the KFD fault handler: when the host accesses a device-resident page, the driver migrates it to host memory; when a device kernel accesses a host-resident page, it triggers migration to device memory or falls back to the XGMI/PCIe path.

`hipMemAdvise()` hints (e.g., `hipMemAdviseSetReadMostly`, `hipMemAdviseSetPreferredLocation`) instruct the SVM subsystem to place data optimally and set up read-only replicas, avoiding unnecessary page migration for read-only data sets.

### 49.5.4 rocm-smi and GPU Monitoring

`rocm-smi` (ROCm System Management Interface) provides device status information analogous to `nvidia-smi`:

```bash
rocm-smi                          # GPU utilization, temperature, memory
rocm-smi --showmeminfo vram       # VRAM used/total
rocm-smi --showclkfrq             # Clock frequencies
rocm-smi --showpower              # Power consumption
rocm-smi --setperflevel high      # Set performance level
```

The `rocm-smi-lib` library provides a programmatic interface; the `rsmi_dev_gpu_clk_freq_get()` and `rsmi_dev_power_ave_get()` functions expose the same information to applications for auto-tuning.

---

## 49.6 HIP vs CUDA Portability

### 49.6.1 The Mapping Layer

The `hip/hip_runtime.h` header provides a complete mapping from CUDA names to HIP names when `HIP_PLATFORM=amd`. The mapping covers:

- **Memory management**: `cudaMalloc` → `hipMalloc`, `cudaFree` → `hipFree`, `cudaMallocAsync` → `hipMallocAsync`, `cudaMemcpy` → `hipMemcpy`, `cudaMemcpyAsync` → `hipMemcpyAsync`
- **Kernel launch**: `cudaLaunchKernel` → `hipLaunchKernel`; the `<<<grid, block, sharedMem, stream>>>` syntax works identically via the compiler's built-in transformation
- **Device management**: `cudaSetDevice` → `hipSetDevice`, `cudaGetDeviceProperties` → `hipGetDeviceProperties`, `cudaDeviceSynchronize` → `hipDeviceSynchronize`
- **Events and streams**: `cudaEventCreate` → `hipEventCreate`, `cudaStreamCreate` → `hipStreamCreate`, `cudaStreamWaitEvent` → `hipStreamWaitEvent`
- **Error handling**: `cudaGetLastError` → `hipGetLastError`, `cudaGetErrorString` → `hipGetErrorString`, `cudaSuccess` → `hipSuccess`

```cpp
// Portable error-checking macro pattern
#define HIPCHECK(cmd) do {                              \
    hipError_t err = (cmd);                            \
    if (err != hipSuccess) {                           \
        fprintf(stderr, "HIP error %s at %s:%d\n",    \
            hipGetErrorString(err), __FILE__, __LINE__);\
        exit(1);                                       \
    }                                                  \
} while (0)

// Usage
HIPCHECK(hipMalloc(&d_a, N * sizeof(float)));
HIPCHECK(hipMemcpy(d_a, h_a, N * sizeof(float), hipMemcpyHostToDevice));
```

### 49.6.2 Warp Size Divergence

The most significant portability hazard between CUDA and HIP is the wavefront/warp size difference:

| Platform | SIMT width | Ballot type | `warpSize` constant |
|----------|-----------|-------------|---------------------|
| CUDA (NVPTX) | 32 | `unsigned` (32 bits) | 32 |
| AMDGPU GFX9/CDNA1/2 | 64 | `unsigned long long` (64 bits) | 64 |
| AMDGPU GFX10+ (RDNA) | 32 or 64 | Configurable per shader | 32 default |
| AMDGPU GFX11+ (RDNA3) | 32 | Fixed 32 on RDNA3 | 32 |

The `warpSize` device variable (64 on CDNA, 32 on RDNA2+, 32 on CUDA) should be used instead of literal `32`. Ballot operations require portable type handling:

```cpp
// Portable ballot result type
#if defined(__HIP_DEVICE_COMPILE__) && defined(__AMDGCN__)
  #if defined(__gfx10__) || defined(__gfx11__) || defined(__gfx12__)
    typedef unsigned ballot_t;    // 32-wide RDNA wavefront
  #else
    typedef unsigned long long ballot_t;  // 64-wide CDNA/GFX9
  #endif
#else
  typedef unsigned ballot_t;              // CUDA 32-wide warp
#endif

// Use __ballot_sync on CUDA; __ballot on HIP (no sync mask needed)
#ifdef __HIP_DEVICE_COMPILE__
  ballot_t mask = __ballot(pred);
#else
  ballot_t mask = __ballot_sync(0xFFFFFFFF, pred);
#endif
```

HIP 5.6+ provides `__hip_ballot_mask_t` as a portable opaque type for this purpose. AMD recommends using the HIP cooperative groups API (`hip/hip_cooperative_groups.h`) for portable warp-level primitives.

### 49.6.3 Async Memory Allocation

`hipMallocAsync` (HIP 5.2+, matching CUDA 11.2's `cudaMallocAsync`) enables stream-ordered memory allocation with memory pools:

```cpp
hipMemPoolProps poolProps = {};
poolProps.allocType = hipMemAllocationTypePinned;
poolProps.location.type = hipMemLocationTypeDevice;
poolProps.location.id = device;

hipMemPool_t pool;
hipMemPoolCreate(&pool, &poolProps);

float *d_buf;
hipMallocFromPoolAsync(&d_buf, N * sizeof(float), pool, stream);
hipLaunchKernelGGL(compute, grid, block, 0, stream, d_buf, N);
hipFreeAsync(d_buf, stream);  // Returns to pool, does not free physical memory
```

The memory pool reduces allocation overhead for applications that repeatedly allocate and free device buffers: freed memory is retained in the pool and reused for subsequent allocations on the same stream, eliminating the HSA `hsa_memory_allocate`/`hsa_memory_free` round-trip.

---

## 49.7 Profiling and Debugging HIP

### 49.7.1 rocprof — The AMD Profiler

`rocprof` is the primary performance profiling tool for HIP, analogous to `nvprof`/Nsight Systems:

```bash
# Collect kernel timing and hardware counters
rocprof --stats --timestamp on \
        -i counters.txt \
        --output-format csv \
        ./matmul

# contents of counters.txt (PMC = performance monitoring counter):
# pmc: SQ_INSTS_VALU, SQ_INSTS_VMEM_RD, SQ_INSTS_VMEM_WR
# pmc: FETCH_SIZE, WRITE_SIZE, TA_TA_BUSY
# range: 0:1
```

Key hardware performance counters for AMDGPU GFX9:

| Counter | Meaning | Bottleneck signal |
|---------|---------|------------------|
| `SQ_INSTS_VALU` | VALU (vector ALU) instructions issued | Compute-bound |
| `SQ_INSTS_VMEM_RD` | VMEM (vector memory) reads issued | Memory-bound |
| `SQ_INSTS_SMEM_RD` | SMEM (scalar memory) reads issued | Constant data usage |
| `TA_TA_BUSY` | Texture Addressing unit busy cycles | Texture/image bound |
| `TCC_HIT` | L2 cache hits | Cache efficiency |
| `GRBM_GUI_ACTIVE` | GPU activity (useful cycles) | Overall occupancy |

`rocprof` wraps the `roctracer` API (runtime API tracing) and the `rocprofiler` API (hardware counter collection). Output is CSV or JSON, viewable in `perfetto` trace viewer or AMD's Radeon GPU Profiler tool.

### 49.7.2 omniperf — Roofline and Microarchitecture Analysis

`omniperf` provides roofline model analysis and detailed microarchitecture breakdowns across multiple profiling passes:

```bash
# Profile (may require root or cap_perfmon capability)
omniperf profile --name matmul -- ./matmul

# Analyze with web interface
omniperf analyze --path workloads/matmul/mi300/

# Text-mode roofline
omniperf analyze --path workloads/matmul/mi300/ \
    --roof-only --roofline-type empirical
```

`omniperf` collects 200+ hardware counters over several profiling runs (different counter groups per pass) and computes derived metrics: arithmetic intensity (FLOPs/byte), memory bandwidth utilisation, VALU efficiency (useful VALU cycles / total cycles), and wavefront occupancy. The roofline chart shows whether a kernel is compute-bound (above the compute ceiling) or memory-bound (below the memory bandwidth roof).

### 49.7.3 rocgdb — Device Debugging

`rocgdb` extends GDB with AMD GPU debugging capabilities via the KFD trap handler:

```gdb
$ rocgdb ./matmul
(rocgdb) break _Z9vectorAddPfS_S_i
(rocgdb) run
# HIP application starts, GPU kernel hits breakpoint
(rocgdb) info threads
# Lists CPU threads AND GPU wavefronts, e.g.:
# Thread 2.1:1:0:0:0 (wave: SIMD 0, CU 1, wave slot 0, lane 0)
(rocgdb) thread 2.1:1:0:0:0     # Switch to specific wavefront
(rocgdb) print $s0               # Print SGPR 0 (uniform value)
(rocgdb) print $v0               # Print VGPR 0 (all 64 lanes shown)
(rocgdb) print $v0[lane 5]       # Print lane 5 of VGPR 0
(rocgdb) info registers sgpr     # Dump all SGPRs
(rocgdb) info registers vgpr     # Dump all VGPRs
```

Device breakpoints work by injecting `s_trap 1` instructions into the ISA stream at the point specified by the DWARF debug information. The `amdgpu` KFD handles the hardware trap, suspending the GPU and notifying `rocgdb`. Debug information uses DWARF 4 with AMD extensions for VGPR lane mapping (`DW_OP_LLVM_aspace_bregx` location expressions).

LLDB with the AMD GPU plugin (`lldb-mi` in the ROCm stack) provides an alternative debugger interface, required for some IDE integrations (Eclipse, CLion) and for the `roc-gdb-scripts` extension for Python-based wavefront analysis.

### 49.7.4 roctx and Application Tracing

`roctx` (ROCm TX — tracing extensions) provides lightweight user-mode range markers:

```cpp
#include <roctx.h>

void run_matrix_multiply(hipStream_t stream, int iter) {
    char label[64];
    snprintf(label, sizeof(label), "GEMM iter=%d", iter);
    roctxRangePush(label);           // Open named range
    hipblasGemmEx(handle, ...);
    roctxMark("GEMM submitted");     // Instantaneous marker
    roctxRangePop();                 // Close range
}
```

`roctxRangePush`/`roctxRangePop` insert HSA signal markers that `rocprof --roctx-trace` captures alongside kernel dispatch timestamps, producing a timeline trace viewable in Perfetto. The `roctxRangePushA` variant takes a pre-allocated string for zero-allocation tracing on hot paths. The `rocprofiler-sdk` (ROCm 6.0+) provides a newer, more flexible API replacing the older `roctracer`/`roctx` combo.

### 49.7.5 Address Sanitizer for HIP

ROCm supports AddressSanitizer for device code from ROCm 5.0+:

```bash
clang++ -x hip --offload-arch=gfx942 \
        -fsanitize=address -shared-libasan \
        -O1 -g -o matmul_asan matmul.hip
ASAN_OPTIONS=protect_shadow_gap=0:halt_on_error=1 ./matmul_asan
```

The device ASan runtime instruments load/store instructions with shadow memory lookups. Out-of-bounds device memory accesses generate ASan reports with device-side stack traces (using the DWARF debug info), which are symbolized and printed to stderr. The shadow memory lives in device global memory; the instrumentation overhead is approximately 2x in memory and 1.5x in runtime. The `-fsanitize=address` flag is passed through to both host and device compilations via the HIP toolchain.

---

## 49.8 Summary

- HIP provides near-complete CUDA source compatibility on AMD GPUs. Clang implements HIP by reusing the CUDA semantic analysis infrastructure in `SemaCUDA.cpp` and `SemaType.cpp`, with the `CUDAFunctionTarget` enum and `HIPAttr` handling both CUDA and HIP annotated functions.
- AMDGPU address spaces are defined in `llvm/include/llvm/Support/AMDGPUAddrSpace.h`: flat=0, global=1, LDS=3, constant=4, private=5. The data layout's `A5` specifier directs alloca instructions to private address space.
- The defining backend challenge is the SGPR/VGPR register split driven by divergence analysis. `AMDGPUDivergenceAnalysis` classifies values; `SIFoldOperands` and `SIInsertSkips` clean up divergent control flow.
- `llvm.amdgcn.*` intrinsics cover thread ID, barrier (`s_barrier`), warp shuffle (`ds_bpermute`), ballot (`ballot.i64`), and FP atomic operations. HSA kernel metadata in ELF notes drives occupancy and runtime launch setup.
- Fat binaries bundle host and device code using `clang-offload-bundler` with `hipv4-` target prefix. The `-fgpu-rdc` flag enables separate device compilation with cross-TU LTO at link time.
- The ROCm ecosystem provides profiling (`rocprof`, `omniperf`), debugging (`rocgdb`, LLDB GPU plugin), tracing (`roctx`), device ASan, and `rocm-smi` for system management.
- The most significant CUDA/HIP portability hazard is wavefront width: 64 on GFX9/CDNA, 32 on RDNA2/3 and CUDA. Ballot types and warp-level reductions must use `warpSize` dynamically or feature macros.

---

*Cross-references:*
- [Chapter 48 — Clang as a CUDA Compiler](../part-07-clang-multilang/ch48-clang-as-cuda-compiler.md) — the shared Sema infrastructure and NVPTX backend for comparison
- [Chapter 50 — Clang as SYCL, OpenCL, and OpenMP-Offload](../part-07-clang-multilang/ch50-clang-as-sycl-opencl-openmp-offload.md) — alternative offload models sharing the AMDGPU backend
- [Chapter 103 — The AMDGPU Backend](../../part-15-targets/ch103-amdgpu-backend.md) — deep dive into `AMDGPUTargetMachine`, instruction selection, and register allocation

*Reference links:*
- [AMDGPUTargetMachine.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp)
- [AMDGPUAddrSpace.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/AMDGPUAddrSpace.h)
- [SemaCUDA.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaCUDA.cpp)
- [clang-offload-bundler](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/tools/clang-offload-bundler/ClangOffloadBundler.cpp)
- [AMDGPUDivergenceAnalysis.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AMDGPU/AMDGPUDivergenceAnalysis.cpp)
- [ROCm Device Libraries](https://github.com/ROCm/ROCm-Device-Libs)
- [HIP Programming Guide](https://rocm.docs.amd.com/projects/HIP/en/latest/user_guide/hip_porting_guide.html)
- [AMDGPU LLVM User Guide](https://llvm.org/docs/AMDGPUUsage.html)
- [HSA Runtime Specification](https://hsafoundation.com/standards/)
