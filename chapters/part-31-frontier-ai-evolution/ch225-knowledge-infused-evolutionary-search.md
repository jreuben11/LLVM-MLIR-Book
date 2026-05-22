# Chapter 225 — Knowledge-Infused Evolutionary Search: Pass Synergy Graphs and ECCO

*Part XXXI — Frontier AI Evolution*

Classical genetic algorithms applied to compiler pass ordering treat passes as opaque tokens: a sequence is a string, crossover is a splice, mutation is an insertion or deletion. This works — given enough generations and a large enough population, random exploration finds good sequences — but it is slow, wasteful, and ignorant of what every compiler engineer knows: certain passes depend on each other, certain passes undo each other's work, and certain clusters of passes consistently outperform any individual pass alone. Knowledge-infused evolutionary search encodes these relationships as explicit structures — *behavioral vectors* that characterize each pass's effect on IR and *synergy graphs* that capture enabling and interfering relationships — and uses them to design genetic operators that respect compiler semantics. ECCO extends this by applying causal reasoning to explain *why* a sequence works and prune it to its effective subset. This chapter covers the theoretical foundations, the data structures, the operators, and the empirical results on production benchmarks.

---

## 225.1 Beyond Blind Genetic Operators

### The Blindness Problem

A standard GA for pass ordering works as follows:

```python
population = [random_pass_sequence(length=10) for _ in range(100)]
for generation in range(1000):
    fitness = [evaluate(seq, ir_corpus) for seq in population]
    parents = select(population, fitness, method="tournament")
    offspring = [crossover(p1, p2) for p1, p2 in parents]
    offspring = [mutate(seq) for seq in offspring]
    population = offspring
```

The `crossover` and `mutate` operations are blind:
- `crossover(p1, p2)`: splice p1's first half with p2's second half at a random position
- `mutate(seq)`: insert/delete/swap a random pass at a random position

Blind crossover can splice together two sequences at a boundary where pass `A` (from p1) requires analysis produced by pass `B` (from p2), which now comes *after* `A`. The resulting sequence either silently skips the dependent transformation or produces incorrect IR.

Blind mutation can insert a pass whose prerequisites are not met — no SSA form when a pass requiring SSA is inserted after an SSA-destroying pass.

Both problems are recoverable (the GA will eventually discard bad sequences) but waste generations on structurally invalid offspring.

### What Domain Knowledge Provides

Compiler engineers know:
- `mem2reg` should run before `gvn` (mem2reg produces SSA form that GVN requires)
- `instcombine` and `reassociate` are mutually beneficial (instcombine simplifies expressions, reassociate reveals new instcombine opportunities)
- `loop-unroll` should run after `loop-rotate` (rotation enables the induction variable pattern that unrolling depends on)
- `simplifycfg` should run after any pass that produces dead basic blocks

Encoding this knowledge directly into the search operators eliminates structurally invalid offspring and focuses exploration on the high-value region of pass-sequence space.

---

## 225.2 Pass Behavioral Vectors

A *behavioral vector* is a fixed-dimension binary or float vector that characterizes each LLVM pass by its effect on IR properties.

### Vector Dimensions

Each dimension represents an IR property:

```
dim 0: establishes_ssa         (pass produces SSA form)
dim 1: requires_ssa            (pass requires SSA form as precondition)
dim 2: reduces_instructions    (pass typically reduces instruction count)
dim 3: increases_instructions  (pass may increase instruction count via inlining/unrolling)
dim 4: reduces_allocas         (pass lowers alloca count, e.g., mem2reg)
dim 5: invalidates_domtree     (pass may change the dominator tree)
dim 6: requires_domtree        (pass requires an up-to-date dominator tree)
dim 7: produces_loops          (pass introduces loop structure)
dim 8: eliminates_loops        (pass may reduce loop count via unrolling)
dim 9: reduces_calls           (pass inlines calls, reducing call count)
...   (24 additional dimensions)
```

### Computing Behavioral Vectors from LLVM's Analysis Framework

LLVM passes declare their analysis requirements and preserved analyses via `getAnalysisUsage`:

```cpp
// From LLVM pass infrastructure:
// A pass declares what it preserves and what it requires
void GVN::getAnalysisUsage(AnalysisUsage& AU) const {
    AU.addRequired<DominatorTreeWrapperPass>();    // requires_domtree = 1
    AU.addRequired<AssumptionCacheTracker>();
    AU.addPreserved<DominatorTreeWrapperPass>();   // invalidates_domtree = 0
    AU.setPreservesCFG();
}
```

