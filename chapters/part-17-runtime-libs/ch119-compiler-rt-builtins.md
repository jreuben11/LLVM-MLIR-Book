# Chapter 119 — compiler-rt Builtins

*Part XVII — Runtime Libraries*

compiler-rt is the LLVM project's replacement for libgcc: a collection of low-level runtime support routines required by code that `clang` emits. Understanding compiler-rt is essential when cross-compiling to bare-metal targets, integrating sanitizers into a custom toolchain, or reasoning about where the handful of arithmetic and intrinsic functions called by compiler-generated code actually live. This chapter covers the builtin library — the arithmetic, integer, floating-point, and utility routines — then turns to the instrumentation runtimes that live in the same tree: profile, coverage, and the sanitizer support already treated in depth in [Chapter 110 — User-Space Sanitizers](../part-16-jit-sanitizers/ch110-user-space-sanitizers.md) and [Chapter 112 — Production Allocators: Scudo and GWP-ASan](../part-16-jit-sanitizers/ch112-production-allocators-scudo-gwp-asan.md).

---

## 19.1 Repository Layout and Build System

compiler-rt lives under `compiler-rt/` in the LLVM monorepo. It is a `LLVM_ENABLE_RUNTIMES` component, meaning it is built against an already-installed (or in-tree-built) LLVM/Clang rather than as part of LLVM itself:

```bash
cmake -G Ninja \
  -DLLVM_ENABLE_RUNTIMES="compiler-rt" \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  ../llvm
ninja runtimes
```

The top-level `compiler-rt/CMakeLists.txt` dispatches into sub-libraries:

```
compiler-rt/
├── lib/
│   ├── builtins/          # libclang_rt.builtins-<arch>.a
│   ├── asan/              # libclang_rt.asan-<arch>.so/.a
│   ├── msan/
│   ├── tsan/
│   ├── ubsan/
│   ├── hwasan/
│   ├── profile/           # libclang_rt.profile-<arch>.a
│   ├── coverage/          # not a separate lib; see profile/
│   ├── fuzzer/            # libclang_rt.fuzzer-<arch>.a
│   ├── scudo/             # libclang_rt.scudo-<arch>.so
│   └── gwp_asan/
├── include/
│   └── sanitizer/         # public sanitizer headers
└── cmake/
    └── Modules/
```

The builtins library is the only one linked by default; all others require explicit opt-in via `-fsanitize=`, `-fprofile-instr-generate`, or `-coverage`.

The library that clang selects at link time is determined by the `--rtlib=` driver flag (defaulting to `compiler-rt` for most LLVM-managed toolchains, to `libgcc` when building against a GCC sysroot). The filename follows the pattern `libclang_rt.builtins-<triple-arch>.a`.

### 19.1.1 C ABI Requirement

All builtins are pure C ABI functions — no C++ name mangling, no C++ exceptions, no global constructors. This is a hard requirement because builtins must be callable from C code, assembly, and very-early startup code before the C++ runtime is initialized. The source files are `.c` or `.S`; there are no `.cpp` files in `lib/builtins/`.

---

## 19.2 Arithmetic Builtins

### 19.2.1 Integer Division

The C standard mandates that integer division and modulo produce specific results even on hardware that lacks a native divide instruction (many embedded CPUs). compiler-rt provides:

| Symbol | Description |
|--------|-------------|
| `__divsi3` | Signed 32-bit divide |
| `__udivsi3` | Unsigned 32-bit divide |
| `__modsi3` | Signed 32-bit modulo |
| `__umodsi3` | Unsigned 32-bit modulo |
| `__divdi3` | Signed 64-bit divide |
| `__udivdi3` | Unsigned 64-bit divide |
| `__moddi3` | Signed 64-bit modulo |
| `__umoddi3` | Unsigned 64-bit modulo |
| `__divti3` | Signed 128-bit divide |
| `__udivti3` | Unsigned 128-bit divide |

The 32-bit variants use a shift-and-subtract loop optimized with the `__builtin_clz`-based leading-zero normalization. The 64-bit variants on 32-bit hosts compute `u64 / u64` by decomposing into a series of 32-bit operations:

```c
// compiler-rt/lib/builtins/udivdi3.c (simplified)
uint64_t __udivdi3(uint64_t a, uint64_t b) {
    if (b == 0) compilerrt_abort("division by zero");
    if (b > a) return 0;
    uint32_t b0 = (uint32_t)b;
    if (b0 == b) {          // b fits in 32 bits
        if (b0 == 1) return a;
        int sr = __builtin_clz(b0) - __builtin_clz((uint32_t)(a >> 32));
        if (sr > 31) return 0;
        // ...shift-based loop...
    }
    // Full 64-bit path via udivmoddi4
    return __udivmoddi4(a, b, NULL);
}
```

