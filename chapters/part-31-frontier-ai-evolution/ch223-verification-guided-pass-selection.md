# Chapter 223 — Verification-Guided Pass Selection: LLM-VeriOpt and the Alive2 Reward Loop

*Part XXXI — Frontier AI Evolution*

Every approach to ML-guided compilation faces the same fundamental problem: how do you know whether a compiler transformation is correct? Test-suite reward is fragile — a miscompilation that changes rarely-tested code paths will score full reward while silently breaking production workloads. Performance reward is circular — it measures optimization quality, not correctness, allowing a model to learn to break programs that happen to benchmark faster. Verification-guided pass selection replaces these approximate signals with a formal one: Alive2's semantic preservation check. If the transformed program is a refinement of the original — meaning every behavior the transformed program can exhibit was already possible in the original — then the transformation is provably correct, and the model receives full reward. This chapter covers the LLM-VeriOpt system (CGO 2026), the theoretical foundations of using Alive2 as a reward signal, the AlphaVerus extension to verified code generation, and the connection to the verified self-modification loop of Chapter 220.

---

## 223.1 The Verification-in-the-Loop Architecture

The standard RL-for-compiler-optimization loop measures performance:

```
LLM proposes pass sequence
    → opt applies it
    → binary is compiled and run
    → runtime measured
    → reward = speedup vs baseline
```

The problem: the loop has no correctness check. A pass sequence that miscompiles a benchmark and produces a faster-but-wrong binary receives high reward. The model learns to miscompile.

Verification-in-the-loop inserts Alive2 between compilation and reward:

```
LLM proposes pass sequence
    → opt applies it to IR
    → Alive2 checks: ∀ behavior b of output IR, b was possible in input IR
    → if incorrect: reward = 0.0, episode terminates immediately
    → if correct: reward = speedup_signal (performance or IR reduction)
```

This structure gives Alive2 the role of a **hard constraint gate**: no reward reaches the model through an incorrect transformation. The model cannot optimize around correctness — a miscompilation that scores better on performance is filtered at the gate before the reward signal is computed.

### Why Alive2 is Uniquely Suited as a Gate

Three properties make Alive2 the right choice (cross-reference [Chapter 170](../part-24-verified-compilation/ch170-alive2.md)):

1. **Zero false positives by construction**: Alive2's refinement check `src ⊑ tgt` uses Z3 to prove that every observable behavior of `tgt` was already possible in `src`. If Z3 returns "correct", the transformation is semantically sound — no human-labeled ground truth required.

2. **Miscompilation discovery on real LLVM bugs**: Alive2 has found >3,000 real LLVM bugs by running against every commit. Its false-negative rate on typical optimization passes is near zero. Using it as a reward gate means the model can never learn the miscompilations that Alive2 was designed to catch.

3. **Already in production LLVM CI**: Alive2 runs on every LLVM commit in CI. The integration infrastructure exists; the LLM-VeriOpt system reuses it rather than building a new verification pipeline.

### Comparison with RLHF

| Property | RLHF (human evaluator) | Alive2 reward |
|----------|----------------------|---------------|
| Correctness signal | Subjective, costly | Formal, free |
| Labelling cost | Per-example human time | Z3 solve time (~50–500 ms) |
| False positive rate | Non-zero (human error) | Zero (by construction) |
| Coverage | Limited to human-reviewed cases | All IR refinement properties |
| Scalability | Bottlenecked by humans | Bottlenecked by Z3 |

---

## 223.2 Alive2 as a Reward Signal

### The Refinement Relation as Binary Gate

Given an input IR function `src` and an output function `tgt`, Alive2 computes:

```
correct(src, tgt) = ∀ s ∈ behaviors(tgt). s ∈ behaviors(src)
```

where `behaviors` includes all possible execution traces including undefined behavior. The binary reward:

```python
reward_correctness = 1.0 if alive2_check(src, tgt) else 0.0
```

This is the hard gate. No pass sequence that produces an incorrect `tgt` receives any reward.

### Reward Shaping Extensions

The binary gate alone gives a sparse reward signal — the model receives 0 for any incorrect sequence and some value for any correct sequence. Shaping improves training efficiency:

**IR reduction as dense proxy**:
```python
def shaped_reward(src_ir, tgt_ir, alive2_result):
    if not alive2_result:
        return 0.0
    # Reward for reducing instruction count (proxy for performance)
    reduction = (count_instructions(src_ir) - count_instructions(tgt_ir))
                 / count_instructions(src_ir)
    return 0.3 + 0.7 * max(0.0, reduction)  # [0.3, 1.0] range for correct transforms
```

