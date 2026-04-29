# Chapter 98 — The RISC-V Backend Architecture

*Part XV — Targets*

RISC-V is an open, modular ISA designed for extensibility. Its standard base plus optional extension model maps directly onto LLVM's subtarget feature system, making the RISC-V backend one of the clearest examples of how LLVM's target infrastructure handles a combinatorially large instruction set. The backend spans RV32/RV64 base integers, the standard extension alphabet (M, A, F, D, C, Zb*, Zk*, V, and more), multiple calling conventions, a sophisticated linker-relaxation pipeline, and support for current industry profiles (RVA22/RVA23). This chapter covers the RISC-V backend's architecture in full: base ISA selection, calling conventions, pseudo-instruction lowering, the relaxation infrastructure, compressed instruction encoding, scheduling models for real cores, and the GlobalISel path that RISC-V 64-bit now uses by default.

---

## 98.1 Base ISA and Extension Alphabet

### 98.1.1 RV32I and RV64I

The base integer ISA has two widths: RV32I (32-bit XLEN) and RV64I (64-bit XLEN). LLVM selects between them based on the triple:

```
riscv32-unknown-elf        → RV32I, ILP32 ABI
riscv64-unknown-linux-gnu  → RV64I, LP64 ABI
riscv64-linux-ohos         → RV64I, LP64D ABI (OpenHarmony)
```

RV64I adds `*W` suffix instructions (`ADDW`, `SUBW`, `SLLW`, `SRLW`, `SRAW`, `MULW`, `DIVW`, `REMW`) for 32-bit arithmetic on the lower 32 bits of 64-bit registers, always producing sign-extended 64-bit results. This is essential for C's `int` type on RV64: a 32-bit add must use `ADDW` to correctly sign-extend rather than 64-bit `ADD`.

The `RISCVSubtarget.cpp` constructor selects XLEN and the consequent type widths used throughout the backend's type legalization.

### 98.1.2 Standard Extension Alphabet

| Extension | Instructions Added | Clang flag |
|-----------|-------------------|------------|
| M | MUL, MULH, MULHSU, MULHU, DIV, DIVU, REM, REMU | `-march=rv64im` |
| A | LR.W/D, SC.W/D, AMOSWAP/ADD/AND/OR/XOR/MAX/MIN.W/D | `+a` |
| F | FADD.S, FSUB.S, FMUL.S, FDIV.S, FSQRT.S, FLW, FSW, FCVT.S.W, FCLASS.S | `+f` |
| D | FADD.D, FDIV.D, FLD, FSD, FCVT.D.S (extends F) | `+d` |
| C | Compressed 16-bit encoding for common ops | `+c` |
| Zba | SH1ADD, SH2ADD, SH3ADD, SH[1-3]ADD.UW | `+zba` |
| Zbb | CLZ, CTZ, CPOP, MIN/MAX, ORC.B, REV8, ROL/ROR, ZEXT.H, SEXT.B/H | `+zbb` |
| Zbc | CLMUL, CLMULH, CLMULR | `+zbc` |
| Zbs | BSET, BCLR, BINV, BEXT | `+zbs` |
| Zfh | FADD.H, FLH, FSH, FCVT.S.H, etc. | `+zfh` |
| V | Vector extension (RVV 1.0) | `+v` |
| Zk* | Scalar crypto (AES, SHA, PRNG) | `+zkn`, `+zksh` |
| Zicfilp | Landing pad CFI | `+zicfilp` |
| Zicfiss | Shadow stack CFI | `+zicfiss` |

The `-march` string specifies extensions by concatenation: `-march=rv64gc_zba_zbb_zbs` = RV64I + G (IMAFD) + C + Zba + Zbb + Zbs.

### 98.1.3 The 'G' Shorthand

`G` is a shorthand for `IMAFD` + `Zicsr` + `Zifencei`. So `-march=rv64gc` means RV64I + M + A + F + D + Zicsr + Zifencei + C. This is the standard general-purpose profile.

---

## 98.2 Calling Conventions

### 98.2.1 ABI Variants

RISC-V defines six standard ABIs:

| ABI | XLEN | FP registers used | Typical target |
|-----|------|-------------------|----------------|
| ILP32 | 32-bit | None (soft float) | RV32 no-FPU |
| ILP32F | 32-bit | Float args in f-regs | RV32F |
| ILP32D | 32-bit | Double/float in f-regs | RV32D |
| LP64 | 64-bit | None (soft float) | RV64 no-FPU |
| LP64F | 64-bit | Float args in f-regs | RV64F |
| LP64D | 64-bit | Double/float in f-regs | RV64D (dominant) |

### 98.2.2 LP64D Register Assignment

For LP64D (the dominant 64-bit Linux ABI):

- **Integer arguments**: a0–a7 (x10–x17), in order
- **FP arguments**: fa0–fa7 (f10–f17), `float` and `double` args use these
- **Integer return**: a0 (32/64-bit), a0+a1 (128-bit or large pair)
- **FP return**: fa0 (scalar), fa0+fa1 (complex)
- **Callee-saved GPRs**: s0–s11 (x8–x9, x18–x27), where s0=fp
- **Callee-saved FPRs**: fs0–fs11 (f8–f9, f18–f27)
- **Temporaries**: t0–t6 (x5–x7, x28–x31), ft0–ft11 (f0–f7, f28–f31)
- **Special**: x0=zero, ra=x1, sp=x2, gp=x3, tp=x4 (TLS)

### 98.2.3 Struct and Aggregate Passing

Structs ≤ 2×XLEN (≤ 16 bytes on RV64) are passed by value in up to two integer registers. Fields of type `float` or `double` may be passed in FP registers if the struct consists only of FP fields (analogous to AArch64 HFA):

```c
struct SmallInt { long a; long b; };  // passed in a0, a1
struct FloatPair { float x; float y; }; // passed in fa0, fa1 (LP64D)
struct Mixed { int a; float b; };     // a in a0, b in fa0 (LP64D)
struct Large { long a, b, c; };       // passed by implicit pointer
```

Varargs functions place all arguments (including FP) in integer registers from a0, bypassing the FP register path entirely.

---

## 98.3 Pseudo-Instruction Lowering

### 98.3.1 `LI`: Load Immediate

Loading a full 64-bit constant into a register may require up to 8 instructions. LLVM's `RISCVMatInt.cpp` computes the optimal expansion using a cost-minimizing search over shift+add sequences:

```asm
; 12-bit immediate (fits ADDI directly)
li a0, 42
→ addi a0, x0, 42

; 32-bit immediate requiring LUI + ADDI
li a0, 0x12345678
→ lui  a0, 0x12345      ; a0 = 0x12345000
   addi a0, a0, 0x678   ; a0 = 0x12345678

; Note: ADDI uses signed 12-bit offset, so if low 12 bits ≥ 0x800,
; LUI must add 1 to compensate for ADDI's sign extension:
li a0, 0x12345800
→ lui  a0, 0x12346      ; +1 to LUI high bits
   addi a0, a0, -0x800  ; -0x800 sign-extends to correct value

; 64-bit constants: up to 8 instructions via shift+OR decomposition
; RISCVMatInt::generateInstSeq() finds the optimal sequence
```

### 98.3.2 `LA`: Load Address

The `LA` pseudo loads a symbol address. Behavior depends on code model:

```asm
; Position-dependent (absolute), small code model:
la a0, symbol
→ lui  a0, %hi(symbol)
  addi a0, a0, %lo(symbol)

; Position-independent code (-fPIC), PCREL:
la a0, symbol
→ .Lref:
  auipc a0, %pcrel_hi(symbol)    ; a0 = PC + high 20-bit offset
  addi  a0, a0, %pcrel_lo(.Lref) ; a0 += low 12-bit offset
```

The `%pcrel_hi`/`%pcrel_lo` relocation pair is used by the relaxation pipeline for linker-time shrinkage.

### 98.3.3 `CALL` and `TAIL` Pseudos

The `CALL` pseudo generates a PLT-compatible far call:

```asm
call symbol
→ (before relaxation — far call)
  .Lrel:
  auipc  ra, %pcrel_hi(symbol)
  jalr   ra, ra, %pcrel_lo(.Lrel)
  ; R_RISCV_CALL_PLT relocation pair

→ (after relaxation — near call, if |offset| < 1MB)
  jal    ra, symbol
  ; single JAL instruction
```

