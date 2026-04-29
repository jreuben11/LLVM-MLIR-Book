# Chapter 87 — Inline Assembly Lowering

*Part XIV — The Backend*

Inline assembly is the escape hatch that allows C and C++ programmers to embed target-specific instructions directly in source code. From the compiler's perspective, inline assembly is a challenge: the compiler must parse constraint strings, assign registers, respect clobbers, thread memory ordering, and ultimately emit the assembly text through the integrated assembler — all without being able to reason about the semantics of the embedded instructions themselves. This chapter traces the full path from an `asm` statement in LLVM IR through constraint parsing, the `ISD::INLINEASM` DAG node, the `INLINEASM` MachineInstr encoding, clobber handling, and final emission through the MC layer.

## 87.1 Inline Assembly in LLVM IR

### 87.1.1 The IR Representation

LLVM IR represents inline assembly using `call` instructions with an inline-asm value as the callee:

```llvm
; AT&T syntax, two operands, output in register:
%result = call i32 asm "mov $1, $0", "=r,r"(i32 %input)

; With clobbers:
call void asm sideeffect "cpuid", "~{eax},~{ebx},~{ecx},~{edx}"()

; Memory clobber:
call void asm sideeffect "mfence", "~{memory}"()

; With tied operands (input/output tied):
%r = call i32 asm "add $2, $0", "=r,0,r"(i32 %a, i32 %b)

; Intel syntax (MS-style):
%r = call i32 asm inteldialect "mov eax, $1", "=r,r"(i32 %x)
```

The `call` target is an `InlineAsm` value created by `InlineAsm::get()`:

```cpp
// llvm/include/llvm/IR/InlineAsm.h
class InlineAsm : public Value {
  std::string AsmString;      // The asm text
  std::string Constraints;    // Constraint string
  FunctionType *FTy;          // Type of the asm call
  bool HasSideEffects;        // "volatile" / "sideeffect"
  bool IsAlignStack;
  AsmDialect Dialect;         // AD_ATT or AD_Intel
};
```

The `HasSideEffects` flag corresponds to `asm volatile` in C. When set, the asm is never eliminated even if its result is unused.

### 87.1.2 Constraint String Syntax

The constraint string follows the GCC constraint syntax, extended by LLVM:

```
output-constraint ::= "=" modifier letter
input-constraint  ::= modifier? letter
clobber           ::= "~{" register-name "}"
separator         ::= ","
```

Modifiers before constraint letters:

| Modifier | Meaning |
|---|---|
| `=` | Output operand (write-only) |
| `+` | Input-output operand (read and write) |
| `&` | Early-clobber: written before inputs are consumed |
| `%` | Commutative: the instruction can swap the operand with the next |

Common constraint letters:

| Letter | Meaning (typical) |
|---|---|
| `r` | Any general-purpose register |
| `f` | Floating-point register |
| `m` | Memory operand |
| `i` | Immediate integer |
| `n` | Non-floating-point immediate |
| `g` | General operand (register, memory, or immediate) |
| `0`–`9` | Tied to the N-th operand (same physical register) |

Target-specific letters extend this:

```
x86:   "a"=eax/rax, "b"=ebx/rbx, "c"=ecx/rcx, "d"=edx/rdx
       "A"=eax/edx pair,  "q"=a/b/c/d,  "Q"=a/b/c  (no edx)
arm:   "l"=r0-r7, "h"=r8-r15, "w"=VFP register
```

## 87.2 SelectionDAGBuilder: Building the INLINEASM Node

### 87.2.1 visitInlineAsm

When `SelectionDAGBuilder` encounters a `call` to an `InlineAsm` value, it calls `visitInlineAsm`:

```cpp
// SelectionDAGBuilder.cpp
void SelectionDAGBuilder::visitInlineAsm(const CallBase &Call,
                                          const BasicBlock *EHPadBB) {
  const InlineAsm *IA = cast<InlineAsm>(Call.getCalledOperand());

  /// Collect all info about this asm:
  TargetLowering::AsmOperandInfoVector TargetConstraints =
    TLI.ParseConstraints(DAG.getDataLayout(), TRI, Call);

  // ... resolve each operand (register, memory, immediate)
  // Build the INLINEASM DAG node:
  std::vector<SDValue> AsmNodeOperands;
  AsmNodeOperands.push_back(Chain);
  AsmNodeOperands.push_back(DAG.getTargetExternalSymbol(
      IA->getAsmString().c_str(), TLI.getPointerTy(DAG.getDataLayout())));
  AsmNodeOperands.push_back(DAG.getMDNode(/*MD=*/nullptr)); // srcloc
  AsmNodeOperands.push_back(DAG.getTargetConstant(
      IA->hasSideEffects() | IA->isAlignStack() << 1 |
      IA->getDialect() << 2, DL, MVT::i32));
  // ... add operands for each constraint
}
```

### 87.2.2 ParseConstraints

`TargetLowering::ParseConstraints` parses the constraint string and returns an `AsmOperandInfoVector`:

```cpp
// TargetLowering.h
struct AsmOperandInfo {
  InlineAsm::ConstraintInfo Type;  // isOutput, isInput, isClobber
  std::string ConstraintCode;      // "r", "m", "0", etc.
  std::string ConstraintType;      // "Register", "Memory", "Immediate"
  ConstraintWeight Weight;         // preference score
  unsigned MatchingInput;          // for tied operands, which input
  EVT ConstraintVT;                // resolved value type
  SmallVector<unsigned, 4> Regs;   // allocated physical registers
};
```

After parsing, `TargetLowering::ComputeConstraintToUse` selects the best constraint letter when an operand has multiple alternatives (e.g., `"r,m"` — prefer register, fall back to memory):

```cpp
void TargetLowering::ComputeConstraintToUse(AsmOperandInfo &OpInfo,
                                             SDValue Op) const {
  // If both "r" and "m" are valid, prefer "r":
  for (const auto &Code : OpInfo.Codes) {
    TargetLowering::ConstraintType T = getConstraintType(Code);
    if (T == TargetLowering::C_Register) {
      OpInfo.ConstraintCode = Code;
      return;
    }
  }
  // Fall back to memory:
  OpInfo.ConstraintCode = OpInfo.Codes[0];
}
```

### 87.2.3 Constraint → Register Class Mapping

For each register constraint letter, `TargetLowering::getRegClassForInlineAsmConstraint` returns the target's register class:

```cpp
// X86ISelLowering.cpp
std::vector<unsigned>
X86TargetLowering::getRegClassForInlineAsmConstraint(
    const TargetRegisterInfo *TRI, StringRef Constraint, MVT VT) const {
  if (Constraint.size() == 1) {
    switch (Constraint[0]) {
    case 'r':
      if (VT == MVT::i64)
        return {X86::GR64RegClassID};
      if (VT == MVT::i32)
        return {X86::GR32RegClassID};
      if (VT == MVT::i16)
        return {X86::GR16RegClassID};
      return {X86::GR8RegClassID};
    case 'f':
      return {X86::RFP64RegClassID};
    case 'y':
      return {X86::VR64RegClassID};  // MMX
    case 'x':
      return {X86::VR128RegClassID}; // SSE
    case 'v':
      return {X86::VR512RegClassID}; // AVX512
    }
  }
  return TargetLowering::getRegClassForInlineAsmConstraint(TRI, Constraint, VT);
}
```

The register class is then used by the register allocator to pick an actual physical register.

## 87.3 The ISD::INLINEASM DAG Node

### 87.3.1 Node Structure

The `ISD::INLINEASM` node (or `ISD::INLINEASM_BR` for asm goto) encodes all information about the inline assembly in a specific operand layout:

```
ISD::INLINEASM node operands (in order):
  [0]  Token chain (ordering with surrounding memory ops)
  [1]  External symbol: the asm string
  [2]  MDNode: source location information
  [3]  i32 flag: HasSideEffects | (IsAlignStack << 1) | (AsmDialect << 2)
  [4..N] Pairs of: (Flag, Operand)
       Flag: encoded as i32 with kind (register, immediate, memory),
             reg class ID, number of registers
       Operand: the SDValue (register, constant, or frame index)
  [N+1] (optional) Glue output
```

