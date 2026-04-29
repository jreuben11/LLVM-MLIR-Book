# Chapter 121 — libc++

*Part XVII — Runtime Libraries*

libc++ is LLVM's implementation of the ISO C++ standard library. It targets the C++03 through C++26 standards and ships as the default C++ standard library in Apple's toolchain, Android's NDK, and most LLVM-based hermetic toolchains. Unlike libstdc++ (GCC's library), libc++ was designed from the start for modularity, ABI versioning, and embedding in proprietary toolchains. This chapter covers the architecture, the inline-namespace ABI versioning trick, hardening modes introduced in LLVM 22, C++ modules support, the PSTL backend abstraction, and the practical aspects of building and deploying libc++ in custom toolchains.

---

## 21.1 Repository Layout

libc++ lives under `libcxx/` in the LLVM monorepo:

```
libcxx/
├── include/                  # Public headers (installed)
│   ├── __algorithm/          # Algorithm internals split by category
│   ├── __atomic/             # Atomic primitives
│   ├── __config               # Configuration macros
│   ├── __format/             # std::format implementation
│   ├── __memory/             # allocator, pointer traits, shared_ptr
│   ├── __ranges/             # C++20 ranges
│   ├── __type_traits/        # Type trait metafunctions
│   ├── algorithm, vector, string... # Umbrella headers
│   └── module.modulemap      # Clang module map
├── src/                      # .cpp files (compiled into the library)
│   ├── algorithm.cpp
│   ├── charconv.cpp
│   ├── filesystem/
│   ├── locale.cpp
│   ├── memory.cpp
│   └── ...
├── modules/                  # C++23 std/std.compat module sources
├── test/                     # lit-based test suite
├── benchmarks/               # Google Benchmark-based perf tests
└── docs/
    ├── Hardening.rst
    ├── ABIStability.rst
    └── UsingLibcxx.rst
```

The split between `include/` (headers installed at build time) and `include/__*/` (implementation headers, also installed) reflects an intentional architectural choice: users `#include <algorithm>` which picks up `include/algorithm`, which includes `include/__algorithm/sort.h`, etc. This allows selective specialization and avoids the monolithic-header ODR problems that plagued older standard library implementations.

---

## 21.2 ABI Versioning and the Inline-Namespace Trick

### 21.2.1 The Problem

Standard library symbols are part of the public ABI. When a new C++ standard adds a member to `std::string` or changes the layout of `std::vector<bool>`, binaries compiled against different library versions must not silently mix. libstdc++ solved this with symbol versioning scripts; libc++ took a different approach.

### 21.2.2 Inline Namespaces for ABI Version

All libc++ symbols live inside an inline namespace whose name encodes the ABI version:

```cpp
// include/__config (simplified)
#ifdef _LIBCPP_BUILDING_LIBRARY
#  define _LIBCPP_BEGIN_NAMESPACE_STD  namespace std { inline namespace __2 {
#  define _LIBCPP_END_NAMESPACE_STD    }}
#else
#  define _LIBCPP_BEGIN_NAMESPACE_STD  namespace std { inline namespace __2 {
#  define _LIBCPP_END_NAMESPACE_STD    }}
#endif
```

Because the namespace is `inline`, user code writes `std::string` and finds `std::__2::string` transparently. But the mangled symbol name is `std::__2::string`, not `std::string`. If a future version changes the layout, it ships as `std::__3::string`, and binaries using the two are link-time incompatible — a deliberate choice to fail noisily rather than silently corrupt.

### 21.2.3 ABI Stability Policy

libc++ maintains a stable ABI within a major version. The `LIBCXX_ABI_VERSION` CMake variable controls which inline namespace is compiled in. The current stable ABI is version 2 (`__2`); an experimental version 3 (`__3`) that incorporates C++23/26 layout changes can be opted into with `LIBCXX_ABI_UNSTABLE=ON`.

The ABI compatibility documentation lives at [ABIStability.rst](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/libcxx/docs/ABIStability.rst).

---

## 21.3 Configuration Macros

libc++ behavior is controlled by a set of `_LIBCPP_*` macros defined in `include/__config`:

| Macro | Effect |
|-------|--------|
| `_LIBCPP_VERSION` | Library version (e.g., `220100` for 22.1.0) |
| `_LIBCPP_STD_VER` | Effective C++ standard year (17, 20, 23, 26) |
| `_LIBCPP_HAS_NO_THREADS` | Disable all threading support |
| `_LIBCPP_HAS_NO_FILESYSTEM` | Disable `<filesystem>` |
| `_LIBCPP_DISABLE_AVAILABILITY` | Ignore `[[clang::availability]]` annotations |
| `_LIBCPP_ENABLE_DEBUG_MODE` | Enable iterator validity checks (deprecated in 22) |
| `_LIBCPP_HARDENING_MODE` | Set hardening level (see §21.4) |
| `_LIBCPP_PSTL_BACKEND_SERIAL` | Force serial PSTL backend |
| `_LIBCPP_PSTL_BACKEND_STD_THREAD` | Use `std::thread` PSTL backend |
| `_LIBCPP_PSTL_BACKEND_LIBDISPATCH` | Use GCD/libdispatch PSTL backend |

Source: [__config](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/libcxx/include/__config)

---

## 21.4 Hardening Modes

LLVM 21 introduced a unified hardening framework replacing the ad-hoc `_LIBCPP_ENABLE_DEBUG_MODE` and `_LIBCPP_ENABLE_ASSERTIONS` macros. LLVM 22 stabilizes this framework.

### 21.4.1 The Four Hardening Levels

Set via `-D_LIBCPP_HARDENING_MODE=_LIBCPP_HARDENING_MODE_<LEVEL>`:

| Level | Macro Value | Checks Enabled | Overhead |
|-------|-------------|----------------|---------|
| `none` | `_LIBCPP_HARDENING_MODE_NONE` | None | 0% |
| `fast` | `_LIBCPP_HARDENING_MODE_FAST` | Critical UB-prevention (out-of-bounds on span/string_view, invalid iterator use) | <1% |
| `extensive` | `_LIBCPP_HARDENING_MODE_EXTENSIVE` | All `fast` checks plus iterator invalidation tracking, container invariant checks | ~5% |
| `debug` | `_LIBCPP_HARDENING_MODE_DEBUG` | All `extensive` checks plus slow O(n) assertions, internal consistency checks | Significant |

The check macro that fires when an assertion fails:

```cpp
// include/__assertion_handler (simplified)
#define _LIBCPP_ASSERT_VALID_ELEMENT_ACCESS(expr, msg)   \
    _LIBCPP_ASSERT_IMPL(_LIBCPP_HARDENING_MODE_FAST, expr, msg)
#define _LIBCPP_ASSERT_VALID_INPUT_RANGE(expr, msg)      \
    _LIBCPP_ASSERT_IMPL(_LIBCPP_HARDENING_MODE_EXTENSIVE, expr, msg)
```

Checks are split by category (element access, input range validity, external preconditions, internal invariants), each mapped to a minimum level. The `fast` mode only fires the cheapest, highest-value checks.

### 21.4.2 Custom Violation Handler

When a hardening check fails, the default handler prints a message and calls `__builtin_trap()`. Production codebases can override it:

```cpp
// Define before including any libc++ header:
#define _LIBCPP_VERBOSE_ABORT(fmt, ...) \
    my_security_log_and_abort(__FILE__, __LINE__, fmt, ##__VA_ARGS__)
```

Or at link time by providing a definition of `__libcpp_verbose_abort`:

```cpp
// my_abort_handler.cpp
extern "C" [[noreturn]] void __libcpp_verbose_abort(
    const char *format, ...) {
    // log to syslog, crash reporter, etc.
    __builtin_trap();
}
```

Source: [Hardening.rst](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/libcxx/docs/Hardening.rst)

---

## 21.5 C++ Modules Support

libc++ provides first-class support for C++23 named modules (`import std;` and `import std.compat;`).

### 21.5.1 Module Files

The module sources live under `libcxx/modules/`:

```
modules/
├── std.cppm       # Exports the 'std' module
├── std.compat.cppm # Exports 'std.compat' (adds C compat macros)
└── CMakeLists.txt
```

`std.cppm` is a module interface unit that re-exports all public headers via `export import` directives:

```cpp
// libcxx/modules/std.cppm (simplified structure)
module;
#include <__config>
export module std;
export import :algorithm;
export import :atomic;
export import :string;
// ... all standard headers as module partitions
```

### 21.5.2 Building with Modules

```bash
# Build libc++ with module support
cmake -DLIBCXX_INSTALL_MODULES=ON ...

# Compile user code using import std
clang++ -std=c++23 -stdlib=libc++ \
        -fmodules -fprebuilt-module-path=/path/to/libc++/modules \
        main.cpp -o main
```

