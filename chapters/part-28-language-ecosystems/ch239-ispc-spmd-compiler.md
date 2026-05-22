# Chapter 239 — ISPC: An SPMD Compiler for CPU SIMD via LLVM

*Part XXVIII — Language Ecosystems and Alternative Front-Ends*

Writing high-performance SIMD code with platform intrinsics is notoriously tedious: `_mm256_fmadd_ps` and its friends require the programmer to reason simultaneously about data layout, lane indices, and shuffle patterns. Auto-vectorization in LLVM's middle-end is powerful but unpredictable — it fires on loops with simple dependences and silently falls back to scalar for anything more complex. ISPC (Intel SPMD Program Compiler) fills the gap with a third approach: a C-like language whose execution model is explicitly SPMD (Single Program, Multiple Data), where the programmer writes code for one "instance" and the compiler generates SIMD code that runs a gang of instances in lockstep. This chapter traces ISPC from its type system through the `FunctionEmitContext` IR emitter to multi-target compilation and the task-parallelism layer.

---

## Table of Contents

- [239.1 The SPMD Execution Model](#2391-the-spmd-execution-model)
- [239.2 The Type System: Variability](#2392-the-type-system-variability)
- [239.3 Control Flow and Masking](#2393-control-flow-and-masking)
- [239.4 The ISPC Frontend: Module and AST](#2394-the-ispc-frontend-module-and-ast)
- [239.5 FunctionEmitContext and IR Emission](#2395-functionemitcontext-and-ir-emission)
- [239.6 Target Widths and the ISPCTarget Enum](#2396-target-widths-and-the-ispctarget-enum)
  - [Runtime dispatch](#runtime-dispatch)
- [239.7 The Task Parallelism Layer](#2397-the-task-parallelism-layer)
- [239.8 C/C++ Interoperability and Production Integration](#2398-cc-interoperability-and-production-integration)
  - [Performance characteristics](#performance-characteristics)
  - [Building and linking](#building-and-linking)
- [Summary](#summary)

---

## 239.1 The SPMD Execution Model

ISPC's central abstraction is the **gang**: a fixed set of program instances that execute together. The gang width equals the SIMD width of the target (8 for AVX2 `float`, 16 for AVX-512 `float`). The programmer writes a scalar-looking function, but every statement executes across all active instances simultaneously.

```ispc
// ISPC source — conceptually one instance
export void scale(uniform float *uniform out,
                  uniform float *uniform in,
                  uniform int count,
                  uniform float factor) {
    foreach (i = 0 ... count) {   // distributes i across gang
        out[i] = in[i] * factor;
    }
}
```

When compiled to AVX2 with gang width 8, the loop body becomes a single `vmulps` operating on `<8 x float>`. The programmer never mentions SIMD; the compiler handles widening.

**Key identifiers visible inside a gang:**
- `programIndex` — the lane index within the gang (0..programCount−1), type `varying int`
- `programCount` — the gang width, type `uniform int` (a compile-time constant per target)

**vs. auto-vectorization**: LLVM's auto-vectorizer analyses a loop after it has been compiled to scalar IR and attempts to widen it. ISPC emits widened LLVM IR directly — the vectorization is guaranteed by construction, not a heuristic.

**vs. CUDA/OpenCL**: CUDA and OpenCL target GPUs with thousands of threads. ISPC targets CPUs with hardware SIMD of width 4–64. No runtime device management, no memory transfer APIs, no warp divergence penalties — the entire program is CPU-resident.

**Production use**: Embree (Intel's ray intersection library), OSPRay (ray tracing renderer), USD Hydra Storm, and RenderMan use ISPC for their intersection and shading kernels. Embree's `rtcIntersect8` call dispatches to ISPC code compiled for the runtime-detected SIMD width.

---

## 239.2 The Type System: Variability

ISPC's type system is a superset of C's with one orthogonal dimension added: **variability**. Every type has a variability qualifier that determines whether a value is the same across all lanes (uniform) or can differ per lane (varying).

The implementation lives in `src/type.h`. The `Variability` struct is:

```cpp
// src/type.h
struct Variability {
    enum VarType { Unbound, Uniform, Varying, SOA };
    VarType type;
    int soaWidth;  // for SOA<N> — struct-of-arrays width
};
```

The default variability is `Varying`. `uniform` must be written explicitly:

```ispc
varying float v;     // per-lane float — the default
uniform float u;     // single float shared across all lanes
float implicit;      // also varying — default
```

**Variability propagation rules:**
- `uniform op uniform → uniform`
- `uniform op varying → varying` (the uniform value is broadcast to a vector)
- `varying op varying → varying`
- `varying → uniform`: requires an explicit reduction (`reduce_add`, `reduce_min`, `extract`, etc.) — the compiler rejects silent narrowing

In LLVM IR terms:
- `uniform float` → `float` (scalar)
- `varying float` on AVX2 → `<8 x float>`

This one-to-one mapping is the key to ISPC's predictability: the programmer can read the LLVM IR and immediately know which operations became scalar and which became vector.

**SOA (Structure of Arrays)**: The `soa<N>` variability marks a struct that should be stored in interleaved struct-of-arrays layout rather than array-of-structs. ISPC emits the interleaving permutation automatically:

```ispc
struct Particle {
    float x, y, z;
    float mass;
};
// soa<8> Particle array: stored as x[8], y[8], z[8], mass[8]
// rather than {x,y,z,mass}[8]
uniform soa<8> Particle particles[1024];
```

---

## 239.3 Control Flow and Masking

The most subtle aspect of SPMD compilation is control flow divergence: when an `if` condition evaluates to `true` for some lanes and `false` for others, both branches must execute with the inactive lanes masked off.

ISPC represents the active-lane set as a **mask** — a `<N x i1>` LLVM vector (or an integer bitmask, target-dependent). The mask is threaded through every basic block.

```ispc
// ISPC
if (x > 0.0f) {
    result = sqrt(x);   // only active for lanes where x > 0
} else {
    result = 0.0f;      // only active for lanes where x <= 0
}
```

Compiled IR (AVX2, width 8):

```llvm
%cond = fcmp ogt <8 x float> %x, zeroinitializer
; then-block executes with mask = old_mask & cond
%sqrt_result = call <8 x float> @llvm.sqrt.v8f32(<8 x float> %x)
; else-block executes with mask = old_mask & ~cond
; final merge using select:
%result = select <8 x i1> %cond, <8 x float> %sqrt_result,
                                  <8 x float> zeroinitializer
```

This "predicated execution" model has an important implication: **both branches always execute**. A side-effecting call in the `else` block runs for all lanes even if only some lanes will "see" the result. For memory operations, masked stores/loads use `@llvm.masked.store`/`@llvm.masked.load` to limit actual memory traffic to active lanes.

**foreach loop**: The `foreach` statement is ISPC's primary parallel iteration construct. It distributes iterations across the gang and handles boundary conditions (when `count` is not a multiple of the gang width):

```ispc
foreach (i = 0 ... n) {
    out[i] = in[i] * factor;
}
```

ISPC emits a peeled loop: a full-width inner loop for `n / programCount` iterations, and a masked tail iteration for the remainder. The tail iteration sets the mask to disable out-of-bounds lanes before entering the loop body.

**foreach_active**: Iterates over only the currently active lanes, extracting one scalar per iteration. Useful for operations that must happen serially (e.g., appending to a per-thread buffer):

```ispc
foreach_active (lane) {
    // lane is uniform — a single active lane index
    append_item(buffers[lane], values[lane]);
}
```

This compiles to a `while (mask != 0)` loop that uses `count_trailing_zeros(mask)` to find the next active lane.

---

## 239.4 The ISPC Frontend: Module and AST

ISPC is a standalone compiler (`src/`) that calls into the LLVM C++ API. It does not use Clang's parser or Sema.

The top-level compilation unit is `Module` (`src/module.h`):

```cpp
class Module {
    SymbolTable *symbolTable;
    llvm::Module *module;          // the LLVM module being built
    llvm::DIBuilder *diBuilder;    // DWARF debug info
    const char *filename;
    std::vector<Decl *> globals;
    std::vector<Function *> functions;
};
```

`Module::CompileFile()` drives the pipeline: lex → parse (a hand-written recursive-descent parser) → type-check → code generation → LLVM IR emission → `opt` pipeline → object file.

**Expression nodes** (`src/expr.h`): Every `Expr` carries a `const Type *type` that includes variability. Binary expressions (`BinaryExpr`) track whether each operand is `uniform` or `varying` and emit the appropriate LLVM instruction:
- Both uniform → scalar `fadd`
- Either varying → `fadd <N x float>`
- Uniform mixed with varying → broadcast the uniform operand first (`insertelement` + `shufflevector`)

**Statement nodes** (`src/stmt.h`):

| ISPC statement | AST class | Key emission |
|----------------|-----------|--------------|
| `if (varying_cond)` | `IfStmt` | Mask push/pop, predicated blocks |
| `foreach (...)` | `ForeachStmt` | Full-width loop + masked tail |
| `foreach_active` | `ForeachActiveStmt` | Scalar loop over active lanes |
| `foreach_unique` | `ForeachUniqueStmt` | Loop over distinct values in varying expr |
| `launch[N] fn(...)` | `LaunchStmt` | `ISPCLaunch` callback |
| `sync` | `SyncStmt` | `ISPCSync` callback |

---

## 239.5 FunctionEmitContext and IR Emission

`FunctionEmitContext` (`src/ctx.h`) is ISPC's equivalent of Clang's `CodeGenFunction`. It wraps an `llvm::IRBuilder<>` and maintains the execution state for one ISPC function:

```cpp
class FunctionEmitContext {
    llvm::Function *function;
    llvm::BasicBlock *bblock;        // current insertion point
    llvm::BasicBlock *allocaBlock;   // function entry for alloca placement

    // Mask management
    AddressInfo *fullMaskAddressInfo;      // full execution mask (function mask & internal mask)
    AddressInfo *internalMaskAddressInfo;  // mask modified by if/foreach
    AddressInfo *functionMaskAddressInfo;  // top-level call mask

    // Control flow lane tracking
    AddressInfo *breakLanesAddressInfo;
    AddressInfo *continueLanesAddressInfo;
    AddressInfo *returnedLanesAddressInfo;
    AddressInfo *loopMaskAddressInfo;
};
```

The mask is stored in an `alloca` (via `AddressInfo`) rather than as an SSA value, because it is updated by multiple control-flow-merging operations that do not fit the single-assignment model.

**Key mask methods:**

```cpp
llvm::Value *GetFunctionMask();        // top-level call mask (from caller)
llvm::Value *GetInternalMask();        // mask within current scope
llvm::Value *GetFullMask();            // AND of function & internal masks
void SetInternalMask(llvm::Value *m);
void SetInternalMaskAnd(llvm::Value *old, llvm::Value *test);
void SetInternalMaskAndNot(llvm::Value *old, llvm::Value *test);
llvm::Value *Any(llvm::Value *mask);   // any active lane?
llvm::Value *All(llvm::Value *mask);   // all lanes active?
void BranchIfMaskAny(llvm::BasicBlock *true_bb, llvm::BasicBlock *false_bb);
```

For an `if` statement, `IfStmt::EmitCode` calls:
1. `ctx->SetInternalMaskAnd(currentMask, condition)` — activate only true-lanes
2. Emit the then-block
3. `ctx->SetInternalMaskAndNot(currentMask, condition)` — activate only false-lanes
4. Emit the else-block
5. Restore the original mask

**Gather and scatter**: When a `varying` pointer is dereferenced, ISPC emits a gather. Scatter is emitted for writes through varying pointers. These go through ISPC's built-in functions (`src/builtins-decl.h`) which map to LLVM intrinsics:

```cpp
// Conceptual mapping in src/expr.cpp (LoadSymbolExpr::GetValue)
if (ptr->GetType()->IsVaryingType()) {
    // varying pointer → gather
    // emits @llvm.masked.gather.v8f32.v8p0
    result = ctx->gather(ptr, mask, type);
} else {
    // uniform pointer → aligned or unaligned load
    result = ctx->LoadInst(ptr, type);
}
```

ISPC's gather/scatter builtins have naming like `__gather32_float`, `__gather_base_offsets32_float`, `__scatter32_float`. These are declared in ISPC's runtime library (bitcode shipped with the compiler) and linked at compile time.

---

## 239.6 Target Widths and the ISPCTarget Enum

The target determines the gang width and the LLVM CPU features enabled. Targets are defined in `src/target_enums.h`:

```cpp
enum class ISPCTarget {
    // SSE
    sse2_i32x4, sse2_i32x8,
    sse4_i32x4, sse4_i32x8,
    // AVX
    avx1_i32x4, avx1_i32x8,
    avx2_i32x4, avx2_i32x8, avx2_i32x16,
    avx2vnni_i32x4, avx2vnni_i32x8, avx2vnni_i32x16,
    // AVX-512
    avx512knl_x16,
    avx512skx_x4, avx512skx_x8, avx512skx_x16, avx512skx_x64,
    avx512spr_x4, avx512spr_x8, avx512spr_x16, avx512spr_x64,
    // ARM NEON
    neon_i32x4, neon_i32x8,
    // WebAssembly
    wasm_i32x4,
    // Fallback
    generic_i1x256, generic_i32x4, generic_i32x8, generic_i32x16,
    // Sentinel
    none, host,
};
```

The naming convention is `<ISA>_<element-type>x<width>`:
- `avx2_i32x8` — AVX2 ISA, 32-bit elements (float/int), gang width 8
- `avx512skx_x16` — AVX-512 Skylake-X, width 16 (the `x` without type means the width is set per-type from the ISA)

The target drives:
1. The LLVM `TargetMachine` features string (`+avx2,+fma` etc.)
2. The `programCount` constant embedded in generated code
3. All `<N x T>` LLVM vector type widths
4. Which built-in bitcode library is linked (e.g., `builtins-avx2-i32x8.bc`)

### Runtime dispatch

When multiple targets are compiled, ISPC generates a dispatcher that queries CPU features at startup and calls the appropriate specialisation:

```cpp
// ISPC generates this automatically with --target=avx2,avx512skx
void ispc_kernel_dispatch(float *out, float *in, int n) {
    if (cpu_has_avx512skx())
        ispc_kernel_avx512skx_x16(out, in, n);
    else if (cpu_has_avx2())
        ispc_kernel_avx2_i32x8(out, in, n);
    else
        ispc_kernel_sse4_i32x4(out, in, n);
}
```

The CPUID check uses ISPC's generated dispatch code or is deferred to the calling application via the `--pic` + weak-symbol model.

---

## 239.7 The Task Parallelism Layer

ISPC's gang model saturates SIMD but uses only one CPU core. The `task` / `launch` / `sync` keywords add coarse-grained multi-core parallelism on top of fine-grained SIMD:

```ispc
task void computeRow(uniform float *uniform out,
                     uniform float *uniform in,
                     uniform int row,
                     uniform int width) {
    foreach (col = 0 ... width) {
        out[row * width + col] = in[row * width + col] * 2.0f;
    }
}

export void computeAll(uniform float *uniform out,
                       uniform float *uniform in,
                       uniform int rows,
                       uniform int width) {
    launch[rows] computeRow(out, in, programIndex, width);
    sync;
}
```

`launch[rows]` spawns `rows` tasks, each running one invocation of `computeRow` with a different value of `programIndex` (used here as the row index). `sync` blocks until all launched tasks complete.

**Runtime callbacks**: ISPC code calls into a runtime that the application must provide (or use one of ISPC's bundled implementations):

```cpp
// Runtime interface (ispc_tasksys.h)
void *ISPCAlloc(void **handle, int64_t size, int32_t alignment);
void  ISPCLaunch(void **handle, void *func, void *data, int count0,
                 int count1, int count2);
void  ISPCSync(void *handle);
```

`ISPCAlloc` allocates the task descriptor struct. `ISPCLaunch` enqueues the task. `ISPCSync` waits for completion. ISPC ships four implementations:

| File | Runtime |
|------|---------|
| `ispc_tasksys.cpp` | Serial (single-threaded, for debugging) |
| `ispc_tasksys_tbb.cpp` | Intel TBB `tbb::task_group` |
| `ispc_tasksys_omp.cpp` | OpenMP `#pragma omp task` |
| `ispc_tasksys_cilk.cpp` | Intel Cilk Plus |

The application links one of these. If no multi-core tasking is needed, the serial implementation suffices and tasks run inline.

---

## 239.8 C/C++ Interoperability and Production Integration

ISPC functions compiled with `export` are callable from C/C++ with standard C calling conventions. ISPC generates a C header alongside the object file:

```ispc
// kernel.ispc
struct Vertex { float x, y, z; };

export void transform(uniform Vertex *uniform verts,
                      uniform int count,
                      uniform float mat[16]) { ... }
```

Generated `kernel_ispc.h`:

```cpp
// kernel_ispc.h (auto-generated)
struct Vertex { float x; float y; float z; };
namespace ispc {
    void transform(Vertex *verts, int32_t count, float mat[]);
}
```

The `uniform` qualifier disappears in the C header — from C's perspective these are ordinary scalar arguments.

**`extern "C"` and `extern "ISPC"`**: ISPC functions can call C functions via `extern "C"`, and C++ code can declare ISPC functions via `extern "C"` in the header (or simply use the generated header).

**SOA layout in practice**: When the C++ side has AOS (array-of-structs) data and ISPC expects SOA, ISPC can perform the transposition automatically using `soa<N>` pointer types:

```ispc
// ISPC side: takes SOA pointer
export void processSOA(uniform soa<8> Vertex *uniform verts, uniform int n);
```

The `soa<8>` qualifier tells ISPC to emit the interleaving loads needed to fill the `<8 x float>` registers from the SOA-laid-out memory.

### Performance characteristics

| Approach | Pros | Cons |
|----------|------|-------|
| LLVM auto-vectorize | No source changes | Unpredictable, loop-only |
| Intel intrinsics | Full control | Non-portable, unreadable |
| ISPC | Portable, predictable, expressive | Separate compilation step |
| OpenCL/CUDA | GPU scalable | No CPU gang semantics |

ISPC typically achieves 85–95% of hand-written intrinsics performance. The remaining gap is usually due to gather/scatter vs. sequential loads in cases where hand-coded versions use explicit shuffle+insert sequences.

### Building and linking

```bash
# Compile ISPC source, targeting AVX2 and AVX-512
ispc --target=avx2-i32x8,avx512skx-x16 --arch=x86-64 \
     -O2 --pic kernel.ispc -o kernel.o -h kernel_ispc.h

# Multiple targets produce kernel_avx2.o and kernel_avx512skx.o
# plus a kernel_dispatch.o with the CPUID-based selector

# Link with C++ application
g++ -O2 main.cpp kernel_avx2.o kernel_avx512skx.o kernel_dispatch.o \
    ispc_tasksys_tbb.cpp -ltbb -o app
```

The `-O2` flag passes through to LLVM's optimization pipeline via ISPC's `runOptimizationPasses()`. ISPC invokes `llvm::PassBuilder` and adds the standard optimization pipeline on top of the already-vectorized IR. Since the IR is already widened, passes like SLP-vectorizer have little to do — the work is inlining, GVN, and dead-store elimination on the vector operations.

---

## Summary

- ISPC implements SPMD execution on CPUs: the programmer writes code for one instance; the compiler emits LLVM IR with explicit `<N x T>` vector types at the target SIMD width.
- The `Variability` struct (`Uniform`, `Varying`, `SOA`) is tracked per-type in the AST; `uniform` maps to LLVM scalar types, `varying` maps to LLVM vector types of width `programCount`.
- Control flow divergence is handled with an execution mask (`<N x i1>` vector); `if (varying_cond)` runs both branches with masked stores/loads.
- `FunctionEmitContext` (`src/ctx.h`) manages the mask stack, alloca placement, and loop-control-flow lane tracking; it wraps `llvm::IRBuilder<>`.
- `foreach` distributes loop iterations across the gang and handles partial-width tails; `foreach_active` and `foreach_unique` serialize over active lanes or distinct values.
- The `ISPCTarget` enum (`src/target_enums.h`) encodes ISA and width; multi-target builds generate per-target object files plus a CPUID-dispatching wrapper.
- Gather/scatter for varying-pointer dereferences emit `@llvm.masked.gather`/`@llvm.masked.scatter` via ISPC's built-in bitcode library.
- The task system (`launch` / `sync` / `ISPCAlloc` / `ISPCLaunch` / `ISPCSync`) adds multi-core parallelism above the SIMD layer; four runtime implementations (serial, TBB, OpenMP, Cilk) are provided.
- ISPC is used in production by Embree, OSPRay, USD Hydra Storm, and RenderMan for CPU ray-tracing and rendering kernels.
