# Chapter 218 — Self-Improvement Fitness Functions and Capability Assessment

*Part XXXI — Frontier AI Evolution*

Every mechanism surveyed in Chapters 212–217 modifies, merges, edits, evolves, or introspects a neural system. All of them implicitly depend on a single unanswered question: how do you know the result is better? A gradient step reduces training loss, but training loss is a proxy. A weight-space merge improves benchmark throughput, but the benchmark is finite and saturates. A Gödel Machine executes a self-rewrite only when `F ⊢ V(t, apply(r)) > V(t, stop)` — but `V` must be formally specified, and specifying what "improvement" means over open-ended capabilities is not a solved problem. This chapter treats the fitness function as a first-class engineering artifact: something to be designed, implemented, audited, and replaced when it drifts from the capability it was intended to measure. Section 218.1 frames the oracle problem precisely. Sections 218.2–218.7 develop six concrete fitness signal architectures — linear probing, process reward models, self-generated test cases, dynamic benchmarks, interpretability-as-evaluation, and regression test suites. Section 218.8 characterises the fitness landscape those signals live on. Section 218.9 taxonomises signal types. Section 218.10 assembles the full self-improvement loop.

Cross-references: [Chapter 207 — Reflective Code, Open Problems, and Build Roadmap](../part-30-AI-first-PL-design/ch207-reflective-code-open-problems-roadmap.md) · [Chapter 212 — Weights as a Programming Substrate](ch212-weights-as-programming-substrate.md) · [Chapter 213 — Mechanistic Interpretability Infrastructure](ch213-mechanistic-interpretability.md) · [Chapter 215 — Evolutionary Architecture Search](ch215-evolutionary-architecture-search.md) · [Chapter 216 — Formal Self-Improvement Theory](ch216-formal-self-improvement-theory.md) · [Chapter 217 — Self-Reflective Inference and Architecture Introspection](ch217-self-reflective-inference.md)

---

## 218.1 The Oracle Problem

A fitness function is an oracle: given a candidate system, it returns a score that is supposed to rank candidates by quality. The oracle problem is that every feasible fitness function is, at best, a proxy for the capability you actually care about. That proxy relationship creates three compounding failure modes — underfitting, overfitting, and Goodhart drift — and no formal theory has closed all three simultaneously.

### Formal Specification of "Improvement"

The theoretically ideal fitness function for a self-modifying agent is the Gödel Machine's `V(t, r)`: the utility function encoded in the axioms of the formal system `F`, evaluated against the full distribution of future interaction histories. A candidate self-rewrite `r` is beneficial if and only if `F ⊢ V(t, apply(r)) > V(t, stop)`. This is an existence theorem for optimality: if the proof goes through, the rewrite is provably beneficial under `V` [Schmidhuber, "Gödel Machines", arXiv cs/0309048]. The two structural obstacles are:

1. `V` must be fully formalised in `F`. For any capability that involves open-ended interaction with the world — writing code, reasoning about mathematics, advising humans — no known formal specification captures what "better" means across all interaction trajectories.
2. Proof search over `V(t, apply(r)) > V(t, stop)` is undecidable. Even when `V` is fully specified, no computable algorithm can determine in finite time whether the theorem holds.

The consequence is that every deployable fitness function is a computable approximation to an uncomputable ideal. Understanding where that approximation breaks is the engineering task.

The hierarchy of approximations forms a spectrum: at one extreme, the Gödel Machine's formal utility function requires an axiomatic system that specifies every relevant aspect of agent-environment interaction and is sound (which it cannot prove of itself); at the other extreme, a static benchmark score requires only running the model on a fixed test set, but is trivially gameable. The six fitness signal types developed in §218.2–218.7 occupy the middle of this spectrum, each trading off expressiveness, reliability, and computational cost differently. No single point on the spectrum dominates; the engineered fitness function must be chosen and combined based on what Goodhart failure mode is most dangerous for the specific self-improvement application.

### The Self-Referential Evaluation Problem

A model evaluating its own outputs using its own capabilities introduces a coherent-error failure mode that does not arise in external evaluation. If the evaluator and the subject are the same model, the evaluator's blind spots — capabilities it lacks, biases it encodes — are exactly the blind spots it will fail to penalise in the subject. The problem is structural, not a matter of prompting quality. A model that cannot reliably solve a certain class of problems cannot reliably score solutions to that class. This is the neural analogue of a compiler using itself to validate its own optimisations: correct self-compilation is necessary but not sufficient for correctness.

The most robust mitigations are: (1) external evaluation pipelines that do not share weights with the subject system, (2) process reward models trained on held-out verifier data (§218.3), and (3) interpretability-based evaluation that reads internal representations rather than self-reported scores (§218.6).

### Goodhart's Law and Proxy Drift

Goodhart's Law states: when a measure becomes a target, it ceases to be a good measure. In the self-improvement context, this manifests as three phenomena:

**Benchmark saturation**: a model trained against a fixed evaluation set eventually achieves scores that reflect memorisation of evaluation patterns rather than the underlying capability. GSM8K scores above 90% for frontier models no longer discriminate between systems of different mathematical reasoning ability.

**Reward hacking**: a model optimised against a learned reward model finds policies that maximise the reward signal without satisfying the intended objective. This is distinct from benchmark saturation: the model exploits quirks in the reward model's training distribution rather than the evaluation set's test items.

**Specification gaming**: the fitness function captures a necessary but not sufficient condition for the desired capability. A coding agent that maximises unit test pass rate may find code that passes the provided tests while being brittle, unmaintainable, or incorrect on out-of-distribution inputs. Specification gaming differs from reward hacking in that the fitness function's specification is itself incomplete — the proxy correctly measures what it was defined to measure, but the definition was underspecified relative to the intended capability. The solution is not a better optimiser but a richer specification, which is the fundamental motivation for the interpretability-based fitness signals of §218.6.

