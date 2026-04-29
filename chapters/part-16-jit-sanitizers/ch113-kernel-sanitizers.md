# Chapter 113 — Kernel Sanitizers

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

The Linux kernel presents a uniquely hostile environment for runtime bug detection. Unlike userspace programs, kernel code runs with full hardware privileges, lacks memory-safe boundaries enforced by the OS, and executes in interrupt and atomic contexts where blocking operations are forbidden. Yet the kernel is also among the most security-critical and concurrency-rich codebases in existence — memory corruption, data races, and uninitialized reads in kernel context mean privilege escalation, data leaks to userspace, and system crashes. This chapter covers the family of LLVM-backed kernel sanitizers: KASAN, KFENCE, KCSAN, KMSAN, kernel UBSan, and CFI, examining their instrumentation strategies, runtime integration with kernel subsystems, and practical use in development and continuous integration.

---

## 113.1 Kernel Sanitizer Overview

### Why Kernel Sanitizers Are Harder

Userspace sanitizers assume a fully cooperating runtime environment: a C library, signal delivery, stack unwinding, and virtual memory managed by the OS. Kernel sanitizers must bootstrap themselves within the very system they instrument.

**No signal handlers.** Kernel bugs cannot be reported via `SIGSEGV`; instead reports go through `pr_err()` / `dump_stack()` / `panic()`. An out-of-bounds access that would produce a sanitizer report in userspace causes a hard CPU fault in the kernel — unless the sanitizer intercepts it first.

**No standard allocator.** Userspace ASan wraps `malloc`/`free`. Kernel sanitizers must hook `kmalloc`/`kfree`, the slab allocator's internal path, page allocators, and stack frame allocation — across SLUB, SLAB, and SLOB backends.

**Preemption and interrupt contexts.** Code running in interrupt context cannot sleep, cannot acquire certain locks, and must complete quickly. Sanitizer runtime code that runs on every load/store must be carefully written to avoid scheduling operations. KASAN's shadow read path, for instance, is simple enough to be safe in NMI context.

**Kernel address space layout.** Shadow memory must be mapped in the kernel virtual address space at a fixed offset known at compile time. This requires architecture-specific early boot support (`kasan_init()`) before the normal memory allocator is usable.

**Early-boot coverage gaps.** The sanitizer cannot be active during the earliest boot stages (before memory is initialized). Kernel sanitizers use `__init` annotations and careful sequencing to avoid false positives and crashes during initialization.

### Sanitizer History in the Linux Kernel

| Year | Sanitizer | Description |
|------|-----------|-------------|
| 2015 | KASAN | Kernel AddressSanitizer; merged in Linux 4.0 |
| 2017 | Kernel UBSan | Undefined behavior detection; `lib/ubsan.c` |
| 2019 | KCSAN | Kernel Concurrency Sanitizer; merged in Linux 5.8 |
| 2021 | KFENCE | Kernel Electric Fence; low-overhead production sanitizer; Linux 5.12 |
| 2021 | KMSAN | Kernel MemorySanitizer; merged in Linux 6.1 |
| 2022 | CFI (KCFI) | Clang CFI for the kernel; Linux 6.1 |

### Common Kernel Infrastructure

All kernel sanitizers share configuration via Kconfig symbols and route their reports through a common reporting infrastructure:

```
lib/ubsan.c                  # UBSan runtime
mm/kasan/                    # KASAN core
  kasan_init.c               # Shadow memory setup
  report.c                   # Bug reporting
  generic.c                  # Generic KASAN instrumentation
  sw_tags.c                  # Software tag-based KASAN
  hw_tags.c                  # Hardware tag-based KASAN (MTE)
  quarantine.c               # Freed-object quarantine
kernel/kcsan/                # KCSAN core
  core.c
  report.c
mm/kmsan/                    # KMSAN core
  kmsan.c
  kmsan_shadow.c
mm/kfence/                   # KFENCE
  core.c
  report.c
```

### Build Configuration

