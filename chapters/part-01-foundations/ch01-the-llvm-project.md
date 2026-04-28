# Chapter 1 — The LLVM Project

*Part I — Foundations*

The LLVM Project is the dominant open-source compiler infrastructure of the 2020s. Its IR, optimizer, and backend are used by production compilers for C, C++, Rust, Swift, Julia, Kotlin/Native, CUDA, HIP, SYCL, OpenCL, and dozens of research and industrial languages. Its subproject MLIR is supplanting bespoke IRs across machine-learning frameworks, HPC, and hardware-synthesis toolchains. Clang, built on LLVM, is the default compiler on macOS, iOS, and Android, and is gaining ground on Linux against GCC.

What makes LLVM remarkable is not any single component but the accumulation of design choices that make its components reusable: a stable, well-specified IR that can be emitted, optimized, and inspected independently; a pass infrastructure that decouples analyses from transformations; a code generator that separates target-independent selection from target-specific scheduling and register allocation; and a library-first philosophy that makes each of these independently linkable. The result is that LLVM simultaneously serves as a production compiler driver, a research platform, a JIT engine substrate, an ML compiler framework backend, and a hardware-synthesis toolchain foundation — often in the same build.

Understanding LLVM's structure — its history, its physical organization, its release model, and the design philosophy that makes it so widely reused — is prerequisite knowledge for every topic that follows. This chapter covers all of it.

---

## 1. Origins and Early History

### The Illinois Research Project (2000–2003)

Chris Lattner began work on LLVM as a graduate research project at the University of Illinois at Urbana-Champaign in 2000, under advisor Vikram Adve. The goal was not to build a production compiler but to study *lifelong program analysis and transformation* — the idea that optimization opportunities exploitable at compile time, link time, install time, and runtime should share a common IR and infrastructure. The phrase "Low Level Virtual Machine" reflected the contemporary framing: an IR-based virtual machine, similar in spirit to the JVM or CLR but operating at a lower level of abstraction, closer to native machine code.

Lattner's MS thesis, *LLVM: An Infrastructure for Multi-Stage Optimization*, was completed in December 2002. It introduced the core design: a typed, Static Single Assignment (SSA) form IR, a pass manager for composable optimization, and a target-independent code generator. His PhD dissertation, *Macroscopic Data Structure Analysis and Optimization*, followed in 2005 and explored pool allocation and data structure analysis atop the LLVM infrastructure.

LLVM 1.0 was released in October 2003 under a BSD-style license. At that stage it was a research system capable of compiling C via a front end built on GCC's internal representation (`llvm-gcc`), but the pass infrastructure and code generator were already the recognizable ancestors of today's codebase.

The architectural decisions made in those early years proved remarkably durable. SSA form was chosen for the IR because it simplifies most dataflow analyses: every use reaches exactly one definition, making def-use chains trivially available. The decision to make the IR a first-class object that could be serialized to disk (as `.bc` bitcode) and deserialized — rather than a purely in-memory representation — enabled the link-time optimization (LTO) model that is now a standard compilation mode. The decision to represent the IR in three isomorphic forms — in-memory C++ objects, textual `.ll` files, and binary `.bc` bitcode — gave developers the ability to inspect and debug IR at every stage of the pipeline using standard text tools. These choices, made in a master's thesis in 2002, define the shape of nearly every major LLVM feature that came after.

### Why the "Virtual Machine" Name Was Dropped

