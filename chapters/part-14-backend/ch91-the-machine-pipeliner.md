# Chapter 91 — The Machine Pipeliner

*Part XIV — The Backend*

Software pipelining is one of the most powerful loop optimisations available at the machine instruction level. By overlapping the execution of multiple loop iterations, a pipeliner can fill every functional unit slot in an in-order or VLIW processor, achieving throughput limited only by the resources of the machine. LLVM's `MachinePipeliner` implements Swing Modulo Scheduling (SMS), a well-studied algorithm that combines modulo scheduling with a careful node ordering to produce compact, schedule-efficient loop kernels. This chapter explains the mathematical foundations of modulo scheduling, traces through the SMS algorithm as implemented in LLVM, describes the rotating register file model used for kernels, and documents which targets currently benefit from software pipelining.

## 91.1 Software Pipelining: Concepts

### 91.1.1 Loop Overlap and the Kernel

Consider a simple loop body with three instructions: Load (latency 4), Multiply (latency 3), Store (latency 1). In a scalar schedule, each iteration takes 8 cycles. A pipelined schedule overlaps iteration N with iteration N+1 at an *initiation interval* II: the number of cycles between starting successive iterations.

```
Cycle  | iter 0       | iter 1       | iter 2
-------+-------------+-------------+----------
  0    | Load(0)      |              |
  1    |              | Load(1)      |
  2    | Mul(0)       |              | Load(2)
  3    |              | Mul(1)       |
  4    | Store(0)     |              | Mul(2)
  5    |              | Store(1)     |
  6    |              |              | Store(2)
```

At II=2, each iteration starts 2 cycles after the previous one. The steady-state "kernel" is the repeating pattern of cycles 2–3; the prologue initialises the pipeline and the epilogue drains it. Steady-state throughput is one iteration per II cycles, versus one per 8 cycles unpipelined — a 4× improvement.

### 91.1.2 Initiation Interval Bounds

The II cannot be made arbitrarily small. Two lower bounds constrain the minimum II (MII):

**Resource minimum (ResMII)**: each functional unit has a finite capacity. If the loop body contains 3 multiply instructions and the target has 1 multiplier, at least 3 cycles must elapse between iterations:

```
ResMII = max over all resources r: ceil(usage(r) / capacity(r))
```

**Recurrence minimum (RecMII)**: cyclic dependences within the loop (carried dependences) impose a minimum length on cycles in the dependence graph:

```
RecMII = max over all cycles C: ceil(sum_of_latencies(C) / sum_of_distances(C))
```

where the "distance" of a dependence edge is the number of loop iterations it spans (0 for intra-iteration, 1 for a dependence from iteration N to N+1, etc.).

The minimum initiation interval is:

```
MII = max(ResMII, RecMII)
```

The scheduler tries to find a valid schedule at II = MII, and if none exists, increments II and tries again up to a configurable maximum.

## 91.2 Modulo Scheduling

Modulo scheduling assigns each instruction a *cycle* within the kernel window [0, II). The schedule must satisfy:

1. **Resource constraints**: in each cycle modulo II, the set of instructions scheduled to that slot must not exceed the resource capacity.
2. **Dependence constraints**: if instruction B depends on instruction A with latency L and dependence distance D, then:
   `sched(B) ≡ sched(A) + L (mod II)` with `sched(B) - sched(A) + D*II ≥ L`

The schedule is *modular*: instructions scheduled at cycle c in the kernel actually execute at cycle `c + n*II` in iteration n.

### 91.2.1 The Stage Vector

The *stage* of an instruction is `sched(inst) / II` (integer division). Instructions in stage 0 execute in the first kernel cycle (at cycle `sched(inst)`); instructions in stage 1 execute one full kernel later, etc. The number of stages determines the number of loop-carried register lifetimes and the depth of the prologue/epilogue.

```
Stage 0 instructions: sched in [0, II)
Stage 1 instructions: sched in [II, 2*II)
...
Stage S instructions: sched in [S*II, (S+1)*II)

Kernel repeats from stage 0 through stage S simultaneously.
```

## 91.3 Swing Modulo Scheduling in LLVM

LLVM's `MachinePipeliner` (`llvm/lib/CodeGen/MachinePipeliner.cpp`) implements the SMS algorithm by Llosa et al. (1996). SMS improves over naive modulo scheduling by using a node ordering that reduces the number of failed scheduling attempts.

### 91.3.1 Stage 1: Node Ordering

