# Chapter 97 — The 32-bit ARM Backend

*Part XV — Targets*

The 32-bit ARM backend in LLVM spans the widest ISA diversity of any supported target: A32 (the classic 32-bit ARM instruction set), Thumb (16-bit compressed), and Thumb-2 (mixed 16/32-bit) co-exist in a single binary and can interwork freely. The target also serves two architecturally distinct markets — application processors (ARMv7-A, Cortex-A7/A9/A15) running Linux and Android, and microcontrollers (ARMv7-M, Cortex-M0/M3/M4/M7) in bare-metal embedded systems. These profiles share the same LLVM target but have drastically different constraints: no NEON on Cortex-M, no MMU, no OS-level stack alignment guarantees, and often no hardware floating-point at all. This chapter covers LLVM's ARM backend from ISA interworking through VFP/NEON SIMD to embedded bare-metal constraints and the inline assembly interface, giving you the practical knowledge to generate, debug, and extend 32-bit ARM code in LLVM 22.1.x.

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
