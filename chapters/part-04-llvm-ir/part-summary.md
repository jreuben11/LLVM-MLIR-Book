# Part IV — LLVM IR — Part Summary

*Part IV is the engineering foundation of the book: twelve chapters that systematically define LLVM IR from its data model and type system through its full instruction set, metadata, attributes, intrinsics, inline assembly, exception handling, coroutines, and atomics, giving the reader complete mastery of the language that every LLVM optimization pass reads, transforms, and produces.*

---

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 16 | IR Structure | Module hierarchy, Value/User/Use triad, .ll and .bc formats |
| 17 | The Type System | Integer, FP, vector, aggregate, opaque pointer, DataLayout |
| 18 | Constants, Globals, and Linkage | ConstantExpr, GlobalVariable, linkage taxonomy, TLS, comdats |
| 19 | Instructions I — Arithmetic and Memory | nsw/nuw/exact flags, GEP, load/store, alloca, freeze, poison |
| 20 | Instructions II — Control Flow and Aggregates | br/switch/phi/select, aggregate ops, EH terminators |
| 21 | SSA, Dominance, and Loops | DominatorTree, LoopInfo, mem2reg, SROA, CycleInfo |
| 22 | Metadata and Debug Info | TBAA, loop hints, profile metadata, DWARF DINodes, RemoveDIs |
| 23 | Attributes, Calling Conventions, and the ABI | Function/param/call attributes, CC catalogue, ABI lowering |
| 24 | Intrinsics | @llvm.* vocabulary: memory, math, vectors, atomics, GC, debug |
| 25 | Inline Assembly | InlineAsm value type, constraint language, lowering to MIR |
| 26 | Exception Handling | Itanium two-phase, Windows funclets, SJLJ, Wasm EH |
| 27 | Coroutines and Atomics | llvm.coro.* split-function ABI, C11 memory model, cmpxchg |

---

## Part Overview

Part IV opens where Part III ended: with the observation that formal type theory must eventually be materialized in machine-checkable data structures. Chapter 16 establishes the four-level containment hierarchy — `Module` → `Function` → `BasicBlock` → `Instruction` — and the `Value`/`User`/`Use` intrusive linked-list triad that is the concrete realization of the SSA def-use graph. Every subsequent chapter in the part builds on this object model. Chapter 17 catalogs every category of LLVM type in operational detail: the arbitrary-width `iN` integers, the full floating-point suite including `bfloat` for ML workloads, fixed-length and scalable vectors for SIMD targets (AArch64 SVE, RISC-V RVV), aggregate types, and the opaque `ptr` model that has replaced typed pointers since LLVM 15. Chapter 18 then populates the type system with values: the constant hierarchy rooted in `ConstantData` and `ConstantExpr`, the `poison` and `undef` special constants, global variables with their linkage types and TLS models, aliases, indirect functions, and the comdat mechanism that makes one-definition-rule semantics work across translation units.

Chapters 19 and 20 form the instruction reference. Chapter 19 covers the instructions with the most optimizer-facing semantic content: the `nsw`/`nuw`/`exact`/`disjoint` flags that permit transformations under the poison contract, the floating-point fast-math flag set, `getelementptr` as a pure pointer arithmetic primitive, `load`/`store` with alignment and volatile semantics, `alloca` and its dissolution by `mem2reg`, the `freeze` instruction that domesticates poison, and the `ptrtoint`/`inttoptr` pair whose provenance implications constrain alias analysis. Chapter 20 covers everything else: the full terminator set (`br`, `switch`, `indirectbr`, `callbr`), function call semantics including `invoke` and tail-call attributes, the `phi` instruction as the SSA join mechanism, `select` for branchless conditionals, aggregate access instructions, vector lane manipulation, and the exception-handling terminators for both the Itanium and Windows funclet models.

