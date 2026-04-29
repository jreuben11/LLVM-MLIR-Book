# Chapter 123 — LLVM-libc

*Part XVII — Runtime Libraries*

LLVM-libc is the LLVM project's implementation of the C standard library. Unlike libc++ (which implements C++), LLVM-libc targets the POSIX C interface: `<string.h>`, `<math.h>`, `<stdio.h>`, `<stdlib.h>`, and the POSIX thread API. Its design goals are unusual among libc implementations: correctness and testability first, full isolation between individual functions, support for both overlay deployment (replacing specific glibc functions) and full hermetic builds, and GPU/embedded portability. This chapter covers the architecture, the public-only-headers approach, per-function selection for overlay builds, the testing infrastructure, and the LLVM 22 state of GPU and embedded support.

---

## 23.1 Design Philosophy

### 23.1.1 Goals Contrasted with glibc/musl

| Aspect | glibc | musl | LLVM-libc |
|--------|-------|------|-----------|
| Primary goal | POSIX completeness | Correctness + size | Per-function correctness + testability |
| Header sharing | Some public | Separate | Full public-only-headers |
| Overlay mode | No | No | Yes (first-class) |
| GPU support | No | No | Yes (NVPTX/AMDGPU) |
| Per-function testing | Partial | No | Mandatory |
| LLVM integration | No | No | Yes (TableGen entrypoints) |

### 23.1.2 The Public-Only-Headers Approach

In glibc, many internal symbols leak into the headers or are visible in the DSO. LLVM-libc's design rule is that every function exposed in the public API is declared in a header that the user can include, but the implementation is entirely in the `.cpp` files under `libc/src/`. No internal state or implementation detail is exposed through headers. This makes it possible to test each function entirely in isolation, without linking the rest of the library.

---

## 23.2 Repository Layout

```
libc/
├── config/                 # Per-target configuration
│   ├── linux/              # Linux (x86_64, aarch64, riscv)
│   ├── baremetal/          # Bare-metal/freestanding
│   ├── gpu/                # NVPTX/AMDGPU
│   └── windows/            # Windows (partial)
├── include/                # Generated public headers (from .h.def files)
├── src/                    # Function implementations
│   ├── string/             # memcpy, memset, strlen, strchr, ...
│   ├── math/               # sin, cos, sqrt, fma, ...
│   ├── stdio/              # printf, fread, fopen, ...
│   ├── stdlib/             # malloc, free, strtol, ...
│   ├── pthread/            # pthread_create, mutex, ...
│   ├── unistd/             # read, write, close, ...
│   └── ...
├── test/                   # Hermetic unit tests per function
│   ├── src/string/         # Tests for each string function
│   ├── src/math/
│   └── ...
├── fuzzing/                # Differential fuzz tests vs reference
├── utils/
│   └── HdrGen/             # TableGen-based header generator
└── docs/
    ├── FullBuildInstructions.rst
    ├── OverlayMode.rst
    └── ...
```

---

## 23.3 TableGen Entrypoints

LLVM-libc uses TableGen to define which functions are included in a given build configuration. Each function is declared as an "entrypoint" in a `.td` file:

```tablegen
// libc/src/string/string_fns.td (simplified)
def memcpy  : CStdFunction;
def memset  : CStdFunction;
def strlen  : CStdFunction;
def strchr  : CStdFunction;
def strstr  : CStdFunction;
```

A platform configuration file (`libc/config/linux/x86_64/entrypoints.txt`) lists the entrypoints to include:

```
# libc/config/linux/x86_64/entrypoints.txt (excerpt)
libc.src.string.memcpy
libc.src.string.memset
libc.src.string.strlen
libc.src.stdio.printf
libc.src.stdlib.malloc
...
```

The build system generates headers and CMake targets from these lists. Adding a new function requires declaring it in the TableGen file, adding a corresponding `entrypoint.cpp` under `src/`, writing a test under `test/src/`, and adding it to the appropriate config file. This pipeline ensures no function is included without a test.

---

## 23.4 The TableGen-to-Header Pipeline

The end-to-end flow from function definition to installed header is worth tracing explicitly because it differs from every other C library.

### 23.4.1 Step 1: Entrypoint .td Declaration

```tablegen
// libc/src/string/string_fns.td
def memcpy  : CStdFunction;
def memset  : CStdFunction;
def strlen  : CStdFunction;
```

### 23.4.2 Step 2: .h.def Header Template

Each public header has a `.h.def` template (in `libc/include/`) that uses the `%%ENTRYPOINT%%` macro:

```
// libc/include/string.h.def (simplified)
#ifndef __LLVM_LIBC_STRING_H
#define __LLVM_LIBC_STRING_H

%%INCLUDE_FILE_HEADER%%

__BEGIN_C_DECLS

%%#entrypoint memcpy
void *memcpy(void *__restrict dst, const void *__restrict src, size_t n);
%%

%%#entrypoint memset
void *memset(void *dst, int c, size_t n);
%%

%%#entrypoint strlen
size_t strlen(const char *s);
%%

__END_C_DECLS

#endif
```

The `HdrGen` tool (in `libc/utils/HdrGen/`) reads the platform's `entrypoints.txt` and the `.h.def` template to produce the installed `string.h`. Functions not listed in the entrypoints file are simply omitted from the generated header, making it impossible to call a function that wasn't selected.

### 23.4.3 Step 3: Implementation and LIBC_NAMESPACE

Every implementation file starts with the `LIBC_NAMESPACE` guard:

```cpp
// libc/src/string/memcpy.cpp
#include "src/string/memcpy.h"
#include "src/__support/macros/config.h"

namespace LIBC_NAMESPACE_DECL {

void *memcpy(void *__restrict dst,
             const void *__restrict src,
             size_t n) {
    return inline_memcpy(dst, src, n);
}

} // namespace LIBC_NAMESPACE_DECL
```

`LIBC_NAMESPACE_DECL` expands to `__llvm_libc_22` (for LLVM 22). This means the symbol in the object file is `__llvm_libc_22::memcpy`, preventing any collision with the system `memcpy`.

### 23.4.4 Step 4: Alias to the Public Name

Each entrypoint object file defines a public alias from the namespace-qualified symbol to the bare C name, using a compiler extension:

```cpp
// Automatically generated by the CMake entrypoint infrastructure
extern "C" void *memcpy(void *dst, const void *src, size_t n)
    __attribute__((alias("_ZN16__llvm_libc_2226memcpyEPvPKvm")));
```

In overlay mode this alias wins because the linker sees it before the system `memcpy`. In full build mode it's the only `memcpy` in the link.

---

## 23.5 Deployment Modes

### 23.4.1 Full Build Mode

A full build produces a standalone `libc.a` / `libc.so` that completely replaces the system C library. This is appropriate for hermetic toolchains (bare-metal, embedded RTOS, container builds with controlled dependencies):

```bash
cmake -G Ninja \
  -DLLVM_ENABLE_RUNTIMES="libc" \
  -DLIBC_TARGET_OS=linux \
  -DLIBC_TARGET_ARCHITECTURE=x86_64 \
  -DLIBC_FULL_BUILD=ON \
  ../llvm
ninja libc
```

In full build mode, LLVM-libc provides `crt0.o`, `crtbegin.o`, and `crtend.o` as well as the library, enabling fully self-contained executables.

### 23.4.2 Overlay Mode

Overlay mode produces a set of object files or a thin `.a` that, when linked with `-whole-archive` before the system libc, overrides specific functions. This is useful for:

- Replacing `memcpy`/`memset` with LLVM-libc's faster or more correct implementations on a system using glibc.
- Using LLVM-libc's correctly-rounded math functions (`sin`, `cos`, `sqrt`) on a system where glibc's math is not correctly rounded.
- Testing LLVM-libc behavior on a development machine without replacing the system libc.

```bash
cmake -DLIBC_ENABLE_OVERLAY=ON ...
ninja libc-overlay
# Links with:
# -Wl,--whole-archive -lllvmlibc_overlay -Wl,--no-whole-archive -lc
```

### 23.4.3 Choosing Build Mode

```
Target environment       → Recommended mode
─────────────────────────────────────────────
Bare-metal (no OS)       → Full build
Container hermetic build → Full build
Linux development host   → Overlay (for specific functions)
GPU offload              → GPU build mode
RTOS with existing libc  → Overlay
```

---

## 23.6 Mathematics: Correctly Rounded Functions

One of LLVM-libc's flagship features is correctly rounded math. The C standard (ISO C99 Annex F) requires that math functions produce the floating-point result closest to the exact mathematical result (i.e., as if computed with infinite precision and then rounded). Most libm implementations, including glibc for many functions, are not correctly rounded for all inputs.

### 23.5.1 Correctly Rounded sin/cos/exp/log

LLVM-libc provides correctly rounded implementations for:

```
sinf, cosf, tanf, asinf, acosf, atanf, atan2f
expf, exp2f, exp10f, logf, log2f, log10f
sqrtf, cbrtf, hypotf, powf
sin, cos, tan, exp, log, sqrt  (double precision)
```

