# Chapter 48 — Clang as a CUDA Compiler

*Part VII — Clang as a Multi-Language Compiler*

CUDA is the dominant programming model for GPU computing, and Clang has supported it as a first-class compilation target since LLVM 3.8. Unlike NVIDIA's `nvcc`, which preprocesses `.cu` files through a host compiler as a separate subprocess, Clang compiles CUDA natively: a single compiler invocation parses the entire `.cu` file, builds one unified AST, and then generates separate host and device code paths. This architecture gives Clang access to full C++23 on both sides, accurate diagnostics, and integration with the entire LLVM optimization and analysis pipeline. This chapter dissects every layer of that stack — from driver orchestration and Sema attribute enforcement, through CodeGen stub synthesis and NVVM intrinsic lowering, to PTX emission by the NVPTX backend and fat binary packaging for the CUDA runtime.

---

## 48.1 The CUDA Compilation Model

### 48.1.1 Single Source, Two Code Paths

A `.cu` file is standard C++ augmented with CUDA extensions. The same translation unit contains functions that execute on the host (x86-64 or AArch64) and functions marked `__global__` or `__device__` that execute on the GPU. Clang's compiler driver performs a **dual compilation** of each `.cu` file: one invocation targeting `nvptx64-nvidia-cuda` (the device pass) and one targeting the native host triple (the host pass). Both invocations share the same source file but are distinct `cc1` processes.

The driver command `clang++ -x cuda --cuda-gpu-arch=sm_90 foo.cu` orchestrates the following pipeline, visible with `-###`:

```
# Pass 1: device compilation (NVPTX target)
clang -cc1 -triple nvptx64-nvidia-cuda -aux-triple x86_64-pc-linux-gnu \
      -fcuda-is-device -target-cpu sm_90 -target-feature +ptx90 \
      -mlink-builtin-bitcode /usr/local/cuda/nvvm/libdevice/libdevice.10.bc \
      -include __clang_cuda_runtime_wrapper.h -S foo.cu -o foo-sm_90.s

# Pass 2: ptxas assembles PTX -> SASS cubin
ptxas -m64 --gpu-name sm_90 -O0 -o foo-sm_90.o foo-sm_90.s

# Pass 3: fatbinary wraps cubin(s) into a single binary blob
fatbinary -64 --create foo.fatbin \
          --image3=kind=elf,sm=90,file=foo-sm_90.o

# Pass 4: host compilation, embedding fat binary
clang -cc1 -triple x86_64-pc-linux-gnu \
      -fcuda-include-gpubinary foo.fatbin \
      -emit-obj foo.cu -o foo.o

# Pass 5: link
clang-linker-wrapper --host-triple=x86_64-pc-linux-gnu \
      --linker-path=/usr/bin/ld ... foo.o -o foo
```

The device pass receives `-fcuda-is-device`, which sets `LangOptions::CUDAIsDevice = true` throughout compilation. The host pass embeds the fat binary blob via `-fcuda-include-gpubinary`, causing the CodeGen layer to emit a `.nv_fatbin` section in the resulting object file.

### 48.1.2 Key Driver Flags

| Flag | Meaning |
|------|---------|
| `--cuda-gpu-arch=sm_90` | Target SM architecture; may be repeated for multi-arch |
| `--cuda-device-only` | Stop after device compilation, emit PTX or device object |
| `--cuda-host-only` | Skip device compilation entirely |
| `--cuda-path=<dir>` | Override CUDA toolkit search path |
| `--cuda-noopt-device-debug` | Disable `ptxas` optimizations for device debug |
| `-fgpu-rdc` | Enable relocatable device code (separate device linking) |
| `--cuda-include-ptx=sm_90` | Embed PTX source (not cubin) for JIT compilation at load time |

### 48.1.3 CUDA Toolkit Detection

The driver's `CudaInstallationDetector` class ([`clang/include/clang/Driver/CudaInstallationDetector.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/CudaInstallationDetector.h)) probes candidate paths including `/usr/local/cuda`, `/usr/local/cuda-*`, and the `CUDA_PATH` environment variable. It detects the installed CUDA version by parsing `version.txt`, stores the version in `CudaVersion` (an enum in [`clang/include/clang/Basic/Cuda.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/Cuda.h)), and warns if the SDK is newer than the latest partially-supported version. As of Clang 22, `CudaVersion::FULLY_SUPPORTED = CUDA_128` and `CudaVersion::PARTIALLY_SUPPORTED = CUDA_129`. The detector also locates `libdevice.10.bc` for each architecture, accessed via `getLibDeviceFile(StringRef Gpu)`, which performs a lookup into an internal `LibDeviceMap`.

