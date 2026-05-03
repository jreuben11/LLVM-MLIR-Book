# Ch200 — Linux Kernel Compilation with LLVM/Clang

*Part XXIX — Compiler Tooling, Kernel Integration, and Binary Analysis*

The Linux kernel shipped for decades with GCC as its only supported compiler. That changed gradually and then all at once: Android mandated Clang for all kernels beginning with Android 12, ChromeOS moved to Clang for security features GCC could not provide, and major cloud providers followed to gain ThinLTO, kernel CFI, and superior diagnostics. Today, building a mainline kernel with the full LLVM toolchain — Clang, LLD, llvm-ar, llvm-objcopy, and the integrated assembler — is the default path on arm64 for Android and ChromeOS, and a first-class option on x86_64 upstream. This chapter explains the mechanics of that integration in depth: the build flags and why each one exists, the two CFI models the kernel supports, ThinLTO's per-module workflow, BPF compilation with CO-RE, the kernel's Clang-backed sanitizers, and the `objtool` binary checker that validates compiler output before the kernel ships.

---

## 1. The ClangBuiltLinux Project

### 1.1 History and Motivation

The ClangBuiltLinux effort began as an informal collaboration around 2015, accelerated by Google's Android kernel team, and reached a stable state where Clang could build a bootable x86_64 kernel by 2018. The technical motivations were precise:

- **Better diagnostics.** Clang's error messages and `-Weverything` granularity caught latent bugs that GCC silently ignored, particularly around pointer aliasing and uninitialized variable analysis.
- **LTO at scale.** GCC's LTO infrastructure imposed a whole-program IR dump incompatible with the kernel's distributed build model. LLVM's ThinLTO summary index enabled per-module bitcode with thin cross-module inlining, fitting naturally into `make -j`.
- **CFI.** LLVM's control-flow integrity passes (`-fsanitize=cfi-icall`, later `-fsanitize=kcfi`) have no GCC equivalent. Android's security requirements mandated CFI on function pointers, which exist pervasively in the kernel's driver model.
- **Modern toolchain consolidation.** Replacing GNU binutils (as, ar, nm, objcopy, objdump, strip) with their LLVM counterparts removes a secondary toolchain dependency, enables faster builds via the integrated assembler, and ensures consistent flag semantics.

### 1.2 Kernel Tree Support

