# Chapter 94 — The MC Layer and MIR Test Infrastructure

*Part XIV — The Backend*

The MC (Machine Code) layer is the lowest abstraction stratum in the LLVM backend, bridging the gap between the register-allocated, frame-lowered `MachineInstr` representation and the concrete bytes or assembly text that constitute the output artifact. It provides a uniform API for emitting assembly and object code that is used by the integrated assembler, the standalone `llvm-mc` tool, the disassembler, and the `AsmPrinter` backend. This chapter covers the MC layer's principal classes — `MCInst`, `MCStreamer`, `MCAssembler`, `MCObjectWriter` — the fixup and relocation pipeline, the disassembly path, and the `.mir` text format with its `lit`-based regression testing infrastructure.

## 94.1 The MC Layer Architecture

The MC layer (`llvm/include/llvm/MC/`, `llvm/lib/MC/`) was designed to factor out the common assembly and object-file emission logic that had previously been duplicated across every target backend. Its design goals are:

1. **Target-independent abstraction**: `MCStreamer` presents a uniform API for emitting labels, instructions, directives, and data, independent of the output format.
2. **Two output paths from one API**: the same `MCStreamer` API drives both text assembly (`.s` files) and binary object files (`.o` files), controlled by which `MCStreamer` implementation is instantiated.
3. **Support for integrated assembler**: by processing assembly directives through the MC layer, LLVM can assemble inline assembly and `.s` files without invoking an external assembler.

```
MachineInstr
     |
  AsmPrinter
     |
  EmitInstruction()
     |
  MCInst
     |
  MCStreamer (abstract)
    /        \
MCAsmStreamer   MCObjectStreamer
   (text)         (binary)
     |                |
  .s file      MCAssembler
                    |
               MCObjectWriter
                    |
               ELF/COFF/MachO
```

## 94.2 MCInst

`MCInst` (`llvm/include/llvm/MC/MCInst.h`) is the machine-code-level instruction representation. It is deliberately stripped of all high-level information that `MachineInstr` carries:

```cpp
class MCInst {
  unsigned Opcode;                    // same opcode numbers as MachineInstr
  SmallVector<MCOperand, 8> Operands;
  SMLoc Loc;                          // source location (for diagnostics)
};
```

`MCOperand` is a union of three kinds:
- **Register** (`isReg()`): a physical register number (no virtual registers)
- **Immediate** (`isImm()`): a 64-bit integer
- **Expression** (`isExpr()`): an `MCExpr` — a symbolic expression (sum of symbols and constants) used for labels, relocations, and fixups

```cpp
// MCOperand examples:
MCOperand RegOp = MCOperand::createReg(AArch64::X0);
MCOperand ImmOp = MCOperand::createImm(42);
MCOperand ExprOp = MCOperand::createExpr(
  MCSymbolRefExpr::create(Sym, MCSymbolRefExpr::VK_None, Ctx));
```

The critical difference between `MachineInstr` and `MCInst` is the complete absence of virtual registers, register classes, live interval information, debug metadata, memory operands, and all other CodeGen-specific information. `MCInst` is a pure instruction encoding helper.

### 94.2.1 MachineInstr to MCInst Conversion

`AsmPrinter::EmitInstruction()` calls `MCInstLowering` (a per-target helper) to convert a `MachineInstr` to an `MCInst`:

```cpp
// AArch64AsmPrinter.cpp (conceptual):
void AArch64AsmPrinter::EmitInstruction(const MachineInstr *MI) {
  MCInst TmpInst;
  LowerAArch64MachineInstrToMCInst(MI, TmpInst, *this);
  EmitToStreamer(*OutStreamer, TmpInst);
}
```

The lowering step handles:
- Converting `MO_GlobalAddress` and `MO_ExternalSymbol` operands to `MCExpr` symbol references
- Converting `MO_FrameIndex` operands (which should have been eliminated by this point — if any remain, this is a bug)
- Converting calling convention / ABI register numbers to MCInst register numbers

## 94.3 MCStreamer

