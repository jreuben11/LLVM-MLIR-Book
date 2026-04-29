# Chapter 5 — LLVM as a Library

*Part I — Foundations*

The LLVM project's most consequential architectural decision is not SSA form, not the code generator's separation of concerns, and not even the IR design. It is the library-first philosophy: every component is a reusable library, every tool is a thin shell around those libraries, and downstream consumers are first-class citizens. A static analysis tool, a JIT engine, a language-server back-end, a sanitizer runtime, or a domain-specific compiler can all link against precisely the LLVM components they need — and only those components — without forking the project or invoking private APIs.

This chapter explains how to do that linking correctly. It covers two integration methods (`llvm-config` for quick builds, CMake's `find_package` for production projects), the component model that makes minimal linking possible, the C API that gives non-C++ language ecosystems a stable ABI target, and the mechanics of writing and loading an out-of-tree pass plugin. It closes with a guide to the out-of-tree backend skeleton and a troubleshooting reference for the linking errors that catch nearly every first-time integrator. All examples are verified against LLVM 22.

---

## 5.1 `llvm-config`: The Quick and Dirty Way

The `llvm-config` binary is the original integration mechanism and remains the fastest way to compile a standalone tool against LLVM. It ships with every LLVM installation, knows the exact compilation flags and library lists for the installed tree, and requires no CMake knowledge.

### What `llvm-config` provides

```bash
# Show the LLVM version installed
llvm-config --version
# 22.1.3

# C++ compiler flags (includes -std=c++17, -fno-exceptions if LLVM was built that way, etc.)
llvm-config --cxxflags

# Linker flags (-L path to LLVM lib directory)
llvm-config --ldflags

# Library list for the requested components
llvm-config --libs core support irreader

# System libraries that LLVM itself depends on (librt, libdl, libm, pthreads, etc.)
llvm-config --system-libs

# Installation directories
llvm-config --includedir
llvm-config --libdir
llvm-config --bindir

# Prefix of the install tree
llvm-config --prefix

# Which components exist in this installation
llvm-config --components
```

Each flag is independently queryable so you can compose them precisely. The `--cxxflags` output is especially important: LLVM enforces its own `-DNDEBUG`, `-fno-rtti`, `-fno-exceptions`, and `-std=c++17` flags, and mixing translation units compiled with and without these flags produces subtle, hard-to-diagnose undefined behavior. Always compile every `.cpp` that touches LLVM headers with the flags returned by `llvm-config --cxxflags`, not just the files that `#include` LLVM directly.

### A minimal command-line build

The canonical one-shot compilation pattern:

```bash
clang++ $(llvm-config --cxxflags) mytool.cpp \
        $(llvm-config --ldflags --libs core support irreader) \
        $(llvm-config --system-libs) \
        -o mytool
```

Breaking down the argument groups:

- `$(llvm-config --cxxflags)` — compilation flags: header search paths, standard version, RTTI/exception settings, optimization level, macro definitions like `_GNU_SOURCE`.
- `$(llvm-config --ldflags)` — linker flags: the `-L` path pointing at LLVM's lib directory.
- `$(llvm-config --libs core support irreader)` — the `-l` flags for the named components and all their transitive dependencies. `llvm-config` resolves the dependency graph internally, so you do not need to list every transitive dependency manually.
- `$(llvm-config --system-libs)` — platform system libraries: `-ldl`, `-lpthread`, `-lm`, and on some platforms `-lrt` or `-lcurses`. Omitting this causes undefined-symbol errors during linking for functions like `dlopen`, `pthread_create`, and `cos`.

### Using `llvm-config` in Makefiles

```makefile
LLVM_CONFIG ?= llvm-config

CXXFLAGS += $(shell $(LLVM_CONFIG) --cxxflags)
LDFLAGS  += $(shell $(LLVM_CONFIG) --ldflags)
LLVMLIBS := $(shell $(LLVM_CONFIG) --libs core support irreader transforms)
SYSLIBS  := $(shell $(LLVM_CONFIG) --system-libs)

mytool: mytool.o
	$(CXX) $(LDFLAGS) $< $(LLVMLIBS) $(SYSLIBS) -o $@

mytool.o: mytool.cpp
	$(CXX) $(CXXFLAGS) -c $< -o $@
```

The `LLVM_CONFIG ?= llvm-config` pattern allows the variable to be overridden from the command line (`make LLVM_CONFIG=/opt/llvm-22/bin/llvm-config`) when multiple LLVM versions are installed, which is the normal situation on developer machines and CI systems.

### Static versus shared: the `--link-static` and `--link-shared` flags

By default `llvm-config --libs` emits static library flags on platforms where LLVM was built statically, and shared library flags where it was not. You can override this:

```bash
# Force static linking (requires a static LLVM build)
llvm-config --link-static --libs core support

# Force shared linking against libLLVM.so
llvm-config --link-shared --libs core support
```

The static build produces large but fully self-contained binaries. A statically linked `opt` binary is approximately 80–120 MB for a full-target build. The shared-library build (`libLLVM-22.so` on Linux, `libLLVM.dylib` on macOS) shrinks each tool to a few hundred kilobytes but introduces an `RPATH` dependency that must be satisfied at runtime. The trade-off:

| Concern | Static | Shared |
|---------|--------|--------|
| Binary size | Large (each tool carries its own copy) | Small (all tools share one DSO) |
| Deployment complexity | Self-contained | Requires `libLLVM.so` in `LD_LIBRARY_PATH` or `RPATH` |
| LTO / PGO of the tool itself | Easy — all code in one unit | Hard — DSO boundary blocks inlining |
| Multiple LLVM versions on one machine | Simple — no DSO conflict | Requires care with RPATH/RUNPATH versioning |
| Build time | Longer (links all static archives) | Faster (only links stubs) |

For shipping a developer tool to end users, static linking is almost always the right answer. For a developer workstation where you rebuild the tool frequently, the faster link time of the shared build is worth the RPATH configuration.

### The `--components` flag and what each component contains

`llvm-config --components` lists all components available in the installed tree. The list varies with the build configuration (which targets were enabled, which experimental features were included), but a typical LLVM 22 install includes around 100 named components. Key groupings:

- **Core infrastructure:** `core`, `support`, `irreader`, `irprinter`, `bitreader`, `bitwriter`
- **Analysis:** `analysis`, `instcombine`, `scalaropts`, `ipo`, `transformutils`
- **Code generation common:** `codegen`, `asmprinter`, `asmparser`, `mc`, `mcdisassembler`
- **Target-specific (X86):** `x86`, `x86codegen`, `x86asmparser`, `x86disassembler`
- **JIT:** `orcjit`, `jitlink`, `executionengine`, `mcjit`
- **Linker:** `linker`, `lto`, `passes`
- **Debugging:** `debuginfodwarf`, `debuginfogsym`, `symbolize`

The `all` pseudo-component links everything. The `all-targets` pseudo-component links all enabled target backends. Both produce very large binaries and should be avoided in production tools.

---

## 5.2 CMake Integration with `find_package(LLVM)`

For any project that will be built more than once — which in practice means any project beyond a one-off script — the CMake `find_package(LLVM)` integration is the correct approach. It handles multi-configuration generators, cross-compilation, `ccache`/`sccache` integration, and dependency tracking in ways that shell-expansion of `llvm-config` flags cannot.

### The `LLVMConfig.cmake` file

When LLVM is built or installed, it writes a CMake package config file at:

```
<install-prefix>/lib/cmake/llvm/LLVMConfig.cmake
```

In a typical install this is:

```
/usr/lib/llvm-22/lib/cmake/llvm/LLVMConfig.cmake   # Debian/Ubuntu package
/opt/llvm-22/lib/cmake/llvm/LLVMConfig.cmake        # custom install
<build-dir>/lib/cmake/llvm/LLVMConfig.cmake         # in-build tree
```

`LLVMConfig.cmake` exports the `LLVM` CMake package with the following variables set after a successful `find_package`:

| Variable | Content |
|----------|---------|
| `LLVM_PACKAGE_VERSION` | The full version string, e.g., `22.1.3` |
| `LLVM_VERSION_MAJOR` | `22` |
| `LLVM_INCLUDE_DIRS` | Semicolon-separated list of include directories |
| `LLVM_LIBRARY_DIRS` | Semicolon-separated list of library directories |
| `LLVM_DEFINITIONS` | Preprocessor definitions required by LLVM headers |
| `LLVM_TOOLS_BINARY_DIR` | Path to LLVM's installed tools (`clang`, `opt`, `llc`, …) |
| `LLVM_ENABLE_ASSERTIONS` | `ON`/`OFF` reflecting the installed build |
| `LLVM_ENABLE_EH` | Whether LLVM was built with exceptions |
| `LLVM_ENABLE_RTTI` | Whether LLVM was built with RTTI |
| `LLVM_TARGETS_TO_BUILD` | Semicolon-separated list of enabled backends |
| `LLVM_CMAKE_DIR` | The directory containing LLVM CMake utilities |

It also `include()`s `AddLLVM.cmake` and `HandleLLVMOptions.cmake`, which define functions like `llvm_map_components_to_libnames()` and `add_llvm_executable()`.

### Pointing CMake at the right LLVM

CMake's package discovery searches `CMAKE_PREFIX_PATH` and well-known system paths. When multiple LLVM versions are installed, you must be explicit:

```bash
cmake -S . -B build \
  -DLLVM_DIR=/opt/llvm-22/lib/cmake/llvm
```

Equivalently, with `CMAKE_PREFIX_PATH`:

```bash
cmake -S . -B build \
  -DCMAKE_PREFIX_PATH=/opt/llvm-22
```

The `LLVM_DIR` variable takes precedence over all automatic search paths and is the reliable way to pin an exact installation.

### A complete, minimal `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.20)
project(mytool CXX)

# Locate LLVM 22 (or any 22.x release)
find_package(LLVM 22 REQUIRED CONFIG)

message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")
message(STATUS "  Include dirs: ${LLVM_INCLUDE_DIRS}")
message(STATUS "  Library dirs: ${LLVM_LIBRARY_DIRS}")

# Apply LLVM's required compile definitions and include paths
include_directories(${LLVM_INCLUDE_DIRS})
separate_arguments(LLVM_DEFINITIONS_LIST NATIVE_COMMAND ${LLVM_DEFINITIONS})
add_definitions(${LLVM_DEFINITIONS_LIST})

# Build the tool
add_executable(mytool mytool.cpp)

# Map component names to library targets
llvm_map_components_to_libnames(llvm_libs
    core
    support
    irreader
)

# Link against the resolved library set
target_link_libraries(mytool PRIVATE ${llvm_libs})
```

### Walking through each directive

**`find_package(LLVM 22 REQUIRED CONFIG)`** — The `REQUIRED` keyword causes CMake to fail immediately with a diagnostic if LLVM is not found; without it, the failure is silent and the missing-variable errors appear later in confusing places. The `CONFIG` keyword bypasses Find modules (which are often outdated) in favor of LLVM's own exported config file.

**`include_directories(${LLVM_INCLUDE_DIRS})`** — Adds LLVM's public headers to the include search path. In modern CMake, prefer `target_include_directories(mytool PRIVATE ${LLVM_INCLUDE_DIRS})` to avoid polluting the global include path; the `include_directories` form is used here for clarity in a single-target project.

**`add_definitions(${LLVM_DEFINITIONS_LIST})`** — Applies macros like `_GNU_SOURCE`, `__STDC_CONSTANT_MACROS`, `__STDC_FORMAT_MACROS`, and `__STDC_LIMIT_MACROS`. These are required by LLVM's C headers and omitting them causes compilation failures in `<stdint.h>` on some platforms. The `separate_arguments` call is needed because `LLVM_DEFINITIONS` is a space-separated string, not a CMake list.

**`llvm_map_components_to_libnames(llvm_libs core support irreader)`** — This function (defined in `AddLLVM.cmake`) resolves the named component graph, finds the actual library names (which may be `LLVMCore`, `LLVMSupport`, `LLVMIRReader` plus all transitive dependencies), and populates the `llvm_libs` variable. It handles platform differences (static vs. shared, `.a` vs. `.lib`, library prefix conventions) and respects whether the installation is a static or shared build.

**`target_link_libraries(mytool PRIVATE ${llvm_libs})`** — Links the resolved libraries. The `PRIVATE` keyword is correct here: LLVM's types do not appear in `mytool`'s public API.

### `add_llvm_executable()` versus `add_executable()`

`add_llvm_executable()` is defined in `AddLLVM.cmake` and is the preferred function for tools that are part of (or closely integrated with) the LLVM build system. It adds:

- Automatic `RPATH` setup for shared builds
- Proper handling of `LLVM_ENABLE_PLUGINS` for pass plugin support
- Integration with `llvm-exegesis` benchmarking infrastructure
- The `SUPPORT_PLUGINS` argument to allow the binary to `--load-pass-plugin`

For purely out-of-tree tools, plain `add_executable()` is simpler and more transparent. If you want your tool to support `--load-pass-plugin` loading, you must add the plugin support libraries explicitly — `add_llvm_executable(mytool SUPPORT_PLUGINS mytool.cpp)` does this automatically.

### Matching compiler flags to the LLVM installation

LLVM's `HandleLLVMOptions.cmake` (included by `LLVMConfig.cmake`) exports the RTTI and exception settings used by the installed build. You must match them in your own targets:

```cmake
if(NOT LLVM_ENABLE_RTTI)
    if(MSVC)
        target_compile_options(mytool PRIVATE /GR-)
    else()
        target_compile_options(mytool PRIVATE -fno-rtti)
    endif()
endif()

if(NOT LLVM_ENABLE_EH)
    if(MSVC)
        target_compile_options(mytool PRIVATE /EHs-c-)
    else()
        target_compile_options(mytool PRIVATE -fno-exceptions)
    endif()
endif()
```

Mismatched RTTI settings are a common source of mysterious ODR violations and `dynamic_cast` failures. An LLVM built without RTTI exports no typeinfo records; if your translation unit is compiled with RTTI and uses `dynamic_cast` or `typeid` on LLVM types, you will get link errors or silent failure depending on the platform.

---

## 5.3 The Component Model

LLVM does not ship as a single monolithic library. It is organized into approximately 100 named components — static libraries or shared-library sub-objects — organized around a dependency graph. Understanding this graph is the key to producing minimal, correctly linked binaries.

### The physical organization

Each component corresponds to one subdirectory of `llvm/lib/`. The mapping is:

| Component name | Library file | Source directory |
|---------------|-------------|-----------------|
| `core` | `libLLVMCore.a` | `llvm/lib/IR/` |
| `support` | `libLLVMSupport.a` | `llvm/lib/Support/` |
| `irreader` | `libLLVMIRReader.a` | `llvm/lib/IRReader/` |
| `bitreader` | `libLLVMBitReader.a` | `llvm/lib/Bitcode/Reader/` |
| `bitwriter` | `libLLVMBitWriter.a` | `llvm/lib/Bitcode/Writer/` |
| `analysis` | `libLLVMAnalysis.a` | `llvm/lib/Analysis/` |
| `transforms` | — | Meta-component (see below) |
| `instcombine` | `libLLVMInstCombine.a` | `llvm/lib/Transforms/InstCombine/` |
| `scalaropts` | `libLLVMScalarOpts.a` | `llvm/lib/Transforms/Scalar/` |
| `ipo` | `libLLVMipo.a` | `llvm/lib/Transforms/IPO/` |
| `vectorize` | `libLLVMVectorize.a` | `llvm/lib/Transforms/Vectorize/` |
| `passes` | `libLLVMPasses.a` | `llvm/lib/Passes/` |
| `codegen` | `libLLVMCodeGen.a` | `llvm/lib/CodeGen/` |
| `asmprinter` | `libLLVMAsmPrinter.a` | `llvm/lib/CodeGen/AsmPrinter/` |
| `mc` | `libLLVMMC.a` | `llvm/lib/MC/` |
| `x86` | — | Meta-component for X86 target |
| `x86codegen` | `libLLVMX86CodeGen.a` | `llvm/lib/Target/X86/` |
| `x86asmparser` | `libLLVMX86AsmParser.a` | `llvm/lib/Target/X86/AsmParser/` |
| `x86disassembler` | `libLLVMX86Disassembler.a` | `llvm/lib/Target/X86/Disassembler/` |

The meta-components like `x86` and `transforms` are convenience aliases that expand to the full set of sub-components. `llvm_map_components_to_libnames()` resolves meta-components before emitting the library list.

### The dependency graph

The dependency graph is acyclic and relatively shallow. The critical spine:

```
support
  └── core
        ├── analysis
        │     ├── instcombine
        │     ├── scalaropts
        │     └── ipo
        ├── bitreader
        │     └── irreader
        ├── bitwriter
        └── transforms/utils
              └── transformutils
```

Target backends sit above this spine:

```
codegen
  ├── mc
  │     ├── mcparser
  │     └── asmprinter
  └── selectiondag
        └── <target>codegen
              └── <target>asmparser
```

`llvm-config` and `llvm_map_components_to_libnames()` both traverse this graph correctly. You never need to spell out transitive dependencies yourself; the tools do it. The graph is encoded in each component's `CMakeLists.txt` under `LINK_COMPONENTS` and in `llvm-config`'s embedded component table.

### Minimizing binary size

The most impactful size reduction comes from selecting only the target backends you actually need. Each backend accounts for 5–30 MB of object code. A tool that only processes X86 IR should specify:

```cmake
llvm_map_components_to_libnames(llvm_libs
    core
    support
    irreader
    analysis
    x86codegen
    x86asmparser
    x86desc
    x86info
)
```

rather than `all-targets`. For a tool that only manipulates IR and never lowers to native code — a static analyzer, a bitcode transformer, an IR pretty-printer — you can omit all target backends entirely:

```cmake
llvm_map_components_to_libnames(llvm_libs
    core
    support
    irreader
    bitwriter
    analysis
    instcombine
    scalaropts
    transformutils
)
```

This produces a binary an order of magnitude smaller than one linked with `all-targets`.

### `LLVMBitReader` and `LLVMBitWriter`

These two components deserve special attention because they are often needed but easily overlooked. Any tool that reads `.bc` files (the binary bitcode format) needs `bitreader`. Any tool that writes `.bc` files needs `bitwriter`. The `irreader` component adds reading of the textual `.ll` format and depends on `bitreader`. A common beginner mistake is requesting `core` and `support` but forgetting `bitreader`, then getting a link error for `LLVMParseIRFileOrString` or a runtime failure reading bitcode input.

### Why you need `--system-libs`

LLVM's `Support` library calls into platform APIs: `dlopen` for plugin loading, `pthread_create` for parallel passes, `mmap` and `mprotect` for JIT code emission, terminal detection via `ncurses` or `isatty`. These dependencies are platform libraries, not LLVM libraries, and `llvm-config --system-libs` emits the correct set of `-l` flags for the current platform. On Linux this is typically `-lrt -ldl -lpthread -lm`. On macOS, the system frameworks are handled automatically. Omitting `--system-libs` causes undefined-symbol errors for `pthread_create`, `dlsym`, `pow`, and similar functions during the final link step.

---

## 5.4 The C API: Stable ABI, Portable Bindings

LLVM exposes a C API as a deliberately constrained surface for non-C++ consumers. It is declared in `llvm/include/llvm-c/` and implemented by thin wrappers around the C++ API in `llvm/lib/IR/Core.cpp` and adjacent files.

### Why the C API exists

C++ has no stable ABI across compiler vendors, compiler versions, or sometimes even compiler flags (RTTI on vs. off, exception model). The C++ API of LLVM is completely unstable across major versions — class layouts change, virtual function tables shift, template instantiation decisions alter symbol names. This makes it impossible to ship a compiled LLVM and a compiled consumer tool as separate binaries and have them interoperate reliably.

The C API solves this with a stable, versioned interface that:
- Uses only C types — structs, pointers, enums, `typedef`-ed handle types
- Exports all symbols with C linkage (no C++ name mangling)
- Hides all implementation details behind opaque pointer handles
- Can be consumed from any language with C FFI support: Python, Rust, Go, Java (via JNA), Haskell (via FFI), OCaml, and dozens more

The C API is not intended for C++ callers. C++ consumers should use the C++ API directly. The C API's value is as an FFI target.

### Core handle types

The `llvm-c/Core.h` header defines the primary handle types via a macro pattern:

```c
typedef struct LLVMOpaqueModule *LLVMModuleRef;
typedef struct LLVMOpaqueContext *LLVMContextRef;
typedef struct LLVMOpaqueType *LLVMTypeRef;
typedef struct LLVMOpaqueValue *LLVMValueRef;
typedef struct LLVMOpaqueBasicBlock *LLVMBasicBlockRef;
typedef struct LLVMOpaqueBuilder *LLVMBuilderRef;
typedef struct LLVMOpaquePassManager *LLVMPassManagerRef;
```

Each `LLVMOpaqueXxx` type is declared but never defined: it is a forward-declared struct that can only be used through a pointer. This ensures no consumer can depend on the internal layout.

### Building IR with the C API

The following complete C program builds the IR equivalent of `int add(int a, int b) { return a + b; }`:

```c
#include "llvm-c/Core.h"
#include "llvm-c/BitWriter.h"
#include <stdio.h>

int main(void) {
    LLVMContextRef ctx  = LLVMContextCreate();
    LLVMModuleRef  mod  = LLVMModuleCreateWithNameInContext("mymod", ctx);
    LLVMBuilderRef bld  = LLVMCreateBuilderInContext(ctx);

    /* Define the function type: i32 (i32, i32) */
    LLVMTypeRef   i32   = LLVMInt32TypeInContext(ctx);
    LLVMTypeRef   params[2] = { i32, i32 };
    LLVMTypeRef   ftype = LLVMFunctionType(i32, params, 2, /*vararg=*/0);

    /* Add the function to the module */
    LLVMValueRef fn = LLVMAddFunction(mod, "add", ftype);
    LLVMSetLinkage(fn, LLVMExternalLinkage);

    /* Name the parameters */
    LLVMValueRef a = LLVMGetParam(fn, 0);
    LLVMValueRef b = LLVMGetParam(fn, 1);
    LLVMSetValueName2(a, "a", 1);
    LLVMSetValueName2(b, "b", 1);

    /* Build the entry basic block */
    LLVMBasicBlockRef entry = LLVMAppendBasicBlockInContext(ctx, fn, "entry");
    LLVMPositionBuilderAtEnd(bld, entry);

    /* %sum = add i32 %a, %b */
    LLVMValueRef sum = LLVMBuildAdd(bld, a, b, "sum");

    /* ret i32 %sum */
    LLVMBuildRet(bld, sum);

    /* Emit bitcode to stdout */
    LLVMWriteBitcodeToFD(mod, /*fd=*/1, /*shouldClose=*/0, /*unbuffered=*/0);

    /* Dispose in reverse order */
    LLVMDisposeBuilder(bld);
    LLVMDisposeModule(mod);
    LLVMContextDispose(ctx);
    return 0;
}
```

Key ownership rules:
- `LLVMContextDispose` must be called after `LLVMDisposeModule` (the module holds a reference to the context internally, but LLVM does not enforce disposal order — getting it wrong produces use-after-free).
- `LLVMValueRef` objects for instructions and basic blocks are owned by the enclosing function/module; do not call a dispose function on them individually.
- `LLVMBuilderRef` is an independent object that must be explicitly disposed.
- `LLVMTypeRef` objects are owned by the context and must not be disposed separately.

### Limitations compared to the C++ API

The C API covers perhaps 40% of the C++ API's surface. Notably absent or reduced:
- **The new pass manager.** The C API exposes only the legacy pass manager. Running a new-PM pipeline (`OptimizationLevel`, `PassBuilder`, `ModulePassManager`) requires the C++ API. This is the single largest gap.
- **Module cloning and merging.** `LLVMLinkModules2` exists, but the full linker (`llvm/lib/Linker/`) is not exposed.
- **DebugInfo metadata construction.** `llvm-c/DebugInfo.h` was added in LLVM 7 and covers the DI builder, but the coverage is incomplete for advanced use cases (DI expressions, multi-location variables).
- **Analysis results.** You cannot ask for `DominatorTree`, `LoopInfo`, or alias analysis results through the C API. Analysis is an in-process C++ concern.
- **Target machine and code emission.** `llvm-c/TargetMachine.h` exposes enough to emit object files, but fine-grained target feature control and MIR-level work are not available.

For any non-trivial tool, the C API is a starting point, not a complete solution. Production compiler infrastructure (llvmlite, llvm-sys) uses the C API for basic IR construction and then calls C++ code directly for the parts the C API does not cover.

### Using the C API from Python via ctypes

The `ctypes` approach is illustrative for understanding the ABI, though `llvmlite` is the production Python binding:

```python
import ctypes
import ctypes.util

# Load the shared LLVM library
lib = ctypes.CDLL("/usr/lib/llvm-22/lib/libLLVM-22.so")

# Declare a handful of functions (minimal type annotations)
lib.LLVMContextCreate.restype = ctypes.c_void_p
lib.LLVMModuleCreateWithNameInContext.argtypes = [ctypes.c_char_p, ctypes.c_void_p]
lib.LLVMModuleCreateWithNameInContext.restype  = ctypes.c_void_p
lib.LLVMDumpModule.argtypes = [ctypes.c_void_p]

# Use them
ctx = lib.LLVMContextCreate()
mod = lib.LLVMModuleCreateWithNameInContext(b"demo", ctx)
lib.LLVMDumpModule(mod)
lib.LLVMDisposeModule(mod)
lib.LLVMContextDispose(ctx)
```

The type annotations on `argtypes` and `restype` are optional but important: without them, ctypes defaults to passing arguments as 32-bit `int`s, which corrupts 64-bit pointer values on 64-bit platforms.

### llvmlite and llvm-sys

The production Python binding `llvmlite` (used by Numba and others) wraps the C API via a thin compiled shim that adds the few missing pieces. It ships its own pinned LLVM build rather than linking against whatever LLVM the system provides, because the C API's stability guarantee only holds within a single major version.

The Rust crate `llvm-sys` generates unsafe Rust FFI bindings directly from the LLVM C API headers using `bindgen`. It requires that the LLVM version used at bindgen time matches the version available at link time; like `llvmlite`, it solves the version-pinning problem by vendoring a specific LLVM build.

---

## 5.5 Writing an Out-of-Tree Pass Plugin

The LLVM new pass manager (introduced in LLVM 11, made default in Clang 13) supports dynamically loaded pass plugins. A pass plugin is a shared library that registers passes with the pass builder at load time; the tool (`opt`, `clang`, a custom driver) loads it via `--load-pass-plugin=./MyPlugin.so` and the plugin's passes become available by name in the pipeline string.

### The plugin entry point

Every pass plugin must define exactly one symbol: `llvmGetPassPluginInfo`. This function returns a `PassPluginLibraryInfo` struct that LLVM uses to register the plugin's passes.

```cpp
// MyPlugin.cpp
#include "llvm/IR/Function.h"
#include "llvm/IR/PassManager.h"
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"
#include "llvm/Support/raw_ostream.h"

using namespace llvm;

// The actual pass: a FunctionPass that prints each function's name
struct HelloWorldPass : PassInfoMixin<HelloWorldPass> {
    PreservedAnalyses run(Function &F, FunctionAnalysisManager &) {
        errs() << "HelloWorldPass: visiting function "
               << F.getName() << "\n";
        return PreservedAnalyses::all();
    }

    // Allow the pass to run on functions with optnone attribute
    static bool isRequired() { return true; }
};

// Plugin registration callback
void registerCallbacks(PassBuilder &PB) {
    // Register as a pipeline-parsing callback so users can name it in -passes=
    PB.registerPipelineParsingCallback(
        [](StringRef Name, FunctionPassManager &FPM,
           ArrayRef<PassBuilder::PipelineElement>) -> bool {
            if (Name == "hello-world") {
                FPM.addPass(HelloWorldPass{});
                return true;   // we handled this pass name
            }
            return false;      // not our pass name
        });
}

// The mandatory plugin entry point
extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
    return {
        LLVM_PLUGIN_API_VERSION,  // must match the LLVM version used to build
        "HelloWorldPlugin",       // plugin name for diagnostics
        LLVM_VERSION_STRING,      // LLVM version this was built against
        registerCallbacks         // the registration function
    };
}
```

Four fields in `PassPluginLibraryInfo`:
- `APIVersion` — Must be `LLVM_PLUGIN_API_VERSION`. LLVM checks this at load time; a mismatch causes an immediate error rather than a crash.
- `PluginName` — A human-readable string, used in error messages.
- `PluginVersion` — Typically `LLVM_VERSION_STRING`; used for diagnostics.
- `RegisterPassBuilderCallbacks` — The callback invoked with a `PassBuilder` reference during pipeline construction.

### The CMakeLists.txt for the plugin

```cmake
cmake_minimum_required(VERSION 3.20)
project(HelloWorldPlugin CXX)

