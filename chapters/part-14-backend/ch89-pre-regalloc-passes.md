# Chapter 89 — Pre-RegAlloc Passes

*Part XIV — The Backend*

Register allocation demands a specific pre-condition: every instruction must be in two-address form, PHI nodes must be eliminated, and the live ranges of virtual registers must be precisely known. A series of preparatory passes transforms the MIR from the three-address SSA form produced by instruction selection into the form that register allocators expect. This chapter traces each preparatory pass in pipeline order: two-address conversion, PHI elimination, slot index assignment, live interval computation, register coalescing, the machine combiner, and pre-RA scheduling. Together these passes both simplify the allocation problem and improve the quality of the code fed to the allocator.

## 89.1 TwoAddressInstructionPass

The most immediate gap between instruction selection output and register allocator input is instruction address count. SelectionDAG and GlobalISel both produce three-address instructions of the form `dst = op src1, src2` where `dst`, `src1`, and `src2` are independent virtual registers. Most x86 instructions require `dst = src1` (the destination aliases the first source); ARM THUMB-2 has similar constraints for certain instructions; many CISC architectures have at least a subset of two-address instructions.

`TwoAddressInstructionPass` (`llvm/lib/CodeGen/TwoAddressInstructionPass.cpp`) converts three-address instructions to two-address form by inserting a `COPY` before the instruction:

```
Before:
  %2 = ADD32rr %0, %1          ; three-address

After:
  %2 = COPY %0                 ; insert copy: dst <- src1
  %2 = ADD32rr %2(tied), %1   ; now dst == src1 (tied operand 0)
```

The pass queries `MCInstrDesc::getOperandConstraint(i, MCOI::TIED_TO)` to discover which output is tied to which input. If the tied input is already the same virtual register as the output (or can be proven equivalent through a copy chain), no additional copy is needed.

### 89.1.1 Commutativity Optimisation

When the tied-to source is the *second* operand of a commutative instruction, the pass first checks whether swapping operands avoids the copy. For `%2 = ADD %0, %1` where `%0` is live-through (used later) but `%1` is killed here, commuting gives `%2 = ADD %1(killed), %0` — now `%1` is the tied source and its kill flag satisfies the two-address form without inserting a new virtual register copy:

```cpp
// TwoAddressInstructionPass.cpp: tryInstructionCommute()
if (MI->isCommutable()) {
  // Try to swap src1 and src2 to avoid the copy:
  if (TII->commuteInstruction(*MI, /*NewMI=*/false))
    return true;
}
```

### 89.1.2 Sink/Hoist Optimisation

When the copy cannot be eliminated by commuting, the pass tries to move the copy instruction as close as possible to its definition (hoisting) or to eliminate the copy by renaming. The inserted copies feed into the RegisterCoalescer for elimination in a subsequent pass.

## 89.2 PHI Elimination

After `TwoAddressInstructionPass`, the MIR is still in SSA form with `PHI` instructions at join points. PHI instructions are not real instructions — no physical hardware provides a "select predecessor value" operation. `PHIElimination` (`llvm/lib/CodeGen/PHIElimination.cpp`) converts each `PHI` into a set of `COPY` instructions in the predecessor blocks.

```
Before (MIR in SSA):
  bb.0:
    %0:gr32 = MOV32ri 0
    JMP_1 %bb.2
  bb.1:
    %1:gr32 = MOV32ri 1
    JMP_1 %bb.2
  bb.2:
    %2:gr32 = PHI %0, %bb.0, %1, %bb.1
    ; ... use %2 ...

After PHI elimination:
  bb.0:
    %0:gr32 = MOV32ri 0
    %2:gr32 = COPY %0        ; copy inserted at end of bb.0
    JMP_1 %bb.2
  bb.1:
    %1:gr32 = MOV32ri 1
    %2:gr32 = COPY %1        ; copy inserted at end of bb.1
    JMP_1 %bb.2
  bb.2:
    ; %2 now defined by copies in predecessors
```

### 89.2.1 Critical Edge Splitting

A critical edge connects a block with multiple successors to a block with multiple predecessors. Inserting a PHI-copy on a critical edge requires splitting the edge with a new basic block; otherwise the copy would execute on paths that should not trigger it. `SplitCriticalEdges` runs as a sub-step of `PHIElimination`:

```cpp
// PHIElimination.cpp: SplitCriticalEdge
MachineBasicBlock *NewBB = MF.CreateMachineBasicBlock();
MF.insert(SuccBB, NewBB);
NewBB->addSuccessor(SuccBB);
PredBB->replaceSuccessor(SuccBB, NewBB);
// Insert COPY in NewBB, then unconditional jump to SuccBB
```

## 89.3 SlotIndexes

`SlotIndexes` (`llvm/lib/CodeGen/SlotIndexes.cpp`) assigns a dense linear ordering to every `MachineInstr` in a function. Slots are spaced 4 apart to allow fine-grained sub-slot numbering:

```
Slot spacing:
  [4N+0] = REG_DEF slot  (register output point)
  [4N+1] = <reserved for sub-register uses>
  [4N+2] = USE slot      (operand read point)
  [4N+3] = <reserved>
```

`SlotIndex` is a lightweight wrapper around a pointer into the index list. Two slot indices can be compared to determine ordering:

```cpp
SlotIndex Start = LIS->getInstructionIndex(MI).getRegSlot();
SlotIndex End   = LIS->getInstructionIndex(SuccMI).getBaseIndex();
bool Precedes = Start < End;
```

`SlotIndexes` also provides `getInstructionFromIndex(SlotIndex)` — the inverse map from index back to `MachineInstr *`. This is heavily used by LiveIntervals.

## 89.4 LiveIntervals

`LiveIntervals` (`llvm/lib/CodeGen/LiveIntervals.cpp`) is the central analysis of the pre-RA pipeline. It computes for every virtual register a `LiveInterval` — the union of half-open `[def, kill)` slot index ranges over which the register holds a live value.

### 89.4.1 LiveInterval and LiveRange

```cpp
// llvm/include/llvm/CodeGen/LiveInterval.h
struct LiveRange {
  SlotIndex start; // inclusive
  SlotIndex end;   // exclusive
  VNInfo   *valno; // value number (which definition)
};

class LiveInterval {
  unsigned Reg;   // virtual register number
  float    Weight; // spill weight (higher = more important to keep in a register)

  SmallVector<LiveRange, 4> Segments; // sorted, non-overlapping

  // Sub-range per lane (for partial register definitions):
  SubRange *SubRanges;
};
```

`VNInfo` records the definition point (slot index of the defining instruction) and tracks PHI values.

### 89.4.2 Computing Live Intervals

The computation is a backward dataflow problem: starting from each use, the algorithm extends the live range backward to the nearest definition. Definitions that span across block boundaries are extended through the liveness sets recorded in each `MachineBasicBlock`. The complete algorithm:

1. Compute live-in sets for every block (using reverse postorder DFS + liveness bit vectors).
2. For each block in reverse postorder, scan backwards: a use extends the live range to the block entry if the register is live-in; a def ends the extension.
3. For values live at block entry, extend through predecessors until the definition block.

```cpp
LiveIntervals *LIS = &getAnalysis<LiveIntervals>();
LiveInterval &LI = LIS->getInterval(VirtReg);
for (const LiveRange &Seg : LI) {
  dbgs() << "[" << Seg.start << ", " << Seg.end << ")\n";
}
```

### 89.4.3 Interference

Two live intervals *interfere* if their live ranges overlap. The register allocator queries interference before assigning physical registers:

```cpp
bool Interferes = LI1.overlaps(LI2);
```

`LiveIntervalUnion` (`llvm/lib/CodeGen/LiveIntervalUnion.cpp`) efficiently represents the union of all live intervals assigned to a given physical register and provides fast interference queries.

## 89.5 RegisterCoalescer

`RegisterCoalescer` (`llvm/lib/CodeGen/RegisterCoalescer.cpp`) eliminates copies by merging copy-related virtual registers. If `%2 = COPY %1` and `%1` and `%2` do not interfere, they can be coalesced into a single virtual register, removing the copy entirely.

### 89.5.1 Coalescing Algorithm

```
For each COPY instruction:
  src = COPY's source register (vreg or phys)
  dst = COPY's destination register

  if LiveInterval(src) and LiveInterval(dst) don't interfere:
    merge them into a single interval
    remove the COPY
```

Coalescing physical registers (e.g., `$eax = COPY %vreg`) is called *trivial coalescing* — it directly constrains the virtual register to that physical register class.

### 89.5.2 Interference Check and Aggressive Coalescing

Conservative coalescing refuses to merge if the combined live range would be harder to color. Aggressive coalescing ignores this concern. LLVM's RegisterCoalescer implements a middle path: it uses a *join-if-profitable* heuristic based on whether the combined interval's spill weight exceeds a threshold:

```cpp
// RegisterCoalescer.cpp: joinCopy()
bool RegisterCoalescer::joinCopy(MachineInstr *CopyMI, bool &Again) {
  CoalescerPair CP(*TRI);
  if (!CP.setRegisters(CopyMI)) return false;
  if (CP.isPhys()) {
    // Physical register coalescing:
    return joinCopies(CP, CopyMI);
  }
  // Virtual-to-virtual: check interference
  LiveInterval &SrcLI = LIS->getInterval(CP.getSrcReg());
  LiveInterval &DstLI = LIS->getInterval(CP.getDstReg());
  if (SrcLI.overlaps(DstLI)) {
    Again = true; // retry after other coalescing
    return false;
  }
  return joinIntervals(CP);
}
```

### 89.5.3 Sub-Register Coalescing

When only part of a wide register is live (e.g., the low 32 bits of a 64-bit register), `RegisterCoalescer` uses lane masks to merge sub-ranges independently. The `SubRange` mechanism in `LiveInterval` enables coalescing of `%xmm0_lo = COPY %vreg_lo` even if the high lane is separately live.

