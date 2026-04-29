# Chapter 77 — LTO and ThinLTO

*Part XIII — Link-Time and Whole-Program*

Link-time optimization (LTO) defers optimization until all translation units are combined at link time, enabling the compiler to inline across module boundaries, eliminate dead code that the linker would otherwise preserve, propagate constants through call graphs that span files, and devirtualize virtual calls whose implementations live in other translation units. LLVM provides two LTO modes: monolithic LTO (full IR merge) and ThinLTO (a scalable alternative using a summary index). This chapter covers both modes in depth, including the ThinLTO summary index format, cross-module inlining and type propagation, integration with LLD and the gold plugin, and Distributed ThinLTO for large-scale builds.

## 77.1 Monolithic LTO (Full LTO)

### 77.1.1 Overview

Monolithic LTO (also called "fat LTO" or simply "LTO") works by:

1. Compiling each translation unit to LLVM IR bitcode (`.bc`) instead of object code (`.o`).
2. At link time, merging all bitcode files into a single LLVM module.
3. Running the full optimization pipeline on the merged module.
4. Compiling the optimized merged module to a native object file.
5. Linking the native object file.

```bash
# Monolithic LTO:
clang -O2 -flto a.cpp -c -o a.o    # a.o contains LLVM bitcode
clang -O2 -flto b.cpp -c -o b.o    # b.o contains LLVM bitcode
clang -O2 -flto a.o b.o -o program # link: merge → optimize → compile → link
```

The `-flto` flag causes Clang to store LLVM IR bitcode inside the object file (in a special ELF section `.llvm.lto`) rather than native machine code. The linker (LLD or gold) detects this and invokes the LTO library.

### 77.1.2 The LTO Plugin Interface

Monolithic LTO is implemented via a **linker plugin** (`LLVMgold.so` for the gold linker, built into LLD). The plugin protocol:

1. Linker calls `plugin_claim_file_hook` for each input file.
2. Plugin claims `.bc` sections; linker gives it symbol visibility information.
3. Linker calls `plugin_all_symbols_read_hook`.
4. Plugin merges all bitcode, runs optimization, writes native objects to temp files.
5. Linker receives the native objects and completes linking normally.

The LTO pipeline executed during step 4 is essentially the same as the regular `-O2`/`-O3` pipeline, with additional IPO passes (GlobalDCE, GlobalOpt, FunctionAttrs, inliner) that benefit from the full merged module.

### 77.1.3 LTO-Specific Optimizations

With the full merged module visible, several additional passes become effective:

- **GlobalDCE**: removes global variables and functions that are dead in the linked program (even if they were externally referenced within a single TU).
- **GlobalOpt**: converts globals that are only written once to constants across the full program.
- **Dead argument elimination across modules**: function arguments that are unused by any caller (visible only after linking all callers) can be eliminated.
- **Full WPD**: Whole-Program Devirtualization (Chapter 69) can see all vtable implementations across all modules, enabling more aggressive devirtualization.
- **Full IPSCCP**: interprocedural sparse conditional constant propagation across all function boundaries.

### 77.1.4 Limitations of Monolithic LTO

Monolithic LTO has a critical scalability problem:

- **Memory**: merging all modules requires holding the entire program's IR in memory simultaneously — gigabytes for large programs.
- **Time**: the merged module is optimized as a single compilation unit; there is no parallelism. Large programs take hours.
- **Incremental builds**: any change requires re-running the entire LTO pipeline.
- **Debug info**: merging large debug info sections causes quadratic blowup.

ThinLTO was designed to address all these limitations.

## 77.2 ThinLTO

### 77.2.1 Architecture

ThinLTO (Teresa Johnson, Mehdi Amini, Xinliang David Li, CGODP 2016) achieves most of the optimization benefits of monolithic LTO while:
- Using O(1) memory per module (not O(N) for all N modules).
- Parallelizing the backend compilation step.
- Supporting incremental builds.

The key insight: most cross-module optimizations only need a **summary** of each module's contents, not the full IR. ThinLTO separates the optimization into:

1. A **summary extraction** phase (fast, parallel, per-module).
2. A **summary merge** phase (at link time, memory-efficient).
3. A **per-module optimization** phase (fast, parallel, incremental).

```bash
# ThinLTO:
clang -O2 -flto=thin a.cpp -c -o a.o
clang -O2 -flto=thin b.cpp -c -o b.o
clang -O2 -flto=thin a.o b.o -o program
```