find_package(LLVM 22 REQUIRED CONFIG)

message(STATUS "Building plugin against LLVM ${LLVM_PACKAGE_VERSION}")

include_directories(${LLVM_INCLUDE_DIRS})
separate_arguments(LLVM_DEFINITIONS_LIST NATIVE_COMMAND ${LLVM_DEFINITIONS})
add_definitions(${LLVM_DEFINITIONS_LIST})

# Match LLVM's RTTI/exception settings
if(NOT LLVM_ENABLE_RTTI)
    if(MSVC)
        add_compile_options(/GR-)
    else()
        add_compile_options(-fno-rtti)
    endif()
endif()

# Build a shared library, not a static archive or executable
add_library(HelloWorldPlugin MODULE MyPlugin.cpp)

# On macOS, shared libraries loaded as plugins use BUNDLE not MODULE
# MODULE is the correct CMake target type for dlopen-loaded plugins
# Do not add target_link_libraries(... ${llvm_libs}) for a plugin:
# the symbols are resolved from the host process at dlopen time

# Allow undefined symbols (they come from the opt/clang binary that loads us)
if(APPLE)
    target_link_options(HelloWorldPlugin PRIVATE
        -Wl,-undefined,dynamic_lookup)
elseif(UNIX)
    target_link_options(HelloWorldPlugin PRIVATE
        -Wl,-z,nodelete)   # keep the plugin loaded even if refcount drops to 0
