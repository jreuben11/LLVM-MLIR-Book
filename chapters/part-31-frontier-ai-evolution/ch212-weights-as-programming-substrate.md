# Chapter 212 — Weights as a Programming Substrate

*Part XXXI — Frontier AI Evolution*

A trained neural network's weight tensor is not merely a numerical array stored on disk — it is the densely compiled form of a cognitive program. The relationship between floating-point weights and the capabilities they implement is precisely analogous to the relationship between machine code and the algorithms it executes: both are the output of an optimization process that discards human-readable structure, both can be analysed with the right tools, and both can — subject to careful constraints — be modified at the binary level without recompiling from source. This chapter develops that analogy into an engineering discipline. It begins with the on-disk formats through which weight tensors are stored and transferred, showing that SafeTensors and GGUF serve the same role as ELF and Mach-O: structured containers that enable efficient, zero-copy loading of the program's data segment. It then characterizes the geometry of weight space — the high-dimensional parameter manifold on which trained models live — through the results of linear mode connectivity, neural collapse, and loss landscape topology. Building on that geometry, it develops capability arithmetic: the algebra by which fine-tuned capabilities encoded as weight-space vectors can be added, negated, scaled, and merged, with working JAX implementations of task vectors, TIES, DARE, and SLERP. The chapter closes by connecting weight-space operations to MLIR/TOSA and XLA HLO as structured IRs over weight tensor computations, and to the JAX programmatic interface through which all these operations are expressed in practice. Chapter 213 takes the complementary view — interpretability as decompilation of those weights — and Chapter 214 shows how gradient-based self-modification closes the loop.

Cross-references: [Chapter 211 — Neural Programs as Compiled Artifacts](ch211-neural-programs-compiled-artifacts.md) · [Chapter 213 — Mechanistic Interpretability Infrastructure](ch213-mechanistic-interpretability.md) · [Chapter 214 — Gradient-Based Self-Modification](ch214-gradient-based-self-modification.md) · [Chapter 154 — HLO and StableHLO](../part-22-xla-openxla/ch154-hlo-and-stablehlo.md) · [Chapter 164 — CUDA Tile IR](../part-23-mlir-production/ch164-cuda-tile-ir.md)

---

## 212.1 Weight Checkpoints as Compiled Binaries

An ELF file begins with a magic header, a program header table describing segment layout, and the segment data itself. A SafeTensors file has an analogous anatomy: an 8-byte little-endian uint64 giving the byte length of a UTF-8 JSON header, followed by that JSON header, followed by the raw tensor data. A GGUF file opens with a 4-byte magic identifier (`GGUF`, 0x47475546), a version field, and a versioned key-value metadata store before the tensor descriptors. Both formats are designed for memory-mapped loading — the data section can be mapped directly without deserialization, exactly as an ELF loader maps `.data` into the process address space with a single `mmap(2)` call.

### SafeTensors Format

SafeTensors was designed by Hugging Face to address two failure modes of the `pickle`-based `.pt` format: arbitrary code execution on load and the requirement to copy tensor data into NumPy arrays before mapping it to other backends. The SafeTensors format has no magic bytes prefix — a common misconception. Its structure is:

```
[0:8]     8-byte little-endian uint64  — byte length N of the JSON header
[8:8+N]   UTF-8 JSON object            — tensor metadata
[8+N:EOF] raw tensor data              — dtype-packed, row-major bytes
```

The JSON header maps tensor names to descriptor objects of the form:

```json
{
  "model.embed_tokens.weight": {
    "dtype":        "BF16",
    "shape":        [32000, 4096],
    "data_offsets": [0, 262144000]
  },
  "model.layers.0.self_attn.q_proj.weight": {
    "dtype":        "BF16",
    "shape":        [4096, 4096],
    "data_offsets": [262144000, 295698432]
  }
}
```

`data_offsets` are byte offsets relative to the start of the raw data region — that is, relative to file byte `8 + N`, not relative to file byte 0. This matters for the mmap loading pattern: the mmap base address must be advanced by `8 + N` before applying the tensor offsets. A `__metadata__` key is reserved for arbitrary string key-value pairs and must be excluded from tensor iteration.

```python
import struct, json, mmap, numpy as np
from pathlib import Path

def load_safetensors_mmap(path: str) -> dict[str, np.ndarray]:
    f = open(path, "rb")
    header_len = struct.unpack("<Q", f.read(8))[0]
    header = json.loads(f.read(header_len))
    data_start = 8 + header_len

    mm = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ)

    dtype_map = {
        "F32": np.float32, "F16": np.float16,
        "BF16": np.uint16,  "I32": np.int32,
        "I64": np.int64,    "U8":  np.uint8,
    }

    tensors = {}
    for name, meta in header.items():
        if name == "__metadata__":
            continue
        dtype = dtype_map[meta["dtype"]]
        start, end = meta["data_offsets"]
        buf = mm[data_start + start : data_start + end]
        arr = np.frombuffer(buf, dtype=dtype).reshape(meta["shape"])
        tensors[name] = arr

    return tensors
```

The `np.frombuffer` call does not copy the data — it creates a view into the memory-mapped region, producing true zero-copy loading. BF16 tensors are returned as `uint16` views since NumPy has no native bfloat16 dtype; the correct conversion on the JAX side uses the `ml_dtypes` package: `np.frombuffer(buf, dtype=ml_dtypes.bfloat16).reshape(meta["shape"])`, which interprets the bytes as bfloat16 without a byte-copy. The `jnp.array()` call then moves the data to device memory as needed.

### GGUF Format

GGUF (GGML Universal File Format) was designed by Georgi Gerganov for the `llama.cpp` ecosystem to support quantized model inference on consumer hardware. Its design goals differ from SafeTensors in one critical way: GGUF must encode quantization metadata because GGUF tensors may use blocked quantization types (Q4_0, Q4_K, Q8_0, and a dozen others) that require dequantization parameters stored alongside the data.

A GGUF file begins with:

```
[0:4]   magic     = 0x47475546  ('G','G','U','F')
[4:8]   version   = uint32      (currently 3)
[8:16]  n_tensors = uint64
[16:24] n_kv      = uint64
```