As LLVM evolved from a research system into a general-purpose compiler toolkit, the "Low Level Virtual Machine" name became actively misleading. LLVM does not interpret bytecode, has no managed runtime in the JVM sense, and is not primarily a virtual machine in any standard meaning. The project FAQ — [llvm.org/docs/FAQ.html](https://llvm.org/docs/FAQ.html) — now states directly: *"The name 'LLVM' itself is not an acronym; it is the full name of the project."* The initialism is treated as a proper noun. This disambiguation happened gradually through the late 2000s as the project's scope became clear; by LLVM 3.x the virtual-machine framing had essentially disappeared from primary documentation.

### Apple Adoption and the Move to Industry

In 2005, Apple hired Chris Lattner. Apple needed a compiler infrastructure it could control, ship with Xcode, and extend without the GPL-licensing obligations attached to GCC. LLVM's BSD license made it usable in commercial products without source-disclosure requirements — a decisive advantage over GCC's GPLv3 path.

Apple's investment transformed LLVM from a research prototype into production-grade infrastructure. Clang was started at Apple in 2007 to replace the GCC front end entirely; it reached production status for C and Objective-C by 2009 and C++ by 2012. Apple contributed extensive engineering effort to the code generator, the ABI lowering infrastructure, the DWARF debug-info system, and the ARM and x86 backends. LLVM became the default toolchain for macOS and iOS development, giving it enormous real-world workload coverage.

The move from research to industry did not happen only through Apple. The broader industry recognized that LLVM's modular, library-based design made it practical to reuse individual components. AMD, NVIDIA, Intel, IBM, Google, Facebook (now Meta), and Microsoft all began contributing to and building on LLVM through the 2010s. The project transitioned from a single research group's output to a federated industrial commons with dozens of major corporate contributors.

### Open Governance and the LLVM Foundation

The LLVM Foundation was incorporated as a 501(c)(3) nonprofit in 2014. It holds the LLVM trademark, manages the infrastructure (build servers, code review tools, mailing lists), and oversees the annual LLVM Developers' Meeting. Lattner left Apple in 2017 and founded Modular in 2022; by that point the project's governance and development were distributed across many organizations, reducing single-point-of-failure dependency on any one employer or contributor.

The code review workflow also underwent a significant transition during this period. Development originally used Phabricator (the `reviews.llvm.org` instance) as the code-review platform. In 2022–2023, the project migrated to GitHub pull requests as the primary review mechanism, aligning LLVM's workflow with the broader open-source ecosystem and making it substantially easier for new contributors to participate. The Phabricator instance was retired in favor of GitHub PR-based review in 2023. The combination of the monorepo, GitHub-hosted review, and the structured release process represents the mature governance model that LLVM 22 development operates under.

### The License Model

LLVM's original license was a BSD-style license with an advertising clause, which permitted commercial use without source disclosure. In 2019, LLVM initiated a relicensing effort — the result of years of contributor agreement changes — to the [Apache License 2.0 with LLVM Exceptions](https://github.com/llvm/llvm-project/blob/main/LICENSE.TXT). The new Developer Policy and Contributor License Agreement were put in place in 2019, and new contributions from that point forward are under Apache 2.0. Existing files contributed under the old license were progressively relicensed as contributors signed the new CLA; this propagation continued through subsequent releases. The LLVM exception is narrow and specifically addresses the concern that linking compiler runtime libraries (compiler-rt, libcxx, libunwind) into a binary compiled with LLVM would otherwise trigger the Apache 2.0 copyleft-like provisions for the linked binary. The exception permits that linking without propagating the Apache license obligations to the resulting executable. This makes LLVM suitable for building commercial closed-source applications, a property that was essential to Apple's original adoption and that the relicensing preserves.

---

## 2. The Monorepo Architecture

### From Separate Repositories to llvm/llvm-project

LLVM's subprojects — the core IR and optimizer, Clang, LLD, LLDB, compiler-rt, and others — were historically maintained in separate SVN repositories under `llvm.org`. As the ecosystem grew, this fragmentation created friction: coordinated changes requiring edits across multiple repositories needed separate commits, cross-project test failures were hard to reproduce atomically, and build system integration between subprojects required manual synchronization.

The migration to a single GitHub monorepo, [github.com/llvm/llvm-project](https://github.com/llvm/llvm-project), was planned from early 2018 and completed with the final cutover in October 2019. The SVN repositories were retired. All development moved to GitHub with a single `main` branch (previously `master`, renamed in 2021) covering all subprojects simultaneously. A single commit can atomically modify Clang, the LLVM optimizer, MLIR, LLD, and compiler-rt — the workflow that had required coordinated multi-repo patches became a routine single pull request.

The monorepo as of LLVM 22 contains over 7 million lines of code and sees several hundred pull requests merged per week. The migration preserved full commit history via `git filter-branch` and `git-svn` tooling; the oldest commits in the monorepo's history trace back to the original SVN import from the early 2000s. The canonical remote is `https://github.com/llvm/llvm-project.git`; read-only mirrors exist at other locations but `github.com/llvm/llvm-project` is the authoritative origin.

### Commit and Review Workflow

Development on the monorepo follows a policy of *committing to main* (no feature branches maintained in the remote; patches are squash-merged via PRs). The commit message format encourages a summary line prefixed with the affected component (e.g., `[Clang][Sema]`, `[LLVM][X86]`, `[MLIR][Linalg]`), followed by a body describing intent and a link to any related GitHub issue or previous Phabricator review. The project uses a pre-merge CI system (GitHub Actions combined with internal buildbot infrastructure) that runs tests across major platforms (Linux/x86-64, Linux/AArch64, macOS/arm64, Windows/MSVC) before a PR is merged. A bot (`@llvmbot`) assists with backporting commits to the active release branch by posting cherry-pick commands in response to `/cherry-pick` comments.

### Physical Layout

The top-level directories in [llvm/llvm-project](https://github.com/llvm/llvm-project) are each a discrete subproject:

```
llvm-project/
├── llvm/           # Core IR, optimizer, code generator, tools
├── clang/          # C/C++/ObjC/ObjC++ front end
├── clang-tools-extra/  # clang-tidy, clang-include-fixer, etc.
├── lld/            # LLVM linker
├── lldb/           # LLVM debugger
├── mlir/           # Multi-Level Intermediate Representation framework
├── flang/          # Fortran front end
├── polly/          # Polyhedral loop optimizer
├── openmp/         # OpenMP runtime
├── compiler-rt/    # Runtime libraries (sanitizers, profiling, builtins)
├── libcxx/         # libc++ (C++ standard library)
├── libcxxabi/      # libc++abi (C++ ABI support library)
├── libunwind/      # libunwind (stack unwinding)
├── bolt/           # Binary Optimization and Layout Tool
├── libc/           # LLVM libc (C standard library)
├── pstl/           # Parallel STL (now part of libcxx)
├── cross-project-tests/  # Tests spanning multiple subprojects
├── utils/          # Shared utilities and scripts
└── cmake/          # Shared CMake modules
```

Each subproject has its own `CMakeLists.txt`, test infrastructure, documentation, and library/binary layout. They share the `cmake/Modules/` directory for shared CMake logic and the top-level `CMakeLists.txt` that allows building any combination via `LLVM_ENABLE_PROJECTS` and `LLVM_ENABLE_RUNTIMES`.

---

## 3. Subproject Map

The table below covers every active subproject. "Key Binaries" refers to installed executables; library-only subprojects are noted accordingly.

| Name | Directory | Purpose | Key Binaries |
|------|-----------|---------|--------------|
| LLVM Core | `llvm/` | IR definition, pass manager, optimizer, code generator, target backends, TableGen, MC layer | `opt`, `llc`, `lli`, `llvm-dis`, `llvm-as`, `llvm-link`, `llvm-ar`, `llvm-nm`, `llvm-objdump`, `llvm-dwarfdump`, `llvm-config` |
| Clang | `clang/` | C, C++, Objective-C, Objective-C++ front end; driver; static analyzer | `clang`, `clang++`, `clang-22`, `clang-check`, `clang-repl`, `clang-scan-deps` |
| clang-tools-extra | `clang-tools-extra/` | Clang-based developer tools: tidy, format (lives in `clang/`), include-fixer, move, query, rename | `clang-tidy`, `clang-include-fixer`, `clang-move`, `clang-query`, `clang-doc` |
| LLD | `lld/` | LLVM's linker; ELF, COFF/PE, Mach-O, WebAssembly, XCOFF backends | `lld`, `ld.lld`, `ld64.lld`, `lld-link`, `wasm-ld` |
| LLDB | `lldb/` | LLVM debugger; Python scriptable; DWARF and MachO/ELF native debugger | `lldb`, `lldb-server`, `lldb-dap` |
| MLIR | `mlir/` | Multi-level IR framework for progressive lowering; dialect ecosystem; pattern rewriting | `mlir-opt`, `mlir-translate`, `mlir-lsp-server`, `tblgen` (dialect variant) |
| Flang | `flang/` | Fortran front end targeting LLVM IR; Fortran 2018 standard compliance; OpenMP/OpenACC support | `flang`, `flang-new` |
| Polly | `polly/` | Polyhedral loop optimizer; dependence analysis; automatic parallelization and vectorization | Loaded as LLVM pass plugin; no standalone binary |
| OpenMP | `openmp/` | OpenMP runtime libraries: `libomp` (host), `libomptarget` (offload), device plugins for CUDA/HIP/SPIRV | `libomp.so`, `libomptarget.so`, device plugins |
| compiler-rt | `compiler-rt/` | Compiler runtime builtins; AddressSanitizer, MemorySanitizer, ThreadSanitizer, UndefinedBehaviorSanitizer, libFuzzer, XRay, profile instrumentation, Shadow Call Stack | `libclang_rt.*.a`, `llvm-symbolizer` |
| libc++ | `libcxx/` | LLVM's implementation of the C++ standard library; targets C++03 through C++26 | `libc++.so`, `libc++.a` (library only) |
| libc++abi | `libcxxabi/` | C++ ABI support library: `__cxa_throw`, `__cxa_catch`, `__cxa_demangle`, `__dynamic_cast`; interfaces with libunwind | `libc++abi.so` (library only) |
| libunwind | `libunwind/` | Stack-unwinding library implementing the Itanium C++ ABI unwinding spec; used by libc++abi and compiler-rt | `libunwind.so` (library only) |
| BOLT | `bolt/` | Binary Optimization and Layout Tool; post-link optimizer using profile data to rearrange code for I-cache efficiency | `llvm-bolt`, `perf2bolt`, `llvm-boltdiff` |
| LLVM libc | `libc/` | Full implementation of the C standard library targeting freestanding and hosted environments; designed for LLVM-only toolchains | `libc.a` (library only); `libc-hdrgen` |
| PSTL | `pstl/` | Intel's Parallel STL, donated to the project; functionality absorbed into libc++ in LLVM 22; directory retained for compatibility | Absorbed into libc++; no standalone binary |

### LLVM Core (`llvm/`)

The `llvm/` subtree is the foundation everything else rests on. It defines the IR (both textual `.ll` and binary `.bc` representations), the type system, the pass manager (both the legacy pass manager and the new pass manager introduced in LLVM 5 and made default in LLVM 13), the analysis infrastructure, all target-independent optimization passes, the SelectionDAG and GlobalISel code generators, the MC layer for direct object-file emission, and the TableGen domain-specific language used to describe instruction sets throughout the codebase.

The internal directory structure of `llvm/` reflects this decomposition:

```
llvm/
├── include/llvm/       # Public C++ headers (organized by subsystem)
│   ├── IR/             # IR classes: Module, Function, Instruction, Type, ...
│   ├── Analysis/       # Analysis pass interfaces and results
│   ├── Transforms/     # Optimization pass interfaces
│   ├── CodeGen/        # Code generator abstractions
│   ├── MC/             # Machine Code layer
│   ├── Target/         # Target-independent target abstractions
│   ├── Support/        # ADT, FileSystem, MemoryBuffer, raw_ostream, ...
│   └── Config/         # Generated: llvm-config.h, abi-breaking.h
├── lib/                # Implementations (mirrors include/llvm/ layout)
│   ├── IR/             # Instruction, Value, Type, Module implementations
│   ├── Analysis/       # AA, dominators, loop info, scalar evolution, ...
│   ├── Transforms/     # InstCombine, SROA, GVN, Inliner, Vectorizers, ...
│   ├── CodeGen/        # SelectionDAG, GlobalISel, RegAlloc, PrologEpilog, ...
│   ├── MC/             # Assembler, disassembler, object file emission
│   ├── Target/         # Target-independent target code
│   ├── Support/        # Triple, APInt, APFloat, raw_ostream implementations
│   └── Linker/         # IR-level linker (for LTO and bitcode linking)
├── lib/Target/         # Per-architecture code generators
│   ├── AArch64/
│   ├── AMDGPU/
│   ├── ARM/
│   ├── Hexagon/
│   ├── NVPTX/
│   ├── RISCV/
│   ├── WebAssembly/
│   ├── X86/
│   └── ... (18 targets total in LLVM 22)
└── tools/              # Standalone tool drivers (opt, llc, lli, llvm-dis, ...)
```

The [`llvm/include/llvm/`](https://github.com/llvm/llvm-project/tree/llvmorg-22.1.0/llvm/include/llvm) directory exposes the public C++ API; [`llvm/lib/`](https://github.com/llvm/llvm-project/tree/llvmorg-22.1.0/llvm/lib) contains the implementations. [Chapter 4 — The LLVM C++ API](ch04-llvm-cpp-api.md) and [Chapter 5 — LLVM as a Library](ch05-llvm-as-a-library.md) cover this in depth.

### Clang (`clang/`)

Clang is the C/C++/Objective-C/Objective-C++ front end that translates source code to LLVM IR. It is also a driver that orchestrates the full compilation pipeline — preprocessing, compilation, assembly, and linking — as a single invocation. Clang's architecture is deliberately library-based: `libclang` exposes a stable C API used by editors and indexing tools; the clangd language server is built on Clang's AST and semantic analysis libraries. The static analyzer (`clang -analyze`) and `clang-tidy` are both built atop Clang's analysis infrastructure. Clang is the reference implementation of the C++ standard that most vendor toolchains follow for new-feature conformance.

### clang-tools-extra (`clang-tools-extra/`)

This subproject houses developer productivity tools that depend on Clang's AST but are not part of the core compiler. `clang-tidy` is a lint and transformation framework with over 500 checks; `clang-include-fixer` adds missing `#include` directives using an index; `clang-move` refactors code between files. `clang-format` is formally maintained in `clang/tools/clang-format/` rather than `clang-tools-extra/`, but it is part of the same tool ecosystem. These tools share the Clang tooling infrastructure in [`clang/include/clang/Tooling/`](https://github.com/llvm/llvm-project/tree/llvmorg-22.1.0/clang/include/clang/Tooling).

### LLD (`lld/`)

LLD is LLVM's linker, providing ELF, COFF/PE, Mach-O, WebAssembly, and XCOFF backends under a unified architecture. It is 2–10× faster than GNU `ld` on large codebases due to its single-pass design and parallelized symbol resolution. Android's NDK uses LLD as its default linker; Rust's toolchain defaults to LLD on most platforms; Chrome and Firefox use it for their release builds. The ELF backend ([`lld/ELF/`](https://github.com/llvm/llvm-project/tree/llvmorg-22.1.0/lld/ELF)) is the most mature and is the primary backend used on Linux.

LLD's architecture deliberately avoids the GNU `ld` design of multiple abstract symbol passes. Instead, it reads all input files, resolves symbols, and writes the output in a single linear pass through the inputs, with parallelism at the file-reading and relocation-application stages. This architecture makes it unsuitable for some link models that depend on archive member ordering or GNU `ld`'s repeated-pass behavior, but those use cases are rare in modern build systems. LLD's ELF backend supports ThinLTO and full LTO via the LLVM LTO library, and is the recommended linker for PGO and BOLT workflows because it can embed profile data and section ordering information directly in the output binary.

### LLDB (`lldb/`)

LLDB is LLVM's debugger, implementing the GDB/MI protocol and LLDB's own `lldb-dap` (Debug Adapter Protocol) for IDE integration. It uses LLVM's DWARF parser, LLVM's disassembler, and Clang's expression evaluator to provide a fully scriptable (Python and Lua) debugging environment. LLDB is the default debugger in Xcode. Its architecture decomposes into a core library (`lldb-core`), a remote debugging server (`lldb-server`), and a command-line front end — making it possible to embed LLDB-based debugging into custom tools.

### MLIR (`mlir/`)

MLIR — Multi-Level Intermediate Representation — is a framework for building compilers that work at multiple levels of abstraction simultaneously. Rather than forcing all programs through a single fixed IR, MLIR defines a system for creating *dialects* (domain-specific IR extensions with custom types, operations, and rewrite patterns) that can coexist in the same module and interoperate through a progressive lowering mechanism. MLIR was started at Google by Jacques Pienaar and others in 2018–2019 to address the fragmentation of IRs across TensorFlow, XLA, TPU compilers, and related projects; it was contributed to the LLVM monorepo in 2020.

The in-tree dialects as of LLVM 22 include `func`, `arith`, `memref`, `tensor`, `linalg`, `affine`, `scf` (structured control flow), `vector`, `gpu`, `llvm` (LLVM IR dialect), `cf` (unstructured control flow), `math`, `index`, `complex`, `ub` (undefined behavior), `transform`, `bufferization`, `sparse_tensor`, `shape`, `tosa`, `pdl`, `irdl`, and others — a catalogue covered in [Part XX — In-Tree Dialects](../part-20-in-tree-dialects/). Today MLIR serves as the foundation for TensorFlow/XLA, IREE, the OpenXLA project, Intel's oneAPI compilers, quantum computing toolchains, and hardware-synthesis flows. MLIR is covered in depth starting with [Part XIX — MLIR Foundations](../part-19-mlir-foundations/).

### Flang (`flang/`)

Flang is LLVM's Fortran front end. The current production Flang ("new Flang," sometimes called `flang-new`) is a complete rewrite from the original `flang` project that predates monorepo inclusion, developed primarily by NVIDIA, AMD, and Arm starting around 2018. It implements Fortran 2018 with OpenMP and OpenACC support, targets LLVM IR, and uses MLIR internally as part of its lowering pipeline. Flang reached production readiness for HPC workloads by LLVM 17–18. The binary `flang` (or `flang-new` in earlier releases) is a drop-in replacement for `gfortran` for most code.

### Polly (`polly/`)

Polly is a polyhedral loop optimizer built on LLVM's pass infrastructure. It models loop nests as polyhedra, uses integer linear programming to compute legal and profitable transformations (tiling, fusion, interchange, parallelization, vectorization), and emits transformed LLVM IR via the ISL (Integer Set Library) and CLooG or AST generation framework. Polly is built as an LLVM pass that can be loaded into `opt` or linked into Clang; it has no standalone binary. The theoretical foundations of the polyhedral model are covered in [Part XI — Polyhedral Theory](../part-11-polyhedral-theory/) and Polly's implementation in [Part XII — Polly](../part-12-polly/).

### OpenMP (`openmp/`)

The `openmp/` subproject provides the runtime libraries required to execute OpenMP-parallelized programs compiled by Clang or Flang. The host runtime, `libomp`, implements the thread team management, work-sharing constructs, and synchronization primitives specified by the OpenMP standard. `libomptarget` provides offload dispatch to accelerators; device plugins support CUDA, HIP, and SPIR-V targets. The source-language OpenMP directives are handled by Clang's front end; this subproject handles only the runtime side.

### compiler-rt (`compiler-rt/`)

`compiler-rt` is a collection of low-level runtime libraries. Its `builtins/` component provides software implementations of arithmetic operations not directly supported by a target (soft-float arithmetic, 128-bit integer operations, etc.), serving as the LLVM replacement for `libgcc_s`. The `sanitizer_common/` and per-sanitizer directories contain AddressSanitizer (ASan), MemorySanitizer (MSan), ThreadSanitizer (TSan), UndefinedBehaviorSanitizer (UBSan), LeakSanitizer (LSan), HWAddressSanitizer (HWASan), and MemTagSanitizer. `profile/` contains the instrumentation runtime for coverage and PGO. `xray/` contains the XRay function tracing runtime. All of these are covered in [Part XVI — JIT, Sanitizers, and Runtime Instrumentation](../part-16-jit-sanitizers/).

### libc++ and libc++abi (`libcxx/`, `libcxxabi/`)

`libcxx` is LLVM's implementation of the ISO C++ standard library, targeting C++03 through C++26. It is the default C++ standard library on macOS, iOS, and Android (via the NDK). `libcxxabi` implements the Itanium C++ ABI support routines — exception throwing, catching, and unwinding; dynamic casting; thread-safe static initialization; and name demangling. The two are designed as a matched pair but can be used with other ABIs. Both are covered in [Part XVII — Runtime Libraries](../part-17-runtime-libs/).

### libunwind (`libunwind/`)

`libunwind` is a small, self-contained stack-unwinding library implementing the Itanium ABI unwinding specification (`_Unwind_*` interfaces). It is used by `libc++abi` for exception propagation and by `compiler-rt` for sanitizer stack traces. It is an alternative to GNU `libunwind` (`libgcc_s`) and is the default on most LLVM-only toolchain configurations.

### BOLT (`bolt/`)

BOLT (Binary Optimization and Layout Tool) is a post-link binary optimizer. It takes an already-linked ELF binary, reads execution profile data (from `perf` or its own instrumentation), reorders functions and basic blocks to maximize instruction-cache utilization, applies hot/cold splitting, and writes back an optimized binary. BOLT typically delivers 5–15% performance improvements on large server binaries. It was developed at Meta and contributed to LLVM in 2021. The tool operates at the MC-layer level, disassembling and re-assembling individual functions without access to source code.

### LLVM libc (`libc/`)

LLVM libc is an implementation of the C standard library designed from the ground up to integrate cleanly with LLVM toolchains, support overlay (partial replacement of the system libc), and provide hermetic builds suitable for embedded and freestanding targets. Unlike `glibc`, it is written in C++17 with LLVM coding conventions, uses LLVM's sanitizer and hardening infrastructure, and is designed to be fully testable with LLVM's test infrastructure. As of LLVM 22, it provides complete POSIX coverage on Linux/x86-64 and Linux/AArch64, with ongoing work on other targets.

### PSTL (`pstl/`)

The Parallel STL subproject was Intel's original `pstl` library, donated to LLVM and integrated into `libcxx` as the parallel execution policy backend for C++17 `std::execution::par` and related algorithms. In LLVM 22, PSTL's standalone `pstl/` include directory is no longer populated as a separate headers tree; the functionality lives entirely within `libcxx`. The `pstl/` directory entry is retained in the monorepo for historical continuity. New code targeting C++17/20 parallel algorithms should use `libcxx` directly via the standard `<execution>` header.

---

## 4. Release Cadence and Versioning

### The Six-Month Major Release Cycle

LLVM follows a strict six-month major release cadence, producing two major releases per calendar year. Even-numbered releases land in the first half of the year (around May–June); odd-numbered releases land in the second half (around November). This cadence was formalized as policy with LLVM 14 (released March 2022), though nominal six-month cycles existed from approximately LLVM 9 (2019). Before LLVM 14, releases routinely slipped by weeks or months; the LLVM 14 cycle introduced scheduled branch-cut dates and a release manager process that enforces them.

### Branch Cut and RC Sequence

The release process begins with a branch cut approximately ten weeks before the planned release date. A new branch named `release/NN.x` (e.g., `release/22.x`) is created from `main`. From that point:

1. No new features are merged to the release branch — only verified bug fixes.
2. The release manager publishes a series of release candidates: `22.1.0-rc1`, `22.1.0-rc2`, and (if needed) `22.1.0-rc3`.
3. Each RC cycle is approximately two weeks, with a community call for blockers and a documented process for backporting fixes.
4. The final release tag (e.g., `llvmorg-22.1.0`) is created when the release manager and community judge the RC clean.

The release branch name uses the format `release/NN.x` rather than `release/NN.0` to accommodate point releases. The branch for LLVM 22 is [`release/22.x`](https://github.com/llvm/llvm-project/tree/release/22.x); the installed toolchain on the system used for this book corresponds to `llvmorg-22.1.3`.

### Point Releases

Point releases (e.g., `22.1.1`, `22.1.2`, `22.1.3`) carry targeted bug fixes that were either found after the final release or were not ready in time for an RC. They do not introduce new features or break ABI. Point releases are created on the same `release/NN.x` branch; the minor version (`1` in `22.1.x`) has historically been pinned to 1 in recent cycles, with the patch number incrementing.

### LLVM Version Numbering History

LLVM used a four-component version scheme (e.g., `3.9.0`) through LLVM 6, then switched to a two-component scheme (`7.0`, `8.0`) before settling on the current `major.minor.patch` scheme. The accelerating release cadence is reflected in the version numbers: LLVM went from version 3.9 (2016) to version 22 (2026) in a decade, driven by the twice-yearly cadence rather than any architectural discontinuity.

The table below places key LLVM milestones against the version numbering timeline:

| Version | Year | Notable Change |
|---------|------|---------------|
| 1.0 | Oct 2003 | First public release; `llvm-gcc` front end; x86 and SPARC backends |
| 2.6 | Oct 2009 | Clang production-ready for C/Objective-C |
| 3.0 | Dec 2011 | Clang C++ support (production quality) |
| 3.5 | Sep 2014 | First fully compliant C++11 and C++14 support in Clang |
| 3.9 | Sep 2016 | Last of the `3.x` series |
| 5.0 | Sep 2017 | New Pass Manager introduced (non-default) |
| 7.0 | Sep 2018 | Switch to two-component version numbers |
| 9.0 | Sep 2019 | Monorepo migration completed; `llvmorg-9.0.0` first monorepo tag |
| 13.0 | Oct 2021 | New Pass Manager becomes default; OrcV2 JIT API stable |
| 14.0 | Mar 2022 | Formalized six-month cadence with scheduled branch-cuts |
| 15.0 | Sep 2022 | Opaque pointers (`ptr`) become the default |
| 17.0 | Sep 2023 | Typed pointer support removed; GlobalISel default for AArch64 |
| 18.0 | Mar 2024 | `RemoveDIs` (non-intrinsic debug info) available as opt-in |
| 22.0 | 2026 | Current release series; Clang 22.1.3 used in this book |

### Backport Policy

When a significant bug or security issue is found after the release of `NN.1.0`, the fix is cherry-picked to the `release/NN.x` branch and a new point release is issued. The decision to issue a point release is made by the release manager in consultation with the community. There is no formally scheduled point-release cadence — they are issued when warranted. Typically, a major release series sees two to four point releases before the next major release supersedes it on the active-support list. Distributors (Debian, Ubuntu, Fedora, Homebrew) typically package point releases rather than the initial `.0` tag, which often contains bugs found during the RC process.

---

## 5. The Umbrella Model and CMake Integration

### Co-Versioning Under a Single Tag

All subprojects in the monorepo are versioned and tagged together. A release tag like `llvmorg-22.1.0` covers every subproject simultaneously — Clang 22.1.0, LLD 22.1.0, LLDB 22.1.0, MLIR 22.1.0, and so forth. This co-versioning eliminates the compatibility matrix problem that existed when subprojects had independent version numbers. A given Clang version is guaranteed to be compatible with the LLVM version it was built against because they share a tag and a build.

### CMake Integration: `LLVM_ENABLE_PROJECTS` and `LLVM_ENABLE_RUNTIMES`

The monorepo build system distinguishes two categories of component:

- **Projects** (`LLVM_ENABLE_PROJECTS`): subprojects that are compiled as part of the LLVM build system's main pass, sharing the CMake invocation and being built by the same compiler that was configured for the build. This includes `clang`, `lld`, `lldb`, `mlir`, `flang`, `polly`, `clang-tools-extra`, and `bolt`.

- **Runtimes** (`LLVM_ENABLE_RUNTIMES`): subprojects that are compiled with the *just-built* Clang, targeting the system ABI. These are `compiler-rt`, `libcxx`, `libcxxabi`, `libunwind`, `openmp`, and `libc`. Runtimes are built in a separate CMake pass after the compiler is ready, allowing them to be compiled with the compiler being bootstrapped.

The distinction matters for bootstrap builds: when building a stage-2 compiler, runtimes are compiled with stage-1 Clang and target the host ABI, ensuring they match the compiler they will ship alongside. This is covered in detail in [Chapter 2 — Building LLVM from Source](ch02-building-llvm-from-source.md).

### ABI Stability: Public vs. Internal APIs

LLVM makes a narrow ABI stability promise: the **C API** (`llvm-c/`) and `libclang` maintain source and binary compatibility within a major version. This allows tools like Python bindings, Rust's `llvm-sys`, and `cindex.py` to rely on a stable interface.

The **C++ API** has no ABI stability guarantee, not even within a point release. The LLVM developers document this explicitly: internal C++ headers under `llvm/include/llvm/` may change between any two commits. Downstream code that links against LLVM's C++ libraries must rebuild against each LLVM version. This is a deliberate choice — ABI stability would constrain the evolution of data structures and virtual dispatch hierarchies that are actively developed. The consequence for production deployments is that LLVM versions are typically distributed as complete, compiled toolchains rather than as shared libraries that multiple programs link against in place.

### The `LLVM_VERSION_MAJOR` Guard Pattern

Because the C++ API is unstable, downstream code that needs to support multiple LLVM versions uses the version macros from [`llvm/include/llvm/Config/llvm-config.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Config/llvm-config.h):

```cpp
#include "llvm/Config/llvm-config.h"

#if LLVM_VERSION_MAJOR >= 22
  // Use new API introduced in LLVM 22
#else
  // Fall back to older API
#endif
```

This pattern is ubiquitous in projects like Rust's `rustc_codegen_llvm`, Julia's LLVM bindings, and MLIR-based tools that want to track LLVM main while also supporting the previous release.

### The Component Model and `llvm-config`

LLVM's build system compiles the monorepo's C++ sources into a set of static or shared libraries, each corresponding to a logical subsystem. The `llvm-config` tool (installed alongside the toolchain) provides a command-line interface to query which libraries are needed for a given component set:

```bash
/usr/lib/llvm-22/bin/llvm-config --libs core analysis native
# Outputs: -lLLVMCore -lLLVMAnalysis -lLLVMAsmPrinter ...

/usr/lib/llvm-22/bin/llvm-config --cxxflags
# Outputs: -I/usr/lib/llvm-22/include -std=c++17 -fPIC ...
```

The available component names (passed to `--libs`) map to the library structure under `llvm/lib/`. A downstream project that only needs IR parsing and constant folding can link just `LLVMCore` and `LLVMSupport`; a project that needs the full x86 backend must also include `LLVMX86CodeGen`, `LLVMX86AsmParser`, and their transitive dependencies. [Chapter 5 — LLVM as a Library](ch05-llvm-as-a-library.md) covers the CMake `find_package(LLVM)` integration and the component graph in detail.

---

## 6. What LLVM Is Not

### Not a Single Compiler

The most common misconception among users encountering LLVM for the first time is treating it as a compiler in the sense that GCC is a compiler — a monolithic program that takes C source and produces object code. LLVM is instead a collection of libraries and tools. No single binary called `llvm` exists. The `clang` binary is a front end and driver that happens to use LLVM libraries; `opt` is an optimizer driver; `llc` is a standalone code generator. A production compiler built on LLVM is typically assembled from: a front end (Clang, Flang, or a custom front end), the optimizer (`PassManager` driven by `opt` or embedded in a driver), the code generator (a `TargetMachine` instantiation), and a linker (LLD or a system linker). Each of these is independently replaceable.

### Not a Language

LLVM IR is a language in the sense that it has a text syntax and a specification, but LLVM itself is not a programming language, a language runtime, or a managed execution environment. LLVM has no garbage collector, no built-in type system for high-level types, no standard library. It provides mechanisms for building these — the OrcJIT API supports JIT execution; [Chapter 16 — IR Structure](../part-04-llvm-ir/ch16-ir-structure.md) covers what the IR actually provides — but does not mandate them.

LLVM IR is also not a stable compilation target in the way that a standard bytecode format is stable. Because the C++ API has no ABI guarantee, the textual `.ll` format — while human-readable and serializable — is not intended as a long-term storage format or a stable interchange format between independently versioned tools. An `.ll` file produced by Clang 22 may not be accepted by `opt` from LLVM 17 without modification, because instruction names, intrinsic signatures, and metadata formats can change between major versions. LLVM IR is an *in-process* intermediate representation optimized for compiler construction, not a stable bytecode format for distribution. (WebAssembly, SPIR-V, and Java bytecode serve that role in their respective ecosystems.) This distinction has practical implications for any project that contemplates caching compiled IR across toolchain updates.

### Not a Runtime

LLVM produces native machine code. Compiled LLVM programs execute as ordinary native binaries; there is no LLVM interpreter running beneath them (unless the program deliberately invokes `lli`, the IR interpreter, which is a developer tool, not a production runtime). This distinguishes LLVM from JVM-based and CLR-based systems where a managed runtime owns the memory model, JIT, and garbage collection.

The OrcJIT API does provide a framework for JIT compilation — it is used by Julia, LuaJIT's LLVM backend variant, and similar systems. But OrcJIT is a *framework for building* a JIT runtime, not a runtime itself. The responsibility for memory management, calling conventions, exception handling, and garbage collection remains with the host language runtime that embeds OrcJIT. LLVM provides the machinery (compilation, relocation, code caching); the semantics of the executed program are entirely the concern of the front end and runtime that surround it.

### Comparison with GCC

GCC is a tightly coupled, monolithic architecture. Its intermediate representations (`GIMPLE` and `RTL`) are not stabilized as public APIs; the GCC plugin API exposes limited hooks but not the full IR. `libgccjit` provides a JIT interface, but it is narrow compared to LLVM's OrcJIT API. GCC's front ends (C, C++, Fortran, Ada, Go) are integrated into the build and share internal data structures in ways that make them harder to swap independently.

LLVM's library-first philosophy means that every significant component — the IR, the pass manager, the code generator, the linker, the debugger — is an independently usable library. This is the source of LLVM's extraordinary reuse: a project like Julia can use LLVM's JIT API without using Clang; a project like SPIRV-LLVM-Translator can use LLVM's IR as a translation pivot without using LLVM's code generator.

The practical consequence is that LLVM has colonized niches that GCC cannot reach: embedded domain-specific compilers, hardware synthesis toolchains, machine learning accelerator compilers, and research compiler infrastructure all build on LLVM components because the components are designed to be used that way.

### Not a Monolith Frozen by Compatibility

A subtler misconception is that LLVM's internal API is stable because it has been around since 2003. It is not. The project routinely makes breaking changes to the C++ API, relocates header files, renames classes and methods, and restructures entire subsystems. Significant architectural transitions include:

- The removal of the old `legacy::PassManager` as the default in LLVM 13 (completed work begun in LLVM 5) in favor of the new `PassManager` in `llvm/include/llvm/Passes/`.
- The transition from typed pointers (where `i32*` was a distinct type from `i8*`) to opaque pointers (`ptr` for all pointer types), completed as default behavior in LLVM 15 and finalized with the removal of typed-pointer support in LLVM 17.
- The `RemoveDIs` project, which replaces `llvm::DbgInfoIntrinsic` debug-info instructions with a non-instruction representation (`DPValue`/`DbgRecord`), significantly affecting IR traversal code and debug-info consumers. This transition was ongoing through LLVM 18–22.
- The ongoing GlobalISel project, which provides a new instruction selection framework (`llvm/lib/CodeGen/GlobalISel/`) as an alternative to SelectionDAG, with the AArch64 backend having already moved and other targets being migrated.

Code written against LLVM 18's C++ API may require non-trivial changes to build against LLVM 22. This is not a failure of the project; it reflects the sustained investment in improving internal architecture that a monolithic, ABI-frozen library cannot afford.

---

## 7. The Downstream Ecosystem

### Apple's Toolchain Fork

Apple maintains an internal fork of LLVM used to produce the Xcode toolchain shipped with macOS. The public-facing upstream of this fork is [swiftlang/llvm-project](https://github.com/swiftlang/llvm-project) (formerly `apple/llvm-project`), which carries Apple-specific patches — primarily Swift interoperability extensions, additional Objective-C and C++ ABI support, and Apple silicon-specific backend work — that have not yet been upstreamed or are specific to Apple's deployment requirements. Apple typically contributes significantly to upstream LLVM but carries a lag between the upstream main branch and the version shipped in Xcode; Xcode 16 ships a toolchain based on approximately the LLVM 18–19 era codebase.

### Android's LLVM (NDK and AOSP)

Android uses LLVM as its system compiler, linker (LLD), and debugger (LLDB) throughout the Android Open Source Project (AOSP). Google's Android team maintains a downstream branch at [android.googlesource.com/toolchain/llvm-project](https://android.googlesource.com/toolchain/llvm-project), carrying patches for Android-specific targets (ARM, AArch64, x86), the Android NDK's ABI requirements, and performance-critical changes that are upstreamed on a rolling basis. The NDK's `clang` is the recommended C/C++ compiler for Android app development; `lld` is the NDK's default linker.

### Rust's `rustc_codegen_llvm`

Rust's compiler, `rustc`, uses LLVM as its primary code generation backend via the [`compiler/rustc_codegen_llvm`](https://github.com/rust-lang/rust/tree/master/compiler/rustc_codegen_llvm) crate. This crate wraps LLVM's C++ API through a combination of the C API (`llvm-c/`) and a custom C++ shim layer (in `rustc_llvm`) that exposes the portions of the API the Rust compiler needs. The Rust project maintains a vendored, patched copy of LLVM in the `src/llvm-project` submodule, typically tracking the most recent LLVM release with additional patches for MIR-specific optimizations, Rust exception-handling semantics, and sanitizer integration. Rust also supports a `rustc_codegen_cranelift` backend (using the Cranelift code generator) and an experimental `rustc_codegen_gcc`, but LLVM remains the production backend.

### Swift

Swift's compiler, `swiftc`, uses LLVM and Clang as its backend and C interoperability layer. Swift maintains its own fork at [swiftlang/llvm-project](https://github.com/swiftlang/llvm-project), shared with Apple's toolchain work. Swift's Intermediate Language (SIL) was an acknowledged influence on MLIR's design — Jacques Pienaar's early MLIR design documents cite SIL's type system as a reference for typed SSA values. Swift's fork is notable for containing extensions to Clang's AST that support Swift's C++ interoperability layer (bidirectional Swift–C++ interop introduced in Swift 5.9).

### Zig

The Zig programming language uses LLVM as its code generation backend. Zig's compiler is written in Zig itself (self-hosted as of Zig 0.10) and invokes LLVM through the C API (`llvm-c/`). Zig does not maintain a persistent fork of LLVM — it links against a released LLVM version — but the Zig toolchain distribution bundles a compiled LLVM together with the Zig compiler binary, providing a hermetic distribution model. Zig is also notable for its use of LLVM to cross-compile to any supported target from any host without a separate cross-toolchain, leveraging LLVM's multi-target capability through its bundled `libc` headers.

### Intel, AMD, and NVIDIA Downstreams

**Intel** uses LLVM in its oneAPI DPC++ Compiler (SYCL/OpenCL/FPGA), its C/C++ compiler in oneAPI Toolkits, and the Intel Graphics Compiler (IGC) for GPU code generation. Intel contributes heavily to the LLVM SPIR-V backend and to MLIR's GPU dialect infrastructure. The [intel/llvm](https://github.com/intel/llvm) fork is the public staging area for Intel's SYCL-related patches.

**AMD** uses LLVM in its ROCm platform (the `amdgpu` backend in upstream LLVM is maintained primarily by AMD) and in its AOCC (AMD Optimizing C/C++ Compiler). AMD's AMDGPU backend is one of the most complex target backends in the LLVM tree, supporting the full gfx7 through gfx12 GPU family. AMD also contributes to MLIR's GPU and vector dialects. AMD's [ROCm/llvm-project](https://github.com/ROCm/llvm-project) fork carries ROCm-specific patches.

**NVIDIA** uses LLVM in its NVVM (CUDA compilation) toolchain, where Clang compiles CUDA device code through the `nvptx` backend, and in its MLIR-based compiler research (NVIDIA contributed Triton's early MLIR integration and is a major contributor to the LLVM GPU dialect ecosystem). NVIDIA's CUDA compiler driver (`nvcc`) optionally delegates to Clang as the host compiler.

### Qualcomm's Fork

Qualcomm maintains a downstream LLVM fork primarily for Hexagon DSP and Snapdragon CPU targets. The Hexagon architecture backend ([`llvm/lib/Target/Hexagon/`](https://github.com/llvm/llvm-project/tree/llvmorg-22.1.0/llvm/lib/Target/Hexagon)) is maintained upstream by Qualcomm engineers. Qualcomm also ships LLVM-based toolchains for Android application processors in its Snapdragon LLVM ARM Compiler, which carries performance tuning and micro-architecture-specific patches for Qualcomm's custom CPU cores.

### Other Notable Downstreams

**Julia** embeds LLVM as a JIT compiler via its `libjulia` runtime. The Julia project maintains a patched LLVM fork at [JuliaLang/llvm-project](https://github.com/JuliaLang/llvm-project) that carries patches for Julia's garbage collector stack-map support, optimizations for Julia's dynamic dispatch patterns, and fixes for LLVM bugs that affect Julia workloads. Julia's usage of LLVM is notable because it exercises the OrcJIT API more deeply than most other downstreams — Julia re-JITs functions at runtime as type specializations become available, making LLVM's JIT compilation latency a first-class performance concern.

**Kotlin/Native** uses LLVM as the backend for Kotlin code compiled to native binaries (the `kotlin-native` toolchain). JetBrains maintains a downstream fork and contributes back optimizations relevant to Kotlin's object model.

**LLPC (LLVM-Based Pipeline Compiler)** is AMD's shader compiler for Vulkan, part of the [GPUOpen](https://github.com/GPUOpen-Drivers/llpc) ecosystem. LLPC uses LLVM's AMDGPU backend to compile SPIR-V shader bytecode to AMDGPU machine code, and makes deep use of LLVM's analysis and optimization infrastructure to optimize GPU shader code.

**Emscripten and wasm-opt** use LLVM's WebAssembly backend (`llvm/lib/Target/WebAssembly/`) to compile C/C++ to WebAssembly. Emscripten ships a custom LLVM build with patches for the Wasm/Emscripten ABI and JavaScript interop.

The pattern across all these downstreams is consistent: a fork or vendored copy of a specific LLVM release, a set of patches addressing target-specific or language-specific concerns, a policy of upstreaming generic improvements, and a version lag of 1–4 major LLVM versions relative to current upstream. The six-month release cadence helps contain this lag by making update cycles predictable.

---

## 8. Chapter Summary

- LLVM began as Chris Lattner's UIUC graduate research in 2000, with the MS thesis completed December 2002 and LLVM 1.0 released October 2003. Apple's 2005 hire of Lattner and subsequent investment transformed it from research infrastructure into production toolchain.

- "LLVM" is no longer an acronym. The "Low Level Virtual Machine" name was abandoned because it misrepresents a toolkit that has no interpreter, no managed runtime, and no bytecode executor. The project FAQ declares LLVM a proper noun.

- The monorepo at [github.com/llvm/llvm-project](https://github.com/llvm/llvm-project) consolidated all subprojects from separate SVN repositories in October 2019. A single commit can now span Clang, the optimizer, MLIR, LLD, and compiler-rt atomically.

- The monorepo contains 16 active subprojects — LLVM core, Clang, clang-tools-extra, LLD, LLDB, MLIR, Flang, Polly, OpenMP, compiler-rt, libc++, libc++abi, libunwind, BOLT, LLVM libc, and PSTL (now absorbed into libc++) — plus shared infrastructure for cross-project tests and CMake modules. Each has a discrete directory, CMakeLists.txt, and purpose.

- LLVM follows a strict six-month major release cadence formalized at LLVM 14. All subprojects are co-versioned and tagged together (`llvmorg-NN.M.P`). The C++ API carries no ABI stability guarantee; only the C API and `libclang` maintain stability.

- LLVM is not a compiler, not a language, and not a runtime — it is a toolkit of independently usable compiler infrastructure libraries. This modularity is the basis for its adoption across a wider range of contexts than any previous open-source compiler infrastructure.

- Major downstreams include Apple (`swiftlang/llvm-project` for Xcode and Swift), Google (Android AOSP toolchain), Rust (`rustc_codegen_llvm`), Intel (`intel/llvm` for SYCL/oneAPI), AMD (ROCm), NVIDIA (NVVM/CUDA), Qualcomm (Hexagon/Snapdragon), and Zig (hermetic LLVM-linked distribution). Each carries patches for target-specific or language-specific requirements while upstreaming generic improvements.

- The downstream ecosystem's breadth — from mobile application processors to GPU accelerators to quantum computing toolchains — is the practical consequence of the library-first design philosophy established in LLVM 1.0 and consistently maintained across 22 major releases.
