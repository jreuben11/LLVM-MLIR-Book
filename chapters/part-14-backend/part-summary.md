---
# Part XIV — Backend — Part Summary

*This part covers the complete LLVM backend pipeline from LLVM IR through instruction selection, register allocation, post-allocation optimization, and final machine-code emission, equipping readers to write, debug, and optimize backend passes for any target architecture.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 81 | Backend Architecture | Pipeline overview, type systems, SelectionDAG vs GlobalISel |
| 82 | TableGen Deep Dive | Language syntax, multiclass/defm, all `-gen-*` backends |
| 83 | The Target Description | TargetMachine, TargetLowering, register classes, scheduling model |
| 84 | SelectionDAG: Building and Legalizing | IR-to-DAG translation, type and operation legalization |
| 85 | SelectionDAG: Combining and Selecting | DAGCombiner peephole passes, pattern-based instruction selection |
| 86 | GlobalISel | IRTranslator, Legalizer, RegBankSelect, InstructionSelect |
| 87 | Inline Assembly Lowering | Constraint parsing, INLINEASM node, clobbers, asm goto |
| 88 | The Machine IR | MachineFunction, MachineInstr, MachineOperand, MIR text format |
| 89 | Pre-RegAlloc Passes | TwoAddress, PHIElimination, LiveIntervals, Coalescer, Scheduler |
| 90 | Register Allocation | Greedy, Basic, Fast, PBQP allocators; spilling and splitting |
| 91 | The Machine Pipeliner | Swing Modulo Scheduling, rotating register files, VLIW bundling |
| 92 | The Machine Outliner | Suffix-tree candidate detection, cost model, IROutliner comparison |
| 93 | Post-RegAlloc and Pre-Emit | PrologueEpilogueInserter, BranchFolding, IfConversion, PostRAScheduler |
| 94 | The MC Layer and MIR Test Infrastructure | MCInst, MCStreamer, MCCodeEmitter, fixups, lit test infrastructure |

## Part Overview

Part XIV is the definitive treatment of what happens between LLVM IR and object code. The part opens by establishing the overall pipeline architecture (Chapter 81) and its three distinct type systems — LLVM IR types, MVT/EVT for SelectionDAG, and LLT for GlobalISel — before surveying the two coexisting instruction selection frameworks. The reader immediately sees that the backend is not a single pass but a layered sequence of progressively more concrete transformations, each governed by properties that the pass manager checks and propagates.

Chapters 82 and 83 address the declarative infrastructure that underpins every target backend: TableGen and the target description classes. TableGen is LLVM's domain-specific language for encoding instruction sets, register hierarchies, calling conventions, scheduling models, and selection patterns as machine-readable `.td` records, from which `llvm-tblgen` generates thousands of lines of C++ include files. The target description classes — `TargetMachine`, `TargetSubtargetInfo`, and `TargetLowering` — connect those generated tables to the backend passes, defining what is legal, what must be expanded, and how calls are lowered. Understanding these two chapters is a prerequisite for every subsequent chapter that touches target-specific behavior.

Chapters 84 through 86 form the instruction selection core. SelectionDAG (Chapters 84–85) builds a directed acyclic graph from IR, runs three interleaved passes of the DAGCombiner peephole optimizer (constant folding, strength reduction, load-store forwarding) and two legalization phases (type legalization via SelectionDAGLegalizeTypes, then operation legalization via SelectionDAGLegalize), and finally pattern-matches the legal DAG against TableGen-defined patterns via the generated `SelectCode` decision tree. GlobalISel (Chapter 86) replaces this with a linear MachineFunction representation using generic `G_*` opcodes, a four-stage pipeline (IRTranslator, Legalizer, RegBankSelect, InstructionSelect), the `LLT` type system, the `LegalizerInfo` API, and the `GICombineRule` TableGen DSL for writing optimization rules. As of LLVM 22, both frameworks coexist: GlobalISel is the default for AArch64 and RISC-V; SelectionDAG remains active for legacy targets and as a fallback.