```bash
# KASAN (generic software):
CONFIG_KASAN=y
CONFIG_KASAN_GENERIC=y
CONFIG_KASAN_OUTLINE=y    # or CONFIG_KASAN_INLINE=y

# KFENCE:
CONFIG_KFENCE=y
CONFIG_KFENCE_SAMPLE_INTERVAL=100  # milliseconds between samples

# KCSAN:
CONFIG_KCSAN=y
CONFIG_KCSAN_REPORT_VALUE_CHANGE_ONLY=y

# KMSAN:
CONFIG_KMSAN=y

# UBSan:
CONFIG_UBSAN=y
CONFIG_UBSAN_BOUNDS=y
CONFIG_UBSAN_SHIFT=y
CONFIG_UBSAN_INTEGER_WRAP=y

# CFI:
CONFIG_CFI_CLANG=y
CONFIG_SHADOW_CALL_STACK=y   # AArch64 only

# Coverage for syzkaller:
CONFIG_KCOV=y
```

Most kernel sanitizers require Clang; GCC support is limited or absent for KMSAN, KCFI, and hardware-tag-based KASAN. The Linux kernel documents state explicitly: "KMSAN requires Clang 14 or later." KCFI similarly requires Clang's `__kcfi_typeid` attribute.

---

## 113.2 KASAN — Kernel AddressSanitizer

### Design Overview

KASAN adapts the userspace AddressSanitizer algorithm (Chapter 110) for the kernel address space. It maintains a shadow memory region where each byte represents the poisoning state of 8 bytes of real memory. A shadow byte value of 0 means all 8 bytes are accessible; a value of N (1–7) means only the first N bytes are accessible; negative values encode special poisoning states (freed memory, redzone, etc.).

### Three Operating Modes

**Generic KASAN** (software shadow): The most portable mode. Works on x86-64, AArch64, and other architectures. Uses a dedicated shadow region mapped at `KASAN_SHADOW_OFFSET`. On x86-64, the offset is `0xdffffc0000000000`; on AArch64 it may vary. The shadow occupies 1/8 of the kernel virtual address space.

Shadow byte encoding:
```
  0x00 — fully accessible
  0x01–0x07 — partially accessible (first N bytes valid)
  0xF1 — left redzone (kmalloc slab)
  0xF2 — right redzone
  0xF5 — use-after-free (quarantined object)
  0xF8 — kmalloc redzone (partial)
  0xE8 — stack left redzone
  0xE9 — stack right redzone
  0xF9 — global redzone
```

**Software tag-based KASAN** (SW-Tag): AArch64 only. Uses the Top Byte Ignore (TBI) hardware feature to embed a random 8-bit tag in pointer bits 56–63. Each allocation is assigned a random tag; the shadow byte stores the tag. On access, the pointer's embedded tag is compared against the shadow tag; mismatch → report. This catches use-after-free even without quarantine and has lower overhead (~10–15%) than generic KASAN.

**Hardware tag-based KASAN** (HW-Tag): AArch64 ARMv8.5-A with MTE (Memory Tagging Extension) only. Uses hardware memory tags — each 16-byte granule of memory has a 4-bit tag stored in a separate hardware tag memory. The processor checks tags on every load/store in hardware. Overhead approaches zero for normal operation; faults only on tag mismatch. Requires `CONFIG_ARM64_MTE=y`.

### Shadow Memory Initialization

[`mm/kasan/kasan_init.c`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/lib/asan/asan_init_version.h) tracks the interface; the kernel side lives in `mm/kasan/init.c`:

```c
// Simplified view of KASAN shadow setup (arch/x86/mm/kasan_init.c):
void __init kasan_init(void)
{
    // Map shadow pages for all kernel memory regions
    // using early_pfn_to_nid() before the allocator is up
    kasan_map_shadow((void *)PAGE_OFFSET,
                     (void *)PAGE_OFFSET + (1UL << MAX_PHYSMEM_BITS));
    // Map shadow for vmalloc range
    kasan_populate_shadow(VMALLOC_START, VMALLOC_END);
    // Mark early boot stacks as valid
    kasan_unpoison_memory(init_task.stack, THREAD_SIZE);
    // Enable kasan_report() -- previously all reports were suppressed
    init_task.kasan_depth = 0;
}
```

