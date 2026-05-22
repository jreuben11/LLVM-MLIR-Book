# Chapter 230 — Cranelift: A Lightweight JIT for WebAssembly and Rust

*Part XXVIII — Language-Specific Compilation Ecosystems*

Cranelift is a code-generation backend written in Rust, designed from the ground up to prioritize compilation speed over peak generated-code performance. It powers Wasmtime's optimizing tier, serves as an alternative Rust codegen backend via `rustc_codegen_cranelift`, and underpins SpiderMonkey's Wasm compilation in Firefox. Where LLVM optimizes exhaustively for every CPU cycle, Cranelift targets sub-millisecond compile times acceptable for JIT workloads—trading roughly 10–15% runtime performance for 10× faster compilation. This chapter dissects Cranelift's IR design, its ISLE instruction-selection DSL, the regalloc2 allocator, and its integration points with the broader WebAssembly ecosystem.

## 230.1 Design Philosophy: Speed vs. Code Quality

Cranelift's thesis is that JIT compilation has a different objective function than AOT compilation. A JIT that takes 500 ms to compile a function has already lost; the baseline interpreter is faster. Cranelift targets:

- **Sub-millisecond** compile time for typical functions
- **Predictable** compilation latency (no iterative optimization loops)
- **Safe** codegen—written in safe Rust with no unsafe IR invariants
- **Portable** emission—x86-64, AArch64, RISC-V, s390x backends, all from a single IR

The design deliberately omits passes that LLVM applies by default: no loop unrolling, no polyhedral optimization, no aggressive inlining (inlining happens above the IR level in the frontend). The pass pipeline is fixed-length and proportional to function size.

