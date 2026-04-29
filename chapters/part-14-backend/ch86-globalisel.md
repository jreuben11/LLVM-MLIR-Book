# Chapter 86 — GlobalISel

*Part XIV — The Backend*

GlobalISel is LLVM's modern instruction selection framework, designed to overcome the fundamental architectural limitations of SelectionDAG. Where SelectionDAG processes IR one basic block at a time, GlobalISel operates across the entire function, enabling global optimization during selection. Where SelectionDAG invents the DAG representation only to immediately serialize it into instructions, GlobalISel works directly on `MachineInstr` objects from the start, using a set of generic opcodes (`G_*`) that are progressively refined to target-specific instructions. This chapter traces the four-stage GlobalISel pipeline — IRTranslator, Legalizer, RegBankSelect, and InstructionSelect — and covers the combiner DSL used to write optimization rules.

## 86.1 Architecture Overview

### 86.1.1 The Generic MachineIR Representation

GlobalISel introduces the concept of *Generic Machine IR* (gMIR): `MachineInstr` objects using generic opcodes defined in [`llvm/include/llvm/Support/TargetOpcodes.def`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/TargetOpcodes.def). These opcodes all start with `G_`:

| Generic opcode | Corresponding LLVM IR |
|---|---|
| `G_ADD`, `G_SUB`, `G_MUL` | `add`, `sub`, `mul` |
| `G_FADD`, `G_FSUB`, `G_FMUL` | `fadd`, `fsub`, `fmul` |
| `G_LOAD`, `G_STORE` | `load`, `store` |
| `G_ICMP`, `G_FCMP` | `icmp`, `fcmp` |
| `G_BR`, `G_BRCOND` | `br` (conditional) |
| `G_PHI` | `phi` |
| `G_CONSTANT`, `G_FCONSTANT` | integer/FP constants |
| `G_INTRINSIC`, `G_INTRINSIC_W_SIDE_EFFECTS` | intrinsic calls |
| `G_MERGE_VALUES`, `G_UNMERGE_VALUES` | value splitting/merging |
| `G_BUILD_VECTOR`, `G_EXTRACT_VECTOR_ELT` | vector construction |
| `G_SEXT`, `G_ZEXT`, `G_TRUNC` | integer extension/truncation |
| `G_BITCAST` | reinterpret bits |

Unlike SelectionDAG where types are `EVT`/`MVT`, gMIR uses `LLT` (Low-Level Type):

```cpp
// llvm/include/llvm/CodeGen/LowLevelType.h
class LLT {
  // Scalar: LLT::scalar(32) → i32
  // Pointer: LLT::pointer(0, 64) → ptr addrspace(0)
  // Vector: LLT::vector(4, 32) → <4 x i32>  (ElementCount, scalar width)
  // None/invalid: LLT()
};
```

`LLT` is more expressive than `MVT` because it directly represents pointer address spaces and scalable vectors (`LLT::scalable_vector`).

### 86.1.2 The Four-Stage Pipeline

```
LLVM IR function
     │
     ▼  IRTranslator
Generic MachineFunction (G_* opcodes, LLT types, no register banks)
     │
     ▼  Legalizer (+ pre-legalization combiner)
Legal Generic MachineFunction (all G_* ops are target-legal)
     │
     ▼  RegBankSelect
Generic MachineFunction with register bank assignments
     │
     ▼  InstructionSelect (+ post-select combiner)
Selected MachineFunction (target-specific opcodes only)
     │
     ▼  (standard post-isel passes: register allocation, etc.)
```

Each stage is an `MachineFunctionPass`. They are registered in the target's `TargetPassConfig::addISelPasses`:

```cpp
// In TargetPassConfig.cpp
void TargetPassConfig::addGlobalInstructionSelect() {
  addPass(createIRTranslatorPass());
  addPass(createLegalizerPass());
  addPass(createRegBankSelectPass());
  addPass(createInstructionSelectPass());
}
```

