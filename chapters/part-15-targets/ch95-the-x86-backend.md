# Chapter 95 — The X86 Backend

*Part XV — Targets*

The x86-64 backend is LLVM's most battle-tested target, shipping billions of binaries per day across every major operating system. It encodes 60 years of ISA evolution — from the 8086's 16-bit word through 32-bit IA-32, 64-bit AMD64, and the ongoing stream of vector extensions culminating in AVX-512 and the forthcoming APX (Advanced Performance Extensions). Mastering this backend means understanding not just code generation, but the deep interaction between ABI contracts, EFLAGS liveness, complex addressing modes, and the layered vector register hierarchy. This chapter dissects the X86 backend in LLVM 22.1.x: register classes, calling conventions, instruction selection for vector ISAs, APX and CET security features, and the DAG combining infrastructure that fuses patterns into efficient x86 sequences.

---

## 95.1 Register Classes and the Register Hierarchy

### 95.1.1 General-Purpose Registers

The x86-64 GPR file contains 16 64-bit registers (RAX–R15), each projecting sub-register views at 32-bit (EAX–R15D), 16-bit (AX–R15W), and 8-bit (AL/AH–R15B). LLVM models these through `X86RegisterInfo.td`:

```tablegen
// llvm/lib/Target/X86/X86RegisterInfo.td
def AL : X86Reg<"al", 0>;
def AX : X86Reg<"ax", 0, [AL, AH]>;
def EAX : X86Reg<"eax", 0, [AX]>;
def RAX : X86Reg<"rax", 0, [EAX]>;

def GR8  : RegisterClass<"X86", [i8],  8,  (add AL, CL, DL, BL, ...)>;
def GR16 : RegisterClass<"X86", [i16], 16, (add AX, CX, DX, BX, ...)>;
def GR32 : RegisterClass<"X86", [i32], 32, (add EAX, ECX, EDX, EBX, ...)>;
def GR64 : RegisterClass<"X86", [i64], 64, (add RAX, RCX, RDX, RBX, ...)>;
```

Sub-register writes have important semantics: writing a 32-bit register (e.g., `movl %eax, %eax`) zero-extends to 64 bits, while writing 8 or 16-bit sub-registers leaves upper bits unchanged. LLVM tracks these implicit definitions via `X86InstrInfo`'s operand constraints.

### 95.1.2 The Vector Register Hierarchy

x86 exposes three overlapping vector register sets:

| Extension | Register Width | Count | Class |
|-----------|---------------|-------|-------|
| SSE/SSE2 | XMM (128-bit) | 16 | `VR128` |
| AVX/AVX2 | YMM (256-bit) | 16 | `VR256` |
| AVX-512 | ZMM (512-bit) | 32 | `VR512` |
| AVX-512 | k (opmask, 64-bit) | 8 | `VK1–VK64` |

XMM registers are the low 128 bits of YMM, which are the low 256 bits of ZMM. The `VZEROALL` and `VZEROUPPER` instructions manage the spurious AVX/SSE transition penalties by zeroing upper halves.

```tablegen
// Opmask register classes for AVX-512
def VK1  : RegisterClass<"X86", [v1i1],   8, (add K0, K1, ..., K7)>;
def VK8  : RegisterClass<"X86", [v8i1],   8, (add K0, K1, ..., K7)>;
def VK16 : RegisterClass<"X86", [v16i1],  8, (add K0, K1, ..., K7)>;
def VK32 : RegisterClass<"X86", [v32i1],  8, (add K0, K1, ..., K7)>;
def VK64 : RegisterClass<"X86", [v64i1],  8, (add K0, K1, ..., K7)>;
```

---

## 95.2 Calling Conventions

### 95.2.1 System V AMD64 ABI

The SysV ABI governs Linux, macOS, FreeBSD, and Solaris:

