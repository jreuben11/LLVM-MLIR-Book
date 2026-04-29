# Chapter 103 — AMDGPU and the ROCm Path

*Part XV — Targets*

The AMDGPU backend in LLVM (`llvm/lib/Target/AMDGPU/`) is one of the most architecturally complex targets in the entire codebase. It simultaneously supports two distinct GPU families — the legacy GCN (Graphics Core Next) architecture and the modern RDNA series — along with a shader pipeline that covers compute kernels, vertex shaders, pixel shaders, tessellation stages, and ray-tracing. It introduced concepts to LLVM that no other backend requires at the same scale: a two-register-file model with explicit SGPR/VGPR partitioning, a 64-bit EXEC mask that gates per-lane execution, a wavefront model that differs in size between GCN (64 lanes) and RDNA (32 lanes by default), and an HSA-compliant kernel descriptor format embedded in ELF rodata. This chapter dissects every layer of that machinery.

---

## 103.1 AMDGPU Architecture Overview

### 103.1.1 GCN vs RDNA Architecture Families

AMD's GPU compute architectures relevant to modern compilers span two major lineages:

**GCN (Graphics Core Next)** — architectures gfx8 (Fiji/Tonga), gfx9 (Vega), gfx906 (Vega20/MI50), gfx908 (Arcturus/MI100), gfx90a (Aldebaran/MI200):
- Wavefront size: 64 lanes per wave
- CU (Compute Unit) contains 4 SIMD16 units, each executing one quarter of the 64-lane wave per cycle
- Register file: 256 VGPRs per lane (256 × 64-bit registers shared across 4 SIMDs on gfx9)
- Matrix core: MFMA (Matrix Fused Multiply-Add) on gfx908+ for AI workloads

**RDNA (Radeon DNA)** — gfx1010 (Navi10/RX 5700), gfx1030 (Navi21/RX 6900), gfx1100 (Navi31/RX 7900), gfx1200 (RDNA4):
- Default wavefront size: 32 lanes (wave32), with optional wave64 mode
- Dual-issue execution: two VALU instructions per clock on the same wave
- Register file: up to 1024 VGPRs per CU (larger than GCN due to wave32 default)
- Matrix core: WMMA (Wave Matrix Multiply-Accumulate) on gfx1100+

The target triple for ROCm compute work is `amdgcn-amd-amdhsa` (HSA = Heterogeneous System Architecture). Game driver work uses `amdgcn-amd-amdpal`.

### 103.1.2 The Compilation Pipeline

```
HIP / OpenCL source
        │
        ▼  clang -x hip --offload-arch=gfx1100
LLVM IR (amdgcn-amd-amdhsa, addrspace annotations, amdgpu_kernel CC)
        │
        ▼  AMDGPU backend (llc -march=amdgcn -mcpu=gfx1100)
GCN ISA (ELF, .text = ISA, .rodata = kernel descriptors)
        │
        ▼  lld / ld.lld (device linking)
AMDGPU ELF (HSA-conformant, bundled into host binary)
        │
        ▼  ROCm runtime (amdhip64.so / hsa-runtime64.so)
GPU execution
```

---

## 103.2 Target Machine and Subtargets

### 103.2.1 AMDGPUTargetMachine

[`AMDGPUTargetMachine.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp) defines two concrete target machine classes: `GCNTargetMachine` (for amdgcn-\*) and `R600TargetMachine` (legacy R600/SI, rarely used). The focus here is `GCNTargetMachine`, which configures the full modern pipeline:

```cpp
class GCNTargetMachine : public AMDGPUTargetMachine {
public:
  GCNTargetMachine(const Target &T, const Triple &TT,
                   StringRef CPU, StringRef FS,
                   const TargetOptions &Options,
                   std::optional<Reloc::Model> RM,
                   std::optional<CodeModel::Model> CM,
                   CodeGenOptLevel OL, bool JIT);

  const GCNSubtarget *getSubtargetImpl(const Function &F) const override;
  TargetPassConfig *createPassConfig(PassManagerBase &PM) override;
  // AMDGPU-specific IR passes: AMDGPUPromoteAlloca, SROAPass,
  // AMDGPUInferAddressSpaces, AMDGPULowerKernelArguments, AMDGPULowerModuleLDS
  void addEarlyAsPossiblePasses(PassManagerBase &PM) override;
};
```

### 103.2.2 GCNSubtarget

[`GCNSubtarget.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AMDGPU/GCNSubtarget.h) represents a specific GPU variant. Key feature predicates:

```cpp
class GCNSubtarget : public AMDGPUGenSubtargetInfo,
                     public AMDGPUSubtarget {
  // Wavefront size: 32 (RDNA) or 64 (GCN)
  bool HasWavefrontSize32 = false;
  bool HasWavefrontSize64 = false;
  // ISA capabilities
  bool HasGFX9Insts  = false;    // gfx9 additions
  bool HasGFX10Insts = false;    // gfx10/RDNA1
  bool HasGFX11Insts = false;    // gfx11/RDNA3
  bool HasGFX12Insts = false;    // gfx12/RDNA4
  // Specific features
  bool HasDPP       = false;     // Data Parallel Primitives
  bool HasMFMAInlineLiteralBug = false;
  bool HasMFMAMixFP8FP6FP4 = false;  // gfx950
  bool HasWMMA      = false;     // RDNA3+ WMMA
  // Register file
  bool HasAGPRs     = false;     // Accumulation GPRs (gfx908+)
  unsigned MaxNumVGPRs = 256;
  unsigned MaxNumSGPRs = 128;
};
```

The data layout for 64-bit amdgcn reflects the multi-address-space pointer model:

```
e-p:64:64-p1:64:64-p2:32:32-p3:32:32-p4:64:64-p5:32:32-p6:32:32-
i64:64-v16:16-v24:32-v32:32-v48:64-v96:128-v192:256-v256:256-
v512:512-v1024:1024-v2048:2048-n32:64-S32-A5-G1-ni:7
```

Notable: `A5` means the default alloca address space is 5 (private/scratch); `G1` means non-integral pointer for global space; `S32` means 32-byte stack alignment.

---

## 103.3 AMDGPU Address Spaces

### 103.3.1 Address Space Mapping

AMDGPU defines more address spaces than most targets, reflecting the hardware memory hierarchy:

| `addrspace(N)` | Name | HW location | Scope |
|---------------|------|-------------|-------|
| 0 | flat / generic | — | wave |
| 1 | global | VMEM (DRAM) | device |
| 2 | region (GDS) | Global Data Share | device |
| 3 | local (LDS) | Local Data Share | workgroup |
| 4 | constant | VMEM (read-only) | device |
| 5 | private / scratch | off-chip scratch | thread |
| 6 | flat (explicit) | flat global | wave |
| 7 | buffer_fat_ptr | descriptor + offset pair | device |
| 8 | buffer_resource | 4-dword descriptor | device |

`addrspace(7)` represents a "fat pointer" that bundles the 128-bit buffer resource descriptor with a 32-bit offset — used by bounds-checked buffer intrinsics. `addrspace(8)` is the raw 128-bit resource descriptor itself, not a conventional pointer. Both were formalized in LLVM 15+ and supersede some earlier raw-descriptor patterns.

