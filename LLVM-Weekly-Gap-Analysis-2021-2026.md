# Gap Analysis Report: LLVM, Clang & MLIR — The Expert Programming Reference
## Based on LLVM Weekly newsletters, 2021–2026

*Generated June 2026. 15 agents, 498 tool uses, ~1.4M tokens.*

---

## Executive Summary

This report synthesizes 58 identified gaps from LLVM Weekly coverage (2021–2026) into a prioritized set of recommendations. After de-duplication (several topics appeared across multiple years with incremental updates), 52 distinct gaps remain. They are grouped into three tiers: new chapters, significant new sections, and minor additions.

---

## Tier 1: New Chapters Recommended

These topics are mature, architecturally significant, and too large to absorb into an existing chapter without displacing other content.

---

### 1. C23 — Clang's Implementation of the C Standard
**Priority: HIGH**

**Gap:** The book covers C++23 in depth (Ch183) but has no coverage of the finalized C23 standard (ISO/IEC 9899:2024), which shipped during the book's coverage window.

**Placement:** New chapter in Part V (Clang Frontend), after Ch40 (C++ Parsing). Suggested title: `ch40b-c23-clang-implementation.md`

**Scope:** Full chapter, ~12 pages. Topics: `_BitInt` arbitrary-width integers, `nullptr`/`nullptr_t` in C, `constexpr` for functions, `typeof`/`typeof_unqual`, `#embed`, fixed-width integer literals, relaxed `va_start`, `-fstrict-flex-arrays`, `-std=c23` flag path through the Clang driver, and interactions with embedded toolchains.

**Priority rationale:** C23 is production-relevant on embedded and systems codebases that use Clang as their C compiler. Omitting it from a book with comprehensive C++ coverage is conspicuous. The `-fstrict-flex-arrays` and `_BitInt` features connect directly to the hardening and LLVM IR type coverage elsewhere.

---

### 2. LoongArch Backend: Architecture, MC Layer, and LLD Integration
**Priority: HIGH**

**Gap:** Ch101 gives LoongArch a single line in a comparison table. By 2024 LoongArch had its own dedicated MC layer, full LLD linker support, TLSDESC TLS, 128/256-bit vector extensions, and RuntimeDyld support — comparable engineering depth to the RISC-V backend.

**Placement:** New chapter in Part XV (Targets), alongside Ch98 (RISC-V Backend Architecture). Suggested title: `ch101b-loongarch-backend.md`

**Scope:** Full chapter, ~12 pages. Topics: LoongArch ISA overview (v1.1), MC layer structure, calling conventions, TLS models (TLSDESC in particular), 128/256-bit LASX/LSX vector support, LLD relaxation of `R_LARCH_ALIGN` relocations, RuntimeDyld integration.

**Priority rationale:** LoongArch is now a tier-1 LLVM architecture used in Chinese domestic hardware (Loongson CPUs). A single-line mention in a survey chapter understates its production importance by 2026. The depth of engineering investment (TLSDESC, LLD, vector ISA) warrants treatment comparable to Ch98.

---

### 3. Scalable Whole-Program Static Analysis with SSAF
**Priority: MEDIUM**

**Gap:** The Scalable Static Analysis Framework (SSAF / clang-ssaf-analyzer) is architecturally separate from the path-sensitive static analyzer (Ch45). It performs whole-program analysis via per-TU summaries aggregated by `clang-ssaf-linker`, analogous to ThinLTO's summary workflow.

**Placement:** New chapter in Part VII (Clang Tooling), after Ch47. Suggested title: `ch47b-ssaf-scalable-analysis.md`

**Scope:** Full chapter, ~12 pages. Topics: SSAF design goals vs. path-sensitive analyzer, `LUSummary` callgraph summarization, `clang-ssaf-linker` aggregation step, JSON serialization format, writing SSAF checkers, integration with build systems.

**Priority rationale:** No existing chapter covers link-time static analysis. The summary-based approach enables analysis of codebases too large for the symbolic executor, filling a distinct architectural gap. Landed in 2026 within the book's target window.

---

## Tier 2: New Sections in Existing Chapters

Significant additions that merit multi-page dedicated subsections within existing chapters.

---

### Security and Hardening

