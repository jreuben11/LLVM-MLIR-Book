# Chapter 174 — Performance Engineering

*Part XXV — Operations, Bindings, and Contribution*

Performance engineering for a compiler has two distinct targets: the compiler's own throughput (how fast it compiles programs) and the quality of code it generates (how fast the compiled programs run). Both matter enormously in production — a compiler that takes 10 minutes to build a large project impedes developer velocity; a compiler that generates code running at 80% of peak hardware performance leaves significant capability on the table. This chapter covers the tools and techniques for measuring, understanding, and improving both dimensions: profiling compilation time with `-ftime-trace`, analyzing generated code quality with `perf` and hardware counters, tuning the LLVM build system for iteration speed, and engineering the optimization pipeline for both compile-time efficiency and runtime code quality.

---

## Table of Contents

- [174.1 Measuring Compiler Performance](#1741-measuring-compiler-performance)
  - [Wall-Clock Benchmarking](#wall-clock-benchmarking)
  - [Hardware Performance Counters with perf stat](#hardware-performance-counters-with-perf-stat)
  - [LLVM's -ftime-report](#llvms-ftime-report)
- [174.2 Compile-Time Profiling](#1742-compile-time-profiling)
  - [-ftime-trace (Chrome Trace)](#ftime-trace-chrome-trace)
  - [Flamegraph from perf](#flamegraph-from-perf)
  - [Analyzing Specific Pass Hot Paths](#analyzing-specific-pass-hot-paths)
- [174.3 LLVM Build Performance](#1743-llvm-build-performance)
  - [Reducing Build Time for Development](#reducing-build-time-for-development)
  - [ccache and sccache](#ccache-and-sccache)
  - [Precompiled Headers (PCH)](#precompiled-headers-pch)
  - [Build Configuration Presets](#build-configuration-presets)
  - [Avoiding Full Rebuilds](#avoiding-full-rebuilds)
- [174.4 IR Optimization Performance](#1744-ir-optimization-performance)
  - [Benchmarking the Optimization Pipeline](#benchmarking-the-optimization-pipeline)
  - [Identifying Quadratic Behavior](#identifying-quadratic-behavior)
  - [Analysis Caching](#analysis-caching)
  - [Memory Usage of Passes](#memory-usage-of-passes)
- [174.5 Runtime Performance of Generated Code](#1745-runtime-performance-of-generated-code)
  - [Benchmarking Generated Code](#benchmarking-generated-code)
  - [Analyzing Generated Code Quality](#analyzing-generated-code-quality)
  - [perf for Generated Code](#perf-for-generated-code)
  - [Profile-Guided Optimization (PGO)](#profile-guided-optimization-pgo)
  - [BOLT: Post-Link Optimization](#bolt-post-link-optimization)
  - [Benchmark Infrastructure](#benchmark-infrastructure)
- [Chapter Summary](#chapter-summary)

---

## 174.1 Measuring Compiler Performance

### Wall-Clock Benchmarking

The most basic measurement is wall-clock time:

```bash
# Simple timing
time clang -O2 large.cpp -o large.o

# More precise with multiple runs (GNU time)
/usr/bin/time -v clang -O2 large.cpp -o large.o
# Output includes:
#   Wall clock (elapsed): 3.24 seconds
#   Maximum resident set size (kbytes): 1,234,567
#   Major (requiring I/O) page faults: 0
#   Minor (reclaiming a frame) page faults: 24,576

# Repeat 10 times and report statistics
hyperfine 'clang -O2 large.cpp -o large.o' --warmup=2
```

Wall-clock time is noisy — it includes disk I/O, OS scheduling, CPU frequency scaling, and many other system effects. Use `perf stat` for more reliable measurements.

### Hardware Performance Counters with perf stat

```bash
# Count hardware events
perf stat clang -O2 large.cpp -o /dev/null

# Output:
#  Performance counter stats for 'clang -O2 large.cpp':
#     3,245,678,901  instructions          #    4.23  insn per cycle
#       767,891,234  cycles
#        23,456,789  cache-misses          #    1.23% of all cache refs
#     1,901,234,567  cache-references
#           123,456  branch-misses         #    0.45% of all branches
#        27,345,678  branches
#        1.234 seconds time elapsed

# Detailed event list
perf stat -e instructions,cycles,L1-dcache-load-misses,L1-dcache-loads,\
  LLC-load-misses,LLC-loads,branch-misses,branches \
  clang -O2 large.cpp -o /dev/null
```

Key metrics:
- **IPC (instructions per cycle)**: `instructions/cycles`. Modern CPUs achieve 3–5 IPC when executing well. Low IPC (< 1) indicates stalls, often from cache misses or branch mispredictions.
- **LLC (Last Level Cache) miss rate**: cache misses per load. High LLC miss rate (> 5%) indicates the working set exceeds cache size; for compilers, this often means the IR data structures are too large.
- **Branch miss rate**: mispredictions per branch. High branch miss rate (> 3%) indicates unpredictable branches; for compilers, these often occur in visitor pattern dispatches on IR nodes.

### LLVM's -ftime-report

```bash
# LLVM's built-in per-pass timing
opt -O2 -ftime-report input.ll -o /dev/null

# Via clang
clang -O2 -mllvm -time-passes large.cpp -o /dev/null

# Output:
# Pass execution timing report
# ===========================
# Total Execution Time: 0.4523 seconds
#
#    ---User Time---   --System Time--   --User+System--   ---Wall Time---   Name
#    0.0312 (  6.9%)   0.0000 (  0.0%)   0.0312 (  6.9%)   0.0313 (  6.9%)  InstCombinePass
#    0.0287 (  6.4%)   0.0000 (  0.0%)   0.0287 (  6.4%)   0.0287 (  6.4%)  GVNPass
#    0.0213 (  4.7%)   0.0000 (  0.0%)   0.0213 (  4.7%)   0.0213 (  4.7%)  LoopVectorizePass
```

This breakdown shows which passes consume the most time — the starting point for compile-time optimization.

---

## 174.2 Compile-Time Profiling

### -ftime-trace (Chrome Trace)

Clang's `-ftime-trace` generates a Chrome Tracing format JSON file:

```bash
clang -O2 -ftime-trace -ftime-trace-granularity=100 large.cpp -o large.o
# Creates: large.cpp.json (or specify with -ftime-trace-file=trace.json)

# View in browser
chrome chrome://tracing  # Drag and drop large.cpp.json
# Or use Perfetto (better for large traces):
# https://ui.perfetto.dev  # Upload trace.json
```

The trace shows a timeline with nested spans:
- **ParseFile**: time to preprocess and parse the source.
- **InstantiateTemplate**: per-template instantiation times (critical for heavy template use).
- **OptModule**: total LLVM optimization time.
- **OptFunction**: per-function optimization time (largest functions appear as widest spans).
- **Backend**: instruction selection and register allocation.
- **CodeGen**: final machine code emission.

Example: a function that appears as a wide `OptFunction` span is a candidate for manual code simplification or `__attribute__((optnone))` to reduce compile time.

### Flamegraph from perf

```bash
# Record with call graphs
perf record -g --call-graph=dwarf -F 99 -- clang -O2 large.cpp -o /dev/null

# Generate flamegraph
perf script | /usr/share/flamegraph/stackcollapse-perf.pl | \
  /usr/share/flamegraph/flamegraph.pl > flamegraph.svg
# Open flamegraph.svg in a browser

# Alternatively: use Brendan Gregg's FlameGraph tools
git clone https://github.com/brendangregg/FlameGraph
perf script | ./FlameGraph/stackcollapse-perf.pl | \
  ./FlameGraph/flamegraph.pl > flamegraph.svg
```

A flamegraph shows which C++ functions consume the most time (width = CPU time). The widest boxes at the top of the flame are the hot leaf functions; the boxes below are their callers. For LLVM, typical hot functions are IR iteration in passes, alias analysis queries, and dominator tree computation.

### Analyzing Specific Pass Hot Paths

```bash
# Focus perf on a specific pass by using opt with only that pass
perf record -g -- opt -passes=instcombine large.ll -o /dev/null
perf report --call-graph dwarf

# Use perf annotate to see hot lines
perf annotate InstCombiner::run
```

---

## 174.3 LLVM Build Performance

### Reducing Build Time for Development

LLVM builds are large. Strategies for faster iteration:

```bash
# Use Ninja instead of Make (parallel, incremental)
cmake -G Ninja ...

# Parallel compilation
ninja -j$(nproc) opt clang mlir-opt

# Use a faster linker
cmake ... -DLLVM_USE_LINKER=lld
# LLD is 2-5x faster than GNU ld for LLVM-scale projects

# Split DWARF (reduces object file size, speeds up linking)
cmake ... -DLLVM_USE_SPLIT_DWARF=ON

# Only build the targets you need
cmake ... -DLLVM_TARGETS_TO_BUILD="X86;AArch64" \
           -DLLVM_ENABLE_PROJECTS="clang;mlir"

# Skip runtime builds during development
cmake ... -DLLVM_BUILD_RUNTIME=OFF
```

### ccache and sccache

```bash
# ccache: local compiler output cache
apt-get install ccache
cmake ... -DLLVM_CCACHE_BUILD=ON
# Or manually:
export CC="ccache clang"
export CXX="ccache clang++"
cmake ...

# sccache: distributed cache (useful in CI)
cargo install sccache
cmake ... -DCMAKE_C_COMPILER_LAUNCHER=sccache \
           -DCMAKE_CXX_COMPILER_LAUNCHER=sccache
```

After a clean build, `ccache` achieves hit rates > 90% for incremental builds that touch a few files. This makes re-running CI after a small change dramatically faster.

### Precompiled Headers (PCH)

LLVM's headers are large. PCH reduces repeated parsing overhead:

```bash
cmake ... -DLLVM_BUILD_TOOLS_PCH=ON
```

The `LLVM_BUILD_TOOLS_PCH` option builds a PCH for LLVM's main headers; subsequent compilations that include those headers skip reparsing them.

### Build Configuration Presets

LLVM 22 ships CMake presets for common development configurations:

```bash
# List available presets
cmake --list-presets

# Use a preset
cmake --preset=debug
cmake --preset=release-developer  # O2, assertions enabled, fast link
cmake --preset=sanitize-address
```

The presets encode recommended combinations of flags for common use cases, avoiding common mistakes like building with RTTI accidentally enabled or using the wrong sanitizer flags.

### Avoiding Full Rebuilds

The biggest compile-time win is incremental builds. Key practices:

```bash
# Add a new file: touch it to avoid rebuilding everything
touch llvm/lib/Transforms/Scalar/MyNewPass.cpp
ninja MyNewPass.o

# After changing a header that many files include: expect a long rebuild
# Consider using "module partitions" or PIMPL to reduce header dependencies

# Check which files would be rebuilt after a change (without actually building)
ninja -n opt 2>&1 | wc -l  # count files that would be recompiled
```

---

## 174.4 IR Optimization Performance

### Benchmarking the Optimization Pipeline

```bash
# Time the optimization pipeline with statistics
opt -O3 large.ll --stats --disable-output

# Benchmark with multiple passes
time opt -passes='instcombine,licm,gvn,loop-vectorize' large.ll -o /dev/null

# Compare two pass orderings
time opt -passes='instcombine,gvn,licm' large.ll -o /dev/null
time opt -passes='gvn,instcombine,licm' large.ll -o /dev/null
```

### Identifying Quadratic Behavior

Many optimization bugs manifest as quadratic compile time on specially crafted inputs. Common causes:

- **Nested iteration**: an outer loop over instructions triggers an inner pass that also iterates over instructions; combined: O(n²).
- **Use-list iteration**: modifying the use list of a value while iterating over users; the iterator may revisit entries.
- **Alias analysis queries**: O(n²) queries for n memory accesses if every pair is checked.

To identify:
```bash
# Generate inputs of increasing size
for n in 100 200 400 800 1600; do
  llvm-stress -size=$n -seed=42 | \
    time opt -passes=my-pass -disable-output
done
# If time grows as n², you have a quadratic algorithm
```

### Analysis Caching

LLVM's pass manager caches analysis results. Passes that invalidate unnecessary analyses cause recomputation overhead:

```cpp
// Bad: invalidating everything (causes all analyses to recompute)
return PreservedAnalyses::none();

// Good: declare what is preserved
PreservedAnalyses PA;
PA.preserve<DominatorTreeAnalysis>();
PA.preserve<LoopAnalysis>();
// Only AliasAnalysis is invalidated (because we changed memory)
return PA;
```

Always declare the minimal set of invalidations. Use `PreservedAnalyses::all()` when the pass is truly read-only.

### Memory Usage of Passes

```bash
# Measure peak memory usage
/usr/bin/time -v opt -O2 large.ll -o /dev/null 2>&1 | grep "Maximum resident"

# Use Massif for heap profile
valgrind --tool=massif --pages-as-heap=yes opt -O2 large.ll -o /dev/null
ms_print massif.out.* | head -100
```

For passes that process large functions, common memory consumers:
- **DenseMap/DenseSet** with many entries: each DenseMap is sized to 75% load factor; consider `SmallDenseMap` for small cases.
- **The use-list**: LLVM values maintain a doubly-linked list of all uses; for values with millions of uses (e.g., a `undef` used everywhere), this is expensive.
- **The worklist**: greedy rewriters maintain a worklist of pending operations; on large inputs, the worklist can grow to millions of entries.

---

## 174.5 Runtime Performance of Generated Code

### Benchmarking Generated Code

```bash
# Use Google Benchmark for microbenchmarks
# (Add to your project's CMakeLists.txt)
find_package(benchmark REQUIRED)
target_link_libraries(my_bench benchmark::benchmark)

# Run with statistical reporting
./my_bench --benchmark_repetitions=10 --benchmark_report_aggregates_only=true

# Or use SPEC CPU 2017 for comprehensive evaluation
# (Requires license; widely used in LLVM performance tracking)
```

### Analyzing Generated Code Quality

```bash
# View assembly output
clang -O2 -S -masm=intel example.c -o example.s
cat example.s

# View annotated assembly with source interleaving
clang -O2 -S -masm=intel -g example.c
objdump --source example.o | less

# Analyze with LLVM-MCA (machine code analyzer)
llvm-mca -mcpu=native example.s
# Output: instruction throughput, resource usage, port utilization
```

### perf for Generated Code

```bash
# Profile the running program
perf stat ./my_benchmark
# Key metrics: IPC, cache miss rates, branch mispredictions

# Find hot functions
perf record -g ./my_benchmark
perf report

# Identify cache behavior
perf stat -e cache-references,cache-misses,L1-dcache-load-misses ./my_benchmark
```

### Profile-Guided Optimization (PGO)

PGO uses runtime profiling data to guide compiler decisions:

```bash
# Step 1: Compile with profiling instrumentation
clang -O2 -fprofile-instr-generate=pgo.profraw program.c -o program.prof

# Step 2: Run on representative workloads
./program.prof < representative_input1.txt
./program.prof < representative_input2.txt

# Step 3: Merge profiles
llvm-profdata merge -output=pgo.profdata pgo*.profraw

# Step 4: Compile with profile data
clang -O2 -fprofile-instr-use=pgo.profdata program.c -o program.opt

# Verification: the optimized binary should run significantly faster
./program.opt < input.txt   # 10-30% speedup typical
```

PGO guides: inline decisions (inline hot callees), branch prediction hints (likely/unlikely), code layout (hot code in the same cache lines), register allocation (spill cold values first).

### BOLT: Post-Link Optimization

BOLT (Binary Optimization and Layout Tool, Chapter 118) applies PGO after linking, reordering functions and basic blocks for cache efficiency:

```bash
# Instrument for BOLT
clang -O2 -Wl,--emit-relocs program.c -o program.bolt.instrumented
llvm-bolt program.bolt.instrumented -instrument -o program.instrumented \
  -instrumentation-file=/tmp/bolt.fdata

# Run instrumented binary
./program.instrumented < representative_input.txt

# Optimize with BOLT
llvm-bolt program.bolt.instrumented -o program.bolt \
  -data=/tmp/bolt.fdata \
  -reorder-blocks=ext-tsp \
  -reorder-functions=hfsort \
  -split-functions \
  -split-eh \
  -dyno-stats
```

BOLT typically achieves 5–15% additional speedup on top of PGO for CPU-bound programs with significant branch mispredictions from code layout.

### Benchmark Infrastructure

```bash
# Run the LLVM test suite (requires external benchmarks setup)
# See: https://github.com/llvm/llvm-test-suite

cmake -G Ninja \
  -DTEST_SUITE_LLVM_SIZE=small \
  -DTEST_SUITE_BENCHMARKING_ONLY=ON \
  /path/to/llvm-test-suite
ninja

# Compare two compiler versions
llvm-lit --param cc=/path/to/clang-new \
         --param cxx=/path/to/clang++-new . > new_results.txt
llvm-lit --param cc=/path/to/clang-old \
         --param cxx=/path/to/clang++-old . > old_results.txt
# Diff the timing reports
```

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **`-ftime-trace` expansion to MLIR pipelines**: Active work on extending Clang/LLVM's Chrome-trace profiling to cover MLIR pass pipelines end-to-end, surfacing per-dialect pass timing in the same JSON artifact; tracked in [D149456](https://reviews.llvm.org/D149456) follow-ons and discussed on LLVM Discourse under "MLIR Performance Tooling".
- **LLVM CMake preset stabilization**: The `CMakePresets.json` infrastructure introduced in LLVM 18 is expected to reach a stable, documented set of developer presets by late 2026, replacing the ad-hoc `-DLLVM_CCACHE_BUILD`/`-DLLVM_USE_LINKER` flag combinations that developers currently memorize manually.
- **`llvm-mca` throughput model updates for Zen 5 and Lion Cove**: AMD Zen 5 and Intel Lion Cove µarch scheduling models were added in LLVM 19–20; remaining latency/throughput gaps (especially for AVX-512 gather/scatter and AMX) are being closed by community contributors tracking vendor disclosure cycles.
- **BOLT integration into the LLVM test-suite performance tracking CI**: Meta and Google are driving BOLT into the upstream `llvm-test-suite` continuous benchmarking pipeline so per-commit BOLT regressions can be caught automatically, complementing existing PGO CI jobs.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Parallel pass manager with finer-grained analysis sharing**: The new pass manager's analysis cache is not thread-safe, blocking module-level parallelism within a single TU. Active RFC threads on discourse.llvm.org ("Parallel NewPM") propose a read-lock-for-analyses / write-lock-for-transforms scheme allowing independent SCC components to be optimized concurrently, potentially halving `-O2` compile times on large modules with many independent translation units merged by LTO.
- **Profile-guided build-system caching via Bazel/Buck2 remote cache integration with PGO profiles**: Efforts to co-locate PGO profile artifacts in the same content-addressable storage used by Bazel remote-execution so incremental PGO rebuilds require transmitting only changed profile slices, rather than full `.profdata` files. This is under discussion in the LLVM build-system working group.
- **`perf` event-based thinLTO feedback loop**: Google's AutoFDO2 project extends sampling-based PGO to ThinLTO by propagating LBR (Last Branch Record) profiles across module boundaries at link time. Upstream integration is planned once the cross-module profile consistency guarantees are formalized.
- **Memory-layout–aware register allocation using hardware PMU feedback**: Research prototypes (see "PMU-Guided RA", CGO 2025) feed L1/LLC miss profiles back into the register allocator's spill cost model, reducing stack-spill traffic for hot functions. Integration with LLVM's `RegAllocGreedy` is a medium-term goal of the LLVM performance working group.

### 5-Year Horizon (Long-Term, by ~2031)

- **Compiler-as-a-service (CaaS) persistent compilation daemons with fine-grained IR caching**: Rather than re-parsing and re-optimizing translation units from scratch, a long-running compiler daemon maintains an IR cache keyed by preprocessed source hash, invalidating at the granularity of top-level declarations. This would make incremental compile times approach zero for single-function edits even in large monorepos, analogous to `clangd`'s preamble PCH but for the optimization pipeline.
- **Machine-learning–driven pass ordering replacing heuristic pipelines**: Building on the `MLGOTraining` and `MLGO` projects already upstreamed for inliner and register allocator, a full learned pass pipeline (predicting optimal `-O3`-equivalent pass sequences per function from IR features) would replace the fixed `PassBuilder` pipeline, particularly for embedded and ML-inference targets where the default x86-tuned heuristics misfire badly.
- **Unified compile-time and runtime profiling representation**: Long-term convergence of `-ftime-trace` (compile-time) and LLVM's SampleProfile/InstrProfile (runtime) into a single hierarchical profiling artifact that maps hot runtime call chains back to the compiler decisions (inlining, vectorization, unrolling) responsible for them — enabling automated feedback from production deployments directly into the optimization pipeline without manual engineering.

---

## Chapter Summary

- **`perf stat`** with `-e instructions,cycles,cache-misses,branch-misses` gives reliable compile-time and runtime measurements; wall-clock time is noisy — prefer instruction counts for algorithmic comparisons.
- **`-ftime-report`** / **`-mllvm -time-passes`** breaks down compilation time by pass; identifies compile-time hot passes.
- **`-ftime-trace`** generates Chrome Tracing format output; shows per-template, per-function, and per-pass timing; viewable in Perfetto or `chrome://tracing`.
- **Build speed**: LLD (`-DLLVM_USE_LINKER=lld`) is 2–5× faster than GNU ld; ccache eliminates redundant compilation; Ninja's incremental dependency tracking avoids unnecessary rebuilds.
- **`LLVM_TARGETS_TO_BUILD`** and **`LLVM_ENABLE_PROJECTS`** reduce build scope dramatically for development; do not build all targets when only x86 is needed.
- **Quadratic pass complexity** is a bug, not a design choice: identify via input-size scaling experiments; fix by restructuring inner loops or using amortized data structures.
- **`PreservedAnalyses::none()`** is the correct return only when a pass changes everything; incorrect overuse causes expensive analysis recomputation.
- **PGO** (profile-guided optimization) gives 10–30% runtime speedup on benchmarks representative of real workloads; combine with BOLT for an additional 5–15%.
- **`llvm-mca`** analyzes assembly for instruction-level throughput, resource usage, and port pressure; identifies machine-code-level bottlenecks invisible at the source level.


---

@copyright jreuben11