The distinction between `addrspace(0)` (generic flat) and `addrspace(6)` (explicit flat global) is subtle but important for instruction selection: `addrspace(6)` always generates `global_load` / `global_store` FLAT instructions, while `addrspace(0)` may be routed through segment detection.

### 103.3.2 Address Space Inference and Promotion

`AMDGPUInferAddressSpaces` ([`AMDGPUInferAddressSpaces.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AMDGPU/AMDGPUInferAddressSpaces.cpp)) propagates known address spaces through GEP/phi/select chains, converting generic `flat_load` to `global_load` or `ds_load` where safe:

```llvm
; Before inference: generic pointer - requires flat load
%ptr_flat = addrspacecast ptr addrspace(3) %lds to ptr
%v = load i32, ptr %ptr_flat

; After inference: specific pointer - uses ds_load
%v = load i32, ptr addrspace(3) %lds
```

`AMDGPUPromoteAlloca` promotes private-space stack allocations to LDS when the entire kernel wavefront accesses the allocation with statically knowable per-lane offsets, reducing expensive scratch memory traffic.

---

## 103.4 Register File Architecture

### 103.4.0 Register File Layout Diagram

The fundamental split between VGPR (per-lane) and SGPR (uniform) register files is the defining characteristic of the AMDGPU ISA. The diagram below shows the wave64 (GCN) model:

```
VGPR register file (gfx9, wave64):
  v0 → [lane0_val] [lane1_val] [lane2_val] ... [lane63_val]  (64 × 32-bit slots)
  v1 → [lane0_val] [lane1_val] [lane2_val] ... [lane63_val]
  ...
  v255→ [lane0_val] [lane1_val] [lane2_val] ... [lane63_val]

  Effective: 256 per-lane 32-bit values = 8 KB per wave

SGPR register file (shared, wave-uniform):
  s0  → [single_value]   ← same for all 64 lanes
  s1  → [single_value]
  ...
  s127→ [single_value]

  Special: s[VCC_LO:VCC_HI] = 64-bit vector condition code
           exec = EXEC mask (64-bit gate, active lanes bitmask)

AGPR register file (gfx908+, MFMA accumulation):
  a0  → [lane0_val] [lane1_val] ... [lane63_val]  (same shape as VGPR)
  ...
  a255→ [...]
  
  Only writable via v_accvgpr_write_b32 / v_mfma_*
```

The EXEC mask is not a VGPR or SGPR — it is a dedicated hardware register that gates all VALU and memory operations. A lane whose EXEC bit is 0 is skipped for writes; loads still fetch but results are discarded.

### 103.4.1 VGPR — Vector General-Purpose Registers

VGPRs hold *divergent* (per-lane) values. Each VGPR logically contains one value per active lane — 64 values on GCN (wave64) or 32 on RDNA (wave32). The hardware register file has:

| Architecture | VGPR count per CU | Lanes per wave | VGPR per lane |
|-------------|------------------|---------------|---------------|
| gfx9 (GCN3) | 256 | 64 | 256 |
| gfx908 | 256 | 64 | 256 |
| gfx1100 (RDNA3) | 1536 | 32 | 1536 |

Instructions operating on VGPRs are prefixed `v_` and use `v0`–`vN` register names.

### 103.4.2 SGPR — Scalar General-Purpose Registers

SGPRs hold *uniform* values — a single value shared across all lanes of a wave. They are used for:
- Kernel arguments and pointers (loaded from the kernarg segment)
- Loop induction variables when provably uniform
- Workgroup IDs and wavefront dispatch parameters
- Branch conditions that are uniform across all lanes (EXEC-invariant)

Instructions operating on SGPRs are prefixed `s_`: `s_load_b32`, `s_add_u32`, `s_cbranch_vccz`. The hardware provides 128 SGPRs (gfx9) or 128+ (gfx10+), with some reserved for VCC (Vector Condition Code), EXEC mask, and STATUS/MODE registers.

A fundamental invariant: SGPR values cannot be written by VGPR-result instructions without a lane-reduction operation (`readlane`, `readfirstlane`).

### 103.4.3 AGPR — Accumulation GPRs

AGPRs appear on gfx908 (MI100) and later for matrix operations. They are a second register file of 256 accumulator registers per lane, accessible only to MFMA instructions and specific move operations. AGPR-to-VGPR copies require `v_accvgpr_read_b32 vN, aN`; VGPR-to-AGPR copies require `v_accvgpr_write_b32 aN, vN`.

### 103.4.4 Register Tuples

Wide data types use adjacent-register tuples. The LLVM register classes are:

| Register class | Width | Usage |
|---------------|-------|-------|
| `VGPR_32` | 32-bit | scalar float/int per lane |
| `VReg_64` | 64-bit | double, 64-bit int, ptr |
| `VReg_96` | 96-bit | (rare: byte-packed loads) |
| `VReg_128` | 128-bit | buffer descriptors, v4i32 |
| `VReg_256` | 256-bit | bulk loads |
| `VReg_512` | 512-bit | bulk vector ops |
| `AReg_128` | 4×32-bit AGPR | 16×16 MFMA accumulator |
| `AReg_512` | 16×32-bit AGPR | 32×32 MFMA accumulator |
| `SGPR_32` | 32-bit SGPR | uniform scalar |
| `SReg_64` | 64-bit SGPR pair | VCC, EXEC, 64-bit uniform |
| `SReg_128` | 4×SGPR | buffer resource descriptor |

### 103.4.5 Occupancy

Occupancy is the fraction of the maximum wavefronts per CU that a kernel achieves. It is constrained by three resources per CU:

- **VGPRs**: A kernel using 64 VGPRs per wave limits to 4 wavefronts (gfx9, where 256/64 = 4)
- **SGPRs**: A kernel using 32 SGPRs per wave; 128 SGPRs supports 4 wavefronts
- **LDS**: A kernel using 32 KB LDS limits to 2 wavefronts (gfx9 has 64 KB LDS, 64/32 = 2)

The AMDGPU backend emits occupancy metadata in the kernel descriptor (`.amdhsa_next_free_vgpr`, `.amdhsa_next_free_sgpr`, `.amdhsa_group_segment_fixed_size`) which the ROCm runtime uses to calculate hardware occupancy at dispatch time.

---

## 103.5 Instruction Set and Lowering

### 103.5.0 TableGen Instruction Definitions

AMDGPU instructions are defined in TableGen across files like [`SIInstructions.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AMDGPU/SIInstructions.td) and [`VOP1Instructions.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AMDGPU/VOP1Instructions.td). The encoding format hierarchy is captured in classes:

```tablegen
// In SIInstrInfo.td: base class for all SI/GCN instructions
class SIInst<dag outs, dag ins, string asm, list<dag> pattern,
             InstrItinClass itin = NoItinerary>
    : AMDGPUInst<outs, ins, asm, pattern> {
  let SubtargetPredicate = isGCN;
}

// VOP2 instruction class (2-source vector ALU):
class VOP2_Pseudo<string opName, VOPProfile P, list<dag> pattern = []>
    : InstSI<P.Outs32, P.Ins32, "", pattern> {
  let isCodeGenOnly = 1;
  let isPseudo = 1;
  string Mnemonic = opName;
  VOPProfile Pfl = P;
}

// Concrete VOP2 instance: v_add_f32
def V_ADD_F32_e32 : VOP2_Pseudo<"v_add_f32",
    VOP_F32_F32_F32,    // src0:VGPR_32, src1:VGPR_32, dst:VGPR_32
    [(set f32:$vdst, (fadd (f32 VGPR_32:$src0), (f32 VGPR_32:$src1)))]>;

// MFMA instruction (gfx908+):
def V_MFMA_F32_16X16X16F16 : VOP3_Pseudo<"v_mfma_f32_16x16x16f16",
    MFMA_F32_16x16x16F16,  // AGPRs dst, VGPR srcs, AGPR acc
    []> {
  let isConvergent = 1;
  let SubtargetPredicate = HasMFMAInsts;
  let hasSideEffects = 1;
}
```

The `VOPProfile` classes capture legal register class combinations. The `SubtargetPredicate` field gates instruction availability based on `GCNSubtarget` feature bits — `HasMFMAInsts` is only true for gfx908+. This is how the backend ensures it never emits `v_mfma_*` for gfx1100 (where WMMA is used instead).

### 103.5.1 Instruction Encoding Families

The GCN/RDNA ISA organizes instructions into encoding families, each with distinct format and operand types:

| Encoding | Format | Operands | Usage |
|----------|--------|----------|-------|
| VOP1 | 1 src VGPR | v_op v_dst, v_src | Single-input VALU |
| VOP2 | 2 src | v_op v_dst, src0, v_src1 | Binary VALU; src0 allows literal/SGPR |
| VOP3 | 3 src, optional modifiers | v_fma_f32, v_mad_f32 | FMA, abs/neg modifiers |
| VOPC | compare → VCC | v_cmp_*, vcc | Sets VCC or SDst |
| VOPD | dual-issue VOP2 pair | (RDNA3+) issue 2 ops | Throughput doubling |
| SMEM | scalar memory | s_load_b32 s_buffer_load | Uniform constant loads |
| MUBUF/MTBUF | VGPR-indexed buffer | buffer_load_dword | Global loads via descriptor |
| FLAT | flat/global | global_load_b32, flat_load | Pointer-based loads |
| DS | LDS/GDS | ds_load_b32, ds_store_b32 | Shared memory |
| MFMA | matrix fma | v_mfma_f32_32x32x1f32 | Matrix multiply |
| SOP2 | scalar binary | s_add_u32, s_and_b64 | SGPR arithmetic |
| SOPP | special scalar | s_cbranch_vccz, s_barrier | Control flow |
| VINTRP | interpolation | v_interp_p1_f32 | Vertex interpolation |

### 103.5.2 A Complete Kernel Example

Vector addition on gfx1100, with verified assembly output:

```llvm
target triple = "amdgcn-amd-amdhsa"
target datalayout = "e-p:64:64-p1:64:64-p2:32:32-p3:32:32-p4:64:64-p5:32:32-p6:32:32-i64:64-v16:16-v24:32-v32:32-v48:64-v96:128-v192:256-v256:256-v512:512-v1024:1024-v2048:2048-n32:64-S32-A5-G1-ni:7"

define amdgpu_kernel void @vec_add(ptr addrspace(1) %a,
                                    ptr addrspace(1) %b,
                                    ptr addrspace(1) %c, i32 %n) {
entry:
  %tid = call i32 @llvm.amdgcn.workitem.id.x()
  %tid_ext = zext i32 %tid to i64
  %ap = getelementptr float, ptr addrspace(1) %a, i64 %tid_ext
  %bp = getelementptr float, ptr addrspace(1) %b, i64 %tid_ext
  %cp = getelementptr float, ptr addrspace(1) %c, i64 %tid_ext
  %av = load float, ptr addrspace(1) %ap
  %bv = load float, ptr addrspace(1) %bp
  %cv = fadd float %av, %bv
  store float %cv, ptr addrspace(1) %cp
  ret void
}

declare i32 @llvm.amdgcn.workitem.id.x()
```

Compiled with `llc -march=amdgcn -mcpu=gfx1100`, this emits:

```asm
vec_add:
    s_load_b128 s[0:3], s[4:5], 0x0   ; load a, b pointers (SGPRs)
    v_and_b32_e32 v0, 0x3ff, v0        ; tid.x = v0 & 0x3ff (wave32: 10-bit)
    s_load_b64 s[4:5], s[4:5], 0x10   ; load c pointer
    s_delay_alu instid0(VALU_DEP_1)
    v_lshlrev_b32_e32 v0, 2, v0        ; tid * sizeof(float) = tid << 2
    s_waitcnt lgkmcnt(0)
    s_clause 0x1
    global_load_b32 v1, v0, s[0:1]    ; load a[tid]
    global_load_b32 v2, v0, s[2:3]    ; load b[tid]
    s_waitcnt vmcnt(0)
    v_add_f32_e32 v1, v1, v2           ; c = a + b
    global_store_b32 v0, v1, s[4:5]   ; store c[tid]
    s_endpgm
```

### 103.5.2b Understanding the GFX1100 Vec-Add Assembly

The verified assembly from section 103.5.2 merits a detailed walkthrough, because several instructions reveal the AMDGPU ISA's particular characteristics:

```asm
s_load_b128 s[0:3], s[4:5], 0x0
```
`s_load_b128` is a scalar memory load — it loads 128 bits (4 × 32-bit SGPRs) from the kernarg segment (pointed to by `s[4:5]`). Loading kernel arguments via SGPR ensures they are uniform across all lanes, enabling optimal register class selection. The two pointers `a` and `b` (8 bytes each = 16 bytes = 128 bits) arrive in `s[0:3]`.

```asm
v_and_b32_e32 v0, 0x3ff, v0
```
`v0` on entry holds the workitem ID (thread index within the workgroup), pre-loaded by the hardware into VGPR 0 based on `.amdhsa_system_vgpr_workitem_id 2`. The `0x3ff` mask limits to 10 bits, extracting the X component of the packed (x, y, z) workitem ID.

```asm
v_lshlrev_b32_e32 v0, 2, v0
```
Multiply `tid` by 4 (left shift by 2) to get the byte offset for `float` elements. The result is a VGPR value — divergent, since each lane has a different `tid`.

```asm
s_clause 0x1
global_load_b32 v1, v0, s[0:1]
global_load_b32 v2, v0, s[2:3]
```
`s_clause 0x1` declares a "clause" of 2 memory operations, hinting to the hardware to pre-fetch both loads before stalling. `global_load_b32` uses the FLAT global address space: `vN = *(s_base + v_offset)`. The SGPR pair provides the uniform base address; the VGPR provides the divergent per-lane offset.

```asm
s_waitcnt vmcnt(0)
```
Wait for all outstanding VMEM (vector memory) operations to complete. The `vmcnt` counter decrements as loads complete. Without this, the subsequent `v_add_f32` would use unloaded data.

### 103.5.3 MFMA Matrix Instructions

MFMA (Matrix Fused Multiply-Accumulate) instructions are the core of AMD's AI compute strategy. They operate on vector register groups representing submatrices and accumulate results into AGPR file registers:

```llvm
; GFX908: 16×16×16 FP16 MFMA → <4 x float> accumulator
define amdgpu_kernel void @mfma_f16(ptr addrspace(1) %out) {
  ; Initialize accumulator to zero (4 AGPR lanes = 16×16 partitioned among 64 threads)
  %acc = call <4 x float> @llvm.amdgcn.mfma.f32.16x16x16f16(
    <4 x half> zeroinitializer,   ; A matrix fragment (4 half elements per thread)
    <4 x half> zeroinitializer,   ; B matrix fragment
    <4 x float> zeroinitializer,  ; C accumulator
    i32 0, i32 0, i32 0)          ; cbsz, abid, blgp modifiers
  %v0 = extractelement <4 x float> %acc, i32 0
  store float %v0, ptr addrspace(1) %out
  ret void
}

declare <4 x float> @llvm.amdgcn.mfma.f32.16x16x16f16(
    <4 x half>, <4 x half>, <4 x float>, i32, i32, i32)
```

Compiled, this emits the MFMA instruction along with `v_accvgpr_write_b32` / `v_accvgpr_read_b32` to transfer between VGPR and AGPR register files:

```asm
    v_accvgpr_write_b32 a0, 0   ; initialize AGPRs for accumulator
    v_accvgpr_write_b32 a1, 0
    v_accvgpr_write_b32 a2, 0
    v_accvgpr_write_b32 a3, 0
    v_mfma_f32_16x16x16f16 a[0:3], v[1:2], v[1:2], a[0:3]
    s_nop 8                     ; hazard: 8 cycle wait after MFMA
    v_accvgpr_read_b32 v1, a0   ; read result from AGPR back to VGPR
```

WMMA instructions (RDNA3 gfx1100+) use a different form targeting VGPRs directly:

```llvm
declare <8 x float> @llvm.amdgcn.wmma.f32.16x16x16.f16(
    <16 x half>, <16 x half>, <8 x float>)
```

### 103.5.3b MFMA Occupancy and Tiling

A critical MFMA programming pattern is managing the 8-cycle latency after MFMA instructions (`s_nop 8` or instruction scheduling to cover it). When writing a matrix multiplication tile, the outer loop must be structured to interleave MFMA operations with memory loads:

```asm
; Pseudo-code for double-buffered MFMA tile loop:
;
; PROLOGUE: Load tile A[0] and B[0] into VGPR registers
; LOOP:
;   MFMA tile: v_mfma_f32_... a[0:31], v_A, v_B, a[0:31]  ← uses prev tile
;   ds_read_b128 v_A_next, ... (next A tile from LDS)       ← pipelined load
;   ds_read_b128 v_B_next, ... (next B tile from LDS)       ← runs during MFMA
;   s_waitcnt lgkmcnt(0)                                     ← wait for LDS
;   (swap A/B double buffers)
; EPILOGUE: v_accvgpr_read_b32 → VGPRs, store result
```

The `cbsz`, `abid`, and `blgp` modifier fields in MFMA intrinsics control which sub-block of the A/B matrix the current wave processes (for large-matrix GEMM where multiple waves cooperate). ROCm's BLAS libraries (hipBLAS, rocBLAS) use these for high-performance GEMM implementations that the AMDGPU backend must correctly lower.

### 103.5.3c s_waitcnt and Memory Ordering

The `SIInsertWaitcnts` pass ([`SIInsertWaitcnts.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AMDGPU/SIInsertWaitcnts.cpp)) inserts `s_waitcnt` instructions based on in-flight operation counters. The GCN/RDNA ISA tracks outstanding operations in three separate counters:

| Counter | Meaning | Operations tracked |
|---------|---------|-------------------|
| `vmcnt(N)` | VMEM in-flight count ≤ N | Global loads, buffer loads, FLAT loads |
| `lgkmcnt(N)` | LGKM in-flight count ≤ N | LDS loads (DS), constant loads (SMEM), messages |
| `expcnt(N)` | EXP in-flight count ≤ N | Export instructions (vertex/pixel shaders) |

A `s_waitcnt` instruction stalls until the specified counter drops to or below N. The pass computes the minimum necessary wait at each point where a result is consumed:

```asm
; Pattern: VMEM load followed by ALU use of result
global_load_b32 v1, v0, s[0:1]  ; outstanding vmcnt increments to 1
global_load_b32 v2, v0, s[2:3]  ; outstanding vmcnt increments to 2
s_waitcnt vmcnt(0)               ; wait for BOTH loads to complete
v_add_f32_e32 v1, v1, v2         ; safe to use v1, v2

; Pattern: LDS store followed by barrier and LDS load
ds_store_b32 v1, v2              ; outstanding lgkmcnt increments
s_barrier                        ; implied: s_waitcnt lgkmcnt(0) before barrier
ds_load_b32 v0, v0               ; outstanding lgkmcnt increments
s_waitcnt lgkmcnt(0)             ; wait for LDS load to complete
v_add_f32_e32 v3, v0, v4
```

