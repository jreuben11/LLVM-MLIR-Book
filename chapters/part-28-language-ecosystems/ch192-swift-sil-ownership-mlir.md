# Chapter 192 — Swift SIL: Ownership, Optimization, and Influence on MLIR

*Part XXVIII — Language Ecosystems, Emerging Targets, and Engineering Practice*

Swift occupies a peculiar position in the LLVM ecosystem: it uses LLVM as its native-code backend but introduces an intermediate representation — the Swift Intermediate Language (SIL) — that sits between the Swift AST and LLVM IR. SIL exists precisely because LLVM IR is insufficient for Automatic Reference Counting optimization. Where LLVM IR treats `call @swift_retain` as an opaque function call with unknown side effects, SIL makes reference counting semantics a first-class part of the IR type system. This chapter explains why LLVM IR's alias analysis and call-graph reasoning cannot eliminate redundant ARC operations without external invariants; how SIL's ownership type system encodes those invariants statically; what the SIL instruction set looks like in practice; how the SIL optimization pipeline is structured; and how SIL's ownership model has influenced MLIR's bufferization infrastructure, the `linalg` dialect's destination-passing style, and the buffer deallocation pass. Readers should be familiar with LLVM IR from [Chapter 14 — Advanced Type Systems](../part-03-type-theory/ch14-advanced-type-systems.md) and codegen structure from [Chapter 39 — CodeGenModule and CodeGenFunction](../part-06-clang-codegen/ch39-codegenmodule-codegenfunction.md); the MLIR sections build on [Chapter 148 — MLIR Bufferization](../part-21-mlir-transformations/ch148-mlir-bufferization.md) and [Chapter 138 — Memory Dialects](../part-20-in-tree-dialects/ch138-memory-dialects.md).

---

## 192.1 Why LLVM IR Is Insufficient for ARC

### The ARC Problem

Swift's memory management model is Automatic Reference Counting. Unlike Objective-C ARC — which is inserted by clang as part of code generation and operates on ObjC objects through the ObjC runtime — Swift ARC operates on both class instances (heap-allocated, reference-counted) and on value types that contain class references (structs, enums, tuples that box references through existentials or generic parameters). The Swift compiler inserts `swift_retain` and `swift_release` calls; the runtime (`libswiftCore`) executes the actual atomic increment and decrement. The compiler's job is to insert no more retain/release pairs than are semantically required, and to eliminate redundant pairs through optimization.

A naive ARC strategy inserts a retain before every copy of a reference and a release at every point where a reference ceases to be needed. This is correct but produces enormous redundancy. Consider a tight loop that reads a class property on every iteration: a naive ARC strategy retains the property value on every iteration and releases it at the end of the loop body. An optimizing strategy hoists the retain before the loop and sinks the release after it, processing one retain/release pair instead of N pairs. This is the central ARC optimization problem.

### Why LLVM IR Cannot Solve It Alone

When SIL is lowered to LLVM IR, `swift_retain` and `swift_release` become ordinary `call` instructions. LLVM's optimizer sees them as opaque calls. The problems are fundamental:

**No ownership encoding**: LLVM IR has no concept of "this value's reference count is +1 and must be decremented on every control-flow path before the function returns." Ownership is an invariant that lives outside the IR. LLVM's alias analysis can prove that two pointers don't alias, but it cannot prove that a particular call is the *unique consumer* of a reference.