## 86.2 IRTranslator

### 86.2.1 Overview

[`IRTranslator`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/GlobalISel/IRTranslator.cpp) converts an LLVM IR `Function` into a `MachineFunction` populated with `G_*` instructions. Unlike `SelectionDAGBuilder`, which works basic block by basic block, `IRTranslator` processes the entire function.

```cpp
// llvm/include/llvm/CodeGen/GlobalISel/IRTranslator.h
class IRTranslator : public MachineFunctionPass {
  MachineFunction *MF;
  MachineRegisterInfo *MRI;
  const TargetLowering *TLI;

  // Map from LLVM Value* to virtual register
  ValueToVReg VMap;

  bool translateInst(const Instruction &Inst,
                     MachineIRBuilder &MIRBuilder);

  bool translateAdd(const User &U, MachineIRBuilder &MIRBuilder);
  bool translateLoad(const LoadInst &LI, MachineIRBuilder &MIRBuilder);
  bool translateStore(const StoreInst &SI, MachineIRBuilder &MIRBuilder);
  bool translateCall(const CallInst &CI, MachineIRBuilder &MIRBuilder);
  bool translateBr(const BranchInst &BrInst, MachineIRBuilder &MIRBuilder);
  bool translatePHI(const PHINode &PI, MachineIRBuilder &MIRBuilder);
  // ...
};
```

### 86.2.2 Translating Simple Instructions

The `MachineIRBuilder` API provides a fluent interface for building instructions:

```cpp
// translateAdd:
bool IRTranslator::translateAdd(const User &U,
                                 MachineIRBuilder &MIRBuilder) {
  Register Dst = getOrCreateVReg(U);
  Register Src0 = getOrCreateVReg(*U.getOperand(0));
  Register Src1 = getOrCreateVReg(*U.getOperand(1));
  MIRBuilder.buildInstr(TargetOpcode::G_ADD, {Dst}, {Src0, Src1});
  return true;
}
```

The `MachineRegisterInfo` assigns an `LLT` to each virtual register at creation time:

```cpp
Register IRTranslator::getOrCreateVReg(const Value &Val) {
  auto It = VMap.find(&Val);
  if (It != VMap.end()) return It->second;

  LLT Ty = getLLTForType(*Val.getType(), *DL);
  Register VReg = MRI->createGenericVirtualRegister(Ty);
  VMap[&Val] = VReg;
  return VReg;
}
```

### 86.2.3 Translating PHI Nodes

PHI nodes are translated to `G_PHI` instructions in a two-pass approach:

1. In the first pass, create the virtual register and emit `G_PHI` with placeholders.
2. In the second pass (after all blocks are translated), fill in the incoming value registers.

```cpp
// First pass — create G_PHI with correct number of operands:
bool IRTranslator::translatePHI(const PHINode &PI,
                                 MachineIRBuilder &MIRBuilder) {
  Register Dst = getOrCreateVReg(PI);
  auto MIB = MIRBuilder.buildInstr(TargetOpcode::G_PHI).addDef(Dst);
  for (unsigned i = 0; i < PI.getNumIncomingValues(); ++i) {
    MIB.addUse(getOrCreateVReg(*PI.getIncomingValue(i)));
    MIB.addMBB(getMBB(*PI.getIncomingBlock(i)));
  }
  return true;
}
```

### 86.2.4 Translating Calls

Call translation is handled by `CallLowering`, a target-overridden class:

```cpp
// llvm/include/llvm/CodeGen/GlobalISel/CallLowering.h
class CallLowering {
public:
  virtual bool lowerCall(MachineIRBuilder &MIRBuilder,
                          CallLoweringInfo &Info) const;
  virtual bool lowerReturn(MachineIRBuilder &MIRBuilder,
                            const Value *Val,
                            ArrayRef<Register> VRegs) const;
  virtual bool lowerFormalArguments(MachineIRBuilder &MIRBuilder,
                                     const Function &F,
                                     ArrayRef<ArrayRef<Register>> VRegs) const;
};
```

