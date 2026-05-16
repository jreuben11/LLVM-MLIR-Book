# Chapter 214 — Gradient-Based Self-Modification: Model Editing, Meta-Learning, and Test-Time Adaptation

*Part XXXI — Frontier AI Evolution*

A compiled program cannot modify its own code segment without extraordinary measures — `mprotect`, self-modifying shellcode, or a JIT compiler wired into the process. A trained neural network faces no such barrier: its "code" is its weight tensor, a mutable floating-point array, and the gradient of its loss function with respect to every weight is computable in a single backward pass. Gradient-based self-modification is therefore the default, not the exception. The techniques in this chapter explore the design space of *how* that gradient should be computed, *which* parameters it should touch, and *what* objective should guide it — from surgical rank-one edits that correct a single fact (ROME, MEMIT) to lifetime accumulation protocols that add capabilities without losing prior ones (EWC, GEM). The unifying abstraction is that of a JIT-compiled adaptive runtime: the model's inference behaviour can be changed at any granularity, from a single associative fact to a full optimisation strategy, using the same differentiation machinery that produced the weights in the first place. Understanding that machinery precisely — its scope, its failure modes, and its connection to classical compiler techniques like profile-guided optimisation and tiered recompilation — is the subject of this chapter.

Cross-references: [Chapter 212 — Weights as a Programming Substrate](ch212-weights-as-programming-substrate.md) · [Chapter 213 — Mechanistic Interpretability Infrastructure](ch213-mechanistic-interpretability.md) · [Chapter 215 — Evolutionary Architecture Search](ch215-evolutionary-architecture-search.md) · [Chapter 217 — Self-Reflective Inference and Architecture Introspection](ch217-self-reflective-inference.md) · [Chapter 108 — ORC JIT: On-Request Compilation and ReOptimizeLayer](../part-16-jit-sanitizers/ch108-the-orc-jit.md)

---

## 214.1 Model Editing: Surgical Weight Modification

### The Editing Problem

Pre-training embeds factual associations in the transformer's feed-forward network layers. The association `(Paris, capital-of, France)` maps the subject token representation `k_Paris` to the attribute representation `v_France` somewhere in the MLP weight matrix `W`. When that fact becomes stale — France restructures, a name changes, a fact is retracted — the question is: which weights encode this specific association, and can they be updated without perturbing everything else?

Naïve fine-tuning on a corrected example fails on both counts: it cannot identify which weights to touch (so it touches all of them), and generalised gradient descent on a single example causes catastrophic forgetting across adjacent associations. The model editing literature addresses this with three progressively sophisticated approaches: ROME, MEMIT, and GRACE.

### ROME: Rank-One Model Editing

