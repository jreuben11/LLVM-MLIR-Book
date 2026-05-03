# Chapter 199 — llvm-mca: Static Performance Analysis

*Part XXIX — Compiler Tooling, Kernel Integration, and Binary Analysis*

You are reviewing a vectorized matrix-multiply kernel before committing it to a performance-critical library. The loop body looks tight — six AVX-512 instructions, a well-unrolled count, no obvious spills. But how many cycles per iteration will it actually consume on a Skylake-X server? Will it be bottlenecked by port 0's FMA units or by port 5's shuffle throughput? You could deploy a benchmark, profile it with `perf stat`, and iterate — a cycle that takes hours on a shared CI machine. Or you could pipe the compiler output through `llvm-mca` and get an answer in milliseconds, without executing a single instruction.

`llvm-mca` is LLVM's static performance analysis tool. It simulates out-of-order execution of a loop kernel on a target CPU micro-architecture using LLVM's own machine scheduling models — the same `SchedMachineModel` TableGen descriptions that guide the instruction scheduler in the backend. It reads assembly input, models dispatch, execution, and retirement in a configurable out-of-order processor, and produces throughput and bottleneck reports without running the code. This chapter covers the tool's architecture, its resource model, how to read its output, and how to integrate it into a development and CI workflow.

---

## 199.1 What llvm-mca Is — and Is Not

### 199.1.1 Scope of the Simulation

`llvm-mca` is a **static, source-level throughput simulator**. It answers one question: given this sequence of instructions repeating in a loop on this CPU model, what is the theoretical steady-state throughput in cycles per iteration?

The simulation operates on `MCInst` objects — the machine-code representation in LLVM's MC layer — decoded from a text assembly file. For each instruction the tool queries `MCInstrInfo` and `MCSubtargetInfo` to retrieve the scheduling class assigned in TableGen, then drives a processor pipeline model consisting of a front-end dispatch stage, an out-of-order execution engine, and a retire stage.

Important constraints the simulation does **not** model:

- **Branch prediction and control flow**: `llvm-mca` treats the input as a straight-line repeating block. Branches inside the region are legal syntax but their control-flow semantics are ignored. The tool warns if it encounters a return instruction (`warning: found a return instruction in the input assembly sequence`).
- **Cache and memory hierarchy**: load and store latencies are the scheduling-model values from TableGen (e.g., 5 cycles for an L1-hit load on Skylake), not actual memory-system latencies. Cache misses, TLB pressure, and memory bandwidth constraints are invisible.
- **Micro-architectural details not in TableGen**: effects such as µop cache (DSB) vs. legacy decode (MITE) switching, store forwarding stalls, partial-flag merging penalties, and micro-fusion rules that differ between front-end and execution are only approximated and partly covered by `CustomBehaviour` plugins.
- **System effects**: interrupts, context switches, NUMA, OS scheduling.

### 199.1.2 When to Use llvm-mca

The tool is most valuable for:

1. **Code review of hot loops**: quickly checking whether a proposed kernel meets its throughput target before benchmarking.
2. **Validating TableGen scheduling models**: comparing simulated IPC/RThroughput against `perf stat` measurements to catch incorrect `WriteRes` latencies or missing port assignments (Section 199.11).
3. **CI performance regression detection**: extracting `BlockRThroughput` from JSON output and asserting it stays below a threshold (Section 199.13).
4. **Compiler output auditing**: piping `clang -O2 -S` output through `llvm-mca` to verify the vectorizer or scheduler made the right choices.

### 199.1.3 Comparison with Other Tools

| Tool | Approach | Strengths | Limitations |
|---|---|---|---|
| `llvm-mca` | Static simulation via TableGen models | Fast, cross-architecture, no hardware needed | Model accuracy depends on TableGen quality |
| `perf stat` | Hardware PMU counters | Accurate real-hardware measurement | Requires execution, no per-port breakdown on all PMUs |
| IACA | Static simulation (Intel-only binary) | Accurate for Intel targets | Discontinued in 2019, Intel targets only |
| uiCA | Static simulation via reverse engineering | High accuracy on Skylake/Zen 4 | x86-only, academic, not integrated into LLVM |
| `llvm-exegesis` | Empirical micro-benchmark per instruction | Ground-truth latency/throughput per opcode | Measures one instruction at a time, requires hardware |
| gem5 | Full-system cycle-accurate simulation | Models cache, DRAM, OS | Extremely slow, not suitable for hot-loop analysis |

`llvm-exegesis` (covered in [Ch174 — Performance Engineering](../part-25-operations-contribution/ch174-performance-engineering.md)) and `llvm-mca` are complementary: `llvm-exegesis` calibrates the per-instruction latency and throughput values in TableGen; `llvm-mca` uses those values to predict kernel-level throughput.

---

## 199.2 Architecture of llvm-mca

### 199.2.1 Source Layout

The tool splits into a library and a driver:

- **`llvm/lib/MCA/`** — the `LLVMMCA` library. All pipeline logic, hardware models, and views live here. It is a stable API usable in other tools.
- **`llvm/tools/llvm-mca/`** — the `llvm-mca` binary driver: parses command-line options, reads assembly via the MC disassembler/parser, constructs a `mca::Context`, builds the pipeline, runs it, and prints views.

The library is intentionally decoupled from the driver. Downstream tools — IDEs, profiling GUIs, pass managers — can embed `LLVMMCA` and drive it programmatically without using the command-line interface.

Relevant headers:

- [`Pipeline.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/MCA/Pipeline.h)
- [`Stages/EntryStage.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/MCA/Stages/EntryStage.h)
- [`Stages/DispatchStage.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/MCA/Stages/DispatchStage.h)
- [`Stages/ExecuteStage.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/MCA/Stages/ExecuteStage.h)
- [`Stages/RetireStage.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/MCA/Stages/RetireStage.h)
- [`HardwareUnits/ResourceManager.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/MCA/HardwareUnits/ResourceManager.h)
- [`InstrBuilder.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/MCA/InstrBuilder.h)
- [`CustomBehaviour.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/MCA/CustomBehaviour.h)

### 199.2.2 mca::Pipeline and Stages

`mca::Pipeline` is a chain of `mca::Stage` objects. Each cycle, the pipeline calls `Stage::execute()` on every stage in order, propagating instructions downward. The canonical pipeline for the default out-of-order model has four stages:

**EntryStage** (`llvm/lib/MCA/Stages/EntryStage.cpp`)

Reads pre-decoded `mca::Instruction` objects from the source region and feeds them into the pipeline one per cycle up to the dispatch width limit. It is the source of the instruction stream and handles the region repeat logic (the `--iterations` count).

**DispatchStage** (`llvm/lib/MCA/Stages/DispatchStage.cpp`)

Simulates the front-end: dispatch width, rename registers against the `RegisterFile` hardware unit, and allocate reorder buffer (ROB) entries. On each cycle it attempts to dispatch up to `DispatchWidth` µops. If the ROB is full, or the register file has no free physical registers, it stalls — modeling back-pressure from the execution engine.