The target's `CallLowering` subclass (e.g., `AArch64CallLowering`) implements the calling convention by examining the ABI and emitting `G_LOAD`, `G_STORE`, `COPY`, `G_SEQUENCE`, and similar instructions to marshal arguments and return values.

### 86.2.5 Translating Returns

```cpp
bool IRTranslator::translateReturn(const ReturnInst &RI,
                                    MachineIRBuilder &MIRBuilder) {
  const Value *Ret = RI.getReturnValue();
  if (!Ret)
    return CLI->lowerReturn(MIRBuilder, nullptr, {});

  SmallVector<Register, 4> VRegs;
  // Get the virtual register(s) holding the return value:
  getOrCreateVRegs(*Ret, VRegs);
  return CLI->lowerReturn(MIRBuilder, Ret, VRegs);
}
```

## 86.3 The Legalizer

### 86.3.1 LegalizerInfo API

[`LegalizerInfo`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/GlobalISel/LegalizerInfo.h) is the table that maps `(opcode, LLT)` pairs to legalization actions. Targets initialize it in their `TargetLowering` subclass constructor:

```cpp
// AArch64LegalizerInfo.cpp (representative)
AArch64LegalizerInfo::AArch64LegalizerInfo(const AArch64Subtarget &ST)
    : LegalizerInfo() {
  using namespace TargetOpcode;
  const LLT s8 = LLT::scalar(8), s16 = LLT::scalar(16);
  const LLT s32 = LLT::scalar(32), s64 = LLT::scalar(64);
  const LLT v4s32 = LLT::vector(4, 32), v2s64 = LLT::vector(2, 64);

  getActionDefinitionsBuilder(G_ADD)
    .legalFor({s32, s64, v4s32, v2s64})  // these types are natively legal
    .widenScalarToNextPow2(0)             // widen scalar types
    .clampNumElements(0, v4s32, v4s32);   // clamp vector sizes

  getActionDefinitionsBuilder(G_LOAD)
    .legalForTypesWithMemDesc({{s8, p0, s8, 8},   // i8 load from ptr
                                {s16, p0, s16, 16},
                                {s32, p0, s32, 32},
                                {s64, p0, s64, 64}})
    .clampScalar(0, s8, s64);

  computeTables();
}
```

The `LegalizeAction` values are:

| Action | Description |
|---|---|
| `Legal` | Operation is natively supported; do nothing |
| `WidenScalar` | Widen the scalar type to the next larger supported type |
| `NarrowScalar` | Narrow the scalar type to a smaller supported type |
| `MoreElements` | Add more vector lanes to use a legal vector type |
| `FewerElements` | Reduce vector lanes (split the vector operation) |
| `Lower` | Expand to a sequence of simpler operations |
| `Libcall` | Replace with a call to a runtime library function |
| `Custom` | Call `LegalizerInfo::legalizeCustom()` |
| `Unsupported` | The operation has no legal form; signal an error |

### 86.3.2 The Legalizer Pass

[`Legalizer`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/GlobalISel/Legalizer.cpp) is the `MachineFunctionPass` that applies these actions. It maintains a worklist of instructions and applies legalizations until fixpoint:

```cpp
bool Legalizer::runOnMachineFunction(MachineFunction &MF) {
  // Initialize worklist with all instructions
  WorkList WL;
  for (auto &MBB : MF)
    for (auto &MI : MBB)
      WL.push_back(&MI);

  LegalizerHelper Helper(MF, LI, Observer, MIRBuilder);

  while (!WL.empty()) {
    MachineInstr *MI = WL.pop_back_val();
    LegalizationArtifactCombiner ArtCombiner(MIRBuilder, MRI, LI, Observer);
    auto Res = Helper.legalizeInstrStep(*MI, WL);
    // Res is {Legalized, AlreadyLegal, UnableToLegalize}
    if (Res != LegalizerHelper::LegalizeResult::Legalized)
      continue;
    // Re-add newly created instructions:
    for (auto *NewMI : InsertedInstrs)
      WL.push_back(NewMI);
  }
}
```

