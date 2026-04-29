# Chapter 93 — Post-RegAlloc and Pre-Emit

*Part XIV — The Backend*

After register allocation completes, the MIR still contains several abstractions that must be concretized before assembly emission: frame indices must become SP/FP-relative offsets, branches may be redundant and removable, predicated execution opportunities may exist, and the instruction schedule may be further improved by knowledge of the actual physical register assignments. This chapter covers the sequence of post-RA transformation passes — prologue/epilogue insertion, frame index elimination, branch folding, tail duplication, if-conversion, post-RA scheduling, function splitting for cold code, and Control Flow Guard instrumentation — collectively responsible for bridging register-allocated MIR to the final, emit-ready instruction sequence.

## 93.1 PrologueEpilogueInserter

`PrologueEpilogueInserter` (`llvm/lib/CodeGen/PrologEpilogInserter.cpp`) is the most consequential post-RA pass. It assigns concrete stack offsets to every frame object and emits the function prologue and epilogue.

### 93.1.1 Callee-Saved Register Determination

The pass scans all `MachineInstr`s in the function to determine which callee-saved registers are actually used. The target provides the list of callee-saved registers (`TargetRegisterInfo::getCalleeSavedRegs`) and their save/restore instruction sequences (`TargetFrameLowering::assignCalleeSavedSpillSlots`):

```cpp
// PrologEpilogInserter.cpp: calculateSaveRestoreBlocks()
void PEI::assignCalleeSavedSpillSlots(MachineFunction &MF, ...) {
  const TargetRegisterInfo *TRI = MF.getSubtarget().getRegisterInfo();
  const MCPhysReg *CSRegs = TRI->getCalleeSavedRegs(&MF);

  // Scan all MBBs for uses of callee-saved physical registers:
  for (MachineBasicBlock &MBB : MF)
    for (MachineInstr &MI : MBB)
      for (MachineOperand &MO : MI.operands())
        if (MO.isReg() && MO.isDef() && isCalleeSaved(MO.getReg(), CSRegs))
          UsedCSRegs.set(MO.getReg());
}
```

For each used callee-saved register, a stack slot is allocated in the frame and a save/restore pair (`PUSH`/`POP` on x86; `STP`/`LDP` on AArch64) is inserted in the prologue and epilogue blocks.

### 93.1.2 Stack Frame Layout

Once callee-saved slots are assigned, the pass computes the full stack frame layout. The canonical frame layout (AArch64 example):

```
High address (before call)
  [incoming arguments     ]   (passed on stack if > 8 args)
  [return address / LR    ]   (if spilled)
  [callee-saved registers ]   (FP, LR, then X19-X28)
  [local alloca objects   ]   (aligned to their required alignment)
  [spill slots            ]   (register allocator spills)
  [outgoing argument area ]   (space for arguments passed on stack)
Low address (SP points here after prologue)
```

The pass computes `MachineFrameInfo::StackSize` as the sum of all object sizes rounded up to the frame alignment. Then it updates `StackSize` in `MachineFrameInfo` and sets concrete offsets for every frame object via `MachineFrameInfo::setObjectOffset()`.

### 93.1.3 Prologue Generation

The prologue is inserted at the function entry block. Target-provided hooks generate the actual instructions:

```cpp
// TargetFrameLowering interface:
virtual void emitPrologue(MachineFunction &MF,
                          MachineBasicBlock &MBB) const = 0;
virtual void emitEpilogue(MachineFunction &MF,
                          MachineBasicBlock &MBB) const = 0;
```

AArch64 prologue example for a function using X19-X20 and a 32-byte local:

```
; Generated prologue:
STP X29, X30, [SP, #-48]!   ; save FP and LR, decrement SP by 48
STP X19, X20, [SP, #16]     ; save callee-saved X19, X20
ADD X29, SP, #0             ; set frame pointer
```

### 93.1.4 Frame Index Elimination

After the stack layout is assigned, all `MO_FrameIndex` operands in MachineInstrs are replaced with concrete `base_register + offset` addressing. This is done by `TargetRegisterInfo::eliminateFrameIndex()`:

```cpp
// x86 frame index elimination (simplified):
void X86RegisterInfo::eliminateFrameIndex(MachineBasicBlock::iterator II,
                                           int SPAdj, ...) const {
  MachineInstr &MI = *II;
  int FrameIndex = MI.getOperand(FIOperandNum).getIndex();
  int Offset = MFI.getObjectOffset(FrameIndex) + SPAdj;
  // Replace the FrameIndex operand with [SP + Offset]:
  MI.getOperand(FIOperandNum).ChangeToRegister(StackPtr, false);
  MI.getOperand(FIOperandNum + 3).ChangeToImmediate(Offset);
}
```

## 93.2 BranchFolding

