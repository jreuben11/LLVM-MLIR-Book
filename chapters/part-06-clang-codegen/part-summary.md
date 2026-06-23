# Part VI — Clang Code Generation — Part Summary

*This part dissects the pipeline that transforms a fully-analyzed Clang AST into LLVM IR, covering the core codegen infrastructure, ABI-boundary translation for every major target, Itanium and Microsoft C++ ABI lowering, and C++20 coroutine state-machine generation.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 39 | CodeGenModule and CodeGenFunction | Core codegen classes and IR emission infrastructure |
| 40 | Lowering Statements and Expressions | AST-to-IR dispatch for all statement and expression forms |
| 41 | Calls, the ABI Boundary, and Builtins | Platform ABI classification, argument marshaling, builtins |
| 42 | C++ ABI Lowering: Itanium | Vtables, name mangling, EH, constructors on UNIX targets |
| 43 | C++ ABI Lowering: Microsoft | MSVC vtables, SEH, member pointers, Windows-specific ABI |
| 44 | Coroutine Lowering in Clang | C++20 coroutine frame, CoroSplit pass, HALO elision |

---

## Part Overview

Part VI spans the translation frontier between the Clang semantic model and the LLVM IR model. Where Part V (Clang Frontend Pipeline) ends — with a fully type-checked, semantics-validated AST — this part begins. The first two chapters establish the foundational machinery: `CodeGenModule` owns the `llvm::Module` for an entire translation unit and coordinates global emission, deferred declarations, type conversion, VTable tables, debug-info generation, and the mangling cache; `CodeGenFunction` is instantiated once per function body and operates a per-function `CGBuilderTy`, alloca insertion point, EH scope stack, and cleanup machinery. Chapter 40 then catalogs every statement and expression visitor in `CGStmt.cpp`, `CGExprScalar.cpp`, `CGExprComplex.cpp`, and `CGExprAgg.cpp`, revealing how a `for` loop becomes a four-block CFG with `!llvm.loop` metadata, how signed overflow becomes `nsw` flags, how `&&`/`||` become conditional branches with `phi` nodes, and how C++ temporaries and RAII cleanups couple through `EHScopeStack`.

Chapter 41 attacks the ABI boundary — arguably the most intricate single concern in the codegen layer. LLVM IR is deliberately ABI-agnostic: it relies on Clang to transform source-level types into the register-passing or memory-passing representations each platform demands. The `ABIArgInfo` taxonomy (`Direct`, `Extend`, `Indirect`, `Expand`, `CoerceAndExpand`, `InAlloca`, `Ignore`, `TargetSpecific`) encodes one decision per argument; `CGFunctionInfo` aggregates these decisions into a full signature descriptor, cached in a `FoldingSet`; and target-specific `ABIInfo` subclasses implement the classification algorithm for x86-64 System V, Win64, AArch64 AAPCS64, ARM, RISC-V, and others. The chapter continues into `__builtin_*` lowering (hundreds of LLVM intrinsic mappings), C11 atomic lowering, and variadic argument extraction.

Chapters 42 and 43 address the two dominant C++ ABI families. Chapter 42 covers the Itanium ABI — the standard for Linux, macOS, Android, FreeBSD, and most UNIX derivatives — dissecting name mangling from `_Z` prefix through substitution compression, vtable layout from offset-to-top through VTT construction vtables, virtual dispatch thunks, RTTI hierarchy (`__class_type_info`, `__vmi_class_type_info`), constructor/destructor C1/C2/D0/D1/D2 variants, guard variables for static locals using `__cxa_guard_acquire`, `operator new` with array cookies, thread-local `__cxa_thread_atexit` initialization, and the full `__gxx_personality_v0` EH machinery with LSDA tables. Chapter 43 mirrors this for the Microsoft ABI (`MicrosoftCXXABI`, `MicrosoftCXXNameMangler`): `?`-prefix mangling in reverse scope order, `vftable`/`vbtable` separation, `CompleteObjectLocator` at `vtable[-1]`, single constructor with a `should_call_vbase_constructors` boolean, scalar/vector deleting destructors, `__CxxFrameHandler3` personality with `catchpad`/`cleanuppad` funclet IR, image-relative RTTI offsets for ASLR, and inheritance-model-dependent member pointer sizes.

