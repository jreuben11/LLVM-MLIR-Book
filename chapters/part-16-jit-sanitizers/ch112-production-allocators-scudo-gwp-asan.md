# Chapter 112 — Production Allocators: Scudo and GWP-ASan

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

A heap allocator is not just a performance component — it is a security boundary. Standard `glibc malloc` was designed for correctness and performance, not adversarial robustness: its metadata structures are inline with user data, predictable in layout, and trivially corrupted by a one-byte heap overflow. An attacker who corrupts heap metadata can redirect arbitrary writes anywhere in the process, making heap exploitation one of the most reliable attack primitives in modern systems exploitation. Scudo, LLVM's hardened allocator, was engineered to make these attacks infeasible while remaining competitive with `jemalloc` in performance. GWP-ASan complements Scudo by offering probabilistic memory safety checking at a fraction of a percent of overhead — making it viable not just for production monitoring, but for always-on deployment in consumer applications. Together, they represent the state of the art in memory-safe production systems software allocation.

---

## 112.1 Production Allocator Requirements

### 112.1.1 Why Standard Allocators Are Insufficient

`glibc malloc` (ptmalloc2) stores the chunk header immediately before the user allocation:

```
glibc malloc chunk layout:
  [prev_size: 8 bytes] ← used only if previous chunk is free
  [size: 8 bytes]      ← includes P/M/N flag bits in low 3 bits
  USER DATA: N bytes   ← this is what malloc() returns
  [next chunk header starts here]
```

A one-byte overflow from the user data into the next chunk's `prev_size` field, or a multi-byte overflow into its `size` field, enables the **unlink attack**: by setting crafted values in `prev_size` and `size`, an attacker can cause `free()` to write an arbitrary value to an arbitrary address. This class of attack has been exploited for decades; glibc has added mitigations (`P == 0`, double-free detection, etc.) but the fundamental problem — metadata adjacent to user data — remains.

**Predictable heap layout**: ptmalloc2's bin structures (`fastbins`, `smallbins`, `tcache`) have predictable addresses relative to `main_arena`. An allocator with address space randomization per size class defeats layout-dependent exploits.

**No integrity checking**: There is no checksum or canary on chunk headers. Corruption is silently accepted until the next allocator operation on that chunk — which may be much later, making attribution difficult.

**No quarantine**: Freed memory is immediately available for reuse. A UAF write into a freed chunk corrupts a subsequently allocated live object, with no detection.

### 112.1.2 Threat Model for Scudo

Scudo is designed to defeat:
1. **Heap metadata corruption**: chunk header checksum catches header overwrites
2. **Use-after-free exploitation**: quarantine delays recycling; MTE tags change on free
3. **Heap layout prediction**: randomized region bases; random size-class selection for border allocations
4. **Out-of-bounds write exploitation (secondary allocations)**: guard pages before and after large allocations
5. **Double-free**: header state machine catches freed-again chunks
6. **Type confusion**: chunk header stores type tag (future enhancement)

Scudo does NOT prevent buffer overflows — it detects exploitation attempts after corruption has occurred, making them unreliable. For prevention, combine with ASan/HWASan/MTE.

---

## 112.2 Scudo Architecture

### 112.2.1 Two-Level Allocator

Scudo (`compiler-rt/lib/scudo/standalone/`) is a template-configured two-level allocator:

```cpp
// Combined.h: top-level allocator class:
template <typename Config>
class Allocator {
  typename Config::Primary Primary;    // size-class slab allocator
  typename Config::Secondary Secondary; // large allocation via mmap
  
  void *allocate(uptr Size, uptr Alignment, AllocType Type);
  void deallocate(void *Ptr, uptr DeleteSize, AllocType Type);
};
```

**Primary allocator** ([`primary64.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/lib/scudo/standalone/primary64.h)): handles allocations from 8 bytes up to a threshold (typically 64KB). Divided into size classes:

| Size range | Example sizes | Number of classes |
|-----------|--------------|-------------------|
| 8–1024 B | 8, 10, 12, 16, 20, 24, 32, … | ~32 |
| 1024 B–8 KB | 1024, 1280, 1664, 2048, … | ~12 |
| 8 KB–64 KB | 8192, 10240, 16384, 32768, 65536 | ~6 |

Total: 56 size classes for the default Android configuration. Each size class has its own slab region, mapped at a randomized base address, divided into fixed-size chunks.

**Secondary allocator** ([`secondary.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/lib/scudo/standalone/secondary.h)): handles allocations above the primary threshold:
- Uses `mmap(MAP_ANONYMOUS)` for each allocation
- Surrounds the user region with guard pages (`PROT_NONE`)
- Metadata stored in a separate hash table, not adjacent to the allocation

