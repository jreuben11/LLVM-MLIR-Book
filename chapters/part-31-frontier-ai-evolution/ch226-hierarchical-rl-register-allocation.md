# Chapter 226 — Hierarchical Reinforcement Learning for Register Allocation and Code Optimization

*Part XXXI — Frontier AI Evolution*

Register allocation is one of the hardest problems in compiler backend design. It is NP-complete in general (equivalent to graph coloring), it interacts with instruction selection and scheduling, and its decisions cascade — spilling one live range changes the interference graph for all other live ranges. LLVM's Greedy allocator is a sophisticated heuristic that makes locally optimal decisions in a priority queue, but it cannot look ahead to see how a spill decision now affects options later in the allocation. Reinforcement learning offers a different approach: treat the entire allocation as a sequential decision process, train an agent that has seen thousands of allocation problems, and let it develop a policy that implicitly reasons about future consequences. This chapter covers the RL formulation, the RL4ReAL hierarchical decomposition, the Pearl generalized code optimization system, and the GNN-based IR representations that make these systems practical.

---

## 226.1 Register Allocation as an MDP

### Baseline: The Greedy Allocator

LLVM's Greedy register allocator (`lib/CodeGen/RegAllocGreedy.cpp`) processes live ranges in priority order. For each live range, it attempts to assign a physical register; if no register is available without evicting a higher-priority live range, it splits the range or spills it to memory.

The greedy approach makes one decision at a time based on local information. It cannot see that the spill decision it is about to make will cause a cascade of evictions that ultimately produce more total spills than an alternative first decision would have.

### MDP Formulation

The register allocation problem as an MDP:

- **State** `s_t`: the interference graph at step `t`, augmented with live range attributes
  - Nodes: unallocated live ranges
  - Edges: interference (two live ranges that are simultaneously live)
  - Node features: use frequency, spill cost, register class, loop depth, number of basic blocks spanning the range

- **Action** `a_t`: assign a physical register to the selected live range, or spill it
  - Discrete action space: |physical registers| + 1 (spill)
  - Typically 16–32 physical registers per class on x86-64

- **Transition**: update the interference graph after the assignment (remove the allocated node, update pressure)

- **Reward** `R`: final code quality metric
  - Primary: number of spill loads/stores inserted (minimize)
  - Secondary: register-to-register copy count
  - Sparse: received only after all live ranges are allocated

- **Episode**: allocate all live ranges in one function

### Why the MDP is Hard

1. **Variable state size**: different functions have different numbers of live ranges (10–10,000); the interference graph has no fixed dimension

2. **High action space at high register pressure**: when many live ranges compete for few registers, the number of valid actions is large

3. **Sparse reward**: the reward is the total spill count after all decisions — credit assignment to individual decisions is unclear

4. **Cascading effects**: spilling one live range reduces pressure for others, potentially avoiding future spills — but also adds memory traffic. The net effect depends on all future decisions.

---

## 226.2 The RL4ReAL Decomposition

RL4ReAL (Reinforcement Learning for Register Allocation, arXiv 2204.02013) observes that register allocation is not a single problem but three coupled sub-problems. Solving them jointly with one RL agent leads to a huge state-action space. Decomposing into three sub-agents — each solving one sub-problem — makes learning tractable.

### Sub-Problem 1: Live Range Splitting

Live range splitting divides a long live range into shorter, non-overlapping pieces. Shorter pieces are easier to color (lower interference) and cheaper to spill (spill around fewer uses).

The splitting agent decides:
- *Whether* to split a given live range
- *Where* to insert the split point (at which program point)

Benefit: splitting before coloring reduces interference, enabling more live ranges to be colored without spilling.

### Sub-Problem 2: Spill Decision

Given a set of live ranges that cannot be simultaneously colored, which live range(s) should be spilled to memory?

The spill agent decides:
- Which live range to spill
- Whether to spill the entire range or only split and spill a specific region

Spill cost depends on use frequency, loop depth, and the relative cost of register-to-register moves vs memory traffic.

### Sub-Problem 3: Coloring

Given split live ranges and spill decisions, assign physical registers to the remaining live ranges. This is equivalent to graph coloring on the interference graph.

