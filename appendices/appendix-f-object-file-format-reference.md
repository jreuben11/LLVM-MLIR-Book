# Appendix F — Object File Format Reference

*Quick Reference | ELF / COFF-PE / MachO / Wasm*

Quick reference for the binary object file formats supported by LLVM's MC layer, LLD, and related tools. For LLVM source see [llvm/lib/Object/](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Object/) and [llvm/include/llvm/Object/](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Object/). For linker internals see [Chapter 78 — LLD](../chapters/part-13-lto-whole-program/ch78-llvm-linker-lld.md) and [Chapter 79 — GOT/PLT/TLS](../chapters/part-13-lto-whole-program/ch79-linker-internals-got-plt-tls.md).

---

## F.1 ELF (Executable and Linking Format)

ELF is the standard on Linux, FreeBSD, Android, and most embedded targets. Header defined in `<elf.h>` / [llvm/BinaryFormat/ELF.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/BinaryFormat/ELF.h).

### ELF Header Fields

| Field | Type | Description |
|---|---|---|
| `e_ident[EI_MAG0–3]` | `\x7fELF` | Magic number |
| `e_ident[EI_CLASS]` | 1=32-bit, 2=64-bit | Address width |
| `e_ident[EI_DATA]` | 1=LE, 2=BE | Endianness |
| `e_ident[EI_VERSION]` | 1 | ELF version (always 1) |
| `e_ident[EI_OSABI]` | 0=SYSV, 3=Linux, 6=Solaris, 9=FreeBSD | OS/ABI |
| `e_type` | ET_REL/ET_EXEC/ET_DYN/ET_CORE | File type |
| `e_machine` | EM_X86_64 (62), EM_AARCH64 (183), EM_RISCV (243), EM_BPF (247), EM_ARM (40) | Target architecture |
| `e_version` | 1 | Object version |
| `e_entry` | addr | Entry point (0 for relocatable) |
| `e_phoff` | offset | Program header table offset |
| `e_shoff` | offset | Section header table offset |
| `e_flags` | target-specific | ABI/ISA flags (e.g., RISC-V: EF_RISCV_RVC, EF_RISCV_FLOAT_ABI_*) |
| `e_shstrndx` | index | Section index of `.shstrtab` |

### Key ELF Sections

| Section | Flags | Description |
|---|---|---|
| `.text` | AX (alloc, exec) | Executable code |
| `.data` | AW (alloc, write) | Initialized read-write data |
| `.bss` | AW, NOBITS | Zero-initialized data; no file bytes |
| `.rodata` | A (alloc) | Read-only data (string literals, vtables) |
| `.symtab` | — | Static symbol table |
| `.strtab` | — | String table for `.symtab` |
| `.shstrtab` | — | Section name string table |
| `.rela.text` | — | Relocations for `.text` with addends |
| `.rel.text` | — | Relocations without addends (32-bit ELF) |
| `.eh_frame` | A | DWARF CFI unwind information (loaded) |
| `.debug_info` | — | DWARF DIEs (not loaded) |
| `.debug_abbrev` | — | DWARF abbreviation table |
| `.debug_line` | — | DWARF line number table |
| `.debug_str` | — | DWARF string pool |
| `.debug_names` | — | DWARF 5 name index |
| `.debug_loclists` | — | DWARF 5 location lists |
| `.debug_rnglists` | — | DWARF 5 range lists |
| `.BTF` | — | BPF Type Format (for eBPF objects) |
| `.BTF.ext` | — | BPF function/line info extensions |
| `.note.gnu.property` | A | Property notes (IBT/SHSTK, PAC) |
| `.note.ABI-tag` | A | Minimum OS version note |
| `.note.gnu.build-id` | A | Unique build identifier (SHA1/MD5/UUID) |
| `.tdata` / `.tbss` | — | Thread-local data / zero-init TLS |
| `.init_array` / `.fini_array` | AW | Constructors/destructors (C++) |
| `.llvm_bc` | — | LTO bitcode embedded in object |
| `.llvm.lto` | — | ThinLTO summary + bitcode |