- **Integer/pointer arguments**: rdi, rsi, rdx, rcx, r8, r9 (first 6)
- **Floating-point arguments**: xmm0–xmm7 (first 8)
- **Return values**: rax (integer), rax+rdx (128-bit integer), xmm0 (scalar FP), xmm0+xmm1 (complex)
- **Callee-saved**: rbx, rbp, r12–r15
- **Red zone**: 128 bytes below RSP are guaranteed scratch space; leaf functions avoid adjusting RSP

Aggregates follow a classification algorithm: structs whose fields fit in integer/SSE registers are passed in those registers; otherwise they are passed by implicit reference (pointer). The algorithm is implemented in `clang/lib/CodeGen/TargetInfo.cpp`:

```cpp
// clang/lib/CodeGen/TargetInfo.cpp
// X86_64ABIInfo::classify() assigns each aggregate field
// to INTEGER, SSE, SSEUP, X87, X87UP, COMPLEX_X87, or MEMORY class
ABIArgInfo X86_64ABIInfo::classifyReturnType(QualType RetTy) const {
  // ... classification code
}
```

### 95.2.2 Windows x64 ABI

Win64 differs significantly:

- **Arguments**: rcx, rdx, r8, r9 for the first 4 integer/pointer args; xmm0–xmm3 for the first 4 FP args (each slot consumes one position, integer or FP)
- **Shadow space**: 32 bytes above the return address, always reserved
- **Callee-saved**: rbx, rbp, rdi, rsi, r12–r15, xmm6–xmm15
- **No red zone**: interrupt handlers may trample below RSP

LLVM selects between these via `CallingConv::C` on Linux targets vs. `CallingConv::Win64`, or through the `x86_64-pc-windows-msvc` triple. The frame lowering in `X86FrameLowering.cpp` accounts for shadow space allocation and XMM spill slots.

---

## 95.3 SSE/AVX/AVX-512 Instruction Selection

### 95.3.1 SSE2 and SSE4 Lowering

SSE2 is the baseline for x86-64. LLVM patterns in `X86InstrSSE.td` match `v4f32`, `v2f64`, `v16i8`, `v8i16`, `v4i32`, `v2i64` operations:

```tablegen
// Packed single-precision add
def ADDPSrr : PSI<0x58, MRMSrcReg, (outs VR128:$dst),
                  (ins VR128:$src1, VR128:$src2),
                  "addps\t{$src2, $dst|$dst, $src2}",
                  [(set VR128:$dst,
                    (v4f32 (fadd VR128:$src1, VR128:$src2)))]>;
```

SSE4.1 adds `DPPS` (dot product), `BLENDPS`, `INSERTPS`, `PBLENDVB`, and `ROUNDSS/ROUNDPS` which LLVM maps from `llvm.x86.sse41.*` intrinsics and from DAG patterns such as `(ftrunc f32:$x)`.

### 95.3.2 AVX and AVX2

AVX introduces 256-bit YMM operations and a non-destructive 3-operand VEX encoding. LLVM's `X86InstrAVX512.td` generates both VEX-encoded (AVX/AVX2) and EVEX-encoded (AVX-512) forms. The backend automatically promotes `v8f32`, `v4f64`, `v32i8`, `v16i16`, `v8i32`, `v4i64` to YMM when `+avx` is present:

```bash
# Verify AVX2 codegen
clang -O2 -mavx2 -emit-llvm -S foo.c -o foo.ll
/usr/lib/llvm-22/bin/llc -march=x86-64 -mattr=+avx2 foo.ll -o foo.s
```

AVX2 extends integer SIMD to 256 bits: `VPADDB/VPADDW/VPADDD/VPADDQ`, `VPUNPCKLBW`, `VPSHUFB`, `VPMULLD`, and the critical gather instructions (`VGATHERDPS`).

### 95.3.3 AVX-512

AVX-512 is a family of extensions:

| Extension | Coverage |
|-----------|----------|
| AVX512F | Foundation: 512-bit FP, integer, masks |
| AVX512BW | Byte and word operations |
| AVX512DQ | Doubleword and quadword extensions |
| AVX512VL | 128/256-bit versions of AVX-512 ops |
| AVX512VNNI | Vector neural network integer instructions |
| AVX512BF16 | bfloat16 support |
| AVX512FP16 | fp16 vector operations |

EVEX encoding adds: 32 ZMM registers, 8 opmask registers (k0–k7), per-element masking (zeroing vs. merging), broadcast memory operands `{1to16}`, and embedded rounding control `{rn-sae}`.

```llvm
; AVX-512 masked add with zero-masking
%res = call <16 x float> @llvm.x86.avx512.mask.add.ps.512(
    <16 x float> %a, <16 x float> %b,
    <16 x float> zeroinitializer, i16 %mask, i32 4)
```

```bash
# Compile with AVX-512
/usr/lib/llvm-22/bin/llc -march=x86-64 -mattr=+avx512f,+avx512bw,+avx512vl input.ll
```

---

## 95.4 APX — Advanced Performance Extensions

Intel APX (targeted for Granite Rapids and Lion Cove) adds three major capabilities to x86-64:

### 95.4.1 Extended General-Purpose Registers (EGPRs)

APX doubles the GPR file from 16 to 32 registers: R16–R31. LLVM's APX support adds `GR64_NOREX2` and expands register allocation:

```tablegen
def R16 : X86Reg<"r16", 16>;
def R17 : X86Reg<"r17", 17>;
// ... through R31
def GR64_APX : RegisterClass<"X86", [i64], 64,
    (add RAX, RCX, ..., R31)>;
```

The new `REX2` prefix encoding (2-byte prefix) enables accessing R16–R31 in non-VEX, non-EVEX contexts.

### 95.4.2 New Data Destination (NDD) Form

NDD instructions permit three-operand non-destructive encoding for most integer ALU operations previously requiring a MOV before the operation:

```asm
; Classic 2-operand (destroys source)
mov    %rax, %rcx
add    %rbx, %rcx     ; rcx = rcx + rbx

; NDD 3-operand APX form
add    %rax, %rbx, %rcx  ; rcx = rax + rbx (rax preserved)
```

LLVM lowers NDD forms via `X86ISelLowering.cpp` when the `+apx` subtarget feature is set.

### 95.4.3 Conditional Instructions (CFCMOV, CFCMOV32)

APX extends `CMOV` to conditional stores and conditional load-modify-store, reducing branch frequency in predicated code:

```asm
; APX conditional move to memory
cfcmovz %eax, (%rdi)   ; store eax to [rdi] if ZF=1
```

---

## 95.5 Intel Control-Flow Enforcement Technology (CET)

### 95.5.1 Shadow Stack (SHSTK)

CET shadow stack maintains a parallel return-address stack in a CPU-enforced memory region. LLVM instruments prologues and epilogues when `-fcf-protection=return` is passed:

```cpp
// X86FrameLowering.cpp — shadow stack save/restore
void X86FrameLowering::emitShadowStackPrologue(...) {
  // SAVEPREVSSP / RSTORSSP instructions
}
```

The shadow stack prevents return-oriented programming (ROP) by detecting mismatches between the architectural stack and the shadow copy.

### 95.5.2 Indirect Branch Tracking (IBT)

IBT requires every valid indirect branch target to begin with `ENDBR64` (or `ENDBR32`). Clang emits `endbr64` at function entry points and indirect jump targets when `-fcf-protection=branch` is enabled:

```asm
foo:
  endbr64          # legal IBT target marker
  pushq  %rbp
  movq   %rsp, %rbp
  ...
```

LLVM marks `endbr64` as an implicit pseudo that the MC layer materializes. The `X86IndirectBranchTracking.cpp` pass inserts `ENDBR64` after indirect branch targets found during analysis.

---

## 95.6 Complex Addressing Modes