`TAIL` is identical but uses `t1` (x6) as the link register instead of `ra`, enabling tail-call without clobbering the return address:

```asm
tail symbol
→ auipc t1, %pcrel_hi(symbol)
  jalr  x0, t1, %pcrel_lo(.Lref)  ; discard return address
```

### 98.3.4 Addressing Mode Pseudos

```asm
; Load from label (absolute)
lw a0, symbol   →   lui  t0, %hi(symbol); lw a0, %lo(symbol)(t0)

; Thread-local storage (TLS)
la.tls.ie a0, tls_var  ; initial-exec TLS access
la.tls.gd a0, tls_var  ; general-dynamic TLS (dynamic linking)
```

---

## 98.4 The RISC-V Relaxation Pipeline

### 98.4.1 Linker Relaxation Overview

RISC-V linker relaxation is a post-layout optimization where the linker replaces conservative multi-instruction sequences with shorter alternatives after final symbol addresses are known. This contrasts with CISC targets (like x86) where instruction lengths are fixed.

The process:
1. Compiler emits relaxable sequences tagged with `R_RISCV_RELAX` marker relocations alongside the primary relocation
2. Linker computes final layout and checks if targets are within relaxation range
3. Linker replaces qualifying sequences with shorter forms
4. All subsequent PC-relative addresses are recomputed

### 98.4.2 Relaxation Cases

| Original sequence | Relaxed form | Condition |
|-------------------|-------------|-----------|
| `AUIPC + JALR` (8 bytes) | `JAL` (4 bytes) | target within ±1 MiB of call site |
| `AUIPC + ADDI` for GP-relative | `ADDI` (4 bytes) | symbol within ±2 KB of GP |
| `AUIPC + LW` for GP-relative | `LW` (4 bytes) | small data near GP |
| `LUI + ADDI` (8 bytes) | `ADDI` (if symbol is 0) | symbol = 0 exactly |

### 98.4.3 GP-Relative Relaxation

The global pointer register (GP/x3) is initialized in the C startup code to `__global_pointer$` (the midpoint of the `.sdata` section). Small data within ±2 KB of GP can be accessed with a single instruction:

```c
// C startup
extern char __global_pointer$;
asm volatile("mv gp, %0" :: "r"(&__global_pointer$));
```

```asm
; GP-relative store (small data within 2KB of GP)
sw  a0, %gprel(small_int_var)(gp)

; vs. without GP relaxation:
auipc a1, %pcrel_hi(small_int_var)
sw    a0, %pcrel_lo(.Lref)(a1)
```

LLVM enables GP relaxation via `-msmall-data-limit=N` which places globals ≤ N bytes into `.sdata`/`.sbss`.

### 98.4.4 RISCVInsertVSETVLI Pass

For vector code, the `RISCVInsertVSETVLI` pass (in `RISCVInsertVSETVLI.cpp`) inserts `VSETVLI` instructions before vector operations. It performs dataflow analysis over the MachineFunction to eliminate redundant `VSETVLI` instructions when consecutive vector ops share the same vtype configuration:

```cpp
// RISCVInsertVSETVLI.cpp
// Phase 1: Compute demanded fields (SEW, LMUL, etc.) for each BB
// Phase 2: Insert VSETVLI at minimal points (DFA-based)
// Phase 3: Cleanup passes to merge or delete redundant VSETVLIs
```

---

## 98.5 Subtarget Feature Flags and Profiles

### 98.5.1 Feature Flags in LLVM

RISC-V subtarget features are defined in `RISCVFeatures.td`. Each extension maps to a C++ boolean in `RISCVSubtarget.h`:

```tablegen
// RISCVFeatures.td
def FeatureStdExtM : SubtargetFeature<"m", "HasStdExtM", "true",
    "Standard RISC-V 'M' extensions", [FeatureStdExtZmmul]>;

def FeatureStdExtZba : SubtargetFeature<"zba", "HasStdExtZba", "true",
    "'Zba' (Address Generation Instructions)", []>;

def FeatureStdExtV : SubtargetFeature<"v", "HasStdExtV", "true",
    "Standard RISC-V 'V' vector extensions",
    [FeatureStdExtZve64x, FeatureStdExtZve64f,
     FeatureStdExtZve64d, FeatureStdExtZvl128b]>;
```

