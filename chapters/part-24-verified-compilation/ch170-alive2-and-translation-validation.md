# Chapter 170 — Alive2 and Translation Validation

*Part XXIV — Verified Compilation*

Proving a compiler correct by construction — as CompCert does — requires immense effort: a full Coq formalization, simulation proofs for every pass, and a memory model that every proof must be consistent with. For LLVM, whose optimization pipeline contains hundreds of passes accumulated over decades of independent development, this approach is not currently feasible. An alternative is **translation validation**: instead of proving the compiler algorithm correct, verify at compile time that the output of each specific compilation is correct, by checking that the transformed IR refines (is at least as defined as) the source IR. Alive2 implements this approach efficiently using Z3, and has proven itself as one of the most effective bug-finding tools in the LLVM ecosystem, discovering hundreds of real miscompilation bugs. This chapter examines the theoretical foundations of Alive2, its refinement semantics, its memory model, the Z3 encoding that makes it practical, and how it integrates into LLVM development.

---

## 170.1 The Miscompilation Problem

### Scale and Frequency

LLVM's bug tracker contains thousands of reports of miscompilations. A systematic study by Yang et al. (CSmith, PLDI 2011) found that at any given snapshot, GCC -O2 and Clang -O2 both have multiple confirmed miscompilation bugs affecting real programs. These are not all obscure corner cases: many involve common patterns like integer arithmetic with signed overflow, load-store forwarding, and loop induction variables.

The scope of "miscompilation" in LLVM is broader than "producing the wrong output for a correct program." It includes:

- **Incorrect flag propagation**: an optimization incorrectly marks a result as `nsw` (no signed wrap) or `nuw` (no unsigned wrap), permitting a later optimization to make a now-invalid assumption.
- **Incorrect fold in InstCombine**: a pattern in `InstCombineCasts.cpp` or `InstCombineCompares.cpp` matches a pattern it should not, producing a semantically different result.
- **Incorrect code motion**: LICM (Loop Invariant Code Motion) or GVN (Global Value Numbering) moves an operation that has side effects (or may trigger UB) past a guard.
- **Incorrect alias assumption**: a pass uses alias information that is overly optimistic, causing an operation to be eliminated when it has observable effects.

### Why Testing Is Insufficient

Testing finds miscompilations by generating inputs, running both source and compiled versions, and comparing outputs. CSmith does this at scale with randomly-generated programs. But:

- Test programs cover only the input space they explore; a miscompilation on a specific input pattern may not be triggered.
- Miscompilations involving UB are particularly hard to test: if the source program invokes UB, both the source and compiled versions can behave arbitrarily, making it hard to know if the compiler "did the right thing."
- Code generators (InstCombine etc.) process millions of distinct patterns across all real programs; random testing samples a tiny fraction.

### Translation Validation as a Solution

**Translation validation** (Pnueli et al., 1998) is a technique where, after compilation, an automatic verifier checks that the transformation performed was semantically valid. The key insight: even if the compiler is not proved correct in advance, we can check the output. If the check passes, we have a proof (for this specific input program) that the compilation was correct; if it fails, a counterexample is generated showing the miscompilation.

Translation validation does not require:
- A Coq formalization of the compiler.
- Proofs of individual optimization algorithms.
- Any modification to the existing compiler.

It requires only:
- A formal semantics for the IR (LLVM IR in this case).
- An automated decision procedure (SMT solver).
- The before and after IR for each transformation.

---

## 170.2 Alive (Original)

### Motivation and Design

Alive (Lopes, Menendez, Nagarakatte, Regehr; PLDI 2015) was the first practical tool for verifying LLVM peephole optimizations automatically. It focused on InstCombine — LLVM's workhorse peephole optimization pass that contains ~20,000 lines of hand-written pattern-matching code.

Alive's approach:
1. The user writes an optimization in a human-readable specification language.
2. Alive automatically encodes the specification as a Z3 SMT query.
3. Z3 checks if the optimization is valid (UNSAT = valid, SAT = counterexample).

The specification language is domain-specific: it uses LLVM IR syntax augmented with preconditions (`Pre:`) and pattern variables.

### Example Alive Specification

An InstCombine rule: `(x + y) - x → y` (with appropriate conditions):

