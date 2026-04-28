# Chapter 4 — The LLVM C++ API

*Part I — Foundations*

LLVM's public C++ API is not just a collection of classes that happen to compile and link. It is an opinionated system built around a handful of design choices that recur across every subsystem: a custom RTTI that is faster and more flexible than `dynamic_cast`; a thread-local context object that owns all type and metadata state; a rich library of performance-tuned containers; a checked, exception-free error model; and a declarative command-line option system that works without a `main`. Understanding these foundations pays compound interest throughout the rest of the book — every pass, every analysis, every dialect in MLIR is built from the same primitives described here.

This chapter is a systematic tour of those primitives with enough depth to be useful the first time you reach for one and enough rationale that the design choices feel inevitable rather than arbitrary.

---

## 4.1 LLVM's Custom RTTI: `isa`, `cast`, `dyn_cast`

C++ provides `dynamic_cast<T*>(p)` for runtime type queries, but it carries two costs that are unacceptable in a hot compiler inner loop: it requires RTTI to be compiled in (a non-trivial binary size penalty), and it is significantly slower than a carefully written `classof` check because the C++ runtime must traverse the entire class hierarchy stored in the typeinfo tables. LLVM's solution is a custom RTTI system, defined in [`llvm/include/llvm/Support/Casting.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/Casting.h), that replaces `dynamic_cast` with four function templates.

### 4.1.1 The Four Casting Operations

| Function | Behaviour | Precondition | Returns |
|---|---|---|---|
| `isa<T>(v)` | Type predicate | `v` must not be null | `bool` |
| `cast<T>(v)` | Checked downcast | `isa<T>(v)` must be true | `T&` or `T*` |
| `dyn_cast<T>(v)` | Optional downcast | `v` must not be null | `T*` or `nullptr` |
| `dyn_cast_or_null<T>(v)` | Optional downcast allowing null input | none | `T*` or `nullptr` |

A fifth operation, `isa_and_nonnull<T>(v)`, combines a null check with `isa<T>` and is equivalent to `v && isa<T>(v)`. It is useful when the argument comes from a container that may hold null sentinels.

```cpp
#include "llvm/IR/Instructions.h"
#include "llvm/Support/Casting.h"

void examineInstruction(llvm::Instruction *I) {
  // isa<>: pure type predicate, no cast performed
  if (llvm::isa<llvm::BranchInst>(I))
    llvm::errs() << "Unconditional or conditional branch\n";

  // dyn_cast<>: returns nullptr if the cast fails
  if (auto *BI = llvm::dyn_cast<llvm::BranchInst>(I)) {
    if (BI->isConditional())
      llvm::errs() << "Conditional branch on: " << *BI->getCondition() << "\n";
  }

  // cast<>: asserts if the cast would fail — use only when you know the type
  if (llvm::isa<llvm::ReturnInst>(I)) {
    auto &RI = llvm::cast<llvm::ReturnInst>(*I);
    if (llvm::Value *RV = RI.getReturnValue())
      llvm::errs() << "Returns: " << *RV << "\n";
  }

  // dyn_cast_or_null<>: safe when the pointer may itself be null
  llvm::Instruction *MaybeNull = nullptr;
  if (auto *CI = llvm::dyn_cast_or_null<llvm::CallInst>(MaybeNull))
    llvm::errs() << "Call: " << *CI << "\n"; // never reached
}
```

**Why not `dynamic_cast`?** The critical path in an optimiser iterates over tens of thousands of instructions per second. Each `dynamic_cast` may invoke `__dynamic_cast` from libstdc++/libc++abi, which must navigate the typeinfo table of the full polymorphic hierarchy — an O(depth) operation with poor cache behaviour. The `classof` dispatch is typically a single integer comparison or a range check, inlineable at the call site.

### 4.1.2 The `classof` Contract

For the casting machinery to work, every class that participates must define a static `classof` method that returns `true` if the argument is an instance of that class or any of its subclasses.

The `Instruction` hierarchy uses a compact opcode encoding: each instruction subclass stores an `unsigned` opcode in a field inherited from `Value`. Groups of related instructions reserve a contiguous range of opcode values and check membership with two comparisons. This is the range-check `classof` idiom:

```cpp
// From llvm/include/llvm/IR/Instructions.h (simplified):
class BinaryOperator : public Instruction {
public:
  static bool classof(const Instruction *I) {
    return I->getOpcode() >= BinaryOpsBegin &&
           I->getOpcode() <  BinaryOpsEnd;
  }
  static bool classof(const Value *V) {
    return isa<Instruction>(V) &&
           classof(cast<Instruction>(V));
  }
};

// A class narrowing the range to a single opcode:
class AddOperator : public ConcreteOperator<BinaryOperator, Instruction::Add> {
public:
  static bool classof(const Instruction *I) {
    return I->getOpcode() == Instruction::Add;
  }
  static bool classof(const Value *V) {
    return isa<Instruction>(V) && classof(cast<Instruction>(V));
  }
};
```

The two-argument overloads (`classof(const Instruction*)` and `classof(const Value*)`) allow the casting machinery to short-circuit checks: if the compiler knows the static type is already `Instruction*`, only the opcode check runs. The sentinel values `BinaryOpsBegin` / `BinaryOpsEnd` are defined in the `Instruction::BinaryOps` enum, and every new arithmetic instruction added to LLVM must fall within them.

When writing your own class hierarchy, the simplest approach is a tag enum stored in the base class:

```cpp
class ASTNode {
public:
  enum NodeKind { NK_Literal, NK_Binary, NK_If, NK_Last };
  NodeKind getKind() const { return Kind; }
protected:
  explicit ASTNode(NodeKind K) : Kind(K) {}
private:
  NodeKind Kind;
};

class BinaryNode : public ASTNode {
public:
  static bool classof(const ASTNode *N) {
    return N->getKind() == NK_Binary;
  }
  explicit BinaryNode() : ASTNode(NK_Binary) {}
};
```

### 4.1.3 The `isa_impl` Trait System (LLVM 14+)

LLVM 14 introduced a more flexible customisation point: the `isa_impl<To, From>` primary template. This allows third parties to specialise the type predicate without modifying the `To` class — useful when `To` is from a different library or when the check cannot be expressed as a static method on `To`.

```cpp
// In llvm/include/llvm/Support/Casting.h (simplified):
template <typename To, typename From, typename Enabler = void>
struct isa_impl {
  // Default: delegate to To::classof(&Val)
  static inline bool doit(const From &Val) {
    return To::classof(&Val);
  }
};

// Specialisation for the trivially-true case (base class → derived class
// when From IS-A To): avoids calling classof at all.
template <typename To, typename From>
struct isa_impl<To, From, std::enable_if_t<std::is_base_of_v<To, From>>> {
  static inline bool doit(const From &) { return true; }
};
```

To add custom casting logic for your own type hierarchy without touching the `classof` pattern:

```cpp
namespace llvm {
template <>
struct isa_impl<MyNode, ASTNodeBase> {
  static bool doit(const ASTNodeBase &V) {
    return V.getKind() >= ASTNodeBase::MyNodeFirst &&
           V.getKind() <= ASTNodeBase::MyNodeLast;
  }
};
} // namespace llvm
```

This is the recommended approach for new hierarchies in LLVM 14 and later. The `classof` pattern still works and is used throughout the existing LLVM codebase, but `isa_impl` specialisation is preferable for new code that lives outside of `llvm::` proper because it requires no changes to the class being cast to.

---

## 4.2 LLVMContext: The Thread-Local Universe

Every piece of LLVM IR is owned by an `LLVMContext`. Types, metadata, attribute sets, constant strings, diagnostic state — all of it lives in the context. The class declaration is in [`llvm/include/llvm/IR/LLVMContext.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/LLVMContext.h); the implementation is in the pImpl object `LLVMContextImpl` in `llvm/lib/IR/LLVMContextImpl.h`.

### 4.2.1 What LLVMContext Owns

- **Type uniquing tables.** `IntegerType::get(Ctx, 32)` returns the same pointer every time within the same context. Two contexts will produce distinct `i32` objects that compare equal by value but differ by address.
- **Metadata ID tables.** Named metadata nodes (`!dbg`, `!tbaa`, …) are indexed by string; the mapping from string to `unsigned` ID is context-local.
- **Attribute sets.** `AttributeSet` objects are uniqued within a context; two functions compiled into the same module share attribute storage.
- **Diagnostic handlers.** The context holds the callback invoked when LLVM emits a diagnostic (remark, warning, or error).
- **Optimization remarks.** The context controls which remarks are emitted and where they go.

### 4.2.2 Thread Safety and the One-Context-Per-Thread Rule

`LLVMContext` is deliberately **not thread-safe**. The design decision is: rather than locking every type-uniquing lookup (which would create a hot global lock), each thread owns its own context and therefore its own type tables. Modules passed between threads must be transferred with explicit synchronisation.

The canonical multi-threaded usage pattern is one context per compilation unit:

```
Thread 0: LLVMContext Ctx0; compile TU0 → Module M0;
Thread 1: LLVMContext Ctx1; compile TU1 → Module M1;
Main thread: link M0 and M1 into M_merged (after both threads complete).
```

The linker step (`llvm::Linker::linkModules`) merges two modules; both must live in the same context before linking, which means moving one module into the other's context using `llvm::Linker`.

### 4.2.3 Diagnostic Handlers

```cpp
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/DiagnosticInfo.h"

llvm::LLVMContext Ctx;

// Register a callback for all diagnostics
Ctx.setDiagnosticHandlerCallBack(
    [](const llvm::DiagnosticInfo &DI, void *) {
      if (DI.getSeverity() == llvm::DS_Error)
        llvm::errs() << "LLVM error: ";
      DI.print(llvm::errs());
      llvm::errs() << "\n";
    },
    /*DiagContext=*/nullptr);
```

The `setDiagnosticHandler` overload accepts a `std::unique_ptr<DiagnosticHandler>`, which is the preferred approach when you need to subclass `DiagnosticHandler` and capture state.

### 4.2.4 Context Disposal

There is no `LLVMContext::reset()` — contexts are destroyed when they go out of scope, which destroys all owned objects (modules, types, metadata) transitively. Destroying a context while any module it owns is still referenced by live IR is undefined behaviour. The safe pattern is to ensure all `Module`, `Function`, and `Value` objects are destroyed before the context.

---

## 4.3 The ADT Library

The ADT (Abstract Data Types) library, in `llvm/include/llvm/ADT/`, is the most widely used part of LLVM outside of the IR proper. It provides containers and string types tuned for the access patterns of a compiler. Every component is header-only or has minimal dependencies, making the ADT library usable independently of the IR.

### 4.3.1 `SmallVector<T, N>`

`SmallVector<T, N>` is a `std::vector`-like container with an in-class inline buffer for `N` elements. When the element count stays below `N`, no heap allocation occurs. When it grows beyond `N`, the vector allocates on the heap and behaves like `std::vector`. The breakeven size where heap allocation begins to dominate is a function of element size and the allocation overhead of the system allocator; `N = 4` or `N = 8` covers the majority of LLVM's use cases.

```cpp
#include "llvm/ADT/SmallVector.h"

// Inline storage for 4 predecessors — typical for a basic block.
llvm::SmallVector<llvm::BasicBlock *, 4> Preds;

// SmallVector is ABI-compatible with SmallVectorImpl<T>, the type-erased base.
// Pass by SmallVectorImpl& to avoid baking N into function signatures.
void collectPreds(llvm::BasicBlock *BB,
                  llvm::SmallVectorImpl<llvm::BasicBlock *> &Out) {
  for (llvm::BasicBlock *P : llvm::predecessors(BB))
    Out.push_back(P);
}
```

**Performance characteristics.** `push_back` is amortised O(1), identical to `std::vector`. Iteration is cache-friendly: elements are contiguous. The important difference from `std::vector` is that `SmallVector<T, N>` with `N > 0` stores its inline buffer as part of the object, not on the heap. A `SmallVector<int, 64>` on the stack costs 64 * 4 = 256 bytes of stack space regardless of actual element count. Do not choose large inline sizes for vectors that are themselves stored inside other data structures.

**Common pitfall: invalidation during push_back.** If you hold a pointer or reference to an element and then `push_back` an element from the same vector, LLVM's `SmallVector` will assert in debug builds if the element address is inside the vector's range (the "safe to add" assertion). This is the same invalidation rule as `std::vector` but made explicit.

**`SmallVector` vs `std::vector`.** Prefer `SmallVector` for containers that are small in the common case and that live on the stack or inside a data structure. Prefer `std::vector` when elements are large (where the inline buffer would be wasteful), when the container is long-lived and heap-allocated anyway, or when you need move semantics into standard library APIs that expect `std::vector`.

### 4.3.2 `StringRef` — The Non-Owning String View

`StringRef` is LLVM's equivalent of C++17's `std::string_view`. It stores a `const char *` pointer and a `size_t` length. It does **not** own the memory it points to, and it is **not** null-terminated (though it usually is when constructed from a string literal or `std::string`).

```cpp
#include "llvm/ADT/StringRef.h"
#include <string>

void processName(llvm::StringRef Name) {
  // StringRef created from std::string (zero-copy)
  std::string S = "hello";
  llvm::StringRef FromStd(S);         // Points into S's buffer

  // StringRef created from a string literal (points into .rodata)
  llvm::StringRef FromLiteral("world");

  // StringRef created explicitly from the function parameter
  llvm::errs() << "Name: " << Name << " (size=" << Name.size() << ")\n";

  // Common operations — all return new StringRefs or values, never allocate
  bool HasPrefix = Name.starts_with("llvm.");
  llvm::StringRef Tail = Name.drop_front(5); // UB if size < 5; check first
  std::string Owned = Name.str();             // Explicit copy to std::string
}
```

**The lifetime invariant.** A `StringRef` must not outlive the string it references. This is the single most common source of `StringRef`-related bugs:

```cpp
// BUG: returns a StringRef pointing into a destroyed temporary
llvm::StringRef dangling() {
  std::string Temp = computeName();
  return llvm::StringRef(Temp); // Temp destroyed here → dangling pointer
}
```

Never store a `StringRef` in a data structure unless you can guarantee the referenced memory outlives all uses. Store `std::string` (or `llvm::SmallString`) instead.

### 4.3.3 `SmallString<N>`

`SmallString<N>` is a `SmallVector<char, N>` with a string-specific API. It can be built up incrementally with `+=` and converted to `StringRef` at any point:

```cpp
#include "llvm/ADT/SmallString.h"

llvm::SmallString<64> Path;
Path += "/usr/lib/llvm-22/";
Path += "bin/clang";
llvm::StringRef SR = Path; // Zero-copy view; valid as long as Path is live
```

Use `SmallString` instead of `std::string` when building temporary strings in a pass, especially in hot paths. The inline buffer avoids heap allocation for common-length names.

### 4.3.4 `Twine` — Lazy String Concatenation

`Twine` is a lazy, non-allocating concatenation type. Instead of building an intermediate `std::string` for every subexpression, `Twine` stores a binary tree of references to its operands. The actual characters are only materialised when you call `str()` or write the `Twine` to a `raw_ostream`.

```cpp
#include "llvm/ADT/Twine.h"

// No heap allocation until the Twine is consumed
llvm::Twine Name = llvm::Twine("func_") + Prefix + "_" + llvm::Twine(Index);
llvm::Function *F = llvm::Function::Create(FTy, Linkage, Name, &M);
```

**The critical invariant: never store a Twine.** A `Twine` holds raw references to its operands; those operands may be temporaries. The following is undefined behaviour:

```cpp
// BUG: Twine holds references to temporaries that are destroyed
llvm::Twine T = llvm::Twine(std::string("foo")) + std::string("bar");
// T is now a dangling reference tree
```

`Twine` is designed exclusively as a function parameter or a temporary in an expression. LLVM function signatures that accept strings use `const Twine &` precisely to allow callers to compose names inline without allocation. The `Twine` documentation in [`llvm/include/llvm/ADT/Twine.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ADT/Twine.h) includes a comment warning that `Twine` should be used only as a temporary.

