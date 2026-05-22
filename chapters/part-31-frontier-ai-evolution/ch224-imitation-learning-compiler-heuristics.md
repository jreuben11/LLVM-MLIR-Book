# Chapter 224 — Imitation Learning for Compiler Heuristics: BC-Max and Behavioral Cloning

*Part XXXI — Frontier AI Evolution*

Compiler heuristics are everywhere: the inliner decides whether each call site should be inlined, the vectorizer decides which loops to vectorize and at what width, the register allocator decides which live range to spill. Each decision is made by a hand-crafted heuristic — a function of a few features that approximates the true cost function. These heuristics were written by compiler engineers based on intuition, microbenchmarks, and decades of accumulated knowledge. Machine learning offers an alternative: learn the heuristic from data rather than writing it by hand. The question is which learning paradigm to use. Reinforcement learning (RL) is the obvious choice — the compiler can be simulated, actions have consequences, and reward can be measured. But RL for compiler tasks is surprisingly hard. This chapter explains why RL fails and when imitation learning succeeds, covers Google's BC-Max framework for inlining, and shows how the same framework extends to register allocation, vectorization, and loop unrolling.

---

## 224.1 Why Reinforcement Learning is Hard for Compiler Tasks

### The Sparse Reward Problem

RL requires a reward signal. For compiler optimization, the natural reward is the performance of the compiled binary — measured by running it. This creates five problems:

1. **Latency**: each RL step requires compiling and running a benchmark. For a 100-function module with 10 inlining decisions each, evaluating one complete inlining policy takes ~1 second. Training 100,000 episodes takes ~28 hours.

2. **Sparsity**: the reward arrives only when the full binary runs. The 1,000 inlining decisions that produced the binary each receive the same global reward — no per-decision credit assignment.

3. **Non-stationarity**: each inlining decision changes the IR, which changes the state for all subsequent decisions. The optimal action at step `t` depends on what decisions were made at steps 1 through `t-1`. This interdependence makes the MDP non-stationary from the perspective of any single decision point.

4. **High variance**: small changes in inlining decisions can have outsized effects on cache behavior, which depends on code layout, which depends on many decisions in sequence. The reward function is high-variance relative to individual decisions.

