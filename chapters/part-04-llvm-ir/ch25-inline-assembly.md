# Chapter 25 — Inline Assembly

*Part IV — LLVM IR*

Inline assembly is the mechanism through which high-level language code reaches down to raw machine instructions that the compiler cannot otherwise generate. It is the escape hatch for privileged operations, specialized CPU features, carefully sequenced hardware interactions, and low-level performance tuning that requires exact control over the instruction stream. Understanding how LLVM represents, lowers, and reasons about inline assembly is essential for systems programmers writing kernel code, drivers, cryptographic routines, and high-performance libraries, and for compiler engineers who must correctly handle the rich GCC-compatible constraint language that decades of systems software depend on.

This chapter covers the complete lifecycle of inline assembly in LLVM: the IR representation rooted in the `InlineAsm` value type, the AT&T and Intel dialects with their syntactic peculiarities, the constraint and clobber language that connects assembly operands to the register allocator, the `volatile` qualifier and its mapping to `sideeffect` in the IR, Clang's GCC-compatible extended-asm front end and its `asm goto` extension, module-level assembly declarations, the lowering path through `SelectionDAGBuilder::visitInlineAsm` to `ISD::INLINEASM` nodes and then to `MachineInstr`, and the verifier checks that catch common errors before they reach the backend. All IR examples are verified against LLVM 22.1.3.

---

## 25.1 Motivation: When Inline Assembly Is Necessary

The optimizer transforms source code according to the semantics of the language. C and C++ are defined in terms of an abstract machine; the compiler is free to reorder, combine, and eliminate operations provided the observable behavior of the abstract machine is preserved. This freedom is exactly what you want for general-purpose code, and exactly wrong for operations that require a specific interaction with the hardware.

Four categories of operation are not expressible as portable C or C++ and require inline assembly.

**Privileged and control-register instructions.** Kernel code must read and write system registers (`CR0`, `CR3`, `EFER`, `MSR`), execute privileged instructions (`HLT`, `WRMSR`, `INVLPG`, `LGDT`, `LIDT`), and control interrupt delivery (`STI`, `CLI`, `IRET`). None of these have C equivalents, and none are exposed as LLVM intrinsics for general use because user-mode code must not emit them.

**Instructions with no LLVM intrinsic.** The LLVM intrinsic set covers common SIMD and atomic patterns, but the full ISA surface—including bit-manipulation instructions (`BSF`, `BSR`, `LZCNT`), segment register operations, x87 transcendentals via `FSIN`/`FCOS`, and hardware-specific sequences like `MONITOR`/`MWAIT`, `XGETBV`, or vendor-specific micro-architectural hints—either has no intrinsic or the intrinsic was added only recently. Inline assembly bridges this gap without requiring a new compiler version.

**Precise ordering barriers.** Memory models depend on the correct placement of fences relative to loads and stores. The `mfence`, `lfence`, `sfence` instructions on x86 and `dmb`/`dsb`/`isb` on AArch64 must appear at exact points in the instruction stream. LLVM provides `llvm.fence` and atomic ordering, but the kernel's `READ_ONCE`/`WRITE_ONCE` idiom, hardware spinlock implementations, and lockless queue primitives sometimes need a memory clobber (`~{memory}`) attached to an inline asm to prevent the compiler from reordering surrounding memory accesses, even when no actual fence instruction is emitted.

**Cycle-count and performance counters.** `RDTSC`, `RDTSCP`, and `RDPMC` have no safe intrinsic representation because their semantics are deeply target-specific (serialize or not, affected by TSC ratio, ring-level accessibility). Profiling code reads them via inline asm. The `CPUID` instruction, which is used both to query CPU features and as a speculation barrier, falls into the same category: it serializes the instruction stream in a way that pure C cannot express.

A secondary motivation is preventing the optimizer from breaking carefully constructed sequences. Even when each individual instruction has an LLVM equivalent, combining them with `volatile` inline assembly ensures the compiler treats the entire block as opaque, preserving the ordering and preventing CSE or DCE from eliminating operations that look redundant to the optimizer but have observable side effects on hardware state.

---

## 25.2 The IR Representation: `InlineAsm` Value Type

