# Chapter 155 — XLA:CPU

*Part XXII — XLA and the OpenXLA Stack*

XLA's CPU backend transforms HLO computations into native code for x86-64 and AArch64 processors. While the GPU backend receives the most attention in ML contexts, the CPU backend is production-critical for inference workloads, unit testing, and platforms where GPUs are absent. Its design is instructive: it emits LLVM IR directly from HLO, leans on Eigen and OneDNN for dense linear algebra, and organizes execution around a **thunk** abstraction that decouples compiled kernels from buffer allocation. This chapter walks through the CPU backend's codegen pipeline, the `IrEmitter` that produces LLVM IR, the OneDNN and Eigen integration paths, and the runtime model that ties compilation output to execution.

---

## 155.1 CPU Backend Architecture

The CPU backend lives in [`xla/backends/cpu/`](https://github.com/openxla/xla/tree/main/xla/backends/cpu) with the core compilation logic in `xla/service/cpu/`. Its pipeline:

```
Optimized HloModule
    │
    ▼
CPU-specific HLO passes
    ├── CpuLayoutAssignment        — prefer row-major for CPU
    ├── CpuFusionPass              — element-wise loop fusion
    ├── ConvCanonicalization       — normalize convolutions
    ├── DotOpEmitter               — choose dot implementation
    └── ParallelTaskAssigner       — assign parallel work
    │
    ▼
IrEmitter (HloInstruction → LLVM IR)
    │
    ▼
LLVM PassManager (optimization)
    │
    ▼
LLVM CodeGen (x86-64 / AArch64)
    │
    ▼
CpuExecutable (native machine code + thunk sequence)
```

The `IrEmitter` is the central artifact: a C++ visitor that walks `HloComputation` in topological order and emits LLVM IR for each instruction. It does not go through MLIR's Linalg representation — this is legacy code that predates MLIR. There is an ongoing effort to route through `linalg.generic` for new ops.

---

## 155.2 CPU-Specific HLO Passes

### 155.2.1 CpuLayoutAssignment

Unlike the GPU backend which prefers column-major (Fortran order) for cuBLAS compatibility, the CPU backend prefers row-major (C order) layouts to match CPU cache behavior and Eigen's default. `CpuLayoutAssignment` propagates these constraints and inserts copy instructions where needed.

### 155.2.2 CpuFusionPass

CPU fusion decisions differ from GPU fusion. On CPU, fusing two element-wise ops into a loop avoids a full second pass over memory, but the tradeoff is different — CPU has fewer threads than a GPU but larger caches. `CpuFusionPass` fuses element-wise chains and also fuses `map` ops into loops:

```cpp
// From xla/service/cpu/cpu_fusible.cc
bool IsCpuFusible(const HloInstruction& inst) {
  // Element-wise ops and broadcasts are fusible
  if (inst.IsElementwise()) return true;
  if (inst.opcode() == HloOpcode::kBroadcast) return true;
  // Transpose is fusible if it can be expressed as a loop permutation
  if (inst.opcode() == HloOpcode::kTranspose) return IsTransposeFusible(inst);
  return false;
}
```

### 155.2.3 DotOpEmitter and OneDNN Selection

`DotOpEmitter` analyzes `kDot` instructions and selects the implementation:

1. **OneDNN GEMM**: for large matrix multiplications on x86-64 with AVX-512
2. **Eigen GEMM**: portable fallback using BLAS-style tiled GEMM
3. **LLVM IR loop**: for small matrices where call overhead dominates
4. **Multi-threaded GEMM**: splits along batch or M dimension for parallel execution

The selection is heuristic-based, considering matrix dimensions, data type, and whether OneDNN is enabled at build time:

```cpp
// Simplified selection logic
DotOpEmitter::DotOpEmitter(const HloInstruction& dot) {
  if (IsOneDNNEnabled() && IsOneDNNSupportedDot(dot)) {
    strategy_ = kOneDNN;
  } else if (ShouldUseEigenForDot(dot)) {
    strategy_ = kEigen;
  } else {
    strategy_ = kLlvmIrLoop;
  }
}
```

---

## 155.3 The IrEmitter

`IrEmitter` is the core of the CPU backend. It inherits from `HloVisitor` and implements a `Handle*` method for each HLO opcode. The emitter maintains an `llvm::IRBuilder<>` and a symbol table mapping HLO values to `llvm::AllocaInst*` or `llvm::GlobalVariable*` entries.

### 155.3.1 Emitting Element-wise Ops

For a fused element-wise computation, `IrEmitter` emits a loop nest over the tensor dimensions:

```cpp
// Simplified from xla/service/cpu/ir_emitter.cc
Status IrEmitter::EmitLoopedElementwiseFusion(
    const HloInstruction& fusion) {
  // Get buffer pointers for all operands and the output
  llvm::Value* output_ptr = GetEmittedValue(fusion);

  // Emit loop nest: for i0 in [0, d0): for i1 in [0, d1): ...
  llvm_ir::ForLoopNest loop_nest(IrName(fusion), &b_);
  std::vector<llvm::Value*> index = loop_nest.AddInductionVariables(shape);

  // Inside the loop body, emit the element computation
  llvm_ir::IrArray output_array(output_ptr, output_type);
  llvm_ir::IrArray::Index element_index(index, shape);

  // Recursively emit the root instruction's expression tree
  TF_ASSIGN_OR_RETURN(llvm::Value* result,
      EmitElementwise(fusion.fused_expression_root(), element_index));
  output_array.EmitWriteArrayElement(element_index, result, &b_);

  return OkStatus();
}
```

The `EmitElementwise` method handles the tree of element-wise ops inside the fused computation recursively, bottom-up.

### 155.3.2 Emitting Reductions

Reduction emission is more complex. For a `reduce([16, 1024], dims=[1], init=0, f=add)`:

```cpp
// Emit: for i0 in [0, 16):
//         acc = init
//         for i1 in [0, 1024):
//           acc = add(acc, input[i0, i1])
//         output[i0] = acc
Status IrEmitter::HandleReduce(HloInstruction* reduce) {
  auto* operand = reduce->operand(0);
  auto* init = reduce->operand(1);
  
  // Create accumulator alloca
  llvm::Value* acc = b_.CreateAlloca(llvm_type, nullptr, "acc");
  b_.CreateStore(GetEmittedValue(*init), acc);
  
  // Emit reduction loops over contracted dimensions
  llvm_ir::ForLoopNest reduction_loops(...);
  for (int64_t dim : reduce->dimensions()) {
    // Inner loop over this dimension
    auto loop = reduction_loops.AddLoop(0, shape.dimensions(dim), "reduce");
    // Inside: acc = f(load(acc), input[...])
    EmitCallToNestedComputation(*reduce->to_apply(), {load_acc, input_elem}, acc);
  }
  // Store final accumulator to output
  ...
}
```

### 155.3.3 Emitting Dot Products

For small dot products emitted as LLVM IR loops:

```cpp
// [m,k] x [k,n] -> [m,n]
// Emit: for i in [0,m): for j in [0,n): for k in [0,k): out[i,j] += lhs[i,k]*rhs[k,j]
Status IrEmitter::HandleDot(HloInstruction* dot) {
  if (dot_emitter_.strategy() == kLlvmIrLoop) {
    // Three nested loops
    return EmitDotLoop(dot);
  } else if (dot_emitter_.strategy() == kEigen) {
    // Emit call to __xla_cpu_runtime_EigenMatMul_F32
    return EmitEigenGemmCall(dot);
  } else {
    // Emit call to OneDNN GEMM
    return EmitOneDNNCall(dot);
  }
}
```

---

## 155.4 OneDNN Integration

OneDNN (formerly MKL-DNN) is Intel's highly-optimized library for deep learning primitives. XLA uses it for GEMM and convolution on x86-64 when available.

### 155.4.1 Matrix Multiply via OneDNN

For a `kDot` selected for OneDNN, `IrEmitter` emits a call to the XLA CPU runtime's OneDNN wrapper:

```cpp
// Runtime function called from emitted LLVM IR:
extern "C" void __xla_cpu_runtime_OneDNNMatMul(
    void* result, void* lhs, void* rhs,
    int64_t m, int64_t n, int64_t k,
    int32_t lhs_stride, int32_t rhs_stride, int32_t res_stride,
    bool transpose_lhs, bool transpose_rhs);
```

This runtime function sets up an `onednn::matmul` primitive descriptor and executes it on the CPU. For BFloat16 inputs on supported Intel CPUs, OneDNN uses the AMX (Advanced Matrix Extensions) instructions, achieving 2× higher throughput than AVX-512 FP32.

### 155.4.2 Convolution via OneDNN

Convolutions follow a similar pattern:

```cpp
// For kConvolution with OneDNN:
extern "C" void __xla_cpu_runtime_OneDNNConvolution(
    void* result, void* input, void* kernel, void* bias,
    const OneDNNConvConfig* config);
```

`OneDNNConvConfig` encodes strides, padding, dilation, groups, and activation type. OneDNN selects the best algorithm (Winograd, direct, or FFT-based) at runtime.

### 155.4.3 OneDNN Graph Fusion

More recent XLA-CPU code integrates OneDNN Graph, which can fuse convolution + bias + activation into a single kernel without XLA's fusion pass:

```cpp
// OneDNN Graph partition: conv + bias_add + relu fused
onednn::graph::partition p = graph.get_partitions()[0];
auto compiled = p.compile({inputs}, {outputs}, engine);
compiled.execute(stream, input_args, output_args);
```

---

## 155.5 Eigen Integration

Eigen is a C++ template library for linear algebra, widely used in TensorFlow's non-XLA path. XLA's CPU backend uses Eigen as a portable fallback when OneDNN is unavailable:

```cpp
// Emitted LLVM IR calls this runtime function:
extern "C" void __xla_cpu_runtime_EigenMatMul_F32(
    const void* run_options, float* out,
    const float* lhs, const float* rhs,
    int64_t m, int64_t n, int64_t k,
    int32_t transpose_lhs, int32_t transpose_rhs);
```

The `run_options` parameter carries the Eigen `ThreadPoolDevice`, enabling multi-threaded execution. Eigen's GEMM uses a tiling algorithm with SIMD inner kernels; it auto-detects SSE4.2, AVX2, and AVX-512 at runtime.

---

## 155.6 Parallelism Model

### 155.6.1 ParallelTaskAssigner

XLA's CPU backend supports intra-op parallelism — splitting a single operation's work across multiple threads. `ParallelTaskAssigner` annotates HLO instructions with a `parallel_task_count` metadata indicating how many parallel splits to use:

```cpp
// From xla/service/cpu/parallel_task_assigner.cc
int64_t ParallelTaskAssigner::GetTargetParallelTaskCount(
    const HloInstruction& inst) {
  // Based on instruction type and available cores
  if (inst.opcode() == HloOpcode::kDot) {
    // Split along the M dimension
    return std::min(shape.dimensions(0), num_threads_);
  }
  if (inst.opcode() == HloOpcode::kConvolution) {
    return EstimateConvParallelism(inst);
  }
  return 1;
}
```

### 155.6.2 Thunk Execution Model

The CPU backend uses a **thunk** execution model where each compiled unit is a `CpuThunk` — a callable that encapsulates one kernel's arguments and invocation. The sequence of thunks constitutes the full program:

```cpp
// From xla/backends/cpu/runtime/thunk.h
class CpuThunk {
 public:
  virtual ~CpuThunk() = default;
  virtual absl::Status Execute(const ExecuteParams& params) = 0;
  
  // Thunk subclasses:
  // KernelThunk: execute a compiled loop kernel
  // CallThunk: invoke a sub-computation
  // ConditionalThunk: conditional branch
  // WhileThunk: while loop with condition + body thunks
  // AllReduceThunk: collective operation
  // CustomCallThunk: delegate to user-registered function
  // OneDNNGemmThunk: OneDNN matmul
  // OneDNNConvThunk: OneDNN convolution
};
```

The thunk sequence is constructed during compilation and stored in `CpuExecutable`. At execution time, thunks are invoked sequentially (or in parallel, for annotated parallel tasks) by the runtime.

### 155.6.3 Thread Pool Integration

XLA CPU supports two thread pool backends:

1. **Eigen ThreadPool**: default; `Eigen::ThreadPoolDevice` provides work-stealing execution
2. **TFRT (TensorFlow Runtime)**: optional; provides `tfrt::ConcurrentWorkQueue` with async task scheduling

The thread pool is injected through `ExecuteParams`:

```cpp
struct CpuExecutable::ExecuteParams {
  Eigen::ThreadPoolDevice* intra_op_thread_pool;
  // Or:
  tfrt::ConcurrentWorkQueue* work_queue;
  // Buffer allocations
  std::vector<se::DeviceMemoryBase> buffers;
};
```

---

## 155.7 Memory Layout and Buffer Management

### 155.7.1 Buffer Allocation Scheme

XLA CPU allocates a single large memory arena for all intermediate buffers. `BufferAssignment` produces a layout where each `BufferAllocation` is a (offset, size) pair within this arena:

```cpp
// Execute: allocate one contiguous block
std::vector<uint8_t> arena(executable->total_allocation_size());

// Each buffer is a slice of the arena
for (auto& alloc : buffer_assignments) {
  void* ptr = arena.data() + alloc.offset();
  buffers[alloc.index()] = ptr;
}
```

This design minimizes `malloc` calls — the entire execution uses one allocation.

### 155.7.2 Input/Output Aliasing

XLA CPU supports output-input aliasing when the calling convention permits. The parameter `donate_args` in JAX's `jit` triggers this: the input buffer is reused as the output buffer if:

1. Input and output have the same shape and layout
2. The input is never used after the computation (last use)
3. The computation's buffer assignment marks the parameter as aliased

---

## 155.8 Vectorization

### 155.8.1 LLVM Auto-Vectorization

XLA CPU relies primarily on LLVM's auto-vectorizer (SLP and Loop vectorizer) to generate SIMD code. The `IrEmitter` emits scalar loops that LLVM then vectorizes:

```cpp
// LLVM optimization pipeline invoked post-emission:
llvm::PassManagerBuilder builder;
builder.OptLevel = 3;
builder.SizeLevel = 0;
builder.Inliner = llvm::createFunctionInliningPass(275);
builder.LoopVectorize = true;
builder.SLPVectorize = true;
```

XLA also adds loop annotations (`llvm.loop.vectorize.width` metadata) to hint the vectorizer when it knows the inner loop trip count is a multiple of the vector width.

### 155.8.2 Target Feature Selection

XLA CPU reads the host CPU features at compile time (when `cpu_id="native"`) or uses target-specific features for cross-compilation:

```cpp
// CPU feature string for compiled code:
// x86-64 with AVX-512
std::string features = "+avx512f,+avx512bw,+avx512dq,+avx512vl";
// AArch64 with SVE
std::string features = "+sve,+sve2";
```

These feature strings are passed to `TargetMachine::create()` and influence both code emission and vectorization decisions.

---

## 155.9 CpuExecutable and the Runtime

`CpuExecutable` (in [`xla/service/cpu/cpu_executable.cc`](https://github.com/openxla/xla/blob/main/xla/service/cpu/cpu_executable.cc)) is the compiled artifact:

```cpp
class CpuExecutable : public Executable {
  // Compiled code, linked as a shared library or in-memory
  std::unique_ptr<OrcJIT> jit_;

  // Thunk sequence for the entry computation
  std::vector<std::unique_ptr<CpuThunk>> thunk_sequence_;

  // Buffer assignment — maps logical HLO values to physical offsets
  std::unique_ptr<BufferAssignment> buffer_assignment_;

  // Constants baked into the executable
  std::vector<se::OwningDeviceMemory> constants_;

  absl::Status ExecuteComputeFunction(
      const ExecuteParams& params,
      absl::Span<const se::DeviceMemoryBase> buffers) override;
};
```

Execution proceeds as:
1. Allocate arena (or reuse from buffer pool)
2. Copy inputs to arena at assigned offsets
3. Execute thunk sequence
4. Extract outputs from arena at assigned offsets

---

## 155.10 Profiling and Debugging CPU Kernels

### 155.10.1 Emitted LLVM IR Dumps

```bash
# Dump LLVM IR before and after optimization
XLA_FLAGS="--xla_dump_to=/tmp/xla --xla_dump_hlo_as_text --xla_cpu_dump_ir" \
    python my_model.py
```

The dump directory will contain `.ll` files for each compiled computation.

### 155.10.2 Intel VTune / Linux Perf

Since XLA CPU compiles to native code via LLVM, standard profiling tools work. For perf to see function names, XLA can register JIT-compiled symbols:

```bash
XLA_FLAGS="--xla_cpu_enable_perf_annotation" python my_model.py
perf record -e cycles python my_model.py
perf report
```

### 155.10.3 CPU Thunk Trace

The `--xla_cpu_enable_thunk_trace` flag logs each thunk execution with wall-clock times, enabling profiling without external tools:

```
[ThunkTrace] KernelThunk "fusion.0" start: 1.234ms
[ThunkTrace] KernelThunk "fusion.0" end: 1.891ms (0.657ms)
[ThunkTrace] OneDNNGemmThunk "dot.1" start: 1.891ms
[ThunkTrace] OneDNNGemmThunk "dot.1" end: 5.234ms (3.343ms)
```

---

## Chapter Summary

- XLA CPU backend emits LLVM IR directly from HLO via the `IrEmitter` visitor; there is no MLIR intermediate step for current production code.
- CPU-specific passes: `CpuLayoutAssignment` (row-major preference), `CpuFusionPass` (element-wise loop fusion), `DotOpEmitter` (chooses OneDNN/Eigen/loop).
- `IrEmitter` emits loop nests for element-wise fusions and reductions; dot products delegate to OneDNN (AMX-capable), Eigen (portable SIMD), or scalar loops.
- OneDNN provides AVX-512 and AMX-accelerated GEMM and convolution; Eigen provides a portable multi-threaded fallback.
- The thunk execution model (`KernelThunk`, `OneDNNGemmThunk`, `WhileThunk`, etc.) separates compiled kernels from buffer management.
- Buffer assignment allocates a single contiguous arena; output aliasing via `donate_argnums` avoids copies.
- LLVM's SLP and Loop vectorizers handle SIMD generation; target feature strings control the ISA level.
- `CpuExecutable` holds the JIT-compiled code, thunk sequence, and buffer assignment; execution is sequential thunk dispatch with optional intra-op parallelism.


---

@copyright jreuben11