Source: [udivdi3.c](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/lib/builtins/udivdi3.c)

### 19.2.2 128-bit Integer Operations

On 64-bit targets, LLVM IR supports `i128` arithmetic. The compiler lowers `i128` multiplication, division, and shifts to calls when hardware support is absent or when the ABI requires soft-emulation:

| Symbol | Operation |
|--------|-----------|
| `__multi3` | 128-bit multiply |
| `__divti3` | Signed 128-bit divide |
| `__udivti3` | Unsigned 128-bit divide |
| `__modti3` | Signed 128-bit modulo |
| `__umodti3` | Unsigned 128-bit modulo |
| `__ashlti3` | 128-bit arithmetic shift left |
| `__ashrti3` | 128-bit arithmetic shift right |
| `__lshrti3` | 128-bit logical shift right |

The 128-bit multiply uses the schoolbook two-limb decomposition:

```c
// __multi3: returns (a * b) with 128-bit wrapping
ti_int __multi3(ti_int a, ti_int b) {
    utwords x, y, r;
    x.all = a; y.all = b;
    r.all = (tu_int)x.s.low * y.s.low;
    r.s.high += (tu_int)x.s.high * y.s.low
              + (tu_int)x.s.low  * y.s.high;
    return r.all;
}
```

Note that only the low 128 bits are computed — the upper cross-terms are discarded, giving correct wrapping semantics.

Source: [multi3.c](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/lib/builtins/multi3.c)

### 19.2.3 Bit Manipulation

Portable implementations of `__builtin_popcount`, `__builtin_clz`, `__builtin_ctz`, and `__builtin_bswap` for targets that don't have single instructions:

| Symbol | Semantics |
|--------|-----------|
| `__popcountsi2` | Population count, 32-bit |
| `__popcountdi2` | Population count, 64-bit |
| `__clzsi2` | Count leading zeros, 32-bit |
| `__clzdi2` | Count leading zeros, 64-bit |
| `__ctzsi2` | Count trailing zeros, 32-bit |
| `__ctzdi2` | Count trailing zeros, 64-bit |
| `__bswapsi2` | Byte swap, 32-bit |
| `__bswapdi2` | Byte swap, 64-bit |
| `__parity{si,di,ti}2` | Parity (XOR of all bits) |
| `__ffs{si,di}2` | Find-first-set |

---

## 19.3 Soft-Float Builtins

Soft-float support is critical for microcontrollers without FPUs (Cortex-M0, many RISC-V bare-metal targets). The naming convention follows the GCC ABI:

- `sf` suffix → single precision (`float`)
- `df` suffix → double precision (`double`)
- `tf` suffix → quad precision (`__float128`/`long double` where 128-bit)
- `xf` suffix → x87 80-bit extended (`long double` on i386/x86_64)

### 19.3.1 Arithmetic Operations

```
__addsf3, __adddf3, __addtf3   – addition
__subsf3, __subdf3, __subtf3   – subtraction
__mulsf3, __muldf3, __multf3   – multiplication
__divsf3, __divdf3, __divtf3   – division
__negsf2, __negdf2, __negtf2   – negation
```

The single-precision multiply in `compiler-rt` uses an unpacked representation (separated sign, exponent, and mantissa fields) to avoid intermediate overflow, then renormalizes:

```c
// Simplified from lib/builtins/mulsf3.c
fp_t __mulsf3(fp_t a, fp_t b) {
    rep_t aRep = toRep(a), bRep = toRep(b);
    int aExp = extractExpSF(aRep) - exponentBias;
    int bExp = extractExpSF(bRep) - exponentBias;
    rep_t aMant = extractMantissaSF(aRep) | implicitBit;
    rep_t bMant = extractMantissaSF(bRep) | implicitBit;

    // Widen and multiply mantissas
    wide_t product = (wide_t)aMant * bMant;
    int productExp = aExp + bExp + exponentBias;
    // Normalize, pack result
    return fromRep(normalize(product, productExp, sign));
}
```

Source: [mulsf3.c](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/lib/builtins/mulsf3.c)

### 19.3.2 Conversion Builtins

Type conversions between FP and integer types:

| Symbol | Conversion |
|--------|-----------|
| `__fixsfsi` | `float` → `int32_t` |
| `__fixdfsi` | `double` → `int32_t` |
| `__fixsfdi` | `float` → `int64_t` |
| `__fixdfdi` | `double` → `int64_t` |
| `__fixunssfsi` | `float` → `uint32_t` |
| `__floatsisf` | `int32_t` → `float` |
| `__floatsidf` | `int32_t` → `double` |
| `__floatdisf` | `int64_t` → `float` |
| `__floatdidf` | `int64_t` → `double` |
| `__extendsfdf2` | `float` → `double` |
| `__truncdfsf2` | `double` → `float` |

### 19.3.3 Comparison Builtins

Floating-point comparisons return integer results that satisfy the C relational operators. The tricky part is NaN semantics:

```
__eqsf2, __eqdf2     – return 0 if equal (!=0 if NaN)
__nesf2, __nedf2     – return nonzero if not equal
__ltsf2, __ltdf2     – return negative if a < b
__lesf2, __ledf2     – return nonzero if a <= b
__gtsf2, __gtdf2     – return positive if a > b
__gesf2, __gedf2     – return nonzero if a >= b
__unordsf2, __unorddf2 – return nonzero if either is NaN
```

### 19.3.4 BFloat16 and FP16 Conversions

LLVM 22 supports `bfloat16` and `half` types natively. compiler-rt provides software conversion routines for targets without hardware support:

```
__truncsfbf2     – float → bfloat16
__truncdfbf2     – double → bfloat16
__truncsfhf2     – float → half
__extendbfsf2    – bfloat16 → float
__extendhfsf2    – half → float
__extendhfdf2    – half → double
```

---

## 19.4 Assembly-Optimized Hot Paths

For performance-critical targets, compiler-rt ships hand-written assembly that replaces the portable C fallbacks:

```
compiler-rt/lib/builtins/
├── arm/          # Thumb-1, Thumb-2, ARM
│   ├── udivsi3.S
│   ├── divmodsi4.S
│   ├── aeabi_*.S # ARM EABI soft-float ABI
│   └── ...
├── aarch64/
│   ├── __arm_softbp*.S
│   └── ...
├── i386/
│   ├── divdi3.S
│   └── ...
└── x86_64/
    └── (mostly covered by hardware; few .S files)
```

The ARM EABI deserves special attention. When targeting ARM bare-metal with `-mfloat-abi=softfp` or `-mfloat-abi=soft`, the compiler calls `__aeabi_*` wrappers rather than the generic `__addsf3`-style names. The ARM-specific directory provides these wrappers:

```asm
// compiler-rt/lib/builtins/arm/aeabi_fadd.S
DEFINE_COMPILERRT_FUNCTION(__aeabi_fadd)
    push    {r4, lr}
    bl      __addsf3          // tail-call the generic implementation
    pop     {r4, pc}
END_COMPILERRT_FUNCTION(__aeabi_fadd)
```

Targets with hardware divide (ARMv7-A with UDIV/SDIV instructions) will pick up the `.S` files that emit a single hardware instruction rather than the C loop.

---

## 19.5 Profile Instrumentation Runtime

When compiling with `-fprofile-instr-generate` (PGO) or `-fcoverage-mapping` (source coverage), clang injects calls to `__llvm_profile_*` functions provided by `libclang_rt.profile`.

### 19.5.1 Data Layout

Every instrumented function gets a statically allocated `__llvm_prf_data` record embedded in the `__llvm_prf_data` section:

```c
// Simplified from compiler-rt/lib/profile/InstrProfData.inc
typedef struct __llvm_profile_data {
    const uint64_t  NameRef;      // hash of mangled function name
    const uint64_t  FuncHash;     // structural hash of function CFG
    const IntPtrT   CounterPtr;   // relative offset to counters
    const IntPtrT   BitmapPtr;    // MC/DC bitmaps (LLVM 22+)
    const void*     FunctionPointer;
    const void*     Values;       // value profiling slots
    const uint32_t  NumCounters;
    const uint16_t  NumValueSites;
    const uint16_t  NumBitmapBytes;
} __llvm_profile_data;
```

Source: [InstrProf.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/lib/profile/InstrProf.h)

### 19.5.2 Counter Storage Models

LLVM 22 supports three counter storage models selectable via `-fprofile-update=`:

| Model | Mechanism | Thread Safety | Overhead |
|-------|-----------|--------------|---------|
| `atomic` (default) | `atomicrmw add` on counter arrays | Yes | ~1.5× |
| `prefer-atomic` | Atomic if target supports it, single otherwise | Best-effort | ~1.2× |
| `single` | Plain store (unsafe with threads) | No | ~1.0× |