5. **Reward hacking**: without a correctness gate (cf. Chapter 223's Alive2 approach), the RL model may find inlining policies that trick the benchmark into registering a speedup without improving actual workload performance.

### What Imitation Learning Avoids

Imitation learning (IL) sidesteps all five problems by learning from *demonstrations* rather than from reward:

- No runtime required for training (no latency bottleneck)
- No credit assignment problem (each demonstration labels each decision)
- No non-stationarity during training (training data is pre-collected)
- Low variance (supervised signal per decision)
- No reward hacking (the demonstrations come from correct, well-tested heuristics)

The cost: IL cannot exceed the performance of its demonstrations. An IL model can only be as good as the best demonstrator it learned from. RL can, in principle, exceed any demonstrator — but only if training is feasible.

---

## 224.2 Inlining as a Prototypical Heuristic Decision Problem

Inlining is the highest-leverage optimization in most compilers: inlining a callee into a caller enables constant folding, dead code elimination, alias analysis, and vectorization that would otherwise be blocked by the call boundary. But inlining also increases code size (potentially causing instruction cache misses), increases register pressure, and can trigger deoptimization in JIT settings.

The inlining decision at each call site is binary: inline or not. The MLGO (ML-Guided Optimization) framework ([Chapter 180](../part-26-ecosystem-frontiers/ch180-ai-guided-compilation.md)) extracts a 24-dimensional feature vector per call site:

| Feature | Description |
|---------|-------------|
| `callee_basic_block_count` | Size proxy for the callee |
| `callee_users` | Number of call sites (hot callees are worth inlining) |
| `cost_estimate` | LLVM's built-in inlining cost model estimate |
| `edge_count_at_callsite` | PGO-derived call frequency |
| `number_of_instructions` | Total instructions in callee |
| `discount_factor` | Depth of call stack |
| `is_callee_avail_externally` | Linkage indicator |
| `callsite_hotness` | Hot/cold/none from PGO |
| `argument_with_direct_uses` | Constant folding potential |
| `callee_conditionals` | Branch coverage indicator |
| ... | (14 more features) |

Source: `llvm/lib/Analysis/MLInlineAdvisor.cpp`, function `MLInlineAdvisor::getAdvice`.

The ground-truth oracle: for training, one could exhaustively try all 2^N combinations for a function with N call sites and find the optimal subset. In practice, a near-optimal oracle is constructed by running several heuristics and taking the best-performing one per call site.

---

## 224.3 The BC-Max Framework (Google, NeurIPS 2024 ML4Sys)

### Behavioral Cloning and Its Limitation

Standard Behavioral Cloning (BC) learns to imitate a single expert:
- Expert policy `π_E` makes decisions `{(s_1, a_1), (s_2, a_2), ...}`
- BC trains a neural network `π_θ` to minimize `E[cross_entropy(π_θ(s), a)]`
- At deployment: `π_θ` mimics `π_E`

The limitation: BC is bounded by `π_E`'s performance. If `π_E` is suboptimal, so is the learned policy.

### BC-Max: Learning from Multiple Baselines

BC-Max extends BC to learn from multiple expert baselines simultaneously:

```python
# Training set construction:
# For each call site s:
#   Run all K baseline policies on the function
#   Label s with the action chosen by the BEST-PERFORMING baseline

def bc_max_label(call_site, baselines, metric="binary_size"):
    baseline_outcomes = {}
    for b in baselines:
        action = b.decide(call_site)
        outcome = compile_and_measure(apply_inline(action, function), metric)
        baseline_outcomes[b] = (action, outcome)

    # Max: pick the action from the best-performing baseline
    best_baseline = max(baseline_outcomes, key=lambda b: baseline_outcomes[b][1])
    return baseline_outcomes[best_baseline][0]
```

The baselines used in Google's deployment:

| Baseline | Strategy |
|----------|----------|
| Default LLVM inliner | LLVM's built-in heuristic (threshold-based cost model) |
| Size-optimizing inliner | Aggressive size reduction (`-Oz` policy) |
| Speed-optimizing inliner | Aggressive inlining for hot paths (PGO-driven) |
| Conservative inliner | Never inline above 10 instructions |

The Max operator selects the label from whichever baseline performed best *for that specific call site and function*. This gives the model the benefit of each baseline's strengths without the weaknesses of any single one.

### Theoretical Justification: Regret Decomposition

Let `π*` be the true optimal policy. The regret of BC-Max decomposes as:

```
Regret(π_BCMax) ≤ Regret(π_bestBaseline) + ε_approximation
```

where `ε_approximation` is the supervised learning error (how well the neural net fits the BC-Max labels). If the best baseline has low regret and the neural net is expressive enough, BC-Max's regret is bounded below the best baseline's regret. This is the key theoretical advantage over single-baseline BC: the bound involves the *best* baseline, not the *average* baseline.

### Empirical Results (Google Production, 2024)

Deployed on Chromium and Google's internal build systems:

| Metric | Result |
|--------|--------|
| Binary size reduction vs `-O2` | 2.1–3.4% |
| Compilation overhead | < 1 ms per function (TFLite inference) |
| Correctness rate | 100% (same as LLVM's existing correctness guarantees) |
| Training data size | 10M call site examples from production builds |
| Model size | 3-layer MLP, 50K parameters |

Source: "Scalable Self-Improvement for Compiler Optimization" (Google Research Blog, NeurIPS 2024 ML4Sys workshop).

---

## 224.4 Feature Engineering and MLGO Architecture

### Feature Extraction

MLGO's feature extractor runs as an LLVM analysis pass:

```cpp
// llvm/lib/Analysis/MLInlineAdvisor.cpp
class MLInlineAdvisor : public InlineAdvisor {
    std::unique_ptr<InlineModelRunner> Runner;
    // ...
    InlineAdvice getAdvice(CallBase& CB) override {
        auto Features = extractFeatures(CB);
        auto Decision = Runner->run(Features);
        return InlineAdvice(Decision > 0.5);
    }
};
```

The `InlineModelRunner` wraps TFLite inference, keeping compilation latency to < 1 ms per call site.

### Model Architecture

The neural network is intentionally small:

```
Input: 24 float32 features
→ Dense(64, relu)
→ Dense(64, relu)
→ Dense(64, relu)
→ Dense(1, sigmoid)  # probability of "inline"
Output: p(inline)
```

Larger models did not improve results meaningfully — the inlining decision is locally determined by the 24 features; deep networks overfit the training distribution.

### TFLite Deployment in LLVM

The trained model is compiled to a TFLite flatbuffer and embedded in LLVM as a binary blob:

```bash
# Build-time: convert model to TFLite
tflite_convert --output_file=inliner_model.tflite \
    --saved_model_dir=trained_model/

# Embed in LLVM source tree
xxd -i inliner_model.tflite > InlinerModel.inc
```

At runtime, `MLModelRunner` loads the embedded TFLite model and runs inference:

```cpp
// llvm/lib/Analysis/MLModelRunner.cpp
class TFLiteModelRunner : public MLModelRunner {
    std::unique_ptr<tflite::Interpreter> Interp;
    // ...
    std::vector<float> run(const std::vector<float>& features) override {
        auto* input = Interp->typed_input_tensor<float>(0);
        std::copy(features.begin(), features.end(), input);
        Interp->Invoke();
        return {Interp->typed_output_tensor<float>(0)[0]};
    }
};
```

The TFLite dependency is optional (guarded by `LLVM_HAVE_TFLITE` CMake flag), so LLVM builds without TFLite remain unchanged.

---

## 224.5 Register Allocator Eviction Advisor

MLGO extends beyond inlining to register allocation. At each spill decision, the Greedy allocator must choose which live range to evict from a physical register. The MLGO eviction advisor replaces the default heuristic with a learned one.

### The Eviction Decision

The Greedy allocator's eviction heuristic (`lib/CodeGen/RegAllocGreedy.cpp`):
```cpp
// Default: evict the live range with the lowest spill weight
float evictionCost(const LiveInterval& LI) {
    return LI.weight() / LI.getSize();  // spill weight / live range length
}
```

The learned advisor receives a feature vector per candidate eviction:

| Feature | Description |
|---------|-------------|
| `nr_broken_hints` | Number of register hints the eviction breaks |
| `nr_rematerializable` | Whether the live range can be rematerialized (cheaper than spill) |
| `nr_same_dequeue_heuristic` | Number of other LRs with same priority |
| `nr_spill_weight` | The default spill weight |
| `nr_cascade` | Cascade level of the eviction |
| `nr_local` | Whether the LR is local (single basic block) |
| ... | (additional features from interference graph) |

Source: `llvm/lib/CodeGen/MLRegAllocEvictAdvisor.cpp`.

### BC-Max for Eviction

The same BC-Max framework applies:
- Baseline 1: default Greedy eviction heuristic
- Baseline 2: evict always the most recently added live range
- Baseline 3: evict the live range with the most call sites (prefer keeping frequently-called registers)
- Label: whichever baseline produced fewer total spills on the training function

Empirical result: 1–2% fewer spills vs the default Greedy heuristic on Chromium builds.

---

## 224.6 Offline vs Online Imitation: Deployment Tradeoffs

### Offline Imitation

Train once on a historical corpus; deploy a frozen model. This is the current production deployment:

```
Training (offline):
  Historical build logs → feature-label pairs → train MLP → convert to TFLite

Serving (online):
  At each inlining decision → TFLite inference → binary decision
```

Advantages: predictable compilation behavior, no training infrastructure in the deployment environment.

Disadvantages: dataset shift — production code evolves, and the model may make suboptimal decisions on new code patterns not in the training corpus.

### Online Imitation: DAgger

The Dataset Aggregation (DAgger) algorithm (Ross et al. 2011) addresses dataset shift by interleaving collection and training:

```
Iteration 1: deploy policy π₁ (initial IL policy)
             collect decisions made by π₁ in production
             label each decision with the best-baseline label
             train π₂ on augmented dataset (original + new)

Iteration 2: deploy π₂
             collect, label, train π₃
...
```

DAgger provably reduces the compounding error of BC by training on the distribution of states the deployed policy actually visits, not just the distribution the expert visited. Google's production system uses quarterly retraining — a lightweight approximation of online DAgger.

---

## 224.7 Comparison with Reinforcement Learning

| Property | BC-Max (IL) | REINFORCE (RL) | A3C/PPO (RL) |
|----------|------------|----------------|--------------|
| Reward signal required? | No | Yes | Yes |
| Correctness gate required? | No (inherits from baselines) | Recommended | Recommended |
| Training stability | High (supervised) | Low (high variance) | Medium |
| Can exceed best baseline? | No | Yes | Yes |
| Training compute | Low (supervised) | High (episodes) | High (episodes) |
| Deployment latency | < 1 ms (TFLite) | Same | Same |
| State-of-the-art result | 2–3% binary size reduction | 5–8% on toy benchmarks | 3–5% |

The practical conclusion: start with BC-Max for fast, stable deployment. Switch to RL (with verification gate from Chapter 223) only if the performance gap justifies the training cost.

---

## 224.8 Generalizing to Other Heuristics

The BC-Max framework is a template. Any compiler heuristic that makes a discrete decision based on a feature vector from IR is a candidate.

### Loop Unrolling Factor

Decision: how many times to unroll a loop body (0, 2, 4, 8, 16).

Features:
- Trip count estimate (static or profile-guided)
- Loop body instruction count
- Memory access count (loads/stores)
- Register pressure estimate
- Whether loop is innermost

Baselines:
- No unrolling
- Full unrolling (trip count times)
- Default LLVM unrolling (factor based on cost model)
- Profile-guided unrolling (unroll hot loops more)

### Vectorization Decision

Decision: vectorize this loop at width 4, 8, or 16 (or not at all).

Features:
- Memory access pattern regularity (stride-1 vs stride-N vs gather)
- Data type width
- Loop body dependency distance
- Trip count modulo vector width
- Available target SIMD width (AVX-512, SVE, NEON)

This is more complex than inlining because the set of valid actions depends on target ISA features — the BC-Max label must be conditioned on the target.

### Instruction Scheduling

Decision: at each ready-list step, which instruction to schedule next.

Features:
- Critical path height of the instruction
- Number of uses (downstream instructions waiting)
- Resource type (ALU, load/store, branch)
- Predicted cache miss probability (from memory access pattern)

The scheduling decision is made at much finer granularity than inlining — hundreds of decisions per function — making it the hardest case for imitation learning (more opportunities for compounding error).

---

## Chapter Summary

- RL for compiler heuristics is hard: sparse reward, non-stationary state, high variance, and reward hacking risk; imitation learning avoids all four at the cost of being bounded by demonstrator performance
- Inlining is the canonical IL use case: binary per-call-site decision, 24-dimensional feature vector from `MLInlineAdvisor`, deployed via TFLite inference in `MLModelRunner`
- BC-Max labels each call site with the action of the best-performing baseline across multiple expert policies; the Max operator gives the bound `Regret(BC-Max) ≤ Regret(best_baseline) + ε_approximation`
- Google's deployment achieves 2–3% binary size reduction on Chromium with < 1 ms compilation overhead; model is a 3-layer MLP with 50K parameters compiled to TFLite
- The register allocator eviction advisor applies the same BC-Max framework to spill decisions in `MLRegAllocEvictAdvisor.cpp`, achieving 1–2% spill reduction
- DAgger (Dataset Aggregation) provides an online IL alternative that corrects for dataset shift by training on the states the deployed policy actually visits; Google uses quarterly retraining as a practical approximation
- The framework generalizes to loop unrolling factor selection, vectorization decisions, and instruction scheduling; all three are discrete decisions with feature vectors extractable from LLVM IR

---

@copyright jreuben11