endif()
```

The critical point: **do not link the plugin against the LLVM libraries**. The plugin will be loaded into a process (`opt`, `clang`, your custom driver) that already has LLVM loaded. Linking the plugin against LLVM again would produce a second copy of all LLVM globals — including the pass registry, the context, and various singleton `ManagedStatic` objects — resulting in catastrophic double-initialization failures. The plugin gets LLVM symbols from the host process via dynamic linking. The `LLVM_ATTRIBUTE_WEAK` annotation on `llvmGetPassPluginInfo` handles the case where the host binary was linked statically (without a weak annotation, a static binary that also defines the symbol would cause a multiple-definition error).

### Building and loading

```bash
# Build the plugin
cmake -S . -B build -DLLVM_DIR=/opt/llvm-22/lib/cmake/llvm
cmake --build build

# Run on a bitcode file
opt --load-pass-plugin=./build/HelloWorldPlugin.so \
    --passes="hello-world" \
    -disable-output \
    input.bc

# The -disable-output flag prevents opt from writing the (unchanged) IR to stdout.
# Expected output:
# HelloWorldPass: visiting function main
# HelloWorldPass: visiting function foo
```

The `--load-pass-plugin` flag is the new-PM mechanism. The legacy `--load` flag (which loaded legacy pass manager plugins) is deprecated and may be removed in a future release. Always use `--load-pass-plugin`.

### Inserting the plugin pass into a pipeline position

The pipeline-parsing callback approach shown above makes the pass available by name in `-passes=` strings. You can also register the pass at fixed pipeline extension points:

```cpp
void registerCallbacks(PassBuilder &PB) {
    // Run HelloWorldPass after every optimization pass (EP_OptimizerLast)
    PB.registerOptimizerLastEPCallback(
        [](ModulePassManager &MPM, OptimizationLevel) {
            MPM.addPass(createModuleToFunctionPassAdaptor(HelloWorldPass{}));
        });

    // Also make it available by name for -passes= strings
    PB.registerPipelineParsingCallback(
        [](StringRef Name, FunctionPassManager &FPM,
           ArrayRef<PassBuilder::PipelineElement>) -> bool {
            if (Name == "hello-world") {
                FPM.addPass(HelloWorldPass{});
                return true;
            }
            return false;
        });
}
```

Extension points that `PassBuilder` provides for plugins:

| Extension point | When it fires |
|-----------------|---------------|
| `registerPeepholeEPCallback` | Inside peephole optimization loops |
| `registerLateLoopOptimizationsEPCallback` | After most loop optimizations |
| `registerLoopOptimizerEndEPCallback` | At the end of the loop optimizer |
| `registerScalarOptimizerLateEPCallback` | After scalar optimizations |
| `registerCGSCCOptimizerLateEPCallback` | After CGSCC (call-graph SCC) optimizations |
| `registerVectorizerStartEPCallback` | Before vectorization |
| `registerPipelineStartEPCallback` | At the very start of the module pipeline |
| `registerOptimizerLastEPCallback` | At the very end of the module pipeline |
| `registerFullLinkTimeOptimizationEarlyEPCallback` | Early in the LTO pipeline |
| `registerFullLinkTimeOptimizationLastEPCallback` | Late in the LTO pipeline |

Pass internals — how to write analyses, how to query `FunctionAnalysisManager`, how to declare preservation of analysis results — are covered in Chapter 60 (Writing a Pass).

---

## 5.6 Out-of-Tree Backend Skeleton

Adding a new code-generation target to LLVM requires significantly more work than writing a pass plugin, and in practice almost all production backends live in-tree. This section explains the registration mechanism and gives a skeleton for the rare cases where an out-of-tree backend makes sense.

### Why backends are almost always in-tree

A backend depends on `llvm/lib/CodeGen/`, `llvm/lib/MC/`, `llvm/lib/Support/`, and the TableGen-generated files that describe the target's instruction set. TableGen must run at build time to produce `.inc` files that the backend includes. This means:

1. The backend's build system must invoke TableGen during LLVM's own build — it cannot be decoupled.
2. The TableGen language and the APIs it generates are unstable across major versions. An out-of-tree backend that compiled against LLVM 21 typically needs non-trivial porting work for LLVM 22.
3. The backend's `CMakeLists.txt` must integrate with `llvm/lib/Target/CMakeLists.txt`, which is inside the LLVM build tree.

Despite these constraints, some projects maintain out-of-tree backends: experimental hardware startups, university research projects, and vendors who cannot upstream their target for business reasons. The mechanism exists; it is just painful.

### Target registration

Target registration happens through a set of static initialization callbacks stored in `TargetRegistry`. Each backend provides one or more registration functions, conventionally named `LLVMInitializeXxxTarget`, `LLVMInitializeXxxTargetInfo`, `LLVMInitializeXxxTargetMC`, `LLVMInitializeXxxAsmPrinter`, and `LLVMInitializeXxxAsmParser`. These are called at process startup if the backend is linked in, or can be called explicitly.

The registration functions call into `TargetRegistry`:

```cpp
// TargetInfo/MyTargetTargetInfo.cpp
#include "llvm/MC/TargetRegistry.h"
#include "MyTarget.h"

