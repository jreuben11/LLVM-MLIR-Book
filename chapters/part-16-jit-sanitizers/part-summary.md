# Part XVI — JIT, Sanitizers & Runtime Tooling — Part Summary

*This part covers LLVM's complete runtime execution, safety, and observability stack — the machinery that runs compiled code, finds bugs in it, and makes it observable in production — spanning JIT execution engines, hardware-assisted and software memory safety tools, kernel-level sanitizers, coverage and fuzzing infrastructure, debugging and profiling, and post-link optimization.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 108 | The ORC JIT | ORC v2 architecture: sessions, layers, lazy compilation, multi-process JIT |
| 109 | JITLink | In-process linker: LinkGraph, GOT/PLT synthesis, plugin architecture |
| 110 | User-Space Sanitizers | ASan, MSan, TSan, UBSan, LSan, DFSan, TySan, NSan shadow memory models |
| 111 | HWASan and MTE | AArch64 top-byte-ignore pointer tagging, hardware memory safety at production overhead |
| 112 | Production Allocators: Scudo and GWP-ASan | Hardened allocator design, probabilistic guard-page UAF detection |
| 113 | Kernel Sanitizers | KASAN, KFENCE, KCSAN, KMSAN, kernel UBSan, CFI in privileged context |
| 114 | LibFuzzer and Coverage-Guided Fuzzing | In-process grey-box fuzzing, SanitizerCoverage, OSS-Fuzz integration |
| 115 | Source-Based Code Coverage | Region/branch/MC/DC coverage with llvm-profdata and llvm-cov |
| 116 | LLDB Architecture | Library-first debugger: plugins, Clang expression evaluator, GDB RSP |
| 117 | DWARF and Debug Info | DWARF 5 DIE hierarchy, location expressions, CFI, split DWARF/DWP |
| 118 | BOLT and Post-Link Optimization | Profile-driven binary rewriting: HFSort, basic-block reordering, ICF |
| 219 | ORC JIT in Production | Five production deployments: clang-repl, LLDB, PostgreSQL, Numba, Halide |
| 220 | Runtime Self-Modification | LibTooling introspection, IncrementalCompiler, ORC hot-loading, fleet propagation |
| 221 | Speculative Optimization, Inline Caches, and Deoptimization | Guards, stackmaps, OSR, V8/HotSpot tiering, ReOptimizeLayer |
| 222 | Plugin Architecture, Dynamic Loading, and ABI Stability | dlopen/dlsym, soname versioning, C++ plugin boundary rules, llvm::sys::DynamicLibrary |
| 228 | Virtual Machine Design | Bytecode interpreters, GC strategies, object models, ORC-connected tiering |
| 232 | LLVM XRay: Low-Overhead Function Tracing | NOP-sled patching, flight recorder, production tracing patterns |

## Part Overview

Part XVI addresses the full runtime lifecycle of compiled programs: execution, safety verification, observability, and continuous improvement. It opens with LLVM's third-generation JIT engine, ORC v2 (Chapters 108–109), establishing the machinery that transforms LLVM IR into callable machine code at runtime. ORC's design — sessions, JITDylibs, materialization units, a layered compilation stack, lazy per-function compilation, and `ExecutorProcessControl` for multi-process separation — provides the execution substrate that all subsequent runtime tooling depends on. JITLink (Chapter 109) replaces the aging RuntimeDyld with a graph-based linker that is concurrently safe, W^X compliant, and extensible through a clean plugin architecture, handling ELF, MachO, and COFF objects across x86_64, AArch64, and RISC-V.

The sanitizer cluster (Chapters 110–113) covers the full spectrum from development-time precision to production-viable monitoring. User-space sanitizers (Chapter 110) represent five distinct shadow memory models — ASan's 1:8 byte encoding, MSan's 1:1 bit propagation, TSan's vector-clock shadow cells, UBSan's shadowless inline branches, and DFSan's taint-tracking labels — each paired with a `compiler-rt` runtime. HWASan and MTE (Chapter 111) move safety checking into hardware: AArch64's Top Byte Ignore enables 8-bit or 4-bit pointer tags that catch UAF and OOB at 10–15% or 2–5% overhead respectively, making always-on production memory safety viable for the first time. Production allocators (Chapter 112) address the threat model that sanitizers miss: Scudo's hardened heap with checksum-protected headers, quarantine, and MTE integration defeats heap exploitation even in deployed binaries, while GWP-ASan's probabilistic guard-page sampling catches bugs at under 1% overhead in consumer applications. Kernel sanitizers (Chapter 113) apply analogous techniques in privileged context — KASAN, KFENCE, KCSAN, KMSAN, and kernel CFI must work without signals, without standard allocators, and in interrupt context, making their design substantially harder than their userspace counterparts.