The final six chapters address the layers of semantic enrichment that surround the core instruction set. Chapter 21 maps SSA theory to LLVM's analysis infrastructure: how Clang emits alloca-first IR that `mem2reg` and SROA later promote to SSA form, and how `DominatorTree`, `PostDominatorTree`, `LoopInfo`, `LoopNest`, and `CycleInfo` are queried from pass code. Chapter 22 covers the metadata system — TBAA, range, loop transformation hints, profile weights, alias scope/noalias, DWARF DINode hierarchies, and the new `DbgVariableRecord` format that decouples debug info from the SSA graph. Chapter 23 provides a comprehensive reference to attributes and calling conventions, and examines the ABI lowering boundary where `sret`, `byval`, and `byref` bridge IR and hardware conventions. Chapter 24 is a systematic reference to the `@llvm.*` intrinsic vocabulary: memory bulk-copy, floating-point math, bit manipulation, overflow-checking arithmetic, vector reductions, masked and scalable-vector operations, lifetime markers, pointer authentication, and GC statepoints. Chapter 25 covers inline assembly from the `InlineAsm` value type through the AT&T/Intel dialect syntax, constraint and clobber language, GCC-compatible extended asm, and the lowering path through `SelectionDAGBuilder` to `MachineInstr`. Chapters 26 and 27 address the two most structurally complex features of the IR: exception handling (two-phase Itanium unwinding, Windows funclets, SJLJ, and WebAssembly EH) and the coroutine/atomics pair (the `llvm.coro.*` split-function transformation and the C11/C++11 memory model expressed through `load atomic`, `cmpxchg`, `atomicrmw`, and `fence` with `AtomicOrdering` qualifiers).

After completing Part IV, the reader can read, write, and verify any legal LLVM 22 IR; understand every flag, attribute, and metadata kind that a Clang-generated module may contain; query and manipulate the IR object model from C++ pass code; and reason correctly about the optimizer's latitude over each instruction form. This competence is the prerequisite for everything that follows in the book: front-end code generation (Part V–VIII), analysis and optimization passes (Part X), polyhedral transformation (Part XII), backend instruction selection (Part XIV), JIT compilation (Part XVI), MLIR dialects (Part XX), and verified compilation (Part XXIV).

---

## Key Concepts Introduced

- **The four-level hierarchy** (`Module` → `Function` → `BasicBlock` → `Instruction`): the strict containment structure that partitions every IR object and drives all pass traversal patterns.
- **`Value`/`User`/`Use` triad**: the intrusive linked-list implementation of the SSA def-use graph; `replaceAllUsesWith` transfers an entire use-list in O(n) time, enabling safe IR mutation without dangling references.
- **Opaque pointers (`ptr`)**: LLVM 22's unified pointer type that replaced typed pointers, eliminating redundant pointee-type information and simplifying GEP semantics across the optimizer.
- **`DataLayout` string**: the module-level encoding of byte order, alignment constraints, pointer sizes per address space, and native integer widths that drives both code generation and alias analysis.
- **Poison semantics and `freeze`**: the contract by which `nsw`/`nuw`/`exact`/`inbounds` flags assert mathematical properties and confer optimizer latitude; `freeze` converts poison to an arbitrary but fixed value, enabling safe use of otherwise undefined computations.
- **`getelementptr` (GEP)**: the sole pointer arithmetic instruction, performing pure offset computation with `inbounds` as an aliasing and overflow assertion; GEP never dereferences memory.
- **Module flags (`!llvm.module.flags`)**: structured key-value metadata with explicit merge behaviors (`Error`, `Max`, `Min`, `Append`) that encode ABI constraints, debug info format, PIC/PIE level, and unwind table requirements across link-time module combination.
- **TBAA metadata (`!tbaa`)**: type-based alias analysis trees attached to loads and stores that communicate strict-aliasing information to the optimizer, enabling load-store reordering and dead-store elimination across opaque `ptr` operands.
- **`DominatorTree` and `LoopInfo`**: the two analysis results consumed by virtually every optimization pass; `DominatorTree` supports the SSA dominance check; `LoopInfo` exposes the natural-loop hierarchy with header, latch, preheader, and exit nomenclature.
- **Intrinsics (`@llvm.*`)**: the extensibility mechanism by which operations outside the core opcode set — memory bulk-copy, transcendental math, vector reductions, overflow checks, GC statepoints, coroutine tokens — are expressed as typed, attributed function calls with precisely specified optimizer semantics.
- **Calling conventions and ABI attributes**: the IR layer through which LLVM encodes hardware calling-convention choices (`fastcc`, `swiftcc`, `x86_fastcallcc`, etc.) and struct-passing conventions (`sret`, `byval`, `byref`, `inalloca`) that the backend must faithfully lower to register assignments and stack layouts.
- **Itanium two-phase exception handling**: the `invoke`/`landingpad`/`resume` model that encodes exceptional control flow edges in the CFG, enabling analysis passes to reason about stores visible on unwind paths and allowing `nounwind` inference to convert `invoke` to `call`.
- **Coroutine split-function ABI (`llvm.coro.*`)**: the pre-split intrinsic representation that CoroSplit transforms into independent resume/destroy funclets with a heap-allocated frame, implementing C++20 coroutines and Swift async functions without library-level stack switching.
- **C11/C++11 memory model in IR**: `load atomic`, `store atomic`, `cmpxchg`, `atomicrmw`, and `fence` with `AtomicOrdering` (`monotonic`, `acquire`, `release`, `seq_cst`) and `SyncScope` qualifiers that `AtomicExpandPass` lowers to target-specific fence and LL/SC instruction sequences.