SMS computes an ordering of instructions that reflects both data dependences and recurrence structure. The order is derived from:

1. **Recurrence decomposition**: identify strongly connected components (SCCs) in the data dependence graph (DDG). SCCs correspond to cyclic dependences; nodes within an SCC are scheduled together.
2. **ASAP/ALAP scheduling**: compute the earliest (ASAP) and latest (ALAP) possible cycles for each node under the dependence constraints. The scheduling window `[ASAP(n), ALAP(n)]` determines the node's flexibility.
3. **Priority ordering**: nodes are ordered by increasing scheduling flexibility (ALAP - ASAP). Nodes with zero slack are on the critical path and are scheduled first.

```cpp
// MachinePipeliner.cpp: computeNodeOrder()
void SwingSchedulerDAG::computeNodeOrder(NodeSetType &NodeSets) {
  for (NodeSet &NSet : NodeSets) {
    // Order each SCC by ASAP/ALAP
    orderNodes(NSet);
  }
  // Merge node sets: recurrence-critical first, then remaining
}
```

### 91.3.2 Stage 2: Scheduling

Once the node ordering is fixed, SMS schedules each instruction within its window `[ASAP(n), ALAP(n)]`. For each instruction, it tries each cycle in the window and checks:

- Resource availability in that cycle (modulo II)
- Data dependence satisfaction

If no slot is available, II is incremented and scheduling restarts from scratch.

```cpp
// MachinePipeliner.cpp: scheduleNode()
bool SMSchedule::insert(SUnit *SU, int StartCycle, int EndCycle, int II) {
  for (int Cycle = StartCycle; Cycle <= EndCycle; ++Cycle) {
    if (ProcResourceModel.isAvailable(SU, Cycle % II)) {
      if (dependencesSatisfied(SU, Cycle, II)) {
        Schedule[Cycle % II].push_back(SU);
        return true;
      }
    }
  }
  return false; // needs larger II
}
```

### 91.3.3 Stage 3: Kernel Construction

After finding a valid schedule, SMS constructs the actual kernel MachineBasicBlock. Instructions from different stages of the same schedule are *interleaved* — stage 0 instructions from iteration N run together with stage 1 instructions from iteration N-1. The kernel represents this steady state:

```
Kernel (II=3, 2 stages):
  Cycle 0: Load[iter N],    Store[iter N-1]
  Cycle 1: Mul[iter N],     Load[iter N+1]
  Cycle 2: Store[iter N],   Mul[iter N+1]
```

The kernel `MachineBasicBlock` is then wrapped with a count-down loop that decrements the iteration count by 1 per kernel execution, since each kernel execution advances the pipeline by one iteration.

## 91.4 The Rotating Register File

Loop-carried values in a pipelined schedule span multiple kernel iterations, so the same "logical" register holds different iteration's values simultaneously. Without special hardware support, this requires a different physical register for each stage — leading to register pressure proportional to the number of stages.

Targets with *rotating register files* (notably Hexagon and Itanium) provide hardware support for this: register renaming happens automatically each kernel cycle, so the code can refer to a single logical register name while the hardware assigns different physical registers per stage.

### 91.4.1 Hexagon Rotating Registers

Hexagon provides a modulo-rotating register file for software pipelining. When pipelining is active, a set of registers (typically R0–R31 in a rotating window) are renamed automatically by the hardware at the start of each kernel iteration. LLVM's Hexagon backend generates pipelined loops using this model:

```
// Hexagon pipelined loop (conceptual):
loop0(.loop_start, trip_count)
.loop_start:
  r0 = memw(r1++#4)        ; Load (stage 0)
  r2 = mpy(r0, r3)         ; Mul  (stage 1, uses r0 from prev iter)
  memw(r4++#4) = r2         ; Store (stage 2)
:endloop0
```

The `loop0`/`endloop0` pair provides hardware loop management including zero-overhead branch and rotation.

### 91.4.2 Software Register Renaming

For targets without rotating register file hardware (x86, ARM, RISC-V), LLVM's pipeliner inserts explicit register copies in the prologue and epilogue to handle stage overlap. The `ModuloScheduleExpander` (`llvm/lib/CodeGen/ModuloSchedule.cpp`) performs this expansion, creating separate virtual registers for each pipeline stage and emitting the corresponding copies.

```cpp
// ModuloSchedule.cpp: expand()
void ModuloScheduleExpander::expand() {
  emitPrologue(Schedule, NumStages);
  emitKernel(Schedule);
  emitEpilogue(Schedule, NumStages);
}
```

