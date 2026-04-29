# Chapter 15 — Type Theory in Practice: From Theory to LLVM and MLIR

*Part III — Type Theory*

The previous three chapters built a formal foundation: the simply typed lambda calculus and its soundness theorems (Chapter 12), parametric polymorphism and Hindley-Milner inference (Chapter 13), and the full spectrum from dependent types through linear and affine type systems (Chapter 14). That theoretical apparatus is not merely academic — it is instantiated, in whole or in part, by every component of the LLVM ecosystem. This chapter is the bridge.

We map each theoretical concept to its engineering counterpart: LLVM IR's deliberately stripped type system, the monomorphization imperative that erases polymorphism before code generation, MLIR's richer parameterized type universe, Clang's layered C/C++ type representation, Rust's affine type checker over MIR, ML compilers' path from Hindley-Milner to LLVM through typed intermediates, Alive2's refinement-style verification, and the `constexpr`/`consteval` subset as a total-functional-programming sublanguage within C++. By the end of the chapter the theory of Parts II and III should feel less like an abstraction and more like a blueprint you can trace into the source tree.

---

## 1. LLVM IR's Type System: A Stripped-Down, Non-Polymorphic Design

### Ground Theory

Chapter 12 established that the simply typed lambda calculus (STLC) has a type universe: a grammar of base types and function types, with terms assigned unique types by inference rules. LLVM IR's type system is best understood as a *flat, ground instantiation* of that universe — every type is concrete and first-order, with no type variables, no quantifiers, and no type-level computation. The polymorphism of Chapter 13 is deliberately absent. What remains is a lean, machine-oriented annotation system whose primary purpose is register allocation, memory layout, and a sanity check on instruction operands. Chapter 17 — The Type System covers every LLVM IR type in full technical detail; this section focuses on the type-theoretic interpretation.

### The Type Grammar

LLVM IR's type grammar (LLVM 22, opaque-pointer era) is:

```
Type  ::=  void
        |  i1 | i8 | i16 | i32 | i64 | i128    -- integer types
        |  half | bfloat | float | double        -- floating-point types
        |  ptr                                   -- opaque pointer
        |  [N x T]                               -- array: N elements of type T
        |  {T1, T2, ..., Tn}                     -- literal (anonymous) struct
        |  <N x T>                               -- fixed-length SIMD vector
        |  <vscale x N x T>                      -- scalable vector (SVE, RVV)
        |  T1(T2, ..., Tn)                       -- function type
        |  %name = type { ... }                  -- named (identified) struct
```

Integer types are parameterized by bit width: `i1` through `i8388607`. There are no separate `signed` and `unsigned` variants — signedness is an attribute of instructions (`add nsw`, `udiv`, `sext`, `zext`), not of types. The type `i1` doubles as the Boolean type.

The vector types ground the Chapter 12 notion of parameterized types at the lowest level. A fixed vector `<4 x i32>` is an `i32` raised to a four-element product, while the scalable `<vscale x 4 x i32>` represents `4 × vscale` elements where `vscale` is a runtime multiplier provided by the target (e.g., SVE's `VL` register or RISC-V's `vlen`). Neither is polymorphic: the element type and (static) multiplier are fixed at compile time.

```llvm
; Fixed vector arithmetic
define <4 x i32> @add_vec4(<4 x i32> %a, <4 x i32> %b) {
  %r = add <4 x i32> %a, %b
  ret <4 x i32> %r
}

; Scalable vector (SVE / RISC-V V extension)
define <vscale x 4 x i32> @add_svec(<vscale x 4 x i32> %a, <vscale x 4 x i32> %b) {
  %r = add <vscale x 4 x i32> %a, %b
  ret <vscale x 4 x i32> %r
}
```

### Nominal vs. Structural Struct Types

LLVM IR has two distinct notions of struct type, mirroring the nominal/structural distinction of Chapter 14 §3:

**Literal (anonymous) struct types** are structurally identified. Two uses of `{i32, i32}` anywhere in a module denote the *same type*. There is no name and no identity beyond the field sequence. Structural equivalence is the rule: two literal struct types are equal if and only if their field types are pairwise equal, recursively.

**Named (identified) struct types** are nominally identified. `%struct.Point = type { i32, i32 }` and `%struct.Color = type { i32, i32 }` are *different types* even though their field sequences are identical, because LLVM identifies them by name within the module. The name is the identity, not the structure. This is exactly the nominal type discipline of Chapter 14 §3: the type checker accepts a `%struct.Point` where a `%struct.Point` is expected and rejects a `%struct.Color`, regardless of layout.

When Clang compiles two distinct C structs with the same fields, they become two distinct named LLVM IR types. The `qualtype_demo` example confirms this:

```llvm
%struct.Container   = type { i32 }      ; Container<int>
%struct.Container.0 = type { double }   ; Container<double>
```

Both have one field, but they are distinct named types. Template instantiation creates fresh named types — a concrete expression of the monomorphization principle covered in Section 2.

Named types enable **recursive types**. A singly-linked list node in C:

```c
struct Node { int val; struct Node* next; };
```

becomes:

```llvm
%struct.Node = type { i32, ptr }
```

Under the opaque-pointer regime (LLVM 16+, mandatory since LLVM 17), the `next` field is simply `ptr` — the recursive structure is recorded by the named type `%struct.Node`, not by an embedded `ptr to %struct.Node`. This is the **opaque struct pattern**: recursive types are expressed through named types whose pointer fields are untyped.

### Function Types

LLVM IR function types follow the Chapter 12 STLC function type grammar: `T1(T2, T3, ...)` is the type of a function returning `T1` and accepting parameters of types `T2`, `T3`, etc. In LLVM IR, function types are **not first-class value types** — you cannot have an `i32(i32, i32)` SSA value directly. You can only have a `ptr` that *refers* to a function. The function type appears in call instructions and in the declaration/definition of the function itself:

```llvm
; Function type: i32(i32, i32) — returns i32, takes two i32 args
define i32 @add(i32 %a, i32 %b) {
  %r = add nsw i32 %a, %b
  ret i32 %r
}

; Indirect call through a function pointer (ptr, not a typed function pointer)
define i32 @indirect_call(ptr %fp, i32 %x, i32 %y) {
  %result = call i32 %fp(i32 %x, i32 %y)
  ret i32 %result
}
```

Under the opaque-pointer regime, function pointers are just `ptr` — the same type as data pointers. The callee's type is specified at the call site in the call instruction itself (`call i32 %fp(i32 %x, i32 %y)` specifies the signature inline). This is a deliberate weakening of the typed-pointer model: the type annotation moves from the pointer's declared type to the individual call instruction, giving the optimizer maximum flexibility to reason about indirect calls.

Function types are structural in LLVM IR: `i32(i32)` declared in two different places in a module is the same type, not two distinct nominal types. This matches the function type rule of STLC: `τ1 → τ2` is identified purely by its domain and codomain types.

### The Opaque Pointer Migration

Before LLVM 15, pointer types carried their pointee type: `i32*`, `[4 x float]*`, `void (i32)*`. This allowed the IR verifier to check that a load or GEP matched the pointee type, but created significant engineering costs:

1. **Alias analysis coupling.** When two pointers had different pointee types, alias analysis could (incorrectly) conclude they could not alias — a type-based alias analysis (TBAA) shortcut that proved brittle as C code with type-punning became common. Clang's `!tbaa` metadata was partially a workaround.
2. **Bitcast proliferation.** Code that worked with polymorphic data structures (e.g., a generic linked list in C) required `bitcast` instructions at every type boundary, cluttering the IR and creating work for the simplifier.
3. **Opaque struct complexity.** Forward-declared structs required `%T = type opaque` and pointer-to-opaque handling in every pass.

The migration proceeded in stages: opaque `ptr` arrived as an opt-in in LLVM 15, became the default in LLVM 16, and typed pointer syntax was removed entirely in LLVM 17. LLVM 22 has no memory of typed pointers; `ptr` is the only pointer type. What is lost is the lightweight type assertion that a GEP-based pointer arithmetic chain is consistent with the pointee type. What is gained is a simpler IR with fewer casts, and a cleaner separation between pointer provenance (tracked through alias analysis metadata) and pointer type (irrelevant).

The pointee-type information now lives entirely in `!tbaa` and `!tbaa.struct` metadata nodes, which are semantically distinct from the type system: they express *aliasing constraints* rather than *structural invariants*. This is a principled separation — alias analysis is a semantic property of the memory model, not a type-theoretic property of the value.

### Integer Signedness: A Type-Level Absence

One of the more consequential design choices in LLVM IR's type system is the *absence* of signed and unsigned integer types. In most type theories (Chapter 12's base type set, Rust's `i32` vs `u32`, Haskell's `Int` vs `Word`), signedness is part of the type. In LLVM IR, it is an *instruction property* rather than a type property.

