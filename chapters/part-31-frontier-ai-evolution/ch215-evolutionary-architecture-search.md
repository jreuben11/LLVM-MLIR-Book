# Chapter 215 — Evolutionary Architecture Search

*Part XXXI — Frontier AI Evolution*

Gradient descent is the workhorse of modern neural network training, but it is structurally incapable of exploring the space of discrete architectures, program structures, or agent source code. A gradient requires a smooth, differentiable loss surface; changing which attention heads exist, which kernel tiles are used, or which lines of code an agent runs introduces discontinuities that gradient descent cannot bridge. The search methods that can — evolutionary algorithms, quality-diversity illumination, and LLM-as-mutation-operator loops — have quietly matured to the point where they match or exceed gradient-based NAS on standard benchmarks and, as the Darwin Gödel Machine demonstrates, can improve coding agent performance by 20–50% on real engineering tasks. This chapter develops the four interlocking ideas: the Darwin Gödel Machine as a deployed evolutionary loop over coding agents (§215.2); NEAT as the canonical topology-and-weight co-evolution framework (§215.3); Quality-Diversity algorithms — MAP-Elites, AURORA, and descriptor-conditioned gradients — as the search architecture that makes population-based search tractable (§215.4); and program synthesis systems (DreamCoder, FunSearch) as sources of structured search operators (§215.5). Sections §215.6 and §215.7 close the loop: kernel-architecture co-design as a concrete fitness landscape, and the cognitive-theoretic relationship to the gradient-based and formal self-improvement approaches in adjacent chapters.

Cross-references: [Chapter 207 — Reflective Code, Open Problems, and Build Roadmap](../part-30-AI-first-PL-design/ch207-reflective-code-open-problems-roadmap.md) · [Chapter 208 — GPU Kernel DSLs: Triton, Helion, and Gluon](ch208-gpu-kernel-dsls-triton-helion-gluon.md) · [Chapter 209 — CUTLASS, Thrust, CuTe, and TileIR](ch209-cutlass-thrust-cute-tileir.md) · [Chapter 214 — Gradient-Based Self-Modification](ch214-gradient-based-self-modification.md) · [Chapter 216 — Formal Self-Improvement Theory](ch216-formal-self-improvement-theory.md)

---

## Table of Contents