### 86.3.3 LegalizerHelper

[`LegalizerHelper`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/GlobalISel/LegalizerHelper.h) contains the actual transformation implementations:

```cpp
// llvm/include/llvm/CodeGen/GlobalISel/LegalizerHelper.h
class LegalizerHelper {
public:
  // Widen scalar operand/result to a larger type:
  LegalizeResult widenScalar(MachineInstr &MI, unsigned TypeIdx, LLT WideTy);

  // Narrow scalar to a smaller type (may introduce merges/unmerges):
  LegalizeResult narrowScalar(MachineInstr &MI, unsigned TypeIdx, LLT NarrowTy);

  // Split vector operation into multiple narrower operations:
  LegalizeResult fewerElementsVector(MachineInstr &MI, unsigned TypeIdx,
                                      LLT NarrowTy);

  // Add lanes to use a legal vector type:
  LegalizeResult moreElementsVector(MachineInstr &MI, unsigned TypeIdx,
                                     LLT WideTy);

  // Expand to a sequence of simpler operations:
  LegalizeResult lower(MachineInstr &MI, unsigned TypeIdx, LLT Ty);

  // Replace with a library call:
  LegalizeResult libcall(MachineInstr &MI);
};
```

### 86.3.4 Concrete Example: Widening G_ADD

If the target declares `G_ADD` legal only for `s32` and `s64`, but the IR contains a `G_ADD s16`, the Legalizer runs `widenScalar`:

```
; Before widening:
%dst:_(s16) = G_ADD %a:_(s16), %b:_(s16)

; After widenScalar(0, s32):
%a_ext:_(s32)   = G_SEXT %a:_(s16)
%b_ext:_(s32)   = G_SEXT %b:_(s16)
%dst_wide:_(s32) = G_ADD %a_ext, %b_ext
%dst:_(s16)     = G_TRUNC %dst_wide:_(s32)
```

For unsigned operations, `G_ZEXT` is used instead of `G_SEXT`. The `G_TRUNC` at the end ensures the result has the original type, as required by the rest of the function.

### 86.3.5 Concrete Example: Splitting a Vector G_MUL

On a target with no vector multiply but with `v4i32` add, a `v8i32` multiply might first be split to two `v4i32` multiplies, then further legalized based on what the target supports for `v4i32` multiply:

```
; Before:
%r:_(<8 x s32>) = G_MUL %a:_(<8 x s32>), %b:_(<8 x s32>)

; After fewerElementsVector(0, <4 x s32>):
%a_lo:_(<4 x s32>), %a_hi:_(<4 x s32>) = G_UNMERGE_VALUES %a
%b_lo:_(<4 x s32>), %b_hi:_(<4 x s32>) = G_UNMERGE_VALUES %b
%r_lo:_(<4 x s32>) = G_MUL %a_lo, %b_lo
%r_hi:_(<4 x s32>) = G_MUL %a_hi, %b_hi
%r:_(<8 x s32>) = G_MERGE_VALUES %r_lo, %r_hi
```

### 86.3.6 Artifact Combining

After each legalization step, a specialized combiner called `LegalizationArtifactCombiner` runs to simplify the legalization artifacts (`G_MERGE_VALUES`, `G_UNMERGE_VALUES`, `G_SEXT`, `G_ZEXT`, `G_TRUNC`). For example:

```
; G_MERGE followed immediately by G_UNMERGE can be cancelled:
%pair:_(<2 x s32>) = G_MERGE_VALUES %lo:_(s32), %hi:_(s32)
%x:_(s32), %y:_(s32) = G_UNMERGE_VALUES %pair
; → replaced by: use %lo where %x was used, %hi where %y was used
```

