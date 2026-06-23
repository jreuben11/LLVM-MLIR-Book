# Chapter 110 — User-Space Sanitizers

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

Sanitizers are the most impactful debugging tools in the modern C++ developer's toolkit. Where traditional debuggers stop at the symptom — the crash, the wrong answer — sanitizers find the root cause: the buffer overflow that happened three stack frames and 200 milliseconds before the access violation, the uninitialized variable that flipped a boolean in a rarely-executed path, the data race that appears only under load. They work by transforming the program at compile time to insert runtime checks, augmented by a runtime library that manages shadow memory and produces detailed error reports. This chapter covers the five major user-space sanitizers shipped with LLVM's `compiler-rt`: AddressSanitizer (ASan), MemorySanitizer (MSan), ThreadSanitizer (TSan), UndefinedBehaviorSanitizer (UBSan), and LeakSanitizer (LSan) — their shadow memory models, instrumentation passes, runtime internals, and practical deployment patterns.

---

## Table of Contents

- [110.1 Sanitizer Architecture](#1101-sanitizer-architecture)
  - [110.1.1 Compile-Time and Runtime Halves](#11011-compile-time-and-runtime-halves)
  - [110.1.2 Shadow Memory](#11012-shadow-memory)
  - [110.1.3 Sanitizer Common Allocator](#11013-sanitizer-common-allocator)
- [110.2 AddressSanitizer (ASan)](#1102-addresssanitizer-asan)
  - [110.2.1 Shadow Memory Encoding](#11021-shadow-memory-encoding)
  - [110.2.2 Heap Allocation and Red Zones](#11022-heap-allocation-and-red-zones)
  - [110.2.3 Stack Instrumentation](#11023-stack-instrumentation)
  - [110.2.4 Global Instrumentation](#11024-global-instrumentation)
  - [110.2.5 Using ASan and Error Reports](#11025-using-asan-and-error-reports)
- [110.3 MemorySanitizer (MSan)](#1103-memorysanitizer-msan)
  - [110.3.1 Shadow Model](#11031-shadow-model)
  - [110.3.2 Taint Propagation](#11032-taint-propagation)
  - [110.3.3 Origin Tracking](#11033-origin-tracking)
  - [110.3.4 Interceptors and Third-Party Libraries](#11034-interceptors-and-third-party-libraries)
- [110.4 ThreadSanitizer (TSan)](#1104-threadsanitizer-tsan)
  - [110.4.1 Data Race Detection](#11041-data-race-detection)
  - [110.4.2 TSan v3](#11042-tsan-v3)
  - [110.4.3 Synchronization Interceptors](#11043-synchronization-interceptors)
  - [110.4.4 Using TSan](#11044-using-tsan)
- [110.5 UndefinedBehaviorSanitizer (UBSan)](#1105-undefinedbehaviorsanitizer-ubsan)
  - [110.5.1 Instrumented Checks](#11051-instrumented-checks)
  - [110.5.2 Instrumentation Pattern](#11052-instrumentation-pattern)
  - [110.5.3 Trap Mode and Minimal Runtime](#11053-trap-mode-and-minimal-runtime)
  - [110.5.4 CFI — Control Flow Integrity](#11054-cfi-control-flow-integrity)
- [110.6 LeakSanitizer (LSan)](#1106-leaksanitizer-lsan)
  - [110.6.1 Algorithm: Conservative GC Scan](#11061-algorithm-conservative-gc-scan)
  - [110.6.2 Integration with ASan](#11062-integration-with-asan)
  - [110.6.3 Standalone LSan](#11063-standalone-lsan)
  - [110.6.4 LSan API](#11064-lsan-api)
- [110.7 Sanitizer Blocklists and Suppressions](#1107-sanitizer-blocklists-and-suppressions)
  - [110.7.1 Compile-Time Ignorelists](#11071-compile-time-ignorelists)
  - [110.7.2 Per-Function Source Attributes](#11072-per-function-source-attributes)
- [110.8 Combining Sanitizers and Performance Overview](#1108-combining-sanitizers-and-performance-overview)
  - [110.8.1 Compatibility Matrix](#11081-compatibility-matrix)
  - [110.8.2 Performance Overhead](#11082-performance-overhead)
  - [110.8.3 Recommended Workflows](#11083-recommended-workflows)
- [DataFlowSanitizer (DFSan)](#dataflowsanitizer-dfsan)
  - [How Taint Tracking Works](#how-taint-tracking-works)
  - [Shadow Memory Architecture](#shadow-memory-architecture)
  - [Building with DFSan](#building-with-dfsan)
  - [The DFSan API](#the-dfsan-api)
  - [Origin Tracking](#origin-tracking)
  - [Use Cases](#use-cases)
  - [Integration with ASan](#integration-with-asan)
  - [Limitations](#limitations)
- [Chapter 110 Summary](#chapter-110-summary)

---

## 110.1 Sanitizer Architecture

### 110.1.1 Compile-Time and Runtime Halves

Every sanitizer has two components that must work together:

**Compile-time instrumentation**: A Clang/LLVM pass in `llvm/lib/Transforms/Instrumentation/` transforms LLVM IR to insert checks around memory accesses, function calls, and other operations of interest. The pass inserts calls to runtime functions (e.g., `__asan_load8`, `__tsan_write4`) or inline check sequences directly into the IR.

**Runtime library**: The `compiler-rt/lib/` tree contains the sanitizer runtimes. They initialize shadow memory on startup, define the `__asan_*` / `__msan_*` / `__tsan_*` / `__ubsan_*` functions, provide replacements for heap allocation (`malloc`, `free`, `new`, `delete`), and intercept libc functions that perform memory operations.

The runtimes share common infrastructure in `compiler-rt/lib/sanitizer_common/`:

```
compiler-rt/lib/sanitizer_common/
├── sanitizer_allocator.h         # Two-level allocator shared by ASan/MSan
├── sanitizer_allocator_primary64.h
├── sanitizer_common.cpp          # Process/signal utilities, abort handlers
├── sanitizer_flags.cpp           # ASAN_OPTIONS / MSAN_OPTIONS parsing
├── sanitizer_interceptors_*.cpp  # libc function wrappers (memcpy, read, ...)
├── sanitizer_stacktrace.cpp      # Fast (FP-chain) and slow (DWARF) unwind
├── sanitizer_symbolizer.cpp      # llvm-symbolizer subprocess integration
└── sanitizer_suppressions.cpp    # Suppression list parsing and matching
```

### 110.1.2 Shadow Memory

The canonical sanitizer technique is **shadow memory**: a compact parallel mapping that stores metadata about each byte of application memory. The shadow is mapped at program startup via `mmap(MAP_ANONYMOUS | MAP_NORESERVE | MAP_FIXED_NOREPLACE)`; most shadow pages are never physically backed (zero pages via `MAP_NORESERVE`) until the corresponding application pages are touched.

The check inserted by the compiler is:

```asm
; Generic shadow check pattern (pseudocode):
shadow_addr = (app_addr >> shift) + shadow_base
shadow_value = *shadow_addr
if (shadow_value != 0) call runtime_error_handler(app_addr)
; ... original load or store ...
```

The `shift` and `shadow_base` are compile-time constants (architecture-specific). The check is typically 4–6 instructions and is almost always not taken — modern branch predictors handle this well.

### 110.1.3 Sanitizer Common Allocator

ASan and MSan replace the system allocator with a custom implementation from `compiler-rt/lib/sanitizer_common/`. The sanitizer allocator is a two-level structure:

- **Primary allocator**: size-class-based slab allocator with per-CPU caches (similar to Scudo's primary, Chapter 112). Handles small and medium allocations (up to ~64KB on 64-bit).
- **Secondary allocator**: mmap-based for large allocations; maps memory at randomly chosen addresses.

Both levels add metadata around allocations for the sanitizer's use (red zones for ASan, origin tracking for MSan).

---

## 110.2 AddressSanitizer (ASan)

### 110.2.1 Shadow Memory Encoding

ASan uses 1 byte of shadow for every 8 bytes of application memory. The mapping is:

```
shadow_addr = (app_addr >> 3) + shadow_base

Shadow byte value meanings:
  0        = all 8 bytes are accessible
  1–7      = first N bytes accessible, bytes [N..7] are red zone
  0xfa (-6) = heap left red zone
  0xfb (-5) = heap right red zone
  0xfc (-4) = heap freed / quarantined
  0xf1 (-15)= stack left red zone
  0xf3 (-13)= stack right red zone
  0xf5 (-11)= stack use after return
  0xf8 (-8) = global left red zone
  0xf9 (-7) = global right red zone
  0xe8 (-24)= asan array cookie
  0xac (-84)= contiguous container overflow marker
```

For a sub-8-byte access (e.g., a 4-byte `int` at `addr`), the check must verify that the access does not span into a poisoned partial-granule region:

```llvm
; ASan check for 4-byte load at %p:
%shadow = getelementptr i8, ptr @__asan_shadow_memory_dynamic_address,
          i64 %shifted_addr   ; shifted_addr = addr >> 3
%shadow_byte = load i8, ptr %shadow, align 1
; If shadow_byte > 0 and (addr & 7) + 4 > shadow_byte → poisoned:
%last_byte = add i8 %granule_offset, 3   ; last byte offset within granule = (addr&7)+3
%is_partial = icmp sgt i8 %shadow_byte, 0
%overflows = icmp sgt i8 %last_byte, %shadow_byte
%error = and i1 %is_partial, %overflows
br i1 %error, label %asan_report, label %ok
asan_report:
  call void @__asan_report_load4(i64 %addr) noreturn
ok:
  %v = load i32, ptr %p, align 4
```

This logic is in [`AddressSanitizerPass::instrumentAddress`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Instrumentation/AddressSanitizer.cpp).

### 110.2.2 Heap Allocation and Red Zones

ASan replaces `malloc`/`free` with instrumented versions in [`compiler-rt/lib/asan/asan_allocator.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/lib/asan/asan_allocator.cpp):

**Allocation layout** (8-byte aligned):
```
[Left Red Zone: 32–128 bytes, shadow=0xfa]
[User allocation: N bytes, shadow=0..7]
[Right Red Zone: 32–128 bytes, shadow=0xfb]
[Chunk header: stored in separate metadata, not adjacent to user data]
```

The chunk header stores: allocation size, chunk state (allocated/quarantined/freed), thread ID, and a compressed stack trace of the allocation site. Stack traces are stored using the frame-pointer-chain fast unwind, typically capturing 30 frames in ~200 nanoseconds.

**Free path**:
1. Verify chunk header magic/checksum (detects double-free, invalid-free)
2. Poison the entire allocation and its red zones as `0xfc` (use-after-free detection)
3. Store the deallocation stack trace in the chunk header
4. Move the chunk to the per-thread quarantine (held up to `quarantine_size_mb` total)
5. When quarantine is flushed, unpoison and return to the size-class free list

### 110.2.3 Stack Instrumentation

[`AddressSanitizerPass::instrumentFunction`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Instrumentation/AddressSanitizer.cpp) transforms stack allocations:

1. All `alloca` instructions in the function are gathered
2. A single large stack allocation is substituted: `alloca [total_size + red_zones]`
3. Each original `alloca` is replaced with a pointer into this large allocation, with 32-byte red zones between variables
4. The prologue calls `__asan_set_shadow_XX` to poison the red zones
5. The epilogue calls the corresponding unpoisoning routines

```cpp
// Before ASan (pseudocode):
void foo() {
  int a[10];   // 40 bytes
  int b[10];   // 40 bytes
  // use a, b...
}

// After ASan (pseudocode):
void foo() {
  // Single large stack allocation:
  char frame[32 + 40 + 32 + 40 + 32];  // red zones between variables
  int *a = (int*)(frame + 32);          // a starts after first red zone
  int *b = (int*)(frame + 32 + 40 + 32); // b starts after a + its red zone
  
  // Prologue: poison red zones
  __asan_set_shadow_f1(frame, 32);         // left red zone (0xf1)
  __asan_set_shadow_f3(frame+72, 32);      // middle red zone (0xf3)
  __asan_set_shadow_f3(frame+144, 32);     // right red zone (0xf3)
  __asan_unpoison_stack_memory(a, 40);     // a is accessible (0x00)
  __asan_unpoison_stack_memory(b, 40);     // b is accessible (0x00)
  
  // ... use a, b ...
  
  // Epilogue: unpoison (allow the stack to be reused safely):
  __asan_unpoison_stack_memory(frame, sizeof(frame));
}
```

### 110.2.4 Global Instrumentation

Globals receive red zones via module-level metadata. The `AddressSanitizerPass::instrumentGlobals` pass creates a parallel global structure for each instrumented global:

```cpp
// Compiler-generated metadata for global "g" (int g = 42):
struct AsanGlobal {
  const void *beg;            // &g
  size_t size;                // sizeof(g)
  size_t size_with_redzone;   // sizeof(g) + redzone_size
  const char *name;           // "g"
  const char *module_name;    // "file.cpp"
  uptr has_dynamic_init;      // 1 if g has dynamic initializer
  const __asan_global_source_location *location; // source location
  uptr odr_indicator;         // for ODR violation detection
};

// Registered via the module's global constructor:
__asan_register_globals(&__asan_global_g, 1);
```

The global itself is extended in memory: `sizeof(g) + 32` bytes are allocated; the last 32 bytes are poisoned as `0xf9` (global right red zone).

### 110.2.5 Using ASan and Error Reports

```bash
# Compile:
clang -fsanitize=address -fno-omit-frame-pointer -g -O1 test.c -o test

# Key compile flags:
# -fno-omit-frame-pointer: required for accurate stack traces
# -g: debug info for symbolizer
# -O1: enough optimization to avoid trivial bugs being eliminated,
#       but keeps most useful stack context

# Runtime options:
ASAN_OPTIONS=\
  detect_leaks=1:\          # enable LSan integration (Linux default)
  abort_on_error=0:\        # print report and exit(1), not SIGABRT
  halt_on_error=0:\         # continue after first error
  quarantine_size_mb=64:\   # larger quarantine = more UAF detection
  redzone=128:\             # larger red zones for deeper overflows
  detect_stack_use_after_return=1:\  # fake stack for UAR detection
  symbolize=1:\             # symbolize in-process (requires llvm-symbolizer)
  log_path=/tmp/asan.log    # write reports to file (useful in CI)
./test
```

A typical heap-buffer-overflow report:

```
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000000050
READ of size 4 at 0x602000000050 thread T0
    #0 0x4011a3 in foo /src/test.c:8:5
    #1 0x4011e2 in main /src/test.c:14:3

0x602000000050 is located 0 bytes after 40-byte region [0x602000000028,0x602000000050)
allocated by thread T0 here:
    #0 0x4e9782 in malloc (/usr/lib/llvm-22/lib/clang/22/lib/libclang_rt.asan-x86_64.so)
    #1 0x401183 in foo /src/test.c:5:15

Shadow bytes around the buggy address:
  0x0c047fff7fb0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c047fff7fc0: 00 00 00 00 00 00 00 00 00 00[fa]fa fa fa fa fa
```

The shadow byte dump shows `fa` (heap right red zone) at the accessed address, confirming the overflow.

---

## 110.3 MemorySanitizer (MSan)

### 110.3.1 Shadow Model

MSan uses a **1:1 bit shadow**: for every application bit, there is one shadow bit indicating whether that bit is "poisoned" (uninitialized). This means MSan's shadow consumes the same amount of address space as the application. Practically, shadow pages are allocated lazily; the shadow of uninitialized stack and heap memory is `0xff...` (all bits uninitialized) and is cleared when memory is written.

MSan's shadow layout on Linux x86_64:

```
Application address range:   0x000000000000 - 0x7fffffffffff
Shadow base:                  0x500000000000
Shadow of address A:          A + 0x500000000000

Origin base (when enabled):   0x200000000000
Origin of address A:          (A >> 2) + 0x200000000000
```

[`MemorySanitizer.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Instrumentation/MemorySanitizer.cpp) is the largest instrumentation pass in LLVM — over 4000 lines — because it must handle the shadow of every IR operation.

### 110.3.2 Taint Propagation

MSan propagates the "uninitialized" property through all computations. The instrumentation pass inserts shadow operations alongside the original IR:

```llvm
; Original: %r = add i32 %a, %b
; MSan instrumentation:
%sa = load i32, ptr shadow(%a)   ; shadow of input a
%sb = load i32, ptr shadow(%b)   ; shadow of input b
%sr = or i32 %sa, %sb            ; result is uninitialized if either input is
store i32 %sr, ptr shadow(%r)    ; update shadow of result

%r = add i32 %a, %b             ; original instruction unchanged

; For a conditional branch:
; Original: br i1 %cond, label %then, label %else
; MSan check (consequential use of uninitialized value):
%scond = load i1, ptr shadow(%cond)
call void @__msan_warning_with_origin(i1 %scond, i32 origin(%cond))
; ... then the original branch
```

The `@__msan_warning_with_origin` function is a no-op if `%scond == 0` (initialized); it reports a bug if `%scond == 1` (uninitialized). The origin parameter provides the allocation site for the uninitialized data.

Special cases the pass handles:
- **`select`**: `shadow(result) = select(shadow(cond) != 0, all_ones, select(cond, shadow(true), shadow(false)))`
- **`load`**: shadow is loaded from shadow memory; check if loading from poisoned address via separate ASan-like check
- **`store`**: write shadow to shadow memory (no check needed for the store itself)
- **`call`**: argument shadows passed via `__msan_param_tls[]`; return value shadow in `__msan_retval_tls`
- **`phi`**: shadow is phi of the argument shadows
- **SIMD intrinsics**: each intrinsic has a hand-written shadow rule in `MemorySanitizer.cpp`

### 110.3.3 Origin Tracking

With `-fsanitize-memory-track-origins=2` (maximum detail), MSan maintains a separate **origin map** alongside the shadow:

- Origin of a poisoned byte = an identifier for the "stack trace of the allocation/stack frame that created this uninitialized byte"
- Origin propagation: `origin(r) = origin(a)` if `shadow(a)` is set; otherwise `origin(b)` — first non-zero input wins
- Origin "chains": MSan maintains a per-process chain-of-custody table that records where uninitialized bytes came from and passed through

```bash
# Enable origin tracking:
clang -fsanitize=memory -fsanitize-memory-track-origins=2 -g test.c
MSAN_OPTIONS=origin_history_size=8:verbosity=1 ./test
```

Origin tracking doubles the memory overhead (origin map = same size as shadow map) and adds ~50% more CPU overhead on top of baseline MSan, but is invaluable for diagnosing complex uninitialized-memory paths.

### 110.3.4 Interceptors and Third-Party Libraries

MSan intercepts all libc memory functions to update shadows correctly:

```cpp
// From compiler-rt/lib/msan/msan_interceptors.cpp:
INTERCEPTOR(void *, memcpy, void *dst, const void *src, uptr n) {
  void *res = REAL(memcpy)(dst, src, n);
  __msan_copy_shadow(dst, src, n);  // copy shadow from src to dst
  return res;
}

INTERCEPTOR(ssize_t, read, int fd, void *buf, uptr n) {
  ssize_t res = REAL(read)(fd, buf, n);
  if (res > 0)
    __msan_unpoison(buf, res);  // kernel wrote initialized data
  return res;
}

INTERCEPTOR(int, scanf, const char *format, ...) {
  // After scanf writes output variables: unpoison all output args
  // ... complex handling via format string analysis
}
```

The main practical challenge: any library not compiled with MSan produces uninitialized-looking outputs, causing false positives. Solutions:

1. Build the entire dependency tree with MSan (the `msan-kit` approach used by Google internally)
2. Use `__msan_unpoison(ptr, size)` after calls to uninstrumented code
3. Use `-fsanitize-ignorelist` to suppress specific functions

---

## 110.4 ThreadSanitizer (TSan)

### 110.4.1 Data Race Detection

TSan detects concurrent unsynchronized accesses to the same memory location where at least one access is a write. It uses a combination of **per-memory shadow cells** and **vector clocks** implementing a happens-before analysis.

**Vector clock**: Thread T maintains vector clock `VC[T]`, a vector of logical timestamps indexed by thread ID. `VC[T][i]` = "the last clock value of thread `i` that thread `T` has observed." When thread T synchronizes with thread S (e.g., T acquires a mutex that S released), T updates `VC[T][i] = max(VC[T][i], VC[S][i])` for all `i`.

**Shadow cells**: For each 8-byte memory granule, TSan stores four 64-bit (or 128-bit on TSan v2) "shadow words". Each shadow word encodes:

| Bits | Field |
|------|-------|
| [39:0] | Thread clock (40 bits in TSan v3 on x86_64) |
| [47:40] | Thread epoch (to detect thread reuse) |
| [53:48] | Thread ID (6 bits → 64 threads per "segment") |
| [55:54] | Access size (0=1B, 1=2B, 2=4B, 3=8B) |
| [56] | Is-write flag |
| [57] | Is-atomic flag |

When a new access to granule G occurs from thread T:
1. Read the four shadow cells for G
2. For each cell C (representing a prior access by thread T' at clock E'):
   - If T' == T: update the cell with the current clock (same thread, no race)
   - If T' != T and E' > VC[T][T']: **race detected** — T's clock has not "seen" this prior access
   - If T' != T and E' <= VC[T][T']: no race (T knows about this access via synchronization)
3. If a race is detected: report both the current access and the racing prior access (from the shadow cell)

### 110.4.2 TSan v3

TSan v3 (merged LLVM 13) made key improvements over TSan v2:

- **Reduced memory**: Per-granule shadow from 32 bytes (4×64-bit cells) to 16 bytes on 64-bit systems. The reduction comes from using smaller shadow words (removing some precision in clock encoding) and a better segment encoding.
- **Global shadow region**: Instead of mmap'ing shadow at program-startup-time fixed offsets, TSan v3 uses a single fixed shadow region computed as `addr * 2` (effectively using the upper half of the virtual address space as shadow for the lower half).
- **AArch64/RISC-V improvements**: Correct handling of AArch64 relaxed memory ordering in TSan shadow cell updates.

[`ThreadSanitizerPass`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Instrumentation/ThreadSanitizer.cpp) instruments:
- All non-atomic loads: `call @__tsan_read{1,2,4,8,16}(ptr %p)`
- All non-atomic stores: `call @__tsan_write{1,2,4,8,16}(ptr %p)`
- Function entry: `call @__tsan_func_entry(ptr callerPC)` (updates clock)
- Function exit: `call @__tsan_func_exit()`
- Atomic operations: replaced by `__tsan_atomic_{load,store,fetch_add,...}` with explicit memory order argument

### 110.4.3 Synchronization Interceptors

TSan intercepts all synchronization primitives to update vector clocks:

| Intercepted function | TSan action |
|---------------------|-------------|
| `pthread_mutex_unlock(m)` | Release: set `m->clock = VC[current_thread]` |
| `pthread_mutex_lock(m)` | Acquire: `VC[current_thread] = max(VC[T], m->clock)` |
| `pthread_cond_signal(cv)` | Release: record current clock in `cv->clock` |
| `pthread_cond_wait(cv, m)` | Acquire: update from `cv->clock` after mutex re-lock |
| `sem_post(s)` | Release semantics |
| `sem_wait(s)` | Acquire semantics |
| `pthread_create(...)` | Child inherits parent's current clock |
| `pthread_join(t, ...)` | Acquire from thread t's final clock |
| `std::atomic<T>::store(acquire_release)` | Atomic store with clock update |

Atomic operations with `memory_order_relaxed` do NOT update vector clocks — they are checked for races but not used for synchronization.

### 110.4.4 Using TSan

```bash
clang -fsanitize=thread -fno-omit-frame-pointer -g -O1 test.cpp -lpthread
TSAN_OPTIONS=\
  halt_on_error=0:\      # find all races, don't stop at first
  history_size=4:\       # per-thread history (higher → more accurate reports)
  report_atomic_races=1 \ # report races on atomic variables too
  ./test

# TSan report format:
# WARNING: ThreadSanitizer: data race (pid=...)
#   Write of size 4 at 0x7f... by thread T2:
#     #0 increment /.../counter.cpp:5
#   Previous read of size 4 at 0x7f... by thread T1:
#     #0 read_counter /.../counter.cpp:12
```

---

## 110.5 UndefinedBehaviorSanitizer (UBSan)

### 110.5.1 Instrumented Checks

UBSan inserts targeted conditional branches for each class of C/C++ UB. Unlike ASan/MSan/TSan, it uses **no shadow memory** — every check is local to the computation being checked. This makes UBSan cheap to add alongside any other sanitizer.

Complete list of commonly-used UBSan checks:

| Flag | `__ubsan_handle_*` function | Detects |
|------|---------------------------|---------|
| `signed-integer-overflow` | `add_overflow`, `sub_overflow`, `mul_overflow` | `INT_MAX + 1` |
| `unsigned-integer-overflow` | (same, but not UB — opt-in) | `UINT_MAX + 1` wraps |
| `shift-base`, `shift-exponent` | `shift_out_of_bounds` | `1 << -1`, `1 << 64` |
| `float-divide-by-zero` | `float_divide_by_zero` | `1.0f / 0.0f` |
| `float-cast-overflow` | `float_cast_overflow` | `(int)(1e40)` |
| `null` | `type_mismatch` | Dereference null pointer |
| `alignment` | `type_mismatch` | Misaligned pointer dereference |
| `bounds` | `out_of_bounds` | `a[10]` where `a` has size 5 |
| `enum` | `load_invalid_value` | Load of invalid enum value from memory |
| `function` | `function_type_mismatch` | Call function through incompatible pointer |
| `vptr` | `dynamic_type_cache_miss` | Virtual call on wrong-type object (vtable) |
| `return` | `missing_return` | Control falls off non-void function |
| `unreachable` | `builtin_unreachable` | `__builtin_unreachable()` actually reached |
| `nonnull-attribute` | `nonnull_arg` | Null passed to `__attribute__((nonnull))` |
| `object-size` | (inline check) | Write past `__builtin_object_size` result |
| `pointer-overflow` | `pointer_overflow` | Pointer arithmetic wraps past end of object |

### 110.5.2 Instrumentation Pattern

For each check, UBSan uses `llvm.sadd.with.overflow` (or equivalent) intrinsics to detect UB without changing the computation's result:

```llvm
; Source: int result = a + b;  (a, b are int)
; UBSan signed-integer-overflow check:

; Compute addition with overflow detection:
%overflow_pair = call {i32, i1} @llvm.sadd.with.overflow.i32(i32 %a, i32 %b)
%result = extractvalue {i32, i1} %overflow_pair, 0
%overflowed = extractvalue {i32, i1} %overflow_pair, 1

; If overflow, call handler (may continue or abort):
br i1 %overflowed, label %overflow_bb, label %no_overflow

overflow_bb:
  ; SourceLocation and TypeDescriptor are compile-time constants:
  call void @__ubsan_handle_add_overflow(
      ptr @add_overflow_data,    ; {SourceLocation, TypeDescriptor}
      i64 zext(i32 %a to i64),  ; left operand
      i64 zext(i32 %b to i64))  ; right operand
  br label %no_overflow          ; continue (or handler calls abort)

no_overflow:
  ; Use %result ...
```

The `add_overflow_data` is a compile-time global:

```cpp
struct OverflowData {
  SourceLocation Loc;       // file, line, column
  TypeDescriptor *Type;     // "int" with bit width and sign info
};
static const OverflowData add_overflow_data = { {"file.c", 42, 13}, &int_type };
```

This means UBSan's runtime has no per-location state — all metadata is in read-only globals, and the runtime functions just format and print the report.

### 110.5.3 Trap Mode and Minimal Runtime

With `-fsanitize-trap=all`, UBSan replaces runtime calls with trap instructions:

```llvm
; Instead of calling @__ubsan_handle_add_overflow:
overflow_bb:
  call void @llvm.trap()  ; → ud2 on x86_64; udf #0 on AArch64; illegal on RISC-V
```

The program terminates with `SIGILL` at the exact UB site. No runtime library is needed, no diagnostic is printed, and the binary is smallest possible. This is the mode used for production hardening: the overhead is one conditional branch per check, and the security benefit is that UB cannot silently corrupt state.

`-fsanitize-minimal-runtime` uses a tiny runtime (few KB) that prints only file:line without the full type information:

```
/path/to/file.c:42: runtime error
```

versus the full runtime's:

```
/path/to/file.c:42:13: runtime error: signed integer overflow:
  2147483647 + 1 cannot be represented in type 'int'
```

### 110.5.4 CFI — Control Flow Integrity

CFI is a security-focused UBSan extension that restricts indirect calls and virtual function calls to valid targets:

```bash
# Forward-edge CFI (indirect calls):
# - Calls through function pointers can only go to functions with matching type
clang -fsanitize=cfi-icall -flto -fvisibility=hidden program.c -o program

# Virtual dispatch CFI:
# - vcalls can only target the correct vtable class
clang -fsanitize=cfi-vcall -flto program.cpp -o program

# Shadow call stack (AArch64):
# - Return addresses stored in a separate shadow stack (x18 on AArch64)
clang -fsanitize=shadow-call-stack program.c -o program \
      -target aarch64-linux-gnu
```

CFI requires LTO because the set of valid call targets must be known at link time. Each indirect call site is instrumented with a check against a "type hash" of the call target. The hash is computed from the function's type signature and stored in a metadata section; the linker creates a lookup table.

---

## 110.6 LeakSanitizer (LSan)

### 110.6.1 Algorithm: Conservative GC Scan

LSan detects memory leaks via a conservative garbage collection algorithm. At process exit (or when `__lsan_do_leak_check()` is called):

1. **Stop the world**: all threads are suspended via `ptrace(PTRACE_ATTACH)` or `kill(SIGSTOP)` to prevent memory mutations during the scan
2. **Enumerate roots**: collect all potential pointers from:
   - The call stack of every thread (scanned word-by-word)
   - Global variables (`.data`, `.bss`, `.rodata` segments)
   - Saved register state (via `getcontext`/`sigaltstack`)
   - Thread-local storage (TLS)
3. **Mark reachable**: treat every word-aligned value that appears to be a valid heap address as a pointer; mark the pointed-to allocation (and all allocations transitively reachable from it) as live
4. **Report unreachable**: any allocation not marked live is a leak; report with the allocation stack trace

```
Root scan:
  stack[0..n] | globals | registers | TLS
       ↓
  is_heap_address(word)?  →  mark allocation
       ↓
  scan allocation's bytes for more pointers
       ↓
  fixpoint (transitive closure of reachable)
       ↓
  leaked = all allocations not in reachable set
```

The algorithm is **conservative** because it cannot distinguish a genuine pointer from an integer that happens to equal a heap address. A leaked allocation that has a stale integer copy of its address on the stack will not be reported. This means LSan can have false negatives (missed leaks) but essentially zero false positives.

### 110.6.2 Integration with ASan

LSan is integrated into ASan by default on Linux:

```bash
# ASan + LSan (default):
clang -fsanitize=address -g test.c
ASAN_OPTIONS=detect_leaks=1 ./test
# Leak report is printed after any ASan report and at exit.
```

When running with ASan, LSan uses ASan's allocation tracking to enumerate all live heap allocations (ASan tracks every malloc/free). This is more precise than standalone LSan, which must intercept the system allocator.

### 110.6.3 Standalone LSan

```bash
# Standalone LSan (lighter than ASan, ~1.1x overhead):
clang -fsanitize=leak -fno-omit-frame-pointer -g test.c
LSAN_OPTIONS=\
  report_objects=1:\          # report the leaked objects' addresses
  print_suppressions=0:\      # don't print suppression stats
  verbosity=1:\               # verbose output
  suppressions=/path/to/lsan.supp \
  ./test

# Suppression file format (lsan.supp):
# leak:CRYPTO_malloc              # by function in allocation chain
# leak:my_global_registry_init   # by function name
# leak:third_party/protobuf/*    # by source path
# called_from_lib:libssl.so.*    # by library
```

### 110.6.4 LSan API

```cpp
#include <sanitizer/lsan_interface.h>

// Trigger a leak check right now (can be called multiple times):
__lsan_do_leak_check();

// Trigger only if leaks exist (same as end-of-process):
__lsan_do_recoverable_leak_check();

// Mark an allocation as intentional (will not be reported as leak):
void *registry = malloc(1024);
__lsan_ignore_object(registry);

// Disable/enable LSan for a scope:
__lsan_disable();
global_singleton = new MySingleton();  // intentional leak
__lsan_enable();
```

---

## 110.7 Sanitizer Blocklists and Suppressions

### 110.7.1 Compile-Time Ignorelists

The `-fsanitize-ignorelist=file.txt` flag (formerly `-fsanitize-blacklist`) instructs the instrumentation pass to skip specific functions or source files. Format:

```
# Skip all instrumentation in this file:
src:*/third_party/*

# Skip a specific function (ASan, MSan, TSan):
fun:my_custom_memcpy_impl

# Skip functions matching a glob:
fun:*_internal_*

# Ignore a specific type from UBSan type checks:
type:MyUnsafeRawBuffer

# ASan: don't poison this global:
global:my_large_preallocated_buffer
```

```bash
# Use separate ignorelists per sanitizer:
clang -fsanitize=address,undefined \
      -fsanitize-ignorelist=asan_ignore.txt \
      -fsanitize-ignorelist=ubsan_ignore.txt \
      test.c
```

### 110.7.2 Per-Function Source Attributes

```cpp
// Suppress ASan for a specific function:
__attribute__((no_sanitize("address")))
void pool_allocator_internal(void *pool, size_t offset, size_t n) {
  // This function manages a custom memory pool;
  // ASan red zones would falsely flag legitimate accesses
}

// Suppress all sanitizers:
__attribute__((no_sanitize("all")))
void runtime_allocator_fast_path(void) {}

// Suppress UBSan unsigned overflow (intentional):
__attribute__((no_sanitize("unsigned-integer-overflow")))
uint32_t fnv1a_hash(const uint8_t *data, size_t len) {
  uint32_t hash = 2166136261u;
  for (size_t i = 0; i < len; ++i)
    hash = (hash ^ data[i]) * 16777619u;  // intentional overflow
  return hash;
}

// C++ sanitizer suppression (Clang extension):
[[clang::no_sanitize("thread")]]
void benign_racy_flag_read(void) {
  if (g_initialized_flag) do_something();  // benign double-checked read
}
```

---

## 110.8 Combining Sanitizers and Performance Overview

### 110.8.1 Compatibility Matrix

| Combination | Compatible? | Notes |
|------------|-------------|-------|
| ASan + UBSan | Yes | Most common development combination |
| ASan + LSan | Yes | Default on Linux; integrated into ASan runtime |
| MSan + UBSan | Yes | Both use inline checks + small runtime |
| TSan + UBSan | Yes | TSan shadow + UBSan inline checks |
| ASan + TSan | No | Shadow memory regions conflict |
| ASan + MSan | No | Shadow memory regions conflict |
| MSan + TSan | No | Shadow memory regions conflict |
| HWASan + UBSan | Yes | HWASan shadow + UBSan inline |
| HWASan + TSan | No | Shadow conflict |

The rule: ASan, MSan, and TSan each occupy large shadow memory regions that conflict with each other. Only one heavy sanitizer at a time. UBSan and LSan have no shadow memory and combine freely.

### 110.8.2 Performance Overhead

| Sanitizer | CPU overhead | Memory overhead | Shadow model | Hot path cost |
|-----------|-------------|-----------------|--------------|--------------|
| ASan | 2–3× | 2× + red zones | 1:8 byte shadow | Shadow load + compare per access |
| MSan | 2–3× | 2× | 1:1 bit shadow | Shadow load + store per access |
| TSan | 5–15× | 5–10× | 4×64-bit cells per 8B | Shadow cell lookup + VClock |
| UBSan | 5–30% | Negligible | None | One conditional branch per check |
| LSan | 1.1–1.2× | Negligible | Allocation list | End-of-process scan only |
| ASan+UBSan | 2–3× | 2× | ASan shadow | Combined hot paths |

### 110.8.3 Recommended Workflows

**Development (all platforms)**:

```bash
# Primary dev build — catches most bugs quickly:
clang -fsanitize=address,undefined \
      -fno-omit-frame-pointer \
      -fsanitize-address-use-after-scope \
      -fsanitize-address-use-after-return=runtime \
      -g -O1 \
      program.c -o program_asan

ASAN_OPTIONS=detect_leaks=1:detect_stack_use_after_return=1 \
UBSAN_OPTIONS=print_stacktrace=1:halt_on_error=0 \
./program_asan
```

**Threading bugs** (separate build required):

```bash
clang -fsanitize=thread -fno-omit-frame-pointer -g -O1 \
      program.cpp -o program_tsan -lpthread
TSAN_OPTIONS=halt_on_error=0:history_size=4 ./program_tsan
```

**Uninitialized memory** (separate build required):

```bash
clang -fsanitize=memory -fno-omit-frame-pointer -g -O1 \
      -fsanitize-memory-track-origins=2 \
      program.c -o program_msan
MSAN_OPTIONS=halt_on_error=0:origin_history_size=8 ./program_msan
```

**Production hardening** (minimal overhead, no runtime):

```bash
clang -fsanitize=undefined -fsanitize-trap=all \
      -fsanitize=cfi-icall,cfi-vcall -flto \
      -fvisibility=hidden -O2 \
      program.cpp -o program_hardened
```

---

## DataFlowSanitizer (DFSan)

DataFlowSanitizer tracks *information flow* through a program — which data values influence which other data values — using a technique called *taint tracking* or *dynamic data flow analysis*. Where ASan asks "was this memory access valid?", DFSan asks "did this sensitive value reach this output?".

### How Taint Tracking Works

DFSan associates each byte of memory and each register with a *label* — an integer identifying which "taint sources" influence that byte. Labels propagate automatically through arithmetic, comparisons, and memory operations: if tainted bytes `a` and `b` are added together, the result carries the union of their labels.

The label union operation is the heart of DFSan. Every binary operation produces a result whose label is the union of the operands' labels. A label of zero means "not tainted by any tracked source". Non-zero labels encode which taint sources have contributed to the value's computation. This transitivity ensures that any value derived — however indirectly — from a labelled source will itself carry a non-zero label.

### Shadow Memory Architecture

DFSan uses *shadow memory* — a parallel address space that shadows every byte of the application's memory. The shadow stores the label for each application byte.

Two label schemes, selected at compile time:

- **`fast8`** (default in LLVM 22): 8-bit labels; supports up to 255 distinct taint sources. Shadow ratio: 1:1 (one shadow byte per application byte). Shadow base: `0x200000000000`.
- **`fast16`**: 16-bit labels; supports up to 65535 sources. Shadow ratio: 2:1 (two shadow bytes per application byte).

The shadow mapping on x86-64:

```
Shadow(app_addr) = (app_addr & ~0x700000000000) | 0x200000000000
```

DFSan instruments every load and store to propagate labels:

```cpp
// Original: int c = a + b;
// DFSan instrumented (conceptual):
dfsan_label la = dfsan_read_label(&a, sizeof(a));
dfsan_label lb = dfsan_read_label(&b, sizeof(b));
int c = a + b;
dfsan_label lc = dfsan_union(la, lb);
dfsan_set_label(lc, &c, sizeof(c));
```

In `fast8` mode, `dfsan_union(l1, l2)` is resolved via a precomputed 256×256 lookup table (64 KiB). This makes the union operation a single table lookup — O(1) with no dynamic allocation — at the cost of limiting the label space to 255 sources. For most auditing and fuzzing applications this is sufficient; when more sources are needed, `fast16` extends the space to 65535.

The shadow pages are mapped lazily via `MAP_NORESERVE`; only pages corresponding to application pages that carry tainted data are ever physically backed. Application memory that was never labelled has a zero shadow and incurs no physical overhead.

### Building with DFSan

```bash
clang -fsanitize=dataflow -fno-sanitize-ignorelist \
      -o target target.c libdfsan_abi.a
```

Key flags:

- `-fsanitize=dataflow`: enables DFSan instrumentation pass and links the DFSan runtime from `compiler-rt/lib/dfsan/`
- `-fsanitize-dataflow-track-origins=1`: enables origin tracking (records where each taint was introduced, at the cost of 4× memory overhead for the origin map)
- `-mllvm -dfsan-fast-8-labels` (default) or `-mllvm -dfsan-fast-16-labels`: selects the label scheme
- `-fno-sanitize-ignorelist`: prevents the default ignorelist from suppressing instrumentation on libc wrappers

DFSan requires that all libraries that the target links against be either (a) compiled with DFSan themselves, or (b) covered by ABI lists (`dfsan_abilist.txt`) that teach DFSan how to propagate taint through their functions. The DFSan runtime ships a default ABI list covering common libc functions.

### The DFSan API

```c
#include <sanitizer/dfsan_interface.h>

// Create a new label for a taint source
dfsan_label secret_label = dfsan_create_label("secret", 0);

// Apply a label to memory
dfsan_set_label(secret_label, &password, sizeof(password));

// Read the label of a value
dfsan_label result_label = dfsan_get_label(computed_value);

// Test label membership
if (dfsan_has_label(result_label, secret_label)) {
    fprintf(stderr, "Secret data reached output!\n");
}

// Get all labels on a memory range
dfsan_label range_label = dfsan_read_label(buf, buf_len);

// Retrieve the human-readable name attached to a label
const dfsan_label_info *info = dfsan_get_label_info(result_label);
fprintf(stderr, "Tainted by: %s\n", info->name);
```

`dfsan_union(l1, l2)` — returns the union of two labels. In `fast8` mode, the union table is a precomputed 256×256 array (64 KiB). In `fast16` mode, the union is stored in a dynamically grown union table and unions are memoised to avoid repeated allocation.

`dfsan_has_label(label, needle)` — returns non-zero if `needle` is in the union represented by `label`. Because `fast8` and `fast16` use flat integers rather than sets, this is implemented by walking the union table backward from `label` looking for `needle` as a component — O(depth of the union tree), typically O(1) for simple cases.

### Origin Tracking

When `-fsanitize-dataflow-track-origins=1` is set, DFSan records the *origin* of each tainted value: the stack trace where `dfsan_set_label` was called, and subsequent stores that propagated the label. Origins are stored in a separate shadow (4 bytes per application byte). `dfsan_get_origin(val)` returns the origin ID; `dfsan_print_origin_trace(origin, desc)` prints the propagation chain.

Origin tracking substantially increases memory overhead (the origin map is 4× the size of the application's memory footprint relative to the label shadow) and adds approximately 30–50% additional CPU cost on top of baseline DFSan overhead. However, it is indispensable when diagnosing violations: without origins, DFSan can tell you that a value at the point of a sink check carries label `secret_label`, but not where in the code that label was first applied or which intermediate operations carried it along.

```bash
# Enable origin tracking and verbose reporting:
clang -fsanitize=dataflow -fsanitize-dataflow-track-origins=1 -g \
      -o target target.c
DFSAN_OPTIONS=warn_unimplemented=0:strict_data_dependencies=0 ./target
```

### Use Cases

**1. Information-flow security analysis**

Verify that secret values (cryptographic keys, passwords, personal data) never flow to logging functions, network outputs, or debug prints. The pattern is to label sensitive inputs at the point they are read, then check labels at all output sinks:

```c
// Read a key from an HSM or config:
unsigned char aes_key[32];
read_key_from_secure_store(aes_key, 32);
dfsan_label key_label = dfsan_create_label("aes_key", 0);
dfsan_set_label(key_label, aes_key, 32);

// ... program executes ...

// At any logging call:
void audited_printf(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    // Check that the format string itself is not tainted:
    if (dfsan_has_label(dfsan_get_label(fmt), key_label)) {
        abort();  // key material in format string — information leak
    }
    // Check each variadic argument (requires type information) ...
    vprintf(fmt, args);
    va_end(args);
}
```

**2. Taint-guided fuzzing**

Use DFSan labels to track which bytes of the input influence a target condition. When DFSan reports that a comparison involves input bytes N..M, the fuzzer can mutate those bytes specifically. AFL++ and libFuzzer both have DFSan integration modes:

```bash
# libFuzzer + DFSan for data-flow guided fuzzing:
clang -fsanitize=fuzzer,dataflow -g target_fuzz.c -o fuzz_target
# libFuzzer will internally call dfsan_get_label on comparison operands
# to guide mutation toward bytes that influence branch decisions.
./fuzz_target -data_flow_trace=1 corpus/
```

The fuzzer reads DFSan labels on both operands of each `cmp` instruction to determine which input bytes are "interesting" for that comparison. This dramatically improves fuzzer efficiency on programs with complex input parsers where random mutations rarely produce valid structures.

**3. Third-party library auditing**

Track whether untrusted input (from a network parser, for example) reaches unsafe operations like `memcpy` length arguments or `exec` arguments:

```c
// Mark data arriving from the network as tainted:
ssize_t n = recv(sockfd, network_buf, sizeof(network_buf), 0);
dfsan_label net_label = dfsan_create_label("network_input", 0);
dfsan_set_label(net_label, network_buf, n);

// In the program logic, check before using as a length:
size_t copy_len = parse_length_field(network_buf);
if (dfsan_has_label(dfsan_get_label(copy_len), net_label)) {
    // copy_len is influenced by network data — validate before use
    if (copy_len > MAX_SAFE_COPY) { handle_error(); }
}
memcpy(dst, src, copy_len);
```

### Integration with ASan

DFSan can be combined with ASan for simultaneous memory safety and information-flow checking:

```bash
clang -fsanitize=dataflow,address -o target target.c
```

Both shadow maps coexist because they use different shadow address ranges. ASan's shadow occupies `0x000000000000`–`0x1fffffffffff` (shifted right by 3), while DFSan's shadow base is `0x200000000000` — they do not overlap on x86-64. The combined build pays the overhead of both sanitizers, but provides the strictest possible safety net during development: memory safety violations (ASan) and information-flow violations (DFSan) are both caught in a single execution.

### Limitations

- **No inter-process taint propagation**: taint does not cross `fork()`/`exec()` boundaries. The child process starts with a clean shadow.
- **Implicit flows not tracked**: control flow depending on tainted data (e.g., `if (secret) { x = 1; } else { x = 0; }`) is not tracked — only explicit data-flow edges are instrumented. A determined adversary can exfiltrate 1 bit per branch, which DFSan cannot detect.
- **Syscall boundaries**: system calls that copy data into user buffers are treated as "uninstrumented writes" — the kernel-side is not instrumented. DFSan relies on interceptors in the ABI list to manually unpoison buffers that receive kernel-written data (e.g., `read(2)` is intercepted to `__dfsan_unpoison` the read buffer).
- **Significant overhead**: 2–10× slowdown, 2–3× memory overhead (comparable to MSan). Not suitable for production use; intended for offline auditing and fuzzing.
- **ABI list completeness**: any library function not covered by the ABI list is treated as "uninstrumented" and receives the "discard" policy (output labels are dropped). Missing ABI entries are a common source of false negatives (taint being silently dropped at library boundaries).

Reference LLVM 22 source links:

- [`DataFlowSanitizer.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Instrumentation/DataFlowSanitizer.cpp)
- [`dfsan_interface.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/include/sanitizer/dfsan_interface.h)

---

## TypeSanitizer (TySan): Type-Confusion Detection

Type confusion — accessing memory through a pointer of a different type than the one used to allocate it — is among the most exploited vulnerability classes in browser engines and large C++ codebases. A canonical example is writing to a `Base*` that actually points to a `Derived` object and then reading a field that exists only in `Base`, or casting an allocation to an unrelated type via `reinterpret_cast` and accessing fields that overlap in unexpected ways. Union-based type punning is another common vector.

### TySan vs. UBSan -fsanitize=vptr

UBSan's `-fsanitize=vptr` check is narrow: it fires only at virtual dispatch, verifying that the vtable pointer of the object is consistent with the declared static type of the pointer at the call site. It does not track allocations and cannot detect type confusion that does not involve a virtual call — for example, writing through a reinterpreted pointer to a non-polymorphic struct field.

TypeSanitizer (TySan) takes a broader approach. It tracks the *allocation type* of every region of memory and detects any subsequent access through a pointer of a mismatched type, regardless of whether virtual dispatch is involved. The check fires on load and store instructions, not only at call sites.

### Shadow Memory Scheme

TySan uses a shadow memory scheme derived from LLVM debug type information (`DIType`). Each distinct C++ type gets a unique 32-bit *type tag* derived from the type's debug info. For each byte of application memory, TySan maintains a shadow 32-bit slot recording which type tag "owns" that memory.

```
Shadow address for application byte A:
  shadow_addr = (A >> 3) * 4 + shadow_base
  (8-byte application granule → 4-byte shadow tag slot)
```

On allocation (heap, stack, or global), TySan records the allocating type's tag into the shadow slots covering the allocated region.

### Instrumentation

The TySan instrumentation pass ([`llvm/lib/Transforms/Instrumentation/TypeSanitizer.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Instrumentation/TypeSanitizer.cpp)) instruments every store to write the type tag of the stored-to pointer's declared type, and every load to compare the actual shadow tag against the expected tag for the pointer's declared type. A mismatch triggers a diagnostic:

```cpp
// TySan runtime diagnostic (simplified):
// __tysan_check(ptr, type_tag, access_size, is_write)
// On tag mismatch:
// ERROR: TypeSanitizer: type-confusion
//   ACCESS of type int at 0x... (declared type)
//   ALLOCATED as float at 0x...
```

**Special cases** handled by the runtime:
- `placement new` on an existing allocation: updates the shadow tag to the new type
- `reinterpret_cast` with an explicit annotation: suppressed if the user has annotated the cast as intentional
- Union access: TySan uses the most recent store's type to determine the "active member"; accesses to other members fire a diagnostic unless the union is declared `may_alias`

### C++ Union Example

```cpp
union FloatInt {
  float f;
  int   i;
};

FloatInt u;
u.f = 3.14f;     // shadow tagged as 'float'
int x = u.i;    // TySan fires: access as 'int', shadow says 'float'
```

```bash
# Compile and run with TySan:
clang-22 -fsanitize=type -g -O1 tysan_example.cpp -o tysan_example
./tysan_example
# ERROR: TypeSanitizer: type-confusion on address 0x7fff...
#   WRITE of type 'float' at 0x7fff... pc 0x401234
#   READ  of type 'int'   at 0x7fff... pc 0x401256
```

### Performance and Comparison

TySan's overhead is typically **2–5× slowdown**, heavier than ASan because it must update type tags on every store (not only on allocation boundary crossings). Memory overhead is roughly 4× due to the 4-byte-per-8-byte-granule shadow.

| Feature | UBSan `-fsanitize=vptr` | TySan `-fsanitize=type` |
|---------|------------------------|------------------------|
| Scope | Virtual dispatch only | Any typed memory access |
| Overhead | ~5–15% | 2–5× |
| Shadow memory | None | 4 bytes per 8-byte granule |
| C++ EH compatibility | Yes | Requires `-g` for type info |
| Shipped since | Clang 3.8 | Clang 19 / 20 |

TySan requires `-g` to generate the `DIType` metadata from which type tags are derived. Without debug info, TySan cannot distinguish types and emits no instrumentation.

---

## alloc-token Sanitizer: Allocator-Metadata UAF Detection

ASan detects use-after-free (UAF) by poisoning quarantined allocations and inserting shadow checks on every access. This works well for the standard allocator, but has gaps:
- Custom allocators that bypass ASan's interceptors (pool allocators, arena allocators, slab allocators) are invisible to ASan's quarantine
- When a dangling pointer is accessed, ASan can identify *that* a UAF occurred but may not distinguish *which* original allocation the pointer referred to when allocations are packed tightly

The `alloc-token` sanitizer addresses these gaps with a per-allocation *token*: a small piece of metadata bound to each allocation at creation time and invalidated at deallocation. Any access through a pointer after its token has been invalidated is a UAF.

### sanitize_alloc_token Attribute

Custom allocator functions are annotated to teach the runtime about them:

```cpp
// Mark a custom allocator's alloc/free functions:
void *pool_alloc(size_t n)
    __attribute__((sanitize_alloc_token, returns_nonnull));

void pool_free(void *p)
    __attribute__((sanitize_alloc_token));
```

The compiler inserts calls to `__sanitizer_alloc_token_create()` at the exit of `pool_alloc` and `__sanitizer_alloc_token_invalidate()` at the entry of `pool_free`.

### __builtin_infer_alloc_token

The `__builtin_infer_alloc_token(ptr)` builtin (landed in Clang 20) extracts the token associated with a pointer. This is used by generic code that holds a pointer and needs to check its validity later:

```cpp
// Save token at acquisition time:
auto token = __builtin_infer_alloc_token(my_ptr);

// ... time passes, my_ptr may be freed ...

// Check validity before use:
if (__sanitizer_check_alloc_token(token)) {
    use(my_ptr);  // safe: token still valid → allocation not freed
} else {
    abort();      // UAF: token invalidated by pool_free
}
```

### How It Differs from ASan

| Aspect | ASan | alloc-token |
|--------|------|-------------|
| Detection mechanism | Shadow memory poisoning | Token invalidation |
| Shadow size | 1 byte per 8 bytes | Per-allocation token (small) |
| Works with custom allocators | Requires interceptor wrapping | Yes, via attribute annotation |
| Cross-allocator UAF | Limited | Yes |
| Overhead | 2–3× CPU, 2× memory | Lower memory; similar CPU |

The token approach is particularly valuable for:
- **Pool allocators**: when an entire pool is reset, all tokens in it are invalidated at once; stale pointers from any allocation in the pool are caught
- **Region-based allocators**: similarly, invalidating a region's token catches all stale references to objects within it
- **Slab allocators**: the token distinguishes between two different allocations that happen to occupy the same slab slot at different times — a capability ASan lacks when a new allocation fills the old slot and the quarantine has been flushed

### Example

```cpp
// Custom pool allocator with alloc-token annotation:
struct Pool {
    char buf[4096];
    size_t offset = 0;
};

__attribute__((sanitize_alloc_token))
void *pool_alloc(Pool *p, size_t n) {
    void *r = p->buf + p->offset;
    p->offset += n;
    return r;
}

__attribute__((sanitize_alloc_token))
void pool_reset(Pool *p) {
    // Invalidates tokens for ALL allocations in the pool:
    p->offset = 0;
}

// UAF caught:
Pool p;
int *x = (int *)pool_alloc(&p, sizeof(int));
*x = 42;
pool_reset(&p);   // x's token is now invalidated
int y = *x;       // alloc-token fires: UAF
```

```bash
clang-22 -fsanitize=alloc-token -g pool_example.cpp -o pool_example
./pool_example
# ERROR: alloc-token: use-after-invalidation at 0x...
#   Access at ...pool_example.cpp:21
#   Token invalidated at ...pool_example.cpp:17
```

---

## NumericalStabilitySanitizer (NSan): Floating-Point Instability Detection

Floating-point instability — catastrophic cancellation, order-sensitivity, accumulated rounding error — is difficult to detect with ordinary testing because the program produces numerically plausible but subtly wrong results. NSan addresses this by running a *shadow computation* in higher precision alongside the program's actual floating-point operations.

### Shadow Arithmetic

NSan instruments every floating-point operation to maintain a shadow value in a higher-precision type:

| Actual type | Shadow type |
|-------------|-------------|
| `float` (32-bit) | `double` (64-bit) |
| `double` (64-bit) | `long double` (80-bit on x86) |

The instrumentation pass ([`llvm/lib/Transforms/Instrumentation/NumericalStabilitySanitizer.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Instrumentation/NumericalStabilitySanitizer.cpp)) inserts shadow operations alongside each FP instruction:

```llvm
; Source: float r = a + b;
; NSan shadow instrumentation:
%shadow_a = load double, ptr shadow(a)    ; shadow of input a
%shadow_b = load double, ptr shadow(b)    ; shadow of input b
%shadow_r = fadd double %shadow_a, %shadow_b  ; shadow add in double
store double %shadow_r, ptr shadow(r)

%r = fadd float %a, %b                    ; actual computation unchanged

; Divergence check: compare float(%shadow_r) to %r
%shadow_r_f = fptrunc double %shadow_r to float
%diff = call float @llvm.fabs.f32(float fsub(%r, %shadow_r_f))
%rel_err = fdiv float %diff, float @llvm.fabs.f32(%shadow_r_f)
%exceeds = fcmp ogt float %rel_err, 1e-6  ; configurable epsilon
br i1 %exceeds, label %nsan_report, label %ok
```

### Divergence Detection

If the shadow (high-precision) result and the actual (low-precision) result diverge by more than a configurable relative epsilon, NSan reports a potential instability. The diagnostic identifies the source line and shows both the actual and shadow values.

```bash
# Enable NSan:
clang-22 -fsanitize=numerical -g catastrophic.cpp -o catastrophic

NSAN_OPTIONS=halt_on_error=0 ./catastrophic
# WARNING: NumericalStabilitySanitizer: floating-point instability
#   at catastrophic.cpp:8:  r = (a + b) - a
#   actual result:   0.000000000 (float)
#   shadow result:   1.000000119e-07 (double)
#   relative error:  1.0
```

### Classic Example: Catastrophic Cancellation

```cpp
// Catastrophic cancellation: (a + b) - a where b << a
float a = 1e7f;
float b = 1.0f;
float r = (a + b) - a;
// In float: (a+b) rounds to a → r = 0.0 (wrong)
// In double: 10000001.0 - 10000000.0 = 1.0 (correct)
// NSan: relative error = 1.0 → reports instability
```

NSan is **experimental** in LLVM 22. Performance overhead is typically **5–20× slowdown** due to shadow computation, making it unsuitable for continuous integration; it is best run on targeted numerical kernels during development. The epsilon threshold is configurable via `NSAN_OPTIONS=relative_error_threshold=<value>`.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **TSan v3 AArch64 MTE integration**: Active LLVM patches ([D154989](https://reviews.llvm.org/D154989) series) extend TSan v3 to use ARM Memory Tagging Extension hardware for tag-based UAF detection, reducing TSan's ~10–15× slowdown by offloading tag checking to hardware on Armv8.5+ cores.
- **NumericalStabilitySanitizer (NSan) stabilization**: NSan (shipped as experimental in LLVM 22) has open RFC discussions on discourse.llvm.org for graduating to non-experimental status, adding `__float128` shadow support, and ARM NEON vector shadow propagation; expected to stabilize in LLVM 23.
- **alloc-token sanitizer upstreaming completion**: The `sanitize_alloc_token` attribute and `__builtin_infer_alloc_token` builtin (landed Clang 20–21) have pending patches to integrate token checking into the sanitizer common allocator so that pool allocator UAF shows up in combined ASan+alloc-token builds without requiring custom interceptors.
- **UBSan `implicit-integer-sign-change` and `implicit-integer-truncation` in `-fsanitize=undefined`**: These checks have been opt-in since Clang 10; the LLVM 23 tracking issue proposes including them in the default `-fsanitize=undefined` group after confirming false-positive rates, which would improve coverage for implicit C integer conversion bugs in Linux kernel builds.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **HWASan on x86_64 via LAM (Linear Address Masking)**: Intel's Linear Address Masking (available from Sapphire Rapids, enabled via `CR4.LAM_U57`) provides a hardware-assisted equivalent of ARM TBI for pointer tagging. LLVM patches ([discourse.llvm.org/t/hwasan-lam](https://discourse.llvm.org/t/hwasan-x86-64-linear-address-masking)) aim to bring HWASan's ~15% overhead to x86_64 workloads, compared to ASan's 2–3× overhead, enabling always-on sanitization in data-center deployments.
- **DFSan scalable label sets beyond fast16**: Replacing DFSan's 16-bit flat labels with a hash-consed trie of label sets (research direction from the "Neutaint" and "FlowDroid" literature) would allow tracking thousands of distinct taint sources simultaneously, enabling full-program GDPR compliance audits where each PII field is a distinct label rather than sharing a coarse "sensitive" bucket.
- **MSan SIMD coverage completeness**: The `MemorySanitizer.cpp` shadow propagation for LLVM SIMD intrinsics (AVX-512, SVE, RISC-V V extension) is a persistent maintenance burden; ongoing work to auto-generate shadow rules from LLVM's `IntrinsicsX86.td` / `IntrinsicsSVE.td` rather than hand-coding them would reduce false negatives from uninstrumented vector paths.
- **Sanitizer-aware LTO inlining**: A known limitation is that sanitizer instrumentation is inserted before LTO inlining, meaning cross-module inlining can expose uninstrumented code paths in the sanitized binary. LLVM RFC discussions propose a post-LTO re-instrumentation pass that re-applies sanitizer checks after the global inliner has run, closing this coverage gap.

### 5-Year Horizon (Long-Term, by ~2031)

- **Always-on production ASan via Arm MTE full-system integration**: Arm's Memory Tagging Extension is already deployed in Android (MTE on Pixel 8+); long-term roadmap (described in the Arm "Memory Safety" white paper and LLVM compiler-rt MTE tracking issue) targets zero-overhead UAF/OOB detection in all production Arm server deployments via hardware tag checking with 16-byte granularity, replacing the software shadow model entirely.
- **Sanitizer-integrated fuzzing with structured taint flow (SanTaint/SanFuzz)**: Academic proposals combining DFSan-style taint tracking with structure-aware fuzzing (as in "SoFi" ASPLOS 2022 and "StructFuzz") aim to guide fuzzers using full data-flow graphs rather than per-byte label heuristics; LLVM integration would likely surface as a `compiler-rt` library with a libFuzzer-compatible callback interface.
- **Formally-verified sanitizer runtime**: Following the CompCert approach (Chapter 212), research projects targeting verified correctness of the ASan shadow check algorithm (i.e., proving that the shadow memory invariant is maintained by the allocator and instrumentation) could yield a certified sanitizer runtime usable in safety-critical embedded contexts where finding memory bugs must be provably sound.

---

## Chapter 110 Summary

- Sanitizers combine compile-time LLVM pass instrumentation with a `compiler-rt` runtime library; shared infrastructure in `sanitizer_common/` provides stack unwinding, symbolization, and environment-variable flag parsing.
- ASan uses 1:8 shadow encoding (1 byte per 8-byte granule) with distinct poison values for heap, stack, and global red zones; the allocator adds per-allocation metadata and quarantine for UAF detection; instrumentation inserts 4–6 instructions per memory access.
- MSan uses 1:1 bit shadow for uninitialized-memory tracking; the instrumentation pass propagates shadow through all IR operations (add, select, phi, call); origin tracking (`-fsanitize-memory-track-origins=2`) records allocation sites; interceptors handle libc calls that produce initialized data.
- TSan uses vector clocks and 4-slot per-granule shadow to detect happens-before violations; v3 reduced shadow size to 16 bytes per granule; interceptors for all pthread synchronization primitives update vector clocks to establish the happens-before relation.
- UBSan inserts targeted conditional branches with zero shadow memory; `-fsanitize-trap=all` emits `ud2`/`udf #0` for near-zero-overhead production use; CFI (`-fsanitize=cfi-icall,cfi-vcall`) restricts indirect/virtual calls to valid targets via LTO-computed type sets.
- LSan performs a stop-the-world conservative GC scan at process exit; it is integrated into ASan by default and available standalone at ~1.1× overhead; `__lsan_ignore_object` and suppression files handle intentional long-lived allocations.
- ASan/MSan/TSan shadows are mutually exclusive; UBSan and LSan combine freely with any of the three; recommended development workflow is ASan+UBSan for most work, with separate TSan and MSan builds.
- Ignorelists (`-fsanitize-ignorelist`), per-function `no_sanitize` attributes, and runtime suppression files allow suppressing third-party library noise and known-benign patterns without disabling the sanitizer globally.


---

@copyright jreuben11
