# Chapter 96 — The AArch64 Backend

*Part XV — Targets*

AArch64 (the 64-bit execution state of the ARMv8-A and later architectures) has become the dominant RISC ISA of the 2020s, powering Apple Silicon, AWS Graviton, Ampere Altra, and an expanding share of Android and Windows devices. The LLVM AArch64 backend is one of the most actively developed targets in the tree, supporting the full spectrum from server-class Neoverse cores to mobile Cortex-A cores, security features including Pointer Authentication and Branch Target Identification, and a scalable vector hierarchy from NEON (128-bit) through SVE/SVE2 to the SME matrix engine. This chapter covers the backend's register model, calling conventions, vector instruction selection, security extensions, and the GlobalISel-based lowering pipeline that AArch64 employs. Reading this chapter will equip you to diagnose codegen issues on AArch64, write idiomatic target-level lowering code, and understand the interaction between architecture security features and the compiler.

---

## 96.1 AAPCS64 Calling Convention

### 96.1.1 Register Assignment Rules

The Procedure Call Standard for the Arm 64-bit Architecture (AAPCS64) defines:

- **Integer arguments**: x0–x7 (first 8 words); additional arguments on stack
- **FP/SIMD arguments**: v0–v7 (first 8 scalars or vectors, any precision)
- **Integer return**: x0 (64-bit), x0+x1 (128-bit or large struct in regs)
- **FP return**: v0 (scalar), v0–v3 (HFA/HVA of up to 4 elements)
- **Callee-saved GPRs**: x19–x28, x29 (frame pointer), x30 (link register)
- **Callee-saved FP**: v8–v15 (only the low 64-bit halves are callee-saved)
- **Caller-saved**: x0–x18, v0–v7, v16–v31, SP temporary scratch

The platform register x18 is reserved as the shadow call stack pointer on Android and certain system code; user application code may use it freely on Linux.

### 96.1.2 Homogeneous Floating-Point and Vector Aggregates

Homogeneous Floating-Point Aggregates (HFAs) of 2–4 `float`, `double`, or `__fp16` elements are passed in consecutive FP registers, entirely bypassing the integer path. Homogeneous Vector Aggregates (HVAs) of 2–4 `float32x4_t` (or similar NEON vector) elements follow the same rule using v-registers. LLVM implements this classification in `AArch64ISelLowering.cpp`'s `analyzeCallOperands()` and `classifyArgumentType()`.

```llvm
; HFA: struct { float x, y, z, w; } passed in s0,s1,s2,s3
define float @hfa_test([4 x float] %v) {
  %x = extractvalue [4 x float] %v, 0
  ret float %x
}
; Compiles to: fmov s0, s0  (trivial — s0 is already x)
```

### 96.1.3 Stack Alignment and Varargs

The stack must be 16-byte aligned at function call boundaries. Variadic functions place all arguments (including FP args) in integer or stack slots from x0, bypassing the FP register path. The `va_list` type on AArch64 is a struct containing `__stack`, `__gr_top`, `__vr_top`, `__gr_offs`, `__vr_offs` fields for walking both integer and FP argument areas.

---

## 96.2 Register Model: XN/WN GPR Pairs

### 96.2.1 General-Purpose Registers

AArch64's 31 general-purpose registers appear as 64-bit Xn or 32-bit Wn views. Writes to Wn zero-extend to Xn — a key difference from x86's 8/16-bit sub-register partial-write behavior:

```tablegen
// AArch64RegisterInfo.td
def W0 : AArch64Reg<0, "w0", []>;
def X0 : AArch64Reg<0, "x0", [W0]>;
def GPR32 : RegisterClass<"AArch64", [i32], 32,
    (add W0, W1, W2, W3, W4, W5, W6, W7,
         W8, W9, W10, W11, W12, W13, W14, W15,
         W16, W17, W18, W19, W20, W21, W22, W23,
         W24, W25, W26, W27, W28, W29, W30, WZR)>;
def GPR64 : RegisterClass<"AArch64", [i64], 64,
    (add X0, X1, ..., X28, FP, LR, SP, XZR)>;
```

