# Chapter 81 — Backend Architecture

*Part XIV — The Backend*

The LLVM backend translates LLVM IR into machine code through a pipeline of machine-level passes. Where the middle end (Part X) operates on SSA values and abstract types, the backend must grapple with real hardware constraints: a fixed set of registers with specific types and sizes, instruction sets where not every operation is directly supported for every type, calling conventions that dictate exactly how arguments are passed, and binary encodings that must be precise to the bit. This chapter describes the backend pipeline from IR input to object code output, the three type systems the backend uses (IR types, LLT, and MVT/EVT), and the coexistence of the two instruction selection frameworks (SelectionDAG and GlobalISel).

## 81.1 The Backend Pipeline

### 81.1.1 Overview

The LLVM backend is a sequence of `MachineFunctionPass` instances that transform a function from LLVM IR through a series of progressively more concrete representations:

```
LLVM IR (Module/Function)
    │
    ├── [SelectionDAG / GlobalISel]  — instruction selection
    │       → MachineFunction (virtual registers, pseudo instructions)
    │
    ├── Pre-RegAlloc passes
    │   ├── TwoAddressInstructionPass  — expand 2-addr instructions
    │   ├── PHI elimination           — lower phi nodes to copies
    │   ├── LiveIntervals             — compute live ranges
    │   ├── RegisterCoalescer         — merge copy-related live ranges
    │   └── MachineScheduler          — pre-RA instruction scheduling
    │
    ├── Register Allocation (Greedy / Basic / Fast / PBQP)
    │   → virtual registers → physical registers + stack slots
    │
    ├── Post-RegAlloc passes
    │   ├── PrologueEpilogueInserter  — stack frame + callee-saved regs
    │   ├── BranchFolding / TailDuplication / IfConversion
    │   ├── PostRAScheduler           — post-RA instruction scheduling
    │   └── MachineFunctionSplitter   — split hot/cold
    │
    ├── MC Layer
    │   ├── AsmPrinter                — MachineInstr → MCInst → asm/obj
    │   └── MCObjectWriter            — write ELF/COFF/MachO
    │
    └── Object file / Assembly
```

The pipeline is registered in `TargetPassConfig` (in `llvm/CodeGen/TargetPassConfig.h`) via `addISelPasses`, `addPreRegAlloc`, `addRegAllocPasses`, `addPostRegAlloc`, and `addPreEmitPasses`.

### 81.1.2 Target Machine and Pass Config

Each target provides a `TargetMachine` subclass that configures the pipeline:

```cpp
// Example: X86TargetMachine (simplified)
class X86TargetMachine : public LLVMTargetMachine {
public:
  TargetPassConfig *createPassConfig(PassManagerBase &PM) override {
    return new X86PassConfig(*this, PM);
  }
};

class X86PassConfig : public TargetPassConfig {
public:
  bool addInstSelector() override {
    addPass(createX86ISelDag(*getX86TargetMachine(), getOptLevel()));
    return false;
  }
  void addPreRegAlloc() override {
    addPass(createX86CmovConverterPass());
    addPass(createX86WinAllocaExpander());
  }
};
```

The `TargetPassConfig::addMachineSSAOptimization()` adds target-independent pre-RA passes; individual targets can override any hook to insert or remove passes.

## 81.2 Type Systems in the Backend

The LLVM backend uses three distinct type systems at different pipeline stages. Understanding when each is used is essential for backend development.

### 81.2.1 LLVM IR Types

At the IR level (before instruction selection), LLVM uses the standard IR type system: `i32`, `i64`, `float`, `double`, `<4 x float>`, `ptr`, etc. These are architecture-independent abstract types.

The IR type system disappears at the boundary of instruction selection: SelectionDAG and GlobalISel convert IR types to their respective machine type representations.

### 81.2.2 MVT: Machine Value Type

`MVT` (Machine Value Type, in `llvm/CodeGen/MachineValueType.h`) is the type system used by **SelectionDAG**. MVT is a compact representation optimized for DAG node type annotation:

```cpp
// Key MVT values:
MVT::i8, MVT::i16, MVT::i32, MVT::i64     // integer
MVT::f32, MVT::f64, MVT::f128             // floating-point
MVT::v4i32, MVT::v8f32, MVT::v16i8        // fixed-width SIMD
MVT::nxv4i32, MVT::nxv8f32                // scalable SIMD (SVE/RVV)
MVT::Untyped, MVT::Glue, MVT::Other        // internal DAG types
MVT::iPTR                                  // pointer (size from DataLayout)
```

MVT values are small integers (encoded as `uint16_t`) for cache efficiency. The `EVT` (Extended Value Type) is a wrapper that can hold either an MVT or an LLVM IR Type (for types that don't fit in MVT, e.g., `i1024`):

```cpp
EVT VT = EVT::getIntegerVT(Ctx, 1024);  // EVT wrapping i1024 IR type
EVT MVT32 = EVT(MVT::i32);              // EVT wrapping MVT
bool isSimple = VT.isSimple();           // false for i1024, true for i32
```

### 81.2.3 LLT: Low-Level Type

`LLT` (Low-Level Type, in `llvm/CodeGen/LowLevelType.h`) is the type system used by **GlobalISel**. LLT is more expressive than MVT, more compact than LLVM IR types, and designed for the legalizer:

```cpp
// LLT construction:
LLT s32 = LLT::scalar(32);             // 32-bit scalar integer or float
LLT p0 = LLT::pointer(0, 64);          // 64-bit pointer (address space 0)
LLT v4s32 = LLT::fixed_vector(4, s32); // <4 x i32> fixed vector
LLT nxv4s32 = LLT::scalable_vector(4, s32); // <vscale x 4 x i32>

// LLT queries:
s32.isScalar()       → true
s32.getSizeInBits()  → 32
v4s32.isVector()     → true
v4s32.getElementType() → s32
v4s32.getNumElements() → 4
p0.isPointer()       → true
p0.getAddressSpace() → 0
```

Unlike MVT, LLT does not distinguish integer from floating-point — it's purely a size/shape descriptor. The type-specific semantics are encoded in the opcode (`G_ADD` for integer add, `G_FADD` for float add).

### 81.2.4 Type System Transition Points

| Pipeline Stage | Type System |
|---------------|------------|
| LLVM IR → SelectionDAG | IR Types → MVT/EVT |
| LLVM IR → GlobalISel | IR Types → LLT |
| SelectionDAG (post-legalize) | MVT |
| GlobalISel (post-legalizer) | LLT (legal types only) |
| MachineInstr (physical regs) | MVT (via register class) |
| MCInst | No type system (raw encoding) |

## 81.3 SelectionDAG: The Traditional Framework

### 81.3.1 The SelectionDAG

The **SelectionDAG** is a directed acyclic graph where each node represents one or more machine operations, edges are data dependencies, and glue edges encode ordering constraints. Nodes have typed outputs (MVT) and typed inputs.

Key node categories:

```
ISD::ADD, ISD::SUB, ISD::MUL, ISD::SDIV    — integer arithmetic
ISD::FADD, ISD::FSUB, ISD::FMUL             — float arithmetic
ISD::LOAD, ISD::STORE                        — memory operations
ISD::BR, ISD::BRCOND, ISD::BRINDIRECT       — control flow
ISD::CopyToReg, ISD::CopyFromReg             — register assignments
ISD::CALLSEQ_START, ISD::CALLSEQ_END         — call sequencing
TargetISD::*                                 — target-specific nodes
```

### 81.3.2 Four Phases of SelectionDAG

The SelectionDAG pipeline has four phases:

1. **Building** (`SelectionDAGBuilder`): converts LLVM IR instructions to DAG nodes. Every IR instruction maps to one or more ISD nodes.

2. **Legalization** (`SelectionDAGLegalizeTypes`, `SelectionDAGLegalize`): converts nodes and types that the target does not support into equivalent sequences that it does. If the target doesn't support `i64` add, split into two `i32` adds. If the target doesn't support SIMD `v8i16` multiply, scalarize it.

3. **Combining** (`DAGCombiner`): peephole optimizations on the DAG — constant folding, algebraic simplification, pattern matching. Runs both before and after legalization.

4. **Instruction selection** (`SelectionDAGISel`): pattern-matches DAG nodes against target patterns (from TableGen `.td` files) and replaces them with `MachineSDNode` objects representing target instructions. Produces a `MachineFunction` with virtual registers.

### 81.3.3 SelectionDAG Advantages and Disadvantages

| | Advantage | Disadvantage |
|-|-----------|-------------|
| Representation | Value-based DAG enables rich combining | DAG must fit in memory for entire function |
| Legalization | Mature, handles complex type/operation combos | Hard to extend; C++ pattern matching is complex |
| Scheduling | DAG naturally exposes ILP | Scheduling after isel, not before |
| Debug | Well-established tooling (`-view-sunit-dags`) | Complex dataflow hard to debug |

## 81.4 GlobalISel: The New Framework

### 81.4.1 Motivation

SelectionDAG's design has several scalability problems for modern targets:
- The DAG must be materialized for the entire function before any processing — memory pressure for large functions.
- The legalization and combining frameworks are complex, duplicating logic.
- Integrating machine-level analysis (live ranges, register banks) into DAG is awkward.

**GlobalISel** (Global Instruction Selection) was designed to address these:
- It operates on a **linear** MachineFunction (not a DAG), enabling integration with later pipeline passes.
- It uses a clean separation between `IRTranslator`, `Legalizer`, `RegBankSelect`, and `InstructionSelect`.
- The `LLT` type system is simpler and more extensible than `MVT`.

### 81.4.2 GlobalISel Pipeline

```
LLVM IR Function
    │
    ├── IRTranslator
    │   IR → MachineFunction with G_* generic opcodes and LLT types
    │
    ├── Legalizer
    │   G_* generic opcodes → target-legal G_* opcodes
    │   (widens, narrows, clamps LLT types to target-supported sizes)
    │
    ├── RegBankSelect
    │   Assigns register banks (GPR, FPR, vector, etc.) to virtual regs
    │
    ├── InstructionSelect
    │   G_* generic opcodes + register banks → target instructions
    │   (using TableGen-generated or hand-written combiners/selecters)
    │
    └── MachineFunction (target instructions, virtual registers)
```

### 81.4.3 G_ Generic Opcodes

GlobalISel defines **generic opcodes** (prefixed `G_`) in `llvm/CodeGen/GenericOpcodes.h`:

```
G_ADD, G_SUB, G_MUL, G_SDIV, G_UDIV
G_FADD, G_FSUB, G_FMUL, G_FDIV
G_LOAD, G_STORE, G_ATOMIC_CMPXCHG
G_ICMP, G_FCMP, G_SELECT
G_PHI, G_CONSTANT, G_FCONSTANT
G_INTRINSIC, G_INTRINSIC_W_SIDE_EFFECTS
G_BUILD_VECTOR, G_EXTRACT_VECTOR_ELT, G_INSERT_VECTOR_ELT
G_MERGE_VALUES, G_UNMERGE_VALUES
```

These generic opcodes are target-independent representations of operations. The `Legalizer` converts them to target-legal variants; `InstructionSelect` converts them to target-specific opcodes.

### 81.4.4 SDAG vs GlobalISel Coexistence

As of LLVM 22.1, SelectionDAG and GlobalISel coexist in the same target backends. GlobalISel is the default for AArch64 (production quality), x86-64 (fast path at `-O0`), and RISC-V (most paths). SelectionDAG is still used as a fallback for:
- Targets without GlobalISel (PowerPC, MIPS, SPARC still use SDAG primarily).
- Constructs that GlobalISel cannot yet handle (unusual calling conventions, complex lowering patterns).
- Debugging: GlobalISel can fall back to SDAG per-function or per-instruction.

```bash
# Force GlobalISel on AArch64:
clang -target aarch64 -mllvm -global-isel=1 -O2 input.c

# Force SDAG:
clang -target aarch64 -mllvm -global-isel=0 -O2 input.c

# GlobalISel with fallback to SDAG:
clang -target aarch64 -mllvm -global-isel=1 \
      -mllvm -global-isel-abort=0 -O2 input.c
```

The `-global-isel-abort` flag controls what happens when GlobalISel encounters an unsupported construct: `1` = abort (crash), `0` = fall back to SDAG, `2` = emit a diagnostic and continue.

## 81.5 MachineFunction and the Machine IR

### 81.5.1 MachineFunction Structure

The output of instruction selection is a `MachineFunction` — the machine-level equivalent of LLVM IR's `Function`:

```cpp
class MachineFunction {
  Function &F;                              // original LLVM IR function
  MachineFrameInfo FrameInfo;             // stack frame layout
  MachineRegisterInfo RegInfo;            // virtual register table
  MachineConstantPool ConstantPool;       // constant pool entries
  SmallVector<MachineBasicBlock *, 8> BBs; // basic block list
};
```

Each `MachineBasicBlock` contains a list of `MachineInstr` objects (the instruction stream). Each `MachineInstr` has an opcode, a list of `MachineOperand` values (virtual registers, physical registers, immediates, memory references), and metadata.

### 81.5.2 Virtual Registers

Before register allocation, the machine function uses **virtual registers** (numbered from 0 upward). Virtual registers have a **register class** (`RC`) that constrains which physical registers they can be assigned to:

```
%0:gr64 = COPY $rdi    ; virtual reg %0 of class GR64 (64-bit GPR)
%1:gr64 = MOV64ri 42   ; load immediate
ADD64rr %0, %1         ; add virtual regs
```

The register allocator assigns physical registers (e.g., `%rax`, `%rbx`) to virtual registers, subject to register class constraints and liveness.

### 81.5.3 Machine Properties

`MachineFunctionProperties` tracks which properties hold after each pass:

```cpp
MachineFunctionProperties::Property::IsSSA    // SSA form (before phi elimination)
MachineFunctionProperties::Property::NoPHIs   // no PHI nodes (after phi elim)
MachineFunctionProperties::Property::NoVRegs  // no virtual regs (after regalloc)
MachineFunctionProperties::Property::TracksLiveness  // liveness info is valid
```

Passes declare which properties they require and which they preserve/invalidate, enabling the pass manager to verify invariants.

---

## Chapter Summary

- The LLVM backend pipeline transforms LLVM IR through instruction selection, pre-regalloc passes, register allocation, post-regalloc passes, and MC layer emission; each stage is a `MachineFunctionPass`.
- **Three type systems**: LLVM IR types (abstract, before isel), MVT/EVT (SelectionDAG, compact 16-bit encodings), and LLT (GlobalISel, size/shape without IR semantics).
- **SelectionDAG** (the traditional framework) builds a DAG from IR, legalizes types and operations, combines patterns, and selects target instructions; it is mature but has memory and complexity drawbacks.
- **GlobalISel** (the new framework) uses a linear MachineFunction with `G_*` generic opcodes, processed by `IRTranslator → Legalizer → RegBankSelect → InstructionSelect`; it is the default for AArch64 and RISC-V.
- SelectionDAG and GlobalISel **coexist**: GlobalISel is preferred for new work and production AArch64/x86; SDAG remains the fallback and is used by targets without GlobalISel support.
- The `MachineFunction` is the common representation for both frameworks after instruction selection: it holds virtual registers (with register classes), `MachineBasicBlock` lists, and `MachineInstr` streams that the subsequent backend passes transform.


---

@copyright jreuben11
