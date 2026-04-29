# Chapter 115 — Source-Based Code Coverage

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

Code coverage — measuring which lines, branches, and conditions a test suite exercises — is a foundational quality metric for any serious software project. LLVM's source-based code coverage system is the most precise and modern coverage mechanism in the Clang toolchain: it operates at the source region level (not line level), supports branch and MC/DC metrics, and generates both human-readable HTML reports and machine-readable JSON and LCOV output for CI integration. Unlike the older GCC-compatible gcov format or SanitizerCoverage (which is designed for fuzzing feedback, not human reports), source-based coverage provides exact, compiler-verified mappings from runtime counters to source code locations. This chapter covers the full pipeline from instrumentation through `llvm-profdata` to `llvm-cov` reporting, including branch coverage, MC/DC for safety-critical compliance, and CI integration patterns.

---

## 115.1 Coverage Approaches in LLVM

LLVM and Clang support three distinct coverage mechanisms, each designed for a different use case:

### 1. gcov-Compatible Coverage

```bash
clang-22 -fprofile-arcs -ftest-coverage source.cpp -o binary
./binary
gcov source.cpp
```

This generates `.gcda` (arc data) and `.gcno` (note) files in GCC's gcov format. Tools like `gcovr` and `lcov`/`genhtml` consume these. The format is line-level and function-level; it does not support sub-statement region coverage. Compatible with GCC toolchains.

### 2. Source-Based Coverage (This Chapter)

```bash
clang-22 -fprofile-instr-generate -fcoverage-mapping source.cpp -o binary
./binary
llvm-profdata merge default.profraw -o default.profdata
llvm-cov report binary -instr-profile=default.profdata
```

This is Clang's native coverage format. Coverage is tracked at region granularity (individual expressions within a line), branch granularity, and — since LLVM 17 — MC/DC granularity. The mapping from counters to source ranges is encoded in the binary itself, not in a separate file.

### 3. SanitizerCoverage

Covered in Chapter 114; designed for fuzzing feedback, not human-readable reports. Uses `__sanitizer_cov_trace_pc_guard` for runtime edge counting with minimal overhead. Not suitable for compliance-oriented coverage reports.

### Choosing Between Formats

| Property | gcov | Source-Based | SanitizerCoverage |
|----------|------|-------------|-------------------|
| Granularity | Line + function | Region + branch + MC/DC | Edge (coarse) |
| Overhead | Medium | Low-Medium | Very low |
| Format | .gcda/.gcno | .profraw/.profdata | In-memory |
| Tools | gcov, gcovr, lcov | llvm-profdata, llvm-cov | libFuzzer internals |
| Use case | GCC compatibility | Clang-native reports, CI | Fuzzing |

For new projects using Clang, source-based coverage is the recommended choice.

---

## 115.2 Instrumentation: How It Works

### Compilation Flags

Two flags are required:

- `-fprofile-instr-generate`: instructs Clang to insert counter increments into the compiled code
- `-fcoverage-mapping`: instructs Clang to embed a mapping from counters to source ranges into the binary

These flags can be separated: `-fprofile-instr-generate` alone produces profiles usable for PGO (Profile-Guided Optimization, Chapter 67); the combination with `-fcoverage-mapping` adds the source-mapping information needed for `llvm-cov`.

### What Gets Instrumented

The Clang frontend, during IR generation, inserts coverage counter increments at:

- **Function entry**: one counter per function, always incremented on function entry
- **Region entry**: one counter per "code region" — a maximal sequence of statements with no branch
- **Branch points**: additional counters (or counter expressions) at if/else, loop conditions, switch cases, ternary operators

Consider this function:

```cpp
int classify(int x) {
    if (x > 0) {      // region 1 starts here
        return 1;     // region 2
    } else if (x < 0) {  // region 3
        return -1;    // region 4
    }
    return 0;         // region 5
}
```

Clang assigns counters to regions 1 and 2 (regions 3, 4, 5 can be derived by subtraction: `count(3) = count(1) - count(2)`). This counter-minimization reduces binary overhead.

### The Coverage Mapping Format

`-fcoverage-mapping` embeds a compact binary structure in the `__llvm_covmap` section of the binary:

```
__llvm_covmap section:
  CoverageMappingHeader
    version: 4 (current)
    nRecords: N
  For each function:
    FunctionMappingRecord:
      nameHash       : u64 (hash of mangled function name)
      dataSize       : u32 (size of mapping data)
      funcHash       : u64 (hash of function body for staleness detection)
      filenames[]    : list of source file references
      mappings[]     : (counter, fileID, startLine, startCol, endLine, endCol)
```

Counter values are not stored in the binary; they are tracked at runtime in the `__llvm_prf_cnts` section (writable data). The `__llvm_covmap` section is read-only and position-independent, making it safe to share between processes and load from the binary even without executing it.

### Optimization and Coverage

Coverage instrumentation is typically combined with optimization disabled (`-O0`) to ensure a 1-to-1 correspondence between source statements and instrumented regions. However, source-based coverage works at all optimization levels — if `-O2` eliminates a dead code path, that path will correctly show as uncovered (it was optimized out; Clang preserves coverage records for eliminated code).

```bash
# Standard development build (O0 for accurate coverage):
clang-22 -fprofile-instr-generate -fcoverage-mapping -O0 -g source.cpp

# CI build (O1 for speed + coverage):
clang-22 -fprofile-instr-generate -fcoverage-mapping -O1 source.cpp
```

---

## 115.3 The Profile Runtime

### Runtime Library Location

