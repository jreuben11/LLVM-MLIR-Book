# Chapter 196 — Cross-Language ABI Interoperability: Binding Generators and UniFFI

*Part XXVIII — Language Ecosystems, Emerging Targets, and Engineering Practice*

A modern production system is never monolingual. The graphics stack might be written in Rust for safety, the legacy numerical core in Fortran or C, the developer tooling in Python, and the mobile SDK in Swift and Kotlin. Making these components interoperate without introducing undefined behavior, memory corruption, or silent data misinterpretation is the applied discipline of foreign-function interface (FFI) engineering. Because all these languages ultimately compile through LLVM to the same machine code, the invariants that must hold across a language boundary are precisely those the compiler enforces within a single language — calling conventions, type layouts, and exception-propagation semantics — but now enforced manually, by the programmer and the binding tool, rather than automatically by a single type system.

This chapter builds on the ABI theory developed in [Chapter 23 — Attributes, Calling Conventions, and the ABI](../part-04-llvm-ir/ch23-attributes-calling-conventions-abi.md), the Clang code-generation perspective in [Chapter 41 — Calls, the ABI Boundary, and Builtins](../part-06-clang-codegen/ch41-calls-abi-builtins.md), and the C++ name-mangling treatment in [Chapter 42 — C++ ABI Lowering: Itanium](../part-06-clang-codegen/ch42-cpp-abi-itanium.md) and [Chapter 43 — C++ ABI Lowering: Microsoft](../part-06-clang-codegen/ch43-cpp-abi-microsoft.md). It treats the tools that automate FFI safety — `bindgen`, `cbindgen`, `cxx`, UniFFI, `wit-bindgen` — as engineering artifacts with a precise relationship to LLVM IR semantics.

---

## Table of Contents

