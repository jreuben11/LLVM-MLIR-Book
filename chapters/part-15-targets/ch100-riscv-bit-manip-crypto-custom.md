# Chapter 100 — RISC-V Bit-Manipulation, Crypto, and Custom Extensions

*Part XV — Targets*

One of RISC-V's most distinctive design choices is its extension composability: a vendor can take RV64GC as a base and layer on exactly the bit-manipulation, cryptography, half-precision floating-point, CFI, or custom instructions needed for the target workload. LLVM's RISC-V backend handles this through a systematic feature-flag architecture where each extension family adds TableGen patterns, `SubtargetFeature` definitions, clang driver flags, and optional intrinsics. This chapter covers the complete picture: the Zb* bit-manipulation suite, the Zk cryptography extensions, Zfh/Zfa floating-point additions, Zicfilp/Zicfiss CFI, the Zc* compressed extensions, vendor-specific extensions, and a step-by-step walkthrough of adding a custom extension to LLVM — from TableGen to the final instruction encoding.

---

## 100.1 Zba: Address Generation Instructions

Zba adds three scaled-add instructions targeting pointer arithmetic with strides of 2, 4, or 8 bytes. These eliminate the shift+add sequences that appear in every array indexed by an integer index on 64-bit systems.

### 100.1.1 Instruction Summary

| Instruction | Operation | Primary use case |
|-------------|-----------|-----------------|
| `SH1ADD rd, rs1, rs2` | rd = rs2 + (rs1 << 1) | int16_t array: base + i*2 |
| `SH2ADD rd, rs1, rs2` | rd = rs2 + (rs1 << 2) | int32_t/float array: base + i*4 |
| `SH3ADD rd, rs1, rs2` | rd = rs2 + (rs1 << 3) | int64_t/double/pointer: base + i*8 |
| `SH1ADD.UW rd, rs1, rs2` | rd = rs2 + (zext32(rs1) << 1) | 32-bit index, 16-bit stride |
| `SH2ADD.UW rd, rs1, rs2` | rd = rs2 + (zext32(rs1) << 2) | 32-bit index, 32-bit stride |
| `SH3ADD.UW rd, rs1, rs2` | rd = rs2 + (zext32(rs1) << 3) | 32-bit index, 64-bit stride |
| `ZEXT.W rd, rs1` | rd = zext32(rs1) | Provided by Zba on RV64 |

### 100.1.2 LLVM Patterns

```tablegen
// RISCVInstrInfoZb.td
def : Pat<(add GPR:$rs2, (shl GPR:$rs1, (i64 1))),
          (SH1ADD GPR:$rs1, GPR:$rs2)>;
def : Pat<(add GPR:$rs2, (shl GPR:$rs1, (i64 2))),
          (SH2ADD GPR:$rs1, GPR:$rs2)>;
def : Pat<(add GPR:$rs2, (shl GPR:$rs1, (i64 3))),
          (SH3ADD GPR:$rs1, GPR:$rs2)>;

// Zero-extended 32-bit index on RV64
def : Pat<(add GPR:$rs2, (shl (zext32 GPR:$rs1), (i64 2))),
          (SH2ADD_UW GPR:$rs1, GPR:$rs2)>;
```

### 100.1.3 Impact on Real Code

```c
// Array access: arr[i] for double arr[] (8-byte elements)
double array_get(double* arr, int i) {
    return arr[i];
}
// Without Zba:
//   slli  a1, a1, 3      ; shift index by 3 (×8)
//   add   a0, a0, a1     ; add to base
//   fld   fa0, 0(a0)     ; load
// With Zba:
//   sh3add a0, a1, a0    ; single instruction: a0 = a0 + a1*8
//   fld    fa0, 0(a0)
```

---

## 100.2 Zbb: Basic Bit Manipulation

Zbb is the core bit-manipulation extension, providing operations that required multiple instructions in base RISC-V.