### ELF Symbol Table Entry

```c
typedef struct {
    Elf64_Word  st_name;   // offset in .strtab
    unsigned char st_info; // (bind << 4) | type
    unsigned char st_other;// visibility
    Elf64_Half  st_shndx;  // section index (SHN_UNDEF, SHN_ABS, SHN_COMMON)
    Elf64_Addr  st_value;  // symbol value / address
    Elf64_Xword st_size;   // symbol size in bytes
} Elf64_Sym;
```

**Binding** (`st_info >> 4`): `STB_LOCAL` (0), `STB_GLOBAL` (1), `STB_WEAK` (2), `STB_GNU_UNIQUE` (10).

**Type** (`st_info & 0xf`): `STT_NOTYPE` (0), `STT_OBJECT` (1), `STT_FUNC` (2), `STT_SECTION` (3), `STT_FILE` (4), `STT_COMMON` (5), `STT_TLS` (6), `STT_GNU_IFUNC` (10).

**Visibility** (`st_other & 0x3`): `STV_DEFAULT` (0), `STV_INTERNAL` (1), `STV_HIDDEN` (2), `STV_PROTECTED` (3).

### x86-64 Relocation Types

| Relocation | Value | Calculation | Use |
|---|---|---|---|
| `R_X86_64_64` | 1 | S + A | Absolute 64-bit |
| `R_X86_64_PC32` | 2 | S + A - P | 32-bit PC-relative |
| `R_X86_64_GOT32` | 3 | G + A | GOT entry 32-bit offset |
| `R_X86_64_PLT32` | 4 | L + A - P | PLT entry PC-relative |
| `R_X86_64_COPY` | 5 | — | Copy from shared lib |
| `R_X86_64_GLOB_DAT` | 6 | S | GOT entry for symbol |
| `R_X86_64_JUMP_SLOT` | 7 | S | PLT GOT entry |
| `R_X86_64_RELATIVE` | 8 | B + A | Base + addend (PIC) |
| `R_X86_64_GOTPCREL` | 9 | G + GOT + A - P | GOT PC-relative |
| `R_X86_64_32` | 10 | S + A | 32-bit absolute (zero-ext) |
| `R_X86_64_32S` | 11 | S + A | 32-bit absolute (sign-ext) |
| `R_X86_64_TPOFF32` | 23 | S + A - TP | TLS local-exec offset |
| `R_X86_64_TLSGD` | 19 | — | TLS global-dynamic GD sequence |
| `R_X86_64_TLSLD` | 20 | — | TLS local-dynamic LD sequence |
| `R_X86_64_DTPOFF32` | 21 | S + A - TLS | TLS offset from TLS block |
| `R_X86_64_GOTTPOFF` | 22 | S + A - GOT | TLS initial-exec IE sequence |
| `R_X86_64_SIZE32` | 32 | Z + A | Symbol size 32-bit |
| `R_X86_64_GOTPCRELX` | 41 | G + GOT + A - P | Relaxable GOTPCREL |

S = symbol value, A = addend, P = place address, B = base, G = GOT offset, L = PLT offset, TP = thread pointer.

### AArch64 Relocation Types