GFX11 introduced `s_wait_loadcnt`, `s_wait_dscnt` and related per-counter instructions that replace the combined `s_waitcnt` encoding from earlier ISA generations. The AMDGPU backend generates the correct form based on the target's `GCNSubtarget` feature bits.

### 103.5.4 Key Correctness Passes

The AMDGPU backend has several correctness-enforcing passes that run late in the pipeline:

- **`SIFixSGPRCopies`** — detects illegal copies between VGPR and SGPR register classes and inserts the required `v_readfirstlane_b32` or phi-of-readfirstlane sequences
- **`SILowerI1Copies`** — lowers i1 (predicate) copies, which cannot be represented directly; expands to VCC-based sequences
- **`SIFoldOperands`** — folds literal constants and SGPRs into VOP instruction src0 slots, reducing register pressure
- **`GCNNSAReassign`** — reassigns NSA (Non-Sequential Addressing) image instruction VGPR arguments to be sequential when profitable
- **`SIInsertWaitcnts`** — inserts `s_waitcnt` instructions to enforce correct memory ordering between VMEM, LGKM (LDS+GDS+constant+message) and EXP domains

---

## 103.6 Calling Conventions and Kernel ABI

### 103.6.1 HSA Kernel Descriptor

The AMDGPU HSA ABI requires a 64-byte kernel descriptor structure in ELF rodata for each kernel. The backend emits this via `.amdhsa_kernel` directives, which the assembler encodes into the `.rodata` section. The runtime loader reads this descriptor before dispatch to configure wavefront size, VGPR/SGPR counts, LDS size, and kernel argument segment size.

The descriptor fields visible in the assembly output:

```asm
.amdhsa_kernel vec_add
    .amdhsa_group_segment_fixed_size 0    ; LDS bytes allocated
    .amdhsa_private_segment_fixed_size 0  ; scratch bytes per thread
    .amdhsa_kernarg_size 288              ; kernel argument segment size (bytes)
    .amdhsa_user_sgpr_count 13            ; # SGPRs for user kernel arguments
    .amdhsa_user_sgpr_dispatch_ptr 1      ; pass dispatch packet ptr?
    .amdhsa_user_sgpr_queue_ptr 1         ; pass HSA queue ptr?
    .amdhsa_user_sgpr_kernarg_segment_ptr 1  ; pass kernarg base ptr?
    .amdhsa_system_sgpr_workgroup_id_x 1  ; pass workgroup ID X?
    .amdhsa_system_sgpr_workgroup_id_y 1  ; pass workgroup ID Y?
    .amdhsa_system_sgpr_workgroup_id_z 1  ; pass workgroup ID Z?
    .amdhsa_system_vgpr_workitem_id 2     ; encode tid.xyz in v0?
    .amdhsa_next_free_vgpr 4              ; occupancy: #VGPRs used
    .amdhsa_next_free_sgpr 16             ; occupancy: #SGPRs used
    .amdhsa_wavefront_size32 1            ; 1=wave32, 0=wave64
    .amdhsa_ieee_mode 1
    .amdhsa_fp16_overflow 0
```

### 103.6.2 Kernel Argument Passing

Kernel arguments in the AMDGPU HSA ABI are passed through the kernarg segment — a memory region pointed to by a dedicated SGPR pair (`s[N:N+1]`, typically `s[4:5]` or `s[8:9]` depending on the prefix SGPR layout). Arguments are laid out sequentially at their natural alignment.

The kernarg segment memory layout for `vec_add(ptr a, ptr b, ptr c, i32 n)`:

```
Kernarg segment (base = s[4:5]):
  Offset 0x00: [8 bytes] ptr a         ← s_load_b64 s[0:1], s[4:5], 0x0
  Offset 0x08: [8 bytes] ptr b         ← part of s_load_b128 s[0:3]
  Offset 0x10: [8 bytes] ptr c         ← s_load_b64 s[4:5], s[4:5], 0x10
  Offset 0x18: [4 bytes] i32 n         ← s_load_b32 s[6], s[4:5], 0x18
  Offset 0x1C: [4 bytes padding]
  Offset 0x20..0x11F: [256 bytes] implicit arguments (dispatch_ptr, etc.)
  Total size: 0x120 = 288 bytes (matches .amdhsa_kernarg_size 288)
```

```asm
; GFX1100 vec_add kernel: four arguments (ptr, ptr, ptr, i32)
; Kernarg segment layout: [a:8][b:8][c:8][n:4] = 28 bytes, padded to 288 for implicit args
s_load_b128 s[0:3], s[4:5], 0x0    ; load a (8B) and b (8B) → s[0:1], s[2:3]
s_load_b64  s[4:5], s[4:5], 0x10   ; load c (8B) → s[4:5]
; n is loaded separately via s_load_b32 s[6], s[4:5], 0x18
```

The 256 bytes of "implicit" arguments appended to explicit args carry runtime-populated fields: grid size, workgroup size, printf buffer pointer, etc. These are the "hidden" arguments `--amdhsa_kernarg_size` accounts for beyond the explicit user arguments.

### 103.6.3 Implicit Kernel Arguments (GFX9 vs GFX10+)

GFX9 and earlier receive several implicit arguments via dedicated SGPR setups: `private_segment_buffer` (4 SGPRs), `dispatch_ptr` (2 SGPRs), `queue_ptr` (2 SGPRs), `kernarg_segment_ptr` (2 SGPRs), and workgroup IDs. GFX10+ simplified this: the kernarg pointer is the primary vehicle, and implicit argument fields live at a fixed negative offset from the kernarg base.