---

## How This Part Fits the Book

Parts I–III provide the theoretical prerequisites: Part I (Chapters 1–5) surveys the compiler pipeline and LLVM toolchain; Part II (Chapters 6–15) covers compiler theory through SSA construction, dataflow, and type theory; Part III (Chapters 14–15) maps formal type systems to their IR and MLIR realizations. Part IV (Chapters 16–27) is the engineering complement to all of that theory — the concrete data structures, APIs, and instruction semantics that pass authors, front-end authors, and backend engineers operate on daily. Every subsequent part of the book consumes the IR model introduced here: Part V (Chapters 28–37) shows how Clang lowers C and C++ to the IR constructs defined in Chapters 16–27; Part X (Chapters 58–72) implements analysis and optimization passes over the `DominatorTree`, `LoopInfo`, and metadata APIs described in Chapters 21–22; Part XIV (Chapters 87–97) shows how the backend lowers the calling-convention attributes of Chapter 23 and the intrinsics of Chapter 24 to `MachineInstr`; and Parts XIX–XXI show how MLIR's region model generalizes the `BasicBlock` container introduced in Chapter 16.

---

## Cross-Part Dependencies

The following chapters in other parts depend directly on material introduced in Part IV:

- Ch 28–37 (Part V — Clang Frontend Pipeline) — Clang `CodeGenFunction` emits the alloca-first IR pattern (Ch 21), GEP sequences (Ch 19), and attribute sets (Ch 23) described here
- Ch 38–44 (Part VI — Clang Codegen) — C++ exception emission uses `invoke`/`landingpad` (Ch 26); C++20 coroutine lowering uses `llvm.coro.*` intrinsics (Ch 27)
- Ch 58–72 (Part X — Analysis and Middle-End) — every optimization pass queries `DominatorTree` (Ch 21), reads TBAA (Ch 22), respects `nounwind` (Ch 26), and patterns over `nsw`/`nuw` flags (Ch 19)
- Ch 73–80 (Part XI — Polyhedral Theory) and Ch 81–86 (Part XII — Polly) — loop analyses consume `LoopInfo` and `LoopNest` (Ch 21); Polly reads loop metadata hints (Ch 22)
- Ch 87–97 (Part XIV — Backend) — instruction selection lowers calling-convention attributes (Ch 23), intrinsics (Ch 24), inline assembly (Ch 25), and atomic instructions (Ch 27) to `SelectionDAG` and `MachineInstr`
- Ch 100–110 (Part XVI — JIT and Sanitizers) — JIT materializes bitcode modules whose format is defined in Ch 16; sanitizer passes inject calls to `@llvm.asan.*` intrinsics (Ch 24)
- Ch 140–155 (Part XIX–XX — MLIR Foundations and In-Tree Dialects) — the `llvm` MLIR dialect is a direct structural reflection of Part IV's IR model, including the `llvm.ptr` type (Ch 17), `llvm.getelementptr` (Ch 19), and coroutine intrinsic analogues (Ch 27)
- Ch 174 (Part XXIV — Verified Compilation) — Vellvm and Alive2 formalize the poison semantics (Ch 19) and memory model (Ch 27) introduced in this part
- Ch 196 (Part XXVI — Ecosystem Frontiers) — LLVM IR as a language-neutral interchange layer builds on the module/bitcode model (Ch 16) and the linkage taxonomy (Ch 18)

---

## Navigation

- ← Part III — Type Theory
- → Part V — Clang Internals: Frontend Pipeline

---

*@copyright jreuben11*
