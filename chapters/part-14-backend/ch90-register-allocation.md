# Chapter 90 — Register Allocation

*Part XIV — The Backend*

Register allocation is the central challenge of code generation: given a function with arbitrarily many virtual registers and a finite set of physical registers, assign a physical register to each virtual register's live range, inserting spill and restore code when no register is available. The quality of register allocation determines instruction count, data cache pressure, and the effectiveness of subsequent scheduling passes. This chapter covers LLVM's allocation infrastructure in depth — the greedy, basic, fast, and PBQP allocators — with particular emphasis on the greedy allocator's live range splitting strategy, which is LLVM's primary mechanism for reducing spilling on register-constrained code.

## 90.1 The Register Allocation Problem

Formally, register allocation is a graph colouring problem: construct an interference graph where each virtual register is a node and each pair of simultaneously live virtual registers is connected by an edge; colour the graph with K colours (physical registers) such that no two adjacent nodes share a colour. When K colours are insufficient, some nodes must be "spilled" — their values are stored to memory and reloaded before each use.

In practice, LLVM's allocators operate on live intervals rather than an explicit graph:

- Each virtual register has a `LiveInterval` (computed by `LiveIntervals`)
- Two virtual registers interfere if their live intervals overlap
- The allocator assigns physical registers by scanning the live interval set, checking for interference with already-assigned physical register intervals

```
Virtual register interference graph (conceptual):
  vreg1 -------- vreg2    (vreg1 and vreg2 live simultaneously)
     \           /
      vreg3----           (vreg3 interferes with vreg1 and vreg2)

Physical registers available: {$r0, $r1, $r2}
Allocation: vreg1 → $r0, vreg2 → $r1, vreg3 → $r2  (no spill)
```

## 90.2 The Greedy Allocator

`RAGreedy` (`llvm/lib/CodeGen/RegAllocGreedy.cpp`) is the default register allocator for optimised builds (-O1 and above). It processes virtual registers in priority order, attempting physical register assignment; when no assignment is possible, it attempts eviction of lower-priority intervals or live range splitting before resorting to spilling.

### 90.2.1 Priority Queue

The allocator maintains a priority queue of live intervals ordered by spill weight (see §90.3). Higher spill-weight intervals are allocated first — they are harder to spill without performance impact. The queue uses a `PriorityQueue<LiveInterval*, std::greater<float>>` ordered by the `weight` field.

```cpp
// RAGreedy.cpp: allocation loop
while (!Queue.empty()) {
  LiveInterval *VirtReg = Queue.top(); Queue.pop();
  MCRegister PhysReg = tryAssign(*VirtReg, AllocationOrder, ...);
  if (PhysReg) {
    Matrix->assign(*VirtReg, PhysReg);
  } else if (tryEvict(*VirtReg, ...)) {
    // evicted something, try again next iteration
  } else if (trySplit(*VirtReg, ...)) {
    // split into smaller intervals, re-enqueue
  } else {
    spillVirtReg(*VirtReg);
  }
}
```

### 90.2.2 AllocationOrder

`AllocationOrder` (`llvm/lib/CodeGen/AllocationOrder.h`) iterates over the physical registers in a register class in a target-preferred order. Targets can provide hints (e.g., preferring callee-saved registers last, preferring the same register used by a preceding `COPY` instruction) via `TargetRegisterInfo::getRegAllocationHints()`. The first register in `AllocationOrder` that doesn't interfere with any already-assigned interval is chosen.

```cpp
// RAGreedy: try to assign a physical register
MCRegister RAGreedy::tryAssign(LiveInterval &VirtReg,
                               AllocationOrder &Order) {
  for (MCRegister PhysReg : Order) {
    if (!Matrix->checkInterference(VirtReg, PhysReg))
      return PhysReg;
  }
  return MCRegister();  // no assignment found
}
```

## 90.3 Spill Weights

Spill weights determine allocation priority and eviction decisions. A higher spill weight means the interval is more expensive to spill. The canonical formula is:

```
SpillWeight = (sum over uses and defs: block_frequency × use_cost) / interval_length
```

Where:
- `block_frequency` is the estimated execution frequency of the block containing the use/def (from `BlockFrequencyInfo`)
- `use_cost` is the type-based cost: 1 for integers, higher for floating-point and vector registers
- `interval_length` is the total span of the live interval in slot index units (longer intervals are cheaper per slot to spill)

PHI operands receive a penalty because spilling a PHI source requires spill code in every predecessor, multiplying the actual cost. The `VirtRegAuxInfo` class computes these weights:

```cpp
// llvm/lib/CodeGen/VirtRegMap.cpp
float VirtRegAuxInfo::weightCalcHelper(LiveInterval &LI, ...) {
  float Weight = 0.0f;
  for (MachineInstr &MI : MRI->reg_nodbg_instructions(LI.reg())) {
    float BlockFreq = MBFI->getBlockFreqRelativeToEntryBlock(MI.getParent());
    Weight += BlockFreq; // simplified; actual code applies per-type factor
  }
  return normalize(Weight, LI.getSize());
}
```