**4. TypeSanitizer (TySan): Type-Confusion Detection**
- **Gap:** Ch110 enumerates all major sanitizers but TySan (shipped Clang 19/20) is absent. Type-confusion detection is distinct from UBSan's `-fsanitize=vptr`.
- **Placement:** Ch110 User-Space Sanitizers
- **Scope:** 3–4 page section. Topics: shadow-memory scheme for type tags, C++ dynamic type tracking, performance overhead model, comparison with UBSan vptr.
- **Priority: HIGH** — type confusion is a common exploit class in browser engines; TySan is production-quality.

**5. alloc-token Sanitizer: Allocator Metadata for Use-After-Free**
- **Gap:** Ch110 covers ASan redzones but not the token-based UAF detection approach. `-fsanitize=alloc-token` with `sanitize_alloc_token` attribute and `__builtin_infer_alloc_token()` (landed Oct/Nov 2025) is a distinct instrumentation class.
- **Placement:** Ch110 User-Space Sanitizers
- **Scope:** 2–3 page section.
- **Priority: HIGH** — complements TySan; both ship in Clang 20/21.

**6. BOLT Security Extensions: Gadget Scanner and PAC-RET Hardening**
- **Gap:** Ch118 describes BOLT's optimization pipeline (reordering, ICF, PGO stacking) but not its hardening capabilities. The 2025 gadget scanner and AArch64 pac-ret hardening represent a qualitatively different BOLT use case.
- **Placement:** Ch118 BOLT and Post-Link Optimization
- **Scope:** 2–3 page section. Topics: gadget detection algorithm, PAC-RET retroactive hardening, integration with distribution hardening pipelines (Linux distros, Android).
- **Priority: HIGH** — post-link security hardening is a production workflow; Ch118 currently has no security coverage.

**7. ptrtoaddr Instruction (Provenance-Preserving Address Extraction)**
- **Gap:** Ch19 covers `ptrtoint`/`inttoptr` and pointer provenance but not the August 2025 `ptrtoaddr` instruction, which extracts an integer address without dropping CHERI capability tags.
- **Placement:** Ch19 Instructions I — Arithmetic and Memory
- **Scope:** 1–2 page section alongside existing `ptrtoint` coverage. Must cross-reference Ch186 (CHERI).
- **Priority: HIGH** — correctness-critical for CHERI targets; changes the IR contract for provenance-aware alias analysis.

**8. Pointer Field Protection (llvm.protected.field.ptr)**
- **Gap:** Ch68 covers CFI variants, ShadowCallStack, SafeStack, SLH, but not the 2026 per-field pointer integrity mechanism.
- **Placement:** Ch68 Hardening and Mitigations
- **Scope:** 2 page section. Topics: `llvm.protected.field.ptr` intrinsic, deactivation symbols for link-time disabling, Clang UAF mitigation using pointer field protection.
- **Priority: MEDIUM**

**9. -Wunsafe-buffer-usage and Safe Buffers Analysis**
- **Gap:** Ch68 omits this 2022 diagnostic that is now used in hardened builds. Connects to Safe Buffers / Hardened libc++ effort.
- **Placement:** Ch68 Hardening and Mitigations
- **Scope:** 1–2 page section. Topics: unsafe patterns detected, span-based fix-it suggestions, interaction with `-fbounds-safety`.
- **Priority: MEDIUM**

**10. counted_by Attribute for Flexible Array Members**
- **Gap:** Ch68 discusses FORTIFY_SOURCE and BoundsCheckingPass but not `__attribute__((counted_by(field)))`, which is the mechanism that makes `__builtin_dynamic_object_size` accurate for FAMs.
- **Placement:** Ch68 Hardening and Mitigations
- **Scope:** 1–2 page section. Cross-reference Ch200 (Linux kernel chapter).
- **Priority: MEDIUM**

**11. AArch64 Lightweight Fault Isolation (LFI) — MCLFIRewriter**
- **Gap:** Ch96 covers PAuth, BTI, CFI but not the 2026 sandboxing mechanism at the MC layer.
- **Placement:** Ch96 The AArch64 Backend
- **Scope:** 2 page section. Topics: LFI instruction-rewriting model, comparison with WebAssembly isolation, MCLFIRewriter design.
- **Priority: MEDIUM**

---

### IR and Type System