### 100.2.1 Count and Population Operations

```c
// These map directly to Zbb instructions
int clz(uint64_t x)   { return __builtin_clzll(x);     }  // CLZ
int ctz(uint64_t x)   { return __builtin_ctzll(x);     }  // CTZ
int cpop(uint64_t x)  { return __builtin_popcountll(x); }  // CPOP
// RV32 variants (32-bit operands)
int clz32(uint32_t x) { return __builtin_clz(x);   }  // CLZW on RV64
int cpop32(uint32_t x){ return __builtin_popcount(x); }  // CPOPW on RV64
```

```tablegen
// RISCVInstrInfoZb.td: direct pattern matching
def : Pat<(ctlz   GPR:$rs), (CLZ  GPR:$rs)>;
def : Pat<(cttz   GPR:$rs), (CTZ  GPR:$rs)>;
def : Pat<(ctpop  GPR:$rs), (CPOP GPR:$rs)>;
// 32-bit variants on RV64
def : Pat<(i64 (ctlz (i64 (zext (i32 (trunc GPR:$rs)))  ))), (CLZW GPR:$rs)>;
```

### 100.2.2 Min/Max Operations

```asm
; Signed max: max(a0, a1) = a0 if a0 >= a1 else a1
MAX   a0, a0, a1     ; signed maximum
MIN   a0, a0, a1     ; signed minimum
MAXU  a0, a0, a1     ; unsigned maximum
MINU  a0, a0, a1     ; unsigned minimum
```

These map from `llvm.smax.i64`, `llvm.umin.i64`, etc. (which are produced by loop vectorization and scalar evolution passes). Without Zbb, each `max` requires a `SLT`+`SELECT` sequence.

### 100.2.3 Byte and Bit Operations

```asm
; Byte reverse (big-endian/little-endian swap)
REV8 a0, a0        ; a0 = bswap64(a0)  → __builtin_bswap64
; On RV32: reverses the 4 bytes of the word

; OR-combine bytes (test if any bit set in each byte)
ORC.B a0, a0       ; byte i of result = 0xFF if (a0 byte i) != 0 else 0
                   ; useful for: strlen (find zero byte)

; Rotate operations
ROL  a0, a0, a1   ; rotate left by a1 bits
ROR  a0, a0, a1   ; rotate right by a1 bits
RORI a0, a0, 16   ; rotate right by immediate 16
; On RV64: ROLW/RORW/RORIW for 32-bit operations
```

### 100.2.4 Zero and Sign Extension

```asm
; These replace multi-instruction sequences:
ZEXT.H  a0, a0    ; a0 = a0 & 0xFFFF (zero-extend 16-bit)
SEXT.B  a0, a0    ; sign-extend 8-bit: (a0 << 56) >> 56
SEXT.H  a0, a0    ; sign-extend 16-bit: (a0 << 48) >> 48
; (ZEXT.W is provided by Zba, not Zbb)
```

### 100.2.5 Additional Zbb Operations

```asm
; Andn, Orn, Xnor (bitwise NOT-AND, NOT-OR, NOT-XOR)
ANDN  a0, a1, a2  ; a0 = a1 & ~a2  (bit-clear)
ORN   a0, a1, a2  ; a0 = a1 | ~a2
XNOR  a0, a1, a2  ; a0 = ~(a1 ^ a2)
```

---

## 100.3 Zbc: Carry-Less Multiply

Zbc provides carry-less (GF(2)[x]) multiplication, the core operation of CRC computation and AES-GCM's GHASH function.

### 100.3.1 Instructions

| Instruction | Operation |
|-------------|-----------|
| `CLMUL rd, rs1, rs2` | Lower XLEN bits of carry-less product |
| `CLMULH rd, rs1, rs2` | Upper XLEN bits of carry-less product |
| `CLMULR rd, rs1, rs2` | Upper XLEN-1 bits, reversed (for CRC) |