The BMI (Binary Module Interface) files for `std` and `std.compat` are installed alongside the headers and library. They cache all template instantiations and declarations from the standard library, dramatically reducing compile time for header-heavy codebases when using `import std` instead of `#include`.

---

## 21.6 PSTL: Parallel Algorithms

The Parallel STL (`<execution>`) is implemented in libc++ through a backend abstraction layer.

### 21.6.1 Backend Architecture

```
User code: std::sort(std::execution::par, v.begin(), v.end())
                |
          libc++ <algorithm>
                |
     ┌──────────┴──────────────┐
     │  PSTL dispatch layer    │
     └──────────┬──────────────┘
        ┌───────┴──────────────────────┐
        │         Backend              │
        │ Serial | std::thread | GCD  │
        └──────────────────────────────┘
```

### 21.6.2 Available Backends

| Backend | CMake Variable | Description |
|---------|---------------|-------------|
| Serial | `_LIBCPP_PSTL_BACKEND_SERIAL` | Falls through to sequential algorithms |
| `std::thread` | `_LIBCPP_PSTL_BACKEND_STD_THREAD` | Spawns threads for parallel work |
| libdispatch (GCD) | `_LIBCPP_PSTL_BACKEND_LIBDISPATCH` | Uses Grand Central Dispatch (Apple platforms) |
| OpenMP | (experimental, out-of-tree) | Via `_OPENMP` pragma backend |

The backend is selected at build time and can be overridden per-TU via the macro. The default on Apple platforms is `libdispatch`; on Linux it is `std::thread`. The serial fallback is useful for embedded targets that include `<execution>` headers but shouldn't actually parallelize.

### 21.6.3 Work Granularity

The `std::thread` backend chunks work into `std::thread::hardware_concurrency()` slices. The chunk size for `std::for_each(par, ...)` is computed as:

```cpp
constexpr ptrdiff_t __chunk_size = 512; // tunable
ptrdiff_t chunk = std::max(__chunk_size,
                           distance / hardware_concurrency);
```

---

## 21.7 Key Implementation Details

### 21.7.1 std::string Small-Buffer Optimization

libc++'s `basic_string` uses a three-word representation. The SBO (Small Buffer Optimization) threshold is 22 bytes on 64-bit platforms (23 chars + null):

```
Layout (64-bit, little-endian):
  Short mode (len <= 22):
    byte[0]:   { is_long:1=0, length:7 }  (length in bits 7:1)
    byte[1..22]: characters
    byte[23]:  null terminator (implicit, byte[0].length shows count)

  Long mode:
    word 0: pointer to heap buffer (bottom bit = 1 → long mode)
    word 1: size (number of chars, not including null)
    word 2: capacity | long_flag
```

Source: [__string/__basic_string.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/libcxx/include/__string/layout_compatibility_check.h)

The SBO means most `std::string` objects holding short strings never touch the heap at all, making them viable for hot-path code.

### 21.7.2 std::vector Growth Policy

libc++'s `vector::push_back` grows by factor 2 (doubling capacity), which is the standard choice and provides amortized O(1) inserts. This differs from MSVC's libcxx which uses 1.5×. The growth factor is not configurable without rebuilding the library.

### 21.7.3 std::shared_ptr Control Block

The control block layout uses a single atomic `use_count` and a separate `weak_count`:

```cpp
struct __shared_count {
    long __use_count_;   // decrements → destructor, 0 → dealloc object
    long __weak_count_;  // decrements, 0 → dealloc control block
    // + virtual destructor table
};
```

`std::make_shared<T>` allocates object and control block in a single allocation to improve locality, which is why `make_shared` is preferred over `shared_ptr<T>(new T(...))`.

---

## 21.8 C++23 and C++26 Additions in LLVM 22

LLVM 22's libc++ ships complete implementations of several C++23 and early C++26 features:

### 21.8.1 std::format and std::print

`<format>` (C++20, complete since LLVM 14) and `<print>` (C++23) are fully implemented:

```cpp
#include <print>
#include <format>

// std::print — direct formatted output (no ostream)
std::print("Hello, {}!\n", name);
std::println(stderr, "Error: {:08x}", error_code);

// std::format — returns a std::string
auto s = std::format("{:>10.3f}", 3.14159);  // "     3.142"

// Format ranges (C++23)
std::vector<int> v{1, 2, 3};
std::print("{}\n", v);  // [1, 2, 3]
```

The `<format>` implementation in libc++ uses compile-time format-string parsing via `std::format_string<Args...>` to detect format errors at compile time and generate type-specialized formatters.

### 21.8.2 std::expected

