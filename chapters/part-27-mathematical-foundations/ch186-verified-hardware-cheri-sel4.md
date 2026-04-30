# Chapter 186 — Verified Hardware: CHERI Capabilities and the seL4 Microkernel

*Part XXVII — Mathematical Foundations and Verified Systems*

Formal compiler verification, as embodied by CompCert ([Chapter 168 — CompCert: Verified C Compilation](../part-24-verified-compilation/ch168-compcert.md)), proves that the transformation from source semantics to machine-code semantics is behavior-preserving. But that proof assumes a trusted hardware model: the processor faithfully executes the instruction encoding, and the memory system enforces no access control beyond what software arranges through page tables and privilege levels. Both assumptions fail in practice. Hardware bugs corrupt executions silently; spatial memory errors enable exploits even in correctly-compiled code; and an unverified kernel can undermine the security properties of every process running above it. This chapter examines two complementary techniques — CHERI capability hardware and the seL4 microkernel — that push formal guarantees down into the hardware-software boundary and upward into the operating system layer, leaving only a thin, independently auditable trusted base. The connection to the compiler toolchain is direct: CHERI requires a modified LLVM backend with new address spaces, new intrinsics, and new ABI conventions ([Chapter 41 — Calls, the ABI Boundary, and Builtins](../part-06-clang-codegen/ch41-calls-abi-builtins.md)); seL4 is verified against a Coq/Isabelle proof stack that closes the gap left open by the CompCert trusted base; and the Sail ISA specification language provides the formal ground truth that links hardware designs, simulator implementations, and LLVM backend correctness for RISC-V ([Chapter 98 — The RISC-V Backend Architecture](../part-15-targets/ch98-riscv-backend.md)) and AArch64 ([Chapter 96 — The AArch64 Backend](../part-15-targets/ch96-aarch64-backend.md)).

---

## 186.1 The Hardware-Software Co-Verification Problem

### 186.1.1 What CompCert Leaves Open

CompCert's correctness theorem is conditional on a *trusted base* that is left unverified. The base includes: the Coq proof checker itself (approximately 10,000 lines of OCaml); the OCaml extraction mechanism; the C standard library used by the extracted compiler binary; and, critically, the hardware on which the compiled code runs. The Coq semantics models the x86/AArch64 instruction set as an abstract relation over machine states. If the processor deviates from that relation — through a speculative execution bug, a microcode defect, or an undocumented behavior — the formal guarantee evaporates.

The hardware trust problem is not hypothetical. Spectre and Meltdown (2018) demonstrated that speculative execution violates the security-relevant behavioral model that all operating system proofs assume. The ROWHAMMER attack exploits DRAM physics in ways no ISA semantics captures. Production microprocessors contain tens of billions of transistors validated only by simulation and testing, not formal proof.

The seL4 security theorems — integrity and information-flow noninterference — similarly depend on a hardware model assumed correct. The seL4 proof shows that *if the hardware executes instructions as specified* and *if the C compiler produces code that implements the C semantics* (an assumption CompCert closes), *then* no unprivileged process can read or write kernel state without authorization. Both conditions must hold simultaneously; neither alone is sufficient.

### 186.1.2 Two Complementary Approaches

CHERI and seL4 attack the trust problem at different layers:

**CHERI** (Capability Hardware Enhanced RISC Instructions) moves memory access control into the hardware itself, making spatial safety a property of the ISA rather than of software instrumentation. Where AddressSanitizer ([Chapter 110 — User-Space Sanitizers](../part-16-jit-sanitizers/ch110-user-space-sanitizers.md)) and HWASan/MTE ([Chapter 111 — HWASan and MTE](../part-16-jit-sanitizers/ch111-hwasan-mte.md)) add software overhead and provide probabilistic or sampled detection, CHERI provides deterministic, zero-overhead enforcement (for non-violating code) at the hardware level. Every pointer carries its own bounds, permissions, and validity bit enforced by the CPU on every dereference.

**seL4** provides a formally verified operating system kernel: a machine-checked proof that the C implementation of the kernel faithfully implements an abstract specification, which is itself proven to enforce integrity and information-flow policies. The proof is carried out in Isabelle/HOL ([Chapter 184 — Proof Assistant Internals: Lean 4, Coq/Rocq, and Isabelle/HOL](../part-27-mathematical-foundations/ch184-proof-assistant-internals-lean4-coq-isabelle.md)) across approximately 200,000 lines of proof script.

The interaction between the two is not accidental. CherIoT — Microsoft's RISC-V32 IoT RTOS — combines CHERI hardware capabilities with seL4-style software capability access control as orthogonal and mutually reinforcing mechanisms, providing defense-in-depth: hardware stops out-of-bounds memory accesses regardless of software policy, while software capabilities enforce coarse-grained object access regardless of spatial proximity.

---

## 186.2 CHERI Capabilities

### 186.2.1 The Capability Format

A CHERI capability is a 128-bit extended pointer: a 64-bit virtual address augmented with 64 bits of metadata. The hardware additionally maintains one **tag bit** per 128-bit capability-aligned memory slot, stored outside the addressable memory space in dedicated on-chip tag SRAM. The tag bit is what makes the capability system unforgeable.

The 128-bit layout (as specified in the CHERI ISA specification v9, [Watson et al., UCAM-CL-TR-987](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-987.pdf)) encodes:

- **Address** (64 bits): the current cursor, the actual virtual address used for memory operations.
- **Compressed bounds** (encoded in the upper half): a 64-bit integer address combined with a compressed bounds representation that encodes `base` and `top` using a floating-point-like exponent scheme. The bounds encode the start address and length of the region the capability authorizes access to. Full precision (exact representability) is available for small objects; for large objects the bounds are rounded outward to the next power-of-two alignment according to an exponent value `E`. This means that `CSetBounds` may need to widen bounds slightly when the requested length is not exactly representable at the given alignment — the `CSetBoundsExact` instruction traps rather than silently rounding.
- **Permissions** (17 bits, packed into the upper half): a bitmask controlling what operations the capability authorizes. The defined permission bits include: `Load`, `Store`, `Execute`, `LoadCap` (load a capability from memory), `StoreCap` (store a capability to memory), `StoreLocalCap` (store a local capability), `Seal`, `Unseal`, `System`, `BranchSealedPair`, `CompartmentID`, `MutableLoad`, `AccessSystemRegs`, `Executive`, `Global`, `Permit_CCall`, and an architectural reserved bit. The full authoritative list is in the CHERI ISA specification.
- **Sealed bit and Object Type**: a sealed capability cannot have its address or metadata modified in software. The object type field (encoded in the upper bits) identifies the sealing type used, enabling type-safe invocation of sealed capabilities. The special object types `UNSEALED` (all-ones or zero depending on encoding), `SENTRY` (sealed for invocation), and `RB` (return capability) have hardware-defined semantics.

