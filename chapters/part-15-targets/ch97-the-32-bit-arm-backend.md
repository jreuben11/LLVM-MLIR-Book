# Chapter 97 — The 32-bit ARM Backend

*Part XV — Targets*

The 32-bit ARM backend in LLVM spans the widest ISA diversity of any supported target: A32 (the classic 32-bit ARM instruction set), Thumb (16-bit compressed), and Thumb-2 (mixed 16/32-bit) co-exist in a single binary and can interwork freely. The target also serves two architecturally distinct markets — application processors (ARMv7-A, Cortex-A7/A9/A15) running Linux and Android, and microcontrollers (ARMv7-M, Cortex-M0/M3/M4/M7) in bare-metal embedded systems. These profiles share the same LLVM target but have drastically different constraints: no NEON on Cortex-M, no MMU, no OS-level stack alignment guarantees, and often no hardware floating-point at all. This chapter covers LLVM's ARM backend from ISA interworking through VFP/NEON SIMD to embedded bare-metal constraints and the inline assembly interface, giving you the practical knowledge to generate, debug, and extend 32-bit ARM code in LLVM 22.1.x.

---

## Table of Contents

- [97.1 A32, Thumb, and Thumb-2 ISA Mix](#971-a32-thumb-and-thumb-2-isa-mix)
  - [97.1.1 A32 (Classic ARM)](#9711-a32-classic-arm)
  - [97.1.2 Thumb-1](#9712-thumb-1)
  - [97.1.3 Thumb-2](#9713-thumb-2)
  - [97.1.4 ISA Selection in ARMSubtarget](#9714-isa-selection-in-armsubtarget)
- [97.2 Thumb Interworking](#972-thumb-interworking)
  - [97.2.1 State Bit in the PC](#9721-state-bit-in-the-pc)
  - [97.2.2 Interworking Veneers](#9722-interworking-veneers)
  - [97.2.3 Function Pointer LSB](#9723-function-pointer-lsb)
- [97.3 AAPCS Calling Convention](#973-aapcs-calling-convention)
  - [97.3.1 Integer Arguments and Return](#9731-integer-arguments-and-return)
  - [97.3.2 FP ABI Variants](#9732-fp-abi-variants)
  - [97.3.3 Stack Alignment](#9733-stack-alignment)
- [97.4 VFP and NEON Coprocessors](#974-vfp-and-neon-coprocessors)
  - [97.4.1 VFP Register File](#9741-vfp-register-file)
  - [97.4.2 NEON Advanced SIMD for A-Profile](#9742-neon-advanced-simd-for-a-profile)
  - [97.4.3 NEON Intrinsics](#9743-neon-intrinsics)
- [97.5 ARMv7-A vs ARMv7-M (Cortex-M) Constraints](#975-armv7-a-vs-armv7-m-cortex-m-constraints)
  - [97.5.1 M-Profile Architecture Differences](#9751-m-profile-architecture-differences)
  - [97.5.2 Cortex-M0/M0+](#9752-cortex-m0m0)
  - [97.5.3 Cortex-M3 and M4](#9753-cortex-m3-and-m4)
  - [97.5.4 Cortex-M7](#9754-cortex-m7)
  - [97.5.5 NVIC and Exception Frame](#9755-nvic-and-exception-frame)
- [97.6 arm-none-eabi Bare-Metal Toolchain](#976-arm-none-eabi-bare-metal-toolchain)
  - [97.6.1 Startup and Linker Scripts](#9761-startup-and-linker-scripts)
  - [97.6.2 Semihosting and Retargeting](#9762-semihosting-and-retargeting)
  - [97.6.3 compiler-rt for Bare-Metal](#9763-compiler-rt-for-bare-metal)
- [97.7 Frame Pointer Chains](#977-frame-pointer-chains)
  - [97.7.1 R11 as Frame Pointer](#9771-r11-as-frame-pointer)
  - [97.7.2 Frame Omission](#9772-frame-omission)
- [97.8 Thumb-2 IT Blocks](#978-thumb-2-it-blocks)
  - [97.8.1 IT Block Encoding](#9781-it-block-encoding)
  - [97.8.2 ARMv8 Deprecation](#9782-armv8-deprecation)
- [97.9 ARM SelectionDAG and GlobalISel Status](#979-arm-selectiondag-and-globalisel-status)
  - [97.9.1 SelectionDAG Path](#9791-selectiondag-path)
  - [97.9.2 LLVM Peephole and Combines](#9792-llvm-peephole-and-combines)
  - [97.9.3 GlobalISel on ARM](#9793-globalisel-on-arm)
- [97.10 Inline Assembly Constraints](#9710-inline-assembly-constraints)
  - [97.10.1 Constraint Letters](#97101-constraint-letters)
  - [97.10.2 Inline Assembly Examples](#97102-inline-assembly-examples)
- [97.11 ARMv8-A AArch32 Mode](#9711-armv8-a-aarch32-mode)
  - [97.11.1 Differences from ARMv7-A](#97111-differences-from-armv7-a)
- [97.12 Scheduling Models](#9712-scheduling-models)
- [Armv8.1-M PACBTI: Pointer Authentication and Branch Target Identification for Cortex-M](#armv81-m-pacbti-pointer-authentication-and-branch-target-identification-for-cortex-m)
- [Chapter Summary](#chapter-summary)

---

## 97.1 A32, Thumb, and Thumb-2 ISA Mix

### 97.1.1 A32 (Classic ARM)

A32 instructions are uniformly 32 bits wide, with a 4-bit condition code field in bits 31:28. This makes every instruction implicitly predicated — a fundamental architectural capability that A32 exploits pervasively for branch-free code:

```asm
; A32: conditional subtract — executed only if condition Z=1
SUBEQ r0, r1, r2     ; r0 = r1 - r2 if Equal

; Shift-as-modifier: barrel shifter in every instruction
ADD   r0, r1, r2, LSL #3  ; r0 = r1 + (r2 << 3) in one instruction
LDR   r0, [r1, r2, ROR #4] ; load from r1 + (r2 rotate-right 4)
```

The barrel shifter integrated into every A32 ALU operation is what makes A32 so efficient for bit manipulation and address computation. LLVM exploits this through patterns that fold shift+operation into a single `ARM_MOVsi`/`ARM_ADDsi` node.

LLVM selects A32 when the target triple is `armv7-linux-gnueabi` or similar, without `+thumb-mode` in the feature string.

### 97.1.2 Thumb-1

Thumb-1 encodes a subset of ARM instructions in 16 bits. Key restrictions: only R0–R7 accessible in most instructions, 5-bit immediate fields only, no barrel-shifter on data ops, no conditional execution (only conditional `B`). Despite its limitations, Thumb-1 achieves roughly 65% of A32 code size.

LLVM uses Thumb-1 for bare-metal Cortex-M0 targets: `-mthumb -mcpu=cortex-m0`.

### 97.1.3 Thumb-2

Thumb-2 (introduced in ARMv6T2, dominant in ARMv7) extends Thumb with 32-bit instruction forms while retaining 16-bit density for the most common operations. A Thumb-2 function seamlessly mixes 2-byte and 4-byte instruction encodings:

```asm
; Thumb-2 mixed encoding
2000: f241 0034  movw    r0, #0x1034    ; 32-bit Thumb-2
2004: 2101       movs    r1, #1         ; 16-bit Thumb
2006: f7ff fffe  bl      <foo>          ; 32-bit Thumb-2 branch
200a: 4770       bx      lr             ; 16-bit Thumb
```

LLVM automatically selects 16-bit vs 32-bit forms based on register operands (low vs. high regs), immediate width, and whether the instruction needs to set flags.

```tablegen
// ARMInstrThumb2.td
// 32-bit Thumb-2 add with 12-bit immediate
def t2ADDri12 : T2I<(outs rGPR:$Rd), (ins GPR:$Rn, imm0_4095:$imm),
    IIC_iALUi, "addw", "\t$Rd, $Rn, $imm",
    [(set rGPR:$Rd, (add GPR:$Rn, imm0_4095:$imm))]>;

// 16-bit Thumb-1 add (restricted to low regs)
def tADDi3 : T1pI<(outs tGPR:$Rd), (ins tGPR:$Rn, imm0_7:$imm),
    IIC_iALUi, "adds", "\t$Rd, $Rn, $imm",
    [(set tGPR:$Rd, (add tGPR:$Rn, imm0_7:$imm))]>;
```

### 97.1.4 ISA Selection in ARMSubtarget

The ARM backend's `ARMSubtarget` class decides the active ISA mix:

```cpp
// ARMSubtarget.cpp
bool ARMSubtarget::isThumb1Only() const {
  return isThumb() && !hasThumb2();
}
bool ARMSubtarget::isThumb2() const {
  return isThumb() && hasThumb2();
}
bool ARMSubtarget::hasV7Ops() const {
  return ARMProcFamily >= CortexA5 || ARMArch >= ARMv7;
}
```

The `-mthumb`/`-marm` flags and `+thumb-mode` attribute control ISA selection, while `-mcpu` sets the processor family that determines available extensions.

---

## 97.2 Thumb Interworking

### 97.2.1 State Bit in the PC

ARM/Thumb execution state is determined by bit 0 of the program counter: 1 = Thumb, 0 = ARM. When the processor branches to an address with bit 0 set, it switches to Thumb mode; when it branches to an address with bit 0 clear, it switches to ARM mode.

The `BX` (Branch and eXchange) and `BLX` (Branch-Link-eXchange) instructions respect the LSB:

```asm
; ARM calling a Thumb function: function pointer has LSB set
LDR  r0, =thumb_func + 1  ; address with bit 0 set → Thumb
BLX  r0                    ; branch to Thumb code

; Thumb returning to ARM caller
BX   lr                    ; lr has bit 0 = 0 (from ARM AAPCS)
```

### 97.2.2 Interworking Veneers

When a branch target is too far for a direct instruction (BL has ±32 MB range), or when an ARM caller needs to call a Thumb function, the linker inserts a veneer:

```asm
; ARM-to-Thumb veneer (inserted by LLD between .text sections)
$a.thumb_fn_from_arm:
    ldr  ip, [pc, #0]  ; load address of Thumb function
    bx   ip            ; switch to Thumb and branch
    .word thumb_fn + 1 ; address with bit 0 set
```

For Thumb-to-ARM veneers, the linker uses a `MOVW/MOVT/BX` sequence on ARMv7:

```asm
$t.arm_fn_from_thumb:
    movw ip, #:lower16:arm_fn
    movt ip, #:upper16:arm_fn
    bx   ip
```

LLVM's `ARMConstantIslandPass` handles constant pool placement and long-range branch fixups within a single function.

### 97.2.3 Function Pointer LSB

Thumb function pointers have their LSB set (`ptr | 1`). The compiler emits this via `ARMTargetLowering::LowerGlobalAddress()` which applies a `+1` offset to the address of Thumb functions. At call sites using `BLX`, the processor automatically interprets the LSB as the state bit.

---

## 97.3 AAPCS Calling Convention

### 97.3.1 Integer Arguments and Return

The ARM AAPCS (for A32) specifies:

- **Integer arguments**: r0–r3 (first 4 words); 64-bit `long long` values align to even-register pairs (e.g., r0:r1 or r2:r3); overflow goes on stack
- **Integer return**: r0 (32-bit), r0+r1 (64-bit), r0+r1+r2+r3 (large struct in regs if ≤4 words)
- **Callee-saved**: r4–r11, r14 (LR) (by convention, not always saved if unused)
- **Scratch**: r0–r3, r12 (ip/intra-procedure-call scratch)
- **Stack pointer**: r13 (sp), 8-byte aligned at public API boundaries
- **Frame pointer**: r11 (A32), or r7 (Thumb-2, GCC/LLVM legacy convention)

### 97.3.2 FP ABI Variants

Three ABI variants handle floating-point differently:

| ABI | `-mfloat-abi` | FP args | Computation |
|-----|---------------|---------|-------------|
| soft | `soft` | Integer regs | SW emulation |
| softfp | `softfp` | Integer regs | HW VFP |
| hardfp | `hard` | VFP s/d regs | HW VFP |

With softfp (the default for most Linux distributions), `float` args pass in r0, and `double` args pass in r0:r1 (EABI) or r2:r3 (when preceded by a float). With hardfp, `float` args use s0–s15 and `double` args use d0–d7.

```bash
# Softfp (arm-linux-gnueabi)
clang --target=armv7-linux-gnueabi -mfloat-abi=softfp foo.c
# Hardfp (arm-linux-gnueabihf)
clang --target=armv7-linux-gnueabihf -mfloat-abi=hard foo.c
```

### 97.3.3 Stack Alignment

AAPCS requires 8-byte stack alignment at public function interfaces (call instructions), but individual functions may use 4-byte alignment internally. The `ARMFrameLowering` inserts alignment padding when necessary.

---

## 97.4 VFP and NEON Coprocessors

### 97.4.1 VFP Register File

VFP (Vector Floating-Point) provides a coprocessor register file:
- **VFPv2**: S0–S31 (32 single-precision, aliased as D0–D15 double-precision)
- **VFPv3**: extends to D0–D31 (32 double-precision registers; single S0–S31 alias lower half)
- **VFPv4**: adds fused multiply-accumulate (`VFMA`, `VFMS`)
- **VFPv5**: adds half-precision conversion (`VCVTB`, `VCVTT`)

```asm
; VFP single-precision arithmetic
VADD.F32  s0, s1, s2    ; s0 = s1 + s2
VMUL.F32  s4, s5, s6    ; s4 = s5 * s6
VFMA.F32  s0, s1, s2    ; s0 += s1 * s2 (VFPv4)

; VFP double-precision
VADD.F64  d0, d1, d2
VDIV.F64  d3, d4, d5

; Load/store
VLDR      s0, [r0]      ; load float from address in r0
VSTR      d1, [r1, #8]  ; store double at r1+8
VLDM      r0, {s0-s3}   ; load multiple
```

### 97.4.2 NEON Advanced SIMD for A-Profile

NEON reuses the VFP register file viewed as D registers (64-bit) or Q registers (128-bit = D-pair):
- D0–D31: 64-bit double-precision or 2×f32, 4×i16, 8×i8 vectors
- Q0–Q15: 128-bit quad vectors = D0:D1 through D30:D31

```tablegen
// ARMInstrNEON.td
// NEON 4×f32 add
def VADDfq : N3VQ<0, 0, 0b10, 0b1101, 0, IIC_VBIND, "vadd", "f32",
    v4f32, v4f32, fadd, 1>;

// NEON 16×i8 unsigned add
def VADDuv16i8 : N3VQ<1, 0, 0b00, 0b1000, 0, IIC_VBINiQ, "vadd", "i8",
    v16i8, v16i8, add, 1>;

// NEON multiply-accumulate
def VMLAfd : NI_VMLAfq<0, 1, IIC_VMACD, "vmla", "f32", v2f32, fmul, fadd>;
```

### 97.4.3 NEON Intrinsics

```c
#include <arm_neon.h>

// Load and add two float32x4 vectors
float32x4_t vec_add(const float* a, const float* b) {
    float32x4_t va = vld1q_f32(a);
    float32x4_t vb = vld1q_f32(b);
    return vaddq_f32(va, vb);  // VADD.F32 q0, q1, q2
}

// Dot product using NEON: sum of a[i]*b[i]
float dot_product(const float* a, const float* b, int n) {
    float32x4_t acc = vdupq_n_f32(0.0f);
    for (int i = 0; i < n; i += 4) {
        float32x4_t va = vld1q_f32(a + i);
        float32x4_t vb = vld1q_f32(b + i);
        acc = vmlaq_f32(acc, va, vb);  // VMLA.F32
    }
    // Horizontal sum
    float32x2_t sum2 = vadd_f32(vget_high_f32(acc), vget_low_f32(acc));
    return vget_lane_f32(vpadd_f32(sum2, sum2), 0);
}
```

---

## 97.5 ARMv7-A vs ARMv7-M (Cortex-M) Constraints

### 97.5.1 M-Profile Architecture Differences

Cortex-M (ARMv7-M) is architecturally distinct from ARMv7-A:

| Feature | ARMv7-A (Cortex-A) | ARMv7-M (Cortex-M) |
|---------|--------------------|--------------------|
| Instruction set | A32 + Thumb/Thumb-2 | Thumb-2 only (Thumb-1 compatible) |
| MMU | Yes (VMSA, virtual memory) | No (MPU, physical memory) |
| NEON | Optional (FPU models) | Not available |
| FPU | VFPv3/v4 + optional NEON | FPv4-SP (Cortex-M4) or FPv5 (M7) |
| Cache | L1/L2 typically present | Simple TCM or no cache |
| Exception model | IRQ/FIQ + supervisor mode | NVIC with tail-chaining |
| OS support | Full OS with MMU | RTOS/bare-metal only |

### 97.5.2 Cortex-M0/M0+

The smallest M-profile cores (Cortex-M0, M0+) implement ARMv6-M, which is Thumb-1 only (no Thumb-2 32-bit forms, no `CBZ`/`CBNZ`, no `IT` blocks). LLVM targets these with `-mcpu=cortex-m0` or `-march=thumbv6m-none-eabi`.

### 97.5.3 Cortex-M3 and M4

Cortex-M3 (ARMv7-M) adds full Thumb-2, DSP extension (`SSAT`, `USAT`, `SMLA*`, `SMLAL*`), and hardware divide (`SDIV`/`UDIV`). Cortex-M4 (ARMv7E-M) adds the optional FPv4-SP single-precision FPU and SIMD MAC instructions:

```bash
# Cortex-M4 with single-precision FPU and hard-float ABI
clang --target=thumbv7em-none-eabi \
    -mcpu=cortex-m4 \
    -mfpu=fpv4-sp-d16 \
    -mfloat-abi=hard \
    -fno-exceptions -fno-rtti \
    -nostdlib \
    foo.c -o foo.elf
```

### 97.5.4 Cortex-M7

Cortex-M7 (ARMv7E-M) implements a 6-stage dual-issue superscalar pipeline and supports FPv5 with optional double-precision (`-mfpu=fpv5-d16`). Its deeper pipeline makes scheduling models important: `ARMSchedCortexM7.td` in LLVM.

### 97.5.5 NVIC and Exception Frame

Cortex-M automatically pushes `{r0-r3, r12, lr, pc, xpsr}` onto the stack on exception entry and pops them on exit — no software frame push/pop required for IRQ handlers. LLVM generates IRQ handlers with `__attribute__((interrupt("IRQ")))` which emits the appropriate `PUSH` sequences if additional registers are used.

---

## 97.6 arm-none-eabi Bare-Metal Toolchain

### 97.6.1 Startup and Linker Scripts

A typical Cortex-M project requires a complete startup chain:

```c
// startup.c — minimal C startup
extern uint32_t _sdata, _edata, _sidata;  // data section boundaries
extern uint32_t _sbss, _ebss;

void Reset_Handler(void) __attribute__((naked));
void Reset_Handler(void) {
    // Copy .data from flash to SRAM
    uint32_t *src = &_sidata, *dst = &_sdata;
    while (dst < &_edata) *dst++ = *src++;
    // Zero .bss
    dst = &_sbss;
    while (dst < &_ebss) *dst++ = 0;
    // Call main
    main();
    for (;;);
}

// Interrupt vector table
__attribute__((section(".isr_vector")))
void (*const vector_table[])(void) = {
    (void*)&_estack,    // initial stack pointer
    Reset_Handler,       // reset handler
    NMI_Handler,
    HardFault_Handler,
    // ... other exception vectors
};
```

```ld
/* Linker script for STM32F407 */
MEMORY {
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K
    SRAM  (rwx) : ORIGIN = 0x20000000, LENGTH = 192K
}
SECTIONS {
    .isr_vector : { KEEP(*(.isr_vector)) } > FLASH
    .text : { *(.text*) *(.rodata*) } > FLASH
    _sidata = LOADADDR(.data);
    .data : { _sdata = .; *(.data*) _edata = .; } > SRAM AT> FLASH
    .bss  : { _sbss = .; *(.bss*) *(COMMON) _ebss = .; } > SRAM
    _estack = ORIGIN(SRAM) + LENGTH(SRAM);
}
```

### 97.6.2 Semihosting and Retargeting

Semihosting (`--specs=rdimon.specs`) routes `printf`/`fopen` calls through the debugger connection via SVC calls. For production embedded code, retargeting replaces the weak `_write`/`_read` stubs:

```c
// Retarget _write to UART
int _write(int fd, char* buf, int len) {
    for (int i = 0; i < len; i++) {
        while (!(USART1->SR & USART_SR_TXE));
        USART1->DR = buf[i];
    }
    return len;
}
```

### 97.6.3 compiler-rt for Bare-Metal

`compiler-rt` provides software implementations of operations not available on bare-metal ARM:

```bash
# Find available compiler-rt builtins for ARM
ls /usr/lib/llvm-22/lib/clang/22/lib/linux/ | grep arm
# armhf, armel variants

# Key builtins: __aeabi_idiv, __aeabi_ldivmod, __aeabi_fadd, __aeabi_dadd
# These are called automatically when hardware divide/FP is absent
```

---

## 97.7 Frame Pointer Chains

### 97.7.1 R11 as Frame Pointer

In A32 code, R11 is the conventional frame pointer. In Thumb-2, LLVM uses R11 as well (matching modern GCC behavior), though historically GCC used R7:

```asm
; A32 function prologue with frame chain
foo:
    push   {r4, r11, lr}      ; callee-save: r4, fp, return address
    add    r11, sp, #4         ; frame pointer = SP + offset-to-saved-fp
    sub    sp, sp, #32         ; allocate 32 bytes local vars
    ; ... function body ...
    sub    sp, r11, #4         ; restore SP from frame pointer
    pop    {r4, r11, pc}       ; restore and return (pc gets lr)
```

### 97.7.2 Frame Omission

`-fomit-frame-pointer` is the default at `-O1` and above, recovering R11/R7 as a general-purpose callee-saved register. On Cortex-M where every register counts, this is especially important. The `ARMFrameLowering::hasFP()` function decides whether a frame pointer is needed:

```cpp
bool ARMFrameLowering::hasFP(const MachineFunction &MF) const {
    return (MF.getTarget().Options.DisableFramePointerElim(MF) ||
            MFI.hasVarSizedObjects() ||
            MFI.isFrameAddressTaken() ||
            MF.getSubtarget<ARMSubtarget>().isThumb1Only());
}
```

---

## 97.8 Thumb-2 IT Blocks

### 97.8.1 IT Block Encoding

The `IT` (If-Then) instruction enables conditional execution for up to 4 subsequent Thumb instructions without branching. The IT opcode encodes the condition and a T/E (Then/Else) mask for up to 3 post-IT instructions:

```asm
; ITTT: execute next 3 instructions if EQ
CMP   r0, r1
ITTT  EQ          ; EQ=then, EQ=then, EQ=then
MOVEQ r2, #1      ; r2 = 1 if EQ
ADDEQ r3, r3, #1  ; r3++ if EQ
SUBEQ r4, r4, #1  ; r4-- if EQ

; ITE: if/else
CMP   r0, #0
ITE   GT
MOVGT r1, #1      ; positive case
MOVLE r1, #-1     ; non-positive case
```

The LLVM `ARMITBlockInsertion` pass groups predicated Thumb-2 instructions into IT blocks after instruction selection. The pass runs late in the machine pass pipeline, after register allocation.

### 97.8.2 ARMv8 Deprecation

ARMv8-A (AArch32 state) deprecated IT blocks with more than one condition (ITETT, ITTEE, etc.), and ARMv8-R/M may not support them. LLVM avoids generating multi-instruction IT blocks when targeting ARMv8:

```cpp
// ARMITBlockInsertion.cpp
// Only generate IT blocks with >1 instruction for pre-ARMv8
if (STI.isThumb2() && !STI.isV8Modes())
    // Can generate ITTT/ITTE/etc.
else
    // Only single IT (ITE{cond}) is safe for ARMv8 AArch32
```

---

## 97.9 ARM SelectionDAG and GlobalISel Status

### 97.9.1 SelectionDAG Path

The ARM backend's SelectionDAG implementation is mature, covering all ARMv5T through ARMv8-A feature combinations. Key files and their responsibilities:

| File | Responsibility |
|------|---------------|
| `ARMISelLowering.cpp` | Custom lowering: calls, FP ops, atomics, vector types |
| `ARMISelDAGToDAG.cpp` | DAG-to-DAG instruction selection, addressing modes |
| `ARMInstrInfo.td` | Base A32 + Thumb-2 instruction definitions |
| `ARMInstrNEON.td` | NEON SIMD instruction patterns |
| `ARMInstrVFP.td` | VFP floating-point instruction patterns |
| `ARMInstrThumb.td` | Thumb-1 instruction definitions |
| `ARMInstrThumb2.td` | Thumb-2 specific instruction definitions |
| `ARMConstantIslandPass.cpp` | Constant pool placement and branch fixup |

### 97.9.2 LLVM Peephole and Combines

The ARM backend includes several machine-level optimizations:

```cpp
// ARMInstrInfo.cpp: fuse CMP+B into CBZ/CBNZ for Thumb-2
bool ARMInstrInfo::optimizeCompareInstr(...) {
  // CMP Rn, #0 followed by BEQ → CBZ Rn (saves instruction)
  // CMP Rn, #0 followed by BNE → CBNZ Rn
}
```

`ARMLoadStoreOptimizer.cpp` merges consecutive loads/stores into `LDM`/`STM` multiple-transfer instructions, a significant throughput win on A-profile cores.

### 97.9.3 GlobalISel on ARM

GlobalISel support for 32-bit ARM is partially implemented but not production-ready as of LLVM 22.1.x. The existing coverage:

- `ARMLegalizerInfo`: covers basic integer, float32/64, simple vector types
- `ARMRegisterBankInfo`: GPR (integer) vs FPR (VFP/NEON) bank assignment
- `ARMInstructionSelector`: handles A32/Thumb-2 for basic integer operations

Known gaps: complex Thumb-1 register restrictions, VFP register pairing constraints, NEON complex shuffles, and atomic operations via LL/SC loops.

---

## 97.10 Inline Assembly Constraints

### 97.10.1 Constraint Letters

ARM inline assembly uses GCC-compatible constraint letters:

| Constraint | Meaning |
|-----------|---------|
| `r` | Any 32-bit GPR (r0–r15) |
| `l` | Low GPR only (r0–r7, Thumb-1 compatible) |
| `h` | High GPR (r8–r15) |
| `w` | VFP/NEON D or Q register |
| `t` | VFP/NEON S register (single-precision) |
| `x` | VFP/NEON S register (high S16–S31) |
| `Uv` | VFP memory operand |
| `Uf` | VFP float register constraint |
| `m` | Any valid memory address |
| `I` | 8-bit ARM immediate (rotated) |
| `J` | Thumb-2 modified immediate |

### 97.10.2 Inline Assembly Examples

```c
// Bitfield extraction using UBFX (Thumb-2 / A32)
static inline uint32_t ubfx(uint32_t val, int lsb, int width) {
    uint32_t result;
    __asm__("ubfx %0, %1, %2, %3"
            : "=r"(result)
            : "r"(val), "I"(lsb), "I"(width));
    return result;
}

// Count leading zeros
static inline int arm_clz(uint32_t x) {
    int result;
    __asm__("clz %0, %1" : "=r"(result) : "r"(x));
    return result;
}

// Read CPSR (current program status register)
static inline uint32_t read_cpsr(void) {
    uint32_t cpsr;
    __asm__("mrs %0, cpsr" : "=r"(cpsr));
    return cpsr;
}

// NEON inline assembly: multiply-accumulate 4×f32
void neon_mac(float* acc, const float* a, const float* b) {
    __asm__ volatile (
        "vld1.32  {q0}, [%1]\n"   // load a[0..3]
        "vld1.32  {q1}, [%2]\n"   // load b[0..3]
        "vld1.32  {q2}, [%0]\n"   // load acc[0..3]
        "vmla.f32  q2, q0, q1\n"  // acc += a * b
        "vst1.32  {q2}, [%0]\n"   // store acc
        : "+r"(acc)
        : "r"(a), "r"(b)
        : "q0", "q1", "q2", "memory"
    );
}

// Coprocessor access (MCR/MRC for system registers)
static inline void write_cp15_r0(uint32_t val) {
    __asm__("mcr p15, 0, %0, c1, c0, 0" : : "r"(val));
}
```

---

## 97.11 ARMv8-A AArch32 Mode

### 97.11.1 Differences from ARMv7-A

ARMv8-A defines an AArch32 execution state for backward compatibility. Notable changes:
- IT blocks restricted to a single conditional instruction (ITTT etc. are deprecated)
- `LDM`/`STM` writeback forms with PC as base deprecated
- CRC32 hardware instructions (`CRC32B`, `CRC32H`, `CRC32W`, `CRC32CB`, `CRC32CH`, `CRC32CW`)
- `SDIV`/`UDIV` available in A32 state (previously only in Thumb-2/ARMv7)
- Large Physical Address Extensions (LPAE) for >4 GB physical memory

```bash
# Target ARMv8-A AArch32
clang --target=armv8a-linux-gnueabihf \
    -mfpu=neon-fp-armv8 \
    -O2 foo.c -o foo

# Verify ARMv8 features
/usr/lib/llvm-22/bin/llc -march=arm \
    -mcpu=cortex-a53 \
    -mattr=+v8 \
    test.ll -o test.s
```

---

## 97.12 Scheduling Models

ARM backend scheduling models reside in `ARMSchedule*.td`:

| Target | Model |
|--------|-------|
| ARM7TDMI | `ARMScheduleV6.td` (simple in-order) |
| Cortex-A7 | `ARMScheduleA7.td` |
| Cortex-A8 | `ARMScheduleA8.td` |
| Cortex-A9 | `ARMScheduleA9.td` |
| Cortex-A15 | `ARMScheduleA15.td` |
| Cortex-A53 | `ARMScheduleA53.td` |
| Cortex-M3/M4 | `ARMScheduleM3.td` |
| Cortex-M7 | `ARMScheduleM7.td` (dual-issue) |
| Swift (Apple A6) | `ARMScheduleSwift.td` |

```bash
# Use Cortex-A15 scheduling model
/usr/lib/llvm-22/bin/llc -march=arm -mcpu=cortex-a15 \
    -mattr=+neon,+vfp4 \
    test.ll -o test_a15.s
```

---

## Armv8.1-M PACBTI: Pointer Authentication and Branch Target Identification for Cortex-M

Pointer Authentication (PAuth) and Branch Target Identification (BTI) on AArch64 (covered in §96.6 and §96.7) are A-profile features targeting application processors running full operating systems. PACBTI is the M-profile equivalent: a subset of PAC and BTI adapted for the Thumb-32 instruction set and the constraints of safety-critical microcontrollers. With Cortex-M85 shipping in 2023 and industrial/automotive firmware increasingly requiring software security certification, PACBTI has become a first-class LLVM 32-bit ARM target feature.

### Armv8.1-M PACBTI Extension

The PACBTI extension is defined in the Armv8.1-M architecture specification and targets M-profile cores that need control-flow integrity without the overhead of a full OS or MPU-based process isolation. Cortex-M85 is the first Cortex-M processor to implement PACBTI; future safety-critical cores are expected to follow.

PACBTI provides three overlapping mechanisms:

**PAC (Pointer Authentication Code)**: Embeds a cryptographic code in the spare bits of a pointer (typically using the frame address and a key). On M-profile, the PAC is computed over the link register (LR) to protect return addresses.

**AUT (Authenticate)**: Strips and verifies the PAC code before a pointer is used. If verification fails, the address is corrupted to a non-canonical form that will trigger a HardFault on use.

**BTI (Branch Target Indication)**: Marks valid targets for indirect branches. On Thumb-32, BTI is encoded as a `PACBTI` instruction at the function entry or as a `BTI` NOP-equivalent at jump targets.

### Key Instructions in Thumb-32

The Armv8.1-M PACBTI instruction set is a subset of the AArch64 PAC/BTI instructions, re-encoded for Thumb-32:

| Instruction | Encoding | Action |
|-------------|----------|--------|
| `PACBTI LR, SP` | 16-bit NOP slot at entry | Sign LR using SP as modifier; mark as BTI landing |
| `PAC LR, SP` | T32 encoding | Sign LR using IA key and SP modifier |
| `AUT LR, SP` | T32 encoding | Authenticate LR against IA key and SP |
| `BTI` | T32 NOP-equivalent | Mark as valid indirect branch target |
| `AUTG R0, LR, SP` | T32 encoding | Authenticate R0 using LR and SP as modifiers |
| `PACG R0, LR, SP` | T32 encoding | Sign R0 using LR and SP as modifiers |

A notable difference from AArch64: the Thumb-32 `BTI` instruction is encoded as a NOP-like hint instruction (`HINT #8`) in the Thumb-32 encoding space, meaning BTI-unaware processors execute it as a NOP without fault. This is intentional for backward compatibility in the M-profile ecosystem, where bootloaders and libraries must execute on a range of cores.

The `PACBTI` combined instruction at function entry simultaneously marks the function as a BTI landing pad **and** signs the link register, eliminating the need for two separate instructions:

```asm
; Armv8.1-M function prologue with PACBTI
foo:
    pacbti  lr, sp           ; BTI landing + sign LR with SP as context
    push    {r4, r5, lr}     ; save callee-saved + signed LR
    ; ... function body ...
    pop     {r4, r5, lr}     ; restore
    aut     lr, sp           ; authenticate LR before return
    bx      lr               ; return (faults if LR was tampered)
```

### Landing Pad Model Under Thumb-32

The Armv8.1-M BTI landing pad model differs from AArch64 in two respects:

1. **NOP-compatible encoding**: As noted, `BTI` is a `HINT` instruction. Hardware that does not implement PACBTI ignores it; hardware that does enforce BTI requires it at indirect branch targets.

2. **No BTI-C/BTI-J distinction at the ISA level**: AArch64 has separate `BTI C` (call targets) and `BTI J` (jump targets) variants. Armv8.1-M has a single `BTI` form that covers both indirect calls (`BLX Rn`) and indirect jumps (`BX Rn`). The LLVM backend emits `BTI` at all indirect branch target sites when PACBTI is enabled.

### Clang Code Emission

PACBTI is enabled via the `-march` flag:

```bash
# Compile for Cortex-M85 with PACBTI
clang --target=thumbv8.1m.main-none-eabi \
    -march=armv8.1-m.main+pacbti+fp.dp \
    -mfloat-abi=hard \
    -mbranch-protection=pac-ret+bti \
    -mcpu=cortex-m85 \
    foo.c -o foo.elf
```

The `-mbranch-protection=pac-ret+bti` flag (the same flag as AArch64) activates both return-address signing and BTI landing pad insertion. On 32-bit ARM with `+pacbti`, Clang routes these to the ARM-specific `ARMAsmPrinter` PACBTI emission, not the AArch64 path.

Relevant LLVM source files:
- `llvm/lib/Target/ARM/ARMAsmPrinter.cpp`: PACBTI prologue/epilogue emission
- `llvm/lib/Target/ARM/ARMBranchTargets.cpp`: BTI landing pad insertion pass
- `llvm/lib/Target/ARM/ARMSubtarget.h`: `hasPACBTI()` predicate

### Interaction with the 32-bit ARM ABI (AAPCS)

Under AAPCS (the 32-bit ARM ABI), functions distinguish between:
- **Leaf functions**: Do not call any other function; LR is not spilled to the stack
- **Non-leaf functions**: LR is saved to the stack in the prologue and restored in the epilogue

PACBTI LR signing applies to both categories, but the signing moment differs:

```asm
; Non-leaf: sign on entry, authenticate before pop
non_leaf:
    pacbti  lr, sp           ; sign LR immediately
    push    {r4, lr}         ; spill signed LR
    bl      helper           ; BL overwrites LR with return address
    pop     {r4, lr}         ; restore saved (signed) LR
    aut     lr, sp           ; authenticate
    bx      lr

; Leaf: sign on entry, authenticate before return
leaf:
    pacbti  lr, sp           ; sign LR
    ; ... no BL calls ...
    aut     lr, sp           ; authenticate before use
    bx      lr
```

This matches the AArch64 `PACIASP`/`AUTIASP` pattern but uses the ARM AAPCS stack pointer (SP, R13) as the signing modifier rather than AArch64's dedicated SP register.

### Use Cases: Safety-Critical and IoT Firmware

PACBTI is particularly relevant in three domains:

**Automotive (ISO 26262 ASIL-D)**: Control units running M-profile microcontrollers are increasingly required to demonstrate software CFI. PACBTI provides hardware-enforced return-address integrity for ISO 26262 Safety Elements out of Context (SEooC).

**Industrial Safety (IEC 61508 SIL 3)**: Industrial PLCs and safety systems using Cortex-M85 can use PACBTI to demonstrate protection against code-injection and ROP chains in IEC 61508 functional safety assessments.

**IoT Firmware Security (PSA Certified)**: Arm's Platform Security Architecture (PSA) recommends PACBTI for Cortex-M85-based devices targeting PSA Certified Level 2 and above. LLVM's Clang is the reference toolchain for Arm SystemReady and PSA toolchains.

### Comparison with AArch64 PAuth

| Property | AArch64 PAuth (§96.6) | Armv8.1-M PACBTI |
|----------|----------------------|-----------------|
| Target profile | A-profile (Cortex-A) | M-profile (Cortex-M) |
| Instructions | `PACIA`, `AUTIA`, `PACIB`, `AUTIB`, etc. | `PAC`, `AUT`, `PACBTI`, `BTI` |
| BTI variants | `BTI C` / `BTI J` / `BTI JC` | Single `BTI` |
| Signing keys | IA/IB/DA/DB/GA | IA only (single key in M-profile) |
| NOP-compatible BTI | No (fault on BTI violation) | Yes (`HINT #8` NOP-compatible) |
| PAC key storage | System registers (EL0-visible) | PAC key stored in CP15 |
| Clang flag | `-mbranch-protection=pac-ret+bti` | Same flag, different backend |

The core intent is the same — cryptographic return-address integrity and hardware-enforced CFI — but the deployment constraints differ markedly. AArch64 PAuth assumes a rich OS with per-process signing keys; Armv8.1-M PACBTI assumes a bare-metal or RTOS environment with a single globally shared key managed by firmware.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **GlobalISel completion for Thumb-1**: The ongoing LLVM effort to complete GlobalISel coverage for ARMv6-M (Cortex-M0/M0+) is tracked in several Phabricator reviews and discourse.llvm.org threads. The primary blockers — Thumb-1 low-register constraints (r0–r7 only) and the absence of 32-bit Thumb-2 forms — are being addressed by extending `ARMLegalizerInfo` and `ARMInstructionSelector` to handle `G_ADD`/`G_SUB`/`G_LOAD`/`G_STORE` under Thumb-1 register constraints.
- **Armv8.1-M Helium (MVE) vector intrinsics stabilization**: The M-Profile Vector Extension (MVE/Helium) introduced in Cortex-M55/M85 exposes 128-bit SIMD over the Thumb-32 encoding space. As of LLVM 22.1.x, the auto-vectorizer backend for MVE is functional but conservative; near-term work focuses on widening the loop vectorizer's cost model to correctly account for MVE's single-issue execution on Cortex-M55 and improving the `ARMMVEVectorIselLowering` patterns for shuffles and reductions.
- **PACBTI key management in TrustZone-M environments**: The Armv8.1-M PACBTI extension interacts with TrustZone-M (Armv8-M Security Extension) in that the IA signing key must be saved/restored across Non-Secure to Secure transitions. An LLVM patch series (tracked under `[ARM] PACBTI TrustZone interaction`) is adding `VLSTM`/`VLLDM` lazy context save support in the prologue/epilogue inserter for functions marked `__attribute__((cmse_nonsecure_entry))`.
- **LLD veneer generation for BTI-enabled M-profile binaries**: When linking Cortex-M85 firmware with BTI enabled, LLD must ensure that auto-generated interworking veneers are themselves BTI-compatible (i.e., begin with a `BTI` or `PACBTI` instruction). The RFC for BTI-safe LLD veneers on M-profile is under active review at discourse.llvm.org/t/bti-safe-veneers-for-m-profile.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Cortex-M52/M55/M85 scheduling models in LLVM**: LLVM currently lacks production-quality scheduling models for the Cortex-M55 (dual-issue in-order, MVE pipeline) and Cortex-M85 (two-way superscalar, branch prediction, PACBTI). Accurate `ARMScheduleM55.td` and `ARMScheduleM85.td` models are needed for the loop vectorizer and instruction scheduler to exploit instruction-level parallelism on these cores.
- **Armv9.2-M (anticipated)**: Arm's public roadmap suggests continued iteration of the M-profile ISA in the Armv9 generation, likely adding Reliability/Availability/Serviceability (RAS) extensions and additional PACBTI capabilities. LLVM will need new `ARMSubtarget` feature bits, updated `ARMTargetInfo`, and encoding support in `ARMDisassembler` for any new hint and diagnostic instructions.
- **32-bit ARM GlobalISel reaching feature parity with SelectionDAG**: The LLVM community goal is for GlobalISel to be the default instruction selection pipeline for 32-bit ARM by the end of this horizon. Remaining work includes: atomic LL/SC lowering, VFP/NEON register pairing during regbank assignment, Thumb-2 complex shuffle lowering, and integration of the `MachineCombiner` for VFP instruction combining.
- **LLVM BOLT support for AArch32 binaries**: BOLT (Binary Optimization and Layout Tool) currently targets AArch64 and x86-64. Extending BOLT to AArch32 (32-bit Linux binaries on A-profile hardware) would enable profile-guided function reordering and basic-block reordering for legacy ARMv7-A Android and embedded Linux workloads. Initial design discussions have appeared on the LLVM mailing list.

### 5-Year Horizon (Long-Term, by ~2031)

- **Automated ISA coverage testing via LLVM MC fuzzing**: As the 32-bit ARM ISA is frozen (no new public A32/Thumb-2 extensions are expected beyond those already in ARMv9), focus shifts to test completeness. Systematic fuzzing of `ARMDisassembler` and `ARMAsmParser` against the Arm Architecture Validation Suite (AVS) and the `riscv-isa-sim`-style reference model will expose encoding corner cases, particularly in IT block decoding, VFP/NEON coexistence, and M-profile-specific Thumb encodings.
- **Formal verification of the 32-bit ARM backend via Alive2 integration**: The Alive2 tool (developed at UCSD/UTAustin, now integrated into LLVM CI for x86) verifies peephole optimizations by encoding source and target patterns as SMT queries. Extending Alive2's backend models to cover ARM32 register constraints, condition codes, and the barrel-shifter operand encoding would allow formal verification of the hundreds of ARM-specific peephole patterns in `ARMInstrInfo.cpp`.
- **Rust and Safety-Critical C++ toolchain certification for Cortex-M**: Safety standards bodies (ISO 26262, IEC 61508, DO-178C) are increasingly interested in certifying LLVM-based toolchains for M-profile targets. By 2031, TÜV SÜD and similar bodies are expected to have published qualification kits for LLVM/Clang targeting Cortex-M3 through Cortex-M85, requiring that the `arm-none-eabi` backend pass deterministic reproducibility, diagnostic quality, and regression baselines against the Arm Compiler 6 reference toolchain.

---

## Chapter Summary

- The 32-bit ARM backend supports three ISA modes: A32 (uniform 32-bit with condition codes and barrel-shifter in every instruction), Thumb-1 (16-bit restricted to low registers), and Thumb-2 (mixed 16/32-bit with most A32 capabilities, including 12-bit immediates and high-register access).
- ARM/Thumb interworking uses BX/BLX to switch state based on bit 0 of the branch target; linker veneers handle long-range and cross-ISA calls, with the LLD linker generating `LDR+BX` (ARM→Thumb) or `MOVW/MOVT+BX` (Thumb→ARM) sequences.
- AAPCS passes the first 4 integer words in r0–r3; three ABI variants control FP argument registers: `soft` (integer regs, software FP), `softfp` (integer regs, hardware FP compute), and `hardfp` (FP args in VFP S/D registers).
- VFP provides S0–S31 (single) / D0–D31 (double) registers; NEON overlays Q0–Q15 (128-bit) on the D register file; VFPv4 adds fused multiply-accumulate (`VFMA`); NEON is absent on Cortex-M profile.
- Cortex-M (ARMv7-M) is Thumb-2-only, has no MMU, no NEON, and optional single-precision FPv4-SP (M4) or FPv5 (M7); bare-metal builds use `-target thumbvXm-none-eabi` with `-nostdlib` and custom linker scripts.
- R11 is LLVM's frame pointer in both A32 and Thumb-2; `-fomit-frame-pointer` at `-O1+` recovers it as a general-purpose callee-saved register.
- Thumb-2 IT blocks permit up to 4 conditionally-executed instructions without branching; `ARMITBlockInsertion` constructs them post-regalloc, but ARMv8 AArch32 deprecates multi-instruction IT blocks.
- ARM inline assembly uses `r` (any GPR), `l` (low GPR r0–r7), `w` (NEON D/Q), `t` (VFP S register), and `I`/`J` (rotated ARM or Thumb-2 modified immediate) constraints.
- The SelectionDAG path is production-ready for all ARM/Thumb/NEON/VFP variants; GlobalISel is partially implemented but not recommended for production as of LLVM 22.1.x.
- Armv8.1-M PACBTI (Cortex-M85+) brings hardware return-address signing (`PAC`/`AUT`) and branch target identification (`BTI`) to M-profile; the `PACBTI` combined instruction at function entry simultaneously marks a BTI landing pad and signs the link register; enabled with `-march=armv8.1-m.main+pacbti -mbranch-protection=pac-ret+bti` for ISO 26262 and IEC 61508 safety-critical firmware.


---

@copyright jreuben11
