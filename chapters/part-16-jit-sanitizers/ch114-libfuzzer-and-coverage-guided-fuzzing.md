# Chapter 114 — LibFuzzer and Coverage-Guided Fuzzing

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

Fuzzing has become one of the most productive bug-finding techniques in software security. Coverage-guided fuzzing — where the fuzzer observes which code paths each input exercises and preferentially mutates inputs that explore new paths — finds bugs that reviewers miss and symbolic execution cannot reach at scale. LibFuzzer is LLVM's in-process, coverage-guided fuzzing engine, embedded in `compiler-rt` and tightly integrated with Clang's instrumentation infrastructure. It powers OSS-Fuzz, drives bug discovery in almost every major open-source C/C++ project, and forms the foundation of more advanced structured fuzzers. This chapter covers libFuzzer's architecture, SanitizerCoverage instrumentation, corpus management, sanitizer integration, and production deployment via OSS-Fuzz.

---

## 114.1 Fuzzing Fundamentals

### What Fuzzing Is

Fuzzing is the practice of feeding randomly or semi-randomly generated inputs to a program and observing whether those inputs cause crashes, assertion failures, memory safety violations, or other anomalous behavior. The core insight is that complex parsers, protocol implementations, and data processors have large input spaces and nontrivial invariants; automated generation explores this space far faster than manual testing.

The simplest fuzzers (dumb fuzzers) generate purely random bytes. While surprisingly effective at finding bugs in simple parsers, they rarely penetrate deep into programs with complex input validation.

### Coverage-Guided Fuzzing

Coverage-guided fuzzers (grey-box fuzzers) address the depth problem by using code coverage feedback:

1. Run an input through the target.
2. Observe which code locations were executed (coverage feedback).
3. If the input executed a new code location not seen before, add it to the corpus.
4. Mutate corpus inputs to generate new inputs.
5. Repeat.

This creates a feedback loop: the corpus gradually accumulates inputs that together cover increasingly deep program logic. Inputs that trigger new coverage are favored for further mutation; inputs that only retread known paths are discarded.

### In-Process vs Fork-Server Models

**AFL/AFL++** (American Fuzzy Lop) uses a fork-server model: the fuzzer forks the target process for each input. The fork-server shortcut (forking from a post-initialization snapshot) reduces per-input overhead, but fork() still costs tens of microseconds.

**libFuzzer** uses an in-process model: the fuzz target function (`LLVMFuzzerTestOneInput`) is called in a tight loop within the same process. This eliminates fork overhead and enables millions of executions per second for small targets. The trade-off: crashes and state corruption from one input may affect subsequent inputs; fuzz targets must be carefully written to avoid persistent state mutations.

### libFuzzer vs AFL-Style Fuzzers

| Property | libFuzzer | AFL/AFL++ |
|----------|-----------|-----------|
| Model | In-process | Fork-server |
| Instrumentation | Compile-time (SanitizerCoverage) | Compile-time or binary-only |
| Corpus format | Directory of files | Directory of files |
| Speed | Very high (millions/s for small targets) | Lower (fork overhead) |
| Structured mutation | `FuzzedDataProvider`, custom mutators | Less integrated |
| Sanitizer integration | Deep (same process) | External (process signal) |
| Multi-target | One binary = one target | One binary = one target |

For targets requiring deep protocol-specific structure, `libprotobuf-mutator` + libFuzzer often outperforms AFL because structured mutation avoids wasting executions on syntactically invalid inputs.

---

## 114.2 Writing a Fuzz Target

### The Fuzz Entry Point

A libFuzzer fuzz target is a C or C++ function with a specific signature:

```cpp
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
    // Process Data[0..Size-1] as a fuzz input.
    // Return 0 always (non-zero return is reserved for future use).
    return 0;
}
```

The `extern "C"` linkage is required to prevent C++ name mangling. The function is called repeatedly by the libFuzzer engine with different inputs derived from corpus mutations.

### Statelessness Requirement