This prevents an explosion of unnecessary merges and unmerges as legalization proceeds.

## 86.4 RegBankSelect

### 86.4.1 Register Banks

A *register bank* is a set of physical registers that can hold a particular kind of value. Most ISAs have at least two register banks:

- **GPR bank**: general-purpose integer registers (`X0`–`X30` on AArch64)
- **FPR bank**: floating-point and vector registers (`V0`–`V31` on AArch64)

Some targets have additional banks:
- AArch64: also has an SVE predicate bank (`P0`–`P15`)
- X86: has general, SSE, AVX512 banks

Register banks are defined in TableGen:

```tablegen
// AArch64RegisterBanks.td
def GPRRegBank : RegisterBank<"GPR", [GPR32, GPR64, ...]>;
def FPRRegBank : RegisterBank<"FPR", [FPR8, FPR16, FPR32, FPR64, FPR128, ...]>;
```

### 86.4.2 The RegBankSelect Pass

[`RegBankSelect`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/GlobalISel/RegBankSelect.cpp) assigns a register bank to every virtual register in the `MachineFunction`. After this pass, every virtual register has both an `LLT` (its type) and a `RegisterBank` (its bank). The actual physical register is still unassigned — that happens in register allocation.

```cpp
// llvm/include/llvm/CodeGen/GlobalISel/RegBankSelect.h
class RegBankSelect : public MachineFunctionPass {
  void assignInstr(MachineInstr &MI);
  const InstructionMapping &
  getBestMapping(MachineInstr &MI);
  void applyMapping(MachineInstr &MI,
                    const InstructionMapping &InstrMapping);
};
```

### 86.4.3 InstructionMappings

For each generic instruction, there may be multiple valid mappings of operands to register banks. `RegisterBankInfo` (a target-overridden class) provides the possible mappings:

```cpp
// llvm/include/llvm/CodeGen/RegisterBankInfo.h
class RegisterBankInfo {
  // For each instruction, return all valid operand→bank mappings:
  virtual InstructionMappings
  getInstrPossibleMappings(const MachineInstr &MI) const;

  // Get the mapping with minimum cost:
  virtual const InstructionMapping &
  getInstrMapping(const MachineInstr &MI) const;
};
```

A typical mapping for `G_FADD`:

```
; G_FADD s32 only makes sense in the FPR bank:
G_FADD: {FPR32(dst), FPR32(src0), FPR32(src1)} → cost 1
```

Whereas `G_ADD s32` might appear in either the GPR or FPR bank (if the target has integer add instructions in both), with the GPR mapping having lower cost.

### 86.4.4 Cross-Bank Copies

When the optimal bank assignment requires a value to cross banks (e.g., the result of `G_LOAD f32` in the GPR bank but used by `G_FADD` which requires FPR), `RegBankSelect` inserts `COPY` instructions between the banks:

```
; Before RegBankSelect:
%val:_(s32)  = G_LOAD %ptr:_(p0)   ; might be in GPR
%sum:_(s32)  = G_FADD %val, %y      ; needs FPR

; After RegBankSelect:
%val_gpr:_(s32) = G_LOAD %ptr    [GPR bank]
%val_fpr:_(s32) = COPY %val_gpr  [cross-bank copy: GPR → FPR]
%sum:_(s32)     = G_FADD %val_fpr, %y  [FPR bank]
```

These cross-bank copies are expensive — they often become actual register-to-register moves through the memory system or bypass paths. The RegBankSelect cost model attempts to minimize them.

## 86.5 InstructionSelect

### 86.5.1 Overview

[`InstructionSelect`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/GlobalISel/InstructionSelect.cpp) is the final GlobalISel pass. It replaces every `G_*` instruction with a target-specific machine instruction. Like SelectionDAG's `SelectCode`, it is primarily driven by TableGen patterns, but the matching infrastructure is different.

