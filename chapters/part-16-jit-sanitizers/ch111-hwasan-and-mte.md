# Chapter 111 — HWASan and MTE

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

Software AddressSanitizer works, but at a price: a 2–3× slowdown and doubled memory consumption make it unsuitable for anything but development builds and CI runs. For language runtimes, server workloads under continuous monitoring, and production mobile applications where memory safety bugs have security consequences, this overhead is unacceptable. Hardware-accelerated AddressSanitizer (HWASan) and the ARMv8.5-A Memory Tagging Extension (MTE) represent a qualitatively different point on the safety/overhead trade-off curve. Both exploit a fundamental property of AArch64 addressing — the top byte of a virtual address is ignored by the MMU — to encode a tag alongside the pointer, enabling per-16-byte-granule memory safety at 10–15% overhead for HWASan and 2–5% for MTE. This chapter covers HWASan's software-tag approach, the hardware support provided by MTE, their integration into Clang and the LLVM backend, stack use-after-return detection without fake stacks, and practical deployment patterns in production systems.

---

## 111.1 Motivation: Software ASan Overhead

### 111.1.1 The Production Monitoring Gap

Software ASan's overhead profile makes it a development tool, not a production monitoring tool:

- **CPU**: 2–3× slowdown on typical mixed workloads. A server handling 50,000 req/s drops to ~20,000 req/s.
- **Memory**: 2× application memory plus red zones and quarantine. A 2GB server process requires 4GB+ with ASan.
- **Startup cost**: The 16TB shadow mapping on 64-bit Linux is reserved at startup (virtual address space, not physical memory), but still consumes `vm.overcommit` budget.

A production C++ service cannot run with ASan enabled. Yet the most dangerous bugs — use-after-free and out-of-bounds heap access — are exactly those that appear only under production load patterns not replicated in tests, and that have the most severe security consequences.

### 111.1.2 AArch64 Top Byte Ignore (TBI)

AArch64 defines **Top Byte Ignore** (TBI): bits [63:56] of a virtual address are ignored by the MMU when performing address translation for data accesses. The hardware transparently masks out these bits on every load and store. User-space code can write arbitrary values into the top byte of any pointer without affecting memory access behavior.

This makes the top byte of every 64-bit pointer available for application metadata at zero additional cost — if the check can be made inexpensive. Both HWASan and MTE exploit this.

```
AArch64 virtual address (64 bits):
 ┌──────────┬─────────────────────────────────────────────┐
 │ TAG [63:56] │        VIRTUAL ADDRESS [55:0]            │
 └──────────┴─────────────────────────────────────────────┘
     8 bits             56 bits (addressing)
  (ignored by MMU      (used for virtual→physical translation)
   on data access)
```

Linux uses `PR_SET_TAGGED_ADDR_CTRL` with `PR_TAGGED_ADDR_ENABLE` to enable TBI for a process. Without this `prctl`, the kernel will SIGBUS a process that uses non-zero top bytes in pointers — so TBI must be explicitly enabled at process startup.

### 111.1.3 Design Space Overview

Three points on the safety/overhead trade-off:

| Property | Software ASan | HWASan | MTE |
|----------|--------------|--------|-----|
| Tag bits | 1 bit (poisoned) | 8 bits | 4 bits |
| Tag granule | 8 bytes | 16 bytes | 16 bytes |
| False negative / access | ~0% | 1/256 | 1/16 |
| CPU overhead | 2–3× | 10–15% | 2–5% (sync) |
| Memory overhead | 2× + red zones | ~15% | ~1% (HW tag RAM) |
| Hardware requirement | None | TBI-capable | ARMv8.5-A MTE |
| Production viable | No | Yes (mobile apps) | Yes (server/mobile) |

---

## 111.2 HWASan: Architecture Overview

### 111.2.1 Tag Model

HWASan assigns an 8-bit tag to every pointer and every 16-byte **granule** of heap/stack memory. The pointer's tag is stored in bits [63:56] (TBI). The granule's expected tag is stored in a compact shadow map:

```
Shadow mapping:
  shadow_addr = (virt_addr >> 4) + shadow_base
  1 byte of shadow per 16-byte granule
  shadow_byte = expected tag for that granule

Example:
  Allocation at 0x0000602000000010 (size 32 bytes, 2 granules)
  Tag assigned: 0x5A
  Pointer returned: 0x5A00602000000010
  
  Shadow at shadow_base + (0x602000000010 >> 4):
    Byte 0: 0x5A  (granule 0: address 0x602000000010..0x60200000001F)
    Byte 1: 0x5A  (granule 1: address 0x602000000020..0x60200000002F)
  Adjacent granules:
    Byte -1: 0x00 (unallocated/untagged)
    Byte 2:  0x3B (different allocation with different tag)
```

### 111.2.2 Check Sequence

The HWASan check on AArch64 (software mode, before hardware acceleration):

```asm
; Check for 8-byte load at x0 (tagged pointer):
mov  x9, x0                    ; x9 = full pointer (with tag)
ubfx x10, x0, #56, #8          ; x10 = pointer tag (bits [63:56])
lsr  x11, x0, #4               ; x11 = address >> 4 (shadow offset)
ldrb w11, [x11, shadow_base]   ; w11 = shadow byte (expected tag)
cmp  w10, w11
b.ne __hwasan_tag_mismatch4    ; mismatch → error handler
; Original load uses x0; TBI means tag in x0[63:56] is ignored by hardware
ldr  x0, [x0]                  ; the actual load
```

The `__hwasan_tag_mismatch` function is in `compiler-rt/lib/hwasan/hwasan.cpp`. It determines the error type (overflow/UAF/stack UAF) and produces a detailed report including the allocation and deallocation stack traces.

On hardware with FEAT_MEMTAG (ARMv8.5-A MTE), this software check is replaced by a single tagged load/store instruction that performs the check in hardware (Section 111.4).

### 111.2.3 Overhead Analysis

The software shadow check adds 4–5 instructions per memory access. On a modern AArch64 out-of-order processor:
- Shadow load: 1 cache miss on first access, ~4 cycles on cache hit (shadow is dense = cache-friendly)
- Branch: predicted not-taken (99.999% of accesses are valid)
- UBFX: single-cycle integer operation

Total: ~5% overhead for memory-intensive code; negligible for I/O-bound code. Memory overhead: 1 byte per 16-byte granule = 6.25% of application memory. Plus per-chunk metadata in the HWASan allocator (~8 bytes per allocation).

---

## 111.3 HWASan Implementation

### 111.3.1 HWAddressSanitizerPass

[`HWAddressSanitizerPass`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Instrumentation/HWAddressSanitizer.cpp) is the LLVM IR pass. It performs three main transformations:

**Load/store instrumentation**: For each load/store, inserts the shadow check sequence described above. The check is inlined rather than called via function pointer, for better branch prediction feedback.

**Stack frame tagging**: Instruments function entry/exit to tag stack variables (Section 111.3.3).

**Pointer tagging for allocas**: Rewrites the `alloca` instruction to return a tagged pointer. All uses of the alloca that are GEPs (pointer arithmetic) must preserve the tag — HWASan inserts `orr x, x, tag_imm` to restore the tag after GEPs that may have affected the top byte.

```bash
# Enable HWASan:
clang -fsanitize=hwaddress -fno-omit-frame-pointer -g -O1 test.c -o test
# AArch64 hardware, or QEMU with -cpu max,tagged-addr=on:
HWASAN_OPTIONS=max_malloc_fill_size=0 ./test
```

### 111.3.2 HWASan Heap Allocation

