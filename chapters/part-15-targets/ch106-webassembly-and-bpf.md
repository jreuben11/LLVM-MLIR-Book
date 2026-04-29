# Chapter 106 — WebAssembly and BPF

*Part XV — Targets*

Two of the most significant non-traditional targets in the LLVM ecosystem share a common design philosophy despite serving radically different domains: both WebAssembly and eBPF are virtual instruction set architectures defined by safety and portability rather than hardware fidelity. WebAssembly provides a sand-boxed, portable execution environment for web browsers and server-side runtimes; eBPF provides a bounded, verifier-checked execution environment inside the Linux kernel for networking, tracing, and security policy. Both make different trade-offs than a CPU target: structured control flow requirements in Wasm, strict loop-bound and stack requirements in BPF. Understanding how LLVM handles these constraints — including the Wasm CFG stackification pass and BPF's CO-RE (Compile Once, Run Everywhere) relocation model — reveals important principles about compiler targets defined by verifier semantics rather than hardware capability.

---

## 106.1 WebAssembly Target Overview

### 106.1.1 The WebAssembly Execution Model

WebAssembly (Wasm) is a binary instruction format designed as a portable compilation target for high-performance web applications. Its execution model differs fundamentally from native CPU targets:

**Stack machine**: Wasm is a stack-based virtual machine. Instructions consume values from and produce values onto an implicit value stack. There are no general-purpose registers visible to the programmer; LLVM's backend must translate its register-based IR into stack form.

**Linear memory**: Each Wasm module has a contiguous linear memory (typed `memN`) addressed by i32 (or i64 in Wasm64) offsets. This is where C heap allocations, global variables, and stack frames reside. There is no pointer type in Wasm's type system — memory accesses use integer offsets.

**Structured control flow**: Wasm does not have arbitrary goto. Control flow is expressed through structured constructs (`block`, `loop`, `if`/`else`) with labeled break (`br`) relative to the enclosing block depth. This is a hard constraint from the Wasm security model — it ensures single-pass validation.

**Modules**: A Wasm module contains function definitions, a table (for indirect calls via `call_indirect`), memories, globals, and import/export declarations. Multiple modules communicate only through explicit imports and exports.

### 106.1.2 Wasm Versions and Proposal Status

| Version | Key Features |
|---------|-------------|
| Wasm 1.0 (MVP, 2019) | Core: i32, i64, f32, f64 types; basic instructions |
| Wasm 2.0 (2022) | SIMD128, bulk memory ops, ref types, multi-value, table instructions |
| Post-2.0 proposals | Threads+atomics, exceptions, tail calls, GC, component model, JSPI |

LLVM tracks these via subtarget features, and Clang exposes them through target attributes.

### 106.1.3 WASI — WebAssembly System Interface

WASI provides a POSIX-like syscall layer for Wasm running outside browsers. The triple `wasm32-unknown-wasi` targets WASI, enabling programs that use `fopen`, `getenv`, `argv`, etc., to compile to Wasm for server-side or CLI deployment with runtimes like Wasmtime, WasmEdge, or WAMR.

---

## 106.2 Wasm Target Machine and Subtarget

### 106.2.1 Target Registration and Triples

The WebAssembly backend lives in [`llvm/lib/Target/WebAssembly/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/WebAssembly/). Supported triples:

| Triple | Description |
|--------|-------------|
| `wasm32-unknown-unknown` | 32-bit Wasm, no OS |
| `wasm64-unknown-unknown` | 64-bit Wasm (memory64 proposal) |
| `wasm32-unknown-wasi` | 32-bit Wasm + WASI syscall layer |
| `wasm32-unknown-emscripten` | 32-bit Wasm with Emscripten JS glue |

The data layout for wasm32: `e-m:e-p:32:32-p10:8:8-p20:8:8-i64:64-n32:64-S128-ni:1:10:20`

### 106.2.2 Subtarget Features

```bash
# Query Wasm subtarget features
/usr/lib/llvm-22/bin/llc -march=wasm32 -mattr=help 2>&1 | head -40
```

Key features:

| Feature | Flag | Description |
|---------|------|-------------|
| SIMD128 | `+simd128` | 128-bit SIMD vector operations |
| Atomics | `+atomics` | Shared memory atomic instructions (threads proposal) |
| Bulk memory | `+bulk-memory` | `memory.copy`, `memory.fill`, `memory.init` |
| Exception handling | `+exception-handling` | `try`/`catch`/`throw` instructions |
| Tail calls | `+tail-call` | `return_call`, `return_call_indirect` |
| Reference types | `+reference-types` | `externref`, `funcref` types; `call_indirect` with new tables |
| Multivalue | `+multivalue` | Functions returning multiple values (used internally) |
| Relaxed SIMD | `+relaxed-simd` | Non-deterministic SIMD for performance |
| Memory64 | `+memory64` | 64-bit linear memory addresses |

### 106.2.3 Target Machine Structure

```cpp
// WebAssemblyTargetMachine.h
class WebAssemblyTargetMachine final : public LLVMTargetMachine {
  mutable StringMap<std::unique_ptr<WebAssemblySubtarget>> SubtargetMap;
public:
  WebAssemblyTargetMachine(const Target &T, const Triple &TT,
                            StringRef CPU, StringRef FS,
                            const TargetOptions &Options,
                            std::optional<Reloc::Model> RM,
                            std::optional<CodeModel::Model> CM,
                            CodeGenOptLevel OL, bool JIT);
  TargetPassConfig *createPassConfig(PassManagerBase &PM) override;
  TargetLoweringObjectFile *getObjFileLowering() const override;
};
```

Source: [`WebAssemblyTargetMachine.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/WebAssembly/WebAssemblyTargetMachine.cpp)

---

## 106.3 WebAssembly Instruction Selection

### 106.3.1 LLVM Virtual Registers to Wasm Stack

LLVM's Wasm backend starts with standard virtual register-based MachineIR (same as every other target) and then transforms it into stack-based form. This is done in two stages:

1. **`WebAssemblyISelDAGToDAG`**: SelectionDAG-based instruction selection producing Wasm MachineInstr nodes in register form
2. **`WebAssemblyRegStackify`**: Converts register-based instructions to stack-based form where possible
3. **`WebAssemblyExplicitLocals`**: Assigns remaining virtual registers to Wasm local variables

### 106.3.2 Wasm Type System Mapping

| LLVM IR Type | Wasm Type | Notes |
|-------------|-----------|-------|
| `i1`, `i8`, `i16`, `i32` | `i32` | Sub-word types extended |
| `i64` | `i64` | |
| `f32` | `f32` | |
| `f64` | `f64` | |
| `<16 x i8>`, `<8 x i16>`, `<4 x i32>`, `<2 x i64>`, `<4 x f32>`, `<2 x f64>` | `v128` | SIMD128 |
| `ptr` (in default AS) | `i32` / `i64` | Linear memory offset |
| `ptr addrspace(10)` | `externref` | Reference type (GC proposal) |
| `ptr addrspace(20)` | `funcref` | Function reference |