The flag encoding is defined in `llvm/include/llvm/IR/InlineAsm.h`:

```cpp
// InlineAsm.h
// Flag word format:
// bits[2:0]  = OperandType (isReg=1, isImm=2, isMem=3, isBB=4)
// bits[15:3] = register class ID (for register constraints)
// bits[31:16] = number of registers (for multi-register constraints)
static unsigned getFlagWord(unsigned Kind, unsigned NumVals) { ... }
static unsigned getFlagWordForRegClass(unsigned InputFlag, unsigned RC) { ... }
```

### 87.3.2 Memory Constraint Representation

For memory constraints (`"m"`), the operand in the INLINEASM node is a base+offset pair or a frame index:

```
; C source: asm("movl %0, %%eax" : : "m"(x))
; DAG:
t5: i32,ch = INLINEASM
    chain: t0
    asm: ExternalSymbol "movl %0, %%eax"
    flags: i32<HasSideEffects=0>
    memflag: i32<MemConstraint|InputFlag>
    ptr: FrameIndex<x>        ; base pointer
    off: i32<0>               ; offset
```

### 87.3.3 Tied Operands

The constraint string `"=r,0"` means the output and the first input share the same physical register. In the INLINEASM node, the input operand for a tied constraint uses a special flag that encodes the index of the output it is tied to:

```cpp
// In SelectionDAGBuilder::visitInlineAsm:
if (OpInfo.isMatchingInputConstraint()) {
  unsigned DefIdx = OpInfo.getMatchedOperand();
  // Flag the input with a "tied" marker pointing to DefIdx:
  AsmNodeOperands.push_back(
      DAG.getTargetConstant(InlineAsm::getFlagWordForMatchingOp(
                                InputFlag, DefIdx),
                            DL, MVT::i32));
}
```

These tied operands become critical for the `TwoAddressInstruction` pass later: the pass must ensure that the input and output of the tied pair are in the same physical register before the asm executes.

## 87.4 The INLINEASM MachineInstr

### 87.4.1 From SDNode to MachineInstr

After instruction selection, the `ISD::INLINEASM` node becomes an `INLINEASM` `MachineInstr`. The operand layout is similar to the DAG node but uses `MachineOperand` objects:

```
INLINEASM:
  MO[0]: ExternalSymbol "asm string"
  MO[1]: Imm (flags: sideeffects, alignstack, dialect)
  [groups of: MO[k] = Imm (operand descriptor), MO[k+1..k+n] = operands]
  last:  Imm (extra info: HasSideEffects, IsAlignStack)
```

The operand descriptor integer encodes the constraint kind:

```cpp
// InlineAsm.h descriptor bits for INLINEASM MachineInstr:
enum : uint32_t {
  Kind_RegDef    = 1,   // output register (def)
  Kind_RegDefEarlyClobber = 6,  // output, early-clobber
  Kind_RegUse    = 3,   // input register (use)
  Kind_Imm       = 5,   // immediate operand
  Kind_Mem       = 4,   // memory operand
  Kind_Func      = 2,   // function label (for asm goto)
};
```

### 87.4.2 Register Allocation and INLINEASM

The register allocator sees the `INLINEASM` MachineInstr and handles it specially. For register operands, it allocates physical registers from the specified register class. For tied operands, it enforces that the tied input and output use the same physical register (inserting copies if needed).

The `InlineAsm` operand descriptor carries the register class ID, which the register allocator uses to restrict allocation:

```cpp
// RegAllocGreedy.cpp (conceptual):
if (MI.isInlineAsm()) {
  for (unsigned i = InlineAsm::MIOp_FirstOperand;
       i < MI.getNumOperands(); ) {
    unsigned Flag = MI.getOperand(i).getImm();
    unsigned Kind = InlineAsm::getKind(Flag);
    if (Kind == InlineAsm::Kind_RegDef || Kind == InlineAsm::Kind_RegUse) {
      unsigned RCID = InlineAsm::getRegClass(Flag);
      const TargetRegisterClass *RC = TRI->getRegClass(RCID);
      // Allocate within RC:
      allocateReg(MI.getOperand(i+1), RC);
    }
    i += 1 + InlineAsm::getNumOperandRegisters(Flag);
  }
}
```