- [215.1 Motivation: Beyond Gradient Descent](#2151-motivation-beyond-gradient-descent)
  - [The Local-Optima Problem](#the-local-optima-problem)
  - [Self-Modification as Search](#self-modification-as-search)
  - [Search Space Geometry](#search-space-geometry)
- [215.2 Darwin Gödel Machine](#2152-darwin-gdel-machine)
  - [215.2.1 System Architecture](#21521-system-architecture)
  - [215.2.2 Reported Empirical Results](#21522-reported-empirical-results)
  - [215.2.3 Mutation Loop Pseudocode](#21523-mutation-loop-pseudocode)
  - [215.2.4 Triton Kernel Variants as Fitness Landscape](#21524-triton-kernel-variants-as-fitness-landscape)
- [215.3 NEAT: Topology-and-Weight Co-Evolution](#2153-neat-topology-and-weight-co-evolution)
  - [215.3.1 Historical Markings](#21531-historical-markings)
  - [215.3.2 Speciation](#21532-speciation)
  - [215.3.3 Connection to Neural Architecture Search](#21533-connection-to-neural-architecture-search)
- [215.4 Quality-Diversity Search](#2154-quality-diversity-search)
  - [215.4.1 MAP-Elites: Illumination Algorithm](#21541-map-elites-illumination-algorithm)
  - [215.4.2 `qdax.core.map_elites` in JAX](#21542-qdaxcoremapelites-in-jax)
  - [215.4.3 AURORA: Unsupervised Behaviour Characterisation](#21543-aurora-unsupervised-behaviour-characterisation)
  - [215.4.4 Descriptor-Conditioned Gradients](#21544-descriptor-conditioned-gradients)
- [215.5 Program Synthesis for Architecture Construction](#2155-program-synthesis-for-architecture-construction)
  - [215.5.1 DreamCoder: Bayesian Program Induction](#21551-dreamcoder-bayesian-program-induction)
  - [215.5.2 FunSearch: LLM Mutation with Evaluator](#21552-funsearch-llm-mutation-with-evaluator)
  - [215.5.3 FunSearch-Style Search over MLIR Lowering Strategies](#21553-funsearch-style-search-over-mlir-lowering-strategies)
- [215.6 Kernel and Architecture Co-Design](#2156-kernel-and-architecture-co-design)
  - [215.6.1 The Co-Design Fitness Landscape](#21561-the-co-design-fitness-landscape)
  - [215.6.2 Population Representation](#21562-population-representation)
  - [215.6.3 Integration with Triton Autotune](#21563-integration-with-triton-autotune)
- [215.7 Cognitive Angle: Evolution as the Complement to Gradient Descent](#2157-cognitive-angle-evolution-as-the-complement-to-gradient-descent)
  - [215.7.1 Two Modes of Self-Improvement](#21571-two-modes-of-self-improvement)
  - [215.7.2 Open-Ended Search and Archive Diversity](#21572-open-ended-search-and-archive-diversity)
  - [215.7.3 Self-Improvement as SICA](#21573-self-improvement-as-sica)
  - [215.7.4 Gödel-Completeness and the Halting Problem](#21574-gdel-completeness-and-the-halting-problem)
  - [215.7.5 Comparing Evolutionary and Gradient-Based Search Efficiency](#21575-comparing-evolutionary-and-gradient-based-search-efficiency)
- [Chapter Summary](#chapter-summary)

---

## 215.1 Motivation: Beyond Gradient Descent

### The Local-Optima Problem

Gradient descent navigates a loss surface by following the direction of steepest local decrease. This works extraordinarily well when the loss surface is smooth and the relevant search variables are continuous — as they are for weight parameters in a fixed architecture. It fails in two structurally distinct ways when the problem involves discrete structure.

First, discrete choices introduce barriers that gradient-based methods cannot cross. Whether to add a skip connection, whether to fuse two kernel operations, whether to use multi-query attention versus multi-head attention — these are binary or categorical variables. Continuous relaxations (Gumbel-softmax, DARTS differentiable NAS) smooth the surface artificially but the resulting topology after discretisation is often not the topology the relaxed optimiser was exploring. The information lost in the discretisation step is not recovered by the continuous gradient.

Second, the fitness function may itself be non-differentiable. Compiler throughput, correctness on a test suite, or performance on a benchmark are not smooth functions of the parameter space. A Triton kernel that fails to compile has no gradient with respect to its tile-size parameters; a coding agent whose patch breaks a test suite has no gradient signal from the test outcome. Evolutionary search treats fitness evaluation as a black box: any function that maps a candidate solution to a scalar (or vector) score is a valid fitness.

### Self-Modification as Search

The deeper motivation for evolutionary methods in this part of the book is self-modification: agents that improve their own code, architectures, or kernel configurations. [Chapter 214 — Gradient-Based Self-Modification](ch214-gradient-based-self-modification.md) covers the gradient side of this problem — meta-learning, model editing, and test-time adaptation. But gradient-based self-modification is constrained to modifications that preserve differentiability. An agent that wants to rewrite its own inference code, change which external tools it calls, or restructure its own reasoning loop must operate in a discrete, non-differentiable space. Evolutionary search — specifically, using a language model as a structured mutation operator — is the natural mechanism for this class of self-modification.

The Gödel Machine theory (see [Chapter 216 — Formal Self-Improvement Theory](ch216-formal-self-improvement-theory.md)) establishes that provably beneficial self-modifications should be preferred. In practice, proof-theoretic guarantees are unattainable for complex systems; the Darwin Gödel Machine (§215.2) replaces formal proof with empirical fitness evaluation in an evolutionary loop, trading theoretical guarantees for practical tractability.

### Search Space Geometry

The space relevant to this chapter is the union of three distinct search problems:

| Search problem | Variables | Fitness function | Scale |
|---|---|---|---|
| Neural Architecture Search (NAS) | Layer types, connectivity, dimensions | Validation accuracy, parameter count | 10²–10⁶ candidates |
| Kernel Configuration Search | Tile sizes, pipeline stages, warp counts | Kernel throughput (TFLOP/s) | 10²–10⁴ candidates |
| Agent Code Search (DGM) | Python source of agent (10³–10⁵ tokens) | Benchmark score (SWE-bench, Polyglot) | 10²–10³ candidates |

All three are amenable to evolutionary approaches; they differ in evaluation cost (milliseconds for kernel benchmarks, minutes for agent benchmarks) and mutation operator design (random perturbation for NAS, LLM for agent code).

---

## 215.2 Darwin Gödel Machine

The Darwin Gödel Machine (DGM) — introduced in [Lehman et al. (Sakana AI / Jeff Clune, 2025), arXiv 2505.22954](https://arxiv.org/abs/2505.22954) — operationalises open-ended self-improvement by treating an LLM as the mutation operator in a population-based evolutionary loop. Each individual in the population is a complete, executable coding agent: a Python program that can receive a task description, write code, execute it, and return results. The agents can modify their own source code as part of their task execution strategy, so self-modification is not a meta-level capability bolted on — it is part of each agent's toolset.

### 215.2.1 System Architecture

The DGM loop operates over three components:

1. **Archive** — a Quality-Diversity archive (§215.4) of (agent-program, behaviour-descriptor, fitness-score) triples. The archive maintains diversity: agents that solve different subsets of benchmark tasks occupy different archive cells even if their aggregate fitness is similar.

2. **LLM mutation operator** — a frontier LLM (the paper uses Claude 3.5 Sonnet and Claude 3.7 Sonnet as the mutation backend) that receives the source code of a parent agent and a natural-language description of candidate improvements, and produces modified source. The mutation operator is not random: it is a structured program transformation guided by in-context reasoning about what the parent agent does well and where it fails.

3. **Evaluator** — a sandboxed execution environment that runs the mutant agent on the benchmark suite, records per-task pass/fail, and computes the aggregate fitness score and the behaviour descriptor (which tasks were solved).

The key claim of arXiv 2505.22954 is that this loop discovers agent improvements that no human designed and that gradient descent cannot find — because the fitness function (benchmark pass rate) is non-differentiable and the search variable (agent source code) is discrete.

### 215.2.2 Reported Empirical Results

The paper reports two primary results on coding benchmarks:

- **SWE-bench Verified**: the DGM loop produces a ~20% improvement in pass rate over the seed agent, starting from a baseline competitive with frontier coding agents.
- **Polyglot**: a ~50% improvement over the seed agent on this multilingual coding benchmark.

These are relative improvements over the starting seed agent, not absolute scores; the absolute figures depend on the choice of seed. The improvements accumulate over hundreds of evolutionary iterations, with each iteration requiring one LLM call to generate the mutation and one sandboxed evaluation run to score it. The cost is non-trivial (frontier LLM calls are expensive), but the resulting agents have made qualitative architectural improvements — adding error-recovery loops, restructuring their reasoning chains, and adapting their tool-use patterns — that no single fine-tuning step would have found.

### 215.2.3 Mutation Loop Pseudocode

The core algorithm is straightforward once the archive data structure and mutation operator are defined:

```python
def dgm_loop(seed_agent_src: str, evaluator, archive, llm_mutator, n_iterations: int):
    initial_fitness, initial_descriptor = evaluator(seed_agent_src)
    archive.add(seed_agent_src, initial_descriptor, initial_fitness)

    for _ in range(n_iterations):
        parent_src = archive.sample_parent()

        improvement_prompt = build_improvement_prompt(parent_src)
        mutant_src = llm_mutator(improvement_prompt)

        if not is_syntactically_valid(mutant_src):
            continue

        fitness, descriptor = evaluator(mutant_src)
        archive.add_if_improves(mutant_src, descriptor, fitness)
```

`archive.sample_parent()` selects from the existing population with a bias toward high-fitness individuals; the precise selection pressure is a hyperparameter. `archive.add_if_improves(src, descriptor, fitness)` adds the mutant if the cell indexed by `descriptor` is empty or contains a lower-fitness individual — the MAP-Elites update rule (§215.4.1).

`build_improvement_prompt` is the critical design choice. The paper uses a structured prompt that includes:
- The parent agent's full source code
- A sample of tasks the parent failed
- A request for a specific, targeted improvement to the source

The structured prompt constrains the LLM to propose *coherent* modifications — not random character substitutions but refactoring-level changes with a stated rationale. This is the key difference between DGM and random program mutation: the LLM's prior over program transformations is concentrated on semantically meaningful edits.

### 215.2.4 Triton Kernel Variants as Fitness Landscape

One concrete instantiation of DGM-style search that does not require an LLM evaluator is kernel variant search. A population of Triton kernel implementations is maintained; each variant is a complete Python function implementing the same mathematical operation with different tile strategies, pipeline depths, or memory access patterns. The fitness function is kernel throughput — measured in TFLOP/s on a fixed input shape on a specific GPU:

```python
import triton
import triton.language as tl
import torch
import time

def evaluate_kernel_fitness(kernel_fn, args, reference_fn, input_shapes) -> float:
    inputs = [torch.randn(*s, device='cuda', dtype=torch.float16) for s in input_shapes]
    ref_output = reference_fn(*inputs)

    try:
        output = kernel_fn(*inputs, *args)
    except Exception:
        return 0.0

    if not torch.allclose(output, ref_output, atol=1e-2, rtol=1e-2):
        return 0.0

    torch.cuda.synchronize()
    t0 = time.perf_counter()
    for _ in range(100):
        kernel_fn(*inputs, *args)
    torch.cuda.synchronize()
    t1 = time.perf_counter()

    elapsed_ms = (t1 - t0) * 1000 / 100
    flops = compute_flop_count(input_shapes)
    return flops / (elapsed_ms * 1e9)
```

This evaluate-compile-benchmark cycle is an exact evolutionary fitness evaluation with no gradient. The population of kernel variants lives in the archive; an LLM (or structured random mutation) proposes new variants; the evaluator runs them through the Triton compiler and benchmarks the result. Archive cells are indexed by the kernel's structural descriptor — e.g., `(BLOCK_M, BLOCK_N, num_stages)` — so the archive maintains a diverse collection of tile strategies rather than converging to a single configuration.

---

## 215.3 NEAT: Topology-and-Weight Co-Evolution

NEAT (NeuroEvolution of Augmenting Topologies), introduced in [Stanley & Miikkulainen (2002), *Evolutionary Computation* 10(2)](https://dl.acm.org/doi/10.1162/106365602320169811), solves a fundamental problem in neural architecture search: how do you apply genetic crossover to networks with different topologies? A simple weight crossover between a 3-layer and a 5-layer network produces garbage because there is no alignment between the gene positions.

### 215.3.1 Historical Markings

NEAT's solution is the *historical marking* (also called an innovation number): every time a new structural element (a new connection or a new node) is added to any network in the population, it receives a globally unique integer tag. This tag is preserved through all subsequent copies of that element. When two networks are crossed over, genes with matching historical markings are aligned; excess and disjoint genes (those without a match in the other parent) are inherited from the more-fit parent.

```
Parent A:  [1][2][3][4]      [6]       nodes: 3
Parent B:  [1][2][3]   [5][6][7]       nodes: 4

Aligned:   [1][2][3]   
Disjoint:  [4]    from A; [5][7] from B
Excess:    [6]    shared — take from fitter parent
```

The result is offspring that can be meaningfully evaluated: the topology is consistent (no dangling connections from misaligned crossover), and the structural diversity from both parents is preserved. This is not possible with naive weight-vector crossover on heterogeneous architectures.

There are two NEAT mutation operators that extend topology:

1. **Add connection**: pick two existing nodes i and j with no existing connection i→j; create connection with historical marking n+1 and random initial weight.
2. **Add node**: pick an existing connection i→j with weight w; disable it; create new node k; create connections i→k (weight 1.0) and k→j (weight w). The unit weight from i→k preserves the original output magnitude initially.

These structural mutations are applied with low probability per generation — typically 0.03 for add-node and 0.05 for add-connection. The small probability prevents the topology from growing unboundedly while still allowing structural exploration over many generations.

A minimal Python representation of a NEAT genome illustrates the historical marking scheme:

```python
from dataclasses import dataclass, field

@dataclass
class ConnectionGene:
    in_node: int
    out_node: int
    weight: float
    enabled: bool
    innovation: int

@dataclass
class NodeGene:
    node_id: int
    node_type: str   # 'input', 'hidden', 'output'

@dataclass
class Genome:
    nodes: list[NodeGene] = field(default_factory=list)
    connections: list[ConnectionGene] = field(default_factory=list)

    def crossover(self, other: 'Genome', self_fitness: float, other_fitness: float) -> 'Genome':
        self_innov = {c.innovation: c for c in self.connections}
        other_innov = {c.innovation: c for c in other.connections}
        all_innovations = set(self_innov) | set(other_innov)

        child_connections = []
        for innov in sorted(all_innovations):
            in_self = innov in self_innov
            in_other = innov in other_innov
            if in_self and in_other:
                gene = self_innov[innov] if self_fitness >= other_fitness else other_innov[innov]
                if not (self_innov[innov].enabled and other_innov[innov].enabled):
                    gene.enabled = False   # disabled in either parent → 75% chance disabled
            elif in_self:
                gene = self_innov[innov]   # excess/disjoint: take from fitter parent
            else:
                gene = other_innov[innov] if other_fitness > self_fitness else None
            if gene is not None:
                child_connections.append(gene)

        child_nodes = list({n.node_id: n for n in self.nodes + other.nodes}.values())
        return Genome(nodes=child_nodes, connections=child_connections)
```

The `crossover` method demonstrates the aligned/disjoint/excess logic: aligned genes are drawn from the fitter parent; disjoint and excess genes are inherited from the fitter parent only.

### 215.3.2 Speciation

The second key mechanism is *speciation*: grouping individuals into species based on a compatibility distance δ, and competing within species rather than globally. The compatibility distance is:

```
δ(G₁, G₂) = c₁·E/N + c₂·D/N + c₃·W̄
```

where E is the number of excess genes, D is the number of disjoint genes, W̄ is the mean weight difference on matching genes, N is the normalisation factor (genome length), and c₁, c₂, c₃ are coefficients. Individuals with δ < δ_t share a species.

Speciation protects topological innovations. A newly added connection or node typically reduces fitness initially — the weights are random and the network needs time to adjust. Without speciation, natural selection eliminates these innovations before their weights are tuned. With speciation, a new individual shares resources only with others of similar structure; it competes in a smaller pool where its structural novelty can be refined before being judged against the global population.

### 215.3.3 Connection to Neural Architecture Search

NEAT predates modern NAS by 15 years, but the connection is direct. DARTS (differentiable NAS), random search NAS, and reinforcement-learning NAS all search the same space NEAT searches — the space of possible network topologies. NEAT's historical markings solve the *competing conventions problem* that differentiable NAS sidesteps by maintaining a fixed supernet topology. For discrete, variable-topology search — which is the relevant case when generating Triton kernel tile structures or transformer attention patterns with different connectivity — NEAT's mechanisms remain the clearest solution.

The speciation mechanism maps onto the archive diversity maintenance in MAP-Elites (§215.4): both protect innovations from premature elimination. The difference is that MAP-Elites uses an explicit behaviour descriptor for diversity, while NEAT uses structural compatibility distance. For the kernel search case, these two approaches converge: tile-size parameters are both the structural descriptor and the performance driver.

| NEAT Concept | MAP-Elites Equivalent | Kernel Search Instance |
|---|---|---|
| Historical marking | Archive cell index | `(BLOCK_M, BLOCK_N, num_stages)` |
| Compatibility distance δ | Behaviour descriptor distance | Tile configuration Euclidean distance |
| Species | Archive cell | Fixed tile-size bucket |
| Innovation protection (speciation) | Cell-local replacement | Replace only if same cell, higher fitness |
| Crossover (aligned genes) | No direct equivalent | Tile parameter interpolation |

---

## 215.4 Quality-Diversity Search

Quality-Diversity (QD) algorithms maintain a population of solutions that is simultaneously high-quality (high fitness) and diverse (covering a wide range of behaviour). Classic evolutionary algorithms optimise for a single fitness objective; QD algorithms illuminate the entire behaviour space — every distinct type of behaviour gets a representative high-quality solution. This chapter uses QD as the population management strategy for both DGM and kernel search.

The three standard QD metrics quantify search progress:

- **QD-score**: Σ_{occupied cells} fitness(cell). Increases when either new cells are found or existing cell fitness improves.
- **Coverage**: |occupied cells| / |total cells|. Measures how much of behaviour space has been explored.
- **Max fitness**: the fitness of the single best solution, regardless of cell. Classical evolutionary metric.

A mature QD run maximises QD-score and coverage jointly; a run that optimises only max fitness is classical evolutionary search. The two objectives are not equivalent: an archive with 500 cells at average fitness 0.8 has QD-score 400 and may be more useful than an archive with 100 cells at average fitness 0.95 (QD-score 95) if the downstream consumer needs solutions for 500 distinct operational conditions.

### 215.4.1 MAP-Elites: Illumination Algorithm

MAP-Elites [Mouret & Clune (2015), arXiv 1504.04909](https://arxiv.org/abs/1504.04909) maintains a structured archive: a grid where each cell corresponds to a region of *behaviour descriptor space*, and each cell stores the highest-fitness solution that falls in that region.

**Algorithm:**

```
1. Initialise: generate N random solutions, evaluate each, add to archive
2. Loop:
   a. Sample x from archive (uniformly or fitness-proportionate)
   b. Generate x' = mutate(x)
   c. (fitness, descriptor) = evaluate(x')
   d. If archive[cell(descriptor)] is empty or fitness > archive[cell(descriptor)].fitness:
          archive[cell(descriptor)] = (x', fitness, descriptor)
3. Return: full archive (not just the best solution)
```

The output of MAP-Elites is not a single solution but a *map* — an illumination of the behaviour-performance space. For kernel search, this means: for every tile configuration that was explored, the best-performing kernel with that configuration is retained. The engineer (or a downstream system) can then select from this map based on secondary criteria — memory footprint, compilation time, numerical stability — that were not part of the primary fitness.

### 215.4.2 `qdax.core.map_elites` in JAX

QDax [Lim et al. (2022), arXiv 2202.01258](https://arxiv.org/abs/2202.01258) implements MAP-Elites and its variants in JAX, enabling GPU-parallel fitness evaluation and JIT-compiled archive updates. The core API:

```python
import jax
import jax.numpy as jnp
from qdax.core.map_elites import MAPElites
from qdax.core.containers.mapelites_repertoire import compute_cvt_centroids
from qdax.core.emitters.mutation_operators import isoline_variation
from qdax.core.emitters.standard_emitters import MixingEmitter
from qdax.utils.metrics import default_qd_metrics
from qdax.types import Fitness, Descriptor, RNGKey
```

A minimal MAP-Elites run over a parameterised kernel configuration space requires defining three functions:

```python
def scoring_fn(params: jnp.ndarray, random_key: RNGKey):
    """Evaluate a batch of kernel configurations. Returns (fitness, descriptor)."""
    block_m = (params[:, 0] * 128).astype(jnp.int32).clip(32, 256)
    block_n = (params[:, 1] * 128).astype(jnp.int32).clip(32, 256)
    num_stages = (params[:, 2] * 4).astype(jnp.int32).clip(1, 5)

    fitness = jax.vmap(benchmark_config)(block_m, block_n, num_stages)
    descriptor = jnp.stack([block_m / 256.0, block_n / 256.0], axis=-1)
    return fitness, descriptor

def benchmark_config(block_m, block_n, num_stages) -> jnp.ndarray:
    """Single-config benchmark — runs outside JIT via pure_callback."""
    return jax.pure_callback(
        _python_benchmark,
        jax.ShapeDtypeStruct((), jnp.float32),
        block_m, block_n, num_stages
    )

emitter = MixingEmitter(
    mutation_fn=lambda x, r: isoline_variation(x, x, 0.05, 0.1, r),
    variation_fn=None,
    variation_percentage=0.0,
    batch_size=256,
)

map_elites = MAPElites(
    scoring_function=scoring_fn,
    emitter=emitter,
    metrics_function=default_qd_metrics,
)
```

Initialisation and the main loop:

```python
centroids, random_key = compute_cvt_centroids(
    num_descriptors=2,
    num_init_cvt_samples=10_000,
    num_centroids=1024,
    minval=0.0,
    maxval=1.0,
    random_key=jax.random.PRNGKey(42),
)

init_params = jax.random.uniform(random_key, shape=(1024, 3))
repertoire, emitter_state, random_key = map_elites.init(
    init_params, centroids, random_key
)

for generation in range(500):
    repertoire, emitter_state, metrics, random_key = map_elites.update(
        repertoire, emitter_state, random_key
    )
    if generation % 50 == 0:
        print(f"gen {generation}: qd_score={metrics['qd_score']:.2f}, "
              f"coverage={metrics['coverage']:.3f}")
```

`qd_score` is the sum of fitnesses across all occupied archive cells — the standard QD metric. `coverage` is the fraction of archive cells that are occupied. A mature MAP-Elites run maximises both: high qd_score means every cell has a high-fitness solution; high coverage means the entire behaviour space has been explored.

### 215.4.3 AURORA: Unsupervised Behaviour Characterisation

MAP-Elites requires a hand-designed behaviour descriptor: the human must decide which dimensions of behaviour matter. AURORA [Cully (2019), arXiv 1905.11874](https://arxiv.org/abs/1905.11874) removes this requirement by learning the descriptor representation from the solutions themselves using an unsupervised autoencoder.

The AURORA loop interleaves MAP-Elites archive updates with periodic autoencoder retraining:

```python
def aurora_loop(scoring_fn, n_generations, encoder_update_period):
    encoder = initialize_encoder(latent_dim=2)
    archive = initialize_archive(descriptor_fn=encoder.encode)

    for gen in range(n_generations):
        candidates = archive.sample_batch(256)
        raw_behaviours, fitnesses = scoring_fn(candidates)
        descriptors = encoder.encode(raw_behaviours)
        archive.update(candidates, descriptors, fitnesses)

        if gen % encoder_update_period == 0:
            all_behaviours = archive.get_all_behaviours()
            encoder.retrain(all_behaviours)
            archive.remap_descriptors(encoder.encode)
```

`remap_descriptors` is the critical step: when the encoder is retrained, all existing archive entries must be re-indexed under the new descriptor function. This causes periodic reorganisation of the archive but ensures the descriptor dimensions always reflect the most informative behavioural axes discovered so far.

For kernel search, the raw behaviour is the full runtime profile — SM occupancy, L2 cache hit rate, DRAM bandwidth, register pressure — and the autoencoder learns a 2D embedding that separates memory-bound from compute-bound configurations without the human needing to specify this distinction in advance.

### 215.4.4 Descriptor-Conditioned Gradients

The MAP-Elites + Descriptor-Conditioned Gradients (DCG-MAP-Elites) algorithm [Lim et al. (2023), arXiv 2303.03832](https://arxiv.org/abs/2303.03832) injects gradient information into the MAP-Elites loop when the fitness function is differentiable with respect to continuous parameters. The key insight is that gradient descent can be used *locally* — to polish a solution within its archive cell — while MAP-Elites manages global diversity.

For the attention head configuration search case (continuous parameters: query/key/value projection dimensions, head counts), DCG-MAP-Elites applies:

1. **Variation step** (MAP-Elites): sample parent from archive, apply random mutation to explore new cells
2. **Improvement step** (gradient): apply k steps of gradient descent on the fitness function, keeping the descriptor approximately constant by penalising descriptor deviation

```python
def dcg_improvement_step(params, fitness_fn, descriptor_fn, target_descriptor,
                          n_steps=10, lr=1e-3, lambda_desc=10.0):
    opt = optax.adam(lr)
    opt_state = opt.init(params)

    for _ in range(n_steps):
        def loss(p):
            f = fitness_fn(p)
            d = descriptor_fn(p)
            desc_penalty = lambda_desc * jnp.sum((d - target_descriptor) ** 2)
            return -f + desc_penalty

        grads = jax.grad(loss)(params)
        updates, opt_state = opt.update(grads, opt_state)
        params = optax.apply_updates(params, updates)

    return params
```

The descriptor penalty keeps the improved solution in (approximately) the same archive cell as the parent, so the improvement step refines quality without collapsing diversity. This is the QD analogue of fine-tuning: global structure from evolution, local quality from gradient descent.

---

## 215.5 Program Synthesis for Architecture Construction

When the search variable is a program — a function that maps inputs to outputs, not just a vector of parameters — random mutation is inefficient because most random program modifications are syntactically invalid or semantically incoherent. Program synthesis systems address this by searching over structured program spaces with compositional primitives.

### 215.5.1 DreamCoder: Bayesian Program Induction

DreamCoder [Ellis et al. (2021), arXiv 2006.08381](https://arxiv.org/abs/2006.08381) frames program synthesis as Bayesian program induction over a library of primitives. The key claim is that the right primitive library makes programs short; the right programs make the primitive library self-evident. DreamCoder alternates between two phases:

**Wake phase**: given the current primitive library, search for programs that solve the training tasks. Search uses a neural recognition model (amortised inference) that proposes program sketches, which are completed by enumeration.

**Sleep phase**: given the programs found during wake, compress them into new library primitives (the `dreamcoder` compression step) and retrain the recognition model on synthetic tasks generated by the current programs.

The compression step extracts common subprograms across the solution set and lifts them into new library primitives. This is analogous to function extraction during compiler optimisation: repeated patterns become named, reusable abstractions.

For neural architecture construction, the primitive library consists of architectural building blocks:

```python
primitives = {
    "attention":    lambda q, k, v, h: MultiHeadAttention(q, k, v, num_heads=h),
    "ffn":          lambda x, d: FeedForward(x, dim=d),
    "layer_norm":   lambda x: LayerNorm(x),
    "residual":     lambda f, x: x + f(x),
    "repeat":       lambda f, n, x: sequential_apply(f, n, x),
    "gate":         lambda f, g, x: f(x) * sigmoid(g(x)),
}
```

A program in this DSL is a tree of primitive applications. DreamCoder's recognition model generates a distribution over trees; enumeration fills in the parameters. The evolutionary pressure comes from retaining programs that solve more tasks and compressing their common substructure into new primitives.

The connection to MLIR is direct: the primitive library corresponds to a set of named MLIR patterns, and a program in the primitive DSL corresponds to an MLIR module. Library primitives map to MLIR ops; program composition maps to `func.call` or op sequencing; the compression step corresponds to pattern extraction passes. DreamCoder's wake-sleep loop can be seen as a structured version of the MLIR `transform` dialect's pattern application, extended with a learning component.

### 215.5.2 FunSearch: LLM Mutation with Evaluator

FunSearch [Romera-Paredes et al. (DeepMind, *Nature*, 2023)](https://www.nature.com/articles/s41586-023-06924-6) evolves mathematical programs using an LLM as the mutation operator and a deterministic evaluator as the fitness function. The system discovered new constructions for the cap set problem and bin-packing algorithms that outperformed all known human solutions.

The FunSearch architecture has four components:

1. **Program database**: a population of (program, score) pairs, organised as a MAP-Elites-style archive indexed by program "island" (a diversity mechanism to prevent convergence).

2. **Sampler**: selects a high-scoring program from the database and constructs a prompt containing the program plus a request to improve it.

3. **LLM**: generates a modified program (mutation in program space).

4. **Evaluator**: runs the program in a sandbox, computes a score, and returns it to the database.

The key design decision in FunSearch is the *function specification interface*: the LLM is asked to fill in the body of a specific function within a larger scaffold. The scaffold defines the problem (inputs, outputs, correctness constraints); the LLM provides the algorithm. This separation keeps the LLM's output in a syntactically constrained space while still allowing algorithmic creativity.

The island model is a second key design decision. Rather than a single MAP-Elites archive, FunSearch maintains k independent "islands", each a separate (program, score) archive. Programs migrate between islands periodically; islands evolve independently between migrations. This prevents premature convergence: a subpopulation that has converged on a local optimum on one island does not prevent another island from exploring a different region of program space.

```python
class FunSearchDatabase:
    def __init__(self, n_islands: int = 10, programs_per_island: int = 100):
        self.islands = [[] for _ in range(n_islands)]
        self.programs_per_island = programs_per_island

    def add(self, island_id: int, program: str, score: float):
        island = self.islands[island_id]
        island.append((score, program))
        island.sort(reverse=True)
        if len(island) > self.programs_per_island:
            island.pop()

    def sample_for_prompt(self, island_id: int, n_examples: int = 2) -> list[str]:
        island = self.islands[island_id]
        if not island:
            return []
        weights = [s for s, _ in island]
        total = sum(weights)
        probs = [w / total for w in weights]
        import random
        chosen = random.choices(island, weights=probs, k=n_examples)
        return [prog for _, prog in chosen]

    def migrate(self, fraction: float = 0.1):
        for i, island in enumerate(self.islands):
            n_migrants = max(1, int(len(island) * fraction))
            migrants = island[:n_migrants]
            target = (i + 1) % len(self.islands)
            for score, prog in migrants:
                self.add(target, prog, score)
```

The `sample_for_prompt` method selects programs proportionally to their score — higher-scoring programs appear more frequently in the context window presented to the LLM. The LLM sees two or three example programs with their scores and is asked to propose an improvement. This in-context learning loop is more efficient than reinforcement fine-tuning because it requires no gradient computation: the model's weights are frozen, and the adaptation is entirely via prompt construction.

### 215.5.3 FunSearch-Style Search over MLIR Lowering Strategies

Applying FunSearch to MLIR lowering strategy selection is a natural fit. The scaffold is a fixed MLIR module with a `TODO` region to be filled; the LLM generates a sequence of `mlir-opt` passes and transforms; the evaluator runs the resulting module and benchmarks it.

```python
LOWERING_SCAFFOLD = """
func.func @matmul(%A: memref<{M}x{K}xf32>, %B: memref<{K}x{N}xf32>,
                  %C: memref<{M}x{N}xf32>) {{
  // LOWERING_STRATEGY_HERE
  return
}}
"""

def funsearch_mlir_step(parent_strategy: str, llm, evaluator) -> tuple[str, float]:
    prompt = f"""
    The following MLIR lowering strategy achieves {evaluator(parent_strategy):.2f} GFLOP/s:

    {parent_strategy}

    Propose an improved lowering strategy for the same matmul. Output only the strategy
    body (the content that replaces // LOWERING_STRATEGY_HERE). Valid transforms include:
    affine loop tiling, vectorization, promotion to shared memory, and unrolling.
    """
    candidate_strategy = llm(prompt)

    full_module = LOWERING_SCAFFOLD.format(M=1024, K=1024, N=1024).replace(
        "// LOWERING_STRATEGY_HERE", candidate_strategy
    )

    try:
        throughput = evaluator(full_module)
        return candidate_strategy, throughput
    except CompilationError:
        return parent_strategy, 0.0
```

The evaluator compiles the module with `mlir-opt` followed by `mlc` or LLVM codegen, runs the resulting binary on a fixed benchmark input, and returns throughput. This is exactly the FunSearch loop with an MLIR compiler in the evaluation slot. The LLM's prior over valid MLIR transform sequences is much stronger than random mutation — it knows that tiling must precede vectorisation, that certain tile sizes are legal for a given architecture, and that affine maps must be invertible for dependence analysis.

A practical evaluator for this workflow invokes `mlir-opt` directly:

```bash
#!/bin/bash
# evaluate_lowering_strategy.sh — wraps mlir-opt + llc + timing
STRATEGY_FILE="$1"
M=1024; K=1024; N=1024

/usr/lib/llvm-22/bin/mlir-opt \
    --affine-loop-tile="tile-size=32" \
    --affine-loop-unroll="unroll-factor=4" \
    --convert-affine-to-standard \
    --convert-linalg-to-loops \
    --lower-affine \
    --convert-scf-to-cf \
    --convert-cf-to-llvm \
    --convert-func-to-llvm \
    "$STRATEGY_FILE" -o /tmp/lowered.mlir 2>/dev/null \
  && /usr/lib/llvm-22/bin/mlir-translate --mlir-to-llvmir /tmp/lowered.mlir \
        -o /tmp/lowered.ll 2>/dev/null \
  && /usr/lib/llvm-22/bin/llc -O3 -march=native /tmp/lowered.ll -o /tmp/lowered.s \
  && gcc /tmp/lowered.s -o /tmp/bench_binary \
  && /usr/bin/time -f "%e" /tmp/bench_binary $M $K $N 2>&1 | tail -1
```

Each candidate lowering strategy produced by the LLM is saved as a temporary MLIR file and passed through this pipeline. The evaluator returns a throughput value (or 0.0 on compilation failure), and the FunSearch database records the result. After a few hundred iterations, the database's top-scoring island contains lowering strategies that discover pass orderings and tiling configurations that the hand-tuned default pipeline misses.

---

## 215.6 Kernel and Architecture Co-Design

The most powerful application of evolutionary search in this chapter is the joint optimisation of neural network architecture and its compiled execution substrate: simultaneously evolving the model's structure (which layers, which attention variants, which activation functions) and the kernel configurations (tile sizes, pipeline stages, memory layouts) that execute it.

### 215.6.1 The Co-Design Fitness Landscape

Single-objective optimisation over architecture (for accuracy) and separate single-objective optimisation over kernels (for throughput) produce suboptimal outcomes because the two objectives interact. An attention variant that achieves 0.2% higher accuracy but requires an irregular access pattern that prevents tensor core utilisation is worse than a slightly less accurate attention variant whose tile structure maps perfectly to WGMMA instructions. The co-design fitness function captures this interaction:

```python
def codesign_fitness(architecture_params, kernel_params):
    model = instantiate_architecture(architecture_params)
    compiled_model = compile_with_kernels(model, kernel_params)

    task_accuracy = evaluate_task(compiled_model, val_dataset)
    kernel_throughput = benchmark_kernels(compiled_model, input_shapes)
    memory_footprint = measure_peak_memory(compiled_model)

    return (
        0.5 * task_accuracy
        + 0.3 * normalise(kernel_throughput, baseline_throughput)
        - 0.2 * normalise(memory_footprint, memory_budget)
    )
```

The weights (0.5, 0.3, −0.2) encode design priorities. A QD search over this fitness with a behaviour descriptor of `(task_accuracy, kernel_throughput)` produces the full Pareto front: every trade-off point between accuracy and throughput gets a representative solution.

### 215.6.2 Population Representation

Each individual in the population is a pair `(architecture_spec, kernel_config)`:

```python
@dataclass
class Individual:
    architecture: dict   # num_heads, d_model, d_ff, num_layers, attn_variant
    kernel_cfg: dict     # BLOCK_M, BLOCK_N, num_stages, num_warps per kernel type
    fitness: float
    descriptor: tuple    # (task_accuracy, throughput_gflops)
```

Mutation operates independently on each component: architectural mutation changes one architectural parameter (e.g., halves the number of attention heads); kernel mutation changes one kernel parameter (e.g., increases `num_stages` by 1). Crossover (where applicable, as in NEAT) aligns architectural genes by their historical marking.

A co-design mutation step that respects hardware constraints:

```python
import random

VALID_BLOCK_SIZES = [32, 64, 96, 128, 192, 256]
VALID_STAGES = [1, 2, 3, 4, 5]
VALID_WARPS = [4, 8, 16]
ATTN_VARIANTS = ['mha', 'mqa', 'gqa_4', 'gqa_8', 'linear']

def mutate_individual(ind: Individual, arch_prob=0.5) -> Individual:
    arch = dict(ind.architecture)
    kern = dict(ind.kernel_cfg)

    if random.random() < arch_prob:
        choice = random.choice(['num_heads', 'd_model', 'd_ff', 'attn_variant'])
        if choice == 'num_heads':
            arch['num_heads'] = random.choice([4, 8, 16, 32])
        elif choice == 'd_model':
            arch['d_model'] = random.choice([512, 768, 1024, 2048])
        elif choice == 'd_ff':
            arch['d_ff'] = arch['d_model'] * random.choice([2, 4, 8])
        elif choice == 'attn_variant':
            arch['attn_variant'] = random.choice(ATTN_VARIANTS)
    else:
        choice = random.choice(['BLOCK_M', 'BLOCK_N', 'num_stages', 'num_warps'])
        if choice == 'BLOCK_M':
            kern['BLOCK_M'] = random.choice(VALID_BLOCK_SIZES)
        elif choice == 'BLOCK_N':
            kern['BLOCK_N'] = random.choice(VALID_BLOCK_SIZES)
        elif choice == 'num_stages':
            kern['num_stages'] = random.choice(VALID_STAGES)
        elif choice == 'num_warps':
            kern['num_warps'] = random.choice(VALID_WARPS)

    return Individual(architecture=arch, kernel_cfg=kern, fitness=0.0, descriptor=(0.0, 0.0))
```

The constraint that `BLOCK_M` and `BLOCK_N` must divide the matrix dimensions is enforced in the evaluator (returning 0.0 fitness for invalid configurations) rather than in the mutation operator. This is intentional: moving constraint enforcement to evaluation is simpler and allows the archive to learn which regions of configuration space are valid, which can accelerate future search by biasing parent selection away from invalid regions.

### 215.6.3 Integration with Triton Autotune

Triton's built-in autotune (`triton.autotune`) evaluates a fixed set of pre-specified configurations. Evolutionary search extends this to configurations not in the pre-specified set — the evolutionary loop proposes configurations that Triton's autotune would never consider because they fall outside the hand-designed grid. The interaction with [Chapter 208 — GPU Kernel DSLs: Triton, Helion, and Gluon](ch208-gpu-kernel-dsls-triton-helion-gluon.md) and [Chapter 209 — CUTLASS, Thrust, CuTe, and TileIR](ch209-cutlass-thrust-cute-tileir.md) is direct: the evolutionary fitness function wraps the Triton compilation pipeline, so the genetic operators work in the configuration space and the Triton compiler handles the hardware mapping.

```bash
# Benchmark a candidate Triton configuration
/usr/lib/llvm-22/bin/python3 -c "
import triton_benchmark
result = triton_benchmark.run(
    kernel='matmul',
    block_m=192, block_n=96, block_k=64,
    num_stages=3, num_warps=8,
    dtype='float16', m=4096, n=4096, k=4096
)
print(f'throughput_tflops={result.tflops:.3f}')
"
```

The CuTe layout algebra from Chapter 209 constrains which configurations are valid: tile dimensions must divide the matrix dimensions evenly, or the kernel must handle the boundary with masking. An evolutionary search that ignores these constraints wastes evaluation budget on invalid individuals; a constraint-aware mutation operator that only proposes valid configurations (by checking divisibility and hardware alignment requirements before evaluation) dramatically improves search efficiency.

---

## 215.7 Cognitive Angle: Evolution as the Complement to Gradient Descent

### 215.7.1 Two Modes of Self-Improvement

The chapters surrounding this one map out the full landscape of machine self-improvement:

- **[Chapter 214 — Gradient-Based Self-Modification](ch214-gradient-based-self-modification.md)**: continuous, differentiable modifications to weights, prompts, or model parameters via gradient signals. Fast per-step, but confined to differentiable fitness landscapes.
- **Chapter 215 (this chapter)**: discrete, non-differentiable modifications to architecture, code, or kernel configuration via evolutionary search with an LLM mutation operator. Slower per-evaluation, but capable of exploring the combinatorial space of programs and architectures.
- **[Chapter 216 — Formal Self-Improvement Theory](ch216-formal-self-improvement-theory.md)**: self-modifications that come with proof-theoretic guarantees of improvement. Soundest, but computationally intractable for all but the simplest systems.

The Darwin Gödel Machine is the practical realisation of Jürgen Schmidhuber's theoretical Gödel Machine [Schmidhuber (2003), arXiv cs/0309048](https://arxiv.org/abs/cs/0309048). The original Gödel Machine requires that a self-modification be *proven* to increase expected utility before it is applied; the DGM relaxes the proof requirement to empirical evaluation, making the loop tractable while abandoning the formal guarantee. This is the same trade-off that DreamCoder makes relative to formal synthesis: tractable search over a structured space, with empirical validation replacing proof.

The LLM mutation operator in DGM is not a generic "write code" prompt. It is a structured improvement proposal that asks the model to reason about which specific aspect of the parent agent's behaviour is suboptimal and what targeted change would improve it. A typical prompt structure:

```python
IMPROVEMENT_PROMPT_TEMPLATE = """
You are improving a coding agent. Here is the agent's current source code:

--- AGENT SOURCE ---
{agent_source}
--- END SOURCE ---

This agent was evaluated on {n_tasks} benchmark tasks and failed {n_failed} of them.
Here are representative failing tasks:

{failing_task_examples}

Identify one specific weakness in the agent's strategy and propose a targeted
improvement. Common improvement categories:
- Error recovery: the agent does not retry or adapt after tool errors
- Context management: the agent loses track of long-horizon task state
- Tool selection: the agent uses suboptimal tools for certain task types
- Verification: the agent does not validate its own outputs before submitting

Output the complete modified agent source code. Make only the changes needed to
address the identified weakness. Do not refactor unrelated code.
"""
```

This structure is important for two reasons. First, naming the weakness category forces the LLM to commit to a specific type of modification rather than producing a diffuse rewrite. Second, providing concrete failing examples gives the LLM in-context signal about which failure mode to address — the same mechanism that makes few-shot prompting effective. The resulting modifications are more coherent than random mutations and more likely to pass the fitness evaluation.

### 215.7.2 Open-Ended Search and Archive Diversity

The archive-based QD approach (MAP-Elites, AURORA) is specifically designed for *open-ended* search — the goal is not to find a single optimal solution but to discover the structure of the solution space. This is qualitatively different from both gradient descent (single trajectory) and formal search (single proof). The DGM uses this property to discover agent improvements that no single training run would have found — because each niche in the archive represents a different strategy for solving the benchmark, and the evolutionary pressure favours strategies that are best in their niche even if they are not globally optimal.

For compiler development, open-ended search over lowering strategies (§215.5.3) has a similar property: the archive produced by a QD run over MLIR lowering strategies maps the trade-off space between compile time, code size, and runtime performance. An engineer querying this archive gets not "the best strategy" but "the best strategy given your constraints" — a fundamentally more useful output than a single-point optimum.

### 215.7.3 Self-Improvement as SICA

The [Chapter 207 — Reflective Code, Open Problems, and Build Roadmap](../part-30-AI-first-PL-design/ch207-reflective-code-open-problems-roadmap.md) chapter establishes the SICA (Self-Improving Coding Agent) framing: a coding agent that can modify its own source code and validate the modifications. DGM is SICA instantiated as an evolutionary loop:

- The population is a set of SICA instances with different source-code histories
- The mutation operator is the LLM that proposes source modifications
- The fitness function is the SICA's benchmark score
- The archive ensures that SICA variants with different strategies all survive, preventing premature convergence to a single coding style

The key insight is that the evolutionary loop is *the* mechanism for escaping local optima in the SICA improvement trajectory. A single SICA instance doing gradient-style self-improvement (adjusting prompts, refining its reasoning chains via RLHF) is confined to a neighbourhood of its starting point. An evolutionary population of SICA instances explores the full space of coding strategies, and the best innovations from any individual propagate (via the archive and parent selection) to the entire population.

### 215.7.4 Gödel-Completeness and the Halting Problem

A theoretical limit deserves acknowledgment. Any self-improving system that evaluates the fitness of proposed modifications must, in general, solve the halting problem — it must determine whether the proposed modification terminates and what it computes. The DGM sidesteps this by using a timeout (sandbox evaluation with a time limit); MAP-Elites sidesteps it by requiring the evaluator to terminate (the sandbox kills long-running candidates). These are pragmatic engineering decisions, not theoretical solutions. Chapter 216 addresses the formal theory; this chapter's scope is the engineering practice that makes evolutionary self-improvement work despite the theoretical limits.

### 215.7.5 Comparing Evolutionary and Gradient-Based Search Efficiency

A common objection to evolutionary search is sample efficiency: gradient descent updates a parameter vector with O(1) evaluations per step (one forward pass, one backward pass); evolutionary search requires many fitness evaluations per generation. The objection is correct but incomplete.

For problems where gradient descent is applicable, it is strictly more sample-efficient than evolution. For problems where it is not applicable — discrete architecture search, program synthesis, agent code modification — evolutionary search is the only option, and its sample efficiency should be measured against the alternative (exhaustive enumeration), not against gradient descent.

The real cost comparison for DGM is: LLM inference (one mutation step) + sandbox evaluation (one benchmark run) per generation, versus gradient fine-tuning (thousands of parameter updates, labelled training data) per improvement cycle. At frontier LLM inference costs and 2026 GPU rental rates, a DGM run of 500 generations is economically competitive with a fine-tuning run of comparable capability improvement — and produces a discrete improvement to the agent's code rather than a weight update that is difficult to inspect or audit.

For kernel configuration search specifically, the comparison is more direct. Triton's built-in autotune exhaustively evaluates every configuration in a pre-specified grid. MAP-Elites with 500 fitness evaluations explores a much larger configuration space with the same evaluation budget by concentrating evaluations in promising regions. The QD metric — QD-score, the sum of fitnesses across all archive cells — strictly dominates the autotune metric (best single configuration) as a measure of search effectiveness when the downstream use case involves choosing different configurations for different input shapes.

| Search method | Evaluations to reach 95% of peak | Configurations explored | Output |
|---|---|---|---|
| Triton autotune (grid) | O(grid size) = 50–500 | Fixed set | Single best config |
| Random search | O(budget) = 500 | Uniform sample | Single best found |
| MAP-Elites (QD) | O(budget) = 500 | Adaptive | Archive: best per cell |
| DGM (agent code) | O(iterations × eval cost) | Open-ended | Modified source code |

The archive output of MAP-Elites is qualitatively different from the single-point output of autotune. An engineering team with an archive can ask: "which tile configuration is best for 512×512 inputs on A100?" and "which is best for 8192×8192 on H100?" and get different answers from the same archive, whereas autotune gives a single answer calibrated to the benchmark shape.

---

## Chapter Summary

- **Gradient descent is insufficient for discrete search**: architecture topology, kernel configuration, and agent source code are combinatorial search problems that require evolutionary or program-synthesis methods.

- **Darwin Gödel Machine (arXiv 2505.22954)** applies an LLM as a structured mutation operator on a population of coding agents, using archive-based Quality-Diversity to maintain diverse strategies. Reported results: 20% improvement on SWE-bench Verified, 50% on Polyglot.

- **Triton kernel variants as fitness landscape**: DGM-style search over tile configurations (BLOCK_M, BLOCK_N, num_stages) uses compiled kernel throughput as a black-box fitness function, with QD diversity maintained over tile-shape descriptor space.

- **NEAT** solves topology-and-weight co-evolution via historical markings (crossover alignment) and speciation (innovation protection). Its mechanisms map directly onto MAP-Elites archive management.

- **MAP-Elites** maintains a behaviour-indexed archive of high-fitness solutions, illuminating the entire behaviour-performance trade-off space rather than converging to a single optimum. `qdax.core.map_elites` provides JAX-accelerated implementation.

- **AURORA** extends MAP-Elites with unsupervised autoencoder-based behaviour characterisation, removing the requirement for hand-designed behaviour descriptors.

- **DCG-MAP-Elites (arXiv 2303.03832)** injects local gradient steps into the MAP-Elites loop where fitness is differentiable, combining global diversity from evolution with local quality from gradient descent.

- **DreamCoder (arXiv 2006.08381)** uses Bayesian program induction with a learned primitive library; the wake-sleep loop simultaneously finds programs and compresses their common structure into reusable primitives.

- **FunSearch (DeepMind, Nature 2023)** evolves mathematical programs using an LLM as mutation operator and a deterministic evaluator; the same architecture applies to MLIR lowering strategy search.

- **Kernel-architecture co-design** defines a joint fitness function over (architecture parameters, kernel configuration) pairs, producing the Pareto front via QD search rather than a single operating point.

- **Evolutionary search is the complement of gradient-based self-modification** (Chapter 214) and the practical relaxation of formal self-improvement (Chapter 216): it trades proof-theoretic soundness for tractability in the discrete, non-differentiable search spaces that characterise real self-improving systems.