The tag bit is the invariant that makes forgery impossible. The tag is set only by the processor when it executes a capability-creating instruction on a valid capability. Ordinary store instructions to capability-aligned memory slots *clear the tag*. The only way to create a capability with the tag set is to derive it from an existing tagged capability using capability-narrowing instructions. Because the tag lives outside addressable memory, it cannot be read, written, or forged by integer arithmetic or byte-level stores — there is literally no address in the virtual address space from which to read or write it.

### 186.2.2 The Monotonicity Guarantee

The central security property of CHERI is **capability monotonicity**: no unprivileged instruction can produce a capability with bounds wider than the input capability, or with permissions that are a strict superset of the input permissions. The instruction set enforces this by construction:

- `CSetBounds cap, base, length` — produces a new capability with the same permissions as `cap` but with bounds `[base, base+length)` intersected with `cap`'s bounds. The new bounds can only be equal to or narrower than the input. If `base < cap.base` or `base+length > cap.top`, the instruction clears the tag (the result is an invalid capability).
- `CSetBoundsExact cap, base, length` — same, but traps if the bounds would need to be rounded (requires exact representability at the object's alignment).
- `CAndPerm cap, mask` — bitwise-ANDs the permissions field of `cap` with `mask`. Since AND can only clear bits, permissions can only be removed, never added.
- `CSeal cap, otype` — seals `cap` using the object type encoded in `otype`; requires that `otype` carry the `Seal` permission. The sealed capability has reduced usability (cannot load/store through it directly) but can be invoked through `CInvoke`/`CCall` with appropriate permissions.

No instruction can widen bounds or add permissions. The only way to obtain a capability with wide bounds is to start with one — and the only source of maximally-privileged capabilities is the hardware reset state (the so-called *root capabilities* with which the firmware initializes the first kernel), which must be explicitly narrowed and delegated through the capability derivation chain.

### 186.2.3 Memory Safety Semantics

CHERI enforces three orthogonal memory safety properties:

**Spatial safety**: every memory access through a capability checks that the target address falls within `[cap.base, cap.top)`. Out-of-bounds accesses trap with a CHERI capability exception. This is hardware-enforced bounds checking with no runtime overhead for in-bounds accesses and zero false negatives — unlike ASan's shadow memory (which misses some intra-object overflows) or HWASan's 4-bit color tags (which provide probabilistic detection).

**Referential integrity**: the tag bit ensures that a program cannot construct a capability by assembling bits from integers. A 128-bit value assembled by integer operations always has tag=0 and cannot be used as a pointer. This closes the class of vulnerabilities that rely on integer-to-pointer conversions to manufacture fake pointers.

**Permission enforcement**: the permissions field ensures that a capability obtained for reading cannot be used for writing, and that code-pointer capabilities cannot be used for data access. The `LoadCap` and `StoreCap` permissions separately control the loading and storing of tagged capabilities, enabling compartmentalization: a component that lacks `StoreCap` cannot leak capabilities to untrusted memory.

**Temporal safety via Cornucopia**: CHERI hardware alone does not prevent use-after-free. The Cornucopia system (Xia et al., 2019) adds temporal safety via a capability revocation GC: a background sweeper walks physical memory looking for capabilities that point into freed regions and clears their tags. Because all capabilities are 128-bit-aligned and tagged, the sweeper can enumerate all live capabilities in memory without any type information. Cornucopia has been deployed in CheriBSD with measured overhead of approximately 2% for typical workloads.

The contrast with software-based sanitizers is instructive. ASan adds approximately 2× memory overhead and 50–100% runtime overhead; it is a debugging tool, not a production security mechanism. HWASan uses ARM's Memory Tagging Extension (MTE) for 4-bit probabilistic tags with approximately 3–5% overhead in production. CHERI provides deterministic enforcement at zero overhead for non-violating code — the only overhead is the 2× memory footprint for capabilities (pointers grow from 8 to 16 bytes) and the initial bounds-setting at allocation time.

---

## 186.3 CHERI ISA Variants

The CHERI architecture has been implemented across four ISA families. The following table summarizes the key distinctions:

| Variant | Base ISA | Pointer Width | Status | Primary Use |
|---|---|---|---|---|
| CHERI-MIPS | MIPS64 | 128-bit | Research only | Original Cambridge prototype |
| CHERI-RISC-V | RISC-V64 | 128+tag | Active research platform | CheriBSD, Sail formal spec |
| Morello | AArch64 | 128+tag | Production silicon | CheriBSD, Android research |
| CherIoT | RISC-V32 | 64+tag | Production IoT RTOS | Microsoft IoT embedded |

### 186.3.1 CHERI-RISC-V

CHERI-RISC-V is the primary research platform, co-designed with the official RISC-V Sail formal specification. Capabilities are 129-bit (128 bits of data plus the out-of-band hardware tag bit). The key new instructions extend the RISC-V encoding:

```
# Capability register file: c0–c31 (each 129 bits; c0 = NULL capability)
# Capability Load/Store:
lc   cd, offset(cb)   # Load 128-bit capability from [cb.address + offset]
sc   cs, offset(cb)   # Store capability to [cb.address + offset]
# Bounds manipulation:
csetbounds    cd, cb, rs     # Set bounds: cd = cb with bounds narrowed to [cb+rs)
csetboundsimm cd, cb, imm   # Immediate form (12-bit unsigned)
# Inspection:
cgetbase  rd, cb    # Get base address of capability
cgetlen   rd, cb    # Get length (top - base) of capability
cgetperm  rd, cb    # Get permissions bitmask
cgetaddr  rd, cb    # Get current address cursor
# Address/permission modification (monotone):
candperm  cd, cb, rs    # AND permissions
csetaddr  cd, cb, rs    # Set address (monotone if within bounds)
cincoffset cd, cb, rs   # Increment address by offset
```

The `__capability` qualifier in Clang marks a pointer type as capability-qualified, causing the compiler to use the capability register file and capability load/store instructions. The `__intcap_t` type is a capability-sized integer type that carries the metadata of the null capability at compile time and enables integer arithmetic on capability values with controlled semantics.

The CHERI-RISC-V Sail model at [`github.com/riscv/sail-riscv`](https://github.com/riscv/sail-riscv) integrates CHERI extensions as a formally specified modification to the base RISC-V model, making the CHERI behavior an explicit part of the official Sail RISC-V specification. This is the formal ground truth against which both hardware implementations and the CHERI LLVM backend are validated.

### 186.3.2 Morello (Arm AArch64 + CHERI)

Morello is a production silicon implementation of CHERI on the Arm Neoverse N1 microarchitecture, produced as part of a UK government/DARPA research program. Morello adds a 129-register capability file alongside the standard 31-register integer file. The addressing modes, instruction encoding, and calling conventions are extended to support both legacy AArch64 code (pure-integer mode) and capability mode.

The key architectural decisions in Morello differ from CHERI-RISC-V in ways that reflect the constraints of production silicon and compatibility with existing software:

- Capability registers `c0`–`c30` correspond to 64-bit integer registers `x0`–`x30`; `csp` is the capability stack pointer; `ddc` is the default data capability (used by legacy non-capability loads/stores).
- The Morello ABI uses capabilities for pointers in pure-capability (PCuABI) mode; in legacy mode, the integer ABI applies and capabilities are only used explicitly via `__capability`-qualified code.
- The Morello LLVM backend implements `MorelloTargetMachine`, derived from the AArch64 target, adding the capability register class, capability instruction selection patterns, and the PCuABI calling convention.

CheriBSD, the research OS for CHERI platforms, runs on both Morello hardware and CHERI-RISC-V via QEMU emulation. Google has published a research port of the Android software stack to Morello, demonstrating compatibility of the major Android components with pure-capability ABI.

### 186.3.3 CherIoT

CherIoT (Microsoft Research, 2023) targets RISC-V32 for IoT bare-metal systems. The design makes several deliberate simplifications relative to full CHERI-RISC-V:

- 32-bit address space (64-bit capabilities with 32-bit address and 32 bits of metadata).
- Capability bounds are always exactly representable (no exponent-based compression needed for 32-bit address spaces).
- Combined hardware capability checks with a seL4-style software capability management layer in the RTOS kernel (~25,000 lines of C++).
- Hardware support for *revocation* baked in: every heap allocation gets a unique revocation cookie, enabling deterministic use-after-free prevention.
- Compartment model: each software compartment has a sealed entry point capability; cross-compartment calls are mediated by the hardware through a `CInvoke` mechanism.

CherIoT demonstrates that CHERI and seL4-style software capabilities are not competing but *complementary*: hardware capabilities enforce spatial safety and prevent capability forgery at the hardware level; software capabilities enforce coarse-grained access control (which compartment may invoke which capability) at the policy level. The combination provides defense-in-depth without requiring either mechanism to be alone sufficient.

---

## 186.4 CHERI in LLVM

### 186.4.1 Address Space 200 and Non-Integral Pointers

The CHERI LLVM fork ([`github.com/CTSRD-CHERI/llvm-project`](https://github.com/CTSRD-CHERI/llvm-project)) introduces address space 200 as the CHERI capability address space. This mirrors the design used by AMDGPU (address space 8 for buffer descriptors) but with critical differences that reflect the nature of capability hardware.

LLVM's `non-integral` address space annotation (introduced for AMDGPU) marks an address space in which pointers cannot be converted to/from integers via `ptrtoint`/`inttoptr` without losing semantics. CHERI capabilities are non-integral pointers: a 128-bit capability contains a 64-bit address plus 64 bits of metadata, but the metadata includes the *out-of-band hardware tag bit* that LLVM's integer type system cannot model. A `ptrtoint` of a CHERI capability discards the metadata; an `inttoptr` into address space 200 produces a capability with tag=0, which is an invalid (NULL-equivalent) capability for memory access purposes.

The target data layout string for CHERI-RISC-V includes:
```
-A200-P200-G200
```
This specifies address space 200 as the alloca address space (A200), code address space (P200), and global variable address space (G200), placing all pointer-typed allocations in the capability address space in pure-capability mode (CheriABI/PCuABI).

### 186.4.2 CHERI Intrinsics

The CHERI-LLVM backend defines capability-manipulation intrinsics in `llvm/lib/Target/CHERI/IntrinsicsCHERI.td` (CHERI-RISC-V) and analogous files for Morello. The key intrinsics:

```llvm
; Bounds manipulation
declare ptr addrspace(200) @llvm.cheri.cap.bounds.set(ptr addrspace(200), i64)
; Set bounds on capability; second arg is length
; Maps to: csetbounds (RISC-V), CSub+CSetBounds (Morello)

declare ptr addrspace(200) @llvm.cheri.cap.bounds.set.exact(ptr addrspace(200), i64)
; As above but traps if bounds are not exactly representable

; Permission manipulation
declare ptr addrspace(200) @llvm.cheri.cap.perms.and(ptr addrspace(200), i64)
; AND permissions mask; always monotone (can only remove permissions)

; Tag inspection
declare i1 @llvm.cheri.cap.tag.get(ptr addrspace(200))
; Returns the hardware tag bit; used in CHERI security assertions

; Address manipulation
declare i64 @llvm.cheri.cap.address.get(ptr addrspace(200))
declare ptr addrspace(200) @llvm.cheri.cap.address.set(ptr addrspace(200), i64)

; Null capability (tag=0)
declare ptr addrspace(200) @llvm.cheri.cap.null()

; Sealing/unsealing
declare ptr addrspace(200) @llvm.cheri.cap.seal(ptr addrspace(200), ptr addrspace(200))
declare ptr addrspace(200) @llvm.cheri.cap.unseal(ptr addrspace(200), ptr addrspace(200))
declare i64 @llvm.cheri.cap.type.get(ptr addrspace(200))
```

The corresponding Clang builtins provide the C/C++ programmer interface:

```c
#include <cheriintrin.h>

void *__capability cap = cheri_ptr(ptr, length);   // __builtin_cheri_bounds_set
void *__capability bounded = cheri_bounds_set(cap, new_length);  // CSetBounds
void *__capability restricted = cheri_perms_and(cap, CHERI_PERM_LOAD);
bool valid = cheri_tag_get(cap);                   // __builtin_cheri_tag_get
size_t len = cheri_length_get(cap);                // __builtin_cheri_length_get
ptraddr_t addr = cheri_address_get(cap);           // __builtin_cheri_address_get
```

Sub-object bounds enforcement — bounds set on individual struct fields and stack allocations — is performed by Clang automatically (with `-cheri-bounds=subobject`) by inserting `CSetBoundsImmediate` instructions when a field address is taken or a local variable address escapes. This transforms C's nominal type safety into hardware-enforced spatial safety without requiring source changes.

### 186.4.3 The CheriABI / PCuABI Calling Convention

In pure-capability mode (PCuABI — Pure-Capability UNIX Application Binary Interface), all pointers are capabilities. This requires systematic changes to the ABI:

- **`sizeof(void*)` = 16** (128 bits). All pointer-sized fields in structs, the stack frame layout, and the system call interface use 16-byte aligned slots.
- **Function pointer passing**: function pointers are sealed capabilities (sentry capabilities) with the `Execute` permission and without `Load`/`Store`. The `Seal` with object type `SENTRY` prevents modification of the function pointer value; the hardware `CJR` (capability jump register) instruction validates the sentry on indirect calls.
- **Varargs with capabilities**: the varargs calling convention is extended to handle 16-byte capability arguments. The `va_list` type contains a capability to the vararg region; `va_arg` for capability types fetches a full 128-bit value.
- **System call boundary**: kernel entry validates that argument capabilities are tagged and within the expected permission set. The kernel ABI enforces that no untagged 128-bit value (integer masquerading as a capability) can be passed as a capability argument.

The LLVM `CallingConv::CHERI_CCall` calling convention and related metadata handle the register allocation differences between legacy and capability ABIs at the IR level.

### 186.4.4 Capability-Aware memcpy and memmove

A critical correctness requirement in CHERI is that `memcpy` and `memmove` preserve capability tags. A naive byte-level copy would read 8 bytes at a time from a 128-bit capability, writing them back without ever performing a capability-granularity load/store — resulting in all destination tags being cleared. CHERI LLVM therefore:

1. Lowers `memcpy` to a loop using `lc`/`sc` (capability load/store) for 16-byte-aligned, capability-sized copies, preserving tags.
2. For unaligned or byte-granularity copies, uses CHERI-specific `memcpy` library routines that handle the mixed case.
3. Provides `@llvm.memcpy.element.unordered.atomic.p200i8.p200i8.i64` variants that respect CHERI alignment requirements.

This is a fundamental difference from the AMDGPU non-integral pointer case: AMDGPU buffer descriptors are purely in-band (all 128 bits are addressable metadata), while CHERI capabilities have out-of-band state (the hardware tag) that must be actively preserved by copy operations.

---

## 186.5 The Sail ISA Specification Language

### 186.5.1 Sail as a Typed DSL

Sail ([Armstrong et al., POPL 2019](https://dl.acm.org/doi/10.1145/3290384)) is a domain-specific language for writing machine-readable ISA specifications. It provides bitvector types as first-class citizens, effect tracking, and formal extraction to multiple target languages. A Sail function for a RISC-V instruction looks like:

```sail
-- CHERI-RISC-V CSetBounds instruction (simplified from the official model)
function clause execute(CSetBounds(cd, cb, rs)) = {
  let cb_val : Capability = C(cb);
  let len    : bits(64)   = X(rs);
  if   ~ (cb_val.tag)
  then { handle_cheri_exception(CapEx_TagViolation); RETIRE_FAIL }
  else if isSealed(cb_val)
  then { handle_cheri_exception(CapEx_SealViolation); RETIRE_FAIL }
  else {
    let base : bits(64) = cb_val.base;
    let new_top : bits(65) = zero_extend(base) + zero_extend(len);
    let old_top : bits(65) = getTop(cb_val);
    if   (zero_extend(base) <_u zero_extend(cb_val.base)) |
         (new_top >_u old_top)
    then { handle_cheri_exception(CapEx_LengthViolation); RETIRE_FAIL }
    else {
      let (exact, new_cap) = setBounds(cb_val, base, len);
      let new_cap' = clearTag_if_not_exact(new_cap, ~ exact);
      C(cd) = new_cap';
      RETIRE_SUCCESS
    }
  }
}
```

Key Sail language features visible here:

- **`bits(N)`**: bitvector type of width N. Width polymorphism allows functions to work over multiple widths with constraints.
- **`Capability`**: a user-defined record type for capability metadata; Sail's type system ensures field access is width-correct.
- **Effect tracking**: Sail tracks whether a function reads/writes registers or memory via its type system, enabling static reasoning about side effects.
- **`clause` dispatch**: instruction semantics are defined as clauses on the `execute` function, dispatched on the decoded instruction value. This mirrors LLVM's `TableGen` `def` mechanism for instruction descriptions.

### 186.5.2 The Official RISC-V Sail Model

The RISC-V Foundation maintains an official Sail model at [`github.com/riscv/sail-riscv`](https://github.com/riscv/sail-riscv). The model covers: base RV32I/RV64I, the M/A/F/D/C standard extensions, the Privileged Architecture (S/U/M modes, physical memory protection), and the CHERI-RISC-V extension. It serves three official functions:

1. **Compliance test suite generation**: Sail-to-C extraction produces a reference simulator; the simulator generates the golden outputs for the RISC-V compliance test suite, which hardware and software implementations must pass.
2. **Formal property verification**: Sail-to-Isabelle/HOL extraction produces an Isabelle theory that can be used to prove properties of the ISA. Armstrong et al. proved register-safety properties (no instruction writes to a register it should not) and certain memory safety lemmas for the CHERI-RISC-V model in Isabelle.
3. **Reference implementation**: the Sail-to-C extracted simulator (the `riscv_sim_RV64` binary) is used in CheriBSD and CHERI-RISC-V bring-up as a reference against which hardware RTL simulation is validated.

### 186.5.3 Sail Extraction Targets and Their Uses

Sail supports three extraction backends:

**Sail-to-C**: produces a single-threaded C simulator with an explicit machine state struct. Used for the RISC-V compliance simulator, CheriBSD emulation (`cheribsd-utils qemu-cheri`), and hardware bring-up validation. The C output is not the *trusted* semantics — it is an executable approximation.

**Sail-to-Isabelle/HOL**: produces an Isabelle theory (`thy` file) formalizing the ISA as a monadic function over a machine state record. Each instruction clause becomes an Isabelle `definition`. The Sail-to-Isabelle translation preserves bitvector arithmetic via Isabelle's `Word` library. This extraction is the formal basis for proving ISA properties in Isabelle; it is used by the CHERI ISA team and by the ARM Sail model (which covers ARMv8-A at approximately 80,000 lines of Sail).

**Sail-to-Lem** (experimental): extraction to Lem, a lightweight functional language with backends to Isabelle, Coq, HOL4, and OCaml, enabling cross-proof-assistant ISA reasoning.

### 186.5.4 Sail and the LLVM RISC-V Backend

The relationship between the Sail formal model and the LLVM RISC-V backend is indirect but important. The Sail model defines what each instruction *must* do; the LLVM backend must produce instruction sequences whose net effect over machine state matches the Sail semantics. The validation chain is:

1. The Sail model is extracted to C and used to generate the compliance test suite.
2. The LLVM backend is tested against the compliance test suite via `llc`-generated code run in a RISC-V emulator.
3. For CHERI-specific instructions, the CHERI LLVM backend is validated by running the CHERI test suite against the Sail-extracted CHERI-RISC-V simulator.

This chain does not constitute a *proof* that the LLVM backend is correct with respect to the Sail model — only testing. Closing this gap formally would require a result analogous to the CompCert theorem for the LLVM backend, applied to the RISC-V Sail model as the target semantics. This remains an open research problem as of 2026.

---

## 186.6 seL4 Architecture

### 186.6.1 L4 Microkernel Principles

seL4 is a member of the L4 microkernel family, descended from Jochen Liedtke's original L4 (1994) and implemented by NICTA (now Data61/CSIRO) and the seL4 Foundation. The defining principle of L4-family kernels is the *IPC primitives are sufficient* thesis: all kernel services beyond address space management and thread scheduling can be implemented as user-space servers communicating over synchronous IPC. The kernel need provide only:

- Thread management and scheduling
- Virtual address space management (capability-controlled)
- IPC endpoints for synchronous and asynchronous message passing
- Capability management

Everything else — device drivers, file systems, networking stacks, POSIX compatibility layers — runs as user-space processes, sandboxed by capabilities. This minimality is what makes formal verification tractable: the seL4 kernel is approximately 10,000 lines of C, versus Linux at approximately 30,000,000.

The IPC model is synchronous by default: `seL4_Call` blocks the caller until the recipient calls `seL4_ReplyRecv`. Asynchronous notification objects allow one-bit event signaling without blocking. The performance overhead of IPC has been engineered to be minimal: a seL4 IPC round-trip on AArch64 is approximately 250 cycles on modern hardware, competitive with Linux system call overhead.

### 186.6.2 Kernel Objects

seL4 manages exactly these kernel object types:

**Thread Control Blocks (TCBs)**: represent executable threads. Each TCB holds the thread's register state, scheduling parameters (priority, maximum controlled priority), IPC buffer (a user-space page mapped into the thread's address space for message payload), and capability to its CSpace (capability table) and VSpace (virtual address space root).

**CNodes** (Capability Nodes): fixed-size arrays of capability slots. Each slot holds either an empty capability (`NullCap`) or a valid capability to some kernel object. CNodes are the building blocks of the *CSpace* (capability space) — the tree of CNodes that defines what capabilities a thread can address. A CSpace lookup descends the CNode tree using a bit-slice of the capability address.

**VSpaces**: root objects of the virtual address space, analogous to page table roots. On AArch64, a VSpace corresponds to a PGD (Page Global Directory). Page tables, page directories, and page frames are all typed kernel objects mapped into the VSpace hierarchy via capabilities.

**Endpoints**: bidirectional synchronous IPC channels. A thread that calls `seL4_Call(ep)` blocks on the endpoint; a thread that calls `seL4_Recv(ep)` blocks waiting for a caller. When both are ready, the kernel transfers the message and switches the calling thread to the receiving thread's reply capability.

**Notification objects**: lightweight semaphore-like objects. `seL4_Signal` sets a bit; `seL4_Wait` blocks until any bit is set. Used for interrupt notification and asynchronous event delivery.

**Untyped memory**: the fundamental source of all kernel objects. The bootloader passes a set of Untyped capabilities to the initial thread (the *root task*), each representing a contiguous range of physical memory. Kernel objects are created by invoking `seL4_Untyped_Retype` on an Untyped capability, which consumes a portion of the untyped memory and returns typed capabilities to the new objects. There is no `free` operation — once memory is typed, it must be explicitly *revoked* (returned to untyped) through the capability derivation tree.

**Reply objects** (seL4 post-3.0): explicit reply capabilities, replacing the implicit reply capability introduced by `seL4_Call`. Reply objects improve the formal model by making reply capability ownership explicit and enabling the scheduler to reason about which thread holds the reply token.

### 186.6.3 The Capability Derivation Tree

Every capability in seL4 exists within the *Capability Derivation Tree* (CDT), also called the *derivation tree* or *MDB* (model database). The CDT is a forest of capability copies: when a capability is *minted* (copied with equal or restricted rights) or *derived* (a child Untyped is created from a parent Untyped), the new capability becomes a child of the source in the CDT.

The CDT has one critical invariant: **`seL4_CNode_Revoke`** on a capability recursively revokes all descendants in the CDT. This enables clean resource reclamation: the owner of a capability can revoke all delegations atomically. A device driver process can be terminated by revoking the root capability to its memory region, which cascades through all delegated capabilities to that region — even if the driver has delegated access to other processes.

The CDT is implemented as a doubly-linked list threaded through the capability slots, with a `depth` field that encodes the parent-child relationship. The seL4 proof establishes that CDT invariants are preserved by every kernel operation.

---

## 186.7 The Three-Level Refinement Proof

### 186.7.1 Proof Architecture

The seL4 formal verification ([Klein et al., SOSP 2009](https://dl.acm.org/doi/10.1145/1629575.1629596)) proves that the seL4 C implementation correctly implements a high-level abstract specification. The proof is structured as a *three-level refinement*:

```
Abstract Specification  (Isabelle/HOL, total function over abstract state)
          ↕ Refinement proof (Abstract → Executable)
Executable Specification (Haskell-prototype-style functional model in Isabelle)
          ↕ Refinement proof (Executable → C)
C Implementation        (via C parser → Simpl → AutoCorres)
```

Each arrow represents a simulation proof: every execution of the lower level has a corresponding execution of the upper level that produces the same observable outputs.

### 186.7.2 The Abstract Specification

The abstract specification defines the seL4 kernel interface as a total, nondeterministic function over an abstract kernel state. The state is a mathematical object — a record of functions and sets — with no reference to memory layouts, C types, or machine registers. In Isabelle/HOL notation (simplified from `l4v/spec/abstract/`):

```isabelle
-- Abstract kernel state
record abstract_state =
  ksCurThread    :: thread_ref
  ksReadyQueues  :: "priority ⇒ thread_ref list"
  ksCObjects     :: "obj_ref ⇒ kernel_object option"
  ksASIDTable    :: "asid_high_bits ⇒ obj_ref option"

-- System call handler as a total function over abstract_state
definition handleSyscall :: syscall → (unit, abstract_state) nondet_monad where
  "handleSyscall sc ≡ case sc of
     SysSend         ⇒ handleSend True
   | SysNBSend       ⇒ handleSend False
   | SysCall         ⇒ handleCall
   | SysWait         ⇒ handleWait True
   | SysReply        ⇒ handleReply
   | SysReplyWait    ⇒ handleReplyWait
   | SysYield        ⇒ handleYield
   | SysNBWait       ⇒ handleWait False"
```

The `nondet_monad` type represents a potentially-nondeterministic computation returning either a successful result or an error, along with the set of possible final states. Nondeterminism in the abstract spec accounts for scheduling choices; the refinement proof must show that the C code deterministically implements one branch of the nondeterminism.

### 186.7.3 The Executable Specification

The executable specification is a Haskell-prototype-style functional model that is close enough to the C implementation to admit a direct simulation proof, but high-level enough to be readable and to serve as a verified reference implementation. It appears in `l4v/spec/haskell/` and `l4v/spec/design/`.

The executable spec introduces C-realistic data structures: arrays with explicit indices, word-sized integers, machine-word bit manipulations. The key design decision is that the executable spec uses the *same* capability data structures as the C implementation, including the CDT invariants, the CNode tree structure, and the scheduler queue format. This makes the Executable → C refinement proof tractable (the data structures correspond directly) at the cost of a more complex Abstract → Executable proof.

```isabelle
-- Executable specification: decodeInvocation matches abstract capInvoke
lemma decodeInvocation_corres:
  "⟦ invs s; valid_cap cap s; ... ⟧ ⟹
   corres (inl_rrel dc) P P'
     (Syscall_A.capInvoke cap args)
     (decodeInvocation label args capIndex slot cap extraCaps)"
```

The `corres` predicate (`Corres.thy` in l4v) is the core refinement relation: `corres r P P' f g` states that when `f` runs in an abstract state satisfying `P` and `g` runs in a concrete state satisfying `P'` that correspond under the state relation, both either fail together or succeed with results related by `r`, and the output states correspond. Establishing `corres` for every kernel function is the bulk of the Abstract → Executable proof.

### 186.7.4 The C Implementation and AutoCorres

The C implementation consists of approximately 10,000 lines of restricted C in `src/` of the seL4 source tree. Key modules:

- `src/kernel/thread.c`: thread scheduling, `schedule()`, `switchToThread()`, `activateThread()`
- `src/object/endpoint.c`: IPC send/receive, `sendIPC()`, `receiveIPC()`
- `src/object/cnode.c`: capability operations, `decodeCNodeInvocation()`, `cteInsert()`, `cteRevoke()`
- `src/api/syscall.c`: system call dispatch, `handleSyscall()`, `handleInvocation()`

The C code is deliberately written in a style amenable to formal reasoning: no dynamic memory allocation, no recursion (except provably bounded), no pointer aliasing across kernel data structures (enforced by the `KernelState` typing invariant), and no undefined behavior (verified by the C parser).

**Michael Norrish's C parser** translates the seL4 C source into `Simpl`, a structured imperative language embedded as a shallow embedding in Isabelle/HOL. `Simpl` (Schirmer, 2006) provides: sequential composition, conditionals, loops with invariants, heap reads/writes, procedure calls with explicit pre/postconditions. The translation is *not* a trusted step — the C parser's correctness is itself proved in Isabelle, making the parser's output a certified intermediate representation of the C semantics.

**AutoCorres** (Greenaway, Andronick, Klein; ITP 2012) automates the abstraction from the byte-level heap representation (where all C values are stored as byte sequences in a typed heap) to a type-safe monadic form where C `int *` pointers access `int`-typed heaps directly. AutoCorres proves the heap abstraction theorem as part of its output, discharging the obligation that the abstraction is sound. The key Isabelle commands:

```isabelle
-- In a thy file for seL4 C verification:
install_C_file "src/object/endpoint.c"    (* invokes C parser *)
autocorres [ts_force pure = sendIPC] "endpoint.c"
(* AutoCorres abstracts heap model; ts_force pure = <fn> forces
   the abstraction of <fn> to a pure (non-heap-accessing) form *)
```

After AutoCorres, each C function `foo` has a corresponding `foo'` that operates on type-safe state, and a proof `foo_refines` that `foo'` simulates the original byte-level semantics. The Executable → C refinement proof then works with the AutoCorres output, relating `foo'` to the corresponding executable specification function.

### 186.7.5 What the Proofs Guarantee

The published seL4 verification results are:

**Functional correctness** (Klein et al., SOSP 2009): the C implementation is a correct implementation of the abstract specification. Every system call's observable behavior matches the specification.

**Integrity** (Sewell, Myreen, Klein, ITP 2011): no thread can write to kernel data structures it should not, and no unprivileged thread can modify the TCB, CSpace, or VSpace of another thread without possessing an appropriate capability. The integrity theorem is:

```isabelle
theorem integrity_enabled:
  "⟦ pas_refined aag s; invs s; (s, t) ∈ data_integrity aag ⟧
   ⟹ integrity aag X s t"
```

Where `pas_refined` asserts that the access control policy `aag` is correctly reflected in the kernel state, and `integrity aag X s t` asserts that transitioning from state `s` to state `t` does not violate the policy.

**Noninterference / Confidentiality** (Murray et al., S&P 2013): the information-flow policy is enforced. A high-security thread's state cannot influence the observable outputs of a low-security thread. This is the strongest guarantee: not only is writing unauthorized, but *reading* is prevented, and timing-channel analysis (within the model) establishes the absence of logical information leaks.

### 186.7.6 What the Proofs Do NOT Cover

The proofs make explicit assumptions that constitute the *trusted base*:

1. **Hardware correctness**: the proofs assume the processor executes instructions as specified by the ISA semantics used in the proof. Hardware bugs (Spectre, Meltdown, Rowhammer) violate this assumption. Spectre in particular fundamentally breaks the confidentiality proof because the model does not include speculative execution.

2. **Compiler correctness**: the proof establishes that the seL4 C source is correct. It does not establish that the compiled binary is correct. The C compiler (typically GCC for production seL4) is in the trusted base. **This is the gap that CompCert ([Chapter 168](../part-24-verified-compilation/ch168-compcert.md)) would close**: using CompCert to compile seL4 would extend the formal guarantee from the C source to the binary, replacing an unverified compiler with a verified one. CompCert-compiled seL4 has been produced in research settings for PowerPC and AArch64 targets.

3. **Timing side-channels**: the noninterference proof is over a logical model that treats all instructions as taking a single step with no timing information. Cache timing, branch predictor state, and memory bandwidth are not modeled; the confidentiality result says nothing about timing-based covert channels.

4. **Boot code and hardware initialization**: the proofs cover the kernel from the first C entry point onward. The hardware initialization code (assembly, firmware) is unverified.

5. **Multicore correctness**: the current seL4 proof covers the uniprocessor case. The multicore seL4 implementation exists and is deployed, but the formal multicore correctness proof is incomplete as of 2026 (discussed in §186.8.3).

---

## 186.8 AutoCorres, l4v, Deployments, and Multicore

### 186.8.1 The l4v Proof Repository

The `l4v` repository ([`github.com/seL4/l4v`](https://github.com/seL4/l4v)) contains the complete seL4 verification artifact: approximately 200,000 lines of Isabelle/HOL proof distributed across several layers:

| Directory | Contents | Approx. Lines |
|---|---|---|
| `spec/abstract/` | Abstract specification | ~8,000 |
| `spec/design/` | Executable specification | ~12,000 |
| `proof/refine/` | Abstract → Executable refinement | ~60,000 |
| `proof/crefine/` | Executable → C refinement | ~50,000 |
| `proof/access-control/` | Integrity and information-flow | ~40,000 |
| `proof/sep-capDL/` | Separation-logic capDL proofs | ~20,000 |
| `lib/` | Shared tactics and lemmas | ~10,000 |

The proof is machine-checked by Isabelle/HOL 2024 (using the `Word` library for bitvector arithmetic, `Monad_WP` for weakest-precondition reasoning over the nondeterministic monad, and `AutoCorres` as described). Running the full proof check from scratch takes approximately 6 hours on a modern workstation.

Key lemma names in the published proof:

- `refine_corres` in `proof/refine/AARCH64/Refine.thy`: the top-level simulation proof for Abstract → Executable on AArch64.
- `ccorres_corres_u_xf` in `proof/crefine/`: the mechanism connecting the `corres` relation (used in abstract refinement) to the `ccorres` relation (used in C refinement via AutoCorres).
- `integrity_trans` in `proof/access-control/`: transitivity of the integrity relation, enabling composition of single-step integrity proofs into multi-step guarantees.

### 186.8.2 CAmkES and System Composition

A correct seL4 kernel is necessary but not sufficient for a verified system: the user-space software must also be correctly composed. **CAmkES** (Component Architecture for Microkernel-based Embedded Systems) is the seL4 system description language for composing seL4-based applications.

A CAmkES description defines *components* (processes with typed interfaces), *connectors* (IPC channels, shared memory regions), and the *assembly* that wires components together. The `.camkes` language compiles to C stubs and seL4 initialization scripts:

```
/* Example CAmkES component: a driver with a single RPC interface */
component UARTDriver {
  provides IOInterface io;   /* server side of RPC interface */
  hardware;                  /* this component directly accesses hardware */
}

component Application {
  uses IOInterface io;       /* client side of RPC interface */
}

assembly {
  composition {
    component UARTDriver  driver;
    component Application app;
    connection seL4RPCCall rpc(from app.io, to driver.io);
  }
}
```

The `capDL` (Capability Description Language) describes the initial capability distribution for a seL4 system: which thread has which capability to which object. CAmkES generates a capDL specification from the system description, which is then used by the `capdl-loader` (a user-space process) to initialize the system from the untyped memory provided by the seL4 kernel.

A separation-logic proof of the capDL initialization (`proof/sep-capDL/` in l4v) establishes that if the system initializes with the capDL specification, the resulting system satisfies the intended capability distribution.

### 186.8.3 Verified Deployments

**DARPA HACMS** (High-Assurance Cyber Military Systems, 2012–2017): seL4 was deployed as the security kernel of the Boeing Unmanned Little Bird (ULB) helicopter's flight control software. The HACMS project demonstrated that a red team of professional security researchers could not compromise the mission computer after approximately 3 months of attempted attacks, attributed in part to seL4's formally verified isolation. The Boeing deployment used seL4 on ARM Cortex-A9.

**F-16 CDRL**: seL4 has been integrated into F-16 avionics systems under a contract delivery requirements list. The integration operates in a DO-178C (aerospace software certification) context, where seL4's formal proof is used as evidence for the software's integrity assurance level (DAL A requirements). This represents the first use of a formally verified OS kernel in certified avionics to our knowledge.

**seL4 on RISC-V**: the seL4 project ([`github.com/seL4/seL4`](https://github.com/seL4/seL4)) supports the `riscv64-unknown-elf` target. The Isabelle proof has been ported to RISC-V; the connection to the RISC-V Sail model (§186.5) means that in principle the hardware-trust assumption could be partially discharged by carrying the Sail-extracted ISA semantics directly into the Isabelle proof context.

### 186.8.4 Multicore seL4 and the CLH Lock Proof

The multicore seL4 implementation adds a global kernel lock (the Big Kernel Lock, BKL) protecting all kernel data structures, with per-CPU scheduling structures. The multicore code ships in production seL4 and is deployed, but the formal proof of multicore correctness remains incomplete as of April 2026.

The principal challenge is *rely/guarantee* reasoning for the interrupt model: when a core executing kernel code is interrupted by an inter-processor interrupt (IPI), the kernel state may be modified by the IPI handler on behalf of the interrupting core before the interrupted core's operation completes. Standard sequential refinement proof techniques do not apply; a concurrent refinement framework is required.

Recent progress: in 2024, the l4v team published a formal proof of the **CLH (Craig-Landin-Hagersten) lock** algorithm in Isabelle/HOL, using a rely/guarantee framework developed for seL4. The CLH lock is a cache-efficient queue lock used in multicore seL4 for protecting per-object locks (distinct from the BKL). The CLH correctness proof in Isabelle establishes mutual exclusion, freedom from deadlock, and FIFO ordering. This is the foundation for the full multicore seL4 proof; the full proof requires extending the rely/guarantee reasoning to the entire kernel's interrupt protocol.

### 186.8.5 seL4 and Rust

The `sel4-sys` crate provides raw Rust bindings to the seL4 system call interface:

```rust
// sel4-sys: raw bindings to seL4 IPC
use sel4_sys::*;

fn send_message(ep: seL4_CPtr, badge: seL4_Word) {
    unsafe {
        seL4_MessageInfo_t msg = seL4_MessageInfo_new(0, 0, 0, 1);
        seL4_SetMR(0, badge);
        seL4_Send(ep, msg);
    }
}
```

Higher-level Rust seL4 libraries (`seL4-microkit-rs`, `rust-sel4`) provide safe abstractions over the raw bindings, leveraging Rust's ownership model to statically enforce single-use of reply capabilities (a natural fit for Rust's `&mut` uniqueness guarantee). The combination of seL4's capability system and Rust's type system provides two independent layers of access control verification.

### 186.8.6 CherIoT: Hardware + Software Capabilities Combined

CherIoT exemplifies the design principle that CHERI hardware capabilities and seL4-style software capabilities are *orthogonal* and *complementary* rather than competing approaches:

- **CHERI hardware capabilities** (layer 1): enforce spatial safety and referential integrity at the hardware level. A buffer overrun in one compartment cannot reach into another compartment's memory, regardless of software policy.
- **Software capability management** (layer 2, seL4-style): the CherIoT RTOS kernel maintains a capability table controlling which compartments may invoke which software capabilities. A compartment that does not hold a capability to a network endpoint cannot call into the network stack, regardless of what it can compute.

The two layers are independent: CHERI stops hardware-level memory safety violations that would be invisible to the software capability system; the software capability system enforces coarse-grained object access that CHERI cannot express (e.g., "only the authentication module may invoke the cryptographic key store"). The ~25,000-line CherIoT kernel is designed for formal verification using similar proof techniques to seL4.

---

## 186.9 Summary

The landscape of hardware-software co-verification, as approached through CHERI and seL4, reveals both the achievable and the open:

**Key takeaways:**

- CHERI capabilities are 128-bit fat pointers with an out-of-band hardware tag bit that prevents forgery. The monotonicity guarantee — `CSetBounds` and `CAndPerm` can only narrow, never widen — is the core security property. No software instruction can widen bounds or add permissions.

- The hardware tag bit lives outside addressable memory (one bit per 128-bit capability-aligned slot), making it impossible to forge a valid capability through integer arithmetic or byte-level stores. This is categorically stronger than ASan's shadow memory or HWASan's probabilistic 4-bit MTE tags.

- CHERI-RISC-V (primary research platform), Morello (Arm production silicon), CherIoT (RISC-V32 IoT), and CHERI-MIPS (original prototype) are the four ISA variants; they share the monotonicity and tag invariants but differ in compression scheme, address width, and deployment context.

- The CHERI LLVM fork uses address space 200 as the capability address space and non-integral pointer semantics. Key intrinsics: `@llvm.cheri.cap.bounds.set`, `@llvm.cheri.cap.perms.and`, `@llvm.cheri.cap.tag.get`. The CheriABI/PCuABI ABI sets `sizeof(void*)` = 16 and requires capability-aware `memcpy` that preserves hardware tag bits.

- Sail is a typed DSL for ISA specifications with bitvector types, effect tracking, and formal extraction to C (for compliance simulators) and Isabelle/HOL (for formal proofs). The official RISC-V Sail model is the ground truth for CHERI-RISC-V semantics.

- seL4 is a formally verified L4 microkernel with approximately 10,000 lines of C and approximately 200,000 lines of Isabelle/HOL proof. The three-level refinement (Abstract → Executable → C) is proved in Isabelle using AutoCorres and the Norrish C parser. The l4v proof covers functional correctness, integrity, and information-flow noninterference.

- The seL4 trusted base includes hardware correctness (Spectre/Meltdown are not modeled), the C compiler (a gap that CompCert would close), timing side-channels (not modeled in the noninterference proof), and boot/initialization code.

- CAmkES enables composing seL4 applications from formally described components with capDL-verified capability distribution. Deployments include the DARPA HACMS Boeing ULB helicopter and F-16 avionics.

- Multicore seL4 is deployed but unproven; the CLH lock correctness proof (2024) is the most recent formal progress, using rely/guarantee reasoning as the foundation for the eventual full multicore proof.

- CherIoT demonstrates that CHERI hardware capabilities and seL4-style software capabilities are orthogonal and complementary: hardware enforces spatial safety; software enforces coarse-grained object access control. Neither alone is sufficient for the full threat model.

- The formal connection between Sail (ISA ground truth), seL4 (kernel verification), and CompCert (compiler verification) represents the most complete verified software stack currently available, covering hardware behavior, OS isolation, and compilation correctness with machine-checked proofs at each layer.

---

## References

- Watson, R.N.M., et al. *CHERI ISA Specification, Version 9*. Technical Report UCAM-CL-TR-987, University of Cambridge, 2023. [https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-987.pdf](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-987.pdf)
- Klein, G., et al. "seL4: Formal Verification of an OS Kernel." *Proc. ACM SOSP 2009*, pp. 207–220. [https://dl.acm.org/doi/10.1145/1629575.1629596](https://dl.acm.org/doi/10.1145/1629575.1629596)
- Murray, T., et al. "seL4: from General Purpose to a Proof of Information Flow Enforcement." *IEEE S&P 2013*, pp. 415–429.
- Armstrong, A., et al. "ISA Semantics for ARMv8-A, RISC-V, and CHERI-MIPS." *Proc. ACM POPL 2019*. [https://dl.acm.org/doi/10.1145/3290384](https://dl.acm.org/doi/10.1145/3290384)
- Woodruff, J., et al. "The CHERI Capability Model: Revisiting RISC in an Age of Risk." *Proc. ISCA 2014*.
- Xia, L., et al. "CHERIvoke: Characterising Pointer Revocation using CHERI Capabilities for Temporal Memory Safety." *Proc. MICRO 2019*.
- Sewell, T.A.L., Myreen, M.O., Klein, G. "Translation Validation for a Verified OS Kernel." *Proc. ACM PLDI 2013*.
- Greenaway, D., Andronick, J., Klein, G. "Bridging the Gap: Automatic Verified Abstraction of C." *Proc. ITP 2012*.
- Schirmer, N. *Verification of Sequential Imperative Programs in Isabelle/HOL*. PhD thesis, TU München, 2006. (Simpl formalization)
- Elliott, A., et al. "CherIoT: Rethinking security for low-cost embedded systems." *Proc. MICRO 2023*.
- CHERI LLVM project: [https://github.com/CTSRD-CHERI/llvm-project](https://github.com/CTSRD-CHERI/llvm-project)
- l4v proof repository: [https://github.com/seL4/l4v](https://github.com/seL4/l4v)
- seL4 kernel source: [https://github.com/seL4/seL4](https://github.com/seL4/seL4)
- sail-riscv: [https://github.com/riscv/sail-riscv](https://github.com/riscv/sail-riscv)
- Sail language: [https://github.com/rems-project/sail](https://github.com/rems-project/sail)
- [Chapter 41 — Calls, the ABI Boundary, and Builtins](../part-06-clang-codegen/ch41-calls-abi-builtins.md)
- [Chapter 96 — The AArch64 Backend](../part-15-targets/ch96-aarch64-backend.md)
- [Chapter 98 — The RISC-V Backend Architecture](../part-15-targets/ch98-riscv-backend.md)
- [Chapter 110 — User-Space Sanitizers](../part-16-jit-sanitizers/ch110-user-space-sanitizers.md)
- [Chapter 111 — HWASan and MTE](../part-16-jit-sanitizers/ch111-hwasan-mte.md)
- [Chapter 168 — CompCert: Verified C Compilation](../part-24-verified-compilation/ch168-compcert.md)
- [Chapter 184 — Proof Assistant Internals: Lean 4, Coq/Rocq, and Isabelle/HOL](../part-27-mathematical-foundations/ch184-proof-assistant-internals-lean4-coq-isabelle.md)
- [Chapter 185 — Mathematical Logic and Model Theory for Compiler Engineers](../part-27-mathematical-foundations/ch185-mathematical-logic-model-theory.md)

---

*@copyright jreuben11*
