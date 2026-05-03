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

## Linker Relaxation

Linker relaxation is a post-assembly optimization where the linker replaces instruction sequences with shorter equivalents once it knows the final layout of the binary — information unavailable to the assembler. The assembler must emit conservative (worst-case-size) instruction sequences because it does not yet know the final addresses of symbols. It emits relocations alongside those sequences so the linker can revisit the decision later. The linker, having resolved all symbol addresses and determined section placement, applies the "relaxed" (cheaper) form wherever the target is within reach. The result is smaller code, fewer cycles, and no runtime overhead — the optimization is entirely static.

### Why Relaxation Exists

Consider what the assembler knows at translation time: the contents of a single translation unit, a handful of local labels, and the sizes of sections it has emitted. It does not know:

- Whether a called function will end up in the same load segment.
- The final value of `__global_pointer$` (the RISC-V GP register datum).
- Whether a thread-local symbol will land in the executable's initial TLS block or in a dynamically-loaded DSO.
- Whether two sections separated by a forward reference will end up within ±1 MiB of each other.

Faced with this uncertainty, the assembler takes the pessimistic choice: it emits the longest encoding that is guaranteed to work at any distance. For a RISC-V function call that might be far away, that means an `auipc`+`jalr` pair (8 bytes) even if the target turns out to be 200 bytes away. For an x86-64 reference to an extern symbol, that means going through the GOT (two memory accesses) even if the symbol is in the same executable and the offset is known statically.

The linker pays no runtime cost to apply relaxation because the substitution happens before the binary is emitted. Contrast this with two related mechanisms:

- **Compile-time constant folding**: the compiler has all the information at once and does not need a two-phase protocol. Relaxation is the linker-level analogue.
- **Runtime dispatch** (e.g., `ifunc`): the decision is deferred to load or call time and carries actual overhead.

The cost model is favorable. Code-size savings from relaxation are commonly 3–8% on RISC-V binaries compiled with `-mrelax`. On constrained embedded targets, this directly reduces flash usage. On cache-sensitive hot paths, denser code means higher instruction-cache hit rates. For GOT-elimination relaxation on x86-64, each relaxed call removes one memory indirection, saving 4–10 cycles on a cold miss path.

The mechanism requires three cooperating pieces:

1. **Assembler**: emits conservative encodings and tags them with relaxation-capable relocation types (e.g., `R_RISCV_CALL` instead of bare `R_RISCV_JAL`).
2. **Linker**: iterates over the relocations, checks whether relaxation is possible given resolved addresses, rewrites the bytes, and adjusts sizes.
3. **Convergence protocol**: because relaxing an instruction changes section size, which changes subsequent symbol offsets, which may unlock further relaxations, the linker must iterate until a fixpoint.

### RISC-V Relaxation (the most comprehensive implementation)

RISC-V is the canonical example of linker relaxation because the ISA was designed with it in mind from the outset. The RISC-V psABI specifies a rich set of relaxation-capable relocations, and LLD's RISC-V implementation in [`RISCV.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/lld/ELF/Arch/RISCV.cpp) is the most complete relaxation engine in any open-source linker.

#### `R_RISCV_CALL` → `R_RISCV_JAL`: Call Relaxation

A function call to an externally-visible symbol that might be in a distant section compiles to two instructions:

```asm
# Before relaxation (assembler output, 8 bytes):
  auipc   ra, %pcrel_hi(target_func)    # R_RISCV_CALL / R_RISCV_CALL_PLT
  jalr    ra, %pcrel_lo(target_func)(ra)
```

The `auipc` loads the upper 20 bits of the PC-relative offset; `jalr` adds the lower 12 bits and jumps. This covers a ±2 GiB range. Once the linker knows `target_func`'s address, it checks: is `target_func` within ±1 MiB of the call site? If yes, the entire two-instruction sequence relaxes to:

```asm
# After relaxation (4 bytes):
  jal     ra, target_func              # R_RISCV_JAL
  nop                                  # (slot reclaimed; deleted if R_RISCV_ALIGN adjusts)
```

`jal` encodes a 21-bit signed offset (±1 MiB). The relocation pair `R_RISCV_CALL` (on the `auipc`) and an implicit `R_RISCV_CALL_PLT` variant trigger this check in LLD's `relaxCall()` function. If the target is a PLT entry, LLD additionally checks whether the PLT stub itself is within range before relaxing.

The saved 4 bytes are not simply filled with a NOP — LLD adjusts section sizes downward and updates all subsequent offsets, which may enable cascading relaxations elsewhere.