**Interprocedural barriers**: LLVM's alias analysis conservatively assumes that any call to an external function can read or write any memory accessible through pointers it can observe. `swift_retain` takes a pointer argument. LLVM's alias analysis must assume `swift_retain` could call arbitrary Swift code (through the RC object's type metadata) that modifies global state. In the general case this is true — weak reference callbacks and deinitializers run during release. LLVM has no information to distinguish a retain/release pair on a locally-owned object (which can always be hoisted) from one on a shared object (which cannot).

**The pairing problem**: to hoist a retain out of a loop, the optimizer must prove that the corresponding release is also in the loop body and that they can be repositioned. LLVM can eliminate dead loads and stores using MemorySSA, but retain/release pairs are not loads/stores — they are function calls. Matching them requires dominator-tree reasoning over call sequences, not over memory operations. LLVM's Loop-Invariant Code Motion (LICM) and Dead Store Elimination (DSE) passes have no semantics for "call A dominates call B and they form a cancel pair."

**The solution**: SIL encodes ownership directly in the IR. Instead of leaving retain/release as uninterpreted calls, SIL gives every SSA value an ownership qualifier that is verified statically. Optimization passes can reason about ownership at the SIL level where the invariants are explicit, and only then lower to retain/release calls in LLVM IR.

### A Brief History: Pre-OSSA vs. OSSA SIL

SIL predates OSSA. The original SIL design (Swift 1.0–5.4) used explicit `strong_retain` / `strong_release` instructions analogous to LLVM IR's eventual `llvm.objc.retain` / `llvm.objc.release` intrinsics. Passes like the **ARC Optimizer** operated on these instructions directly, matching retain/release pairs through a reference count state machine. This approach was effective but fragile: the pairing analysis was sensitive to aliasing and required significant conservatism in the presence of unknown function calls.

**Ownership SSA (OSSA)** was introduced as a first-class SIL mode starting in Swift 5.5 (shipped in Xcode 13, 2021). In OSSA, the fundamental currency of ARC operations is shifted from raw `strong_retain` / `strong_release` calls to high-level ownership instructions (`copy_value`, `destroy_value`, `begin_borrow`, `end_borrow`). The OSSA verifier, implemented in [`lib/SIL/Verifier/SILVerifier.cpp`](https://github.com/swiftlang/swift/blob/main/lib/SIL/Verifier/SILVerifier.cpp), runs after SILGen and after every mandatory pass, checking that every `@owned` SSA value is consumed exactly once on every control-flow path. The verifier is run in debug builds by default and can be enabled in release builds with `-Xfrontend -sil-verify-all`.

The key advantage of OSSA is that the *structure* of the IR encodes the ownership graph — there is no pairing analysis to do. The OSSA lowering pass (run just before IRGen) converts the high-level ownership instructions back to `strong_retain` / `strong_release` calls, but at this point the optimization window is closed: all ownership-level optimizations have already been applied. Separating the optimization phase (which reasons about `copy_value` / `destroy_value` linearity) from the lowering phase (which emits actual ARC calls) is the same separation of concerns that MLIR's dialect lowering achieves: operate at a high level until you must commit to low-level mechanics.

The transition from pre-OSSA to OSSA SIL is analogous to LLVM's transition from explicit `llvm.objc.retain` intrinsics to the `@clang.arc.use` / `clang_arc_noop_use` family introduced as part of ARC contract optimization in clang's ARC codegen (see [Chapter 41 — Calls, the ABI Boundary, and Builtins](../part-06-clang-codegen/ch41-calls-abi-builtins.md)). Both move from uninterpreted call sites to semantically rich IR constructs that the optimizer can reason about structurally.

---

## 192.2 The SIL Type System and Ownership Qualifiers

SIL is a fully typed SSA IR. Every SIL value has a type from Swift's type system (applied to SIL's representation layer) and an **ownership kind** — a qualifier that specifies what relationship the holder has to the value's reference count. SIL enforces these qualifiers through a static verification pass called the **Memory Lifetime Verifier** and, since Swift 5.5, through **Ownership SSA (OSSA)** — an ownership-aware extension of SSA form.

### The Four Ownership Qualifiers

| Qualifier | Meaning | ARC effect | Required consumer |
|-----------|---------|------------|------------------|
| `@owned` | Caller holds a +1 retain; responsibility transferred from producer to consumer | One `strong_retain` has been performed | Must be consumed exactly once on every CF path |
| `@guaranteed` | Borrowed reference; liveness guaranteed by enclosing scope | No ARC operation — caller guarantees liveness | Must not be retained; scope ends at matching `end_borrow` |
| `@unowned` | Unsafe unretained reference; no ARC operation | No ARC operation | Caller guarantees liveness; no compiler verification |
| `@inout` | Exclusive mutable access; value passed by address | No ARC operation on the address | Callee must leave a valid value; access ends at matching `end_access` |

The `@owned` qualifier is the critical one. In OSSA form, every `@owned` SSA value in a SIL function must be consumed **exactly once** on every control-flow path. "Consumed" means one of: passed to `destroy_value` (which decrements the reference count and potentially calls `deinit`), stored into memory through a `store [assign]` or `store [init]`, passed as an `@owned` argument to an `apply`, or forwarded to a phi-node argument. The static verifier proves this linear-use property. A function that has an `@owned` value on a path that doesn't consume it is ill-formed SIL — the verifier rejects it. A function that consumes the same `@owned` value on two paths is also rejected.

This is the key insight articulated in Jordan Rose's 2018 LLVM Dev Meeting talk "Ownership Is Membership" ([LLVM Dev Meeting 2018](https://llvm.org/devmtg/2018-10/)): ownership invariants should be **membership invariants in the IR's type system**, not properties reconstructed by alias analysis over an untyped blob of function calls.

### Function Type Syntax

Ownership qualifiers appear in function types, making calling conventions visible in the IR:

```sil
// A method that borrows its receiver and returns an owned result
%f : $@convention(method) (@guaranteed Node) -> @owned String

// A closure capturing a value; the capture is consumed (moved into the closure)
%g : $@convention(thin) (@owned Context) -> @owned Result

// An inout function — exclusive write access to an Int
%h : $@convention(thin) (@inout Int) -> ()
```

The `@convention` attribute specifies the calling convention: `thin` (no implicit self), `method` (implicit self passed as first argument), `objc_method`, `block`, `c`, or `witness_method` (protocol dispatch through a witness table).

### Ownership vs. Rust's Borrow Semantics

Both SIL and Rust MIR enforce linear use of owned values at the IR level. The following table maps SIL ownership qualifiers to their Rust type-system counterparts:

| SIL qualifier | Rust equivalent | Linear use enforced | ARC/drop operation |
|--------------|----------------|--------------------|--------------------|
| `@owned T` (class) | `Box<T>` or owned `T: !Copy` | Yes — exactly once on every path | `destroy_value` → `deinit` / `drop` |
| `@guaranteed T` | `&T` (shared reference) | N/A — borrow scope, not consumed | None; scope-bounded |
| `@inout T` | `&mut T` (exclusive reference) | N/A — must leave valid value | None; scope-bounded |
| `@unowned T` | Raw pointer `*const T` | No | None |
| `~Copyable` type (Swift 5.9+) | `T: !Copy` (move-only) | Yes — SIL prevents implicit `copy_value` | `destroy_value` |
| `consume` operator | `drop(x)` or `let _ = x` | Explicit consumption | Triggers `destroy_value` |

The key architectural difference: Rust enforces move semantics at the type-system level in the HIR/THIR and borrow checks at the MIR level before any IR lowering (see [Chapter 177 — rustc: Architecture, MIR, and Codegen Backends](../part-26-ecosystem-frontiers/ch177-rustc-architecture-mir-codegen-backends.md)). Swift enforces ownership at the SIL level, *after* AST type checking but *before* LLVM IR generation. SIL's OSSA form is closer to MIR's borrow-check representation than to LLVM IR — both are ownership-aware SSA forms where linear use is statically verified.

---

## 192.3 The SIL Instruction Set

SIL's instruction set is defined in [`include/swift/SIL/SILInstruction.h`](https://github.com/swiftlang/swift/blob/main/include/swift/SIL/SILInstruction.h) and fully documented in [`docs/SIL.rst`](https://github.com/swiftlang/swift/blob/main/docs/SIL.rst). The following walkthrough uses this Swift class as a running example:

```swift
class Node {
    var value: Int
    var next: Node?
    init(_ v: Int) { value = v; next = nil }
    deinit { print("freed \(value)") }
}

func process(_ n: Node) -> Int {
    return n.value
}

func makeChain(_ v: Int) -> Node {
    let n = Node(v)
    n.next = Node(v + 1)
    return n
}
```

### Allocation Instructions

```sil
// Heap allocation — returns @owned $Node
// Runtime calls swift_allocObject with the class's type metadata
%n = alloc_ref $Node                    // %n : $Node, ownership: @owned

// Stack allocation — returns an address ($*T), not a value
// No ARC involved; address is valid until dealloc_stack
%addr = alloc_stack $Int               // %addr : $*Int
dealloc_stack %addr : $*Int            // must appear on every path

// Explicit stack allocation for a ref-counted class (stack promotion)
%n2 = alloc_ref [stack] $Node          // dealloc_ref [stack] must balance
```

### Ownership Operations

```sil
// copy_value: produce a new @owned reference with an incremented retain count
// This is the OSSA replacement for strong_retain in pre-OSSA SIL
%n2 = copy_value %n : $Node            // %n2 : $Node, @owned (RC += 1)

// destroy_value: consume an @owned value; RC -= 1; may call deinit
destroy_value %n2 : $Node              // %n2 is consumed; cannot be used after this

// begin_borrow: open a guaranteed borrow scope
// %n remains @owned; %nb is @guaranteed for the duration of the scope
%nb = begin_borrow %n : $Node          // %nb : $Node, @guaranteed
// ... use %nb ...
end_borrow %nb : $Node                 // must dominate all uses of %nb
// %n is still @owned and must still be consumed
```

### Access Instructions (Exclusivity Enforcement)

```sil
// begin_access / end_access enforce the Law of Exclusivity:
// no two accesses to the same variable can overlap if either is a write.
// [read] / [modify] = access kind; [static] / [dynamic] = enforcement mode
%field_addr = ref_element_addr %nb : $Node, #Node.value  // address of field
%acc = begin_access [read] [static] %field_addr : $*Int
%v = load [trivial] %acc : $*Int       // load an Int (trivial type — no ARC)
end_access %acc : $*Int
```

### Function Application

```sil
// Method call with @guaranteed receiver — no ARC on the argument
// The function type shows the calling convention explicitly
%process_fn = function_ref @process : $@convention(thin) (@guaranteed Node) -> Int
%result = apply %process_fn(%nb) : $@convention(thin) (@guaranteed Node) -> Int

// Partial application — closure captures %ctx with @owned semantics
// The resulting closure object is itself @owned
%ctx = copy_value %n : $Node           // make an @owned copy for the closure
%closure = partial_apply [callee_guaranteed] %make_closure(%ctx)
    : $@convention(thin) (@owned Node) -> ()
// %closure : $@callee_guaranteed () -> (), @owned
```

### Enum Dispatch and Existentials

```sil
// switch_enum — dispatch on Optional<Node>
switch_enum %opt : $Optional<Node>, case #Optional.some!enumelt: bb1,
                                     case #Optional.none!enumelt: bb2

bb1(%unwrapped : @owned $Node):        // .some payload; received as @owned
  // must consume %unwrapped on this path
  destroy_value %unwrapped : $Node
  br bb3

bb2:
  br bb3

bb3:
  // rejoin

// Existential opening: open_existential_ref
// Produces an @opened existential type — a unique archetype for this specific value
%p : $any Drawable                     // a protocol existential (box + value)
%opened = open_existential_ref %p : $any Drawable
    to $@opened("F4C3A2B1-...") any Drawable
// The opened existential has a concrete (but unknown) type; dispatch now possible
```

### Witness Tables and Protocol Dispatch

```sil
// Protocol witness method lookup — selects implementation from the witness table
// This is a candidate for devirtualization if the concrete type is known
%wm = witness_method $Node, #Hashable.hash : <Self: Hashable> (Self) -> () -> Int
    : $@convention(witness_method: Hashable) <τ_0_0: Hashable> (@in_guaranteed τ_0_0) -> Int

// mark_dependence — Swift 5.9+: establish lifetime dependency for ~Escapable types
// The result %slice is non-escapable; its lifetime depends on %buffer
%slice = mark_dependence %raw_slice : $BufferSlice on %buffer : $ContiguousArray<Int>
```

### Comparing Raw SIL and Optimized SIL

```bash
# Emit Raw SIL (SILGen output — before mandatory passes)
swiftc -emit-silgen -O example.swift -o - 2>/dev/null | swift-demangle

# Emit Canonical SIL (after mandatory passes, before performance opts)
swiftc -emit-sil example.swift -o - 2>/dev/null | swift-demangle

# Emit Optimized SIL (after full optimization pipeline)
swiftc -emit-sil -O example.swift -o - 2>/dev/null | swift-demangle

# Emit LLVM IR (after IRGen)
swiftc -emit-ir -O example.swift -o - 2>/dev/null

# Run sil-opt passes manually (analogous to mlir-opt or llvm's opt)
sil-opt --help
sil-opt -enable-copy-propagation -arc-sequence-opts input.sil -o output.sil
```

In Raw SIL (`-emit-silgen`), the output reflects direct AST lowering: explicit `copy_value` / `destroy_value` pairs appear at every assignment, every borrow scope is explicitly bracketed with `begin_borrow` / `end_borrow`, and `begin_access` / `end_access` brackets appear around every variable access. Pre-OSSA compilers (pre-Swift 5.5) emitted `strong_retain` / `strong_release` at this stage; OSSA-aware SILGen emits the higher-level ownership instructions instead.

After mandatory passes (`-emit-sil`), many obvious redundancies are already eliminated: paired `copy_value` / `destroy_value` on the same value in the same basic block are cancelled. After performance optimization (`-emit-sil -O`), the output shows significant simplification: `process` becomes a single `ref_element_addr` / `load` sequence with no ARC operations (the `@guaranteed` parameter means no retain was ever needed).

---

## 192.4 The SIL Pipeline

SIL's compilation pipeline is a sequence of transformation stages, each producing a more-lowered form of SIL. The pipeline is driven by `lib/SILOptimizer/PassManager/Passes.cpp` and the `SILPassManager` class ([`include/swift/SILOptimizer/PassManager/PassManager.h`](https://github.com/swiftlang/swift/blob/main/include/swift/SILOptimizer/PassManager/PassManager.h)).

### Stage 1: SILGen (Raw SIL)

**SILGen** (`lib/SILGen/`) lowers the typed Swift AST directly to Raw SIL. At this stage:
- Every assignment generates an explicit `copy_value` of the RHS.
- Every variable destruction generates a `destroy_value`.
- Every borrow is explicit: reading a property of a class instance opens a `begin_borrow` scope.
- All accesses generate `begin_access` / `end_access` pairs (both static and dynamic markers present; dynamic ones will be removed or made efficient by later passes).
- Closures that capture by reference generate `project_box` operations.
- Generics are not specialized — calls remain generic with `witness_method` / `class_method` lookups.

Raw SIL is OSSA-form: the verifier runs after SILGen to confirm that every `@owned` value is consumed exactly once on every path.

### Stage 2: Mandatory Passes (Canonical SIL)

The mandatory pass pipeline runs on every compilation regardless of optimization level (`-Onone` or `-O`). These passes establish correctness invariants that downstream stages depend on:

**Definite Initialization (DI)**: verifies that every variable is initialized before use. Implemented as a dataflow analysis over SIL basic blocks that tracks per-variable initialization state at each point. Reports "variable used before being initialized" errors. Also marks `store [init]` vs `store [assign]` on stores (the distinction matters for ARC: a `[init]` store does not need to release the previous value; an `[assign]` does).

**Memory Lifetime Verifier**: runs after DI to confirm OSSA invariants — every `@owned` value consumed exactly once; `begin_borrow` / `end_borrow` properly nested; `begin_access` / `end_access` properly nested. If the verifier reports an error, it indicates a compiler bug in SILGen or a prior mandatory pass.

**Exclusivity Enforcement**: The Law of Exclusivity states that a variable cannot have two simultaneous accesses if either is a write. The enforcement pass either (a) proves statically (`[static]`) that no overlap exists and removes the dynamic check, or (b) leaves a `[dynamic]` access marker that compiles to a runtime exclusivity check. The runtime check calls `swift_beginAccess` and `swift_endAccess`, which maintain a thread-local stack of active accesses and trap on violation.

**Mandatory Inlining**: inlines functions marked `@_transparent` (e.g., arithmetic operators, type conversions) and `@inline(__always)`. This is a correctness pass: `@_transparent` functions must be inlined before any analysis because they participate in the caller's ownership model — they are effectively syntactic sugar for inline expressions.

**Closure Lifetime Fixup**: ensures that closures that capture `@owned` values properly extend the lifetime of those values to the closure's own lifetime.

After the mandatory passes, SIL is in **Canonical SIL** form — OSSA-valid, all mandatory rewrites applied, ready for performance optimization or direct IRGen (in the `-Onone` path).

### Stage 3: Performance Optimization Passes (Optimized SIL)

The performance pass pipeline (`-O`) runs a series of function and module-level passes, iterated to a fixpoint:

**ARC Optimizer** (`lib/SILOptimizer/Transforms/ARCCodeMotion.cpp`, `ARCOptimize.cpp`): the central pass. Implements a global dataflow analysis that identifies `copy_value` / `destroy_value` pairs that can be eliminated (if they are balanced and the value is not aliased through an escaping path) or repositioned (hoisted before loops, sunk after joins). The analysis builds a **Reference Count State** lattice over basic blocks and solves it to a fixpoint. Key result: retain/release pairs on locally-owned objects in loops are hoisted/sunk; pairs in adjacent basic blocks are cancelled.

**Devirtualizer** (`lib/SILOptimizer/Analysis/ClassHierarchyAnalysis.cpp`): uses Swift's closed-world class hierarchy (or whole-module analysis) to convert `class_method` virtual dispatch to `function_ref` direct calls and `witness_method` protocol dispatch to `function_ref` when the concrete type is statically known. Devirtualized calls can then be inlined.

**Generic Specializer** (`lib/SILOptimizer/Transforms/GenericSpecializer.cpp`): monomorphizes generic functions. When a call site supplies concrete type arguments (e.g., `sort<[Int]>(…)`), the specializer clones the generic function, substitutes concrete types for type parameters, and marks the clone with `[specialized]`. The result is a `specialized $sort<Int>(…)` function that is concrete and can be devirtualized and inlined. This is analogous to C++ template instantiation but driven by the optimizer rather than the frontend.

**SIL Combiner** (`lib/SILOptimizer/Transforms/SILCombiner.cpp`): local peephole optimization. The SIL analog of LLVM's `InstructionCombiner`. Applies pattern-matching rewrites like: cancel `begin_borrow` / `end_borrow` pairs with no uses, fold `copy_value` of a value that is immediately `destroy_value`d into nothing, simplify `switch_enum` on a known-variant optional.

**Stack Promotion** (`lib/SILOptimizer/Transforms/AllocBoxToStack.cpp`, `EscapeAnalysis.cpp`): performs escape analysis over SIL values. If an `alloc_ref` object can be proven not to escape its allocating function (no `@owned` value derived from it is stored to a location that outlives the function, passed to a non-inlined callee, or captured by an escaping closure), the allocator is rewritten to `alloc_ref [stack]`, which compiles to stack allocation via `alloca` in LLVM IR. This eliminates the `swift_allocObject` call and the corresponding `swift_release` / `deinit` chain.

**Copy Propagation** (`lib/SILOptimizer/Transforms/CopyPropagation.cpp`): the central OSSA-era pass for eliminating `copy_value` / `destroy_value` pairs. In OSSA form, every assignment of a reference-typed value inserts a `copy_value` to satisfy the linear-use invariant (the original value is still live and must be independently consumed). Copy propagation tracks the use-def chain of each `@owned` SSA value and identifies cases where a `copy_value` is immediately dominated by a `destroy_value` of the original — in which case the copy is redundant and both instructions can be deleted. The pass also handles the more complex case of canonical copy shortcutting: if `%x = copy_value %n` and `%x` is consumed on all paths while `%n` is also consumed immediately after all uses of `%x` complete, the copy can be replaced by forwarding the original ownership. This is the OSSA-era replacement for the pre-OSSA retain/release pair cancellation that the ARC Optimizer performed through state machine matching.

**Function Signature Optimizer** (`lib/SILOptimizer/FunctionTransforms/FunctionSignatureOpts.cpp`): removes unused function parameters, converts `@owned` parameters that are only used in `@guaranteed` positions to `@guaranteed` (eliminating a retain/release pair at each call site), and removes return values that are always ignored.

**Semantic Arc Opts** (`lib/SILOptimizer/Transforms/SemanticARCOpts.cpp`): OSSA-aware peephole pass that performs optimizations on the semantic ARC instructions directly, without converting to `strong_retain` / `strong_release`. Key rewrites: eliminate a `begin_borrow` / `end_borrow` pair where the borrowed value is only used through the borrow (the borrow scope is unnecessary if the enclosing `@owned` value is not consumed inside the scope), eliminate `copy_value` of an `@guaranteed` value followed by `destroy_value` (the copy was only needed for the consume, which can be removed since the `@guaranteed` value is already live), and simplify `copy_value` / `begin_borrow` sequences.

### Stage 4: Lowered SIL and IRGen

After optimization, the SIL PassManager runs a final lowering phase that rewrites all OSSA ownership instructions to explicit ARC calls (`strong_retain` / `strong_release`), removes `begin_borrow` / `end_borrow` markers (they become no-ops — the liveness information they encode has already been used), and removes `begin_access [static]` markers. Only `begin_access [dynamic]` markers remain and lower to runtime calls.

**IRGen** (`lib/IRGen/`) then translates Lowered SIL to LLVM IR instruction by instruction.

---

## 192.5 Existentials, Generics, and the Resilience Model

### Protocol Witness Tables

SIL's generic dispatch mechanism is the **protocol witness table** — a runtime structure analogous to a C++ vtable but generated per *conformance* rather than per *class*. For a type `MyType` conforming to protocol `Hashable`, the Swift compiler generates a witness table `MyType: Hashable` containing function pointers to `MyType`'s implementations of each `Hashable` protocol requirement. In SIL:

```sil
witness_table MyType: Hashable {
  method #Hashable.hash: @MyType_hash
  method #Hashable._rawHashValue: @MyType_rawHashValue
}
```

Generic function calls that dispatch through witness tables use `witness_method` in SIL and resolve to an indirect function pointer load in LLVM IR. The devirtualizer eliminates witness table lookups when the concrete conforming type is statically known.

### Value Witness Tables

Every type in Swift — including generic types and existential types — has a **value witness table** that describes its runtime layout: size, stride, alignment, and function pointers for `initializeWithCopy`, `assignWithCopy`, `initializeWithTake` (a move), `assignWithTake`, and `destroy`. The value witness table is the mechanism by which generic code can copy or destroy values of unknown type: it calls the appropriate function pointer from the table. This is analogous to LLVM's generic `memcpy`/`memset` but with type-aware ARC semantics embedded.

### Resilient Types and Library Evolution

When Swift code is compiled with `-enable-library-evolution` (the setting used for the Swift Standard Library and Apple frameworks distributed in the OS), types that are public and not `@frozen` are **resilient**: their layout is not fixed at compile time. Client code cannot directly access fields; it must use generated accessors. In SIL, access to a resilient struct field goes through a `_read` coroutine (which produces a `@guaranteed` value via `yield`) or a `_modify` coroutine (which produces an `@inout` value via `yield`). This enables the library to change the field layout without breaking ABI compatibility — a key requirement for distributing frameworks in the OS.

---

## 192.6 Move-Only Types: Swift's `~Copyable`

Swift 5.9 introduced **noncopyable types** via the `~Copyable` constraint, tracked through Swift Evolution proposals SE-0366 (noncopyable generics), SE-0377 (`borrow` and `consume` operators), and SE-0430 (`~Escapable` types).

### `~Copyable` at the Type Level

A type annotated `~Copyable` cannot be implicitly copied by the compiler. Any operation that would insert a `copy_value` in SIL — assignment, passing as an argument without an explicit `borrow` — is a compile error. The only way to duplicate such a type is to implement a custom initializer that takes a `borrowing` argument and explicitly constructs a new value. This is directly analogous to Rust's `!Copy` trait: a type that does not implement `Copy` cannot be duplicated by memcpy; ownership must be explicitly transferred.

```swift
struct FileDescriptor: ~Copyable {
    private var fd: Int32
    init(fd: Int32) { self.fd = fd }
    consuming func close() { /* close(fd) */ }
    deinit { /* close if not already closed */ }
}

func use(_ desc: consuming FileDescriptor) {
    // 'consuming' means this function takes @owned FileDescriptor
    desc.close()
    // desc is consumed; no deinit call needed — close() transferred ownership
}
```

In SIL, a `consuming` method takes its `self` as `@owned`. A `borrowing` method takes `self` as `@guaranteed`. A `mutating` method takes `self` as `@inout`. For `~Copyable` types, the verifier enforces that no implicit `copy_value` is inserted — the only permitted introduction of a new `@owned` value is through an explicit `consume` expression or through a constructor.

### `~Escapable` and `mark_dependence`

SE-0430 introduces `~Escapable` — types whose values cannot outlive the scope that contains them. A slice type that holds a pointer into a buffer is `~Escapable`: the slice is only valid while the buffer is alive. In SIL, the compiler inserts `mark_dependence` to express this:

```sil
// %slice's lifetime depends on %buffer being alive
%slice = mark_dependence %raw_slice : $BufferSlice
    on %buffer : $ContiguousArray<Int>
```

The `mark_dependence` instruction tells the optimizer that `%buffer` must remain alive for as long as `%slice` is reachable. This prevents stack promotion of `%buffer` past the point where `%slice` is last used, and prevents the ARC optimizer from sinking the retain of `%buffer` below the last use of `%slice`. `mark_dependence` is the SIL mechanism for enforcing lifetime relationships that Rust enforces through its borrow checker's lifetime regions.

---

## 192.7 SIL's Influence on MLIR

SIL's design informed several central design decisions in MLIR's infrastructure. The connection is direct: several MLIR contributors were Swift compiler contributors, and the design discussions around MLIR's bufferization were explicitly informed by SIL's ownership model.

### Bufferization as Ownership Transfer

MLIR's **one-shot bufferization** (see [Chapter 148 — MLIR Bufferization](../part-21-mlir-transformations/ch148-mlir-bufferization.md)) solves a problem structurally identical to SIL's `@owned` / `@guaranteed` distinction. At the `tensor` level, MLIR operations produce new tensors with value semantics — every operation conceptually produces a distinct tensor. Bufferization must decide:

1. When a tensor-typed SSA value is the **unique owner** of its backing storage (→ in-place update is safe, analogous to `@owned`)
2. When a tensor-typed SSA value is a **read-only view** of storage that may be shared (→ must copy, analogous to `@guaranteed`)

The `bufferization.alloc_tensor` operation creates a new tensor with unique ownership semantics — exactly like SIL's `alloc_ref [stack]` for a value that is known not to escape. The `bufferization.to_tensor` / `bufferization.to_memref` operations form a boundary layer between the tensor (value-semantic) world and the memref (address-semantic) world, directly analogous to SIL's boundary between `@owned` values and `$*T` addresses.

One-shot bufferization's alias analysis determines which tensor values alias (share storage) and which do not. Values that do not alias are in-placed; values that might alias are copied. This is the MLIR restatement of SIL's OSSA analysis: the OSSA verifier proves that `@owned` values are not aliased through consuming uses; one-shot bufferization proves that tensor values are not aliased through conflicting writes.

### The `linalg` Dialect's Destination-Passing Style

The `linalg` dialect (see [Chapter 139 — Tensor and Linalg](../part-20-in-tree-dialects/ch139-tensor-linalg.md)) distinguishes `ins` operands (inputs, read-only) from `outs` operands (outputs, written to) in its generic and named operations. The `outs` operand is the **destination** — it is consumed and replaced by the result:

```mlir
// ins: @guaranteed reads; outs: @owned destination (consumed, result produced)
%result = linalg.matmul
    ins(%A, %B : tensor<4x4xf32>, tensor<4x4xf32>)
    outs(%C    : tensor<4x4xf32>) -> tensor<4x4xf32>
```

This `ins`/`outs` distinction is a direct encoding of SIL's `@guaranteed` vs `@owned` semantics: `ins` operands are borrowed (no ownership transfer; the caller still holds a live reference), `outs` operands are owned (ownership transferred to the operation; the result is a new owned value). This design avoids the aliasing ambiguity that would arise if all operands were treated uniformly.

### The Buffer Deallocation Pass

MLIR's buffer deallocation pass (`mlir/lib/Dialect/MemRef/Transforms/BufferDeallocation.cpp`) inserts `memref.dealloc` operations to balance `memref.alloc` operations. The pass performs a liveness analysis over the control-flow graph to determine the last point on every path where each allocated buffer is live, and inserts a `dealloc` at the appropriate dominance frontier. This is structurally identical to SIL's ARC optimizer: both identify the last use of an owned value on every control-flow path and insert the corresponding "consume" (release / dealloc) at that point. The SIL ARC optimizer uses a Reference Count State lattice; the buffer deallocation pass uses a Liveness lattice — different domains, same structural analysis.

### Region-Based Ownership for Nested Regions

MLIR's region model introduces a structural ownership concept: values defined in an outer region and used in an inner region (e.g., loop iteration variables captured by a `scf.for` body) must remain live for the duration of the inner region's execution. This is enforced by the dominance rules of MLIR's SSA form — the outer value dominates all uses in the inner region — and by the fact that MLIR regions may not let values escape: an inner-region SSA value cannot be used in an outer region (values must be explicitly yielded through `scf.yield` or equivalent). This is the MLIR analog of SIL's `@guaranteed` scoping through `begin_borrow` / `end_borrow`: the borrow scope of the outer value is the duration of the inner region's execution.

### The Influence on MLIR's Value Semantics Design

The broader design principle that SIL contributed to MLIR's architecture is the separation between **value semantics** and **reference semantics** at the IR level. In SIL, class types have reference semantics (ARC-managed), value types have value semantics (copied freely, no ARC), and the boundary between them is explicit in the type system. MLIR similarly separates `tensor` types (value semantics, no aliasing, owned by their SSA value) from `memref` types (reference semantics, aliased, managed through deallocation passes). The `bufferization.to_tensor` / `bufferization.to_memref` boundary operations are the MLIR equivalent of SIL's boundary between value and reference worlds.

This design choice — that IRs should encode the semantic contract of values, not just their physical representation — is the common thread running from SIL through MLIR's dialect design. When MLIR operations specify which operands they consume (take ownership of) vs. which they merely read (borrow), they implement the same principle that SIL's `@owned` / `@guaranteed` qualifiers encode. The `linalg.generic` `ins`/`outs` separation, the bufferization aliasing analysis, and the buffer deallocation liveness analysis are all downstream consequences of taking this principle seriously in IR design. The influence is neither accidental nor superficial: several of the engineers who designed MLIR's bufferization system had direct experience with SIL's ownership model and applied its lessons explicitly when designing the tensor/memref split and one-shot bufferization algorithm.

---

## 192.8 IRGen: SIL to LLVM IR

IRGen (`lib/IRGen/`) is the final translation stage, converting Lowered SIL to LLVM IR. After OSSA lowering, SIL values no longer carry ownership qualifiers — they are raw pointers and integers — and the IRGen pass performs a straightforward structural translation.

### Heap Allocation and the Swift Object Header

`alloc_ref $Node` (that was not stack-promoted) lowers to a call to `swift_allocObject`:

```llvm
; The type metadata pointer identifies the class layout and vtable
%0 = call noalias %swift.refcounted* @swift_allocObject(
    %swift.type* @"$s4main4NodeCN",   ; type metadata
    i64 32,                            ; size of instance
    i64 7                              ; alignment mask (8-byte aligned → mask 7)
)
; Returns a pointer to the object (header + fields follow)
```

The object header (defined in `stdlib/public/SwiftShims/HeapObject.h`) consists of:
- A pointer to the class's **isa** (type metadata, which contains the vtable)
- A **reference count** word (contains the strong RC, unowned RC, and weak RC in bitfields using a lock-free scheme)

### ARC Calls in LLVM IR

After OSSA lowering:
- `copy_value %x : $Node` → `call void @swift_retain(%swift.refcounted* %x)`
- `destroy_value %x : $Node` → `call void @swift_release(%swift.refcounted* %x)`
- Stack-promoted `alloc_ref [stack] $Node` → `alloca [N x i8]` + GEP to the instance data

`swift_retain` and `swift_release` are defined in `stdlib/public/runtime/HeapObject.cpp`. They use a custom lock-free reference counting scheme with bitfields for strong, unowned, and weak counts packed into a single `uintptr_t`.

### Witness Table Dispatch in LLVM IR

A SIL `witness_method` + `apply` sequence lowers to:

```llvm
; Load the witness table pointer (passed as an implicit extra parameter to generic functions)
%wt = load %swift.protocol_witness_table*, %swift.protocol_witness_table** %wt_addr

; Index into the witness table to get the function pointer
%fn_ptr_addr = getelementptr inbounds %swift.protocol_witness_table,
    %swift.protocol_witness_table* %wt,
    i32 0, i32 <slot_index>
%fn_ptr = load i8*, i8** %fn_ptr_addr

; Indirect call through the function pointer
%result = call swiftcc i64 %fn_ptr(%swift.opaque* %value, %swift.type* %T_metadata,
    %swift.protocol_witness_table* %wt)
```

### Boxing Value Types for Existentials

When a value type (struct or enum) is stored into an existential (`any Protocol`), it must be **boxed** — wrapped in a heap-allocated container if it exceeds the existential's inline buffer size (3 pointer words on 64-bit platforms). `swift_allocBox` allocates the box; `swift_projectBox` extracts the pointer to the value:

```llvm
; alloc_box $Int lowers to:
%box_pair = call { %swift.refcounted*, i64* } @swift_allocBox(%swift.type* @"$sSiN")
%box_ref  = extractvalue { %swift.refcounted*, i64* } %box_pair, 0
%box_data = extractvalue { %swift.refcounted*, i64* } %box_pair, 1
; %box_data is the address of the Int stored in the box
```

### LLVM IR for a Simple Method Call

A complete example: `process(_ n: Node) -> Int` compiled with `-O`:

```llvm
define swiftcc i64 @"$s4main7processySiAA4NodeCF"(%swift.refcounted* %0) {
entry:
  ; n is @guaranteed — no swift_retain needed on entry
  ; Compute the address of the 'value' field (offset 16 past the object header)
  %field_ptr = getelementptr inbounds <{ %swift.refcounted, %TSi }>,
      <{ %swift.refcounted, %TSi }>* %0,
      i32 0, i32 1, i32 0
  %value = load i64, i64* %field_ptr, align 8
  ret i64 %value
  ; n is @guaranteed — no swift_release needed
}
```

With `-O` and a `@guaranteed` parameter, the generated LLVM IR contains no ARC calls at all. The retain/release elimination happened at the SIL level — the LLVM IR never sees them. This is the payoff of encoding ownership in SIL: optimizations that would require sophisticated alias analysis in LLVM IR (proving that `swift_retain` / `swift_release` calls can be removed) are instead performed structurally in SIL's OSSA form, where they require only linear-use verification.

---

## 192.9 Cross-Cutting Themes and Practical Guidance

### Debugging SIL

The most important diagnostic workflow when investigating ARC issues or compiler crashes is:

```bash
# Dump SIL at a specific pass boundary
swiftc -emit-sil -Xfrontend -sil-print-pass-name=<PassName> example.swift

# Enable OSSA verification after every mandatory pass
swiftc -Xfrontend -sil-verify-all example.swift

# Show SIL for a specific function only
swiftc -emit-sil -Xfrontend -emit-verbose-sil \
    -Xfrontend -sil-print-function=process example.swift | swift-demangle

# Run sil-opt on a SIL file with specific passes
sil-opt -enable-library-evolution -arc-sequence-opts -compute-dominance-info \
    -sil-print-pass-name example.sil
```

### Relationship to ABI Stability

Swift's ABI stability (introduced in Swift 5.0 for Apple platforms) imposes constraints on SIL optimization. Resilient types must be accessed through their accessors; witness tables for public conformances must have a stable layout; generic specializations for public generic functions cannot be assumed to exist at client call sites (the client may link against a binary compiled before the specialization existed). These constraints are encoded in `lib/SILOptimizer/Analysis/ResilienceExpansion.cpp` and affect which optimizations are legal for `public` vs `internal` declarations.

In particular, the **ARC optimizer** must be conservative about removing retain/release operations on resilient types: a resilient struct's size is not known at compile time for clients, so the compiler cannot know whether a value witness operation will call user code. The generic specializer cannot specialize public generic functions for client code unless the function is marked `@_specialize` with an exported specialization attribute. These restrictions reflect the same tension present in LLVM's IPO framework between whole-program optimization (which can see all call targets) and modular compilation (which cannot) — discussed in [Chapter 65 — Inter-Procedural Optimizations](../part-10-analysis-middle-end/ch65-ipo.md).

### Whole-Module Optimization

Swift's equivalent of LTO is **Whole-Module Optimization (WMO)**, enabled with `-whole-module-optimization`. In WMO mode, the compiler compiles all source files in a module simultaneously, giving SILGen and the SIL optimizer visibility into every function and type in the module. This enables:

- **Cross-file devirtualization**: the class hierarchy is fully known; `class_method` calls to `internal` classes can always be devirtualized.
- **Cross-file generic specialization**: a generic function in one file can be specialized for concrete types used only in another file.
- **Cross-file inlining**: `@_inlineable` and `@inline(__always)` functions defined in one file are inlined at call sites in other files.

WMO is the standard build mode for release builds in Xcode and Swift Package Manager (`-c release`). It is structurally similar to LLVM LTO (see the LTO chapters in Part XIII) but operates at the SIL level rather than the LLVM IR level, which means all OSSA-level ownership optimizations benefit from the whole-program view. After WMO optimization, IRGen produces LLVM IR that is then passed to the LLVM backend for machine-code generation in the standard way.

### LLVM IR for `makeChain` — The Full Picture

A more complex example shows the full ARC lifecycle. `makeChain` returns an `@owned` `Node` and calls the `Node` initializer, which itself returns `@owned`:

```llvm
; Simplified IRGen output for makeChain(_:) with -O
define swiftcc %swift.refcounted* @"$s4main9makeChainySiF"(i64 %v) {
entry:
  ; Allocate first Node: swift_allocObject(metadata, size=32, alignMask=7)
  %n = call noalias %swift.refcounted*
      @swift_allocObject(%swift.type* @"$s4main4NodeCN", i64 32, i64 7)
  ; Store 'value' field (offset 16 = 8-byte header + 8-byte isa)
  %n_val_ptr = getelementptr inbounds i8, i8* %n, i64 16
  %n_val_i64 = bitcast i8* %n_val_ptr to i64*
  store i64 %v, i64* %n_val_i64, align 8

  ; Compute v+1 for the next node
  %v_plus_1 = add i64 %v, 1

  ; Allocate second Node
  %next = call noalias %swift.refcounted*
      @swift_allocObject(%swift.type* @"$s4main4NodeCN", i64 32, i64 7)
  %next_val_ptr = getelementptr inbounds i8, i8* %next, i64 16
  %next_val_i64 = bitcast i8* %next_val_ptr to i64*
  store i64 %v_plus_1, i64* %next_val_i64, align 8

  ; Store %next into n.next (Optional<Node>.some case)
  ; next field is at offset 24; stored as an Optional (tagged pointer or enum)
  %n_next_ptr = getelementptr inbounds i8, i8* %n, i64 24
  %n_next_typed = bitcast i8* %n_next_ptr to %swift.refcounted**
  ; swift_retain(%next) — because we're storing into n.next (n.next takes @owned)
  ; while %next is still alive as the local variable
  call void @swift_retain(%swift.refcounted* %next)
  store %swift.refcounted* %next, %swift.refcounted** %n_next_typed, align 8
  ; destroy local 'next' variable — swift_release
  call void @swift_release(%swift.refcounted* %next)

  ; Return %n as @owned — the caller is responsible for releasing
  ret %swift.refcounted* %n
}
```

Notice the `swift_retain` / `swift_release` pair around the store of `%next` into `n.next`. At the SIL level, this was a `copy_value %next : $Node` followed by a `store [assign]` into the field address, then a `destroy_value %next : $Node`. The Copy Propagation pass could not eliminate this pair because `%next` is used twice (assigned to `n.next` and the local variable `n` both hold a reference). After OSSA lowering the pair becomes explicit ARC calls. LLVM's optimizer may then be able to apply `swiftcc`-specific annotations and `swift_retain` / `swift_release` pairing via the `ObjCARCOpt` pass if enabled, but this is a secondary optimization layer.

### The ARC Reference Count Word

Swift's reference count word (defined in `stdlib/public/SwiftShims/RefCount.h`) encodes four conceptual counts in a lock-free bitfield scheme: strong retain count, unowned retain count, weak retain count, and a set of flags (is-deiniting, is-immortal, side-table pointer). The lock-free scheme uses `std::atomic<uint32_t>` with compare-exchange loops. Understanding this layout is essential when interpreting IRGen output for retain/release sequences, as the actual atomic operations are more complex than a simple `fetch_add`.

### Interaction with LLVM IPO Passes

After IRGen produces LLVM IR, the standard LLVM optimization pipeline runs. At this point the ARC calls are opaque to LLVM unless marked with specific attributes. The Swift compiler marks `swift_retain` / `swift_release` with a combination of LLVM function attributes (`inaccessiblememonly`, `nounwind`, no `readnone` — retain/release have side effects on the heap object's RC word) that allow LLVM to reason about them partially. In particular, LLVM's LICM pass will not hoist a `swift_retain` call out of a loop unless it can prove the call has no side effects visible to the loop body — which it cannot in general. This is precisely why SIL-level ARC optimization is essential: by the time LLVM sees the IR, the opportunity for structural ARC optimization has already been exploited in SIL. What remains in LLVM IR are only the ARC calls that genuinely cannot be eliminated — those that cross function boundaries or alias through memory that SIL's analysis could not track. For interprocedural ARC optimization in LLVM IR, see the `ObjCARCOpts` pass (`llvm/lib/Transforms/ObjCARC/ObjCARCOpts.cpp`) which implements similar pairing analysis but using LLVM's memory model — a useful cross-reference from [Chapter 65 — Inter-Procedural Optimizations](../part-10-analysis-middle-end/ch65-ipo.md).

---

## Chapter Summary

- **SIL exists because LLVM IR cannot optimize ARC**: LLVM's alias analysis treats `swift_retain` / `swift_release` as opaque calls; SIL makes ownership an explicit first-class invariant of the IR type system through Ownership SSA.

- **Four ownership qualifiers govern every SIL value**: `@owned` (linear — must be consumed exactly once), `@guaranteed` (borrowed — no ARC, scope-bounded), `@inout` (exclusive write access), and `@unowned` (unsafe unretained). These qualifiers appear in function types and are verified by the Memory Lifetime Verifier.

- **The SIL instruction set expresses ownership explicitly**: `alloc_ref`, `copy_value`, `destroy_value`, `begin_borrow`/`end_borrow`, `begin_access`/`end_access`, `witness_method`, `partial_apply`, `mark_dependence` — all carry or enforce ownership semantics that LLVM IR cannot represent.

- **The SIL pipeline is staged**: SILGen (Raw SIL, OSSA) → Mandatory passes (DI, exclusivity enforcement, mandatory inlining) → Performance optimization (ARC optimizer, devirtualizer, generic specializer, stack promotion) → OSSA lowering → IRGen to LLVM IR.

- **SIL's ARC optimizer and stack promoter eliminate retain/release pairs and heap allocations** at the SIL level, before LLVM IR is produced — this is the structural advantage of encoding ownership in the IR.

- **Move-only types (`~Copyable`, `~Escapable`) extend SIL's ownership model** to user-defined types, analogous to Rust's `!Copy` and lifetime-bounded references; `mark_dependence` encodes the non-escapability constraint in the IR.

- **MLIR's bufferization, `linalg` destination-passing style, and buffer deallocation pass are directly informed by SIL**: the `@owned` / `@guaranteed` distinction maps to tensor ownership / read-only view; one-shot bufferization's alias analysis parallels OSSA's linear-use verification; the buffer deallocation pass implements the same Reference Count State lattice analysis as SIL's ARC optimizer.

- **IRGen lowers `@owned` to `swift_retain` / `swift_release`** and witness table calls to indirect function pointer loads; a `@guaranteed` parameter produces LLVM IR with no ARC calls at all — the ownership information has already served its purpose at the SIL level.

- **Canonical sources**: [SIL reference manual](https://github.com/swiftlang/swift/blob/main/docs/SIL.rst); Swift Evolution SE-0366, SE-0377, SE-0430; Jordan Rose, "Ownership Is Membership" (LLVM Dev Meeting 2018); Groff/Lattner, *Swift: A Modern Programming Language for Safety* (2015).

---

*@copyright jreuben11*
