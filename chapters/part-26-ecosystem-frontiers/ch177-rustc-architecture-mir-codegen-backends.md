# Chapter 177 — rustc: Architecture, MIR, and Codegen Backends

*Part XXVI — Ecosystem and Frontiers*

The Rust compiler (rustc) is the most sophisticated production compiler outside of GCC and LLVM, and it sits atop LLVM as its primary native-code backend. Understanding rustc's internals is valuable for two distinct reasons. First, rustc makes architectural choices that no other major compiler has made at production scale: a query-based incremental computation model, a mid-level IR (MIR) that exposes borrow-checking to formal analysis, a formally-specified memory model (Stacked Borrows / Tree Borrows) implemented as an interpreter, and a borrow checker (Polonius) expressed as a Datalog program. Second, rustc is actively replacing components of LLVM with its own: the Cranelift backend outperforms LLVM at debug-build speed, and the GCC backend extends Rust's reach to targets LLVM does not support. This chapter walks through every major layer of rustc from source text to machine code, covering the MIR definition, Miri, Polonius, a-mir-formality, the LLVM codegen backend, panic/unwind mechanics, LTO and PGO, cross-compilation, Cranelift, and the GCC backend. The chapter targets readers who already understand LLVM IR and compiler middle-end concepts; Rust language knowledge is helpful but the text is self-contained.

---

## 177.1 rustc Architecture Overview

### Driver Phases

The rustc driver is not a linear pipeline; it is a demand-driven computation graph. But the high-level stages are recognizable from any production compiler. Invoked as `rustc src/main.rs --edition 2024`, the driver executes the following sequence:

1. **Argument parsing** (`rustc_driver_impl::run_compiler`): command-line flags are normalized into a `Config` struct. Session-level options (target triple, optimization level, panic strategy, codegen backend) are fixed here.

2. **Crate loading** (`rustc_metadata`): `.rlib` and `.rmeta` files for dependencies are located on the search path. Metadata is decoded lazily using the `rustc_metadata::decoder` module.

3. **Parsing** (`rustc_parse`): the source text is lexed into tokens and parsed into an AST. The parser produces an `ast::Crate`. Error recovery is built into the parser using `Parser::recover_*` methods; rustc reports as many errors as possible before stopping.

4. **Macro expansion** (`rustc_expand`): proc-macros, `macro_rules!`, built-in derives, and compiler plugins are expanded. The result is a fully-expanded AST whose spans point back to original source positions.

5. **Name resolution** (`rustc_resolve`): identifiers are bound to items; `use` declarations are resolved; the visibility of every item is established.

6. **HIR lowering** (`rustc_ast_lowering`): the expanded AST is lowered to the High-level IR (HIR). HIR retains explicit types, lifetimes, and pattern structures. It is the primary IR for type checking.

7. **Type checking** (`rustc_hir_analysis`, `rustc_hir_typeck`): Hindley-Milner type inference extended with trait bounds. The trait solver (currently the "new" solver enabled by `-Z next-solver`) resolves trait obligations. Lifetime elaboration annotates every reference with a concrete region.

8. **THIR construction** (`rustc_mir_build`): HIR is lowered first to the Typed HIR (THIR), which makes all coercions and pattern bindings explicit. THIR is never serialized; it exists only during MIR building.

9. **MIR building** (`rustc_mir_build`): THIR is lowered to MIR. Each function body becomes a `mir::Body` containing a vector of `BasicBlockData`.

10. **MIR optimization** (`rustc_mir_transform`): a pass pipeline runs over MIR before codegen. Passes include `ConstProp`, `SimplifyLocals`, `InstCombine`, `Inline`, and others described in §177.2.

11. **Codegen** (`rustc_codegen_llvm`, `rustc_codegen_cranelift`, or `rustc_codegen_gcc`): each `mir::Body` is translated to the target backend's IR and compiled to object code.