#### GP-Relative Relaxation (`--relax-gp`)

The RISC-V ABI reserves register `x3` (conventionally named `gp`) as the **global pointer**, initialized once by the CRT startup code to the symbol `__global_pointer$`. The linker places `__global_pointer$` at the midpoint of the `.sdata`/`.data` region (typically `__data_start + 0x800`), chosen so that the ±2 KiB reach of a 12-bit signed immediate covers as many globals as possible.

A variable access that might be far from the PC but close to the GP:

```asm
# Before GP-relaxation (auipc + load, 8 bytes):
  auipc   t0, %pcrel_hi(global_var)       # R_RISCV_PCREL_HI20
  lw      a0, %pcrel_lo(global_var)(t0)   # R_RISCV_PCREL_LO12_I
```

When `|global_var - __global_pointer$| < 2048`, LLD rewrites to:

```asm
# After GP-relaxation (single instruction, 4 bytes):
  lw      a0, %gp_rel(global_var)(gp)     # gp-relative 12-bit offset
```

This requires two opt-ins:
- **Assembler**: `-mrelax` (emitted by Clang/GCC by default for RISC-V bare-metal targets).
- **Linker**: `--relax-gp` (opt-in in LLD because GP-relative addressing assumes `gp` is not clobbered by callee code).

LLD's `relaxHi20Lo12()` function handles this case. It computes `sym_addr - gp_value` and checks that the result fits in a signed 12-bit field. For data accesses in the primary `.data`/`.sdata`/`.sbss` sections, this relaxation fires frequently on typical embedded workloads, halving the instruction count for global variable reads and writes.

#### `R_RISCV_ALIGN`: Alignment Relaxation

When the assembler emits a `.align N` directive, it pads the current position to a multiple of `N` bytes — typically with NOP instructions. If relaxation later removes preceding instructions (shrinking the section), those NOPs become wasteful: the alignment is already satisfied, or could be satisfied with fewer NOPs.

The `R_RISCV_ALIGN` relocation encodes the required alignment (and the maximum amount of padding the assembler assumed) so LLD can reduce or eliminate the padding. The relocation is unusual: it does not patch an address, it controls how much of the NOP sequence to keep. LLD's `relaxAlign()` recalculates the alignment requirement after each pass and removes excess NOPs.

This interaction is subtle. Call relaxation shrinks instructions by 4 bytes. If a function was 4-byte aligned with 4 bytes of NOP padding before it, removing a call-site NOP might push the alignment point so that the function's own padding can also be removed. `R_RISCV_ALIGN` makes this chain possible without any approximation.

#### LLD Implementation: Iterative Fixpoint

LLD's top-level relaxation loop in [`RISCV.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/lld/ELF/Arch/RISCV.cpp) drives a `relaxOnce()` function that makes a single pass over all RISC-V input sections:

```
relaxOnce():
  for each InputSection with R_RISCV_RELAX relocations:
    for each relocation pair:
      if relaxCall() succeeds: shrink section by 4, mark changed
      if relaxHi20Lo12() succeeds: shrink section by 4, mark changed
      if relaxGot() succeeds: shrink section by 4, mark changed
      if relaxAlign() succeeds: reduce NOP count, mark changed
  return changed
```

The outer loop in `relaxSections()` calls `relaxOnce()` repeatedly until it returns `false` (no changes). Because each relaxation can only remove bytes (sections never grow), the fixpoint is guaranteed to be reached. In practice, two or three passes suffice for most binaries; pathological cases (a dense graph of near-threshold calls all just under ±1 MiB) may require more.

After the fixpoint, LLD updates all symbol values, section offsets, and non-relaxation relocation addends to reflect the reduced section sizes, then writes the final binary.

### AArch64 Relaxation

AArch64's relaxation is less pervasive than RISC-V's but covers two significant cases: address materialization and GOT elimination for local symbols.

#### `ADRP`+`ADD` → `ADR`

Loading the address of a symbol on AArch64 normally requires two instructions:

```asm
# Two-instruction sequence (8 bytes):
  adrp    x0, :pg_hi21:sym      # load page (4 KiB-aligned) address of sym
  add     x0, x0, :lo12:sym     # add page offset
```

The `adrp` instruction covers ±4 GiB (21-bit signed page offset). When `sym` is within ±1 MiB of the PC (a 21-bit signed byte offset), a single `adr` instruction suffices:

```asm
# Relaxed form (4 bytes):
  adr     x0, sym               # PC + signed 21-bit offset
```