Architecture support is validated by `CheckCudaVersionSupportsArch(OffloadArch)`. The `OffloadArch` enum in [`clang/include/clang/Basic/OffloadArch.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/OffloadArch.h) lists every supported SM from `SM_20` through the latest Blackwell variants (`SM_120a`, `SM_121a`) alongside AMD GFX architectures (for HIP).

---

## 48.2 CUDA Language Extensions in Clang

### 48.2.1 Function Space Qualifiers

CUDA introduces four function qualifiers that determine where a function executes and from where it may be called:

| CUDA Qualifier | Clang Attribute | Execution Space | Callable From |
|----------------|-----------------|-----------------|---------------|
| `__global__` | `CUDAGlobalAttr` | Device | Host only (kernel launch) |
| `__device__` | `CUDADeviceAttr` | Device | Device only |
| `__host__` | `CUDAHostAttr` | Host | Host only |
| `__host__ __device__` | Both attrs | Both | Both (overload resolution) |

These map to `CUDAFunctionTarget` enum values defined in `Cuda.h`: `Device`, `Global`, `Host`, `HostDevice`, and `InvalidTarget`. Clang stores the attributes directly on `FunctionDecl` nodes; the presence or absence of these attributes is what `SemaCUDA::IdentifyTarget()` inspects rather than any separate side table.

The `__global__` qualifier additionally requires the return type to be `void` and forbids virtual functions, explicit template instantiations of non-empty destructors on the device side, and dynamic memory operations in older CUDA versions.

### 48.2.2 Variable Space Qualifiers

Storage class qualifiers control which GPU memory space a variable occupies:

| CUDA Qualifier | Clang Attribute | LLVM Address Space | Lifetime |
|----------------|-----------------|---------------------|----------|
| `__device__` | `CUDADeviceAttr` | 1 (global) | Program |
| `__constant__` | `CUDAConstantAttr` | 4 (constant) | Program, read-only |
| `__shared__` | `CUDASharedAttr` | 3 (shared) | Thread block |
| `__managed__` | `CUDAManagedAttr` | 1 (with UVM) | Program, accessible from host |

The `LangAS` enum in [`clang/include/clang/Basic/AddressSpaces.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/AddressSpaces.h) maps these to `LangAS::cuda_device`, `LangAS::cuda_constant`, and `LangAS::cuda_shared`. The CodeGen layer translates these to NVPTX IR address spaces (1, 4, 3 respectively).

A `__shared__` variable is always implicitly static and cannot have a non-trivial constructor or destructor. Clang enforces this in `SemaCUDA::checkAllowedInitializer(VarDecl*)`. A `__constant__` variable may have a constant initializer evaluated at compile time; `SemaCUDA::MaybeAddConstantAttr()` may implicitly add `CUDAConstantAttr` to `const` device variables when appropriate.

### 48.2.3 Execution Configuration Syntax

The triple-angle-bracket `<<<grid, block, sharedMem, stream>>>` syntax is parsed by the Clang parser's `ParseCUDAKernelCallExpr` path. The parser calls `SemaCUDA::ActOnExecConfigExpr()` to build the configuration expression, then wraps the kernel call into a `CUDAKernelCallExpr` AST node ([`clang/include/clang/AST/ExprCXX.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/ExprCXX.h)).

`CUDAKernelCallExpr` inherits from `CallExpr` and carries a `Config` sub-expression (the `<<<...>>>` arguments). During CodeGen, this node drives the stub function call sequence: the grid and block dimensions are packed into two `dim3` values, `__cudaPushCallConfiguration()` is called with those values, and then the device stub function (generated for each `__global__`) calls `__cudaPopCallConfiguration()` and `cudaLaunchKernel()`.

### 48.2.4 Launch Bounds

`__launch_bounds__(maxTPB, minBlocks)` becomes a `CUDALaunchBoundsAttr` on the `FunctionDecl`. During NVPTX CodeGen, this is translated into `!maxntidx`, `!maxntidy`, `!maxntidz`, and `!minctasm` NVVM metadata nodes attached to the function, which `ptxas` uses to optimize register allocation.

---

## 48.3 Sema CUDA Handling

### 48.3.1 The SemaCUDA Class