Each call to `LLVMFuzzerTestOneInput` must leave global state unchanged (or reset it). Any persistent state that differs between calls can cause false positives (a previous input's corruption crashes a later input) or false negatives (a previously set flag suppresses a bug).

```cpp
// BAD: global state persists:
static std::vector<int> g_parsed;
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
    g_parsed.clear();    // reset — but what if push_back throws?
    parse_into(g_parsed, Data, Size);
    return 0;
}

// GOOD: all state is local:
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
    std::vector<int> parsed;
    parse_into(parsed, Data, Size);
    return 0;
}
```

For targets that necessarily use global state (e.g., targets that initialize a library once), `LLVMFuzzerInitialize` handles one-time setup (see Section 114.6).

### A Complete Working Example

```cpp
// fuzz_png.cpp: fuzz a PNG decoder
#include <cstdint>
#include <cstddef>
#include <cstring>
#include <png.h>

// libpng error handling via longjmp
struct PngReadState {
    png_structp png_ptr;
    png_infop   info_ptr;
    bool        error;
};

static void png_error_fn(png_structp png_ptr, png_const_charp) {
    PngReadState *st = (PngReadState *)png_get_error_ptr(png_ptr);
    st->error = true;
    longjmp(png_jmpbuf(png_ptr), 1);
}

struct MemReader { const uint8_t *data; size_t size; size_t pos; };
static void mem_read_fn(png_structp png_ptr, png_bytep out, png_size_t count) {
    MemReader *r = (MemReader *)png_get_io_ptr(png_ptr);
    size_t avail = r->size - r->pos;
    if (count > avail) count = avail;
    memcpy(out, r->data + r->pos, count);
    r->pos += count;
}

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
    PngReadState st = {};
    MemReader reader = {Data, Size, 0};

    st.png_ptr = png_create_read_struct(PNG_LIBPNG_VER_STRING,
                                         &st, png_error_fn, nullptr);
    if (!st.png_ptr) return 0;
    st.info_ptr = png_create_info_struct(st.png_ptr);
    if (!st.info_ptr) {
        png_destroy_read_struct(&st.png_ptr, nullptr, nullptr);
        return 0;
    }
    if (setjmp(png_jmpbuf(st.png_ptr))) {
        png_destroy_read_struct(&st.png_ptr, &st.info_ptr, nullptr);
        return 0;
    }
    png_set_read_fn(st.png_ptr, &reader, mem_read_fn);
    png_read_info(st.png_ptr, st.info_ptr);
    // Read rows:
    png_uint_32 width = png_get_image_width(st.png_ptr, st.info_ptr);
    png_uint_32 height = png_get_image_height(st.png_ptr, st.info_ptr);
    if (width > 0 && height > 0 && width < 10000 && height < 10000) {
        png_read_image(st.png_ptr, nullptr);  // will longjmp on error
    }
    png_destroy_read_struct(&st.png_ptr, &st.info_ptr, nullptr);
    return 0;
}
// Compile:
// clang-22 -fsanitize=fuzzer,address -g -O1 fuzz_png.cpp -lpng -o fuzz_png
```

### Compilation and Invocation

```bash
# Compile fuzz target:
clang-22 -fsanitize=fuzzer,address -g -O1 fuzz_target.cpp -o fuzz_target

# Run with a seed corpus:
mkdir corpus seeds
# Copy some valid PNG files to seeds/:
./fuzz_target seeds/ corpus/ -jobs=4 -workers=4

# Run for a time limit:
./fuzz_target corpus/ -max_total_time=3600

# Run on a single input (for crash reproduction):
./fuzz_target crash-7a3f9b2e

# Minimize a crash:
./fuzz_target -minimize_crash=1 -runs=10000 crash-7a3f9b2e
```

---

## 114.3 SanitizerCoverage Instrumentation

### How SanitizerCoverage Works

`-fsanitize=fuzzer` implies `-fsanitize-coverage=trace-pc-guard`, which is LLVM's SanitizerCoverage (SanCov) instrumentation. SanCov instruments every edge in the program's control flow graph with a callback:

```llvm
; LLVM IR generated by -fsanitize-coverage=trace-pc-guard:
; At each basic block entry (edge guard):
call void @__sanitizer_cov_trace_pc_guard(i32* @__sancov_guard_N)
```

Where `__sancov_guard_N` is a static 32-bit counter, one per CFG edge.

The runtime function `__sanitizer_cov_trace_pc_guard` in libFuzzer:

```cpp
// compiler-rt/lib/fuzzer/FuzzerTracePC.cpp:
extern "C" void __sanitizer_cov_trace_pc_guard(uint32_t *Guard) {
    if (!*Guard) return;  // already covered, fast path
    *Guard = 0;           // mark as seen
    fuzzer::TPC.HandleCallerCallee(GET_CALLER_PC(), GET_CALLEE_PC());
}
```

The `if (!*Guard)` check is the hot path: once an edge is covered, subsequent calls return immediately after a single load-and-branch. This makes the coverage instrumentation lightweight for already-covered code.

### Coverage Granularities

SanitizerCoverage offers multiple levels of instrumentation:

| Flag | Granularity | Description |
|------|------------|-------------|
| `-fsanitize-coverage=edge` | CFG edges | Default; one guard per edge |
| `-fsanitize-coverage=bb` | Basic blocks | One guard per basic block |
| `-fsanitize-coverage=func` | Functions | One guard per function |
| `-fsanitize-coverage=trace-pc` | Every BB | Uses raw PC, no guard array |
| `-fsanitize-coverage=trace-pc-guard` | Every BB | Guard array version (libFuzzer default) |

For libFuzzer, `trace-pc-guard` is the default. It provides edge-level feedback with an efficient implementation.

### Comparison Coverage (`trace-cmp`)

`-fsanitize-coverage=trace-cmp` instruments comparison operations to give the fuzzer hints about "interesting" byte values:

```cpp
// For: if (x == 0xDEADBEEF)
// Clang adds:
__sanitizer_cov_trace_cmp4(x, 0xDEADBEEF);
if (x == 0xDEADBEEF) { ... }
```

LibFuzzer records compared constants in a "value profile" table. When mutating inputs, it can try to make `x` equal `0xDEADBEEF` — enabling it to penetrate deep conditional checks that would require astronomical luck to satisfy randomly.

```bash
# Enable comparison tracing:
clang-22 -fsanitize=fuzzer,address \
         -fsanitize-coverage=trace-pc-guard,trace-cmp \
         -g target.cpp -o fuzz_target
```

### PC Table

`-fsanitize-coverage=pc-table` adds a static table of (PC, flags) pairs for every instrumented location. LibFuzzer uses this for more accurate coverage symbolization and for the `-print_coverage` feature:

```bash
./fuzz_target corpus/ -print_coverage=1 2>&1 | head -20
# Outputs: COVERED/NOT_COVERED per source function
```

### Inline 8-bit Counters

`-fsanitize-coverage=inline-8bit-counters` replaces the guard-based scheme with 8-bit saturating counters, giving hit-count information rather than just coverage:

```cpp
// Instead of: __sanitizer_cov_trace_pc_guard(&guard_N)
// Clang emits inline code:
++__sancov_cntrs_N;  // 8-bit saturating increment
```

LibFuzzer uses hit counts to distinguish "edge covered once" from "edge covered many times" as separate corpus signals — useful for loop-intensive code.

---

## 114.4 The libFuzzer Engine

### Source Organization

LibFuzzer lives in [`compiler-rt/lib/fuzzer/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/lib/fuzzer/FuzzerLoop.cpp). Key files:

| File | Role |
|------|------|
| `FuzzerDriver.cpp` | Entry point, argument parsing, main loop |
| `FuzzerLoop.cpp` | Core fuzzing loop, corpus management |
| `FuzzerMutate.cpp` | Mutation engine |
| `FuzzerCorpus.cpp` | Corpus data structures |
| `FuzzerTracePC.cpp` | SanitizerCoverage callback implementations |
| `FuzzerSHA1.cpp` | Input hashing (for dedup) |
| `FuzzerIO.cpp` | File I/O for corpus |

### The Fuzzing Loop

The main fuzzing loop in `FuzzerLoop.cpp`:

```
FuzzerDriver::Loop():
  1. Load seed corpus from disk
  2. For each seed: run it, record coverage
  3. Loop:
     a. Pick an input from corpus (weighted by fitness)
     b. Apply mutations (MutationDispatcher)
     c. Run mutated input through LLVMFuzzerTestOneInput()
     d. Check for crash/timeout/OOM
     e. If new coverage gained: add to corpus, save to disk
     f. Update counters, print stats periodically
```

Coverage tracking uses the `__sancov_guard` array. After each run, `TracePC::UpdateObservedPCs()` scans the array for newly-zero'd guards (newly covered edges) and records them.

### The Mutation Engine

`MutationDispatcher` applies one of 20+ mutation strategies, selected randomly:

```cpp
// compiler-rt/lib/fuzzer/FuzzerMutate.cpp (strategy list):
{&MutationDispatcher::Mutate_EraseBytes,           "EraseBytes"},
{&MutationDispatcher::Mutate_InsertByte,           "InsertByte"},
{&MutationDispatcher::Mutate_InsertRepeatedBytes,  "InsertRepeatedBytes"},
{&MutationDispatcher::Mutate_ChangeByte,           "ChangeByte"},
{&MutationDispatcher::Mutate_ChangeBit,            "ChangeBit"},
{&MutationDispatcher::Mutate_ShuffleBytes,         "ShuffleBytes"},
{&MutationDispatcher::Mutate_ChangeASCIIInteger,   "ChangeASCIIInteger"},
{&MutationDispatcher::Mutate_ChangeBinaryInteger,  "ChangeBinaryInteger"},
{&MutationDispatcher::Mutate_CopyPart,             "CopyPart"},
{&MutationDispatcher::Mutate_CrossOver,            "CrossOver"},
{&MutationDispatcher::Mutate_AddWordFromManualDictionary, "ManualDict"},
{&MutationDispatcher::Mutate_AddWordFromPersistentAutoDictionary, "AutoDict"},
// ... more strategies
```

**Crossover**: takes two inputs from the corpus and splices them together — effective for finding interactions between different valid input fragments.

**Dictionary**: `ManualDict` uses tokens from `-dict=dict.txt`; `AutoDict` automatically collects constants from comparison operations (`trace-cmp`) and tries to insert them.

### Corpus Fitness

Inputs in the corpus are scored by:
- Number of new edges they cover (primary metric)
- How recently they were added (recency bonus)
- Execution time (shorter = slightly preferred, to keep iteration speed high)

The corpus is bounded by `-max_corpus_size` (default: unlimited). Corpus bloat (too many redundant inputs) slows the fuzzer by diluting selection of useful inputs; `-merge=1` corpus minimization addresses this.

### Important Runtime Options

```bash
# Run with key options:
./fuzz_target corpus/ seeds/ \
  -max_len=4096          \   # max input size (bytes)
  -timeout=10            \   # per-input timeout (seconds)
  -rss_limit_mb=2048     \   # memory limit
  -dict=magic.dict       \   # dictionary file
  -runs=1000000          \   # fixed number of runs then exit
  -print_final_stats=1   \   # print stats on exit
  -jobs=4 -workers=4         # parallel jobs (4 processes, 4 CPU cores)
```

---

## 114.5 Sanitizer Integration

### ASan + libFuzzer

The most common combination is `-fsanitize=fuzzer,address`. ASan detects:
- Heap buffer overflows (including off-by-one in `malloc`'d buffers)
- Stack buffer overflows
- Use-after-free
- Double-free
- Use-after-scope
- Global buffer overflows

When ASan detects a violation, it calls `__asan_report_error()` which prints a report and calls `abort()`. LibFuzzer intercepts `abort()` via `signal(SIGABRT, ...)` and reports the input as a crash:

```
artifact_prefix='./'; Test unit written to ./crash-7a3f9b2ec3...
==2341==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000000411
READ of size 1 at 0x602000000411 thread T0
    #0 0x404e12 in parse_header /src/target.cpp:123
    #1 0x40512f in LLVMFuzzerTestOneInput /src/fuzz_target.cpp:45
    ...
```

### UBSan + libFuzzer

```bash
clang-22 -fsanitize=fuzzer,undefined -g target.cpp -o fuzz_target
```

UBSan's checks (integer overflow, shift violations, null pointer dereference, etc.) trigger `__ubsan_handle_*` which by default calls `abort()`. The `-fsanitize-undefined-trap-on-error` flag is NOT recommended for fuzzing — it traps rather than reporting, making crash analysis harder.

### MSan + libFuzzer

MSan is the most difficult sanitizer to combine with libFuzzer because MSan requires that the *entire* program, including all libraries, be compiled with MSan instrumentation. Uninstrumented code propagates "initialized" taint incorrectly:

```bash
# MSan build requires all dependencies to be MSan-instrumented:
# Use a pre-built MSan-instrumented sysroot or build everything from source.
clang-22 -fsanitize=fuzzer,memory \
         -fsanitize-memory-track-origins=2 \
         -g target.cpp -o fuzz_target_msan
```

### OOM and Timeout Detection

LibFuzzer has built-in mechanisms for detecting resource exhaustion that would otherwise cause the fuzzer to hang:

**Timeout**: `-timeout=N` sets a per-input wall-clock timeout. If `LLVMFuzzerTestOneInput` runs longer than N seconds, libFuzzer kills the process and saves the input as `timeout-<sha>`:

```cpp
// FuzzerLoop.cpp: alarm-based timeout
void Fuzzer::AlarmCallback() {
    if (Options.UnitTimeoutSec > 0 && TimeOfMoreRecentStartOfAnExecution + 
        Options.UnitTimeoutSec < system_clock::now()) {
        TPC.DumpCoverage();
        Printf("ALARM: working on the last Unit for %zd seconds\n", ...);
        DumpCurrentUnit("timeout-");
        _Exit(1);
    }
}
```

**RSS limit**: `-rss_limit_mb=N` checks RSS after each run via `/proc/self/status`. If exceeded, the input is saved as `oom-<sha>` and fuzzing halts.

### Signal Handling

LibFuzzer installs signal handlers for `SIGSEGV`, `SIGBUS`, `SIGFPE`, `SIGILL`, `SIGABRT`, and others. When these signals arrive, the current input is saved as a crash artifact and the stack trace is printed.

This means that even bugs detected by hardware (null pointer dereference → SIGSEGV, divide by zero → SIGFPE) are correctly attributed to the fuzzer input that triggered them.

---

## 114.6 Structured and Grammar-Based Fuzzing

### FuzzedDataProvider

Raw byte mutation works well for binary formats but poorly for structured formats (JSON, XML, source code). The `FuzzedDataProvider` utility class provides type-safe accessors into the raw byte stream, making it easy to construct typed values:

```cpp
#include <fuzzer/FuzzedDataProvider.h>

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
    FuzzedDataProvider FDP(Data, Size);

    // Consume typed values:
    int32_t  a     = FDP.ConsumeIntegral<int32_t>();
    uint8_t  b     = FDP.ConsumeIntegralInRange<uint8_t>(0, 127);
    float    f     = FDP.ConsumeFloatingPoint<float>();
    bool     flag  = FDP.ConsumeBool();
    std::string s  = FDP.ConsumeRandomLengthString(/*max_length=*/256);
    
    // Consume a vector:
    auto bytes = FDP.ConsumeBytes<uint8_t>(16);
    
    // Consume an enum:
    auto op = FDP.ConsumeEnum<MyOpcode>();

    // Call the target with structured inputs:
    my_api_function(a, b, f, flag, s, bytes.data(), bytes.size(), op);
    return 0;
}
```

`FuzzedDataProvider` is header-only; include it from the Clang installation or copy from `compiler-rt/include/fuzzer/FuzzedDataProvider.h`.

The key property: `FuzzedDataProvider` consumes bytes from the input sequentially. LibFuzzer's mutations (bit flips, byte insertions, etc.) on the raw bytes translate naturally into mutations of the typed values.

### LLVMFuzzerInitialize

For targets that need one-time initialization (e.g., initializing a library that reads global config), use `LLVMFuzzerInitialize`:

```cpp
extern "C" int LLVMFuzzerInitialize(int *argc, char ***argv) {
    // Called once before any LLVMFuzzerTestOneInput call.
    // Can modify argc/argv to process fuzzer-specific flags.
    MyLibrary_Initialize();
    return 0;
}
```

### LLVMFuzzerCustomMutator

For targets where the default mutation strategies are ineffective, implement a custom mutator:

```cpp
extern "C" size_t LLVMFuzzerCustomMutator(uint8_t *Data, size_t Size,
                                           size_t MaxSize, unsigned int Seed) {
    // Mutate Data[0..Size-1] in-place, returning new size.
    // Use Seed for deterministic mutation.
    MyStructuredMutator(Data, Size, MaxSize, Seed);
    return new_size;
}
```

Custom mutators are most effective when combined with a "cross-over" mutator:

```cpp
extern "C" size_t LLVMFuzzerCustomCrossOver(const uint8_t *Data1, size_t Size1,
                                             const uint8_t *Data2, size_t Size2,
                                             uint8_t *Out, size_t MaxOutSize,
                                             unsigned int Seed) {
    return MyCrossOver(Data1, Size1, Data2, Size2, Out, MaxOutSize, Seed);
}
```

### libprotobuf-mutator

For protocol buffer-based targets, `libprotobuf-mutator` provides mutation-aware fuzzing:

```cpp
#include <libprotobuf-mutator/src/libfuzzer/libfuzzer_macro.h>
#include "my_request.pb.h"

DEFINE_PROTO_FUZZER(const MyRequest &request) {
    // 'request' is a valid, mutated protobuf message
    process_request(request);
}
```

The `DEFINE_PROTO_FUZZER` macro handles the conversion from raw bytes to a protobuf message and back, and installs a custom mutator that preserves protobuf structure during mutation.

### Dictionary Files

A dictionary provides known-good tokens that the fuzzer should try inserting:

```
# http.dict — tokens for HTTP fuzzing
"GET "
"POST "
"HTTP/1.1"
"Content-Length: "
"Transfer-Encoding: chunked"
"\r\n\r\n"

# Usage:
./fuzz_http corpus/ -dict=http.dict
```

LibFuzzer's `AutoDict` feature (from `trace-cmp`) automatically discovers comparison constants during fuzzing and adds them to an internal dictionary, often making manual dictionaries unnecessary for well-instrumented targets.

---

## 114.7 OSS-Fuzz Integration

### What is OSS-Fuzz?

[OSS-Fuzz](https://github.com/google/oss-fuzz) is Google's free continuous fuzzing service for critical open-source software. It runs fuzz targets 24/7 on Google's infrastructure, manages corpora, deduplicates crashes, files bug reports, and verifies fixes. Projects integrated with OSS-Fuzz include OpenSSL, curl, FFmpeg, SQLite, Chromium's PDFium, and hundreds more.

The ClusterFuzz backend powers OSS-Fuzz. It orchestrates:
- Multiple fuzzer instances per target
- Corpus synchronization between instances
- Automated crash triage and deduplication (by stack trace similarity)
- Automated minimization
- Regression tracking

### Integration Structure

An OSS-Fuzz integration requires:

```
oss-fuzz/projects/myproject/
├── Dockerfile           # Build environment
├── build.sh             # Build script
└── project.yaml         # Project metadata
```

**Dockerfile** sets up the build environment:

```dockerfile
FROM gcr.io/oss-fuzz-base/base-builder
RUN apt-get update && apt-get install -y libpng-dev
COPY build.sh $SRC/
COPY *.cpp $SRC/
WORKDIR $SRC/myproject
```

**build.sh** compiles the fuzz targets using OSS-Fuzz's environment variables:

```bash
#!/bin/bash -eu
# OSS-Fuzz provides:
# $CC, $CXX, $CFLAGS, $CXXFLAGS (includes -fsanitize=... per build type)
# $LIB_FUZZING_ENGINE (e.g., -fsanitize=fuzzer)
# $OUT (output directory for fuzz target binaries)
# $SRC (source directory)

# Build the library under test:
cmake -DCMAKE_C_COMPILER="$CC" \
      -DCMAKE_CXX_COMPILER="$CXX" \
      -DCMAKE_C_FLAGS="$CFLAGS" \
      -DCMAKE_CXX_FLAGS="$CXXFLAGS" \
      -DBUILD_SHARED_LIBS=OFF \
      ../
make -j$(nproc)

# Build fuzz targets:
$CXX $CXXFLAGS -std=c++17 \
    $SRC/fuzz_target.cpp \
    -I../include \
    ./libmylib.a \
    $LIB_FUZZING_ENGINE \
    -o $OUT/fuzz_target

# Copy seed corpus (optional):
cp $SRC/seeds.zip $OUT/fuzz_target_seed_corpus.zip
```

OSS-Fuzz builds each target in multiple configurations:
- ASan + libFuzzer
- MSan + libFuzzer
- UBSan + libFuzzer
- Coverage-only (for coverage reports)

### LLVM's Own Fuzz Targets on OSS-Fuzz

LLVM itself has fuzz targets in [`llvm/tools/llvm-isel-fuzzer/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/tools/llvm-isel-fuzzer/llvm-isel-fuzzer.cpp) and `compiler-rt/test/fuzzer/`:

- `clang-fuzzer`: fuzzes Clang's compiler frontend with random C/C++ code
- `llvm-opt-fuzzer`: fuzzes the LLVM optimizer with random IR
- `llvm-mc-assemble-fuzzer`: fuzzes the MC assembler with random assembly
- `llvm-dwarfdump-fuzzer`: fuzzes the DWARF parser

These targets have found hundreds of bugs in LLVM itself, including compiler crashes on valid and invalid inputs.

---

## 114.8 Corpus Management and Triage

### Crash Reproduction

When libFuzzer finds a crash, it saves the input as `crash-<sha1hash>`. Reproducing the crash requires only:

```bash
./fuzz_target crash-7a3f9b2ec3f840d9a3de0a17b8a2e49f
# Output: full sanitizer report + stack trace
```

For crash reproduction in a different environment (e.g., without ASan, just to debug in gdb):

```bash
# Compile a non-fuzzer, non-sanitizer version for GDB:
clang-22 -g -O0 target.cpp driver.cpp -o target_debug

# Create a simple driver that reads from file:
// driver.cpp:
int main(int argc, char **argv) {
    // Read crash-... file and pass to target logic
}

gdb --args ./target_debug crash-7a3f9b2ec3f840d9a3de0a17b8a2e49f
```

### Crash Minimization