By parsing all passes' `getAnalysisUsage` declarations, we can automatically populate the `requires_*` and `invalidates_*` dimensions. The `reduces_instructions` and `increases_instructions` dimensions are empirically computed: run the pass on a corpus of 10,000 IR functions and record the mean instruction count delta.

### Pass Similarity and Independence

The cosine distance between behavioral vectors measures how *different* two passes are:

```python
def pass_independence(v_A, v_B):
    # High independence = passes affect different IR properties
    # → can be composed in either order with minimal interference
    return 1.0 - cosine_similarity(v_A, v_B)
```

Passes with high independence (distance > 0.8) are good crossover boundary candidates: splicing two sequences at a boundary between independent passes produces offspring where each half can function correctly without the other half's context.

---

## 225.3 Pass Synergy Graphs

A *synergy graph* is a weighted directed graph where nodes are passes and edges encode empirically measured synergy relationships.

### Edge Semantics

An edge `A → B` with weight `w` means: "applying A *before* B produces w% more instruction count reduction than applying B alone (or A alone, or B before A)."

```python
def compute_synergy(pass_A, pass_B, ir_corpus):
    # Measure reduction from B alone
    r_B = mean([count_instructions(apply(B, f)) / count_instructions(f)
                for f in ir_corpus])
    # Measure reduction from A then B
    r_AB = mean([count_instructions(apply(B, apply(A, f))) / count_instructions(f)
                 for f in ir_corpus])
    # Synergy: how much A enables B
    return (r_B - r_AB) / r_B  # positive = A helps B
```

### Extracting the Graph from Empirical Data

```python
synergy = {}
for pass_A in pass_vocabulary:
    for pass_B in pass_vocabulary:
        w = compute_synergy(pass_A, pass_B, training_corpus)
        if w > SYNERGY_THRESHOLD:  # e.g., 0.05 = 5% benefit
            synergy[(pass_A, pass_B)] = w

# Convert to directed graph
G = nx.DiGraph()
G.add_nodes_from(pass_vocabulary)
for (A, B), w in synergy.items():
    G.add_edge(A, B, weight=w)
```

### Known High-Synergy Clusters

From empirical measurement on LLVM's test suite:

| Pass cluster | Synergy description |
|-------------|---------------------|
| `mem2reg → gvn → instcombine` | mem2reg promotes allocas to SSA; GVN eliminates redundancies; instcombine cleans up |
| `loop-rotate → loop-unroll` | rotation normalizes the loop header; unrolling depends on normalized form |
| `reassociate → instcombine → early-cse` | reassociate reveals canonicalization; instcombine applies it; CSE eliminates duplicates |
| `inliner → simplifycfg → dce` | inlining expands code; simplifycfg cleans dead branches; DCE removes unreachable code |
| `sroa → mem2reg → gvn` | SROA scalarizes aggregates; mem2reg promotes scalars; GVN eliminates redundant scalars |

The graph has ~200 nodes (one per in-tree pass) and ~800 high-synergy edges.

### Strongly Connected Components

Passes within the same SCC of the synergy graph are mutually beneficial and should be kept together:

```python
sccs = list(nx.strongly_connected_components(G))
# Largest SCCs typically contain the instcombine/simplifycfg/dce cluster
# These form the "LLVM canonicalization kernel"
```

---

## 225.4 Knowledge-Guided Genetic Operators

### Synergy-Guided Crossover

Standard crossover splices at a random position. Synergy-guided crossover splices at a *synergy edge boundary* — a position where the last pass of p1's prefix and the first pass of p2's suffix have high independence (low mutual synergy):

```python
def synergy_guided_crossover(p1, p2, synergy_graph):
    # Find all valid splice points: positions where p1[i] and p2[i+1]
    # have low synergy (high independence)
    valid_points = []
    for i in range(1, len(p1) - 1):
        if synergy_graph.get_edge_weight(p1[i], p2[i+1], default=0) < 0.02:
            valid_points.append(i)

    if not valid_points:
        return random_crossover(p1, p2)  # fallback

    # Select a random valid splice point (preferring points with high independence)
    splice = random.choice(valid_points)
    return p1[:splice] + p2[splice:]
```

### Prerequisite-Respecting Mutation

Before inserting a random pass, check that its prerequisites are met:

```python
def prerequisite_respecting_mutation(seq, behavioral_vectors, synergy_graph):
    operation = random.choice(["insert", "delete", "swap"])

    if operation == "insert":
        # Select a pass to insert
        new_pass = random.choice(pass_vocabulary)
        position = random.randint(0, len(seq))

        # Check: are all prerequisites of new_pass satisfied by seq[:position]?
        required = behavioral_vectors[new_pass]["requires"]
        provided = union([behavioral_vectors[p]["establishes"] for p in seq[:position]])
        if not required.issubset(provided):
            # Insert the prerequisite pass first (repair operator)
            prereq = find_prerequisite_for(required - provided)
            seq = seq[:position] + [prereq] + seq[position:]
            position += 1  # shift insertion point

        return seq[:position] + [new_pass] + seq[position:]

    # ... delete and swap operations
```

