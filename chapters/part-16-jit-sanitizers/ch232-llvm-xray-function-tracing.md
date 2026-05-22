# Chapter 232 — LLVM XRay: Low-Overhead Function Tracing

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

Profiling production systems involves a fundamental trade-off: instruments with low overhead run in production but miss details; instruments with high accuracy slow the system too much to be practical. LLVM XRay resolves this by separating instrumentation from activation — compiler-inserted NOP sleds cost nothing when disabled and are patched atomically at runtime when tracing is needed. The result is a function-level tracer with sub-2% overhead when active, zero overhead when inactive, and nanosecond-resolution timestamps that distinguish microsecond-level latencies. This chapter covers the NOP-sled mechanism, the three runtime modes, the `llvm-xray` analysis tools, and patterns for production deployment.

---

## 232.1 Design Goals and Comparison with Other Tools

### Design Goals

| Goal | Mechanism |
|------|-----------|
| Zero overhead when disabled | 5-byte NOP sled costs one fetch cycle; no branch taken |
| ~2% overhead when enabled | One `call` instruction per function entry/exit |
| Runtime enable/disable | `__xray_patch()` / `__xray_unpatch()` without restart |
| Nanosecond timestamps | RDTSC / RDTSCP per event |
| Signal safety | Patch operation uses lock-prefixed writes, no locks held |

### Comparison with Other Profiling Approaches

| Tool | Mechanism | Overhead (active) | Overhead (inactive) | Granularity |
|------|-----------|--------------------|---------------------|-------------|
| **XRay** | Compiler NOP sled | ~2% | ~0% | Per function entry/exit |
| **perf** | Hardware PMU sampling | ~0.1% | 0% | Statistical (sampled) |
| **gprof** | `-pg` instrumentation | ~20-30% | N/A (always on) | Per function (statistical timing) |
| **Valgrind/Callgrind** | Binary instrumentation | 10-100× | N/A | Per instruction |
| **eBPF uprobe** | Kernel trap on symbol | ~3-5% | ~0% | Per probe site |
| **Tracy** | Manual annotations | ~1-3% | ~0% | User-defined spans |
| **Intel VTune** | Hardware counters | ~2-5% | ~0% | Per instruction (sampled) |

XRay occupies a unique position: it has perf's near-zero inactive overhead, gprof's complete function coverage, and is fully contained in userspace (no kernel involvement when enabled).

---

## 232.2 NOP Sled Instrumentation

### Compiler-Side: Inserting the Sled

The `-fxray-instrument` flag instructs Clang to insert a NOP sled at each instrumented function's entry and exit:

```bash
clang++ -fxray-instrument -fxray-instruction-threshold=1 \
        -O2 foo.cpp -o foo -lxray
```

On x86-64, Clang inserts exactly 5 bytes of NOPs at the function prologue:

```asm
; Before patching (NOPs)
foo:
  nop
  nop
  nop
  nop
  nop
  ; actual function body begins here
  push rbp
  mov rbp, rsp
  ...
```

The 5-byte sled is large enough to hold a `call rel32` instruction (1-byte opcode + 4-byte relative offset), enabling atomic in-place patching on all x86-64 processors.

AArch64 uses a 2-instruction sled (8 bytes): two NOP instructions that can be patched to a `bl` (branch-and-link) to the XRay handler.

### Runtime Patching: `__xray_patch()`

When XRay is activated at runtime:

```cpp
#include "xray/xray_interface.h"

// Activate all XRay instrumentation sites
__xray_patch();

// ... run workload ...

// Deactivate all XRay instrumentation sites
__xray_unpatch();
```

The patch operation:
1. Locates all NOP sleds via the `xray_instr_map` ELF section
2. For each sled, computes the relative offset to the XRay handler
3. Writes the `call rel32` encoding (5 bytes) using a lock-prefixed write to guarantee atomicity

```asm
; After patching (call to XRay handler)
foo:
  call __xray_FunctionEntry   ; 5 bytes, atomically written
  push rbp
  mov rbp, rsp
  ...
  call __xray_FunctionExit    ; at return site (2 bytes on x86-64 ret)
```

The patch is signal-safe: the 5-byte write using `lock cmpxchg` ensures no thread can observe a torn instruction.

### ELF Sections

XRay creates two ELF sections in the instrumented binary:

**`xray_instr_map`**: maps each instrumented site (function entry or exit) to its patch address. Iterated by `__xray_patch()`:

```
Entry 0: {funcid=1, kind=ENTRY, address=0x401020, ...}
Entry 1: {funcid=1, kind=EXIT,  address=0x401080, ...}
Entry 2: {funcid=2, kind=ENTRY, address=0x4010A0, ...}
...
```

**`xray_fn_idx`**: index mapping function IDs to their `xray_instr_map` entries. Enables per-function patching via `__xray_patch_function(funcid)`.

Source: `compiler-rt/lib/xray/xray_interface.cpp`, `xray_init.cpp`.

---

## 232.3 XRay Modes

XRay supports three runtime modes, selected before or at activation:

### Basic Mode

Records every function entry and exit to a per-thread in-memory log:

```cpp
#include "xray/xray_log_interface.h"

XRayLogInitStatus status = __xray_log_init_mode("basic", "xray-log.xray");
__xray_patch();

// ... run workload ...

__xray_unpatch();
__xray_log_finalize();
__xray_log_flushLog();  // write to disk
```

Each log entry is an `XRayRecord`:

```cpp
struct XRayRecord {
  uint16_t RecordType;   // NORMAL (0) or ARG1 (1)
  uint8_t  CPU;          // CPU core
  uint8_t  Type;         // ENTRY (0), EXIT (1), TAIL (2)
  int32_t  FuncId;       // function ID (matches xray_fn_idx)
  int64_t  TSC;          // RDTSC timestamp
  uint32_t TId;          // thread ID
  uint32_t PId;          // process ID
};
```

### Flight Recorder Mode

Maintains a circular ring buffer per thread. Only the most recent N records are retained — older records are overwritten. Ideal for post-hoc analysis of anomalous requests (long-tail latency investigation):

```cpp
// Configure flight recorder: 256 MB circular buffer
XRayLogInitStatus status = __xray_log_init_mode(
    "xray-fdr", "buffer_size=268435456");
__xray_patch();  // now always active in background

// Later, after detecting an anomaly:
__xray_log_finalize();
__xray_log_flushLog();  // dump only last N records from ring buffer
```

The flight recorder is the most common production deployment pattern: XRay is always armed, incurring ~2% overhead, but logs are only captured and written on anomaly detection.

### Custom Handler Mode

Installs a user-provided handler called on every XRay event:

```cpp
void my_handler(int32_t funcid, XRayEntryType type) {
    // called synchronously on every entry/exit
    auto now = __rdtsc();
    if (type == XRayEntryType::ENTRY)
        thread_local_stack.push({funcid, now});
    else if (type == XRayEntryType::EXIT) {
        auto [fid, start] = thread_local_stack.top();
        thread_local_stack.pop();
        record_latency(fid, now - start);
    }
}

__xray_set_handler(my_handler);
__xray_patch();
```

Custom handlers must be fast (they execute on every function entry/exit on the hot path) and must not call XRay-instrumented functions recursively.

---

## 232.4 Custom Handlers and Flame Graph Collection

### Building a Flame Graph Collector

A common use of custom handlers is collecting flame-graph-compatible stacks:

```cpp
#include "xray/xray_interface.h"
#include <atomic>
#include <thread>
#include <vector>

struct CallRecord { int32_t funcid; uint64_t tsc_enter; };

thread_local std::vector<CallRecord> call_stack;
thread_local std::vector<std::string> completed_stacks;

void flame_graph_handler(int32_t funcid, XRayEntryType type) {
    if (type == XRayEntryType::ENTRY) {
        call_stack.push_back({funcid, __rdtsc()});
    } else {  // EXIT or TAIL
        if (call_stack.empty()) return;
        auto [fid, start] = call_stack.back();
        call_stack.pop_back();
        uint64_t duration = __rdtsc() - start;
        // Build stack string: "func1;func2;func3 duration"
        std::string stack_str;
        for (auto& e : call_stack)
            stack_str += lookup_func_name(e.funcid) + ";";
        stack_str += lookup_func_name(fid);
        completed_stacks.push_back(stack_str + " " + std::to_string(duration));
    }
}
```

### Argument Logging

`-fxray-instrument-with-types` enables logging the first argument of each function call:

```cpp
void my_arg_handler(int32_t funcid, XRayEntryType type, uint64_t arg1) {
    // arg1 is the first integer/pointer argument
}
__xray_set_handler_arg1(my_arg_handler);
```