LLD's `relaxAdrpAdd()` in [`AArch64.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/lld/ELF/Arch/AArch64.cpp) checks whether the `R_AARCH64_ADR_PREL_PG_HI21` + `R_AARCH64_ADD_ABS_LO12_NC` relocation pair can be replaced with `R_AARCH64_ADR_PREL_LO21`. If yes, it rewrites the `adrp` opcode to `adr` (the top two bits of the encoding differ) and removes the `add` instruction.

#### GOT → PC-Relative for Protected/Local Symbols

When a DSO references a symbol with `protected` or `hidden` visibility (or when linking a position-independent executable where the symbol is local), going through the GOT is unnecessary. The two-instruction GOT load:

```asm
# GOT-indirect (8 bytes):
  adrp    x0, :got:sym
  ldr     x0, [x0, :got_lo12:sym]
```

can be replaced with the direct PC-relative address computation:

```asm
# Relaxed (8 bytes; same size, but eliminates one memory load):
  adrp    x0, sym
  add     x0, x0, :lo12:sym
```

This relaxation does not shrink the code but eliminates a memory indirection — the GOT load is replaced by an arithmetic operation. LLD performs this for `R_AARCH64_ADR_GOT_PAGE` + `R_AARCH64_LD64_GOT_LO12_NC` pairs when the symbol is known to be non-preemptible.

#### Thunks for Long-Range Branches (Inverse of Relaxation)

AArch64's `B` and `BL` instructions encode a 26-bit signed offset covering ±128 MiB. When a function call target is beyond that range — common in large binaries with aggressive dead-code stripping — the linker must insert a **veneer thunk**: a small trampoline that loads the target address via an indirect branch.

```asm
# AArch64BranchThunk (generated by LLD):
__AArch64AbsLongThunk_target_func:
  ldr     x16, =target_func_abs   # load absolute address
  br      x16                     # indirect branch
.target_func_abs: .quad target_func
```

LLD's `AArch64BranchThunk` class in [`AArch64.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/lld/ELF/Arch/AArch64.cpp) synthesizes these thunks. Thunk insertion *grows* the binary and is therefore not a form of relaxation — it is the opposite transformation, applied when the optimistic (short) form turns out to be out of range.

### x86-64 GOT Relaxation

x86-64's relaxation is focused on GOT elimination for symbols that do not actually require dynamic resolution. The mechanism is encoded in two modern relocation types added to the psABI in 2015.

#### `R_X86_64_GOTPCRELX` and `R_X86_64_REX_GOTPCRELX`

Legacy x86-64 PIC code uses `R_X86_64_GOTPCREL`, which tells the linker to patch an address-of-GOT-entry displacement. This relocation carries no information about the *instruction* that uses it, so the linker cannot safely rewrite the opcode. The newer relocations `R_X86_64_GOTPCRELX` (non-REX instructions) and `R_X86_64_REX_GOTPCRELX` (REX-prefixed instructions) encode the full instruction context — the assembler emits the relocation on the displacement byte of a known instruction pattern — enabling opcode rewriting.

For a reference to a local or hidden symbol, the GOT indirection is unnecessary. LLD's `relaxGot()` in [`X86_64.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/lld/ELF/Arch/X86_64.cpp) rewrites the instruction:

| Original instruction | Relocation | Relaxed instruction | Relaxed relocation |
|---|---|---|---|
| `movq sym@GOTPCREL(%rip), %rax` | `R_X86_64_REX_GOTPCRELX` | `leaq sym(%rip), %rax` | `R_X86_64_PC32` |
| `callq *sym@GOTPCREL(%rip)` | `R_X86_64_GOTPCRELX` | `callq sym` | `R_X86_64_PC32` |
| `jmpq *sym@GOTPCREL(%rip)` | `R_X86_64_GOTPCRELX` | `jmpq sym` | `R_X86_64_PC32` |

The opcode transformations are a simple byte substitution. For the `movq → leaq` case, the ModRM byte changes from `0x8B` (MOV r64, r/m64) to `0x8D` (LEA r64, m). For `callq *` → `callq`, the opcode changes from `0xFF /2` to `0xE8`. For `jmpq *` → `jmpq`, it changes from `0xFF /4` to `0xE9`. These two-byte sequences are compact enough that the instruction does not change size — only the opcode and the displacement are rewritten.

The impact is significant: each extern function call or global variable access in a shared library compiled with `-fpic` goes through the GOT by default. Relaxation eliminates that indirection for symbols that do not need runtime resolution, turning a load + branch-through-pointer into a direct PC-relative branch. This matters most for:

- **Cold code**: the GOT entry may not be in L1 cache; eliminating the load saves a full cache-miss penalty.
- **Shared libraries with `protected` visibility**: symbols marked `protected` are not preemptible; LLD can relax them even in DSO links.
- **Position-independent executables (PIE)**: all symbols in a PIE are non-preemptible; relaxation fires broadly.

On a typical Linux application built with `-fpie`, `R_X86_64_REX_GOTPCRELX` relaxation eliminates GOT lookups for the majority of internal function calls, reducing the GOT section size and improving hot-path instruction density.

### TLS Relaxation Chains

TLS relaxation is the most impactful form because it can eliminate entire function calls (`__tls_get_addr`) that are on the critical path of every TLS variable access. The four TLS models described in §79.3 form a lattice from most general to most specific, and the linker walks down the lattice as far as the final link configuration permits.

#### GD → IE Relaxation

When linking a shared library (`.so`) that uses GD TLS for a variable defined in the main executable, the GD call sequence can be replaced with an IE GOT load. The linker knows the variable is in the executable (it is `BINDING_GLOBAL` with no `DYNAMIC` dependency), so `__tls_get_addr` is not needed — the offset is fixed relative to the thread pointer at load time.

On x86-64, the GD sequence:

```asm
# GD (16 bytes + PLT call overhead):
  leaq    var@tlsgd(%rip), %rdi      # R_X86_64_TLSGD
  callq   __tls_get_addr@plt
  # %rax = &var for this thread
```

becomes the IE sequence:

```asm
# IE (7 bytes, no call):
  movq    var@GOTTPOFF(%rip), %rax   # R_X86_64_GOTTPOFF: load TP offset from GOT
  movq    %fs:(%rax), %rax           # access via thread pointer
```

LLD detects this when it finds a `R_X86_64_TLSGD` relocation in a `.so` link whose target symbol is defined in the executable. The rewrite touches two instruction words: the `leaq`→`movq` substitution and removal of the `callq __tls_get_addr` (replaced with a `nop` or folded into the subsequent access).

#### GD → LE Relaxation

When linking the main executable and the variable is defined in the same executable, the GD sequence collapses entirely to Local Exec:

```asm
# GD → LE (x86-64): 7 bytes, no GOT, no call:
  movq    %fs:0, %rax                # load thread pointer base
  leaq    var@tpoff(%rax), %rax      # add compile-time constant TP offset
```

Or, when the compiler can fold the base load:

```asm
# Single-instruction LE (x86-64):
  movq    %fs:var@tpoff, %rax        # segment-override direct offset
```

The 16-byte `leaq`+`callq` sequence has vanished; only an `fs:`-prefixed memory access remains. LLD applies this in `TlsRelaxation.cpp` by rewriting the TLSGD relocation pair and patching the preceding instruction bytes. The instruction stream change is significant: not only is the call gone, but the register pressure drops (no more spill of `%rdi` to set up the argument) and branch prediction is not disrupted.

#### IE → LE Relaxation

For a symbol defined in the executable with IE TLS, the GOT slot that holds the TP offset is constant — it never changes between threads (only the thread pointer base changes). The IE GOT load:

```asm
# IE (GOT load + TP-relative access):
  movq    var@GOTTPOFF(%rip), %rax   # GOT load of TP offset
  movl    %fs:(%rax), %eax           # access variable
```

relaxes to the LE immediate:

```asm
# LE (immediate TP offset, no GOT):
  movl    %fs:var@tpoff, %eax        # single instruction, constant offset
```

This fires when `R_X86_64_GOTTPOFF` is encountered in an executable (not a `.so`) link and the symbol is local to the executable. The GOT entry is eliminated entirely.

#### AArch64 TLSDESC Relaxation Chain

AArch64's preferred TLS access mechanism is **TLSDESC** (TLS Descriptor), which uses an indirect call through a descriptor pair rather than `__tls_get_addr`. The TLSDESC model is itself already faster than GD, but LLD can further relax it:

**TLSDESC → TLSIE**: when the descriptor's resolver is the standard GOT-based resolver (i.e., the variable is in a known-loaded module), LLD replaces the `adrp`+`ldr`+`add`+`blr` sequence with an `adrp`+`ldr` GOT load. The descriptor pair is replaced with a single GOT slot holding the TP offset.

**TLSIE → TLSLE**: when the variable is in the executable and the offset fits in an immediate, LLD replaces the GOT load with a `movz`/`movk` immediate load of the TP offset, then folds that into a `mrs x0, tpidr_el0; add x0, x0, #tpoff` pair — or, for small offsets, a single `mrs` + load with offset.