using namespace llvm;

Target &llvm::getTheMyTarget() {
    static Target TheMyTarget;
    return TheMyTarget;
}

extern "C" LLVM_EXTERNAL_VISIBILITY
void LLVMInitializeMyTargetTargetInfo() {
    RegisterTarget<Triple::mytarget> X(
        getTheMyTarget(),
        "mytarget",          // registered name (used in Triple parsing)
        "My Custom Target",  // human-readable name
        "MyTarget"           // backend name (used for pass naming)
    );
}
```

The `RegisterTarget<>` template, on construction, calls `TargetRegistry::RegisterTarget()` with function pointers for target construction, MCInstrInfo, MCRegisterInfo, MCSubtargetInfo, MCTargetAsmParser, MCCodeEmitter, and several other MC-layer components. Each is registered with its own `RegisterXxx` template call.

### The `LLVM_EXPERIMENTAL_TARGETS_TO_BUILD` variable

For backends that are not ready for the standard `LLVM_TARGETS_TO_BUILD` list, LLVM provides:

```bash
cmake -S llvm -B build \
  -DLLVM_TARGETS_TO_BUILD="X86;AArch64" \
  -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD="MyTarget"
```

Experimental targets are included in the build but excluded from the defaults. They receive relaxed CI requirements and are not subject to the same API-stability expectations as production targets. Several current experimental targets include SPIRV and M68k (which were eventually promoted to official targets after stabilization).

### A minimal skeleton `CMakeLists.txt` for an experimental in-tree backend

```cmake
# llvm/lib/Target/MyTarget/CMakeLists.txt
set(LLVM_TARGET_DEFINITIONS MyTarget.td)