Large crash inputs are difficult to analyze. LibFuzzer's crash minimization reduces the input to the smallest form that still triggers the crash:

```bash
./fuzz_target -minimize_crash=1 -runs=100000 crash-7a3f9b2ec3
# Writes: minimized-from-7a3f9b2ec3 (smaller file that still crashes)
```

Minimization works by repeatedly removing bytes or sections and re-running; if the crash persists, the reduced input is kept.

For crashes found by AFL or other fuzzers, `creduce` or `cvise` can minimize C source code inputs.

### Corpus Minimization

Over time, a corpus may grow to contain thousands of inputs that cover largely the same code paths. Corpus minimization removes inputs that don't contribute unique coverage:

```bash
# Merge corpus_big/ into corpus_min/, keeping only unique-coverage inputs:
./fuzz_target -merge=1 corpus_min/ corpus_big/

# The -merge=1 mode:
# 1. Runs all inputs in corpus_min/ (the base, kept as-is)
# 2. Runs all inputs in corpus_big/ 
# 3. Adds to corpus_min/ only inputs from corpus_big/ that add new coverage
```

Regular corpus minimization keeps the fuzzing loop efficient and reduces CI storage costs.

### Coverage Reporting

```bash
# Build with source-based coverage (for human-readable reports):
clang-22 -fsanitize=fuzzer \
         -fprofile-instr-generate -fcoverage-mapping \
         -g target.cpp -o fuzz_target_cov

# Run corpus:
LLVM_PROFILE_FILE="coverage-%p.profraw" \
  ./fuzz_target_cov corpus/ -runs=0  # -runs=0: process corpus only, don't fuzz

# Merge profiles:
/usr/lib/llvm-22/bin/llvm-profdata merge \
  coverage-*.profraw -o coverage.profdata

# Generate HTML report:
/usr/lib/llvm-22/bin/llvm-cov show \
  ./fuzz_target_cov \
  -instr-profile=coverage.profdata \
  --format=html \
  -output-dir=coverage_report/

# Open coverage_report/index.html to see which lines the fuzzer covers
```