### 100.3.2 CRC Implementation

CRC32 using Zbc with the reflected polynomial:

```c
#include <riscv_bitmanip.h>  // for __riscv_clmul*

// CRC32/CRC32C using Zbc CLMUL (Barrett reduction)
// Reference: Gopal et al., "Fast CRC Computation for iSCSI Data Streams"
static uint32_t crc32_byte_zbc(uint32_t crc, uint8_t data) {
    // Barrett reduction: quotient = (data XOR crc_low) * k / 2^32
    uint64_t x = (uint64_t)((crc ^ data) & 0xFF) << 24;
    uint64_t q = (uint64_t)__riscv_clmulh_32(
        (uint32_t)(x >> 32), CRC32_BARRETT_CONST);
    uint64_t r = x ^ ((uint64_t)__riscv_clmul_32(
        (uint32_t)q, CRC32_POLY) << 1);
    return (crc >> 8) ^ (uint32_t)(r >> 8);
}
```

---

## 100.4 Zbs: Single-Bit Operations

Zbs provides efficient single-bit set, clear, invert, and extract:

### 100.4.1 Instructions

| Instruction | Operation | C equivalent |
|-------------|-----------|-------------|
| `BSET rd, rs1, rs2` | rd = rs1 \| (1 << (rs2 & (XLEN-1))) | `rs1 \| (1UL << rs2)` |
| `BSETI rd, rs1, imm` | rd = rs1 \| (1 << imm) | `rs1 \| (1UL << imm)` |
| `BCLR rd, rs1, rs2` | rd = rs1 & ~(1 << (rs2 & (XLEN-1))) | `rs1 & ~(1UL << rs2)` |
| `BCLRI rd, rs1, imm` | rd = rs1 & ~(1 << imm) | `rs1 & ~(1UL << imm)` |
| `BINV rd, rs1, rs2` | rd = rs1 ^ (1 << (rs2 & (XLEN-1))) | `rs1 ^ (1UL << rs2)` |
| `BEXT rd, rs1, rs2` | rd = (rs1 >> (rs2 & (XLEN-1))) & 1 | `(rs1 >> rs2) & 1` |

### 100.4.2 TableGen Patterns

```tablegen
// BSET pattern
def : Pat<(or GPR:$rs1, (shl (i64 1), GPR:$rs2)),
          (BSET GPR:$rs1, GPR:$rs2)>;
// BCLR pattern
def : Pat<(and GPR:$rs1, (not (shl (i64 1), GPR:$rs2))),
          (BCLR GPR:$rs1, GPR:$rs2)>;
// BEXT pattern
def : Pat<(and (srl GPR:$rs1, GPR:$rs2), (i64 1)),
          (BEXT GPR:$rs1, GPR:$rs2)>;
```

---

## 100.5 Zk Cryptography Extensions

### 100.5.1 Zkn: NIST Algorithms (AES and SHA-2)

**Scalar AES (Zkn/Zknd/Zkne/Zknh):**

The scalar AES instructions operate on 64-bit halves of the 128-bit AES state:

```asm
; AES-128 round function on RV64 (full state = xmm[0]:xmm[1] in rs/rd pairs)
; Encryption forward round (with MixColumns)
AES64ESM  rd, rs1, rs2    ; SubBytes + ShiftRows + MixColumns

; Encryption final round (without MixColumns)
AES64ES   rd, rs1, rs2    ; SubBytes + ShiftRows only

; Key schedule
AES64KS1I rd, rs1, rnum   ; key schedule round i
AES64KS2  rd, rs1, rs2    ; final XOR for key schedule
```

**Vector AES (Zvkn/Zvkned/Zvkned):**

Vector AES operates on full 128-bit AES state (one element = one AES block):

