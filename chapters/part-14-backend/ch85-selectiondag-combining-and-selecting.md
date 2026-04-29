# Chapter 85 — SelectionDAG: Combining and Selecting

*Part XIV — The Backend*

Once the SelectionDAG has been built from LLVM IR and legalized to types and operations that the target supports, two further processes transform it into a sequence of actual machine instructions: the DAGCombiner peephole optimizer, which simplifies the DAG algebraically and structurally, and the instruction selector, which pattern-matches legal DAG subtrees against target instruction patterns. This chapter examines both in depth — the DAGCombiner's rule set and extension points, and the TableGen-generated `SelectCode` function that drives greedy instruction selection.

## 85.1 The DAGCombiner

### 85.1.1 Architecture and Invocation

[`DAGCombiner`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/SelectionDAG/DAGCombiner.cpp) is the largest single file in the SelectionDAG framework, exceeding 25,000 lines. It implements a worklist-driven peephole optimizer: every node in the DAG is placed on a worklist, and the combiner processes each node, attempting to replace it with a simpler or cheaper equivalent. When a replacement is made, all users of the replaced node are re-added to the worklist.

```cpp
// DAGCombiner.cpp
class DAGCombiner {
  SelectionDAG &DAG;
  const TargetLowering &TLI;
  CombineLevel Level;  // BeforeLegalizeTypes, AfterLegalizeTypes,
                       // AfterLegalizeVectorOps, AfterLegalizeDAG
  SmallSetVector<SDNode*, 64> WorklistContents;

  SDValue combine(SDNode *N);
  SDValue visitADD(SDNode *N);
  SDValue visitSUB(SDNode *N);
  SDValue visitMUL(SDNode *N);
  SDValue visitSHL(SDNode *N);
  SDValue visitAND(SDNode *N);
  // ... one visitor per ISD opcode
};
```

The `Level` field controls which combines are legal: before type legalization, the combiner is conservative about introducing new types; after full legalization it can assume all types are target-native and freely fold across type boundaries.

The combiner is invoked three times per basic block (as shown in Chapter 84):

1. **BeforeLegalizeTypes** — after DAG building, before any legalization.
2. **AfterLegalizeTypes** — after type legalization, before operation legalization.
3. **AfterLegalizeDAG** — after operation legalization, before instruction selection.

### 85.1.2 Constant Folding

The first and simplest class of combines is constant folding. When all operands of a node are `ConstantSDNode` or `ConstantFPSDNode`, the combiner evaluates the result at compile time:

```cpp
// visitADD (simplified):
SDValue DAGCombiner::visitADD(SDNode *N) {
  SDValue N0 = N->getOperand(0), N1 = N->getOperand(1);
  EVT VT = N->getValueType(0);

  // Constant folding:
  if (SDValue Folded = DAG.FoldConstantArithmetic(ISD::ADD, DL, VT, {N0, N1}))
    return Folded;

  // x + 0 = x:
  if (isNullConstant(N1)) return N0;

  // Commutativity — put constants on the right:
  if (isa<ConstantSDNode>(N0) && !isa<ConstantSDNode>(N1))
    return DAG.getNode(ISD::ADD, DL, VT, N1, N0);

  // ...
}
```

`DAG.FoldConstantArithmetic` dispatches to `APInt` or `APFloat` arithmetic for integer and floating-point constants respectively. The result is a new `ConstantSDNode` or `ConstantFPSDNode`.

### 85.1.3 Strength Reduction

Strength reduction replaces expensive operations with cheaper equivalents. The most common case is replacing multiplication by a power-of-two constant with a shift:

```cpp
// visitMUL:
if (ConstantSDNode *C = isConstOrConstSplat(N1)) {
  APInt Val = C->getAPIntValue();
  // Multiply by power of 2 → left shift:
  if (Val.isPowerOf2()) {
    return DAG.getNode(ISD::SHL, DL, VT, N0,
                       DAG.getConstant(Val.logBase2(), DL, ShiftAmtTy));
  }
  // Multiply by 2^k - 1 → (x << k) - x:
  if ((Val + 1).isPowerOf2()) {
    unsigned Log = (Val + 1).logBase2();
    SDValue Shl = DAG.getNode(ISD::SHL, DL, VT, N0,
                               DAG.getConstant(Log, DL, ShiftAmtTy));
    return DAG.getNode(ISD::SUB, DL, VT, Shl, N0);
  }
  // Multiply by 2^k + 1 → (x << k) + x:
  if ((Val - 1).isPowerOf2()) {
    unsigned Log = (Val - 1).logBase2();
    SDValue Shl = DAG.getNode(ISD::SHL, DL, VT, N0,
                               DAG.getConstant(Log, DL, ShiftAmtTy));
    return DAG.getNode(ISD::ADD, DL, VT, Shl, N0);
  }
}
```

