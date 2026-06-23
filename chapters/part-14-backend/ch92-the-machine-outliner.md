# Chapter 92 — The Machine Outliner

*Part XIV — The Backend*

Code size is a critical metric in embedded systems, mobile applications, and any environment where instruction cache footprint determines performance. The Machine Outliner addresses code size by detecting repeated sequences of machine instructions across functions and replacing them with calls to a shared outlined function. Unlike IR-level function merging, the Machine Outliner operates after register allocation, where the exact instruction sequences are known, making its size estimates accurate and its transformations safe. This chapter covers the suffix tree algorithm used to find repeated sequences, the cost model for deciding what to outline, the per-target hooks that govern correctness, and the practical impact of outlining in AArch64 production code.

## Table of Contents

- [92.1 The Machine Outliner Pass](#921-the-machine-outliner-pass)
  - [92.1.1 Pass Placement in the Pipeline](#9211-pass-placement-in-the-pipeline)
  - [92.1.2 High-Level Algorithm](#9212-high-level-algorithm)
- [92.2 Suffix Tree Construction](#922-suffix-tree-construction)
  - [92.2.1 Instruction Tokenisation](#9221-instruction-tokenisation)
  - [92.2.2 The Suffix Tree](#9222-the-suffix-tree)
- [92.3 Candidate Selection](#923-candidate-selection)
  - [92.3.1 Benefit Formula](#9231-benefit-formula)
  - [92.3.2 Greedy Selection](#9232-greedy-selection)
- [92.4 Target Hooks](#924-target-hooks)
  - [92.4.1 MachineOutlinerInfo](#9241-machineoutlinerinfo)
  - [92.4.2 getOutliningType](#9242-getoutliningtype)
  - [92.4.3 buildOutlinedFrame](#9243-buildoutlinedframe)
- [92.5 AArch64 Outlining](#925-aarch64-outlining)
  - [92.5.1 Link Register Saving](#9251-link-register-saving)
  - [92.5.2 BTI Landing Pads](#9252-bti-landing-pads)
  - [92.5.3 PC-Relative Addressing Constraints](#9253-pc-relative-addressing-constraints)
- [92.6 Size vs. Performance Trade-offs](#926-size-vs-performance-trade-offs)
  - [92.6.1 Optimisation Level Integration](#9261-optimisation-level-integration)
  - [92.6.2 Interaction with LTO](#9262-interaction-with-lto)
  - [92.6.3 Interaction with Profile Data](#9263-interaction-with-profile-data)
- [IR-Level Outlining: IROutliner and IRSimilarityIdentifier](#ir-level-outlining-iroutliner-and-irsimilarityidentifier)
  - [Distinction from MachineOutliner](#distinction-from-machineoutliner)
  - [IRSimilarityIdentifier](#irsimilarityidentifier)
    - [Canonical Form and Hashing](#canonical-form-and-hashing)
    - [Suffix-Tree-Based Sequence Matching](#suffix-tree-based-sequence-matching)
    - [Cost Model and Minimum Thresholds](#cost-model-and-minimum-thresholds)
  - [IROutliner Pass](#iroutliner-pass)
    - [Region Requirements: Single-Entry/Single-Exit](#region-requirements-single-entrysingle-exit)
    - [Phi Node Handling](#phi-node-handling)
    - [Argument Passing and Return Value Extraction](#argument-passing-and-return-value-extraction)
    - [Output: One Outlined Function per Similarity Class](#output-one-outlined-function-per-similarity-class)
  - [Code Size Reduction in Practice](#code-size-reduction-in-practice)
  - [Enabling IROutliner](#enabling-iroutliner)
  - [Interaction with IPO Passes](#interaction-with-ipo-passes)
  - [When to Prefer IROutliner over MachineOutliner](#when-to-prefer-iroutliner-over-machineoutliner)
- [Chapter Summary](#chapter-summary)

---

## 92.1 The Machine Outliner Pass

`MachineOutliner` (`llvm/lib/CodeGen/MachineOutliner.cpp`) is a `ModuleMachinePass` — it operates on all `MachineFunction`s in a module simultaneously, since finding repeated sequences requires a global view. The pass runs after register allocation and frame lowering, so it sees the final physical register assignments and stack frame structure.

### 92.1.1 Pass Placement in the Pipeline

The Machine Outliner runs in the `post-RA` pass sequence, after `PrologueEpilogueInserter` (frame lowering) and before `AsmPrinter`. This placement is deliberate:

- After register allocation: physical register assignments are final, so the instruction sequences that will be outlined are precisely known.
- After frame lowering: stack frame layout is final, so the outliner can correctly reason about `sp`-relative addresses.
- Before AsmPrinter: the outlined functions must be created as `MachineFunction` objects before emission.

```bash
# Check pass order around outlining:
llc -mtriple=aarch64-unknown-linux-gnu \
    --print-pipeline-passes -o /dev/null input.ll 2>&1 | \
    grep -A 5 -B 5 "machine-outliner"
```

### 92.1.2 High-Level Algorithm

```
1. Collect all MachineInstrs from all functions into a flat sequence
2. Replace each instruction with an integer opcode "token"
3. Build a suffix tree over the token sequence
4. Find all repeated substrings (candidates) in the suffix tree
5. For each candidate, compute profitability:
      benefit = (occurrences - 1) × caller_overhead + outlined_function_size_savings
6. Select profitable candidates (greedily, largest benefit first)
7. For each selected candidate:
   a. Create a new outlined MachineFunction
   b. Replace each occurrence with a CALL to the outlined function
   c. Insert function prologue/epilogue as needed
```

## 92.2 Suffix Tree Construction

Finding all repeated substrings efficiently is the core algorithmic challenge. `MachineOutliner` uses Ukkonen's algorithm to construct a suffix tree in O(n) time and space over the token sequence of all instructions.

### 92.2.1 Instruction Tokenisation

Each `MachineInstr` is assigned an integer token. Instructions that cannot be outlined receive a special "illegal" token that acts as a separator — it ensures that no candidate spans an outlinable and a non-outlinable instruction:

```cpp
// MachineOutliner.cpp: InstructionMapper
struct InstructionMapper {
  DenseMap<MachineInstr*, unsigned, MIHasher> InstrToID;
  unsigned IllegalInstrNumber;  // token for un-outlinable instructions
  unsigned InvisibleInstrNumber; // token for debug instructions

  unsigned mapToIllegal(MachineInstr &MI) { return IllegalInstrNumber; }
  unsigned mapToLegal(MachineInstr &MI) {
    // Assign a canonical ID based on opcode + operand structure:
    auto [It, Inserted] = InstrToID.try_emplace(&MI, CurrentID);
    if (Inserted) CurrentID++;
    return It->second;
  }
};
```

Two instructions with different opcodes get different tokens. Two instructions with the same opcode but different operands (e.g., different immediate values) also get different tokens, ensuring that only truly identical sequences are candidates.

### 92.2.2 The Suffix Tree

`SuffixTree` (`llvm/lib/Support/SuffixTree.cpp`) constructs a compressed suffix tree (also known as a Patricia trie or Ukkonen tree). Every internal node with at least two leaf children represents a repeated substring. The `SuffixTree::RepeatedSubstringIterator` traverses these internal nodes in order of occurrence count and length.

```cpp
// Iterating over repeated substrings:
SuffixTree ST(Mapper.UnsignedVec);
for (SuffixTree::RepeatedSubstring &RS : ST) {
  unsigned Length     = RS.Length;
  unsigned Occurrences = RS.StartIndices.size();
  dbgs() << "Repeated substring length=" << Length
         << " occurrences=" << Occurrences << "\n";
}
```

The suffix tree guarantees that every maximal repeated substring is found. The total number of repeated substrings is O(n²) in the worst case, but suffix tree traversal visits them all in O(n) total time.

## 92.3 Candidate Selection

Not every repeated substring is profitable to outline. The outliner computes a benefit score and only proceeds with candidates above a threshold.

### 92.3.1 Benefit Formula

```
OutlinedFunctionBytes = (Length × avg_instr_size) + prologue_size + epilogue_size

CallOverhead = call_instr_size + (LR_save_size if needed)

TotalSavedBytes = (occurrences - 1) × (Length × avg_instr_size)
                  - occurrences × CallOverhead
                  - OutlinedFunctionBytes
```

A candidate is profitable if `TotalSavedBytes > 0`. The call overhead term accounts for the fact that each occurrence must insert a `BL`/`CALL` instruction in place of the outlined sequence, and the outlined function itself takes space.

### 92.3.2 Greedy Selection

When multiple candidates overlap (an instruction can be part of at most one outlined region), the outliner uses a greedy algorithm: sort candidates by `TotalSavedBytes` (descending) and select each candidate that doesn't overlap with an already-selected candidate. This is a weighted interval scheduling problem; the greedy approach is not optimal but is fast enough for production use.

```cpp
// MachineOutliner.cpp: findCandidates() and pruneOverlaps()
void MachineOutliner::pruneOverlaps(
    std::vector<OutlinedFunction> &FunctionList) {
  // Sort by benefit, then remove any candidate whose occurrence range
  // overlaps with a higher-benefit candidate's range:
  std::stable_sort(FunctionList.begin(), FunctionList.end(),
    [](const OutlinedFunction &A, const OutlinedFunction &B) {
      return A.getBenefit() > B.getBenefit();
    });
  // Mark overlapping candidates as invalid...
}
```

## 92.4 Target Hooks

Correctness of outlining depends critically on the target. `MachineOutliner` consults the target's `TargetInstrInfo` for every instruction:

### 92.4.1 MachineOutlinerInfo

Each outlined candidate region is associated with a `MachineOutlinerInfo` struct that the target populates:

```cpp
// llvm/include/llvm/CodeGen/TargetInstrInfo.h
struct MachineOutlinerInfo {
  unsigned SequenceSize;      // bytes in the candidate sequence
  unsigned BranchTailCallSize;// branch instruction size (for tail calls)
  unsigned CallOverhead;      // call instruction size + any LR save
  unsigned FrameOverhead;     // prologue+epilogue added to outlined func
  unsigned FrameConstructionID; // identifies the frame type required
};
```

Targets return this information via:

```cpp
virtual MachineOutlinerInfo getOutliningCandidateInfo(
    const MachineModuleInfo &MMI,
    OutlinedFunction &OF,
    bool TailCall) const;
```

### 92.4.2 getOutliningType

For each `MachineInstr`, the target returns its outlining legality:

```cpp
virtual outliner::InstrType getOutliningType(
    const MachineModuleInfo &MMI,
    MachineInstr &MI, unsigned Flags) const;
```

The return values:

| Type | Meaning |
|---|---|
| `Legal` | Instruction can be outlined |
| `Illegal` | Cannot be outlined; acts as separator token |
| `Invisible` | Not counted in sequence length (e.g., debug instructions) |

Instructions that are `Illegal` include:
- Calls to functions that clobber the link register (LR/RA) without saving it
- Instructions with unique relocation requirements (e.g., `ADRP` that must stay within 4 KB of its paired `ADD`)
- Frame setup/teardown instructions
- Instructions that reference frame indices (stack layout may differ between caller and outlined function)

### 92.4.3 buildOutlinedFrame

After selecting candidates, the target is responsible for constructing the outlined function's frame:

```cpp
virtual void buildOutlinedFrame(
    MachineBasicBlock &MBB,
    MachineFunction &MF,
    const outliner::OutlinedFunction &OF) const;
```

This hook adds the prologue (save LR if needed) and epilogue (restore LR, return instruction) to the outlined function's entry block.

## 92.5 AArch64 Outlining

AArch64 is the primary target for the Machine Outliner, and its implementation (`llvm/lib/Target/AArch64/AArch64InstrInfo.cpp`) illustrates the correctness challenges clearly.

### 92.5.1 Link Register Saving

AArch64's calling convention uses `X30` (the link register, LR) to hold the return address. When outlining a sequence, the outlined function must save and restore `X30`:

- If the outlined sequence contains no calls: `X30` can be preserved via a tail call (branch to `X30`) or by storing/restoring it via the stack.
- If the outlined sequence contains calls: `X30` is clobbered, so the outlined function must save it to a stack slot in its prologue.

The AArch64 backend recognizes three outline frame types:
1. **No LR save** (`NoLRSave`): outlined function is a leaf; ends with `RET`
2. **LR save to register** (`SaveRestoreLR`): uses a scratch register to preserve LR
3. **LR save to stack** (`SaveReturnAddress`): `STR X30, [SP, #-16]!` in prologue, `LDR X30, [SP], #16` in epilogue

### 92.5.2 BTI Landing Pads

When Branch Target Identification (BTI) is enabled (`-mbranch-protection=bti`), every call target must begin with a `BTI C` instruction. Outlined functions must start with `BTI C` when BTI is active:

```cpp
// AArch64InstrInfo.cpp: buildOutlinedFrame() with BTI
if (MF.getInfo<AArch64FunctionInfo>()->branchTargetEnforcement()) {
  BuildMI(MBB, MBB.begin(), DebugLoc(), TII->get(AArch64::HINT))
    .addImm(34); // HINT #34 = BTI C
}
```

### 92.5.3 PC-Relative Addressing Constraints

AArch64's `ADRP`/`ADD` pair for loading symbol addresses requires the two instructions to be within 4 KB of each other (the `ADRP` page offset applies to the address of `ADRP` itself). Outlining an `ADRP` away from its paired `ADD` into a different function that may be placed far away in the binary would produce incorrect addresses.

AArch64's `getOutliningType()` returns `Illegal` for `ADRP` instructions (and `ADRP`-like instructions such as `LDR x0, [x0, :got_lo12:]` sequences) to prevent this:

```cpp
// AArch64InstrInfo.cpp: getOutliningType() (simplified)
case AArch64::ADRP:
  return outliner::InstrType::Illegal;
case AArch64::ADDXri:
  // Illegal if previous instruction was ADRP (they must stay together):
  if (PrevMI && PrevMI->getOpcode() == AArch64::ADRP)
    return outliner::InstrType::Illegal;
  break;
```

## 92.6 Size vs. Performance Trade-offs

The Machine Outliner is a code-size optimisation. Outlined functions introduce function call overhead at each occurrence:

- A `BL` (branch-and-link) instruction: 4 bytes on AArch64, plus the taken-branch cost (typically 1–3 cycles on modern out-of-order cores)
- LR save/restore if the outlined function is not a leaf: 8–16 bytes of prologue/epilogue plus memory access latency

### 92.6.1 Optimisation Level Integration

The outliner is enabled by default at optimisation levels that prioritise size:

| Flag | Outliner Enabled? | Notes |
|---|---|---|
| `-Oz` | Yes | Aggressive size reduction |
| `-Os` | Yes | Moderate size reduction |
| `-O2` | No | Performance preferred |
| `-O3` | No | Performance preferred |
| `-O2 -mllvm -enable-machine-outliner` | Yes | Explicit override |

The threshold for profitability is also adjusted by optimisation level: at `-Oz`, even single-byte savings are considered profitable; at `-Os`, a minimum benefit of 4 bytes is required.

### 92.6.2 Interaction with LTO

When Link-Time Optimisation is active, the Machine Outliner can operate across translation unit boundaries. The `ThinLTO` backend runs the outliner as a module pass over the combined IR-to-MachineFunction stream, giving it visibility into repeated sequences that span separately compiled `.o` files. This significantly increases the number of outlinable candidates — common library patterns (e.g., error handling prologues, container bounds checks) that appear in many functions become consolidatable.

```bash
# LTO with outlining at Oz:
clang -Oz -flto=thin -mllvm -enable-machine-outliner \
      -o output input1.c input2.c input3.c
```

### 92.6.3 Interaction with Profile Data

When PGO data is available, the outliner can weight its profitability calculation by actual execution frequency. Sequences that appear frequently in hot paths are penalised (outlined only if the saved bytes justify the call overhead at the observed frequency); sequences in cold code are outlined aggressively. This is controlled by `BlockFrequencyInfo` in the profitability calculation.

## IR-Level Outlining: IROutliner and IRSimilarityIdentifier

The Machine Outliner described above operates on `MachineInstr` sequences after register allocation — where physical registers, stack frame layout, and exact instruction encodings are fixed. LLVM provides a complementary outliner that works earlier in the pipeline, at the LLVM IR level: `IROutliner`, driven by `IRSimilarityIdentifier`. The two passes share the same high-level goal (deduplicate code, reduce binary size) but occupy fundamentally different positions in the compilation pipeline and reason at different levels of abstraction.

### Distinction from MachineOutliner

| Property | MachineOutliner | IROutliner |
|---|---|---|
| Operates on | `MachineInstr` (post-RA) | `Instruction` (LLVM IR, pre-RA) |
| Pass placement | After `PrologueEpilogueInserter` | Before register allocation; in IR pass pipeline |
| Similarity metric | Exact opcode + operand token match | Structural similarity: opcode + type; ignores SSA names |
| Interaction with IPO | None (post-IPO) | Interacts with inliner, function merging, globalopt |
| Target sensitivity | High (LR saving, BTI, ADRP, etc.) | None (ISA-independent IR) |
| Typical savings | 3–8% at `-Oz` on AArch64 | 5–15% at `-Oz` on large, uniform codebases |
| Pipeline file | `llvm/lib/CodeGen/MachineOutliner.cpp` | `llvm/lib/Transforms/IPO/IROutliner.cpp` |

`IROutliner` trades the precision of post-RA machine code knowledge for ISA independence and earlier pipeline placement. An identical IR sequence outlined once produces a single outlined function that is then compiled to machine code by the backend; MachineOutliner never sees the relationship between the two call sites because they already materialized into identical instruction streams in separate functions.

When binary portability across ISAs matters — for example, a codebase compiled to both AArch64 and RISC-V — `IROutliner` amortizes the outlining work in the IR representation that is shared across targets. `MachineOutliner` must then run per-target and performs its own independent analysis.

### IRSimilarityIdentifier

`IRSimilarityIdentifier` (`llvm/include/llvm/Analysis/IRSimilarityIdentifier.h`, `llvm/lib/Analysis/IRSimilarityIdentifier.cpp`) detects structurally similar instruction sequences across functions. It does not require sequences to be textually identical; it requires them to be *canonically equivalent*: same opcode sequence, same type structure, same use–def structure within the region, and consistent operand mapping.

#### Canonical Form and Hashing

Each `Instruction` in a candidate sequence is reduced to a canonical descriptor that ignores SSA value names and considers only:

1. **Opcode**: the `Instruction::getOpcode()` value (e.g., `Instruction::Add`, `Instruction::Load`).
2. **Type of each operand and result**: `Type*` pointers, compared structurally.
3. **Use–def relationships within the region**: whether an operand is defined inside the region (an "internal" value) or comes from outside (an "external" operand). External operands from different instances of the same sequence do not need to match — they will be passed as arguments to the outlined function.

```cpp
// IRSimilarityIdentifier.cpp: IRInstructionData
struct IRInstructionData {
  Instruction *Inst;
  unsigned    Legal;       // false if this instruction cannot be outlined
  SmallVector<Value *, 4> OperVals; // operands in canonical order
  // Comparison operator: checks opcode, types, and operand structure
  bool isSimilar(const IRInstructionData &Other) const;
};
```

Sequences are hashed as chains: a *rolling hash* over the opcode+type descriptor of each instruction position, concatenated across the sequence length. The `IRSimilarityIdentifier` then clusters sequences with matching hash chains:

```cpp
// Simplified view of the similarity detection loop:
IRSimilarityIdentifier IRSI;
SimilarityGroupList &Groups = IRSI.findSimilarity(M);
for (IRSimilarityGroup &Group : Groups) {
  // Each Group holds a set of IRSimilarityCandidate objects:
  // same structure, possibly different operand values.
  for (IRSimilarityCandidate &Cand : Group)
    dbgs() << "Candidate in function: "
           << Cand.getFunction()->getName() << "\n";
}
```

#### Suffix-Tree-Based Sequence Matching

Like `MachineOutliner`, `IRSimilarityIdentifier` uses a suffix tree over the flattened instruction stream to find all maximal repeated substrings efficiently. The suffix tree is built over integer tokens derived from the instruction hashes, then filtered by minimum sequence length (default: 2 instructions) and minimum estimated savings threshold. The `SuffixTree` from `llvm/lib/Support/SuffixTree.cpp` is reused here — the same data structure underpins both outliners.

#### Cost Model and Minimum Thresholds

`IRSimilarityIdentifier` applies two gates before reporting a similarity group as actionable:

1. **Minimum sequence length**: controlled by `-ir-outlining-no-cost` and internal heuristics; sequences shorter than ~3–5 instructions are rarely profitable after argument-passing overhead.
2. **Minimum savings estimate**: for each occurrence, the sequence's instruction count multiplied by a target-independent instruction cost (defaulting to 1 unit per instruction) minus the argument-passing and call overhead (one `call` instruction plus one argument per external operand) must be positive across the group.

### IROutliner Pass

`IROutliner` (`llvm/lib/Transforms/IPO/IROutliner.cpp`) consumes the similarity groups from `IRSimilarityIdentifier` and performs the actual transformation: it extracts each group's shared instruction sequence into a new outlined function and replaces every occurrence with a call.

#### Region Requirements: Single-Entry/Single-Exit

`IROutliner` requires each candidate region to be a *single-entry, single-exit* (SESE) region with respect to control flow:

- **Single entry**: the first instruction in the region is the unique entry point; no edge from outside the region jumps to any instruction in the interior.
- **Single exit**: the last instruction in the region transfers control to exactly one successor outside the region. Mid-region branches that leave the region (early exits, `throw`, `longjmp`) disqualify the sequence.

Regions that contain a `call` to a function with attribute `noinline` or that touch `alloca`s that escape the region are also rejected, since those cannot be cleanly extracted without invalidating the caller's frame layout.

```cpp
// IROutliner.cpp: isCompatibleWithOutlining() (simplified)
bool isCompatibleWithOutlining(IRSimilarityCandidate &Cand) {
  // Check for multi-exit regions:
  BasicBlock *StartBB = Cand.getStartBB();
  BasicBlock *EndBB   = Cand.getEndBB();
  if (StartBB != EndBB) {
    // Ensure no internal block has a successor outside [StartBB..EndBB]:
    for (BasicBlock *BB : Cand.getBlocks())
      for (BasicBlock *Succ : successors(BB))
        if (!Cand.contains(Succ) && Succ != EndBB->getTerminator()...)
          return false;
  }
  return true;
}
```

#### Phi Node Handling

Outlined regions must not begin or end at a `PHINode` boundary. A `PHINode` at the start of a basic block merges values from multiple predecessor edges; splitting that edge would require duplicating the phi logic or constructing a new merge block, which significantly complicates the transformation and rarely yields net savings. `IROutliner` skips any candidate whose start instruction is a `PHINode` or whose single-exit block terminates with a branch that feeds a `PHINode` in the successor.

#### Argument Passing and Return Value Extraction

For each outlined group, `IROutliner` computes:

- **Arguments**: all SSA values defined outside the candidate region but used inside it become function parameters.
- **Return values**: all values defined inside the region that are used outside it must be returned. If there are multiple such values, `IROutliner` packs them into an aggregate return type (a `struct` of the live-out values).

```cpp
// IROutliner.cpp: createFunction()
Function *IROutliner::createFunction(IRSimilarityGroup &Group,
                                     DenseMap<Value*, unsigned> &ArgMap) {
  SmallVector<Type *, 8> ArgTypes;
  // One parameter per external operand (deduplicated across instances):
  for (Value *V : Group.getExternalOperands())
    ArgTypes.push_back(V->getType());
  // Return type: aggregate of live-out values, or void:
  Type *RetTy = buildReturnType(Group.getLiveOutValues());
  FunctionType *FTy = FunctionType::get(RetTy, ArgTypes, false);
  return Function::Create(FTy, GlobalValue::InternalLinkage,
                          "outlined_ir_func", M);
}
```

#### Output: One Outlined Function per Similarity Class

`IROutliner` creates exactly one outlined `Function` per similarity group. All occurrences of that group's sequence are replaced by a `CallInst` to that single function, passing the appropriate external operands as arguments and extracting return values:

```llvm
; Before IROutliner: two similar loops
define void @process_a(ptr %arr, i64 %len) {
entry:
  br label %loop
loop:
  %i = phi i64 [ 0, %entry ], [ %i.next, %loop ]
  %p = getelementptr i32, ptr %arr, i64 %i
  %v = load i32, ptr %p, align 4
  %w = mul i32 %v, 7
  store i32 %w, ptr %p, align 4
  %i.next = add i64 %i, 1
  %done = icmp eq i64 %i.next, %len
  br i1 %done, label %exit, label %loop
exit:
  ret void
}

define void @process_b(ptr %buf, i64 %n) {
entry:
  br label %loop
loop:
  %i = phi i64 [ 0, %entry ], [ %i.next, %loop ]
  %p = getelementptr i32, ptr %buf, i64 %i
  %v = load i32, ptr %p, align 4
  %w = mul i32 %v, 7
  store i32 %w, ptr %p, align 4
  %i.next = add i64 %i, 1
  %done = icmp eq i64 %i.next, %n
  br i1 %done, label %exit, label %loop
exit:
  ret void
}
```

`IRSimilarityIdentifier` identifies the GEP + load + mul + store + increment + compare sequence as structurally identical (same opcode/type chain; `%arr`/`%buf` and `%len`/`%n` are external operands that differ between instances). `IROutliner` extracts it:

```llvm
; After IROutliner:
define internal void @outlined_ir_func.0(ptr %0, i64 %1) {
  ; Outlined body: GEP, load, mul, store, loop back-edge
  %p = getelementptr i32, ptr %0, i64 %1
  %v = load i32, ptr %p, align 4
  %w = mul i32 %v, 7
  store i32 %w, ptr %p, align 4
  ret void
}

define void @process_a(ptr %arr, i64 %len) {
entry:
  br label %loop
loop:
  %i = phi i64 [ 0, %entry ], [ %i.next, %loop ]
  call void @outlined_ir_func.0(ptr %arr, i64 %i)
  %i.next = add i64 %i, 1
  %done = icmp eq i64 %i.next, %len
  br i1 %done, label %exit, label %loop
exit:
  ret void
}
; @process_b similarly calls @outlined_ir_func.0
```

### Code Size Reduction in Practice

On large `-Oz` builds with uniform coding patterns — embedded firmware, generated code, or large C++ template instantiations — `IROutliner` typically achieves 5–15% binary size reduction, outperforming `MachineOutliner` on codebases where the similarity is structural rather than byte-for-byte. Template-heavy C++ is a particularly strong case: every instantiation of a template over related types produces near-identical IR that `IRSimilarityIdentifier` can match across translation units.

The savings are sensitive to argument count: each additional live-in operand adds one argument to the call, increasing call overhead. Sequences with five or more distinct external operands rarely yield net savings once the argument-marshaling cost is counted. `IROutliner` respects this in its internal cost model and skips groups where the argument overhead exceeds the instruction savings.

### Enabling IROutliner

`IROutliner` is included in the `-Oz` pipeline automatically in LLVM 22. It runs in the module optimization phase, after early inlining and before the final simplification passes:

```bash
# Run IROutliner explicitly on an IR file:
opt -passes=iroutliner -S input.ll -o output.ll

# Verify which similarity groups were found (debug output):
opt -passes=iroutliner -S input.ll \
    -debug-only=iroutliner 2>&1 | head -40

# IROutliner is included in -Oz automatically via clang:
clang -Oz -o output input.c

# Check the full -Oz pass pipeline to see IROutliner's position:
clang -Oz -### input.c 2>&1 | grep -o 'iroutliner'
opt -O3 --print-pipeline-passes -disable-output /dev/null 2>&1 | \
    tr ',' '\n' | grep -i outliner
```

To disable `IROutliner` while keeping `-Oz` (useful for benchmarking its contribution):

```bash
clang -Oz -mllvm -ir-outlining-no-cost=0 \
      -mllvm -enable-ir-outliner=false -o output input.c
```

### Interaction with IPO Passes

Because `IROutliner` runs pre-register-allocation and within the IR pass pipeline, it interacts with other interprocedural optimizations:

- **Inliner**: outlined functions are marked `internal` and attributed with `noinline` to prevent the inliner from immediately reversing the transformation. If the outlined function is subsequently inlined (e.g., by a forced-inline attribute on the call site), the deduplication is lost.
- **Function merging (`MergeFunctions`)**: `MergeFunctions` handles the case of *identical* functions; `IROutliner` handles *similar sub-sequences within different functions*. They are complementary: run `MergeFunctions` first (it removes identical functions entirely), then `IROutliner` on the remaining structural similarities.
- **GlobalOpt / Dead Argument Elimination**: after outlining, unused arguments in the outlined function (because one operand turned out to be a constant in all instances) can be removed by `DeadArgumentElimination`, improving call overhead.

The canonical `-Oz` pass order in LLVM 22 places `IROutliner` after `MergeFunctions` and before `GlobalDCE`, ensuring these interactions are handled in the correct sequence.

### When to Prefer IROutliner over MachineOutliner

Choose `IROutliner` when:

1. **ISA-independent deduplication is required**: the same source tree is compiled for multiple ISAs; outlining at IR level amortizes the work once rather than once per target.
2. **Template-heavy C++** or generated code produces large identical or near-identical IR blobs across TUs — `IRSimilarityIdentifier`'s opcode+type matching will catch them; `MachineOutliner` may miss them if register allocation produces divergent machine code.
3. **Pre-RA pass order interaction matters**: an outlined IR function benefits from all subsequent per-function optimizations (loop opts, vectorization, instruction selection tuning) applied to the outlined body once rather than N times.
4. **Interaction with LTO**: under `ThinLTO`, `IROutliner` runs over the merged IR module and sees all functions simultaneously, maximizing cross-TU similarity detection without requiring a second post-RA module pass.

Choose `MachineOutliner` when:

1. **Exact byte-level savings are critical**: post-RA sequences encode precise instruction encodings, giving the benefit formula exact byte counts.
2. **Target-specific constraints dominate**: `MachineOutliner`'s per-instruction legality hooks handle AArch64 LR saving, ADRP constraints, and BTI requirements that have no IR-level equivalent.
3. **`IROutliner` was already run** and further per-target deduplication is desired at the machine level.

In practice, production `-Oz` pipelines on AArch64 often run both: `IROutliner` captures structural similarity at the IR level, and `MachineOutliner` then captures any remaining repeated sequences visible only after register allocation and instruction selection.

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **RISC-V Machine Outliner parity with AArch64**: The RISC-V backend's `getOutliningType` and `buildOutlinedFrame` hooks are undergoing refinement (tracked in [D149568](https://reviews.llvm.org/D149568) and follow-on patches) to handle RA (return address) save/restore strategies analogous to AArch64's three-frame-type model, including support for Zicfiss (shadow-stack CFI) landing pads when the extension is enabled.
- **IROutliner + MachineOutliner pipeline ordering study**: An LLVM RFC thread (discourse.llvm.org, late 2025) proposes a formal cost-model study measuring the interaction between IROutliner and MachineOutliner at `-Oz` on AArch64, with the goal of automatically suppressing MachineOutliner for regions already handled by IROutliner to avoid redundant suffix-tree construction overhead.
- **Outlined function section placement under `-ffunction-sections`**: When `-ffunction-sections` places each function in its own ELF section, outlined functions currently land in a default `.text.outlined.*` section. A pending patch adds a heuristic to co-locate the outlined function section with its most frequent call-site's section, reducing inter-section branch penalties on targets with 26-bit branch-range limits.
- **Improved debug-info fidelity for outlined regions**: DWARF location-list generation for outlined machine instructions is incomplete when the call-site and the outlined function body are in different CUs; active work on `DWARFLinker` and the outliner's `DebugInstrRef` integration aims to produce correct `DW_AT_abstract_origin` chains for outlined frames by LLVM 23.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Profile-guided outlining threshold tuning**: Integration with the AutoFDO / BOLT feedback loop is planned to allow the outliner's `TotalSavedBytes` threshold and call-overhead constants to be calibrated per-binary from sampled profiling data, rather than relying on static ISA-average estimates. This aligns with the broader LLVM initiative to make cost models data-driven (see the "learned cost models" RFC on discourse.llvm.org, 2025).
- **Cross-DSO outlining under `-fvisibility-default`**: Current MachineOutliner constraints prohibit outlining across shared-library boundaries because outlined functions cannot have `PLT`-mediated calls. Extending the outliner to emit `local_ifunc` trampolines (ELF) or `__declspec(selectany)` stubs (COFF) would allow cross-DSO deduplication for large frameworks shipping many shared objects with common runtime patterns.
- **Suffix-tree replacement with Aho–Corasick-based indexing**: The `SuffixTree` construction at O(n) time but O(n) memory is a bottleneck for very large ThinLTO modules (10 M+ instructions). Research prototypes (presented at the 2024 LLVM Developers' Meeting) explore Aho–Corasick automata over a rolling-hash alphabet as a drop-in replacement, reducing peak memory by ~40% with equivalent candidate recall.
- **IRSimilarityIdentifier extension to MLIR**: Analogous IR similarity detection for MLIR ops is being prototyped (`mlir-outliner` RFC, MLIR discourse, 2025) to enable ISA-independent outlining of TOSA/LinAlg op sequences before lowering. This would bring Machine-Outliner-style savings to ML compiler pipelines targeting heterogeneous accelerators.

### 5-Year Horizon (Long-Term, by ~2031)

- **Learned outlining policies via reinforcement learning**: Research groups (UC San Diego, ARM Research) are exploring RL agents trained on code-size/performance Pareto frontiers to replace the greedy `pruneOverlaps` heuristic with a policy that jointly optimizes outlining decisions across an entire module, anticipating downstream inliner and scheduler interactions.
- **Outliner-aware linker deduplication (COMDAT generalization)**: A long-term standards-track effort in the ELF ABI working group aims to define a "structural COMDAT" mechanism where the linker, not the compiler, performs final deduplication of identical machine-code sections; this would allow outlining to operate at link time across independently compiled static archives without requiring LTO.
- **Formal verification of outliner correctness**: Extending the Alive2 / MC-Semantics framework to verify that `buildOutlinedFrame` transformations preserve program semantics for all legal LR-save frame types, including interactions with stack unwinding (`.eh_frame`) and sanitizer shadow stacks, addressing a class of correctness bugs historically found only through fuzzing.

---

## Chapter Summary

- `MachineOutliner` operates as a module-level post-RA pass, detecting repeated instruction sequences globally across all functions using a suffix tree.
- Instruction tokenisation maps each `MachineInstr` to an integer; non-outlinable instructions receive an "illegal" token that acts as a sequence separator.
- The suffix tree (built with Ukkonen's O(n) algorithm) enumerates all maximal repeated substrings; the outliner filters them by profitability: `(occurrences-1) × sequence_size - occurrences × call_overhead - outlined_func_overhead > 0`.
- Target hooks (`getOutliningType`, `getOutliningCandidateInfo`, `buildOutlinedFrame`) control per-instruction legality, cost estimation, and frame construction.
- AArch64 is the primary target: the three frame types handle LR-saving strategy, BTI landing pads must be inserted when branch-target enforcement is active, and `ADRP` instructions are illegal to outline due to PC-relative addressing constraints.
- The outliner is enabled at `-Oz` and `-Os`; at `-O2`/`-O3`, the performance cost of the call overhead outweighs the icache benefit unless explicitly overridden.
- Under ThinLTO, the outliner gains cross-TU visibility, substantially increasing the number of candidates in real-world codebases.


---

@copyright jreuben11