# Run TableGen to produce instruction info, register info, etc.
tablegen(LLVM MyTargetGenRegisterInfo.inc      -gen-register-info)
tablegen(LLVM MyTargetGenInstrInfo.inc         -gen-instr-info)
tablegen(LLVM MyTargetGenSubtargetInfo.inc     -gen-subtarget)
tablegen(LLVM MyTargetGenCallingConv.inc       -gen-callingconv)
tablegen(LLVM MyTargetGenAsmWriter.inc         -gen-asm-writer)
tablegen(LLVM MyTargetGenDAGISel.inc           -gen-dag-isel)

add_public_tablegen_target(MyTargetCommonTableGen)

add_llvm_target(MyTargetCodeGen
  MyTargetAsmPrinter.cpp
  MyTargetFrameLowering.cpp
  MyTargetISelDAGToDAG.cpp
  MyTargetISelLowering.cpp
  MyTargetInstrInfo.cpp
  MyTargetRegisterInfo.cpp
  MyTargetSubtarget.cpp
  MyTargetTargetMachine.cpp

  LINK_COMPONENTS
    Analysis
    AsmPrinter
    CodeGen
    Core
    MC
    SelectionDAG
    Support
    Target
    TransformUtils

  ADD_TO_COMPONENT MyTarget
)