### 87.4.3 Dumping INLINEASM MachineInstrs

The `INLINEASM` MachineInstr can be dumped in a human-readable form:

```bash
# Print the MIR after register allocation, focusing on INLINEASM:
llc -mtriple=x86_64 -print-after=regallocgreedy input.ll 2>&1 | grep -A5 INLINEASM
```

Example output:
```
INLINEASM &"mov $1, $0" [sideeffect] [attdialect],
  def %eax (tied-def 3), regclass:GR32,
  %ecx (tied 0), regclass:GR32,
  implicit-def $eflags
```

## 87.5 Clobbers

### 87.5.1 Register Clobbers

Clobbers listed as `~{regname}` in the constraint string tell the compiler which registers are destroyed by the inline assembly, even though they are not explicit operands. These become implicit-def operands on the `INLINEASM` MachineInstr:

```cpp
// SelectionDAGBuilder::visitInlineAsm — clobber handling:
for (const auto &Clobber : IA->getClobbers()) {
  if (Clobber == "memory" || Clobber == "cc") continue;
  // Map register name → physical register:
  Register Reg;
  if (TRI->getRegFromName(Clobber, Reg)) {
    Ops.push_back(DAG.getRegister(Reg, MVT::Other));  // as clobber
  }
}
```

In the MachineInstr form, these become `implicit-def` operands for the physical registers:

```
INLINEASM &"cpuid",
  implicit-def $eax, implicit-def $ebx,
  implicit-def $ecx, implicit-def $edx
```

The register allocator uses these `implicit-def` marks to know that the values of those registers are dead after the asm.

### 87.5.2 The "memory" Clobber

The `~{memory}` clobber is special: it does not clobber a specific register but instead signals that the inline assembly may read or write arbitrary memory. The backend handles this by:

1. Setting the `HasSideEffects` flag on the `ISD::INLINEASM` node (preventing it from being moved or eliminated).
2. Adding a memory operand to the `MachineInstr` that conservatively marks all memory as potentially accessed.
3. Preventing load-store optimizations from reordering across the asm.

```cpp
// In SelectionDAGBuilder:
if (IA->hasSideEffects() || hasMemClobber) {
  // Fold the asm into the chain to prevent reordering:
  SDValue Chain = getRoot();
  // ...
  // The INLINEASM node will have MachineMemOperand covering all memory
}
```

### 87.5.3 The "cc" (Condition Code) Clobber

Many architectures have condition code registers (`EFLAGS` on x86, `CPSR` on ARM). The `~{cc}` clobber marks these as modified. On x86, this maps to `$eflags`:

```
INLINEASM &"add $1, $0", ..., implicit-def $eflags
```

The liveness analysis will mark `$eflags` as dead at the end of any use-def chain through the asm.

## 87.6 Constraint Enforcement: TwoAddressInstruction

### 87.6.1 Tied Operand Coalescing

The `TwoAddressInstructionPass` handles tied operands in inline assembly. When an input is tied to an output (`"=r,0"`), the pass inserts a copy of the input into the output register before the asm:

```
; Before TwoAddressInstruction:
%2 = COPY %0
INLINEASM "add $2,$0", def %1(tied-def 3), %0(tied 0), %2

; After TwoAddressInstruction:
%1 = COPY %0          ; copy input to output register
INLINEASM "add $2,$0", def %1(tied-def 3), %1(tied 0), %2
```

After register allocation, if `%0` and `%1` are assigned different physical registers, the `COPY` becomes a real register-to-register move. If they are coalesced to the same register, the COPY is eliminated.

### 87.6.2 Early-Clobber Operands

Early-clobber output operands (`=&r`) must not share a register with any input operand. The `TwoAddressInstructionPass` enforces this by renaming conflicting inputs before the asm:

```
; "=&r" means output is written before all inputs are consumed.
; Input operand must not be in the same register as the output.
INLINEASM "xchg $0, $1", def $eax [earlyclobber], $ebx
; If the allocator tried to put $ebx in $eax, it would be wrong.
; The early-clobber constraint prevents this.
```

## 87.7 MC Layer: From MachineInstr to Emitted Text

### 87.7.1 AsmPrinter and Inline Asm

The `AsmPrinter` pass, which runs after register allocation, is responsible for converting `MachineInstr` objects to either object code (via MC) or assembly text. For `INLINEASM` MachineInstrs, `AsmPrinter::EmitInlineAsm` handles the emission:

```cpp
// AsmPrinter.cpp
void AsmPrinter::EmitInlineAsm(const MachineInstr *MI) const {
  assert(MI->isInlineAsm() && "Not an inline asm!");

  // Extract the asm string:
  const char *AsmStr = MI->getOperand(0).getSymbolName();

  // Collect all operands (registers, immediates, memory refs):
  SmallVector<const MachineOperand*, 8> MachineOperands;
  // ... extract operands from the MI ...

  // Substitute $0, $1, ... with actual register names:
  std::string SubstitutedStr;
  for (const char *p = AsmStr; *p; ++p) {
    if (*p == '$') {
      // Parse operand reference and replace with register name:
      SubstitutedStr += getRegisterName(MachineOperands[N]);
    } else {
      SubstitutedStr += *p;
    }
  }

  // Pass to the MC parser:
  EmitInlineAsm(SubstitutedStr, ...);
}
```

### 87.7.2 Operand Substitution

Substituting `$0`, `$1`, etc. with actual register names requires knowing the allocated physical registers. At this point (after register allocation), all virtual registers have been replaced by physregs:

```cpp
// For a register operand:
std::string AsmPrinter::getRegisterNameForOperand(
    const MachineOperand &MO) const {
  if (MO.isReg()) {
    Register Reg = MO.getReg();
    // Use the IA output modifier to pick sub-register:
    return TRI->getRegAsmName(Reg);
  }
  if (MO.isImm())
    return "$" + std::to_string(MO.getImm());
  // memory: emit base+offset notation
  // ...
}
```

Output modifiers like `%w0` (AArch64: 32-bit name of 64-bit register) or `%k0` (x86: `e`-prefix form) modify the register name. These are parsed during operand substitution by `AsmPrinter::AP_OUTPUT_modifier`.

### 87.7.3 The Integrated Assembler

After operand substitution, the resulting string is passed to the MC parser (integrated assembler) via `MCStreamer::EmitRawText` or, preferably, through full parsing:

```cpp
// AsmPrinter::EmitInlineAsm (lower-level path):
void AsmPrinter::EmitInlineAsm(StringRef Str, ...) const {
  std::unique_ptr<MemoryBuffer> Buffer = MemoryBuffer::getMemBuffer(Str);
  SourceMgr SrcMgr;
  SrcMgr.AddNewSourceBuffer(std::move(Buffer), SMLoc());

  // Create a new assembler parser for this string:
  std::unique_ptr<MCAsmParser> Parser(
      createMCAsmParser(SrcMgr, *MMI->getContext(), *OutStreamer, *MAI));
  std::unique_ptr<MCTargetAsmParser> TargetParser(
      TM.getTarget().createMCAsmParser(
          *Streamer->getTargetStreamer(),
          *Parser, *MII, MCOptions));
  Parser->setTargetParser(*TargetParser);
  Parser->Run(/*NoInitialTextSection=*/true, /*NoFinalize=*/true);
}
```

The integrated assembler parses the substituted asm string, encodes the instructions to binary, and emits them into the object file's current section. This has the advantage over a two-pass approach (compile to text, then assemble) that the compiler's context (symbol table, section layout) is available during assembly.

### 87.7.4 Inline Asm in Object Files

The encoded bytes from the integrated assembler are placed directly in the `.text` section (or whatever section is currently active). Relocations for symbols referenced inside the asm are emitted as usual through the `MCObjectWriter`.