Chapters 87 through 90 cover the transformation of selected machine instructions into register-allocated physical-register code. Inline assembly (Chapter 87) follows its own lowering path through constraint parsing, the `ISD::INLINEASM` DAG node, the `INLINEASM` MachineInstr, clobber handling, and final emission via the integrated assembler. Chapter 88 provides a thorough reference to the Machine IR data structures — `MachineFunction`, `MachineBasicBlock`, `MachineInstr`, `MachineOperand`, `MachineMemOperand`, and the `.mir` text format — and the `MachineVerifier` invariants that enforce correctness across the entire backend. Chapter 89 traces the preparatory pre-allocation passes: `TwoAddressInstructionPass`, `PHIElimination`, `SlotIndexes`, `LiveIntervals`, `RegisterCoalescer`, `MachineCombiner`, and the pre-RA `MachineScheduler`. Chapter 90 presents register allocation in depth: the greedy allocator's priority queue, eviction cascade, and live range splitting via `SplitKit`; the fast `-O0` allocator; the PBQP quadratic-programming allocator; and spill placement optimization via min-cut on the CFG.

The final four chapters address advanced optimizations and the output path. The Machine Pipeliner (Chapter 91) implements Swing Modulo Scheduling for in-order and VLIW targets, computing minimum initiation intervals from resource and recurrence constraints. The Machine Outliner (Chapter 92) uses Ukkonen's suffix tree algorithm to detect repeated instruction sequences globally across all functions and replace them with calls, introducing the complementary IR-level `IROutliner` that operates on structural IR similarity. Post-allocation passes (Chapter 93) concretize the stack frame, eliminate redundant branches, duplicate small tails, convert predicated control flow, reschedule with full physical-register alias knowledge, and split cold code. The MC layer (Chapter 94) completes the journey: `MCInst`, `MCStreamer`, `MCCodeEmitter`, fixups, relocations, `MCAssembler`, and `MCObjectWriter` produce the final ELF/COFF/MachO artifact, while the `.mir` text format and `lit`/`FileCheck` infrastructure enable pass-level regression testing in isolation.

A reader who completes this part will be able to: add a SelectionDAG or GlobalISel pattern for a new instruction; write a custom `TargetLowering::LowerOperation` handler; author a `GICombineRule` combiner; set up `LegalizerInfo` tables for a new target; read and write `.mir` regression tests; understand the greedy allocator's allocation, eviction, and splitting decisions; diagnose codegen failures using `-verify-machineinstrs`, `-view-dag-combine*`, and `llc --stop-after`; and trace an object file's binary bytes back to the LLVM IR that produced them.

## Key Concepts Introduced

- **SelectionDAG** — a directed acyclic graph of typed `SDNode` objects representing one basic block's IR at the machine level; the substrate for legalization, combining, and pattern-based instruction selection.
- **MVT / EVT** — the Machine Value Type system used by SelectionDAG; a compact `uint16_t`-encoded type representation for scalars, vectors, and pointers.
- **LLT (Low-Level Type)** — GlobalISel's type system encoding scalar width, vector shape, and pointer address space without distinguishing integer from float.
- **GlobalISel / G_* opcodes** — the modern instruction selection framework operating on a linear MachineFunction with generic opcodes, enabling cross-block analysis and avoiding DAG construction overhead.
- **LegalizerInfo** — the table mapping `(opcode, LLT)` pairs to legalization actions (`Legal`, `WidenScalar`, `FewerElements`, `Lower`, `Libcall`, `Custom`) used by GlobalISel's Legalizer pass.
- **TableGen / `llvm-tblgen`** — LLVM's domain-specific language for declaring registers, instructions, calling conventions, scheduling models, and selection patterns; `multiclass`/`defm` and `!`-operators enable factored, parameterized hardware descriptions.
- **TargetLowering** — the primary hook class connecting SelectionDAG legalization to the target; `getOperationAction` returns the action for each `(ISD opcode, MVT)` pair; `LowerOperation` implements custom lowering.
- **MachineFunction / MachineInstr / MachineOperand** — the three-level MIR hierarchy: a function containing basic blocks containing instructions containing typed operands (registers, immediates, frame indices, symbols, etc.).
- **LiveIntervals** — the central pre-RA analysis computing `[start, end)` slot-index ranges for every virtual register; provides the interference information on which all register allocators depend.
- **RAGreedy (greedy allocator)** — LLVM's default `-O1+` register allocator; processes intervals by spill weight, attempts assignment, then eviction (with cascade prevention), then live range splitting (local or region-based via `SplitKit`), and finally spilling.
- **Spill weights** — a per-virtual-register priority score combining use frequency (from `BlockFrequencyInfo`), type cost, and interval length; determines allocation order and eviction eligibility.
- **Register coalescing** — the elimination of copy instructions by merging non-interfering live intervals; runs as both a pre-RA pass (`RegisterCoalescer`) and implicitly during allocation.
- **Swing Modulo Scheduling (SMS)** — a loop-pipelining algorithm that overlaps successive iterations with initiation interval II = max(ResMII, RecMII); LLVM's `MachinePipeliner` implements SMS via SCC-based node ordering.
- **Machine Outliner / IROutliner** — code-size optimization passes that replace repeated instruction or IR sequences with calls to a shared outlined function, detected via a suffix tree over tokenized instruction streams.
- **PrologueEpilogueInserter** — the post-RA pass that determines callee-saved register usage, computes the concrete stack frame layout, generates prologue and epilogue code, and eliminates abstract frame indices.
- **MC Layer (MCInst / MCStreamer / MCCodeEmitter)** — the final abstraction boundary between CodeGen and output: `MCInst` holds only physical registers and symbol expressions; `MCStreamer` provides a format-neutral emission API; `MCCodeEmitter` encodes instructions to bytes and produces fixups for unresolvable references.
- **MIR text format / `.mir` files** — LLVM's YAML-based serialization of `MachineFunction` state, enabling `llc --stop-after=<pass>` / `--run-pass=<pass>` workflows for isolated pass testing and debugging.

