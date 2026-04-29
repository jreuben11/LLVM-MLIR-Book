# Chapter 68 — Hardening and Mitigations

*Part X — Analysis and the Middle-End*

Security hardening transforms program code and data layout to raise the cost of exploitation. LLVM provides a comprehensive suite of mitigations: control-flow integrity (CFI), safe and shadow call stacks, speculative load hardening, stack clash protection, stack protectors, pointer authentication (PAuth/BTI), Intel CET/IBT, and FORTIFY_SOURCE-style bounds checking. This chapter covers each mitigation — what attack it defends against, how it is implemented in LLVM, and its performance cost.

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


---

@copyright jreuben11
