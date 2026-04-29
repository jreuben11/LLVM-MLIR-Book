# Chapter 79 — Linker Internals: GOT, PLT, TLS

*Part XIII — Link-Time and Whole-Program*

Dynamic linking, thread-local storage, and IFUNC resolution are implemented through several interdependent linker-generated data structures: the Global Offset Table (GOT), the Procedure Linkage Table (PLT), and TLS descriptor tables. These structures mediate between static compilation-time addresses and runtime-determined addresses — for DSO base addresses, thread-local variable offsets, and runtime function resolution. This chapter develops the mechanics of each structure, their interplay with CPU-level support (FSBASE/GSBASE for TLS, CFI for PLT), LLD's implementation, and the related `.ctors`/`.dtors`/`.init_array` mechanisms.

## 79.1 Position-Independent Code and the Need for GOT

### 79.1.1 The Problem

A **shared library** (DSO, dynamic shared object) may be loaded at any virtual address. Instructions that reference symbols by absolute address must be patched at load time (dynamic relocations). For a DSO with thousands of such references, patching at load time is slow and — more critically — prevents the OS from sharing read-only pages between processes.

**Position-Independent Code** (PIC) solves this: code references data via the **Global Offset Table** (GOT) rather than by absolute address. The GOT is a per-process, per-DSO writable table of absolute addresses. Code reads the GOT at runtime rather than encoding absolute addresses.

```asm
; Non-PIC (needs runtime patching):
movq   $0x601020, %rdi   ; absolute address of global_var — needs relocation

; PIC (no patching needed in text section):
movq   global_var@GOTPCREL(%rip), %rdi  ; load GOT entry address (PC-relative)
movq   (%rdi), %rdi                      ; load actual address from GOT
```

The PIC sequence is two instructions instead of one, but the `.text` section is read-only and sharable; only the GOT (in `.data`) needs patching.

### 79.1.2 GOT Layout

The GOT is a table of pointers in `.got` and `.got.plt`:

```
.got:
  GOT[0]: address of .dynamic section  (special)
  GOT[1]: link map pointer              (special, filled by ld.so)
  GOT[2]: ld.so lazy binding resolver  (special, filled by ld.so)
  GOT[n]: address of symbol n          (filled at load time or lazily)

.got.plt: (separate for PLT lazy binding)
  PLT entries' GOT slots (initially: second instruction of PLT stub)
```

LLD generates the `.got` and `.got.plt` sections by collecting all GOT-generating relocations (`R_X86_64_GOT*`) across all input sections.

### 79.1.3 GOT Relocation: GOTPCREL

The `R_X86_64_GOTPCREL` relocation tells the linker: "this instruction loads the address of the GOT entry for symbol S, PC-relative." LLD:

1. Allocates a GOT entry for symbol S.
2. Fills the GOT entry with S's absolute address at load time (via a `R_X86_64_GLOB_DAT` dynamic relocation).
3. Patches the instruction's displacement to be `GOT_entry_address − (instruction_address + 4)`.

```
// Example: movq global_var@GOTPCREL(%rip), %rax
// After LLD:
// .text: movq 0x1234(%rip), %rax   (displacement = GOT[n] - (%rip))
// .got:  [slot n]: R_X86_64_GLOB_DAT global_var  (filled by ld.so)
```

## 79.2 The Procedure Linkage Table (PLT)

### 79.2.1 Lazy Binding

When a program calls a function in a shared library for the first time, the library may not even be mapped yet (if on-demand loading is enabled). Even if mapped, the function's address must be looked up in the DSO's symbol table. This lookup is **lazy**: deferred until the first call.

The **PLT** implements lazy binding:

1. The call site calls `function@plt` (a stub in the PLT).
2. The PLT stub jumps through the GOT. Initially, the GOT entry points to the *second instruction of the PLT stub* (the push+jmp).
3. The push+jmp pushes the PLT slot number and jumps to the **lazy resolver** (`GOT[2]`).
4. The resolver (`_dl_runtime_resolve` in glibc) looks up the symbol, updates the GOT entry with the real address, and jumps to the function.
5. On subsequent calls, the GOT entry already has the real address — the lazy binding overhead is a single extra memory indirection.