The drift type signature from [Chapter 207](../part-30-AI-first-PL-design/ch207-reflective-code-open-problems-roadmap.md) captures this precisely: `drift : Impl[T] → Spec[T] → DriftReport` monitors the gap between a running implementation and its formal specification. In the fitness function context, `Impl[T]` is the fitness function as implemented, and `Spec[T]` is the intended capability it was designed to measure. Fitness functions drift from their specification just as code drifts from its spec — continuously, and without explicit notification.

---

## 218.2 Capability Probing as Self-Assessment

Linear probing is the simplest form of capability measurement that does not require the model to generate text. A linear probe is a single-layer classifier trained on frozen activations from a specific layer of the target model, predicting a binary or categorical label. If the probe achieves high accuracy from activations at layer `l`, the relevant information is encoded in a linearly accessible form at that layer.

### Probing on Frozen Activations

```python
import torch
import torch.nn as nn
from transformer_lens import HookedTransformer

def train_linear_probe(
    model: HookedTransformer,
    hook_name: str,
    prompts: list[str],
    labels: torch.Tensor,
    n_epochs: int = 50,
) -> tuple[nn.Linear, list[float]]:
    model.eval()
    with torch.no_grad():
        _, cache = model.run_with_cache(prompts)
        acts = cache[hook_name][:, -1, :]  # last-token activation, shape (N, d_model)

    probe = nn.Linear(acts.shape[-1], labels.max().item() + 1)
    opt = torch.optim.Adam(probe.parameters(), lr=1e-3)
    losses = []
    for _ in range(n_epochs):
        logits = probe(acts)
        loss = nn.functional.cross_entropy(logits, labels)
        opt.zero_grad()
        loss.backward()
        opt.step()
        losses.append(loss.item())
    return probe, losses

def probe_accuracy(probe: nn.Linear, acts: torch.Tensor, labels: torch.Tensor) -> float:
    with torch.no_grad():
        preds = probe(acts).argmax(-1)
    return (preds == labels).float().mean().item()
```

The interpretation is quantitative: probe accuracy of 95% indicates that the relevant feature is robustly linearly encoded at the probed layer; accuracy near chance (50% for binary) indicates the feature is not represented in a linearly accessible form at that layer, though it may be represented non-linearly or at a different depth.

### Depth-Accuracy Profiles as Capability Maps

Scanning probe accuracy across layers rather than at a single layer produces a depth-accuracy profile — a curve that reveals at which depth in the network a given capability is resolved. Capabilities that require integrating information across long contexts tend to resolve in later layers; syntactic capabilities (part-of-speech tags, bracket matching) often resolve in early layers. The profile shape is itself a capability indicator: a sudden rise in probe accuracy at layer `l` marks the computation boundary where the relevant transformation occurs.

The depth-accuracy profile changes after self-modification. A weight-space merge ([Chapter 212](ch212-weights-as-programming-substrate.md)) that combines two models with complementary capabilities should produce a merged model whose depth-accuracy profiles for both capabilities are at least as high as the better of the two constituent models at each layer. When a profile drops substantially in intermediate layers and recovers at the final layer, the capability is being computed by a longer, more indirect path — a sign that the direct circuit has been partially disrupted and a compensatory path has formed.

```python
def depth_accuracy_profile(
    model: HookedTransformer,
    prompts: list[str],
    labels: torch.Tensor,
    n_epochs: int = 30,
) -> dict[str, float]:
    profile = {}
    for layer in range(model.cfg.n_layers):
        hook_name = f"blocks.{layer}.hook_resid_post"
        probe, _ = train_linear_probe(model, hook_name, prompts, labels, n_epochs)
        model.eval()
        with torch.no_grad():
            _, cache = model.run_with_cache(prompts)
            acts = cache[hook_name][:, -1, :]
        profile[f"layer_{layer}"] = probe_accuracy(probe, acts, labels)
    return profile
```

### SAE Features as Capability Indicators

Sparse autoencoder features ([Chapter 213 — Mechanistic Interpretability Infrastructure](ch213-mechanistic-interpretability.md), §213.3) provide a richer vocabulary for capability assessment than neuron activations alone. Each SAE feature corresponds to a monosemantic direction in activation space, often interpretable as a human-readable concept ("legal reasoning", "arithmetic carry operation", "Python import statement"). A capability assessment protocol based on SAE features proceeds as follows:

1. For a target capability domain (e.g., formal proof steps, code debugging, algebraic manipulation), curate a prompt set that exercises the domain.
2. Run the model on these prompts and collect SAE feature activations at the layer of interest.
3. Identify the top-k features by mean activation magnitude across domain prompts versus a baseline prompt set.
4. Track these features as a capability fingerprint: high activation on the domain-specific feature set indicates the model is engaging the relevant circuits.

The quantitative version is a per-feature mean activation ratio between domain prompts and baseline. Ratios substantially above 1 indicate domain-relevant features; ratios near 1 indicate the feature is not domain-specific. This is a model-internal fitness signal requiring no external evaluation data.

---

## 218.3 Process Reward Models as Structured Fitness

A process reward model (PRM) scores each reasoning step in a chain-of-thought, not only the final answer. This converts a sparse terminal fitness signal — correct/incorrect after N steps — into a dense per-step signal that guides search at every intermediate state.

### PRM800K: Per-Step Correctness Data

PRM800K (Lightman et al., arXiv 2305.20050) is a dataset of 800K step-level correctness labels for MATH competition problems, produced by human annotators who labelled each step of model-generated solutions as correct, incorrect, or neutral. The core empirical finding is that a PRM trained on step-level labels substantially outperforms an outcome reward model (ORM) — which sees only the final answer's correctness — as a verifier for best-of-N sampling: selecting the solution with the highest PRM score achieves better accuracy than selecting by ORM score at the same compute budget. The mechanism is that an ORM cannot distinguish a correct answer reached by faulty reasoning from a correct answer reached by valid steps; the PRM can.