The implementation uses Sollya-generated polynomial approximations with extended-precision (128-bit or double-double) intermediate arithmetic to ensure the final result is correctly rounded. The technique is based on the CR-LIBM research project.

Example — `sinf` implementation structure:

```
1. Range reduction: x → [−π/4, π/4] using Payne-Hanek reduction
2. Polynomial evaluation: sin(x) ≈ x + x³·P(x²)
   (P generated by Sollya for correctly-rounded result)
3. Range reconstruction: apply sin/cos addition formulas
4. Final rounding: ensure result is correctly rounded (ULP = 0.5)
```

Source: [src/math/sinf.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/libc/src/math/generic/sinf.cpp)

### 23.5.2 FEnv and Rounding Mode Support

LLVM-libc math functions honor the current floating-point rounding mode (set via `fesetround`). This is in contrast to many libm implementations that hardcode round-to-nearest. The rounding mode is checked at the end of computation if the result might not be correctly rounded in all modes.

---

## 23.7 String Functions

LLVM-libc provides optimized implementations of the core string functions, with architecture-specific backends:

### 23.6.1 memcpy / memset Strategy

```
memcpy dispatch:
  if (size <= 16)   → inline scalar copy
  if (size <= 128)  → SIMD loop (SSE2/NEON)
  if (size <= 4096) → rep movsb (x86) or DC ZVA (AArch64)
  else              → ERMS / non-temporal stores for large copies
```

The implementations live in `libc/src/string/` with platform specializations in `libc/src/string/x86_64/` and `libc/src/string/aarch64/`.

### 23.6.2 The LIBC_INLINE_ASM Approach

Unlike glibc (which uses raw inline assembly), LLVM-libc wraps architecture-specific intrinsics via C++ builtins where possible:

```cpp
// libc/src/string/memory_utils/x86_64/inline_memcpy.h
LIBC_INLINE void inline_memcpy_avx512(void *dst,
                                       const void *src,
                                       size_t count) {
    using T = cpp::array<uint8_t, 64>;
    for (; count >= 64; count -= 64, dst += 64, src += 64)
        store<T>(dst, load<T>(src));
    // Handle tail (< 64 bytes) with overlap technique
}
```

This keeps the code readable and allows the optimizer to fold the loop further.

---

## 23.8 GPU Support

LLVM-libc is the first C standard library implementation with first-class GPU support. The GPU configurations target NVPTX (CUDA) and AMDGPU (ROCm/HIP).

### 23.7.1 GPU Build Configuration

```bash
cmake -DLIBC_TARGET_OS=gpu \
      -DLIBC_GPU_ARCHITECTURES="gfx90a;sm_80" \
      -DLIBC_GPU_BACKEND=amdhsa \
      ...
```

The GPU build produces bitcode files (`.bc`) that can be linked into device-side CUDA/HIP kernels, providing device-callable versions of string functions, math functions, and utilities.

### 23.7.2 Available GPU Functions

Functions available in the GPU build (LLVM 22):

| Category | Functions |
|----------|-----------|
| Math | `sinf/cosf/expf/logf/sqrtf` and double variants |
| String | `memcpy/memset/memcmp/strlen/strcpy` |
| Stdlib | `atoi/atof/strtol/strtof` |
| Printf | `printf` (limited format spec, no `%s` with VA args) |

The GPU `printf` routes through a host-side buffer flush mechanism compatible with both CUDA's `printf` semantics and AMD's `printf` extension.

### 23.7.3 RPC: Remote Procedure Calls for I/O

GPU kernels cannot directly call the OS. LLVM-libc implements a Ring Buffer Protocol (RPC) for GPU-to-CPU communication, allowing device code to invoke host-side I/O:

```
GPU thread: printf("value = %d\n", x)
    │
    ▼
Writes to shared RPC ring buffer in device-visible memory
    │
    ▼
Host CPU polling thread reads ring buffer, executes printf
    │
    ▼
Returns status to GPU thread (blocking or non-blocking)
```

Source: [libc/src/gpu/rpc/](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/libc/src/gpu/rpc/)

---

## 23.9 Testing Infrastructure

### 23.8.1 Per-Function Hermetic Tests

Every function has a test under `libc/test/src/<category>/<function>_test.cpp`. Tests are hermetic: they link only against the specific function under test, not the full library. This catches unintended inter-function dependencies.