For assembly text output (`-S` flag), `EmitRawText` writes the substituted string directly to the output stream, preserving it as human-readable assembly.

## 87.8 MS-Style Inline Assembly

### 87.8.1 Intel Dialect

LLVM supports two inline assembly dialects:

- **AT&T dialect** (default): source before destination, `%reg` register prefix, `$imm` immediate prefix.
- **Intel dialect**: destination before source, bare register names, no `$` for immediates.

In LLVM IR, the dialect is specified with `inteldialect`:

```llvm
; Intel dialect: destination first, no $ prefix
%r = call i32 asm inteldialect "mov eax, $1", "={eax},r"(i32 %x)
```

The dialect flag is stored in bit 2 of the INLINEASM flags operand and consulted during operand substitution.

### 87.8.2 MS-Style Named Output Operands

The Microsoft Visual C++ inline assembler extension uses `__asm` blocks with named variables instead of `$N` syntax:

```c
// MSVC-style (not natively supported; Clang translates to LLVM asm):
int x = 5;
__asm {
  mov eax, x
  add eax, 1
  mov x, eax
}
```

Clang's `CGStmt.cpp` (`CodeGenFunction::EmitAsmStmt`) translates `__asm` blocks to the LLVM inline assembly call syntax. The translation resolves named operands to numbered `$N` references and constructs the appropriate constraint string.

### 87.8.3 Parsing MS-Style Asm in Clang

For MSVC-compatible targets, Clang includes a special path for `__asm` blocks. The `MCAsmParser` is invoked in Microsoft mode:

```cpp
// Clang frontend: CGStmt.cpp
void CodeGenFunction::EmitMSAsmStmt(const MSAsmStmt &S) {
  // Collect inputs/outputs/clobbers from the block:
  // ...
  // Create the LLVM InlineAsm node:
  llvm::InlineAsm *IA = llvm::InlineAsm::get(
      FTy, AsmString, ConstraintStr,
      /*HasSideEffects=*/true, /*IsAlignStack=*/false,
      llvm::InlineAsm::AD_Intel);
  // Emit the call:
  Builder.CreateCall(IA, Args);
}
```

The `AD_Intel` flag causes the operand substitution to use Intel syntax during `AsmPrinter::EmitInlineAsm`.

## 87.9 Debugging Inline Assembly

### 87.9.1 Printing the INLINEASM MachineInstr

```bash
# Print the function's MIR after register allocation:
llc -mtriple=x86_64 -print-after=regallocgreedy input.ll 2>&1 | grep -B2 -A10 INLINEASM

# Print before and after TwoAddress:
llc -mtriple=x86_64 -print-before=twoaddressinstruction \
    -print-after=twoaddressinstruction input.ll 2>&1
```

### 87.9.2 Inspecting the INLINEASM DAG Node

```bash
# View the DAG just before instruction selection (after legalization):
llc -mtriple=x86_64 -view-legalize-dags input.ll
# The INLINEASM node will appear with all its operands labeled.
```

### 87.9.3 Constraint Verification

LLVM does limited verification of inline asm constraints. To catch problems early:

```bash
# Use clang with -Wall to catch obvious constraint errors:
clang -mtriple=x86_64 -Wall -O1 input.c -o /dev/null

# The integrated assembler will report parse errors in the asm string:
# error: inline asm: invalid operand in input list
```

Common error patterns:
- Mismatch between number of `$N` references and constraints.
- Using a register class that doesn't exist for the target.
- Tied operand index out of range.
- `"memory"` clobber missing when asm writes through a pointer.

### 87.9.4 End-to-End Example

```c
// test.c
unsigned add_asm(unsigned a, unsigned b) {
  unsigned result;
  __asm__("addl %2, %0"
          : "=r"(result)
          : "0"(a), "r"(b));
  return result;
}
```

```bash
# Compile to LLVM IR:
clang -target x86_64-linux-gnu -O1 -S -emit-llvm test.c -o test.ll
```