```asm
; Vector AES encrypt middle round (with MixColumns)
vaesem.vv  vd, vs2         ; vd = AESRound(vd, vs2)

; Vector AES encrypt final round
vaesef.vv  vd, vs2         ; final round (no MixColumns)

; Vector AES expand key round i (all blocks use same round key)
vaeskf1.vi vd, vs2, rnum   ; key expansion round

; AES decrypt
vaesdm.vv  vd, vs2         ; middle decrypt round
vaesdf.vv  vd, vs2         ; final decrypt round
```

**SHA-256/512 Instructions:**

```asm
; SHA-256 transformation functions
SHA256SUM0  rd, rs    ; Σ0(a) = rotr(a,2) ^ rotr(a,13) ^ rotr(a,22)
SHA256SUM1  rd, rs    ; Σ1(e) = rotr(e,6) ^ rotr(e,11) ^ rotr(e,25)
SHA256SIG0  rd, rs    ; σ0(w) = rotr(w,7) ^ rotr(w,18) ^ (w>>3)
SHA256SIG1  rd, rs    ; σ1(w) = rotr(w,17) ^ rotr(w,19) ^ (w>>10)

; SHA-512 (RV64 only)
SHA512SUM0  rd, rs    ; Σ0 for SHA-512
SHA512SUM1  rd, rs    ; Σ1 for SHA-512
SHA512SIG0  rd, rs    ; σ0 for SHA-512
SHA512SIG1  rd, rs    ; σ1 for SHA-512
```

### 100.5.2 Zkr: Entropy Source

The Zkr extension provides a SEED CSR for hardware entropy:

```asm
; Read entropy from hardware source
csrrw  a0, seed, x0    ; a0 = entropy word
; a0[1:0] = status bits:
;   00 = WAIT (entropy not ready)
;   01 = BIST (built-in self-test in progress)
;   10 = ES16 (a0[31:16] contains 16 bits of entropy)
;   11 = DEAD (entropy source failed)
```

LLVM exposes this via `__builtin_riscv_seed()` which lowers to a `CSRRW` instruction on the `seed` CSR.

### 100.5.3 Zksh: SM3/SM4 (Chinese Standards)

Zksh adds SM3 hash function instructions (Chinese national standard GB/T 32905-2016):

```asm
SM3P0  rd, rs    ; SM3 P0 permutation function
SM3P1  rd, rs    ; SM3 P1 permutation function
```

Zksed adds SM4 block cipher instructions:

```asm
SM4ED  rd, rs1, rs2, bs  ; SM4 encryption/decryption round
SM4KS  rd, rs1, rs2, bs  ; SM4 key schedule round
```

---

## 100.6 Zfh and Zfa: Extended Floating-Point

### 100.6.1 Zfh: Half-Precision Floating-Point

Zfh adds complete IEEE 754 binary16 (`f16`) support to the FPU:

```asm
; FP16 arithmetic
FADD.H   fd, fs1, fs2   ; fd = fs1 + fs2 (half precision)
FSUB.H   fd, fs1, fs2
FMUL.H   fd, fs1, fs2
FDIV.H   fd, fs1, fs2
FSQRT.H  fd, fs1
FMADD.H  fd, fs1, fs2, fs3  ; fd = fs1*fs2 + fs3

; Conversions
FCVT.S.H fd, fs1          ; half → single
FCVT.H.S fd, fs1          ; single → half (with rounding)
FCVT.D.H fd, fs1          ; half → double
FCVT.H.D fd, fs1          ; double → half

; Load/store
FLH      fd, offset(rs)   ; load half-precision float
FSH      fs, offset(rd)   ; store half-precision float
```

```c
// Clang with Zfh
_Float16 half_add(_Float16 a, _Float16 b) {
    return a + b;   // FADD.H fa0, fa0, fa1
}
_Float16 half_fma(_Float16 a, _Float16 b, _Float16 c) {
    return __builtin_fmaf16(a, b, c);  // FMADD.H
}
```