x86 supports a rich addressing mode: `[Base + Index * Scale + Displacement]` where Scale ∈ {1, 2, 4, 8}. LLVM encodes these as 5-field MachineOperand sequences:

```cpp
// MachineInstr addressing mode fields (in order)
// Base register | Scale | Index register | Displacement | Segment
// movq  foo(%rip), %rax  →  [RIP + 0 + 0 + foo_offset + 0]
```

The `X86AddressMode` structure in `X86ISelLowering.cpp` and the `matchAddress` DAG combiner pattern drive complex-addressing mode folding. The combiner attempts to absorb `add`, `mul`-by-scale, and constant-offset nodes into a single addressing mode operand, reducing instruction count significantly.

```tablegen
// Pattern for scaled index
def : Pat<(load (add (shl GR64:$idx, (i8 3)), GR64:$base)),
          (MOV64rm GR64:$base, 8, GR64:$idx, 0, 0)>;
```

---

## 95.7 Function Multiversioning and IFUNC

### 95.7.1 `__attribute__((target(...)))` Clones

GCC/Clang multiversioning creates multiple copies of a function optimized for different microarchitectures, dispatched via a CPU feature check at runtime:

```cpp
__attribute__((target("avx2")))
void compute_avx2(float* a, float* b, int n) { /* AVX2 path */ }

__attribute__((target("sse4.2")))
void compute_sse42(float* a, float* b, int n) { /* SSE4.2 fallback */ }

__attribute__((target("default")))
void compute(float* a, float* b, int n) { /* scalar fallback */ }
```

Clang generates a resolver function that calls `__cpu_supports()` from `compiler-rt` and dispatches to the appropriate clone. The resolver is an IFUNC (`STT_GNU_IFUNC`) on ELF platforms.

### 95.7.2 IFUNC Resolver Pattern

```c
// Generated resolver (conceptual)
static void (*resolve_compute(void))(float*, float*, int) {
  if (__builtin_cpu_supports("avx2")) return compute_avx2;
  if (__builtin_cpu_supports("sse4.2")) return compute_sse42;
  return compute_default;
}
__attribute__((ifunc("resolve_compute")))
void compute(float*, float*, int);
```

LLVM's `X86TargetInfo::isValidIFuncAttr()` enforces that IFUNCs appear only in ELF targets, and `X86TargetLowering::lowerGlobalTLSAddress()` handles the GOT-indirect reference for IFUNC PLT slots.

---

## 95.8 x86-Specific Lowering

### 95.8.1 SETCC and CMOV

x86 condition codes are produced by instructions like `CMP`, `TEST`, `SUB`. LLVM models EFLAGS as a virtual register but tracks liveness carefully through `X86FlagsCopyLowering.cpp` which handles copies of EFLAGS across basic blocks (e.g., for `LAHF`/`SAHF` patterns):

```tablegen
// SETCC lowering
def : Pat<(i8 (setne (i32 (trunc (and GR64:$l, GR64:$r))), (i32 0))),
          (SETNEr (TEST64rr GR64:$l, GR64:$r))>;
```

`CMOV` (conditional move) lowers `select` nodes when the condition is a comparison result. The `X86ISelLowering::lowerSELECT()` function handles cases involving chains of conditions, integer vs. FP select, and vector select (which maps to `BLENDV`).

### 95.8.2 Division and Shift Counts

Integer division on x86 requires `IDIV`/`DIV` which consume RDX:RAX and produce quotient in RAX and remainder in RDX. LLVM wraps this through a custom legalization:

```cpp
// X86ISelLowering.cpp
SDValue X86TargetLowering::LowerUDIVREM(SDValue Op, SelectionDAG &DAG) {
  // Extends dividend to RDX:RAX with CDQ/CQO
  // Issues IDIV/DIV
  // Extracts quotient (RAX) and remainder (RDX)
}
```