`i32` in LLVM IR is neither signed nor unsigned — it is a 32-bit pattern. The same bit pattern `0xFFFFFFFF` may be interpreted as `-1` (two's complement) or `4294967295` (unsigned) depending on the instruction that reads it. The instruction carries the signedness:

```llvm
; Same i32 type; different signedness semantics per instruction
%signed_div   = sdiv i32 %a, %b    ; signed division (rounds toward zero)
%unsigned_div = udiv i32 %a, %b    ; unsigned division

%signed_ext   = sext i32 %a to i64    ; sign-extend
%unsigned_ext = zext i32 %a to i64    ; zero-extend

%signed_cmp   = icmp slt i32 %a, %b   ; signed less-than
%unsigned_cmp = icmp ult i32 %a, %b   ; unsigned less-than
```

This is a deliberate engineering decision that reduces the number of distinct types (no `s32` vs `u32` proliferation) at the cost of making signedness ambiguous when looking at a value's type alone. The same `i32` value can be correct in one context and wrong in another, depending entirely on whether the next instruction uses `sdiv` or `udiv`.

From a type-theoretic perspective, this is a *type erasure* of the signedness dimension. A source-level distinction that is semantically significant (signed and unsigned division produce different results for the same bit pattern) is represented at the IR level by an annotation on operations rather than on types. This is analogous to OCaml's erasure of the `mutable` qualifier: mutable fields and immutable fields have the same runtime representation, differing only in what operations the type system permits.

The practical consequence is that LLVM optimization passes must propagate signedness information themselves, through flags like `nsw` (no signed wrap) and `nuw` (no unsigned wrap), and through value range analysis. The optimizer *does* track signedness, but it does so through dynamic analysis (range inference) rather than static types.

### Type Safety in LLVM IR: Deliberately Weak

Chapter 12's type soundness theorem says: if a well-typed term takes a step, the result is either a value or another well-typed term (preservation), and well-typed terms are never stuck (progress). LLVM IR's type system makes no such guarantee. It is a *sanity check*, not a safety proof.

The `inttoptr` and `ptrtoint` instructions are the canonical escape hatches:

```llvm
; Convert an integer to a pointer — no type-level restriction
define ptr @int_to_ptr(i64 %addr) {
  %p = inttoptr i64 %addr to ptr
  ret ptr %p
}

; Convert a pointer to an integer
define i64 @ptr_to_int(ptr %p) {
  %i = ptrtoint ptr %p to i64
  ret i64 %i
}
```

After `inttoptr`, the resulting `ptr` can be loaded from or stored to like any other pointer. The IR verifier does not object. Type safety at the LLVM IR level means: every instruction's operands have the types the instruction expects — not that the execution is well-defined. Poison and `undef` (covered formally in Chapter 171) represent further holes: `undef` is an uninitialized value that may produce any bit pattern on each read, while `poison` is a stronger condition that taints any computation that uses it, permitting the optimizer to assume the value is never reached. Both are accommodations to the demands of aggressive optimization, not features of a safe type system. The source-level safety guarantee is entirely the responsibility of the frontend and the formal models above LLVM IR — Clang's Sema, Rust's borrow checker, and the formal proofs of Alive2 (Section 7).

---

## 2. Why Generics Monomorphize Before Reaching LLVM

### The Polymorphism Erasure Imperative

Chapter 13 established that a term of type `∀α. τ` can be used at any ground instance `τ[T/α]`. The question for a compiler is: at runtime, what machine code executes for `id 42` versus `id true`? There are two fundamental strategies:

- **Monomorphization**: compile a separate copy of the code for each type instantiation. `id_int` and `id_bool` are distinct functions with distinct LLVM IR.
- **Type erasure / boxing**: compile a single copy that works uniformly over a runtime representation of values (typically a pointer or a fat pointer), at the cost of not knowing the layout of the underlying type.

LLVM IR supports neither approach natively: it has no `∀` type constructor. Every generic must be resolved to ground types before the LLVM layer sees it. The strategy choice is made by each language frontend.

### C++ Templates: Full Monomorphization

C++ templates are monomorphized entirely during Clang's template instantiation pass (Chapter 34). Each instantiation `std::vector<int>` and `std::vector<float>` produces a distinct set of Clang AST nodes, which in turn produce distinct LLVM IR functions with distinct mangled names:

```llvm
; Two monomorphized instantiations of a trivial add<T> template
; Source: template<typename T> T add(T a, T b) { return a + b; }

define dso_local noundef i32 @_Z7add_intii(i32 noundef %0, i32 noundef %1) {
  %3 = add nsw i32 %1, %0
  ret i32 %3
}

define dso_local noundef float @_Z9add_floatff(float noundef %0, float noundef %1) {
  %3 = fadd float %0, %1
  ret float %3
}
```

The mangled names `_Z7add_intii` and `_Z9add_floatff` encode the template instantiation into the symbol. There is no shared code path. The LLVM optimizer sees each instantiation independently and can specialize it fully: constant folding, inlining, vectorization all proceed without the overhead of runtime type dispatch.

The trade-off is binary size and compile time. `std::vector<T>` instantiated for 20 different element types produces 20 sets of member function definitions. LTO (Chapter 77) can partially mitigate this through dead code elimination after link-time visibility, but the baseline cost is real. This is why `std::vector<int>` and `std::vector<std::string>` each appear separately in the symbol table of any non-trivial C++ binary — a direct manifestation of monomorphization at the ABI level.

### Rust Generics: Monomorphize by Default, Erase via `dyn Trait`

Rust follows the same monomorphization strategy for generic functions:

```rust
fn add<T: std::ops::Add<Output=T>>(a: T, b: T) -> T { a + b }

let _i = add(1i32, 2i32);   // compiled as add::<i32>
let _f = add(1.0f64, 2.0f64); // compiled as add::<f64>
```

Each concrete instantiation produces its own LLVM IR function. The trait bound `T: Add` is resolved at monomorphization time: the compiler statically selects the `Add` implementation for `i32` or `f64` and inlines it. No runtime dispatch occurs.

The alternative is **`dyn Trait`**, Rust's dynamic dispatch mechanism. A `&dyn Trait` is a **fat pointer**: a pair `(data_ptr, vtable_ptr)` where the vtable contains function pointers for each trait method. This is type erasure: the type `T` is forgotten at the `dyn Trait` boundary, and method calls go through the vtable. The LLVM IR for a function taking `&dyn Trait` receives a `ptr` pair — no monomorphization occurs, no type information survives, and the optimizer cannot inline across the vtable call.

The fat pointer is a direct implementation of the existential type `∃T. (T, Trait_vtable(T))` discussed in Chapter 13 §3. The vtable encodes the witness to the trait implementation — analogous to the dictionary passed in Haskell's type-class translation, but embedded in a pointer pair rather than a function parameter.

### Go Generics (1.18+): GC Shape Stenciling

Go 1.18 introduced generics with a compromise between full monomorphization and full type erasure. The compiler groups type instantiations by their **GC shape** — a type descriptor that captures the garbage-collector's view of a value: its size, alignment, and pointer bitmap. Two types with the same GC shape share a single compiled copy of the generic function.

Concretely: `int32` and `float32` have the same GC shape (4 bytes, no pointers) and share code; `*int` and `*string` both have the GC shape of a single pointer and share code. The shared copy receives an extra implicit parameter — a runtime type descriptor (the `gcshape dictionary`) — and uses it for type-specific operations like equality comparisons or interface method calls. For types with pointer-free GC shapes, the shared code is nearly as efficient as a monomorphized copy. For types whose shape differs, the compiler falls back to separate copies.

This is a pragmatic point in the design space: less code bloat than full monomorphization (C++, Rust) but less optimization opportunity than full monomorphization, and less runtime overhead than full boxing (Java/Haskell erasure-style). The GC shape dictionary is a restricted form of the dictionary-passing translation of Chapter 13 §5 — the "dictionary" is not a full type class witness but a small runtime descriptor.

### Haskell: Dictionary Passing Through GHC Core

Haskell type classes (Chapter 13 §5) are compiled by dictionary passing. A function:

```haskell
sort :: Ord a => [a] -> [a]
```

becomes, at the GHC Core level, a function taking an explicit `Ord a` dictionary:

```
sort :: forall a. Ord# a -> [a] -> [a]
```

where `Ord# a` is a record of function pointers `{compare : a -> a -> Ordering, (<) : a -> a -> Bool, ...}`. Each call site fills in the appropriate dictionary for the concrete type. The generated LLVM IR is a plain function taking a pointer to this dictionary struct — no polymorphism, no generics, no quantifiers. The `Ord` instance for `Int` is a global constant struct; the `Ord` instance for `[Char]` is a function that constructs the dictionary dynamically.

GHC's `SPECIALIZE` pragma forces monomorphization for a specific type instantiation:

```haskell
{-# SPECIALIZE sort :: [Int] -> [Int] #-}
```

This generates a copy of `sort` specialized for `[Int]`, inlining the `Ord Int` dictionary and enabling further optimizations — identical in effect to C++ template instantiation, just controlled by an explicit pragma rather than implicit instantiation.

GHC Core (System FC) is the typed intermediate representation in which this elaboration happens. It is a dependently-typed language with explicit type applications and coercion proofs. The elaborated `sort` is well-typed in System FC with the dictionary type visible. When GHC lowers Core to STG and then to Cmm and LLVM IR, the types are progressively erased until the LLVM layer sees only `ptr` and integer operands.

### Java and the JVM: Full Type Erasure with Runtime Reification

Java generics (introduced in Java 5) take the opposite extreme from C++ and Rust. The Java compiler erases all generic type parameters before generating bytecode:

```java
// Source: List<String> vs. List<Integer>
// Both erase to the same bytecode using raw List

List<String>  strings  = new ArrayList<>();
List<Integer> integers = new ArrayList<>();
// At runtime, both are just ArrayList — the type parameter is gone
```

The JVM sees only the erased type `Object` in place of `T`. Every value of a generic type variable must be a reference type (Java primitives cannot be generic type arguments without boxing — hence `List<Integer>`, not `List<int>`). No LLVM IR is involved (the JVM is bytecode-based), but the principle generalizes: the JVM executes a single copy of `sort(List<T> list)` regardless of `T`, with all element accesses going through `Object` references. This is pure type erasure in the sense of Chapter 13 §6: the `∀α` is implemented by the absence of any type-specific code.

The cost is boxing and unboxing. An `int` placed into a `List<Integer>` is heap-allocated as an `Integer` object; retrieving and using it requires unboxing. This is a runtime cost absent from monomorphized code. Project Valhalla (Java 22+) is working to add "value types" that allow unboxed generic primitives, which would effectively introduce partial monomorphization into the JVM model.

### Swift: Witness Tables as an Intermediate Position

Swift generics use **witness tables** (analogous to Haskell's dictionaries and Rust's vtables, but with a distinct runtime model). A generic function:

```swift
func sort<T: Comparable>(_ array: inout [T]) { ... }
```

is compiled to a single machine-code function that receives an implicit pointer to the `Comparable` witness table for type `T`. The witness table contains function pointers for `<`, `==`, etc. At call sites, Swift may specialize the function (like C++ monomorphization) or call the generic version (like dictionary passing), choosing based on whether the concrete type is visible.

Swift's specialization optimizer (`-Onone` uses witness tables; `-O` triggers specialization when beneficial) is a practical demonstration that the monomorphization/erasure binary is not absolute: it is a compile-time optimization decision that the compiler can make per-call-site. The final LLVM IR may contain both specialized and generic versions of the same function.

### The Design Decision

The monomorphization/erasure trade-off distills to: **optimization opportunity vs. code size and compile time**. The design space can be summarized as a continuum:

| Strategy | Example | Code size | Optimization | Runtime overhead |
|---|---|---|---|---|
| Full monomorphization | C++, Rust | High | Maximum | None |
| GC shape stenciling | Go 1.18+ | Medium | High for scalars | Small (type dict) |
| Witness table + specialize | Swift | Medium | Medium-High | Small when unspecialized |
| Dictionary passing | Haskell (no SPECIALIZE) | Low | Low (no inline) | Dict lookup |
| Type erasure + boxing | Java generics | Lowest | None for generics | Boxing/unboxing |

Monomorphization allows the optimizer to see concrete types, enabling inlining, specialization, and vectorization. Dictionary passing or boxing requires indirect calls through vtables or dictionary structs, preventing many optimizations but yielding a single compiled copy. There is no universally correct answer; the choice is part of a language's identity and performance contract with its users.

---

## 3. MLIR's Richer, Theory-Aligned Type System

### Beyond LLVM IR's Flat Types

MLIR is explicitly designed to host higher-level IRs that preserve structural information from the source program. Its type system is correspondingly richer than LLVM IR's, reflecting the Chapter 13 notion of parameterized types and the Chapter 14 notion of dependent-like type constructors.

The key design decisions are:

1. **Extensible type universe**: new types are added by dialects; the core provides only a handful of built-in types. A type unknown to the LLVM IR type checker is simply rejected; a type unknown to a given MLIR pass can be passed through opaquely and verified when it reaches a pass that understands it.
2. **Parameterized types**: types carry structural parameters (element type, shape, memory space) as part of their identity.
3. **Verifier-enforced invariants**: every type can define a `verify` method that checks its parameters; the MLIR verifier runs it on every use.
4. **Type interfaces**: structural properties (shapedness, pointedness) are expressed as interfaces that multiple concrete types can implement.

### Parameterized Types: tensor, memref, vector

The three canonical shaped types illustrate the parameterization principle:

```mlir
// Statically shaped 4×4 tensor of f32
tensor<4x4xf32>

// Dynamically shaped tensor (unknown dimensions)
tensor<?x?xf32>

// Ranked memref with a 2D static layout in memory space 0
memref<4x4xf32>

// Unranked memref (rank not known statically)
memref<*xf32>

// Fixed-length SIMD vector
vector<8xf32>

// Scalable vector (analogous to LLVM's <vscale x N x T>)
vector<[4]xf32>
```

`tensor<4x4xf32>` is the MLIR type that corresponds most directly to the dependent type `Matrix(f32, 4, 4)` in Martin-Löf Type Theory (Chapter 14 §1). The shape parameters `4` and `4` are values baked into the type. MLIR is not a dependent type system — the shape must be statically known or represented as `?` — but the *spirit* of dependent indexing is present: two tensors of different static shapes have different types, and the verifier enforces shape consistency across operations.

The `RankedTensorType` C++ class in the MLIR source tree [`mlir/include/mlir/IR/BuiltinTypes.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/BuiltinTypes.h) carries `ArrayRef<int64_t> shape` and `Type elementType` as fields in its `TypeStorage`. Two `RankedTensorType` instances with the same shape and element type are the *same type object* due to MLIR's uniquing mechanism — `TypeStorage` objects are interned by their parameters in a `StorageUniquer`. This is the structural identity rule: identity is determined entirely by parameters, not by nominal declarations, contrasting with LLVM IR's named struct nominality.

### Dialect-Defined Types: Extending the Universe

MLIR's most theory-significant capability is the ability to add entirely new types without touching the core. A dialect author defines a new type by:

1. Declaring a `TypeDef` in ODS (Operation Definition Spec) or C++ that specifies the type's parameters.
2. Implementing `TypeStorage` for the parameters.
3. Optionally implementing type interfaces the type satisfies.
4. Registering the type with the dialect.

A hardware design dialect might define a type `hw.int<8>` representing an 8-bit hardware integer distinct from LLVM's `i8`. A machine learning dialect might define `ml.quantized_tensor<8x8xui8, scale=0.1, zero_point=128>` carrying quantization parameters in the type. These types are first-class citizens in the MLIR type system: they appear in operation signatures, can be passed through generic passes as opaque values, and are verified by their own verifier.

This extensibility corresponds to the **open type universe** of type theories with universe polymorphism (Chapter 14 §1): new types can be added to the universe without a global case analysis. Contrast with LLVM IR, which has a closed type grammar — no extension mechanism exists short of modifying the LLVM source.

The connection to the Chapter 14 treatment of effect systems and session types is also worth noting. Some MLIR dialect types carry richer semantic content: the `spirv.coopmatrix` type from the SPIR-V dialect encodes the scope, element type, rows, columns, and use (A/B/accumulator) in its type parameters. A SPIR-V operation that expects an accumulator matrix is statically prevented from receiving an A-matrix — an invariant enforced by the type system at dialect verification time, analogous to the state-thread types of Chapter 14 §8.

### TypeStorage Uniquing

MLIR's type system is built on a storage-uniquing pattern. Every parameterized type defines a `TypeStorage` subclass holding its parameters:

```cpp
// Simplified illustration of RankedTensorType storage
struct RankedTensorTypeStorage : public TypeStorage {
  using KeyTy = std::pair<ArrayRef<int64_t>, Type>;

  RankedTensorTypeStorage(ArrayRef<int64_t> shape, Type elementType)
      : shape(shape), elementType(elementType) {}

  static llvm::hash_code hashKey(const KeyTy& key) {
    return llvm::hash_combine(key.first, key.second);
  }
  bool operator==(const KeyTy& key) const {
    return key == KeyTy{shape, elementType};
  }

  ArrayRef<int64_t> shape;
  Type elementType;
};
```

When user code calls `RankedTensorType::get(ctx, {4, 4}, Float32Type::get(ctx))`, the `StorageUniquer` hashes the key `({4,4}, f32)`, looks it up in a global table, and returns the existing instance if found or creates a new one if not. All uses of `tensor<4x4xf32>` in a module share a single C++ object. This uniquing is both a memory optimization and a type equality mechanism: pointer equality of type objects implies structural equality.

### Type Interfaces: Ad-Hoc Polymorphism

MLIR's type interface mechanism is the precise counterpart to type classes (Chapter 13 §5). An interface defines a set of method signatures without providing an implementation; a type opts in by implementing those methods. The `ShapedType` interface, for example, is implemented by `RankedTensorType`, `UnrankedTensorType`, `MemRefType`, `UnrankedMemRefType`, and `VectorType`:

```cpp
// Using the ShapedType interface generically
void processShape(mlir::ShapedType shaped) {
  if (shaped.hasRank()) {
    llvm::ArrayRef<int64_t> shape = shaped.getShape();
    mlir::Type elem = shaped.getElementType();
    // works for tensor, memref, or vector
  }
}
```

A pass that operates on `ShapedType` works uniformly over tensors, memrefs, and vectors without any dynamic dispatch in the type-theory sense — the dispatch occurs through the interface vtable, but from the type-theoretic perspective this is ad-hoc polymorphism: a single function name resolved statically to the correct implementation based on the concrete type. Compare with Haskell's `Foldable` or `Functor` — the interface describes a contract, and concrete types provide witnesses.

This mirrors the `TypeClass` / instance relationship of Chapter 13 §5 more faithfully than LLVM IR does. LLVM IR has no interfaces: every pass must switch on the concrete `Type::TypeID` or use `isa<>`/`dyn_cast<>` to recover type-specific information. MLIR's interface mechanism allows a pass to operate at the level of the interface, never needing to know the concrete type.

### Op Type Constraints: Verification as a Type-Level Property

MLIR operations are typed in a richer sense than LLVM IR instructions. Every op declares its operand and result type constraints in its ODS definition. The MLIR verifier checks these constraints when an op is created and when the module is verified. For example, the `linalg.matmul` op requires:

```mlir
// linalg.matmul with explicit operand types
%C = linalg.matmul
        ins(%A : memref<?x?xf32>, %B : memref<?x?xf32>)
        outs(%C : memref<?x?xf32>) -> memref<?x?xf32>
```

The verifier checks that:
- `%A` and `%B` are `MemRefType` with compatible ranks
- `%C` is a `MemRefType` compatible with the output shape
- The element types are consistent (all `f32` here)

A constraint violation — say, passing `memref<?x?xf64>` where `memref<?x?xf32>` is expected — is caught at op construction time, before any transformation runs. This is early error detection at the IR level, analogous to a type annotation in STLC that is checked before evaluation begins.

The constraint system is backed by ODS `TypeConstraint` objects, which can be as simple as `I32` (must be `i32`) or as complex as `TensorOf<[F32, F64]>` (must be a ranked or unranked tensor of f32 or f64). Custom verifier methods can check relationships between operand types (e.g., "the output rank equals the sum of the two input ranks"). This is a limited but practical form of dependent type checking: the verifier is a custom predicate on the op's type signature, evaluated at compile time.

### MLIR vs. LLVM IR: A Type-Theoretic Comparison

| Property | LLVM IR | MLIR |
|---|---|---|
| Base types | Fixed (`i32`, `f32`, ...) | Extensible (dialect-defined) |
| Parameterized types | No (types are ground) | Yes (`tensor<NxMxT>`, etc.) |
| Recursive types | Named structs | Named types, dialect-defined |
| Polymorphism | None | Type interfaces (ad-hoc) |
| Dependent-like indexing | None | Shape parameters in tensor/memref |
| Nominal vs. structural | Named=nominal, literal=structural | Storage uniquing = structural |
| Type verification | Instruction-level sanity | Verifier-enforced invariants |
| Regions / nested structure | No | Yes (nested regions typed) |

---

## 4. Clang's C/C++ Type Representation: QualType, Sugar, Dependent Types

### The QualType / Type* Duality

Chapter 12's type systems operate on a single abstract type grammar. C's type system adds a complication absent from the theory: *qualifiers* — `const`, `volatile`, `restrict`, `_Atomic` — are orthogonal to the structural type they modify. Clang's `QualType` [clang/include/clang/AST/Type.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Type.h) is a pointer-sized pair `(Type*, Qualifiers)` that packs qualifiers into the low bits of the pointer, exploiting the fact that `Type` objects are aligned to at least 8 bytes. This makes `QualType` a value type that is cheap to copy while carrying full qualifier information. The full `Type*` hierarchy — all sugar and canonical subclasses — is detailed in Chapter 36 — The Clang AST in Depth; here we focus on the type-theoretic principles governing canonical types and their role in template instantiation.

The `Type*` itself is the structural type — the part that corresponds to the theoretical grammar. The `Qualifiers` component adds `const` (bit 0), `volatile` (bit 1), `restrict` (bit 2), and `_Atomic` (bit 3) as flags. So `const int*` in C is represented as a `QualType` pointing to a `PointerType` whose pointee `QualType` points to `BuiltinType::Int` with `const` set.

### Canonical Types and the Sugar Taxonomy

Chapter 13's Algorithm W requires a unique canonical representative for each type (the principality of type schemes relies on structural uniqueness). Clang distinguishes between **sugar** types (source-level elaborations that preserve the programmer's phrasing) and **canonical** types (the unique structural representative). The `getCanonicalType()` method strips all sugar.

Sugar types include:

- **`TypedefType`**: `typedef int MyInt;` — `MyInt` is sugar for `int`. When Clang compares `MyInt` with `int` for type equivalence, it canonicalizes both to `BuiltinType::Int`.
- **`ElaboratedType`**: `struct Point p;` — the `struct` keyword creates an `ElaboratedType` wrapping the underlying `RecordType`. Useful for error messages ("expected `struct Point`, got `int`"), stripped for equivalence.
- **`ParenType`**: `int (*fp)(int)` — the parentheses around `*fp` are recorded by a `ParenType` node so that pretty-printing reconstructs the source syntax.
- **`AttributedType`**: `__attribute__((address_space(1))) int*` — carries ABI and extension attributes.
- **`MacroQualifiedType`**: preserves which macro expansion introduced a qualifier.

The distinction is not cosmetic. Type equivalence is always decided on canonical types — two `QualType` values are equivalent if and only if their canonical types are equivalent. Sugar is preserved only for diagnostic output and AST-level tools (clangd, clang-tidy, Chapter 47). Once Clang descends into code generation (Chapter 39), it works exclusively on canonical types.

```cpp
typedef int MyInt;
typedef const MyInt* ConstMyIntPtr;

// ConstMyIntPtr and const int* are canonically identical:
//   getCanonicalType(ConstMyIntPtr) == getCanonicalType(const int*)
//   == QualType(BuiltinType::Int, {const}) via PointerType

void test(ConstMyIntPtr p, const int* q) {
    // Clang checks: canonical(ConstMyIntPtr) == canonical(const int*)
    // Sugar differs; canonical types match.
}
```

At the LLVM IR level, all sugar is erased. `MyInt` and `int` produce identical LLVM IR — `i32` — and `ConstMyIntPtr` and `const int*` both produce `ptr` (LLVM 22 opaque pointer). The canonical type machinery lives entirely within Clang's Sema layer.

### ASTContext as the Type Factory

Clang uses `ASTContext` as the single point of allocation for all type objects, ensuring uniqueness. `ASTContext::getPointerType(QualType T)` returns the unique `PointerType` for pointee `T`; a global `FoldingSet<PointerType>` ensures that multiple calls with the same `T` return the same `PointerType*`. This is the same structural uniquing mechanism that MLIR's `StorageUniquer` provides — canonical structural types are interned.

Key factory methods:
- `getPointerType(QualType T)` — `T*`
- `getConstantArrayType(QualType T, const APInt& N, ...)` — `T[N]`
- `getFunctionType(QualType Result, ArrayRef<QualType> Args, ...)` — function type
- `getRecordType(const RecordDecl* D)` — struct/union/class type

The `RecordType` is nominal: it holds a pointer to the `RecordDecl`, and two structs with identical fields but different `RecordDecl`s are different types. This matches the Chapter 14 §3 nominal discipline exactly.

### Built-in Types and Target Sensitivity

`int`, `long`, `size_t`, and `ptrdiff_t` have target-dependent widths. Clang models this through `BuiltinType` subtypes (`BuiltinType::Int`, `BuiltinType::Long`, etc.) combined with `TargetInfo::IntWidth`, `TargetInfo::LongWidth`, and friends. On a 64-bit Linux x86-64 target, `long` is 64 bits; on Windows x86-64, `long` is 32 bits. Both map to `BuiltinType::Long`, but the `TargetInfo` instance fills in the width when Clang needs to know the bit count for code generation.

This target sensitivity is resolved in Clang's Sema and CodeGen layers; by the time LLVM IR is produced, `long` is already either `i32` or `i64`, and the LLVM type system sees no ambiguity.

### Dependent Types in Templates

Chapter 14 §1 introduced dependent types as types that can reference values from the program context. C++ template dependent types are not quite this — they are *parameterized* types that can reference template *type parameters* rather than runtime values — but they share the property of not being fully resolved until instantiation.

Clang represents pre-instantiation template types with a family of `Type` subclasses:

- **`DependentNameType`**: `typename T::value_type` — a name whose resolution depends on `T`.
- **`TemplateSpecializationType`**: `std::vector<T>` before `T` is known.
- **`SubstTemplateTypeParmType`**: after instantiation, records that `T` was substituted with `int` (preserving the substitution history for diagnostics).
- **`PackExpansionType`**: `T...` in a variadic template.

Two-phase lookup (Chapter 34) requires Clang to delay resolution of dependent names to instantiation time. The `DependentNameType` node is a placeholder in the AST that survives until Sema resolves it during instantiation. This is analogous to the substitution of a dependent type: `Π(n:Nat). Vec A n` is a type family; C++'s `std::vector<T>` is a type family parameterized by a type rather than a value, but the structural principle — the type cannot be fully evaluated until the parameter is supplied — is the same.

### The Canonical Type Check for Template Specialization

Clang's template instantiation produces `SubstTemplateTypeParmType` nodes that record the substitution history. When two template instantiations are compared for equivalence — for example, to determine whether `Container<int>` in one translation unit matches `Container<int>` in another — Clang strips to canonical types. `SubstTemplateTypeParmType` is sugar; its canonical type is the substituted argument's canonical type. Two instantiations with the same canonical argument types produce the same canonical type, enabling cross-TU merging of instantiations during LTO.

The practical implication: when Clang lowers `Container<int>` and `Container<int>` from two different TUs in a link, the linker uses the mangled names (which encode canonical types) to identify duplicate definitions. Clang's name mangling (Itanium C++ ABI, Chapter 42) is based entirely on canonical types, not sugar types. This is why `typedef int MyInt; Container<MyInt>` has the same mangled name as `Container<int>` — the typedef sugar is stripped during canonicalization before mangling.

### Type Errors in Template Instantiation

Chapter 13's principal-type property says that a well-typed term has a unique most-general type, and type errors are detected at the point of the type annotation. C++ templates violate the second part: template code is type-checked only at instantiation time, not at definition time. A function template with a bug:

```cpp
template<typename T>
T broken(T x) { return x.nonexistent_method(); }
```

passes Clang's first parsing phase without error. The error appears only when `broken<SomeConcreteType>` is instantiated with a type that lacks `nonexistent_method`. This is a deliberate consequence of C++'s two-phase lookup design and a well-known source of cryptic error messages.

C++20 **Concepts** (Chapter 34) restore the early error detection that STLC's type system provides: a concept-constrained template:

```cpp
template<typename T>
  requires std::ranges::range<T>
auto sum(T&& r) { return std::accumulate(begin(r), end(r), 0); }
```

is checked against the concept at definition time. If the template body uses an operation not guaranteed by the concept, Clang reports an error at the definition — analogous to a type annotation in STLC that catches mismatches before the term is applied to a specific value. Concepts are thus C++'s step toward the principality guarantee that HM inference provides in ML: the type constraint is expressed and checked at the point of the generic function definition.

---

## 5. Rust's Borrow Checker as a Substructural Type System

### MIR: The IR Where Borrowing Lives

Chapter 14 §4 developed affine types: types where each value may be used *at most once*. Rust's ownership model is an affine type system (with `Copy` types as the structural exception). The borrow checker that enforces this model operates not on the surface syntax or HIR, but on **MIR** (Mid-level Intermediate Representation), a CFG-based IR interposed between the type-checked HIR and LLVM IR.

MIR makes ownership and lifetimes explicit in a form amenable to dataflow analysis. Every MIR statement is one of:

- **Assignment**: `_1 = _2` — a *move* from place `_2` to place `_1`; after this, `_2` is *uninitialized*
- **Borrow**: `_1 = &'a _2` — creates an immutable borrow; `_2` may not be moved while `'a` is live
- **Mutable borrow**: `_1 = &'a mut _2` — creates an exclusive borrow; no other access to `_2` during `'a`
- **Drop**: `drop _1` — explicit destruction of the value at place `_1`

The MIR CFG makes control flow explicit, enabling non-lexical lifetimes (NLL): a borrow's liveness region is determined by the dataflow analysis over the CFG, not by the syntactic scope of the block.

### Constraint Generation and Solving

The NLL borrow checker (RFC 2094) operates in two phases:

1. **Constraint generation**: traverse the MIR and emit lifetime constraints of the form `'a: 'b` ("lifetime `'a` must include all points in lifetime `'b`") and `{point} ⊆ 'a` ("lifetime `'a` must include this program point"). Every borrow, every use of a borrowed reference, and every function call boundary generates constraints.

2. **Constraint solving**: compute the minimal set of program points that satisfies all constraints for each lifetime variable. This is a least-fixed-point computation over the CFG — precisely the kind of dataflow analysis described in Chapter 10. The experimental **Polonius** solver ([https://github.com/rust-lang/polonius](https://github.com/rust-lang/polonius)) reformulates this as a Datalog program, enabling more precise (but more expensive) analysis that can accept some programs the legacy solver rejects.

The result is a set of lifetime regions: for each borrow `&'a T` in the program, the lifetime `'a` is the set of program points between the borrow creation and its last use. If any two borrows' regions overlap in a way that violates the **xor-of-references invariant** — at most one mutable borrow *or* any number of immutable borrows active at the same point — the borrow checker emits an error.

### The Affine Type Discipline in Practice

The xor-of-references invariant is the operational manifestation of Chapter 14 §4's affine type rule. Recall:

- `T` (non-`Copy`): affine — used at most once (moved)
- `&T`: shared — can be duplicated freely; gives read access
- `&mut T`: linear-like — there is exactly one live `&mut T` at any point; gives exclusive write access

A Rust program that compiles is a proof that these constraints are satisfied throughout every execution path. Consider a simplified MIR fragment:

```
// Rust source:
//   let mut v = vec![1, 2, 3];
//   let r = &v[0];
//   v.push(4);  // ERROR: mutable borrow of v while r is live
//   println!("{}", r);

// Illustrative MIR (not exact rustc output):
bb0:
  _1 = Vec::new();               // _1 = v
  _2 = &_1[0];                   // _2 = r = &'a v[0]
  // 'a must cover bb0[2] through bb0[4]
  _3 = &mut _1;                  // mutable borrow of _1 while 'a is live
  // CONFLICT: &_1 (shared, 'a live) and &mut _1 (exclusive) overlap
  Vec::push(move _3, 4i32);
  _4 = copy _2;
  drop _1;
```

The constraint solver determines that `'a` covers the point where `v.push` occurs; at that point, an immutable borrow of `_1` (through `_2`) and a mutable borrow of `_1` (for `Vec::push`) coexist. The overlap violates the xor invariant, and the borrow checker reports an error — precisely the error the programmer expects.

### Two-Phase Borrows and the Borrow Checker's Edge Cases

The NLL borrow checker introduced **two-phase borrows** to handle patterns that were unnecessarily rejected by the lexical borrow checker. The canonical example is calling a method that mutably borrows `self` while passing an expression that immutably borrows `self` as an argument:

```rust
// Two-phase borrow: allowed in NLL
let mut v = vec![1, 2, 3];
v.push(v.len());
// Desugars to:
//   tmp = v.len();   // immutable borrow of v — "reservation" phase
//   v.push(tmp);     // mutable borrow of v — "activation" phase
// The two borrows are sequenced, not concurrent
```

Under two-phase borrow rules, an `&mut` borrow is first *reserved* (a placeholder) at the point of the borrow expression, then *activated* when the mutation actually occurs. Between reservation and activation, a concurrent immutable borrow is permitted. This is a relaxation of the strict xor-of-references invariant, justified by the observation that the two borrows are temporally ordered by the call sequencing.

The Polonius analysis reformulates the borrow checker as a Datalog program over facts derived from the MIR. Its key insight is that the legacy NLL algorithm is overly conservative because it reasons about *regions* (sets of program points) rather than *loan propagation* (tracking which loan caused which access to be invalid). Polonius can accept programs with non-lexical borrows that NLL rejects, at the cost of higher compile-time overhead. Polonius's formulation as Datalog makes it amenable to formal verification — the rules can be checked against a reference semantics and proved sound.

### Rust's Type-Level Guarantees and Their LLVM IR Manifestation

The RustBelt project [Jung et al., JACM 2021] provides a machine-verified semantic proof of Rust's type system: for any safe Rust program that the type checker accepts, the compiled code satisfies a semantic safety property defined in the Iris program logic. The proof handles the full type system including lifetimes, trait objects, and `unsafe` blocks — the latter are proved correct by asserting their preconditions as obligations on the programmer.

Concretely, RustBelt proves:
- **Memory safety**: no use-after-free, no double-free, no dangling pointer dereferences
- **Type safety**: no type confusion — an `i32` value is never used as a pointer
- **Absence of data races**: no concurrent mutable and immutable accesses to the same location (for the safe fragment)

These guarantees are exactly what the LLVM optimizer assumes when compiling Rust code. The `noalias` attribute that Rust attaches to exclusive references (`&mut T`) in LLVM IR is the encoding of the xor-of-references invariant at the LLVM level:

```llvm
; Rust &mut T becomes a noalias pointer in LLVM IR
define void @update(ptr noalias noundef align 4 %x, ptr noalias noundef align 4 %y) {
  %v = load i32, ptr %y, align 4
  store i32 %v, ptr %x, align 4
  ret void
}
```

The `noalias` attribute tells LLVM that `%x` and `%y` do not alias. LLVM's alias analysis exploits this to prove that the load from `%y` is independent of the store to `%x`, enabling load-store reordering and redundant load elimination. The compile-time borrow checking proof becomes a runtime optimization permission.

### The Transition to LLVM IR

Crucially, the borrow checker enforces these invariants *at compile time* and then discards the evidence. The LLVM IR that Rust generates contains no lifetime annotations, no borrow markers, no ownership tags. It is plain loads and stores, indistinguishable from what a C compiler would produce. The type-level proof that no undefined behavior occurs due to aliasing — the RustBelt theorem [Jung et al., JACM 2021] — guarantees that the plain LLVM IR is safe: no data races, no use-after-free, no double-free for well-typed safe Rust.

This is the affine type system's payoff: heavy compile-time verification that amortizes to zero runtime cost. The LLVM IR optimizer can freely apply optimizations that assume no aliasing between a mutable reference and any other reference to the same allocation — because the borrow checker proved that no such aliasing exists.

---

## 6. ML Compilers: From HM to LLVM via Typed Intermediates

### GHC's Typed Pipeline: System FC and STG

The journey of Haskell source code to LLVM IR is the most elaborate illustration of typed intermediate representations in production use. GHC maintains strong types through most of the pipeline, erasing them only at the Cmm (C-like intermediate) stage.

**GHC Core / System FC** [Sulzmann et al., TLDI 2007] is GHC's principal typed intermediate language. It is a variant of System F extended with *coercion proofs* — explicit proof terms that witness type equality. Where System F (Chapter 13 §2) has type abstractions `Λα. t` and type applications `t[τ]`, System FC adds coercion abstraction and application. A newtype in Haskell:

```haskell
newtype Age = MkAge Int
```

generates a coercion `co : Age ~R Int` (a *representational equality* proof). The function `coerce :: Age -> Int` is compiled as a Core term `\(x : Age) -> x |> co` — the `|> co` applies the coercion, reinterpreting the `Age` value as `Int` with zero runtime cost. The coercion is a compile-time proof; it is erased before code generation.

System FC is key to GHC's type-safe compilation of GADTs (Chapter 14 §1), type families, and `GeneralizedNewTypeDeriving`. Every program transformation in the Core-to-Core optimizer (inlining, case-of-case, worker-wrapper, etc.) must preserve Core's typing invariants. A transformation that violates these invariants would be caught by Core Lint — GHC's type-checker for Core terms.

**STG (Spineless Tagless G-machine)** [Peyton Jones 1992] is the next stage. Types are largely erased here; STG is concerned with the evaluation model (lazy thunks, closures, constructors) rather than the type structure. STG's `StgExpr` and `StgArg` carry only minimal type information needed for code generation.

**Cmm** (C-minus-minus) is GHC's low-level IR: essentially a typed assembly with explicit stack manipulation. Types at the Cmm level are the machine-word types `B8`, `B16`, `B32`, `B64`, `F32`, `F64` and pointer `Ptr`. Polymorphism has been fully erased; the GC root map for each stack frame is computed from type information before this point.

**LLVM IR** is then generated from Cmm by GHC's LLVM backend. At this stage, GHC emits standard LLVM IR with `i64`, `f64`, and `ptr` types. The Haskell type system's rich constraints — type class instances, higher-kinded types, GADTs — are all gone. What remains are function pointers, closure records, and a small runtime ABI.

### A System FC Snippet: Seeing the Coercions

A concrete illustration of System FC coercions clarifies the theory. Haskell's `newtype` optimization is a zero-cost abstraction enforced by the type system: `Age` and `Int` have identical runtime representations, but the type checker treats them as distinct. System FC makes this precise through coercion terms.

A simplified System FC representation for a function that converts an `Age` list to an `Int` list via coercions:

```
-- Haskell source:
-- newtype Age = MkAge { getAge :: Int }
-- coerce :: Coercible a b => a -> b

-- GHC Core (System FC), simplified:
-- Co_Age : Age ~R Int   (representational equality coercion)
-- Co_List_Age : [Age] ~R [Int]  (lifted coercion for lists)

convertAges :: [Age] -> [Int]
convertAges xs = xs `cast` Co_List_Age

-- After optimization: no-op at the machine level
-- Generated LLVM IR: the function is compiled away entirely
-- (identity function on the pointer-sized list representation)
```

The `cast` term applies the coercion proof `Co_List_Age` to reinterpret the `[Age]` list as `[Int]`. Since `Age` and `Int` have the same representation, the cast generates no machine instructions. GHC Lint verifies that the coercion proof is well-formed: it checks that the coercion derivation is valid according to the System FC rules for representational equality. If the programmer writes `coerce` where no valid coercion exists, Lint rejects the Core term.

### MLton: Standard ML to Flat IR

MLton ([http://mlton.org](http://mlton.org)) compiles Standard ML with full monomorphization and whole-program optimization. The pipeline is:

1. **Elaboration and Defunctorization**: SML functors (analogous to parameterized modules) are defunctorized — instantiated at each use site — producing a monomorphic program.
2. **Type-directed representation analysis**: the compiler chooses flat representations for datatypes based on their structure. Unit becomes 0, booleans become 0/1, small enumerations become tagged integers. This is a semantic simplification guided by type information.
3. **A-normal form and closure conversion**: the program is put in A-normal form (every subexpression is named), and lambdas are closure-converted to heap-allocated records.
4. **SSA IR**: MLton's SSA-based IR is a flat, functional CFG similar to LLVM IR. Every function has a fixed signature of ground types (no polymorphism).
5. **LLVM IR generation**: the SSA IR is lowered to LLVM IR, where further backend optimization occurs.

MLton's SSA IR is particularly close in spirit to LLVM IR: both are typed SSA forms with explicit `phi` nodes, both operate on ground types, and both permit a rich set of dataflow analyses. The key difference is that MLton's SSA preserves ML-style algebraic-datatype tags in the IR, enabling ML-specific optimizations like constructor fusion, while LLVM IR works at the level of integers and pointers.

### OCaml: Lambda IR and Flambda2

OCaml's compilation pipeline goes through **Lambda IR** (a largely untyped lambda-calculus-like IR) and, with the `flambda2` optimizer, through **Flambda2** (a CPS-based IR with explicit closures). OCaml erases most type information at the Lambda IR stage:

- **Uniform representation**: every OCaml value is represented as an `int` or a pointer. Integers are tagged (low bit set); pointers to heap blocks are untagged. Floats in arrays may be unboxed (as an optimization), but otherwise every value is a word.
- **GC root tracking**: because every word-sized value might be a heap pointer, the OCaml garbage collector must scan the entire stack. Flambda2 tracks GC roots explicitly to enable precise collection.
- **Unboxing**: Flambda2's inliner can unbox `float` values and `int`-valued records across call boundaries, reducing allocations. This optimization is type-directed at the Flambda2 level but produces untyped integer operations at the LLVM IR level.

The OCaml LLVM backend (available as a non-default option) emits LLVM IR with `i63`-tagged integers (the low bit being the tag bit), `ptr` for heap values, and `f64` for unboxed floats. The HM type system that governs the OCaml source has been completely erased by this point.

### The Type-Erasure Gradient Across ML Compilers

Comparing the three ML-family compilers side by side reveals a systematic type-erasure gradient:

| Stage | GHC (Haskell) | MLton (SML) | OCaml |
|---|---|---|---|
| Source | Full HM + type classes + GADTs | HM + modules + functors | HM + objects + modules |
| High-level IR | GHC Core (System FC, typed) | SXML (typed) | Lambda IR (mostly untyped) |
| Mid-level IR | STG (partially typed) | RSSA (typed) | Flambda2 (typed for opts) |
| Low-level IR | Cmm (machine-word typed) | Machine (word-typed) | Cmm-like (word-typed) |
| LLVM IR | `i64`/`f64`/`ptr` | `i64`/`f64`/`ptr` | `i63`/`f64`/`ptr` (tagged) |

In each case, the type information is fully exploited at the source and high-level IR stages (type checking, optimization decisions, representation analysis) and progressively erased as the compiler descends toward machine code. LLVM IR is the final state of complete erasure: only flat scalar types and pointers remain.

The practical lesson for compiler engineers is that types are a *resource*: they carry information useful for analysis and optimization, and that information has a cost to maintain. The optimal erasure point varies by language and optimization goal. GHC maintains types the longest (through Core Lint), enabling type-directed Core-to-Core optimizations. MLton erases earlier in exchange for more aggressive whole-program analysis. OCaml's Lambda IR erases early, relying on profile-guided and inlining-based optimization rather than type-directed specialization.

---

## 7. Alive2 and Refinement-Style Verification on LLVM IR

### The Problem Alive2 Solves

LLVM contains hundreds of peephole optimizations: instruction combining rules that rewrite patterns like `(x & 0xFF) | (y & 0xFF)` into `(x | y) & 0xFF`. Each such rule must preserve the *semantics* of the original code. A bug in an optimization rule — incorrect handling of poison, wrong signedness assumption, off-by-one in a shift — can silently corrupt the programs that LLVM compiles. Type safety in LLVM IR does not prevent this: the optimized code can be well-typed and semantically wrong simultaneously.

Alive2 [Lopes, Lee, Hur et al., PLDI 2021] (Chapter 170) addresses this by translating pairs of LLVM IR snippets (before and after optimization) into SMT formulas and checking that the *after* version is a **refinement** of the *before* version. Refinement here means: for every input on which the *before* code is defined (not poison, not undefined behavior), the *after* code produces the same result.

### Refinement as Subtyping

The connection to Chapter 14's refinement types (§7) is precise. Recall that a refinement type `{x:T | P(x)}` is a subtype of `T`: any value satisfying `P(x)` is also a valid `T`. Alive2's refinement relation on LLVM IR programs is analogous: the optimized program is a *more defined* (or equally defined) version of the original, with no additional undefined behavior and the same outputs on all defined inputs. Formally, for single-value outputs:

```
before ⊑ after  iff
  ∀ input values:
    UB(before) → UB(after)               [optimizer may introduce UB only where before already had it]
    ¬UB(before) → output(after) = output(before)  [same output on defined inputs]
    Poison(before) → Poison(after) ∨ output(after) = any  [poison propagation]
```

This refinement relation is a preorder on LLVM IR programs. An optimization is *correct* iff it produces a program that is a refinement of the original. Alive2 mechanically verifies this using Z3 or another SMT solver, encoding LLVM IR semantics (including the precise poison/freeze semantics from Chapter 171) into the solver's logic.

### Poison and Undef in the Type Lattice

Alive2's treatment of `poison` and `undef` illuminates their type-theoretic role. In the LLVM IR semantic model:

- `undef` of type `i32` is not a specific value — it is a *free variable* that the optimizer may instantiate to any `i32` value independently at each use. This makes `undef` semantically nondeterministic: `(undef + undef) ≠ 2 * undef` in general.
- `poison` is a stronger condition: any computation that reads a `poison` value produces `poison`, and any side-effecting operation (store, branch, call) that observes `poison` invokes undefined behavior. Unlike `undef`, `poison` is *strict*: it taints all downstream results.

In a lattice picture, `poison` is below the concrete values and `undef` is above them (since `undef` can represent any concrete value). This matches a type lattice with `poison` at the bottom (unusable, triggers UB) and a top element representing nondeterminism. Alive2 encodes this lattice explicitly in its SMT formulas, ensuring that optimizations respect the monotonicity of the refinement order.

### Alive2 in Practice: A Caught Optimization Bug

To make the refinement mechanism concrete, consider an optimization that incorrectly transforms a division:

```llvm
; BEFORE: signed division by power-of-two
define i32 @sdiv_by_4(i32 %x) {
  %r = sdiv i32 %x, 4
  ret i32 %r
}

; INCORRECT AFTER: arithmetic right shift
; (sdiv rounds toward zero; ashr rounds toward -infinity)
define i32 @sdiv_by_4_wrong(i32 %x) {
  %r = ashr i32 %x, 2
  ret i32 %r
}
```

For `x = -1`: `sdiv -1, 4 = 0` (rounds toward zero), but `ashr -1, 2 = -1` (arithmetic shift fills with sign bit). The results differ. Alive2 encodes both functions as SMT formulas, queries Z3 for a witness where the outputs differ, and returns `x = -1` as a counterexample. The `before ⊑ after` relation is violated: `after` produces `-1` where `before` is defined and produces `0`.

The correct optimization is `ashr i32 %x, 2` only when `%x ≥ 0` (which can be established by preceding range analysis), or alternatively `(x + 3) >> 2` for negative-rounding semantics. Alive2 has caught dozens of such bugs in LLVM's InstCombine pass, several of which had been in production for years.

### Freeze and the Elimination of Undef

The `freeze` instruction (introduced in LLVM 10) addresses a specific weakness in the `undef` semantic model. When the compiler introduces `undef` through optimizations (e.g., replacing an uninitialized load with `undef`), downstream uses can observe different "values" at each read — an incoherent behavior that complicates both semantics and optimization. The `freeze` instruction fixes a single arbitrary value:

```llvm
%v = freeze i32 undef   ; %v is now some fixed but unspecified i32
                        ; unlike undef, %v is the same on every read
```

Alive2's semantic model handles `freeze` by treating it as a nondeterministic choice that is made once and held fixed. This models the intended semantics: the program is as if the compiler picked an arbitrary value at the `freeze` point. Any optimization that moves or removes a `freeze` must be verified to preserve this semantic, and Alive2 can check it.

The migration from `undef`-based to `poison`-and-`freeze`-based semantics in LLVM 10–15 is a concrete example of a type-theoretic refinement: the original `undef` model was too permissive (any value per read → incoherent), and the replacement with `poison` (strict: triggers UB when observed) plus `freeze` (explicit nondeterminism anchoring) is a more precise specification of the intended semantics.

### What "Type Safety" Means at LLVM IR Level

The practical upshot is that LLVM IR type safety and semantic correctness are orthogonal:

- A well-typed LLVM IR program (passes the IR verifier) can still have undefined behavior due to `poison` propagation, integer overflow with `nsw`/`nuw` flags violated, null pointer dereference, or integer-to-pointer conversions that alias analysis cannot resolve.
- Alive2's refinement check catches semantic bugs that the IR verifier misses entirely.

LLVM IR's type system is better understood as a *liveness annotation* — it ensures that values flowing into instructions have the right width and category (integer, float, pointer) — rather than a *safety proof*. The deeper safety arguments live at the source-language level (Rust's borrow checker, C++'s Sema) or in verified compilation frameworks (CompCert, Vellvm, Alive2).

---

## 8. constexpr/consteval as Total Functional Programming

### The constexpr Sublanguage

Chapter 14 §2 noted that termination (strong normalization) is a property of certain type systems: all well-typed terms reduce to values. C++'s `constexpr` mechanism carves out a *terminating sublanguage* of C++ that can be evaluated at compile time. This is not a formally proven property — C++ `constexpr` functions can be recursively defined in ways that might not terminate — but the Clang constant evaluator (Chapter 35) enforces a step limit and treats non-termination as a compilation error for `consteval` functions.

A `constexpr` function is one that:
1. Has a non-`void` return type (relaxed in C++14+)
2. Does not invoke undefined behavior during compile-time evaluation
3. Does not call non-`constexpr` functions (with exceptions for `std::is_constant_evaluated()`)

```cpp
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
constexpr int fac5 = factorial(5);   // computed at compile time: 120
```

The Clang-generated LLVM IR for `get_fac()` — a function that returns `fac5` — shows that the entire computation has been folded at compile time:

```llvm
define dso_local noundef i32 @_Z7get_facv() local_unnamed_addr {
  ret i32 120
}
```

The constant `120` (= 5!) is embedded directly in the IR. No call to `factorial` appears; the type-level argument that `factorial` is a total function on small `int` inputs was exploited by the Clang constant evaluator to fold the result before LLVM sees it.

### The Evolution: C++11 → C++23

The `constexpr` sublanguage has expanded in each C++ standard revision, moving closer to a general-purpose total functional programming language:

| Standard | Key expansion |
|---|---|
| C++11 | Single return statement, recursion, conditional expressions |
| C++14 | Local variables, loops, `if`/`switch`, multiple statements |
| C++17 | `if constexpr`, fold expressions, constexpr lambdas |
| C++20 | `try`/`catch` in constexpr, `dynamic_cast`, `virtual` functions, heap allocation (`new`/`delete`) |
| C++23 | `static` local variables, `std::is_constant_evaluated` in more contexts, constexpr `std::bitset` |

C++20's addition of heap allocation in `constexpr` context is particularly significant: it means that arbitrary data structures (trees, graphs, hash maps) can be constructed and manipulated at compile time, as long as all heap memory is released before the constant expression is complete. This approaches the expressiveness of a total functional programming language with a garbage-collected heap.

### consteval: Mandatory Compile-Time Evaluation

A `consteval` function is an **immediate function**: it *must* be evaluated at compile time. Any call that cannot be evaluated at compile time is a hard compilation error. This is a stronger guarantee than `constexpr`: `constexpr` functions may be evaluated at runtime when called with non-constant arguments, but `consteval` functions never appear in LLVM IR at all — they are purely a compile-time computation device.

```cpp
consteval int square(int n) { return n * n; }
constexpr int sq9 = square(9);    // ok: 81 at compile time
// int runtime_sq = square(x);    // ERROR: x is not a constant expression
```

The `consteval` qualifier makes the compile-time computation intent explicit and prevents the function from being accidentally called at runtime. In the Curry-Howard reading of Chapter 12, a `consteval` function is a proof term that lives in the compile-time world and cannot be projected into the runtime world.

### Template Metaprogramming as Accidental Turing Completeness

Before `constexpr`, C++ programmers encoded compile-time computation in template metaprograms:

```cpp
template<int N>
struct Fib {
    static constexpr int value = Fib<N-1>::value + Fib<N-2>::value;
};
template<> struct Fib<0> { static constexpr int value = 0; };
template<> struct Fib<1> { static constexpr int value = 1; };

int fib10 = Fib<10>::value;  // computed by template instantiation: 55
```

The LLVM IR for `fib10` is:

```llvm
@fib10 = dso_local local_unnamed_addr global i32 55, align 4
```

Again, the computation is entirely erased. But the template metaprogram is notoriously hard to write, error-prone, and produces catastrophic compiler diagnostics. It is Turing-complete [Veldhuizen 2003] — the template instantiation mechanism can simulate a Turing machine — but it was never designed as a programming language. `constexpr` and `consteval` replace it with an intentional, readable, debuggable compile-time computation facility.

### constinit and the Initialization Fiasco

`constinit` (C++20) is a related but distinct keyword: it asserts that a variable with static or thread-local storage duration is initialized by a constant expression (or constinit initializer), preventing the "static initialization order fiasco" — the undefined order in which static variables in different translation units are initialized. `constinit` does not prevent runtime mutation (that requires `constexpr` or `const`), but it guarantees that the initial value is computed at compile time and embedded in the binary's `.rodata` or `.data` section.

The connection to type theory is indirect but real: `constinit` enforces that the initializer belongs to the `constexpr` sublanguage (the total, terminating fragment), which is the compile-time world of the Curry-Howard correspondence. If the initializer contains UB or non-termination, the compiler rejects it at compile time rather than producing a binary with undefined static initialization behavior.

### Immediate-Mode Constant Evaluation and the Clang Interpreter

The Clang constant evaluator (Chapter 35) implements a full interpreter for the `constexpr` sublanguage within the compiler. It traverses the Clang AST and evaluates:

- Arithmetic and logical operations on integers, floats, and booleans
- Conditionals (`if`, ternary)
- Function calls to `constexpr` functions (recursive, with a call-depth limit)
- Local variable initialization and mutation
- C++20: heap allocation and deallocation, virtual dispatch, `try`/`catch`
- Aggregate construction and member access

The interpreter is implemented in [`clang/lib/AST/ExprConstant.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/AST/ExprConstant.cpp) — one of the largest files in the Clang codebase (~17,000 lines). Its correctness is critical: an incorrect evaluation of a `consteval` function produces a wrong compile-time constant that is embedded in the binary, with no runtime check to catch the error.

The interpreter architecture mirrors a small-step operational semantics (Chapter 12 §2): it maintains an evaluation context (a map from declared variables to values), reduces expressions step by step, and produces a `APValue` (Arbitrary Precision Value) result. This is precisely the substitution model of beta-reduction applied to a subset of C++ expressions.

For `consteval` functions, if the interpreter reaches an operation that cannot be evaluated at compile time — a virtual call through an incomplete type, a non-constant system call, an `asm` statement — it emits a diagnostic and the compilation fails. This is the "totality check" for the `constexpr` sublanguage: undefined behavior is detected at compile time (because the interpreter models the semantics precisely), and non-termination is prevented by the recursion limit.

### Constant Propagation to LLVM IR

The interaction between Clang's constant evaluator and LLVM IR code generation is straightforward: any expression that Clang can evaluate to a constant at compile time is emitted as an LLVM IR constant rather than a sequence of instructions. This includes:

- `constexpr` global variables → LLVM IR `@var = constant T <value>`
- Template non-type parameters used in array sizes → array type `[N x T]` with N folded
- `constexpr` local variables in non-constexpr functions → LLVM `i32 constant` folded by LLVM's constant propagation (which redoes some of what Clang's evaluator did)
- Computed array indices → `getelementptr inbounds` with a constant offset

The recursive `factorial(5)` from earlier produces `ret i32 120` directly — the call is gone. No LLVM optimization is required; Clang folds it before emitting IR. LLVM's own constant folding (`ConstantFolding.cpp`) then handles expressions that appear constant after inlining — a second pass of the same principle at a lower level.

### The Dependent Type Connection

`constexpr` values can appear as non-type template parameters:

```cpp
template<std::size_t N>
struct FixedArray { int data[N]; };

constexpr std::size_t kSize = 64;
FixedArray<kSize> arr;      // arr has type FixedArray<64>
```

Here `kSize` is a value that appears in a type — exactly the dependent type idiom of Chapter 14 §1. C++ is not a dependent type system (type checking does not require arbitrary computation at check time), but non-type template parameters are a restricted form of value-indexed types. The type `FixedArray<64>` is distinct from `FixedArray<128>` in exactly the way `Vec A 64` is distinct from `Vec A 128` in Martin-Löf Type Theory — the index is part of the type's identity, and the type checker enforces consistency.

`constexpr if` (C++17) extends this: code branches guarded by a `constexpr bool` condition are *type-checked in their respective branches only*, mirroring dependent pattern matching in Agda or Idris. The eliminated branch is not just dead code — it is not type-checked at all in the context of the instantiation. This enables type-safe multi-way dispatch that would require dependent types to express cleanly in a theoretical setting.

---

## 9. Chapter Summary

This chapter grounded the type theory of Chapters 12–14 in the concrete implementations of LLVM, Clang, MLIR, Rust, and the ML compiler family. The discussion covered the full arc from theoretical constructs — dependent Π types, System F quantification, Hindley-Milner inference, linear and affine types, refinement types — to their engineering manifestations in production compiler infrastructure. The key correspondences are:

- **LLVM IR is a ground instantiation of STLC without polymorphism.** Its type grammar is flat and first-order (Chapter 12's base types and function types, nothing more). Named struct types are nominal; literal struct types are structural. The opaque pointer migration (LLVM 16–17) removed the last vestige of pointee-type tracking from the type system, replacing it with alias analysis metadata.

- **Generics monomorphize before reaching LLVM.** The `∀α. τ` of System F (Chapter 13) never appears in LLVM IR. C++ templates, Rust generics, and GHC `SPECIALIZE` pragmas all produce distinct ground LLVM IR per type instantiation. Go's GC-shape stenciling and Haskell's dictionary passing are the two points in the design space that permit *some* sharing of compiled code, at the cost of a runtime type descriptor.

- **MLIR's type system is closer to the theory than LLVM IR.** Parameterized types (`tensor<4x4xf32>`) resemble dependent type constructors (Chapter 14 §1). Type interfaces provide ad-hoc polymorphism analogous to type classes (Chapter 13 §5). Storage uniquing makes type equality structurally determined, in contrast to LLVM IR's nominal named-struct identity.

- **Clang's QualType / canonical type split separates source-level sugar from type-theoretic identity.** The sugar types (TypedefType, ElaboratedType, ParenType) serve diagnostics and tooling; canonical types, uniqued in ASTContext, are what the type checker uses for equivalence. Dependent types in templates (DependentNameType, TemplateSpecializationType) are pre-instantiation placeholders resolved during Sema, analogous to open terms in dependent type theory.

- **Rust's borrow checker is an affine type system over MIR.** The linear/affine discipline of Chapter 14 §4 is enforced by NLL constraint generation and solving over the MIR CFG. The xor-of-references invariant (at most one mutable borrow or any number of immutable borrows at each program point) is the operational content of the affine typing rules. The borrow checker produces no runtime overhead; LLVM IR receives plain loads and stores.

- **ML compilers maintain types through multiple IRs before erasing them.** GHC Core (System FC) is a coercion-proof-carrying typed IR that preserves the Haskell type structure through the optimizer. MLton's SSA IR preserves ML algebraic-datatype structure. OCaml's Flambda2 uses type information for unboxing. All erase to untyped or weakly-typed LLVM IR at the final lowering stage.

- **Alive2 verifies LLVM optimizations via a refinement relation.** The refinement `before ⊑ after` is the type-theoretic notion of subtyping in a semantic type (the denotational model of LLVM IR programs). Poison/undef inhabit a type lattice with definite ordering; correct optimizations must preserve the refinement order. LLVM IR type safety and semantic correctness are orthogonal.

- **`constexpr`/`consteval` carve a total functional programming sublanguage from C++.** The Clang constant evaluator implements an interpreter for this sublanguage. Values computed at compile time disappear entirely from LLVM IR — they become constants in the binary. Non-type template parameters and `constexpr if` are restricted forms of dependent types (Chapter 14 §1), grounding that theory in C++ without requiring a full dependent type checker.

- **The theory-to-practice gap is a type-erasure gradient.** Full dependent types and refinement types live at the source level; affine types live through MIR; nominal and structural types survive into LLVM IR in erased form; at LLVM IR, only ground simple types remain. Each stage of the gradient is a principled decision about which invariants must be preserved for the downstream analysis and which can be safely erased once they have been verified.

- **C++20 Concepts and `constexpr` close the expressivity gap.** Concepts restore the early error-detection property of typed lambda calculi to C++ templates. `constexpr` and `consteval` carve a terminating sublanguage amenable to compile-time evaluation. Non-type template parameters and `constexpr if` are restricted forms of dependent typing. Together they bring C++ substantially closer to a language where type-theoretic guarantees are enforced systematically rather than incidentally.

- **Every engineering choice in this chapter is a trade-off studied in the theory.** Monomorphization vs. erasure is the System F instantiation trade-off. Structural vs. nominal types is the Chapter 14 §3 distinction. Fat pointers for `dyn Trait` are existential types with witnesses. GHC's System FC coercions are proof terms for representational equality. Rust's `noalias` is the affine type's aliasing guarantee expressed as LLVM IR metadata. The theory provides the vocabulary; the implementations make the engineering choices within that vocabulary.

---

*Next chapter: [Chapter 16 — IR Structure](../part-04-llvm-ir/ch16-ir-structure.md) begins the detailed study of LLVM IR that the preceding theory has prepared us for.*


---

@copyright jreuben11