```
Name: sub add
Pre: true
%add = add %x, %y
%r = sub %add, %x
=>
%r = %y
```

With overflow flags:

```
Name: sub add nsw
Pre: true
%add = add nsw %x, %y
%r = sub %add, %x
=>
%r = %y
```

For this second rule, Alive checks:
- If neither `add` nor `sub` trigger signed overflow (i.e., the source is defined), does the target produce the same result?
- Can the target produce `poison` when the source does not?

Alive's Z3 encoding: let `x` and `y` be 32-bit bitvector variables. The source computes `bvadd(x, y) - x`; the target computes `y`. These are equal for all bitvectors by bitvector algebra — Z3 instantly verifies this.

For the `nsw` case: the source has `poison` when `bvadd(x, y)` overflows (signed). The target `y` is never poison. So the target is strictly less undefined than the source — the transformation is valid (the target is more defined, which is safe; it can only make previously-undefined programs better-defined, not worse).

### Alive's Impact

Alive verified 334 InstCombine transformations in its initial paper, finding 8 incorrect ones. The incorrect transformations were genuine bugs: they produced incorrect results for specific inputs. All 8 were fixed in LLVM before the paper was published.

The original Alive tool was written in Python, operated on text descriptions of optimizations, and was not integrated into LLVM's build system. It was a research tool, not an automated gate in the CI pipeline.

---

## 170.3 Alive2 Architecture

### Rewrite in C++

Alive2 (Lopes et al.; from 2019) is a ground-up rewrite of Alive in C++. Key improvements:
- Verifies **full LLVM IR transformations**, not just abstract specifications.
- Can be **loaded as a Clang plugin** to automatically intercept and verify every transformation applied by `opt`.
- Handles a much larger fragment of LLVM IR: all integer operations, floating-point (QF_FP), memory (with a custom model), and `undef`/`poison` semantics.
- Uses Z3 directly via its C++ API.

### Components

```
alive2/
  ir/           -- LLVM IR representation (independent of LLVM's internal types)
    function.h  -- Function: the unit of verification
    instr.h     -- Instruction types (BinOp, ICmp, Load, Store, ...)
    memory.h    -- Memory model
    value.h     -- Value hierarchy (Constant, Register, ...)
    state.h     -- Symbolic execution state
  smt/          -- Z3 interface
    ctx.h       -- SMT context (wraps Z3's z3::context)
    expr.h      -- SMT expression builders
    solver.h    -- Solver wrapper
  tools/
    alive-tv.cpp  -- translation validator: alive-tv before.ll after.ll
    alive.cpp     -- standalone verifier
```

### Intercept Mode

The most powerful mode: loaded as a Clang or `opt` pass that intercepts every IR transformation:

```bash
opt -O2 -load alive2.so input.ll -o output.ll
# Alive2 checks every transformation applied by -O2
```

For each pass that modifies the IR, Alive2:
1. Captures the IR before the pass (the "source").
2. Captures the IR after the pass (the "target").
3. Runs the refinement check on the changed functions.
4. Reports any violation found.

This automatic interception mode requires no human annotation. It found hundreds of bugs in LLVM when first deployed in 2019, with zero effort beyond running it on LLVM's existing test suite.

---

## 170.4 The Refinement Relation

### Behavioral Semantics

Alive2's semantics assigns a set of possible behaviors to each LLVM IR function. A **behavior** is a pair:
- A return value (a concrete value, or poison, or UB).
- A memory state (the observable effects on memory after the function returns).

For functions that call other functions, the model uses uninterpreted functions (calls whose behavior is abstracted).

### The Refinement Ordering

Alive2 checks that the target **refines** the source: `refines(tgt, src)` means the target's behaviors are a subset of the source's behaviors. Formally, for each possible input:

```
refines(tgt, src) iff
  ∀ input.
    (1) If src(input) is UB → tgt(input) can be anything (UB allows any behavior)
    (2) If src(input) produces a non-poison value v →
          tgt(input) must also produce v (same value, no UB)
    (3) If src(input) produces poison →
          tgt(input) may produce any value or poison (more undefined is ok)
    (4) Memory effects: if src produces observable memory changes, tgt must too
```