All CUDA-specific semantic analysis is concentrated in `SemaCUDA` ([`clang/include/clang/Sema/SemaCUDA.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/SemaCUDA.h)), a `SemaBase` subclass introduced with the refactored Sema split in LLVM 18. It holds:

- `LocsWithCUDACallDiags`: a `DenseSet<FunctionDeclAndLoc>` tracking source locations where a deferred diagnostic has already been emitted, preventing duplicate warnings.
- `DeviceKnownEmittedFns`: an inverse call graph mapping emitted callees to their callers, used to propagate deferred diagnostics when a host-device function is ultimately compiled for the device.
- `CurCUDATargetCtx`: a `CUDATargetContext` struct recording the current global compilation context kind (e.g., `CTCK_InitGlobalVar`) when analyzing code outside any function body.

### 48.3.2 Target Identification

`SemaCUDA::IdentifyTarget(const FunctionDecl *D)` returns a `CUDAFunctionTarget` by examining the function's attribute set. The logic is nuanced:

- If `D` has `CUDAGlobalAttr` → `CUDAFunctionTarget::Global`
- If `D` has both `CUDAHostAttr` and `CUDADeviceAttr` → `CUDAFunctionTarget::HostDevice`
- If `D` has only `CUDADeviceAttr` → `CUDAFunctionTarget::Device`
- If `D` has only `CUDAHostAttr` → `CUDAFunctionTarget::Host`
- Otherwise → `CUDAFunctionTarget::Host` (the default for unannotated C++ functions)

The overload `IdentifyTarget(const ParsedAttributesView&)` is used during declaration parsing before the full `FunctionDecl` is built, to enable early rejection of invalid combinations.

For variables, `SemaCUDA::IdentifyTarget(const VarDecl *D)` returns a `CUDAVariableTarget`:
- `CVT_Device` — emitted on device side, with a host-side shadow
- `CVT_Host` — host only
- `CVT_Both` — emitted in both address spaces with distinct addresses
- `CVT_Unified` — `__managed__` variables using unified virtual addressing

### 48.3.3 Call Legality and Preferences

`SemaCUDA::CheckCall(SourceLocation Loc, FunctionDecl *Callee)` enforces the CUDA execution model constraints. It delegates to `IdentifyPreference(Caller, Callee)` which returns a `CUDAFunctionPreference`:

```
enum CUDAFunctionPreference {
  CFP_Never,      // Invalid: host calling __device__, or vice versa
  CFP_WrongSide,  // __host__ __device__ calling same-side-only function
  CFP_HostDevice, // Any call to a __host__ __device__ function
  CFP_SameSide,   // __host__ __device__ caller, matching compilation mode
  CFP_Native,     // host→host or device→device (best preference)
};
```

`CFP_Never` triggers an immediate error. `CFP_WrongSide` creates a **deferred diagnostic** attached to the caller — it fires only if and when the caller is actually codegen'd for the wrong side. This deferred-diagnostic mechanism is critical for `__host__ __device__` template code that is instantiated but never compiled for the conflicting side.

Overload resolution among multiple candidates uses `SemaCUDA::EraseUnwantedMatches()`, which prunes the candidate set to retain only those with the highest `CUDAFunctionPreference`, ensuring that `__device__` overloads are preferred in device context and `__host__` in host context.

### 48.3.4 Implicit Host-Device Attributes

`SemaCUDA::maybeAddHostDeviceAttrs()` adds implicit `__host__ __device__` to functions that qualify under CUDA's implicit rules: lambdas defined in device context, and certain implicit special members (constructors, destructors, copy/move operators) where all bases and fields are themselves `__host__ __device__`. The method `inferTargetForImplicitSpecialMember()` recursively infers the target for implicitly-defined specials, diagnosing conflicts where a required base constructor is device-only but the synthesized special member is being used from host.

---

## 48.4 CodeGen for CUDA

### 48.4.1 CGCUDARuntime and CGNVCUDARuntime

The CodeGen layer abstracts GPU runtime interactions behind `CGCUDARuntime`, a pure virtual class in [`clang/lib/CodeGen/CGCUDARuntime.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGCUDARuntime.h). The concrete NVIDIA implementation is `CGNVCUDARuntime` in [`clang/lib/CodeGen/CGCUDANV.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGCUDANV.cpp), instantiated when the target triple is `nvptx64-nvidia-cuda` or when the host pass compiles with CUDA enabled.

The key responsibilities of `CGNVCUDARuntime` are:

1. **Device stub synthesis** — For each `__global__` function, generate a host-side wrapper
2. **Module registration** — Emit the `__cuda_register_globals()` constructor
3. **Fat binary embedding** — Wire the `.nv_fatbin` section into the registration path

### 48.4.2 Device Stub Functions

For every `__global__` function `foo()`, `CGNVCUDARuntime` synthesizes a host-side stub named `__device_stub__foo()` with an identical signature. When the user writes `foo<<<grid, block>>>(args...)`, the `CUDAKernelCallExpr` lowers to a call to this stub. Verified with actual Clang 22 output:

```llvm
; Host-side IR for: __global__ void saxpy(float* a, float* b, float* c, float alpha, int n)
define dso_local void @_Z20__device_stub__saxpyPfS_S_fi(
    ptr noundef %0, ptr noundef %1, ptr noundef %2,
    float noundef %3, i32 noundef %4) {
  ; Build argv array: pointers to each argument
  %17 = alloca ptr, i64 5, align 16
  ; ... store &arg_i into argv[i] ...

  ; Pop grid/block configuration pushed by <<<>>>
  %23 = call i32 @__cudaPopCallConfiguration(ptr %11, ptr %12, ptr %13, ptr %14)

  ; Pack dim3 structs into two i64+i32 pairs for the runtime ABI
  %27 = load i64, ptr %26, align 8   ; gridDim packed
  %29 = load i32, ptr %28, align 8
  %31 = load i64, ptr %30, align 8   ; blockDim packed
  %33 = load i32, ptr %32, align 8

  ; Launch the kernel
  call i32 @cudaLaunchKernel(
      ptr @_Z20__device_stub__saxpyPfS_S_fi,
      i64 %27, i32 %29,   ; gridDim
      i64 %31, i32 %33,   ; blockDim
      ptr %17,             ; argv array
      i64 %24,             ; sharedMem bytes
      ptr %25)             ; stream
  ret void
}
```

The stub passes its own address as the `func` argument to `cudaLaunchKernel`. The CUDA runtime uses this address as a key into the registered-kernel table built at program startup.

### 48.4.3 Module Registration

`CGNVCUDARuntime` emits a module-level constructor (priority 101, before user constructors) that performs the following sequence:

```cpp
// Pseudocode for emitted __cuda_register_globals()
void __cuda_module_ctor() {
    void** fatbinHandle = __cudaRegisterFatBinary(&__cuda_fatbin_wrapper);
    __cudaRegisterFunction(fatbinHandle,
        (const char*)__device_stub__saxpy,  // host func address
        "_Z5saxpyPfS_S_fi",                 // device func mangled name
        "_Z5saxpyPfS_S_fi", -1, 0, 0, 0, 0, NULL);
    // ... repeat for each __global__ ...
    __cudaRegisterFatBinaryEnd(fatbinHandle);
    atexit(__cuda_module_dtor);
}
```

`__cudaRegisterFatBinary` receives a `__fatBinC_Wrapper_t` struct that includes the magic number `0x466243B1`, the version, and a pointer to the embedded fat binary blob. When `CudaFeatureEnabled(Version, CUDA_USES_FATBIN_REGISTER_END)` is true (CUDA 10.1+), Clang emits `__cudaRegisterFatBinaryEnd` after registering all functions and variables. This call is guarded so that the old API is used on older toolkits.

---

## 48.5 Device IR Generation

### 48.5.1 NVPTX Target Triple and Data Layout

Device compilation targets the triple `nvptx64-nvidia-cuda`. The target data layout set by the NVPTX backend is:

```
e-p6:32:32-i64:64-i128:128-i256:256-v16:16-v32:32-n16:32:64
```

Key points: `p6:32:32` makes generic address space 6 pointers 32-bit (the PTX generic pointer is narrower than the 64-bit global pointer in some contexts, though this layout is for specific sub-uses); the `n16:32:64` indicates native integer widths. Global memory pointers are 64-bit, matching the `nvptx64` convention.

### 48.5.2 Thread and Block Index Intrinsics

The CUDA built-in variables (`threadIdx`, `blockIdx`, `blockDim`, `gridDim`) are defined in `__clang_cuda_builtin_vars.h` using `__declspec(property)`. Each field access (`threadIdx.x`) generates a call to a corresponding NVVM intrinsic:

| CUDA Built-in | NVVM Intrinsic | PTX Register |
|---------------|----------------|--------------|
| `threadIdx.x` | `llvm.nvvm.read.ptx.sreg.tid.x` | `%tid.x` |
| `threadIdx.y` | `llvm.nvvm.read.ptx.sreg.tid.y` | `%tid.y` |
| `threadIdx.z` | `llvm.nvvm.read.ptx.sreg.tid.z` | `%tid.z` |
| `blockIdx.x` | `llvm.nvvm.read.ptx.sreg.ctaid.x` | `%ctaid.x` |
| `blockIdx.y` | `llvm.nvvm.read.ptx.sreg.ctaid.y` | `%ctaid.y` |
| `blockIdx.z` | `llvm.nvvm.read.ptx.sreg.ctaid.z` | `%ctaid.z` |
| `blockDim.x` | `llvm.nvvm.read.ptx.sreg.ntid.x` | `%ntid.x` |
| `gridDim.x` | `llvm.nvvm.read.ptx.sreg.nctaid.x` | `%nctaid.x` |
| `warpSize` | `llvm.nvvm.read.ptx.sreg.warpsize` | `%warpsize` |

These intrinsics carry range attributes that LLVM's value tracking exploits. For example:
```llvm
declare noundef range(i32 0, 1024) i32 @llvm.nvvm.read.ptx.sreg.tid.x()
declare noundef range(i32 1, 1025) i32 @llvm.nvvm.read.ptx.sreg.ntid.x()
declare noundef range(i32 0, 2147483647) i32 @llvm.nvvm.read.ptx.sreg.ctaid.x()
```
The range metadata enables dead-code elimination (e.g., removing code guarded by `if (threadIdx.x < 0)`) and induction variable analysis for loop vectorization.

### 48.5.3 Verified Device IR: SAXPY Kernel

Compiling the SAXPY kernel with `clang++ -x cuda --cuda-device-only -emit-llvm -S --cuda-gpu-arch=sm_75` produces:

```llvm
; target triple = "nvptx64-nvidia-cuda"

define dso_local ptx_kernel void @_Z5saxpyPfS_S_fi(
    ptr noundef %0, ptr noundef %1, ptr noundef %2,
    float noundef %3, i32 noundef %4) #0 {
  ; ...alloca for parameters...
  %12 = call noundef i32 @llvm.nvvm.read.ptx.sreg.ctaid.x()  ; blockIdx.x
  %13 = call noundef i32 @llvm.nvvm.read.ptx.sreg.ntid.x()   ; blockDim.x
  %14 = mul i32 %12, %13
  %15 = call noundef i32 @llvm.nvvm.read.ptx.sreg.tid.x()    ; threadIdx.x
  %16 = add i32 %14, %15                                      ; global thread id
  ; ...bounds check, load, fmul, fadd, store...
  ret void
}

attributes #0 = { convergent mustprogress noinline norecurse nounwind optnone
  "target-cpu"="sm_75" "target-features"="+ptx88,+sm_75"
  "uniform-work-group-size"="true" }