### 79.2.2 PLT Stub Structure

On x86-64, each PLT stub is 16 bytes:

```asm
function@plt:
  jmp    *function@GOTPLT(%rip)   ; indirect jump through GOT entry
  pushq  $plt_index               ; PLT slot number (for resolver)
  jmp    plt_start                ; jump to resolver trampoline

plt_start:                        ; resolver trampoline (first PLT entry)
  pushq  *GOT[1](%rip)            ; push link-map
  jmp    *GOT[2](%rip)            ; jump to _dl_runtime_resolve
```

LLD generates the PLT stub for each dynamically-bound external function.

### 79.2.3 `-z now` (Eager Binding)

With `-z now` (or the `BIND_NOW` flag), all PLT entries are resolved at startup rather than lazily:

```bash
# Eager binding (resolve all PLT entries at startup):
ld.lld -z now a.o -o program
```

With `-z now`, the `.got.plt` entries are filled by `ld.so` before `main()` executes. PLT stubs still exist (the code uses them), but the lazy binding overhead is shifted entirely to startup time.

Combined with `-z relro -z now`, the entire `.got.plt` can be marked read-only after startup (full RELRO), preventing GOT overwrite attacks.

### 79.2.4 IRELATIVE and IFUNC

**GNU IFUNC** (`__attribute__((ifunc("resolver")))`) selects a function implementation at load time based on CPU capabilities:

```c
static void *memcpy_resolver(void) {
  if (cpu_has_avx512()) return memcpy_avx512;
  if (cpu_has_avx2())  return memcpy_avx2;
  return memcpy_generic;
}
void *memcpy(void *, const void *, size_t)
    __attribute__((ifunc("memcpy_resolver")));
```

At link time, LLD generates an `R_X86_64_IRELATIVE` relocation for the memcpy GOT entry. At load time, `ld.so` calls `memcpy_resolver()` and stores the returned function pointer in the GOT.

```
.got:
  [n]: R_X86_64_IRELATIVE (base + resolver_addr)  → ld.so calls resolver, stores result
```

## 79.3 Thread-Local Storage (TLS)

TLS variables (`__thread` / `thread_local`) have a separate instance per thread. Each thread has a **Thread Control Block (TCB)** containing a pointer to its TLS block. On x86-64, the TCB is at `%fs:0`; on AArch64, it is at `TPIDR_EL0`. The TLS variable access computes an offset from the thread pointer.

### 79.3.1 Four TLS Models

| Model | Name | Use | Flags |
|-------|------|-----|-------|
| General Dynamic (GD) | Most general | Variables in any DSO | Default for DSOs |
| Local Dynamic (LD) | Within-module variables | Multiple `thread_local` in same DSO | `-ftls-model=local-dynamic` |
| Initial Exec (IE) | Non-DSO accesses from DSO | Variables in executable, accessed from DSO | `-ftls-model=initial-exec` |
| Local Exec (LE) | Fastest | Variables in executable, accessed from executable | `-ftls-model=local-exec` |

### 79.3.2 General Dynamic TLS

GD TLS is the most general model — it works when the variable is in *any* DSO, but requires two function calls per access:

```asm
; GD TLS access (x86-64):
leaq  var@tlsgd(%rip), %rdi       ; load address of TLS descriptor pair
                                   ; R_X86_64_TLSGD relocation
call  __tls_get_addr@plt           ; call runtime to get thread-specific address
; (%rax) now contains &var for this thread
movl  (%rax), %ecx                 ; load the value
```

The `__tls_get_addr` call is expensive (may involve hash table lookup in the TLS runtime). LLD generates a `R_X86_64_TLSGD` relocation that triggers an `R_X86_64_DTPMOD64`/`R_X86_64_DTPOFF64` pair at load time.

