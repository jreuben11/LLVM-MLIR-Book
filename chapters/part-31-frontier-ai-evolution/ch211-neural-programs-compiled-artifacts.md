# Chapter 211 — Neural Programs as Compiled Artifacts: The Self-Aware Execution Stack

*Part XXXI — Frontier AI Evolution*

Every neural network inference call is, at bottom, the execution of a compiled program. The weights are a data segment; the computation graph is a code section; the entire compiler chain — JAX tracing through XLA, MLIR lowering, LLVM code generation, and PTX assembly — is the toolchain that produced the executable. This view is not a metaphor. SafeTensors files have the same structural anatomy as ELF object files: a structured header describing layout, followed by a raw data buffer. `jax.make_jaxpr(model)(x)` returns the model's forward pass as a first-class Python object — homoiconic IR, inspectable and rewritable. `jax.jit` compiles that IR and installs the result in a cache keyed on computation graph shape, exactly as a JIT compiler installs compiled code in a code cache.

The significance for self-improving systems is precise: **the compiler is available at runtime**. A conventional compiled C program ships without its compiler; the compilation artifact (the binary) is fixed. A JAX model ships with the full JAX/XLA/MLIR/LLVM toolchain available and reachable from Python. When the system modifies its own weights or architecture, the toolchain is standing by to recompile the modified computation. This chapter makes the analogy exact at each level of the IR chain, examines weight checkpoints as structured object files, traces how XLA HLO achieves homoiconicity, and describes the full infer → profile → introspect → modify → recompile feedback loop. It is the bridge between the compiler infrastructure of Chapters 208–210 and the cognitive self-modification techniques of Chapters 212–218.

Cross-references: [Chapter 108 — The ORC JIT](../part-16-jit-sanitizers/ch108-the-orc-jit.md) · [Chapter 154 — HLO and StableHLO](../part-22-xla-openxla/ch154-hlo-and-stablehlo.md) · [Chapter 163 — Triton: A Compiler for GPU Kernels](../part-23-mlir-production/ch163-triton-a-compiler-for-gpu-kernels.md) · [Chapter 207 — Reflective Code, Open Problems, and Build Roadmap](../part-30-AI-first-PL-design/ch207-reflective-code-open-problems-roadmap.md) · [Chapter 208 — GPU Kernel DSLs](ch208-gpu-kernel-dsls-triton-helion-gluon.md) · [Chapter 209 — CuTe, Thrust, and TileIR](ch209-cutlass-thrust-cute-tileir.md) · [Chapter 210 — The JAX Ecosystem](ch210-jax-ecosystem-functional-neural-compilation-stack.md) · [Chapter 212 — Weights as Programming Substrate](ch212-weights-as-programming-substrate.md) · [Chapter 213 — Mechanistic Interpretability and SAE Features](ch213-mechanistic-interpretability-sae-features.md)

---

## 211.1 The Stack Is the Substrate

The conventional understanding of a compiled program distinguishes three phases that are cleanly separated in time: compilation (source → binary), loading (binary → process memory), and execution (instructions → side effects). The compiler is a development-time artifact; the binary is a deployment-time artifact; the runtime is an execution-time artifact. Self-modification in this model means either rewriting source and recompiling (expensive, requires the toolchain at runtime) or patching binary instructions (fragile, architecture-specific, rarely correct for complex programs).

Neural networks dissolve this three-phase separation at every boundary.

**The source is always available.** JAX model definitions are Python source. Python is an interpreted language with a persistent runtime. The source code of a Flax NNX module is not compiled away — it is executed by the Python interpreter on every call to `model.apply`, and the JAX tracer intercepts the execution to extract the computation graph. Modifying the model definition means editing a Python class, which takes effect immediately on the next `jax.jit` call.

**The compiler is always available.** `jax.jit`, `jax.export`, and `mlir.passmanager` are ordinary Python imports. The full JAX→XLA→MLIR→LLVM→PTX compilation chain executes inside the Python process that also runs inference. Recompilation costs single-digit seconds for typical transformer layers; JAX's persistent compilation cache (`JAX_COMPILATION_CACHE_DIR`) reduces recompilation to cache lookup for previously seen computation shapes.

**The binary is mutable data.** The "binary" of a neural network — its weights — are floating-point arrays in device memory or memory-mapped files on disk. JAX's `jax.tree_util.tree_map` transforms them as pure data. Orbax writes and reads them with the same versioned checkpoint API. They are not marked executable; they are loaded with `MAP_PRIVATE` semantics; they can be modified with arithmetic operations expressed as JAX functions.

The analogy to conventional compiled programs is therefore:

| Conventional Program | Neural Network |
|---|---|
| C/C++ source | Python/Flax source |
| Intermediate representation (IR) | `jaxpr` / StableHLO |
| Object file | Exported StableHLO module |
| Executable binary | Compiled XLA computation (cached) |
| Data segment | Weight arrays (SafeTensors / Orbax checkpoint) |
| Dynamic linker | `jax.jit` cache + XLA persistent cache |
| JIT recompiler | `jax.jit` with `donate_argnames` on modified weights |
| Runtime code cache | XLA compilation cache |

The key difference from conventional programs is not that neural networks are compiled differently, but that **the compiled artifact (the weights) changes during the system's operational lifetime**, and the compiler is present and reachable to produce a new compiled form whenever the computation graph changes. Chapter 207 identifies this homoiconic property as central to the AI-first programming language vision. JAX realises it not through a reflective type theory but through the concrete mechanics of functional IR tracing and the Python runtime.

Three implications of this structure for self-modifying systems deserve emphasis.

First, **the address space is unified**. The JAX process that runs inference also imports `mlir`, `jax.export`, and Flax. The compiler toolchain is not a separate process, not a separate machine, not a privilege boundary away. Modifying the computation means calling Python functions in the same address space. The OS-level protection boundary between "user code" and "compiler" does not exist in this architecture.

Second, **the compilation cache is a first-class object**. XLA's compilation cache maps (computation_graph_hash, hardware_signature) to compiled `Executable` objects. When a self-modifying system changes weights, the computation graph hash does not change — the cache hit is immediate. When the system changes architecture, the hash changes — recompilation begins automatically. The system does not need explicit logic to decide when to recompile; the JAX/XLA cache implements that policy.

Third, **the modification and the execution substrate are the same language**. Weight updates are expressed as JAX functions (pure, differentiable, vmappable). The inference computation is a JAX function. The gradient computation is a JAX function. The MLIR analysis pass is invoked from Python via JAX's Python bindings. Everything composes through the same functional interface — there is no impedance mismatch between "the part that runs the model" and "the part that modifies the model".