Chapter 44 closes the part by covering C++20 coroutine lowering, which is unlike any other codegen transformation. Clang's Sema layer builds a `CoroutineBodyStmt` that captures synthesized sub-statements for the promise protocol, initial/final suspend, exception handler, and parameter moves. `EmitCoroutineBody()` in `CGCoroutine.cpp` emits a pre-split monolithic function using `llvm.coro.id`, `llvm.coro.alloc`, `llvm.coro.begin`, `llvm.coro.save`, `llvm.coro.suspend`, and `llvm.coro.end` intrinsics as opaque markers. `CoroSplitPass` then materializes the coroutine frame struct using `SuspendCrossingInfo` dataflow analysis, splits the function into a ramp, a resume switch-dispatch function, and a destroy function, and inserts `musttail` calls for symmetric transfer. `CoroElidePass` eliminates heap allocation when the frame provably does not escape.

After completing this part, the reader will be able to trace any C or C++ source construct from AST node through emitted LLVM IR, understand why an argument appears in a register or behind a pointer for any of the five major target ABIs, diagnose miscompilations rooted in ABI classification or cleanup-scope ordering, extend Clang's codegen for new language constructs, and reason about the full coroutine lowering pipeline from `CoroutineSuspendExpr` through post-split resume IR.

---

## Key Concepts Introduced

- **`CodeGenModule`** — Module-level singleton owning the `llvm::Module`, global emission queue (`DeferredDeclsToEmit`), mangling cache, `CodeGenTypes`, `CodeGenVTables`, and the `CGCXXABI` dispatch object; created once per translation unit.
- **`CodeGenFunction`** — Per-function IR builder that owns `CGBuilderTy`, `EHScopeStack`, `AllocaInsertPt`, `LocalDeclMap`, and the `JumpDest`-based control flow stack; instantiated once per function body.
- **`Address` / `LValue`** — Clang's typed-pointer abstractions that carry LLVM IR's opaque `ptr` value together with pointee type, alignment, TBAA access info, and volatile/restrict qualifiers for correct load/store emission.
- **`ABIArgInfo`** — Per-argument ABI classification encoding how a source-level type crosses the ABI boundary: `Direct` (possibly with type coercion), `Indirect` (pass-by-pointer with or without `byval`), `Expand`, `CoerceAndExpand`, `InAlloca`, `Ignore`, `Extend`, or `TargetSpecific`.
- **`CGFunctionInfo`** — ABI-lowered function signature aggregating `ABIArgInfo` for return type and all parameters, stored in a `FoldingSet` for deduplication; produced by `ABIInfo::computeInfo()` on each target.
- **`EHScopeStack` and cleanup scopes** — A linked stack of `EHCleanupScope`, `EHCatchScope`, and related frames that tracks active destructors and exception handlers; `EmitBranchThroughCleanup` walks this stack to emit cleanup code between jump sources and targets.
- **`CGCXXABI` abstraction** — The virtual interface (`ItaniumCXXABI` vs `MicrosoftCXXABI`) that decouples all C++ ABI specifics — vtable emission, name mangling, constructor/destructor variant selection, guard variables, member pointer representation, and EH personality — from target-independent codegen.
- **Itanium name mangling** — Encoding of C++ external names as `_Z`-prefixed strings with nested-name compression (`N...E`), template argument brackets (`I...E`), single-letter built-in type codes, and substitution compression (`S_`/`S0_`/`S1_`).
- **Itanium vtable layout** — In-memory representation with offset-to-top, RTTI pointer, and function pointer array; primary and secondary vtables for multiple/virtual inheritance; VTT (virtual table table) for construction vtable navigation.
- **MSVC vftable/vbtable separation** — MSVC virtual function tables contain only function pointers with `CompleteObjectLocator` at slot `−1`; separate `vbtable` arrays provide virtual-base offsets via a `vbptr` field in each object; object layout can carry multiple `vfptr` fields for multiple-inheritance bases.
- **Windows EH funclet IR** — `catchswitch`/`catchpad`/`cleanuppad`/`catchret`/`cleanupret` instruction forms that model Windows EH's function-within-a-function semantic; every call inside a funclet must carry a `"funclet"(token)` operand bundle.
- **`CoroutineBodyStmt` and suspension intrinsics** — Clang's AST representation of a coroutine body with synthesized sub-statements for all protocol machinery; LLVM `llvm.coro.*` intrinsics (`coro.id`, `coro.alloc`, `coro.begin`, `coro.save`, `coro.suspend`, `coro.end`, `coro.free`) serve as opaque markers until `CoroSplitPass` materializes the frame.
- **`CoroSplitPass` and `SuspendCrossingInfo`** — The CGSCC pass that identifies which values live across suspension points (via dataflow analysis), materializes the coroutine frame struct, and splits the function into ramp / resume / destroy; inserts `musttail` for O(1) symmetric transfer.
- **Heap Allocation eLision Optimization (HALO)** — `CoroElidePass` replaces coroutine frame heap allocation with a caller-stack `alloca` when the frame does not escape and all destroys are post-dominated by the call site.

