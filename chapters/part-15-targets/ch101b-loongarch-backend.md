# Chapter 101b — The LoongArch Backend

*Part XV — Target Implementations*

LoongArch is a 64-bit RISC ISA developed by Loongson Technology Corporation, deployed in Chinese domestic processors since 2021 and now a tier-1 LLVM target with a complete backend, LLD support, and JITLink capability. Unlike the earlier MIPS-based Loongson chips, LoongArch is a clean-slate design with its own encoding, ABI, and extension namespace. As of LLVM 22.1.x and Clang 22, the backend covers the full LP64D ABI, 128-bit LSX and 256-bit LASX vector extensions, TLSDESC-based TLS, linker relaxation, and ORC JIT via JITLink. This chapter provides a deep technical account of the architecture and its LLVM implementation, with code examples generated against `clang 22.1.6 --target=loongarch64-linux-gnu`.

---

## Table of Contents

- [101b.1 LoongArch ISA Overview](#101b1-loongarch-isa-overview)
  - [101b.1.1 LA32 and LA64 Base ISAs](#101b11-la32-and-la64-base-isas)
  - [101b.1.2 Instruction Encoding](#101b12-instruction-encoding)
  - [101b.1.3 PC-Relative Addressing](#101b13-pc-relative-addressing)
  - [101b.1.4 Comparisons with MIPS64 and AArch64](#101b14-comparisons-with-mips64-and-aarch64)
- [101b.2 Register File and ABI Names](#101b2-register-file-and-abi-names)
  - [101b.2.1 General-Purpose Registers](#101b21-general-purpose-registers)
  - [101b.2.2 Floating-Point Registers](#101b22-floating-point-registers)
  - [101b.2.3 SIMD and Extended Registers](#101b23-simd-and-extended-registers)
- [101b.3 LP64D Calling Convention](#101b3-lp64d-calling-convention)
  - [101b.3.1 Integer and Pointer Arguments](#101b31-integer-and-pointer-arguments)
  - [101b.3.2 Floating-Point Arguments](#101b32-floating-point-arguments)
  - [101b.3.3 Aggregate Passing Rules](#101b33-aggregate-passing-rules)
  - [101b.3.4 Callee-Saved Registers and Stack Frame](#101b34-callee-saved-registers-and-stack-frame)
- [101b.4 The LLVM LoongArch Backend](#101b4-the-llvm-loongarch-backend)
  - [101b.4.1 Source Organization](#101b41-source-organization)
  - [101b.4.2 Subtarget Features and CPUs](#101b42-subtarget-features-and-cpus)
  - [101b.4.3 Instruction Selection](#101b43-instruction-selection)
  - [101b.4.4 MC Layer and Code Emission](#101b44-mc-layer-and-code-emission)
- [101b.5 TLS Models and TLSDESC](#101b5-tls-models-and-tlsdesc)
  - [101b.5.1 Local Exec (LE) Model](#101b51-local-exec-le-model)
  - [101b.5.2 General Dynamic (GD) Model](#101b52-general-dynamic-gd-model)
  - [101b.5.3 TLSDESC Model](#101b53-tlsdesc-model)
  - [101b.5.4 Relocation Types](#101b54-relocation-types)
- [101b.6 Vector Extensions: LSX and LASX](#101b6-vector-extensions-lsx-and-lasx)
  - [101b.6.1 LSX: 128-Bit SIMD](#101b61-lsx-128-bit-simd)
  - [101b.6.2 LASX: 256-Bit SIMD](#101b62-lasx-256-bit-simd)
  - [101b.6.3 Auto-Vectorization](#101b63-auto-vectorization)
  - [101b.6.4 Intrinsic Access](#101b64-intrinsic-access)
- [101b.7 LLD LoongArch Support](#101b7-lld-loongarch-support)
  - [101b.7.1 ELF Linking and Relocations](#101b71-elf-linking-and-relocations)
  - [101b.7.2 GOT, PLT, and GOTPCREL](#101b72-got-plt-and-gotpcrel)
  - [101b.7.3 R_LARCH_ALIGN Relaxation](#101b73-r_larch_align-relaxation)
- [101b.8 JITLink Support](#101b8-jitlink-support)
  - [101b.8.1 ELF/LoongArch Backend](#101b81-elfloongarch-backend)
  - [101b.8.2 Relocation Edge Kinds](#101b82-relocation-edge-kinds)
  - [101b.8.3 ORC JIT Use Cases](#101b83-orc-jit-use-cases)
- [101b.9 Production Hardware and OS Status](#101b9-production-hardware-and-os-status)
- [101b.10 Complete Code Examples](#101b10-complete-code-examples)
  - [101b.10.1 Loop with LSX Vectorization](#101b101-loop-with-lsx-vectorization)
  - [101b.10.2 PC-Relative and GOT Addressing](#101b102-pc-relative-and-got-addressing)
  - [101b.10.3 TLS Access: LE vs TLSDESC](#101b103-tls-access-le-vs-tlsdesc)
- [Chapter Summary](#chapter-summary)

---

## 101b.1 LoongArch ISA Overview

### 101b.1.1 LA32 and LA64 Base ISAs

LoongArch defines two base integer ISAs:

- **LA32**: 32-bit word size, 32-bit pointers; the baseline for embedded and low-cost deployments. Integer registers are 32 bits wide.
- **LA64**: 64-bit word size, 64-bit pointers; the ISA of all production Loongson 3A/3C processors.

Both are little-endian. The LA32/LA64 distinction is about data-model width, not endianness — LoongArch does not have a big-endian mode in production use. All current Linux deployments use LA64 with the LP64D ABI.

The LLVM triple for the standard 64-bit Linux target is `loongarch64-linux-gnu`. Cross-compilation:

```bash
clang --target=loongarch64-linux-gnu -O2 -march=la64+lsx foo.c -o foo
```

### 101b.1.2 Instruction Encoding

All LoongArch instructions are fixed-width 32 bits, with no compressed or variable-length forms. The encoding space is divided by the top 6 bits (`[31:26]`), yielding clean decode logic. Key instruction formats:

| Format | Fields | Used by |
|--------|--------|---------|
| 2R | `op[31:22], rj[14:10], rd[4:0]` | Unary ops, moves |
| 3R | `op[31:15], rk[14:10], rj[9:5], rd[4:0]` | ALU, FP |
| 2RI12 | `op[31:22], imm12[21:10], rj[9:5], rd[4:0]` | Load/store, ADDI |
| 2RI16 | `op[31:26], imm16[25:10], rj[9:5], rd[4:0]` | Branch |
| 1RI21 | `op[31:26], imm21, rd[4:0]` | BEQZ, BNEZ |
| I26 | `op[31:26], imm26[25:0]` | B, BL |

The clean 4-byte alignment ensures that PC arithmetic is always a multiple of 4, which simplifies the relaxation and alignment infrastructure.

### 101b.1.3 PC-Relative Addressing

LoongArch uses a two-instruction sequence for PC-relative symbol addressing, analogous to AArch64's `ADRP+LDR` and RISC-V's `AUIPC+LD`:

```asm
; Load a PC-relative symbol (non-PIC, local symbol):
pcalau12i  $a0, %pc_hi20(sym)     ; a0 = (PC + sign_ext(hi20(sym-PC))) & ~0xFFF
ld.w       $a0, $a0, %pc_lo12(sym)  ; a0 = *[a0 + lo12(sym-PC)]
```

`pcalau12i` (PC Add Upper 12-bit Immediate, Aligned) sets the upper 52 bits of the destination to the page-aligned PC-relative offset of the symbol. The low 12 bits are then supplied by the immediate of the subsequent load or `addi.d`. This is confirmed by actual compiler output (see Section 101b.10.2).

For far PC-relative function calls, the two-instruction `call36` pseudo expands to:

```asm
pcaddu18i  $t8, %call36(func)   ; t8 = PC + sign_ext(imm18 << 18)
jr         $t8                   ; indirect call through t8
```

`pcaddu18i` adds an 18-bit-shifted immediate to the PC, reaching ±128 GiB from the call site — sufficient for all practical executables.

### 101b.1.4 Comparisons with MIPS64 and AArch64

LoongArch draws architectural lineage from both MIPS64 and early RISC-V design principles while making cleaner choices in several areas:

| Property | MIPS64 | AArch64 | LoongArch LA64 |
|----------|--------|---------|----------------|
| Branch delay slots | Yes | No | No |
| Link register | GPR `$ra` (R31) via `jalr` | X30 dedicated | `$ra` (r1) |
| Condition codes | None (compare-and-branch) | NZCV flags | None (compare-and-branch) |
| PC-relative load | Pseudo | ADRP+LDR | PCALAU12I+LD |
| SIMD width | 128-bit MSA | 128-bit NEON / 512-bit SVE | 128-bit LSX / 256-bit LASX |
| TLS descriptor | No | Yes | Yes (TLSDESC, 2023+) |

The removal of branch delay slots (inherited from MIPS) is a significant simplification. LoongArch compare-and-branch instructions operate directly on register values: `beq`, `bne`, `blt`, `bltu`, `bge`, `bgeu`, `beqz`, `bnez`.

---

## 101b.2 Register File and ABI Names

### 101b.2.1 General-Purpose Registers

LoongArch LA64 has 32 general-purpose registers, each 64 bits wide:

| Register | ABI name | Role |
|----------|----------|------|
| r0 | `zero` | Hardwired zero |
| r1 | `ra` | Return address (link register) |
| r2 | `tp` | Thread pointer (TLS base) |
| r3 | `sp` | Stack pointer |
| r4–r11 | `a0`–`a7` | Arguments; `a0`–`a1` also return values |
| r12–r20 | `t0`–`t8` | Temporaries (caller-saved) |
| r21 | (reserved) | Reserved by ABI; not used by compiler or user code |
| r22 | `fp` (also `s9`) | Frame pointer (callee-saved) |
| r23–r31 | `s0`–`s8` | Callee-saved |

Note: r21 is **reserved** in the LP64D ABI. It is not given an ABI name and must not be used by portable code. Some system software uses it as a platform-specific scratch register, but the LLVM backend does not assign it to any virtual register.

In assembly, registers are written with a `$` prefix: `$a0`, `$sp`, `$s0`, `$zero`.

### 101b.2.2 Floating-Point Registers

32 floating-point registers, each 64 bits wide:

| Register range | ABI name | Role |
|----------------|----------|------|
| f0–f7 | `fa0`–`fa7` | FP arguments / return values |
| f8–f23 | `ft0`–`ft15` | FP temporaries (caller-saved) |
| f24–f31 | `fs0`–`fs7` | FP callee-saved |

A scalar `float` occupies the low 32 bits of an FP register; `double` uses the full 64 bits. Single-precision operations use the `.s` suffix (`fadd.s`, `fmul.s`); double-precision uses `.d` (`fadd.d`, `fmul.d`).

### 101b.2.3 SIMD and Extended Registers

LSX (128-bit SIMD) adds 32 vector registers:

| Register range | Name | Width |
|----------------|------|-------|
| vr0–vr31 | `$vr0`–`$vr31` | 128 bits |

LASX (256-bit SIMD) extends the same physical registers:

| Register range | Name | Width |
|----------------|------|-------|
| xr0–xr31 | `$xr0`–`$xr31` | 256 bits |

`$vr0` and `$xr0` refer to the same physical register file entry; `$vr0` accesses the low 128 bits of `$xr0`. This aliasing model resembles x86 `xmm`/`ymm`. LSX and LASX operations cannot be freely mixed without understanding the upper-lane zeroing semantics.

---

## 101b.3 LP64D Calling Convention

The LP64D ABI is the standard Linux application binary interface for LoongArch LA64. It mandates the D (double-precision FP) extension. A LP64F variant (single-precision only) and LP64S (soft-float) also exist but are rare in production.

### 101b.3.1 Integer and Pointer Arguments

Up to 8 integer or pointer arguments are passed in `$a0`–`$a7` (r4–r11). Arguments beyond 8 are passed on the stack, in 8-byte slots aligned to 8 bytes from the stack pointer. Return values: a single integer/pointer in `$a0`; a 128-bit integer in `$a0`+`$a1`.

### 101b.3.2 Floating-Point Arguments

Up to 8 FP scalar arguments are passed in `$fa0`–`$fa7`. If both integer and FP arguments are present, each uses its own bank independently — an 8-argument function with 4 integers and 4 FP values uses `$a0`–`$a3` and `$fa0`–`$fa3` simultaneously. This is verified directly by compiler output:

```asm
; double compute(double a, double b, ..., double h)
compute:
    fadd.d  $fa0, $fa0, $fa1
    fadd.d  $fa0, $fa0, $fa2
    fadd.d  $fa0, $fa0, $fa3
    fadd.d  $fa0, $fa0, $fa4
    fadd.d  $fa0, $fa0, $fa5
    fadd.d  $fa0, $fa0, $fa6
    fadd.d  $fa0, $fa0, $fa7
    ret
```

The 8 `double` arguments arrive in `$fa0`–`$fa7` directly; `$fa0` accumulates the sum and is returned.

### 101b.3.3 Aggregate Passing Rules

Aggregates (structs, unions, arrays) follow these rules in order of precedence:

1. **Homogeneous FP Aggregates (HFA)**: A struct composed of 1–4 identical FP scalar members is passed member-by-member in consecutive `$fa` registers, analogous to AArch64's HFA rule. A 3-element `double` struct uses `$fa0`, `$fa1`, `$fa2`.

2. **Aggregates ≤ 16 bytes with mixed FP+int fields**: Split across `$a` and `$fa` registers following field order. If registers are exhausted, the remainder spills to the stack.

3. **Aggregates > 16 bytes or larger HFAs**: Passed by reference (pointer in `$a` register). The caller allocates space on the stack.

The `dot` product example illustrates rule 3 — a 3-`double` struct (24 bytes) is passed by pointer, with the callee loading via `fld.d`:

```asm
; double dot(Vec3 a, Vec3 b):  Vec3 = {double x, y, z} (24 bytes > 16)
dot:
    fld.d   $fa0, $a0, 0        ; a.x from pointer a
    fld.d   $fa1, $a1, 0        ; b.x from pointer b
    fld.d   $fa2, $a0, 8        ; a.y
    fld.d   $fa3, $a1, 8        ; b.y
    fld.d   $fa4, $a0, 16       ; a.z
    fld.d   $fa5, $a1, 16       ; b.z
    fmul.d  $fa2, $fa2, $fa3    ; a.y * b.y
    fmadd.d $fa0, $fa0, $fa1, $fa2   ; a.x*b.x + a.y*b.y
    fmadd.d $fa0, $fa4, $fa5, $fa0   ; + a.z*b.z
    ret
```

### 101b.3.4 Callee-Saved Registers and Stack Frame

Callee-saved GPRs: `$s0`–`$s8` (r23–r31), `$fp` (r22).
Callee-saved FP: `$fs0`–`$fs7` (f24–f31).
The stack grows downward. At function entry, the frame is allocated in one `addi.d $sp, $sp, -N` instruction. The return address is saved in `$ra` (r1) by the `bl`/`jirl` instruction; leaf functions need not spill it. Stack alignment is 16 bytes at call boundaries.

---

## 101b.4 The LLVM LoongArch Backend

### 101b.4.1 Source Organization

The backend lives in `llvm/lib/Target/LoongArch/`:

```
LoongArch/
├── AsmParser/           LoongArchAsmParser.cpp
├── Disassembler/        LoongArchDisassembler.cpp
├── MCTargetDesc/        LoongArchMCCodeEmitter.cpp
│                        LoongArchELFObjectWriter.cpp
│                        LoongArchMCAsmInfo.cpp
├── TargetInfo/          LoongArchTargetInfo.cpp
├── LoongArchAsmPrinter.cpp
├── LoongArchFrameLowering.cpp
├── LoongArchISelDAGToDAG.cpp      ; complex addressing selection
├── LoongArchISelLowering.cpp      ; TargetLowering: type legalization, SDNode lowering
├── LoongArchInstrInfo.cpp
├── LoongArchRegisterInfo.cpp
├── LoongArchSubtarget.cpp
├── LoongArchTargetMachine.cpp
├── LoongArchInstrInfo.td          ; main instruction definitions
├── LoongArchInstrLSX.td           ; LSX 128-bit SIMD
├── LoongArchInstrLASX.td          ; LASX 256-bit SIMD
└── LoongArchRegisterInfo.td       ; register file declaration
```

Canonical source URLs (llvmorg-22.1.0):
- [`LoongArchISelLowering.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/LoongArch/LoongArchISelLowering.cpp)
- [`LoongArchInstrInfo.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/LoongArch/LoongArchInstrInfo.td)
- [`LoongArchRegisterInfo.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/LoongArch/LoongArchRegisterInfo.td)

### 101b.4.2 Subtarget Features and CPUs

The LLVM 22.1.x LoongArch backend defines the following subtarget features (from `llc -march=loongarch64 -mattr=help`):

| Feature flag | Description |
|-------------|-------------|
| `+64bit` | LA64 base ISA (default for `loongarch64`) |
| `+32bit` | LA32 base ISA |
| `+d` | Double-precision FP (implies `+f`) |
| `+f` | Single-precision FP |
| `+lsx` | 128-bit SIMD (Loongson SIMD Extension) |
| `+lasx` | 256-bit SIMD (Loongson Advanced SIMD Extension; implies `+lsx`) |
| `+lbt` | Binary Translation Extension |
| `+lvz` | Virtualization Extension |
| `+ual` | Allow unaligned memory accesses |
| `+relax` | Enable linker relaxation |
| `+frecipe` | Approximate reciprocal: `frecipe.s`, `frsqrte.d` |
| `+lam-bh` | Atomic byte/halfword swap and add |
| `+lamcas` | Atomic compare-and-swap byte/halfword/word/dword |
| `+scq` | `sc.q` 128-bit atomic store-conditional |

Named CPU targets include `la464` (Loongson 3A5000/3A6000 micro-architecture) and `la664` (next generation). The `loongarch64` alias resolves to a baseline without optional extensions. The default for `--target=loongarch64-linux-gnu` is equivalent to `+64bit,+d,+f,+lsx,+relax,+ual`, matching what Loongson hardware ships.

### 101b.4.3 Instruction Selection

Instruction selection uses the SelectionDAG path (`LoongArchISelDAGToDAG.cpp`). Custom lowering handles:

- **Addresses**: `pcalau12i`/`ld` pairs for module-local symbols; `pcalau12i`/`ld.d` pairs through GOT for external symbols in PIC code.
- **Integer division**: `div.w`, `div.wu`, `div.d`, `div.du` lower directly from `sdiv`/`udiv`. No pseudo-expansion needed.
- **Atomics**: `ll.w`/`sc.w` (32-bit) and `ll.d`/`sc.d` (64-bit) for compare-exchange. With `+lamcas`, `amcas.w` and `amcas.d` lower `cmpxchg` directly.
- **Bit manipulation**: `bstrpick.d` (bit-string pick) and `bstrins.d` (bit-string insert) match LLVM `(srl (and X, mask), shift)` patterns.
- **Shift-and-add**: `alsl.d rd, rj, rk, imm` (arithmetic left shift then add) lowers `(add (shl X, N), Y)` for N ∈ {1,2,3,4}.

A TableGen pattern for `alsl.d`:

```tablegen
// LoongArchInstrInfo.td
def : Pat<(add (shl GPR:$rj, (i64 1)), GPR:$rk),
          (ALSL_D GPR:$rj, GPR:$rk, 1)>;
```

### 101b.4.4 MC Layer and Code Emission

`LoongArchMCCodeEmitter` in `MCTargetDesc/LoongArchMCCodeEmitter.cpp` encodes each instruction to its 32-bit binary representation. Fixups for relocatable fields (PC-relative offsets, GOT references) are emitted as `MCFixup` records resolved during assembly or linking.

`LoongArchMCAsmInfo` registers LoongArch-specific assembler syntax, including the `$` register prefix, the `%pc_hi20(sym)` relocation specifier syntax, and comment character (`#`).

`LoongArchAsmPrinter` handles the AsmPrinter pass, lowering `MachineInstr` to `MCInst` and emitting `.addrsig` sections for LTO symbol visibility tracking.

---

## 101b.5 TLS Models and TLSDESC

Thread-local storage on LoongArch/Linux follows the ELF TLS ABI with the thread pointer in `$tp` (r2). The linker allocates TLS segments; at runtime, `$tp` points to the thread-local storage block for the current thread.

### 101b.5.1 Local Exec (LE) Model

For TLS variables defined in the main executable and accessed only from the executable (no shared library involvement), the compiler emits the most efficient sequence using `%le_hi20_r`/`%le_add_r`/`%le_lo12_r` relocation specifiers:

```asm
; get_tls(): return local TLS variable (LE model, main executable)
get_tls:
    lu12i.w  $a0, %le_hi20_r(tls_var)      ; load upper 20 bits of TP-relative offset
    add.d    $a0, $a0, $tp, %le_add_r(tls_var)  ; add TP + upper offset
    ld.w     $a0, $a0, %le_lo12_r(tls_var) ; load from TP + full offset
    ret
```

This is a three-instruction, zero-call sequence. The linker resolves all three relocations at link time to constants, ultimately folding to a single `ld.w $a0, $tp, offset` in many cases after relaxation.

### 101b.5.2 General Dynamic (GD) Model

For TLS variables defined in a shared library (or when the access site is in a shared library), the GD model calls `__tls_get_addr`:

```asm
; get_ext_tls(): extern __thread int ext_tls_var (GD model, fPIC)
get_ext_tls:
    addi.d    $sp, $sp, -16
    st.d      $ra, $sp, 8
    pcalau12i $a0, %gd_pc_hi20(ext_tls_var)   ; load GD descriptor addr (upper)
    addi.d    $a0, $a0, %got_pc_lo12(ext_tls_var)  ; + lower offset
    pcaddu18i $ra, %call36(__tls_get_addr)     ; prepare call to __tls_get_addr
    jirl      $ra, $ra, 0                      ; call → returns ptr to TLS block
    ld.w      $a0, $a0, 0                      ; load value from TLS block
    ld.d      $ra, $sp, 8
    addi.d    $sp, $sp, 16
    ret
```

The `pcalau12i`/`addi.d` pair loads the address of the GD descriptor from the GOT. The call to `__tls_get_addr` returns a pointer to the thread-local storage block for the variable; the final `ld.w` loads the actual value.

### 101b.5.3 TLSDESC Model

TLSDESC (Thread Local Storage Descriptor) was introduced in the LoongArch ABI in 2023 and is supported by Clang 22 via `-mtls-dialect=desc`. It replaces the two-call overhead of GD with a single indirect call through a descriptor stored in the GOT:

```asm
; get_ext_tls() with -mtls-dialect=desc (TLSDESC model)
get_ext_tls:
    addi.d    $sp, $sp, -16
    st.d      $ra, $sp, 8
    pcalau12i $a0, %desc_pc_hi20(ext_tls_var)   ; load descriptor addr (upper)
    addi.d    $a0, $a0, %desc_pc_lo12(ext_tls_var)  ; + lower
    ld.d      $ra, $a0, %desc_ld(ext_tls_var)   ; load function pointer from descriptor
    jirl      $ra, $ra, %desc_call(ext_tls_var) ; call descriptor resolver
    add.d     $a0, $a0, $tp                     ; add thread pointer to returned offset
    ld.w      $a0, $a0, 0                       ; load value
    ld.d      $ra, $sp, 8
    addi.d    $sp, $sp, 16
    ret
```

The descriptor in the GOT contains a function pointer and an addend. On first access, the dynamic linker fills the descriptor with a fast-path resolver. After the first thread accesses the variable, the descriptor is updated to a trivial function that simply returns the pre-computed TP-relative offset. Subsequent calls become a near-zero-overhead indirect call to a one-instruction function. This eliminates the `__tls_get_addr` ABI overhead present in the GD model. The `$a0` returned by the call is the TP-relative offset; `add.d $a0, $a0, $tp` converts it to the absolute address.

### 101b.5.4 Relocation Types

| Relocation | Model | Description |
|------------|-------|-------------|
| `R_LARCH_TLS_GD_PC_HI20` | GD | Upper 20 bits of GOT-relative GD descriptor |
| `R_LARCH_TLS_GD_HI20` | GD | Upper 20 bits (absolute) |
| `R_LARCH_TLS_LE_HI20_R` | LE | Upper 20 bits of TP-relative offset |
| `R_LARCH_TLS_LE_ADD_R` | LE | Relocation in ADD instruction |
| `R_LARCH_TLS_LE_LO12_R` | LE | Lower 12 bits of TP-relative offset |
| `R_LARCH_TLS_DESC_PC_HI20` | DESC | Upper 20 bits of descriptor GOT entry |
| `R_LARCH_TLS_DESC_PC_LO12` | DESC | Lower 12 bits |
| `R_LARCH_TLS_DESC_LD` | DESC | Load of descriptor function pointer |
| `R_LARCH_TLS_DESC_CALL` | DESC | Call through descriptor |

---

## 101b.6 Vector Extensions: LSX and LASX

### 101b.6.1 LSX: 128-Bit SIMD

LSX (Loongson SIMD Extension) provides 128-bit vector operations on 32 vector registers `$vr0`–`$vr31`. Data types are expressed using element suffixes:

| Suffix | Element type | Elements per register |
|--------|-------------|----------------------|
| `.b` | int8 | 16 |
| `.h` | int16 | 8 |
| `.w` | int32 | 4 |
| `.d` | int64 | 2 |
| `.q` | int128 | 1 |
| `.s` | float32 | 4 |
| `.d` | float64 | 2 |

Representative instructions:

```asm
vadd.w    $vr0, $vr1, $vr2     ; 4×i32 add
vsub.h    $vr3, $vr4, $vr5     ; 8×i16 subtract
vmul.w    $vr0, $vr1, $vr2     ; 4×i32 multiply (low 32 bits)
vfadd.d   $vr0, $vr1, $vr2     ; 2×f64 add
vfmadd.d  $vr0, $vr1, $vr2, $vr3  ; 2×f64 fused multiply-add: vr0 = vr1*vr2 + vr3
vmin.bu   $vr0, $vr1, $vr2     ; 16×u8 minimum
vhaddw.w.h $vr0, $vr1, $vr2    ; 8×i16 widen-and-add to 4×i32
vld       $vr0, $a0, 0         ; 128-bit aligned load
vst       $vr0, $a1, 0         ; 128-bit aligned store
vrepli.b  $vr0, 0              ; broadcast immediate 0 to all 16 bytes
```

LLVM IR vector types map to LSX as follows:
- `<16 x i8>` → `.b` operations
- `<8 x i16>` → `.h` operations
- `<4 x i32>` → `.w` operations
- `<2 x i64>` → `.d` operations
- `<4 x float>` → `.s` operations
- `<2 x double>` → `.d` operations

### 101b.6.2 LASX: 256-Bit SIMD

LASX (Loongson Advanced SIMD Extension) doubles the vector width to 256 bits, using the `$xr0`–`$xr31` register names. Operations are syntactically prefixed with `x`:

```asm
xvadd.w   $xr0, $xr1, $xr2     ; 8×i32 add (256-bit)
xvsub.h   $xr3, $xr4, $xr5     ; 16×i16 subtract
xvfmul.s  $xr0, $xr1, $xr2     ; 8×f32 multiply
xvfmadd.d $xr0, $xr1, $xr2, $xr3  ; 4×f64 fused multiply-add
xvld      $xr0, $a0, 0          ; 256-bit aligned load
xvst      $xr0, $a1, 0          ; 256-bit aligned store
```

LASX adds 256-bit LLVM IR vector types:
- `<8 x i32>` → `xvadd.w` / `xvmul.w`
- `<4 x i64>` → `xvadd.d`
- `<8 x float>` → `xvfadd.s`
- `<4 x double>` → `xvfadd.d`

Since `$xr0` and `$vr0` alias the same physical register (LASX extends LSX), mixing LSX and LASX operations on the same logical value requires care. The LLVM register allocator treats them as separate register classes (`VR` for LSX, `XR` for LASX) and does not coalesce across the boundary without an explicit extract/insert.

### 101b.6.3 Auto-Vectorization

The LLVM Loop Vectorizer generates LSX and LASX instructions automatically when vector extensions are enabled. For a simple i32 addition loop:

```bash
clang --target=loongarch64-linux-gnu -O2 -mlsx -fvectorize vec_add.c -S
```

The vectorizer selects `vadd.w` / `vld` / `vst` for LSX and `xvadd.w` / `xvld` / `xvst` for LASX. The choice between 128-bit and 256-bit vectorization depends on the `-mlsx`/`-mlasx` flag. For the `int32_t` addition loop on 8-element blocks, LASX (-mlasx) emits:

```asm
xvld    $xr0, $t0, -32         ; load 8×i32 from a
xvld    $xr1, $a7, -32         ; load 8×i32 from b
xvadd.w $xr0, $xr2, $xr0      ; 8×i32 add
xvst    $xr0, $a6, -32         ; store 8×i32 to dst
```

`LoongArchTargetTransformInfo` (in `LoongArchTargetTransformInfo.cpp`) reports vector element costs and register counts to drive the vectorizer's profitability analysis. LSX provides 32 vector registers (more than x86 AVX2's 16), which benefits the unrolling heuristics.

### 101b.6.4 Intrinsic Access

LSX and LASX are accessible via Clang intrinsics. The intrinsic headers are:
- `<lsxintrin.h>` for LSX (`__builtin_lsx_*`)
- `<lasxintrin.h>` for LASX (`__builtin_lasx_*`)

Example LSX arithmetic intrinsics (from `BuiltinsLoongArchLSX.def`):

```c
#include <lsxintrin.h>

// 4×i32 add: maps to vadd.w
__m128i a = __lsx_vadd_w(x, y);

// 2×f64 fused multiply-add: maps to vfmadd.d
__m128d r = __lsx_vfmadd_d(a, b, c);

// 4×i32 load
__m128i v = __lsx_vld(ptr, 0);

// Predefined feature macros:
// __loongarch_sx  — defined when -mlsx is active
// __loongarch_asx — defined when -mlasx is active
```

Builtin names follow the pattern `__builtin_lsx_<instr>_<type>` and `__builtin_lasx_<instr>_<type>`, mirroring the assembly mnemonic.

---

## 101b.7 LLD LoongArch Support

### 101b.7.1 ELF Linking and Relocations

LLD supports LoongArch ELF binaries via the `--target=loongarch64-linux-gnu` or `-m elf64loongarch` linker emulation. The ELF object writer is in `llvm/lib/MC/MCTargetDesc/LoongArchELFObjectWriter.cpp`; the LLD port is in `lld/ELF/Arch/LoongArch.cpp`.

LoongArch ELF uses `EM_LOONGARCH` (258) as the `e_machine` field. The ELF class is `ELFCLASS64` for LA64 targets. Dynamic linking follows the standard ELF PLT/GOT model with LoongArch-specific relocation types prefixed `R_LARCH_`.

### 101b.7.2 GOT, PLT, and GOTPCREL

External symbol references in position-independent code use a two-instruction GOT load:

```asm
; &global_extern in PIC code
get_ptr:
    pcalau12i  $a0, %got_pc_hi20(global_extern)  ; a0 = page of GOT entry
    ld.d       $a0, $a0, %got_pc_lo12(global_extern)  ; a0 = GOT[global_extern]
    ret
```

The `%got_pc_hi20`/`%got_pc_lo12` relocation specifiers are resolved by the linker to the GOT slot's PC-relative offset. For PLT calls (indirect call through a stub):

```asm
; call to external function: uses call36 pseudo (plt stub)
call_it:
    pcaddu18i  $t8, %call36(call_extern)  ; $t8 = PLT stub address
    jr         $t8                         ; tail call via PLT
```

`pcaddu18i` with `%call36` reaches ±128 GiB from the call site without a GOT load, making it the preferred sequence when the PLT stub is close enough.

### 101b.7.3 R_LARCH_ALIGN Relaxation

LoongArch requires all instructions to be 4-byte aligned. The assembler inserts `nop` instructions (padding) at alignment directives (`.p2align`) in the optimistic allocation phase. The linker then removes redundant `nop` padding via the `R_LARCH_ALIGN` relaxation:

1. The assembler emits `nop` padding at alignment points and marks them with `R_LARCH_ALIGN` relocations.
2. The linker recomputes alignment requirements after section layout and removes unnecessary `nop` instructions.
3. Remaining symbols and branch offsets are adjusted via `R_LARCH_ADD32`/`R_LARCH_SUB32` relocation pairs on debug info sections.

This two-phase approach (optimistic emission + linker relaxation) is the same strategy used by RISC-V (`R_RISCV_ALIGN`). LoongArch enables it by default with `+relax`.

---

## 101b.8 JITLink Support

### 101b.8.1 ELF/LoongArch Backend

LoongArch JIT support in LLVM 22.1.x uses JITLink (the ORC JIT linker), not the older RuntimeDyld. The JITLink backend is located at:

- [`llvm/include/llvm/ExecutionEngine/JITLink/loongarch.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/JITLink/loongarch.h) — edge kinds and utilities
- [`llvm/include/llvm/ExecutionEngine/JITLink/ELF_loongarch.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/JITLink/ELF_loongarch.h) — `createLinkGraphFromELFObject_loongarch`
- `llvm/lib/ExecutionEngine/JITLink/ELF_loongarch.cpp` — ELF relocation parsing and fixup application

The entry point for loading a LoongArch ELF relocatable object into the JIT is:

```cpp
// llvm/include/llvm/ExecutionEngine/JITLink/ELF_loongarch.h
Expected<std::unique_ptr<LinkGraph>>
createLinkGraphFromELFObject_loongarch(
    MemoryBufferRef ObjectBuffer,
    std::shared_ptr<orc::SymbolStringPool> SSP);
```

This creates a `LinkGraph` (JITLink's internal IR) from the ELF object, with each section becoming a `LinkGraph::Section` and each relocation becoming a typed `Edge`.

### 101b.8.2 Relocation Edge Kinds

The `EdgeKind_loongarch` enum (in `loongarch.h`) defines all supported JITLink fixup kinds:

| Edge kind | Description |
|-----------|-------------|
| `Pointer64` | Absolute 64-bit pointer |
| `Pointer32` | Absolute 32-bit pointer |
| `Branch16PCRel` | PC-relative branch ±128 KiB (16-bit operand) |
| `Branch21PCRel` | PC-relative branch ±4 MiB (21-bit operand, `BEQZ`/`BNEZ`) |
| `Branch26PCRel` | PC-relative branch ±128 MiB (26-bit operand, `B`/`BL`) |
| `Page20` | Upper 20 bits of page-relative offset (`pcalau12i`) |
| `PageOffset12` | Lower 12-bit offset within page |
| `PCAddHi20` | `pcaddu12i` upper 20 bits |
| `PCAddLo12` | Lower 12 bits for `pcaddu12i` sequence |
| `Call36PCRel` | `pcaddu18i`+`jr` 36-bit PC-relative call |
| `RequestGOTAndTransformToPage20` | Allocate GOT entry, rewrite as `Page20` |
| `RequestGOTAndTransformToPCAddHi20` | Allocate GOT entry, rewrite as `PCAddHi20` |
| `AlignRelaxable` | Alignment padding that may be removed |

The GOT-allocating edges (`RequestGOT*`) cause JITLink to materialize a GOT slot for the target symbol and rewrite the edge to a PC-relative reference to that slot — implementing the same GOT model as static ELF linking, but at JIT link time.

### 101b.8.3 ORC JIT Use Cases

With JITLink support, ORC JIT can execute LoongArch code compiled on-target (on Loongson 3A5000/3A6000 hardware running a Linux distribution). Use cases in practice:

- **Julia language**: Julia uses ORC JIT to compile and execute LLVM IR natively; on LoongArch this requires JITLink to resolve relocations in the JIT-compiled code.
- **LLVM REPL / lli**: The LLVM interpreter and REPL (`lli --jit-kind=orc`) uses JITLink on LoongArch.
- **Numba**: Python JIT compilation via LLVM uses ORC when targeting loongarch64.

To confirm JITLink availability for LoongArch in LLVM 22.1.x:

```bash
lli --jit-kind=orc --target-triple=loongarch64-linux-gnu \
    --relocation-model=pic simple.ll
```

---

## 101b.9 Production Hardware and OS Status

### Hardware

| Processor | Micro-arch | Cores | Frequency | LoongArch version | Release |
|-----------|-----------|-------|-----------|------------------|---------|
| Loongson 3A5000 | LA464 | 4 | 2.5 GHz | v1.0 | 2021 |
| Loongson 3A6000 | LA664 | 4 (SMT) | 2.5 GHz | v1.1 | 2023 |
| Loongson 3C6000 | LA664 | 16 (SMT) | 2.2 GHz | v1.1 | 2024 |

The 3A6000 implements simultaneous multithreading (2-way SMT) and the LoongArch v1.1 ISA, which adds the `TLSDESC` relocation family, `LAMCAS` (compare-and-swap atomics), and `LAM_BH` (byte/halfword atomics). In published benchmarks, the 3A6000 is competitive with mid-range x86-64 processors (AMD Zen 2 class) in integer workloads.

### Operating System and Toolchain

Linux kernel mainlined LoongArch support in 5.19 (July 2022). Major Linux distributions with LoongArch ports as of mid-2026:

- **Arch Linux (LoongArch)**: Community port with up-to-date packages
- **Debian**: `loong64` architecture, in official ports
- **Fedora**: `loongarch64` secondary architecture
- **openEuler** (Huawei): Tier-1 LoongArch support
- **AOSC OS**: LoongArch as a primary architecture

Toolchain status in GCC and Clang upstream:
- **GCC**: LoongArch support merged in GCC 13 (2023); current GCC 14/15 is production-ready
- **LLVM/Clang**: LoongArch backend merged in LLVM 14 (2022); LLVM 22.1.x is fully production-ready for base ISA + FP + LSX + LASX

---

## 101b.10 Complete Code Examples

### 101b.10.1 Loop with LSX Vectorization

The LLVM IR for a 64-bit integer sum loop (targeting loongarch64 with LSX enabled):

```llvm
; target triple = "loongarch64-unknown-linux-gnu"
; target features: +64bit,+d,+f,+lsx,+relax,+ual

define dso_local i64 @sum_array(ptr noundef readonly %arr, i32 noundef signext %n)
    local_unnamed_addr #0 {
entry:
  %gt_zero = icmp sgt i32 %n, 0
  br i1 %gt_zero, label %check_vlen, label %exit.zero

check_vlen:
  %n64 = zext nneg i32 %n to i64
  %small = icmp ult i32 %n, 4
  br i1 %small, label %scalar.ph, label %vector.ph

vector.ph:
  %trip = and i64 %n64, 2147483644  ; round down to multiple of 4
  br label %vector.body

vector.body:
  ; Two-accumulator unrolled LSX loop (2×<2 x i64>)
  %idx   = phi i64 [ 0, %vector.ph ], [ %idx.next, %vector.body ]
  %acc0  = phi <2 x i64> [ zeroinitializer, %vector.ph ], [ %sum0, %vector.body ]
  %acc1  = phi <2 x i64> [ zeroinitializer, %vector.ph ], [ %sum1, %vector.body ]
  %p0    = getelementptr inbounds i64, ptr %arr, i64 %idx
  %p1    = getelementptr inbounds i8,  ptr %p0, i64 16
  %v0    = load <2 x i64>, ptr %p0, align 8
  %v1    = load <2 x i64>, ptr %p1, align 8
  %sum0  = add <2 x i64> %v0, %acc0
  %sum1  = add <2 x i64> %v1, %acc1
  %idx.next = add nuw i64 %idx, 4
  %done  = icmp eq i64 %idx.next, %trip
  br i1 %done, label %vector.exit, label %vector.body

vector.exit:
  %combined = add <2 x i64> %sum1, %sum0
  %reduced  = call i64 @llvm.vector.reduce.add.v2i64(<2 x i64> %combined)
  ; ... scalar remainder loop
  ret i64 %reduced
```

The corresponding LoongArch64 assembly (compiled with `-O2`, LSX auto-enabled for loongarch64):

```asm
sum_array:
    blez    $a1, .exit_zero
    ori     $a2, $zero, 4
    bgeu    $a1, $a2, .vector_entry
.scalar_prelude:
    move    $a3, $zero
    move    $a2, $zero
    b       .scalar_body
.exit_zero:
    move    $a2, $zero
    move    $a0, $a2
    ret
.vector_entry:
    bstrpick.d $a2, $a1, 30, 2     ; trip count = n & ~3
    slli.d     $a3, $a2, 2
    vrepli.b   $vr0, 0              ; acc0 = 0
    addi.d     $a2, $a0, 16
    vori.b     $vr1, $vr0, 0        ; acc1 = 0
    move       $a4, $a3
.vector_body:                        ; =>This Inner Loop Header
    vld     $vr2, $a2, -16          ; load 2×i64 chunk 0
    vld     $vr3, $a2, 0            ; load 2×i64 chunk 1
    vadd.d  $vr0, $vr2, $vr0        ; acc0 += chunk0
    vadd.d  $vr1, $vr3, $vr1        ; acc1 += chunk1
    addi.d  $a4, $a4, -4
    addi.d  $a2, $a2, 32
    bnez    $a4, .vector_body
    vadd.d  $vr0, $vr1, $vr0        ; combine accumulators
    vhaddw.q.d $vr0, $vr0, $vr0    ; horizontal add: [lo+hi, lo+hi]
    vpickve2gr.d $a2, $vr0, 0       ; extract scalar result
    ; ... scalar remainder ...
    move    $a0, $a2
    ret
```

### 101b.10.2 PC-Relative and GOT Addressing

Non-PIC local symbol access uses `pcalau12i` + offset load; external symbol access in PIC code goes through the GOT:

```asm
; ---- Non-PIC: local symbol (PC-relative direct) ----
get_global:                      ; int get_global() { return global_val; }
    pcalau12i  $a0, %pc_hi20(global_val)     ; a0 = page(global_val - PC)
    ld.w       $a0, $a0, %pc_lo12(global_val) ; a0 = *[a0 + offset]
    ret

; ---- PIC: extern symbol via GOT ----
get_ptr:                         ; int *get_ptr() { return &global_extern; }
    pcalau12i  $a0, %got_pc_hi20(global_extern)   ; a0 = page of GOT entry
    ld.d       $a0, $a0, %got_pc_lo12(global_extern) ; a0 = GOT[global_extern]
    ret

; ---- PIC: PLT call via call36 ----
call_it:                         ; void call_it() { call_extern(); }
    pcaddu18i  $t8, %call36(call_extern)     ; t8 = PLT stub (±128 GiB)
    jr         $t8                            ; tail call
```

The `pcalau12i` mnemonic (PC Add Large Upper 12-bit Immediate, Aligned) is the correct LoongArch instruction for page-aligned PC-relative address computation — not `pcaddu12i` (which exists but computes a non-page-aligned result). The difference: `pcalau12i` clears the low 12 bits of the result, aligning to page boundaries, while `pcaddu12i` does not.

### 101b.10.3 TLS Access: LE vs TLSDESC

For a locally-defined TLS variable in the main executable (LE model):

```asm
; int get_tls() { return tls_var; }   (LE model — static executable)
get_tls:
    lu12i.w  $a0, %le_hi20_r(tls_var)        ; a0 = upper 20 bits of TP-offset
    add.d    $a0, $a0, $tp, %le_add_r(tls_var) ; a0 = TP + upper offset
    ld.w     $a0, $a0, %le_lo12_r(tls_var)   ; a0 = *(a0 + lo12)
    ret
```

For an external TLS variable in shared library code with TLSDESC (`-fPIC -mtls-dialect=desc`):

```asm
; int get_ext_tls() { return ext_tls_var; }  (TLSDESC model)
get_ext_tls:
    addi.d    $sp, $sp, -16
    st.d      $ra, $sp, 8
    pcalau12i $a0, %desc_pc_hi20(ext_tls_var)    ; a0 = page of descriptor GOT slot
    addi.d    $a0, $a0, %desc_pc_lo12(ext_tls_var) ; a0 = &descriptor
    ld.d      $ra, $a0, %desc_ld(ext_tls_var)    ; ra = descriptor->fn_ptr
    jirl      $ra, $ra, %desc_call(ext_tls_var)  ; call fn_ptr(a0=&descriptor)
    ; After call: a0 = TP-relative offset
    add.d     $a0, $a0, $tp                       ; a0 = TP + offset (absolute addr)
    ld.w      $a0, $a0, 0
    ld.d      $ra, $sp, 8
    addi.d    $sp, $sp, 16
    ret
```

The TLSDESC sequence saves one call compared to GD: it calls the descriptor function pointer directly rather than going through `__tls_get_addr` with a separate GOT descriptor argument. After the dynamic linker fills the descriptor's function pointer with a direct-offset function, subsequent accesses execute the `jirl` and return the TP-relative offset in `$a0` in a few cycles with no further indirection.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **GlobalISel completion for LoongArch**: The SelectionDAG path is production-ready, but GlobalISel coverage is still expanding. Ongoing LLVM patches (tracked on discourse.llvm.org under the LoongArch label) aim to enable GlobalISel for all scalar integer and FP operations by late 2026, unlocking faster compile times and better register allocation for `-O0` builds on LoongArch hardware.
- **LoongArch v1.2 ISA ratification**: Loongson has signaled an upcoming v1.2 specification adding new atomic instructions (extended `amcas` variants), a `LBT` binary translation extension revision, and additional CSR registers. LLVM subtarget feature flags (`+lbt-v2`, tentative) and corresponding MC layer patches are expected to follow the specification release.
- **TLSDESC linker relaxation in LLD**: LLD's LoongArch port currently emits TLSDESC sequences without relaxing the descriptor-GOT reference to a direct TP-relative offset in static executables. A patch series (analogous to the AArch64 TLSDESC-to-LE relaxation) is in review to fold TLSDESC to LE when the variable is in the same link unit, eliminating the indirect call entirely at link time.
- **`R_LARCH_PCREL20_S2` linker relaxation**: Ongoing work in LLD to support the `R_LARCH_PCREL20_S2` relocation for relaxing `pcaddu18i`+`jr` call36 pairs to shorter `bl` direct calls when the target is within ±128 MiB, reducing PLT stub indirections in large executables.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Loongson 3A7000/LA864 micro-architecture support**: The next Loongson generation (informally LA864, targeting 2027 production) is expected to introduce wider out-of-order execution, SVE-class scalable vector extensions, and SMT improvements. LLVM will need a new `la864` CPU definition, updated cost models in `LoongArchTargetTransformInfo`, and scheduling models for the new pipeline.
- **Scalable vector (LSX-SVE or LASE) intrinsics**: Loongson has explored a scalable-vector extension analogous to RISC-V RVV or Arm SVE, tentatively called LASE (LoongArch Scalable Extension). If ratified, LLVM would gain a new register class, VectorLength predicate infrastructure, and auto-vectorization cost models — following the RVV precedent established in LLVM's RISC-V backend.
- **LoongArch Clang sanitizer completion**: AddressSanitizer and UBSan are functional on LoongArch; MemorySanitizer and ThreadSanitizer shadow-call-stack support still require upstream patches. By 2028, full parity with x86-64/AArch64 sanitizer coverage (including HWASan via tagged pointers if the LA864 ISA adds memory tagging) is expected.
- **openEuler and Debian LoongArch bootstrap to tier-1**: Loongson is investing in enterprise Linux distribution quality for loongarch64. By 2028, loongarch64 is projected to reach Debian tier-1 and openEuler primary-architecture status, driving upstream demands for LTO, PGO, and BOLT profile-guided optimization support on LoongArch in LLVM.

### 5-Year Horizon (Long-Term, by ~2031)

- **LoongArch as an HPC platform**: Loongson's 3C6000 (16-core) and successors target HPC workloads in Chinese national computing facilities. LLVM's LoongArch backend will need polyhedral optimization (Polly) integration, OpenMP offload support, and LASX→LASE vectorization upgrades comparable to AVX-512 on x86-64. This mirrors the trajectory of the RISC-V HPC stack (SG2042, Sophon).
- **LoongArch MLIR target and IREE backend**: As MLIR-based ML inference frameworks (IREE, TensorFlow, JAX via XLA) expand to non-x86 targets, LoongArch will need an MLIR lowering path from `linalg`/`vector` dialects through `loongarch-lsx` / `loongarch-lasx` intrinsic lowering — analogous to the existing AArch64 NEON and RISC-V RVV MLIR lowering paths.
- **Formal ABI stability and LLVM LoongArch tier promotion**: The LoongArch ABI has evolved rapidly (v1.0 → v1.1 → v1.2). By 2031, with a stable v2.0 ABI, LLVM is expected to promote LoongArch to an official tier-1 supported target (currently tier-2), with continuous integration on loongarch64 hardware in the LLVM build infrastructure and full `check-all` gating on every commit.

---

## Chapter Summary

- **LoongArch** is a 32/64-bit little-endian RISC ISA from Loongson Technology, with LA32 (32-bit pointer) and LA64 (64-bit pointer) variants; all production hardware uses LA64.

- **Fixed-width 32-bit instructions** with no branch delay slots and no condition-code flags; branches compare registers directly (`beq`, `blt`, `bgeu`, etc.).

- **PC-relative addressing** uses `pcalau12i`+load for local/PIC references; `pcaddu18i`+`jr` (call36) for far function calls up to ±128 GiB. Note the distinction: `pcalau12i` aligns the result to a 4 KiB page boundary; `pcaddu12i` does not.

- **LP64D ABI** passes up to 8 integer args in `$a0`–`$a7`, up to 8 FP args in `$fa0`–`$fa7`, independently. Aggregates over 16 bytes are passed by pointer. Register `r21` is ABI-reserved and must not be used by portable code.

- **TLSDESC** (`-mtls-dialect=desc`) eliminates the `__tls_get_addr` call overhead of the GD model by storing a function pointer and argument in the GOT; after initial resolver invocation, subsequent accesses call a trivial offset-return function.

- **LSX** (128-bit, `$vr0`–`$vr31`) and **LASX** (256-bit, `$xr0`–`$xr31`) provide SIMD at the same register count as x86 (32 registers); LASX `$xr0` aliases the upper half of LSX `$vr0`.

- **Auto-vectorization** selects `vadd.w`/`vld`/`vst` for LSX and `xvadd.w`/`xvld`/`xvst` for LASX automatically; `LoongArchTargetTransformInfo` drives the profitability analysis.

- **LLD** supports full ELF linking, GOT/PLT generation, `R_LARCH_ALIGN` relaxation (removes optimistic `nop` padding after layout), and all TLSDESC relocation types.

- **JITLink** (ORC JIT) supports LoongArch ELF via `createLinkGraphFromELFObject_loongarch`, with edge kinds covering PC-relative branches, page/offset GOT references, `call36`, and alignment relaxation.

- **Production status**: LLVM 22.1.x is fully production-ready for loongarch64 with the SelectionDAG path; GlobalISel coverage is expanding; Linux mainlined support in 5.19; Debian, Fedora, Arch Linux, and openEuler all ship LoongArch ports.

---

*Cross-references:*
- [Chapter 98 — The RISC-V Backend Architecture](../part-15-targets/ch98-the-riscv-backend-architecture.md) — comparable PC-relative addressing, ABI design, and linker relaxation
- [Chapter 99 — The RISC-V Vector Extension (RVV)](../part-15-targets/ch99-the-riscv-vector-extension-rvv.md) — comparison of scalable vs fixed-width SIMD approaches
- [Chapter 101 — PowerPC, SystemZ, MIPS, SPARC, and LoongArch](../part-15-targets/ch101-powerpc-systemz-mips-sparc-loongarch.md) — LoongArch overview alongside other tier-2 targets
- [Chapter 96 — The AArch64 Backend](../part-15-targets/ch96-the-aarch64-backend.md) — TLSDESC history (AArch64 pioneered the model) and HFA aggregate passing rules