The `-march` string is parsed by `RISCVISAInfo.cpp` which maps extension names to feature flags, validates extension combinations, and sets implied features.

### 98.5.2 RVA22 and RVA23 Profiles

RISC-V International defines mandatory profile specifications for application-class processors:

| Profile | Key mandatory extensions |
|---------|------------------------|
| RVA22U64 | RV64I, M, A, F, D, C, Zicsr, Zicntr, Ziccif, Ziccrse, Ziccamoa, Zicclsm, Za64rs, Zihpm, Zba, Zbb, Zbs, Zfhmin, Zkt |
| RVA23U64 | RVA22U64 + V (vector), Zvfhmin, Zfbfmin, Zvbfhmin, Supm |
| RVA23S64 | RVA23U64 + supervisor-mode extensions |

```bash
# Target RVA22 profile (explicit march expansion)
clang --target=riscv64-linux-gnu \
    -march=rv64imafdcv_zba_zbb_zbs_zfhmin \
    -mabi=lp64d \
    foo.c -o foo

# Future: profile shorthand (in progress in LLVM)
# clang --target=riscv64-linux-gnu -march=rva23u64 foo.c
```

---

## 98.6 Compressed (C Extension) Instruction Selection

### 98.6.1 C Extension Instruction Classes

The C extension provides 16-bit encodings for instructions whose operands satisfy specific constraints:

| Class | Constraints | Example instructions |
|-------|------------|---------------------|
| CI format | rd ≠ x0, 6-bit immediate | `c.addi rd, nzimm` |
| CL format | rd ∈ {s0-a5}, rs1 ∈ {s0-a5} | `c.lw rd', offset(rs1')` |
| CS format | rs1 ∈ {s0-a5}, rs2 ∈ {s0-a5} | `c.sw rs2', offset(rs1')` |
| CR format | rd ≠ x0 | `c.add rd, rs2` |
| CSS format | imm must fit | `c.swsp rs2, offset(sp)` |

The "prime" notation (rd', rs1') refers to the restricted 3-bit register encoding for registers x8–x15 (s0/fp through a5).

### 98.6.2 Encoding Selection

LLVM selects 16-bit encodings transparently at the MC layer via `RISCVMCCodeEmitter::encodeInstruction()`. The same `MachineInstr` nodes can produce either 32-bit or 16-bit output — the emitter checks whether operands satisfy C-extension constraints and emits the shorter form if so:

```asm
; addi sp, sp, -16  (SP arithmetic, fits c.addi16sp)
→ c.addi16sp -16        ; 16-bit: 0x7179

; lw a0, 0(sp)  (SP-relative load of low register)
→ c.lwsp a0, 0          ; 16-bit: 0x4082

; add a0, a0, a1  (both are low registers)
→ c.add a0, a1          ; 16-bit: 0x952a (if using CR form)
```

The C extension reduces code size by approximately 25–30% on typical programs.

### 98.6.3 C.JAL and C.J

Compressed jump instructions expand the effective jump range to ±2 KB (C.J) or ±128 KB (C.JAL on RV32 only):

```asm
c.j    offset      ; 16-bit unconditional jump, ±2KB range
c.jal  ra, offset  ; 16-bit call, ±2KB (RV32 only — removed in RV64)
c.jr   rs1         ; 16-bit indirect jump via register
c.jalr rs1         ; 16-bit indirect call
```

---

## 98.7 RISC-V Scheduling Models

### 98.7.1 Rocket Core Model

The original Berkeley Rocket core is a simple 5-stage in-order pipeline. LLVM's scheduling model for Rocket is in `RISCVSchedRocket.td`:

```tablegen
def RocketModel : SchedMachineModel {
  let IssueWidth = 1;      // single-issue
  let MicroOpBufferSize = 0;  // strictly in-order
  let LoadLatency = 3;     // L1 cache hit
  let MispredictPenalty = 3;
  let PostRAScheduler = 0;
}
def Rocket_ALU : ProcResource<1>;
def Rocket_FDiv : ProcResource<1>;
def Rocket_Mem  : ProcResource<1>;
```