### The Homoiconic jaxpr

The deepest expression of this property is `jax.make_jaxpr`:

```python
import jax
import jax.numpy as jnp
import flax.linen as nn

class SingleHead(nn.Module):
    d_model: int
    d_head: int

    @nn.compact
    def __call__(self, x):
        q = nn.Dense(self.d_head)(x)
        k = nn.Dense(self.d_head)(x)
        v = nn.Dense(self.d_head)(x)
        scale = self.d_head ** -0.5
        attn = jnp.einsum('bsd,btd->bst', q, k) * scale
        attn = jax.nn.softmax(attn, axis=-1)
        return jnp.einsum('bst,btd->bsd', attn, v)

model = SingleHead(d_model=64, d_head=32)
x = jnp.zeros((2, 8, 64))           # batch=2, seq=8, d_model=64
params = model.init(jax.random.PRNGKey(0), x)

# The forward pass returned as a first-class Python object
jpr = jax.make_jaxpr(model.apply)(params, x)

# jpr.jaxpr.eqns is a list of JaxprEqn — every primitive in the forward pass
for eq in jpr.jaxpr.eqns[:6]:
    print(f"{eq.primitive.name:20s}  in={[v.aval for v in eq.invars[:2]]}")
```

The returned `ClosedJaxpr` is not a string or a blob — it is a structured Python object. `jpr.jaxpr.eqns` is a list of `JaxprEqn` records, each with `.primitive`, `.invars`, `.outvars`, and `.params`. The computation graph of the model's forward pass is a Python data structure that can be traversed, pattern-matched, and rewritten by ordinary Python code. This is compiler-level homoiconicity: **the code is data**.

---

## 211.2 The IR Chain: Seven Layers of Introspection

The path from Python source to GPU binary passes through seven distinct representation layers. Each is a full introspection point: it can be read, analysed, and in most cases transformed or substituted. The table below summarises all seven layers before the subsections treat each one.

| Layer | Representation | Inspection tool | Modification cost |
|---|---|---|---|
| 0 | Python/Flax source | Python AST (`ast` module) | Re-trace (ms) |
| 1 | `jaxpr` | `jax.make_jaxpr(f)(x)` | Re-lower (ms–s) |
| 2 | StableHLO (MLIR) | `jax.export.export(jax.jit(f))(x).mlir_module()` | Re-XLA-compile (s) |
| 3 | MLIR Linalg/Tensor | `mlir-opt` + Python `mlir` bindings | Re-LLVM-lower (s) |
| 4 | LLVM IR | `opt`, `llc`, `llvm-dis` | Re-PTX-assemble (s) |
| 5 | PTX | `cuobjdump --dump-ptx`, `ptxas` | Re-SASS-assemble (ms) |
| 6 | SASS | `nvdisasm`, `cuobjdump --dump-sass` | Not typically modified programmatically |

### Layer 0 — Python/Flax Source

The highest layer and the most readable. Python's `ast` module gives full access to the source AST; `inspect.getsource(module)` retrieves the source text. Programmatic modification at this layer means generating or rewriting Python source, then calling `exec` or importing the rewritten module. The cost is a new JAX trace — typically milliseconds for a single forward function, seconds if the model architecture changes significantly.

For architecture search (Chapter 215), this is the natural layer: a search procedure generates Python source defining a new `nn.Module` subclass, `exec`s it into the current namespace, and hands it to JAX for tracing. The Python runtime acts as the macro expander of an AI-first programming language.

### Layer 1 — jaxpr

The jaxpr is a typed, functional, single-assignment intermediate representation. It is produced by JAX's tracing pass and is the first representation with a precise operational semantics independent of Python. Each equation (`JaxprEqn`) names an output variable, specifies a primitive, and lists input variables.

```python
# Inspect jaxpr equations for dot_general (matmul) operations
def find_matmuls(jpr):
    return [eq for eq in jpr.jaxpr.eqns
            if eq.primitive.name == 'dot_general']

matmuls = find_matmuls(jpr)
for m in matmuls:
    dims = m.params['dimension_numbers']
    print(f"contracting: {dims.lhs_contracting} × {dims.rhs_contracting}  "
          f"batch: {dims.lhs_batch}")
```

Modification at the jaxpr layer means constructing new `JaxprEqn` objects and calling `jax.core.eval_jaxpr` to interpret the modified graph. This is low-level: jaxpr rewrites require understanding JAX's primitive system. The practical path for most modifications is to work at Layer 0 or Layer 2.

### Layer 2 — StableHLO

StableHLO is the serialisable, versioned, portable IR that JAX produces before handing off to XLA. It is an MLIR dialect. Obtaining the StableHLO text for a JAX computation:

```python
# Obtain the StableHLO MLIR text for a JAX function
lowered = jax.jit(model.apply).lower(params, x)
stablehlo_text = lowered.as_text()          # StableHLO MLIR text
stablehlo_module = lowered.compiler_ir()    # mlir.ir.Module object

# Apply MLIR passes via Python bindings
from mlir.passmanager import PassManager
with stablehlo_module.context:
    pm = PassManager.parse("builtin.module(stablehlo-legalize-to-linalg)")
    pm.run(stablehlo_module.operation)
```

StableHLO is the format used by `jax.export` for cross-runtime portability. An exported module can be loaded by a different process, a different accelerator runtime (IREE, TensorRT), or a different language via the C API. For programmatic analysis (§211.4), StableHLO is the most useful layer: it is simultaneously high-level enough to identify attention heads and FFN layers, and formal enough to support MLIR passes.

### Layer 3 — MLIR Linalg/Tensor

After StableHLO, XLA's MLIR pipeline lowers to the `linalg` and `tensor` dialects for hardware-agnostic loop-level optimisation. At this layer, `linalg.generic` and `linalg.matmul` operations are visible along with tiling annotations. Running this lowering from the command line:

```bash
# Lower StableHLO through Linalg
mlir-opt --stablehlo-legalize-to-linalg \
         --linalg-fuse-elementwise-ops \
         --convert-linalg-to-loops \
         attention.mlir -o attention_loops.mlir
```

The tiling, fusion, and bufferisation passes available at this layer are described in detail in Chapter 209. For self-modifying systems, this layer is primarily an analysis target: understanding which `linalg.matmul` operations were fused, and at what tile sizes, informs kernel-level optimisation decisions (Chapter 208).