In LLVM IR, an inline assembly block is represented as an `InlineAsm` value, defined in [`llvm/include/llvm/IR/InlineAsm.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/InlineAsm.h). `InlineAsm` inherits from `Value` and holds five components:

- **AsmString** — the raw assembly text, with `$N` placeholders for operands in AT&T dialect or `$N` / `%N` in Intel dialect.
- **Constraints** — a comma-delimited string encoding all output constraints, input constraints, clobbers, and label references.
- **HasSideEffects** — corresponds to the `sideeffect` token in IR text; maps to `volatile` in GCC asm.
- **IsAlignStack** — requests that the stack be aligned to the ABI maximum alignment before the asm block executes; uncommon outside Windows x64 calling convention usage.
- **Dialect** — an `InlineAsm::AsmDialect` enum: `AD_ATT` (default) or `AD_Intel`.
- **CanThrow** (LLVM 15+) — indicates the asm may raise an exception via unwind; extremely rare.

`InlineAsm` objects are uniqued via `ConstantUniqueMap<InlineAsm>`. The factory function is:

```cpp
static InlineAsm *InlineAsm::get(
    FunctionType *Ty,
    StringRef AsmString,
    StringRef Constraints,
    bool hasSideEffects,
    bool isAlignStack = false,
    AsmDialect asmDialect = AD_ATT,
    bool canThrow = false);
```

The `FunctionType` describes the signature of the asm block: its return type corresponds to the output operands (void for zero outputs, a scalar type for one output, a struct type for multiple outputs), and its parameter types correspond to the input operands.

An `InlineAsm` value is always used as the callee of a `call` instruction (or `callbr` for `asm goto`). Inline asm is never a function; it is an opaque token that instruction selection expands into actual machine instructions. The IR text representation uses the `asm` keyword:

```llvm
; call with inline asm callee, single i32 output, one i32 input
%1 = call i32 asm sideeffect "incl $0", "=r,0"(i32 %val)
```

The `sideeffect` token sets `HasSideEffects = true`. In IR text, `inteldialect` sets `Dialect = AD_Intel`. These tokens are emitted by `InlineAsm::getExtraInfoNames()` and parsed back by the IR parser.

The `Flag` inner class encodes per-operand metadata as a packed 32-bit integer. The low three bits encode the operand kind (`Kind::RegUse`, `Kind::RegDef`, `Kind::RegDefEarlyClobber`, `Kind::Clobber`, `Kind::Imm`, `Kind::Mem`, `Kind::Func`). Bits 15–3 hold the operand count. Bits 29–16 hold the register class ID or memory constraint code. Bit 31 marks tied operands. This encoding is used on the `ISD::INLINEASM` SelectionDAG node and on the lowered `MachineInstr`, where it appears as an immediate operand preceding each group of actual operands.

The module-level `module asm` construct is distinct from per-function inline asm: it appends text directly to the output assembly module:

```llvm
module asm ".section .note.GNU-stack,\22\22,@progbits"
module asm ".symver memcpy, memcpy@GLIBC_2.2.5"
```

This is processed by `AsmPrinter` via `Module::getModuleInlineAsm()` and emitted verbatim by `MCStreamer` before any function code. The LLVM IR parser accumulates multiple `module asm` strings with newline separators.

---

## 25.3 AT&T Dialect Syntax

The default AT&T dialect, derived from the Unix assembler syntax used by GCC, has several conventions that differ from most assembly documentation:

**Source before destination.** The AT&T convention places the source operand first and the destination second: `movl %eax, %ebx` copies `eax` into `ebx`. This is the opposite of Intel documentation order and is the source of most AT&T confusion.

**Size suffixes.** Instructions carry an explicit size suffix: `b` (byte, 8-bit), `w` (word, 16-bit), `l` (long, 32-bit), `q` (quad, 64-bit). `movq %rax, %rbx` is a 64-bit move. Without the suffix some instructions are ambiguous; the assembler may infer the size from operand types but inline asm in the kernel explicitly names the size.

**Register naming.** Register names are prefixed with `%`: `%rax`, `%rbx`, `%xmm0`, `%zmm15`. In GCC extended asm, `%0`, `%1`, `%2` reference the numbered constraint operands. The `%%` escape produces a literal `%` in the output, which the assembler then sees as a register prefix.

**Immediate prefix.** Immediate values are prefixed with `$`: `movl $42, %eax`. In inline asm the literal `$` must not be confused with the operand placeholder syntax `$0`, `$1`.

**Memory operand format.** Memory references use `offset(base, index, scale)` format: `8(%rbp)` is `[rbp + 8]`, and `(%rax, %rbx, 4)` is `[rax + rbx*4]`.

**Segment register access.** Segment-relative memory is written as `%gs:0` or `%fs:40`. The `%%` prefix is used: `movq %%fs:40, %0` reads 8 bytes at offset 40 from the FS segment base.

Common pitfalls: forgetting the `%%` prefix before register names (the assembler sees `%rax` but the inline asm template needs `%%rax` to produce it), confusing operand placeholder `$0` with the immediate prefix `$0` (the former is a compiler substitution, the latter is a literal zero), and forgetting to escape `$` in constraint strings when it appears as part of assembly (use `$$` to produce a literal `$`).

---

## 25.4 Intel Dialect Syntax

The Intel dialect reverses the operand order (destination first, source second), omits size suffixes (operand types are inferred from register names or explicit `DWORD PTR`, `QWORD PTR` qualifiers), and uses bare register names without `%` prefix.

To select Intel dialect in Clang source code, pass `-masm=intel` to the compiler. Clang then emits `inteldialect` in the IR:

```llvm
; Verified with llvm-as 22
define i32 @intel_add(i32 %a, i32 %b) {
  %ret = call i32 asm inteldialect "add $1, $0",
    "=r,r,0,~{dirflag},~{fpsr},~{flags}"(i32 %b, i32 %a)
  ret i32 %ret
}
```

In Intel dialect, `add $1, $0` means: add the value substituted for operand 1 into the register substituted for operand 0 (destination, then source). The constraint `0` on the second input operand ties it to output operand 0, meaning the same register is used for both input `a` and the output.

The `dialect=intel` flag can also appear in individual `asm` blocks in GCC extended syntax when the `-masm=intel` flag is active or via `__asm__ (".intel_syntax noprefix\n" ...)`, though using `-masm=intel` is cleaner and ensures consistent behavior across the translation unit.

Intel dialect inline asm does not permit explicit register naming with the `{regname}` syntax for output operands in the same way AT&T does. Both dialects share the constraint letter vocabulary; only the asm template text syntax differs.

---

## 25.5 Constraint Strings

Constraint strings are the interface between inline assembly operands and the compiler's register allocator and instruction selector. A full constraint string for a call with M output operands and N input operands looks like:

```
output0, output1, ..., outputM-1, input0, input1, ..., inputN-1, ~{clobber0}, ~{clobber1}, ...
```

Each token separated by commas corresponds to one operand or clobber entry. The `InlineAsm::ParseConstraints()` static method parses this string into a `ConstraintInfoVector`.

### 25.5.1 Output Constraints

Output constraints begin with `=` (write-only) or `+` (read-write). The underlying register class letter follows.

| Prefix | Meaning |
|--------|---------|
| `=r` | Output: write result into any general-purpose register |
| `=m` | Output: write result to a memory location (passes address) |
| `=&r` | Early-clobber output: result register may not overlap any input. The `&` means the output is written before all inputs are consumed. This prevents the allocator from assigning the same physical register to an input and this output when they need to coexist. |
| `+r` | Read-write: the operand is both an input (initial value) and output (final value) in the same register. Clang maps `+` to `=` with a matched input. |
| `=*m` | Indirect output: the corresponding call argument is a pointer; the asm writes through that pointer. In the IR, the pointer is passed directly and annotated with the `elementtype` attribute. |

In the IR text, `=r` becomes one output parameter in the return struct of the asm's `FunctionType`, while `=m` is encoded as `=*m` in the IR constraint string with the corresponding argument being the pointer.

### 25.5.2 Input Constraints

Input constraints have no prefix. Common letters:

| Letter | Meaning |
|--------|---------|
| `r` | Any general-purpose register |
| `m` | Any memory location |
| `i` | An integer immediate; the value must be a compile-time constant |
| `n` | A numeric integer immediate; narrower than `i` (for instructions that only accept small immediates) |
| `g` | General: register, memory, or immediate — the most permissive |
| `0`, `1`, ... | Tied operand: this input must use the same physical register as output operand N |
| `%0`, `%1` | Commutative with the next operand: the allocator may swap this and the following input |

### 25.5.3 Register-Class Letters: x86

x86 uses single-letter codes to name subsets of the register file:

| Letter | Register class | Notes |
|--------|---------------|-------|
| `a` | `{eax}`/`{rax}` only | Used for instructions that implicitly use EAX/RAX (e.g., MUL, DIV, CPUID input) |
| `b` | `{ebx}`/`{rbx}` only | Callee-saved; often used as the base register |
| `c` | `{ecx}`/`{rcx}` only | Shift-count register; LOOP instruction |
| `d` | `{edx}`/`{rdx}` only | High half of MUL/DIV result; RDX in RDTSC |
| `S` | `{esi}`/`{rsi}` only | String source index (MOVS, CMPS) |
| `D` | `{edi}`/`{rdi}` only | String destination index |
| `A` | EDX:EAX pair (32-bit) or RAX (64-bit) | Used for 64-bit results from 32-bit MUL: `"=A"` gets both halves |
| `r` | Any general-purpose register | Most common; equivalent to `q` in 64-bit mode |
| `q` | Legacy 8-bit accessible registers (EAX, EBX, ECX, EDX) in 32-bit; any GPR in 64-bit | |
| `Q` | EAX, EBX, ECX, EDX only (have 8-bit subregisters in 32-bit) | |
| `f` | x87 FP stack registers | Legacy; rarely used with SSE |
| `t` | Top of x87 stack (`%st(0)`) | |
| `u` | Second x87 stack element (`%st(1)`) | |
| `y` | MMX registers | Legacy |
| `x` | SSE/AVX XMM registers (lowest 16) | |
| `v` | SSE/AVX/AVX-512 vector registers | Including YMM/ZMM when target features enable them |
| `k` | AVX-512 mask registers (k0–k7) | |
| `Yz` | XMM0 specifically | For instructions that require a specific register |

The mapping from these letters to LLVM register classes is implemented in [`X86ISelLowering.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/X86/X86ISelLowering.cpp#L58846) via `X86TargetLowering::getRegForInlineAsmConstraint`.

### 25.5.4 Register-Class Letters: AArch64

| Letter | Register class |
|--------|---------------|
| `r` | General-purpose register: GPR32 for 32-bit values, GPR64 for 64-bit values |
| `w` | FP/SIMD/SVE register: FPR16/32/64/128 or ZPR for scalable vectors |
| `x` | Lower 128-bit SIMD register (FPR128_lo); required by some SIMD instructions |
| `y` | SVE ZPR register with 3-bit encoding |
| `m` | Memory operand |
| `{cc}` | NZCV condition code register |
| `{za}` | SME ZA matrix register |
| `{zt0}` | SME2 ZT0 lookup table register |

Implemented in [`AArch64ISelLowering.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AArch64/AArch64ISelLowering.cpp#L11665).

### 25.5.5 Register-Class Letters: RISC-V

| Letter / Constraint | Register class |
|---------------------|---------------|
| `r` | Any GPR (GPRNoX0 to avoid x0) |
| `f` | FP register: FPR16 / FPR32 / FPR64 depending on extension and value type |
| `vr` | Vector register (VR, VRM2, VRM4, or VRM8 as type-appropriate) |
| `vm` | Vector mask register (VMV0) |
| `{a0}`–`{a7}` | Explicit ABI name: maps to x10–x17 |
| `{s0}`/`{fp}` | Frame pointer alias: x8 |
| `{zero}` | x0 (hardwired zero) |

RISC-V supports both the architectural register names (`{x10}`) and ABI names (`{a0}`) in brace constraints. This is an LLVM extension to accommodate frontends that do not perform ABI-to-architectural name translation. Implemented in [`RISCVISelLowering.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/RISCV/RISCVISelLowering.cpp#L20498).

### 25.5.6 Tied Operands

A tied operand constraint is a digit referring to an earlier output operand. It tells the register allocator that the input must reside in the same physical register as the referenced output. This is the mechanism for read-modify-write operations:

```llvm
; add %b into %a (AT&T: addl src, dst), output and input share a register
%out = call i32 asm "addl $2, $0",
    "=r,=r,r,1,~{dirflag},~{fpsr},~{flags}"(i32 %extra, i32 %b)
```

Here constraint `1` ties the second input to output operand 1 (the second `=r`). The actual `add` operand `$0` is the first output register, `$2` is the third argument (`%b`). The instruction reads `$0` (same as the tied input) and writes back `$0`.

For simpler cases with one output and a read-modify-write, use `+r`:

```c
// C source:  __asm__ ("addl %1, %0" : "+r"(a) : "r"(b));
// Clang emits: call { i32, i32 } asm "addl $2, $0", "=r,=r,r,1,~{dirflag},~{fpsr},~{flags}"(...)
```

### 25.5.7 Multiple Alternative Constraints

Multiple alternatives allow the register allocator to choose between different operand encodings for the same instruction. The alternatives are separated by `|` in the constraint string, with each position specifying what register class or memory operand is acceptable:

```llvm
; Clang emits | as a separator for multi-alternative constraints
%ret = call i32 asm "movl $1, $0",
    "=r|r,r|m,~{dirflag},~{fpsr},~{flags}"(i32 %a)
```

The first alternative requires `(=r, r)` — output in a register, input in a register. The second alternative allows `(=r, m)` — output in a register, input from memory. The register allocator chooses the alternative that minimizes spills.

### 25.5.8 Named Operands

GCC extended asm supports naming operands with `[name]` syntax:

```c
__asm__ volatile (
    "movl %[val_a], %[tmp]\n\t"
    "movl %[val_b], %[val_a]\n\t"
    "movl %[tmp],   %[val_b]"
    : [val_a] "+m"(*a), [val_b] "+m"(*b), [tmp] "=&r"(tmp)
);
```

The `[name]` prefix is purely a source-level convenience. Clang resolves named operands to positional indices during `EmitAsmStmt` in [`clang/lib/CodeGen/CGStmt.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGStmt.cpp#L2630); the IR constraint string uses only numeric positions.

---

## 25.6 Clobber Lists

Clobbers declare registers and resources that the inline assembly modifies in ways not captured by the output constraints. They appear after the last input constraint, prefixed with `~`:

### 25.6.1 Register Clobbers

`~{rax}`, `~{rbx}`, `~{xmm3}`, etc. declare that the named register is modified by the asm block, even though it is not listed as an output. The register allocator must not assign these registers to live values across the asm. In the `InlineAsm::Flag` encoding, clobbers are represented with `Kind::Clobber`.

For example, `cpuid` modifies EAX, EBX, ECX, and EDX. If the input leaf value is in EAX and the four outputs are in those same four registers, all four must appear as outputs (not clobbers) to capture their values. But if you only care about EAX and will discard EBX/ECX/EDX, those three become clobbers:

```c
uint32_t cpuid_family(uint32_t leaf) {
    uint32_t family;
    __asm__ volatile ("cpuid"
        : "=a"(family)
        : "a"(leaf)
        : "ebx", "ecx", "edx");
    return family;
}
```

The IR generated by Clang 22 for a function with `"ebx", "ecx", "edx"` clobbers includes `~{bx},~{cx},~{dx}` in the constraint string (Clang normalizes 64-bit mode register names to their short form in the constraint string).

### 25.6.2 The `memory` Clobber

`~{memory}` is not a physical register clobber. It is a compiler directive instructing the optimizer to assume the asm block reads and writes arbitrary memory. Its effects are:

1. All cached values of memory locations are invalidated across the asm block. The compiler must reload from memory after the asm any value it had cached in a register.
2. All pending stores must be flushed to memory before the asm. The compiler cannot defer a store past the asm block.
3. The asm itself cannot be hoisted, sunk, or reordered with respect to surrounding memory operations.

This makes `~{memory}` an effective compiler-level barrier without emitting any hardware fence instruction. It is the mechanism used by the Linux kernel's `barrier()` macro:

```c
#define barrier() __asm__ volatile("" ::: "memory")
```

This emits no instructions but prevents the C compiler from reordering memory operations across it. Compare with `mfence`, which also prevents the CPU from reordering: `barrier()` controls the compiler, `mfence` controls the CPU.

In the generated IR, a `~{memory}` clobber appears in the constraint string and the asm node is marked `sideeffect`. The `AliasAnalysis` pipeline recognizes that a sideeffect asm with a memory clobber may access any memory.

### 25.6.3 Condition Code Clobbers

`~{cc}` declares that the asm modifies the processor's condition code (flags) register. On x86 this is EFLAGS; on AArch64 it is NZCV. Any arithmetic or comparison instruction modifies the flags. Without `~{cc}`, the compiler might assume flags set before the asm are still valid after it.

In LLVM IR, Clang automatically adds `~{dirflag},~{fpsr},~{flags}` to every x86 inline asm constraint string it emits, regardless of whether you requested them. This is visible in all the generated IR shown in this chapter. The `dirflag` clobber declares the x86 direction flag (DF) modified; `fpsr` declares the x87/SSE floating-point status register modified; `flags` is the general condition flags register.

---

## 25.7 The `volatile` Qualifier and `sideeffect`

The `volatile` qualifier on a GCC extended asm statement (`__asm__ volatile(...)`) sets `HasSideEffects = true` in the `InlineAsm` object. In IR text, this appears as the `sideeffect` keyword. The optimizer treats a sideeffect inline asm as an opaque side-effecting operation:

- It will not be removed as dead code, even if its outputs are unused.
- It will not be hoisted out of loops, even if its inputs are loop-invariant.
- It will not be CSE'd with an identical asm block elsewhere.
- It will not be reordered with other side-effecting operations (stores, calls to non-`readonly` functions, other `sideeffect` asm blocks).

Without `volatile` (i.e., `HasSideEffects = false`), the asm is treated like a pure function of its inputs. The optimizer may remove it if its outputs are never used, may hoist it out of loops, and may CSE it with other identical asm invocations.

```llvm
; Non-volatile (may be optimized away if result is unused)
%1 = call i32 asm "movl $1, $0", "=r,r"(i32 %x)

; Volatile (never removed, never reordered past other side effects)
%2 = call i32 asm sideeffect "movl $1, $0", "=r,r"(i32 %x)
```

The practical rule is: use `volatile` whenever the asm has any observable effect beyond its output values. This includes asm that modifies hardware state, asm that is a timing barrier, and asm that reads hardware counters. Omit `volatile` only for asm that computes a pure function of its inputs with no architectural side effects — extremely rare in practice.

When in doubt, use `volatile`. The cost is that the optimizer treats the block as a potential side effect, which is almost always the correct conservatism for inline asm.

---

## 25.8 GCC-Compatible Extended Asm in Clang

Clang's primary interface for inline assembly is GCC-compatible extended asm syntax. The full form is:

```c
__asm__ [volatile] (
    "asm template"
    : output-operands
    : input-operands
    : clobbers
);
```

Each section may be empty. The template string uses `%0`, `%1`, etc. to reference operands by position, or `%[name]` to reference named operands.

### 25.8.1 Operand Modifiers

Template substitutions support modifier characters that adjust how the operand is printed:

| Modifier | x86 Meaning | Example |
|----------|-------------|---------|
| `%c0` | Print as a bare immediate constant (no `$` prefix) | `movl %c0, %eax` → `movl 42, %eax` |
| `%n0` | Print negated immediate | Useful for subtraction encoded as add-negative |
| `%P0` | Print for use as a memory reference in PLT calls | Addresses without `@PLT` suffix |
| `%q0` | Print 64-bit (quad) register name | `%q0` → `%rax` even when `i32` type |
| `%k0` | Print 32-bit register name | `%k0` → `%eax` |
| `%w0` | Print 16-bit register name | `%w0` → `%ax` |
| `%b0` | Print 8-bit register name | `%b0` → `%al` |
| `%h0` | Print high-byte register name | `%h0` → `%ah` |
| `%l0` | Print as a label (for `asm goto` targets) | Used with label operands |
| `%a0` | Print as memory address (indirect, no `*` prefix in AT&T) | |

These modifiers are parsed by `X86AsmPrinter::PrintAsmOperand` in [`X86AsmPrinter.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/X86/X86AsmPrinter.cpp).

### 25.8.2 `asm goto` — Label Outputs

GCC 4.5 introduced `asm goto`, which allows inline assembly to transfer control to a C label. Clang supports it fully. The IR uses `callbr` rather than `call`:

```c
int check_bit(int val, int bit) {
    __asm__ goto (
        "bt %1, %0\n\t"
        "jc %l[bit_set]"
        :
        : "r"(val), "r"(bit)
        :
        : bit_set
    );
    return 0;
bit_set:
    return 1;
}
```

The label `bit_set` appears in the fourth (label) section of the asm statement. Clang emits:

```llvm
; Verified with llvm-as 22
define i32 @check_bit(i32 %val, i32 %bit) {
entry:
  callbr void asm sideeffect "bt $1, $0\0A\09jc ${2:l}",
    "r,r,!i,~{dirflag},~{fpsr},~{flags}"(i32 %val, i32 %bit)
    to label %fallthrough [label %bit_set]

fallthrough:
  ret i32 0

bit_set:
  ret i32 1
}
```

The `!i` constraint prefix marks a label operand. The `callbr` instruction has one normal destination (`fallthrough`) and one or more indirect destinations (`bit_set`). The SelectionDAG lowers `callbr` with an inline asm callee to an `ISD::INLINEASM_BR` node, handled by `SelectionDAGBuilder::visitCallBr` at line 3425 of [`SelectionDAGBuilder.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/SelectionDAG/SelectionDAGBuilder.cpp#L3425). The `%l[name]` modifier in the template expands to the label's target address.

`asm goto` with output values (a Linux kernel 5.13+ feature) is also supported in LLVM 22; the `callbr` may have both return values and label outputs simultaneously.

### 25.8.3 `__attribute__((naked))` Functions

A `naked` function has no compiler-generated prologue or epilogue. Combined with a single inline asm block constituting the entire function body, it gives the programmer complete control over the instruction stream. This pattern is used for interrupt service routines, context switchers, and handwritten assembly stubs:

```c
__attribute__((naked)) void my_isr(void) {
    __asm__ volatile (
        "pushq %rax\n\t"
        "pushq %rbx\n\t"
        /* ... handler body ... */
        "popq %rbx\n\t"
        "popq %rax\n\t"
        "iretq"
    );
}
```

Clang emits the `naked` function attribute into IR as `"frame-pointer"` omission. The backend's prologue/epilogue insertion pass (`PrologEpilogInserter`) checks for the `naked` attribute and skips all frame setup. The behavior of calling conventions, stack unwinding, and `__builtin_return_address` inside a naked function is undefined in the standard C sense; the programmer owns the calling convention.

### 25.8.4 Basic Constraints Example: RDTSC

The RDTSC instruction writes the 64-bit timestamp counter into EDX:EAX. The idiomatic implementation:

```c
#include <stdint.h>
uint64_t rdtsc(void) {
    uint32_t lo, hi;
    __asm__ volatile ("rdtsc" : "=a"(lo), "=d"(hi));
    return ((uint64_t)hi << 32) | lo;
}
```

Clang 22 emits:

```llvm
define i64 @rdtsc() {
  %ret = call { i32, i32 } asm sideeffect "rdtsc",
    "={ax},={dx},~{dirflag},~{fpsr},~{flags}"()
  %lo = extractvalue { i32, i32 } %ret, 0
  %hi = extractvalue { i32, i32 } %ret, 1
  %hi64 = zext i32 %hi to i64
  %lo64 = zext i32 %lo to i64
  %shifted = shl i64 %hi64, 32
  %result = or i64 %shifted, %lo64
  ret i64 %result
}
```

The struct return type `{ i32, i32 }` captures both EAX and EDX outputs. The `={ax}` and `={dx}` constraint notation uses braces to name specific registers rather than a register class letter, overriding the allocator's choice.

---

## 25.9 MSVC-Style Inline Assembly

MSVC provides a different inline assembly syntax: `__asm { ... }` blocks without the GCC constraint language. The assembly block can directly reference C variables by name:

```c
// MSVC __asm — not valid GCC syntax
__asm {
    mov eax, myVar
    add eax, 1
    mov myVar, eax
}
```

**Limitations.** MSVC inline asm is restricted to 32-bit (`/arch:IA32`) code; there is no MSVC inline asm support for 64-bit x86-64 or AArch64 targets. Microsoft's recommendation for 64-bit code is to use compiler intrinsics (`_mm_add_ps`, etc.) or external assembly files (`.asm` files assembled with MASM or NASM).

**Clang's partial support.** Clang implements MSVC-style `__asm` blocks for 32-bit x86 targets when compiling in MSVC compatibility mode (`-fms-compatibility`). The implementation is in `MSAsmStmt` in the Clang AST and is processed by the same `CodeGenFunction::EmitAsmStmt` path as GCC asm, but via `MSAsmStmt`-specific handling. Variable name resolution in the asm body traverses the local scope to find the corresponding stack slot or register allocation.

In LLVM IR, Clang emits MSVC asm using the `inteldialect` keyword, since MASM uses Intel syntax. The constraint string is synthesized by Clang's MSVC asm parser rather than provided explicitly by the programmer; the programmer does not write constraint strings in this model.

Clang's MSVC asm support diverges from true MSVC in several ways: label handling differs, `__asm` blocks in C++ member functions may behave differently, and some MASM directives are not recognized. For production Windows kernel code using inline asm, the standard recommendation is to use separate `.asm` files compiled with MASM/NASM and linked against the C/C++ objects.

---

## 25.10 Module-Level Assembly

Module-level assembly, written in C as `asm("...");` at file scope (not inside a function), appends text verbatim to the output assembly for the entire translation unit. In LLVM IR it appears as `module asm "..."`:

```llvm
module asm ".section .note.GNU-stack,\22\22,@progbits"
module asm ".symver memcpy, memcpy@GLIBC_2.2.5"
```

Multiple `module asm` lines are collected by the LLVM IR parser into `Module::ModuleInlineAsm`, a single string with newline separators. This string is emitted by `AsmPrinter::doFinalization` via the `MCStreamer` path. The `MCStreamer::EmitRawText` method writes the string directly into the `.s` file or the in-memory `MCAsmBackend` when integrated assembly is active.

Common uses of module-level asm:

**Stack non-executability.** The `.note.GNU-stack` section directive tells the GNU linker that this object requires no executable stack. Without it on x86 Linux, the linker may produce a `PT_GNU_STACK` segment with execute permission. Clang emits this automatically when targeting Linux ELF.

**Symbol versioning.** `.symver` directives create symbol aliases with specific GLIBC version tags. This enables shared libraries to bind to a specific version of a system symbol, ensuring ABI compatibility across glibc upgrades.

**Linker script commands.** EFI firmware builds and embedded systems use module asm to inject linker directives like `.global _start` or `.section .text.startup` that must appear in the object file rather than the linker script, so they're position-correct relative to other sections.

**Weak symbol declarations.** `__attribute__((weak))` handles the common case, but occasionally the `module asm` path is needed for aliases that must be declared at the assembly level.

The `MCAsmInfo` class ([`llvm/include/llvm/MC/MCAsmInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/MC/MCAsmInfo.h)) provides target-specific assembly dialect information consumed by the printer when emitting module asm. Targets that use integrated assembly (most modern LLVM targets) never write a `.s` file; instead `MCStreamer` calls into the `MCObjectStreamer`, which directly emits ELF/Mach-O/COFF bytes. Module asm is handled by parsing it with the target's `MCAsmParser` in this path.

---

## 25.11 Lowering Inline Assembly in LLVM

The path from an `InlineAsm`-bearing `call` instruction in IR to machine code involves four distinct stages: the `SelectionDAGBuilder`, constraint resolution, the `ISD::INLINEASM` SelectionDAG node, and the final `MachineInstr`.

### 25.11.1 `SelectionDAGBuilder::visitInlineAsm`

When the SelectionDAG builder encounters a `call` instruction whose callee is an `InlineAsm` value, it dispatches to `visitInlineAsm` at line 9776 of [`SelectionDAGBuilder.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/SelectionDAG/SelectionDAGBuilder.cpp#L9776). The function:

1. Calls `TargetLowering::ParseConstraints(DL, TRI, Call)` to decompose the constraint string into a `TargetLowering::AsmOperandInfoVector`. Each element is a `TargetLowering::AsmOperandInfo`, which extends `InlineAsm::ConstraintInfo` with the resolved `ConstraintCode` (the actual constraint letter chosen after multi-alternative resolution) and the `CallOperandVal` (the IR `Value*` for the operand).

2. Calls `TLI.ComputeConstraintToUse(OpInfo, SDValue)` for each operand to select among multiple alternatives and determine whether to use the register form or the memory form of a `rm` constraint.

3. Calls `TLI.getRegForInlineAsmConstraint(TRI, OpInfo.ConstraintCode, OpInfo.ConstraintVT)` to map the constraint code to a `TargetRegisterClass*` and, for fixed-register constraints like `={ax}`, to a specific physical register.

4. Builds the `AsmNodeOperands` list: a flat vector of SDValues preceded by `InlineAsm::Flag` immediates that describe what follows. The structure is:
   - Fixed operands: chain, asm string, MD node (source location), extra-info flags.
   - For each output, input, and clobber: a `Flag` immediate (`MVT::i32`) followed by the register SDValues.

5. Creates either an `ISD::INLINEASM` (for `call`) or `ISD::INLINEASM_BR` (for `callbr`) SelectionDAG node with the assembled operand list.

The `ISD::INLINEASM` opcode is defined at line 1137 of [`ISDOpcodes.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/ISDOpcodes.h#L1137):

```cpp
/// INLINEASM - Represents an inline asm block.  This node always has two
/// return values: a chain and a glue result.
INLINEASM,

/// INLINEASM_BR - Branching version of inline asm. Used by asm-goto.
INLINEASM_BR,
```

### 25.11.2 From SelectionDAG to MachineInstr

The `INLINEASM` SelectionDAG node is not a target-specific node; it is handled by the `TargetInstrInfo::getInlineAsmLength` mechanism and ultimately lowered by `InstrEmitter::EmitMachineNode`. The resulting `MachineInstr` has opcode `TargetOpcode::INLINEASM` or `TargetOpcode::INLINEASM_BR` and carries:

- `MIOp_AsmString` (operand index 0): the assembly string as a `MachineOperand` of kind `MO_ExternalSymbol`.
- `MIOp_ExtraInfo` (operand index 1): the packed flags immediate encoding `HasSideEffects`, `IsAlignStack`, `AsmDialect`, `MayLoad`, `MayStore`, `IsConvergent`, and `MayUnwind`.
- `MIOp_FirstOperand` (operand index 2) onward: alternating `Flag` immediates and their corresponding `MachineOperand` values (physical or virtual registers, immediates, frame index references).

The register allocator's `ProcessImplicitDefs` and `VirtRegRewriter` passes treat the `INLINEASM` MachineInstr as a black box whose register inputs and outputs are pre-fixed by the constraint resolution. Clobbers become implicit `def`s of dead physical registers. The `RAGreedy` allocator must not allocate live-range values into clobbered registers across the asm instruction.

### 25.11.3 `InlineAsmMemConstraint` and Indirect Operands

Memory constraints (`=m`, `m`, `*`) cause Clang to pass the address of the operand rather than its value. In the IR, the pointer argument has an `elementtype` attribute naming the pointed-to type. The `SelectionDAGBuilder` recognizes `isIndirect` in the `ConstraintInfo` and emits a load (for input indirect) or store (for output indirect) around the asm node. The asm node itself receives the address, and the `Flag` encoding uses `Kind::Mem` with the appropriate `ConstraintCode` enum value from `InlineAsm::ConstraintCode`.

---

## 25.12 Verification and Diagnostics

The LLVM IR verifier checks inline asm call sites in `Verifier::verifyInlineAsmCall` at line 2501 of [`Verifier.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/IR/Verifier.cpp#L2501). The checks it performs:

- **Indirect operands must be pointers.** A constraint with `isIndirect = true` requires the corresponding call argument to be a pointer type.
- **Indirect operands must have `elementtype` attribute.** Without this attribute the verifier cannot determine the pointed-to type for correct IR semantics.
- **Non-indirect operands must not have `elementtype`.** Having `elementtype` on a non-indirect operand is a malformed constraint.
- **Label count must match `callbr` indirect destination count.** The number of `isLabel` constraints in the string must equal the number of indirect branch targets in the `callbr` instruction. Label constraints require `callbr`; they are illegal on plain `call`.

The `InlineAsm::verify` static method (line 273 of [`InlineAsm.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/IR/InlineAsm.cpp#L273)) checks the structural consistency of the constraint string against the `FunctionType`:

- The return type must be void for zero outputs, a scalar for one output, or a struct with N elements for N outputs.
- The number of input constraints must match the number of function parameters.
- Output constraints must precede input constraints in the string.
- Input constraints must precede clobber constraints.
- Clobbers must not appear before labels when `!i` constraints are present.

Common errors caught at this level:

```
; Wrong operand count
%1 = call i32 asm "movl %1, %0", "=r"()
; ERROR: number of input constraints does not match number of parameters

; Output constraint after input
%2 = call i32 asm "...", "r,=r"(i32 %x)
; ERROR: output constraint occurs after input constraint

; Label outside callbr
call void asm "jz %l[label]", "!i"()
; ERROR: Label constraints can only be used with callbr
```

Machine-level verification via `llc -verify-machineinstrs` catches a separate class of errors that only manifest after register allocation: an inline asm input assigned to a register that the constraint letter does not permit (e.g., `"a"` constraint assigned to `rcx`), an early-clobber output sharing a physical register with an input despite `&`, or a register clobber that the allocator failed to honor by keeping a live value in that register across the asm.

---

## 25.13 Interaction with the Optimizer

### 25.13.1 Memory Effects of Inline Asm

The optimizer's alias analysis and memory-effect models treat inline asm conservatively. A `sideeffect` inline asm node is treated as a potential read and write of all memory (equivalent to `memory(readwrite)`) unless the constraint string contains only register and immediate operands with no memory clobber. In that case the optimizer may assign `memory(none)` — visible in the generated IR as `attributes #2 = { nounwind memory(none) }` on call sites where Clang detects a pure asm.

A `~{memory}` clobber forces the optimizer to assume the asm reads and writes arbitrary memory, even if `sideeffect` is absent. This is the reason `barrier()` works: the empty-body asm with `~{memory}` and `volatile` forces all pending stores to be committed to memory before the asm and invalidates all cached loads after it.

### 25.13.2 Ordering Relative to Other Operations

Within the IR, a `sideeffect` inline asm that also has `~{memory}` acts as a full compiler-level fence. The optimizer will not move loads or stores past it. This is implemented in `MemorySSA` and `AliasAnalysis`: the asm is treated as a clobber of all memory locations. No `!llvm.mem.parallel_loop_access` metadata applies across such a block.

Without `~{memory}`, a sideeffect asm with only register operands is ordered relative to other side effects (function calls, stores, etc.) but does not affect the movement of pure memory operations. This is the correct model for an asm that executes a hardware instruction with no memory interaction, such as `CPUID` used only for serialization.

### 25.13.3 Non-Volatile Asm and the Optimizer

Non-volatile inline asm (without `sideeffect`) participates in standard optimization. GVN will CSE two identical asm blocks with the same inputs. DCE will remove asm blocks whose outputs are unused. The loop optimizer will hoist loop-invariant asm out of inner loops. This is correct for asm that computes a pure function — for example, a bitfield extraction using `BEXTR` on architectures where the intrinsic is unavailable — but is almost never what you want in practice. The absence of `volatile` should be an intentional choice, not an oversight.

---

## 25.14 Portable and Safe Patterns

### 25.14.1 Memory Barriers

The portable way to emit a full hardware fence is via `__sync_synchronize()`, which LLVM lowers to the appropriate fence instruction for the target (`mfence` on x86, `dmb ish` on AArch64, `fence iorw` on RISC-V). This is preferable to inline asm because it participates in LLVM's memory model reasoning:

```c
// Preferred: uses llvm.fence intrinsic, optimizer-aware
__sync_synchronize();

// Alternative: inline asm fence, opaque to optimizer
__asm__ volatile ("mfence" ::: "memory");
```

For a compiler-only barrier (no hardware fence instruction), the empty asm with `~{memory}` remains the standard pattern.

### 25.14.2 `READ_ONCE` and `WRITE_ONCE`

The Linux kernel defines `READ_ONCE(x)` and `WRITE_ONCE(x, val)` to prevent the compiler from synthesizing multiple accesses from a single C expression. The implementation uses `volatile` casts in modern kernels, but the original pattern used inline asm:

```c
// Kernel pattern to force a single load from memory
#define READ_ONCE(x) \
    ({ union { typeof(x) __val; char __c[1]; } __u; \
       __asm__ volatile ("" : "=m"(__u.__c) : "m"(x)); \
       __u.__val; })
```

The `"=m"` constraint forces the load to memory; `"m"` as input ensures the address is computed correctly. In practice, the `volatile` cast approach is now standard in the kernel, as it is better supported by compiler analysis.

### 25.14.3 The CPUID Idiom

CPUID is used both for feature detection and as a speculation barrier (it serializes the pipeline). The standard idiom:

```c
void cpuid(uint32_t leaf,
           uint32_t *eax, uint32_t *ebx,
           uint32_t *ecx, uint32_t *edx) {
    __asm__ volatile (
        "cpuid"
        : "=a"(*eax), "=b"(*ebx), "=c"(*ecx), "=d"(*edx)
        : "a"(leaf)
    );
}
```

This generates:

```llvm
; Verified with llvm-as 22
define void @cpuid(i32 %leaf, ptr %eax, ptr %ebx, ptr %ecx, ptr %edx) {
  %ret = call { i32, i32, i32, i32 } asm sideeffect "cpuid",
    "={ax},={bx},={cx},={dx},{ax},~{dirflag},~{fpsr},~{flags}"(i32 %leaf)
  %r0 = extractvalue { i32, i32, i32, i32 } %ret, 0
  %r1 = extractvalue { i32, i32, i32, i32 } %ret, 1
  %r2 = extractvalue { i32, i32, i32, i32 } %ret, 2
  %r3 = extractvalue { i32, i32, i32, i32 } %ret, 3
  store i32 %r0, ptr %eax
  store i32 %r1, ptr %ebx
  store i32 %r2, ptr %ecx
  store i32 %r3, ptr %edx
  ret void
}
```

The struct return type captures all four output registers. When used purely as a serialization barrier (discarding the results), the outputs still need to be listed; omitting them and using clobbers instead would work but prevents the compiler from knowing that EAX/EBX/ECX/EDX are defined after the instruction.

### 25.14.4 Stack Canary Access

The GCC stack protector uses the FS segment register (on Linux x86-64) to read the canary value:

```c
uint64_t get_stack_canary(void) {
    uint64_t canary;
    __asm__ volatile ("movq %%fs:40, %0" : "=r"(canary));
    return canary;
}
```

Clang emits:

```llvm
define i64 @get_stack_canary() {
  %canary = call i64 asm sideeffect "movq %fs:40, $0",
    "=r,~{dirflag},~{fpsr},~{flags}"()
  ret i64 %canary
}
```

Note that the `%%fs` in the C source becomes `%fs` in the asm string after the `%%` escape is resolved. The FS base address at offset 40 is where the kernel stores the thread's stack canary under the `pthread` thread-local storage layout on Linux x86-64.

### 25.14.5 Returning Multiple Values via Tied Operands

For instructions that write multiple independent registers, use a struct return type. For instructions that both read and write the same register (read-modify-write), use tied operands via the `+r` constraint or explicit digit binding:

```c
int32_t atomic_xchg(int32_t *ptr, int32_t newval) {
    int32_t old;
    __asm__ volatile (
        "xchgl %0, %1"
        : "=r"(old), "+m"(*ptr)
        : "0"(newval)
        : "memory"
    );
    return old;
}
```

The `"0"` constraint ties the second input to output operand 0: `old` and `newval` start in the same register, the `xchgl` atomically swaps that register with `*ptr`, and the register holds the old value of `*ptr` on exit. The `memory` clobber ensures the memory write to `*ptr` is not reordered.

This generates the IR shown earlier in section 25.2:

```llvm
define i32 @atomic_xchg(ptr %ptr, i32 %newval) {
  %ret = call i32 asm sideeffect "xchgl $0, $1",
    "=r,=*m,0,*m,~{memory},~{dirflag},~{fpsr},~{flags}"(
        ptr elementtype(i32) %ptr, i32 %newval,
        ptr elementtype(i32) %ptr)
  ret i32 %ret
}
```

The `=*m` output constraint passes `ptr` as an indirect output (the memory write), and `*m` as an input (the memory read for the swap source). The `elementtype(i32)` attribute on both pointer arguments is required by the verifier and tells the backend the element type for memory operand sizing.

### 25.14.6 Alternatives to Inline Assembly

Before reaching for inline asm, consider whether an LLVM intrinsic or a compiler builtin covers the case. Chapter 24 covers the full intrinsic vocabulary. For common patterns:

- Arithmetic-with-overflow: `llvm.uadd.with.overflow`, `llvm.sadd.with.overflow` — no need for `ADDO` sequences in asm.
- Bit manipulation: `llvm.ctlz`, `llvm.cttz`, `llvm.bswap`, `llvm.bitreverse` — compile to `BSR`/`BSF`/`LZCNT`/`TZCNT`/`BSWAP` automatically.
- Fences: `llvm.fence` (via `atomic_thread_fence` in C11/C++11) — participates in the memory model and generates the correct fence for the target.
- SIMD: `llvm.x86.avx2.*`, `llvm.aarch64.neon.*` — target-specific intrinsics that are type-checked and produce well-typed IR.
- Prefetch: `llvm.prefetch` — portable across targets.

Inline assembly is the tool of last resort: powerful, portable across compilers that implement the GCC constraint language (Clang, GCC, ICC), but opaque to analysis, fragile under ISA evolution, and a source of subtle bugs when constraints are mis-specified. When an intrinsic exists, prefer it.

---

## Chapter Summary

- The `InlineAsm` class (`llvm/IR/InlineAsm.h`) holds the asm string, constraint string, and the `hasSideEffects`, `isAlignStack`, dialect, and `canThrow` flags; it is created via `InlineAsm::get()` and used as the callee of a `call` or `callbr` instruction.
- AT&T dialect uses source-before-destination operand order, `%` register prefixes, `$` immediate prefix, and size suffixes (`b`/`w`/`l`/`q`); Intel dialect (enabled via `-masm=intel` or the `inteldialect` IR keyword) uses destination-first and bare register names.
- Output constraints begin with `=` (write-only) or `+` (read-write); `&` marks early-clobber outputs. Register class letters (`r`, `m`, `a`, `b`, `c`, `d`, `x`, `w`, `f`) are target-specific and map to register classes via `TargetLowering::getRegForInlineAsmConstraint`.
- The `~{memory}` clobber is a compiler fence, not a hardware fence: it prevents the optimizer from moving memory operations across the asm but does not emit an instruction. `~{cc}` declares modification of the flags register.
- `sideeffect` (`volatile` in C) prevents the optimizer from removing, hoisting, or CSE-ing the asm block. Non-volatile asm may be optimized like a pure function.
- `asm goto` lowers to `callbr` in IR with label operands encoded as `!i` constraints; label destinations become indirect successors of the `callbr`.
- Module-level asm (`module asm "..."`) is collected into `Module::ModuleInlineAsm` and emitted verbatim by `AsmPrinter::doFinalization` through the `MCStreamer` interface.
- `SelectionDAGBuilder::visitInlineAsm` resolves constraints via `TargetLowering::ParseConstraints` and `ComputeConstraintToUse`, then builds an `ISD::INLINEASM` or `ISD::INLINEASM_BR` node with packed `InlineAsm::Flag` operand descriptors.
- The IR verifier (`Verifier::verifyInlineAsmCall`) checks indirect operand pointer types and `elementtype` attributes; `InlineAsm::verify` checks structural consistency of the constraint string against the function type.
- Prefer `llvm.fence`, arithmetic-with-overflow intrinsics, and SIMD intrinsics over inline asm wherever they exist; reserve inline asm for privileged instructions, hardware-specific sequences, and operations with no intrinsic equivalent.

---

*Cross-references:*
- [Chapter 19 — Instructions I: Arithmetic and Memory](ch19-instructions-arithmetic-and-memory.md) — load, store, and atomic instructions that inline asm bypasses.
- [Chapter 23 — Attributes, Calling Conventions, and the ABI](ch23-attributes-calling-conventions-abi.md) — `naked` function attribute, calling convention effects on register availability.
- [Chapter 24 — Intrinsics](ch24-intrinsics.md) — the preferred alternative to inline asm for most hardware operations.
- [Chapter 87 — Inline Assembly Lowering in the Backend](../part-14-backend/ch87-inline-asm-lowering.md) — `SelectionDAGBuilder::visitInlineAsm` in depth, register class constraint resolution, MachineInstr encoding.

*Reference links:*
- [`llvm/include/llvm/IR/InlineAsm.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/InlineAsm.h) — `InlineAsm` class definition, `Flag` encoding, `ConstraintCode` enum.
- [`llvm/lib/IR/InlineAsm.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/IR/InlineAsm.cpp) — `InlineAsm::verify`, `ParseConstraints` implementation.
- [`llvm/lib/IR/Verifier.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/IR/Verifier.cpp#L2501) — `verifyInlineAsmCall` at line 2501.
- [`llvm/lib/CodeGen/SelectionDAG/SelectionDAGBuilder.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/SelectionDAG/SelectionDAGBuilder.cpp#L9776) — `visitInlineAsm` at line 9776.
- [`llvm/include/llvm/CodeGen/ISDOpcodes.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/ISDOpcodes.h#L1137) — `ISD::INLINEASM` and `ISD::INLINEASM_BR` opcodes.
- [`llvm/include/llvm/CodeGen/TargetLowering.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/TargetLowering.h#L4955) — `AsmOperandInfo`, `ParseConstraints`, `ComputeConstraintToUse`.
- [`llvm/lib/Target/X86/X86ISelLowering.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/X86/X86ISelLowering.cpp#L58846) — x86 `getRegForInlineAsmConstraint`.
- [`llvm/lib/Target/AArch64/AArch64ISelLowering.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/AArch64/AArch64ISelLowering.cpp#L11665) — AArch64 `getRegForInlineAsmConstraint`.
- [`llvm/lib/Target/RISCV/RISCVISelLowering.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/RISCV/RISCVISelLowering.cpp#L20498) — RISC-V `getRegForInlineAsmConstraint`.
- [`clang/lib/CodeGen/CGStmt.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGStmt.cpp#L2630) — `CodeGenFunction::EmitAsmStmt`, `GCCAsmStmt` and `MSAsmStmt` lowering.
- [`llvm/include/llvm/MC/MCAsmInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/MC/MCAsmInfo.h) — `MCAsmInfo` class for target assembly syntax information.
- [GCC Inline Assembly HOWTO](https://gcc.gnu.org/onlinedocs/gcc/Using-Assembly-Language-with-C.html) — canonical reference for GCC-compatible extended asm syntax.
- [LLVM Language Reference — Inline Assembler Expressions](https://llvm.org/docs/LangRef.html#inline-assembler-expressions) — official IR-level inline asm documentation.