The counters are 64-bit integers stored in a section named `__llvm_prf_cnts`. The profile writer runtime (`__llvm_profile_write_file`) serializes the raw format (`.profraw`) at program exit via an `atexit` handler registered by `__llvm_profile_init`.

### 19.5.3 Continuous Mode

For long-running servers, the profile data can be mmap'd directly to the output file, enabling continuous, crash-safe updates without an explicit flush. Enabled with `-fprofile-instr-generate=... -Xclang -fprofile-instrument=... -fprofile-continuous`.

### 19.5.4 Coverage Mapping

Source-based coverage (`-fcoverage-mapping`) adds a `__llvm_covmap` section containing compressed coverage mapping records alongside the counter data. The `llvm-cov` tool reads both sections to produce per-line and per-region coverage reports. See [Chapter 115 — Source-Based Code Coverage](../part-16-jit-sanitizers/ch115-source-based-code-coverage.md) for the full coverage workflow.

---

## 19.6 Sanitizer Support Infrastructure

The instrumentation runtimes for ASan, MSan, TSan, and UBSan live in `compiler-rt/lib/` as separate subdirectories, each producing a shared or static library. The high-level architecture was covered in [Chapter 110](../part-16-jit-sanitizers/ch110-user-space-sanitizers.md); here we note compiler-rt structural points:

### 19.6.1 sanitizer_common

All sanitizer runtimes share a foundational layer: `compiler-rt/lib/sanitizer_common/`. This provides:

- Platform abstraction (Linux/macOS/Windows/Fuchsia/Android)
- Memory allocation via `MmapOrDie` / `UnmapOrDie`
- Signal handlers and stack unwinding
- `Printf`-compatible formatting without libc dependency
- Symbolization via `DlAddrSymbolizer`, `LLVMSymbolizer`, or a subprocess protocol
- `SanitizerToolName`, `Die()`, `CheckFailed()`, `Report()`

The choice to avoid libc for sanitizer output is intentional: a sanitizer that calls `malloc` internally to format its error message can deadlock or corrupt state if it is catching a `malloc` bug.

### 19.6.2 Interceptor Mechanism

Many sanitizers intercept standard library functions (e.g., `memcpy`, `malloc`, `pthread_create`). The interception mechanism varies by platform:

- **Linux**: The sanitizer DSO is loaded before libc via `LD_PRELOAD` or the `-z interpose` linker flag; the interceptors are simply higher-priority symbols.
- **macOS**: `libinterpose.dylib` uses `DYLD_INSERT_LIBRARIES`; interception through the `INTERPOSE` section.
- **Static linking**: The sanitizer archive is linked first; interceptors are defined as strong symbols that override the libc versions.

The macro-based interception API:

```cpp
// sanitizer_common/sanitizer_interceptors.h (sketch)
#define INTERCEPTOR(ret_type, func, ...)            \
    ret_type WRAP(func)(__VA_ARGS__);               \
    ret_type func(__VA_ARGS__)                      \
      __attribute__((alias(STR(WRAP(func)))));      \
    ret_type WRAP(func)(__VA_ARGS__)
```

---

## 19.7 Weak and Init Functions

### 19.7.1 __attribute__((constructor)) and CRT Init

The sanitizer runtimes register themselves at startup using `__attribute__((constructor))` (priority 1) to ensure they run before user constructors. This relies on the `.init_array` / `.preinit_array` linker sections — which means compiler-rt makes assumptions about the host ELF linker behavior. Bare-metal toolchains that don't process `.init_array` will not get sanitizer initialization.

### 19.7.2 Weak Symbols for Customization

compiler-rt exposes customization hooks via weak symbols:

```c
// Override to redirect sanitizer reports
void __sanitizer_report_error_summary(const char *error_summary)
    __attribute__((weak));

// Override to customize OOM behavior
void __sanitizer_handle_no_memory(void) __attribute__((weak));

// For ASan: customize quarantine size
uint64_t __asan_get_estimated_allocated_size(uint64_t size)
    __attribute__((weak));
```

---

## 19.8 Bare-Metal and Cross-Compilation

### 19.8.1 Multilib Layout

When building a cross-toolchain, the builtins library must be compiled for each target triple and ABI variant. LLVM's multilib support selects the correct variant at link time based on the target triple encoded in the library filename.

For a bare-metal AArch64 target:
```bash
clang --target=aarch64-none-elf -ffreestanding -fno-exceptions \
      -rtlib=compiler-rt \
      hello.c -o hello
# links: libclang_rt.builtins-aarch64.a
```

### 19.8.2 Freestanding Environments