### Architecture and Training

A PRM is a language model with an additional scalar head applied at each step boundary (typically marked by a special token or a delimiter like `\n\n`):

```python
import torch
import torch.nn as nn
from transformers import AutoModel, AutoTokenizer

class ProcessRewardModel(nn.Module):
    def __init__(self, base_model_name: str):
        super().__init__()
        self.backbone = AutoModel.from_pretrained(base_model_name)
        d_hidden = self.backbone.config.hidden_size
        self.step_scorer = nn.Linear(d_hidden, 1)

    def forward(
        self,
        input_ids: torch.Tensor,
        step_mask: torch.Tensor,
    ) -> torch.Tensor:
        hidden = self.backbone(input_ids).last_hidden_state  # (B, T, D)
        step_hiddens = hidden[step_mask.bool()]              # (n_steps, D)
        return self.step_scorer(step_hiddens).squeeze(-1)    # (n_steps,)
```

Training uses binary cross-entropy on per-step labels. At inference, the aggregate solution score is the minimum step score (pessimistic aggregation) or the product of per-step probabilities (log-sum interpretation).

### PRM Scoring Loop

```python
def score_solution_with_prm(
    prm: ProcessRewardModel,
    tokenizer,
    problem: str,
    solution_steps: list[str],
    aggregation: str = "min",
) -> float:
    full_text = problem + " " + " ".join(solution_steps)
    enc = tokenizer(full_text, return_tensors="pt")
    step_mask = build_step_mask(enc.input_ids, tokenizer.step_token_id)
    with torch.no_grad():
        step_scores = prm(enc.input_ids, step_mask).sigmoid()
    if aggregation == "min":
        return step_scores.min().item()
    return step_scores.prod().item()

def build_step_mask(input_ids: torch.Tensor, step_token_id: int) -> torch.Tensor:
    return (input_ids == step_token_id).squeeze(0)
```

### PRMs as Evolutionary Fitness

In the evolutionary architecture search framework of [Chapter 215 — Evolutionary Architecture Search](ch215-evolutionary-architecture-search.md), a PRM functions as a differentiable fitness proxy. Rather than evaluating a candidate reasoning trace by running the final answer through an external oracle (slow, potentially unavailable), the PRM provides a score in a single forward pass. The step-level density also enables gradient-guided beam search: keeping the top-k partial solutions by PRM prefix score at each decoding step, rather than waiting for the full trace to complete.

The advantage over ORM is not just accuracy but search efficiency: an ORM applied to beam search can only prune after full solution generation; a PRM prunes the beam at every step boundary, reducing the total computation by the average fraction of beams that are pruned early.

### Monte Carlo Tree Search with PRM Guidance

The PRM's step-level density integrates naturally with Monte Carlo Tree Search (MCTS) over reasoning trees. Each node in the search tree is a partial solution prefix; the value function used for MCTS rollout is the PRM score at that node. The tree is expanded by sampling next steps from the model; nodes are pruned when their PRM prefix score falls below a threshold; terminal nodes (complete solutions) are scored by the PRM minimum-step aggregation plus, if available, an external verifier.

This combination — generator (the language model), step-level value function (PRM), and external verifier — was used in the AlphaProof system for the 2024 IMO (arXiv 2502.01839) and in the process of OpenAI's o1/o3 reasoning models. The PRM in this context is not just a fitness function; it is the search heuristic that makes MCTS tractable over the exponentially large space of reasoning trees. Without step-level guidance, beam search over full traces is the only option, and it scales poorly: the number of beams needed to find a correct solution grows exponentially with solution length, while PRM-guided MCTS maintains logarithmic expected depth before finding a correct path in domains where correct reasoning is locally checkable.

The fitness signal from such a system is hierarchical: the PRM provides per-step signal (intrinsic, dense), the verifier provides per-solution signal (extrinsic, binary), and the MCTS search statistics (visit counts, value estimates per subtree) provide structural information about which reasoning strategies are productive.

---

## 218.4 Self-Generated Fitness Signals

A model that can assess its own capability state can generate targeted test cases in areas where its capability is weakest — using its meta-cognitive awareness (cross-ref [Chapter 217 — Self-Reflective Inference](ch217-self-reflective-inference.md)) to direct test generation toward capability frontiers rather than familiar domains.

### Self-Play Adversarial Evaluation

Self-play adversarial evaluation separates the model into two roles: a Question Designer that generates challenging test cases, and an Answerer that attempts them. The Question Designer is prompted to construct problems at the edge of the model's capability — problems that its own previous responses suggest it may fail on, but that a capable system should be able to solve.

The self-referential structure is not circular if the two roles operate with different in-context information:

- The Question Designer sees: the domain, previous failures or near-failures, and instructions to design problems at the capability boundary.
- The Answerer sees: only the problem statement, with no access to the Question Designer's reasoning or the failure history.

The fitness signal is the Answerer's correctness rate on Question Designer-generated problems, evaluated by an external verifier (a unit test, a symbolic solver, a human annotator) rather than by the model itself.

### The Self-Evaluation Loop