The shadow mapping uses `memblock_reserve` during early boot, before `mm_init()` runs.

### Compiler Instrumentation

Clang (via `-fsanitize=kernel-address`) inserts shadow checks before every load and store. In "outline" mode, each access generates a call to a runtime function:

```c
// Generated for: int x = *ptr;
__asan_load4(ptr);  // check 4 bytes at ptr
// Actual load:
int x = *ptr;

// Generated for: *ptr = 42;
__asan_store4(ptr);  // check 4 bytes at ptr
// Actual store:
*ptr = 42;
```

In "inline" mode (`CONFIG_KASAN_INLINE=y`), the shadow check is inlined directly:

```c
// Inline shadow check for load4:
void *shadow = (void *)(((unsigned long)ptr >> 3) + KASAN_SHADOW_OFFSET);
int8_t shadow_byte = *(int8_t *)shadow;
if (shadow_byte && shadow_byte < (int8_t)((unsigned long)ptr & 7) + 4) {
    kasan_report(ptr, 4, false, _RET_IP_);
}
```

Inline mode has lower overhead (avoids function call overhead) but generates larger code. On performance-sensitive kernels, inline mode is preferred despite the binary size increase.

### Slab Integration

KASAN integrates with the SLUB allocator to poison/unpoison memory around each allocation:

- **On `kmalloc(size)`**: the actual slab object of size `size_with_redzone` is allocated; KASAN marks bytes 0..size-1 as accessible (shadow 0), bytes size..size_with_redzone-1 as right redzone (`0xF2`).
- **On `kfree()`**: the object is marked as quarantined (`0xF5`); the actual memory is not returned to the slab until the quarantine is flushed. Any access to the quarantined object triggers a use-after-free report.
- **Redzones**: small allocations get redzones on both sides within the slab object. Adjacent allocations cannot overflow into each other without hitting a redzone.

```c
// Excerpt from mm/kasan/common.c
static void *kasan_kmalloc(struct kmem_cache *cache, const void *object,
                            size_t size, gfp_t flags)
{
    unsigned long redzone_start = round_up((unsigned long)object + size,
                                           KASAN_GRANULE_SIZE);
    unsigned long redzone_end = round_up((unsigned long)object +
                                          cache->object_size,
                                          KASAN_GRANULE_SIZE);
    kasan_unpoison_memory(object, size);
    kasan_poison_memory((void *)redzone_start,
                         redzone_end - redzone_start,
                         KASAN_KMALLOC_REDZONE);
    return (void *)object;
}
```

### KASAN Reports

A KASAN report looks like:

```
==================================================================
BUG: KASAN: slab-out-of-bounds in copy_from_user_nofault+0x...
Write of size 4 at addr ffff888102b4e7f0 by task syzbot/1234

CPU: 0 PID: 1234 Comm: syzbot Not tainted
Hardware name: QEMU Standard PC (Q35 + ICH9, 2009)
Call Trace:
 kasan_report+0x...
 copy_from_user_nofault+0x...
 ...

Allocated by task 1234:
 kmalloc_trace+0x...
 ...

Freed by task 0:
 (not freed)

The buggy address belongs to the object at ffff888102b4e7e0
 which belongs to the cache kmalloc-32 of size 32
The buggy address is located 16 bytes inside of
 32-byte region [ffff888102b4e7e0, ffff888102b4e800)
==================================================================
```

### Stack Instrumentation

`CONFIG_KASAN_STACK=y` adds stack frame redzones. Clang's `-fsanitize=kernel-address` flag, combined with `-fstack-protector`, inserts fake allocas with redzone poisoning for local variable arrays:

```c
// For: char buf[64];
// Clang generates:
kasan_unpoison_memory(buf, 64);
// ... use buf ...
kasan_poison_memory(buf, 64, KASAN_STACK_MID);  // at scope exit
```

---

## 113.3 KFENCE — Kernel Electric Fence

### Design Philosophy