**12. ConstraintElimination Pass**
- **Gap:** Ch62's scalar pass catalogue omits `ConstraintElimPass`, enabled by default at `-O2` since early 2023. It performs bounds-check elimination using linear arithmetic constraint graphs.
- **Placement:** Ch62 Scalar Optimizations
- **Scope:** 2–3 page section. Topics: constraint graph construction from branch conditions, integer arithmetic reasoning, interaction with ASan/UBSan instrumented code.
- **Priority: HIGH** — default-enabled at `-O2`; any reader examining `-print-pipeline` output will encounter it.

**13. Convergence Control Intrinsics**
- **Gap:** Ch24 has only a type-hierarchy mention of `ConvergenceControlInst`. The formal convergence IR (`llvm.experimental.convergence.entry/loop/anchor`) is essential for correct GPU barrier representation.
- **Placement:** Ch24 Intrinsics
- **Scope:** 3 page section. Topics: convergent region semantics, motivation (why `convergent` attribute was insufficient), the three intrinsics, GlobalISel opcodes, interaction with Ch102/103/104 (GPU chapters).
- **Priority: HIGH** — correctness-critical for GPU targets; prerequisite for understanding AMDGPU, NVPTX, SPIR-V chapters.

**14. memory(...) Attribute Unification**
- **Gap:** Ch23 describes the pre-LLVM-15 scattered attributes (`readnone`, `readonly`, `writeonly`, `argmemonly`, `inaccessiblememonly`) without explaining the unified `memory(...)` replacement. Current LLVM IR will confuse readers who see only old coverage.
- **Placement:** Ch23 Attributes, Calling Conventions, and the ABI
- **Scope:** 2 page section. Topics: `memory(argmem: read, inaccessiblemem: write)` syntax, per-location encoding, migration from old attributes.
- **Priority: HIGH** — the book's LLVM 22 target means all IR examples should use the current attribute model.

**15. BranchInst Split into UncondBr / CondBr**
- **Gap:** Ch20 covers `BranchInst` API. The 2026 split into separate `UncondBr`/`CondBr` opcodes with `BranchInst` deprecated is a breaking API change affecting every pass that creates or matches branches.
- **Placement:** Ch20 Instructions II — Control Flow and Aggregates
- **Scope:** 2 page section. Topics: new opcode model, C++ API migration, pattern-matching implications, cross-references to Ch60 (Writing a Pass) and Ch62.
- **Priority: MEDIUM** — breaking change within the book's LLVM 22 target window.

**16. FP4/FP6 (OCP MX) Types in APFloat and the LLVM Type System**
- **Gap:** Ch17 enumerates `bfloat`, `fp128`, `x86_fp80`, `ppc_fp128` but omits the 2024 ML-quantization types (FP6 E2M3/E3M2, FP4 E2M1) now part of APFloat.
- **Placement:** Ch17 The Type System
- **Scope:** 1–2 page section. Topics: OCP MX specification, APFloat encoding, flow through LLVM IR into NVPTX/AMDGPU backends.
- **Priority: MEDIUM** — increasingly relevant for quantized-inference compilation.

**17. LLVM Byte Type (Raw Memory Type)**
- **Gap:** Ch17 covers opaque pointers and target extension types but not the 2026 byte type proposal (initial implementation landed), which is positioned to eventually replace `undef` in memory operations.
- **Placement:** Ch17 The Type System
- **Scope:** 1 page section, cross-referencing Ch171 (undef/poison/freeze).
- **Priority: MEDIUM**

**18. Assignment Tracking Debug Info (llvm.dbg.assign)**
- **Gap:** Ch22 covers MD_* metadata families and the RemoveDIs format but not the 2022 assignment tracking infrastructure (`DIAssignID`, `llvm.dbg.assign`, `AssignmentTrackingAnalysis`), now the default for `-g` builds.
- **Placement:** Ch22 Metadata and Debug Info
- **Scope:** 2–3 page section. Topics: semantic shift from instruction-order to assignment-semantic tracking, interaction with optimization passes that modify stores, `DIAssignID` attachment.
- **Priority: MEDIUM** — any pass author modifying stores must understand this.

**19. llvm.is.fpclass Intrinsic and FP Mode Control Intrinsics**
- **Gap:** Ch24's constrained FP section does not catalogue `llvm.is.fpclass` (replacing ad-hoc isnan/isinf patterns) or `llvm.get/set/reset.fpmode`. Note: the 2022 and 2023 gap entries describe the same intrinsic; merged here.
- **Placement:** Ch24 Intrinsics
- **Scope:** 2 page section. Topics: bitmask encoding of FP classes, targets with native fpclass instructions (AArch64 FCCMP, x86 VFPCLASSPS), fpmode intrinsics for save/restore of FP environment.
- **Priority: MEDIUM**

