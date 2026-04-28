# Chapter 2 — Building LLVM from Source

*Part I — Foundations*

Building LLVM from source is not optional knowledge for a compiler engineer working in the LLVM ecosystem. The binary distributions provided by Linux distributions lag by months and strip out headers, CMake exports, and optional components. If you intend to modify passes, write out-of-tree backends, instrument the compiler itself, or profile LLVM under PGO, you need a build you control. This chapter walks through every consequential CMake variable, explains the architectural distinction between projects and runtimes, dissects multi-stage bootstrapping, covers the full menu of build accelerators, and shows how to produce a cross-compiler targeting a foreign architecture — all verified against LLVM 22.1.x.

---

## 1. Prerequisites and Dependencies

### Host compiler requirements

LLVM's CMake scripts enforce a minimum host compiler version. As of LLVM 22, the requirements are:

| Compiler | Minimum version | Notes |
|----------|----------------|-------|
| GCC | 7.4 | libstdc++ ≥ 7 required; GCC 10+ strongly recommended |
| Clang | 6.0 | Use a recent system Clang or bootstrapped Clang for best results |
| Apple Clang | 12.0 | Ships with Xcode 12+ |
| MSVC | 19.14 (VS 2017 15.7) | Windows only; 64-bit toolchain required |

In practice, using the system Clang (here, 22.1.3) as the host compiler for all builds is the right default on Linux — it produces tighter code than GCC for the LLVM sources and enables LLVM-specific CMake optimizations like `-gsplit-dwarf`.

### Build system

CMake 3.20 or later is required. CMake 3.27+ unlocks `CMAKE_UNITY_BUILD_BATCH_SIZE` tuning and better dep-file handling for Ninja. Ninja 1.9+ is required; Ninja 1.11+ is recommended. On Ubuntu 24.04 or Debian bookworm, `apt install cmake ninja-build` delivers current enough versions.

### Required system libraries

The core LLVM build has no mandatory third-party library dependencies beyond the C++ runtime. Optional features pull in additional dependencies:

| Feature | Dependency | CMake variable |
|---------|-----------|----------------|
| Python bindings | Python 3.6+ dev headers | `LLVM_ENABLE_BINDINGS` |
| XML documentation | libxml2 | `LLVM_ENABLE_LIBXML2` |
| z3 SMT solver | libz3-dev | `LLVM_ENABLE_Z3_SOLVER` |
| Terminfo/ncurses | libncurses-dev | `LLVM_ENABLE_TERMINFO` |
| zlib compression | libz-dev | `LLVM_ENABLE_ZLIB` |
| BOLT profiling | perf/libbfd | `BOLT_ENABLE_RUNTIME_LIBS` |

For a focused developer build, disable every optional dependency that you do not actively use — it speeds up CMake configuration and avoids spurious link-time surprises.

### Disk space and RAM

A full debug build of LLVM + Clang + LLD occupies approximately 80–120 GB of disk space. The debug-info explosion is the dominant factor: DWARF for `clang` alone can exceed 30 GB. A `RelWithDebInfo` build (the practical default) takes 25–40 GB. A plain `Release` build lands around 8–12 GB. Plan for at least 150 GB free if you intend to run both a debug and a release tree simultaneously.

Peak RAM during linking scales with link job parallelism. A debug link of `libclang-cpp.so` can require 12–16 GB per job. Set `-DLLVM_PARALLEL_LINK_JOBS=2` on machines with 32 GB, and to 1 on machines with 16 GB. The Ninja `--j` flag controls compile parallelism independently.

### Python and the test infrastructure

LLVM's test infrastructure — `lit` (LLVM Integrated Tester) and `FileCheck` — are required if you plan to run tests. `lit` is a Python package installed via pip:

```bash
pip install lit
```

The `llvm-lit` wrapper in `build/bin/` is a project-specific lit instance that knows the build configuration. `FileCheck` is compiled as part of the LLVM build (it is a C++ binary, not a Python script). Always verify that `FileCheck` in your `PATH` is from your build tree, not from a system installation, because different versions have different directive syntax:

```bash
which FileCheck
# Should be build/bin/FileCheck or /usr/lib/llvm-22/bin/FileCheck
FileCheck --version
```

---

## 2. Cloning the Repository

The LLVM project uses a single monorepo at `https://github.com/llvm/llvm-project`. All major subprojects — Clang, LLD, LLDB, libc++, compiler-rt, Flang, MLIR, and others — live under the top-level directory. The layout is flat: each subproject has its own top-level directory rather than being a nested git submodule.

```bash
# Shallow clone for working on a specific release
git clone --depth=1 --branch llvmorg-22.1.3 \
  https://github.com/llvm/llvm-project.git
cd llvm-project

# Full clone for development (history required for bisection)
git clone https://github.com/llvm/llvm-project.git
cd llvm-project
git checkout llvmorg-22.1.3
```

The monorepo is large: a full clone transfers approximately 2–3 GB. The `--filter=blob:none` partial clone option (blobless clone) reduces this to under 500 MB while still allowing full bisection:

```bash
git clone --filter=blob:none \
  https://github.com/llvm/llvm-project.git
```

Blobless clones fetch blob objects on demand. They are appropriate for CI and initial exploration but can produce delays the first time you run `git log -p` on a file not yet fetched.

The CMake entry point is `llvm/CMakeLists.txt`. You always invoke CMake pointing at `<src>/llvm`, not at the top-level monorepo directory. Subprojects are pulled in via `LLVM_ENABLE_PROJECTS` or `LLVM_ENABLE_RUNTIMES` (covered in §5).

Out-of-tree build directories are mandatory — in-source builds are explicitly rejected by the CMake scripts. Convention is `build/` at the monorepo root:

```bash
mkdir build && cd build
cmake -G Ninja [options] ../llvm
```

### Keeping up with upstream

For active development on the main branch, the standard workflow is to rebase your local changes on top of the latest upstream:

```bash
git fetch origin
git rebase origin/main
```

For bisection — finding which commit introduced a regression — the blobless clone is ideal because git can reconstruct any historical commit without fetching all historical object data upfront. With a shallow clone (`--depth=1`), bisection requires deepening the clone first (`git fetch --unshallow`), which can be slow.

When working on a release branch fix (e.g., backporting a patch to `release/22.x`):

```bash
git checkout -b fix/my-patch origin/release/22.x
# ... make changes ...
git cherry-pick <commit-from-main>
```

LLVM uses GitHub pull requests for all code review. The `llvm/utils/git/` directory contains helper scripts including `git-llvm`, which automates the process of pushing a branch and opening a pull request against the correct upstream branch.

---

## 3. CMake Configuration In Depth

### The minimal developer invocation

```bash
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="X86;AArch64" \
  -DLLVM_ENABLE_ASSERTIONS=ON \
  ../llvm
```

Every flag here is load-bearing:

- `-G Ninja` — selects Ninja as the build system (§4)
- `CMAKE_BUILD_TYPE=RelWithDebInfo` — optimized with debug info retained (§3.2)
- `LLVM_ENABLE_PROJECTS="clang;lld"` — enables Clang and LLD in the same build tree (§5)
- `LLVM_TARGETS_TO_BUILD="X86;AArch64"` — restricts target backend registration to two architectures; without this flag, all ~25 targets are compiled, roughly doubling build time
- `LLVM_ENABLE_ASSERTIONS=ON` — turns on `llvm_unreachable`, `assert`, and all `LLVM_DEBUG` checks; essential for development but costs ~15% runtime in hot paths

### Build types

LLVM supports the four standard CMake build types plus a few LLVM-specific hybrids:

| `CMAKE_BUILD_TYPE` | Optimization | Debug info | Assertions default | Use case |
|---------------------|-------------|-----------|-------------------|----------|
| `Debug` | `-O0` | Full DWARF | ON | Step-through debugging, gdb/lldb sessions |
| `RelWithDebInfo` | `-O2` | Split DWARF | OFF (set ON explicitly) | Day-to-day development — fast enough, debuggable |
| `Release` | `-O3` | None | OFF | Shipping binaries, benchmarks |
| `MinSizeRel` | `-Os` | None | OFF | Embedded or space-constrained toolchain distributions |

`RelWithDebInfo` is the workhorse. Setting `LLVM_ENABLE_ASSERTIONS=ON` alongside it gives you optimized-but-asserting code — the standard configuration for serious development work. The combination catches bugs that only manifest under real workloads while still being fast enough to run your full test suite in minutes rather than hours.

### The CMake variable table