| Relocation | Use |
|---|---|
| `R_AARCH64_ABS64` | Absolute 64-bit |
| `R_AARCH64_ABS32` | Absolute 32-bit |
| `R_AARCH64_PREL32` | PC-relative 32-bit |
| `R_AARCH64_ADR_PREL_PG_HI21` | `ADRP` instruction: page-relative hi21 |
| `R_AARCH64_ADD_ABS_LO12_NC` | `ADD` instruction: absolute lo12 (no check) |
| `R_AARCH64_LDST64_ABS_LO12_NC` | Load/store 64-bit: absolute lo12 |
| `R_AARCH64_LDST32_ABS_LO12_NC` | Load/store 32-bit: absolute lo12 |
| `R_AARCH64_LDST16_ABS_LO12_NC` | Load/store 16-bit: absolute lo12 |
| `R_AARCH64_LDST8_ABS_LO12_NC` | Load/store 8-bit: absolute lo12 |
| `R_AARCH64_JUMP26` | Unconditional branch (26-bit offset) |
| `R_AARCH64_CALL26` | `BL` instruction (26-bit offset) |
| `R_AARCH64_ADR_GOT_PAGE` | GOT page offset for `ADRP` |
| `R_AARCH64_LD64_GOT_LO12_NC` | GOT lo12 for `LDR` |
| `R_AARCH64_TLSDESC` | TLS descriptor GOT pair |
| `R_AARCH64_TLSIE_ADR_GOTTPREL_PAGE21` | TLS IE: `ADRP` to GOTTPREL page |
| `R_AARCH64_TLSIE_LD64_GOTTPREL_LO12_NC` | TLS IE: `LDR` from GOTTPREL |
| `R_AARCH64_TLSLE_ADD_TPREL_HI12` | TLS LE: `ADD` hi12 of TP offset |
| `R_AARCH64_TLSLE_ADD_TPREL_LO12_NC` | TLS LE: `ADD` lo12 of TP offset |

### Dynamic Linking Sections

| Section | Description |
|---|---|
| `.dynamic` | Array of `Elf64_Dyn` tag-value pairs; parsed by the dynamic linker |
| `.got` | Global Offset Table: addresses of global data symbols |
| `.got.plt` | GOT entries for PLT stubs (lazily resolved) |
| `.plt` | PLT stubs for external function calls |
| `.plt.got` | PLT stubs using `.got` (non-lazy) |
| `.interp` | Path to the dynamic linker (`/lib64/ld-linux-x86-64.so.2`) |

Key `DT_*` tags in `.dynamic`:

| Tag | Value | Description |
|---|---|---|
| `DT_NEEDED` | 1 | Name of required shared library |
| `DT_PLTRELSZ` | 2 | Size of PLT reloc table |
| `DT_STRTAB` | 5 | Address of string table |
| `DT_SYMTAB` | 6 | Address of symbol table |
| `DT_RELA` | 7 | Rela relocation table address |
| `DT_RELASZ` | 8 | Size of Rela table |
| `DT_INIT` | 12 | Address of initialization function |
| `DT_FINI` | 13 | Address of finalization function |
| `DT_SONAME` | 14 | Shared library name |
| `DT_RPATH` | 15 | Runtime library search path (deprecated) |
| `DT_RUNPATH` | 29 | Runtime library search path |
| `DT_INIT_ARRAY` | 25 | Address of `.init_array` |
| `DT_FLAGS` | 30 | DF_BIND_NOW, DF_PIE, etc. |
| `DT_FLAGS_1` | 0x6ffffffb | DF_1_NOW, DF_1_PIE, DF_1_NODELETE |
| `DT_GNU_HASH` | 0x6ffffef5 | GNU hash table for faster symbol lookup |

---

## F.2 COFF/PE (Windows)

PE32+ is used on 64-bit Windows. Source: [llvm/BinaryFormat/COFF.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/BinaryFormat/COFF.h).

### PE32+ Structure

```
[DOS MZ stub]
[PE signature "PE\0\0"]
[COFF File Header]
[Optional Header (PE32+)]
[Section Table]
[Section Data...]
```

### COFF File Header

| Field | Size | Description |
|---|---|---|
| `Machine` | 2 | 0x8664=x86-64, 0xAA64=ARM64, 0x1C0=ARM |
| `NumberOfSections` | 2 | Section count |
| `TimeDateStamp` | 4 | Creation timestamp (Unix) |
| `PointerToSymbolTable` | 4 | Offset to COFF symbol table (0 in PE) |
| `NumberOfSymbols` | 4 | Symbol count |
| `SizeOfOptionalHeader` | 2 | Must be non-zero for executables |
| `Characteristics` | 2 | File flags: IMAGE_FILE_EXECUTABLE_IMAGE, DLL, etc. |

### Optional Header Key Fields (PE32+)