`XZR` and `WZR` are hardwired-zero registers not allocatable by the register allocator but usable in instruction operands to discard results. `SP` (stack pointer) and `FP` (X29) are tracked separately through `AArch64RegisterInfo::getFrameRegister()`.

### 96.2.2 Floating-Point and SIMD Registers

The 32 V-registers support multiple element views:

| Width | Letter | Access form | Typical use |
|-------|--------|-------------|-------------|
| 8-bit | B | Bn | Byte scalar |
| 16-bit | H | Hn | Half-precision scalar |
| 32-bit | S | Sn | Single-precision scalar |
| 64-bit | D | Dn | Double-precision scalar |
| 128-bit | Q/V | Qn, Vn.16b, Vn.8h, Vn.4s, Vn.2d | NEON vectors |

```tablegen
def FPR8  : RegisterClass<"AArch64", [i8],   8,  (add B0, B1, ..., B31)>;
def FPR32 : RegisterClass<"AArch64", [f32],  32, (add S0, S1, ..., S31)>;
def FPR64 : RegisterClass<"AArch64", [f64],  64, (add D0, D1, ..., D31)>;
def FPR128 : RegisterClass<"AArch64", [f128, v16i8, v8i16, v4i32, v2i64,
                                        v4f32, v2f64], 128,
    (add Q0, Q1, ..., Q31)>;
```

---

## 96.3 NEON: 128-bit Advanced SIMD

### 96.3.1 Element Types and Operations

NEON operates on V registers viewed as fixed-length 128-bit vectors. Each NEON register can be interpreted as:

| View | LLVM type | Mnemonic suffix | Example instruction |
|------|-----------|-----------------|---------------------|
| 16 × i8 | v16i8 | .16b | `add v0.16b, v1.16b, v2.16b` |
| 8 × i16 | v8i16 | .8h | `sadd v0.8h, v1.8h, v2.8h` |
| 4 × i32 | v4i32 | .4s | `mul v0.4s, v1.4s, v2.4s` |
| 2 × i64 | v2i64 | .2d | `add v0.2d, v1.2d, v2.2d` |
| 4 × f32 | v4f32 | .4s | `fadd v0.4s, v1.4s, v2.4s` |
| 2 × f64 | v2f64 | .2d | `fadd v0.2d, v1.2d, v2.2d` |

### 96.3.2 NEON Instruction Selection Patterns

LLVM's `AArch64InstrNEON.td` contains patterns for all major NEON operations. The DAG combiner in `AArch64ISelLowering.cpp` performs NEON-specific combines, including fused multiply-add recognition:

```tablegen
// FMLA: fused multiply-add vector accumulate
def : Pat<(v4f32 (fadd (fmul V128:$Rn, V128:$Rm), V128:$Ra)),
          (FMLAv4f32 V128:$Ra, V128:$Rn, V128:$Rm)>;

// FMLS: fused multiply-subtract
def : Pat<(v4f32 (fsub V128:$Ra, (fmul V128:$Rn, V128:$Rm))),
          (FMLSv4f32 V128:$Ra, V128:$Rn, V128:$Rm)>;
```

### 96.3.3 Shuffle Lowering

NEON shuffle lowering maps `shufflevector` to:
- `ZIP1`/`ZIP2`: interleave lower/upper halves of two vectors
- `UZP1`/`UZP2`: deinterleave even/odd elements
- `TRN1`/`TRN2`: transpose element pairs
- `TBL1`/`TBL2`: table lookup (general byte-level shuffle)
- `REV16`/`REV32`/`REV64`: element reversals

`AArch64ISelLowering::LowerVECTOR_SHUFFLE()` contains the cascade logic that selects the most specific (and cheapest) shuffle instruction for each mask pattern.

### 96.3.4 Dot Products and ML Instructions

Dot-product instructions from FEAT_DotProd provide 4×i8→i32 accumulation:

```cpp
// int8 SDOT: 4 signed 8-bit multiplies, sum into 32-bit accumulator
// Intrinsic: llvm.aarch64.neon.sdot
// Maps to: SDOT <Vd>.4S, <Vn>.16B, <Vm>.16B
__attribute__((target("dotprod")))
int32x4_t dot_product(int32x4_t acc, int8x16_t a, int8x16_t b) {
    return vdotq_s32(acc, a, b);
}
// Unsigned: UDOT (vusdotq_u32)
// Mixed: USDOT (vusdotq_s32 — unsigned src, signed weights)
```

FEAT_I8MM (int8 matrix multiply) adds `SMMLA`/`UMMLA`/`USMMLA` which accumulate 2×8 × 8×2 int8 matrix products, critical for transformer model inference. LLVM exposes these through `llvm.aarch64.neon.smmla`, etc.

### 96.3.5 NEON Intrinsics API

The `arm_neon.h` header provides typed C intrinsics:

```c
#include <arm_neon.h>

// Load four 32-bit floats into a NEON register
float32x4_t load_quad(const float* p) {
    return vld1q_f32(p);   // VLD1.32 {v0.4s}, [x0]
}

// Multiply-accumulate: accum = a * b + c
float32x4_t mac_quad(float32x4_t a, float32x4_t b, float32x4_t c) {
    return vfmaq_f32(c, a, b);  // FMLA v0.4s, v1.4s, v2.4s
}

// Transpose matrix (4×4 float)
void transpose_4x4(float32x4_t rows[4]) {
    float32x4x2_t r01 = vtrnq_f32(rows[0], rows[1]);
    float32x4x2_t r23 = vtrnq_f32(rows[2], rows[3]);
    // ZIP to complete transpose
    rows[0] = vcombine_f32(vget_low_f32(r01.val[0]), vget_low_f32(r23.val[0]));
    rows[1] = vcombine_f32(vget_low_f32(r01.val[1]), vget_low_f32(r23.val[1]));
    rows[2] = vcombine_f32(vget_high_f32(r01.val[0]), vget_high_f32(r23.val[0]));
    rows[3] = vcombine_f32(vget_high_f32(r01.val[1]), vget_high_f32(r23.val[1]));
}
```

---

## 96.4 SVE and SVE2: Scalable Vector Extensions

### 96.4.1 Architecture Overview

SVE (Scalable Vector Extension, ARMv8.2-A+) introduces length-agnostic scalable vectors. The hardware implements any VLEN that is a power-of-two multiple of 128 bits (128–2048 bits). Key registers:

- **Z registers** (Z0–Z31): scalable-length data vectors (`<vscale x N x T>`)
- **P registers** (P0–P15): per-bit predicate registers (one bit per byte lane)
- **FFR** (First-Fault Register): records elements that would have faulted in non-faulting loads

LLVM represents SVE types as `<vscale x N x T>` where `vscale = svcntb() / sizeof(T_in_bytes)`:

```llvm
; SVE i32 vector — runtime length = vscale*4 elements
; On a 256-bit SVE implementation: 2 * 4 = 8 elements
%v = alloca <vscale x 4 x i32>

; Query vscale at compile time via intrinsic
%vs = call i64 @llvm.vscale.i64()  ; = svcntd() on SVE2 = VL/64
```

### 96.4.2 TableGen Patterns for SVE Instructions

```tablegen
// SVE integer add with predicate (merge semantics)
def ADD_ZPmZ_S : sve_int_bin_pred_arit_0<0b10, "add",
    AArch64add_m1, ZPR32, PPR_3b>;

// Scalable vector unit-stride load LD1W
def LD1W_IMM : sve_mem_cld_si<0b1010, "ld1w", ZPR32, GPR64sp>;

// SVE whilelt: produce predicate register for loop control
// P0.S = {i < n for i in 0..VL-1}
def WHILELT_PWW_S : sve_int_while_rr<0b0, 0b10, 0b10, "whilelt",
    int_aarch64_sve_whilelt_nxv4i1>;
```

### 96.4.3 SVE Loop Vectorization

LLVM's Loop Vectorizer generates SVE code when targeting an SVE-capable core. The canonical SVE loop structure:

```asm
; Vectorized add loop: c[i] = a[i] + b[i]
    mov     x8, #0             ; i = 0
.loop:
    whilelt p0.s, w8, w3       ; p0 = {j < n : j in [i, i+VL)}
    ld1w    {z0.s}, p0/z, [x0, x8, lsl #2]  ; load a[i..i+VL)
    ld1w    {z1.s}, p0/z, [x1, x8, lsl #2]  ; load b[i..i+VL)
    add     z0.s, z0.s, z1.s             ; add
    st1w    {z0.s}, p0, [x2, x8, lsl #2] ; store c[i..i+VL)
    incd    x8                            ; i += VL/4 (elements)
    b.mi    .loop                        ; loop if not last
```

### 96.4.4 Gather/Scatter Operations

SVE supports sophisticated gather/scatter addressing:

```llvm
; Gather load: result[i] = *(base + offsets[i])
%result = call <vscale x 4 x float>
    @llvm.aarch64.sve.ld1.gather.nxv4f32(
        <vscale x 4 x i1> %pred,
        float* %base,
        <vscale x 4 x i32> %offsets)

; Scatter store: *(base + indices[i]) = values[i]
call void @llvm.aarch64.sve.st1.scatter.nxv4f32(
    <vscale x 4 x float> %values,
    <vscale x 4 x i1> %pred,
    float* %base,
    <vscale x 4 x i32> %indices)
```

These lower to `LD1W` with `[Xbase, Zoffset.s, sxtw #2]` addressing (vector-plus-vector with extension and shift).

### 96.4.5 SVE2 Additional Operations

SVE2 (ARMv9-A, all ARMv9 cores) extends SVE with:
- `HISTCNT`/`HISTSEG`: histogram accumulation for ML feature bucketization
- `MATCH`/`NMATCH`: string pattern matching across vector elements
- `SVDOT`/`SMMLA` equivalents: widening integer dot products
- Complex arithmetic: `FCMLA`/`FCADD` (complex multiply/add) applied per-element
- Narrowing bitwise operations: `RSHRNB`, `RSHRNT` for packed 8/16-bit results

---

## 96.5 SME: Scalable Matrix Extension

### 96.5.1 ZA Tile Architecture

SME (ARMv9.2-A) adds a 2D tile register file (`ZA`). The ZA array contains SVLB × SVLB bytes (where SVLB = VLENB for the streaming SVE mode). ZA is subdivided into:
- `ZA[i]` — complete rows
- `ZAB`, `ZAH`, `ZAS`, `ZAD` — byte/half/single/double-element horizontal/vertical slices

Key instructions:
- `MOPA`/`MOPS`: outer-product accumulate into ZA tiles
- `LD1ROW`/`LD1ROH`/`LD1ROS`/`LD1ROD`: load tile row/slice
- `ST1B`/`ST1H`/`ST1S`/`ST1D`: store from tile
- `SMSTART ZA`/`SMSTOP ZA`: enable/disable ZA storage

### 96.5.2 Streaming SVE Mode

SME introduces a "streaming SVE mode" where the VLEN may differ from non-streaming mode. Functions that use streaming mode are marked with the `__arm_streaming` attribute; the compiler emits `SMSTART SM`/`SMSTOP SM` on entry/exit and handles the incompatibility between streaming and non-streaming SVE code. LLVM models this via the `ArmSMEABI` and function attribute `aarch64_streaming`.

### 96.5.3 ZA ABI

ZA state is "lazy": it is saved/restored through a runtime mechanism (`__arm_tpidr2_save`/`__arm_tpidr2_restore`) rather than explicit instructions in the function prologue. LLVM generates the appropriate save/restore calls when a function uses ZA and calls another function that may clobber it.

---

## 96.6 Pointer Authentication (PAuth)

### 96.6.1 PAC Architecture

ARMv8.3-A Pointer Authentication uses spare bits (56–63 under 4-level paging with TBI) to embed a cryptographic MAC. The AArch64 CPU has four signing key pairs (IA, IB, DA, DB) plus a generic key (GA). Signing uses a 3-input QARMA cipher:

```
PAC = sign(address, key, modifier)
Signed_ptr = (PAC << 56) | address
```

| Instruction | Action | Key |
|-------------|--------|-----|
| `PACIA X0, X1` | Sign X0 using IA key, modifier X1 | IA |
| `PACIB X0, X1` | Sign X0 using IB key, modifier X1 | IB |
| `AUTIA X0, X1` | Authenticate X0 against IA | IA |
| `AUTIB X0, X1` | Authenticate X0 against IB | IB |
| `PACDZA X0` | Sign X0 with IA key, modifier=0 | IA |
| `RETAA` | Authenticate LR with IA, return | IA |
| `BLRAAZ X0` | Auth X0 with IA, zero mod, branch-link | IA |
| `XPACLRI` | Strip PAC bits from LR | – |

### 96.6.2 LLVM PAuth Instrumentation

With `-mbranch-protection=pac-ret`:

```asm
callee:
    paciasp          // sign LR: LR = PAC(LR, IA, SP)
    stp  x29, x30, [sp, #-16]!
    mov  x29, sp
    ...              // function body
    ldp  x29, x30, [sp], #16
    autiasp          // authenticate: verify LR against IA/SP
    ret              // abort if mismatch (BTYPE fault)
```

The `AArch64PointerAuth.cpp` pass inserts PAC/AUT instructions during frame lowering. `-mbranch-protection=standard` enables both pac-ret and bti together.

### 96.6.3 PAuth ABI for Function Pointers

The LLVM PAuth ABI specification assigns schema identifiers to function pointer signing. When a program signs a function pointer with schema 0 (the default IA+discriminator scheme), the runtime, dynamic linker, and compiler must all use the same signing policy. LLVM encodes the schema in LLVM IR via the `ptrauth` constant expression and the `@llvm.ptrauth.sign` / `@llvm.ptrauth.auth` intrinsics.

---

## 96.7 Branch Target Identification (BTI)

### 96.7.1 BTI Landing Pads

BTI (ARMv8.5-A) adds hardware CFI for indirect branches. When a processor flag enables BTI enforcement, any indirect branch to an address that does not begin with a `BTI` instruction causes a fault.

```asm
foo:
    bti    c         ; valid target for BLR (indirect call)
    stp    x29, x30, [sp, #-16]!

jumptable_target:
    bti    j         ; valid target for BR (indirect jump)
    ldr    x0, [x1]
```

BTI variants:
- `BTI C`: landing pad for indirect calls (`BLR`, `BLRAAZ`, etc.)
- `BTI J`: landing pad for indirect jumps (`BR`, `BRAAZ`, etc.)
- `BTI JC`: landing pad for both

### 96.7.2 LLVM BTI Generation

LLVM's `AArch64BranchTargets.cpp` pass inserts `BTI` instructions:
- At function entry points (BTI C)
- At exception handling landing pads
- At `address_taken` labels (jump table targets, BTI J)
- Preceding explicit `__attribute__((btilanding))` markers

The ELF `PT_GNU_PROPERTY` note `GNU_PROPERTY_AARCH64_FEATURE_1_BTI` is set in object files that have all function entries marked with BTI, allowing the dynamic linker to enforce BTI only when all loaded objects support it.

---

## 96.8 LSE Atomics

### 96.8.1 LL/SC vs LSE

Without LSE, atomic operations require load-linked/store-conditional (LL/SC) loops:

```asm
; Without LSE: compare-and-swap loop
.retry:
    ldaxr  w0, [x1]          ; exclusive load (acquire)
    cmp    w0, w2             ; compare
    b.ne   .fail
    stlxr  w3, w4, [x1]      ; exclusive store (release)
    cbnz   w3, .retry        ; retry if store failed
.fail:
```

### 96.8.2 LSE Single-Instruction Atomics

ARMv8.1-A Large System Extensions provide true single-instruction atomics:

| LSE Instruction | Operation |
|-----------------|-----------|
| `CAS Ws, Wt, [Xn]` | Compare-and-swap |
| `CASA Ws, Wt, [Xn]` | CAS with acquire |
| `CASAL Ws, Wt, [Xn]` | CAS with acquire-release |
| `LDADD Ws, Wt, [Xn]` | Atomic add, return old |
| `LDSET Ws, Wt, [Xn]` | Atomic OR |
| `LDCLR Ws, Wt, [Xn]` | Atomic AND-NOT |
| `SWP Ws, Wt, [Xn]` | Atomic exchange |
| `STADD Ws, [Xn]` | Store-add (no return) |

LLVM lowers `cmpxchg` and `atomicrmw` to LSE when `+lse` is set:

```llvm
; LLVM IR
%old = atomicrmw add i32* %ptr, i32 %val seq_cst
; → LDADDAL W<val>, W<old>, [X<ptr>]   (LSE, seq_cst)
; → LDADD   W<val>, W<old>, [X<ptr>]   (LSE, release only)
; → LL/SC loop                          (no LSE)
```

The `AArch64AtomicExpand.cpp` pass performs the LSE lowering after legalization. For lock-free data structures, LSE provides substantial performance improvements on contended workloads by eliminating the retry overhead of LL/SC loops.

---

## 96.9 Memory Tagging Extension (MTE)

### 96.9.1 Tag Architecture

MTE (ARMv8.5-A) embeds 4-bit allocation tags in 16-byte-granule memory regions and stores matching "logical tags" in the top bits of the virtual address (using AArch64 TBI, top-byte ignore). On a tag mismatch at load or store, the CPU generates a `Tag Check Fault`.

### 96.9.2 MTE Instructions

```asm
; Allocate random tag for a memory region
IRG  X0, X1, X2       ; X0 = X1 with random tag bits (XZR excludes tags in X2)

; Store tag for 16-byte granule
STG  X0, [X1]          ; store tag of X0 to [X1] granule

; Store two granules at once
ST2G X0, [X1]

; Load and check a tag
LDG  X0, X0, [X1]      ; X0 = address X0 with tag from [X1]

; Arithmetic on tagged addresses
ADDG X0, X1, #16, #0   ; X0 = X1 + 16, tag unchanged
SUBG X0, X1, #0,  #1   ; subtract 1 from tag
```

### 96.9.3 MTE in LLVM

LLVM's MTE support:
- `-fsanitize=memtag` emits tag-annotated memory operations for sanitizer use (see Chapter 111)
- The `AArch64MTE.td` file defines instruction encodings for the tag-manipulation instructions
- The `AArch64MemTagInstrInfo.td` provides the IRG/STG/LDG/ST2G patterns

---

## 96.10 Apple Silicon Specifics

### 96.10.1 AMX and Neural Engine

Apple M-series processors include the AMX (Apple Matrix Coprocessor) accessible via undocumented `EXTRX`/`EXTRY`/`FMA`/`MATFP`/`MATVEC` instruction sequences. AMX operates on large 512×8-element or 512×4-element tile registers. LLVM does not officially expose AMX; high-level frameworks (Core ML, Metal Performance Shaders) use it through compiler plugin extensions.

### 96.10.2 commpage

The kernel maps a read-only 4 KB page at a fixed virtual address (`_COMM_PAGE_START`, typically 0xFFFFF000 on 32-bit, `0x0000000FFFFFC000` on 64-bit AArch64 macOS). It provides:

```c
// commpage fields
uint64_t _COMM_PAGE_TIME_DATA_START;   // mach_absolute_time
uint8_t  _COMM_PAGE_NCPUS;            // CPU count
uint8_t  _COMM_PAGE_ACTIVE_CPUS;
uint64_t _COMM_PAGE_PTHREAD_TSD_BASE; // TLS base
```

Access via the commpage avoids syscall overhead for `gettimeofday()`, `clock_gettime(CLOCK_MONOTONIC)`, and `pthread_self()`.

### 96.10.3 CPU-Specific Scheduling Models

LLVM includes `AArch64SchedAppleM*.td` scheduling models for Apple cores:

```tablegen
// AArch64SchedAppleM1.td
def AppleM1Model : SchedMachineModel {
  let IssueWidth = 8;
  let MicroOpBufferSize = 0;  // out-of-order, 630-entry ROB
  let LoadLatency = 4;
  let MispredictPenalty = 11;
}
// Execution units: P-cluster (Firestorm): 8-wide
// E-cluster (Icestorm): 4-wide, shared AluExUnits
```

```bash
# Target Apple M3 specifically
clang --target=arm64-apple-macos14.0 \
    -mcpu=apple-m3 -O3 \
    foo.c -o foo_m3
```

---

## 96.11 AArch64 Outliner

The Machine Outliner (Chapter 92) is particularly profitable on AArch64 due to the fixed 4-byte instruction size, making outlining worthwhile at smaller granularities than on x86. AArch64-specific handling in `AArch64InstrInfo.cpp`:

### 96.11.1 LR Preservation in Outlined Functions

When an outlined function itself calls another function, the link register (LR/X30) must be saved. LLVM detects this and inserts a stack slot:

```asm
_OUTLINED_FUNCTION_0:
    stp  x30, xzr, [sp, #-16]!  ; save LR if outlined fn makes calls
    ; ... outlined sequence ...
    ldp  x30, xzr, [sp], #16
    ret
```

### 96.11.2 BTI and PAuth Interaction

When outlining is combined with BTI (`bti c`) or PAuth (`paciasp`), the outliner must ensure:
1. Each outlined function begins with `BTI C` if branch-protection is active
2. Functions containing `paciasp`/`autiasp` are outlined as complete units (prologue + epilogue together)
3. The `AArch64InstrInfo::isFunctionSafeToOutlineFrom()` check prevents outlining across PAuth signing boundaries

---

## 96.12 Register Bank Selection and GlobalISel

### 96.12.1 GlobalISel Pipeline on AArch64

AArch64 is a first-class GlobalISel target, enabled by default for many configurations in LLVM 22.1.x. The pipeline:

1. **IRTranslator** → generic MachineIR: `G_ADD`, `G_LOAD`, `G_FCMP`, `G_INTRINSIC`
2. **Legalizer** (`AArch64LegalizerInfo`) → ensures all types and ops are legal for the target
3. **RegBankSelect** (`AArch64RegisterBankInfo`) → assigns each vreg to GPR or FPR register bank
4. **InstructionSelect** (`AArch64InstructionSelector`) → maps generic ops to real instructions

### 96.12.2 Register Bank Decisions

Register bank selection is critical: an `i64` on the GPR bank maps to Xn; the same value on the FPR bank maps to Dn (scalar double). Misassignment forces a cross-bank `FMOV` copy. `AArch64RegisterBankInfo.cpp` contains a cost model:

```cpp
// AArch64RegisterBankInfo.cpp
const RegisterBankInfo::InstructionMapping &
AArch64RegisterBankInfo::getInstrMapping(const MachineInstr &MI) const {
  // FP ops always go to FPR bank
  // Integer ops go to GPR bank
  // Load/store: bank of the value type
  // G_INTRINSIC: determined by intrinsic ID
}
```

### 96.12.3 GlobalISel Legalization Examples

```cpp
// AArch64LegalizerInfo.cpp
// Narrow 128-bit integer ops to two 64-bit ops
getActionDefinitionsBuilder(G_ADD)
    .legalFor({s32, s64, v2s32, v4s32, v2s64})
    .clampScalar(0, s32, s64)
    .widenScalarToNextPow2(0);

// f16 arithmetic: legal with +fp16, otherwise widen to f32
getActionDefinitionsBuilder(G_FADD)
    .legalFor({s16, s32, s64, v4s16, v8s16, v2s32, v4s32, v2s64})
    .narrowScalarIf(
        [=](const LegalityQuery &Q) {
            return Q.Types[0] == LLT::scalar(16) && !ST.hasFullFP16();
        }, changeTo(0, LLT::scalar(32)));
```

---