Unspillable intervals (those that hold `undef` values or are marked as hints for the calling convention) receive weight `huge_valf` to prevent the allocator from ever considering them for spilling.

## 90.4 Eviction

When no physical register is directly available for a virtual register `V`, the greedy allocator tries to *evict* an already-assigned interval `E` that is currently occupying a needed physical register, provided that `E`'s spill weight is lower than `V`'s. Evicting `E` frees the physical register for `V`, and `E` is re-enqueued at lower priority.

### 90.4.1 Cascade Eviction Prevention

Naive eviction can cascade: evicting `E` for `V` might require evicting `F` for `E`, then `G` for `F`, potentially devolving into an infinite loop. LLVM prevents this via *eviction cascade counters*: each interval carries a `CascadeCountIn` value; an eviction is only allowed if the evictee's cascade count is strictly less than the evicting interval's count. This ensures the eviction chain forms a DAG:

```cpp
// RAGreedy.cpp: canEvictInterference()
bool RAGreedy::canEvictInterference(LiveInterval &VirtReg,
                                    MCRegister PhysReg,
                                    bool IsHint,
                                    EvictionCost &MaxCost) {
  for (LiveInterval *Intf : Matrix->query(VirtReg, PhysReg)) {
    // Can only evict lower-cascade or lower-weight intervals:
    if (ExtraInfo->getCascade(Intf->reg()) >= ExtraInfo->getCascade(VirtReg.reg()))
      return false;
    if (Intf->weight() >= VirtReg.weight() && !IsHint)
      return false;
  }
  return true;
}
```

## 90.5 Live Range Splitting

When both assignment and eviction fail, the greedy allocator attempts live range splitting: breaking a long live interval into shorter pieces so that each piece can individually be allocated to some available physical register. The pieces are connected by `COPY` instructions (which may themselves be coalesced later).

### 90.5.1 SplitAnalysis

`SplitAnalysis` (`llvm/lib/CodeGen/SplitKit.h`) analyses the current live interval to identify block-level use sites and compute splitting heuristics. It categorises each MBB that intersects the live interval:

- **UseBlocks**: blocks containing at least one use or def
- **ThroughBlocks**: blocks where the interval is live but has no use/def (pure through blocks — candidate for insertion of split points)

```cpp
SplitAnalysis SA(VRM, LIS, Loops);
SA.analyze(&LI);
ArrayRef<SlotIndex> UseSlots = SA.getUseSlots();
```

### 90.5.2 SplitKit

`SplitKit` (`llvm/lib/CodeGen/SplitKit.cpp`) performs the actual splitting. It inserts `COPY` instructions at chosen split points, creates new virtual registers for each piece, and updates `LiveIntervals`. Two main splitting strategies are available:

**Local splitting** (`splitSingleBlock`): splits within a single basic block at the point where the best split reduces the number of interference overlaps. A `COPY` is inserted to transfer the value to the new virtual register just before the first use in the block, and another `COPY` transfers back after the last def.

**Region splitting** (`splitAroundRegion`): for intervals that are live through many blocks with few uses, region splitting cuts the interval at region boundaries. The "region" is typically a loop body:

```
Before region split (loop-carried interval):
  preheader: [live] --------> [loop body: uses] --------> [exit]

After region split:
  preheader: COPY %vreg -> %vreg_region
  loop body: uses %vreg_region (now shorter interval)
  exit:      COPY %vreg_region -> %vreg_orig
```

The key insight is that `%vreg_region` is only live within the loop and may have a different (more available) physical register assignment than the surrounding interval.

```cpp
// SplitKit.cpp: openIntv / selectIntv / closeIntv sequence
SK.reset(&SA, &MDT, &MBFI);
SlotIndex Idx = SK.openIntv();
SK.selectIntv(1);
SK.useIntv(Start, End);  // mark the new interval as "used" in [Start,End)
ArrayRef<unsigned> IntvMap = SK.finish();
```

## 90.6 The Basic Allocator

`RABasic` (`llvm/lib/CodeGen/RegAllocBasic.cpp`) is a simplified linear scan allocator. It processes virtual registers in order of decreasing spill weight and assigns the first available physical register without attempting eviction or splitting. Intervals that cannot be assigned are spilled immediately.

`RABasic` is useful as a baseline for testing allocation correctness and as a fallback for targets where the greedy allocator's complexity is not warranted. It is also the foundation used to test new target descriptions and register class constraints.

## 90.7 The Fast Allocator