KFENCE (Kernel Electric Fence) trades detection coverage for near-zero production overhead. Rather than monitoring every allocation, KFENCE intercepts a small random fraction (by default, one every 100ms) and places that allocation in a specially protected pool where hardware page faults serve as the detection mechanism.

This makes KFENCE suitable for running permanently in production kernels — including on servers and embedded devices where KASAN's 2–5× overhead is unacceptable. The trade-off is probabilistic detection: bugs that affect only non-sampled allocations go undetected.

### The Fenced Pool

KFENCE maintains a static pool of memory during `__init`:

```
KFENCE_POOL:
┌──────────────────────────────────────────────────────┐
│ guard  │ object1 │ guard  │ object2 │ guard  │ ...   │
│ (page) │ (≤page) │ (page) │ (≤page) │ (page) │       │
└──────────────────────────────────────────────────────┘
  ^NO-PTE  ^R/W       ^NO-PTE  ^R/W      ^NO-PTE
```

Each "slot" is two pages: one guard page (unmapped, PTE absent) followed by one object page. The allocated object is placed either at the start of the page (left-aligned, for overflow detection) or at the end (right-aligned, for underflow / left-overflow detection). The choice can be randomized.

Any access beyond the object boundary hits the guard page and causes a hardware page fault, which `kfence_handle_page_fault()` catches before the normal fault handler:

```c
// arch/x86/mm/fault.c (simplified):
bool __kfence_handle_page_fault(unsigned long addr)
{
    if (!is_kfence_address((void *)addr))
        return false;
    kfence_report_error(addr, ...);
    return true;
}
```

### Sampling Mechanism

A `hrtimer` fires every `kfence.sample_interval` milliseconds (default 100ms). On each timer expiry, KFENCE marks one slot as "available for next allocation." When the next `kmalloc` call arrives (from any code path), `kfence_alloc()` intercepts it and fulfills the allocation from the available slot:

```c
// mm/kfence/core.c:
static void *kfence_alloc(struct kmem_cache *s, size_t size, gfp_t flags)
{
    struct kfence_metadata *meta;
    if (!READ_ONCE(kfence_allocation_gate))
        return NULL;  // no slot available this cycle
    if (!atomic_try_cmpxchg(&kfence_allocation_gate, &val, 0))
        return NULL;
    meta = kfence_get_meta_for_alloc(size);
    if (!meta)
        return NULL;
    return kfence_setup_meta(meta, s, size);
}
```

The `kfence_allocation_gate` is an atomic flag set by the timer and cleared on first allocation. This ensures exactly one KFENCE allocation per sample interval (approximately).

### Use-After-Free Detection

KFENCE also detects use-after-free. When a KFENCE-allocated object is freed, KFENCE:

1. Records the free stack trace in the object's metadata.
2. Poisons the object's page with a distinct pattern (`0xCD`).
3. Removes the page's PTE (makes it inaccessible).

Any subsequent access to the freed object faults, and `kfence_handle_page_fault()` reports a use-after-free with both the allocation and free stack traces.

### Boot Parameter

```bash
# Disable KFENCE at boot:
kfence.sample_interval=0

# Sample every 50ms (more aggressive):
kfence.sample_interval=50

# Status:
cat /sys/kernel/debug/kfence/stats
```

---

## 113.4 KCSAN — Kernel Concurrency Sanitizer

### The Data Race Problem in the Kernel

The Linux kernel uses fine-grained locking, lock-free algorithms, RCU, seqlocks, and atomic operations for concurrency. Data races — concurrent unsynchronized accesses where at least one is a write — are undefined behavior in C and cause real bugs (torn reads, missing memory barriers, compiler reordering). ThreadSanitizer (TSan) handles userspace; KCSAN handles the kernel.

### KCSAN's Watchpoint Algorithm

TSan uses vector clocks, which require O(threads) space per object and are too expensive for kernel use. KCSAN instead uses a sampling-based watchpoint approach:

1. On every memory access, KCSAN (with some probability) installs a **watchpoint** for that address in a small global watchpoint array.
2. Subsequent accesses to the same address — from any CPU — check whether a watchpoint exists for that address.
3. If an access finds an existing watchpoint pointing to the same memory region, and the concurrent access is a write (or both are writes), KCSAN reports a data race.

The watchpoint array is intentionally small (default 64 entries). This makes KCSAN a **sampling** detector: it catches races that happen to activate a watchpoint and a concurrent access in the same window. It will miss races that are never sampled, but over time on a heavily exercised kernel it finds genuine races.

### Instrumentation

Clang with `-fsanitize=thread` (for the kernel: `-fsanitize=kernel-thread`) inserts calls to KCSAN's access functions:

```c
// For: x = shared_var;
__tsan_read4(&shared_var);
x = shared_var;

// For: shared_var = y;
__tsan_write4(&shared_var);
shared_var = y;
```

KCSAN implements these functions in `kernel/kcsan/core.c`. The key function `kcsan_setup_watchpoint()`:

```c
static noinline void kcsan_setup_watchpoint(const volatile void *ptr,
                                             int size, int type)
{
    // With probability 1/CONFIG_KCSAN_SAMPLE_INTERVAL:
    long *slot = find_watchpoint_slot(ptr);
    if (slot == NULL) return;  // no free slot
    // Install watchpoint (encoded address + size):
    WRITE_ONCE(*slot, encode_watchpoint(ptr, size, type));
    // Small delay: let concurrent accesses arrive
    udelay(CONFIG_KCSAN_UDELAY_TASK);  // default ~10µs
    // Check if watchpoint was hit:
    if (READ_ONCE(*slot) != encode_watchpoint(ptr, size, type))
        kcsan_report_race(ptr, size, type);
    // Remove watchpoint:
    WRITE_ONCE(*slot, INVALID_WATCHPOINT);
}
```

The delay is the key insight: by sleeping for a brief period after installing the watchpoint, KCSAN gives concurrent accesses time to trigger the watchpoint. If another CPU writes the same address during this window, it will find the watchpoint and fire a race report.

### Benign Races and Annotations

Not all concurrent accesses are bugs. Linux uses `READ_ONCE()` and `WRITE_ONCE()` to document intentional racy accesses (which the compiler must not optimize). KCSAN understands these annotations:

```c
// KCSAN skips instrumentation for READ_ONCE/WRITE_ONCE:
x = READ_ONCE(shared_var);     // intentional racy read
WRITE_ONCE(shared_var, val);   // intentional racy write

// data_race() macro: mark a block as intentionally racy:
data_race(shared_var++);

// kcsan_nestable_atomic_begin/end: atomic context (IRQ disabled):
kcsan_nestable_atomic_begin();
shared_var++;   // inside lock, no race possible
kcsan_nestable_atomic_end();
```

### KCSAN Report

```
==================================================================
BUG: KCSAN: data-race in process_one_work / worker_thread

write to 0xffff8881027e3a18 of 8 bytes by task 42 on cpu 1:
 process_one_work+0x284/0x760
 worker_thread+0x14c/0x4b0
 kthread+0x168/0x1b0

read to 0xffff8881027e3a18 of 8 bytes by task 43 on cpu 0:
 worker_thread+0x1a0/0x4b0
 kthread+0x168/0x1b0
==================================================================
```

---

## 113.5 KMSAN — Kernel MemorySanitizer

### Overview

KMSAN (Kernel MemorySanitizer) detects use of uninitialized kernel memory. This matters particularly for information leaks: when the kernel copies uninitialized bytes to userspace via `copy_to_user()`, those bytes may contain sensitive kernel data (pointer values, stack contents from prior operations, cryptographic material).

KMSAN is a direct port of LLVM's MemorySanitizer to the kernel. It requires Clang (GCC does not implement the MSan instrumentation interface).

### Shadow and Origin Tracking

Like MSan, KMSAN maintains two shadow regions for every byte of kernel memory:

1. **Shadow memory**: 1 bit per memory bit (or 1 byte per byte for simplicity), indicating whether the byte is initialized (`0`) or uninitialized (`1`).
2. **Origin memory**: 4 bytes per 4-byte group, storing a compressed stack trace ID identifying *where* the uninitialized memory was allocated.

The origin tracking enables KMSAN to report not just "you used uninitialized memory here" but also "this memory was allocated at this stack frame and never initialized."

```c
// mm/kmsan/kmsan_shadow.c:
struct kmsan_shadow_origin_pair kmsan_get_shadow_origin(void *addr, int size)
{
    void *shadow = kmsan_get_shadow_address(addr);
    u32 *origin = kmsan_get_origin_address(addr);
    return (struct kmsan_shadow_origin_pair){ shadow, origin };
}
```

### Instrumentation Interface

Clang generates calls to KMSAN's runtime on every uninitialized-value use:

```c
// For: int x = uninitialized_var;
// Clang adds shadow check and origin propagation:
__msan_unpoison_stack_with_origin(dst, size);
__msan_check_mem_is_initialized(dst, size);
```

KMSAN implements these in `mm/kmsan/kmsan.c`. The key check:

```c
void kmsan_check_memory(const void *addr, size_t size)
{
    void *shadow = kmsan_get_shadow(addr);
    // If any shadow byte is non-zero, memory is uninitialized:
    for (size_t i = 0; i < size; i++) {
        if (((u8 *)shadow)[i] != 0) {
            kmsan_report(addr, size, /*is_store=*/false, _RET_IP_);
            return;
        }
    }
}
```

### Userspace Leak Detection

The most important KMSAN feature is detecting kernel-to-userspace information leaks. Every `copy_to_user()` call is wrapped:

```c
// arch/x86/lib/copy_user_64.S calls into:
long __kmsan_copy_to_user(void __user *to, const void *from, unsigned long n)
{
    // Check that 'from' has no uninitialized bytes:
    kmsan_check_memory(from, n);
    // Then perform the actual copy:
    return raw_copy_to_user(to, from, n);
}
```

This catches bugs where kernel struct fields have padding bytes or where functions return structs with uninitialized members that then leak to userspace — a common source of information disclosure vulnerabilities.

### Practical Use

KMSAN overhead is 3–5× slower than normal kernel execution, comparable to userspace MSan. It is not suitable for production use but is valuable in CI with a broad test suite:

```bash
# Build kernel with KMSAN:
make CC=clang-22 LLVM=1 CONFIG_KMSAN=y

# Boot with QEMU and run tests:
./run_tests.sh 2>&1 | grep "KMSAN:"
```

---

## 113.6 Kernel UBSan

### Overview

Kernel UBSan applies Clang's UndefinedBehaviorSanitizer to kernel code. The kernel's `lib/ubsan.c` provides a custom runtime that replaces the userspace `libubsan`. Reports go through `pr_err()` with a stack trace rather than calling `abort()`.

### Enabled Checks

```
CONFIG_UBSAN_BOUNDS         — array index out of bounds
CONFIG_UBSAN_SHIFT          — shift by negative / shift >= width
CONFIG_UBSAN_INTEGER_WRAP   — signed integer overflow
CONFIG_UBSAN_UNREACHABLE    — __builtin_unreachable() reached
CONFIG_UBSAN_BOOL           — invalid bool value
CONFIG_UBSAN_ENUM           — invalid enum value
CONFIG_UBSAN_ALIGNMENT      — misaligned pointer dereference (costly)
```

Signed integer overflow is disabled on some kernel builds because certain kernel subsystems deliberately rely on two's complement wraparound behavior that predates strict C UB rules; these can be annotated with `__no_sanitize_undefined` or the code can be fixed.

### The Kernel UBSan Runtime

```c
// lib/ubsan.c (simplified):
void __ubsan_handle_out_of_bounds(struct out_of_bounds_data *data,
                                   unsigned long index)
{
    unsigned long flags;
    if (suppress_report(&data->location))
        return;
    ubsan_prologue(&data->location, "array-index-out-of-bounds", &flags);
    pr_err("index %lu is out of range for type '%s'\n",
           index, data->array_type->type_name);
    ubsan_epilogue(&flags);
}
```

