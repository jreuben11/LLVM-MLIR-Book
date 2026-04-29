# Chapter 88 — The Machine IR

*Part XIV — The Backend*

Between SelectionDAG lowering and final assembly emission lies the Machine IR (MIR) — LLVM's target-aware intermediate representation that carries instructions from isel through register allocation, scheduling, and all post-isel transformations to the MC layer. Where LLVM IR is intentionally target-agnostic, Machine IR is intimately aware of register classes, calling conventions, stack frames, and instruction encoding details. Understanding MIR in depth is essential for writing backend passes, debugging codegen failures, and reading the output of `llc --stop-after`. This chapter dissects the MIR data structures, the textual `.mir` format used in regression testing, and the MachineVerifier invariants that keep the representation consistent.

## 88.1 MachineFunction

`MachineFunction` is the root container for a function's machine-level representation. It owns all subsidiary structures and is created by `MachineModuleInfo` during the codegen pipeline. The key data structures it aggregates:

```cpp
// llvm/include/llvm/CodeGen/MachineFunction.h
class MachineFunction {
  const Function &F;               // The IR Function
  const TargetMachine &Target;
  const TargetSubtargetInfo *STI;

  MachineRegisterInfo  *RegInfo;       // virtual register table
  MachineFrameInfo      FrameInfo;     // stack frame layout
  MachineConstantPool  *ConstantPool;  // target-level constants
  MachineJumpTableInfo *JumpTableInfo; // jump table data

  // Basic block list (owns all MBBs):
  ilist<MachineBasicBlock> BasicBlocks;

  // Per-function properties (updated as passes run):
  MachineFunctionProperties Properties;

  // Unique counter for new virtual registers, instructions:
  unsigned VRegs;
  ...
};
```

The `MachineFunctionProperties` bitmask records which invariants currently hold: whether SSA form is maintained, whether register allocation is complete, whether all frame indices have been eliminated. Pass managers query these properties to enforce ordering constraints.

### 88.1.1 MachineFrameInfo

`MachineFrameInfo` models the abstract stack frame before any concrete SP-relative offsets are assigned. Each stack object is identified by a non-negative integer index:

```cpp
// llvm/include/llvm/CodeGen/MachineFrameInfo.h
class MachineFrameInfo {
  // Fixed objects (negative indices) represent function arguments
  // at fixed positions on the incoming stack:
  SmallVector<StackObject> Objects;  // [fixed objects] [spill/local objects]

  uint64_t StackSize;          // set by PrologueEpilogueInserter
  Align    MaxAlign;           // maximum alignment required
  bool     HasCalls;           // does the function call anything?
  bool     FrameAddressTaken;  // @llvm.frameaddress used?
  bool     ReturnAddressTaken; // @llvm.returnaddress used?
  ...
};
```

Frame index 0 and above are local objects (spill slots, `alloca`s). Negative indices represent fixed incoming argument slots in the caller's frame. During `PrologueEpilogueInserter`, each abstract frame index is replaced with an SP/FP-relative `MachineMemOperand` or direct register+offset addressing.

### 88.1.2 MachineRegisterInfo

`MachineRegisterInfo` (MRI) is the virtual register table. Before register allocation every `def` or `use` of a virtual register is recorded here:

```cpp
// llvm/include/llvm/CodeGen/MachineRegisterInfo.h
class MachineRegisterInfo {
  // Per-virtual-register info:
  struct VRegInfo {
    const TargetRegisterClass *RC;   // or nullptr if RegBank-allocated
    const RegisterBank        *RB;
    MachineOperand            *DefListHead;
    MachineOperand            *UseListHead;
  };
  IndexedMap<VRegInfo> VRegInfo;  // vreg -> VRegInfo

  // Physical register use/def tracking:
  std::vector<MachineOperand*> PhysRegUseDefLists[TRI::FirstVirtualRegister];

  bool IsSSA;    // true until PHI elimination
  ...
};
```

The def-use chains are maintained as intrusive linked lists through `MachineOperand::NextOperand`. This gives O(1) iteration over all uses of a virtual register — critical for RegisterCoalescer and live interval computation. The `MachineRegisterInfo::use_operands()` and `def_operands()` iterators traverse these chains.

### 88.1.3 MachineConstantPool

`MachineConstantPool` holds constants that the target needs to materialise from a `.rodata`-like section (e.g., floating-point immediates on x86 that must be loaded from memory, or SIMD shuffle masks). Each entry is uniqued by value and alignment:

```cpp
// llvm/include/llvm/CodeGen/MachineConstantPool.h
struct MachineConstantPoolEntry {
  union {
    const Constant *ConstVal; // LLVM IR constant
    MachineConstantPoolValue *MachineCPVal; // target-specific
  };
  Align Alignment;
  bool  isMachineConstantPoolEntry() const;
};

class MachineConstantPool {
  SmallVector<MachineConstantPoolEntry, 4> Constants;
  unsigned getConstantPoolIndex(const Constant *C, Align A);
  ...
};
```

Target-specific subclasses of `MachineConstantPoolValue` (e.g., `ARMConstantPoolValue`) allow the target to store PC-relative symbol references or stub addresses in the constant pool, enabling PIC code generation on platforms that require it.

## 88.2 MachineBasicBlock

`MachineBasicBlock` (MBB) is the unit of linear control flow within a `MachineFunction`. Each MBB corresponds to a basic block in the CFG and holds an intrusive doubly-linked list of `MachineInstr`s.

```cpp
// llvm/include/llvm/CodeGen/MachineBasicBlock.h
class MachineBasicBlock : public ilist_node_with_parent<...> {
  const BasicBlock *BB;           // original IR block (may be null)
  MachineFunction  *Parent;
  ilist<MachineInstr> Insts;

  // CFG edges:
  SmallVector<MachineBasicBlock *, 4> Predecessors;
  SmallVector<MachineBasicBlock *, 4> Successors;

  // Edge probabilities (BranchProbability, summing to 1 across succs):
  SmallVector<BranchProbability, 4> Probs;

  // Liveness at entry (physical registers):
  LivePhysRegs LiveIns;
  // Alternatively, pre-RA: the set of virtual reg livein pairs
  std::vector<RegisterMaskPair> LiveInRegs;

  // Block properties:
  bool IsEHPad;         // exception landing pad
  bool IsEHFuncletEntry;
  bool Alignment;       // desired alignment for the block label
  bool IsBeginSection;  // start of a code section
  ...
};
```

### 88.2.1 Predecessor/Successor Lists

The CFG is represented explicitly in MBB: each block maintains ordered predecessor and successor lists. Passes that modify control flow must call `addSuccessor`/`removeSuccessor` to keep this representation consistent. Branch probability is tracked per-successor as a `BranchProbability` rational in [0,1].

### 88.2.2 Liveness Sets

Before register allocation, `LiveInRegs` records `(virtual_register, lane_mask)` pairs indicating which virtual registers are live at block entry. After register allocation, the field switches to `LiveIns` containing physical register numbers. The `MachineLiveness` analysis and the `LiveIntervals` pass populate these sets; they are consumed by the register allocator and post-RA scheduling.

## 88.3 MachineInstr

`MachineInstr` (MI) is the fundamental instruction node. It is always heap-allocated and owned by its parent MBB's instruction list.

```cpp
// llvm/include/llvm/CodeGen/MachineInstr.h
class MachineInstr : public ilist_node_with_parent<...> {
  const MCInstrDesc &MCID;   // opcode + static descriptor
  MachineBasicBlock *Parent;

  // Operand vector (first NumDefs entries are defs, rest are uses):
  SmallVector<MachineOperand, 8> Operands;

  // Memory operand list (for loads/stores):
  ArrayRef<MachineMemOperand *> MemRefs;

  // Source location:
  DebugLoc DbgLoc;

  // Instruction flags (packed into a uint32_t):
  // isKill (on use operand), isDead (on def operand),
  // isImplicit, isUndef, isInternalRead, isEarlyClobber, etc.
  ...
};
```

### 88.3.1 Opcode and MCInstrDesc

The opcode is an integer index into the target's generated `*InstrInfo.inc` table. The `MCInstrDesc` (accessible via `MI.getDesc()`) records:

- `NumDefs` and `NumOperands` (statically known from TableGen)
- `Flags` bitmask: `isReturn`, `isBranch`, `isCall`, `mayLoad`, `mayStore`, `isCommutable`, etc.
- Per-operand `MCOperandInfo`: operand kind, tied-to index, register class

