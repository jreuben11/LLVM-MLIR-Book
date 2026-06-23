---
# Part IX — Frontend Authoring — Part Summary

*This part teaches how to build a complete, production-quality language frontend on top of LLVM, from source text to verified LLVM IR, including the runtime mechanisms that make managed languages viable.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 55 | Building a Frontend | Lexer, parser, AST, type checker, CMake integration |
| 56 | Lowering AST to IR | IRBuilder, alloca/mem2reg, control flow, debug info |
| 57 | Lowering High-Level Constructs | Structs, vtables, closures, tagged unions, coroutines |
| 58 | Language Runtime Concerns | GC statepoints, TLS, atomics, stack maps, signal safety |

## Part Overview

Part IX answers the question every language designer eventually asks: how do I make my language compile through LLVM? Clang is a frontend, not the frontend — Rust, Swift, Julia, Zig, Flang, and dozens of domain-specific languages all drive the same LLVM backend through independently written pipelines. This part builds that pipeline from first principles, using a running example language called *Cal* that extends the canonical Kaleidoscope tutorial with integer types, explicit type annotations, and a full statement/expression split.

The arc begins with the mechanical foundations. Chapter 55 establishes the four-stage pipeline — lexer, parser, AST, type checker — and makes each stage independently testable. It covers the practical engineering decisions that dominate production frontend code: hand-written recursive descent over generated parsers, Pratt (top-down operator precedence) parsing for expression hierarchies, arena allocation via `BumpPtrAllocator` instead of per-node heap allocation, `llvm::StringSwitch` for keyword dispatch, and CMake integration via `find_package(LLVM)` and `llvm_map_components_to_libnames`. Chapter 56 takes the typed AST and drives it into LLVM IR through `IRBuilder<>`, covering every expression and statement lowering pattern, the alloca-then-mem2reg idiom for mutable variables, phi node construction for value-producing branches, loop emission with explicit back-edges, and `DIBuilder`-based DWARF debug information. The chapter ends with a concrete before/after comparison of the `fib` function showing what `PromotePass` eliminates.

Chapter 57 addresses the constructs that have no direct LLVM counterpart. Real languages have structs accessed through `getelementptr`, virtual dispatch through vtable globals and function-pointer loads, closures as `{fn_ptr, env_ptr}` pairs with heap-allocated environment records, tagged unions as discriminant-plus-byte-array payloads lowered to `switch` on the tag, coroutines via the `@llvm.coro.*` intrinsic family, string literals as `PrivateLinkage` constant byte arrays, and RTTI under either the Itanium C++ ABI or a custom pointer-sized discriminator. Chapter 58 completes the picture with runtime concerns that every production language must address: GC safe points using the statepoint model with `gc.relocate` for moving collectors, thread-local storage lowering models, the full C11/C++11 atomic memory model mapped to LLVM's `AtomicOrdering` enum, stack maps and patchpoints for JIT deoptimization and profiling, and the constraints signal handlers impose on IR emission.

After completing this part, a reader can build a new language frontend from a blank directory: implement the lexer and parser, design and type-check the AST, emit correct LLVM IR for every construct in the language including aggregates, closures, and sum types, attach DWARF debug information, and wire in the runtime mechanisms required for a GC-managed, multi-threaded, safely interruptible language. All code is verified against LLVM 22.1.x.

## Key Concepts Introduced