**Partial credit for SMT timeout**: when Z3 times out (common for complex functions), the transformation cannot be verified but also cannot be disproved. Assigning a partial reward (e.g., 0.3) for timeout encourages the model to prefer transformations that are at least not provably wrong:

```python
def timeout_reward(src_ir, tgt_ir, timeout_seconds=2.0):
    result = alive2_check(src_ir, tgt_ir, timeout=timeout_seconds)
    if result == "correct":     return 1.0
    if result == "timeout":     return 0.3
    if result == "incorrect":   return 0.0
```

### Latency Constraint: Batching Z3 Calls

Alive2's Z3 check takes 50–500 ms per function. At a training throughput of 1,000 episodes/hour, this is 50–500 seconds/hour in verification alone — a bottleneck. Mitigations:

- **Function-level batching**: verify all functions in a module in parallel (Alive2 is function-scoped)
- **Caching**: cache `(src_hash, tgt_hash) → result`; many RL training steps apply the same pass to similar IR
- **Early termination**: verify the first function in the module; if it fails, skip the rest (lower throughput but reduces Z3 load)
- **Selective verification**: only verify functions marked `hot` (by PGO data), skip cold functions

---

## 223.3 The LLM-VeriOpt System (CGO 2026)

LLM-VeriOpt trains a small language model to select LLVM pass sequences using Alive2 as the correctness oracle and IR improvement as the performance signal.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    LLM-VeriOpt Training Loop                │
│                                                             │
│  LLVM IR (from training corpus)                             │
│      │                                                      │
│      ▼                                                      │
│  Qwen-3B policy model                                       │
│  Input: tokenised IR (truncated to 2048 tokens)             │
│  Output: pass name token (from vocabulary of ~100 passes)   │
│      │                                                      │
│      ▼                                                      │
│  opt applies selected pass to current IR                    │
│      │                                                      │
│  ┌───▼───────────────────┐                                  │
│  │  Alive2 check:        │  → 0.0 (incorrect)              │
│  │  alive2(src, tgt)     │  → 0.3 (timeout)                │
│  │                       │  → 1.0 + IR reduction (correct)  │
│  └───────────────────────┘                                  │
│      │                                                      │
│  REINFORCE gradient update on Qwen-3B policy                │
└─────────────────────────────────────────────────────────────┘
```

### Policy Model: Qwen-3B

Qwen-3B (3 billion parameters) is chosen for the policy because:
- Small enough for fast inference during training (< 50 ms per pass selection)
- Large enough to learn IR structural patterns
- Pre-trained on code, so it starts with some understanding of LLVM IR syntax

The model is fine-tuned (not trained from scratch) using REINFORCE on the compiler optimization task.

### Pass Vocabulary

The action space is the set of ~100 named LLVM passes exposed by `opt`:

```
vocab = ["instcombine", "mem2reg", "gvn", "licm", "inliner",
         "simplifycfg", "loop-unroll", "loop-vectorize",
         "sroa", "dce", "adce", "early-cse", "sccp",
         "loop-idiom", "reassociate", "constprop",
         # ... ~85 more
         ]
```

Each pass name is a single token in the vocabulary. The model selects passes autoregressively, one per step, for up to 10 steps per episode.

### State Representation

The state at step `t` is the *current* IR (after applying all passes selected at steps 1 through `t-1`). The IR is tokenised at the byte level (LLVM IR is ASCII text) and truncated at 2,048 tokens — enough for most small-to-medium functions. Longer functions are handled by sliding-window summarization (embedding the first and last 1024 tokens).

### Training Results (CGO 2026)

On a held-out set of 10,000 LLVM IR functions from SPEC CPU 2006 and open-source projects:

| Metric | Value |
|--------|-------|
| Correctness rate | 90.4% (Alive2 verified) |
| Mean instruction count reduction vs baseline | 8.3% |
| Speedup on SPEC 2006 subsets | 6.1% vs `-O2` |
| Alive2 timeout rate | 7.2% |
| False positive rate | 0.0% (by construction) |

The 90.4% correctness rate means 9.6% of generated sequences are either incorrect (Alive2 counterexample) or unverifiable (Z3 timeout). For deployment, only verified sequences (Alive2 returns "correct") are applied.

---

## 223.4 Pass Ordering as an RL MDP

### Formal MDP Definition

The pass ordering problem as a Markov Decision Process:

- **State** `s_t`: the current LLVM IR function at step `t`, represented as a text token sequence
- **Action** `a_t`: a pass name selected from the vocabulary
- **Transition** `T(s_t, a_t) = s_{t+1}`: apply pass `a_t` to `s_t` using `opt`
- **Reward** `R(s_0, s_T)`: Alive2(s_0, s_T) × IR_reduction_bonus; received only at episode end
- **Terminal state**: after `T` passes or if Alive2 returns "incorrect"
- **Discount factor** `γ = 1.0` (undiscounted, short episode)

### The Sparse Reward Problem

The reward is received only at episode end (after the full pass sequence has been applied). This is standard sparse reward — the model must learn to associate early actions with delayed consequences. Approaches:

**Baseline in REINFORCE**: subtract a running mean reward from each episode's return, reducing variance:
```python
baseline = running_mean(past_returns, window=100)
advantage = episode_return - baseline
gradient ∝ advantage × log_prob(selected_passes)
```

**Hindsight Experience Replay (HER)**: for failed episodes, retroactively assign reward by finding the prefix that would have been correct (apply Alive2 to each prefix). This mines correct sub-sequences from failed episodes.

**Intermediate reward approximation**: use a fast differentiable surrogate model (trained offline on (IR, pass_sequence) → speedup pairs) to give dense intermediate rewards during training.

### PassManager as the Transition Function

```cpp
// Integration point: use opt's PassManager as the MDP transition
llvm::PassBuilder PB;
llvm::ModulePassManager MPM;
llvm::ModuleAnalysisManager MAM;
PB.registerModuleAnalyses(MAM);