## 96.13 AArch64 Scheduling and Target Info

### 96.13.1 Available Scheduling Models

LLVM ships scheduling models for:

| Core | Model file |
|------|-----------|
| Apple M1 (Firestorm/Icestorm) | `AArch64SchedAppleM1.td` |
| Apple M2 (Avalanche/Blizzard) | `AArch64SchedAppleM2.td` |
| Cortex-A55 | `AArch64SchedA55.td` |
| Cortex-A510 | `AArch64SchedA510.td` |
| Cortex-A57 | `AArch64SchedA57.td` |
| Cortex-A72 | `AArch64SchedA72.td` |
| Neoverse N1 | `AArch64SchedNeoverseN1.td` |
| Neoverse N2 | `AArch64SchedNeoverseN2.td` |
| Neoverse V1 | `AArch64SchedNeoverseV1.td` |

### 96.13.2 Using llc for AArch64

```bash
# AArch64 with SVE2
/usr/lib/llvm-22/bin/llc -march=aarch64 \
    -mattr=+sve2,+sve2-bitperm \
    -mcpu=neoverse-n2 \
    input.ll -o output.s

# Apple M2 with all extensions
/usr/lib/llvm-22/bin/llc -march=aarch64 \
    -mcpu=apple-m2 \
    -mattr=+sve2,+i8mm,+bf16,+dotprod \
    input.ll -o output.s

# Check GlobalISel pipeline
/usr/lib/llvm-22/bin/llc -march=aarch64 \
    -global-isel \
    -print-after-all \
    input.ll -o /dev/null 2>&1 | grep "After.*RegBankSelect"

# Verify BTI output
clang --target=aarch64-linux-gnu \
    -mbranch-protection=standard \
    -O2 -S foo.c -o foo.s && grep -c bti foo.s
```

---

## Chapter Summary

- AAPCS64 passes 8 integer args in x0–x7 and 8 FP/SIMD args in v0–v7; Homogeneous Floating-Point Aggregates (HFAs) of 2–4 same-type FP elements bypass integer registers and land in consecutive S/D/Q registers; varargs place all args in integer registers.
- AArch64 has 31 GPRs (Xn/Wn dual views), 32 V-registers (Bn/Hn/Sn/Dn/Qn scalar views plus Vn.T vector views); XZR/WZR are hardwired zero, not allocatable.
- NEON provides fixed 128-bit SIMD; FMLA fusion, TBL-based shuffle lowering, SDOT/UDOT (4×i8→i32 dot), and SMMLA/UMMLA (int8 matrix multiply) are key patterns for ML workloads.
- SVE/SVE2 uses `<vscale x N x T>` LLVM types where vscale = hardware VLEN/128; gather/scatter maps to indexed LD1/ST1; predicates drive tail-loop elimination; SVE2 adds HISTCNT, MATCH, and complex number operations.
- SME adds a 2D ZA tile file for matrix outer-product accumulation; the streaming SVE mode may use a different VLEN; ZA state is preserved lazily through runtime save/restore rather than explicit prologue/epilogue instructions.
- Pointer Authentication (PAuth) signs return addresses and function pointers using the QARMA cipher; LLVM emits `PACIASP`/`AUTIASP` frame instrumentation under `-mbranch-protection=pac-ret`.
- BTI marks valid indirect branch targets with `BTI C`/`BTI J` landing-pad instructions; the ELF property note enables dynamic linker enforcement when all loaded objects support BTI.
- LSE atomics (`CAS`, `LDADD`, `SWP`, etc.) replace LL/SC loops when `+lse` is active, providing single-instruction atomic read-modify-write without retry overhead.
- MTE embeds 4-bit memory tags in address top-bytes and 16-byte memory granules, hardware-detecting use-after-free and buffer-overflow via tag mismatch faults.
- GlobalISel on AArch64 is the production path; it assigns vregs to GPR or FPR banks during RegBankSelect, avoiding unnecessary cross-bank FMOV copies, and provides full legalizer coverage for all ARMv8–ARMv9 types.


---

@copyright jreuben11
