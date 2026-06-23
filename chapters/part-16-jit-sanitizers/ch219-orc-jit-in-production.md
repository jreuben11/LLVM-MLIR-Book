# Chapter 219 — ORC JIT in Production: clang-repl, LLDB, PostgreSQL, Numba, and Halide

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

Chapter 108 covered the ORC JIT machinery: JITDylibs, the layered compilation stack, lazy emission, `ReOptimizeLayer`, redirectable symbols, and the W^X memory model. This chapter covers the other half — what that machinery looks like when five different production systems wire it up. Interactive C++ REPLs, debugger expression evaluators, database query compilers, scientific Python runtimes, and image-processing pipeline compilers all converge on the same underlying pattern. Reading the five implementations side-by-side makes the pattern explicit, and the taxonomy in §219.8 names the two axes along which every ORC deployment varies.

---

## Table of Contents

- [219.1 The ORC Production Recipe](#2191-the-orc-production-recipe)
- [219.2 clang-repl and Cling: Interactive C++ over ORC](#2192-clang-repl-and-cling-interactive-c-over-orc)
  - [Background](#background)
  - [Incremental Compilation via `IncrementalCompiler`](#incremental-compilation-via-incrementalcompiler)
  - [Redefinition and Layer Shadowing](#redefinition-and-layer-shadowing)
  - [Value Printing](#value-printing)
- [219.3 LLDB: JIT-Compiling Debugger Expressions](#2193-lldb-jit-compiling-debugger-expressions)
  - [The Expression Evaluator Pipeline](#the-expression-evaluator-pipeline)
  - [Module Synthesis](#module-synthesis)
  - [Symbol Resolution Against the Inferior](#symbol-resolution-against-the-inferior)
  - [DynamicCheckerFunctions](#dynamiccheckerfunctions)
  - [Ephemeral Lifetime](#ephemeral-lifetime)
- [219.4 PostgreSQL: Plan-Time Query JIT](#2194-postgresql-plan-time-query-jit)
  - [Why PostgreSQL JITs](#why-postgresql-jits)
  - [Cost Thresholds](#cost-thresholds)
  - [IR Construction via the C API](#ir-construction-via-the-c-api)
  - [JIT Compilation and Lookup](#jit-compilation-and-lookup)
  - [Performance Characteristics](#performance-characteristics)
- [219.5 Numba: Type-Specialised Python via LLVM](#2195-numba-type-specialised-python-via-llvm)
  - [The JIT Decorator](#the-jit-decorator)
  - [llvmlite: LLVM IR from Python](#llvmlite-llvm-ir-from-python)
  - [Type Specialization Cache](#type-specialization-cache)
  - [CUDA Target](#cuda-target)
- [219.6 Halide: Schedule-Driven Pipeline JIT](#2196-halide-schedule-driven-pipeline-jit)
  - [Algorithm and Schedule Separation](#algorithm-and-schedule-separation)
  - [JIT Compilation](#jit-compilation)
  - [AOT Mode: Same IR, Different Emission](#aot-mode-same-ir-different-emission)
  - [Target and CPU Feature Detection](#target-and-cpu-feature-detection)
- [219.7 WAVM: WebAssembly via LLVM ORC](#2197-wavm-webassembly-via-llvm-orc)
  - [The Translation Pipeline](#the-translation-pipeline)
  - [JITDylib Layout](#jitdylib-layout)
  - [Linear Memory Model](#linear-memory-model)
  - [WAVM versus Wasmtime/Cranelift](#wavm-versus-wasmtimecranelift)
- [219.8 Taxonomy: Six Systems, One Pattern](#2198-taxonomy-six-systems-one-pattern)
  - [Two Axes](#two-axes)
  - [The Self-Modification Loop](#the-self-modification-loop)
- [Chapter Summary](#chapter-summary)

---

## 219.1 The ORC Production Recipe

Every production use of ORC JIT follows a variant of the same sequence. Naming it first makes the per-system sections easier to read.

```
1. Obtain LLVM IR
      Clang lowering (clang-repl, LLDB)
      IRBuilder<> construction (PostgreSQL, Numba via llvmlite)
      Language-specific codegen (Halide's CodeGen_LLVM, WAVM's translator)

2. Create or select a JITDylib
      ES.createJITDylib("name")    — for scoped, releasable units
      J->getMainJITDylib()         — for persistent accumulation

3. Add the module
      J->addIRModule(JD, ThreadSafeModule(std::move(M), Ctx))

4. Resolve the entry point
      auto Sym = cantFail(J->lookup("function_name"));
      auto *FP  = Sym.toPtr<RetTy(ArgTy...)>();

5. Call
      RetTy result = FP(args...);

6. Release (optional)
      RT->remove();   // drop a ResourceTracker; collected at next safe point
```

The minimal working skeleton in LLVM 22:

```cpp
#include "llvm/ExecutionEngine/Orc/LLJIT.h"
#include "llvm/IR/IRBuilder.h"

using namespace llvm;
using namespace llvm::orc;

ExitOnError ExitOnErr;

int main() {
  InitializeNativeTarget();
  InitializeNativeTargetAsmPrinter();

  auto J = ExitOnErr(LLJITBuilder().create());

  // Build a trivial module: int add(int a, int b) { return a + b; }
  auto Ctx = std::make_unique<LLVMContext>();
  auto M   = std::make_unique<Module>("recipe", *Ctx);
  IRBuilder<> B(*Ctx);

  auto *FTy = FunctionType::get(B.getInt32Ty(),
                                {B.getInt32Ty(), B.getInt32Ty()}, false);
  auto *F   = Function::Create(FTy, Function::ExternalLinkage, "add", *M);
  B.SetInsertPoint(BasicBlock::Create(*Ctx, "entry", F));
  auto Args = F->args().begin();
  Value *A = &*Args++, *B_ = &*Args;
  B.CreateRet(B.CreateAdd(A, B_));

  ExitOnErr(J->addIRModule(
      ThreadSafeModule(std::move(M), std::move(Ctx))));

  auto Sym = ExitOnErr(J->lookup("add"));
  auto *AddFn = Sym.toPtr<int(int, int)>();
  llvm::outs() << "3 + 4 = " << AddFn(3, 4) << "\n";
}
```

Two parameters vary across every real deployment:

**IR provenance** — where the LLVM IR comes from. The three cases are Clang lowering (a full frontend parse-and-lower pipeline), programmatic `IRBuilder<>` construction, and a language-specific codegen layer that sits between the two.

**JITDylib lifetime** — whether compiled code is ephemeral (released after each use), persistent-but-versioned (accumulates with redefinition), or persistent-and-shared (lives for the process lifetime). This choice drives nearly every architectural difference between the systems below.

---

## 219.2 clang-repl and Cling: Interactive C++ over ORC

### Background

`clang-repl` is an interactive C++ interpreter built on top of Clang and ORC JIT, shipped in-tree since LLVM 13. It is the mainline successor to Cling, the interpreter developed at CERN for the ROOT data analysis framework. Cling pioneered the approach; `clang-repl` absorbed the lessons and integrated them upstream. Both systems implement the same core model: each line of C++ input is compiled and executed inside a running process, with the accumulated symbol environment available to subsequent lines.

```bash
$ clang-repl
clang-repl> int x = 6 * 7;
clang-repl> x
(int) 42
clang-repl> auto square = [](int n) { return n * n; };
clang-repl> square(x)
(int) 1764
```

### Incremental Compilation via `IncrementalCompiler`

The central abstraction is `clang::IncrementalCompiler` (`clang/include/clang/Interpreter/Interpreter.h`). Each input unit goes through a full Clang pipeline — preprocessing, parsing, semantic analysis, and IR generation — but only for the incremental delta: declarations and statements not yet seen in this session.

```cpp
clang::IncrementalCompilerBuilder CB;
CB.SetCompilerArgs({"-std=c++20"});
auto CI = llvm::cantFail(CB.CreateCpp());

auto Interp = llvm::cantFail(clang::Interpreter::create(std::move(CI)));

// Each call compiles and executes one translation unit fragment
llvm::cantFail(Interp->ParseAndExecute("int answer = 42;"));
llvm::cantFail(Interp->ParseAndExecute("answer *= 2;"));

// Retrieve a value back across the JIT boundary
clang::Value V;
llvm::cantFail(Interp->ParseAndExecute("answer", &V));
// V now holds 84
```

Internally, each `ParseAndExecute` call:
1. Feeds the input to `ASTConsumer` → `CodeGenerator`, producing an LLVM `Module`.
2. Wraps global initializers in a synthetic `__clang_repl_module_init_N()` function.
3. Adds the module to a fresh `ResourceTracker` in the session's primary `JITDylib`.
4. Calls `J->lookup("__clang_repl_module_init_N")` and immediately invokes the result to run top-level statements.

### Redefinition and Layer Shadowing

When the user redefines a function — `int f(int x) { return x+1; }` followed later by `int f(int x) { return x*2; }` — the second definition is added in a new JITDylib layer. ORC's symbol lookup walks layers in LIFO order, so the new `f` shadows the old. The old compiled code is not destroyed: any execution frames already inside the old `f` continue safely. The old `ResourceTracker` is retained until explicitly dropped (via `%undo`) or until the session ends.

The `%undo` meta-command pops the most recent `ResourceTracker` from the layer stack and calls `RT->remove()`, effectively rolling back the last input. This is the mechanism ROOT uses to let physicists correct mistakes at the REPL without restarting the analysis session.

### Value Printing

When the user types an expression that produces a value, `clang-repl` synthesizes a wrapper:

```cpp
// User types: x * 2
// clang-repl synthesizes:
void __clang_repl_Result_print() {
  __clang_repl_Result = (x * 2);
  // call ValuePrinter<decltype(__clang_repl_Result)>
}
```

The `ValuePrinter` is a template that uses `std::ostream` overloads, `std::to_string`, or a fallback hex dump depending on the type. The result is printed without the user writing `std::cout`.

---

## 219.3 LLDB: JIT-Compiling Debugger Expressions

### The Expression Evaluator Pipeline

Every expression you type at the `(lldb)` prompt — `p my_vec.size()`, `expr x * 2`, `call printf("%d\n", n)` — is compiled by a full Clang instance and executed inside the debugged process. The round-trip is: parse → synthesize module → JIT compile → inject into inferior → call → retrieve result → discard compiled code.

The entry point is `CommandObjectExpression::DoExecute` in `lldb/source/Commands/CommandObjectExpression.cpp`, which delegates to `Target::EvaluateExpression`, which constructs a `UserExpression` and calls `UserExpression::Evaluate`.

### Module Synthesis

LLDB's `ClangUserExpression` (`lldb/source/Plugins/ExpressionParser/Clang/ClangUserExpression.cpp`) wraps the user's expression in a synthesized function body:

```cpp
// User types: p vec.size()
// LLDB synthesizes approximately:
extern "C" void $__lldb_expr(void *$__lldb_arg) {
  // inject locals from the current stack frame as references
  std::vector<int> &vec = *reinterpret_cast<std::vector<int>*>(
      ((char*)$__lldb_arg) + offsetof(__lldb_locals, vec));
  // the actual user expression
  __lldb_expr_result_ptr = &(vec.size());
}
```

Local variables from the stopped stack frame are injected as pointer-offsetted fields in a synthesized struct, whose address is passed as `$__lldb_arg`. This lets the expression read and write locals in the inferior process without any special compiler support.

### Symbol Resolution Against the Inferior

JIT'd expressions must be able to call functions that live in the debugged process — `printf`, `std::vector<int>::size`, anything in the inferior's loaded libraries. LLDB implements a custom `SymbolResolver` that, during JIT linking, resolves undefined symbols by:

1. Searching the inferior's symbol table via `Process::GetSymbolAddress`.
2. For lazy-bound PLT stubs, using the dynamic linker's already-resolved GOT entries.
3. For debug-info-only symbols (inlined functions, static functions without export), using DWARF to locate the code.

The resolved addresses point directly into the inferior process's virtual address space. The JIT'd code and the inferior share the same address space (LLDB and the debuggee run in the same process on macOS with `SBDebugger::Create`; on Linux, LLDB uses `ptrace` + `mmap` to load the compiled code into the inferior's address space and executes it there via a `longjmp`-style trampoline).

### DynamicCheckerFunctions

LLDB optionally injects safety guards into JIT'd expressions via `DynamicCheckerFunctions` (`lldb/source/Expression/DynamicCheckerFunctions.cpp`). Before every pointer dereference in the expression, LLDB inserts a call to `$__lldb_check_ptr_size` — a function resident in the inferior — which validates that the pointer is non-null and lies within a mapped, readable region. If the check fails, it raises `SIGTRAP` rather than crashing with a SIGSEGV, giving LLDB control to report a clean error.

### Ephemeral Lifetime

Unlike `clang-repl`, LLDB does not accumulate compiled expressions. Each expression gets its own `ResourceTracker`; the tracker is released as soon as `UserExpression::Evaluate` returns. The JITDylib for expressions is cleared between evaluations. This keeps the LLDB JIT from accumulating stale code across a debugging session that may evaluate thousands of expressions.

---

## 219.4 PostgreSQL: Plan-Time Query JIT

### Why PostgreSQL JITs

PostgreSQL's expression evaluator is a tree-walking interpreter: for each row it processes, it walks the expression tree, dispatches on node type, and recursively evaluates sub-expressions. For analytical queries that process millions of rows, the interpreter overhead — the indirect calls, the switch dispatch, the per-node heap allocations — dominates over the actual arithmetic. JIT-compiling the expression to a native function eliminates the interpreter loop entirely.

Since PostgreSQL 11, the JIT subsystem (`src/backend/jit/llvm/`) engages automatically when query cost exceeds a threshold, replacing the interpreter with a freshly compiled native function for the duration of that query.

### Cost Thresholds

Two GUC (Grand Unified Configuration) parameters govern engagement:

```sql
SET jit_above_cost = 100000;         -- enable JIT at all (default: 100000)
SET jit_optimize_above_cost = 500000; -- use O2 instead of O0 (default: 500000)
```

At plan time, the planner estimates total row-processing cost. If it exceeds `jit_above_cost`, `ExecInitExpr` flags the expression for JIT. If it exceeds `jit_optimize_above_cost`, the module is compiled at `LLVMCodeGenLevelDefault` (O2); below that threshold but above `jit_above_cost`, it uses `LLVMCodeGenLevelNone` (O0) to minimize compilation latency.

### IR Construction via the C API

PostgreSQL uses LLVM's C API (`llvm-c/`) rather than the C++ API, deliberately trading expressive power for ABI stability across LLVM major versions. The JIT module is loaded as a shared library (`pg_llvm.so`); using the C API means PostgreSQL's server binary does not need to be compiled against a specific LLVM C++ ABI.

Expression evaluation IR is built in `llvmjit_expr.c`:

```c
LLVMValueRef
llvm_compile_expr(ExprState *state)
{
    LLVMJitContext *context = (LLVMJitContext *) state->parent->state->es_jit;
    LLVMModuleRef  module   = context->module;
    LLVMBuilderRef builder  = LLVMCreateBuilder();

    /* one function per ExprState */
    LLVMTypeRef  sig  = LLVMFunctionType(LLVMInt1Type(),
                            (LLVMTypeRef[]){LLVMPointerType(LLVMVoidType(), 0),
                                           LLVMPointerType(LLVMVoidType(), 0)},
                            2, false);
    LLVMValueRef func = LLVMAddFunction(module, "eval_expr", sig);
    LLVMBasicBlockRef entry = LLVMAppendBasicBlock(func, "entry");
    LLVMPositionBuilderAtEnd(builder, entry);

    /* walk ExprEvalSteps, emit IR for each opcode */
    for (int i = 0; i < state->steps_len; i++) {
        ExprEvalStep *op = &state->steps[i];
        switch (op->opcode) {
        case EEOP_CONST:
            /* load constant value from op->d.constval */
            ...
        case EEOP_ADD:
            LLVMBuildAdd(builder, lhs, rhs, "add");
            ...
        }
    }
    LLVMBuildRet(builder, result);
    LLVMDisposeBuilder(builder);
    return func;
}
```

Tuple deforming — extracting column values from PostgreSQL's on-disk heap tuple format — gets the same treatment in `llvmjit_deform.c`. Both functions are emitted into the same module and compiled together.

### JIT Compilation and Lookup

```c
/* compile and load */
LLVMOrcLLJITRef      jit     = context->jit;
LLVMOrcJITDylibRef   dylib   = LLVMOrcLLJITGetMainJITDylib(jit);
LLVMOrcThreadSafeModuleRef tsm =
    LLVMOrcCreateNewThreadSafeModule(module, context->ts_ctx);

LLVMOrcLLJITAddLLVMIRModule(jit, dylib, tsm);  /* ownership transferred */

/* resolve the compiled function */
LLVMOrcJITTargetAddress addr;
LLVMOrcLLJITLookup(jit, &addr, "eval_expr");
state->evalfunc = (ExprStateEvalFunc)(uintptr_t) addr;
```

After `evalfunc` is populated, the expression evaluator calls the native function directly instead of the interpreter for every row in the query. At query teardown (`ExecEndNode`), `LLVMOrcReleaseResourceTracker` discards the compiled module.

### Performance Characteristics

Typical speedups on TPC-H analytical workloads (Q1, Q6, Q14) run 10–30% faster with JIT engaged. The speedup is negligible for OLTP queries that process few rows — compilation overhead (5–50 ms depending on expression complexity) dominates. PostgreSQL's threshold model is specifically designed to ensure JIT only engages when the query is long enough that compilation pays for itself many times over.

---

## 219.5 Numba: Type-Specialised Python via LLVM

### The JIT Decorator

Numba (`numba.pypi.org`) is a JIT compiler for Python functions that uses LLVM as its code-generation backend. The `@jit` decorator intercepts the first call, infers concrete types for all arguments, generates LLVM IR, compiles it, and caches the result keyed on the type signature.

```python
from numba import jit
import numpy as np

@jit(nopython=True)
def dot_product(a, b):
    total = 0.0
    for i in range(len(a)):
        total += a[i] * b[i]
    return total

x = np.ones(1_000_000, dtype=np.float64)
y = np.ones(1_000_000, dtype=np.float64)

# First call: type inference → IR generation → LLVM compile → execute
result = dot_product(x, y)   # slow (~200 ms including compilation)

# Subsequent calls with same types: cached native function
result = dot_product(x, y)   # fast (~1 ms, pure native code)
```

The `nopython=True` flag disables the fallback to the Python interpreter for operations Numba cannot lower. Without it, a function that encounters an unsupported operation silently degrades to interpreter execution; with it, an `TypingError` is raised at JIT time, forcing the programmer to fix the type issue rather than silently losing the speedup.

### llvmlite: LLVM IR from Python

Numba does not use the LLVM C++ API directly. It uses `llvmlite` (`github.com/numba/llvmlite`), a lightweight Python binding to a small subset of LLVM's C API. `llvmlite` exposes `IRBuilder` functionality as Python objects:

```python
from llvmlite import ir

# Numba's codegen layer eventually produces this kind of IR:
dbl  = ir.DoubleType()
fnty = ir.FunctionType(dbl, [ir.PointerType(dbl), ir.PointerType(dbl),
                              ir.IntType(64)])
mod  = ir.Module(name="dot_product")
fn   = ir.Function(mod, fnty, name="dot_product_float64_float64")
block = fn.append_basic_block(name="entry")
builder = ir.IRBuilder(block)
# ... emit loop, load, fmul, fadd ...
builder.ret(accum)

llvm_ir_str = str(mod)   # textual LLVM IR as a Python string
```

The LLVM IR string is passed to `llvmlite.binding.create_module_ref_from_ir()`, which hands it to LLVM's parser, runs the optimization pipeline, and compiles it via a `MCJIT`-style `ExecutionEngine`. Numba's current backend uses MCJIT rather than ORC; migration to ORC is in progress as of LLVM 22 to gain lazy compilation, better resource management, and support for removing specializations when memory pressure is high.

### Type Specialization Cache

Numba maintains a per-function `Dispatcher` object that maps type signatures to compiled function pointers:

```python
# After two calls with different types:
dot_product(np.ones(100, dtype=np.float32), np.ones(100, dtype=np.float32))
dot_product(np.ones(100, dtype=np.float64), np.ones(100, dtype=np.float64))

# Two separate compiled variants exist:
print(dot_product.signatures)
# [(array(float32, 1d, C), array(float32, 1d, C)),
#  (array(float64, 1d, C), array(float64, 1d, C))]
```

Each specialization is a separate LLVM IR module compiled to a separate native function. The dispatcher resolves the right function pointer at call time by matching the argument types against the cache.

### CUDA Target

`@numba.cuda.jit` follows the same pattern but emits NVPTX IR instead of host IR:

```python
from numba import cuda

@cuda.jit
def vector_add(a, b, c):
    i = cuda.grid(1)
    if i < a.shape[0]:
        c[i] = a[i] + b[i]
```

The compiled PTX is loaded into the GPU via the CUDA Driver API (`cuModuleLoadData`, `cuModuleGetFunction`) rather than ORC — ORC handles the host side; the GPU side uses CUDA's own module loading infrastructure. The two compilation paths (CPU ORC, GPU CUDA) share Numba's type inference and IR-generation frontend.

---

## 219.6 Halide: Schedule-Driven Pipeline JIT

### Algorithm and Schedule Separation

Halide (`halide-lang.org`) is a domain-specific language and compiler for image and array processing pipelines. Its central design principle is the separation of *algorithm* — what to compute — from *schedule* — how to compute it. The same algorithm can be compiled with hundreds of different schedules (different loop orderings, tile sizes, vectorization widths, parallelism strategies) without modifying the algorithm specification.

```cpp
#include "Halide.h"
using namespace Halide;

// Algorithm: a 3×3 box blur
Func blur_x, blur_y;
Var x, y, c;

blur_x(x, y, c) = (input(x-1,y,c) + input(x,y,c) + input(x+1,y,c)) / 3;
blur_y(x, y, c) = (blur_x(x,y-1,c) + blur_x(x,y,c) + blur_x(x,y+1,c)) / 3;

// Schedule variant A: simple serial
blur_y.compute_root();

// Schedule variant B: tiled + vectorized + parallelized
blur_y.tile(x, y, xi, yi, 256, 32)
      .vectorize(xi, 8)
      .parallel(y);
blur_x.compute_at(blur_y, yi).vectorize(x, 8);
```

The algorithm is identical in both cases. Only the schedule changes. And the schedule determines the generated LLVM IR — loop structure, vector widths, thread dispatch — completely.

### JIT Compilation

`Pipeline::compile_jit()` takes the current schedule and generates LLVM IR via Halide's `CodeGen_LLVM` backend, then compiles it via ORC:

```cpp
Pipeline p(blur_y);

// Compile with schedule variant B
p.compile_jit();

// Execute on a 4K image
Buffer<uint8_t> input = load_image("input.png");
Buffer<uint8_t> output(input.width(), input.height(), input.channels());
p.realize(output);   // calls the JIT'd function
```

Under the hood, `compile_jit()` calls `JITModule::compile_module_to_llvm_module()`, then `LLJITBuilder().create()`, adds the module, and caches the resulting function pointer in `JITModule`. Subsequent `realize()` calls invoke the cached pointer without recompilation.

To try a different schedule, you modify the schedule and call `compile_jit()` again. Halide discards the previous compiled version (drops the old `JITModule`) and compiles the new one. This compile-then-benchmark cycle is the core of Halide's autoscheduling workflow:

```cpp
// Autoscheduler inner loop (simplified):
for (auto &candidate_schedule : schedule_search_space) {
    apply_schedule(blur_y, candidate_schedule);
    p.compile_jit(Target("host"));
    double t = benchmark([&]{ p.realize(output); });
    if (t < best_time) { best_time = t; best_schedule = candidate_schedule; }
}
```

### AOT Mode: Same IR, Different Emission

Halide's AOT mode (`Pipeline::compile_to_static_library`, `compile_to_object`) uses the same `CodeGen_LLVM` backend — the same IR is generated from the same algorithm + schedule pair. The only difference is the emission step: JIT mode feeds the IR to ORC; AOT mode feeds it to `TargetMachine::addPassesToEmitFile`. This makes Halide a clear illustration that JIT and AOT are not different compilation strategies but different ways of consuming the same compiled artifact.

### Target and CPU Feature Detection

```cpp
// JIT target auto-detects the host CPU
Target t = get_jit_target_from_environment();
// e.g., "x86-64-linux-avx2-avx512f-fma" on a modern Skylake-X

p.compile_jit(t);
```

`get_jit_target_from_environment()` calls `llvm::sys::getHostCPUFeatures()` and maps the result to Halide's `Target` feature flags. A schedule that calls `.vectorize(x, 16)` only emits AVX-512 instructions if the target includes the `avx512f` feature; otherwise, it falls back to 256-bit or 128-bit vectors. The schedule is written once; the vectorization width is resolved at compile time against the detected target.

---

## 219.7 WAVM: WebAssembly via LLVM ORC

### The Translation Pipeline

WAVM (`github.com/WAVM/WAVM`) is a WebAssembly runtime that uses LLVM as its JIT backend, in contrast to Wasmtime which uses Cranelift. WAVM translates the Wasm binary format to LLVM IR function-by-function:

```
Wasm binary (.wasm)
    ↓  WAVM parser (src/WASM/WASMSerialization.cpp)
Wasm IR (WAVM's internal decoded form)
    ↓  WAVM LLVM translator (src/LLVMJIT/LLVMJITCompile.cpp)
LLVM IR (one Function per Wasm function)
    ↓  ORC LLJIT
Native code
```

Each Wasm function maps to one LLVM `Function`. Wasm's structured control flow (`block`, `loop`, `if`) maps cleanly to LLVM basic blocks and branches; the structured nature of Wasm (no arbitrary jumps) eliminates the CFG recovery problem that makes binary lifting hard. Wasm's operand stack maps to LLVM SSA values — each push/pop becomes a value definition/use.

### JITDylib Layout

WAVM creates one ORC `JITDylib` per Wasm module instance. Wasm's import/export model maps directly to JITDylib symbol resolution:

```
Wasm module A exports: "memory", "add_func"
Wasm module B imports: "add_func" from module A

→ Module A's JITDylib defines symbols: "add_func", "__wavm_memory_base"
→ Module B's JITDylib links against Module A's JITDylib
```

This is precisely ORC's intended use case: hierarchical JITDylib linking with explicit import/export control.

### Linear Memory Model

Wasm's linear memory — the flat byte array at the heart of every Wasm module — is mapped as a large anonymous `mmap` region:

```cpp
// WAVM allocates ~4 GB of virtual address space per Wasm memory
void* memory_base = mmap(nullptr, 4ULL * 1024 * 1024 * 1024,
                         PROT_NONE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
// Only committed pages are actually backed by physical memory
mprotect(memory_base, initial_pages * 65536, PROT_READ | PROT_WRITE);
```

The upper portion of the 4 GB region is left `PROT_NONE` — accessing it triggers SIGSEGV. Wasm's 32-bit address space fits entirely within the lower 4 GB, so the guard region catches all out-of-bounds accesses without explicit bounds checks in the JIT'd code. This is the same guard-page technique used by JVMs for Java array bounds checking.

### WAVM versus Wasmtime/Cranelift

| Property | WAVM (LLVM) | Wasmtime (Cranelift) |
|----------|-------------|----------------------|
| Code quality | Higher (LLVM O2 applies full pass pipeline) | Lower (Cranelift is a single-pass compiler) |
| Compilation speed | Slower (~50–200 ms per module) | Faster (~5–20 ms per module) |
| Memory use | Higher (LLVM IR + optimization data structures) | Lower |
| Startup latency | High (unsuitable for serverless cold starts) | Low |
| Maintenance status | Less active | Actively developed |

The trade-off is fundamental: LLVM's optimizer is deep and slow; Cranelift is designed to be fast enough for production JIT use at the cost of leaving optimization on the table. For latency-sensitive deployments (serverless, browser-side Wasm), Cranelift's compilation speed wins. For throughput-oriented workloads where the compiled code runs for seconds or minutes, LLVM's code quality matters more. Most production Wasm runtimes have converged on Cranelift for baseline compilation and either Cranelift-with-optimizations or LLVM as a tiered upgrade for hot code.

---

## 219.8 Taxonomy: Six Systems, One Pattern

The table below places all six systems (including Julia from [Chapter 193](../part-28-language-ecosystems/ch193-julia-type-inference-driven-llvm-specialization.md)) on the two axes identified in §219.1.

| System | IR Provenance | JITDylib Lifetime | Lookup Strategy | ResourceTracker Policy | Distinguishing Choice |
|--------|--------------|-------------------|-----------------|----------------------|-----------------------|
| **clang-repl** | Clang full pipeline | Persistent, layer-stack | Implicit (init function called directly) | Per-input; `%undo` drops layers | Redefinition via layer shadowing; incremental AST |
| **LLDB** | Clang synthesis (expression wrapper) | Ephemeral per expression | By synthesized function name | Dropped immediately after call | Inferior-process symbol resolution; DynamicCheckerFunctions |
| **PostgreSQL** | C API `IRBuilder` | Ephemeral per query plan | By fixed function name (`"eval_expr"`) | Dropped at query teardown | C API for ABI stability; cost-threshold gating |
| **Numba** | `llvmlite` Python `IRBuilder` | Persistent per type signature | Cached function pointer in `Dispatcher` | Never dropped (MCJIT; ORC migration pending) | Python-embedded `IRBuilder`; type-specialization cache |
| **Halide** | `CodeGen_LLVM` schedule-driven | Persistent per schedule; replaced on recompile | Cached in `JITModule` | Dropped when schedule changes | Schedule-as-compiler; AOT/JIT share codegen path |
| **WAVM** | Wasm-to-LLVM-IR translator | Persistent per Wasm module instance | ORC symbol table via import/export map | Never dropped (module instance lifetime) | Guard-page linear memory; JITDylib-per-instance mirrors Wasm import model |
| **Julia** (Ch193) | Julia compiler (`src/codegen.cpp`) | Persistent per `MethodInstance` | Cached in `jl_method_instance_t` | Invalidated on method redefinition | Type-inference-driven specialization; incremental native image |

### Two Axes

**IR provenance** determines compilation latency and expressiveness. Clang-lowered IR (clang-repl, LLDB) pays the full frontend parse cost but can handle arbitrary C++ including templates and overloads. `IRBuilder`-constructed IR (PostgreSQL, Numba) pays only the IR construction cost but requires the calling system to implement its own type system and lowering rules. Language-specific codegen (Halide, Julia, WAVM) is the middle path: a custom frontend that produces LLVM IR without invoking Clang.

**JITDylib lifetime** determines memory use and redefinition semantics. Ephemeral JITDylibs (LLDB, PostgreSQL) are the simplest: compile, run, release. They accumulate no state and have no redefinition semantics. Persistent-versioned JITDylibs (clang-repl, Julia) accumulate compiled specializations and handle redefinition via shadowing or invalidation. Persistent-and-replaced JITDylibs (Halide) discard the old compiled version entirely when the configuration changes.

### The Self-Modification Loop

The pure self-modification pattern described in Chapter 108 — instrument, profile, recompile hotter code, hot-swap — is the `ReOptimizeLayer` use case. The systems in this chapter do not quite reach that model; they compile-on-demand but do not profile and recompile autonomously. The `ReOptimizeLayer` closes the loop: it adds profiling instrumentation to tier-1 code, monitors call counts, and triggers tier-2 recompilation automatically. Julia's `--pkgimages` precompilation and `Revise.jl` are the closest production analogue: profile-informed ahead-of-time caching that approximates the continuous recompilation loop without requiring runtime instrumentation infrastructure.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **Numba ORC migration completion**: Numba's in-progress transition from MCJIT to ORC (tracked in `numba/numba` GitHub issues and llvmlite PRs) is expected to land in llvmlite 0.44+, enabling lazy compilation, `ResourceTracker`-based specialization eviction under memory pressure, and concurrent JIT compilation across Python threads without the GIL as a serialization barrier.
- **clang-repl REPL protocol stabilization**: The `clang::Interpreter` C++ API is being hardened for embedding in third-party tools (e.g., ROOT 6.34+, xeus-clang-repl Jupyter kernel). LLVM discourse RFC "Stabilize the IncrementalCompiler API" (posted Q1 2026) proposes versioned headers under `clang/Interpreter/` separate from the main Clang headers to allow downstream embedding without tracking internal header churn.
- **PostgreSQL 18 JIT improvements**: PostgreSQL 18 development cycle (targeting mid-2026 release) includes patches to jit expression compilation that extend the JIT'd opcode coverage to `EEOP_NULLIF`, `EEOP_PARAM_EXEC`, and aggregate transitions, reducing the fraction of query plans that fall back to the interpreter for partial evaluation.
- **ORC concurrent-compile API hardening**: LLVM 22 introduced `ConcurrentIRCompiler` and `IRCompileLayer` thread-pool support; the 23.x development cycle (open on llvm.org/docs/ORCv2.html) plans to document and test the concurrent path end-to-end, fixing races in `JITDylib::define` when multiple threads race to add the same weak symbol.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Halide autoscheduler + LLVM profile-guided feedback loop**: The Halide community is prototyping an autoscheduler variant that uses LLVM's `ReOptimizeLayer` to collect real execution profiles of compiled schedule candidates, feeding back into the schedule search instead of relying solely on benchmarking. This closes the self-modification loop described in §219.8 for image-pipeline workloads, where hardware prefetcher behavior and cache contention are difficult to model statically.
- **WAVM tiered compilation with Cranelift baseline + LLVM tier-2**: WAVM has an open issue and RFC discussion to adopt a two-tier model: Cranelift for fast tier-1 compilation (matching Wasmtime's startup latency), with ORC `ReOptimizeLayer` promoting hot Wasm functions to LLVM-compiled tier-2. This directly addresses the startup latency problem identified in the §219.7 comparison table while retaining LLVM code quality for sustained workloads.
- **LLDB expression evaluator migration to ORC v2 fully**: LLDB's expression evaluator still retains MCJIT fallback paths for older platforms in LLVM 22 (`lldb/source/Plugins/ExpressionParser/Clang/ClangExpressionParser.cpp`). By 2028, the plan (tracked on LLDB's GitHub project board) is to remove all MCJIT paths, replacing them with ORC's `EPCGenericJITLinkMemoryManager` for remote-process injection, enabling expression evaluation in out-of-process debugger scenarios (e.g., cross-device debugging on embedded targets).
- **Numba CUDA target via MLIR NVVM dialect**: The Numba project is evaluating replacing direct PTX emission from llvmlite with a path through MLIR's `nvvm` and `gpu` dialects, then lowering to PTX via `mlir-translate`. This would give Numba access to MLIR's loop transformation infrastructure (`affine` and `linalg` dialects) for GPU kernel optimization, improving code quality for stencil and tensor workloads beyond what the current direct-PTX backend achieves.

### 5-Year Horizon (Long-Term, by ~2031)

- **Unified JIT ABI for embedding in language runtimes**: The proliferation of per-project JIT embedding patterns (each of the six systems in this chapter has its own bespoke ORC wiring) has motivated a proposal in the LLVM community for a stable `libjit` C ABI — a versioned shared library wrapping ORC that language runtimes (Numba, Julia, PostgreSQL, Halide) could link against without tracking LLVM C++ ABI changes. This would replicate PostgreSQL's C-API approach at the level of the full ORC stack, enabling independent LLVM upgrades in packaging systems like conda and Homebrew.
- **ORC-native ahead-of-time caching (persistent code cache)**: All six systems currently recompile from scratch on process restart. A persistent ORC code cache — serializing compiled `ObjectLayer` output keyed on a cryptographic hash of the input IR + target features — is a long-requested feature (LLVM RFC #0072, discourse.llvm.org, 2023). By 2031, a stable cache format would allow clang-repl sessions, PostgreSQL query plans, and Halide schedules to survive restarts, reducing startup latency for warm workloads to near-zero.
- **Wasm GC and component model support in WAVM-style LLVM backends**: The W3C WebAssembly GC proposal (finalized 2024) and the Component Model specification (in progress) introduce reference types, struct/array heap types, and inter-component function calls that do not map cleanly to LLVM's current type system. LLVM IR extensions for GC reference types (the `gc` attribute family, `statepoint`/`patchpoint` infrastructure) will need to be extended to handle Wasm GC's type hierarchy; this is an active area at the intersection of the Wasm CG and LLVM community, with prototype patches expected by 2027–2028 and production-quality support by 2030–2031.

---

## Chapter Summary

- The ORC production recipe has six steps: obtain IR, create JITDylib, add module, look up symbol, call, release. Every deployment in this chapter is a variation on these steps.
- `clang-repl` accumulates a layer-stack JITDylib; redefinition works by shadowing; `%undo` pops layers. Each input runs a full Clang compilation pipeline.
- LLDB synthesizes a wrapper function around each debugger expression, resolves symbols against the inferior process, optionally injects safety guards, and releases the compiled code immediately after the call.
- PostgreSQL uses the LLVM C API for ABI stability, builds IR for expression evaluation and tuple deforming, and engages JIT only when estimated query cost exceeds a configurable threshold.
- Numba performs type inference on first call, emits LLVM IR via the `llvmlite` Python binding, and caches one compiled specialization per concrete type signature. CUDA target uses the same frontend with an NVPTX backend.
- Halide separates algorithm from schedule; `compile_jit()` compiles the current schedule to a native function; changing the schedule and recompiling is the autoscheduling inner loop. AOT and JIT use the same `CodeGen_LLVM` path.
- WAVM translates Wasm bytecode to LLVM IR function-by-function, mirrors the Wasm import/export model as JITDylib linking, and uses guard-page linear memory to eliminate explicit bounds checks. LLVM produces better code than Cranelift; Cranelift compiles faster.
- The two axes that distinguish ORC deployments are IR provenance (Clang, IRBuilder, or language-specific codegen) and JITDylib lifetime (ephemeral, persistent-versioned, or persistent-replaced).
