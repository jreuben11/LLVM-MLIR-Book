# Chapter 217 — Self-Reflective Inference and Architecture Introspection

*Part XXXI — Frontier AI Evolution*

The standard inference loop is a one-way transaction: a fixed weight tensor consumes a prompt and emits a completion, with no mechanism for the model to assess whether its reasoning is reliable, its knowledge current, or its capabilities appropriate to the task. Every technique in Chapters 208–216 assumes this loop as the baseline and then modifies the *weights* to alter that behaviour. This chapter concerns a different intervention: instead of changing what the model knows, we change *how it reasons about itself during the forward pass*. Self-reflective inference redirects test-time compute away from external task solving and toward self-understanding — identifying knowledge gaps, tracing reasoning failures, critiquing and refining outputs, probing activation patterns to assess capability state, and generating synthetic tests in areas of discovered weakness. This is not introspection as a philosophical luxury. It is the computational mechanism that closes the loop between passive inference and active self-modification: without runtime self-assessment, [Chapter 214 — Gradient-Based Self-Modification](ch214-gradient-based-self-modification.md) has no signal specifying which parameters to edit, [Chapter 215 — Evolutionary Architecture Search](ch215-evolutionary-architecture-search.md) has no fitness evaluation to guide mutation, and the formal guarantees of [Chapter 216 — Formal Self-Improvement Theory](ch216-formal-self-improvement-theory.md) have no computable oracle for "beneficial". The chapter is structured as follows. Section 217.1 frames the problem and establishes self-knowledge as a first-class computational primitive. Sections 217.2–217.4 cover the three core prompt-level techniques — chain-of-thought self-debugging, iterative self-refinement (SELF-REFINE), and self-play fine-tuning (SPIN). Section 217.5 moves below the token level to activation-based introspection using TransformerLens. Section 217.6 develops self-generated evaluation sets as capability probes. Section 217.7 examines DeepSeek-R1 as a reference implementation of inference-time self-reflection at scale. Sections 217.8–217.9 close the loop to test-time training and to the evolutionary improvement architecture.

Cross-references: [Chapter 212 — Weights as a Programming Substrate](ch212-weights-as-programming-substrate.md) · [Chapter 213 — Mechanistic Interpretability Infrastructure](ch213-mechanistic-interpretability.md) · [Chapter 214 — Gradient-Based Self-Modification](ch214-gradient-based-self-modification.md) · [Chapter 215 — Evolutionary Architecture Search](ch215-evolutionary-architecture-search.md) · [Chapter 216 — Formal Self-Improvement Theory](ch216-formal-self-improvement-theory.md) · [Chapter 218 — Self-Improvement Fitness Functions and Capability Assessment](ch218-self-improvement-fitness-functions.md)

---

## 217.1 Test-Time Compute for Self-Understanding

### External Reasoning vs. Self-Directed Reasoning

Test-time compute scaling — allocating more tokens, more forward passes, or more gradient steps to a single inference — has been studied almost exclusively in the context of *external* task solving: chain-of-thought reasoning over math problems, Monte Carlo tree search over game states, Best-of-N sampling for code synthesis. In all of these, the model's additional computation is directed at the problem domain. The model asks, in effect, "What is the answer to this question?" not "What are my limitations in answering this class of questions?"

Self-directed test-time compute inverts this focus. The model allocates inference tokens and forward passes to reasoning about its *own* knowledge state, capability boundaries, and reasoning reliability. This requires a qualitatively different prompt structure, a different evaluation criterion for the reasoning trace, and — when activation-level introspection is involved — a different instrumentation infrastructure. The payoff is a capability that purely external reasoning cannot provide: a runtime signal specifying *where* and *how* the model's performance is degraded, which can drive targeted adaptation rather than wholesale retraining.

The distinction maps precisely onto the compiler workflow: a compiler that runs its analysis passes on the target program is performing external reasoning; a compiler that profiles its own analysis times and infers that a particular optimization pass is the bottleneck is performing self-directed reasoning. Profile-guided optimization and autotuning are the compiler discipline's closest analogs. TTT (§217.8) is the neural network equivalent of profile-guided recompilation, but it requires a profiling step to identify *which* aspects of the test input the model is handling poorly before it can direct the gradient usefully.

### Self-Knowledge as a Computational Primitive

Treating self-knowledge as a computational primitive has three concrete implications. First, it requires that the model's introspective outputs — confidence estimates, failure diagnoses, capability boundary assessments — be *structured* rather than free-form. An unstructured statement that "I'm not sure about this" carries no information about which specific capability is degraded or how large the degradation is. A structured output that identifies the specific sub-step where the reasoning diverged from a checkable fact is actionable — it can be converted into a targeted query, a TTT training signal, or an evolutionary fitness signal.

Second, it requires that the introspective signal be *calibrated*. A model that expresses high confidence in systematically incorrect outputs is worse than useless — it provides a misleading signal that could corrupt downstream adaptation. Calibration is not guaranteed by instruction-tuning; it must be enforced by training on verification data (for the structured critique) and, at the activation level, by probing classifiers trained on held-out data from the capability in question.

Third, treating self-knowledge as a primitive connects inference to the evolutionary loop of [Chapter 215](ch215-evolutionary-architecture-search.md). In the Darwin Gödel Machine, each agent is evaluated by a fitness function; inference *is* the evaluation step. A self-reflective agent that produces a structured assessment of its own capability state is not merely providing an output — it is providing the fitness signal that drives the next round of mutation. The agent that can articulate "I fail on this class of inputs at step 3 of my reasoning chain" has already solved half the fitness evaluation problem.

### The Taxonomy of Self-Directed Computation

The techniques in this chapter span four distinct levels of self-direction, differentiated by the depth of access to the model's own internals:

| Level | Technique | Access | Output |
|---|---|---|---|
| L0 — Output-level | SELF-REFINE (§217.3) | Final token output only | Refined output |
| L1 — Logit-level | SPIN discriminator (§217.4) | Token log-probabilities | Policy update |
| L2 — Activation-level | TransformerLens probe (§217.5) | Residual stream at layer L | Capability score |
| L3 — Gradient-level | TTT with targeting (§217.8) | Full parameter gradients | Weight update |

Each level provides a qualitatively different type of self-knowledge. L0 techniques are fully model-agnostic — they work on any black-box model that accepts text prompts. L1 techniques require access to the model's log-probabilities — available in most open-source models and through some API endpoints. L2 techniques require access to intermediate activations — available via hook-based instrumentation (TransformerLens, PyTorch hooks) but not typically through production APIs. L3 techniques require the ability to perform backward passes through the model — requiring either the full model in training mode or a LoRA adapter that exposes differentiable parameters.

Production deployments typically use L0 and L1 techniques exclusively, with L2 and L3 reserved for development-time capability assessment and offline fine-tuning. The four levels correspond to the compiler's static analysis (L2/L3, offline), runtime profiling (L1, inference-time logit access), and post-hoc output analysis (L0, output-only evaluation). The complete self-reflective inference architecture stacks all four: L0 self-refinement improves the immediate output quality; L1 SPIN fine-tuning improves the long-run policy; L2 probing assesses the current capability state; L3 TTT adapts parameters to the current test distribution.

---

## 217.2 Chain-of-Thought Self-Debugging

### Wei et al. and the Chain-of-Thought Result

