# Chapter 120 — libunwind

*Part XVII — Runtime Libraries*

libunwind is the LLVM implementation of the stack-unwinding ABI: the low-level infrastructure used to walk the call stack frame by frame, reading DWARF CFI (Call Frame Information) or SEH unwind tables to determine how to restore registers and find the return address at each frame. It underpins C++ exception handling, `longjmp`, debugger stack traces, and sanitizer symbolization. This chapter covers the LLVM libunwind implementation (`libunwind/`), the `_Unwind_*` ABI it provides, DWARF CFI internals, SEH integration for Windows, and use in freestanding environments. For the higher-level exception-handling model built on top of libunwind, see [Chapter 26 — Exception Handling](../part-04-llvm-ir/ch26-exception-handling.md) and [Chapter 122 — libc++abi](ch122-libcxxabi.md).

---

## 20.1 The Unwinding ABI: _Unwind_RaiseException

The Itanium C++ ABI defines a language-independent unwinding interface. The key entry point is `_Unwind_RaiseException`:

```
phase 1: search phase — walk frames upward, call personality routines
         until one says _URC_HANDLER_FOUND
phase 2: cleanup phase — walk frames again, call personality routines
         to execute cleanups; stop at the frame that said HANDLER_FOUND
```

The ABI functions, all provided by libunwind:

| Function | Purpose |
|----------|---------|
| `_Unwind_RaiseException` | Begin two-phase unwinding |
| `_Unwind_Resume` | Resume unwinding after a catch that re-throws |
| `_Unwind_ForcedUnwind` | Force unwinding (used by `pthread_cancel`) |
| `_Unwind_GetIP` | Get instruction pointer at current frame |
| `_Unwind_SetIP` | Set instruction pointer (redirect execution) |
| `_Unwind_GetRegionStart` | Get start of the current try region |
| `_Unwind_GetLanguageSpecificData` | Pointer to `.gcc_except_table` LSDA |
| `_Unwind_GetCFA` | Get canonical frame address |
| `_Unwind_SetGR` | Set a general-purpose register in the context |
| `_Unwind_GetGR` | Get a general-purpose register from context |
| `_Unwind_Resume_or_Rethrow` | Resume or rethrow (for foreign exceptions) |
| `_Unwind_DeleteException` | Free an exception object |
| `_Unwind_Backtrace` | Walk stack, invoke callback per frame |

Source: [libunwind.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/libunwind/include/libunwind.h)

---

## 20.2 LLVM libunwind Architecture

### 20.2.1 Repository Layout

```
libunwind/
├── include/
│   ├── libunwind.h        # Public API
│   ├── unwind.h           # Compatibility header (maps to libunwind.h)
│   └── mach-o/            # macOS compact unwind records
├── src/
│   ├── UnwindCursor.hpp   # Core unwinding logic
│   ├── UnwindRegisters.S  # Architecture-specific register save/restore
│   ├── DwarfParser.hpp    # CIE/FDE parser
│   ├── CompactUnwinder.hpp # macOS compact unwind decoder
│   ├── Registers.hpp      # Per-arch register context
│   ├── AddressSpace.hpp   # Memory access abstraction
│   ├── libunwind.cpp      # C ABI entry points
│   └── Unwind-EHABI.cpp   # ARM EHABI unwinding
└── cmake/
```

### 20.2.2 UnwindCursor

The central abstraction is `UnwindCursor<A, R>`, templated on `AddressSpace A` and register set `R`. It maintains the current unwinding context (all register values at the current frame) and knows how to step one frame:

```cpp
// Simplified from UnwindCursor.hpp
template <typename A, typename R>
class UnwindCursor {
public:
    // Step to caller frame using DWARF CFI or compact unwind
    int step();
    // Get/set registers in the current frame context
    pint_t getReg(int regNum);
    void   setReg(int regNum, pint_t value);
    // Get address of language-specific data area
    pint_t getLSDA();
    // Get the personality routine pointer
    pint_t getPersonalityRoutine();
private:
    R             _registers;         // Current register context
    A&            _addressSpace;      // Memory accessor
    UnwindInfoSections _info;         // CFI section pointers
    DwarfFDE      _currentFDE;        // Parsed FDE for this frame
};
```