```cpp
bool InstructionSelect::runOnMachineFunction(MachineFunction &MF) {
  // Process instructions in reverse post-order (uses before defs):
  for (MachineBasicBlock *MBB : post_order(&MF)) {
    for (MachineInstr &MI : make_early_inc_range(reverse(*MBB))) {
      if (!ISel->select(MI))
        reportGISelFailure(MF, TPC, MORE, "gisel-select", "cannot select", MI);
    }
  }
}
```

### 86.5.2 TableGen Patterns for GlobalISel

GlobalISel instruction selection is driven by `GINodeEquiv` and `Pat` declarations in TableGen:

```tablegen
// Tell GlobalISel that G_ADD with i32 type matches the same patterns
// as ISD::ADD with MVT::i32:
def : GINodeEquiv<G_ADD, add>;
def : GINodeEquiv<G_LOAD, load>;
```

Target instructions with `Pattern` fields are automatically available to GlobalISel when `GINodeEquiv` entries map the `ISD::` nodes to `G_*` opcodes. `llvm-tblgen -gen-global-isel` generates the `select()` function.

### 86.5.3 GICombineRule: The Combiner DSL

GlobalISel includes a combiner framework with a TableGen DSL for writing optimization rules. Rules are written as `GICombineRule` records:

```tablegen
// llvm/include/llvm/Target/GlobalISel/Combine.td
def MulByNegOne : GICombineRule<
  (defs root:$root),
  (match (G_MUL $dst, $x, -1:$neg)),
  (apply (G_NEG $dst, $x))>;
```

A `GICombineRule` has three components:
- `defs`: declares which instruction is the "root" (the one being replaced)
- `match`: a pattern that must hold in the `MachineFunction`
- `apply`: the replacement instructions

More complex rules can use C++ match and apply functions:

```tablegen
def RedundantAnd : GICombineRule<
  (defs root:$root),
  (match (G_AND $dst, $x, $mask),
         [{ return matchRedundantAnd(*${root}, MRI); }]),
  (apply [{ applyRedundantAnd(*${root}, MRI, B); }])>;
```

```cpp
// In the target's combiner implementation:
bool matchRedundantAnd(MachineInstr &MI, MachineRegisterInfo &MRI) {
  Register Dst = MI.getOperand(0).getReg();
  Register Src = MI.getOperand(1).getReg();
  auto Mask = getIConstantVRegValWithLookThrough(MI.getOperand(2).getReg(), MRI);
  if (!Mask) return false;
  LLT Ty = MRI.getType(Src);
  // If mask covers all bits of the type, the AND is redundant:
  return Mask->Value.isAllOnes();
}

void applyRedundantAnd(MachineInstr &MI, MachineRegisterInfo &MRI,
                        MachineIRBuilder &B) {
  B.buildCopy(MI.getOperand(0).getReg(), MI.getOperand(1).getReg());
  MI.eraseFromParent();
}
```

### 86.5.4 Pre- and Post-Legalization Combiners

GlobalISel combiners run at multiple points in the pipeline:

1. **Pre-legalization combiner** (`-run-pass=combiner`, before Legalizer): Simplifies the gMIR before legalization, reducing the work the Legalizer must do.
2. **Post-legalization combiner** (between Legalizer and RegBankSelect): Cleans up legalization artifacts.
3. **Post-select combiner** (after InstructionSelect): Performs target-specific combines on selected instructions.

Each combiner is a separate `MachineFunctionPass` configured with a set of `GICombineGroup` rules:

```tablegen
// AArch64PreLegalizerCombiner.td
def AArch64PreLegalizerCombinerHelper : GICombinerHelper<
  "AArch64PreLegalizerCombinerHelperImpl",
  [all_combines,
   AArch64SpecificCombines,
   fabs_fneg_fold]>;
```

## 86.6 Falling Back to SelectionDAG

### 86.6.1 Fallback Mechanisms

