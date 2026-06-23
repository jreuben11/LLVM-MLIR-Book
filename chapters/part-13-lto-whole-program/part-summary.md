# Part XIII — LTO & Whole-Program Analysis — Part Summary

*This part covers the technology that operates after individual translation units are compiled: link-time optimization, the LLD linker's internals, low-level ABI structures (GOT, PLT, TLS), and the content-addressable build infrastructure that makes large-scale LLVM builds reproducible and incrementally efficient.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 77 | LTO and ThinLTO | Cross-module optimization via bitcode and the ThinLTO summary index |
| 78 | The LLVM Linker (LLD) | LLD architecture, symbol resolution, ICF, and profile-guided ordering |
| 79 | Linker Internals: GOT, PLT, TLS | Dynamic linking data structures and linker relaxation |
| 80 | llvm-cas and Content-Addressable Builds | Reproducible hermetic builds via content-addressed artifact storage |

## Part Overview

Part XIII addresses the final phase of the compilation pipeline: everything that happens after the compiler has emitted per-module artifacts and before the finished binary reaches the operating system loader. The four chapters form a layered treatment, moving from high-level cross-module optimization (Ch77), through the linker's execution model and output organization (Ch78), down to the ABI-level data structures that mediate between the static linker's decisions and the dynamic loader's runtime behavior (Ch79), and finally to the build infrastructure layer that makes large-scale, distributed, reproducible linking practical (Ch80).

Chapter 77 establishes the central idea of the part: that optimizing each translation unit in isolation leaves significant performance and code-size gains unrealized. Monolithic LTO recaptures those gains by merging all LLVM IR bitcode into a single module at link time, but pays a prohibitive memory and latency cost for large programs. ThinLTO resolves this tension with a compact *module summary index* — a per-module record of GUIDs, call graph edges, function attributes, and type-check metadata — that enables cross-module import decisions without materializing the full IR for every module. Distributed ThinLTO and the UnifiedLTO bitcode format extend this to cluster-scale builds and eliminate the need for separate per-LTO-mode object directories.

Chapter 78 covers LLD, the linker that is the required execution environment for ThinLTO's native multi-threaded backend. LLD's architecture — four independent format drivers (ELF, COFF, Mach-O, Wasm) sharing common infrastructure, a single-pass execution model, and parallel I/O — gives it a 2–5x speed advantage over GNU ld. The chapter develops symbol resolution priority rules, COMDAT deduplication, ICF (Identical Code Folding) for template-heavy C++ codebases, linker scripts, partial linking, and profile-guided function ordering via Pettis-Hansen call-graph clustering, which reduces cold startup page faults by 10–30%.

Chapter 79 descends to the mechanism level, explaining the data structures that LLD synthesizes at link time to support dynamic linking and thread-local storage. The Global Offset Table (GOT) enables position-independent code by indirecting absolute addresses through a per-process writable table; the Procedure Linkage Table (PLT) implements lazy dynamic symbol binding through a GOT-based trampoline that the dynamic linker patches on first call. Thread-local storage introduces four addressing models (GD, LD, IE, LE) trading generality against runtime cost, and LLD applies TLS relaxation to convert the general model toward the fastest applicable model. A substantial extended section covers linker relaxation across RISC-V, AArch64, and x86-64, including the iterative fixpoint algorithm that guarantees convergence.

Chapter 80 addresses the build infrastructure concern that is increasingly central to large LLVM deployments: how to make the entire pipeline — compilation, module caching, LTO, and linking — reproducible and incrementally efficient at team scale. The LLVM CAS (Content-Addressable Store) assigns every artifact a SHA-256 content hash (CASID), provides a hermetic virtual filesystem (CASFS) that captures all compiler inputs including system headers, and exposes an action cache that maps input hashes to output CASIDs so that unchanged compilation steps are never re-run. The chapter extends into supply-chain security, covering the Thompson trusting-trust problem, quines and their relationship to self-hosting compilers, `SOURCE_DATE_EPOCH` reproducibility flags, bootstrappable build chains, diffoscope-based verification, SBOMs, and Sigstore/SLSA attestation frameworks.

## Key Concepts Introduced