This refinement relation models the key insight: **undefined behavior is permission for the compiler to do anything**. A source program that invokes UB gives the compiler the right to produce any output. The compiler is not allowed to introduce UB where the source had none (case 2).

### The Direction Matters

Refinement is **directional**: `refines(tgt, src)` and `refines(src, tgt)` are different claims. Alive2 checks only `refines(tgt, src)` — the target must be at least as defined as the source.

This means:
- A transformation that makes a program more defined (e.g., replacing `undef` with a specific value) is valid.
- A transformation that introduces UB where none existed is not valid.
- Two transformations that make the same program "more specific" in different ways may both be valid, but their composition may not be (one may restrict the other).

In LLVM terms: the optimizer may narrow UB (replace `undef` with a constant, remove a branch through UB), but it may not widen UB (turn a defined operation into one that may trigger UB under some inputs the source would not).

### Refinement with Poison

Poison values participate in the refinement:

```
refines_value(tgt_val, src_val) iff
  src_val = poison  →  tgt_val = anything
  src_val = v (concrete)  →  tgt_val = v (same concrete value, not poison)
```

So poison is "more undefined" than concrete values: the target may produce poison where the source produced a concrete value only if the source later uses that value in a way that would trigger UB (poison propagates to UB when used as a branch condition, function argument with `nonnull`, etc.).

This is a subtle but important distinction. Consider:
```llvm
; Source
%r = add nsw i32 %x, 1     ; poison if overflow
; Target  
%r = add i32 %x, 1         ; always concrete (wraps)
```

This transformation (`nsw` removal) is **not valid** via Alive2's refinement: the source produces `poison` when `%x = INT_MAX`, but the target produces `INT_MIN` (concrete). The target is more defined than the source — it turns a `poison` into a concrete value — but **which concrete value** is not constrained by the source semantics. The target must produce the *same* concrete value as the source; since the source has no concrete value for this case (it produces poison), any concrete target value would be wrong. Therefore the transformation narrows UB (turns poison into a value) without the constraint being satisfiable — it's invalid.

---

## 170.5 Memory Model in Alive2

### Abstract Memory Representation

Alive2's memory model (`ir/memory.h`) is a layer between LLVM's abstract block-based memory model and the concrete SMT representation. Key concepts:

**Memory blocks**: each heap/stack allocation is a block with:
- An abstract block ID (a Z3 bitvector variable).
- A size (concrete integer, or SMT variable if allocation size is dynamic).
- A liveness flag: whether the block is currently allocated (freed blocks cannot be accessed).
- Whether it is a heap, stack, or global block (affects aliasing rules).

**Pointer representation**: a pointer is a pair `(block_id, offset)` encoded as Z3 bitvectors. Two pointers are equal iff they have the same block_id and offset. Two pointers alias iff they may access overlapping byte ranges within their blocks.

**Byte-level store**: each block's contents are modeled as a Z3 array `Array(index: bv64, value: bv8)` — an array of bytes indexed by offset. `load i32, ptr %p` is modeled as: read four consecutive bytes from `p`'s block starting at `p`'s offset, interpret as a little-endian (or big-endian) 32-bit integer.

### Alias Analysis Integration

Alive2 integrates with LLVM's alias analysis to generate more precise memory constraints. For each pair of pointers:
- **NoAlias**: the SMT constraint `block_id₁ ≠ block_id₂ ∨ offset₁ + size₁ ≤ offset₂ ∨ offset₂ + size₂ ≤ offset₁` is added.
- **MustAlias**: `block_id₁ = block_id₂ ∧ offset₁ = offset₂`.
- **MayAlias**: no constraint.

When an optimization like GVN or alias analysis-based load forwarding is verified, Alive2 checks that the alias information used is consistent with the memory constraints. If the optimization assumes `NoAlias` between two pointers, Alive2 checks that this assumption is valid for all inputs — if not, the optimization is incorrect.

### Undefined Behavior in Memory Operations

Memory UB in LLVM IR:
- Loading from an invalid (null, freed, out-of-bounds) pointer: UB.
- Storing to an invalid pointer: UB.
- GEP `inbounds` where the result is outside the allocated object: UB (poison).
- Integer-to-pointer cast without provenance: UB (under the strict provenance memory model).

