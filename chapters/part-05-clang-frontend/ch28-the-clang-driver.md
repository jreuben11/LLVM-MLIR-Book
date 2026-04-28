# Chapter 28 — The Clang Driver

*Part V: The Clang Frontend*

The Clang driver is the component that users interact with when they invoke `clang` from the command line, yet it is architecturally distinct from the compiler itself. Understanding this separation — driver versus frontend — is prerequisite to extending Clang, building custom toolchains, diagnosing build failures at the subprocess level, or embedding Clang in tools. This chapter dissects the driver from its entry point through argument parsing, phase construction, toolchain selection, job execution, and the interface through which it hands off work to the cc1 frontend.

The driver's historical role is to replicate the GCC driver interface precisely enough that build systems, IDE integrations, and Makefiles written for GCC function unchanged with Clang. This compatibility requirement explains many of the driver's design decisions: option parsing must tolerate legacy flags, the `--target=` triple must accept GCC-style vendor/OS/environment components, and the `-Bprefix` mechanism for finding tools must be honoured. Simultaneously, the driver supports Clang-specific capabilities that GCC never had: heterogeneous offload compilation, whole-archive LTO orchestration, integrated assembler short-circuiting, and in-process cc1 execution. Navigating these two requirements — backward compatibility and forward capability — is the central tension in the driver's design.

---

## 28.1 Architecture Overview: Driver vs. cc1

When a user executes `clang -O2 foo.c -o foo`, three separate processes typically run: the preprocessor+compiler (invoked as `clang -cc1`), the assembler (either the integrated assembler or an external `as`), and the linker. The `clang` binary that the user invokes is the *driver*; it orchestrates all of these subprocesses but performs none of the actual compilation work itself.

The critical distinction is between `clang` (the driver) and `clang -cc1` (the frontend). Running `clang -cc1` directly bypasses the driver entirely and speaks directly to the frontend via a different — and much larger — option table. Driver flags like `-O2` or `-march=x86-64` do not exist in the cc1 option table; instead, the driver translates them into cc1-level flags such as `-mllvm -O2` or `-target-cpu x86-64`. This translation layer is where the driver spends most of its effort.

The separation exists for several reasons. First, it allows the driver to orchestrate heterogeneous toolchains: a single `clang` invocation with `--offload-arch=sm_90` spawns both a host cc1 process and a device cc1 process targeting NVPTX. Second, it maintains backward compatibility with GCC's driver interface, which expects the compiler executable to be a driver that understands `-Wl,`, `-Wa,`, and linker selection flags. Third, it allows driver-level diagnostics (invalid flag combinations, missing files) to be handled before any expensive frontend work begins.

**Source layout.** Driver code lives entirely in `clang/lib/Driver/` and `clang/include/clang/Driver/`. The entry point for the `clang` executable is `clang/tools/driver/driver.cpp`, which calls `cc1_main()` in `clang/tools/driver/cc1_main.cpp` when the first argument is `-cc1`, or constructs a `clang::driver::Driver` object otherwise. The frontend library (`clang/lib/Frontend/`, `clang/lib/FrontendTool/`) is never touched by a driver-only invocation.

```
clang/
  tools/driver/
    driver.cpp        ← main() for the clang binary
    cc1_main.cpp      ← cc1_main(): frontend entry when first arg is -cc1
    cc1as_main.cpp    ← cc1as_main(): assembler via -cc1as
  include/clang/Driver/
    Driver.h          ← Driver class
    Compilation.h     ← Compilation, owns the JobList
    Action.h          ← Action hierarchy
    ToolChain.h       ← ToolChain base class
    Job.h             ← Command, CC1Command, JobList
    Phases.h          ← phases::ID enum
    Types.h / Types.def  ← types::ID enum and file extension table
    Multilib.h        ← Multilib, MultilibSet
  lib/Driver/
    Driver.cpp        ← Driver::BuildCompilation, BuildActions, BuildJobs
    ToolChains/       ← Linux.cpp, Darwin.cpp, MSVC.cpp, CUDA.cpp, HIP.cpp, ...
```

The `driver.cpp` entry point inspects `argv[1]`: if it equals `-cc1`, it calls `cc1_main(argv+2, argv[0], mainAddr)`, which initialises `CompilerInstance` and runs the frontend pipeline. If `argv[1]` is `-cc1as`, it calls `cc1as_main` to run the assembler frontend. Otherwise, `ExecuteCC1Tool` is not triggered and a `Driver` is constructed for the full compilation pipeline.

The in-process execution shortcut is enabled by `Driver::CC1Main`, a `function_ref<int(SmallVectorImpl<const char*>&)>` that `driver.cpp` sets to point at `ExecuteCC1Tool`. When `CC1Command::Execute` runs, it first checks whether `CC1Main` is set; if so, it calls it directly rather than spawning a subprocess. This avoids `fork/exec` overhead on every translation unit and is the reason that `clang -v -c foo.c` reports `(in-process)` before the cc1 invocation line. The in-process path also enables crash recovery: if cc1 crashes, the driver can catch signals and regenerate diagnostics by re-running the translation unit preprocessed.

---

## 28.2 The Driver Class