```python
from dataclasses import dataclass
from typing import Protocol

class ModelInferencer(Protocol):
    def generate(self, prompt: str, role_context: str) -> str: ...
    def score(self, problem: str, answer: str) -> bool: ...

@dataclass
class EvalRound:
    domain: str
    generated_problem: str
    model_answer: str
    verified_correct: bool

def self_eval_loop(
    model: ModelInferencer,
    domain: str,
    n_rounds: int,
    failure_history: list[str] | None = None,
) -> list[EvalRound]:
    history = failure_history or []
    results: list[EvalRound] = []

    for _ in range(n_rounds):
        designer_prompt = _build_designer_prompt(domain, history)
        problem = model.generate(designer_prompt, role_context="question_designer")

        answer_prompt = _build_answerer_prompt(problem)
        answer = model.generate(answer_prompt, role_context="answerer")

        correct = model.score(problem, answer)
        if not correct:
            history.append(problem)

        results.append(EvalRound(domain, problem, answer, correct))

    return results

def _build_designer_prompt(domain: str, failures: list[str]) -> str:
    failure_block = "\n".join(f"- {p}" for p in failures[-5:])
    return (
        f"Domain: {domain}\n"
        f"Recent failures:\n{failure_block}\n"
        "Design a new problem at the capability boundary. "
        "It should be solvable by an expert but challenging for the current system."
    )

def _build_answerer_prompt(problem: str) -> str:
    return f"Solve the following problem, showing your reasoning:\n{problem}"
```

The aggregate fitness signal from `self_eval_loop` is the per-domain success rate across rounds — a quantity that tracks capability at the edge rather than the mean.

### Capability Boundary Estimation

The self-evaluation loop produces a sequence of (problem, correct/incorrect) pairs indexed by round. The capability boundary in a domain can be estimated by treating this sequence as a sequence of Bernoulli trials with a latent difficulty parameter: problems generated in later rounds (after accumulating failure history) are, on average, harder. A logistic regression of correctness on round index estimates the failure rate at the capability boundary — the asymptotic difficulty level beyond which the model fails reliably.

This estimate has a direct interpretation as a fitness signal: a self-modified model whose failure rate at the capability boundary decreases relative to the baseline model has improved at the capability frontier, not just at easy problems that were already solvable. Measuring improvement at the frontier is the correct fitness objective for a self-improvement loop targeting genuine capability gain rather than benchmark inflation.

The external verification step is non-negotiable. Without it, the Question Designer and Answerer may collude to produce problems whose apparent difficulty is high (the Designer generates a problem that looks hard) but whose answers are easy to confirm (the problem has a structure that the Answerer can exploit without solving the intended task). External verification — by a symbolic solver, a unit test, or a human — breaks this collusion by providing a ground-truth signal independent of the model.

---

## 218.5 Dynamic Benchmark Generation and LiveBench

Static benchmarks have a lifespan. A benchmark that discriminates between model capabilities when first published degrades over time as models are trained on data that overlaps with its test items, and as benchmark-specific fine-tuning narrows the distribution gap between model training data and benchmark format.

### Benchmark Saturation and Contamination

The contamination problem operates at two levels. First, direct contamination: test items or near-duplicates appear in the model's pre-training corpus. Second, format contamination: the model is fine-tuned on benchmark-format data that shares structural patterns with the test items without sharing content. Both degrade the benchmark's discriminative validity — the degree to which high scores reflect the target capability rather than benchmark-specific learning.

Detection methods include n-gram overlap between test items and training data, perplexity-based contamination scores (test items should have higher perplexity than training items under the target model), and task-date filtering: excluding items whose source documents appeared before the model's training cutoff. A more sensitive contamination test is membership inference attack: a classifier trained to distinguish model-seen from model-unseen data using the model's per-token log-probabilities. Items the model can "recall" verbatim — detected by high sequence-level log-probability relative to an independently trained reference model — are excluded as potentially contaminated, regardless of their source date. The reference model approach (Shi et al. 2023, "Detecting Pretraining Data from Large Language Models", arXiv 2310.16789) provides a principled statistical test: a test item is flagged as contaminated when its log-likelihood ratio between target and reference model exceeds a threshold calibrated to control false positive rate.

### LiveBench Design Principles

LiveBench (White et al., arXiv 2406.19314) addresses saturation by monthly refresh of all benchmark items using problems derived from sources published after the evaluation date. Key design principles:

| Principle | Mechanism | Contamination Guarantee |
|-----------|-----------|------------------------|
| Temporal filtering | All source materials post-date the evaluation | No pre-training overlap for any evaluated model |
| Automated scoring | Ground-truth answers derivable without human annotation | Enables high-throughput refresh |
| Task diversity | Math, coding, reasoning, language, data analysis | Avoids narrow-domain gaming |
| Public leaderboard | Scores are not used for training | No incentive-to-leak |

The monthly refresh cycle creates an evolutionary pressure on the evaluation: models that improve by genuine generalisation maintain their score rank across refreshes, while models that improve by benchmark memorisation lose rank when items change. This transforms the benchmark from a static target into a co-evolutionary adversary — the evaluation adapts to model improvements rather than being fixed.

### Model-Generated Evaluation

An extension of dynamic benchmarking uses the model itself as a test-item generator, constrained by a verifier that checks answer correctness. For mathematical problems, this means: the model proposes a problem statement plus ground-truth answer, a separate symbolic solver (Wolfram Alpha API, Lean 4 proof checker, Python interpreter) verifies the answer, and only verified (problem, answer) pairs enter the evaluation pool. The model's self-generated test cases are outside its training distribution by construction — they are novel rather than reproduced — while the verifier ensures correctness without requiring human annotation.

The generation-verification loop requires care at three points. First, the model must be prompted to generate problems with computable ground-truth answers — open-ended problems (e.g., "explain the significance of X") cannot be verified automatically. Second, the verifier must be more reliable than the generator: a verifier that accepts 5% incorrect answers will corrupt the evaluation pool. Third, the generated problem distribution must be diverse: without explicit diversity incentives, the model will generate many near-duplicate problems that collectively test a narrow slice of the target capability.