```cpp
// Iterate operands:
for (MachineOperand &MO : MI.operands()) {
  if (MO.isReg())
    dbgs() << printReg(MO.getReg(), TRI);
  else if (MO.isImm())
    dbgs() << MO.getImm();
}

// Access the descriptor:
const MCInstrDesc &Desc = MI.getDesc();
bool IsLoad  = Desc.mayLoad();
bool IsStore = Desc.mayStore();
unsigned NumDefs = Desc.getNumDefs();
```

### 88.3.2 Instruction Flags

Key per-operand and per-instruction flags:

| Flag | Location | Meaning |
|---|---|---|
| `isKill` | Use operand | Last use of this virtual/physical reg in this range |
| `isDead` | Def operand | Defined value is never used |
| `isImplicit` | Any operand | Not in the instruction encoding; side-effect |
| `isUndef` | Use operand | Value is undefined; reg allocator may pick any phys reg |
| `isEarlyClobber` | Def operand | Written before inputs are read (inline asm) |
| `isDebug` | MI | Debug value instruction; not emitted as real code |
| `isInternalRead` | Use operand | Read of a bundle-internal virtual register |

The `DebugLoc` attaches source location metadata (file, line, column, inline call stack) for debug info emission. It is a reference-counted pointer to a `DILocation` node.

## 88.4 MachineOperand Types

`MachineOperand` is a discriminated union covering every kind of operand that can appear in a machine instruction:

```cpp
// llvm/include/llvm/CodeGen/MachineOperand.h
class MachineOperand {
  unsigned OpKind : 8;  // MachineOperandType enum

  // Payload union:
  union {
    struct { unsigned RegNo; /* ... */ } Reg;
    int64_t ImmVal;
    const ConstantFP *CFP;
    int Index;  // for FI, CPI, JTI
    const GlobalValue *GV;
    const char *SymName;
    const BlockAddress *BA;
    MachineBasicBlock *MBB;
    MCSymbol *Sym;
    ...
  };
  ...
};
```

The operand types, with their accessor methods:

| Kind | Accessor | Usage |
|---|---|---|
| `MO_Register` | `getReg()` / `setReg()` | Virtual or physical register |
| `MO_Immediate` | `getImm()` | 64-bit integer immediate |
| `MO_FPImmediate` | `getFPImm()` | Floating-point constant |
| `MO_MachineBasicBlock` | `getMBB()` | Jump target |
| `MO_FrameIndex` | `getIndex()` | Abstract stack slot |
| `MO_ConstantPoolIndex` | `getIndex()` | Entry in MachineConstantPool |
| `MO_JumpTableIndex` | `getIndex()` | Jump table |
| `MO_GlobalAddress` | `getGlobal()` | Reference to a GlobalValue |
| `MO_ExternalSymbol` | `getSymbolName()` | Library or runtime symbol |
| `MO_RegisterMask` | `getRegMask()` | Bitmask of clobbered phys regs |
| `MO_Metadata` | `getMetadata()` | Attached MDNode (e.g., tbaa) |
| `MO_MCSymbol` | `getMCSymbol()` | Direct MCSymbol reference |
| `MO_CFIIndex` | `getCFIIndex()` | CFI directive index |

### 88.4.1 Register Operands: Virtual vs Physical

Register operands are identified by a 32-bit register number. Physical registers are encoded in the range `[1, FirstVirtualRegister)` and are pre-defined in the target's `*RegisterInfo.inc`. Virtual registers begin at `Register::FirstVirtualRegister` (a target-defined constant, typically 0x4000_0000).

```cpp
Register Reg = MO.getReg();
if (Register::isVirtualRegister(Reg)) {
  unsigned VRegNo = Register::virtReg2Index(Reg);
  const TargetRegisterClass *RC = MRI.getRegClass(Reg);
} else {
  // Physical register -- use TRI to print
  dbgs() << TRI->getName(Reg);
}
```

Sub-registers are selected by a `SubReg` field on the operand: `MO.getSubReg()` returns an index into `TargetRegisterInfo::getSubRegIndexName()`.

## 88.5 MachineMemOperand

Loads and stores carry a `MachineMemOperand` (MMO) list that records alias analysis information at the machine level:

```cpp
// llvm/include/llvm/CodeGen/MachineMemOperand.h
class MachineMemOperand {
  MachinePointerInfo PtrInfo;  // base + offset
  LLT               MemoryType; // size and type
  Align             BaseAlignment;
  AtomicOrdering    Ordering;

  // Flags:
  bool isVolatile() const;
  bool isNonTemporal() const;
  bool isInvariant() const;
  bool isUnordered() const;

  // AA metadata:
  AAMDNodes AAInfo;  // tbaa, tbaa.struct, noalias, alias.scope
};
```