### 4.3.5 `ArrayRef<T>` and `MutableArrayRef<T>`

`ArrayRef<T>` is the read-only equivalent of `StringRef` for typed arrays. It stores a `const T*` pointer and a count. It can be constructed from a `SmallVector`, a `std::vector`, a C array, or a brace-initializer list.

```cpp
#include "llvm/ADT/ArrayRef.h"

void printTypes(llvm::ArrayRef<llvm::Type *> Types) {
  for (llvm::Type *T : Types)
    T->print(llvm::errs());
}

// Callers can pass SmallVector, std::vector, or a temporary list
llvm::SmallVector<llvm::Type *, 4> ArgTypes = { Int32Ty, Int64Ty };
printTypes(ArgTypes);
printTypes({ Int32Ty, Int64Ty }); // brace-initializer creates temporary array
```

`MutableArrayRef<T>` adds write access. The same lifetime caution as `StringRef` applies: an `ArrayRef` must not outlive the container it references.

### 4.3.6 `DenseMap<K, V>`

`DenseMap` is an open-addressed hash table using quadratic probing. It stores key-value pairs in a flat array, giving excellent cache performance for small-to-medium tables. The requirement is that `DenseMapInfo<K>` must be specialised for the key type — providing an "empty" key and a "tombstone" key that the table uses as sentinel values.

```cpp
#include "llvm/ADT/DenseMap.h"

// Map from Value* to a use count — a common analysis pattern
llvm::DenseMap<llvm::Value *, unsigned> UseCount;
UseCount[V] = 0;
UseCount[V]++;

auto It = UseCount.find(V);
if (It != UseCount.end())
  llvm::errs() << "Uses: " << It->second << "\n";
```

**Performance characteristics.** Lookup and insert are O(1) average case. The open-addressed layout means that iterating a `DenseMap` is cache-friendly, unlike `std::unordered_map`'s chained buckets.

**Common pitfall: pointer invalidation.** Inserting into a `DenseMap` may rehash, invalidating all iterators and references. The following pattern is a silent bug:

```cpp
// BUG: operator[] on K1 may allocate a slot, then inserting K2 may rehash.
// After the rehash, the reference to Map[K1] is dangling.
llvm::DenseMap<llvm::Value *, unsigned> Map;
Map[K1] = computeFromMap(Map, K2); // RHS may trigger rehash while LHS is live
```

The safe pattern is to compute the right-hand side first, then assign:

```cpp
unsigned Val = computeFromMap(Map, K2); // no insertion yet
Map[K1] = Val;                          // now safe to insert/update
```

Equivalently: never hold an iterator, pointer, or reference into a `DenseMap` across any operation that may insert into the same map.

**The empty/tombstone sentinel requirement.** The key type must have two "impossible" values available for internal use. For pointer keys the default uses `nullptr` (empty) and a small non-null integer (tombstone). For integer keys LLVM uses `~0U` and `~0U - 1`. You must ensure these sentinel values never appear as real keys. If they can, you must provide a custom `DenseMapInfo<K>` specialisation.

### 4.3.7 `DenseSet<T>`

`DenseSet<T>` is built on `DenseMap<T, detail::DenseSetEmpty>` and provides the set-theoretic interface: `insert`, `count`, `contains`, `erase`. It has the same performance and pitfall profile as `DenseMap`.

### 4.3.8 `StringMap<V>`

`StringMap<V>` maps `StringRef`-compatible keys to values. Unlike `DenseMap<std::string, V>`, `StringMap` stores the key string directly in the hash bucket entry, avoiding a separate heap allocation per key. It accepts `StringRef` for lookups, making it natural to pair with `StringRef` parameters:

```cpp
#include "llvm/ADT/StringMap.h"

llvm::StringMap<unsigned> PassTimings;
PassTimings["InstCombine"] = 42;
PassTimings["LoopUnroll"] = 17;

// Lookup by StringRef — no copy
llvm::StringRef PassName = "InstCombine";
auto It = PassTimings.find(PassName);
```

**When to choose `StringMap` over `DenseMap<std::string, V>`.** `StringMap` avoids the heap allocation for keys and supports `StringRef` lookups without conversion to `std::string`. It is the right choice whenever string keys come from StringRef-producing APIs (function names, pass names, metadata keys). Its iteration order is not deterministic.

### 4.3.9 `SetVector<T>` and `SmallSetVector<T, N>`

`DenseSet` is unordered: iterating it gives elements in an arbitrary, nondeterministic order that varies between runs (because pointer hash values depend on ASLR). This nondeterminism causes flaky tests and can produce non-reproducible builds when the iteration order affects the output (instruction ordering in a basic block, the order of passes triggered by a set of analyses, etc.). `SetVector<T>` solves this by maintaining insertion order while providing O(1) membership testing:

```cpp
#include "llvm/ADT/SetVector.h"

// Worklist that processes each basic block exactly once, in insertion order
llvm::SetVector<llvm::BasicBlock *> Worklist;
Worklist.insert(&F.getEntryBlock());

while (!Worklist.empty()) {
  llvm::BasicBlock *BB = Worklist.pop_back_val();
  processBlock(BB);
  for (llvm::BasicBlock *Succ : llvm::successors(BB))
    Worklist.insert(Succ); // insert is a no-op if already present
}
```

`SetVector<T>` internally stores a `std::vector<T>` (for ordering) and a `DenseSet<T>` (for O(1) membership). The space cost is therefore approximately twice that of `DenseSet`. `SmallSetVector<T, N>` uses a `SmallVector` and a `SmallDenseSet` to avoid heap allocation when the set is small.

**When to use `SetVector` instead of `DenseSet`.** Use `SetVector` any time your algorithm produces a set of values that will be iterated and you need reproducible output. Worklists for iterative dataflow algorithms, pass worklists, and sets of functions to process are the canonical cases. If you never iterate the set (only use `count`/`insert`/`erase`), `DenseSet` is cheaper.

### 4.3.10 `MapVector<K, V>`

`MapVector<K, V>` is to `DenseMap` what `SetVector` is to `DenseSet`: it maintains insertion order while providing O(1) lookup. Internally it uses a `DenseMap<K, unsigned>` (mapping key to vector index) plus a `std::vector<std::pair<K, V>>`. Iteration visits elements in insertion order.

```cpp
#include "llvm/ADT/MapVector.h"

// Collect functions in the order they appear in the module,
// but look them up by name in O(1)
llvm::MapVector<llvm::StringRef, llvm::Function *> FuncByName;
for (llvm::Function &F : M)
  FuncByName[F.getName()] = &F;

// Iteration is in insertion order (i.e., module order), not hash order
for (auto &[Name, Func] : FuncByName)
  llvm::errs() << Name << "\n";
```

Erasure from `MapVector` is O(n) — it removes the element from the vector — so `MapVector` is not appropriate when elements are frequently removed. If you need O(1) erase with ordering, combine a `DenseMap` and a `std::list` manually.

### 4.3.11 `StringSwitch<T>`

`StringSwitch<T>` provides a clean alternative to chains of `if (S == "foo") ... else if (S == "bar") ...` for dispatch on string values. It constructs a switch-like chain that is more readable and less error-prone than hand-written comparisons:

```cpp
#include "llvm/ADT/StringSwitch.h"

int getTargetPointerWidth(llvm::StringRef Arch) {
  return llvm::StringSwitch<int>(Arch)
      .Case("i386",    32)
      .Case("x86_64",  64)
      .Case("aarch64", 64)
      .Case("armv7",   32)
      .Case("riscv32", 32)
      .Case("riscv64", 64)
      .Default(-1);           // returned if no case matches
}
```

`StringSwitch` is particularly common in target-description code (determining target ABI, triple parsing, attribute string matching). It is not a hash-table dispatch — it performs linear comparisons — but for the small number of cases typically involved in compiler dispatch, this is faster than constructing a hash table.

### 4.3.12 `std::optional<T>`

LLVM's own `llvm::Optional<T>` was deprecated in LLVM 15 and removed in LLVM 17. All code now uses `std::optional<T>` (C++17). The API is standard: `std::nullopt` to construct the empty state, `has_value()`, `value()`, and `value_or(default)`. No LLVM-specific documentation is needed; use the cppreference page.

### 4.3.13 `llvm::Error` and `llvm::Expected<T>`

These are covered in depth in section 4.5.

---

## 4.4 Support Utilities

The `llvm/include/llvm/Support/` directory contains utilities that do not depend on the IR but are commonly used by compiler code.

### 4.4.1 Output Streams: `raw_ostream`, `errs`, `outs`, `dbgs`

LLVM does not use `std::cout` or `std::cerr`. Its own `raw_ostream` hierarchy provides buffered, format-supporting output with an API modelled on the standard streams but without the `locale` overhead.

```cpp
#include "llvm/Support/raw_ostream.h"
#include "llvm/Support/Debug.h"

// Standard output — corresponds to stdout
llvm::outs() << "Processing module: " << M.getName() << "\n";

// Standard error — corresponds to stderr; tied to outs() so output ordering
// is preserved when both are redirected to the same sink
llvm::errs() << "Warning: unsupported intrinsic\n";

// Debug output — goes to stderr only when LLVM_DEBUG is enabled
// (i.e., when -debug or -debug-only=<component> is passed)
LLVM_DEBUG(llvm::dbgs() << "Visiting instruction: " << I << "\n");

// Writing to a std::string in memory
std::string Buffer;
llvm::raw_string_ostream SS(Buffer);
SS << "Function " << F.getName() << " has " << F.size() << " basic blocks";
std::string Result = SS.str(); // Flushes and returns reference to Buffer
```

The `raw_ostream` is also the type accepted by LLVM's `print` methods (`Value::print`, `Module::print`, `Type::print`), which is why these methods can target files, strings, or stderr uniformly.

`llvm::nulls()` returns a `raw_ostream` that discards all output — useful for discarding printer output when only the side-effect (e.g., verifying IR can be printed) is needed.

### 4.4.2 Formatting: `llvm::format` and `llvm::formatv`

For printf-style formatting, `llvm::format` produces a format object that can be inserted into a `raw_ostream`:

```cpp
#include "llvm/Support/Format.h"
llvm::errs() << llvm::format("%-30s %8.3f ms\n", PassName.str().c_str(), Ms);
```

The `llvm::formatv` function (from `llvm/Support/FormatVariadic.h`) provides a type-safe alternative using Python-style `{N}` placeholders:

```cpp
#include "llvm/Support/FormatVariadic.h"
std::string Msg = llvm::formatv("Pass {0} took {1:F3} ms", PassName, Ms);
llvm::errs() << Msg << "\n";
```

`formatv` supports format specifiers for integers (hex, decimal, binary), floats, and strings. It works with any type that has an `llvm::format_provider<T>` specialisation, which can be added for custom types.

### 4.4.3 Filesystem Utilities: `sys::path` and `sys::fs`

`llvm/Support/Path.h` provides path manipulation:

```cpp
#include "llvm/Support/Path.h"

llvm::SmallString<256> P("/usr/lib/llvm-22/bin/clang");
llvm::sys::path::remove_filename(P);      // P = "/usr/lib/llvm-22/bin"
llvm::StringRef Ext = llvm::sys::path::extension("foo.cpp"); // ".cpp"
llvm::StringRef Stem = llvm::sys::path::stem("foo.cpp");     // "foo"
```

`llvm/Support/FileSystem.h` (`sys::fs`) handles file I/O operations: `sys::fs::exists`, `sys::fs::createTemporaryFile`, `sys::fs::rename`, `sys::fs::remove`, and the `sys::fs::TempFile` RAII type that automatically cleans up on failure.