Shift counts on x86 must be in CL (or an immediate). The backend inserts `COPY_TO_REGCLASS` to force the shift count into GR8, and the register allocator uses the `X86RegisterInfo::getRegAllocationHints()` mechanism to prefer CL for shift-count operands.

### 95.8.3 x87 Legacy FPU

The x87 FPU uses a stack-based register file (ST(0)–ST(7)) and is still generated for `long double` (80-bit extended precision) on x86-64. LLVM's x87 handling in `X86ISelLowering.cpp` converts 80-bit operations through stack-push (`FLD`), operation, and stack-pop (`FSTP`), and legalizes `f80` types. The register allocator uses `RFP80` register class and the `X86FloatToX87` pass converts FP pseudo-ops to x87 stack instructions.

---

## 95.9 EFLAGS Modelling and Liveness

EFLAGS is a fundamental challenge: dozens of x86 instructions implicitly produce or consume condition flags. LLVM represents EFLAGS as a single virtual register for most of the compilation pipeline, then resolves copies and uses late.

Key passes:
- **`X86FlagsCopyLowering`** (`X86FlagsCopyLowering.cpp`): lowers `COPY %eflags` by recomputing the condition via the original compare
- **`X86CondBrFolding`**: folds condition-branch sequences for compact dispatch tables
- **`X86FixupBWInsts`** (`X86FixupBWInsts.cpp`): replaces byte/word operations with 32-bit equivalents where EFLAGS are not live-out, exploiting the zero-extension behavior

The EFLAGS liveness problem is particularly acute around `LAHF`/`SAHF` sequences used by `setjmp`/`longjmp` on Windows.

---

## 95.10 The X86 DAG Combining Infrastructure

`X86DAGToDAGISel.cpp` implements `Select()` which post-processes the generic DAG combiner's output. Key combining patterns:

### 95.10.1 Load-Op-Store Folding

```cpp
// Recognize read-modify-write: load → add → store to same address
// Generates: addl $1, (%rdi)  instead of: movl (%rdi), %eax / addl $1, %eax / movl %eax, (%rdi)
```

### 95.10.2 LEA Synthesis

`X86ISelLowering.cpp` contains `combineToX86LEA()` which fuses `add + shift_by_1_2_3` sequences into `LEA` instructions, capturing complex addressing for arithmetic (not just memory access):

```cpp
// a + b*2 + c  →  leal (%rax, %rbx, 2), %rcx  (if c fits in imm)
```

### 95.10.3 Shuffle Lowering

Vector shuffle is one of the most complex lowering tasks. `X86ISelLowering.cpp`'s `LowerVECTOR_SHUFFLE()` dispatches through a cascade of patterns:
1. Identity / broadcast → `MOVDDUP`, `VBROADCAST`
2. Unpack high/low → `PUNPCKLBW`, `PUNPCKHBW`
3. Byte shuffle → `VPSHUFB`
4. Cross-lane → `VPERM2F128`, `VPERMQ`
5. General → `VPERMILPS` + `VPERM2I128` composition

---

## 95.11 X86 Scheduling Models and Target Microarchitectures

LLVM includes scheduling models for over 20 x86 microarchitectures, selected via `-mcpu`. The model files live in `llvm/lib/Target/X86/X86Schedule*.td`:

| CPU model | File | Issue width | Notes |
|-----------|------|------------|-------|
| Skylake | `X86SchedSkylakeClient.td` | 4-wide OOO | Baseline for modern development |
| Skylake-X | `X86SchedSkylakeServer.td` | 4-wide + AVX-512 | Data center Skylake-SP |
| Cascade Lake | Similar to SKX | 4-wide | DL boost, VNNI |
| Icelake | `X86SchedIceLake.td` | 5-wide | Better 512-bit throughput |
| Alderlake | `X86SchedAlderlake.td` | Mixed P/E cores | Hybrid model |
| Sapphire Rapids | `X86SchedSapphireRapids.td` | 6-wide | AMX, AVX-512FP16 |
| Zen 4 (AMD) | `X86SchedZen4.td` | 6-wide | AVX-512 support |
| Zen 5 (AMD) | `X86SchedZen5.td` | 8-wide | 512-bit native |

