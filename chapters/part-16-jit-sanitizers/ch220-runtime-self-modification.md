# Chapter 220 — Runtime Self-Modification: Source Introspection, Incremental Recompilation, and ORC Hot-Loading

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

A conventional JIT compiler is a one-way pipeline: a program is given to the compiler, the compiler produces machine code, and execution begins. Runtime self-modification inverts this: the executing program becomes the compiler's client, feeding updated source or IR back into the compilation pipeline while it runs, then hot-loading the result into its own address space. This chapter assembles the LLVM mechanisms that make this possible — LibTooling for AST introspection, `IncrementalCompiler` for incremental recompilation, ORC's `RedirectableSymbolManager` for atomic hot-swap, and embedded bitcode for self-contained IR analysis — into a coherent architecture. The chapter also covers the final frontier: propagating locally verified improvements to peer instances in a distributed fleet.

---

## Table of Contents

- [220.1 The Self-Modification Loop](#2201-the-self-modification-loop)
  - [Why Running-Process-as-Compiler is Hard](#why-running-process-as-compiler-is-hard)
- [220.2 Source Access at Runtime](#2202-source-access-at-runtime)
  - [Build-Time Source Embedding](#build-time-source-embedding)
  - [Source-Path Registry](#source-path-registry)
  - [Serialized AST (PCH/Module Cache)](#serialized-ast-pchmodule-cache)
- [220.3 Runtime AST Introspection via LibTooling](#2203-runtime-ast-introspection-via-libtooling)
  - [Setting Up an In-Process ClangTool](#setting-up-an-in-process-clangtool)
  - [What AST Can and Cannot Tell You](#what-ast-can-and-cannot-tell-you)
  - [Extracting Call Graphs at Runtime](#extracting-call-graphs-at-runtime)
- [220.4 Reasoning: Static Analysis and LLM Integration](#2204-reasoning-static-analysis-and-llm-integration)
  - [Static Analysis Path](#static-analysis-path)
  - [LLM Integration Path](#llm-integration-path)
- [220.5 Incremental Compilation Inside the Process](#2205-incremental-compilation-inside-the-process)
  - [Via IncrementalCompiler (Source-Level)](#via-incrementalcompiler-source-level)
  - [Via IRBuilder / IR Transformation (IR-Level)](#via-irbuilder-ir-transformation-ir-level)
- [220.6 Verification Before Hot-Swap](#2206-verification-before-hot-swap)
  - [ResourceTracker Rollback Pattern](#resourcetracker-rollback-pattern)
  - [Alive2-Style Bounded SMT Equivalence](#alive2-style-bounded-smt-equivalence)
- [220.7 ORC Hot-Loading and Call-Site Redirection](#2207-orc-hot-loading-and-call-site-redirection)
  - [RedirectableSymbolManager](#redirectablesymbolmanager)
  - [ABI Constraints on Hot-Swap](#abi-constraints-on-hot-swap)
- [220.8 Safety Constraints and Failure Modes](#2208-safety-constraints-and-failure-modes)
  - [ABI Drift](#abi-drift)
  - [Global State Corruption](#global-state-corruption)
  - [Compiler Re-Entrancy](#compiler-re-entrancy)
  - [Security: Source and Patch Injection](#security-source-and-patch-injection)
- [220.9 Reference Implementations](#2209-reference-implementations)
  - [SICA (Self-Improving Coding Agent, 2025)](#sica-self-improving-coding-agent-2025)
  - [Pharo Smalltalk](#pharo-smalltalk)
  - [Julia Revise.jl](#julia-revisejl)
  - [Brown et al. Speculative ORC Layers](#brown-et-al-speculative-orc-layers)
- [220.10 IR Self-Analysis via Embedded Bitcode](#22010-ir-self-analysis-via-embedded-bitcode)
  - [Embedding Bitcode at Build Time](#embedding-bitcode-at-build-time)
  - [Reading the Own Binary's Bitcode at Runtime](#reading-the-own-binarys-bitcode-at-runtime)
  - [Running Analysis Passes on Self-Contained IR](#running-analysis-passes-on-self-contained-ir)
  - [Limitations and Security Implications](#limitations-and-security-implications)
- [220.11 Self-Propagation: Distributing Improvements to Peer Instances](#22011-self-propagation-distributing-improvements-to-peer-instances)
  - [The Propagation Protocol](#the-propagation-protocol)
  - [Serialization](#serialization)
  - [Remote Verification and Consistency Protocol](#remote-verification-and-consistency-protocol)
  - [Consistency Protocol Across a Fleet](#consistency-protocol-across-a-fleet)
  - [Reference: Julia Distributed.jl](#reference-julia-distributedjl)
  - [Security: Network-Delivered Bitcode is Arbitrary Code Execution](#security-network-delivered-bitcode-is-arbitrary-code-execution)
- [Chapter Summary](#chapter-summary)

---

## 220.1 The Self-Modification Loop

Self-modification is not a single operation; it is a closed-loop control system with five phases:

```
┌─────────────────────────────────────────────────────────────────┐
│                    SELF-MODIFICATION LOOP                       │
│                                                                 │
│  ┌─────────┐   ┌────────┐   ┌─────────┐   ┌────────┐   ┌────┐ │
│  │ OBSERVE │──▶│ REASON │──▶│ COMPILE │──▶│ VERIFY │──▶│SWAP│ │
│  └─────────┘   └────────┘   └─────────┘   └────────┘   └────┘ │
│       ▲                                                    │    │
│       └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

**OBSERVE**: Gather evidence about current behavior. Sources include runtime profiling data (`perf`, `__builtin_expect` counters, ORC `AddProfilerFunc` call counts), AST introspection via LibTooling, and LLVM IR loaded from embedded bitcode. The observer must answer: *which function is worth recompiling, and why?*

**REASON**: Decide what to change. Two strategies:
- *Static analysis*: a custom Clang checker or LLVM pass identifies optimization opportunities (redundant allocations, unvectorized loops, missed inlining candidates) and produces a structured delta specification — a `ΔSpec` encoding the transformation.
- *LLM-guided*: the AST fragment is serialized as a prompt; an LLM API returns a source patch or IR transformation. LLM output is never trusted; it is a *hypothesis* that must survive VERIFY.

**COMPILE**: Translate the ΔSpec into a new LLVM module. Two paths:
- Source-level: `clang::Interpreter::ParseAndExecute` (the `IncrementalCompiler` API from ch219)
- IR-level: clone the existing `Module*`, apply `opt` passes or `IRBuilder` edits, emit to bitcode

**VERIFY**: Check correctness before committing. Strategies range from type-checking and IR verification (cheap, catches structural errors) through test-harness equivalence checking (moderate) to bounded SMT equivalence via Alive2 (expensive, strong). A new `ResourceTracker` is created before the module is exposed; `RT->remove()` rolls back cleanly on failure.

**SWAP**: Atomically redirect callers to the new implementation. ORC's `RedirectableSymbolManager::redirect()` patches the pointer cell with `std::memory_order_release`; callers see the new function body on their next indirect call without any lock.

The loop's invariant: **at no point is incorrect code observable by callers**. VERIFY guarantees this before SWAP; the `ResourceTracker` mechanism guarantees rollback when verification fails.

### Why Running-Process-as-Compiler is Hard

External JIT (a JIT process compiling for a different target) is straightforward. Running the compiler *inside* the process being modified adds complications:

- **Re-entrancy**: the compiler's memory allocator may be active when the compiler is invoked recursively
- **Symbol table consistency**: a module being compiled may reference symbols that are concurrently being executed
- **W^X enforcement**: pages being compiled must be writable; pages being executed must not be; the swap must be atomic at the pointer level, not the page level
- **Signal safety**: LLVM's pass manager and the ORC session lock are not async-signal-safe

The architecture in this chapter addresses each constraint.

---

## 220.2 Source Access at Runtime

Before the REASON and COMPILE phases can operate on source, the program must be able to read its own source text. Three strategies:

### Build-Time Source Embedding

The simplest: embed source files as byte arrays in the binary at build time.

```cmake
# CMakeLists.txt — embed hot_func.cpp as a C array
add_custom_command(
  OUTPUT hot_func_src.cpp
  COMMAND xxd -i hot_func.cpp > hot_func_src.cpp
  DEPENDS hot_func.cpp
)
```

This generates:
```cpp
// hot_func_src.cpp (generated)
unsigned char hot_func_cpp[] = {
  0x76, 0x6f, 0x69, 0x64, ...
};
unsigned int hot_func_cpp_len = 1234;
```

At runtime, pass `hot_func_cpp` directly to a `MemoryBuffer`:

```cpp
auto MB = llvm::MemoryBuffer::getMemBuffer(
    llvm::StringRef(reinterpret_cast<const char*>(hot_func_cpp),
                    hot_func_cpp_len),
    "hot_func.cpp", /*RequiresNullTerminator=*/false);
```

Advantages: no filesystem dependency, no path resolution. Disadvantages: binary size bloat; source is static (cannot incorporate runtime-discovered specializations).

### Source-Path Registry

Register source file paths at build time, re-read from the filesystem at runtime:

```cpp
// Registered via __FILE__ macro at build time
static const char* kHotFuncPath = PROJECT_SRC_ROOT "/src/hot_func.cpp";

llvm::Expected<std::unique_ptr<llvm::MemoryBuffer>> readSource() {
    return llvm::MemoryBuffer::getFile(kHotFuncPath);
}
```

`PROJECT_SRC_ROOT` is set by CMake's `configure_file`. Requires the source tree to be present at the path recorded at compile time — acceptable for development environments, problematic for deployed binaries.

### Serialized AST (PCH/Module Cache)

Ship a precompiled header (`.gch`) or a Clang module cache (`.pcm`) alongside the binary. At runtime, construct a `CompilerInstance` with `-include-pch` pointing to the serialized AST. Re-parsing cost drops to milliseconds because the AST is already analyzed.

```cpp
clang::CompilerInvocation::CreateFromArgs(
    CI->getInvocation(),
    {"-x", "c++", "-include-pch", "/deploy/hot_func.gch",
     "-", /* stdin placeholder */},
    *Diags);
```

This strategy works when source is unavailable but a precompiled representation is acceptable. The PCH must be regenerated whenever headers change.

---

## 220.3 Runtime AST Introspection via LibTooling

LibTooling (covered in [Chapter 46](../part-07-clang-multilang/ch46-libtooling.md)) was designed for offline batch processing of source trees. It can also run *inside* the process being analyzed, treating the running binary as both the tool runner and the analysis subject.

### Setting Up an In-Process ClangTool

```cpp
#include "clang/Tooling/Tooling.h"
#include "clang/ASTMatchers/ASTMatchers.h"
#include "clang/ASTMatchers/ASTMatchFinder.h"

using namespace clang::ast_matchers;

// Matcher: find all functions with a nested loop > 3 levels deep
auto LoopyFunc = functionDecl(
    hasDescendant(
        forStmt(hasDescendant(
            forStmt(hasDescendant(forStmt().bind("inner")))))),
    unless(isExpansionInSystemHeader()))
    .bind("candidate");

class HotspotCollector : public MatchFinder::MatchCallback {
public:
    std::vector<std::string> candidates;
    void run(const MatchFinder::MatchResult& R) override {
        if (auto* FD = R.Nodes.getNodeAs<clang::FunctionDecl>("candidate"))
            candidates.push_back(FD->getQualifiedNameAsString());
    }
};

std::vector<std::string> findDeepLoopFunctions(const std::string& srcPath) {
    std::string code = readSourceFile(srcPath);
    HotspotCollector collector;
    MatchFinder finder;
    finder.addMatcher(LoopyFunc, &collector);
    clang::tooling::runToolOnCode(
        std::make_unique<clang::tooling::MatchFinderAction>(finder), code);
    return collector.candidates;
}
```

### What AST Can and Cannot Tell You

| Question | AST answer | Runtime profiling answer |
|----------|-----------|--------------------------|
| Which functions have deep loop nests? | Yes (structural) | N/A |
| Which functions are called most often? | No | Yes (perf/PAPI) |
| Which allocations are redundant? | Partially (escape analysis) | Yes (heap profiler) |
| Which branches are predictable? | No | Yes (branch predictor miss rate) |
| Which inlining opportunities exist? | Yes (callee size) | Yes (call frequency) |

The two sources are complementary. The self-modification loop combines both: AST identifies *structurally* eligible candidates; runtime profiles select which candidates are *worth* the recompilation cost.

### Extracting Call Graphs at Runtime

```cpp
#include "clang/Analysis/CallGraph.h"

clang::CallGraph CG;
CG.addToCallGraph(TU->getASTContext().getTranslationUnitDecl());
for (auto& Node : CG) {
    if (Node.second->getDecl())
        llvm::outs() << Node.second->getDecl()->getNameAsString() << "\n";
}
```

The in-process call graph combined with ORC symbol lookup data (which functions are currently executing on which threads) allows the REASON phase to exclude live functions from hot-swap candidates.

---

## 220.4 Reasoning: Static Analysis and LLM Integration

### Static Analysis Path

A custom Clang checker or standalone LLVM pass identifies optimization opportunities and emits a structured `ΔSpec`:

```json
{
  "target_function": "compute_kernel",
  "transformation": "vectorize",
  "params": {
    "loop_variable": "i",
    "vector_width": 8,
    "interleave_count": 2
  },
  "confidence": 0.92
}
```

The `ΔSpec` is consumed by the COMPILE phase, which instantiates the corresponding transformation pass.

Example: a simple analyzer that detects unvectorized hot loops and emits a vectorization spec:

```cpp
void analyzeFunction(const llvm::Function& F,
                     llvm::BlockFrequencyInfo& BFI,
                     std::vector<DeltaSpec>& specs) {
    for (auto& BB : F) {
        if (!isHotBlock(BB, BFI)) continue;
        for (auto& I : BB) {
            if (auto* SI = dyn_cast<llvm::StoreInst>(&I)) {
                if (isLoopInvariantStore(SI, F))
                    specs.push_back({"hoist_invariant_store",
                                     F.getName().str(),
                                     SI->getDebugLoc()});
            }
        }
    }
}
```

### LLM Integration Path

The LLM path serializes an AST fragment as natural language or structured JSON, calls an LLM API, and interprets the response as a source patch:

```cpp
std::string buildPrompt(const clang::FunctionDecl* FD, ASTContext& Ctx) {
    std::string src;
    llvm::raw_string_ostream OS(src);
    FD->print(OS, PrintingPolicy(Ctx.getLangOpts()));
    return "Optimize this C++ function for throughput. "
           "Return only the replacement function body as C++ code:\n" + src;
}

std::string callLLMAPI(const std::string& prompt);  // HTTP call

// The response is a *hypothesis*, not trusted code.
// It must pass VERIFY before being compiled or loaded.
```

Critical constraint: **LLM output is never loaded directly**. It is parsed by Clang's semantic analyzer, type-checked, and run through at least a test harness before any SWAP. The LLM is a code-generation oracle, not an authority.

Cross-references: [Chapter 180 — AI-Guided Compilation](../part-23-mlir-production/ch180-ai-guided-compilation.md), [Chapter 207 — SICA](../part-31-ai-self-modification/ch207-weight-substrates.md), [Chapter 218 — Self-Improvement Fitness Functions](../part-31-ai-self-modification/ch218-self-improvement-fitness.md).

---

## 220.5 Incremental Compilation Inside the Process

Two compilation paths serve different trade-offs:

### Via IncrementalCompiler (Source-Level)

```cpp
#include "clang/Interpreter/Interpreter.h"

// Reuse the IncrementalCompiler from the embedding binary
// (initialized at startup; kept alive for the process lifetime)
extern clang::Interpreter* gInterp;

llvm::Error hotRecompileFromSource(const std::string& newSrc) {
    // Each call to ParseAndExecute adds a new module to a new JITDylib layer.
    // The new definition shadows the old one for future lookups.
    if (auto Err = gInterp->ParseAndExecute(newSrc))
        return Err;
    return llvm::Error::success();
}
```

`clang::Interpreter::ParseAndExecute` is documented in `clang/include/clang/Interpreter/Interpreter.h`. Each call:
1. Parses the source fragment in the context of all previously declared types/functions
2. Lowers to a fresh `llvm::Module`
3. Adds the module to a new `JITDylib` layer (shadowing any earlier definition)
4. Executes any top-level initializers

Trade-offs: full semantic analysis catches all type errors; re-parsing is slower than IR-level editing (~50–200 ms for a 100-line function on a warm compiler instance).

### Via IRBuilder / IR Transformation (IR-Level)

```cpp
llvm::Expected<std::unique_ptr<llvm::Module>>
hotRecompileFromIR(llvm::Module* original,
                   const DeltaSpec& spec,
                   llvm::LLVMContext& Ctx) {
    // Clone the module to avoid mutating live IR
    auto NewM = llvm::CloneModule(*original);

    llvm::Function* F = NewM->getFunction(spec.targetFunction);
    if (!F) return llvm::make_error<llvm::StringError>(
        "function not found", llvm::inconvertibleErrorCode());

    // Apply transformation (example: force inline)
    F->addFnAttr(llvm::Attribute::AlwaysInline);

    // Run selected optimization passes
    llvm::PassBuilder PB;
    llvm::ModulePassManager MPM;
    llvm::ModuleAnalysisManager MAM;
    PB.registerModuleAnalyses(MAM);
    MPM.addPass(llvm::AlwaysInlinerPass());
    MPM.run(*NewM, MAM);

    // Verify before returning
    if (llvm::verifyModule(*NewM, &llvm::errs()))
        return llvm::make_error<llvm::StringError>(
            "module verification failed", llvm::inconvertibleErrorCode());

    return NewM;
}
```

IR-level recompilation is ~10–100× faster than source-level (no parsing, no name lookup) but limited to transformations expressible as IR mutations — no signature changes, no new types.

---

## 220.6 Verification Before Hot-Swap

Correctness verification before committing a new implementation is non-negotiable. The cost/confidence trade-off:

| Strategy | Cost | Confidence | Use when |
|----------|------|-----------|----------|
| LLVM IR verifier | Microseconds | Structural only | Always (mandatory) |
| Clang semantic analysis | Milliseconds | Type/ABI correctness | Source-level changes |
| Test harness (representative inputs) | Variable | Empirical | Algorithmic changes |
| Alive2 bounded SMT equivalence | Seconds–minutes | Proved for bounded bit-widths | Safety-critical code |
| Formal proof assistant export | Hours | Full correctness | Never for online hot-swap |

### ResourceTracker Rollback Pattern

```cpp
// Create a ResourceTracker BEFORE exposing the new module
auto RT = JD.createResourceTracker();
auto TSM = llvm::orc::ThreadSafeModule(std::move(NewM), TSCtx);

if (auto Err = LLJIT->addIRModule(RT, std::move(TSM))) {
    // Compilation failed — remove the tracker, JITDylib is unchanged
    llvm::cantFail(RT->remove());
    return Err;
}

// Run verification
if (!verify(oldImpl, newImpl, testInputs)) {
    llvm::cantFail(RT->remove());  // rollback: callers see old impl
    return llvm::make_error<llvm::StringError>(
        "equivalence check failed", llvm::inconvertibleErrorCode());
}

// Verification passed — proceed to SWAP
```

The `ResourceTracker::remove()` call unloads all JITLink atoms associated with the tracker, restoring the JITDylib to its prior state. Because the new symbol has a higher layer priority but has not yet been redirected, callers still see the old implementation during the verification window.

### Alive2-Style Bounded SMT Equivalence

For algorithmic rewrites where the test harness may miss corner cases, Alive2's API can be called in-process:

```cpp
// llvm/lib/Transforms/InstCombine/InstructionCombining.cpp uses
// alive2::checkTransformation internally in verification builds
// For in-process use, embed the Alive2 library and call:
alive2::Transform t;
t.src = *oldFunc;
t.tgt = *newFunc;
alive2::VerifyTransformation(t);  // throws on counterexample
```

This is expensive (SAT solving per basic block) and only practical for small functions or offline pre-validation. Cross-reference: [Chapter 170 — Alive2 and Refinement](../part-24-verified-compilation/ch170-alive2.md).

---

## 220.7 ORC Hot-Loading and Call-Site Redirection

### RedirectableSymbolManager

The `RedirectableSymbolManager` (header: `llvm/include/llvm/ExecutionEngine/Orc/RedirectableSymbolManager.h`) manages a pointer cell per symbol. The JITLink backend (`JITLinkRedirectableSymbolManager`) allocates an anonymous pointer and a jump stub:

```
Caller instruction:     call [indirect_stub_ptr]
                              │
                   ┌──────────▼──────────┐
                   │  anonymous pointer  │  ← writable page (rw-)
                   │  [0x7f8801234560]   │    value = current impl addr
                   └──────────┬──────────┘
                              │
                   ┌──────────▼──────────┐
                   │  tier-1 or tier-2   │  ← executable page (r-x)
                   │  function body      │
                   └─────────────────────┘
```

`redirect(JD, {{"compute_kernel", newAddr}})` atomically stores `newAddr` into the pointer cell with `std::memory_order_release`. Any thread that loads the pointer after this store will call the new implementation. No existing call instruction is patched; only the pointer value changes.

```cpp
using namespace llvm::orc;

// After VERIFY passes:
SymbolMap redirectMap;
redirectMap[ES.intern("compute_kernel")] =
    ExecutorSymbolDef(newFuncAddr, JITSymbolFlags::Exported);

llvm::cantFail(RSM->redirect(JD, redirectMap));

// Keep the old ResourceTracker alive until in-flight calls drain
// (implementation-specific; simplest: delay RT->remove() by one epoch)
std::this_thread::sleep_for(std::chrono::milliseconds(10));
llvm::cantFail(oldRT->remove());
```

### ABI Constraints on Hot-Swap

Only function *bodies* can be replaced — not signatures. If the new implementation has a different parameter list, callers compiled against the old signature will pass arguments in wrong registers, producing undefined behavior.

ABI-safe changes:
- Pure algorithmic rewrites (same in/out types)
- Adding `[[likely]]`/`[[unlikely]]` hints (no ABI impact)
- Changing optimization level (inline expansion, vectorization)
- Changing memory access patterns (tiling, prefetch insertion)

ABI-unsafe changes (require recompiling all callers before redirect):
- Adding/removing parameters
- Changing return type
- Changing exception specification
- Changing struct layout of any type in the signature

**Inlined callers are not redirectable.** If a hot-swap candidate was inlined into another function before the swap, that copy of the code runs forever — the redirect only affects calls through the JITDylib stub. This is a fundamental limitation of function-granularity hot-swap; whole-module replacement (recompiling the caller module too) is required to retire inlined copies.

---

## 220.8 Safety Constraints and Failure Modes

### ABI Drift

The most common failure: the new implementation has a subtly incompatible ABI with an existing caller. Symptoms: crashes, wrong results, heap corruption. Mitigation: enforce a compile-time ABI compatibility check by linking the new module against the same header snapshot that produced the callers.

### Global State Corruption

Code rollback via `RT->remove()` undoes the *code*. It does not undo mutations to global state that the new code made during the verification phase. If VERIFY runs the new code on test inputs and the new code writes to a global cache, the rollback leaves the global cache in a modified state.

Mitigation: run verification in a subprocess (fork, exec in sandbox, communicate results via pipe) or restrict verification to pure functions.

### Compiler Re-Entrancy

If the self-modification loop is triggered by a signal handler or from within the allocator, and the Clang frontend itself allocates memory, a deadlock or heap corruption can result. The ORC `ExecutionSession` carries a global lock; attempting to compile from a signal handler that preempted an ORC operation will deadlock.

Mitigation: queue self-modification requests to a dedicated background thread; never trigger compilation from signal handlers.

### Security: Source and Patch Injection

Embedded source is readable by any process with `ptrace` or access to `/proc/self/mem`. An attacker who can control the LLM API response, the source path, or the bitcode payload has arbitrary code execution. Trust model requirements:

1. Source paths must be absolute, resolved at build time, compared against an allowlist
2. LLM responses must be treated as untrusted user input and run through Clang's full semantic analysis before use
3. Bitcode received from the network must be verified with a cryptographic signature (HMAC or asymmetric signature over the bitcode payload) before `parseBitcodeFile`

---

## 220.9 Reference Implementations

### SICA (Self-Improving Coding Agent, 2025)

SICA is the closest production example of the full self-modification loop in a deployed system. It operates on Python source (not LLVM IR), but the loop structure — observe behavior, reason about improvements, emit a source patch, test the patch, apply it — is identical. The key insight from SICA: the bottleneck is not compilation speed but verification; SICA spends 80% of loop time running the test suite. Cross-reference: [Chapter 207](../part-31-ai-self-modification/ch207-weight-substrates.md).

### Pharo Smalltalk

Pharo is the mature production reference for live mutable MetaModel with hot-swap. The Pharo VM supports class-level hot-swap (`#become:`) with invariant preservation: all existing instances of the old class are migrated to the new class layout in a single atomic step. The Pharo experience shows that hot-swap is manageable at scale — but requires the language runtime to be designed for it from the start. LLVM's ORC approach approximates this for compiled languages.

### Julia Revise.jl

[`Revise.jl`](https://github.com/timholy/Revise.jl) is a filesystem watcher that detects source changes and incrementally recompiles modified Julia functions using the Julia compiler's internal incremental compilation API. It is the closest production analogue in a compiled language:

```julia
using Revise
# Edit hot_func.jl in an editor; Revise detects the change,
# recompiles the affected method, and replaces the method table entry.
```

Julia's method table uses pointer indirection at the call site, making function replacement atomic from the caller's perspective — the same mechanism as ORC's `RedirectableSymbolManager`. Cross-reference: [Chapter 193](../part-28-julia/ch193-julia-llvm.md).

### Brown et al. Speculative ORC Layers

The academic antecedent: a research system that profiles running ORC-compiled code, identifies hot paths, recompiles with stronger assumptions, and redirects via stub patching. The key contribution is the *epoch-based deallocation* of tier-1 code: old code is kept alive for one epoch (measured in calls, not wall time) to drain in-flight invocations before `RT->remove()`. This is the pattern adopted by `ReOptimizeLayer` in production.

---

## 220.10 IR Self-Analysis via Embedded Bitcode

When source is unavailable at the deployment site, the program can analyze its own compiled IR by reading the bitcode section embedded in its own binary.

### Embedding Bitcode at Build Time

```bash
# Embed LLVM bitcode in the ELF .llvmbc section:
clang -fembed-bitcode -O2 -c hot_func.cpp -o hot_func.o

# Verify the section exists:
llvm-readobj --sections hot_func.o | grep llvmbc
# Section { Name: .llvmbc ... }
```

On macOS, the equivalent section is `__LLVM,__bitcode` in the `__TEXT` segment (used by Apple's App Store thin-archiving).

### Reading the Own Binary's Bitcode at Runtime

```cpp
#include "llvm/Object/ObjectFile.h"
#include "llvm/Bitcode/BitcodeReader.h"
#include "llvm/Support/MemoryBuffer.h"

llvm::Expected<std::unique_ptr<llvm::Module>>
loadSelfBitcode(llvm::LLVMContext& Ctx) {
    // Open the running binary
    std::string exePath = llvm::sys::fs::getMainExecutable(nullptr, nullptr);
    auto MBOrErr = llvm::MemoryBuffer::getFile(exePath);
    if (!MBOrErr) return llvm::createFileError(exePath, MBOrErr.getError());

    // Parse as an object file
    auto ObjOrErr = llvm::object::ObjectFile::createObjectFile(**MBOrErr);
    if (!ObjOrErr) return ObjOrErr.takeError();

    // Find the .llvmbc section
    for (const auto& Sec : (*ObjOrErr)->sections()) {
        llvm::Expected<llvm::StringRef> NameOrErr = Sec.getName();
        if (!NameOrErr) continue;
        if (*NameOrErr != ".llvmbc") continue;

        llvm::Expected<llvm::StringRef> ContentsOrErr = Sec.getContents();
        if (!ContentsOrErr) return ContentsOrErr.takeError();

        // Parse the bitcode
        auto BCBuf = llvm::MemoryBuffer::getMemBuffer(
            *ContentsOrErr, "self.bc", /*RequiresNullTerminator=*/false);
        return llvm::parseBitcodeFile(*BCBuf, Ctx);
    }
    return llvm::make_error<llvm::StringError>(
        "no .llvmbc section found", llvm::inconvertibleErrorCode());
}
```

### Running Analysis Passes on Self-Contained IR

Once the `Module*` is recovered, any LLVM analysis pass can run over it:

```cpp
auto MOrErr = loadSelfBitcode(Ctx);
if (!MOrErr) { handleError(MOrErr.takeError()); return; }
auto& M = **MOrErr;

llvm::PassBuilder PB;
llvm::ModuleAnalysisManager MAM;
llvm::FunctionAnalysisManager FAM;
PB.registerModuleAnalyses(MAM);
PB.registerFunctionAnalyses(FAM);
MAM.registerPass([&]{ return llvm::FunctionAnalysisManagerModuleProxy(FAM); });

// Example: compute call graph
auto& CG = MAM.getResult<llvm::CallGraphAnalysis>(M);
for (auto& Node : CG) {
    if (Node.second->getFunction())
        llvm::outs() << Node.second->getFunction()->getName() << " callers: "
                     << Node.second->getNumReferences() << "\n";
}
```

### Limitations and Security Implications

The bitcode reflects IR at *compile time*, not at *runtime*. It does not contain specializations produced by ORC reoptimization or type-specialization caches. It is a static snapshot.

Security: the `.llvmbc` section exposes the full optimized IR of the binary, which may reveal proprietary algorithms. Strip it for distribution:

```bash
llvm-bitcode-strip -r hot_func.o -o hot_func_stripped.o
# Or at link time:
ld -bitcode_hide_symbols ...  # macOS
```

Use in the self-modification loop: when source is unavailable, the embedded IR serves as the input to the REASON phase. A static analysis pass operating over the IR (rather than the AST) can identify unvectorized loops, missed inlining opportunities, and redundant stores — producing a `ΔSpec` that the COMPILE phase translates into a transformed module.

---

## 220.11 Self-Propagation: Distributing Improvements to Peer Instances

A single-process self-modification loop improves one instance. In a distributed fleet, the same improvement is relevant to all peers running the same binary. Self-propagation extends the loop by distributing a verified improvement to peer processes.

### The Propagation Protocol

```
Local process:
  OBSERVE → REASON → COMPILE → VERIFY → SWAP (local)
                                    │
                           serialize to bitcode
                                    │
                            sign payload (HMAC/EdDSA)
                                    │
                     ┌──────────────▼──────────────┐
                     │  propagation channel         │
                     │  (gRPC, shared-mem ring,     │
                     │   message queue)             │
                     └──────────────┬──────────────┘
                                    │
                    ┌───────────────▼──────────────────┐
                    │         Peer process N            │
                    │  receive → verify signature       │
                    │  → LLVM IR verifier               │
                    │  → sandboxed test harness         │
                    │  → SWAP (if verification passes)  │
                    └───────────────────────────────────┘
```

### Serialization

After local SWAP succeeds, extract the new function from the JITDylib and serialize to bitcode:

```cpp
llvm::SmallVector<char, 0> BitcodeBuffer;
llvm::raw_svector_ostream OS(BitcodeBuffer);
llvm::WriteBitcodeToFile(*NewModule, OS);

// Sign the payload
std::vector<uint8_t> signature = signPayload(
    llvm::ArrayRef<char>(BitcodeBuffer), secretKey);
```

`llvm::WriteBitcodeToFile` is declared in `llvm/include/llvm/Bitcode/BitcodeWriter.h`.

### Remote Verification and Consistency Protocol

Each peer independently verifies the received module before applying the swap:

```cpp
bool applyRemotePatch(llvm::ArrayRef<uint8_t> bitcode,
                      llvm::ArrayRef<uint8_t> signature,
                      llvm::orc::LLJIT& LLJIT,
                      llvm::LLVMContext& Ctx) {
    if (!verifySignature(bitcode, signature, trustedPublicKey))
        return false;  // reject: unauthenticated payload

    auto MB = llvm::MemoryBuffer::getMemBuffer(
        llvm::StringRef(reinterpret_cast<const char*>(bitcode.data()),
                        bitcode.size()), "remote.bc");
    auto MOrErr = llvm::parseBitcodeFile(*MB, Ctx);
    if (!MOrErr) return false;

    if (llvm::verifyModule(**MOrErr, &llvm::errs()))
        return false;  // structural integrity check

    if (!runTestHarness(**MOrErr))  // empirical equivalence
        return false;

    // All checks passed — apply swap
    auto RT = LLJIT->getMainJITDylib().createResourceTracker();
    llvm::cantFail(LLJIT->addIRModule(RT, llvm::orc::ThreadSafeModule(
        std::move(*MOrErr), TSCtx)));
    llvm::cantFail(RSM->redirect(LLJIT->getMainJITDylib(), newSymbols));
    return true;
}
```

### Consistency Protocol Across a Fleet

For systems where all instances must run the same code version (correctness invariants across a distributed protocol), a two-phase commit ensures atomicity:

1. **Prepare**: leader broadcasts bitcode + signature; all peers verify and report `ACCEPT` or `REJECT`
2. **Commit/Abort**: if all peers respond `ACCEPT`, leader broadcasts `COMMIT`; any `REJECT` broadcasts `ABORT`
3. Each peer applies `redirect()` on `COMMIT`; discards the ResourceTracker on `ABORT`

This is expensive (two round trips) but necessary for protocols where mixed-version operation is unsound.

### Reference: Julia Distributed.jl

Julia's `Distributed.jl` provides `@everywhere` to broadcast code to all workers:

```julia
@everywhere using HotFix  # load new module on all workers
@everywhere function compute_kernel(x)  # redefine on all workers
    # new implementation
end
```

This is a higher-level interface over the same concept: serialize compiled code, distribute to peers, apply on each. The underlying mechanism uses Julia's serialization format (a superset of its bitcode) and the Julia method table's pointer indirection for atomic replacement.

### Security: Network-Delivered Bitcode is Arbitrary Code Execution

Any system that accepts bitcode over the network and JIT-loads it is accepting arbitrary native code. Mandatory requirements:

- Cryptographic authentication of the payload (signature verified against a pinned public key)
- Structural verification (`llvm::verifyModule`)
- Sandboxed pre-execution (subprocess, seccomp, namespace isolation)
- Rate limiting and audit logging of every applied patch

Unauthenticated propagation (no signature, no verification) is equivalent to a remote code execution vulnerability by design.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **ORC `ReOptimizeLayer` stabilization (LLVM 22/23)**: The `ReOptimizeLayer` added in LLVM 17 (llvm/lib/ExecutionEngine/Orc/ReOptimizeLayer.cpp) is being hardened for production use — epoch-based old-code reclamation, improved concurrency in `RedirectableSymbolManager`, and support for speculative inlining. Expect stable API in LLVM 23 after the RFC thread on discourse.llvm.org ("ORC Re-optimization Production Readiness", 2025-Q4).
- **`clang::Interpreter` incremental ABI stabilization**: The `ParseAndExecute` / `Interpreter` API (exposed via `clang-repl`) is gaining stable C ABI bindings targeted at embedding in non-C++ hosts (Python, Julia foreign-call). Tracked in LLVM Phabricator D157234 and follow-up patches for LLVM 22.x.
- **`llvm-bitcode-strip` tooling for selective section control**: Apple and LLVM upstream are coordinating on fine-grained bitcode section control — per-function bitcode embedding vs. whole-module — to reduce binary size overhead of `-fembed-bitcode` for self-analysis use cases (relevant to section 220.10).
- **Alive2 in-process library mode**: The Alive2 team (Nuno Lopes et al.) has an open PR to make Alive2's `VerifyTransformation` callable as a library without spawning an external `alive-tv` process, directly enabling the in-process bounded SMT equivalence pattern from section 220.6.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Whole-function live patching via JITLink trampolines**: The JITLink-backend `RedirectableSymbolManager` currently patches only indirect-call stubs; a proposed extension would insert trampolines at function prologues (similar to Linux kernel `kpatch`) to redirect direct-call and inlined sites. This requires cooperation with the register allocator to reserve trampoline space — see the "JITLink live patch" design thread on llvm-dev.
- **LLM-guided IR transformation with MLIR dialect round-trips**: Research prototypes (e.g., Google DeepMind's AlphaCode-IR work) are exploring LLM-generated MLIR transformations round-tripped through the LLVM pipeline instead of source patches. The `clang::Interpreter` path would be replaced by MLIR's `PassManager` applying LLM-specified dialect lowering sequences, with stronger structural guarantees than text-level source patches.
- **`ResourceTracker` cross-dylib rollback**: Current `RT->remove()` operates within a single `JITDylib`. Multi-dylib programs (plugins, shared libraries) that perform cross-library hot-swap need transactional rollback spanning multiple dylibs. The ORC working group has a design sketch for a `TransactionTracker` abstraction covering this case.
- **Distributed fleet self-modification with Byzantine-fault-tolerant consensus**: Extending the two-phase commit propagation protocol (section 220.11) to tolerate Byzantine peers (malicious or compromised instances) requires PBFT or HotStuff-class consensus rather than simple 2PC. Research collaborations between the LLVM JIT group and distributed systems labs (e.g., CMU, ETH) are exploring this for safety-critical autonomous systems.

### 5-Year Horizon (Long-Term, by ~2031)

- **Formally verified hot-swap correctness**: Integration of the Vellvm (Verified LLVM) framework with ORC's `redirect()` semantics to produce machine-checked proofs that the pointer-cell atomic-store swap preserves sequential consistency from the caller's perspective. This would extend CompCert-style verified compilation to cover runtime code modification, an open problem in verified systems.
- **Language-runtime-level hot-swap for C++ with object migration**: C++ lacks Pharo's `#become:` analog for migrating existing object instances to a new class layout after a hot-swap. A proposed C++ extension ("C++ live object migration", joint proposal from LLVM and the C++ committee SG7 reflection group) would use compiler-generated migration trampolines driven by LLVM RTTI metadata, enabling full class-level hot-swap including vtable and data-member layout changes.
- **Self-modifying safety-critical firmware with hardware root-of-trust integration**: Embedding the ORC self-modification loop in RTOS firmware (LLVM Embedded Toolchain for Arm target) with attestation via ARM TrustZone or RISC-V PMP so that each hot-swap is cryptographically attested before execution, enabling firmware self-improvement in deployed IoT and automotive ECU contexts without a service window.

---

## Chapter Summary

- The runtime self-modification loop has five phases: OBSERVE, REASON, COMPILE, VERIFY, SWAP; each phase maps to specific LLVM APIs
- Source access at runtime uses three strategies: build-time embedding (`xxd` arrays), source-path registry (`__FILE__`/`LLVM_SRC_ROOT`), and serialized AST (PCH/`.pcm`)
- LibTooling's `ClangTool` and `ASTMatchers` can run inside the process being analyzed; they complement runtime profiling by identifying structurally eligible optimization candidates
- The REASON phase uses either static analysis (custom checker → `ΔSpec`) or LLM-guided patch generation; LLM output must be verified, never directly loaded
- Two compilation paths serve different cost/flexibility trade-offs: `IncrementalCompiler::ParseAndExecute` for source-level changes (full semantics, slower) and `CloneModule` + pass application for IR-level changes (faster, limited to signature-preserving transforms)
- `ResourceTracker::remove()` provides atomic rollback; the VERIFY phase runs under the tracker before `redirect()` exposes the new implementation
- `RedirectableSymbolManager::redirect()` atomically patches callers via an indirect pointer; inlined call sites are not redirectable and require whole-module replacement
- Embedded bitcode (`.llvmbc` / `__LLVM,__bitcode`) enables IR self-analysis without source; `llvm::object::ObjectFile::createObjectFile()` + `parseBitcodeFile()` recovers the `Module*` at runtime
- Self-propagation serializes a verified improvement to bitcode, ships it over a channel with a cryptographic signature, and applies independent VERIFY + SWAP on each peer; unauthenticated network-delivered bitcode is an arbitrary code execution primitive

---

@copyright jreuben11