`MachinePointerInfo` records the abstract base pointer for the access. When the base is a frame index or constant pool entry, it uses a `PseudoSourceValue` rather than an IR `Value`.

### 88.5.1 PseudoSourceValue

`PseudoSourceValue` is an alias-analysis-aware proxy for non-IR memory objects:

```cpp
// llvm/include/llvm/CodeGen/PseudoSourceValue.h
class PseudoSourceValue {
public:
  enum PSVKind {
    Stack,         // local stack slots (may alias each other)
    GOT,           // Global Offset Table
    JumpTable,     // jump table data
    ConstantPool,  // constant pool entries (read-only, non-aliasing)
    FixedStack,    // incoming argument frame slots
    GlobalValueCallEntry,
    ExternalSymbolCallEntry,
    ...
  };
  PSVKind Kind;
  bool isConstant(const MachineFrameInfo *) const;
  bool isAliased(const MachineFrameInfo *) const;
};
```

The AliasAnalysis machinery (`MachineMemOperand::isAliasedWith()`) uses `PseudoSourceValue::isAliased()` and `isConstant()` to reason that loads from `ConstantPool` cannot alias stores to `Stack`, enabling the scheduler and load-store optimizer to reorder memory operations around constant pool loads.

## 88.6 The MIR Text Format

LLVM provides a textual serialization of `MachineFunction` as a YAML document with an embedded machine basic block body. This format is used for regression testing the backend — you can capture the state after any pass with `llc --stop-after=<pass-name>` and inspect or hand-edit the result.

### 88.6.1 File Structure

A `.mir` file contains an optional LLVM IR module followed by one YAML document per `MachineFunction`:

```yaml
# Example .mir fragment (x86-64 add function)
---
name:            add_func
alignment:       16
tracksRegLiveness: true
frameInfo:
  maxAlignment:  1
  hasCalls:      false
constants:       []
body:             |
  bb.0.entry:
    liveins: $edi, $esi

    %0:gr32 = COPY $edi
    %1:gr32 = COPY $esi
    %2:gr32 = ADD32rr %0, %1, implicit-def $eflags
    $eax = COPY %2
    RET64 implicit $eax
...
```

Key sections:
- `frameInfo`: stack frame properties
- `constants`: constant pool entries (index, value, alignment)
- `body`: the basic block sequence, in indented form

### 88.6.2 MBB Syntax

Machine basic blocks in the `.mir` body use labels `bb.N.name:` (where N is the MBB number and name is the optional IR block name), followed by optional `liveins:` and `successors:` annotations:

```yaml
  bb.1.loop:
    successors: %bb.1(0x7fffffff), %bb.2(0x00000001)
    liveins: $edi, $xmm0

    %5:gr64 = PHI %4, %bb.0, %7, %bb.1
    ...
    JMP_1 %bb.1
```

The `successors:` line lists branch targets with their `BranchProbability` in parentheses (as a 32-bit fixed-point fraction with denominator 2^32 − 1).

### 88.6.3 MachineInstr Syntax

Each instruction line begins with optional tied defs and uses:

```
[%reg:RC =] OPCODE [flags] [%reg:RC[, subref][killed]], [imm], ...
```

Operand flags appear as keywords: `killed`, `dead`, `implicit`, `implicit-def`, `undef`, `early-clobber`. Frame index operands are written `%stack.N`, constant pool entries as `%const.N`:

```yaml
    %0:gr64 = MOV64rm %rip, 1, $noreg, @global, $noreg :: (load (s64) from got)
    MOV32mr %stack.0, 1, $noreg, 0, $noreg, %1:gr32 :: (store (s32) into %stack.0)
```

The `:: (...)` suffix is the `MachineMemOperand` annotation: access kind (`load`/`store`), type, and source.

### 88.6.4 Producing and Consuming .mir Files

```bash
# Stop after isel (SelectionDAGISel pass):
llc -mtriple=x86_64-unknown-linux-gnu \
    --stop-after=instruction-select \
    -o /tmp/after_isel.mir input.ll

# Stop after register allocation:
llc -mtriple=x86_64-unknown-linux-gnu \
    --stop-after=greedy \
    -o /tmp/after_ra.mir input.ll

# Resume from a specific pass:
llc -mtriple=x86_64-unknown-linux-gnu \
    --start-after=greedy \
    -o output.s after_ra.mir

# List all codegen passes (to find stop-after names):
llc --print-pipeline-passes -o /dev/null input.ll
```

