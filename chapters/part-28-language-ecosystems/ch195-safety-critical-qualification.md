# Chapter 195 — Safety-Critical Toolchain Qualification: DO-178C, ISO 26262, and Ferrocene

*Part XXVIII — Language Ecosystems, Emerging Targets, and Engineering Practice*

Every deployed compiler makes thousands of silent decisions — constant folding, alias disambiguation, branch inversion, dead-code removal — any of which can corrupt a program whose correctness depends on a property the compiler treats as undefined. In consumer software, a miscompilation is a bug report. In a flight control computer, an automotive brake controller, or a railway switching unit, it is a latent fatality. Safety-critical standards exist precisely to force explicit, auditable answers to the question: *can you trust this toolchain not to introduce errors into your safety-relevant binary?* This chapter explains the regulatory machinery for toolchain qualification across the major standards (DO-178C/DO-330 for avionics, ISO 26262 for automotive, IEC 61508 for industrial), covers the specific obligations placed on compiler vendors, analyzes where LLVM's optimizer creates qualification risk, examines MISRA C:2023 and SPARK 2014 as language-level complements, and details Ferrocene — the first qualified Rust compiler — as a worked example of what a complete qualification package looks like in practice. Readers should be familiar with LLVM's pass infrastructure from [Chapter 59 — The New Pass Manager](../part-10-analysis-middle-end/ch59-new-pass-manager.md), optimizer semantics from [Chapter 62 — Scalar Optimizations](../part-10-analysis-middle-end/ch62-scalar-optimizations.md), sanitizer methodology from [Chapter 110 — User-Space Sanitizers](../part-16-jit-sanitizers/ch110-user-space-sanitizers.md), and formal verification from [Chapter 167 — Hoare Logic and Separation Logic](../part-24-verified-compilation/ch167-hoare-separation-logic.md).

---

## 195.1 What Toolchain Qualification Means

### The Trusted Base Problem

Formal verification (see [Chapter 168 — CompCert](../part-24-verified-compilation/ch168-compcert.md)) proves that a compiler preserves observable semantics across all inputs. That proof is powerful but not the same as qualification. A certified airborne system does not require that the compiler be *provably correct*; it requires that a documented, auditable process establishes *sufficient confidence* that the compiler does not silently corrupt outputs. The distinction matters: CompCert closes the mathematical gap but does not produce the regulatory artifacts that a certification authority (the FAA, EASA, or a National Type Authority) accepts as evidence. Qualification and formal verification are complementary; neither replaces the other.

The problem qualification solves is the trusted base problem. Every compiled binary is only as trustworthy as the toolchain that produced it. Without qualification, safety standards require either:

1. **Manual output review**: every generated object file is read by a human against the source, or
2. **Independent retranslation**: the program is compiled by a second, independent toolchain and results compared at the object level.

Both alternatives are expensive enough that qualifying the toolchain — once, for a given version and target — is almost always the economic choice for high-volume production.

### Tool Confidence Levels

The regulatory concept shared across standards is the **Tool Confidence Level (TCL)**, which rates how much harm a tool error could cause and how likely such an error is to go undetected. The TCL is determined by two factors:

- **Tool Impact (TI)**: can a tool error propagate into the safety-relevant output? TI-1 means no impact possible; TI-2 means impact is possible.
- **Tool Error Detection (TD)**: is a tool error detectable downstream (by reviews, tests, or independent means) before the system is deployed? TD-1 means detection is very likely; TD-2 means detection is less likely.

The product of TI and TD determines which qualification measures are mandatory:

| TI | TD | TCL | Required Measures |
|----|----|-----|-------------------|
| 1  | any | TCL-1 | None — tool outputs not safety-relevant |
| 2  | 1  | TCL-2 | Validation evidence (test report, anomaly list) |
| 2  | 2  | TCL-3 | Validation evidence + formal qualification test suite + independent assessment |

A compiler is almost always TI-2 by definition — its output is the artifact that executes on the target hardware. A compilation toolchain at the highest safety level therefore reaches TCL-3 in virtually all frameworks, requiring both validation evidence and a structured test suite.

### The "No Additional Measures" Threshold

A fully qualified tool operating within its qualified configuration is said to require *no additional measures* with respect to its output. This phrasing is significant: for a qualified Level A/ASIL D compiler, the project does not need to manually review the generated assembly or run a second independent compilation. The qualification evidence substitutes for those measures. Conversely, using a non-qualified compiler — or using a qualified compiler outside its documented configuration (wrong optimization level, unapproved target, flags not in the qualification scope) — reinstates the requirement for additional measures. The economic argument for qualification is therefore strongest when the same compiler version and configuration is used across many projects over the tool's certification lifetime (typically five to ten years for avionics, three to five years for automotive).

The "qualified configuration" concept is operationally important. A compiler qualified at `-O1` for `thumbv7em-none-eabi` is *not* qualified at `-O2` for the same target, and not at `-O1` for a different target, unless those configurations are explicitly covered in the qualification package. Every change in compiler flags, compiler version, or target triple requires either a re-qualification or a change-impact analysis showing that the change does not affect the qualification evidence.

---

## 195.2 DO-178C and DO-330 (Avionics)

### Software Levels and DO-178C