**20. New 2026 IR Intrinsics (llvm.clmul, llvm.structured.gep/alloc, llvm.cond.loop, llvm.looptrap)**
- **Gap:** Ch24 does not cover these five 2026 additions spanning cryptographic, memory-safety-constrained, and loop-control categories.
- **Placement:** Ch24 Intrinsics
- **Scope:** 3 page section across the five intrinsics, grouped by category.
- **Priority: MEDIUM**

**21. IROutliner and IR Similarity Analysis**
- **Gap:** Ch92 covers only the post-RA `MachineOutliner`. The IR-level `IROutliner` pass (pre-RA, using `IRSimilarityIdentifier`) is a distinct code-size reduction pass.
- **Placement:** Ch92 The Machine Outliner
- **Scope:** 2–3 page section. Topics: `IRSimilarityIdentifier` suffix-tree algorithm, single-entry/single-exit region support, cost model, comparison with `MachineOutliner` (when each applies).
- **Priority: MEDIUM**

---

### Profiling and Optimization

**22. MemProf — Context-Sensitive Heap Profiling**
- **Gap:** Ch67 covers instrumentation PGO, SamplePGO, CSSPGO, BOLT, but MemProf is entirely absent. The 2022–2025 additions (IR metadata, profile matching, YAML deserialization, `-ftemporal-profile`, data layout profiling) represent a mature, distinct profiling mode.
- **Placement:** Ch67 Profile-Guided Optimization
- **Scope:** 3–4 page section. Topics: heap-use context sensitivity vs. frequency profiling, `-fmemory-profile`, `llvm-profdata` integration, cold-heap removal, allocation-site annotation in IR, production use at Google.
- **Priority: HIGH** — distinct profiling mode with production deployment; used to drive data layout optimization.

**23. Temporal Profiling for IRPGO (-ftemporal-profile)**
- **Gap:** Ch67 covers frequency/count profiling but not temporal ordering. `-ftemporal-profile` (January 2025) enables co-execution clustering for instruction-cache locality.
- **Placement:** Ch67 Profile-Guided Optimization (subsection within or alongside MemProf)
- **Scope:** 1–2 page subsection.
- **Priority: LOW**

**24. Profi Flow-Based Profile Inference Algorithm**
- **Gap:** Ch67 does not describe how LLVM handles missing/inconsistent counters. Profi (2021) solves this via max-flow over the CFG.
- **Placement:** Ch67 Profile-Guided Optimization
- **Scope:** 1–2 page section.
- **Priority: LOW**

**25. LLD Profile-Guided Function Ordering**
- **Gap:** Ch78 covers LLD internals without addressing its 2024 profile-driven layout capability. Distinct from BOLT (post-link) and PGO (mid-end).
- **Placement:** Ch78 The LLVM Linker (LLD)
- **Scope:** 2 page section. Topics: perf/sample profile input, hot-function grouping algorithm, startup page-fault reduction.
- **Priority: MEDIUM**

**26. LLVM Sandbox Vectorizer**
- **Gap:** Ch64 covers Loop Vectorizer, VPlan, SLP Vectorizer, but not the experimental Sandbox Vectorizer infrastructure (LLVM 19/20+).
- **Placement:** Ch64 Vectorization Deep Dive
- **Scope:** 2 page section. Topics: design goals (IR sandbox isolation), relationship to SLP/Loop vectorizers, how to use it for experimentation.
- **Priority: MEDIUM**

---

### Debug Infrastructure

**27. Clang Lifetime Safety Analysis (production graduation)**
- **Gap:** Ch45 covers the ExplodedGraph-based analyzer but not the lifetime safety analysis, which graduated from experimental in 2026 with five new checkers and its own documentation.
- **Placement:** Ch45 The Static Analyzer
- **Scope:** 3 page section. Topics: use-after-return detection, `[[clang::lifetimebound]]` suggestion automation, non-trivially destructed temporaries, field-reference-to-out-of-scope-local.
- **Priority: HIGH** — production-graduated in the book's target window; has dedicated user documentation.