```llvm
; Generated IR:
define i32 @add_asm(i32 %a, i32 %b) {
  %1 = call i32 asm "addl $2, $0", "=r,0,r"(i32 %a, i32 %b)
  ret i32 %1
}
```

```bash
# Compile to assembly, showing the final output:
clang -target x86_64-linux-gnu -O1 -S test.c -o -
```

```asm
# Generated x86-64 assembly:
add_asm:
    movl    %edi, %eax      # copy first arg (tied operand)
    #APP
    addl    %esi, %eax      # the inline asm instruction
    #NO_APP
    retq
```

The `#APP` / `#NO_APP` markers bracket the inline assembly in the text output. The copy of `%edi` to `%eax` was inserted by the tied-operand handling.

## 87.10 The asm goto Extension

### 87.10.1 ISD::INLINEASM_BR

GCC's `asm goto` extension allows inline assembly to branch to C labels:

```c
// asm goto: assembly may jump to 'label'
asm goto("jmp %l[label]" : : : : label);
label:
  return 1;
```

In LLVM IR:

```llvm
callbr void asm "jmp ${0:l}", "X,!i"(label %label)
         to label %fallthrough [label %label]
```

The `callbr` instruction uses `ISD::INLINEASM_BR` in the DAG, which has additional basic block successors. The constraint `"X"` marks an indirect-branch operand; `"!i"` is a constraint on the label operand.

`ISD::INLINEASM_BR` is selected to `INLINEASM_BR` MachineInstr, which has both a fall-through successor and indirect branch target successors. The branch target labels are passed as `MO_MBB` operands.

### 87.10.2 Indirect Branch Operands

The label `%l[label]` in the asm string is substituted with the address of the target label. On x86, this becomes:

```asm
jmp .Lbb0_label   # substituted absolute address
```

On targets requiring position-independent code, the indirect branch operand may be a PC-relative offset or a GOT entry.

## Chapter Summary

- In LLVM IR, inline assembly is represented as a `call` to an `InlineAsm` value carrying the asm string, constraint string, side-effects flag, and dialect. The constraint string encodes output (`=r`), input (`r`), tied (`0`), and clobber (`~{reg}`) operands using GCC constraint syntax.

- `SelectionDAGBuilder::visitInlineAsm` calls `TargetLowering::ParseConstraints` to parse the constraint string into `AsmOperandInfo` records, then builds an `ISD::INLINEASM` DAG node. The node operand layout encodes each constraint as a flag-word integer followed by the actual operand SDValues.

- Register constraints are resolved to register class IDs via `getRegClassForInlineAsmConstraint`. Memory constraints produce base+offset operand pairs. Tied operands are flagged with the index of the associated output.

- The `INLINEASM` MachineInstr preserves the full operand structure from the DAG node. The `TwoAddressInstructionPass` enforces tied-operand co-location, inserting copies as needed. Early-clobber operands (`=&r`) prevent the allocator from assigning the output to any input register.

- Register clobbers (`~{eax}`) become `implicit-def` operands on the MachineInstr. The `~{memory}` clobber sets `HasSideEffects`, preventing reordering across the asm. The `~{cc}` clobber clobbers condition code registers.

- `AsmPrinter::EmitInlineAsm` substitutes `$N` references with actual physical register names after register allocation, then passes the result to the integrated assembler (MCAsmParser) for parsing and binary encoding.

- Intel dialect (`inteldialect` in IR, `AD_Intel` enum) reverses the operand order and omits `%` and `$` prefixes. MS-style `__asm` blocks are lowered by Clang's `CodeGenFunction::EmitMSAsmStmt`.

- `asm goto` is represented by `callbr` in IR and `ISD::INLINEASM_BR` in the DAG, adding basic block successor operands to the normal INLINEASM structure. Label targets are substituted as PC-relative or absolute addresses in the asm string.

- Debugging relies on `-print-after=regallocgreedy` to inspect the final MachineInstr, `-view-legalize-dags` to see the DAG node structure, and clang's `-Wall` plus the integrated assembler's error messages for constraint validation.


---

@copyright jreuben11