**ExecuteStage** (`llvm/lib/MCA/Stages/ExecuteStage.cpp`)

The heart of the simulation. It interacts with `ResourceManager` to:

1. Issue ready instructions (all register operands available, resource available) to the appropriate `ProcResourceUnit`.
2. Advance execution by one cycle — decrement remaining cycles for in-flight instructions.
3. Complete instructions whose execute latency has expired and write back results.

**RetireStage** (`llvm/lib/MCA/Stages/RetireStage.cpp`)

Retires instructions in program order from the ROB via `RetireControlUnit`. It notifies `HWEventListener` subscribers (the views) when each instruction retires, providing the timeline data that `TimelineView` uses.

### 199.2.3 mca::InstrBuilder

`mca::InstrBuilder` (`llvm/lib/MCA/InstrBuilder.cpp`) converts an `MCInst` into an `mca::Instruction`. The conversion queries:

- `MCInstrInfo::get(Opcode)` for the instruction descriptor.
- `MCSubtargetInfo::getSchedModel()` to get the `MCSchedModel`.
- The scheduling class index from `MCInstrInfo` and the `MCSchedModel`'s write/read latency tables.

The result is an `mca::Instruction` annotated with: list of `WriteState` objects (output operands with latency), list of `ReadState` objects (input operands), number of µops, and the set of `ProcResourceUnits` the instruction requires.

### 199.2.4 ResourceManager

`mca::ResourceManager` (`llvm/lib/MCA/HardwareUnits/ResourceManager.cpp`) maintains the availability of every `ProcResourceUnit` and `ProcResGroup` declared in the target's TableGen `SchedMachineModel`. On each cycle it:

1. Accepts newly-issued instructions and marks resources busy.
2. Decrements occupancy counters.
3. Frees resources when instructions complete.
4. Reports resource pressure to the `HWEventListener` subscribers.

### 199.2.5 Other Hardware Units

- **`LSUnit`** (`llvm/include/llvm/MCA/HardwareUnits/LSUnit.h`) — models the load/store queue. Size is configurable via `--lqueue` and `--squeue`. It tracks memory dependencies between loads and stores.
- **`RetireControlUnit`** — models the reorder buffer. Tracks in-order retirement and signals the pipeline when the ROB is full.
- **`RegisterFile`** — models the physical register file. Tracks rename mappings from architectural to physical registers. Configurable via `--register-file-size`.

### 199.2.6 HWEventListener

Views (SummaryView, TimelineView, ResourcePressureView, BottleneckAnalysis) implement `mca::HWEventListener` and register with the pipeline. They receive callbacks such as `onEvent<HWInstructionIssuedEvent>`, `onEvent<HWInstructionRetiredEvent>`, and `onEvent<HWPressureEvent>` and accumulate statistics for printing at the end of the simulation.

### 199.2.7 Bootstrap: From Assembly Text to MCInst

Before the MCA pipeline receives its first instruction, the `llvm-mca` driver performs the assembly parsing that the MC layer handles everywhere else in LLVM (see [Ch94 — The MC Layer](../part-14-backend/ch94-the-mc-layer-and-mir-test-infrastructure.md)). The driver:

1. Calls `TargetRegistry::lookupTarget()` with the triple string to get a `Target`.
2. Creates an `MCSubtargetInfo` for the target CPU (e.g., `skylake`) and feature set.
3. Creates an `MCAsmInfo`, `MCInstrInfo`, `MCRegisterInfo`, and `MCContext`.
4. Creates an `MCAsmParser` with an `MCAsmLexer` over the input stream.
5. The parser emits `MCInst` objects (via `MCAsmBackend` and `MCStreamer`) which are intercepted by a custom `MCStreamer` subclass — `llvm::mca::CodeRegionGenerator::MCStreamerWrapper` — that accumulates them into `mca::CodeRegion` objects, one per `# LLVM-MCA-BEGIN` / `# LLVM-MCA-END` pair.

Each `CodeRegion` is a flat list of `MCInst` objects. The `mca::SourceMgr` wraps these regions and feeds them to `mca::InstrBuilder` which transforms each `MCInst` into an `mca::Instruction` with its scheduling annotations. This two-step process (parse → decode → annotate) means `llvm-mca` reuses the full MC layer assembly parser — ensuring it handles the same pseudo-instructions, directives, and AT&T/Intel syntax variants as the assembler itself.

The architecture is summarized as a pipeline of data transformations:

```text
Input .s file
     |
     v  MCAsmParser (MC layer)
MCInst stream
     |
     v  mca::InstrBuilder  +  MCSchedModel (from TableGen)
mca::Instruction stream (with WriteState/ReadState/ResourceDef)
     |
     v  mca::Pipeline (EntryStage → DispatchStage → ExecuteStage → RetireStage)
HWEventListener callbacks
     |
     v  Views (SummaryView, TimelineView, ResourcePressureView, BottleneckAnalysis)
Text / JSON output
```

---

## 199.3 The Processor Resource Model

### 199.3.1 TableGen Scheduling Primitives

The resource model that `llvm-mca` simulates is the same model described in TableGen and used by the machine instruction scheduler in the LLVM backend. The key TableGen constructs are (see [Ch82 — TableGen Deep Dive](../part-14-backend/ch82-tablegen-deep-dive.md) and [Ch83 — The Target Description](../part-14-backend/ch83-target-description.md)):

- **`ProcResource<N>`** / **`ProcResourceUnits`**: A named execution resource with a given number of units. Each physical execution port on the CPU maps to one `ProcResourceUnits` record.
- **`ProcResGroup`**: A group of `ProcResourceUnits` that can collectively satisfy a resource requirement. Skylake's integer ALUs (ports 0, 1, 5, 6) form `SKLPort0156`.
- **`SchedWriteRes`** / **`WriteRes`**: Binds a scheduling write class to the resources it consumes, the total latency, and the number of µops it produces.
- **`ReadAdvance`**: Shortens the effective latency for a consumer instruction when reading a specific type of value (e.g., reading from a forwarding path).

### 199.3.2 Skylake Resource Example