`BranchFolding` (`llvm/lib/CodeGen/BranchFolding.cpp`) cleans up the CFG after instruction selection and register allocation introduce redundant control flow. It performs several transformations:

### 93.2.1 Fall-Through Elimination

When a conditional branch is followed immediately by its false-target block, the fall-through is redundant. `BranchFolding` removes the explicit branch:

```
Before:
  bb.0:
    JNE bb.2
    JMP bb.1       ; redundant: bb.1 is the fall-through

After:
  bb.0:
    JNE bb.2
    ; fall through to bb.1
```

### 93.2.2 Identical Successor Merging

When two successors of a block have identical content (same sequence of instructions with the same outcomes), `BranchFolding` merges them:

```
Before:
  bb.0:
    JE bb.1
    JMP bb.2
  bb.1: (then-branch)
    RET 0
  bb.2: (else-branch)
    RET 0

After: bb.1 and bb.2 merged into one
  bb.0:
    JMP bb.1       ; both branches go to same block
  bb.1:
    RET 0
```

The comparison uses a hash of instruction sequences to quickly identify common tails.

### 93.2.3 Block Tail Merging

Even when entire blocks are not identical, their trailing instruction sequences may be. `BranchFolding` extracts common tails into a shared block and adjusts branches to jump to the shared tail. This is particularly effective when multiple return paths in a function share a common sequence of stack cleanup operations.

## 93.3 TailDuplication

`TailDuplicate` (`llvm/lib/CodeGen/TailDuplication.cpp`) is the inverse of tail merging: it duplicates small blocks at the end of the control flow graph into their predecessors, eliminating unconditional jumps and enabling fall-through execution.

```
Before:
  bb.0: ...  JNE bb.2  ; falls through to bb.1
  bb.1: MOV $eax, 0   JMP bb.3   ; small block
  bb.2: ...  JMP bb.1
  bb.3: RET

After tail duplication of bb.1 into bb.0 and bb.2:
  bb.0: ...  JNE bb.2 ; (then fall-through copy of bb.1)
        MOV $eax, 0   JMP bb.3
  bb.2: ...
        MOV $eax, 0   JMP bb.3  ; duplicated copy
  bb.3: RET
```

Tail duplication is bounded by `TailDupSize` (default: blocks of ≤4 instructions on most targets). Larger blocks are not duplicated because the code size increase would not be worth the branch elimination.

## 93.4 IfConversion

`IfConverter` (`llvm/lib/CodeGen/IfConversion.cpp`) converts conditional control flow patterns to predicated (conditional) execution on architectures that support it (ARM, AArch64, Hexagon, and to a limited extent x86 with CMOVcc).

### 93.4.1 Triangle Pattern

```
Before (triangle — one branch taken, one falls through):
  bb.0:
    CMP r0, #0
    BNE bb.2       ; if r0 != 0 skip bb.1
  bb.1:           ; then-block (small)
    ADD r1, r1, #1
  bb.2:           ; merge block
    ...

After (AArch64 with predication):
  bb.0:
    CMP x0, #0
    CINC x1, x1, EQ  ; conditionally increment r1 if EQ
    ...
```

### 93.4.2 Diamond Pattern

```
Before (diamond — both branches taken):
  bb.0:
    CMP r0, #0
    BNE bb.2
  bb.1 (then):    MOV r1, #1  JMP bb.3
  bb.2 (else):    MOV r1, #2
  bb.3 (merge):   ...

After (if-converted):
  bb.0:
    CMP r0, #0
    MOVN r1, #1     ; conditional move if NE
    MOVEQ r1, #2    ; conditional move if EQ
    ...
```

The profitability check in `IfConverter` uses the target's `TargetInstrInfo::isProfitableToIfCvt()` hook, which considers the number of instructions in the converted blocks, the branch prediction cost, and the latency of the predicated instructions.

## 93.5 Post-RA Scheduling

`PostRAScheduler` (`llvm/lib/CodeGen/PostRAScheduler.cpp`) performs instruction scheduling after register allocation, using the final physical register assignments. Post-RA scheduling can eliminate more hazards than pre-RA scheduling because physical register aliases are now known (e.g., a write to `$eax` is a hazard for a subsequent read of `$rax`).

### 93.5.1 Hazard Detection

The post-RA scheduler uses `TargetHazardRecognizer` to model processor-specific hazards:

```cpp
// TargetHazardRecognizer interface:
virtual HazardType getHazardType(SUnit *SU, int Stalls) = 0;
virtual void EmitInstruction(SUnit *SU) = 0;
virtual void AdvanceCycle() = 0;
```

The default implementation (`ScoreboardHazardRecognizer`) uses a *scoreboard* — a circular buffer of instruction issue records — to track which functional units are busy and which pipeline stages have pending results.