`RAFast` (`llvm/lib/CodeGen/RegAllocFast.cpp`) is the `-O0` allocator. It allocates registers one basic block at a time in a single forward scan, spilling and reloading at block boundaries. It maintains no inter-block live interval information:

```
For each MBB:
  For each MI in order:
    For each use operand: ensure physical register assigned (reload if spilled)
    Assign physical register to each def operand
    At block boundary: spill all live virtual registers
```

`RAFast` intentionally sacrifices quality for compile-time: it is O(n) in the number of instructions per block. The spill/reload code it generates at block exits/entries is cleaned up by subsequent coalescing and dead-code elimination.

## 90.8 PBQP Allocator

`RAPBQP` (`llvm/lib/CodeGen/RegAllocPBQP.cpp`) models register allocation as a Partitioned Boolean Quadratic Programming problem. Each virtual register is a node with a cost vector (one entry per physical register candidate + one entry for spill). Pairs of interfering virtual registers are connected by cost matrices encoding the penalty for assigning the same physical register.

The PBQP solver (`llvm/lib/Target/PBQP/...`) finds an assignment that minimises total cost using a graph-reduction algorithm. It is particularly useful for targets with complex heterogeneous register constraints where the greedy allocator's heuristics produce poor results (notably MIPS with its special-purpose registers).

```
PBQP cost vector for vreg with 3 candidates:
  [cost_$r0, cost_$r1, cost_spill]

PBQP interaction matrix for two interfering vregs:
  (row = assignment for vreg1, col = assignment for vreg2)
  [    0,    0,    0  ]   ($r0, $r0 = infinity since they conflict)
  [    0,    0,    0  ]
  [    0,    0,    0  ]
  (off-diagonal entries are 0; diagonal entries that correspond to
   the same physical register get a large penalty)
```

## 90.9 Spill Placement Optimisation

`SpillPlacementAnalysis` (`llvm/lib/CodeGen/SpillPlacement.cpp`) determines the optimal basic blocks in which to insert spill and restore code when a virtual register must be spilled. Rather than spilling at every use/def, it places spill code at block boundaries to minimise expected spill frequency:

The algorithm is a max-flow / min-cut problem on the CFG: for each "through" block (the interval is live but has no use/def), placing spill code at the block's entry (a "restore") or exit (a "spill") has a cost proportional to that block's execution frequency. The optimal placement minimises total weighted spill cost.

```cpp
SpillPlacement::BlockConstraint C;
C.Number = MBB->getNumber();
C.Entry  = SpillPlacement::PrefSpill; // want a spill at entry
C.Exit   = SpillPlacement::PrefReg;  // want a register at exit
SA.getConstraints(Constraints);
BP.prepare(LI);
BP.addConstraints(Constraints);
BP.finish();
```

## 90.10 Register Bank Allocation in GlobalISel

GlobalISel introduces an intermediate allocation step: *register bank selection* (`RegBankSelect` pass), which runs before full register allocation. Register banks are coarser than register classes (e.g., "general purpose integer" vs. "floating point"), and assigning a bank early allows the legalizer and combiner to make better decisions.

After `RegBankSelect`, `RegAllocFast` (for `-O0`) or a bank-aware variant of `RAGreedy` assigns physical registers within the selected bank. The key difference from SelectionDAG-based allocation is that GlobalISel's `VirtRegMap` tracks bank assignments alongside physical register assignments:

```cpp
// GlobalISel register bank allocation:
const RegisterBank *RB = RBI->getRegBank(VReg, MRI, *TRI);
MRI.setRegBank(VReg, *RB);
// Later, RAGreedy assigns a physical register within the bank:
MCRegister PhysReg = tryAssignWithinBank(VReg, RB, ...);
```

## Chapter Summary

- Register allocation maps virtual registers to physical registers, inserting spill/restore code when physical registers are exhausted; the problem is NP-hard (graph colouring) but tractable with good heuristics.
- `RAGreedy` processes intervals by decreasing spill weight, attempting assignment, eviction, and live range splitting before spilling; cascade counters prevent infinite eviction chains.
- Spill weights combine per-use execution frequency, type cost, and interval length; PHI operands receive an extra penalty for multi-block spill propagation.
- Live range splitting (`SplitKit`) divides a long interval into shorter pieces connected by copies; local splitting handles single-block splits, region splitting handles loop-boundary splits.
- `RABasic` is a quality-baseline linear scan allocator with no eviction or splitting; `RAFast` is the O(n)-per-block `-O0` allocator that spills at every block boundary.
- `RAPBQP` models allocation as a quadratic optimisation problem, useful for targets with heterogeneous register constraints.
- `SpillPlacementAnalysis` uses a min-cut algorithm on the CFG to minimise weighted spill code insertion.
- GlobalISel's `RegBankSelect` assigns coarse register banks before full physical-register allocation, enabling bank-aware combining and legalization.


---

@copyright jreuben11
