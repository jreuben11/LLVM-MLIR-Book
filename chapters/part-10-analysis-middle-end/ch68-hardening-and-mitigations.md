# Chapter 68 — Hardening and Mitigations

*Part X — Analysis and the Middle-End*

Security hardening transforms program code and data layout to raise the cost of exploitation. LLVM provides a comprehensive suite of mitigations: control-flow integrity (CFI), safe and shadow call stacks, speculative load hardening, stack clash protection, stack protectors, pointer authentication (PAuth/BTI), Intel CET/IBT, and FORTIFY_SOURCE-style bounds checking. This chapter covers each mitigation — what attack it defends against, how it is implemented in LLVM, and its performance cost.

## Table of Contents

- [68.1 Control-Flow Integrity (CFI)](#681-control-flow-integrity-cfi)
  - [68.1.1 The Attack: ROP/JOP](#6811-the-attack-ropjop)
  - [68.1.2 Clang CFI Variants](#6812-clang-cfi-variants)
  - [68.1.3 Implementation: Type Test and Type Checked Load](#6813-implementation-type-test-and-type-checked-load)
  - [68.1.4 KCFI (Kernel CFI)](#6814-kcfi-kernel-cfi)
- [68.2 SafeStack](#682-safestack)
- [68.3 ShadowCallStack](#683-shadowcallstack)
- [68.4 Speculative Load Hardening (SLH)](#684-speculative-load-hardening-slh)
- [68.5 Stack Protector (Canary)](#685-stack-protector-canary)
- [68.6 Stack Clash Protection](#686-stack-clash-protection)
- [68.7 PAuth and BTI (AArch64)](#687-pauth-and-bti-aarch64)
- [68.8 Intel CET/IBT](#688-intel-cetibt)
- [68.9 FORTIFY_SOURCE](#689-fortifysource)
- [68.10 Hot/Cold Splitting](#6810-hotcold-splitting)
- [Chapter Summary](#chapter-summary)

---

## 68.1 Control-Flow Integrity (CFI)

### 68.1.1 The Attack: ROP/JOP

Return-oriented programming (ROP) and jump-oriented programming (JOP) attacks chain together existing code snippets ("gadgets") by manipulating control flow. They exploit the gap between the program's intended control-flow graph and what the hardware actually allows.

CFI restricts indirect calls and indirect jumps so they can only target valid destinations defined by the program's call graph.

### 68.1.2 Clang CFI Variants

Clang implements multiple CFI schemes, activated with `-fsanitize=cfi-*` flags:

| Scheme | Flag | What it checks |
|--------|------|---------------|
| VTable pointer CFI | `-fsanitize=cfi-vcall` | Virtual calls land on valid vtable entries |
| Non-virtual call CFI | `-fsanitize=cfi-nvcall` | Non-virtual C++ member calls have the right type |
| Function pointer CFI | `-fsanitize=cfi-icall` | Indirect function calls match the declared type |
| Derived-class cast CFI | `-fsanitize=cfi-derived-cast` | `static_cast` to derived type is valid |
| Unrelated-cast CFI | `-fsanitize=cfi-unrelated-cast` | Casts between unrelated classes are absent |

All schemes require `-flto` (LTO) to compute the global type hierarchy at link time.

### 68.1.3 Implementation: Type Test and Type Checked Load

LLVM implements CFI via two intrinsics:

- `@llvm.type.test(ptr %ptr, metadata !"type_id")`: tests whether `%ptr` points into a valid object of the given type. Returns `i1`.
- `@llvm.type.checked.load(ptr %ptr, i32 %offset, metadata !"type_id")`: loads a function pointer from a vtable and simultaneously tests its type.

```llvm
; CFI-protected virtual call
%vtable = load ptr, ptr %obj    ; load vtable pointer from object
%check = call i1 @llvm.type.test(ptr %vtable, metadata !"class_Foo_vtable_type")
call void @llvm.assume(i1 %check)   ; abort if check fails
%fn_ptr = load ptr, ptr %vtable     ; load virtual function
call void %fn_ptr(ptr %obj, ...)    ; call
```

The `LowerTypeTestsPass` (in `llvm/Transforms/IPO/LowerTypeTests.h`) handles the link-time phase: it groups all objects with the same type into contiguous memory regions and replaces `@llvm.type.test` with a range check:

```llvm
; After LowerTypeTests:
%vtable_offset = sub i64 %vtable, %__start_vtable_section
%in_range = icmp ult i64 %vtable_offset, %vtable_region_size
```

This replaces the abstract type test with a concrete range check — valid vtables are guaranteed to be in the region, invalid ones are not.

### 68.1.4 KCFI (Kernel CFI)

`KCFIPass` (in `llvm/Transforms/Instrumentation/KCFI.h`) is a CFI variant for the Linux kernel. It stores a hash of the function's type in the 4 bytes preceding the function entry and checks that hash before indirect calls:

```llvm
; Kernel CFI check:
%type_hash_addr = sub ptr %fn_ptr, 4
%stored_hash = load i32, ptr %type_hash_addr
%expected_hash = <compile-time hash of target type>
%match = icmp eq i32 %stored_hash, %expected_hash
br i1 %match, %call, %trap
```

KCFI requires no LTO (it works per-module) but is less precise than Clang's LTO-based CFI.

## 68.2 SafeStack

**SafeStack** separates the stack into two regions:
- **Safe stack**: read-only by attackers; contains return addresses, stack pointers, and variables whose address is not taken.
- **Unsafe stack**: contains variables whose address is taken (and could be overwritten by a buffer overflow).

Implementation (`llvm/CodeGen/SafeStack.h`): the `SafeStackPass` analyzes which allocas are "safe" (their address is never passed to untrusted code) and which are "unsafe". Safe allocas remain on the hardware stack; unsafe allocas are moved to the unsafe stack (a separate allocation in thread-local storage).

```bash
clang -fsanitize=safe-stack program.cpp
```

The unsafe stack pointer is stored in a TLS variable (`__safestack_unsafe_stack_ptr`). On context switch, both the safe and unsafe stacks are preserved.

## 68.3 ShadowCallStack

ShadowCallStack protects return addresses from stack buffer overflows by saving each return address to a separate "shadow stack" that the attacker cannot reach:

```asm
; Function prologue (ShadowCallStack):
ldr  x16, [sp, #return_addr_slot]   ; load return addr from caller stack
str  x16, [x18, #shadow_stack_ptr]  ; save to shadow stack (x18)
add  x18, x18, #8                    ; advance shadow stack pointer

; Epilogue:
sub  x18, x18, #8
ldr  x16, [x18]                      ; load from shadow stack
ldr  x17, [sp, #return_addr_slot]   ; load from (potentially tampered) stack
cmp  x16, x17                        ; compare
b.ne __shadowcallstack_abort         ; abort if tampered
```

On AArch64, register `x18` is the shadow stack pointer (reserved for this purpose). On x86-64 with Intel CET, hardware shadow stacks (SHSTK) provide the same protection in hardware.

```bash
clang -fsanitize=shadow-call-stack program.cpp  # AArch64 only
```

## 68.4 Speculative Load Hardening (SLH)

**Spectre variant 1** exploits the CPU's speculative execution: after a mispredicted branch, speculative loads can access memory that the program would not access on the true path, leaking information via the cache.

Speculative Load Hardening (SLH) mitigates Spectre v1 by masking loaded values after conditional branches to prevent speculative loads from producing useful data to an attacker:

```llvm
; Before SLH:
br i1 %bounds_check, %safe, %oob
safe:
  %secret = load i64, ptr %arr_ptr   ; could speculatively leak if %bounds_check is mispredicted

; After SLH:
; Compute a "safe" mask: 0 if branch was mispredicted speculatively, -1 if correct
%mask = select i1 %bounds_check, i64 -1, i64 0   ; in reality, uses conditional moves
%masked_ptr = and ptr %arr_ptr, %mask             ; zero the pointer on mispredicted path
%secret = load i64, ptr %masked_ptr               ; load produces 0 if speculative
```

SLH is expensive (it inserts a mask computation after every conditional branch feeding a load) and is typically applied only to security-critical code:

```bash
clang -mspeculative-load-hardening program.cpp
```

## 68.5 Stack Protector (Canary)

The classic stack protector (GCC's `-fstack-protector`) places a random "canary" value on the stack between local variables and the return address. On function return, the canary is verified:

```llvm
; Function prologue:
%canary = call i64 @__stack_chk_guard_addr()  ; load canary from TLS
; ... function body ...
; Epilogue:
%canary_check = call i64 @__stack_chk_guard_addr()
%ok = icmp eq i64 %canary, %canary_check
br i1 %ok, %return, %fail
fail:
  call void @__stack_chk_fail()
  unreachable
```

Clang offers three levels:
- `-fstack-protector`: only protect functions with `char[]` locals or `alloca`.
- `-fstack-protector-strong`: also protect functions with any local whose address is taken.
- `-fstack-protector-all`: protect every function.

The LLVM IR for stack protectors is generated by `StackProtector.cpp` (in `llvm/CodeGen/StackProtector.h`) and lowered to the platform's `__stack_chk_guard` scheme.

## 68.6 Stack Clash Protection

A **stack clash** attack allocates a large stack frame that jumps over the guard page, landing in the heap or another memory region. Stack clash protection inserts explicit page touches ("probes") when allocating large stack frames:

```llvm
; Large alloca: alloca 64 KB of stack space
; With stack clash protection, emit probes every page (4 KB):
sub %rsp, 4096;  mov (%rsp), %r11  ; touch page 1
sub %rsp, 4096;  mov (%rsp), %r11  ; touch page 2
...
sub %rsp, 4096;  mov (%rsp), %r11  ; touch page 16
```

This ensures the OS's guard page is triggered on any sequential stack growth, preventing jumps over it.

```bash
clang -fstack-clash-protection program.cpp   # Linux x86-64, AArch64
```

## 68.7 PAuth and BTI (AArch64)

**Pointer Authentication (PAuth)** and **Branch Target Identification (BTI)** are AArch64 hardware security features:

- **PAuth**: adds a cryptographic signature to pointers stored in memory. The signature uses a key in the CPU and the pointer's memory address as context. Forging or modifying a signed pointer causes a fault on use.
- **BTI**: marks valid indirect branch targets with a `bti` instruction. The CPU traps if an indirect jump lands anywhere other than a `bti` marker.

```bash
clang -mbranch-protection=bti+pac-ret program.cpp  # Enable both
```

In IR, PAuth generates:
```llvm
; Sign the return address before storing
call void @llvm.aarch64.hint(i32 25)  ; PACIA1716 variant
```

The LLVM AArch64 backend emits the appropriate `pacia`/`pacib`/`autia`/`autib` instructions and `bti c`/`bti j` landing pad markers.

## 68.8 Intel CET/IBT

Intel **Control-flow Enforcement Technology** (CET) provides hardware enforcement of shadow stacks (SHSTK) and indirect branch tracking (IBT):

- **SHSTK**: hardware maintains a separate stack for return addresses. Any `ret` that doesn't match the SHSTK triggers a fault.
- **IBT**: all valid indirect branch targets must begin with `endbr32`/`endbr64`. Indirect jumps that land elsewhere trigger a fault.

```bash
clang -fcf-protection=full program.cpp  # Enable CET SHSTK + IBT
```

The LLVM x86 backend inserts `endbr64` at each function entry and indirect call target when CET is enabled.

## 68.9 FORTIFY_SOURCE

**FORTIFY_SOURCE** is a glibc-level mitigation that replaces unsafe C standard library calls with bounds-checked variants:

```c
// With FORTIFY_SOURCE=2, __builtin___strcpy_chk replaces strcpy:
strcpy(dest, src);
// Becomes: __strcpy_chk(dest, src, __builtin_object_size(dest, 0));
// If object_size is known and src > dest_size, abort at runtime.
```

Clang implements `__builtin_object_size` using `ObjectSizeOffsetVisitor`, which statically computes the allocation size for allocas, globals, and malloc calls. LLVM's `BoundsCheckingPass` (in `llvm/Transforms/Instrumentation/BoundsChecking.h`) adds runtime bounds checks for array accesses.

## 68.10 Hot/Cold Splitting

`HotColdSplittingPass` moves cold basic blocks (those with profile-derived frequency below a threshold) into a separate "cold" function, improving icache performance for the hot code path:

```
hot_function:
  ... hot code ...
  call cold_function.cold (outlined cold path)
  ... more hot code ...

cold_function.cold:   // outlined in a separate section/function
  ... error handling, logging, rare paths ...
  ret
```

The cold section is placed in a different ELF section (`.text.cold`) that the linker can lay out away from the hot section, reducing icache pressure on the hot path.

```bash
opt -passes='hotcoldsplit' -S input.ll
clang -O2 -fprofile-use=profile.profdata program.cpp  # enables splitting with PGO
```

## 68.11 -Wunsafe-buffer-usage and Safe Buffers

Raw pointer arithmetic is the dominant source of memory safety vulnerabilities in C++ — buffer overruns, out-of-bounds reads, and stale pointer accesses all stem from unconstrained `ptr + i` or `ptr[i]` idioms. Clang's `-Wunsafe-buffer-usage` diagnostic (introduced in Clang 16, hardened in Clang 22) flags these patterns at the source level and nudges developers toward span-based rewrites.

### The Diagnostic

`-Wunsafe-buffer-usage` warns on:
- Raw pointer arithmetic: `ptr + n`, `ptr - n`, `ptr++`, `ptr--`.
- Array subscript on raw pointers: `ptr[i]`.
- Passing a raw pointer to a function that expects `unsafe_buffer_usage` patterns.
- Returning a raw pointer from a function when a `span<T>` is appropriate.

```cpp
// Warning: pointer arithmetic on raw pointer
void process(int *data, int n) {
  for (int i = 0; i < n; i++)
    data[i] = 0;   // warning: unsafe buffer access via 'data'
}

// Suggested rewrite using span:
void process(std::span<int> data) {
  for (int &x : data)
    x = 0;   // no warning — span carries its own bounds
}
```

### Fix-It Hints

Clang provides fix-it suggestions when the unsafe usage can be mechanically rewritten:

```
warning: unsafe buffer access [-Wunsafe-buffer-usage]
  data[i] = 0;
  ^~~~~~
note: rewrite as 'std::span<int>' to use bounds-safe access
```

The fix-it infrastructure understands common patterns: a function parameter `T *p, size_t n` is a candidate for replacement with `std::span<T>`, and all uses of `p[i]` within the function become `p[i]` on the span (where bounds checking is enforced by the `span::operator[]` debug mode).

### Hardened Builds

`-Wunsafe-buffer-usage` is used in:
- **Hardened libc++**: the libc++ headers emit `[[clang::unsafe_buffer_usage]]` on unsafe overloads to trigger the diagnostic when callers use them.
- **Chrome**: Chrome's codebase uses `-Wunsafe-buffer-usage` as part of its memory safety effort, tracking reduction of raw pointer arithmetic over time.
- **Android**: the Android build system enables the warning in new C++ code as part of its security hardening posture.

### Safe Buffers Inter-Procedural Analysis

A naive implementation of `-Wunsafe-buffer-usage` produces false positives: a function that receives a raw pointer but never does arithmetic on it (just passes it along) would be flagged. Clang's **Safe Buffers analysis** is an inter-procedural check that traces `span<T>` and `array_ref<T>` propagation across function boundaries to reduce false positives. If a pointer flows from a span source through multiple functions without escaping into unsafe arithmetic, those intermediate functions are not flagged.

### -fbounds-safety (Apple Extension)

Apple developed `-fbounds-safety` (Clang 18+) as a complementary, attribute-based bounds safety system for C code — particularly the Darwin kernel and system libraries. It uses `__counted_by`, `__sized_by`, and `__ended_by` attributes on pointer parameters to attach static or dynamic size information:

```c
void fill(int *__counted_by(n) buf, size_t n);
```

At call sites, the compiler verifies that the passed array has at least `n` elements. This is distinct from `-Wunsafe-buffer-usage` (which is a warning-only diagnostic) — `-fbounds-safety` adds runtime checks and rejects programs that cannot be statically verified.

### Opt-Out for Intentional Unsafe Code

When a function intentionally performs raw pointer arithmetic (e.g., a custom allocator's `free()`), it can be suppressed with:

```cpp
[[clang::unsafe_buffer_usage]]
void dangerous_realloc(char *p, size_t old_size, size_t new_size) {
  // intentional pointer arithmetic — suppressed
  memcpy(p + old_size, new_alloc, new_size - old_size);
}
```

The attribute also applies to entire headers (`#pragma clang unsafe_buffer_usage begin/end`) to suppress warnings in third-party code that cannot be rewritten.

---

## 68.12 counted_by Attribute for Flexible Array Members

C's flexible array member (FAM) idiom — a struct with a trailing zero-length array — is ubiquitous in the Linux kernel, network protocols, and embedded firmware. The problem is that `__builtin_dynamic_object_size()`, which underpins `FORTIFY_SOURCE` (§68.9) and `BoundsCheckingPass`, cannot determine the size of a FAM without additional annotation: `sizeof(struct foo)` gives the base size, not the allocated size including the array.

```c
struct packet {
  uint32_t len;
  uint8_t  data[];  // FAM — actual size unknown without annotation
};
```

Without size information, `__builtin_dynamic_object_size(p->data, 1)` returns `-1ULL` (unknown), and `FORTIFY_SOURCE` cannot check `memcpy` into `p->data`.

### __attribute__((counted_by(field)))

Clang 18 introduced `__attribute__((counted_by(count_field)))` to annotate a FAM with the name of the sibling field holding its element count:

```c
struct packet {
  uint32_t len;
  uint8_t  data[] __attribute__((counted_by(len)));
};
```

This tells the compiler that `p->data[i]` is valid for `i < p->len`. With this annotation:

```c
struct packet *p = malloc(sizeof(*p) + n * sizeof(p->data[0]));
p->len = n;

// Before counted_by: __builtin_dynamic_object_size(p->data, 1) == -1ULL
// After  counted_by: __builtin_dynamic_object_size(p->data, 1) == n * sizeof(uint8_t)
```

### Effect on FORTIFY_SOURCE and BoundsCheckingPass

With `counted_by`, `__builtin_dynamic_object_size()` returns the correct runtime size for FAM fields. This enables:

- **FORTIFY_SOURCE:** `__memcpy_chk(p->data, src, len, __builtin_dynamic_object_size(p->data, 0))` now correctly aborts if `len > p->len`.
- **BoundsCheckingPass (`-fsanitize=bounds`):** generates a precise runtime check `i < p->len` before each `p->data[i]` access.
- **UBSan array-bounds:** reports an accurate error for out-of-bounds FAM accesses.

### Interaction with -fsanitize=array-bounds

```bash
clang -fsanitize=array-bounds -O1 program.c
```

Without `counted_by`, the sanitizer cannot check FAM accesses — it lacks the size. With `counted_by`, the sanitizer inserts:

```llvm
; Before FAM access p->data[i]:
%len = load i32, ptr %p_len_field
%ok = icmp ult i32 %i, %len
br i1 %ok, %continue, %trap
```

This provides the same precision as a fixed-size array bounds check.

### Kernel Adoption

The Linux kernel adopted `counted_by` for many struct definitions in 2024, working through the `__counted_by()` macro (a wrapper for `__attribute__((counted_by(...)))`) as part of the broader kernel effort to adopt Clang hardening features. Hundreds of kernel structures — network packet headers, filesystem inodes with variable-length names, SCSI command blocks — gained `counted_by` annotations.

### Clang Implementation

The semantic analysis for `counted_by` is in `clang/lib/Sema/SemaDeclAttr.cpp`. The attribute is stored in the `FieldDecl`'s attribute list and accessed during code generation. `CGExprScalar.cpp` and `CGBuiltin.cpp` query the annotation when generating `__builtin_dynamic_object_size()` calls.

### Example: DOAS Before and After

```c
#include <stddef.h>
#include <string.h>

struct flex_buf {
  size_t count;
  char   data[] __attribute__((counted_by(count)));
};

void fill(struct flex_buf *fb, const char *src, size_t n) {
  // Without counted_by: object_size = -1ULL → no FORTIFY check
  // With    counted_by: object_size = fb->count → FORTIFY checks n <= fb->count
  memcpy(fb->data, src, n);
}
```

Compiling with `-D_FORTIFY_SOURCE=2 -O2`:

```bash
# Without counted_by annotation:
clang -D_FORTIFY_SOURCE=2 -O2 -c flex_buf_unsafe.c
# __memcpy_chk called with size -1ULL — FORTIFY disabled for this call

# With counted_by annotation:
clang -D_FORTIFY_SOURCE=2 -O2 -c flex_buf_safe.c
# __memcpy_chk called with fb->count — FORTIFY enforces the bound
```

---

## 68.13 Pointer Field Protection (llvm.protected.field.ptr)

Use-after-free (UAF) vulnerabilities account for a large fraction of modern memory safety exploits. Unlike buffer overflows, UAF is harder to detect statically: the freed memory may be reallocated and reused before the dangling pointer is dereferenced, making it invisible to simple lifetime analysis. A frequent exploitation pattern is overwriting a freed object's pointer fields with attacker-controlled data, then triggering a load through the now-dangling pointer.

Per-field pointer integrity (introduced in LLVM 22) protects specific pointer fields within structs against this pattern using a tag-verification mechanism anchored to the `llvm.protected.field.ptr` intrinsic.

### The Mechanism

The protection works at two points:

1. **On store:** When a pointer value is written into a protected field, the compiler inserts a tag-signing operation that encodes metadata (a field-specific tag, optionally the object's address) into unused pointer bits.
2. **On load:** Before the protected pointer is used (dereferenced or passed to a function), the compiler verifies the tag. A mismatch aborts (or signals a sanitizer handler).

```llvm
; Annotated LLVM IR: store into a protected pointer field
%tagged = call ptr @llvm.protected.field.ptr.sign(
    ptr %raw_ptr,        ; the pointer value to store
    ptr %field_addr,     ; address of the field (for address-based tags)
    i64 42)              ; field-specific tag discriminant

store ptr %tagged, ptr %field_addr    ; store tagged pointer

; Load from protected field and verify:
%stored_tagged = load ptr, ptr %field_addr
%verified = call ptr @llvm.protected.field.ptr.auth(
    ptr %stored_tagged,
    ptr %field_addr,
    i64 42)
; %verified is the original pointer if tag matches; otherwise, trap
call void %verified(...)
```

### Relationship to CFI and HWASan

| Mechanism | Threat model | Granularity | Hardware required |
|-----------|-------------|-------------|-------------------|
| CFI (§68.1) | Control-flow hijack via indirect calls | Function pointer type | None (software range check) |
| llvm.protected.field.ptr | UAF via dangling struct field pointer | Per struct-field | None (software) / PAC (AArch64) |
| HWASan MTE | General heap UAF | Per allocation | ARMv8.5 MTE |

`llvm.protected.field.ptr` complements CFI: CFI protects function pointers at the call site; field pointer protection protects data pointers (including member function pointers stored in struct fields) against corruption after free.

On AArch64 hardware that supports **Pointer Authentication (PAC)** (§68.7), the `sign`/`auth` operations lower to `pacia`/`autia` instructions — essentially free in terms of latency on capable microarchitectures. On platforms without PAC, a software fallback uses a canary stored adjacent to the field.

On AArch64 hardware with **MTE (Memory Tagging Extension)**, `llvm.protected.field.ptr` can also integrate with HWASan's tag-based tracking: the freed object's MTE tag is reset, so any load through the dangling pointer triggers a tag mismatch before the `autia` verification is even needed, giving two independent layers of protection.

### Deactivation Symbols

For performance-critical code paths where field protection overhead is unacceptable, LLVM provides a **link-time deactivation mechanism**: a sentinel symbol `__llvm_pfield_unprotected_<mangled_field_name>` can be defined in the binary. If the linker sees this symbol defined, the protection for the corresponding field is elided at link time (the `sign`/`auth` intrinsics lower to no-ops). This allows selective opt-out without recompiling the source.

```bash
# Compile with field protection enabled:
clang -fprotect-parens -O2 program.cpp -o program_protected

# Selectively disable for a specific field at link time:
# (define the deactivation symbol in an assembly stub)
echo ".global __llvm_pfield_unprotected_Foo_ptr_field" > disable.s
clang program.o disable.o -o program_selective
```

### Clang Interface

Field protection is enabled via `-fprotect-parens` (protecting all pointer fields that escape to potential UAF — conservative over-approximation) or via field-level annotations:

```cpp
struct Node {
  int data;
  Node *next [[clang::field_ptr_protected]];  // protect only 'next'
};
```

The attribute `[[clang::field_ptr_protected]]` instructs the Clang front end to emit `llvm.protected.field.ptr.sign` on stores and `llvm.protected.field.ptr.auth` on loads for that specific field.

### Example: IR With and Without Protection

```llvm
; Without protection (vulnerable):
define void @use_node(ptr %node) {
  %next_ptr = load ptr, ptr %node   ; could be dangling
  call void @process(ptr %next_ptr)
  ret void
}

; With llvm.protected.field.ptr:
define void @use_node_protected(ptr %node) {
  %raw = load ptr, ptr %node
  ; Verify tag before use:
  %verified = call ptr @llvm.protected.field.ptr.auth(
      ptr %raw,
      ptr %node,    ; address of the 'next' field within the struct
      i64 0x4E4F4445)  ; discriminant = hash of "Node::next"
  call void @process(ptr %verified)
  ret void
}
```

If `%node` has been freed and the `next` field overwritten by an attacker, the tag mismatch in `auth` traps before `@process` is invoked, preventing the UAF exploit.

---

## Chapter Summary

- **CFI** uses `@llvm.type.test` (at IR level) and range checks after `LowerTypeTestsPass` to restrict indirect calls to valid targets; requires LTO; KCFI is a per-module variant for the Linux kernel.
- **SafeStack** splits the stack into safe (return-address-bearing) and unsafe (address-taken variables) halves, preventing buffer overflows from reaching return addresses.
- **ShadowCallStack** saves return addresses to a TLS-protected shadow stack and verifies them on return; hardware shadow stacks (Intel CET SHSTK) provide the same in hardware.
- **Speculative Load Hardening** masks loaded values with a speculation-safe condition, preventing Spectre v1 from leaking data through speculative loads.
- **Stack protector** (`-fstack-protector`) inserts a canary between locals and the return address; `-fstack-protector-all` protects every function.
- **Stack clash protection** inserts stack probes every page in large stack allocations, preventing attacks that jump over the OS guard page.
- **PAuth/BTI** (AArch64) and **CET/IBT** (x86) provide hardware-enforced pointer signing and branch target restrictions.
- **FORTIFY_SOURCE** uses `__builtin_object_size` to replace unsafe stdlib calls with bounds-checked variants; `BoundsCheckingPass` adds runtime array bounds checks.
- **Hot/cold splitting** moves cold basic blocks into a separate outlined function/section, improving icache utilization for the hot path.
- **-Wunsafe-buffer-usage** warns on raw pointer arithmetic and array subscript through raw pointers; fix-it hints suggest `std::span<T>` rewrites; Safe Buffers inter-procedural analysis reduces false positives; `[[clang::unsafe_buffer_usage]]` suppresses warnings for intentional unsafe code.
- **counted_by** (`__attribute__((counted_by(field)))`) annotates flexible array members with their element count field, enabling `__builtin_dynamic_object_size()` to return a precise runtime size; this activates `FORTIFY_SOURCE` checks and `-fsanitize=array-bounds` verification for FAM accesses; adopted by the Linux kernel in 2024.
- **llvm.protected.field.ptr** protects specific struct pointer fields against use-after-free exploitation via tag signing on store and tag verification on load; lowers to PAC instructions on AArch64 MTE-capable hardware; deactivation symbols allow link-time opt-out for performance-critical paths.


---

@copyright jreuben11