The following table covers all variables a compiler engineer is likely to need to touch. Defaults are for a standard Linux/x86_64 host. Full documentation is at `llvm/docs/CMake.rst` in the source tree, and rendered at [https://llvm.org/docs/CMake.html](https://llvm.org/docs/CMake.html).

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `CMAKE_BUILD_TYPE` | STRING | — | `Debug`, `Release`, `RelWithDebInfo`, `MinSizeRel` |
| `CMAKE_INSTALL_PREFIX` | PATH | `/usr/local` | Install destination for `ninja install` |
| `CMAKE_C_COMPILER` | FILEPATH | system cc | Host C compiler for building LLVM itself |
| `CMAKE_CXX_COMPILER` | FILEPATH | system c++ | Host C++ compiler for building LLVM itself |
| `CMAKE_C_COMPILER_LAUNCHER` | STRING | — | Launcher prefix for C compiles (ccache, sccache) |
| `CMAKE_CXX_COMPILER_LAUNCHER` | STRING | — | Launcher prefix for C++ compiles |
| `CMAKE_EXE_LINKER_FLAGS` | STRING | — | Extra linker flags; use `-fuse-ld=lld` to switch to LLD |
| `LLVM_ENABLE_PROJECTS` | STRING | — | Semicolon-separated list of in-tree projects to build |
| `LLVM_ENABLE_RUNTIMES` | STRING | — | Semicolon-separated list of runtimes to build with stage-1 clang |
| `LLVM_TARGETS_TO_BUILD` | STRING | `all` | Semicolons-separated list of target backends; `all` builds every target |
| `LLVM_ENABLE_ASSERTIONS` | BOOL | `OFF` (Release) / `ON` (Debug) | Enable `assert()` and `llvm_unreachable()` |
| `LLVM_ENABLE_EXPENSIVE_CHECKS` | BOOL | `OFF` | Enable very slow internal consistency checks (e.g., dominator recalculation verification) |
| `LLVM_BUILD_TESTS` | BOOL | `OFF` | Build lit-based unit tests |
| `LLVM_BUILD_EXAMPLES` | BOOL | `OFF` | Build tutorial pass examples |
| `LLVM_INCLUDE_TESTS` | BOOL | `ON` | Include test infrastructure in the build |
| `LLVM_INCLUDE_BENCHMARKS` | BOOL | `ON` | Build microbenchmarks using Google Benchmark |
| `LLVM_ENABLE_LTO` | STRING | `OFF` | Enable LTO: `OFF`, `Full`, `Thin` |
| `LLVM_ENABLE_LLD` | BOOL | `OFF` | Use LLD as the linker when building LLVM (requires LLD in `LLVM_ENABLE_PROJECTS` or pre-installed) |
| `LLVM_PARALLEL_COMPILE_JOBS` | STRING | — | Cap compile parallelism (Ninja uses `-j` for this; set here to override globally) |
| `LLVM_PARALLEL_LINK_JOBS` | STRING | — | Cap link parallelism; critical on low-RAM systems |
| `LLVM_USE_SPLIT_DWARF` | BOOL | `OFF` | Emit DWARF into `.dwo` side-car files; dramatically reduces link time and binary size in debug builds |
| `LLVM_OPTIMIZED_TABLEGEN` | BOOL | `OFF` | Build `llvm-tblgen` with optimizations even in debug builds; saves 10–20 min in `Debug` builds |
| `LLVM_CCACHE_BUILD` | BOOL | `OFF` | Shorthand for enabling ccache via `CMAKE_C/CXX_COMPILER_LAUNCHER`; deprecated in favor of the launcher variables |
| `LLVM_BUILD_INSTRUMENTED` | STRING | `OFF` | Instrument the build for PGO: `IR`, `Frontend`, `CSIR` |
| `LLVM_PROFDATA_FILE` | FILEPATH | — | PGO profile data file to apply during optimized builds |
| `LLVM_USE_SANITIZER` | STRING | — | Sanitizer build: `Address`, `Memory`, `MemoryWithOrigins`, `Undefined`, `Thread`, `Address;Undefined` |
| `LLVM_ENABLE_LIBCXX` | BOOL | `OFF` | Link the LLVM tools against libc++ (requires `libcxx` in runtimes) |
| `LLVM_INSTALL_UTILS` | BOOL | `OFF` | Install utility programs like `FileCheck`, `not`, `count` |
| `LLVM_INCLUDE_UTILS` | BOOL | `ON` | Build utility programs |
| `BUILD_SHARED_LIBS` | BOOL | `OFF` | Build LLVM as a collection of shared libraries instead of one monolithic archive; dramatically speeds up linking during development but not suitable for distribution |
| `LLVM_LINK_LLVM_DYLIB` | BOOL | `OFF` | Build and link against `libLLVM.so`, the single combined shared library |
| `LLVM_BUILD_LLVM_DYLIB` | BOOL | `OFF` | Build `libLLVM.so` without linking tools to it |
| `LLVM_HOST_TRIPLE` | STRING | auto-detected | The triple for the machine running the build (host) |
| `LLVM_DEFAULT_TARGET_TRIPLE` | STRING | host triple | The default target triple for Clang when no `-target` is given |
| `CMAKE_CROSSCOMPILING` | BOOL | `OFF` | Set `ON` when host ≠ target |

### Split DWARF and link-time acceleration

In debug builds, the single biggest build-time improvement available is split DWARF. Set:

```bash
-DLLVM_USE_SPLIT_DWARF=ON
```

Split DWARF emits debug information into `.dwo` files alongside object files rather than embedding it in the object files themselves. The linker sees dramatically smaller inputs, reducing link time for `clang` from ~4 minutes to under 60 seconds on an NVMe-backed build.

For development builds, combine with `BUILD_SHARED_LIBS=ON` and `LLVM_OPTIMIZED_TABLEGEN=ON`:

```bash
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DLLVM_ENABLE_ASSERTIONS=ON \
  -DBUILD_SHARED_LIBS=ON \
  -DLLVM_USE_SPLIT_DWARF=ON \
  -DLLVM_OPTIMIZED_TABLEGEN=ON \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=lld" \
  ../llvm
```

`BUILD_SHARED_LIBS=ON` replaces the monolithic `libLLVM.a` (which can be 1.5 GB unoptimized) with per-component shared libraries. Each tool links only what it uses, slashing link times. Do not use this for distribution; the resulting layout is incompatible with downstream CMake `find_package(LLVM)` usage.

### LLVM_TARGETS_TO_BUILD

The full set of backend targets in LLVM 22 is: `AArch64`, `AMDGPU`, `ARC`, `ARM`, `AVR`, `BPF`, `CSKY`, `DirectX`, `Hexagon`, `Lanai`, `LoongArch`, `M68k`, `Mips`, `MSP430`, `NVPTX`, `PowerPC`, `RISCV`, `Sparc`, `SPIR-V`, `SystemZ`, `VE`, `WebAssembly`, `X86`, `XCore`, `Xtensa`. The `all` keyword builds all of them.

For a developer working on X86 optimization, building only `X86` reduces compilation time by roughly 60% compared to `all`. For a cross-compilation toolchain targeting AArch64, use `"X86;AArch64"` (you need the host target too for the JIT and for the tools that introspect target data layouts).

---

## 4. Build Systems: Ninja and Make

### Why Ninja is the standard for LLVM

LLVM's build is structured as hundreds of thousands of independent compilation units with a complex dependency graph. GNU Make evaluates this dependency graph with a top-down recursive approach that introduces significant overhead — Makefiles are re-read on every invocation, and the default shell-subprocess-per-rule model scales poorly.

Ninja was designed specifically for the case where another tool (CMake, in practice) generates the build files and humans never edit them. Its design choices are uniformly favorable for LLVM:

- Ninja reads its entire build graph into memory at startup and never re-evaluates it during a build. On a graph with 200,000 nodes, this saves seconds per incremental build.
- Ninja uses a single progress line that overwrites itself, making the build output readable even at `-j64`.
- Ninja's scheduler is work-stealing and builds the critical path first; Make's parallel mode is less aware of the critical path.
- Ninja's dependency file (`.ninja_deps`, `.ninja_log`) format is binary and O(1) to update, whereas Make's implicit rule scanning is O(n) in directory size.

On a 16-core machine, a full Clang+LLD `RelWithDebInfo` build takes approximately:

| Build system | Full build | Incremental (1 file) |
|-------------|-----------|---------------------|
| Ninja | ~15 min | ~4 sec |
| Make | ~25 min | ~8 sec |

These are rough estimates; actual times depend heavily on disk I/O (NVMe vs SATA), RAM (enough to keep `.o` files in page cache), and compiler flags.

### Invoking Ninja

```bash
# Build everything configured
ninja -C build/

# Build a specific target
ninja -C build/ clang

# Build with explicit parallelism (default: number of CPUs)
ninja -C build/ -j32

# Cap link job parallelism separately via CMake (not a Ninja flag)
# Pass -DLLVM_PARALLEL_LINK_JOBS=4 at cmake time

# Verbose build (shows full compiler commands)
ninja -C build/ -v

# Dry run
ninja -C build/ -n
```

The `LLVM_PARALLEL_LINK_JOBS` CMake variable is important because Ninja otherwise treats link steps like compile steps and will spawn as many linkers as `-j` allows. A debug link of `libclang-cpp.so` can use 12 GB of RAM; eight simultaneous links will exhaust 96 GB. Always set `LLVM_PARALLEL_LINK_JOBS` to `(available RAM in GB) / 14` on debug builds.

### Build artifacts layout

After a successful build:

```
build/
├── bin/                    # All executables: clang, clang++, lld, llvm-config, opt, ...
├── lib/                    # Static archives (.a) and shared objects (.so)
│   ├── libLLVMCore.a
│   ├── libLLVMSupport.a
│   ├── libclang-cpp.so.22  # Shared Clang library (if LLVM_BUILD_LLVM_DYLIB=ON)
│   ├── clang/
│   │   └── 22/
│   │       ├── include/    # Clang resource directory (stddef.h, sanitizer headers)
│   │       └── lib/        # compiler-rt and sanitizer libraries (if built)
│   └── cmake/
│       ├── llvm/           # LLVM CMake exports (find_package(LLVM) targets)
│       └── clang/          # Clang CMake exports
├── include/                # Generated headers (intrinsics tables, etc.)
│   ├── llvm/
│   │   └── IR/             # TableGen-generated intrinsics headers (Intrinsics*.h)
│   └── clang/
├── tools/                  # Source for tooling (mirrors src; not the output)
├── runtimes/               # Runtime build trees (if LLVM_ENABLE_RUNTIMES used)
│   ├── runtimes-bins/      # Runtime executables
│   └── lib/                # Runtime libraries
└── utils/                  # Build utilities (llvm-lit, FileCheck)
```

The `build/bin/llvm-config` binary is the authoritative source for paths, component lists, and compiler flags for downstream out-of-tree projects. Always use it rather than hardcoding paths:

```bash
# Get the include and library flags for an out-of-tree project
build/bin/llvm-config --cxxflags --ldflags --libs core support irreader

# Get the component list
build/bin/llvm-config --components

# Get the build mode
build/bin/llvm-config --build-mode
# RelWithDebInfo
```

The `lib/cmake/llvm/LLVMConfig.cmake` file is used by `find_package(LLVM REQUIRED CONFIG)` in downstream CMakeLists.txt. Point `LLVM_DIR` at `build/lib/cmake/llvm/` when building out-of-tree projects against a build tree rather than an installed LLVM.

---

## 5. LLVM_ENABLE_PROJECTS vs LLVM_ENABLE_RUNTIMES

This is one of the most important and most frequently confused distinctions in the LLVM build system.

### Projects: built with the host compiler

`LLVM_ENABLE_PROJECTS` lists subprojects that are built as part of the same CMake configure-and-build invocation using the **host compiler**. They share the same CMake build tree, the same `build/` directory, and can reference each other's CMake targets directly.

```bash
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DLLVM_ENABLE_PROJECTS="clang;lld;lldb;clang-tools-extra;mlir" \
  ../llvm
```

Valid entries in `LLVM_ENABLE_PROJECTS` for LLVM 22:

| Project | Directory | Description |
|---------|-----------|-------------|
| `clang` | `clang/` | The Clang C/C++/ObjC frontend |
| `clang-tools-extra` | `clang-tools-extra/` | clangd, clang-tidy, clang-format |
| `lld` | `lld/` | The LLD linker |
| `lldb` | `lldb/` | The LLDB debugger |
| `mlir` | `mlir/` | The MLIR framework |
| `flang` | `flang/` | The Flang Fortran compiler |
| `polly` | `polly/` | The Polly polyhedral optimizer |
| `bolt` | `bolt/` | BOLT post-link optimizer |
| `openmp` | `openmp/` | OpenMP runtime (libiomp5) |

The key semantic: because projects are compiled with the host compiler, there is no chicken-and-egg problem. You do not need a freshly-built Clang to compile Clang itself. This is what makes a single-stage build possible.

### Runtimes: built with the just-built compiler

`LLVM_ENABLE_RUNTIMES` lists subprojects that are compiled using the **compiler produced by this build** (the stage-1 Clang), not the host compiler. They have a separate CMake configuration and build area under `build/runtimes/`. This is the correct way to build `libc++`, `compiler-rt`, `libunwind`, and `libc` because these libraries must be compiled with exactly the version of Clang that will ship alongside them.

```bash
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libunwind;compiler-rt" \
  ../llvm
```

Valid entries in `LLVM_ENABLE_RUNTIMES`:

| Runtime | Directory | Description |
|---------|-----------|-------------|
| `compiler-rt` | `compiler-rt/` | builtins, sanitizer runtimes, profiling support, BlocksRuntime |
| `libcxx` | `libcxx/` | LLVM's libc++ implementation |
| `libcxxabi` | `libcxxabi/` | libc++abi, the C++ ABI library underlying libc++ |
| `libunwind` | `libunwind/` | LLVM libunwind, the unwinder library |
| `libc` | `libc/` | LLVM's emerging full libc implementation |
| `openmp` | `openmp/` | OpenMP runtime (can appear in either list; runtimes gives correct compiler) |
| `pstl` | `pstl/` | The Parallel STL (ParallelSTL / oneTBB-backed) |

### When to use which

The practical rule: use `LLVM_ENABLE_PROJECTS` for everything that is a compiler or tool, and `LLVM_ENABLE_RUNTIMES` for everything that is a library that gets shipped alongside the compiler and must match its ABI.

If you are building a toolchain for distribution — a Clang that users will `apt install` — you should build the runtimes separately so they are compiled with the finished Clang, not with the host GCC. For personal development where you just want to iterate on the middle end, `LLVM_ENABLE_PROJECTS="clang;lld"` with no runtimes at all is the right answer.

The build graph difference is also visible in the build directory: `build/bin/clang` comes from the projects build; `build/runtimes/runtimes-bins/` (or `build/lib/`) contains runtime artifacts. The runtimes CMake configure step runs as a custom target during the build (after Clang is compiled), not at the outer CMake configure time.

---

## 6. Stage Builds: From 1-Stage to 3-Stage Bootstrap

### Why stage builds exist

A stage build addresses a fundamental question: which compiler built the compiler you trust? The compiler you ship should be compiled with itself — a property called **compiler consistency** or the bootstrapping property. A self-hosted Clang binary that passes all its own tests is stronger evidence of correctness than one built with a potentially older or buggier host GCC.

Beyond correctness, stage builds enable PGO (profile-guided optimization) and LTO on the compiler binary itself, which yield 10–20% compile-speed improvements.

### 1-Stage build

The minimal build. The host compiler (any sufficiently recent Clang or GCC) builds LLVM + Clang directly. The output is a Clang binary, but it was not compiled by itself.

```
Host compiler → stage1 Clang + LLVM tools
```

This is what the minimal developer cmake invocation in §3 produces. It is correct for development and testing but not appropriate for a distribution binary.

### 2-Stage bootstrap build

The standard bootstrap configuration for distribution. LLVM's CMake has built-in support for multi-stage builds through the `CLANG_ENABLE_BOOTSTRAP` option and the `BOOTSTRAP_CMAKE_ARGS` variable.

```
Host compiler → stage1 Clang → stage2 Clang
```

Stage 1 is a minimal, fast build: only the X86 (or native) backend, `RelWithDebInfo`, assertions off. Stage 2 uses the stage-1 Clang to compile the real production binary with `Release` and, typically, LTO.

```bash
# 2-stage bootstrap from the LLVM build directory
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="X86;AArch64" \
  -DCLANG_ENABLE_BOOTSTRAP=ON \
  -DBOOTSTRAP_CMAKE_ARGS="\
    -DCMAKE_BUILD_TYPE=Release;\
    -DLLVM_ENABLE_LTO=Thin;\
    -DLLVM_ENABLE_PROJECTS=clang;lld;\
    -DLLVM_TARGETS_TO_BUILD=X86;AArch64;\
    -DLLVM_ENABLE_ASSERTIONS=OFF" \
  ../llvm

ninja -C build/ stage2
```

The `BOOTSTRAP_CMAKE_ARGS` value is passed verbatim as extra CMake arguments to the stage-2 configure step. The stage-2 build directory is nested at `build/tools/clang/stage2-bins/`. The `stage2` Ninja target is synthesized by the bootstrap CMake machinery.

To install the stage-2 result:

```bash
ninja -C build/ stage2-install
```

### The `stage2` Ninja target and the bootstrap directory

When `CLANG_ENABLE_BOOTSTRAP=ON` is set, the CMake machinery generates a custom target named `stage2` in the outer build. Building `stage2` triggers:

1. Compilation of the stage-1 LLVM + Clang
2. A fresh CMake configure step for stage 2 (in `build/tools/clang/stage2-bins/`) using the stage-1 Clang as `CMAKE_C_COMPILER` and `CMAKE_CXX_COMPILER`
3. Compilation of the stage-2 LLVM + Clang using the stage-1 Clang

The stage-2 artifacts are in `build/tools/clang/stage2-bins/bin/`. To install stage 2:

```bash
ninja -C build/ stage2-install
# or equivalently:
ninja -C build/tools/clang/stage2-bins/ install
```

For CI, you can also build each stage separately — configure stage 1, build it, then manually configure stage 2 using stage 1's Clang — if you need finer control over stage-2 flags than `BOOTSTRAP_CMAKE_ARGS` provides.

### 3-Stage bootstrap: PGO-guided

A 3-stage build adds profile-guided optimization. Stage 1 builds an instrumented compiler; that instrumented compiler compiles a representative workload to collect profiles; stage 3 uses those profiles to build the final optimized compiler.

```
Host → stage1 → stage2 (instrumented) → [collect profiles] → stage3 (PGO-optimized)
```

LLVM's CMake supports this through `LLVM_BUILD_INSTRUMENTED`:

```bash
# Stage 1: host → instrumented stage2
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="X86;AArch64" \
  -DCLANG_ENABLE_BOOTSTRAP=ON \
  -DLLVM_BUILD_INSTRUMENTED=IR \
  -DBOOTSTRAP_CMAKE_ARGS="\
    -DCMAKE_BUILD_TYPE=Release;\
    -DLLVM_ENABLE_LTO=Thin;\
    -DLLVM_ENABLE_PROJECTS=clang;lld;\
    -DLLVM_TARGETS_TO_BUILD=X86;AArch64;\
    -DLLVM_BUILD_INSTRUMENTED=IR" \
  ../llvm

ninja -C build/ stage2

# Generate profiles by compiling a representative workload with the instrumented binary
# The instrumented clang writes .profraw files to /tmp by default
LLVM_PROFILE_FILE="clang-%p.profraw" \
  build/tools/clang/stage2-bins/bin/clang++ -O2 -c large_source.cpp

# Merge profiles
llvm-profdata merge -output=clang.profdata *.profraw

# Stage 3: apply profiles
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="X86;AArch64" \
  -DLLVM_ENABLE_LTO=Thin \
  -DLLVM_PROFDATA_FILE="$(pwd)/clang.profdata" \
  ../llvm

ninja -C build-stage3/
```

In practice, Apple, Google, and LLVM's own CI use variants of this approach. The LLVM project ships pre-collected PGO profiles in `llvm/utils/collect_and_build_with_pgo.py` and the `LLVM_BUILD_INSTRUMENTED_COVERAGE` flag (which uses source-based coverage rather than IR-level instrumentation for coverage-guided profile collection, a distinct use case from PGO).

### Verifying bootstrap correctness: the stage-2 vs stage-3 bit-for-bit test

A key correctness test for a bootstrapped compiler is that stage 2 and stage 3 produce identical output — if the stage-2 compiler compiles itself and produces the same binary as stage 2 did, the compiler is self-consistent. LLVM's CI runs this test for release candidates. Locally:

```bash
# Build stage 2 and stage 3
ninja -C build/ stage2 stage3

# Compare the stage-2 and stage-3 clang binaries
diff build/tools/clang/stage2-bins/bin/clang \
     build/tools/clang/stage2-bins/tools/clang/stage3-bins/bin/clang
# Should produce no output (identical binaries)
```

If the binaries differ, the compiler is not deterministic in its own compilation — a subtle but real indicator of correctness problems. In practice, most differences are benign (embedded build timestamps, PGO counter ordering) and can be suppressed with `-DCLANG_VENDOR=""` and timestamp-elimination flags.

### When to use each stage count

| Stage count | Build time | Use case |
|-------------|-----------|----------|
| 1 | ~15 min | Development, testing, CI quick builds |
| 2 | ~45 min | Distribution builds that need self-hosting consistency |
| 3 | ~90 min + profile collection | Maximum-performance toolchain for distribution |

---

## 7. Build Acceleration: ccache, sccache, distcc, and llvm-cas

### ccache

ccache works by hashing the preprocessed source and all relevant compiler flags, then returning a cached compiled output if the hash matches. For LLVM development — where you frequently check out different commits, bisect, or switch between branches — ccache provides dramatic speedups because the LLVM source files themselves rarely change between bisect steps.

ccache integration with CMake is via `CMAKE_C_COMPILER_LAUNCHER` and `CMAKE_CXX_COMPILER_LAUNCHER`:

```bash
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="X86;AArch64" \
  -DLLVM_ENABLE_ASSERTIONS=ON \
  -DCMAKE_C_COMPILER_LAUNCHER=ccache \
  -DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
  ../llvm
```

Recommended ccache configuration for LLVM in `~/.config/ccache/ccache.conf`:

```ini
max_size = 50G
compression = true
compression_level = 1
# LLVM uses __TIME__ and __DATE__ in very few places; slotting them out
# loses correctness but is acceptable for development
# sloppiness = time_macros  # uncomment only if you understand the tradeoff

# Direct mode bypasses the preprocessor; safe with most LLVM source
direct_mode = true

# Avoid rehashing identical preprocessed output across compilers
hash_dir = false
```

Set the cache directory explicitly if your home directory is on a slow filesystem:

```bash
export CCACHE_DIR=/fast-nvme/ccache
```

On a warm cache, a full clean rebuild of LLVM + Clang takes 2–3 minutes instead of 15 minutes. The first build populates the cache and takes normal time.

### sccache

sccache is a Rust-based drop-in replacement for ccache that adds support for distributed caching and cloud storage backends (S3, GCS, Azure Blob Storage). The integration is identical to ccache:

```bash
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="X86;AArch64" \
  -DCMAKE_C_COMPILER_LAUNCHER=sccache \
  -DCMAKE_CXX_COMPILER_LAUNCHER=sccache \
  ../llvm
```

sccache runs as a local daemon (`sccache --start-server`). For a team sharing an S3 bucket:

```bash
export SCCACHE_BUCKET=my-team-llvm-cache
export SCCACHE_REGION=us-east-1
# AWS credentials from environment or instance profile
sccache --start-server
```

The primary advantage of sccache over ccache for larger teams is that the cache is shared automatically: when any developer or CI machine builds a given TU, the result is available to all others instantly. In practice, a well-configured sccache setup on a CI farm reduces clean-build time by 80–90% because most commits touch only a handful of files.

sccache also handles MSVC on Windows, where ccache integration is less reliable.

### distcc

distcc distributes compilation across a cluster of machines. Unlike sccache, distcc does not cache; it purely parallelizes. For teams with a dedicated compilation cluster, distcc can bring a full debug build from 20 minutes to under 5 minutes.

```bash
# On the build machine
export DISTCC_HOSTS="machine1 machine2 machine3/8"
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DLLVM_ENABLE_PROJECTS="clang" \
  -DCMAKE_C_COMPILER_LAUNCHER="distcc" \
  -DCMAKE_CXX_COMPILER_LAUNCHER="distcc" \
  ../llvm

# distcc and ccache can be combined: pump mode + ccache
cmake ... \
  -DCMAKE_C_COMPILER_LAUNCHER="ccache;distcc" \
  -DCMAKE_CXX_COMPILER_LAUNCHER="ccache;distcc" \
  ...
```

The CMake launcher chain `"ccache;distcc"` is a semicolon-separated list: CMake invokes `ccache distcc clang++ [args]`. ccache first checks its local cache; on a miss, it invokes distcc, which ships the work to a remote host. Remote hits land in the local cache for future use.

distcc requires that all hosts have identical compiler installations (same paths, same versions). This constraint is often managed with NFS-mounted toolchains.

### Choosing between the accelerators

The three tools serve different points in the build-acceleration design space:

| Tool | Local cache | Shared cache | Distributes work | Cloud backend |
|------|------------|-------------|-----------------|---------------|
| ccache | Yes | No (without NFS trick) | No | No |
| sccache | Yes (daemon) | Yes (S3/GCS/Azure) | No | Yes |
| distcc | No | No | Yes | No |

For a solo developer: use ccache. For a small team without cloud infrastructure: sccache with a shared S3 bucket. For a large organization with excess CPU but limited bandwidth: distcc (possibly with ccache fronting it). For the highest throughput on a well-funded CI fleet: sccache with S3 for sharing, and separately a remote execution service (Bazel RBE or Buildbud) for distribution.

### llvm-cas: Content-Addressable Storage for hermetic builds

`llvm-cas` (Content-Addressable Storage) is a newer LLVM project, developed primarily at Apple, that provides a hermetic caching layer for compiler invocations. Rather than hashing preprocessed inputs like ccache, llvm-cas captures the complete semantic content of a compilation: the full include tree, all macro definitions, and module dependencies — producing a fine-grained content address that enables more aggressive deduplication.

The key scenario where llvm-cas outperforms ccache is **C++ module compilation**. When a module interface unit (`.cppm`) is compiled, its output is a Binary Module Interface (BMI) file whose content is shared across all consumers of that module. llvm-cas stores and retrieves BMIs by content address, allowing a module compiled once by any machine in a build cluster to be reused by all others without recompilation.

LLVM's CAS infrastructure lives in `llvm/include/llvm/CAS/` and `llvm/lib/CAS/`. The relevant CMake variable is `LLVM_ENABLE_PLUGINS`, and the CAS plugin is enabled at configure time when the `LLVM_ENABLE_OCAMLDOC` or the appropriate CAS build flag is set (the public CAS API stabilized in the LLVM 21–22 timeframe; consult the in-tree `docs/CAS.rst` for the current build flags).

#### CASFS

CASFS (Content-Addressable Storage File System) is a virtual filesystem layer in Clang that maps source file accesses through the CAS. When CASFS is active, Clang does not read files from the real filesystem but from CAS objects. This enables:

1. **Reproducible builds**: the build graph is fully content-addressed; identical inputs always produce identical outputs regardless of filesystem timestamps
2. **Hermetic caching**: any CAS hit is guaranteed correct because the key includes all transitive inputs
3. **Remote execution**: compilation jobs can be shipped to any machine that has access to the CAS store without copying inputs

#### Apple's use in Xcode

Apple has integrated llvm-cas into Xcode as of Xcode 15 (LLVM 16/17 era) and continues to develop it. The Xcode build system uses a local CAS database (by default in `~/Library/Developer/Xcode/DerivedData/CAS/`) and optionally a shared CAS server for team use. Swift and Clang module compilations in Xcode are CAS-backed when the "Compilation Caching" build setting is enabled.

The practical result is that in a large Xcode workspace with many modules, switching branches or cleaning the derived data does not cause full recompilation — CAS hits from previous builds on the same or other machines are immediately reused.

For open-source LLVM builds, llvm-cas is available but the server-backed deployment (for team sharing) requires the `llvm-cas-server` binary which is currently only in Apple's fork. The local on-disk CAS works with any LLVM 22 build.

### Enabling CAS in a Clang build

To enable CAS-backed compilation caching with the on-disk backend:

```bash
# Configure LLVM with CAS support
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_ENABLE_ASSERTIONS=ON \
  -DLLVM_ENABLE_PLUGINS=ON \
  ../llvm

# Use the built clang with CAS caching
build/bin/clang \
  -Xclang -fcas-path -Xclang ~/.cache/llvm-cas \
  -Xclang -fcache-compile-job \
  -O2 -c myfile.cpp -o myfile.o
```

The `-fcas-path` option specifies the on-disk CAS store directory. The `-fcache-compile-job` flag enables the compile caching layer. On a cache hit, the `.o` file is materialised from the CAS without invoking the compiler backend. The cache is persistent across build sessions and survives `ninja clean`.

---

## 8. Cross-Compilation

### The cross-compilation model

LLVM cross-compilation is a two-step process:

1. Build a **cross-compiler**: a Clang binary that runs on the host machine but produces object code for the target architecture.
2. Optionally build the **runtimes** for the target: libc++, compiler-rt, etc., using the cross-compiler.

The distinction between host, build, and target triples is critical:

- **Build**: the machine where CMake and Ninja are running
- **Host**: the machine where the built artifacts (Clang binary) will run
- **Target**: the machine whose code Clang will produce

For most cross-compilation scenarios, build = host (you are building a cross-compiler on your development machine to run on that same machine). In embedded CI systems, all three may differ.

### Key CMake variables for cross-compilation

| Variable | Meaning |
|----------|---------|
| `LLVM_HOST_TRIPLE` | The triple of the machine where the built tools will run. If not set, auto-detected from the host compiler. |
| `LLVM_DEFAULT_TARGET_TRIPLE` | The default target for Clang when `-target` is not given. Set this to the cross-target. |
| `CMAKE_CROSSCOMPILING` | Set to `ON` explicitly when building a cross-toolchain. Suppresses host-assumes in some CMake logic. |
| `CMAKE_SYSTEM_NAME` | The target OS name (e.g., `Linux`, `Darwin`, `BareMetal`). |
| `CMAKE_SYSROOT` | Path to the sysroot for the target. Clang will pass `--sysroot` to itself and the linker. |
| `CMAKE_TOOLCHAIN_FILE` | Path to a CMake toolchain file that sets all of the above. The cleanest approach for reusable cross-compile configurations. |

### Building a cross-compiler targeting AArch64 Linux

This example builds a Clang that runs on x86_64 and targets `aarch64-linux-gnu`:

```bash
# Install the AArch64 sysroot (Debian/Ubuntu)
apt install gcc-aarch64-linux-gnu \
            binutils-aarch64-linux-gnu \
            qemu-user-static

# Create the build directory
mkdir build-aarch64-cross && cd build-aarch64-cross

cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="AArch64;X86" \
  -DLLVM_HOST_TRIPLE="x86_64-pc-linux-gnu" \
  -DLLVM_DEFAULT_TARGET_TRIPLE="aarch64-linux-gnu" \
  -DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=lld" \
  -DLLVM_ENABLE_ASSERTIONS=OFF \
  ../llvm
```

The resulting `build-aarch64-cross/bin/clang` is an x86_64 executable. When invoked without `-target`, it targets `aarch64-linux-gnu`. Supply it with a sysroot to get correct library paths:

```bash
build-aarch64-cross/bin/clang \
  --sysroot=/usr/aarch64-linux-gnu \
  -O2 hello.c -o hello_aarch64
file hello_aarch64
# hello_aarch64: ELF 64-bit LSB executable, ARM aarch64, ...
```

### Cross-compiling LLVM itself for AArch64

A more advanced scenario: you want a Clang binary that runs natively on an AArch64 host. This requires cross-compiling LLVM itself — not just using LLVM as a cross-compiler.

This is cleanest with a CMake toolchain file:

```cmake
# aarch64-linux-gnu.cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(CMAKE_C_COMPILER clang)
set(CMAKE_CXX_COMPILER clang++)
set(CMAKE_C_COMPILER_TARGET aarch64-linux-gnu)
set(CMAKE_CXX_COMPILER_TARGET aarch64-linux-gnu)

set(CMAKE_SYSROOT /usr/aarch64-linux-gnu)
set(CMAKE_FIND_ROOT_PATH /usr/aarch64-linux-gnu)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)

# Use LLD to avoid needing an aarch64-specific binutils
set(CMAKE_EXE_LINKER_FLAGS "-fuse-ld=lld")
set(CMAKE_SHARED_LINKER_FLAGS "-fuse-ld=lld")
```

Then:

```bash
cmake -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE=../aarch64-linux-gnu.cmake \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="AArch64" \
  -DLLVM_HOST_TRIPLE="aarch64-linux-gnu" \
  -DLLVM_DEFAULT_TARGET_TRIPLE="aarch64-linux-gnu" \
  -DCMAKE_CROSSCOMPILING=ON \
  -DLLVM_TABLEGEN=/path/to/host/build/bin/llvm-tblgen \
  -DCLANG_TABLEGEN=/path/to/host/build/bin/clang-tblgen \
  ../llvm
```

The `LLVM_TABLEGEN` and `CLANG_TABLEGEN` variables are critical: `llvm-tblgen` and `clang-tblgen` run during the build to generate C++ code from `.td` files. In a cross-compilation, these generators must be host binaries (x86_64 executables), not cross-compiled AArch64 binaries. Point these variables at binaries from a prior native host build.

### Bare-metal cross-compilation (arm-none-eabi)

Targeting bare-metal ARM (no OS, no dynamic linking, no standard C library from the host):

```bash
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="ARM;X86" \
  -DLLVM_DEFAULT_TARGET_TRIPLE="arm-none-eabi" \
  -DLLVM_ENABLE_RUNTIMES="compiler-rt" \
  -DRUNTIMES_arm-none-eabi_COMPILER_RT_BAREMETAL_BUILD=ON \
  -DRUNTIMES_arm-none-eabi_COMPILER_RT_BUILD_BUILTINS=ON \
  -DRUNTIMES_arm-none-eabi_COMPILER_RT_BUILD_SANITIZERS=OFF \
  -DRUNTIMES_arm-none-eabi_COMPILER_RT_BUILD_PROFILE=OFF \
  -DRUNTIMES_arm-none-eabi_COMPILER_RT_BUILD_XRAY=OFF \
  -DRUNTIMES_arm-none-eabi_CMAKE_SYSTEM_NAME=BareMetal \
  ../llvm
```

The `RUNTIMES_<triple>_` prefix passes variables into the per-triple runtime build. For bare-metal, you typically want only the compiler builtins (soft-float helpers, integer division stubs, memcpy/memset) and no sanitizers, profile, or XRay infrastructure.

The resulting `compiler-rt` builtins archive goes to:

```
build/lib/clang/22/lib/arm-none-eabi/libclang_rt.builtins.a
```

Clang automatically finds this archive when invoked with `--target=arm-none-eabi` if the Clang resource directory is configured correctly.

---

## 9. Recommended Build Configurations

### Developer build (fast iteration)

Optimizes for rebuild speed and debuggability. Every incremental rebuild of a single file should complete in under 5 seconds.

```bash
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=lld" \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DLLVM_ENABLE_ASSERTIONS=ON \
  -DBUILD_SHARED_LIBS=ON \
  -DLLVM_USE_SPLIT_DWARF=ON \
  -DLLVM_OPTIMIZED_TABLEGEN=ON \
  -DCMAKE_C_COMPILER_LAUNCHER=ccache \
  -DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
  -DLLVM_PARALLEL_LINK_JOBS=4 \
  ../llvm
```

Expected build time on a modern 16-core workstation with NVMe: ~8 minutes cold, ~30 seconds warm (with ccache). The `BUILD_SHARED_LIBS=ON` flag is the dominant speedup: linking against many small `.so` files is much faster than linking against one monolithic `.a`. The tradeoff is that `build/bin/` executables have deep `RPATH` dependencies and cannot be copied to another machine without the shared library tree.

### Release build (for shipping)

Use ThinLTO for link-time inlining without the memory cost of full LTO. Do a 2-stage bootstrap for self-hosting. No assertions.

```bash
# Stage 1 (minimal, just to get a Clang for stage 2)
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="X86;AArch64" \
  -DLLVM_ENABLE_ASSERTIONS=OFF \
  -DCLANG_ENABLE_BOOTSTRAP=ON \
  -DBOOTSTRAP_CMAKE_ARGS="\
    -DCMAKE_BUILD_TYPE=Release;\
    -DLLVM_ENABLE_LTO=Thin;\
    -DCMAKE_EXE_LINKER_FLAGS=-fuse-ld=lld;\
    -DLLVM_ENABLE_PROJECTS=clang;lld;\
    -DLLVM_TARGETS_TO_BUILD=X86;AArch64;\
    -DLLVM_ENABLE_RUNTIMES=libcxx;libcxxabi;libunwind;compiler-rt;\
    -DLLVM_ENABLE_ASSERTIONS=OFF" \
  ../llvm

ninja -C build-release/ stage2
ninja -C build-release/ stage2-install
```

For shipping, also set:

```bash
-DCMAKE_INSTALL_PREFIX=/usr/local/llvm-22
-DLLVM_BUILD_LLVM_DYLIB=ON     # Build libLLVM.so for downstream consumers
-DLLVM_INSTALL_UTILS=ON        # Install FileCheck, not, llvm-lit
```

### Sanitizer build (ASAN + UBSan)

Sanitizer builds are used to find bugs in LLVM's own code — memory errors, undefined behavior, and type-confusion bugs in the pass infrastructure.

```bash
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DLLVM_ENABLE_ASSERTIONS=ON \
  -DLLVM_USE_SANITIZER="Address;Undefined" \
  -DLLVM_PARALLEL_LINK_JOBS=2 \
  -DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=lld" \
  ../llvm
```

`LLVM_USE_SANITIZER` passes the appropriate `-fsanitize=` flags to all compile and link commands via `llvm/cmake/modules/HandleLLVMOptions.cmake`. The valid values are:

| Value | Sanitizers enabled |
|-------|--------------------|
| `Address` | ASan (detects heap/stack/global buffer overflows, use-after-free, use-after-return) |
| `Memory` | MSan (detects reads of uninitialized memory; requires all dependencies compiled with MSan) |
| `MemoryWithOrigins` | MSan with origin tracking (slower but shows where uninitialized memory was created) |
| `Undefined` | UBSan (detects signed overflow, invalid shifts, null dereferences, type aliasing violations) |
| `Thread` | TSan (detects data races) |
| `Address;Undefined` | ASan + UBSan combined; the most common development configuration |
| `DataFlow` | DFSan (data-flow sanitizer; taint tracking) |

For the sanitizer build, set `LLVM_PARALLEL_LINK_JOBS` to 1 or 2 — ASan-instrumented binaries are very large and their link is memory-intensive.

Running tests under the sanitizer build:

```bash
ninja -C build-asan/ check-clang
# lit respects the ASAN_OPTIONS environment variable
ASAN_OPTIONS=detect_leaks=0 ninja -C build-asan/ check-llvm
```

`detect_leaks=0` is commonly needed because LLVM's `ManagedStatic` objects are intentionally not freed at exit (they use a fast shutdown path). The `ManagedStatic` leak is a known false positive; suppressing it makes the sanitizer output actionable.

For UBSan specifically, you will also want to set:

```bash
UBSAN_OPTIONS=print_stacktrace=1:halt_on_error=1
```

`halt_on_error=1` causes the process to abort on the first UBSan finding rather than continuing and potentially producing misleading subsequent errors. `print_stacktrace=1` gives a full stack trace rather than just the instruction address.

MSan builds (detecting reads of uninitialized memory) are the hardest to get right for LLVM because MSan requires that every library linked into the process is also MSan-instrumented. This includes libc++. For a proper MSan build, you first build an MSan-instrumented libc++ (using `LLVM_ENABLE_RUNTIMES="libcxx;libcxxabi"` with `LLVM_USE_SANITIZER=Memory`), then use that libc++ for the LLVM MSan build. The `llvm/utils/sanitizers/` directory contains build scripts that automate this multi-step process.

### Coverage build

Source-based coverage via `llvm-cov` and `llvm-profdata`, not to be confused with the gcov-compatible coverage:

```bash
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DLLVM_BUILD_INSTRUMENTED_COVERAGE=ON \
  ../llvm

# Build and run tests to collect coverage data
ninja -C build-cov/ check-clang

# Merge raw profiles
llvm-profdata merge \
  -sparse \
  build-cov/profiles/*.profraw \
  -o build-cov/merged.profdata

# Generate HTML report
llvm-cov show \
  build-cov/bin/clang \
  -instr-profile=build-cov/merged.profdata \
  -format=html \
  -output-dir=coverage-report/ \
  -show-line-counts-or-regions \
  -show-instantiations \
  $(find build-cov -name "*.cpp" | head -50)

open coverage-report/index.html
```

`LLVM_BUILD_INSTRUMENTED_COVERAGE=ON` sets `-fprofile-instr-generate -fcoverage-mapping` on all LLVM source files. The `.profraw` files are written to a `profiles/` directory under the build tree by default; the directory is created by the CMake scripts.

Coverage builds are substantially slower at runtime (3–5x) due to the counter increment overhead and are not useful for performance analysis. They are the right tool for measuring test coverage of a new pass or subsystem.

### Running specific tests

Every build configuration pairs with the LLVM Integrated Tester (`lit`) via Ninja test targets. The main test targets are:

| Target | What it runs |
|--------|-------------|
| `check-llvm` | All LLVM core tests (`llvm/test/`) |
| `check-clang` | All Clang tests (`clang/test/`) |
| `check-lld` | LLD tests |
| `check-mlir` | MLIR tests |
| `check-all` | All test suites for all enabled projects |
| `clang-test-depends` | Builds test dependencies without running tests |

To run a single test file directly:

```bash
# Run one test with full verbose output
build/bin/llvm-lit -v llvm/test/Analysis/BasicAA/basic.ll

# Run all tests in a directory
build/bin/llvm-lit -j8 llvm/test/Transforms/InstCombine/

# Run tests matching a pattern
build/bin/llvm-lit --filter=".*fold.*" llvm/test/
```

The `-j` flag to `llvm-lit` controls test parallelism independently of the build parallelism. For debugging a flaky test, `-j1` ensures sequential execution.

---

## 10. Chapter Summary

- **CMake is the only supported build system generator for LLVM.** Always use `-G Ninja`; Ninja's superior dependency tracking and parallel scheduling make a measurable difference on LLVM's large build graph.

- **`LLVM_TARGETS_TO_BUILD` is the single most impactful optimization variable.** Restricting to the targets you need reduces build time by 40–70% compared to the `all` default.

- **`CMAKE_BUILD_TYPE=RelWithDebInfo` with `LLVM_ENABLE_ASSERTIONS=ON` is the correct developer default** — optimized enough to be fast, with debug info for backtraces, and assertions to catch invariant violations.

- **`LLVM_ENABLE_PROJECTS` and `LLVM_ENABLE_RUNTIMES` serve fundamentally different roles.** Projects are compiled with the host compiler and share the build tree; runtimes are compiled by the just-built Clang with a separate CMake configure step. Use projects for compilers and tools; use runtimes for libraries that ship alongside the compiler.

- **Multi-stage builds provide correctness (2-stage) and performance (3-stage with PGO).** The `CLANG_ENABLE_BOOTSTRAP` + `BOOTSTRAP_CMAKE_ARGS` mechanism handles the nested CMake configuration automatically. Distribution builds should be at least 2-stage.

- **`BUILD_SHARED_LIBS=ON` + `LLVM_USE_SPLIT_DWARF=ON` + ccache** is the combination that makes day-to-day development fast. Shared libs reduce link time from minutes to seconds; split DWARF reduces binary size and linker input; ccache makes clean rebuilds nearly instant on cached inputs.

- **sccache adds team-shared caching** over ccache's local model, with S3/GCS backends and correct MSVC support. The `CMAKE_C/CXX_COMPILER_LAUNCHER` mechanism is the CMake-endorsed integration point for both.

- **Cross-compilation requires explicit `LLVM_TABLEGEN` and `CLANG_TABLEGEN` pointing at host-native builds.** TableGen generators are build-time tools that must run on the build machine; they cannot be cross-compiled artifacts. A toolchain file (`CMAKE_TOOLCHAIN_FILE`) is the cleanest encapsulation for reusable cross-compile configurations.

- **Sanitizer builds** (`LLVM_USE_SANITIZER="Address;Undefined"`) are the right tool for finding memory and undefined-behavior bugs in LLVM's own code. Set `LLVM_PARALLEL_LINK_JOBS` low to avoid OOM during the link of large instrumented binaries.

- **Coverage builds** (`LLVM_BUILD_INSTRUMENTED_COVERAGE=ON`) use LLVM's own source-based coverage infrastructure (not gcov) and pair with `llvm-profdata merge` + `llvm-cov show` for HTML reports.

---

*Cross-references:*
- *[Chapter 1 — The LLVM Project](ch01-the-llvm-project.md) — the monorepo layout and subproject map*
- *[Chapter 3 — The Compilation Pipeline](ch03-the-compilation-pipeline.md) — how the built tools fit together*
- *[Chapter 5 — LLVM as a Library](ch05-llvm-as-a-library.md) — using `find_package(LLVM)` and `llvm-config` from a build tree*

*Reference documentation:*
- [LLVM CMake documentation](https://llvm.org/docs/CMake.html) — `llvm/docs/CMake.rst`
- [LLVM Bootstrap build documentation](https://llvm.org/docs/AdvancedBuilds.html) — `llvm/docs/AdvancedBuilds.rst`
- [Building LLVM with Clang](https://llvm.org/docs/BuildingADistribution.html) — `llvm/docs/BuildingADistribution.rst`
- [LLVM cross-compilation guide](https://llvm.org/docs/HowToCrossCompileLLVM.html) — `llvm/docs/HowToCrossCompileLLVM.rst`
- [HandleLLVMOptions.cmake](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/cmake/modules/HandleLLVMOptions.cmake) — the authoritative source for how CMake variables translate to compiler flags