`MCStreamer` (`llvm/include/llvm/MC/MCStreamer.h`) is the abstract interface for all output generation. Its API mirrors the structure of an assembly file:

```cpp
class MCStreamer {
public:
  // Section management:
  virtual void SwitchSection(MCSection *Section, const MCExpr *Subsection=nullptr);

  // Symbol emission:
  virtual void emitLabel(MCSymbol *Symbol, SMLoc Loc = SMLoc());
  virtual void emitSymbolAttribute(MCSymbol *Symbol, MCSymbolAttr Attribute);

  // Data emission:
  virtual void emitBytes(StringRef Data);
  virtual void emitValueToAlignment(Align Alignment, int64_t Value=0, ...);
  virtual void emitIntValue(uint64_t Value, unsigned Size);

  // Instruction emission (the key method):
  virtual void emitInstruction(const MCInst &Inst, const MCSubtargetInfo &STI);

  // CFI directives:
  virtual void emitCFIStartProc(bool IsSimple, SMLoc Loc);
  virtual void emitCFIDefCfaOffset(int64_t Offset, SMLoc Loc);
  virtual void emitCFIEndProc(SMLoc Loc);

  // Debug info:
  virtual void emitDwarfLocDirective(unsigned FileNo, unsigned Line, ...);
};
```

### 94.3.1 MCAsmStreamer

`MCAsmStreamer` (`llvm/lib/MC/MCAsmStreamer.cpp`) implements `MCStreamer` by printing textual assembly. When `emitInstruction` is called, it calls the target's `MCInstPrinter::printInst()` to render the `MCInst` as a human-readable assembly string:

```cpp
void MCAsmStreamer::emitInstruction(const MCInst &Inst,
                                    const MCSubtargetInfo &STI) {
  SmallString<256> Buf;
  raw_svector_ostream OS(Buf);
  InstPrinter->printInst(&Inst, 0, "", STI, OS);
  // Print the string to the output stream:
  OS_out << "\t" << Buf << "\n";
}
```

`MCInstPrinter` is generated by TableGen from the target's `AsmString` in instruction definitions.

### 94.3.2 MCObjectStreamer

`MCObjectStreamer` (`llvm/lib/MC/MCObjectStreamer.cpp`) drives object file generation. Rather than emitting text, it buffers instructions and data into `MCSection` fragments, then invokes `MCAssembler` to resolve symbols and write the final binary:

```cpp
void MCObjectStreamer::emitInstruction(const MCInst &Inst,
                                       const MCSubtargetInfo &STI) {
  // Encode the instruction into the current code fragment:
  MCCodeEmitter &Emitter = getAssembler().getEmitter();
  SmallVector<MCFixup, 4> Fixups;
  SmallString<8> Code;
  Emitter.encodeInstruction(Inst, Code, Fixups, STI);
  // Append encoded bytes + fixups to the current fragment
  insert(new MCEncodedFragment(Code, Fixups));
}
```

## 94.4 MCCodeEmitter

`MCCodeEmitter` (`llvm/include/llvm/MC/MCCodeEmitter.h`) encodes an `MCInst` into a byte sequence. It is generated largely by TableGen from the instruction's `Inst` bit pattern declaration:

```cpp
// Generated by TableGen for each target (e.g., AArch64MCCodeEmitter):
void AArch64MCCodeEmitter::encodeInstruction(
    const MCInst &MI, SmallVectorImpl<char> &CB,
    SmallVectorImpl<MCFixup> &Fixups,
    const MCSubtargetInfo &STI) const {
  uint32_t Bits = getBinaryCodeForInstr(MI, Fixups, STI);
  support::endian::write<uint32_t>(CB, Bits, support::little);
}
```

Operands that cannot be encoded at assembly time (because they reference symbols whose addresses are not yet known) produce `MCFixup` objects instead of concrete bits. The fixup records the location, offset within the instruction, and the fixup kind (which encodes the relocation type).

## 94.5 Fixups and Relocations

