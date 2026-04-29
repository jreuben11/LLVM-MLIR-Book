# Chapter 158 — SPMD, GSPMD, and Auto-Sharding

*Part XXII — XLA and the OpenXLA Stack*

Training modern large language models requires distributing computation across hundreds or thousands of accelerators. XLA's approach to multi-device execution — the SPMD (Single Program Multiple Data) partitioner, GSPMD (Generalized SPMD), and the ILP-based auto-sharding pass — automates the insertion of communication primitives and the partitioning of tensors across device meshes. This chapter covers the `HloSharding` annotation system, the SPMD partitioner pass, GSPMD's generalization to arbitrary topologies, collective operations, and the auto-sharding pass that uses an integer linear program to find optimal tensor distributions.

---

## 158.1 The Distribution Problem

A typical transformer model with 70 billion parameters cannot fit in the 80 GB memory of a single A100 GPU. Distributed training strategies address this:

| Strategy | What is distributed | XLA mechanism |
|----------|---------------------|---------------|
| Data parallelism | Input batch | `replica_count`, gradient AllReduce |
| Tensor (model) parallelism | Weight matrices | `HloSharding::Tile` on parameters |
| Pipeline parallelism | Model layers | Manual stage annotation |
| Expert parallelism (MoE) | Expert weights | `HloSharding::Tile` with custom routing |
| Fully sharded (ZeRO-3) | Parameters + optimizer states | `HloSharding::Tile` + `ReduceScatter` |

XLA's SPMD partitioner automates tensor and expert parallelism given `HloSharding` annotations. Data parallelism is handled by `replica_count` and automatic gradient AllReduces.

---

## 158.2 HloSharding: Tensor Distribution Annotations

`HloSharding` is XLA's representation of how a tensor is distributed across a set of devices. The representation is defined in [`xla/hlo/ir/hlo_sharding.h`](https://github.com/openxla/xla/blob/main/xla/hlo/ir/hlo_sharding.h).

### 158.2.1 Replicated Sharding

The simplest sharding: the same tensor exists on every device:

```
HloSharding::Replicate()
Text: {replicated}
```

Used for: small weight tensors shared across all devices, scalar loss values.

### 158.2.2 Tiled Sharding

A tensor is split along one or more dimensions across a device mesh:

```
// 8-device tensor parallel: split the weight [1024, 8192] along dim 1
// Device mesh is flat [8]
HloSharding::Tile(/*tile_assignment=*/Array<int64_t>({8}))
Text: {devices=[8]0,1,2,3,4,5,6,7}
```

For a 2D mesh (e.g., 4 nodes × 8 GPUs per node):

```
// 32-device mesh: split [batch, seq, hidden] along batch and hidden dims
HloSharding::Tile(Array<int64_t>({4, 8}))
Text: {devices=[4,8]0,1,2,3,...,31}
```

A `[batch=64, seq=2048, hidden=8192]` activation sharded over a `[4,8]` mesh:
- Dimension 0 (batch) split across 4 nodes: each node handles batch/4 = 16 samples
- Dimension 2 (hidden) split across 8 GPUs per node: each GPU handles hidden/8 = 1024 features
- Dimensions 1 (seq) is not split: each device has the full sequence

### 158.2.3 Partial Replication (Hybrid)

A combination of tiling and replication:

```
// 32 devices, tensor split across 4, replicated 8 times
HloSharding::PartialTile(Array<int64_t>({4, 8}))
Text: {devices=[4,8]0,...,31 last_tile_dim_replicate}
```

Used in ZeRO-3 / FSDP: parameters are sharded, but a subset of devices holds the same shard.

### 158.2.4 Manual Sharding

`HloSharding::Manual()` marks a tensor as manually partitioned — the user takes full responsibility. The SPMD partitioner does not touch manually sharded ops; they are emitted as-is.

---

## 158.3 The SPMD Partitioner Pass

`SpmdPartitioner` is an `HloModulePass` that transforms a single-device HLO program annotated with `HloSharding` into a multi-device program:

```
Input: HloModule with HloSharding annotations
    │
    ▼
SpmdPartitioner::Run()
    ├── Propagate shardings to all instructions
    ├── For each instruction:
    │   ├── If all operands and output have consistent tiled shardings:
    │   │   → emit the instruction on the local shard
    │   ├── If shardings are incompatible:
    │   │   → insert collective communication (AllReduce, AllGather, etc.)
    │   └── If output sharding requires redistribution:
    │       → insert resharding ops
    └── Replace HloSharding annotations with actual partitioned ops
    │
    ▼
Output: HloModule running on 1/N the data, with explicit collectives
```