```

The `ptx_kernel` calling convention is the device-side analog of `x86_64` calling conventions — it signals the NVPTX backend to emit a `.visible .entry` PTX directive rather than a `.func`.

The `convergent` attribute marks functions that must not be reordered or duplicated across divergent control flow boundaries, critical for correctness of warp-synchronous operations.

### 48.5.4 Shared Memory

`__shared__` variables compile to LLVM IR globals in address space 3. Function-local `__shared__` arrays are promoted to implicit static globals:

```llvm
; __shared__ float sdata[256];
@_ZZ6reducePfS_iE5sdata = internal addrspace(3) global [256 x float] undef, align 4
```

Access from generic code requires an `addrspacecast`:
```llvm
%ptr = addrspacecast ptr addrspace(3) @_ZZ6reducePfS_iE5sdata to ptr
%elem = getelementptr inbounds [256 x float], ptr %ptr, i64 0, i64 %tid
```

### 48.5.5 Constant Memory

`__constant__` variables land in address space 4:

```llvm
; __constant__ float c_coeff[16];
@c_coeff = dso_local addrspace(4) externally_initialized constant [16 x float]
           zeroinitializer, align 4
```

In PTX, this becomes a `.const` segment variable. The CUDA runtime initializes it via `cudaMemcpyToSymbol`. Access from device code goes through the constant cache, a read-only 64 KB per-SM cache with broadcast capability — all threads in a warp reading the same address hit the cache with a single fetch.

### 48.5.6 Barrier and Synchronization

`__syncthreads()` lowers to `llvm.nvvm.barrier.cta.sync.aligned.all(i32 0)` in current Clang 22:

```llvm
call void @llvm.nvvm.barrier.cta.sync.aligned.all(i32 0)
```

This maps to the PTX `bar.sync 0` or `barrier.sync 0` instruction, synchronizing all threads in the CTA (cooperative thread array, i.e., the block) and issuing a memory fence.

---

## 48.6 NVPTX Backend and PTX Generation

### 48.6.1 NVPTXTargetMachine

The NVPTX backend's top-level class is `NVPTXTargetMachine` in `llvm/lib/Target/NVPTX/NVPTXTargetMachine.cpp`. It supports two sub-targets: `nvptx` (32-bit pointers) and `nvptx64` (64-bit pointers). Clang always selects `nvptx64` for CUDA targets. The `NVPTXTargetMachine` constructor registers the target-specific pass pipeline through `NVPTXPassConfig`.

The SM architecture and PTX version are specified via `-target-cpu sm_75 -target-feature +ptx88`. As of LLVM 22, PTX 8.8 (corresponding to CUDA 12.9) is the maximum version; Clang selects the appropriate PTX ISA version based on the detected CUDA SDK.

### 48.6.2 Key NVPTX Passes

The NVPTX backend inserts several machine-independent IR transformation passes before instruction selection:

| Pass | Purpose |
|------|---------|
| `NVPTXLowerAggrCopies` | Expand `memcpy`/`memmove`/`memset` on aggregates to scalar loads/stores |
| `NVPTXLowerAlloca` | Convert `alloca`s to local memory (`.local` PTX segment) |
| `NVPTXAssignValidGlobalNames` | Sanitize mangled names to legal PTX identifiers |
| `NVPTXFavorNonGenericAddrSpaces` | Promote generic pointer operations to specific address spaces where provable |
| `InferAddressSpaces` | LLVM-generic pass, aggressive for NVPTX: infers that global memory accesses don't need generic-to-specific casts |
| `NVPTXTagInvariantLoad` | Mark loads from constant memory with `!invariant.load` for backend optimization |

### 48.6.3 PTX Output Structure

The NVPTX backend emits PTX assembly. For the SAXPY example:

```ptx
.version 8.8
.target sm_75
.address_size 64

