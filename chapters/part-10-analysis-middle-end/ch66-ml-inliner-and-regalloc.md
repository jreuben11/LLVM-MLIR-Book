# Chapter 66 — The ML Inliner and ML Register Allocation

*Part X — Analysis and the Middle-End*

Machine learning has entered the compiler's optimization decision loop. Where threshold-based heuristics use hand-tuned constants to guide inlining and register allocation, ML-trained policies learn from billions of actual compilation outcomes to make better decisions. This chapter covers LLVM's two production ML components: the reinforcement-learning-trained inline advisor (`MLInlineAdvisor`) and the ML-guided register-allocation eviction policy.

## 66.1 Background: Optimization as Decision Making

Both inlining and register allocation are decision problems:
- **Inlining**: for each call site, should we inline or not?
- **Register allocation**: when a physical register is needed and all are occupied, which live range should we spill (evict) to memory?

Traditional heuristics use features like "callee instruction count < threshold" for inlining or "spill cost / benefit ratio" for eviction. These formulas are hand-written and fixed. ML policies, by contrast, are parameterized functions trained on real compilation data to predict which decision will lead to the best program performance.

## 66.2 The ML Inline Advisor

### 66.2.1 Architecture

The ML inline advisor is declared in `llvm/Analysis/InlineAdvisor.h` and implemented in `llvm/lib/Analysis/MLInlineAdvisor.cpp`. It replaces the default `DefaultInlineAdvisor` with an ML model evaluation path:

```
Call site features
  (instruction count, call depth, argument count, hotness, etc.)
       │
       ▼
Feature extraction (C++ → feature tensor)
       │
       ▼
TFLite runtime (evaluates saved ML model)
       │
       ▼
Inlining decision: inline / don't inline
```

### 66.2.2 Feature Extraction

For each candidate call site, the advisor extracts a fixed-size feature vector. The features include:
- Callee instruction count
- Caller instruction count
- Call site's block frequency (hotness)
- Number of callees with the same callee (call frequency)
- Caller's argument count
- Whether the callee is always-inline or never-inline
- The inline cost estimate from the traditional cost model
- The current inlining depth
- Whether this is a direct or indirect call

These features are normalized and packed into a tensor that the TFLite runtime evaluates.

### 66.2.3 TFLite Runtime Dependency

LLVM's ML components use **TFLite** (TensorFlow Lite) to run inference on saved models. TFLite is a lightweight runtime for executing TensorFlow Lite `.tflite` models without the full TensorFlow runtime.

The dependency is conditional: LLVM can be built with `LLVM_ENABLE_TENSORFLOW_AOT=ON` (compiling the model into a C++ AOT artifact, no runtime TFLite dependency) or `LLVM_ENABLE_TENSORFLOW_LITE_SUPPORT=ON` (using the TFLite shared library at runtime). The former is preferred for distribution:

```bash
cmake -DLLVM_ENABLE_TENSORFLOW_AOT=ON \
      -DTENSORFLOW_AOT_PATH=/path/to/tf-aot \
      ../llvm
```

When TFLite support is not compiled in, the ML inliner falls back to the default heuristic advisor.

### 66.2.4 Offline Training Pipeline

The model is trained offline using reinforcement learning (RL):

```
Compiler (LLVM with instrumented heuristic) 
    │ generates traces (features, decisions, outcomes)
    ▼
Training corpus (large codebase: Chromium, Linux kernel, etc.)
    │ (feature, decision, reward) triples
    ▼
RL training (Python, TensorFlow)
    │ policy gradient / PPO
    ▼
Trained policy (TFLite model file: inliner_model.tflite)
    │ exported
    ▼
LLVM build with --ml-advisor=release
```

The **reward signal** is the actual performance improvement (or degradation) caused by each inlining decision, measured by running benchmarks on the compiled binary. RL refines the policy to maximize total reward (program speedup) across the training corpus.

The training implementation is in the LLVM project under `llvm/lib/Analysis/models/` and the MLGO project at `https://github.com/google/ml-compiler-opt`.

### 66.2.5 Using the ML Advisor

```bash
# Build clang with ML inliner (requires TFLite AOT or TFLite library)
clang -O3 -mllvm -ml-inliner=release input.cpp -o output

# Or via opt (with model path for non-AOT builds)
opt -passes='cgscc(inline)' -ml-inliner=release -ml-inliner-model-path=./model.tflite \
    -S input.ll
```

The `release` mode uses the trained model; `default` uses the traditional heuristic; `development` is an instrumented mode for collecting training data.

```cpp
// Selecting the advisor programmatically
InlineAdvisorAnalysis::Result::initializeAdvisor(
    Module &M, FunctionAnalysisManager &FAM,
    InlineAdvisorAnalysis::AdvisorMode::Release);
```

## 66.3 ML Register Allocation Eviction Policy

### 66.3.1 The Eviction Problem