The kernel tracks the minimum Clang version it requires in
[`Documentation/process/changes.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/process/changes.rst)
and in `scripts/min-tool-version.sh`. As of Linux 6.10, the minimum is Clang 16; LLVM 22 is fully supported. The build infrastructure lives in:

- `scripts/Makefile.clang` — Clang-specific flags injected via `CC=clang`
- `scripts/clang-tools/` — `clang-tidy` and `clang-format` integration for `make clang-tidy`
- `scripts/lld-version.sh` — version check ensuring ld.lld meets the minimum
- `Documentation/kbuild/llvm.rst` — the authoritative kernel documentation for LLVM builds

Android's `android/toolchain/` tree pins a specific `clang-r<revision>` snapshot (e.g., `clang-r536225`) so that kernel builds are hermetic and reproducible across Android versions.

### 1.3 The `LLVM=1` Build Variable

The single most important build variable is `LLVM=1`. Passing it to `make` sets all compiler and binutils variables in one shot:

```makefile
# Effective result of LLVM=1 in scripts/Makefile.clang
CC        := clang
LD        := ld.lld
AR        := llvm-ar
NM        := llvm-nm
OBJCOPY   := llvm-objcopy
OBJDUMP   := llvm-objdump
READELF   := llvm-readelf
STRIP     := llvm-strip
```

Without `LLVM=1`, you can set individual tools: `CC=clang LD=ld.lld` while retaining GNU `ar` and `nm`. This is the compatibility mode used when the host has LLVM Clang but no full LLVM binutils installation.

`LLVM_IAS=1` enables Clang's integrated assembler, replacing the GNU `as` invocation for inline assembly and hand-written `.S` files. This is now the default on most architectures when `CC=clang`; to force GNU as, pass `LLVM_IAS=0` explicitly. The integrated assembler enforces stricter directive compatibility: `.macro` behavior, `.rept` nesting, and some architecture-specific directives that GNU as accepts with extensions may require adjustment for `LLVM_IAS=1`.

---

## 2. Kernel-Specific Clang Flags

The kernel's Makefile passes a set of flags that differ substantially from user-space compilation defaults. Each has a specific reason rooted in kernel semantics.

### 2.1 Type Aliasing and Null Pointer Handling

**`-fno-strict-aliasing`**

Standard C aliasing rules allow the compiler to assume that pointers of different types never alias. The kernel violates this pervasively through the `container_of` macro, which casts a member pointer back to the enclosing struct type:

```c
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))
```

Without `-fno-strict-aliasing`, Clang's TBAA (Type-Based Alias Analysis) pass may reorder or eliminate loads and stores around such casts. The kernel opts out entirely rather than attempting to annotate every such site with `__may_alias__`.

**`-fno-delete-null-pointer-checks`**

Standard C defines dereferencing a null pointer as undefined behavior, allowing the optimizer to assume it never happens and to delete code guarding against it. The kernel deliberately maps physical address 0 as valid memory on some architectures (particularly older x86 and embedded targets), and the kernel also has long-standing patterns like checking `if (ptr)` after use. Without this flag, Clang may eliminate the guard, creating a real security vulnerability.

### 2.2 Linkage and Stack Security

**`-fno-common`**

Without this flag, a C translation unit with `int foo;` at file scope produces a COMMON symbol — a tentative definition that the linker merges with others of the same name. This hides duplicate global definitions. `-fno-common` turns every tentative definition into a strong definition, causing a linker error on duplicates. The kernel uses this to enforce clean global namespacing across thousands of translation units. Note that GCC 10+ also defaults to `-fno-common`, so this flag's behavior is now consistent across compilers.

**`-fstack-protector-strong`**

This inserts a stack canary check in functions that have arrays, address-taken local variables, or other patterns that risk stack buffer overflow. The `-strong` variant (vs. `-all`) applies the canary only where static analysis determines the function is at risk, avoiding the overhead of protecting every function. The kernel supplies `__stack_chk_guard` and `__stack_chk_fail` in `arch/x86/kernel/stackprotector.c` (x86) and equivalent files per architecture.

### 2.3 Floating-Point and Position

**`-msoft-float` / `-mno-fp-ret-in-387` (x86)**

The kernel does not use FPU registers in kernel context. Using them would require saving and restoring the full FPU state on every kernel entry, unacceptably expensive. `-msoft-float` prevents the compiler from emitting FPU instructions. `-mno-fp-ret-in-387` prevents returning floating-point values in x87 registers. On arm64, the equivalent is `-mgeneral-regs-only`, which prohibits NEON and FP register use outside of explicitly designated sections.

**`-fno-pic` / `-fno-pie`**

The kernel's virtual address layout is fixed at compile time for most configurations. Position-independent code (PIC/PIE) requires a GOT (Global Offset Table) for relocations, which the kernel neither needs nor wants: it resolves its own relocations at boot via `arch/x86/boot/compressed/kaslr.c` and the early linker scripts. When `CONFIG_RANDOMIZE_BASE` (KASLR) is enabled, the kernel builds with `CFLAGS_KERNEL += -fno-pie` anyway because KASLR is handled at load time, not compile time — the kernel binary is position-specific, the boot loader places it.

### 2.4 Dead Code Elimination and Warning Suppression

**`-ffunction-sections` / `-fdata-sections` with `--gc-sections`**

Controlled by `CONFIG_LD_DEAD_CODE_DATA_ELIMINATION`, these flags place each function and data item in its own linker section. The linker then discards unreferenced sections via `--gc-sections`. This can shrink kernel images by several percent on constrained embedded targets and also enables finer-grained LTO cross-reference analysis.

**`-Wno-address-of-packed-member`**

The kernel extensively uses `__attribute__((packed))` on protocol headers and hardware register structures. Taking the address of a member of a packed struct may produce an unaligned pointer. Clang warns about this; the kernel suppresses the warning because the patterns are intentional and hand-audited.

**`-Wno-gnu`**

GCC extensions like zero-length arrays (`struct s { char data[0]; }`), statement expressions (`({ ... })`), and `__attribute__` syntax are used throughout the kernel's compatibility layer. Clang accepts these in non-strict mode but can warn about them; `-Wno-gnu` suppresses the entire category.

---

## 3. Clang CFI for the Kernel

Control-flow integrity protects indirect calls — function pointers — from being redirected to arbitrary code by an attacker who has achieved limited write primitives. The kernel has two CFI mechanisms, with different trade-offs.

### 3.1 kCFI: Per-Module CFI Without LTO

`kCFI` is enabled with `-fsanitize=kcfi`. It was designed specifically for the kernel to provide CFI without requiring whole-program LTO, which is incompatible with loadable kernel modules. The mechanism:

1. At each call site with an indirect call through a function pointer of type `T`, Clang inserts a `__kcfi_typeid` hash check inline before the call instruction.
2. Each function definition emits a `__kcfi_typeid` value in the four bytes immediately preceding the function prologue.
3. At the call site, Clang computes the hash of the expected type and compares it against the callee's embedded typeid via a load from `callee_addr - 4`.
4. A mismatch traps via an architecture-specific mechanism (on arm64, a `brk` instruction; on x86, an `ud2`).

The type hash is derived from the function's parameter and return types. This is intentionally not a cryptographic hash — it is a fast collision-resistant fingerprint. The design means two functions with the same signature hash to the same typeid, which is the same type-based granularity as `-fsanitize=cfi-icall`.

kCFI became available upstream in Linux 6.1 and was enabled by default on arm64 in Linux 6.2. It requires Clang; GCC has no equivalent. The kernel flag is `CONFIG_CFI_CLANG` with `CONFIG_LTO_CLANG` not required.

The `__nocfi` attribute (defined in `include/linux/compiler_types.h`) disables kCFI checks for a specific function. It is applied to interrupt and exception entry paths where the check overhead or the calling convention deviates from what Clang expects:

```c
/* include/linux/compiler_types.h */
#define __nocfi  __attribute__((__no_sanitize__("kcfi")))
```

Functions marked `__nocfi` still emit a valid `__kcfi_typeid` in their prologue (so they can be called through checked pointers), but incoming indirect calls to them bypass the check.

### 3.2 CFI-icall with ThinLTO

`-fsanitize=cfi-icall` is the stricter user-space CFI mode, usable in the kernel only when ThinLTO is active (`CONFIG_LTO_CLANG_THIN`). Unlike kCFI, it uses the `LowerTypeTests` LLVM pass to replace each indirect call with a range check against a jump table, and the type sets are resolved globally across the entire linked binary.

The `LowerTypeTests` pass implementation lives in
[`llvm/lib/Transforms/IPO/LowerTypeTests.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/IPO/LowerTypeTests.cpp).
It operates on `llvm.type.test` intrinsics inserted by Clang for each indirect call site, replacing them with a compact bit-set test. The result is smaller and faster than kCFI's per-callee prologue hash, but requires LTO to construct the type sets.