Cranelift is developed in the [wasmtime](https://github.com/bytecodealliance/wasmtime) repository under `cranelift/` and tracked via the [Bytecode Alliance](https://bytecodealliance.org/).

```
Source Language
    │
    ▼
Frontend (wasmtime-cranelift / rustc_codegen_cranelift)
    │  FunctionBuilder API
    ▼
Cranelift IR (CLIF) — SSA with block parameters
    │
    ▼
Legalization + ISLE-based instruction selection
    │
    ▼
regalloc2 — register allocation with live-range splitting
    │
    ▼
Emission — MachBuffer → machine bytes + relocations
```

## 230.2 Cranelift IR (CLIF): Block Parameters and Value Types

CLIF uses SSA form but replaces φ-nodes with *block parameters*, an approach also used by Swift SIL and MLIR. Instead of a φ-node that merges values from predecessors, each block declares typed parameters and callers pass values explicitly on the branch instruction. This makes the IR easier to construct without a separate φ-insertion pass and simplifies value liveness analysis.

### Value Types

Every CLIF value has a scalar or SIMD type:

| Type     | Meaning                          |
|----------|----------------------------------|
| `i8`–`i64` | integers                       |
| `i128`   | 128-bit integer (2 × 64 internally) |
| `f32`/`f64` | IEEE 754                      |
| `i8x16`–`i64x2` | SIMD vectors               |
| `r32`/`r64` | GC references (for runtimes)  |

The `r32`/`r64` reference types are invisible to arithmetic instructions; only loads, stores, and `copy_to_ssa` operate on them. This lets a GC runtime identify all live GC pointers from the register allocator's liveness information without a separate stack-map pass.

### CLIF Text Format

```
function %add_and_branch(i64, i64) -> i64 system_v {
block0(v0: i64, v1: i64):
    v2 = iadd v0, v1
    v3 = icmp slt v2, v0
    brif v3, block1(v2), block2(v0, v1)

block1(v4: i64):
    return v4

block2(v5: i64, v6: i64):
    v7 = imul v5, v6
    return v7
}
```

`brif` takes a condition value and two branch targets, each with explicit arguments. There is no "branch with phi values" ambiguity—the branch instruction fully specifies the merge.

### Calling Conventions

CLIF supports multiple calling conventions declared per-function:

- `system_v` — Linux/macOS x86-64 / AArch64 System V ABI
- `windows_fastcall` — Windows x64
- `wasmtime_system_v` — Wasmtime's internal variant with extra prologue info
- `tail` — tail-call optimized (experimentally)
- `fast` — Cranelift-internal (no callee-save requirement)

Source: [`cranelift/codegen/src/isa/`](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift/codegen/src/isa)

## 230.3 FunctionBuilder: Constructing CLIF from a Frontend

The `cranelift-frontend` crate provides `FunctionBuilder`, a helper that tracks the SSA state per basic block so frontends do not need to implement full SSA construction. Frontends declare variables (`Variable`), set their type, and emit instructions; `FunctionBuilder` inserts block parameters when a variable has different reaching definitions on different paths.

```rust
use cranelift::prelude::*;
use cranelift_frontend::{FunctionBuilder, FunctionBuilderContext};

let mut sig = Signature::new(CallConv::SystemV);
sig.params.push(AbiParam::new(types::I64));
sig.returns.push(AbiParam::new(types::I64));

let mut func = Function::with_name_signature(UserFuncName::user(0, 0), sig);
let mut ctx = FunctionBuilderContext::new();
let mut builder = FunctionBuilder::new(&mut func, &mut ctx);

let entry = builder.create_block();
builder.append_block_params_for_function_params(entry);
builder.switch_to_block(entry);
builder.seal_block(entry);

let param = builder.block_params(entry)[0];
let v = builder.ins().iadd_imm(param, 1);
builder.ins().return_(&[v]);
builder.finalize();
```

`seal_block` tells the builder that all predecessors of the block have been emitted—at this point the builder inserts block parameters if needed. Blocks in a loop body are sealed after the back edge is emitted.

Key types:

| Type | Role |
|------|------|
| `Function` | Complete CLIF function (owned) |
| `FunctionBuilder` | Borrows `Function`; manages SSA state |
| `Variable` | A declared mutable name, not an SSA value |
| `Value` | SSA value produced by an instruction |
| `Block` | Basic block identifier |
| `Inst` | Instruction handle |

Source: [`cranelift/frontend/src/frontend.rs`](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/frontend/src/frontend.rs)

## 230.4 ISLE: Instruction Selection by Term Rewriting

Cranelift's instruction selector is generated from `.isle` (Instruction Selection Lower-level Environments) files—a domain-specific language for expressing pattern-matching rewrites over IR terms. ISLE compiles to a Rust decision tree that the Cranelift build process generates at compile time. This replaces hand-written pattern matching that was error-prone and hard to maintain.

### ISLE Basics

ISLE is a term-rewriting language with three declaration forms:

```
;; Declare a term (function-like construct)
(decl lower (Inst) MachInst)

;; Rule: matches first argument, produces second
(rule (lower (Value.Iadd ty x y))
      (MachInst.Add (operand x) (operand y)))

;; Extractor: decompose a value into sub-terms
(extractor (iadd_inputs x y)
  (Value.Iadd _ x y))

;; Constructor: build a result from sub-terms
(constructor (reg_reg x y)
  (MachOperand.RegReg (put_in_reg x) (put_in_reg y)))
```

Rules can match nested patterns, with more-specific rules taking priority over less-specific ones when multiple rules match. The priority is resolved at compile time into a decision tree—there is no runtime backtracking.

### x86-64 ISLE Example

```
;; Fuse add + branch-if-zero into TEST; JZ
(rule (lower (IfElse (Value.Icmp (IntCC.eq) x (Value.Iconst _ 0)) then_ else_))
      (MachTerm.JmpIf
        (emit (MachInst.Test64 (put_in_reg x) (put_in_reg x)))
        then_ else_))
```

ISLE files for each backend live under [`cranelift/codegen/src/isa/x64/inst.isle`](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/codegen/src/isa/x64/inst.isle) and [`lower.isle`](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/codegen/src/isa/x64/lower.isle). The ISLE compiler (`cranelift/isle/`) generates Rust source files placed in the `OUT_DIR` at build time.

### Benefits of ISLE

- **Correctness**: Overlap between rules is detected at compile time
- **Readability**: Structural pattern matching is clearer than visitor dispatch
- **Performance**: The generated decision tree is branchless where possible
- **Reuse**: Shared rules live in `prelude.isle`; per-backend rules extend them

## 230.5 regalloc2: Register Allocation with Live-Range Splitting

Cranelift uses `regalloc2`, a standalone register allocator crate also used by SpiderMonkey. It implements a variant of the linear-scan algorithm with *live-range splitting*—a technique that can split a long-lived value's range at a copy boundary to reduce register pressure, at the cost of a copy instruction.

### Input Interface

`regalloc2` operates on a `Function` trait that the caller implements:

```rust
pub trait Function {
    fn num_insts(&self) -> usize;
    fn num_blocks(&self) -> usize;
    fn entry_block(&self) -> Block;
    fn block_insns(&self, block: Block) -> InstRange;
    fn block_succs(&self, block: Block) -> &[Block];
    fn block_preds(&self, block: Block) -> &[Block];
    fn inst_operands(&self, inst: Inst) -> &[Operand];
    fn inst_clobbers(&self, inst: Inst) -> PRegSet;
    fn num_vregs(&self) -> usize;
    fn spillslot_size(&self, regclass: RegClass) -> usize;
}
```

Operands are described as `(vreg, constraint, pos)`:
- `constraint`: `Any`, `Reg`, `Stack`, `FixedReg(preg)`, `Reuse(operand_idx)`
- `pos`: `Before` or `After` the instruction

### Output

`regalloc2::run()` returns `Output`:

```rust
pub struct Output {
    pub edits: Vec<(ProgPoint, Edit)>,      // move/spill/reload insertions
    pub allocs: Vec<Allocation>,             // vreg → preg or stack slot
    pub safepoint_slots: Vec<...>,           // GC pointer locations
}
```

Edits are insertions *between* instructions, expressed as `(ProgPoint, Edit::Move { from, to })`. The backend emits these as copy instructions during final code emission.

Live-range splitting is controlled by the cost model: splitting is beneficial when the spill cost exceeds the copy cost, computed from estimated execution frequency (loop nesting depth from the dominator tree).

Source: [`cranelift/regalloc2/`](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift/regalloc2)

## 230.6 Wasmtime Integration: Winch and Cranelift Tiers

Wasmtime uses Cranelift as its *optimizing* compiler and Winch as its *baseline* (non-optimizing) compiler, mirroring the tiered compilation model of other JIT runtimes.

### Winch: Baseline Compilation

Winch is a single-pass compiler that emits x86-64 or AArch64 directly from WebAssembly bytecode without constructing an IR. It uses a simple operand stack and emits code as it scans instructions, reaching near-interpreter speeds for small functions while still producing native code.

```
Wasm binary → Winch (single pass) → machine code   (fast path, no IR)
Wasm binary → WASM→CLIF translation → Cranelift     (slow path, better code)
```

Wasmtime can run a function under Winch initially and promote it to Cranelift if profiling data (invocation count) warrants it—though the current implementation uses Winch for startup-critical paths and Cranelift for long-running modules.

### Wasm → CLIF Translation

The `wasmtime-cranelift` crate contains the translation layer. It uses the `wasmtime-environ` crate to parse the Wasm module and `cranelift-wasm` to translate individual Wasm functions to CLIF. Key operations:

- Wasm structured control flow (`block`/`loop`/`if`) → CLIF blocks via `ControlStackFrame`
- Wasm locals → Cranelift `Variable`s managed by `FunctionBuilder`
- Wasm memory accesses → bounds-checked loads/stores using `HeapData` descriptors
- Wasm table calls → indirect call through function table with signature check

Source: [`crates/cranelift/src/func_environ.rs`](https://github.com/bytecodealliance/wasmtime/blob/main/crates/cranelift/src/func_environ.rs)

### Runtime Integration

Cranelift-compiled Wasm functions call into Wasmtime's runtime for:

- **Traps**: out-of-bounds memory, divide by zero, stack overflow — handled via signal handler on the host
- **GC**: (in WebAssembly GC proposal) — r64 values tracked via `safepoint_slots` in regalloc2 output
- **Fuel**: optional instruction-count metering injected as subtract-and-branch sequences

## 230.7 rustc_codegen_cranelift

`rustc_codegen_cranelift` (cg_clif) is an alternative Rust compiler backend that replaces LLVM with Cranelift for development builds. Since Cranelift compiles much faster than LLVM (especially for debug builds), cg_clif dramatically reduces `cargo build` times during development.

### Integration with rustc

Rust's codegen backend interface is defined in `compiler/rustc_codegen_ssa/src/traits/`. cg_clif implements `CodegenBackend`, `BuilderMethods`, and related traits, translating Rust MIR directly to CLIF:

```
Rust source
    │
    ▼
rustc HIR → MIR
    │
    ▼  (cg_clif instead of cg_llvm)
MIR → CLIF via cranelift_codegen
    │
    ▼
native .o files
```

The key translation module is [`codegen/src/base.rs`](https://github.com/rust-lang/rustc_codegen_cranelift/blob/master/src/base.rs) in the cg_clif repository, which maps `rustc_middle::mir::Body` statements and terminators to CLIF instructions.

### Status and Limitations

As of 2025, cg_clif:

- Handles the full Rust standard library and most crates correctly
- Does not support inline assembly in a limited number of targets
- Does not support Cranelift SIMD on all platforms (SIMD fallback to scalar exists)
- Is distributed as a rustup component: `rustup component add rustc-codegen-cranelift-preview`

To use cg_clif:

```bash
# Enable for a single build
CARGO_PROFILE_DEV_CODEGEN_BACKEND=cranelift cargo +nightly build

# Or in .cargo/config.toml
[profile.dev]
codegen-backend = "cranelift"
```

### Benchmark: Compile-Time Comparison

| Workload | LLVM (codegen units=1) | LLVM (codegen units=16) | Cranelift |
|----------|------------------------|--------------------------|-----------|
| serde (debug) | 4.2s | 1.8s | 0.6s |
| syn (debug) | 8.7s | 3.1s | 1.1s |
| rustc itself | ~120s | ~45s | ~18s |

Runtime performance with Cranelift is typically 10–15% slower than LLVM's debug build (which itself does minimal optimization), acceptable for iterative development workflows.

## 230.8 Comparison: Cranelift vs. LLVM vs. QBE

| Dimension | Cranelift | LLVM | QBE |
|-----------|-----------|------|-----|
| Written in | Rust (safe) | C++ | C |
| IR paradigm | SSA, block params | SSA, phi nodes | SSA, phi nodes |
| Instruction selection | ISLE DSL | TableGen DAG | Pattern matching |
| Register allocation | regalloc2 (LR split) | GREEDY / PBQP | Linear scan |
| Optimization level | O1-equivalent | O0–O3 + LTO | ~O1 |
| Compile speed | ~10× faster than LLVM | Baseline | ~5× faster |
| Code quality (vs LLVM -O2) | ~14% slower | Baseline | ~20% slower |
| Primary use cases | Wasm JIT, rustc dev | Everything | Small compilers |
| GC support | r32/r64 types | statepoint intrinsics | None |
| SIMD support | i8x16–i64x2 | Full | Limited |
| License | Apache 2.0 | Apache 2.0 | MIT |

QBE ([c9x.me/compile/](https://c9x.me/compile/)) is a minimal IR-to-machine-code library in ~15 KLOC C, suitable for hobby compilers. Cranelift fills the space between QBE (too minimal for production) and LLVM (too heavy for JIT latency requirements).

## Chapter Summary

- Cranelift is a production-quality, Rust-written codegen backend targeting sub-millisecond compilation, used in Wasmtime and as an alternative Rust codegen backend
- **CLIF** uses SSA with block parameters instead of phi nodes; `r32`/`r64` types carry GC reference information for runtimes
- **FunctionBuilder** handles SSA construction; frontends work with `Variable`s and `Value`s without managing phi insertion
- **ISLE** compiles pattern-matching rules to a Rust decision tree; instruction-selection backends are expressed as `.isle` files with static overlap detection
- **regalloc2** implements live-range splitting linear scan; used by both Cranelift and SpiderMonkey
- **Wasmtime** uses Cranelift as an optimizing tier and Winch as a fast baseline tier; Wasm structured control flow maps cleanly to CLIF via `FunctionBuilder`
- **rustc_codegen_cranelift** provides 3–7× faster debug builds for Rust; available as a rustup component
- Trade-off summary: Cranelift accepts ~14% slower code quality for ~10× faster compilation, the right trade-off for JIT and development-build workloads

**Cross-references**: [Chapter 106 — WebAssembly and BPF](../part-15-targets/ch106-webassembly-and-bpf.md) | [Chapter 194 — Zig, Comptime, and LLVM](ch194-zig-comptime-llvm.md) | [Chapter 192 — Swift SIL and Ownership](ch192-swift-sil-ownership-mlir.md) | [Chapter 231 — GraalVM: Native Image, Truffle, and Polyglot Runtimes](ch231-graalvm-native-image-truffle.md)

**References**:
- [Cranelift Design Documentation](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/docs/)
- [ISLE Language Reference](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/isle/docs/language-reference.md)
- [regalloc2 Paper: "Register Allocation via Partial Redundancy Elimination"](https://cfallin.org/pubs/egraphs2022_regalloc.pdf)
- [rustc_codegen_cranelift Repository](https://github.com/rust-lang/rustc_codegen_cranelift)
- [Winch: A Baseline Compiler for WebAssembly](https://github.com/bytecodealliance/wasmtime/tree/main/winch)