### 98.7.2 SiFive U7 (Out-of-Order)

SiFive U7 (used in HiFive Unmatched, FU740) is a 4-wide superscalar OOO processor:

```tablegen
def SiFiveU7Model : SchedMachineModel {
  let IssueWidth = 4;
  let MicroOpBufferSize = 128;
  let LoadLatency = 3;
  let MispredictPenalty = 7;
}
def U7_IEX : ProcResource<2>;  // 2 integer ALUs
def U7_FEX : ProcResource<1>;  // 1 FP unit
def U7_MEM : ProcResource<1>;  // 1 memory unit

// Integer ALU ops use U7_IEX with 1 cycle latency
def : WriteRes<WriteIALU, [U7_IEX]> { let Latency = 1; }
// FP multiply: 3 cycles
def : WriteRes<WriteFMul32, [U7_FEX]> { let Latency = 3; }
```

### 98.7.3 SiFive U74 (JH7110/VisionFive2)

The StarFive JH7110 in VisionFive2 uses the SiFive U74 Multicore CPU:

```bash
# SiFive U74 targeting
/usr/lib/llvm-22/bin/llc -march=riscv64 \
    -mattr=+m,+a,+f,+d,+c \
    -mcpu=sifive-u74 \
    input.ll -o output.s
```

Key U74 characteristics: 5-issue OOO, 40-entry ROB, 3-cycle FP mul latency, 4-cycle FP div latency. The scheduling model drives loop unrolling decisions and instruction scheduling to maximize throughput.

### 98.7.4 Xiangshan (OpenSource High-Performance)

The Xiangshan (香山) OOO core (6-issue, 256-entry ROB) is an open-source Chinese RISC-V processor. LLVM 22.1.x includes an early `XiangShanModel` based on publicly documented microarchitecture information.

---

## 98.8 GlobalISel on RISC-V

### 98.8.1 Production Status

GlobalISel on RISC-V 64-bit is production-ready as of LLVM 22.x for RV64GC targets. The complete pipeline:

1. **IRTranslator** → generic MIR: `G_ADD`, `G_LOAD`, `G_STORE`, `G_FCMP`, `G_INTRINSIC`
2. **RISCVLegalizerInfo** → legalizes all types to RISC-V native widths
3. **RISCVRegisterBankInfo** → assigns GPR or FPR bank
4. **RISCVInstructionSelector** → selects real RISC-V instructions

### 98.8.2 Legalizer Rules

```cpp
// RISCVLegalizerInfo.cpp
// i32 on RV64: may use ADDW/SUBW/etc (W-suffix variants)
// i8/i16: widen to i32, insert sign/zero-extend
// i128: split into two i64 operations

getActionDefinitionsBuilder(G_ADD)
    .legalFor({s32, s64})
    .widenScalarToNextPow2(0)
    .clampScalar(0, s32, s64);

// Pointer: legal as s64 on RV64
getActionDefinitionsBuilder(G_LOAD)
    .legalForTypesWithMemDesc({{s8, p0, s8, 8},
                               {s16, p0, s16, 8},
                               {s32, p0, s32, 8},
                               {s64, p0, s64, 8}})
    .widenScalarToNextPow2(0);
```

### 98.8.3 Register Bank Assignment

RISC-V has two register banks: GPR (integer registers x0–x31) and FPR (floating-point registers f0–f31). The `RISCVRegisterBankInfo` assigns banks based on operation type:

```cpp
// Integer ops, pointers, and atomic ops → GPR bank
// Float32/float64 ops → FPR bank
// Integer-FP conversions: consult source/dest types
const RegisterBankInfo::InstructionMapping &
RISCVRegisterBankInfo::getInstrMapping(const MachineInstr &MI) const {
    unsigned Opc = MI.getOpcode();
    if (isFloatingPoint(MI))
        return FPRMapping;
    return GPRMapping;
}
```

---

## 98.9 RISC-V ABI Nuances

### 98.9.1 Stack Frame Layout

A RISC-V function's stack frame:

```
High addresses
┌─────────────────────────────┐ ← previous SP (incoming)
│ caller's stack args          │
├─────────────────────────────┤ ← callee's SP after entry
│ saved ra (return address)   │
│ saved s0/fp (frame pointer) │
│ other callee-saved regs      │
├─────────────────────────────┤ ← frame pointer (s0/fp)
│ local variables              │
│ spill slots                  │
├─────────────────────────────┤
│ outgoing stack arguments     │
└─────────────────────────────┘ ← current SP
Low addresses
```

### 98.9.2 TLS Access Patterns

Thread-local storage on RISC-V uses the TP register (x4) as the thread pointer. There are four TLS access models:

```asm
; Local-exec (static binary, most efficient)
add  a0, tp, %tprel_lo(tls_var)
lui  t0, %tprel_hi(tls_var)
add  a0, a0, t0, %tprel_add(tls_var)

; Initial-exec (ELF shared library with IE TLS)
la.tls.ie a0, tls_var     ; → AUIPC + LW from GOT[tls_var@TPREL]
add a0, a0, tp

; General-dynamic (fully dynamic TLS)
la.tls.gd a0, tls_var     ; → AUIPC + ADDI + call __tls_get_addr
```

---

## 98.10 Complete Compilation Example

```bash
# Full compilation demonstrating extension use
cat > example.c << 'EOF'
#include <stdint.h>
#include <string.h>

// Uses Zbb: CLZ
int leading_zeros(uint64_t x) {
    return __builtin_clzll(x);
}

// Uses Zba: SH3ADD for 8-byte array indexing
double array_access(double* arr, int i) {
    return arr[i];   // → sh3add a0, a1, a0 on RV64+Zba
}

// Uses A: atomic add
void atomic_inc(int* p) {
    __atomic_fetch_add(p, 1, __ATOMIC_SEQ_CST); // → amoadd.w
}
EOF

# Compile with multiple extensions
clang --target=riscv64-linux-gnu \
    -march=rv64gc_zba_zbb_zbs \
    -mabi=lp64d \
    -O2 -S example.c -o example.s

# Check for expected instructions
echo "=== CLZ (Zbb) ==="
grep -E "clz|cpop|ctz" example.s

echo "=== SH3ADD (Zba) ==="
grep "sh3add" example.s

echo "=== AMOADD (A) ==="
grep "amoadd" example.s

# Check GlobalISel is active
/usr/lib/llvm-22/bin/llc -march=riscv64 \
    -mattr=+m,+a,+f,+d,+c,+zba,+zbb \
    -global-isel \
    -print-after=irtranslator \
    example.ll 2>&1 | head -60
```

---

## Chapter Summary

- The RISC-V base ISA comes in two widths (RV32I/RV64I); RV64I adds `*W`-suffix instructions for 32-bit arithmetic with sign-extension, essential for correct `int` semantics.
- The extension alphabet is modular: `-march=rv64gc_zba_zbb_zbs` precisely selects RV64 + G(IMAFD+Zicsr+Zifencei) + C + address-gen + basic bit-manip + single-bit ops.
- LP64D is the dominant 64-bit Linux ABI: integer args in a0–a7, FP args (float/double) in fa0–fa7; struct fields of matching FP type are passed in FP registers when the struct contains only FP fields.
- Pseudo-instructions `LI`, `LA`, `CALL`, and `TAIL` expand into 1–8 real instructions depending on operand values; `RISCVMatInt.cpp` minimizes the sequence length via a shift+add search.
- Linker relaxation shrinks `AUIPC+JALR` (8 bytes) to `JAL` (4 bytes) for calls within ±1 MiB, and collapses GP-relative `AUIPC+ADDI` to single `ADDI` for small data within ±2 KB of GP.
- The C extension selects 16-bit instruction encodings transparently at the MC layer based on register number and immediate width, reducing code size ~25–30%.
- RVA22U64 and RVA23U64 profiles mandate specific extension supersets for application-class Linux deployment; profile-targeted compilation enables the optimizer to use all mandatory instructions.
- Scheduling models for Rocket (in-order), SiFive U7/U74 (OOO superscalar), and other cores drive instruction scheduling and loop unrolling; select via `-mcpu=sifive-u74` etc.
- GlobalISel is production-ready on RV64GC, performing GPR/FPR bank assignment, type legalization (widening narrow types, splitting i128), and instruction selection without constructing a SelectionDAG.