[Wei et al. (2022), "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models", arXiv 2201.11903](https://arxiv.org/abs/2201.11903) demonstrated that eliciting intermediate reasoning steps before a final answer improves accuracy on arithmetic, commonsense, and symbolic reasoning benchmarks by margins of 10–50% for large models. The mechanism is not primarily that intermediate tokens carry additional information not present in the prompt — it is that intermediate tokens *constrain subsequent tokens*: a reasoning step that commits to a partially correct intermediate result removes from the completion distribution all tokens that are inconsistent with it, channelling the model toward the correct final answer. This is the token-level equivalent of enforcing loop invariants: intermediate assertions narrow the feasible completion space.

The self-debugging application of CoT reverses the direction. Instead of using intermediate reasoning steps to reach a better *answer*, the model uses intermediate reasoning steps to reach a better *diagnosis of its own failure*. Given a (question, incorrect-answer) pair, the model is prompted to trace the reasoning chain that produced the incorrect answer, identify the first step where the reasoning diverges from a checkable fact or logical rule, and attribute the error to a specific type — factual gap, arithmetic error, logical fallacy, distributional mismatch, or missing context.

### Self-Diagnosis Prompting Patterns

The prompt structure for CoT self-debugging has three obligatory components: a *recall* phase (reproduce the original reasoning trace), an *inspect* phase (check each step against a stated rule or fact), and an *attribute* phase (name the failure mode and its causal step). The attribute phase is critical — it is the output that can be converted into a structured training signal or fitness assessment.

```python
SELF_DEBUG_TEMPLATE = """\
You produced the following answer to a question. Trace through your
reasoning step-by-step and identify where it went wrong.

Question: {question}
Your answer: {wrong_answer}

Step 1 — RECALL: Write out the full reasoning chain that produced this answer.
         Be explicit: list each inference you made, in order.

Step 2 — INSPECT: For each step, check:
         (a) Is this a factual claim? If so, is it verifiable?
         (b) Is this an arithmetic operation? Check it explicitly.
         (c) Is this a logical inference? State the rule applied.

Step 3 — ATTRIBUTE: Identify the FIRST step where the reasoning fails.
         Classify the failure as one of:
         [FACTUAL_GAP | ARITHMETIC_ERROR | LOGICAL_FALLACY |
          DISTRIBUTIONAL_MISMATCH | MISSING_CONTEXT | UNKNOWN]
         Explain what correction would be needed at that step.

Begin with Step 1:
"""

def self_debug(model_fn, question: str, wrong_answer: str) -> dict:
    prompt = SELF_DEBUG_TEMPLATE.format(
        question=question,
        wrong_answer=wrong_answer,
    )
    trace = model_fn(prompt)
    failure_type = _extract_failure_type(trace)
    failure_step = _extract_failure_step(trace)
    return {
        "trace": trace,
        "failure_type": failure_type,
        "failure_step": failure_step,
    }

def _extract_failure_type(trace: str) -> str:
    tags = [
        "FACTUAL_GAP", "ARITHMETIC_ERROR", "LOGICAL_FALLACY",
        "DISTRIBUTIONAL_MISMATCH", "MISSING_CONTEXT", "UNKNOWN",
    ]
    for tag in tags:
        if tag in trace:
            return tag
    return "UNKNOWN"

def _extract_failure_step(trace: str) -> int | None:
    import re
    m = re.search(r"[Ss]tep\s+(\d+)", trace.split("ATTRIBUTE")[-1])
    return int(m.group(1)) if m else None
```

The `failure_type` output is a structured signal: `FACTUAL_GAP` indicates that the model lacks a specific piece of world knowledge; `DISTRIBUTIONAL_MISMATCH` indicates that the input is out-of-distribution relative to the model's training; `ARITHMETIC_ERROR` can be corrected by tool use. Each failure type implies a different remediation path — factual gap may warrant a retrieval call or TTT update; distributional mismatch may warrant SPIN (§217.4); arithmetic error may warrant chain-of-thought with explicit verification steps.

---

## 217.3 Iterative Self-Refinement (SELF-REFINE)

### The Generate-Critique-Refine Loop

[Madaan et al. (2023), "SELF-REFINE: Iterative Refinement with Self-Feedback", arXiv 2303.17651](https://arxiv.org/abs/2303.17651) introduced the generate→critique→refine loop as a method for improving model outputs without any external feedback signal. The model produces an initial output, critiques that output against a set of quality criteria, and then produces a refined output conditioned on both the original output and the critique. The loop iterates until a stopping criterion is met. The key empirical finding is that this loop consistently improves outputs across code generation, mathematical reasoning, constrained generation, and dialogue tasks, even though the same model serves as both the generator and the critic — roles that might seem to create an echo chamber. The explanation is that *generating* a critique has different distributional properties from *generating* an output: the critique prompt explicitly solicits adversarial attention, which activates different completion patterns than the generation prompt does.

### Implementation

```python
from dataclasses import dataclass

@dataclass
class RefinementState:
    output: str
    critique: str
    iteration: int

GENERATION_PROMPT = """\
Task: {task}
Produce a solution:
"""

CRITIQUE_PROMPT = """\
Task: {task}
Solution: {output}

Critique this solution. List specific flaws under these categories:
- Correctness: Does it solve the task? Are there errors?
- Completeness: Are any required elements missing?
- Clarity: Is the reasoning clear and unambiguous?
- Efficiency: Is this the most direct approach?

If the solution is satisfactory with no significant flaws, write STOP.
Otherwise, enumerate each flaw explicitly.
"""

REFINEMENT_PROMPT = """\
Task: {task}
Previous solution: {output}
Critique: {critique}

Produce an improved solution that addresses each point in the critique:
"""

def self_refine(
    model_fn,
    task: str,
    max_iterations: int = 5,
) -> list[RefinementState]:
    history: list[RefinementState] = []

    current_output = model_fn(GENERATION_PROMPT.format(task=task))

    for iteration in range(max_iterations):
        critique = model_fn(CRITIQUE_PROMPT.format(
            task=task,
            output=current_output,
        ))

        history.append(RefinementState(
            output=current_output,
            critique=critique,
            iteration=iteration,
        ))

        if "STOP" in critique.split("\n")[0].upper():
            break

        current_output = model_fn(REFINEMENT_PROMPT.format(
            task=task,
            output=current_output,
            critique=critique,
        ))

    return history
```

### Convergence Properties and Stopping Criteria

Madaan et al. test three stopping criteria: fixed iteration count, critic emitting a no-improvement signal ("STOP"), and a score threshold when the task admits automatic scoring. The `"STOP"` check above is extracted from the first line of the critique to avoid false positives from embedded STOP strings in the body. In practice, Madaan et al. find that most gains accrue in the first one to two iterations; iterations beyond three rarely improve and occasionally regress — a phenomenon they attribute to the critique becoming less discriminating as the output improves (the critique prompt's adversarial signal weakens when there are fewer genuine flaws to find). A robust implementation therefore caps iterations at three to five and uses the `"STOP"` signal as the primary terminator. The `history` list enables post-hoc analysis of the refinement trajectory, which is the input to self-generated evaluation sets (§217.6): tasks where refinement takes more than two iterations are candidates for synthetic test-case generation.

### Task-Conditional Critique Dimensions

The critique prompt above uses four generic dimensions (correctness, completeness, clarity, efficiency). Madaan et al. demonstrate substantially better performance when the critique dimensions are task-conditional: for code generation, the dimensions are syntactic correctness, test coverage, edge-case handling, and algorithmic efficiency; for constrained text generation (e.g., "write a review with exactly three paragraphs and at least one specific example"), the dimensions are constraint satisfaction, content quality, and specificity. The mapping from task type to critique dimension set is itself a hyperparameter that can be optimised — a connection to [Chapter 218](ch218-self-improvement-fitness-functions.md)'s fitness function design, where the choice of evaluation rubric determines what the model learns to improve.

The deeper theoretical question is why self-critique works at all. The standard argument against self-critique — that a model's failure modes should be symmetric across generation and evaluation, so the critic should miss the same errors as the generator — is empirically false. The asymmetry has two sources. First, the critique prompt explicitly specifies *what to look for*, which is a stronger conditioning signal than the generation prompt's implicit "produce a good output". Second, the model has seen the output before critiquing it; the critique generation is conditioned on the output as context, which gives the model a different distributional position than generation from a prompt alone. These asymmetries are sufficient to make the critic meaningfully stronger than the generator on the relevant dimensions, even when they are the same model. Madaan et al. report 10–40% quality improvements across tasks including code, mathematical reasoning, and constrained generation — comparable to the gains from switching to a significantly larger model without the inference cost increase.

---

## 217.4 Self-Play Fine-Tuning (SPIN)

### Self-as-Adversary

[Chen et al. (2024), "Self-Play Fine-Tuning Converts Weak Language Models to Strong Language Models", arXiv 2401.01335](https://arxiv.org/abs/2401.01335) introduces SPIN (Self-Play INstruction fine-tuning), which improves a model by training it to discriminate between its own previous generations and reference data, using a game-theoretic formulation where the model plays both the generator and the discriminator. The training procedure does not require human feedback — the reference distribution is the supervised fine-tuning dataset, and the model's own previous-iteration outputs are the negative examples. This makes SPIN the self-play analog of RLHF: where RLHF contrasts model outputs against human preference labels, SPIN contrasts model outputs against the reference distribution using the model itself as the reward signal.

### Discriminator Loss

The SPIN objective is derived from the self-play game. Let `π_θ` be the current policy (the model being trained), `π_{θ_t}` be the policy at the previous iteration, and `p_data(y | x)` be the reference distribution (SFT data). The generator is `π_{θ_t}`, fixed; the discriminator objective trains `π_θ` to assign higher likelihood to reference completions than to the generator's completions:

```
L_SPIN(θ; π_{θ_t}) = -E_{(x,y)~p_data} E_{y'~π_{θ_t}(·|x)} [
    log σ(λ · (log π_θ(y|x)  - log π_{θ_t}(y|x)) -
          λ · (log π_θ(y'|x) - log π_{θ_t}(y'|x)))
]
```

where `λ > 0` is a temperature parameter, `y` is the reference completion, and `y'` is the previous-model generation. The inner expression `log π_θ(y|x) - log π_{θ_t}(y|x)` is the log-ratio of the current policy to the reference policy — the DPO-style implicit reward. The discriminator succeeds when the implicit reward for the reference completion exceeds that for the self-generated completion. The `σ` (sigmoid) function converts the margin to a probability, and the expectation is taken jointly over the data distribution and the generative distribution of the previous-iteration model.

```python
import torch
import torch.nn.functional as F

def spin_loss(
    log_probs_theta_y: torch.Tensor,   # (batch,) log π_θ(y|x)
    log_probs_theta_yp: torch.Tensor,  # (batch,) log π_θ(y'|x)
    log_probs_ref_y: torch.Tensor,     # (batch,) log π_{θ_t}(y|x)
    log_probs_ref_yp: torch.Tensor,    # (batch,) log π_{θ_t}(y'|x)
    lam: float = 0.1,
) -> torch.Tensor:
    reward_y  = log_probs_theta_y  - log_probs_ref_y
    reward_yp = log_probs_theta_yp - log_probs_ref_yp
    margin = lam * (reward_y - reward_yp)
    return -F.logsigmoid(margin).mean()
```

### SPIN Training Loop

The SPIN iteration replaces the generator each round with the updated policy, so the self-play game is non-stationary — each round's discriminator trains against a stronger generator:

```
θ_0 = SFT initialisation
for t = 0, 1, 2, ..., T:
    sample (x, y) ~ p_data                        # reference (prompt, completion) pairs
    sample y' ~ π_{θ_t}(· | x)                    # generator rollout from frozen previous model
    compute L_SPIN(θ; π_{θ_t}) over {(x, y, y')}  # discriminator objective
    θ_{t+1} = Adam(θ_t, ∇_θ L_SPIN)               # update current policy
    freeze π_{θ_t} ← π_{θ_{t+1}}                  # advance generator
```

Chen et al. show that this converges to `π_θ ≈ p_data` at the Nash equilibrium: when the policy matches the reference distribution, no self-generated negative example can be distinguished from the reference, so the loss flattens. In practice they run three to four iterations, each a full SFT epoch, and report that Mistral-7B-SFT closes 80–90% of the gap to the corresponding RLHF-trained model on AlpacaEval. The connection to Chapter 214's MAML is structural: SPIN's per-iteration policy update is the outer-loop update of a two-player game; the inner loop is the generator rollout. SPIN is MAML in the self-play sense — the "inner task" is generation, the "outer objective" is discrimination.

---

## 217.5 Inference-Time Activation Analysis

### From Circuit Discovery to Capability Assessment

Chapter 213 uses TransformerLens hooks to discover *circuits* — the specific attention heads and MLP layers that implement a named capability. This is a *static* use of activation analysis: the analysis is run offline on a representative dataset to produce a circuit map that is then interpreted as a structural fact about the model. Inference-time activation analysis applies the same tooling to a *dynamic* purpose: during the forward pass on a specific input, extract activation patterns and infer from them whether the model's capability is engaged and operating normally for that input.

The distinction is analogous to the difference between static analysis and dynamic profiling in a compiler workflow. Static analysis (circuit discovery) characterizes the program's potential behaviour; dynamic profiling (inference-time activation probing) characterizes its actual behaviour on a specific workload. Both are necessary: static analysis provides the probe locations; dynamic profiling provides the runtime signal.

The practical use case is capability-conditional routing: before committing to a multi-step reasoning chain on a difficult input, the model probes its own residual stream to assess whether the relevant reasoning capability is active. A probing classifier trained on labelled activation data returns a capability confidence score; inputs below the confidence threshold are routed to a retrieval augmentation call, a TTT adaptation step (§217.8), or a query decomposition strategy — whichever the structured self-diagnosis (§217.2) identifies as appropriate.

### TransformerLens Activation Probing at Inference Time

```python
import torch
import torch.nn as nn
from transformer_lens import HookedTransformer

model = HookedTransformer.from_pretrained("gpt2-medium")

class CapabilityProbe(nn.Module):
    def __init__(self, d_model: int):
        super().__init__()
        self.linear = nn.Linear(d_model, 1)

    def forward(self, resid: torch.Tensor) -> torch.Tensor:
        return torch.sigmoid(self.linear(resid.mean(dim=1)))

probe = CapabilityProbe(d_model=1024)

def assess_capability_at_inference(
    prompt: str,
    probe: CapabilityProbe,
    probe_layer: int = 18,
    threshold: float = 0.6,
) -> dict:
    tokens = model.to_tokens(prompt)

    _, cache = model.run_with_cache(
        tokens,
        names_filter=lambda name: name == f"blocks.{probe_layer}.hook_resid_post",
    )

    resid = cache[f"blocks.{probe_layer}.hook_resid_post"]

    with torch.no_grad():
        score = probe(resid).item()

    return {
        "capability_score": score,
        "capable": score >= threshold,
        "layer_probed": probe_layer,
    }
```

The `names_filter` kwarg limits cache materialization to the single hook point, avoiding the memory overhead of caching all residual streams — critical for production inference where `run_with_cache` on all layers is prohibitive. The probe is trained offline on a labelled dataset: positive examples are inputs where the model produces correct outputs for the capability under study; negative examples are inputs where it fails. The training objective is binary cross-entropy on `probe(resid_post_layer_L)`, where L is chosen by linear probing sweep (the layer with highest classification accuracy on held-out data).

### Activation Patterns and Weight-Space Geometry

The runtime activation pattern at layer L is a point in ℝ^d_model. Its relationship to weight-space geometry ([Chapter 212 — Weights as a Programming Substrate](ch212-weights-as-programming-substrate.md), §212.2) is direct: the neural collapse result establishes that in a well-trained model, the activation centroids for each class form an Equiangular Tight Frame in the penultimate layer. If the probing layer is close to the penultimate layer, the capability probe is essentially measuring how close the current input's activation is to the ETF centroid for the "capable" class.

This geometric interpretation has a diagnostic implication. When the capability score is low, the activation is not in the basin of the ETF centroid — the input is distributional mismatch. The appropriate remediation depends on the degree of mismatch: small mismatch responds to TTT adaptation (a few gradient steps move the model toward the input's distribution); large mismatch may require retrieval augmentation or task decomposition. The probe score is therefore not just a binary gate but a continuous signal that can be calibrated against the expected TTT gradient step size needed to recover full capability.

---

## 217.6 Self-Generated Evaluation Sets

### Model as Its Own Test Designer

A model that can identify its own failure modes (§217.2) and assess its own capability state (§217.5) has the inputs necessary to design targeted evaluation sets. Self-generated evaluation sets exploit this: the model generates probe questions in areas where it has identified low confidence or systematic failure, answers those questions, assesses the quality of the answers, and accumulates the low-confidence (question, answer) pairs as a synthetic test set that characterizes its capability boundary.

The key property that makes this useful is *coverage*: a model-generated test set is by construction concentrated on the model's own capability boundaries rather than on the average difficulty of the task domain. A human-designed benchmark tests the task; a self-generated test set tests the model's specific failure modes. The two are complementary, not redundant.

### Capability Boundary Discovery via Self-Play

The discovery algorithm uses the model in two roles simultaneously — questioner and answerer — in a structure analogous to the adversarial self-play of SPIN:

```python
import json

PROBE_GENERATION_PROMPT = """\
You are probing your own knowledge boundaries in the domain: {domain}

Generate a question that:
1. Is in this domain
2. You are genuinely uncertain whether you can answer correctly
3. Has a verifiable correct answer (not opinion-based)
4. Is specific enough that "I don't know" is not an acceptable answer

Output JSON: {{"question": "...", "difficulty": "medium|hard|very_hard"}}
"""

CONFIDENCE_ASSESSMENT_PROMPT = """\
Question: {question}
Your answer: {answer}

Assess your confidence in this answer on a scale of 0.0 to 1.0.
Consider:
- Did you recall specific facts or reason from general principles?
- Are there parts of the answer you are not certain about?
- Could you verify this answer if asked to?

Output JSON: {{"confidence": 0.X, "uncertain_aspects": ["...", "..."]}}
"""

def discover_capability_boundary(
    model_fn,
    domain: str,
    n_probes: int = 20,
    confidence_threshold: float = 0.6,
) -> list[dict]:
    weak_areas = []

    for _ in range(n_probes):
        raw = model_fn(PROBE_GENERATION_PROMPT.format(domain=domain))
        try:
            probe = json.loads(raw)
        except json.JSONDecodeError:
            continue

        question = probe["question"]
        answer = model_fn(f"Answer this question: {question}")

        raw_conf = model_fn(CONFIDENCE_ASSESSMENT_PROMPT.format(
            question=question,
            answer=answer,
        ))
        try:
            assessment = json.loads(raw_conf)
        except json.JSONDecodeError:
            continue

        if assessment["confidence"] < confidence_threshold:
            weak_areas.append({
                "domain": domain,
                "question": question,
                "answer": answer,
                "confidence": assessment["confidence"],
                "uncertain_aspects": assessment.get("uncertain_aspects", []),
                "difficulty": probe.get("difficulty", "unknown"),
            })

    return weak_areas
```

The `weak_areas` list is the self-generated evaluation set. Its connection to [Chapter 218 — Self-Improvement Fitness Functions and Capability Assessment](ch218-self-improvement-fitness-functions.md) is direct: these (question, answer, confidence) triples become the fitness signal for the next evolutionary or gradient-based adaptation step. An agent that produces low-confidence answers on a class of inputs has identified that class as a priority for improvement — the self-generated test set is the fitness function's input specification.

The self-play structure also enables calibration: by tracking the model's confidence assessments against the actual correctness of its answers (verified by an oracle where available, or by a separate evaluation round), the system can measure the correlation between stated confidence and actual accuracy. A well-calibrated model should have confidence ≈ accuracy on the self-generated test set; systematic over-confidence in a domain is itself a diagnostic signal for targeted adaptation.

Calibration measurement requires an external verifier for at least a subset of the self-generated questions — the model cannot verify its own answers without circularity. For mathematical and programming domains, automated verifiers are available (symbolic arithmetic, unit test execution). For factual domains, a separate retrieval call can serve as a noisy oracle. The calibration metric used in practice is the Expected Calibration Error (ECE): divide confidence values into B bins, compute accuracy within each bin, and average the absolute difference |accuracy_b - confidence_b| weighted by the fraction of examples in each bin. A model with ECE < 0.05 on its self-generated test set is providing a reliable capability signal; ECE > 0.15 indicates systematic miscalibration that corrupts the downstream adaptation signal. When ECE is high, the capability confidence score from §217.5 (activation-level probe) provides a more reliable signal than the token-level confidence in §217.6, because the probing classifier was trained on externally labelled data rather than the model's own self-assessment.

---

## 217.7 DeepSeek-R1 as Reference Implementation

### Extended Reasoning Traces as Self-Reflection

[DeepSeek-R1 (DeepSeek-AI, 2025), "DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning", arXiv 2501.12948](https://arxiv.org/abs/2501.12948) is the most complete publicly documented implementation of inference-time self-reflection at production scale. The model produces extended `<think>...</think>` token sequences before its final answer — a structured reasoning trace that serves simultaneously as the generation scratchpad and as a self-monitoring log. During the thinking trace, the model explicitly revisits earlier conclusions ("Wait, I made an error above — let me reconsider"), checks intermediate results against known constraints, and reorganises its argument structure before committing to a final answer.

### GRPO Training Mechanism

DeepSeek-R1 is trained using Group Relative Policy Optimization (GRPO), a variant of PPO that eliminates the need for a separate critic model by estimating the baseline from a group of sampled completions. For a prompt x, GRPO samples G completions `{y_1, ..., y_G}` from the current policy `π_θ`, computes a verifiable reward `r_i` for each (for math, the reward is 1 if the final answer is correct and 0 otherwise), and computes the group-relative advantage:

```
A_i = (r_i - mean(r_1, ..., r_G)) / std(r_1, ..., r_G)
```

The policy gradient update maximises the clipped advantage:

```
L_GRPO(θ) = -E[min(π_θ(y_i|x) / π_{θ_old}(y_i|x) · A_i,
                    clip(π_θ(y_i|x) / π_{θ_old}(y_i|x), 1-ε, 1+ε) · A_i)]
```

The verifiable reward — correctness of the final numerical answer — eliminates the need for a separate reward model trained on human preferences. This is the key enabling constraint: GRPO works because math and coding tasks have ground-truth evaluators, allowing the model to train on self-generated rollouts without human labelling. The `<think>...</think>` traces emerge as a side effect of optimising this objective: the model learns that longer, more self-critical thinking traces produce higher-reward final answers on average.

### The Aha Moment and Emergent Self-Reflection

DeepSeek-R1's ablation study (their §3.1 on R1-Zero, trained with GRPO from the base model without any SFT cold-start) reports an emergent phenomenon they call the "aha moment": partway through GRPO training, the model begins allocating thinking tokens to re-examine its earlier reasoning steps, a behaviour that was not explicitly incentivised by the reward function. The model learned to self-reflect because self-reflection is instrumentally useful for producing correct final answers — it reduces the rate of compounding reasoning errors that would otherwise propagate to an incorrect final answer. This is not self-reflection as a trained behaviour but as an emergent strategy: the model discovered it through reinforcement on verifiable outcomes. This positions DeepSeek-R1 as a direct empirical instance of the convergence result in [Chapter 216](ch216-formal-self-improvement-theory.md) §216.3: a sufficiently capable agent with a verifiable utility function will discover self-monitoring as a strategy because self-monitoring increases expected utility.

The connection to SELF-REFINE (§217.3) and SPIN (§217.4) is the training lineage: SELF-REFINE's generate→critique→refine loop is the prompt-level prototype; SPIN's self-play discriminator provides the fine-tuning objective that internalizes that loop; GRPO on verifiable rewards scales the internalized loop to the point where the reasoning trace itself becomes the primary inference mechanism rather than a post-hoc overlay.

### Reasoning Token Scaling and Self-Reflection Depth

A key quantitative finding in DeepSeek-R1 is that the model allocates more thinking tokens to harder problems without being explicitly instructed to do so — an emergent form of adaptive compute allocation. For easy arithmetic problems (single-step), the model uses minimal thinking tokens; for multi-step competition mathematics, it uses thousands of tokens including multiple self-correction cycles. This scaling behaviour is not programmed: it emerges from the GRPO training because problems that require more self-correction to solve correctly benefit from longer thinking traces, so the policy that generates longer traces for harder problems receives higher average reward than the policy that generates fixed-length traces.

The practical implication is that DeepSeek-R1's inference cost scales with problem difficulty in the same way that a compiler's analysis depth scales with code complexity: simple straight-line code requires no iteration; a loop nest with loop-carried dependencies requires many fixpoint passes. The self-reflective inference mechanism is itself a form of dynamic analysis depth control. This adaptive depth property is what separates inference-time self-reflection from static chain-of-thought: chain-of-thought uses a fixed number of reasoning steps; self-reflective inference uses as many steps as the model's internal self-assessment determines are necessary.

The format convention — `<think>...</think>` tags bracketing the reasoning trace before the final answer — serves a second purpose beyond readability: it provides a natural separation between the model's "working memory" (the think section) and its "committed output" (the text after `</think>`). The think section is not evaluated by the reward function on correctness; it is evaluated only indirectly through its effect on the final answer. This separation is structurally equivalent to the distinction between a compiler's internal IR (subject to transformations, never directly visible to the user) and the emitted binary (the committed output that is evaluated for correctness).

---

## 217.8 TTT Integration: Gradient Updates at Inference Time

### Self-Reflection as the Adapter Selector

[Chapter 214 — Gradient-Based Self-Modification](ch214-gradient-based-self-modification.md) §214.2 develops test-time training as a gradient loop executed during the forward pass:

```
θ' = θ - α · ∇_θ L_ss(f_θ(x_test))
```

where `L_ss` is a self-supervised loss on the test input (typically next-token prediction or masked reconstruction). The gradient step adapts the model to the distributional properties of the test input before committing to a prediction. The weakness of basic TTT is that it applies the gradient update uniformly to the test context without knowing which capability is degraded. Self-reflective inference provides the missing signal.

The integrated loop has four phases:

1. **Reflect**: Run the capability probe (§217.5) on the test input. Identify the capability score and the specific aspect of capability that is degraded (from the probe's feature attribution).
2. **Identify**: Run self-diagnosis (§217.2) to name the failure mode (`FACTUAL_GAP`, `DISTRIBUTIONAL_MISMATCH`, etc.).
3. **Adapt**: Execute TTT with a targeted self-supervised loss. For `DISTRIBUTIONAL_MISMATCH`, the loss is next-token prediction on the test context. For `FACTUAL_GAP`, the loss may be on a retrieved context containing the missing fact. For `ARITHMETIC_ERROR`, the loss may be on a chain-of-thought trace with explicit verification steps.
4. **Reflect again**: Re-run the capability probe on `f_{θ'}(x_test)` to confirm that the adaptation improved the capability score. If not, the diagnosis may be incorrect — escalate to a different adaptation strategy.

```python
def reflect_identify_adapt(
    model,
    probe: CapabilityProbe,
    prompt: str,
    ttt_steps: int = 3,
    alpha: float = 1e-4,
) -> dict:
    assessment = assess_capability_at_inference(prompt, probe)

    if assessment["capable"]:
        return {"adapted": False, "assessment": assessment}

    diagnosis = self_debug(
        model_fn=lambda p: model.generate(p),
        question=prompt,
        wrong_answer=model.generate(prompt),
    )

    optimizer = torch.optim.Adam(model.parameters(), lr=alpha)
    tokens = model.to_tokens(prompt)

    for _ in range(ttt_steps):
        optimizer.zero_grad()
        logits = model(tokens)
        loss = torch.nn.functional.cross_entropy(
            logits[0, :-1].reshape(-1, logits.shape[-1]),
            tokens[0, 1:].reshape(-1),
        )
        loss.backward()
        optimizer.step()

    post_assessment = assess_capability_at_inference(prompt, probe)

    return {
        "adapted": True,
        "diagnosis": diagnosis["failure_type"],
        "pre_score": assessment["capability_score"],
        "post_score": post_assessment["capability_score"],
    }
```

The `reflect again` step is essential for diagnostic integrity: a TTT update that does not improve the capability score indicates that the self-diagnosis was incorrect or that the failure mode is not addressable by gradient adaptation on the test context. In such cases, the appropriate escalation is to evolutionary search over prompt strategies ([Chapter 215](ch215-evolutionary-architecture-search.md)) or to a retrieval augmentation call.

Chapter 214's §214.5 notes that the entire reflect→identify→adapt pipeline compiles to a single JAX/XLA program when the model is expressed in JAX: `jax.jit` traces through the capability probe, the diagnosis extraction, and the TTT gradient loop, producing a device-native program with no Python overhead in the hot path. The only non-JIT component is the string parsing of the diagnosis output — which must be done in Python before the TTT loop begins.

### Targeted vs. Uniform Adaptation

The central advantage of self-reflection–directed TTT over basic TTT is *targeting*. Basic TTT applies next-token-prediction gradient updates to all parameters simultaneously, using the test context as the adaptation signal. This is a uniform intervention: every parameter moves slightly toward better fitting the test distribution, regardless of which parameters are actually responsible for the capability degradation identified by the self-diagnosis. For a large transformer with billions of parameters, most of those gradient updates are noise — they update parameters not causally involved in the failure mode.

Self-reflection narrows the intervention to the causally relevant parameters. The failure type from §217.2 maps to a set of candidate causal layers from the circuit analysis infrastructure of [Chapter 213](ch213-mechanistic-interpretability.md): `FACTUAL_GAP` maps to specific MLP layers in the range identified by causal tracing as storing factual associations (the ROME targeting procedure from Ch214 §214.1); `DISTRIBUTIONAL_MISMATCH` maps to the attention layers responsible for in-context retrieval; `LOGICAL_FALLACY` maps to the feed-forward layers in the reasoning-relevant attention heads identified by circuit analysis. The targeted TTT update restricts gradient flow to only these layers using a boolean mask on the optimizer state:

```python
def targeted_ttt_step(
    model,
    tokens: torch.Tensor,
    target_layers: list[int],
    alpha: float = 1e-4,
) -> float:
    for name, param in model.named_parameters():
        layer_idx = _extract_layer_idx(name)
        param.requires_grad = (layer_idx in target_layers)

    optimizer = torch.optim.Adam(
        filter(lambda p: p.requires_grad, model.parameters()),
        lr=alpha,
    )
    optimizer.zero_grad()
    logits = model(tokens)
    loss = torch.nn.functional.cross_entropy(
        logits[0, :-1].reshape(-1, logits.shape[-1]),
        tokens[0, 1:].reshape(-1),
    )
    loss.backward()
    optimizer.step()
    return loss.item()

def _extract_layer_idx(param_name: str) -> int | None:
    import re
    m = re.search(r"\.(\d+)\.", param_name)
    return int(m.group(1)) if m else None
```

The `requires_grad` masking restricts the backward pass computation to the target layers, reducing the TTT gradient step cost from O(L) to O(|target_layers|) where L is the total layer count. For a 96-layer model where the failure mode implicates three to five layers, this is a 20–32× reduction in adaptation cost. The tradeoff is that the targeting depends on the accuracy of the self-diagnosis: if the failure classification is wrong (e.g., `LOGICAL_FALLACY` when the true cause is `FACTUAL_GAP`), the targeted layers will not include the causally relevant parameters and the TTT step will have no effect on the failure mode. The `reflect again` verification step (the capability probe re-evaluation after TTT) detects this situation and can trigger a broader-scope adaptation as a fallback.

---

## 217.9 Cognitive Angle: Self-Reflection as the Bridge

### Runtime Complement to Weight-Level Modification

The techniques in this chapter occupy a specific position in the architecture of self-improvement. Chapters 208–211 cover passive inference: the model receives input and produces output, with no modification to the weight tensor or the inference procedure. Chapters 212–215 cover active modification: the weight tensor is changed by arithmetic operations (Ch212), editing (Ch214), or evolutionary mutation (Ch215). Self-reflective inference (this chapter) is the *bridge* between these regimes — it is active computation (it consumes test-time compute beyond a single forward pass) but it is not necessarily modification (the weight tensor may remain unchanged). It is the computational layer that converts passive inference into informed modification.

This bridge function is what makes self-reflective inference non-optional in a complete self-improvement architecture. A system that can edit its weights ([Chapter 214](ch214-gradient-based-self-modification.md)) but cannot identify which aspects of its behaviour need editing will apply edits blindly — the equivalent of a compiler running all optimisation passes uniformly regardless of hot-path profiling. A system that can assess its own capability state and diagnose its failure modes can target its modifications with the precision of profile-guided optimisation: apply ROME edits to the specific FFN layer implicated in a factual gap; apply TTT adaptation with the specific self-supervised loss that matches the identified failure mode; flag the identified weakness domain as a priority for evolutionary mutation in the Darwin Gödel Machine loop.

### Connection to the Evolutionary Loop (Ch215 / DGM)

In the Darwin Gödel Machine ([Chapter 215](ch215-evolutionary-architecture-search.md) §215.2), each agent in the population is evaluated by running it on a benchmark task and recording its score. The score is the fitness signal that drives population-level selection and mutation. Self-reflective inference inserts a second evaluation layer: before the agent attempts the benchmark task, it assesses its own capability state relative to the task. If the assessment identifies a mismatch between the task requirements and the agent's current capability, the agent can request targeted mutation rather than accepting the fitness penalty of attempting a mismatched task.

This is the operational realisation of the Gödel Machine's proof-theoretic ideal: the Gödel Machine (§216.2) only executes a self-rewrite when it can prove the rewrite is beneficial; the DGM replaces formal proof with empirical fitness evaluation; self-reflective inference adds a capability pre-assessment that narrows the search space for beneficial rewrites before the evolutionary loop even begins. The three layers form a coherent search hierarchy: formal proof (ideal, uncomputable) → empirical fitness (practical, expensive) → capability pre-assessment (cheap, approximate). Each layer filters the candidates that are passed to the next.

### The Full Self-Improvement Stack

The complete architecture connecting Part XXXI's chapters is:

| Layer | Chapter | Mechanism | Timescale |
|---|---|---|---|
| Capability encoding | Ch212 | Weight-space arithmetic | Offline |
| Structural analysis | Ch213 | Circuit discovery, SAE | Offline |
| Surgical editing | Ch214 | ROME, MEMIT, TTT | Milliseconds–seconds |
| Population search | Ch215 | DGM evolutionary loop | Minutes–hours |
| Formal bounding | Ch216 | Gödel Machine, AIXI | Theoretical |
| Runtime self-assessment | **Ch217** | CoT debug, probe, SPIN | Milliseconds |
| Fitness evaluation | Ch218 | Benchmark, self-test | Seconds–minutes |

Self-reflective inference (Ch217) sits at the intersection of the offline and online regimes: it uses artifacts from offline analysis (probe classifiers from Ch213, capability maps from Ch212) to produce runtime signals that drive online adaptation (TTT from Ch214, evolutionary selection from Ch215) and feed the fitness evaluation pipeline (Ch218). It is the component that makes the rest of the stack responsive to the specific distributional properties of the current input, rather than averaging over all possible inputs in a training-time approximation.

---

## Chapter Summary

- **Test-time compute for self-understanding** redirects inference tokens from solving external tasks to reasoning about the model's own capability state — knowledge gaps, failure modes, and distributional mismatch relative to the current input.

- **Chain-of-thought self-debugging** (Wei et al. 2022, arXiv 2201.11903) applies intermediate reasoning steps to trace the causal origin of incorrect outputs. A structured three-phase prompt — recall, inspect, attribute — produces a typed failure classification (`FACTUAL_GAP`, `ARITHMETIC_ERROR`, `DISTRIBUTIONAL_MISMATCH`, etc.) that is directly actionable as a remediation signal.

- **SELF-REFINE** (Madaan et al. 2023, arXiv 2303.17651) iterates the generate→critique→refine loop using the model as its own critic. No external feedback is required. Most quality gains accrue in one to two iterations; beyond three, the critique signal weakens. The refinement history identifies tasks that require more iterations — the primary input to self-generated evaluation sets.

- **SPIN** (Chen et al. 2024, arXiv 2401.01335) fine-tunes the model to discriminate between its own previous-generation outputs and reference data. The discriminator loss `L_SPIN` is structurally equivalent to DPO but without human labels — the reference distribution is the SFT dataset and the self-generated distribution is the negative. SPIN converges when the policy matches the reference distribution (Nash equilibrium), closing 80–90% of the gap to RLHF-trained models on instruction-following benchmarks.

- **Inference-time activation analysis** uses TransformerLens `run_with_cache` at a single diagnostic layer to evaluate a probing classifier trained on capability-labelled data. The probe score is a continuous capability confidence measure, not a binary gate. Its geometric interpretation — proximity to the ETF centroid for the "capable" class — connects runtime probing to the neural collapse result of Chapter 212 §212.2.

- **Self-generated evaluation sets** exploit the model's self-diagnosis capability to generate synthetic test cases concentrated on the model's own capability boundaries. These (question, answer, confidence) triples become the fitness signal for Chapter 218's capability assessment pipeline.

- **DeepSeek-R1** (arXiv 2501.12948) demonstrates that inference-time self-reflection is an emergent strategy under GRPO training with verifiable rewards. The `<think>...</think>` reasoning traces — including explicit self-correction steps — arise because self-reflection instrumentally reduces compounding reasoning errors and thereby increases expected reward. The training lineage runs from SELF-REFINE's prompt-level loop through SPIN's fine-tuning objective to GRPO's RL-scale incentivisation.

- **TTT integration** completes the reflect→identify→adapt→reflect loop: self-reflective inference identifies the failure mode and specifies the appropriate self-supervised loss; TTT executes the gradient update; a second probe assessment confirms the adaptation's effectiveness. The full loop compiles to a single JAX/XLA program for production deployment.

- **The cognitive position** of self-reflective inference is as the bridge between passive inference (Ch208–Ch211) and active weight modification (Ch212–Ch215): it converts the open-loop inference transaction into a closed-loop adaptive system, directing modification resources to exactly the capability aspects that the current input has exposed as deficient.