### 112.2.2 Per-CPU and Per-Thread Caches

Scudo uses a two-level cache hierarchy to minimize lock contention:

**Per-CPU cache** ([`local_cache.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/lib/scudo/standalone/local_cache.h)): indexed by `sched_getcpu() % NumShards`. Each CPU shard has a small cache of free chunks per size class. Allocation from the per-CPU cache requires only:
1. Disable interrupts/preemption (or use `cmpxchg` on the count)
2. Pop from the size-class array
3. Re-enable

This is ~15–25 nanoseconds per allocation — comparable to `jemalloc`'s tcache path.

**Per-thread cache**: each thread has a larger cache that feeds the per-CPU cache when the CPU cache empties, and drains from the CPU cache when the thread cache fills. The thread cache size is configurable (default 256KB across all size classes).

**Batch transfer**: when the per-CPU cache for a size class is exhausted, Scudo fetches a batch of `TransferBatchSize` chunks from the primary's slab (via the `TransferBatch` structure). This amortizes slab-level locking across many allocations.

### 112.2.3 Primary Slab Layout

The primary allocator divides each slab region into fixed-size chunks. For size class `C` (chunk size `S`):

```
Primary region (slab) for size class C:
  ┌──────────────────────────────────────────────────────┐
  │ Region base (randomized address)                      │
  ├──────────────────────────────────────────────────────┤
  │ Chunk 0: S bytes  │ Chunk 1: S bytes │ Chunk 2: ...  │
  │ [CompactHeader]   │ [CompactHeader]  │               │
  │ [User data]       │ [User data]      │               │
  ├──────────────────────────────────────────────────────┤
  │ (slab continues for RegionSize / S chunks)           │
  └──────────────────────────────────────────────────────┘
```

Wait — Scudo's chunk headers are NOT inline with user data. Let me correct this:

```
Primary chunk layout:
  ┌──────────────────────────────────────────────────────────┐
  │ CompactHeader (8 bytes): stored at chunk_start - 8       │
  │   ClassId (8 bits) | State (2 bits) | Size (15 bits) |   │
  │   Checksum (16 bits) | Offset (13 bits) | ...            │
  ├──────────────────────────────────────────────────────────┤
  │ USER DATA (S - 8 bytes)                                  │
  │   malloc() returns a pointer to here                     │
  └──────────────────────────────────────────────────────────┘
```

The header is stored **before** the user data pointer (at `ptr - header_size`). While this is adjacent in memory to the user data, the checksum makes corruption detectable:

```cpp
// From chunk.h:
struct PackedHeader {
  u64 Offset            : 16;  // distance from chunk start to user ptr
  u64 ClassId           :  8;  // size class
  u64 State             :  2;  // Allocated/Quarantined/Available
  u64 OriginOrWasteSize : 13;  // allocation origin or wasted bytes
  u64 SizeOrUnusedBytes : 15;  // requested size (not chunk size)
  u64 Checksum          : 16;  // CRC32(header ^ ptr ^ epoch_key)
};
// Total: 70 bits, packed into a u64 + u8

// Checksum computation:
u16 computeChecksum(u64 HeaderVal, uptr Ptr, u32 EpochKey) {
  // CRC32 using hardware instruction if available (AArch64: CRC32CX, x86: _mm_crc32_u64):
  u32 CRC = computeCRC32(EpochKey, Ptr);
  CRC = computeCRC32(CRC, HeaderVal);
  return (u16)(CRC ^ (CRC >> 16));
}
```

`EpochKey` is a random 32-bit value generated at process startup using `getentropy()` or `/dev/urandom`. An attacker must know this key to forge a valid header — defeating generic heap metadata attacks.

---

## 112.3 Scudo Security Features

### 112.3.1 Header Checksum

Every chunk operation (allocation, deallocation, reallocation) begins with:

```cpp
// chunk.h: verify header on free():
PackedHeader Header = loadAtomicHeader(Ptr);
u16 Expected = computeChecksum(Header.clearChecksum(), Ptr, EpochKey);
if (Expected != Header.Checksum) {
  dieWithMessage("corrupted chunk header at %p (expected %x, got %x)",
                 Ptr, Expected, Header.Checksum);
}
```

Detected corruption causes immediate program termination with a diagnostic message, rather than silent exploitation. This catches:
- Buffer overflows that extend past user data into the header
- Use-after-free writes that corrupt freed chunk headers
- Double-free attempts (header.State would be `Available`, not `Allocated`)

### 112.3.2 Random Region Base Addresses

Each size class's slab region is mapped at a random address:

```cpp
// primary64.h:
uptr MapBase = 0;
if (LIKELY(TryMapPackedCounterArrayBatch)) {
  uptr Rand = getRandomU64(&RandState);
  MapBase = (Rand % (MapEnd / RegionSize)) * RegionSize;
}
// MapBase is then used as the base address for this size class's slab
```

This ensures that the offset between an allocation in one size class and an allocation in another size class is unpredictable — thwarting exploits that depend on heap layout predictability.

### 112.3.3 Quarantine

[`quarantine.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/lib/scudo/standalone/quarantine.h) holds freed chunks for a configurable period before recycling:

```
Thread 1 frees chunk P:
  1. Header.State = Quarantined
  2. Add P to per-thread quarantine list
  3. Per-thread quarantine size += chunk_size
  4. If per-thread quarantine size > QuarantinePerThreadSizeKb:
     5. Move oldest chunks to per-cache quarantine
     6. If total quarantine size > QuarantineSizeMb:
        7. Oldest quarantined chunks → size-class free list
           (Header.State = Available)
```

While quarantined, a chunk's shadow/tag (if using ASan/HWASan/MTE) remains set to the "freed" marker. Any access to quarantined memory triggers the shadow check. Even without a sanitizer, the Header.State check on re-use detects if quarantined memory was freed twice.

### 112.3.4 Canaries

With `SCUDO_ENABLE_CANARY=1` (compile-time flag), Scudo adds a random word at the end of each allocation:

```
Layout with canary:
  [Header: 8 bytes] [User data: N bytes] [Canary: 8 bytes] [Alignment padding]
```

On `free()`, Scudo reads the canary and compares it to the stored value (from the header). A buffer overflow that extends past the user data but doesn't reach the next chunk's header is caught by the canary check.

### 112.3.5 Out-of-Line Metadata (Secondary)

Large allocations (secondary allocator) store all metadata in a separate region:

```
Secondary allocation layout:
  [PROT_NONE guard page: 4KB]
  [optional leading padding: for alignment]
  [User data: N bytes] (aligned to end of page for right overflow detection)
  [trailing padding]
  [PROT_NONE guard page: 4KB]

Metadata stored separately in:
  LargeBlock::Header {
    uptr MapBase;      // base of the mmap region
    uptr MapSize;      // total mmap size (includes guard pages)
    uptr BlockBegin;   // address returned to user
    uptr BlockSize;    // size of user-accessible region
    u64  AllocTime;    // timestamp (for age-based UAF detection)
    ...
  };
  // Stored in a separately-mmap'd metadata array, not adjacent to user data
```

A linear buffer overflow from user data into the trailing padding hits the `PROT_NONE` guard page and causes an immediate `SIGSEGV` — even without a sanitizer.

---

## 112.4 Scudo + MTE Integration

### 112.4.1 Tag-Per-Allocation

When Scudo is built with MTE support (automatically when compiled for AArch64 with `mte` CPU feature and kernel support), each allocation and deallocation uses MTE instructions:

```cpp
// combined.h (simplified MTE path):
template <typename Config>
void *Allocator<Config>::allocate(uptr Size, uptr Alignment) {
  // ... size class selection, cache lookup ...
  void *Ptr = getChunkFromCache(ClassId);
  uptr TaggedPtr = reinterpret_cast<uptr>(Ptr);
  
  if (Primary.SupportsMemoryTagging) {
    // IRG: generate random tag, excluding current tag (to detect UAF):
    // In C using ARM ACLE intrinsics:
    u8 OldTag = extractTag(TaggedPtr);
    uptr ExcludeMask = (1UL << OldTag);  // exclude old tag
    TaggedPtr = __arm_mte_create_random_tag(TaggedPtr, ExcludeMask);
    
    // STZ2G: tag memory AND zero fill (atomic tag + zero):
    uptr AlignedSize = RoundUpTo(Size, 16);
    tagAndZeroMemory(reinterpret_cast<void*>(TaggedPtr), AlignedSize);
    // tagAndZeroMemory emits: stz2g / stz2g / ... for each 32 bytes
  }
  
  return reinterpret_cast<void*>(TaggedPtr);
}

void Allocator<Config>::deallocate(void *TaggedPtr) {
  if (Primary.SupportsMemoryTagging) {
    // Retag with a cyclic tag (OldTag + 1 mod 16, never 0):
    u8 OldTag = extractTag(reinterpret_cast<uptr>(TaggedPtr));
    u8 FreeTag = ((OldTag % 15) + 1);  // cycles 1..15, skips 0
    uptr UntaggedPtr = untagPointer(reinterpret_cast<uptr>(TaggedPtr));
    uptr Size = getChunkSize(UntaggedPtr);
    tagMemory(reinterpret_cast<void*>(UntaggedPtr), Size, FreeTag);
    // tagMemory: stg / stg / ... for each 16 bytes
  }
  // ... quarantine / return to free list ...
}
```

The cyclic tag strategy (OldTag + 1) ensures:
- Every reuse of the same memory slot has a different tag
- A use-after-free access via the old pointer (with OldTag) will always fail (FreeTag ≠ OldTag)

### 112.4.2 MTE and Scudo Performance

With MTE async mode and Scudo:
- **Hot path** (per-CPU cache hit): allocate ~20–30ns + `IRG` + `STZ2G` per granule ≈ 30–50ns total
- **Free hot path**: `STG` per granule + quarantine update ≈ 20–40ns
- **Net overhead vs un-tagged Scudo**: ~20% (dominated by tag memory writes)
- **Net overhead vs glibc malloc**: ~30–40% (Scudo's base overhead + tagging)

For workloads where allocation/deallocation is not in the critical path (most server software), this is acceptable in production.

---

## 112.5 Scudo Configuration and Deployment

### 112.5.1 Compile-Time Configuration

Scudo's behavior is configured via template parameters. Different platforms define their own configurations:

```cpp
// Example: Android's Scudo configuration (simplified from bionic):
struct AndroidSizeClassConfig {
  static const uptr NumBits = 7;             // 128 size classes max
  static const uptr MinSizeLog = 4;          // minimum 16 bytes
  static const uptr MidSizeLog = 8;          // inflection at 256 bytes
  static const uptr MaxSizeLog = 17;         // maximum 128KB in primary
  using SizeClassMap = CombinedSizeClassMap<NumBits, MinSizeLog,
                                             MidSizeLog, MaxSizeLog>;
};

struct AndroidAllocatorConfig {
  static const bool MaySupportMemoryTagging = true;  // enable MTE
  static const bool EnableCRC32 = true;              // header checksum
  using Primary = SizeClassAllocator64<AndroidSizeClassConfig>;
  using Secondary = MapAllocator<DefaultMapAllocatorConfig>;
  static const u32 QuarantineSize = 2 << 20;    // 2MB quarantine
  static const u32 ThreadLocalQuarantine = 64 << 10; // 64KB per-thread
};
```

### 112.5.2 Deployment Options

```bash
# Method 1: Compile-time (links Scudo from compiler-rt):
clang -fsanitize=scudo program.c -o program
# Or for a C++ program:
clang++ -fsanitize=scudo program.cpp -o program

# Method 2: LD_PRELOAD at runtime (no recompile needed):
LD_PRELOAD=/usr/lib/llvm-22/lib/clang/22/lib/linux/libclang_rt.scudo-x86_64.so \
    ./existing_program

# Method 3: Android (built-in, no action required on API 30+)
# Scudo is bionic's default malloc since Android 11

# Scudo runtime options:
SCUDO_OPTIONS=\
  quarantine_size_mb=32:\     # total quarantine size
  thread_local_quarantine_size_kb=256:\
  zero_contents=1:\           # zero memory on free
  abort_on_error=1:\          # abort on heap error (for crash reports)
  soft_rss_limit_mb=2048:\    # kill process if RSS exceeds this
  hard_rss_limit_mb=4096:\    # unconditional kill above this
  dealloc_type_mismatch=1:\   # detect delete[] vs delete mismatches
  delete_size_mismatch=1:\    # detect sized-delete with wrong size
  min_free_limit_mb=100       # don't quarantine if free memory < this
./program
```

### 112.5.3 Android Integration Details

Android uses Scudo in all app processes since Android 11. The integration is in:

```
bionic/libc/bionic/scudo_wrapper.cpp  # Android-specific Scudo configuration
bionic/libc/bionic/malloc_common.cpp  # dispatch table
```

Android-specific Scudo features:
- `bionic_scudo_set_heap_tagging_level(tag_level)`: runtime API to enable/disable MTE; allows per-app MTE level based on `android:gwpAsanMode` manifest attribute
- Tombstone integration: crash reports in `/data/tombstones/` include Scudo error details (chunk state, allocation stack trace)
- Per-Zygote-fork randomization: each `fork()` from Zygote re-randomizes Scudo's `EpochKey` and region bases

---

## 112.6 GWP-ASan Architecture

### 112.6.1 Core Concept: Probabilistic Guard Pages

GWP-ASan (Guarded With Probabilistic-ASan, [`compiler-rt/lib/gwp_asan/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/compiler-rt/lib/gwp_asan/)) takes a different approach: instead of checking every allocation, it samples allocations at a low rate and applies full exact guard-page checking to the sample.

```
Normal path (N-1 out of N allocations, default N=5000):
  malloc(size) → regular allocator, no overhead at all

Sampled path (1 out of N allocations):
  malloc(size) → place on a guarded slot

Guarded slot layout (1 user page + 2 guard pages):
  ┌─────────────────┐
  │ PROT_NONE guard │  ← 4KB guard page (catches underflow)
  ├─────────────────┤
  │ ...padding...   │  ← padding to right-align user data
  │ [user data]     │  ← allocation placed at RIGHT edge of page
  ├─────────────────┤
  │ PROT_NONE guard │  ← 4KB guard page (catches overflow)
  └─────────────────┘
```

By placing the allocation at the **right edge** of the usable page, any single byte past the end falls on the guard page and causes immediate `SIGSEGV`. By keeping the guard page after free, any use-after-free access also faults.

### 112.6.2 GWP-ASan Pool Structure

The pool is a static allocation of `MaxSimultaneousAllocations` slots:

```
GWP-ASan pool (128 slots × 3 pages each = ~1.5MB):
  
  Slot[0]:  [guard:4KB][...pad...user_data_N...][guard:4KB]
  Slot[1]:  [guard:4KB][...pad...user_data_M...][guard:4KB]
  ...
  Slot[127]:[guard:4KB][...pad...user_data_P...][guard:4KB]

Alignment: user data is aligned to the right edge of the middle page:
  allocation_end = slot_start + 2*page_size
  allocation_start = allocation_end - RoundUp(size, alignment)
  
Metadata array (128 entries × ~64 bytes):
  Slot[0]: {AllocationAddress, Size, AllocationTrace[30], IsDeallocated,
             DeallocationTrace[30], AllocationThreadID, DeallocationThreadID}
  ...
```

The metadata array is stored separately from the slot pages — it is in regular read-write memory, not in the guarded region.

### 112.6.3 Sampling and Slot Allocation

```cpp
// gwp_asan/guarded_pool_allocator.cpp:
void *GuardedPoolAllocator::allocate(size_t Size, size_t Alignment) {
  // Sampling: decrement counter and check:
  if (UNLIKELY(--ThreadLocals.NextSampleCounter <= 0)) {
    // Resample: new counter = geometric distribution with mean SampleRate
    ThreadLocals.NextSampleCounter = getSampleCount(State.Options.SampleRate);
    
    // Try to get a free slot:
    u32 FreeSlot = reserveSlot();
    if (FreeSlot != kInvalidSlotId) {
      // Compute the guarded allocation address (right-aligned):
      uptr SlotBase = FirstPageAddr + FreeSlot * (2 * PageSize + PageSize);
      uptr UserPage = SlotBase + PageSize;  // middle (user) page
      uptr AlignedEnd = UserPage + PageSize;
      uptr UserStart = AlignedEnd - RoundUpTo(Size, Alignment);
      
      // Record metadata:
      AllocationMetadata *Meta = &Metadata[FreeSlot];
      Meta->Addr = UserStart;
      Meta->Size = Size;
      Meta->IsDeallocated = false;
      Meta->AllocationTrace.capture(Options.Backtrace);
      Meta->AllocationThreadID = getThreadID();
      
      // Make the middle page read-write:
      mprotect((void*)UserPage, PageSize, PROT_READ | PROT_WRITE);
      
      // Zero-initialize (for deterministic behavior):
      memset((void*)UserStart, 0, Size);
      
      return (void*)UserStart;
    }
  }
  
  // Fall through to normal allocator:
  return nullptr;  // caller falls back to real malloc
}
```

### 112.6.4 Fault Handling

When a `SIGSEGV` fires in a GWP-ASan-monitored process:

```cpp
// gwp_asan/guarded_pool_allocator.cpp:
void GuardedPoolAllocator::trapOnAddress(uintptr_t ErrorAddr,
                                          gwp_asan::Error E) {
  // Find which slot this address belongs to:
  size_t Slot = getSlot(ErrorAddr);
  AllocationMetadata *Meta = &Metadata[Slot];
  
  // Determine error type:
  bool IsBuffer = isAdjacentToGuard(ErrorAddr, Slot);
  bool IsUAF = Meta->IsDeallocated;
  
  // Print detailed report:
  if (IsUAF) {
    Printf("ERROR: GWP-ASan: use-after-free on address %p\n", ErrorAddr);
    Printf("  Allocation at:\n");
    printStackTrace(Meta->AllocationTrace, Options.PrintBacktrace);
    Printf("  Deallocation at:\n");
    printStackTrace(Meta->DeallocationTrace, Options.PrintBacktrace);
  } else if (IsBuffer) {
    Printf("ERROR: GWP-ASan: %s on address %p\n",
           isOverflow(ErrorAddr, Slot) ? "buffer-overflow" : "buffer-underflow",
           ErrorAddr);
    Printf("  Allocation of %zu bytes at %p\n", Meta->Size, Meta->Addr);
    Printf("  Allocated at:\n");
    printStackTrace(Meta->AllocationTrace, Options.PrintBacktrace);
  }
}
```

The metadata includes both the allocation and deallocation stack traces, captured at the time of the allocation/free call. This is what makes GWP-ASan reports so actionable: you immediately know both where the memory came from and where it was freed.

---

## 112.7 GWP-ASan Integration

### 112.7.1 Allocator Integration

GWP-ASan wraps an existing allocator. The integration pattern:

```cpp
// Wrapper malloc:
void *malloc(size_t size) {
  // Try guarded allocation first (with low probability):
  void *Guarded = GWPAsan.allocate(size, /*Alignment=*/1);
  if (Guarded) return Guarded;
  
  // Fall through to base allocator:
  return scudo_malloc(size);  // or glibc, jemalloc, etc.
}

void free(void *ptr) {
  if (GWPAsan.pointerIsMine(ptr)) {
    GWPAsan.deallocate(ptr);
    return;
  }
  scudo_free(ptr);  // delegate to base allocator
}

bool pointerIsMine(const void *ptr) {
  // O(1) check: is ptr in the GWP-ASan slot range?
  uptr P = reinterpret_cast<uptr>(ptr);
  return P >= FirstPageAddr && P < LastPageAddr;
}
```

**Scudo + GWP-ASan** (default on Android): Scudo's `Combined.h` includes native GWP-ASan integration. When `GWPAsanOptions.Enabled` is true, Scudo routes 1-in-SampleRate allocations through the guarded pool automatically.

**glibc + GWP-ASan**: glibc 2.32+ integrates GWP-ASan via `malloc_hook` and tunable `glibc.malloc.gwp_asan.enable`:
```bash
GLIBC_TUNABLES=glibc.malloc.gwp_asan.enable=1:glibc.malloc.gwp_asan.sample_freq=5000 \
    ./program
```

### 112.7.2 Android Integration

Android enables GWP-ASan by default for installed apps on Android 11+. Configuration via `AndroidManifest.xml`:

```xml
<application
    android:gwpAsanMode="always">  <!-- always-on, ~1/1000 sample rate -->
    <!-- or "default" (on with default ~1/5000 rate) -->
    <!-- or "never" (disabled) -->
</application>
```

When GWP-ASan detects a bug:
1. `SIGSEGV` fires in the app process
2. The debuggerd crash daemon reads the GWP-ASan fault data via `/proc/PID/maps`
3. The tombstone file in `/data/tombstones/` includes the full GWP-ASan report
4. On Google Play: the Android Vitals dashboard aggregates GWP-ASan crashes
5. Developers see: exact error type, allocation/deallocation stack traces, and the faulting address

### 112.7.3 Chrome Integration

Chrome has used GWP-ASan since Chrome 80. Integration in `base/allocator/`:

```cpp
// Chrome uses custom backtrace functions optimized for its process model:
gwp_asan::options::Options GWPOptions;
GWPOptions.Enabled = true;
GWPOptions.SampleRate = process_type == "renderer" ? 1000 : 5000;
GWPOptions.MaxSimultaneousAllocations = 128;
GWPOptions.Backtrace = base::debug::CollectStackTrace;
GWPOptions.PrintBacktrace = base::debug::FormatStackTrace;

allocator::InitializeGwpAsanWithOptions(GWPOptions);
```

Chrome uses a higher sample rate for renderer processes (which execute untrusted web content) than for browser/GPU processes. GWP-ASan crashes in Chrome go to the Breakpad/Crashpad crash database, where they are triaged with security priority.

### 112.7.4 GWP-ASan Configuration Options

```cpp
// gwp_asan/options.h:
struct Options {
  bool Enabled = false;                    // off by default
  int SampleRate = 5000;                   // 1-in-N sampling
  int MaxSimultaneousAllocations = 128;    // number of guarded slots
  bool InstallSignalHandlers = true;       // register SIGSEGV handler
  bool InstallForkHandlers = true;         // re-initialize on fork()
  BacktraceFunction Backtrace = nullptr;   // stack capture function
  PrintBacktraceFunction PrintBacktrace = nullptr;  // symbolization
  size_t GuardedPagePoolSize = 0;          // override default pool size
};
```

Default memory cost: 128 slots × 3 pages × 4096 bytes = 1,572,864 bytes ≈ 1.5MB. Plus 128 × ~128 bytes metadata = ~16KB. Total: ~1.53MB, independent of allocation rate.

---

## 112.8 Stack Traces in Production

### 112.8.1 Fast Unwind: Frame Pointer Chain

Both Scudo (on error) and GWP-ASan (at every sample allocation and deallocation) need stack traces. The frame pointer chain traversal is fast (~20ns for 30 frames):

```cpp
// sanitizer_common/sanitizer_stacktrace.cpp (simplified):
size_t FastUnwind(uintptr_t *TraceBuffer, size_t MaxDepth,
                  uintptr_t PC, uintptr_t BP) {
  size_t Depth = 0;
  TraceBuffer[Depth++] = PC;
  
  uintptr_t *FP = reinterpret_cast<uintptr_t*>(BP);
  while (Depth < MaxDepth && FP != nullptr) {
    // Frame layout (standard ABI):
    // FP[0] = saved frame pointer (next FP)
    // FP[1] = return address (caller's PC)
    uintptr_t ReturnAddr = FP[1];
    if (ReturnAddr == 0) break;
    
    TraceBuffer[Depth++] = ReturnAddr;
    
    uintptr_t *NextFP = reinterpret_cast<uintptr_t*>(FP[0]);
    // Sanity check: FP must increase (towards higher stack addresses):
    if (NextFP <= FP || !isAligned(NextFP, sizeof(uintptr_t))) break;
    FP = NextFP;
  }
  return Depth;
}
```

Requires: all code in the call chain compiled with `-fno-omit-frame-pointer`. If any frame breaks the chain (e.g., a leaf function without a frame pointer), the trace stops at that frame.

### 112.8.2 DWARF CFI Unwind

For accurate traces through code without frame pointers, `_Unwind_Backtrace` uses DWARF `.eh_frame` data:

```cpp
// sanitizer_common/sanitizer_unwind_posix_libcdep.cpp:
struct UnwindState {
  uintptr_t *Buffer;
  size_t MaxDepth;
  size_t Depth;
};

static _Unwind_Reason_Code unwindCallback(_Unwind_Context *Ctx, void *Arg) {
  auto *State = static_cast<UnwindState*>(Arg);
  if (State->Depth >= State->MaxDepth) return _URC_END_OF_STACK;
  
  uintptr_t PC = _Unwind_GetIP(Ctx);
  State->Buffer[State->Depth++] = PC;
  return _URC_NO_REASON;
}

size_t SlowUnwind(uintptr_t *Buffer, size_t MaxDepth) {
  UnwindState State = {Buffer, MaxDepth, 0};
  _Unwind_Backtrace(unwindCallback, &State);
  return State.Depth;
}
```

DWARF unwind is ~10× slower than frame-pointer unwind (~200ns for 30 frames). GWP-ASan uses fast unwind for the allocation hot path; Scudo's error reporter uses either based on configuration.

### 112.8.3 Compressed Stack Traces

GWP-ASan compresses stack traces to minimize metadata memory:

```cpp
// gwp_asan/stack_trace_compressor.h:
// Compress using LEB128-encoded delta encoding:
// - Compute delta = addr[i] - addr[i-1] (for i > 0), addr[0] for i = 0
// - Encode each delta as a signed LEB128 variable-length integer
// Typical compression: 30 frames × 8 bytes = 240 bytes raw
//                   → 30 frames × 2–4 bytes avg = 60–120 bytes compressed

size_t compress(const uintptr_t *Addrs, size_t NumAddrs,
                uint8_t *Out, size_t OutSizeBytes) {
  uintptr_t Prev = 0;
  uint8_t *P = Out;
  for (size_t i = 0; i < NumAddrs; ++i) {
    intptr_t Delta = (intptr_t)(Addrs[i] - Prev);
    P += encodeSLEB128(Delta, P, Out + OutSizeBytes - P);
    Prev = Addrs[i];
    if (P >= Out + OutSizeBytes) break;
  }
  return P - Out;
}
```

### 112.8.4 Symbolization

Raw return addresses must be converted to human-readable function names:

```bash
# llvm-symbolizer (supports ELF, MachO, DWARF, split DWARF):
llvm-symbolizer --obj=/path/to/binary 0x4011a3 0x4011e2

# Output:
# foo
#   /src/test.c:8:5
# main
#   /src/test.c:14:3

# addr2line (GNU binutils, simpler):
addr2line -e /path/to/binary -f 0x4011a3

# Android ndk-stack (reads tombstone, symbolizes using NDK symbol files):
ndk-stack -sym /path/to/app/obj/local/arm64-v8a -dump tombstone.txt
```

For GWP-ASan reports in production, symbolization happens offline using the binary stored in the crash reporting infrastructure (e.g., Breakpad/Crashpad symbol files, Android symbol zips).

---

## 112.9 Comparing Scudo, GWP-ASan, and ASan

### 112.9.1 Full Comparison Table

| Property | glibc malloc | Scudo | GWP-ASan | ASan |
|----------|-------------|-------|----------|------|
| Security hardening | None | Strong (CRC, randomization, guard) | None (allocator policy) | N/A |
| Buffer overflow detection | No | Partial (canary, guard pages for large) | Yes (sampled, exact) | Yes (all) |
| Use-after-free detection | No | Partial (quarantine) | Yes (sampled, exact) | Yes (all) |
| Double-free detection | Partial (ptmalloc mitigations) | Yes (header state check) | No | Yes |
| Memory leak detection | No | No | No | Yes (via LSan) |
| CPU overhead | Baseline | 5–10% | ~0.02% | 2–3× |
| Memory overhead | Baseline | 10–20% | ~1.5MB fixed | 2× |
| Detection rate per bug per process | 0% | variable | 0.02% | ~100% |
| Production safe | Yes | Yes | Yes | No |
| Requires recompile | No | No (LD_PRELOAD) | No (LD_PRELOAD) | Yes |
| MTE integration | No | Yes | No | No |
| Platform | All | All | All | All |

### 112.9.2 Deployment Architecture

A production service with full defense-in-depth:

```
Development and CI:
┌─────────────────────────────────────────────────┐
│  ASan + UBSan (-fsanitize=address,undefined)    │
│  → All heap/stack/global bugs in test paths     │
│  → All UB in test paths                         │
│  TSan (separate build)                          │
│  → All data races in test paths                 │
└─────────────────────────────────────────────────┘

Production (x86_64 or AArch64 without MTE):
┌─────────────────────────────────────────────────┐
│  Scudo allocator (-fsanitize=scudo or LD_PRELOAD)│
│  → Defeats metadata exploitation attempts       │
│  → Quarantine-based UAF hardening               │
│  + GWP-ASan (1/5000 sampling)                   │
│  → Catches missed heap bugs in production       │
│  → Near-zero overhead (~0.02%)                  │
└─────────────────────────────────────────────────┘

Production (AArch64 with MTE hardware, ARMv8.5-A+):
┌─────────────────────────────────────────────────┐
│  Scudo + MTE async (-fsanitize=memtag)          │
│  → Every heap allocation tagged with random tag  │
│  → Heap overflow/UAF detected at ~1-2% overhead │
│  + AArch64 stack tagging (memtag-stack)         │
│  → Stack overflow/UAR detected at ~1% overhead  │
│  GWP-ASan still active for coverage monitoring  │
└─────────────────────────────────────────────────┘
```

---

## Chapter 112 Summary

- Standard allocators (glibc ptmalloc2) store metadata adjacent to user data with no integrity protection, enabling heap metadata corruption exploits; Scudo addresses this via CRC32-protected headers, randomized region bases, out-of-line large-allocation metadata, and quarantine.
- Scudo uses a two-level architecture: primary (per-size-class slabs with per-CPU and per-thread caches at 15–30ns hot-path latency) and secondary (mmap with guard pages for large allocations).
- Chunk header checksum `CRC32(header ^ ptr ^ epoch_key)` detects corruption on every allocation/deallocation operation; `epoch_key` is randomized at process startup, preventing header forgery.
- Scudo integrates with MTE by calling `IRG`/`STZ2G` to assign random tags at allocation and cycling tags at deallocation; tag mismatch faults are decoded using chunk metadata for accurate error reports.
- GWP-ASan samples 1-in-N (default 5000) allocations, places each on a guarded slot (user data right-aligned in a page flanked by `PROT_NONE` guard pages), and provides exact buffer-overflow and UAF detection at ~0.02% CPU overhead and ~1.5MB fixed memory cost.
- GWP-ASan captures allocation and deallocation stack traces using fast frame-pointer-chain unwind (~20ns) at sample time; compressed via delta LEB128 encoding; fault handler includes both traces for actionable bug reports.
- Integration targets: Scudo is Android's default allocator since Android 11; GWP-ASan is default in Android apps, Chrome, ChromeOS; production crash reports include exact error type and dual stack traces.
- Stack trace symbolization uses `llvm-symbolizer` offline; DWARF CFI slow unwind (~200ns) provides accurate traces through non-frame-pointer code; compressed traces in GWP-ASan metadata reduce per-slot cost from 240 to 60–120 bytes.
- Production defense-in-depth: ASan+UBSan in CI, Scudo+GWP-ASan in production x86_64, Scudo+MTE async for AArch64 hardware — together providing comprehensive coverage at near-zero production overhead.