.visible .entry _Z5saxpyPfS_S_fi(
    .param .u64 .ptr .align 1 _Z5saxpyPfS_S_fi_param_0,
    .param .u64 .ptr .align 1 _Z5saxpyPfS_S_fi_param_1,
    .param .u64 .ptr .align 1 _Z5saxpyPfS_S_fi_param_2,
    .param .f32 _Z5saxpyPfS_S_fi_param_3,
    .param .u32 _Z5saxpyPfS_S_fi_param_4
) {
    .reg .pred %p<2>;
    .reg .b32  %r<14>;
    .reg .b64  %rd<12>;
    .reg .f32  %f<4>;

    ld.param.b64  %rd1, [_Z5saxpyPfS_S_fi_param_0];
    mov.u32  %r3, %ctaid.x;
    mov.u32  %r4, %ntid.x;
    mul.lo.s32  %r5, %r3, %r4;
    mov.u32  %r6, %tid.x;
    add.s32  %r7, %r5, %r6;
    setp.ge.s32  %p1, %r7, [param4];
    @%p1 bra $L__BB0_2;
    // ... float arithmetic ...
$L__BB0_2:
    ret;
}
```

Key PTX sections:
- `.entry` — kernel entry point (corresponds to `__global__`)
- `.func` — device function (corresponds to `__device__`)
- `.param` — kernel parameter space; parameters passed by value via `.param` loads
- `.reg` — virtual register declarations (PTX is in SSA form)
- `.shared` — shared memory declarations (address space 3)
- `.const` — constant memory (address space 4)
- `.local` — per-thread local memory (address space 5)

### 48.6.4 ptxas and SASS

`ptxas` (the NVIDIA PTX assembler) compiles PTX to SASS (Streaming Assembler), the actual GPU instruction encoding. Clang invokes `ptxas` automatically when a full build is requested. Relevant flags:

```bash
ptxas -m64 -O3 --gpu-name sm_90 --output-file kernel.cubin kernel.ptx
```

`-Xcuda-ptxas --maxrregcount=N` passes through to `ptxas` to cap register usage, trading register spills for higher occupancy. During `--cuda-noopt-device-debug`, Clang passes `-O0` to `ptxas`, preserving full debug info at the cost of occupancy.

---

## 48.7 Fat Binary Creation

### 48.7.1 The Fat Binary Format

A CUDA fat binary bundles device code for multiple GPU architectures into a single binary blob. NVIDIA's `fatbinary` tool wraps one or more cubin or PTX images into a fat binary with the magic header `0x466243B1`. The standard Clang pipeline uses NVIDIA's `fatbinary` tool directly:

```bash
fatbinary -64 --create foo.fatbin \
    --image3=kind=elf,sm=80,file=foo-sm_80.o \
    --image3=kind=elf,sm=90,file=foo-sm_90.o \
    --image=profile=compute_90,file=foo-sm_90.ptx  # PTX for JIT
```

Including PTX images allows the CUDA runtime to JIT-compile for future architectures not present as cubins.

### 48.7.2 Embedding in the Host Object

The host compilation pass (`-fcuda-include-gpubinary foo.fatbin`) causes `CGNVCUDARuntime` to embed the fat binary blob directly into the host object file. The blob is placed in the `.nv_fatbin` ELF section, annotated as `"aw","@progbits"` so the linker includes it in the data segment. A `__fatBinC_Wrapper_t` struct at a known symbol name (`__cuda_fatbin_wrapper`) points to this section:

```c
struct __fatBinC_Wrapper_t {
  int  magic;     // 0x466243B1
  int  version;   // 1
  const void *data;   // pointer to .nv_fatbin section
  void *filename_or_fatbins;
};
```

This struct is passed to `__cudaRegisterFatBinary()` in the module constructor. The CUDA runtime driver (`libcuda.so`) extracts and loads the appropriate cubin for the current GPU.

### 48.7.3 Relocatable Device Code and the New Offload Model

With `-fgpu-rdc` (relocatable device code), device code can call functions defined in other translation units, analogous to host-side separate compilation. The pipeline changes:

1. `ptxas` is invoked with `-c` to produce relocatable device object files
2. `llvm-offload-binary` wraps the device object into an offload binary section
3. The host object embeds this via `-fembed-offload-object`
4. `clang-linker-wrapper` extracts device objects from all host `.o` files and invokes `clang-nvlink-wrapper` → `nvlink` for device-side linking
5. The linked device binary is re-embedded into the final host executable

As of Clang 22, the default pipeline uses `--offload-new-driver`, which routes fat binary handling through `clang-linker-wrapper` rather than the legacy `fatbinary` + embedded-section approach. The `llvm-offload-binary` tool packs device images with their metadata (triple, arch, kind) into a container format that `clang-linker-wrapper` understands.

---

## 48.8 CUDA Intrinsics and Builtins

### 48.8.1 Warp Shuffle Intrinsics

Warp shuffle operations move values between lanes without going through shared memory. `__shfl_down_sync(mask, val, delta, width)` in `__clang_cuda_intrinsics.h` calls `llvm.nvvm.shfl.sync.down.f32` (for float) or `llvm.nvvm.shfl.sync.down.i32`:

```llvm
; __shfl_down_sync(0xffffffff, val, offset, 32)
%result = call contract float @llvm.nvvm.shfl.sync.down.f32(
    i32 -1,          ; mask (0xffffffff)
    float %val,      ; value to shuffle
    i32 %offset,     ; delta
    i32 31)          ; clamp = (32-1) | (0 << 8) = 31
