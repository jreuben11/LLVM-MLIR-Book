# Ch201 — Binary Lifting to LLVM IR

*Part XXIX — Compiler Tooling, Kernel Integration, and Binary Analysis*

Binary lifting is the process of recovering a compiler-quality intermediate representation — specifically LLVM IR — from a compiled binary executable, without access to the original source code. For security engineers, this means running LLVM's entire optimization and analysis infrastructure on binaries from closed vendors, malware samples, or firmware images. For compiler researchers, it opens a path to retargeting proprietary machine code to new architectures, recompiling stripped executables with sanitizers enabled, or checking whether a vendor binary embeds GPL-licensed code. The engineering challenge is substantial: a compiled binary has shed type information, calling conventions are inferred rather than declared, and the control-flow graph is only partially visible from static analysis alone. Yet the payoff — a semantically faithful LLVM module that can be fed to `opt`, `llvm-link`, and the full LLVM backend pipeline — makes lifting one of the most powerful techniques in the binary analysis toolkit.

This chapter walks through the complete lifting pipeline, the major open-source tools that implement it (Remill, McSema, RetDec), the hard unsolved problems that limit precision, and the post-lifting optimization and validation workflow that turns raw lifted IR into something useful.

---

## 1. Motivation and Use Cases

Binary lifting targets several distinct engineering goals, each with different precision requirements.

**Decompilation for human review.** Security analysts reverse-engineer firmware, malware, and closed-source libraries daily. A decompiler that produces C-like source from lifted LLVM IR is easier to read than raw disassembly, and LLVM's alias analysis, loop detection, and scalar evolution passes can reconstruct structure that assembly obscures.

**Symbolic execution and vulnerability discovery.** Tools such as KLEE and S2E operate on LLVM IR. Lifting a binary to LLVM IR makes it possible to apply symbolic execution without source. Every undefined behavior that would trigger a sanitizer becomes a path condition that the symbolic executor can explore. This is the primary use case that drove Remill's development at Trail of Bits.

**Binary hardening by recompilation.** An organization may receive a third-party library in binary form but need to run it with AddressSanitizer or MemorySanitizer instrumentation to audit it before shipping. If the binary can be faithfully lifted to LLVM IR and the semantics of every memory access preserved, the IR can be instrumented with LLVM's sanitizer passes and recompiled. See [Ch110 — User-Space Sanitizers](../part-16-jit-sanitizers/ch110-user-space-sanitizers.md) for the sanitizer instrumentation infrastructure.

**Retargeting to new architectures.** An embedded vendor may have ARM Cortex-M binaries they wish to run on RISC-V after a silicon transition, or a team may want to run x86-64 server binaries on Apple Silicon without a hypervisor. Lifting to IR and recompiling is one approach, though the calling convention and ABI mismatch between host and target require careful handling.

**License compliance auditing.** Open-source code embedded in proprietary binaries often retains characteristic constant patterns, string literals, and algorithmic structures even after compilation and stripping. Lifted IR can be passed through normalized hashing pipelines or symbolic analysis to detect GPL/LGPL code fragments.

**Binary analysis without source.** This is the catch-all: fuzzing closed binaries with libFuzzer, applying taint analysis, checking security properties via model checking, or computing tight complexity bounds — all become tractable when the binary is expressed as LLVM IR.

**The information loss tradeoff.** Lifting from binary recovers most semantic content but loses information that was present in the object file or compilation unit: DWARF debug info (partially recoverable), type names, high-level control-flow constructs (replaced by explicit branch IR), and the `noalias`/`restrict` annotations that the compiler used for aggressive optimization. The lifted IR is semantically equivalent to the binary but opaque to alias analysis in ways the original source was not. Post-lifting optimization (Section 7) partially recovers some of this structure.

---

## 2. The Lifting Pipeline

A binary lifter must solve four sequential subproblems before it can produce LLVM IR. Each stage feeds the next, and errors propagate — a missed basic block in CFG recovery means an unreachable region in the lifted IR.

### 2.1 Disassembly

The first stage converts raw bytes into a sequence of decoded instructions with operands. Two strategies exist:

**Linear sweep** scans the binary image sequentially, decoding each instruction at the current program counter and advancing by the instruction's byte length. It is fast and complete over the image bytes, but fails on data interleaved with code — a common pattern in x86 binaries where jump tables, alignment padding, and constants appear between instruction sequences. Linear sweep interprets these bytes as instructions, producing garbage decode results that corrupt CFG recovery.

**Recursive traversal (recursive descent)** starts from known entry points — the ELF `e_entry`, exported symbol table entries, exception handler tables — and follows control flow edges: direct branches, calls, and returns. It only decodes bytes reachable via control flow, avoiding data bytes. The cost is incompleteness: indirect branches (computed jumps, virtual dispatch, function pointers) halt traversal until their targets are resolved. Most production lifters use recursive traversal with supplemental heuristics to identify additional entry points.