Alive2 encodes these UB conditions explicitly. For `%r = load i32, ptr %p`:
- The SMT encoding includes: `valid_pointer(p, 4) ∨ UB`
- If `valid_pointer(p, 4)` is false (pointer invalid for a 4-byte read), the instruction triggers UB.
- The UB flag propagates to the function's behavior: if UB is triggered, the function behavior is unconstrained.

This precise UB encoding is what makes Alive2 correct: it never flags a valid transformation as incorrect because the UB conditions are faithfully modeled.

---

## 170.6 Finding Real Bugs with Alive2

### Systematic Deployment

When Alive2 was first systematically run against LLVM's optimization passes in 2019–2020, it found bugs at a rate of several per week. The most productive areas:

**InstCombine** (`llvm/lib/Transforms/InstCombine/`): contains thousands of hand-written patterns, each of which is a potential Alive2 test case. Alive2 found incorrect fold rules for:
- `icmp` with arithmetic overflow: a rule that folded `icmp slt (add nsw %x, c1), c2` incorrectly when `c1` and `c2` were specific values.
- `select` with `undef`: a rule that propagated `undef` through `select` in a way inconsistent with `poison` semantics.
- Cast combinations: incorrect folds of sequences like `trunc (sext ...)` or `zext (trunc ...)`.

**GVN (Global Value Numbering)**: found cases where load elimination was performed without adequately checking that no store between the two loads could have modified the loaded address.

**LICM (Loop Invariant Code Motion)**: found cases where a potentially-UB instruction (e.g., a division that could divide by zero) was hoisted out of a loop, converting a "may UB on some iterations" to "always UB."

**SROA (Scalar Replacement of Aggregates)**: incorrect handling of `undef` bytes when splitting a struct into its individual fields.

### Notable Bug Classes

**Incorrect nsw/nuw flag promotion**: a pass concludes that an addition result cannot overflow (based on some analysis) and adds the `nsw` flag. But the analysis is too optimistic — there exists an input for which the result does overflow. Alive2 finds the counterexample.

**Incorrect icmp fold**: a rule rewrites `icmp ult %x, C` to `false` when some condition is believed to imply `%x ≥ C`. If the condition is wrong, the fold is incorrect. Alive2 generates the counterexample `%x = C-1`.

**Transformation of code with side effects**: moving a load that aliases a store, causing the load to read a different value.

### Bug Report Format

When Alive2 finds a bug, it generates:
```
ERROR: Target is more undefined than Source

Example:
  %x = i32 0x7FFFFFFF (2147483647)
  %y = i32 0x00000001 (1)

Source:
  %add = add nsw i32 %x, %y  ; = 2147483648 (UB: overflow with nsw)
  ret i32 %add

Target:
  %add = add i32 %x, %y  ; = -2147483648 (wraps; concrete value)
  ret i32 %add

SOURCE returns: poison (due to nsw overflow)
TARGET returns: i32 -2147483648

Transformation is not valid!
```

This report is directly actionable: the input, the source behavior (poison), and the target behavior (concrete wrong value) are all shown.

---

## 170.7 Practical Translation Validation

### alive-tv: Before/After Validation

The `alive-tv` tool validates a before/after LLVM IR pair:

```bash
# Write before.ll and after.ll with corresponding functions
alive-tv before.ll after.ll
```

Output on success:
```
Processing function 'foo'
Transformation seems to be correct!
```

Output on failure:
```
Processing function 'foo'
ERROR: Target is more undefined than Source
...counterexample...
```

`alive-tv` is the standard tool for LLVM developers who want to verify a new InstCombine rule before submitting a PR. The workflow:
1. Write the transformation as two `.ll` files (before and after applying the pattern).
2. Run `alive-tv` to check correctness.
3. If correct, submit the PR with the note "verified with alive-tv."

### alive-tv in LLVM CI

LLVM's CI does not run `alive-tv` on every PR (too slow; requires building Alive2 separately). But the LLVM project maintains a set of InstCombine test cases that were historically found to be bugs, each accompanied by an `alive-tv` check to prevent regression.

Some InstCombine patches now include alive-tv output in the commit message as a form of documentation:

```
; From alive-tv:
; Name: fold icmp eq (add X, C1), C2
; Pre: C2 - C1 doesn't overflow
; %r = icmp eq (add %x, C1), C2
; =>
; %r = icmp eq %x, C2 - C1
; Transformation verified with alive-tv ✓
```

### llvm-mc-alive: Machine Code Validation

An experimental extension, `llvm-mc-alive`, applies Alive2's ideas to machine code transformations. Instead of LLVM IR, it verifies transformations on MachineIR (the pre-RA and post-RA machine instruction representation). The Z3 encoding uses the ISA's precise instruction semantics (from LLVM's TableGen definitions) rather than the LLVM IR bitvector model.

As of 2024, `llvm-mc-alive` is experimental and covers only a subset of x86-64 instructions.

### Performance and Limitations

**Performance**: a single function pair check typically takes milliseconds for pure arithmetic (Z3's QF_BV solver is fast for small functions). Memory-heavy functions take longer (the memory model adds axioms). Large functions with many instructions may time out (Alive2 has a configurable timeout, default 10 seconds per function).

**Completeness limitation**: Alive2 is sound (if it says a transformation is valid, it is valid) but not complete (it may time out without giving an answer on hard queries). The memory model is also an approximation: some valid pointer aliasing patterns are not representable in Alive2's SMT encoding.

**Interprocedural limitation**: Alive2 verifies individual functions in isolation. Transformations that require cross-function reasoning (e.g., whole-program devirtualization or LTO-enabled link-time optimizations) are out of scope.

**No loop reasoning**: loop transformations (unrolling, vectorization, loop-invariant code motion) involve reasoning about loop iterations, which is beyond QF_BV. Alive2 can check individual loop iterations but not the loop's overall behavior.

### Integration with MLIR

The ideas underlying Alive2 are applicable to MLIR transformations. MLIR's dialect lowering pipeline applies transformations similar to InstCombine (canonicalization, pattern rewriting). An "Alive2 for MLIR" — verifying dialect lowering patterns against a formal semantics for each dialect — is an active research direction as of 2024, but no complete tool exists yet.

---

## Chapter Summary

- **Translation validation** checks correctness of a specific compilation rather than the compiler algorithm; it requires a formal IR semantics and an automated verifier.
- **Alive** (Python, PLDI 2015) verified 334 InstCombine peephole patterns via Z3 bitvector encoding; found 8 bugs.
- **Alive2** (C++, from 2019) verifies full LLVM IR transformations by intercepting `opt` passes; found hundreds of bugs in LLVM's InstCombine, GVN, LICM, and SROA.
- The **refinement relation** `refines(tgt, src)`: for all inputs, if the source is defined (non-UB, non-poison), the target must produce the same value; if the source has UB, the target can do anything.
- **Poison semantics**: the target may produce poison where the source produces poison; it may not produce concrete values where the source produces concrete values, nor introduce UB where the source had none.
- **Memory model**: block-based; pointers are `(block_id, offset)` encoded as SMT bitvectors; alias information is integrated as SMT constraints; UB conditions (invalid pointer, out-of-bounds) are explicit SMT predicates.
- **`alive-tv`**: command-line tool for before/after IR validation; standard workflow for verifying new InstCombine rules.
- **Z3** (QF_BV, QF_FP, QF_AX) is the SMT backend; typical queries take milliseconds; memory-heavy functions may time out (10s default).
- Key limitations: no interprocedural reasoning, no loop reasoning, memory model is an approximation.

### References

- Lopes, N.P., Lee, D., Hur, C.K., Liu, Z., Regehr, J. (2015). "Provably Correct Peephole Optimizations with Alive." *PLDI 2015*.
- Lopes, N.P., Menendez, D., Nagarakatte, S., Regehr, J. (2021). "Alive2: Bounded Translation Validation for LLVM." *PLDI 2021*.
- Pnueli, A., Siegel, M., Singerman, E. (1998). "Translation Validation." *TACAS 1998*.
- Yang, X., Chen, Y., Eide, E., Regehr, J. (2011). "Finding and Understanding Bugs in C Compilers." *PLDI 2011*.
- Alive2 repository: [github.com/AliveToolkit/alive2](https://github.com/AliveToolkit/alive2)
- de Moura, L. and Bjørner, N. (2008). "Z3: An Efficient SMT Solver." *TACAS 2008*.
