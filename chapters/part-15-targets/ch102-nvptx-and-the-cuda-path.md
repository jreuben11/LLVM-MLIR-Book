# Chapter 102 — NVPTX and the CUDA Path

*Part XV — Targets*

NVIDIA GPU programming demands a compiler backend that speaks PTX — the Parallel Thread Execution virtual ISA that NVIDIA's driver compiles to device-specific SASS machine code at load time. LLVM's NVPTX backend (`llvm/lib/Target/NVPTX/`) is a production-quality implementation that serves as the downstream compilation target for Clang's CUDA and SYCL offload pipelines, OpenCL via POCL, and a range of research compilers. Understanding the NVPTX backend means understanding how high-level GPU programming models are grounded in hardware reality: how address spaces become PTX qualifiers, how warps and barriers turn into synchronization instructions, and how kernel entry points acquire the metadata annotations that tell the CUDA runtime how to dispatch work. This chapter covers every layer of that translation, from the NVPTX target machine configuration down to the binary PTX text that `ptxas` consumes.

---

## 102.1 NVPTX Architecture Overview

### 102.1.1 CUDA Thread Hierarchy

Before examining the backend mechanics, the CUDA thread hierarchy is the logical model that every NVPTX IR construct maps onto:

```
Grid (a kernel launch)
  └── Block [0,0] ... Block [Gx-1, Gy-1, Gz-1]    ← blockIdx
        └── Warp 0 ... Warp (blockDim/32 - 1)
              └── Thread 0 ... Thread 31             ← threadIdx (divergence unit)
```

- **Grid**: the entire launched kernel invocation; up to 2^31-1 blocks per dimension
- **Block (CTA — Cooperative Thread Array)**: up to 1024 threads; shares shared memory (`addrspace(3)`) and can synchronize with `bar.sync`
- **Warp**: 32 threads executing in SIMT lockstep; the unit of divergence and scheduling
- **Thread**: a single lane; `threadIdx.x/y/z` identifies it within the block

The compiler must map this hierarchy into PTX special registers (`%tid.x`, `%ntid.x`, `%ctaid.x`, `%nctaid.x`) and reason about which values are *uniform* (same across a warp) versus *divergent* (thread-specific).

### 102.1.2 PTX as a Virtual ISA and Compilation Pipeline

PTX (Parallel Thread Execution) is deliberately not a physical machine-code format. Like LLVM IR itself, it is a stable virtual ISA with a versioned specification — PTX ISA 8.x as of CUDA 12.x — that NVIDIA can retarget to successive GPU microarchitectures without breaking binary compatibility. A PTX binary ships inside a `.cubin` or fat binary container and is JIT-compiled by the CUDA driver to SASS (Shader ASSembly), the actual GPU instruction encoding for a specific SM version.

This two-layer model creates an important separation of concerns for LLVM's NVPTX backend: LLVM is responsible for generating correct, well-optimized PTX; NVIDIA's closed-source `ptxas` assembler handles the final ISA scheduling, register allocation within hardware register file constraints, and SM-specific code generation. LLVM never touches SASS.

The full two-step pipeline is:

```
CUDA C++ source
     │
     ▼ clang -x cuda --offload-arch=sm_90a
LLVM IR (nvptx64-nvidia-cuda, addrspace annotations, nvvm metadata)
     │
     ▼ NVPTX backend (llc -march=nvptx64 -mcpu=sm_90a)
PTX text (.ptx)
     │
     ▼ ptxas --gpu-name sm_90a
Cubin (.cubin, SASS machine code)
     │
     ▼ CUDA driver JIT or fatbinary embed
Executable / .fatbin
```

### 102.1.3 PTX ISM Versions and Compute Capabilities

PTX ISM (Instruction Set Model) versions gate which instructions are available:

| PTX Version | CUDA Version | Key additions |
|-------------|-------------|---------------|
| 6.0         | CUDA 9.0    | `wmma.*` tensor core |
| 7.0         | CUDA 11.0   | Asynchronous copies (`cp.async`) |
| 7.4         | CUDA 11.4   | `wgmma.*` (Hopper preview) |
| 8.0         | CUDA 12.0   | Hopper wgmma, TMA, bar.arrive |
| 8.5         | CUDA 12.5   | Blackwell instructions |

SM (Streaming Multiprocessor) versions (`sm_XX`) map to physical generations:
- `sm_70/75`: Volta/Turing — tensor cores, independent thread scheduling
- `sm_80/86/87/89`: Ampere — BFloat16 tensor core, async memory
- `sm_90/90a`: Hopper — wgmma, TMA, cluster-level synchronization
- `sm_100`: Blackwell

LLVM emits the `.version` and `.target` directives based on the `-mcpu` flag passed to `llc` or the `--offload-arch` flag passed to Clang:

```bash
# Emit PTX targeting sm_89 (Ada Lovelace), PTX ISM 7.8
/usr/lib/llvm-22/bin/llc -march=nvptx64 -mcpu=sm_89 kernel.ll -o kernel.ptx
# Output begins:
# .version 7.8
# .target sm_89
# .address_size 64
```

### 102.1.4 The NVPTX Target Triple

The target triple for NVPTX is `nvptx64-nvidia-cuda` (64-bit) or `nvptx-nvidia-cuda` (32-bit, legacy). Clang also uses `nvptx64-nvidia-nvcl` for the OpenCL path. The triple's architecture field drives backend selection; the `nvidia-cuda` OS/environment portion carries no runtime ABI meaning but is recognized by the CUDA toolchain.

The data layout for 64-bit NVPTX is:

```
e-i64:64-i128:128-v16:16-v32:32-n16:32:64
```

This expresses: little-endian, 64-bit i64 alignment, 128-bit i128 alignment, 16-bit vector alignment, 32-bit vector alignment, native integer widths 16/32/64. There is no natural pointer alignment for global memory — PTX treats pointer arithmetic as 64-bit integer operations rather than enforcing alignment.

### 102.1.5 NVPTX TableGen Instruction Definitions

NVPTX instructions are defined in TableGen with a pattern system that maps LLVM DAG nodes to PTX emission strings. The key file is [`NVPTXInstrInfo.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/NVPTX/NVPTXInstrInfo.td). A representative excerpt showing how a global load is defined:

```tablegen
// In NVPTXInstrInfo.td:
// Generic load DAG node pattern → PTX ld.global.b32
def LD_i32_avar : NVPTXInst<(outs Int32Regs:$dst),
                              (ins Int32Regs:$src),
                              "ld.global.b32 \t$dst, [$src];",
                              [(set Int32Regs:$dst,
                                (int_nvvm_ld_gen_i32 Int32Regs:$src))]>;

// Atomic add for float in global memory:
def INT_PTX_ATOM_ADD_G_F32 : NVPTXInst<(outs Float32Regs:$result),
    (ins Float32Regs:$src, Float32Regs:$val),
    "atom.global.add.f32 \t$result, [$src], $val;",
    [(set Float32Regs:$result,
      (int_nvvm_atomic_load_add_f32 Float32Regs:$src, Float32Regs:$val))]>;

// Warp shuffle down (Volta+ sync variant):
def SHFL_SYNC_DOWN_B32rr : NVPTXInst<(outs Int32Regs:$dst, Int1Regs:$pred),
    (ins Int32Regs:$mask, Int32Regs:$src, Int32Regs:$delta, Int32Regs:$c),
    "shfl.sync.down.b32 \t$dst|$pred, $src, $delta, $c, $mask;",
    []>;
```

The `NVPTXInst` class wires together: the output register class, input register classes, the PTX assembly string template, and the DAG pattern that SelectionDAG must match. Instructions with no DAG pattern (empty `[]`) are emitted only when explicitly inserted by NVPTX-specific lowering code.

---

## 102.2 NVPTX Target Machine and Subtarget

### 102.2.1 NVPTXTargetMachine

[`NVPTXTargetMachine.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/NVPTX/NVPTXTargetMachine.cpp) defines the top-level target machine class. It registers NVPTX-specific passes and configures the code generation pipeline:

```cpp
// NVPTXTargetMachine inherits TargetMachine, not LLVMTargetMachine, because
// NVPTX drives SelectionDAG with a custom AsmPrinter path.
class NVPTXTargetMachine : public LLVMTargetMachine {
  std::unique_ptr<TargetLoweringObjectFile> TLOF;
  mutable StringMap<std::unique_ptr<NVPTXSubtarget>> SubtargetMap;
public:
  NVPTXTargetMachine(const Target &T, const Triple &TT,
                     StringRef CPU, StringRef FS,
                     const TargetOptions &Options,
                     std::optional<Reloc::Model> RM,
                     std::optional<CodeModel::Model> CM,
                     CodeGenOptLevel OL, bool is64bit);

  const NVPTXSubtarget *getSubtargetImpl(const Function &F) const override;
  TargetPassConfig *createPassConfig(PassManagerBase &PM) override;
  // Adds NVPTX-specific IR passes: NVPTXLowerArgs, NVPTXLowerAlloca,
  // NVPTXInferAddressSpaces, NVVMReflect, NVPTXAtomicLower
  void addEarlyAsPossiblePasses(PassManagerBase &PM) override;
};
```

Key NVPTX-specific IR passes added before instruction selection:
- **`NVVMReflect`** — evaluates `@__nvvm_reflect` calls (compile-time query of NVVM flags like `__CUDA_FTZ`), replacing them with constants
- **`NVPTXLowerArgs`** — lowers kernel argument passing: byval pointer arguments are copied into `.param` space
- **`NVPTXLowerAlloca`** — converts stack allocas to PTX `.local` address space when profitable
- **`NVPTXInferAddressSpaces`** — propagates address space information through pointer chains to eliminate generic-space operations
- **`NVPTXAtomicLower`** — lowers LLVM atomic instructions to NVPTX atomic intrinsics

### 102.2.2 NVPTXSubtarget

[`NVPTXSubtarget.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/NVPTX/NVPTXSubtarget.h) holds per-function SM and PTX version information:

```cpp
class NVPTXSubtarget : public NVPTXGenSubtargetInfo {
  unsigned SmVersion;    // e.g., 89 for sm_89
  unsigned PTXVersion;   // e.g., 78 for PTX 7.8
  bool Ptx40  = false;
  bool Ptx70  = false;
  bool Ptx80  = false;
  bool Ptx85  = false;
  // Feature bits for specific SM capabilities
  bool HasFP16Math = false;       // sm_53+
  bool HasBF16Math = false;       // sm_80+
  bool HasTensorCore = false;     // sm_70+ (wmma)
  bool HasWGMMA = false;          // sm_90+ (wgmma)
  // ...
};
```

Feature flags are set via `-target-feature` arguments or derived from `-mcpu`:

```bash
# Explicitly enable PTX 8.0 and sm_90a features:
llc -march=nvptx64 \
    -target-feature=+ptx80 \
    -target-feature=+sm_90a \
    kernel.ll -o kernel.ptx
```

---

## 102.3 Address Spaces in NVPTX

### 102.3.1 PTX Memory Hierarchy

PTX formalizes the GPU memory hierarchy into distinct address spaces, each with different scope, lifetime, and performance characteristics:

| LLVM `addrspace(N)` | PTX qualifier | Scope | Lifetime |
|--------------------|--------------|-------|----------|
| 0 (generic)        | (none)       | wave (any lane can dereference) | — |
| 1 (global)         | `.global`    | device | application |
| 3 (shared)         | `.shared`    | CTA (block) | kernel |
| 4 (constant)       | `.const`     | device | application |
| 5 (local)          | `.local`     | thread | kernel |
| 7 (param)          | `.param`     | thread | kernel |

These are not just annotations — PTX load and store instructions encode the address space in their opcode: `ld.global.b32`, `st.shared.b64`, `ld.const.b128`, `ld.param.u64`.

### 102.3.2 Address Space Inference

The `NVPTXInferAddressSpaces` pass ([`NVPTXInferAddressSpaces.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/NVPTX/NVPTXInferAddressSpaces.cpp)) is critical for performance. Generic address space accesses require a hardware cycle to consult a segment descriptor; direct `ld.global` or `ld.shared` is faster. The pass walks pointer def-use chains, propagating known address spaces through `GEP`, `select`, `phi`, and `bitcast` instructions. When all predecessors of a phi carry the same address space, the phi's type is narrowed.

The `cvta.to.global.u64` / `cvta.to.shared.u64` PTX instructions convert generic pointers to specific-space pointers. These appear in PTX when LLVM must emit a generic load but later instructions operate on the specific-space value:

```llvm
; LLVM IR — generic pointer cast to global
%g = addrspacecast ptr %generic_ptr to ptr addrspace(1)
%val = load float, ptr addrspace(1) %g
```

```asm
; PTX output (without inference optimization):
cvta.to.global.u64   %rd1, %rd0
ld.global.b32        %r1, [%rd1]
```

### 102.3.3 Shared Memory Declaration

Static shared memory is declared as a module-level global in addrspace(3):

```llvm
; LLVM IR: 256-element shared float array
@shared_tile = internal addrspace(3) global [256 x float] undef, align 4
```

LLVM's NVPTX backend "demotes" such globals — they appear in the PTX output as `.shared` allocations local to the kernel function rather than as module-level symbols, because PTX `.shared` state is per-CTA and cannot be shared across kernels:

```asm
.visible .entry my_kernel(...) {
    // demoted variable:
    .shared .align 4 .b8 shared_tile[1024];
    ...
}
```

Dynamic shared memory is expressed via an extern global with zero size:

```llvm
@dynshared = external addrspace(3) global [0 x float]
; Use runtime-provided size; kernel launched with
; <<< grid, block, sharedBytes >>>
```

---

## 102.4 Kernel Entry Points and Launch Parameters

### 102.4.1 Kernel Annotation Metadata

In NVPTX LLVM IR, the distinction between a GPU kernel (`__global__` in CUDA) and a device function (`__device__`) is encoded in the `!nvvm.annotations` module-level metadata node:

```llvm
define void @my_kernel(ptr addrspace(1) %A, i64 %N) {
  ; kernel body
  ret void
}

; Module-level metadata: identifies @my_kernel as a kernel entry point
!nvvm.annotations = !{!0}
!0 = !{ptr @my_kernel, !"kernel", i32 1}
```

The triple `{ptr, !"kernel", i32 1}` tells `NVPTXAsmPrinter` to emit this function with the `.entry` directive instead of `.func`. Additional annotations control PTX metadata:

```llvm
; Reqntid annotation: required thread block dimension (launch_bounds lower bound)
!1 = !{ptr @my_kernel, !"reqntidx", i32 128}
; Maxntid annotation: maximum thread block dimension
!2 = !{ptr @my_kernel, !"maxntidx", i32 256}
!nvvm.annotations = !{!0, !1, !2}
```

These become `.reqntid` and `.maxntid` PTX directives that inform `ptxas` during register allocation.

### 102.4.2 PTX Entry Directive and Parameter Passing

The `NVPTXAsmPrinter` ([`NVPTXAsmPrinter.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/NVPTX/NVPTXAsmPrinter.cpp)) walks the function's argument list and emits parameter declarations:

```asm
; Emitted by NVPTXAsmPrinter for a kernel:
.visible .entry my_kernel(
    .param .u64 .ptr .global .align 1 my_kernel_param_0,
    .param .u64 my_kernel_param_1
)
```

Scalar arguments are passed as `.param` space values loaded with `ld.param`. Pointer arguments tagged `byval` in LLVM IR are materialized as copies into the `.param` space — the `NVPTXLowerArgs` pass handles this transformation before instruction selection.

For device functions (`.func`), parameters use a different convention: they are passed via a stack-like argument area in `.local` space, with the caller responsible for setting them up before an `call` instruction.

### 102.4.3 Built-in Thread Index Registers

CUDA's `threadIdx`, `blockIdx`, `blockDim`, `gridDim` map to PTX special registers, accessed through NVVM intrinsics:

```llvm
; Thread and block indices
%tid.x  = call i32 @llvm.nvvm.read.ptx.sreg.tid.x()
%tid.y  = call i32 @llvm.nvvm.read.ptx.sreg.tid.y()
%tid.z  = call i32 @llvm.nvvm.read.ptx.sreg.tid.z()
%ntid.x = call i32 @llvm.nvvm.read.ptx.sreg.ntid.x()  ; blockDim.x
%ctaid.x = call i32 @llvm.nvvm.read.ptx.sreg.ctaid.x() ; blockIdx.x
%nctaid.x = call i32 @llvm.nvvm.read.ptx.sreg.nctaid.x() ; gridDim.x
```

These lower directly to PTX `mov.u32` of the special register:

```asm
mov.u32    %r1, %tid.x
mov.u32    %r2, %ntid.x
mov.u32    %r3, %ctaid.x
```

A complete element-wise kernel with address computation:

```llvm
target triple = "nvptx64-nvidia-cuda"
target datalayout = "e-i64:64-i128:128-v16:16-v32:32-n16:32:64"

define void @saxpy(float %alpha, ptr addrspace(1) %x,
                   ptr addrspace(1) %y, i32 %n) {
entry:
  %tid   = call i32 @llvm.nvvm.read.ptx.sreg.tid.x()
  %ntid  = call i32 @llvm.nvvm.read.ptx.sreg.ntid.x()
  %ctaid = call i32 @llvm.nvvm.read.ptx.sreg.ctaid.x()
  %blk_off = mul i32 %ntid, %ctaid
  %i = add i32 %tid, %blk_off
  %n_cmp = icmp ult i32 %i, %n
  br i1 %n_cmp, label %body, label %exit
body:
  %i64 = zext i32 %i to i64
  %xp  = getelementptr float, ptr addrspace(1) %x, i64 %i64
  %yp  = getelementptr float, ptr addrspace(1) %y, i64 %i64
  %xv  = load float, ptr addrspace(1) %xp
  %yv  = load float, ptr addrspace(1) %yp
  %ax  = fmul float %alpha, %xv
  %res = fadd float %ax, %yv
  store float %res, ptr addrspace(1) %yp
  br label %exit
exit:
  ret void
}

declare i32 @llvm.nvvm.read.ptx.sreg.tid.x()
declare i32 @llvm.nvvm.read.ptx.sreg.ntid.x()
declare i32 @llvm.nvvm.read.ptx.sreg.ctaid.x()

!nvvm.annotations = !{!0}
!0 = !{ptr @saxpy, !"kernel", i32 1}
```

---

## 102.5 NVPTX Instruction Set and Lowering

### 102.5.1 Instruction Encoding Categories

PTX instructions encode type, size, and rounding mode in their mnemonics. The pattern is `opcode.modifier.type`:

| Category | Example PTX instructions |
|----------|-------------------------|
| Integer arithmetic | `add.s32`, `mul.lo.s32`, `mad.lo.s32`, `div.s32`, `rem.u32` |
| Float arithmetic | `add.f32`, `mul.f64`, `fma.rn.f32`, `div.rn.f64` |
| Float conversion | `cvt.rn.f32.f64`, `cvt.rzi.s32.f32` |
| Fast approx | `sqrt.approx.f32`, `rcp.approx.f32`, `sin.approx.f32` |
| Integer compare | `setp.eq.s32`, `setp.lt.u64` (result is predicate register) |
| Memory: global | `ld.global.b32`, `st.global.b64`, `ld.global.nc.b128` (non-coherent) |
| Memory: shared | `ld.shared.b32`, `st.shared.b64` |
| Memory: param | `ld.param.u64`, `ld.param.f32` |
| Atomic | `atom.global.add.s32`, `atom.global.cas.b64`, `atom.global.add.f32` |
| Control flow | `bra`, `call`, `ret`, `bar.sync`, `bar.arrive` |
| Warp intrinsics | `shfl.sync.down.b32`, `vote.sync.ballot.b32`, `match.any.sync.b32` |
| Tensor core | `wmma.load.a.sync.aligned.row.m16n16k16.f16`, `wmma.mma.sync` |

### 102.5.2 Predicate Registers

PTX uses predicate registers (`%p0`, `%p1`, ...) as one-bit boolean values produced by comparison instructions. Conditional branches consume predicates, and most arithmetic instructions have an optional predicate guard:

```asm
; Conditional execution pattern:
setp.lt.s32     %p1, %r1, %r2       ; %p1 = (r1 < r2)
@%p1 add.s32    %r3, %r1, 1         ; if (p1) r3 = r1 + 1
@!%p1 mov.s32   %r3, %r2            ; else r3 = r2
```

LLVM's NVPTX backend lowers `icmp`/`fcmp` to `setp` instructions and maps `select` instructions to guarded moves.

### 102.5.3 Select and Predicate Instructions

The `select` instruction in LLVM IR is the primary way to express conditional value computation without a branch. NVPTX lowers it through the `setp`/`selp` sequence:

```llvm
; LLVM IR: clamp x to [lo, hi]
define float @clamp(float %x, float %lo, float %hi) {
  %cmp1 = fcmp olt float %x, %lo
  %x1   = select i1 %cmp1, float %lo, float %x
  %cmp2 = fcmp ogt float %x1, %hi
  %x2   = select i1 %cmp2, float %hi, float %x1
  ret float %x2
}
```

Compiled PTX (verified with `llc -march=nvptx64 -mcpu=sm_89`):

```asm
    ld.param.b32    %r1, [clamp_param_0]  ; x
    ld.param.b32    %r2, [clamp_param_1]  ; lo
    setp.lt.f32     %p1, %r1, %r2         ; p1 = (x < lo)
    selp.f32        %r3, %r2, %r1, %p1   ; r3 = p1 ? lo : x
    ld.param.b32    %r4, [clamp_param_2]  ; hi
    setp.gt.f32     %p2, %r3, %r4         ; p2 = (r3 > hi)
    selp.f32        %r5, %r4, %r3, %p2   ; r5 = p2 ? hi : r3
    st.param.b32    [func_retval0], %r5
    ret;
```

`setp` produces a predicate register; `selp` is the predicated select. This is branchless — both paths execute in one instruction, with the predicate register selecting the result.

### 102.5.5 Multiplication and FMA

Integer multiply in PTX comes in several widths:

```asm
; 32×32→32 low half:
mul.lo.s32      %r2, %r0, %r1

; 32×32→64 wide multiply:
mul.wide.s32    %rd0, %r0, %r1

; Multiply-add (32-bit):
mad.lo.s32      %r2, %r0, %r1, %r3    ; r2 = r0*r1 + r3 (low 32)
mad.hi.s32      %r2, %r0, %r1, %r3    ; r2 = (r0*r1)>>32 + r3
```

Floating-point FMA is the preferred fusion:

```asm
; IEEE-754 round-to-nearest FMA:
fma.rn.f32      %f2, %f0, %f1, %f3    ; f2 = f0*f1 + f3
fma.rn.f64      %fd2, %fd0, %fd1, %fd3
```

### 102.5.6 Memory and Atomics Example

A complete histogram kernel illustrating global loads, address arithmetic, and atomic float add:

```llvm
target triple = "nvptx64-nvidia-cuda"
target datalayout = "e-i64:64-i128:128-v16:16-v32:32-n16:32:64"

declare float @llvm.nvvm.atomic.load.add.f32.p1(ptr addrspace(1), float)
declare i32 @llvm.nvvm.read.ptx.sreg.tid.x()

define void @histogram(ptr addrspace(1) %hist,
                        ptr addrspace(1) %data, i32 %n) {
  %tid   = call i32 @llvm.nvvm.read.ptx.sreg.tid.x()
  %tid64 = zext i32 %tid to i64
  %dptr  = getelementptr i32, ptr addrspace(1) %data, i64 %tid64
  %val   = load i32, ptr addrspace(1) %dptr
  %bin   = urem i32 %val, 256
  %bin64 = zext i32 %bin to i64
  %hptr  = getelementptr float, ptr addrspace(1) %hist, i64 %bin64
  %old   = call float @llvm.nvvm.atomic.load.add.f32.p1(
               ptr addrspace(1) %hptr, float 1.0)
  ret void
}

!nvvm.annotations = !{!0}
!0 = !{ptr @histogram, !"kernel", i32 1}
```

The NVPTX backend emits:

```asm
ld.global.b8     %r2, [%rd4]          ; load 1-byte bin value
mul.wide.u32     %rd5, %r2, 4         ; scale to float offset
add.s64          %rd6, %rd1, %rd5     ; hist + bin*4
atom.global.add.f32  %r3, [%rd6], 0f3F800000  ; atomic add 1.0f
```

### 102.5.7 Warp-Level Operations

Warp shuffle (32-thread cross-lane communication) requires the active-mask argument in Volta and later:

```llvm
; Warp shuffle: get value from lane (tid.x - delta), butterfly reduction
declare float @llvm.nvvm.shfl.sync.down.f32(i32 %mask, float %val,
                                             i32 %delta, i32 %c)

; Barrier synchronization
declare void @llvm.nvvm.barrier0()

; Full warp butterfly reduction:
define float @warp_reduce_sum(float %val) {
  %v1 = call float @llvm.nvvm.shfl.sync.down.f32(i32 -1, float %val,
                                                   i32 16, i32 31)
  %s1 = fadd float %val, %v1
  %v2 = call float @llvm.nvvm.shfl.sync.down.f32(i32 -1, float %s1,
                                                   i32 8, i32 31)
  %s2 = fadd float %s1, %v2
  %v3 = call float @llvm.nvvm.shfl.sync.down.f32(i32 -1, float %s2,
                                                   i32 4, i32 31)
  %s3 = fadd float %s2, %v3
  %v4 = call float @llvm.nvvm.shfl.sync.down.f32(i32 -1, float %s3,
                                                   i32 2, i32 31)
  %s4 = fadd float %s3, %v4
  %v5 = call float @llvm.nvvm.shfl.sync.down.f32(i32 -1, float %s4,
                                                   i32 1, i32 31)
  %s5 = fadd float %s4, %v5
  ret float %s5
}
```

These lower to `shfl.sync.down.b32` in PTX with the active mask passed as the first operand.

---

## 102.6 Warp-Uniform Operations and Divergence

### 102.6.1 The CUDA SIMT Model

CUDA's SIMT (Single Instruction, Multiple Threads) model executes 32 threads per warp in lockstep. When threads within a warp take different branches (diverge), the hardware serializes execution: the active lanes for each path are tracked via a mask, and the warp cycles through each taken branch serially, skipping inactive lanes. This has two consequences: (1) divergent branches multiply execution time by the number of distinct paths taken; (2) memory accesses with divergent addresses cannot be coalesced into a single wide transaction.

LLVM's `DivergenceAnalysis` (or its machine-level successor `MachineDivergenceAnalysis`) marks each value as either *uniform* (same across all lanes in a warp) or *divergent* (potentially lane-specific). Note: uniformity is a warp-level property — "uniform" means the same value across all 32 active lanes of the warp, not just thread-local constancy. This information drives optimization decisions — notably, uniform values can be hoisted from divergent branches and `ptxas` can exploit this to generate more efficient per-SM code.

### 102.6.2 Divergence Analysis in Practice

The divergence lattice starts with a set of *divergence sources* — values that are inherently different per lane:

| Divergent source | Reason |
|-----------------|--------|
| `tid.x/y/z` (thread index) | Each lane has a unique thread index |
| Loads from global memory with per-thread address | Data depends on tid |
| PHI nodes with divergent predecessors | Different lanes may come from different paths |

Values derived only from uniform sources remain uniform. Consider:

```llvm
; tid is DIVERGENT (different per lane)
%tid   = call i32 @llvm.nvvm.read.ptx.sreg.tid.x()
; bid is UNIFORM (same for all lanes in the block)
%bid   = call i32 @llvm.nvvm.read.ptx.sreg.ctaid.x()
; block_scale: a kernel parameter → UNIFORM
; block_off = bid * block_scale → UNIFORM (uniform × uniform)
%block_off = mul i32 %bid, %block_scale
; idx = tid + block_off → DIVERGENT (divergent + uniform)
%idx   = add i32 %tid, %block_off
; cond: icmp on divergent value → DIVERGENT BRANCH
%cond  = icmp ult i32 %tid, 16
br i1 %cond, label %do_write, label %skip
```

The divergent branch above produces the following PTX (verified with `llc -march=nvptx64 -mcpu=sm_89`):

```asm
.visible .entry gather(...) {
    .reg .pred  %p<2>;
    .reg .b32   %r<6>;
    .reg .b64   %rd<7>;

    mov.u32      %r2, %tid.x;           ; divergent: different per lane
    setp.gt.u32  %p1, %r2, 15;          ; predicate for divergent branch
    @%p1 bra     $L__BB0_2;             ; divergent branch: inactive lanes skip body
    ; ... load, multiply, store ...     ; only lanes with tid < 16 execute this
    ld.param.b32 %r3, [gather_param_2]; ; uniform: block_scale from .param
    mov.u32      %r4, %ctaid.x;         ; uniform: block ID
    mad.lo.s32   %r5, %r4, %r3, %r2;   ; uniform*uniform + divergent → divergent
$L__BB0_2:
    ret;
}
```

The `@%p1 bra` is a *predicated branch* — lanes where `%p1` is false fall through to the branch target immediately; only lanes where `%p1` is true follow the branch. On pre-Volta hardware this is implemented with convergence masks; on Volta+ (independent thread scheduling) the hardware manages reconvergence at the IPDOM (immediate postdominator).

### 102.6.3 Barrier and Synchronization Intrinsics

The CUDA `__syncthreads()` call lowers to:

```llvm
call void @llvm.nvvm.barrier0()
```

Which emits `bar.sync 0` in PTX — a CTA-wide barrier that all 32 warps in the block must reach before any can proceed. Additional barrier forms:

```llvm
; Named barriers (Kepler+): named.sync with count
declare void @llvm.nvvm.barrier.n(i32 %bar_id, i32 %thread_count)
; Hopper cluster barriers (sm_90+):
declare void @llvm.nvvm.barrier.cluster.arrive()
declare void @llvm.nvvm.barrier.cluster.wait()
```

### 102.6.4 Vote Instructions

Warp-level predicate reduction:

```llvm
; All threads in warp have pred set? → single bit result
declare i1 @llvm.nvvm.vote.all.sync(i32 %mask, i1 %pred)
; Any thread has pred set?
declare i1 @llvm.nvvm.vote.any.sync(i32 %mask, i1 %pred)
; Ballot: 32-bit bitmask of pred across warp
declare i32 @llvm.nvvm.vote.ballot.sync(i32 %mask, i1 %pred)
```

Lowered to `vote.sync.all.pred`, `vote.sync.any.pred`, `vote.sync.ballot.b32`.

### 102.6.5 Tiled Reduction with Shared Memory and Warp Shuffles

Combining shared memory, barriers, and warp shuffles in a complete block reduction:

```llvm
target triple = "nvptx64-nvidia-cuda"
target datalayout = "e-i64:64-i128:128-v16:16-v32:32-n16:32:64"

declare i32 @llvm.nvvm.read.ptx.sreg.tid.x()
declare void @llvm.nvvm.barrier0()
declare float @llvm.nvvm.shfl.sync.down.f32(i32, float, i32, i32)

@shared_partial = internal addrspace(3) global [32 x float] undef, align 4

; Block-wide sum reduction:
; - Each thread loads one element
; - Warp shuffle reduces within each warp of 32
; - Warp leaders write to shared memory
; - Thread 0 sums the warp partial results
define void @block_reduce_sum(ptr addrspace(1) %in,
                               ptr addrspace(1) %out, i32 %n) {
  %tid = call i32 @llvm.nvvm.read.ptx.sreg.tid.x()
  %tid64 = zext i32 %tid to i64
  %in_ptr = getelementptr float, ptr addrspace(1) %in, i64 %tid64
  %val = load float, ptr addrspace(1) %in_ptr

  ; Step 1: Warp-level reduction via shuffle
  %s1 = call float @llvm.nvvm.shfl.sync.down.f32(i32 -1, float %val,  i32 16, i32 31)
  %v1 = fadd float %val, %s1
  %s2 = call float @llvm.nvvm.shfl.sync.down.f32(i32 -1, float %v1,  i32 8,  i32 31)
  %v2 = fadd float %v1, %s2
  %s3 = call float @llvm.nvvm.shfl.sync.down.f32(i32 -1, float %v2,  i32 4,  i32 31)
  %v3 = fadd float %v2, %s3
  %s4 = call float @llvm.nvvm.shfl.sync.down.f32(i32 -1, float %v3,  i32 2,  i32 31)
  %v4 = fadd float %v3, %s4
  %s5 = call float @llvm.nvvm.shfl.sync.down.f32(i32 -1, float %v4,  i32 1,  i32 31)
  %v5 = fadd float %v4, %s5

  ; Step 2: Lane 0 of each warp writes its partial sum to shared mem
  %lane = urem i32 %tid, 32
  %warp = udiv i32 %tid, 32
  %is_lane0 = icmp eq i32 %lane, 0
  br i1 %is_lane0, label %write_partial, label %barrier_wait

write_partial:
  %sh_ptr = getelementptr [32 x float], ptr addrspace(3) @shared_partial,
            i32 0, i32 %warp
  store float %v5, ptr addrspace(3) %sh_ptr
  br label %barrier_wait

barrier_wait:
  call void @llvm.nvvm.barrier0()

  ; Step 3: Thread 0 sums warp partial results
  %is_t0 = icmp eq i32 %tid, 0
  br i1 %is_t0, label %final_sum, label %done

final_sum:
  ; (elided: load and sum all 32/warp elements from shared_partial)
  store float %v5, ptr addrspace(1) %out
  br label %done

done:
  ret void
}

!nvvm.annotations = !{!0}
!0 = !{ptr @block_reduce_sum, !"kernel", i32 1}
```

This pattern generates a sequence of `shfl.sync.down.b32` instructions for the warp shuffle phase and `bar.sync 0` for the shared memory barrier, exactly matching what CUDA device libraries like `cub::BlockReduce` produce.

### 102.6.6 Cooperative Groups and Cluster-Level Synchronization

Hopper (sm_90) introduces cluster-level synchronization, where groups of up to 16 CTAs can share data via distributed shared memory and synchronize with cluster barriers:

```llvm
; Cluster barrier arrival (sm_90+):
declare void @llvm.nvvm.barrier.cluster.arrive()
declare void @llvm.nvvm.barrier.cluster.wait()

; Distributed shared memory: access peer CTA's shared memory
; via addrspace(3) pointer with cluster addressing
declare ptr addrspace(3) @llvm.nvvm.mapa.shared.cluster(ptr addrspace(3), i32)
```

These lower to `bar.cluster.arrive`, `bar.cluster.wait`, and `mapa.shared.cluster` PTX instructions. Cluster-level programming requires setting `.reqclustersize` in the kernel descriptor, expressed through NVVM annotations.

---

## 102.7 PTX Emission

### 102.7.1 NVPTXAsmPrinter

The `NVPTXAsmPrinter` ([`NVPTXAsmPrinter.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/NVPTX/NVPTXAsmPrinter.cpp)) extends `AsmPrinter` and performs the final PTX text emission. Unlike most `AsmPrinter` subclasses that emit MC objects, NVPTX emits PTX text directly — there is no assembler/disassembler pair for PTX in LLVM because NVIDIA's `ptxas` fills that role.

