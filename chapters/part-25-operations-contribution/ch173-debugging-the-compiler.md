# Chapter 173 — Debugging the Compiler

*Part XXV — Operations, Bindings, and Contribution*

Debugging a compiler is fundamentally different from debugging application code. The program you are debugging is itself a program transformer: the symptoms appear in the output (wrong assembly, wrong IR, a crash in a later pass), and the root cause may be in a pass that ran dozens of transformations earlier. Localizing a compiler bug requires working through two levels: first finding which pass or transformation introduced the error, then understanding why it did so. LLVM and MLIR provide a rich set of tools for both levels — diagnostic flags that dump IR at every stage, debug-only output controlled at fine granularity, bisection tools that isolate the failing pass, and standard interactive debugger techniques adapted for the compiler's deeply recursive structure. This chapter covers the full debugging toolkit, from the first sign of trouble to a precise, actionable bug report.

---

## Table of Contents

- [173.1 Debugging LLVM Passes](#1731-debugging-llvm-passes)
  - [IR Dumping Flags](#ir-dumping-flags)
  - [LLVM_DEBUG and dbgs()](#llvmdebug-and-dbgs)
  - [Pass Statistics](#pass-statistics)
  - [Time Passes](#time-passes)
- [173.2 Crash Debugging](#1732-crash-debugging)
  - [Reading LLVM Crash Messages](#reading-llvm-crash-messages)
  - [llvm-extract: Isolating a Function](#llvm-extract-isolating-a-function)
  - [llvm-reduce: Automated Minimization](#llvm-reduce-automated-minimization)
  - [Clang Crash Reproducers](#clang-crash-reproducers)
  - [bugpoint (Legacy)](#bugpoint-legacy)
- [173.3 Bisecting Regressions](#1733-bisecting-regressions)
  - [git bisect for Compiler Regressions](#git-bisect-for-compiler-regressions)
  - [Pass-Level Bisection](#pass-level-bisection)
  - [Narrowing to a Function](#narrowing-to-a-function)
- [173.4 Debugging MLIR](#1734-debugging-mlir)
  - [MLIR Debug Flags](#mlir-debug-flags)
  - [Pattern Rewriting Debug](#pattern-rewriting-debug)
  - [mlir-opt with GDB](#mlir-opt-with-gdb)
  - [dump() in Debug Builds](#dump-in-debug-builds)
  - [Verifying MLIR Invariants](#verifying-mlir-invariants)
- [173.5 Profile-Guided Compiler Debugging](#1735-profile-guided-compiler-debugging)
  - [Profiling a Slow Compilation](#profiling-a-slow-compilation)
  - [Clang's -ftime-trace](#clangs-ftime-trace)
  - [AddressSanitizer for Pass Development](#addresssanitizer-for-pass-development)
  - [UBSan for Pass Development](#ubsan-for-pass-development)
  - [Valgrind and Memory Debugging](#valgrind-and-memory-debugging)
- [llubi: UB-Aware LLVM IR Interpreter](#llubi-ub-aware-llvm-ir-interpreter)
  - [Design and Shadow State](#design-and-shadow-state)
  - [Tracked UB Categories](#tracked-ub-categories)
  - [Use Cases](#use-cases)
  - [Comparison with UBSan](#comparison-with-ubsan)
  - [Status in LLVM 22](#status-in-llvm-22)
  - [Running llubi](#running-llubi)
- [MLIR IDE Support: mlir-lsp-server and mlir-query](#mlir-ide-support-mlir-lsp-server-and-mlir-query)
  - [mlir-lsp-server](#mlir-lsp-server)
  - [Architecture](#architecture)
  - [Building and Running](#building-and-running)
  - [VS Code Integration: mlir-vscode](#vs-code-integration-mlir-vscode)
  - [Custom Dialect Registration](#custom-dialect-registration)
  - [mlir-pdll-lsp-server](#mlir-pdll-lsp-server)
  - [tblgen-lsp-server](#tblgen-lsp-server)
  - [mlir-query: Command-Line IR Pattern Search](#mlir-query-command-line-ir-pattern-search)
  - [Practical Workflow](#practical-workflow)
- [Chapter Summary](#chapter-summary)

---

## 173.1 Debugging LLVM Passes

### IR Dumping Flags

The most basic debugging tool: print the IR before and after passes.

```bash
# Print IR before and after every pass
opt -O2 --print-before-all --print-after-all input.ll -o /dev/null 2>passes.txt

# Print IR only when a pass changes it (less verbose)
opt -O2 --print-changed input.ll -o /dev/null

# Print IR for a specific pass
opt -O2 --print-before=instcombine --print-after=instcombine input.ll -o /dev/null

# Filter to specific passes
opt -O2 --print-changed --filter-print-funcs=interesting_function input.ll
```

The `--print-changed` flag is particularly useful: it shows exactly which passes modified the IR, narrowing down the search space.

### LLVM_DEBUG and dbgs()

LLVM's passes use the `LLVM_DEBUG` macro for conditional debugging output:

```cpp
// In pass code
#include "llvm/Support/Debug.h"
#define DEBUG_TYPE "my-pass"

// This output only appears with -debug or -debug-only=my-pass
LLVM_DEBUG(dbgs() << "Processing instruction: " << *I << "\n");
LLVM_DEBUG({
  dbgs() << "Dominance tree:\n";
  DT.print(dbgs());
});
```

At the command line:
```bash
# Enable all debug output (very verbose)
opt --debug -passes=instcombine input.ll -o /dev/null

# Enable debug output for a specific pass
opt --debug-only=instcombine -passes=instcombine input.ll -o /dev/null

# Multiple passes
opt --debug-only=instcombine,licm -passes=instcombine,licm input.ll
```

The `DEBUG_TYPE` macro sets the tag used by `--debug-only`. Most LLVM passes define their own tag. Common tags:
- `instcombine`: InstCombine
- `licm`: LICM
- `gvn`: GVN
- `loop-rotate`: loop rotation
- `reg-alloc`: register allocation
- `isel`: instruction selection

### Pass Statistics

```bash
# Print statistics collected by passes
opt -O2 --stats input.ll -o /dev/null

# Output (example):
# 23 instcombine - Number of instructions simplified
#  7 licm         - Number of instructions hoisted out of loops
#  3 gvn          - Number of loads eliminated
```

Pass statistics require `LLVM_ENABLE_STATS=ON` in the CMake build configuration (it is off by default in release builds). Statistics help identify which passes are doing work; a pass with zero statistics may not be firing at all, which can itself be a bug.

### Time Passes

```bash
# Print per-pass timing
opt -O2 --time-passes input.ll -o /dev/null

# Output:
# Pass execution timing report
# ----------------------------
# Total Execution Time: 0.123 seconds
# ...InstCombine...    0.045s (36%)
# ...GVN...            0.031s (25%)
# ...LICM...           0.012s (10%)
```

Timing helps identify which pass is slow (for performance debugging) and confirms which passes are actually running.

---

## 173.2 Crash Debugging

### Reading LLVM Crash Messages

LLVM installs signal handlers that catch SIGSEGV, SIGABRT, and assertion failures, printing a backtrace and relevant IR before exiting:

```
LLVM ERROR: Broken module found, compilation aborted!
; ModuleID = 'input.ll'
; Function 'foo': use of undefined value '%undef_val'
Stack dump:
0. Program arguments: opt -passes=instcombine input.ll
1. Running pass 'InstCombinePass' on module 'input.ll'
2. Running pass 'InstCombinePass' on function '@foo'
...
```

Key information in a crash dump:
- **The failing pass**: the most recently running pass at the time of the crash.
- **The function**: which function was being processed.
- **The IR state**: a dump of the current module (usually before the pass that crashed it).

### llvm-extract: Isolating a Function

When debugging a crash in a large module, extract only the crashing function:

```bash
llvm-extract --func=foo large_module.ll -o small.ll
opt -passes=instcombine small.ll
```

If the crash reproduces with the small module, you have a minimal reproducer. If not, the crash depends on cross-function interactions (inline decisions, alias analysis context, etc.).

### llvm-reduce: Automated Minimization

```bash
# Create an "interesting" script that returns 0 when the crash occurs
cat > crash_check.sh << 'EOF'
#!/bin/bash
opt -passes=instcombine "$1" 2>&1 | grep -q "Assertion.*failed" || \
  opt -passes=instcombine "$1" 2>&1 | grep -q "Segmentation fault"
EOF
chmod +x crash_check.sh

llvm-reduce --test=./crash_check.sh --test-arg=%s large_crashing.ll
```

`llvm-reduce` applies successive reductions:
1. Remove functions (keeping only the crashing one).
2. Remove basic blocks.
3. Remove instructions.
4. Replace values with `undef`.
5. Simplify types.
6. Remove attributes and metadata.

The result is typically a 5–50 line LLVM IR file that reliably reproduces the crash.

### Clang Crash Reproducers

Clang generates crash reproducers automatically:

```bash
# Generate a reproducer when Clang crashes
clang -O2 crashing.c
# If Clang crashes, it creates: crashing.c.crash and clang_crash_reproducer.tar.gz

# Manually request a reproducer
clang -gen-reproducer crashing.c
```

The reproducer tarball contains:
- The preprocessed source (`crashing.cpp.i` or `crashing.c.i`).
- The exact Clang command line.
- Any relevant compilation database entries.

This is the canonical format for filing Clang bug reports.

### bugpoint (Legacy)

**Note on bugpoint:** The bugpoint tool (previously in LLVM) was removed in LLVM 17. Use llvm-reduce instead. llvm-reduce applies a set of reduction passes iteratively to find a minimal reproducer. Example: `llvm-reduce --test=./crash.sh input.ll`

For historical reference, `bugpoint` searched for miscompilations by binary-searching over pass orderings and function subsets. Its removal eliminated a significant maintenance burden: `llvm-reduce` achieves the same goal with a simpler interface and better performance.

---

## 173.3 Bisecting Regressions

### git bisect for Compiler Regressions

When a miscompilation was introduced in a specific commit:

```bash
# Start bisection
git bisect start
git bisect bad        # current HEAD is broken
git bisect good v17.0.0  # LLVM 17 was fine

# Git checks out a middle commit; test it
opt -passes=instcombine test.ll > output.ll
diff expected.ll output.ll && git bisect good || git bisect bad

# Or use a script
cat > check.sh << 'EOF'
#!/bin/bash
cd /build && ninja opt 2>/dev/null
./bin/opt -passes=instcombine /test/input.ll -S 2>/dev/null | \
  diff /test/expected.ll - && exit 0
exit 1
EOF
git bisect run ./check.sh
```

`git bisect` typically converges in 15–20 steps (binary search over the ~10,000 commits in a release cycle).

### Pass-Level Bisection

When the commit is known but not the specific optimization:

```bash
# Disable a specific pass to test if it's the culprit
opt -O2 --disable-pass=instcombine input.ll -o output.ll
opt -O2 --disable-pass=licm input.ll -o output.ll

# Note: the flag name depends on LLVM version
# In LLVM 22, use the new pass manager names:
opt -passes='default<O2>' input.ll  # check --help for individual disabling
```

For Clang:
```bash
# Disable individual middle-end passes
clang -O2 -mllvm -disable-instcombine input.c
clang -O2 -mllvm -disable-loop-unrolling input.c

# Use -fno-optimize-sibling-calls etc. for specific frontend opts
clang -O2 -fno-inline input.c
```

### Narrowing to a Function

If the regression is function-specific:

```bash
# Disable optimization for a specific function
# (Add __attribute__((optnone)) to the function in source)
clang -O2 input.c  # with optnone attribute on suspect function

# Or use targeted pass manager:
opt -passes='default<O2>' --debug-pass-manager 2>&1 | grep "Running"
# Shows which passes run; manually apply subset
opt -passes='instcombine,licm' input.ll
opt -passes='instcombine' input.ll  # is instcombine alone sufficient?
```

---

## 173.4 Debugging MLIR

### MLIR Debug Flags

```bash
# Print IR before and after every pass
mlir-opt --mlir-print-ir-before-all --mlir-print-ir-after-all input.mlir

# Print only when IR changes
mlir-opt --mlir-print-ir-after-change input.mlir

# Verify after every pass
mlir-opt --mlir-verify-each input.mlir

# Print before/after specific pass
mlir-opt --mlir-print-ir-before=canonicalize \
          --mlir-print-ir-after=canonicalize input.mlir
```

### Pattern Rewriting Debug

MLIR's pattern rewriting infrastructure has detailed debug output:

```bash
# Trace pattern matching in the greedy rewriter
mlir-opt --debug-only=greedy-rewriter input.mlir 2>&1 | head -100

# Output shows:
# Trying to match 'arith.addi' against pattern 'FoldConstantAdd'...
# Pattern 'FoldConstantAdd' matched and rewrote
# IR after rewrite:
# ...
```

```bash
# Debug dialect conversion
mlir-opt --debug-only=dialect-conversion input.mlir

# Debug a specific transformation
mlir-opt --debug-only=canonicalize input.mlir
```

### mlir-opt with GDB

```bash
# Run mlir-opt under GDB
gdb --args mlir-opt --mlir-verify-each failing_input.mlir

# In GDB:
(gdb) run
# When it crashes:
(gdb) bt        # backtrace
(gdb) frame 3   # navigate to MLIR frame
(gdb) p op->dump()  # dump the current operation
(gdb) call mlir::Operation::print(mlir::Operation*, mlir::raw_ostream&)
```

Useful GDB breakpoints for MLIR debugging:
```gdb
break mlir::emitError            # all diagnostics
break mlir::Operation::verify    # verification failures
break llvm_unreachable           # LLVM unreachable assertions
break __assert_fail              # POSIX assertion failures
```

### dump() in Debug Builds

MLIR operations have `dump()` methods callable from GDB or from debug prints:

```cpp
// In C++ pass code:
op->dump();                        // prints to stderr
op->print(llvm::errs());           // same

// For specific IR elements:
func.dump();
block.dump();
value.dump();

// Print with context
mlir::OpPrintingFlags flags;
flags.enableDebugInfo();
op->print(llvm::errs(), flags);
```

### Verifying MLIR Invariants

MLIR has a comprehensive IR verifier that can catch many bugs:

```bash
# Verify after every pass
mlir-opt --mlir-verify-each input.mlir

# Verify a specific operation in C++ code
if (failed(op->verify())) {
  op->emitError("Verification failed after transformation");
  return failure();
}
```

If `--mlir-verify-each` reveals a failure in a specific pass, the pass has a bug: it produced invalid IR. The verification error message usually pinpoints the exact constraint violated (e.g., "operand type mismatch", "block has no terminator").

---

## 173.5 Profile-Guided Compiler Debugging

### Profiling a Slow Compilation

```bash
# Profile a long compilation with perf
perf record -g --call-graph=dwarf -- clang -O2 large.cpp -o large.o
perf report --call-graph dwarf --no-children

# Alternative: use Google's gperftools
LD_PRELOAD=/usr/lib/libprofiler.so CPUPROFILE=/tmp/prof.out clang -O2 large.cpp
pprof --pdf $(which clang) /tmp/prof.out > profile.pdf
```

### Clang's -ftime-trace

```bash
clang -O2 -ftime-trace large.cpp
# Generates: large.cpp.json (Chrome trace format)

# View in Chrome
open chrome://tracing
# Load large.cpp.json
```

The trace shows:
- Source file parsing time per include.
- Template instantiation time per template.
- LLVM pass time (total and per function).
- Code generation time.

This is the fastest way to identify compilation performance bottlenecks.

### AddressSanitizer for Pass Development

When developing a new pass, build LLVM itself with sanitizers to catch memory errors early:

```bash
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DLLVM_USE_SANITIZER=Address \
  -DLLVM_ENABLE_ASSERTIONS=ON \
  -DLLVM_ENABLE_PROJECTS="clang;mlir" \
  ../llvm

ninja opt clang mlir-opt

# Run the sanitized build
./bin/opt -passes=my-pass input.ll
# ASAN will report heap-use-after-free, heap-buffer-overflow, etc.
```

### UBSan for Pass Development

```bash
cmake ... -DLLVM_USE_SANITIZER=Undefined ...
ninja opt

./bin/opt -passes=my-pass input.ll
# UBSan reports: runtime/ir/instructions.cpp:N: runtime error: signed integer overflow
```

UBSan is particularly useful for finding undefined behavior in LLVM's own code — integer overflow in pass computations, out-of-bounds array indexing, etc.

### Valgrind and Memory Debugging

```bash
# Run under Valgrind (very slow but thorough)
valgrind --tool=memcheck --leak-check=full --track-origins=yes \
  opt -passes=instcombine input.ll -o /dev/null

# Use Massif for memory profiling
valgrind --tool=massif opt -O2 large.ll -o /dev/null
ms_print massif.out.* | head -50
```

---

## MLIR IDE Support: mlir-lsp-server and mlir-query

MLIR ships two tools that bring compiler-development-quality IDE support to `.mlir` files and that make IR exploration interactive: `mlir-lsp-server` implements the Language Server Protocol for MLIR textual IR, and `mlir-query` provides command-line pattern search over IR dumps.

### mlir-lsp-server

`mlir-lsp-server` (`mlir/tools/mlir-lsp-server/`) is a Language Server Protocol (LSP) server for `.mlir` files. It provides:
- **Hover**: hovering over an op shows its ODS documentation, operand types, result types, and attribute constraints
- **Go to definition**: jumping from an op use to its `%result` definition, from a symbol reference to the symbol op
- **Find all references**: all uses of a `func.func @foo` symbol or a `%val` SSA value
- **Code completion**: completing op names, attribute names, and type names based on registered dialects
- **Inlay hints**: showing inferred types on `%val : !unknown` uses
- **Diagnostics**: real-time syntax errors and verifier failures as you type (squiggly underlines in the IDE)
- **Document symbols**: outline view showing all functions, modules, and named symbols

### Architecture

`mlir-lsp-server` is built on LLVM's `mlir/lib/Tools/mlir-lsp-server/` library. Key components:

```
MLIRLspServerMain
├── MLIRServer (handles LSP protocol messages)
│   ├── parseSourceFile() → calls mlir::parseSourceFile()
│   ├── getHover() → walks the parsed IR to find the op at cursor
│   ├── getDefinition() → resolves symbol references and SSA defs
│   └── getCodeActions() → suggests fixes for verifier errors
└── MlirLspServerSetupFn — user-provided dialect registration
```

The server re-parses the `.mlir` file on every edit (incremental parsing is a future goal). This is fast enough for files up to ~10k lines.

### Building and Running

```bash
# Build (included in default LLVM build with LLVM_BUILD_TOOLS=ON)
cmake --build build --target mlir-lsp-server

# Run standalone (not needed if using VS Code extension)
mlir-lsp-server --help
```

The server communicates over stdin/stdout using JSON-RPC (LSP protocol). You never run it directly — the IDE extension manages it.

### VS Code Integration: mlir-vscode

The `mlir-vscode` extension (`mlir/utils/vscode/`) is the official VS Code client:

```bash
# Install from VS Code marketplace
code --install-extension mlir-vscode

# Or install from source (for development)
cd mlir/utils/vscode && npm install && vsce package
code --install-extension mlir-vscode-*.vsix
```

Extension settings (`settings.json`):
```json
{
  "mlir.server.path": "/usr/lib/llvm-22/bin/mlir-lsp-server",
  "mlir.pdll.server.path": "/usr/lib/llvm-22/bin/mlir-pdll-lsp-server",
  "mlir.tablegen.server.path": "/usr/lib/llvm-22/bin/tblgen-lsp-server"
}
```

### Custom Dialect Registration

The default `mlir-lsp-server` binary only knows about in-tree dialects. To get IDE support for your custom dialect, build a custom server:

```cpp
// tools/my-lsp-server/main.cpp
#include "mlir/Tools/mlir-lsp-server/MlirLspServerMain.h"
#include "my-project/Dialect/MyDialect.h"

int main(int argc, char **argv) {
  mlir::DialectRegistry registry;
  // Register all in-tree dialects
  mlir::registerAllDialects(registry);
  // Register your custom dialect
  registry.insert<myproject::MyDialect>();
  // Register any external models needed for verification
  myproject::registerMyDialectExternalModels(registry);

  return mlir::MlirLspServerMain(argc, argv, registry);
}
```

```cmake
# CMakeLists.txt
add_llvm_tool(my-lsp-server main.cpp)
target_link_libraries(my-lsp-server PRIVATE
  MLIRLspServerLib
  MyDialect
  MLIRAllDialects)
```

Point VS Code at the custom binary:
```json
{ "mlir.server.path": "/path/to/build/bin/my-lsp-server" }
```

### mlir-pdll-lsp-server

`mlir-pdll-lsp-server` (`mlir/tools/mlir-pdll-lsp-server/`) provides LSP support specifically for `.pdll` files (PDLL pattern files, cross-ref Ch135). It understands PDLL's `Pattern`, `Constraint`, and `Rewrite` constructs, provides hover for constraint types, and validates PDLL syntax in real time.

### tblgen-lsp-server

`tblgen-lsp-server` (`mlir/tools/tblgen-lsp-server/`) provides LSP support for `.td` TableGen files — the ODS op definitions, trait definitions, and interface definitions. Hover shows the generated C++ interface; go-to-definition navigates between `def Foo` and its `multiclass` instantiation.

### mlir-query: Command-Line IR Pattern Search

`mlir-query` (`mlir/tools/mlir-query/`) provides a `grep`-like interface for querying MLIR IR. It is to MLIR what `clang-query` is to Clang ASTs.

```bash
mlir-query module.mlir -c "m hasOpName(\"linalg.generic\")"
mlir-query module.mlir -c "m isOp<linalg::GenericOp>()"
```

The query language uses the same matcher DSL as `mlir-opt`'s pattern infrastructure:

```bash
# Find all ops that have a certain result type
mlir-query module.mlir \
  -c 'm hasResultType(isF32Type())'

# Find all ops in a specific region that match a pattern
mlir-query lowered.mlir \
  -c 'm allOf(hasOpName("memref.load"), hasParentOp(hasOpName("func.func")))'
```

`mlir-query` is especially useful for debugging lowering pipelines: run `mlir-opt --print-ir-after-all` to get IR at each stage, then use `mlir-query` to locate specific patterns.

### Practical Workflow

A typical MLIR development session using these tools:

1. Write `.mlir` file in VS Code with `mlir-lsp-server` active — get real-time verifier feedback
2. Write `.pdll` patterns with `mlir-pdll-lsp-server` — hover shows constraint types
3. Write ODS definitions in `.td` with `tblgen-lsp-server` — navigate interface inheritance
4. Use `mlir-query` on IR dumps to find unexpected ops after lowering
5. Use `mlir-opt --mlir-print-op-on-diagnostic` to print the failing op on verification error

Reference LLVM 22 source links:
- [`MlirLspServerMain.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Tools/mlir-lsp-server/MlirLspServerMain.h)
- [`mlir-lsp-server/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/tools/mlir-lsp-server/)
- [`mlir-query/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/tools/mlir-query/)

---

## llubi: UB-Aware LLVM IR Interpreter

Alive2 provides static SMT-based verification of LLVM IR transformations ([Chapter 174](../part-24-verified-compilation/ch174-alive2.md)); KLEE performs symbolic execution over LLVM IR to find reachable error states. `llubi` occupies a distinct niche: concrete interpretation of LLVM IR with per-value undefined-behavior tracking. Where Alive2 requires Z3 to discharge refinement queries (and can time out on complex transformations), llubi executes IR on actual inputs and reports UB as it arises, with precise per-instruction provenance.

### Design and Shadow State

`llubi` (the LLVM UB-aware IR Interpreter) executes LLVM IR instruction-by-instruction, maintaining a parallel **UB-tracking shadow** alongside each SSA value. Each shadow records:

- Whether the value carries **poison** (propagated through arithmetic, comparison, GEPs that use poison operands)
- Whether the value is **uninitialized** (derived from memory loaded before a `llvm.lifetime.start` marker or from an alloca that was never written)
- Whether any **out-of-bounds GEP** was performed to produce a pointer

The shadow is computed by the interpreter alongside the concrete value; a shadow bit set to "poisoned" or "UB" does not immediately abort execution — it is tracked and reported only when that value reaches a position where UB is observable (e.g., passed to a branch condition, stored to memory and reloaded, or used as a divisor).

### Tracked UB Categories

`llubi` covers the following IR-level UB classes:

| UB Category | Triggering Condition |
|-------------|---------------------|
| **GEP out-of-bounds** | `getelementptr` computes a pointer outside the bounds of the allocated object; detected by comparing the concrete pointer against the allocation range tracked per `alloca` / `malloc` call |
| **Invalid `inttoptr`** | An integer value is cast to a pointer using `inttoptr` where the integer is not a previously valid pointer value; tracked by maintaining a set of live pointer provenance identities |
| **Uninitialized memory** | Memory is loaded from a range that overlaps with no prior store and is bracketed by `llvm.lifetime.start` / `llvm.lifetime.end` markers; llubi tracks lifetime regions to flag reads of dead allocations |
| **Poison propagation** | Poison values (from `add nsw` on overflow, from `udiv` by zero before the division is reached, etc.) propagate through operands; llubi tracks the poison shadow through each instruction and reports when poison becomes observable |

### Use Cases

**Debugging UB introduced by passes**: after running a suspect optimization pass, feed the input and output IR to llubi with the same concrete inputs. If the output IR triggers UB that the input IR did not, the pass introduced the UB.

**Validating transformations on concrete inputs**: for cases where Alive2's Z3 backend times out (large loop bodies, floating-point heavy code), llubi provides a lightweight concrete check over a representative test corpus.

**Complement to Alive2**: static and dynamic validation are complementary. Alive2 proves correctness for all inputs but can time out; llubi checks specific inputs quickly and catches UB that static analysis misses due to imprecision in alias reasoning.

### Comparison with UBSan

UBSan instruments C/C++ source code at the Clang front-end level, before optimization. It catches source-level UB: signed integer overflow, null-pointer dereference, array out-of-bounds with known sizes, misaligned loads. Crucially, UBSan runs *before* the optimizer, so it cannot detect UB introduced *by* optimization passes.

`llubi` works at the LLVM IR level post-optimization. It catches:
- IR-level UB introduced by passes (InstCombine, GVN, LICM, vectorization)
- Poison propagation through `nsw`/`nuw`/`exact` flags added by passes without proof
- GEP out-of-bounds arising from strength-reduction or loop transformations that incorrectly infer bounds

### Status in LLVM 22

`llubi` is experimental in LLVM 22, shipped as the `llvm-interp` tool (or as `llubi` in some build configurations). It is not enabled in the default LLVM distribution build; it must be built explicitly:

```bash
cmake -G Ninja -DLLVM_ENABLE_PROJECTS="llvm" \
      -DLLVM_BUILD_TOOLS=ON \
      -DLLVM_TOOL_LLUBI_BUILD=ON \
      ../llvm
ninja llvm-interp
```

### Running llubi

```bash
# Run a .ll file through llubi; reports UB on first triggering instruction
llvm-interp input.ll

# Specify entry point and concrete arguments
llvm-interp --entry-function=main --arg=42 input.ll

# Example UB report output:
# UB detected in function 'compute' at instruction:
#   %6 = getelementptr inbounds i32, ptr %arr, i64 %idx
# GEP out-of-bounds: base=0x7ffd3a000000, size=40 bytes, computed offset=48 bytes
# Value %idx = 12, base allocation size = 10 elements
# Shadow trace:
#   %idx originates from: %3 = load i64, ptr %n_ptr   (possibly untrusted input)
```

The UB report identifies the exact instruction, the concrete pointer value, the allocation bounds, and the call chain leading to the UB. When combined with `--print-ir-after-all` dumps from `opt`, this pinpoints the pass that introduced the errant GEP.

---

## Chapter Summary

- **`--print-before-all`/`--print-after-all`/`--print-changed`**: dump IR at every pass boundary; `--print-changed` is the most useful for narrowing down which pass introduced a bug.
- **`LLVM_DEBUG` + `--debug-only=TAG`**: conditional, tag-filtered debug output from pass internals; use `dbgs()` for pass-specific diagnostic output.
- **`--stats`**: print counts of optimizations performed; useful for confirming passes are running and doing work.
- **`llvm-reduce`**: automatically minimizes a crashing LLVM IR input using delta debugging; faster than `bugpoint` and directly driven by a "is this interesting?" script.
- **Clang `--gen-reproducer`**: generates a tarball with preprocessed source and command line for reliable crash reproduction.
- **`git bisect run`**: automated regression bisection over LLVM commit history; converges in ~15 steps.
- **MLIR debugging**: `--mlir-print-ir-before-all`, `--mlir-verify-each`, `--debug-only=greedy-rewriter`; `op->dump()` callable from GDB.
- **GDB breakpoints**: `mlir::emitError`, `mlir::Operation::verify`, `llvm_unreachable` for interactive MLIR debugging.
- **`-ftime-trace`**: Chrome trace output showing per-include and per-pass compilation times; fastest way to find compilation performance bottlenecks.
- Building LLVM with `LLVM_USE_SANITIZER=Address` or `Undefined` catches memory errors and UB in pass development; essential for any non-trivial pass.


---

@copyright jreuben11
