# Chapter 173 — Debugging the Compiler

*Part XXV — Operations, Bindings, and Contribution*

Debugging a compiler is fundamentally different from debugging application code. The program you are debugging is itself a program transformer: the symptoms appear in the output (wrong assembly, wrong IR, a crash in a later pass), and the root cause may be in a pass that ran dozens of transformations earlier. Localizing a compiler bug requires working through two levels: first finding which pass or transformation introduced the error, then understanding why it did so. LLVM and MLIR provide a rich set of tools for both levels — diagnostic flags that dump IR at every stage, debug-only output controlled at fine granularity, bisection tools that isolate the failing pass, and standard interactive debugger techniques adapted for the compiler's deeply recursive structure. This chapter covers the full debugging toolkit, from the first sign of trouble to a precise, actionable bug report.

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

`bugpoint` is an older automated test case reducer that predates `llvm-reduce`:

```bash
# Find which pass causes a crash
bugpoint -compile-error -opt-args -O2 crashing.ll

# Find a miscompilation (behavioral difference between -O0 and -O2)
bugpoint -miscompilation crashing.ll
```

`bugpoint` is slower than `llvm-reduce` and harder to configure; for new work, prefer `llvm-reduce`.

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