The printer:
1. Emits the `.version` and `.target` preamble
2. Emits `.global`, `.shared`, `.const`, `.local` variable declarations
3. For each function, emits `.entry` (kernel) or `.func` (device function) with parameter declarations
4. Emits the register declarations (`.reg .b32 %r<N>`, `.reg .b64 %rd<M>`, etc.)
5. Emits each basic block's instructions

### 102.7.2 Register Naming Conventions

PTX uses typed virtual registers with naming conventions that the printer enforces:

| PTX register prefix | Type | LLVM equivalent |
|--------------------|------|-----------------|
| `%r<N>` | `.b32` — 32-bit integer/float | i32, f32 |
| `%rd<N>` | `.b64` — 64-bit integer | i64 |
| `%f<N>` | `.f32` — 32-bit float (legacy) | f32 |
| `%fd<N>` | `.f64` — 64-bit float | f64 |
| `%p<N>` | predicate register | i1 |
| `%h<N>` | `.b16` — 16-bit | i16, f16 |

This is the declaration form from the earlier example:

```asm
{
    .reg .b32    %r<4>;    ; 4 × 32-bit registers: %r0..%r3
    .reg .b64    %rd<6>;   ; 6 × 64-bit registers: %rd0..%rd5
    .reg .pred   %p<2>;    ; 2 predicate registers: %p0, %p1
```