```

The full set of shuffle intrinsics: `llvm.nvvm.shfl.sync.{up,down,bfly,idx}.{i32,f32}`.

### 48.8.2 Vote and Ballot Intrinsics

Warp voting operations query boolean predicates across all active lanes:

| CUDA Function | NVVM Intrinsic |
|---------------|----------------|
| `__ballot_sync(mask, pred)` | `llvm.nvvm.vote.ballot.sync` |
| `__all_sync(mask, pred)` | `llvm.nvvm.vote.all.sync` |
| `__any_sync(mask, pred)` | `llvm.nvvm.vote.any.sync` |
| `__activemask()` | `llvm.nvvm.activemask` |

### 48.8.3 Tensor Core (WMMA) Intrinsics

Tensor core operations via `nvcuda::wmma` header calls compile to `llvm.nvvm.wmma.*` intrinsics. For example, `wmma::mma_sync` targeting `__half` 16x16x16 matrices on sm_80+:

```llvm
; wmma::mma_sync: D = A * B + C (fp16 inputs, fp32 accumulator)
call void @llvm.nvvm.wmma.m16n16k16.mma.col.col.f32.f16(
    ptr %d_frag, ptr %a_frag, ptr %b_frag, ptr %c_frag)
```

These intrinsics are lowered by the NVPTX backend to `wmma.*` PTX instructions introduced in PTX 6.0 for sm_70+.

### 48.8.4 Read-Only Cache: __ldg

`__ldg(ptr)` (load via read-only texture/L1 cache) compiles to `llvm.nvvm.ldg.global.*` intrinsics:

```llvm
; float __ldg(const float* ptr)
%val = call float @llvm.nvvm.ldg.global.f.f32.p1(ptr addrspace(1) %ptr, i32 4)
```

In PTX, this becomes `ld.global.nc.f32` using the non-coherent (NC) cache path.

### 48.8.5 Half and BFloat16 Types

`__half` (16-bit IEEE 754 float) and `__nv_bfloat16` (Brain Float) are defined in CUDA headers as struct types wrapping a 16-bit field. Clang 22 recognizes `_Float16` natively and maps CUDA's `__half` arithmetic operations to LLVM `half` type operations or `llvm.nvvm.fma.rn.f16` / `llvm.nvvm.fma.rn.bf16` intrinsics on architectures supporting native fp16/bf16 execution (sm_53+ for fp16, sm_80+ for bf16).

---

## 48.9 Unified Memory and Address Spaces

### 48.9.1 NVPTX Address Space Numbering

LLVM IR targeting NVPTX uses the following address space numbers, matching `nvptxintrin.h`:

| Address Space | PTX Segment | Purpose |
|---------------|-------------|---------|
| 0 | `.generic` | Generic (flat) pointer; can refer to any |
| 1 | `.global` | GPU global memory (device DRAM) |
| 3 | `.shared` | Per-block shared memory (SRAM) |
| 4 | `.const` | Read-only constant memory |
| 5 | `.local` | Per-thread local (register spill) memory |

These correspond to `LangAS::cuda_device` (→ 1), `LangAS::cuda_shared` (→ 3), and `LangAS::cuda_constant` (→ 4) in the Clang frontend. The CodeGen layer in `CGCUDARuntime` calls `getContext().getTargetAddressSpace(LangAS::cuda_shared)` to get 3 at code generation time.

Generic pointers (address space 0) are the default; the NVPTX backend inserts `cvta.to.global`, `cvta.to.shared` PTX instructions when converting between specific and generic pointers, or `isspacep` checks for runtime predication. The `NVPTXFavorNonGenericAddrSpaces` and `InferAddressSpaces` passes aggressively eliminate unnecessary generic pointer operations to reduce instruction overhead.

### 48.9.2 __managed__ Variables

`__managed__` variables are allocated via `cudaMallocManaged()` at runtime, accessible from both host and device with CUDA's Unified Memory (UVM) subsystem migrating pages on demand. In the IR, they are declared in global address space 1 but marked `externally_initialized` because their address is set at runtime. `SemaCUDA::IdentifyTarget(VarDecl*)` returns `CVT_Unified` for these variables.

### 48.9.3 __restrict__ and noalias

`__restrict__` on device function pointer parameters lowers to LLVM `noalias` parameter attributes, identically to C99 restrict. The NVPTX backend can exploit `noalias` for more aggressive load/store reordering within the memory consistency model, and for proving that texture/LDG accesses do not alias writable global memory.

---

## 48.10 Debugging and Profiling Support

### 48.10.1 Host Debug Info

`-g` on a CUDA compilation produces DWARF for the host code path via the standard Clang debug info generation in `CGDebugInfo`. Device code compiled without `-G` is optimized and produces no device DWARF.

### 48.10.2 Device Debug Info: -G

`-G` (capital) enables full device-side debug information. Internally, this passes `-g` to the device `cc1` invocation and `-O0` to `ptxas` (via `--cuda-noopt-device-debug`). The NVPTX backend emits DWARF sections targeting the device code, and `ptxas` preserves them in the cubin. `cuda-gdb` (NVIDIA's debugger) reads these sections to provide source-level debugging of GPU kernels.

Device DWARF uses the standard DWARF 4 format but with NVPTX-specific location expressions for GPU registers and memory spaces. Clang 22 correctly tags address space 3 (shared) and address space 4 (constant) in DW_AT_address_class attributes.

### 48.10.3 Lightweight Line Info

`--cuda-noopt-device-debug` without `-G` (or equivalently, passing `-Xcuda-ptxas --generate-line-info`) inserts PTX `.loc` directives but disables full DWARF generation. This allows NVIDIA's profiling tools (Nsight Compute, Nsight Systems) to map performance counters back to source lines with minimal binary bloat.

### 48.10.4 Register Count Control

`-Xcuda-ptxas --maxrregcount=N` is forwarded directly to `ptxas`, capping the per-thread register usage at `N` registers. Lower register counts increase occupancy (more concurrent warps per SM) at the cost of more register spills to local memory. Clang also accepts `__launch_bounds__(maxTPB, minBlocks)` which gives `ptxas` the occupancy target to drive its register allocator.

---

## 48.11 CUDA in CMake and Build Systems

### 48.11.1 CMake CUDA Language Support

CMake 3.18+ supports CUDA as a first-class language. With Clang as the CUDA compiler:

```cmake
cmake_minimum_required(VERSION 3.18)
project(MyCUDA LANGUAGES CXX CUDA)