Source: [UnwindCursor.hpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/libunwind/src/UnwindCursor.hpp)

The `step()` method:

1. Locates the FDE (Frame Description Entry) for the current PC using a binary search over the `PT_GNU_EH_FRAME` segment.
2. Parses the CIE (Common Information Entry) to get the return address column and data/code alignment factors.
3. Interprets the DWARF CFI instructions in the FDE to reconstruct the caller's register state.
4. Updates `_registers` with the restored state.

---

## 20.3 DWARF Call Frame Information

### 20.3.1 CFI Structure

DWARF CFI lives in the `.eh_frame` section (ELF) or `__eh_frame` section (Mach-O). It consists of CIEs followed by FDEs:

```
.eh_frame:
  CIE:
    length         4 bytes
    CIE_id         4 bytes (0 for CIE)
    version        1 byte
    augmentation   string (e.g., "zR" for pointer encoding)
    code_align     ULEB128 (instruction size: 4 for AArch64)
    data_align     SLEB128 (CFA offset step: -8 on x86_64)
    return_addr_col ULEB128 (which register holds return address)
    instructions   CFI opcodes

  FDE:
    length         4 bytes
    CIE_pointer    4 bytes (offset back to CIE)
    initial_loc    pointer (PC range start)
    address_range  pointer size (range length)
    instructions   CFI opcodes
```

### 20.3.2 CFI Opcodes

The CFI opcode set defines a virtual machine for expressing how to compute the CFA (Canonical Frame Address) and the saved value of each register at any given PC offset:

| Opcode | Effect |
|--------|--------|
| `DW_CFA_def_cfa reg, offset` | CFA = reg + offset |
| `DW_CFA_def_cfa_offset offset` | CFA offset (same reg) |
| `DW_CFA_def_cfa_register reg` | CFA register (same offset) |
| `DW_CFA_offset col, factored_offset` | Reg saved at CFA + offset |
| `DW_CFA_restore col` | Restore reg to initial rule |
| `DW_CFA_undefined col` | Register has no value |
| `DW_CFA_same_value col` | Register unchanged from caller |
| `DW_CFA_register col1, col2` | col1 is in col2 (copy) |
| `DW_CFA_advance_loc delta` | Advance PC by delta*code_align |
| `DW_CFA_remember_state` | Push state onto stack |
| `DW_CFA_restore_state` | Pop state from stack |

### 20.3.3 Example: x86_64 Function Prologue

A typical x86_64 function prologue generates the following CFI annotations (as seen in `.s` output):

```asm
foo:
    .cfi_startproc
    pushq   %rbp
    .cfi_def_cfa_offset 16     ; CFA = RSP + 16
    .cfi_offset rbp, -16       ; RBP saved at CFA-16
    movq    %rsp, %rbp
    .cfi_def_cfa_register rbp  ; CFA = RBP + 16 (now RBP-relative)
    subq    $32, %rsp
    ; ... function body ...
    movq    %rbp, %rsp
    popq    %rbp
    .cfi_def_cfa rsp, 8        ; CFA = RSP + 8 (pre-push state)
    retq
    .cfi_endproc
```

The DWARF parser in `DwarfParser.hpp` interprets these annotations at runtime to reconstruct the caller's `%rbp` and return address when unwinding through this frame.

---

## 20.4 Register Contexts Per Architecture

libunwind provides architecture-specific register context classes in `Registers.hpp`:

### 20.4.1 x86_64