GlobalISel is not yet complete for all targets or all constructs. When it cannot handle something, it can fall back to SelectionDAG for that function. The fallback behavior is controlled by:

```bash
# Abort on GlobalISel failure (default in release builds):
llc -global-isel-abort=1 input.ll

# Fall back to SelectionDAG on failure (useful for testing):
llc -global-isel-abort=0 input.ll

# Diagnose failures but continue (verbose fallback):
llc -global-isel-abort=2 input.ll
```

### 86.6.2 Per-Function Fallback

The fallback is implemented in `SelectionDAGISel::runOnMachineFunction` and related plumbing. When `InstructionSelect` fails on an instruction:

```cpp
// InstructionSelect.cpp
if (!ISel->select(MI)) {
  if (TPC.isGlobalISelAbortEnabled())
    reportGISelFailure(MF, TPC, MORE, "gisel-select", "cannot select", MI);
  return false;  // Signal failure to the pass manager
}
```

The `MachineFunctionPass` returning `false` does not automatically trigger SelectionDAG. Instead, the `TargetPassConfig` includes a `GlobalISelFallbackAnalysis` pass that, when GlobalISel fails, deletes the `MachineFunction` contents and re-runs SelectionDAG isel on the function.

### 86.6.3 Per-Instruction Fallback

For cases where most of the function can be selected by GlobalISel but one instruction cannot, some targets implement a hybrid approach: the instruction is pre-lowered to a sequence that GlobalISel can handle, or the `G_*` instruction is left unselected and the instruction selector falls through to SelectionDAG for just that region.

### 86.6.4 Checking GlobalISel Status by Target

As of LLVM 22:

| Target | GlobalISel status |
|---|---|
| AArch64 | Default for `-O0`; stable for all opt levels |
| X86 | Experimental; `-O0` only |
| RISC-V | Active development; used at `-O0` |
| ARM (32-bit) | Partial; fallback common |
| AMDGPU | Well-supported; default selection path |
| Mips | Partial |

```bash
# Check if GlobalISel is enabled for a target:
llc -mtriple=aarch64 -print-pipeline-passes /dev/null 2>&1 | grep -i "global-isel"
```

## 86.7 Debugging GlobalISel

### 86.7.1 Dumping the Machine Function

Each GlobalISel pass can dump the `MachineFunction` before and after:

```bash
# Print MIR after each GlobalISel pass:
llc -mtriple=aarch64 -global-isel -print-after-all input.ll 2>&1 | less

# Print only after IRTranslator:
llc -mtriple=aarch64 -global-isel \
    -print-after=irtranslator input.ll 2>&1

# Print only after Legalizer:
llc -mtriple=aarch64 -global-isel \
    -print-after=legalizer input.ll 2>&1
```

### 86.7.2 Machine IR Text Format

MIR (Machine IR) is the text format for `MachineFunction`. It is used for testing and debugging:

```llvm
# example.mir
---
name: test_add
body: |
  bb.0:
    liveins: $x0, $x1
    %0:_(s64) = COPY $x0
    %1:_(s64) = COPY $x1
    %2:_(s64) = G_ADD %0, %1
    $x0 = COPY %2:_(s64)
    RET_ReallyLR implicit $x0
...
```

```bash
# Run just the Legalizer on MIR input:
llc -mtriple=aarch64 -global-isel -run-pass=legalizer example.mir -o -

# Verify the MIR is valid after a pass:
llc -mtriple=aarch64 -global-isel -run-pass=legalizer \
    -verify-machineinstrs example.mir
```

### 86.7.3 GlobalISel Observers

The `GISelObserver` mechanism allows inspection of every instruction created or erased during GlobalISel:

```cpp
struct DebugObserver : public GISelObserver {
  void changingInstr(MachineInstr &MI) override {
    dbgs() << "About to change: "; MI.dump();
  }
  void changedInstr(MachineInstr &MI) override {
    dbgs() << "Changed to: "; MI.dump();
  }
  void erasingInstr(MachineInstr &MI) override {
    dbgs() << "Erasing: "; MI.dump();
  }
};
```