This workflow (Chapter 115 covers it in depth) answers: "Which code has the fuzzer explored?" and "What do I need to add to the corpus to improve coverage?"

### Triage and Deduplication

When a fuzzer finds many crashes, deduplication is essential to avoid filing the same bug multiple times. Strategies:

1. **Stack hash deduplication**: group crashes by the hash of the top N frames of the stack trace. Crashes with identical top-5 stack frames are assumed to be the same bug.

2. **ASan error type**: distinguish `heap-buffer-overflow`, `use-after-free`, `stack-buffer-overflow` — different error types at the same address are distinct bugs.

3. **Coverage-based deduplication**: two inputs that trigger the exact same set of new coverage edges are likely triggering the same bug.

ClusterFuzz automates this deduplication. For local triage:

```bash
# Quick dedup by ASan report type + top frame:
for crash in crash-*; do
  ./fuzz_target $crash 2>&1 | grep "ERROR: AddressSanitizer" | head -1
  ./fuzz_target $crash 2>&1 | grep "#0 " | head -1
  echo "---"
done | sort -u
```

---

## Chapter Summary

- **LibFuzzer** is LLVM's in-process coverage-guided fuzzer. It calls `LLVMFuzzerTestOneInput` in a tight loop, using SanitizerCoverage feedback to guide mutation toward unexplored code paths.