Linear scan register allocation (used in LLVM's greedy allocator) processes live intervals in order and assigns physical registers. When all physical registers are occupied and a new live interval needs one, the allocator must choose which existing interval to **evict** (spill to a stack slot).

The default eviction policy in LLVM's greedy allocator uses a cost function:
```
eviction_cost = spill_weight(candidate) / interference_count
```
where `spill_weight` is proportional to the interval's use frequency and its type's spill cost. The interval with the lowest cost (cheapest to spill) is evicted.

### 66.3.2 The ML Eviction Model

The ML eviction model (`MLEvictAdvisor` in `llvm/lib/CodeGen/RegAllocEvictionAdvisor.cpp`) replaces this formula with a learned policy. At each eviction decision point, it computes a feature vector and queries the model:

**Features per candidate eviction target:**
- Spill weight
- Cascade depth (has this interval been previously evicted?)
- Priority (assigned by the greedy allocator)
- Whether the interval is a hint from a coalesced copy
- The number of physical register candidates available for the evictee
- The number of interference counts

The model outputs a score per candidate; the one with the highest score is evicted.

### 66.3.3 Training

The eviction model is trained with a similar RL loop:
1. Compile many programs with an instrumented allocator that logs (features, eviction choice, downstream quality).
2. Use the downstream quality (total spill cost, code size) as the reward.
3. Train a small neural network (feature extractor + MLP head) to predict good evictions.

The model is intentionally small (a few hundred parameters) to minimize inference overhead during compilation. The TFLite AOT compilation converts it to a direct C++ function call, adding essentially zero overhead compared to the heuristic formula.

### 66.3.4 Using the ML Eviction Advisor

```bash
# Requires LLVM_ENABLE_TENSORFLOW_AOT=ON at build time
clang -O2 -mllvm -regalloc-enable-advisor=release input.cpp

# Or via llc
llc -regalloc-enable-advisor=release -O2 input.ll
```

```cpp
// Programmatically select the advisor
llvm::RegAllocEvictionAdvisorAnalysis::setAdvisorMode(
    llvm::RegAllocEvictionAdvisorAnalysis::AdvisorMode::Release);
```

## 66.4 Model Compilation: AOT vs Runtime

### 66.4.1 AOT (Ahead-of-Time) Compilation

In AOT mode, the TFLite model is compiled to C++ source during the LLVM build:

```bash
# Part of LLVM's cmake build when TENSORFLOW_AOT=ON
tensorflow/compiler/aot/tfcompile \
    --graph=model.pb \
    --config=model_config.pbtxt \
    --cpp_class=InlinerModel \
    --out_session_module=inliner_model.h
```

The result is a C++ class that can be called like any other C++ function, with zero dynamic dispatch and no shared library dependency. This is the preferred distribution mode.

### 66.4.2 Runtime TFLite

In runtime mode, LLVM loads the `.tflite` model file at startup and evaluates it through the TFLite C API. This requires `libtensorflowlite.so` to be present. Runtime mode is useful for:
- Experimenting with different models without recompiling LLVM.
- Collecting new training data with a custom reward function.
- A/B testing between model versions.

## 66.5 Interplay with Traditional Heuristics

The ML policies are not independent replacements for heuristics — they are composable:

1. **ML as a filter**: the ML advisor is only consulted for call sites where the traditional advisor is uncertain (cost near the threshold). This reduces ML overhead.
2. **Feature engineering**: many ML features are derived from the traditional heuristic's intermediate computations (inline cost, spill weight), so the ML model effectively learns a correction factor on top of the heuristic.
3. **Conservative fallback**: if the ML runtime is unavailable or the model predicts a very low confidence, the system falls back to the traditional heuristic.

## 66.6 Performance Results

Published results (from the MLGO paper, Trofin et al. ASPLOS 2021):

- **ML inliner**: 1–3% code size reduction over the default heuristic on large C++ codebases (Chrome, Tensorflow), with equivalent or better performance.
- **ML eviction**: 1–2% code size reduction and 0.5–1% performance improvement on CPU-bound benchmarks.

The improvements are modest in absolute terms but significant at Google-scale compilation (millions of binary files per day). The ML approach also avoids the manual tuning required to update heuristics when the codebase or hardware changes.

---

## Chapter Summary

- LLVM's ML inliner replaces the threshold-based `DefaultInlineAdvisor` with a TFLite-evaluated model that learned from reinforcement learning on large codebases.
- The inliner model takes a fixed feature vector per call site (instruction counts, hotness, cost estimates) and outputs an inline/don't-inline decision.
- Training uses an offline RL loop: LLVM compiles programs with an instrumented compiler, collects (features, decision, reward) triples, and trains a policy with PPO or similar.
- TFLite AOT compilation converts the model to a C++ class for zero-overhead inference; runtime TFLite allows model swapping without rebuilding LLVM.
- The ML eviction advisor for register allocation replaces the spill-weight formula with a small neural network that predicts which live interval to evict, trained with spill cost as the reward.
- Both ML components are built with `LLVM_ENABLE_TENSORFLOW_AOT=ON`; they fall back to traditional heuristics when the runtime is unavailable.
- Published results show 1–3% code size reduction and 0.5–1% performance improvement over traditional heuristics on large C++ codebases.