# Tell CMake to use Clang for CUDA
set(CMAKE_CUDA_COMPILER /usr/lib/llvm-22/bin/clang++)
set(CMAKE_CUDA_ARCHITECTURES "75;80;90")

add_executable(myapp main.cpp kernel.cu)

# Pass Clang-specific options to CUDA compilation
target_compile_options(myapp PRIVATE
    $<$<COMPILE_LANGUAGE:CUDA>:--cuda-gpu-arch=sm_90>
    $<$<COMPILE_LANGUAGE:CUDA>:-fgpu-rdc>)
```

`CMAKE_CUDA_ARCHITECTURES` is a semicolon-separated list of SM numbers. CMake translates each to `--cuda-gpu-arch=sm_NN` for Clang (versus `-arch sm_NN` for nvcc).

### 48.11.2 Mixed-Language Files and Properties

To compile a `.cpp` file containing CUDA syntax (unusual but valid):
```cmake
set_source_files_properties(heterogeneous.cpp PROPERTIES LANGUAGE CUDA)
```

For per-file architecture targeting:
```cmake
set_source_files_properties(sm90_only.cu PROPERTIES
    COMPILE_OPTIONS "--cuda-gpu-arch=sm_90;--cuda-gpu-arch=sm_90a")
```

### 48.11.3 Separate Device Linking

With `-fgpu-rdc`, device linking must occur explicitly. In CMake 3.25+:
```cmake
set_target_properties(myapp PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
```
CMake invokes `clang-linker-wrapper` (which drives `clang-nvlink-wrapper` → `nvlink`) for the device link step. The device link produces a single device image that is then embedded in the final host executable.

---

## 48.12 Differences from nvcc

### 48.12.1 C++ Standard Compliance

Clang parses CUDA code as standard C++23 on both host and device sides, with CUDA extensions layered on top. `nvcc` passes host code through a separate host compiler (by default `g++`), which means `nvcc` can be more lenient with non-standard host-side extensions (GCC builtins, statement-expressions, etc.) while device code is compiled by `nvcc`'s own front-end with a more restricted C++ subset.

Clang's stricter parsing catches:
- Template instantiation across device boundaries that `nvcc` silently accepts
- Invalid `__host__` / `__device__` combinations in overload sets
- Use of `constexpr` virtual functions on device (disallowed by the CUDA spec)
- Standard conformance issues in device lambda captures

### 48.12.2 Template Instantiation

Clang instantiates templates for both host and device contexts simultaneously from the same AST. This means that a templated `__host__ __device__` function is correctly verified for both execution environments in a single compilation. `nvcc` defers device instantiation, which can miss cross-side errors until link time.

A common portability issue: Clang rejects `__noinline__` (an nvcc extension) and requires `__attribute__((noinline))` or `[[clang::noinline]]`. Similarly, `__forceinline__` works in both, but Clang also accepts `[[clang::always_inline]]`.

### 48.12.3 libdevice.bc Linking

Both Clang and nvcc link against NVIDIA's `libdevice.10.bc` — a bitcode library providing device-side implementations of math functions (`sinf`, `cosf`, `expf`, etc.). Clang passes `-mlink-builtin-bitcode /path/to/libdevice.10.bc` to `cc1`, causing the bitcode to be linked during device compilation via LLVM's `BitcodeLinker`. After linking, the standard optimization pipeline runs over the combined module, enabling inlining of libdevice functions into kernels.

Functions in `libdevice.bc` use the `nvvm-reflect` mechanism to query compile-time architecture features: `llvm.nvvm.reflect("__CUDA_ARCH")` returns the SM version as an integer, enabling conditional code specialization within the bitcode without separate compilation.

### 48.12.4 Exception Handling

Device code cannot use C++ exceptions. Clang enforces `-fno-exceptions` implicitly on the device pass. Any attempt to use `try`/`catch`/`throw` in `__device__` code produces an error. `nvcc` historically emitted device code with exceptions silently disabled without error; Clang's explicit rejection is more correct.

### 48.12.5 Summary Comparison

| Feature | Clang | nvcc |
|---------|-------|------|
| Host parser | Clang (full C++23) | Host compiler subprocess (g++/cl) |
| Template instantiation | Unified, single pass | Deferred device instantiation |
| Standard enforcement | Strict | Lenient for host side |
| `__noinline__` | Not recognized | Supported |
| `__forceinline__` | Supported | Supported |
| Deferred diagnostics | Yes, precise | Sometimes missing |
| LTO with host | Yes (via LLVM LTO) | Limited |
| Integration with sanitizers | Yes (ASan, UBSan on host) | No |
| Debug info quality | High (DWARF 4/5) | Good (DWARF 4) |
| Fat binary format | Uses NVIDIA fatbinary tool | Same |
| Separable compilation | `-fgpu-rdc` | `-dc` / `-rdc=true` |

---

## Chapter Summary

- Clang compiles `.cu` files with a dual-pass model: one `cc1` invocation for `nvptx64-nvidia-cuda` (device) and one for the host triple, orchestrated by the driver's CUDA-aware action builder.
- `CudaInstallationDetector` locates the CUDA toolkit; `CudaVersion` and `OffloadArch` enums encode the supported version matrix from CUDA 7.0 through 13.0+.
- CUDA function qualifiers (`__global__`, `__device__`, `__host__`) become `CUDAGlobalAttr`, `CUDADeviceAttr`, `CUDAHostAttr` on `FunctionDecl`; variable qualifiers become `CUDASharedAttr`, `CUDAConstantAttr`, mapping to address spaces in `LangAS`.
- `SemaCUDA::IdentifyTarget()` classifies functions into `CUDAFunctionTarget::{Device, Global, Host, HostDevice}`; `SemaCUDA::CheckCall()` enforces execution-space legality using the `CUDAFunctionPreference` ordering, with deferred diagnostics for `__host__ __device__` code.
- `CGNVCUDARuntime` (in `CGCUDANV.cpp`) generates device stubs for each `__global__`, emitting `cudaLaunchKernel()` calls; the module constructor registers kernels via `__cudaRegisterFunction` and fat binaries via `__cudaRegisterFatBinary`.
- Device IR targets `nvptx64-nvidia-cuda` with address spaces 1/3/4/5 for global/shared/constant/local memory; thread index built-ins lower to `llvm.nvvm.read.ptx.sreg.*` intrinsics with range metadata; `__syncthreads()` becomes `llvm.nvvm.barrier.cta.sync.aligned.all`.
- The NVPTX backend emits PTX ISA via `NVPTXTargetMachine`; key passes include `NVPTXLowerAlloca`, `NVPTXFavorNonGenericAddrSpaces`, and `InferAddressSpaces`; `ptxas` assembles PTX to SASS.
- Fat binaries embed multi-arch cubins (and optionally PTX for JIT) via NVIDIA's `fatbinary` tool; the `.nv_fatbin` section is embedded into host objects and registered at startup.
- With `-fgpu-rdc`, `clang-nvlink-wrapper` handles device-side linking across translation units; `clang-linker-wrapper` coordinates the host and device link steps.
- Warp intrinsics (`__shfl_*_sync`, `__ballot_sync`) map to `llvm.nvvm.shfl.*` and `llvm.nvvm.vote.*`; tensor core operations map to `llvm.nvvm.wmma.*`.
- Clang enforces stricter C++ compliance than nvcc on both host and device sides, rejecting nvcc-specific extensions and providing more precise template instantiation diagnostics.

---

*Cross-references:*
- [Chapter 28 — The Clang Driver](../part-05-clang-frontend/ch28-clang-driver.md) — Driver architecture, action construction, and the offload toolchain framework
- [Chapter 39 — CodeGenModule and CodeGenFunction](../part-06-clang-codegen/ch39-codegenmodule-codegenfunction.md) — Base CodeGen infrastructure that CGNVCUDARuntime extends
- [Chapter 40 — Lowering Statements and Expressions](../part-06-clang-codegen/ch40-lowering-statements-expressions.md) — Expression lowering for CUDAKernelCallExpr
- [Chapter 41 — Calls, the ABI Boundary, and Builtins](../part-06-clang-codegen/ch41-calls-abi-builtins.md) — Calling conventions and builtin lowering relevant to device stubs
- [Chapter 49 — Clang as a HIP Compiler](ch49-clang-as-hip-compiler.md) — AMD GPU compilation using the same dual-pass infrastructure
- [Chapter 102 — The NVPTX Backend](../part-15-targets/ch102-nvptx-backend.md) — Deep dive into NVPTXTargetMachine, instruction selection, and PTX optimization

*Reference links:*
- [`clang/include/clang/Basic/Cuda.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/Cuda.h) — CudaVersion, CUDAFunctionTarget enums
- [`clang/include/clang/Basic/OffloadArch.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/OffloadArch.h) — OffloadArch enum (SM_20 through SM_121a)
- [`clang/include/clang/Driver/CudaInstallationDetector.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/CudaInstallationDetector.h) — Toolkit detection
- [`clang/include/clang/Sema/SemaCUDA.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/SemaCUDA.h) — SemaCUDA class, CUDAFunctionPreference
- [`clang/include/clang/AST/ExprCXX.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/ExprCXX.h) — CUDAKernelCallExpr
- [`clang/include/clang/Basic/AddressSpaces.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/AddressSpaces.h) — LangAS::cuda_device/constant/shared
- [`clang/lib/CodeGen/CGCUDANV.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGCUDANV.cpp) — CGNVCUDARuntime implementation
- [`llvm/lib/Target/NVPTX/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/NVPTX/) — NVPTX backend source tree
- [`clang/lib/Driver/ToolChains/Cuda.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Driver/ToolChains/Cuda.cpp) — CUDA toolchain driver logic
- [`llvm/lib/Transforms/Scalar/InferAddressSpaces.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Scalar/InferAddressSpaces.cpp) — Address space inference pass