| Field | Description |
|---|---|
| `Magic` | 0x20B for PE32+ (64-bit) |
| `AddressOfEntryPoint` | RVA of entry point |
| `ImageBase` | Default load address (0x140000000 for exe) |
| `SectionAlignment` | Alignment of sections in memory (4096 typical) |
| `FileAlignment` | Alignment of sections in file (512 typical) |
| `SizeOfImage` | Total size of loaded image |
| `SizeOfHeaders` | Size of all headers |
| `Subsystem` | 2=WINDOWS_GUI, 3=WINDOWS_CONSOLE, 9=NATIVE |
| `DllCharacteristics` | NX compat, ASLR support, CFG, etc. |
| `SizeOfStackReserve` / `Commit` | Stack size |
| `DataDirectory[16]` | RVA+size of tables: exports, imports, resources, relocs, debug, TLS, IAT, CLR, etc. |

### Key PE Sections

| Section | Description |
|---|---|
| `.text` | Executable code |
| `.data` | Initialized read-write data |
| `.rdata` | Read-only data (string literals, import tables) |
| `.bss` | Zero-initialized data |
| `.pdata` | Exception handler data (RUNTIME_FUNCTION array for x64 SEH) |
| `.xdata` | Unwind information for SEH (UNWIND_INFO structures) |
| `.tls` | Thread-local storage template |
| `.reloc` | Base relocations (IMAGE_BASE_RELOCATION blocks) |
| `.edata` | Export directory |
| `.idata` | Import directory (IDT + IAT + ILT) |
| `.rsrc` | Resources (icons, manifests, version info) |
| `/4` | Compiler flags record (MSVC `cl`) |
| `.debug$S` | CodeView symbol records |
| `.debug$T` | CodeView type records |

### Import Directory Table

Each DLL import: `OriginalFirstThunk` (ILT: names), `TimeDateStamp`, `ForwarderChain`, `Name` (DLL name RVA), `FirstThunk` (IAT: fixed up at load time).

### COFF Relocations (x86-64)

| Type | Value | Description |
|---|---|---|
| `IMAGE_REL_AMD64_ABSOLUTE` | 0 | No relocation |
| `IMAGE_REL_AMD64_ADDR64` | 1 | 64-bit VA |
| `IMAGE_REL_AMD64_ADDR32` | 2 | 32-bit VA |
| `IMAGE_REL_AMD64_ADDR32NB` | 3 | 32-bit RVA (no base) |
| `IMAGE_REL_AMD64_REL32` | 4 | 32-bit relative to next instruction |
| `IMAGE_REL_AMD64_REL32_1/2/3/4/5` | 5–9 | As above with additional offset |
| `IMAGE_REL_AMD64_SECTION` | 10 | 16-bit section index |
| `IMAGE_REL_AMD64_SECREL` | 11 | 32-bit offset from section start |
| `IMAGE_REL_AMD64_SECREL7` | 12 | 7-bit offset from section start |
| `IMAGE_REL_AMD64_TOKEN` | 13 | CLR metadata token |

---

## F.3 MachO (macOS / iOS / iPadOS)

Mach-O is the binary format for Apple platforms. Source: [llvm/BinaryFormat/MachO.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/BinaryFormat/MachO.h).

### Mach-O Header

```c
struct mach_header_64 {
    uint32_t magic;      // 0xFEEDFACF (LE) or 0xCFFAEDFE (BE)
    cpu_type_t cputype;  // CPU_TYPE_X86_64 (0x01000007), CPU_TYPE_ARM64 (0x0100000C)
    cpu_subtype_t cpusubtype; // CPU_SUBTYPE_ALL, CPU_SUBTYPE_ARM64E (PAC)
    uint32_t filetype;   // MH_OBJECT, MH_EXECUTE, MH_DYLIB, MH_DYLINKER, MH_BUNDLE, MH_DSYM, MH_KEXT_BUNDLE
    uint32_t ncmds;      // number of load commands
    uint32_t sizeofcmds; // total size of load commands
    uint32_t flags;      // MH_NOUNDEFS, MH_DYLDLINK, MH_TWOLEVEL, MH_PIE, MH_HAS_TLV_DESCRIPTORS
    uint32_t reserved;
};
```