### 79.3.3 Initial Exec TLS

IE TLS is used when the variable is in the main executable but accessed from a shared library. The offset is computed once at load time and stored in the GOT:

```asm
; IE TLS access (x86-64):
movq  var@GOTTPOFF(%rip), %rax    ; load TLS offset from GOT
                                   ; R_X86_64_GOTTPOFF relocation
movl  %fs:(%rax), %ecx            ; load via thread pointer + offset
```

LLD generates a `R_X86_64_TPOFF64` dynamic relocation for the GOT entry: `ld.so` fills it with `var's offset from TP`. The GOT read is one extra indirection compared to LE, but only one function call is needed (none at access time).

### 79.3.4 Local Exec TLS

LE TLS is the fastest model — the variable is in the main executable and accessed from the main executable. The offset from the thread pointer is a compile-time constant:

```asm
; LE TLS access (x86-64):
movl  var@TPOFF(%rip), %eax       ; load from TP-relative offset (compile-time const)
movl  %fs:(%rax), %ecx            ; or: movl %fs:var@TPOFF, %ecx (single instruction)
```

No GOT entry, no function call — the thread pointer plus a known offset gives the variable's address directly.

### 79.3.5 TLS Relaxation by LLD

LLD applies TLS relaxation to convert the more general (and slower) TLS access models to the faster one when the linker can determine the symbol is in the main executable:

| From | To | Condition |
|------|----|-----------|
| GD | IE | linking a shared library that uses a GD variable from the executable |
| GD | LE | linking the main executable (final link, static TLS known) |
| LD | LE | same: local dynamic can become local exec in the executable |
| IE | LE | symbol is in the main executable and offset fits 32 bits |

LLD's `TlsRelaxation.cpp` applies these relaxations by rewriting the instruction sequence and updating the relocation type.

### 79.3.6 TLSDESC (AArch64, x86-64)

**TLSDESC** (TLS Descriptor, RFC 6063) is an alternative to the `__tls_get_addr`-based GD model that eliminates the call overhead in the common case. The descriptor is a two-word structure:

```
tlsdesc_pair:
  [0]: resolver function pointer (filled by ld.so)
  [1]: argument (TLS offset or other data)
```

At access time, the resolver is called via an indirect `blr` (AArch64) or `call [descriptor]` (x86-64). For modules in the initial TLS block (loaded before threads start), the resolver is a no-op that just returns the pre-computed offset — much faster than `__tls_get_addr`.

```asm
; AArch64 TLSDESC access:
adrp  x0, :tlsdesc:var         ; load descriptor address (PC-relative)
ldr   x1, [x0, #:tlsdesc_lo12:var]    ; load resolver pointer
add   x0, x0, :tlsdesc_lo12:var       ; compute argument address
blr   x1                       ; call resolver (may be no-op for initial TLS)
mrs   x1, tpidr_el0            ; get thread pointer
ldr   w0, [x1, x0]             ; access TLS variable
```

LLD supports `TLSDESC` via the `--tls-descriptor` flag and emits `R_AARCH64_TLSDESC_*` / `R_X86_64_TLSDESC_*` relocations.

## 79.4 `.eh_frame` and Exception Handling

### 79.4.1 The `.eh_frame` Section

`.eh_frame` is a DWARF-based call frame information (CFI) section that records how to unwind the stack for each instruction in each function. It is used by:
- C++ `throw`: the unwinder reads `.eh_frame` to walk the call stack and find matching `catch` handlers.
- `backtrace()`: stack trace utilities read `.eh_frame` to decode return addresses.
- Profilers (perf, gdb): unwinding for call-graph sampling.

LLD merges all input `.eh_frame` sections, deduplicates common FDE (Frame Description Entry) records, and emits the merged `.eh_frame`.