### 4.4.4 Timers: `Timer` and `TimeRegion`

```cpp
#include "llvm/Support/Timer.h"

llvm::TimerGroup TG("my_pass", "My Pass Timers");
llvm::Timer T("transform", "Transform Phase", TG);

{
  llvm::TimeRegion TR(T); // starts timer on construction, stops on destruction
  performTransformation();
}

// At program exit (or on explicit request), TimerGroup prints a table of
// wall/user/sys time for each named timer.
```

`Timer` measures wall time, user CPU time, and system CPU time. `TimerGroup` collects related timers and prints them as a formatted table. LLVM's `-time-passes` flag enables a predefined timer around every pass.

### 4.4.5 `MemoryBuffer`: Owning File Contents

`llvm::MemoryBuffer` (`llvm/include/llvm/Support/MemoryBuffer.h`) is the standard way to read a file into memory in LLVM. It memory-maps or reads the file into an owned buffer, then presents it as an `ArrayRef<uint8_t>` or `StringRef`. All IR-reading APIs (`parseBitcodeFile`, `parseIRFile`, etc.) accept a `MemoryBufferRef` rather than a file path, which decouples them from the filesystem and makes them directly testable with in-memory strings.

```cpp
#include "llvm/Support/MemoryBuffer.h"

// Read a file; returns Expected<unique_ptr<MemoryBuffer>>
auto MBOrErr = llvm::MemoryBuffer::getFile("/tmp/foo.bc");
if (!MBOrErr) {
  llvm::errs() << "Failed to read file: "
               << MBOrErr.getError().message() << "\n";
  return;
}
std::unique_ptr<llvm::MemoryBuffer> MB = std::move(*MBOrErr);

// MemoryBufferRef is the non-owning view — pass this to parsers
llvm::MemoryBufferRef MBRef = MB->getMemBufferRef();
llvm::StringRef Contents = MBRef.getBuffer();  // the raw bytes
llvm::StringRef Name     = MBRef.getBufferIdentifier(); // the filename
```

`getFile` uses `mmap` on POSIX systems when the file is large enough to benefit (>4KB by default). For small files and for stdin it falls back to `read`. The `WritableMemoryBuffer` variant provides a mutable view — used by tools that patch object files in place. `MemoryBuffer::getMemBuffer` creates a buffer from an existing `StringRef` without copying, which is the test-infrastructure pattern: pass a string literal into any LLVM parser.

### 4.4.6 Target Triple: `llvm::sys::getProcessTriple`

```cpp
#include "llvm/TargetParser/Host.h"

std::string Triple = llvm::sys::getProcessTriple();
// On an x86-64 Linux system: "x86_64-unknown-linux-gnu"
```

`getProcessTriple()` returns the triple describing the process running right now — useful for JIT compilation (Chapter 108) and for setting up the target machine for the current host without hardcoding a triple.

---

## 4.5 Error Handling: `llvm::Error` and `llvm::Expected<T>`

LLVM was written before C++ exceptions were acceptable in LLVM's build configuration (they were disabled by default for a long time, and many embedded LLVM uses still disable them). Rather than silently ignoring failures or returning `nullptr` on error, LLVM 4.0 introduced a checked error type with a mandatory consumption invariant.

### 4.5.1 `llvm::Error`