## How This Part Fits the Book

Part XIV builds directly on the LLVM IR foundation established in Part IV (Chapters 27–36), which defines the abstract instruction set that `SelectionDAGBuilder` and `IRTranslator` consume as their starting point, and on the middle-end analyses from Part X (Chapters 57–68), whose results (`BlockFrequencyInfo`, `AliasAnalysis`, profile data) feed the backend's cost models, scheduler heuristics, and spill weight calculations. Part XV — Targets (Chapters 95–107) extends this part by applying the backend infrastructure to specific ISA families (x86, AArch64, RISC-V, AMDGPU, and others), showing how each target populates its `TargetLowering`, TableGen descriptions, and scheduling models; readers should master Part XIV before tackling the per-target chapters.

## Cross-Part Dependencies

- Ch 27–36 (Part IV — LLVM IR): SelectionDAGBuilder and IRTranslator consume LLVM IR; every concept in this part presupposes familiarity with SSA, IR types, and IR instruction semantics.
- Ch 57–68 (Part X — Analysis and Middle End): `BlockFrequencyInfo` feeds spill weight computation (Ch 90); `AliasAnalysis` feeds `MachineMemOperand` and DAGCombiner memory-operation combining (Ch 84–85); profile data feeds the Machine Outliner's profitability model (Ch 92) and MachineFunctionSplitter (Ch 93).
- Ch 69–74 (Part XII — Polly): Polly's polyhedral dependence information improves `RecMII` bounds for the Machine Pipeliner (Ch 91); the Polly-LLVM boundary is where affine loop analysis hands off to modulo scheduling.
- Ch 75–80 (Part XIII — LTO and Whole-Program Analysis): ThinLTO module pass ordering is where the Machine Outliner and IROutliner gain cross-TU visibility (Ch 92); LTO also exercises the `TargetMachine` subtarget-sharing and per-function subtarget mechanisms (Ch 83).
- Ch 95–107 (Part XV — Targets): Every chapter in Part XV is a direct application of this part's infrastructure — each target chapter explains how AArch64, x86, RISC-V, etc. instantiate `TargetLowering`, `SchedMachineModel`, `LegalizerInfo`, and `TableGen` records described here.
- Ch 108–112 (Part XVI — JIT and Sanitizers): The ORC JIT uses `MCCodeEmitter` and `MCObjectWriter` for in-process code emission (Ch 94); sanitizer instrumentation hooks into the backend pass pipeline using the `MachineFunctionPass` infrastructure introduced in Ch 81.
- Ch 143–148 (Part XIX — MLIR Foundations): The MLIR-to-LLVM-IR lowering pipeline terminates at the LLVM IR level and then re-enters this part's backend; the discussion of MLIR backend dialects and MIR-as-MLIR proposals references the MachineFunction structures from Ch 88.
- Ch 196 (Part XXIII — MLIR Production): The LLVM IR interchange layer discussion in Ch 196 directly references the SelectionDAG / GlobalISel duality (Ch 81, 84–86) as the point where MLIR-lowered IR enters the physical code generation pipeline.

## Navigation

- ← Part XIII — LTO & Whole-Program Analysis
- → Part XV — Targets

---

*@copyright jreuben11*