`MCFixup` (`llvm/include/llvm/MC/MCFixup.h`) represents a location in the emitted byte stream that needs to be patched at link time:

```cpp
struct MCFixup {
  uint32_t           Offset;  // byte offset within the fragment
  const MCExpr      *Value;   // the expression to evaluate
  MCFixupKind        Kind;    // how to apply the fixup
  SMLoc              Loc;
};
```

`MCFixupKind` is a target-defined enum identifying the relocation type. For ELF targets, each fixup kind corresponds to an ELF relocation type (e.g., `R_X86_64_PC32`, `R_AARCH64_CALL26`).

### 94.5.1 Fixup to Relocation Conversion

`MCAssembler` processes fixups after the layout phase (once all section sizes are known). For fixups whose value can be resolved at assembly time (e.g., a forward reference within the same section), the assembler applies the fixup directly. For fixups requiring link-time resolution, it creates an `ELF::Elf64_Rela` relocation entry:

```cpp
// MCELFObjectTargetWriter.cpp: recordRelocation()
void ELFObjectWriter::recordRelocation(MCAssembler &Asm,
                                        const MCAsmLayout &Layout,
                                        const MCFragment *Fragment,
                                        const MCFixup &Fixup,
                                        MCValue Target,
                                        uint64_t &FixedValue) {
  unsigned Type = TargetObjectWriter->getRelocType(Ctx, Target, Fixup, ...);
  ELFRelocationEntry Entry(FixedOffset, Sym, Type, Addend);
  Relocations[Section].push_back(Entry);
}
```

The mapping from `MCFixupKind` to ELF relocation type is implemented in target-specific `MCELFObjectTargetWriter` subclasses (e.g., `X86ELFObjectWriter`, `AArch64ELFObjectWriter`).

## 94.6 MCAssembler and MCObjectWriter

`MCAssembler` (`llvm/include/llvm/MC/MCAssembler.h`) orchestrates the assembly of a complete object file:

1. **Layout**: compute the size and offset of every section and fragment, resolving expressions as they become computable.
2. **Relaxation**: if a branch instruction was encoded with a short displacement that is now too small (because sections grew), relax it to a wider form and iterate.
3. **Fixup application**: for fixups resolved within the file, apply them; for external fixups, generate relocation entries.
4. **Writing**: invoke `MCObjectWriter` to produce the final binary.

```cpp
// MCAssembler.cpp: Finish()
bool MCAssembler::Finish(raw_pwrite_stream &OS, MCObjectWriter &OW) {
  MCAsmLayout Layout(*this);
  bool Changed;
  do {
    Changed = layoutOnce(Layout);
  } while (Changed); // iterate until fixpoint (relaxation)
  OW.writeObject(*this, Layout);
  return false;
}
```

`MCObjectWriter` has three implementations: `ELFObjectWriter` (`llvm/lib/MC/ELFObjectWriter.cpp`), `WinCOFFObjectWriter` (`llvm/lib/MC/WinCOFFObjectWriter.cpp`), and `MachObjectWriter` (`llvm/lib/MC/MachObjectWriter.cpp`).

## 94.7 MCDisassembler

`MCDisassembler` (`llvm/include/llvm/MC/MCDisassembler/MCDisassembler.h`) is the inverse pipeline: it converts a byte sequence to `MCInst`s. It is used by `llvm-objdump`, GDB's LLVM disassembler plugin, and the JIT for disassembling generated code.

```cpp
// MCDisassembler interface:
class MCDisassembler {
public:
  enum DecodeStatus { Fail, SoftFail, Success };

  virtual DecodeStatus getInstruction(MCInst &Instr, uint64_t &Size,
                                       ArrayRef<uint8_t> Bytes,
                                       uint64_t Address,
                                       raw_ostream &CStream) const = 0;
};
```

The disassembler for each target is generated by TableGen from the instruction encoding patterns. The generated `decodeInstruction()` function uses a cascade of decoder tables to identify the opcode and decode each field.