### 77.2.2 The Module Summary Index

The **module summary index** (`ModuleSummaryIndex` in `llvm/IR/ModuleSummaryIndex.h`) is a compact, link-time-aggregatable summary of each module:

```
ModuleSummaryIndex:
  GlobalValue summary per function:
    - GUID (hash of mangled name)
    - linkage type
    - visibility
    - function flags: readnone, readonly, norecurse, notEligibleToImport
    - call graph edges: (callee GUID, call frequency, hot/cold)
    - referenced globals: GUIDs of all globals this function reads/writes
    - type tests / type checked loads (for CFI/WPD)
  GlobalVariable summary per variable:
    - linkage, visibility, type
    - whether it is read-only or write-only
```

The summary is stored in the bitcode alongside the full IR. At link time, the linker reads only the summaries (not the full IR) from all modules, merges them into a combined index, and uses it to make optimization decisions.

```bash
# Inspect the module summary:
llvm-dis --show-summary input.bc
# or:
opt -passes='print-module-summary' input.bc -o /dev/null
```

### 77.2.3 Cross-Module Inlining (Importing)

ThinLTO's main optimization is **function importing**: inlining hot callees from other modules. The process:

1. **Analysis**: the combined summary identifies call edges and their hotness (from PGO data or heuristics).
2. **Import decision**: for each function F in module M, ThinLTO decides which callee definitions to import from other modules. A callee G from module N is imported if:
   - G appears in F's call graph.
   - G is not marked `notEligibleToImport`.
   - The import would be profitable (based on size and hotness).
