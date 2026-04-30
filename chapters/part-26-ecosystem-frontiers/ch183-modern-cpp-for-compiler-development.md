# Chapter 183 — Modern C++ for Compiler Development: C++23, Contracts, and Reflection

*Part XXVI — Ecosystem and Frontiers*

LLVM was born in C++98. Its current codebase is a monument to careful incremental modernisation: C++11 arrived in LLVM 3.1, C++14 in LLVM 3.9, C++17 became the baseline in LLVM 16, and C++20 features have been landing selectively since LLVM 17. As of April 2026 the project compiles against C++17 with a growing body of C++20 code, is evaluating a C++23 baseline transition, and is watching three C++26 proposals — contracts (P2900), static reflection (P2996), and pattern matching (P2688/P1371) — that would each eliminate a significant class of boilerplate or safety hazard in the codebase. This chapter traces the arc from the existing LLVM vocabulary types and coding standards, through practical C++20 and C++23 features already being deployed, to the experimental C++26 proposals that will reshape how compiler infrastructure is written. The audience is a developer who already writes production C++17 and wants to understand both what to use today and where the language is going for compiler work specifically.

---

## 183.1 LLVM's C++ Baseline and Coding Standards

### 183.1.1 What the Standards Mandate and Ban

The [LLVM Coding Standards](https://llvm.org/docs/CodingStandards.html) impose constraints that are unusual compared to application C++. Two are structural: **no C++ exceptions** in any LLVM library code (the entire codebase is compiled with `-fno-exceptions`), and **no RTTI** (`-fno-rtti`). These choices were made in the early 2000s to reduce code size, link times, and binary size of compiler tools distributed as libraries. The consequences are pervasive: `dynamic_cast` does not work (RTTI disabled), so LLVM provides its own replacement hierarchy; `std::exception` and `throw` do not work, so LLVM provides `llvm::Error` and `llvm::Expected<T>` as a structured error-passing system. Exception-safe destructors still matter because objects can be destroyed during stack unwinding in callers that do enable exceptions (notably libclang C API users), but LLVM code itself never throws.

The standard library is used selectively. `std::vector`, `std::string`, `std::map`, `std::set`, `std::unordered_map`, and `std::unordered_set` appear in LLVM headers and source, but most hot paths prefer LLVM's own containers. The main substitutions are:

| std:: type | LLVM replacement | Rationale |
|---|---|---|
| `std::vector<T>` | `SmallVector<T,N>` | Inline buffer avoids heap for small N |
| `std::string` | `StringRef` (non-owning) / `SmallString<N>` | Avoids copies for identifiers and names |
| `std::string_view` | `StringRef` | `StringRef` predates `string_view` and adds LLVM utilities |
| `std::span<T>` | `ArrayRef<T>` (non-owning) / `MutableArrayRef<T>` | Same design; predates C++20 `span` |
| `std::map<K,V>` | `DenseMap<K,V>` | Open-addressing hash map with tombstones |
| `std::set<T>` | `DenseSet<T>` | Hash set backed by `DenseMap` |
| `std::map<K,V>` (ordered) | `MapVector<K,V>` | Preserves insertion order |
| Hybrid | `SetVector<T>` | Set semantics with vector-backed iteration order |

The coding standard also forbids implementation-defined behaviour used as informal extensions: `int` bit-field widths above 32, signed integer overflow (UB in C++), `reinterpret_cast` for type punning (replaced by `llvm::bit_cast` or `memcpy`).

### 183.1.2 LLVM's Vocabulary Types

**Type hierarchy navigation.** Because `dynamic_cast` is disabled, LLVM uses `classof` static methods on each class and four template functions defined in [`include/llvm/Support/Casting.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/Casting.h):

```cpp
// isa<T>(val) — true if val's dynamic type is T or a subclass of T
if (isa<BinaryOperator>(V)) { ... }

// cast<T>(val) — asserts isa<T> then returns T*; UB if wrong
auto *BO = cast<BinaryOperator>(V);

// dyn_cast<T>(val) — returns T* or nullptr; safe downcast
if (auto *BO = dyn_cast<BinaryOperator>(V)) { ... }

// dyn_cast_or_null<T>(val) — like dyn_cast but accepts null input
auto *BO = dyn_cast_or_null<BinaryOperator>(maybeNull);

// cast_or_null<T>(val) — like cast but null-safe
auto *BO = cast_or_null<BinaryOperator>(maybeNull);
```

Each class participates by implementing `static bool classof(const Base *V)` which checks a kind-enum tag set at construction. This is O(1) per call, avoids the vtable-based RTTI, and works across shared-library boundaries.

**Container vocabulary.**

```cpp
#include "llvm/ADT/SmallVector.h"
#include "llvm/ADT/StringRef.h"
#include "llvm/ADT/ArrayRef.h"
#include "llvm/ADT/DenseMap.h"

// SmallVector<T,N>: N elements inline, heap-spills if N exceeded
SmallVector<Value *, 8> Operands;

// StringRef: non-owning {data, length} pair; zero-copy
StringRef Name = F.getName();

// ArrayRef<T>: non-owning slice of any contiguous T storage
void processOps(ArrayRef<Value *> Ops) { ... }

// DenseMap<K,V>: open-addressed hash map; K must have DenseMapInfo<K>
DenseMap<Instruction *, unsigned> InstNumbers;
InstNumbers[&I] = nextNum++;
```

**Debug output.**

```cpp
#define LLVM_DEBUG(X) do { if (::llvm::DebugFlag && ...) { X; } } while (false)

LLVM_DEBUG(dbgs() << "Processing: " << F.getName() << "\n");
LLVM_DEBUG({
  dbgs() << "Operand list:\n";
  for (auto *Op : Operands)
    Op->dump();
});
```

`LLVM_DEBUG` macros compile to nothing in release builds (when `NDEBUG` is defined and the debug flag is not asserted), guaranteeing zero runtime overhead for debug output in production.

### 183.1.3 The C++ Standard Upgrade Trajectory

The monorepo's minimum standard follows a deliberate RFC process. Relevant milestones:

| LLVM version | Baseline C++ standard | Key enabling RFC |
|---|---|---|
| 14 | C++14 | D74622 — approved 2020 |
| 16 | C++17 | D130689 — approved 2022 |
| 17+ | C++17 + selective C++20 | Per-feature opt-in patches |
| ~18–19 | C++20 (proposed) | Ongoing RFC discussion |
| Future | C++23 / C++26 | Depends on toolchain baseline |

The gating constraint is compiler support in the distributions LLVM CI builds against: GCC 7.1+, Clang 5.0+, MSVC 2019+. As those minimums advance, new standard features become available. The practical effect is that C++20 features are used in new code but the codebase cannot yet assume all compilers being used by downstream consumers support them; features are guarded with `#if __cplusplus >= 202002L` or feature-test macros where portability matters.

---

## 183.2 C++20 in LLVM Today

### 183.2.1 Concepts Replacing SFINAE

C++20 concepts replace the `std::enable_if_t` SFINAE guards that appear throughout `ArrayRef.h` and `SmallVector.h`. Compare old-style SFINAE:

```cpp
// C++17 style — still present in ArrayRef.h in LLVM 22
template <typename U,
          typename = std::enable_if_t<std::is_same<U, T>::value>>
ArrayRef(const SmallVectorTemplateCommon<T, U> &Vec) noexcept;
```

With the concept-based equivalent that new code can use:

```cpp
// C++20 concept-based — clearer and produces better error messages
template <typename Container>
  requires std::is_same_v<std::ranges::range_value_t<Container>, T>
ArrayRef(const Container &C) noexcept
  : Data(C.data()), Length(C.size()) {}
```

The concept form produces a diagnostic that names the failed constraint directly rather than producing a template instantiation traceback through `enable_if`. For LLVM's container APIs — where every misuse produces a long error message — this improvement is measurable in developer productivity.

A relevant in-tree example: [`STLExtras.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ADT/STLExtras.h) defines `llvm::is_integral_or_enum` as a type trait. The C++20 version defines a concept:

```cpp
// In new LLVM code targeting C++20
template <typename T>
concept IntegralOrEnum = std::is_integral_v<T> || std::is_enum_v<T>;

// Constrains hash map key requirements
template <IntegralOrEnum KeyT, typename ValueT>
class SmallDenseMap { ... };
```

### 183.2.2 `std::span` vs `ArrayRef<T>`

`ArrayRef<T>` predates C++20's `std::span` by a decade and implements the same design: a non-owning `{T *data, size_t size}` struct with no ownership semantics. New LLVM APIs targeting C++20 are increasingly written to accept `std::span<const T>` instead of `ArrayRef<T>` because:

1. `std::span` is a standard vocabulary type: users do not need to know the LLVM ADT header.
2. `std::span` participates in standard range algorithms directly.
3. `std::span<T, N>` supports fixed-extent spans; `ArrayRef` does not.

A migration in progress in the LLVM 22 codebase: new `llvm::orc` APIs added in LLVM 21 use `std::span<const JITDylib *>` for dylib lists. Existing APIs remain `ArrayRef<T>` for source compatibility.

### 183.2.3 `[[likely]]` and `[[unlikely]]` on Hot Paths

The `[[likely]]` and `[[unlikely]]` attributes from C++20 appear in LLVM's JIT fast paths and interpreter loops. The key insight is that they direct branch prediction hints into the generated machine code without requiring GCC's `__builtin_expect`:

```cpp
// Before C++20 (still in some LLVM files for GCC compatibility)
if (__builtin_expect(Hit, 1)) {
  // Cache hit path
}

// C++20 — portable across Clang, GCC 9+, MSVC 2022
if (Hit) [[likely]] {
  // Cache hit path
} else [[unlikely]] {
  // Cache miss: fetch from JITDylib
}
```

In LLVM's `OrcJIT` lookup path and the `ExecutionEngine` fast-path, incorrect branch prediction on the cache-miss branch stalls the pipeline; `[[likely]]` ensures the compiler keeps the hit path fall-through and the miss path as a taken branch.

### 183.2.4 `constinit`, `consteval`, and Compile-Time Globals

`constinit` (C++20) guarantees that a variable with static or thread-local storage duration is initialised at compile time, eliminating the static-initialisation-order-fiasco for LLVM's global registries:

```cpp
// C++20 — guaranteed compile-time initialisation; no order dependency
constinit static std::atomic<int> PassRegistrationCount{0};
```

`consteval` marks a function that must be evaluated at compile time. LLVM's `APInt` constant-folding infrastructure benefits from `consteval` helper functions for small constant creation:

```cpp
// C++20 consteval — always a compile-time computation
consteval uint64_t makeOneBits(unsigned N) {
  return N == 64 ? ~uint64_t(0) : (uint64_t(1) << N) - 1;
}
static_assert(makeOneBits(8) == 0xFF);
```

### 183.2.5 `std::bit_cast` Replacing `memcpy`-Based Type Punning

LLVM's `include/llvm/ADT/bit.h` defines `llvm::bit_cast<To>(from)` as a pre-C++20 compatibility shim — a `constexpr` function using `__builtin_bit_cast` when available and `memcpy` otherwise. In C++20 contexts, `std::bit_cast` is the preferred form because it is `constexpr` unconditionally:

```cpp
// Old pattern: UB if used outside constexpr context
float f = ...;
uint32_t bits;
std::memcpy(&bits, &f, sizeof(bits));  // defined, but not constexpr

// LLVM's shim (in llvm/ADT/bit.h)
uint32_t bits = llvm::bit_cast<uint32_t>(f);  // constexpr on Clang/GCC

// C++20 standard — constexpr, no UB, no shim needed
uint32_t bits = std::bit_cast<uint32_t>(f);
```

The `llvm::SwapByteOrder.h` header already uses `llvm::bit_cast` for float-to-integer reinterpretation. Migrating to `std::bit_cast` is a straightforward quality improvement as the C++20 baseline is adopted.

### 183.2.6 Designated Initialisers for Aggregate Initialisation

C++20 designated initialisers eliminate positional initialisation for aggregate structs with many fields — a common pattern in `MCInst`, `MachineOperand`, and LLVM's target descriptor tables:

```cpp
// C++17 aggregate init — positional, fragile
MCInstrDesc Desc = {
  RISC_V::ADD, 3, 1, 0, 0, 0, nullptr, nullptr, nullptr, 0, 0
};

// C++20 designated — named, survives field reordering
MCInstrDesc Desc = {
  .Opcode         = RISCV::ADD,
  .NumOperands    = 3,
  .NumDefs        = 1,
  .Size           = 4,
  .SchedClass     = 0,
  .ImplicitUses   = nullptr,
  .ImplicitDefs   = nullptr,
};
```

For the target descriptor tables that TableGen generates, designated initialisers also serve as documentation — a future reader can see field names rather than having to count positions.

---

## 183.3 C++23 for Compiler Infrastructure

### 183.3.1 `std::expected<T,E>` vs `llvm::Expected<T>`

LLVM's error handling predates `std::expected` (standardised in C++23) by over a decade. `llvm::Expected<T>` and `llvm::Error` enforce a strict ownership model: an `llvm::Error` **must** be checked before destruction; failing to do so triggers an assertion in debug builds and a `llvm_unreachable` in release. This is enforced via a destructor that checks a `Checked` flag:

```cpp
// llvm::Expected<T> — enforced-checked error type (pre-C++23)
llvm::Expected<uint64_t> readInteger(llvm::StringRef S) {
  uint64_t Val;
  if (S.getAsInteger(10, Val))
    return llvm::createStringError(llvm::inconvertibleErrorCode(),
                                   "not an integer: %s", S.str().c_str());
  return Val;
}

// At the call site — must handle the error
if (auto ValOrErr = readInteger(Token)) {
  use(*ValOrErr);
} else {
  llvm::handleAllErrors(ValOrErr.takeError(), [](const llvm::StringError &E) {
    llvm::errs() << E.getMessage() << "\n";
  });
}
```

`std::expected<T,E>` provides the same value-or-error type but as a standard vocabulary:

```cpp
// C++23 — requires: clang++ -std=c++23
#include <expected>
#include <string>

std::expected<uint64_t, std::string> readInteger(std::string_view S) {
  uint64_t Val = 0;
  for (char c : S) {
    if (c < '0' || c > '9')
      return std::unexpected("not an integer: " + std::string(S));
    Val = Val * 10 + (c - '0');
  }
  return Val;
}

// Monadic chaining — available in C++23
auto result = readInteger(token)
  .transform([](uint64_t v) { return v * 2; })
  .transform_error([](const std::string &e) {
    return "parse error: " + e;
  });
```

The critical difference is composability. `std::expected` supports monadic operations (`and_then`, `or_else`, `transform`, `transform_error`) that allow chaining operations without explicit `if` checks. `llvm::Expected<T>` has no monadic interface; each call site must explicitly check and handle. For new LLVM utility code targeting C++23, `std::expected` is the better choice; existing APIs built on `llvm::Expected<T>` will remain for ABI stability.

### 183.3.2 `std::mdspan`: Multi-Dimensional Views and MLIR MemRef

`std::mdspan` (C++23) is a non-owning, multi-dimensional array view. Its type signature is `std::mdspan<T, Extents, LayoutPolicy, AccessorPolicy>` where `Extents` encodes the dimensionality and sizes. This maps precisely to the MLIR MemRef buffer descriptor:

A `memref<?x?xf32>` in MLIR lowering produces a descriptor with fields: `{T *allocated_ptr, T *aligned_ptr, intptr_t offset, intptr_t sizes[rank], intptr_t strides[rank]}`. The `std::mdspan` C++ equivalent:

```cpp
// C++23 — requires: clang++ -std=c++23
#include <mdspan>
#include <vector>

// Create a shaped view over a flat buffer — equivalent to memref<?x?xf32>
std::vector<float> buffer(M * N);
std::mdspan<float,
            std::dextents<std::size_t, 2>,   // 2D dynamic extents
            std::layout_stride>              // arbitrary strides
  view(buffer.data(),
       std::dextents<std::size_t, 2>{M, N},
       std::layout_stride::mapping{
           std::dextents<std::size_t, 2>{M, N},
           std::array<std::size_t, 2>{N, 1}  // row-major strides
       });

// Element access — same semantics as MLIR memref load
float elem = view[i, j];  // C++23 multi-subscript operator
```

The three layout policies mirror MLIR tiling strategies:
- `std::layout_right` — row-major (C order); matches `memref<?x?xf32>` with default strides
- `std::layout_left` — column-major (Fortran order); matches transposed MemRef
- `std::layout_stride` — arbitrary strides; matches tiled or non-contiguous MemRef descriptors

For vectorisation cost models and linalg tiling analysis, `std::mdspan` provides a C++-native way to express shaped array views that are semantically equivalent to what MLIR's `-convert-memref-to-llvm` produces. A pass that analyses MLIR linalg ops can prototype its cost model using `std::mdspan` over the buffer descriptors extracted from the lowered IR.

### 183.3.3 `std::flat_map` and `std::flat_set`

`std::flat_map<K,V>` and `std::flat_set<T>` (C++23) are sorted-vector-backed associative containers. Unlike `std::map` (tree nodes, pointer indirection, poor cache behaviour) or `std::unordered_map` (hash bucket overhead), `std::flat_map` stores keys and values in two contiguous sorted vectors. This gives:

- O(log n) lookup (binary search on contiguous data — cache-friendly)
- O(n) insertion (sorted-vector shift — expensive for frequent insertions)
- Optimal for read-heavy, small-to-medium collections built once and queried repeatedly

The performance profile matches `llvm::SmallDenseMap` for typical instruction attribute maps: a `MachineInstr` holds a small number of per-operand attributes (constraint, tied register, implicit-def flag) that are set during instruction selection and read many times during scheduling and register allocation:

```cpp
// C++23 — requires: clang++ -std=c++23
#include <flat_map>

// Instruction attribute map — built once during selection, queried during RA
std::flat_map<unsigned, OperandConstraint> OperandConstraints;
OperandConstraints.insert_or_assign(0, RC_GPR32);
OperandConstraints.insert_or_assign(1, RC_GPR32);
OperandConstraints.insert_or_assign(2, RC_GPR32_OR_IMM12);

// O(log n) binary search on a contiguous key vector — predictable prefetch
auto it = OperandConstraints.find(opIdx);
if (it != OperandConstraints.end()) {
  applyConstraint(it->second);
}
```

For LLVM's `DenseMap<K,V>` use cases where K is a small integer, `std::flat_map` can replace it with better constant factors due to SIMD-friendly sequential search at small sizes.

### 183.3.4 Deducing `this` (P0847) and the End of CRTP

C++23's deducing-`this` feature (P0847, standardised) adds an explicit `this` parameter to member functions, allowing a single function template to serve the role previously requiring CRTP:

```cpp
// C++23 — requires: clang++ -std=c++23

// CRTP pattern — C++17 and earlier
template <typename Derived>
class VisitorBase {
public:
  // Must upcast through Derived — two-phase lookup, verbose
  void visitAll(llvm::Function &F) {
    for (auto &BB : F)
      for (auto &I : BB)
        static_cast<Derived *>(this)->visit(I);
  }
};

class MyVisitor : public VisitorBase<MyVisitor> {
public:
  void visit(llvm::Instruction &I) { /* ... */ }
};

// Deducing-this equivalent — no CRTP, no static_cast
class VisitorBase2 {
public:
  // 'this auto& self' deduces to the concrete derived type
  void visitAll(this auto& self, llvm::Function &F) {
    for (auto &BB : F)
      for (auto &I : BB)
        self.visit(I);  // dispatches to derived visit() at compile time
  }
};

class MyVisitor2 : public VisitorBase2 {
public:
  void visit(llvm::Instruction &I) { /* ... */ }
};
```

The MLIR `PassWrapper<PassT, BaseT>` from [`mlir/Pass/Pass.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Pass/Pass.h#L469) is a textbook CRTP: it takes `PassT` as a template parameter solely to synthesise `classof` and `clonePass` — both of which could be written as deducing-`this` member functions or `std::derived_from` concept constraints. The cost of CRTP at scale is concrete: each instantiation of `PassWrapper<MyPass, OperationPass<func::FuncOp>>` instantiates a fresh set of member function templates, multiplying symbol count and compile time across the ~200 in-tree MLIR passes.

### 183.3.5 `std::ranges` Views for IR Iteration

`std::ranges` (C++20, increasingly used in C++23 contexts) provides lazy, composable views over sequences. LLVM's IR iterators — `Function::iterator`, `BasicBlock::iterator`, `Region::iterator` — satisfy `std::forward_range`, making them directly composable with range views:

```cpp
// C++20/23 — requires: clang++ -std=c++20 or -std=c++23
#include <ranges>

// Find all call instructions in a function — lazy filter
auto calls = F | std::views::join   // flatten Function -> BasicBlocks -> Instructions
               | std::views::filter([](const llvm::Instruction &I) {
                   return isa<llvm::CallInst>(I);
                 });

for (const llvm::Instruction &I : calls) {
  const auto *CI = cast<llvm::CallInst>(I);
  processCall(*CI);
}

// Transform: extract all operand Values from a block's instructions
auto operands = BB
  | std::views::transform([](const llvm::Instruction &I) {
      return I.operands();
    })
  | std::views::join;

// Take the first N successors — useful for bounded analysis
auto firstTwo = llvm::successors(&BB) | std::views::take(2);
```

The key advantage is that `std::views::filter` and `std::views::transform` are lazy: they do not materialise a new container; they apply the predicate or transform only when the iterator advances. This avoids the `SmallVector` temporaries that current LLVM code builds for filtered collections. The cost is that the composed range type is complex and its type name is unwritable — `auto` is mandatory. This can cause issues with structured bindings in range-for loops; prefer explicit `auto &&` to avoid surprises with proxy iterators.

---

## 183.4 C++26 Contracts (P2900)

### 183.4.1 Syntax and Placement

C++26 contracts (proposal P2900, in SG21 Safety and Security Study Group) add three kinds of assertion directly to C++ function declarations and bodies:

```cpp
// requires: -fcontracts (experimental, Clang 22)

// pre — precondition; evaluated before function body executes
// post(result: expr) — postcondition; 'result' names the return value
// contract_assert — mid-function invariant assertion

int divide(int a, int b)
  pre(b != 0)                    // precondition
  post(result: result * b == a)  // postcondition — not always expressible, but illustrative
{
  contract_assert(a >= 0);       // mid-function contract
  return a / b;
}
```

Contracts are **not** exception-based and **not** debug-only (unlike `assert`). They have four evaluation semantics selectable per build:

| Semantics | Violation handling | DCE allowed | Zero overhead |
|---|---|---|---|
| `ignore` | Not evaluated | Yes | Yes |
| `observe` | Call violation handler; continue | No | No |
| `enforce` | Call violation handler; terminate | No | No |
| `quick_enforce` | UB if violated; allows aggressive DCE | Yes | Yes (as UB) |

`ignore` is the intended release mode for shipped binaries where the performance overhead of checking must be zero. `enforce` is the debug and testing mode. `observe` is for monitoring in production without terminating. `quick_enforce` allows the compiler to assume contracts are always satisfied and use them as optimisation hints — equivalent to `__builtin_assume` in Clang today.

### 183.4.2 Applying Contracts to LLVM Pass APIs

The `PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM)` interface is the primary LLVM pass entry point. Today it relies on convention and `llvm_unreachable` for precondition checking. With contracts:

```cpp
// requires: -fcontracts (experimental, Clang 22)
#include "llvm/IR/Function.h"
#include "llvm/Analysis/AnalysisManager.h"

struct MyPass {
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM)
    pre(!F.isDeclaration())            // only run on definitions
    pre(AM.isNotInvalidated(F))        // analysis manager must be fresh for F
    post(result: result != PreservedAnalyses::none());  // must preserve something
  {
    // Internal invariant: entry block must have a terminator
    contract_assert(!F.getEntryBlock().empty());
    contract_assert(F.getEntryBlock().getTerminator() != nullptr);

    // ... pass logic ...
    return PreservedAnalyses::all();
  }
};
```

The `pre(!F.isDeclaration())` replaces the common pattern:

```cpp
// Current LLVM style — assert abort in debug, UB in release
assert(!F.isDeclaration() && "Pass should not be run on declarations");
```

The contract form is semantically richer: in `observe` mode it calls the violation handler and continues, collecting contract violations across a test suite without stopping at the first failure. In `enforce` mode it terminates with a location-annotated error that identifies the exact pre/postcondition that failed.

**Contracts as complement, not replacement.** Contracts operate at API boundaries — the interface between the caller (pass manager) and the callee (pass implementation). `llvm::verifyFunction(F)` checks internal IR consistency; `AssertingVH<>` checks that value handles remain valid across transformations; `LLVM_DEBUG` guards diagnostic output. These mechanisms target different invariant classes and are all complementary. Contracts on `run()` establish the calling convention; `verifyFunction` establishes IR structural integrity; `AssertingVH` establishes handle lifetime. None substitutes for the others.

### 183.4.3 Contracts on MLIR PatternRewriter

MLIR's `PatternRewriter` interface is another natural fit. Today `replaceOp` requires that the replacement values have compatible types with the op's results, but this is checked by an assertion buried in the implementation. With contracts:

```cpp
// requires: -fcontracts (experimental, Clang 22)

// On PatternRewriter::replaceOp(Operation *op, ValueRange newValues):
void replaceOp(Operation *op, ValueRange newValues)
  pre(op != nullptr)
  pre(op->getNumResults() == newValues.size())
  pre(llvm::all_of(llvm::zip(op->getResultTypes(), newValues),
        [](auto pair) {
          return std::get<0>(pair) == std::get<1>(pair).getType();
        }))
  post(op->use_empty())   // all uses have been replaced
{
  // implementation
}
```

The `post(op->use_empty())` postcondition documents the semantic guarantee that was previously only in a comment: after `replaceOp`, the original op's results have no remaining uses. A pattern that incorrectly calls `replaceOp` without replacing all uses would have this postcondition fire in `enforce` mode, immediately identifying the bug.

### 183.4.4 WG21 Design History

The contract proposal has been in committee since P0542 (2016). The key design questions were: syntax (attributes `[[pre:]]` vs keyword `pre(...)`), placement (declaration only vs definition only vs both), and violation handling model (single global handler vs build-mode selection vs per-contract semantics). P2900 adopts keyword syntax (not attributes), placement on declarations and definitions, and a build-mode selection model where all contracts in a translation unit share an evaluation semantic.

The violation handler model was contentious: the "three-level" model (check always, check at debug, assume for optimisation) maps directly to `ignore`/`enforce`/`quick_enforce`. The `observe` semantic (check but continue) was added to support telemetry workflows where a violation in production should be logged but not terminate the process. SG21's rationale: terminating on precondition violation in production software that has never failed its test suite is too aggressive; `observe` bridges the deployment gap.

### 183.4.5 Relationship to Verification in Ch181

C++ contracts differ fundamentally from the pre/postcondition systems in Dafny and Verus discussed in [Chapter 181 — Formal Verification in Practice](../part-26-ecosystem-frontiers/ch181-formal-verification-in-practice.md). Dafny and Verus discharge pre/postconditions **statically** via SMT solvers: a program that compiles in Dafny is guaranteed, at compile time, to satisfy all specifications on all inputs. C++ contracts are **runtime assertions**: they fire when the program executes with a violating input. A C++ contract can express properties that no SMT solver could verify (e.g., properties involving arbitrary heap state) but provides no static guarantee. The two mechanisms are complementary: contracts provide cheap runtime checking for API boundary invariants; Dafny/Verus provide static guarantees for algorithm correctness. An ideal workflow uses contracts as a safety net during development and testing, and applies formal verification to the most critical algorithmic cores.

---

## 183.5 C++26 Static Reflection (P2996)

### 183.5.1 Core Mechanism

P2996 proposes a compile-time reflection facility based on two operators:

- `^expr` — the **reflect** operator; produces a `std::meta::info` constant representing the reflected entity (type, function, variable, member, namespace)
- `[:r:]` — the **splice** operator; takes a `std::meta::info` constant and splices it back into a syntactic context as a type, expression, or declaration

```cpp
// requires: P2996 experimental implementation (EDG / Clang P2996 branch)
#include <meta>

// Reflect a type into a meta::info constant
constexpr auto info = ^int;

// Splice it back as a type
[:info:] x = 42;  // equivalent to: int x = 42;

// Reflect a class member
struct Point { int x; int y; };
constexpr auto members = std::meta::members_of(^Point);
// members is a constexpr range of meta::info values

// Query member properties
static_assert(std::meta::name_of(members[0]) == "x");
static_assert(std::meta::offset_of(members[0]) == 0);
```

Key metafunctions from the `std::meta` namespace:

| Metafunction | Returns |
|---|---|
| `members_of(^T)` | Range of `meta::info` for all members of `T` |
| `bases_of(^T)` | Range of `meta::info` for all base classes |
| `type_of(r)` | The type reflected by `r` |
| `name_of(r)` | `std::string_view` of the entity's unqualified name |
| `qualified_name_of(r)` | Fully qualified name |
| `offset_of(r)` | Byte offset of a data member |
| `is_public(r)` | True if the reflected member has public access |
| `define_class(info, members)` | Synthesises a new class type at compile time |

### 183.5.2 Application 1: Replacing X-Macro Dialect Registration

The classic LLVM pattern for op-name dispatch is the X-macro:

```cpp
// LLVM X-macro pattern — current practice
#define FOR_ALL_OPS(X)    \
  X(AddOp, "add")         \
  X(MulOp, "mul")         \
  X(SubOp, "sub")

// Generate the dispatch table
const char *getOpName(OpKind K) {
  switch (K) {
#define X(Op, Name) case OpKind::Op: return Name;
    FOR_ALL_OPS(X)
#undef X
  }
}

// Register ops
void registerOps(DialectRegistry &R) {
#define X(Op, Name) R.register<Op>();
  FOR_ALL_OPS(X)
#undef X
}
```

With P2996 reflection, the dispatch table is synthesised from the type list directly:

```cpp
// requires: P2996 experimental implementation (EDG / Clang P2996 branch)

// Tag types in the dialect struct
struct MyDialect {
  using Ops = std::tuple<AddOp, MulOp, SubOp>;
};

// Reflection-based registration — no X-macro
template <typename DialectT>
void registerAllOps(DialectRegistry &R) {
  // Iterate over all tuple element types at compile time
  []<std::size_t... I>(std::index_sequence<I...>, DialectRegistry &R) {
    (R.template registerOp<std::tuple_element_t<I, typename DialectT::Ops>>(), ...);
  }(std::make_index_sequence<std::tuple_size_v<typename DialectT::Ops>>{}, R);
}

// With P2996 — iterate directly over annotated members
consteval auto getDialectOps() {
  return std::meta::members_of(^MyDialect)
       | std::views::filter([](std::meta::info m) {
           return std::meta::is_type(m) &&
                  std::meta::has_annotation(m, ^mlir_op);
         });
}
```

The reflection form eliminates the synchronisation problem inherent in X-macros: adding a new op requires updating one location (the struct definition with the annotation) rather than updating the `FOR_ALL_OPS` macro and all the boilerplate files that include it.

### 183.5.3 Application 2: TableGen-Like ODS Boilerplate from C++ Structs

MLIR's TableGen ODS (Operation Definition Specification) generates `getNumOperands()`, `getResult(unsigned)`, `getAttr<T>()`, and verification methods from a `.td` file. With P2996, this generation can happen entirely within C++ using annotated structs:

```cpp
// requires: P2996 experimental implementation (EDG / Clang P2996 branch)

// Annotated op struct — replaces ODS .td file
struct AddOpDef {
  [[= mlir_operand]] mlir::Value lhs;
  [[= mlir_operand]] mlir::Value rhs;
  [[= mlir_result]]  mlir::Value result;
  [[= mlir_attr]]    IntegerAttr overflow_flags;
};

// Reflection-generated accessors — synthesised at compile time
template <typename OpDef>
struct ReflectedOp {
  // Count operands by filtering annotated members
  static consteval unsigned numOperands() {
    std::size_t count = 0;
    for (auto m : std::meta::members_of(^OpDef))
      if (std::meta::has_annotation(m, ^mlir_operand))
        ++count;
    return count;
  }
  static_assert(numOperands() >= 0); // force consteval evaluation

  // Generate getOperand(unsigned idx) that returns the indexed operand
  // (full implementation requires define_class to synthesise the method body)
};
```

This approach does not eliminate the complexity of MLIR ODS — the `.td` system handles constraints, traits, and dialect registration in ways that reflection alone cannot replace without `define_class` and more. But for straightforward ops without complex constraints, a reflection-based ODS alternative would compile significantly faster than TableGen, because it operates within a single C++ compilation unit rather than requiring a separate `.td` → `.inc` code generation step.

### 183.5.4 Application 3: Replacing `isa<>` / `dyn_cast<>` Chains

LLVM's `isa<>` infrastructure works through `classof` static methods. With P2996, the type hierarchy can be queried at compile time to generate exhaustive dispatch:

```cpp
// requires: P2996 experimental implementation (EDG / Clang P2996 branch)

// Base class annotated with its known subclasses
struct [[= has_subclasses(AddInst, SubInst, MulInst, DivInst)]] ArithInst
  : Instruction {};

// Reflection-generated exhaustive type switch
template <typename BaseT, typename F>
auto reflectedTypeSwitch(BaseT *inst, F &&handler) {
  constexpr auto subclasses = getAnnotation(^BaseT, ^has_subclasses);
  // Iterate over annotated subclasses and generate isa<> checks
  return [&]<std::meta::info... Cls>(/* subclasses */) {
    return (... || (isa<[:Cls:]>(inst)
                    && (handler(cast<[:Cls:]>(inst)), true)));
  }(subclasses...);
}

// Usage — replaces cascaded dyn_cast chain
reflectedTypeSwitch(inst,
  [](AddInst *add) { foldAdd(*add); },
  [](MulInst *mul) { foldMul(*mul); }
);
```

The exhaustiveness guarantee is the key advantage over hand-written `dyn_cast` chains: if a new subclass is added to `has_subclasses(...)`, a `static_assert` can verify that the handler set covers all cases.

### 183.5.5 Comparison with Rust Proc-Macros

Rust's procedural macros ([Chapter 178 — The Rust Compiler Ecosystem](../part-26-ecosystem-frontiers/ch178-the-rust-compiler-ecosystem.md)) operate on token streams at the syntactic level: a proc-macro receives a `TokenStream` representing the annotated item and emits a `TokenStream` of generated code. This makes them powerful (any syntactic transformation is expressible) but weakly typed: the macro sees tokens, not resolved types, so cross-crate type dependencies must be managed manually.

P2996 reflection operates on fully-typed, fully-resolved C++ entities within a single `consteval` context. The type system is fully available: `std::meta::type_of(r)` returns the resolved type, `std::meta::bases_of(^T)` returns the complete base class list. This makes reflection more principled than proc-macros for tasks that require type information, but less flexible for tasks that require syntactic transformation before type checking (e.g., domain-specific notation embedded in function bodies). Both are compile-time code generation mechanisms; they solve different problems.

### 183.5.6 Current Status

The EDG reference implementation and the Clang P2996 experimental branch provide a working implementation of P2996 reachable via Compiler Explorer (`clang++ -std=c++26 -freflection`). Key open design questions involve interaction with C++20 modules (how `import` interacts with `consteval` reflection across module boundaries) and `consteval` restrictions on what can be done inside a constant-evaluation context. P2996 is a C++26 candidate; its landing depends on resolving these interactions in the upcoming Wrocław plenary.

---

## 183.6 C++26 Pattern Matching (P2688 / P1371)

### 183.6.1 Syntax

C++26 pattern matching (proposals P2688 and P1371) adds an `inspect` expression and statement. The syntax:

```cpp
// C++26/29 proposal — not yet standardised
// Pattern matching preview syntax

inspect (expr) {
  pattern1 => expression1;
  pattern2 => expression2;
  _        => default_expression;
}
```

Patterns include:
- `_` — wildcard, matches anything
- `x` — binding pattern, binds matched value to `x`
- `<T>` — type-test pattern, matches if expression has dynamic type `T`
- `[a, b]` — structured-binding pattern, destructures
- `== value` — equality pattern
- `pattern if guard` — guarded pattern

### 183.6.2 Replacing `dyn_cast` Chains in LLVM

The current LLVM pattern for instruction dispatch:

```cpp
// Current C++ — cascaded dyn_cast chain
if (auto *BO = dyn_cast<BinaryOperator>(V)) {
  foldBinaryOp(*BO);
} else if (auto *CI = dyn_cast<CallInst>(V)) {
  inlineIfProfitable(*CI);
} else if (auto *PHI = dyn_cast<PHINode>(V)) {
  simplifyPHI(*PHI);
} else {
  // unhandled
}
```

With pattern matching:

```cpp
// C++26/29 proposal — not yet standardised
inspect (*V) {
  <BinaryOperator> bo => foldBinaryOp(bo);
  <CallInst>       ci => inlineIfProfitable(ci);
  <PHINode>       phi => simplifyPHI(phi);
  _                   => { /* unhandled */ }
};
```

The `inspect` form is shorter and its exhaustiveness can be checked by the compiler if the pattern set is declared total. Today, the cascaded `dyn_cast` chain silently falls through to the final `else` for unhandled cases; a mistakenly missing case produces no diagnostic. A pattern matching `inspect` with no wildcard and a closed type hierarchy can require the compiler to prove coverage.

### 183.6.3 Replacing `llvm::TypeSwitch` in MLIR

MLIR's `llvm::TypeSwitch<Operation *, LogicalResult>` is a fluent builder for op dispatch. It is functional today but has awkward ergonomics and no exhaustiveness checking:

```cpp
// Current MLIR style — TypeSwitch
return llvm::TypeSwitch<Operation *, LogicalResult>(op)
  .Case<AddOp>([&](AddOp add) { return foldAdd(add); })
  .Case<MulOp>([&](MulOp mul) { return foldMul(mul); })
  .Default([](Operation *) { return failure(); });
```

With `inspect`:

```cpp
// C++26/29 proposal — not yet standardised
return inspect (op) {
  <AddOp> add => foldAdd(add);
  <MulOp> mul => foldMul(mul);
  _           => failure();
};
```

For MLIR type hierarchy dispatch (integer types, MemRef types, vector types):

```cpp
// C++26/29 proposal — not yet standardised
return inspect (type) {
  <IntegerType>  it => handleInteger(it);
  <MemRefType>   mt => handleMemRef(mt);
  <VectorType>   vt => handleVector(vt);
  <FunctionType> ft => handleFunction(ft);
  _                 => failure();
};
```

### 183.6.4 Current WG21 Status

P2688 and P1371 are active proposals in WG21 with evolved design through successive revisions. Pattern matching is a C++29 candidate if it does not reach C++26 due to remaining design questions around:

- **Exhaustiveness checking**: should `inspect` require exhaustiveness for closed class hierarchies, as Rust `match` does? If so, how is "closed" declared for open class hierarchies in shared libraries?
- **Fall-through semantics**: C++ switch falls through by default; `inspect` must not, but interaction with existing `switch` habits is a language evolution concern.
- **Binding scope**: does `<T> x => expr` bind `x` with the type `T*`, `T&`, or `T` (by value)? The value category rules interact with LLVM's pointer-heavy IR representation.

Comparison with Rust `match`: Rust match is **exhaustive by default** — the compiler requires all cases to be handled or a wildcard to be present. This is the fundamental safety property that makes Rust match more useful than C++ switch for IR dispatch. C++ pattern matching is designed to reach the same guarantee for closed class hierarchies; for open hierarchies (where subclasses may be added in shared libraries), a wildcard or `Default` arm remains necessary.

---

## 183.7 Template Metaprogramming Modernisation in LLVM

### 183.7.1 CRTP Today and Its Costs

CRTP (`template <typename Derived> class Base`) is pervasive in LLVM and MLIR. The canonical uses:

- `PassWrapper<PassT, BaseT>` in [`mlir/Pass/Pass.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Pass/Pass.h#L469) — synthesises `classof`, `clonePass`, `getName`
- `VisitorBase<Derived>` in instruction visitor infrastructure
- `RecursiveASTVisitor<Derived>` in Clang's AST visitor

The costs of CRTP are concrete:
1. **Symbol explosion**: each `PassWrapper<ConcretePass, OperationPass<FuncOp>>` instantiation creates a fresh vtable and debug info.
2. **Two-phase name lookup**: method calls through `static_cast<Derived*>(this)->method()` are resolved at instantiation time, not definition time, making error messages harder to read.
3. **Concept constraints not expressible on `Derived`**: CRTP cannot use C++20 concepts to constrain what `Derived` must provide — only `static_assert` with trait checks, which are less readable.

C++23 deducing `this` (Section 183.3.4) and C++20 concepts together provide a migration path. The `PassWrapper` `classof` method is the only method that truly requires `PassT` today; with `TypeID::get<>()` and concepts, the CRTP can be replaced by a single non-template base class with a virtual `getTypeID()` method that each pass defines via a `= TypeID::get<std::remove_cv_t<decltype(*this)>>()` default — which requires deducing `this` for the default argument.

### 183.7.2 `if constexpr` Replacing Template Specialisations

`if constexpr` (C++17, used widely in LLVM since LLVM 16) replaces template specialisation for code that branches on type properties at compile time:

```cpp
// C++17 if constexpr — replaces 3 template specialisations
template <typename T>
void emitConstant(MCStreamer &Streamer, T Value) {
  if constexpr (std::is_integral_v<T>) {
    Streamer.emitIntValue(static_cast<uint64_t>(Value), sizeof(T));
  } else if constexpr (std::is_same_v<T, float>) {
    Streamer.emitIntValue(llvm::bit_cast<uint32_t>(Value), 4);
  } else if constexpr (std::is_same_v<T, double>) {
    Streamer.emitIntValue(llvm::bit_cast<uint64_t>(Value), 8);
  }
}
```

The `if constexpr` form compiles faster (one instantiation, not three specialisations) and the logic is in one place.

### 183.7.3 Variadic Fold Expressions

Variadic fold expressions (C++17) are used throughout LLVM's range utilities and MLIR's multi-operand builders:

```cpp
// Fold expression — variadic logical AND over a pack
template <typename... Ts>
bool allNonNull(Ts *...ptrs) {
  return (... && (ptrs != nullptr));  // unary left fold over &&
}

// MLIR TypeRange builder — expand type pack into range
template <typename... Types>
TypeRange makeTypeRange(Types... types) {
  SmallVector<Type, sizeof...(Types)> vec{types...};  // pack expansion
  return TypeRange(vec);
}
```

In TableGen-emitted multi-operand builders, the generated C++ uses fold expressions to initialise `OperandRange` from a pack of `Value` arguments.

### 183.7.4 The Future: Reflection Superseding TMP

C++26 static reflection (Section 183.5) largely supersedes hand-written template metaprogramming for boilerplate generation. The key shift: TMP expresses "for each type in a type list, synthesise a function" through recursive template instantiation, which is hard to read, slow to compile, and produces unreadable error messages. Reflection expresses the same computation as `consteval` functions iterating over `std::meta::info` ranges, using ordinary C++ loops, conditionals, and function calls. The reflection computation is a first-class C++ program; the TMP recursion is a Turing-complete but unintended extension of the type system.

For LLVM specifically: the `llvm::isa<>` / `dyn_cast<>` infrastructure, the X-macro dialect registration, the CRTP pass and op base classes, and the TableGen-emitted boilerplate are all targets for eventual replacement by reflection-based generation. The replacement would not change LLVM's runtime semantics — the same `classof` checks, the same virtual dispatch, the same IR structure — but would move the generation from external tools and TMP idioms into standard C++ `consteval` functions.

---

## 183.8 The C++/Rust Boundary

### 183.8.1 Why LLVM Will Not Switch to Rust

LLVM is approximately 4 million lines of C++. The practical constraints against language migration are not primarily technical but operational:

- **ABI stability across the monorepo**: the LLVM C++ ABI — name mangling, vtable layout, inline function inlining decisions — is relied upon by 35+ year-old downstream consumers. A Rust rewrite would break every static-library user, every JNI/JNA binding, and every CMake-based downstream.
- **Developer familiarity**: the ~100 active LLVM committers and ~1000 contributors are overwhelmingly C++ experts. Rust proficiency is rarer, and the MLIR / Clang infrastructure requires deep knowledge of both the language and the codebase — two simultaneous learning curves.
- **The `unsafe` analogy**: C++ with sanitizers, contracts, and careful review achieves a level of safety that, while not Rust's ownership-model safety, is sufficient for a tool that compiles other programs. LLVM's use of AddressSanitizer on CI and Alive2 for misoptimisation detection is a runtime-checking approach to the same problem Rust solves statically.

The honest comparison is: Rust's ownership model prevents a class of bugs that C++ contracts and sanitizers catch only probabilistically. But the migration cost — in engineering time, in downstream disruption, in lost productivity during the transition — exceeds the expected benefit for a codebase with LLVM's scale and maturity.

### 183.8.2 What Contracts + Reflection Would Concretely Improve

Without changing the language, C++26 would deliver three concrete improvements to the LLVM codebase:

1. **API boundary safety via contracts**: `pre`/`post` on `PassWrapper::run()`, `PatternRewriter::replaceOp()`, `IRBuilder::CreateAdd()` would catch API misuse in `enforce` mode during testing, replacing ad hoc `assert` calls that either abort immediately or do nothing in release builds.

2. **Elimination of TableGen for boilerplate**: P2996 reflection can synthesise the `getNumOperands()`, `getResult(unsigned)`, `getAttr<T>()`, `verifyOp()`, and dialect registration code that `.td` files currently generate via an external code generator. This would collapse the two-step (TableGen → compile) build into a single compilation, reducing both build latency and the surface area of the TableGen DSL that new contributors must learn.

3. **Cleaner visitor dispatch via pattern matching**: the `llvm::TypeSwitch`, `dyn_cast` chains, and Clang's `RecursiveASTVisitor` CRTP all solve variations of "dispatch on dynamic type, bind to typed variable." Pattern matching `inspect` is a single, uniform, exhaustiveness-checked mechanism for all three.

### 183.8.3 Rust `unsafe` vs C++ Contracts

The two mechanisms are orthogonal. Rust's `unsafe` is a **scope marker** that shifts the burden of proof: inside `unsafe {}`, the programmer takes responsibility for invariants the borrow checker cannot enforce (raw pointer arithmetic, FFI, manually-managed lifetimes). The `unsafe` block is a narrowing of scope — it is intentionally small and auditable.

C++ contracts are **assertion-at-boundary** mechanisms. They state properties that must hold when a function is called or returns. They do not narrow scope or shift language semantics; they add checkable preconditions to existing function calls. In Rust terms, contracts would be useful on `unsafe fn` boundaries — precisely the places where the borrow checker's guarantees end and programmer-enforced invariants begin. C++26 contracts on LLVM's API surface are the C++ equivalent of `#[requires]`/`#[ensures]` on Rust's `unsafe fn` in Kani ([Chapter 181](../part-26-ecosystem-frontiers/ch181-formal-verification-in-practice.md)).

### 183.8.4 The Co-Evolution Path

Rust's LLVM bindings ([Chapter 177 — rustc: Architecture, MIR, and Codegen Backends](../part-26-ecosystem-frontiers/ch177-rustc-architecture-mir-codegen-backends.md)) mean that improvements to the C++ LLVM API directly benefit Rust callers. A C++26-era LLVM where:

- Pass APIs carry `pre`/`post` contracts visible to tools that process the LLVM headers (IDEs, linters, Clang Static Analyzer with contract-aware checkers)
- Op boilerplate is generated by `consteval` reflection rather than TableGen, reducing the build dependency graph
- TypeSwitch is replaced by exhaustiveness-checked pattern matching

would be safer and more composable for both C++ and Rust callers — because the safety properties live in the API declaration itself, not in comments or out-of-band documentation.

The two languages are converging on the same insight from different directions. Rust reaches safety through the type system enforced at compile time. C++ reaches safety incrementally through contracts enforced at runtime in test builds, through reflection eliminating error-prone boilerplate, and through concepts making API requirements explicit in signatures. Neither path is complete; both move in the same direction. For the compiler developer who must maintain LLVM, understand its C++ patterns, and potentially contribute to its evolution, fluency in both paths is the practical preparation for the next decade of compiler infrastructure.

---

## Chapter Summary

- LLVM compiles with `-fno-exceptions -fno-rtti` across all library code. The vocabulary types that replace STL — `SmallVector`, `StringRef`, `ArrayRef`, `DenseMap`, `llvm::isa<>`/`cast<>`/`dyn_cast<>` — are required knowledge for any LLVM contributor. The canonical reference is the [LLVM Coding Standards](https://llvm.org/docs/CodingStandards.html).

- The C++ standard baseline was C++17 as of LLVM 16, with C++20 features landing selectively from LLVM 17+. The trajectory is toward a C++20 baseline in the near term and C++23 afterward; each transition follows an RFC process gated on CI toolchain minimums.

- **C++20 in active use**: concepts replacing SFINAE in ADT type traits; `std::span` entering new JIT APIs alongside `ArrayRef`; `[[likely]]`/`[[unlikely]]` on hot dispatch paths; `constinit`/`consteval` for globals and compile-time computation; `std::bit_cast` replacing `memcpy`-based type punning in `llvm/ADT/bit.h`; designated initialisers for target descriptor aggregates.

- **C++23 key features**: `std::expected<T,E>` provides monadic error chaining that `llvm::Expected<T>` lacks; `std::mdspan` maps directly to MLIR MemRef buffer descriptors with `layout_right`/`layout_left`/`layout_stride` policies; `std::flat_map`/`std::flat_set` match the performance profile of `SmallDenseMap` for small attribute maps; deducing `this` (P0847) eliminates CRTP in `PassWrapper`, `VisitorBase`, and similar infrastructure; `std::ranges` views enable lazy, composable IR iteration.

- **C++26 Contracts (P2900)**: `pre`, `post`, and `contract_assert` add runtime-checked API boundary assertions with four evaluation semantics (`ignore`, `observe`, `enforce`, `quick_enforce`) selected per build. Clang 22 provides `-fcontracts` as an experimental flag. Contracts complement (not replace) `llvm::verifyFunction`, `AssertingVH`, and `LLVM_DEBUG`. They are runtime checks, not static proofs — in contrast to Dafny/Verus which discharge pre/postconditions statically via SMT.

- **C++26 Static Reflection (P2996)**: `^T` reflects a type to `std::meta::info`; `[:r:]` splices it back. Three direct applications to LLVM: replacing X-macro dialect registration with reflection-iterated type lists; synthesising TableGen ODS boilerplate (getOperand, getResult, verifyOp) from annotated C++ structs; generating `isa<>`/`dyn_cast<>` dispatch from reflected subclass registries. Available experimentally via EDG reference implementation and the Clang P2996 branch. Differs from Rust proc-macros in operating on fully-resolved types rather than token streams.

- **C++26 Pattern Matching (P2688/P1371)**: `inspect(v) { <T> x => expr; ... }` replaces cascaded `dyn_cast` chains and `llvm::TypeSwitch` with exhaustiveness-checked dispatch. A C++29 candidate if not finalized for C++26; key open questions are exhaustiveness for open class hierarchies and binding value categories. The exhaustiveness guarantee is the core advantage over existing Clang/LLVM dispatch patterns.

- **TMP modernisation**: `if constexpr` replacing multi-way template specialisations (C++17, widely used), variadic fold expressions in range utilities and multi-operand builders, concepts replacing ad hoc SFINAE. C++26 reflection largely supersedes TMP for boilerplate generation, moving the computation from recursive template instantiation to readable `consteval` functions.

- **The C++/Rust boundary**: LLVM will not migrate to Rust due to ABI stability, developer familiarity, and migration cost. Contracts + reflection + pattern matching address the same safety and boilerplate problems incrementally within C++. Rust's LLVM bindings mean C++ API improvements benefit Rust callers. The two ecosystems co-evolve: Rust enforces safety through the type system at compile time; C++ approximates it through contracts at test time and through reflection eliminating error-prone manual boilerplate.

@copyright jreuben11
