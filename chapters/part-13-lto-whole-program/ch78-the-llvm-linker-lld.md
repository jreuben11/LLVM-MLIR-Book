# Chapter 78 — The LLVM Linker (LLD)

*Part XIII — Link-Time and Whole-Program*

LLD is LLVM's production linker, supporting ELF (Linux, BSD, embedded), COFF (Windows), Mach-O (macOS/iOS), and WebAssembly object formats. It was written from scratch to overcome the scalability and maintainability limitations of GNU ld and gold. LLD is the default linker for Android, Chrome, Fuchsia, and many other large software projects. This chapter covers LLD's architecture, the per-format driver design, symbol resolution, relocation and relaxation, linker scripts, ICF (identical code folding), and partial linking.

## Table of Contents

- [78.1 Architecture Overview](#781-architecture-overview)
  - [78.1.1 Format-Specific Drivers](#7811-format-specific-drivers)
  - [78.1.2 Single-Pass Execution Model](#7812-single-pass-execution-model)
- [78.2 Symbol Resolution](#782-symbol-resolution)
  - [78.2.1 The Symbol Table](#7821-the-symbol-table)
  - [78.2.2 Archive Handling](#7822-archive-handling)
  - [78.2.3 Version Scripts](#7823-version-scripts)
  - [78.2.4 Symbol Wrapping](#7824-symbol-wrapping)
- [78.3 Relocations and Relaxations](#783-relocations-and-relaxations)
  - [78.3.1 Relocation Types](#7831-relocation-types)
  - [78.3.2 Relaxations](#7832-relaxations)
  - [78.3.3 RISC-V Relaxations](#7833-risc-v-relaxations)
- [78.4 Section Layout and Output](#784-section-layout-and-output)
  - [78.4.1 Output Section Assignment](#7841-output-section-assignment)
  - [78.4.2 PHDR and Segment Layout](#7842-phdr-and-segment-layout)
  - [78.4.3 COMDAT Section Deduplication](#7843-comdat-section-deduplication)
- [78.5 Identical Code Folding (ICF)](#785-identical-code-folding-icf)
  - [78.5.1 Overview](#7851-overview)
  - [78.5.2 Algorithm](#7852-algorithm)
  - [78.5.3 Safe vs. All ICF](#7853-safe-vs-all-icf)
- [78.6 Linker Scripts](#786-linker-scripts)
  - [78.6.1 Overview](#7861-overview)
  - [78.6.2 INSERT BEFORE/AFTER](#7862-insert-beforeafter)
- [78.7 Partial Linking](#787-partial-linking)
- [78.8 LLD Performance](#788-lld-performance)
- [Profile-Guided Function Ordering in LLD](#profile-guided-function-ordering-in-lld)
  - [The Function Placement Problem](#the-function-placement-problem)
  - [Input Format: Symbol Ordering Files](#input-format-symbol-ordering-files)
  - [Call-Graph Profile Sort](#call-graph-profile-sort)
  - [Section Layout: hot, warm, cold](#section-layout-hot-warm-cold)
  - [Comparison with BOLT](#comparison-with-bolt)
  - [Comparison with PGO Mid-End Reordering](#comparison-with-pgo-mid-end-reordering)
  - [Startup Improvement Measurements](#startup-improvement-measurements)
  - [Integration with ThinLTO + PGO](#integration-with-thinlto--pgo)
  - [macOS-Specific LLD Behaviors](#macos-specific-lld-behaviors)
- [Chapter Summary](#chapter-summary)

---

## 78.1 Architecture Overview

### 78.1.1 Format-Specific Drivers

LLD has four independent drivers sharing a common utility library:

```
lld/
├── Common/      — shared infrastructure (memory, error handling, args)
├── ELF/         — ELF linker (ld.lld, used on Linux/BSD/embedded)
├── COFF/        — COFF linker (lld-link, used on Windows)
├── MachO/       — Mach-O linker (ld64.lld, used on macOS/iOS)
└── Wasm/        — WebAssembly linker (wasm-ld, used for WASM targets)
```

Each driver is a complete, standalone linker that shares no state with the others. The common infrastructure provides: argument parsing, archive member extraction, a concurrent output writer, and the LTO integration layer.

```bash
# Invoke LLD as ELF linker:
clang -fuse-ld=lld -o program a.o b.o    # uses ld.lld

# Invoke directly:
ld.lld a.o b.o -o program
lld-link a.obj b.obj /out:program.exe
```

### 78.1.2 Single-Pass Execution Model

Unlike GNU ld (which uses multiple passes over the input files), LLD processes each input file once in a **single pass**. This is achieved by:

1. **Lazy archive member inclusion**: archive members are added to the link only when their exported symbols are referenced; this is tracked via a `SymbolTable` that records undefined symbols.
2. **Graph-based symbol resolution**: LLD builds a complete symbol graph before generating any output.
3. **Parallel section hashing** (for ICF) and **parallel relocation application**.

The single-pass model dramatically reduces LLD's link time compared to GNU ld — LLD is typically 2–5× faster than GNU ld on large programs.

## 78.2 Symbol Resolution

### 78.2.1 The Symbol Table

LLD's `SymbolTable` is a hash map from symbol name to `Symbol` object. Each `Symbol` has a current **binding** (undefined, common, defined, lazy/archive, shared/DSO) and a **priority** that determines which definition wins:

```
Resolution order (highest to lowest priority):
1. Defined (strong) — from object files or link-time synthetic symbols
2. Common (tentative) — from `.comm` sections (C `int x;` at file scope)
3. Shared (from DSO) — from dynamic libraries
4. Lazy (from archives) — pulled in only if needed
5. Undefined — referenced but not yet defined
```

**Strong symbols** override **weak symbols** (declared with `__attribute__((weak))` or in C++ linkonce_odr). Multiple strong definitions of the same symbol is an error ("multiple definition").

### 78.2.2 Archive Handling

Archives (`.a` files) use **lazy extraction**: LLD reads the archive's symbol table (the `__.SYMDEF` or `ar` index), registering all exported symbols as `LazyArchive`. When an undefined symbol is resolved to a `LazyArchive`, the archive member is extracted and added to the link:

```cpp
// ArchiveFile.cpp (ELF driver):
void ArchiveFile::extract(const Archive::Symbol &Sym) {
  auto Member = Sym.getMember();
  // Parse the object file from the archive:
  InputFile *File = createObjectFile(Member.getBuffer());
  Driver->addFile(File);
}
```

### 78.2.3 Version Scripts

ELF version scripts control symbol visibility in shared libraries:

```
# Version script (version.script):
{
  global:
    exported_function;
    exported_class*;   # glob pattern
  local:
    *;                 # everything else is local (hidden)
};
```

LLD processes version scripts during symbol resolution: symbols not matching a `global:` pattern are marked `STB_LOCAL` in the ELF symbol table, preventing them from being visible to DSO consumers.

```bash
ld.lld -shared -version-script=version.script -o libfoo.so a.o b.o
```

### 78.2.4 Symbol Wrapping

`--wrap=symbol` replaces all references to `symbol` with `__wrap_symbol`, and references to `__real_symbol` with the original `symbol`. This is used to interpose functions at link time without modifying source:

```bash
ld.lld --wrap=malloc a.o my_malloc_interposer.o -o program
# References to malloc → __wrap_malloc
# __real_malloc inside interposer → original malloc
```

## 78.3 Relocations and Relaxations

### 78.3.1 Relocation Types

A **relocation** is an instruction to the linker: "at this offset in this section, patch the value to be the address of this symbol plus this addend." ELF defines dozens of relocation types per architecture.

```
ELF relocation (Elf64_Rela):
  r_offset: byte offset within section
  r_info:   (symbol index << 32) | relocation type
  r_addend: addend value
```

Common x86-64 relocation types:

| Type | Symbol | Use |
|------|--------|-----|
| `R_X86_64_64` | absolute 64-bit | data references |
| `R_X86_64_PC32` | PC-relative 32-bit | direct function calls |
| `R_X86_64_GOT32` | GOT-relative 32-bit | global variable access via GOT |
| `R_X86_64_PLT32` | PLT-relative | dynamic function calls via PLT |
| `R_X86_64_GOTPCREL` | GOT pointer PC-relative | load GOT entry address |
| `R_X86_64_TLSGD` | TLS global dynamic | general TLS access |
| `R_X86_64_TPOFF32` | TLS initial exec | thread-local access via TP |

LLD processes relocations in `Relocations.cpp`, computing the final values and writing them into the output section.

### 78.3.2 Relaxations

**Relocation relaxation** rewrites code sequences to use more efficient addressing when the linker knows the exact addresses. Common relaxations:

**TLS relaxation** (Chapter 79) is the most impactful: the general-dynamic TLS sequence (which requires two function calls) is relaxed to the initial-exec sequence (one instruction) when the symbol is in the same executable:

```asm
; Before (GD TLS, R_X86_64_TLSGD):
leaq  var@tlsgd(%rip), %rdi
call  __tls_get_addr@plt
; movq  (%rax), %rax   (separate access)

; After relaxation (LE TLS, using TP directly):
movq  %fs:0, %rax
movl  var@tpoff(%rip), %ecx
; (offset known at link time: direct TP-relative access)
```

**Call relaxation**: a `CALL rel32` instruction has a 32-bit range (±2 GB). If the target is within 32-bit range, no relaxation is needed. For ARM and AArch64, branch range is limited (±128 MB for BL), and LLD inserts **thunks** (trampolines) when the target is out of range:

```asm
; AArch64 range extension thunk:
thunk_far_function:
  adrp x16, far_function@page
  add  x16, x16, far_function@pageoff
  br   x16
```

### 78.3.3 RISC-V Relaxations

RISC-V has particularly aggressive relaxations because its instruction set uses two-instruction sequences for many addressing modes. LLD's RISC-V backend applies:

- **AUIPC+JALR → JAL**: when the callee is within ±1 MB, replace the two-instruction indirect-call sequence with a single JAL instruction.
- **AUIPC+ADDI → NOP+ADDI** (for GP-relative references): when the symbol is within ±2 KB of the global pointer, use a GP-relative load instead of the PC-relative two-instruction sequence.
- **LUI+ADDI → ADDI** (for small immediate values): when the value fits in 12 bits, the upper 20-bit lui instruction is unnecessary.

RISC-V relaxation requires an iterative algorithm (relaxation shrinks code, which can bring previously-out-of-range targets into range, enabling further relaxation), so LLD repeats the relaxation pass until convergence.

## 78.4 Section Layout and Output

### 78.4.1 Output Section Assignment

LLD assigns each input section to an output section based on:
1. Section name (`--section-start`, `-T` linker script, or default rules).
2. Section type (PROGBITS, NOBITS, NOTE, etc.).
3. Section flags (SHF_ALLOC, SHF_EXECINSTR, SHF_WRITE).

Default ELF section layout:

```
Segment PT_LOAD (r-x):   .text, .rodata, .eh_frame
Segment PT_LOAD (rw-):   .data, .bss, .tbss
Segment PT_DYNAMIC:      .dynamic
Segment PT_GNU_STACK:    (stack executable flag)
Segment PT_GNU_RELRO:    (read-only after relocation sections)
```

### 78.4.2 PHDR and Segment Layout

LLD constructs program headers (segments) from output sections. The `-z relro` flag marks certain sections as read-only after dynamic linking (RELRO):

```
RELRO sections (mapped read-only after ld.so relocations):
  .init_array, .fini_array, .jcr, .dynamic, .got (partial)
```

Non-RELRO `.got.plt` (which contains PLT-updated GOT entries) must remain writable during lazy binding.

### 78.4.3 COMDAT Section Deduplication

C++ templates and inline functions can appear in multiple translation units. These are placed in **COMDAT sections** (ELF group sections): at most one copy is retained at link time. LLD deduplicates COMDAT sections by group signature, keeping the first occurrence and discarding the rest:

```cpp
// ELF COMDAT deduplication:
bool shouldReplace(const ComdatGroup &Existing, const ComdatGroup &Incoming) {
  // Keep the first one seen (or apply --icf for more aggressive dedup)
  return false;  // always keep existing
}
```

## 78.5 Identical Code Folding (ICF)

### 78.5.1 Overview

**Identical Code Folding** (`--icf=all`) merges functions with identical machine code (after relocation) into a single copy. This is particularly effective for C++ template instantiations, where many different type instantiations produce identical code:

```cpp
// Two template instantiations with identical code:
int add_int(int a, int b)   { return a + b; }   // add i32
int add_short(short a, short b) { return a + b; } // potentially same asm
```

After ICF, one function pointer replaces both. All references to the eliminated function are updated to point to the surviving copy.

### 78.5.2 Algorithm

ICF uses a **hash-based equivalence** algorithm with iterative refinement:

1. **Initial hashing**: hash each function's code and relocation pattern (symbol + addend pairs). Functions with identical hashes are candidates for merging.

2. **Relocation equivalence**: two functions are equivalent if their relocations point to equivalent targets. This is circular (equivalence depends on equivalence), so ICF uses an iterative approach:
   - Start: two functions are equivalent if their code bytes are identical.
   - Refine: two functions are equivalent if their code bytes match AND all their relocations point to equivalent functions.
   - Iterate until no new equivalences are found.

3. **Merging**: for each equivalence class, pick one representative and redirect all references from the others to it.

```bash
# Enable ICF (all modes):
ld.lld --icf=all program.o -o program

# Safe ICF only (avoids breaking address comparisons):
ld.lld --icf=safe program.o -o program
```

ICF typically reduces binary size by 2–10% for C++ programs with heavy template use.

### 78.5.3 Safe vs. All ICF

`--icf=all` may break code that compares function pointers (`func_ptr == &function`), because two function pointers that were distinct before ICF may now be equal. `--icf=safe` avoids merging functions whose address is taken in non-call contexts, preserving pointer-comparison semantics.

## 78.6 Linker Scripts

### 78.6.1 Overview

Linker scripts give fine-grained control over output section layout. They are used for:
- Embedded systems: placing code in specific ROM addresses, data in specific RAM addresses.
- Custom section ordering: placing hot functions contiguously.
- Symbol definition: defining `__data_start`, `__bss_end` symbols used by startup code.

```
/* Minimal ELF linker script */
ENTRY(_start)
SECTIONS {
  . = 0x400000;          /* load address */
  .text : {
    *(.text.startup)     /* startup code first */
    *(.text .text.*)     /* then all other text */
  }
  . = ALIGN(4096);
  .data : { *(.data) }
  .bss  : { *(.bss) BYTE(0) }   /* zero-fill */
  _end = .;              /* define _end symbol */
}
```

LLD supports the GNU linker script format but enforces stricter constraints: some complex expressions (involving non-linear address arithmetic) may not be supported.

### 78.6.2 INSERT BEFORE/AFTER

LLD supports inserting sections relative to other sections without a full linker script:

```
SECTIONS { .my_section : { *(.my_section) } INSERT BEFORE .text; }
```

This is useful for instrumentation (inserting coverage data sections before `.text`) without rewriting the entire default layout.

## 78.7 Partial Linking

**Partial linking** (`-r` flag) combines multiple object files into one larger object file without resolving external references:

```bash
ld.lld -r a.o b.o -o ab.o   # combine without resolving
ld.lld ab.o c.o -o program   # final link
```

Partial linking is used in:
- Kernel builds: the Linux kernel link script uses partial linking to combine subsystem objects.
- Distributed builds: groups of modules are partially linked before final linking.
- LTO staging: partial linking preserves LLVM IR for a subsequent LTO pass.

In partial-link mode, LLD does not apply relocations, does not lay out sections into segments, and preserves all symbols as-is.

## 78.8 LLD Performance

LLD achieves 2–5× faster link times than GNU ld on large programs, primarily through:

- **Parallel I/O**: input files are read in parallel using `llvm::ThreadPool`.
- **Parallel section processing**: section content is processed and hashed in parallel.
- **Parallel output writing**: output sections are written concurrently, flushed to disk once.
- **Aggressive caching**: symbol lookups use cache-friendly hash maps (`DenseMap`).
- **Single-pass design**: no repeated passes over the input.

```bash
# Measure link time:
time clang -fuse-ld=bfd -O2 a.o b.o ... -o prog_bfd
time clang -fuse-ld=gold -O2 a.o b.o ... -o prog_gold
time clang -fuse-ld=lld -O2 a.o b.o ... -o prog_lld
```

Typical result for a 500 KLOC C++ program: bfd ≈ 15s, gold ≈ 6s, lld ≈ 2s.

## Profile-Guided Function Ordering in LLD

### The Function Placement Problem

The order of functions in an ELF `.text` section has measurable runtime impact. A binary with randomly ordered functions causes cold page faults on startup (pages containing unrelated cold functions are loaded alongside hot startup functions), wastes instruction-cache capacity (hot functions span multiple cache lines interleaved with cold code), and causes branch predictor thrash. Historically, improving function placement required BOLT (a post-link binary rewriter that reorganizes code at the basic-block level) or custom linker scripts with manually maintained function order lists. LLD's profile-guided layout (integrated in LLD 18+, improved in LLD 20–22) provides automated, profile-driven function ordering directly at link time.

### Input Format: Symbol Ordering Files

LLD accepts an explicit symbol ordering file via `--symbol-ordering-file=<file>`. The file lists symbol names one per line in the desired order; LLD places the listed symbols' sections first, in order, before all unlisted sections:

```bash
# Generate a symbol order from a perf profile:
perf record -g ./program workload_representative
perf report --sort=symbol --no-header -q 2>/dev/null \
  | awk '{print $NF}' > symbol_order.txt

# Link with the ordering:
ld.lld --symbol-ordering-file=symbol_order.txt \
       a.o b.o -o program
```

For BOLT-compatible workflows, BOLT's profiling phase produces a `profile.fdata` file from which function order can be derived, then fed to LLD for the initial link before BOLT applies finer-grained (basic-block level) reordering.

### Call-Graph Profile Sort

A more automated approach uses `--call-graph-profile-sort`, which reads call-graph profile data embedded in object files (from `-fprofile-use` or AFDO/AutoFDO instrumentation) and applies a call-chain clustering algorithm to order functions:

```bash
# Compile with PGO instrumentation:
clang -O2 -fprofile-generate -fcs-profile-generate a.cpp -c -o a.o
./program workload
llvm-profdata merge -output=merged.profdata *.profraw

# Compile with profile use (embeds call-graph weights in objects):
clang -O2 -fprofile-use=merged.profdata a.cpp -c -o a.o

# Link with call-graph-profile-sort:
ld.lld --call-graph-profile-sort a.o b.o -o program
```

The algorithm used by LLD for call-graph sorting is a variant of **Pettis-Hansen** (Improving Program Locality, PLDI 1990): functions that call each other frequently are clustered together so that call targets are on the same or adjacent cache lines. LLD implements this in `lld/ELF/CallGraphSort.cpp`.

### Section Layout: hot, warm, cold

LLD's profile-guided layout partitions functions into three tiers based on their profile weight:

```
Output section layout (profile-guided):
  .text.hot   — top-N% of profile weight; placed first (startup page-faults minimized)
  .text       — remaining functions above cold threshold
  .text.cold  — functions never called during profile run; placed last
```

Hot functions land on the first pages of the binary's executable segment, ensuring that the startup working set fits in a small number of OS pages. Cold functions are isolated, so their pages are never faulted in for typical workloads.

```bash
# Verify section layout after linking:
llvm-objdump -t program | grep -E '\.(text\.hot|text\.cold)' | head -20
readelf -S program | grep '\.text'
```

### Comparison with BOLT

BOLT (Binary Optimization and Layout Tool, `llvm/tools/bolt/`) performs post-link reordering at the basic-block level using hardware performance counters (LBR/SPE). The comparison:

| Property | LLD profile-guided layout | BOLT |
|----------|--------------------------|------|
| Granularity | Function-level | Basic-block level |
| Pipeline stage | Link time | Post-link (second tool invocation) |
| Profile source | Compiler PGO or perf LBR | Hardware LBR / perf.data |
| Requires re-link | No | No (rewrites binary in-place) |
| Startup improvement | 10–30% page fault reduction | Up to 50% startup improvement |
| `.text` size impact | Neutral | May increase (due to split functions) |
| Incremental builds | Relink reapplies ordering | Must re-BOLT on each binary change |

For most production workflows, LLD profile-guided ordering provides significant wins with minimal toolchain complexity; BOLT is reserved for binaries where maximal startup performance justifies the additional toolchain step.

### Comparison with PGO Mid-End Reordering

The compiler's mid-end PGO pipeline (Chapter 67) also reorders basic blocks within a single function using `MachineBlockPlacement` (the machine-level layout pass). The distinction:

- **Mid-end** (`MachineBlockPlacement`): reorders basic blocks within a function; operates per-module at compile time.
- **LLD profile-guided layout**: reorders whole functions within the linked binary; operates at link time with cross-module visibility.
- **BOLT**: reorders both functions and basic blocks post-link; uses runtime hardware profiles rather than compiler-instrumented profiles.

These three mechanisms are complementary and stack: compile with `-fprofile-use` (mid-end block placement), link with `--call-graph-profile-sort` (function ordering), optionally apply BOLT (basic-block reordering post-link).

### Startup Improvement Measurements

Measured startup page-fault reductions on production binaries with `--call-graph-profile-sort`:

| Binary type | Cold startup pages (baseline) | Cold startup pages (sorted) | Reduction |
|------------|------------------------------|----------------------------|-----------|
| 50 MB C++ server | ~850 pages | ~620 pages | ~27% |
| 20 MB language runtime | ~310 pages | ~240 pages | ~23% |
| 100 MB browser component | ~1600 pages | ~1200 pages | ~25% |

Reductions of 10–30% are typical; the exact number depends on how well the profile represents the startup workload.

### Integration with ThinLTO + PGO

Function ordering survives ThinLTO because LLD applies symbol ordering after the LTO optimization phase (which may rename or inline functions but does not alter externally visible symbols used for ordering). The recommended combined workflow:

```bash
# Combined ThinLTO + PGO + profile-guided ordering:
clang -O2 -flto=thin -fprofile-use=merged.profdata \
      a.cpp -c -o a.o
clang -O2 -flto=thin -fprofile-use=merged.profdata \
      b.cpp -c -o b.o

ld.lld --lto=thin \
       --call-graph-profile-sort \
       --symbol-ordering-file=startup_order.txt \
       a.o b.o -o program
```

ThinLTO cross-module inlining may alter which symbols appear in the final binary; `--call-graph-profile-sort` reads the post-LTO call graph weights (embedded by the ThinLTO backend into the native objects it produces), so the ordering reflects the actual linked program's call graph rather than the pre-LTO per-module call graphs.

### macOS-Specific LLD Behaviors

LLD's Mach-O driver (`ld64.lld`, `lld/MachO/`) implements several macOS-specific link-time behaviors that differ from ELF:

**Two-level namespace**: every symbol in a Mach-O dynamic library carries a (dylib-name, symbol-name) pair rather than just a symbol name. This allows the same symbol name to exist in multiple dylibs without conflict; the dynamic linker resolves symbols by searching the explicitly recorded dylib rather than a global namespace. LLD enforces two-level namespace by default, matching Apple's `ld` behavior. Flat namespace (`-flat_namespace`) is available for compatibility but discouraged.

**Weak dylib linking**: the load command `LC_LOAD_WEAK_DYLIB` marks a dylib as optional. If the dylib is absent at runtime, all its exported symbols default to `NULL` rather than causing a link error. LLD emits `LC_LOAD_WEAK_DYLIB` when the dylib is linked with `-weak_framework` or `-weak-l`. User code must null-check weak-linked symbols before use.

**Chained fixups** (`DYLD_CHAINED_FIXUPS`): introduced in macOS 12 / iOS 15, chained fixups replace the classic `dyld_info` rebasing and binding opcodes with a compact linked-list structure that the dynamic linker can process with fewer memory accesses. LLD emits chained fixups for arm64 targets by default; the older `dyld_info` format is used for x86-64 compatibility targets. Chained fixups reduce dyld startup time for large binaries by 10–40%.

**arm64 stack alignment**: the AArch64 ABI requires 16-byte stack alignment at all call sites. LLD's Mach-O driver enforces this at link time by checking the `STP/LDP` instruction alignment requirements and rejecting objects that violate the 16-byte alignment contract.

**LD64 compatibility**: `ld64.lld` accepts the full Apple `ld` command-line flag set, including `-arch`, `-platform_version`, `-sdk_version`, `-rpath`, `-install_name`, `-exported_symbols_list`, and the full suite of `-framework`, `-weak_framework`, `-reexport_framework` dylib linking flags. This enables LLD to serve as a drop-in replacement for Apple's `ld` in Xcode and CMake toolchain files without requiring build system changes.

---

## Chapter Summary

- LLD has four independent format drivers (ELF, COFF, Mach-O, Wasm) sharing common infrastructure; the ELF driver (`ld.lld`) is the focus for Linux/BSD development.
- **Symbol resolution** uses a hash-table `SymbolTable` with priority rules: strong > common > shared > lazy (archive) > undefined; archive members are extracted lazily on demand.
- **Relocations** patch addresses into code/data; **relaxations** rewrite instruction sequences when tighter (or more efficient) addressing modes become available at link time (TLS GD→LE, range extension thunks for ARM/AArch64, multi-relax for RISC-V).
- **COMDAT sections** deduplicate C++ template instantiations that appear in multiple TUs; **ICF** (`--icf=all`) further deduplicates functions with identical code, reducing binary size by 2–10%.
- **Linker scripts** control output section layout, load addresses, and symbol definitions; LLD supports the GNU linker script format for embedded and custom layouts.
- **Partial linking** (`-r`) combines objects without resolving externals, used in kernel builds and distributed LTO pipelines.
- LLD achieves 2–5× faster link times than GNU ld through parallel I/O, parallel section processing, and a single-pass design.
- **Profile-guided function ordering** (`--call-graph-profile-sort`, `--symbol-ordering-file`) clusters hot functions at the start of `.text` using Pettis-Hansen call-graph clustering; this reduces cold startup page faults by 10–30% and complements (but does not replace) BOLT's finer-grained basic-block reordering.
- **macOS-specific LLD behaviors** include two-level namespace symbol resolution, `LC_LOAD_WEAK_DYLIB` for optional dylibs, chained fixups (`DYLD_CHAINED_FIXUPS`) for faster dyld startup, 16-byte arm64 stack alignment enforcement, and full LD64 flag compatibility enabling drop-in replacement of Apple's linker.


---

@copyright jreuben11