### 106.3.3 RegStackify Pass

The [`WebAssemblyRegStackify`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/WebAssembly/WebAssemblyRegStackify.cpp) pass is the core transformation that distinguishes the Wasm backend. It attempts to prove that a value produced by instruction A and consumed immediately by instruction B does not need to be stored in a local variable — it can instead flow through the implicit operand stack.

The algorithm is a reverse scan: for each instruction (processed last-to-first), for each operand that is a virtual register, the pass looks back through the MachineBasicBlock for the single definition of that register. Stackification is safe if:
1. The register has exactly one use (single-use SSA value)
2. The definition and use are in the same basic block
3. No instruction between them has side-effects that could invalidate the stack state
4. No instruction between them has another definition that the use might depend on

```
Before RegStackify:                    After RegStackify:
  MBB:                                 MBB:
    %0 = I32_ADD %a, %b                  IMPLICIT_DEF      ; %a pushed
    LOCAL_SET $t, %0                     IMPLICIT_DEF      ; %b pushed
    LOCAL_GET $t                         I32_ADD            ; pops %b, %a; pushes result
    %2 = I32_MUL ..., %1 (uses $t)       IMPLICIT_DEF      ; operand for mul
                                         I32_MUL            ; pops result, ...; pushes final
```

Concretely, when the pass stackifies instruction `I` (the def) for use in instruction `J` (the use):
- `I` is marked with the `STACKIFY` flag, meaning it implicitly pushes its result
- `J` is marked with the corresponding `STACKIFY` flag on the operand, meaning it implicitly pops
- The explicit `LOCAL_SET`/`LOCAL_GET` pair is removed

The pass also handles the **tree** pattern: if a value is used in a chain where each use is also stackified, the entire expression tree can be made stack-form, avoiding locals entirely for complex arithmetic.

```
LLVM IR:                              Wasm stack form:
  %t1 = fmul float %a, %b             f32.const 3.14
  %pi = sitofp i32 3, float           local.get $a
  %t2 = fadd float %pi, %t1           local.get $b
  store float %t2, float* %p          f32.mul
                                      f32.add
                                      f32.store 0($p)
```

The three intermediate values (`%t1`, `%pi`, `%t2`) all become stack slots, and no locals are needed for any of them.

### 106.3.4 ExplicitLocals Pass

After `RegStackify`, remaining virtual registers that could not be stackified become Wasm **local variables**. The [`WebAssemblyExplicitLocals`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/WebAssembly/WebAssemblyExplicitLocals.cpp) pass inserts explicit `local.get` and `local.set` instructions (previously implicit in register operands) and assigns each virtual register a local index.

---

## 106.4 Structured Control Flow

### 106.4.1 The CFGStackify Pass

The most architecturally distinctive pass in the Wasm backend is [`WebAssemblyCFGStackify`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/WebAssembly/WebAssemblyCFGStackify.cpp). It converts LLVM's unstructured CFG (arbitrary basic blocks with conditional/unconditional branches) into Wasm's structured control flow.

The key insight is that LLVM's loop analysis and dominance tree already encode most of the structure that Wasm requires. The algorithm operates in several phases:

**Phase 1 — Place loop markers**: For every natural loop detected by LLVM's loop analysis, insert `LOOP`/`END_LOOP` pseudo-instructions at the loop header and exit. The `LOOP` instruction in Wasm is the branch target for `br` instructions going backward (continuing the loop).

**Phase 2 — Place block markers for forward branches**: Scan forward branches and multi-successor blocks. For each basic block that is the target of a forward branch, insert a `BLOCK`/`END_BLOCK` pair around the region. The `END_BLOCK` is placed at the branch target; `br N` with depth N reaches the `END_BLOCK` of the Nth enclosing `block`.

**Phase 3 — If/Else conversion**: For conditional branches where both successors are reachable and one is the fallthrough, convert to an `IF`/`ELSE`/`END_IF` structure.

**Phase 4 — Eliminate original branch instructions**: The explicit conditional/unconditional branches are replaced by `BR_IF`/`BR` instructions with block depths computed from the nesting level.

A worked example — a loop with early exit:

```
LLVM CFG (basic blocks):
  entry → header
  header: cond → early_exit (if done), → body (otherwise)
  body → header (back-edge)
  early_exit → end

Wasm structured output:
  block $early_exit_target     ; wraps the loop; br 1 = exit loop
    loop $loop_header          ; br 0 = back-edge to header
      ; header body (compute cond)
      br_if $early_exit_target  ; if done, break out
      ; body body
      br $loop_header           ; continue loop
    end_loop
  end_block                     ; early_exit lands here
  ; end body
```

The CFGStackify pass in LLVM 22 uses a stack-based algorithm: it maintains a list of open `BLOCK`/`LOOP`/`IF` constructs (each with their label) and consults the list to compute the depth for each `br`/`br_if` instruction. Irreducible control flow — CFGs where a loop has multiple entry points — cannot be directly represented in Wasm and requires a transformation (loop rotation or a dispatch variable). The `WebAssemblyFixIrreducibleControlFlow` pass handles this case before CFGStackify runs.

### 106.4.2 Branch Depth Encoding

Wasm branches encode targets as integer depths into the block/loop stack:

```wasm
(func $example (local i32)
  i32.const 0
  local.set 0            ;; i = 0
  block $outer           ;; depth 1 from inside
    loop $inner          ;; depth 0 from inside (innermost)
      local.get 0
      i32.const 10
      i32.ge_s
      br_if $outer       ;; break: depth 1 (skip to end of block $outer)
      ;; loop body
      local.get 0
      i32.const 1
      i32.add
      local.set 0
      br $inner          ;; continue: depth 0 (restart loop $inner)
    end                  ;; end loop
  end                    ;; end block
  ;; post-loop
)
```

### 106.4.3 Exception Handling

Wasm exception handling (the `+exception-handling` feature) uses a Wasm-specific EH scheme different from the Itanium LSDA model used by other targets:

```wasm
(func $try_example
  try
    call $might_throw
  catch $cpp_exception_tag  ;; C++ exceptions use a tag
    ;; exception payload on stack
    call $handle_exception
  catch_all
    ;; catch any non-matching exception
    call $handle_unknown
  end
)
```

The LLVM Wasm backend rewrites LLVM `landingpad`/`invoke` instructions into Wasm `try`/`catch`/`throw` sequences via the [`WebAssemblyLowerEmscriptenEHSjLj`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/WebAssembly/WebAssemblyLowerEmscriptenEHSjLj.cpp) pass (for Emscripten) or directly for native Wasm EH.

---

## 106.5 Wasm SIMD128

### 106.5.1 The v128 Type

SIMD128 introduces the `v128` type: a 128-bit value interpretable as multiple lanes of smaller types. Lane interpretations:

| Lane type | Count | LLVM IR vector type |
|-----------|-------|---------------------|
| `i8x16` | 16 × i8 | `<16 x i8>` |
| `i16x8` | 8 × i16 | `<8 x i16>` |
| `i32x4` | 4 × i32 | `<4 x i32>` |
| `i64x2` | 2 × i64 | `<2 x i64>` |
| `f32x4` | 4 × f32 | `<4 x float>` |
| `f64x2` | 2 × f64 | `<2 x double>` |

### 106.5.2 Auto-Vectorization to SIMD128

LLVM's loop vectorizer targets Wasm SIMD128 when the feature is enabled:

```c
// This loop auto-vectorizes to Wasm SIMD128 with -O2 -msimd128
void add_floats(float *a, float *b, float *c, int n) {
    for (int i = 0; i < n; i++)
        c[i] = a[i] + b[i];
}
```

```bash
/usr/lib/llvm-22/bin/clang -O2 --target=wasm32-unknown-unknown \
    -msimd128 -S -emit-llvm add.c -o add_wasm.ll
```

The vectorized output uses `<4 x float>` LLVM IR vectors, which the Wasm backend lowers to `v128.load` / `f32x4.add` / `v128.store` sequences.

### 106.5.3 Wasm SIMD Intrinsics

Direct SIMD access uses Clang's `wasm_simd128.h` header:

```c
#include <wasm_simd128.h>

v128_t wasm_simd_dot(const float *a, const float *b) {
    v128_t va = wasm_v128_load(a);
    v128_t vb = wasm_v128_load(b);
    v128_t prod = wasm_f32x4_mul(va, vb);
    // Horizontal sum: shuffle and add
    v128_t shuf = wasm_i32x4_shuffle(prod, prod, 1, 0, 3, 2);
    v128_t sum1 = wasm_f32x4_add(prod, shuf);
    v128_t shuf2 = wasm_i32x4_shuffle(sum1, sum1, 2, 3, 0, 1);
    return wasm_f32x4_add(sum1, shuf2);
}
```

These functions are backed by LLVM intrinsics like `@llvm.wasm.swizzle.v16i8`, `@llvm.wasm.shuffle`, and the standard LLVM vector ISA intrinsics (`@llvm.fmuladd.v4f32`).

---

### 106.5.4 Tail Call Instructions

With `+tail-call`, Wasm gains `return_call` and `return_call_indirect` — direct equivalents of LLVM's `musttail` call + `ret` pattern. These are critical for functional-language and continuation-passing-style (CPS) code compiled to Wasm:

```wasm
(func $factorial_cps (param i64 i64) (result i64)
  ;; arg0 = n, arg1 = accumulator
  local.get 0
  i64.const 1
  i64.le_s
  if (result i64)
    local.get 1
  else
    local.get 0
    i64.const 1
    i64.sub
    local.get 0
    local.get 1
    i64.mul
    return_call $factorial_cps   ;; tail call: no stack growth
  end)
```