The key references are [compiler/rustc_driver_impl/src/lib.rs](https://github.com/rust-lang/rust/blob/master/compiler/rustc_driver_impl/src/lib.rs) for the top-level driver and the [Rust Compiler Dev Guide — Overview](https://rustc-dev-guide.rust-lang.org/overview.html).

### HIR and THIR in Detail

**HIR** (High-level IR) is produced by `rustc_ast_lowering` and stored as `hir::Crate`. Unlike the AST, HIR has a flat map from `HirId` to `Node` — there is no ownership tree. This enables efficient lookup of any node by ID. HIR nodes include `Item` (function, struct, enum, trait, impl), `Expr`, `Pat`, `Ty`, `Generics`, and `Body`. Every `ItemId` in HIR has a corresponding `DefId` in the global definition table.

Key HIR properties:
- Desugared: `for` loops are desugared to `loop { match <IntoIterator>::next(iter) { Some(x) => ..., None => break } }`. `?` is desugared to a `match` on `ControlFlow`. `async fn` is desugared to a function returning `impl Future`.
- Lifetimes are elaborated: anonymous lifetimes (`&T`) are given fresh lifetime variables. `impl Trait` in argument position generates a fresh implicit lifetime parameter.
- `#[derive(...)]` macros are expanded before HIR construction; the derived `impl` blocks appear as regular HIR items.

**THIR** (Typed HIR) is a transient representation, constructed in `rustc_mir_build::thir` and immediately consumed during MIR building. THIR makes all implicit coercions explicit as `Expr::Cast` nodes and gives every expression a `Ty`. Pattern bindings in THIR are fully resolved: `&(a, b)` in a pattern becomes an explicit dereference followed by a tuple destructure. THIR is the level at which exhaustiveness checking runs (`rustc_mir_build::check_match`) — the checker walks THIR patterns to determine whether a `match` expression covers all cases.

THIR is never stored in the query cache; querying `tcx.thir_body(def_id)` constructs it from HIR on demand and immediately consumes it. This keeps memory usage bounded.

### The Query System

The critical insight in rustc's design is that every computation is expressed as a *query*: a named, memoized function from a key to a value, where the key is typically a `DefId` (a cross-crate identifier for any named Rust item). The query system is implemented in `rustc_middle::query` and exposed through the `TyCtxt` type, which is the central context object passed everywhere in the compiler.

A query is declared with the `rustc_queries!` macro:

```rust
rustc_queries! {
    query mir_borrowck(key: LocalDefId) -> &'tcx BorrowCheckResult<'tcx> {
        desc { |tcx| "borrow-checking `{}`", tcx.def_path_str(key) }
    }
}
```

When `tcx.mir_borrowck(def_id)` is called, the query infrastructure checks whether the result is cached. If so, it returns the cached value. If not, it runs the query's provider function, stores the result, and returns it. Circular queries are detected and reported as errors.

The query system enables **incremental compilation** via the *red-green algorithm*: each query result is fingerprinted. On a subsequent compilation, the system re-runs only queries whose inputs have changed (red nodes); unchanged queries (green nodes) reuse cached results from disk. The dependency graph is stored in `dep_graph/` in the incremental compilation directory.

Two important consequences: (1) rustc performs borrow-checking lazily — `mir_borrowck` is not called unless the function is reachable from a codegen root, and (2) the query system naturally parallelizes across functions because query results are immutable once computed (the `-Z parallel-compiler` flag enables threading the query system, though as of early 2026 it is still not the default).

The [Rust Compiler Dev Guide — Queries](https://rustc-dev-guide.rust-lang.org/query.html) is the authoritative description of this system.

### Dependency Tracking and `DepNode`

Each query has an associated `DepNode` — a node in the dependency graph. When a query is evaluated, every other query it calls becomes a dependency (an edge in the graph). The `DepGraph` is built lazily during compilation and serialized at the end of the session.

`DepNode` variants are defined in [compiler/rustc_middle/src/dep_graph/dep_node.rs](https://github.com/rust-lang/rust/blob/master/compiler/rustc_middle/src/dep_graph/dep_node.rs). Each variant corresponds to a query kind (e.g., `DepKind::mir_borrowck`, `DepKind::type_of`, `DepKind::layout_of`). The node is keyed by its query parameter, hashed to a `Fingerprint`.

On a subsequent incremental build, the system:
1. Loads the previous `DepGraph` from disk.
2. Marks *changed* inputs red (source files, `rustc` version, command-line flags).
3. Propagates redness: if a node's dependency changed, the node becomes red.
4. Re-evaluates all red nodes; their new results are fingerprinted and compared to the previous results. If the fingerprint matches (output unchanged despite changed input), downstream nodes are *not* re-evaluated — this is the "green" outcome.

This *output fingerprint* check is the key to incremental efficiency: a function whose type signature did not change (even though its body file changed) will not force re-type-checking of all callers.

### `TyCtxt`, `Ty<'tcx>`, and the Type Arena

`TyCtxt<'tcx>` (defined in [compiler/rustc_middle/src/ty/context.rs](https://github.com/rust-lang/rust/blob/master/compiler/rustc_middle/src/ty/context.rs)) is an arena-allocated structure. The `'tcx` lifetime is the lifetime of the entire compilation session's arena. All `Ty<'tcx>` values are interned: `tcx.mk_ty(TyKind::Ref(...))` returns a pointer into the arena, and the same logical type always returns the same pointer. Pointer equality is sufficient for type equality — no deep structural comparison is needed. This is the reason the `'tcx` lifetime appears throughout all of rustc's compiler types.

`Ty<'tcx>` is a thin wrapper around an interned `TyKind`:

| `TyKind` variant | Represents |
|-----------------|-----------|
| `Bool`, `Char`, `Int(IntTy)`, `Uint(UintTy)`, `Float(FloatTy)` | Primitive scalars |
| `Str`, `Never` | String slice type, diverging type |
| `Array(Ty, Const)`, `Slice(Ty)` | Aggregate sequences |
| `Tuple(&[Ty])` | Tuple type |
| `Ref(Region, Ty, Mutability)` | Reference type; carries a lifetime |
| `RawPtr(TypeAndMut)` | Raw pointer |
| `Adt(AdtDef, SubstsRef)` | Named type: struct, enum, union |
| `FnPtr(PolyFnSig)` | Function pointer |
| `FnDef(DefId, SubstsRef)` | Zero-sized function item type |
| `Closure(DefId, SubstsRef)` | Closure type |
| `Coroutine(DefId, SubstsRef, Movability)` | Generator/coroutine type |
| `Dynamic(&[Binder<ExistentialPredicate>], Region, DynKind)` | `dyn Trait` object |
| `Param(ParamTy)` | Type parameter (`T` in a generic function) |
| `Bound(DebruijnIndex, BoundTy)` | Higher-ranked bound type |
| `Infer(InferTy)` | Inference variable during type checking |
| `Error(ErrorGuaranteed)` | Propagated type error (suppresses further errors) |

---

## 177.2 MIR: Rust's Mid-Level IR

MIR (Mid-level IR) was introduced in RFC 1211 and first stabilized in the 2016 timeframe. It occupies the position between HIR (which retains high-level Rust syntax structures) and LLVM IR (which is in SSA form and target-aware). The key distinction from LLVM IR is that MIR is *not* in SSA form; instead it uses named *locals* (`_0`, `_1`, ...) that may be assigned multiple times. Borrow checking runs on MIR, so MIR retains lifetime information that LLVM IR does not.

### Core Data Types

The full definitions live in [compiler/rustc_middle/src/mir/mod.rs](https://github.com/rust-lang/rust/blob/master/compiler/rustc_middle/src/mir/mod.rs).

**`Place`**: a storage location — a local variable plus zero or more projections.

```
Place = Local | Place.field(f) | Place[index] | Place as Variant | *Place
```

`Local` is a `u32` index into `Body::local_decls`. Projection operators are: `Field(index, ty)`, `Index(local)`, `ConstantIndex`, `Subslice`, `Downcast(variant_index)`, and `Deref`. A `Place` such as `_3.0.1` means "the second field of the first field of local `_3`".

**`Rvalue`**: a pure computation that produces a value without side effects on the heap (other than through borrows). Key variants:

| Variant | Meaning |
|---------|---------|
| `Use(Operand)` | Copy or move of an operand |
| `Repeat(Operand, count)` | Array repeat |
| `Ref(region, borrow_kind, Place)` | Create a reference — this is where borrow information lives |
| `AddressOf(mutability, Place)` | Raw pointer creation |
| `BinaryOp(op, lhs, rhs)` | Arithmetic, comparison |
| `CheckedBinaryOp(op, lhs, rhs)` | Overflow-checked arithmetic; yields `(T, bool)` |
| `NullaryOp(op, ty)` | `size_of`, `align_of` |
| `UnaryOp(op, operand)` | Negation, `!` |
| `Discriminant(Place)` | Read enum discriminant |
| `Aggregate(kind, operands)` | Construct a struct, tuple, array, enum variant, or closure |
| `ShallowInitBox(Operand, ty)` | Special for `Box::new` |
| `CopyForDeref(Place)` | Deref coercion without move |

**`Operand`**: either `Copy(Place)`, `Move(Place)`, or `Constant(Box<Constant>)`. Moving an operand kills the place — subsequent use of that place is a borrow-check error.

**`Statement`**: a non-branching instruction in a basic block. The dominant variants:

| Variant | Meaning |
|---------|---------|
| `Assign(Place, Rvalue)` | The assignment statement: `_3 = Add(_1, _2)` |
| `StorageLive(Local)` | Marks a local as "storage allocated" for liveness analysis |
| `StorageDead(Local)` | Marks a local as out of scope; storage may be reused |
| `SetDiscriminant(Place, variant)` | Set the discriminant of an enum without writing fields |
| `Deinit(Place)` | Mark a place as deinitialized (used in MIR cleanup) |
| `Retag(kind, Place)` | Instrumentation for Stacked Borrows (inserted by `-Z mir-emit-retag`) |
| `PlaceMention(Place)` | Tells the borrow checker a place was mentioned (for diagnostics) |

**`Terminator`**: the last instruction of a basic block. Controls flow:

| Variant | Successors | Meaning |
|---------|-----------|---------|
| `Goto { target }` | 1 | Unconditional branch |
| `SwitchInt { discr, targets }` | n | Switch on integer value |
| `Return` | 0 | Function return |
| `Unreachable` | 0 | UB if reached |
| `Call { func, args, destination, target, unwind }` | 1–2 | Function call; `target` on success, `unwind` on panic |
| `Assert { cond, expected, msg, target, unwind }` | 1–2 | Runtime assertion (bounds check, overflow) |
| `Drop { place, target, unwind }` | 1–2 | Drop a value |
| `Yield { value, resume, resume_arg, drop }` | 2 | Generator yield |
| `GeneratorDrop` | — | Drop a generator |
| `FalseEdge` | 2 | Borrow-check-only edge; erased before codegen |
| `FalseUnwind` | 2 | Borrow-check-only edge for cleanup |

### MIR CFG and Basic Blocks

A `mir::Body` is a vector of `BasicBlockData`. Each `BasicBlockData` contains:
- `statements: Vec<Statement>` — executed sequentially
- `terminator: Terminator` — determines which block runs next
- `is_cleanup: bool` — true for the blocks that run during stack unwinding

The CFG edges are implicit in the terminator's successor sets. MIR differs from LLVM IR in that there are no phi nodes: instead of φ(b1: v1, b2: v2), MIR uses a writable local that both blocks write before the join point. The borrow checker understands join points via dataflow, not SSA form.

### MIR in Practice: A Dump

Consider this Edition 2024 Rust function:

```rust
fn sum_positive(values: &[i32]) -> i32 {
    let mut total = 0i32;
    for &v in values {
        if v > 0 {
            total = total.saturating_add(v);
        }
    }
    total
}
```

Dump its MIR with:

```bash
rustc +nightly --edition 2024 -Z unpretty=mir src/main.rs 2>&1 | head -80
```

The output (simplified) shows:

```
fn sum_positive(_1: &[i32]) -> i32 {
    let mut _0: i32;            // return place
    let mut _2: i32;            // total
    let mut _3: std::slice::Iter<'_, i32>;
    let mut _4: Option<&i32>;
    let _5: &i32;
    let _6: i32;                // v (the copy of *_5)
    let mut _7: bool;           // v > 0
    let mut _8: i32;            // saturating_add result

    bb0: {
        _2 = const 0_i32;
        _3 = <[i32]>::iter(move _1) -> [return: bb1, unwind: bb7];
    }

    bb1: {
        _4 = <std::slice::Iter<'_, i32> as Iterator>::next(move _3) -> ...;
    }

    bb2: {
        _5 = ((_4 as Some).0: &i32);
        _6 = copy *_5;
        _7 = Lt(const 0_i32, copy _6);
        switchInt(move _7) -> [0: bb4, otherwise: bb3];
    }

    bb3: {
        _8 = i32::saturating_add(copy _2, copy _6) -> [return: bb4, ...];
    }

    bb4: {
        _2 = move _8;
        goto -> bb1;
    }

    bb5: {
        _0 = copy _2;
        return;
    }
    ...
}
```

Note how `StorageLive`/`StorageDead` annotations (elided above for brevity) bracket each local's live range. The borrow checker operates over these live ranges, not over syntactic scopes.

### Non-Lexical Lifetimes (NLL)

Before MIR, Rust's borrow checker operated on the AST, producing famously restrictive errors when borrows appeared to overlap but did not logically. NLL (RFC 2094) moved borrow checking to MIR and inferred *region constraints* from dataflow on MIR basic blocks.

Each `Ref` rvalue in MIR carries a lifetime variable (a *region*). The NLL borrow checker (`rustc_borrowck`) walks the MIR CFG and collects constraints of the form "region R1 must outlive region R2 at program point P". These constraints are solved by a fixed-point iteration over the CFG. The resulting *live ranges* for each borrow are sets of program points where the borrow must remain valid.

NLL is sound: it never accepts unsafe programs. But it has false positives — programs that are safe but that NLL rejects due to the imprecision of its location-insensitive region inference. Polonius (§177.4) addresses this.

### The `mir::Body` Structure in Full

A `mir::Body<'tcx>` ([compiler/rustc_middle/src/mir/mod.rs](https://github.com/rust-lang/rust/blob/master/compiler/rustc_middle/src/mir/mod.rs)) contains:

```
Body<'tcx> {
    basic_blocks: BasicBlocks<'tcx>,    // indexed by BasicBlock (u32)
    source_scopes: IndexVec<SourceScope, SourceScopeData<'tcx>>,
    coroutine: Option<Box<CoroutineInfo<'tcx>>>,  // for async fn / generators
    local_decls: IndexVec<Local, LocalDecl<'tcx>>,
    user_type_annotations: CanonicalUserTypeAnnotations<'tcx>,
    arg_count: usize,                   // number of function parameters
    spread_arg: Option<Local>,          // for variadic ABI
    var_debug_info: Vec<VarDebugInfo<'tcx>>,  // DWARF variable locations
    span: Span,                         // span of the entire function
    required_consts: Vec<ConstOperand<'tcx>>,  // consts that must evaluate
    is_polymorphic: bool,               // false after monomorphization
    tainted_by_errors: Option<ErrorGuaranteed>,
    injection_phase: Option<MirPhase>,
    pass_count: usize,                  // for pass naming in diagnostics
}
```

`LocalDecl` for local `_N` contains `ty: Ty<'tcx>`, `source_info: SourceInfo` (span + scope), and `local_info: Box<LocalInfo<'tcx>>` (explains whether the local is a user variable, a compiler temporary, or an argument). Locals are indexed as: `_0` = return place, `_1` through `_N` = function arguments (N = `arg_count`), remaining locals are temporaries.

`BasicBlocks` is a wrapper that provides efficient predecessor computation and a dominators cache. Predecessor sets are computed lazily on first access; the dominator tree is computed by the standard iterative dataflow algorithm.

### MIR Phases

MIR passes through several phases, tracked by the `MirPhase` enum:

| Phase | Contents | Who runs |
|-------|---------|---------|
| `Built` | Raw output from HIR/THIR lowering; `FalseEdge`/`FalseUnwind` terminators present | `rustc_mir_build` |
| `Promoted` | Promoted constants extracted; `ConstOperand` nodes reference promoted bodies | `rustc_mir_transform::promote_consts` |
| `BorrowChecked` | After `mir_borrowck`; NLL regions attached | `rustc_borrowck` |
| `Runtime(Unoptimized)` | `FalseEdge`/`FalseUnwind` removed; `ElaborateDrops` run | `rustc_mir_transform` |
| `Runtime(Optimized)` | After the full optimization pass list | `rustc_mir_transform` |

Codegen receives `Runtime(Optimized)` MIR. The borrow checker runs on `Built` MIR. Passes that run before borrow checking (in `MirPhase::Built`) must preserve borrow-checking validity; passes that run after have more freedom.

### MIR Optimization Passes

The pass pipeline in `rustc_mir_transform` runs before codegen. Key passes:

| Pass | Effect |
|------|--------|
| `ConstProp` | Propagates `const` values through arithmetic and comparisons |
| `SimplifyLocals` | Removes unused locals and dead assignments |
| `SimplifyCfg` | Merges chains of `goto` blocks; removes unreachable blocks |
| `InstCombine` | Peephole rewrites: `a - 0 → a`, `a * 1 → a`, etc. |
| `Inline` | Inlines small functions (governed by a heuristic inline cost model) |
| `MatchBranchSimplification` | Simplifies `switchInt` with a single target into a `goto` |
| `ElaborateDrops` | Converts `Drop` terminators into calls to the drop glue |
| `GeneratorLower` | Converts generator bodies to state machines |
| `CopyProp` | Propagates copies across assignments |
| `DeadStoreElimination` | Removes stores whose value is never read |

These passes run on *optimized* MIR (after borrow checking), so they may not see the same MIR that the borrow checker operated on. The borrow checker runs on the "built" MIR, before any optimization transforms it.

### Monomorphization

Because Rust is a compiled generics language (not a runtime-parameterized one like Java), every use of a generic function `fn foo<T>(...)` at a concrete type `T = Foo` produces a distinct monomorphized instance. The monomorphization collector (`rustc_monomorphize`) starts from the crate's root items (the `main` function, `#[no_mangle]` functions, and exported items) and recursively discovers all `Instance<'tcx>` values reachable from the call graph.

An `Instance<'tcx>` is a pair `(DefId, SubstsRef<'tcx>)` — a definition plus its type substitution. The monomorphization pass queries `tcx.optimized_mir(instance.def_id())` and substitutes `SubstsRef` into all type variables in the MIR, producing a monomorphized `mir::Body` that contains no `TyKind::Param` types. This body is then handed to the codegen backend.

For cross-crate generics, monomorphized MIR bodies are *not* cached in `.rlib` metadata — only the generic body is. The calling crate re-monomorphizes from the generic MIR. This is why generic functions must be available for downstream crates to monomorphize (hence the "generic implementations must be in header files" analogy from C++ templates — though Rust avoids the dual-compilation-unit problem by storing MIR in metadata).

Closures and coroutines also go through monomorphization. An `async fn` is desugared to a coroutine type whose `poll` method is the closure body. The coroutine state machine (lowered by `GeneratorLower` or `CoroutineLower` in MIR) is a struct with one field per local that must live across a yield point, plus a discriminant field for the current suspend point.

---

## 177.3 Miri: The MIR Interpreter

Miri is an interpreter for MIR that executes Rust programs at the MIR level rather than at machine code level. Because it interprets every memory operation abstractly, it can detect undefined behavior that sanitizers miss: use of uninitialized memory (not just stack — heap too), integer overflow in debug mode, out-of-bounds pointer arithmetic, type-confused casts, and memory model violations.

### Architecture: `InterpCx<MiriMachine>`

The interpreter is structured as a generic interpreter framework (`rustc_const_eval::interpret::InterpCx`) parameterized by a `Machine` trait. Miri provides `MiriMachine` as its `Machine` implementation, found in [src/tools/miri/src/machine.rs](https://github.com/rust-lang/rust/blob/master/src/tools/miri/src/machine.rs).

`InterpCx` manages:
- A **stack** of `Frame`s, each holding the current `mir::Body`, the program counter (current basic block + statement index), and the local variable map.
- An **abstract heap** (`Allocation` objects): each allocation has a backing byte array, a provenance map (which pointer tags are valid at which byte offsets), and a boolean mask for initialized bytes.
- A **memory** module that enforces alignment, provenance, and bounds on every read and write.

`MiriMachine` extends this with:
- A **global state** for detecting data races (thread metadata, vector clocks for the TSan-style analysis enabled by `-Zmiri-preemption-rate`).
- A **Stacked Borrows** (or Tree Borrows) state: per-allocation tag stacks.
- A **symbol table** for extern functions: Miri provides shims for many libc functions, and custom shims can be written using `#[no_mangle]`.

### What Miri Detects

Running `cargo miri test` on a crate with a use-after-free:

```rust
fn dangling() -> &'static i32 {
    let x = 42i32;
    unsafe { &*(&x as *const i32) }
}

#[test]
fn test_ub() {
    let r = dangling();
    println!("{}", *r);
}
```

```bash
$ cargo miri test
error: Undefined Behavior: pointer to alloc1234 was dereferenced after this
       allocation got freed
   --> src/lib.rs:7:20
    |
  7 |     println!("{}", *r);
    |                    ^^ pointer to alloc1234 was dereferenced...
    |
    = note: this error was reported by Miri; see
            https://doc.rust-lang.org/nightly/unstable-book/compiler-flags/miri.html
```

Miri detects:

| Category | Detection mechanism |
|----------|-------------------|
| Use-after-free (stack or heap) | Deallocated allocations retain their ID; dereference of a freed pointer is caught |
| Uninitialized memory read | Per-byte initialization mask; reading an uninitialized byte is UB |
| Out-of-bounds access | Every pointer carries its allocation size; pointer arithmetic beyond bounds is caught |
| Invalid enum discriminant | On read of a place of enum type, the discriminant is validated |
| Misaligned pointer dereference | Alignment is tracked per allocation; checked on every access |
| Data races | Vector-clock analysis (TSan algorithm) when `-Zmiri-preemption-rate=0.01` is set |
| Stacked Borrows violations | See below |

### Stacked Borrows Memory Model

Stacked Borrows ([Ralf Jung's dissertation, 2020](https://research.ralfj.de/phd/thesis-screen.pdf)) is the memory model Miri enforces by default. The key idea is that every allocation maintains a *stack of items*, where each item is a `(tag, permission)` pair. A *tag* is a unique identifier assigned to each borrow; permissions are `Unique` (writable, exclusive), `SharedReadOnly` (readable, may coexist with other `SharedReadOnly`), or `Disabled` (no longer valid).

When a mutable reference `&mut T` is created from a raw pointer or another `&mut T`:
1. A new tag is generated.
2. A `Unique` item with the new tag is *pushed* onto the stack for every byte of the pointee.
3. Items below the new top that are incompatible with the new borrow (e.g., previously pushed `Unique` items) are popped or invalidated.

When a shared reference `&T` is created:
1. A new tag is generated.
2. A `SharedReadOnly` item is pushed.

When a borrow is used (read or write), the stack is searched top-to-bottom for the item with the matching tag. If found, all items above it that are incompatible with the access are popped. If the access is a write and the tag has `SharedReadOnly` permission, UB is triggered.

This enforces the aliasing invariant: a `&mut T` reference is always the only live pointer to its referent while it is being used.

A concrete Stacked Borrows violation that Miri catches:

```rust
fn stacked_borrows_violation() {
    let mut x = 42i32;
    let raw: *mut i32 = &mut x as *mut i32;
    let r: &mut i32 = unsafe { &mut *raw };
    // At this point the stack for x is: [Unique(raw_tag), Unique(r_tag)]
    // r_tag is on top; raw_tag is Disabled for write access
    *r = 1;                        // OK: r_tag is on top
    unsafe { *raw = 2; }           // Stacked Borrows violation:
                                   // raw_tag was Disabled when r was created
    println!("{}", *r);
}
```

```bash
$ cargo miri run
error: Undefined Behavior: attempting a write access using <raw_tag> at ...
       but that tag does not exist in the borrow stack for this location
```

LLVM can and does exploit Stacked Borrows semantics in optimization: if a function receives `&mut i32`, LLVM annotates it with `noalias`, allowing it to hoist loads out of loops under the assumption that no other path modifies the pointed-to memory. The Stacked Borrows model is the formal justification for this optimization.

### Tree Borrows

Tree Borrows (Neven-Yang et al., 2023) is an evolving successor to Stacked Borrows, accessible via `-Zmiri-tree-borrows` (still opt-in as of April 2026). Instead of a per-allocation stack, Tree Borrows maintains a *tree* of pointer relationships rooted at the original allocation. Each node represents a borrow and carries a permission state: `Active`, `Frozen`, `Disabled`, or `Reserved`.

The key advantage over Stacked Borrows: Tree Borrows accepts certain patterns that Stacked Borrows incorrectly rejects, particularly around two-phase borrows and interior mutability with raw pointers. It is designed to be the long-term foundation for Rust's aliasing model.

### Using Miri

```bash
rustup component add miri
cargo miri test
cargo miri run

MIRIFLAGS="-Zmiri-tree-borrows" cargo miri test
MIRIFLAGS="-Zmiri-preemption-rate=0.01" cargo miri test
MIRIFLAGS="-Zmiri-ignore-leaks" cargo miri test
```

Providing an extern stub in Edition 2024 syntax:

```rust
unsafe extern "C" {
    fn my_c_function(x: i32) -> i32;
}

#[cfg(miri)]
#[no_mangle]
unsafe extern "C" fn my_c_function(x: i32) -> i32 {
    x * 2
}
```

Note the `unsafe extern "C" { ... }` block syntax, which is the Edition 2024 form (previously `extern "C" { ... }` was implicitly unsafe; Edition 2024 requires the explicit `unsafe` annotation on extern blocks per RFC 3484).

**Limitations**: Miri cannot execute FFI calls to non-stub functions; inline assembly is not supported (the interpreter has no machine model); Miri runs roughly 100×–1000× slower than native code, making it impractical for large integration test suites but excellent for unit tests and focused UB investigations.

### Miri vs Sanitizers: Complementary Coverage

Miri and compile-time sanitizers (AddressSanitizer, ThreadSanitizer, MemorySanitizer) detect overlapping but distinct categories of bugs:

| Capability | Miri | ASan | TSan | MSan |
|-----------|------|------|------|------|
| Heap use-after-free | Yes | Yes | — | — |
| Stack use-after-return | Yes | Partial | — | — |
| Uninitialized memory read | Yes (exact) | No | — | Yes (shadow) |
| Data races | Yes (all paths) | — | Yes (runtime) | — |
| Integer overflow (debug) | Yes | No | — | — |
| Stacked/Tree Borrows violations | Yes | No | No | No |
| FFI functions | Stub only | Yes | Yes | Yes |
| Inline assembly | No | Yes | Yes | Yes |
| Overhead | 100×–1000× | 2×–5× | 5×–15× | 3×–5× |
| Requires native execution | No | Yes | Yes | Yes |

ASan works at the machine level and catches bugs in FFI code and inline assembly that Miri cannot reach. Miri works at the MIR level and detects memory model violations (Stacked Borrows, uninitialized reads) with precision that ASan cannot achieve. The recommended strategy is to run `cargo miri test` for unit tests and `cargo test` with `RUSTFLAGS="-C sanitizer=address"` for integration/system tests.

### Const Evaluation: `InterpCx` Without `MiriMachine`

The same `InterpCx` framework that powers Miri also powers `const` evaluation (`rustc_const_eval`). The difference is the `Machine` implementation: the `const` evaluator uses `CompileTimeMachine`, which is more restrictive than `MiriMachine` — it disallows raw pointer arithmetic, mutable static access, and non-const function calls. Anything evaluated at compile time (in `const` items, const generics, `static` initializers, or `const fn` calls evaluated at the call site) goes through `InterpCx<CompileTimeMachine>`. This means the interpreter framework is shared; only the policy on which operations are permitted differs.

---

## 177.4 Polonius: The New Borrow Checker

NLL is sound but not complete. The canonical false-positive example:

```rust
fn get_or_insert<'a>(map: &'a mut HashMap<u32, u32>, key: u32) -> &'a u32 {
    if let Some(v) = map.get(&key) {
        return v;  // NLL error: `map` is borrowed mutably and immutably
    }
    map.insert(key, 0);
    map.get(&key).unwrap()
}
```

NLL rejects this because after `map.get(&key)` returns `Some(v)`, the immutable borrow of `map` (for the lifetime of `v`) appears to overlap with the mutable borrow that `map.insert` requires — even though the two code paths are mutually exclusive. The program is safe, but NLL's region inference is not flow-sensitive enough to see this.

Polonius reformulates borrow checking as a Datalog program, enabling flow-sensitive analysis.

### The Datalog Formulation

The Polonius analysis is defined in terms of three *base relations* and a set of derived relations. The base relations are inputs derived from the MIR:

| Relation | Meaning |
|----------|---------|
| `loan_issued_at(origin, loan, point)` | At `point`, `origin` was created by issuing `loan` |
| `loan_killed_at(loan, point)` | Loan is killed (storage location overwritten) at `point` |
| `loan_invalidated_at(point, loan)` | At `point`, a conflicting access to the loan's place occurs |
| `cfg_edge(point1, point2)` | MIR CFG edge |
| `var_used_at(var, point)` | Variable is used at `point` |
| `var_defined_at(var, point)` | Variable is defined (overwritten) at `point` |
| `var_dropped_at(var, point)` | Variable is dropped at `point` |

From these, Polonius derives:

```prolog
% An origin is live at a point if some variable using it is live there
origin_live_on_entry(origin, point) :-
    var_used_at(var, point),
    var_origin_live_on_entry(var, origin, point).

% A loan is live if its origin is live and the loan hasn't been killed
loan_live_at(loan, point) :-
    loan_issued_at(origin, loan, point_issued),
    origin_contains_loan_on_entry(origin, loan, point),
    origin_live_on_entry(origin, point).

% Error: an invalidated loan is still live
errors(loan, point) :-
    loan_invalidated_at(point, loan),
    loan_live_at(loan, point).
```

A program is accepted if and only if the `errors` relation is empty.

The [Polonius book](https://rust-lang.github.io/polonius/the_algorithm/rules.html) defines the complete rule set.

### Solving Strategies

Polonius provides three solving strategies, selected at compile time:

**Location-insensitive** (`LocationInsensitive`): ignores program points; propagates loans everywhere an origin is mentioned. Fast (linear), but produces many false positives. Used as a pre-filter: if a program is accepted by the location-insensitive solver, it is definitely accepted by the precise solvers.

**Naive** (`Naive`): straightforward Datalog evaluation using immediate consequences — the standard bottom-up fixpoint. Correct and complete but O(n³) in the size of the program. Impractical for large codebases.

**DataFrog** (`DataFrog`): the production algorithm, implemented using the `datafrog` crate. It computes a hybrid: location-insensitive propagation first, then targeted location-sensitive refinement only where the location-insensitive solver found potential errors. This achieves near-linear performance in practice while remaining sound.

### Activating Polonius

```bash
rustc +nightly --edition 2024 -Z polonius=next src/main.rs   # Polonius v2 (a-mir-formality-based)
rustc +nightly --edition 2024 -Z polonius=yes  src/main.rs   # Polonius v1 (Datalog, DataFrog engine)
```

`-Z polonius=next` (Polonius v2) is the integration of Polonius with the new trait solver and a-mir-formality. It is still under active development as of April 2026. `-Z polonius=yes` is the stable Datalog implementation, passing the full test suite.

The false-positive `get_or_insert` example above is accepted by Polonius v1 because the Datalog encoding tracks which loans are live on which CFG paths, making the mutual exclusion of the two branches visible.

### NLL Constraint Solving in Detail

The NLL borrow checker in `rustc_borrowck` runs in two phases. **Phase 1 — constraint generation**: the checker walks the MIR and generates `RegionConstraint` instances. The constraint `RegionConstraint::Outlives(r1, r2, location)` asserts that region `r1` must contain at least every point that `r2` contains at `location`. Constraints arise from:
- Subtyping: a reference with lifetime `'short` assigned to a slot expecting lifetime `'long` generates `'long: 'short` at the assignment point.
- Function signatures: a call to `fn foo<'a>(x: &'a T) -> &'a U` at call site `p` generates constraints between the actual argument lifetimes and `'a`.
- `Ref` rvalues: creating `&'r T` at point `p` generates a constraint that `r` must contain `p` and all points where the borrow is live.

**Phase 2 — solving**: constraints are solved by a Bellman-Ford-style fixed-point iteration over the CFG. Each region variable starts as an empty set of program points. The solver iterates until no region grows: if `r1: r2` at point `p` and `p ∈ r2`, then `p ∈ r1`. After solving, the checker verifies that no two conflicting borrows are live at the same program point — mutable-immutable overlap, or two mutable borrows of the same place.

The NLL constraint solver is described in [compiler/rustc_borrowck/src/region_infer/mod.rs](https://github.com/rust-lang/rust/blob/master/compiler/rustc_borrowck/src/region_infer/mod.rs) and the [NLL RFC](https://rust-lang.github.io/rfcs/2094-nll.html).

### The `use<>` Precise Capture Syntax (Edition 2024)

Edition 2024 introduces `use<>` syntax for precise lifetime capture in `impl Trait` return types, which directly addresses an NLL/region-inference interaction. In pre-2024 editions, an `impl Trait` return type in a function with lifetime parameters captures *all* lifetimes in scope, even if the returned type does not actually use them. This caused borrow-check errors when the caller tried to use values whose lifetimes were spuriously captured.

With Edition 2024:

```rust
fn first_positive<'a, 'b>(
    values: &'a [i32],
    fallback: &'b i32,
) -> impl use<'a> Iterator<Item = i32> {
    values.iter().copied().filter(|&x| x > 0)
}
```

The `use<'a>` annotation tells the compiler that the returned `impl Iterator` captures only `'a` (the slice lifetime), not `'b`. The caller is free to drop the `fallback` reference immediately after the call, even while the iterator is still live. Without `use<'a>`, both lifetimes would be captured and the caller would be forced to hold `fallback` as long as the iterator is used. This is a direct reduction in false-positive borrow errors driven by lifetime inference improvements.

---

## 177.5 a-mir-formality: The Formal Model

a-mir-formality ([GitHub](https://github.com/rust-lang/a-mir-formality)) is a mechanized formal model of Rust's core type system — traits, lifetimes, and MIR typing — written in PLT Redex (a domain-specific language for operational semantics embedded in Racket). It is not a toy: it is intended to be the authoritative specification of what the Rust type system *should* accept, driving the implementation of Polonius v2 and the next-generation trait solver.

### What It Formalizes

The formalism covers:

- **Rust's trait system**: trait definitions, `impl` blocks, associated types, where-clauses, and the coherence rules that prevent overlapping impls. The judgment `(rust-trait-ref-ok Γ TraitRef)` asserts that `TraitRef` holds under context `Γ`.

- **Well-formedness**: the judgment `(rust-type-ok Γ T)` asserts that type `T` is well-formed — all its lifetime and type parameters satisfy their bounds.

- **Borrow-checking judgments**: the formalism includes a simplified MIR type system with region constraints. A MIR `Body` is well-typed if each statement and terminator can be given a type derivation under the borrow environment.

- **Lifetime outlives reasoning**: the judgment `(rust-outlives Γ r1 r2)` asserts that region `r1` outlives `r2` under context `Γ`. This corresponds to the `'a: 'b` constraint syntax.

### Relationship to Chalk and the Next-Gen Trait Solver

**Chalk** was the first attempt at a Prolog-style formalization of Rust's trait system (ca. 2017–2021). It represented trait queries as Horn clauses and solved them via SLD resolution. Chalk identified many correctness issues in the existing trait solver and introduced the concept of *canonical queries* (queries with universally quantified variables, enabling caching across instantiations). However, Chalk's architecture proved difficult to integrate into rustc incrementally.

The **next-generation trait solver** (`rustc_trait_selection::solve`, enabled by `-Z next-solver`) is a reimplementation in Rust that incorporates Chalk's canonical-query insight while being directly integrated into the rustc codebase. It uses the same `Canonical<T>` wrapper for queries and handles higher-ranked lifetimes and associated type normalization more correctly than the old solver.

a-mir-formality supersedes Chalk as the specification: rather than a working Prolog implementation, it is a mathematical model that the trait solver is required to agree with. Discrepancies between a-mir-formality's judgments and the trait solver's behavior are tracked as bugs. The formality drives Polonius v2 by providing the precise definition of what "a borrow is live" means at the type-theoretic level.

### Using the Formality

The formality is not user-facing. To experiment with it:

```bash
git clone https://github.com/rust-lang/a-mir-formality
cd a-mir-formality
racket src/tests/run-pass/trait-matching/basic-impl.rkt
```

Each test file encodes a Rust program fragment as a Redex term and checks that the relevant judgment holds (or fails with the expected error). The formality does not execute; it only type-checks.

### Trait Solver Architecture in Brief

The **next-generation trait solver** (`-Z next-solver`, stabilizing gradually as of 2026) solves Rust's trait obligations using a goal-directed search. A *goal* is a `Goal<Predicate>` — a predicate (e.g., `T: Display`, `<T as Iterator>::Item = u32`) in an inference context. The solver calls `rustc_trait_selection::solve::EvalCtxt::evaluate_goal`, which:

1. Canonicalizes the goal: replaces inference variables with universally quantified placeholders, producing a `Canonical<Goal<Predicate>>`.
2. Looks up the canonical goal in the solver cache.
3. If not cached, dispatches to a `GoalKind`-specific assembly step that collects *candidates* (all `impl` blocks and where-clauses that could possibly satisfy the goal).
4. Evaluates each candidate, filtering to those that succeed. If exactly one candidate succeeds, the goal is solved. If zero, the goal fails. If more than one, the solver reports ambiguity (usually resolved later when more types are inferred).

The key correctness improvement over the old solver: the new solver handles *coinductive* cycles correctly. A `T: Auto` bound (e.g., `T: Send + Sync`) is coinductively cyclic — `Vec<T>: Send` if `T: Send`, which is proved by applying `Vec`'s `Send` impl, which requires `T: Send` again. The old solver handled this by special-casing auto traits; the new solver handles it generically via a tabling mechanism similar to XSB Prolog's tabled SLD resolution.

---

## 177.6 rustc_codegen_llvm: MIR → LLVM IR

The LLVM backend is in [compiler/rustc_codegen_llvm/](https://github.com/rust-lang/rust/blob/master/compiler/rustc_codegen_llvm/). It implements the `CodegenBackend` trait, which requires: `provide(providers)`, `codegen_crate(tcx, ...)`, `join_codegen(...)`, and `link(sess, ...)`.

### Context Hierarchy

Codegen is structured around two lifetime-parameterized context types:

**`CodegenCx<'ll, 'tcx>`** ([src/context.rs](https://github.com/rust-lang/rust/blob/master/compiler/rustc_codegen_llvm/src/context.rs)): owns one LLVM `Module`. The `'ll` lifetime is tied to the LLVM thread-local context; `'tcx` is the rustc type context lifetime. `CodegenCx` holds:
- `module: &'ll llvm::Module`
- `tcx: TyCtxt<'tcx>` — the rustc type-checking context
- `llvm_type(ty)` — cache from `Ty<'tcx>` to `&'ll Type`
- `get_fn(instance)` — cache from `Instance<'tcx>` to `&'ll Value`
- `vtables: FxHashMap<...>` — vtable constants, generated lazily

**`FunctionCx<'a, 'll, 'tcx>`** ([src/mir/mod.rs](https://github.com/rust-lang/rust/blob/master/compiler/rustc_codegen_llvm/src/mir/mod.rs)): owns one LLVM function, built during MIR translation. Contains:
- `cx: &'a CodegenCx<'ll, 'tcx>`
- `llfn: &'ll Value` — the LLVM function value
- `mir: &'tcx mir::Body<'tcx>`
- `locals: IndexVec<mir::Local, LocalRef<'ll>>` — maps each MIR local to either an `OperandRef` (a value in a register) or a `PlaceRef` (a pointer to stack-allocated storage)
- `llbbs: IndexVec<mir::BasicBlock, &'ll BasicBlock>` — maps MIR basic blocks to LLVM basic blocks

### The Translation Loop

[compiler/rustc_codegen_llvm/src/builder.rs](https://github.com/rust-lang/rust/blob/master/compiler/rustc_codegen_llvm/src/builder.rs) implements the `Builder` type wrapping `LLVMBuilderRef`. Translation proceeds:

1. For each MIR `BasicBlock`: create an LLVM `BasicBlock` with `LLVMAppendBasicBlock`.
2. For each `Statement` in the block: call `codegen_statement`, which dispatches on the statement kind. `Assign` is the common case: translate the rvalue to an `OperandRef` or `PlaceRef` using `codegen_rvalue`, then store it at the destination place using `codegen_place`.
3. For the `Terminator`: call `codegen_terminator`. `Call` terminators become LLVM call or invoke instructions (invoke for unwind-capable calls); `Return` becomes `LLVMBuildRet`; `SwitchInt` becomes `LLVMBuildSwitch`; `Drop` terminators have already been elaborated into calls to drop glue by the `ElaborateDrops` MIR pass.

**`LocalRef` and register promotion**: a MIR local that is used only once and whose address is never taken is allocated as an `OperandRef` — a logical SSA value, potentially in a register. A local whose address is taken or which is written multiple times on different CFG paths is allocated as a `PlaceRef` — an `alloca` on the LLVM stack. LLVM's `mem2reg` pass (part of the standard `-O1` pipeline) promotes most `PlaceRef` allocas to SSA values, recovering SSA form from MIR's non-SSA representation. This is the key bridge: MIR uses mutable locals, but after `mem2reg`, the LLVM IR is in SSA form.

**Type translation** uses `CodegenCx::llvm_type(ty)`:

| Rust type | LLVM type |
|-----------|-----------|
| `bool` | `i1` |
| `i8`/`u8` | `i8` |
| `i32`/`u32` | `i32` |
| `i64`/`u64` | `i64` |
| `f32`/`f64` | `float`/`double` |
| `*const T` / `*mut T` | `ptr` (opaque pointer) |
| `&T` / `&mut T` | `ptr` (with `nonnull` + `dereferenceable` attrs) |
| `[T; N]` | `[N x llvm_type(T)]` |
| struct `{ f1: T1, f2: T2 }` | `{ llvm_type(T1), llvm_type(T2) }` |
| `dyn Trait` | `{ ptr, ptr }` (data + vtable) |
| `Option<Box<T>>` (niche) | `ptr` (as scalar) |

All pointer types use LLVM's opaque pointer type (`ptr`) since LLVM 15+, following the removal of typed pointers. rustc explicitly opted into opaque pointers and has no legacy typed-pointer code paths. See [Chapter 4 — LLVM IR](../part-04-llvm-ir/ch04-llvm-ir.md) for the opaque pointer rationale.

### ADT Layout and the `TyAndLayout` Type

Rust's type layout is computed by `rustc_target::abi::layout` and cached in the query system as `tcx.layout_of(ty)`, returning a `TyAndLayout<'tcx>`. The `Layout` contains:
- `size: Size` — total byte size
- `align: AbiAndPrefAlign` — ABI alignment and preferred alignment
- `abi: Abi` — either `Scalar(scalar)`, `ScalarPair(s1, s2)`, `Vector { element, count }`, or `Aggregate { sized }`
- `variants: Variants` — for enums: either `Single { index }` (single-variant; no discriminant needed) or `Multiple { tag, tag_encoding, tag_field, variants }` (multi-variant)
- `fields: FieldsShape` — `Primitive`, `Arbitrary { offsets, memory_index }`, or `Array { stride, count }`

**Niche optimization** exploits unused bit patterns in a field to encode the discriminant without extra space. The canonical example: `Option<Box<T>>`. `Box<T>` is a non-null pointer; the null bit pattern is unused. The `Option` layout stores `None` as the null pointer and `Some(box)` as the pointer itself, making `Option<Box<T>>` exactly pointer-sized:

```rust
fn id(x: Option<Box<i32>>) -> Option<Box<i32>> { x }
```

The LLVM IR for this function (simplified, at `-O1`):

```llvm
define noundef nonnull ptr @id(ptr noundef %x) unnamed_addr #0 {
start:
  ret ptr %x
}
```

The entire function reduces to a pass-through: `Option<Box<i32>>` is represented as a single pointer (`ptr`), where null is `None` and any non-null value is `Some`. The discriminant check at the use site is a `icmp eq %ptr, null` instruction. No extra word is needed, and the discriminant is never materialized as a field.

For enums without a niche (e.g., `Result<i32, i32>`), the layout uses a tagged union: a discriminant field (usually `i8` or `i32`) followed by a union of the variant payloads.

### Trait Object Fat Pointers

A `dyn Trait` object is represented as a *fat pointer*: a `ScalarPair` of `(data_ptr: *mut (), vtable_ptr: *const VTable)`. The vtable layout is:

| Offset | Contents |
|--------|---------|
| 0 | Drop glue function pointer |
| 1 | `size_of::<ConcreteType>()` |
| 2 | `align_of::<ConcreteType>()` |
| 3..n | Method pointers (in declaration order from the trait definition) |

The vtable is emitted as a constant LLVM global. Multiple `dyn Trait` objects for the same concrete type share the same vtable (it is deduplicated by the linker via COMDAT).

### Rust-Specific LLVM Attributes

rustc emits several LLVM function and parameter attributes that enable optimizations:

| Attribute | Where applied | Meaning |
|-----------|--------------|---------|
| `nounwind` | Function | Rust functions with `panic=abort` never unwind; LLVM can omit landing pad infrastructure |
| `noalias` | `&mut T` parameters | The referenced memory is not aliased by any other pointer in scope; enables LLVM's alias-analysis-dependent transforms |
| `dereferenceable(N)` | Reference parameters | The first N bytes of the pointee are safe to read without a null check; N = `size_of::<T>()` for `&T` |
| `noundef` | Return values and parameters | The value is always initialized (never LLVM poison); enables value-range optimizations |
| `align(N)` | Reference parameters | The pointer is at least N-byte aligned |
| `nonnull` | Reference parameters | The pointer is never null (Rust references are guaranteed non-null) |
| `readonly` | `&T` parameters | The function does not write through the pointer (shared reference) |

These attributes are set in [compiler/rustc_codegen_llvm/src/attributes.rs](https://github.com/rust-lang/rust/blob/master/compiler/rustc_codegen_llvm/src/attributes.rs).

### Drop Glue and `ElaborateDrops`

Rust's ownership model means that any local of a non-`Copy` type must be dropped when it goes out of scope, including along panic unwind paths. MIR handles this via the `ElaborateDrops` pass, which runs before codegen on `Runtime(Unoptimized)` MIR.

A MIR `Drop { place, target, unwind }` terminator does not itself call the destructor — `ElaborateDrops` replaces it with a conditional call to *drop glue*. Drop glue is a synthesized MIR function that:
1. Checks a *drop flag* (a `bool` stored in the local's shadow slot): if the local has been moved out of, the drop flag is false and drop glue skips it.
2. For composite types (structs, enums, tuples), calls the drop glue for each field.
3. For types implementing `Drop`, calls the user's `drop()` method before dropping fields.

Drop glue is a key reason that Rust's code size can be larger than equivalent C++ with manual RAII: each type with a non-trivial destructor gets a distinct drop glue function. `--emit=llvm-ir` with `-C opt-level=0` will show these `_ZN4core4mem4drop...` glue functions in the output.

For `dyn Trait` objects, the vtable's first entry is the drop glue pointer for the concrete type, allowing `Box<dyn Trait>` to be dropped without knowing the concrete type at the drop site.

### A Worked Example: `fn max_pair`

Consider a function that returns the larger of two pairs (demonstrating struct layout, inlining, and LLVM attribute emission):

```rust
#[derive(Clone, Copy)]
struct Pair { x: i64, y: i64 }

#[inline(never)]
fn max_pair(a: Pair, b: Pair) -> Pair {
    if a.x + a.y >= b.x + b.y { a } else { b }
}
```

The LLVM IR (at `-O1`, simplified):

```llvm
%Pair = type { i64, i64 }

define noundef %Pair @max_pair(%Pair %a, %Pair %b) unnamed_addr #0 {
start:
  %a.x = extractvalue %Pair %a, 0
  %a.y = extractvalue %Pair %a, 1
  %b.x = extractvalue %Pair %b, 0
  %b.y = extractvalue %Pair %b, 1
  %sum_a = add i64 %a.x, %a.y
  %sum_b = add i64 %b.x, %b.y
  %cmp   = icmp sge i64 %sum_a, %sum_b
  %res   = select i1 %cmp, %Pair %a, %Pair %b
  ret %Pair %res
}

attributes #0 = { nounwind nonlazybind uwtable }
```

Points of note:
- `Pair` is passed by value as an LLVM struct, not as a pointer (the `Scalar` ABI applies for small structs on SysV AMD64).
- `noundef` on the return: the returned value is always initialized because both `a` and `b` are `Copy` types passed by value with no uninitialized fields.
- `nounwind` is present because the crate was compiled with `-C panic=unwind` but the function contains no panic points.
- No `noalias`: by-value struct arguments are not pointers, so the attribute doesn't apply.

---

## 177.7 Panics, Unwinding, and the Personality Function

### `-C panic=unwind` (Default)

When a Rust function calls `panic!()`, the call chain is:
1. `std::panicking::begin_panic` or `core::panic::panic_fmt`
2. `__rust_start_panic(payload: *mut dyn PanicPayload)` — the Rust panic runtime hook
3. The registered panic runtime (either `std::panic` with backtrace, or a custom runtime via `#[panic_handler]`) calls `_Unwind_RaiseException` with a Rust-specific exception object.
4. The Itanium C++ ABI unwinding machinery walks up the stack, calling each frame's *personality function* to decide whether the frame handles the exception.
5. The personality function registered by rustc is `rust_eh_personality`, found in `library/panic_unwind/src/gcc.rs`. It calls `_Unwind_GetLanguageSpecificData` to read the LSDA (Language-Specific Data Area) for the frame and determines whether a landing pad (a `catch_unwind` boundary) is present.
6. If a landing pad is found, `_Unwind_RaiseException` transfers control there. The landing pad calls `__rust_panic_cleanup` to extract the payload and resume normal Rust execution.

LLVM emits `invoke` instead of `call` for any function call that could unwind, with the second target being a landing pad basic block. The landing pad uses `landingpad` instruction with the `cleanup` keyword:

```llvm
%ret = invoke noundef i32 @might_panic()
        to label %continue unwind label %cleanup

cleanup:
  %exn = landingpad { ptr, i32 }
    cleanup
  call void @__rust_drop_all_locals()
  resume { ptr, i32 } %exn
```

### `-C panic=abort`

With `-C panic=abort`, panic calls are compiled to `core::intrinsics::abort()`, which lowers to `LLVMBuildUnreachable` after inserting a trap or abort call. No landing pads are generated, and `nounwind` is placed on all functions. This eliminates the unwinding infrastructure entirely, reducing binary size and enabling more aggressive optimization (LLVM can reason that calls never transfer control to a landing pad).

### `catch_unwind` and FFI Safety

`std::panic::catch_unwind` creates a landing pad boundary explicitly. It requires its closure to be `UnwindSafe` (a marker trait). For FFI, `std::panic::AssertUnwindSafe` can wrap a closure that the programmer vouches is safe across unwind. Panics must not propagate across C FFI boundaries (Rust panics are not C++ exceptions; the personality functions are different); doing so is undefined behavior, which is why `extern "C" fn` declarations implicitly abort on panic rather than unwind.

### Windows SEH Unwinding

On Windows, Rust uses Structured Exception Handling (SEH) rather than the Itanium ABI. The personality function is `__C_specific_handler` (for MSVC targets) or `_gcc_personality_v0` (for `*-pc-windows-gnu` targets using DWARF unwinding). The MSVC SEH path uses `llvm.eh.sjlj.*` intrinsics internally and emits `__try`/`__except` blocks in the LLVM IR. This makes Rust-on-MSVC panic semantics compatible with Windows structured exception filtering, but it also means that C++ exceptions from MSVC runtime code will not automatically unwind through Rust frames — they will terminate the process unless caught explicitly with Windows' `__try` mechanism.

The platform-specific panic runtime is selected at compile time: `panic_unwind` on platforms with DWARF/EH, `panic_windows` for MSVC SEH, and the user's custom `#[panic_handler]` on `no_std` targets.

---

## 177.8 LTO, PGO, and Cross-Compilation

### Link-Time Optimization

**ThinLTO** (`-C lto=thin`): each crate is compiled to LLVM bitcode (stored in the `.rlib`'s `.llvmbc` section alongside the object code). At link time, the linker plugin reads bitcode from all crates, constructs a *summary index* (function summaries: size, call graph, reference graph), and uses the index to identify cross-crate inlining opportunities. Only the functions that are actually inlined across crate boundaries are imported and recompiled; the bulk of each crate's bitcode is optimized in isolation. ThinLTO is parallelizable across crates and much faster than fat LTO. Results are cached in the incremental compilation directory.

**Fat LTO** (`-C lto=fat`): all crate bitcode is linked into a single LLVM module before optimization. The optimizer sees the entire program at once. This produces the best code quality but requires serializing all modules through a single optimization run. Use fat LTO only when maximizing performance of a release artifact where build time is not a constraint.

Enabling in `Cargo.toml`:

```toml
[package]
name    = "my-crate"
version = "0.1.0"
edition = "2024"

[profile.release]
lto = "thin"
```

### Profile-Guided Optimization

PGO improves inlining decisions, branch prediction, and code layout by providing the optimizer with runtime frequency data.

```bash
# Step 1: instrument build
RUSTFLAGS="-C profile-generate=/tmp/pgo-data" cargo build --release
# Step 2: run representative workload against instrumented binary
./target/release/my-crate --bench
# Step 3: merge profile data
/usr/lib/llvm-22/bin/llvm-profdata merge \
    -output=/tmp/pgo-data/merged.profdata \
    /tmp/pgo-data/*.profraw
# Step 4: use merged data
RUSTFLAGS="-C profile-use=/tmp/pgo-data/merged.profdata" cargo build --release
```

The resulting binary has hot functions placed in contiguous memory (improving icache density) and rare branches pushed to cold sections.

**BOLT integration**: rustc can optionally apply LLVM BOLT ([Chapter 74 — LTO and Whole-Program Optimization](../part-13-lto-whole-program/ch74-lto-and-whole-program-optimization.md)) to the final binary for a second round of layout optimization based on runtime profiles. The Rust project uses BOLT on rustc itself as part of the CI perf pipeline, achieving an additional ~5% wall-clock speedup on top of PGO.

### Cross-Compilation

rustc uses LLVM targets directly. Each `--target` triple maps to an LLVM `TargetMachine`. The sysroot for a cross-target contains the pre-compiled standard library for that target.

```bash
rustup target add aarch64-unknown-linux-gnu
cargo build --target aarch64-unknown-linux-gnu
```

A `Cargo.toml` snippet specifying a cross-target linker:

```toml
[package]
name    = "my-crate"
version = "0.1.0"
edition = "2024"

[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"
```

`build.rs` scripts receive target information via environment variables:

```rust
fn main() {
    let target_arch = std::env::var("CARGO_CFG_TARGET_ARCH").unwrap();
    let target_os   = std::env::var("CARGO_CFG_TARGET_OS").unwrap();
    if target_arch == "aarch64" && target_os == "linux" {
        println!("cargo:rustc-cfg=has_neon");
    }
}
```

The `TARGET` env var in `build.rs` is the target triple; `HOST` is the host triple. `CARGO_CFG_TARGET_*` variables are set for each component of the target tuple.

**Sysroot layout**: the cross-compilation sysroot installed by `rustup target add` contains pre-compiled `.rlib` files for `core`, `alloc`, `std`, `compiler_builtins`, and the panic runtime for the target. These are placed in `~/.rustup/toolchains/<toolchain>/lib/rustlib/<target>/lib/`. When compiling `--target aarch64-unknown-linux-gnu`, rustc searches this directory for pre-built standard library rlibs rather than recompiling them. Custom targets without a pre-built sysroot require building the sysroot from source with `cargo build -Zbuild-std`.

**Target specification files**: custom targets are described in JSON `TargetSpec` files (passed via `--target path/to/my-target.json`). A target spec includes: `llvm-target` (the LLVM target triple), `data-layout` (the LLVM data layout string), pointer width, calling convention, OS/environment strings, and linker flavor. The [Rust Embedded Book](https://docs.rust-embedded.org/book/) describes how to write target specs for bare-metal embedded targets.

### rustc's Vendored LLVM

rustc does not use the system LLVM. It vendors a fork of LLVM in [llvm-project/](https://github.com/rust-lang/rust/tree/master/src/llvm-project) (a git submodule pinned to a specific LLVM commit). The vendored LLVM typically tracks LLVM trunk with a delay of one to two major releases: nightly tracks closest to trunk, while the stable channel uses a more conservative snapshot. The fork carries a small set of patches for: Rust-specific calling convention adjustments, MachO weak imports, and incremental compilation bitcode format extensions. The upgrade process involves rebasing these patches onto the new LLVM base and running the full rustc test suite.

---

## 177.9 The Cranelift Backend (rustc_codegen_cranelift)

Cranelift ([github.com/bytecodealliance/wasmtime/tree/main/cranelift](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift)) is a compiler backend written in Rust, developed primarily by the Bytecode Alliance for use in Wasmtime. It has been integrated into rustc as `rustc_codegen_cranelift` ([compiler/rustc_codegen_cranelift/](https://github.com/rust-lang/rust/tree/master/compiler/rustc_codegen_cranelift/)), providing a faster alternative to LLVM for debug builds.

### CLIF: Cranelift Intermediate Format

Cranelift's IR (CLIF) is an SSA-form IR with basic blocks and a defined value type system. Key types: `i8`, `i16`, `i32`, `i64`, `i128`, `f32`, `f64`, `b1` (boolean), and SIMD vector types like `i32x4`. There are no target-specific types in CLIF; legalization handles the conversion to machine-level representations.

A CLIF function for `a + b * c`:

```
function %fma(i32, i32, i32) -> i32 system_v {
block0(v0: i32, v1: i32, v2: i32):
    v3 = imul v1, v2        ; v3 = b * c
    v4 = iadd v0, v3        ; v4 = a + (b * c)
    return v4
}
```

Key CLIF concepts:

- **`function`**: top-level unit; carries calling convention (`system_v`, `fast`, `cold`, `winfast`, etc.)
- **`block`**: basic block; arguments replace phi nodes (CLIF uses block parameters)
- **`Value` (`v0`, `v1`, ...)**:  SSA values; each defined exactly once
- **`Inst`**: an instruction; each produces zero or one values (multi-result instructions produce a `ValueList`)

CLIF block parameters are the SSA equivalent of phi nodes: instead of `phi(b1: v1, b2: v2)` at the merge block, the predecessors pass arguments when branching: `jump block3(v1)` and `jump block3(v2)`.

### Compilation Pipeline

1. **CLIF construction** (`rustc_codegen_cranelift`): MIR is translated to CLIF by a loop analogous to `FunctionCx` in the LLVM backend. `Place` values map to stack slots (`StackSlot` in CLIF) or SSA values.

2. **Verification**: the CLIF verifier checks SSA validity, type consistency, and CFG well-formedness. This runs in debug builds of Cranelift.

3. **Legalization**: types that the target hardware cannot represent directly (e.g., `i128` on a 64-bit target) are split into multiple values. SIMD types are emulated if the target lacks the corresponding instructions.

4. **`regalloc2`** (SSA-aware register allocation): the `regalloc2` crate ([github.com/bytecodealliance/regalloc2](https://github.com/bytecodealliance/regalloc2)) implements SSA-based linear-scan allocation with live-range splitting. Unlike LLVM's greedy allocator, `regalloc2` operates on the SSA form directly, exploiting SSA properties to reduce the number of spills. For debug builds (no optimization), `regalloc2` is dramatically faster than LLVM's allocation pipeline: it avoids the cost of instruction scheduling and the regalloc tuning that LLVM performs for optimization.

5. **Emission** (`MachInst` → binary): Cranelift's machine layer translates abstract `MachInst` instructions to target-specific binary. The `cranelift-codegen` crate emits x86-64, AArch64, RISC-V, and s390x.

### Cranelift vs LLVM for Debug Builds

The performance difference is significant. For large Rust codebase:
- LLVM backend at `-O0`: dominates compile time with instruction selection overhead, slow alias analysis, and `O(n log n)` greedy register allocation. Even without any optimization, LLVM's SelectionDAG and instruction scheduler run, adding substantial latency.
- Cranelift: skips most optimization infrastructure; legalization and `regalloc2` are cache-friendly and SSA-exploiting. Reported speedups: 1.5×–3× faster compilation for debug builds in `rustc` itself and Wasmtime.

The `regalloc2` allocator is particularly important. Because CLIF is in SSA form with block parameters, `regalloc2` can exploit SSA properties: live ranges that do not interfere can share a register without explicit analysis, and the "SSA destruction" step (inserting parallel copies at critical edges to lower block-parameter passing to register moves) is handled cleanly. LLVM's greedy allocator, by contrast, operates after SSA destruction by `phi` lowering and must use a more expensive interference graph.

Cranelift does not support LTO, PGO, or sanitizers. It is not suitable for release builds. The roadmap (as of April 2026) includes stabilizing the Cranelift backend in rustc for debug builds, which would make it opt-out rather than opt-in on nightly.

### Wasmtime Integration

`cranelift-wasm` translates WebAssembly binary format → CLIF, using `wasmparser` for decoding. The translation is type-directed: WebAssembly's stack machine is translated to SSA CLIF by maintaining a type stack and creating CLIF `Value`s as items are pushed. Wasm structured control flow (`block`, `loop`, `if`) is lowered to CLIF blocks with branch edges; Wasm's stack-based `br_table` becomes a CLIF `jump_table`.

Wasmtime supports two compilation modes:

- **AOT** (`wasmtime compile`): compiles a `.wasm` file to a platform-native shared library (`*.cwasm`). The CLIF pipeline runs offline. The resulting `.cwasm` is memory-mapped and executed directly.
- **JIT** (`wasmtime run`): compiles Wasm modules at load time, caching compiled code in a per-process `CodeMemory` region. The JIT respects Wasm's sandbox: all memory accesses go through a linear memory bounds check (a single comparison against the linear memory size, or elided on 64-bit platforms with sufficient virtual address space guard pages).

Wasmtime uses Cranelift's tiered compilation: modules start at the baseline tier (fast compilation, minimal optimization) and are recompiled at the optimizing tier on demand. The baseline tier is distinct from the optimizing pipeline — it uses a simpler single-pass CLIF construction without instruction combining.

### Activating Cranelift in rustc

```bash
CARGO_PROFILE_DEV_CODEGEN_BACKEND=cranelift \
    cargo +nightly build -Z codegen-backend
```

Or in `Cargo.toml` (nightly only):

```toml
[package]
name    = "my-crate"
version = "0.1.0"
edition = "2024"

[profile.dev]
codegen-backend = "cranelift"
```

This is still nightly-only as of April 2026. The Cranelift backend passes the majority of the rustc test suite but has known gaps in inline assembly support and a small number of intrinsics.

### Async Closures and Coroutine Lowering (Edition 2024)

Edition 2024 stabilizes `async` closures — closures that return a `Future` and can `await` inside. Prior to Edition 2024, `async` blocks inside closures required workarounds because the `Fn` and `FnMut` traits did not have async versions. Edition 2024 introduces `AsyncFn`, `AsyncFnMut`, and `AsyncFnOnce` in `core::ops`, allowing async closures to be passed to APIs expecting these traits.

A concrete example using `async` closures:

```rust
use std::future::Future;

async fn map_async<T, U, F>(values: &[T], f: F) -> Vec<U>
where
    F: AsyncFn(&T) -> U,
{
    let mut out = Vec::with_capacity(values.len());
    for v in values {
        out.push(f(v).await);
    }
    out
}

#[tokio::main]
async fn main() {
    let nums = vec![1u32, 2, 3];
    let doubled = map_async(&nums, async |x| x * 2).await;
    println!("{doubled:?}");
}
```

The `async |x| x * 2` is an async closure: it desugars to a coroutine type that implements `AsyncFn(&u32) -> u32`. Without Edition 2024, the call site would require `|x| async move { x * 2 }`, which creates a new `Future` allocation per element and has different capture semantics.

The codegen impact: an `async` closure is lowered to a coroutine (generator) type, similar to an `async fn`. Its MIR body contains `Yield` terminators at each `await` point. `GeneratorLower` (renamed `CoroutineLower` in recent rustc) transforms the body into a state machine: a `match` on the coroutine's suspend point index, with each arm containing the code between two `await`s.

The resulting state machine struct contains one field per live local across an `await` point, sized precisely to avoid padding where possible. This is why Rust's `async fn` can be allocation-free (stored on the stack as an opaque `impl Future`) while Java's `CompletableFuture` always heap-allocates.

Cranelift handles coroutine state machines without special support — they are plain structs after MIR lowering. The `Yield` terminator is not visible at the CLIF level; it has been desugared away before codegen receives the MIR.

---

## 177.10 The GCC Backend (rustc_codegen_gcc)

`rustc_codegen_gcc` ([github.com/rust-lang/rustc_codegen_gcc](https://github.com/rust-lang/rustc_codegen_gcc)) implements the `CodegenBackend` trait using `libgccjit` as the IR and code generator rather than LLVM. It targets targets LLVM does not support.

### libgccjit API

`libgccjit` is a shared library that exposes GCC's middle-end and backend as a C API. Key types:

| C type | Purpose |
|--------|---------|
| `gcc_jit_context` | Global compilation context; owns all objects |
| `gcc_jit_type` | A GCC type (`int`, `void *`, struct, function pointer, etc.) |
| `gcc_jit_function` | A function being defined |
| `gcc_jit_block` | A basic block within a function |
| `gcc_jit_rvalue` | A pure value: constant, local read, binary/unary op, function call |
| `gcc_jit_lvalue` | A writable location: local, global, dereference, field access |
| `gcc_jit_param` | A function parameter (also an lvalue) |

A minimal libgccjit example (in C, to show the raw API surface):

```c
gcc_jit_context *ctx = gcc_jit_context_acquire();
gcc_jit_type *int_t = gcc_jit_context_get_type(ctx, GCC_JIT_TYPE_INT);

gcc_jit_param *a = gcc_jit_context_new_param(ctx, NULL, int_t, "a");
gcc_jit_param *b = gcc_jit_context_new_param(ctx, NULL, int_t, "b");
gcc_jit_param *params[] = { a, b };
gcc_jit_function *fn = gcc_jit_context_new_function(
    ctx, NULL, GCC_JIT_FUNCTION_EXPORTED, int_t, "add", 2, params, 0);

gcc_jit_block *bb = gcc_jit_function_new_block(fn, "entry");
gcc_jit_rvalue *sum = gcc_jit_context_new_binary_op(
    ctx, NULL, GCC_JIT_BINARY_OP_PLUS, int_t,
    gcc_jit_param_as_rvalue(a), gcc_jit_param_as_rvalue(b));
gcc_jit_block_end_with_return(bb, NULL, sum);

gcc_jit_result *result = gcc_jit_context_compile(ctx);
```

`rustc_codegen_gcc` wraps this API with safe Rust types in [compiler/rustc_codegen_gcc/src/](https://github.com/rust-lang/rustc_codegen_gcc). It mirrors the LLVM backend's structure: `CodegenCx<'gcc, 'tcx>` owns a `gcc_jit_context`; `FunctionCx` translates each MIR `Body` into libgccjit blocks.

### How `rustc_codegen_gcc` Maps to `rustc_codegen_llvm`

The `CodegenBackend` trait ([compiler/rustc_codegen_ssa/src/traits/backend.rs](https://github.com/rust-lang/rust/blob/master/compiler/rustc_codegen_ssa/src/traits/backend.rs)) is the interface between the Rust compiler frontend and any codegen backend. The `rustc_codegen_ssa` crate provides backend-agnostic infrastructure: the MIR translation logic, the `FunctionCx` translation loop, the `PlaceRef`/`OperandRef` abstractions, and the intrinsics dispatch table. Backends only need to implement the low-level builder trait (`BuilderMethods`) and the type/value creation traits.

`rustc_codegen_gcc` implements these traits by delegating to `libgccjit`:
- `BuilderMethods::add(lhs, rhs)` → `gcc_jit_context_new_binary_op(ctx, NULL, GCC_JIT_BINARY_OP_PLUS, ty, lhs, rhs)`
- `BuilderMethods::load(ptr, ty, align)` → dereference `gcc_jit_lvalue` with explicit type
- `BuilderMethods::call(fn_val, args, ...)` → `gcc_jit_block_add_eval(bb, NULL, gcc_jit_context_new_call(...))`
- `BuilderMethods::br(cond, true_bb, false_bb)` → `gcc_jit_block_end_with_conditional(...)`

Because `rustc_codegen_ssa` handles the MIR-to-builder translation, adding a new backend does not require understanding MIR — only the target IR's builder interface. This separation allowed the GCC backend to be developed by a small team (led by Antoni Boucher) while sharing the bulk of the translation logic with LLVM.

### Motivation

GCC supports targets that LLVM does not (as of April 2026):

| Architecture | LLVM support | GCC support |
|-------------|-------------|-------------|
| m68k | No | Yes |
| ia64 (Itanium) | Removed in LLVM 12 | Yes |
| SPARC (older) | Limited | Full |
| AVR (ATmega) | Limited | Full |
| RISC-V embedded (RV32I no-std) | Good | Good |
| s390x (IBM Z) | Good | Good |

Additionally, GCC's Graphite polyhedral optimizer (built on ISL — the same library used by Polly; see [Chapter 112 — Polly](../part-12-polly/ch112-polly.md)) and GCC's RTL-level peephole optimizer can produce better code for some workloads.

### Status as of April 2026

`rustc_codegen_gcc` compiles `core`, `alloc`, and most of `std` successfully. Known gaps:

- **Inline assembly**: GCC's inline assembly dialect uses AT&T syntax constraints; LLVM uses a different constraint encoding. Many `asm!` blocks in `core` and `std` use LLVM-specific constraints that need manual porting.
- **Intrinsics**: some LLVM intrinsics (e.g., `llvm.ctlz`, `llvm.fshl`) have direct GCC equivalents (`__builtin_clz`, `__builtin_ia32_shldq`) but not all are mapped yet.
- **Sanitizers**: GCC's address sanitizer is different from LLVM's; integration is partial.
- **LTO**: libgccjit does not expose GCC's LTO infrastructure; whole-program optimization is not available through the GCC backend.

The GCC backend is the path to running Rust on embedded and legacy targets without bootstrapping an LLVM cross-compiler. As rustc's primary user community shifts more toward systems programming (embedded, firmware), the GCC backend's target coverage becomes strategically important.

Building the GCC backend requires a libgccjit-enabled GCC build (not the system GCC — a custom build with `--enable-host-shared --enable-languages=jit`). The `rustc_codegen_gcc` README documents the exact GCC configure flags needed. Once built, the backend is activated via:

```bash
RUSTC_CODEGEN=gcc cargo +nightly build -Z codegen-backend=gcc
```

Testing uses the same `x test` infrastructure as the LLVM backend, with a subset of the test suite marked `//@ ignore-backends: gcc` for tests that exercise LLVM-specific behavior.

---

## 177.11 rustc's Vendored LLVM and Version Delta Strategy

Although covered briefly in §177.8, the LLVM version management deserves its own discussion because it affects every part of the LLVM codegen story.

rustc maintains its own fork of the LLVM monorepo as a git submodule ([src/llvm-project/](https://github.com/rust-lang/rust/tree/master/src/llvm-project)). The fork strategy:

1. **Pin to a specific LLVM commit**: the submodule is updated by rustc contributors, not automatically. The stable channel uses a tested LLVM snapshot; nightly tracks closer to LLVM trunk.

2. **Carry patches**: changes that have not yet been merged upstream or that are specific to rustc's needs are applied directly to the submodule — ARM softfloat ABI adjustments, debug-info improvements for MIR-level inlining, incremental bitcode format extensions.

3. **Upgrade process**: upgrading LLVM requires (a) rebasing patches, (b) updating `rustc_llvm/build.rs` to link the correct set of LLVM libraries, (c) running the full `x test` bootstrap suite, (d) filing upstream patches for any API changes in LLVM that affect rustc's usage.

The vendored LLVM typically lags current LLVM trunk by one to two major releases. The version delta between rustc's LLVM and the system LLVM means that `llvm-sys` crates that link to the system LLVM are ABI-incompatible with rustc's internal LLVM. This is why `rustc` is a monolithic binary: it links LLVM statically and cannot safely expose LLVM symbols to dynamically-loaded backends.

### rustc's LLVM Wrapper Crate

All LLVM C++ API calls from rustc go through a thin C wrapper layer in [compiler/rustc_llvm/](https://github.com/rust-lang/rust/blob/master/compiler/rustc_llvm/). This wrapper re-exports the LLVM C API (`llvm-c/Core.h`) with Rust-friendly names and extends it with custom functions that LLVM's C API does not expose (e.g., `LLVMRustAddCallSiteAttr`, `LLVMRustSetDataLayoutFromTargetMachine`, and functions for emitting debug metadata). The wrapper is compiled by `build.rs` using the `cc` crate and linked directly.

The decision to use a C wrapper rather than the C++ API directly (via cxx or bindgen) is intentional: C++ ABI stability across LLVM versions is not guaranteed, but the C API (`llvm-c/`) has a more stable interface. Where the C API is insufficient, custom wrappers are added and maintained in the rustc tree.

The `rustc_llvm` build script (`build.rs`) determines which LLVM libraries to link by calling `llvm-config --libs --system-libs` on the vendored LLVM build. The link command for a typical build includes: `libLLVMCore`, `libLLVMBitWriter`, `libLLVMCodeGen`, `libLLVMX86CodeGen`, `libLLVMAArch64CodeGen`, `libLLVMDebugInfoDWARF`, and approximately 40 other LLVM component libraries. Static linking ensures that the system LLVM (which may be a different version) cannot interfere.

---

## Chapter Summary

- **rustc's driver** is a demand-driven query graph (`TyCtxt::query`) providing memoization and incremental compilation via the red-green dependency tracking algorithm; the pipeline runs in phases: parsing → HIR → type checking → THIR → MIR → codegen.

- **MIR** is rustc's mid-level IR: a CFG of basic blocks with `Place`/`Rvalue`/`Statement`/`Terminator` nodes; it is not in SSA form but uses named locals; it is the level at which borrow checking, Miri interpretation, and MIR optimizations operate.

- **Non-Lexical Lifetimes (NLL)** annotates MIR with region constraints solved by dataflow; it is sound but has false positives for flow-sensitive reasoning.

- **Miri** is an interpreter over `mir::Body` parameterized by `MiriMachine`; it detects use-after-free, uninitialized memory, data races, and memory model violations; Stacked Borrows enforces aliasing via per-allocation tag stacks; Tree Borrows (opt-in via `-Zmiri-tree-borrows`) is the evolving successor with per-node permission trees.

- **Polonius** reformulates borrow checking as a Datalog program over `loan_issued_at`, `loan_killed_at`, and `loan_invalidated_at` relations; the DataFrog engine provides near-linear-time solving; Polonius v1 (`-Z polonius=yes`) is the production Datalog implementation; Polonius v2 (`-Z polonius=next`) integrates with a-mir-formality and the next-gen trait solver.

- **a-mir-formality** is a PLT Redex mechanization of Rust's trait system, lifetimes, and MIR typing; it drives Polonius v2 and the next-gen trait solver; it supersedes Chalk as the authoritative specification.

- **`rustc_codegen_llvm`** translates MIR to LLVM IR via `CodegenCx<'ll, 'tcx>` and `FunctionCx<'a, 'll, 'tcx>`; ADT layout uses `TyAndLayout` with niche optimization for types like `Option<Box<T>>`; trait objects are fat pointers to `(data_ptr, vtable)`; Rust-specific LLVM attributes (`noalias`, `dereferenceable`, `noundef`, `nounwind`, `nonnull`) enable LLVM optimizations beyond what C++ achieves.

- **Panics** under `-C panic=unwind` use the Itanium C++ ABI unwinding machinery with `rust_eh_personality`; under `-C panic=abort` all panics compile to `abort()` with no landing pad infrastructure, enabling `nounwind` on every function.

- **LTO**: ThinLTO (`-C lto=thin`) compiles per-crate bitcode with a summary index for cross-crate inlining; fat LTO (`-C lto=fat`) merges all bitcode into a single module. **PGO** uses `-C profile-generate` / `llvm-profdata merge` / `-C profile-use` to feed runtime frequency data to LLVM.

- **Cross-compilation** uses rustup target triples (`rustup target add`), per-target linker configuration in `Cargo.toml`, and `build.rs` `CARGO_CFG_TARGET_*` variables; rustc's vendored LLVM fork is separate from the system LLVM.

- **Cranelift** (`rustc_codegen_cranelift`) uses CLIF (SSA IR with block parameters instead of phi nodes), legalization, and `regalloc2` (SSA-aware linear scan) to achieve 1.5×–3× faster debug builds than LLVM; it is the primary backend for Wasmtime's JIT and AOT modes; activated in rustc via `codegen-backend = "cranelift"` (nightly).

- **`rustc_codegen_gcc`** uses `libgccjit` to target architectures LLVM does not support (m68k, ia64, SPARC, AVR); it shares the `CodegenBackend` trait with the LLVM backend; as of April 2026 it compiles `core`/`alloc`/most of `std` with known gaps in inline assembly dialects and some intrinsics.