### Population Seeding from Known Good Sequences

Instead of random initialization, seed the population from LLVM's standard pipelines:

```python
seeds = [
    parse_pipeline_string("-O2"),        # standard -O2 pipeline
    parse_pipeline_string("-O3"),        # standard -O3 pipeline
    parse_pipeline_string("-Os"),        # size-optimizing pipeline
    parse_pipeline_string("-Oz"),        # aggressive size pipeline
    # Plus mutations of each seed
]
population = seeds + [mutate(random.choice(seeds)) for _ in range(96)]
```

This biases the search toward the region of pass-sequence space that LLVM's engineers have already found effective, rather than starting from random noise.

---

## 225.5 ECCO: Causal Reasoning for Pass Combination Explanation

ECCO (Explainable Compiler Optimization, arXiv 2602.00087) applies counterfactual causal analysis to answer: *which passes in a sequence actually contribute to the observed optimization?*

### Counterfactual Ablation

For a sequence `[p1, p2, ..., pN]` that achieves metric `M`:

```python
def ecco_ablate(sequence, ir_function, metric):
    contributions = {}
    baseline = evaluate(sequence, ir_function, metric)
    for i, pass_i in enumerate(sequence):
        ablated = sequence[:i] + sequence[i+1:]  # remove pass i
        ablated_score = evaluate(ablated, ir_function, metric)
        contributions[pass_i] = baseline - ablated_score  # marginal contribution
    return contributions
```

Passes with near-zero marginal contribution are redundant. Passes with high contribution are essential.

### Iterative Pruning

ECCO applies ablation iteratively, removing the lowest-contribution pass at each step:

```
Original: [mem2reg, gvn, instcombine, loop-rotate, loop-unroll, simplifycfg, dce]
          Score: 100 (baseline)

Step 1: ablate each pass, find "dce" contributes 0.3% → remove
        [mem2reg, gvn, instcombine, loop-rotate, loop-unroll, simplifycfg]
        Score: 99.7 (acceptable: within 0.5% tolerance)

Step 2: ablate each pass, find "loop-rotate" contributes 0.1% → remove
        [mem2reg, gvn, instcombine, loop-unroll, simplifycfg]
        Score: 99.6

Step 3: ablate each pass, find all remaining contribute > 1% → stop
        Final: [mem2reg, gvn, instcombine, loop-unroll, simplifycfg]
```

The pruned 5-pass sequence achieves 99.6% of the original 7-pass sequence's performance and is easier to verify (fewer passes = smaller Z3 search space for Alive2, cross-reference Chapter 223).

### Causal Graph Construction

ECCO also constructs a causal graph explaining *how* each pass enables subsequent passes:

```python
def ecco_causal_graph(sequence, ir_corpus):
    G = nx.DiGraph()
    for i, p_i in enumerate(sequence):
        for j, p_j in enumerate(sequence[i+1:], start=i+1):
            # Does removing p_i reduce p_j's effectiveness?
            without_pi = sequence[:i] + sequence[i+1:j+1]
            with_pi    = sequence[:j+1]
            delta = evaluate(with_pi[-1:], ir_corpus) - evaluate(without_pi[-1:], ir_corpus)
            if delta > CAUSAL_THRESHOLD:
                G.add_edge(p_i, p_j, weight=delta)
    return G
```

This causal graph captures not just pairwise synergy (which is symmetric) but directional enabling relationships.

---

## 225.6 The Hybrid Framework

The full knowledge-infused evolutionary pipeline combines all components:

```
1. INITIALIZATION: seed population from -O2/-O3/-Os pipelines + mutations

2. FITNESS EVALUATION: for each sequence in population
   a. apply to IR corpus
   b. measure metric (instruction count, estimated speedup, binary size)
   c. [optional] Alive2 correctness check (filter incorrect sequences)

3. SELECTION: tournament selection on fitness

4. REPRODUCTION:
   a. synergy-guided crossover (splice at independence boundaries)
   b. prerequisite-respecting mutation (insert/delete/swap with repair)

5. ECCO PRUNING: for top-10 sequences in generation
   a. iterative ablation to find minimal equivalent sequence
   b. [optional] Alive2 verify pruned sequence

6. EARLY TERMINATION: if best fitness unchanged for 20 generations, stop

7. LLM INITIALIZATION EXTENSION (arXiv 2510.14292):
   a. every 50 generations, call LLM with "suggest a novel pass combination for this IR type"
   b. add LLM-proposed sequences to the population as new individuals
   c. LLM diversifies search; GA refines
```