Without `return_call`, recursive functions in Wasm grow the Wasm call stack (separate from C's linear memory stack). With tail calls, mutual recursion and trampolines can execute in O(1) call stack space.

---

## 106.6 Wasm Object Format and Linking



### 106.6.1 Wasm Object Files

Wasm object files are not ELF files. They use the Wasm binary format itself as the object file format, with additional custom sections carrying linking metadata. Key sections:

| Section | Purpose |
|---------|---------|
| `Type` | Function type signatures |
| `Import` | Symbols imported from other modules |
| `Function` | Function index → type index table |
| `Table` | Function tables for `call_indirect` |
| `Memory` | Linear memory declaration |
| `Global` | Global variable declarations |
| `Export` | Exported symbols |
| `Code` | Function bodies (the actual bytecode) |
| `Data` | Static data initializers |
| `linking` (custom) | Symbol table, relocations, section metadata |
| `reloc.*` (custom) | Relocation entries for each section |
| `name` (custom) | Debug names for functions, locals |

The `reloc.*` custom sections encode relocations with the following types:

| Relocation Type | Code | Description |
|----------------|------|-------------|
| `R_WASM_FUNCTION_INDEX_LEB` | 0 | Function index (LEB128 encoded) |
| `R_WASM_TABLE_INDEX_SLEB` | 1 | Table slot index (signed LEB128) |
| `R_WASM_TABLE_INDEX_I32` | 2 | Table slot as i32 (for data sections) |
| `R_WASM_MEMORY_ADDR_LEB` | 3 | Memory address (unsigned LEB128) |
| `R_WASM_MEMORY_ADDR_SLEB` | 4 | Memory address (signed LEB128) |
| `R_WASM_MEMORY_ADDR_I32` | 5 | Memory address as i32 |
| `R_WASM_TYPE_INDEX_LEB` | 6 | Type section index |
| `R_WASM_GLOBAL_INDEX_LEB` | 7 | Global variable index |
| `R_WASM_FUNCTION_OFFSET_I32` | 8 | Offset within a function |
| `R_WASM_SECTION_OFFSET_I32` | 9 | Offset within a section |
| `R_WASM_TAG_INDEX_LEB` | 10 | Exception tag index |
| `R_WASM_TABLE_NUMBER_LEB` | 13 | Table number |

These relocation types are necessary because Wasm's binary format uses LEB128-encoded indices everywhere, and relocation patching must produce valid LEB128 values of the correct width.

### 106.6.2 LLD Wasm Linker

LLD's Wasm backend ([`lld/wasm/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/lld/wasm/)) handles Wasm object files and produces a linked Wasm module:

```bash
# Link Wasm objects with LLD
/usr/lib/llvm-22/bin/wasm-ld \
    --no-entry \
    --export-all \
    -o output.wasm \
    foo.o bar.o \
    -L/path/to/wasi-libc/lib \
    -lc
```

The linker performs:
- **Symbol resolution**: matching imports to exports across object files
- **Data segment merging**: combining `.data`, `.rodata`, `.bss` sections
- **Table construction**: building function tables for `call_indirect`
- **Stack allocation**: reserving a region of linear memory for the C stack
- **GOT emulation**: for position-independent code in dynamic linking scenarios

### 106.6.2b LLD Wasm Linker Internals

LLD's Wasm linker maintains several internal data structures that differ from the ELF linker's approach:

**Symbol table**: Wasm symbols are categorized as function symbols, data symbols, or global symbols. Unlike ELF, there is no concept of a GOT (Global Offset Table) for most Wasm symbols — data symbols are resolved to fixed offsets within the merged linear memory layout at link time. Dynamic linking uses a separate `__indirect_function_table` for function pointer resolution.

**Table construction**: The indirect function table (`__indirect_function_table`) is a Wasm `table` of `funcref` type. Any function whose address is taken (e.g., used as a function pointer) is assigned a table slot index. At link time, LLD collects all such functions across all input objects, assigns contiguous indices, and emits the table with an element section initializing each slot.

**Memory layout**: LLD places sections in this order in linear memory:
```
Linear Memory Layout:
  [0x00000000]  (null guard — address 0 never mapped)
  [0x00000008]  .rodata (merged from all objects)
  [aligned]     .data (initialized, copy from Wasm data section at startup)
  [aligned]     .bss  (zero-initialized, initialized by wasi start function)
  [growing]     heap (malloc uses sbrk(), expanding to memory limit)
  [top of mem]  C stack (grows downward from a fixed address)
```

The stack pointer (`__stack_pointer`) is a Wasm global, not a register, and is explicitly managed by compiler-generated code in function prologues/epilogues for functions that need stack frames.

**Weak symbols and comdat**: LLD Wasm supports weak symbols (unused if a strong definition exists) and COMDAT groups (one-definition-rule for template instantiations). The COMDAT mechanism in Wasm object files uses the `linking` custom section's `COMDAT_INFO` subsection to mark function and data segments as comdat members.

### 106.6.3 Emscripten Toolchain

Emscripten wraps Clang + LLD Wasm + JavaScript glue for browser deployment:

```bash
# Emscripten: compile C to Wasm + JS
emcc -O2 -s WASM=1 -s EXPORTED_FUNCTIONS='["_main"]' \
     -o output.html source.c

# Produces: output.html, output.js (loader), output.wasm
```

Emscripten handles the `--target=wasm32-unknown-emscripten` triple and provides:
- JavaScript glue code for DOM access, WebGL, filesystem emulation
- `EM_ASM` / `emscripten_run_script` for inline JavaScript from C
- `ASYNCIFY` transformation for async/await-like programming in synchronous C

---

### 106.6.4 The Wasm GC Proposal

The Wasm GC proposal (part of Wasm 3.0) adds garbage-collected reference types to Wasm, enabling high-level languages (Java, Kotlin, Dart, OCaml) to compile to Wasm without implementing their own garbage collector in linear memory. LLVM supports this via address space 10 (`externref`) and address space 20 (`funcref`):

```llvm
; Wasm GC: managed reference types in LLVM IR
; An 'externref' is an opaque reference into the Wasm GC heap
%obj = call ptr addrspace(10) @alloc_object()
; becomes: struct.new $MyType (if using Wasm GC types)

; Store to struct field (GC-managed struct)
call void @set_field(ptr addrspace(10) %obj, i32 42)

; GC-safe function reference
%fn = call ptr addrspace(20) @get_callback()
call void @invoke_fn(ptr addrspace(20) %fn, i32 %arg)
```

The Wasm GC feature requires the `+gc` subtarget feature and is largely incomplete in LLVM 22 for LLVM-compiled languages; it is more mature in the Emscripten and Binaryen ecosystem for Java/Kotlin.

## 106.7 BPF Target Overview

### 106.7.1 eBPF Architecture

Extended BPF (eBPF) is the modern evolution of the Berkeley Packet Filter virtual machine, now used far beyond packet filtering:

**Networking**: XDP (eXpress Data Path), TC (traffic control), socket filters  
**Tracing**: kprobes, uprobes, tracepoints, perf events  
**Security**: seccomp-BPF, LSM hooks  
**Scheduling**: sched_ext (custom CPU scheduler in BPF, Linux 6.12+)

The BPF ISA:
- **Registers**: R0–R10 (64-bit). R0=return value, R1–R5=function arguments, R6–R9=callee-saved, R10=read-only stack frame pointer
- **Instructions**: Fixed-width 64-bit encoding. Load/store, ALU64/ALU32, branch, call, exit
- **Memory model**: Stack (512 bytes max), maps (via kernel helpers), context pointer (e.g., `struct xdp_md *`)
- **Safety model**: The kernel BPF verifier checks all programs before execution; programs must terminate (no unbounded loops in older kernels, bounded loops via loop unrolling or `bpf_loop()` helper)

### 106.7.2 BPF Triples and Big-Endian Variant

| Triple | Endianness | Description |
|--------|-----------|-------------|
| `bpf-unknown-unknown` | Host (usually LE) | Standard eBPF |
| `bpfeb-unknown-unknown` | Big-endian | For big-endian hosts |
| `bpfel-unknown-unknown` | Little-endian | Explicit little-endian |

BPF programs are always executed on the host CPU in the kernel, so the byte order matches the host. The `bpfeb` triple is used for network packet processing where big-endian byte order is required.

### 106.7.3 BTF — BPF Type Format

BTF (BPF Type Format) is a compact, DWARF-inspired type information format embedded in BPF ELF objects:

```bash
# Compile BPF program with BTF
clang -g -O2 -target bpf -c prog.bpf.c -o prog.bpf.o

# Inspect BTF
bpftool btf dump file prog.bpf.o
```

BTF enables:
- **CO-RE** (Compile Once, Run Everywhere): programs compiled against one kernel version can relocate field accesses at load time for other kernel versions
- **bpftool skeleton generation**: generates C header with load/attach helpers
- **`bpf_printk` format checking**: `bpf_printk()` uses BTF for format string validation

---

## 106.8 BPF Target Machine and Subtarget

### 106.8.1 Backend Structure

The BPF backend lives in [`llvm/lib/Target/BPF/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/BPF/). It is a relatively straightforward RISC-like target with one major complication: the kernel verifier's restrictions mean the compiler must produce code that is statically verifiable in polynomial time.

```cpp
// BPFTargetMachine.cpp
BPFTargetMachine::BPFTargetMachine(const Target &T, const Triple &TT,
                                    StringRef CPU, StringRef FS,
                                    const TargetOptions &Options,
                                    std::optional<Reloc::Model> RM,
                                    std::optional<CodeModel::Model> CM,
                                    CodeGenOptLevel OL, bool JIT)
    : LLVMTargetMachine(T, computeDataLayout(TT), TT, CPU, FS,
                        Options, RM.value_or(Reloc::PIC_),
                        CM.value_or(CodeModel::Small), OL) { ... }
```

Data layout: `e-m:e-p:64:64-i64:64-i128:128-n32:64-S128`

### 106.8.2 BPF Subtarget Features

```tablegen
// BPFSubtarget features (BPF.td)
def FeatureALU32 : SubtargetFeature<"alu32",
    "HasALU32", "true",
    "Enable ALU32 support">;

def FeatureDwarfRIS : SubtargetFeature<"dwarfris",
    "UseDwarfRIS", "true",
    "Disable MCAsmInfo DwarfUsesRelocationsAcrossSections">;

def FeatureSolaris : SubtargetFeature<"solaris",
    "IsSolaris", "true",
    "Enable Solaris compatibility">;
```

The `+alu32` feature enables 32-bit ALU operations (subregister arithmetic), which reduces the need for zero-extension and can improve verifier performance by keeping values provably 32-bit.

### 106.8.3 BPF Calling Convention

BPF uses a fixed calling convention enforced by both the compiler and the kernel verifier:

| Register | Role |
|----------|------|
| R0 | Return value |
| R1 | First argument / context pointer |
| R2 | Second argument |
| R3 | Third argument |
| R4 | Fourth argument |
| R5 | Fifth argument |
| R6 | Callee-saved (preserved across BPF-to-BPF calls) |
| R7 | Callee-saved |
| R8 | Callee-saved |
| R9 | Callee-saved |
| R10 | Stack frame pointer (read-only) |

BPF helper functions (kernel functions called via the `BPF_CALL` instruction) also use this convention. The kernel exports a set of ~200 helper functions (e.g., `bpf_map_lookup_elem`, `bpf_probe_read_kernel`, `bpf_redirect`) that BPF programs may call.

### 106.8.3b BPF Instruction Encoding

Understanding the BPF instruction encoding helps when reading LLVM's `BPFInstrInfo.td` and interpreting `llvm-objdump` output of BPF programs:

```
BPF instruction format (64 bits):
  [7:0]   op    -- 8-bit opcode
  [11:8]  dst   -- destination register (R0–R15, but R11-R15 invalid)
  [15:12] src   -- source register
  [31:16] off   -- signed 16-bit offset (for memory/branch)
  [63:32] imm   -- signed 32-bit immediate

Opcode encoding:
  [2:0]  class: BPF_LD=0, BPF_LDX=1, BPF_ST=2, BPF_STX=3,
                BPF_ALU=4, BPF_JMP=5, BPF_JMP32=6, BPF_ALU64=7
  [3]    source: BPF_K=0 (imm), BPF_X=1 (register)
  [7:4]  operation code (varies by class)
```

Example BPF assembly with encoding:
```
; r0 = r1 + 42   (ALU64 ADD immediate)
07 00 00 00 2a 00 00 00
^     ^       ^
|     |       imm=42 (0x2a)
|     dst=r0 (0), src=r0 (0), off=0
BPF_ALU64 | BPF_ADD | BPF_K = 0x07

; r0 = *(u64 *)(r1 + 8)  (LDX DW)
79 01 08 00 00 00 00 00
^  ^  ^
|  |  off=8
|  dst=r0 (0), src=r1 (1)  -> 0x01 (low nibble=dst, high nibble=src)
BPF_LDX | BPF_MEM | BPF_DW = 0x79
```

The LLVM BPF backend's `BPFMCCodeEmitter` generates this encoding from `MachineInstr` nodes. The DWORD instruction format means BPF programs are always 8-byte aligned — a constraint the LLVM MC layer enforces automatically.

### 106.8.4 Target Lowering

[`BPFTargetLowering`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/BPF/BPFISelLowering.cpp) enforces BPF's type restrictions:

- **No floating-point**: BPF does not support FP in pre-v4 kernels; `BPFTargetLowering` marks all FP operations as `Expand`. Modern kernels (5.0+) added limited FP support but it remains rarely used.
- **No 128-bit integers**: expanded to pairs of 64-bit operations
- **No dynamic stack allocation**: `dynamic_alloca` is not legal
- **Atomic operations**: supported via `BPF_ATOMIC` instructions in kernels 5.12+; older kernels require spin locks via maps

---

## 106.9 BPF-Specific Passes and CO-RE

### 106.9.1 CO-RE: Compile Once, Run Everywhere

CO-RE is the mechanism by which BPF programs access kernel struct fields without requiring recompilation for each kernel version. It relies on three components:

1. **`__builtin_preserve_access_index()`**: A Clang builtin that records the type and field being accessed
2. **BTF relocation records**: Embedded in the object file to describe each field access
3. **libbpf relocation resolver**: Applied at load time, resolving field offsets against the target kernel's BTF

```c
// BPF program using CO-RE
#include "vmlinux.h"          // auto-generated kernel types
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_core_read.h>

SEC("kprobe/tcp_sendmsg")
int kprobe_tcp_sendmsg(struct pt_regs *ctx) {
    struct sock *sk = (struct sock *)PT_REGS_PARM1(ctx);

    // CO-RE field access: reads sk->sk_family at whatever offset
    // the running kernel places it, using BTF relocations
    __u16 family = BPF_CORE_READ(sk, sk_family);
    bpf_printk("tcp_sendmsg: family=%d\n", family);
    return 0;
}

char _license[] SEC("license") = "GPL";
```

`BPF_CORE_READ(sk, sk_family)` expands to a call to `__builtin_preserve_access_index()`:

```c
#define BPF_CORE_READ(src, a)  \
    bpf_core_read(&(src)->a, sizeof((src)->a), \
                  __builtin_preserve_access_index(&((typeof(src))0)->a))
```

### 106.9.1b BTF Relocation Kinds

libbpf processes CO-RE relocations at program load time. Each relocation has a kind that determines what information to look up in the target kernel's BTF:

| Kind | Value | Description |
|------|-------|-------------|
| `BPF_CORE_FIELD_BYTE_OFFSET` | 0 | Byte offset of a struct field |
| `BPF_CORE_FIELD_BYTE_SIZE` | 1 | Byte size of a field |
| `BPF_CORE_FIELD_EXISTS` | 2 | 1 if field exists, 0 otherwise |
| `BPF_CORE_FIELD_SIGNED` | 3 | 1 if field is signed integer |
| `BPF_CORE_FIELD_LSHIFT_U64` | 4 | Left shift for bitfield extraction |
| `BPF_CORE_FIELD_RSHIFT_U64` | 5 | Right shift for bitfield extraction |
| `BPF_CORE_TYPE_ID_LOCAL` | 6 | BTF type ID in local (compiled) BTF |
| `BPF_CORE_TYPE_ID_TARGET` | 7 | BTF type ID in target kernel BTF |
| `BPF_CORE_TYPE_EXISTS` | 8 | 1 if type exists in target BTF |
| `BPF_CORE_TYPE_SIZE` | 9 | sizeof(type) in target |
| `BPF_CORE_ENUMVAL_EXISTS` | 10 | 1 if enum value exists |
| `BPF_CORE_ENUMVAL_VALUE` | 11 | Numeric value of an enum constant |
| `BPF_CORE_TYPE_MATCHES` | 12 | 1 if type matches (structural check) |

The most common is `BPF_CORE_FIELD_BYTE_OFFSET`: when libbpf sees this relocation, it:
1. Looks up the struct type and field name in the target kernel's BTF
2. Computes the field's byte offset in the target kernel's struct layout
3. Patches the BPF instruction's immediate operand (the offset) with the target value

This allows a BPF program to access `task_struct->pid` reliably across kernel versions where the field may move because earlier fields changed size, even though the compiled object was built against a different (or synthetic) `vmlinux.h`.

### 106.9.2 BPFAbstractMemberAccess Pass

The [`BPFAbstractMemberAccess`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/BPF/BPFAbstractMemberAccess.cpp) pass processes `__builtin_preserve_access_index()` intrinsic calls in the IR. It:

1. Identifies GEP (GetElementPointer) instructions wrapped in the preserve intrinsic
2. Extracts the access chain (struct type → field name → offset)
3. Replaces the GEP with a `BPF_PSEUDO_BTFLD` instruction encoding the BTF field ID
4. Records a CO-RE relocation entry in a new `.BTF.ext` section

The resulting relocation tells libbpf: "at this instruction offset, look up field `sk_family` in struct `sock` via BTF and patch the offset accordingly."

### 106.9.3 BPFPreserveDIType Pass

[`BPFPreserveDIType`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/BPF/BPFPreserveDIType.cpp) ensures that DIType (DWARF type information) nodes referenced by `__builtin_preserve_type_info()` calls are not optimized away. Since BTF is derived from DWARF, prematurely eliminating type info nodes breaks BTF generation.

### 106.9.4 BPFMIChecking Pass

[`BPFMIChecking`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/BPF/BPFMIChecking.cpp) is a post-instruction-selection pass that validates the machine IR for BPF-specific constraints before emission:

- No indirect calls (only direct calls to known BPF helper IDs)
- No load/store from arbitrary kernel pointers without `bpf_probe_read_*`
- Stack frame pointer (R10) not modified
- No use of undefined registers in conditional branches

### 106.9.5 Stack Depth and Loop Restrictions

The 512-byte stack limit is checked by the kernel verifier at load time, but the compiler can help by:

```bash
# Show stack usage
clang -target bpf -O2 -Wframe-larger-than=400 prog.bpf.c -o prog.bpf.o
```

For loops, the kernel verifier historically required loop unrolling (bounded by a fixed trip count) or use of the `bpf_loop()` helper (kernel 5.17+):

```c
// Force loop unrolling for the verifier (older kernels)
#pragma unroll
for (int i = 0; i < MAX_ENTRIES; i++) {
    // ...
}

// Modern approach: bpf_loop() helper (kernel 5.17+)
struct ctx { int sum; };
static long sum_callback(__u64 idx, struct ctx *c) {
    c->sum += idx;
    return 0;
}

long ret = bpf_loop(100, sum_callback, &ctx, 0);
```

---

## 106.10 Clang BPF Toolchain

### 106.10.1 Compilation Pipeline

```bash
# Basic BPF compilation
clang -g -O2 -target bpf -c prog.bpf.c -o prog.bpf.o

# With specific BTF and architecture
clang -g -O2 -target bpf \
    -D__TARGET_ARCH_x86 \
    -I/usr/include/bpf \
    -c prog.bpf.c -o prog.bpf.o

# Inspect generated BPF instructions
llvm-objdump -d prog.bpf.o

# Inspect BTF types
bpftool btf dump file prog.bpf.o format c
```

The `-g` flag is essential for BPF programs — without it, no BTF is generated and CO-RE relocations cannot be emitted.

### 106.10.2 BPF ELF Object Format

BPF uses standard ELF as its object format (unlike Wasm). Special sections:

| Section | Contents |
|---------|---------|
| `.text` | BPF program bytecode |
| `SEC("xdp")` | XDP program (named section) |
| `SEC("kprobe/func")` | kprobe attachment |
| `SEC("tracepoint/sched/sched_switch")` | Tracepoint |
| `.maps` | BPF map definitions |
| `.BTF` | BTF type information (compressed DWARF) |
| `.BTF.ext` | CO-RE relocations, line info, func info |
| `.rodata` | Read-only data (const globals via readonly maps) |

### 106.10.3 libbpf Loading and CO-RE Relocation

libbpf handles the userspace side of BPF program management:

```c
// userspace loader using libbpf skeleton
#include "prog.skel.h"   // generated by bpftool gen skeleton

int main() {
    struct prog *skel = prog__open_and_load();
    if (!skel) { perror("open_and_load"); return 1; }

    // Attach kprobe
    int err = prog__attach(skel);
    if (err) { fprintf(stderr, "attach failed: %d\n", err); }

    // Run until Ctrl-C
    printf("Running... Ctrl-C to stop\n");
    for (;;) {
        int key, value;
        if (!bpf_map__lookup_elem(skel->maps.my_map,
                                   &key, sizeof(key),
                                   &value, sizeof(value), 0))
            printf("map[%d] = %d\n", key, value);
        sleep(1);
    }

    prog__destroy(skel);
    return 0;
}
```

The skeleton (`prog.skel.h`) is generated by:
```bash
bpftool gen skeleton prog.bpf.o > prog.skel.h
```

### 106.10.4 bpftrace and bcc

Two higher-level BPF toolchains build on LLVM's BPF backend:

**bpftrace**: A DTrace-inspired tracing language that compiles scripts to BPF at runtime:

```
# Trace TCP sendmsg latency
bpftrace -e '
kprobe:tcp_sendmsg { @start[tid] = nsecs; }
kretprobe:tcp_sendmsg /@start[tid]/ {
    @latency_us = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
}'
```

bpftrace compiles each `kprobe:` / `kretprobe:` clause to a separate BPF program via LLVM's JIT, attaches them to kernel probes, and handles map reading/printing.

**BCC** (BPF Compiler Collection): Python/C API for embedding C BPF code strings compiled at runtime:

```python
from bcc import BPF
prog = r"""
#include <uapi/linux/ptrace.h>
BPF_HASH(counts, u32, u64);
int count_syscall(struct pt_regs *ctx) {
    u32 pid = bpf_get_current_pid_tgid();
    u64 *cnt = counts.lookup_or_init(&pid, &(u64){0});
    (*cnt)++;
    return 0;
}
"""
b = BPF(text=prog)
b.attach_kprobe(event=b.get_syscall_fnname("write"), fn_name="count_syscall")
```

---

## 106.11 XDP, BPF Tail Calls, and sched_ext

### 106.11.0 BPF Tail Calls

BPF tail calls allow one BPF program to jump to another without growing the call stack — essential for modular BPF programs that need to implement dispatch tables, policy pipelines, or state machines:

```c
// Tail call through a BPF_MAP_TYPE_PROG_ARRAY
struct {
    __uint(type, BPF_MAP_TYPE_PROG_ARRAY);
    __uint(max_entries, 10);
    __uint(key_size, sizeof(__u32));
    __uint(value_size, sizeof(__u32));
} prog_array SEC(".maps");

SEC("xdp")
int xdp_router(struct xdp_md *ctx) {
    __u32 proto = parse_proto(ctx);
    // Tail-call to protocol-specific handler (index = protocol ID)
    bpf_tail_call(ctx, &prog_array, proto);
    // Falls through if tail call fails (prog slot empty or limit reached)
    return XDP_PASS;
}
```

`bpf_tail_call` lowers to a `BPF_JMP | BPF_CALL | BPF_X` instruction with `src_reg = BPF_PSEUDO_CALL` and `imm = BPF_FUNC_tail_call`. The verifier limits tail call chains to 33 hops to prevent infinite loops.

### 106.11.0b sched_ext — BPF CPU Scheduler

Linux 6.12 introduced `sched_ext`, allowing BPF programs to implement the CPU scheduler dispatch loop. This is among the most complex BPF use cases:

```c
// Minimal sched_ext scheduler (always enqueue to a global FIFO)
#include <scx/common.bpf.h>

char _license[] SEC("license") = "GPL";
UEI_DEFINE(uei);

void BPF_STRUCT_OPS(minimal_enqueue, struct task_struct *p, u64 enq_flags) {
    scx_bpf_dispatch(p, SCX_DSQ_GLOBAL, SCX_SLICE_DFL, enq_flags);
}

void BPF_STRUCT_OPS(minimal_dispatch, s32 cpu, struct task_struct *prev) {
    scx_bpf_consume(SCX_DSQ_GLOBAL);
}

SEC(".struct_ops.link")
struct sched_ext_ops minimal_ops = {
    .enqueue  = (void *)minimal_enqueue,
    .dispatch = (void *)minimal_dispatch,
    .name     = "minimal",
};
```

The `BPF_STRUCT_OPS` macro arranges for the scheduler callbacks to be compiled as individual BPF programs attached to the `sched_ext_ops` struct. LLVM compiles each callback independently, respecting BPF's per-program stack and instruction limits.

### 106.11.1 XDP Programs

XDP (eXpress Data Path) programs run in the NIC driver's RX path, before SKB allocation, enabling line-rate packet processing:

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_drop_icmp(struct xdp_md *ctx) {
    void *data     = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;
    if ((void*)(eth + 1) > data_end) return XDP_PASS;
    if (eth->h_proto != bpf_htons(ETH_P_IP)) return XDP_PASS;

    struct iphdr *ip = (void*)(eth + 1);
    if ((void*)(ip + 1) > data_end) return XDP_PASS;
    if (ip->protocol == IPPROTO_ICMP) return XDP_DROP;
    return XDP_PASS;
}
char _license[] SEC("license") = "GPL";
```

Bounds checks (the `> data_end` tests) are mandatory — the verifier proves that all packet accesses are within bounds. LLVM generates code that the verifier can track via range analysis.

Return values: `XDP_DROP` (discard), `XDP_PASS` (continue to kernel stack), `XDP_TX` (retransmit on same interface), `XDP_REDIRECT` (redirect to another interface or userspace via AF_XDP).

### 106.11.1b AF_XDP Zero-Copy Userspace

AF_XDP extends XDP by providing a zero-copy path directly to userspace via shared memory rings:

```c
// AF_XDP socket setup (userspace, using libxdp/libbpf)
#include <xdp/xsk.h>

struct xsk_socket_config cfg = {
    .rx_size = XSK_RING_CONS__DEFAULT_NUM_DESCS,
    .tx_size = XSK_RING_PROD__DEFAULT_NUM_DESCS,
    .libbpf_flags = 0,
    .xdp_flags    = XDP_FLAGS_DRV_MODE,  // Native driver mode
    .bind_flags   = XDP_USE_NEED_WAKEUP,
};

// umem: userspace memory region shared with kernel
struct xsk_umem *umem;
void *bufs;
posix_memalign(&bufs, getpagesize(), NUM_FRAMES * FRAME_SIZE);
xsk_umem__create(&umem, bufs, NUM_FRAMES * FRAME_SIZE,
                  &fq, &cq, NULL);

// Create AF_XDP socket
struct xsk_socket *xsk;
xsk_socket__create(&xsk, "eth0", 0, umem, &rx_ring, &tx_ring, &cfg);
```

The BPF program attached at XDP selects which packets are forwarded to AF_XDP by calling `bpf_redirect_map(&xsks_map, ctx->rx_queue_index, XDP_PASS)`. Packets that match are placed directly into the userspace ring without copying through the kernel networking stack.

### 106.11.2 Vectorization and BPF

LLVM's auto-vectorizer is generally incompatible with BPF: the BPF ISA has no SIMD instructions, and more importantly, vector code would produce IR that the verifier cannot analyze. The BPF target marks all vector operations as `Expand` in `BPFTargetLowering::setOperationAction()`, preventing vectorization. The `-fno-vectorize` flag is commonly recommended in BPF compilation guides, though the target lowering makes it redundant.

### 106.11.3 BPF Global Data and Maps

BPF programs access per-CPU and shared state through **maps** — kernel data structures accessed via helper calls. LLVM represents BPF map accesses as global variable references that libbpf resolves to map file descriptors at load time:

```c
// Map definition (BTF-based, modern style)
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key, u32);
    __type(value, u64);
} my_map SEC(".maps");

// Map access in BPF program
u64 *val = bpf_map_lookup_elem(&my_map, &key);
```

The `&my_map` reference compiles to a special `BPF_PSEUDO_MAP_FD` / `BPF_PSEUDO_MAP_IDX_VALUE` relocation that libbpf patches with the actual map file descriptor at load time.

---

## 106.12 BPF Verifier Interaction and Compiler Cooperation

### 106.12.1 What the Verifier Checks

The Linux kernel BPF verifier performs a static analysis of every BPF program at load time. Understanding what it checks explains many seemingly odd requirements in BPF compiler output:

**Memory safety**: Every load/store address must be provably within a valid memory region (stack, map value, context, packet data). The verifier tracks the type, size, and offset bounds of every register at every instruction — a form of abstract interpretation. Out-of-bounds access causes a load failure, not a runtime fault.

**Type safety**: BPF registers have tracked types beyond just value ranges: `PTR_TO_CTX`, `PTR_TO_MAP_VALUE`, `PTR_TO_STACK`, `SCALAR_VALUE`, etc. Loading a value from one type and using it as a pointer of another type is rejected. Arithmetic on context pointers is generally forbidden (context struct layout is kernel-version-specific).

**Liveness**: Registers that have not been initialized cannot be read. The verifier flags any branch that might lead to reading an uninitialized register, even if the runtime path never reaches it. This sometimes requires adding `else` branches with explicit zeroing that are dead code from a semantic standpoint.

**Loop bounds**: In Linux 5.2 and earlier, all loops required unrolling. From 5.3 onward, bounded loops are allowed but the verifier must prove the loop terminates — typically by tracking a counter register's bounds across iterations. LLVM's BPF backend can emit code that the verifier can analyze, but certain transformations (e.g., strength reduction of induction variables to pointers) can confuse the verifier's register tracking.

### 106.12.2 Writing Verifier-Friendly BPF Programs

Several coding patterns help the verifier and the compiler work together:

```c
// Pattern 1: explicit null check after map lookup
u64 *val = bpf_map_lookup_elem(&my_map, &key);
if (!val)               // REQUIRED: verifier tracks that val may be NULL
    return XDP_PASS;
*val += 1;              // now verifier knows val is non-null PTR_TO_MAP_VALUE

// Pattern 2: explicit bounds check for packet access
void *data     = (void *)(long)ctx->data;
void *data_end = (void *)(long)ctx->data_end;
struct ethhdr *eth = data;
// Verifier cannot allow eth->h_proto without this:
if ((void*)(eth + 1) > data_end)
    return XDP_PASS;    // safe: eth+1 <= data_end is now proven

// Pattern 3: bounded array index with masking (ensures < MAX_ENTRIES)
u32 idx = some_value & (MAX_ENTRIES - 1);  // verifier sees: 0 <= idx < MAX_ENTRIES
u64 *slot = &my_array[idx];                // access is now proven safe
```

The LLVM BPF backend is careful not to eliminate these bounds checks even at high optimization levels: LLVM's range analysis cooperates with the verifier's range tracking, and removing a bounds check that the verifier depends on would cause the program to be rejected at load time.

### 106.12.3 BPF and LLVM Optimization Interaction

The verifier's abstract interpretation is a path-sensitive analysis that tracks register value ranges. Certain LLVM optimizations produce code that is semantically equivalent but breaks the verifier's tracking:

- **Value range widening**: LLVM may replace `u32 idx = x & 0xF` with a sign-extended version that the verifier does not recognize as bounded. The `+alu32` subtarget feature helps by keeping 32-bit values in 32-bit subregisters, which the verifier tracks separately.
- **PHI node placement**: Multiple paths joining at a PHI node may produce a value that the verifier cannot prove is a valid pointer type (even if both predecessors are valid). Explicit type annotations via `bpf_cast_to_kern_ctx()` (a CO-RE helper) can re-establish type information.
- **Inlining thresholds**: BPF has a hard limit of 1 million instructions verified per program. Aggressive inlining can cause this limit to be exceeded. Use `__attribute__((noinline))` on large helper functions to force them to be separate BPF subprograms (each verified independently).

## 106.13 Wasm Threads and Atomics

### 106.13.1 The Wasm Threads Proposal

Wasm threads are enabled by the `+atomics` and `+bulk-memory` features together. The programming model uses SharedArrayBuffer on the Web and shared memory segments in WASI; the key new instructions are:

```wasm
;; Atomic load/store
i32.atomic.load  0              ;; load i32 atomically (offset 0)
i32.atomic.store 0              ;; store i32 atomically
i64.atomic.rmw.add 0            ;; i64 atomic fetch-add

;; Memory ordering (always seq_cst in Wasm 1.0)
memory.atomic.wait32 0          ;; futex wait (i32 value + timeout ns)
memory.atomic.notify 0          ;; futex wake (wake count)
```

Clang's `<stdatomic.h>` and C++ `<atomic>` headers compile correctly to Wasm atomic instructions when targeting `wasm32-unknown-unknown` with `+atomics`. C11's `memory_order_relaxed`, `memory_order_acquire`, etc., all map to `seq_cst` in Wasm 1.0 — the Wasm memory model provides only sequentially consistent atomics.

**pthread on Wasm**: Emscripten provides a pthreads implementation backed by Web Workers + SharedArrayBuffer. Clang compiles with `-pthread` for the `wasm32-unknown-emscripten` target:

```bash
# Emscripten pthreads
emcc -O2 -pthread -s USE_PTHREADS=1 \
     -s PTHREAD_POOL_SIZE=4 \
     parallel.c -o parallel.html
```

For standalone Wasm (WASI), threading is implemented via `wasi-threads` (a community proposal); thread creation uses the `wasi_thread_spawn` import.

### 106.13.2 Wasm Exception Handling Implementation

The Wasm exception handling proposal (`+exception-handling`) provides native try/catch with tags to distinguish exception types. LLVM's Wasm EH lowers C++ exceptions differently depending on the target:

**Native Wasm EH** (recommended for new code):
```bash
clang --target=wasm32-unknown-unknown \
      -fwasm-exceptions \     # Use native Wasm EH
      -O2 \
      -c exceptions.cpp -o exceptions.o
```

This generates `try`/`catch`/`throw` Wasm instructions. C++ uses a specific Wasm `tag` for `std::exception` and derived types; the Itanium ABI personality function is replaced by a Wasm-specific exception dispatch table.

**Emscripten JS-based EH** (legacy):
```bash
emcc -fexceptions exceptions.cpp -o exceptions.js
# Uses JavaScript setjmp/longjmp emulation — slower but broadly compatible
```

**Emscripten Wasm EH** (modern Emscripten with +exception-handling):
```bash
emcc -fwasm-exceptions exceptions.cpp -o exceptions.js
# Uses native Wasm EH instructions + JS glue for type matching
```

The `WebAssemblyLowerEmscriptenEHSjLj` pass handles the Emscripten-specific transformation of LLVM `invoke`/`landingpad` to JS-compatible patterns.

### 106.13.3 Component Model and Interface Types

The Wasm Component Model (post-Wasm 2.0 proposal) defines a type-safe interface layer between Wasm modules, replacing raw memory sharing with explicitly typed interfaces:

```wit
// WebAssembly Interface Type (WIT) definition
interface image-processing {
  /// Process an image in-place
  process: func(img: list<u8>, width: u32, height: u32) -> result<list<u8>, string>
}

world image-processor {
  import image-processing
  export run: func()
}
```

The `wasm-tools` toolchain generates Wasm component adapters from WIT definitions. LLVM's role is compiling the core Wasm module; the component adapter wraps it. This architecture is the foundation for the WASI 0.2 ecosystem (replacing WASI preview 1's C API).

## Chapter 106 Summary

- **WebAssembly** is a stack machine with structured control flow; LLVM's Wasm backend uniquely must convert register-based MachineIR into stack form via the `WebAssemblyRegStackify` and `WebAssemblyExplicitLocals` passes.
- **CFGStackify** converts LLVM's unstructured CFG to Wasm's required `block`/`loop`/`if` nesting, using the dominance tree and natural loop analysis to find the implied structure.
- Wasm **SIMD128** maps directly to `<4 x float>`, `<16 x i8>`, etc., LLVM vector types; the auto-vectorizer targets Wasm v128 when `+simd128` is enabled.
- **Wasm object files** use the Wasm binary format (not ELF) with custom `linking` and `reloc.*` sections; LLD's Wasm linker handles symbol resolution, segment merging, and table construction.
- **eBPF** is a kernel-hosted RISC VM with 11 registers, a 512-byte stack limit, and a kernel verifier that enforces termination and memory safety; LLVM's BPF backend targets `bpf-unknown-unknown` and `bpfel`/`bpfeb` triples.
- **CO-RE** (Compile Once, Run Everywhere) uses `__builtin_preserve_access_index()` compiled by `BPFAbstractMemberAccess` to emit BTF relocations allowing field-offset patching at load time by libbpf.
- **BPF loops** historically required unrolling (verifier limitation); modern kernels (5.17+) support `bpf_loop()` and bounded loop verification.
- **Higher-level BPF toolchains** (bpftrace, bcc) compile BPF programs at runtime using LLVM's JIT or AOT pipeline, then load them via libbpf's CO-RE relocation resolver.
- Cross-references: [Chapter 78 — The LLVM Linker (LLD)](../part-13-lto-whole-program/ch78-the-llvm-linker.md) for LLD Wasm linker internals; [Chapter 94 — The MC Layer](../part-14-backend/ch94-mc-layer-mir-test.md) for MC emission to Wasm binary format.


---

@copyright jreuben11