Division by a constant integer is replaced by a multiply-and-shift sequence using the Hacker's Delight magic number algorithm, implemented in `SelectionDAGTargetInfo::BuildSDIVSequence` and `BuildUDIVSequence`.

### 85.1.4 Algebraic Simplification

The combiner encodes a large set of algebraic identities:

```
x + 0    →  x
x * 1    →  x
x * 0    →  0
x - x    →  0
x ^ x    →  0
x & x    →  x
x | 0    →  x
x | -1   →  -1
~~x      →  x            (double NOT)
-(-x)    →  x
x >> 0   →  x
x << 0   →  x
```

And more complex reassociation patterns:

```
(x + C1) + C2  →  x + (C1+C2)
(x << C1) << C2  →  x << (C1+C2)   [if no overflow]
(x & C1) & C2  →  x & (C1 & C2)
```

```cpp
// visitAND — constant mask folding:
SDValue DAGCombiner::visitAND(SDNode *N) {
  SDValue N0 = N->getOperand(0), N1 = N->getOperand(1);
  // (x & C1) & C2  →  x & (C1&C2):
  if (N0.getOpcode() == ISD::AND)
    if (auto *C1 = dyn_cast<ConstantSDNode>(N0.getOperand(1)))
      if (auto *C2 = dyn_cast<ConstantSDNode>(N1))
        return DAG.getNode(ISD::AND, DL, VT, N0.getOperand(0),
                           DAG.getConstant(C1->getAPIntValue() &
                                           C2->getAPIntValue(), DL, VT));
  // ...
}
```

### 85.1.5 Bitfield Extraction Patterns

A common pattern is extracting a bitfield using shifts and masks. The combiner recognizes these and converts them to `ISD::EXTRACT_BITS` or target-specific bit-extraction instructions:

```cpp
// (x >> C1) & C2  where C2 == (1 << width) - 1
// This is a bitfield extract of 'width' bits starting at bit C1.
// On x86, this may become BEXTR; on ARM, UBFX.
SDValue DAGCombiner::visitSRL(SDNode *N) {
  // ...
  // Check for shift-and-mask pattern:
  if (N->getValueType(0).isInteger()) {
    if (auto *MaskN = dyn_cast<ConstantSDNode>(N->getOperand(1))) {
      if (N0.getOpcode() == ISD::AND) {
        // Fold: (x & mask) >> amt  when mask is aligned
        // ...
      }
    }
  }
}
```

### 85.1.6 Memory Operation Combines

The combiner also optimizes memory operations:

- **Load-store forwarding**: A `LOAD` from the same address as a recent `STORE` in the same chain can be replaced by the stored value.
- **Load combining**: Two adjacent byte loads can be combined into a 16-bit load.
- **Dead store elimination**: A `STORE` followed immediately by another `STORE` to the same address can be simplified (the first store is dead).
- **Zero-extension of loads**: A `ZEXTLOAD` of 8 bits into 32 bits, followed by a mask with `0xFF`, can drop the mask.

```cpp
// Load-store forwarding in visitLOAD:
if (StoreSDNode *ST = dyn_cast<StoreSDNode>(Chain.getNode())) {
  if (ST->getBasePtr() == Ptr &&
      ST->getMemoryVT() == LD->getMemoryVT())
    return CombineTo(N, ST->getValue(), Chain);
}
```

## 85.2 Target DAG Combines

### 85.2.1 The PerformDAGCombine Hook

Beyond the generic combines, targets can implement custom combining logic by overriding `TargetLowering::PerformDAGCombine`:

```cpp
// TargetLowering.h
virtual SDValue PerformDAGCombine(SDNode *N, DAGCombinerInfo &DCI) const {
  return SDValue();  // Default: no combine
}
```

The DAGCombiner calls this after attempting all generic combines. The target's implementation checks the opcode and applies target-specific knowledge:

```cpp
// X86ISelLowering.cpp (simplified)
SDValue X86TargetLowering::PerformDAGCombine(SDNode *N,
                                              DAGCombinerInfo &DCI) const {
  switch (N->getOpcode()) {
  case ISD::VECTOR_SHUFFLE:
    return combineVectorShuffle(N, DCI.DAG, *this, Subtarget);
  case X86ISD::PSHUFB:
    return combinePSHUFB(N, DCI.DAG, *this, Subtarget);
  case ISD::AND:
    return combineAnd(N, DCI, Subtarget);
  // ...
  }
  return SDValue();
}
```

A representative target combine on X86: two `PSHUFB` (packed shuffle bytes) instructions operating on the same register can often be fused into one, or simplified to a simpler permutation.

### 85.2.2 DAGCombinerInfo

The `DAGCombinerInfo` struct passed to `PerformDAGCombine` exposes:

```cpp
struct DAGCombinerInfo {
  SelectionDAG &DAG;
  CombineLevel Level;         // where in the pipeline we are
  bool IsBeforeLegalize() const;
  bool IsBeforeLegalizeOps() const;
  bool IsAfterLegalizeDAG() const;
  void AddToWorklist(SDNode *N);  // schedule for re-examination
  SDValue CombineTo(SDNode *N, SDValue Res, bool AddTo = true);
};
```

Targets use `IsBeforeLegalize()` to guard combines that introduce potentially illegal types, avoiding the introduction of nodes that would immediately need re-legalization.

## 85.3 Pattern-Based Instruction Selection

### 85.3.1 The TableGen isel Backend

Instruction selection converts the legal, combined DAG into a sequence of `MachineInstr` objects. LLVM uses a pattern-matching approach defined in `.td` TableGen files. Each instruction definition in a target's `.td` files has an optional `Pattern` field:

```tablegen
// AArch64InstrArith.td (representative)
def ADDWrr : InstA64<...> {
  let Pattern = [(set GPR32:$Rd, (add GPR32:$Rn, GPR32:$Rm))];
}

def ADDXri : InstA64<...> {
  let Pattern = [(set GPR64:$Rd, (add GPR64:$Rn, imm12:$imm))];
}
```

`llvm-tblgen -gen-dag-isel` processes these patterns and generates the `SelectCode` function, a decision tree that matches DAG nodes bottom-up.

### 85.3.2 Pattern Syntax

Patterns are written in a Lisp-like syntax:

```tablegen
(set ResultReg, (OpNode Operand1, Operand2))
```

Leaf nodes in patterns can be:

| Leaf type | Example | Meaning |
|---|---|---|
| Register class | `GPR32:$r` | Any virtual register in GPR32 |
| Integer immediate | `imm:$n` | Any immediate value |
| Complex pattern | `addr:$ptr` | Matched by a C++ function |
| Constant | `(i32 0)` | Literal zero of type i32 |
| Node result | `(add ...)` | Nested pattern |

Patterns can use `(PatFrag ...)` to factor out common sub-patterns:

```tablegen
def load : PatFrag<(ops node:$ptr), (unindexed_load node:$ptr)>;
def sextloadi8 : PatFrag<(ops node:$ptr),
                          (sext_load node:$ptr), [{
  return cast<LoadSDNode>(N)->getMemoryVT() == MVT::i8;
}]>;
```

The inline C++ guard (in `[{ ... }]`) is a predicate that must return `true` for the pattern to match.

### 85.3.3 The SelectCode Function

`llvm-tblgen` generates a massive `SelectCode` function (often tens of thousands of lines for complex targets) that implements a decision tree. The function takes a DAG node and walks through checks in a specific order to find the first matching pattern. A stylized excerpt:

```cpp
// AArch64GenDAGISel.inc (generated)
SDNode *AArch64DAGToDAGISel::SelectCode(SDNode *N) {
  MVT::SimpleValueType NVT = N->getValueType(0).getSimpleVT().SimpleTy;
  switch (N->getOpcode()) {
  case ISD::ADD: {
    switch (NVT) {
    case MVT::i32: {
      // Check: is operand 1 a 12-bit immediate?
      if (auto *C = dyn_cast<ConstantSDNode>(N->getOperand(1)))
        if (C->getZExtValue() < 4096) {
          SDValue Ops[] = { N->getOperand(0),
                            CurDAG->getTargetConstant(C->getZExtValue(),
                                                      SDLoc(N), MVT::i32) };
          return CurDAG->getMachineNode(AArch64::ADDWri, SDLoc(N),
                                        MVT::i32, Ops);
        }
      // Fallback: register operand
      return CurDAG->getMachineNode(AArch64::ADDWrr, SDLoc(N), MVT::i32,
                                    N->getOperand(0), N->getOperand(1));
    }
    case MVT::i64: { /* ... */ }
    }
  }
  // ... thousands more cases
  }
}
```