The key-value metadata store follows immediately, encoding model provenance, tokenizer configuration, rope scaling parameters, and any metadata the exporter chose to record. Each KV entry has a null-terminated string key, a uint32 type tag, and a typed value. After the KV store comes the tensor descriptor table, with one entry per tensor giving its name, number of dimensions, dimension sizes, quantization type, and offset into the data section. The data section is page-aligned (typically to 32 bytes) and contains tensor data in the quantization format specified per-tensor.

```python
import struct
from dataclasses import dataclass
from typing import Any

GGUF_MAGIC = 0x47475546

GGUF_TYPE_MAP = {
    0: ("uint8",   "B", 1),
    1: ("int8",    "b", 1),
    2: ("uint16",  "H", 2),
    3: ("int16",   "h", 2),
    4: ("uint32",  "I", 4),
    5: ("int32",   "i", 4),
    6: ("float32", "f", 4),
    7: ("bool",    "?", 1),
    8: ("string",  None, None),
    9: ("array",   None, None),
    10: ("uint64", "Q", 8),
    11: ("int64",  "q", 8),
    12: ("float64","d", 8),
}

@dataclass
class GGUFHeader:
    version: int
    n_tensors: int
    n_kv: int
    metadata: dict[str, Any]

def read_gguf_string(f) -> str:
    length = struct.unpack("<Q", f.read(8))[0]
    return f.read(length).decode("utf-8")

def read_gguf_header(path: str) -> GGUFHeader:
    with open(path, "rb") as f:
        magic = struct.unpack("<I", f.read(4))[0]
        if magic != GGUF_MAGIC:
            raise ValueError(f"Not a GGUF file: magic {magic:#010x}")
        version, n_tensors, n_kv = struct.unpack("<IQQ", f.read(20))
        metadata = {}
        for _ in range(n_kv):
            key = read_gguf_string(f)
            type_id = struct.unpack("<I", f.read(4))[0]
            if type_id == 8:
                metadata[key] = read_gguf_string(f)
            elif type_id in GGUF_TYPE_MAP:
                _, fmt, size = GGUF_TYPE_MAP[type_id]
                metadata[key] = struct.unpack(f"<{fmt}", f.read(size))[0]
        return GGUFHeader(version, n_tensors, n_kv, metadata)
```

### Format Comparison

| Property | SafeTensors | GGUF |
|---|---|---|
| Magic identifier | None — 8-byte header-length prefix | `GGUF` (0x47475546) |
| Header format | UTF-8 JSON | Versioned binary KV store |
| Quantization support | No (full-precision dtypes only) | Yes (Q4_0, Q4_K, Q8_0, ...) |
| Zero-copy mmap | Yes | Partial (dequant required for most types) |
| Metadata | `__metadata__` JSON key | Structured KV entries with typed values |
| Primary ecosystem | Hugging Face / Python | llama.cpp / C / consumer inference |
| Alignment | Not guaranteed | 32-byte padding before data section |

---

## 212.2 Weight-Space Geometry

A neural network with P parameters occupies a point in ℝ^P. Training traces a path through this space from random initialisation toward a basin of the loss landscape. Two independently trained models on the same data and architecture occupy two such points. The geometry of this high-dimensional space — its topology, curvature, and the distribution of low-loss regions — governs how models relate to one another and, crucially, whether arithmetic operations on weight vectors produce meaningful results.

### Linear Mode Connectivity

Two parameter vectors θ_A and θ_B are *linearly mode connected* (LMC) if the linear interpolant

```
θ(t) = (1 - t)·θ_A + t·θ_B,  t ∈ [0, 1]
```

passes through no loss barrier — that is, if the loss along the interpolant never significantly exceeds the loss at either endpoint. The loss barrier is formally:

```
B(θ_A, θ_B) = max_{t ∈ [0,1]} L(θ(t)) - (L(θ_A) + L(θ_B)) / 2
```