### 158.3.1 Sharding Propagation

Before partitioning, `ShardingPropagation` infers shardings for all unannotated instructions. The rules are intuitive:

- Element-wise ops: output sharding = input sharding (same tiling)
- `kDot`: if LHS is tiled on batch dim and RHS is tiled on batch dim, output is tiled on batch dim
- `kReduce` reducing over a sharded dim: output is replicated (must AllReduce after)
- `kTranspose`: sharding permuted by the transpose permutation

### 158.3.2 Dot Partitioning

The most important case is distributing `kDot` operations. Consider `[M, K] x [K, N] → [M, N]`:

**Case 1: LHS is sharded on M (split batch)**
```
Each device owns [M/d, K] of LHS, full [K, N] of RHS
→ Compute [M/d, N] locally (no communication)
→ Result is tiled on M dimension
```

**Case 2: RHS is sharded on N (tensor parallel, column-wise)**
```
Each device owns full [M, K] of LHS, [K, N/d] of RHS
→ Compute [M, N/d] locally (no communication)
→ Result is tiled on N dimension
```

**Case 3: Contracting dimension K is sharded (weight-stationary)**
```
Each device owns [M, K/d] of LHS, [K/d, N] of RHS
→ Compute partial [M, N] locally
→ AllReduce over the d-way split K
→ Result is replicated
```

The SPMD partitioner selects the strategy based on the `HloSharding` of operands and output.

### 158.3.3 Reduce Partitioning

For `kReduce` on a sharded tensor:

```
// Input: [batch=64, hidden=1024], sharded on batch (4 devices, 16 each)
// Reduce over batch dimension

// Each device reduces its local shard:
local_result = reduce(local_shard)  // [hidden=1024]

// AllReduce across devices:
result = all-reduce(local_result, op=SUM)  // [hidden=1024]
```

If the reduce dimension is not sharded, each device performs the full reduction independently (no communication needed).

---

## 158.4 GSPMD: Generalized SPMD

GSPMD (Generalized SPMD) extends the SPMD model to handle arbitrary device mesh topologies and more complex sharding patterns. It was introduced in the paper "GSPMD: General and Scalable Parallelism for ML Computation Graphs" (Xu et al., 2021) and is the basis for Google's large-scale LLM training infrastructure.

### 158.4.1 Multi-Axis Meshes

GSPMD introduces the concept of named mesh axes. A 2D mesh `[data=4, model=8]` (32 devices total) allows independent sharding along each axis:

```python
# JAX GSPMD usage
import jax
from jax.sharding import Mesh, PartitionSpec as P

# Create a 4x8 device mesh
devices = jax.devices()
mesh = Mesh(np.array(devices).reshape(4, 8), ('data', 'model'))

# Annotate a weight matrix: sharded on 'model' axis
weight_sharding = jax.sharding.NamedSharding(mesh, P(None, 'model'))
# [hidden=8192, ffn=32768] → each device holds [8192, 4096]

# Annotate activations: sharded on 'data' axis  
activation_sharding = jax.sharding.NamedSharding(mesh, P('data', None))
```

The JAX `Mesh` and `PartitionSpec` map to `HloSharding::Tile` with the appropriate tile assignment array.

### 158.4.2 Sharding Propagation Through Complex Ops

GSPMD handles ops that simple SPMD cannot:

**Attention**:
```
Q: [batch/data, seq, heads/model, head_dim]
K, V: [batch/data, seq, heads/model, head_dim]
→ Attention scores: [batch/data, heads/model, seq, seq]
→ AllGather over seq dimension needed for full-seq attention
→ Or use ring attention to avoid the AllGather
```

**Expert routing (MoE)**:
```
Token routing: [batch/data, seq, num_experts] where tokens are assigned to experts
Expert weights: [num_experts/model, expert_hidden]
→ AllToAll to redistribute tokens to their assigned expert's device
→ Compute expert forward pass locally
→ AllToAll back to original token ordering
```

### 158.4.3 The `use_spmd_partitioning` Flag

GSPMD is enabled by setting:

```cpp
HloModuleConfig config;
config.set_use_spmd_partitioning(true);
config.set_num_partitions(32);  // total devices
```

With `num_partitions > 1`, XLA runs `SpmdPartitioner` before backend codegen.

---

## 158.5 Collective Communication Operations

### 158.5.1 AllReduce

AllReduce reduces a tensor across all devices in a group and broadcasts the result:

```
HLO: %result = all-reduce(%partial_sum),
         to_apply=%add_computation,
         replica_groups={{0,1,2,3}}

Runtime: NCCL ncclAllReduce(buf, buf, count, NCCL_FLOAT, ncclSum, comm, stream)
```