- **Recursive descent parsing**: Each grammar production becomes a function; the parser maintains a one-token lookahead and calls `expect`/`eat` helpers to consume tokens with clean error attribution.
- **Pratt (top-down operator precedence) parsing**: Handles left-recursive expression grammars with multi-level precedence cleanly without grammar transforms; each operator's binding power is encoded in a `precedence()` table.
- **Arena allocation (`BumpPtrAllocator`)**: All AST nodes are allocated from a bump-pointer slab, amortizing per-node allocation cost to a single `free` per block; raw pointers replace `unique_ptr` throughout the AST.
- **`llvm::StringSwitch` keyword dispatch**: Generates efficient, branchless keyword lookup over `std::string_view`; defined in `llvm/include/llvm/ADT/StringSwitch.h`.
- **`IRBuilder<>`**: The typed instruction factory for LLVM IR; its insertion point determines where new instructions land; the default specialization folds constants during construction.
- **Alloca-then-mem2reg idiom**: Mutable variables are emitted as entry-block `alloca`/`load`/`store` sequences; `PromoteMemoryToRegisterPass` (`PromotePass`) converts them to SSA phi-form after the module is complete, keeping the frontend simple.
- **Phi node construction for control flow**: `if`-expression and loop emission requires re-capturing the active insertion block after nested code regions (because nested branches may have moved it) before calling `PHINode::addIncoming`.
- **`DIBuilder` DWARF emission**: Attaches `DICompileUnit`, `DISubprogram`, `DILocalVariable`, and `DILocation` metadata to functions, variables, and instructions; `diBuilder->finalize()` must be called after all functions are emitted to resolve forward references.
- **`getelementptr inbounds` for field and array access**: `CreateStructGEP` is shorthand for a constant-indexed GEP into a struct; dynamic array access uses a two-level GEP `{0, idx}`; bounds-safe languages emit `icmp`/`br` guards before each dynamic GEP.
- **Vtable-based dynamic dispatch**: A vtable is a `[N x ptr]` global constant with Itanium-ABI prefix slots (offset-to-top, RTTI); dispatch loads the vtable pointer at offset 0 of the object, GEPs to the slot, loads the function pointer, and calls through it.
- **Closure lowering**: A closure becomes a `{ptr fn, ptr env}` pair; captured values are stored into a heap-allocated environment struct; the lifted function takes the environment pointer as its first argument.
- **Tagged union (sum type) lowering**: A discriminated union maps to `{i32 tag, [N x i8] payload}` where `N` is the size of the largest variant; pattern matching emits `switch` on the tag followed by typed GEPs into the payload for each arm.
- **GC statepoint model**: Every call that may trigger GC is replaced by `@llvm.experimental.gc.statepoint`, listing all live GC-managed pointers; `gc.relocate` loads potentially-moved values after the call; the `.llvm_stackmaps` ELF section records safe-point locations for the GC runtime.
- **Stack maps and patchpoints**: `@llvm.experimental.stackmap` records live values at a point for profiling or deoptimization; `@llvm.experimental.patchpoint` reserves a NOP sled that the JIT runtime can later overwrite (via dual-mapped `memfd_create` on Linux or `MAP_JIT` on Apple Silicon) without violating W^X; `StackMapParser` (v3 format in LLVM 22) locates records by patchpoint ID.
- **LLVM atomic memory model**: `LoadInst::setAtomic`, `StoreInst::setAtomic`, `CreateAtomicRMW`, and `CreateAtomicCmpXchg` map directly to C11/C++11 orderings via `llvm::AtomicOrdering`; `SyncScope::System` vs. `SyncScope::SingleThread` controls the scope of ordering guarantees.

## How This Part Fits the Book

Parts I–VIII provide the foundations this part builds on: Part IV (LLVM IR) defines the target representation and the invariants a frontend must satisfy; Part V (Clang Frontend) and Part VIII (ClangIR) show how the production C/C++ frontend traverses the same pipeline. Part X (Analysis & Middle-End) follows directly — it assumes the reader can emit valid LLVM IR and focuses on what the optimizer does with it, including the alias analysis, loop analysis, and scalar evolution passes that operate on the IR generated by patterns taught in Chapters 55–58. Part XVI (JIT & Sanitizers) extends Chapter 58's statepoint and patchpoint material with full ORC v2 JIT construction and runtime sanitizer instrumentation.

## Cross-Part Dependencies

- Ch 29 (Part V) — Clang's `ASTContext` and `RecoveryExpr` provide the production model for the arena-allocated, typed-error-hole AST patterns introduced in Ch 55.
- Ch 33 (Part VI) — Clang's `CodeGen` layer is the production counterpart to the `IREmitter` built in Ch 56; understanding both in parallel clarifies which patterns are universal vs. Clang-specific.
- Ch 42 (Part VIII) — ClangIR introduces CIR as an intermediate dialect between the typed AST and LLVM IR; Ch 56's direct `IRBuilder` approach is the lower-level alternative that CIR's lowering pipeline ultimately performs.
- Ch 60 (Part X) — Alias analysis results depend on the `noalias`, `readonly`, and address-space annotations a frontend attaches at IR construction time (Ch 56–57).
- Ch 70 (Part X) — The `mem2reg` / `PromotePass` run in Ch 56 is formally described as part of the scalar optimization pipeline; reading Ch 70 explains why the phi-form output of `mem2reg` is more amenable to subsequent optimizations.
- Ch 91 (Part XVI) — ORC v2 JIT construction extends the patchpoint and statepoint infrastructure from Ch 58 into a full tiered-compilation runtime.
- Ch 92 (Part XVI) — Sanitizer instrumentation passes instrument the IR emitted by patterns from Ch 56–57; understanding the source IR is prerequisite to understanding what the sanitizer passes insert.
- Ch 109 (Part XVIII) — Flang's frontend follows the same four-stage pipeline structure established in Ch 55, adapted for Fortran's more complex type system and intrinsic resolution.
- Ch 119 (Part XIX) — MLIR's `func`/`arith`/`cf` dialect combination is the multi-level alternative to the direct `IRBuilder` approach; the comparison is explicit in Ch 56's R&D roadmap.

## Navigation

- ← Part VIII — ClangIR
- → Part X — Analysis & Middle-End

---

*@copyright jreuben11*