- **Monolithic LTO**: merges all LLVM IR bitcode into one module at link time; enables full IPO (GlobalDCE, WPD, IPSCCP, dead argument elimination) but requires O(N) memory and serial optimization.
- **ThinLTO module summary index** (`ModuleSummaryIndex`): a compact per-module record of GUIDs, call graph edges with hotness, function attribute flags, and type metadata; merged at link time to drive cross-module import decisions without full IR materialization.
- **Function importing (cross-module inlining)**: ThinLTO's primary optimization; hot callee IR is fetched from other modules' bitcode, enabling the per-module optimizer to inline, constant-fold, and DCE through imported functions.
- **UnifiedLTO**: a single bitcode object format containing both the full LLVM IR module and a ThinLTO summary; allows the linker to select Full LTO or ThinLTO at link time via `--lto=full`/`--lto=thin`, eliminating per-mode object directories.
- **Distributed ThinLTO**: separates the summary-merge phase (link coordinator) from per-module optimization (cluster workers), enabling build-cluster parallelism with module-granularity incremental caching.
- **LLD's single-pass execution model**: processes each input file once using lazy archive extraction and a graph-based symbol table; achieves 2–5x faster link times than GNU ld through parallel I/O and section processing.
- **Identical Code Folding (ICF)**: iterative hash-based equivalence algorithm that merges functions with identical code (after relocation), reducing binary size by 2–10% for template-heavy C++ programs.
- **Profile-guided function ordering** (`--call-graph-profile-sort`): applies Pettis-Hansen call-chain clustering to place hot functions contiguously at the start of `.text`, reducing cold startup page faults by 10–30%.
- **Global Offset Table (GOT)**: a per-process writable table of absolute addresses referenced by PIC code via PC-relative loads; avoids patching read-only `.text` sections and allows OS page sharing across processes.
- **Procedure Linkage Table (PLT)**: a set of per-function stubs that implement lazy binding by routing first calls through `_dl_runtime_resolve`, which patches the GOT entry for subsequent direct calls.
- **TLS relaxation**: linker-applied conversion of General Dynamic TLS (two function calls) through Initial Exec (one GOT read) to Local Exec (single TP-relative instruction), triggered when the linker can determine the symbol's final module placement.
- **Linker relaxation (shrink-only fixpoint)**: iterative replacement of conservative instruction sequences (e.g., RISC-V `auipc`+`jalr`) with shorter equivalents (e.g., `jal`) once final addresses are known; guaranteed to converge because sections only shrink.
- **LLVM CAS (Content-Addressable Store)**: identifies every compilation artifact by a SHA-256 CASID derived from its content and referenced object IDs; stores artifacts in a content-addressed on-disk or remote store.
- **CASFS (CAS Filesystem)**: a hermetic virtual filesystem that presents the compiler with a read-only CAS-rooted view of all inputs including system headers, eliminating implicit input dependencies and enabling CASID-keyed reproducibility verification.
- **Action cache**: maps a hash of all compilation inputs (source CASID, flags, compiler version) to the CASID of the compilation output; enables build-wide caching so unchanged compilations are never re-run.

## How This Part Fits the Book

Part XIII builds directly on Part IV (LLVM IR) and Part X (Analysis and Middle-End): LTO reuses the same optimization passes (inliner, IPSCCP, GlobalDCE, WPD from Ch69) on a link-time-merged or summary-guided module, and ThinLTO's import decisions depend on the function attribute and call-graph analysis covered in those parts. Parts XIV–XV (Backend and Targets) consume the native object files that LLD produces, and the relocation, TLS, and section layout details of Ch79 directly constrain what the code generator must emit. Part XVI (JIT and Sanitizers) depends on LLD's symbol wrapping, COMDAT handling, and section insertion mechanisms introduced in Ch78.

## Cross-Part Dependencies

- Ch69 (Part X) — Whole-Program Devirtualization: Full LTO (Ch77 §77.1.3) and ThinLTO type propagation (§77.2.4) are the primary deployment vehicles for WPD; the type metadata and `TypeTest`/`TypeCheckedLoad` infrastructure described in Ch69 feeds directly into the ThinLTO summary.
- Ch67 (Part X) — Profile-Guided Optimization: ThinLTO's hotness-aware import decisions (§77.2.5) consume PGO profile metadata; LLD's call-graph profile sort (Ch78) also consumes PGO data; both chapters assume the instrumentation and profdata workflow from Ch67.
- Ch91 (Part XVI) — AddressSanitizer / Sanitizers: sanitizer instrumentation relies on LLD's symbol wrapping (`--wrap`), COMDAT section deduplication, and the `.init_array` initialization ordering described in Ch78 and Ch79.
- Ch99 (Part XVII) — compiler-rt Runtime Library: the compiler-rt startup code (`_start`, `__libc_csu_init`) is the consumer of the `.init_array`/`.fini_array` ordering mechanism described in Ch79 §79.5.
- Ch134 (Part XXIII) — MLIR Production Deployment: Ch80's CAS infrastructure and CASFS hermetic environments (§80.6 roadmap) are targeted for extension into MLIR's `mlir-opt`/`mlir-translate` pipeline for reproducible AI/ML compiler deployments.
- Ch176 (Part XXIV) — CompCert and Verified Compilation: Ch80's discussion of bootstrappable builds, DDC (Diverse Double Compilation), and the formal limits of self-verification connects directly to CompCert's mechanized correctness proofs and the Vellvm verification work covered in Ch176.

## Navigation

- ← Part XII — Polly
- → Part XIV — Backend

---

*@copyright jreuben11*
