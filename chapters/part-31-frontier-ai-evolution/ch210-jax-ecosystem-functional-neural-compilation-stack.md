# Chapter 210 — The JAX Ecosystem: A Functional Neural Compilation Stack

*Part XXXI — Frontier AI Evolution*

Modern machine learning systems face a foundational tension: deep learning frameworks must be simultaneously expressive enough for research experimentation and efficient enough for production deployment at scale. The JAX ecosystem resolves this tension by reconceptualising the ML stack as a **functional compiler pipeline** rather than an imperative runtime. JAX's transforms — `jit`, `grad`, `vmap`, `pmap`, `scan` — are composable compiler passes that operate on pure Python functions. Every library in the ecosystem inherits this property: Flax NNX models are JAX-compatible PyTrees; Optax optimizers are gradient transformations that compose like functions; NumPyro programs are ordinary Python functions whose execution is *interpreted* by different effect handlers. The compilation path is concrete: JAX traces a Python function to a `jaxpr` IR, lowers to StableHLO, passes through MLIR's Linalg and Tensor dialects, and reaches native LLVM IR for CPU, PTX for GPU, or XLA HLO for TPU. This chapter maps the entire ecosystem with precision — not as a catalogue, but as a coherent functional compilation stack where each library occupies a well-defined semantic position.

Cross-references: [Chapter 153 — XLA Architecture](../part-22-xla-openxla/ch153-xla-architecture.md) · [Chapter 154 — HLO and StableHLO](../part-22-xla-openxla/ch154-hlo-and-stablehlo.md) · [Chapter 158 — SPMD, GSPMD, and Auto-Sharding](../part-22-xla-openxla/ch158-spmd-gspmd-and-auto-sharding.md) · [Chapter 161 — torch-mlir, ONNX-MLIR, and JAX/TF Bridges](../part-23-mlir-production/ch161-torch-mlir-onnx-mlir-and-jax-tf-bridges.md) · [Chapter 207 — Reflective Code, Open Problems, and Build Roadmap](../part-30-AI-first-PL-design/ch207-reflective-code-open-problems-roadmap.md) · [Chapter 211 — Neural Programs as Compiled Artifacts](ch211-neural-programs-as-compiled-artifacts.md) · [Chapter 214 — Gradient-Based Self-Modification](ch214-gradient-based-self-modification.md)

---

## Table of Contents