For the kernel, the implication is that `cfi-icall` protections cover the kernel image but not loadable modules by default — modules have separate type sets and must be linked with LTO themselves. Android's GKI addresses this by building the kernel with monolithic LTO or ThinLTO from the start.

### 3.3 The `noinstr` Annotation

Low-level interrupt and exception handlers must not be instrumented — any instrumentation that accesses per-CPU variables, calls out to tracing infrastructure, or inserts patchable NOPs can cause recursive exceptions or corruption if it runs before the CPU state is fully initialized. The kernel marks these functions `noinstr` (defined as `__attribute__((__section__(".noinstr.text"), __noinline__))`), and `objtool` verifies at build time that no instrumented calls escape from `.noinstr.text` sections. kCFI respects `noinstr` by also applying `__nocfi` semantics to functions in that section.

---

## 4. ThinLTO for the Kernel

### 4.1 Configuration and Workflow

ThinLTO is enabled via `CONFIG_LTO_CLANG_THIN=y`. The build workflow changes in two places:

1. **Compilation phase**: Each translation unit is compiled to bitcode (`.bc`) rather than native object code, via `-flto=thin` injected by the kbuild system. Kbuild stores these as `.o` files that happen to contain LLVM bitcode, maintaining compatibility with the rest of the build system.
2. **Link phase**: `ld.lld` receives all bitcode `.o` files and performs the ThinLTO link: it reads the summary index from each module, decides which functions to import for cross-module inlining, performs thin-link optimization on each module in parallel, then emits native code and links the final binary.