The HWASan allocator ([`compiler-rt/lib/hwasan/hwasan_allocator.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/lib/hwasan/hwasan_allocator.cpp)):

```cpp
void *__hwasan_malloc(uptr size) {
  // 1. Allocate aligned memory (aligned to 16 bytes for granule boundary):
  uptr real_size = RoundUpTo(size, 16);  // ensure full granules
  void *ptr = BackendAllocate(real_size);
  
  // 2. Generate a random 8-bit tag (tag != 0, tag != prev_tag for UAF):
  uint8_t tag = GetRandomTag();  // uses thread-local random state
  // Ensure the tag is different from the tag of adjacent freed memory:
  while (tag == 0 || tag == GetAllocSiteTag(ptr)) tag = GetRandomTag();
  
  // 3. Store the tag in the shadow for each 16-byte granule:
  TagMemory(ptr, real_size, tag);
  // TagMemory writes `tag` to shadow_base + ((uptr)ptr >> 4) for each granule
  
  // 4. Return a tagged pointer:
  return TagPointer(ptr, tag);  // sets ptr[63:56] = tag
}

void __hwasan_free(void *tagged_ptr) {
  uint8_t old_tag = GetTag(tagged_ptr);         // extract tag from top byte
  void *ptr = UntagPointer(tagged_ptr);         // clear top byte
  uptr size = GetChunkSize(ptr);
  
  // 5. Retag freed memory with a different random tag (UAF detection):
  uint8_t free_tag;
  do { free_tag = GetRandomTag(); } while (free_tag == old_tag || free_tag == 0);
  TagMemory(ptr, size, free_tag);
  
  // 6. Free with untagged pointer:
  BackendFree(ptr);
}
```

The retagging on free means that any subsequent access to the freed pointer (which has `old_tag` in its top byte) will fail the shadow check (shadow now contains `free_tag`). Detection probability per invalid access = 255/256 ≈ 99.6%.

### 111.3.3 Stack Tagging

[`HWAddressSanitizer::instrumentStack`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Instrumentation/HWAddressSanitizer.cpp) tags stack allocations:

**Prologue**:
1. Use a random per-frame tag (derived from the stack pointer at function entry for reproducibility)
2. Write the tag to the shadow for each alloca's granules
3. Return a tagged pointer for each alloca (tag in top byte)

**Epilogue** (including all early-return paths):
1. Write tag 0 to the shadow for all tagged allocas — any escaped pointer now points to "untagged" memory; the next granule-boundary access with the old tag will fault

```asm
; Simplified HWASan stack frame (AArch64, single local variable):
; prologue:
sub  sp, sp, #32            ; allocate frame (2 granules)
mrs  x9, tpidr_el0          ; read thread-local register (contains tag seed)
eor  x9, x9, sp             ; mix with SP for randomness
and  x9, x9, #0xFF          ; keep only 8 bits as tag
orr  x9, x9, #1             ; ensure tag != 0
lsl  x10, x9, #56           ; put tag in top byte position
lsr  x11, sp, #4            ; granule index for sp
add  x12, x11, shadow_base
strb w9, [x12]              ; tag granule 0
strb w9, [x12, #1]          ; tag granule 1
orr  x1, sp, x10            ; x1 = tagged pointer to local variable at sp

; epilogue:
strb wzr, [x12]             ; clear tag for granule 0 (tag 0 = untagged)
strb wzr, [x12, #1]         ; clear tag for granule 1
add  sp, sp, #32
ret
```

At function return, `x1` (or any other escaped pointer to the stack variable) has `old_tag` in its top byte, but the shadow now has 0. A subsequent access via the escaped pointer triggers `__hwasan_tag_mismatch`, catching use-after-return.

### 111.3.4 x86_64 HWASan Mode

On x86_64, TBI is not available (the hardware uses all 64 bits for addressing). HWASan emulates TBI via an alternative: it uses bits [62:56] (7 bits, avoiding the kernel/user space split bit [63]) and performs an explicit mask operation on every pointer use. x86_64 HWASan is primarily used for cross-platform development where the developer wants identical behavior on both AArch64 (production) and x86_64 (development). Performance is worse on x86_64 due to the lack of hardware TBI, but the API and detection logic are identical.

---

## 111.4 MTE: Memory Tagging Extension

### 111.4.1 ARMv8.5-A Hardware Structure

Memory Tagging Extension (MTE) is a hardware feature in ARMv8.5-A. Unlike HWASan's software shadow, MTE stores tags in dedicated hardware storage:

- **Allocation Tag**: 4 bits per 16-byte granule, stored in a special tag memory accessible via new instructions
- **Logical Address Tag**: bits [59:56] of the virtual address (4 bits, inside the TBI region)
- **Check**: the hardware compares the Logical Address Tag with the Allocation Tag on every tagged load/store; a mismatch triggers a fault

The tag memory is separate from the page tables and main DRAM. On current ARM implementations, it is stored in spare ECC bits of the main memory — the 4-bit tag per 16-byte granule costs 1 extra bit per 32 bytes of data, which fits in the typically unused ECC bits of LPDDR5/DDR5.

### 111.4.2 MTE Instructions

```asm
; IRG (Insert Random Tag): generate a random 4-bit tag for a pointer
irg  Xd, Xn, Xm     ; Xd = Xn with random tag in [59:56]; Xm is exclusion mask

; Example: generate random tag excluding tag 0 and tag 5:
mov  x10, #0b10001   ; bitmask: exclude tag 0 (bit 0) and tag 5 (bit 5) → wait,
                     ; actually bits in Xm correspond to tags: bit 0 = exclude tag 0
irg  x0, x0, x10    ; x0 gets new pointer with random tag, excluding tags 0 and 4

; ADDG (Add with Tag): add offset and optionally increment tag
addg Xd, Xn, #imm6, #imm4   ; Xd = Xn + imm6, with tag = (Xn.tag + imm4) & 0xF

; STG (Store Allocation Tag): store tag to granule
stg  [Xn, #imm]     ; tag of address Xn (bits[59:56]) → allocation tag of granule at Xn

; ST2G: store tag to two consecutive granules
st2g [Xn, #imm]

; STZG (Store Tag and Zero): store tag AND zero the granule content
stzg [Xn, #imm]     ; tag Xn's granule AND write 16 zero bytes

; STZ2G: store tag and zero for two granules
stz2g [Xn, #imm]

; LDG (Load Allocation Tag): load tag from granule into pointer
ldg  Xt, [Xn]       ; Xt = Xn with Xt[59:56] = allocation tag of granule at Xn

; GVA (Generate/Clean Virtual Address tags): bulk clean
dc   gva, Xn        ; clean allocation tags for cache line at Xn (privileged)
dc   gzva, Xn       ; clean and zero

; STGM/LDGM: store/load all tags for a 64KB block (EL1 only)
stgm Xt, [Xn]       ; privileged: write tag for all granules in 64KB block
ldgm Xt, [Xn]       ; privileged: read all tags for a 64KB block
```

The `PSTATE.TCO` bit (Tag Check Override) can be set to temporarily disable tag checking for specific code regions (e.g., bulk memory operations in the kernel).

### 111.4.3 Linux Kernel Support

Linux exposes MTE to user space via:

**`getauxval(AT_HWCAP2) & HWCAP2_MTE`**: non-zero if the CPU supports MTE.

**`prctl(PR_SET_TAGGED_ADDR_CTRL, flags, 0, 0, 0)`**: enables tagged addresses for the current thread:

```c
#include <sys/prctl.h>
#include <linux/prctl.h>

// Enable synchronous MTE (for debugging):
int rc = prctl(PR_SET_TAGGED_ADDR_CTRL,
               PR_TAGGED_ADDR_ENABLE  |  // enable TBI
               PR_MTE_TCF_SYNC        |  // synchronous fault mode
               (0xfffe << PR_MTE_TAG_SHIFT),  // exclude tag 0 from IRG
               0, 0, 0);
// PR_MTE_TAG_SHIFT is typically 3; 0xfffe << 3 excludes bit 0 (tag 0) from IRG

// Enable asynchronous MTE (for production):
rc = prctl(PR_SET_TAGGED_ADDR_CTRL,
           PR_TAGGED_ADDR_ENABLE  |
           PR_MTE_TCF_ASYNC       |  // async mode: buffer faults
           (0xfffe << PR_MTE_TAG_SHIFT),
           0, 0, 0);
```

**`mmap(addr, size, PROT_READ|PROT_WRITE|PROT_MTE, MAP_ANON|MAP_PRIVATE, -1, 0)`**: allocates pages with MTE tag storage enabled. Without `PROT_MTE`, tagging instructions have no effect on the page.

**`SIGSEGV` with `si_code = SEGV_MTESERR`**: synchronous tag check fault. The `siginfo_t.si_addr` is the exact faulting address with the logical address tag in [59:56].

**`SIGSEGV` with `si_code = SEGV_MTEAERR`**: asynchronous mode fault. The `si_addr` may not be exact.

### 111.4.4 Synchronous vs Asynchronous Modes

**Synchronous mode** (`PR_MTE_TCF_SYNC`):
- Every tagged load/store performs the tag check in the CPU's execute pipeline
- A mismatch stalls the pipeline and delivers `SIGSEGV/SEGV_MTESERR` immediately
- `si_addr` contains the exact faulting virtual address (with logical tag)
- CPU overhead: ~3–5% (the check is on the critical path)
- Use: debugging and testing where exact fault addresses are needed

**Asynchronous mode** (`PR_MTE_TCF_ASYNC`):
- The tag check is done speculatively; the CPU may execute multiple load/stores before reporting a fault
- Faults are batched in a per-core register and delivered on a context switch, system call, or `MRS TFSRE0_EL1` read
- `si_addr` is approximate (the program counter at delivery time, not at the faulting instruction)
- CPU overhead: ~1–2% (check is off critical latency path)
- Use: production monitoring where the bug class matters more than exact fault location

**Asymmetric mode** (`PR_MTE_TCF_ASYMM`, ARMv8.7+):
- Stores are checked synchronously (no overhead for reads)
- Loads are checked asynchronously (low overhead)
- Catches buffer overflows via stores accurately; UAF via loads approximately
- Best production/security trade-off for write-heavy workloads

---

## 111.5 Clang/LLVM MTE Integration

### 111.5.1 Clang Flags

```bash
# Stack tagging only (prologue/epilogue STG instructions):
clang -fsanitize=memtag-stack test.c -target aarch64-linux-gnu

# Heap tagging (allocator must be MTE-aware):
clang -fsanitize=memtag-heap test.c -target aarch64-linux-gnu

# Global variable tagging:
clang -fsanitize=memtag-globals test.c -target aarch64-linux-gnu

# All three:
clang -fsanitize=memtag test.c -target aarch64-linux-gnu

# Set the MTE mode:
clang -fsanitize=memtag -fsanitize-memtag-mode=sync test.c   # debugging
clang -fsanitize=memtag -fsanitize-memtag-mode=async test.c  # production
clang -fsanitize=memtag -fsanitize-memtag-mode=asymm test.c  # ARMv8.7+

# Enable MTE-relevant CPU feature:
clang -march=armv8.5-a+memtag -fsanitize=memtag test.c
```

### 111.5.2 AArch64StackTaggingPass

[`AArch64StackTaggingPass`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AArch64/AArch64StackTagging.cpp) is a machine-level pass that runs after register allocation and replaces stack tagging intrinsics with actual `IRG`/`STG`/`ADDG` instructions.

The pass inserts:

**Prologue** (per-function):
1. One `IRG` instruction to generate a random tag for the frame (excluding tag 0 and optionally adjacent frame tags for use-between-scope detection)
2. For each tagged alloca: compute the stack offset, emit `ADDG` to create a tagged pointer with the appropriate sub-frame tag, emit `STZ2G` or `STZG` to initialize the allocation tag for the memory granules

**Epilogue** (per function exit, including unwind paths):
1. `STG xzr, [sp+offset]` for each granule of each tagged alloca — writes tag 0, making any escaped pointer stale

The pass also handles scope-aware tagging: variables that go out of scope before the function returns receive `STG xzr` at the end of their lexical block, not just at function exit. This detects use-after-scope bugs mid-function.

```asm
; Complete MTE-tagged function (two local arrays, AArch64):
; C source: void foo() { int a[8]; int b[8]; ... }

foo:
  ; Frame: a[8] at [sp+0..31], b[8] at [sp+32..63]
  sub  sp, sp, #64         ; allocate 64 bytes (4 granules)
  irg  x9, sp, xzr         ; random tag for frame, excluding tag 0
  addg x10, x9, #32, #1   ; x10 = &b with tag+1 (different tag per variable)
  
  ; Tag and zero-init 'a' (2 granules at sp+0):
  stz2g x9, [sp]           ; tag[sp..sp+31] = x9.tag, zero content
  
  ; Tag and zero-init 'b' (2 granules at sp+32):
  stz2g x10, [sp, #32]     ; tag[sp+32..sp+63] = x10.tag, zero content
  
  ; x9 is the tagged pointer to 'a', x10 to 'b'
  ; ... function body ...
  
  ; Epilogue: untag (write tag 0 to all granules):
  stg  xzr, [sp]           ; tag[sp..sp+15] = 0
  stg  xzr, [sp, #16]      ; tag[sp+16..sp+31] = 0
  stg  xzr, [sp, #32]      ; tag[sp+32..sp+47] = 0
  stg  xzr, [sp, #48]      ; tag[sp+48..sp+63] = 0
  add  sp, sp, #64
  ret
```

### 111.5.3 AArch64GlobalsTaggingPass

The `AArch64GlobalsTaggingPass` assigns unique tags to global variables. Tagged globals are placed in a special output section (`.data.mte`, `.rodata.mte`), and the linker or dynamic loader assigns a tag per global at load time. The pass emits `@llvm.aarch64.tagp` intrinsics that the assembler/linker resolves.

Each global gets a unique tag derived from its symbol address (using the linker's knowledge of the address layout) or a random value. Global buffer overflows are caught because adjacent globals have different tags.

### 111.5.4 Heap Integration Requirements

`-fsanitize=memtag-heap` does not itself insert allocation instructions; it marks the intent. The actual tagging must be performed by the allocator:

- **Scudo** (Chapter 112): built with `SCUDO_ALLOCATOR_ENABLE_TAGS=1` handles MTE allocation tagging natively
- **Android Bionic**: MTE-enabled by default since Android 13 on MTE-capable hardware
- **Custom allocator**: must call `mmap(PROT_MTE)` for tag-capable pages and use `IRG`/`STG` for each allocation

---

## 111.6 Comparison: ASan vs HWASan vs MTE

### 111.6.1 Feature Table

| Feature | ASan | HWASan | MTE |
|---------|------|--------|-----|
| Shadow memory | 1:8 software | 1:16 software | Hardware (no software shadow) |
| Tag bits | 1 bit (poisoned/OK) | 8 bits | 4 bits |
| Granule size | 8 bytes | 16 bytes | 16 bytes |
| CPU overhead | 2–3× | 10–15% | 2–5% sync / 1–2% async |
| Memory overhead | 2× + red zones | ~15% | ~1% (tag RAM) |
| Heap buffer overflow | Yes | Yes (>0 bytes past boundary) | Yes (>0 bytes past granule) |
| Heap use-after-free | Yes (quarantine) | Yes (tag change on free) | Yes (tag change on free) |
| Stack buffer overflow | Yes (red zones) | Yes | Yes |
| Stack use-after-return | Yes (fake stack) | Yes (tag at exit) | Yes (tag 0 at scope exit) |
| Global buffer overflow | Yes | Yes | Yes |
| False negative rate | ~0% | 1/256 per access | 1/16 per access |
| Requires hardware | No | No (soft mode) | Yes (ARMv8.5-A) |
| Production viable | No | Yes | Yes |
| Sanitizer flag | `-fsanitize=address` | `-fsanitize=hwaddress` | `-fsanitize=memtag` |

### 111.6.2 False Negative Analysis

For a systematic bug (same buffer overflow on every call):
- With HWASan (8-bit tag, 256 possible tags): if the overflow hits the same granule with the same adjacent granule every call, the probability of detection on any single call is 255/256 ≈ 99.6%. Across 1000 calls, the probability that at least one call is detected = 1 - (1/256)^1000 ≈ 100%.
- With MTE (4-bit tag, 16 possible tags): detection probability per call = 15/16 ≈ 93.75%. For a fleet of millions of processes, virtually every systematic bug is detected.

For a non-deterministic bug (intermittent overflow at random times): both HWASan and MTE will eventually detect it in a long-running process. The detection rate is high enough that real exploitable bugs are caught in production.

### 111.6.3 Production Adoption

**Android**: Pixel 8 (Cortex-X3, 2023) is the first Pixel phone with MTE-capable hardware. Android 14 enables MTE in async mode for system services and opted-in apps. Crash reports include the MTE fault type and (if async, approximately) the faulting address.

**Linux kernel**: The kernel itself uses MTE for `KASAN_HW_TAGS` mode (Chapter 113). Kernel MTE is enabled via `CONFIG_KASAN=y CONFIG_KASAN_HW_TAGS=y`.

**ChromeOS**: HWASan is used on AArch64 Chromebooks for ongoing vulnerability discovery in Chrome's renderer process.

---

## 111.7 Stack Use-After-Return Detection

### 111.7.1 The Use-After-Return Problem

Stack use-after-return (UAR) occurs when a pointer to a function's local variable is returned or stored and later dereferenced after the function's stack frame has been freed:

```cpp
int *get_local() {
  int x = 42;
  return &x;        // returning pointer to stack-allocated x
}

void test() {
  int *p = get_local();
  *p = 99;          // use-after-return: get_local's frame is gone
}
```

This is the most common stack memory safety bug in C/C++ (apart from stack overflows). Compilers warn about it, but complex cases involving aliasing and callbacks escape static analysis.

### 111.7.2 ASan Fake Stack

ASan detects UAR via a "fake stack": local variables are allocated from a per-thread fake stack on the heap instead of the real stack frame. When `get_local` returns, the fake stack frame is marked as poisoned (shadow bytes = `0xf5`) but not immediately deallocated. If `*p = 99` is executed, ASan checks the shadow and finds `0xf5`, reporting a use-after-return.

Downsides:
- Each function call with local variables requires a heap allocation + TLS lookup (extra ~30–50ns)
- Fake stack frames are not on the real stack, so stack-walking tools (perf, GDB backtraces) may show incorrect frame depths
- Memory cost: each function call duplicates its locals onto the heap

ASan fake stack must be enabled at runtime: `ASAN_OPTIONS=detect_stack_use_after_return=1`.

### 111.7.3 HWASan Tagged Stack

HWASan detects UAR without a fake stack:

1. At function entry: tag the stack frame with a random tag; return tagged pointers to locals
2. At function exit: write tag 0 to the shadow for all tagged granules (the stack memory itself is not touched)
3. Any escaped pointer to a local has the old frame tag (e.g., `0x5A`) in its top byte
4. After return, the shadow byte for that granule is 0 (untagged)
5. Subsequent `*p = 99`: shadow byte = 0 ≠ pointer tag 0x5A → `__hwasan_tag_mismatch`

**Advantages over fake stack**:
- No per-call heap allocation
- No TLS lookup on each call
- Stack frames remain on the real stack (correct backtraces)
- Always-on: no `detect_stack_use_after_return=1` flag needed

**Detection window**: From function return until the stack frame is reused by another function that overwrites the shadow byte with a new tag. This window is typically many milliseconds to seconds in practice.

### 111.7.4 MTE Tagged Stack UAR

MTE handles UAR identically to HWASan's tagged stack approach but with hardware checking:

```asm
; MTE UAR example — get_local():
sub  sp, sp, #16              ; allocate local x (1 granule)
irg  x9, sp, xzr              ; random tag for frame
addg x0, x9, #0, #0          ; x0 = &x as tagged pointer
stzg x9, [sp]                 ; tag granule at sp with x9.tag, zero content
                              ; x is now live and tagged

; Return tagged pointer (caller can store it):
; ... store/compute x ...
add  sp, sp, #16              ; reset stack (but DON'T clear tag!)
; Wait: we must clear the tag to detect UAR!
stg  xzr, [sp, #-16]         ; clear tag for the granule (write tag 0)
add  sp, sp, #16
ret

; After return, if caller does:
str  w2, [x0]                ; x0 has old tag, shadow has tag 0
                              ; → SIGSEGV/SEGV_MTESERR immediately (sync mode)
                              ; or buffered then SEGV/SEGV_MTEAERR (async)
```

The key: `stg xzr, [sp-offset]` writes tag 0 to the granule's allocation tag **before** `add sp`. After the function returns and the caller dereferences `x0` (with old tag), the allocation tag at that address is 0, not old tag → hardware fault.

---

## 111.8 Practical Deployment Patterns

### 111.8.1 Development Build on AArch64

```bash
# HWASan development build (full error detail, all detection types):
clang -fsanitize=hwaddress \
      -fno-omit-frame-pointer \
      -fsanitize-hwaddress-abi=platform \
      -g -O1 \
      program.c -o program_hwasan -target aarch64-linux-gnu

# Run on QEMU (user-mode, fast):
qemu-aarch64 -cpu max,tagged-addr=on ./program_hwasan

# Run on real AArch64 hardware (Cortex-A78+, Cortex-X1+, Apple M1+):
HWASAN_OPTIONS=allocator_may_return_null=1:detect_stack_use_after_return=1 \
./program_hwasan
```

### 111.8.2 Production Build with MTE Async

```bash
# Production build — minimal overhead, approximate fault location:
clang -fsanitize=memtag \
      -fsanitize-memtag-mode=async \
      -march=armv8.5-a+memtag \
      -O2 -fno-omit-frame-pointer \
      program.c -o program_mte \
      -target aarch64-linux-gnu

# The process must enable MTE at startup (or use a wrapper):
# Either in main() via prctl(), or by linking with a startup library that does this.

# When a bug fires: SIGSEGV with si_code=SEGV_MTEAERR
# The crash handler captures si_addr and si_code for the crash report.
```

### 111.8.3 QEMU MTE Testing

```bash
# QEMU full-system with MTE (QEMU 6.2+):
qemu-system-aarch64 \
  -machine virt \
  -cpu max,mte=on \
  -kernel /path/to/kernel \
  -append "kasan=on"

# QEMU user-mode with MTE:
qemu-aarch64 -cpu max,mte=on,tagged-addr=on ./program_mte
```

### 111.8.4 Checking MTE Hardware at Runtime

```c
#include <sys/auxv.h>
#include <sys/prctl.h>
#include <linux/prctl.h>

bool mte_available(void) {
  unsigned long hwcap2 = getauxval(AT_HWCAP2);
  return (hwcap2 & HWCAP2_MTE) != 0;
}

void enable_mte_async(void) {
  if (!mte_available()) return;
  prctl(PR_SET_TAGGED_ADDR_CTRL,
        PR_TAGGED_ADDR_ENABLE | PR_MTE_TCF_ASYNC |
        ((uint64_t)0xFFFE << PR_MTE_TAG_SHIFT),  // exclude tag 0
        0, 0, 0);
}
```

### 111.8.5 GDB/LLDB Support

GDB 13+ and LLDB 16+ have MTE-aware memory inspection:

```
# LLDB: inspect allocation tags
(lldb) memory tag read 0x0500602000000010 0x0500602000000050
Logical tag: 0x5
Allocation tags:
  0x602000000010: 0x5 (match)
  0x602000000020: 0x5 (match)
  0x602000000030: 0x3 (mismatch - pointer tag 0x5 ≠ alloc tag 0x3)

# LLDB: stop at MTE fault
Process 1234 stopped
* thread #1, stop reason = Exception Type: EXC_BAD_ACCESS (SIGSEGV)
  Exception Codes: SEGV_MTESERR at address=0x0a00602000000038
  The fault address carries logical tag 0xa;
  the allocation tag at that address is 0x3.
```

---

## Chapter 111 Summary

- AArch64 TBI (Top Byte Ignore) makes bits [63:56] of a pointer available for metadata; HWASan uses all 8 bits as a random tag; MTE uses 4 bits [59:56] backed by hardware tag storage.
- HWASan uses a 1:16 software shadow (1 byte per 16-byte granule); the check is 4–5 instructions per memory access; on AArch64, the check sequence is `UBFX`+`LSR`+`LDRB`+`CMP`+`B.NE`; 10–15% CPU overhead.
- `HWAddressSanitizerPass` instruments loads/stores; `__hwasan_malloc` generates a random tag + calls `TagMemory`; `__hwasan_free` changes the tag to a different random value for UAF detection (255/256 detection probability per access).
- MTE provides hardware 4-bit allocation tags per 16-byte granule; `IRG` generates random tags, `STG`/`ST2G`/`STZG` store tags, `LDG` reads tags; synchronous mode faults immediately (`SEGV_MTESERR`), async mode buffers faults (`SEGV_MTEAERR`).
- Linux exposes MTE via `prctl(PR_SET_TAGGED_ADDR_CTRL)` with `PR_MTE_TCF_SYNC` or `PR_MTE_TCF_ASYNC`; `mmap(PROT_MTE)` allocates tag-capable pages; `HWCAP2_MTE` signals hardware support.
- `AArch64StackTaggingPass` lowers stack tagging via `IRG`+`ADDG`+`STZ2G` in the prologue and `STG xzr` in the epilogue; scope-aware untagging enables use-after-scope detection mid-function.
- Stack use-after-return: ASan uses a fake stack (heap allocation per frame, ~30–50ns overhead), HWASan/MTE use shadow/tag clearing at function exit (near-zero overhead, no fake memory copies).
- Production adoption: Android 14 enables MTE async on Pixel 8+ for all system services; HWASan in Chrome's AArch64 renderer; MTE in Linux kernel via `CONFIG_KASAN_HW_TAGS`.


---

@copyright jreuben11