```cpp
class Registers_x86_64 {
    uint64_t _registers[21]; // RAX..RIP, plus SSE regs
    v128 _xmm[16];           // SSE/AVX-128 state
public:
    static int lastDwarfRegNum() { return 66; }
    uint64_t getRegister(int regNum) const;
    void     setRegister(int regNum, uint64_t value);
    double   getFloatRegister(int regNum) const;
    void     jumpto(); // execute register context (longjmp-style)
};
```

The `jumpto()` method is implemented in assembly (`UnwindRegisters.S`) and restores all callee-saved registers then jumps to the saved RIP — this is the low-level mechanism that implements the `catch` clause landing.

### 20.4.2 AArch64

```cpp
class Registers_arm64 {
    uint64_t _registers[32]; // x0..x30, sp
    double   _vectorHalfRegisters[32]; // d0..d31
public:
    uint64_t getRegister(int regNum) const;
    void     setRegister(int regNum, uint64_t value);
    void     jumpto();
    static bool validVectorRegister(int regNum);
    v128     getVectorRegister(int regNum) const;
};
```

For AArch64, DWARF register numbers 0-30 map to x0-x30, 31 to SP, and 64-95 to v0-v31 (the full 128-bit NEON registers).

### 20.4.3 ARM EHABI

32-bit ARM uses a different unwinding ABI: ARM EHABI (Embedded ABI). Instead of DWARF `.eh_frame`, it uses `.ARM.exidx` and `.ARM.extab` sections with a compact 32-bit instruction encoding:

```
.ARM.exidx:
  [function_address_offset] [unwind_data]
  ; If bit 31 of unwind_data set: EXIDX_CANTUNWIND
  ; If bits 31:24 == 0x80: compact model 0 (16 ops inline)
  ; Otherwise: pointer to .ARM.extab
```

The ARM EHABI defines three compact unwind personalities:

| Personality | Model | Description |
|-------------|-------|-------------|
| `__aeabi_unwind_cpp_pr0` | Su16 | Short frame: inline 16-bit opcodes |
| `__aeabi_unwind_cpp_pr1` | Lu16 | Long frame: 8-bit opcode stream, vsp update |
| `__aeabi_unwind_cpp_pr2` | Lu32 | Long frame: 32-bit opcode stream for Cortex-M |

The compact model 0 encoding packs up to four 8-bit unwind opcodes in the inline `unwind_data` word. The opcode set:

| Opcode | Bits | Action |
|--------|------|--------|
| `00nnnnnn` | 6 bits | vsp += 4*(n+1) (pop words from stack) |
| `01nnnnnn` | 6 bits | vsp -= 4*(n+1) (move stack pointer up) |
| `1000iiii iiiiiiii` | 16 bits | Pop r4-r15 per bitmask |
| `10010000`..`10011101` | 8 bits | Set vsp = reg (r0-r15, minus sp) |
| `10101nnn` | 3 bits | Pop r4-r(4+n), pop r14 |
| `10110001 0000iiii` | 12 bits | Pop r0-r3 per bitmask |

The ARM EHABI support is in `Unwind-EHABI.cpp` and `UnwindCursor.hpp` (ARM specializations). This path is taken when targeting `arm-*-eabi` or `thumb-*-eabi` triples.

### 20.4.4 .eh_frame_hdr and Binary Search Index

Locating the correct FDE for a given PC requires searching the `.eh_frame` section. A linear scan is O(n) in the number of functions. To speed this up, the linker generates a `.eh_frame_hdr` section containing a sorted array of `(initial_pc, fde_offset)` pairs that supports O(log n) binary search:

```
.eh_frame_hdr:
  version       = 1
  eh_frame_ptr_enc = DW_EH_PE_pcrel | DW_EH_PE_sdata4
  fde_count_enc = DW_EH_PE_udata4
  table_enc     = DW_EH_PE_datarel | DW_EH_PE_sdata4
  eh_frame_ptr  (pointer to .eh_frame)
  fde_count     (number of FDE entries)
  binary_search_table:
    [initial_pc_offset, fde_ptr_offset]*  (sorted by initial_pc)
```