The decision tree structure ensures that more specific patterns (e.g., those with immediate operands) are checked before less specific ones (register-register forms), reflecting the priority order defined in the `.td` source by the `AddedComplexity` field on pattern alternatives.

### 85.3.4 Pattern Priority and AddedComplexity

When multiple patterns can match the same DAG subtree, `llvm-tblgen` selects the highest-priority one. Priority is determined by:

1. **Pattern specificity**: More specific patterns win. An instruction that matches a register+immediate pattern wins over one that matches register+register, when the immediate fits.
2. **`AddedComplexity`**: An explicit integer priority that can be set on an instruction or pattern:

```tablegen
// Give this pattern a bonus score so it wins ties
def : Pat<(mul GPR64:$r, (i64 imm:$n)),
          (MULXri GPR64:$r, imm:$n)>,
      Requires<[HasExtM]>,
      AddedComplexity<10>;
```

3. **Pattern depth**: Patterns that match more nodes at once win over shallow patterns.

### 85.3.5 ComplexPattern Matchers

Some operands cannot be described purely declaratively in TableGen. The `ComplexPattern` mechanism delegates matching to a C++ function:

```tablegen
// AArch64 address matching:
def addr : ComplexPattern<iPTR, 2, "SelectArithImmed", []>;
```

```cpp
// AArch64ISelDAGToDAG.cpp
bool AArch64DAGToDAGISel::SelectArithImmed(SDValue N, SDValue &Val,
                                            SDValue &Shift) {
  // Can this node be represented as a 12-bit immediate shifted by 0 or 12?
  if (auto *CN = dyn_cast<ConstantSDNode>(N)) {
    uint64_t Immed = CN->getZExtValue();
    if (Immed >> 12 == 0) {
      Val = CurDAG->getTargetConstant(Immed, SDLoc(N), MVT::i32);
      Shift = CurDAG->getTargetConstant(0, SDLoc(N), MVT::i32);
      return true;
    }
    if ((Immed & 0xFFF) == 0 && Immed >> 24 == 0) {
      Val = CurDAG->getTargetConstant(Immed >> 12, SDLoc(N), MVT::i32);
      Shift = CurDAG->getTargetConstant(12, SDLoc(N), MVT::i32);
      return true;
    }
  }
  return false;
}
```

Memory address selection is the canonical use case: modern ISAs support base+offset, base+scaled-register, and similar addressing modes, all of which are best described in C++ rather than declarative patterns.

## 85.4 Custom Selection in C++

### 85.4.1 The Select() Override

When TableGen patterns are insufficient, targets override `SelectionDAGISel::Select()`:

```cpp
// AArch64ISelDAGToDAG.cpp
void AArch64DAGToDAGISel::Select(SDNode *N) {
  if (N->isMachineOpcode()) {
    N->setNodeId(-1);
    return;  // Already selected
  }

  switch (N->getOpcode()) {
  case ISD::LOAD:
    if (trySelectCastFixedLengthToScalableVector(N)) return;
    break;
  case AArch64ISD::LDRPRE:
    SelectPredicatedLoad(N, /* ... */); return;
  // Many special cases...
  }

  // Fallback to generated SelectCode:
  SelectCode(N);
}
```

The `Select()` function is called for each node in reverse topological order (leaves first, root last). After processing, a node has either been replaced by a `MachineSDNode` (via `CurDAG->getMachineNode(...)`) or left for `SelectCode` to handle.

### 85.4.2 Emitting MachineNodes

`getMachineNode` creates a node that represents a specific machine instruction:

```cpp
// Creating a machine node for AArch64 MOV immediate:
SDNode *Mov = CurDAG->getMachineNode(
  AArch64::MOVi32imm, SDLoc(N), MVT::i32,
  CurDAG->getTargetConstant(Imm, SDLoc(N), MVT::i32));
ReplaceNode(N, Mov);
```

Once `ReplaceNode` is called, all users of `N` in the DAG automatically receive the new node.

### 85.4.3 Multi-Result Instructions

Some machine instructions produce multiple results (e.g., `SDIVREM` produces both quotient and remainder). These are selected by emitting a `MachineNode` with multiple result types:

```cpp
SDVTList VTs = CurDAG->getVTList(MVT::i32, MVT::i32);  // quot, rem
SDNode *DivRem = CurDAG->getMachineNode(X86::IDIV32r, DL, VTs,
                                         Dividend, Divisor);
// Users of the quotient use DivRem:0
// Users of the remainder use DivRem:1
```