`llvm::Error` (in [`llvm/include/llvm/Support/Error.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/Error.h)) represents either success or a polymorphic error value. It is marked `[[nodiscard]]`. The invariant: an `Error` value **must** be consumed before destruction — either by checking whether it is an error and handling it, or by passing it to `llvm::cantFail`. Violating this invariant terminates the program in debug builds.

```cpp
#include "llvm/Support/Error.h"

// A function that may fail returns llvm::Error
llvm::Error parseAndProcess(llvm::StringRef Input) {
  auto ParseResult = tryParse(Input);
  if (!ParseResult)
    return ParseResult.takeError();

  // Success case: return explicitly constructed success
  return llvm::Error::success();
}
```

### 4.5.2 `llvm::Expected<T>`

`llvm::Expected<T>` is either a `T` value or an `llvm::Error`. It is the return type when a function succeeds with a value or fails with an error. It is also `[[nodiscard]]`.

The example below uses `llvm::MemoryBuffer` — LLVM's owned buffer abstraction for file contents (`llvm/include/llvm/Support/MemoryBuffer.h`). `MemoryBuffer::getFile` returns an `Expected<std::unique_ptr<MemoryBuffer>>`, and `parseBitcodeFile` returns an `Expected<std::unique_ptr<Module>>`. Both follow the `Expected<T>` protocol: check with `!result` before dereferencing.

```cpp
llvm::Expected<std::unique_ptr<llvm::Module>>
loadModule(llvm::StringRef Path, llvm::LLVMContext &Ctx) {
  auto MB = llvm::MemoryBuffer::getFile(Path);
  if (!MB)
    return llvm::createFileError(Path, MB.getError());

  return llvm::parseBitcodeFile(MB->get()->getMemBufferRef(), Ctx);
}

// Consuming the result
void driver(llvm::LLVMContext &Ctx) {
  auto MaybeM = loadModule("/tmp/foo.bc", Ctx);
  if (!MaybeM) {
    // Extract and print the error, consuming it
    llvm::handleAllErrors(MaybeM.takeError(), [](const llvm::ErrorInfoBase &EI) {
      llvm::errs() << "Load failed: " << EI.message() << "\n";
    });
    return;
  }
  std::unique_ptr<llvm::Module> M = std::move(*MaybeM);
  // use M
}
```

### 4.5.3 `handleErrors` and Error Categories

`handleErrors` dispatches to typed handlers based on the runtime error type:

```cpp
llvm::Error E = doSomething();
E = llvm::handleErrors(
    std::move(E),
    [](const llvm::ECError &ECE) -> llvm::Error {
      // Handle std::error_code-wrapped errors
      llvm::errs() << "System error: " << ECE.message() << "\n";
      return llvm::Error::success(); // consumed
    },
    [](const llvm::StringError &SE) -> llvm::Error {
      // Handle string-message errors
      if (SE.message().contains("recoverable"))
        return llvm::Error::success();
      return llvm::make_error<llvm::StringError>(SE.message(),
                                                  SE.convertToErrorCode());
    });
// Any unhandled error sub-types remain in E
if (E)
  llvm::cantFail(std::move(E)); // assert — should have handled all types
```

### 4.5.4 `llvm::cantFail`

`cantFail` asserts that an `Error` or `Expected<T>` represents success, terminating the program with a message if it does not:

```cpp
// When you are certain a call cannot fail (e.g., fixed input, tested path):
llvm::cantFail(verifyModule(M, &llvm::errs()));

int X = llvm::cantFail(computeResult());
```

Use `cantFail` sparingly — it is a debugging aid, not a production error-handling strategy. Its value is documentation: it makes the assumption explicit and fails loudly rather than silently propagating an unconsumed error.

### 4.5.5 Defining Custom Error Types

User-defined error types allow `handleErrors` to dispatch to typed handlers. A custom error inherits from `llvm::ErrorInfo<Derived>` and provides a static `ID` field and a `log` method:

```cpp
#include "llvm/Support/Error.h"

// A domain-specific error type
class ParseError : public llvm::ErrorInfo<ParseError> {
public:
  static char ID; // unique address used as type tag

  explicit ParseError(llvm::StringRef Msg, unsigned Line)
      : Msg(Msg.str()), Line(Line) {}

  void log(llvm::raw_ostream &OS) const override {
    OS << "parse error at line " << Line << ": " << Msg;
  }

  std::error_code convertToErrorCode() const override {
    return llvm::inconvertibleErrorCode();
  }

  unsigned getLine() const { return Line; }

private:
  std::string Msg;
  unsigned Line;
};

char ParseError::ID = 0; // definition required; address is the type tag

// Creating and handling the error
llvm::Error parse(llvm::StringRef Input) {
  if (Input.empty())
    return llvm::make_error<ParseError>("unexpected end of input", 1);
  return llvm::Error::success();
}

void runParser(llvm::StringRef Input) {
  auto Err = parse(Input);
  llvm::handleAllErrors(std::move(Err),
    [](const ParseError &PE) {
      llvm::errs() << "Line " << PE.getLine() << ": " << PE << "\n";
    });
}
```

The `static char ID` pattern gives each error class a unique address in the process image, which `isa<>` uses internally to dispatch typed handlers. The `log` method is called by `errs() << E` and by `toString(E)`. `convertToErrorCode` enables bridging to `std::error_code`; return `llvm::inconvertibleErrorCode()` if no meaningful mapping exists.

### 4.5.6 Bridging to `std::error_code`

`llvm::errorToErrorCode(E)` converts an `Error` to a `std::error_code` for interoperability with standard library APIs. The reverse direction uses `llvm::errorCodeToError(EC)`. These exist for bridging at API boundaries; prefer `llvm::Error` throughout new LLVM code.

### 4.5.7 How This Differs from Exceptions

The exception analogy is approximate. `llvm::Error` is synchronous and stack-based — the error value travels up the call stack as an explicit return value, and the compiler will warn (via `[[nodiscard]]`) if you discard it. There is no stack unwinding, no `try`/`catch` machinery, no `std::terminate` path from throwing during stack unwinding. The result is fully deterministic performance: an error return has the same overhead as a successful return.

---

## 4.6 Command-Line Options with `cl::opt`

LLVM's command-line library (`llvm/include/llvm/Support/CommandLine.h`) uses a registration pattern: `cl::opt<T>` objects are global variables that register themselves at static initialization time. `cl::ParseCommandLineOptions` then walks the registered set. This design means that adding a new flag requires only declaring a global — no change to a central dispatch table.

### 4.6.1 Basic Option Declarations

```cpp
#include "llvm/Support/CommandLine.h"

// A boolean flag (presence = true)
static llvm::cl::opt<bool> Verbose("verbose",
    llvm::cl::desc("Enable verbose output"),
    llvm::cl::init(false));

// An option with a string argument
static llvm::cl::opt<std::string> OutputFile("o",
    llvm::cl::desc("Output filename"),
    llvm::cl::value_desc("filename"),
    llvm::cl::init("a.out"));

// An option with an integer argument
static llvm::cl::opt<unsigned> InlineThreshold("inline-threshold",
    llvm::cl::desc("Inliner cost threshold"),
    llvm::cl::init(225));

// An option accepting multiple values
static llvm::cl::list<std::string> InputFiles(
    llvm::cl::Positional,
    llvm::cl::desc("<input files>"),
    llvm::cl::ZeroOrMore);
```

### 4.6.2 Option Categories

```cpp
static llvm::cl::OptionCategory MyToolCat("My Tool Options");

static llvm::cl::opt<bool> DumpIR("dump-ir",
    llvm::cl::desc("Dump IR after each pass"),
    llvm::cl::cat(MyToolCat));
```

`HideUnrelatedOptions(MyToolCat)` makes `--help` show only your category, avoiding the full LLVM option flood — essential for tools built on top of LLVM that want clean `--help` output.

### 4.6.3 Invoking the Parser

```cpp
int main(int argc, char *argv[]) {
  llvm::cl::ParseCommandLineOptions(argc, argv, "My LLVM Tool\n");

  if (Verbose)
    llvm::errs() << "Input: " << InputFiles[0] << "\n";
  // ...
}
```

`ParseCommandLineOptions` processes `argv`, fills all `cl::opt` globals, and calls `exit(1)` on a parse error (printing usage). The function returns `true` on success; when the `-help` flag is present, it prints help and exits.

### 4.6.4 Enum Options and Aliases

For options that choose between a fixed set of named values, `cl::opt` supports enum arguments with `cl::values`:

```cpp
enum class OptLevel { O0, O1, O2, O3 };

static llvm::cl::opt<OptLevel> OptimizationLevel(
    "opt-level",
    llvm::cl::desc("Optimization level"),
    llvm::cl::values(
        clEnumValN(OptLevel::O0, "O0", "No optimizations"),
        clEnumValN(OptLevel::O1, "O1", "Basic optimizations"),
        clEnumValN(OptLevel::O2, "O2", "Default optimizations"),
        clEnumValN(OptLevel::O3, "O3", "Aggressive optimizations")),
    llvm::cl::init(OptLevel::O2));
```

`cl::alias` creates an alternative name for an existing option — useful for backward compatibility and for providing short names alongside long names:

```cpp
static llvm::cl::alias OutputFileAlias("output",
    llvm::cl::desc("Alias for -o"),
    llvm::cl::aliasopt(OutputFile));
```

The `cl::alias` is not a copy of the option — it is a reference. Setting the alias sets the original option's value. Both names appear in `--help` output.

### 4.6.5 How Passes Register Options

Passes in the LLVM pass manager use the same mechanism: a `cl::opt<>` defined at file scope in the pass's `.cpp` file is registered when the pass translation unit is linked. This is why passing `-mllvm -inline-threshold=500` from the Clang driver works: the Clang driver passes through `-mllvm` arguments directly to LLVM's option parser, which was populated by whatever passes are linked into the executable.

The ordering of `cl::opt` registration is not guaranteed across translation units (it is static initialization order, which is unspecified across TUs in C++). LLVM relies on the fact that all option names are globally unique. If two TUs define a `cl::opt` with the same name, the second registration will issue a warning and ignore the duplicate.

---

## 4.7 Visitor and Iterator Patterns

### 4.7.1 Iterating Over IR: The Range-Based API

LLVM's IR objects implement the range protocol, making range-`for` the idiomatic iteration style:

```cpp
#include "llvm/IR/Module.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/BasicBlock.h"
#include "llvm/IR/Instruction.h"
#include "llvm/IR/Instructions.h"

// Iterate over all instructions in a module
void visitAllInstructions(llvm::Module &M) {
  for (llvm::Function &F : M) {               // Module → Function
    if (F.isDeclaration()) continue;           // skip external declarations

    for (llvm::BasicBlock &BB : F) {           // Function → BasicBlock
      for (llvm::Instruction &I : BB) {        // BasicBlock → Instruction

        // Per-instruction logic using the casting system
        if (auto *CI = llvm::dyn_cast<llvm::CallInst>(&I)) {
          llvm::errs() << "Call to: "
                       << (CI->getCalledFunction()
                             ? CI->getCalledFunction()->getName()
                             : llvm::StringRef("<indirect>"))
                       << "\n";
        } else if (auto *SI = llvm::dyn_cast<llvm::StoreInst>(&I)) {
          llvm::errs() << "Store to: " << *SI->getPointerOperand() << "\n";
        }
      }
    }
  }
}
```

The three-level nesting — `Module → Function → BasicBlock → Instruction` — is the fundamental access pattern for any whole-module analysis or transformation.

### 4.7.2 Predecessors and Successors

```cpp
#include "llvm/IR/CFG.h"

// Range-for over predecessor basic blocks
for (llvm::BasicBlock *Pred : llvm::predecessors(&BB))
  llvm::errs() << "Predecessor: " << Pred->getName() << "\n";

// Range-for over successor basic blocks
for (llvm::BasicBlock *Succ : llvm::successors(&BB))
  llvm::errs() << "Successor: " << Succ->getName() << "\n";
```

`predecessors(BB)` is implemented as a range over the `use_list` of `BB` — specifically over the `BasicBlock` uses of the terminator instructions that branch to `BB`. The complexity is O(number of predecessors), not O(number of instructions in the function).

### 4.7.3 Use-Def and Def-Use Chains

Every `llvm::Value` maintains a **use list** — the set of instructions that reference it as an operand. Traversing the use list is the foundation of every def-use analysis:

```cpp
#include "llvm/IR/Value.h"

// Iterate over all uses of a value (def-use traversal)
void printUsers(llvm::Value *V) {
  for (llvm::Use &U : V->uses()) {
    llvm::User *UsrInst = U.getUser();
    llvm::errs() << "  Used by: " << *UsrInst << "\n";
  }
}

// Iterate over all users (same information, different entry point)
for (llvm::User *U : V->users()) {
  if (auto *I = llvm::dyn_cast<llvm::Instruction>(U))
    llvm::errs() << "Instruction user in " << I->getParent()->getName() << "\n";
}
```

**`uses()` vs `users()`.** `uses()` yields `llvm::Use&` objects — each `Use` contains both a reference to the user instruction and the operand index within that instruction. This is necessary when the same instruction uses the same value in multiple operand slots (e.g., `add i32 %x, %x`). `users()` yields `llvm::User*` directly and de-duplicates: if `%x` appears twice in `add i32 %x, %x`, `users()` yields the `add` instruction once. Use `uses()` when you need the operand index or position; use `users()` when you only need to know which instructions use the value.

**Replacing all uses.** A common transformation pattern replaces all uses of one value with another:

```cpp
// Replace all uses of OldVal with NewVal throughout the module
OldVal->replaceAllUsesWith(NewVal);
```

`replaceAllUsesWith` walks the use list and updates every operand pointer atomically. After the call, `OldVal->use_empty()` is true. The function is O(number of uses) and safe to call while iterating the use list of a *different* value (but not while iterating the use list of `OldVal` itself).

**Iterating uses during modification.** The use list is a doubly-linked intrusive list. Erasing an element invalidates only that element's iterator, so it is safe to iterate with a cached "next" pointer:

```cpp
// Collect all users into a separate vector before modifying
llvm::SmallVector<llvm::Instruction *, 8> ToProcess;
for (llvm::User *U : V->users())
  if (auto *I = llvm::dyn_cast<llvm::Instruction>(U))
    ToProcess.push_back(I);

for (llvm::Instruction *I : ToProcess)
  processUser(I, V);
```

This copy-then-process pattern is the safest way to handle modifications that may remove uses during traversal.

### 4.7.4 Depth-First and Breadth-First Graph Iterators

```cpp
#include "llvm/ADT/DepthFirstIterator.h"
#include "llvm/ADT/BreadthFirstIterator.h"

// Depth-first traversal of the control-flow graph
for (llvm::BasicBlock *BB : llvm::depth_first(&F.getEntryBlock()))
  llvm::errs() << "DFS: " << BB->getName() << "\n";

// Inverse depth-first (traverses predecessors instead of successors)
for (llvm::BasicBlock *BB :
     llvm::inverse_depth_first(&F.getEntryBlock()))
  llvm::errs() << "inv-DFS: " << BB->getName() << "\n";

// Breadth-first traversal
for (llvm::BasicBlock *BB : llvm::breadth_first(&F.getEntryBlock()))
  llvm::errs() << "BFS: " << BB->getName() << "\n";
```

These iterators operate on any graph type for which `llvm::GraphTraits<T>` is specialised. `GraphTraits` is specialised for `Function*`, `BasicBlock*`, `Inverse<BasicBlock*>`, and several MachineIR types. You can specialise it for your own graph types to get depth-first and breadth-first traversal for free.

The `GraphTraits` contract requires three things: a `NodeRef` type (a pointer or handle to a node), a `ChildIteratorType` (an iterator over successor nodes), and three static functions — `getEntryNode(G)` to get the root, `child_begin(N)` and `child_end(N)` to get the successor range. Providing these six items enables all of LLVM's generic graph algorithms for your type:

```cpp
namespace llvm {
template <>
struct GraphTraits<MyGraph *> {
  using NodeRef = MyNode *;
  using ChildIteratorType = MyNode::SuccIterator;

  static NodeRef getEntryNode(MyGraph *G) { return G->getEntry(); }
  static ChildIteratorType child_begin(NodeRef N) { return N->succ_begin(); }
  static ChildIteratorType child_end(NodeRef N)   { return N->succ_end(); }
};
} // namespace llvm

// Now depth_first and breadth_first work for MyGraph *
for (MyNode *N : llvm::depth_first(MyG))
  process(N);
```

### 4.7.5 `make_range` and Custom Iterator Ranges

`llvm::make_range(begin, end)` turns a pair of iterators into a range usable in range-`for`. It is the bridge between LLVM's legacy begin/end iterator pairs (common in older code) and the modern range-for style:

```cpp
#include "llvm/ADT/iterator_range.h"

// Older LLVM APIs return begin/end pairs; make_range wraps them
auto Users = llvm::make_range(V->use_begin(), V->use_end());
for (llvm::Use &U : Users)
  llvm::errs() << "User: " << *U.getUser() << "\n";
```

### 4.7.6 `InstVisitor`: The Visitor Pattern for Instructions

`llvm::InstVisitor<SubClass, RetTy>` (in [`llvm/include/llvm/IR/InstVisitor.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/InstVisitor.h)) implements the classic visitor design pattern via CRTP. The template machinery dispatches from the `visit(Instruction &I)` entry point to a `visitXXX` method named for the concrete instruction subclass, falling back to `visitInstruction` if no specific handler is defined.