RTCA DO-178C ([*Software Considerations in Airborne Systems and Equipment Certification*](https://www.rtca.org/publications/do-178c/), 2011) defines five software levels based on the consequence of failure:

| Level | Failure Consequence | Example |
|-------|--------------------|---------| 
| A | Catastrophic — loss of aircraft | Primary flight control |
| B | Hazardous — serious injury or death | Autopilot |
| C | Major — passenger discomfort / workload increase | Cabin pressurization |
| D | Minor — operational limitation | In-flight entertainment |
| E | No safety effect | Maintenance logging |

Level A software demands the most rigorous development and verification process. A compiler used to generate Level A code must itself be qualified. DO-178C references tool qualification via the **DO-330** supplement.

### DO-330: Tool Qualification Levels

[RTCA DO-330](https://www.rtca.org/publications/do-330/) (*Software Tool Qualification Considerations*, 2011) introduces Tool Qualification Levels (TQL-1 through TQL-5) mapped to the software levels the tool supports and to the tool's role in the development process. The mapping is mediated by three **tool criteria**:

- **Criterion 1**: Tool output is part of the airborne software (e.g., a compiler whose object code is loaded onto the aircraft). This is the most demanding criterion.
- **Criterion 2**: Tool automates a verification process (e.g., a model checker verifying requirements coverage).
- **Criterion 3**: Tool removes or prevents the application of other development/verification processes.

The TQL assignments by criterion and software level are:

| Tool Criterion | Level A | Level B | Level C | Level D |
|---------------|---------|---------|---------|---------|
| Criterion 1 (output in airborne SW) | TQL-1 | TQL-2 | TQL-3 | TQL-4 |
| Criterion 2 (automates verification) | TQL-4 | TQL-5 | TQL-5 | TQL-5 |
| Criterion 3 (eliminates a dev/verif activity) | TQL-1 | TQL-2 | TQL-3 | TQL-4 |

A Criterion 1 development tool for Level A software is classified **TQL-1** — the highest qualification burden. A Criterion 2 verification tool for Level A software is typically TQL-4 or TQL-5 because a separate independent test activity exists to catch its errors. This nuance matters: a qualified compiler carries a TQL-1 obligation, while a qualified coverage measurement tool for the same Level A project may only require TQL-4.

### DO-330 Qualification Artifacts

A TQL-1 qualification package consists of:

**Tool Qualification Plan (TQP)**: defines the qualification scope, target environment, tool version, lifecycle data to be produced, and the plan for traceability from Tool Operational Requirements through test cases to evidence. The TQP must be reviewed and agreed upon with the certification authority before the qualification activity begins.

**Tool Operational Requirements (TOR)**: a formal specification of what the tool shall do. For a C compiler, TORs cover every normative behaviour: correct lowering of every C construct, faithful implementation of every calling convention, correct handling of every target-architecture instruction class. TORs are the specification that qualification tests must cover.

**Tool Verification Cases and Procedures (TVCP)**: the test cases. Each TVCP must be traceable to one or more TOR items, have defined expected outputs, and be executed on the actual qualified version of the tool on the actual qualified hardware platform. For GCC or LLVM-based compilers, this suite typically runs into thousands of cases covering language constructs, optimization interactions, linker behavior, and assembler output fidelity.

**Tool Accomplishment Summary (TAS)**: the completed evidence package — TQP, TORs, TVCP execution records, tool version hash, target hardware description, and a closure statement that all TOR items are covered. The TAS is the document handed to the certification authority.

AdaCore's **GNAT Pro** for Ada/SPARK is qualified to DO-178C Level A with a TQL-1 DO-330 package, making it the most widely deployed DO-178C-qualified compiler. The LDRA Tool Suite (test coverage measurement and requirements tracing) and Polyspace Code Prover (abstract interpretation–based runtime error detection) are commonly used as DO-330 Criterion 2 and Criterion 3 qualified verification tools alongside a qualified compiler.

The DO-178C ecosystem also has three other supplement standards: **DO-331** (model-based development, covering tools like MATLAB/Simulink), **DO-332** (object-oriented technology, adding rules for C++ dynamic dispatch and templates), and **DO-333** (formal methods, providing credit for deductive proof, abstract interpretation, and model checking in lieu of some structural coverage objectives).

---

## 195.3 ISO 26262 (Automotive)

### ASIL Levels

[ISO 26262:2018](https://www.iso.org/standard/68383.html) (*Functional Safety — Road Vehicles*) defines four Automotive Safety Integrity Levels (ASIL A through D) plus QM (Quality Management — no specific safety requirement):

| Level | Residual Risk | Example |
|-------|--------------|---------|
| QM | Not safety-relevant | Infotainment |
| ASIL A | Lowest | Seat position memory |
| ASIL B | Moderate | Windshield wipers (rain sensor) |
| ASIL C | High | Active suspension |
| ASIL D | Highest — single-point fault coverage ≥99% | Electronic power steering, ABS |

ASIL D is the hardest to achieve. The probability of systematic (design) failures must be "extremely low" — ISO 26262 does not give a single number for systematic failures (unlike random hardware faults, which have a quantitative bound), but it imposes detailed process and method requirements to achieve this.

### Tool Classification Under ISO 26262 Part 8

ISO 26262 Part 8 (Supporting Processes), Clause 11, covers software tool qualification. The framework uses the same TI × TD matrix described in §195.1. A compiler generating ASIL D code is TI-2; its error detectability by the development process determines TD and hence TCL.

A TCL-3 compiler qualification under ISO 26262 requires:
- Documented tool validation report covering the tool's functional scope
- A qualification test suite run on the exact tool version and target compiler configuration
- Tool version control evidence (hash, release notes, known anomaly list)
- Evidence that the tool's known anomalies do not affect the project's use case

**Example determination**: a compiler optimization that generates dead code in an edge case — TI=2 (generated code is deployed to the ECU), TD=2 (code review alone may not catch wrong dead code), TCL=3. A requirement management tool that generates no binary output — TI=1, TCL=1 (no qualification needed regardless of TD).

### ASIL Decomposition and Compiler Implications

ISO 26262 permits **ASIL decomposition**: splitting an ASIL D safety function into two independent ASIL B channels (written ASIL B(D) in ISO 26262 notation) where each channel alone cannot cause the catastrophic failure but both must fail simultaneously. The two channels are typically implemented as independent software running on independent hardware, with cross-monitoring. The compiler qualification implication is significant: if the two channels are compiled by different compilers, each compiler only needs to be qualified at ASIL B rather than ASIL D, substantially reducing the qualification burden. This is one reason automotive suppliers sometimes maintain two separate toolchains — one GCC-based and one Clang-based — for dual-channel ASIL D implementations.

ASIL decomposition does *not* reduce the combined safety goal. If both channels are compiled by the same Ferrocene toolchain, there is a common-cause failure risk: a Ferrocene bug could corrupt both channels identically. ISO 26262 Part 9 (Automotive Safety Integrity Level-oriented and Safety-oriented Analysis) requires a common-cause failure analysis that accounts for this. A typical mitigation is to use different compiler versions, different optimization settings, or different host machines for the two channels, introducing sufficient independence that a single compiler defect is unlikely to affect both outputs identically.

### Qualification Test Suite Approach

The standard qualification approach for an automotive compiler toolchain runs the validation test suite on target hardware (typically an Arm Cortex-M or Cortex-R device, or a RISC-V ECU processor) and compares outputs against reference outputs:

```bash
# Compile each test case for the target ECU
arm-none-eabi-gcc -O1 -mcpu=cortex-m4 -mfpu=fpv4-sp-d16 -mfloat-abi=hard \
    tc_001_integer_overflow.c -o tc_001.elf

# Run on hardware-in-the-loop and compare expected output
hirel-runner --board stm32f429 --elf tc_001.elf --expected tc_001.expected.txt

# Cross-check with the qualification test report
python3 tools/gen_tvcp_report.py --results results/ --tors tors.csv --out tvcp_report.pdf
```

Green Hills MULTI and Wind River Diab compilers have long-standing ISO 26262 ASIL D qualification certificates. GCC-based toolchains can be qualified using commercial qualification kits from suppliers such as LDRA or Hitex. As of 2025, Ferrocene (§195.5) is the first Rust compiler to reach ASIL D qualification.

---

## 195.4 IEC 61508 and Sector Standards

### IEC 61508: The Base Standard

[IEC 61508:2010](https://www.iec.ch/publication/5515) (*Functional Safety of Electrical/Electronic/Programmable Electronic Safety-related Systems*) is the horizontal standard from which most sector safety standards derive. It defines four Safety Integrity Levels (SIL 1–4), with SIL 4 being the most demanding:

| SIL | PFD (demand mode) | PFH (continuous mode, per hour) |
|-----|-------------------|----------------------------------|
| 1   | 10⁻² to 10⁻¹     | 10⁻⁶ to 10⁻⁵                   |
| 2   | 10⁻³ to 10⁻²     | 10⁻⁷ to 10⁻⁶                   |
| 3   | 10⁻⁴ to 10⁻³     | 10⁻⁸ to 10⁻⁷                   |
| 4   | 10⁻⁵ to 10⁻⁴     | 10⁻⁹ to 10⁻⁸                   |

SIL applies to the entire Safety Function, not just software. **Software SIL** adds requirements on design methodology, testing thoroughness, and tool confidence. IEC 61508 Part 3 (Software Requirements) prescribes which design techniques and measures are:
- Mandatory (M) at a given SIL
- Highly recommended (HR) — expected unless justified otherwise
- Recommended (R) — optional but beneficial

For software SIL 3–4, formal verification (in the sense of DO-333 and Hoare logic) is *Highly Recommended*. Structured programming, fully defined language subsets (e.g., MISRA C), and dynamic analysis (coverage measurement, MC/DC) are *Mandatory*. The compiler is explicitly listed as a tool requiring confidence assessment. IEC 61508 Part 6 Annex C provides the tool confidence determination guidelines that underlie the TI × TD matrix.

MC/DC (**Modified Condition/Decision Coverage**) is a structural coverage criterion required at SIL 3–4 in IEC 61508 and at Level A in DO-178C. It requires that every condition in every decision can independently affect the decision's outcome. A 100% MC/DC test suite run on the final compiled binary is a primary evidence artifact; a qualified compiler ensures that the binary's control flow faithfully represents the source-level decisions without spurious optimization-induced merges or deletions.

**Sector standards** derived from IEC 61508:

| Standard | Sector | Integrity Levels |
|----------|--------|-----------------|
| IEC 61508 | General industrial (E/E/PE) | SIL 1–4 |
| ISO 26262 | Road vehicles | ASIL A–D |
| DO-178C | Civil avionics | Level A–E |
| EN 50128 | Railway | SIL 0–4 (mirrors 61508) |
| IEC 62304 | Medical device software | Class A/B/C |
| IEC 61511 | Process industry (oil & gas) | SIL 1–3 |

EN 50128 (railway software) maps almost directly to IEC 61508 structure; a SIL 4 railway signaling system faces tool qualification requirements essentially identical to IEC 61508 SIL 4. IEC 62304 (medical devices) is more process-focused but requires compiler tool qualification for Class C software (life-supporting devices). Ferrocene's IEC 61508 SIL 4 certificate therefore extends transitively toward railway and medical uses.

---

## 195.5 Ferrocene — Qualified Rust Toolchain

### Architecture and Lineage

[Ferrocene](https://ferrocene.dev) is a downstream fork of `rustc` maintained jointly by Ferrous Systems and AdaCore. It tracks upstream Rust releases closely — Ferrocene 25.02 (February 2025) is based on Rust 1.84 — and adds the qualification infrastructure on top: the Ferrocene Language Specification, the FerreQual test suite, and the regulatory certification documents.

Ferrocene's relationship to upstream `rustc` (covered in [Chapter 177 — rustc: Architecture, MIR, and Codegen Backends](../part-26-ecosystem-frontiers/ch177-rustc-architecture-mir-codegen.md)) is analogous to how a commercial Linux distribution relates to the upstream kernel: the code base is substantially identical, but the distribution adds validation, binary reproducibility guarantees, and long-term support commitments. Ferrocene does not fork the LLVM backend; it uses exactly the LLVM version that upstream rustc targets, which means LLVM codegen bugs are shared. The qualification evidence documents the scope of what is qualified — including which LLVM passes execute at which optimization levels for which targets.

Current qualification status (Ferrocene 25.02):
- **ISO 26262 ASIL D** (automotive) — assessed by TÜV SÜD
- **IEC 61508 SIL 4** (industrial) — assessed by TÜV SÜD
- **DO-178C DAL A** qualification in progress (target: 2026)

Qualified targets in 25.02:
- `aarch64-unknown-none` — bare-metal AArch64 (ARM Cortex-A series without OS)
- `x86_64-unknown-linux-gnu` — Linux x86-64 (host development and some embedded Linux applications)
- `thumbv7em-none-eabi` — ARM Cortex-M4/M7, the dominant MCU family in automotive ECUs
- `aarch64-unknown-linux-gnu` — Linux AArch64

### The Ferrocene Language Specification (FLS)

The **Ferrocene Language Specification** ([https://spec.ferrocene.dev](https://spec.ferrocene.dev)) is a formally structured, standalone specification of the Rust language semantics. It is distinct from the official [Rust Reference](https://doc.rust-lang.org/reference/) — the Reference is a community document with informal prose; the FLS is an engineering artifact structured as numbered rules with normative language ("shall", "is defined as") that can be directly mapped to test cases.

The FLS covers:
- Lexical structure (tokenization, string literal escaping, numeric literal syntax)
- Items (functions, structs, enums, traits, impls, `extern` blocks, `use` declarations)
- Expressions and their evaluation order guarantees
- Type system rules (trait bounds, lifetime constraints, coercions, type inference obligations)
- Memory model (ownership, borrowing, interior mutability guarantees from `UnsafeCell`)
- Panic behavior and unwinding semantics
- Foreign function interface obligations

Every FLS rule is numbered (e.g., `FLS-§0023`: *"A place expression can be used in a value context, in which case the value located at the place is moved or copied into the expression's value"*). These numbers are the linking mechanism between the specification and the test suite.

### FerreQual Qualification Test Suite

The FerreQual test suite maps FLS rules to compiler behaviors through a three-step chain:

```
FLS rule number → FerreQual test case ID → compiler output → conformance verdict
```

A test case is a minimal Rust program that exercises exactly one FLS rule, with a documented expected output (either a specific `rustc` diagnostic or a specific assembly pattern). The qualification report records, for each FLS rule, the set of test cases that cover it, the test execution results for each qualified target, and the conformance verdict.

```rust
// FerreQual TC-0023-01: place expression used in value context triggers copy for Copy types
// FLS: FLS-§0023
// Expected: integer value is copied, not moved; original binding remains valid

fn main() {
    let x: i32 = 42;  // i32 implements Copy
    let y = x;        // value context: x is copied
    let _ = x;        // original x still accessible; would be compile error for non-Copy
    assert_eq!(y, 42);
}
```

The test suite is not public (it is a commercial qualification artifact), but the FLS itself is open source under CC-BY 4.0 at [https://github.com/ferrocene/specification](https://github.com/ferrocene/specification).

The FerreQual coverage model distinguishes three test categories:

| Category | Purpose | Example |
|----------|---------|---------|
| Conformance tests | Verify that a valid program compiles and produces the correct output | Integer overflow wraps with `-C overflow-checks=true` |
| Rejection tests | Verify that an invalid program is rejected with the correct diagnostic | Missing lifetime annotation rejected with `E0106` |
| Codegen tests | Verify that the generated assembly matches the expected pattern on a target | `thumbv7em` Cortex-M4 FPU instructions emitted for `f32` arithmetic |

Rejection and conformance tests together provide evidence that the Rust type system rules, borrow checker, and lifetime system behave as specified by the FLS. Codegen tests provide target-specific evidence and are executed on each qualified target separately, since the `thumbv7em-none-eabi` assembly patterns differ from `x86_64` patterns.

### Installing and Using Ferrocene

Ferrocene is distributed through `criticalup`, the Ferrous Systems package manager for safety-qualified toolchains (similar in concept to `rustup` but with cryptographic supply-chain guarantees and version pinning for qualification):

```bash
# Install criticalup (the Ferrocene distribution manager)
curl --proto '=https' --tlsv1.2 -LsSf \
    https://github.com/ferrocene/criticalup/releases/latest/download/criticalup-installer.sh \
    | sh

# Authenticate and install the qualified toolchain
criticalup auth set --token $CRITICALUP_TOKEN
criticalup install --channel ferrocene-25.02
```

A project using Ferrocene pins the toolchain via a `rust-toolchain.toml` in the project root:

```toml
# rust-toolchain.toml — pins the Ferrocene toolchain for qualification traceability
[toolchain]
channel = "ferrocene-25.02"
targets = ["thumbv7em-none-eabi"]
```

And a `Cargo.toml` for an embedded automotive application:

```toml
[package]
name    = "ecu-brake-controller"
version = "0.1.0"
edition = "2024"

[dependencies]
cortex-m     = "0.7"
cortex-m-rt  = "0.7"

[profile.release]
opt-level = 1        # O1: lower qualification risk than O2/O3; see §195.6
overflow-checks = true
panic = "abort"      # No unwinding in safety-critical bare-metal
```

Compilation for the `thumbv7em-none-eabi` Cortex-M4 ECU target:

```bash
# Build with the qualified toolchain and target
criticalup run -- cargo build \
    --release \
    --target thumbv7em-none-eabi

# Verify that exactly the qualified tool version was used (for traceability)
criticalup run -- rustc --version
# ferrocene 25.02.0 (based on rustc 1.84.0) qualified ISO 26262 ASIL D / IEC 61508 SIL 4
```

### The Safety Argument

The complete safety argument for using Ferrocene in an ASIL D system takes the form:

1. **Language specification**: FLS defines the semantics of every Rust construct accepted by the compiler.
2. **Compiler qualification**: FerreQual demonstrates that Ferrocene implements FLS correctly for the qualified targets and optimization configurations.
3. **TÜV SÜD assessment**: an independent functional safety assessor audited the qualification process and the evidence package.
4. **Tool integration**: the project's safety plan references the Ferrocene certificate, pins the toolchain version, and documents any FLS clauses exercised by the project's source code.

This is the same pattern used by AdaCore for GNAT Pro / DO-178C, and it mirrors the formal verification safety argument in [Chapter 181 — Formal Verification in Practice](../part-26-ecosystem-frontiers/ch181-formal-verification-practice.md). Ferrocene enables Rust in safety-critical systems without abandoning the regulatory framework that aviation, automotive, and industrial standards require.

---

## 195.6 LLVM-Specific Qualification Concerns

### Undefined Behavior Exploitation by the Optimizer

The core risk that LLVM poses to safety-critical builds is **UB exploitation**: LLVM passes legally assume that no undefined behavior occurs in the source program, then use this assumption to delete branches, reorder memory accesses, and remove null checks. In the presence of UB in the source, these transformations are semantics-preserving under the C standard — but the optimized binary can behave differently from what a developer reasoned about at the source level.

Key transformations that carry qualification risk:

**`instcombine` and `nsw`/`nuw` flags**: LLVM IR integer arithmetic can be tagged `nsw` (no signed wrap) or `nuw` (no unsigned wrap). When `clang` emits `add nsw i32 %a, %b`, it asserts that the addition cannot overflow. `instcombine` uses this to delete explicit overflow checks. Consider the C source:

```c
// Safety check: detect wraparound on signed addition
// Types: int a, int b — signed overflow is UB in C
if (a + b < a) {
    handle_overflow();
}
```

At `-O2`, Clang emits approximately:

```llvm
; a + b is signed add — clang promotes with nsw if types allow
%sum = add nsw i32 %a, %b
; The following compare is now dead: nsw means sum >= a always
%cmp = icmp slt i32 %sum, %a
br i1 %cmp, label %overflow, label %continue
```

`instcombine` recognizes that `%sum = add nsw %a, %b` implies `%sum >= %a` (for non-negative values), folds `%cmp` to `false`, and deletes the branch. The safety check vanishes. The fix is `-fwrapv`, which suppresses `nsw` flags on signed arithmetic (making signed overflow defined as two's-complement wraparound), keeping the overflow check alive. Note that `-fwrapv` does not affect unsigned arithmetic — unsigned wraparound is already defined behavior in C, so `nuw` flags arise from separate source patterns unaffected by this flag. See [Chapter 62 — Scalar Optimizations](../part-10-analysis-middle-end/ch62-scalar-optimizations.md) for the full instcombine mechanics.

**TBAA and type punning**: Type-Based Alias Analysis classifies pointer accesses by their LLVM IR type. Two accesses to different types are assumed non-aliasing, enabling reordering. Code that accesses the same memory through pointers of different types (hardware register overlays, union-based serialization common in embedded code) can be reordered past each other, breaking memory-mapped I/O protocols. A concrete embedded pattern at risk:

```c
typedef union {
    uint32_t raw;
    struct { uint16_t lo; uint16_t hi; } fields;
} ControlReg;

volatile ControlReg *reg = (volatile ControlReg *)0x40020000U;
reg->raw = 0;            // write via uint32_t alias
uint16_t lo = reg->fields.lo;  // read via struct alias — may be reordered before write
```

TBAA treats the `uint32_t` write and the `uint16_t` read as non-aliasing and may hoist the read before the write. `-fno-strict-aliasing` disables this: all memory accesses are conservatively treated as potentially aliasing regardless of the IR type annotation.

**`poison`/`undef` propagation**: LLVM's `poison` and `undef` values propagate through operations, allowing passes to replace entire computations with `poison` if any input is poison. A use of an uninitialized variable that reaches a branch condition can cause the branch to be treated as dead. Concretely: the LLVM IR pattern

```llvm
; Clang with -O2 on code that reads an uninitialized local
%uninit = load i32, ptr %local_addr  ; UB: uninitialized
%cond   = icmp sgt i32 %uninit, 0
br i1 %cond, label %safety_check, label %bypass
```

becomes, after SimplifyCFG's UB folding, an unconditional branch to `%bypass` — the safety check is dead. The formal treatment of poison propagation is in [Chapter 170 — Alive2 and Peephole Verification](../part-24-verified-compilation/ch170-alive2-peephole.md).

**Null pointer deletion**: GCC and Clang both optimize `if (ptr != NULL) { *ptr = 0; }` into an unconditional store when `-fdelete-null-pointer-checks` is active, because dereferencing `ptr` is UB if `ptr` is null — so the pointer must be non-null. Embedded systems sometimes use address zero for legitimate hardware registers; the deletion silently corrupts behavior.

### Mandated Compiler Flags for Qualified Builds

A safety-critical LLVM/Clang build must disable the optimizer behaviors that exploit UB:

| Flag | Effect | Qualification Rationale |
|------|--------|------------------------|
| `-fno-strict-aliasing` | Disables TBAA; all memory accesses potentially alias | Allows type-punning without reordering risk |
| `-fwrapv` | Signed integer overflow wraps two's-complement | Disables `nsw`-based overflow check deletion |
| `-fno-delete-null-pointer-checks` | Preserves null pointer guards | Required when address 0 is a valid device register |
| `-O1` (not `-O2`/`-O3`) | Fewer passes; each pass is individually qualifiable | Lower optimization = fewer UB exploitations; narrower test surface |
| `-fstack-protector-strong` | Stack canaries on all at-risk frames | Defense-in-depth against corruption reaching safety functions |
| `-fno-exceptions` (C++) | Disable C++ exception unwinding | Unwinding tables not generated; simpler, smaller binary footprint |
| `-ffreestanding` | No standard library assumptions | Required for bare-metal targets; prevents implicit OS calls |

These flags should be documented in the Tool Operational Requirements as part of the "qualified configuration." A Ferrocene qualification certificate covers specific configurations; building with `-O3` on a project whose certificate was granted for `-O1` takes the project outside the qualified configuration.

### Alive2 as a Qualification Aid

[Alive2](https://github.com/AliveToolkit/alive2) ([Chapter 170 — Alive2 and Peephole Verification](../part-24-verified-compilation/ch170-alive2-peephole.md)) encodes each LLVM peephole rewrite as an SMT query and verifies that the rewrite is semantics-preserving across all inputs. A rewrite that exploits UB — deleting a check that is "only needed when UB occurs" — will fail the Alive2 check because Alive2's semantics include the case where UB does occur and asks whether the transformed program matches the source program's behavior in all non-UB cases.

Running Alive2 over the `instcombine` and `SimplifyCFG` pass transformations is now standard practice in the LLVM project's nightly regression suite. From a qualification perspective, the set of Alive2-verified rewrites constitutes a machine-checked subset of the optimizer that can be more easily argued correct in a TOR. A qualification strategy built on `opt -O1 -passes=instcombine,simplifycfg` with `-fwrapv` and Alive2 verification of those passes is closer to a qualifiable configuration than one using the full `-O3` pipeline.

```bash
# Run Alive2 translation validation on a specific optimization
alive-tv original.ll optimized.ll --disable-undef-input
# Correct: transformation preserves semantics for all defined inputs
# Incorrect: alive-tv prints the counterexample witnessing the semantic difference
```

The `--disable-undef-input` flag restricts Alive2 to non-UB inputs — precisely the semantics relevant to safety-critical code where UB should never occur. A transformation that passes with this flag can be claimed as semantics-preserving for all well-defined programs.

For sanitizer-based qualification testing, [Chapter 110 — User-Space Sanitizers](../part-16-jit-sanitizers/ch110-user-space-sanitizers.md) and [Chapter 111 — HWASan and MTE](../part-16-jit-sanitizers/ch111-hwasan-mte.md) describe how UBSan can detect at runtime the very UB constructs that the optimizer exploits statically — using sanitizer-instrumented builds during development to hunt UB before the final qualified build is produced. A standard safety-critical development workflow uses UBSan instrumentation throughout the integration test phase, then produces the final build with the mandated flags and no instrumentation.

---

## 195.7 MISRA C:2023 and MISRA C++:2023

### Rule Structure

The Motor Industry Software Reliability Association's **MISRA C:2023** (4th edition, [https://misra.org.uk](https://misra.org.uk)) defines 175 rules in three categories:

- **Mandatory**: must be complied with; no permitted deviations. Violations are always findings. Example: Rule 1.3 — *No occurrence of undefined behavior or critical unspecified behavior.*
- **Required**: must be complied with unless a formal deviation is recorded and justified. Example: Rule 21.3 — *The memory allocation and deallocation functions of `<stdlib.h>` shall not be used* (no `malloc`/`free` in safety-critical code).
- **Advisory**: best practice; deviations require documentation but are routinely accepted. Example: Rule 15.5 — *A function should have a single point of exit at the end.*

Key rules most directly relevant to LLVM-compiled code:

| Rule | Category | Substance |
|------|----------|-----------|
| 1.3 | Mandatory | No UB or critical unspecified behavior |
| 10.1 | Required | Operands of essentially Boolean type must not be used with inappropriate operators |
| 14.3 | Required | Controlling expressions shall not be invariant (catches always-true conditions the optimizer may remove) |
| 21.3 | Required | No `malloc`/`free`; no dynamic memory allocation |
| 21.4 | Required | `<setjmp.h>` shall not be used |
| 22.1 | Required | All resources acquired dynamically shall be explicitly released |

MISRA C:2023 replaces MISRA C:2012 with 172 amended and 3 new rules, tightening coverage of C17 features. **MISRA C++:2023** (simultaneously released) extends equivalent guidelines to C++17 and replaces the obsolete MISRA C++:2008 standard.

### Clang Integration

`clang-tidy` covers a significant subset of MISRA rules through the `cert-*`, `bugprone-*`, `hicpp-*`, and `cppcoreguidelines-*` check families. No open-source checker covers all 175 MISRA C:2023 rules; the gap is filled by commercial tools:

| Tool | Vendor | MISRA Coverage |
|------|--------|----------------|
| LDRA Tool Suite | LDRA | Full MISRA C:2023, MISRA C++:2023 |
| Helix QAC | Perforce | Full MISRA C:2023, MISRA C++:2023 |
| Parasoft C/C++test | Parasoft | Full MISRA C:2023 |
| PC-lint Plus | Gimpel Software | Substantial MISRA C:2023 |
| clang-tidy | Open source | Partial (cert-*, bugprone-*) |

**MISRA Compliance:2020** ([https://misra.org.uk/misra-compliance-2020/](https://misra.org.uk/misra-compliance-2020/)) defines the formal compliance workflow: each rule is either complied with, or a deviation is formally raised. A deviation must cite the rule, the code location, the safety argument for why the deviation does not introduce risk, and a permit from the relevant authority. A compliance matrix documents the disposition of every rule in the project.

---

## 195.8 SPARK 2014 and GNATprove

### SPARK as an Analyzable Ada Subset

**SPARK 2014** is an analyzable subset of Ada designed for formal verification. It restricts Ada to eliminate features that resist static analysis:
- No unbounded recursion by default (the `SPARK_Mode` aspect permits explicit termination arguments)
- No aliasing through access (pointer) types in most cases — SPARK's alias analysis proves or flags potential aliasing
- No unchecked type conversions without explicit justification
- No access to uninitialized variables (proved statically, not just checked at runtime)
- No side effects in functions (pure functions in the mathematical sense)

These restrictions make the program amenable to **GNATprove**, the static analysis tool for SPARK, which combines:
- **Flow analysis**: tracks data dependencies between variables; proves that every read is preceded by a write; detects information flow violations. Flow analysis operates on the Ada/SPARK AST and does not require LLVM or any backend; it is sound by construction for the SPARK subset.
- **Proof**: generates verification conditions (VCs) and discharges them using Alt-Ergo, CVC5, and Z3 (the same SMT solver infrastructure discussed in [Chapter 181 — Formal Verification in Practice](../part-26-ecosystem-frontiers/ch181-formal-verification-practice.md)). VCs are generated from subprogram contracts (preconditions, postconditions, loop invariants, type invariants) and from the Ada semantics of arithmetic, array indexing, and record field access.

### SPARK Proof Levels

GNATprove operates at four levels of rigor:

| Level | What is verified | DO-178C credit available |
|-------|-----------------|--------------------------|
| Bronze | No uninitialized data; no aliasing violations | Partial (flow analysis) |
| Silver | No runtime exceptions (no integer overflow, no array bounds violations, no null dereferences) | Structural coverage partial credit |
| Gold | Full functional correctness — all preconditions and postconditions are proved | Replaces some MC/DC structural coverage |
| Platinum | Termination proofs in addition to functional correctness | Maximum formal methods credit per DO-333 |

### Contract Syntax

SPARK pre- and postconditions use Ada aspect syntax:

```ada
-- Saturating increment: verified by GNATprove at Silver level
-- (no overflow possible: precondition ensures X < Integer'Last)
procedure Increment (X : in out Integer)
  with SPARK_Mode,
       Pre  => X < Integer'Last,
       Post => X = X'Old + 1;
```

The `X'Old` attribute in the `Post` condition refers to the value of `X` on entry. GNATprove produces a VC:

```
-- VC for Increment: postcondition
-- Assuming: X_entry < Integer'Last
-- Proving:  X_exit = X_entry + 1
-- Discharged by: Alt-Ergo (trivial arithmetic)
```

A more complex example — array bounds verification at Gold level:

```ada
-- Proved: no out-of-bounds access; returns exact partial sum
function Partial_Sum (A : Integer_Array; N : Natural) return Integer
  with SPARK_Mode,
       Pre  => N <= A'Length and then
               (for all I in A'First .. A'First + N - 1 =>
                  A (I) >= 0 and then A (I) <= Integer'Last / N),
       Post => Partial_Sum'Result >= 0;
```

Loop invariants are required when GNATprove cannot discharge bounds automatically:

```ada
function Partial_Sum (A : Integer_Array; N : Natural) return Integer
  with SPARK_Mode,
       Pre  => N <= A'Length and then
               (for all I in A'First .. A'First + N - 1 =>
                  A (I) >= 0 and then A (I) <= Integer'Last / N),
       Post => Partial_Sum'Result >= 0
is
   Sum : Integer := 0;
begin
   for I in A'First .. A'First + N - 1 loop
      pragma Loop_Invariant (Sum >= 0);
      pragma Loop_Invariant (I - A'First <= N);  -- progress bound
      Sum := Sum + A (I);
   end loop;
   return Sum;
end Partial_Sum;
```

The `pragma Loop_Invariant` assertions are discharged by Z3 at each iteration boundary. GNATprove checks that the invariant holds on entry to the first iteration (base case) and is preserved by the loop body (inductive step), following the same Hoare-style reasoning described in [Chapter 167 — Hoare Logic and Separation Logic](../part-24-verified-compilation/ch167-hoare-separation-logic.md).

### SPARK in Practice

SPARK has been used in production for:
- **NVIDIA CryptographyNG**: key management firmware for GPU security processors, using SPARK for the cryptographic core
- **Airbus**: avionics software at DO-178C Level A (using both GNAT Pro and GNATprove together)
- **BAE Systems / Hawk trainer aircraft**: flight control software components verified with SPARK at DO-178C Level A
- **Capgemini / Thales**: railway interlocking software at EN 50128 SIL 4, using SPARK Gold to replace portions of the mandatory structural coverage test effort

The key practical observation is that many SPARK VCs discharge automatically without user-supplied proof hints (ghost code or lemmas). The Silver proof level — guaranteeing absence of runtime errors — is achievable for well-structured code with modest engineering investment. Gold level (functional correctness) requires writing specifications as contracts but replaces significant testing effort under DO-333's formal methods credit.

**GNATprove invocation** in a DO-178C context:

```bash
# Run GNATprove at Silver level (prove no runtime errors)
gnatprove -P project.gpr --level=2 --mode=silver --report=all

# Promote to Gold (prove functional contracts)
gnatprove -P project.gpr --level=4 --mode=gold \
    --prover=cvc5,z3,altergo --timeout=60 --report=all

# Generate proof report for TAS submission
gnatprove -P project.gpr --report=all --output-dir=proof_evidence/
```

The `--report=all` flag generates a per-subprogram proof summary that maps each VC to its discharge status and the prover that discharged it. This output constitutes part of the DO-330 TVCP evidence when SPARK is used under a DO-333 formal methods credit.

GNATprove documentation: [https://docs.adacore.com/spark2014-docs](https://docs.adacore.com/spark2014-docs).

---

## 195.9 The Verified/Qualified Software Stack

The three pillars of trustworthy systems software are formal verification (mathematical proof of correctness), regulatory qualification (audited evidence of reliability), and architectural isolation (hardware-enforced containment of failures). These three approaches converge in a single deployable stack:

**CompCert** ([Chapter 168 — CompCert](../part-24-verified-compilation/ch168-compcert.md)) closes the C-compiler correctness gap. Its Coq proof guarantees that every transformation preserves source semantics — no optimizer miscompilation, no undefined-behavior exploitation, no silent data corruption. CompCert is the C compiler of choice where the trusted base must be mathematically bounded.

**seL4** ([Chapter 186 — Verified Hardware: CHERI Capabilities and the seL4 Microkernel](../part-27-mathematical-foundations/ch186-verified-hardware-cheri-sel4.md)) provides a formally verified OS kernel. The seL4 proof — also in Isabelle/HOL — guarantees integrity, confidentiality, and availability properties for isolated processes. When CompCert compiles the application code and seL4 isolates processes, the trusted base shrinks to the hardware and the two formal proofs.

**Ferrocene** closes the Rust-compiler gap. Where CompCert handles C, Ferrocene brings the same confidence — through qualification rather than formal proof — to the Rust language. For systems where Rust's ownership discipline prevents whole classes of memory-safety bugs, Ferrocene enables writing safety-critical components in Rust without abandoning certification requirements.

**SPARK/GNATprove** covers the Ada layer with functional proof — bridging regulation and verification: SPARK Silver is equivalent to formal absence of runtime errors, SPARK Gold constitutes the strongest available evidence for DO-178C structural coverage objectives, recognized under DO-333.

Together, this stack covers four distinct trust concerns:

| Concern | Solution | Trust Basis |
|---------|---------|-------------|
| OS provides isolation | seL4 | Isabelle/HOL proof of integrity, confidentiality, availability |
| C compiler correct | CompCert | Coq proof of semantic preservation for all inputs |
| Rust compiler trustworthy | Ferrocene | ISO 26262 ASIL D / IEC 61508 SIL 4 qualification, TÜV SÜD assessed |
| Application logic correct | SPARK GNATprove Gold | Z3/CVC5-discharged VCs covering all contracts |

The stack has a natural layering in which each layer's correctness argument depends on layers below it. seL4's proof assumes correct hardware behavior — CHERI capability hardware ([Chapter 186 — Verified Hardware: CHERI Capabilities and the seL4 Microkernel](../part-27-mathematical-foundations/ch186-verified-hardware-cheri-sel4.md)) extends the trusted base down to the hardware capability mechanism. CompCert's proof assumes the OCaml extraction and the GCC/C runtime it links against are correct — a gap acknowledged in the CompCert literature. Ferrocene's qualification assumes that the FerreQual test suite is comprehensive enough to reveal any systematic deviation from the FLS — an empirical rather than mathematical claim. SPARK Gold proofs assume that the SMT solvers (Z3, CVC5, Alt-Ergo) are sound — a gap addressed by the fact that these solvers produce independently checkable certificates.

No single layer eliminates all risk. The value of the combined stack is that each layer's residual risks are *different in character* from the others' — an attacker or a systematic defect that defeats CompCert's semantics is unlikely to also defeat seL4's isolation, Ferrocene's qualification, and SPARK's proofs simultaneously. Defense-in-depth across the trust argument is as important as defense-in-depth in the architecture itself.

---

## 195.10 Standards Comparison

| Standard | Domain | Level System | Tool Qualification | Key Compiler Flags |
|----------|--------|-------------|-------------------|-------------------|
| DO-178C + DO-330 | Civil avionics | Level A–E (A=catastrophic) | TQL-1–5 by tool criterion; Level A Criterion 1 → TQL-1 | `-O1 -fno-strict-aliasing -fwrapv -fno-delete-null-pointer-checks` |
| ISO 26262:2018 | Road vehicles | ASIL A–D + QM (D=highest) | TCL 1–3 via TI×TD matrix; ASIL D compiler → TCL-3 | Same as above; `-ffreestanding` for ECU bare-metal |
| IEC 61508:2010 | General industrial E/E/PE | SIL 1–4 (4=most demanding) | TCL 1–3 (same TI×TD framework) | Same; PFD/PFH targets drive architecture more than flags |
| EN 50128 | Railway | SIL 0–4 (mirrors 61508) | Derives from IEC 61508 tool requirements | Same; EN 50128 recommends SPARK/formal methods at SIL 3–4 |
| IEC 62304 | Medical device software | Class A/B/C (C=life-supporting) | Tool validation required at Class C | `-O1 -fwrapv`; IEC 62304 focuses on lifecycle more than flags |

---

## Chapter Summary

- **Toolchain qualification** is the regulatory/industrial path to trusting a commercial compiler; it is complementary to formal verification (CompCert, Alive2) rather than a substitute.

- **DO-330 TQL** is determined by the tool's role (Criterion 1/2/3) and the software level it supports. A Criterion 1 compiler for Level A avionics software must satisfy TQL-1, requiring a Tool Qualification Plan, Tool Operational Requirements, Tool Verification Cases, and a Tool Accomplishment Summary.

- **ISO 26262 Part 8** uses the TI × TD matrix to determine TCL (1–3). An ASIL D compiler is almost always TCL-3 — requiring a validation report, qualification test suite, and version control evidence.

- **IEC 61508 SIL 4** is the most demanding general industrial integrity level; its tool requirements parallel ISO 26262's TCL-3. EN 50128 (railway) and IEC 62304 (medical) derive from IEC 61508.

- **Ferrocene** (Ferrous Systems + AdaCore) is the first qualified Rust compiler. Ferrocene 25.02 is certified to ISO 26262 ASIL D and IEC 61508 SIL 4 by TÜV SÜD. Its qualification rests on the Ferrocene Language Specification (FLS) and the FerreQual test suite mapping every FLS rule to observed compiler behavior.

- **LLVM-specific risks** for safety-critical builds: `instcombine` deletes signed overflow checks via `nsw` flags; TBAA reorders memory-mapped I/O accesses via type-punning assumptions; null pointer deletion removes hardware register guards. Mitigation: `-fno-strict-aliasing -fwrapv -fno-delete-null-pointer-checks -O1`.

- **Alive2** ([Chapter 170](../part-24-verified-compilation/ch170-alive2-peephole.md)) provides machine-checked verification of LLVM peephole rewrites; the Alive2-verified pass set is the most qualifiable subset of the LLVM optimizer.

- **MISRA C:2023** (175 rules, Mandatory/Required/Advisory) prohibits dynamic memory allocation and undefined behavior. Commercial checkers (LDRA, Helix QAC, Parasoft) provide full coverage; `clang-tidy` covers a meaningful subset.

- **SPARK 2014 + GNATprove** enables formal proof of Ada code: Silver level eliminates all runtime errors; Gold level proves functional correctness and earns DO-333 credit in place of structural coverage obligations.

- **The complete safety stack** — seL4 (verified OS) + CompCert (verified C compiler) + Ferrocene (qualified Rust compiler) + SPARK (formally verified application logic) — provides defense-in-depth from hardware through the application language, with each layer's trust bounded by either mathematical proof or qualified evidence.

---

*@copyright jreuben11*