The profile runtime lives in [`compiler-rt/lib/profile/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/lib/profile/InstrProfilingFile.c). Key source files:

| File | Role |
|------|------|
| `InstrProfiling.c` | Core profiling data structures |
| `InstrProfilingBuffer.c` | In-memory counter management |
| `InstrProfilingFile.c` | Writing `.profraw` files on exit |
| `InstrProfilingRuntime.cpp` | C++ entry points, atexit registration |
| `InstrProfilingValue.c` | Value profiling (indirect call targets, etc.) |
| `InstrProfilingPlatformLinux.c` | Linux-specific memory mapping |

### Counter Storage

When compiled with `-fprofile-instr-generate`, the binary contains:

- `__llvm_prf_cnts`: writable section containing one `uint64_t` counter per region
- `__llvm_prf_data`: per-function metadata (function hash, pointer to counters, pointer to name)
- `__llvm_prf_names`: compressed function name table
- `__llvm_covmap`: coverage mapping (if `-fcoverage-mapping` is set)

At runtime, counter increments are simple memory stores to the `__llvm_prf_cnts` section:

```llvm
; Instrumented IR for a branch region:
%pgo_counter = load i64, i64* getelementptr(... @__profc_classify, i64 0, i64 1)
%inc = add i64 %pgo_counter, 1
store i64 %inc, i64* getelementptr(... @__profc_classify, i64 0, i64 1)
```

### Writing the Profile

On process exit, `__llvm_profile_write_file()` is called (registered via `atexit`). It writes the raw profile to disk in the `.profraw` format.

The output path is controlled by:

1. **`LLVM_PROFILE_FILE` environment variable** (highest priority):
   ```bash
   LLVM_PROFILE_FILE="profile-%p-%h.profraw" ./binary
   # %p: replaced by process PID
   # %h: replaced by hostname
   # %Nm: merge mode (see below)
   ```

2. **Default**: `default.profraw` in the current directory

3. **Programmatic control**:
   ```cpp
   // Call from C/C++ code:
   extern "C" int __llvm_profile_write_file(void);
   extern "C" void __llvm_profile_reset_counters(void);
   extern "C" int __llvm_profile_is_continuous_mode_enabled(void);
   ```

### Continuous Coverage Mode

For long-running services (web servers, daemons), exiting to flush profiles is not an option. Continuous coverage mode writes profile data incrementally:

```bash
# Enable continuous mode:
LLVM_PROFILE_FILE="%Nmprofile.profraw" ./server
# The %Nm prefix enables memory-mapped continuous mode
```

In continuous mode, the `__llvm_prf_cnts` section is memory-mapped directly to the `.profraw` file. Counter increments write through to the file without an explicit flush. This allows you to send SIGKILL to the process and still recover a valid profile.

```cpp
// Check if continuous mode is active:
if (__llvm_profile_is_continuous_mode_enabled()) {
    // Profile is being written continuously
} else {
    // Profile will be written on exit
    __llvm_profile_write_file();  // can call manually
}
```

### Multi-Process and Parallel Test Coverage

When running parallel tests, each process writes its own `.profraw` file. The `%p` pattern ensures uniqueness:

```bash
# Run tests in parallel:
LLVM_PROFILE_FILE="coverage-%p.profraw" ctest -j16

# Merge all profiles:
/usr/lib/llvm-22/bin/llvm-profdata merge \
  coverage-*.profraw \
  -o merged.profdata

# Generate report from merged profile:
/usr/lib/llvm-22/bin/llvm-cov report \
  ./binary -instr-profile=merged.profdata
```

---

## 115.4 llvm-profdata

`llvm-profdata` is the tool for merging, inspecting, and manipulating profile data files.

### Merging Profiles

```bash
# Basic merge:
llvm-profdata merge profile1.profraw profile2.profraw -o merged.profdata

# Merge a directory of profiles:
llvm-profdata merge coverage-*.profraw -o merged.profdata

# Weighted merge (profile2 counts twice as much as profile1):
llvm-profdata merge \
  -weighted-input=1,profile1.profraw \
  -weighted-input=2,profile2.profraw \
  -o merged.profdata

# Merge with failure tolerance (ignore profiles that don't match binary):
llvm-profdata merge --failure-mode=all coverage-*.profraw -o merged.profdata
```

Weighted merging is useful when combining profiles from different workloads — production traces, unit tests, and integration tests — where each should contribute proportionally.

### Inspecting Profile Contents

```bash
# Show all functions and their execution counts:
llvm-profdata show merged.profdata --all-functions

# Show a specific function:
llvm-profdata show merged.profdata --function=classify

# Output:
# classify:
#   Hash: 0x7f3a9b2ec3f840d9
#   Counters: 5
#   Block counts: [1024, 512, 0, 512, 0]

# Show profile summary (max count, coverage ratio):
llvm-profdata show merged.profdata --counts --topn=10

# Show in YAML format:
llvm-profdata show merged.profdata --all-functions --output=profile.yaml
```

### Profile Overlap Analysis

```bash
# Compute coverage overlap between two profiles:
llvm-profdata overlap profile_a.profdata profile_b.profdata

# Output:
# Filename            Func overlap    Block overlap
# source.cpp          85.00%          73.00%
# Total               85.00%          73.00%
```

The overlap analysis helps identify whether two test suites are testing the same code paths.

### Profile Formats

| Extension | Format | Description |
|-----------|--------|-------------|
| `.profraw` | Raw binary | Output of instrumented binary; not indexed |
| `.profdata` | Indexed binary | Output of `llvm-profdata merge`; indexed by function name |
| `.proftext` | Text | Human-readable; round-trips with `show`/`merge` |

The `.profraw` format has a magic number `0x81726570666c6c77` (little-endian "llvmprof"). The `.profdata` format adds a function name → offset index for fast lookup by `llvm-cov`.

---

## 115.5 llvm-cov

`llvm-cov` consumes an instrumented binary plus a `.profdata` file to generate coverage reports.

### Source Annotation: `llvm-cov show`

```bash
# Annotated source for all files:
llvm-cov show ./binary -instr-profile=merged.profdata

# Annotated source for a specific file:
llvm-cov show ./binary -instr-profile=merged.profdata src/parser.cpp

# With demangled C++ names:
llvm-cov show ./binary -instr-profile=merged.profdata --Demangle

# HTML output:
llvm-cov show ./binary -instr-profile=merged.profdata \
  --format=html \
  --output-dir=coverage_html/ \
  src/
```

The text output format:

```
      |  1| int classify(int x) {
 1024 |  2|   if (x > 0) {
  512 |  3|     return 1;
      |  4|   } else if (x < 0) {
    0 |  5|     return -1;
  512 |  6|   }
      |  7|   return 0;
      |  8| }
```

Numbers on the left show execution counts per region. `0` indicates an uncovered region. Lines with no count are non-executable (declarations, braces, etc.).

### Summary Report: `llvm-cov report`

```bash
# Per-file summary:
llvm-cov report ./binary -instr-profile=merged.profdata

# Output:
# Filename            Regions  Missed  Cover   Functions  Missed  Cover    Lines  Missed  Cover
# /src/parser.cpp     120      18      85.00%  24         2       91.67%   342    31      90.94%
# /src/lexer.cpp      89       4       95.51%  18         0       100.00%  214    7       96.73%
# TOTAL               209      22      89.47%  42         2       95.24%   556    38      93.17%

# Sort by coverage (ascending = worst first):
llvm-cov report ./binary -instr-profile=merged.profdata \
  --sort-regions-by-count
```

### JSON Export: `llvm-cov export`

For CI integration and custom dashboards:

```bash
# Export to JSON:
llvm-cov export ./binary -instr-profile=merged.profdata \
  --format=text > coverage.json

# Export to LCOV format (for genhtml, Codecov, Coveralls):
llvm-cov export ./binary -instr-profile=merged.profdata \
  --format=lcov > coverage.lcov

# Generate HTML from LCOV (traditional workflow):
genhtml coverage.lcov --output-directory coverage_html/
```

The JSON export schema includes, for each function:

```json
{
  "name": "classify",
  "count": 1024,
  "regions": [
    { "start": [2, 5], "end": [8, 2], "count": 1024, "kind": "code" },
    { "start": [3, 5], "end": [3, 15], "count": 512, "kind": "code" }
  ],
  "branches": [
    { "start": [2, 7], "end": [2, 12], "count": 512, "falseCount": 512 }
  ]
}
```

### Filtering and Object Files

For multi-binary or shared library projects:

```bash
# Coverage from multiple object files:
llvm-cov show \
  -object binary \
  -object libfoo.so \
  -instr-profile=merged.profdata

# Show only specific source paths (filter):
llvm-cov show binary -instr-profile=merged.profdata \
  --ignore-filename-regex='\.pb\.cc$|third_party/'

# Show with source path remapping (for out-of-tree builds):
llvm-cov show binary -instr-profile=merged.profdata \
  --path-equivalence=/build/src,/home/user/src
```

---

## 115.6 Branch Coverage and MC/DC

### Branch Coverage

Branch coverage tracks whether each branch of a conditional expression was taken in both directions. This catches cases where line coverage reports 100% but one branch of an `if` was never tested:

```bash
# Enable branch coverage display:
llvm-cov show binary -instr-profile=merged.profdata --show-branches=count

# Output for: if (x > 0 && y < 0):
#   2|  if (x > 0 && y < 0) {
#    |  ^-- Branch (taken: 100, not taken: 0)
#    |              ^-- Branch (taken: 50, not taken: 50)
```

Branch counters are stored alongside region counters. Clang generates branch counter instrumentation automatically with `-fcoverage-mapping`.

### MC/DC — Modified Condition/Decision Coverage

MC/DC is required by DO-178C (aviation software), IEC 61508 (safety-critical systems), and similar safety standards. It is stronger than branch coverage and weaker than path coverage.

**Definition**: For a decision `D` containing conditions `C1, C2, ..., Cn`, MC/DC requires that:
1. Every condition is evaluated as both `true` and `false`.
2. Each condition independently affects the outcome of the decision.

For a decision with N independent boolean conditions, MC/DC requires N+1 test cases (not 2^N).

```bash
# Enable MC/DC instrumentation:
clang-22 -fprofile-instr-generate -fcoverage-mapping \
         -fcoverage-mcdc \
         -O0 source.cpp -o binary
```

Example: `if (a && b && c)` requires 4 MC/DC test cases:

| Test | a | b | c | Decision |
|------|---|---|---|----------|
| T1   | T | T | T | T        |
| T2   | F | - | - | F        | (a uniquely determines)
| T3   | T | F | - | F        | (b uniquely determines)
| T4   | T | T | F | F        | (c uniquely determines)

MC/DC display in `llvm-cov show`:

```bash
llvm-cov show binary -instr-profile=merged.profdata --show-mcdc
# Output shows per-condition MC/DC pairs and whether each is covered
```

MC/DC tracking uses a bitmask approach: each decision maintains a counter per unique input/output condition combination observed at runtime. After the run, `llvm-cov` checks which (condition, decision) independent pairs are satisfied.

---

## 115.7 Integration with Build Systems and CI

### CMake Integration

```cmake
# CMakeLists.txt:
option(ENABLE_COVERAGE "Enable source-based coverage" OFF)

if(ENABLE_COVERAGE)
    if(NOT CMAKE_C_COMPILER_ID MATCHES "Clang")
        message(FATAL_ERROR "Source-based coverage requires Clang")
    endif()
    
    add_compile_options(
        -fprofile-instr-generate
        -fcoverage-mapping
    )
    add_link_options(
        -fprofile-instr-generate
    )
    
    # Custom target for generating coverage report:
    find_program(LLVM_PROFDATA llvm-profdata
                 PATHS /usr/lib/llvm-22/bin REQUIRED)
    find_program(LLVM_COV llvm-cov
                 PATHS /usr/lib/llvm-22/bin REQUIRED)
    
    add_custom_target(coverage
        # Run tests:
        COMMAND ${CMAKE_CTEST_COMMAND} --test-dir ${CMAKE_BINARY_DIR}
        # Merge profiles:
        COMMAND ${LLVM_PROFDATA} merge
                ${CMAKE_BINARY_DIR}/coverage-*.profraw
                -o ${CMAKE_BINARY_DIR}/merged.profdata
        # Generate HTML report:
        COMMAND ${LLVM_COV} show
                ${CMAKE_BINARY_DIR}/your_binary
                -instr-profile=${CMAKE_BINARY_DIR}/merged.profdata
                --format=html
                --output-dir=${CMAKE_BINARY_DIR}/coverage_html/
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    )
endif()
```

```bash
# Build and run coverage:
cmake -DENABLE_COVERAGE=ON -DCMAKE_C_COMPILER=clang-22 \
      -DCMAKE_CXX_COMPILER=clang++-22 ..
LLVM_PROFILE_FILE="${CMAKE_BINARY_DIR}/coverage-%p.profraw" \
  make coverage
```

### Bazel Integration

Bazel's built-in `--collect_code_coverage` uses gcov by default. To use LLVM source-based coverage:

```python
# .bazelrc:
build:coverage --collect_code_coverage
build:coverage --copt=-fprofile-instr-generate
build:coverage --copt=-fcoverage-mapping
build:coverage --linkopt=-fprofile-instr-generate
build:coverage --define=LLVM_COVERAGE=1

# BUILD rule:
cc_test(
    name = "my_test",
    srcs = ["my_test.cpp"],
    # Coverage flags applied automatically from .bazelrc
)
```

```bash
bazel coverage --config=coverage //my/package:my_test
```

### GitHub Actions Integration

```yaml
# .github/workflows/coverage.yml
name: Coverage
on: [push, pull_request]

jobs:
  coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install LLVM 22
        run: |
          wget -O- https://apt.llvm.org/llvm-snapshot.gpg.key | sudo apt-key add -
          sudo add-apt-repository "deb http://apt.llvm.org/jammy/ llvm-toolchain-jammy-22 main"
          sudo apt-get install -y clang-22 llvm-22
      
      - name: Configure
        run: |
          cmake -B build -DCMAKE_C_COMPILER=clang-22 \
                -DCMAKE_CXX_COMPILER=clang++-22 \
                -DENABLE_COVERAGE=ON
      
      - name: Build and Test
        run: |
          cmake --build build
          LLVM_PROFILE_FILE="build/coverage-%p.profraw" \
            ctest --test-dir build
      
      - name: Merge profiles
        run: |
          /usr/lib/llvm-22/bin/llvm-profdata merge \
            build/coverage-*.profraw \
            -o build/merged.profdata
      
      - name: Generate LCOV report
        run: |
          /usr/lib/llvm-22/bin/llvm-cov export \
            build/my_binary \
            -instr-profile=build/merged.profdata \
            --format=lcov \
            --ignore-filename-regex='third_party/' \
            > build/coverage.lcov
      
      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: build/coverage.lcov
          fail_ci_if_error: true
```

### Parallel Test Sharding

For large test suites split across multiple CI shards:

```bash
# Shard 1 (out of 4):
LLVM_PROFILE_FILE="shard1-%p.profraw" \
  ./run_tests --shard=1/4

# Shard 2:
LLVM_PROFILE_FILE="shard2-%p.profraw" \
  ./run_tests --shard=2/4

# ... (upload all .profraw files as artifacts)

# Final merge job (downloads all artifacts):
llvm-profdata merge shard*.profraw -o merged.profdata
llvm-cov export binary -instr-profile=merged.profdata --format=lcov \
  > coverage.lcov
```

This pattern is common in projects like LLVM itself, where the test suite takes too long to run on a single machine.

---

## 115.8 Coverage for Fuzzing Feedback

### Measuring Fuzzer Effectiveness

Source-based coverage can measure how much of the target code a fuzzer's corpus has exercised — a more precise metric than SanitizerCoverage's edge count:

```bash
# Compile fuzz target with BOTH fuzzer instrumentation AND coverage mapping:
clang-22 -fsanitize=fuzzer,address \
         -fprofile-instr-generate -fcoverage-mapping \
         -O1 fuzz_target.cpp target_lib.cpp -o fuzz_target_cov

# Run the corpus (don't fuzz, just replay):
LLVM_PROFILE_FILE="fuzzcov-%p.profraw" \
  ./fuzz_target_cov corpus/ -runs=0

# Merge and report:
llvm-profdata merge fuzzcov-*.profraw -o fuzzcov.profdata
llvm-cov report ./fuzz_target_cov -instr-profile=fuzzcov.profdata \
  --ignore-filename-regex='fuzzer/'  # Exclude fuzzer engine itself
```

This workflow answers: "Which source lines/branches has the fuzzer corpus exercised?"

### Identifying Uncovered Code

After measuring coverage, identify which functions or files have low coverage — these are candidates for:

1. Additional seed inputs (if there exist valid inputs that exercise these paths)
2. Manual test cases
3. Dictionary additions (if coverage-blocking is a magic constant check)

```bash
# Find functions with 0% coverage:
llvm-cov report binary -instr-profile=coverage.profdata \
  --output-delimiter=, 2>/dev/null \
  | awk -F, '$3 == "0.00%" {print $1, $2}'

# Find files with < 50% coverage:
llvm-cov report binary -instr-profile=coverage.profdata \
  | awk '$NF < "50.00%" {print}'
```

### Differential Coverage

Comparing coverage before and after adding a new test:

```bash
# Before: coverage from existing tests
llvm-profdata merge before-*.profraw -o before.profdata

# After: coverage from existing tests + new test
llvm-profdata merge after-*.profraw -o after.profdata

# Overlap analysis:
llvm-profdata overlap before.profdata after.profdata
# Shows which additional functions/blocks the new test covers
```

### Coverage-Guided CI Gates

Many projects set minimum coverage thresholds in CI. A simple implementation:

```bash
#!/bin/bash
COVERAGE_PCT=$(llvm-cov report binary -instr-profile=merged.profdata \
  | tail -1 | awk '{print $NF}' | tr -d '%')

if (( $(echo "$COVERAGE_PCT < 80.0" | bc -l) )); then
  echo "ERROR: Coverage ${COVERAGE_PCT}% is below 80% threshold"
  exit 1
fi
echo "Coverage: ${COVERAGE_PCT}% (threshold: 80%)"
```

---

## Chapter Summary

- **Three coverage mechanisms**: gcov-compatible (GCC format), source-based (Clang-native, most precise), and SanitizerCoverage (fuzzing feedback). This chapter covers source-based coverage.

- **Instrumentation** requires `-fprofile-instr-generate -fcoverage-mapping`. Clang inserts counter increments at CFG region entries; the mapping from counters to source ranges is embedded in `__llvm_covmap`.

- **The profile runtime** (`compiler-rt/lib/profile/`) manages in-memory counters and writes `.profraw` files on exit. `LLVM_PROFILE_FILE` controls the output path; `%p` pattern handles parallel processes. Continuous mode (`%Nm`) enables live profile writing for long-running services.

- **`llvm-profdata merge`** combines multiple `.profraw` files into an indexed `.profdata` file. Weighted merging combines profiles from different workloads proportionally. The `show` and `overlap` subcommands inspect profile contents.

- **`llvm-cov show`** annotates source with per-region execution counts. `llvm-cov report` provides per-file/function coverage percentages. `llvm-cov export --format=lcov` enables integration with Codecov, Coveralls, and other platforms.

- **Branch coverage** tracks true/false outcomes at every conditional. **MC/DC** (`-fcoverage-mcdc`) provides modified condition/decision coverage required by DO-178C avionics compliance standards.

- **Build system integration**: CMake custom targets, Bazel `.bazelrc` overrides, GitHub Actions with LCOV upload, and parallel test shard merging.

- **Fuzzing feedback**: run the fuzzer corpus with `-runs=0` to generate a source-based coverage report showing exactly which code the fuzzer has explored, enabling targeted corpus augmentation.