The observability arc (Chapters 114–118 and 232) covers how programs are exercised and inspected. LibFuzzer (Chapter 114) combines SanitizerCoverage's feedback bits with a corpus-guided mutation engine to find security bugs automatically; its OSS-Fuzz integration has discovered tens of thousands of bugs in production software. Source-based coverage (Chapter 115) provides precise region- and branch-level measurement with MC/DC for safety-critical compliance, producing the instrumented binary through the same `llvm-profdata`/`llvm-cov` toolchain that drives PGO. LLDB (Chapter 116) is the debugger for the entire LLVM ecosystem: a library-first architecture with an embedded Clang compiler for live expression evaluation via ORC, plugin-based process control, and remote debugging over GDB RSP. DWARF (Chapter 117) is the foundational format underpinning all debugger, unwinder, and symbolizer operation — its DIE hierarchy, location expressions, and call frame information appear in every chapter that produces debug output. BOLT (Chapter 118) and XRay (Chapter 232) bookend the observability arc at the binary level: BOLT reconstructs CFGs from final binaries and uses perf LBR profiles to reorder code for i-cache efficiency, achieving 5–15% speedup on large binaries; XRay instruments entry/exit NOP sleds that cost nothing when inactive and enable nanosecond-resolution function tracing when activated.

The advanced JIT cluster (Chapters 219–222, 228) synthesizes the preceding material into complete production systems. Chapter 219 examines five production ORC deployments — clang-repl, LLDB, PostgreSQL, Numba, Halide, and WAVM — revealing the two-axis taxonomy of all JIT architectures: IR origin (Clang lowering vs. IRBuilder construction) and lifetime model (ephemeral-per-query vs. persistent-and-accumulated). Chapter 220 closes the loop on runtime self-modification: a running process can introspect its own source via LibTooling, recompile changed functions with `IncrementalCompiler`, verify equivalence with Alive2-style SMT checking, and hot-swap the implementation via `RedirectableSymbolManager`, then propagate improvements to peer instances over a fleet protocol. Chapter 221 provides the theoretical grounding for speculative compilation — inline caches, guard IR, deoptimization bundles, stackmap-based frame reconstruction, OSR, and the V8/HotSpot tiering systems — and connects them to ORC's `ReOptimizeLayer`. Chapter 222 covers the orthogonal mechanism of dynamic loading (`dlopen`/`dlsym`, ABI stability, soname versioning) and the C++ plugin boundary rules that govern extension points in production systems. Chapter 228 grounds the entire JIT discussion in bytecode VM design, covering the interpreter architectures (stack vs. register, switch vs. computed-goto dispatch, CPython's copy-and-patch) and GC strategies that every JIT-enabled VM must provide before ORC can accelerate it.

After completing this part, a reader can build a production-quality ORC JIT with lazy compilation, tiered reoptimization, and multi-process execution; design a memory-safe allocator that survives adversarial input; instrument a codebase with the appropriate sanitizer combination for their threat model; integrate libFuzzer with ASan for automated vulnerability discovery; read DWARF section dumps and CFI tables; deploy BOLT and XRay in production for performance monitoring; and understand the speculative optimization and deoptimization machinery that every high-performance dynamic language runtime requires.

## Key Concepts Introduced