**28. Clang FlowSensitive Dataflow Analysis Framework**
- **Gap:** Ch45 covers only the path-sensitive ExplodedGraph engine. The `clang/Analysis/FlowSensitive/` framework (2021, extended 2022 with SAT solver, widening API) is a separate infrastructure for lattice-based CFG-level analyses.
- **Placement:** Ch45 The Static Analyzer
- **Scope:** 3–4 page section. Topics: `DataflowAnalysisContext` API, transfer-function lattice model, SAT-solver constraint propagation, `noreturn` destructor support, comparison with path-sensitive engine.
- **Priority: MEDIUM**

**29. debuginfod and LLVM HTTP Client**
- **Gap:** Ch117 covers DWARF emission and split DWARF but not the network-retrieval side (debuginfod protocol, `llvm/Support` HTTP client with libcurl).
- **Placement:** Ch117 DWARF and Debug Info
- **Scope:** 2 page section.
- **Priority: LOW**

**30. SFrame Unwind Format**
- **Gap:** Ch117 covers DWARF 5, split DWARF, DWP, CodeView but not the July/September 2025 SFrame format.
- **Placement:** Ch117 DWARF and Debug Info
- **Scope:** 2 page section. Topics: SFrame binary format, comparison with `.eh_frame`, targeted use cases.
- **Priority: MEDIUM**

**31. Windows Secure Hot-Patching via CodeView**
- **Gap:** Ch117 covers CodeView as a debug format but not its use for live hot-patching (June 2025).
- **Placement:** Ch117 DWARF and Debug Info
- **Scope:** 2 page section.
- **Priority: MEDIUM**

**32. DWARFLinkerParallel**
- **Gap:** Ch117 treats DWARF linking as sequential. The 2023 parallel DWARF linker is relevant for build-time performance.
- **Placement:** Ch117 DWARF and Debug Info
- **Scope:** 1 page subsection.
- **Priority: LOW**

**33. Intel Processor Trace (IPT) in LLDB**
- **Gap:** Ch116 covers process plugins but not hardware execution tracing via IPT (2022).
- **Placement:** Ch116 LLDB Architecture
- **Scope:** 2–3 page section. Topics: IPT ring-buffer model, LLDB integration for execution replay, comparison with XRay sampling.
- **Priority: MEDIUM**

**34. LLDB Symbol Plugins: SymbolFileJSON and CTF**
- **Gap:** Ch116's SymbolFile plugin table is incomplete; both were added in 2023.
- **Placement:** Ch116 LLDB Architecture
- **Scope:** 1 page addition to the existing plugin table.
- **Priority: LOW**

**35. LLDB GPU Hardware Accelerator Debugging**
- **Gap:** Ch116 has no coverage of the 2026 GPU process plugin category.
- **Placement:** Ch116 LLDB Architecture
- **Scope:** 2 page section.
- **Priority: MEDIUM**

**36. llubi: UB-Aware LLVM IR Interpreter**
- **Gap:** Ch170 focuses on Alive2 (static Z3-based); llubi (2026) occupies the distinct niche of concrete IR interpretation with UB tracking.
- **Placement:** Ch170 Alive2 and Translation Validation
- **Scope:** 2 page section. Topics: GEP out-of-bounds, invalid `inttoptr`, uninitialized memory via lifetime markers, comparison with Alive2 (static) and KLEE (symbolic).
- **Priority: MEDIUM**

---

### LTO, Linking, and Object Formats

**37. Unified LTO**
- **Gap:** Ch77 describes the full-vs-thin LTO choice as a compile-time binary decision, which is outdated since UnifiedLTO (LLVM 17+) allows the same bitcode to serve both workflows.
- **Placement:** Ch77 LTO and ThinLTO
- **Scope:** 2 page section. Topics: dual-summary bitcode format, `-funified-lto` flag, build system design implications.
- **Priority: MEDIUM**

**38. lld-macho Backend Specifics**
- **Gap:** Ch78 lists the MachO driver but devotes no subsections to macOS-specific behaviors (two-level namespace, weak dylib linking, chained fixups, arm64 ABI nuances).
- **Placement:** Ch78 The LLVM Linker (LLD)
- **Scope:** 3 page section.
- **Priority: LOW**

**39. CREL Compact Relocation Format**
- **Gap:** Ch94 covers `MCStreamer`, relocations and fixups but not the 2024 CREL format.
- **Placement:** Ch94 The MC Layer and MIR Test Infrastructure
- **Scope:** 1 page subsection.
- **Priority: LOW**