## 91.5 VLIW Instruction Bundling

On VLIW targets (Hexagon, EPIC), multiple operations are bundled into a single "wide instruction" that issues simultaneously. The machine pipeliner must respect the bundling constraints when scheduling: operations that belong to the same bundle must be scheduled at the same cycle and must fit within the bundle's resource constraints.

LLVM represents VLIW bundles using the `BUNDLE` pseudo-instruction followed by the component MachineInstrs. The `MachineInstr::isBundled()` predicate distinguishes bundled components from standalone instructions:

```cpp
// Iterating over bundle contents:
for (auto It = BundleMI.getBundleIterator(); !It.isEnd(); ++It) {
  MachineInstr &ComponentMI = *It;
  // process each component
}
```

The Hexagon `PacketizerDAGMutation` (`llvm/lib/Target/Hexagon/HexagonPacketizer.cpp`) post-processes the scheduler output to form legal packets, respecting slot constraints and resource limits.

## 91.6 Target Uptake

Not all LLVM targets enable the machine pipeliner. The pipeliner requires a complete `SchedMachineModel` with resource descriptions and dependence latencies. Current target status:

| Target | Status | Notes |
|---|---|---|
| Hexagon | Production | Primary use case; rotating register file support |
| PowerPC | Enabled | Primarily for in-order embedded cores (e6500) |
| RISC-V | Experimental | Behind `--enable-pipeliner` flag |
| ARM Thumb-2 | Disabled | In-order cores not common enough to justify complexity |
| x86 | Disabled | Out-of-order execution makes software pipelining less valuable |

A target opts into pipelining by:
1. Returning `true` from `TargetSubtargetInfo::enableMachinePipeliner()`
2. Providing a `SchedMachineModel` with accurate resource and latency data
3. Implementing `TargetInstrInfo::isLoopDependentOnLoop()` and related hooks

## 91.7 Profitability Analysis

The `MachinePipeliner` only pipelines loops that are expected to benefit. Key profitability criteria:

**Minimum trip count**: pipelining is unprofitable for very short-running loops because the prologue and epilogue code bloat overshadows any kernel speedup. The pipeliner requires `trip_count ≥ num_stages + 1` (the number of stages must fit within the trip count). The threshold is configurable via `MaxIterations`.

**Maximum II**: if the computed MII exceeds a threshold (default 20 cycles), the loop is considered too complex to pipeline profitably.

**Dependency distance**: if all recurrence distances are 0 (no loop-carried dependences), modulo scheduling degenerates to regular instruction scheduling and the overhead is not justified.

```cpp
// MachinePipeliner.cpp: canPipelineLoop()
bool MachinePipeliner::canPipelineLoop(MachineLoop &L) {
  if (L.getNumBlocks() != 1) return false;  // single-block loops only
  if (!L.getLoopLatch()) return false;
  unsigned TC = TII->getLoopConstantTripCount(L);
  if (TC < MinTripCount) return false;
  return true;
}
```

**Register pressure**: if pipelining would increase register pressure beyond the available register count (because multiple pipeline stages require separate registers for the same logical value), the pipeliner falls back to a non-pipelined schedule.

## Chapter Summary

- Software pipelining overlaps loop iterations to fill functional unit slots; the initiation interval II is bounded below by `max(ResMII, RecMII)` where ResMII is the resource limit and RecMII is the recurrence limit.
- Modulo scheduling assigns instructions to cycles modulo II; instructions with the same cycle-mod-II slot compete for the same resources in every kernel iteration.
- SMS (Swing Modulo Scheduling) improves upon naive modulo scheduling via SCC-based node ordering and ASAP/ALAP-driven priority scheduling; LLVM's `MachinePipeliner` implements SMS.
- The kernel is the steady-state repeating loop body; the prologue fills the pipeline and the epilogue drains it; stage count equals `max_sched_cycle / II`.
- Rotating register files (Hexagon, Itanium) provide hardware register renaming across iterations; targets without rotating registers use explicit copies inserted by `ModuloScheduleExpander`.
- VLIW targets require instruction bundling constraints to be respected during scheduling; Hexagon's `PacketizerDAGMutation` forms legal packets from the scheduler output.
- Software pipelining is currently production-grade on Hexagon, enabled on PowerPC, and experimental on RISC-V; profitability requires sufficient trip count, bounded II, and manageable register pressure.