ROME (Meng et al. 2022, [arXiv 2202.05262](https://arxiv.org/abs/2202.05262)) operationalises the causal tracing result from Chapter 213 (§213.5): factual associations are stored in specific FFN layers at the subject token position. Given the causal trace identifying layer `l*` as the critical site for a fact `(s, r, o)`, ROME computes a rank-one update to the MLP's value projection matrix `W_l*` that implants the new association `(s, r, o*)` while preserving all other rows.

The MLP layer in a transformer computes `FFN(x) = W_out · ReLU(W_in · x)`. The value projection `W_out ∈ ℝ^{d_model × d_ffn}` maps hidden-state directions to residual-stream contributions. Each column of `W_out` is a "memory slot": when the corresponding hidden unit fires, that column's residual-stream contribution is added to the forward pass.

The ROME derivation treats the final-layer MLP as a linear associative memory (`W_out K = V`, where columns of `K` are the keys and columns of `V` are the stored values) and frames the editing problem as: find a rank-one additive matrix `Δ` such that `(W_out + Δ) k* = v*` while `(W_out + Δ) k_i ≈ W_out k_i` for all retained keys `k_i`. The closed-form solution is:

```
Δ = (v* - W_out k*) · k*^T · C^{-1} / (k*^T C^{-1} k*)
```

where `C = K K^T` is the uncentered covariance of keys observed over a representative text corpus, pre-computed once and cached. The denominator `k*^T C^{-1} k*` measures the "surprise" of key `k*` relative to the corpus distribution — it scales the update to be larger for rare keys (which need a stronger update) and smaller for common keys (where a large update would contaminate many related facts).

```python
import torch

def compute_rome_update(
    W_out: torch.Tensor,      # (d_model, d_ffn)
    k_star: torch.Tensor,     # (d_ffn,)  — target key from causal trace
    v_star: torch.Tensor,     # (d_model,) — desired value vector
    C_inv: torch.Tensor,      # (d_ffn, d_ffn) — precomputed covariance inverse
) -> torch.Tensor:
    v_current = W_out @ k_star
    residual = v_star - v_current                        # (d_model,)
    C_inv_k = C_inv @ k_star                             # (d_ffn,)
    scalar = (k_star @ C_inv_k)                          # normalisation
    delta = torch.outer(residual, C_inv_k) / scalar      # (d_model, d_ffn)
    return W_out + delta

def compute_target_key(
    model,
    prompt_tokens: torch.Tensor,
    subject_last_pos: int,
    layer: int,
) -> torch.Tensor:
    hooks, activations = [], {}
    def hook_fn(module, inp, out):
        activations["k"] = inp[0][0, subject_last_pos, :].detach()
    handle = model.transformer.h[layer].mlp.c_fc.register_forward_hook(hook_fn)
    with torch.no_grad():
        model(prompt_tokens)
    handle.remove()
    return activations["k"]

def compute_target_value(
    model,
    target_repr: torch.Tensor,
    W_out: torch.Tensor,
    k_star: torch.Tensor,
    layer: int,
    n_steps: int = 25,
    lr: float = 0.5,
) -> torch.Tensor:
    v = (W_out @ k_star).clone().requires_grad_(True)
    optimizer = torch.optim.Adam([v], lr=lr)
    for _ in range(n_steps):
        optimizer.zero_grad()
        loss = -torch.nn.functional.cosine_similarity(v.unsqueeze(0), target_repr.unsqueeze(0))
        loss.backward()
        optimizer.step()
    return v.detach()
```

The key vector `k*` is recovered by hooking the MLP's input at the subject's last token position (the causal trace result). The value vector `v*` is optimised via gradient descent to maximise the model's probability of producing the new object token `o*` when reading from that key — it is not derived analytically but learned from the target association. The final edit writes `W_out ← W_out + Δ` in place, a permanent weight modification costing O(d_model × d_ffn) flops.

The ROME update satisfies three key properties by construction: *specificity* (only the row corresponding to `k*` in key space is significantly changed), *consistency* (all retained associations are preserved exactly to the extent `C^{-1}` captures them), and *locality* (only a single layer's weight matrix is modified).

### MEMIT: Mass-Editing Across Layers

ROME's single-layer edit saturates when hundreds or thousands of facts are inserted: all updates pile on the same `W_out^{l*}`, and the cumulative rank of the perturbation matrix eventually exceeds what the layer can absorb without distorting adjacent representations. MEMIT (Meng et al. 2022, [arXiv 2210.07229](https://arxiv.org/abs/2210.07229)) distributes the editing load across a range of FFN layers `{l_1, ..., l_L}` by solving a system of linear constraints simultaneously.

The MEMIT formulation is: find matrices `{Δ_l}_{l=l_1}^{l_L}` minimising:

```
Σ_l || Δ_l ||_F^2   subject to:  Σ_l W_l^{out}(Δ_l) k_i* = v_i* - W_0 k_i*  for all edits i
```

where `W_l^{out}(Δ_l) = W_l^{out} + Δ_l` is the edited matrix and `W_0` captures the baseline model's current output for `k_i*`. This is a minimum-norm solution to a set of linear equalities — a standard linear algebra problem with a closed form:

```python
def memit_updates(
    W_outs: list[torch.Tensor],  # [d_model, d_ffn] × L layers
    K_star: torch.Tensor,        # (d_ffn, N) — N target keys stacked
    V_star: torch.Tensor,        # (d_model, N) — N target values stacked
    C_invs: list[torch.Tensor],  # [(d_ffn, d_ffn)] × L precomputed
) -> list[torch.Tensor]:
    L = len(W_outs)
    residual = V_star.clone()
    deltas = []
    for l in range(L):
        R_l = C_invs[l] @ K_star                         # (d_ffn, N)
        norm_l = (K_star * R_l).sum(0, keepdim=True)     # (1, N)
        delta_l = (residual / (L - l)) @ R_l.T / norm_l.mean()
        deltas.append(delta_l)
        residual = residual - (W_outs[l] + delta_l) @ K_star + W_outs[l] @ K_star
    return deltas
```

By spreading rank-one updates across L = 5–10 layers rather than concentrating them at a single critical layer, MEMIT achieves near-linear scaling in edit count — empirically supporting thousands of edits before accuracy degrades. The distributed update is analogous to distributing a hot loop's recompilation across multiple JIT tiers: no single tier is overwhelmed, and each contributes a targeted fraction of the total optimisation.

### GRACE: Retrieval-Augmented Editing Without Weight Modification

GRACE (Hartvigsen et al. 2022, [arXiv 2211.11031](https://arxiv.org/abs/2211.11031)) takes a fundamentally different approach: it never touches the pretrained weights. Instead, GRACE maintains an external codebook `(K, V)` — a key-value store where keys are activation vectors at a specific layer and values are corrected output logits or representations. At inference time, a nearest-neighbor lookup over the codebook detects whether the current input activation falls within radius `ε` of a stored key. If so, the codebook value overrides the model's output; if not, the model proceeds normally.

This is the neural analogue of a dynamic linker symbol table: the base binary (pretrained model) is unchanged; a patch layer (codebook) intercepts specific calls (input activations) and redirects them to patched implementations (corrected outputs).

```python
import torch
import faiss
import numpy as np

class GRACECodebook:
    def __init__(self, d_model: int, epsilon: float = 0.1):
        self.epsilon = epsilon
        self.index = faiss.IndexFlatL2(d_model)
        self.values: list[torch.Tensor] = []

    def add_edit(self, key_activation: torch.Tensor, corrected_value: torch.Tensor):
        k_np = key_activation.float().cpu().numpy().reshape(1, -1)
        self.index.add(k_np)
        self.values.append(corrected_value)

    def lookup(self, query: torch.Tensor) -> torch.Tensor | None:
        q_np = query.float().cpu().numpy().reshape(1, -1)
        distances, indices = self.index.search(q_np, k=1)
        if distances[0, 0] < self.epsilon ** 2:
            return self.values[indices[0, 0]]
        return None
```

GRACE's strength is safety: the pretrained weights are never modified, so there is zero risk of collateral damage to unedited associations. Its weakness is scalability: the codebook lookup adds O(N) inference latency for N edits (mitigated by approximate nearest neighbor indices like FAISS HNSW), and the codebook itself is not a generalisable representation — it stores discrete corrections rather than integrating them into the weight manifold.

### Editing, Weights, and Interpretability: The Three-Chapter Arc

Chapter 212 establishes that weights are a programming substrate — mutable, typed, and structured. Chapter 213 provides the disassembler: causal tracing recovers *which* weights encode *which* capabilities. Chapter 214 provides the assembler: given the disassembly result, rank-one updates rewrite specific weight entries while preserving the surrounding "program". The three chapters form a complete read-modify-write cycle at the weight level, directly analogous to the patch → disassemble → reassemble cycle used in binary instrumentation (DynamoRIO, Frida).

The critical information flow is:

1. **Chapter 212** defines what a weight matrix *is* semantically: `W_{out}` rows are output directions; `W_{out}` columns are per-hidden-unit contributions to the residual stream.
2. **Chapter 213** (§213.5 causal tracing) identifies *which layer's* `W_{out}` is causally responsible for storing fact `(s, r, o)` — specifically, layer `l*` at the subject token's last position.
3. **Chapter 214** (§214.1 ROME) applies the rank-one update to that layer's `W_{out}^{l*}`, changing the column corresponding to key `k*` to produce the corrected value `v*`.

Without step 2, step 3 is a search problem over all L × d_ffn possible target columns — computationally intractable and destructive if applied blindly. With step 2, step 3 is a single O(d_model · d_ffn) matrix operation. Mechanistic interpretability is not an academic exercise; it is the prerequisite that makes surgical weight editing computationally tractable.

The three approaches form a clean hierarchy along the permanence-safety axis:

| Approach | Scope | Weight Modification | Capacity | Reversibility |
|---|---|---|---|---|
| GRACE | Single fact per entry | None | Unlimited (external store) | Trivial (delete entry) |
| ROME | Single fact | One layer | O(rank-1 per edit) | Requires re-editing |
| MEMIT | Batch of facts | L layers | O(thousands of edits) | Requires re-editing |

The connection to Chapter 213 is direct and architectural: ROME and MEMIT require causal tracing to identify the correct target layer `l*`. Without the mechanistic interpretability result that factual associations localise to specific FFN layers at the subject token position, there is no principled basis for choosing which weight matrix to edit. Interpretability precedes editability; §213.5 is the prerequisite for §214.1.

---

## 214.2 Test-Time Training

### The TTT Loop

Standard inference is stateless with respect to model parameters: `θ` is fixed, and `f_θ(x)` computes the output. Test-time training (TTT) breaks this stationarity by executing a gradient update on `θ` during the forward pass, using a self-supervised loss computed from the test input itself before committing to a prediction.

The TTT protocol is:

```
Given test input x, model parameters θ_base:
  1. Construct self-supervised task from x (e.g., masked prediction, rotation prediction, SSM-style)
  2. Compute gradient:  g = ∇_θ L_ss(f_θ(x))
  3. Update:            θ' = θ - α · g
  4. Predict:           ŷ = f_{θ'}(x)
  5. Discard θ' (or retain for next token in streaming TTT)
```

The self-supervised loss `L_ss` measures a property of the input that does not require ground-truth labels: for language models, this is typically next-token prediction on the test context itself; for vision models, it may be masked patch reconstruction or spatial consistency. Gandelsman et al. (2024, [arXiv 2407.04620](https://arxiv.org/abs/2407.04620)) demonstrate that TTT with a single gradient step on the test context's masked language modelling loss improves performance on long-context tasks where the fixed pretrained model systematically fails — the adapted `θ'` has specialised to the distributional properties of the test document.

TTT is the neural equivalent of profile-guided optimisation at inference time: just as PGO re-optimises hot paths after observing actual execution traces, TTT re-optimises model parameters after observing the actual test distribution. The key difference is granularity: PGO operates at the function level; TTT operates at the individual example level.

### JAX Implementation

JAX's functional parameter discipline makes the TTT loop composable with any model:

```python
import jax
import jax.numpy as jnp
import optax
from typing import Any

Params = Any

def self_supervised_loss(params: Params, apply_fn, x_tokens: jnp.ndarray) -> jnp.ndarray:
    logits = apply_fn(params, x_tokens[:-1])
    targets = x_tokens[1:]
    return optax.softmax_cross_entropy_with_integer_labels(logits, targets).mean()

def ttt_step(
    params: Params,
    apply_fn,
    x_tokens: jnp.ndarray,
    alpha: float = 1e-4,
) -> Params:
    grad_fn = jax.grad(self_supervised_loss)
    grads = grad_fn(params, apply_fn, x_tokens)
    return jax.tree_util.tree_map(lambda p, g: p - alpha * g, params, grads)

def ttt_predict(
    params: Params,
    apply_fn,
    x_tokens: jnp.ndarray,
    query_tokens: jnp.ndarray,
    alpha: float = 1e-4,
) -> jnp.ndarray:
    adapted_params = ttt_step(params, apply_fn, x_tokens, alpha)
    return apply_fn(adapted_params, query_tokens)
```

The critical property is that `jax.grad` traces through `apply_fn` without any special instrumentation: the model's forward pass is automatically differentiable with respect to `params` as a pytree. `jax.jit(ttt_predict)` compiles the full TTT loop — self-supervised gradient computation plus inference — into a single XLA program, with no Python overhead in the hot path.

The TTT Hidden State (Sun et al. 2024, [arXiv 2407.04620](https://arxiv.org/abs/2407.04620)) paper introduces the TTT layer as a first-class sequence model layer that learns to perform gradient-based adaptation within its recurrent state: the "hidden state" of a TTT RNN is the set of model weights `θ_t`, updated by a gradient step at each position `t`. This is TTT embedded inside the architecture, not applied post-hoc — the architecture *is* the gradient loop. Cross-reference Chapter 217's self-reflective inference architecture for how this style of inference-time computation interacts with the model's own output distribution.

### Multi-Step TTT and Streaming

Single-step TTT is a conservative approximation. Multi-step TTT iterates the adaptation loop `n` times before committing to a prediction:

```python
def multi_step_ttt(
    params: Params,
    apply_fn,
    x_tokens: jnp.ndarray,
    query_tokens: jnp.ndarray,
    n_steps: int = 5,
    alpha: float = 1e-4,
) -> jnp.ndarray:
    def body(carry, _):
        p = carry
        g = jax.grad(self_supervised_loss)(p, apply_fn, x_tokens)
        return jax.tree_util.tree_map(lambda p_, g_: p_ - alpha * g_, p, g), None
    adapted_params, _ = jax.lax.scan(body, params, None, length=n_steps)
    return apply_fn(adapted_params, query_tokens)
```

`jax.lax.scan` unrolls the TTT loop into a single XLA while_loop, compiling the n-step adaptation into a device-native operation rather than n separate kernel launches.

---

## 214.3 Meta-Learning: Learning to Learn

### MAML: Bi-Level Optimisation

Model-Agnostic Meta-Learning (MAML, Finn et al. 2017, [arXiv 1703.03400](https://arxiv.org/abs/1703.03400)) solves the problem of learning an initialisation point `θ` from which rapid few-shot adaptation is possible. The bi-level objective is:

```
Inner loop (per task τ_i):      θ'_i = θ - α ∇_θ L_{τ_i}(θ)
Outer loop (across tasks):      θ ← θ - β ∇_θ Σ_i L_{τ_i}(θ'_i)
```

The outer gradient `∇_θ L_{τ_i}(θ'_i)` differentiates through the inner gradient step — it is the gradient of a gradient, requiring second-order derivatives (Hessian-vector products) when computed exactly. The connection to TTT is direct: MAML learns to be good at TTT before seeing any test input; TTT performs the MAML inner-loop adaptation at inference time.

JAX's `jax.grad` is composable with itself, making the bi-level computation natural:

```python
import jax
import jax.numpy as jnp
import optax
from functools import partial

def task_loss(params: Params, apply_fn, x_support, y_support) -> jnp.ndarray:
    logits = apply_fn(params, x_support)
    return optax.softmax_cross_entropy_with_integer_labels(logits, y_support).mean()

def inner_update(params: Params, apply_fn, x_support, y_support, alpha: float) -> Params:
    grads = jax.grad(task_loss)(params, apply_fn, x_support, y_support)
    return jax.tree_util.tree_map(lambda p, g: p - alpha * g, params, grads)

def maml_outer_loss(
    params: Params,
    apply_fn,
    tasks: list[tuple],
    alpha: float = 0.01,
) -> jnp.ndarray:
    total = jnp.array(0.0)
    for (x_support, y_support, x_query, y_query) in tasks:
        adapted = inner_update(params, apply_fn, x_support, y_support, alpha)
        total += task_loss(adapted, apply_fn, x_query, y_query)
    return total / len(tasks)

def maml_step(
    params: Params,
    apply_fn,
    tasks: list[tuple],
    optimizer_state,
    optimizer: optax.GradientTransformation,
    alpha: float = 0.01,
) -> tuple[Params, Any]:
    outer_grad = jax.grad(maml_outer_loss)(params, apply_fn, tasks, alpha)
    updates, new_state = optimizer.update(outer_grad, optimizer_state, params)
    new_params = optax.apply_updates(params, updates)
    return new_params, new_state
```

`jax.grad(maml_outer_loss)` differentiates through `inner_update`, which itself calls `jax.grad`. JAX traces through the inner `jax.grad` call automatically because `jax.grad` is just a Python function that returns a new JAX-traceable function — there is no special "second-order" API. The Hessian-vector product is computed via forward-over-reverse mode differentiation, which JAX handles without user intervention.

First-order MAML (FOMAML) drops the second-order terms by treating the inner-adapted parameters as detached constants:

```python
def fomaml_outer_loss(params, apply_fn, tasks, alpha=0.01):
    total = jnp.array(0.0)
    for (x_support, y_support, x_query, y_query) in tasks:
        adapted = jax.lax.stop_gradient(
            inner_update(params, apply_fn, x_support, y_support, alpha)
        )
        total += task_loss(adapted, apply_fn, x_query, y_query)
    return total / len(tasks)
```

`jax.lax.stop_gradient` wraps the adapted parameters, blocking gradient flow through the inner update. FOMAML sacrifices the second-order correction but eliminates the Hessian computation, making it 2–3× faster per step with modest accuracy loss on most benchmarks.

### iMAML: Implicit Differentiation

The second-order computation in MAML grows expensive for deep networks: the full Hessian of `L_task` with respect to `θ` is O(|θ|^2) to store. iMAML (Rajeswaran et al. 2019, [arXiv 1909.04630](https://arxiv.org/abs/1909.04630)) avoids this by noting that when the inner optimisation is run to convergence — producing `θ' = argmin_θ L_task(θ)` — the gradient of the outer loss with respect to `θ` can be computed via implicit differentiation:

```
∂θ'/∂θ = (I + λ H_θ)^{-1}
```

where `H_θ = ∇²_θ L_task(θ')` is the Hessian at the converged solution and `λ` is the inner regularisation coefficient. Rather than computing and inverting the full Hessian, iMAML uses the conjugate gradient (CG) method to compute `(I + λ H_θ)^{-1} v` for the required vector `v` (the outer gradient), requiring only Hessian-vector products (HVPs):

```python
def hvp(f, params, v):
    return jax.jvp(jax.grad(f), [params], [v])[1]

def conjugate_gradient(hvp_fn, b, n_steps=10, tol=1e-6):
    x = jax.tree_util.tree_map(jnp.zeros_like, b)
    r = b
    p = r
    r_dot_r = sum(jnp.vdot(ri, ri) for ri in jax.tree_util.tree_leaves(r))
    for _ in range(n_steps):
        Ap = hvp_fn(p)
        alpha = r_dot_r / sum(jnp.vdot(pi, Api) for pi, Api in zip(
            jax.tree_util.tree_leaves(p), jax.tree_util.tree_leaves(Ap)))
        x = jax.tree_util.tree_map(lambda xi, pi: xi + alpha * pi, x, p)
        r = jax.tree_util.tree_map(lambda ri, Api: ri - alpha * Api, r, Ap)
        r_dot_r_new = sum(jnp.vdot(ri, ri) for ri in jax.tree_util.tree_leaves(r))
        if r_dot_r_new < tol:
            break
        beta = r_dot_r_new / r_dot_r
        p = jax.tree_util.tree_map(lambda ri, pi: ri + beta * pi, r, p)
        r_dot_r = r_dot_r_new
    return x
```

The `jax.jvp` (forward-mode) applied to `jax.grad` (reverse-mode) computes HVPs in O(|θ|) time and O(1) memory overhead relative to a single gradient pass — the critical property making iMAML practical for large models.

### Hypernetworks: Weight Generation

A hypernetwork (Ha et al. 2016, [arXiv 1609.09106](https://arxiv.org/abs/1609.09106)) is a network `h_φ(z) → W` that generates the weights of a target network `f_W(x)` from a task embedding `z`. The hypernetwork `h` is trained end-to-end alongside the target architecture; the weights `W = h_φ(z)` are produced by a forward pass through `h` rather than being stored as learnable parameters.

Haiku's functional transform API makes hypernetworks natural in JAX:

```python
import haiku as hk
import jax
import jax.numpy as jnp

def target_network(x: jnp.ndarray) -> jnp.ndarray:
    return hk.Linear(10)(x)

def hypernetwork(z: jnp.ndarray, target_fn) -> jnp.ndarray:
    target_params = hk.lift(hk.transform(target_fn).init)(
        hk.next_rng_key(), jnp.zeros((1, 32))
    )
    generated = {}
    for name, param_dict in target_params.items():
        generated[name] = {}
        for pname, param in param_dict.items():
            generated[name][pname] = hk.Linear(
                int(jnp.prod(jnp.array(param.shape))), name=f"gen_{name}_{pname}"
            )(z).reshape(param.shape)
    return generated

transformed_hyper = hk.transform(lambda z: hypernetwork(z, target_network))
```

In practice, hypernetworks for large targets generate low-rank components or LoRA adapters rather than full weight matrices — generating a 70B-parameter weight matrix from a single embedding vector is not parameter-efficient. Modern hypernetwork applications include: per-document LoRA adapter generation from a document embedding (personalisation without per-user fine-tuning), architecture search via differentiable weight generators (the weight generator itself is the search variable), and continual learning via task-conditioned weight generation (different `z` for each task suppresses interference).

---

## 214.4 Gradient Surgery for Multi-Objective Learning

### The Conflicting Gradient Problem

Training a single model on multiple tasks exposes a fundamental gradient interference problem: for two tasks `A` and `B`, if `∇_θ L_A` and `∇_θ L_B` point in opposite directions along some component of the weight space, a naive gradient sum pushes `θ` in a direction that helps neither task. This occurs when tasks share features but differ in the function mapping those features to outputs — the shared representation must be pulled in incompatible directions simultaneously.

PCGrad (Yu et al. 2020, [arXiv 2001.06782](https://arxiv.org/abs/2001.06782)) addresses this by projecting each task's gradient onto the normal plane of the other task's gradient when they conflict (i.e., when their inner product is negative):

```
If ⟨g_A, g_B⟩ < 0:   g_A^{proj} = g_A - (⟨g_A, g_B⟩ / ||g_B||²) · g_B
Else:                  g_A^{proj} = g_A
```

```python
import jax
import jax.numpy as jnp
from typing import Sequence

def pcgrad(grads_list: Sequence[jnp.ndarray]) -> jnp.ndarray:
    n = len(grads_list)
    flat = [g.ravel() for g in grads_list]
    projected = []
    for i in range(n):
        g_i = flat[i]
        for j in range(n):
            if i == j:
                continue
            g_j = flat[j]
            dot = jnp.dot(g_i, g_j)
            g_i = jnp.where(dot < 0, g_i - (dot / (jnp.dot(g_j, g_j) + 1e-12)) * g_j, g_i)
        projected.append(g_i)
    return sum(projected).reshape(grads_list[0].shape)

def pcgrad_pytree(grads_list: list[dict]) -> dict:
    leaves_list = [jax.tree_util.tree_leaves(g) for g in grads_list]
    n_leaves = len(leaves_list[0])
    proj_leaves = [
        pcgrad([leaves[i] for leaves in leaves_list])
        for i in range(n_leaves)
    ]
    return jax.tree_util.tree_unflatten(
        jax.tree_util.tree_structure(grads_list[0]), proj_leaves
    )
```

### CAGrad: Constrained Average Gradient

CAGrad (Liu et al. 2021, [arXiv 2110.14735](https://arxiv.org/abs/2110.14735)) frames the gradient selection as a constrained optimisation problem: find the update direction `d` that minimises the maximum task loss improvement, subject to `||d - ĝ|| ≤ c · ||ĝ||`, where `ĝ` is the average gradient and `c` is a constraint radius. Unlike PCGrad's heuristic projection, CAGrad provides a theoretically motivated update that improves the worst-performing task subject to staying close to the average direction:

```python
def cagrad(grads_list: list[jnp.ndarray], c: float = 0.5) -> jnp.ndarray:
    flat = jnp.stack([g.ravel() for g in grads_list])        # (n_tasks, d)
    g_avg = flat.mean(0)                                       # (d,)
    GGT = flat @ flat.T                                        # (n_tasks, n_tasks)
    n = len(grads_list)
    phi = jnp.ones(n) / n
    g_0_norm = jnp.linalg.norm(g_avg)
    for _ in range(20):
        g_phi = phi @ flat                                     # (d,)
        dot = flat @ g_phi                                     # (n_tasks,)
        obj_grad = dot - c * g_0_norm / jnp.sqrt((phi @ GGT @ phi) + 1e-12) * (GGT @ phi)
        phi = phi - 0.01 * obj_grad
        phi = jnp.maximum(phi, 0.0)
        phi = phi / phi.sum()
    return (phi @ flat).reshape(grads_list[0].shape)
```

Gradient surgery techniques are particularly relevant when extending a pretrained model with new capabilities (the new task's gradient interferes with the capability-preserving gradients for existing tasks) — precisely the setup of continual learning in §214.6.

---

## 214.5 In-Context Learning as Implicit Gradient Descent

### The Linear Attention Duality

An attention head with softmax replaced by the identity function (linear attention) computes:

```
Attn(Q, K, V) = (Q K^T) V / n  =  Q (K^T V / n)
```

where `K^T V / n ∈ ℝ^{d_k × d_v}` is the associative memory matrix — a sum of outer products of key-value pairs from the context. Dai et al. (2022, [arXiv 2212.10559](https://arxiv.org/abs/2212.10559)) show that under linear attention, a transformer's in-context learning of a linear regression task is *functionally equivalent* to one step of gradient descent on a squared-error objective, with the context examples serving as the training data. Specifically, if the context consists of pairs `{(x_i, y_i)}_{i=1}^N` and the query is `x_q`, the linear attention output computes:

```
W_GD = W_0 - α/N · Σ_i (W_0 x_i - y_i) x_i^T
output = W_GD x_q
```

This is exactly gradient descent on `L = Σ_i ||W x_i - y_i||²` starting from `W_0 = W_V W_Q^T`. Each transformer layer implements one gradient descent step; a transformer with L layers implements L steps. The context window is the training set; the attention weights encode the gradient-descent update rule.

**Critical scope qualification:** this equivalence holds under *linear* attention, not the standard softmax attention used in production models. The follow-up paper (Ahn et al. 2023, [arXiv 2311.07772](https://arxiv.org/abs/2311.07772)) revisits and qualifies the duality: under softmax attention and with multiple heads, transformers implement a variant of preconditioned gradient descent that adapts the step size and direction per-query — closer to Newton's method than vanilla GD. The duality is approximate and input-dependent, not exact. Expert readers should not interpret the original Dai et al. result as applying directly to GPT-4 or Llama-class models; it is a mechanistic hypothesis about the *type* of algorithm that self-attention implements, not a proved identity for production architectures.

### Formal Derivation Under Linear Attention

The derivation proceeds in three steps. First, represent an in-context training set as `D = {(x_i, y_i)}_{i=1}^N` embedded in the transformer's key and value sequences: keys are formed from query token embeddings `K = [x_1, ..., x_N, x_q] W_K` and values from label representations `V = [y_1, ..., y_N, 0] W_V`. Under linear attention, the output for query `x_q` is:

```
output = W_Q x_q · (K^T V / N)
       = (W_Q x_q) · (W_K^T [x_1..x_N]^T [y_1..y_N] W_V / N)
```

Second, define `W_0 = W_Q W_K^T / N` (the initialisation) and note that the gradient of squared loss on the in-context data with respect to `W_0` is:

```
∇_{W_0} Σ_i ||W_0 x_i - y_i||² = Σ_i 2(W_0 x_i - y_i) x_i^T
```

One gradient descent step from `W_0` with step size `α = 1/2` gives:

```
W_1 = W_0 - (1/2N) Σ_i (W_0 x_i - y_i) x_i^T
    = W_0 - (1/2N) [W_0 K^T K - V^T K]
```

Third, substituting back, `W_1 x_q = W_0 x_q - (W_0 K^T K - V^T K) x_q / (2N)`, which matches the linear attention output up to scale. Each layer implements one GD step with the weight matrix `W_0` as initialisation; layer depth corresponds directly to gradient descent depth.

### Implications for Model Understanding

The ICL-as-GD perspective yields three concrete engineering insights. First, the *number of in-context examples* is analogous to the *number of gradient steps* — adding more examples should monotonically improve performance up to the capacity of the available context window, exactly as more gradient steps improve convergence. Second, the *ordering* of in-context examples corresponds to the *order of SGD mini-batches* — recency effects in ICL (last examples weighted more heavily) correspond to the momentum and step-size sensitivity of SGD. Third, the *quality of in-context examples* determines the gradient signal — noisy or mislabelled examples in the context produce poor gradient estimates, degrading ICL performance in the same way that noisy mini-batches degrade ordinary fine-tuning.

The correspondence also reframes the role of the pretrained initialisation: `W_0 = W_Q W_K^T` is the starting point for in-context gradient descent. A model pretrained on code and mathematics starts its in-context GD from a different `W_0` than one pretrained only on web text — the pretraining objective shapes the gradient landscape that ICL navigates. This connects model editing (§214.1) to ICL: editing `W_V` or `W_Q` changes the effective initialisation `W_0` of the in-context GD algorithm for all subsequent queries.

### Weight-Stationary, Functionally Gradient-Descent-Equivalent

The ICL-as-GD perspective resolves what appears to be a paradox: ICL "learns" from examples without updating weights (weight-stationary), yet adapts its output as if it had been fine-tuned. The resolution is that the weight-stationarity is *parametric* (the stored weights θ do not change) but the *functional computation* is not stationary — each additional in-context example changes the output of the forward pass in the same direction as an explicit gradient step would. The model is not frozen; it is running a gradient computation in activation space rather than in weight space.

This has a direct implication for the model editing tools of §214.1: ROME and MEMIT change the weights to permanently encode a new fact; ICL encodes the same fact transiently in the context window. The difference is persistence: weight edits survive across sessions; context edits do not. A production system may choose between the two based on expected longevity of the fact — ephemeral corrections belong in context; permanent corrections belong in the weights.

---

## 214.6 Continual Learning: Capability Accumulation

### The Catastrophic Forgetting Problem

A model trained sequentially on tasks `T_1, T_2, ..., T_N` with standard gradient descent typically forgets `T_1` by the time `T_N` is learned. This is catastrophic forgetting: gradients for `T_N` overwrite the weight configuration that solved `T_1`. Unlike human learners, who interleave retrieval and consolidation, unconstrained SGD has no mechanism to protect weight configurations that were useful for past tasks.

Continual learning (CL) is the discipline of training strategies that accumulate capabilities without regression. The landscape is surveyed comprehensively in [De Lange et al. (2024), arXiv 2403.05175](https://arxiv.org/abs/2403.05175); the LLM-specific challenges are surveyed in [Shi et al. (2024), arXiv 2404.16789](https://arxiv.org/abs/2404.16789). Three foundational approaches are examined here: EWC, GEM, and A-GEM.

### EWC: Elastic Weight Consolidation

EWC (Kirkpatrick et al. 2017, [arXiv 1612.00796](https://arxiv.org/abs/1612.00796)) adds a quadratic regularisation term that penalises movement of weights that were important for previous tasks. Importance is measured by the diagonal of the Fisher information matrix `F` — a curvature estimate that is large for weights that are sensitive to the training distribution of past tasks:

```
L_EWC(θ) = L_new(θ) + λ/2 · Σ_i F_i · (θ_i - θ_i*)²
```

where `θ*` is the parameter vector after completing the previous task and `F_i` is the `i`-th diagonal Fisher estimate. The Fisher diagonal is approximated by squared gradients of the log-likelihood over the previous task's data:

```python
import jax
import jax.numpy as jnp
import optax

def compute_fisher_diagonal(
    params: Params,
    apply_fn,
    data_loader,
    n_samples: int = 1000,
) -> Params:
    grad_fn = jax.grad(lambda p, x, y: -apply_fn(p, x)[jnp.arange(y.shape[0]), y].mean())
    fisher_accum = jax.tree_util.tree_map(jnp.zeros_like, params)
    count = 0
    for x_batch, y_batch in data_loader:
        if count >= n_samples:
            break
        grads = grad_fn(params, x_batch, y_batch)
        fisher_accum = jax.tree_util.tree_map(
            lambda f, g: f + g ** 2, fisher_accum, grads
        )
        count += x_batch.shape[0]
    return jax.tree_util.tree_map(lambda f: f / count, fisher_accum)

def ewc_loss(
    params: Params,
    apply_fn,
    x: jnp.ndarray,
    y: jnp.ndarray,
    fisher: Params,
    params_star: Params,
    lambda_ewc: float = 1000.0,
) -> jnp.ndarray:
    task_loss = optax.softmax_cross_entropy_with_integer_labels(
        apply_fn(params, x), y
    ).mean()
    ewc_penalty = jax.tree_util.tree_map(
        lambda f, p, p_star: f * (p - p_star) ** 2,
        fisher, params, params_star
    )
    penalty = sum(jnp.sum(v) for v in jax.tree_util.tree_leaves(ewc_penalty))
    return task_loss + (lambda_ewc / 2) * penalty
```

The Fisher diagonal is the neural equivalent of the compiler's liveness information for register allocation: it identifies which "registers" (weights) are "live" (important) at the boundary of a task, signalling that they must not be spilled (overwritten) when allocating space for the next task.

EWC's computational cost is O(|θ|) per Fisher diagonal (one additional gradient pass per batch), and the regularisation overhead is also O(|θ|) per step — negligible compared to the forward-backward pass. Its weakness is that the Fisher diagonal approximation ignores off-diagonal correlations (the full Fisher is O(|θ|²) to store), and the approximation degrades when the tasks are very different (high-curvature directions in task N's loss landscape may not align with task N-1's Fisher estimate).

### GEM and A-GEM: Gradient Episodic Memory

GEM (Lopez-Paz & Ranzato 2017, [arXiv 1706.08840](https://arxiv.org/abs/1706.08840)) maintains an episodic memory `M_k` — a small buffer of examples from each past task `k`. At each gradient step for the current task, GEM solves a constrained optimisation: find the update direction `d` closest to the unconstrained gradient `g` that does not increase the loss on any past task memory buffer:

```
minimise   ||d - g||²
subject to: ⟨d, g_k⟩ ≥ 0   for all past tasks k
```

where `g_k = ∇_θ L(θ, M_k)` is the gradient evaluated on task k's memory buffer. The constraint requires that the update does not conflict with any past task gradient — gradient surgery applied to the continual learning setting.

A-GEM (Chaudhry et al. 2019, [arXiv 1812.00420](https://arxiv.org/abs/1812.00420)) approximates the per-task constraints with a single average constraint, dramatically reducing computational cost:

```python
def agem_project(
    grad_current: jnp.ndarray,   # gradient for current task (flat)
    grad_memory: jnp.ndarray,    # average gradient over episodic memory (flat)
) -> jnp.ndarray:
    dot = jnp.dot(grad_current, grad_memory)
    mem_norm_sq = jnp.dot(grad_memory, grad_memory) + 1e-12
    projected = grad_current - jnp.where(
        dot < 0,
        (dot / mem_norm_sq) * grad_memory,
        jnp.zeros_like(grad_current),
    )
    return projected

def agem_step(
    params: Params,
    apply_fn,
    x_current: jnp.ndarray,
    y_current: jnp.ndarray,
    memory_buffer: list[tuple],
    optimizer_state,
    optimizer: optax.GradientTransformation,
) -> tuple[Params, Any]:
    x_mem = jnp.concatenate([m[0] for m in memory_buffer])
    y_mem = jnp.concatenate([m[1] for m in memory_buffer])
    loss_fn = lambda p, x, y: optax.softmax_cross_entropy_with_integer_labels(
        apply_fn(p, x), y
    ).mean()
    grad_current = jax.grad(loss_fn)(params, x_current, y_current)
    grad_memory = jax.grad(loss_fn)(params, x_mem, y_mem)
    flat_curr, unravel = jax.flatten_util.ravel_pytree(grad_current)
    flat_mem, _ = jax.flatten_util.ravel_pytree(grad_memory)
    flat_proj = agem_project(flat_curr, flat_mem)
    proj_grad = unravel(flat_proj)
    updates, new_state = optimizer.update(proj_grad, optimizer_state, params)
    return optax.apply_updates(params, updates), new_state
```

### Connection to ORC JIT ReOptimizeLayer

Chapter 108's ReOptimizeLayer implements tiered recompilation: a JIT-compiled function starts in a baseline tier (fast compilation, low-quality code) and is re-optimised at higher tiers as invocation counts accumulate. The tier transition preserves semantic correctness (the same input produces the same output) while improving efficiency. This is structurally identical to the continual learning contract: new information (invocation counts / new task data) triggers re-optimisation that improves the current capability while not regressing the baseline guarantee (semantic correctness / past task performance).

The analogy is precise at the mechanism level:

| ORC ReOptimizeLayer | Continual Learning |
|---|---|
| Invocation counter threshold | Task boundary signal |
| Baseline JIT tier (no optimisation) | Pre-trained initialisation |
| Optimised recompilation | EWC/GEM gradient-constrained update |
| Semantic correctness guarantee | Task k performance constraint `⟨d, g_k⟩ ≥ 0` |
| Code cache versioning | Checkpoint versioning (Orbax) |
| Hot-code detection (PGO) | Fisher diagonal importance estimation |

Both systems face the same fundamental trade-off: more aggressive optimisation at each tier transition risks violating the invariant that prior behaviour is preserved. ORC manages this with correctness verification; EWC manages it with the Fisher quadratic bound; GEM manages it with explicit gradient constraints. The conceptual unity is not coincidental — tiered recompilation is a compiler-domain solution to the same problem as continual learning: how to improve a running system without breaking what it currently does.

---

## 214.7 The Cognitive Angle: A Spectrum of Self-Modification

Gradient-based self-modification spans four levels of temporal and representational scope. Together they form a hierarchy from the most surgical to the most comprehensive:

**Fact-level (model editing):** A single associative fact is corrected — `(Paris, capital-of, France)` → `(Paris, capital-of, NewRepublic)`. The intervention is sub-millisecond (one rank-one update), touches O(d_ffn) parameters in a single layer, and requires causal tracing to locate the fact's encoding. ROME and MEMIT operate here. The corresponding compiler analogy is hot-patching a binary: replacing a single instruction or constant without recompiling.

**Example-level (TTT):** The model adapts to the distributional properties of a single test document or prompt. The intervention takes one to tens of gradient steps, touches all parameters but with a small step size, and uses the test input itself as the training signal. TTT operates here. The compiler analogy is profile-guided code motion within a function: reordering or specialising based on a single observed execution.

**Distribution-level (MAML/meta-learning):** The model learns an initialisation from which rapid adaptation to any task in a distribution is possible. Training time is long (full meta-training loop), but adaptation at test time is fast (one inner loop). Hypernetworks extend this by parameterising the task-specific weights as a function of task embeddings. The compiler analogy is speculative optimisation or PGO at the library level: the library is compiled with profile feedback from expected usage distributions, so any specific caller pattern adapts quickly.

**Lifetime-level (continual learning):** The model accumulates capabilities across its operational lifetime without forgetting. EWC, GEM, and A-GEM operate here. The intervention is continuous — every gradient step respects the continual learning constraint. The compiler analogy is Chapter 108's ReOptimizeLayer: indefinite tiered recompilation that improves performance monotonically without semantic regression.

The spectrum also characterises the relationship between gradient-based self-modification and evolutionary architecture search (Chapter 215). Gradient methods are surgical: they have a cost function, a gradient, and a well-defined optimisation geometry. They are effective when the modification target is differentiably reachable from the current weights — when the change is smooth in parameter space. Evolutionary methods are exploratory: they sample discontinuous architectural changes (adding layers, changing activation functions, altering connectivity) that are not reachable by gradient descent without architectural surgery. The two paradigms are complementary: gradient methods refine what exists; evolutionary methods discover what is structurally possible.

### The Self-Improvement Loop Topology

The techniques of this chapter do not operate in isolation — they interlock into a self-improvement loop that spans Chapters 212–218. The full topology is:

1. **Capability auditing (Ch213):** SAE feature analysis and causal tracing identify which capabilities exist, where they are localised, and which weights encode them.
2. **Surgical correction (§214.1):** ROME/MEMIT apply rank-one weight updates to correct specific failures identified by causal tracing.
3. **Test-time specialisation (§214.2):** TTT adapts the corrected model to the test distribution in real time, consuming only inference-time computation.
4. **Meta-initialisation (§214.3):** MAML meta-training shapes the model's initialisation to make future TTT steps maximally effective.
5. **Multi-objective gradient management (§214.4):** Gradient surgery maintains balance between the model's existing capabilities and new task objectives during meta-training.
6. **Implicit in-context learning (§214.5):** The forward pass itself implements gradient descent in activation space, enabling rapid in-context adaptation without any parameter update.
7. **Lifetime accumulation (§214.6):** EWC/GEM constraints prevent catastrophic forgetting as capabilities accumulate across the model's operational lifetime.

Each stage feeds the next. Causal tracing specifies the ROME target; the ROME update improves the meta-initialisation quality for MAML; the improved initialisation makes TTT more efficient; the TTT gradient is managed by PCGrad when combined with other adaptation signals; the entire loop operates within EWC/GEM constraints that guarantee backward transfer is non-negative. The result is a gradient-based self-improvement architecture that operates simultaneously at four time scales: microseconds (causal tracing), milliseconds (ROME update), seconds (TTT), and epochs (continual learning).

Chapter 217's self-reflective inference architecture closes the loop: by reading concept directions from its own residual stream (the linear representation hypothesis, §213.8), the model can *self-diagnose* which capabilities need TTT adaptation on the current input before committing to a prediction, triggering the TTT gradient loop only when the introspective signal indicates distributional mismatch. Gradient-based self-modification is thereby self-directed: the model uses its own internals as the oracle for when and how to adapt.

---

## Chapter Summary

- **Model editing operationalises mechanistic interpretability**: ROME (arXiv 2202.05262) uses the causal tracing result from Chapter 213 — that factual associations localise to specific FFN layers — to compute a closed-form rank-one update `W_out ← W_out + (v* - W k*)k*^T C^{-1} / (k*^T C^{-1} k*)`. Interpretability is the prerequisite; editing is the consequence.

- **MEMIT** (arXiv 2210.07229) distributes edits across L FFN layers to avoid single-layer saturation, enabling mass editing of thousands of facts while maintaining specificity across the unedited associative memory.

- **GRACE** (arXiv 2211.11031) avoids weight modification entirely by maintaining an external codebook with nearest-neighbor lookup, trading inference latency for zero collateral damage to pretrained weights — the neural analogue of a dynamic patch table.

- **Test-time training** executes a gradient loop inside the forward pass: `θ' = θ - α∇L_ss(f_θ(x))` adapts parameters to the test-input distribution before committing to a prediction. JAX's functional transform makes the TTT loop composable with any model via `jax.grad(self_supervised_loss)`.

- **MAML** (arXiv 1703.03400) learns an initialisation from which rapid few-shot adaptation is possible via bi-level optimisation: the outer loop differentiates through the inner gradient step using `jax.grad(jax.grad(...))`, which JAX handles without a special second-order API.

- **iMAML** (arXiv 1909.04630) replaces the explicit second-order computation with conjugate gradient inversion of the implicit differentiation identity `∂θ'/∂θ = (I + λH)^{-1}`, using HVPs computed via `jax.jvp(jax.grad(...))` in O(|θ|) time.

- **Hypernetworks** (arXiv 1609.09106) generate target network weights from task embeddings `z → W`, enabling per-task specialisation without storing per-task weight copies. Modern applications generate LoRA adapters rather than full weight matrices.

- **Gradient surgery** (PCGrad, arXiv 2001.06782; CAGrad, arXiv 2110.14735) prevents multi-task gradient interference by projecting conflicting task gradients onto the normal plane of each other, or by solving a constrained optimisation that improves the worst-performing task without straying far from the average gradient.

- **In-context learning as gradient descent** (Dai et al. 2022, arXiv 2212.10559) shows that under *linear* attention, a transformer forward pass implements gradient descent on the in-context training examples — each layer is one GD step. The ICL-GD revisit (arXiv 2311.07772) qualifies this to approximate preconditioned GD under softmax attention.

- **Continual learning** — EWC (arXiv 1612.00796), GEM (arXiv 1706.08840), A-GEM (arXiv 1812.00420) — accumulates capabilities without regression using Fisher-diagonal regularisation, episodic memory gradient constraints, or approximate average-gradient projection. The ReOptimizeLayer in Chapter 108 is the compiler-domain isomorph: tiered recompilation that improves performance monotonically while guaranteeing semantic correctness preservation.

- **The self-modification spectrum** runs from fact-level (ROME, sub-millisecond, one layer) through example-level (TTT, seconds, all layers) through distribution-level (MAML, hours of meta-training, global initialisation) through lifetime-level (continual learning, indefinite, all tasks). Each level is a qualitatively different relationship between gradient computation and the model's operational time scale.

- **Gradient-based and evolutionary methods are complementary**: gradient surgery (§214.4) refines behaviour within the differentiable manifold of the current architecture; evolutionary architecture search (Chapter 215) explores discontinuous structural changes inaccessible to gradient descent. A complete self-improvement system requires both: gradient methods for precision within a fixed topology, evolutionary methods for escaping local optima that require architectural change.

- **The full self-improvement loop** (Chapters 212–218) interleaves: mechanistic interpretability (capability auditing) → model editing (surgical correction) → TTT (test-time specialisation) → MAML (meta-initialisation) → gradient surgery (multi-objective balance) → ICL (in-context implicit adaptation) → continual learning (lifetime accumulation). The Chapter 217 self-reflective inference architecture provides the introspective trigger: the model reads its own residual stream to decide *when* TTT adaptation is warranted, closing the loop between self-knowledge and self-modification.