The kernel maps `.eh_frame_hdr` via the `PT_GNU_EH_FRAME` program header segment. libunwind's `DwarfFDECache::findFDE` uses `dl_iterate_phdr` to find all loaded objects' `PT_GNU_EH_FRAME` segments, then binary-searches within each.

### 20.4.5 Signal Frame Handling

Signal handlers are invoked with a modified stack layout — the return from a signal handler does not follow normal call conventions. The CIE for signal frames uses augmentation character `'S'` to mark them:

```
CIE augmentation: "zRS"
  'z' = has augmentation data length
  'R' = FDE pointer encoding follows
  'S' = this is a signal frame
```

When libunwind encounters a signal frame (identified by the `'S'` augmentation), it adjusts the PC correction: normally it subtracts 1 from the saved return address (to point inside the call instruction) to find the correct FDE. For signal frames the return address is already at the instruction that was executing, so no adjustment is made.

---

## 20.5 SEH Unwinding on Windows

### 20.5.1 Structured Exception Handling

On Windows, stack unwinding uses SEH (Structured Exception Handling) rather than DWARF. The unwind tables are stored in `.pdata` and `.xdata` sections:

```
.pdata section:
  RUNTIME_FUNCTION:
    BeginAddress   DWORD  (relative VA of function start)
    EndAddress     DWORD  (relative VA of function end)
    UnwindData     DWORD  (relative VA of UNWIND_INFO in .xdata)

.xdata section:
  UNWIND_INFO:
    Version:3, Flags:5, SizeOfProlog:8, CountOfCodes:8
    FrameRegister:4, FrameOffset:4
    UnwindCode[]
    ExceptionHandler (optional)
    ExceptionData    (optional)
```

### 20.5.2 libunwind SEH Implementation

LLVM libunwind on Windows implements both the DWARF path (for MinGW-built code) and the SEH path (for MSVC-ABI code). The SEH path calls into `RtlVirtualUnwind` / `RtlLookupFunctionEntry` from the Windows kernel, delegating the actual stack frame computation to the OS:

```cpp
// UnwindCursor.hpp, Windows specialization
bool getInfoFromSEH(pint_t pc) {
    ULONG64 imageBase;
    PRUNTIME_FUNCTION rf = RtlLookupFunctionEntry(pc, &imageBase, NULL);
    if (rf == NULL) return false; // leaf frame
    _info.unwindFunctionStart = imageBase + rf->BeginAddress;
    _info.xdata = imageBase + rf->UnwindData;
    return true;
}
```