---

### GPU and Offloading

**40. Clang Linker Wrapper and Unified LLVM GPU Offload API**
- **Gap:** Ch28 covers the Clang driver and offload bundling but not `clang-linker-wrapper` (2022) or `clang-nvlink-wrapper` + `-foffload-via-llvm` (2024).
- **Placement:** Ch28 The Clang Driver (linker wrapper) and Ch48 Clang as a CUDA Compiler (Offload API pathway)
- **Scope:** 2 page section in Ch28; 2 page section in Ch48.
- **Priority: MEDIUM**

**41. MLIR OpenMP GPU Target Offload**
- **Gap:** Ch127 covers MLIR OpenMP for CPU and OpenACC GPU path but not the `omp.target`-to-GPU lowering pipeline, significantly redesigned in 2024 around `OpenMP_Clause`.
- **Placement:** Ch127 Flang OpenMP and OpenACC
- **Scope:** 3 page section.
- **Priority: MEDIUM**

---

### Architecture-Specific

**42. ARM64EC and ARM64X Object Format**
- **Gap:** Ch96 covers Apple variants, SME, PAuth, BTI but not the Windows ABI for incremental AArch64 migration (ARM64EC/ARM64X), with COFF backend and LLD support (2023).
- **Placement:** Ch96 The AArch64 Backend
- **Scope:** 2–3 page section.
- **Priority: MEDIUM**

**43. Armv8.1-M PACBTI Extension**
- **Gap:** Ch97 covers A32/Thumb-2 without PACBTI. Now standard for Cortex-M85 and automotive/IoT firmware.
- **Placement:** Ch97 The 32-bit ARM Backend
- **Scope:** 2 page section.
- **Priority: MEDIUM**

**44. RISC-V CHERI (RVY) Backend Support**
- **Gap:** Ch186 covers CHERI conceptually; Ch98 covers upstream RISC-V. The 2026 MC layer upstreaming of RVY creates a gap between the two.
- **Placement:** Ch98 The RISC-V Backend Architecture (or Ch100 RISC-V Extensions)
- **Scope:** 2 page section.
- **Priority: MEDIUM**

**45. WebAssembly GlobalISel, PIC Graduation, and Component Model Threading**
- **Gap:** Ch106 covers existing Wasm backend but not the 2026 GlobalISel skeleton, PIC/dynamic linking stabilization, or LLD Component Model cooperative multithreading.
- **Placement:** Ch106 WebAssembly and BPF
- **Scope:** 2–3 page section.
- **Priority: MEDIUM**

**46. Windows ARM SEH in Clang**
- **Gap:** Ch43 covers MS SEH-driven cleanup for x86/x64 but not AArch64 Windows SEH support (2026).
- **Placement:** Ch43 C++ ABI Lowering: Microsoft
- **Scope:** 1–2 page section.
- **Priority: MEDIUM**

---

### Dialects and MLIR Infrastructure

**47. MLIR Extensible Dialects (IRDL / Dynamic Dialect Extension)**
- **Gap:** Ch132 covers ODS-based static dialect definition but not the 2022 runtime dialect extensibility mechanism (`IRDL` dialect, `DynamicDialect`, `DynamicOperation`/`DynamicType`).
- **Placement:** Ch132 Defining Dialects with ODS
- **Scope:** 2–3 page section.
- **Priority: MEDIUM**

**48. MLIR MPI Dialect and Mesh-to-MPI Lowering**
- **Gap:** Ch146 covers OpenMP and OpenACC dialects but not the MPI dialect (December 2024/February 2025) or `ConvertMeshToMPIPass`.
- **Placement:** Ch146 Async, OpenMP, OpenACC, DLTI, EmitC
- **Scope:** 2–3 page section.
- **Priority: MEDIUM**

**49. WasmSSA MLIR Dialect**
- **Gap:** The August 2025 upstream WasmSSA dialect is missing. No existing chapter covers the MLIR-native AOT-Wasm compilation path.
- **Placement:** Ch146 Async, OpenMP, OpenACC, DLTI, EmitC
- **Scope:** 2 page section.
- **Priority: MEDIUM**

---

### Flang