add_subdirectory(TargetInfo)
add_subdirectory(MCTargetDesc)
```

The `add_llvm_target()` macro is what registers the target with the build system and causes the initialization function to be included in `InitializeAllTargets.h` and similar umbrella headers.

### Out-of-tree backend linkage (the research-project case)

If you truly need an out-of-tree backend (for example, because you cannot modify the LLVM source tree), the approach is to build LLVM from source with `LLVM_EXPERIMENTAL_TARGETS_TO_BUILD` pointing at a symlink or subdirectory outside the monorepo. This is supported but awkward: the backend's source directory must be placed (or symlinked) into `llvm/lib/Target/`, the `llvm/lib/Target/CMakeLists.txt` must be patched to `add_subdirectory(MyTarget)`, and the top-level `CMakeLists.txt` must include `MyTarget` in the target list. In practice, this means patching the build system. A truly independent out-of-tree backend, built against an installed LLVM without modifying its source, is not feasible with current LLVM architecture — the TableGen dependency prevents it.

---

## 5.7 Troubleshooting: Common Linking and Integration Errors

This section documents the linking errors that appear repeatedly in the LLVM mailing lists and Stack Overflow, explains their root cause, and gives the corrective action.

### Undefined reference to `LLVMInitializeX86Target` (and variants)

**Symptom:**
```
undefined reference to `LLVMInitializeX86Target()'
undefined reference to `LLVMInitializeX86TargetInfo()'
undefined reference to `LLVMInitializeX86TargetMC()'
```