```tablegen
// X86SchedSkylakeClient.td
def SkylakeClientModel : SchedMachineModel {
  let IssueWidth = 4;
  let MicroOpBufferSize = 224;  // OOO window
  let LoadLatency = 4;
  let MispredictPenalty = 14;
  let PostRAScheduler = 1;
}
// Execution ports: P0, P1, P5, P6 (ALU); P2, P3 (load); P4 (store)
def SKLPort0 : ProcResource<1>;
def SKLPort1 : ProcResource<1>;
def SKLPort5 : ProcResource<1>;
def SKLPort6 : ProcResource<1>;
```

Scheduling models drive:
1. **Machine instruction scheduling** (`X86InstrItineraries`): per-instruction port binding and latency
2. **Loop unroll factors**: the vectorizer queries `getUnrollFactor()` which uses IssueWidth
3. **Software pipelining**: for high-throughput loops the pipeliner uses port pressure

---

## 95.12 X86 ABI Lowering Details

### 95.12.1 Stack Frame Layout

The standard x86-64 Linux stack frame:

```
High addresses
┌─────────────────────────────┐ ← previous RSP (caller's stack)
│ Caller's local variables     │
├─────────────────────────────┤
│ Alignment padding (if needed) │
├─────────────────────────────┤
│ Stack arguments (>6 ints)    │
├─────────────────────────────┤ ← RSP on entry (8-byte aligned)
│ Return address (from CALL)   │ (pushed by hardware)
├─────────────────────────────┤ ← RSP after function entry
│ Saved RBP (if -fno-omit-fp) │
├─────────────────────────────┤ ← RBP (frame pointer)
│ Callee-saved regs            │
│ (RBX, RBP, R12-R15)         │
├─────────────────────────────┤
│ Local variables              │
│ Spill slots                  │
├─────────────────────────────┤
│ Red zone (128 bytes below)   │  (SysV only; not on Win64)
└─────────────────────────────┘ ← RSP (16-byte aligned at call sites)
```

The `X86FrameLowering::emitPrologue()` and `emitEpilogue()` functions generate the frame setup:

```asm
; Standard function prologue (with frame pointer)
foo:
    pushq  %rbp           ; save frame pointer
    movq   %rsp, %rbp     ; establish frame pointer
    subq   $48, %rsp      ; allocate locals + alignment
    ; callee-saved registers pushed as needed
```

### 95.12.2 Exception Handling and .eh_frame

On Linux, exception handling uses the System V ABI's DWARF-based `.eh_frame` section. LLVM generates CFI directives for every stack-modifying instruction:

```asm
foo:
    .cfi_startproc
    pushq  %rbp
    .cfi_def_cfa_offset 16
    .cfi_offset %rbp, -16
    movq   %rsp, %rbp
    .cfi_def_cfa_register %rbp
    ...
    popq   %rbp
    .cfi_def_cfa %rsp, 8
    retq
    .cfi_endproc
```

The `X86FrameLowering` inserts these CFI directives through the MachineFunction's `CFIInstructions` vector, which the AsmPrinter renders as `.cfi_*` directives.

### 95.12.3 TLS Implementation

Thread-local storage on x86-64 Linux uses the FS segment register:

```asm
; Initial-exec TLS (fast path for statically-linked TLS)
movq   %fs:0, %rax           ; read thread pointer
movq   foo@GOTTPOFF(%rip), %rcx  ; load IE offset
movq   (%rax, %rcx), %rax   ; load TLS variable

; General-dynamic TLS (shared library, fully relocatable)
leaq   foo@TLSGD(%rip), %rdi
call   __tls_get_addr@PLT    ; returns address of TLS slot
movq   (%rax), %rax
```