### 102.7.3 Module-Level PTX Structure

A complete PTX module structure:

```asm
//
// Generated by LLVM NVPTX Back-End
//
.version 7.8          // PTX ISM version
.target sm_89         // Minimum SM version
.address_size 64      // 64-bit addressing

// Global variables:
.global .align 4 .b8 my_global[4096];

// Visible (exported) kernel entry:
.visible .entry saxpy(
    .param .f32 saxpy_param_0,         // alpha
    .param .u64 .ptr .global .align 4 saxpy_param_1,  // x
    .param .u64 .ptr .global .align 4 saxpy_param_2,  // y
    .param .u32 saxpy_param_3          // n
)
.reqntid 128           // hint: launch with 128 threads/block
{
    .reg .b32  %r<8>;
    .reg .b64  %rd<4>;
    .reg .f32  %f<4>;
    .reg .pred %p<2>;

    // ... instructions ...
    ret;
}
```

---

## 102.7b The SelectionDAG Lowering Path

### 102.7b.1 From LLVM IR to Machine Instructions

After the NVPTX-specific IR passes run, the standard SelectionDAG pipeline drives instruction selection. Key steps:

1. **DAGCombiner** — target-specific combines run via `NVPTXTargetLowering::PerformDAGCombine`; for example, it combines `add(mul(a,b),c)` into `mad.lo.s32` patterns
2. **SelectionDAG legalization** — ensures all types are legal for NVPTX (e.g., i8/i16 scalars are extended to i32 since PTX uses 32-bit minimum for arithmetic)
3. **Instruction selection** — `NVPTXInstrInfo.td` patterns match DAG nodes to PTX instructions
4. **Register allocation** — NVPTX uses a virtual register allocator; `ptxas` performs the actual hardware register assignment
5. **`NVPTXAsmPrinter`** — prints the allocated virtual registers using PTX naming conventions