### Layer 4 — LLVM IR

The LLVM IR level exposes the CPU/GPU computation after all MLIR lowering has completed. For GPU (PTX), this is the NVPTX backend IR. For CPU workloads running through XLA:CPU, this is standard LLVM IR amenable to the full `opt` pass pipeline.

```bash
# Inspect LLVM IR produced by XLA:CPU for a JAX function
# (GPU path goes through PTX; see Layer 5)
python3 -c "
import jax, jax.numpy as jnp
f = lambda x: jnp.tanh(x @ x.T)
x = jnp.ones((4, 4))
lowered = jax.jit(f).lower(x)
print(lowered.as_text(dialect='hlo'))
"
```

For GPU paths, the LLVM NVPTX IR is an intermediate within the XLA:GPU compilation pipeline; it is not easily extracted without patching XLA's compilation internals, but `CUDA_LAUNCH_BLOCKING=1` with nvprof gives the PTX. Profiling-guided optimisation (PGO) at this level is available through LLVM's standard `llvm-profdata`/`clang -fprofile-use` infrastructure for CPU paths.

### Layer 5 — PTX

PTX (Parallel Thread eXecution) is NVIDIA's virtual ISA — human-readable GPU assembly that NVIDIA's driver JITs to SASS for the specific GPU microarchitecture. It is the lowest layer routinely inspected by ML performance engineers.

```bash
# Dump PTX from a compiled CUDA module
cuobjdump --dump-ptx compiled_module.cubin | head -60

# Inspect register usage and occupancy from PTX
ptxas --gpu-name sm_90 --verbose kernel.ptx 2>&1 | grep "Used.*registers"
```

PTX is human-writable for targeted micro-optimisations (register spill elimination, barrier placement), but programmatic generation at this layer is rarely worthwhile — Triton and CUDA C are more productive at the same abstraction level. Its value is diagnostic: PTX reveals register pressure, predication patterns, and memory access patterns that are invisible at the MLIR level.

### Layer 6 — SASS

SASS (Shader ASSembly) is the final binary encoding for a specific NVIDIA GPU architecture (Ampere, Hopper, Blackwell). It is architecture-specific and not forwards-compatible. `nvdisasm` disassembles it for diagnostic purposes.

```bash
cuobjdump --dump-sass compiled_module.cubin | grep -A 4 "HMMA\|GMMA"
```

SASS disassembly reveals actual instruction scheduling decisions — whether HMMA (tensor core) or FMUL instructions are used, warp divergence in conditionals, and load/store grouping. For a self-aware system, this is telemetry only: no programmatic modification at this layer is practical. The correct intervention when SASS analysis reveals a problem is to modify the kernel at Layer 2 (StableHLO) or Layer 5 (PTX), and let the compiler regenerate SASS.

---

## 211.3 Weight Checkpoints as Compiled Binaries

The analogy between neural network weight files and binary object files is structural, not merely metaphorical. Both are self-describing binary formats with a header section encoding layout metadata and an aligned data section containing the payload. The key difference is semantic: an ELF binary's text section contains executable instructions; a weight file's data section contains floating-point coefficients. The loading mechanics are identical.

### ELF, SafeTensors, and GGUF Side by Side

**ELF (Executable and Linkable Format)** — the standard object file format on Linux — has the structure:

```
Offset  Field
0x00    Magic: 0x7f 'E' 'L' 'F'  (4 bytes)
0x04    Class, Data, Version, OS/ABI, ...
0x10    e_type, e_machine, e_version
0x18    e_entry (program entry point)
0x20    e_phoff (program header table offset)
0x28    e_shoff (section header table offset)
...     section headers: name, type, flags, addr, offset, size
...     data: .text, .data, .rodata, .bss, ...
```