`std::expected<T, E>` (C++23, `<expected>`) provides a value-or-error return type without throwing exceptions:

```cpp
#include <expected>

std::expected<int, std::string> parse_int(std::string_view s) {
    int result;
    auto [ptr, ec] = std::from_chars(s.data(), s.data() + s.size(),
                                      result);
    if (ec != std::errc{})
        return std::unexpected(std::string("parse error"));
    return result;
}

auto val = parse_int("42");
if (val)       std::print("Got {}\n", *val);
else           std::print("Error: {}\n", val.error());
```

libc++'s `expected` implementation uses a `union` between the value and error types, with a `bool __has_val_` discriminant. It is `constexpr` throughout, supporting compile-time error handling.

### 21.8.3 std::generator (C++23)

`std::generator<T>` is a stackless coroutine-based range that produces values lazily:

```cpp
#include <generator>

std::generator<int> fibonacci() {
    int a = 0, b = 1;
    while (true) {
        co_yield a;
        auto next = a + b;
        a = b;
        b = next;
    }
}

// Use with ranges
for (int x : fibonacci() | std::views::take(10))
    std::print("{} ", x);  // 0 1 1 2 3 5 8 13 21 34
```

`std::generator` is implemented using the coroutine frame infrastructure from `<coroutine>` (see [Chapter 27 — Coroutines and Atomics](../part-04-llvm-ir/ch27-coroutines-and-atomics.md)). It supports `std::ranges::input_range` so it composes with all range algorithms.

### 21.8.4 std::chrono Time Zone Database

C++20's `<chrono>` time zone support (`std::chrono::time_zone`, `std::chrono::zoned_time`) requires a time zone database. libc++ ships two implementations:

- **Embedded database** (`LIBCXX_ENABLE_TIME_ZONE_DATABASE=ON`): bundles the IANA tz database into the library binary.
- **System database** (default on Linux/macOS): reads from `/usr/share/zoneinfo` at runtime.

```cpp
#include <chrono>
using namespace std::chrono;

auto utc_now = system_clock::now();
auto tz = locate_zone("America/New_York");
zoned_time ny_time{tz, utc_now};
std::print("New York: {}\n", ny_time);
```

The `locate_zone` function searches the tz database in O(log n) using a static sorted table.

---

## 21.9 C++23 Flat Containers and mdspan

### 21.9.1 std::mdspan

`std::mdspan<T, Extents, Layout, Accessor>` (C++23, `<mdspan>`) is a non-owning multidimensional array view:

```cpp
#include <mdspan>
#include <vector>

std::vector<float> data(6 * 4);
auto mat = std::mdspan(data.data(),
                       std::extents<size_t, 6, 4>{});

// Row-major access:
mat[2, 3] = 1.0f;  // element at row 2, col 3

// Custom layout: column-major (Fortran order)
using ColMajor = std::layout_left;
auto col = std::mdspan<float, std::dextents<size_t, 2>,
                        ColMajor>(data.data(), 6, 4);
```

`mdspan` is zero-overhead: all indexing computes the flat offset at compile time for static extents, and uses a single multiply-add at runtime for dynamic extents. The Layout policy abstracts strided, column-major, and tiled storage. libc++ ships the standard layouts (`layout_right`, `layout_left`, `layout_stride`) and supports user-defined layouts.

### 21.9.2 std::flat_map and std::flat_set