### 102.7b.2 Type Legalization

NVPTX has no 8-bit or 16-bit arithmetic instructions for general-purpose computation (only for packed FP16 operations via `__half2` and for byte-level memory accesses). The legalizer promotes:

- `i8` arithmetic → `i32` (zero/sign-extend inputs, truncate output)
- `i16` arithmetic → `i32`
- `f16` arithmetic → `f32` (unless the `+sm_75` feature enables native FP16 math)

Memory operations (`ld.b8`, `ld.b16`, `st.b8`, `st.b16`) are still legal for loads/stores; only arithmetic on sub-32-bit values is promoted.

---

## 102.7c Tensor Core Intrinsics

### 102.7c.1 wmma — First-Generation Tensor Cores (Volta)

The Volta (`sm_70`) tensor core instruction `wmma` operates on 16×16×16 submatrices in mixed precision. LLVM exposes these through `@llvm.nvvm.wmma.*` intrinsics:

```llvm
; wmma load: load a 16×16 FP16 matrix fragment from global memory
declare void @llvm.nvvm.wmma.m16n16k16.load.a.row.stride.f16.p1(
    {<2 x i32>, <2 x i32>, <2 x i32>, <2 x i32>, <2 x i32>, <2 x i32>, <2 x i32>, <2 x i32>}* %dst,
    ptr addrspace(1) %src, i32 %stride)

; wmma mma: multiply-accumulate
declare {float, float, float, float, float, float, float, float}
    @llvm.nvvm.wmma.m16n16k16.mma.row.row.f16.f16.f32.f32(
    <2 x i32>, <2 x i32>, <2 x i32>, <2 x i32>,  ; A fragment (8 × i32)
    <2 x i32>, <2 x i32>, <2 x i32>, <2 x i32>,  ; B fragment (8 × i32)
    float, float, float, float, float, float, float, float)  ; C accumulator
```

These lower to `wmma.load.a.sync.aligned.row.m16n16k16.f16` and `wmma.mma.sync.aligned.row.row.m16n16k16.f32.f16.f16.f32` PTX instructions. Each instruction is executed by a warp of 32 threads cooperatively.

### 102.7c.2 Async Memory Copy (Ampere)

Ampere (`sm_80+`) introduces asynchronous global-to-shared memory copies that bypass the register file, enabling computation-memory overlap:

```llvm
; Async copy: global → shared (sm_80+)
declare void @llvm.nvvm.cp.async.ca.shared.global.4(
    ptr addrspace(3) %dst,   ; shared memory destination
    ptr addrspace(1) %src,   ; global memory source
    i32 4)                   ; 4/8/16 bytes per copy

; Commit the async group and wait:
declare void @llvm.nvvm.cp.async.commit.group()
declare void @llvm.nvvm.cp.async.wait.group(i32 %n)
```

These lower to `cp.async.ca.shared.global` and `cp.async.commit_group` / `cp.async.wait_group` PTX instructions. The software-managed double-buffering pattern looks like:

```llvm
; Pseudo-pipeline for async GEMM tile loop:
; Phase 1: issue async copies for tile i
call void @llvm.nvvm.cp.async.ca.shared.global.16(ptr addrspace(3) %shA,
                                                    ptr addrspace(1) %glA, i32 16)
call void @llvm.nvvm.cp.async.commit.group()
; Phase 2: compute with tile i-1 while copying tile i
; ... wmma instructions using already-loaded data ...
; Phase 3: wait for tile i to arrive
call void @llvm.nvvm.cp.async.wait.group(i32 0)
call void @llvm.nvvm.barrier0()
; Phase 4: compute with tile i
```

---

## 102.7d Cooperative Launch and Multi-GPU

### 102.7d.1 Cooperative Groups in PTX

CUDA's Cooperative Groups API (sm_60+) allows dynamic thread groups that span warps, blocks, or even multiple GPUs. At the PTX level, cooperative kernel launches use a special entry form:

```asm
; PTX cooperative kernel: all CTAs launch together, can sync at grid level
.visible .entry cooperative_matmul(
    .param .u64 .ptr .global .align 4 matA,
    .param .u64 .ptr .global .align 4 matB,
    .param .u64 .ptr .global .align 4 matC
)
.reqntid 256, 1, 1   ; exactly 256 threads per block required
{
    ; ... WMMA tile computation ...
    ; Grid-level barrier (cooperative launch required):
    bar.arrive          %rd0, %r0;   ; arrive at named barrier
    bar.sync            %rd0;        ; wait for all CTAs in grid
    ; ... reduction across CTAs ...
    ret;
}
```