```cpp
#include "llvm/IR/InstVisitor.h"

// A visitor that collects statistics about a function
struct InstructionStats : public llvm::InstVisitor<InstructionStats> {
  unsigned NumLoads = 0;
  unsigned NumStores = 0;
  unsigned NumCalls = 0;
  unsigned NumOther = 0;

  void visitLoadInst(llvm::LoadInst &LI)   { ++NumLoads; }
  void visitStoreInst(llvm::StoreInst &SI) { ++NumStores; }
  void visitCallInst(llvm::CallInst &CI)   { ++NumCalls; }

  // Fallback: any instruction type not handled above
  void visitInstruction(llvm::Instruction &I) { ++NumOther; }

  void report(llvm::StringRef FuncName) {
    llvm::outs() << llvm::formatv(
        "{0}: loads={1}, stores={2}, calls={3}, other={4}\n",
        FuncName, NumLoads, NumStores, NumCalls, NumOther);
  }
};

// Usage
void analyseFunction(llvm::Function &F) {
  InstructionStats Stats;
  Stats.visit(F);      // visits every instruction in every basic block
  Stats.report(F.getName());
}
```

**How the dispatch works.** `InstVisitor` defines a `visit(Instruction &I)` that switches on `I.getOpcode()` using a table of function pointers, each pointing to a `visitXXX` method in the subclass. This is faster than a virtual dispatch: the opcode is already stored in the instruction, and the table jump is direct. The CRTP pattern means the `visitXXX` methods are resolved at compile time — no vtable is needed.