LLVM's `X86ISelLowering::lowerGlobalTLSAddress()` dispatches between the four TLS access models (local-exec, initial-exec, local-dynamic, general-dynamic) based on the code model and the symbol's visibility.

---

## 95.13 X86 Vector Lowering Pipeline

### 95.13.1 Type Legalization for Vectors

LLVM's type legalizer for x86 handles vectors by widening or splitting based on the available ISA:

```
v8i32 on target without AVX:
  → split into two v4i32 → mapped to two XMM operations

v8i32 on target with AVX2:
  → legal as YMM operation

v16i32 on target with AVX-512F:
  → legal as ZMM operation

v16i32 on target with AVX2 but no AVX-512:
  → split into two v8i32 YMM operations
```

The `X86TargetLowering::getTypeAction()` function defines legality for each combination.

### 95.13.2 AVX-512 Mask Legalization

AVX-512 opmask registers require special legalization. A `v16i1` (16 boolean mask) is:
- Not legal as a general register type
- Legal as a `VK16` opmask register
- Converted from/to integer via `KMOVW`/`KMOVD`/`KMOVQ`

```tablegen
// Mask register moves
def KMOVWkr : I<0x90, MRMSrcReg, (outs VK16:$dst), (ins GR32:$src),
    "kmovw\t{$src, $dst|$dst, $src}",
    [(set VK16:$dst, (v16i1 (bitconvert GR32:$src)))]>,
    T8PD, VEX, Requires<[HasAVX512]>;
```

### 95.13.3 Auto-Vectorization for x86

LLVM's Loop Vectorizer and SLP Vectorizer both generate x86 SIMD code. Key settings:

```bash
# Force 256-bit vectorization (YMM)
clang -O3 -mavx2 -mprefer-vector-width=256 foo.c -S -o foo.s

# Force 512-bit vectorization (ZMM) — may be slower due to frequency scaling
clang -O3 -mavx512f -mprefer-vector-width=512 foo.c -S -o foo.s

# Disable 512-bit to avoid frequency throttling on Skylake-X
clang -O3 -mavx512f -mprefer-vector-width=256 foo.c -S -o foo.s

# Check what vectorization happened
clang -O3 -mavx2 -Rpass=loop-vectorize foo.c 2>&1 | grep vectorized
```

---

## 95.14 X86 Instruction Encoding and the MC Layer

### 95.14.1 Instruction Prefix Anatomy

x86-64 instructions can be preceded by up to 15 bytes of prefixes:

```
Legacy prefixes (0–4 bytes): 
  F0 (LOCK), F2 (REPNZ), F3 (REP/REPZ), 66 (operand size), 67 (address size)
  CS/SS/DS/ES/FS/GS (segment overrides)

REX prefix (0–1 byte, byte 4xX):
  W bit: 64-bit operand size
  R bit: extends ModRM.reg to 4 bits
  X bit: extends SIB.index to 4 bits  
  B bit: extends ModRM.r/m or SIB.base to 4 bits

VEX prefix (2–3 bytes):
  2-byte form: C5 followed by R/vvvv/L/pp
  3-byte form: C4 followed by R/X/B/m-mmmm then W/vvvv/L/pp

EVEX prefix (4 bytes): 62 followed by 3 bytes
  z bit: zeroing masking
  aaa: opmask register (k0–k7)
  b bit: broadcast / embedded rounding

APX REX2 prefix (2 bytes):
  D5 followed by M/R4/X4/B4/W/R3/X3/B3
```

The `X86MCCodeEmitter` in `X86MCCodeEmitter.cpp` selects the appropriate prefix encoding for each instruction based on the register operands and feature requirements.

### 95.14.2 ModRM and SIB Encoding

The ModRM byte encodes the operand mode:

```
ModRM: [ mod(2) | reg(3) | r/m(3) ]
mod=11: register-register
mod=00: [r/m]  or  [RIP + disp32] (r/m=101)
mod=01: [r/m + disp8]
mod=10: [r/m + disp32]
r/m=100 with mod≠11: SIB follows

SIB: [ ss(2) | index(3) | base(3) ]
address = base_reg + index_reg × 2^ss + displacement
```

LLVM's `X86AsmPrinter` calls `X86MCCodeEmitter::encodeInstruction()` which marshals these fields from the `MCInst` representation. The `X86InstrInfo.td` definitions include the `MRM` encoding specifiers (`MRMSrcReg`, `MRMDestMem`, etc.) that drive this encoding.

---

## 95.15 Using llc for X86 Code Generation

```bash
# Emit x86-64 assembly with AVX-512 features
/usr/lib/llvm-22/bin/llc -march=x86-64 \
    -mattr=+avx512f,+avx512bw,+avx512vl,+avx512dq \
    -mcpu=skylake-avx512 \
    input.ll -o output.s

# Show register allocation details
/usr/lib/llvm-22/bin/llc -march=x86-64 -mattr=+avx2 \
    -print-after=greedy \
    input.ll -o /dev/null 2>&1 | head -100

# Verify addressing mode folding
/usr/lib/llvm-22/bin/llc -march=x86-64 \
    -debug-only=isel \
    input.ll -o /dev/null 2>&1 | grep -i "addressing"

# APX code generation (requires APX-capable build)
clang -O2 -march=graniterapids-x86-64 -mapx-features=egpr,push2pop2,ndd \
    foo.c -S -o foo_apx.s

# Profile-guided optimization with AutoFDO
clang -O3 -march=native -fprofile-use=profile.afdo \
    -mllvm -enable-chr=true \
    foo.c -o foo_pgo

# Check scheduling model is active
/usr/lib/llvm-22/bin/llc -march=x86-64 -mcpu=alderlake \
    -debug-only=machine-scheduler \
    input.ll -o /dev/null 2>&1 | grep "Scheduling" | head -20
```

---

## Chapter Summary

- x86-64 has a layered register hierarchy: 16 GPRs with 8/16/32/64-bit projections, 32 ZMM/16 YMM/16 XMM vector registers (aliased), and 8 opmask registers for AVX-512 predication.
- SysV AMD64 passes 6 integer args in registers (rdi/rsi/rdx/rcx/r8/r9) with a 128-byte red zone; Win64 passes 4 in shadow-space layout (rcx/rdx/r8/r9) with mandatory shadow space, no red zone.
- AVX-512's EVEX encoding enables 32 ZMM registers, per-element masking (merging/zeroing), broadcast memory operands, and embedded rounding—all modelled in LLVM through EVEX-prefixed instruction definitions and VK register classes.
- APX adds 16 new EGPRs (R16–R31), NDD 3-operand integer ALU, and conditional store/load-modify-store instructions, encoded via the new REX2 prefix.
- Intel CET is implemented in two halves: shadow-stack (SHSTK) instruments function prologues/epilogues, while indirect-branch tracking (IBT) inserts `endbr64` at all legal indirect branch targets via the `X86IndirectBranchTracking` pass.
- x86's complex addressing mode (Base + Index × Scale + Disp) is exploited by the DAG combiner via `matchAddress()`, which absorbs arithmetic nodes into addressing operands.
- Function multiversioning with `__attribute__((target(...)))` produces per-ISA clones dispatched through IFUNC resolvers using `__cpu_supports()` from `compiler-rt`.
- EFLAGS liveness is tracked as a virtual register throughout codegen, resolved by `X86FlagsCopyLowering` which rewrites EFLAGS copies as condition recomputation.
- The X86 DAG combiner performs load-op-store folding, LEA synthesis from add+shift patterns, and a hierarchical shuffle decomposition that maps general shuffle masks to specific x86 shuffle instructions.