C++23 `std::flat_map<K, V>` is a sorted-array-backed associative container (similar to Abseil's `btree_map` at small sizes):

```cpp
#include <flat_map>

std::flat_map<int, std::string> m;
m.insert({3, "three"});
m.insert({1, "one"});
m.insert({2, "two"});
// Internal storage: two sorted parallel vectors
//   keys:   [1, 2, 3]
//   values: ["one", "two", "three"]

// Lookup is binary search: O(log n) comparisons
// Range iteration is cache-friendly (no pointer chasing)
auto it = m.find(2);
```

Unlike `std::map` (red-black tree with heap allocations per node), `flat_map` stores keys and values in contiguous `std::vector`s. Insertions are O(n) (requires shifting), making it suited for read-heavy workloads built in batch. libc++ also provides `std::flat_set`, `std::flat_multimap`, and `std::flat_multiset`.

---

## 21.10 Availability Annotations (Apple Platforms)

On Apple platforms, libc++ features that depend on newer OS-level C++ runtime support are guarded with `[[clang::availability]]` annotations. For example, `std::bad_variant_access` requires a symbol in the system libc++.dylib:

```cpp
// include/variant (simplified)
class _LIBCPP_AVAILABILITY_BAD_VARIANT_ACCESS bad_variant_access
    : public exception { ... };

// include/__availability (annotation definition)
#define _LIBCPP_AVAILABILITY_BAD_VARIANT_ACCESS \
    __attribute__((availability(macos,introduced=10.14)))
```

When deploying to an older OS, calls that use such types generate a link-time error (with `_LIBCPP_DISABLE_AVAILABILITY` the check is suppressed, moving the failure to runtime).

---

## 21.11 Building and Deploying libc++

### 21.9.1 Standard Build

```bash
cmake -G Ninja \
  -DLLVM_ENABLE_RUNTIMES="libunwind;libcxxabi;libcxx" \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DLIBCXX_ABI_VERSION=2 \
  -DLIBCXX_HARDENING_MODE=fast \
  ../llvm
ninja cxx cxxabi unwind
ninja install-cxx install-cxxabi install-unwind
```

### 21.9.2 Standalone (Hermetic) Deployment

For a fully hermetic toolchain that does not rely on the system libc++:

```bash
clang++ -stdlib=libc++ \
        -Wl,-rpath,/opt/mylibc++/lib \
        -L/opt/mylibc++/lib \
        -lc++ -lc++abi -lunwind \
        main.cpp -o main
```

Or statically:

```bash
clang++ -stdlib=libc++ \
        -static-libstdc++ \
        -Wl,-Bstatic -lc++ -lc++abi -lunwind -Wl,-Bdynamic \
        main.cpp -o main
```

### 21.9.3 Embedded / No-OS Deployment

For bare-metal targets that need the subset of the C++ standard library that doesn't require OS support (no `<thread>`, no `<filesystem>`):

```cmake
set(LIBCXX_HAS_MUSL_LIBC    ON)   # or set LIBCXX_HAS_NEWLIB etc.
set(LIBCXX_ENABLE_THREADS    OFF)
set(LIBCXX_ENABLE_FILESYSTEM OFF)
set(LIBCXX_ENABLE_MONOTONIC_CLOCK OFF)
set(LIBCXX_ENABLE_SHARED     OFF)
```

This produces a static `libc++.a` with containers, algorithms, and utilities but no threading or filesystem primitives.

---

## 21.12 Testing Infrastructure

libc++'s test suite (`libcxx/test/`) uses LLVM's `lit` test runner with a custom configuration:

```bash
# Run the full test suite
ninja check-cxx

# Run a specific test file
llvm-lit libcxx/test/std/containers/sequences/vector/push_back.pass.cpp

# Run with a specific hardening level
llvm-lit --param hardening_mode=extensive \
         libcxx/test/std/
```

Tests are organized by the standard's chapter structure: `test/std/algorithms/`, `test/std/containers/`, `test/std/strings/`, etc. Each `.pass.cpp` test is a self-contained program that must compile and run successfully; `.compile.pass.cpp` tests must only compile; `.verify.cpp` tests verify expected compilation errors via `clang -verify`.

---

## Chapter Summary

- libc++ is LLVM's C++ standard library, targeting C++03 through C++26, serving as the default on Apple/Android and in hermetic toolchains.
- The inline-namespace ABI trick (`std::__2::`) makes ABI versions explicitly incompatible at link time rather than silently binary-incompatible.
- LLVM 22 stabilizes four hardening levels: `none`, `fast`, `extensive`, and `debug`, each enabling progressively more expensive runtime checks; `fast` mode adds <1% overhead with the highest-value safety assertions.
- C++ modules (`import std;`, `import std.compat;`) are first-class in libc++ 22, with installed BMI files and a `LIBCXX_INSTALL_MODULES=ON` build option.
- The PSTL backend abstraction supports serial, `std::thread`, and libdispatch (GCD) backends for `<execution>` parallel algorithms, selected at build time.
- `std::string` uses a 23-byte SBO threshold on 64-bit platforms; `std::shared_ptr` uses a virtual control block with separate use/weak counts; `make_shared` merges object and control block allocations.
- Availability annotations gate C++ runtime features on Apple OS version requirements; `_LIBCPP_DISABLE_AVAILABILITY` suppresses these for embedded/cross-platform builds.
- Bare-metal deployment is supported by disabling threads, filesystem, and shared-library builds via CMake flags, producing a static `libc++.a` suitable for RTOS or firmware use.
- The lit-based test suite (`ninja check-cxx`) is organized by standard chapter and supports per-test hardening level parameterization.


---

@copyright jreuben11
