# Chapter 67 — Profile-Guided Optimization

*Part X — Analysis and the Middle-End*

Profile-guided optimization (PGO) uses dynamic profile data — counts of how often each code path executes at runtime — to make better optimization decisions. Inlining threshold increases for hot call sites, loop unrolling for high-trip-count loops, and function layout for better icache behavior all depend on knowing *which code is actually hot*. This chapter covers LLVM's instrumentation-based PGO, SampleProfile/AutoFDO, pseudo-probes, context-sensitive PGO (CSSPGO), the `llvm-profdata` tool, and PGO integration with ThinLTO and BOLT.

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


---

@copyright jreuben11