The `suppress_report()` mechanism uses a static key to prevent the same location from reporting more than once, avoiding log spam.

### Trap Mode

`CONFIG_UBSAN_TRAP=y` replaces all UBSan runtime calls with a trap instruction (`UD2` on x86, `BRK #5` on AArch64). This causes an immediate kernel oops on any undefined behavior detection, with no recovery. It is suitable for hardening production kernels where you want to abort rather than log-and-continue:

```bash
# Clang flag equivalent:
clang -fsanitize=undefined -fsanitize-undefined-trap-on-error ...
```

The trap approach has near-zero overhead on the non-triggered path (no function calls, just a potential branch to `UD2`), making it viable for production use on certain checks (array bounds, alignment).

---

## 113.7 CFI in the Kernel

### Control Flow Integrity Overview

CFI prevents attackers from hijacking indirect calls and vtable dispatches to arbitrary code. In the kernel, CFI is especially important because an attacker with kernel write primitives might redirect function pointers to arbitrary locations. Linux kernel CFI uses Clang's `kcfi` sanitizer, which is distinct from the userspace `cfi` sanitizer.

### KCFI vs Userspace CFI

Userspace CFI (`-fsanitize=cfi`) uses a jump table scheme where every indirect call is redirected through a verified jump table. This requires all function addresses to be jump table entries, breaking the kernel's use of function pointers directly as values.

KCFI (`CONFIG_CFI_CLANG=y`) uses an inline type hash approach:

```c
// For each function, Clang inserts a type hash before the entry point:
// (emitted just before function symbol)
.4byte __kcfi_typeid_<mangled_type>   // 32-bit type hash

// Before each indirect call, Clang inserts:
movl -4(%callee_ptr), %ecx           // load hash from before entry
cmpl $EXPECTED_HASH, %ecx            // compare with expected type hash
jne  __cfi_check_failed              // mismatch → CFI violation
call *%callee_ptr                    // actual call
```

