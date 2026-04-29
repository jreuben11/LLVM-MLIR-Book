# Chapter 78 — The LLVM Linker (LLD)

*Part XIII — Link-Time and Whole-Program*

LLD is LLVM's production linker, supporting ELF (Linux, BSD, embedded), COFF (Windows), Mach-O (macOS/iOS), and WebAssembly object formats. It was written from scratch to overcome the scalability and maintainability limitations of GNU ld and gold. LLD is the default linker for Android, Chrome, Fuchsia, and many other large software projects. This chapter covers LLD's architecture, the per-format driver design, symbol resolution, relocation and relaxation, linker scripts, ICF (identical code folding), and partial linking.

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

---

## Chapter Summary

- LLD has four independent format drivers (ELF, COFF, Mach-O, Wasm) sharing common infrastructure; the ELF driver (`ld.lld`) is the focus for Linux/BSD development.
- **Symbol resolution** uses a hash-table `SymbolTable` with priority rules: strong > common > shared > lazy (archive) > undefined; archive members are extracted lazily on demand.
- **Relocations** patch addresses into code/data; **relaxations** rewrite instruction sequences when tighter (or more efficient) addressing modes become available at link time (TLS GD→LE, range extension thunks for ARM/AArch64, multi-relax for RISC-V).
- **COMDAT sections** deduplicate C++ template instantiations that appear in multiple TUs; **ICF** (`--icf=all`) further deduplicates functions with identical code, reducing binary size by 2–10%.
- **Linker scripts** control output section layout, load addresses, and symbol definitions; LLD supports the GNU linker script format for embedded and custom layouts.
- **Partial linking** (`-r`) combines objects without resolving externals, used in kernel builds and distributed LTO pipelines.
- LLD achieves 2–5× faster link times than GNU ld through parallel I/O, parallel section processing, and a single-pass design.