- **SanitizerCoverage** (`-fsanitize-coverage=trace-pc-guard`) instruments every CFG edge with a guard counter. `trace-cmp` additionally captures comparison constants to enable magic-byte penetration.

- **The mutation engine** applies 20+ strategies including byte flips, integer changes, crossover, and dictionary insertion. `trace-cmp`-derived constants populate an automatic dictionary for comparison-guided mutation.

- **Sanitizer integration**: combine `-fsanitize=fuzzer,address` (ASan + libFuzzer) for the most productive fuzzing. UBSan adds undefined behavior detection; MSan catches uninitialized reads but requires fully-instrumented builds.

- **FuzzedDataProvider** enables structured fuzzing by providing typed accessors into the raw byte stream. Custom mutators (`LLVMFuzzerCustomMutator`) and `libprotobuf-mutator` handle grammar-based targets.

- **OSS-Fuzz** provides free continuous fuzzing for open-source projects. Integration requires a `Dockerfile` + `build.sh`; ClusterFuzz handles corpus management, triage, and bug tracking.

- **Corpus management** — minimization (`-merge=1`), crash minimization (`-minimize_crash=1`), and coverage reporting — keeps the fuzzer efficient and analysis tractable over long campaigns.

- LLVM itself is continuously fuzzed on OSS-Fuzz via targets in `llvm/tools/` and `compiler-rt/test/fuzzer/`.


---

@copyright jreuben11