This requires position-dependent code (callee's 4 bytes before `entry_point` must be accessible and not executable), which is satisfied by the kernel's code layout.

### Shadow Call Stack

`CONFIG_SHADOW_CALL_STACK=y` (AArch64 only) implements a return address protection scheme that complements CFI:

- The `x18` register is reserved as the Shadow Call Stack Pointer.
- On function entry (in the function prologue), `lr` (link register) is pushed to `[x18]` and `x18` incremented.
- On return, the saved `lr` is popped from the shadow stack and compared to the current `lr`. Mismatch → panic.

This prevents return-oriented programming (ROP) attacks that overwrite return addresses on the normal stack:

```asm
// SCS prologue (AArch64):
str  x30, [x18], #8     // push lr to shadow stack, advance x18
// ... function body ...
// SCS epilogue:
ldr  x30, [x18, #-8]!   // pop from shadow stack, restore x18
ret
```

### Kernel Module CFI

Kernel modules must be compiled with the same CFI configuration as the kernel itself. Module type hashes must match the types registered in the running kernel. KCFI enforces this: loading a module compiled without CFI against a CFI-enabled kernel fails with a type hash mismatch on the first indirect call from that module.

---

## 113.8 Workflow: Finding Bugs with Kernel Sanitizers

### syzkaller: Coverage-Guided Kernel Fuzzing

[syzkaller](https://github.com/google/syzkaller) is Google's coverage-guided syscall fuzzer for the Linux kernel. It requires:

- **KCOV** (`CONFIG_KCOV=y`): per-task code coverage collection. Each task has a coverage buffer; after each syscall, syzkaller reads the set of covered basic blocks.
- **KASAN** or **KCSAN** for bug detection.
- A QEMU VM running the sanitizer-enabled kernel.

```
syzkaller architecture:
┌───────────────────────────────────────────────────────────────┐
│ syz-manager (host)                                            │
│  ├── Corpus management (input database)                       │
│  ├── Coverage-guided mutation (maximize new basic blocks)     │
│  └── Crash detection (KASAN/KCSAN output)                     │
│                                                               │
│ syz-executor (in QEMU guest)                                  │
│  ├── Executes syscall programs                                │
│  ├── Reads KCOV coverage after each execution                 │
│  └── Reports crashes to syz-manager                           │
└───────────────────────────────────────────────────────────────┘
```

### syzbot: Continuous Fuzzing

Google runs syzbot, a public continuous fuzzing infrastructure that runs syzkaller 24/7 on the latest kernel mainline and -next trees. When a sanitizer report is triggered:

1. syzbot records the sanitizer report.
2. Automatically attempts to find a minimal reproducer.
3. Files a public bug report on the kernel mailing list with title, report, and `syz repro`.
4. Tracks whether a patch fixes the issue.

The syzbot dashboard at `syzkaller.appspot.com` shows hundreds of active bugs at any given time, the vast majority found via KASAN, KCSAN, or KMSAN.

### Typical Development Workflow

```bash
# 1. Build kernel with sanitizers:
make defconfig
scripts/config -e KASAN -e KASAN_GENERIC -e KCOV -e DEBUG_INFO
make CC=clang-22 LLVM=1 -j$(nproc) bzImage

# 2. Create QEMU VM image (Debian/Buildroot):
./create-image.sh

# 3. Boot with sanitizers:
qemu-system-x86_64 \
  -kernel arch/x86/boot/bzImage \
  -append "console=ttyS0 kasan.fault=panic" \
  -drive file=bullseye.img,format=raw \
  -net user,host=10.0.2.10,hostfwd=tcp::10022-:22 \
  -net nic -nographic

# 4. Run a specific test or fuzzer:
ssh -p 10022 root@localhost /root/syz-executor

# 5. Read KASAN reports from dmesg:
dmesg | grep -A 40 "BUG: KASAN"
```

### Report to Fix Pipeline

When a sanitizer report fires, the typical information includes:

1. **Bug type**: `slab-out-of-bounds`, `use-after-free`, `data-race`, `uninitialized-value`
2. **Access address** and its relationship to known kernel objects (SLUB metadata identifies the slab cache)
3. **Call stack at the bug site**: the instruction that caused the bug
4. **Allocation stack**: where the affected memory was allocated
5. **Free stack** (for UAF): where it was freed

This information is usually sufficient to identify the bug. KASAN in particular tends to produce reports with enough detail to write a fix without needing to run the reproducer manually.

---

## Chapter Summary

- **KASAN** adapts userspace ASan to the kernel with three modes: generic software (portable, highest overhead), software tag-based (AArch64 TBI, ~15% overhead), and hardware tag-based (AArch64 MTE, near-zero overhead). It instruments every load/store and integrates with the SLUB slab allocator.

- **KFENCE** is a production-grade complement to KASAN: it uses hardware page faults to detect bugs in a sampled 1-in-N allocation set, with essentially zero overhead on non-sampled paths. Suitable for always-on deployment.

- **KCSAN** detects kernel data races using a watchpoint-based sampling algorithm. It installs a watchpoint on a random subset of memory accesses and detects concurrent conflicting accesses during a brief delay window. Understands `READ_ONCE`/`WRITE_ONCE` as benign annotations.

- **KMSAN** detects uninitialized memory use, especially dangerous when uninitialized kernel data reaches `copy_to_user()`. Requires Clang; overhead is 3–5× due to shadow and origin tracking.

- **Kernel UBSan** catches undefined behavior such as out-of-bounds array accesses, shift violations, and integer overflow. `CONFIG_UBSAN_TRAP` mode replaces runtime calls with trap instructions for near-zero overhead.

- **KCFI** provides Control Flow Integrity via inline type hash checks before indirect calls. **Shadow Call Stack** (`x18`-based) complements CFI against ROP attacks on AArch64.

- **syzkaller + syzbot** use `CONFIG_KCOV` for coverage-guided kernel fuzzing, combined with KASAN/KCSAN for bug detection. This infrastructure drives the majority of kernel bug reports in the Linux security community.


---

@copyright jreuben11