The `AMDGPUFunctionArgInfo` class ([`AMDGPUFunctionArgInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AMDGPU/AMDGPUFunctionArgInfo.h)) tracks which implicit arguments are live for each kernel, enabling the backend to omit unused ones and reduce SGPR pressure.

---

## 103.6b GFX9 vs GFX10+ Implicit Argument Differences

The implicit argument scheme changed significantly between GFX9 (Vega/CDNA) and GFX10+ (RDNA). Understanding this prevents subtle ABI bugs when targeting multiple GPU generations.

### 103.6b.1 GFX9 Prefix SGPR Layout

GFX9 kernels receive a fixed set of SGPRs before user-defined kernel arguments, with each present/absent slot indicated by `.amdhsa_user_sgpr_*` descriptor bits:

```
GFX9 SGPR layout at kernel entry (max configuration):
  s[0:3] = private_segment_buffer   (4 SGPRs; base + size for scratch)
  s[4:5] = dispatch_ptr              (2 SGPRs; HSA dispatch packet)
  s[6:7] = queue_ptr                 (2 SGPRs; HSA queue)
  s[8:9] = kernarg_segment_ptr       (2 SGPRs; base of kernarg segment)
  s[10:11] = dispatch_id             (2 SGPRs; unique dispatch ID)
  s[12:13] = flat_scratch_init       (2 SGPRs; scratch base + size)
  s[14] = workgroup_id_x
  s[15] = workgroup_id_y
  s[16] = workgroup_id_z
```

Kernarg arguments are then loaded from `s[8:9]` (kernarg_segment_ptr) at byte offsets.

### 103.6b.2 GFX10/GFX11 Simplified Layout

RDNA (gfx10/gfx11) removed `private_segment_buffer` and simplified scratch access. The kernarg pointer is typically at `s[0:1]`:

```
GFX10/GFX11 SGPR layout (minimal dispatch):
  s[0:1] = kernarg_segment_ptr   (2 SGPRs, first user SGPR)
  s[2]   = workgroup_id_x
  s[3]   = workgroup_id_y
  s[4]   = workgroup_id_z
  v0     = workitem_id (packed xyz in 10+10+10 bit fields, or split)
```

This is why the GFX1100 assembly in section 103.5.2 uses `s_load_b128 s[0:3], s[4:5], 0x0` — `s[4:5]` is the kernarg pointer when `dispatch_ptr` and `queue_ptr` are also enabled. The exact offset depends on which implicit arguments are present, as tracked by `AMDGPUFunctionArgInfo`.

---

## 103.7 Divergence and Wave-Level Control Flow

### 103.7.1 The EXEC Mask

The EXEC mask is a 64-bit (wave64) or 32-bit (wave32) register — accessible as `exec` / `exec_lo` / `exec_hi` — that gates which lanes are active for each instruction. A bit set in EXEC means the corresponding lane participates; a cleared bit suppresses writes and memory operations for that lane.

Divergent control flow is implemented by manipulating EXEC:

```asm
; Divergent branch: if (tid < threshold)
; Generated by AMDGPU backend for gfx1100 (wave32):
v_cmp_le_u32_e32 vcc_lo, s0, v0       ; compute branch condition per lane
s_and_saveexec_b32 s0, vcc_lo         ; EXEC &= cond; save old EXEC in s0
s_xor_b32 s0, exec_lo, s0            ; s0 = lanes that took false branch
    ; true branch code here (EXEC = lanes with tid < threshold)
s_and_not1_saveexec_b32 s0, s0        ; EXEC = s0 (false branch lanes)
    ; false branch code here
s_or_b32 exec_lo, exec_lo, s0        ; restore EXEC (all lanes active)
```

This sequence is generated by `SIAnnotateControlFlow` and its successors. The "save/restore EXEC" pattern ensures correct phi-node semantics: each path writes to a subset of lanes, and the merge point sees all lanes' results through the EXEC mask operations.

### 103.7.1b EXEC Mask State Diagram

The following diagram traces the EXEC mask across a divergent branch for a wave32 example where `threshold = 8` (only lanes 0–7 take the true branch, lanes 8–31 take the false branch):

```
Initial EXEC (all 32 lanes active, wave32):
  exec_lo = 0xFFFFFFFF  (bits 31..0 all set)

After v_cmp_le_u32 vcc_lo, threshold, v0:
  vcc_lo  = 0x00FF00FF  (example: lanes where tid < threshold = bits 0..7 set)

After s_and_saveexec_b32 s0, vcc_lo:
  exec_lo = 0x000000FF  (only lanes 0..7 active: TRUE branch executes)
  s0      = 0xFFFFFFFF  (saved original EXEC)

After s_xor_b32 s0, exec_lo, s0:
  s0      = 0xFFFFFF00  (lanes that did NOT take true branch = lanes 8..31)

  [TRUE BRANCH CODE: only lanes 0..7 execute]

After s_and_not1_saveexec_b32 s0, s0:
  exec_lo = 0xFFFFFF00  (only lanes 8..31 active: FALSE branch executes)
  s0      = 0x000000FF  (stash of true-branch lanes, for OR at end)

  [FALSE BRANCH CODE: only lanes 8..31 execute]

After s_or_b32 exec_lo, exec_lo, s0:
  exec_lo = 0xFFFFFFFF  (all 32 lanes active again at merge point)
```

This is the exact sequence emitted by `SIAnnotateControlFlow` for the divergent branch in section 103.7.1. At each point only the active lanes write to registers or memory; inactive lanes are architecturally invisible to those instructions.

### 103.7.2 Ballot and Cross-Lane Communication

Warp-level ballot and cross-lane reads on AMDGPU use SGPR-based intrinsics:

```llvm
; ballot: i64 (wave64) bitmask of lanes where pred is true
%ballot = call i64 @llvm.amdgcn.ballot.i64(i1 %pred)

; readfirstlane: extract VGPR value from lane 0 into SGPR
%uniform = call i32 @llvm.amdgcn.readfirstlane.i32(i32 %divergent_val)

; readlane: extract VGPR value from specific lane into SGPR
%from_lane3 = call i32 @llvm.amdgcn.readlane.i32(i32 %val, i32 3)
```

These lower to:
```asm
s_ballot_b64   s[0:1], exec      ; ballot of active lanes
v_readfirstlane_b32  s2, v0      ; first-lane read
v_readlane_b32  s3, v0, 3       ; specific lane read
```

### 103.7.3 DPP — Data Parallel Primitives

DPP (Data Parallel Primitives) enables cross-lane communication within a wave without going through LDS, using lane shift patterns encoded in instruction modifiers:

```asm
; Shift all lanes left by 1 within each row of 16:
v_add_f32 v0, v0, v0 row_shr:1

; Butterfly permute for reduction:
v_add_f32 v0, v0, v0 row_xmask:1   ; XOR lane index with 1
```

LLVM exposes DPP through intrinsics like `@llvm.amdgcn.update.dpp.i32`.

### 103.7.4 s_barrier and Whole-Quad Mode

The workgroup barrier is `s_barrier`, emitted for `@llvm.amdgcn.s.barrier()`. It synchronizes all wavefronts in the workgroup and enforces memory visibility for LDS operations (DS instructions).

Whole-Quad Mode (WQM) is used by texture sampling instructions, which require a 2×2 quad of lanes to compute derivatives even when some lanes are masked off. `SIWholeQuadMode` inserts the EXEC mask management needed to ensure quads are intact for texture calls.

---

## 103.8 LDS: Local Data Share

### 103.8.1 LDS Usage Pattern

LDS (Local Data Share) is the high-bandwidth, low-latency shared memory accessible to all wavefronts in a workgroup (analogous to CUDA shared memory). LDS uses DS instructions:

```llvm
; LDS tiled reduction kernel
@lds_tile = internal addrspace(3) global [64 x float] undef, align 4

define amdgpu_kernel void @lds_reduce(ptr addrspace(1) %in,
                                       ptr addrspace(1) %out) {
  %tid = call i32 @llvm.amdgcn.workitem.id.x()
  %tid64 = zext i32 %tid to i64
  %in_ptr  = getelementptr float, ptr addrspace(1) %in, i64 %tid64
  %val     = load float, ptr addrspace(1) %in_ptr
  %lds_ptr = getelementptr [64 x float], ptr addrspace(3) @lds_tile,
             i32 0, i32 %tid
  store float %val, ptr addrspace(3) %lds_ptr
  call void @llvm.amdgcn.s.barrier()
  ; ...
}
```

The compiler emits `ds_store_b32` and `ds_load_b32` for LDS accesses, with byte-addressed offsets. The `.amdhsa_group_segment_fixed_size` descriptor field records LDS allocation for occupancy computation.

### 103.8.2 Bank Conflicts and Padding

LDS is divided into 32 banks of 4 bytes each (on most GCN/RDNA hardware). Accesses from multiple lanes within a wavefront to the same bank in a single cycle are serialized — a "bank conflict." The bank of an address is determined by `(byte_address / 4) % 32`.

A classic example: a matrix transpose where a 32×32 matrix stored in column-major order is loaded row-by-row from LDS. Lane `i` accesses column `i` of a row, which means addresses `i * 32 * 4` bytes apart — all with the same bank (`i * 32 % 32 == 0`), causing a 32-way bank conflict.

The standard fix is column padding:

```llvm
; WITHOUT padding: 32×32 floats, 32-way bank conflict on transpose load
@tile_bad  = internal addrspace(3) global [32 x [32 x float]] undef, align 4

; WITH padding: 32×33 floats, the extra column offsets banks by 1 per row
@tile_good = internal addrspace(3) global [32 x [33 x float]] undef, align 4
; Lane i accesses column i of row: stride = 33*4 bytes
; bank = (row * 33 * 4 + i * 4) / 4 % 32 = (row*33 + i) % 32
; Different rows have different base banks → no conflict
```

The AMDGPU backend does not automatically insert this padding — it is a programmer or compiler pass responsibility. Some MLIR GEMM codegen passes (e.g., in IREE and Triton) compute optimal LDS padding automatically.

`ds_load_2addr` is a packed LDS load that loads two elements from two separate addresses in one instruction, halving LDS access cycles for non-conflicting patterns:

```asm
; Packed LDS load: load from two addresses simultaneously
ds_load_2addr v[0:1], v2, 0x0, 0x4   ; v0 = LDS[v2+0], v1 = LDS[v2+4]
```

---

## 103.8b AMDGPU Target Lowering and Legalization

### 103.8b.1 AMDGPUTargetLowering

`AMDGPUTargetLowering` ([`AMDGPUISelLowering.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AMDGPU/AMDGPUISelLowering.cpp)) handles the type legalization and custom lowering required before SelectionDAG can map LLVM IR to GCN instructions.

Key lowering decisions:

- **i64 → i32 pair**: AMDGPU has no native 64-bit integer arithmetic on the VALU. `add i64` is lowered to `v_add_co_u32` + `v_addc_co_u32` (add with carry). The `co` suffix means the carry output goes to VCC, which participates in the chained add.