```bash
# Disassemble an ELF object file:
llvm-objdump -d --mattr=+avx2 input.o

# Disassemble raw bytes (useful for fuzzing/testing):
echo "48 89 c7" | xxd -r -p | \
  llvm-objdump -d --triple=x86_64 -
```

## 94.8 The .mir Text Format Revisited

The `.mir` format was introduced in [Chapter 88 — The Machine IR](ch88-the-machine-ir.md) from a data structure perspective. This section covers the format from the perspective of writing regression tests.

### 94.8.1 Full .mir File Structure

A `.mir` file is a YAML stream. The optional preamble before the first `---` marker is LLVM IR (used to provide type and global variable declarations):

```llvm
; A minimal .mir test file for x86-64
; The IR preamble defines the function signature:
define i32 @add(i32 %a, i32 %b) { ret i32 0 }

---
name:            add
alignment:       16
tracksRegLiveness: true
liveins:
  - { reg: '$edi', virtual-reg: '%0' }
  - { reg: '$esi', virtual-reg: '%1' }
frameInfo:
  maxAlignment:  1
  hasCalls:      false
constants:       []
body:             |
  bb.0.entry:
    liveins: $edi, $esi

    %0:gr32 = COPY $edi
    %1:gr32 = COPY $esi
    %2:gr32 = ADD32rr %0, killed %1, implicit-def dead $eflags
    $eax = COPY %2
    RET64 implicit $eax
...
```

The `tracksRegLiveness: true` flag tells the parser that liveness information in the file is accurate and should be trusted by subsequent passes.

### 94.8.2 YAML Header Fields

Key fields in the machine function YAML header:

| Field | Type | Meaning |
|---|---|---|
| `name` | string | Function name (must match IR preamble) |
| `alignment` | int | Code alignment in bytes |
| `tracksRegLiveness` | bool | Whether liveness is explicitly tracked |
| `hasWinCFI` | bool | Function has Windows CFI unwind info |
| `frameInfo` | object | `MachineFrameInfo` state |
| `constants` | list | `MachineConstantPool` entries |
| `stack` | list | Frame index objects (type, size, alignment) |
| `body` | string | MIR body (multiline YAML literal block) |

The `stack` list allows tests to specify frame objects that would normally be created by register allocation spilling:

```yaml
stack:
  - { id: 0, type: spill-slot, offset: -8, size: 8, alignment: 8,
      callee-saved-register: '$rbx' }
```

### 94.8.3 Using llc --stop-after and --start-after

```bash
# Find all pass names available as stop points:
llc -mtriple=x86_64-pc-linux-gnu --print-pipeline-passes \
    -o /dev/null /dev/null 2>&1

# Capture state after instruction selection:
llc -mtriple=x86_64-pc-linux-gnu \
    --stop-after=instruction-select \
    -o /tmp/after_isel.mir input.ll

# Modify the .mir file, then continue from after isel:
llc -mtriple=x86_64-pc-linux-gnu \
    --start-after=instruction-select \
    -o output.s /tmp/after_isel.mir

# Run a single pass in isolation:
llc -mtriple=x86_64-pc-linux-gnu \
    --run-pass=branch-folder \
    -o /tmp/after_branch_fold.mir /tmp/after_ra.mir
```

## 94.9 lit Test Infrastructure

LLVM's regression tests use `lit` (LLVM Integrated Tester) together with `FileCheck` to verify backend pass output. Backend tests for the machine layer are located in `llvm/test/CodeGen/<Target>/`.

### 94.9.1 RUN Line Conventions

A typical backend `.mir` test:

```llvm
# NOTE: Assertions have been autogenerated by utils/update_mir_test_checks.py
# RUN: llc -mtriple=aarch64-unknown-linux-gnu \
# RUN:     --run-pass=machine-scheduler \
# RUN:     -o - %s | FileCheck %s

---
name: sched_test
tracksRegLiveness: true
body: |
  bb.0:
    liveins: $w0, $w1
    %0:gpr32 = COPY $w0
    %1:gpr32 = COPY $w1
    %2:gpr32 = ADDWrr %0, %1
    $w0 = COPY %2
    RET_ReallyLR implicit $w0

# CHECK-LABEL: name: sched_test
# CHECK:       ADDWrr
# CHECK-NEXT:  $w0 = COPY
```