`Zfhmin` provides only the FCVT.S.H and FCVT.H.S conversion instructions (for storage/interchange), without arithmetic. This is used in systems that want to store ML weights in FP16 but compute in FP32.

### 100.6.2 Zfa: Additional FP Operations

Zfa fills gaps in the IEEE 754-2019 operation set:

| Instruction | Operation | IEEE 754-2019 function |
|-------------|-----------|----------------------|
| `FMINM.S` | IEEE minimum (propagates NaN) | `minimum` |
| `FMAXM.S` | IEEE maximum | `maximum` |
| `FROUND.S` | Round to integer in FP format | `roundToIntegralExact` |
| `FROUNDNX.S` | Round, signal inexact | `roundToIntegralSignaling` |
| `FMVH.X.D` | Move high 32 bits of double to integer | (bit manipulation) |
| `FMVP.D.X` | Move integer pair to double register | (bit manipulation) |
| `FLEQ.S` | FP less-than-or-equal (quiet) | `compareQuietLessEqual` |
| `FLTQ.S` | FP less-than (quiet) | `compareQuietLessThan` |
| `FLI.S` | Load floating-point immediate | (immediate load) |

### 100.6.3 FLI: Floating-Point Load Immediate

`FLI.S` can load one of 32 predefined constants directly into an FP register without a constant pool:

| Encoding | Value |
|----------|-------|
| 0 | -1.0 |
| 1 | min_positive_normal |
| 2 | 1.0/8 (0.125) |
| 3 | 1.0/4 |
| 4 | 1.0/2 |
| 5 | 1.0 |
| 6 | 2.0 |
| 7 | 4.0 |
| ... | ... (powers of 2 and fractions) |
| 31 | +∞ |

```asm
FLI.S fa0, 1.0    ; fa0 = 1.0f  (encoding 5)
FLI.D fa1, 2.0    ; fa1 = 2.0   (encoding 6)
```

LLVM recognizes these values and emits `FLI.S`/`FLI.D` instead of `LUI+FMV.W.X` sequences when `+zfa` is enabled.

---

## 100.7 Zicfilp and Zicfiss: Control-Flow Integrity

### 100.7.1 Zicfilp: Landing Pad

Zicfilp provides CFI for indirect branches (forward-edge protection). Every valid indirect branch target must begin with `LPAD`:

```asm
; Function that may be called indirectly
func_indirect_target:
    lpad  0            ; marks as valid indirect branch target
    ; optional: lpad N where N is a label identifier (0 = any)
    addi  sp, sp, -16
    sd    ra, 8(sp)
    ...
```

When Zicfilp enforcement is active, an indirect jump (`JALR`) to an address not beginning with `LPAD 0` causes a `software-check exception`. The `N` operand enables "labeled landing pads" for more granular checking — the caller can specify the expected label in a register before the indirect call.

LLVM emits `LPAD` at:
- Function entry points (when marked as address-taken or exported)
- Jump-table targets
- Exception handling landing pads

```bash
# Enable Zicfilp protection
clang --target=riscv64-linux-gnu \
    -march=rv64gc_zicfilp \
    -fcf-protection=branch \
    -O2 foo.c -o foo
```

### 100.7.2 Zicfiss: Shadow Stack

Zicfiss provides return-address protection (backward-edge CFI) via a hardware shadow stack:

```asm
; Function prologue with shadow stack
func:
    sspush  ra         ; pushes ra onto shadow stack
    addi    sp, sp, -16
    sd      ra, 8(sp)
    ...

; Function epilogue with shadow stack check
    ld      ra, 8(sp)
    addi    sp, sp, 16
    sspopchk ra        ; pops shadow stack, faults if ≠ ra
    ret
```

The shadow stack pointer is maintained in `ssp` CSR (supervisor shadow stack pointer or user shadow stack pointer depending on privilege mode). LLVM instruments prologues and epilogues when `-fcf-protection=return` is specified:

```bash
clang --target=riscv64-linux-gnu \
    -march=rv64gc_zicfiss \
    -fcf-protection=return \
    -O2 foo.c -o foo
```

---

## 100.8 Zc* Compressed Extensions

### 100.8.1 Zcb: Basic Additional 16-bit Instructions

Zcb extends Thumb-C with additional 16-bit encodings for common operations that previously required 32-bit forms:

```asm
c.lbu   rd', offset(rs1')   ; load byte unsigned (16-bit)
c.lhu   rd', offset(rs1')   ; load halfword unsigned
c.lh    rd', offset(rs1')   ; load halfword signed
c.sb    rs2', offset(rs1')  ; store byte
c.sh    rs2', offset(rs1')  ; store halfword
c.mul   rd', rs2'            ; multiply (rd' = rd' * rs2')
c.zext.b rd'                 ; zero-extend byte
c.sext.b rd'                 ; sign-extend byte
c.zext.h rd'                 ; zero-extend halfword
c.sext.h rd'                 ; sign-extend halfword
```

### 100.8.2 Zcmp: Compressed Push/Pop

Zcmp adds multi-register push/pop instructions that save/restore register lists with a single 16-bit encoding:

```asm
; Save {ra, s0-s3} and adjust SP downward by 32 bytes
cm.push {ra, s0-s3}, -32

; Restore {ra, s0-s3}, adjust SP upward by 32, and return
cm.popret {ra, s0-s3}, 32

; Restore without returning (for non-leaf functions that jump elsewhere)
cm.pop {ra, s0-s3}, 32
```

Supported register lists: `{ra}`, `{ra, s0}`, `{ra, s0-s1}`, ..., `{ra, s0-s11}`. These are particularly valuable for Cortex-M-class RISC-V cores (e.g., Nuclei N series) where function prologue/epilogue dominates small-function overhead.

### 100.8.3 Zcmb: Move A0/A1 ↔ S Registers

```asm
cm.mva01s   s0, s1   ; move a0 = s0, a1 = s1 (before a call)
cm.mvsa01   s0, s1   ; move s0 = a0, s1 = a1 (after a call, save results)
```

---

## 100.9 Vendor Extensions

### 100.9.1 XTHeadVector (T-Head C906/C910/C920)

Alibaba's T-Head cores implement a pre-ratification RVV draft (approximately RVV 0.7) called XTHeadVector. It uses the same opcode space as RVV but with different instruction semantics and vtype encoding. LLVM includes compatibility support in `RISCVTargetLowering.cpp`:

```bash
# Target T-Head C906 (Allwinner D1, used in RISC-V Linux boards)
clang --target=riscv64-linux-gnu \
    -march=rv64gcxtheadv \
    -mcpu=c906 \
    -O2 foo.c -o foo

# LLVM translates standard RVV intrinsics to XTHead encoding
# when targeting xtheadvector-enabled cores
```

Key differences from RVV 1.0: LMUL=8 not supported, no fractional LMUL, different vtype bit layout.

### 100.9.2 XSiFiveF14/F31 (SiFive DSP Cores)

SiFive's Intelligence X280 and P670 cores add custom instructions for matrix operations:

```tablegen
// SiFive XSiFiveF14 — matrix acceleration instruction
def XSFVQMACCDOD : SiFiveVCustom<...> {
  let mayLoad = 0;
  let mayStore = 0;
  // 8-bit matrix multiply-accumulate to 32-bit
}
```

These are exposed as vendor-specific intrinsics in LLVM.

### 100.9.3 Ventana Micro (XVentanaCondOps)

Ventana's VT-x280 adds conditional-operation extensions:

```asm
vt.maskc  rd, rs1, rs2   ; rd = rs2 if rs1 != 0 else 0
vt.maskcn rd, rs1, rs2   ; rd = rs2 if rs1 == 0 else 0
```

These can eliminate branches in predicated code and are modeled in LLVM as `XVentanaCondOps`.