LLVM exposes cooperative grid synchronization through:
```llvm
; Grid-level barrier (requires cooperative launch):
declare void @llvm.nvvm.barrier.cta.sync.aligned()
```

### 102.7d.2 Performance Considerations for PTX Generation

Several PTX-level optimizations are visible in the LLVM NVPTX backend output:

**Predicated execution vs branches**: The backend prefers `selp` (predicated select) for short if-else sequences rather than predicated branches, because `selp` has no branch prediction overhead in a SIMT environment.

**Instruction-level parallelism**: PTX does not expose a pipeline delay model — `ptxas` handles scheduling. The NVPTX backend focuses on correctness and legality rather than ILP scheduling, leaving that to `ptxas`.

**Register pressure**: Virtual registers in PTX can be numerous because `ptxas` handles register allocation. The LLVM NVPTX backend does not perform register coalescing aggressively since `ptxas` will re-allocate anyway. This differs from GCN where register pressure directly affects VGPR count and therefore occupancy.

**Memory transaction coalescing**: `ptxas` can coalesce multiple `ld.global.b32` from consecutive addresses into a wider `ld.global.b128`, but only if the LLVM backend emits the loads with appropriate alignment hints. The `NVPTXInferAddressSpaces` pass ensures alignment metadata survives to the PTX output.

---

## 102.8 The libNVVM Path

### 102.8.1 NVVM IR vs Open NVPTX

NVIDIA ships `libNVVM` — a closed-source LLVM-based compiler library bundled with the CUDA toolkit — as an alternative compilation path. NVVM IR is LLVM IR augmented with the same `!nvvm.annotations` metadata conventions used by the open NVPTX backend. The API:

```cpp
#include <nvvm.h>

nvvmProgram prog;
nvvmCreateProgram(&prog);
nvvmAddModuleToProgram(prog, llvm_ir_bitcode, size, "mymodule");
// Compile options passed as strings: "-arch=sm_90a", "-ftz=1"
nvvmCompileProgram(prog, num_options, options);
size_t ptx_size;
nvvmGetCompiledResultSize(prog, &ptx_size);
std::vector<char> ptx(ptx_size);
nvvmGetCompiledResult(prog, ptx.data());
```

### 102.8.2 Choosing Between NVPTX and libNVVM

| Dimension | LLVM NVPTX backend | libNVVM |
|-----------|-------------------|---------|
| Availability | Open source, in-tree | Closed, CUDA toolkit only |
| Pass pipeline | Full LLVM pass manager | NVIDIA's internal pipeline |
| New SM support | Lags CUDA toolkit by months | Immediate at CUDA release |
| NVVM extension coverage | Growing; matches libNVVM for common ops | Complete |
| Debugging | LLVM MIR dump, `-print-after-all` | Opaque |
| Used by | Clang CUDA, POCL, research compilers | NVCC, Numba, JAX |

The NVPTX backend's advantages are full integration with LLVM's optimization pipeline — interprocedural optimizations, vectorization, loop transformations — and debuggability. libNVVM's advantage is guaranteed correctness and completeness for every new PTX ISM feature at CUDA toolkit launch.

---

## 102.8b NVPTXTargetLowering: ABI Decisions

### 102.8b.1 Calling Convention and Return Values