```
.eh_frame layout:
  CIE (Common Information Entry): encoding parameters common to many FDEs
  FDE (Frame Description Entry): per-function unwind instructions
    - PC range (start address, length)
    - Call frame instructions: how to recover registers at each PC
```

### 79.4.2 `.eh_frame_hdr`

The `.eh_frame_hdr` section is a sorted, binary-searchable index into `.eh_frame`, enabling O(log N) unwind frame lookup instead of O(N) linear scan. LLD generates `.eh_frame_hdr` when linking with `-z eh-frame-hdr` (the default for Linux targets):

```
.eh_frame_hdr:
  version, encoding, FDE count
  initial_pc_sorted_table[]:
    (initial_pc, FDE_ptr)  — sorted by initial_pc
```

The linker generates the sorted table; the unwinder does binary search on `initial_pc` to find the FDE for a given PC.

## 79.5 `.ctors`, `.dtors`, `.init_array`, `.fini_array`

### 79.5.1 C++ Constructors and Destructors

C++ global objects with constructors and destructors require initialization before `main()` and cleanup after. The linker organizes these through special sections:

**Legacy `.ctors`/`.dtors`** (used by older GCC):
```
.ctors:
  ULONG -1         (sentinel start)
  constructor_ptr1
  constructor_ptr2
  ULONG 0          (sentinel end)
```

The C runtime's `__do_global_ctors` function walks this table at startup, calling each function pointer.

**`.init_array`/`.fini_array`** (modern, ELF standard):
```
.init_array:
  ctor1_ptr
  ctor2_ptr
  ...
```

Sorted by `__attribute__((init_priority(N)))`: lower priority values run first. LLD sorts `.init_array` entries by their priority annotations.

The CRT startup code (`_start`) calls `__libc_csu_init`, which calls each entry in `.init_array` in order before `main()`.

### 79.5.2 Priority Ordering

Constructor priority controls initialization order across translation units:

```c
// Initialized before default-priority constructors (lower number = earlier):
__attribute__((constructor(100)))
void early_init() { /* runs before main(), before priority-65535 ctors */ }

__attribute__((constructor(65535)))
void late_init() { /* runs last before main() */ }
```

LLD sorts `.init_array` by priority at link time, ensuring correct initialization order across the entire program.

### 79.5.3 `__attribute__((visibility("hidden")))` and `.init_array`

Constructors marked with hidden visibility are placed in the same `.init_array` sort group, but their symbols are not exported from the DSO. This prevents accidental execution from external code and reduces DSO symbol table size.

---

## Chapter Summary

- The **GOT** (Global Offset Table) in `.got` and `.got.plt` stores absolute addresses that are filled at load time by `ld.so`, allowing PIC code in `.text` to remain read-only and sharable; `GOTPCREL` relocations encode load offsets relative to the current PC.
- The **PLT** (Procedure Linkage Table) implements lazy dynamic symbol binding: the first call goes through a stub that invokes `_dl_runtime_resolve`; subsequent calls through the same PLT stub use the GOT entry filled by the resolver, costing only one extra memory indirection.
- **TLS** has four models: General Dynamic (two calls, fully general), Local Dynamic (one call, within-module), Initial Exec (GOT read + TP-relative, no call), and Local Exec (TP-relative constant, fastest); LLD applies **TLS relaxation** to convert GD/LD → IE → LE when the linker can determine final placement.
- **TLSDESC** is an alternative TLS descriptor mechanism that avoids the `__tls_get_addr` overhead by using an indirect call through a resolver that is often a no-op for initial-exec modules.
- **`.eh_frame`** records DWARF CFI for stack unwinding; LLD merges and deduplicates FDE records and generates `.eh_frame_hdr` (a binary-searchable sorted table) for O(log N) unwind frame lookup.
- **`.init_array`/`.fini_array`** replace the legacy `.ctors`/`.dtors` mechanism for C++ global constructor/destructor registration; LLD sorts entries by `init_priority` attribute value, controlling cross-TU initialization order.