The Legalizer and Combiner passes accept observers for diagnostic purposes.

### 86.7.4 FileCheck Testing

GlobalISel transformations are tested with LLVM's FileCheck infrastructure:

```bash
# Run a test:
llvm-lit llvm/test/CodeGen/AArch64/GlobalISel/add.ll

# A typical test file:
# RUN: llc -mtriple=aarch64 -global-isel -verify-machineinstrs \
# RUN:     -stop-after=legalizer %s -o - | FileCheck %s
# CHECK: G_SEXT
# CHECK: G_ADD
# CHECK: G_TRUNC
```

## 86.8 Writing a Target GlobalISel Backend

### 86.8.1 Required Components

To add GlobalISel support to a new target:

1. **`CallLowering` subclass**: implement `lowerFormalArguments`, `lowerReturn`, `lowerCall` for the target's ABI.
2. **`LegalizerInfo` subclass**: declare which `(opcode, LLT)` pairs are legal.
3. **`RegisterBankInfo` subclass**: define register banks and `InstructionMappings`.
4. **`InstructionSelector` subclass**: implement `select()`, driven by TableGen `GINodeEquiv` and patterns.
5. **`GlobalISel/Combiner` rules**: add `GICombineRule` entries for target-specific optimizations.

### 86.8.2 Integrating with TargetPassConfig

```cpp
// MyTargetPassConfig.cpp
void MyTargetPassConfig::addISelPasses() {
  if (getOptLevel() == CodeGenOpt::None) {
    // Use GlobalISel at -O0:
    addPass(createIRTranslatorPass());
    addPreLegalizeMachineIRPass(); // optional combiner
    addPass(createLegalizerPass());
    addPass(createRegBankSelectPass());
    addPass(createInstructionSelectPass());
    return;
  }
  // Fall back to SelectionDAG at -O1+:
  TargetPassConfig::addISelPasses();
}
```

## Chapter Summary

- GlobalISel operates on the entire function (not just one basic block), using Generic MachineIR (`G_*` opcodes) with `LLT` types. This enables cross-block optimizations during instruction selection.

- The four-stage pipeline: `IRTranslator` (IR → G_*), `Legalizer` (make G_* operations target-legal), `RegBankSelect` (assign register banks), `InstructionSelect` (G_* → target instructions).

- `IRTranslator` maps LLVM IR instructions to `G_*` MachineInstr objects using `MachineIRBuilder`. PHI nodes, calls, and returns use dedicated lowering infrastructure (`CallLowering`).

- `LegalizerInfo` declares legality via a fluent API: `legalFor`, `widenScalarToNextPow2`, `clampNumElements`, etc. `LegalizeAction` values include `Legal`, `WidenScalar`, `NarrowScalar`, `MoreElements`, `FewerElements`, `Lower`, and `Libcall`.

- `LegalizerHelper` implements the actual transformations: `widenScalar` inserts `G_SEXT`/`G_ZEXT` + `G_TRUNC`; `fewerElementsVector` inserts `G_UNMERGE_VALUES` + per-half operations + `G_MERGE_VALUES`.

- `RegBankSelect` assigns each virtual register to a register bank (GPR, FPR, etc.) by choosing the `InstructionMapping` with minimum cost. Cross-bank copies are inserted automatically.

- `InstructionSelect` is driven by TableGen `GINodeEquiv` mappings and `Pat` patterns, generating a `select()` function. GlobalISel combiners use the `GICombineRule` DSL to write declarative and C++-gated optimization rules.

- GlobalISel can fall back to SelectionDAG per-function when it cannot select an instruction, controlled by `-global-isel-abort=0/1/2`.

- Debugging relies on `-print-after=legalizer`, MIR text format, `-verify-machineinstrs`, and `GISelObserver` hooks for tracing instruction creation and erasure.