```cpp
// libc/test/src/string/memcpy_test.cpp
#include "src/string/memcpy.h"
#include "test/UnitTest/Test.h"

TEST(LlvmLibcMemcpyTest, ForwardSrc) {
    char src[10] = {'1','2','3','4','5','6','7','8','9','0'};
    char dst[10] = {0};
    LIBC_NAMESPACE::memcpy(dst, src, 10);
    EXPECT_STR_EQ(dst, src);
}

TEST(LlvmLibcMemcpyTest, Overlapping) {
    char buf[] = "123456789";
    LIBC_NAMESPACE::memcpy(buf + 2, buf, 7);
    // check buffer contents
}
```

The `LIBC_NAMESPACE` macro resolves to the LLVM-libc internal namespace (`__llvm_libc_22`) which prevents collisions with the system libc.

### 23.8.2 Differential Fuzzing

The `libc/fuzzing/` directory contains fuzz harnesses that compare LLVM-libc output against the system reference implementation (glibc or Apple libc):

```cpp
// libc/fuzzing/math/sin_fuzz.cpp
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    if (size < sizeof(double)) return 0;
    double x; memcpy(&x, data, sizeof(double));
    double llvmlibc_result = __llvm_libc::sin(x);
    double reference_result = __builtin_sin(x);
    // Check they agree to within 1 ULP
    if (!within_1_ulp(llvmlibc_result, reference_result)) {
        __builtin_trap();
    }
    return 0;
}
```

This is particularly valuable for math functions where correctness is non-obvious.

### 23.8.3 ULP Testing for Math

Math function tests use a ULP (Units in the Last Place) framework to verify correct rounding:

```cpp
// From libc/test/src/math/sin_test.cpp
TEST_F(LlvmLibcSinTest, SpecialNumbers) {
    EXPECT_FP_EQ(aNaN, __llvm_libc::sin(aNaN));
    EXPECT_FP_EQ(0.0f, __llvm_libc::sin(0.0f));
    EXPECT_FP_EQ(-0.0f, __llvm_libc::sin(-0.0f));
}

TEST_F(LlvmLibcSinTest, InRangeNumbers) {
    // Test correctness by exhaustive float sweep (for sinf)
    for (uint32_t bits = 0; bits <= UINT32_MAX; ++bits) {
        float x = cpp::bit_cast<float>(bits);
        if (isnan(x) || isinf(x)) continue;
        ASSERT_FP_EQ(__llvm_libc::sinf(x), reference_sinf(x));
    }
}
```

For `float` (32-bit), exhaustive testing over all ~4 billion values is feasible at ~1ms per thousand comparisons. For `double`, random and targeted sampling is used.

---

## 23.10 Embedded and Freestanding Support

### 23.9.1 Baremetal Configuration

```bash
cmake -DLIBC_TARGET_OS=baremetal \
      -DLIBC_TARGET_ARCHITECTURE=aarch64 \
      -DLIBC_ENABLE_FILESYSTEM=OFF \
      -DLIBC_ENABLE_THREADS=OFF \
      -DLIBC_FULL_BUILD=ON \
      ...
```

The baremetal configuration provides:
- `string.h` functions (all of them)
- `math.h` functions
- `stdlib.h` basics (`atoi`, `strtol`, `qsort`, `bsearch`)
- No `stdio.h` (file I/O), no `pthread.h`, no `unistd.h`

### 23.9.2 Scudo Integration

LLVM-libc can be configured to use Scudo (the hardened allocator from compiler-rt) for `malloc`/`free`/`realloc`:

```cmake
set(LIBC_USE_SCUDO ON)
```

This creates a combination where the C library's memory management is performed by Scudo's hardened allocator, giving LLVM-libc full deployment hardened memory safety without requiring a separate `LD_PRELOAD`. See [Chapter 112 — Production Allocators: Scudo and GWP-ASan](../part-16-jit-sanitizers/ch112-production-allocators-scudo-gwp-asan.md) for Scudo internals.

---

## 23.11 printf Dispatch and Linux Syscall Wrappers

### 23.11.1 printf Format Dispatch

LLVM-libc implements `printf` via a table-driven format dispatcher that avoids recursion and dynamic allocation in the hot path:

```cpp
// libc/src/stdio/printf_core/dispatcher.h (simplified)
template <typename OutFn>
int dispatch(OutFn &writer, const char *fmt, va_list vlist) {
    while (*fmt) {
        if (*fmt != '%') {
            writer.write(*fmt++);
            continue;
        }
        ++fmt; // skip '%'
        FormatSection section = parse_format_string(fmt);
        switch (section.conv_name) {
        case 'd': case 'i':
            writer << convert_int(section, vlist);   break;
        case 'u': case 'o': case 'x': case 'X':
            writer << convert_uint(section, vlist);  break;
        case 'f': case 'F': case 'e': case 'E':
        case 'g': case 'G':
            writer << convert_float(section, vlist); break;
        case 's':
            writer << convert_string(section, vlist); break;
        case 'p':
            writer << convert_pointer(section, vlist); break;
        case '%':
            writer.write('%'); break;
        }
    }
    return writer.bytes_written();
}
```