The coloring agent decides:
- Which physical register to assign to each uncolored live range
- Which evictions are acceptable (Kempe's theorem: a node with degree < k can always be colored; nodes with degree ≥ k may need eviction)

### Hierarchical Agent Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Manager Policy π_M                    │
│  Input: interference graph embedding (GNN)               │
│  Output: which sub-agent to activate next, and for which │
│          live range                                      │
└────────────────────────────────────────────────────────-┘
              │                 │                │
              ▼                 ▼                ▼
     ┌──────────────┐  ┌─────────────┐  ┌─────────────┐
     │ Splitting     │  │ Spill        │  │ Coloring     │
     │ Agent π_S    │  │ Agent π_K   │  │ Agent π_C   │
     │              │  │             │  │              │
     │ Input: LR    │  │ Input: LR   │  │ Input: LR   │
     │ features +   │  │ features +  │  │ features +  │
     │ program      │  │ interference│  │ available   │
     │ context      │  │ graph       │  │ registers   │
     └──────────────┘  └─────────────┘  └─────────────┘
```

The manager coordinates: it selects which sub-agent to activate and on which live range, based on the current interference graph state. Sub-agents are specialized and shallow — they receive a focused view of the current problem rather than the full state.

### Reward Structure

- **Sub-agent local rewards**: each sub-agent receives a shaped reward for its immediate decisions
  - Splitting agent: −0.1 × splits_introduced (penalize excessive splits)
  - Spill agent: −1.0 × spills_introduced + 0.5 × evictions_avoided
  - Coloring agent: +0.3 × colors_assigned (progress reward)
- **Manager global reward**: total spill count at episode end (the true objective)

The shaped rewards reduce variance during training without biasing the policy toward suboptimal solutions (as long as the shaping function is *potential-based*).

---

## 226.3 Hierarchical Agent Architecture in Detail

### GNN for Interference Graph Encoding

Both the manager and the sub-agents receive the interference graph as a Graph Neural Network embedding.

```python
import torch
import torch_geometric

class InterferenceGNN(torch_geometric.nn.MessagePassing):
    def __init__(self, node_features=12, hidden=64, layers=3):
        super().__init__(aggr='mean')
        self.encoder = nn.Linear(node_features, hidden)
        self.gnn_layers = nn.ModuleList([
            torch_geometric.nn.GCNConv(hidden, hidden) for _ in range(layers)
        ])
        self.decoder = nn.Linear(hidden, hidden)

    def forward(self, x, edge_index):
        x = F.relu(self.encoder(x))
        for layer in self.gnn_layers:
            x = F.relu(layer(x, edge_index))
        return self.decoder(x)  # per-node embeddings
```

Node features per live range:
1. Use count (total uses across all blocks)
2. Loop nest depth of dominant use
3. Spill weight (LLVM's computed heuristic spill cost)
4. Number of basic blocks spanning the range
5. Register class (integer, float, vector) as one-hot
6. Is rematerializable (can be recomputed without a spill load)
7. Number of interfering live ranges (local pressure)
8. Copy hint (is this live range coalesced with another)
9. Is call-crossing (live across a function call — expensive)
10. Definition type (constant, memory load, computation)
11. Degree in interference graph
12. Priority rank in Greedy allocator's queue

### Policy Networks

Each agent's policy is a small MLP on top of the GNN embeddings:

```python
class SpillAgent(nn.Module):
    def __init__(self, gnn_hidden=64):
        super().__init__()
        self.gnn = InterferenceGNN(hidden=gnn_hidden)
        self.policy_head = nn.Sequential(
            nn.Linear(gnn_hidden, 32),
            nn.ReLU(),
            nn.Linear(32, 2),  # [spill, keep]
        )

    def forward(self, x, edge_index, candidate_lr_idx):
        embeddings = self.gnn(x, edge_index)
        lr_embedding = embeddings[candidate_lr_idx]
        return F.softmax(self.policy_head(lr_embedding), dim=-1)
```

---

## 226.4 Pearl: Deep RL for General Code Optimization

Pearl (arXiv 2506.01880) generalizes RL4ReAL's hierarchical approach to the entire code optimization pipeline, not just register allocation.

### Architecture

```
High-level Policy (Manager):
  Input: IR summary (function-level features + hot-path statistics)
  Action: select which optimization to apply (inlining, vectorization,
           unrolling, register allocation strategy, instruction scheduling)

Low-level Policy (Worker):
  Input: local context of the selected optimization
  Action: set parameters of the selected optimization
          (inline threshold, vector width, unroll factor, eviction strategy)
```

### The Multi-Level MDP

State decomposition:
- **High-level state** `s^H_t`: summary of the entire IR module (call graph depth, loop count, hotness distribution, estimated register pressure)
- **Low-level state** `s^L_t`: local view relevant to the selected optimization (interference graph for allocation, loop structure for vectorization)

Action hierarchy:
- **High-level action** `a^H_t`: select optimization type
- **Low-level action** `a^L_t`: select parameters for the chosen optimization

Reward:
- **High-level reward**: global compilation quality (binary size, estimated runtime)
- **Low-level reward**: shaped immediate reward for the selected optimization

### Pearl vs Individual-Heuristic RL

The key advantage of Pearl over separate RL agents for each heuristic: **joint optimization across heuristics**. Inlining decisions affect register pressure, which affects allocation decisions, which affect instruction scheduling decisions. Pearl's high-level policy can learn to inline less aggressively when register pressure is already high — a cross-heuristic relationship that individual agents cannot model.

### Empirical Results on Chromium

Pearl trained on Chromium build artifacts (thousands of IR files from real browser code):

| Metric | Pearl vs `-O3` |
|--------|---------------|
| Binary size reduction | 3.1–4.8% |
| Compile time overhead | 8% (GNN inference) |
| Correctness | 100% (no semantic bugs observed) |
| Training time | 72 hours on 8 GPU cluster |

---

## 226.5 GNN-Based IR Representation: ProGraML

The choice of IR representation for the GNN has a large impact on policy quality. Three representations are commonly used:

### 1. Use-Def Graph (LLVM DataFlowGraph)

```
Nodes: instructions (each SSA value)
Edges: def→use edges (SSA def edges)
Node features: opcode, type, constant flag
```

Simple and efficient, but misses control flow structure (no edges across basic blocks for non-SSA values).

### 2. ProGraML (Cummins et al., arXiv 1906.00148)

ProGraML constructs a unified graph with three edge types:

```
Nodes: instructions + variables
Edges:
  Control edges: predecessor→successor basic block relationships
  Data edges: def→use relationships (SSA edges)
  Call edges: caller→callee relationships
```

ProGraML encodes control, data, and call flow in a single graph, giving the GNN a complete view of the IR. It has shown strong results on downstream tasks (algorithm classification, optimisation prediction) across multiple compiler IRs.

```python
def build_programl_graph(function):
    G = nx.MultiDiGraph()
    for bb in function.basic_blocks:
        for inst in bb.instructions:
            G.add_node(inst.id, features=encode_instruction(inst))
        # Control edges (CFG)
        for succ in bb.successors:
            G.add_edge(bb.terminator.id, succ.first_instruction.id,
                       edge_type=0)  # control
        # Data edges (use-def)
        for inst in bb.instructions:
            for operand in inst.operands:
                if isinstance(operand, llvm.Instruction):
                    G.add_edge(operand.id, inst.id, edge_type=1)  # data
    # Call edges
    for call_inst in function.call_instructions:
        G.add_edge(call_inst.id, call_inst.callee.entry.id, edge_type=2)
    return G
```

### 3. MLIR Region Tree

For MLIR-based systems, the region tree provides a hierarchical graph:

```
Nodes: operations (at any dialect level)
Edges: def-use edges + containment edges (op → region → block → op)
```

The containment hierarchy naturally represents loop nesting, function scoping, and module structure — information that flat LLVM IR graphs encode implicitly through dominance relationships.

---

## 226.6 Training Infrastructure and Reward Shaping

### Distributed Episode Collection

Training requires millions of allocation episodes. A distributed setup:

```
┌──────────────────────────────────────────────────────┐
│  Parameter Server (policy network parameters)         │
│      ↑ gradient updates  ↓ policy parameters          │
│                                                       │
│  Workers (N=64):                                      │
│    - Each worker runs opt on IR files from corpus     │
│    - Invokes RL agent at each allocation decision     │
│    - Collects (state, action, reward) trajectories    │
│    - Sends trajectories to parameter server           │
└──────────────────────────────────────────────────────┘
```

Each worker is a modified `opt` that calls back into the RL agent at each decision point. The IR corpus is sampled from open-source projects (Linux kernel, LLVM itself, Chromium).

### Curriculum Learning

Training starts on small functions (< 50 live ranges) and gradually increases size:

```
Phase 1 (episodes 0–10K):     functions with < 50 LRs
Phase 2 (episodes 10K–50K):   functions with < 200 LRs
Phase 3 (episodes 50K–200K):  functions with < 1000 LRs
Phase 4 (episodes 200K–500K): production-size functions
```

Curriculum prevents the agent from being overwhelmed by large interference graphs before it has learned basic allocation strategies.

### Reward Shaping Details

Potential-based reward shaping preserves the optimal policy:

```python
def shaped_reward(state_t, state_{t+1}, action, final_reward=None):
    if final_reward is not None:
        # Terminal reward: negative of total spill count
        return -final_reward

    # Intermediate shaping: potential function
    phi_t = -count_high_pressure_nodes(state_t)   # pressure before
    phi_{t+1} = -count_high_pressure_nodes(state_{t+1})  # pressure after
    shaping = GAMMA * phi_{t+1} - phi_t
    return shaping  # ∈ [-0.5, 0.5] typically
```

Potential-based shaping (Ng et al. 1999) guarantees that the shaping function does not change the optimal policy — it only reduces variance.

---

## 226.7 Comparison with Classical Approaches

### PBQP (Pseudo-Boolean Quadratic Problem)

PBQP models register allocation as a quadratic optimization problem over a constraint graph:

- Each node: a live range with options (physical registers + spill)
- Each edge: cost of assigning two interfering live ranges to the same or different registers
- Objective: minimize total cost (spill cost + copy cost)

PBQP finds the *optimal* allocation for the modeled costs — but the model is an approximation, and PBQP is NP-hard (solved by local search for large instances). On small functions PBQP often outperforms Greedy; on large functions it falls back to heuristics.

RL4ReAL vs PBQP: RL can scale to arbitrary sizes (bounded only by GNN forward pass time) but is approximate; PBQP is exact for small instances but degrades to heuristic on large ones.

### MLGO Eviction Advisor (Chapter 224)

The eviction advisor makes a single spill decision at a time based on local features. It cannot see the global allocation state. RL4ReAL's spill agent has access to the full interference graph embedding, giving it a global view. The trade-off: MLGO's advisor has < 1 ms inference; RL4ReAL's spill agent with GNN inference takes ~5 ms.

### Greedy Allocator with ML Eviction Advisor

| Property | Greedy | ML Eviction | RL4ReAL |
|----------|--------|-------------|---------|
| Decision scope | Local | Local | Global (GNN) |
| Look-ahead | None | None | Implicit (trained) |
| Spill reduction vs Greedy | baseline | 1–2% | 3–5% |
| Inference latency | 0 ms | < 1 ms | 5–15 ms |
| Production deployment | Yes (LLVM default) | Yes (MLGO, optional) | Research |

### Production vs Research Status

MLGO's eviction advisor is deployed in production at Google (optional, enabled by `--regalloc-enable-advisor=release`). RL4ReAL is a research system. The boundary: production requires < 1 ms overhead and 100% correctness; RL4ReAL's 5–15 ms GNN inference is acceptable only in ahead-of-time compilation settings where compilation time is not critical (e.g., offline model compilation, not interactive development).

---

## Chapter Summary

- Register allocation is an NP-complete, cascading decision problem; the Greedy allocator makes locally optimal decisions but cannot reason about future consequences; RL offers global reasoning at the cost of approximate decisions
- RL4ReAL decomposes allocation into three sub-MDPs: live-range splitting, spill decision, and coloring; a manager policy coordinates three specialized sub-agents; hierarchical decomposition makes the joint MDP tractable
- The GNN encodes the interference graph as per-node embeddings; node features include use frequency, loop depth, spill weight, and interference degree; ProGraML provides a richer representation with control, data, and call edges
- Pearl generalizes the hierarchical RL framework to the full optimization pipeline: a high-level policy selects which optimization to apply; a low-level policy sets parameters; joint optimization captures cross-heuristic interactions (e.g., inlining aggressiveness conditional on register pressure)
- Curriculum learning (small functions → large functions) and potential-based reward shaping are essential for stable training; distributed episode collection (64 workers sampling IR from open-source projects) provides the training data scale needed
- Comparison: PBQP is optimal for small instances, degrades for large; MLGO eviction advisor has < 1 ms latency (production-ready) but local scope; RL4ReAL achieves 3–5% spill reduction with 5–15 ms latency (research, suitable for AOT compilation)

---

@copyright jreuben11