## 85.5 Instruction Selection Debugging

### 85.5.1 Viewing DAG Stages

LLVM provides GraphViz-based visualization for each pipeline stage. When `llc` is built with GraphViz support:

```bash
# View the initial DAG (before any combining):
llc -view-dag-combine1 input.ll

# View after first DAGCombiner pass:
llc -view-dag-combine2 input.ll

# View the legalized DAG:
llc -view-legalize-dags input.ll

# View after the final DAGCombiner pass:
llc -view-dag-combine-before input.ll

# View the selected DAG (machine instructions):
llc -view-sunit-dags input.ll
```

Each command emits a `.dot` file and attempts to open it with `xdg-open` or a configured viewer. The DOT files can be rendered manually with `dot -Tsvg`.

### 85.5.2 Text Debug Output

For scripted debugging:

```bash
# Print the isel trace for every node:
llc -debug-only=isel input.ll 2>&1 | grep -A3 "Selecting:"

# Print all combine decisions:
llc -debug-only=dagcombine input.ll 2>&1 | less

# Print scheduling DAGs:
llc -debug-only=sched input.ll 2>&1 | less
```

The `isel` debug channel prints messages like:

```
Selecting: t5: i32 = add t3, Constant:i32<4>
ISEL: Starting pattern match
  Initial Opcode index to 12345
  Match failed at index 12360
  OPC_CheckOpcode, checking ISD::Constant... SUCCESS
  Creating MachineNode: ADDWri
```

### 85.5.3 The MachineVerifier

After isel, the `MachineVerifier` pass verifies the resulting `MachineFunction`. It checks:

- All virtual register uses have a single definition.
- No uses of physical registers across basic block boundaries (unless specially modeled).
- All register class constraints are satisfied.
- No dangling nodes or broken use-def chains.

```bash
# Enable machine verifier after each pass:
llc -verify-machineinstrs input.ll

# Verify at a specific pass:
llc -run-pass=instruction-select -verify-machineinstrs input.ll
```

### 85.5.4 FastISel and its Debug Mode

For debug builds, LLVM can use `FastISel` — a simpler instruction selector that selects instructions greedily without DAG construction. It is significantly faster but less optimal. When `FastISel` cannot handle an instruction, it falls back to the full SelectionDAG pipeline:

```bash
# Force full SelectionDAG (disable FastISel):
llc -fast-isel=false input.ll

# Print FastISel fallback reasons:
llc -debug-only=fast-isel input.ll 2>&1 | grep "FastISel failed"
```

## 85.6 Post-Selection Cleanup

### 85.6.1 CopyToReg / CopyFromReg Elimination

After `SelectCode` runs, the DAG contains a mix of `MachineSDNode` objects and residual `CopyToReg`/`CopyFromReg` nodes from the block boundary stitching. `SelectionDAGISel::FinishBasicBlock` translates the selected DAG into actual `MachineInstr` objects in the `MachineBasicBlock`.

The translation traverses the selected DAG in topological order. For each `MachineSDNode`, it creates a corresponding `MachineInstr`. For `CopyToReg`/`CopyFromReg`, it creates `COPY` pseudo-instructions that the register allocator will later handle.

### 85.6.2 PHI Lowering

PHI nodes from LLVM IR are represented as `COPY` instructions at the end of each predecessor block, feeding into PHI `MachineInstr` at the beginning of the successor. The `TwoAddressInstruction` pass and the register allocator later convert these into physical-register assignments.

## 85.7 The Scheduler After isel

The selected DAG is handed to the pre-register-allocation instruction scheduler (if scheduling is enabled at this point). This scheduler operates on the `ScheduleDAG` representation derived from the selected DAG. The scheduling model from the target's `.td` file (read and emit latencies, hazards) guides the scheduler's decisions. Chapter 83 covered the scheduling model infrastructure; here it suffices to note that the connection from isel to scheduling is:

```
SelectCode  →  ScheduleDAGSDNodes::BuildSchedGraph()
            →  ScheduleDAGSDNodes::Schedule()
            →  InstrEmitter::EmitSchedule()
            →  MachineBasicBlock instructions
```

The inline scheduler choices (LinearOrder, ILP, VLIW, etc.) do not alter correctness but significantly affect performance on out-of-order processors.

## 85.8 Pattern Matching Internals

### 85.8.1 The OPC_ Instruction Stream