- [The Three Invariants for Correct FFI](#the-three-invariants-for-correct-ffi)
- [`extern "C"` and Name Mangling](#extern-c-and-name-mangling)
  - [The Mangling Problem](#the-mangling-problem)
  - [Language-by-Language Syntax](#language-by-language-syntax)
  - [LLVM IR Representation](#llvm-ir-representation)
  - [Symbol Visibility and the Linker](#symbol-visibility-and-the-linker)
- [Type Layout Compatibility](#type-layout-compatibility)
  - [Rust `#[repr(...)]` Attributes](#rust-repr-attributes)
  - [Struct Argument Passing: `byval`, `sret`, `inalloca`](#struct-argument-passing-byval-sret-inalloca)
  - [Fat Pointers and Vtables Cannot Cross FFI Boundaries](#fat-pointers-and-vtables-cannot-cross-ffi-boundaries)
  - [Zig and Swift Layout](#zig-and-swift-layout)
- [bindgen — C/C++ Headers to Rust](#bindgen-cc-headers-to-rust)
  - [Architecture](#architecture)
  - [`build.rs` Integration](#buildrs-integration)
  - [Handling Tricky Cases](#handling-tricky-cases)
- [cbindgen — Rust API to C/C++](#cbindgen-rust-api-to-cc)
  - [What cbindgen Does](#what-cbindgen-does)
  - [Configuration](#configuration)
  - [Example Round-Trip](#example-round-trip)
  - [Build System Integration and CI Gating](#build-system-integration-and-ci-gating)
- [cxx — Zero-Overhead Rust/C++ Bridge](#cxx-zero-overhead-rustc-bridge)
  - [The Problem with Raw FFI](#the-problem-with-raw-ffi)
  - [Bridge Declaration](#bridge-declaration)
  - [Cross-Language Ownership Types](#cross-language-ownership-types)
  - [Exception Handling](#exception-handling)
  - [Build System Integration](#build-system-integration)
- [Swift/C++ Interoperability](#swiftc-interoperability)
  - [Direct C++ Import (Swift 5.9+)](#direct-c-import-swift-59)
  - [Exporting Swift to C and C++](#exporting-swift-to-c-and-c)
- [Python FFI](#python-ffi)
  - [ctypes](#ctypes)
  - [cffi](#cffi)
  - [pybind11 and nanobind](#pybind11-and-nanobind)
- [JNI — Java Native Interface](#jni-java-native-interface)
  - [Structure](#structure)
  - [Object Lifecycle](#object-lifecycle)
  - [Android NDK and ABI Splits](#android-ndk-and-abi-splits)
  - [Alternatives: Kotlin/Native cinterop and GraalVM](#alternatives-kotlinnative-cinterop-and-graalvm)
- [UniFFI — Mozilla's Multi-Language FFI Generator](#uniffi-mozillas-multi-language-ffi-generator)
  - [Overview](#overview)
  - [Interface Definition Modes](#interface-definition-modes)
  - [Binding Generation](#binding-generation)
  - [What the Generated Code Looks Like](#what-the-generated-code-looks-like)
  - [Production Use in Firefox](#production-use-in-firefox)
  - [Versioning and ABI Stability](#versioning-and-abi-stability)
- [WebAssembly Component Model and WIT](#webassembly-component-model-and-wit)
  - [The Component Model](#the-component-model)
  - [WIT Interface Definition Language](#wit-interface-definition-language)
  - [Rust Components with cargo-component](#rust-components-with-cargo-component)
  - [Hosting with wasmtime](#hosting-with-wasmtime)
  - [wasm-bindgen for the Web](#wasm-bindgen-for-the-web)
- [LLVM IR as a Potential Interchange Layer](#llvm-ir-as-a-potential-interchange-layer)
  - [The Universal-IR Thesis](#the-universal-ir-thesis)
  - [Apple Bitcode — The Only Large-Scale Deployment](#apple-bitcode-the-only-large-scale-deployment)
  - [LTO Bitcode as Cross-Language Optimization Substrate](#lto-bitcode-as-cross-language-optimization-substrate)
  - [Why LLVM IR Cannot Be a Stable FFI Contract](#why-llvm-ir-cannot-be-a-stable-ffi-contract)
  - [WebAssembly as the Realized Universal IR](#webassembly-as-the-realized-universal-ir)
  - [MLIR as the Structured-IR Alternative](#mlir-as-the-structured-ir-alternative)
- [Exception and Unwinding Interoperability](#exception-and-unwinding-interoperability)
  - [Four Incompatible Error Models](#four-incompatible-error-models)
  - [`extern "C-unwind"` (Rust 1.71)](#extern-c-unwind-rust-171)
  - [`catch_unwind` at FFI Boundaries](#catchunwind-at-ffi-boundaries)
  - [Choosing Between `panic=unwind` and `panic=abort`](#choosing-between-panicunwind-and-panicabort)
- [Fuzzing FFI Boundaries](#fuzzing-ffi-boundaries)
  - [cargo-fuzz at the Rust Side](#cargo-fuzz-at-the-rust-side)
  - [Sanitizers Across the FFI Boundary](#sanitizers-across-the-ffi-boundary)
  - [bindgen/cbindgen Round-Trip Testing](#bindgencbindgen-round-trip-testing)
- [FFI Approach Comparison](#ffi-approach-comparison)
- [Chapter Summary](#chapter-summary)

---

## The Three Invariants for Correct FFI

Two separately compiled translation units can call each other through an FFI boundary safely if and only if three invariants hold:

**Invariant 1: Calling convention match.** Both sides must agree on which registers carry arguments, which registers are caller-saved, how the stack frame is laid out, and where the return value is placed. LLVM encodes this in the `cc N` attribute on function definitions and call instructions. The C calling convention (`cc 0`, spelled `ccc` in LLVM IR) is the universal substrate: every language that participates in FFI must either use it natively or provide a shim that converts to it.

**Invariant 2: Type layout match.** Every scalar, struct, and enum passed across the boundary must have the same width, alignment, and field ordering on both sides. This is non-trivial even for primitive types: `long` is 32-bit on Windows and 64-bit on most Unix targets; `bool` is 1 byte in C but Rust's `bool` is also 1 byte, while a C++ `bool` in a struct may be padded differently. For structs the compiler is ordinarily free to reorder fields for performance; FFI disables this freedom.

**Invariant 3: Exception and panic isolation.** If the calling side uses C++ exceptions, Rust panics, or Swift typed errors and the callee does not expect them, the runtime unwinder will corrupt the stack of the callee's language runtime. Safe FFI either prevents panics/exceptions from crossing the boundary or explicitly negotiates an unwinding personality that both sides understand.

C occupies a privileged position: it is the only language with a universally stable ABI on every platform. Every language surveyed in this chapter achieves interoperability by mapping its own types and functions to C representations, using C's calling convention at the boundary, and optionally relying on C's lack of exceptions to isolate unwinding.

---

## `extern "C"` and Name Mangling

### The Mangling Problem

A linker resolves symbols by name. When Rust compiles `fn process(x: u64)` it emits a symbol whose name encodes the crate, module path, and type parameters — something like `_ZN7example7process17hd3c7a6e9e1f2b3c4E` (using a truncated hash). C++ uses Itanium mangling (covered in [Chapter 42 — C++ ABI Lowering: Itanium](../part-06-clang-codegen/ch42-cpp-abi-itanium.md)) or MSVC mangling ([Chapter 43 — C++ ABI Lowering: Microsoft](../part-06-clang-codegen/ch43-cpp-abi-microsoft.md)). Swift uses its own mangling. When two separately compiled libraries need to call each other, mangled names form an additional compatibility contract on top of type layout.

The solution is `extern "C"` in C++ and the equivalent construct in each language: opt into the C ABI, where function names are either unmangled or mangled only by prefixing an underscore on some targets (`_process` on macOS).

### Language-by-Language Syntax

**C++:**
```cpp
// Declares that these symbols use C linkage (no name mangling)
extern "C" {
    void process(uint64_t x);
    int  compute(const char* s, size_t len);
}
```
A C++ function defined inside `extern "C"` is callable from C, Rust, Swift, Zig, or any other language that can call C.

**Rust:**
```rust
// Importing a C function into Rust
extern "C" {
    fn process(x: u64) -> i32;
    fn compute(s: *const u8, len: usize) -> i32;
}

// Exporting a Rust function with C ABI and C-compatible name
#[no_mangle]
pub extern "C" fn rust_process(x: u64) -> i32 {
    // ...
    0
}
```
`#[no_mangle]` suppresses Rust's own symbol mangling. `extern "C"` selects the platform C calling convention. Both attributes are required for an exported function to be callable from C.

**Swift:**
```swift
// Export a Swift function with a stable C name
@_cdecl("swift_process")
public func processValue(_ x: UInt64) -> Int32 {
    return 0
}
```

**Zig:**
```zig
// export fn automatically applies C calling convention and suppresses Zig mangling
export fn zig_process(x: u64) i32 {
    return 0;
}
```

### LLVM IR Representation

The calling convention appears as a keyword immediately after `define` or `call`:

```llvm
; C ABI — cc 0 (the default, usually omitted)
define i32 @process(i64 %x) {
  ret i32 0
}

; Rust ABI (compiler-internal, not stable across versions)
define i32 @_ZN7example7process17hd3c7a6e9e1f2b3c4E(i64 %x) "rust-abi" {
  ret i32 0
}

; Swift calling convention — cc 16 (swiftcc)
define swiftcc i32 @swift_process(i64 %x) {
  ret i32 0
}
```

LLVM's [`CallingConv.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/CallingConv.h) enumerates all calling conventions. The critical ones for FFI are:
- `C` (0): the C calling convention — the FFI substrate
- `Fast` (8): LLVM's fastcc for internal use; not ABI-stable
- `Swift` (16): Swift's calling convention, used in `swiftcc`
- `X86_StdCall` (64), `X86_FastCall` (65): Windows-only C variants
- `Win64` (79): Microsoft x86-64 ABI

### Symbol Visibility and the Linker

The calling convention determines how arguments are passed; symbol visibility determines whether the linker can find the symbol at all. On Linux, shared library symbols default to `STV_DEFAULT` visibility (exported). Rust and C++ both trim their symbol tables: Rust hides non-`#[no_mangle]` symbols by default, and C++ hides symbols marked `__attribute__((visibility("hidden")))`.

Verifying that the symbols align requires inspection after the link step:

```bash
# Check that the expected C-ABI symbol is exported from a Rust .so
nm -D target/release/libmylib.so | grep rust_process
# Expected: 0000... T rust_process

# Check that no Rust-mangled internals leaked out
nm -D target/release/libmylib.so | grep '_ZN'
# Should be empty for a well-sealed library
```

LLD — covered in [Chapter 78 — The LLVM Linker (LLD)](../part-13-lto-whole-program/ch78-lld.md) — resolves FFI symbols at link time exactly as it would C symbols. When Rust is compiled with `-C link-arg=-Wl,--version-script=exports.map`, only the symbols listed in the version script are exported, preventing accidental exposure of internal Rust symbols that happen to be `pub`:

```
# exports.map — restrict the shared library's public surface
{
  global: rust_process; point_distance;
  local: *;
};
```

This linker-level filtering is the final defense against ABI leakage.

---

## Type Layout Compatibility

### Rust `#[repr(...)]` Attributes

By default, Rust is free to reorder struct fields to minimize padding. An FFI struct must opt into a specified layout:

```rust
// #[repr(C)]: fields in declaration order, C-compatible alignment and padding
#[repr(C)]
pub struct Point {
    pub x: f64,  // offset 0
    pub y: f64,  // offset 8
}

// #[repr(transparent)]: identical layout to the single non-ZST field
// Safe to transmute between Wrapper and f64
#[repr(transparent)]
pub struct NonNegative(f64);

// #[repr(packed)]: no padding between fields (may cause unaligned access)
#[repr(C, packed)]
pub struct PackedHeader {
    pub magic: u32,
    pub version: u16,
    pub flags: u8,
}
```

The niche optimization warrants particular attention. Rust represents `Option<Box<T>>` as a single pointer — null means `None`. This is correct Rust but a C caller examining the raw pointer type cannot see the `Option` wrapper. Similarly, `Option<NonNull<T>>` is a nullable pointer with zero overhead. The `#[repr(C)]` attribute disables niche optimizations for the type it annotates, forcing an explicit discriminant.

```rust
// WITHOUT #[repr(C)]: Rust may use niche; C sees a raw pointer with hidden semantics
pub struct Opaque(*mut u8);

// WITH #[repr(C)]: explicit, predictable layout; safe to pass to C
#[repr(C)]
pub struct ExplicitOption {
    pub present: u8,
    pub _pad: [u8; 7],
    pub ptr: *mut u8,
}
```

### Struct Argument Passing: `byval`, `sret`, `inalloca`

When a struct is passed as a function argument, the ABI determines whether it travels in registers or on the stack. This is precisely what [Chapter 41 — Calls, the ABI Boundary, and Builtins](../part-06-clang-codegen/ch41-calls-abi-builtins.md) analyses at the Clang level. The LLVM IR attributes are:

- `byval(%T)`: the argument is a pointer to a caller-allocated copy of the struct; callee must not modify the original
- `sret(%T)`: the first argument is a pointer where the return value is written (used for large struct returns)
- `inalloca(%T)`: the argument is allocated in the caller's stack frame and passed by address

```llvm
; Large struct passed by value on the stack
define void @take_large(%LargeStruct* byval(%LargeStruct) align 8 %s) { ... }

; Struct return via hidden pointer
define void @make_large(%LargeStruct* sret(%LargeStruct) align 8 %ret) { ... }
```

Getting these attributes right is what binding generators must solve correctly — a struct that is 16 bytes on one platform may be passed in two registers, while the same struct on another platform is passed on the stack, with entirely different LLVM IR encoding.

A concrete example shows the distinction. On x86-64 System V (Linux/macOS), a `Point { f64, f64 }` fits in two SSE registers and is passed without any stack allocation:

```llvm
; x86-64 SysV: Point returned in xmm0/xmm1 — no sret needed
define { double, double } @make_point(double %x, double %y) {
  %r0 = insertvalue { double, double } undef, double %x, 0
  %r1 = insertvalue { double, double } %r0, double %y, 1
  ret { double, double } %r1
}
```

On 32-bit x86 (or when the struct exceeds register capacity), the same return type requires an `sret` pointer:

```llvm
; x86-32 or large struct: caller allocates space, passes hidden pointer
define void @make_point(%Point* sret(%Point) align 8 %ret,
                        double %x, double %y) {
  %px = getelementptr %Point, %Point* %ret, i32 0, i32 0
  %py = getelementptr %Point, %Point* %ret, i32 0, i32 1
  store double %x, double* %px
  store double %y, double* %py
  ret void
}
```

A `bindgen`-generated binding that hard-codes the wrong calling-convention assumption will compile and link successfully but produce wrong values at runtime — the most insidious class of FFI bug.

### Fat Pointers and Vtables Cannot Cross FFI Boundaries

Rust trait objects (`&dyn Trait`, `Box<dyn Trait>`) are *fat pointers*: 16 bytes on 64-bit platforms, carrying a data pointer and a vtable pointer. There is no C-compatible representation for a fat pointer — the vtable layout is an internal Rust implementation detail, not part of any stable ABI. Attempting to pass a `*mut dyn Trait` across an FFI boundary is undefined behavior.

```rust
// WRONG: dyn Trait is not FFI-safe; rustc emits a warning
#[no_mangle]
pub extern "C" fn bad_api(cb: *mut dyn Fn(i32) -> i32) { ... }

// CORRECT: use a concrete function pointer + void* context
#[repr(C)]
pub struct Callback {
    pub func: unsafe extern "C" fn(ctx: *mut std::ffi::c_void, x: i32) -> i32,
    pub ctx:  *mut std::ffi::c_void,
}
```

The function-pointer-plus-context pattern is the universal C-compatible substitute for a vtable-based callback. C++ virtual dispatch faces the same limitation when crossing language boundaries: a C++ `virtual` class with a vtable cannot be used from Rust or Swift unless the virtual functions are re-exported through a C function table.

### Zig and Swift Layout

**Zig** provides three struct layouts:

```zig
// extern struct: C-compatible layout, C ABI
const CPoint = extern struct {
    x: f64,
    y: f64,
};

// packed struct: explicit bit-level packing, no implicit padding
const Header = packed struct {
    magic: u32,
    version: u16,
    flags: u8,
};

// default struct: implementation-defined (Zig may reorder for size)
const Internal = struct {
    x: f64,
    y: f64,
};
```

**Swift** `@objc` classes use the Objective-C runtime's `isa`-pointer layout and are reference-counted via `retain`/`release`. Swift structs with `@frozen` layout (the library evolution model) have a stable, documented binary layout. Non-frozen Swift structs are opaque to C callers.

---

## bindgen — C/C++ Headers to Rust

### Architecture

`bindgen` ([rust-lang.github.io/rust-bindgen](https://rust-lang.github.io/rust-bindgen/)) is the official Rust project for generating Rust FFI bindings from C and C++ headers. It operates by invoking `libclang` — the Clang C API described in [Chapter 46 — libtooling and AST Matchers](../part-07-clang-multilang/ch46-libtooling-ast-matchers.md) — to parse the header under the same preprocessor and type-resolution logic that Clang uses when compiling the corresponding C/C++ source. This ensures that `bindgen`'s view of the type layout, preprocessor macros, and anonymous structs matches what the C compiler actually produces.

The binding-generation pipeline:

1. Parse the header with `libclang`, visiting the AST
2. For each visited type, function, constant, and enum, compute the Rust equivalent, applying `#[repr(C)]`, `#[repr(u32)]`, etc.
3. For anonymous struct/union members, generate `__bindgen_anon_N` names
4. For opaque types, generate a zero-sized struct with private fields
5. Emit a `.rs` file to `OUT_DIR`; `include!()` in the Rust source pulls it in at compile time

### `build.rs` Integration

```rust
// build.rs
fn main() {
    // Tell cargo to re-run this script if the header changes
    println!("cargo:rerun-if-changed=wrapper.h");

    let bindings = bindgen::Builder::default()
        .header("wrapper.h")
        // Pass flags to libclang, same as passing to clang -c
        .clang_arg("-I/usr/include/mylib")
        .clang_arg("-DMYLIB_STATIC=1")
        // Allowlist: only generate bindings for these symbols
        .allowlist_function("foo_.*")
        .allowlist_type("Foo.*")
        .allowlist_var("FOO_VERSION")
        // Block types that Rust should not see (e.g., platform internals)
        .blocklist_type("__.*")
        // Treat this type as opaque (forward declaration only)
        .opaque_type("FooInternalState")
        // Derive standard traits where possible
        .derive_debug(true)
        .derive_default(true)
        .derive_eq(true)
        // Notify cargo of header file dependencies
        .parse_callbacks(Box::new(bindgen::CargoCallbacks::new()))
        .generate()
        .expect("Unable to generate bindings");

    let out_path = std::path::PathBuf::from(
        std::env::var("OUT_DIR").unwrap()
    );
    bindings
        .write_to_file(out_path.join("bindings.rs"))
        .expect("Couldn't write bindings!");
}
```

The corresponding `lib.rs` includes the generated file:

```rust
// src/lib.rs
#![allow(non_upper_case_globals, non_camel_case_types, non_snake_case)]

include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

### Handling Tricky Cases

**Anonymous structs and unions:**

```c
// C header
typedef struct {
    union {
        int i;
        float f;
    };
    double d;
} Value;
```
bindgen generates:
```rust
#[repr(C)]
pub struct Value {
    pub __bindgen_anon_1: Value__bindgen_ty_1,
    pub d: f64,
}
#[repr(C)]
pub union Value__bindgen_ty_1 {
    pub i: ::std::os::raw::c_int,
    pub f: f32,
}
```

**Variadic functions:**

```rust
// In the extern "C" block
extern "C" {
    pub fn printf(format: *const ::std::os::raw::c_char, ...) -> ::std::os::raw::c_int;
}
```

**Bitfields** are the most treacherous C construct for bindgen. C does not specify bitfield layout beyond width — field ordering within a storage unit, endianness of split fields, and padding between bitfield members are implementation-defined:

```c
// C header with bitfields
struct Flags {
    unsigned int a : 3;
    unsigned int b : 5;
    unsigned int c : 8;
};
```

bindgen cannot safely emit native Rust fields for bitfields (there is no Rust language-level bitfield syntax). Instead it generates a byte-array storage unit with generated accessor methods via `__bindgen_bitfield_unit`:

```rust
#[repr(C)]
pub struct Flags {
    pub _bitfield_align_1: [u32; 0],
    pub _bitfield_1: __BindgenBitfieldUnit<[u8; 4usize]>,
}
impl Flags {
    #[inline]
    pub fn a(&self) -> ::std::os::raw::c_uint {
        unsafe { ::std::mem::transmute(self._bitfield_1.get(0usize, 3u8) as u32) }
    }
    #[inline]
    pub fn set_a(&mut self, val: ::std::os::raw::c_uint) {
        unsafe { self._bitfield_1.set(0usize, 3u8, val as u64) }
    }
    // ... similar for b, c
}
```

The layout is determined by querying `libclang` for the exact bit offsets, so it matches whatever the C compiler would produce for the same target triple.

**Function pointer typedefs** in C become `Option<unsafe extern "C" fn(...)>` in Rust. The `Option` wrapper is required because C function pointers can be null:

```c
// C typedef for a callback
typedef int (*ProcessFn)(const char* data, size_t len, void* ctx);
```

bindgen generates:

```rust
pub type ProcessFn = ::std::option::Option<
    unsafe extern "C" fn(
        data: *const ::std::os::raw::c_char,
        len: usize,
        ctx: *mut ::std::os::raw::c_void,
    ) -> ::std::os::raw::c_int,
>;
```

The `Option<fn>` representation uses the niche optimization: the null function pointer serves as `None`, so `Option<ProcessFn>` is the same size as a raw pointer.

**Flexible array members** (C99 `struct { int n; char data[]; }`) generate a zero-length array in Rust; the user must use `unsafe` pointer arithmetic to access elements.

bindgen is used in the Linux kernel Rust bindings (`rust/kernel/`), Firefox's `gecko-media`, Servo's graphics stack, and the `libc` crate's platform binding layer.

---

## cbindgen — Rust API to C/C++

### What cbindgen Does

`cbindgen` ([github.com/mozilla/cbindgen](https://github.com/mozilla/cbindgen)) is the complement to `bindgen`: given a Rust crate exposing a `pub` API with `#[repr(C)]` types and `#[no_mangle] pub extern "C"` functions, it generates a C or C++ header. This header can then be used by C or C++ callers to link against the compiled Rust library.

The tool parses Rust source (not the compiled binary) to find:
- `#[repr(C)]` structs and enums
- `#[no_mangle]` functions with `extern "C"`
- `pub const` values

### Configuration

A `cbindgen.toml` in the crate root controls output:

```toml
language = "C"
include_guard = "MYLIB_H"
autogen_warning = "/* Warning: auto-generated by cbindgen. Do not edit. */"

[export]
include = ["MyStruct", "MyEnum", "my_function"]

[parse]
parse_deps = true
include = ["mylib-types"]
```

For C++ output:

```toml
language = "C++"
namespace = "mylib"
namespaces = ["mylib", "internal"]

[export.rename]
"MyStruct" = "my_struct_t"
```

### Example Round-Trip

Rust source:

```rust
#[repr(C)]
pub struct Point {
    pub x: f64,
    pub y: f64,
}

#[no_mangle]
pub extern "C" fn point_distance(a: *const Point, b: *const Point) -> f64 {
    let a = unsafe { &*a };
    let b = unsafe { &*b };
    ((a.x - b.x).powi(2) + (a.y - b.y).powi(2)).sqrt()
}
```

Generated C header (`cbindgen --lang c`):

```c
/* Warning: auto-generated by cbindgen. Do not edit. */

#ifndef MYLIB_H
#define MYLIB_H

#include <stdarg.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdlib.h>

typedef struct Point {
  double x;
  double y;
} Point;

double point_distance(const Point *a, const Point *b);

#endif  /* MYLIB_H */
```

cbindgen is used by Firefox WebRender, Mozilla NSS, the Gecko rendering pipeline, and the Servo web engine. Running `cbindgen --lang c++ --output include/mylib.hpp` produces a C++ header with `extern "C"` wrapping and optional namespace.

### Build System Integration and CI Gating

cbindgen is typically invoked either as a `build.rs` step or as a separate CI job that verifies the committed header is up to date. The `build.rs` approach regenerates the header on every build:

```rust
// build.rs — regenerate the C header automatically
fn main() {
    let crate_dir = std::env::var("CARGO_MANIFEST_DIR").unwrap();
    cbindgen::Builder::new()
        .with_crate(crate_dir)
        .with_language(cbindgen::Language::C)
        .generate()
        .expect("Unable to generate bindings")
        .write_to_file("include/mylib.h");
}
```

The CI-gating approach keeps the header in source control and fails the build if the committed header is stale:

```bash
# In CI: regenerate and diff against committed header
cbindgen --lang c -o /tmp/mylib_fresh.h
diff include/mylib.h /tmp/mylib_fresh.h || \
    (echo "cbindgen header is stale — run cbindgen and commit"; exit 1)
```

This ensures that the C/C++ callers always compile against a header that matches the Rust implementation, catching type mismatches at the header level rather than at runtime.

---

## cxx — Zero-Overhead Rust/C++ Bridge

### The Problem with Raw FFI

Raw `bindgen`/`cbindgen` FFI requires the programmer to manually ensure that every type passed across the boundary has a matching `#[repr(C)]` layout on the Rust side and is used correctly on the C++ side. C++ exceptions escaping into Rust or Rust panics escaping into C++ are undefined behavior. The cxx crate ([cxx.rs](https://cxx.rs)) addresses this by generating both Rust and C++ glue from a single shared type declaration, verifying at compile time that the bridge is internally consistent.

### Bridge Declaration

```rust
#[cxx::bridge]
mod ffi {
    // Types shared between Rust and C++
    struct Point {
        x: f64,
        y: f64,
    }

    // C++ functions callable from Rust
    unsafe extern "C++" {
        include!("mylib/widget.h");

        type Widget;  // Opaque C++ type

        fn new_widget(name: &str) -> UniquePtr<Widget>;
        fn render(self: Pin<&mut Widget>);
        fn name(self: &Widget) -> &CxxString;
    }

    // Rust functions callable from C++
    extern "Rust" {
        fn process_widget(w: &Widget) -> String;
        fn transform_points(points: &[Point]) -> Vec<Point>;
    }
}
```

cxx generates a `.rs` file with the Rust side of the bridge and a `.cpp` file with the C++ shim layer. The C++ shim for a function declared in `extern "Rust"` looks like:

```cpp
// Generated by cxx — excerpt of the C++ shim for process_widget
extern "C" {
  // Rust ABI function — generated scaffolding in Rust
  ::rust::repr::PtrLen mylib$cxxbridge1$process_widget(
      const ::Widget& w,
      ::rust::repr::PtrLen* return_ptr) noexcept;
}

::rust::String process_widget(const Widget& w) {
  ::rust::ManuallyDrop<::rust::String> return$;
  ::rust::repr::PtrLen error$ =
      mylib$cxxbridge1$process_widget(w, &return$.value);
  if (error$.ptr) {
    throw ::rust::impl<::rust::Error>::error(error$);
  }
  return ::std::move(return$.value);
}
```

The shim converts the Rust panic representation (`PtrLen` error slot) into a C++ `throw`. The `noexcept` on the inner `extern "C"` call — the actual Rust ABI boundary — tells the C++ compiler that no C++ exceptions propagate outward from Rust into the shim; the shim then re-throws using `rust::Error`. This layering is what makes the "zero-overhead, safe" claim precise: the shim is a single function call plus a conditional branch, inlinable by the C++ compiler when link-time optimization is active.

Both sides are type-checked against the shared declaration.

### Cross-Language Ownership Types

| cxx Type | Rust Equivalent | C++ Equivalent | Ownership |
|---|---|---|---|
| `UniquePtr<T>` | `Box<T>` (approximately) | `std::unique_ptr<T>` | C++ owns, Rust holds |
| `SharedPtr<T>` | `Arc<T>` (approximately) | `std::shared_ptr<T>` | shared ref-count |
| `CxxString` | `String` (opaque) | `std::string` | C++ allocated |
| `rust::Box<T>` | `Box<T>` | `rust::Box<T>` | Rust owns, C++ holds |
| `rust::Vec<T>` | `Vec<T>` | `rust::Vec<T>` | Rust allocated |
| `rust::Str` | `&str` | `rust::Str` | Rust borrow |
| `rust::Slice<T>` | `&[T]` | `rust::Slice<T>` | Rust borrow |

### Exception Handling

C++ exceptions thrown in an `extern "C++"` function are caught by the cxx shim at the bridge and converted to a Rust `Result::Err`. Rust panics in `extern "Rust"` functions are caught by the cxx shim and re-thrown as C++ exceptions with a `rust::Error` type. This makes the bridge safe by default: neither side can have its stack corrupted by the other's error model.

The `cxx-async` extension bridges `std::future<T>` with Rust's `Future` trait, enabling async code on both sides of the bridge to await each other.

### Build System Integration

cxx integrates with both `cargo` and `cmake`. In a Cargo build:

```toml
# Cargo.toml
[dependencies]
cxx = "1.0"

[build-dependencies]
cxx-build = "1.0"
```

```rust
// build.rs
fn main() {
    cxx_build::bridge("src/main.rs")
        .file("src/widget.cc")
        .std("c++17")
        .compile("mylib-cxx");

    println!("cargo:rerun-if-changed=src/main.rs");
    println!("cargo:rerun-if-changed=src/widget.cc");
    println!("cargo:rerun-if-changed=include/mylib/widget.h");
}
```

`cxx-build` invokes `cc` (the Cargo C/C++ compilation crate) to compile the generated C++ shim alongside the handwritten C++ source, then links everything into the Rust binary. The generated `.cpp` shim is placed in `OUT_DIR` and is never edited by hand.

In a CMake project that embeds Rust (using `corrosion`):

```cmake
find_package(Corrosion REQUIRED)
corrosion_import_crate(MANIFEST_PATH Cargo.toml)

# Link the cxx shim against the C++ target
target_link_libraries(my_cpp_target PRIVATE mylib-cxx)
```

---

## Swift/C++ Interoperability

### Direct C++ Import (Swift 5.9+)

Swift 5.9 introduced the Swift/C++ Interoperability workgroup's direct import model: Swift can import a Clang module map and use C++ types directly, without a bridging header, for a growing subset of C++ types.

```swift
// Package.swift
.target(
    name: "MySwiftTarget",
    interoperabilityMode: .cxx
)
```

```swift
import MyCxxLib  // Imports the Clang module map

let w = Widget()  // C++ value type, Swift stack-allocated
w.render()        // Calling C++ member function directly
```

C++ value types (those with copy and move constructors) map to Swift value types. Reference types (heap-allocated C++ objects) require annotation:

```cpp
// In the C++ header, annotate for Swift
class Widget SWIFT_SHARED_REFERENCE(retain_widget, release_widget) {
    // ...
};
```

### Exporting Swift to C and C++

The Swift compiler generates a `${TargetName}-Swift.h` umbrella header for any Swift declarations marked `@objc` or `@_cdecl`. This header can be included by C, Objective-C, and C++ callers:

```swift
// Swift
@_cdecl("swift_compute")
public func compute(x: Double, y: Double) -> Double {
    return x * x + y * y
}

@objc public class SwiftService: NSObject {
    @objc public func process(value: Int32) -> String { ... }
}
```

The generated header exposes `swift_compute` as a plain C function and `SwiftService` as an Objective-C class declaration.

---

## Python FFI

### ctypes

Python's `ctypes` module provides direct C library access without a compilation step:

```python
import ctypes
import ctypes.util

# Load a shared library
lib = ctypes.CDLL("./libfoo.so")

# Declare argument and return types
lib.foo_process.restype = ctypes.c_int
lib.foo_process.argtypes = [ctypes.c_void_p, ctypes.c_size_t]

# Define a C-compatible struct
class Point(ctypes.Structure):
    _fields_ = [("x", ctypes.c_double), ("y", ctypes.c_double)]

lib.point_distance.restype = ctypes.c_double
lib.point_distance.argtypes = [
    ctypes.POINTER(Point), ctypes.POINTER(Point)
]

a = Point(1.0, 2.0)
b = Point(4.0, 6.0)
dist = lib.point_distance(ctypes.byref(a), ctypes.byref(b))
```

`ctypes` performs no type checking at the C level; incorrect `argtypes` silently produce wrong results or crashes.

### cffi

`cffi` provides two modes. API mode (preferred) compiles a small C wrapper that the Python runtime links at import time, providing full type safety:

```python
from cffi import FFI
ffi = FFI()

ffi.cdef("""
    typedef struct { double x; double y; } Point;
    double point_distance(const Point *a, const Point *b);
""")

# API mode: compiles a C extension module
lib = ffi.dlopen("./libfoo.so")  # ABI mode
# or: ffi.set_source + ffi.compile() for API mode

a = ffi.new("Point *", {"x": 1.0, "y": 2.0})
b = ffi.new("Point *", {"x": 4.0, "y": 6.0})
dist = lib.point_distance(a, b)
```

### pybind11 and nanobind

For C++ → Python bindings with RAII semantics, `pybind11` and its faster successor `nanobind` generate CPython extension modules:

```cpp
// nanobind example
#include <nanobind/nanobind.h>
namespace nb = nanobind;

NB_MODULE(mylib, m) {
    m.def("process", [](double x, double y) {
        return x * x + y * y;
    }, nb::arg("x"), nb::arg("y"));
}
```

`nanobind` reduces binary size by 4× and import time by 2× compared to `pybind11` by eliminating runtime type registration overhead. Both manage the GIL explicitly: `nb::gil_scoped_release` releases the GIL for C++ work, `nb::gil_scoped_acquire` re-acquires it before touching Python objects.

The **Python stable ABI** (`Py_LIMITED_API=0x03030000`) allows building a single wheel that runs on Python 3.3+. The wheel tag `cp3-abi3-linux_x86_64` signals stable-ABI compliance.

---

## JNI — Java Native Interface

### Structure

JNI is the C-level interface between the JVM and native code. Every JNI function receives a `JNIEnv*` pointer (a vtable of JNI functions) and operates through it:

```c
#include <jni.h>

JNIEXPORT jint JNICALL
Java_com_example_MyClass_computeNative(
    JNIEnv *env,
    jobject this_obj,
    jlong x,
    jlong y)
{
    // Access Java objects through env
    jclass cls = (*env)->GetObjectClass(env, this_obj);
    jfieldID fid = (*env)->GetFieldID(env, cls, "state", "I");
    jint state = (*env)->GetIntField(env, this_obj, fid);

    return (jint)(x + y + state);
}
```

The function name encodes the fully qualified class name using underscores: `Java_com_example_MyClass_computeNative`. `JNIEXPORT` and `JNICALL` expand to the appropriate visibility and calling convention annotations on each platform.

### Object Lifecycle

Java objects passed to JNI are *local references* valid only for the duration of the native call. Storing them beyond the call requires promoting to a *global reference*:

```c
// Pin a Java object against GC; must be explicitly freed
jobject global = (*env)->NewGlobalRef(env, local_obj);
// ... store global in a C data structure ...
(*env)->DeleteGlobalRef(env, global);  // when done
```

Failure to delete global references causes memory leaks that the JVM GC cannot collect.

JNI crossing overhead is approximately 100 ns per call on a modern JVM — the JIT cannot inline across the JNI boundary. For high-frequency native calls, batch operations or the `Critical` JNI variant (`GetPrimitiveArrayCritical`) should be used to amortize the crossing cost. The `Critical` variant pins the Java array in place (suspending GC for its duration) and returns a direct C pointer to the underlying memory, eliminating the copy overhead for array operations.

### Android NDK and ABI Splits

Android applications package JNI native libraries in architecture-specific `.so` files inside the APK: `lib/armeabi-v7a/`, `lib/arm64-v8a/`, `lib/x86_64/`. Each must be compiled for the target ABI using the Android NDK's standalone Clang toolchain:

```bash
# Build a JNI library for arm64-v8a
export ANDROID_NDK=$HOME/Android/Sdk/ndk/26.1.10909125
$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android24-clang \
    -fPIC -shared -o lib/arm64-v8a/libnative.so \
    native.c -ljnigraphics
```

Rust targets the same architectures through `cargo`'s cross-compilation support:

```bash
cargo build --target aarch64-linux-android --release \
    --config "target.aarch64-linux-android.linker='aarch64-linux-android24-clang'"
```

The resulting `.so` is placed in the APK's `lib/arm64-v8a/` directory and loaded at runtime via `System.loadLibrary("native")`.

### Alternatives: Kotlin/Native cinterop and GraalVM

**Kotlin/Native cinterop** generates Kotlin bindings from C headers, providing a type-safe Kotlin API without JNI:

```bash
cinterop -def libfoo.def -o libfoo.klib
```

The `.def` file specifies headers, compiler flags, and linker flags; `cinterop` generates Kotlin stubs using the same header-parsing approach as `bindgen`.

**GraalVM Native Image** ahead-of-time compiles JVM bytecode to a native binary, eliminating the JVM startup overhead and enabling `@CEntryPoint` functions that are callable from C without JNI overhead:

```java
@CEntryPoint(name = "java_compute")
public static long compute(IsolateThread thread, long x, long y) {
    return x + y;
}
```

---

## UniFFI — Mozilla's Multi-Language FFI Generator

### Overview

UniFFI ([mozilla.github.io/uniffi-rs](https://mozilla.github.io/uniffi-rs/)) generates bindings for Python, Kotlin, Swift, and Ruby from a single Rust implementation. The Rust library exposes its API through one of two interface definition mechanisms and UniFFI generates the glue code for each target language, including:
- The native language wrapper (Python class, Kotlin class, Swift struct, etc.)
- A C-ABI intermediate layer that the wrapper calls
- Scaffolding Rust code that the intermediate layer calls into

### Interface Definition Modes

**UDL mode** uses an IDL-like `.udl` file:

```
// math.udl
namespace math {
    double add(double a, double b);
    [Throws=MathError]
    double divide(double a, double b);
};

dictionary Point {
    double x;
    double y;
};

[Error]
enum MathError {
    "DivisionByZero",
    "Overflow",
};

interface Calculator {
    constructor();
    double add(double a, double b);
    [Throws=MathError]
    double divide(double a, double b);
};
```

**Proc-macro mode** (preferred for new code) annotates Rust source directly:

```rust
#[derive(uniffi::Record)]
pub struct Point {
    pub x: f64,
    pub y: f64,
}

#[derive(uniffi::Error)]
pub enum MathError {
    DivisionByZero,
    Overflow,
}

#[uniffi::export]
pub fn add(a: f64, b: f64) -> f64 {
    a + b
}

#[uniffi::export]
pub fn divide(a: f64, b: f64) -> Result<f64, MathError> {
    if b == 0.0 {
        Err(MathError::DivisionByZero)
    } else {
        Ok(a / b)
    }
}

uniffi::include_scaffolding!("math");
```

### Binding Generation

```bash
# Generate Swift bindings from a compiled dynamic library
uniffi-bindgen generate \
    --library target/release/libmath.dylib \
    --language swift \
    --out-dir swift/

# Generate Kotlin bindings
uniffi-bindgen generate \
    --library target/release/libmath.so \
    --language kotlin \
    --out-dir android/

# Generate Python bindings
uniffi-bindgen generate \
    --library target/release/libmath.so \
    --language python \
    --out-dir python/
```

The generated Swift code calls the C-ABI scaffolding layer; the generated Kotlin code uses JNA (Java Native Access, a JNI alternative) to call the same scaffolding.

### What the Generated Code Looks Like

Understanding what UniFFI actually generates demystifies the abstraction. For the Python target, `uniffi-bindgen` produces a pure-Python module that calls the C scaffolding through `ctypes`:

```python
# Excerpt of generated math.py (abbreviated for clarity)
from ctypes import cdll, c_double, c_int32, byref, Structure
import ctypes, sys

_lib = cdll.LoadLibrary("libmath.so")

# Scaffolding functions declared in Rust by uniffi::include_scaffolding!
_lib.uniffi_math_fn_func_add.restype = c_double
_lib.uniffi_math_fn_func_add.argtypes = [c_double, c_double]

_lib.uniffi_math_fn_func_divide.restype = c_double
_lib.uniffi_math_fn_func_divide.argtypes = [
    c_double, c_double,
    ctypes.POINTER(RustCallStatus),  # out-param for error
]

class RustCallStatus(Structure):
    _fields_ = [("code", c_int32), ("error_buf", RustBuffer)]

def add(a: float, b: float) -> float:
    return _lib.uniffi_math_fn_func_add(a, b)

def divide(a: float, b: float) -> float:
    _status = RustCallStatus()
    _retval = _lib.uniffi_math_fn_func_divide(a, b, byref(_status))
    _check_call_status(_status)  # raises MathError on error
    return _retval
```

The Rust scaffolding layer (generated by `uniffi::include_scaffolding!`) exposes `uniffi_math_fn_func_add` as a `#[no_mangle] pub extern "C"` function. The C-ABI layer is the only crossing point; the Python wrapper above it and the Rust implementation below it are both idiomatic in their own languages.

### Production Use in Firefox

UniFFI is the standard FFI mechanism for Firefox components written in Rust:
- **WebRender**: GPU rendering engine, generating Swift and Kotlin bindings for iOS and Android
- **Nimbus SDK**: feature flag system, generating bindings for all four languages
- **Application Services**: sync engine for bookmarks, logins, and history
- **geckodriver**: WebDriver implementation for browser automation testing

The UniFFI scaffolding layer handles error propagation automatically: Rust `Result::Err` values are serialized into a C-compatible error structure (`RustCallStatus`) that the generated wrappers in each target language deserialize into native exceptions or error types.

### Versioning and ABI Stability

UniFFI generates a checksum for each function signature (derived from function name, argument types, and return type) and embeds it in both the scaffolding and the generated bindings. When the Rust library is loaded, the generated wrapper verifies that its checksum matches the scaffolding's checksum. This catches version skew — a common failure mode when a mobile app ships an old binding against a new library version — at load time rather than at crash time.

```
// Generated Kotlin — checksum verification at init
checkContractApiVersion(lib)  // verifies API version matches
checkContractAbiVersion(lib)  // verifies ABI version matches
uniffiCheckApiChecksums(lib)   // verifies per-function checksums
```

The proc-macro mode derives checksums from the Rust type system, so changing a function signature automatically invalidates the checksum, preventing silent misuse of old bindings against new libraries.

---

## WebAssembly Component Model and WIT

### The Component Model

The [W3C WebAssembly Component Model](https://component-model.bytecodealliance.org) is a language-agnostic component standard that goes beyond the Wasm core binary format: components declare their interface using WIT (WebAssembly Interface Types), can be composed with other components, and expose a common ABI that runtimes implement.

### WIT Interface Definition Language

```wit
package component:math@0.1.0;

interface calculator {
    /// Add two floating-point numbers
    add: func(a: f64, b: f64) -> f64;

    /// Compute square root, returning an error for negative input
    sqrt: func(x: f64) -> result<f64, string>;
}

world math-world {
    export calculator;
}
```

WIT types map to Wasm component types: `func`, `record`, `variant`, `enum`, `option`, `result`, `list`, `tuple`. The component model's canonical ABI defines the exact memory layout and lifting/lowering semantics for each type.

### Rust Components with cargo-component

```bash
cargo install cargo-component
cargo component new --lib math-component
```

```rust
// src/lib.rs — generated by cargo-component
use bindings::exports::component::math::calculator::Guest;

struct Component;

impl Guest for Component {
    fn add(a: f64, b: f64) -> f64 {
        a + b
    }

    fn sqrt(x: f64) -> Result<f64, String> {
        if x < 0.0 {
            Err(format!("{x} is negative"))
        } else {
            Ok(x.sqrt())
        }
    }
}

bindings::export!(Component with_types_in bindings);
```

### Hosting with wasmtime

```rust
use wasmtime::component::{bindgen, Component, Linker};
use wasmtime::{Engine, Store};

bindgen!({
    world: "math-world",
    path: "wit/math.wit",
});

fn main() -> anyhow::Result<()> {
    let engine = Engine::default();
    let component = Component::from_file(&engine, "math.wasm")?;
    let linker = Linker::new(&engine);
    let mut store = Store::new(&engine, ());

    let (bindings, _) = MathWorld::instantiate(&mut store, &component, &linker)?;
    let result = bindings.component_math_calculator().call_add(&mut store, 1.0, 2.0)?;
    println!("1 + 2 = {result}");
    Ok(())
}
```

### wasm-bindgen for the Web

For browser targets, `wasm-bindgen` provides the Rust/JavaScript bridge:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}
```

`wasm-bindgen` generates JavaScript glue that handles UTF-8 string encoding/decoding, memory management, and type mapping between JS and Wasm. Strings cross the boundary by writing UTF-8 bytes into the Wasm linear memory and passing a pointer/length pair; the JavaScript side reads them back via `TextDecoder`. This copy is unavoidable given the Wasm memory model — there is no shared-memory string type between the JS engine heap and Wasm linear memory.

For types that cross frequently, `wasm-bindgen` supports the `js_sys` crate (bindings to JavaScript built-ins like `Array`, `Map`, `Promise`) and the `web-sys` crate (bindings to Web APIs), both auto-generated from WebIDL definitions using the same principle as `bindgen` for C headers.

---

## LLVM IR as a Potential Interchange Layer

C's dominance as the FFI lingua franca is a matter of ABI stability, not representational power. LLVM IR is strictly more expressive than C: it captures calling conventions, type layouts, attribute sets, and optimization metadata in a single serializable format. Since every LLVM-targeting language lowers to the same IR, a plausible thesis is that LLVM IR itself should serve as the interop layer — let each language compile to bitcode, and compose at the IR level rather than at the C ABI level. This section examines what happened when that thesis was tested in practice.

### The Universal-IR Thesis

The appeal is concrete. A Rust crate and a C++ translation unit compiled to LLVM IR share the same SSA value model, the same `datalayout` string, and the same `DataLayout`-derived struct sizes. In principle, a caller in one language can directly reference a callee in the other without any ABI negotiation — the calling convention is already encoded in the IR. Cross-language inlining, devirtualization, and dead-code elimination become possible without any special tooling. No header files, no marshalling, no binding generators.

The thesis is partially realized in one concrete context: link-time optimization. When Rust and C/C++ are compiled with `-flto`, both emit LLVM bitcode into their object files (in the `.llvm.lto` ELF section). LLD reads all bitcode modules, merges them, and runs the standard optimization pipeline over the combined IR — inlining Rust functions into C callers, propagating constants across the boundary, and eliminating dead code that only becomes visible at whole-program scope. This is genuine cross-language optimization at the IR level, and it works. But it is an optimization artifact, not an interop contract: the languages still agree on a C calling convention at their boundary. The IR sharing is invisible to the programmer.

### Apple Bitcode — The Only Large-Scale Deployment

The closest any production system came to using LLVM IR as a distribution and interop layer was Apple's App Store bitcode requirement, active from Xcode 7 (2015) to Xcode 14 (2022).

**The mechanism.** When building for iOS, watchOS, or tvOS with bitcode enabled, Xcode instructed Clang and swiftc to embed a copy of the compiled LLVM IR bitcode alongside the native machine code in the binary. The ELF-equivalent Apple Mach-O section is `__LLVM,__bitcode` in the `__TEXT` segment. The App Store accepted this binary, stripped the native code, and retained the bitcode. When Apple released a new device (e.g., iPhone 6 with a new ARM microarchitecture variant) or wanted to apply a new optimization pass retroactively, it could recompile all submitted apps from their stored bitcode without requiring developers to resubmit.

```
# What Xcode passed to clang:
clang -fembed-bitcode -arch arm64 -o MyApp.o MyApp.c

# The resulting object contained both:
#   __TEXT,__text       — native arm64 machine code
#   __LLVM,__bitcode    — LLVM IR bitcode for recompilation
```

**Why Apple deprecated it.** Bitcode-based recompilation turned out to be fragile in practice. Compiler intrinsics, target-specific attributes, and optimization-affecting metadata all changed between LLVM releases. Bitcode produced by Xcode 7's LLVM 3.7 was incompatible with the optimizer in Xcode 12's LLVM 11. Apple had to maintain multiple LLVM versions to recompile old bitcode, and the bitcode files were large (often 2–5× the native code size), adding significant App Store upload and storage cost. Apple announced the deprecation of bitcode in Xcode 14 (2022), citing these costs and the reduced value now that Apple Silicon Macs and the M-series iPhone SoCs had stabilized the microarchitecture landscape. The [App Store bitcode deprecation release notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-14-release-notes) note that Xcode 14 no longer submits bitcode to the App Store by default.

### LTO Bitcode as Cross-Language Optimization Substrate

The most durable use of LLVM IR as a cross-language medium is LTO, described in detail in [Chapter 77 — LTO and ThinLTO](../part-13-lto-whole-program/ch77-lto-and-thinlto.md). Its cross-language dimension is worth examining here.

When a project links Rust and C together with ThinLTO, the following happens at the IR level:

```bash
# Rust crate compiled to bitcode
rustc --edition 2024 -C lto=thin -C opt-level=2 --emit=llvm-bc lib.rs -o lib.bc

# C translation unit compiled to bitcode
clang -O2 -flto=thin -c util.c -o util.o   # util.o embeds bitcode in .llvm.lto section

# LLD links both, sees bitcode from both languages, merges and optimizes:
clang -O2 -flto=thin -o app app.o util.o liblib.rlib
```

LLD's ThinLTO pass sees one unified IR module containing functions from both languages. A Rust `#[inline]` function called from C can be inlined across the language boundary; a C function with `__attribute__((pure))` can have its side-effect-free attribute propagated into Rust call sites. The optimization is genuine and measurable — projects like the Firefox JavaScript engine use LTO across their Rust and C++ components specifically to recover this cross-language inlining.

The boundary condition still applies: the Rust function must be declared `extern "C"` (or `extern "C-unwind"`) and the C function must use the C calling convention. The IR sharing gives the optimizer freedom within those conventions, but it does not replace them.

### Why LLVM IR Cannot Be a Stable FFI Contract

Four structural reasons prevent LLVM IR from replacing C as the FFI lingua franca:

**1. No version stability.** LLVM IR is explicitly versioned and makes no stability promises across major releases. The `ModuleID`, `source_filename`, and `datalayout` all embed version-sensitive information. A bitcode file produced by LLVM 17 may not parse correctly in LLVM 22 if any instruction or type encoding changed. The C ABI, by contrast, is frozen per platform: SysV x86-64 ABI has been stable since 2001.

**2. No standardized type system for interop.** LLVM IR has a type system (`i8`, `i32`, `ptr`, `{i64, i8*}`, etc.) but no notion of ownership, variance, or safety invariants. Two languages can agree on `{i64, ptr}` as the representation of a slice but disagree on whether the pointer is nullable, whether the slice is mutable, or who frees it. The C ABI has the same problem, but decades of tooling (header files, binding generators, API conventions) have grown around it. LLVM IR has no equivalent ecosystem.

**3. Language-specific IR features are not shared.** Rust's IR uses `noundef`, `nonnull`, `noalias`, and `align` attributes extensively, generating IR that is UB if those attributes are violated. Swift's IR uses `swiftself` and `swifterror` calling convention extensions. Exception-handling IR (`landingpad`, `cleanupret`, `catchswitch`) differs between Itanium EH (used by GCC-compatible C++) and SEH (used by MSVC). A C caller receiving a Rust function's IR contract must honor all these attributes or invoke UB — and there is no tool to check this.

**4. The optimization contract is one-way.** LTO cross-language inlining works because both sides compile together under one optimizer's authority. A distributed FFI — where one language publishes a compiled artifact and another language consumes it — cannot use LTO bitcode without coupling the consumer to the producer's LLVM version, optimization level, and IR assumptions. This is precisely the coupling that ABI stability is designed to avoid.

```llvm
; Rust-emitted function: the noalias, noundef, nonnull attributes are
; part of the IR contract that C callers are not expected to honor.
define void @process_slice(ptr noalias noundef nonnull %ptr, i64 %len) {
  ; A C caller passing a null pointer would invoke UB even though
  ; the C-level signature is   void process_slice(void*, size_t)
  ...
}
```

### WebAssembly as the Realized Universal IR

The universal-IR thesis was not abandoned — it was redirected. WebAssembly (Wasm) is the realization of what LLVM IR could not be: a stable, portable, language-agnostic binary format that multiple languages compile to and that runtimes across platforms execute.

Wasm succeeds where LLVM IR fails for three reasons:

- **Stable binary format.** The [WebAssembly Core Specification](https://webassembly.github.io/spec/core/) is a W3C standard. A `.wasm` binary produced today will be valid in any conforming runtime indefinitely. LLVM IR makes no such promise.
- **No host ABI dependency.** Wasm's linear memory model and value types (`i32`, `i64`, `f32`, `f64`, `v128`) are platform-independent. There is no `datalayout` string; struct layout is determined by the Wasm module itself, not by a host platform ABI.
- **The Component Model as typed FFI.** The WebAssembly Component Model (§ "WebAssembly Component Model and WIT" above) adds exactly what raw Wasm lacks: a type system for interop, with WIT as the IDL. Two components compiled from different source languages compose through WIT interfaces without any knowledge of each other's internal representation.

LLVM itself targets WebAssembly ([Chapter 106 — WebAssembly and BPF](../part-15-targets/ch106-webassembly-and-bpf.md)), closing the loop: languages use LLVM as their compiler backend to emit Wasm, and Wasm is the stable interchange format.

### MLIR as the Structured-IR Alternative

Within the ML/HPC ecosystem, MLIR ([Chapter 129 — MLIR Philosophy](../part-19-mlir-foundations/ch129-mlir-philosophy.md)) pursues a different answer to the same question: a multi-level IR with stable, versioned dialects that multiple domain-specific languages and frameworks can target.

StableHLO ([Chapter 154 — HLO and StableHLO](../part-22-xla-openxla/ch154-hlo-and-stablehlo.md)) is the concrete realization: a versioned MLIR dialect with a compatibility guarantee, used as the interchange format between ML frameworks (JAX, PyTorch, TensorFlow) and hardware compilers (XLA, IREE). A StableHLO program produced by one framework version is guaranteed to be accepted by any compiler that claims StableHLO compatibility, regardless of the internal MLIR version either side uses.

This is precisely the LLVM-IR-as-universal-IR thesis applied with the lessons learned: stability requires explicit versioning and a compatibility guarantee, not just a shared representation.

| Format | Stability | Type system for interop | Cross-language | Used for |
|---|---|---|---|---|
| C ABI | Frozen per platform | No (header files are the contract) | Yes (universally) | System FFI |
| LLVM IR bitcode | Version-coupled | No | Yes (within one build) | LTO optimization |
| Apple Bitcode | Deprecated 2022 | No | No | App re-optimization |
| WebAssembly | W3C standard | Via Component Model / WIT | Yes | Portable binary distribution |
| StableHLO | Versioned dialect | Yes (ML tensor ops) | Framework-to-compiler | ML compilation stack |

---

## Exception and Unwinding Interoperability

### Four Incompatible Error Models

| Language | Error Model | Unwinding Mechanism |
|---|---|---|
| C | Return codes, `errno` | None |
| C++ | Exceptions (`throw`/`catch`) | Itanium LSDA / MSVC SEH |
| Rust | `panic!` + `Result<T,E>` | Itanium LSDA (same as C++) |
| Swift | Typed errors, `throws` | Objective-C exceptions (ObjC interop) |
| ObjC | `NSException` | Objective-C runtime |

### `extern "C-unwind"` (Rust 1.71)

Before stabilization of `extern "C-unwind"`, allowing a C++ exception to unwind through a Rust `extern "C"` frame was undefined behavior — the Rust frame did not have an unwind table entry. `extern "C-unwind"` (stabilized in Rust 1.71) generates an unwind table entry for the Rust frame that correctly propagates foreign exceptions:

```rust
// Allows C++ exceptions (and Rust panics) to pass through this frame
extern "C-unwind" {
    fn cpp_function_that_may_throw(x: i32) -> i32;
}
```

When called with `extern "C-unwind"`, the Rust frame participates in the unwinding process: C++ exceptions pass through without termination, and Rust panics propagate outward as well.

### `catch_unwind` at FFI Boundaries

The canonical pattern for a Rust library called from C is to wrap every exported function body in `std::panic::catch_unwind`:

```rust
use std::panic;

#[no_mangle]
pub extern "C" fn safe_process(x: i64) -> i32 {
    match panic::catch_unwind(|| {
        // Rust code that might panic
        do_work(x)
    }) {
        Ok(result) => result,
        Err(_) => {
            // Log the panic, return an error sentinel
            eprintln!("Rust panic in safe_process");
            -1
        }
    }
}
```

Without `catch_unwind`, a panic that reaches an `extern "C"` boundary invokes undefined behavior (in Rust editions before 2024) or aborts the process (in Rust 2024 edition with `-C panic=abort`). The `catch_unwind` pattern is non-negotiable for Rust code embedded in a larger system that does not want process termination on Rust panics.

The libunwind library — discussed in [Chapter 121 — libc++](../part-17-runtime-libs/ch121-libcxx.md) — is the common unwinding substrate on Linux and macOS. Both Rust and C++ compile to the Itanium LSDA format when targeting these platforms, which is why `extern "C-unwind"` can work: the unwinding tables speak a common language.

### Choosing Between `panic=unwind` and `panic=abort`

Rust supports two panic strategies, selected via `-C panic=`:

- **`panic=unwind`** (default): panics unwind the Rust stack, running destructors, before reaching the nearest `catch_unwind`. This is compatible with C++ exception handling but requires that every `extern "C"` boundary be guarded with `catch_unwind`.
- **`panic=abort`**: any panic immediately terminates the process via `abort(3)`. No unwinding occurs. This is safe across `extern "C"` boundaries because panics never propagate — the process ends instead. `panic=abort` produces smaller binaries (no unwind tables, no landing pads) and is the preferred mode for embedded systems and WebAssembly targets.

For a Rust library embedded in a larger native application where crashing the host process is unacceptable, `panic=unwind` with `catch_unwind` at every exported boundary is the correct choice. For standalone Rust programs or Wasm components, `panic=abort` is typically better: it removes the unwinding machinery entirely, the optimizer can eliminate more code, and the binary is measurably smaller.

The Cargo profile setting:

```toml
[profile.release]
panic = "abort"  # No unwind tables; panics abort the process
```

cannot be mixed with C++ exception handling in the same binary if C++ code relies on exception-based cleanup — if a C++ exception propagates into a Rust frame compiled with `panic=abort`, the runtime will terminate rather than propagate the exception.

---

## Fuzzing FFI Boundaries

### cargo-fuzz at the Rust Side

Any Rust function that receives data from C is a natural fuzzing target. `cargo-fuzz` runs `libFuzzer` against a Rust harness:

```bash
cargo fuzz init
cargo fuzz add ffi_roundtrip
```

```rust
// fuzz/fuzz_targets/ffi_roundtrip.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if data.len() < 8 { return; }
    unsafe {
        // Call C function with fuzz-generated data
        let result = my_c_lib::process_bytes(
            data.as_ptr() as *const libc::c_void,
            data.len() as libc::size_t,
        );
        // Assert postconditions on result
        assert!(result >= 0 || result == -1);
    }
});
```

### Sanitizers Across the FFI Boundary

AddressSanitizer and UBSan work across FFI boundaries when both the C/C++ library and the Rust binary are compiled with `-fsanitize=address,undefined`. The ASAN runtime is a single shared library that intercepts allocator calls from both languages. A buffer overread in C code called from Rust is detected and reported with a full cross-language stack trace.

```bash
# Compile C library with ASAN
clang -fsanitize=address,undefined -fPIC -shared -o libfoo.so foo.c

# Compile and run Rust binary with ASAN
RUSTFLAGS="-Zsanitizer=address" \
ASAN_OPTIONS="detect_leaks=1" \
cargo +nightly test --target x86_64-unknown-linux-gnu
```

See [Chapter 110 — User-Space Sanitizers](../part-16-jit-sanitizers/ch110-user-space-sanitizers.md) and [Chapter 111 — HWASan and MTE](../part-16-jit-sanitizers/ch111-hwasan-mte.md) for the sanitizer runtime architecture.

### bindgen/cbindgen Round-Trip Testing

A powerful correctness technique: generate a C header from a Rust struct with cbindgen, then run bindgen over the generated header, and verify that the resulting Rust struct has identical field types and offsets. Discrepancies indicate either a cbindgen generation bug or a layout assumption mismatch.

```bash
# Generate C header from Rust
cbindgen --lang c -o /tmp/mylib.h

# Generate Rust bindings from the C header
bindgen /tmp/mylib.h -o /tmp/roundtrip.rs \
    --with-derive-debug \
    --with-derive-partialeq

# Diff the original Rust struct layout against the roundtrip
diff <(grep -A20 "struct MyStruct" src/lib.rs) \
     <(grep -A20 "struct MyStruct" /tmp/roundtrip.rs)
```

---

## FFI Approach Comparison

| Tool | Direction | Language Support | Overhead | Primary Use Case |
|---|---|---|---|---|
| `bindgen` | C/C++ → Rust | Rust | Zero (raw FFI) | Wrapping C system libraries |
| `cbindgen` | Rust → C/C++ | C, C++ | Zero (raw FFI) | Exposing Rust libraries to C callers |
| `cxx` | Rust ↔ C++ | Rust, C++ | ~1 ns per call (shim) | Safe bidirectional Rust/C++ bridge |
| `UniFFI` | Rust → multiple | Python, Kotlin, Swift, Ruby | ~50–200 ns (scaffolding) | Mobile/desktop SDK distribution |
| `ctypes` | C → Python | Python | ~200 ns (Python overhead) | Quick C library integration |
| `cffi` API mode | C → Python | Python | ~100 ns | Type-safe Python/C integration |
| `pybind11`/`nanobind` | C++ → Python | Python | ~50 ns | C++ algorithm exposure to Python |
| JNI | C → Java/Kotlin | Java, Kotlin (JVM) | ~100 ns | Android NDK, embedded JVM |
| `cinterop` | C → Kotlin/Native | Kotlin/Native | ~5 ns | Native Kotlin targets |
| `wit-bindgen` | WIT → any | Rust, C, and others | Wasm component ABI | Wasm component composition |
| `wasm-bindgen` | Rust ↔ JS | JavaScript/TypeScript | Wasm/JS bridge overhead | Browser Wasm integration |
| Swift C++ interop | C++ ↔ Swift | Swift, C++ | Zero (direct import) | Apple platform mixed-language code |

---

## Chapter Summary

- **C is the universal FFI substrate**: every language achieves cross-language calls by mapping to C calling conventions (`cc 0` in LLVM IR), C type layouts, and C-compatible symbol names
- **Three invariants**: calling convention match, type layout match, and exception/panic isolation are the necessary and sufficient conditions for correct FFI
- **`#[repr(C)]` in Rust** forces C-compatible struct layout; without it, Rust may reorder fields or apply niche optimizations that break C interop
- **`bindgen`** generates Rust FFI bindings from C/C++ headers via `libclang`, handling anonymous structs, opaque types, variadic functions, and flexible array members; integrates with `build.rs`
- **`cbindgen`** generates C/C++ headers from Rust `pub` APIs marked with `#[repr(C)]` and `#[no_mangle]`; used by Firefox WebRender and Mozilla NSS
- **`cxx`** provides a zero-overhead, safe Rust/C++ bridge with ownership-typed cross-language types (`UniquePtr<T>`, `rust::Box<T>`, `rust::Vec<T>`) and automatic exception-to-panic conversion at the bridge
- **`UniFFI`** generates bindings for Python, Kotlin, Swift, and Ruby from a single Rust implementation using either a UDL file or proc-macro annotations; the standard mechanism for Firefox component distribution
- **The WebAssembly Component Model** and WIT define a language-agnostic component interface standard; `cargo-component`, `wit-bindgen`, and `wasmtime::component::bindgen!` implement it in Rust
- **LLVM IR was attempted as a universal interchange layer**: Apple Bitcode (2015–2022) embedded IR in App Store submissions for re-optimization; LTO bitcode enables genuine cross-language inlining between Rust and C/C++ within a single build. Neither became a stable FFI contract because LLVM IR has no version stability guarantee, no agreed ownership semantics, and language-specific IR attributes that callers cannot safely ignore
- **WebAssembly is the realized universal IR**: a W3C-stable binary format with no host-ABI dependency; the Component Model adds typed interop via WIT, making it the practical successor to the LLVM-IR-as-universal-IR thesis. **StableHLO** applies the same lesson to the ML stack: a versioned MLIR dialect with an explicit compatibility guarantee used as the interchange format between ML frameworks and hardware compilers
- **`extern "C-unwind"`** (Rust 1.71) allows C++ exceptions to propagate through Rust frames safely; `catch_unwind` at every exported FFI boundary prevents Rust panics from crossing into C callers as undefined behavior
- **Sanitizers cross FFI boundaries**: ASAN and UBSan detect memory errors across C and Rust when both sides are compiled with sanitizer instrumentation

---

*@copyright jreuben11*
