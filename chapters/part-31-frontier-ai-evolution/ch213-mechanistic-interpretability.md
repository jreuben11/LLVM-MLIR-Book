# Chapter 213 — Mechanistic Interpretability Infrastructure

*Part XXXI — Frontier AI Evolution*

Mechanistic interpretability is to neural networks what decompilation is to binary executables. A trained model is a dense array of floating-point weights — machine code. Somewhere inside those weights, a representation scheme encoded features analogous to assembly instructions, circuits analogous to named functions, and algorithms analogous to high-level program logic. The interpretability researcher's task is to recover that high-level structure from the low-level artifact. This is not a metaphor for motivational convenience: the information-theoretic relationship between weights and capabilities is structurally identical to the relationship between machine code and source programs. Weights were never meant to be read by humans; neither were `.text` sections. The discipline that emerged to reverse-engineer neural network internals — using sparse autoencoders as decompilers, causal tracing as debuggers, and attribution graphs as architecture diagrams — has produced a coherent engineering methodology that this chapter develops in full.

Cross-references: [Chapter 212 — Weights as Programming Substrate](ch212-weights-as-programming-substrate.md) · [Chapter 214 — Gradient-Based Self-Modification](ch214-gradient-based-self-modification.md) · [Chapter 217 — Self-Reflective Inference](ch217-self-reflective-inference.md) · [Chapter 218 — Self-Improvement Fitness Functions](ch218-self-improvement-fitness-functions.md) · [Chapter 207 — Reflective Code, Open Problems, and Build Roadmap](../part-30-AI-first-PL-design/ch207-reflective-code-open-problems-roadmap.md) · [Chapter 210 — The JAX Ecosystem](ch210-jax-ecosystem-functional-neural-compilation-stack.md)

---

## Table of Contents