```asm
# TLSDESC (AArch64, 4 instructions):
  adrp    x0, :tlsdesc:var
  ldr     x1, [x0, :tlsdesc_lo12:var]
  add     x0, x0, :tlsdesc_lo12:var
  blr     x1

# After TLSIE relaxation (2 instructions):
  adrp    x0, :gottprel:var
  ldr     x0, [x0, :gottprel_lo12:var]   # GOT slot = TP offset

# After TLSLE relaxation (2 instructions, no GOT):
  mrs     x0, tpidr_el0
  add     x0, x0, #var@tpoff             # compile-time constant
```

LLD performs TLSDESC-to-TLSLE relaxation in a single pass when it detects that the symbol is local to the executable, collapsing the 4-instruction+GOT path to 2 instructions.

### Relaxation and Section Layout

#### The Iterative Fixpoint

Relaxation changes section sizes, which changes the positions of all subsequent symbols, which may enable or disable additional relaxations. LLD's relaxation engine must therefore iterate:

```
repeat:
  for each InputSection:
    apply all applicable relaxations (shrink-only)
  recalculate all symbol values and section offsets
until no changes
```

The **shrink-only invariant** — relaxations never grow sections — is critical. It guarantees convergence: each pass either makes progress (removes bytes) or terminates. Because section sizes are non-negative integers and each iteration strictly decreases the total byte count when changes occur, the process terminates in at most `sum_of_all_savings / 4` iterations in the pathological case. In practice, two or three passes suffice.

If relaxation could grow sections (as thunk insertion can), the fixpoint would not be guaranteed — an offset change could enable a new thunk, which grows the section, which pushes another call out of range, requiring another thunk, and so on. LLD handles thunk insertion separately (in a pre-relaxation pass) and does not mix it with the shrink-only relaxation loop.

#### Interaction with `--gc-sections`

Dead-code elimination (`--gc-sections`) removes unreachable sections. This must complete **before** relaxation: relaxation requires stable final section sizes to compute symbol offsets accurately. If a dead section were removed after relaxation, the offsets computed during relaxation would be wrong. LLD's pipeline therefore orders: GC → relaxation → relocation application → output.

#### Interaction with `-z relro` and Page Alignment

Some relaxations interact with page-alignment constraints. The `.got.plt` section under `-z relro -z now` is marked read-only after startup via `PT_GNU_RELRO`. This segment must end on a page boundary, which imposes alignment padding. If GOT relaxation eliminates enough GOT entries to shrink `.got.plt` significantly, LLD adjusts the padding to maintain page alignment. Conversely, a relaxation that would move a protected symbol across a page boundary (breaking `relro` invariants) is suppressed.

Similarly, `-z relro` applies to the `.data.rel.ro` section, which contains constants that require runtime relocations (C++ vtable pointers, for instance). Relaxation that eliminates dynamic relocations for local symbols reduces the size of this section, which can affect the `PT_GNU_RELRO` segment boundary and downstream alignment. LLD recalculates all segment boundaries after each relaxation pass to ensure alignment constraints are satisfied before emitting the final layout.

---

## Chapter Summary

- The **GOT** (Global Offset Table) in `.got` and `.got.plt` stores absolute addresses that are filled at load time by `ld.so`, allowing PIC code in `.text` to remain read-only and sharable; `GOTPCREL` relocations encode load offsets relative to the current PC.
- The **PLT** (Procedure Linkage Table) implements lazy dynamic symbol binding: the first call goes through a stub that invokes `_dl_runtime_resolve`; subsequent calls through the same PLT stub use the GOT entry filled by the resolver, costing only one extra memory indirection.
- **TLS** has four models: General Dynamic (two calls, fully general), Local Dynamic (one call, within-module), Initial Exec (GOT read + TP-relative, no call), and Local Exec (TP-relative constant, fastest); LLD applies **TLS relaxation** to convert GD/LD → IE → LE when the linker can determine final placement.
- **TLSDESC** is an alternative TLS descriptor mechanism that avoids the `__tls_get_addr` overhead by using an indirect call through a resolver that is often a no-op for initial-exec modules.
- **`.eh_frame`** records DWARF CFI for stack unwinding; LLD merges and deduplicates FDE records and generates `.eh_frame_hdr` (a binary-searchable sorted table) for O(log N) unwind frame lookup.
- **`.init_array`/`.fini_array`** replace the legacy `.ctors`/`.dtors` mechanism for C++ global constructor/destructor registration; LLD sorts entries by `init_priority` attribute value, controlling cross-TU initialization order.


---

@copyright jreuben11