```asm
; 64-bit add: (s64 vdst) = v_add64(v_src0, v_src1)
; Lowered to two VOP2 instructions:
v_add_co_u32_e32   v0, vcc, v0, v2    ; low 32 bits + carry out in VCC
v_addc_co_u32_e32  v1, vcc, v1, v3, vcc  ; high 32 bits + carry in from VCC
```

- **f64**: RDNA3 (gfx1100) does not have native FP64 VALU support (FP64 is removed in consumer GPUs). The subtarget predicate `HasFP64` gates FP64 instruction availability. Without it, `double` arithmetic is software-emulated.

- **Vectors**: Operations on `<2 x i32>` are lowered to dual VOP2 instructions; `<4 x i32>` to four instructions. The AMDGPU backend does not have a native SIMD32/64 vector unit — the "vectorization" is the wavefront itself.

### 103.8b.2 SGPR-VGPR Copy Legality

The most distinctive legality constraint: scalar (SGPR) values cannot be produced by vector (VGPR) instructions without an explicit cross-file operation. The SIFixSGPRCopies pass enforces this:

```llvm
; This IR is valid LLVM but illegal for AMDGPU without fixup:
%uniform_val = call i32 @llvm.amdgcn.workgroup.id.x()  ; → SGPR (uniform)
%divergent_ptr = inttoptr i64 %some_divergent to ptr   ; → VGPR (divergent)
; Illegal: trying to write a VGPR result into an SGPR location
%bad_copy = add i32 %uniform_val, %some_vgpr_val       ; forces VGPR result
```

`SIFixSGPRCopies` detects when a value defined in a VGPR context is used as an SGPR operand and inserts `v_readfirstlane_b32` to perform the lane-0 extraction:

```asm
; Auto-inserted by SIFixSGPRCopies:
v_readfirstlane_b32 s5, v3    ; extract lane-0 value from v3, assign to SGPR
s_add_u32 s2, s2, s5          ; now safe to use in SGPR add
```

This is semantically correct only when the VGPR value is genuinely uniform across lanes. If it is not, the program has undefined behavior — this is a GPU ABI responsibility on the programmer/compiler, not something LLVM can verify.

### 103.8b.3 AMDGPUPromoteAlloca

The `AMDGPUPromoteAlloca` pass promotes function-local allocas from private (scratch) space to LDS when profitable. Scratch memory on AMDGPU is off-chip DRAM — extremely high latency (~400 cycles). LDS is on-chip (~4-cycle access). A small array accessed with statically known per-lane offsets is a candidate:

```llvm
; Before promotion: in private (scratch) space
define amdgpu_kernel void @foo() {
  %arr = alloca [8 x float], addrspace(5)   ; private scratch
  %tid = call i32 @llvm.amdgcn.workitem.id.x()
  %ptr = getelementptr [8 x float], ptr addrspace(5) %arr, i32 0, i32 %tid
  store float 1.0, ptr addrspace(5) %ptr
  ; ...
}
```

After promotion (if all accesses have statically determinable offsets within the wave):
```llvm
; After promotion: in LDS
@_promoted_arr = internal addrspace(3) global [8 x float] undef
; Access via LDS instructions instead of scratch
```

The promotion is gated on occupancy analysis: promoting a large array that exhausts LDS would reduce occupancy, potentially harming throughput more than it helps latency.

---

## 103.9 The ROCm Software Stack

### 103.9.1 HIP to ELF

The full compilation pipeline for a HIP kernel follows:

```bash
# Step 1: Clang compiles .hip → LLVM IR + host wrapper
clang -x hip -offload-arch=gfx1100 kernel.hip \
      --offload-device-only -emit-llvm -S -o kernel_device.ll

# Step 2: AMDGPU backend compiles IR → GCN ELF
/usr/lib/llvm-22/bin/llc -march=amdgcn -mcpu=gfx1100 \
    -filetype=obj kernel_device.ll -o kernel_device.o

# Step 3: Link device ELFs
ld.lld -flavor gnu --no-undefined -shared \
    kernel_device.o -o kernel_device.elf

# Step 4: Inspect the ELF
llvm-readelf --notes kernel_device.elf
```

The ELF produced is an HSA-conformant binary with:
- `.text` section: GCN ISA machine code
- `.rodata` section: kernel descriptor structures (64 bytes each)
- `.note` sections: AMDGPU metadata (ISA version, kernel metadata, code object version)

### 103.9.2 AMDGPU Metadata in ELF

The AMDGPU ELF notes carry metadata consumed by the HSA runtime:

```bash
llvm-readelf --notes kernel_device.elf
# Outputs something like:
# NT_AMD_HSA_ISA_NAME: amdgcn-amd-amdhsa--gfx1100
# NT_AMD_HSA_METADATA: yaml document with kernel names, arg types, sizes
```

The YAML metadata encodes argument types, qualifiers, sizes, and offsets in a format that ROCm's `amdhsa` runtime uses to set up dispatch packets without requiring the source-level information.

### 103.9.3 Debugging and Profiling

```bash
# Compile to assembly for inspection:
/usr/lib/llvm-22/bin/llc -march=amdgcn -mcpu=gfx1100 \
    kernel.ll -o kernel.s

# Disassemble object file:
llvm-objdump --arch=amdgcn --mcpu=gfx1100 -d kernel.o

# Inspect MIR at a specific pass:
/usr/lib/llvm-22/bin/llc -march=amdgcn -mcpu=gfx1100 \
    --stop-after=si-fix-sgpr-copies --mir-print-level=2 \
    kernel.ll -o kernel.mir

# Estimate occupancy from assembly:
# .amdhsa_next_free_vgpr and .amdhsa_group_segment_fixed_size
# determine hardware occupancy at runtime via the HSA descriptor
```

ROCm-specific tooling:

```bash
# Profile with ROCm profiler (requires ROCm runtime):
rocprof --stats --timestamp on ./app_gpu

# Hardware performance counters:
rocprof --hardware-counters <counter_list> ./app_gpu
```

### 103.9.4 ROCm Profiling Deep Dive

ROCm provides `rocprof` for hardware performance counter sampling and trace collection. A typical profiling workflow:

```bash
# Profile with hardware counters (occupancy + memory bandwidth):
rocprof --stats \
        --hardware-counters SQ_WAVES,FETCH_SIZE,WRITE_SIZE \
        ./my_hip_app

# Output a CSV with per-kernel metrics:
# KernelName, DurationNs, SQ_WAVES, FETCH_SIZE, WRITE_SIZE

# Estimate occupancy via rocminfo and the .amdhsa_next_free_vgpr value:
# Occupancy = floor(MAX_WAVES_PER_CU / ceil(NEXT_FREE_VGPR / GRANULE))
# Where GRANULE = 4 for gfx9, 8 for gfx10/gfx11
```

The compiler emits `.amdhsa_next_free_vgpr 4` in the kernel descriptor (verified in the `lds_reduce` example above): this means 4 VGPRs are allocated. On gfx1100 with 1536 VGPRs per CU and wave32:
- Max waves per CU = 1536 / ceil(4/8)*8 = 1536 / 8 = 192 waves
- At a workgroup size of 64, this means 3 wavefronts per workgroup, 64 workgroups per CU at 1536/8 VGPRs

### 103.9.5 Shader-Specific Calling Conventions