**SafeTensors** (Hugging Face, 2022 — [github.com/huggingface/safetensors](https://github.com/huggingface/safetensors)) uses a simpler two-part structure. There are no magic bytes; the file begins immediately with an 8-byte little-endian unsigned integer giving the JSON header length:

```
Offset  Width  Field
0x00    8      header_length: u64 LE  — byte count of the JSON that follows
0x08    N      JSON header: UTF-8, describes all tensors
0x08+N  ...    tensor data buffer (raw bytes, dtype and shape from header)
```

The JSON header is a dict mapping tensor names to metadata objects:

```json
{
  "model.embed.weight":   {"dtype": "BF16", "shape": [32000, 4096],
                           "data_offsets": [0, 262144000]},
  "layers.0.attn.wq":    {"dtype": "BF16", "shape": [4096, 4096],
                           "data_offsets": [262144000, 296408064]},
  "__metadata__": {"format": "pt", "version": "0.4.0"}
}
```

`data_offsets` are byte offsets **within the data buffer** (i.e., relative to `0x08 + header_length`). This layout is directly analogous to ELF section headers: a compact descriptor table followed by the raw data at the described offsets.

**GGUF** (llama.cpp, 2023 — [github.com/ggml-org/llama.cpp/blob/master/docs/gguf.md](https://github.com/ggml-org/llama.cpp/blob/master/docs/gguf.md)) is more ambitious: it is a self-describing format that carries not just tensor data but model architecture hyperparameters (layer count, head count, rope theta) in a structured key-value metadata store:

```
Offset  Field
0x00    Magic: 'G' 'G' 'U' 'F'  (4 bytes, ASCII)
0x04    Version: u32 LE (current: 3)
0x08    tensor_count: u64 LE
0x10    metadata_kv_count: u64 LE
...     metadata KV pairs: string key, typed value
...     tensor info array: name, n_dims, shape[], ggml_type, data_offset
...     tensor data (aligned to 32 bytes per tensor)
```

GGUF's `ggml_type` field encodes quantisation format (F32, F16, Q8_0, Q4_K_M, Q2_K) directly in the tensor descriptor — the metadata encodes not just layout but the dequantisation algorithm required to recover float values. This is analogous to ELF dynamic relocations: metadata describing a transformation that must be applied to the raw data before it can be used.

### Zero-Copy Weight Loading via mmap

The performance-critical property shared by both SafeTensors and ELF shared objects is **zero-copy loading via `mmap`**. The OS virtual memory system maps the file into the process address space without copying bytes. Weights are accessed on demand through page faults, and the OS's LRU page eviction policy acts as an automatic weight cache. For a 70B-parameter model at BF16, this is 140 GB of data that need not be copied into a separate buffer.

```python
import struct
import json
import mmap
import numpy as np

def load_safetensors_header(path: str) -> dict:
    """Read the JSON header from a SafeTensors file without loading weights."""
    with open(path, 'rb') as f:
        # First 8 bytes: header length as little-endian u64
        header_len = struct.unpack('<Q', f.read(8))[0]
        header_bytes = f.read(header_len)
    return json.loads(header_bytes)

def mmap_tensor(path: str, name: str) -> np.ndarray:
    """Memory-map a single tensor by name from a SafeTensors file."""
    import ml_dtypes                        # pip install ml-dtypes
    with open(path, 'rb') as f:
        header_len = struct.unpack('<Q', f.read(8))[0]
        header = json.loads(f.read(header_len))
        meta = header[name]
        dtype_map = {
            'BF16': ml_dtypes.bfloat16,    # native BF16 via ml_dtypes (LLVM 22 era)
            'F16':  np.float16,
            'F32':  np.float32,
            'F64':  np.float64,
            'I32':  np.int32,
            'I64':  np.int64,
        }
        dtype = dtype_map[meta['dtype']]
        start, end = meta['data_offsets']
        data_base = 8 + header_len          # offset of data buffer in file
        mm = mmap.mmap(f.fileno(), length=0, access=mmap.ACCESS_READ)
        arr = np.frombuffer(mm, dtype=dtype,
                            offset=data_base + start,
                            count=(end - start) // np.dtype(dtype).itemsize)
        return arr.reshape(meta['shape'])

# Example: inspect the embedding weight of a Llama checkpoint
header = load_safetensors_header('llama-3-8b/model.safetensors')
tensor_names = [k for k in header if k != '__metadata__']
print(f"Total tensors: {len(tensor_names)}")
# → Total tensors: 291

embed = mmap_tensor('llama-3-8b/model.safetensors', 'model.embed_tokens.weight')
print(f"Embedding: {embed.shape} {embed.dtype}")
# → Embedding: (32000, 4096) bfloat16   (ml_dtypes.bfloat16, arithmetically usable)
```

The `mmap.ACCESS_READ` call issues `mmap(fd, MAP_PRIVATE | MAP_FILE, ...)` — precisely the same syscall the dynamic linker uses to map shared library text sections. From the operating system's perspective, loading a language model's weights and loading a shared library are the same operation.

The implication for self-modifying systems: weight surgery at the checkpoint level (Chapter 212's `ROME`/`MEMIT` edits) does not require loading the full model into GPU memory first. The system can inspect the header, identify target tensors by name, load only those tensors via `mmap`, compute the edited values, and write a new checkpoint — without instantiating the full model. This is the compiled-binary analogue of a binary patch: targeted modification of specific sections without touching the rest.

---

## 211.4 XLA HLO as a Homoiconic Computation Graph

StableHLO (the stable, versioned subset of HLO) is the representation at which JAX hands off to XLA. It is simultaneously JAX's output format and XLA's input format — a shared language across the framework boundary. Its homoiconic property is fundamental: the forward pass of a model IS an HLO module, and the backward pass is derived from that same module by HLO-level automatic differentiation.

### Core HLO Operations in a Transformer

A transformer's computational primitives map cleanly onto a small set of HLO operations:

| Neural operation | HLO primitive | Key attributes |
|---|---|---|
| General matrix multiply | `dot_general` | `lhs_contracting_dims`, `rhs_contracting_dims`, `lhs_batch_dims` |
| Attention softmax numerator | `exp` + `reduce` | `dimensions=[axis]` |
| Multi-head reshape | `reshape` | shape |
| KV cache scatter | `scatter` | `scatter_dims_to_operand_dims` |
| Positional encoding rotation | `gather` | `offset_dims`, `start_index_map` |
| Layer normalisation | `reduce` + arithmetic | `dimensions=[hidden_dim]` |
| Autoregressive loop | `while` | body computation, condition computation |
| Mixture-of-experts routing | `conditional` | N branch computations |

The `dot_general` operation encodes the full generality of batched matrix multiplication in its `DotDimensionNumbers` attribute. Understanding which dimensions are contracting (multiplied and summed) vs. batch (parallelised over) vs. free (kept in output) is the key to understanding a model's computational structure.

### Extracting the StableHLO Module

```python
import jax
import jax.numpy as jnp
from flax import linen as nn

class AttentionLayer(nn.Module):
    num_heads: int = 8
    d_model: int = 512

    @nn.compact
    def __call__(self, x):
        d_head = self.d_model // self.num_heads
        B, S, D = x.shape
        qkv = nn.Dense(3 * self.d_model)(x)                # [B,S,3D]
        q, k, v = jnp.split(qkv, 3, axis=-1)
        # Reshape to [B, num_heads, S, d_head]
        q = q.reshape(B, S, self.num_heads, d_head).transpose(0, 2, 1, 3)
        k = k.reshape(B, S, self.num_heads, d_head).transpose(0, 2, 1, 3)
        v = v.reshape(B, S, self.num_heads, d_head).transpose(0, 2, 1, 3)
        scale = d_head ** -0.5
        attn = jnp.einsum('bhsd,bhtd->bhst', q, k) * scale
        attn = jax.nn.softmax(attn, axis=-1)
        out = jnp.einsum('bhst,bhtd->bhsd', attn, v)
        out = out.transpose(0, 2, 1, 3).reshape(B, S, D)
        return nn.Dense(self.d_model)(out)

model = AttentionLayer()
x_shape = jnp.zeros((2, 16, 512))
params = model.init(jax.random.PRNGKey(0), x_shape)

# Lower to StableHLO and extract the MLIR module text
lowered = jax.jit(model.apply).lower(params, x_shape)
hlo_text = lowered.as_text()           # StableHLO MLIR text
# hlo_text contains func @main with stablehlo.dot_general, stablehlo.reduce, etc.

# Inspect: count dot_general operations (= attention Q/K/V projections + output proj)
dot_count = hlo_text.count('stablehlo.dot_general')
print(f"dot_general operations: {dot_count}")
# → dot_general operations: 4  (Wq, Wk, Wv, Wo)
```

The `as_text()` method returns valid MLIR text that can be piped to `mlir-opt` for further lowering, saved to disk, shipped to a different runtime (IREE, TensorRT via torch-mlir), or parsed back into an `mlir.ir.Module` via Python bindings for programmatic analysis.

### Reading StableHLO Text

A concrete fragment of StableHLO MLIR text for the Q projection in the attention layer above shows the actual IR syntax:

```mlir
func.func @main(%arg0: tensor<2x16x512xf32>, %arg1: tensor<512x512xf32>,
                %arg2: tensor<512xf32>) -> tensor<2x16x512xf32> {
  // Q projection: [B,S,D] × [D,D] → [B,S,D]
  %0 = stablehlo.dot_general %arg0, %arg1,
         contracting_dims = [2] x [0]
         : (tensor<2x16x512xf32>, tensor<512x512xf32>) -> tensor<2x16x512xf32>
  %1 = stablehlo.broadcast_in_dim %arg2, dims = [2]
         : (tensor<512xf32>) -> tensor<2x16x512xf32>
  %2 = stablehlo.add %0, %1
         : (tensor<2x16x512xf32>, tensor<2x16x512xf32>) -> tensor<2x16x512xf32>
  return %2 : tensor<2x16x512xf32>
}
```

The `contracting_dims = [2] x [0]` annotation specifies that dimension 2 of the left operand (the `D` axis of the `[B,S,D]` activation tensor) contracts with dimension 0 of the right operand (the input dimension of the weight matrix `[D_in, D_out]`). The output has shape `[B,S,D_out]`. For a self-analysing system, this encoding is machine-readable: parsing `contracting_dims` from the StableHLO text directly yields the architectural relationship between the activation dimension and the weight dimension, without any model-specific configuration file.

The `stablehlo.reduce` operation that implements softmax reveals the attention pattern:

```mlir
// Numerically-stable softmax denominator: reduce over S dimension (key sequence)
%max = stablehlo.reduce(%attn_scores init: %neg_inf)
         applies stablehlo.maximum across dimensions = [3]
         : (tensor<2x8x16x16xf32>, tensor<f32>) -> tensor<2x8x16xf32>
%exp_shifted = stablehlo.exponential
         stablehlo.subtract(%attn_scores, %max_broadcast)
         : tensor<2x8x16x16xf32>
%sum = stablehlo.reduce(%exp_shifted init: %zero)
         applies stablehlo.add across dimensions = [3]
         : (tensor<2x8x16x16xf32>, tensor<f32>) -> tensor<2x8x16xf32>
```

The `dimensions = [3]` attribute identifies axis 3 as the key-sequence dimension — the softmax normalises over the sequence of keys. A self-aware system analysing this text learns that the model implements standard causal or non-causal attention (determined by the mask applied before the reduction) with 8 heads operating on 16-token sequences.

### The Homoiconic Backward Pass

The deepest HLO homoiconicity property is automatic differentiation. `jax.grad` does not operate on Python code — it operates on the jaxpr derived from tracing the forward pass, and produces a new jaxpr encoding the backward pass. Both jaxprs are lowered to StableHLO, and both appear in the compiled XLA computation:

```python
# The backward pass is derived at the IR level, not the Python level
grad_fn = jax.grad(lambda p, x: model.apply(p, x).sum())

# The jaxpr of grad_fn includes both forward and backward computation
grad_jpr = jax.make_jaxpr(grad_fn)(params, x_shape)
forward_eqns = sum(1 for eq in grad_jpr.jaxpr.eqns
                   if 'dot_general' in eq.primitive.name)
# Both forward dot_generals and their VJP (transpose) dot_generals are present
```

This is precisely the "code is data" property that Chapter 207 identifies as the goal of AI-first programming languages. The backward pass is not hand-written — it is an HLO-to-HLO transformation applied to the forward pass HLO. The forward and backward computations are data structures in the same IR format, inspectable and transformable by the same tools. The gradient flow through a specific layer is not a mathematical abstraction but a set of named HLO operations in a Python-accessible data structure.

### Structural Analysis via HLO Inspection

For self-improving systems, HLO inspection enables architectural self-awareness. A system that can enumerate its own `dot_general` operations knows how many attention heads it has, what their contracting dimensions are, and therefore what computational patterns they implement:

```python
def analyse_model_structure(params, model, x):
    """Extract architectural facts from HLO without running inference."""
    jpr = jax.make_jaxpr(model.apply)(params, x)
    dot_ops = [eq for eq in jpr.jaxpr.eqns
               if eq.primitive.name == 'dot_general']
    structure = []
    for op in dot_ops:
        dims = op.params['dimension_numbers']
        lhs_shape = op.invars[0].aval.shape
        rhs_shape = op.invars[1].aval.shape
        out_shape  = op.outvars[0].aval.shape
        structure.append({
            'lhs': lhs_shape, 'rhs': rhs_shape, 'out': out_shape,
            'lhs_contracting': dims.lhs_contracting,
            'rhs_contracting': dims.rhs_contracting,
            'lhs_batch': dims.lhs_batch,
        })
    return structure

structure = analyse_model_structure(params, model, x_shape)
for i, s in enumerate(structure):
    print(f"dot_general[{i}]: {s['lhs']} @ {s['rhs']} → {s['out']}")
```

This analysis requires no model documentation, no configuration file, no architecture-specific parsing. The model tells you its own structure through its HLO — a form of compiler-mediated self-knowledge.

---

## 211.5 The Self-Aware Feedback Loop

The preceding sections established the individual components: the seven IR layers as introspection points, weight checkpoints as mutable structured binaries, and HLO as a homoiconic computation graph. This section assembles them into the complete operational feedback loop.

```
┌─────────────────────────────────────────────────────────────────┐
│                  The Self-Aware Execution Loop                   │
│                                                                   │
│  Python/Flax ──jit──► jaxpr ──lower──► StableHLO               │
│       │                                     │                    │
│       │            ◄──recompile──          XLA                   │
│       ▼                                     │                    │
│    modify                                 LLVM IR                │
│   (weights /                               │                    │
│  architecture)                           PTX / SASS             │
│       ▲                                     │                    │
│       │                               GPU execution             │
│       │                                     │                    │
│   introspect ◄───────────────────── profile │                    │
│  (SAE, causal                      (Triton perf,                 │
│   tracing)                          XLA/Perfetto timing)        │
└─────────────────────────────────────────────────────────────────┘
```

### Step 1 — Infer

```python
import jax
import jax.numpy as jnp
from flax import nnx

# Compiled inference — jax.jit compiles on first call, caches thereafter
@nnx.jit
def inference_step(model, x):
    return model(x)

# With explicit weight donation: tell JAX the weight buffer can be reused
@functools.partial(jax.jit, donate_argnames=['params'])
def inference_step_donating(params, x):
    return model.apply(params, x)
```

The compiled computation is cached by XLA using the computation graph's shape signature as a key. Subsequent calls with the same input shapes skip recompilation and hit the cache, executing in microseconds (GPU kernel launch overhead only).

### Step 2 — Profile

Profiling operates at two levels: kernel-level (GPU) and graph-level (HLO).

**Kernel-level profiling** uses JAX's Perfetto-based tracer:

```python
import jax.profiler

# Capture a Perfetto trace
with jax.profiler.trace('/tmp/jax-trace', create_perfetto_link=True):
    for _ in range(10):
        y = inference_step(model, x)
    y.block_until_ready()
# → Opens Perfetto UI showing per-kernel timing on the GPU timeline
```

For Triton kernels (Chapter 208), NVIDIA Nsight Compute provides per-kernel SM occupancy, memory bandwidth utilisation, and arithmetic intensity. A self-aware system running its own Triton kernels can automate this via the `ncu` command-line tool and parse its structured CSV output.

**Graph-level profiling** extracts HLO-level timing. XLA annotates HLO operations with per-operation execution times inside the same Perfetto trace — the JSON trace emitted by `jax.profiler.trace` contains XLA-level spans with names derived from HLO operation types. For offline inspection, XLA can dump the raw HLO text and dot-graph representations:

```bash
# Dump XLA HLO artifacts for offline analysis
XLA_FLAGS="--xla_dump_to=/tmp/xla-dump --xla_dump_hlo_as_text" \
python3 run_model.py

# The dump directory contains:
#   module_xxxx.before_optimizations.txt  — pre-optimisation HLO
#   module_xxxx.after_optimizations.txt   — post-optimisation HLO (what ran)
ls /tmp/xla-dump/
```

Comparing pre- and post-optimisation HLO reveals which fusions XLA performed, which operations were eliminated, and which `dot_general` invocations were merged into a single kernel. This is HLO-level structural introspection without per-operation timing; the Perfetto trace provides the timing, and the dump provides the structural context that makes the timing meaningful.

### Step 3 — Introspect

Introspection operates at the activation level, above the IR level. TransformerLens hooks and Sparse Autoencoder (SAE) features (Chapter 213) identify which model components are responsible for specific capabilities:

```python
# TransformerLens-style activation patching (simplified)
import transformer_lens as tl

hooked_model = tl.HookedTransformer.from_pretrained('gpt2')

# Cache activations at every layer's residual stream
_, cache = hooked_model.run_with_cache("The Eiffel Tower is in")

# Locate the components that encode location knowledge
for layer in range(hooked_model.cfg.n_layers):
    resid = cache[f'blocks.{layer}.hook_resid_post']     # [B, S, D]
    print(f"Layer {layer}: resid norm = {resid.norm(dim=-1).mean():.3f}")
```

Causal tracing (Chapter 214) patches specific activations to determine which components are causally necessary for correct factual recall. The combination of HLO structural analysis (§211.4) and activation-level causal analysis gives the system both the compiler-level map ("these three `dot_general` operations implement multi-head attention in layer 7") and the semantic map ("layer 7's attention is causally necessary for the Eiffel Tower → Paris association").

### Step 4 — Modify

Weight modification uses `jax.tree_util.tree_map` for arithmetic operations over parameter PyTrees:

```python
import jax.tree_util as jtu

# Scale down a specific attention head's value projection
def patch_head_value(params, layer: int, head: int, scale: float):
    def _patch(path, leaf):
        key = '/'.join(str(k) for k in path)
        if f'layers_{layer}' in key and 'attn' in key and 'v_proj' in key:
            # Zero out rows corresponding to specific head
            d_head = leaf.shape[-1] // 8   # assume 8 heads
            mask = jnp.ones_like(leaf)
            mask = mask.at[head * d_head:(head + 1) * d_head].set(scale)
            return leaf * mask
        return leaf
    return jtu.tree_map_with_path(_patch, params)

# Apply and immediately donate buffer for next compiled call
patched_params = patch_head_value(params, layer=7, head=3, scale=0.0)
```

For architecture modification (Chapter 215), the modification is at Layer 0: a new Python class definition is generated, `exec`d, and passed to JAX for tracing. The compilation cache handles the rest.

### Step 5 — Recompile

JAX's compilation cache uses the computation graph's structure and input shapes as the cache key. If weights change but the computation graph does not (standard weight editing), no recompilation occurs — the cached compiled computation runs with the new weight values via the device buffer reference. If the architecture changes (new layer count, different head configuration), `jax.jit` detects a new computation graph and recompiles.

```python
# donate_argnames tells JAX the weight buffer will be replaced
# Useful when editing produces a new param dict of the same structure
@functools.partial(jax.jit, donate_argnames=['params'])
def compiled_apply(params, x):
    return model.apply(params, x)

# After editing: new params, same compiled computation graph
result = compiled_apply(patched_params, x)
# No recompilation — same graph, new data

# Persistent compilation cache: survives process restarts
import os
os.environ['JAX_COMPILATION_CACHE_DIR'] = '/tmp/jax-cache'
# Subsequent jax.jit calls look up the cache before recompiling
```

The persistent cache (`JAX_COMPILATION_CACHE_DIR`) serialises XLA compiled computations to disk. For large models, the first compilation takes tens of seconds; cache hits take milliseconds. This is the neural-network analogue of the object file cache in a build system: expensive compilation is amortised across multiple runs.

---

## 211.6 MLIR as a Universal Analysis Layer

MLIR's Python bindings make the entire pass infrastructure available from the same Python process that runs inference. This means compiler passes — originally designed to transform code — can be applied to model computations for analysis and optimisation discovery, not just code generation.

### Applying Passes via Python Bindings

```python
from mlir import ir
from mlir.passmanager import PassManager

# Obtain the MLIR module for the JAX computation
lowered = jax.jit(model.apply).lower(params, x_shape)
module = lowered.compiler_ir(dialect='stablehlo')   # mlir.ir.Module

# Run MLIR analysis pass in-process
with module.context:
    pm = PassManager.parse(
        "builtin.module("
        "  stablehlo-legalize-to-linalg,"
        "  linalg-fuse-elementwise-ops"
        ")"
    )
    pm.run(module.operation)
    fused_text = str(module)
```

This pattern enables architecture-aware fusion analysis: by running `linalg-fuse-elementwise-ops` on the lowered computation, the system discovers which elementwise operations XLA will fuse with adjacent matrix multiplications. If a candidate activation function (e.g., replacing ReLU with GELU) would break a fusion, the MLIR pass reveals this before any GPU kernel is launched.

### DOT Graph Visualisation

The MLIR `print-op-graph` pass emits a DOT-format computation graph. Rendered with Graphviz, this provides a visual map of the model's computational structure at the MLIR level:

```bash
# Dump the computation graph as DOT for visualisation
mlir-opt --print-op-graph attention.mlir 2>attention.dot
dot -Tsvg attention.dot -o attention.svg
```

For automated structural analysis, the DOT output is parseable — counting nodes by operation type, identifying paths through the graph, detecting subgraph patterns that correspond to known architectural motifs (MLP blocks, attention sub-graphs). A self-aware system that identifies "the computation graph for this model has 96 `stablehlo.dot_general` nodes arranged in 32 groups of 3 (Q, K, V) followed by a 4th output projection" has derived its own architectural specification from its compiled form.

### MLIR Dialects as a Semantic Stratification

The layered structure of MLIR dialects — StableHLO → Linalg → Affine → SCF → LLVM — is itself an information-stratified view of the same computation. Each lowering step discards some semantic information (the `dot_general` identity disappears when it becomes `linalg.matmul`, which disappears when it becomes loop nests) and reveals new information (tile structure, loop bounds, memory access patterns). A self-modifying system must choose the dialect level appropriate to its analysis goal:

| Analysis goal | Best dialect level | Why |
|---|---|---|
| Count attention heads, detect architecture | StableHLO | `dot_general` directly maps to projection matrices |
| Identify fusion opportunities | Linalg | `linalg-fuse-elementwise-ops` is the relevant pass |
| Estimate memory traffic | Affine loops | Dependence analysis and access function analysis |
| Predict register pressure | LLVM IR / PTX | Phi nodes, register allocation visible |
| Measure actual kernel latency | SASS / Nsight | Real hardware timing |

A system operating at StableHLO for architecture analysis and at Linalg for fusion analysis, without descending further, exploits the dialect stratification efficiently. It asks each dialect layer the questions that dialect can answer, and no more.

### Standalone Pass Experimentation with mlir-opt

For offline pass pipeline development, `mlir-opt` allows running arbitrary pass sequences on saved MLIR files:

```bash
# Save the StableHLO module to disk
python3 -c "
import jax, jax.numpy as jnp, flax.linen as nn

class FFN(nn.Module):
    @nn.compact
    def __call__(self, x):
        return nn.Dense(4096)(jax.nn.gelu(nn.Dense(16384)(x)))

m = FFN(); x = jnp.zeros((2,16,4096))
p = m.init(jax.random.PRNGKey(0), x)
print(jax.jit(m.apply).lower(p, x).as_text())
" > ffn.mlir

# Experiment with tiling strategies
mlir-opt \
  --stablehlo-legalize-to-linalg \
  --linalg-tile-loops='linalg-tile-sizes=16,16' \
  --convert-linalg-to-affine-loops \
  ffn.mlir -o ffn_tiled.mlir

# Count tiled loop nests
grep -c 'affine.for' ffn_tiled.mlir
```

This workflow — JAX lowers, MLIR explores — allows a system to evaluate tiling strategies for candidate architectures without running GPU kernels. The analysis is cheap (seconds on CPU) and produces concrete predictions about memory access patterns that Triton kernel generation (Chapter 208) can act on.

---

## 211.7 Tiered Recompilation: Neural Networks and JIT

Chapter 108 describes ORC JIT's tiered compilation strategy: a function is first compiled at a low optimisation tier (fast compilation, lower runtime performance) and executed immediately. A profiling shim counts calls. When a threshold is reached, the `ReOptimizeLayer` recompiles at a higher tier (slower compilation, higher runtime performance) and installs the new version. The caller transparently begins executing the optimised version.

### ORC JIT ReOptimizeLayer

The ORC implementation in `llvm/lib/ExecutionEngine/Orc/ReOptimizeLayer.cpp` ([source](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/ReOptimizeLayer.cpp)) maintains a per-function `ReOptState` tracking invocation count and the current materialisation unit. When `CallThroughManager::callThrough` detects the threshold, it schedules recompilation via `IRTransformLayer` at the new optimisation level and atomically swaps the function pointer in the stub.

The neural network analogue uses JAX's JIT caching as the "stub" mechanism:

| ORC JIT Concept | JAX Neural Analogue |
|---|---|
| Tier-0 compilation (fast, unoptimised) | `jax.jit` with default settings (first call) |
| Call count profiling | JAX profiler trace + Perfetto analysis |
| Tier-1 recompilation trigger | Weight editing or architecture change detected |
| `ReOptimizeLayer.reoptimize()` | New `jax.jit` call with modified params/architecture |
| Atomic stub swap | JAX cache replacement (new cache key for new graph) |
| Profile-guided optimisation | Triton tile size search (Chapter 208) |

The key architectural parallel: both systems separate **the decision to recompile** (a policy decision) from **the mechanism of recompilation** (the compiler). In ORC JIT, the policy is invocation count; in a self-improving neural system, the policy is capability measurement (Chapter 212) or interpretability analysis (Chapter 213).

### The Neural Tier-0 / Tier-1 Cycle

A practical neural tiered recompilation cycle looks like:

**Tier 0 — Frozen inference.** The model runs with its checkpoint weights, compiled once to an XLA/PTX computation. This is the deployment-ready baseline. The system records performance metrics (task accuracy, latency, capability probe scores).

**Profiling gate.** After N inference steps, the system evaluates whether a capability metric falls below a threshold, or whether interpretability analysis (Chapter 213) identifies a specific capability deficit that maps to identifiable weight components.

**Tier 1 — Weight editing.** If the gate triggers, the system applies a targeted weight edit (Chapter 212: ROME, MEMIT, or gradient-based fine-tuning) to the identified components. Because the computation graph structure does not change, no XLA recompilation is needed — the new weights execute through the cached compiled computation immediately.

**Tier 2 — Architecture change.** If weight editing does not resolve the capability deficit (e.g., the model lacks a head with the right attention pattern), the system modifies the architecture (Chapter 215: add a head, modify the FFN ratio, insert an adapter layer). This triggers full recompilation — a new `jax.jit` trace, new StableHLO lowering, new XLA compilation. The cost is seconds; the benefit is a structurally different model.

```python
def tiered_reoptimise(model, params, eval_fn, x_eval,
                      edit_fn, arch_fn,
                      capability_threshold=0.85):
    """
    ORC-JIT-style tiered reoptimisation for a neural model.
    Tier 1: weight edit (no recompile).
    Tier 2: architecture change (full recompile).
    """
    score = eval_fn(params, x_eval)
    if score >= capability_threshold:
        return model, params      # no action needed

    # Tier 1: try weight edit first (cheap — no recompile)
    new_params = edit_fn(params, x_eval)
    new_score = eval_fn(new_params, x_eval)
    if new_score >= capability_threshold:
        print(f"Tier-1 edit: {score:.3f} → {new_score:.3f}")
        return model, new_params

    # Tier 2: architecture change (recompile required)
    new_model = arch_fn(model)
    # jax.jit sees a new computation graph → recompiles automatically
    new_score_arch = eval_fn(new_params, x_eval, model=new_model)
    print(f"Tier-2 arch: {score:.3f} → {new_score_arch:.3f}  [recompiled]")
    return new_model, new_params
```

### The Critical Difference: Data vs. Code

ORC JIT reoptimisation changes only the generated code — the data (user inputs, heap state) is unchanged. Neural tiered reoptimisation can change both the generated code (architecture modification → new XLA computation) AND the data (weight editing → new parameter values). This gives neural self-improvement a richer modification space, but also a more complex consistency requirement: after a weight edit, the system must verify that the modified weights remain in the expected value range, that downstream capabilities are not degraded, and that the checkpoint is valid for the new parameter structure.

The ORC JIT guarantee — "after reoptimisation, the function computes the same value" — has no automatic neural analogue. The neural system must construct its own correctness criterion (capability probes, output distribution matching, mechanistic verification) and enforce it explicitly. This is the design space that Chapters 212–218 explore.

### Connecting to jax.jit's Static/Dynamic Axis

ORC JIT's tier boundary is triggered by a count; JAX's recompilation trigger is structural — it fires when the computation graph changes shape. This is expressible precisely: `jax.jit` maintains a cache keyed on the abstract type signatures of all arguments (dtype, shape, and the Python identity of any `static_argnums`). A weight edit that preserves parameter shapes does not change the cache key. An architecture edit that changes the number of layers or the hidden dimension changes the parameter PyTree structure, and therefore the abstract signature, and therefore triggers recompilation.

```python
# Static-argnums example: recompile only when the boolean flag changes
@functools.partial(jax.jit, static_argnames=['use_cache'])
def inference_step(params, x, *, use_cache: bool = False):
    if use_cache:
        return model_with_kv_cache(params, x)
    return model_no_cache(params, x)

# First call: compiles use_cache=False branch
out1 = inference_step(params, x, use_cache=False)
# Second call: cache hit — no recompilation
out2 = inference_step(params_edited, x, use_cache=False)
# Third call: use_cache=True is a new static value → recompile
out3 = inference_step(params, x, use_cache=True)
```

This maps cleanly onto ORC JIT's guard mechanism: the `use_cache` static argument is the "guard condition" that, when violated, triggers recompilation. The `params_edited` change does not trigger recompilation because weight values are not in the cache key — only their shapes and dtypes are. This is the neural self-improvement system's performance contract: weight surgery is free (no recompile), and architectural surgery costs exactly one recompilation per distinct architecture shape.

---

## Chapter Summary

- **Neural networks are compiled programs**: Python/Flax source traces to `jaxpr` (functional SSA), lowers to StableHLO (portable MLIR), passes through Linalg/Tensor dialects, generates LLVM IR, and assembles to PTX/SASS. The full compiler is available at runtime — the neural system ships with its toolchain, unlike conventional compiled binaries.

- **Seven IR layers are distinct introspection points**: Python source (AST rewriting), jaxpr (programmatic equation traversal via `JaxprEqn`), StableHLO (MLIR passes via Python bindings and `mlir-opt`), Linalg/Tensor (tiling/fusion analysis), LLVM IR (PGO for CPU paths), PTX (register pressure, predication), and SASS (final microarchitecture scheduling). Each layer has defined inspection tools and a modification cost ranging from milliseconds (Python edits) to not-practical (SASS).

- **Weight checkpoints are structured object files**: SafeTensors stores an 8-byte header-length prefix followed by a JSON tensor descriptor table and a raw data buffer — directly analogous to ELF section headers + data segments. GGUF additionally encodes quantisation type and model architecture metadata in the tensor info array. Both formats support zero-copy `mmap` loading via `MAP_PRIVATE`, identical to how the dynamic linker maps shared libraries.

- **XLA HLO achieves homoiconicity**: the forward pass IS a StableHLO module; `jax.jit(f).lower(x).as_text()` returns the forward computation as inspectable MLIR text. `jax.grad` derives the backward pass at the jaxpr level — the gradient computation is a data structure in the same IR format as the forward pass. The `dot_general` operations in an HLO module map directly to identifiable architectural components (Q/K/V projections, FFN layers).

- **The complete feedback loop** — infer → profile (Perfetto/XLA HLO timing) → introspect (TransformerLens hooks, SAE features) → modify (weight edits via `jax.tree_util.tree_map` or architecture edits via Python source) → recompile (JAX JIT cache, with persistent `JAX_COMPILATION_CACHE_DIR`) — is implementable today with available open-source tools across the JAX/MLIR/LLVM stack.

- **MLIR passes apply to model computations**: `lowered.compiler_ir()` returns an `mlir.ir.Module` object; `mlir.passmanager.PassManager` runs arbitrary MLIR passes in-process. This enables fusion opportunity analysis, tiling strategy evaluation, and DOT-format computation graph visualisation without GPU execution — cheap offline architectural reasoning for self-modifying systems.

- **ORC JIT's tiered recompilation principle applies directly**: the `ReOptimizeLayer`'s profile → threshold → recompile at higher tier pattern maps to the neural tier-0 (frozen weights, cached XLA), tier-1 (weight edit, no recompile), tier-2 (architecture change, full recompile) cycle. The critical difference: neural reoptimisation can modify both code (computation graph) and data (weights), requiring explicit correctness criteria that ORC JIT's value-preservation guarantee provides automatically.

- **This chapter bridges Chapters 208–210 to Chapters 212–218**: the compiler stack (Triton kernels, MLIR lowering, JAX functional transforms) is the substrate that makes neural self-modification tractable. Chapters 212–218 describe what is modified (weights, architecture, inference procedure) and why (capability deficits, interpretability analysis, evolutionary pressure); this chapter establishes how the compiled artifact structure enables safe, efficient modification and redeployment.