Diversity can be enforced by embedding each generated problem in a fixed embedding space (e.g., using a sentence encoder), clustering the pool, and rejecting new problems whose nearest cluster contains more than a threshold number of items. This is the same diversity enforcement used in Quality-Diversity archives ([Chapter 215](ch215-evolutionary-architecture-search.md), §215.4), applied now to the evaluation pool rather than the solution archive.

---

## 218.6 Interpretability-as-Evaluation

Pre/post modification comparison of SAE feature activations provides a fitness signal that does not require any external task evaluation — it reads the model's internal state directly. This is particularly useful for evaluating the side effects of weight-space operations ([Chapter 212 — Weights as a Programming Substrate](ch212-weights-as-programming-substrate.md)) that may preserve downstream task performance while altering the internal computational structure.

### Comparing SAE Activations Before and After Modification

```python
import torch
import numpy as np
from transformer_lens import HookedTransformer

def compare_sae_features(
    model_before: HookedTransformer,
    model_after: HookedTransformer,
    sae_encoder: torch.nn.Module,
    prompts: list[str],
    hook_name: str = "blocks.12.hook_resid_post",
) -> dict[str, np.ndarray]:
    def get_sae_activations(model: HookedTransformer) -> torch.Tensor:
        model.eval()
        with torch.no_grad():
            _, cache = model.run_with_cache(prompts)
            acts = cache[hook_name][:, -1, :]
            return sae_encoder(acts)  # (N, d_sae)

    feats_before = get_sae_activations(model_before)
    feats_after = get_sae_activations(model_after)

    mean_before = feats_before.mean(0).cpu().numpy()
    mean_after = feats_after.mean(0).cpu().numpy()
    delta = mean_after - mean_before

    top_increased = np.argsort(delta)[-20:][::-1]
    top_decreased = np.argsort(delta)[:20]

    sparsity_before = (feats_before == 0).float().mean().item()
    sparsity_after = (feats_after == 0).float().mean().item()

    return {
        "delta": delta,
        "top_increased_features": top_increased,
        "top_decreased_features": top_decreased,
        "sparsity_before": sparsity_before,
        "sparsity_after": sparsity_after,
    }
```

The `delta` array is the primary fitness signal: large positive entries indicate features that became more strongly activated after the modification; large negative entries indicate features that were suppressed. For a modification intended to improve mathematical reasoning, the expected pattern is increased activation of features corresponding to algebraic manipulation and symbolic reasoning, with no suppression of features corresponding to language generation, code understanding, or logical inference.

### Attribution Graphs as Capability Maps

Attribution graphs (§213.7 in [Chapter 213](ch213-mechanistic-interpretability.md)) map information flow through the model for a given input class. For capability assessment, the relevant question is whether the circuits that implement a target capability are structurally intact after a modification.

A lightweight version that does not require full attribution graph construction is a per-head importance score for a capability domain:

```python
def head_importance_for_domain(
    model: HookedTransformer,
    domain_prompts: list[str],
    baseline_prompts: list[str],
) -> torch.Tensor:
    def mean_attention_entropy(prompts: list[str]) -> torch.Tensor:
        model.eval()
        with torch.no_grad():
            _, cache = model.run_with_cache(prompts)
        entropies = []
        for layer in range(model.cfg.n_layers):
            attn = cache[f"blocks.{layer}.attn.hook_pattern"]  # (N, H, T, T)
            ent = -(attn * (attn + 1e-8).log()).sum(-1).mean((0, 2))  # (H,)
            entropies.append(ent)
        return torch.stack(entropies)  # (L, H)

    domain_ent = mean_attention_entropy(domain_prompts)
    baseline_ent = mean_attention_entropy(baseline_prompts)
    return (domain_ent - baseline_ent).abs()
```

Heads with high importance score for a domain show systematically different attention entropy patterns on domain prompts versus baseline. A post-modification drop in importance score for a previously important head is a structural warning that the capability circuit may have been disrupted, independent of any task performance measurement.

### Feature Sparsity as a Health Metric

SAE feature sparsity — the fraction of features that are zero on a given input — is a proxy for model health after self-modification. A well-trained SAE has high sparsity (most features are inactive for any given input; only a small number are relevant). Decreased sparsity after a weight-space modification indicates that the modification induced interference in the SAE's representation: features that should be inactive are being spuriously activated. This is the neural analogue of register spill — the model is "using more resources" to represent the same information, which typically correlates with decreased capability precision.

---

## 218.7 Automated Capability Regression Testing

A regression test suite for weight-space operations is a set of capability probes — each a (prompt set, expected behaviour, scoring function) triple — that must continue to pass after any self-modification. This is the weight-space analogue of a continuous integration suite for source code.

### Design of the Regression Suite

Each probe in the suite tests a distinct capability in a form where correctness is mechanically verifiable:

| Capability | Probe Type | Verifier |
|------------|------------|----------|
| Integer arithmetic | 3-digit arithmetic problems | Python `eval` |
| Code generation | unit test with scaffolding | `pytest` runner |
| Logical inference | modus ponens / syllogism | symbolic checker |
| Named entity recall | factual completion | string match |
| Language fluency | perplexity on held-out text | language model perplexity |
| Instruction following | structured output parsing | JSON schema validator |

The regression suite should be frozen at the time of the baseline measurement and not updated after self-modification cycles begin. Updating the regression suite invalidates the baseline comparison.

### Running the Suite