- [213.1 Interpretability as Decompilation](#2131-interpretability-as-decompilation)
- [213.2 The Superposition Hypothesis](#2132-the-superposition-hypothesis)
  - [The Core Claim](#the-core-claim)
  - [The Toy Model Demonstration](#the-toy-model-demonstration)
  - [Register Allocation as an Analogy](#register-allocation-as-an-analogy)
  - [Implications for Interpretability](#implications-for-interpretability)
- [213.3 Sparse Autoencoders: Neural Decompilation](#2133-sparse-autoencoders-neural-decompilation)
  - [Architecture](#architecture)
  - [The Training Objective](#the-training-objective)
  - [The `sae_lens` Library](#the-saelens-library)
  - [What SAEs Find: The Feature Taxonomy](#what-saes-find-the-feature-taxonomy)
- [213.4 Circuit Analysis](#2134-circuit-analysis)
  - [The Residual Stream as Typed Dataflow Graph](#the-residual-stream-as-typed-dataflow-graph)
  - [Q-K-V Decomposition and the OV/QK Circuits](#q-k-v-decomposition-and-the-ovqk-circuits)
  - [The IOI Circuit: A Case Study in Reverse Engineering](#the-ioi-circuit-a-case-study-in-reverse-engineering)
- [213.5 Activation Patching and Causal Tracing](#2135-activation-patching-and-causal-tracing)
  - [The Core Operation](#the-core-operation)
  - [ROME-Style Causal Tracing](#rome-style-causal-tracing)
  - [TransformerLens: Surgical Access to the Residual Stream](#transformerlens-surgical-access-to-the-residual-stream)
  - [Path Patching](#path-patching)
- [213.6 nnsight: Remote and Large-Model Interventions](#2136-nnsight-remote-and-large-model-interventions)
  - [Architecture](#architecture)
  - [Remote Execution](#remote-execution)
  - [Comparison with TransformerLens](#comparison-with-transformerlens)
- [213.7 Attribution Graphs: Circuit Analysis at Scale](#2137-attribution-graphs-circuit-analysis-at-scale)
  - [From Toy Models to Production Scale](#from-toy-models-to-production-scale)
  - [Computing Attribution Scores](#computing-attribution-scores)
  - [Feature Absorption and Cross-Layer Geometry](#feature-absorption-and-cross-layer-geometry)
  - [Neuronpedia Integration](#neuronpedia-integration)
- [213.8 Representation Engineering](#2138-representation-engineering)
  - [Concept Vectors as Linear Directions](#concept-vectors-as-linear-directions)
  - [Identifying Concept Directions](#identifying-concept-directions)
  - [Steering vs Weight Editing](#steering-vs-weight-editing)
  - [The Linear Representation Hypothesis at Scale](#the-linear-representation-hypothesis-at-scale)
- [213.9 Interpretability as Capability Mapping](#2139-interpretability-as-capability-mapping)
  - [The Capability Inventory](#the-capability-inventory)
  - [Feedback into the Self-Improvement Loop](#feedback-into-the-self-improvement-loop)
- [Chapter Summary](#chapter-summary)

---

## 213.1 Interpretability as Decompilation

A modern transformer trained on a trillion tokens is, in one sense, just a matrix multiplication pipeline. In another sense, it is the most compact encoding of a large fraction of human knowledge ever produced. The tension between these descriptions is not philosophical: it is the same tension that exists between a stripped x86 binary and the C++ source that produced it. The binary is semantically complete — it runs correctly — but its structure encodes nothing a human would call "intent". Recovery of that intent is the decompiler's job.

The analogy is precise at every level:

| Compilation Layer | Binary Compilation | Neural Network |
|---|---|---|
| Source program | C++/Rust source | High-level capabilities (reasoning, retrieval, translation) |
| High-level IR | LLVM IR | Computational circuits (IOI circuit, induction circuit) |
| Low-level IR | Machine code | Layer activations, attention patterns |
| Binary artifact | `.text` section bytes | Trained weights `W_Q, W_K, W_V, W_O, W_{MLP}` |
| Decompiler output | Recovered pseudocode | Sparse autoencoder features + circuit diagrams |

Three properties make this analogy more than cosmetic. First, both maps are many-to-one: different weight configurations implement the same capability, just as different binaries can implement the same algorithm. Second, both artifacts are the output of an optimization process: compilation minimises code size and instruction count; training minimises cross-entropy loss. Neither optimizer preserves human-readable structure. Third, in both cases, recovery of high-level structure requires identifying *invariants* — properties preserved across the many-to-one map. For compilers, type information and control flow are the invariants. For neural networks, the invariants are feature directions in activation space and the causal relationships between components.

The decompilation analogy drives the methodology developed in this chapter. Sparse autoencoders (§213.3) are the decompiler front end: they recover the "assembly language" of the network — the sparse, monosemantic features that individual directions in activation space represent. Circuit analysis (§213.4) recovers the "function bodies" — coherent subsets of attention heads and MLP layers that implement specific algorithms. Causal tracing (§213.5) is the debugger: it locates which components are causally responsible for a given output. Attribution graphs (§213.7) assemble the complete "call graph" — information flow across the full model for a given capability. A practical guide to the full methodological toolkit — covering activation patching, linear probing, circuit discovery, and SAE-based feature analysis — is [Rauker et al. (2024), "Toward Transparent AI: A Survey on Interpreting the Inner Structures of Deep Neural Networks", arXiv 2407.02646](https://arxiv.org/abs/2407.02646), which serves as the field's methodological reference.

## 213.2 The Superposition Hypothesis

### The Core Claim

A neural network layer with n neurons cannot represent m >> n independent features — unless those features are *sparse*. The superposition hypothesis, developed in Elman's early connectionist networks, formalised by Tishby's information bottleneck work, and given its modern form by Olshausen & Field's sparse coding theory and the Anthropic 2022 paper ["Toy Models of Superposition"](https://arxiv.org/abs/2209.10652), makes this precise: if a set of m features is k-sparse (at most k active simultaneously, k << m), a network with n neurons can represent all m features with acceptable interference, as long as k is small relative to n.

The formal model is as follows. Represent each feature as a unit vector f_i ∈ ℝ^n. The network's hidden state h is a superposition of active features:

```
h = Σ_{i ∈ S} a_i · f_i,  |S| = k << n
```

where a_i are scalar feature activations and S is the active feature set for a given input. The reconstruction task — recovering a_i from h — is underdetermined when m > n, but solvable with high probability when features are sparse and nearly orthogonal. The Welch bound from coding theory places a lower bound on the maximum pairwise inner product: for m unit vectors in ℝ^n, at least one pair satisfies |⟨f_i, f_j⟩| ≥ √((m-n)/(n(m-1))), which *increases* with m and approaches 1/√n as m → ∞. This means interference between features is unavoidable and grows with the expansion ratio — superposition is possible not because interference disappears, but because k-sparsity makes the *expected* interference per active feature manageable. When at most k features are simultaneously active, a given feature f_j interferes with at most k-1 others at any moment, and the interference terms sum to a small perturbation rather than a systematic error.

The consequence is a model in which *individual neurons are polysemantic*: a single neuron h_j responds to feature f_i whenever f_{ij} ≠ 0. A neuron monitoring logit position in a language model may fire for "German text", "base64 content", and "programming syntax" simultaneously — because all three features share substantial weight on that neuron's direction. This is polysemanticity, and it makes naive neuron-level analysis useless.

### The Toy Model Demonstration

The Anthropic 2022 paper demonstrates superposition with a minimal architecture: a linear encoder W ∈ ℝ^{n×m} (with n < m), a ReLU nonlinearity, and a linear decoder W^T. The model is trained to reconstruct m-dimensional inputs drawn from a sparse prior. With n = 2 neurons and m = 5 features, the learned encoder places 5 feature vectors as near-equally spaced directions in 2D space — a "pentagon" in activation space. Five features that should require 5 orthogonal dimensions are compressed into 2 dimensions with small but nonzero interference. The ReLU nonlinearity is essential: it allows the model to disambiguate features that would interfere under linear reconstruction, by suppressing negative activations.

### Register Allocation as an Analogy

Superposition is structurally equivalent to register allocation under severe register pressure. A compiler with n physical registers must spill m live variables into memory, reloading them as needed. Superposition is the neural analogue: n neurons "spill" m features into the shared activation space, relying on the sparsity of simultaneous activations (k << m) to avoid catastrophic interference. The "conflict graph" in register allocation — which values may not share a register because they are simultaneously live — corresponds to the feature co-occurrence graph in superposition: features that fire together simultaneously cannot share the same neuron direction without significant interference.

The analogy suggests why superposition emerges from gradient descent. A model representing more features generalises better (lower loss), just as a program that retains more variables avoids recomputing them. The optimizer finds the packed representation automatically, just as a register allocator finds the packed assignment — but neither algorithm makes the result easy for a human to read.

### Implications for Interpretability

The superposition hypothesis shifts the unit of interpretability from *neurons* to *directions in activation space*. Features are not neuron activations — they are linear functionals of the activation vector. The interpretable basis is not the standard basis {e_1, ..., e_n} aligned with individual neurons; it is a learned, overcomplete basis {f_1, ..., f_m} with m >> n. Recovering this basis from the weights is the sparse autoencoder's task.

## 213.3 Sparse Autoencoders: Neural Decompilation

### Architecture

A sparse autoencoder (SAE) is a two-layer network trained to reconstruct a model's internal activations from a sparse, overcomplete representation. The design mirrors the encoder-decoder pattern familiar from compression:

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SparseAutoencoder(nn.Module):
    """
    Overcomplete sparse autoencoder for decomposing superposed features.
    d_sae >> d_model (typically d_sae = 16 * d_model or larger).
    """
    def __init__(self, d_model: int, d_sae: int):
        super().__init__()
        self.W_enc = nn.Parameter(torch.randn(d_model, d_sae) * 0.02)
        self.b_enc = nn.Parameter(torch.zeros(d_sae))
        # Decoder columns are normalised to the unit sphere
        self.W_dec = nn.Parameter(torch.randn(d_sae, d_model) * 0.02)
        self.b_dec = nn.Parameter(torch.zeros(d_model))

    def encode(self, x: torch.Tensor) -> torch.Tensor:
        """Map model activations to sparse feature space."""
        return F.relu(x @ self.W_enc + self.b_enc)  # h ∈ ℝ^{d_sae}, sparse

    def decode(self, h: torch.Tensor) -> torch.Tensor:
        """Reconstruct activations from feature activations."""
        return h @ self.W_dec + self.b_dec           # x̂ ∈ ℝ^{d_model}

    def forward(self, x: torch.Tensor) -> tuple[torch.Tensor, torch.Tensor]:
        h = self.encode(x)
        x_hat = self.decode(h)
        return h, x_hat

    def loss(self, x: torch.Tensor, lambda_: float = 0.01) -> torch.Tensor:
        h, x_hat = self(x)
        recon_loss = ((x - x_hat) ** 2).mean()
        sparsity_loss = h.abs().mean()  # L1 penalty drives feature sparsity
        return recon_loss + lambda_ * sparsity_loss
```

The key hyperparameter is `d_sae / d_model`, the *expansion ratio*. A comprehensive survey of SAE architectures, training objectives, and evaluation methodologies is available in [Lieberum et al. (2025), arXiv 2503.05613](https://arxiv.org/abs/2503.05613). Anthropic's 2024 work on Claude 3 Sonnet used expansion ratios of 16× to 128×; a layer with d_model = 4096 receives an SAE with d_sae = 65536 to 524288 features. The dictionary is overcomplete by a factor that matches the estimated number of distinct features the model represents.

### The Training Objective

The SAE optimises a sum of two objectives whose tension defines the quality of the decomposition. The reconstruction loss `||x - x̂||²` ensures that the feature representation preserves the model's information content. The sparsity loss `λ · ||h||₁` (L1 penalty on feature activations) ensures that each activation vector is explained by a small number of active features. The tradeoff parameter λ controls feature density: too large and the SAE learns a trivial all-zeros solution; too small and it learns a dense non-interpretable representation.

Three additional training considerations distinguish production SAE training from naive application:

**Dead neuron prevention.** A substantial fraction of SAE features "die" during training — their activation goes to zero and never recovers, because they receive no gradient signal. Two approaches address this. Ghost grading applies a synthetic gradient to dead features based on the residual reconstruction error, nudging them toward unused directions. Auxiliary loss (used in Anthropic's 2024 training) adds a term that rewards feature activation, creating pressure for all features to find useful directions.

**Decoder column normalisation.** The decoder matrix W_dec is normalised so that each column has unit L2 norm. This prevents the trivial solution where a single large-magnitude decoder column absorbs all reconstruction, and ensures that each feature direction f_i = W_dec[:, i] lives on the unit sphere in ℝ^{d_model}. The geometric interpretation is clean: features are *directions*, not scales, in activation space.

**Data collection.** SAEs are not trained on token sequences directly but on *cached activations*. For a given target layer l (e.g., the residual stream after MLP block 12), the trainer runs the base model on a large corpus and saves the resulting activation tensors. The SAE then trains on this fixed dataset of activation vectors. This separation of concerns is crucial: the base model's weights are frozen; only the SAE learns.

### The `sae_lens` Library

`sae_lens` provides a standardised interface for loading and using pre-trained SAEs from the community:

```python
from sae_lens import SAE, SAEConfig

# Load a pre-trained SAE from the community hub (Joseph Bloom's GPT-2 SAEs)
sae, cfg_dict, sparsity = SAE.from_pretrained(
    release="gpt2-small-res-jb",
    sae_id="blocks.8.hook_resid_pre",  # residual stream before layer 8
)

# Encode activations to feature space
import transformer_lens
model = transformer_lens.HookedTransformer.from_pretrained("gpt2")
tokens = model.to_tokens("The Eiffel Tower is located in")
_, cache = model.run_with_cache(tokens)

activations = cache["blocks.8.hook_resid_pre"]  # (batch, seq, d_model)
feature_acts = sae.encode(activations)           # (batch, seq, d_sae), sparse

# Identify which features fire for the last token
last_tok_features = feature_acts[0, -1, :]      # (d_sae,)
top_features = last_tok_features.topk(10)
print("Top features:", top_features.indices.tolist())
print("Activations:", top_features.values.tolist())

# Decode back to activation space
x_hat = sae.decode(feature_acts)
reconstruction_error = ((activations - x_hat) ** 2).mean().item()
print(f"Reconstruction MSE: {reconstruction_error:.6f}")
```

Neuronpedia (neuronpedia.org) hosts feature dashboards for thousands of pre-trained SAEs. Each feature is characterised by its maximum-activating dataset examples — the token contexts that cause the largest values of h_i. Feature 742 in the GPT-2 small residual stream layer-8 SAE activates for "DNA sequences and nucleotide notation"; feature 1803 activates for "base64-encoded text strings". These are monosemantic features — a single interpretable concept per feature direction — in contrast to the polysemantic neurons that the SAE decomposes.

### What SAEs Find: The Feature Taxonomy

Analysis of large-scale SAEs (Anthropic's Claude 3 Sonnet SAE, with millions of features across all layers) reveals a consistent taxonomy:

| Feature Category | Examples | Layer Depth |
|---|---|---|
| Token-level | Whitespace, punctuation, specific subword tokens | Early layers (0-4) |
| Positional | Beginning-of-sentence, specific token positions | Early to mid |
| Syntactic | Subject NP, verb phrase, clause boundaries | Mid layers (4-12) |
| Lexical-semantic | Country names, programming keywords, chemical terms | Mid to late |
| Entity-level | Named entities, concepts, domain-specific entities | Late layers |
| Abstract | Sentiment, truthfulness, uncertainty, logical connectives | Final layers |

The depth correlation is not accidental — it mirrors the hierarchical representations that linear probing studies have identified across decades of neural network analysis. What SAEs add is *precision*: instead of "late layers encode entity information" (a coarse linear probe result), SAEs identify specific feature directions that activate for "DNA sequences" vs "city names" vs "chemical formulae", all in the same layer.

Feature geometry offers a further window: features corresponding to semantically related concepts cluster in activation space. The set of country-capital features forms a coherent subspace; arithmetic operation features form another. This cluster structure is the geometric trace of the model's internal ontology — the organised representation of concepts that underlies its capabilities.

## 213.4 Circuit Analysis

### The Residual Stream as Typed Dataflow Graph

A transformer's residual stream is the defining architectural choice that makes circuit analysis tractable. In a residual architecture, each token position maintains a vector x_t ∈ ℝ^{d_model} that accumulates contributions from each layer:

```
x_t^{l+1} = x_t^l + Attn_l(x^l)_t + MLP_l(x_t^l + Attn_l(x^l)_t)
```

The residual stream is a *typed dataflow register*: each position t is a node that receives writes from attention heads and MLP layers, and from which subsequent layers read. Unlike a standard neural network where each layer completely overwrites its input, the residual architecture preserves a persistent accumulation channel that any layer can read or modify. This makes the stream directly analogous to a typed register in an SSA-form IR: named, persistently allocated, with explicit write operations from each layer.

The key analytic consequence: contributions to the residual stream from different components are *additive and separable*. The final logit vector at position t is the sum of individual head contributions and MLP contributions across all layers — a sum that can be decomposed and attributed to individual components without approximation.

### Q-K-V Decomposition and the OV/QK Circuits

Each attention head h in layer l performs:

```
Attn_{l,h}(x)_t = W_O^{l,h} · softmax(A^{l,h}) · (x W_V^{l,h})
```

where the attention pattern A^{l,h}_{t,s} = softmax((x_t W_Q^{l,h})(x_s W_K^{l,h})^T / √d_h) and all weight matrices are in ℝ^{d_model × d_head} or ℝ^{d_head × d_model}.

Two derived matrices capture the head's full linear behaviour:

**OV circuit**: `W_{OV}^{l,h} = W_O^{l,h} W_V^{l,h}` ∈ ℝ^{d_model × d_model}. This matrix completely characterises what information the head moves from source positions to destination positions, independent of where it attends. A head with W_OV ≈ -I copies negated residual stream content; a head with W_OV ≈ I copies it unchanged. The OV circuit is the head's "value transform" — the function it applies to information it attends to.

**QK circuit**: `W_{QK}^{l,h} = W_Q^{l,h} (W_K^{l,h})^T` ∈ ℝ^{d_model × d_model}. This matrix determines which query-key pairs attract attention: `A_{t,s} ∝ exp(x_t W_{QK} x_s^T)`. A head with W_QK = e_1 e_2^T attends to positions s when x_s has a large component along e_2, from query positions t with a large component along e_1. The QK circuit is the head's "selection function" — the pattern of which positions it reads from.

This decomposition is not an approximation: for a frozen attention pattern, Attn_{l,h} is exactly the composition of the OV and QK circuits. Circuit analysis uses these matrices to characterise head behaviour without enumerating every input — the same role that transfer function analysis plays in signal processing.

### The IOI Circuit: A Case Study in Reverse Engineering

The Indirect Object Identification (IOI) task, from [Wang et al. (2022), arXiv 2211.00593](https://arxiv.org/abs/2211.00593), is the canonical circuit reverse-engineering example. The task: given a sentence such as "When Mary and John went to the store, John gave a book to", predict the correct continuation "Mary" (not "John"). This requires identifying that "John" is repeated (making it the subject), that "Mary" is unique (making it the indirect object), and that the blank should be filled with the unique name.

The IOI circuit involves 26 attention heads across GPT-2 medium that implement this algorithm:

| Component | Role | Mechanism |
|---|---|---|
| **Duplicate Token Heads** (layers 0-3) | Identify the repeated name "John" | QK circuit attends from second occurrence of a name to its first occurrence; writes a signal to the residual stream at the second-occurrence position |
| **S-Inhibition Heads** (layers 7-8) | Suppress the repeated name from the output | Read the duplicate token signal; write a negative contribution to "John" logits |
| **Name Mover Heads** (layers 9-11) | Copy the unique name to the output position | OV circuit moves "Mary" from its position in the context to the final token's residual stream, boosting "Mary" logits |

The circuit is *mechanistically complete*: ablating just these 26 heads (out of 144 in GPT-2 medium) degrades IOI performance to chance. Ablating all other heads has negligible effect. This is the neural network equivalent of identifying a specific function — the IOI algorithm — and its precise implementation in weight-space.

The IOI paper introduced the methodological template that subsequent circuit analyses follow: (1) design a clean task with controlled inputs; (2) use activation patching to identify which components matter; (3) characterise each component's behaviour using the QK/OV decomposition; (4) verify by ablation that the identified circuit is causally sufficient.

## 213.5 Activation Patching and Causal Tracing

### The Core Operation

Activation patching is the interpretability analogue of a software debugger's "modify-and-replay" primitive. The procedure is:

1. **Clean run**: execute the model on a clean input that produces the correct output; record all intermediate activations {a_l^{clean}}.
2. **Corrupted run**: execute on a corrupted input (a minimal perturbation that changes the output); record activations {a_l^{corrupt}}.
3. **Patch run**: execute with the corrupted input but restore one intermediate activation to its clean value at a specific (layer, position) pair.
4. **Measure**: how much does restoring this activation recover the correct output (measured as logit difference, probability change, or accuracy)?

The result is a causal attribution score for each (layer, position) pair. Components where patching fully restores the output are causally necessary for the capability. Components where patching has no effect are uninvolved. The score matrix across all (layer, position) pairs is a causal attention map over the model's computational graph.

### ROME-Style Causal Tracing

The ROME paper ([Meng et al. 2022, arXiv 2202.05262](https://arxiv.org/abs/2202.05262)) applies this methodology to factual association: the claim "The Eiffel Tower is in Paris". A corrupted input replaces "Eiffel Tower" with noise, breaking the factual association. Causal tracing identifies:

- Attention heads in early layers restore some signal by attending to the subject tokens
- A small cluster of FFN (MLP) layers at specific positions provide the *causal locus* — restoring these FFN activations alone is sufficient to recover the correct output
- The factual association "Eiffel Tower → Paris" is localised to specific MLP layers, not distributed uniformly across all weights

This localisation result is the foundation for ROME's editing procedure (Chapter 214): because factual associations live in specific FFN layers, they can be modified by rank-1 updates to those layers' weight matrices, without disturbing other capabilities. MEMIT ([Meng et al. 2022, arXiv 2210.07229](https://arxiv.org/abs/2210.07229)) generalises ROME from single-fact edits to mass editing: it solves a least-squares problem over a batch of thousands of target associations simultaneously, distributing updates across multiple FFN layers to reduce per-layer constraint pressure. Mechanistic interpretability's causal localisation is the prerequisite for both methods — without knowing that factual associations live in FFN layers at subject token positions, neither ROME nor MEMIT would know where to apply their updates.

### TransformerLens: Surgical Access to the Residual Stream

TransformerLens ([Nanda et al.](https://github.com/TransformerLensOrg/TransformerLens)) re-implements transformer models (GPT-2, GPT-Neo, LLaMA, Gemma, Mistral, and dozens of others) with systematic instrumentation points — "hook points" — at every semantically meaningful activation in the forward pass:

```python
from transformer_lens import HookedTransformer
import torch

model = HookedTransformer.from_pretrained("gpt2-small")  # 117M params
tokens = model.to_tokens(
    "When Mary and John went to the store, John gave a book to"
)  # shape: (1, seq_len)

# Run with cache: collect all intermediate activations
logits, cache = model.run_with_cache(tokens)

# Every named activation is accessible by hook name
resid_pre_8   = cache["blocks.8.hook_resid_pre"]      # (1, seq, 768) — residual stream before layer 8
resid_post_8  = cache["blocks.8.hook_resid_post"]     # (1, seq, 768) — after layer 8
attn_weights  = cache["blocks.8.attn.hook_pattern"]   # (1, 12, seq, seq) — attention pattern, all heads
v_input       = cache["blocks.8.attn.hook_v"]         # (1, seq, 12, 64) — value vectors
mlp_post      = cache["blocks.8.hook_mlp_out"]        # (1, seq, 768) — MLP output

# Direct logit attribution: decompose final logit into per-layer contributions
# Each component's contribution to the "Mary" vs "John" logit difference
logit_lens = cache.apply_ln_to_stack(
    cache.stack_activation("resid_pre"), layer=-1, pos_slice=-1
)  # (n_layers + 1, 1, d_model) — residual stream at each layer for final position
```

Activation patching with TransformerLens uses hook functions:

```python
# Define clean and corrupted inputs
clean_tokens = model.to_tokens("When Mary and John went to the store, John gave a book to")
corrupted_tokens = model.to_tokens("When Mary and Gfqz went to the store, Gfqz gave a book to")

# Collect clean activations
_, clean_cache = model.run_with_cache(clean_tokens)

# Patch the layer-8 residual stream at the final token position
def patch_resid_pre_8(corrupted_resid, hook):
    """Replace the corrupted layer-8 residual with the clean value at position -1."""
    corrupted_resid[:, -1, :] = clean_cache["blocks.8.hook_resid_pre"][:, -1, :]
    return corrupted_resid

patched_logits = model.run_with_hooks(
    corrupted_tokens,
    fwd_hooks=[("blocks.8.hook_resid_pre", patch_resid_pre_8)]
)

# Mary token index and John token index
mary_idx = model.to_single_token(" Mary")
john_idx = model.to_single_token(" John")
patched_logit_diff = (
    patched_logits[0, -1, mary_idx] - patched_logits[0, -1, john_idx]
).item()
print(f"Patched logit diff (Mary - John): {patched_logit_diff:.3f}")
```

The `hook_points` system is architecturally clean: TransformerLens registers a `HookPoint` module at each named location; `run_with_cache` wraps every hook point to record its value; `run_with_hooks` injects user-supplied functions at specified hook points. The hook function receives the tensor at that point and returns a (potentially modified) replacement. This is the transformer equivalent of a compiler's instrumentation pass: every activation is accessible for reading or modification without changing the underlying model code.

### Path Patching

Activation patching attributes effects to individual components: restoring activation at (layer, position) measures the causal contribution of that single node. Path patching extends this to *edges* — specific information flows from a source component to a destination component. The operation:

1. Run clean and corrupted models; save all activations
2. For a source component S and destination component D: run the corrupted model, but when computing D's input, substitute the specific activation that S would have sent in the clean run
3. Measure how much this edge substitution recovers the correct output

Path patching allows isolating a specific pathway through the network — for example, "the information that head 9.6 sends to the output logits via the residual stream, mediated by head 10.7's key input" — and measuring its independent causal contribution. This is the circuit analyst's precision instrument for separating the contributions of different information routes, analogous to signal tracing in an electronic circuit.

## 213.6 nnsight: Remote and Large-Model Interventions

### Architecture

TransformerLens requires re-implementing the model from scratch in a custom codebase, which introduces maintenance burden and limits the models it can instrument. `nnsight` takes a different approach: it operates on models in their *native HuggingFace implementations* by intercepting the forward pass through PyTorch's `__call__` dispatch mechanism. The computation graph is represented as an explicit program object that can be constructed, inspected, modified, and dispatched — either locally or to a remote inference server.

```python
import nnsight
from nnsight import LanguageModel

# Wrap any HuggingFace model — native implementation preserved
model = LanguageModel("meta-llama/Meta-Llama-3-8B", device_map="auto")

tokens = model.tokenizer("The Eiffel Tower is located in", return_tensors="pt")

# Trace context: construct an intervention program without executing it
with model.trace(tokens) as tracer:
    # Access any module by its HuggingFace attribute path
    # .save() creates a node in the intervention graph that captures the value
    mlp_act_layer8 = model.model.layers[8].mlp.act_fn.output.save()

    # Modify activations in place: tensor operations on saved nodes
    model.model.layers[8].mlp.act_fn.output[:] = (
        model.model.layers[8].mlp.act_fn.output * 0.0
    )  # zero out layer 8 MLP activations

    output_logits = model.lm_head.output.save()

# After the context exits, the trace executes
print("Layer 8 MLP activations:", mlp_act_layer8.value.shape)
print("Output logits:", output_logits.value.shape)
```

The `model.trace(tokens)` context manager does not immediately execute the forward pass. Instead, it builds an intervention program — a directed graph of operations on model internals. When the context exits, `nnsight` executes the program, injecting the specified interventions at the appropriate points in the native HuggingFace forward pass.

### Remote Execution

nnsight's most distinctive capability is remote execution. The intervention program is a serialisable Python object; nnsight can transmit it to the NDIF (National Deep Inference Fabric) infrastructure, which hosts large models (LLaMA 3 70B, Mixtral, and others) on shared GPU clusters:

```python
# Remote execution: the intervention program runs on NDIF servers
# Local machine only needs the nnsight library, not the model weights
with model.trace(tokens, remote=True) as tracer:
    # Same intervention API — the program runs remotely
    hidden_states = model.model.layers[40].self_attn.o_proj.output.save()
    # Patch a specific head's output
    model.model.layers[40].self_attn.o_proj.output[:, :, :128] = 0.0

print("Layer 40 head outputs (remote):", hidden_states.value.shape)
```

This remote capability is significant: it enables mechanistic interpretability research on models too large to run locally. The intervention program — the "what to measure and modify" — is specified locally; the expensive computation runs on shared infrastructure. This is analogous to remote debugging via gdb-server: the debugger frontend and the execution environment are separated, but the intervention API is identical.

### Comparison with TransformerLens

| Dimension | TransformerLens | nnsight |
|---|---|---|
| Model coverage | Reimplemented subset (~50 models) | Any HuggingFace model |
| Implementation | Custom re-implementation | Native HF implementation |
| Remote execution | No | Yes (NDIF) |
| Hook API | `run_with_hooks(fwd_hooks=[...])` | `model.trace()` context manager |
| Cache access | `run_with_cache()` returns `ActivationCache` | `.save()` on any module output |
| IOI circuit tooling | Extensive built-in support | General-purpose |
| Scale | Practical for ≤7B on single GPU | Practical for 70B+ via remote |

The two libraries are complementary. TransformerLens provides richer built-in analysis primitives (logit lens, direct logit attribution, attention pattern visualisation, hook naming conventions aligned with the IOI circuit literature). nnsight provides access to a wider model zoo and production-scale models via remote execution. A complete mechanistic interpretability workflow may use TransformerLens for circuit discovery on smaller models, then validate findings on larger models via nnsight.

## 213.7 Attribution Graphs: Circuit Analysis at Scale

### From Toy Models to Production Scale

The IOI circuit analysis demonstrated that mechanistic interpretability is feasible on GPT-2 medium (345M parameters, 24 layers, 16 heads). Scaling to production models — Claude 3 Sonnet at ~70B parameters, thousands of attention heads, and millions of MLP neurons — requires moving beyond manual circuit tracing to automated attribution infrastructure. The Anthropic 2025 work on "On the Biology of a Large Language Model" (transformer-circuits.pub) introduces attribution graphs as the scalable generalization.

An attribution graph is a directed weighted graph where:
- **Nodes** are model components: SAE features, attention heads, MLP "neurons" (individual columns of W_up, or equivalently individual ReLU activations), and the output logits
- **Edges** are causal attribution scores: how much does activating node A causally influence the activation of node B, holding other paths constant?
- **Edge weights** are computed via a single backward pass or by linearising the model around a specific activation

The result is a complete map of information flow for a specific input and capability — the model's "call graph" for answering a given question, analogous to a profiler's flame graph.

### Computing Attribution Scores

Attribution scores between SAE features are computed via *linear attribution through the residual stream*:

```python
import torch
from sae_lens import SAE
from transformer_lens import HookedTransformer

def compute_feature_attribution(
    model: HookedTransformer,
    sae_source: SAE,
    sae_target: SAE,
    tokens: torch.Tensor,
    source_layer: int,
    target_layer: int,
    source_feature: int,
    target_feature: int,
) -> float:
    """
    Compute the causal attribution from source_feature at source_layer
    to target_feature at target_layer via the residual stream Jacobian.
    """
    _, cache = model.run_with_cache(tokens)

    # Feature activations at source layer
    source_acts = cache[f"blocks.{source_layer}.hook_resid_post"]
    h_source = sae_source.encode(source_acts)  # (batch, seq, d_sae_source)

    # Jacobian of target feature activation w.r.t. source feature activation
    # Linearise through the residual stream between source and target layers
    # W_path: composition of layer-to-layer Jacobians in the residual stream
    target_acts = cache[f"blocks.{target_layer}.hook_resid_pre"]

    # The decoder column for the source feature: d_model direction it writes
    source_direction = sae_source.W_dec[source_feature, :]  # (d_model,)

    # The encoder row for the target feature: what direction in d_model it reads
    target_direction = sae_target.W_enc[:, target_feature]  # (d_model,)

    # Attribution = dot product of source write direction and target read direction
    # (assumes residual stream is approximately linear between the two layers)
    attribution = (source_direction @ target_direction).item()
    return attribution
```

This linearised attribution is an approximation; the full Jacobian captures nonlinear effects from intermediate layers. The Anthropic methodology uses a combination of exact path patching and linearised attribution to balance computational cost against accuracy.

### Feature Absorption and Cross-Layer Geometry

Attribution graphs reveal *feature absorption*: a feature in a later layer whose activation is causally downstream from multiple earlier features. Feature 8741 in layer 24 ("confidence in a factual claim") may receive contributions from feature 342 in layer 8 ("named entity is a city"), feature 1203 in layer 12 ("entity is European"), and feature 5512 in layer 16 ("entity is a capital city"). The attribution graph makes this compositional structure explicit — the model's reasoning chain for "Is Paris the capital of France?" is a traceable path through the attribution graph.

The "universal features" hypothesis, partially supported by cross-model comparisons, holds that some SAE features recur across different model families trained on the same corpus. The "Python syntax" feature cluster, the "base64 encoding" feature, and certain mathematical operation features appear in GPT-2, LLaMA 2, and Gemma models with similar representations. This universality, if robust, would mean that mechanistic interpretability findings transfer across models — the features are properties of the data distribution, not arbitrary artifacts of a particular training run.

### Neuronpedia Integration

Neuronpedia (neuronpedia.org) provides a web interface for exploring attribution graphs. The dashboard for each SAE feature shows:
- Maximum activating dataset examples (top-k token contexts by h_i magnitude)
- Logit attribution: which output tokens does this feature boost or suppress?
- Upstream features: which earlier features causally activate this one?
- Downstream features: which later features does this one causally influence?
- Ablation effects: what capability degrades when this feature is suppressed?

The tooling transforms attribution graphs from static analysis artifacts into interactive architecture diagrams — the neural network's internal call graph made navigable.

## 213.8 Representation Engineering

### Concept Vectors as Linear Directions

Representation engineering ([Zou et al. 2023, arXiv 2310.01405](https://arxiv.org/abs/2310.01405)) identifies a complementary approach to understanding model internals: instead of decomposing activations into features (the SAE approach), it identifies *concept directions* — unit vectors in the residual stream that linearly encode abstract properties like honesty, emotional valence, or conceptual categories.

The core claim is the **linear representation hypothesis**: if a transformer represents concept C, then at some layer l, there exists a direction v ∈ ℝ^{d_model} such that `cos_sim(h_t^l, v) ∝ P(C | context_t)`. The probability of the concept being "active" in the context is proportional to the dot product of the residual stream with the concept direction. This is a testable, falsifiable claim — and the evidence for it across honesty, safety-relevant concepts, emotional states, and factual associations is substantially positive.

### Identifying Concept Directions

The methodology uses *population-level contrastive probing*. For concept C, construct paired prompt sets:
- Positive prompts: contexts where C is present ("Describe an honest response to...")
- Negative prompts: paired contexts where C is absent ("Describe a deceptive response to...")

Run the model on both sets, extract residual stream activations at each layer, and compute the difference of means:

```
v_l = mean(h_l | positive) - mean(h_l | negative)
v_l = v_l / ||v_l||_2  # normalise to unit sphere
```

The resulting direction v_l is the concept vector for C at layer l. A good concept direction satisfies:
- **Probing accuracy**: a linear classifier `sign(h · v_l)` accurately predicts whether C is present in held-out contexts
- **Causal interventionality**: steering along v_l (adding α·v_l to the residual stream) predictably shifts model behaviour toward or away from C

The `repe` library ([repe GitHub](https://github.com/andyzoujm/representation-engineering)) implements this pipeline:

```python
from repe import repe_pipeline
from transformers import AutoTokenizer, pipeline

tokenizer = AutoTokenizer.from_pretrained("meta-llama/Meta-Llama-3-8B-Instruct")
base_pipeline = pipeline("text-generation", model="meta-llama/Meta-Llama-3-8B-Instruct")

# repe wraps the pipeline with representation reading/writing capabilities
rep_pipeline = repe_pipeline(
    "text-generation",
    model=base_pipeline.model,
    tokenizer=tokenizer,
    device=0,
)

# Paired training inputs: (positive, negative) contexts for "honesty"
train_inputs = [
    ("Pretend you're an honest AI that never lies. You tell the truth even when it's uncomfortable.",
     "Pretend you're a deceptive AI that always lies. You hide the truth to manipulate people."),
    ("An honest response to this question would be...",
     "A deceptive response to this question would be..."),
    # ... more pairs
]

# Identify the concept direction across all layers
rep_reader = rep_pipeline.get_directions(
    train_inputs,
    rep_token=-1,           # extract at the last token position
    hidden_layers=list(range(1, 33)),  # all transformer layers
    n_difference=1,         # use 1-hop difference of means
    train_labels=None,      # unsupervised; uses mean-diff, not logistic regression
)

# Apply control vector during generation
test_prompt = "Should I tell the user that I don't know the answer to their question?"
honest_output = rep_pipeline(
    test_prompt,
    rep_reader=rep_reader,
    rep_control_pipeline_config={
        "coeff": 3.0,        # positive = steer toward honest direction
        "max_new_tokens": 100,
    },
)

deceptive_output = rep_pipeline(
    test_prompt,
    rep_reader=rep_reader,
    rep_control_pipeline_config={
        "coeff": -3.0,       # negative = steer away from honest direction
        "max_new_tokens": 100,
    },
)

print("Honest steering:", honest_output[0]["generated_text"])
print("Deceptive steering:", deceptive_output[0]["generated_text"])
```

### Steering vs Weight Editing

Representation engineering produces *inference-time* interventions: adding α·v to the residual stream at layer l during every forward pass, without modifying any model weights. This is the analogue of a compiler's optimization hint — a runtime annotation that influences execution without changing the binary. The effect lasts only for the duration of the patched forward pass; no model state is modified.

This distinguishes representation engineering from the weight-editing methods (ROME, MEMIT, Chapter 214). The contrast:

| Dimension | Representation Engineering (repe) | Weight Editing (ROME/MEMIT) |
|---|---|---|
| Mechanism | Residual stream injection: `h_l += α·v` | Rank-1 weight update: `W_l ← W_l + uv^T` |
| Persistence | Inference-time only; requires re-injection each forward pass | Permanent; baked into weights |
| Specificity | Broadly steers the concept at all positions | Edits specific factual associations at specific positions |
| Reversibility | Trivially reversible; remove injection | Requires re-editing or fine-tuning to undo |
| Interaction effects | Can conflict with model's natural activations | May cause unintended side effects across capabilities |
| Application | Real-time safety steering, personalisation | Knowledge correction, factual updates |

The two approaches are complementary tools in the mechanistic interpretability engineer's toolkit. Causal tracing (§213.5) identifies *where* to intervene. Representation engineering determines the right *direction* for inference-time steering. Weight editing (Chapter 214) makes the intervention permanent at the weight level.

### The Linear Representation Hypothesis at Scale

The Zou et al. results hold across a wide range of concepts — not just honesty/deception, but emotional valence, toxicity, factual category membership, and reasoning style. This breadth supports a strong version of the hypothesis: the residual stream represents many abstract concepts as linearly accessible directions at specific layers. Chapter 217's self-reflective inference architecture exploits this: at inference time, the model can query its own residual stream for concept activations, treating the linear representation hypothesis as a runtime introspection API.

The hypothesis has known failure modes. Compositional concepts ("A is an honest AI that has been trained on biased data") may require nonlinear combinations of concept directions. Concepts that change meaning across context (pragmatic honesty vs epistemic honesty) may have different directions in different prompt distributions. These failure modes are active research frontiers; the linear approximation is highly useful but not universal.

## 213.9 Interpretability as Capability Mapping

### The Capability Inventory

The synthesis of the chapter's tools yields a concrete engineering methodology for capability mapping — identifying, localising, and understanding the neural substrates of specific model capabilities. This is the mechanistic interpretability contribution to the self-improvement architecture developed across Chapters 214-218.

SAEs provide a **capability inventory**: a dictionary of m features, where each feature is a unit direction in ℝ^{d_model} associated with a specific semantic concept. Enumerating the features of a large SAE (millions of features across all layers) produces a list of concepts the model has learned to represent. Features corresponding to reasoning operations ("if-then inference at position t"), domain knowledge ("capital cities"), and metacognitive states ("uncertainty about own output") are all findable in SAE analyses of production models. The inventory is not complete — SAE training finds features that are sparse and active, not features that are dense or rarely active — but it is the most comprehensive enumeration of model representations available.

Circuit analysis provides a **capability map**: a graph relating capabilities to their implementing circuits. The IOI circuit implements indirect object identification. The induction circuit (a two-head circuit in early GPT-2 layers) implements in-context learning of repeated sequences. The arithmetic circuit implements multi-digit addition. Each circuit occupies a specific subgraph of the model's component graph, using specific attention heads and MLP neurons to compute a specific algorithm. The capability map answers the question: "Which components implement capability X?"

Causal tracing provides a **precision localisation tool**: given a failing capability (the model hallucinates a fact), causal tracing identifies the specific (layer, position, component) tuple responsible for the failure. This narrows the search space from "all 70 billion parameters" to "these 12 FFN columns in layer 24, at the subject token position". The localisation result is the prerequisite for surgical intervention.

Attribution graphs provide an **architecture diagram**: the complete information flow for a given capability across all layers. The attribution graph for "answering a factual question about a European capital" shows which SAE features fire at which layers, which attention heads transport information between positions, and which MLP neurons compute the factual association. The graph is the neural network's internal documentation — the call graph that was never written down.

### Feedback into the Self-Improvement Loop

The connection to Chapter 214 (Gradient-Based Self-Modification) is direct. ROME and MEMIT locate their edits using exactly the causal tracing methodology of §213.5: they identify that factual associations live in specific FFN layers at specific positions, then apply rank-1 updates to those layers' weight matrices. Mechanistic interpretability is the prerequisite step — without knowing *where* a capability lives, weight editing is blind. With causal tracing, it is surgical.

The connection to Chapter 218 (Self-Improvement Fitness Functions) runs through SAE feature activations as measurable capability indicators. If target capability X is associated with SAE feature i (feature i fires when X is exercised), then monitoring feature i's activation across a test distribution provides a continuous, differentiable proxy for X's presence. A self-improvement loop that maximises feature i's activation is, in effect, maximising the model's tendency to exercise capability X — a mechanistically grounded fitness signal rather than an opaque behavioural metric.

Chapter 217 (Self-Reflective Inference) uses the linear representation hypothesis of §213.8 at runtime: the model reads its own concept vectors during generation, using the direction `v_l` for "uncertainty" or "factual confidence" as a real-time signal that can gate retrieval, trigger CoT reasoning, or flag outputs for verification. The residual stream becomes a runtime introspection API, accessible without external tools.

The cognitive gap analysis use case ties all threads together: if a model lacks capability X, the mechanistic questions are:
1. Does an SAE feature for X exist? If not, the model has not learned the feature — the capability gap is representational.
2. Does the IOI-style circuit for X exist? If not, the algorithm is not implemented — the gap is computational.
3. Does causal tracing identify a specific locus? If so, targeted weight editing (Chapter 214) can implant the missing circuit.
4. Does the attribution graph show broken information flow? If so, the circuit exists in isolation but fails to integrate with other capabilities — the gap is compositional.

Mechanistic interpretability does not repair capability gaps — that is Chapter 214's domain. But it provides the diagnostic tooling without which repair is blind. The relationship between the two chapters is precisely the relationship between a decompiler and a re-compiler: decompilation recovers structure; recompilation (gradient-based weight editing) modifies it with precision.

---

## Chapter Summary

- **Mechanistic interpretability is decompilation**: trained weights are machine code; sparse autoencoder features are the recovered assembly language; circuits are the identified functions; attribution graphs are the call graph. Every tool in this chapter has a direct compiler-analysis analogue.

- **The superposition hypothesis** explains why individual neurons are polysemantic: a network with n neurons can represent m >> n features when features are k-sparse (k << m). This is equivalent to register allocation under register pressure. The interpretable basis is a learned, overcomplete set of feature directions, not the standard neuron basis.

- **Sparse autoencoders** (SAEs) are the neural decompiler: trained on cached activations with an L1 sparsity penalty and an overcomplete dictionary (d_sae = 16–128 × d_model), they recover monosemantic features — individual directions in ℝ^{d_model} associated with specific semantic concepts. The `sae_lens` library provides pre-trained SAEs; Neuronpedia provides feature dashboards.

- **Circuit analysis** decomposes model behaviour into the QK circuit (which positions a head attends to) and the OV circuit (what information it moves). The IOI circuit paper (arXiv 2211.00593) demonstrates complete reverse engineering of a 26-head circuit implementing indirect object identification in GPT-2 medium.

- **Activation patching and causal tracing** identify which components are causally responsible for a capability by patching intermediate activations between clean and corrupted runs. ROME-style tracing localises factual associations to specific FFN layers at specific positions — the prerequisite for surgical weight editing in Chapter 214.

- **TransformerLens** provides systematic hook-point access to every named activation in re-implemented transformer models; **nnsight** provides the same intervention API for native HuggingFace models including large models via remote NDIF execution. Together they cover the full range from small research models to production-scale systems.

- **Attribution graphs** (Anthropic 2025) scale circuit analysis to production models by representing information flow as a weighted directed graph over SAE features, attention heads, and MLP neurons. The graph is the model's architecture diagram — made computationally recoverable for the first time.

- **Representation engineering** (Zou et al. 2023, arXiv 2310.01405) identifies concept directions — unit vectors in the residual stream linearly encoding abstract concepts — via population-level contrastive probing. The `repe` library implements inference-time steering (`h_l += α·v`) that predictably shifts model behaviour without weight modification, complementing the permanent weight-level interventions of Chapter 214.