The `writer` template parameter is swapped to implement `printf` (writes to stdout fd), `sprintf` (writes to a buffer), `fprintf` (writes to a `FILE*`), and `snprintf` (bounded buffer). All reuse the same dispatch logic.

Floating-point conversion in LLVM-libc uses the Ryu algorithm (fast shortest-decimal) for `%g` and `%e`, and a direct digit-extraction approach for `%f`. The implementations guarantee the same decimal representation that `strtod` would round-trip back to.

### 23.11.2 Linux Syscall Wrappers

OS-level operations in LLVM-libc go through a thin syscall abstraction layer rather than calling glibc wrappers:

```cpp
// libc/src/__support/OSUtil/linux/syscall.h (simplified)
template <typename... Args>
long syscall(long number, Args... args) {
#if defined(__x86_64__)
    long result;
    __asm__ volatile(
        "syscall"
        : "=a"(result)
        : "0"(number), args_to_registers(args...)
        : "rcx", "r11", "memory");
    return result;
#elif defined(__aarch64__)
    register long x8 asm("x8") = number;
    // ... AArch64 svc #0 ...
#endif
}
```

A specific syscall wrapper like `write`:

```cpp
// libc/src/unistd/linux/write.cpp
ssize_t write(int fd, const void *buf, size_t count) {
    long ret = syscall(SYS_write, (long)fd, (long)buf, (long)count);
    if (ret < 0) {
        libc_errno = -ret;  // thread-local errno
        return -1;
    }
    return (ssize_t)ret;
}
```

The `libc_errno` is a thread-local variable defined in `libc/src/__support/libc_errno.h`. Unlike glibc's `errno` which is typically a macro expanding to `(*__errno_location())`, LLVM-libc's `errno` is a straightforward `_Thread_local int`. This makes it portable to environments where `__errno_location()` is not available (bare-metal, GPU).

---

## 23.12 LLVM 22 Status

As of LLVM 22.1, the state of LLVM-libc:

| Category | Status |
|----------|--------|
| `string.h` | Complete; optimized for x86_64/AArch64/RISC-V |
| `math.h` (float) | Complete; all functions correctly rounded |
| `math.h` (double) | Most functions complete; a few in progress |
| `stdio.h` | `printf`/`scanf`/file I/O on Linux; partial on other platforms |
| `stdlib.h` | Most functions complete; `malloc` via Scudo optional |
| `pthread.h` | Core functions (create, join, mutex, cond) on Linux |
| `unistd.h` | Core syscall wrappers on Linux |
| GPU (NVPTX) | Math + string + printf via RPC; sm_70+ |
| GPU (AMDGPU) | Math + string + printf via RPC; gfx90a+ |
| Windows | Partial (string, math only) |
| Full build (Linux) | Usable for hermetic containers |
| Baremetal | string + math on AArch64/RISC-V |

---

## Chapter Summary

- LLVM-libc implements the C standard library with a focus on correctness, testability, and per-function isolation rather than monolithic build.
- The public-only-headers approach means no implementation details leak through headers, enabling fully hermetic per-function testing.
- TableGen entrypoint definitions drive header generation and build inclusion; no function ships without a corresponding test.
- Two deployment modes: full build (standalone libc replacement) for hermetic/bare-metal use, and overlay mode (thin `.a` linked before system libc) for selective function replacement on a host system.
- Correctly rounded math functions (`sinf/cosf/expf/logf` and double variants) use Sollya-generated polynomials with extended-precision intermediates to guarantee 0.5 ULP error.
- GPU support (NVPTX/AMDGPU) provides device-callable math/string/`printf` via an RPC ring-buffer protocol for host-side I/O flushing.
- The test suite uses per-function hermetic unit tests, differential fuzzing against reference libm, and exhaustive float-range sweeping for `float`-precision math functions.
- Scudo integration (`LIBC_USE_SCUDO=ON`) replaces `malloc`/`free` with the hardened allocator for memory-safe full-build deployments.
- LLVM 22 reaches production readiness for Linux x86_64 full builds, GPU math/string, and bare-metal string/math; `stdio` and `pthread` are functional on Linux but not yet all platforms.