LLVM-compiled Windows code that uses `-fexceptions` ends up using DWARF-style CFI even on Windows (when compiled with clang's defaults), because LLVM's EH lowering generates `.eh_frame`. The SEH path is primarily for interoperability with MSVC-compiled code.

---

## 20.6 macOS Compact Unwind

macOS linkers produce a compact unwind format in the `__compact_unwind` section (Mach-O), which is a space-efficient alternative to DWARF CFI. The compact format encodes standard prologues in a 32-bit integer:

```
Compact Unwind Encoding (x86_64, mode = RBP frame):
  bits 31-24: mode = UNWIND_X86_64_MODE_RBP_FRAME (0x01)
  bits 18-16: frame offset (rbp offset from CFA)
  bits 14-0:  saved register bitmask (which of rbx,r12..r15 saved)
```

When the linker cannot represent the unwind info compactly, it falls back to DWARF. libunwind's `CompactUnwinder.hpp` handles both formats, preferring compact when available.

---

## 20.7 Backtrace API

libunwind provides a non-standard but widely used backtrace interface:

```cpp
#include <libunwind.h>

void print_backtrace() {
    unw_context_t ctx;
    unw_cursor_t  cursor;
    unw_getcontext(&ctx);
    unw_init_local(&cursor, &ctx);

    while (unw_step(&cursor) > 0) {
        unw_word_t ip, sp, off;
        unw_get_reg(&cursor, UNW_REG_IP, &ip);
        unw_get_reg(&cursor, UNW_REG_SP, &sp);
        char name[256];
        if (unw_get_proc_name(&cursor, name, sizeof(name), &off) == 0)
            printf("  0x%lx  %s+0x%lx (sp=0x%lx)\n",
                   (long)ip, name, (long)off, (long)sp);
        else
            printf("  0x%lx  <unknown>\n", (long)ip);
    }
}
```

`unw_init_local` captures the current machine state; `unw_step` advances one frame, returning 0 at the bottom of the stack. `unw_get_proc_name` resolves the IP to a symbol using `dladdr` (or `SymFromAddr` on Windows).

---

## 20.8 Bare-Metal and Freestanding libunwind

### 20.8.1 Disabling Dynamic Lookup

In a freestanding environment there is no dynamic linker, so `dl_iterate_phdr` (used to find `.eh_frame` sections) is unavailable. libunwind can be built with `LIBUNWIND_ENABLE_STATIC_UNWIND_TABLES=ON`, in which case the application links a pre-computed list of `.eh_frame` section addresses:

```cpp
// Application provides this registration function
extern "C" void __register_frame(const void* fde);
extern "C" void __deregister_frame(const void* fde);

// At startup, register all .eh_frame sections:
__register_frame(__start_eh_frame);
```

### 20.8.2 OS-Less Environments

For targets with no OS at all (embedded RTOS, bare-metal), libunwind can be configured to use a statically linked, application-supplied address space resolver:

```cmake
set(LIBUNWIND_ENABLE_THREADS OFF)
set(LIBUNWIND_ENABLE_SHARED OFF)
set(LIBUNWIND_USE_COMPILER_RT ON)
set(CMAKE_C_FLAGS "-ffreestanding -nostdlib")
```

This produces a static `libunwind.a` with no pthread or libc dependencies, suitable for use in firmware.

---

## 20.9 _Unwind_Backtrace and Sanitizers

The sanitizer runtimes in compiler-rt use `_Unwind_Backtrace` for stack capture during error reporting:

```cpp
// Simplified from sanitizer_common/sanitizer_stacktrace_libcdep.cpp
struct BacktraceData {
    StackTrace *stack;
    uptr max_depth;
};

static _Unwind_Reason_Code UnwindCallback(
    struct _Unwind_Context *ctx, void *arg) {
    auto *d = (BacktraceData*)arg;
    uptr ip = _Unwind_GetIP(ctx);
    if (ip == 0 || d->stack->size >= d->max_depth)
        return _URC_END_OF_STACK;
    d->stack->trace_buffer[d->stack->size++] = ip;
    return _URC_NO_REASON;
}

void SlowUnwindStack(uptr *trace, uptr max_depth) {
    BacktraceData data = {trace, max_depth};
    _Unwind_Backtrace(UnwindCallback, &data);
}
```

The "fast" unwind path used in the hot path (during every malloc/free for HeapSan) bypasses libunwind entirely and reads frame pointers directly from the stack, requiring `-fno-omit-frame-pointer` compilation. See [Chapter 112](../part-16-jit-sanitizers/ch112-production-allocators-scudo-gwp-asan.md) for the Scudo/GWP-ASan fast-unwind implementation.

---

## 20.10 Leaf Frames and No-Unwind Annotations

### 20.10.1 Leaf Frames

A leaf frame is a function that makes no other calls (or only tail calls). On many architectures, leaf frames do not save the return address to the stack — it remains in the link register (AArch64: LR, ARM: LR, RISC-V: ra). This means the leaf frame has no CFI instructions: the unwinder cannot step through it using DWARF.

libunwind handles leaf frames via a special EXIDX entry on ARM (`EXIDX_CANTUNWIND`) or by checking whether the FDE lookup returns NULL on other architectures. When the unwinder cannot find a FDE for the current PC, it tries to recover using the frame pointer (if `LIBUNWIND_ALLOW_FP_BASED_UNWINDING` is enabled) or stops with `_URC_END_OF_STACK`.

On AArch64 with pointer authentication (`-mbranch-protection=pac-ret`), the return address in LR is signed with PAC. libunwind must strip the PAC before using the return address as a code pointer, which it does by calling the ABI-defined `ptrauth_strip` operation.

### 20.10.2 -fno-exceptions and -fno-unwind-tables

When compiling with `-fno-exceptions`, Clang does not emit `.eh_frame` CFI entries, saving code size. This means `_Unwind_Backtrace` cannot walk through such frames. Functions compiled with `-fno-exceptions` appear as opaque "black box" frames in sanitizer stack traces.

The mitigation is to compile with `-fasynchronous-unwind-tables` (which emits CFI even when exceptions are disabled) or to rely on the frame-pointer chain (available with `-fno-omit-frame-pointer`).

Compiling the full codebase with `-funwind-tables` (synchronous only) or `-fasynchronous-unwind-tables` (for signal handlers) controls the tradeoff between code size and unwindability:

| Flag | Effect |
|------|--------|
| `-fno-unwind-tables` | No CFI; smallest code |
| `-funwind-tables` | CFI for exceptions/backtraces but not signal handlers |
| `-fasynchronous-unwind-tables` | Full CFI; required for `perf` and `gdb` stack traces |

---

## 20.11 Relationship to libc++abi

libunwind provides the `_Unwind_*` ABI; libc++abi provides the higher-level `__cxa_*` ABI on top of it. The call chain for a C++ `throw`:

```
__cxa_throw (libc++abi)         – allocates exception, calls personality
  └─ _Unwind_RaiseException (libunwind) – walks frames
       └─ personality (libc++abi: __gxx_personality_v0)
            └─ _Unwind_SetIP → lands in catch block
```

Both libraries must be built with compatible configurations. When building the LLVM runtimes, both are enabled together:

```bash
cmake -DLLVM_ENABLE_RUNTIMES="libunwind;libcxxabi;libcxx" ...
```

See [Chapter 122 — libc++abi](ch122-libcxxabi.md) for the personality function implementation and `__cxa_*` ABI details.

---

## Chapter Summary

- libunwind implements the Itanium `_Unwind_*` ABI — the language-independent stack-walking interface used by C++ exceptions, `pthread_cancel`, and debugger backtraces.
- The core abstraction is `UnwindCursor<A,R>`, which interprets DWARF CFI opcodes to reconstruct caller register state one frame at a time.
- DWARF CFI is encoded in `.eh_frame` as CIEs (per function type) and FDEs (per function), containing a bytecode VM that describes how to compute the CFA and restore saved registers at any PC.
- Architecture-specific `Registers_<arch>` classes and `UnwindRegisters.S` provide the register save/restore and `jumpto()` trampoline for each supported target.
- 32-bit ARM uses the ARM EHABI (`.ARM.exidx`/`.ARM.extab`) instead of DWARF; macOS uses compact unwind records in `__compact_unwind`.
- On Windows, the SEH path delegates to `RtlVirtualUnwind`; LLVM-compiled code can use either SEH or DWARF unwind tables depending on the ABI target.
- Bare-metal use requires `LIBUNWIND_ENABLE_STATIC_UNWIND_TABLES=ON` and manual `__register_frame` calls since `dl_iterate_phdr` is unavailable.
- Sanitizer runtimes use `_Unwind_Backtrace` for slow stack capture and bypass libunwind with direct frame-pointer reads for the fast path.
- libunwind is a dependency of libc++abi; the two must be built and deployed together for correct C++ exception semantics.


---

@copyright jreuben11