// Apply selected pass (the RL action)
MPM.addPass(PB.parsePassPipeline(selected_pass_name));
MPM.run(*Module, MAM);  // s_t → s_{t+1}

// Now Module contains s_{t+1}
```

The `PassBuilder::parsePassPipeline` method accepts any `opt`-compatible pass name, making the full pass vocabulary directly available as RL actions.

---

## 223.5 AlphaVerus: Tree Search for Verified Code Generation

AlphaVerus extends the verification-reward concept from pass selection to code generation: instead of selecting passes over existing IR, it *generates* Verus-annotated Rust code that is verified by the Verus type checker.

### Treefinement Algorithm

```
1. Root: the initial specification (a formal Verus `requires`/`ensures` contract)
2. At each node: LLM generates candidate code implementations
3. For each candidate: Verus type checker runs (the oracle)
   - Verified: leaf node, return the implementation
   - Failed: expand node with a refinement prompt ("the proof failed at line X, fix it")
4. MCTS guides exploration: prefer nodes with higher past verification rate
5. Backpropagation: update node values based on verification success
```

The Verus type checker (cross-reference [Chapter 181](../part-26-ecosystem-frontiers/ch181-formal-verification.md)) serves the same role as Alive2: a zero-false-positive oracle that provides binary correctness feedback.

### Application to LLVM IR Generation

AlphaVerus's tree search transfers directly to LLVM IR generation:

```
Root: an IR function specification (input/output types + semantic contract)
At each node: LLM generates candidate LLVM IR function bodies
Oracle: Alive2(spec_function, candidate_function)
  - Alive2 "correct": leaf node, return the IR
  - Alive2 counterexample: refinement prompt includes the counterexample inputs
MCTS: prefer candidate IR that passes Alive2 or narrows the counterexample
```

This is stronger than LLM-VeriOpt: instead of selecting passes over existing IR, it generates new IR from a specification. The oracle guides the search toward provably correct implementations.

---

## 223.6 Integration with LLVM's CI

Alive2 is already integrated into LLVM's CI pipeline:

```bash
# How Alive2 runs on LLVM commits:
# From llvm-project/utils/alive2-validate.sh
for ir_file in $(find test/ -name "*.ll"); do
    alive2 --smt-to=30 $ir_file
done
```

The `llvm-project` CI runs Alive2 against all `.ll` test files on every commit. Extending this to include LLM-VeriOpt validation:

```bash
# Extended CI step: validate LLM-generated pass sequences
for ir_file in $(sample_ir_corpus --n 100 --seed $BUILD_ID); do
    # Generate pass sequence with LLM-VeriOpt
    pass_seq=$(lvm-veriopt-select $ir_file)
    # Apply sequence
    opt $pass_seq $ir_file -o /tmp/optimized.ll
    # Verify with Alive2
    alive2 --smt-to=30 $ir_file /tmp/optimized.ll \
        || echo "FAIL: $ir_file with $pass_seq"
done
```

This CI extension serves two purposes: (1) catching regressions in the LLM-VeriOpt model as IR patterns change, and (2) continuously expanding the training corpus with new verified (IR, pass_sequence) pairs from real LLVM tests.

---

## 223.7 Extending to Verified Self-Modification

The LLM-VeriOpt architecture composes directly with the self-modification loop from [Chapter 220](ch220-runtime-self-modification.md):

```
Self-modification loop with Alive2 verification:

OBSERVE: profile data → hot function identified
     ↓
REASON: LLM-VeriOpt selects pass sequence for the hot function's IR
     ↓