---

## 100.10 Adding a Custom Extension to LLVM

This section provides a complete walkthrough for adding a hypothetical custom extension `Xcustom` containing one new instruction: `XFMA4 rd, rs1, rs2, rs3` (4-operand fused multiply-add: `rd = rs1 * rs2 + rs3`).

### 100.10.1 Step 1: SubtargetFeature Definition

```tablegen
// Add to llvm/lib/Target/RISCV/RISCVFeatures.td

def FeatureExtXcustom : SubtargetFeature<
    "xcustom", "HasExtXcustom", "true",
    "'Xcustom' (Custom 4-operand FMA extension)",
    []>;  // no dependencies

// Add to RISCVProcessors.td for processors that support it
def : ProcessorModel<"xcustom-cpu",
    RocketModel,
    [FeatureRV64Bit, FeatureStdExtM, FeatureStdExtA,
     FeatureStdExtF, FeatureStdExtD, FeatureExtXcustom]>;
```

```cpp
// Add to llvm/lib/Target/RISCV/RISCVSubtarget.h
bool HasExtXcustom = false;
bool hasExtXcustom() const { return HasExtXcustom; }
```

### 100.10.2 Step 2: Instruction TableGen Definition

```tablegen
// Create llvm/lib/Target/RISCV/RISCVInstrInfoXcustom.td

// XFMA4 encoding: custom-0 opcode space (0x0B)
// R4-type format: funct2 | rs3 | funct3 | rs1 | rs2 | funct3 | rd | opcode
def XFMA4 : RVInstR4<0b00, 0b111, OPC_CUSTOM_0,
    (outs FPR64:$rd),
    (ins FPR64:$rs1, FPR64:$rs2, FPR64:$rs3),
    "xfma4\t$rd, $rs1, $rs2, $rs3",
    [(set FPR64:$rd,
        (fadd (fmul FPR64:$rs1, FPR64:$rs2), FPR64:$rs3))]> {
  let Predicates = [HasExtXcustom, HasStdExtD];
  let isCommutable = 0;
}

// Also add to RISCVInstrInfo.td include list:
// include "RISCVInstrInfoXcustom.td"
```

### 100.10.3 Step 3: LLVM Intrinsic (Optional)

For explicit use (when pattern matching is insufficient):

```tablegen
// Add to llvm/include/llvm/IR/IntrinsicsRISCV.td

def int_riscv_xcustom_xfma4 : DefaultAttrsIntrinsic<
    [llvm_double_ty],        // return: double
    [llvm_double_ty,         // rs1: double
     llvm_double_ty,         // rs2: double
     llvm_double_ty],        // rs3: double
    [IntrNoMem, IntrSpeculatable, IntrWillReturn]>;
```

```cpp
// Add to RISCVISelLowering.cpp
case Intrinsic::riscv_xcustom_xfma4:
  return DAG.getNode(RISCVISD::XFMA4, DL, Op.getValueType(),
                     Op.getOperand(1), Op.getOperand(2), Op.getOperand(3));
```

### 100.10.4 Step 4: Clang Driver Integration

```cpp
// clang/lib/Driver/ToolChains/Arch/RISCV.cpp
// In the extension parsing function:
if (Ext.compare_lower("xcustom") == 0) {
  Features.push_back("+xcustom");
  continue;
}
```

```cpp
// clang/lib/Basic/Targets/RISCV.h
// Add to supported extension list
{"xcustom", {RISCV::AEK_XCUSTOM}, "+xcustom"}
```

### 100.10.5 Step 5: C Header with Intrinsic

```c
// Include in your project or add to clang/lib/Headers/riscv_xcustom.h
#ifndef __RISCV_XCUSTOM_H
#define __RISCV_XCUSTOM_H

#ifdef __riscv_xcustom
static __inline__ double __attribute__((__always_inline__, __nodebug__,
    __target__("xcustom")))
__riscv_xcustom_xfma4(double rs1, double rs2, double rs3) {
    return __builtin_riscv_xcustom_xfma4(rs1, rs2, rs3);
}
#endif

#endif
```

