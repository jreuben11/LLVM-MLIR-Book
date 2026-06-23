# Chapter 101 — PowerPC, SystemZ, MIPS, SPARC, and LoongArch

*Part XV — Targets*

Beyond the dominant x86, AArch64, and RISC-V targets, LLVM maintains production-quality backends for a set of architecturally distinct server-class, mainframe, and legacy platforms. PowerPC powers IBM POWER servers, OpenPOWER ecosystem hardware, and game consoles; SystemZ underpins IBM Z mainframes running millions of banking transactions per second; MIPS persists in networking silicon and legacy embedded systems; SPARC remains in some safety-critical embedded and legacy server contexts; and LoongArch is a new Chinese RISC ISA rapidly gaining traction in consumer devices and data center deployments. This chapter surveys each target's register model, ABI, SIMD capabilities, and notable LLVM implementation details — providing sufficient depth to work with, cross-compile for, or extend these backends without duplicating platform-specific documentation.

---

## Table of Contents

- [101.1 PowerPC: Register Model, VSX, and ELFv2 ABI](#1011-powerpc-register-model-vsx-and-elfv2-abi)
  - [101.1.1 Register Architecture](#10111-register-architecture)
  - [101.1.2 AltiVec/VMX/VSX SIMD](#10112-altivecvmxvsx-simd)
  - [101.1.3 ELFv2 ABI (OpenPOWER)](#10113-elfv2-abi-openpower)
  - [101.1.4 GNU/IBM Extended-Precision FP](#10114-gnuibm-extended-precision-fp)
  - [101.1.5 LLVM Pipeliner on Power10 MMA](#10115-llvm-pipeliner-on-power10-mma)
- [101.2 SystemZ: IBM Z Architecture](#1012-systemz-ibm-z-architecture)
  - [101.2.1 Register Architecture](#10121-register-architecture)
  - [101.2.2 z/Architecture Vector Instruction Set](#10122-zarchitecture-vector-instruction-set)
  - [101.2.3 S/390 Linux ABI](#10123-s390-linux-abi)
  - [101.2.4 Packed Decimal and EXECUTE](#10124-packed-decimal-and-execute)
  - [101.2.5 Hardware Transactional Memory (HTM)](#10125-hardware-transactional-memory-htm)
- [101.3 MIPS and MIPS64](#1013-mips-and-mips64)
  - [101.3.1 Register Architecture](#10131-register-architecture)
  - [101.3.2 o32, n32, n64 ABIs](#10132-o32-n32-n64-abis)
  - [101.3.3 MIPS16 and microMIPS](#10133-mips16-and-micromips)
  - [101.3.4 MSA SIMD (MIPS SIMD Architecture)](#10134-msa-simd-mips-simd-architecture)
  - [101.3.5 MIPS R6 Breaking Changes](#10135-mips-r6-breaking-changes)
- [101.4 SPARC and SPARC64](#1014-sparc-and-sparc64)
  - [101.4.1 Register Windows](#10141-register-windows)
  - [101.4.2 SPARC V9 / SPARC64](#10142-sparc-v9-sparc64)
  - [101.4.3 LEON Embedded SPARC](#10143-leon-embedded-sparc)
- [101.5 LoongArch](#1015-loongarch)
  - [101.5.1 Architecture Overview](#10151-architecture-overview)
  - [101.5.2 Register Architecture](#10152-register-architecture)
  - [101.5.3 LP64D ABI](#10153-lp64d-abi)
  - [101.5.4 LoongArch SIMD: LSX and LASX](#10154-loongarch-simd-lsx-and-lasx)
  - [101.5.5 LBT: Binary Translation Helpers](#10155-lbt-binary-translation-helpers)
  - [101.5.6 LLVM LoongArch Maturity](#10156-llvm-loongarch-maturity)
- [101.6 LoongArch in Depth](#1016-loongarch-in-depth)
  - [101.6.1 ISA Overview: LoongArch v1.1](#10161-isa-overview-loongarch-v11)
  - [101.6.2 TLS Models Including TLSDESC](#10162-tls-models-including-tlsdesc)
  - [101.6.3 Vector Extensions: LSX and LASX in Detail](#10163-vector-extensions-lsx-and-lasx-in-detail)
  - [101.6.4 LLD and RuntimeDyld Support](#10164-lld-and-runtimedyld-support)
  - [101.6.5 Production Status](#10165-production-status)
- [101.7 VE: NEC SX-Aurora Tsubasa](#1017-ve-nec-sx-aurora-tsubasa)
- [101.8 SystemZ z/OS Support](#1018-systemz-zos-support)
- [101.9 Cross-Compilation Reference](#1019-cross-compilation-reference)
  - [101.9.1 Compilation Flags](#10191-compilation-flags)
  - [101.9.2 Target-Specific IR Inspection](#10192-target-specific-ir-inspection)
- [101.10 Comparative Architecture Summary](#10110-comparative-architecture-summary)
- [Chapter Summary](#chapter-summary)

---

## 101.1 PowerPC: Register Model, VSX, and ELFv2 ABI

### 101.1.1 Register Architecture

PowerPC 64-bit (ppc64) has a rich register file:

- **General-Purpose Registers (GPRs)**: R0–R31 (64-bit on ppc64, 32-bit on ppc32). Special roles: R0=volatile (not base register), R1=stack pointer, R2=TOC pointer (ELFv2), R13=TLS base pointer on Linux, R31=frame pointer convention.
- **Floating-Point Registers (FPRs)**: F0–F31 (64-bit double); overlap with VSX lower half (VS0–VS31).
- **VSX registers**: 64 × 128-bit registers. VS0–VS31 alias FPRs F0–F31 (as upper 64 bits); VS32–VS63 alias VMX (AltiVec) VR0–VR31. VSX instructions can reference the full 64-register file.
- **AltiVec/VMX**: VR0–VR31 (128-bit); aliased as VS32–VS63 under VSX.
- **Condition Register (CR)**: 8 × 4-bit fields (CR0–CR7); each has LT, GT, EQ, SO (summary overflow).
- **Special-purpose**: LR (link register), CTR (count register, also used for counted loops), XER (exception register: SO, OV, CA bits, and byte count).

```tablegen
// PPCRegisterInfo.td
def R1  : GPR<1,  "r1">;    // stack pointer
def R2  : GPR<2,  "r2">;    // TOC pointer (ELFv2)
def R3  : GPR<3,  "r3">;    // first int arg / return
def LR  : SPR<8,  "lr">;    // link register
def CTR : SPR<9,  "ctr">;   // count register
```

### 101.1.2 AltiVec/VMX/VSX SIMD

Three overlapping SIMD instruction sets:

**AltiVec/VMX** (POWER6+): 128-bit operations on VR0–VR31 only:
```asm
; AltiVec integer add: 4×i32
vadduwm  v0, v1, v2

; AltiVec permute: byte-level (16-byte index in v4)
vperm    v5, v6, v7, v8

; AltiVec compare: 4×i32 signed greater-than → mask
vcmpgtsw v0, v1, v2
```

**VSX** (POWER7+): 128-bit operations on all 64 VS registers with new encodings:
```asm
; VSX double-precision FP: 2×f64 using d-register view
xvadddp  vs0, vs1, vs2

; VSX single-precision: 4×f32
xvaddsp  vs0, vs1, vs2

; VSX permute and blend: 2 elements
xxpermdi vs5, vs6, vs7, 0b01
```

**Power10 MMA** (Matrix-Multiply Accumulate): 512-bit accumulator registers (ACC0–ACC7), each comprising 4 consecutive VSX registers (VS0:VS3, VS4:VS7, etc.):
```asm
; Outer product of two 4×f32 vectors into 4×4 matrix accumulator
xvf32gerpp ACC0, vs4, vs5    ; ACC0 += vs4 ⊗ vs5  (4×4 outer product)

; Reduction of accumulator to output
pmxvf32ger ACC1, vs2, vs3, 3, 7  ; masked 4×4 outer product
```

### 101.1.3 ELFv2 ABI (OpenPOWER)

ELFv2 (introduced for ppc64le, little-endian POWER8+) simplified the legacy ELFv1 ABI:

**Eliminated function descriptor**: In ELFv1, function pointers pointed to a 3-word descriptor (code address, TOC pointer, environment pointer). In ELFv2, function pointers point directly to the first instruction.

**Register assignment:**
- **Integer arguments**: R3–R10 (first 8 words); additional on stack
- **FP arguments**: F1–F13 (first 13 double-precision)
- **VMX arguments**: V2–V13 (first 12 vector args)
- **Return**: R3 (integer), F1 (FP), V2 (vector)
- **Callee-saved**: R14–R31, F14–F31, V20–V31, CR2/CR3/CR4 fields
- **TOC pointer**: R2 always holds module's TOC; cross-module calls use a stub that restores R2

**Local entry points**: ELFv2 distinguishes between global (from outside module) and local (within module) entry points; local calls skip the TOC setup prefix code, saving 2–3 instructions per call.

```bash
# ELFv2 little-endian POWER10
clang --target=powerpc64le-linux-gnu \
    -mcpu=power10 \
    -maltivec -mvsx \
    -O3 foo.c -o foo

# Check ELFv2 ABI local entry points
objdump -d foo | grep ".localentry"
```

### 101.1.4 GNU/IBM Extended-Precision FP

PowerPC supports 128-bit floating-point in two forms:
- **Software `__float128`** (GCC extension): pair of double-precision values (IBM long double, `ppc_fp128` in LLVM IR)
- **Hardware IEEE 754 quad** (`_Float128`): using `LXVQP`/`STXVQP` for load/store and `xsaddqp`/`xsmulqp`/`xsdivqp` for arithmetic on POWER9+

```cpp
// IBM long double (pair of doubles, 107-bit mantissa)
__ibm128 a = 1.0Q + 1.0e-30Q;

// True IEEE 754 quad (128-bit) on POWER9+
__attribute__((target("power9-fp")))
_Float128 ieee_quad_add(_Float128 a, _Float128 b) {
    return a + b;  // → xsaddqp
}
```

LLVM represents IBM long double as `ppc_fp128` type and lowers it to pairs of F registers.

### 101.1.5 LLVM Pipeliner on Power10 MMA

Power10's MMA outer-product instructions form tightly coupled 4×4 matrix accumulations. LLVM's Modulo Software Pipeliner (Chapter 91) is applied to MMA-heavy loops, scheduling accumulator initialization (`xxsetaccz`), outer product accumulation (`xvf32gerpp`), and result extraction (`xvfmaddasp`/`mfvsrd`) in a pipeline that hides the MMA latency:

```cpp
// Power10 MMA: batched matrix multiply
// LLVM activates the pipeliner via PPCMachineFunctionInfo
// which tracks accumulator register pairs (ACC0 = VS0:VS3)
void gemm_mma(float* C, const float* A, const float* B, int M, int N, int K) {
    // Inner loop unrolled 4× with MMA outer products
    // LLVM generates: xxsetaccz, vspltisw, xvf32gerpp (×4), xvfmaddasp
}
```

---

## 101.2 SystemZ: IBM Z Architecture

### 101.2.1 Register Architecture

IBM Z (s390x) is a register-rich load-store CISC architecture:

- **GPRs**: R0–R15 (64-bit). R0=volatile (not usable as base register in many instructions), R15=stack pointer, R14=return address, R13=frame pointer (by convention in Linux ABI). Even-odd pairs (R0:R1, R2:R3, ...) used for 128-bit operations, multi-register load/store, and long division.
- **FPRs**: F0–F15 (64-bit). F0:F2 and F4:F6 form 128-bit extended-precision pairs. F1, F3, F5, F7 are caller-saved temporaries.
- **Vector registers (z13+)**: V0–V31 (128-bit). V0–V15 alias F0–F15 in their low 64 bits. All V registers support integer, FP, and byte-vector operations.
- **Access registers**: AR0–AR15 (32-bit): used for dual address-space operations (rarely used in Linux user code).
- **Control registers**: CR0–CR15: kernel-only, for exception control and features.

```tablegen
// SystemZRegisterInfo.td
def R0D  : SystemZReg<"r0",  0>;  // volatile
def R15D : SystemZReg<"r15", 15>; // stack pointer
def R14D : SystemZReg<"r14", 14>; // return address
def R13D : SystemZReg<"r13", 13>; // frame pointer (Linux convention)
```

### 101.2.2 z/Architecture Vector Instruction Set

The VECTOR facility (introduced z13, 2015) provides 128-bit SIMD via V0–V31:

```asm
; Vector byte add
VAB    V1, V2, V3           ; V1[i] = V2[i] + V3[i] (byte elements)
VAH    V1, V2, V3           ; halfword
VAF    V1, V2, V3           ; fullword (4×i32)
VAG    V1, V2, V3           ; doubleword (2×i64)

; Vector FP operations
VFADB  V1, V2, V3           ; 2×f64 add
VFASB  V1, V2, V3           ; 4×f32 add
VFMSB  V1, V2, V3, V4       ; 4×f32 fused multiply-subtract

; Vector permute (byte-level shuffle using V4 as index)
VPERM  V1, V2, V3, V4       ; general byte permute

; Vector replicate (broadcast)
VREPB  V1, V2, 5            ; V1 = {V2[5], V2[5], ... } (16 bytes)

; Vector string operations (z14+)
VFAE   V1, V2, V3, M5       ; find any element equal
VSTRC  V1, V2, V3, V4, M5   ; string range compare
```

### 101.2.3 S/390 Linux ABI

The s390x Linux ABI specifics:

- **Integer args**: R2–R6 (first 5 32/64-bit args); R2–R3 for 64-bit pairs
- **FP args**: F0, F2, F4, F6 (first 4 floating-point)
- **Vector args** (z13+): V24, V26, V28, V30 (first 4 vector args, using high-halves of VRs)
- **Return**: R2 (integer), F0 (FP), V24 (vector)
- **Callee-saved**: R6–R15, F8–F15
- **Backchain**: R15 (SP) points to the previous frame; the saved SP value at offset 0 forms a linked list of frames

The s390x stack frame is well-defined: 160 bytes minimum overhead (return address at offset 112, GPR save area at 48–112, FPR save area at 128–192).

### 101.2.4 Packed Decimal and EXECUTE

Packed decimal operations (`CVB`=convert from BCD to binary, `CVD`=binary to BCD, `CVBG`/`CVDG` for 64-bit) support COBOL and financial applications that work in base-10.

The `EXECUTE` instruction (`EX Rx, addr`) executes the instruction at `addr` with its second byte OR'd with the low 8 bits of Rx. This enables parametric string operations:

```asm
; Copy exactly N bytes (N in R1)
EX    R1, mvc_template     ; executes MVC with length from R1[56:63]
; (places EX target in separate fragment to avoid self-modification)
mvc_template: MVC  0(1, R3), 0(R2)  ; length=1+R1[56:63]

; EXRL: PC-relative form (z10+), avoids base register
EXRL  R1, mvc_template
```

### 101.2.5 Hardware Transactional Memory (HTM)

IBM z196 introduced hardware transactional memory with the Transaction Execution facility:

```asm
TBEGIN   0(R1), 0xFF00    ; begin unrestricted transaction
; ... transactional body ...
TEND                       ; commit transaction

TBEGINC  0(R1), 0xFF00    ; begin constrained transaction (simpler)

; Abort handlers check condition code:
; CC=0: transaction committed
; CC=1: transaction aborted (check TAR/TDB for reason)
; CC=2: transient abort (retry)
; CC=3: persistent abort (use fallback lock path)
```

LLVM exposes HTM through `__builtin_tbegin()`, `__builtin_tend()`, `__builtin_tabort()`, and `__builtin_tx_nesting_depth()`, which lower to `TBEGIN`/`TEND`/`TABORT`/`ETND` instructions via `SystemZISD::TBEGIN`, etc.

---

## 101.3 MIPS and MIPS64

### 101.3.1 Register Architecture

MIPS has 32 GPRs and 32 FPRs with well-defined ABI roles:

| Register | Name | Role |
|----------|------|------|
| $0 | zero | Hardwired zero |
| $1 | at | Assembler temporary (reserved) |
| $2–$3 | v0–v1 | Return values |
| $4–$7 | a0–a3 | Arguments (o32); extended to a7 in n32/n64 |
| $8–$15 | t0–t7 | Temporaries (caller-saved) |
| $16–$23 | s0–s7 | Callee-saved |
| $24–$25 | t8–t9 | Temporaries (t9=function address for PIC) |
| $26–$27 | k0–k1 | Kernel reserved |
| $28 | gp | Global pointer |
| $29 | sp | Stack pointer |
| $30 | s8/fp | Frame pointer (callee-saved) |
| $31 | ra | Return address |

FPRs $f0–$f31: In MIPS32r2+, 64-bit FPRs (accessible as single $f0, $f2, ...). In MIPS64, all FPRs are 64-bit. HI and LO registers hold the product/quotient of MUL/DIV operations (removed in MIPS R6).

### 101.3.2 o32, n32, n64 ABIs

| ABI | Width | Args (int) | Args (FP) | Stack | Notes |
|-----|-------|-----------|-----------|-------|-------|
| o32 | 32-bit | a0–a3 (4) | $f12, $f14 | 8-byte | Most legacy 32-bit MIPS |
| n32 | 32-bit (n64 regs) | a0–a7 (8) | $f12–$f19 | 16-byte | LP64 registers, ILP32 types |
| n64 | 64-bit | a0–a7 (8) | $f12–$f19 | 16-byte | Full 64-bit MIPS64 ABI |

The n32 ABI uses 64-bit registers for efficiency but defines `long` as 32 bits — useful for code that needs 64-bit FPRs but 32-bit pointer-sized integers.

```bash
# MIPS32r2 with o32 ABI
clang --target=mipsel-linux-gnu -mabi=32 -O2 foo.c -o foo

# MIPS64 with n64 ABI
clang --target=mips64el-linux-gnuabi64 -mabi=n64 -O2 foo.c -o foo
```

### 101.3.3 MIPS16 and microMIPS

**MIPS16**: A 16-bit compressed encoding variant for code-size reduction. Only a small register subset (R2–R7, R16–R17) is accessible in most MIPS16 instructions. Interworking with MIPS32 is via `jalrc` and function stubs (similar to ARM Thumb interworking). LLVM's MIPS16 support is incomplete and not recommended for new code.

**microMIPS** (MIPS32r5/r6, MIPS64r5/r6): A more capable 16/32-bit mixed encoding, similar to Thumb-2. Provides most MIPS32 instructions in 16-bit form when operands meet width constraints. LLVM has a more complete microMIPS implementation via separate instruction definitions in `MipsInstrInfo16.td`.

### 101.3.4 MSA SIMD (MIPS SIMD Architecture)

MSA (introduced MIPS32r5 / MIPS64r5) provides 128-bit SIMD on W0–W31 registers (independent from FPRs):

```asm
; 4×i32 add
addv.w    $w0, $w1, $w2

; 4×f32 fused multiply-add
fmadd.w   $w0, $w1, $w2    ; $w0 = $w0 + $w1 * $w2

; 16×i8 compare (signed greater-than)
clt_s.b   $w5, $w6, $w7

; Byte shuffle using constant index
shf.b     $w4, $w5, 0x1B   ; reverse byte order within each 4-byte word
```

### 101.3.5 MIPS R6 Breaking Changes

MIPS Release 6 (2014) introduced significant architecture changes:

- **Removed**: `MFHI`/`MFLO` (multiply/divide) replaced with `MUL`/`MULU`/`MUHU`/`DIV`/`MOD`/`DIVU`/`MODU` with explicit register destinations
- **Removed**: Delay slots from most branches; replaced with "compact branches" (`BC`, `BALC`, `BEQZC`, etc.)
- **Added**: `LSA` (load scaled address), `DLSA`; `ALIGN`/`DALIGN` (byte-alignment)
- **Changed**: `LWL`/`LWR`/`SWL`/`SWR` (unaligned load/store) removed from privileged mode

LLVM handles R6 via the `-mips-r6` feature flag which activates alternative instruction patterns and disables delay slot scheduling.

---

## 101.4 SPARC and SPARC64

### 101.4.1 Register Windows

SPARC's most architecturally distinctive feature is its overlapping register windows. Each procedure call context has:
- **G registers** (g0–g7): global, g0=hardwired zero
- **O registers** (o0–o7): outputs; o6=SP, o7=return_PC-8 after CALL instruction
- **L registers** (l0–l7): local, strictly caller-private
- **I registers** (i0–i7): inputs; i6=frame pointer, i7=return address after SAVE

The `SAVE %sp, -frame_size, %sp` instruction slides the window: the caller's O registers become the callee's I registers, and a fresh set of L and O registers is activated. `RESTORE` slides back.

```asm
; SPARC function with register window
foo:
    save   %sp, -96, %sp    ! rotate window, allocate 96-byte frame
                            ! caller's %o0 is now %i0 (arg 0)
    mov    %i0, %o0         ! forward arg0 to next call
    call   bar              ! leaves return address in %o7
    nop                     ! delay slot
    mov    %o0, %i0         ! return value of bar → our return value
    ret                     ! jmpl %i7+8, %g0 (return via %i7)
    restore                 ! delay slot: slide window back
```

When all 8 (SPARC-V8) or 32 (SPARC-V9) register windows are exhausted, a window overflow trap saves the oldest window to the stack (`FLUSHW` on SPARC-V9).

### 101.4.2 SPARC V9 / SPARC64

SPARC V9 extends to 64-bit addressing and expands the FPR file:
- 32 single-precision S0–S30 (even-numbered: aliased to lower half of doubles)
- 32 double-precision D0–D30 (even-numbered)
- 16 quad-precision Q0–Q28 (multiples of 4)
- 64-bit integer in O/L/I/G registers

The 64-bit Linux ABI (`sparc64-linux-gnu`):
- **Integer args**: o0–o5 (first 6)
- **FP args**: f1–f5 (first 5 single-precision), f2–f6 (double-precision)
- **Return**: o0 (integer), f0 (FP)

### 101.4.3 LEON Embedded SPARC

LEON (from Aeroflex Gaisler, now Cobham) implements SPARC-V8 for space and safety-critical systems:
- Used on ESA spacecraft: Rosetta, Mars Express, BepiColombo
- LEON3/LEON4 variants implement a configurable number of register windows (typically 8)
- Radiation-hardened or radiation-tolerant versions available

LLVM's SPARC backend includes LEON-specific workarounds (`-mcpu=leon3`, `-mcpu=leon4`):

```cpp
// SparcTargetMachine.cpp
// LEON3 lacks certain SPARC V8+ instructions; LLVM avoids them
void SparcTargetLowering::adjustSubtargetForLEON(
    const SparcSubtarget &ST, MachineFunction &MF) const {
  // Disable errata-affected instructions for LEON3
}
```

---

## 101.5 LoongArch

### 101.5.1 Architecture Overview

LoongArch was designed by Loongson Technology Corporation (STCA, Guangdong Province) and published in 2021. It is a 32/64-bit RISC ISA inspired by MIPS64 and RISC-V but with distinct instruction encoding and ABIs.

Core ISA features:
- **LA32**: 32-bit base ISA (RV32-class)
- **LA64**: 64-bit base ISA with pointer and long types both 64-bit
- **Clean RISC design**: 64 registers (not Windows-centric), no branch delay slots
- **Integer only baseline**: FP is optional (but present on all production chips)

### 101.5.2 Register Architecture

LoongArch LA64 has:

| Register range | ABI name | Role |
|---------------|----------|------|
| r0 | zero | Hardwired zero |
| r1 | ra | Return address |
| r2 | tp | Thread pointer (TLS base) |
| r3 | sp | Stack pointer |
| r4–r11 | a0–a7 | Function arguments / return values |
| r12–r20 | t0–t8 | Temporaries (caller-saved) |
| r21 | (reserved) | Platform specific |
| r22 | fp/s9 | Frame pointer (callee-saved) |
| r23–r31 | s0–s8 | Callee-saved |

FPRs:
- f0–f7 (fa0–fa7): FP arguments and return values
- f8–f23 (ft0–ft15): FP temporaries
- f24–f31 (fs0–fs7): FP callee-saved

SIMD registers:
- vr0–vr31: 128-bit LSX (LoongArch SIMD eXtension)
- xr0–xr31: 256-bit LASX (LoongArch Advanced SIMD eXtension)

### 101.5.3 LP64D ABI

The primary Linux ABI for LoongArch (LP64D):

- **Integer args**: a0–a7 (r4–r11) for up to 8 integer/pointer arguments
- **FP args**: fa0–fa7 (f0–f7) for up to 8 `float`/`double` arguments
- **Return**: a0[+a1 for 128-bit], fa0[+fa1 for complex]
- **Callee-saved**: s0–s8 (r23–r31), fs0–fs7 (f24–f31), fp (r22)
- **Stack alignment**: 16-byte at call boundaries

Homogeneous FP aggregates (≤4 elements of same FP type) are passed in consecutive fa registers, analogous to AArch64 HFA.

### 101.5.4 LoongArch SIMD: LSX and LASX

**LSX (128-bit)**:

```asm
; 4×i32 add
vadd.w    $vr0, $vr1, $vr2

; 4×f32 fused multiply-add
vfmadd.s  $vr0, $vr1, $vr2, $vr3   ; vr0 = vr1*vr2 + vr3

; Byte shuffle (SSSE3 PSHUFB equivalent)
vshuf.b   $vr0, $vr1, $vr2, $vr3   ; general byte permute

; Horizontal add
vhaddw.w.h $vr0, $vr1, $vr2         ; widen-and-add adjacent halfwords
```

**LASX (256-bit)**:

```asm
; 8×i32 add using 256-bit registers
xvadd.w   $xr0, $xr1, $xr2

; 8×f32 multiply
xvfmul.s  $xr3, $xr4, $xr5

; 256-bit lane permute
xvperm.w  $xr0, $xr1, $xr2
```

LLVM patterns in `LoongArchInstrLSX.td` and `LoongArchInstrLASX.td`:

```tablegen
// LSX 4×f32 add
def : Pat<(v4f32 (fadd VR:$src1, VR:$src2)),
          (VFADD_S VR:$src1, VR:$src2)>;

// LASX 8×f32 multiply
def : Pat<(v8f32 (fmul XR:$src1, XR:$src2)),
          (XVFMUL_S XR:$src1, XR:$src2)>;
```

### 101.5.5 LBT: Binary Translation Helpers

The LBT (LoongArch Binary Translation) extension provides shadow registers that mirror x86 EFLAGS and ARM CPSR, enabling efficient dynamic binary translation:

```asm
; Shadow x86 EFLAGS update from an add operation
x86add.w   $r4, $r5        ; perform add and set x86 CF/OF/SF/ZF/PF
x86mtflag  $r0, 0x3f       ; set x86 flags from register

; Read condition flags
movfcsr2gr $r0, $fcsr3     ; move FP condition flag register to GPR

; ARM condition code shadow
armsetflags $r1             ; set ARM N/Z/C/V flags from r1[3:0]
```

These instructions are used by QEMU (which runs LoongArch host), ExaGear, and Loongson's commercial binary translators to achieve near-native performance when running x86 or ARM applications.

### 101.5.6 LLVM LoongArch Maturity

The LoongArch backend was merged in LLVM 14.0 (2022) and has progressed rapidly. As of LLVM 22.1.x:

- **Production-ready**: LA64 base ISA + D (double FP) + LSX + LASX
- **SelectionDAG**: Complete and production-ready
- **GlobalISel**: In-progress; basic integer and FP operations supported
- **LTO**: Full support via ThinLTO pipeline
- **Sanitizers**: ASan, UBSan, TSan, MSSan ports available

```bash
# LoongArch LA64 with LASX
clang --target=loongarch64-linux-gnu \
    -march=la64+lasx \
    -O3 foo.c -o foo

# Low-level verification
/usr/lib/llvm-22/bin/llc -march=loongarch64 \
    -mattr=+d,+lsx,+lasx \
    input.ll -o output.s

# Check LASX vectorization
clang --target=loongarch64-linux-gnu \
    -march=la64+lasx -O3 \
    -fopt-info-vec-all \
    saxpy.c -S -o saxpy.s 2>&1 | grep "vectorized"
```

---

## 101.6 LoongArch in Depth

The LoongArch backend, introduced in LLVM 14 and reaching tier-1 status by LLVM 18, has undergone substantial development in LLVM 22. This section provides the depth missing from §101.5's survey: TLS internals, SIMD coverage, linker details, and JIT support.

### 101.6.1 ISA Overview: LoongArch v1.1

LoongArch v1.1 (published 2023 by the LoongArch International Standards Organization) is the current baseline for production deployments. The ISA has a clean RISC design inspired by MIPS64 and AArch64 but with distinct encoding:

- **Instruction width**: Fixed 32-bit (4-byte aligned). No compressed encoding variant (unlike RISC-V's RVC or ARM Thumb).
- **Register file**: 32 GPRs (r0–r31), each 64-bit in LA64. r0 is hardwired zero. No HI/LO registers or delay slots inherited from MIPS ancestry.
- **Addressing modes**: Base+immediate (signed 12-bit), PC-relative (for globals and branches). No complex memory addressing modes.
- **Branch encoding**: PC-relative, ranging ±128 MB (26-bit offset) for direct branches; 16-bit offset for conditional branches (±128 KB range, extended via `PCADDI` + `JIRL` pattern).
- **Privileged architecture**: CSR registers (Control and Status Registers) for privilege levels 0 (kernel) through 3 (user), analogous to RISC-V's CSR model.

The LA64 baseline (64-bit) is the only variant shipped in Loongson 3A5000/3A6000 processors. The 32-bit LA32 variant exists in the specification but has no production silicon.

```bash
# Query LoongArch features on a Loongson 3A6000
cat /proc/cpuinfo | grep -E "model|Features"
# Features: cpucfg lam ual fpu lsx lasx complex crypto lbt lvz
```

The `cpucfg` (CPU configuration) instruction allows software to query available extensions at runtime, analogous to CPUID on x86 or system register reads on AArch64.

### 101.6.2 TLS Models Including TLSDESC

LoongArch supports all four standard ELF TLS access models, plus the TLSDESC (Thread Local Storage Descriptor) model added to the LLVM LoongArch backend in 2024.

**Global Dynamic (GD)**: Standard TLS, requires two GOT entries per variable and a call to `__tls_get_addr`. Suitable for shared libraries where the TLS variable may come from another DSO.

**Local Dynamic (LD)**: Optimized for accessing multiple TLS variables from the same module; a single `__tls_get_addr` call retrieves the module's TLS base, and individual variables are accessed via offsets from it.

**Initial Exec (IE)**: For TLS variables in a DSO loaded at process startup. The TLS offset is stored in the GOT; a single GOT load gives the offset, which is added to the thread pointer (`tp`, r2).

**Local Exec (LE)**: For TLS variables in the main executable. The offset is known at link time; a single add-immediate from the thread pointer suffices. The fastest model.

```asm
; Local Exec TLS access (fastest path — main executable only)
lu12i.w  $r4, %le_hi20(var)     ; load upper 20 bits of TLS offset
ori      $r4, $r4, %le_lo12(var); or in lower 12 bits
add.d    $r4, $r2, $r4           ; TLS_ptr = thread_ptr + offset
ld.d     $r0, $r4, 0             ; load TLS variable

; Initial Exec TLS access (DSO, loaded at startup)
pcaddu12i $r4, %ie_pc_hi20(var) ; PC-relative GOT entry address
ld.d      $r4, $r4, %ie_lo12(var) ; load TLS offset from GOT
add.d     $r4, $r2, $r4           ; TLS_ptr = thread_ptr + offset
```

**TLSDESC (Thread Local Storage Descriptor)**: Added to the LoongArch ABI in 2023 and implemented in the LLVM LoongArch backend in LLVM 18–19, TLSDESC replaces the `__tls_get_addr` call overhead with a descriptor-based lazy-binding mechanism. The TLS descriptor contains a function pointer and a data word; on first access, the dynamic linker fills in the function pointer with a fast resolver; subsequent accesses call the resolver directly without going through `__tls_get_addr`.

```asm
; TLSDESC TLS access (lazy-binding, lower overhead at runtime)
pcaddu12i  $r4, %desc_pc_hi20(var)  ; load descriptor GOT entry
addi.d     $r4, $r4, %desc_lo12(var)
ld.d       $r12, $r4, %desc_ld(var) ; load descriptor function pointer
jirl       $r1, $r12, %desc_call(var); call resolver (clobbers r1=ra)
add.d      $r4, $r2, $r0            ; TLS_ptr = thread_ptr + r0 (return)
```

TLSDESC requires linker support for the `R_LARCH_TLS_DESC_PC_HI20`, `R_LARCH_TLS_DESC_PC_LO12`, `R_LARCH_TLS_DESC_LD`, and `R_LARCH_TLS_DESC_CALL` relocations, all present in LLD 18+. The GCC LoongArch toolchain added TLSDESC support contemporaneously.

LLVM selects the TLS model based on the `-ftls-model` flag and linkage:

```bash
# Force TLSDESC for all TLS accesses
clang --target=loongarch64-linux-gnu \
    -ftls-model=global-dynamic \
    -mllvm -loongarch-enable-tlsdesc \
    -O2 foo.c -o foo.so
```

### 101.6.3 Vector Extensions: LSX and LASX in Detail

Beyond the instruction-level survey in §101.5.4, the LLVM auto-vectorizer pipeline for LoongArch warrants examination.

**LSX (LoongArch SIMD eXtension, 128-bit)**:

LSX provides 32 vector registers (`vr0`–`vr31`), each 128 bits. The design is clearly influenced by SSE/AVX: element-wise arithmetic, integer saturation, horizontal operations, and byte-level shuffles all have direct equivalents. LLVM's `LoongArchInstrLSX.td` defines approximately 700 LSX instruction patterns.

The auto-vectorizer activates LSX when `+lsx` is in the feature string. Vectorization decisions use the `LoongArchTTI` (Target Transform Info) cost model:

```cpp
// LoongArchTargetTransformInfo.cpp
InstructionCost LoongArchTTIImpl::getArithmeticInstrCost(
    unsigned Opcode, Type *Ty, ...) const {
  // LSX: v4i32, v4f32 → cost 1
  // LASX: v8i32, v8f32 → cost 1
  // i128: not directly supported → scalarize
}
```

**LASX (LoongArch Advanced SIMD eXtension, 256-bit)**:

LASX provides 32 wide registers (`xr0`–`xr31`), each 256 bits. The lower 128 bits of each `xr` register alias the corresponding `vr` register — an important detail for mixing LSX and LASX code.

The LLVM vectorizer generates LASX code for target machines with the Loongson 3A6000 CPU model (`-mcpu=la664`). The scheduler model `LoongArchScheduleLA664.td` reflects the 3A6000's 4-issue out-of-order pipeline with dual 256-bit vector execution units.

```bash
# Verify LASX vectorization of a kernel loop
clang --target=loongarch64-linux-gnu \
    -mcpu=la664 -O3 \
    -Rpass=loop-vectorize \
    saxpy.c -S -o saxpy.s 2>&1 | head -20

# Inspect generated LASX instructions
grep -E "^	xv" saxpy.s
```

### 101.6.4 LLD and RuntimeDyld Support

**LLD LoongArch support** covers:
- Full static and dynamic ELF linking
- `R_LARCH_ALIGN` relaxation: LoongArch requires instruction bundles to be naturally aligned; `R_LARCH_ALIGN` allows the assembler to emit NOP padding that the linker later removes during relaxation when sections are compacted
- GOT/PLT generation for all four TLS models and TLSDESC
- Relaxation of `PCADDU12I + LD.D` GOT sequences to `PCADDI` + offset when the target is within ±2 GB (medium code model range)

```bash
# Verify LLD is handling R_LARCH_ALIGN relaxation
ld.lld --target=loongarch64-linux-gnu \
    -z relro --hash-style=gnu \
    -dynamic-linker /lib64/ld-linux-loongarch-lp64d.so.1 \
    crt1.o crtbegin.o foo.o -lc crtend.o -o foo \
    --verbose 2>&1 | grep "ALIGN"
```

**RuntimeDyld LoongArch support** (added in LLVM 18–19): enables the LLVM JIT infrastructure (`lli`, `ExecutionEngine`, and `LLJIT`) to load and execute LoongArch ELF objects dynamically. This is essential for:
- The `lli` interpreter running LoongArch bitcode
- LLVM-based JIT compilers (MLIR, Julia, etc.) targeting LoongArch hardware
- `LLJITBuilder` / `ORCJit` pipelines on Loongson hardware

```cpp
// Use LLJIT on LoongArch
auto JITB = LLJITBuilder().create();
auto &J = *JITB;
// Add a LoongArch object file to the JIT
auto ObjBuf = MemoryBuffer::getFile("module.o");
J.addObjectFile(std::move(*ObjBuf));
// Look up a function symbol
auto Sym = J.lookup("my_func");
// Call it via function pointer
auto Fn = Sym->toPtr<int(*)(int)>();
int result = Fn(42);
```

### 101.6.5 Production Status

As of LLVM 22.1.x, the LoongArch backend has reached tier-1 status:

- **Hardware**: Loongson 3A5000 (la464 microarchitecture, 4-wide OOO), 3A6000 (la664, 6-wide OOO with LASX), shipping in consumer laptops (Lemote 4000, Dragon Laptop) and server blade modules.
- **OS support**: Linux kernel 6.0+ has full LoongArch support; LoongArch is an official tier-1 architecture in Debian, Gentoo, and Arch Linux.
- **LLVM features**: SelectionDAG complete; GlobalISel covering basic integer and FP (in-progress for SIMD); LTO via ThinLTO; all four standard sanitizers (ASan, TSan, UBSan, MSan).
- **LLVM source**: `llvm/lib/Target/LoongArch/` (main backend), `llvm/lib/MC/MCTargetDesc/LoongArch*.cpp` (MC layer), `llvm/lib/Target/LoongArch/LoongArchInstrLSX.td`, `LoongArchInstrLASX.td` (SIMD patterns).

```bash
# Full LoongArch LA64 + LASX build and check
clang --target=loongarch64-linux-gnu \
    -mcpu=la664 \
    -march=la64+lasx \
    -O3 -funroll-loops \
    gemm.c -o gemm_la64

# RuntimeDyld: run LoongArch LLVM IR via lli
/usr/lib/llvm-22/bin/lli --mtriple=loongarch64-linux-gnu \
    --march=loongarch64 \
    --mcpu=la664 \
    module.ll
```

---

## 101.7 VE: NEC SX-Aurora Tsubasa

The NEC SX-Aurora Tsubasa is a vector processor architecture designed for HPC and AI inference, implemented in the NEC Vector Engine (VE) accelerator cards. Unlike GPU accelerators, VE is a conventional scalar-vector processor that runs a standard Linux ABI (`ve-linux`) and can execute general-purpose programs with very wide SIMD.

### Architecture Overview

The VE ISA features:
- **256-element vector registers**: 64 vector registers (v0–v63), each holding 256 64-bit elements (256 × 64-bit = 16 KB per register). This is the widest SIMD register file in any LLVM-supported target.
- **256-lane scalar-vector execution**: Vector operations process all 256 lanes simultaneously, achieving extremely high throughput for array-heavy numerical codes.
- **Scalar registers**: 64 general-purpose scalar registers (s0–s63), 64-bit.
- **Vector mask registers (VM)**: 16 vector mask registers (vm0–vm15), one bit per lane.
- **512-bit scalar operations**: VE includes 512-bit FP operations for extended precision (not SIMD — single values).

The VE is a PCIe accelerator card; the host CPU (x86) communicates with VE via DMA and the `veoffload` library. Programs execute natively on the VE Linux environment (a separate Linux kernel instance running on the VE).

### LLVM VE Backend

The LLVM VE backend (`llvm/lib/Target/VE/`) was contributed by NEC and merged in LLVM 11. Key features:

- **SelectionDAG**: Complete for scalar and basic vector operations.
- **Vectorization**: The standard LLVM loop vectorizer generates VE vector instructions; the `VLEN` (vector length) is controlled by `setvl` and tracked via `VL` (Vector Length register).
- **llvm-ve-rv project**: An independent research project building advanced VE vectorization passes on top of LLVM, providing more aggressive loop transformations than the standard vectorizer.

```bash
# Compile for NEC SX-Aurora VE
clang --target=ve-linux \
    -O3 \
    -fvectorize \
    saxpy.c -o saxpy_ve

# Inspect VE vector instructions
/usr/lib/llvm-22/bin/llc -march=ve \
    -mattr=+ve \
    module.ll -o module_ve.s
grep -E "^	(vld|vst|vaddd|vmulw)" module_ve.s
```

A representative VE vector add loop (256-element chunks):

```asm
; VE: saxpy loop (y[i] += a * x[i])
; Assumes N divisible by 256 for clarity
.loop:
    lvl       256             ; set vector length to 256
    vld       %v0, 8(%s1)    ; load 256 doubles from x[i]
    vfmadd.d  %v1, %s0, %v0, %v1  ; v1 += a * v0 (fused MA)
    vst       %v1, 8(%s2)    ; store 256 doubles to y[i]
    addu.l    %s1, 2048, %s1 ; advance x pointer by 256*8 bytes
    addu.l    %s2, 2048, %s2 ; advance y pointer
    subu.l    %s3, %s3, 256  ; decrement count
    bgt       %s3, %s3, .loop
```

### Status in LLVM 22

The VE backend is a **tier-3 target** in LLVM 22.1.x. This means:
- It is maintained and does not break the tree, but is not tested in the main CI pipeline on physical hardware.
- NEC maintains the backend and provides CI resources.
- Vectorization support is functional but less aggressively optimized than tier-1 targets.
- GlobalISel is not implemented; SelectionDAG is the only codegen path.

---

## 101.8 SystemZ z/OS Support

The SystemZ backend in LLVM has historically targeted Linux on IBM Z (s390x-linux-gnu). LLVM 22 extends support to z/OS, IBM's proprietary mainframe operating system, adding the `s390x-ibm-zos` triple and the associated object format and character encoding requirements.

### z/OS Triple and EBCDIC

z/OS uses EBCDIC (Extended Binary Coded Decimal Interchange Code) rather than ASCII for character data. This has a pervasive impact on the compiler:

- **String literals**: A C string `"hello"` compiled for z/OS must encode characters in EBCDIC. `'h'` = 0x88 (EBCDIC) vs 0x68 (ASCII).
- **`<ctype.h>` tables**: `isalpha()`, `isdigit()`, etc. must use EBCDIC character classification tables.
- **Clang z/OS string handling**: Clang for `s390x-ibm-zos` emits EBCDIC-encoded string literals. The `ZOSTargetInfo` class in `clang/lib/Basic/Targets/SystemZ.cpp` sets `CharIsSigned = true` and activates the EBCDIC code page translation.

```cpp
// clang/lib/Basic/Targets/SystemZ.cpp
class ZOSTargetInfo : public SystemZTargetInfo {
public:
  ZOSTargetInfo(const llvm::Triple &Triple, const TargetOptions &Opts)
      : SystemZTargetInfo(Triple, Opts) {
    // EBCDIC character set
    resetDataLayout(
        "E-m:e-i1:8:16-i8:8:16-i64:64-f128:64-v128:64-a:8:16-n32:64");
    // z/OS-specific ABI adjustments
    LongDoubleWidth = LongDoubleAlign = 128;
  }
};
```

### GOFF: Generalized Object File Format

z/OS uses GOFF (Generalized Object File Format) rather than ELF for object files. GOFF is IBM's proprietary object format that has been in use since the OS/390 era. It provides:
- **External Symbol Dictionary (ESD)**: Declares external symbols and their properties (analogous to ELF symbol tables)
- **Relocation Dictionary (RLD)**: Describes relocations (analogous to ELF `.rela` sections)
- **Text (TXT)**: Contains the actual code and data bytes

LLVM 22 includes a basic GOFF object writer in `llvm/lib/MC/GOFFObjectWriter.cpp` (contributed via the IBM z/OS Clang project). The writer handles:
- GOFF ESD records for function symbols and data symbols
- GOFF TXT records for code/data sections
- GOFF RLD records for relocations

```bash
# Compile a simple file for z/OS (requires z/OS SDK headers)
clang --target=s390x-ibm-zos \
    -fno-integrated-as \
    -mabi=lp64 \
    hello.c -o hello.o

# Inspect GOFF object (IBM utility, z/OS only)
# iobind /DUMP HELLO.OBJ
```

### LLD GOFF Backend

LLD 22 includes the beginning of a GOFF linker backend in `lld/GOFF/`. The implementation in LLVM 22 supports:
- Linking simple programs with a small number of GOFF object files
- GOFF symbol resolution and relocation processing
- Generating a z/OS load module (the z/OS executable format, analogous to an ELF executable)

Full z/OS system library support (linking against `CEE` runtime, `SYS1.CSSLIB`, and `DSNHLI`) requires the IBM z/OS SDK, which provides the system headers and libraries. The LLVM-provided toolchain handles the compilation stage; linking against system libraries requires the IBM-provided binder (`IEWL`) or the LLD GOFF backend with the system libraries accessible.

### z/OS ABI Notes

The z/OS 64-bit ABI (LP64 mode) is largely consistent with the s390x Linux ABI described in §101.2.3, with the following differences:
- **No POSIX signal handling** in the conventional sense; z/OS uses condition tokens and SRBs (Service Request Blocks) for exception handling.
- **RENT (re-entrant) code**: z/OS programs are conventionally compiled as re-entrant (`-fno-common`, no writable static storage in the code object). This aligns with LLVM's default behavior.
- **AMODE 64 / RMODE ANY**: z/OS programs specify addressing mode (64-bit) and residence mode (anywhere in memory) in the object file header, encoded in GOFF ESD records.

---

## 101.9 Cross-Compilation Reference

### 101.9.1 Compilation Flags

```bash
# PowerPC POWER10 (little-endian, ELFv2)
clang --target=powerpc64le-linux-gnu \
    --sysroot=/opt/cross/ppc64le-sysroot \
    -mcpu=power10 -maltivec -mvsx -mpower10-mma \
    -O3 foo.c -o foo_ppc64le

# SystemZ z15 (Linux)
clang --target=s390x-linux-gnu \
    --sysroot=/opt/cross/s390x-sysroot \
    -march=z15 -mvx \
    -O2 foo.c -o foo_z15

# SystemZ z/OS (GOFF output)
clang --target=s390x-ibm-zos \
    -mabi=lp64 \
    -O2 foo.c -o foo_zos.o

# MIPS64 n64 ABI
clang --target=mips64el-linux-gnuabi64 \
    --sysroot=/opt/cross/mips64el-sysroot \
    -mabi=n64 -march=mips64r2 -mmsa \
    -O2 foo.c -o foo_mips64

# SPARC64
clang --target=sparc64-linux-gnu \
    --sysroot=/opt/cross/sparc64-sysroot \
    -mcpu=ultrasparc3 \
    -O2 foo.c -o foo_sparc64

# LoongArch LA64 with LASX and TLSDESC
clang --target=loongarch64-linux-gnu \
    --sysroot=/opt/cross/la64-sysroot \
    -march=la64+lasx \
    -ftls-model=global-dynamic \
    -O3 foo.c -o foo_la64

# NEC SX-Aurora VE
clang --target=ve-linux \
    -O3 -fvectorize \
    foo.c -o foo_ve
```

### 101.9.2 Target-Specific IR Inspection

```bash
# PowerPC: verify VSX vectorization
/usr/lib/llvm-22/bin/llc -march=ppc64le \
    -mcpu=power9 -mattr=+vsx,+altivec \
    test.ll -o test_ppc.s
grep -E "xvadd|xvmul|vadduwm" test_ppc.s

# SystemZ Linux: verify vector instructions
/usr/lib/llvm-22/bin/llc -march=systemz \
    -mcpu=z14 -mattr=+vector \
    test.ll -o test_z.s
grep -E "^	v[a-z]" test_z.s

# MIPS64 MSA: verify SIMD
/usr/lib/llvm-22/bin/llc -march=mips64el \
    -mcpu=mips64r5 -mattr=+msa \
    test.ll -o test_mips.s
grep -E "addv\.|fmadd\." test_mips.s

# LoongArch LASX
/usr/lib/llvm-22/bin/llc -march=loongarch64 \
    -mattr=+d,+lsx,+lasx \
    test.ll -o test_la.s
grep -E "^	xv" test_la.s

# NEC VE: verify vector instructions
/usr/lib/llvm-22/bin/llc -march=ve \
    test.ll -o test_ve.s
grep -E "^	v(ld|st|add|mul)" test_ve.s
```

---

## 101.10 Comparative Architecture Summary

| Target | ISA class | SIMD width | Primary SIMD unit | GlobalISel | OOO scheduling |
|--------|----------|-----------|------------------|------------|---------------|
| PowerPC (ppc64le) | RISC | 128–512-bit | AltiVec/VSX/MMA | Partial | POWER8–10 models |
| SystemZ Linux (s390x) | CISC | 128-bit | VECTOR (z13+) | No | z13–z16 models |
| SystemZ z/OS (s390x) | CISC | 128-bit | VECTOR (z13+) | No | z13–z16 models |
| MIPS32/64 | RISC | 128-bit | MSA | No | P5600, I6400 |
| SPARC64 | RISC | 64-bit | VIS (limited) | No | HyperSPARC |
| LoongArch LA64 | RISC | 256-bit | LSX/LASX | Partial | Loongson 3A6000 |
| NEC VE | RISC+Vector | 256-element | 256-lane VR | No | SX-Aurora VE1/VE2 |

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **LoongArch GlobalISel completion**: The LoongArch GlobalISel port is actively expanding; LLVM 23 is expected to bring coverage of LSX/LASX vector operations through GlobalISel, closing the gap with the complete SelectionDAG path. Track patches on `llvm/lib/Target/LoongArch/GISel/` and the LLVM Discourse `[LoongArch]` tag.
- **Power10 MMA auto-vectorization**: IBM is upstreaming passes that recognize BLAS-pattern loops and automatically lower them to Power10 MMA `xvf32gerpp`/`xvf64gerpp` outer-product sequences without manual intrinsics. Related RFC: "PowerPC MMA auto-detection in SLP vectorizer" (discourse.llvm.org, 2025).
- **SystemZ z16 full instruction coverage**: IBM Z z16 (2022) added the NNPA (Neural Network Processing Assist) instruction set; LLVM patches to expose `NNPA` instructions as target intrinsics and lower `tosa`/`linalg` matmul ops to NNPA are in review as of early 2026.
- **LoongArch TLSDESC default enablement**: The `-mllvm -loongarch-enable-tlsdesc` flag is expected to become the default for shared-library TLS in Clang 23, mirroring AArch64's TLSDESC promotion; the Loongson toolchain team is coordinating with the glibc LoongArch port maintainers for dynamic linker readiness.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **LoongArch v2.0 ISA extensions**: The LoongArch International Standards Organization is developing v2.0 extensions including LVZ (LoongArch Virtualization) and LCT (LoongArch Cryptography), with hardware shipping in the Loongson 3B6000/3C6000 server class. LLVM support will follow initial silicon tape-out, likely requiring new `LoongArchInstrLVZ.td` and `LoongArchInstrCrypto.td` files analogous to the existing LSX/LASX pattern.
- **MIPS backend maintenance status**: LLVM's MIPS backend is at risk of demotion from tier-2 to tier-3 given declining upstream maintainer bandwidth. The LLVM Technical Council is expected to publish a formal policy on backend tiers; MIPS R6 microMIPS and MIPS16 support may be removed as orphaned code unless new maintainers step forward (Discourse thread: "MIPS backend future" by @lenary, 2025).
- **PowerPC ELFv2 on OpenPOWER 5**: IBM is defining the POWER5 microarchitecture ABI extensions for POWER11 generation hardware (internal codename "Salazar"), including expanded MMA accumulator banks and a new form of the VSX register file addressing. LLVM codegen for `power11` (`-mcpu=power11`) is expected to begin upstreaming by 2027.
- **NEC VE2 backend improvements**: NEC's SX-Aurora VE2 (second-generation vector engine) achieves 2× the vector register bandwidth of VE1; the `llvm-ve-rv` research project (github.com/sx-aurora-dev/llvm-ve-rv) is integrating its outer-loop vectorizer passes into upstream LLVM to enable VE2 targets to vectorize non-innermost loops and achieve 80–90% of peak hardware throughput.

### 5-Year Horizon (Long-Term, by ~2031)

- **LoongArch as a first-class LLVM development target**: As Loongson 3A7000/3C7000 hardware proliferates in Chinese data centers, LoongArch is on a trajectory to become a tier-1 LLVM target with dedicated bot coverage, GlobalISel parity, and MLIR codegen backends for `vector` and `gpu` dialects analogous to AArch64 SVE support.
- **SystemZ quantum-classical hybrid compilation**: IBM's quantum computing roadmap for 2029–2031 ("IBM Quantum System Two") includes hybrid classical-quantum programs where the classical control code runs on IBM Z. Research into LLVM IR extensions that model quantum gates as side-effectful intrinsics, with SystemZ as the host backend for the classical scheduler, is ongoing in IBM Research.
- **SPARC sunset and LLVM deprecation**: SPARC has no active silicon roadmap beyond existing safety-critical LEON4/GR740 radiation-hardened variants. By 2031, SPARC is likely to follow the path of the Alpha backend (removed LLVM 3.5) — moved out-of-tree or placed on formal deprecation notice unless the space-systems embedded community (Cobham, CAES) takes active maintainership.

---

## Chapter Summary

- PowerPC's VSX register file (VS0–VS63) unifies FPRs and VMX vector registers; ELFv2 (ppc64le) eliminates function descriptors and passes up to 8 integer args in R3–R10, 13 FP args in F1–F13, and 12 vector args in V2–V13; Power10 MMA adds 512-bit accumulator registers for matrix outer-product instructions exploited by LLVM's modulo scheduler.
- SystemZ (s390x) has 16 GPR even-odd pairs for 128-bit operations, V0–V31 128-bit vector registers aliasing FPRs (z13+), packed decimal instructions for COBOL/financial workloads, EXECUTE for parametric string operations, and the Transaction Execution facility for hardware transactional memory.
- MIPS has three principal ABIs (o32/n32/n64) distinguished by argument register count and struct passing; MSA provides 128-bit SIMD on independent W0–W31 registers; MIPS R6 replaces HI/LO multiply with regular register results and introduces compact branches without delay slots.
- SPARC's register-window architecture overlaps caller O registers with callee I registers via `SAVE`/`RESTORE`, passing arguments without memory; LEON3/4 variants target space-qualified embedded SPARC-V8 systems with LLVM workarounds for LEON-specific errata.
- LoongArch LA64 is a modern clean-slate RISC ISA with LP64D ABI (integer args in a0–a7, FP args in fa0–fa7), 128-bit LSX and 256-bit LASX SIMD, and LBT shadow registers for x86/ARM flag emulation in binary translators; the LLVM backend has been production-ready since LLVM 14.
- Cross-compilation for all five targets uses `--target=<triple> --sysroot=<path>` with LLVM's integrated assembler; the `llc -march=<arch> -mattr=+<features>` pipeline verifies codegen output for each target.
- Among these targets, only PowerPC and LoongArch have partial GlobalISel support; the others rely on the mature SelectionDAG path; LoongArch GlobalISel coverage is expanding rapidly with each LLVM release.
- LoongArch v1.1 (LA64) is a clean fixed-width RISC ISA with LP64D ABI; beyond the §101.5 survey, TLSDESC (added 2024) provides lazy-binding TLS without `__tls_get_addr` call overhead; 128-bit LSX and 256-bit LASX auto-vectorization targets the Loongson 3A6000 (la664 CPU model); RuntimeDyld support enables LLVM JIT infrastructure on LoongArch; LLD handles `R_LARCH_ALIGN` relaxation and all four TLS models plus TLSDESC.
- The NEC SX-Aurora VE backend provides the widest SIMD registers in LLVM: 64 × 256-element 64-bit vector registers processed in 256 lanes simultaneously; it is a tier-3 target maintained by NEC; the `llvm-ve-rv` project provides advanced vectorization passes beyond the standard loop vectorizer.
- SystemZ z/OS support (triple `s390x-ibm-zos`) in LLVM 22 adds EBCDIC string literal encoding, GOFF (Generalized Object File Format) object emission via `llvm/lib/MC/GOFFObjectWriter.cpp`, and the beginning of an LLD GOFF linker backend; z/OS programs are conventionally RENT (re-entrant) and specify AMODE 64 / RMODE ANY in GOFF ESD records.


---

@copyright jreuben11