- **ORC ExecutionSession / JITDylib**: The root session owns all JIT state; JITDylibs provide symbol namespaces with configurable search order; the session serializes only symbol-state transitions, not compilation itself.
- **MaterializationUnit / MaterializationResponsibility**: Deferred work contracts — a unit declares the symbols it will provide; when the session needs them, it calls `materialize()` and the unit must eventually call `notifyResolved` + `notifyEmitted` or `failMaterialization`.
- **Layer model**: Composable transformations stacked as `IRTransformLayer` → `IRCompileLayer` → `ObjectLinkingLayer`; each layer receives a `ThreadSafeModule` or object file and delegates downward.
- **CompileOnDemandLayer / LazyCallThroughManager / IndirectStubsManager**: Lazy per-function compilation via platform-specific trampolines (in `compiler-rt/lib/orc/`) and dual-mapped stub/pointer-cell pairs that respect W^X hardware constraints.
- **LinkGraph**: JITLink's format-independent intermediate representation — `Section`→`Block`→`Edge` (relocations) + `Symbol`s; enables concurrent, plugin-extensible linking across ELF/MachO/COFF with GOT relaxation and range-extension thunk synthesis.
- **Shadow memory**: The foundational sanitizer technique — a compact parallel mapping (1:8 for ASan, 1:1 bit for MSan, 4-cells per 8-byte granule for TSan) storing per-byte metadata checked on every access.
- **AArch64 Top Byte Ignore (TBI)**: Hardware property that makes bits [63:56] of virtual addresses transparent to the MMU, enabling 8-bit (HWASan) or 4-bit (MTE) pointer tags for near-zero-overhead memory safety.
- **Scudo two-level allocator**: Primary slab allocator with per-CPU caches and out-of-line metadata for size-class allocations, plus mmap-based secondary for large allocations; header checksum and quarantine defeat heap exploitation.
- **GWP-ASan probabilistic guard pages**: Random sampling (typically 1-in-N allocations) routes selected allocations through guard-paged slots; UAF and OOB are caught by SIGSEGV faults with allocation/deallocation stack traces.
- **SanitizerCoverage / libFuzzer feedback loop**: Per-edge coverage bits updated on the JIT path drive corpus selection — inputs exercising new edges are retained; the mutation engine applies bit-flips, insertions, and cross-over mutations preferentially to corpus members.
- **DWARF Call Frame Information (CFI)**: Compact bytecode in `.eh_frame` / `.debug_frame` encoding the CFA (canonical frame address) and register save locations at each program point; consumed by both debuggers (for backtraces) and C++ unwinders (for exception propagation).
- **ReOptimizeLayer / RedirectableSymbolManager**: Tiered recompilation — tier-1 code is instrumented with call counters; `reoptimizeIfCallFrequent` triggers tier-2 compilation at threshold; `redirect(JD, SymbolMap)` atomically patches the pointer cell, redirecting all future calls without halting any CPU thread.
- **BOLT HFSort / basic-block reordering**: Profile-driven function reordering clusters hot functions together to reduce i-cache footprint; basic-block reordering eliminates cold-path fall-through penalties; together these achieve 5–15% throughput improvement on large server binaries.
- **XRay NOP sled**: Compiler inserts a 5-byte NOP sled at each function entry/exit; `__xray_patch()` atomically overwrites the NOP with a `call` to the handler using a lock-prefixed write; zero overhead when unpatched.
- **Speculative guards / deoptimization bundles / stackmaps**: `@llvm.experimental.guard` encodes a speculative assumption as a conditional branch with a deopt bundle; `.llvm_stackmaps` records live-value locations (register/stack offset) for every stackmap ID; the deopt handler reads this metadata to reconstruct interpreter state when a guard fires.

## How This Part Fits the Book

Part XV (Targets, Chapters 100–107) provides the platform-specific backends that produce the machine code ORC and JITLink link and execute; in particular, the AArch64 backend's `AArch64StackTaggingPass` (Chapter 111) and the x86_64 relocation tables that JITLink's `ELF_x86_64.cpp` implements are products of Part XV's target code generators. Parts XIV (Backend, Chapters 91–99) and XIII (LTO, Chapters 85–90) produce the IR and object files that ORC's `IRCompileLayer` and JITLink consume, and BOLT's binary rewriting (Chapter 118) logically completes the work that LTO and PGO begin. Part XVII (Runtime Libraries, Chapters 119–130) builds on the allocator and sanitizer runtimes introduced here — `compiler-rt`'s libcxx, libunwind, and builtins are the runtime foundation that sanitizer-instrumented and JIT-compiled code links against; Chapter 117's DWARF CFI is the unwinding metadata that libunwind consumes at throw time.

## Cross-Part Dependencies

- Ch 58 (Part X) — `@llvm.experimental.patchpoint` and `.llvm_stackmaps` generation described here is consumed by ORC's deoptimization mechanism in Chapters 108 and 221
- Ch 67 (Part XIII) — PGO profile collection and merging via `llvm-profdata` is the same toolchain as Chapter 115's source-based coverage pipeline; BOLT (Chapter 118) also consumes PGO profiles
- Ch 91–99 (Part XIV) — backend code generators produce the ELF/MachO/COFF object files that JITLink (Chapter 109) links; AArch64 backend emits MTE instructions used in Chapter 111
- Ch 100–107 (Part XV) — AArch64StackTaggingPass (Chapter 111), RISC-V relaxation tables (Chapter 109), and x86_64 GOT-relocation encodings (Chapter 109) are produced by Part XV targets
- Ch 119–130 (Part XVII) — `compiler-rt` libunwind consumes DWARF CFI (Chapter 117); libc++ uses the sanitizer-instrumented allocators (Chapters 110, 112); the ORC runtime (`compiler-rt/lib/orc/`) trampolines are the other half of Chapter 108's lazy compilation
- Ch 144 (Part XXIV) — Alive2 equivalence checking mentioned in Chapter 220's verification-before-hot-swap pattern is covered in depth in the verified compilation part
- Ch 193 (Part XXVIII) — Julia's `@generated` functions and ORC-based type-specialized JIT are production uses of Chapter 108 and Chapter 219's material
- Ch 207 (Part XXXI) — theoretical treatment of certified self-modification (ρ-calculus, Lean 4 quotation, SICA) references the practical ORC self-modification infrastructure in Chapters 108 and 220

## Navigation

- ← Part XV — Targets
- → Part XVII — Runtime Libraries

---

*@copyright jreuben11*