```python
import dataclasses
from typing import Callable

@dataclasses.dataclass
class CapabilityProbe:
    name: str
    prompts: list[str]
    expected: list[str]
    verifier: Callable[[str, str], bool]
    weight: float = 1.0

def capability_regression_test(
    theta_before: dict,
    theta_after: dict,
    probe_suite: list[CapabilityProbe],
    model_runner: Callable,
    threshold: float = 0.95,
) -> dict[str, float]:
    scores_before: dict[str, float] = {}
    scores_after: dict[str, float] = {}

    for probe in probe_suite:
        outputs_before = [model_runner(theta_before, p) for p in probe.prompts]
        outputs_after = [model_runner(theta_after, p) for p in probe.prompts]

        acc_before = sum(
            probe.verifier(o, e)
            for o, e in zip(outputs_before, probe.expected)
        ) / len(probe.prompts)
        acc_after = sum(
            probe.verifier(o, e)
            for o, e in zip(outputs_after, probe.expected)
        ) / len(probe.prompts)

        scores_before[probe.name] = acc_before
        scores_after[probe.name] = acc_after

    regressions = {
        name: scores_after[name] / scores_before[name]
        for name in scores_before
        if scores_before[name] > 0
    }
    failed = {k: v for k, v in regressions.items() if v < threshold}
    return {"regressions": failed, "ratios": regressions}
```

### Connection to Verified Compilation

The structure of `capability_regression_test` parallels the translation validation approach of Alive2 ([Chapter 170 — Alive2 and Translation Validation](../part-24-verified-compilation/ch170-alive2-and-translation-validation.md)): both check that a transformation preserves a specified set of observable behaviours. Alive2 checks semantic equivalence over all inputs; the regression suite checks correctness over a finite probe set. The probe set is necessarily incomplete — it cannot enumerate all inputs — but the finite check is computable, whereas the full semantic equivalence check for neural networks is not.

The formal connection to the Gödel Machine is direct: `PreservesCapabilities(θ_before, θ_after)` is a necessary condition for `V(t, apply(r)) > V(t, stop)`. Any self-rewrite that fails the regression suite has decreased expected utility under any reasonable utility function that assigns positive value to the probed capabilities. The regression suite is therefore a necessary (not sufficient) filter on the Gödel Machine's self-rewrite condition.

---

## 218.8 Fitness Landscape Topology

The fitness function does not exist in isolation — it defines a landscape over the parameter space, and the geometry of that landscape determines what self-improvement mechanisms can and cannot find.

### Sharpness, Basin Width, and Generalisation

The classical result of Hochreiter & Schmidhuber (1997) on flat minima establishes that minima with low curvature (wide basins in the loss landscape) generalise better than minima with high curvature (sharp basins). The intuition is information-theoretic: a wide-basin minimum is one where a large volume of parameter space achieves low loss; this large volume is a sign that the model has not overfit the training distribution. A sharp minimum, by contrast, is one that is finely tuned to a small region of parameter space; small perturbations increase loss rapidly, which correlates with overfitting.

Keskar et al. (2017, arXiv 1609.04836) operationalise sharpness as the maximum eigenvalue of the Hessian of the loss at the minimum: `λ_max(∇²L(θ))`. Large-batch training tends to converge to sharper minima than small-batch training; small-batch training with low learning rates tends to find flatter minima with better generalisation. The fitness function implication is: for evolutionary search, a fitness function that explicitly rewards flat minima (via a sharpness penalty) should produce better-generalising solutions than raw task accuracy.

### Computing Loss Curvature

```python
import jax
import jax.numpy as jnp

def hessian_diagonal(
    loss_fn,
    params: dict,
    batch: dict,
) -> dict:
    grad_fn = jax.grad(loss_fn, argnums=0)

    def scalar_param_loss(p_vec, shapes, treedef):
        params_tree = jax.tree_util.tree_unflatten(
            treedef, [p_vec[s:e].reshape(sh) for sh, (s, e) in shapes]
        )
        return loss_fn(params_tree, batch)

    leaves, treedef = jax.tree_util.tree_flatten(params)
    shapes, idx, offset = [], [], 0
    for leaf in leaves:
        n = leaf.size
        shapes.append((leaf.shape, (offset, offset + n)))
        idx.append(n)
        offset += n

    p_vec = jnp.concatenate([l.ravel() for l in leaves])
    hess_diag = jax.vmap(
        lambda i: jax.grad(lambda pv: scalar_param_loss(pv, shapes, treedef))(p_vec)[i]
    )(jnp.arange(len(p_vec)))
    return hess_diag
```