**The fallback chain.** If `visitBinaryOperator` is not defined in the subclass, `InstVisitor` falls back to `visitInstruction`. If `visitCallInst` is not defined but `visitCallBase` is, the call falls back through the class hierarchy: `visitCallInst → visitCallBase → visitInstruction`. This makes it easy to handle groups of related instructions at the right level of abstraction.

**Visitor vs. range-for.** For simple per-instruction processing, range-`for` with `dyn_cast` is often clearer. `InstVisitor` shines when:
- You handle many distinct instruction types with different logic for each.
- The dispatch logic must be shared across different traversal entry points (you can call `visit(Module)`, `visit(Function)`, `visit(BasicBlock)`, or `visit(Instruction)` on the same visitor).
- You want the compiler to warn you (via an unimplemented `visitXXX` if you make it pure virtual in a derived interface) when a new instruction kind is added to LLVM.

---

## 4.8 Chapter Summary

- **Custom RTTI** (`isa<>`, `cast<>`, `dyn_cast<>`, `dyn_cast_or_null<>`) replaces `dynamic_cast` throughout LLVM. The `classof` static method is the required hook; the `isa_impl<To, From>` trait (LLVM 14+) provides a non-intrusive alternative for external hierarchies.

- **`LLVMContext`** is the thread-local universe for all IR state. One context per thread is the rule; type uniquing and metadata IDs are context-scoped. Diagnostic handlers attach to the context, not to modules.

- **`SmallVector<T, N>`** avoids heap allocation for small element counts; prefer `SmallVectorImpl<T>&` in function signatures to avoid baking `N` into the ABI.

- **`StringRef`** is a non-owning string view; it must not outlive the string it references and must not be stored persistently without copying to `std::string` or `SmallString`.

- **`Twine`** enables zero-allocation string concatenation in function call expressions; it must never be stored — only used as a temporary or passed by `const Twine &`.

- **`DenseMap<K,V>`** and **`DenseSet<T>`** are open-addressed hash containers with strong cache performance; their empty/tombstone sentinel requirement must be respected. Insertions invalidate all iterators and references into the same map.

- **`SetVector<T>`** and **`SmallSetVector<T,N>`** provide O(1) membership testing with deterministic insertion-order iteration — the right choice for worklists in analyses where output must be reproducible.

- **`MapVector<K,V>`** provides O(1) lookup with insertion-order iteration; prefer it over `DenseMap` when iteration order affects downstream correctness.

- **`StringMap<V>`** stores string keys inline, accepts `StringRef` lookups, and avoids per-key heap allocation — the right choice for string-keyed metadata.

- **`StringSwitch<T>`** cleanly dispatches on string values without nesting `if`/`else if` chains; common in target-description and ABI-selection code.

- **`std::optional<T>`** is the replacement for the removed `llvm::Optional<T>` (removed in LLVM 17).

- **`raw_ostream`** is the universal output channel. `errs()`, `outs()`, `dbgs()`, and `raw_string_ostream` cover the common cases. `llvm::formatv` provides type-safe, extensible formatting.

- **`llvm::Error`** and **`llvm::Expected<T>`** implement checked, exception-free error handling. Both are `[[nodiscard]]`; every error must be consumed before the owning object is destroyed.

- **`cl::opt<T>`** uses static registration; passing `-mllvm <flag>` from Clang reaches any `cl::opt` linked into the executable. Categories and `HideUnrelatedOptions` clean up `--help` output for tools built on LLVM.

- **Range-`for` over `Module`, `Function`, `BasicBlock`** gives clean, idiomatic access to the IR hierarchy. `predecessors()`, `successors()`, `depth_first()`, `breadth_first()`, and `inverse_depth_first()` complete the CFG traversal toolkit.

- **Use-def traversal** via `V->uses()` and `V->users()` is the foundation of every def-use analysis. `replaceAllUsesWith` is the standard transformation primitive; always copy users to a separate container before modifying the use list in place.

- **`InstVisitor<SubClass>`** implements CRTP dispatch to `visitXXX` methods, enabling clean separation of per-instruction logic without runtime polymorphism. The fallback chain (`visitCallInst → visitCallBase → visitInstruction`) makes it easy to handle instruction groups at the right level of abstraction.

These primitives form the vocabulary of every LLVM subsystem. Chapter 60 puts them to work in a complete pass using the new pass manager. Chapter 134 covers the MLIR C++ API, which reuses many of these patterns — particularly `isa<>`/`dyn_cast<>`, `SmallVector`, and `StringRef` — while adding dialect-specific abstractions for operations, attributes, and types.

---

*References*
- [`llvm/include/llvm/Support/Casting.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/Casting.h) — the complete `isa`/`cast`/`dyn_cast` implementation
- [`llvm/include/llvm/IR/LLVMContext.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/LLVMContext.h) — `LLVMContext` public API
- [`llvm/include/llvm/ADT/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ADT/) — the ADT library headers
- [`llvm/include/llvm/Support/Error.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/Error.h) — `Error` and `Expected<T>`
- [`llvm/include/llvm/Support/CommandLine.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/CommandLine.h) — `cl::opt` and friends
- [`llvm/include/llvm/IR/InstVisitor.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/InstVisitor.h) — `InstVisitor<>` template
- LLVM Programmer's Manual: [https://llvm.org/docs/ProgrammersManual.html](https://llvm.org/docs/ProgrammersManual.html)