3. **IR fetching**: ThinLTO fetches the full IR for imported callees from the bitcode (each module's bitcode is stored separately).
4. **Per-module optimization**: module M's IR + imported callees → run the optimization pipeline → compile.

```cpp
// Import decision in ThinLTOBackend:
void computeImportForModule(
    const ModuleSummaryIndex &Index,
    const StringRef ModuleName,
    FunctionImporter::ImportMapTy &ImportList) {
  // For each function in ModuleName, walk call graph in Index:
  // Import if call edge is hot and callee is importable
}
```

The per-module optimization then sees the imported callees and can inline them, constant-fold through them, and apply all the standard IPO optimizations.

### 77.2.4 Type Propagation Across Modules

ThinLTO propagates function attributes across module boundaries using the summary:

- **`readnone`/`readonly`**: the summary records memory effect flags. If callee G is `readonly` in its own module, any module importing G can also use this attribute.
- **`norecurse`/`nofree`/`nosync`**: similarly propagated.
- **Constant argument propagation**: if all callers pass the same constant for an argument (visible in the summary's call graph edges), ThinLTO can propagate this constant into the callee after importing.

```
Summary edge: F calls G with arg0 = 42 (constant, from IPSCCP analysis)
→ ThinLTO imports G and runs constant propagation: arg0 becomes literal 42
→ SCCP eliminates dead code in G's body
```

### 77.2.5 ThinLTO and PGO

ThinLTO integrates with PGO (Chapter 67) to make hotness-aware import decisions:

- Profile data attaches `!prof !{..."branch_weights"...}` metadata, which is summarized in the index as function hotness scores.
- Hot functions (frequently called) are imported preferentially.
- Cold functions are not imported (to avoid bloating each module with rarely-used code).

```bash
# ThinLTO + PGO workflow:
clang -O2 -flto=thin -fprofile-generate a.cpp -c -o a.o
clang -O2 -flto=thin -fprofile-generate b.cpp -c -o b.o
clang -O2 -flto=thin a.o b.o -o prog_instr
./prog_instr workload
llvm-profdata merge -output=merged.profdata *.profraw
clang -O2 -flto=thin -fprofile-use=merged.profdata a.cpp -c -o a.o
clang -O2 -flto=thin -fprofile-use=merged.profdata b.cpp -c -o b.o
clang -O2 -flto=thin a.o b.o -o program
```

## 77.3 LLD and Gold Plugin Integration

### 77.3.1 LLD's Native ThinLTO

LLD (Chapter 78) has native ThinLTO support built in — it does not use the plugin interface. When LLD detects bitcode sections in input files with `-flto=thin`, it:

1. Reads module summaries from all input files.
2. Merges summaries into the combined index.
3. Runs the import analysis on the combined index.
4. Distributes per-module optimization tasks to worker threads.
5. Assembles the native object files and continues linking normally.

```
LLD ThinLTO execution:
  Thread 0: summarize → merge → analyze imports
  Thread 1: module A optimization (a.o + imported callees)
  Thread 2: module B optimization (b.o + imported callees)
  Thread N: module N optimization
  Main thread: wait for all, then link native objects
```

LLD uses `llvm::lto::ThinBackend` from `llvm/LTO/LTO.h`:

```cpp
// LLD's ThinLTO backend setup:
lto::Config Config;
Config.OptLevel = 2;
Config.UseNewPM = true;

auto Backend = lto::createInProcessThinBackend(
    llvm::heavyweight_hardware_concurrency());

lto::LTO Lto(std::move(Config), std::move(Backend));
for (auto &Input : InputFiles)
  Lto.add(*lto::InputFile::create(Input));
Lto.run(AddStream, Cache);
```

### 77.3.2 The Gold Plugin

The gold linker (GNU binutils) uses Polly's plugin interface (`LLVMgold.so`) for LTO. Gold LTO is functional but has several limitations compared to LLD:

- **Single-threaded**: gold's plugin calls are sequential; LLD parallelizes the backend.
- **No native ThinLTO**: gold's plugin interface was designed for monolithic LTO; ThinLTO support is available but requires special flags.
- **Less maintained**: the community has shifted to LLD; gold plugin receives fewer updates.

```bash
# Gold LTO (Linux):
clang -O2 -flto -fuse-ld=gold a.cpp b.cpp -o program
# LLD LTO (preferred):
clang -O2 -flto -fuse-ld=lld a.cpp b.cpp -o program
```

## 77.4 Distributed ThinLTO

### 77.4.1 The Build System Problem

For very large programs (millions of lines of code), even ThinLTO's parallelism may be insufficient — the per-module optimization jobs must all run on a single machine. **Distributed ThinLTO** extends ThinLTO to distribute the per-module optimization jobs across a build cluster.

### 77.4.2 The Two-Phase Design

Distributed ThinLTO works in two phases:

**Phase 1: Summary generation** (at compile time, parallel):

```bash
# Each module generates a thin bitcode object (summary + IR):
clang -O2 -flto=thin -c a.cpp -o a.o
clang -O2 -flto=thin -c b.cpp -o b.o
# ... N modules in parallel
```

**Phase 2: Linking (generates import lists)**:

```bash
# Link phase generates per-module import lists, but does NOT optimize yet:
lld -plugin-opt=save-temps -o /dev/null *.o
# Produces: a.o.imports, b.o.imports (import lists from summary analysis)
```

**Phase 3: Distributed per-module optimization**:

```bash
# Each module optimized independently (can run on different machines):
clang -O2 -flto=thin -x ir a.o -o a_opt.o \
    -fthinlto-index=merged.thinlto -fprofile-use=merged.profdata
clang -O2 -flto=thin -x ir b.o -o b_opt.o \
    -fthinlto-index=merged.thinlto
# ... distributed across build cluster
```

**Phase 4: Final linking**:

```bash
lld -o program a_opt.o b_opt.o
```

This design allows any build system with distributed task execution (Bazel, Buck, Icecc, distcc) to parallelize the ThinLTO optimization phase across a cluster.

### 77.4.3 Caching

ThinLTO supports an on-disk **module cache** (`-Wl,--thinlto-cache-dir=`): the hash of each module's IR + imports is computed; if it matches a cached result, the optimization is skipped and the cached object is used directly.

```bash
clang -O2 -flto=thin a.cpp b.cpp -o program \
    -Wl,--thinlto-cache-dir=/tmp/thinlto_cache \
    -Wl,--thinlto-cache-policy=cache_size_bytes:500m
```

With caching enabled, incremental builds that change only a few modules optimize only those modules (and modules that import from them), not the entire program. This makes ThinLTO practical for large-scale iterative development.

## 77.5 LTO API

### 77.5.1 The `llvm::lto::LTO` Class

The `llvm::lto::LTO` class (`llvm/LTO/LTO.h`) provides the high-level API for both monolithic and ThinLTO:

```cpp
#include "llvm/LTO/LTO.h"
#include "llvm/LTO/Config.h"

// Configure LTO:
lto::Config C;
C.OptLevel = 3;                         // optimization level
C.CGOptLevel = CodeGenOpt::Default;     // code generation level
C.RelocModel = Reloc::PIC_;            // position-independent code

// Create LTO runner:
auto Backend = lto::createInProcessThinBackend(4);  // 4 threads
lto::LTO Lto(std::move(C), std::move(Backend));

// Add input files:
for (auto &FileName : InputFiles) {
  auto Input = lto::InputFile::create(MemoryBuffer::getFile(FileName).get());
  if (auto Err = Lto.add(std::move(*Input), {}))
    // handle error
}

// Run LTO (produces optimized objects via callback):
unsigned MaxTasks = Lto.getMaxTasks();
std::vector<SmallString<0>> Buffers(MaxTasks);
auto AddStream = [&](size_t Task) -> std::unique_ptr<raw_pwrite_stream> {
  return std::make_unique<raw_svector_ostream>(Buffers[Task]);
};

if (auto Err = Lto.run(AddStream, /*Cache=*/lto::NativeObjectCache()))
  // handle error

// Buffers now contain the optimized native object code
```

### 77.5.2 The `llvm-lto` Tool

`llvm-lto` is the command-line interface to the LTO library for testing and debugging:

```bash
# Run LTO and write optimized IR:
llvm-lto -thinlto-action=optimize \
    -thinlto-module-id=a.o \
    a.o b.o -o a_optimized.bc

# Show the combined summary:
llvm-lto -thinlto-action=thinlink a.o b.o -o combined.bc
llvm-dis combined.bc -o - | grep "^summary"

# Check which functions would be imported:
llvm-lto -thinlto-action=import a.o b.o -o /dev/null \
    -print-imports 2>&1 | head -20
```

## 77.6 LTO Performance

### 77.6.1 Typical Gains

Measured improvements from LTO on production C++ codebases:

| Optimization | Effect | Typical Gain |
|-------------|--------|-------------|
| Cross-module inlining | Remove call overhead, enable further opts | 2–8% faster |
| GlobalDCE | Remove dead code across modules | 2–10% smaller |
| WPD devirtualization | Remove virtual call overhead | 0.5–3% faster |
| Constant propagation across modules | Fold dead code after propagation | 0.5–2% faster |
| Function merging (LTO + mergefunc) | Deduplicate identical cross-module functions | 1–5% smaller |

Combined effect (typical): **3–15% performance improvement** and **10–20% binary size reduction** compared to compiling without LTO.

### 77.6.2 Build Time Impact

| Mode | Memory | Compile Time | Incremental |
|------|--------|-------------|-------------|
| No LTO | O(1)/module | Fast | Fast |
| Monolithic LTO | O(N) program | Slow (single-threaded) | None |
| ThinLTO (local) | O(1)/module | Moderate (parallel) | Module cache |
| Distributed ThinLTO | O(1)/module | Fast (cluster) | Module cache |

ThinLTO with caching typically adds 10–30% to clean build times compared to non-LTO, but provides near-zero overhead for incremental builds of unchanged modules.

---

## Chapter Summary

- **Monolithic LTO** merges all LLVM IR bitcode into one module at link time, runs the full optimization pipeline, and produces a single native object; it enables all cross-module optimizations but is impractical for large programs due to memory and time requirements.
- **ThinLTO** uses a **module summary index** (compact per-module metadata: call graph edges, function attributes, type tests) to make cross-module import decisions without materializing all IR; per-module optimization jobs then run in parallel.
- **Cross-module inlining** imports callee IR from other modules based on the summary; each module's optimization pass then inlines, constant-folds, and DCEs through the imported callees.
- **LLD** has native ThinLTO support with multi-threaded per-module optimization; the **gold plugin** (`LLVMgold.so`) provides LTO for the GNU gold linker but is single-threaded and less maintained.
- **Distributed ThinLTO** separates the summary phase (compile time) from the optimization phase (link time), enabling the per-module optimization jobs to run on a build cluster; a module cache enables incremental builds.
- **The `llvm::lto::LTO` API** provides the programmatic interface for both LTO modes; `llvm-lto` is the command-line tool for testing and debugging.
- LTO typically provides 3–15% performance improvement and 10–20% binary size reduction; ThinLTO with caching adds 10–30% to clean build times with near-zero overhead for incremental builds.