`clang::driver::Driver` ([`clang/include/clang/Driver/Driver.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Driver.h)) encapsulates all of the orchestration logic. Its constructor signature is:

```cpp
Driver(StringRef ClangExecutable, StringRef TargetTriple,
       DiagnosticsEngine &Diags,
       std::string Title = "clang LLVM compiler",
       IntrusiveRefCntPtr<llvm::vfs::FileSystem> VFS = nullptr);
```

The first argument is the full path to the `clang` binary, used to compute `InstalledDir` and `ResourceDir`. `TargetTriple` seeds the default target before command-line arguments are parsed. The `VFS` parameter allows tests and embedding environments to substitute an in-memory filesystem.

Key public fields (FIXME-public in the source due to historical reasons):

| Field | Type | Meaning |
|---|---|---|
| `Name` | `std::string` | Executable name as invoked (e.g., `clang++`) |
| `Dir` | `std::string` | Directory containing the driver executable |
| `ClangExecutable` | `std::string` | Full path to the clang binary |
| `ResourceDir` | `std::string` | Path to compiler-rt, include/clang, etc. |
| `SysRoot` | `std::string` | Value of `--sysroot`, if present |
| `DriverTitle` | `std::string` | String shown in `--version` |
| `CCCPrintBindings` | `unsigned:1` | Set by `-###` |
| `CCPrintOptions` | `unsigned:1` | Set by `CC_PRINT_OPTIONS` env var |

The primary API flow is:

```cpp
// 1. Create driver
DiagnosticsEngine Diags(...);
driver::Driver D("/usr/lib/llvm-22/bin/clang",
                 llvm::sys::getDefaultTargetTriple(), Diags);

// 2. Build compilation from argv
std::unique_ptr<driver::Compilation> C(D.BuildCompilation(argv));
if (!C || C->containsError()) return 1;

// 3. Execute all jobs
SmallVector<std::pair<int, const driver::Command *>, 4> FailingCmds;
int Res = D.ExecuteCompilation(*C, FailingCmds);
```

`BuildCompilation(ArrayRef<const char*>)` is the workhorse. It:
1. Calls `expandResponseFiles` to handle `@file` arguments.
2. Calls `ParseArgStrings` to produce an `InputArgList`.
3. Calls `TranslateInputArgs` to produce a `DerivedArgList` that collapses aliases.
4. Calls `getToolChain` to select the appropriate `ToolChain` for the host triple.
5. Calls `CreateOffloadingDeviceToolChains` if any offload models are requested.
6. Calls `BuildUniversalActions` → `BuildActions` → `BuildJobs` to populate the `JobList`.
7. Returns the heap-allocated `Compilation` object. A null return means no jobs were built (e.g., `--version`).

`ExecuteCompilation` iterates the `JobList`, calls `ExecuteCommand` for each `Command`, handles crash diagnostics, and returns the final exit code. The `-###` flag (`CCCPrintBindings`) causes `ExecuteCompilation` to log each command without executing it, by setting `LogOnly = true` in the `ExecuteJobs` call.

Driver-level logging is controlled by environment variables: `CC_PRINT_OPTIONS` causes every driver invocation's arguments to be logged. `CC_LOG_DIAGNOSTICS` emits frontend diagnostics in machine-readable form to a file. `CC_PRINT_PROC_STAT` enables subprocess performance reporting.

**Configuration files.** The driver loads configuration files before processing command-line arguments. It searches for `<triple>.cfg` in the toolchain's `InstalledDir/../etc/clang/` and then in the user's `~/.config/clang/` directory. A configuration file is a response file: each line contains one argument, and the driver prepends its contents to the command line. For example, a file at `/etc/clang/x86_64-linux-gnu.cfg` containing:

```
-fstack-protector-strong
-D_FORTIFY_SOURCE=2
```

would be silently prepended to every `clang` invocation targeting `x86_64-linux-gnu`. The `--config=<file>` flag overrides the automatic search. This mechanism supports distribution-level policy injection (e.g., hardening flags) without requiring every caller to pass them explicitly. Configuration files are loaded via `Driver::loadConfigFiles` → `Driver::readConfigFile`, which uses `llvm::cl::ExpansionContext::expandResponseFile` for the actual parsing.

---

## 28.3 Toolchains

A toolchain encodes the complete set of knowledge about how to compile, assemble, and link for a specific platform. `clang::driver::ToolChain` ([`clang/include/clang/Driver/ToolChain.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/ToolChain.h)) is the abstract base class; every supported platform provides a concrete subclass.

The driver selects the toolchain inside `Driver::getToolChain(const ArgList&, const Triple&)` (a private method). The selection logic in `clang/lib/Driver/Driver.cpp` switches on `Triple.getOS()` and `Triple.getEnvironment()`:

```
x86_64-unknown-linux-gnu   →  toolchains::Linux
x86_64-apple-macosx14.0    →  toolchains::Darwin (or MachO)
x86_64-pc-windows-msvc     →  toolchains::MSVC
wasm32-unknown-wasi         →  toolchains::WebAssembly
nvptx64-nvidia-cuda         →  toolchains::CudaToolChain
amdgcn-amd-amdhsa           →  toolchains::ROCMToolChain (HIP)
x86_64-pc-windows-gnu       →  toolchains::MinGW
arm-none-eabi               →  toolchains::BareMetal
```

The toolchain hierarchy mirrors the platform taxonomy in `clang/lib/Driver/ToolChains/`. Each concrete class overrides a handful of key virtual methods:

```cpp
// Build a Tool object for the assembler subprocess
virtual Tool *buildAssembler() const;

// Build a Tool object for the linker subprocess
virtual Tool *buildLinker() const;

// Select the right Tool for a given ActionClass
virtual Tool *SelectTool(const JobAction &JA) const;

// Translate driver-level DerivedArgList into cc1-level ArgStringList
virtual void addClangTargetOptions(const llvm::opt::ArgList &DriverArgs,
                                   llvm::opt::ArgStringList &CC1Args,
                                   Action::OffloadKind DeviceOffloadKind) const;

// Add system include paths to the cc1 invocation
virtual void AddClangSystemIncludeArgs(const llvm::opt::ArgList &DriverArgs,
                                       llvm::opt::ArgStringList &CC1Args) const;
```

`SelectTool` looks up the cached tool objects (`Clang`, `Assemble`, `Link`, etc.) and returns the appropriate one. The base class implementation calls `getTool(AC)`, which in turn calls `buildAssembler()` or `buildLinker()` lazily on first access. Individual toolchains may intercept specific action classes to substitute platform-specific tools; for example, the Darwin toolchain returns `tools::darwin::Linker` instead of the generic lld linker.

The toolchain also owns the sysroot. When `--sysroot=/path/to/sysroot` is passed, `Driver::SysRoot` is populated. Every toolchain subclass that builds include paths or library paths prefixes them with `D.SysRoot`. Cross-compilation is achieved by passing `--target=aarch64-linux-gnu --sysroot=/path/to/aarch64-sysroot`, which causes the driver to select the `Linux` toolchain for aarch64 and route all include/library searches through the sysroot.

**GCC installation detection.** The `Linux` toolchain constructor scans for GCC installations in common directories (`/usr/lib/gcc`, `/usr/lib/gcc-cross`, paths under the sysroot) via `Generic_GCC::GCCInstallationDetector`. It evaluates candidate installations against the target triple and selects the highest-version matching installation. In `clang -v` output this appears as:

```
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/13
Selected GCC installation: /usr/lib/gcc/x86_64-linux-gnu/13
Candidate multilib: .;@m64
Selected multilib: .;@m64
```

The selected GCC installation provides: the GCC include path (`/usr/lib/gcc/x86_64-linux-gnu/13/include`), the crt files (`crt1.o`, `crtbegin.o`, `crtend.o`), and the libgcc search path. Clang uses GCC's crt objects for startup rather than supplying its own, which is why the GCC installation matters even when compiling with Clang.

**Tool selection.** `ToolChain::SelectTool(const JobAction&)` is the central dispatch. The base class implementation calls `getTool(JA.getKind())`, which lazy-initialises and returns one of the cached tool objects:

```cpp
Tool *ToolChain::getTool(Action::ActionClass AC) const {
  switch (AC) {
  case Action::AssembleJobClass:   return getAssemble();
  case Action::IfsMergeJobClass:   return getIfsMerge();
  case Action::LinkJobClass:       return getLink();
  case Action::StaticLibJobClass:  return getStaticLibTool();
  // ...
  default:  return getClang();  // cc1 for all compile-phase actions
  }
}
```

For most job types, `getClang()` returns a `tools::Clang` object, which is the `Tool` subclass that knows how to construct a `CC1Command`. The `tools::Clang::ConstructJob` method in `clang/lib/Driver/ToolChains/Clang.cpp` is the largest function in the driver — it translates hundreds of driver-level flags into cc1-level flags and assembles the final `ArgStringList`.

**Sanitizer integration.** The `SanitizerArgs` class (`clang/include/clang/Driver/SanitizerArgs.h`) parses `-fsanitize=` flags and emits the appropriate cc1 options, runtime link libraries (`-lclang_rt.asan`), and linker flags. Toolchains call `SanitizerArgs::addArgs` from their `ConstructJob` implementations. The `XRayArgs` class similarly handles `-fxray-instrument` options.

---

## 28.4 Compilation and Jobs

`clang::driver::Compilation` ([`clang/include/clang/Driver/Compilation.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Compilation.h)) owns three collections:

- **`AllActions`**: All `Action` objects created via `MakeAction<T>(...)`. Ownership only; consumers use `Actions`.
- **`Actions`**: The root-level action list passed to `BuildJobs`.
- **`Jobs`**: The `JobList` — the final ordered list of `Command` objects to execute.

It also caches translated argument lists per `(ToolChain*, BoundArch, OffloadKind)` tuple in `TCArgs`, so that each toolchain translates arguments exactly once.

`clang::driver::JobList` is a `SmallVector<std::unique_ptr<Command>, 4>`. The elements are typically `CC1Command` objects for the compile step and plain `Command` objects for external assembler/linker invocations.

`clang::driver::Command` ([`clang/include/clang/Driver/Job.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Job.h)) holds:
- `Executable`: the path to the subprocess binary.
- `Arguments`: `llvm::opt::ArgStringList` — a `SmallVector<const char*>` of null-terminated strings.
- `InputInfoList`: description of input files.
- `OutputFilenames`: output file paths.
- `ResponseSupport`: whether and how to spill arguments to a response file.

`Command::Execute` calls `llvm::sys::ExecuteAndWait` after optionally writing a response file. The virtual override `CC1Command::Execute` short-circuits the subprocess by calling `Driver::CC1Main` (a `function_ref<int(SmallVectorImpl<const char*>&)>`) in-process when available — this is how `clang.exe` avoids the overhead of spawning a child process for the cc1 step.

```cpp
// The -### flag: print commands without executing
clang -### -c foo.c -o foo.o 2>&1
# Output: "/usr/lib/llvm-22/bin/clang" "-cc1" "-triple" ...
```

`InputInfo` wraps either a filename (`const char*`) or a linker input (`const llvm::opt::Arg*`), tagged with a `types::ID` and a `BaseInput` (the original source file that initiated the compilation pipeline).

---

## 28.5 Phases and Actions

The `phases::ID` enum ([`clang/include/clang/Driver/Phases.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Phases.h)) enumerates the logical compilation stages in dependency order:

```cpp
enum ID {
  Preprocess,   // -E: run the preprocessor, produce .i/.ii/.mi
  Precompile,   // -x c-header: produce .pch or C++20 module interface
  Compile,      // -S (frontend to LLVM IR or ASM)
  Backend,      // LLVM backend: IR → machine code
  Assemble,     // -c (without -S): run assembler
  Link,         // Link object files
  IfsMerge,     // Interface stub files merge (.ifs)
};
```

Each phase maps to an `Action` subclass. `Action` ([`clang/include/clang/Driver/Action.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Action.h)) is the node type in the compilation DAG; the driver builds a DAG from inputs to final output, then topologically sorts it into a `JobList`.

The `ActionClass` enum captures every kind of action:

| ActionClass | Corresponding subclass | Typical output type |
|---|---|---|
| `InputClass` | `InputAction` | Source file type |
| `BindArchClass` | `BindArchAction` | Same as input (fat-binary arch binding) |
| `OffloadClass` | `OffloadAction` | Host+device combination |
| `PreprocessJobClass` | `PreprocessJobAction` | `TY_PP_C` / `TY_PP_CXX` |
| `PrecompileJobClass` | `PrecompileJobAction` | `TY_PCH` |
| `ExtractAPIJobClass` | `ExtractAPIJobAction` | `TY_API_INFO` |
| `AnalyzeJobClass` | `AnalyzeJobAction` | Analyzer output |
| `CompileJobClass` | `CompileJobAction` | `TY_LLVM_IR` or `TY_Asm` |
| `BackendJobClass` | `BackendJobAction` | `TY_Asm` or `TY_LLVM_BC` |
| `AssembleJobClass` | `AssembleJobAction` | `TY_Object` |
| `LinkJobClass` | `LinkJobAction` | `TY_Image` |
| `LipoJobClass` | `LipoJobAction` | fat binary (Darwin only) |
| `DsymutilJobClass` | `DsymutilJobAction` | `.dSYM` bundle (Darwin) |
| `VerifyDebugInfoJobClass` | `VerifyDebugInfoJobAction` | (verification) |
| `OffloadBundlingJobClass` | `OffloadBundlingJobAction` | fat object |
| `OffloadUnbundlingJobClass` | `OffloadUnbundlingJobAction` | per-target objects |
| `OffloadPackagerJobClass` | `OffloadPackagerJobAction` | packaged offload binary |
| `LinkerWrapperJobClass` | `LinkerWrapperJobAction` | final linked image |
| `StaticLibJobClass` | `StaticLibJobAction` | `.a` archive |

`Driver::BuildActions` constructs the action chain for a single architecture. For a simple `clang -c foo.c` invocation, the chain is:

```
InputAction(TY_C) → PreprocessJobAction(TY_PP_C) → CompileJobAction(TY_LLVM_IR)
  → BackendJobAction(TY_Asm) → AssembleJobAction(TY_Object)
```

`Driver::BuildUniversalActions` wraps each per-architecture action set in `BindArchAction` nodes when multiple `-arch` flags are present (macOS universal binaries), then adds a `LipoJobAction` to merge them.

`Driver::ConstructPhaseAction` creates the appropriate `JobAction` for each phase given the current flags. If `-fsyntax-only` is present, the pipeline is truncated at the `Compile` phase with no output. If `-emit-llvm` is present, the `Backend` phase produces bitcode rather than assembly.

After actions are built, `Driver::BuildJobs` walks the action DAG and calls `BuildJobsForAction` recursively. This function asks the toolchain (`SelectTool`) for the `Tool` that handles each `JobAction`, then calls `Tool::ConstructJob` to produce a `Command`. The `Command` is appended to `Compilation::Jobs`.

**Action collapsing.** A critical optimisation is action collapsing. When the integrated assembler is active (the default since LLVM 3.5), the `Compile`, `Backend`, and `Assemble` phases are all handled by a single cc1 invocation. The driver implements this by checking whether adjacent actions can be handled by the same `Tool`. If `tools::Clang::ConstructJob` is selected for both the `CompileJobClass` and `AssembleJobClass`, they are collapsed into one `CC1Command` with `-emit-obj` rather than separate `-emit-llvm` and `as` invocations. This collapsing is suppressed when `-save-temps` is passed, which forces the full pipeline to materialise all intermediate files.

**Output naming.** `Driver::GetNamedOutputPath` determines the output file name for each action. For top-level actions (`-c foo.c`), the output is the explicit `-o` target or a derived name (`foo.o`). For intermediate actions, `CreateTempFile(C, Prefix, Suffix)` creates a temporary file in the system temp directory. Temporary files are registered in `Compilation::TempFiles` and cleaned up by `Compilation::CleanupFileList` at exit. With `-save-temps`, intermediate files are placed in the current working directory (`SaveTempsCwd`) or alongside the output file (`SaveTempsObj`), and are not registered for cleanup.

**The `ActionList` type.** `ActionList` is `SmallVector<Action*, 3>` defined in `clang/include/clang/Driver/Util.h`. The three-element inline buffer reflects the common case of a small number of top-level actions (one per input file). The `Compilation::AllActions` vector owns all `Action` objects (via `MakeAction<T>` which wraps them in `unique_ptr`), while `Compilation::Actions` is the externally visible list of root actions that consumers modify.

---

## 28.6 Argument Parsing and Option Tables

The driver uses LLVM's `llvm::opt` library for argument parsing. The option table is defined in TableGen at [`clang/include/clang/Driver/Options.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Options.td) and compiled into `clang/Options/Options.inc`. The generated `OptTable` subclass is accessed via `getDriverOptTable()`.

The distinction between `InputArgList` and `DerivedArgList` is important:

- `llvm::opt::InputArgList`: owns the parsed arguments exactly as they appeared on the command line, in order.
- `llvm::opt::DerivedArgList`: a view over an `InputArgList` with additional translated/synthesized arguments. `TranslateInputArgs` produces one of these from the raw `InputArgList`. Toolchains each produce their own `DerivedArgList` via `ToolChain::TranslateArgs`.

Each argument is an `llvm::opt::Arg`, which pairs an `Option` (the matched option descriptor from the table) with zero or more value strings. Querying arguments:

```cpp
// Check if -O2 is present (last wins)
if (Args.hasArg(options::OPT_O2)) { ... }

// Get the final value of -std=
const Arg *StdArg = Args.getLastArg(options::OPT_std_EQ);
if (StdArg)
  StringRef StdVal = StdArg->getValue();  // e.g., "c++20"

// Iterate all -W flags
for (const Arg *A : Args.filtered(options::OPT_W_Group)) {
  A->claim();
  // process warning flag
}
```

Option IDs are generated into the `clang::driver::options` namespace with the `OPT_` prefix. The naming follows the flag: `-O2` becomes `OPT_O2`, `-std=` becomes `OPT_std_EQ`, `-Weverything` matches `OPT_W_Group`.

**Visibility.** Not every option is visible in every driver mode. The option table uses `Visibility` bitmasks (`ClangOption`, `CC1Option`, `CLOption`, `FlangOption`, `DXCOption`, `CC1AsOption`) to restrict which options appear in which contexts. `Driver::getOptionVisibilityMask()` computes the correct visibility mask for the current driver mode, filtering out irrelevant options from the `OptTable` when parsing.

**Claiming arguments.** Unclaimed arguments produce `warn_drv_unused_argument`. Each `Arg` has a claimed/unclaimed flag. `Args.claimAllArgs()` suppresses these warnings entirely. `Arg::claim()` marks individual arguments as consumed. Tools typically call `Args.AddAllArgs(CmdArgs, options::OPT_f_Group)` to forward whole option groups, which also marks them claimed.

**-Xclang.** Driver-level invocations that need to pass cc1-only flags use `-Xclang <flag>`, which is forwarded verbatim to the cc1 argument list without driver-level interpretation:

```bash
clang -c foo.cpp -Xclang -ast-dump
# Passes -ast-dump directly to cc1; the driver does not understand it
```

The `-Xclang` mechanism is the primary escape hatch for accessing cc1-only features. For linker flags the analogous mechanism is `-Wl,<flag>`, which passes flags verbatim to the linker. `-Wa,<flag>` passes flags to the assembler. `-Xassembler <flag>` is equivalent to `-Wa,<flag>`.

**Argument translation mechanics.** `Driver::TranslateInputArgs` performs standard alias expansion and argument normalisation. For example, `-fpic` and `-fPIC` are aliases that get normalised to a canonical form. GCC compatibility aliases (`-gstabs`, `-gdwarf-2`) are translated to Clang equivalents. The `DerivedArgList` returned by `TranslateInputArgs` replaces the raw `InputArgList` for all subsequent processing; the original `InputArgList` is preserved for diagnostics and the `getInputArgs()` accessor.

**--driver-mode.** The driver mode can be forced explicitly: `--driver-mode=gcc` (default for `clang`), `--driver-mode=g++` (default for `clang++`), `--driver-mode=cpp` (preprocessor-only), `--driver-mode=cl` (MSVC-compatible), `--driver-mode=flang`, `--driver-mode=dxc`. The mode is stored in the `Driver::Mode` private enum field. It is also inferred from the executable name: an executable named `i686-linux-gnu-clang++` parses to `TargetPrefix=i686-linux-gnu`, `ModeSuffix=g++`, via `ParsedClangName`.

**LTO mode.** The `-flto`, `-flto=thin`, and `-flto=full` flags are parsed by `Driver::setLTOMode` and stored in `Driver::LTOMode`. The toolchain's linker construction checks `D.isUsingLTO()` and `D.getLTOMode()` to add the appropriate LTO linker plugin or pass the bitcode objects directly to lld. Offload LTO (`-foffload-lto`) is tracked separately in `Driver::OffloadLTOMode`.

---

## 28.7 Response Files and Argument Expansion

When the total command-line length approaches the OS limit (8192 bytes on older Windows, 128 KB on Linux), the driver spills arguments to a response file. Response files use the `@filename` syntax: the driver writes all arguments to a temporary file and replaces the argument list with a single `@/path/to/args.rsp`.

`clang::expandResponseFiles` ([`clang/include/clang/Driver/Driver.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Driver.h)) expands `@file` references before argument parsing:

```cpp
llvm::Error expandResponseFiles(SmallVectorImpl<const char *> &Args,
                                bool ClangCLMode,
                                llvm::BumpPtrAllocator &Alloc,
                                llvm::vfs::FileSystem *FS = nullptr);
```

This calls `llvm::cl::ExpandResponseFiles` internally, handling recursive expansion. The `--rsp-quoting=windows|posix` flag controls whether arguments in response files are tokenised using POSIX shell quoting or Windows `CommandLineToArgvW` rules.

The `ResponseFileSupport` struct in `Job.h` encodes per-command response file capability. Most commands support `RF_Full` (all arguments can be placed in the file). Apple's old `ld64` linker supports only `RF_FileList` (input file names in a file but flags on the command line). When the driver decides to use a response file, `Driver::setUpResponseFiles` is called before `ExecuteCompilation` to write the file and update the `Command`'s argument list.

The three `ResponseFileSupport` factory methods correspond to common scenarios:

```cpp
// Clang cc1 and most external tools
ResponseFileSupport::AtFileUTF8()   // RF_Full, UTF-8, "@" prefix

// MinGW-style tools on Windows (ANSI code page)
ResponseFileSupport::AtFileCurCP()  // RF_Full, current code page, "@" prefix

// MSVC CL.exe and LINK.exe
ResponseFileSupport::AtFileUTF16()  // RF_Full, UTF-16, "@" prefix

// Tools that cannot use response files
ResponseFileSupport::None()         // RF_None
```

The threshold at which the driver decides to use a response file is computed from the platform's `getArgMax()` limit, typically `POSIX_ARG_MAX` (131072 on Linux, much smaller on older Windows). The total argument string length is estimated, and if it exceeds roughly two thirds of the limit, a response file is used.

---

## 28.8 Input File Processing and Language Detection

The `types::ID` enum encodes the type of each input file. It is defined by the `TYPE` macro in [`clang/include/clang/Driver/Types.def`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Types.def) and expanded into `TY_` constants:

| ID | Meaning | Extension | Phases |
|---|---|---|---|
| `TY_C` | C source | `.c` | Preprocess→Compile→Backend→Assemble→Link |
| `TY_CXX` | C++ source | `.cpp`, `.cc`, `.cxx` | same |
| `TY_ObjC` | Objective-C | `.m` | same |
| `TY_ObjCXX` | Objective-C++ | `.mm` | same |
| `TY_PP_C` | Preprocessed C | `.i` | Compile→Backend→Assemble→Link |
| `TY_PP_CXX` | Preprocessed C++ | `.ii` | Compile→Backend→Assemble→Link |
| `TY_LLVM_IR` | LLVM IR text | `.ll` | Backend→Assemble→Link |
| `TY_LLVM_BC` | LLVM bitcode | `.bc` | Backend→Assemble→Link |
| `TY_Asm` | Assembly with cpp | `.S` | Preprocess→Assemble→Link |
| `TY_PP_Asm` | Assembly no cpp | `.s` | Assemble→Link |
| `TY_Object` | Object file | `.o` | Link |
| `TY_PCH` | Precompiled header | `.pch`, `.gch` | (terminal) |
| `TY_CXXModule` | C++20 module | `.cppm` | Preprocess→Precompile→Compile→… |
| `TY_CUDA` | CUDA source | `.cu` | (split host+device) |
| `TY_HIP` | HIP source | `.hip` | (split host+device) |
| `TY_HLSL` | HLSL shader | `.hlsl` | Preprocess→Compile→Backend→Assemble |

`types::lookupTypeForExtension(StringRef Ext)` performs the mapping. If the extension is not recognised, the file is treated as a linker input (`TY_Object`).

The `-x` flag overrides automatic detection: `-x c++` forces all subsequent files to be treated as C++ regardless of extension. `Driver::BuildInputs` processes the raw `InputArgList`, producing the `InputList` (pairs of `types::ID` and `Arg*`) by iterating arguments and calling `lookupTypeForExtension` on non-flag arguments, applying `-x` overrides when present.

**Phase filtering.** Not all phases are applicable to all file types. `types::getCompilationPhases(ID, LastPhase)` returns the ordered list of phases applicable to a given type, trimmed at `LastPhase`. For example, `TY_Object` has no phases before `Link` — it is immediately a linker input. `TY_LLVM_IR` starts at `Backend`. The driver uses `getFinalPhase(DerivedArgList, &FinalPhaseArg)` to determine the last phase the user requested (e.g., `-E` → `Preprocess`, `-S` → `Compile`, `-c` → `Assemble`, no flag → `Link`), then calls `types::getCompilationPhases(Ty, LastPhase)` to build the per-file action chain.

The `-std=` flag is forwarded to cc1 unchanged. The driver does not parse it for semantic content; it is the cc1 frontend that uses `-std=` to set `LangStandard`.

**Module interface units.** C++20 named module interface units (`TY_CXXModule`, extension `.cppm`) are compiled with a `Precompile` phase that produces a BMI (binary module interface), followed by a separate compile/link of the implementation. The driver handles the `--precompile` flag and `-fmodule-output=<file>` to coordinate this. The resulting `.o` file from a module interface unit is also linked into the final executable, unlike PCH files which are not linked.

---

## 28.9 Driver Modes

**GCC-compatible mode (default).** The driver behaves like GCC, accepting POSIX-style flags. `-W`, `-f`, `-m`, and `-O` flags are understood. Unrecognised flags produce `warning: unknown warning option` or `error: unknown argument`. This mode uses the `Linux`, `Darwin`, or other OS-specific toolchain.

**clang-cl (`--driver-mode=cl`).** When invoked as `clang-cl` or with `--driver-mode=cl`, the driver enters MSVC compatibility mode. It accepts `/` as a flag prefix, maps MSVC flags (`/Zi`, `/MT`, `/EHsc`, `/MD`, `/W4`) to their Clang equivalents, and uses the `MSVC` toolchain (`clang/lib/Driver/ToolChains/MSVC.cpp`). The translation is performed in `MSVC::TranslateArgs` and related functions. For example, `/Zi` becomes `-g -gcodeview`, `/O2` becomes `-O2 -fbuiltin`, `/EHsc` becomes `-fexceptions -fcxx-exceptions`.

Key behaviours in CL mode:
- Paths can use backslash separators.
- Response files use Windows quoting by default.
- Default stdlib is MSVC's headers when a Windows SDK is present.
- `/link` introduces linker flags (`/link /SUBSYSTEM:CONSOLE`).
- `/Fo<path>` sets the output object file (equivalent to `-o <path>`).
- `/Yc<header>` creates a PCH; `/Yu<header>` uses one — handled by `Driver::GetClPchPath`.
- `/Z7` and `/Zi` map to `-g -gcodeview`; `/Zd` maps to `-g -gline-tables-only`.

The `IsClangCL(StringRef DriverMode)` free function checks whether a mode string represents CL mode, and is used by `expandResponseFiles` to select the correct quoting convention.

**Flang mode (`--driver-mode=flang`).** Selects Fortran toolchain handling. The flang frontend (`flang -fc1`) replaces cc1 for Fortran source files. The driver mode is set when the executable name is `flang` or `flang-new`.

**DXC mode (`--driver-mode=dxc`).** DirectX Shader Compiler compatibility, targeting HLSL sources with DXIL output. Accepts `/T <profile>` for shader target profiles.

---

## 28.10 The cc1 Interface

The driver communicates with cc1 via a flat `ArgStringList` — a `SmallVector<const char*, N>` of null-terminated strings. The entire cc1 invocation is visible in `clang -### -c foo.c` output:

```bash
/usr/lib/llvm-22/bin/clang -### -c foo.c -o foo.o 2>&1
# Output (single cc1 command):
"/usr/lib/llvm-22/bin/clang" "-cc1" \
  "-triple" "x86_64-pc-linux-gnu"   \
  "-emit-obj"                        \
  "-main-file-name" "foo.c"          \
  "-mrelocation-model" "pic"         \
  "-pic-level" "2"                   \
  "-pic-is-pie"                      \
  "-mframe-pointer=all"              \
  "-target-cpu" "x86-64"             \
  "-resource-dir" "/usr/lib/llvm-22/lib/clang/22" \
  "-internal-isystem" "/usr/lib/llvm-22/lib/clang/22/include" \
  "-internal-externc-isystem" "/usr/include" \
  "-ferror-limit" "19"               \
  "-fgnuc-version=4.2.1"             \
  "-faddrsig"                        \
  "-o" "foo.o" "-x" "c" "foo.c"
```

Key cc1 flags that have no driver-level equivalents include:

| cc1 Flag | Meaning |
|---|---|
| `-triple <triple>` | Target triple, always present |
| `-emit-obj` | Produce a native object file |
| `-emit-llvm` | Produce LLVM IR text (`.ll`) |
| `-emit-llvm-bc` | Produce LLVM bitcode (`.bc`) |
| `-main-file-name <name>` | Source file name for debug info |
| `-include-pch <file>` | Load a precompiled header |
| `-fmodule-cache-path <dir>` | Module cache directory |
| `-mrelax-all` | Relax all instructions (integrated assembler) |
| `-resource-dir <dir>` | Compiler-rt and builtins location |
| `-internal-isystem <dir>` | System include path (not user-visible) |
| `-internal-externc-isystem <dir>` | Extern-C system include path |
| `-fgnuc-version=4.2.1` | GNU version compatibility level |
| `-faddrsig` | Emit address-significance tables |
| `-mrelocation-model <model>` | Relocation model (static/pic/ropi/rwpi) |
| `-target-cpu <cpu>` | Specific CPU for target features |
| `-tune-cpu <cpu>` | Tuning target (may differ from target-cpu) |

`cc1_main()` in `clang/tools/driver/cc1_main.cpp` receives the `ArgStringList`, constructs `CompilerInvocation` via `CompilerInvocation::CreateFromArgs`, creates a `CompilerInstance`, attaches a `DiagnosticsEngine`, and calls `ExecuteCompilerInvocation`. The compiler invocation is a serialisable snapshot of all compiler options; it is the boundary between driver-layer and frontend-layer.

Running `clang -cc1 -help` shows the full cc1 option table, which is significantly larger than the driver option table and exposes internal options not meant for direct user consumption.

**CompilerInvocation serialisation.** The cc1 interface is not merely ad hoc string passing — it is a fully serialisable form of `CompilerInvocation`. `CompilerInvocation::CreateFromArgs` parses the cc1 `ArgStringList` and populates structured option groups: `LangOptions`, `CodeGenOptions`, `TargetOptions`, `FrontendOptions`, `PreprocessorOptions`, `AnalyzerOptions`, and `HeaderSearchOptions`. The driver's job is to translate the user-facing driver flags into exactly the right cc1 argument string. A round-trip property is enforced in debug builds: `CompilerInvocation` can regenerate the cc1 arguments from its own state, and Clang verifies that parsing the regenerated arguments produces an identical `CompilerInvocation`.

`clang/include/clang/Driver/CreateInvocationFromArgs.h` exposes `createInvocation(ArrayRef<const char*>, CreateInvocationOptions)`, which runs the driver pipeline internally and directly returns a `CompilerInvocation` without executing any subprocesses. This is used by `clangd`, `libclang`, and other tools that need to analyse source files without actually compiling them.

**The -fmodule-cache-path and module building.** When explicit module maps (`-fmodule-map-file=`) or implicit modules (`-fmodules`) are enabled, cc1 uses the module cache directory specified by `-fmodule-cache-path` to store built `.pcm` files. The driver sets this from `--fmodule-cache-path=<dir>` or computes a default via `Driver::getDefaultModuleCachePath`. Module compilation is coordinated through the cc1 interface: each module is compiled as a separate cc1 invocation, and the driver does not have special knowledge of module graph structure (that is handled within cc1 by the `ModuleManager`).

---

## 28.11 Multilib and Sysroot

Multilib is the mechanism by which a single toolchain installation provides multiple ABI variants of system libraries. `clang::driver::Multilib` ([`clang/include/clang/Driver/Multilib.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Multilib.h)) represents a single variant, identified by three directory suffixes:

```cpp
class Multilib {
  std::string GCCSuffix;     // e.g., "/32" for 32-bit on x86_64
  std::string OSSuffix;      // OS library suffix
  std::string IncludeSuffix; // Include path suffix
  flags_list Flags;          // Selection flags, e.g., "-m32", "+march=arm"
  std::string ExclusiveGroup;
};
```

`MultilibSet` collects all available multilibs for a toolchain. Its `select(const Driver&, const Multilib::flags_list&, SmallVectorImpl<Multilib>&)` method evaluates the flag predicates against the current compilation's flags and returns the matching multilib variants. The `ToolChain::Multilibs` field holds this set; concrete toolchain subclasses (e.g., `toolchains::Linux`) populate it during construction by scanning the GCC installation.

Multilib YAML configuration (new in LLVM 16, carried forward) allows embedded-systems toolchains to describe their multilib layout in a file parsed by `MultilibSet::parseYaml`. This enables toolchain vendors to ship multilib metadata without patching Clang.

**Multilib selection.** `MultilibSet::select(const Driver&, const Multilib::flags_list&, SmallVectorImpl<Multilib>&)` evaluates each `Multilib` in the set by checking that all its required flags are present in the compilation's flag set (and no negated flags are present). The `FlagMatcher` list in `MultilibSet` allows flag normalisation — for example, treating both `-mfloat-abi=soft` and `-mfloat-abi=softfp` as `-mfloat-abi=soft` for the purpose of multilib selection. Multiple multilibs can be selected simultaneously if they have different `ExclusiveGroup` labels; within the same `ExclusiveGroup`, only the last matching multilib is used.

The selected multilibs affect three path sets on the `ToolChain`:
1. `LibraryPaths`: used for `-L` flags passed to the linker.
2. `FilePaths`: used for crt object file lookup.
3. Include paths: via the `IncludeDirsCallback`.

For the default x86_64 Linux case, the `Candidate multilib: .;@m64` / `Selected multilib: .;@m64` output from `-v` shows the empty-suffix default multilib was selected (`.` means no path suffix, `@m64` is the flag).

**Sysroot and cross-compilation.** The sysroot is the root of the target system's file hierarchy when cross-compiling. It is set by `--sysroot=<path>` (long form preferred) or `-isysroot <path>` (macOS legacy). Every toolchain subclass that builds include paths calls `D.SysRoot` as a prefix before system directories. Libraries are found by prepending `D.SysRoot` to `/usr/lib`, `/lib`, etc.

```bash
# Cross-compile for AArch64 Linux
clang --target=aarch64-linux-gnu \
      --sysroot=/path/to/aarch64-sysroot \
      -c foo.c -o foo.o
```

`-target` and `--target=` are synonymous in Clang 22; the double-dash form is preferred. The `-arch` flag is macOS-specific and not equivalent to `-target`.

The driver distinguishes `-isysroot` (which affects header search but not library search on Darwin) from `--sysroot` (which affects both). On Linux toolchains, both are treated as equivalent by `addLibStdCXXIncludePaths` and related helpers.

---

## 28.12 Offloading

Clang's offloading architecture handles heterogeneous computation models — CUDA, HIP, OpenMP target, SYCL — within a single driver invocation. The abstraction layer uses `OffloadAction`, `Action::OffloadKind`, and per-model toolchain selection.

`Action::OffloadKind` is a bitmask:

```cpp
enum OffloadKind {
  OFK_None   = 0x00,
  OFK_Host   = 0x01,
  OFK_Cuda   = 0x02,
  OFK_OpenMP = 0x04,
  OFK_HIP    = 0x08,
  OFK_SYCL   = 0x10,
};
```

When the driver encounters a `.cu` file with CUDA enabled, `Driver::BuildOffloadingActions` constructs:
1. A host `CompileJobAction` (produces host object file).
2. A device `CompileJobAction` per GPU architecture (produces PTX or cubin).
3. An `OffloadBundlingJobAction` that combines them into a fat object, or an `OffloadPackagerJobAction` for the new offloading model.

`--offload-arch=sm_90` (or `--cuda-gpu-arch=sm_90` in the CUDA-specific spelling) specifies the device architecture. Multiple `--offload-arch=` flags create multiple device code paths. `Driver::getOffloadArchs` collects them:

```bash
# Compile for two GPU architectures simultaneously
clang++ --offload-arch=sm_90 --offload-arch=sm_80 foo.cu -o foo
```

The new (LLVM 14+) offloading model uses `clang-offload-packager` to embed device images and `clang-offload-linker` (the linker wrapper) to extract and link them. The old model used `clang-offload-bundler` to create fat binaries. Both are still supported in Clang 22; `--offload-new-driver` selects the new model.

`Driver::CreateOffloadingDeviceToolChains` is called from `BuildCompilation` to instantiate device toolchains. For CUDA, this instantiates `toolchains::CudaToolChain`; for HIP, `toolchains::HIPAMDToolChain` or `toolchains::HIPSPVToolChain`. These toolchains override `SelectTool` to provide device-specific compilation tools.

`OffloadAction` encodes either a host dependence, a set of device dependences, or both. The `OffloadAction::DeviceDependences` inner class carries parallel arrays of actions, toolchains, bound architectures, and offload kinds.

**The CUID mechanism.** For CUDA and HIP, each compilation unit needs a unique identifier (CUID) to ensure that device-side symbols from different translation units do not collide. `CUIDOptions` (defined in `Driver.h`) manages this. The `--cuid=<value>` flag forces a specific CUID; by default, a hash of the input filename is used (`Kind::Hash`). The CUID is passed to cc1 via `-cuid=<value>` and embedded in the fat binary.

**OpenMP target offloading.** With `-fopenmp --offload-arch=nvptx64`, the driver generates separate device compilation jobs for OpenMP target regions. The OpenMP runtime kind is selected by `Driver::getOpenMPRuntime`, which returns `OMPRT_OMP` (libomp), `OMPRT_GOMP` (GNU OpenMP), or `OMPRT_IOMP5` (Intel OpenMP). The new offloading model (enabled by default in recent Clang) uses `LinkerWrapperJobAction` to extract device images from the fat binary at link time and invoke device-specific linkers.

**SYCL.** The `OFK_SYCL` offload kind handles SYCL sources (extension `.sycl`). The `SyclInstallationDetector` class (`clang/include/clang/Driver/SyclInstallationDetector.h`) probes for the Intel oneAPI DPC++ toolchain installation and configures the SYCL device toolchain accordingly.

---

## 28.13 Driver Diagnostics

The `DiagnosticsEngine` is constructed *before* the `Driver`, because the driver constructor may immediately need to emit diagnostics about the toolchain or flags:

```cpp
IntrusiveRefCntPtr<DiagnosticIDs> DiagIDs(new DiagnosticIDs());
IntrusiveRefCntPtr<DiagnosticOptions> DiagOpts(new DiagnosticOptions());
DiagnosticsEngine Diags(DiagIDs, DiagOpts, new TextDiagnosticPrinter(...));
driver::Driver D(path, triple, Diags);
```

Driver-level diagnostic IDs live in `clang/Basic/DiagnosticDriverKinds.inc` (generated from `clang/include/clang/Basic/DiagnosticDriver.td`). The `err_drv_` prefix distinguishes driver errors from frontend errors:

| Diagnostic ID | Meaning |
|---|---|
| `err_drv_unknown_argument` | Unrecognised flag |
| `err_drv_argument_not_allowed_with` | Conflicting flags |
| `err_drv_argument_only_allowed_with` | Flag requires another flag |
| `err_drv_no_such_file_or_directory` | Input file not found |
| `err_drv_command_failed` | Subprocess returned non-zero |
| `err_drv_command_signalled` | Subprocess killed by signal |
| `err_drv_expand_response_file` | Failed to read `@file` |
| `warn_drv_unused_argument` | Argument with no effect for this phase |

`Driver::Diag(unsigned DiagID)` is a forwarding wrapper that returns `DiagnosticsEngine::Report(DiagID)`, allowing the driver to use the builder pattern:

```cpp
D.Diag(diag::err_drv_argument_not_allowed_with) << "-mfloat-abi=hard" << "-mfloat-abi=soft";
```

The cc1 frontend's `DiagnosticsEngine` is separate and created anew inside `cc1_main` from the `-W` flags that the driver forwarded. Driver diagnostics and frontend diagnostics share the same `DiagnosticIDs` table but different consumer objects.

**Crash diagnostics and reproducers.** When a cc1 subprocess crashes (signal delivery or abnormal exit), the driver calls `generateCompilationDiagnostics`. This re-runs the failing command with `-frewrite-includes` to produce a self-contained preprocessed source, collects the original command line, and writes them to a temporary directory alongside a shell script that can reproduce the crash. The `FORCE_CLANG_DIAGNOSTICS_CRASH` environment variable triggers this path unconditionally for testing. The `--gen-reproducer` flag controls the `ReproLevel` (Off, OnCrash, OnError, Always), which determines how aggressively crash reports are generated.

**Print actions.** `Driver::PrintActions` (triggered by `-ccc-print-phases`) dumps the action DAG in a human-readable format that shows the phase chain, offload kinds, and output types. This is useful for understanding how the driver interprets a complex command line with multiple `-arch` values, `-x` overrides, or offloading flags:

```bash
clang -ccc-print-phases -c foo.c
# 0: input, "foo.c", c
# 1: preprocessor, {0}, cpp-output
# 2: compiler, {1}, ir
# 3: backend, {2}, assembler
# 4: assembler, {3}, object
```

With `--offload-arch=sm_90 --offload-arch=sm_80 foo.cu`, the phase dump reveals the host/device split and the bundling step.

---

## 28.14 Extending the Driver

**Adding a new toolchain.** Subclass `ToolChain` in a new file under `clang/lib/Driver/ToolChains/`. Implement the minimal set of virtual methods:

```cpp
class MyPlatform : public ToolChain {
public:
  MyPlatform(const Driver &D, const llvm::Triple &Triple,
             const llvm::opt::ArgList &Args);

  bool IsIntegratedAssemblerDefault() const override { return true; }
  bool isPICDefault() const override { return false; }
  bool isPIEDefault(const llvm::opt::ArgList &Args) const override {
    return false;
  }
  bool isPICDefaultForced() const override { return false; }

protected:
  Tool *buildLinker() const override;
  Tool *buildAssembler() const override;

public:
  void AddClangSystemIncludeArgs(const llvm::opt::ArgList &DriverArgs,
                                 llvm::opt::ArgStringList &CC1Args) const override;

  void addClangTargetOptions(const llvm::opt::ArgList &DriverArgs,
                             llvm::opt::ArgStringList &CC1Args,
                             Action::OffloadKind DeviceOffloadKind) const override;
};
```

Register the new toolchain in the `Driver::getToolChain` switch in `clang/lib/Driver/Driver.cpp` by matching on `Triple.getOS()` or `Triple.getEnvironment()`. Also register in `getOffloadToolChain` if offloading support is needed.

**Adding a new driver flag.** Flags are defined in `clang/include/clang/Driver/Options.td` using TableGen. A minimal new flag looks like:

```tablegen
def my_flag : Flag<["-"], "my-flag">,
  Visibility<[ClangOption, CC1Option]>,
  HelpText<"Enable my feature">,
  Group<f_Group>;
```

After running TableGen (`cmake --build . --target ClangDriverOptions`), the new option is available as `options::OPT_my_flag`. Consume it in the relevant toolchain's `ConstructJob` or `addClangTargetOptions`:

```cpp
if (Args.hasArg(options::OPT_my_flag)) {
  CmdArgs.push_back("-my-cc1-flag");
}
```

**The `Tool` abstract class.** `clang::driver::Tool` ([`clang/include/clang/Driver/Tool.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Tool.h)) represents a program that can execute one compilation step. Each `Tool` subclass implements `ConstructJob`, which is responsible for adding arguments to the `Command`'s argument list and calling `C.addCommand(std::make_unique<Command>(...))`:

```cpp
class Tool {
public:
  // Human-readable name for diagnostics ("clang", "GNU assembler", etc.)
  const char *getShortName() const;

  // Can this tool process C-family preprocessing internally?
  virtual bool hasIntegratedCPP() const = 0;

  // Does the tool produce good error messages (suppresses extra driver diag)?
  virtual bool hasGoodDiagnostics() const { return false; }

  // Build the Command(s) for the given JobAction and add to C.Jobs
  virtual void ConstructJob(Compilation &C, const JobAction &JA,
                            const InputInfo &Output,
                            const InputInfoList &Inputs,
                            const llvm::opt::ArgList &TCArgs,
                            const char *LinkingOutput) const = 0;
};
```

The `tools::Clang` subclass (`clang/lib/Driver/ToolChains/Clang.cpp`) is the workhorse: its `ConstructJob` implementation spans several thousand lines and handles every possible combination of flags that the driver might need to forward to cc1. Key responsibilities include: selecting the output type (`-emit-obj`, `-emit-llvm`, `-emit-llvm-bc`, `-S`), forwarding sanitizer options, setting up debug info flags (`-debug-info-kind=`, `-dwarf-version=`), configuring target features (`-target-feature`, `-target-abi`), and handling profile instrumentation (`-fprofile-instr-generate`, `-fprofile-use=`).

**Programmatic driver invocation.** Embedding the driver in a tool (such as a build system or language server):

```cpp
#include "clang/Driver/Driver.h"
#include "clang/Driver/Compilation.h"
#include "clang/Basic/DiagnosticOptions.h"
#include "llvm/Support/raw_ostream.h"

using namespace clang;
using namespace clang::driver;

int compileFile(llvm::StringRef SourceFile) {
  IntrusiveRefCntPtr<DiagnosticIDs>  DiagIDs(new DiagnosticIDs());
  IntrusiveRefCntPtr<DiagnosticOptions> DiagOpts(new DiagnosticOptions());
  DiagnosticsEngine Diags(DiagIDs, DiagOpts,
                          new TextDiagnosticPrinter(llvm::errs(), DiagOpts.get()));

  Driver D("/usr/lib/llvm-22/bin/clang",
           llvm::sys::getDefaultTargetTriple(), Diags);
  D.setTitle("my-tool");

  SmallVector<const char *, 8> Args = {
    "clang", "-O2", "-c", SourceFile.data(), "-o", "out.o"
  };

  std::unique_ptr<Compilation> C(D.BuildCompilation(Args));
  if (!C || C->containsError()) return 1;

  SmallVector<std::pair<int, const Command *>, 4> Failing;
  return D.ExecuteCompilation(*C, Failing);
}
```

---

## Chapter Summary

- The `clang` binary is a driver that orchestrates subprocesses; `clang -cc1` bypasses the driver and invokes the frontend directly.
- `clang::driver::Driver::BuildCompilation` is the central method: it parses arguments, selects toolchains, builds the action DAG, and translates actions into jobs.
- `ToolChain` subclasses encode all platform-specific knowledge: include paths, library directories, system assembler and linker selection, and translation of driver flags to cc1 flags.
- The `phases::ID` enum and `Action` hierarchy model the compilation DAG; `BuildActions` builds it, `BuildJobs` lowers it to `Command` objects.
- `llvm::opt::InputArgList` and `DerivedArgList` manage argument ownership and toolchain-specific translations; option IDs come from the TableGen-generated `Options.td`.
- Response files (`@file`) and argument expansion handle command-line length limits; `ResponseFileSupport` encodes per-tool capability.
- Driver modes (`GCCMode`, `CLMode`, `FlangMode`, `DXCMode`) alter argument parsing, toolchain selection, and flag translation semantics.
- `CC1Command::Execute` runs cc1 in-process via `CC1Main` function pointer when available, avoiding process creation overhead.
- Multilib selects library directory variants; `--sysroot` redirects all system path lookups for cross-compilation.
- Offloading splits compilation into host and device action sub-graphs, with `OffloadAction` encoding the host/device relationships.
- Driver diagnostics use `err_drv_`-prefixed IDs from `DiagnosticDriverKinds.inc`; the `DiagnosticsEngine` is constructed before the `Driver`.

---

## Cross-References

- [Chapter 29 — SourceManager and File System Abstraction](ch29-source-manager.md): the `SourceManager` constructed inside cc1, which the driver never directly touches.
- [Chapter 30 — The DiagnosticsEngine](ch30-diagnostics-engine.md): details of the `DiagnosticsEngine` that the driver constructs and passes to cc1.
- [Chapter 31 — The Lexer and Preprocessor](ch31-lexer-preprocessor.md): the first stage of the cc1 pipeline that the driver hands off to.
- [Chapter 39 — CodeGenModule and the IR Emitter](ch39-codegenmodule.md): the backend of cc1 that the driver triggers via `-emit-obj` or `-emit-llvm`.
- [Chapter 41 — ABI Boundaries and Calling Conventions](ch41-abi-boundary.md): how the driver's `-mabi=` and `-mfloat-abi=` flags affect cc1-level ABI selection.

## Reference Links

- [`clang/include/clang/Driver/Driver.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Driver.h)
- [`clang/include/clang/Driver/ToolChain.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/ToolChain.h)
- [`clang/include/clang/Driver/Compilation.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Compilation.h)
- [`clang/include/clang/Driver/Job.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Job.h)
- [`clang/include/clang/Driver/Action.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Action.h)
- [`clang/include/clang/Driver/Phases.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Phases.h)
- [`clang/include/clang/Driver/Types.def`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Types.def)
- [`clang/include/clang/Driver/Multilib.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Multilib.h)
- [`clang/tools/driver/driver.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/tools/driver/driver.cpp)
- [`clang/tools/driver/cc1_main.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/tools/driver/cc1_main.cpp)
- [`clang/lib/Driver/Driver.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Driver/Driver.cpp)
- [`clang/include/clang/Driver/Options.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Options.td)
- [`clang/lib/Driver/ToolChains/Linux.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Driver/ToolChains/Linux.cpp)
- [`clang/lib/Driver/ToolChains/MSVC.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Driver/ToolChains/MSVC.cpp)
- [`clang/lib/Driver/ToolChains/CUDA.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Driver/ToolChains/CUDA.cpp)
