# Ch200 — Linux Kernel Compilation with LLVM/Clang

*Part XXIX — Compiler Tooling, Kernel Integration, and Binary Analysis*

The Linux kernel shipped for decades with GCC as its only supported compiler. That changed gradually and then all at once: Android mandated Clang for all kernels beginning with Android 12, ChromeOS moved to Clang for security features GCC could not provide, and major cloud providers followed to gain ThinLTO, kernel CFI, and superior diagnostics. Today, building a mainline kernel with the full LLVM toolchain — Clang, LLD, llvm-ar, llvm-objcopy, and the integrated assembler — is the default path on arm64 for Android and ChromeOS, and a first-class option on x86_64 upstream. This chapter explains the mechanics of that integration in depth: the build flags and why each one exists, the two CFI models the kernel supports, ThinLTO's per-module workflow, BPF compilation with CO-RE, the kernel's Clang-backed sanitizers, and the `objtool` binary checker that validates compiler output before the kernel ships.

---

## Table of Contents

- [1. The ClangBuiltLinux Project](#1-the-clangbuiltlinux-project)
  - [1.1 History and Motivation](#11-history-and-motivation)
  - [1.2 Kernel Tree Support](#12-kernel-tree-support)
  - [1.3 The `LLVM=1` Build Variable](#13-the-llvm1-build-variable)
- [2. Kernel-Specific Clang Flags](#2-kernel-specific-clang-flags)
  - [2.1 Type Aliasing and Null Pointer Handling](#21-type-aliasing-and-null-pointer-handling)
  - [2.2 Linkage and Stack Security](#22-linkage-and-stack-security)
  - [2.3 Floating-Point and Position](#23-floating-point-and-position)
  - [2.4 Dead Code Elimination and Warning Suppression](#24-dead-code-elimination-and-warning-suppression)
- [3. Clang CFI for the Kernel](#3-clang-cfi-for-the-kernel)
  - [3.1 kCFI: Per-Module CFI Without LTO](#31-kcfi-per-module-cfi-without-lto)
  - [3.2 CFI-icall with ThinLTO](#32-cfi-icall-with-thinlto)
  - [3.3 The `noinstr` Annotation](#33-the-noinstr-annotation)
- [4. ThinLTO for the Kernel](#4-thinlto-for-the-kernel)
  - [4.1 Configuration and Workflow](#41-configuration-and-workflow)
  - [4.2 Interaction with CONFIG_MODVERSIONS](#42-interaction-with-configmodversions)
  - [4.3 Verifying ThinLTO Output](#43-verifying-thinlto-output)
- [5. BPF Program Compilation](#5-bpf-program-compilation)
  - [5.1 The BPF Target Triple](#51-the-bpf-target-triple)
  - [5.2 CO-RE: Compile Once, Run Everywhere](#52-co-re-compile-once-run-everywhere)
  - [5.3 BTF Generation](#53-btf-generation)
  - [5.4 Verifier Constraints Affecting Compilation](#54-verifier-constraints-affecting-compilation)
- [6. Kernel Sanitizers via Clang](#6-kernel-sanitizers-via-clang)
  - [6.1 KASAN: Kernel Address Sanitizer](#61-kasan-kernel-address-sanitizer)
  - [6.2 KMSAN: Kernel Memory Sanitizer](#62-kmsan-kernel-memory-sanitizer)
  - [6.3 KCSAN: Kernel Concurrency Sanitizer](#63-kcsan-kernel-concurrency-sanitizer)
- [7. objtool: Post-Compilation Binary Validation](#7-objtool-post-compilation-binary-validation)
  - [7.1 Role and Architecture](#71-role-and-architecture)
  - [7.2 ORC: The DWARF Replacement](#72-orc-the-dwarf-replacement)
  - [7.3 Stack Frame Requirements for Clang Output](#73-stack-frame-requirements-for-clang-output)
- [8. Android GKI and ABI Stability](#8-android-gki-and-abi-stability)
  - [8.1 Generic Kernel Image Mandate](#81-generic-kernel-image-mandate)
  - [8.2 GKI Symbol List and ABI Monitoring](#82-gki-symbol-list-and-abi-monitoring)
  - [8.3 Symbol Visibility and randstruct](#83-symbol-visibility-and-randstruct)
  - [8.4 Module ABI Compatibility: GCC vs Clang Modules](#84-module-abi-compatibility-gcc-vs-clang-modules)
- [9. Practical Workflow: Full LLVM Kernel Build with kCFI and ThinLTO](#9-practical-workflow-full-llvm-kernel-build-with-kcfi-and-thinlto)
  - [9.1 Prerequisites](#91-prerequisites)
  - [9.2 Enabling ThinLTO and kCFI in the Config](#92-enabling-thinlto-and-kcfi-in-the-config)
  - [9.3 Full Build Command](#93-full-build-command)
  - [9.4 Verifying CFI and ThinLTO Are Active](#94-verifying-cfi-and-thinlto-are-active)
  - [9.5 Building an Out-of-Tree Module Against a kCFI + ThinLTO Kernel](#95-building-an-out-of-tree-module-against-a-kcfi-thinlto-kernel)
- [10. Linux Kernel Live Patching and Runtime Code Modification](#10-linux-kernel-live-patching-and-runtime-code-modification)
  - [10.1 The Livepatch Subsystem](#101-the-livepatch-subsystem)
  - [10.2 `text_poke_bp()` — Atomic Instruction Replacement](#102-textpokebp-atomic-instruction-replacement)
  - [10.3 ftrace-Based Function Redirection](#103-ftrace-based-function-redirection)
  - [10.4 Userspace Dynamic Probes](#104-userspace-dynamic-probes)
- [11. Chapter Summary](#11-chapter-summary)

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

## 10. Linux Kernel Live Patching and Runtime Code Modification

The preceding sections covered compile-time toolchain choices. This section covers runtime modification of the running kernel — applying security fixes or bug patches to a live system without a reboot. The mechanisms involve the same dual-mapping and instruction-replacement techniques used by user-space JITs, but in the kernel address space with additional synchronization requirements.

### 10.1 The Livepatch Subsystem

The Linux kernel livepatch subsystem (`kernel/livepatch/`, public API in `include/linux/livepatch.h`) supports hot-replacing individual kernel functions at runtime. The subsystem is architecture-independent; the low-level instruction replacement is delegated to `ftrace` and `text_poke_bp` (Sections 10.3 and 10.2).

The data structures form a three-level hierarchy:

```c
/* A replacement for one kernel function. */
struct klp_func {
    const char *old_name;     /* function to replace, e.g. "vfs_read"    */
    unsigned long old_sympos; /* disambiguates when multiple symbols match;
                                 0 = first match                         */
    void *new_func;           /* pointer to the replacement function      */
    /* runtime fields filled by the subsystem: */
    unsigned long old_addr;   /* resolved address of old_name             */
    /* ... */
};

/* A kernel module (or vmlinux) whose functions are being patched. */
struct klp_object {
    const char *name;        /* NULL for vmlinux; module name otherwise   */
    struct klp_func *funcs;  /* NULL-terminated array of klp_func         */
    struct klp_callbacks callbacks; /* pre_patch, post_patch, pre_unpatch,
                                       post_unpatch hooks                 */
};

/* The top-level patch descriptor registered with the kernel. */
struct klp_patch {
    struct module *mod;      /* the livepatch module itself               */
    struct klp_object *objs; /* NULL-terminated array of klp_object       */
    bool enabled;            /* true after klp_enable_patch() succeeds    */
    /* ... */
};
```

A minimal livepatch module patches a single kernel function:

```c
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/livepatch.h>

/* Replacement for the hypothetical vulnerable_fn in vmlinux */
static int patched_vulnerable_fn(int arg)
{
    if (arg < 0)
        return -EINVAL;      /* bounds check the original forgot */
    return original_logic(arg);
}

static struct klp_func funcs[] = {
    { .old_name = "vulnerable_fn", .new_func = patched_vulnerable_fn },
    { }   /* sentinel */
};

static struct klp_object objs[] = {
    { .name = NULL, .funcs = funcs },  /* NULL name → patch vmlinux */
    { }
};

static struct klp_patch patch = {
    .mod  = THIS_MODULE,
    .objs = objs,
};

static int __init livepatch_init(void) { return klp_enable_patch(&patch); }
static void __exit livepatch_exit(void) { }

module_init(livepatch_init);
module_exit(livepatch_exit);
MODULE_LICENSE("GPL");
MODULE_INFO(livepatch, "Y");  /* required; marks this as a livepatch module */
```

**Consistency model.** After `klp_enable_patch()`, the kernel does not immediately redirect all callers. Instead it waits for a *safe point*: each task is migrated to the new code only after it has fully unwound out of any frame that contains `old_func`. This prevents the deopt scenario where a task is executing old code that has been removed from underneath it. The consistency model is called "per-task consistency" and is implemented in `kernel/livepatch/transition.c`.

### 10.2 `text_poke_bp()` — Atomic Instruction Replacement

`text_poke_bp()` (`arch/x86/include/asm/text-patching.h`, implementation in `arch/x86/kernel/alternative.c`) is the low-level primitive that atomically replaces an instruction sequence of up to `POKE_MAX_OPCODE_SIZE = 5` bytes in the kernel's text without taking the kernel offline. The atomicity guarantee is provided by an int3-based protocol that prevents any CPU from executing a partially-written instruction:

```
Step 1: Replace the first byte of the target instruction with int3 (0xCC).
        Any CPU about to execute the instruction hits the breakpoint instead.

Step 2: IPI (inter-processor interrupt) to all other CPUs — wait for them
        to acknowledge the int3 is in place (smp_call_function_many).

Step 3: Write the remaining bytes of the new instruction sequence.
        The int3 serializes instruction fetch, so no CPU sees a torn instruction.

Step 4: IPI to all CPUs again to flush their instruction pipelines.

Step 5: Replace the int3 with the final first byte of the new instruction.
        The instruction is now atomically visible to all CPUs.
```

The int3 breakpoint handler (`poke_int3_handler()` in `arch/x86/kernel/alternative.c`) emulates the old instruction's effect while the replacement is in progress, ensuring forward progress. For `ftrace`-size patches (a `CALL_INSN_SIZE = 5` byte call instruction), the sequence patches the call target in one text_poke_bp operation.

For patching multiple sites atomically, use the batching API:

```c
/* Queue multiple patches (does not flush yet): */
text_poke_queue(addr1, opcode1, len1, emulate1);
text_poke_queue(addr2, opcode2, len2, emulate2);
/* Apply all queued patches atomically with a single IPI round: */
text_poke_finish();
```

**The `poking_mm` dual-mapping.** Because kernel `.text` is mapped read-only in the page tables, `text_poke` cannot write directly. It uses `poking_mm` — a separate `mm_struct` maintained by the kernel that maps the same physical kernel-text page frames with write permission (`poking_addr` in `arch/x86/kernel/alternative.c`). The write is performed via a temporary mapping in `poking_mm`; the actual kernel virtual address remains read-only throughout. This is the kernel's implementation of the same dual-mapping pattern used by user-space JITs.

### 10.3 ftrace-Based Function Redirection

`ftrace` (function tracer) is the mechanism that livepatch uses to redirect calls to patched functions. At kernel build time, Clang with `-pg` (or `-mfentry` for x86_64) emits a `call __fentry__` instruction — a 5-byte `CALL_INSN_SIZE` call — at the very start of every non-inline function prologue. This call is initially to a NOP stub but can be redirected at runtime.

Build the kernel with ftrace-compatible Clang flags:

```bash
# -pg emits the __fentry__ call on x86_64:
make CC=clang LLVM=1 CONFIG_FUNCTION_TRACER=y -j$(nproc) vmlinux
```

At runtime, `ftrace_modify_call()` (`kernel/trace/ftrace.c`) patches the `call __fentry__` instruction to call a trampoline. The livepatch subsystem registers `ftrace_ops` handlers for each function being patched:

```
Before patching:
  call __fentry__  (no-op; returns immediately)
  <function prologue>
  <function body>

After klp_enable_patch():
  call klp_ftrace_handler  (→ checks klp_task_in_transition, redirects to new_func)
  <function prologue — now unreachable for patched tasks>
```

The `klp_ftrace_handler` in `kernel/livepatch/patch.c` redirects the call by modifying the return address on the stack (`ftrace_regs`) before `__fentry__` returns — effectively a return-address hijack that is safe because the entire protocol is synchronized.

`ftrace` itself uses `text_poke_bp` to install the redirected call, completing the layering: livepatch → ftrace → `text_poke_bp` → `poking_mm` dual-mapping.

### 10.4 Userspace Dynamic Probes

The same code-patching concepts extend to userspace via `uprobes` (userspace probes). A uprobe installs an int3 breakpoint at a virtual address in a running process — including inside shared libraries — and fires a handler on every hit.

```bash
# Attach a probe to the entry of function 'handle_request' in a running server:
perf probe --exec=/usr/sbin/my_server --add 'handle_request'

# Record all hits with arguments (if DWARF debug info is present):
perf record -e probe:handle_request -aR sleep 10
perf script
```

Under the hood, `perf probe --add` calls `uprobe_register()` in the kernel (`kernel/events/uprobes.c`), which:
1. Locates the virtual address of the symbol via `/proc/<pid>/maps` and ELF symbol lookup.
2. Writes an `int3` to the target page in the process's address space via `get_user_pages` + `copy_to_user_page`.
3. On each `int3` hit, the kernel calls the registered uprobe handler before returning to userspace.

Uprobes work identically to kprobes (kernel dynamic probes) but in user virtual memory. They require no recompilation and no source access — only the ELF binary and, for argument capture, DWARF debug information.

For Clang-compiled binaries, ensure DWARF info is present:

```bash
clang -O2 -g my_server.c -o my_server
# -g emits DWARF; uprobe argument capture uses it to locate registers/offsets
```

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **GCC-13 minimum removal and Clang parity enforcement**: The ClangBuiltLinux community is tracking the removal of GCC 12 and older from the minimum-version table in `Documentation/process/changes.rst`, clearing the path to deprecate kernel workarounds (e.g., `-Wno-gnu` attribute quirks, `zero-length-array` compatibility) that exist only to tolerate GCC behavior. Clang 18+ is expected to become the effective floor in Android GKI.
- **kCFI for RISC-V**: kCFI landed on arm64 in Linux 6.2; the upstream effort to enable `CONFIG_CFI_CLANG` on RISC-V (tracking LLVM RISC-V backend support for the four-byte preamble typeid and `ebreak`-based trap) is active on the ClangBuiltLinux mailing list and in LLVM's RISC-V backend work.
- **`objtool` instruction-set extensions for Clang 22 prologues**: Clang 22 introduces new prologue patterns under `-fomit-frame-pointer` and ShadowCallStack on arm64. The kernel's `tools/objtool/arch/` decoder tables must be updated; patches are expected before Linux 6.12 merges LLVM 22 as a tested compiler version.
- **BPF token and unprivileged BPF hardening**: The upstream BPF token patchset (RFC on netdev/bpf list, 2025) enables fine-grained delegation of BPF program loading rights without `CAP_BPF`. Clang's BPF backend will need to emit new relocation types to allow the kernel verifier to enforce token-scoped CO-RE relocations at load time.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Clang-only kernel builds as the upstream default for arm64**: Based on current trajectory (Android GKI already mandates Clang; ChromeOS and AWS follow), mainline Linux arm64 `defconfig` may drop the fallback `CC=gcc` path for new architectures. This requires resolving remaining inline-assembly dialect gaps in LLVM's integrated assembler and completing the `LLVM_IAS=1` coverage for all `arch/arm64` `.S` files.
- **Full LTO (monolithic) kernel builds via the distributed cache model**: ThinLTO is the practical LTO mode today; full LTO (`CONFIG_LTO_CLANG_FULL`) is feasible only for small kernels due to link-time memory consumption. Distributed LTO caching via `llvmcache` (similar to ccache for bitcode units) is a research area that would make full LTO viable for production kernel CI, enabling stronger whole-program CFI and dead-code elimination than ThinLTO provides.
- **MTE v2 + KASAN hardware-tag integration on ARMv9.4**: ARM MTE version 2 (part of ARMv9.4-A) extends tag storage from 4 bits to 8 bits per granule and adds asynchronous-mode improvements. KASAN hw-tags in the kernel will need updates to `mm/kasan/hw_tags.c` and Clang's `-march=armv9.4-a+memtag2` target flag. Academic research (ARM Ltd. white papers, 2024) suggests MTE v2 can provide quarantine-free temporal safety at <2% overhead.
- **Livepatch + kCFI compatibility on arm64**: The current livepatch `klp_ftrace_handler` redirects execution by rewriting the return address in `ftrace_regs`. With kCFI, the replacement function's `__kcfi_typeid` must match the callee type at the original call site; the livepatch subsystem currently bypasses this check via `__nocfi`. A proposed upstream design integrates kCFI typeid patching into the livepatch enable/disable path, tracked in the Linux livepatch mailing list.

### 5-Year Horizon (Long-Term, by ~2031)

- **Formal verification of objtool CFG analysis using Lean 4 or Coq**: objtool's forward-propagation of stack-pointer adjustments across arbitrary x86_64 control-flow graphs is safety-critical: an incorrect ORC table causes silent unwind failures in production oops handlers. The CompCert and Vellvm communities are exploring formal models of x86_64 stack frames; a verified objtool CFG pass connected to a Coq model of x86_64 semantics would eliminate this class of silent correctness bugs.
- **Compiler-assisted kernel ABI stability via MLIR type export**: The Android GKI ABI monitor today relies on `pahole` + XML diffing, which misses semantic changes (e.g., changed enum values, behavior-changing default arguments). A prospective design would have Clang emit MLIR-based type descriptors into a dedicated ELF section at compile time, making struct layout, function signature, and semantic annotations machine-verifiable across kernel versions without a separate pahole pass.
- **BPF program synthesis and compiler-verifier co-design**: The BPF verifier's constraint set (512-byte stack, no unbounded loops, no indirect calls) limits expressibility for complex BPF programs. A research direction being explored in the USENIX papers (e.g., "Safe and Efficient eBPF", 2024) involves a co-designed Clang BPF backend that emits programs already in a verifier-normal form, eliminating the retry-and-reformulate loop developers currently face; this would require a new Clang BPF dialect in MLIR and a verifier-feedback IR pass.

---

## 11. Chapter Summary

- **`LLVM=1`** replaces the entire GNU binutils toolchain with LLVM equivalents in a single make variable; `LLVM_IAS=1` adds the integrated assembler and is now default for most architectures.

- **Kernel-specific Clang flags** each address a real kernel-language invariant: `-fno-strict-aliasing` for `container_of`, `-fno-delete-null-pointer-checks` for address-0 mapping, `-fno-pic`/`-fno-pie` for the fixed kernel address space, and `-fstack-protector-strong` for stack security.

- **kCFI** (`-fsanitize=kcfi`) provides per-module CFI without LTO by embedding a type-hash in the four bytes preceding each function. It is the default on arm64 since Linux 6.2. **`__nocfi`** opts individual functions out of incoming checks. CFI-icall (`-fsanitize=cfi-icall`) is the stricter LTO-dependent variant that uses jump tables.

- **ThinLTO** (`CONFIG_LTO_CLANG_THIN`) compiles each translation unit to LLVM bitcode, performs parallel thin-link optimization, and emits native code at link time. `--thinlto-jobs` parallelizes per-module codegen. Interaction with `CONFIG_MODVERSIONS` requires a two-pass link.

- **BPF compilation** uses `clang --target=bpf -O2 -g`. CO-RE (`__builtin_preserve_access_index`, `BPF_CORE_READ`) emits field-access relocations resolved by libbpf at load time against the running kernel's BTF. Verifier constraints (512-byte stack, no unbounded loops, no function pointers) are enforced at load time, not compile time.

- **KASAN** instruments kernel memory accesses with a shadow check; hardware-tag mode on arm64 uses ARM MTE for zero hot-path overhead. **KMSAN** tracks uninitialized bytes via Clang's `-fsanitize=kernel-memory`. **KCSAN** detects data races via `__tsan_read*`/`__tsan_write*` stubs implemented as a watchpoint table.

- **objtool** runs on each `.o` after Clang and before the linker, validating stack frames for ORC unwinder generation, checking `noinstr` section isolation, validating `uaccess` regions, and enforcing retpoline compliance. It emits `.orc_unwind` tables as a compact, allocation-free DWARF replacement.

- **Android GKI** mandates Clang for all Android kernels since Android 12, enforces symbol-list-based ABI stability checked by `abi_gki_*` scripts, and uses `randstruct` for struct layout randomization with per-build seeds encoded in module metadata.

- **Linux kernel live patching** uses a three-level hierarchy (`klp_func` → `klp_object` → `klp_patch`) registered via `klp_enable_patch()`; the per-task consistency model (`kernel/livepatch/transition.c`) ensures tasks are migrated to new code only after unwinding out of all old-function frames, preventing mid-execution code removal.

- **`text_poke_bp()`** (`arch/x86/include/asm/text-patching.h`) atomically replaces up to `POKE_MAX_OPCODE_SIZE = 5` bytes in kernel text via an int3-based protocol: write int3 → IPI-sync → write remaining bytes → IPI-sync → write final first byte; the `poking_mm` dual-mapping provides a writable alias to the read-only kernel text pages without violating W^X; `text_poke_queue()`+`text_poke_finish()` batch multiple patches under one IPI round.

- **ftrace-based redirection**: Clang with `-pg`/`-mfentry` emits a `CALL_INSN_SIZE = 5` byte `call __fentry__` NOP at each function prologue; `ftrace_modify_call()` patches it to call a livepatch trampoline; `klp_ftrace_handler` redirects execution to the replacement function by rewriting the return address in `ftrace_regs` before `__fentry__` returns; the full chain is livepatch → ftrace → `text_poke_bp` → `poking_mm`.

- **Userspace dynamic probes** (`perf probe --add` / `uprobe_register()`): install an int3 breakpoint in a live process via `get_user_pages`+`copy_to_user_page`; fire a handler on every hit without recompilation; DWARF debug info enables argument capture; uprobes apply the same int3-based patching as kprobes but in user virtual memory.

- A complete x86_64 kernel build with full LLVM toolchain requires only `make LLVM=1 LLVM_IAS=1 -j$(nproc) bzImage`; all compiler and binutils substitutions follow automatically.
