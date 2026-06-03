# Chapter 38b — C23: Clang's Implementation of the C Standard

*Part V — Clang Internals: Frontend Pipeline*

C23 — formally ISO/IEC 9899:2024 — is the first major revision to the C standard since C11 and the largest since C99 introduced VLAs and compound literals. Where C11 addressed threads and atomics as largely optional annexes, C23 makes a sweeping pass through the core language: it removes deprecated features, regularizes existing GNU extensions into the standard, introduces genuinely new types and syntax, and eliminates numerous categories of undefined behavior that have plagued correct C code for decades. Clang 17 began tracking C23 as it evolved through WG14 drafts; Clang 19 completed the core feature set; Clang 22 ships with full conformance at `-std=c23` and `-std=gnu23`. For embedded systems firmware, OS kernels, cryptographic libraries, and any codebase where C++ remains inappropriate, C23 is the production target — and understanding how Clang implements its new features at the AST, Sema, and IR levels is essential for both compiler users and contributors.

---

## Table of Contents

- [38b.1 Overview of C23](#38b1-overview-of-c23)
  - [38b.1.1 WG14 Timeline and Clang Support Status](#38b11-wg14-timeline-and-clang-support-status)
  - [38b.1.2 Key Design Principles](#38b12-key-design-principles)
  - [38b.1.3 Additional C23 Language Changes](#38b13-additional-c23-language-changes)
- [38b.2 `_BitInt`: Arbitrary-Width Integer Types](#38b2-_bitint-arbitrary-width-integer-types)
  - [38b.2.1 Syntax and Semantics](#38b21-syntax-and-semantics)
  - [38b.2.2 Clang AST Representation](#38b22-clang-ast-representation)
  - [38b.2.3 LLVM IR Lowering](#38b23-llvm-ir-lowering)
  - [38b.2.4 ABI Considerations](#38b24-abi-considerations)
  - [38b.2.5 Code Example: 128-Bit `_BitInt` Arithmetic](#38b25-code-example-128-bit-_bitint-arithmetic)
  - [38b.2.6 Cryptographic and Hardware-Register Use Cases](#38b26-cryptographic-and-hardware-register-use-cases)
- [38b.3 `nullptr` and `nullptr_t` in C](#38b3-nullptr-and-nullptr_t-in-c)
  - [38b.3.1 Language-Level Semantics](#38b31-language-level-semantics)
  - [38b.3.2 Clang Implementation](#38b32-clang-implementation)
  - [38b.3.3 Contrast with C++ `nullptr`](#38b33-contrast-with-c-nullptr)
  - [38b.3.4 Interaction with Function Pointers](#38b34-interaction-with-function-pointers)
- [38b.4 `constexpr` Variables in C23](#38b4-constexpr-variables-in-c23)
  - [38b.4.1 Scope and Restrictions](#38b41-scope-and-restrictions)
  - [38b.4.2 Clang Implementation: `ConstantExpressionEvaluator`](#38b42-clang-implementation-constantexpressionevaluator)
  - [38b.4.3 Interaction with `static_assert` and Array Dimensions](#38b43-interaction-with-static_assert-and-array-dimensions)
- [38b.5 `typeof` and `typeof_unqual`](#38b5-typeof-and-typeof_unqual)
  - [38b.5.1 Standardization of a GNU Extension](#38b51-standardization-of-a-gnu-extension)
  - [38b.5.2 AST Nodes and Interaction with `_Generic`](#38b52-ast-nodes-and-interaction-with-_generic)
- [38b.6 The `#embed` Preprocessor Directive](#38b6-the-embed-preprocessor-directive)
  - [38b.6.1 Syntax and Parameters](#38b61-syntax-and-parameters)
  - [38b.6.2 Clang Preprocessor Implementation](#38b62-clang-preprocessor-implementation)
  - [38b.6.3 IR Result and Use Cases](#38b63-ir-result-and-use-cases)
- [38b.7 Fixed-Width Integer Literals: `wb` and `uwb` Suffixes](#38b7-fixed-width-integer-literals-wb-and-uwb-suffixes)
- [38b.8 Relaxed `va_start`](#38b8-relaxed-va_start)
- [38b.9 `-fstrict-flex-arrays`](#38b9--fstrict-flex-arrays)
  - [38b.9.1 Background and Motivation](#38b91-background-and-motivation)
  - [38b.9.2 Flag Levels and Their Effects](#38b92-flag-levels-and-their-effects)
  - [38b.9.3 Impact on Sanitizers and FORTIFY_SOURCE](#38b93-impact-on-sanitizers-and-fortify_source)
- [38b.10 Interaction with LLVM IR and Embedded Toolchains](#38b10-interaction-with-llvm-ir-and-embedded-toolchains)
  - [38b.10.1 New IR Patterns from C23 Features](#38b101-new-ir-patterns-from-c23-features)
  - [38b.10.2 Embedded Toolchain Considerations](#38b102-embedded-toolchain-considerations)
- [38b.11 Chapter Summary](#38b11-chapter-summary)

---

## 38b.1 Overview of C23

### 38b.1.1 WG14 Timeline and Clang Support Status

WG14, the ISO working group responsible for the C standard, chartered C23 in 2016 under the project number N2310. The standard completed its ballot cycle in late 2023 and was published as ISO/IEC 9899:2024 in the first quarter of 2024 — though it carries the internal revision year "C23" in all implementation literature. The defining macro for conforming implementations is `__STDC_VERSION__`, which evaluates to `202311L` under C23.

Clang tracks WG14 through its `LangStandard` enumeration defined in [`clang/include/clang/Basic/LangStandard.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/LangStandard.h). The relevant entries are:

```
lang_c23    → -std=c23, -std=c2x  (c2x retained as alias)
lang_gnu23  → -std=gnu23, -std=gnu2x
```

Feature test macros follow the C23 `<stdbit.h>` / `<stdckdint.h>` convention. Individual features can be probed with `__has_feature` and `__has_extension`:

```c
#if __has_feature(c_bitint)
// _BitInt is available and enabled
#endif
#if __has_extension(c_bitint)
// _BitInt is available as an extension (possibly in older -std modes)
#endif
```

The Clang C23 feature delivery timeline:

| Clang Version | C23 Features Added |
|--------------|-------------------|
| 17 | `_BitInt`, `nullptr`, `typeof`, `typeof_unqual`, `[[nodiscard]]`/`[[deprecated]]`/`[[fallthrough]]` attributes |
| 18 | `#embed` (experimental), `<stdbit.h>`, `<stdckdint.h>` |
| 19 | `constexpr` variables, relaxed `va_start`, `wb`/`uwb` suffixes, `#embed` (conformant) |
| 20–21 | Defect reports and edge-case conformance fixes |
| 22 | Full C23 conformance; `-std=c23` is the production flag |

### 38b.1.2 Key Design Principles

C23's design is shaped by three goals that cut across every feature:

**Eliminate UB where possible.** Signed integer overflow, left-shifting into the sign bit, and certain null pointer operations remain UB for performance reasons, but C23 removes the UB on several previously-dangerous constructs: `{}` as a valid initializer (equivalent to zero-initialization), omitting the trailing `void` in a zero-parameter function prototype (now well-defined), and using `unreachable()` from `<stddef.h>` instead of relying on the optimizer to infer it.

**Regularize GNU extensions.** `typeof`, `__attribute__((deprecated))` via `[[deprecated]]`, zero-length struct member initializers via `{}`, and several other GCC/Clang extensions are standardized with minor spelling changes, so existing code often requires only a `-std=c23` flag change rather than source modification.

**Better integer types.** `_BitInt(N)`, checked integer arithmetic via `<stdckdint.h>`, and the portable bit-manipulation functions in `<stdbit.h>` (population count, leading/trailing zeros, rotate left/right) address decades of complaints about C's integer model.

### 38b.1.3 Additional C23 Language Changes

Several C23 changes are important to Clang users but are not the focus of individual sections:

**Standard attributes.** C23 standardizes the `[[attribute]]` syntax (double square brackets) for `[[deprecated("msg")]]`, `[[fallthrough]]`, `[[nodiscard]]`, `[[maybe_unused]]`, `[[noreturn]]`, and `[[unsequenced]]`/`[[reproducible]]`. Clang's attribute parser in [`clang/lib/Parse/ParseDecl.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Parse/ParseDecl.cpp) has supported `[[...]]` syntax since Clang 3.9; C23 simply makes these portable. The `[[unsequenced]]` and `[[reproducible]]` attributes (marking functions as pure/const in portable C) correspond to Clang's `__attribute__((pure))` and `__attribute__((const))`.

**`bool` as a keyword.** C23 makes `bool`, `true`, and `false` keywords, ending the need to `#include <stdbool.h>`. The `_Bool` spelling remains available. Clang's lexer maps all three from the keyword table when `LangOpts.C23` is set.

**`static_assert` without a message.** C11 `_Static_assert` and C23 `static_assert` (now a keyword, not a macro) both accept a single-argument form: `static_assert(expr)`. Clang's parser was updated to accept this form in all C23 contexts.

**Checked arithmetic headers.** `<stdckdint.h>` provides `ckd_add`, `ckd_sub`, `ckd_mul` — generic functions that detect overflow without UB. Clang implements these as `__builtin_add_overflow` etc. under the hood, which lower to LLVM's `@llvm.sadd.with.overflow` / `@llvm.uadd.with.overflow` intrinsics.

**`<stdbit.h>` bit utilities.** `stdc_leading_zeros`, `stdc_bit_width`, `stdc_popcount`, and related functions. Clang maps these to `__builtin_clz`, `__builtin_popcount`, and friends, which lower to target-specific instructions (`lzcnt`, `popcnt` on x86-64; `clz`, `rbit` on AArch64).

---

## 38b.2 `_BitInt`: Arbitrary-Width Integer Types

### 38b.2.1 Syntax and Semantics

`_BitInt(N)` declares an integer with exactly N bits of value representation plus (for signed types) one sign bit. N must be a positive integer constant expression; `unsigned _BitInt(N)` omits the sign bit and uses all N bits for magnitude. The implementation-defined upper bound is exposed as `BITINT_MAXWIDTH` in `<limits.h>`; for Clang on x86-64 and AArch64, the limit is 128; targets with software-emulated wide integers may support higher values.

```c
#include <stddef.h>
#include <limits.h>

_BitInt(7)          x = 63;       // signed 7-bit: range [-64, 63]
unsigned _BitInt(8) u = 255U;     // unsigned 8-bit: range [0, 255]
_BitInt(24)         r;            // signed 24-bit — hardware register model
_BitInt(1)          b;            // range {0, -1} — only 0 and -1 representable
_BitInt(BITINT_MAXWIDTH) big;     // widest supported
```

Unlike historical fixed-width types that promote to `int` in arithmetic expressions, `_BitInt` operands do **not** promote. The result of arithmetic on `_BitInt(N)` operands is `_BitInt(N)` (or the wider type if the operands differ in width, following the usual arithmetic conversions extended for `_BitInt`). Specifically:

- If both operands are `_BitInt`, the result is `_BitInt` of the larger width.
- If one operand is `_BitInt` and the other is a standard integer type, the `_BitInt` is widened (or the integer is narrowed) to produce a common type.
- Division and modulo for `_BitInt` follow the same truncation-toward-zero semantics as `int`.
- Overflow is defined as two's-complement wrapping for unsigned `_BitInt` and remains undefined for signed `_BitInt` (consistent with `int`).

### 38b.2.2 Clang AST Representation

Clang models `_BitInt(N)` with `BitIntType`, declared in [`clang/include/clang/AST/Type.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Type.h). The class stores two fields: `unsigned NumBits` and `bool IsUnsigned`. Type uniquing is handled through `ASTContext::getBitIntType(bool Unsigned, unsigned NumBits)`, which looks up or inserts the canonical type into the fold-set keyed on `(IsUnsigned, NumBits)`.

```cpp
// clang/include/clang/AST/Type.h (simplified)
class BitIntType : public Type, public llvm::FoldingSetNode {
  unsigned NumBits : 24;
  unsigned IsUnsigned : 1;
public:
  bool isUnsigned() const { return IsUnsigned; }
  unsigned getNumBits() const { return NumBits; }
  static void Profile(llvm::FoldingSetNodeID &ID,
                      bool IsUnsigned, unsigned NumBits);
  // ...
};
```

During Sema, `SemaType::BuildBitIntType` (in [`clang/lib/Sema/SemaType.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaType.cpp)) validates that N is a valid constant, that it is at most `BITINT_MAXWIDTH`, and that the bit-width is at least 1 for unsigned or 2 for signed (a 1-bit signed `_BitInt` can only represent 0 and -1, which is permitted). The diagnostic `err_bit_int_bad_size` is emitted for out-of-range N; `warn_bit_int_one_bit_signed` is a warning for the edge case of `_BitInt(1)`.

The usual arithmetic conversions extended for `_BitInt` are implemented in `Sema::UsualArithmeticConversions` in `SemaExpr.cpp`. When a `_BitInt(M)` and a `_BitInt(N)` meet with M ≠ N, the narrower is extended to the wider; sign is determined by whether either operand is unsigned (mirroring the usual unsigned-dominates rule).

Literal values of `_BitInt` type use the `wb` and `uwb` suffixes discussed in section 38b.7; for now, integer literals without those suffixes are implicitly converted via `IntegralCast` using the same truncation/extension rules as other integer conversions, and Clang will diagnose truncation with `-Wimplicit-int-conversion`.

### 38b.2.3 LLVM IR Lowering

LLVM's type system supports arbitrary-width integers as `iN` for any positive N, making the IR translation from `_BitInt(N)` direct:

| C23 Type              | LLVM IR Type |
|-----------------------|-------------|
| `_BitInt(1)`          | `i1`        |
| `_BitInt(7)`          | `i7`        |
| `unsigned _BitInt(8)` | `i8`        |
| `_BitInt(24)`         | `i24`       |
| `_BitInt(64)`         | `i64`       |
| `_BitInt(128)`        | `i128`      |

This translation is performed in the general CodeGen path through `CodeGenTypes::ConvertType` in [`clang/lib/CodeGen/CodeGenTypes.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CodeGenTypes.cpp), which handles `BitIntType` by calling `llvm::IntegerType::get(getLLVMContext(), T->getNumBits())`. For more detail on LLVM's integer type system, see [Chapter 17 — The Type System](../part-04-llvm-ir/ch17-the-type-system.md).

Arithmetic operations on non-power-of-two widths such as `i24` or `i37` are fully legal in LLVM IR. The backend legalizes them during instruction selection; the exact strategy depends on the target's `LegalizeTypes` configuration:

- **x86-64**: `i24` legalizes to `i32`; arithmetic is performed on 32-bit registers with appropriate sign-/zero-extension after loads and before stores. `i24` stores use a `trunc` followed by a 16-bit store and an 8-bit store (the backend's generic type legalization for non-power-of-two widths).
- **AArch64**: similarly widens to the next larger machine integer; the register allocator uses 32-bit or 64-bit registers.
- **RISC-V**: same approach; the `Zbb` extension's bit-manipulation instructions can be exploited post-legalization by the backend.

The optimizer sees the unlegalized `i24` type across all mid-level passes (InstCombine, GVN, LICM) and can reason about modular arithmetic over the 24-bit range, enabling value-range analysis and loop-strength reduction that would be incorrect if the value were silently promoted to 32 bits.

### 38b.2.4 ABI Considerations

C23 does not mandate a specific ABI for `_BitInt` parameters; it defers to the platform ABI. The LLVM community documented a `_BitInt` ABI extension for the Itanium C++ ABI (used by most ELF targets) in the ABI supplement for Clang 17. Under this scheme:

| Bit width N | Passed as |
|-------------|-----------|
| N ≤ 8 | 8-bit register slot (same as `int8_t`) |
| N ≤ 16 | 16-bit slot |
| N ≤ 32 | 32-bit slot |
| N ≤ 64 | 64-bit slot |
| N ≤ 128 | Two 64-bit slots (same as `__int128`) |
| N > 128 | Indirect (pointer to caller-allocated buffer) |

For signed types, the value is sign-extended into the slot; for unsigned, it is zero-extended. This means the ABI type for `_BitInt(24)` as a function argument is an `i32` with the upper 8 bits sign-extended — the caller and callee must both agree on this encoding, which is guaranteed when both are compiled with Clang 17+.

The Clang ABI implementation for each target lives in `TargetInfo::classifyArgumentType` and `TargetInfo::classifyReturnType` in the per-target files under [`clang/lib/CodeGen/Targets/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/Targets/). For example, `X86_64ABIInfo::classifyArgumentType` in `X86.cpp` handles `BitIntType` by mapping it to the `INTEGER` class for N ≤ 64 or a two-element `INTEGER` pair for N ≤ 128.

Embedded targets (ARM Cortex-M, RISC-V bare-metal) follow the same rules but with narrower register widths; on ARMv7-M, a `_BitInt(64)` argument occupies two 32-bit registers `r0:r1`.

### 38b.2.5 Code Example: 128-Bit `_BitInt` Arithmetic

```c
// bitint128.c — compile with: clang -std=c23 -O1 -emit-llvm -S -o bitint128.ll bitint128.c

_BitInt(128) multiply(_BitInt(128) a, _BitInt(128) b) {
    return a * b;
}

unsigned _BitInt(128) umask(unsigned _BitInt(128) x) {
    return x & 0xDEADBEEFCAFEBABEUWB;
}

// Demonstrates no-promotion: result stays _BitInt(24)
_BitInt(24) add24(_BitInt(24) a, _BitInt(24) b) {
    return a + b;   // result is _BitInt(24), not int
}
```

The resulting LLVM IR for `multiply` and `add24` (abbreviated, with opaque pointers):

```llvm
define i128 @multiply(i128 %a, i128 %b) {
entry:
  %r = mul nsw i128 %a, %b
  ret i128 %r
}

define i128 @umask(i128 %x) {
entry:
  ; 0xDEADBEEFCAFEBABE zero-extended to i128
  %r = and i128 %x, 16045690984833335998
  ret i128 %r
}

define i24 @add24(i24 %a, i24 %b) {
entry:
  %r = add nsw i24 %a, %b
  ret i24 %r
}
```

The `nsw` (no signed wrap) flag on `mul` and `add` signals to the optimizer that signed overflow is undefined behavior, enabling transformations like reassociation and strength reduction. For `add24`, the optimizer knows the result fits in 24 bits; no truncation instruction is emitted because `i24` is the native arithmetic type here.

### 38b.2.6 Cryptographic and Hardware-Register Use Cases

`_BitInt` fills two important niches in systems programming:

**Cryptographic code.** Elliptic-curve cryptography over prime fields like P-256 (256-bit) or Curve25519 (255-bit) traditionally requires either multi-limb arrays of 64-bit integers (with hand-written carry chains) or library-specific bignum types. With `_BitInt(256)`, the C source can express field arithmetic naturally, and the optimizer can apply scalar-evolution and loop unrolling. The backend then legalizes to the available word size. This is not a silver bullet — specialized crypto libraries (e.g., HACL*, Botan) use intrinsics and hand-tuned assembler — but it is a major improvement for portable implementations.

**Hardware register models.** Embedded peripheral registers are commonly 12-bit ADC values, 10-bit DAC values, or 20-bit timer counts. A `_BitInt(12)` field accurately models the hardware range; value-range analysis in LLVM can eliminate redundant bounds checks that would be needed with `uint16_t`. The improved precision also helps UBSan: overflow of a `_BitInt(12)` variable is detected immediately rather than requiring the programmer to mask after every operation.

---

## 38b.3 `nullptr` and `nullptr_t` in C

### 38b.3.1 Language-Level Semantics

C23 introduces `nullptr` as a keyword of type `nullptr_t`. Its value is the null pointer constant. `nullptr_t` is defined in `<stddef.h>` as a distinct type — not a typedef for `void*` or an integer. Key behavioral points:

- `nullptr` implicitly converts to any pointer type and to `bool` (as `false`).
- `nullptr` does **not** implicitly convert to an integer type (unlike `NULL`, which is typically `0` or `(void*)0` and can be compared to integers).
- `nullptr` does **not** implicitly convert to `void*` without a cast (this differs from C++, where `nullptr` does convert to `void*`).
- Two values of `nullptr_t` compare equal; `nullptr_t` is not an arithmetic type.
- `sizeof(nullptr_t)` is implementation-defined but equal to `sizeof(void*)` on all Clang targets.

This addresses long-standing type-safety issues with `NULL`: passing `NULL` to a variadic function expecting a sentinel pointer compiles with warnings in C11 but no hard error; `nullptr` produces a `-Wincompatible-pointer-types` error when used where an integer is expected.

### 38b.3.2 Clang Implementation

Clang represents `nullptr` with `CXXNullPtrLiteralExpr` in the AST — the same node used for C++ `nullptr`. The type of this expression is `Context.NullPtrTy`, which is `BuiltinType::NullPtr`. This reuse is intentional; the semantics differ only in what conversions are allowed, and those are enforced by `Sema::CheckAssignmentConstraints` and `Sema::TryImplicitConversion`.

In `<stddef.h>` (Clang's built-in header at [`clang/lib/Headers/stddef.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Headers/stddef.h)), the C23 path adds:

```c
#if defined(__STDC_VERSION__) && __STDC_VERSION__ >= 202311L
typedef typeof(nullptr) nullptr_t;
#endif
```

The keyword `nullptr` itself is recognized by the lexer when `LangOpts.C23` is set, via the keyword table in [`clang/include/clang/Basic/TokenKinds.def`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/TokenKinds.def):

```
KEYWORD(nullptr, KEYCXX11|KEYC23)
```

The parser's `ParseCastExpression` handles `tok::kw_nullptr` identically to C++, constructing a `CXXNullPtrLiteralExpr`. IR lowering: `nullptr` reduces to a null pointer constant (`ptr null` in opaque-pointer IR) through the standard `CodeGenModule::EmitNullValue` path.

Diagnostic behaviour: assigning `nullptr` to an `int` variable produces `error: assigning to 'int' from incompatible type 'nullptr_t'`. Passing `nullptr` to `printf`'s `%d` format produces a warning from the format-string checker, because `nullptr_t` is not an integer type.

### 38b.3.3 Contrast with C++ `nullptr`

The behavioral differences from C++ matter for mixed-language headers:

| Behaviour | C23 `nullptr` | C++ `nullptr` |
|-----------|--------------|--------------|
| Type | `nullptr_t` | `std::nullptr_t` |
| Converts to `void*` | No (explicit cast required) | Yes |
| Converts to `bool` | Yes (`false`) | Yes (`false`) |
| Converts to integer | No | No |
| Usable in `_Generic` | Yes | N/A |
| `sizeof` | `sizeof(void*)` | `sizeof(void*)` (impl-defined) |

For C/C++ shared headers, the portable pattern is:

```c
#ifndef __cplusplus
#  if __STDC_VERSION__ >= 202311L
     // nullptr is a keyword; nullptr_t from <stddef.h>
#  else
#    define nullptr ((void*)0)
     typedef void* nullptr_t;   // approximation only; lacks type safety
#  endif
#endif
```

### 38b.3.4 Interaction with Function Pointers

In C (both C11 and C23), converting between data pointers and function pointers is not guaranteed to be meaningful (POSIX extends this guarantee; the C standard does not). `nullptr` is expressly a null pointer constant that converts to any **pointer** type, including function pointer types:

```c
typedef void (*callback_t)(int);
callback_t cb = nullptr;       // valid: nullptr converts to function pointer
if (cb != nullptr) { cb(42); } // comparison is well-formed

// Explicit cast needed for void*:
void *p = (void*)nullptr;      // C23: explicit cast required (unlike C++)
```

This is a deliberate asymmetry from C++. In C, there is no uniform `void*` ↔ function-pointer conversion, so it is cleaner not to have `nullptr` implicitly produce `void*`. Clang enforces this with `Sema::CheckImplicitConversion` issuing `diag::warn_impcast_null_pointer_to_integer` for the `void*` case.

---

## 38b.4 `constexpr` Variables in C23

### 38b.4.1 Scope and Restrictions

C23 adopts the `constexpr` keyword, but its scope is deliberately narrower than C++ `constexpr`. In C23:

- `constexpr` applies **only to object declarations** (variables), not to functions. There are no constexpr functions in C23.
- A `constexpr` variable must have a type that is a scalar (integer, floating-point, pointer, enumeration) or an aggregate/union whose members recursively satisfy scalar-type requirements.
- The initializer must be a **constant expression** as defined in C23 §6.6 — which allows integer constant expressions, floating-point constant expressions, address constant expressions, and arithmetic thereof.
- `constexpr` variables have `static` storage duration if declared at file scope, or automatic storage duration at block scope (the initializer must still evaluate at compile time).
- The `constexpr` specifier implies `const` for the declared object.

```c
constexpr int kBufSize = 4096;               // file scope, static storage
constexpr double kPi = 3.14159265358979323846;
constexpr unsigned kFlags = 0x0001u | 0x0040u;

void f(void) {
    constexpr int kLocal = 128;              // block scope, automatic
    char buf[kLocal];                        // VLA replaced by true constant
    static_assert(kLocal > 0);              // also valid in C23
}

// NOT valid in C23 (function constexpr is C++ only):
// constexpr int add(int a, int b) { return a + b; }

// Valid — struct with scalar members:
typedef struct { int x; int y; } Point;
constexpr Point kOrigin = { .x = 0, .y = 0 };
```

The primary practical benefit is replacing the brittle `static const int kBufSize = 4096;` idiom, which was technically a read-only object rather than an integer constant expression in C11 (you cannot use a `static const int` as an array dimension in pre-C23 C without VLAs). C23 `constexpr int` is a true integer constant expression and can be used wherever one is required.

### 38b.4.2 Clang Implementation: `ConstantExpressionEvaluator`

Clang's constant evaluator, rooted in [`clang/lib/AST/ExprConstant.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/AST/ExprConstant.cpp), is shared between C++ and C23 `constexpr` evaluation. The `EvalInfo` context carries a `ConstantContext` field that distinguishes evaluation modes; C23 uses `EvalContext::C23ConstantExpr`.

The entry point for C23 variable initialization is `Expr::EvaluateAsConstantExpr`, which invokes `EvaluateInPlace` with the C23 mode. The evaluator enforces the C23 restrictions that distinguish it from C++:

- No function calls are permitted (no constexpr functions in C23).
- No VLA-related expressions.
- No `_Alignof` of VLA types.
- Floating-point operations that would raise exceptions (division by zero, overflow) are disallowed.

Sema marks a `VarDecl` with `isConstexpr() == true` after verifying the initializer evaluates successfully via `VarDecl::checkInitIsICE`. During CodeGen:

- File-scope `constexpr` variables are emitted as `@` globals with `constant` linkage and a constant initializer via `CGM.EmitConstantInit`.
- Block-scope `constexpr` variables become compile-time-known immediates folded by `CGF.EmitScalarExpr`, or `alloca`s initialized with a constant if their address is taken.

The feature test macro is `__has_feature(c_constexpr)`, which evaluates to `1` in Clang 19+ at `-std=c23`.

### 38b.4.3 Interaction with `static_assert` and Array Dimensions

The C23 `static_assert` keyword (no longer a macro; `_Static_assert` is the alternate spelling) accepts both one-argument and two-argument forms:

```c
constexpr int kMaxNodes = 1024;
static_assert(kMaxNodes > 0, "node count must be positive");
static_assert(kMaxNodes <= 65536);   // C23: message is optional
```

Array dimensions require an integer constant expression; C23 `constexpr` integer variables qualify, eliminating the need for object-like macros:

```c
// C11 — required preprocessor macro:
#define MAX_ITEMS 256
static char pool[MAX_ITEMS];

// C23 — constexpr variable, full type safety:
constexpr int kMaxItems = 256;
static char pool[kMaxItems];   // valid: kMaxItems is an integer constant expr
```

---

## 38b.5 `typeof` and `typeof_unqual`

### 38b.5.1 Standardization of a GNU Extension

GCC introduced `__typeof__` (and the alias `__typeof`) in the 1980s; Clang has supported it since its earliest releases. C23 standardizes this facility under the spellings `typeof` and `typeof_unqual`, making `__typeof__` an alias for the same functionality rather than a GNU-only extension.

`typeof(expr)` yields the type of the operand expression without evaluating it. `typeof_unqual(expr)` yields the same type with all top-level qualifiers (`const`, `volatile`, `restrict`, `_Atomic`) stripped. Both also accept a type name as the operand:

```c
int x = 42;
typeof(x)        a = 0;   // int
const int       *p;
typeof(p)        q;       // const int *  (pointer itself is unqualified)
typeof_unqual(p) r;       // int *        (top-level type's qualifiers removed)

const int * const cp;
typeof(cp)        s;      // const int * const
typeof_unqual(cp) t;      // const int *    (top-level const stripped from pointer)

typeof(1 + 1.0)  d;      // double (usual arithmetic conversions applied)
```

The `typeof_unqual` semantics for pointer types are subtle. The WG14 specification strips qualifiers from the top-level type being named. For `const int *`, the type is "pointer to `const int`" — the `const` qualifies the pointed-to type, not the pointer itself. So `typeof_unqual(const int *)` yields `int *` (the `const` on `int` is part of the pointed-to type, not the top-level pointer type). For `const int * const`, the top-level `const` qualifies the pointer itself, so `typeof_unqual` strips it to yield `const int *`.

In `-std=c23` mode, `typeof` and `typeof_unqual` are keywords. In `-std=c11` and earlier, Clang accepts them as extensions under `-fgnu-keywords` (the default for non-strict modes) and maps them to `__typeof__`; `-pedantic -std=c11` emits a `pedantic` warning.

### 38b.5.2 AST Nodes and Interaction with `_Generic`

Clang represents these types using `TypeofExprType` (for expression operands) and `TypeofType` (for type-name operands), both declared in [`clang/include/clang/AST/Type.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Type.h). The `typeof_unqual` variant sets the `IsUnqual` flag on these nodes. During canonical type computation, `ASTContext::getTypeOfExprType` and `ASTContext::getTypeOfType` return the canonical form directly; the `TypeofExprType` wrapper is only kept for source fidelity in the non-canonical form used by diagnostics and refactoring tools.

The interaction with `_Generic` is valuable: `typeof` enables writing type-generic macros that work correctly with `_Generic` selection, since the association type must match the expression type exactly after lvalue-to-rvalue conversion:

```c
// Type-preserving swap using typeof:
#define SWAP(a, b) do {         \
    typeof(a) _tmp = (a);       \
    (a) = (b);                  \
    (b) = _tmp;                 \
} while (0)

// Checked-arithmetic generic add using typeof + _Generic:
#define CKD_ADD(a, b, res) _Generic((typeof(a)){0},   \
    int:               ckd_add((res), (a), (b)),      \
    unsigned int:      ckd_add((res), (a), (b)),      \
    long:              ckd_add((res), (a), (b)),       \
    unsigned long:     ckd_add((res), (a), (b))        \
)
```

The `typeof_unqual` variant is particularly useful in macro implementations that need to strip qualifiers for temporary storage:

```c
// Remove const for a mutable temporary copy:
#define COPY_AND_MODIFY(x) do {             \
    typeof_unqual(x) _tmp = (x);            \
    modify(&_tmp);                          \
    /* x unchanged, _tmp has the result */  \
} while (0)
```

---

## 38b.6 The `#embed` Preprocessor Directive

### 38b.6.1 Syntax and Parameters

`#embed` is C23's answer to the decades-old pattern of converting binary blobs to C arrays with tools like `xxd -i` or custom Python scripts. Its syntax is:

```
#embed <resource-path>
#embed "resource-path"  [ embed-parameter ... ]
```

The optional embed parameters are:

| Parameter | Effect |
|-----------|--------|
| `limit(N)` | Include at most N elements (each element is one `unsigned char`-sized integer in [0, UCHAR_MAX]) |
| `prefix(token-sequence)` | Tokens prepended before the embedded data if the resource is non-empty |
| `suffix(token-sequence)` | Tokens appended after the embedded data if the resource is non-empty |
| `if_empty(token-sequence)` | Token sequence used if the resource is empty (replaces the directive entirely) |

An `#embed` directive expands to a comma-separated sequence of integer constant expressions — one per byte of the resource — suitable for use anywhere such a sequence is legal: array initializers, function call arguments, braced initializer lists.

```c
// Embed an entire font file:
static const unsigned char kFont[] = {
#embed "fonts/inconsolata-bold.ttf"
};
// kFont is a complete, const-qualified array with SIZEOF(font-file) elements.

// Embed only the first 64 bytes:
static const unsigned char kHeader[] = {
#embed "firmware.bin" limit(64)
};

// Embed with null terminator appended, handle empty file gracefully:
static const unsigned char kCert[] = {
#embed "server.crt" suffix(, 0x00) if_empty(0x00)
};

// Embed a SPIR-V shader with alignment prefix:
static const uint32_t kShader[] = {
#embed "shader.spv" prefix(/* SPIR-V magic: */ 0x07230203,)
};
```

The standard also defines `__has_embed` as a preprocessor operator (analogous to `__has_include`) that returns one of three values: `__STDC_EMBED_FOUND__` (1), `__STDC_EMBED_EMPTY__` (2), or `__STDC_EMBED_NOT_FOUND__` (0). This enables conditional compilation depending on whether the resource exists and is non-empty.

### 38b.6.2 Clang Preprocessor Implementation

The preprocessor implementation lives in [`clang/lib/Lex/PPDirectives.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Lex/PPDirectives.cpp) (dispatch from `HandleDirective`) and dedicated embed logic in `clang/lib/Lex/PPEmbedParameters.cpp` plus `clang/include/clang/Lex/PPEmbedParameters.h`. The central data structures are:

- **`LexEmbedParameterSequence`** — parses the embed parameter list into an `EmbedParameters` struct containing `std::optional<uint64_t> Limit`, `SmallVector<Token> Prefix`, `SmallVector<Token> Suffix`, and `SmallVector<Token> IfEmpty`.
- **`EmbedAnnotationData`** — stored as an annotation token (`tok::annot_embed`) in the preprocessor's token stream; it holds a `StringRef` into a memory-mapped view of the resource file, plus the parsed parameters.
- **`HandleEmbedDirective`** in `PPDirectives.cpp` — validates parameters, resolves the resource path through the header search (following `<>` vs `""` lookup rules, embed-specific paths from `-fembed-path=`), memory-maps the file via `FileManager`, and produces the annotation token.

**Lazy expansion** is the critical performance property. Unlike a naïve implementation that materializes every byte as a separate pp-token (a 4 MB file would produce 4 million tokens), Clang's preprocessor records the embed as a single `tok::annot_embed` annotation token with a pointer into the mmap'd buffer. Expansion into actual integer tokens happens only when the parser's `ParseInitializerWithPotentialDesignator` (or any other consumer of a token sequence) encounters the annotation, driven by `Preprocessor::HandleAnnotations()`. The expansion produces chunks of tokens on demand, keeping the pp-token stream compact and avoiding blowing the parser's lookahead buffers for megabyte-scale embedded resources.

The `__has_embed` operator is handled in `Preprocessor::EvaluateHasEmbed`, which performs the file lookup, checks the limit parameter to determine emptiness, and returns the appropriate `__STDC_EMBED_*` constant without actually mapping the file content.

### 38b.6.3 IR Result and Use Cases

After preprocessing and parsing, an `#embed` directive in an array initializer becomes a sequence of `IntegerLiteral` AST nodes (each `unsigned char`-valued), which CodeGen lowers to a constant data section. For large arrays, LLVM's `ConstantDataArray` representation stores the raw bytes compactly in the `LLVMContext` as a `StringRef` over a single contiguous buffer, without boxing each element as a separate `ConstantInt` object. The resulting IR and object code are identical to what a manually written `unsigned char kFont[] = {0x00, 0x01, ...}` would produce — the difference is entirely in compile speed and source manageability.

The linker places the resulting bytes in `.rodata` (or its equivalent) and the linker's identical-code-folding pass (`--icf=all` in lld) or `COMDAT` merging can deduplicate identical embedded resources across translation units.

Typical use cases in systems and embedded programming:

| Use Case | Example |
|----------|---------|
| Fonts | Bare-metal GUI firmware without a filesystem |
| X.509 certificates | TLS libraries with embedded root CA store |
| SPIR-V / PTX shaders | GPU compute applications shipping shaders as data |
| WASM modules | Runtimes loading WASM without file I/O |
| Firmware blobs | Device driver initialization sequences |
| FPGA bitstreams | Embedded Linux drivers for FPGA configuration |

Build system integration: `#embed` interacts with dependency tracking. Clang emits a `Makefile` dependency on the embedded file via the standard `-MD`/`-MMD` flags, so `make` and Ninja correctly rebuild when the embedded resource changes.

---

## 38b.7 Fixed-Width Integer Literals: `wb` and `uwb` Suffixes

C23 introduces two new integer literal suffixes to produce `_BitInt` literals without relying on implicit conversion:

- `N wb` — produces a **signed** `_BitInt` literal at the minimum bit width needed to represent N without truncation.
- `N uwb` — produces an **unsigned** `_BitInt` literal at the minimum width.

```c
auto x = 127wb;            // _BitInt(8) — minimum signed width for 127 is 8 bits
auto y = 128wb;            // _BitInt(9) — 128 needs 9 bits signed (would overflow 8-bit signed)
auto z = 0xDEADuwb;        // unsigned _BitInt(16) — minimum for 0xDEAD is 16 unsigned bits
auto big = 0xDEADBEEFuwb;  // unsigned _BitInt(32)
_BitInt(32) w = 17wb;      // implicitly extended from _BitInt(5) to _BitInt(32)
```

The minimum width rule: for `wb` (signed), the bit width is `floor(log2(|value|)) + 2` (enough bits for the value plus sign); for `0wb`, the width is 1. For `uwb` (unsigned), the width is `floor(log2(value)) + 1`; for `0uwb`, the width is 1.

Clang's lexer handles these suffixes in [`clang/lib/Lex/LiteralSupport.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Lex/LiteralSupport.cpp) in the `NumericLiteralParser` class. The suffix scanner recognizes `wb` and `uwb` (case-insensitive per C23) and sets `MicrosoftInteger = false` (to distinguish from Microsoft's `i8`/`i16` suffixes), `IsWidthModifier = true`, and `IsUnsigned` accordingly. After parsing the numeric value into an `llvm::APInt`, the parser computes `BitWidth` as:

- `APInt::getMinSignedBits(value)` for `wb` (returns the minimum number of bits to hold the signed two's-complement representation)
- `APInt::getActiveBits(value)` for `uwb` (returns the position of the highest set bit + 1, which is the minimum unsigned width)

The resulting `IntegerLiteral` AST node has `BitIntType` as its type, and Sema's `ActOnIntegerConstant` accepts this type directly.

At the use site, `_BitInt` literals interact with `_Generic` in a useful way: `17wb` has type `_BitInt(5)`, which is distinct from `int`, `long`, and other standard types, enabling `_Generic` to dispatch on exact-width types.

---

## 38b.8 Relaxed `va_start`

C11 requires `va_start(ap, parm_n)` to name the last declared parameter before the `...` explicitly. The historical reason is that pre-ANSI `varargs.h` implementations used the named parameter's address to locate the variadic arguments on the stack. Modern calling conventions and Clang's implementation via LLVM's `va_list` intrinsics do not use the named parameter at all — the ABI-specified variadic area location is derived from the function's ABI without needing a specific argument's address.

C23 relaxes this: `va_start(ap)` with a single argument is valid. The second argument, when supplied, is still accepted for compatibility but is semantically ignored.

```c
#include <stdarg.h>
#include <stdio.h>

void log_msg(const char *fmt, ...) {
    va_list ap;
    va_start(ap);          // C23: no need to name 'fmt'
    vprintf(fmt, ap);
    va_end(ap);
}

// C11 compatibility shim (if compiling with -std=c11):
void log_msg_c11(const char *fmt, ...) {
    va_list ap;
    va_start(ap, fmt);     // still valid in C23, second arg ignored
    vprintf(fmt, ap);
    va_end(ap);
}
```

Clang's implementation: `__builtin_va_start` is a builtin with arity checking in [`clang/lib/Sema/SemaChecking.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaChecking.cpp) in `SemaBuiltinVAStart`. In C23 mode (`LangOpts.C23`), the check accepts one or two arguments; in earlier modes, exactly two are required and the second must name the last non-variadic parameter. The single-argument form internally resolves the last named parameter by calling `FD->getParamDecl(FD->getNumParams() - 1)` to satisfy the backend's `@llvm.va_start` requirements — the IR produced is identical to the two-argument form.

An edge case: if the function has no named parameters at all (e.g., `void f(...)` — valid in C23 but not C11), then `va_start(ap)` is the only legal form; Clang handles this by emitting `@llvm.va_start` with `ptr undef` as the second argument, which the backend ignores.

---

## 38b.9 `-fstrict-flex-arrays`

### 38b.9.1 Background and Motivation

Flexible array members (FAMs) — `struct Foo { int len; char data[]; };` — are standard C99 and allow a struct to be allocated with a variable-length trailing array. The pattern is common in Linux kernel data structures, network protocol headers, and embedded frame formats. However, many C codebases predate C99 FAMs and use approximations:

```c
// These are NOT standard FAMs, but have historically been used as such:
struct Buf0 { int len; char data[0]; };  // GNU extension (zero-length array)
struct Buf1 { int len; char data[1]; };  // ISO C trick (accessing beyond [0] is UB)
```

The Linux kernel uses both patterns heavily. Sanitizers and `__builtin_object_size` need to know which trailing arrays are "real" FAMs to avoid false-positive bounds violations when correct code accesses `data[2]` in a `Buf0` or `Buf1` struct.

### 38b.9.2 Flag Levels and Their Effects

`-fstrict-flex-arrays=N` controls this classification:

| Level | Treatment of trailing arrays |
|-------|------------------------------|
| `0` (default) | Any trailing array — including `[0]` and `[1]` and even multi-element arrays — treated as a FAM for object-size purposes |
| `1` | Trailing `[0]` and `[1]` treated as FAMs; other sizes (e.g., `[2]`) are not |
| `2` | Only true C99 FAMs (`[]`) treated as FAMs; `[0]` and `[1]` are not |
| `3` | Strictest: only true FAMs; `[0]` is rejected as a GNU extension even in `-std=gnu23`; helps identify legacy patterns |

C23 itself does not add new rules about FAMs, but it tightens the wording around object representation in ways that make level-2 behavior the most conformant. The Linux kernel now builds with `-fstrict-flex-arrays=3` in sanitizer configurations to identify all non-conformant FAM approximations.

### 38b.9.3 Impact on Sanitizers and FORTIFY_SOURCE

The flag affects three security-relevant mechanisms:

**`__builtin_object_size(ptr, 1)`** — used by glibc's `FORTIFY_SOURCE` to know the maximum safe buffer size for `memcpy`, `strcpy`, etc. At level 0, a `struct { int len; char data[1]; }` allocated with `malloc(sizeof(S) + N)` is treated as having an object size of `sizeof(S) + N`; at level 2, `data` has size 1 (the declared array bound), so FORTIFY would refuse to copy more than 1 byte into it.

**`__builtin_dynamic_object_size(ptr, 1)`** — the runtime variant, lowered to LLVM's `@llvm.objectsize` intrinsic with a `dynamic` flag. The `counted_by` attribute (see [Chapter 68 — Security Attributes and FORTIFY](../../appendices/ch68-security-attrs.md)) adds a dynamic count field reference so the runtime value is accurate even at level 2.

**AddressSanitizer** — with `-fstrict-flex-arrays=2`, ASan treats `data[2]` in a `struct { int len; char data[1]; }` as an out-of-bounds access. This is correct (it is UB), but breaks existing code using `data[1]` as a poor man's FAM. Level 0 suppresses these reports.

The implementation in Clang is in `CodeGen/CGBuiltin.cpp` (`EmitBuiltinObjectSize`) and in LLVM's `Analysis/MemoryBuiltins.cpp` (`getObjectSize`). The `StrictFlexArraysLevel` from `LangOptions` is threaded into `ObjectSizeOpts::StrictFlexArraysLevel`.

---

## 38b.10 Interaction with LLVM IR and Embedded Toolchains

### 38b.10.1 New IR Patterns from C23 Features

Two C23 features produce IR patterns that did not arise from C11 code:

**`_BitInt(N)` with non-power-of-two N.** A `_BitInt(24)` variable produces `i24` SSA values. While LLVM handles these legally and the optimizer processes them without widening, the backend's `LegalizeTypes` pass must widen or split them to the target's supported integer widths. On x86-64, `i24` arithmetic requires a `movzx` (zero-extend) after a load and a `mov`+`shl`+`shr` sequence before a store, amounting to a small but measurable overhead compared to `i32`. For tight inner loops operating on `_BitInt(24)` arrays, profiling the assembly is recommended.

The LLVM `SelectionDAGTargetInfo::isOperationLegal` query gates which operations can be directly selected; for `i24`, `ISD::ADD` is legal on x86-64 (it maps to a 32-bit add followed by an `and` mask for unsigned or a sign-extend for signed) but `ISD::MUL` for `i24` is legal only after widening.

**`#embed` producing large `ConstantDataArray` globals.** Embedding megabyte-scale resources produces large `.rodata` sections. The linker's `--gc-sections` flag eliminates unreferenced embed globals efficiently — the whole section is discarded because each embedded global becomes a separate section with `-ffunction-sections -fdata-sections`. During LTO, the IR for all embedded resources is materialized in memory; for ThinLTO, partition large embedded resources into separate translation units to avoid concentrating LTO memory pressure in a single module summary.

**`nullptr` in IR.** No new IR patterns — `nullptr` lowers to `ptr null`, identical to `(void*)0`. The value-range analysis improvements come from the type system: pointer arguments annotated with `nonnull` attribute can now be more precisely diagnosed at the source level (passing `nullptr` to a `nonnull` parameter is an error in Clang).

### 38b.10.2 Embedded Toolchain Considerations

Using C23 with embedded targets requires attention to three areas:

**Toolchain invocation.** The cross-compilation invocation for bare-metal targets:

```bash
# ARM Cortex-M4 with FPU, hard float ABI
clang -std=c23 --target=arm-none-eabi -mcpu=cortex-m4 -mfpu=fpv4-sp-d16 \
      -mfloat-abi=hard -ffreestanding -nostdlib -c firmware.c -o firmware.o

# RISC-V RV32IMFC, ILP32F ABI
clang -std=c23 --target=riscv32-unknown-elf -march=rv32imfc -mabi=ilp32f \
      -ffreestanding -nostdlib -c main.c -o main.o

# RISC-V RV64GC, LP64D ABI (Linux-capable embedded)
clang -std=c23 --target=riscv64-linux-gnu -march=rv64gc -mabi=lp64d \
      -c application.c -o application.o
```

**C runtime library compatibility.** Newlib and picolibc are the dominant C runtimes for embedded targets:

| Runtime | Minimum version with C23 headers |
|---------|----------------------------------|
| Newlib | 4.4.0 (`<stdbit.h>`, `<stdckdint.h>`, `nullptr_t` in `<stddef.h>`) |
| picolibc | 1.8.0 (full C23 header set) |
| musl | 1.2.5+ (partial; `<stdbit.h>` available) |

For older runtimes, Clang's built-in headers (installed in `$(clang --print-resource-dir)/include/`) provide the C23 additions without depending on a system libc. The built-in `<stdbit.h>` implements the bit-manipulation functions as `static inline` wrappers over `__builtin_clz`/`__builtin_popcount` etc., and `<stdckdint.h>` wraps `__builtin_*_overflow`.

**`_BitInt` in interrupt handlers and signal contexts.** Passing `_BitInt(N)` values across an interrupt boundary is safe if the compiler knows about both sides. For `_BitInt(N)` with N ≤ 64, values live in standard integer registers and are saved/restored by the interrupt entry trampoline like any other register. For N > 64, the value is passed by pointer; if a `_BitInt(128)` lives on the interrupted thread's stack and the interrupt handler accesses it through a shared pointer, the access is subject to the usual races.

**`#embed` and flash section placement.** The default `.rodata` section placement may not match required linker script sections for flash-resident constant data on microcontrollers with Harvard memory architectures. Use the `section` attribute alongside `#embed`:

```c
// Force embedded data to a specific flash region:
__attribute__((section(".flash_cert"), used))
static const unsigned char kRootCert[] = {
#embed "root_ca.crt" suffix(, 0x00)
};

// Embedded firmware update image in a dedicated OTA region:
__attribute__((section(".ota_image"), aligned(4)))
static const unsigned char kFwUpdate[] = {
#embed "firmware_v2.bin"
};
```

The `aligned` attribute ensures the embedded array starts at a 4-byte boundary, which matters for DMA-based flash programming on some SoCs.

---

## 38b.11 Chapter Summary

- **C23 (ISO/IEC 9899:2024)** is the largest C revision since C99; Clang 22 supports the full feature set at `-std=c23` with `__STDC_VERSION__ == 202311L`. The `-std=c2x` alias is retained for backward compatibility.

- **`_BitInt(N)`** provides arbitrary-width integers from 1 to `BITINT_MAXWIDTH` bits without promotion to `int`. Clang maps them to LLVM `iN` types directly via `BitIntType` in the AST and `CodeGenTypes::ConvertType`. ABI follows a power-of-two register-slot rounding scheme; widths above 128 bits pass by pointer. The type enables precise-width cryptographic field arithmetic and exact-range hardware register modeling.

- **`nullptr` and `nullptr_t`** bring type-safe null pointers to C, reusing Clang's `CXXNullPtrLiteralExpr`/`BuiltinType::NullPtr` but enforcing stricter conversion rules than C++: no implicit conversion to `void*` or integer types. The keyword is recognized at the `LangOpts.C23` gate in the lexer.

- **`constexpr` for variables** (not functions) enables true integer constant expressions without preprocessor macros; Clang evaluates them via the shared `ExprConstant.cpp` evaluator in `C23` mode. Block-scope `constexpr` int variables replace VLAs for fixed compile-time dimensions.

- **`typeof` and `typeof_unqual`** standardize the long-standing GNU `__typeof__` extension; `typeof_unqual` strips top-level cv/restrict/atomic qualifiers and is essential for type-generic macro patterns. AST nodes `TypeofExprType` and `TypeofType` are shared with the GNU extension implementation.

- **`#embed`** eliminates the build-tool blob-to-C-array conversion step by embedding binary resources directly in translation units. Clang's preprocessor uses a single `tok::annot_embed` annotation token with lazy expansion to avoid materializing millions of pp-tokens for large files. The resulting IR is an LLVM `ConstantDataArray` in `.rodata`.

- **`wb`/`uwb` literal suffixes** create `_BitInt` literals at minimum required width, computed via `APInt::getMinSignedBits` / `APInt::getActiveBits` in the lexer's `NumericLiteralParser`. They enable type-safe initialization without implicit widening.

- **Relaxed `va_start`** allows `va_start(ap)` without naming the last parameter; the single-argument form internally resolves the parameter from `FunctionDecl` and produces identical IR to the two-argument form. Functions declared as `void f(...)` (no named parameters) are now valid in C23.

- **`-fstrict-flex-arrays=N`** at levels 0–3 controls how Clang treats trailing array members for `__builtin_object_size`, `__builtin_dynamic_object_size`, and AddressSanitizer. Level 2 matches C23 semantics; level 3 is the strictest and recommended for new security-hardened code. The Linux kernel uses level 3 in sanitizer builds.

- **Embedded toolchain integration** requires Newlib 4.4+/picolibc 1.8+ for C23 headers; Clang's built-in headers provide C23 additions for older runtimes. `#embed` globals need `__attribute__((section(...)))` when targeting specialized flash regions; `_BitInt(N)` with N > 64 passes by pointer on most embedded ABIs.

---

*Cross-references:*
- [Chapter 17 — The Type System](../part-04-llvm-ir/ch17-the-type-system.md) — LLVM `iN` integer types and legalization
- [Chapter 31 — The Lexer and Preprocessor](../part-05-clang-frontend/ch31-the-lexer-and-preprocessor.md) — preprocessor directive handling and annotation token design
- [Chapter 33 — Sema: Names, Lookups, Conversions](../part-05-clang-frontend/ch33-sema-names-lookups-conversions.md) — implicit conversion rules and `CheckAssignmentConstraints`
- [Chapter 35 — The Constant Evaluator](../part-05-clang-frontend/ch35-the-constant-evaluator.md) — `ConstantExpressionEvaluator` internals and `EvalInfo`
- [Chapter 36 — The Clang AST in Depth](../part-05-clang-frontend/ch36-the-clang-ast-in-depth.md) — `Type` hierarchy, `ASTContext`, and `QualType`