Ring AllReduce (NCCL's default) has communication volume O(2 × N × (d-1)/d) where N is tensor size and d is the number of devices — nearly optimal.

### 158.5.2 AllGather

AllGather collects sharded tensors from all devices:

```
// Device 0 has A[0:256], device 1 has A[256:512], ..., device 3 has A[768:1024]
%result = all-gather(%shard), dimensions={0}, replica_groups={{0,1,2,3}}
// Each device now has A[0:1024]
```

Used in ZeRO-1 to collect parameters before a forward pass.

### 158.5.3 ReduceScatter

ReduceScatter combines allreduce semantics with sharding — each device gets a different slice of the reduced result:

```
// Each device has a full gradient [1024]
// After ReduceScatter, device i has sum(gradient)[i*256:(i+1)*256]
%result = reduce-scatter(%grad),
    to_apply=%add_computation,
    scatter_dimension=0,
    replica_groups={{0,1,2,3}}
```

Used in ZeRO-2/ZeRO-3 for parameter shard updates.

### 158.5.4 AllToAll

AllToAll permutes tensor slices: each device sends different data to different devices:

```
// Each device has 4 expert inputs to route
%result = all-to-all(%tokens),
    split_dimension=0,  // split along expert dimension
    concat_dimension=0,
    split_count=4,
    replica_groups={{0,1,2,3}}
```

Used for Mixture-of-Experts (MoE) token routing.

---

## 158.6 Auto-Sharding: ILP-Based Optimal Distribution

Manual sharding annotation is tedious for complex models. The `AutoSharding` pass finds optimal shardings automatically using an Integer Linear Program (ILP).

### 158.6.1 Problem Formulation

Auto-sharding models the problem as:

**Decision variables**: For each HLO instruction, a binary variable `x_{i,s}` where `s` is a candidate sharding strategy.

**Cost terms**:
1. **Communication cost**: AllReduce/AllGather cost for resharding between incompatible shardings
2. **Memory cost**: maximum memory usage summed over all devices
3. **Recomputation cost**: for gradient checkpointing decisions

**Objective**: minimize total cost = α × communication_cost + β × memory_cost

**Constraints**:
- Exactly one sharding selected per instruction: Σ_s x_{i,s} = 1
- Aliased tensors have the same sharding: x_{i,s} = x_{j,s'}
- Tuple element shardings must be consistent

### 158.6.2 ILP Solver Integration

XLA uses OR-Tools (Google's operations research library) as the ILP solver:

```cpp
// From xla/service/spmd/auto_sharding.cc
operations_research::MPSolver solver("auto_sharding",
    operations_research::MPSolver::GLOP_LINEAR_PROGRAMMING);

// Variables: binary selector per instruction per strategy
for (auto& inst : instructions) {
  for (auto& strategy : strategies[inst]) {
    auto* var = solver.MakeBoolVar(strategy.name);
    x[inst][strategy] = var;
  }
}

// Objective: minimize communication + memory cost
auto* obj = solver.MutableObjective();
for (auto& inst : instructions) {
  for (auto& strategy : strategies[inst]) {
    double cost = strategy.communication_cost + alpha * strategy.memory_cost;
    obj->SetCoefficient(x[inst][strategy], cost);
  }
}

// Solve
solver.Solve();
```

### 158.6.3 Strategy Enumeration

For each instruction, candidate strategies are enumerated:

```cpp
// For a kDot [M, K] x [K, N] → [M, N] with d=4 devices:
Strategies for dot:
  S0: replicated × replicated → replicated (no comm, 4× memory)
  S1: tiled(M) × replicated → tiled(M) (AllGather needed if M-sharded LHS)
  S2: replicated × tiled(N) → tiled(N) (no comm, split N)
  S3: tiled(K) × tiled(K) → replicated + AllReduce (AllReduce cost)
  S4: tiled(M) × tiled(N) → tiled(M,N) (no contraction split, no comm)
```

Communication costs are estimated using bandwidth models for the network topology (NVLink, InfiniBand, etc.).

### 158.6.4 Limitations and Practical Use

The ILP formulation can be slow for large models (millions of variables). Practical mitigations:
- **Graph clustering**: cluster the HLO module into segments and solve per-segment
- **Heuristic seeding**: warm-start the ILP with a data-parallel sharding
- **Time limit**: abort ILP and use the best feasible solution found

JAX exposes auto-sharding through `jax.experimental.auto_sharding`:

```python
import jax
from jax.experimental import auto_sharding

mesh = Mesh(jax.devices(), ('devices',))

@jax.jit
@auto_sharding.auto_sharding(mesh)
def train_step(params, batch):
    # Shardings are determined automatically
    return loss(params, batch)
```

---

## 158.7 JAX Sharding APIs

JAX's sharding model provides a clean interface to XLA's SPMD infrastructure.

### 158.7.1 PartitionSpec and NamedSharding

```python
import jax
import jax.numpy as jnp
from jax.sharding import Mesh, PartitionSpec as P, NamedSharding

# 2D mesh: 4 nodes × 2 GPUs
devices = np.array(jax.devices()).reshape(4, 2)
mesh = Mesh(devices, ('batch', 'model'))

# Sharding specs
batch_sharding = NamedSharding(mesh, P('batch', None))  # split batch
model_sharding = NamedSharding(mesh, P(None, 'model'))  # split model dim

# Move data to devices with specific sharding
large_matrix = jnp.ones((1024, 512))
sharded_matrix = jax.device_put(large_matrix, model_sharding)
```

### 158.7.2 jax.jit with Sharding Constraints

```python
@jax.jit
def transformer_layer(params, x):
    # Declare that x is sharded on batch dimension
    x = jax.lax.with_sharding_constraint(x, batch_sharding)
    
    # Declare that weight is sharded on model dimension  
    w = jax.lax.with_sharding_constraint(params['w'], model_sharding)
    
    # XLA will insert AllReduce after this dot (contracting on sharded dim)
    return x @ w

# SPMD partitioner infers communication primitives
result = transformer_layer(params, x)
```

### 158.7.3 shard_map for Manual Control

`jax.experimental.shard_map` gives explicit per-shard control:

```python
from jax.experimental.shard_map import shard_map

@functools.partial(
    shard_map,
    mesh=mesh,
    in_specs=(P('batch', None), P(None, 'model')),
    out_specs=P('batch', 'model'),
    check_rep=False)
def matmul_collective(x, w):
    # x is [local_batch, k], w is [k, local_n]
    # Compute local partial result and AllReduce
    local = x @ w
    return jax.lax.psum(local, axis_name='model')
```

---

## 158.8 Collective Performance Optimization

### 158.8.1 Topology-Aware Replica Groups

NVLink is faster than PCIe; IB NIC is slower than both. XLA allows specifying replica groups that match hardware topology:

```
// 8 GPUs: 4 pairs with NVLink within each node
// AllReduce first within each NVLink pair, then across nodes
%local_reduce = all-reduce(%grad), replica_groups={{0,1},{2,3},{4,5},{6,7}}
%global_reduce = all-reduce(%local_reduce), replica_groups={{0,2,4,6},{1,3,5,7}}
```

### 158.8.2 Async Collectives

XLA 2024+ supports fully asynchronous collective communication that overlaps with compute:

```
HLO:
%start = all-reduce-start(%grad), replica_groups={{0,1,2,3}}
%next_compute = dot(%activations, %weights)  // runs while AR is in flight
%done = all-reduce-done(%start)
```

### 158.8.3 NCCL Communication Groups

XLA creates NCCL communicators at compile time based on the replica groups in `kAllReduce` instructions:

```cpp
// From xla/service/gpu/nccl_api.cc
NcclComm::Ptr NcclComm::Create(
    int nranks, const NcclUniqueId& id, int rank) {
  ncclComm_t comm;
  ncclCommInitRank(&comm, nranks, id, rank);
  return std::make_unique<NcclComm>(comm);
}
```

---

## Chapter Summary

- `HloSharding` annotates tensors as `Replicated`, `Tile` (split along one or more dims), `PartialTile` (split + replicated), or `Manual`; the tile assignment array maps logical shards to physical devices.
- `SpmdPartitioner` transforms a single-device annotated HLO into a partitioned multi-device HLO by splitting ops over local shards and inserting collectives (AllReduce, AllGather, ReduceScatter, AllToAll) where shardings are incompatible.
- GSPMD generalizes SPMD to named multi-axis device meshes; it handles complex patterns like MoE expert routing and multi-dimensional tensor parallelism.
- The four collective ops cover all distribution patterns: AllReduce (sum across devices), AllGather (collect shards), ReduceScatter (sum + scatter), AllToAll (permute).
- `AutoSharding` formulates sharding selection as an ILP — minimizing communication + memory cost — using OR-Tools; practical use requires time limits and heuristic seeding.
- JAX exposes the system via `NamedSharding`, `PartitionSpec`, `device_put`, `with_sharding_constraint`, and `shard_map`; these map directly to `HloSharding` annotations.
- Async collectives (`all-reduce-start` / `all-reduce-done`) overlap communication with computation; topology-aware replica groups exploit NVLink vs. IB bandwidth differences.