This is useful for tracking which specific objects or IDs are flowing through hot code paths.

### Handler Constraints

Custom handlers must:
- Execute in O(1) time (no allocation, no blocking)
- Not call XRay-instrumented functions (infinite recursion)
- Be reentrant (multiple threads call the handler concurrently)
- Not hold locks (the handler is called in a signal-like context)

---

## 232.5 The `llvm-xray` Tool Suite

XRay logs are in a binary format. The `llvm-xray` tool converts and analyzes them.

Source: `llvm/tools/llvm-xray/`.

### `llvm-xray account`

Produces per-function call counts and timing statistics:

```bash
llvm-xray account xray-log.xray -instr_map=./foo \
    -sort=funcid -sortorder=dsc -output=text

# Output:
# funcid  count     min(ns)  median(ns)  max(ns)   sum(ns)
#   1      10234     120      340         98230     4523780
#   2      1023      540      1200        45000     1392840
```

Useful for identifying which functions are called most frequently and which have high-latency outliers.

### `llvm-xray convert`

Converts binary XRay logs to other formats:

```bash
# Chrome trace format (for chrome://tracing)
llvm-xray convert xray-log.xray -instr_map=./foo \
    -output-format=trace_event -output=trace.json

# Perf script format (for flamegraph.pl)
llvm-xray convert xray-log.xray -instr_map=./foo \
    -output-format=perf-script -output=perf.script

# Run through flamegraph tools
stackcollapse-perf.pl perf.script | flamegraph.pl > flame.svg
```

The Chrome trace format produces a timeline viewable in `chrome://tracing` or Perfetto UI, showing function call lifetimes as colored spans on a per-thread timeline.

### `llvm-xray stack`

Reconstructs call stacks from the log and outputs folded stack format:

```bash
llvm-xray stack xray-log.xray -instr_map=./foo \
    -aggregation-type=time -output=stacks.txt
```

Produces lines like `main;process;compute 12345678` suitable for flamegraph rendering.

### `llvm-xray graph`

Generates a DOT call graph with per-edge timing:

```bash
llvm-xray graph xray-log.xray -instr_map=./foo -output=cg.dot
dot -Tsvg cg.dot -o callgraph.svg
```

---

## 232.6 Compiler and LTO Integration

### Instrumentation Threshold

`-fxray-instruction-threshold=N` controls which functions are instrumented. Only functions with at least N instructions are instrumented:

```bash
# Instrument only functions with ≥ 50 instructions (reduces overhead)
clang++ -fxray-instrument -fxray-instruction-threshold=50 foo.cpp -o foo
```

The default threshold is 1 (instrument all functions). Setting a higher threshold reduces instrumentation overhead for trivial functions at the cost of missing their contribution in traces.

### Mutual Exclusivity with PGO

XRay is incompatible with `-fprofile-generate` (PGO instrumentation). Both insert function-entry instrumentation and will conflict at link time. Use one or the other:

```bash
# PGO build (instrument for profile collection)
clang++ -fprofile-generate foo.cpp -o foo

# XRay build (instrument for tracing)
clang++ -fxray-instrument foo.cpp -o foo -lxray

# Not both simultaneously
```

### ThinLTO Interaction

With ThinLTO, XRay instrumentation is applied **after** LTO inlining. This means that when function A is inlined into B, only B's entry/exit point is traced — the inlined A calls are not separately recorded. This is the correct behavior: the user sees B's total latency, which already includes the inlined A cost.

The `xray_fn_idx` and `xray_instr_map` sections are emitted per object file and merged by the linker into the final binary.

### Runtime Library

XRay's runtime is in `compiler-rt/lib/xray/`. The library is linked via `-lxray` (or automatically when using `clang++` with `-fxray-instrument`). The library initializes XRay's internal state at program startup and registers signal handlers for safe shutdown.

---

## 232.7 Selective Tracing

### Attribute-Based Control

```cpp
// Force instrumentation regardless of threshold
[[clang::xray_always_instrument]]
void critical_path() { /* ... */ }

// Suppress instrumentation (e.g., hot inner loop)
[[clang::xray_never_instrument]]
inline void tight_inner_loop() { /* ... */ }

// Log first argument on entry
[[clang::xray_log_args(1)]]
void process_request(int request_id) { /* ... */ }
```

### Per-Function Patching

`__xray_patch_function(funcid)` enables tracing on a single function without activating all sites:

```cpp
// Find function ID by name (requires debug symbols or xray_fn_idx scan)
int32_t funcid = find_xray_funcid("process_request");
__xray_patch_function(funcid);   // trace only this function

// Later
__xray_unpatch_function(funcid);
```

This is the foundation for **sampling-based XRay**: enable a small random subset of function IDs each minute, rotate the set, and collect representative traces with O(1%) overhead rather than O(2%).

### Combining with `-fxray-ignore-loops`

`-fxray-ignore-loops` suppresses instrumentation of functions that are purely loop bodies (detected via loop dominators). This avoids high-frequency instrumentation of iteration bodies while preserving entry/exit recording for the enclosing function.

---

## 232.8 Production Deployment Patterns

### Flight Recorder for Latency Anomaly Detection

The most common production XRay pattern:

1. Build the binary with `-fxray-instrument`
2. At startup, activate flight recorder mode with a 128–256 MB circular buffer
3. Run normally: XRay is always active, incurring ~2% overhead
4. When a request latency exceeds a threshold (e.g., p99 > 100ms), atomically snapshot the ring buffer
5. Write the snapshot to disk for post-hoc analysis
6. Use `llvm-xray convert` and flamegraph.pl to visualize the anomalous call path

This pattern was deployed at Facebook/Meta for production latency investigations.

### Per-Function Sampling Rotation

For lower overhead (< 0.5%):

```cpp
// Every 60 seconds, rotate the set of traced function IDs
void rotate_traced_functions() {
    static std::vector<int32_t> prev_set;
    for (int32_t fid : prev_set) __xray_unpatch_function(fid);

    auto new_set = sample_function_ids(100);  // 100 random functions
    for (int32_t fid : new_set) __xray_patch_function(fid);
    prev_set = new_set;
}
```

### Comparison: XRay vs eBPF Uprobes

| Property | XRay | eBPF Uprobe |
|----------|------|-------------|
| Kernel involvement | None | Required (kernel trap on breakpoint) |
| Overhead when active | ~2% | ~3-5% per probe |
| Number of probe sites | All functions | Selected symbols |
| Custom logic in handler | Yes (userspace C++) | Yes (eBPF programs, limited) |
| Live attachment to running process | No (binary must be built with `-fxray`) | Yes (`perf probe -x` on any binary) |
| Cross-process tracing | No | Yes |

XRay is preferable when building instrumented binaries from source. eBPF uprobes are preferable for live attachment to existing production binaries.

### XRay vs Tracy Profiler

[Tracy](https://github.com/wolfpld/tracy) is a popular frame profiler requiring manual `ZoneScoped` annotations in source code. XRay requires no source changes — the compiler instruments all functions automatically. Tracy provides a richer GUI (timeline, memory view, GPU integration); XRay provides coverage depth and production-safe zero-overhead deactivation.

---

## Chapter Summary

- XRay inserts 5-byte NOP sleds at function entry (x86-64) at compile time; when inactive, these are free; `__xray_patch()` atomically replaces them with `call __xray_FunctionEntry(funcid)` using a lock-prefixed write
- Two ELF sections: `xray_instr_map` (maps function IDs to patch addresses), `xray_fn_idx` (indexed lookup for per-function patching)
- Three modes: basic (log all entries/exits per-thread), flight recorder (circular ring buffer for post-hoc anomaly analysis), custom (user handler called per event)
- `XRayRecord` carries `{type, cpu, tsc, funcid, tid, pid}`; TSC provides nanosecond resolution; custom handlers enable flame graph collection, argument logging, and sampling patterns
- `llvm-xray account` (per-function stats), `convert` (→ Chrome trace JSON / perf script), `stack` (folded stacks for flamegraph), `graph` (DOT call graph)
- `-fxray-instruction-threshold=N` controls which functions are instrumented; mutually exclusive with `-fprofile-generate`; ThinLTO: instrumentation applied post-inlining for consistent function identity
- `[[clang::xray_always_instrument]]` / `[[clang::xray_never_instrument]]` / `[[clang::xray_log_args(1)]]` attributes provide source-level control; `__xray_patch_function(funcid)` enables per-function tracing for sampling-based deployment
- Production pattern: flight recorder mode always active (~2% overhead), snapshot on latency anomaly; vs eBPF uprobes: XRay requires instrumented build but operates entirely in userspace with lower per-call overhead

---

@copyright jreuben11