`NVPTXTargetLowering` ([`NVPTXISelLowering.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/NVPTX/NVPTXISelLowering.cpp)) defines the PTX calling convention. For device functions (`.func`), return values are materialized via a `.param` space return area:

```asm
; Device function returning a struct (two floats):
.visible .func  (.param .align 8 .b8 retval0[8]) my_pair_func(
    .param .b32 my_pair_func_param_0)
{
    .reg .b32 %r<3>;
    ld.param.b32    %r1, [my_pair_func_param_0];
    fma.rn.f32      %r2, %r1, %r1, %r1;
    st.param.b32    [retval0], %r1;
    st.param.b32    [retval0+4], %r2;
    ret;
}
```

The return value area in `.param` space is written before `ret` and read by the caller after `call`. Scalar return values that fit in a single register use a simplified form (`st.param.b32 [func_retval0]`).

### 102.8b.2 Constant Memory and Texture Bindings

NVPTX supports module-level constant memory (`.const` state space = `addrspace(4)`):

```llvm
; Module-level constant buffer accessible to all kernels
@kernel_params = internal addrspace(4) constant [4 x float] [
    float 1.0, float 2.0, float 3.0, float 4.0], align 16
```

PTX output:
```asm
.const .align 16 .b8 kernel_params[16] = {0,0,128,63, 0,0,0,64, 0,0,64,64, 0,0,128,64};
```

Loads from constant memory use `ld.const.b32 %r1, [kernel_params+4]` — a broadcast load that provides the same value to all threads in a warp in a single cycle (unlike global memory which serves divergent addresses one lane at a time).

### 102.8b.3 Inline PTX Assembly in NVPTX

LLVM's inline assembly mechanism works for NVPTX through `asm` blocks:

```llvm
; Inline PTX: read nanosecond clock (sm_52+)
define i64 @get_clock() {
  %clk = call i64 asm "mov.u64 $0, %globaltimer;",
             "=l"()
  ret i64 %clk
}

; Inline PTX: write to texture coordinate register
define void @tex_grad(float %dpdx, float %dpdy) {
  call void asm sideeffect
      "tex.grad.2d.v4.f32.f32 {$0,$1,$2,$3}, [$4, {$5,$6}], {$7,$8}, {$9,$10};",
      "=f,=f,=f,=f,r,f,f,f,f,f,f"
      (ptr null, float %dpdx, float %dpdy, float 0.0, float 0.0,
       float 0.0, float 0.0, float 0.0, float 0.0)
  ret void
}
```

The `NVPTXAsmPrinter` passes inline `asm` blocks directly into the PTX output without modification — they bypass SelectionDAG instruction selection entirely.

---

## 102.9 Debugging and Inspecting PTX

### 102.9.1 Generating PTX with llc

```bash
# Compile LLVM IR to PTX targeting sm_89:
/usr/lib/llvm-22/bin/llc \
    -march=nvptx64 \
    -mcpu=sm_89 \
    kernel.ll \
    -o kernel.ptx

# Stop at a specific codegen stage for inspection:
/usr/lib/llvm-22/bin/llc \
    -march=nvptx64 -mcpu=sm_89 \
    --stop-after=nvptx-infer-addrspace \
    kernel.ll \
    -o kernel.mir
```

### 102.9.2 IR-Level Pass Inspection

Inspect what `NVPTXInferAddressSpaces` changes:

```bash
/usr/lib/llvm-22/bin/opt \
    -passes='nvptx-infer-addrspace' \
    -S kernel.ll
```

Dump the full NVPTX pass pipeline:

```bash
/usr/lib/llvm-22/bin/llc \
    -march=nvptx64 -mcpu=sm_89 \
    -debug-pass=Structure \
    kernel.ll -o /dev/null 2>&1 | grep -E "Pass|NVPTX"
```

### 102.9.3 cubin Inspection Tools

After `ptxas` compiles PTX to a cubin:

```bash
# Disassemble cubin to SASS:
nvdisasm --print-code kernel.cubin

# Dump sections and metadata from fat binary:
cuobjdump -ptx -elf -sass kernel.cubin

# Inspect PTX embedded in executable:
cuobjdump -ptx a.out
```

### 102.9.4 Workflow Summary

```
CUDA C++ → clang → NVPTX LLVM IR
                        │
            opt -passes='nvptx-*'   ← IR-level NVPTX passes
                        │
              llc -march=nvptx64    ← SelectionDAG → PTX text
                        │
                      ptxas          ← PTX → SASS cubin
                        │
                  nvdisasm / cuobjdump  ← inspect SASS
```

### 102.9.5 PTX Versioning and Feature Testing

NVPTX passes and lowering code frequently query the subtarget for feature support. A recurring pattern in the NVPTX codebase is:

```cpp
// In NVPTXISelLowering.cpp — check PTX version before emitting an instruction:
if (STI.getPTXVersion() >= 70) {
  // Use cp.async.* instructions (PTX 7.0+)
  return LowerCpAsync(Op, DAG);
}
// Fallback to register-based copy for older PTX targets

// Check SM version for tensor core support:
if (STI.getSmVersion() >= 70) {
  // wmma instructions available (sm_70+)
}
if (STI.getSmVersion() >= 80) {
  // wgmma, cp.async, TF32 tensor core (sm_80+)
}
if (STI.getSmVersion() >= 90) {
  // wgmma.mma_async, TMA, cluster sync (sm_90+)
}
```

This pattern — querying `STI.getSmVersion()` and `STI.getPTXVersion()` — is why the NVPTX backend must be invoked with the correct `-mcpu` flag: it gates not just emission directives but actual instruction lowering paths.

Cross-references: Clang's CUDA compilation driver is detailed in [Chapter 48 — Clang as a CUDA Compiler](../part-07-clang-multilang/ch48-clang-as-cuda-compiler.md). The HIP path through the AMDGPU backend is covered in the following chapter. MLIR-based GPU dialect lowering to NVPTX is covered in [Chapter 147](../part-20-in-tree-dialects/ch147-gpu-dialect.md).

---

## 102.10 The NVPTX Pass Pipeline in Detail

### 102.10.1 Full Pass Ordering

The complete pass pipeline for an NVPTX module (abridged from `-debug-pass=Structure` output for a typical kernel):

```
IR-Level Passes (before SelectionDAG):
  NVVMReflect                    ← resolve __nvvm_reflect() constants
  NVPTXLowerArgs                 ← lower byval kernel args to .param copies
  NVPTXLowerAlloca               ← promote allocas to .local space
  NVPTXInferAddressSpaces        ← propagate addrspace info through GEP/phi
  NVPTXAtomicLower               ← lower LLVM atomic instructions to NVVM intrinsics
  NVPTXFavorNonGenericAddrSpaces ← prefer non-generic memory ops

SelectionDAG Passes:
  DAGCombiner (pre-legalization)
  Legalize Types                 ← promote i8/i16 arithmetic to i32
  Legalize DAG
  DAGCombiner (post-legalization)
  Instruction Selection (NVPTXInstrInfo.td patterns)
  
Machine-Level Passes:
  MachineInstr Copy Propagation
  Dead Machine Instruction Elimination
  Peephole Optimizer (target-specific)
  Machine Block Placement        ← basic block reordering for PTX layout
  
Emission:
  NVPTXAsmPrinter                ← PTX text emission
```

### 102.10.2 NVPTXLowerArgs in Detail

`NVPTXLowerArgs` handles the fundamental ABI transformation for kernel arguments. In CUDA, kernel arguments are passed via the `.param` state space — a read-only, per-thread region that holds the kernel launch parameters. For pointer arguments marked `byval` (pass by value semantics), the CUDA ABI copies the pointed-to data into `.param` space before the kernel executes. `NVPTXLowerArgs` inserts this copy:

```llvm
; BEFORE NVPTXLowerArgs: byval pointer argument
define void @kernel({float, float}* byval({float, float}) %data) {
  %x = getelementptr {float, float}, {float, float}* %data, i32 0, i32 0
  %v = load float, float* %x
  ; ...
}

; AFTER NVPTXLowerArgs: explicit .param space copy
define void @kernel({float, float}* %data_param) {
  ; Materialize .param space address, copy to local:
  %local = alloca {float, float}, addrspace(5)
  %param_val = load {float, float}, ptr addrspace(7) %data_param  ; .param space
  store {float, float} %param_val, ptr addrspace(5) %local
  %x = getelementptr {float, float}, ptr addrspace(5) %local, i32 0, i32 0
  %v = load float, ptr addrspace(5) %x
  ; ...
}
```

This transformation is essential for correctness: PTX `.param` space is read-only, so any writes through the parameter pointer must go to a mutable local copy.

### 102.10.3 NVPTXInferAddressSpaces in Detail

The address space inference pass implements a dataflow analysis over pointer values. The key insight is that in typed LLVM IR (before opaque pointers), each `addrspacecast` was explicit — with opaque pointers, the address space is embedded in the pointer type's addrspace attribute.

The pass builds a use-def chain traversal:
1. Start with values that have known concrete address spaces (global allocations with `addrspace(N)`, kernel parameters annotated `ptx_global`, etc.)
2. Propagate through `getelementptr` (preserves space), `phi` (if all predecessors agree), `select` (if both operands agree)
3. Replace `addrspacecast` to/from generic space with direct same-space usage where provable

The benefit: a load `ld.flat` (flat/generic, requires segment detection at runtime) becomes `ld.global` (direct global space, no segment detection, lower latency). On sm_80+, global memory loads also have cache hints encoded in the opcode — `ld.global.L2::128B.b32` — which are only legal for specifically-typed global pointers.

---

## Chapter Summary

- PTX is a stable virtual ISA that LLVM generates; NVIDIA's `ptxas` compiles it to device-specific SASS machine code.
- The NVPTX backend uses `NVPTXTargetMachine` and `NVPTXSubtarget`, with SM version and PTX ISM version derived from `-mcpu` or `-target-feature` flags.
- Six address spaces (`addrspace(0..7)`) map to PTX state space qualifiers (`.global`, `.shared`, `.local`, `.const`, `.param`); `NVPTXInferAddressSpaces` propagates known spaces to eliminate generic loads.
- Kernels are distinguished from device functions by `!nvvm.annotations = !{!{ptr @fn, !"kernel", i32 1}}` module metadata; `NVPTXAsmPrinter` emits `.entry` for kernels and `.func` for device functions.
- NVPTX-specific IR passes — `NVVMReflect`, `NVPTXLowerArgs`, `NVPTXLowerAlloca`, `NVPTXAtomicLower` — run before SelectionDAG instruction selection to lower CUDA-specific constructs.
- PTX instruction encoding embeds type, size, and rounding mode in the mnemonic (`fma.rn.f32`, `ld.global.b64`, `atom.global.add.f32`).
- Warp-level primitives — `shfl.sync`, `vote.sync.ballot`, `bar.sync` — are exposed as NVVM intrinsics and lower to native PTX warp instructions.
- The open LLVM NVPTX backend and NVIDIA's closed libNVVM both consume NVVM IR; the open backend offers full debuggability and LLVM pass integration at the cost of lagging new SM feature coverage.
- `llc --stop-after=<pass>` and `opt -passes='nvptx-*'` enable stage-by-stage inspection; `nvdisasm` and `cuobjdump` inspect the resulting cubin.