---

## How This Part Fits the Book

Part V (Chapters 28–38) delivers a fully analyzed, type-checked Clang AST — `QualType`, `ASTContext`, `Decl`/`Stmt`/`Expr` hierarchies, target info, and diagnostics. Part VI consumes that output directly: every `CodeGenFunction` method in this part calls into `ASTContext`, `QualType`, and the `Sema`-built `CoroutineBodyStmt` nodes established in Part V. Part IV (Chapters 15–27) defines the LLVM IR model that this part emits — `llvm::Module`, `llvm::Function`, `llvm::BasicBlock`, `llvm::Instruction`, `landingpad`/`invoke`/`resume`, the coroutine intrinsic ABI, and the `!tbaa`/`!prof`/`!llvm.loop` metadata families; readers unfamiliar with the IR model should review Part IV before Chapters 39–44. Parts VII and VIII (Clang Multi-Language & Tooling, and ClangIR) build directly on this part: Chapter 45 onward covers Objective-C runtime codegen and OpenCL address-space lowering that extend `CodeGenModule`, while Part VIII (Chapters 47–52) documents ClangIR's `CIRGenModule`/`CIRGenFunction` mirror of the infrastructure established here.

---

## Cross-Part Dependencies

- Ch 16 (Part IV) — LLVM IR Structure: `llvm::Module`, `llvm::Function`, `llvm::BasicBlock` — the output types produced by every function in this part
- Ch 19 (Part IV) — Instructions I: Arithmetic and Memory: `add nsw`, `load`, `store`, `getelementptr` — the IR instructions emitted by expression lowering (Ch 40)
- Ch 20 (Part IV) — Instructions II: Control Flow and Aggregates: `br`, `switch`, `phi`, `indirectbr`, `select` — used throughout statement lowering (Ch 40)
- Ch 23 (Part IV) — Attributes, Calling Conventions, and the ABI: LLVM IR parameter attributes (`byval`, `sret`, `zeroext`, `signext`, `noalias`) and calling-convention IDs that are set by ABI classification (Ch 41)
- Ch 26 (Part IV) — Exception Handling: `landingpad`, `invoke`, `resume`, `.eh_frame`/LSDA model — the IR EH constructs driven by `EHScopeStack` (Ch 40) and the Itanium personality (Ch 42)
- Ch 27 (Part IV) — Coroutines and Atomics: LLVM coroutine intrinsic reference (`llvm.coro.*`) and the Switch/Retcon/Async ABI variants — the intrinsic layer materialized by `CoroSplitPass` (Ch 44)
- Ch 36 (Part V) — The Clang AST in Depth: `QualType`, `ASTContext`, `Decl`/`Stmt`/`Expr` hierarchies — the AST consumed by every chapter in this part
- Ch 38 (Part V) — Sema: Type Checking and Overload Resolution: `CGFunctionInfo` arrangement uses Sema-computed `Linkage` and `CallingConv` values; `CoroutineBodyStmt` is synthesized by `SemaCoroutine.cpp` (Ch 44)
- Ch 47 (Part VIII) — ClangIR Foundations: `CIRGenModule`/`CIRGenFunction` mirror the `CodeGenModule`/`CodeGenFunction` architecture of Ch 39, requiring this part as a prerequisite
- Ch 68 (Part X) — Analysis and the Middle End: TBAA, loop metadata, lifetime markers, and `nsw`/`nuw` flags emitted in this part feed LLVM's alias analysis and loop optimization passes
- Ch 80 (Part XII) — Polly: Loop metadata (`!llvm.loop`) and the structured control-flow patterns emitted in Ch 40 are prerequisites for polyhedral loop analysis
- Ch 96 (Part XV) — The AArch64 Backend: PAC-authenticated indirect calls (`ptrauth` operand bundles), HFA `CoerceAndExpand` argument classification, and the AArch64 Windows SEH unwind opcodes (`UOP_PACSignLR`) originate in Chs 41 and 43
- Ch 120 (Part XVI) — Sanitizers: UBSan overflow checks (`llvm.sadd.with.overflow`), address sanitizer stack lifetime markers, and the `sanitize_address`/`sanitize_thread` function attributes emitted in Chs 39 and 40
- Ch 155 (Part XIX) — MLIR Foundations: The coroutine frame materialization and `SuspendCrossingInfo` dataflow analysis in Ch 44 parallel MLIR's liveness analysis infrastructure; understanding the LLVM coroutine pass pipeline is valuable context for MLIR's async dialect

---

## Navigation

- ← Part V — Clang Internals: Frontend Pipeline
- → Part VII — Clang Multi-Language & Tooling

---

*@copyright jreuben11*