### 93.5.2 Register Anti-Dependence Breaking

Post-RA scheduling can rename physical registers to break anti-dependences (WAR hazards) — but only to the extent that free registers exist. `AggressiveAntiDepBreaker` (`llvm/lib/CodeGen/AggressiveAntiDepBreaker.cpp`) identifies anti-dependence edges in the scheduling DAG and rewrites the interfering instruction to use a different (free) physical register:

```
Before (WAR hazard: r1 read by ADD before r1 written by MOV):
  ADD r0, r1, r2  ; reads r1 (latency 1)
  MOV r1, r3      ; writes r1 -- hazard: must wait 1 cycle

After anti-dep breaking (rename r1 to r4 in MOV):
  ADD r0, r1, r2  ; reads r1
  MOV r4, r3      ; now writes r4 -- no hazard with r1
```

### 93.5.3 Machine Trace Metrics

The post-RA scheduler optionally queries `MachineTraceMetrics` to guide scheduling decisions for superscalar cores. This analysis computes the expected execution time of instruction traces through the function, enabling the scheduler to prioritise instructions that are on the predicted hot execution path.

## 93.6 MachineFunctionSplitter

`MachineFunctionSplitter` (`llvm/lib/CodeGen/MachineFunctionSplitter.cpp`) splits cold basic blocks out of a function and places them in a separate `.text.cold` section. This is the machine-level analogue of the IR-level `HotColdSplitting` pass, and it operates on the final physical instruction layout:

```cpp
// MachineFunctionSplitter.cpp: runOnMachineFunction()
bool MachineFunctionSplitter::runOnMachineFunction(MachineFunction &MF) {
  for (MachineBasicBlock &MBB : MF) {
    if (isColdBlock(MBB, MBFI, PSI)) {
      MBB.setSectionID(MBBSectionID(MBBSectionID::ColdSectionID));
    }
  }
  // Blocks with ColdSectionID are later placed in .text.cold
  return true;
}
```

Cold blocks are identified by block frequency: blocks whose frequency is less than a threshold fraction of the function's entry block frequency (default 1/10000) are considered cold. Splitting cold code out of the hot section improves instruction cache utilisation by increasing the density of frequently-executed code.

The cold split also feeds the linker's `--gc-sections` mechanism: if a cold section's code is never executed, the linker can discard it entirely.

## 93.7 CFGuard

Windows Control Flow Guard (CFG) instrumentation is added to functions by the `CFGuardLongjmp` pass and the `MachineFunction`'s function attribute. CFG/CFGuard requires:

1. **Call targets registered**: every indirect call target (function that is called via a function pointer) must be registered in a special `.cfguard` section.
2. **Check calls**: every indirect call must be preceded by a call to the CFG check stub.

```cpp
// CFGuard pass: instrumenting indirect calls (x64)
// The CFG check: __guard_dispatch_icall_fptr (function pointer)
BuildMI(MBB, MI, MI.getDebugLoc(), TII->get(X86::CALL64m))
  .addReg(X86::RIP).addImm(1).addReg(0)
  .addExternalSymbol("__guard_dispatch_icall_fptr")
  .addReg(0);
```

The function table (registered call targets) is emitted by `AsmPrinter::emitCFIInstruction` into the `.gfids` (Guard Function ID) section. The Windows loader validates this table at process load time; if an indirect call target is not in the table, the OS terminates the process.

`CFGuard` is enabled by the `-cfguard` Clang flag (`-fsanitize=cfi-icall` uses a different, software-only mechanism). The LLVM implementation is in `llvm/lib/Target/X86/X86CFGuard.cpp` and `llvm/lib/CodeGen/CFGuardLongjmp.cpp`.

## Chapter Summary

- `PrologueEpilogueInserter` scans for used callee-saved registers, allocates stack slots, computes the final frame layout, and generates prologue/epilogue code; `eliminateFrameIndex()` replaces abstract frame indices with concrete SP/FP-relative offsets.
- `BranchFolding` removes redundant branches (fall-through, identical successors) and merges common tails to reduce code size and branch pressure.
- `TailDuplicate` duplicates small end-of-function blocks into their predecessors to eliminate unconditional jumps and enable fall-through.
- `IfConverter` transforms triangle and diamond CFG patterns to predicated instructions on architectures that support conditional execution, eliminating branch overhead.
- `PostRAScheduler` reschedules after register allocation with full physical register alias awareness; `AggressiveAntiDepBreaker` renames physical registers to eliminate WAR hazards.
- `MachineFunctionSplitter` moves cold basic blocks to a separate `.text.cold` section, improving hot-path instruction cache density.
- CFGuard instrumentation adds Windows Control Flow Guard checks at every indirect call and registers all valid call targets in a `.cfguard` section consulted by the OS loader.