- [210.1 JAX: Composable Functional Transforms as a Compiler](#2101-jax-composable-functional-transforms-as-a-compiler)
  - [Tracing Semantics and Abstract Evaluation](#tracing-semantics-and-abstract-evaluation)
  - [The Transform Composition Model](#the-transform-composition-model)
  - [scan: Structured Recursion](#scan-structured-recursion)
  - [pmap: SPMD Data Parallelism](#pmap-spmd-data-parallelism)
  - [The Compilation Path](#the-compilation-path)
  - [Custom PRNG and Functional Randomness](#custom-prng-and-functional-randomness)
- [210.2 Flax NNX: Mutable Stateful Modules](#2102-flax-nnx-mutable-stateful-modules)
  - [Module Design](#module-design)
  - [State Extraction and Reinjection](#state-extraction-and-reinjection)
  - [Variable Types and Metadata](#variable-types-and-metadata)
  - [Evaluation Mode and Model Introspection](#evaluation-mode-and-model-introspection)
- [210.3 Optax: A Composable Optimizer DSL](#2103-optax-a-composable-optimizer-dsl)
  - [Core Transforms and Composition](#core-transforms-and-composition)
  - [Schedule Composition](#schedule-composition)
  - [Custom Transforms: Gradient Surgery](#custom-transforms-gradient-surgery)
  - [Masked Updates and Parameter-Specific Schedules](#masked-updates-and-parameter-specific-schedules)
- [210.4 Equinox: PyTree-Native Neural Networks](#2104-equinox-pytree-native-neural-networks)
  - [Models as PyTrees](#models-as-pytrees)
  - [filter_jit and filter_grad](#filterjit-and-filtergrad)
  - [PyTree Filtering and Programmatic Self-Modification](#pytree-filtering-and-programmatic-self-modification)
  - [Equinox and Custom Layers](#equinox-and-custom-layers)
- [210.5 Haiku: DeepMind's Functional Modules](#2105-haiku-deepminds-functional-modules)
  - [The transform/apply Split](#the-transformapply-split)
  - [Stateful Transforms](#stateful-transforms)
  - [Haiku vs Flax NNX: A Design Comparison](#haiku-vs-flax-nnx-a-design-comparison)
  - [hk.next_rng_key and Automatic Key Splitting](#hknextrngkey-and-automatic-key-splitting)
  - [Lifting Haiku into Standard JAX Transforms](#lifting-haiku-into-standard-jax-transforms)
- [210.6 NumPyro: Probabilistic Programming on JAX](#2106-numpyro-probabilistic-programming-on-jax)
  - [Probabilistic Programs as Functions](#probabilistic-programs-as-functions)
  - [Inference Algorithms as Effect Handlers](#inference-algorithms-as-effect-handlers)
  - [Latent Variable Models and Amortised Inference](#latent-variable-models-and-amortised-inference)
  - [The Algebraic Effect Connection](#the-algebraic-effect-connection)
  - [Cognitive Angle: Uncertainty Over Model Architectures](#cognitive-angle-uncertainty-over-model-architectures)
- [210.7 BlackJAX: Composable MCMC Samplers](#2107-blackjax-composable-mcmc-samplers)
  - [The init/step Interface](#the-initstep-interface)
  - [Parallel Chains via vmap](#parallel-chains-via-vmap)
  - [Stochastic Gradient Langevin Dynamics](#stochastic-gradient-langevin-dynamics)
  - [Step-Size Adaptation](#step-size-adaptation)
- [210.8 Distrax and Orbax](#2108-distrax-and-orbax)
  - [Distrax: JAX-Native Probability Distributions](#distrax-jax-native-probability-distributions)
  - [Orbax: Async Checkpointing for JAX Models](#orbax-async-checkpointing-for-jax-models)
  - [Cross-Framework Checkpoint Compatibility](#cross-framework-checkpoint-compatibility)
- [210.9 Brax: Differentiable Physics as Cognitive Substrate](#2109-brax-differentiable-physics-as-cognitive-substrate)
  - [Massively Parallel Simulation](#massively-parallel-simulation)
  - [Differentiable RL and Cognitive Self-Improvement](#differentiable-rl-and-cognitive-self-improvement)
  - [Brax and PPO: A Fully Compiled RL Pipeline](#brax-and-ppo-a-fully-compiled-rl-pipeline)
- [210.10 The Ecosystem as a Functional Compiler Stack](#21010-the-ecosystem-as-a-functional-compiler-stack)
  - [The Interoperability Architecture](#the-interoperability-architecture)
  - [MAML: grad(grad(...)) for Meta-Learning](#maml-gradgrad-for-meta-learning)
  - [vmap for Ensemble Cognition](#vmap-for-ensemble-cognition)
  - [The jaxpr as Homoiconic IR](#the-jaxpr-as-homoiconic-ir)
  - [Limits and Open Problems](#limits-and-open-problems)
- [Chapter Summary](#chapter-summary)

---

## 210.1 JAX: Composable Functional Transforms as a Compiler

JAX's central insight is simple but radical: **mathematical transformations of functions should themselves be functions**. `grad(f)` is not a special operation on a computation graph — it is a higher-order function that accepts a Python callable and returns a new Python callable representing the derivative. The same holds for `jit`, `vmap`, `pmap`, and `scan`. Because all transforms accept and return callables with the same interface, they compose in any order.

### Tracing Semantics and Abstract Evaluation

When `jit(f)(x)` executes for the first time, JAX does not run `f` eagerly. Instead it **traces** `f` by passing abstract `ShapedArray` values through it — values that carry dtype and shape but no concrete data. Every JAX primitive encountered during tracing is recorded. The result is a `jaxpr` (JAX expression): a typed, functional, SSA-like intermediate representation.

This tracing model has important consequences. Python control flow that depends on concrete values is **frozen into the trace** at JIT-time. A branch `if x > 0` inside `jit` must have `x` available as a concrete value when tracing occurs, or must be expressed using `jax.lax.cond` so both branches are traced and the selection is a JAX primitive. This is the discipline that enables the jaxpr to be a **closed, first-order** IR: there are no Python callbacks, no dynamic dispatch, no allocation of Python objects at runtime after compilation.

```python
# Wrong: Python-level branch frozen at trace time
def bad_relu(x):
    if x > 0:     # x is abstract at trace time — JAX raises ConcretizationTypeError
        return x
    return 0.0

# Correct: lax.cond traces both branches; the condition is a JAX primitive
def good_relu(x):
    return jax.lax.cond(x > 0, lambda: x, lambda: 0.0)

# Also correct: jnp.where vectorises the condition
def vectorised_relu(x):
    return jnp.where(x > 0, x, 0.0)
```

`ShapedArray` is the minimal abstraction: `ShapedArray((4, 8), jnp.float32)` records that the traced value is a float32 array with shape (4, 8) but carries no element values. Higher-level abstract values (`ConcreteArray`, `DShapedArray` for dynamic shapes) exist for specialised use cases, but standard JIT uses `ShapedArray` to enable shape-polymorphic compilation — the same compiled function serves any batch size that matches the traced shape.

The static/dynamic distinction is the key design axis. Arguments declared as `static_argnums` to `jax.jit` are treated as compile-time constants and trigger recompilation when they change; all other arguments become abstract `ShapedArray` leaves in the trace.

```python
import jax
import jax.numpy as jnp

def two_layer_mlp(params, x):
    w1, b1, w2, b2 = params
    h = jnp.tanh(x @ w1 + b1)
    return h @ w2 + b2

# Construct dummy params and input
key = jax.random.PRNGKey(0)
x = jnp.ones((4,))
w1 = jnp.ones((4, 8))
b1 = jnp.zeros((8,))
w2 = jnp.ones((8, 2))
b2 = jnp.zeros((2,))
params = (w1, b1, w2, b2)

# Inspect the jaxpr
jpr = jax.make_jaxpr(two_layer_mlp)(params, x)
print(jpr)
```

The printed jaxpr has the structure:

```
{ lambda ; a:f32[4,8] b:f32[8] c:f32[8,2] d:f32[2] e:f32[4]. let
    f:f32[8] = dot_general[dimension_numbers=...] e a
    g:f32[8] = add f b
    h:f32[8] = tanh g
    i:f32[2] = dot_general[dimension_numbers=...] h c
    j:f32[2] = add i d
  in (j,) }
```

The jaxpr encodes: typed variables (`f32[4,8]`), primitives (`dot_general`, `tanh`, `add`), and a single output. It is a **functional SSA** — no side effects, no mutable state, explicit data flow. Crucially, the jaxpr is an inspectable Python object: `jpr.jaxpr.eqns` is a list of `JaxprEqn` records. Chapter 211 exploits this homoiconicity — the jaxpr can be read, transformed, and recompiled by ordinary Python.

### The Transform Composition Model

The power emerges from composing transforms. All combinations are valid:

| Expression | Semantics |
|---|---|
| `jit(f)` | Compile `f` once, cache, reuse |
| `grad(f)` | Return a function computing `∂f/∂x` |
| `jit(grad(f))` | Differentiate then compile: one XLA compilation of the derivative |
| `vmap(f)` | Vectorise `f` over a batch axis |
| `vmap(grad(f))` | Per-element gradient — efficient batch Jacobians without materialising the full Jacobian matrix |
| `jit(vmap(grad(f)))` | Compiled batched gradient: the standard training step |
| `grad(jit(f))` | Differentiate through a compiled function (works via `residuals` caching) |

```python
# Standard training step — the idiom every JAX user writes
@jax.jit
def train_step(params, x, y):
    loss, grads = jax.value_and_grad(loss_fn)(params, x, y)
    return loss, grads

# Batched Jacobian — vmap(grad(f)) materialises rows one at a time
batched_grad = jax.vmap(jax.grad(scalar_fn))
J = batched_grad(x_batch)  # shape: (batch, input_dim)
```

### scan: Structured Recursion

`jax.lax.scan(fn, carry, xs)` is JAX's primitive for **structured recursion** over a sequence. Unlike Python loops, `scan` traces `fn` once and unrolls into a single compiled loop body — critical for RNNs and sequence models where Python-level loops would produce O(T) distinct operations in the jaxpr.

```python
def rnn_step(carry, x_t):
    h = jnp.tanh(carry @ W_h + x_t @ W_x + b)
    return h, h  # (new_carry, output)

# Trace the entire sequence as a single compiled op
final_hidden, all_hidden = jax.lax.scan(rnn_step, h_init, xs)
# xs: shape (T, input_dim); all_hidden: shape (T, hidden_dim)
```

`scan` generates a `scan` primitive in the jaxpr, which XLA lowers to a hardware loop instruction — no loop unrolling overhead.

### pmap: SPMD Data Parallelism

`pmap(f)` maps `f` across local devices (GPUs or TPUs), implementing **Single-Program Multiple-Data** parallelism. Each device receives one shard of the input and executes the same compiled program. Cross-device communication uses explicit `lax` collectives:

```python
@functools.partial(jax.pmap, axis_name='devices')
def parallel_train_step(params, x_shard, y_shard):
    loss, grads = jax.value_and_grad(loss_fn)(params, x_shard, y_shard)
    # Average gradients across all devices
    grads = jax.lax.pmean(grads, axis_name='devices')
    return loss, grads

# x shape: (num_devices, batch_per_device, features)
losses, grads = parallel_train_step(replicated_params, x_sharded, y_sharded)
```

See Chapter 158 for GSPMD's extension of this model to arbitrary sharding strategies.

### The Compilation Path

The full lowering pipeline is:

```
Python function
    ↓  JAX tracing (ShapedArray propagation)
jaxpr (functional SSA IR)
    ↓  jaxpr → StableHLO lowering (jax-mlir bridge, Ch161)
StableHLO (MLIR dialect)
    ↓  StableHLO → Linalg/Tensor/Func dialects
MLIR generic dialect graph
    ↓  XLA/IREE backend (Ch153, Ch162)
PTX / HLO / native LLVM IR
```

The `jax-mlir` bridge, accessible via `jax.export`, serialises a traced computation as a StableHLO module. This is the concrete realisation of Chapter 161's JAX/TF bridge: `jax.export.export(jit(f))(x)` produces a portable StableHLO artifact that can be deployed without a JAX runtime.

### Custom PRNG and Functional Randomness

JAX's approach to randomness is explicitly functional: there is no global random state. Every random operation requires an explicit `PRNGKey`, and keys must be **split** before reuse to maintain independence:

```python
key = jax.random.PRNGKey(42)
key, subkey = jax.random.split(key)        # split: one key per use
noise = jax.random.normal(subkey, shape=(batch, dim))

# Vectorised key splitting for parallel operations
keys = jax.random.split(key, num=N)        # N independent keys
samples = jax.vmap(lambda k: jax.random.normal(k, (dim,)))(keys)
```

This design is not a restriction — it is what makes `vmap` and `pmap` composable with random operations. Each vectorised lane receives its own independent key; the compiler sees no shared mutable state. The ThreeFry and Philox PRNG backends generate keys via counter-based hashing, so `split` is a deterministic function, not a side-effecting operation. The jaxpr for a random operation simply contains the key as an input — no global state, no hidden randomness.

JAX's PRNG model is the same philosophical position as the rest of the ecosystem: **randomness is data**, not an ambient capability. This enables `jax.grad` through sampling operations via the reparameterisation trick:

```python
# Reparameterised sampling: sample z ~ N(mu, sigma) as z = mu + sigma * epsilon
def sample_gaussian(params, key):
    mu, log_sigma = params
    eps = jax.random.normal(key, mu.shape)
    return mu + jnp.exp(log_sigma) * eps   # differentiable w.r.t. mu and log_sigma

# grad differentiates through the sample
grad_fn = jax.grad(lambda params, key, y: loss(sample_gaussian(params, key), y))
```

---

## 210.2 Flax NNX: Mutable Stateful Modules

Flax's **NNX** API (introduced 2024, superseding Linen for most use cases) takes a deliberate step back from Linen's functional discipline. NNX modules are Python objects with mutable internal state, closer to PyTorch's `nn.Module` in style but with explicit mechanisms for extracting and reinjecting state for JAX transforms.

### Module Design

```python
import flax.nnx as nnx
import optax

class TransformerBlock(nnx.Module):
    def __init__(self, d_model: int, n_heads: int, rngs: nnx.Rngs):
        self.attn = nnx.MultiHeadAttention(
            num_heads=n_heads,
            in_features=d_model,
            qkv_features=d_model,
            rngs=rngs,
        )
        self.norm1 = nnx.LayerNorm(d_model, rngs=rngs)
        self.norm2 = nnx.LayerNorm(d_model, rngs=rngs)
        self.ff1 = nnx.Linear(d_model, 4 * d_model, rngs=rngs)
        self.ff2 = nnx.Linear(4 * d_model, d_model, rngs=rngs)

    def __call__(self, x):
        # Self-attention with residual
        attn_out = self.attn(inputs_q=x, inputs_k=x, inputs_v=x)
        x = self.norm1(x + attn_out)
        # Feed-forward with residual
        ff_out = self.ff2(jax.nn.relu(self.ff1(x)))
        return self.norm2(x + ff_out)

rngs = nnx.Rngs(0)
block = TransformerBlock(d_model=256, n_heads=8, rngs=rngs)
```

`nnx.Rngs(0)` creates a PRNG key manager that automatically splits keys as each parameter is initialised, ensuring reproducibility without manual key management.

### State Extraction and Reinjection

NNX's compatibility with JAX transforms relies on `nnx.split` and `nnx.merge`:

```python
# Extract state for use with JAX transforms
graphdef, state = nnx.split(block)

# state is a nested structure of nnx.VariableState objects (JAX arrays)
# graphdef carries the structural information (non-array metadata)

# Use jax.grad over the state
def loss_fn_on_state(state, x, y):
    model = nnx.merge(graphdef, state)
    logits = model(x)
    return optax.softmax_cross_entropy_with_integer_labels(logits, y).mean()

grad_fn = jax.grad(loss_fn_on_state)
grads = grad_fn(state, x_batch, y_batch)
```

`nnx.jit` and `nnx.grad` wrap this split/merge cycle automatically:

```python
@nnx.jit
def train_step(model: TransformerBlock, optimizer: nnx.Optimizer, x, y):
    loss, grads = nnx.value_and_grad(loss_fn)(model, x, y)
    optimizer.update(grads)
    return loss

optimizer = nnx.Optimizer(block, optax.adamw(learning_rate=1e-4, weight_decay=1e-2))
loss = train_step(block, optimizer, x_batch, y_batch)
```

`nnx.Optimizer` wraps an Optax `GradientTransformation` and applies it in-place to the module's variables. The mutable-OOP interface on top hides the functional plumbing below — but `nnx.split`/`nnx.merge` always gives access to the underlying JAX arrays, enabling the programmatic weight modification central to Chapter 214.

### Variable Types and Metadata

NNX's variable system goes beyond raw JAX arrays. Every parameter is stored as an `nnx.Variable` subclass that carries type metadata:

| Variable type | Semantic meaning |
|---|---|
| `nnx.Param` | Trainable parameter — differentiated by default |
| `nnx.BatchStat` | BatchNorm running statistics — not differentiated |
| `nnx.Cache` | Attention KV cache — not checkpointed |
| `nnx.Intermediate` | Activation cached for later use |

This type information propagates through `nnx.split`:

```python
graphdef, params, batch_stats = nnx.split(model, nnx.Param, nnx.BatchStat)

# Differentiate only w.r.t. Param, not BatchStat
def loss_fn(params, batch_stats, x, y):
    model = nnx.merge(graphdef, params, batch_stats)
    return cross_entropy(model(x, training=True), y)

param_grads = jax.grad(loss_fn)(params, batch_stats, x_batch, y_batch)
```

The variable-type system solves a longstanding problem in functional ML frameworks: how to distinguish parameters from non-gradient state without wrapping everything in a special container type. NNX handles it at the variable registration level — modules annotate their variables, and `nnx.split` filters them accordingly.

### Evaluation Mode and Model Introspection

NNX modules carry their own training/evaluation mode:

```python
model.eval()   # sets all BatchNorm, Dropout etc. to eval mode
logits = model(x_test)

model.train()  # back to training mode

# Inspect parameter count
n_params = sum(x.size for x in jax.tree_util.tree_leaves(
    nnx.state(model, nnx.Param)))
print(f"Parameters: {n_params:,}")
```

---

## 210.3 Optax: A Composable Optimizer DSL

Optax formalises gradient transformations as a minimal algebraic interface: every optimizer is a pair `(init, update)` where `init(params)` creates an initial optimizer state and `update(grads, state, params)` returns `(new_grads, new_state)`. This interface is closed under composition.

### Core Transforms and Composition

```python
import optax

# Atomic transforms
clip     = optax.clip_by_global_norm(max_norm=1.0)
adam_tx  = optax.adam(learning_rate=1e-3)
scale_tx = optax.scale_by_learning_rate(1e-3)  # negates for descent

# Chain composes left-to-right: clip first, then Adam
optimizer = optax.chain(
    optax.clip_by_global_norm(1.0),
    optax.adam(1e-3),
)

opt_state = optimizer.init(params)
updates, opt_state = optimizer.update(grads, opt_state, params)
new_params = optax.apply_updates(params, updates)
```

The left-to-right ordering matters: `clip_by_global_norm` rescales the raw gradients before `adam` computes its momentum and variance estimates.

### Schedule Composition

Learning-rate schedules are functions `int → float` that Optax integrates via `optax.inject_hyperparams` or dedicated schedule-aware optimizers:

```python
# Warmup for 1000 steps, then cosine decay to 0
schedule = optax.warmup_cosine_decay_schedule(
    init_value=0.0,
    peak_value=1e-3,
    warmup_steps=1000,
    decay_steps=50_000,
    end_value=1e-5,
)

optimizer = optax.chain(
    optax.clip_by_global_norm(1.0),
    optax.scale_by_adam(),          # accumulate first/second moments
    optax.scale_by_learning_rate(schedule),   # apply scheduled LR
    optax.add_decoupled_weight_decay(1e-2),   # decoupled L2
)
```

### Custom Transforms: Gradient Surgery

Optax's interface admits arbitrary gradient transformations. PCGrad (gradient surgery for multi-task learning) is implementable as a first-class `GradientTransformation`:

```python
def pcgrad() -> optax.GradientTransformation:
    """Project conflicting task gradients to resolve interference.

    Note: in practice, per-task gradients are computed outside Optax
    (via jax.grad per task), then passed to this transform's update_fn
    as a stacked PyTree. This sketch illustrates the algorithm structure.
    """

    def init_fn(params):
        return {}  # no state needed

    def update_fn(grads_list, state, params=None):
        # grads_list: list of per-task gradient trees (one per task)
        # Project each task gradient away from conflicting task gradients
        projected = []
        for i, g_i in enumerate(grads_list):
            g_proj = g_i
            for j, g_j in enumerate(grads_list):
                if i == j:
                    continue
                dot = sum(jnp.dot(a.ravel(), b.ravel())
                          for a, b in zip(jax.tree_util.tree_leaves(g_proj),
                                          jax.tree_util.tree_leaves(g_j)))
                norm_sq = sum(jnp.dot(b.ravel(), b.ravel())
                              for b in jax.tree_util.tree_leaves(g_j))
                # Subtract conflicting component only if dot < 0
                factor = jnp.where(dot < 0, dot / (norm_sq + 1e-8), 0.0)
                g_proj = jax.tree_util.tree_map(
                    lambda a, b: a - factor * b, g_proj, g_j)
            projected.append(g_proj)
        # Average projected gradients
        result = jax.tree_util.tree_map(
            lambda *gs: jnp.mean(jnp.stack(gs), axis=0), *projected)
        return result, state

    return optax.GradientTransformation(init_fn, update_fn)
```

Because Optax transforms are pure functions over JAX PyTrees, they compose correctly with `jit` and `grad` — the optimizer update is differentiable if needed (relevant for Chapter 214's meta-learning setup).

### Masked Updates and Parameter-Specific Schedules

Real training pipelines often require different hyperparameters for different parameter groups. Optax handles this with `optax.masked`:

```python
# Apply weight decay only to non-bias, non-embedding parameters
def is_decay_param(path, value):
    return 'bias' not in path and 'embedding' not in path

optimizer = optax.chain(
    optax.clip_by_global_norm(1.0),
    optax.masked(
        optax.add_decoupled_weight_decay(1e-2),
        mask=is_decay_param,
    ),
    optax.adam(schedule),
)
```

`optax.multi_transform` goes further — applying entirely different gradient transformations to different parameter groups:

```python
# Transformer training: lower LR for embeddings, higher for attention
param_labels = {'embedding': 'slow', 'attention': 'fast', 'mlp': 'fast'}

optimizer = optax.multi_transform(
    transforms={
        'slow': optax.adam(1e-4),
        'fast': optax.adam(1e-3),
    },
    param_labels=param_labels,
)
```

This composability extends to **meta-optimization**: because `optimizer.update` is a pure function, `jax.grad(optimizer.update)` computes the gradient of the updated parameters with respect to optimizer hyperparameters — enabling learned optimizers (Chapter 214) to be expressed as standard JAX computations.

---

## 210.4 Equinox: PyTree-Native Neural Networks

Equinox ([Kidger 2021, arXiv:2111.00254](https://arxiv.org/abs/2111.00254)) makes a single design bet that proves extremely powerful: **neural networks are JAX PyTrees**. A model is a nested Python dataclass whose leaves are JAX arrays. There is no framework-level state management, no `init`/`apply` split, no module registry — just Python objects and JAX transforms.

### Models as PyTrees

```python
import equinox as eqx

class MLP(eqx.Module):
    layers: list

    def __init__(self, sizes: list[int], key: jax.Array):
        keys = jax.random.split(key, len(sizes) - 1)
        self.layers = [
            eqx.nn.Linear(sizes[i], sizes[i+1], key=keys[i])
            for i in range(len(sizes) - 1)
        ]

    def __call__(self, x: jax.Array) -> jax.Array:
        for layer in self.layers[:-1]:
            x = jax.nn.relu(layer(x))
        return self.layers[-1](x)

key = jax.random.PRNGKey(42)
model = MLP([784, 256, 128, 10], key)
```

`eqx.Module` registers the class as a JAX PyTree. `jax.tree_util.tree_leaves(model)` returns all JAX arrays (weights and biases). Non-array attributes (Python lists, integers, strings) are treated as static structure.

### filter_jit and filter_grad

Standard `jax.jit` requires distinguishing dynamic (array) arguments from static ones. `eqx.filter_jit` does this automatically by inspecting the PyTree leaves:

```python
@eqx.filter_jit
def train_step(model, x, y, opt_state):
    loss, grads = eqx.filter_value_and_grad(loss_fn)(model, x, y)
    updates, opt_state = optimizer.update(grads, opt_state, eqx.filter(model, eqx.is_array))
    model = eqx.apply_updates(model, updates)
    return loss, model, opt_state
```

`eqx.filter_grad(fn)` differentiates `fn` only with respect to array leaves. Non-array fields (e.g., an activation function stored as a Python callable) are silently excluded from the gradient computation — no `jax.lax.stop_gradient` annotation required.

### PyTree Filtering and Programmatic Self-Modification

`eqx.partition(model, filter_spec)` splits a model into two PyTrees — one matching the filter (e.g., all arrays), one not — with the same structure. `eqx.combine(part1, part2)` recombines them. This enables fine-grained parameter freezing:

```python
# Freeze all layers except the last
is_last_layer = lambda leaf: isinstance(leaf, eqx.nn.Linear) and leaf is model.layers[-1]
trainable, frozen = eqx.partition(model, eqx.is_array)

# Only differentiate w.r.t. trainable part
def loss_fn_partial(trainable, frozen, x, y):
    model = eqx.combine(trainable, frozen)
    return cross_entropy(model(x), y)

grads = eqx.filter_grad(loss_fn_partial)(trainable, frozen, x, y)
```

The most powerful self-modification primitive is `eqx.tree_at`:

```python
# Replace the weight matrix of layer 0 with a pruned version
pruned_weight = model.layers[0].weight * (jnp.abs(model.layers[0].weight) > threshold)

modified_model = eqx.tree_at(
    where=lambda m: m.layers[0].weight,
    pytree=model,
    replace=pruned_weight,
)
```

`eqx.tree_at` returns a new model with the targeted leaf replaced, while leaving all other leaves untouched. Because the result is still a valid PyTree, it passes immediately to `filter_jit` and `filter_grad`. This makes Equinox the natural substrate for Chapter 214's gradient-based self-modification: weight replacement, architecture surgery, and structural model edits are ordinary Python operations on data — not framework calls.

### Equinox and Custom Layers

Equinox's PyTree philosophy extends to **custom layer types** without any registration boilerplate:

```python
class LoRALinear(eqx.Module):
    """Low-Rank Adaptation of a frozen linear layer."""
    weight: jax.Array       # frozen base weight — excluded from grad
    lora_a: jax.Array       # trainable low-rank factors
    lora_b: jax.Array
    rank: int = eqx.field(static=True)  # static: not a JAX array

    def __call__(self, x: jax.Array) -> jax.Array:
        # Full weight = frozen weight + low-rank update
        delta = self.lora_a @ self.lora_b
        return x @ (self.weight + delta)

# Freeze base weight, train only LoRA factors
def is_lora(leaf):
    return isinstance(leaf, LoRALinear)

# Partition: only lora_a and lora_b are trainable
trainable_filter = lambda leaf: eqx.is_array(leaf) and not_frozen(leaf)
```

The `eqx.field(static=True)` annotation marks `rank` as a non-JAX-array attribute: it contributes to the PyTree structure but not to the leaf arrays that `jax.grad` differentiates. This is the mechanism by which Equinox modules carry Python-level metadata (integers, strings, callables) alongside JAX arrays without any special casing.

---

## 210.5 Haiku: DeepMind's Functional Modules

Haiku ([Hennigan et al. 2020](https://github.com/google-deepmind/dm-haiku)) occupies the opposite end of the state-management spectrum from NNX. Where NNX embeds state inside module objects, Haiku **lifts stateful module code into pure functions** via a transform that separates parameter initialisation from application.

### The transform/apply Split

```python
import haiku as hk

def net_fn(x):
    mlp = hk.Sequential([
        hk.Linear(256),
        jax.nn.relu,
        hk.Linear(128),
        jax.nn.relu,
        hk.Linear(10),
    ])
    return mlp(x)

# Transform: returns (init, apply) pair
net = hk.transform(net_fn)

# init returns params (a nested dict of arrays) — no side effects
rng = jax.random.PRNGKey(0)
params = net.init(rng, x_dummy)   # params: {'linear': {'w': ..., 'b': ...}, ...}

# apply is a pure function: (params, rng, x) → output
logits = net.apply(params, rng, x_batch)
```

`params` is a plain Python dict of dicts of JAX arrays — a valid PyTree. `jax.grad(net.apply)` differentiates through the network with respect to `params` directly.

### Stateful Transforms

Modules with non-parameter state (BatchNorm running statistics, RNN hidden states) use `hk.transform_with_state`:

```python
def net_with_bn(x, is_training: bool):
    return hk.Sequential([
        hk.Linear(256),
        hk.BatchNorm(create_scale=True, create_offset=True, decay_rate=0.99),
        lambda x: x  # BatchNorm handles training/eval internally
    ])(x, is_training)

net = hk.transform_with_state(net_with_bn)

# init returns (params, state) — state holds running mean/variance
params, state = net.init(rng, x_dummy, is_training=True)

# apply returns (output, new_state)
logits, new_state = net.apply(params, state, rng, x_batch, is_training=True)
```

### Haiku vs Flax NNX: A Design Comparison

| Dimension | Haiku | Flax NNX |
|---|---|---|
| State location | External (caller holds `params`) | Internal (module owns variables) |
| Transform discipline | Functional — modules define computation, transform extracts state | OOP — explicit `nnx.split`/`nnx.merge` at JAX boundaries |
| PRNG management | `hk.next_rng_key()` — automatic key splitting inside transform | `nnx.Rngs` — explicit key manager passed at construction |
| Composition | Arbitrary function composition | Module hierarchy with `nnx.Module` base class |
| Self-modification | Natural — params are plain dicts, modify with standard dict ops | Requires `nnx.split`/`nnx.merge` cycle |

Haiku's external-state model makes it straightforward to apply any function over `params` — including `jax.tree_util.tree_map` to scale all weights, `jax.grad` to differentiate through the application, or Python dict operations to freeze, splice, or replace individual layers. For Chapter 214's second-order meta-learning, the `(params, state)` tuples that Haiku returns are first-class Python values — exactly the representation MAML needs.

### hk.next_rng_key and Automatic Key Splitting

Inside a `hk.transform`, randomness is managed through `hk.next_rng_key()`, which automatically splits the top-level RNG key for each call site:

```python
class StochasticLayer(hk.Module):
    def __init__(self, output_size: int, dropout_rate: float):
        super().__init__()
        self.linear = hk.Linear(output_size)
        self.dropout_rate = dropout_rate

    def __call__(self, x, is_training: bool):
        x = self.linear(x)
        if is_training:
            key = hk.next_rng_key()          # automatic key for this call
            mask = jax.random.bernoulli(key, 1 - self.dropout_rate, x.shape)
            x = x * mask / (1 - self.dropout_rate)
        return x

# Each call to net.apply with the same rng produces different dropout masks
# because hk.next_rng_key() folds the module path and call count into the split
net = hk.transform(lambda x, training: StochasticLayer(128, 0.1)(x, training))
y1 = net.apply(params, rng, x, True)
y2 = net.apply(params, rng, x, True)  # same rng → same mask; deterministic
y3 = net.apply(params, rng2, x, True) # different rng → different mask
```

Haiku's PRNG management is **lexically scoped**: the key is derived from the module's name path and call counter. This makes dropout deterministic given the same top-level key — a property critical for reproducibility in scientific experiments and for gradient correctness when differentiating through sampling (no random state leaks between `init` and `apply`).

### Lifting Haiku into Standard JAX Transforms

Because `net.apply(params, rng, x)` is a pure function, every JAX transform applies directly:

```python
# Differentiate w.r.t. params
loss_grad = jax.grad(
    lambda p, k, x, y: cross_entropy(net.apply(p, k, x, False), y)
)(params, rng, x_batch, y_batch)

# Vectorise over a batch of RNG keys for Monte Carlo dropout
mc_keys = jax.random.split(rng, 20)
mc_preds = jax.vmap(
    lambda k: net.apply(params, k, x_single, True)
)(mc_keys)  # shape: (20, output_dim) — 20 stochastic forward passes
uncertainty = mc_preds.var(axis=0)

# JIT the full training step
@jax.jit
def haiku_train_step(params, opt_state, rng, x, y):
    rng, dropout_rng = jax.random.split(rng)
    loss, grads = jax.value_and_grad(
        lambda p: cross_entropy(net.apply(p, dropout_rng, x, True), y)
    )(params)
    updates, opt_state = optimizer.update(grads, opt_state, params)
    return optax.apply_updates(params, updates), opt_state, rng, loss
```

Monte Carlo dropout — sampling from the posterior over functions implied by dropout — requires only `jax.vmap` over different RNG keys applied to the same `net.apply`. This is one of the clearest demonstrations that Haiku's functional discipline pays concrete dividends: a technique that requires framework-level support in PyTorch is three lines of standard JAX in Haiku.

---

## 210.6 NumPyro: Probabilistic Programming on JAX

NumPyro ([Phan et al. 2019](https://arxiv.org/abs/1912.11554)) realises the algebraic effect model sketched in Chapter 207's `{Stochastic}` design: probabilistic programs are **ordinary Python functions** annotated with `numpyro.sample` statements, and inference algorithms are **handlers** that interpret those statements differently. The connection is not metaphorical — NumPyro's effect handler system is a concrete implementation of the algebraic effect semantics Ch207 describes as a language design goal.

### Probabilistic Programs as Functions

```python
import numpyro
import numpyro.distributions as dist
from numpyro.infer import MCMC, NUTS, SVI, Trace_ELBO

def bayesian_regression(x, y=None):
    # Priors — numpyro.sample IS the {Stochastic} effect
    alpha = numpyro.sample('alpha', dist.Normal(0., 10.))
    beta  = numpyro.sample('beta',  dist.Normal(jnp.zeros(x.shape[1]),
                                                 10. * jnp.ones(x.shape[1])))
    sigma = numpyro.sample('sigma', dist.Exponential(1.))

    # Deterministic transform
    mu = alpha + x @ beta

    # Likelihood — plate for vectorised iid observations
    with numpyro.plate('data', x.shape[0]):
        numpyro.sample('obs', dist.Normal(mu, sigma), obs=y)
```

`bayesian_regression` is a pure Python function. It never returns a value in the traditional sense — its execution is *interpreted* by whichever handler wraps it.

### Inference Algorithms as Effect Handlers

The **same model function** admits multiple inference algorithms:

```python
import jax

rng_key = jax.random.PRNGKey(0)

# Handler 1: NUTS (No-U-Turn Sampler) — full Bayesian MCMC
#   Uses jax.grad internally for Hamiltonian dynamics
kernel = NUTS(bayesian_regression)
mcmc = MCMC(kernel, num_warmup=500, num_samples=1000)
mcmc.run(rng_key, x_train, y=y_train)
posterior_samples = mcmc.get_samples()

# Handler 2: SVI — variational inference with Optax optimizer
from numpyro.infer.autoguide import AutoNormal
guide = AutoNormal(bayesian_regression)
svi = SVI(bayesian_regression, guide,
          optax.adam(1e-3),
          loss=Trace_ELBO())
svi_result = svi.run(rng_key, 5000, x_train, y=y_train)
params = svi_result.params

# Handler 3: Posterior predictive distribution
from numpyro.infer import Predictive
predictive = Predictive(bayesian_regression, posterior_samples)
predictions = predictive(rng_key, x_test)['obs']
```

NUTS uses `jax.grad` to compute the gradient of the log-posterior for Hamiltonian dynamics — the probabilistic model is differentiated through automatically. `jax.vmap` is used internally to vectorise multiple MCMC chains. The entire sampler is JIT-compiled: `MCMC(..., progress_bar=False)` followed by `jax.jit(mcmc.run)` compiles the full sampling loop to a single XLA computation.

### Latent Variable Models and Amortised Inference

NumPyro's expressive power becomes clear in latent variable models — hierarchical Bayesian models with structured uncertainty:

```python
def hierarchical_model(group_ids, x, y=None):
    n_groups = group_ids.max() + 1

    # Hyperpriors — population-level distribution parameters
    mu_alpha = numpyro.sample('mu_alpha', dist.Normal(0., 5.))
    sigma_alpha = numpyro.sample('sigma_alpha', dist.HalfNormal(1.))

    # Group-level effects — one alpha per group, drawn from hyperprior
    with numpyro.plate('groups', n_groups):
        alpha = numpyro.sample('alpha', dist.Normal(mu_alpha, sigma_alpha))

    # Observation model — each datapoint selects its group's intercept
    sigma = numpyro.sample('sigma', dist.Exponential(1.))
    mu = alpha[group_ids] + x
    with numpyro.plate('data', len(x)):
        numpyro.sample('obs', dist.Normal(mu, sigma), obs=y)

# Amortised inference: AutoNormal guide learns a per-parameter Gaussian
from numpyro.infer.autoguide import AutoNormal
guide = AutoNormal(hierarchical_model)
svi = SVI(hierarchical_model, guide, optax.adam(1e-2), Trace_ELBO())
svi_result = svi.run(rng_key, 10_000, group_ids, x_train, y=y_train)
```

The same `hierarchical_model` function runs under NUTS for exact posterior sampling or SVI for scalable variational inference. This symmetry is the algebraic effect model in operation: the model specifies *what* is random; the handler specifies *how* to reason about randomness.

### The Algebraic Effect Connection

Chapter 207 describes `{Stochastic}` as an algebraic effect with three handlers: sample-once, enumerate, and integrate. NumPyro's architecture is the production realisation of this design:

| Ch207 Effect Handler | NumPyro Realisation | Mechanism |
|---|---|---|
| Sample-once (draw) | `NUTS` / `HMC` | Execute model, intercept `sample` sites, return a draw from the chain |
| Enumerate | `infer_discrete` + `TraceEnum_ELBO` | Marginalise discrete latent variables exactly |
| Integrate (variational) | `SVI` + guide + `Trace_ELBO` | Optimise ELBO lower bound with Optax |
| Posterior predictive | `Predictive` | Re-execute model with sites fixed to posterior samples |

### Cognitive Angle: Uncertainty Over Model Architectures

NumPyro enables a qualitatively different kind of uncertainty quantification: not just uncertainty over parameter values, but uncertainty over **model architectures**. A neural network's structure can be a latent variable:

```python
def architecture_model(x, y=None):
    # Latent variable: which architecture to use
    arch = numpyro.sample('arch', dist.Categorical(probs=jnp.array([0.3, 0.4, 0.3])))

    # Three candidate networks (pre-initialised)
    outputs = jnp.stack([net_a(x), net_b(x), net_c(x)])
    logits  = outputs[arch]

    with numpyro.plate('data', x.shape[0]):
        numpyro.sample('obs', dist.Categorical(logits=logits), obs=y)
```

Posterior inference over `arch` yields a probability distribution over which network architecture best explains the data — Bayesian model selection as an effect handler invocation.

---

## 210.7 BlackJAX: Composable MCMC Samplers

BlackJAX ([Lao et al. 2020](https://github.com/blackjax-devs/blackjax)) provides sampling algorithms as **pure JAX functions** rather than opaque framework objects. Every sampler exposes the same two-function interface: `init(position)` and `step(rng, state)`. This design allows samplers to be composed, differentiated through, and JIT-compiled like any other JAX function.

### The init/step Interface

```python
import blackjax

# Define the log-density of the target distribution
def log_prob_fn(position):
    return dist.MultivariateNormal(mu, Sigma).log_prob(position['x'])

# NUTS sampler — step_size and inverse_mass_matrix set adaptation targets
nuts = blackjax.nuts(log_prob_fn, step_size=0.01,
                     inverse_mass_matrix=jnp.ones(d))

# Initialise from a starting position
state = nuts.init({'x': x_init})

# Single MCMC step — pure function, JIT-compilable
rng, step_rng = jax.random.split(rng)
state, info = nuts.step(step_rng, state)
# info.acceptance_rate, info.num_integration_steps, etc.
```

### Parallel Chains via vmap

Because `step` is a pure function, `jax.vmap` parallelises across chains trivially:

```python
# Vectorise over N independent chains simultaneously
batch_init = jax.vmap(nuts.init)(jnp.stack([x_init + 0.1 * jax.random.normal(jax.random.PRNGKey(i), (d,))
                                             for i in range(N)]))
batch_step = jax.jit(jax.vmap(nuts.step))

rng_keys = jax.random.split(rng, N)
batch_state, batch_info = batch_step(rng_keys, batch_init)
```

N chains run simultaneously on a single GPU — each chain is a lane in the vectorised computation.

### Stochastic Gradient Langevin Dynamics

`blackjax.sgld` brings Bayesian reasoning into the deep learning regime by adding Gaussian noise calibrated to the gradient magnitude, approximating samples from the posterior over weights:

```python
sgld = blackjax.sgld(log_prob_fn, learning_rate=1e-3)
state = sgld.init(model_params)

@jax.jit
def sgld_step(rng, state, x_batch, y_batch):
    # log_prob_fn uses the minibatch — stochastic gradient estimate
    return sgld.step(rng, state)

# After burn-in, collect weight samples for Bayesian deep learning
weight_samples = []
for _ in range(num_steps):
    rng, step_rng = jax.random.split(rng)
    state, _ = sgld_step(step_rng, state, next(data_loader))
    weight_samples.append(state.position)
```

SGLD produces a sequence of weight configurations distributed approximately according to the posterior `p(θ | data)`. Ensemble predictions over `weight_samples` give calibrated uncertainty estimates.

### Step-Size Adaptation

The step-size hyperparameter critically affects NUTS and HMC mixing. BlackJAX provides `window_adaptation` as a wrapper that automatically tunes step size and mass matrix during a warmup phase:

```python
warmup = blackjax.window_adaptation(blackjax.nuts, log_prob_fn,
                                     num_steps=1000)

# Warmup phase — returns tuned state and tuned parameters
(state, parameters), _ = warmup.run(rng_key, initial_position)

# Production sampling with tuned step size
nuts_tuned = blackjax.nuts(log_prob_fn, **parameters)

@jax.jit
def one_step(carry, rng):
    state, _ = nuts_tuned.step(rng, carry)
    return state, state.position

keys = jax.random.split(rng_key, num_samples)
final_state, positions = jax.lax.scan(one_step, state, keys)
```

The `scan`-based sampling loop compiles the entire chain as a single XLA computation — no Python overhead per step, no host-device synchronisation until all samples are collected. A chain of 10,000 NUTS steps compiled with `jax.lax.scan` runs dramatically faster than an equivalent Python `for` loop calling `step` repeatedly, because the XLA compiler can apply kernel fusion and memory layout optimisations across the entire loop body.

---

## 210.8 Distrax and Orbax

### Distrax: JAX-Native Probability Distributions

[Distrax](https://github.com/google-deepmind/distrax) provides probability distributions as **JAX-differentiable Python objects**. Unlike `scipy.stats`, every Distrax distribution supports `jax.grad`, `jax.jit`, and `jax.vmap` through all its methods.

```python
import distrax

# Distributions are JAX-compatible objects
normal = distrax.Normal(loc=0.0, scale=1.0)
samples = normal.sample(seed=rng, sample_shape=(1000,))
log_probs = normal.log_prob(samples)          # differentiable
kl = distrax.Normal(mu_1, sigma_1).kl_divergence(distrax.Normal(mu_2, sigma_2))

# Batch distributions via vmap
batch_normal = distrax.Normal(loc=jnp.zeros(batch_size),
                               scale=jnp.ones(batch_size))
```

Distrax bijectors compose normalising flows:

```python
# Affine coupling flow
bijector = distrax.Chain([
    distrax.ScalarAffine(shift=mu, scale=jnp.exp(log_sigma)),
    distrax.Tanh(),
])
transformed = distrax.Transformed(distrax.Normal(0., 1.), bijector)
log_prob = transformed.log_prob(x)    # includes log|det J| correction
```

Because `log_prob` is differentiable, normalising flow training is `jax.grad(transformed.log_prob)` — no special flow framework required.

### Orbax: Async Checkpointing for JAX Models

[Orbax](https://orbax.readthedocs.io) provides production-grade checkpointing for JAX PyTrees, including Flax NNX, Equinox, and Haiku models.

```python
import orbax.checkpoint as ocp

# Checkpointing a Flax NNX model
checkpointer = ocp.PyTreeCheckpointer()
manager = ocp.CheckpointManager(
    directory='/tmp/model_checkpoints',
    checkpointers={'state': checkpointer},
    options=ocp.CheckpointManagerOptions(max_to_keep=3, save_interval_steps=1000),
)

# Save — synchronous by default, async variant available
graphdef, state = nnx.split(model)
manager.save(step=1000, items={'state': state})

# Async saving — returns immediately, writes in background thread
async_checkpointer = ocp.AsyncCheckpointer(ocp.PyTreeCheckpointHandler())
async_checkpointer.save('/tmp/checkpoint_async', args=ocp.args.PyTreeSave(state))
async_checkpointer.wait_until_finished()

# Restore with memory-mapped arrays — zero-copy for large checkpoints
restored_state = manager.restore(step=1000, items={'state': state})
```

Orbax's `mmap` loading avoids copying large weight tensors into host memory before transferring to device — critical for models with billions of parameters. The checkpoint format stores each leaf as a separate file keyed by its PyTree path, making partial restoration (loading only certain layers) straightforward.

The connection to Chapter 212's "Weights as Programming Substrate" is architectural: Orbax checkpoints are the JAX-ecosystem equivalent of SafeTensors — a structured, versioned, partially-addressable serialisation of a program's learned parameters. Loading a specific checkpoint and modifying individual layers before resuming training is a two-line operation: `restore` then `eqx.tree_at`.

### Cross-Framework Checkpoint Compatibility

Orbax's `PyTreeCheckpointHandler` serialises the checkpoint as a directory tree where each PyTree leaf is stored as a separate zarr/tensorstore shard named by its PyTree path:

```
/tmp/model_checkpoints/1000/
    state/
        layers/
            0/
                kernel/  ← zarr array for layers[0].kernel
                bias/
            1/
                kernel/
                bias/
```

This structure means that partial restoration — loading only certain layers from a checkpoint — does not require reading the entire checkpoint into memory. A fine-tuning workflow that modifies only the final classifier layer can restore the backbone from a checkpoint and initialise the new head from scratch:

```python
# Restore backbone weights; skip the classifier layer
restored = checkpointer.restore('/tmp/pretrained_checkpoint/1000',
                                 args=ocp.args.PyTreeRestore(target_structure))

# Graft new classifier weights onto restored backbone
new_model = eqx.tree_at(
    lambda m: m.classifier,
    restored_model,
    replace=new_classifier,
)
```

The `target_structure` passed to `restore` guides the deserialization: any leaf in `target_structure` that is set to `None` is skipped in the restored checkpoint, enabling selective layer loading without custom serialisation code.

---

## 210.9 Brax: Differentiable Physics as Cognitive Substrate

[Brax](https://github.com/google/brax) implements rigid-body physics simulation — joints, contacts, constraints — as a **pure JAX function**. A Brax environment's `step` function takes a state PyTree and an action array and returns a new state PyTree. Because it is written in JAX, it is automatically differentiable, JIT-compilable, and vectorisable.

```python
import brax
import brax.envs

# Create a physics environment — ant robot locomotion
env = brax.envs.create('ant')

# JIT the physics step — compiles the entire rigid-body simulation
jit_step = jax.jit(env.step)
jit_reset = jax.jit(env.reset)

# Single environment step
rng, step_rng = jax.random.split(rng)
state = jit_reset(step_rng)
action = jnp.zeros(env.action_size)
next_state = jit_step(state, action)
```

### Massively Parallel Simulation

The defining capability is `vmap` over thousands of physics worlds simultaneously:

```python
# Initialise 4096 parallel environments
rng_keys = jax.random.split(rng, 4096)
batch_reset = jax.jit(jax.vmap(env.reset))
batch_step  = jax.jit(jax.vmap(env.step))

states = batch_reset(rng_keys)           # 4096 independent physics states
actions = policy_network(states.obs)     # batched policy forward pass
next_states = batch_step(states, actions)  # 4096 physics steps in parallel
```

On a single A100 GPU, Brax simulates tens of thousands of physics environments per second — faster than real-time simulation by many orders of magnitude. The entire RL training loop — environment step, reward computation, policy gradient update — runs as a single compiled XLA computation.

### Differentiable RL and Cognitive Self-Improvement

Because `env.step` is differentiable, model-based RL becomes `jax.grad` through the environment:

```python
def trajectory_return(policy_params, initial_state, horizon: int):
    def step_fn(carry, _):
        state = carry
        action = policy_apply(policy_params, state.obs)
        next_state = env.step(state, action)
        return next_state, next_state.reward

    _, rewards = jax.lax.scan(step_fn, initial_state, None, length=horizon)
    return -rewards.sum()  # negative for minimisation

# Differentiate through the entire horizon — policy gradient with exact gradients
policy_grad = jax.grad(trajectory_return)(policy_params, initial_state, horizon=100)
```

This is the cognitive self-improvement loop made concrete: the agent's policy is a JAX PyTree; the environment is a JAX function; `jax.grad` produces exact policy gradients through the entire simulated trajectory. The distinction between "environment" and "policy" dissolves into a single compiled computation graph — precisely the neural-program-as-artifact framing that Chapter 211 develops.

### Brax and PPO: A Fully Compiled RL Pipeline

Brax ships with reference implementations of Proximal Policy Optimisation (PPO) written entirely in JAX. The training loop — experience collection, advantage estimation, policy gradient update — is a single `jax.lax.scan` over time steps, `jax.vmap` over environments, and `jax.jit` over the entire step:

```python
from brax.training.agents.ppo import train as ppo_train

# Trains for 10M environment steps; the entire training loop is JIT-compiled
make_inference_fn, params, metrics = ppo_train(
    environment=brax.envs.create('humanoid'),
    num_timesteps=10_000_000,
    episode_length=1000,
    num_envs=2048,              # 2048 parallel environments via vmap
    learning_rate=3e-4,
    entropy_cost=1e-2,
    progress_fn=lambda step, metrics: print(step, metrics),
)
```

The `num_envs=2048` argument vectorises over 2048 parallel physics simulations simultaneously. Brax's PPO achieves wall-clock training speeds that previously required CPU clusters, because the environment, policy network, and optimizer update all execute in a single XLA launch — zero Python overhead per environment step.

The differentiable physics substrate opens a qualitatively different learning regime: instead of estimating policy gradients via Monte Carlo rollouts (PPO), one can differentiate *analytically* through the environment dynamics:

```python
# Analytic policy gradient — gradient flows through the physics simulation
physics_grad = jax.grad(
    lambda params: -trajectory_return(params, state, horizon=50)
)(policy_params)
```

Analytic gradients are lower-variance than Monte Carlo estimates and converge faster on smooth control problems — but require differentiable dynamics, which Brax provides. This regime — compiled differentiable physics + analytic policy gradient — is the frontier of JAX-ecosystem cognitive self-improvement research.

---

## 210.10 The Ecosystem as a Functional Compiler Stack

The ten libraries surveyed in this chapter are not independent tools — they are **layers in a coherent functional compilation stack**, each providing a well-defined semantic service.

### The Interoperability Architecture

```
Application layer:
  Brax (physics env)  ←→  NumPyro (uncertainty)
         ↓                        ↓
  Flax NNX / Equinox / Haiku (model representations)
         ↓                        ↓
  Optax (gradient transforms)   BlackJAX (MCMC)
         ↓                        ↓
  Distrax (distributions)    Orbax (checkpointing)
         ↓                        ↓
─────────────── JAX core ────────────────────────
  jit │ grad │ vmap │ pmap │ scan
         ↓
  jaxpr (functional SSA IR)
         ↓
  StableHLO → MLIR → XLA / LLVM
```

Every library in the stack exposes its state as a JAX-compatible PyTree. `jax.tree_util.tree_map` operates uniformly across the entire system:

```python
# Scale all parameters by 0.5 — works for Flax NNX, Equinox, Haiku, Optax state
scaled_state = jax.tree_util.tree_map(lambda x: x * 0.5, system_state)

# Zero all gradients — works regardless of library provenance
zero_grads = jax.tree_util.tree_map(jnp.zeros_like, grads)
```

### MAML: grad(grad(...)) for Meta-Learning

Model-Agnostic Meta-Learning ([Finn et al. 2017](https://arxiv.org/abs/1703.03400)) requires differentiating through an inner gradient update — `grad(grad(loss))`. In JAX, this is direct:

```python
def maml_outer_loss(meta_params, support_x, support_y, query_x, query_y,
                    inner_lr: float = 0.01, inner_steps: int = 5):
    # Inner loop: adapt to the support set
    adapted = meta_params
    for _ in range(inner_steps):
        inner_grad = jax.grad(loss_fn)(adapted, support_x, support_y)
        adapted = jax.tree_util.tree_map(
            lambda p, g: p - inner_lr * g, adapted, inner_grad)

    # Outer loss: evaluate adapted params on query set
    return loss_fn(adapted, query_x, query_y)

# jax.grad differentiates through the inner loop automatically
meta_grad = jax.grad(maml_outer_loss)(meta_params, *task_data)
meta_params = jax.tree_util.tree_map(
    lambda p, g: p - outer_lr * g, meta_params, meta_grad)
```

JAX's forward-mode and reverse-mode AD compose correctly through the inner `jax.grad` calls. The jaxpr for `maml_outer_loss` captures the inner gradient steps as ordinary arithmetic — the compiler sees a flat computation graph and optimises it as such. Chapter 214 extends this pattern to higher-order self-modification.

### vmap for Ensemble Cognition

`jax.vmap` transforms the per-example semantics of `grad` into batched semantics:

```python
# Evaluate N model variants in parallel — ensemble inference
# Each model has the same PyTree structure but different leaf values
stacked_params = jax.tree_util.tree_map(lambda *ps: jnp.stack(ps), *ensemble)

@jax.vmap
def ensemble_forward(params, x):
    return model_apply(params, x)

# ensemble_logits: shape (N, batch, classes)
ensemble_logits = ensemble_forward(stacked_params, x_batch)
mean_prediction = ensemble_logits.mean(axis=0)
predictive_uncertainty = ensemble_logits.var(axis=0)
```

The N ensemble members run simultaneously in a single compiled kernel — no Python loop overhead, no separate GPU launches. This is ensemble cognition at compiler resolution: the parallel evaluation of N hypotheses about the world is a single XLA operation.

### The jaxpr as Homoiconic IR

The `jaxpr` is the linchpin connecting all of the above to Chapter 211's "Neural Programs as Compiled Artifacts" thesis. `jax.make_jaxpr(f)(x)` produces a Python object that can be:

- **Inspected**: `jpr.jaxpr.eqns` lists every primitive operation
- **Transformed**: rewrite rules over `JaxprEqn` objects implement custom IR transformations
- **Re-lowered**: `jax.core.eval_jaxpr(jpr.jaxpr, jpr.consts, *args)` interprets the jaxpr

This homoiconicity — the program is its own inspectable data structure — is the functional-language analogue of the reflective properties Chapter 207 identifies as the goal of AI-first programming languages. JAX does not require a reflective type theory to achieve introspection over computations; it uses the Python runtime and the jaxpr data structure instead.

The full compilation pipeline assembled from this chapter is:

1. **Model** (Equinox PyTree or Flax NNX module) defines the computation
2. **Optax** transforms gradients with composable schedule and clipping
3. **NumPyro** quantifies uncertainty over parameters or architectures via effect handlers
4. **BlackJAX** samples from posteriors using pure functional MCMC
5. **Brax** provides a differentiable environment for RL-based self-improvement
6. **Distrax** provides differentiable distributions throughout
7. **Orbax** serialises and restores the full system state as versioned checkpoints
8. **JAX core** compiles the entire pipeline to a single StableHLO module via jaxpr

The result is a **closed functional compiler stack**: from Python-level model definition through to native hardware execution, every component is a pure function over JAX PyTrees, every state transition is differentiable, and the entire system can be JIT-compiled, checkpointed, restored, and modified programmatically.

### Limits and Open Problems

The functional purity that makes the JAX ecosystem powerful also introduces real frictions:

- **Dynamic shapes**: JAX traces with fixed shapes by default; `jax.jit` recompiles when shapes change. Models with variable-length sequences must either pad to a fixed length or use `jax.pure_callback` to delegate dynamic-shape operations outside the JIT boundary. MLIR's `DynamicShapedType` and JAX's experimental `jax.jit(abstracted_axes=...)` address this incrementally.
- **Compilation latency**: JIT compilation of a large transformer can take tens of seconds on first call. Persistent compilation cache (`JAX_COMPILATION_CACHE_DIR`) mitigates recompilation, but cold-start latency remains a real deployment concern.
- **Python control flow in models**: models with data-dependent branching (mixture of experts with dynamic routing, tree-structured neural networks) must encode their branching as JAX primitives (`lax.switch`, `lax.while_loop`), which can be unintuitive and limit expressive power.
- **Debugging**: abstract trace values give stack traces that point to JAX-internal code rather than user code. `jax.debug.print` and `jax.debug.breakpoint` partially address this, but debugging JIT-compiled code remains harder than debugging eager-mode PyTorch.

These are engineering frontier problems, not fundamental limitations. The compilation path through StableHLO and MLIR (Chapters 153-154) gives the JAX ecosystem a rigorous lowering strategy; future JAX versions will progressively relax the fixed-shape requirement through MLIR's dynamic shape infrastructure.

---

## Chapter Summary

- **JAX's transforms** (`jit`, `grad`, `vmap`, `pmap`, `scan`) are composable higher-order functions that act as compiler passes; their composition model is the foundation of the entire ecosystem. The `jaxpr` — JAX's internal functional SSA IR — is produced by tracing Python functions with abstract `ShapedArray` values and lowered to StableHLO → MLIR → native backends.

- **Flax NNX** provides mutable, OOP-style neural network modules with `nnx.split`/`nnx.merge` as the bridge to JAX transforms; `nnx.Optimizer` wraps Optax and applies updates in-place while preserving the functional substrate beneath.

- **Optax** formalises gradient transformations as composable `(init, update)` pairs; `optax.chain` sequences transforms left-to-right; schedule integration and custom transforms (PCGrad, gradient surgery) are first-class citizens of the same algebraic structure.

- **Equinox** treats neural networks as JAX PyTrees outright; `eqx.filter_jit`, `eqx.filter_grad`, `eqx.partition`, and `eqx.tree_at` provide fine-grained control over which leaves participate in compilation, differentiation, and programmatic modification — making Equinox the natural substrate for self-modifying neural programs.

- **Haiku** separates module state from computation via `hk.transform` → `(init, apply)` pairs; parameters are plain Python dicts of JAX arrays that compose with any JAX transform, enabling Haiku's natural use in second-order meta-learning.

- **NumPyro** realises the `{Stochastic}` algebraic effect model from Chapter 207: probabilistic programs are ordinary Python functions; `NUTS`, `SVI`, and `Predictive` are effect handlers that interpret `numpyro.sample` sites differently, all backed by JAX's `grad` and `vmap`.

- **BlackJAX**, **Distrax**, and **Orbax** complete the probabilistic and operational layers: BlackJAX provides composable pure-function MCMC; Distrax provides fully differentiable probability distributions with bijectors for normalising flows; Orbax provides async, versioned, memory-mapped checkpointing of arbitrary JAX PyTrees.

- **Brax** closes the loop by providing differentiable physics simulation as a pure JAX function; `jax.vmap` over thousands of parallel environments combined with `jax.grad` through the trajectory enables compiled, exact-gradient RL — the entire agent-environment loop as a single XLA computation, realising the cognitive self-improvement architecture that Chapter 214 analyses in depth.
