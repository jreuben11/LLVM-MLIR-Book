# Chapter 67 — Profile-Guided Optimization

*Part X — Analysis and the Middle-End*

Profile-guided optimization (PGO) uses dynamic profile data — counts of how often each code path executes at runtime — to make better optimization decisions. Inlining threshold increases for hot call sites, loop unrolling for high-trip-count loops, and function layout for better icache behavior all depend on knowing *which code is actually hot*. This chapter covers LLVM's instrumentation-based PGO, SampleProfile/AutoFDO, pseudo-probes, context-sensitive PGO (CSSPGO), the `llvm-profdata` tool, and PGO integration with ThinLTO and BOLT.

## Table of Contents

- [67.1 Instrumentation-Based PGO](#671-instrumentation-based-pgo)
  - [67.1.1 Two-Stage Build](#6711-two-stage-build)
  - [67.1.2 Instrumentation Internals](#6712-instrumentation-internals)
  - [67.1.3 Profile Data Consumption](#6713-profile-data-consumption)
  - [67.1.4 Value Profiling](#6714-value-profiling)
- [67.2 SampleProfile and AutoFDO](#672-sampleprofile-and-autofdo)
- [67.3 Pseudo-Probes](#673-pseudo-probes)
- [67.4 Context-Sensitive PGO (CSSPGO)](#674-context-sensitive-pgo-csspgo)
- [67.5 `llvm-profdata` Tool](#675-llvm-profdata-tool)
- [67.6 ThinLTO + PGO](#676-thinlto-pgo)
- [67.7 PGO + BOLT](#677-pgo-bolt)
- [Chapter Summary](#chapter-summary)

---

## 67.1 Instrumentation-Based PGO

### 67.1.1 Two-Stage Build

Classic PGO has two stages:

```
Stage 1: Instrumented build
  clang -O2 -fprofile-generate program.cpp -o program_instr

Stage 2: Profile collection
  ./program_instr typical_workload
  # produces default.profraw

Stage 3: Profile merging
  llvm-profdata merge -output=merged.profdata default.profraw

Stage 4: Optimized build
  clang -O2 -fprofile-use=merged.profdata program.cpp -o program_opt
```

The instrumented binary writes raw profile data to `default.profraw` (or the file named by `LLVM_PROFILE_FILE`). Multiple runs can be merged:

```bash
llvm-profdata merge -output=merged.profdata run1.profraw run2.profraw run3.profraw
```

### 67.1.2 Instrumentation Internals

`-fprofile-generate` inserts counters at every function entry and every conditional branch taken. The instrumentation is implemented in `llvm/Transforms/Instrumentation/InstrProfiling.h` and `PGOInstrumentation.h`.

Two counter schemes:
- **Edge profiling** (default): counts each control-flow edge. The profdata contains edge-to-count mappings, from which block frequencies are inferred.
- **Block profiling**: counts each basic block independently. More counters, but no inference needed.

The raw counters are stored in a special ELF section (`__llvm_prf_cnts`, `__llvm_prf_data`, `__llvm_prf_names`) and written at program exit.

### 67.1.3 Profile Data Consumption

With `-fprofile-use`, Clang reads the `.profdata` file and attaches `!prof` branch weight metadata:

```llvm
; Before PGO:
br i1 %cond, label %hot_path, label %cold_path

; After PGO: branch weight metadata
br i1 %cond, label %hot_path, label %cold_path,
   !prof !0
!0 = !{!"branch_weights", i32 9990, i32 10}
; 99.9% of the time the hot_path is taken
```

This metadata feeds into `BranchProbabilityAnalysis` → `BlockFrequencyAnalysis` → all passes that query block frequencies.

### 67.1.4 Value Profiling

Beyond branch counts, value profiling records the most frequent values of certain operations:
- **Indirect call targets**: which function pointer is most frequently called.
- **Memory intrinsic sizes**: the most common argument to `memcpy`, `memset`.
- **Integer values**: for switch target prediction.

Value profiles are stored as `!vpid` metadata on `CallInst` and `MemIntrinsicInst`:

```llvm
call void @llvm.memcpy(ptr %dst, ptr %src, i64 %size, i1 false),
    !vpid !1
!1 = !{i64 1234, i64 10, i64 256, i64 8, ...}
; Most common size is 256 (10 times), next is 8 times, ...
```

The `MemoryProfileInfo` pass uses value profiles to specialize `memcpy` calls for the most common size.

## 67.2 SampleProfile and AutoFDO

Instrumentation requires recompiling and rerunning the program, which is expensive for large production services. **AutoFDO** (Automatic Feedback-Directed Optimization) collects profiles *from production without recompiling*:

1. Run the production binary (non-instrumented).
2. Collect hardware performance counter samples (Linux `perf record`).
3. Convert samples to LLVM's sample profile format with `create_llvm_prof` or Google's `perf_data_converter`.
4. Compile with `-fprofile-sample-use=profile.prof`.

```bash
perf record -b -e cycles:ppp ./program workload
perf script -i perf.data > perf.script
create_llvm_prof --binary=./program --profile=perf.script --out=profile.prof
clang -O2 -fprofile-sample-use=profile.prof program.cpp -o program_opt
```

The sample profile is stored in LLVM's `SampleProf` format (ASCII or binary, declared in `llvm/ProfileData/SampleProf.h`):

```
function_name:total_count:self_count
 line_offset: count [callee:count]*
```

The `SampleProfileLoaderPass` in `llvm/Transforms/IPO/SampleProfile.h` reads the profile, annotates branch weights and call hotness, and triggers inline decisions based on sample hotness.

## 67.3 Pseudo-Probes

Instrumentation-based PGO can become inaccurate after optimization: inlining and code motion move the instrumented counters relative to the source code, making profile matching imprecise. **Pseudo-probes** solve this by inserting stable, non-intrusive markers at key points in the IR before optimization:

```llvm
; A pseudo-probe intrinsic — zero cost, just a stable marker
call void @llvm.pseudoprobe(i64 12345, i64 1, i32 0, i64 -1)
; args: function GUID, probe ID, type, discriminator
```

Pseudo-probes persist through optimization (they are not removed by DCE or moved by LICM) and survive inlining (they record the call-site context). The sample profiler matches profiles to pseudo-probes, not to source line numbers, providing more stable profile-to-IR mapping.

```bash
clang -O2 -fprofile-sample-accurate -fpseudo-probe-for-profiling \
    -fprofile-sample-use=profile.prof program.cpp
```

Pseudo-probes are implemented in `llvm/Transforms/IPO/SampleProfile.h` and the `PseudoProbeInserterPass`.

## 67.4 Context-Sensitive PGO (CSSPGO)

Standard PGO uses a single count per call site, regardless of the calling context. A function called from a hot path and a cold path shares a single profile, diluting the hot-path signal.

CSSPGO (Context-Sensitive Sample Profile Guided Optimization) maintains a **context-sensitive profile**: the profile is indexed by the full call stack (function chain), not just the function + line. Each inlining context gets its own count:

```
outer_function
  call inner_function [hotness: 5000]
    inner_function:inner_block [count: 4800]

other_function
  call inner_function [hotness: 10]
    inner_function:inner_block [count: 5]
```

With CSSPGO, `inner_function` is inlined aggressively into `outer_function` (where it's hot) but not into `other_function` (where it's cold).

```bash
clang -O2 -fprofile-sample-use=csspgo.prof \
    -fcs-profile-generate program.cpp      # Stage 1: context-sensitive instrumentation
# ... collect profile ...
llvm-profdata merge -output=merged.profdata csspgo.profraw
clang -O2 -fprofile-sample-use=merged.profdata program.cpp  # Stage 2: optimized build
```

CSSPGO uses the `SampleProfileLoader` with the CSSPGO format, which stores a trie of call stacks.

## 67.5 `llvm-profdata` Tool

`llvm-profdata` is the command-line tool for manipulating profile data:

```bash
# Merge multiple raw profiles
llvm-profdata merge -output=merged.profdata *.profraw

# Show profile statistics
llvm-profdata show merged.profdata

# Show per-function information
llvm-profdata show -function=hot_function merged.profdata

# Convert between formats (instrumentation ↔ sample)
llvm-profdata merge -sample -output=sample.prof instr.profdata

# Overlap two profiles (find coverage commonality)
llvm-profdata overlap profile1.profdata profile2.profdata

# Thin profiles for ThinLTO (only include functions present in summary)
llvm-profdata merge -output=thin.profdata --thin-link-bitcode=module.bc *.profraw
```

`llvm-profdata` is also used for coverage data (`.gcda` files, when using `--instr` or `--gcov` mode).

## 67.6 ThinLTO + PGO

ThinLTO (Chapter 77) performs link-time optimization across modules without requiring the full LTO merge. With PGO, ThinLTO can inline functions across modules based on profile data:

```bash
# Stage 1: compile all objects with PGO instrumentation
clang -O2 -fprofile-generate -flto=thin -c a.cpp -o a.o
clang -O2 -fprofile-generate -flto=thin -c b.cpp -o b.o

# Stage 2: link (runs ThinLTO pre-link, collects profile)
clang -O2 -fprofile-generate -flto=thin a.o b.o -o program_instr
./program_instr workload
llvm-profdata merge -output=merged.profdata *.profraw

# Stage 3: optimized ThinLTO build
clang -O2 -fprofile-use=merged.profdata -flto=thin a.cpp -c -o a.o
clang -O2 -fprofile-use=merged.profdata -flto=thin b.cpp -c -o b.o
clang -O2 -fprofile-use=merged.profdata -flto=thin a.o b.o -o program_opt
```

The ThinLTO summary index carries per-function hotness information derived from the profile, enabling cross-module inlining decisions that respect the profile.

## 67.7 PGO + BOLT

BOLT (Binary Optimization and Layout Tool, Chapter 118) optimizes already-compiled binaries using profile data. The BOLT + PGO combination is a two-level approach:

1. Compile with standard PGO (profile-guided compilation).
2. Run BOLT on the resulting binary with a separate perf profile.
3. BOLT applies binary-level transformations: basic block reordering, function layout, and hot/cold splitting.

```bash
# Compile with PGO
clang -O3 -fprofile-use=merged.profdata -o program_pgo program.cpp

# Instrument binary for BOLT
llvm-bolt program_pgo -instrument -instrumentation-file=bolt.fdata -o program_bolt_instr

# Run instrumented binary
./program_bolt_instr workload

# Apply BOLT optimizations
llvm-bolt program_pgo -data=bolt.fdata -reorder-blocks=ext-tsp \
    -reorder-functions=hfsort -split-functions -split-all-cold \
    -o program_final
```

BOLT transformations complement compiler PGO: the compiler's PGO affects code generation decisions (inlining, vectorization), while BOLT's reordering reduces instruction cache misses in the final binary layout.

## 67.8 MemProf: Context-Sensitive Heap Profiling

Standard PGO (§67.1) measures how often each branch is taken or each function is called, but it says nothing about *how memory is allocated*. A function may be on the hot path and allocate mostly short-lived temporary objects — knowledge that would change inlining, specialization, and even allocator selection decisions. **MemProf** (Memory Profiler) fills this gap: it is a separate profiling mode that records per-allocation-site heap behavior, including lifetime, access frequency, and call-stack context.

MemProf was developed primarily by Google, deployed in production to reduce resident set size (RSS) in large C++ services, and merged into LLVM in 2022.

### Instrumentation Phase

```bash
# Compile with MemProf instrumentation:
clang -O2 -fmemory-profile program.cpp -o program_memprof

# Run under typical workload:
./program_memprof typical_workload
# produces: default.memprofraw (or the path in MEMPROF_PROFILE_FILE)
```

The instrumentation intercepts `malloc`/`free` call sites. At each `malloc`, it records:
- A **call-stack hash**: a compact identifier for the full call chain leading to the allocation.
- The **allocation size**.
- At `free` time: the **lifetime** (ticks between alloc and free), distinguishing short-lived (stack-like) from long-lived (persistent) allocations.
- **Access counts**: how many loads/stores touch the allocated memory during its lifetime.

This data is written to `.memprofraw` in a binary format.

### Profile Merging and Format

```bash
# Merge MemProf raw profiles (same llvm-profdata tool):
llvm-profdata merge --memory -output=memprof.profdata default.memprofraw

# Inspect profile contents:
llvm-profdata show --memory memprof.profdata

# YAML serialization (LLVM 25+): for human inspection and editing:
llvm-profdata show --memory --yaml memprof.profdata > memprof.yaml
```

The merged `.profdata` contains **MemProf records** alongside any instrumentation PGO records. Each MemProf record maps a call-stack hash to allocation statistics: total bytes allocated, total access count, average lifetime category (cold/warm/hot).

### IR Annotation: !memprof Metadata

In the use phase, `malloc` call sites in the IR are annotated with `!memprof` metadata listing per-context profiles:

```llvm
; An allocation site called from two different call stacks:
%p = call ptr @malloc(i64 %size), !memprof !10

!10 = !{!11, !12}
!11 = !{!13, !"notcold"}         ; call stack A: allocation is warm
!12 = !{!14, !"cold"}            ; call stack B: allocation is cold/short-lived
!13 = !{i64 1234, i64 5678}     ; call stack A hashes
!14 = !{i64 2345, i64 6789}     ; call stack B hashes
```

The `!"cold"` annotation indicates that in call-stack B, this allocation is short-lived. The compiler can redirect such allocations to a **cold allocator** (e.g., a bump allocator with no fragmentation concern) or simply mark the region as cold for the OS's memory reclamation heuristics.

### Use Phase: Enabling MemProf Optimization

```bash
# Compile with MemProf profile use:
clang -O2 -fmemory-profile-use=memprof.profdata program.cpp -o program_opt
```

The pass `MemProfContextDisambiguationPass` (in `llvm/Transforms/IPO/MemProfContextDisambiguation.cpp`) performs **allocation context matching**: it walks the call-stack trie encoded in the profile and matches each profiled call stack to an IR call site via the module's call graph. Call sites that are uniquely hot receive one IR annotation; cold contexts receive another.

### Context Sensitivity: Same malloc, Different Callers

A single `malloc()` call site can be reached from many call stacks. MemProf tracks each stack separately:

```
hot_function() → helper() → malloc()   [1M allocations, avg lifetime 5s]
cold_function() → helper() → malloc()  [100 allocations, avg lifetime 0.01s]
```

With context sensitivity, the compiler knows that when `helper()` is inlined into `hot_function()`, the resulting allocation is long-lived and should use the standard allocator; when inlined into `cold_function()`, it is short-lived and a cold path allocation is appropriate.

### Cold Heap Removal and Allocation Routing

```llvm
; After context disambiguation, cold allocations may be routed:
%p = call ptr @malloc_cold(i64 %size)   ; replaced for cold contexts
```

The exact cold-allocator API is platform-specific (Google's TCMalloc exposes `tcmalloc::MallocExtension::RecordAllocationProfile`; glibc builds may use custom hooks). LLVM's contribution is the IR-level annotation and the context-disambiguation analysis; the runtime plumbing is application-specific.

### Data Layout Profiling

As of 2024, MemProf was extended to record **per-field access counts** for struct allocations. This enables the compiler to suggest or apply struct field reordering to improve cache utilization for the hot fields. The infrastructure is in `llvm/include/llvm/ProfileData/MemProf.h`. Field reordering itself is a separate pass that consumes the layout profile.

### Integration with ThinLTO

MemProf profiles survive module boundaries in ThinLTO: the module summary index carries per-allocation-site hotness annotations, and the context-disambiguation pass runs at link time to match call-stack profiles to the merged IR. This is essential because call stacks frequently cross module boundaries.

### Example: Full Workflow

```bash
# Stage 1: Build instrumented binary
clang -O2 -fmemory-profile -fmemory-profile-use= \
    program.cpp -o program_memprof

# Stage 2: Collect profile
MEMPROF_PROFILE_FILE=run1.memprofraw ./program_memprof workload1
MEMPROF_PROFILE_FILE=run2.memprofraw ./program_memprof workload2

# Stage 3: Merge profiles
llvm-profdata merge --memory -output=merged.memprof run1.memprofraw run2.memprofraw

# Stage 4: Build optimized binary with MemProf use
clang -O2 -fmemory-profile-use=merged.memprof program.cpp -o program_opt
```

---

## 67.9 Profi: Flow-Based Profile Inference

Instrumentation-based PGO adds counters at branches and function entries, but instrumented profiles are rarely complete. Dead-code elimination may remove some counter instrumentation points before they execute. Compiler transformations (inlining, loop unrolling, code motion) can produce IR where some edges have no counter but neighboring edges do. The result is a **partial profile**: some block/edge counts are known, others are zero or missing.

The naive approach is to treat missing counts as zero, which underestimates execution frequency. **Profi** (Profile Inference, 2021) solves this by formulating count inference as a **min-cost max-flow problem** over the CFG.

### Formulation

Given a function's CFG with some edge/block counts known from instrumentation, Profi solves for the full set of counts such that:

1. **Flow conservation:** At every internal node, the sum of in-edge counts equals the sum of out-edge counts (Kirchhoff's law for program flow).
2. **Non-negativity:** All counts ≥ 0.
3. **Fidelity to observed counts:** Deviations from measured counts are penalized proportionally to confidence in the measurement.

The optimization objective is to minimize total adjustment cost (weighted by counter reliability), subject to flow conservation. This is a standard **min-cost flow** problem on a directed graph, solvable in polynomial time.

Formally: let `f(e)` be the inferred count for edge `e`, `c(e)` the observed count (if any), and `w(e)` the confidence weight. Profi minimizes:

```
minimize   Σ_e  w(e) * |f(e) - c(e)|
subject to:
  Σ_{e into v} f(e) = Σ_{e out of v} f(e)   for all internal v
  f(e) ≥ 0
```

### Implementation

Profi's inference logic is in [`llvm/lib/ProfileData/InstrProf.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ProfileData/InstrProf.cpp) and `InstrProfWriter.cpp`, invoked via `InstrProfWriter::setValueProfilingData` and the profile reader's balancing step. The flow solver uses a standard shortest-path min-cost flow algorithm over the augmented CFG graph.

### Comparison with Counter Lookup

| Approach | Missing counter | Inconsistent counters |
|----------|-----------------|----------------------|
| Naive lookup | Returns 0 (under-estimates hot paths) | Uses raw value (inconsistent profile) |
| Profi | Infers from neighbors via flow | Resolves inconsistency by min-cost adjustment |

Profi is more robust: a profile collected with `-fprofile-partial-training` (only a subset of workloads represented) produces a partial profile. Without Profi, cold-looking blocks may receive inlining or layout decisions appropriate for cold code even though they are actually warm (just not covered by the training run). Profi's flow balancing propagates the warm signal from instrumented edges to adjacent uninstrumented edges.

### Interaction with ProfileSummaryInfo and BranchProbabilityInfo

After Profi runs (during profile loading in `llvm-profdata` or in the use-phase profile reader), the inferred counts are written as `!prof` branch weight metadata on branches. `BranchProbabilityAnalysis` converts these weights to per-edge probabilities, and `BlockFrequencyAnalysis` computes per-block frequencies. All downstream passes that query `BFI.getBlockFreq(BB)` then see Profi-inferred values rather than raw (possibly missing) instrumentation counts.

`ProfileSummaryInfo` (which classifies blocks as hot, warm, or cold using quantile thresholds) uses the same inferred block frequencies, ensuring that hot/cold splitting (§68.10 context) and inlining threshold adjustments are based on the corrected profile.

### When Profi Is Active

Profi is automatically engaged during the use phase when the profile has gaps:

```bash
# Use phase: Profi inference runs automatically if the profile is partial:
clang -O2 -fprofile-instr-use=partial.profdata program.cpp -o program_opt
```

It can also be invoked explicitly via `llvm-profdata`:

```bash
llvm-profdata merge --infer-missing-counters \
    -output=balanced.profdata partial.profraw
```

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **MemProf YAML serialization stabilization**: The `--yaml` flag for `llvm-profdata show --memory` (merged in LLVM 25) is being hardened for round-trip fidelity; expect RFC patches on discourse.llvm.org to finalize the schema so tooling (e.g., Google's internal profile analysis pipelines) can consume it portably.
- **Binary PGO profile format v9**: An ongoing series of LLVM patches (tracked in `llvm/lib/ProfileData/`) aims to encode value-profiling histograms more compactly by switching from flat arrays to sparse run-length encoding, reducing `.profdata` file sizes by 30–50% for large binaries.
- **CSSPGO + ThinLTO full-pipeline integration**: The `SampleProfileLoader` context-sensitive path historically required a separate LTO pass ordering; LLVM 22 patches are landing to unify the hot/cold call-graph trie propagation with the ThinLTO summary index in a single pipeline step (see discourse.llvm.org thread "CSSPGO ThinLTO unification").
- **Profi counter pruning**: A follow-on to the 2021 Profi paper proposes pruning redundant counters at instrumentation time (not just at inference time), reducing instrumentation overhead by ~15% without sacrificing inference accuracy; patches are in review in `llvm/Transforms/Instrumentation/PGOInstrumentation.cpp`.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Hardware-assisted PGO via Intel PEBS / ARM SPE**: Rather than software edge counters, near-term processors expose precise branch sampling (Intel PEBS, AMD IBS, ARM Statistical Profiling Extension) with sub-percent overhead. LLVM's AutoFDO pipeline is being extended to consume SPE profiles natively without the `create_llvm_prof` conversion step, enabling always-on production profiling at negligible cost.
- **MemProf struct field reordering as a first-class pass**: The per-field access count infrastructure added in 2024 (`llvm/include/llvm/ProfileData/MemProf.h`) will evolve into an opt-in `-fmemory-layout-optimize` pass that rewrites struct definitions and their use sites, directly reducing cache line pressure for hot allocations — a feature previously available only via manual annotation or PGO-informed link-time struct splitting tools.
- **Temporal PGO (tPGO)**: Meta and Google are experimenting with profiles that record *when* during program execution code is hot (startup vs. steady-state), not just *how often*. This enables startup-specific code placement separate from steady-state hot paths, potentially reducing time-to-first-request for services. LLVM infrastructure work for tPGO is tracked in the `llvm-dev` list discussion "Temporal profile encoding for startup optimization."
- **Probabilistic profiling for ML workloads**: As LLVM backends target ML accelerators (NVPTX, AMDGPU, TPU via MLIR), traditional edge-count PGO doesn't translate. Research into probabilistic profile representations that encode distribution of tensor shapes and loop trip counts (rather than single counts) is active, with prototypes in the IREE project and proposals to upstream shape-profile metadata to LLVM's `!prof` infrastructure.

### 5-Year Horizon (Long-Term, by ~2031)

- **Profile-guided register allocation**: Current PGO feeds inlining and branch prediction but rarely influences register allocator spill decisions. Long-term research (building on the PBQP and greedy allocator work) aims to use block frequency information to bias spill selection toward cold blocks, reducing spill/reload traffic on hot paths — a direction outlined in several CGO and PLDI papers on profile-aware RA.
- **Continuous runtime reoptimization (JIT + PGO loop)**: Combining LLVM's ORC JIT (Chapter 103) with an always-on MemProf + edge-count profiler, future runtimes could automatically recompile hot functions with updated profiles without stopping execution. This closes the PGO feedback loop to seconds rather than hours, following the direction of Java HotSpot's tiered compilation but at the C++ binary level.
- **Differential PGO for A/B testing**: Enterprise deployments want to compare profile-optimized binaries across code versions. A proposed `llvm-profdata diff` sub-command (beyond the current `overlap`) would compute statistically significant hotness deltas between profdata files, enabling automated regression detection in CI when a code change shifts the hot path.

---

## Chapter Summary

- Instrumentation-based PGO uses `-fprofile-generate` / `-fprofile-use`; the instrumented binary writes `default.profraw`, merged with `llvm-profdata merge`.
- Profile data attaches `!prof` branch weight metadata to branches and `!vpid` to indirect calls and memcpy, feeding `BranchProbabilityAnalysis` and value-profile-based specialization.
- AutoFDO / SampleProfile collects production profiles from `perf record` without recompilation; profiles are converted with `create_llvm_prof` and consumed with `-fprofile-sample-use`.
- Pseudo-probes (`-fpseudo-probe-for-profiling`) insert stable IR markers that persist through optimization, providing more reliable sample-to-IR matching.
- CSSPGO maintains context-sensitive profiles indexed by call stack, enabling per-context inlining decisions for functions called from both hot and cold callers.
- `llvm-profdata` merges, shows, converts, and overlaps profile data; it also supports ThinLTO thin-profile generation.
- ThinLTO + PGO propagates hotness information through the module summary index for cross-module inlining decisions.
- BOLT + PGO provides a two-level optimization: PGO for compiler decisions during code generation, BOLT for binary-level basic block layout and function ordering.
- **MemProf** (`-fmemory-profile` / `-fmemory-profile-use`) instruments `malloc`/`free` call sites with call-stack hashes to record allocation lifetime and access frequency; IR call sites receive `!memprof` metadata; context disambiguation (`MemProfContextDisambiguationPass`) matches per-stack profiles to IR; cold allocations can be routed to cold allocators; survives ThinLTO via the module summary index.
- **Profi** formulates profile inference as a min-cost max-flow problem over the CFG, filling in missing or inconsistent counters from a partial instrumentation profile; inferred counts feed `BranchProbabilityAnalysis` and `BlockFrequencyAnalysis`, ensuring correct hot/cold decisions downstream.


---

@copyright jreuben11