In freestanding mode (`-ffreestanding`), the compiler cannot assume libc is available. compiler-rt's builtins are designed for this: they have no libc dependencies. The `abort()` call used in division-by-zero checks is the only exception, but it can be replaced by linking against a stub.

For deeply embedded targets, a minimal link line is:
```bash
clang --target=armv7m-none-eabi \
      -mfloat-abi=softfp \
      -mfpu=none \
      -ffreestanding \
      -nostdlib \
      -rtlib=compiler-rt \
      crt0.S main.c \
      -lclang_rt.builtins-armv7m \
      -o firmware.elf
```

### 19.8.3 Builtins vs libgcc

The functional difference between compiler-rt builtins and libgcc is minimal — both implement the same GCC runtime ABI. The practical differences:

| Aspect | compiler-rt | libgcc |
|--------|-------------|--------|
| License | Apache 2 with LLVM exception | GPL with runtime exception |
| 128-bit int | Yes (`__int128`) | Partial |
| `__float128` | Yes | GCC-only extension |
| BFloat16 | Yes | No |
| AEABI (ARM) | Full coverage | Full coverage |
| Windows SEH | Yes | Limited |
| Bare-metal | Preferred | Traditional |

---

## 19.9 Calling Convention Nuances

### 19.9.1 The `__attribute__((used))` Pattern

Because builtins are called via soft-symbol linkage (no `#include` declaration), they must not be dead-stripped. compiler-rt marks them:

```c
COMPILER_RT_ALIAS(__udivsi3, __aeabi_uidiv);
```

The `COMPILER_RT_ALIAS` macro expands to a `.weak` definition and a `SET` symbol assignment in assembly, preventing the linker from discarding the symbol.

### 19.9.2 AArch64 and the EABI

AArch64 does not have an ARM EABI (`__aeabi_*`) since those names were defined for 32-bit ARM. The 64-bit AArch64 target uses the standard Itanium-derived ABI for function calls. compiler-rt builtins on AArch64 are primarily needed for 128-bit integer operations and soft-float on targets that have FP disabled.

---

## 19.10 Building compiler-rt for Custom Toolchains

A common workflow when setting up a new toolchain target:

```cmake
# CMakeCache entries for a cross build of compiler-rt builtins only
set(CMAKE_C_COMPILER   "/path/to/clang" CACHE PATH "")
set(COMPILER_RT_DEFAULT_TARGET_TRIPLE "riscv32-unknown-none-elf" CACHE STRING "")
set(COMPILER_RT_BAREMETAL_BUILD ON CACHE BOOL "")
set(COMPILER_RT_BUILD_BUILTINS ON CACHE BOOL "")
set(COMPILER_RT_BUILD_SANITIZERS OFF CACHE BOOL "")
set(COMPILER_RT_BUILD_XRAY OFF CACHE BOOL "")
set(COMPILER_RT_BUILD_LIBFUZZER OFF CACHE BOOL "")
set(COMPILER_RT_BUILD_PROFILE OFF CACHE BOOL "")
```

The key variable is `COMPILER_RT_BAREMETAL_BUILD`: it disables OS-dependent features (shadow maps, interceptors, symbolization subprocesses) and produces a minimal static archive suitable for bare-metal linking.

---

## Chapter Summary

- compiler-rt replaces libgcc; it is a pure C ABI library with no C++ dependencies, safe for freestanding use.
- The builtins library provides integer division/modulo (32/64/128-bit), soft-float arithmetic/conversion/comparison, and bit-manipulation polyfills for targets lacking hardware support.
- ARM targets require `__aeabi_*` wrappers; hand-written `.S` files in `arm/` replace the C fallbacks where hardware divide instructions are available.
- BFloat16 and FP16 conversion builtins (`__truncsfbf2`, `__extendhfsf2`) are provided for LLVM 22's expanded type support.
- The profile runtime (`libclang_rt.profile`) stores counters in ELF sections and serializes `.profraw` on exit; it supports atomic, prefer-atomic, and single-threaded counter update modes, plus continuous mmap mode.
- Sanitizer runtimes share `sanitizer_common` for platform abstraction, signal handling, and symbolization; they intercept libc functions via platform-specific mechanisms (LD_PRELOAD on Linux, DYLD_INSERT_LIBRARIES on macOS).
- Bare-metal toolchains set `COMPILER_RT_BAREMETAL_BUILD=ON` to suppress OS-dependent features and produce a minimal static builtins archive.
- The Apache 2.0 license (with LLVM exception) makes compiler-rt suitable for proprietary firmware, unlike GPL-licensed libgcc.


---

@copyright jreuben11