```bash
# Effective compilation command with ThinLTO
clang -flto=thin -fvisibility=hidden -c kernel/sched/core.c -o kernel/sched/core.o

# Effective link command (simplified)
ld.lld --lto=thin --thinlto-jobs=$(nproc) \
    -o vmlinux vmlinux.a arch/x86/kernel/head64.o ...
```

The `--thinlto-jobs` flag parallelizes the per-module codegen phase across available cores. On a 64-core machine, ThinLTO link times for a full kernel are typically 3–5 minutes, versus 15–20 minutes for monolithic LTO.

### 4.2 Interaction with CONFIG_MODVERSIONS

`CONFIG_MODVERSIONS` exports a CRC of each kernel symbol's type signature, allowing the module loader to detect ABI mismatches. With LTO, symbol types are fully resolved at link time — but `modversions` requires the type information at object file creation time, before linking. The kbuild system resolves this by running a two-pass scheme: a first link pass extracts the symbol versions from the partially-linked vmlinux, and a second pass incorporates them. This is implemented in `scripts/mod/modpost.c` and the `vmlinux.export.h` generation step.

### 4.3 Verifying ThinLTO Output

After a ThinLTO kernel build, you can verify that hot functions have been inlined cross-module:

```bash
# Check that scheduler fast path is in .text (not a module boundary stub)
llvm-readelf --sections vmlinux | grep '\.text'

# Confirm cross-module inlining occurred (function from sched/core.c
# appears inlined into kernel/fork.c's text range)
llvm-objdump --syms vmlinux | grep copy_process
llvm-addr2line -e vmlinux <address_of_interest>
```

See [Ch77 — LTO and ThinLTO](../part-13-lto-whole-program/ch77-lto-and-thinlto.md) for the detailed ThinLTO summary index format and the mechanics of thin cross-module inlining.

---

## 5. BPF Program Compilation

### 5.1 The BPF Target Triple

Clang's BPF backend compiles C programs into BPF bytecode that runs inside the Linux kernel's verifier-checked sandbox. The target triple is one of:

| Triple | Meaning |
|---|---|
| `bpf` | Host-endian (auto-detected) |
| `bpfeb` | Big-endian BPF |
| `bpfel` | Little-endian BPF |