### Empirical Results (arXiv 2510.14292)

Evaluated on SPEC CPU 2006 with LLVM 17 IR:

| Method | Mean speedup vs `-O3` | Correctness |
|--------|----------------------|-------------|
| Random GA (baseline) | 7.3% | 94% |
| Synergy-guided GA | 11.1% | 97% |
| Synergy-guided + ECCO | 13.8% | 97% |
| + LLM initialization | 14.9% | 95% |
| + Alive2 filtering | 14.6% | 100% |

The Alive2 filtering step (Chapter 223) trades 0.3% speedup for 100% correctness — the combination is the production-ready configuration.

---

## 225.7 Integration with ORC ReOptimizeLayer

The evolutionary search is an offline process: it runs on a training corpus and produces a *pass policy* — a mapping from IR characteristics to effective pass sequences.

### Policy Representation

```python
# Policy: for each IR "fingerprint" → effective pass sequence
policy = {
    "small_loop_heavy":    ["loop-rotate", "loop-unroll", "instcombine"],
    "call_heavy":          ["inliner", "simplifycfg", "dce", "gvn"],
    "memory_heavy":        ["sroa", "mem2reg", "gvn", "early-cse"],
    "arithmetic_heavy":    ["reassociate", "instcombine", "sccp"],
}

def fingerprint_ir(module):
    loop_count = count_loops(module)
    call_count = count_calls(module)
    memory_ops = count_memory_ops(module)
    # ... classify into categories
    return classify(loop_count, call_count, memory_ops)
```

### ReOptimizeLayer Integration

At JIT compile time, the `ReOptimizeFunc` retrieves the policy and applies the evolutionary-search-optimized sequence:

```cpp
auto ReOptimize = [&Policy](llvm::orc::ReOptimizeLayer& ROL,
                             llvm::orc::ReOptimizeLayer::ReOptMaterializationUnitID MUID,
                             unsigned CurVersion,
                             llvm::orc::ThreadSafeModule& TSM) -> llvm::Error {
    return TSM.withModuleDo([&](llvm::Module& M) -> llvm::Error {
        // Get the offline-trained pass sequence for this IR type
        std::string fingerprint = fingerprintModule(M);
        std::string passSequence = Policy.lookup(fingerprint);

        // Apply the evolutionary-optimized pass sequence
        llvm::PassBuilder PB;
        llvm::ModulePassManager MPM;
        if (auto Err = PB.parsePassPipeline(MPM, passSequence))
            return Err;

        llvm::ModuleAnalysisManager MAM;
        PB.registerModuleAnalyses(MAM);
        MPM.run(M, MAM);
        return llvm::Error::success();
    });
};
```

The key advantage: the evolutionary search amortises its cost (hours of offline computation) across all future compilations of IR that matches the fingerprint. Each hot function recompilation at tier-2 pays only the cost of running the pre-computed pass sequence — zero evolutionary search at runtime.

---

## Chapter Summary

- Blind genetic operators (random crossover and mutation) waste generations on structurally invalid sequences; knowledge-infused operators encode pass prerequisites and synergy relationships directly into crossover and mutation
- Pass behavioral vectors characterize each pass by its IR property requirements and effects; cosine distance between vectors measures independence; independent passes are safe crossover boundary candidates
- Pass synergy graphs encode empirically measured enabling relationships: edge `A → B` with weight `w` means A improves B's effectiveness by w%; synergy is extracted by pairwise measurement on an IR corpus
- Knowledge-guided crossover splices at independence boundaries; prerequisite-respecting mutation inserts a repair pass when a selected insertion violates prerequisites
- ECCO uses counterfactual ablation to find the minimal equivalent subsequence: iteratively remove the lowest-marginal-contribution pass while staying within a 0.5% tolerance; the pruned sequence is easier to verify and deploy
- The hybrid pipeline (synergy-guided GA + ECCO + LLM initialization + Alive2 filtering) achieves 14.6% mean speedup vs `-O3` on SPEC CPU 2006 with 100% Alive2-verified correctness
- Offline evolutionary search produces a fingerprint→pass-sequence policy; `ReOptimizeLayer`'s `ReOptimizeFunc` retrieves and applies the policy at JIT tier-2 recompilation, amortising search cost across all future compilations of similar IR

---

@copyright jreuben11