The LLVM project provides the `MCDisassembler` API for instruction decoding, defined in [`llvm/include/llvm/MC/MCDisassembler/MCDisassembler.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/MC/MCDisassembler/MCDisassembler.h). The core method is:

```cpp
DecodeStatus getInstruction(MCInst &Instr, uint64_t &Size,
                             ArrayRef<uint8_t> Bytes, uint64_t Address,
                             raw_ostream &CStream) const;
```

`MCDisassembler` is target-specific; you obtain one via `TheTarget->createMCDisassembler(*STI, Ctx)` where `STI` is an `MCSubtargetInfo`. The LLVM MC layer is covered in depth in [Ch94 — The MC Layer and MIR Test Infrastructure](../part-14-backend/ch94-the-mc-layer-and-mir-test-infrastructure.md).

External disassembly libraries — Capstone (C, multi-architecture, widely used in Python tooling) and libopcodes (binutils) — provide similar functionality with varying architecture coverage. RetDec uses Capstone as its primary disassembler. Binary Ninja and IDA Pro provide commercial disassemblers with more aggressive heuristic coverage.

### 2.2 CFG Recovery

Given a set of decoded instructions, CFG recovery identifies basic block boundaries and the edges between them. A basic block boundary occurs at: any instruction that is a branch target, any branch/call/return instruction, and the entry point. Edges are direct (statically encoded in the instruction) or indirect (computed at runtime).

Direct edges are trivially recovered. Indirect edges require analysis:

- **Value-set analysis (VSA)**: an abstract interpretation over the register file and memory, tracking sets of possible values each register can hold at each program point. When the analysis converges on a singleton value-set for the branch target register, the edge is recovered precisely. VSA is expensive (super-polynomial in general) and implemented with widening in practice.
- **Switch table pattern matching**: most compilers generate switch statements as a load from a jump table: `jmp [rax*8 + table_base]`. Pattern matching on this idiom recovers the table entries by reading the binary image at `table_base`.
- **Concolic execution**: run the binary under a concrete+symbolic executor (e.g., angr), collecting concrete branch targets observed during execution, and add them to the CFG. Provides completeness for paths exercised but not for all paths.

### 2.3 Type Recovery

Raw disassembly gives you register-width operations — `rax` is 64 bits, `eax` is 32 bits — but not C-level types. Type recovery assigns higher-level types to values:

**Calling convention inference** maps argument registers and stack slots to function parameters using architecture-specific ABI rules (System V AMD64 ABI: `rdi, rsi, rdx, rcx, r8, r9` for integer arguments; `xmm0–xmm7` for floating-point). The return value register convention (`rax` for integers, `xmm0` for floats) identifies return types. Mismatch between caller and callee parameter count is a common failure mode.

**Stack frame reconstruction** identifies the local variable layout by tracking how the function adjusts `rsp` and which stack offsets are read before being written (incoming parameters on the stack) versus written before being read (local variables).

**Pointer type recovery** distinguishes pointer-typed values from integer-typed values by observing whether a value is used as a base address for memory loads/stores. Struct layout inference applies when multiple offsets from the same base are accessed — the struct member positions can be recovered even without member names.

DWARF debug information, when present, provides ground truth for all of the above. `DW_AT_type`, `DW_AT_location`, and `DW_AT_frame_base` directly encode types and frame layouts. Stripping debug info is the single most effective way to impede type recovery.

### 2.4 Semantic Lifting

The final stage maps each decoded machine instruction to a sequence of LLVM IR that has the same semantic effect on the abstract machine state. This is the core differentiator between lifting tools: the fidelity and completeness of the instruction-to-IR translation determines whether the lifted module can be used for precise analysis.

The two main approaches are:

1. **Semantics function libraries** (Remill's approach): each instruction family is pre-compiled to an LLVM IR function. Lifting calls that function with the current machine state. The semantics library is written in C++ with full flags modeling.

2. **Direct IR generation** (RetDec's approach): a code generator directly emits LLVM IR instructions for each decoded machine instruction, handling flags and side effects inline.

---

## 3. Remill — Instruction Semantics as LLVM IR

[Remill](https://github.com/lifting-bits/remill) (Trail of Bits) is a library for lifting machine code to LLVM IR. Its central design decision is to represent every machine instruction as a C++ function that manipulates a typed register-file struct and a thread-through `Memory` pointer. This means Remill's instruction semantics are themselves LLVM IR — compiled from C++ and linked into the lifted module — rather than a code generator that emits IR strings.

### 3.1 The State Struct

Remill models the CPU register file as a C struct, compiled to an LLVM IR aggregate type. For x86-64, this is `X86State` defined in the Remill source at `remill/lib/Arch/X86/Runtime/State.h`. The struct contains explicitly sized fields for every architectural register:

```cpp
// Simplified excerpt — actual struct is more elaborate
struct X86State {
  // General-purpose registers
  union { uint64_t rax; uint32_t eax; uint16_t ax; struct { uint8_t al; uint8_t ah; }; };
  union { uint64_t rbx; uint32_t ebx; /* ... */ };
  // ... 15 more GPRs ...

  // Flags — each is an i8 in the LLVM struct
  uint8_t CF;   // carry flag
  uint8_t PF;   // parity flag
  uint8_t AF;   // auxiliary carry flag
  uint8_t ZF;   // zero flag
  uint8_t SF;   // sign flag
  uint8_t OF;   // overflow flag

  // Floating-point / SSE / AVX state
  // ...
};
```

When compiled to LLVM IR, each field becomes a named element in the aggregate type. The lifter generates IR that uses `getelementptr` to address individual register fields within the state struct, `load` to read them, and `store` to write them.

Each flag is modeled as an `i8` (one byte, not one bit). This avoids LLVM's notoriously difficult `i1` bitfield handling and makes flags individually addressable. Remill uses a lazy evaluation pattern for expensive flag computations: for the x86 `PF` (parity flag), which requires counting set bits in the low byte of a result, the computation is deferred until the flag is actually read. If the flag is never read between two writes, the computation is elided.

### 3.2 The Memory Model

Every lifted function receives a `Memory *` as its first argument and returns a `Memory *`. The `Memory` type is an opaque struct — its definition is intentionally incomplete. All memory accesses go through intrinsic-like helper functions:

```cpp
// These are declared but not defined in the semantics headers.
// The lifter inserts calls to these; at analysis time, they can be
// given concrete implementations or left as abstract.
Memory *__remill_write_memory_8(Memory *, addr_t addr, uint8_t val);
Memory *__remill_write_memory_16(Memory *, addr_t addr, uint16_t val);
Memory *__remill_write_memory_32(Memory *, addr_t addr, uint32_t val);
Memory *__remill_write_memory_64(Memory *, addr_t addr, uint64_t val);

uint8_t  __remill_read_memory_8(Memory *, addr_t addr);
uint16_t __remill_read_memory_16(Memory *, addr_t addr);
uint32_t __remill_read_memory_32(Memory *, addr_t addr);
uint64_t __remill_read_memory_64(Memory *, addr_t addr);
```

Threading the `Memory *` through every instruction ensures that memory operations are sequentially consistent in the lifted IR even after LLVM optimization — the memory pointer is a dependency chain that prevents the optimizer from reordering loads and stores across instruction boundaries. This is the correct behavior: the lifted IR must be semantically equivalent to the binary, which executed instructions in order.

### 3.3 Control-Flow Escape Hatches

Because the CFG is not fully known at lift time, Remill uses sentinel functions to represent unknown control-flow targets:

```cpp
// Called when the lifted block reaches a direct function call.
Memory *__remill_function_call(State &, addr_t callee_pc, Memory *);

// Called for indirect branches (computed jumps).
Memory *__remill_jump(State &, addr_t target_pc, Memory *);

// Called when lifting encounters a block whose successors are not yet known.
Memory *__remill_missing_block(State &, addr_t pc, Memory *);

// Called for function returns (indirect branch to return address).
Memory *__remill_function_return(State &, addr_t ret_pc, Memory *);
```

A higher-level lifter (e.g., McSema, or a custom tool built on Remill) resolves these placeholders by replacing `__remill_missing_block` calls with direct branches once CFG recovery has identified all successors.

### 3.4 The Instruction Semantics Library

Remill's semantics for each instruction are implemented as C++ functions in `remill/lib/Arch/X86/Semantics/`. The file `BINARY.cpp` handles binary arithmetic instructions. Each semantics function is templated on the operand types (register vs. immediate vs. memory) and the operand width. After compilation with `clang`, these become LLVM IR bitcode that is linked into the lifted module.

A concrete example: lifting the x86 instruction `add rax, rbx` produces IR that:

1. Loads `rbx` from the `State` struct via `getelementptr` + `load`.
2. Loads `rax` from the `State` struct.
3. Calls the `ADD_64rr` semantics function, passing the state pointer, memory pointer, and the two loaded values.
4. The semantics function computes the sum, stores the result to `rax` in the state struct, computes and stores each flag (`CF`, `OF`, `SF`, `ZF`, `PF`, `AF`), and returns the memory pointer unchanged.

In simplified LLVM IR (with GEP indices elided for readability):

```llvm
; Pseudocode representation of lifted "add rax, rbx"
define ptr @lifted_block_0x401000(ptr %state, ptr %mem) {
entry:
  ; Load rax and rbx from the State struct
  %rax_ptr = getelementptr %X86State, ptr %state, i32 0, i32 0  ; &state.rax
  %rbx_ptr = getelementptr %X86State, ptr %state, i32 0, i32 1  ; &state.rbx
  %rax_val = load i64, ptr %rax_ptr
  %rbx_val = load i64, ptr %rbx_ptr

  ; Compute the ADD result and flags
  %result   = add i64 %rax_val, %rbx_val
  %of_tmp   = call i1 @llvm.sadd.with.overflow.i64(i64 %rax_val, i64 %rbx_val)
  %cf_tmp   = call i1 @llvm.uadd.with.overflow.i64(i64 %rax_val, i64 %rbx_val)

  ; Store result and flags back to State
  store i64 %result, ptr %rax_ptr
  %cf_ptr = getelementptr %X86State, ptr %state, i32 0, i32 16  ; &state.CF
  %of_ptr = getelementptr %X86State, ptr %state, i32 0, i32 21  ; &state.OF
  store i8 (zext i1 %cf_tmp to i8), ptr %cf_ptr
  store i8 (zext i1 %of_tmp to i8), ptr %of_ptr
  ; ... ZF, SF, PF, AF similarly ...

  ; Control flow: fall through to next block
  %next_mem = call ptr @__remill_jump(ptr %state, i64 0x401003, ptr %mem)
  ret ptr %next_mem
}
```

### 3.5 The Lifter Classes

Remill's public API exposes:

- `remill::Arch` — an architecture-specific object that knows how to decode `remill::Instruction` structs and access the semantics module for that architecture.
- `remill::InstructionLifter` — given a single `remill::Instruction`, emits the corresponding LLVM IR into a basic block by calling into the semantics library.
- `remill::TraceLifter` — lifts a contiguous trace of instructions (a path through the CFG) by calling `InstructionLifter` repeatedly and connecting the resulting basic blocks.

A consumer of the Remill API typically: loads the semantics bitcode module for the target architecture, creates an `InstructionLifter`, iterates over decoded instructions from the disassembler, and calls `LiftIntoBlock` for each instruction, accumulating an LLVM function per CFG basic block.

---

## 4. McSema — Binary-Level Lifting

[McSema](https://github.com/lifting-bits/mcsema) (Trail of Bits) is a complete binary lifting tool built on top of Remill. Where Remill provides instruction-level semantics, McSema adds the whole-binary workflow: disassembly via an external disassembler, CFG recovery, data section handling, and the runtime library needed to execute the lifted binary.

### 4.1 Workflow

The McSema pipeline has two stages:

```bash
# Stage 1: Disassembly and CFG extraction
# Uses IDA Pro (or Binary Ninja) as the disassembler backend.
mcsema-disass \
  --disassembler /opt/idapro/idat64 \
  --binary target.elf \
  --os linux \
  --arch amd64 \
  --output cfg.pb \
  --entrypoint main

# Stage 2: IR lifting from the CFG protobuf
mcsema-lift \
  --os linux \
  --arch amd64 \
  --cfg cfg.pb \
  --output lifted.bc

# Stage 3: Compile and link the lifted IR
clang -O2 lifted.bc -o lifted_binary \
  -Wl,--allow-multiple-definition \
  $(mcsema-lift --print-library-path)
```

The CFG protobuf (`cfg.pb`) encodes the recovered control-flow graph in a schema defined in `mcsema/CFG/CFG.proto`. It contains:

- `Module` — the top-level container, carrying the binary name, OS, and architecture.
- `Function` — a lifted function, with a list of `Block` entries and metadata: entry address, name (from symbol table if present), and whether it is externally visible.
- `Block` — a basic block, with the list of `Instruction` entries and successor edges (by address). Indirect branch targets appear as `__mcsema_unknown_block` references.
- `ExternalFunction` — a call target that resolves to a shared library function (e.g., `printf`). McSema generates a stub that restores the System V ABI from the Remill `State` representation and calls the real function.
- `GlobalVariable`, `Segment` — data sections from the binary image, reconstructed as LLVM global variables.

### 4.2 External Function Handling

When the lifted binary calls a libc function, the Remill `State` struct holds arguments in its register fields rather than in physical registers. McSema generates a "call wrapper" that:

1. Extracts argument values from the `State` struct (e.g., `state.rdi` for the first integer argument).
2. Calls the real external function using the native C calling convention.
3. Stores the return value back into `state.rax`.
4. Returns control to the lifted IR.

This bridging is critical for correctness — without it, lifted code that calls `malloc` would pass the size argument in the `State` struct rather than in `rdi`, and `malloc` would receive garbage.

### 4.3 Data Sections and Relocations

Global variables in the binary appear as `Segment` entries in the CFG. McSema reconstructs them as `@llvm.global_ctors`-registered LLVM global variables with the binary image content as the initializer. Relocations (pointer-sized entries in `.data` that point to other symbols) are reconstructed as LLVM constant expressions or `GlobalAlias` entries.

A key limitation: McSema requires a capable disassembler (IDA Pro or Binary Ninja) for the CFG extraction phase. The `mcsema-disass` tool itself is a thin wrapper that invokes the disassembler's scripting API; IDA Pro's IDAPython scripting is the primary supported path. This creates a dependency on commercial tooling for the input stage.

---

## 5. RetDec — A Complete Decompiler

[RetDec](https://github.com/avast/retdec) (Avast) is a fully open-source decompiler that goes further than IR lifting: it produces compilable C output from a binary. Its architecture is a multi-stage pipeline that uses Capstone for disassembly, produces LLVM IR as an intermediate representation, and then applies a high-level language backend (`llvmir2hll`) to generate structured C.

### 5.1 Architecture and Tool Usage

```bash
# Decompile an x86-64 ELF to C
retdec-decompiler \
  --arch x86 \
  --format elf \
  --keep-unreach-funcs \
  target.elf \
  -o output.c

# Outputs:
#   output.c        — decompiled C source
#   output.bc       — LLVM bitcode (the intermediate lifted IR)
#   output.ll       — LLVM IR in text form
#   output.config   — JSON configuration describing the binary
```

The decompiler processes the binary through these stages:

1. **Binary parsing**: LIEF or a custom format parser reads the ELF/PE/Mach-O file, extracting segments, symbol tables, and import tables.
2. **Disassembly**: Capstone decodes instructions. RetDec applies recursive descent with heuristic function boundary detection.
3. **Control-flow reconstruction**: identifies loops, if/else structures, and switch patterns for the high-level backend.
4. **IR generation**: each decoded instruction is translated to LLVM IR. Unlike Remill, RetDec generates IR directly rather than calling pre-compiled semantics functions. The generated IR uses actual LLVM arithmetic instructions for ALU operations and explicit flag computation for comparison semantics.
5. **LLVM optimization**: `mem2reg`, `instcombine`, `simplifycfg`, and GVN are applied to the raw lifted IR to produce cleaner IR.
6. **Type reconstruction**: based on how values are used (added to a pointer → pointer arithmetic; divided → integer or float; passed to `printf` with `%s` format → null-terminated string pointer).
7. **`llvmir2hll` output**: the optimized IR is pattern-matched against high-level C constructs and rendered as readable C.

### 5.2 Type Reconstruction

RetDec's type reconstruction operates in two modes:

**DWARF-based** (when debug info is present): the `.debug_info` section is parsed to extract `DW_TAG_variable`, `DW_TAG_formal_parameter`, `DW_TAG_base_type`, and `DW_TAG_structure_type` entries. These directly provide C-level types for variables and function signatures. RetDec's `llvmir2hll` uses these to annotate the IR before output.

**Heuristic-based** (stripped binaries): RetDec applies a constraint propagation pass. If a value flows into `%rdi` before a call that is known to be `malloc`, then that value must be `size_t`. If a value is loaded at an 8-byte-aligned offset from another value, and the base value was passed to `fopen`, then the 8-byte offset is likely a member of `FILE`. This propagation is unsound — it fails when aliasing is complex or when calling conventions deviate from the standard ABI.

### 5.3 The `llvmir2hll` Backend

`llvmir2hll` is RetDec's LLVM IR → C translator. It is structured as a set of pattern matchers that recognize IR idioms and emit the corresponding C syntax:

- A `br` with a `phi` on the back edge → `while` or `for` loop.
- A chain of `icmp` + `br` checking a single variable against constants → `switch`.
- Alloca + store-before-load → local variable declaration.
- A `call` to an `@llvm.memcpy` intrinsic → `memcpy(dst, src, size)`.

The output is human-readable but not source-equivalent: variable names are typically `v1`, `v2`, ... unless DWARF provides names; goto statements appear when the CFG has irreducible edges; and heavily inlined functions produce large, monolithic C functions.

### 5.4 Limitations

- **Obfuscated code**: virtualization-based obfuscators (Themida, VMProtect) replace native instructions with a custom bytecode interpreter. Capstone decodes the interpreter dispatch loop, but the semantic content is hidden inside the bytecode. Neither RetDec nor Remill can lift through a custom VM without first deobfuscating.
- **Indirect calls through function pointer tables**: without value-set analysis, `call [rax]` produces an `@__remill_function_call` stub in Remill or a `call` to an unresolved pointer in RetDec, breaking the decompiled output at that point.
- **C++ virtual dispatch**: the pattern `mov rax, [rcx]; call [rax+0x18]` (load vptr, call at offset 0x18) is syntactically recognizable but semantically opaque without the class hierarchy. RetDec does not recover C++ class hierarchies; it produces raw pointer arithmetic.

---

## 6. Lifting Challenges

### 6.1 Indirect Branches and Computed Jumps

Indirect branches are the hardest problem in binary lifting. The x86 instruction `jmp rax` is a single byte but requires the lifter to know every value `rax` can hold at that point. The standard approaches each have failure modes:

**Value-set analysis (VSA)** computes abstract values (intervals, strided intervals, or sets) for every register at every program point. It is precise for simple cases (loop induction variables, small switch tables) and diverges to `TOP` (any value possible) for complex cases (heap-allocated pointers, values read from files). When VSA produces `TOP`, the indirect branch adds an edge to every possible target — a conservative over-approximation that causes lifted functions to be much larger than necessary.

**Pattern-based switch table recovery** recognizes the compiler-generated pattern for switch statements: a bounds check followed by a table-indexed load. Most compilers generate a form of:

```asm
cmp  eax, N        ; N = number of cases
ja   .default      ; branch if out of range
lea  rcx, [rip + table]
movsxd rax, dword ptr [rcx + rax*4]
add  rax, rcx
jmp  rax
```

The lifter identifies the load from a PC-relative table, reads the table entries from the binary image, and registers each entry as a CFG successor. This fails for computed switch tables, sparse switches (where the compiler emits a binary search tree instead of a table), or if the table is in a writable data segment that may be modified at runtime.

**Concolic execution** discovers jump targets by actually running the binary under a combined concrete+symbolic executor and observing the targets taken. It provides high coverage for well-exercised code paths but cannot guarantee completeness — a jump target only reached on a specific input will be missed if that input is not in the test suite.

The precision/completeness tradeoff is fundamental: the three approaches above are ordered from complete-but-imprecise (VSA with widening) to precise-but-incomplete (concolic execution). Production lifting systems combine all three, using pattern matching first, VSA for fallback, and concolic execution for residual cases.

### 6.2 Self-Modifying Code

Static lifting assumes the binary image is fixed. Self-modifying code — JIT compilers (V8, SpiderMonkey, LLVM ORC), packers (UPX, ASPack), and copy-protection schemes — writes to executable pages at runtime and then executes the new bytes. A static lifter that snapshots the binary image at load time will lift the pre-modification bytes, producing IR that does not reflect what actually executes.

There is no general static solution. Specialized approaches include: (a) running the binary under a dynamic binary translator (QEMU, DynamoRIO) that lifts code at the point of execution, (b) unpacking the binary first using an emulator, or (c) restricting lifting to post-JIT snapshots. The LLVM ORC JIT (see [Ch108](../part-16-jit-sanitizers/ch108-the-orc-jit.md)) is architecturally transparent to lifting if the JIT-compiled code is captured after emission but before execution.

### 6.3 Calling Convention Inference

The System V AMD64 ABI specifies that integer arguments go in `rdi, rsi, rdx, rcx, r8, r9` and floating-point arguments in `xmm0–xmm7`. A function that is compiled with this ABI and exported via a symbol table is easy to type: the lifter reads the ABI rule and the number of arguments from the CFG's `ExternalFunction` entry.

Internal (non-exported) functions present the challenge. The lifter must infer the number, order, and types of arguments from the calling sites. Heuristics look at which registers are live-in to the function (registers read before being written are incoming arguments) and which registers are live-out at the return (the return value register). This fails for:

- Functions that use a non-standard calling convention (e.g., custom register passing schemes in kernel code or heavily optimized inner loops).
- Functions whose argument registers are partially set by the caller and partially inherited from caller state (tail call optimization creates this pattern).
- Variadic functions where the argument count varies per call site.

When calling convention inference fails, the lifted function has the wrong signature, and calls to it produce IR with incorrect operand counts — a silent semantic error rather than a compile-time type error.

### 6.4 `noalias` and Pointer Provenance in Lifted IR

LLVM's alias analysis (AA) infrastructure is designed to take advantage of source-level information: `restrict` qualifiers, `noalias` function attributes, and the pointer provenance model defined by the LLVM Language Reference. See [Ch16 — IR Structure](../part-04-llvm-ir/ch16-ir-structure.md) for the IR-level semantics of these attributes.

Lifted IR violates the assumptions underlying AA in two ways:

1. **The abstract `Memory *` pointer** threads through every memory operation via the `__remill_read/write_memory_N` intrinsics. From LLVM's perspective, every memory operation goes through a single pointer, which means AA must conservatively assume that every load may alias every store — there is no provenance separation between stack variables, heap objects, and globals.

2. **All registers collapse to `getelementptr` into the `State` struct.** The `State` struct holds all registers, so any IR that touches a register field can theoretically alias any other IR that touches any other register field (in the absence of struct TBAA metadata).

The consequence is that most LLVM optimization passes that rely on AA — LICM, GVN, MemCpyOpt — see heavily conservative alias sets in lifted IR and produce fewer transformations than they would on source-compiled IR. Post-lifting optimization (Section 7) partially addresses this, but the structural constraint is fundamental.

### 6.5 Exception Handler Discovery

Windows structured exception handling (SEH) stores handler tables in `.pdata` and `.xdata` sections in PE format. On Linux, DWARF `.eh_frame` encodes the unwind tables needed by C++ exception handling. When a lifter reconstructs functions from binary code, it must also reconstruct these tables to preserve exception handling semantics.

The DWARF `.eh_frame` section encodes Call Frame Information (CFI) directives — `DW_CFA_def_cfa`, `DW_CFA_offset`, etc. — that describe how to unwind the stack at each program point. Remill and McSema do not currently reconstruct `.eh_frame` in the lifted IR; the lifted binary has no unwind tables, meaning C++ exceptions thrown from lifted code will not unwind through lifted frames correctly. This is a known gap in the current lifting infrastructure.

For binaries that use SJLJ (setjmp/longjmp-based) exception handling, the situation is better: `setjmp` and `longjmp` are external library calls that McSema can represent as `ExternalFunction` stubs, preserving their semantics through the abstraction layer.

### 6.6 C++ Virtual Dispatch Recovery

The x86-64 vtable lookup pattern — `mov rax, [rcx]; call [rax + offset]` — is recognized as a virtual call by its structure, but the target class and method cannot be determined without type recovery. If RTTI (Run-Time Type Information) is present and not stripped, the type name strings embedded in the vtable's RTTI block can be used to identify the class. The vtable layout (array of function pointers) can then be parsed to recover the method address for each slot.

For stripped C++ binaries, vtable identification is a heuristic: a read-only data region containing function pointer values that align to the `.text` section is likely a vtable. Binary analysis frameworks such as Binary Ninja and Ghidra have dedicated vtable recognition passes. Recovering the full class hierarchy, however, remains an open research problem.

---

## 7. Post-Lifting Optimization

Raw lifted IR is verbose and conservative. The Remill state-threading approach produces a function per basic block, each returning a `Memory *` and taking a `State *` — the SSA form is not exploited at all, because every register access goes through memory (the `State` struct). A standard LLVM optimization pipeline can recover much of the structure:

```bash
# Apply a post-lifting optimization pipeline to a McSema output module
opt -passes='mem2reg,instcombine,simplifycfg,gvn,adce' \
    lifted.bc -o opt.bc

# Inspect the result
llvm-dis opt.bc -o opt.ll
```

The passes address specific structural problems in lifted IR:

**`mem2reg`** promotes `alloca` slots and, when register-field GEPs are expressed as `alloca`s rather than struct GEPs, converts them to SSA values. If the lifter emits each used register as a distinct `alloca` (a common optimization in some lifting implementations), `mem2reg` converts these to direct value definitions, enabling subsequent SSA-based passes to see the register values directly rather than going through memory.

**`simplifycfg`** collapses the per-instruction-block structure that some lifters emit. A Remill-based lifter may emit one LLVM basic block per machine basic block, with explicit `br` terminators connecting them in the CFG-dispatch pattern. `simplifycfg` merges blocks where the only predecessor has a single successor, collapsing the dispatch structure into a simpler CFG.

**`instcombine`** folds peephole patterns: constant-offset GEPs into constants, zero-extended comparisons into their natural width, and redundant casts that the semantics library introduces to match C type promotion rules.

**`gvn`** (Global Value Numbering) identifies common subexpressions. In lifted IR, the same register is often loaded multiple times within a basic block because each instruction's semantics function loads and stores independently. GVN eliminates the redundant loads once `mem2reg` has converted them to SSA values.

**`adce`** (Aggressive Dead Code Elimination) removes flag computations for flags that are never read. If a block computes `SF`, `PF`, and `AF` from an `add` instruction but no subsequent instruction reads those flags before they are overwritten, `adce` removes the dead computations.

After this pipeline, a simple lifted function that computes an arithmetic expression over local variables may be reduced from hundreds of IR instructions to a handful, closely resembling what Clang would have produced from the source.

A key limitation remains: because lifted IR threads memory through an abstract `Memory *`, the AA infrastructure cannot be exploited by LICM (Loop Invariant Code Motion) or MemCpyOpt. Loads from the abstract memory may alias any store, so loop-invariant memory reads cannot be hoisted without special analysis. Solving this requires either a concrete memory model (replacing `Memory *` with real `alloca` + LLVM memory) or annotating the memory intrinsics with `noalias` attributes under a sufficiently restricted memory model.

---

## 8. Alive2 for Translation Validation

A central question in binary lifting is: how do you know the lifted IR is correct? The semantics of `add rax, rbx` may seem obvious, but 16-bit prefix handling, flag interactions, and memory operand addressing modes create hundreds of instruction variants, each requiring a faithful IR translation.

**Translation validation** answers this by checking semantic equivalence between two representations: the original source (or binary) and the lifted IR. Alive2, the LLVM IR refinement checker, can be applied at the source-to-IR boundary. See [Ch170 — Alive2 and Translation Validation](../part-24-verified-compilation/ch170-alive2-and-translation-validation.md) for Alive2's architecture.

For lifting validation, the approach is:

1. **If source is available**: compile the function with Clang to produce a reference IR module. Lift the corresponding compiled binary to produce a lifted IR module. Apply `alive-tv` to check whether the lifted function refines the reference function — that is, whether any input that produces a defined result in the reference also produces the same result in the lifted version.

2. **Without source**: construct a formal model of the target ISA instruction semantics (e.g., from the Intel SDM or from Sail's formal x86 spec) and check the lifting's IR translation against the formal model for each instruction variant independently. This is the approach Remill's developers use for testing the semantics library: each instruction's C++ semantics function is compiled and checked against a formal specification using bounded model checking.

The practical limitation of applying `alive-tv` to full functions is scalability: Alive2 uses an SMT solver (Z3) internally, and verifying equivalence of functions with unbounded loops requires loop invariants that must be supplied manually. Translation validation is most productive when applied per-instruction (checking one semantics function at a time) rather than at the whole-function level.

---

## 9. Related Tools and IRs

Binary lifting to LLVM IR is one point in a design space populated by several other tools that make different tradeoffs.

### 9.1 B2R2

[B2R2](https://github.com/B2R2-org/B2R2) is a binary analysis framework written in F# that produces a RISC-like IR (called B2R2 IL or BIL) from binary code and can optionally emit LLVM IR for interfacing with the LLVM optimization pipeline. B2R2 targets the .NET ecosystem and provides higher-level analysis primitives (def-use chains, slice computation) out of the box. Its LLVM IR output is less mature than Remill's but the framework's F# type safety provides strong guarantees against common implementation bugs.

### 9.2 Ghidra and P-Code

[Ghidra](https://ghidra-sre.org) (NSA) uses a custom IR called P-Code, defined by the SLEIGH ISA specification language. SLEIGH specs describe instruction encodings and their P-Code semantics in a single file per architecture; there are SLEIGH specs for x86, ARM, PowerPC, MIPS, RISC-V, and dozens of embedded architectures. P-Code is a RISC-like, typed IL with explicit operations for memory reads/writes, flag computation, and control flow. It is SSA-convertible (Ghidra includes a decompiler that applies SSA transformation before generating C output). P-Code's design point differs from LLVM IR in that it prioritizes architecture-independence and decompiler integration over optimization infrastructure compatibility. There is active work on P-Code → LLVM IR bridges, but they are not yet part of the mainline Ghidra distribution.

### 9.3 angr and VEX IR

[angr](https://angr.io) is a Python binary analysis framework widely used in CTF competitions and security research. It uses VEX IR — the intermediate representation developed by Valgrind — as its internal representation. VEX IR is a statement-based IR with explicit temporaries, typed memory reads/writes, and exit statements for control flow. angr's `pyvex` layer lifts x86, ARM, MIPS, and other architectures to VEX IR using the `libvex` library. Unlike Remill, VEX IR has no connection to LLVM: it cannot be passed to `opt` or benefit from LLVM's optimization passes. angr compensates with its own analysis library (`claripy` for constraint solving, `ailment` for its higher-level AIL representation). For security researchers who need a quick Python-scriptable analysis without the C++ overhead of Remill, angr is the most accessible entry point.

### 9.4 Binary Ninja MLIL/HLIL

[Binary Ninja](https://binary.ninja) (Vector35) is a commercial reverse engineering platform that defines a multi-level IL stack: LLIL (Low-Level IL, close to machine code), MLIL (Medium-Level IL, with type recovery), and HLIL (High-Level IL, with structured control flow). Binary Ninja's ILs are not LLVM IR, but Binary Ninja provides a Python API that can be used to drive Remill or McSema for LLVM IR export. The commercial MLIL/HLIL layer provides better automatic type recovery and C++ virtual dispatch analysis than open-source alternatives, making it attractive as a frontend for Remill-based lifting pipelines that require higher precision.

### 9.5 Summary Comparison

| Tool        | Language   | IR           | Arch coverage          | Open source | LLVM integration  |
|-------------|------------|--------------|------------------------|-------------|-------------------|
| Remill      | C++        | LLVM IR      | x86, AArch64, SPARC    | Yes (Apache) | Native            |
| McSema      | C++ / Python | LLVM IR    | x86, x86-64, AArch64  | Yes (Apache) | Native (via Remill) |
| RetDec      | C++        | LLVM IR → C  | x86, ARM, MIPS, PPC   | Yes (MIT)   | Native (LLVM passes) |
| B2R2        | F#         | BIL / LLVM IR | x86, x86-64, ARM      | Yes (MIT)   | Optional          |
| Ghidra      | Java       | P-Code       | 30+ architectures      | Yes (Apache) | Via bridges       |
| angr/VEX    | Python     | VEX IR       | x86, ARM, MIPS, PPC   | Yes (BSD)   | None              |
| Binary Ninja | Python API | LLIL/MLIL/HLIL | 15+ architectures    | No (commercial) | Via API        |

---

## Chapter Summary

- **Binary lifting** translates compiled machine code to LLVM IR, enabling the full LLVM optimization and analysis infrastructure to operate on binaries without source access. Use cases include decompilation, vulnerability discovery via symbolic execution, binary hardening with sanitizers, architecture retargeting, and license compliance.

- **The lifting pipeline** has four stages: disassembly (linear sweep vs. recursive traversal using `MCDisassembler` or Capstone), CFG recovery (direct edges plus indirect branch resolution via VSA, pattern matching, or concolic execution), type recovery (calling convention inference, stack frame reconstruction, DWARF parsing), and semantic lifting (mapping instructions to IR).

- **Remill** models each instruction as a C++ semantics function compiled to LLVM IR bitcode. The `State` struct holds the register file; each flag is an `i8` field. An abstract `Memory *` threads through all memory operations via `__remill_read/write_memory_N` intrinsics. Control-flow unknowns are represented as calls to `__remill_jump`, `__remill_function_call`, and `__remill_missing_block`.

- **McSema** extends Remill to whole-binary lifting: `mcsema-disass` extracts a CFG protobuf via IDA Pro or Binary Ninja; `mcsema-lift` converts the protobuf to an LLVM bitcode module. External function calls are handled via ABI-bridging stubs.

- **RetDec** is a complete decompiler: Capstone disassembly → LLVM IR → LLVM optimization passes → `llvmir2hll` → C output. Type reconstruction uses DWARF when present and heuristic constraint propagation for stripped binaries.

- **Core lifting challenges**: indirect branches require VSA or concolic execution; self-modifying code defeats static lifting entirely; calling convention inference fails on non-ABI-conforming code; lifted IR's abstract `Memory *` prevents effective alias analysis; exception handler tables are not reconstructed in current tools; C++ virtual dispatch recovery requires RTTI or vtable identification heuristics.

- **Post-lifting optimization** with `mem2reg`, `simplifycfg`, `instcombine`, `gvn`, and `adce` converts verbose state-threading IR into something closer to source-compiled IR, but alias analysis remains limited by the abstract memory model.

- **Translation validation** via Alive2 can verify per-instruction semantics correctness. Full function equivalence checking is expensive but tractable for individual instruction semantic functions checked against formal ISA specifications.

- **Alternative tools** cover the design space: Ghidra P-Code prioritizes architecture coverage over LLVM integration; angr/VEX IR provides a Python-scriptable security research workflow; Binary Ninja MLIL/HLIL offers the best commercial type recovery and serves as a high-quality frontend for Remill-based pipelines.