### 88.6.5 Writing .mir-Based Tests

The LLVM regression test suite uses `.mir` files with `RUN:` lines invoking `llc` and `FileCheck`:

```llvm
# RUN: llc -mtriple=x86_64-unknown-linux-gnu \
# RUN:     --run-pass=post-RA-sched \
# RUN:     -o - %s | FileCheck %s

---
name: foo
body: |
  bb.0:
    liveins: $edi
    %0:gr32 = COPY $edi
    $eax = COPY %0
    RET64 implicit $eax
...

# CHECK: RET64
# CHECK-NOT: COPY
```

The `--run-pass=<name>` flag runs a single pass on the input `.mir` file, making it easy to unit-test individual backend passes in isolation.

## 88.7 MachineVerifier

`MachineVerifier` (`llvm/lib/CodeGen/MachineVerifier.cpp`) is the integrity checker for MIR, analogous to `verifyModule` for LLVM IR. It checks a comprehensive set of structural invariants and is invoked via `-verify-machineinstrs` or automatically by debug builds.

### 88.7.1 Invariants Checked

**Register class consistency**: Every virtual register use must satisfy the register class (or register bank) constraint recorded in MRI. A `gr32` virtual register cannot be used in an instruction slot expecting `gr64`.

**Def-use chain integrity**: Every use of a virtual register must have exactly one reaching definition (SSA property), and the def-use chain linked list must be consistent.

**Operand count and type**: The number and kinds of operands must match the `MCInstrDesc`'s `NumOperands` and per-operand type constraints.

**Kill/dead flags**: A kill flag must be placed on the last use, and a dead flag on a def with no uses. The verifier recomputes live sets and compares them.

**CFG consistency**: The `Predecessors`/`Successors` lists must be consistent (if A lists B as a successor, B must list A as a predecessor). `BranchProbability` sums must equal 1 across all successors of a block.

**Frame index validity**: All `MO_FrameIndex` operands must reference indices in the range `[minFixedObject, numObjects)`.

**Liveness after register allocation**: After `MachineFunctionProperties::Property::NoPHIs` is set, no `PHI` instructions may exist. After `Property::RegBankSelected`, every virtual register must have an assigned register bank.

### 88.7.2 Invoking the Verifier

```bash
# Run with machine instruction verification:
llc -mtriple=x86_64-unknown-linux-gnu \
    -verify-machineinstrs \
    -o output.s input.ll

# Run only the verifier on a .mir file:
llc -mtriple=x86_64-unknown-linux-gnu \
    --run-pass=verify \
    -o /dev/null input.mir
```

In C++ tests, the verifier can be called directly:

```cpp
MachineVerifier Verifier(nullptr, nullptr);
bool Failed = Verifier.verify(*MF);
```

Pass implementers should call `MF->verify(this, "after my pass")` at the end of any pass that modifies MIR to catch regressions early. The verifier prints the first failing invariant together with the offending MachineInstr, making it a first-line debugging tool.

## Chapter Summary

- `MachineFunction` is the root container, aggregating `MachineRegisterInfo`, `MachineFrameInfo`, `MachineConstantPool`, and the `MachineBasicBlock` list.
- `MachineFrameInfo` models the abstract stack layout using integer frame indices, later concretized by `PrologueEpilogueInserter`.
- `MachineRegisterInfo` maintains def-use chains for all virtual registers as intrusive linked lists, enabling O(1) use-iteration.
- `MachineBasicBlock` holds the CFG edges and per-block liveness information; `MachineInstr` holds the opcode, operand list, memory operands, and source location.
- `MachineOperand` is a discriminated union covering registers (virtual/physical), immediates, frame indices, global values, external symbols, jump tables, constant pool indices, and register masks.
- `MachineMemOperand` carries alias analysis metadata; `PseudoSourceValue` abstracts non-IR memory objects (stack slots, constant pool, GOT) for AA.
- The `.mir` textual format enables pass-level regression testing: use `llc --stop-after=<pass>` to capture state and `--run-pass=<pass>` to test a single pass in isolation.
- `MachineVerifier` enforces structural invariants across the MIR lifecycle; enable it with `-verify-machineinstrs` and call `MF->verify()` in custom passes.