The `--run-pass=machine-scheduler` flag runs only the machine scheduler pass and outputs the result as `.mir`. `FileCheck` then verifies the expected instruction order.

### 94.9.2 update_mir_test_checks.py

Rather than writing `FileCheck` patterns by hand, LLVM provides `utils/update_mir_test_checks.py` to automatically regenerate the `# CHECK:` lines from the current pass output:

```bash
python3 llvm/utils/update_mir_test_checks.py \
    --llc-binary /usr/lib/llvm-22/bin/llc \
    llvm/test/CodeGen/AArch64/my_new_test.mir
```

This script runs the test's `RUN:` commands, captures the output, and rewrites the `CHECK:` lines. When adding a new test or modifying an existing pass, run this script to regenerate the expected output rather than writing patterns manually.

### 94.9.3 IR-Level Backend Tests

Not all backend tests use `.mir` format. Many tests start from LLVM IR and use `llc` to produce assembly text, then use `FileCheck` on the assembly output:

```llvm
; RUN: llc -mtriple=x86_64-unknown-linux-gnu -O2 -o - %s | FileCheck %s

define i32 @add_const(i32 %x) {
  %r = add i32 %x, 42
  ret i32 %r
}

; CHECK-LABEL: add_const:
; CHECK:       addl $42, %edi
; CHECK-NEXT:  movl %edi, %eax
; CHECK-NEXT:  retq
```

IR-level tests are easier to write but harder to pin to a specific pass. `.mir` tests are preferred when testing a specific pass in isolation.

### 94.9.4 MC-Level Tests

The MC layer itself has a separate test directory: `llvm/test/MC/<Target>/`. These tests exercise the assembler, disassembler, and object file writer directly using `llvm-mc`:

```bash
# Test assembly:
# RUN: llvm-mc -triple=x86_64-unknown-linux-gnu -filetype=asm %s | FileCheck %s

# Test disassembly:
# RUN: llvm-mc -triple=aarch64-unknown-linux-gnu \
# RUN:         --disassemble %s | FileCheck %s

# Test object file output:
# RUN: llvm-mc -triple=x86_64-unknown-linux-gnu -filetype=obj %s | \
# RUN:   llvm-objdump -d - | FileCheck %s
```

The `llvm-mc` tool is the standalone MC layer driver; it can assemble `.s` files to objects, disassemble hex bytes, and dump object file structure.

## Chapter Summary

- The MC layer (`MCInst`, `MCStreamer`, `MCAssembler`, `MCObjectWriter`) bridges register-allocated `MachineInstr`s and concrete object file output; the same API drives both text assembly and binary object generation.
- `MCInst` is a stripped-down instruction with only physical registers, immediates, and symbol expressions; `AsmPrinter` converts `MachineInstr` to `MCInst` via target-specific `MCInstLowering`.
- `MCStreamer` is the abstract output interface: `MCAsmStreamer` writes text assembly, `MCObjectStreamer` drives binary object generation.
- `MCCodeEmitter` encodes `MCInst` to bytes using TableGen-generated decoder tables; unresolvable operands produce `MCFixup` objects that become ELF/COFF/MachO relocation entries.
- `MCAssembler` lays out sections, relaxes short-form encodings, applies resolvable fixups, and invokes `MCObjectWriter` to produce the final binary.
- `MCDisassembler` inverts `MCCodeEmitter`: it decodes bytes to `MCInst` and is used by `llvm-objdump`, debuggers, and the JIT.
- The `.mir` text format enables pass-level regression testing: `llc --stop-after=<pass>` captures MIR, `--run-pass=<pass>` tests a single pass, and `--start-after=<pass>` resumes from a captured state.
- `lit` + `FileCheck` is the standard LLVM test harness for backend tests; `update_mir_test_checks.py` automates `CHECK:` line generation; MC tests use `llvm-mc` directly.