### Key Load Commands

| Command | Description |
|---|---|
| `LC_SEGMENT_64` | Map segment of file into memory; contains section list |
| `LC_SYMTAB` | Symbol table offset and string table location |
| `LC_DYSYMTAB` | Dynamic symbol table indices (local, defined, undefined, indirect) |
| `LC_LOAD_DYLIB` | Required shared library: name, version, compatibility version |
| `LC_LOAD_WEAK_DYLIB` | Weakly required shared library |
| `LC_ID_DYLIB` | Library identification (in dylibs) |
| `LC_RPATH` | Runtime search path for dylibs |
| `LC_UUID` | 128-bit unique identifier (used for debug symbol matching) |
| `LC_CODE_SIGNATURE` | Code signature blob offset/size |
| `LC_FUNCTION_STARTS` | Compressed list of function start addresses |
| `LC_DATA_IN_CODE` | Inline data in code regions (ARM constants) |
| `LC_DYLD_INFO_ONLY` | Dyld bind/rebase/lazy bind/export info (compressed) |
| `LC_DYLD_EXPORTS_TRIE` | Export trie (macOS 12+, dyld4) |
| `LC_DYLD_CHAINED_FIXUPS` | Chained fixups (dyld3/dyld4 deployment) |
| `LC_BUILD_VERSION` | Platform, minos, SDK, tool versions |
| `LC_SOURCE_VERSION` | Source version number |
| `LC_LINKER_OPTIMIZATION_HINT` | LTO optimization hints |
| `LC_NOTE` | Arbitrary note data (LLDB extensions) |

### Segments and Sections

| Segment | Section | Description |
|---|---|---|
| `__TEXT` | `__text` | Executable code |
| `__TEXT` | `__stubs` | PLT stubs (indirect symbol trampolines) |
| `__TEXT` | `__stub_helper` | PLT stub helper (lazy bind) |
| `__TEXT` | `__cstring` | C string literals (merged) |
| `__TEXT` | `__const` | Read-only constants |
| `__TEXT` | `__unwind_info` | Compact unwind data (Apple format) |
| `__TEXT` | `__eh_frame` | DWARF CFI unwind (alternative/legacy) |
| `__DATA` | `__data` | Initialized read-write data |
| `__DATA` | `__bss` | Zero-initialized data |
| `__DATA` | `__la_symbol_ptr` | Lazy symbol pointer table |
| `__DATA` | `__nl_symbol_ptr` | Non-lazy symbol pointer table |
| `__DATA` | `__got` | Indirect symbol pointers (GOT) |
| `__DATA` | `__thread_data` | TLS data template |
| `__DATA` | `__thread_bss` | Zero-init TLS |
| `__DATA` | `__thread_vars` | TLS descriptor pointers |
| `__DATA_CONST` | `__got` | Read-only after dyld fixups (W^X) |
| `__LINKEDIT` | — | Symbol table, string table, code signature, dyld info |

### MachO Relocations (x86-64)

| Type | Description |
|---|---|
| `X86_64_RELOC_BRANCH` | 32-bit relative to PC (for `CALL`/`JMP`) |
| `X86_64_RELOC_GOT_LOAD` | `movq symbol@GOTPCREL(%rip), %reg` |
| `X86_64_RELOC_GOT` | GOT-relative (not a load) |
| `X86_64_RELOC_SIGNED` | Signed 32-bit displacement |
| `X86_64_RELOC_SIGNED_1/2/4` | Signed with 1/2/4-byte offset adjustment |
| `X86_64_RELOC_UNSIGNED` | 64-bit absolute |
| `X86_64_RELOC_SUBTRACTOR` | Difference of two symbols (paired) |
| `X86_64_RELOC_TLV` | Thread-local variable access |

### Fat Binaries (Universal Binaries)