### 100.10.6 Step 6: Test Infrastructure

```llvm
; test/CodeGen/RISCV/xcustom-xfma4.ll
; RUN: llc -mtriple=riscv64 -mattr=+d,+xcustom < %s | FileCheck %s

define double @test_xfma4(double %a, double %b, double %c) {
  ; CHECK-LABEL: test_xfma4
  ; CHECK: xfma4 fa0, fa0, fa1, fa2
  %mul = fmul double %a, %b
  %add = fadd double %mul, %c
  ret double %add
}

define double @test_no_xcustom(double %a, double %b, double %c)
    #0 {
  ; CHECK-LABEL: test_no_xcustom
  ; CHECK: fmul.d
  ; CHECK: fadd.d
  %mul = fmul double %a, %b
  %add = fadd double %mul, %c
  ret double %add
}
attributes #0 = { "target-features"="-xcustom" }
```

```bash
# Run the test
/usr/lib/llvm-22/bin/llc -mtriple=riscv64 \
    -mattr=+d,+xcustom \
    test/CodeGen/RISCV/xcustom-xfma4.ll | grep "xfma4"
```

---

## Chapter Summary

- Zba adds `SH1ADD`/`SH2ADD`/`SH3ADD` for scaled pointer arithmetic (stride 2/4/8), directly matched from `base + idx << {1,2,3}` DAG patterns, collapsing two instructions into one for array indexing.
- Zbb provides a complete basic bit-manipulation toolkit: CLZ/CTZ/CPOP (count), MIN/MAX (signed and unsigned), REV8 (byte swap for endian conversion), ORC.B (byte-wise OR-combine for `strlen`), ROL/ROR (rotation), and ZEXT.H/SEXT.B/SEXT.H (narrowing extension without AND masks).
- Zbc adds `CLMUL`/`CLMULH`/`CLMULR` for carry-less multiplication, the foundation of CRC-32/64 and AES-GCM GHASH; LLVM patterns map Barrett-reduction CRC loops to these instructions.
- Zbs provides `BSET`/`BCLR`/`BINV`/`BEXT` for single-bit set/clear/invert/extract, replacing 2–3 instruction sequences involving shifts and AND/OR/XOR.
- Zkn (scalar AES via `AES64ES[M]` and vector AES via `VAESEM`/`VAESKF1`/`VSHA2*`) and Zksh (SHA-256/512 sigma functions) implement NIST cryptographic primitives in hardware; Zkr provides a hardware entropy source via the `SEED` CSR.
- Zfh adds IEEE 754 binary16 (`_Float16`) arithmetic including `FADD.H`/`FMUL.H`/`FMADD.H`; Zfhmin provides only conversions; Zfa adds IEEE 754-2019 `minimum`/`maximum`, rounding operations, and `FLI.S`/`FLI.D` for loading 32 predefined FP constants without a constant pool.
- Zicfilp (landing pads via `LPAD`) and Zicfiss (shadow stack via `SSPUSH`/`SSPOPCHK`) provide RISC-V's CFI mechanisms for forward-edge and backward-edge protection respectively.
- Zcmp/Zcmpe add compressed multi-register push/pop (`CM.PUSH`, `CM.POP`, `CM.POPRET`) that pack prologue/epilogue into 16-bit encodings, critical for code density on embedded RISC-V cores.
- Adding a custom extension to LLVM requires six steps: `SubtargetFeature` in `RISCVFeatures.td`, instruction encoding in a new `RISCVInstrInfoX*.td` with ISel patterns, optional LLVM intrinsic in `IntrinsicsRISCV.td`, a clang driver `-march` parser entry, a C header, and a FileCheck test confirming correct instruction emission.