Beyond `amdgpu_kernel`, the AMDGPU backend supports shader-specific calling conventions used by the Mesa/RADV Vulkan driver and AMDPAL game driver paths:

| Calling convention | LLVM CC name | Usage |
|-------------------|-------------|-------|
| `amdgpu_kernel` | `CC_AMDGPU_KERNEL` | HSA compute kernels |
| `amdgpu_vs` | `CC_AMDGPU_VS` | Vertex shaders |
| `amdgpu_gs` | `CC_AMDGPU_GS` | Geometry shaders |
| `amdgpu_ps` | `CC_AMDGPU_PS` | Pixel/fragment shaders |
| `amdgpu_cs` | `CC_AMDGPU_CS` | Compute shaders (AMDPAL) |
| `amdgpu_hs` | `CC_AMDGPU_HS` | Hull/tessellation shaders |
| `amdgpu_ls` | `CC_AMDGPU_LS` | Local (pre-hull) shaders |
| `amdgpu_es` | `CC_AMDGPU_ES` | Export (pre-geometry) shaders |

Pixel shaders (`amdgpu_ps`) have special VGPRs: `v0`, `v1`, `v2` hold barycentric coordinates (or sample position for rasterized pixels), and `v3` may hold the sample mask. Vertex shaders receive their vertex buffer data via SGPR descriptors. The `AMDGPUCallLowering` class ([`AMDGPUCallLowering.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AMDGPU/AMDGPUCallLowering.cpp)) dispatches to different argument lowering implementations based on the calling convention.

Cross-references: The Clang HIP compilation driver is covered in [Chapter 49 — Clang as a HIP Compiler](../part-07-clang-multilang/ch49-clang-as-hip-compiler.md). The MLIR AMDGPU dialect and ROCDL lowering path are in [Chapter 147](../part-20-in-tree-dialects/ch147-gpu-dialect.md). The SPIR-V backend used by ROCm OpenCL is the next chapter, [Chapter 104](../part-15-targets/ch104-the-spirv-backend.md).

---

## 103.10 The AMDGPU Pass Pipeline in Detail

### 103.10.1 GCN-Specific Machine Passes

The AMDGPU backend adds a large number of machine-level passes beyond the standard LLVM pipeline. Key passes in order:

```
Pre-RA Machine Passes:
  AMDGPUPerfHintAnalysis         ← analyze memory access patterns for hints
  SIAnnotateControlFlow          ← structurize CFG for EXEC mask management
  SIWholeQuadMode                ← insert WQM for texture derivative quads
  SILowerI1Copies                ← lower i1 copies to VCC-based sequences
  SIShrinkInstructions           ← convert VOP3 → VOP2 where possible
  SIFoldOperands                 ← fold literals/SGPRs into VOP src0

Register Allocation:
  GCNRegisterAlloc               ← VReg/SReg separate allocation
  SIFixSGPRCopies                ← fix illegal VGPR→SGPR copies post-RA
  SILowerCopies                  ← lower COPY instrs to hardware moves

Post-RA Passes:
  SIInsertWaitcnts               ← insert s_waitcnt for memory ordering
  GCNNSAReassign                 ← fix NSA image instruction VGPR layout
  SIPreEmitPeephole              ← late peepholes: s_cbranch_vccz optimization
  BranchRelaxation               ← handle out-of-range branches
  SIInsertHardClauses            ← insert s_clause hints for pipelined loads
```

### 103.10.2 SIAnnotateControlFlow

`SIAnnotateControlFlow` is the pass that converts LLVM's structured control flow (with convergent semantics) into EXEC-mask management code. It inserts calls to special AMDGPU intrinsics that become EXEC-modifying instructions:

```llvm
; After SIAnnotateControlFlow:
; (intrinsics that get lowered to s_and_saveexec / s_or_b64 / s_xor_b64)
declare { i1, i64 } @llvm.amdgcn.if.i64(i1 %cond)      ; if: save EXEC, apply cond
declare i64 @llvm.amdgcn.else.i64(i64 %saved)           ; else: flip to false-branch lanes
declare void @llvm.amdgcn.end.cf.i64(i64 %saved)        ; end: restore EXEC
declare { i1, i64 } @llvm.amdgcn.loop.i64(i64 %saved)  ; loop back-edge
```

These intrinsics lower to the `s_and_saveexec_b64`, `s_xor_b64`, `s_or_b64 exec` sequences shown in the EXEC mask diagram in section 103.7.1b.

### 103.10.3 GCN Register Allocation

The AMDGPU backend uses a custom register allocation strategy that handles the VGPR/SGPR dichotomy:

- **Divergence-aware register class selection**: Values marked divergent by `AMDGPUMachineDivergenceAnalysis` must use VGPR register classes; uniform values prefer SGPR classes
- **VGPR granularity**: VGPRs are allocated in groups of 4 on gfx9 and 8 on gfx10+ for occupancy alignment
- **SGPR pair requirements**: 64-bit values require even-aligned SGPR pairs (`s[0:1]`, `s[2:3]`, not `s[1:2]`)
- **VCC and EXEC**: These special registers are handled separately — VCC (`s[VCC_LO:VCC_HI]`) is the implicit output of VOPC comparison instructions and implicit input of `v_addc_co_u32`

The `GCNRegPressureAnalyzer` estimates VGPR and SGPR pressure during scheduling to avoid unnecessary pressure spikes that would reduce occupancy.

---

## Chapter Summary

- The AMDGPU backend covers GCN (wave64, MFMA) and RDNA (wave32, WMMA) architectures via `GCNTargetMachine` and `GCNSubtarget` parameterized by feature bits for ISA generation.
- AMDGPU defines eight address spaces; `addrspace(1)` is global DRAM, `addrspace(3)` is LDS (shared memory), `addrspace(5)` is private/scratch. `AMDGPUInferAddressSpaces` specializes generic flat pointers to generate more efficient load instructions.
- Two register files — VGPRs (divergent per-lane) and SGPRs (uniform per-wavefront) — enforce a fundamental invariant: SGPR writes from VGPR results require `v_readfirstlane_b32`. AGPRs on gfx908+ accumulate MFMA results separately from the VGPR file.
- The HSA kernel descriptor is a 64-byte structure in ELF `.rodata` that encodes VGPR/SGPR/LDS usage for occupancy calculation; it is emitted via `.amdhsa_kernel` assembler directives.
- Divergent control flow is implemented via EXEC mask manipulation: `s_and_saveexec_b32`, `s_xor_b32`, `s_or_b32` sequences generated by `SIAnnotateControlFlow`.
- MFMA instructions (gfx908+) operate on AGPR register groups; WMMA instructions (gfx1100+) target VGPRs directly. Both require precise hazard NOP insertions (`s_nop`) after matrix operations.
- Correctness passes — `SIFixSGPRCopies`, `SILowerI1Copies`, `SIInsertWaitcnts` — enforce SGPR/VGPR legality and memory ordering invariants that the ISA cannot express in instruction encoding alone.
- The ROCm compilation path produces HSA-conformant ELF objects consumed by `amdhip64.so` / `hsa-runtime64.so`; `llvm-readelf --notes` and `llvm-objdump --arch=amdgcn` are primary inspection tools.


---

@copyright jreuben11
