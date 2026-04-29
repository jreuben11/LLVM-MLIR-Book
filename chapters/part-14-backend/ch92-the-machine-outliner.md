# Chapter 92 — The Machine Outliner

*Part XIV — The Backend*

Code size is a critical metric in embedded systems, mobile applications, and any environment where instruction cache footprint determines performance. The Machine Outliner addresses code size by detecting repeated sequences of machine instructions across functions and replacing them with calls to a shared outlined function. Unlike IR-level function merging, the Machine Outliner operates after register allocation, where the exact instruction sequences are known, making its size estimates accurate and its transformations safe. This chapter covers the suffix tree algorithm used to find repeated sequences, the cost model for deciding what to outline, the per-target hooks that govern correctness, and the practical impact of outlining in AArch64 production code.

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