## 89.6 MachineCombiner

`MachineCombiner` (`llvm/lib/CodeGen/MachineCombiner.cpp`) performs algebraic reassociation on the instruction dependency graph to expose instruction-level parallelism. It operates before register allocation so that it can freely create new virtual registers.

The classic example is reassociating a linear chain of additions into a balanced tree:

```
Before (sequential dependency chain — latency N):
  %1 = ADD %a, %b
  %2 = ADD %1, %c
  %3 = ADD %2, %d
  %4 = ADD %3, %e

After reassociation (balanced tree — latency log2(N)):
  %t1 = ADD %a, %b
  %t2 = ADD %c, %d
  %t3 = ADD %t1, %t2
  %4  = ADD %t3, %e
```

The pass queries the target's `TargetInstrInfo::getMachineCombinerPatterns()` to find associative/commutative patterns and `TargetInstrInfo::genAlternativeCodeSequence()` to generate the alternative sequence. The trade-off check uses `MachineCombiner::improvesCriticalPathLen()` to ensure the transformation doesn't lengthen the critical path.

## 89.7 MachineScheduler (Pre-RA)

`MachineScheduler` (`llvm/lib/CodeGen/MachineScheduler.cpp`) performs instruction scheduling before register allocation. Pre-RA scheduling is distinct from post-RA scheduling: it operates on virtual registers, so it cannot use register pressure as a concrete metric but uses estimated pressure based on register class sizes.

### 89.7.1 The Scheduling DAG

The scheduler builds a `ScheduleDAGMI` — a directed acyclic graph of scheduling units (SUnits). Edges represent:

- **Data dependences**: def → use of same register
- **Anti-dependences**: use → def of same register (WAR hazard)
- **Output dependences**: def → def of same register (WAW hazard)
- **Control dependences**: branches and their targets
- **Memory ordering**: ordered loads/stores

Each `SUnit` records its latency (from the `TargetSchedModel`), its node in the DAG, and the set of successors that depend on it.

### 89.7.2 SchedMachineModel

The `TargetSubtargetInfo::getSchedModel()` provides a `MCSchedModel` describing functional units, pipeline resources, and per-instruction issue constraints. This model is generated by TableGen from `SchedMachineModel` records:

```tablegen
// Example: Cortex-A55 simple ALU issue model
def CortexA55Model : SchedMachineModel {
  let IssueWidth = 2;       // 2 instructions per cycle
  let MicroOpBufferSize = 0; // in-order
  let CompleteModel = 1;
}
def : WriteRes<WriteIALU, [CortexA55UnitALU]> { let Latency = 1; }
def : WriteRes<WriteFMUL, [CortexA55UnitFP]>  { let Latency = 4; }
```

### 89.7.3 GenericScheduler

`GenericScheduler` is the default scheduling strategy, implementing a bidirectional list scheduling algorithm with a priority queue. It maintains two scheduling frontiers — top-down from the entry and bottom-up from the exit — and merges them at each cycle. The heuristics are:

1. **Critical path**: prefer instructions on the critical path (those whose removal would shorten the longest latency path)
2. **Register pressure**: prefer instructions that reduce the number of simultaneously live virtual registers
3. **Resource balance**: prefer instructions that utilise underutilised functional units

```cpp
// GenericScheduler's heuristic priority function (simplified):
int GenericScheduler::computePriority(SUnit *SU) {
  int Priority = 0;
  if (isOnCriticalPath(SU)) Priority += CritPathBonus;
  Priority -= RegisterPressureDelta(SU);
  Priority += ResourceBalance(SU);
  return Priority;
}
```

The pre-RA scheduler cooperates with the RegisterCoalescer: when register coalescing opportunities exist (a `COPY` that can be eliminated if its source and destination are adjacent), the scheduler tries to position source and destination definitions close together to maximise coalescing success.

## Chapter Summary

- `TwoAddressInstructionPass` converts three-address MIR instructions into two-address form by inserting `COPY` instructions; commutativity is exploited to avoid unnecessary copies.
- `PHIElimination` replaces SSA `PHI` instructions with `COPY` instructions in predecessor blocks; critical edges are split to maintain correctness.
- `SlotIndexes` assigns a dense linear ordering to every `MachineInstr`, providing the index space on which live intervals are defined.
- `LiveIntervals` computes `[start, end)` live ranges for every virtual register using backward dataflow; interference detection uses `LiveInterval::overlaps()`.
- `RegisterCoalescer` eliminates `COPY` instructions by merging non-interfering virtual register live intervals; sub-register lanes are coalesced independently.
- `MachineCombiner` reassociates commutative instruction chains into balanced trees to expose ILP before register allocation inflates dependencies.
- `MachineScheduler` (pre-RA) uses a DAG-based bidirectional list scheduling algorithm driven by `SchedMachineModel` resources; `GenericScheduler` balances critical path, register pressure, and resource utilisation.