COMPILE: opt applies the pass sequence to produce new IR module
     ↓
VERIFY: Alive2 checks new_IR ⊑ original_IR (formal gate)
        • Counterexample found → reject, penalize model, return to REASON
        • Timeout → partial acceptance, fallback to test-harness verification
        • Correct → proceed to SWAP
     ↓
SWAP: RedirectableSymbolManager::redirect() atomically patches callers
```

The key improvement over Chapter 220's VERIFY phase: replacing the test harness (empirical, finite sample coverage) with Alive2 (formal, covers all inputs for bounded bit-width). The self-modification loop now has a formal correctness guarantee for the code it installs.

### ResourceTracker Integration

```cpp
// VERIFY phase: Alive2 check before committing the ResourceTracker
auto RT = JD.createResourceTracker();
auto TSM = llvm::orc::ThreadSafeModule(std::move(OptimizedModule), TSCtx);
llvm::cantFail(LLJIT->addIRModule(RT, std::move(TSM)));

// Run Alive2 check on (OriginalModule, OptimizedModule)
bool alive2_ok = runAlive2Check(OriginalModule, *OptimizedModule);
if (!alive2_ok) {
    llvm::cantFail(RT->remove());  // rollback
    return llvm::make_error<llvm::StringError>(
        "Alive2 counterexample: transformation is incorrect",
        llvm::inconvertibleErrorCode());
}

// Formal check passed — proceed to SWAP
llvm::cantFail(RSM->redirect(JD, newSymbols));
```

---

## 223.8 Practical Constraints and Open Problems

### Alive2 Coverage Gaps

Alive2 is function-scoped. It checks the refinement of individual functions, not whole-program semantic preservation. Inter-procedural transformations (inlining, interprocedural alias analysis, whole-program devirtualization) are not fully covered. The LLM-VeriOpt system therefore targets intra-procedural passes; passes that cross function boundaries are excluded from the pass vocabulary.

Ongoing Alive2 work (tracked in `llvm/utils/alive2/`) extends coverage to inter-procedural cases via [bi-abduction](https://research.fb.com/publications/bi-abduction-and-folklore-discovered/) style inference, but this is not yet production-ready.

### Undefined Behaviour in Input IR

Alive2 operates on the assumption that the input IR is free of undefined behavior. If `src` already contains UB (e.g., a `load` of a null pointer that happens never to execute on the training inputs), then Alive2 may vacuously accept an incorrect transformation because any behavior is refinement of UB-containing code. Mitigation: run `clang -fsanitize=undefined` to detect UB in the training corpus before using it.

### LLM Pass Vocabulary Limitations

The pass vocabulary contains only in-tree LLVM passes. Custom out-of-tree passes (domain-specific transformations for ML accelerators, safety-critical embedded targets) cannot be included without retraining the model. Transfer learning from the in-tree vocabulary to a domain-specific vocabulary is an open problem.

### Scalability: Whole-Program Alive2

For the distributed self-modification scenario (Chapter 220 §220.11), verifying an entire module (not just one function) requires Alive2 to check all functions in parallel. At 50–500 ms per function and 1,000 functions in a large module, this takes 50–500 seconds — incompatible with real-time hot-swap. The open research question: can a learned surrogate model predict Alive2's verdict in < 1 ms, with near-zero false positive rate?

---

## Chapter Summary

- Verification-in-the-loop replaces fragile performance reward with Alive2's formal refinement check as a hard correctness gate; no incorrect transformation can receive reward
- Alive2's `src ⊑ tgt` refinement relation provides a zero-false-positive binary reward signal; SMT timeout is handled via partial reward shaping
- LLM-VeriOpt (CGO 2026) trains Qwen-3B with REINFORCE on a vocabulary of ~100 LLVM passes; achieves 90% correctness (Alive2 verified) and 8% IR reduction over baseline on SPEC 2006
- The pass ordering MDP: state = current IR; action = pass name; reward = Alive2(original, final) × IR reduction; `PassBuilder::parsePassPipeline` is the transition function
- AlphaVerus generalizes the approach to code generation via MCTS with the Verus or Alive2 type-checker as the oracle; Treefinement guides search toward verified implementations
- LLM-VeriOpt integrates naturally with LLVM CI (Alive2 already runs on every commit) and with the ORC self-modification loop (Chapter 220): Alive2 replaces the test harness in the VERIFY phase, providing formal correctness guarantees before `redirect()` is called
- Practical constraints: Alive2 is function-scoped (no inter-procedural coverage); UB in input IR can cause vacuous acceptance; pass vocabulary excludes out-of-tree passes; whole-program verification is too slow for real-time hot-swap

---

@copyright jreuben11