On x86_64 hosts, `bpf` resolves to `bpfel`. The BPF backend source lives in
[`llvm/lib/Target/BPF/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/BPF/).

A minimal BPF program compilation:

```bash
clang --target=bpf -O2 -g -c prog.c -o prog.o
```

The `-O2` is mandatory in practice: BPF bytecode is limited to 1 million instructions per program, and unoptimized Clang output for non-trivial programs frequently exceeds this. The `-g` flag generates BTF (BPF Type Format), described below.

### 5.2 CO-RE: Compile Once, Run Everywhere

The fundamental challenge of BPF programs is portability: a program compiled against the kernel headers of kernel version A may access struct fields at wrong offsets when loaded onto kernel version B, where a struct has been reorganized. CO-RE solves this by recording field accesses as relocations rather than fixed offsets.

**`__builtin_preserve_access_index`** is the compiler primitive. When Clang sees a field access inside this builtin, it emits a BPF `LD_IMM64` instruction with a special relocation record (type `BPF_CORE_RELO`) in the `.BTF.ext` section rather than a hardcoded offset:

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_core_read.h>

SEC("kprobe/sys_execve")
int trace_execve(struct pt_regs *ctx) {
    struct task_struct *task = (struct task_struct *)bpf_get_current_task();
    pid_t pid = BPF_CORE_READ(task, pid);
    bpf_printk("pid = %d\n", pid);
    return 0;
}
```

`BPF_CORE_READ(task, pid)` expands to a `__builtin_preserve_access_index` wrapped access. When the BPF loader (libbpf) loads the program, it consults the running kernel's BTF (which the kernel exports via `/sys/kernel/btf/vmlinux`) and rewrites the relocation to the actual offset of `task_struct.pid` on this kernel version.

### 5.3 BTF Generation

BTF is a compact type format derived from DWARF, stored in ELF sections `.BTF` and `.BTF.ext`. Clang generates BTF when `-g` is passed with the BPF target. The kernel's own BTF is generated during the kernel build via `scripts/pahole-flags.sh` invoking `pahole --btf_encode_detached` on `vmlinux`.

BTF encodes:
- Type descriptors (structs, unions, enums, typedefs, function prototypes)
- Variable and function definitions with their types
- Line-number information correlating bytecode to source (in `.BTF.ext`)
- CO-RE relocation records mapping field accesses to type/field name pairs

```bash
# Inspect BTF in a compiled BPF object
bpftool btf dump file prog.o
# Generate a BPF skeleton for user-space loading
bpftool gen skeleton prog.o > prog.skel.h
```

The skeleton header `prog.skel.h` provides typed C wrappers around `bpf_object__open`, `bpf_object__load`, and map/program accessors, making user-space BPF programs fully type-safe.

### 5.4 Verifier Constraints Affecting Compilation

The BPF verifier imposes constraints that affect how Clang must generate code:

- **Stack depth ≤ 512 bytes per frame.** The verifier tracks stack usage statically. Deep recursion, large stack arrays, and large `struct`-by-value arguments cause verification failure. Clang will compile them but the verifier rejects the program at load time.
- **No unbounded loops.** The verifier must be able to prove termination. Clang generates loops that the verifier can unroll or bound-check; loops that depend on runtime data without a bounded trip count are rejected. Use `bpf_loop()` (kernel ≥ 5.17) for large bounded iterations.
- **No function pointers at runtime.** BPF does not support indirect calls except through the explicit `bpf_tail_call` mechanism. Clang's BPF backend inlines all calls that can be resolved statically; calls through C function pointers produce a verifier error.
- **No global variables with dynamic initializers.** BPF global data sections (`.data`, `.rodata`, `.bss`) are supported since kernel 5.2, but only for POD types.

See [Ch106 — WebAssembly and BPF](../part-15-targets/ch106-webassembly-and-bpf.md) for the BPF backend's register allocation and the CO-RE relocation encoding in detail.

---

## 6. Kernel Sanitizers via Clang

### 6.1 KASAN: Kernel Address Sanitizer

KASAN (`CONFIG_KASAN`) detects out-of-bounds and use-after-free accesses in kernel memory. It has three modes:

**Software shadow (`CONFIG_KASAN_GENERIC`)** maps each 8 bytes of kernel memory to 1 byte of shadow memory at a fixed virtual address (`KASAN_SHADOW_OFFSET`). The Clang instrumentation pass (`-fsanitize=address` for user-space, but for kernel the shadow is injected via `CONFIG_KASAN` without the user-space runtime) inserts a shadow check before every load and store:

```c
// Conceptual instrumentation inserted by Clang
void *shadow = (void *)((addr >> 3) + KASAN_SHADOW_OFFSET);
if (*shadow && (*shadow <= ((addr & 7) + access_size - 1)))
    kasan_report(addr, size, is_write, ip);
```

The kernel's `mm/kasan/` directory contains the shadow management and reporting infrastructure. Unlike user-space ASan, there is no `malloc` interception: kernel allocations from `kmalloc`, `vmalloc`, and the slab allocator are instrumented directly in those allocators to poison freed memory and unpack redzone bytes.

**Hardware tag-based (`CONFIG_KASAN_HW_TAGS`, arm64 only)** uses ARM Memory Tagging Extension (MTE). Each 16-byte granule of memory carries a 4-bit tag in physical memory; each pointer carries a matching tag in bits 56–59 (using the Top Byte Ignore architecture feature). The CPU hardware checks tag match on every load and store, with zero software overhead on the hot path. Clang enables this via `-march=armv8.5-a+memtag` and MTE-aware allocator code in `mm/kasan/hw_tags.c`.

### 6.2 KMSAN: Kernel Memory Sanitizer

KMSAN (`CONFIG_KMSAN`) detects uninitialized memory reads in the kernel. It is analogous to user-space MemorySanitizer but adapted for the kernel environment. Clang instruments the kernel with `-fsanitize=kernel-memory`, maintaining a shadow state that tracks which bytes have been initialized. The kernel's `mm/kmsan/` directory provides the origin tracking infrastructure that attributes uninitialized reads to their allocation site.

KMSAN is more expensive than KASAN at runtime — every memory operation carries a shadow check, and the shadow state propagates through arithmetic and memory copies. It is used in development and fuzzing (syzkaller), not production.

### 6.3 KCSAN: Kernel Concurrency Sanitizer

KCSAN (`CONFIG_KCSAN`) detects data races — concurrent accesses to the same memory location where at least one is a write and neither is protected by a lock. The implementation uses a watchpoint mechanism: Clang instruments every memory access to call `__tsan_read*` and `__tsan_write*` stubs. The kernel's KCSAN runtime (`kernel/kcsan/`) implements these stubs using a global watchpoint table rather than TSan's shadow memory, because TSan's approach requires user-space `mmap` patterns incompatible with the kernel address space.

```c
/* KCSAN instrumentation hook (simplified) */
void __tsan_write4(void *addr) {
    kcsan_check_access(addr, 4, KCSAN_ACCESS_WRITE);
}
```

KCSAN respects `data_race()` annotations (from `include/linux/compiler.h`) that mark intentional benign races, and `WRITE_ONCE`/`READ_ONCE` which suppress the race check by using volatile accesses.

For a complete treatment of these sanitizers' runtime implementations, see [Ch113 — Kernel Sanitizers](../part-16-jit-sanitizers/ch113-kernel-sanitizers.md) and [Ch110 — User-Space Sanitizers](../part-16-jit-sanitizers/ch110-user-space-sanitizers.md) for the user-space counterparts that share Clang's instrumentation infrastructure.

---

## 7. objtool: Post-Compilation Binary Validation

### 7.1 Role and Architecture

`objtool` is the kernel's post-compilation object file validator, written in C and living in `tools/objtool/`. It runs on each `.o` file after Clang produces it and before the linker sees it. Its job is to catch compiler output that violates kernel-specific invariants that neither the compiler nor the standard linker check:

- **Stack frame validation**: Every function must have a correct and complete ORC (Oops Rewind Capability) stack unwinding annotation. objtool reads the compiled object, traces all code paths, and computes the stack pointer offset at each instruction. It then emits ORC entries (stored in `.orc_unwind` and `.orc_unwind_ip` ELF sections) or reports an error.
- **`noinstr` violations**: Any instrumented call (to tracing infrastructure, to kasan hooks, to anything outside `.noinstr.text`) from a `noinstr`-marked function is a hard error. objtool tracks which sections are `noinstr`-qualified and validates that no cross-section instrumented calls exist.
- **`uaccess` region validation**: Functions that copy data between kernel and user space must execute with SMAP (Supervisor Mode Access Prevention) disabled. objtool validates that `user_access_begin` / `user_access_end` bracket all user memory accesses correctly.
- **Retpoline validation**: Indirect branches on x86 must use the `__x86_indirect_thunk_*` retpoline trampolines (or be replaced with IBRS-based mitigations). objtool validates that no direct indirect branches (`jmp *reg`, `call *mem`) survive into the final object.

### 7.2 ORC: The DWARF Replacement

The kernel replaced DWARF-based unwinding with ORC (Oops Rewind Capability) because DWARF is too expensive to parse in an oops handler: it requires heap allocation and complex parsing at the worst possible time. ORC stores a compact, fixed-stride table of `(sp_offset, bp_offset, type)` tuples indexed by instruction address. Given a faulting RIP, the unwinder binary-searches the ORC table in `arch/x86/kernel/unwind_orc.c` and walks the stack frame without any heap allocation.

objtool generates ORC entries by forward-propagating stack-pointer adjustments through the control-flow graph of each function. This requires understanding every instruction that modifies RSP — including `push`, `pop`, `sub rsp`, `add rsp`, and complex prologue/epilogue patterns. When Clang emits non-standard prologues (e.g., due to `-fomit-frame-pointer` or `-fsanitize=kcfi`'s type-id insertion), objtool must recognize the pattern. The Clang-kernel team maintains `tools/objtool/arch/x86/decode.c` as the shared understanding of what Clang may emit.

### 7.3 Stack Frame Requirements for Clang Output

Clang's output must satisfy objtool's expectations to produce a bootable kernel. The key constraints:

- The function prologue (push rbp / mov rbp, rsp) must appear before any `call` instruction when `-fno-omit-frame-pointer` is set.
- Interrupt handlers (marked `SYM_FUNC_START_NOALIGN`) must begin with the full `PUSH_AND_CLEAR_REGS` macro sequence before any C-generated code.
- kCFI type-id bytes (`__kcfi_typeid`) are placed in the four bytes preceding the function's first instruction. objtool skips these bytes during prologue analysis.

When a new Clang version changes prologue patterns — for instance, Clang 18 began emitting `sub rsp` before `push rbp` in certain register-pressure situations — the `objtool` instruction decoder must be updated before the kernel can build cleanly with that Clang version. This synchronization is tracked in the ClangBuiltLinux issue tracker.

---

## 8. Android GKI and ABI Stability

### 8.1 Generic Kernel Image Mandate

Android's Generic Kernel Image (GKI) program, introduced with Android 12, requires all Android kernels to be compiled with Clang. The rationale is three-fold: CFI (kCFI on arm64), LTO for kernel image size, and consistent toolchain behavior across the Android ecosystem. The GKI kernel ships as a prebuilt binary; device vendors ship their device-specific drivers as loadable modules (`ko` files) that must be ABI-compatible with the GKI.

### 8.2 GKI Symbol List and ABI Monitoring

The GKI ABI is defined by a symbol list (`android/abi_gki_aarch64`) enumerating every kernel symbol that vendor modules may reference. The ABI monitor workflow:

```bash
# Build GKI and extract ABI
BUILD_CONFIG=common/build.config.gki.aarch64 build/build_abi.sh

# Compare against baseline
python3 tools/bazel/abi_diff.py \
    --baseline abi_gki_aarch64_baseline.xml \
    --current abi_gki_aarch64_current.xml
```

The `abi_gki_*` scripts call `llvm-objdump` and `pahole` to extract struct layouts and function signatures, then encode them in an XML representation. Any change to a struct that is part of a symbol's type signature — even adding a field — is flagged as an ABI break.

### 8.3 Symbol Visibility and randstruct

Exported GKI symbols use `EXPORT_SYMBOL_NS` with a namespace qualifier. The `randstruct` feature randomizes struct field layout at compile time using a per-build seed, making it harder for exploits that rely on knowing struct offsets. Clang supports this natively via `__attribute__((randomize_layout))` and the `-frandomize-layout-seed=<seed>` driver flag; the kernel passes a build-specific seed generated by `scripts/gen-randstruct-seed.sh`. Vendor modules compiled with a different seed will have incompatible struct layouts, detected at module load time via the `MODULE_INFO(randstruct_seed, ...)` tag.

### 8.4 Module ABI Compatibility: GCC vs Clang Modules

Mixing GCC-compiled and Clang-compiled modules in the same kernel is officially unsupported but sometimes attempted. The compatibility risks:

- **Calling conventions**: GCC and Clang agree on the Itanium C++ ABI and Linux kernel ABI (no C++ in the kernel ABI), but may disagree on parameter passing for `__int128`, vector types, or packed structs in edge cases.
- **`MODULE_INFO(compiler, ...)`**: The kernel records the compiler used to build each module in this ELF note. The module loader does not reject mismatches but the information aids debugging.
- **randstruct seed**: If the kernel and module are compiled with different randstruct seeds, the module will access struct fields at wrong offsets, causing silent corruption or a crash.

---

## 9. Practical Workflow: Full LLVM Kernel Build with kCFI and ThinLTO

### 9.1 Prerequisites

```bash
# Verify LLVM 22 toolchain
clang --version       # Ubuntu clang version 22.1.3
ld.lld --version      # LLD 22.1.3
llvm-ar --version     # LLVM version 22.1.3

# Kernel source checked out, defconfig as baseline
cd /path/to/linux
make defconfig
```

### 9.2 Enabling ThinLTO and kCFI in the Config

```bash
# Enable ThinLTO
scripts/config --enable CONFIG_LTO_CLANG_THIN
# Enable kCFI (depends on LTO_CLANG or can be used standalone)
scripts/config --enable CONFIG_CFI_CLANG
# Enable KASAN (optional, for a debug build)
scripts/config --enable CONFIG_KASAN
scripts/config --set-val CONFIG_KASAN_INLINE y
```

### 9.3 Full Build Command

```bash
make LLVM=1 LLVM_IAS=1 \
     CC=clang \
     LD=ld.lld \
     AR=llvm-ar \
     NM=llvm-nm \
     OBJCOPY=llvm-objcopy \
     OBJDUMP=llvm-objdump \
     STRIP=llvm-strip \
     -j$(nproc) \
     bzImage modules
```

With `LLVM=1`, the `CC=clang` and other assignments are redundant but explicit, which is recommended in CI scripts for clarity and in case `LLVM=1` semantics change across kernel versions.

### 9.4 Verifying CFI and ThinLTO Are Active

```bash
# Confirm kCFI type-ids are present (4 bytes before each function entry)
llvm-objdump -d --no-show-raw-insn vmlinux | grep -A1 "__kcfi_typeid"

# Confirm ThinLTO summary sections are absent in final vmlinux
# (they are consumed by the linker; the final binary is native code)
llvm-readelf --sections vmlinux | grep llvm

# Check that objtool ran cleanly (no errors in the build log)
make LLVM=1 -j$(nproc) bzImage 2>&1 | grep -i "objtool"

# Verify CFI config is set
grep CONFIG_CFI_CLANG .config
# CONFIG_CFI_CLANG=y

# Check kCFI in Kconfig output
grep CONFIG_LTO_CLANG_THIN .config
# CONFIG_LTO_CLANG_THIN=y
```

### 9.5 Building an Out-of-Tree Module Against a kCFI + ThinLTO Kernel

Out-of-tree modules must be compiled with matching settings or they will fail to load if the kernel was built with `CONFIG_MODVERSIONS` and mismatched CFI seeds:

```bash
# Build an out-of-tree module against a kCFI-enabled kernel
make -C /path/to/linux M=$(pwd) \
     LLVM=1 LLVM_IAS=1 \
     CC=clang \
     modules

# The module inherits CFLAGS from the kernel's Kbuild infrastructure,
# including -fsanitize=kcfi if CONFIG_CFI_CLANG=y.
```

---

## 10. Chapter Summary

- **`LLVM=1`** replaces the entire GNU binutils toolchain with LLVM equivalents in a single make variable; `LLVM_IAS=1` adds the integrated assembler and is now default for most architectures.

- **Kernel-specific Clang flags** each address a real kernel-language invariant: `-fno-strict-aliasing` for `container_of`, `-fno-delete-null-pointer-checks` for address-0 mapping, `-fno-pic`/`-fno-pie` for the fixed kernel address space, and `-fstack-protector-strong` for stack security.

- **kCFI** (`-fsanitize=kcfi`) provides per-module CFI without LTO by embedding a type-hash in the four bytes preceding each function. It is the default on arm64 since Linux 6.2. **`__nocfi`** opts individual functions out of incoming checks. CFI-icall (`-fsanitize=cfi-icall`) is the stricter LTO-dependent variant that uses jump tables.

- **ThinLTO** (`CONFIG_LTO_CLANG_THIN`) compiles each translation unit to LLVM bitcode, performs parallel thin-link optimization, and emits native code at link time. `--thinlto-jobs` parallelizes per-module codegen. Interaction with `CONFIG_MODVERSIONS` requires a two-pass link.

- **BPF compilation** uses `clang --target=bpf -O2 -g`. CO-RE (`__builtin_preserve_access_index`, `BPF_CORE_READ`) emits field-access relocations resolved by libbpf at load time against the running kernel's BTF. Verifier constraints (512-byte stack, no unbounded loops, no function pointers) are enforced at load time, not compile time.

- **KASAN** instruments kernel memory accesses with a shadow check; hardware-tag mode on arm64 uses ARM MTE for zero hot-path overhead. **KMSAN** tracks uninitialized bytes via Clang's `-fsanitize=kernel-memory`. **KCSAN** detects data races via `__tsan_read*`/`__tsan_write*` stubs implemented as a watchpoint table.

- **objtool** runs on each `.o` after Clang and before the linker, validating stack frames for ORC unwinder generation, checking `noinstr` section isolation, validating `uaccess` regions, and enforcing retpoline compliance. It emits `.orc_unwind` tables as a compact, allocation-free DWARF replacement.

- **Android GKI** mandates Clang for all Android kernels since Android 12, enforces symbol-list-based ABI stability checked by `abi_gki_*` scripts, and uses `randstruct` for struct layout randomization with per-build seeds encoded in module metadata.

- A complete x86_64 kernel build with full LLVM toolchain requires only `make LLVM=1 LLVM_IAS=1 -j$(nproc) bzImage`; all compiler and binutils substitutions follow automatically.