**Cause:** Your code calls `InitializeAllTargets()` (or one of the target-specific `LLVMInitializeXxxTarget` family) but has not linked the X86 backend libraries. `InitializeAllTargets()` is a macro that expands to calls to every registered target's init function; if those libraries are not linked, the symbols are missing.

**Fix:** Add the target component(s) to your library list:
```cmake
llvm_map_components_to_libnames(llvm_libs
    ...
    x86codegen    # adds LLVMInitializeX86Target
    x86info       # adds LLVMInitializeX86TargetInfo
    x86desc       # adds LLVMInitializeX86TargetMC
)
```
Or use the `all-targets` pseudo-component if you need all targets and binary size is not a concern.

### Multiple definition of `LLVMInitializeX86Target`

**Symptom:**
```
/usr/bin/ld: libLLVMX86CodeGen.a(X86TargetMachine.cpp.o): multiple definition of
  `LLVMInitializeX86Target'; libLLVMX86CodeGen.a(X86TargetMachine.cpp.o): already
  defined here
```

**Cause:** The same static library is appearing on the link line twice, most often because `llvm_map_components_to_libnames` is being called multiple times with overlapping component sets and the results are concatenated. Or the same component appears both explicitly and through a meta-component expansion.

**Fix:** Call `llvm_map_components_to_libnames` once with the full component list. If you are accumulating components across multiple `CMakeLists.txt` files, collect them into a list variable first, then call the function once:
```cmake
list(APPEND my_components core support irreader x86codegen x86info)
llvm_map_components_to_libnames(llvm_libs ${my_components})
target_link_libraries(mytool PRIVATE ${llvm_libs})
```

### `error: unknown argument: '-fno-rtti'` when compiling against LLVM

**Symptom:** Compilation fails with an unsupported flag error.

**Cause:** `llvm-config --cxxflags` includes flags appropriate for the compiler that built LLVM (e.g., Clang), but your tool is being compiled with a different compiler (e.g., an older GCC that uses a different flag syntax or does not recognize the flag).

**Fix:** Either compile with the same compiler family that built LLVM, or filter `llvm-config --cxxflags` output to remove flags your compiler does not understand. In CMake, use `check_cxx_compiler_flag()` to conditionally add flags.

### `LLVM_ENABLE_RTTI` mismatch: strange `dynamic_cast` failures or linker errors involving typeinfo

**Symptom:** Either link-time errors of the form:
```
undefined reference to 'typeinfo for llvm::Value'
```
or silent runtime failures where `dyn_cast<>` always returns null.

**Cause:** LLVM was built without RTTI (`-fno-rtti`), but your code was compiled with RTTI (or vice versa). C++ typeinfo records are not emitted for LLVM classes in the no-RTTI build, so `dynamic_cast` cannot work and typeinfo symbols are absent.

**Fix:** Match the RTTI setting. In CMake, query `LLVM_ENABLE_RTTI` after `find_package(LLVM)` and apply `-fno-rtti` to your targets if it is `OFF`. LLVM provides its own `dyn_cast<>`, `cast<>`, and `isa<>` templates in `llvm/Support/Casting.h` that work without RTTI and should replace `dynamic_cast` in all code that operates on LLVM types.

### Plugin load failure: `Plugin API version mismatch`

**Symptom:**
```
error: Failed to load passes from './MyPlugin.so'.
Plugin API version mismatch: got 42, expected 44.
```

**Cause:** The plugin was compiled against a different LLVM major version than the `opt`/`clang` binary that is loading it. `LLVM_PLUGIN_API_VERSION` is an integer that increments with major releases.

**Fix:** Rebuild the plugin against the exact LLVM version that `opt` was compiled with. Pass plugins are not cross-version compatible. In CI, pin the LLVM version in both the plugin build and the test driver.

### Plugin loads but passes are not available: `-passes=hello-world` gives `unknown pass name`

**Symptom:**
```
opt: error: unknown pass name 'hello-world'
```
even though `--load-pass-plugin=./MyPlugin.so` did not error.

**Cause:** The plugin loaded but `registerCallbacks` was never called, or the pass name string in `registerPipelineParsingCallback` does not exactly match the string given to `-passes=`.

**Fix:** Add a debug print to `registerCallbacks` to verify it fires. Check that the `Name == "hello-world"` comparison matches the case, spelling, and absence of leading/trailing whitespace in the `-passes=` argument. Verify that `llvmGetPassPluginInfo` returns the correct function pointer in the `RegisterPassBuilderCallbacks` field.

### Linker error: `cannot find -lLLVMCore` (static build not found)

**Symptom:**
```
/usr/bin/ld: cannot find -lLLVMCore
```

**Cause:** Your LLVM installation is a shared-library build (`libLLVM-22.so`), but you requested static linking via `--link-static` or the components are being listed as `-lLLVMCore` which does not exist in a shared-only installation.

**Fix:** Use `llvm-config --shared-mode` to determine whether the installation is shared or static. Use `--link-shared` with a shared installation. If you need static linking, build LLVM from source with `-DBUILD_SHARED_LIBS=OFF` (note: this is different from `LLVM_BUILD_LLVM_DYLIB=ON` which builds the monolithic `libLLVM.so`).

### `LLVMContext` initialized multiple times: crashes in `ManagedStatic`

**Symptom:** Sporadic crashes, double-free errors, or assertion failures in `ManagedStatic::operator->` when running a pass plugin.

**Cause:** The pass plugin was accidentally linked against LLVM libraries (see §5.5 above). There are now two copies of every LLVM `ManagedStatic` object in the process — one in the host binary, one in the plugin. The first destruction hits the singleton in the plugin's copy, then the host binary's shutdown hits its copy again.

**Fix:** Remove all `target_link_libraries(MyPlugin ...)` calls that reference LLVM component libraries. The plugin must get LLVM symbols exclusively from the host process. Use `-Wl,-undefined,dynamic_lookup` (macOS) or ensure the host binary exports LLVM symbols (`-Wl,-E` or `-rdynamic` on Linux) to make them visible to `dlopen`-loaded plugins.

### `opt: error: '--load' is not supported with the new pass manager`

**Symptom:**
```
opt: error: '-load' is only supported with the legacy pass manager
```

**Cause:** You are using the old `--load` flag (legacy pass manager) with `opt`'s new pass manager mode, which is the default since LLVM 16.

**Fix:** Switch to `--load-pass-plugin`. If you are maintaining a legacy pass, either port it to the new pass manager or explicitly invoke the legacy PM with `--enable-new-pm=false` (deprecated, may be removed in a future release).

---

## 5.8 Chapter Summary

- **`llvm-config`** is the fast integration path: `$(llvm-config --cxxflags)` for compilation, `$(llvm-config --ldflags --libs <components>)` for linking, `$(llvm-config --system-libs)` for platform dependencies. Always include `--system-libs` or you will get undefined-symbol errors for `dlopen`, `pthread_create`, and math functions.

- **`find_package(LLVM REQUIRED CONFIG)`** is the correct CMake integration. Set `LLVM_DIR` to point at `<install>/lib/cmake/llvm/` when multiple LLVM versions are installed. Use `llvm_map_components_to_libnames()` to resolve component names to library targets. Match `LLVM_ENABLE_RTTI` and `LLVM_ENABLE_EH` in your own targets.

- **The component model** organizes LLVM into approximately 100 named libraries with an explicit dependency graph. List only the components you use to minimize binary size. Never use `all-targets` in production tools. `bitreader` and `bitwriter` are commonly forgotten for tools that process `.bc` files.

- **The C API** (`llvm-c/Core.h`) provides a stable, C-linkage surface for FFI consumers: Python (llvmlite), Rust (llvm-sys), Go, and others. It covers basic IR construction and bitcode I/O well. It does not cover the new pass manager, analysis results, or most of the MC layer. For C++ tools, use the C++ API directly.

- **Out-of-tree pass plugins** are the standard mechanism for distributing passes that do not belong in the LLVM tree. The plugin must define `llvmGetPassPluginInfo()` returning `PassPluginLibraryInfo`, register passes via `PassBuilder::registerPipelineParsingCallback` or one of the extension-point callbacks, build as a `MODULE` shared library, and **not** link against LLVM libraries. Load with `opt --load-pass-plugin=./MyPlugin.so --passes="my-pass"`.

- **Out-of-tree backends** are possible but require integration with the TableGen build step and are nearly always done by placing the backend directory inside `llvm/lib/Target/` and enabling it via `LLVM_EXPERIMENTAL_TARGETS_TO_BUILD`. A fully independent out-of-tree backend without build-tree modifications is not currently feasible.

- **Common errors** group into three families: missing components (add them to `llvm_map_components_to_libnames`), RTTI/exception flag mismatch (query `LLVM_ENABLE_RTTI` and match it), and plugin double-initialization (never link a plugin against LLVM libraries). The `--load` vs `--load-pass-plugin` distinction catches everyone who learned LLVM pass loading before LLVM 13.


---

@copyright jreuben11