**50. Fortran 2023 Language Support in Flang**
- **Gap:** Ch125 covers the Fortran 90/95/2003 baseline but not the systematic Fortran 2023 compliance effort landing throughout 2026.
- **Placement:** Ch125 Flang Architecture and Driver
- **Scope:** 2–3 page section enumerating Fortran 2023 feature status at LLVM 22 target.
- **Priority: LOW**

---

## Tier 3: Minor Additions

**51. Significant Deprecations and Removals (Correctness Fixes)**
- **Priority: HIGH** — accuracy issues in existing chapters.
- `bugpoint` deleted: Ch173 must remove it and note `llvm-reduce` as the replacement.
- `llvm.convert.to/from.fp16` removed: Ch24 must remove coverage.
- `Os`/`Oz` deprecation: Ch59 should note these are deprecated in favor of `-O2 -optsize` in LLVM 22.
- x86 profile-guided prefetch removal: note in Ch95.
- OpenMP standalone build removal: note in Ch124.

**52. Minor Catalogue Additions**

| Topic | Placement | Note |
|---|---|---|
| GraalVM calling convention (`CallingConv::GRAAL`) | Ch23 calling conventions table | One-row addition |
| `preserve_none` calling convention | Ch23 calling conventions table | Completes the preserve_all/most/none trilogy |
| VE (NEC SX-Aurora) target | Ch101 survey chapter | 256-element vector registers, HPC use |
| SystemZ z/OS and GOFF object format | Ch101 survey chapter | z/OS triple, GOFF format basics, EBCDIC |
| `clang-pseudo` robust C++ parser | Ch47 tooling chapter | Brief mention; still experimental |
| Trojan Source detection (`misc-misleading-bidirectional`) | Ch47 clang-tidy section | Security checker example |
| Polly tensor contraction pattern matching | Ch76 Polly in Practice | Narrow addition to "what Polly detects" section |
| `ConstantFPRange` for FP range analysis | Ch198 Value Tracking Infrastructure | FP analogue to ConstantRange |
| Machine Pipeliner window scheduling algorithm | Ch91 The Machine Pipeliner | SMS vs window scheduling trade-offs |
| DXIL analysis pass infrastructure | Ch105 DXIL and DirectX Shader Compilation | DXILMetadataAnalysis, DXILResourceAnalysis |
| `NumericalStabilitySanitizer` (NSan) | Ch110 User-Space Sanitizers | FP instability detection, shadow-memory scheme |
| LLDB SymbolFileJSON and CTF | Ch116 LLDB Architecture | Complete the symbol plugin table |
| `-fwrapv-pointer` pointer overflow semantics | Ch19 Instructions I | 1-paragraph note on separate UB control for pointer overflow |
| Strict FP RFC / nofpclass-on-loads | Ch24 Intrinsics | Note trajectory toward inline fpenv metadata |

---

## Implementation Priority Order

**Immediate (correctness and accuracy):**
1. Remove `bugpoint` from Ch173, update `llvm.convert.to/from.fp16` in Ch24, note `Os`/`Oz` deprecation in Ch59
2. Add `memory(...)` attribute to Ch23 — current IR examples are stale
3. Add `BranchInst` split coverage to Ch20 — breaking API change

**High priority (major gaps in production toolchain coverage):**
4. MemProf section in Ch67
5. ConstraintElimination pass in Ch62
6. Convergence control intrinsics in Ch24
7. TypeSanitizer in Ch110
8. Clang Lifetime Safety Analysis in Ch45
9. `ptrtoaddr` in Ch19
10. BOLT security extensions in Ch118

**New chapters (plan as standalone writing tasks):**
11. C23 chapter
12. LoongArch chapter
13. SSAF chapter

**Remaining sections** can be scheduled in any order consistent with the book's parallelism rules defined in CLAUDE.md.

---

## Coverage Statistics

- **58** raw gap entries identified across 2021–2026
- **6** de-duplicated (MemProf, FlowSensitive, IROutliner, llvm.is.fpclass, linker wrapper, temporal profiling)
- **52** distinct recommendations: 3 new chapters, 38 new sections, 11 minor additions
- **Chapters most affected:** Ch24 (Intrinsics) — 5 additions; Ch117 (DWARF) — 4 additions; Ch68 (Hardening) — 4 additions; Ch67 (PGO) — 4 additions; Ch110 (Sanitizers) — 4 additions; Ch116 (LLDB) — 4 additions