```c
struct fat_header {
    uint32_t magic;    // 0xCAFEBABE
    uint32_t nfat_arch;
};
struct fat_arch {
    cpu_type_t cputype;
    cpu_subtype_t cpusubtype;
    uint32_t offset;   // file offset of object for this arch
    uint32_t size;
    uint32_t align;    // power of 2
};
```

Tool: `lipo` — create (`-create`), extract (`-extract <arch>`), info (`-info`), thin (`-thin <arch>` to extract single arch).

---

## F.4 WebAssembly Object Format

Wasm objects follow the WebAssembly binary format with LLVM-specific custom sections for linking. Reference: [WebAssembly Object File Linking spec](https://github.com/WebAssembly/tool-conventions/blob/main/Linking.md).

### Wasm Section Types

| Section ID | Name | Description |
|---|---|---|
| 1 | Type | Function type signatures |
| 2 | Import | Imported functions/globals/memories/tables |
| 3 | Function | Map function index → type index |
| 4 | Table | Tables (function pointer tables) |
| 5 | Memory | Memory limits |
| 6 | Global | Module-level globals |
| 7 | Export | Exported names |
| 8 | Start | Start function index |
| 9 | Element | Passive/active element segments |
| 10 | Code | Function bodies |
| 11 | Data | Data segments |
| 0 | Custom | Named; see below |

### Wasm Custom Sections

| Name | Description |
|---|---|
| `name` | Debug names for functions, locals, types |
| `linking` | Linking metadata: symbol table, segment info, init functions, comdat |
| `reloc.*` | Relocation entries for code and data sections |
| `producers` | Tool chain info: language, tools, SDK |
| `target_features` | Required/used Wasm features (simd128, bulk-memory, etc.) |
| `sourceMappingURL` | Reference to source map file |

### Wasm Relocation Types (selected)

| Type | Description |
|---|---|
| `R_WASM_FUNCTION_INDEX_LEB` | LEB-encoded function index |
| `R_WASM_TABLE_INDEX_SLEB` | SLEB-encoded table index (for indirect calls) |
| `R_WASM_TABLE_INDEX_I32` | 32-bit table index |
| `R_WASM_MEMORY_ADDR_LEB` | LEB memory address |
| `R_WASM_MEMORY_ADDR_SLEB` | SLEB memory address |
| `R_WASM_MEMORY_ADDR_I32` | 32-bit memory address |
| `R_WASM_TYPE_INDEX_LEB` | LEB-encoded type index |
| `R_WASM_GLOBAL_INDEX_LEB` | LEB global index |
| `R_WASM_FUNCTION_OFFSET_I32` | Offset within function body |
| `R_WASM_SECTION_OFFSET_I32` | Offset within section |
| `R_WASM_TAG_INDEX_LEB` | Exception tag index (exception proposal) |

---

## F.5 LLVM MC Layer

The LLVM MC (Machine Code) layer handles encoding and emission of all object formats. Key classes:

| Class | Role |
|---|---|
| `MCContext` | Central context for symbols, sections, streaming state |
| `MCStreamer` | Abstract interface: `MCAsmStreamer` (text), `MCObjectStreamer` (binary) |
| `MCELFStreamer` | ELF binary object emission |
| `MCMachOStreamer` | MachO binary emission |
| `MCWinCOFFStreamer` | COFF binary emission |
| `MCWasmStreamer` | Wasm binary emission |
| `MCSymbol` | Named symbol with value/section |
| `MCSection` | Named section with alignment and flags |
| `MCInst` | In-memory machine instruction (opcode + operands) |
| `MCFixup` | Relocation request from encoder |
| `MCAssembler` | Assembles MCInst sequences and emits fixups |
| `MCObjectWriter` | Final ELF/COFF/MachO/Wasm writer |

Source: [llvm/lib/MC/](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/MC/) and [Chapter 94 — The MC Layer](../chapters/part-14-backend/ch94-mc-layer-and-mir-test-infrastructure.md).


---

@copyright jreuben11