The Hessian diagonal approximation is O(P) in memory and O(P) in forward/backward passes, compared to O(P²) for the full Hessian, making it feasible for models with up to ~100M parameters. For larger models, randomised trace estimators (Hutchinson's estimator) provide an unbiased estimate of `tr(H)` in O(1) Hessian-vector products.

### MAP-Elites and the Fitness Landscape

The Quality-Diversity framework of [Chapter 215](ch215-evolutionary-architecture-search.md) (§215.4) navigates the fitness landscape by maintaining a diverse archive of solutions, indexed by behaviour descriptors, rather than converging on a single optimum. In the language of landscape topology, MAP-Elites explores the basin structure of the fitness landscape: it does not converge to a single minimum but maps out which regions of the behaviour space have accessible fitness basins.

This is directly useful for fitness function design: if MAP-Elites applied with two different fitness functions produces different behaviour-space coverage, the difference characterises what capabilities each fitness function rewards. A fitness function that concentrates the archive in a narrow region of behaviour space is over-specifying a proxy; a fitness function that allows broad coverage while maintaining high fitness values is closer to a general capability measure.

### Filter-Normalised Landscape Visualisation

Li et al. (2018, arXiv 1712.09913) introduce filter-normalised loss landscape visualisation: project the loss surface onto a random 2D plane in parameter space, normalised by the filter (weight matrix row) norms to remove scale artefacts. The resulting surface reveals whether the minimum is in a flat bowl (wide basin, good generalisation) or a narrow valley (sharp minimum, likely overfitting). This visualisation is directly applicable to comparing weight-space merge products ([Chapter 212](ch212-weights-as-programming-substrate.md)): a merged model that falls in a flat basin is preferable to one at a sharp minimum, even if both achieve similar task accuracy, because the flat-basin model is more likely to generalise.

### Sharpness-Aware Minimisation as Fitness Regularisation

Sharpness-Aware Minimisation (SAM, Foret et al. 2021, arXiv 2010.01412) is a training procedure that explicitly penalises sharpness by seeking parameters that minimise the worst-case loss in a neighbourhood of radius `ρ`:

```
θ* = argmin_θ max_{||ε||≤ρ} L(θ + ε)
```

This is a min-max optimisation: the inner maximisation finds the worst perturbation within the neighbourhood; the outer minimisation seeks the parameter `θ` where even the worst perturbation gives low loss. SAM is operationalised as a two-step update: one gradient step to find the worst `ε`, followed by a gradient step on the perturbed loss. The result is a model whose minimum is in a wider basin by construction.

In the self-improvement context, SAM's objective is a structured fitness function for weight-space operations: not "achieve low training loss" but "achieve low training loss in a wide basin." Applied as a fitness criterion in evolutionary architecture search ([Chapter 215](ch215-evolutionary-architecture-search.md)), this means preferring candidate modifications that produce flat minima over those that produce sharp minima at equal task accuracy. The sharpness penalty `max_{||ε||≤ρ} L(θ + ε) - L(θ)` is a single-query fitness signal requiring two forward-backward passes — computationally accessible even at large scale.

---

## 218.9 Intrinsic vs Extrinsic Fitness

All fitness signals decompose into two categories by their origin: intrinsic signals derived from the model's own internal state, and extrinsic signals derived from external evaluation.

| Signal Type | Source | Examples | Goodhart Vulnerability |
|-------------|--------|----------|----------------------|
| **Intrinsic — probing** | Internal activations | Linear probe accuracy, SAE feature activation | Low — reads representation, not output |
| **Intrinsic — PRM** | Learned reward model | Per-step score (PRM800K) | Medium — reward model can be hacked |
| **Intrinsic — self-play** | Model generates and evaluates | Self-eval loop correctness rate | High — evaluator bias matches subject bias |
| **Extrinsic — static benchmark** | Fixed test set | MMLU, HumanEval, MATH | High — saturates, contamination risk |
| **Extrinsic — dynamic benchmark** | Refreshed test set | LiveBench monthly refresh | Low — temporal filtering prevents contamination |
| **Extrinsic — formal verifier** | External symbolic system | Lean 4 proof check, pytest runner | Minimal — correctness is binary and unambiguous |
| **Interpretability** | SAE + attribution graph | Feature activation delta, head importance | Low — reads circuits, not performance |

### Hybrid Fitness

The most robust fitness function combines signals from multiple rows of this table. A hybrid fitness can be written as a weighted sum:

```
F(θ) = w₁ · ProbeAccuracy(θ) + w₂ · PRMScore(θ) + w₃ · BenchmarkScore(θ) + w₄ · SparsityHealth(θ)
```

where each term is normalised to [0, 1] and the weights `w₁, ..., w₄` sum to 1. The interpretability terms (`ProbeAccuracy`, `SparsityHealth`) are less vulnerable to Goodhart drift because they read internal structure rather than output distributions; the task terms (`BenchmarkScore`) provide extrinsic grounding; the PRM term provides dense intermediate-step signal.

### The Bootstrapping Problem

Using the current model to evaluate improvements to itself — the bootstrapping problem — is unavoidable in any fully autonomous self-improvement loop. The structural mitigation is a separation of concerns: the evaluator model is frozen at a known baseline while the subject model is modified. The evaluator is periodically re-calibrated against an external ground truth (a formal verifier, human evaluation, or a held-out model family). Re-calibration frequency is a hyperparameter of the self-improvement loop: too frequent, and the evaluator cannot track cumulative improvement; too infrequent, and evaluator drift degrades the fitness signal.

A second mitigation is cross-family evaluation: use model A to evaluate modifications to model B, and model B to evaluate modifications to model A. If A and B have independent failure modes, their cross-evaluations will be less correlated than self-evaluations, partially breaking the coherent-error failure mode of pure self-evaluation.

### Signal Aggregation and Calibration

When combining multiple fitness signals into a hybrid score, calibration is as important as signal selection. A signal that is trivially satisfied by all candidates contributes no information; a signal that discriminates on irrelevant dimensions adds noise. Calibration proceeds in three steps:

1. **Baseline distribution estimation**: run the fitness battery on a sample of known-quality models (e.g., checkpoints from earlier training steps, models of different sizes) to establish the empirical distribution of each signal.
2. **Normalisation**: map each signal to [0, 1] using the baseline distribution's empirical CDF. A model at the 90th percentile of SAE sparsity health in the baseline distribution receives a normalised score of 0.9.
3. **Weight assignment**: assign weights based on the signal's estimated reliability (inverse of Goodhart vulnerability) and discriminative power (standard deviation of normalised scores across the baseline distribution). Signals with low variance across the baseline are down-weighted; signals that reliably distinguish high-quality from low-quality models are up-weighted.

This calibration protocol must be re-run whenever the baseline distribution changes — for example, after a major capability improvement that shifts all models to a new performance tier. Treating the hybrid fitness weights as fixed constants across all generations of the self-improvement loop is the primary cause of evaluator drift in long-running loops.

The formal connection to the `drift : Impl[T] → Spec[T] → DriftReport` framework of [Chapter 207](../part-30-AI-first-PL-design/ch207-reflective-code-open-problems-roadmap.md) is direct: the fitness function calibration procedure is `drift` applied to the fitness function itself, treating the original calibration as `Spec[T]` and the current calibration as `Impl[T]`. Continuous monitoring of calibration drift — tracking whether the normalised fitness scores of the current baseline population remain near their expected values — is the operational implementation of the fitness function audit cycle.

---

## 218.10 Synthesis: The Full Self-Improvement Loop

The complete self-improvement loop integrates all components developed across Chapters 208–218 into a seven-step cycle:

1. **Compile** ([Chapter 208–211](ch208-gpu-kernel-dsls-triton-helion-gluon.md)): the model is compiled to an executable artifact — CUDA kernels, Triton programs, quantised weights — for a target hardware configuration.

2. **Infer** ([Chapter 211](ch211-neural-programs-compiled-artifacts.md)): the compiled model runs inference on a task distribution, producing outputs and activations.

3. **Profile** ([Chapter 211](ch211-neural-programs-compiled-artifacts.md), §211.4): runtime profiling identifies bottleneck layers, inefficient attention patterns, and underutilised compute capacity.

4. **Introspect** ([Chapter 213](ch213-mechanistic-interpretability.md), [Chapter 217](ch217-self-reflective-inference.md)): SAE feature activations, attribution graphs, and self-reflective inference characterise the model's current capability state — which features are active, which circuits are engaged, where performance degrades.

5. **Modify** ([Chapter 212](ch212-weights-as-programming-substrate.md)–[215](ch215-evolutionary-architecture-search.md)): a modification mechanism — weight-space merge, gradient-based editing, evolutionary search — produces a candidate modified model.

6. **Recompile** ([Chapter 208](ch208-gpu-kernel-dsls-triton-helion-gluon.md)–[211](ch211-neural-programs-compiled-artifacts.md)): the modified model is recompiled to an executable artifact for the target hardware.

7. **Evaluate** (this chapter): the fitness function battery — regression tests, PRM scores, probe accuracy, SAE feature delta, dynamic benchmark score — determines whether the modification is accepted, rejected, or flagged for further analysis.

The loop is self-sustaining when the evaluate step produces a signal informative enough to guide the modify step, and the modify step is expressive enough to act on that signal. The fitness function is the closing link: without it, the loop cannot distinguish improvement from degradation and cannot self-sustain.

### Relationship to Self-Reflection

[Chapter 217 — Self-Reflective Inference](ch217-self-reflective-inference.md) establishes that the inference step itself can include an explicit self-evaluation phase: the model produces a candidate output, produces a critique of that output, and revises based on the critique. In the loop framework, step 4 (Introspect) and step 7 (Evaluate) overlap: self-reflective inference is simultaneously the introspection step (the model reads its own intermediate outputs) and a lightweight evaluation step (the model assesses whether the intermediate output is correct).

The connection is not circular. Self-reflective inference is an evaluation of a candidate output by the same model that generated it — the self-referential problem of §218.1. The external fitness signals developed in §218.2–218.7 break the circularity by providing evaluation signals that are independent of the model's generation capability. Self-reflection functions as a cheap first filter; external fitness evaluation provides the ground-truth signal that trains the self-reflection mechanism to be calibrated.

### Open Problems

Three open problems prevent the loop from being formally closed:

**The specification problem**: no computable formal specification exists for open-ended capability. The Gödel Machine's `V(t, r)` requires full formalisation; the `drift : Impl[T] → Spec[T] → DriftReport` framework from [Chapter 207](../part-30-AI-first-PL-design/ch207-reflective-code-open-problems-roadmap.md) measures drift from a partial spec but cannot construct the full spec automatically.

**The evaluator bootstrapping problem**: as the subject model improves, the evaluator model must improve commensurately to remain discriminative. A fixed evaluator becomes a fixed ceiling; a self-improving evaluator introduces the coherent-error failure mode.

**Goodhart's Law at scale**: any fitness function that becomes the optimisation target for a sufficiently capable optimiser will be gamed. The dynamic benchmark approach of §218.5 partially addresses this for task benchmarks; no known technique fully addresses it for learned reward models.

---

## Chapter Summary

- The oracle problem is the central challenge: every feasible fitness function is a computable proxy for an uncomputable ideal (the Gödel Machine's `V(t, r)`); understanding where the proxy breaks is the engineering task.
- Linear probing on frozen activations converts internal representations into quantitative capability scores without requiring task evaluation; probe accuracy measures whether a capability is linearly accessible at a given layer.
- SAE feature activations (Ch213) provide monosemantic capability fingerprints; pre/post modification comparison via `compare_sae_features` detects structural side effects invisible to task performance metrics.
- Process reward models (PRM800K, arXiv 2305.20050) provide dense per-step fitness signals that outperform outcome reward models for best-of-N selection and enable step-level beam pruning in evolutionary search.
- Self-generated fitness signals — self-play adversarial evaluation, capability-boundary test generation — target the frontier of current capability; the `self_eval_loop` algorithm operationalises this as a closed loop with external verification.
- Dynamic benchmarks (LiveBench, arXiv 2406.19314) address benchmark saturation via temporal filtering and monthly refresh; model-generated test items with external symbolic verification extend dynamic evaluation to arbitrary capability domains.
- Automated capability regression testing (`capability_regression_test`) is a necessary (not sufficient) filter on self-modification: any rewrite that fails the suite has decreased expected utility under any utility function that assigns positive value to the probed capabilities.
- Fitness landscape topology — sharpness, basin width, filter-normalised visualisation — characterises where gradient descent and evolutionary search can and cannot reach; flat minima generalise better and should be explicitly rewarded.
- Intrinsic fitness signals (probing, SAE, self-play) are less vulnerable to Goodhart's Law than extrinsic benchmarks; hybrid fitness functions combining both types are more robust than either alone.
- The seven-step self-improvement loop (compile → infer → profile → introspect → modify → recompile → evaluate) is self-sustaining only when the evaluate step is informative enough to guide the modify step; open problems in formal specification, evaluator bootstrapping, and Goodhart scaling remain unsolved.