The Intel Skylake model lives in [`llvm/lib/Target/X86/X86SchedSkylake.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/X86/X86SchedSkylake.td). A simplified excerpt showing port declarations and a simple `WriteADD` binding:

```tablegen
// Port declarations
def SKLPort0  : ProcResource<1>;
def SKLPort1  : ProcResource<1>;
def SKLPort5  : ProcResource<1>;
def SKLPort6  : ProcResource<1>;

// Port group: integer ALU can use any of ports 0, 1, 5, 6
def SKLPort0156 : ProcResGroup<[SKLPort0, SKLPort1, SKLPort5, SKLPort6]>;

// A simple integer add: 1 µop, 1-cycle latency, dispatches to port 0156
def SKLWriteResGroup_ADD : SchedWriteRes<[SKLPort0156]> {
  let Latency = 1;
  let NumMicroOps = 1;
}
def : InstRW<[SKLWriteResGroup_ADD], (instregex "ADD(8|16|32|64)rr")>;

// A scalar FP multiply: 1 µop, 4-cycle latency, dispatches to port 0 or 1
def SKLWriteResGroup_FMUL : SchedWriteRes<[SKLPort01]> {
  let Latency = 4;
  let NumMicroOps = 1;
}
def : InstRW<[SKLWriteResGroup_FMUL], (instregex "VMULSSrr|VMULPSrr")>;
```

`llvm-mca` reads these values at runtime through `MCSubtargetInfo::getSchedModel()` — the same interface the instruction scheduler uses during compilation. This is why `llvm-mca` results reflect the current state of the LLVM scheduling model for a target, and why inaccuracies in the model produce inaccurate `llvm-mca` predictions.

---

## 199.4 Running llvm-mca

### 199.4.1 Basic Invocation

The binary ships in the LLVM tools directory. On Ubuntu with LLVM 22:

```bash
# Analyse a Skylake loop kernel
llvm-mca -mtriple=x86_64 -mcpu=skylake --iterations=100 loop.s

# Analyse an AArch64 Cortex-A57 kernel
llvm-mca -mtriple=aarch64 -mcpu=cortex-a57 kernel.s

# Add all views at once
llvm-mca -mtriple=x86_64 -mcpu=znver4 --all-views kernel.s
```

Key flags:

| Flag | Default | Meaning |
|---|---|---|
| `--iterations=N` | 100 | Number of times the block repeats |
| `--dispatch=N` | from model | Override the dispatch width |
| `--timeline` | off | Enable TimelineView |
| `--resource-pressure` | on | Enable ResourcePressureView |
| `--bottleneck-analysis` | off | Enable BottleneckAnalysis view |
| `--json` | off | Output machine-readable JSON |
| `--instruction-tables` | off | Print per-instruction resource table |
| `--timeline-max-iterations=N` | all | Limit timeline rows |
| `--timeline-max-cycles=N` | 80 | Limit timeline width in cycles |

### 199.4.2 Region Markers

Without region markers, `llvm-mca` analyses the entire assembly file. To target a specific block — the hot loop body inside a larger function — use the annotation comments:

```asm
# Preamble: set up loop invariants (not analysed)
    vmovaps (%rsi), %zmm8
    vmovaps (%rdi), %zmm9

# LLVM-MCA-BEGIN fused-multiply-add-loop
    vfmadd231ps %zmm0, %zmm1, %zmm2
    vfmadd231ps %zmm3, %zmm4, %zmm5
    vfmadd231ps %zmm6, %zmm7, %zmm8
    addq $4, %rax
    cmpq %rdx, %rax
    jne .Lloop
# LLVM-MCA-END
```

Multiple named regions can coexist in one file. Each region produces a separate report labelled with the name following `LLVM-MCA-BEGIN`.

### 199.4.3 Using with Compiler Output

The most practical workflow is to pipe compiler output directly:

```bash
# Compile to assembly, analyse on Zen4
clang -O2 -march=znver4 -S -o - kernel.c | \
    llvm-mca -mtriple=x86_64 -mcpu=znver4

# Analyse the inner loop only (mark it in source with inline asm)
clang -O2 -S -o kernel.s kernel.c
# Then add LLVM-MCA-BEGIN/END markers around the loop body and run:
llvm-mca -mtriple=x86_64 -mcpu=znver4 --bottleneck-analysis kernel.s
```

You can also insert region markers directly into C source using inline assembly:

```cpp
void hot_kernel(float* __restrict__ a, float* __restrict__ b, int n) {
    for (int i = 0; i < n; i += 8) {
        __asm volatile("# LLVM-MCA-BEGIN hot-kernel" ::: );
        // ... loop body ...
        __asm volatile("# LLVM-MCA-END" ::: );
    }
}
```

When the source is compiled with `-S`, the markers appear in the assembly output and `llvm-mca` picks them up.

---

## 199.5 Reading the Output: SummaryView

The SummaryView is printed first and provides the headline metrics. Below is annotated output from a four-instruction AVX2 dot-product kernel on Skylake:

```text
[0] Code Region - dot-product-loop

Iterations:        100
Instructions:      400        # 4 instructions × 100 iterations
Total Cycles:      407        # wall-clock cycles in the simulation
Total uOps:        400        # total µops dispatched

Dispatch Width:    6          # from SKL SchedMachineModel
uOps Per Cycle:    0.98       # Total uOps / Total Cycles
IPC:               0.98       # Instructions / Total Cycles
Block RThroughput: 2.0        # theoretical minimum cycles/iteration
```

**IPC (Instructions Per Cycle)**: the simulated throughput. Compare against the theoretical `Block RThroughput` to see how far the kernel is from the resource limit.

**Block RThroughput**: the reciprocal throughput of the entire block — the theoretical minimum cycles per iteration if all resources were perfectly utilized and no dependencies existed. It equals `max(sum_of_resource_demand / resource_capacity)` across all `ProcResourceUnits`. A kernel is resource-bound when `IPC ≈ Instructions / BlockRThroughput`. A kernel where simulated total cycles significantly exceed `Iterations × BlockRThroughput` is latency-bound or back-pressure-bound.

In the example above: `Block RThroughput = 2.0` means the resource ceiling is one iteration per 2 cycles. The simulated IPC is `0.98`, and `Total Cycles = 407 ≈ 100 × 4.07` cycles per iteration — roughly twice the resource ceiling. The bottleneck is not resource pressure but the data dependency chain through `vaddps` consuming the output of `vmulps` (latency = 4 cycles).

Three performance regimes:

| Regime | Condition | Action |
|---|---|---|
| **Resource-bound** | IPC ≈ Instructions / BlockRThroughput | Saturating the bottleneck port; restructure to use other ports or widen vectors |
| **Latency-bound** | IPC << Instructions / BlockRThroughput; bottleneck analysis shows dependency | Unroll and interleave independent chains to hide latency |
| **Back-pressure-bound** | Dispatch stalls with ROB full; `--dispatch-stats` shows frequent full ROB | Increase ROB depth (`--dispatch`), reduce µop count per instruction, or reduce register pressure |

In practice the majority of carefully written vector kernels fall into the resource-bound regime — they have been optimized to remove dependency chains but then hit a port capacity ceiling. `llvm-mca` is most useful at this stage to identify exactly which port is saturated and by how many µops per iteration, enabling precision restructuring.

**uOps Per Cycle**: counts µops rather than macro-instructions. When an instruction fuses (e.g., a `test`+`jne` macro-fuses into one µop on modern x86) or cracks (e.g., a wide `ld1` on AArch64 expands to multiple µops), `uOps Per Cycle` and IPC diverge. For Skylake, the dispatch width is 6 µops/cycle; the maximum `uOps Per Cycle` is therefore 6.0. If `Total uOps > Instructions`, the kernel contains cracking instructions. If `Total uOps < Instructions`, macro-fusion occurred (possible with test+branch pairs on x86). Both affect how many iterations fit within the ROB and dispatch budget, making `Total uOps` an important figure when sizing unroll factors.

**Dispatch Width** reflects `MCSchedModel::IssueWidth` from TableGen. It can be overridden with `--dispatch=N` to model a narrower front-end (e.g., to simulate in-order or restricted dispatch mode) without changing the execution engine resource model. This is useful for studying the sensitivity of a kernel's throughput to front-end width.

---

## 199.6 TimelineView

Enable with `--timeline`. The timeline shows the per-instruction lifecycle across cycles: **D** = dispatched (entered the ROB and scheduler), **e** = executing (occupying a functional unit), **E** = last execution cycle (writeback), **R** = retired. Dots indicate subsequent cycles in which the instruction is idle.

Below is an annotated timeline from a latency-bound multiply-add-multiply dependency chain on Skylake (`imulq → addq → imulq`):

```text
Timeline view:
                    012
Index     0123456789

[0,0]     DeeeER    . .   imulq  %rax, %rbx    # dispatched cycle 0, executes 1-3, retires 4
[0,1]     D===eER   . .   addq   %rbx, %rcx    # waits (===) for %rbx; executes cycle 4
[0,2]     D====eeeER. .   imulq  %rcx, %rdx    # waits for %rcx from addq; executes 5-7

[1,0]     D===eeeE-R. .   imulq  %rax, %rbx    # iteration 1: waits behind iter-0 imulq
[1,1]     D======eER. .   addq   %rbx, %rcx
[1,2]     D=======eeeER   imulq  %rcx, %rdx
```

Reading the notation:

- `D` at column 0 means the instruction dispatched at cycle 0.
- `===` after `D` are stall cycles in the scheduler's reservation station (waiting for operands).
- `eee` are execution cycles (three cycles for `imulq`, latency 3 on Skylake).
- `E` marks the writeback cycle.
- `-` between `E` and `R` means the instruction completed but cannot yet retire because an older instruction in the ROB is still executing (in-order retirement constraint).
- `R` is the retire cycle.

The cascade of `===` stalls across the dependency chain makes the latency bottleneck immediately visible: the loop-carried chain is `imulq (latency 3) → addq (latency 1) → imulq (latency 3)` = 7 cycles minimum per iteration, confirmed by the timeline expanding across 10 cycles per iteration pair.

Average wait time statistics follow:

```text
Average Wait times:
[0]: Executions  [1]: Avg wait in scheduler queue
[2]: Avg wait while ready  [3]: Avg WB-to-retire delay

      [0]    [1]    [2]    [3]
0.     2     2.5    0.5    0.5    imulq  %rax, %rbx
1.     2     5.5    0.0    0.0    addq   %rbx, %rcx
2.     2     6.5    0.0    0.0    imulq  %rcx, %rdx
```

Column `[2]` (avg time ready but waiting) of 0.0 for instructions 1 and 2 confirms they are never stalled by resource pressure — they are always waiting on data, never on a free port.

---

## 199.7 ResourcePressureView

The ResourcePressureView shows per-port utilization across the simulated iterations. It is enabled by default and appears as two tables: resource pressure per iteration and resource pressure per instruction.

For the AVX2 dot-product kernel (`vmulps`, `vaddps`, `vmulps`, `vaddps`) on Skylake:

```text
Resources:
[0] - SKLDivider    [1] - SKLFPDivider
[2] - SKLPort0      [3] - SKLPort1
[4] - SKLPort2      [5] - SKLPort3
[6] - SKLPort4      [7] - SKLPort5
[8] - SKLPort6      [9] - SKLPort7

Resource pressure per iteration:
[0]   [1]   [2]   [3]   [4]   [5]   [6]   [7]   [8]   [9]
 -     -    2.00  2.00   -     -     -     -     -     -

Resource pressure by instruction:
[0]   [1]   [2]   [3]   [4]   [5]   [6]   [7]   [8]   [9]   Instructions:
 -     -     -    1.00   -     -     -     -     -     -     vmulps %ymm0, %ymm1, %ymm2
 -     -    0.01  0.99   -     -     -     -     -     -     vaddps %ymm2, %ymm3, %ymm3
 -     -    1.00   -     -     -     -     -     -     -     vmulps %ymm4, %ymm5, %ymm6
 -     -    0.99  0.01   -     -     -     -     -     -     vaddps %ymm6, %ymm7, %ymm7
```

The `2.00` for both `SKLPort0` and `SKLPort1` means each port is occupied 2 µop-cycles per iteration. The block RThroughput of `2.0` follows: two ports each at 100% utilization, providing a combined throughput of 2 µops per cycle from those ports against 4 µops per iteration, giving a minimum of 2 cycles per iteration.

Fractional values in the per-instruction table (e.g., `0.99` / `0.01` for `vaddps`) arise because the resource manager dynamically load-balances across a `ProcResGroup`. Over many iterations it averages out, but individual iterations see slight variation — reflected in the fractional pressures.

A **port at or near `1.0` per-iteration pressure** is your bottleneck port. When `Block RThroughput` matches the occupancy of a single port, that port is the saturated resource limiting throughput.

---

## 199.8 Instruction Tables View

The `--instruction-tables` flag (or `--instruction-tables=full`) prints a static per-instruction table derived directly from the `MCSchedModel` without running the pipeline simulation. It is the fastest way to look up scheduling properties for a set of instructions on a target:

```bash
llvm-mca -mtriple=x86_64 -mcpu=skylake --instruction-tables=full simple_add.s
```

```text
Resources:
[0]  - SKLDivider:1
[1]  - SKLFPDivider:1
[2]  - SKLPort0:1
[3]  - SKLPort1:1
...
[21] - SKLPort0156:4  SKLPort0, SKLPort1, SKLPort5, SKLPort6

Instruction Info:
[1]: #uOps  [2]: Latency  [3]: RThroughput
[4]: MayLoad  [5]: MayStore  [6]: HasSideEffects
[7]: Bypass Latency
[8]: Resources (<Name> | <Name>[ReleaseAtCycle] | <Name>[AcquireAt,ReleaseAt])

[1]  [2]  [3]   [4] [5] [6]  [7]   [8]
 1    1   0.25               1     SKLPort0156[1]   addq %rax, %rbx
 1    1   0.25               1     SKLPort0156[1]   addq %rcx, %rdx
```

The `[8]` Resources column lists which specific `ProcResourceUnits` or `ProcResGroups` the instruction occupies and for how many cycles (`[ReleaseAtCycle]` is the cycle at which the resource is freed relative to issue). For multi-µop instructions, each µop contributes a separate resource entry. This view is particularly useful when auditing a new `InstRW` binding in TableGen: add the instruction to a test file, run `--instruction-tables=full`, and verify that the displayed `Latency`, `RThroughput`, and resource set match the values specified in the `.td` file.

The `--instruction-tables` view does not account for data dependencies or resource contention between instructions — it simply reads the static scheduling model. For interaction effects you need the full pipeline simulation.

---

## 199.9 BottleneckAnalysis

The `--bottleneck-analysis` flag adds a diagnosis section between the summary and instruction info:

```text
Cycles with backend pressure increase [ 55.04% ]
Throughput Bottlenecks:
  Resource Pressure       [ 13.51% ]
  - SKLPort0  [ 13.51% ]
  - SKLPort1  [ 13.51% ]
  Data Dependencies:      [ 41.52% ]
  - Register Dependencies [ 41.52% ]
  - Memory Dependencies   [  0.00% ]

Critical sequence based on the simulation:

              Instruction                           Dependency Information
 +----< 0.    vmulps  %ymm0, %ymm1, %ymm2
 |
 |    < loop carried >
 |
 +----> 0.    vmulps  %ymm0, %ymm1, %ymm2     ## RESOURCE interference: SKLPort1 [28%]
 |      1.    vaddps  %ymm2, %ymm3, %ymm3
 +----> 2.    vmulps  %ymm4, %ymm5, %ymm6     ## RESOURCE interference: SKLPort1 [39%]
 +----> 3.    vaddps  %ymm6, %ymm7, %ymm7     ## REGISTER dependency:  %ymm6
```

**Interpreting the fields:**

- **Cycles with backend pressure increase**: the percentage of simulated cycles where the dispatch stage was stalled by back-pressure from the execution engine (either the ROB was full, a resource was unavailable, or a dependency stall prevented issue).
- **Resource Pressure %**: of those stall cycles, what fraction is attributable to resource contention on specific ports.
- **Data Dependencies %**: what fraction is attributable to register or memory dependency chains.
- **Critical sequence**: the chain of instructions that dominates the simulated cycle count, with annotations identifying whether each edge in the chain is a resource interference (port conflict) or a register dependency (RAW hazard). The `< loop carried >` annotation marks a dependence that spans iterations.

A kernel dominated by **Resource Pressure** is compute-bound; adding `ReadAdvance` or splitting a bottleneck write class across more ports in TableGen (or restructuring the kernel) will help. A kernel dominated by **Data Dependencies** is latency-bound; software pipelining, increased unrolling to overlap independent chains, or using instructions with shorter latency will help.

### 199.9.1 Interpreting Mixed Bottlenecks

The dot-product example shows both resource pressure (13%) and data dependencies (41%): this is a **mixed bottleneck**. The loop has some independent pairs (`vmulps #0` and `vmulps #2` can overlap; similarly for `vaddps #1` and `vaddps #3`) but the dependency `vmulps #0 → vaddps #1` and `vmulps #2 → vaddps #3` imposes a 4-cycle latency each. The resource interference at SKLPort1 arises because both `vmulps` instructions compete for the same port in the same cycle window.

For mixed bottlenecks, unrolling and interleaving independent accumulator chains is the most effective strategy: having 8 or 16 independent `vfmadd231ps` chains hides the 4-cycle FMA latency by keeping 4–8 FMAs in flight simultaneously across ports 0 and 1. The `Block RThroughput` of the unrolled kernel does not change — it remains 0.5 cycles per FMA — but the simulated IPC approaches the resource ceiling because the dependency stalls are eliminated.

### 199.9.2 The Probability Annotation

The `## RESOURCE interference: SKLPort1 [ probability: 39% ]` annotation on the critical sequence means that in 39% of simulated iterations, instruction 2 (`vmulps %zmm4`) was delayed specifically because SKLPort1 was occupied by another instruction. This probability is computed by the bottleneck analysis over all 100 simulated iterations. A high probability (>50%) on a single edge with a specific port is strong evidence that port is the binding constraint for that instruction in that context — distinct from the aggregate resource pressure percentage, which is averaged over all stall cycles.

---

## 199.10 The CustomBehaviour Plugin API

`llvm-mca`'s hardware model is deliberately extensible. The `CustomBehaviour` interface ([`llvm/include/llvm/MCA/CustomBehaviour.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/MCA/CustomBehaviour.h)) allows targets to inject simulation hooks that TableGen's declarative model cannot express.

```cpp
// llvm/include/llvm/MCA/CustomBehaviour.h (simplified)
namespace llvm::mca {

class CustomBehaviour {
public:
  virtual ~CustomBehaviour() = default;

  // Called before an instruction is issued to a resource.
  // Return true to prevent issue (model a stall).
  virtual bool shouldBlockIssue(const InstRef &IR) { return false; }

  // Called after an instruction retires.
  virtual void instructionRetired(const InstRef &IR) {}

  // Called each cycle to update any state the CB maintains.
  virtual void cycleEnd() {}
};

} // namespace llvm::mca
```

The X86 target provides `X86CustomBehaviour` ([`llvm/lib/Target/X86/MCA/X86CustomBehaviour.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/X86/MCA/X86CustomBehaviour.cpp)) which models the MITE (legacy decode)/DSB (µop cache) mode switching penalty and the `LDSTAM` instruction sequencing rules. Targets that implement `CustomBehaviour` register it via the target's `TargetSubtargetInfo` through the `MCA` plugin registry:

```cpp
// In X86Subtarget.cpp (schematic)
std::unique_ptr<mca::CustomBehaviour>
X86Subtarget::getCustomBehaviour(mca::SourceMgr &SrcMgr,
                                 const mca::InstrBuilder &IB) const {
  return std::make_unique<X86CustomBehaviour>(SrcMgr, IB, *this);
}
```

Disable custom behaviour with `--disable-cb` to isolate whether a discrepancy between `llvm-mca` and measurement comes from the base TableGen model or from the `CustomBehaviour` extensions.

---

## 199.11 Validating TableGen Scheduling Models

### 199.11.1 The Validation Workflow

`llvm-mca` provides the most value for target authors when used to validate the scheduling model being written or maintained. The workflow:

1. Write a micro-benchmark kernel in assembly for the instruction(s) under test.
2. Run `llvm-mca` to get simulated IPC and `Block RThroughput`.
3. Measure the same kernel on real hardware with `perf stat -e instructions,cycles`.
4. Compare. Significant divergence (>10%) indicates a model error.
5. Fix the relevant `WriteRes` latency or port assignment in the target's `.td` file.
6. Rebuild LLVM and repeat.

Three categories of model error produce different symptoms:

| Error type | Symptom | Diagnosis |
|---|---|---|
| Wrong `Latency` | Simulated cycles per iter too low in a dependency chain | Use a latency-chain kernel (all insts depend on previous); `perf` shows more cycles |
| Missing port | Simulated `BlockRThroughput` too low (predicts too-fast) | Independent-instruction throughput kernel; `perf` shows fewer cycles |
| Wrong `NumMicroOps` | Dispatch stall appears too early; dispatch width headroom incorrect | Observe dispatch stats; compare per-inst µop count from `--instruction-tables` |

The `llvm-exegesis` tool automates step 3 for individual instruction classes by running micro-benchmarks that isolate latency vs. throughput. After running `llvm-exegesis --mode=latency` and `--mode=throughput` on a target instruction, its output can be compared against the `Latency` and `RThroughput` values in the `--instruction-tables` output of `llvm-mca`. Any discrepancy of more than one cycle should be fixed before the model is considered production-quality.

### 199.11.2 A Concrete Discrepancy and Fix

Suppose a new CPU target `FooCPU` declares its FP multiply to dispatch to port `FooPort1`:

```tablegen
// FooSchedule.td — INCORRECT initial model
def FooPort1 : ProcResource<1>;    // FP multiply/add

def FooWriteFMUL : SchedWriteRes<[FooPort1]> {
  let Latency = 4;
  let NumMicroOps = 1;
}
def : InstRW<[FooWriteFMUL], (instregex "VMULPS.*")>;
```

`llvm-mca` predicts `Block RThroughput = 1.0` for a loop with two independent `vmulps` instructions:

```text
Block RThroughput: 1.0
```

But `perf stat` on the real CPU shows the loop runs at 0.5 cycles per iteration — indicating the CPU has two FP multiply ports. The fix adds a second port and groups them:

```tablegen
// FooSchedule.td — CORRECTED model
def FooPort1 : ProcResource<1>;    // FP multiply/add
def FooPort2 : ProcResource<1>;    // FP multiply/add (second pipe)
def FooFPMulGroup : ProcResGroup<[FooPort1, FooPort2]>;

def FooWriteFMUL : SchedWriteRes<[FooFPMulGroup]> {
  let Latency = 4;
  let NumMicroOps = 1;
}
```

After rebuilding, `llvm-mca` now predicts `Block RThroughput = 0.5`, matching hardware.

### 199.11.3 Comparing Latency

Latency errors are trickier to detect with `Block RThroughput` alone because RThroughput is resource-limited, not latency-limited, for throughput kernels. Use a **latency chain kernel** — a sequence of dependent instructions — to expose latency:

```asm
# LLVM-MCA-BEGIN latency-chain
  vmulss %xmm0, %xmm0, %xmm0    # xmm0 → xmm0 (dependent chain)
  vmulss %xmm0, %xmm0, %xmm0
  vmulss %xmm0, %xmm0, %xmm0
  vmulss %xmm0, %xmm0, %xmm0
# LLVM-MCA-END
```

In this four-instruction chain with no independent work, the simulated cycles per iteration equal `4 × Latency(vmulss)`. If `perf stat` gives 16 cycles per iteration and `llvm-mca` gives 12, the model has `Latency = 3` but hardware uses `Latency = 4`.

The fix in TableGen is straightforward — adjust the `Latency` field in the appropriate `SchedWriteRes`:

```tablegen
// Before (incorrect):
def FooWriteFMUL : SchedWriteRes<[FooFPMulGroup]> {
  let Latency = 3;         // wrong — hardware is 4 cycles
  let NumMicroOps = 1;
}

// After (corrected):
def FooWriteFMUL : SchedWriteRes<[FooFPMulGroup]> {
  let Latency = 4;         // matches perf measurement
  let NumMicroOps = 1;
}
```

After applying the fix and rebuilding, `llvm-mca` on the latency-chain kernel will show `Total Cycles ≈ 400` (100 iterations × 4 insts × 1-cycle latency chain depth = effectively 400 execute cycles), matching `perf stat`.

A `ReadAdvance` value can further refine the model by modelling forwarding paths. For example, if a target CPU can forward the output of an FP multiply to an FP add after only 3 cycles (despite 4-cycle total latency), the `ReadAdvance` reduces the effective latency from the perspective of the consumer:

```tablegen
def : ReadAdvance<ReadFPMul, 1>;  // consumer reads 1 cycle early via bypass
```

This models the real hardware's bypass network and produces more accurate simulated IPC for mixed multiply-accumulate chains.

---

## 199.12 Performance Case Studies

### 199.12.1 AVX-512 Horizontal Reduction — Port 5 Shuffle Pressure

A horizontal sum of a `zmm` register requires shuffle instructions. On Skylake-X (`skylake-avx512` / `-mcpu=skx`), the shuffle `vpermd` dispatches exclusively to `SKXPort5`, a single-width port. A naive reduction:

```asm
# LLVM-MCA-BEGIN avx512-horz-reduction
  vmulps  %zmm0, %zmm1, %zmm2           # FP multiply
  vaddps  %zmm2, %zmm3, %zmm3           # accumulate
  vpermd  %zmm3, %zmm4, %zmm5           # permute for reduction
  vaddps  %zmm5, %zmm3, %zmm3           # add permuted lanes
  vextractf64x4  $1, %zmm3, %ymm6       # extract upper 256 bits
  vaddps  %ymm6, %ymm3, %ymm3           # final add
# LLVM-MCA-END
```

Running on Skylake-X:

```bash
llvm-mca -mtriple=x86_64 -mcpu=skylake-avx512 --bottleneck-analysis avx512.s
```

```text
Block RThroughput: 2.0
IPC:               0.33

Throughput Bottlenecks:
  Resource Pressure       [  0.00% ]
  Data Dependencies:      [ 84.95% ]
  - Register Dependencies [ 84.95% ]
```

The dominant bottleneck is the register dependency chain: `vmulps → vaddps → vpermd → vaddps → vextractf64x4 → vaddps`. Each instruction depends on the previous, creating an 18-cycle latency chain (4 + 4 + 3 + 4 + 3 + 4). The simulated IPC of 0.33 (6 instructions / 18 cycles) confirms this: the kernel is almost purely serialized.

The fix identified by `llvm-mca` analysis is architectural: the horizontal reduction must be hoisted out of the main accumulation loop. Inside the loop, the kernel should perform only `vfmadd231ps` instructions accumulating into independent `zmm` registers. The horizontal sum is then a single post-loop sequence that `llvm-mca` need not optimize — it executes once, not N times. The corrected loop body:

```asm
# LLVM-MCA-BEGIN corrected-fma-loop
  vfmadd231ps %zmm0, %zmm1, %zmm8    # accumulator A
  vfmadd231ps %zmm2, %zmm3, %zmm9    # accumulator B (independent)
  vfmadd231ps %zmm4, %zmm5, %zmm10   # accumulator C
  vfmadd231ps %zmm6, %zmm7, %zmm11   # accumulator D
# LLVM-MCA-END
```

```text
Block RThroughput: 2.0
IPC:               2.0
```

Four independent FMA chains at 0.5 cycles each achieves full port 0/1 saturation: IPC equals `Block RThroughput`, the best achievable.

### 199.12.2 Memory-Bound Scatter Kernel — AGU Bottleneck

A gather-scatter pattern that reads from one array and writes to another using base+index addressing:

```asm
# LLVM-MCA-BEGIN scatter-kernel
  movl  (%rsi,%rax,4), %ecx      # load from src[i]
  movl  %ecx, (%rdi,%rax,4)     # store to dst[i]
  movl  4(%rsi,%rax,4), %ecx    # load from src[i+1]
  movl  %ecx, 4(%rdi,%rax,4)    # store to dst[i+1]
  addq  $2, %rax
  cmpq  %rdx, %rax
# LLVM-MCA-END
```

`llvm-mca` on Skylake:

```text
Block RThroughput: 2.0
IPC:               2.88

Throughput Bottlenecks:
  Resource Pressure       [ 68.75% ]
  - SKLPort2  [ 45.19% ]
  - SKLPort3  [ 45.19% ]
  - SKLPort4  [ 68.75% ]
  - SKLPort7  [ 29.81% ]
```

Ports 2 and 3 are Skylake's load AGUs; port 4 is the store data port; port 7 is the store AGU. Two loads and two stores consume 4 AGU slots — the maximum Skylake can sustain is 2 loads and 1 store per cycle at L1 bandwidth. The kernel is memory-bottlenecked at the AGU even before cache effects are considered.

The `Block RThroughput = 2.0` comes from the maximum port pressure: SKLPort4 (store data) at 2 µops per iteration × 1 unit = 2 cycles. The IPC of 2.88 is actually close to the resource ceiling (6 instructions / 2.0 cycles = 3.0 theoretical), which shows the kernel is near the AGU limit.

Remediation options identified from this analysis:

1. **Widen to `ymm` registers**: replace 4 × `movl` (4 bytes each) with 2 × `vmovdqu` (32 bytes each). Same 4 memory operations, but 8× the data throughput per AGU slot.
2. **Non-temporal stores**: if destination memory is write-once, `vmovntdq` bypasses the L1 store queue, reducing port 4 pressure and allowing the store to complete without occupying a store buffer entry waiting for L1 writeback.
3. **Software prefetching**: the current kernel's `Block RThroughput` assumes L1 hits. On real hardware, `_mm_prefetch` into L2/L3 can be inserted 8–16 iterations ahead to hide the latency penalty from cache misses — a concern invisible to `llvm-mca` but motivating a follow-up `perf stat` run on the real workload.

### 199.12.3 AArch64 NEON Dot-Product Loop

A NEON unsigned dot-product loop using `udot` (ARMv8.2 dot-product extension) on Cortex-A57:

```asm
# LLVM-MCA-BEGIN neon-dot-product
  .arch armv8.2-a+dotprod
  udot  v0.4s, v1.16b, v2.16b    # 4-way dot, accumulate into v0
  udot  v3.4s, v4.16b, v5.16b    # independent accumulator
  add   v0.4s, v0.4s, v3.4s      # merge accumulators
  ld1   { v1.16b, v2.16b }, [x0], #32   # load next 32 bytes
  ld1   { v4.16b, v5.16b }, [x1], #32   # load next 32 bytes
# LLVM-MCA-END
```

```bash
llvm-mca -mtriple=aarch64 -mcpu=cortex-a57 --bottleneck-analysis neon.s
```

```text
Block RThroughput: 4.0
IPC:               0.83

Throughput Bottlenecks:
  Resource Pressure       [ 20.30% ]
  - A57UnitL  [ 20.30% ]
  Data Dependencies:      [ 30.69% ]
  - Register Dependencies [ 30.69% ]
```

Each `ld1 {v.16b, v.16b}` on Cortex-A57 expands to **3 µops** and has a 6-cycle latency, consuming 2 cycles of `A57UnitL` (the load unit). Two such loads per iteration — 4 cycles of load unit pressure — matches `Block RThroughput = 4.0`. The loop is load-bound.

The bottleneck analysis shows `A57UnitL` at 20% resource pressure and register dependencies at 30%, indicating that even if the load bottleneck were resolved (by loading more aggressively), the RAW dependency from `ld1` into `udot` (a 6-cycle load latency on A57 before the dot product can execute) would still constrain throughput. The resource pressure and data dependency combine multiplicatively: the load unit cannot sustain the dual-accumulator compute.

Remediation options:

1. **Multi-iteration unrolling**: unroll 4× the loop body, interleaving four independent `udot` chains. This gives 4 independent pairs of accumulators feeding 8 `udot` operations per 4-load block. Each load feeds a `udot` that is 6 iterations away, reducing RAW stalls from load latency.
2. **`prfm` prefetch**: `prfm PLDL1KEEP, [x0, #128]` before the loads hints the hardware prefetcher and can hide DRAM latency that `llvm-mca` does not model.
3. **Consider `ld4` or `ld2`**: if the input data is interleaved (RGBA pixels, AoS structure), `ld4 {v1.16b-v4.16b}, [x0]` loads four 16-byte vectors simultaneously in 4 µops vs. the 6-µop cost of two separate `ld1` pairs, potentially reducing AGU pressure.

---

## 199.13 CI Integration

### 199.13.1 Extracting Metrics from JSON Output

`llvm-mca --json` emits structured data that is easy to parse in a CI script. The `SummaryView` object contains `BlockRThroughput` and `IPC`:

```bash
#!/usr/bin/env bash
# ci_perf_check.sh — assert kernel RThroughput stays below threshold

THRESHOLD=4.0
RESULT=$(llvm-mca -mtriple=x86_64 -mcpu=znver4 --json kernel.s 2>&1)

RTHROUGHPUT=$(echo "$RESULT" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for region in data['CodeRegions']:
    sv = region.get('SummaryView', {})
    if sv:
        print(sv['BlockRThroughput'])
        break
")

python3 -c "
rt = float('$RTHROUGHPUT')
threshold = float('$THRESHOLD')
if rt > threshold:
    print(f'FAIL: BlockRThroughput {rt} exceeds threshold {threshold}')
    exit(1)
else:
    print(f'PASS: BlockRThroughput {rt} <= threshold {threshold}')
"
```

The JSON `SummaryView` keys available for scripting:

```json
"SummaryView": {
    "BlockRThroughput": 2.0,
    "DispatchWidth":    6,
    "IPC":              3.88,
    "Instructions":     400,
    "Iterations":       100,
    "TotalCycles":      103,
    "TotaluOps":        400,
    "uOpsPerCycle":     3.88
}
```

### 199.13.2 CMake Integration

For LLVM tree development, a CMake custom target can run `llvm-mca` over a set of kernel files and compare against a baseline:

```cmake
# In your target's CMakeLists.txt
add_custom_target(check-mca-kernels
  COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_mca.sh
          ${LLVM_BINARY_DIR}/bin/llvm-mca
          ${CMAKE_CURRENT_SOURCE_DIR}/kernels/
  COMMENT "Running llvm-mca performance regression checks"
  DEPENDS llvm-mca
)
```

### 199.13.3 Regression Detection Pattern

Store a baseline `Block RThroughput` per kernel in a text file. In CI, re-run `llvm-mca --json` and compare:

```python
#!/usr/bin/env python3
# check_regression.py
import json, subprocess, sys

KERNELS = {
    "fma_loop.s":    {"mcpu": "znver4",  "max_rth": 2.0},
    "memcpy_avx.s":  {"mcpu": "skylake", "max_rth": 4.0},
    "neon_dot.s":    {"mcpu": "cortex-a57", "max_rth": 5.0},
}

failures = []
for kernel, spec in KERNELS.items():
    result = subprocess.run(
        ["llvm-mca", f"-mcpu={spec['mcpu']}", "--json", kernel],
        capture_output=True, text=True
    )
    data = json.loads(result.stdout)
    for region in data["CodeRegions"]:
        sv = region.get("SummaryView", {})
        if not sv:
            continue
        rth = sv["BlockRThroughput"]
        if rth > spec["max_rth"]:
            failures.append(
                f"{kernel}: RThroughput {rth} > {spec['max_rth']}"
            )

if failures:
    print("MCA REGRESSIONS DETECTED:")
    for f in failures:
        print(f"  {f}")
    sys.exit(1)
print("All MCA checks passed.")
```

This pattern catches compiler regressions (e.g., a new optimization pass that inadvertently de-vectorizes a loop or introduces a scheduling anomaly) before they reach production.

---

## 199.14 Comparison with Measurement Tools

### 199.14.1 perf stat

`perf stat -e instructions,cycles` measures real hardware counters:

```bash
perf stat -e instructions,cycles,uops_dispatched.thread \
          ./kernel_bench
```

Strengths: ground truth IPC, reliable `cycles` and `instructions` counts. Weaknesses: requires a running binary, no port-level breakdown without PMU-specific events (which vary by microarchitecture and kernel version), no per-instruction attribution.

```bash
# Skylake-specific port events (Linux perf, Intel-specific PMU)
perf stat -e cpu/event=0xa1,umask=0x01,name=port0_uops/ \
          -e cpu/event=0xa1,umask=0x02,name=port1_uops/ \
          ./kernel_bench
```

PMU event availability varies across CPU generations and kernel versions; `llvm-mca` provides consistent port-level data without this complexity. The combination is powerful: use `llvm-mca` for rapid design-time feedback, then confirm with `perf stat` at validation time.

The recommended validation loop for a new kernel is:

1. Prototype in C/assembly, compile with `-O2 -S`, pipe through `llvm-mca --bottleneck-analysis`.
2. Identify the dominant bottleneck (resource or dependency).
3. Restructure the kernel (unroll, rearrange, change instruction selection).
4. Iterate with `llvm-mca` until `IPC ≈ Instructions / BlockRThroughput`.
5. Benchmark the final kernel with `perf stat`. If the `perf` IPC differs from `llvm-mca` IPC by more than 15%, investigate cache behavior (not modelled by `llvm-mca`) or file a TableGen model bug.

This loop typically takes minutes per iteration in `llvm-mca`, vs. hours for a full benchmark cycle.

### 199.14.2 IACA (Intel Architecture Code Analyzer)

IACA was Intel's official static throughput analyzer, available 2012–2019. It was Intel-only (x86 Haswell through Skylake), required a binary instrumented with special start/end markers, and has been discontinued. Its replacement from Intel is not generally available. `llvm-mca` is the natural open-source successor, supporting all LLVM targets.

### 199.14.3 uiCA

uiCA (µop Cache Analyser) is an academic tool from TU Munich that reverse-engineers Skylake and Zen4 port assignments from measurement data. It is more accurate than `llvm-mca` for those specific targets because it captures effects not modelled in TableGen (µop cache effects, partial register writes). It is x86-only and not integrated into any build system. Use `uiCA` when maximum accuracy on Skylake/Zen4 is critical and the gap between `llvm-mca` prediction and measurement is under investigation.

### 199.14.4 llvm-exegesis

`llvm-exegesis` (discussed in [Ch174 — Performance Engineering](../part-25-operations-contribution/ch174-performance-engineering.md)) micro-benchmarks individual instructions on real hardware to measure their latency, reciprocal throughput, and port usage. Its output is directly consumable by the workflow in Section 199.11: run `llvm-exegesis` to measure a target instruction, then update the TableGen scheduling model to match, then re-run `llvm-mca` to verify the model predicts kernel-level throughput correctly. This is the standard calibration loop for LLVM target maintainers.

### 199.14.5 gem5

gem5 is a full-system cycle-accurate simulator supporting multiple ISAs. It models caches, DRAM, the OS, and the complete processor pipeline. It is appropriate for architecture exploration (new cache hierarchies, prefetcher designs, memory access pattern studies) but far too slow for hot-loop analysis: simulating one second of real time can take hours in gem5. `llvm-mca` simulates 100 iterations of a 10-instruction loop in under a millisecond.

---

## Chapter Summary

- `llvm-mca` is LLVM's static throughput simulator. It models out-of-order execution using TableGen `SchedMachineModel` data — the same data the instruction scheduler uses during compilation — without executing code.

- The tool's pipeline consists of `EntryStage`, `DispatchStage`, `ExecuteStage`, and `RetireStage`, with `ResourceManager` managing `ProcResourceUnit` and `ProcResGroup` availability. Views implement `HWEventListener` to collect statistics.

- `InstrBuilder` converts `MCInst` → `mca::Instruction` by querying `MCSubtargetInfo` for scheduling class data. Accuracy is bounded by TableGen model accuracy.

- Region markers `# LLVM-MCA-BEGIN` / `# LLVM-MCA-END` isolate specific code regions in a larger assembly file. Compiler output can be piped directly: `clang -O2 -S -o - foo.c | llvm-mca -mcpu=skylake`.

- **SummaryView** provides IPC, `Block RThroughput` (the resource-ceiling cycles per iteration), dispatch width, and µop counts. A gap between IPC and `Instructions / BlockRThroughput` indicates latency or back-pressure bottlenecks.

- **TimelineView** (`--timeline`) shows the dispatch (D), execute (e/E), and retire (R) cycle for each instruction, with `=` marking scheduler stalls. It makes dependency chains and ROB pressure visible cycle by cycle.

- **ResourcePressureView** shows per-port µop occupancy. A port at or near 1.0 µop per iteration with `BlockRThroughput` matching its reciprocal is the saturated bottleneck.

- **BottleneckAnalysis** (`--bottleneck-analysis`) classifies stall cycles as resource pressure vs. register/memory dependency, traces the critical instruction sequence, and annotates each edge with the specific port or register causing the delay.

- `CustomBehaviour` plugins allow targets to model micro-architectural effects beyond TableGen's declarative scope (e.g., MITE/DSB switching on x86, instruction ordering constraints).

- The model validation workflow pairs `llvm-mca` predictions against `perf stat` measurements. Discrepancies expose incorrect `WriteRes` latencies, missing port assignments, or wrong µop counts in the target's `.td` files. `llvm-exegesis` provides the per-instruction ground-truth measurements for calibration.

- `--json` output enables CI integration: extract `SummaryView.BlockRThroughput`, compare against a per-kernel threshold, and fail the build when a regression is detected.

- `llvm-mca` complements rather than replaces hardware measurement. Use it for fast iteration during development and code review; use `perf stat` and `llvm-exegesis` for ground-truth calibration and production validation.