`llvm-tblgen` encodes the SelectCode decision tree as a byte sequence of opcodes interpreted by an embedded interpreter in `SelectionDAGISel.cpp`. Representative opcodes:

| Opcode | Meaning |
|---|---|
| `OPC_SwitchOpcode` | Dispatch on node opcode |
| `OPC_CheckType` | Verify value type matches |
| `OPC_CheckChild0Type` | Check type of first operand |
| `OPC_CheckInteger` | Verify a constant value |
| `OPC_CheckPatternPredicate` | Run a C++ predicate |
| `OPC_ComplexPattern` | Call a ComplexPattern function |
| `OPC_EmitNode` | Create a machine node |
| `OPC_MorphNodeTo` | Replace the current node |
| `OPC_RecordNode` | Save a matched node for later use |

This interpreter lives in `SelectionDAGISel::SelectCodeCommon()`. The TableGen-generated `SelectCode` function sets up the opcode array and calls `SelectCodeCommon`.

### 85.8.2 Predicate Guards

Many patterns have predicates that gate their applicability. These predicates check CPU feature flags, value constraints, or other conditions:

```tablegen
// Only match when the target has NEON:
def VADDQ : NeonInst<...> {
  let Pattern = [(set VPR128:$Vd,
                  (v4i32 (add VPR128:$Vn, VPR128:$Vm)))];
  let Predicates = [HasNEON];
}
```

The generated `OPC_CheckPatternPredicate` instruction checks the predicate at match time:

```cpp
// Generated:
case OPC_CheckPatternPredicate:
  if (!checkPatternPredicate(MatcherTable[MatcherIndex++]))
    goto FAIL;
  break;
```

And the predicate function:

```cpp
bool AArch64DAGToDAGISel::checkPatternPredicate(unsigned PredNo) {
  switch (PredNo) {
  case 0: return Subtarget->hasNEON();
  case 1: return Subtarget->hasSVE();
  // ...
  }
}
```

### 85.8.3 Multi-Instruction Patterns

A single DAG subtree can be matched by a pattern that emits multiple machine instructions. This is handled via `OPC_EmitNode` (emit but don't finalize) followed by more pattern checks, then `OPC_MorphNodeTo` to finalize:

```tablegen
// Match (add x, (mul y, z)) → MADD on AArch64
def : Pat<(i64 (add GPR64:$Ra, (mul GPR64:$Rn, GPR64:$Rm))),
          (MADDXrrr GPR64:$Rn, GPR64:$Rm, GPR64:$Ra)>;
```

This pattern consumes the `mul` sub-tree entirely and emits a single MADD instruction, saving one instruction compared to selecting `MUL` and `ADD` separately.

## Chapter Summary

- `DAGCombiner` is the primary peephole optimizer for SelectionDAGs; it runs three times per basic block (before type legalization, after type legalization, and after operation legalization) to simplify the DAG algebraically and structurally.

- Key combine categories: constant folding via `FoldConstantArithmetic`, strength reduction (multiplication by constant → shift sequences), algebraic identities (`x+0=x`, `x^x=0`), bitfield extraction recognition, and memory-operation forwarding and dead-store elimination.

- Targets extend combining via `TargetLowering::PerformDAGCombine`, gated on `CombineLevel` to avoid introducing nodes that would require re-legalization.

- Pattern-based instruction selection is driven by TableGen: `.td` Pattern fields declare which DAG subtrees map to which machine instructions. `llvm-tblgen -gen-dag-isel` generates the `SelectCode` function as a decision tree interpreter operating over a byte-encoded opcode stream.

- Pattern priority is determined by specificity (immediate patterns win over register-register), the `AddedComplexity` attribute, and pattern depth (deeper patterns win).

- `ComplexPattern` matchers bridge declarative TableGen patterns and C++ logic; they are essential for address mode selection where architectural addressing modes require complex constraint checking.

- Custom selection via `SelectionDAGISel::Select()` override handles cases that are too complex for TableGen patterns: multi-result instructions, context-sensitive selection, and target-specific optimizations like AArch64's scalable vector addressing.

- Debugging tools include `-view-dag-combine1`, `-view-legalize-dags`, `-view-sunit-dags` for visual DAG inspection, and `-debug-only=isel` / `-debug-only=dagcombine` for textual traces of every selection and combining decision.

- After SelectCode completes, `InstrEmitter` translates the selected DAG nodes into `MachineInstr` objects, and the pre-RA scheduler reorders them according to the target's scheduling model before register allocation begins.


---

@copyright jreuben11
