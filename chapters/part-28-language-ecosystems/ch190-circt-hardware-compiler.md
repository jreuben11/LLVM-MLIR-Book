# Chapter 190 — CIRCT: Circuit IR Compilers and Tools

*Part XXVIII — Language Ecosystems, Emerging Targets, and Engineering Practice*

The open-source EDA (Electronic Design Automation) world has long suffered from a profound infrastructure deficit. RTL designers write Verilog or VHDL; proprietary synthesis tools from Synopsys and Cadence consume those files through undocumented internal IRs; simulation, formal verification, and physical synthesis each operate on separate, incompatible data models. There is no open intermediate representation, no composable pass manager, no principled type system, and no shared infrastructure for rewriting hardware descriptions. CIRCT — the Circuit IR Compilers and Tools project ([`github.com/llvm/circt`](https://github.com/llvm/circt)) — applies the LLVM/MLIR methodology to this problem. It defines a family of hardware-oriented dialects at multiple levels of abstraction, implements a lowering pipeline from high-level hardware description languages to synthesizable SystemVerilog, and provides first-class support for formal verification and high-level synthesis. This chapter covers the CIRCT dialect stack, the FIRRTL-to-SystemVerilog compilation pipeline, HLS infrastructure, formal verification facilities, and CIRCT's relationship with upstream MLIR. Readers should be familiar with MLIR fundamentals covered in [Chapter 129 — MLIR Philosophy](../part-19-mlir-foundations/ch129-mlir-philosophy.md), dialect definition techniques from [Chapter 132 — Defining Dialects with ODS](../part-19-mlir-foundations/ch132-defining-dialects-ods.md), and the pass infrastructure from [Chapter 149 — MLIR Analysis and Transformation Infrastructure](../part-21-mlir-transformations/ch149-mlir-analysis-transformation.md).

---

## Table of Contents

- [190.1 CIRCT Motivation and Architecture](#1901-circt-motivation-and-architecture)
  - [Why Hardware Compilers Need Compiler Infrastructure](#why-hardware-compilers-need-compiler-infrastructure)
  - [CIRCT as a Downstream MLIR Consumer](#circt-as-a-downstream-mlir-consumer)
  - [Repository Layout](#repository-layout)
  - [`firtool`: The Primary Driver](#firtool-the-primary-driver)
- [190.2 The CIRCT Dialect Stack](#1902-the-circt-dialect-stack)
- [190.3 The FIRRTL Dialect](#1903-the-firrtl-dialect)
  - [FIRRTL Background](#firrtl-background)
  - [FIRRTL Types](#firrtl-types)
  - [FIRRTL Textual Form (Chisel Output)](#firrtl-textual-form-chisel-output)
  - [Key FIRRTL Transformation Passes](#key-firrtl-transformation-passes)
- [190.4 The HW, Comb, and Seq Dialects](#1904-the-hw-comb-and-seq-dialects)
  - [HW: Structural Hardware](#hw-structural-hardware)
  - [Comb: Combinational Logic](#comb-combinational-logic)
  - [Seq: Sequential Elements](#seq-sequential-elements)
- [190.5 The Arc Dialect](#1905-the-arc-dialect)
  - [Arc Functions as Pure Bitvector Computations](#arc-functions-as-pure-bitvector-computations)
- [190.6 The Handshake Dialect](#1906-the-handshake-dialect)
  - [Token-Based Dataflow Execution](#token-based-dataflow-execution)
- [190.7 The SV Dialect and SystemVerilog Emission](#1907-the-sv-dialect-and-systemverilog-emission)
  - [SV Dialect Ops](#sv-dialect-ops)
  - [ExportVerilog: The Emission Pass](#exportverilog-the-emission-pass)
  - [File Splitting](#file-splitting)
- [190.8 The FIRRTL-to-SystemVerilog Compilation Pipeline](#1908-the-firrtl-to-systemverilog-compilation-pipeline)
  - [End-to-End Flow](#end-to-end-flow)
  - [The LowerFIRRTLToHW Pass](#the-lowerfirrtltohw-pass)
  - [Chisel 6 Interoperability](#chisel-6-interoperability)
  - [firtool Invocation Examples](#firtool-invocation-examples)
- [190.9 High-Level Synthesis](#1909-high-level-synthesis)
  - [The Calyx Language](#the-calyx-language)
  - [Dynamatic: Dynamic Scheduling via Handshake](#dynamatic-dynamic-scheduling-via-handshake)
  - [Comparison: CIRCT HLS vs. Proprietary Tools](#comparison-circt-hls-vs-proprietary-tools)
- [190.10 Formal Verification in CIRCT](#19010-formal-verification-in-circt)
  - [The Verif Dialect](#the-verif-dialect)
  - [circt-bmc: Bounded Model Checking](#circt-bmc-bounded-model-checking)
  - [The SMT Dialect](#the-smt-dialect)
- [190.11 MLIR Integration Details](#19011-mlir-integration-details)
  - [Tracking Upstream MLIR](#tracking-upstream-mlir)
  - [Hardware-Specific Op Interfaces](#hardware-specific-op-interfaces)
  - [Python Bindings and PyCDE](#python-bindings-and-pycde)
  - [ESSENT: Fast C++ Simulation](#essent-fast-c-simulation)
  - [Testing Infrastructure](#testing-infrastructure)
- [Chapter Summary](#chapter-summary)

---

## 190.1 CIRCT Motivation and Architecture

### Why Hardware Compilers Need Compiler Infrastructure

Contemporary digital design flows are structurally similar to the pre-LLVM compiler landscape. Every EDA vendor maintains its own closed IR that cannot be inspected, extended, or targeted by third-party passes. When a designer writes SystemVerilog, the synthesis tool parses it, converts it to a proprietary gate-level representation, applies technology-dependent optimization passes, and produces a netlist — all in a black box. The consequences are:

- **No composable analyses**: adding a custom dataflow analysis requires forking a proprietary tool or writing a Verilog parser from scratch.
- **No formal semantics**: SystemVerilog's language standard runs to 1,400 pages and contains ambiguities that different tools resolve differently.
- **No shared optimization infrastructure**: a new high-level synthesis tool must reimplement dead-code elimination, constant folding, and loop transformations rather than reusing a shared library.
- **No open debugging story**: when a synthesis pass miscompiles a design, there is no IR dump to inspect.

CIRCT's founding insight, articulated in Eldridge et al., *CIRCT: MLIR for Hardware Design* (ICCAD 2021, [doi:10.1109/ICCAD51958.2021.9643478](https://ieeexplore.ieee.org/document/9643478)), is that MLIR's architecture solves exactly these problems. MLIR provides: a general IR with recursive region structure, a type system extensible without patching the core, a pass manager with dependency tracking, ODS-generated operation definitions with verifiers, and a pattern-rewriting framework. CIRCT plugs hardware-specific dialects into this infrastructure rather than building a new compiler framework.

The structural parallel between hardware and software compilation is tighter than it first appears. A register-transfer level (RTL) design has modules (analogous to functions), ports (analogous to function arguments), wires (analogous to SSA values), and a graph of operations that compute new values from existing ones (analogous to instructions). The key differences are that hardware operations execute concurrently rather than sequentially, state is held in registers rather than a call stack, and the "execution time" of an operation is measured in clock cycles rather than nanoseconds. MLIR's region-based IR accommodates all of these differences: concurrent execution is modeled by the absence of sequential ordering between ops in the same region, registers introduce explicit clock-domain dependencies via the `!seq.clock` type, and the pass manager's dependency tracking correctly handles the non-standard notions of liveness and dataflow that hardware compilation requires.

A secondary motivation is toolchain integration. The LLVM/MLIR ecosystem already contains the testing infrastructure (`llvm-lit`, `FileCheck`), the build system integration (CMake macros), the documentation toolchain (`mlir-tblgen`), and the Python bindings infrastructure that hardware tools need. By building on MLIR, CIRCT inherits all of these for free — a new CIRCT dialect gets `FileCheck`-based testing, auto-generated C++ boilerplate from TableGen, and Python bindings from `mlir-tblgen --gen-python-bindings` without writing a single line of that infrastructure code.

### CIRCT as a Downstream MLIR Consumer

CIRCT is a downstream project: it tracks the upstream MLIR API (the `CIRCT_LLVM_COMMIT` variable in `cmake/modules/AddCIRCT.cmake` pins the exact LLVM revision) and uses all standard MLIR machinery. Operations are defined with ODS (`include/circt/Dialect/*/IR/*.td` TableGen files; cf. [Chapter 132 — Defining Dialects with ODS](../part-19-mlir-foundations/ch132-defining-dialects-ods.md)), pattern rewrites use PDL and `mlir::RewritePatternSet` (cf. [Chapter 135 — PDL and PDLL](../part-19-mlir-foundations/ch135-pdl-pdll.md)), and passes implement `mlir::OperationPass<>` against the MLIR pass manager. Op interfaces follow the pattern from [Chapter 133 — Op Interfaces and Traits](../part-19-mlir-foundations/ch133-op-interfaces-traits.md): hardware-specific interfaces like `FModuleLike` and `InnerSymbolOpInterface` are declared in `include/circt/Dialect/HW/HWOpInterfaces.h` and `include/circt/Dialect/FIRRTL/FIRRTLOpInterfaces.h`.

### Repository Layout

```
circt/
├── include/circt/Dialect/
│   ├── FIRRTL/   # FIRRTL dialect: ops, types, attrs, passes
│   ├── HW/       # Structural hardware dialect
│   ├── Comb/     # Combinational operations
│   ├── Seq/      # Sequential elements (registers, memories)
│   ├── SV/       # SystemVerilog constructs
│   ├── Arc/      # Optimizable RTL arc functions
│   ├── Handshake/# Dataflow/elastic circuits
│   ├── Calyx/    # Calyx structural HLS IR
│   ├── Verif/    # Formal verification properties
│   └── SMT/      # SMT solver interface
├── lib/Dialect/  # Implementation (.cpp files)
├── lib/Transforms/
├── tools/
│   ├── firtool/  # Primary compilation driver
│   └── circt-opt/ # Generic MLIR-opt-style tool
└── test/         # FileCheck-based regression tests
```

### `firtool`: The Primary Driver

`firtool` is to hardware what `mlir-opt` is to software MLIR — the command-line harness that runs the full compilation pipeline. Its primary function is transforming FIRRTL (Chisel's emitted IR) through the CIRCT dialect stack and emitting SystemVerilog. Key output-format flags:

| Flag | Stops after |
|------|-------------|
| `--parse-only` | Parsing + annotation lowering |
| `--ir-fir` | Full FIRRTL transformation passes |
| `--ir-hw` | Lowering to HW/Comb/Seq |
| `--ir-sv` | SV dialect construction |
| `--ir-verilog` | Verilog-level IR |
| `--verilog` (default) | SystemVerilog text output |
| `--split-verilog` | SV text, one file per module |
| `--btor2` | BTOR2 for model checking |

---

## 190.2 The CIRCT Dialect Stack

The table below summarizes the full dialect hierarchy from the highest abstraction to emitted text.

| Dialect | Level | Key Ops | Purpose |
|---------|-------|---------|---------|
| FIRRTL | RTL with Chisel semantics | `firrtl.module`, `firrtl.wire`, `firrtl.reg`, `firrtl.when`, `firrtl.connect` | Chisel compiler target; typed RTL with conditional connects |
| HW | Structural RTL | `hw.module`, `hw.instance`, `hw.output`, `hw.array_create` | Explicit-port structural hardware with InnerRef |
| Comb | Combinational logic | `comb.add`, `comb.mux`, `comb.icmp`, `comb.concat`, `comb.extract` | Pure, side-effect-free combinational ops; safe for DCE/CSE |
| Seq | Sequential elements | `seq.firreg`, `seq.compreg`, `seq.read_port`, `seq.write_port` | Clock-domain-aware registers and memory ports |
| Arc | Optimizable RTL | `arc.define`, `arc.call` | Pure bitvector functions enabling standard compiler opts |
| Handshake | Dataflow/elastic | `handshake.fork`, `handshake.join`, `handshake.buffer`, `handshake.mux` | Token-based dynamic scheduling for HLS |
| Calyx | Pipelined HLS | `calyx.component`, `calyx.group`, `calyx.seq`, `calyx.par` | Structural accelerator IR with explicit control |
| SV | SystemVerilog AST | `sv.if`, `sv.always`, `sv.assign`, `sv.reg`, `sv.wire` | Verilog constructs prior to text emission |
| Verif | Formal properties | `verif.assert`, `verif.assume`, `verif.cover` | SVA-style LTL/CTL properties for model checking |
| SMT | SMT interface | `smt.declare_fun`, `smt.assert`, `smt.check_sat` | Direct SMT encoding for Z3/Bitwuzla |

---

## 190.3 The FIRRTL Dialect

### FIRRTL Background

FIRRTL (Flexible Internal Representation for RTL) is the IR emitted by Chisel ([chisel-lang.org](https://www.chisel-lang.org)), a Scala-embedded hardware description DSL developed at UC Berkeley. The FIRRTL specification ([`github.com/chipsalliance/firrtl-spec`](https://github.com/chipsalliance/firrtl-spec)) defines a typed RTL language with explicit clock and reset signals, conditional connections (`when`/`else`), bundle (struct) and vector (array) aggregate types, and a semantics based on last-connect resolution. In CIRCT, FIRRTL is a first-class MLIR dialect: the `firrtl.circuit` operation is the top-level container, analogous to `builtin.module`, and every FIRRTL construct has a corresponding MLIR op and type.

### FIRRTL Types

The FIRRTL type system is richer than most RTL representations:

- `!firrtl.uint<N>` / `!firrtl.sint<N>` — unsigned and signed integer of bit-width N; width-inferred forms exist
- `!firrtl.clock` — a clock signal (not a Boolean)
- `!firrtl.reset` / `!firrtl.asyncreset` — synchronous and asynchronous reset
- `!firrtl.bundle<field: uint<8>, valid: uint<1>, ready: uint<1, flip>>` — struct with optional direction flip
- `!firrtl.vector<uint<8>, 16>` — fixed-size array
- `!firrtl.ref<T>` — reference type for debug signal probes (introduced for Grand Central)

### FIRRTL Textual Form (Chisel Output)

Chisel emits `.fir` files in FIRRTL's own textual syntax, distinct from the MLIR generic format. The following is a synchronously-reset 8-bit counter in FIRRTL textual form:

```firrtl
circuit Counter :
  module Counter :
    input clock : Clock
    input reset : UInt<1>
    input enable : UInt<1>
    output count : UInt<8>

    reg countReg : UInt<8>, clock with :
      reset => (reset, UInt<8>("h0"))

    node nextCount = add(countReg, UInt<8>("h1"))
    node nextTrunc = tail(nextCount, 1)

    when enable :
      countReg <= nextTrunc
    else :
      countReg <= countReg

    count <= countReg
```

`firtool` parses this into the FIRRTL MLIR dialect. The same circuit in MLIR textual form, after parsing:

```mlir
firrtl.circuit "Counter" {
  firrtl.module @Counter(
      in %clock: !firrtl.clock,
      in %reset: !firrtl.uint<1>,
      in %enable: !firrtl.uint<1>,
      out %count: !firrtl.uint<8>) {

    %c0_ui8 = firrtl.constant 0 : !firrtl.uint<8>
    %c1_ui8 = firrtl.constant 1 : !firrtl.uint<8>

    %countReg = firrtl.regreset %clock, %reset,
        %c0_ui8 : !firrtl.clock, !firrtl.uint<1>, !firrtl.uint<8>,
        !firrtl.uint<8>

    %0 = firrtl.add %countReg, %c1_ui8 :
        (!firrtl.uint<8>, !firrtl.uint<8>) -> !firrtl.uint<9>
    %nextTrunc = firrtl.tail %0, 1 : (!firrtl.uint<9>) -> !firrtl.uint<8>

    firrtl.when %enable : !firrtl.uint<1> {
      firrtl.connect %countReg, %nextTrunc : !firrtl.uint<8>, !firrtl.uint<8>
    } else {
      firrtl.connect %countReg, %countReg : !firrtl.uint<8>, !firrtl.uint<8>
    }
    firrtl.connect %count, %countReg : !firrtl.uint<8>, !firrtl.uint<8>
  }
}
```

### Key FIRRTL Transformation Passes

Before lowering to HW/Comb/Seq, a battery of FIRRTL-level passes runs in `firtool`'s pipeline. These passes operate on the FIRRTL MLIR dialect and are each implemented as a standard `mlir::OperationPass<firrtl::CircuitOp>` or `mlir::OperationPass<firrtl::FModuleOp>`:

| Pass | File | Purpose |
|------|------|---------|
| `InferWidths` | `lib/Dialect/FIRRTL/Transforms/InferWidths.cpp` | Resolve width-inferred integer types via constraint propagation |
| `InferResets` | `lib/Dialect/FIRRTL/Transforms/InferResets.cpp` | Propagate reset types (sync/async) through module hierarchy |
| `LowerTypes` | `lib/Dialect/FIRRTL/Transforms/LowerTypes.cpp` | Flatten bundle and vector aggregates to ground-type wires |
| `ExpandWhens` | `lib/Dialect/FIRRTL/Transforms/ExpandWhens.cpp` | Eliminate conditional connects; encode as mux trees |
| `Dedup` | `lib/Dialect/FIRRTL/Transforms/Dedup.cpp` | Structural hash-based module deduplication |
| `GrandCentral` | `lib/Dialect/FIRRTL/Transforms/GrandCentral.cpp` | Inject debug signal access paths via `!firrtl.ref<T>` |
| `LowerMemory` | `lib/Dialect/FIRRTL/Transforms/LowerMemory.cpp` | Lower FIRRTL memory to `seq` memory ports |
| `FlattenMemory` | `lib/Dialect/FIRRTL/Transforms/FlattenMemory.cpp` | Optionally flatten banked memories |

Several of these passes deserve closer inspection. **`InferWidths`** is a constraint-propagation pass: FIRRTL allows integers with unspecified widths (`UInt` without `<N>`), and the pass solves a system of width constraints by walking the dataflow graph. It uses a worklist algorithm that propagates minimum-width requirements forward through arithmetic operators — `add` requires output width `max(lhs_width, rhs_width) + 1`, `mul` requires `lhs_width + rhs_width`, and so on — until a fixed point is reached.

**`ExpandWhens`** implements FIRRTL's last-connect semantics for conditional wires. A FIRRTL `when` block assigns a wire conditionally; FIRRTL's semantics state that the last assignment in the sequential execution order wins. `ExpandWhens` transforms this into a tree of `firrtl.mux` operations, removing all `firrtl.when` regions from the module. This pass is order-sensitive and must run after `InferResets` and before `LowerTypes`.

**`Dedup`** performs module-level dead code elimination and deduplication simultaneously. It computes a structural hash of each module's body — considering op types, attributes, and topology, but not module names or instance names — and replaces all but one copy of structurally identical modules with instances of the canonical representative. For large Chisel designs that instantiate thousands of parametrically identical SRAM wrappers or leaf-level flip-flop cells, `Dedup` routinely reduces IR size by 10–100x.

**`GrandCentral`** implements the FIRRTL AugmentedBundle annotation mechanism, which allows external debug infrastructure (e.g., a signal tap or a JTAG-accessible register file) to inject read access to internal design signals without modifying the RTL source. Internally annotated wires are promoted to `!firrtl.ref<T>` reference types, which are then resolved across module boundaries and ultimately materialized as `sv.verbatim` output ports in the emitted SystemVerilog.

---

## 190.4 The HW, Comb, and Seq Dialects

### HW: Structural Hardware

The HW dialect represents technology-independent structural hardware — modules, instances, and aggregate types — with explicit port lists. It intentionally has no sequential or combinational semantics of its own; those are provided by Seq and Comb respectively. The separation is deliberate: a tool that operates only on structural hierarchy (e.g., a netlist traversal or a cross-module clock-domain analysis) need not understand combinational or sequential ops at all. This mirrors the MLIR philosophy of progressive lowering with well-scoped dialect responsibilities.

`hw.module` is the central structural op. Its signature lists explicit `in` and `out` ports with MLIR types (`i8`, `!seq.clock`, `!hw.array<16 x i32>`, etc.); there is no implicit clock or reset. `hw.instance` instantiates another module by symbol name, passing named port arguments and receiving named results. `hw.output` is the mandatory terminator that maps SSA values to the module's output ports.

```mlir
// A parameterized 8-bit adder module
hw.module @Adder(in %a : i8, in %b : i8, out sum : i9) {
  %c0_i1 = hw.constant 0 : i1
  %result = comb.add %a, %b : i8
  %ext = comb.concat %c0_i1, %result : i1, i8   // zero-extend to i9
  hw.output %ext : i9
}

// Instantiation
hw.module @Top(in %x : i8, in %y : i8, out %z : i9) {
  %sum = hw.instance "adder0" @Adder(a: %x: i8, b: %y: i8) -> (sum: i9)
  hw.output %sum : i9
}
```

The **InnerRef mechanism** enables cross-module references: an operation can carry an `inner_sym` attribute (a `hw.InnerSymAttr`), and `hw.hierpath` ops record a hierarchical path through the design for annotation and debug signal injection. A `hw.hierpath @myPath [@Top::@u_core, @Core::@u_alu, @ALU::@u_adder]` creates a stable name for a module instance within a design hierarchy. This mechanism is used by FIRRTL's GrandCentral pass to attach annotations to deep signals, and by SystemVerilog emission to generate `bind` statements and hierarchical signal references. The inner symbol namespace is managed by `InnerSymbolNamespaceCollection` to guarantee uniqueness across the design.

HW also provides aggregate type operations: `hw.array_create` constructs an array from a list of elements, `hw.array_get` indexes into an array, `hw.struct_create` and `hw.struct_extract` manipulate named struct types. These parallel FIRRTL's bundle and vector types but with MLIR-standard aggregate type syntax (`!hw.array<4 x i8>`, `!hw.struct<valid: i1, data: i8>`).

### Comb: Combinational Logic

`comb` operations are pure and side-effect-free — they have no clock, no state, and no memory effects. This makes them safe targets for DCE, CSE, and algebraic simplification passes. The purity guarantee is enforced by the MLIR effect system: `comb` ops declare no `MemoryEffects`, which means the pass manager's dead-code elimination and common-subexpression elimination passes apply without any hardware-specific modifications. This is a significant advantage over SystemVerilog, where there is no machine-checkable way to assert that a Verilog expression is side-effect-free.

An important `comb` design decision: all `comb` arithmetic ops produce results the same width as their inputs. There is no implicit extension. `comb.add` on `i8` operands produces an `i8` result, truncating the carry. If the carry is needed, the operands must be explicitly zero-extended first with `comb.concat %c0_i1, %x : i1, i8` to produce `i9` inputs. This strict no-implicit-extension rule makes bit-width reasoning straightforward: the type of every value in a `comb`-only circuit is known statically without any constraint propagation.

Core arithmetic: `comb.add`, `comb.sub`, `comb.mul`, `comb.divs`, `comb.divu`, `comb.mods`, `comb.modu`

Bitwise: `comb.and`, `comb.or`, `comb.xor`

Shifts: `comb.shl`, `comb.shru` (unsigned right), `comb.shrs` (arithmetic right)

Structural: `comb.concat` (bit concatenation), `comb.extract` (bit slice), `comb.replicate`

Comparison: `comb.icmp` with predicates `eq`, `ne`, `ult`, `ugt`, `ule`, `uge`, `slt`, `sgt`, `sle`, `sge`

Multiplexer: `comb.mux %sel, %true_val, %false_val`

Reduction: `comb.parity` (XOR reduction)

A combinational 4-bit ripple-carry adder in HW/Comb form:

```mlir
hw.module @FullAdder(in %a : i1, in %b : i1, in %cin : i1,
                     out sum : i1, out cout : i1) {
  %axb  = comb.xor %a, %b     : i1
  %sum  = comb.xor %axb, %cin : i1
  %aab  = comb.and %a, %b     : i1
  %xac  = comb.and %axb, %cin : i1
  %cout = comb.or  %aab, %xac : i1
  hw.output %sum, %cout : i1, i1
}

hw.module @Adder4(in %a : i4, in %b : i4,
                  out sum : i4, out cout : i1) {
  %a0 = comb.extract %a from 0 : (i4) -> i1
  %a1 = comb.extract %a from 1 : (i4) -> i1
  %a2 = comb.extract %a from 2 : (i4) -> i1
  %a3 = comb.extract %a from 3 : (i4) -> i1
  %b0 = comb.extract %b from 0 : (i4) -> i1
  %b1 = comb.extract %b from 1 : (i4) -> i1
  %b2 = comb.extract %b from 2 : (i4) -> i1
  %b3 = comb.extract %b from 3 : (i4) -> i1

  %c0 = hw.constant 0 : i1
  %s0, %c1 = hw.instance "fa0" @FullAdder(a:%a0:i1, b:%b0:i1, cin:%c0:i1)
                  -> (sum:i1, cout:i1)
  %s1, %c2 = hw.instance "fa1" @FullAdder(a:%a1:i1, b:%b1:i1, cin:%c1:i1)
                  -> (sum:i1, cout:i1)
  %s2, %c3 = hw.instance "fa2" @FullAdder(a:%a2:i1, b:%b2:i1, cin:%c2:i1)
                  -> (sum:i1, cout:i1)
  %s3, %c4 = hw.instance "fa3" @FullAdder(a:%a3:i1, b:%b3:i1, cin:%c3:i1)
                  -> (sum:i1, cout:i1)

  %sum_bits = comb.concat %s3, %s2, %s1, %s0 : i1, i1, i1, i1
  hw.output %sum_bits, %c4 : i4, i1
}
```

### Seq: Sequential Elements

The `seq` dialect models synchronous state: registers and memory ports. It makes clock domain membership explicit rather than implying it structurally.

**Clock type**: `!seq.clock` is a distinct type from `i1`; `seq.to_clock` and `seq.from_clock` convert between them. This type distinction enables static clock-domain analysis.

**`seq.firreg`** — a FIRRTL-style register with synchronous reset. Semantics: on the rising edge of `clock`, if `reset` is asserted, the register loads `resetValue`; otherwise it loads `next`.

```mlir
// 8-bit register with synchronous reset to 0, enabled update
hw.module @FlipFlop8(in %clk : !seq.clock, in %rst : i1,
                     in %en : i1, in %d : i8, out %q : i8) {
  %zero = hw.constant 0 : i8
  %q_reg = seq.firreg %next clock %clk reset sync %rst, %zero
               : i8
  %next = comb.mux %en, %d, %q_reg : i8
  hw.output %q_reg : i8
}
```

**`seq.compreg`** — comprehensively modeled register with optional clock enable and reset signals as named operands, useful when targeting different reset strategies:

```mlir
// Register with clock enable and asynchronous reset
hw.module @AsyncReg(in %clk : !seq.clock, in %arst : i1,
                    in %en : i1, in %d : i8, out %q : i8) {
  %zero = hw.constant 0 : i8
  %q_out = seq.compreg.ce %d, %clk, %en
               reset %arst, %zero : i8
  hw.output %q_out : i8
}
```

**Memory ports**: `seq.read_port` and `seq.write_port` model SRAM-style memories with separate read and write port operations; the memory itself is declared as an `seq.hlmem` op. The `seq.hlmem` representation captures the memory's data type, depth, and number of read/write ports without committing to a particular SRAM macro or memory compiler, leaving macro selection to a later technology mapping step.

The `!seq.clock` type distinction serves a deeper purpose than documentation. CIRCT's clock-domain analysis passes — notably the `CheckCombCycles` pass and the clock domain crossing checker — operate on the static type information to identify paths where a `!seq.clock`-typed value drives combinational logic directly (a design error) or where values cross from one clock domain to another without a synchronizer. These analyses are impossible in SystemVerilog because clock signals have no distinct type.

---

## 190.5 The Arc Dialect

### Arc Functions as Pure Bitvector Computations

The Arc dialect, developed within CIRCT, remodels synthesizable RTL as a collection of *arc functions* — pure functions from bitvector inputs to bitvector outputs with no hidden state and no clock references. An arc function is defined with `arc.define` and invoked with `arc.call`. Because arc functions are pure, they admit the full repertoire of standard compiler optimizations: DCE, CSE, constant folding, inlining, and function merging.

```mlir
// Arc function: combinational logic fragment
arc.define @ComputeNext(%state: i8, %inc: i1) -> i8 {
  %one  = hw.constant 1 : i8
  %zero = hw.constant 0 : i8
  %added = comb.add %state, %one : i8
  %next  = comb.mux %inc, %added, %state : i8
  arc.output %next : i8
}

// Invocation inside a module
hw.module @Counter(in %clk : !seq.clock, in %inc : i1, out %val : i8) {
  %next = arc.call @ComputeNext(%reg_out, %inc) : (i8, i1) -> i8
  %reg_out = seq.compreg %next, %clk : i8
  hw.output %reg_out : i8
}
```

The **arc canonicalization pipeline** runs standard MLIR canonicalization passes over `arc.define` bodies, then applies CIRCT-specific arc-merging passes that find structurally equivalent arc functions via hashing and replace duplicate definitions with a single canonical form. This directly mirrors the Dedup pass at the FIRRTL level but operates at the arc function granularity.

The Arc dialect's primary application is enabling CIRCT's fast simulation path. By extracting combinational logic into pure arc functions, the ESSENT simulator generator can treat each arc as a C++ function call in a simulation timestep scheduler. Two arcs with the same content are deduplicated and compiled once; the optimizer can also apply standard whole-function optimizations (LICM, GVN, vectorization) to arc function bodies before emitting C++. Designs compiled through the arc path consistently show 2–5x simulation speedup over Verilog-level simulators on large benchmark circuits.

There is also a formal verification application: an arc function is precisely a pure mathematical function over finite bitvectors, which admits direct translation to SMT bit-vector theory formulas. The `circt-bmc` tool (§190.10) uses this property by converting arc functions to `smt` dialect operations and encoding circuit unrolling as iterated function composition.

---

## 190.6 The Handshake Dialect

### Token-Based Dataflow Execution

The Handshake dialect implements a *dataflow circuit* model: each operation fires when all its input tokens have arrived; outputs are produced as tokens consumed by downstream operations. This elastic execution model is the foundation of dynamically-scheduled high-level synthesis, where loop bounds and memory latencies are unknown at compile time.

```mlir
handshake.func @DotProduct(%a_mem: memref<8xi32>, %b_mem: memref<8xi32>,
                            %start: none) -> i32 {
  %ctrl = handshake.control_merge %start : none
  // loop body elided for brevity
  handshake.return %sum : i32
}
```

Key ops:

- `handshake.merge` — non-deterministic merge of N input channels; fires when any input arrives; output is the value of whichever input was first
- `handshake.control_merge` — merge with a data output indicating which input fired; essential for building loop back-edges where a second input on iteration N must not block the first input on iteration N+1
- `handshake.fork` — replicates a single token to N outputs; implements the join-the-slowest discipline: the fork does not accept a new input token until all N outputs have been consumed
- `handshake.join` — synchronization barrier: fires only when all N input tokens have arrived; output is a control token used to sequence dependent computations
- `handshake.buffer` — introduces pipeline registers (elastic buffering) between producer and consumer; `[2] <dataflow>` specifies a 2-slot FIFO with dataflow (cut-through) semantics; `[2] <fifo>` specifies an opaque 2-slot FIFO that breaks back-pressure paths
- `handshake.mux` — conditional routing: a selector token determines which of N input data channels is forwarded; unused channels are drained on subsequent operations
- `handshake.d_return` / `handshake.return` — function exit, producing the output token

The correctness of a Handshake circuit depends on deadlock freedom: every token produced must be consumed, and no operation must be starved by a missing predecessor token. The Handshake dialect includes a `CheckFIFODepth` analysis pass that uses linear programming to compute minimum buffer sizes that guarantee deadlock-free execution under the elastic circuit model.

The Handshake dialect serves as the target for Dynamatic (§190.9) and can be lowered to HW/Comb/Seq for FPGA synthesis via the `HandshakeToHW` conversion pass. The conversion maps each Handshake op to an HW module instance that implements the ready/valid handshaking protocol: each channel becomes a triple of `(data, valid, ready)` wires, and the fire condition (`valid && ready`) implements the token semantics at the RTL level.

---

## 190.7 The SV Dialect and SystemVerilog Emission

### SV Dialect Ops

The SV dialect captures SystemVerilog constructs that have no counterpart in the structural HW/Comb/Seq dialects — procedural blocks, initial blocks, and arbitrary SV expressions. It is the last MLIR-level representation before text emission.

```mlir
hw.module @RegFileEntry(in %clk : !seq.clock, in %rst : i1,
                        in %we : i1, in %din : i8, out %dout : i8) {
  %c0_i8 = hw.constant 0 : i8
  %reg = sv.reg  : !hw.inout<i8>
  sv.always posedge %clk {
    sv.if %rst {
      sv.passign %reg, %c0_i8 : i8
    } else {
      sv.if %we {
        sv.passign %reg, %din : i8
      }
    }
  }
  %read = sv.read_inout %reg : !hw.inout<i8>
  hw.output %read : i8
}
```

Key SV ops:
- `sv.always posedge %clk` / `sv.alwayscomb` — Verilog `always` blocks
- `sv.initial` — simulation-only initialization
- `sv.assign` — continuous assignment (`assign wire = expr;`)
- `sv.passign` — procedural assignment inside `always`
- `sv.if` / `sv.ifdef` — conditional structure and ifdef guards
- `sv.verbatim` — raw Verilog text injection (escape hatch)
- `sv.wire`, `sv.reg`, `sv.logic` — declaration ops
- `sv.system` — `$display`, `$assert`, and other system tasks

### ExportVerilog: The Emission Pass

The `ExportVerilog` pass converts the HW/Comb/Seq/SV dialect mix into SystemVerilog text. It walks the IR in topological module order and emits each `hw.module` as a `module ... endmodule` block. `comb` ops become continuous assignment expressions (`assign wire = expr;`), `seq.firreg` ops become `always_ff` blocks with `if (reset) ... else ...` reset logic, and `sv.*` ops are emitted according to their direct SV analogue.

The emission pass handles several subtle emission correctness requirements. First, it enforces operator precedence for generated Verilog expressions, inserting parentheses when the tree structure does not match Verilog's precedence rules (avoiding the classic `a | b & c` vs. `a | (b & c)` ambiguity). Second, it manages name legalization: MLIR SSA values have arbitrary string names that may not be legal Verilog identifiers; the emission pass runs a `NameUniquer` to assign legal, unique Verilog names to all signals. Third, it handles the split between `wire` and `reg` declarations: signals driven by `always` blocks must be declared `reg` (or `logic`), while continuously-assigned signals are `wire`; this determination is made by analyzing the uses of each `sv.inout` value.

### File Splitting

For large designs, `emit.file` and `emit.file_list` control output organization. `firtool --split-verilog -o build/sv/` emits one `.sv` file per module into the specified directory, replicating the file-per-module convention of production EDA flows. The `emit.file` op carries a `fileName` attribute; the `ExportSplitVerilog` pass uses this attribute to group `hw.module` ops by file before invoking the per-file emission logic. A companion `emit.file_list` op produces a `.f` file (a standard EDA file list) that enumerates all emitted `.sv` files for downstream tools such as Synopsys VCS, Cadence Xcelium, or Verilator.

---

## 190.8 The FIRRTL-to-SystemVerilog Compilation Pipeline

### End-to-End Flow

The full pipeline from Chisel source to synthesizable SystemVerilog:

```
Chisel 6 (Scala DSL)
        │  sbt run / mill
        ▼
  FIRRTL text (.fir)
        │  firtool --format=fir
        ▼
  FIRRTL MLIR dialect
        │  InferWidths → InferResets → LowerTypes
        │  ExpandWhens → LowerMemory → Dedup
        │  GrandCentral → (other FIRRTL passes)
        ▼
  FIRRTL (flattened, ground types only)
        │  LowerFIRRTLToHW
        ▼
  HW + Comb + Seq + SV
        │  Arc canonicalization (optional)
        │  PrepareForEmission
        │  ExportVerilog
        ▼
  SystemVerilog text
```

### The LowerFIRRTLToHW Pass

`LowerFIRRTLToHW` is the conversion pass that bridges the FIRRTL and HW/Comb/Seq dialects. It runs after the full battery of FIRRTL transformation passes and converts each `firrtl.module` to an `hw.module`, each `firrtl.wire` to an `sv.wire`, each `firrtl.regreset`/`firrtl.reg` to a `seq.firreg`, each arithmetic op to a `comb.*` op, and each `firrtl.connect` to an `sv.assign`. The conversion is type-directed: `!firrtl.uint<N>` becomes `iN`, `!firrtl.clock` becomes `!seq.clock`, `!firrtl.bundle<...>` (after `LowerTypes` has run) has already been eliminated. Port directions are preserved: FIRRTL's `in`/`out` map directly to `hw.module`'s `in`/`out` port directions.

### Chisel 6 Interoperability

Chisel 6 (`chisel-lang.org`) is a Scala 3 DSL that generates FIRRTL. A minimal Chisel module:

```scala
import chisel3._
import chisel3.util._

class Counter(width: Int = 8) extends Module {
  val io = IO(new Bundle {
    val enable = Input(Bool())
    val count  = Output(UInt(width.W))
  })

  val countReg = RegInit(0.U(width.W))
  when(io.enable) {
    countReg := countReg + 1.U
  }
  io.count := countReg
}

object CounterMain extends App {
  emitVerilog(new Counter(), Array("--target-dir", "generated"))
}
```

Running `sbt run` invokes the Chisel elaborator, which calls `circt.stage.ChiselStage` internally, producing `Counter.fir`. From there, `firtool` drives the rest.

### firtool Invocation Examples

```bash
# Parse and emit FIR dialect MLIR (inspect after parsing + annotation lowering)
firtool --parse-only --format=fir Counter.fir -o - | less

# Run FIRRTL transformation passes, emit FIRRTL MLIR
firtool --ir-fir Counter.fir -o counter_fir.mlir

# Stop after HW/Comb/Seq lowering
firtool --ir-hw Counter.fir -o counter_hw.mlir

# Stop after SV dialect construction
firtool --ir-sv Counter.fir -o counter_sv.mlir

# Full pipeline: emit SystemVerilog to stdout
firtool Counter.fir

# Full pipeline: emit one file per module into build/sv/
firtool --split-verilog -o build/sv/ Counter.fir

# Emit with split output and file list for downstream tools
firtool --split-verilog -o build/sv/ \
        --output-annotation-file=build/annots.json \
        Counter.fir

# Enable arc optimizations before SV emission
firtool --mlir-print-ir-after-all --O2 Counter.fir -o counter_opt.sv
```

For a design with FIRRTL already in MLIR form (e.g., produced by a custom front end):

```bash
# Use circt-opt for MLIR-level pass pipeline experimentation
circt-opt \
  --firrtl-infer-widths \
  --firrtl-infer-resets \
  --firrtl-lower-types \
  --firrtl-expand-whens \
  --lower-firrtl-to-hw \
  --export-verilog \
  counter_parsed.mlir -o counter_out.sv
```

---

## 190.9 High-Level Synthesis

### The Calyx Language

Calyx ([`calyxir.org`](https://calyxir.org)) is a structural intermediate language designed specifically for pipelined hardware accelerators. Published in Nigam et al., *Predictable Accelerator Design with Time-Sensitive Affine Types* (ASPLOS 2021, [doi:10.1145/3445814.3446712](https://doi.org/10.1145/3445814.3446712)), Calyx separates a component's *structure* (wires, cells, groups) from its *control flow* (sequences, parallel sections, conditionals, loops).

```
// Calyx source: an 8-cycle dot product accumulator
component DotProd(clk: 1, reset: 1, @go go: 1)
          -> (@done done: 1) {
  cells {
    acc = std_reg(32);
    add = std_add(32);
    mul = std_mult_pipe(32);
  }
  wires {
    group compute {
      mul.left  = a_val.out;
      mul.right = b_val.out;
      mul.go    = !mul.done ? 1'd1;
      add.left  = acc.out;
      add.right = mul.out;
      acc.in    = add.out;
      acc.write_en = mul.done;
      compute[done] = acc.done;
    }
  }
  control {
    seq {
      @bound(8) while lt.out with cond { compute; }
    }
  }
}
```

The CIRCT Calyx dialect (`include/circt/Dialect/Calyx/`) represents this IR in MLIR form with `calyx.component`, `calyx.group`, `calyx.seq`, `calyx.par`, `calyx.if`, and `calyx.while` ops. The `CalyxToSequential` pass lowers Calyx to Seq/Comb for FPGA targeting.

Calyx's design philosophy is that separating *what* hardware computes (groups of assignments) from *when* it computes (the control program) makes pipelining and scheduling optimizations tractable. A compiler targeting Calyx from a high-level language (Python NumPy operations, polyhedral affine loops, or ML operator kernels) emits the structural cells and wires, then separately emits a control program. The Calyx compiler (written in Rust, distinct from CIRCT) can then apply **interval-based scheduling**: each group is assigned an interval of clock cycles during which it is active, and the control program enforces that non-overlapping groups share hardware resources. This achieves static initiation intervals for pipelined loops without requiring the front end to reason about clock cycles.

Within CIRCT, the Calyx dialect serves as the meeting point between HLS front ends (Polygeist, which compiles C through MLIR's affine dialect to Calyx; and Dahlia, a type-safe HLS language) and the hardware backend. The `CalyxToFSM` conversion materializes the Calyx control program as an explicit finite-state machine in the `fsm` dialect, which is then lowered to `seq`/`comb` for synthesis.

### Dynamatic: Dynamic Scheduling via Handshake

Dynamatic (ETH Zürich, [`github.com/EPFL-LAP/dynamatic`](https://github.com/EPFL-LAP/dynamatic)) implements a full HLS flow using the Handshake dialect for dynamic scheduling. It was the first production tool to adopt CIRCT's Handshake dialect as its core IR, replacing the earlier Dynamatic's custom C++ IR with MLIR operations and enabling it to reuse CIRCT's analysis and transformation infrastructure directly.

1. **C/C++ input** is compiled to LLVM IR by Clang.
2. **Affine analysis** extracts loop structure from LLVM IR.
3. **Handshake generation**: each LLVM instruction becomes a Handshake operation; memory accesses become LSQ (Load-Store Queue) tokens.
4. **Buffer insertion**: `handshake.buffer` ops are inserted to prevent deadlock and meet timing constraints.
5. **Handshake → HW/Comb**: the `HandshakeToHW` lowering produces a structural netlist.
6. **SV emission** via firtool/circt-opt for FPGA implementation.

The key advantage over static HLS tools (Vitis HLS, Bambu): Dynamatic produces circuits whose throughput adapts at runtime to actual memory latencies. A loop with unpredictable cache behavior achieves higher throughput than a statically-scheduled pipeline that must assume worst-case latency. The penalty paid is circuit area: each channel requires valid/ready handshaking wires, and each operation requires a firing-condition check circuit. For memory-bound workloads where latency variance is high (e.g., DRAM accesses from FPGA), the dynamic scheduling benefit outweighs the area overhead. For compute-bound loops with known latencies, Calyx's static scheduling produces smaller, faster circuits.

The CIRCT Handshake-to-hardware lowering is what enables Dynamatic to target actual FPGAs: the `HandshakeToHW` pass in CIRCT converts each Handshake op to a pre-characterized hardware module instance from a standard library (a "component library"), producing a structural HW/Comb netlist ready for firtool's SV emission stage.

### Comparison: CIRCT HLS vs. Proprietary Tools

| Property | Vitis HLS (Xilinx/AMD) | Bambu HLS | CIRCT (Dynamatic/Calyx) |
|----------|------------------------|-----------|--------------------------|
| IR openness | Closed internal IR | GIMPLE + custom | MLIR (open, extensible) |
| Scheduling | Static (II-constrained) | Static | Dynamic (Handshake) or static (Calyx) |
| Formal semantics | None published | None published | MLIR type system + verifiers |
| Custom pass injection | Plugin API (limited) | None | Standard MLIR pass interface |
| Formal verification | None | None | `verif` dialect + circt-bmc |
| Open source | No | Yes | Yes (Apache 2.0) |

---

## 190.10 Formal Verification in CIRCT

### The Verif Dialect

The `verif` dialect exposes SVA (SystemVerilog Assertions)-style properties as first-class MLIR operations, enabling formal verification as part of the compilation pipeline rather than as a post-synthesis afterthought.

```mlir
hw.module @CounterSpec(in %clk : !seq.clock, in %rst : i1,
                        in %en : i1, out %count : i8) {
  %c0_i8   = hw.constant 0   : i8
  %c1_i8   = hw.constant 1   : i8
  %c255_i8 = hw.constant 255 : i8

  // Register: synchronous reset to 0
  %count_reg = seq.firreg %next clock %clk reset sync %rst, %c0_i8 : i8
  %incremented = comb.add %count_reg, %c1_i8 : i8
  %next = comb.mux %en, %incremented, %count_reg : i8

  // Safety property: count never exceeds 200
  %max = hw.constant 200 : i8
  %in_range = comb.icmp ult %count_reg, %max : i8
  verif.assert %in_range : i1

  // Liveness property encoded as cover: count reaches 255
  %at_max = comb.icmp eq %count_reg, %c255_i8 : i8
  verif.cover %at_max : i1

  // Assumption: reset is asserted in cycle 0 (caller constrains input)
  verif.assume %rst : i1

  hw.output %count_reg : i8
}
```

- `verif.assert %cond` — the condition must hold in all reachable states; violation is a bug.
- `verif.assume %cond` — constrain the environment; only traces satisfying the assumption are considered.
- `verif.cover %cond` — the condition must be reachable under the assumptions; absence is a coverage gap.

### circt-bmc: Bounded Model Checking

`circt-bmc` is a standalone tool that unrolls a circuit for N clock cycles and encodes the resulting combinational circuit as a SAT/SMT formula. The check passes if no assignment satisfying the negation of every `verif.assert` is found within the bound. Conceptually, if a circuit has state vector `S_t` and input `I_t` at cycle `t`, and a combinational transition function `f(S_t, I_t) = (S_{t+1}, O_t)`, then a BMC check for N cycles asserts:

```
∀ t ∈ {0..N-1}: f(S_t, I_t) satisfies all verif.assert conditions
```

The negation (∃ some assignment violating an assertion) is encoded as a SAT/SMT query. `circt-bmc` uses the Arc dialect as an intermediate step: the circuit's combinational logic is first extracted into arc functions (each `arc.define` body), and the unrolling is performed symbolically over arc function calls. This design means `circt-bmc` benefits automatically from arc-level optimizations (constant folding, deduplication) before encoding the SMT formula.

```bash
# Check counter properties for 300 clock cycles
circt-bmc --bound=300 counter_spec.mlir

# Emit BTOR2 for use with a third-party model checker (e.g., btormc)
firtool --btor2 counter_spec.fir -o counter.btor2
btormc counter.btor2
```

### The SMT Dialect

For cases where bounded model checking is insufficient, the `smt` dialect provides a direct interface to SMT solvers (Z3, Bitwuzla). Circuit semantics are encoded as quantifier-free bit-vector formulas:

```mlir
smt.solver () : () -> () {
  %x = smt.declare_fun "x" : !smt.bv<8>
  %y = smt.declare_fun "y" : !smt.bv<8>
  %sum = smt.bvadd %x, %y : !smt.bv<8>
  %overflow_cond = smt.bvult %sum, %x : !smt.bv<8>
  smt.assert %overflow_cond
  smt.check_sat
}
```

The SMT dialect is primarily used by CIRCT's internal verification infrastructure and by `circt-bmc` when encoding multi-cycle unrollings. It is also the foundation for CIRCT's equivalence checking infrastructure: two implementations of the same specification can be encoded as a miter circuit (the XOR of their outputs), and an SMT unsatisfiability result proves the two implementations are observationally equivalent. This is the hardware analogue of the Alive2 translation validation approach described for LLVM IR in [Chapter 149 — MLIR Analysis and Transformation Infrastructure](../part-21-mlir-transformations/ch149-mlir-analysis-transformation.md).

The SMT dialect bridges two worlds that traditionally require separate tools: hardware design (which uses SystemVerilog simulators and waveform viewers) and formal verification (which uses model checkers and SAT solvers). By representing both the circuit and the proof obligation as MLIR operations, CIRCT enables a unified compilation pipeline where verification conditions are generated automatically from the annotated design IR, passed to an SMT solver, and the results reported through the same diagnostic infrastructure used for compilation errors. A designer who adds a `verif.assert` to their CIRCT module gets model-checking feedback within the same build step that generates SystemVerilog output, with no separate tool invocation or file format conversion.

---

## 190.11 MLIR Integration Details

### Tracking Upstream MLIR

CIRCT pins the exact upstream MLIR commit in `cmake/modules/AddCIRCT.cmake` via the `CIRCT_LLVM_COMMIT` variable. This pinning strategy is intentional: MLIR's C++ API evolves rapidly, and CIRCT dialects directly call core MLIR functions such as `mlir::PatternRewriter::replaceOpWithNewOp`, `mlir::TypeConverter::addConversion`, and `mlir::ConversionPatternRewriter::notifyMatchFailure`. A broken upstream change would silently produce incorrect hardware without the pin. The CIRCT project runs a continuous integration bot that periodically proposes bump PRs when the upstream MLIR API stabilizes enough for the full CIRCT test suite to pass. This pin is updated periodically by the CIRCT maintainers following the LLVM release cadence. Building CIRCT from source requires cloning LLVM at the pinned commit and building it alongside CIRCT:

```bash
git clone https://github.com/llvm/circt
cd circt
git submodule update --init --recursive
cmake -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DMLIR_DIR=$(pwd)/llvm/build/lib/cmake/mlir \
  -DLLVM_DIR=$(pwd)/llvm/build/lib/cmake/llvm \
  -DCIRCT_ENABLE_PYTHON_BINDINGS=ON
ninja -C build firtool circt-opt
```

### Hardware-Specific Op Interfaces

CIRCT defines several hardware-domain op interfaces built on the standard MLIR `OpInterface` mechanism (cf. [Chapter 133 — Op Interfaces and Traits](../part-19-mlir-foundations/ch133-op-interfaces-traits.md)):

- **`FModuleLike`** (`include/circt/Dialect/FIRRTL/FIRRTLOpInterfaces.h`): implemented by `firrtl.module` and `firrtl.extmodule`; provides `getPortList()`, `getPortType()`, `getPortName()`, and direction accessors. Passes that operate uniformly on any FIRRTL module use this interface rather than static type dispatch.

- **`InnerSymbolOpInterface`** (`include/circt/Dialect/HW/HWOpInterfaces.h`): implemented by any op that can carry an `inner_sym` attribute; provides `getInnerSymAttr()` and `setInnerSymAttr()`. This enables `hw.hierpath` to reference any op in the design hierarchy by a stable name.

- **`PortList`**: a companion interface to `FModuleLike` that abstracts over hw.module and firrtl.module port list access patterns.

- **`HWModuleLike`**: analogous to `FModuleLike` for the HW dialect; implemented by `hw.module`, `hw.module.extern`, and `hw.module.generated`.

### Python Bindings and PyCDE

CIRCT provides comprehensive Python bindings through the standard MLIR Python binding infrastructure. `pycde` (Python CIRCT Design Environment) adds a high-level design API on top:

```python
from pycde import System, Module, Clock, Reset
from pycde.types import UInt, Bits
from pycde import generator

class Counter(Module):
    clk   = Clock()
    rst   = Reset()
    en    = Input(Bits(1))
    count = Output(UInt(8))

    @generator
    def construct(ports):
        reg = Reg(UInt(8), clk=ports.clk, rst=ports.rst, rst_value=0)
        reg.assign(Mux(ports.en, reg + 1, reg))
        ports.count = reg

system = System([Counter])
system.generate()
system.compile()     # invokes firtool internally
system.emit_outputs()
```

pycde is located in `frontends/PyCDE/` in the CIRCT repository. It targets the same FIRRTL→SV pipeline as Chisel but from Python, making CIRCT accessible without a Scala/JVM toolchain.

### ESSENT: Fast C++ Simulation

ESSENT ([`github.com/ucsc-vama/essent`](https://github.com/ucsc-vama/essent)) is a high-performance RTL simulator that consumes CIRCT's HW/Comb/Seq MLIR and emits optimized C++ simulation code. By operating on CIRCT's typed IR rather than on Verilog text, ESSENT can apply aggressive optimizations — dead-register elimination, state partitioning, parallel evaluation scheduling — that are impossible on a Verilog-level simulator. Designs compiled through ESSENT from CIRCT consistently outperform Verilator on large RISC-V core designs.

The key insight is that a simulation timestep has a fixed structure: (1) evaluate all combinational logic (arc functions) to compute next-state values, (2) commit next-state values to all registers. By representing step (1) as a set of arc function calls with explicit dataflow dependencies, ESSENT can topologically sort the arc calls and emit a C++ evaluation function that processes them in dependency order with no dynamic dispatch. The resulting simulation binary has a predictable, cache-friendly memory access pattern — each register's state is a `uint64_t` in a struct-of-arrays layout — that maps well to modern out-of-order processors. ESSENT also partitions the design into independently evaluable *fragments* (disjoint sub-circuits with no combinational paths between them), enabling thread-parallel simulation on multicore hosts.

### Testing Infrastructure

CIRCT's test infrastructure mirrors LLVM's exactly: regression tests are FileCheck-annotated `.mlir` files in `test/Dialect/*/`. Each test runs a pass pipeline via `circt-opt` and checks the output with `FileCheck`. For example:

```bash
# Run the FIRRTL type lowering pass and check the output
# test/Dialect/FIRRTL/lower-types.mlir (first few lines):
# RUN: circt-opt --firrtl-lower-types %s | FileCheck %s
# CHECK: hw.module @Counter
# CHECK-NOT: firrtl.bundle
circt-opt --firrtl-lower-types test/Dialect/FIRRTL/lower-types.mlir
```

End-to-end integration tests use a `lit` configuration that invokes `firtool` on `.fir` or `.mlir` inputs and checks the emitted `.sv` output. This design ensures that every committed pass in CIRCT's pipeline is tested at both the unit (single-pass) and integration (full pipeline) level, with the same `llvm-lit` test runner used across the LLVM monorepo.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **FIRRTL 3.x spec alignment**: The ChipsAlliance FIRRTL specification continues evolving toward FIRRTL 3.x semantics (eliminating the fixed-point `UInt`/`SInt` vs. `Integer` distinction), and CIRCT's `firrtl` dialect is actively tracking these changes — see the ongoing discourse threads at [discourse.llvm.org/c/projects-that-want-to-use-llvm/CIRCT](https://discourse.llvm.org/c/projects-that-want-to-use-llvm/) and the FIRRTL spec repository PRs at `github.com/chipsalliance/firrtl-spec`.
- **Chisel 7 / firtool integration**: Chisel 7 targets tight firtool embedding (replacing the separate Chisel elaborator + firtool invocation with a unified JVM/MLIR compilation flow via the `chisel-mlir` project); near-term work focuses on direct Chisel-to-MLIR serialization without an intermediate `.fir` file, tracked in `github.com/chipsalliance/chisel`.
- **`circt-bmc` SMT backend for unbounded checking**: Expansion of `circt-bmc` to support k-induction (sufficient for unbounded property verification) by encoding the base case and inductive step as two separate SMT queries; PRs adding k-induction support to CIRCT's `Verif` pipeline are in active review as of early 2026.
- **Arc dialect simulation performance**: The ESSENT simulator integration is receiving scheduled loop-parallelism improvements — partitioning arc functions by SCC (strongly connected component) structure for multi-threaded simulation evaluation — with benchmarks targeting RISC-V core designs from the Chipyard framework.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **CIRCT as an LLVM sub-project**: CIRCT community discussions (discourse.llvm.org, 2024–2025) have repeatedly revisited formal LLVM sub-project status; if adopted, CIRCT would be included in the LLVM monorepo, sharing the LLVM release cadence directly rather than pinning individual commits, significantly reducing the CIRCT API breakage burden.
- **Calyx static scheduling via polyhedral methods**: Integration of the Polly-style polyhedral scheduler (cf. Chapter 112) into the Calyx compilation path, enabling automatic II (initiation interval) computation for nested loops with affine dependences — replacing the current manually-annotated `@bound` hints with analysis-driven static scheduling.
- **SPIR-V and MLIR GPU dialect roundtrip via CIRCT**: As GPU hardware description (shader compilation, spatial architectures) increasingly uses MLIR SPIR-V and `gpu` dialects, CIRCT is positioned as the structural lowering layer; mid-term work targets a verified roundtrip from `gpu.func` through HW/Comb to synthesizable RTL for spatial GPU architectures (e.g., AMD CDNA3-derived reconfigurable fabric tiles).
- **Formal equivalence checking between FIRRTL refinements**: Extending the SMT dialect's miter circuit encoding to support whole-module equivalence checking between FIRRTL refinement levels (e.g., verifying that a hand-optimized HW/Comb lowering is observationally equivalent to the original FIRRTL specification), analogous to the Alive2 translation validation approach for LLVM IR.

### 5-Year Horizon (Long-Term, by ~2031)

- **Open synthesis backend via CIRCT**: A fully open-source technology-mapping and place-and-route flow driven by CIRCT IRs — replacing Yosys + nextpnr with MLIR-native passes that lower HW/Comb/Seq through a technology-library dialect to a gate-level netlist IR, enabling end-to-end open EDA from Chisel source to FPGA bitstream within the LLVM ecosystem.
- **Temporal logic and liveness in `verif` dialect**: Extension of the `verif` dialect beyond safety properties (assertions that must hold in every reachable state) to full LTL/CTL liveness properties (assertions that something *eventually* happens), backed by new symbolic model-checking passes that integrate with IC3/PDR algorithms rather than pure BMC unrolling.
- **CIRCT for analog-mixed-signal (AMS) design**: A new MLIR dialect for continuous-time behavior (differential equations, Laplace-domain transfer functions) alongside the existing discrete-time RTL dialects, enabling co-simulation of analog and digital sub-circuits within a single CIRCT pipeline — analogous to the role of Verilog-AMS in the proprietary EDA world, but with open IR and formal semantics.

---

## Chapter Summary

- **CIRCT** applies LLVM/MLIR methodology to open-source EDA: composable passes, formal semantics, open IR, and a shared optimization infrastructure across hardware design tools.
- **FIRRTL** is the primary input dialect, serving as Chisel's backend IR. Its type system (`!firrtl.clock`, `!firrtl.bundle`, `!firrtl.ref`) captures hardware-specific semantics absent from software IRs.
- **The dialect stack** — FIRRTL → HW/Comb/Seq → Arc → SV — corresponds to progressive lowering of abstraction, mirroring the LLVM IR → MachineIR → MCInst pipeline in the software world.
- **HW** models structural netlist hierarchy; **Comb** provides pure combinational ops (safe for DCE/CSE); **Seq** models clock-domain-aware registers and memories with distinct `!seq.clock` type.
- **Arc** functions remodel RTL as pure bitvector functions, enabling the full MLIR canonicalization and optimization infrastructure to fire on hardware descriptions.
- **Handshake** enables dynamically-scheduled HLS: LLVM IR → Handshake → HW/Comb → FPGA, with runtime-adaptive throughput via elastic token passing.
- **Calyx** provides a structural pipelined HLS IR with explicit control separation; it lowers to Seq/Comb for synthesis.
- **firtool** is the primary driver; `--ir-fir`, `--ir-hw`, `--ir-sv`, `--verilog`, `--split-verilog`, and `--btor2` control pipeline depth and output format.
- **Formal verification** is first-class: `verif.assert/assume/cover`, `circt-bmc` for bounded model checking, and the `smt` dialect for direct Z3/Bitwuzla encoding.
- **pycde** provides Python-level hardware design targeting the same firtool pipeline, removing the Chisel/Scala dependency for Python-centric accelerator teams.
- CIRCT demonstrates that the MLIR downstream model scales to non-software domains: all standard MLIR infrastructure — ODS, PDL, pass manager, Python bindings, FileCheck tests — applies directly to hardware dialects.

---

*@copyright jreuben11*