[Garipov et al. (2018), "Loss Surfaces, Mode Connectivity, and Fast Ensembling of DNNs", arXiv 1802.10026](https://arxiv.org/abs/1802.10026) demonstrated that independently trained models on the same architecture are typically *not* linearly mode connected — there are loss barriers of several nats between them. However, models that share a common pretraining checkpoint are empirically LMC: fine-tuned variants of the same pretrained model sit in the same low-loss basin, connected by near-flat paths. This is the geometric fact that makes capability arithmetic possible — fine-tuned models are close enough in weight space that linear combinations retain semantic coherence.

```python
import jax
import jax.numpy as jnp

def loss_barrier(
    loss_fn,
    theta_a: dict,
    theta_b: dict,
    n_points: int = 20,
) -> float:
    ts = jnp.linspace(0.0, 1.0, n_points)

    def interpolated_loss(t):
        theta_t = jax.tree_util.tree_map(
            lambda a, b: (1.0 - t) * a + t * b,
            theta_a, theta_b,
        )
        return loss_fn(theta_t)

    losses = jax.vmap(interpolated_loss)(ts)
    endpoint_avg = (losses[0] + losses[-1]) / 2.0
    return float(jnp.max(losses) - endpoint_avg)
```

The `jax.vmap` over `interpolated_loss` evaluates all interpolation points in a single vectorised call, making barrier estimation practical even for large models when the loss function is batched over a small calibration set.

### Neural Collapse

As training proceeds beyond the interpolation threshold, the representations of training examples in the penultimate layer of a classifier converge to a precise geometric configuration described by [Papyan, Han, and Donoho (2020), "Prevalence of Neural Collapse during the Terminal Phase of Deep Learning Training", arXiv 2008.08186](https://arxiv.org/abs/2008.08186). This phenomenon, *neural collapse*, has four properties:

1. **Within-class variability collapse**: All training examples of class k converge to a single point (the class mean μ_k) in the feature space.
2. **Convergence to equiangular tight frame**: The class mean vectors, centered at their global mean, converge to the vertices of a simplex ETF — the maximally equiangular arrangement of K vectors in ℝ^d for d ≥ K.
3. **Convergence of classifiers to class means**: The weight matrix of the final linear layer converges (up to scaling) to the matrix of class mean vectors.
4. **Simplification of decision rule**: The Bayes-optimal decision rule converges to nearest-class-mean classification.

The ETF geometry is characterised by the matrix:

```
Φ = sqrt(K / (K - 1)) · (I_K - (1/K) · 1·1^T)
```

where I_K is the K×K identity, 1 is the all-ones vector, K is the number of classes, and each column of Φ is a class prototype vector. The Gram matrix Φ^T Φ has 1 on the diagonal and -1/(K-1) on every off-diagonal entry — the maximum possible equiangularity given the simplex constraint.

Neural collapse is not an academic curiosity for capability arithmetic. It implies that the final-layer representations of a fine-tuned classifier are organised on a predictable, maximally symmetric geometric structure. Task vectors (§212.3) computed from the final layer of fine-tuned models therefore point in directions that are near-orthogonal to the ETF axes of other tasks — which explains empirically why task vectors for different tasks interfere less than random vectors would.

### Loss Landscape Topology

[Li et al. (2018), "Visualizing the Loss Landscape of Neural Nets", arXiv 1712.09913](https://arxiv.org/abs/1712.09913) introduced filter-normalised random projections to visualise the loss surface of large networks. For a network with parameters θ*, two random direction vectors δ and η are drawn and filter-normalised: for each filter (row of a weight matrix), the direction vector is scaled so that its norm matches the norm of the corresponding filter in θ*. The loss is then evaluated on the grid:

```
L(α, β) = Loss(θ* + α·δ + β·η)
```

The resulting two-dimensional surface reveals qualitative differences between architectures and training procedures: networks trained with batch normalisation and skip connections exhibit wide, smooth basins (low surface curvature), while networks without skip connections exhibit chaotic, sharp basins with many local minima. For pretrained language models, the basin around a pretrained checkpoint is empirically extremely flat — skip connections through residual streams and layer norms combine to produce a near-convex local geometry. This flatness is the geometric prerequisite for stable task vector arithmetic: operations that perturb θ by a task vector remain within the flat basin, where the loss landscape behaves approximately linearly.

The filter normalisation step in Li et al. is essential for making the visualisation scale-invariant. Without it, directions in layers with large weight norms dominate the surface because a unit step in those directions produces a large parameter change. Filter-normalising each direction vector δ by setting δ_{ij} ← δ_{ij} · ‖W_{ij}‖ / ‖δ_{ij}‖ (where W_{ij} is the j-th filter of layer i) ensures that a unit step in δ produces a parameter change of the same relative magnitude across all layers. The practical consequence for capability arithmetic is that task vectors inherit this scale-invariance property implicitly: since fine-tuned models are generated by gradient descent from the same initialisation, the update magnitudes in each layer reflect the layer's natural scale, and the filter-normalised loss surface is the correct space in which to interpret their geometry.

Three quantitative observations from Li et al. and subsequent work bear directly on the stability of weight-space arithmetic. First, the minimum sharpness of the loss surface — the smallest eigenvalue of the Hessian along any direction — is inversely correlated with the basin width as measured by filter-normalised surface curvature. Models that occupy flatter basins are more robust to weight perturbations of a fixed relative magnitude, which is precisely what task vector addition constitutes. Second, batch size during training inversely correlates with basin flatness: large-batch training finds sharper minima, and models trained with large batches exhibit higher loss barriers between task vector combinations. Third, the flatness of the basin increases with pretraining scale: models trained on more tokens from larger datasets occupy progressively flatter basins, explaining why task vector arithmetic works substantially better on frontier pretrained models than on smaller or less-trained models.

---

## 212.3 Capability Arithmetic

The observations of §212.2 — LMC between fine-tuned variants, ETF geometry in final-layer representations, flat basins around pretrained checkpoints — combine to motivate a precise algebraic structure on the space of fine-tuned models. If the loss landscape is locally flat and fine-tuned models are linearly connected to their shared pretrained initialisation, then the *displacement vectors* in weight space encode the fine-tuned capability as a first-class algebraic object that can be added, subtracted, and interpolated.

### Task Vectors

[Ilharco et al. (2023), "Editing Models with Task Arithmetic", arXiv 2212.04089](https://arxiv.org/abs/2212.04089) defines a *task vector* as the element-wise difference between a fine-tuned model and its pretrained initialisation:

```
τ = θ_ft - θ_pre
```

The edited model for that task is recovered by:

```
θ_new = θ_pre + λ·τ
```

where λ ∈ ℝ is a scaling coefficient. The key empirical finding is that task vectors support meaningful arithmetic: adding two task vectors merges their capabilities; negating a task vector suppresses the corresponding capability. A model skilled at both French translation and arithmetic can be produced by adding the French task vector and the arithmetic task vector to a pretrained base, without requiring multi-task fine-tuning.

```python
def task_vector(theta_pre: dict, theta_ft: dict) -> dict:
    return jax.tree_util.tree_map(lambda p, f: f - p, theta_pre, theta_ft)

def apply_task_vector(theta_pre: dict, tau: dict, lam: float = 1.0) -> dict:
    return jax.tree_util.tree_map(lambda p, t: p + lam * t, theta_pre, tau)

def add_task_vectors(tau_a: dict, tau_b: dict) -> dict:
    return jax.tree_util.tree_map(lambda a, b: a + b, tau_a, tau_b)

def negate_task_vector(tau: dict) -> dict:
    return jax.tree_util.tree_map(lambda t: -t, tau)
```

The scaling coefficient λ controls the strength of the applied capability. Values λ > 1 amplify the task; λ < 0 suppresses it (useful for removing an unwanted capability from a model, such as detoxification by negating a toxicity task vector).

The empirical evidence for task vector arithmetic is strong across modalities. Ilharco et al. demonstrate that on eight fine-tuned CLIP image classifiers (trained on Cars, DTD, EuroSAT, GTSRB, MNIST, RESISC45, SUN397, and SVHN), adding the task vectors for any subset of tasks and applying the combined vector to the pretrained ViT-B/32 backbone recovers 90–97% of the accuracy achievable by fine-tuning on all tasks jointly. The task vector for negation is equally striking: applying a negated Cars classifier task vector to a Cars-fine-tuned model degrades Cars accuracy from 77% to 19% while leaving MNIST accuracy (79% → 79%) nearly unaffected — demonstrating that the negated vector selectively removes a capability with minimal collateral damage.

For language models, the analogous result holds for instruction-following and safety fine-tuning. A safety fine-tuned model's refusal behaviour can be approximately removed by negating the safety task vector and applying it to the safety-trained model, recovering the pretrained model's compliance to adversarial prompts. This makes safety-relevant weight-space operations an active research concern: the same algebra that enables capability composition also enables capability subtraction, and the security implications are non-trivial.

The interference between task vectors for different tasks is quantified by the cosine similarity between τ_A and τ_B: tasks with low pairwise cosine similarity combine with minimal interference. Ilharco et al. report that pairwise cosine similarities between the eight CLIP task vectors are all below 0.1, confirming the near-orthogonality predicted by the ETF geometry of neural collapse (§212.2). For language model task vectors, interference is higher — language tasks share more weight structure than vision tasks — which motivates the more sophisticated TIES and DARE merging strategies.

### TIES Merging

Task vector addition fails when multiple fine-tuned models have been trained independently (not merely differently fine-tuned from the same checkpoint) or when the task vectors point in conflicting directions for a substantial fraction of parameters. [Yadav et al. (2023), "TIES-Merging: Resolving Interference When Merging Models", arXiv 2306.01708](https://arxiv.org/abs/2306.01708) identifies *interference* as the primary failure mode and proposes a three-step algorithm:

**Step 1 — Trim**: For each task vector τ^(m), retain only the top-k% of parameters by absolute magnitude; zero out the rest. This removes small, potentially noisy deltas that contribute interference without contributing signal.

**Step 2 — Elect**: For each parameter position i, compute the sign voted by the majority of the trimmed task vectors that are non-zero at position i. Formally, for n task vectors:

```
γ_i = sign(Σ_m sign(τ^(m)_i))
```

where the sum is over tasks m that have non-zero values at position i after trimming.

**Step 3 — Disjoint merge**: For each parameter position i, average only the task vectors whose sign agrees with γ_i:

```
τ^merged_i = (1/|A_i|) Σ_{m ∈ A_i} τ^(m)_i
where A_i = {m : sign(τ^(m)_i) = γ_i and τ^(m)_i ≠ 0}
```

The final merged model is `θ_pre + λ·τ^merged`.

The three steps address three distinct sources of interference that naive task vector addition suffers from. Trimming addresses magnitude-based noise: the distribution of delta parameter magnitudes is heavy-tailed, with many near-zero parameters that are artifacts of gradient noise rather than learned capability. Retaining only the top-k% by magnitude removes these noise terms. Sign election addresses directional interference: when two task vectors have opposite signs at a parameter position, their naive average is near zero — neither capability is preserved. Electing a dominant sign ensures that one capability's direction wins rather than producing destructive cancellation. Disjoint merge addresses asymmetric contribution: by averaging only the sign-concordant parameters, the merge gives equal weight to all tasks that agree on a direction, without penalising tasks that have zero values at that position after trimming.

```python
def ties_merge(
    theta_pre: dict,
    task_vectors: list[dict],
    trim_fraction: float = 0.8,
    lam: float = 1.0,
) -> dict:
    def merge_param(*deltas):
        stacked = jnp.stack(deltas)              # shape: (n_tasks, *param_shape)
        n_tasks = stacked.shape[0]
        flat = jnp.abs(stacked).reshape(n_tasks, -1)
        thresholds = jnp.quantile(flat, trim_fraction, axis=1)  # per-task threshold
        thresh_bc = thresholds.reshape((n_tasks,) + (1,) * (stacked.ndim - 1))
        trimmed = jnp.where(jnp.abs(stacked) >= thresh_bc, stacked, 0.0)

        sign_sum = jnp.sum(jnp.sign(trimmed), axis=0)
        elected_sign = jnp.sign(sign_sum)

        mask = (jnp.sign(trimmed) == elected_sign[None, ...]) & (trimmed != 0.0)
        n_agree = jnp.sum(mask, axis=0).clip(min=1)
        merged = jnp.sum(jnp.where(mask, trimmed, 0.0), axis=0) / n_agree
        return merged

    merged_tau = jax.tree_util.tree_map(merge_param, *task_vectors)
    return jax.tree_util.tree_map(
        lambda p, t: p + lam * t, theta_pre, merged_tau
    )
```

### DARE: Drop-and-Rescale Delta Parameters

[Yu et al. (2023), "Language Models are Super Mario: Absorbing Abilities from Homologous Models as a Free Lunch", arXiv 2311.03099](https://arxiv.org/abs/2311.03099) introduces DARE (Drop And REscale), a pre-processing step that applies random pruning to task vectors before merging. The observation motivating DARE is that most delta parameters in fine-tuned language models are redundant: randomly pruning a high fraction (85–99%) and rescaling the survivors by 1/(1-p) produces task vectors that preserve performance nearly identically to the full task vector, while drastically reducing interference with other task vectors during merging.

The procedure for a single task vector τ with drop probability p:

```
m ~ Bernoulli(1-p)  (iid per parameter)
τ_DARE = m ⊙ τ / (1 - p)
```

The rescaling by 1/(1-p) makes the operation an unbiased estimator of the original task vector in expectation, analogous to dropout during training.

```python
def dare(
    tau: dict,
    drop_prob: float = 0.9,
    rng_key: jax.Array = None,
) -> dict:
    if rng_key is None:
        rng_key = jax.random.PRNGKey(0)
    leaves, treedef = jax.tree_util.tree_flatten(tau)
    keys = jax.random.split(rng_key, len(leaves))

    def drop_rescale(leaf, key):
        mask = jax.random.bernoulli(key, 1.0 - drop_prob, shape=leaf.shape)
        return mask * leaf / (1.0 - drop_prob)

    new_leaves = [drop_rescale(l, k) for l, k in zip(leaves, keys)]
    return jax.tree_util.tree_unflatten(treedef, new_leaves)
```

DARE is typically applied before TIES: each task vector is DARE-pruned independently, then the pruned vectors are merged with TIES. This two-stage pipeline achieves better separation between tasks than either method alone.

### SLERP: Spherical Linear Interpolation

Linear interpolation in weight space, when applied to task vectors of different magnitudes, can change the norm of the interpolant in ways that distort the result — a task vector halfway between a large fine-tune and a small fine-tune ends up closer in character to the large fine-tune than the actual midpoint. Spherical linear interpolation (SLERP) corrects this by interpolating along the geodesic of the unit sphere in each parameter subspace, preserving direction throughout:

```
θ(t) = sin((1-t)·Ω) / sin(Ω) · θ_A  +  sin(t·Ω) / sin(Ω) · θ_B
where Ω = arccos( <θ_A, θ_B> / (‖θ_A‖ · ‖θ_B‖) )
```

For PyTree-structured weight dictionaries, SLERP must be applied independently to each leaf tensor (flattening the tensor to a vector, computing the angle, interpolating, and reshaping). When Ω ≈ 0 (the two vectors are nearly parallel), SLERP degenerates to LERP — a numerical guard is necessary.

```python
def slerp_leaf(a: jnp.ndarray, b: jnp.ndarray, t: float) -> jnp.ndarray:
    flat_a = a.ravel().astype(jnp.float32)
    flat_b = b.ravel().astype(jnp.float32)

    norm_a = jnp.linalg.norm(flat_a)
    norm_b = jnp.linalg.norm(flat_b)

    safe_a = flat_a / jnp.where(norm_a > 1e-8, norm_a, 1.0)
    safe_b = flat_b / jnp.where(norm_b > 1e-8, norm_b, 1.0)

    cos_omega = jnp.clip(jnp.dot(safe_a, safe_b), -1.0, 1.0)
    omega = jnp.arccos(cos_omega)

    nearly_parallel = omega < 1e-4
    lerp_result = (1.0 - t) * flat_a + t * flat_b
    coeff_a = jnp.sin((1.0 - t) * omega) / jnp.where(nearly_parallel, 1.0, jnp.sin(omega))
    coeff_b = jnp.sin(t * omega) / jnp.where(nearly_parallel, 1.0, jnp.sin(omega))
    slerp_result = coeff_a * flat_a + coeff_b * flat_b

    result = jnp.where(nearly_parallel, lerp_result, slerp_result)
    return result.reshape(a.shape).astype(a.dtype)

def slerp(theta_a: dict, theta_b: dict, t: float) -> dict:
    return jax.tree_util.tree_map(
        lambda a, b: slerp_leaf(a, b, t),
        theta_a, theta_b,
    )
```

SLERP is the method of choice for model interpolation tasks where preserving the total magnitude of fine-tuned capabilities matters — for instance, when blending two instruct-tuned models with different capability profiles while keeping the aggregate instruction-following strength stable.

The choice between LERP and SLERP depends on the magnitude relationship between the two weight vectors being interpolated. When ‖θ_A‖ ≈ ‖θ_B‖ — as is typically the case for two models fine-tuned with the same learning rate schedule from the same checkpoint — LERP and SLERP produce nearly identical results because the linear interpolant stays on a nearly-spherical path. When the magnitudes differ substantially — as occurs when one model has been fine-tuned for many more steps, or with a larger learning rate, than the other — LERP produces a midpoint whose norm is approximately the arithmetic mean of the two norms, weighted by t. This biases the interpolant toward the larger-magnitude model. SLERP normalises both vectors to the unit sphere before interpolating, then rescales the result — but the rescaling factor (interpolated from the two norms) must be chosen carefully; a common implementation uses LERP on the norms alongside SLERP on the directions.

For task vector arithmetic as opposed to direct model interpolation, SLERP applies to the task vector tensors themselves rather than to the full parameter vectors. SLERP between two task vectors τ_A and τ_B at t=0.5 finds the geodesic midpoint of the two capability directions — a merged capability that draws equally from both fine-tunes while preserving the aggregate capability magnitude. This is the operation that has driven the commercial "model merging" ecosystem, where practitioners combine fine-tuned models from community repositories to produce hybrid capabilities not available in any individual fine-tune.

A systematic comparison of these and further merging strategies — including gradient-based merging, Fisher-weighted averaging, and evolutionary search over merge configurations — appears in [Goddard et al. (2024), "Arcee's MergeKit: A Toolkit for Merging Large Language Models", arXiv 2403.13257](https://arxiv.org/abs/2403.13257) and the survey [Wang et al. (2024), "Model Merging in LLMs, MLLMs, and Beyond: Methods, Theories, Applications and Opportunities", arXiv 2408.07666](https://arxiv.org/abs/2408.07666). The survey catalogues over 50 distinct merging methods published between 2020 and 2024, partitioned into three families: basic interpolation (LERP, SLERP), task-vector-based methods (Task Arithmetic, TIES, DARE, and combinations thereof), and alignment-based methods that use Fisher information or gradient matching to identify which parameters are most important for each task before merging.

---

## 212.4 MLIR/TOSA and XLA HLO as Weight IR

The preceding sections treat weight tensors as mathematical objects — vectors in ℝ^P. For compiler practitioners, the natural question is: what IR best represents the computation *over* those weight tensors, and can that IR be used to reason about capability arithmetic at the operation level rather than the parameter level?

### TOSA: Tensor Operator Set Architecture

The TOSA dialect in MLIR was designed by Arm to provide a stable, backend-neutral IR for tensor operations drawn from the neural network domain. Unlike the `linalg` dialect, which expresses tensor operations through generic region-based iteration, TOSA uses a fixed set of named operations — each with a precise mathematical specification — making it suitable as a target for frontend compilers (torch-mlir, onnx-mlir) and a source for backend lowering (to `linalg`, `vector`, and then `llvm`).

The two weight-heavy operations in TOSA are:

```mlir
// Matrix multiply: C[b, m, k] = A[b, m, n] * B[b, n, k]
// In LLVM 22 TOSA, a_zp and b_zp are required zero-point operands (0 for float)
%zero_zp = "tosa.const"() {value = dense<0> : tensor<i32>} : () -> tensor<i32>
%result = tosa.matmul %activation, %weight, %zero_zp, %zero_zp
    : (tensor<2x8x4096xf32>, tensor<2x4096x4096xf32>, tensor<i32>, tensor<i32>)
    -> tensor<2x8x4096xf32>

// Convolution: O[n, oh, ow, oc] = Σ_{kh,kw,ic} I[n, oh+kh, ow+kw, ic] * W[oc, kh, kw, ic]
// LLVM 22 TOSA requires input_zp, weight_zp operands and acc_type attribute
%conv_out = tosa.conv2d %input, %filter, %bias, %zero_zp, %zero_zp {
    pad      = array<i64: 1, 1, 1, 1>,
    stride   = array<i64: 1, 1>,
    dilation = array<i64: 1, 1>,
    acc_type = f32
} : (tensor<1x224x224x3xf32>, tensor<64x3x3x3xf32>, tensor<64xf32>,
     tensor<i32>, tensor<i32>)
    -> tensor<1x224x224x64xf32>
```

TOSA enables weight-tensor reasoning at the operation level. An MLIR analysis pass over a TOSA module can enumerate all `tosa.matmul` operations and their associated weight tensors, inspect their shapes, and infer the parameter count per layer — information used in structured pruning and layer-selective task vector application (§212.5). The `tosa.variable` and `tosa.variable_read`/`tosa.variable_write` operations provide mutable weight semantics within a TOSA computation, making in-graph weight updates expressible as IR transformations rather than external mutation.

The lowering chain from TOSA to executable code proceeds through MLIR's transformation passes: TOSA → `linalg` named ops (via `--tosa-to-linalg-named`) → `linalg` on tensors (via tiling and bufferisation) → `vector` dialect → LLVM dialect → LLVM IR → PTX/native code. This chain is invoked by IREE and torch-mlir for production inference and is directly inspectable with `mlir-opt`.

```bash
# Inspect TOSA lowering for a matmul
mlir-opt input.mlir \
  --tosa-to-linalg-named \
  --linalg-bufferize \
  --convert-linalg-to-loops \
  --lower-affine \
  --convert-scf-to-cf \
  --convert-cf-to-llvm \
  --convert-func-to-llvm \
  -o lowered.mlir
```

### XLA HLO as a Homoiconic Forward-Pass Graph

As established in [Chapter 211 — Neural Programs as Compiled Artifacts](ch211-neural-programs-compiled-artifacts.md), JAX traces the forward pass of a neural network into a `jaxpr` and lowers it to StableHLO (a stable, versioned subset of XLA HLO). The HLO computation graph is homoiconic in the strongest sense: it is a data structure in Python that *represents* the forward pass as a graph of operations, and can be manipulated as a graph using the Python mlir bindings.

Weight tensors appear in an HLO module as `constant` operations at the module level or as `parameter` operations in the computation's parameter list. A task vector application in HLO corresponds to inserting an `add` operation between the parameter and the fine-tuned constant. Capability arithmetic at the HLO level therefore becomes graph surgery: the HLO module for a pretrained model can be modified by replacing parameter constants or adding elementwise operations to implement scaling, addition, or sign election without retracing the Python model.

```python
import jax
import jax.numpy as jnp

def forward(params, x):
    return jnp.dot(x, params["W"]) + params["b"]

x = jnp.ones((4, 8))
params = {"W": jnp.ones((8, 4)), "b": jnp.zeros(4)}

# Extract the HLO computation as a first-class Python object
lowered = jax.jit(forward).lower(params, x)
stablehlo_text = lowered.as_text()

# The StableHLO module can be further compiled or serialized
compiled = lowered.compile()
```

The connection between HLO and capability arithmetic is developed further in [Chapter 154 — HLO and StableHLO](../part-22-xla-openxla/ch154-hlo-and-stablehlo.md). For GPU execution, the HLO module is lowered through the CUDA Tile IR layer described in [Chapter 164 — CUDA Tile IR](../part-23-mlir-production/ch164-cuda-tile-ir.md), ultimately producing PTX that executes the weight tensor operations on device.

---

## 212.5 Programmatic Weight-Space Operations via JAX

The JAX ecosystem provides the primary programmatic interface for weight-space operations in practice. Its functional design — where model parameters are explicit data structures rather than implicit object state — makes parameter manipulation a first-class operation, not a workaround.

### `jax.tree_util.tree_map` for Element-Wise PyTree Operations

JAX's PyTree abstraction treats nested Python containers (dicts, lists, NamedTuples, Flax `FrozenDict`s) as trees of leaves, where the leaves are JAX arrays. `jax.tree_util.tree_map` applies a function element-wise across corresponding leaves of one or more PyTrees of the same structure. All the capability arithmetic operations in §212.3 are expressed as `tree_map` calls because the PyTree structure exactly mirrors the weight dictionary structure of a Flax model.

```python
import jax
import jax.numpy as jnp
import optax

def param_stats(params: dict) -> dict:
    return jax.tree_util.tree_map(
        lambda p: {"mean": jnp.mean(p), "std": jnp.std(p), "norm": jnp.linalg.norm(p)},
        params,
    )

def clip_task_vector(tau: dict, max_norm: float) -> dict:
    leaves, treedef = jax.tree_util.tree_flatten(tau)
    total_norm = jnp.sqrt(sum(jnp.sum(l**2) for l in leaves))
    scale = jnp.minimum(1.0, max_norm / (total_norm + 1e-8))
    clipped = [l * scale for l in leaves]
    return jax.tree_util.tree_unflatten(treedef, clipped)
```

The `tree_flatten`/`tree_unflatten` pair exposes the leaves as a flat Python list for operations that need global statistics (such as computing a global gradient norm for clipping), then re-assembles them into the original PyTree structure.

### `flax.traverse_util` for Parameter Path Filtering

Task vectors applied uniformly to all parameters of a large language model are effective, but the interpretability evidence from [Chapter 213 — Mechanistic Interpretability Infrastructure](ch213-mechanistic-interpretability.md) suggests that different architectural components encode different types of information: attention heads implement in-context retrieval and pattern matching, while MLP layers implement fact storage and feature transformation. Layer-selective task vector application — applying a task vector only to attention parameters — can therefore transfer an in-context learning capability without disturbing the factual knowledge stored in MLP weights.

`flax.traverse_util.ModelParamTraversal` provides path-aware filtering over Flax parameter trees:

```python
from flax import traverse_util

def attention_only_task_vector(
    theta_pre: dict,
    theta_ft: dict,
    lam: float = 1.0,
) -> dict:
    tau = jax.tree_util.tree_map(lambda p, f: f - p, theta_pre, theta_ft)

    flat = traverse_util.flatten_dict(tau, sep="/")
    filtered = {
        k: v for k, v in flat.items()
        if "attention" in k or "self_attn" in k or "q_proj" in k
        or "k_proj" in k or "v_proj" in k or "out_proj" in k
    }

    zeros = jax.tree_util.tree_map(jnp.zeros_like, tau)
    flat_zeros = traverse_util.flatten_dict(zeros, sep="/")
    for k in filtered:
        flat_zeros[k] = filtered[k] * lam

    return traverse_util.unflatten_dict(flat_zeros, sep="/")
```

### `tree_map_with_path` for Layer-Wise Operations

JAX 0.4.14 introduced `jax.tree_util.tree_map_with_path`, which passes the path to each leaf as the first argument, enabling depth-dependent and name-dependent operations without the explicit flatten/filter pattern:

```python
def layer_scaled_task_vector(
    theta_pre: dict,
    theta_ft: dict,
    n_layers: int,
    decay: float = 0.9,
) -> dict:
    tau = jax.tree_util.tree_map(lambda p, f: f - p, theta_pre, theta_ft)

    def scale_by_depth(path, leaf):
        path_str = "/".join(str(p.key) for p in path)
        for layer_idx in range(n_layers):
            if f"layers_{layer_idx}" in path_str or f"layers.{layer_idx}" in path_str:
                scale = decay ** layer_idx
                return leaf * scale
        return leaf

    return jax.tree_util.tree_map_with_path(scale_by_depth, tau)
```

This applies exponential depth decay to the task vector, down-weighting the deltas in later layers relative to earlier layers — a heuristic motivated by the finding that later transformer layers are more task-specific and may require smaller perturbations for stable capability transfer.

### Practical Example: Applying an Arithmetic Task Vector to Attention Only

The following end-to-end example constructs an arithmetic task vector from a hypothetical arithmetic fine-tuned model, applies it only to attention projection weights, and computes the loss barrier to verify that the merged model remains in the same loss basin:

```python
import jax
import jax.numpy as jnp
from flax import traverse_util

def selective_merge(
    theta_pre: dict,
    theta_arithmetic_ft: dict,
    eval_loss_fn,
    lam: float = 0.7,
    attn_keys: tuple[str, ...] = ("q_proj", "k_proj", "v_proj", "out_proj"),
) -> dict:
    tau = jax.tree_util.tree_map(
        lambda p, f: f - p, theta_pre, theta_arithmetic_ft
    )

    flat_pre = traverse_util.flatten_dict(theta_pre, sep="/")
    flat_tau = traverse_util.flatten_dict(tau, sep="/")

    merged_flat = {}
    for k, v in flat_pre.items():
        if any(attn_key in k for attn_key in attn_keys):
            merged_flat[k] = v + lam * flat_tau[k]
        else:
            merged_flat[k] = v

    theta_merged = traverse_util.unflatten_dict(merged_flat, sep="/")

    barrier = loss_barrier(eval_loss_fn, theta_pre, theta_merged, n_points=10)
    print(f"Loss barrier (pre → merged): {barrier:.4f} nats")

    return theta_merged
```

This pattern generalises straightforwardly to multi-task merging: compute one task vector per fine-tune, apply DARE to each, combine with TIES, and then selectively apply the merged task vector to the architectural components most relevant to the merged tasks.

---

## 212.6 Cognitive Angle: The Algebra of Capability

The technical machinery of §212.3–212.5 admits an interpretation that is not merely convenient metaphor but a precise cognitive science claim. If weight-space arithmetic can compose fine-tuned capabilities linearly, then those capabilities have an algebraic structure — they form a module over the reals, closed under the operations of addition, scalar multiplication, and negation. Adding the French translation task vector to an arithmetic reasoning task vector and recovering a model that does both is equivalent to demonstrating that "French translation capability" and "arithmetic reasoning capability" are linearly independent vectors in the capability module, at least approximately.

The approximation holds to the degree that the LMC condition holds and the loss landscape is locally flat. The empirical record from task arithmetic and TIES suggests that this linear model is accurate to within a few percentage points on benchmark tasks when the task vectors are derived from fine-tunes of the same pretrained checkpoint — a condition that is universally satisfied in the modern practice of fine-tuning from a large pretrained base.

The structure is made precise by the following observation: a pretrained model defines an *origin* in weight space, and fine-tuning defines a *displacement* from that origin. The set of all displacements reachable by fine-tuning on tasks from a given distribution forms a convex body in weight space — empirically near-ellipsoidal, with principal axes corresponding to the major capability directions. Task vectors are the projections of this body onto its principal axes. SLERP navigates the surface of this body; TIES selects a weighted average of the principal components that agree in sign; DARE sparsifies the projection without biasing its direction. The entire algebra of capability merging is therefore an algebra of projections and combinations within this fine-tuning cone.

### The Linearity Assumption and Its Limits

The linear model fails in two regimes. First, when the fine-tuning tasks are *contradictory* — when encoding capability A requires encoding the *negation* of some feature that capability B requires — adding the task vectors produces destructive interference that cannot be resolved by sign election alone. An example is the tension between verbosity (task vectors that increase generation length) and conciseness (task vectors that decrease it): these are approximately antipodal in the relevant weight subspace, and their sum cancels the capability of both. In this regime, the correct approach is conditional routing — activating different task vectors depending on the inference context — rather than static merging.

Second, when the pretrained model is *not* locally flat at the operating point of the fine-tune — as occurs for very large fine-tuning learning rates, many fine-tuning steps, or fine-tunes that change the model's data distribution substantially — the LMC assumption breaks down and the task vector leaves the flat basin. The loss barrier `B(θ_pre, θ_pre + τ)` can be measured directly using the `loss_barrier` function in §212.2 as a diagnostic; values above 0.5 nats suggest that the fine-tune has escaped the flat basin and that arithmetic operations on its task vector will not reliably preserve capabilities.

### Capability Arithmetic as the Dual of Mechanistic Interpretability

What the capability algebra does not capture is the non-linear interaction structure inside individual attention heads and MLP sublayers. Adding two task vectors produces a model that behaves as if it had been multi-task fine-tuned, but the internal mechanism by which it does so — the specific attention head circuits, the specific MLP feature directions — may differ from what multi-task training would produce. The capability is present at the behavioural level; whether it is implemented by the same circuit as a jointly trained model is an open question that requires interpretability tools to answer.

[Chapter 213 — Mechanistic Interpretability Infrastructure](ch213-mechanistic-interpretability.md) develops the tools for probing this internal structure: sparse autoencoders that recover monosemantic feature directions from polysemantic neuron activations, and circuit analysis that identifies which components are causally responsible for a given merged capability. The interpretability view and the weight-arithmetic view are complementary in a precise sense: capability arithmetic is the *forward* map from weight-space operations to behavioural outcomes; mechanistic interpretability is the *inverse* map from behavioural outcomes to weight-space mechanisms. A complete theory of neural network cognition requires both.

The connection to SAE features from Chapter 213 is direct: if a sparse autoencoder has been trained on the activations of a pretrained model, its feature directions form a dictionary in which each direction corresponds to an interpretable semantic concept. A task vector expressed in this dictionary basis — by projecting τ onto the SAE feature directions — reveals which semantic features the fine-tune has strengthened or weakened, at a resolution finer than parameter level. This cross-analysis of weight-space arithmetic with activation-space interpretability is an active research direction that does not yet have a canonical methodology, but the tools of both disciplines are available and composable in the JAX ecosystem.

### Self-Modification as Weight-Space Navigation

[Chapter 214 — Gradient-Based Self-Modification](ch214-gradient-based-self-modification.md) closes the loop by treating gradient descent itself as a programmatic weight-space operation — one that, unlike task vector arithmetic, is guided by a differentiable objective and therefore can navigate the curvature of the loss landscape rather than relying on the flatness assumption. The relationship between capability arithmetic and gradient-based modification is analogous to the relationship between binary patching and source recompilation: patching is fast and coarse; recompilation is slow but precise. Both are valid tools in a system that has access to both the binary (the weights) and the compiler (the JAX/XLA toolchain).

The distinction matters operationally: task vector arithmetic executes in time proportional to the number of parameters (a single pass of elementwise addition), while gradient-based modification requires forward and backward passes over training data — orders of magnitude more compute. For a system that must modify its own capabilities at inference time, task vector arithmetic is the only computationally feasible approach. For a system with access to a training loop, gradient-based modification is more precise and can target objectives that task vectors cannot express. The two approaches are not alternatives but complements: task vectors provide a fast, coarse capability steering mechanism; gradient descent provides slow, precise capability engineering. A self-modifying system uses task vectors for rapid adaptation and gradient-based methods for sustained capability development.

The cognitive framing — capabilities as vectors, understanding as geometry, modification as algebra — is not a post-hoc rationalisation. It is the mathematical structure that the empirical evidence of LMC, neural collapse, and task arithmetic jointly enforce. A system that treats its own weights as a programming substrate is not using a metaphor: it is using the most accurate available description of what its parameters are, and the most productive framework for engineering its own cognitive development.

---

## Chapter Summary

- **SafeTensors** stores weight tensors as an 8-byte LE uint64 header-length prefix, a UTF-8 JSON tensor-descriptor header (with `data_offsets` relative to the data region, not the file start), and raw dtype-packed tensor bytes — no magic identifier. Memory-mapped loading is zero-copy and backend-agnostic.

- **GGUF** provides a 4-byte magic `GGUF` (0x47475546), a versioned binary KV metadata store, and a tensor table supporting blocked quantization types (Q4_0, Q4_K, Q8_0, etc.) required for consumer-hardware inference.

- **Linear mode connectivity** (Garipov et al., arXiv 1802.10026) holds between fine-tuned variants of the same pretrained checkpoint: they occupy the same flat loss basin, connected by near-flat paths. This is the geometric prerequisite for weight-space arithmetic.

- **Neural collapse** (Papyan et al., arXiv 2008.08186) organises final-layer representations into a simplex ETF geometry with Gram matrix entries -1/(K-1) off-diagonal, providing near-orthogonality between class directions that reduces inter-task interference in task vector addition.

- **Task vectors** (Ilharco et al., arXiv 2212.04089) are the weight differences τ = θ_ft - θ_pre. They form a module over ℝ, closed under addition, scalar multiplication, and negation; these operations transfer, amplify, and suppress capabilities.

- **TIES merging** (Yadav et al., arXiv 2306.01708) resolves interference through three steps: trim small deltas by magnitude, elect a dominant sign by majority vote per parameter, and merge only the sign-concordant deltas. This outperforms naive task vector addition for independently fine-tuned models.

- **DARE** (Yu et al., arXiv 2311.03099) pre-processes task vectors by randomly dropping a fraction (85–99%) of delta parameters and rescaling survivors by 1/(1-p), producing unbiased sparse approximations that reduce inter-task interference when combined with TIES.

- **SLERP** interpolates weight vectors along geodesics of the unit sphere, preserving total magnitude throughout the interpolation and avoiding the norm distortion of linear interpolation when blending models of different magnitudes.

- **TOSA dialect** provides structured MLIR operations (`tosa.matmul`, `tosa.conv2d`) whose weight tensor operands are inspectable by analysis passes, enabling layer-selective task vector application based on operation type and shape.

- **XLA HLO** is a homoiconic forward-pass graph: weight constants and parameters appear as named entities in the HLO module and can be surgically replaced or augmented to implement capability arithmetic at the graph level.

- **`jax.tree_util.tree_map`** is the universal interface for weight-space operations, with `flax.traverse_util` enabling path-aware filtering for architectural-component-selective operations and `tree_map_with_path` providing depth-dependent scaling.

- The capability algebra is not metaphorical: LMC, neural collapse, and task arithmetic jointly establish that fine-tuned capabilities are approximately linearly independent vectors in a high-dimensional parameter module, making addition, negation, and interpolation semantically meaningful operations on cognitive function.
