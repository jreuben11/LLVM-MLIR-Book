# Chapter 35 — The Constant Evaluator

*Part V: The Clang Frontend*

Every C and C++ program contains expressions that the compiler must evaluate before it emits any object code: array dimensions, enumerator values, case labels, bit-field widths, non-type template arguments, `static_assert` conditions, concept satisfaction queries, and the entire body of every `constexpr` function called at compile time. Clang handles all of these through a single subsystem: the constant evaluator, implemented in [`clang/lib/AST/ExprConstant.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/AST/ExprConstant.cpp). This chapter dissects that subsystem from its public API entry points down to the value representation and diagnostic infrastructure, giving you the conceptual model and the concrete class names required to read, extend, or call the evaluator yourself.

---

## 35.1 Why Constant Evaluation Exists in the Compiler

The C standard has required integer constant expressions (ICEs) since C89. An ICE must be evaluable by the compiler without knowledge of runtime state; the classic uses are:

- **Array bounds** — `int buf[N]` requires `N` to be an ICE in C.
- **Enumerator values** — `enum { A = 1 << 20 }` must be evaluated to assign the discriminant.
- **Case labels** — every `case` value must be a unique, compiler-known integer.
- **Bit-field widths** — `struct { unsigned x : W; }` requires `W` to be a compile-time constant.

C++ dramatically widened the scope of required constant evaluation. Since C++11, the language defines a hierarchy of constant expression categories: *integral constant expressions* (ICE), *constant expressions* (the broader C++11 category that also covers floating-point, pointers, and references), *core constant expressions*, and *manifestly constant-evaluated* expressions. The additions each version brought are:

| Standard | New constant-expression feature |
|----------|--------------------------------|
| C++11 | `constexpr` functions (single-return body), non-type template arguments of all literal types |
| C++14 | Multi-statement `constexpr` functions, local variable mutation |
| C++17 | `if constexpr`, `constexpr` lambdas |
| C++20 | `consteval` (immediate functions), `constinit`, `std::is_constant_evaluated()`, dynamic allocation in constant expressions (`new`/`delete`) |
| C++23 | `if consteval`, relaxed pointer conversions in constant expressions |
| C++26 | `static_assert` with `std::string` message |

Beyond language correctness, constant folding at the AST level (rather than in the IR optimizer) has concrete benefits: it enables better diagnostics (Clang can blame a specific expression in a `constexpr` function), it feeds template instantiation and SFINAE, it backs `static_assert` and concept satisfaction checking, and it allows the code generator to emit `.rodata` initializers without calling the linker's startup-initialization machinery.

The evaluator is also the backing implementation for `__builtin_constant_p`, which GCC-compatible code uses to detect whether an expression is a compile-time constant so it can select between a fast constant-argument path and a general runtime path.

---

## 35.2 Entry Points on `Expr`

The entire evaluator surface is exposed through methods on `clang::Expr`, declared in [`clang/include/clang/AST/Expr.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Expr.h).

### 35.2.1 `EvalStatus` and `EvalResult`

Before calling any evaluator entry point, callers prepare an `Expr::EvalStatus` or `Expr::EvalResult` object to capture the evaluation outcome:

```cpp
struct EvalStatus {
  bool HasSideEffects = false;      // unevaluated side effects encountered
  bool HasUndefinedBehavior = false; // UB was encountered but folded anyway
  SmallVectorImpl<PartialDiagnosticAt> *Diag = nullptr;
};

struct EvalResult : EvalStatus {
  APValue Val;           // the evaluated result
  bool isGlobalLValue() const;
};
```

`HasSideEffects` is set when the evaluator short-circuits past an expression with observable side-effects (for example, `(sideEffect(), 0)` can be folded to `0` while noting the side-effect). `HasUndefinedBehavior` is set when the evaluated result is known but the evaluation path included undefined behavior (signed integer overflow, out-of-bounds pointer arithmetic). The `Diag` pointer, when non-null, receives a stack of `PartialDiagnosticAt` notes explaining why the expression is not a constant expression under the applicable standard rules.

### 35.2.2 `SideEffectsKind`

Most evaluator entry points accept a `SideEffectsKind` argument controlling how aggressively the evaluator attempts to fold:

```cpp
enum SideEffectsKind {
  SE_NoSideEffects,          // strict: fail if any side-effect is encountered
  SE_AllowUndefinedBehavior, // fold through UB but not observable side-effects
  SE_AllowSideEffects        // fold aggressively, ignoring all side-effects
};
```

### 35.2.3 `EvaluateAsRValue` and Typed Variants

```cpp
bool Expr::EvaluateAsRValue(EvalResult &Result, const ASTContext &Ctx,
                             bool InConstantContext = false) const;

bool Expr::EvaluateAsInt(EvalResult &Result, const ASTContext &Ctx,
                         SideEffectsKind AllowSideEffects = SE_NoSideEffects,
                         bool InConstantContext = false) const;

bool Expr::EvaluateAsFloat(llvm::APFloat &Result, const ASTContext &Ctx,
                           SideEffectsKind AllowSideEffects = SE_NoSideEffects,
                           bool InConstantContext = false) const;

bool Expr::EvaluateAsFixedPoint(EvalResult &Result, const ASTContext &Ctx,
                                SideEffectsKind AllowSideEffects,
                                bool InConstantContext = false) const;

bool Expr::EvaluateAsLValue(EvalResult &Result, const ASTContext &Ctx,
                             bool InConstantContext = false) const;

bool Expr::EvaluateAsInitializer(APValue &Result, const ASTContext &Ctx,
                                  const VarDecl *VD,
                                  SmallVectorImpl<PartialDiagnosticAt> &Notes,
                                  bool IsConstantInitialization) const;
```

`EvaluateAsRValue` is the most general: it attempts folding with any technique available and, if the expression is a glvalue, performs the implicit lvalue-to-rvalue conversion. `EvaluateAsInt` wraps it and additionally asserts that the result is an integer (it accepts only integer-typed expressions). `EvaluateAsLValue` succeeds only if the expression evaluates to a stable global address suitable for a constant initializer. `EvaluateAsInitializer` is used for static and thread-local variable initializers; it sets up the `EvaluatingDecl` context inside `EvalInfo` so the evaluator knows which declaration's storage is being initialized, enabling detection of self-referential initializers.

### 35.2.4 `EvaluateAsConstantExpr`

```cpp
bool Expr::EvaluateAsConstantExpr(
    EvalResult &Result, const ASTContext &Ctx,
    ConstantExprKind Kind = ConstantExprKind::Normal) const;
```

This is the standard-conforming entry point. It evaluates the expression under the `EM_ConstantExpression` mode (Section 35.4), which means the evaluator will fail and produce diagnostic notes if it encounters anything that is not a constant expression under the applicable language rules — including reads of non-constexpr variables, calls to non-constexpr functions, `reinterpret_cast`, and most forms of undefined behavior.

The `ConstantExprKind` enum controls subtle policy differences:

| Kind | Behavior |
|------|---------|
| `Normal` | Standard constant expression rules |
| `NonClassTemplateArgument` | Allows references to dllimported functions (for mangling only) |
| `ClassTemplateArgument` | Template parameter object; result must be usable as a code-gen constant |
| `ImmediateInvocation` | `consteval` call; temporaries except the final result are destroyed |

### 35.2.5 `isPotentialConstantExpr`

Two static methods probe whether a function or expression *might* be constant without committing to an answer:

```cpp
static bool Expr::isPotentialConstantExpr(
    const FunctionDecl *FD,
    SmallVectorImpl<PartialDiagnosticAt> &Diags);

static bool Expr::isPotentialConstantExprUnevaluated(
    Expr *E,
    const FunctionDecl *FD,
    SmallVectorImpl<PartialDiagnosticAt> &Diags);
```

These are called during the definition phase of `constexpr` functions. Rather than evaluating a specific call with concrete arguments, they evaluate the function body with an unknown `this` and unknown argument values, looking for constructs that can *never* be constant (such as inline assembly or a call to a function that is definitively not `constexpr`). Failures populate `Diags` but do not issue diagnostics; the failures are only reported if and when the function is actually called in a constant context.

### 35.2.6 ICE Checking and `Sema::VerifyIntegerConstantExpression`

The `Expr` methods for ICE checking are:

```cpp
std::optional<llvm::APSInt>
Expr::getIntegerConstantExpr(const ASTContext &Ctx) const;
bool Expr::isIntegerConstantExpr(const ASTContext &Ctx) const;
bool Expr::isCXX98IntegralConstantExpr(const ASTContext &Ctx) const;
bool Expr::isCXX11ConstantExpr(const ASTContext &Ctx,
                                APValue *Result = nullptr,
                                SourceLocation *Loc = nullptr) const;
```

Semantic analysis calls these indirectly through `Sema::VerifyIntegerConstantExpression`, which wraps the evaluation in proper diagnostic emission:

```cpp
ExprResult Sema::VerifyIntegerConstantExpression(
    Expr *E, llvm::APSInt *Result,
    VerifyICEDiagnoser &Diagnoser,
    AllowFoldKind CanFold = AllowFoldKind::No);
```

`VerifyICEDiagnoser` is an abstract base class that sub-diagnosers override to emit the specific error for the context — `diag::err_expr_not_cce` with a selector argument distinguishes case values (selector 0), enumerator values (1), non-type template arguments (2), and array sizes (4) among others.

### 35.2.7 `VarDecl::evaluateValue`

For `static` and thread-local variables with non-trivial initializers, the evaluator is driven through `VarDecl`:

```cpp
APValue *VarDecl::evaluateValue() const;
APValue *VarDecl::getEvaluatedValue() const;  // returns cached value or null
bool VarDecl::evaluateDestruction(
    SmallVectorImpl<PartialDiagnosticAt> &Notes) const;
```

The result is cached in an `EvaluatedStmt` struct that hangs off the `VarDecl`:

```cpp
struct EvaluatedStmt {
  bool WasEvaluated : 1;
  bool HasConstantInitialization : 1;
  bool HasConstantDestruction : 1;
  bool HasICEInit : 1;
  APValue Evaluated;  // the stored result
  // ...
};
```

`HasConstantInitialization` drives the code generator's decision to emit the variable into `.rodata` (or `.data` with a compile-time constant value) rather than into `.bss` with a startup constructor call.

---

## 35.3 `APValue` — The Constant Value Representation

Every evaluated constant is represented as an `APValue`, defined in [`clang/include/clang/AST/APValue.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/APValue.h). `APValue` is a discriminated union over all the types a C++ constant expression can produce.

### 35.3.1 `ValueKind` Enum

```cpp
enum ValueKind {
  None,            // outside its lifetime (not yet constructed / already destroyed)
  Indeterminate,   // uninitialized storage (C++ [basic.indet])
  Int,             // llvm::APSInt
  Float,           // llvm::APFloat
  FixedPoint,      // llvm::APFixedPoint
  ComplexInt,      // pair of APSInt
  ComplexFloat,    // pair of APFloat
  LValue,          // pointer / reference (base + designator path)
  Vector,          // N × APValue
  Array,           // N × APValue (with optional "filler" for trailing zero-init)
  Struct,          // NumBases base-class APValues + NumFields field APValues
  Union,           // active FieldDecl* + APValue of active member
  MemberPointer,   // ValueDecl* + derivation path
  AddrLabelDiff    // GNU extension: difference of two &&label addresses
};
```

`None` and `Indeterminate` have no payload. Reading an `Indeterminate` value is undefined behavior in C++20 and causes the evaluator to emit a diagnostic. The remaining kinds carry their payload in a `DataType` union (a `llvm::AlignedCharArrayUnion` large enough for the largest member).

### 35.3.2 `APValue::LValueBase`

An `LValue`-kind `APValue` identifies a storage location through its *base* and a *designator path*. The base is an `LValueBase`:

```cpp
class LValueBase {
  // A pointer union over:
  //   const ValueDecl*         — a named declaration (variable, function)
  //   const Expr*              — a temporary or compound literal
  //   TypeInfoLValue           — typeid(T) object
  //   DynamicAllocLValue       — heap-allocated object (constexpr new)
  PointerUnion<const ValueDecl*, const Expr*,
               TypeInfoLValue, DynamicAllocLValue> Ptr;
  unsigned CallIndex;  // which call frame (for local variables)
  unsigned Version;    // version counter to distinguish temporaries
};
```

The `CallIndex` and `Version` fields are non-zero only for local variables and temporaries in constexpr function calls. They serve to distinguish multiple calls to the same function and multiple temporaries with the same type within a single call. A null base with zero offset represents the null pointer.

### 35.3.3 `LValuePathEntry` and Designators

The path from the base to the specific subobject is represented as an `SmallVector<LValuePathEntry>`. Each `LValuePathEntry` is a 64-bit value that encodes either an array index (for array subscripting) or a `BaseOrMemberType` (a `PointerIntPair<const Decl*, 1, bool>` holding a field or base class record, with the bool indicating a virtual base):

```cpp
struct LValuePathEntry {
  static LValuePathEntry ArrayIndex(uint64_t Index);
  BaseOrMemberType getAsBaseOrMember() const;
  uint64_t getAsArrayIndex() const;
};
```

The evaluator uses `APValue::getLValueBase()`, `getLValueOffset()`, `getLValuePath()`, and `isLValueOnePastTheEnd()` to reconstruct the full pointer semantics.

### 35.3.4 Struct, Array, and Union APValues

For aggregates, `APValue` stores sub-values in heap-allocated arrays:

```cpp
// Struct: NumBases base-class values followed by NumFields field values
APValue &APValue::getStructBase(unsigned I);
APValue &APValue::getStructField(unsigned I);

// Array: stores the first N initialized elements; the remainder are
//        implicitly the "array filler" (zero or the trailing element value)
APValue &APValue::getArrayInitializedElt(unsigned I);
APValue &APValue::getArrayFiller();

// Union: active field descriptor and the value
const FieldDecl *APValue::getUnionField() const;
APValue &APValue::getUnionValue();
```

The distinction between `getArrayInitializedElt()` and the filler matters for large zero-initialized arrays: storing every element would blow out memory. The evaluator stores only the explicitly initialized prefix and provides a filler for the rest.

---

## 35.4 `EvalInfo` — Evaluation Context and State

`EvalInfo` is a large internal struct defined entirely within `ExprConstant.cpp`; it is not part of the public API. It inherits from `interp::State` (the bytecode interpreter's state base class) and carries all mutable state for one complete evaluation:

```cpp
class EvalInfo : public interp::State {
public:
  ASTContext &Ctx;
  Expr::EvalStatus &EvalStatus;

  CallStackFrame *CurrentCall;    // top of the constexpr call stack
  unsigned CallStackDepth;        // current depth
  unsigned NextCallIndex;         // monotonically increasing call index
  unsigned StepsLeft;             // countdown from LangOpts.ConstexprStepLimit
  bool EnableNewConstInterp;      // use the bytecode interpreter?

  CallStackFrame BottomFrame;     // sentinel bottom frame
  llvm::SmallVector<Cleanup, 16> CleanupStack;

  APValue::LValueBase EvaluatingDecl;
  APValue *EvaluatingDeclValue;   // storage being initialized

  unsigned SpeculativeEvaluationDepth = 0; // for non-committing evaluation
  bool HasActiveDiagnostic;
  bool HasFoldFailureDiagnostic;
  bool CheckingPotentialConstantExpression;
  bool CheckingForUndefinedBehavior;
  bool InConstantContext;
  // ...
};
```

### 35.4.1 `EvaluationMode`

The `EvalMode` field selects the policy for handling non-constant constructs:

```cpp
enum EvaluationMode {
  EM_ConstantExpression,           // fail on anything non-constant
  EM_ConstantExpressionUnevaluated,// as above but in unevaluated context
  EM_ConstantFold,                 // fold through side-effects if possible
  EM_IgnoreSideEffects,            // fold aggressively, ignore all side-effects
} EvalMode;
```

`EM_ConstantExpression` corresponds to a genuine required-constant-expression context such as a template argument or `static_assert`. `EM_ConstantFold` is used by the optimizer-level `EvaluateAsRValue` queries that want the value if available but will not report diagnostics on failure. `EM_IgnoreSideEffects` is used for `__builtin_constant_p` evaluation, where the goal is simply to determine whether a value is known.

### 35.4.2 `CallStackFrame`

Each constexpr function invocation pushes a `CallStackFrame` onto the linked list rooted at `EvalInfo::CurrentCall`:

```cpp
class CallStackFrame : public interp::Frame {
  EvalInfo &Info;
  CallStackFrame *Caller;
  const FunctionDecl *Callee;
  const LValue *This;
  const Expr *CallExpr;
  CallRef Arguments;

  // Local variables and temporaries, keyed by (ParmVarDecl*, version)
  std::map<std::pair<const void*, unsigned>, APValue> Temporaries;
  SourceRange CallRange;
  unsigned Index;   // monotone call index from EvalInfo::NextCallIndex
};
```

The `Temporaries` map is intentionally a `std::map` (not `DenseMap`) so that references and pointers into it remain stable across insertions — the evaluator takes `APValue*` pointers into this map when evaluating `constexpr` local variables.

### 35.4.3 `EvalInfo::setEvaluatingDecl`

When evaluating the initializer of a static or thread-local variable, the evaluator must know which declaration is being initialized so it can detect self-referential initializers (reading the variable's own storage during its initialization is undefined behavior). The method `setEvaluatingDecl` establishes this context:

```cpp
void EvalInfo::setEvaluatingDecl(APValue::LValueBase Base, APValue &Value,
                                   EvaluatingDeclKind EDK = EvaluatingDeclKind::Ctor) {
  EvaluatingDecl = Base;
  IsEvaluatingDecl = EDK;
  EvaluatingDeclValue = &Value;
}
```

The `EvaluatingDeclKind` enum distinguishes between evaluating a constructor (the common case), evaluating a destructor for `HasConstantDestruction`, and no-evaluation. When a `DeclRefExpr` is visited during evaluation and its declaration matches `EvaluatingDecl`, the evaluator returns the partially-constructed `EvaluatingDeclValue` rather than the declaration's fully-initialized stored value, which would be the same object. This allows `constexpr` constructors to self-reference their own members during initialization.

### 35.4.4 `noteFailure` and `SpeculativeEvaluationRAII`

When a potential constant expression needs to be checked speculatively (for example, to determine whether either branch of a conditional could be constant), the evaluator uses `SpeculativeEvaluationRAII` to suppress committed side-effects:

```cpp
class SpeculativeEvaluationRAII {
  // On construction: Info.SpeculativeEvaluationDepth = Info.CallStackDepth + 1
  // On destruction: restores previous depth, discards any accumulated state
};
```

The `noteFailure()` method on `EvalInfo` decides whether evaluation should continue after a failure (for potential-constant-expression checking, it must continue to discover all failures):

```cpp
[[nodiscard]] bool noteFailure() {
  // Returns true if we should continue evaluating to find more problems.
  // Returns false if we should abort immediately.
}
```

### 35.4.5 The `InConstantContext` Flag and `IgnoreSideEffectsRAII`

Alongside `EvalMode`, the boolean `Info.InConstantContext` marks whether evaluation is running inside a genuine required-constant-expression context (as opposed to a speculative fold). Several builtins check this flag — most importantly `__builtin_is_constant_evaluated` and `__builtin_constant_p`. The flag is set by the entry points:

```cpp
// In EvaluateAsConstantExpr:
Info.InConstantContext = true;

// In EvaluateAsRValue (folding, not required-constant):
Info.InConstantContext = InConstantContext;  // caller-supplied
```

The `IgnoreSideEffectsRAII` RAII helper temporarily switches `EvalMode` to `EM_IgnoreSideEffects` for the duration of its scope, used internally when the evaluator needs to probe a subexpression for a constant value without caring about side-effects:

```cpp
struct IgnoreSideEffectsRAII {
  EvalInfo &Info;
  EvaluationMode OldMode;
  IgnoreSideEffectsRAII(EvalInfo &Info)
      : Info(Info), OldMode(Info.EvalMode) {
    Info.EvalMode = EvalInfo::EM_IgnoreSideEffects;
  }
  ~IgnoreSideEffectsRAII() { Info.EvalMode = OldMode; }
};
```

### 35.4.6 Diagnostic Emission: `FFDiag`, `CCEDiag`, and `Note`

Two diagnostic methods on `EvalInfo` (and forwarded from `ExprEvaluatorBase`) produce the structured failure notes:

- **`FFDiag`** (fold-failure diagnostic): used when the expression *cannot be folded at all*. Stored in `EvalStatus.Diag` and flagged by `HasFoldFailureDiagnostic`.
- **`CCEDiag`** (core-constant-expression diagnostic): used when the expression *can be folded* but is not a valid constant expression under the standard rules. In `EM_ConstantFold` and `EM_IgnoreSideEffects` modes these are suppressed; in `EM_ConstantExpression` mode they become fatal.

The result is an `OptionalDiagnostic` — a wrapper that collects streaming arguments only if a `Diag` vector is actually attached to the `EvalStatus`.

Limit-related failures have dedicated diagnostics:
- `diag::note_constexpr_depth_limit_exceeded` — call depth exceeds `LangOpts.ConstexprCallDepth` (default 512).
- `diag::note_constexpr_call_limit_exceeded` — total call count exceeds `LangOpts.ConstexprCallLimit` (default 1048576).
- `diag::note_constexpr_step_limit_exceeded` — evaluation steps exceed `LangOpts.ConstexprStepLimit`.

---

## 35.5 Integer Constant Expression Evaluation

The `IntExprEvaluator` class handles all expressions whose result type is integral or enumeration:

```cpp
class IntExprEvaluator
    : public ExprEvaluatorBase<IntExprEvaluator> {
  APValue &Result;
  // ...
  bool VisitIntegerLiteral(const IntegerLiteral *E);
  bool VisitCharacterLiteral(const CharacterLiteral *E);
  bool VisitDeclRefExpr(const DeclRefExpr *E);
  bool VisitBinaryOperator(const BinaryOperator *E);
  bool VisitUnaryOperator(const UnaryExprOrTypeTraitExpr *E);
  bool VisitOffsetOfExpr(const OffsetOfExpr *E);
  bool VisitCallExpr(const CallExpr *E);
  // ... 60+ more Visit methods
};
```

### 35.5.1 The `ExprEvaluatorBase` CRTP Base

All five concrete evaluator classes share a CRTP base:

```cpp
template <class Derived>
class ExprEvaluatorBase : public ConstStmtVisitor<Derived, bool> {
protected:
  EvalInfo &Info;

  OptionalDiagnostic CCEDiag(const Expr *E, diag::kind D);
  bool ZeroInitialization(const Expr *E) { return Error(E); }
  bool IsConstantEvaluatedBuiltinCall(const CallExpr *E);

public:
  ExprEvaluatorBase(EvalInfo &Info) : Info(Info) {}
  // Default Visit handlers for shared expression kinds:
  bool VisitStmt(const Stmt *S);   // error: unexpected statement
  bool VisitExpr(const Expr *E);   // error: unhandled expression
  bool VisitParenExpr(const ParenExpr *E);
  bool VisitUnaryExtension(const UnaryOperator *E);
  bool VisitUnaryPlus(const UnaryOperator *E);
  bool VisitChooseExpr(const ChooseExpr *E);
  bool VisitGenericSelectionExpr(const GenericSelectionExpr *E);
  bool VisitSubstNonTypeTemplateParmExpr(const SubstNonTypeTemplateParmExpr *E);
  bool VisitConstantExpr(const ConstantExpr *E);
  bool VisitCXXReinterpretCastExpr(const CXXReinterpretCastExpr *E);
  // ...
};
```

The `VisitConstantExpr` handler is particularly important: when the evaluator encounters a `ConstantExpr` node that already has a cached result, it short-circuits by returning that cached `APValue` directly without re-evaluating the subexpression. This is the mechanism by which evaluated template arguments and pre-evaluated `consteval` calls avoid redundant work during code generation and template instantiation.

The `VisitCXXReinterpretCastExpr` default handler calls `CCEDiag` with `note_constexpr_invalid_cast`, making `reinterpret_cast` a constant-expression failure in `EM_ConstantExpression` mode but silently failing in fold modes.

### 35.5.2 Binary Operator Handling and Overflow

Integer binary operations go through `handleIntIntBinOp`, which calls `llvm::APSInt` arithmetic and then `HandleOverflow` for signed types:

```cpp
static bool HandleOverflow(EvalInfo &Info, const Expr *E,
                            const APSInt &Value, QualType Type) {
  // If Value is out of range for Type:
  Info.CCEDiag(E, diag::note_constexpr_overflow) << Value << Type;
  return Info.noteFailure();
}
```

Signed integer overflow, left-shift of negative values, and shifts whose amount equals or exceeds the type's bit-width all go through `CCEDiag` with specific notes: `note_constexpr_overflow`, `note_constexpr_negative_shift`, `note_constexpr_large_shift`, `note_constexpr_lshift_of_negative`, `note_constexpr_lshift_discards`.

The actual arithmetic for checked addition and multiplication uses a widened intermediate:

```cpp
// BO_Add:
CheckedIntArithmetic(Info, E, LHS, RHS, LHS.getBitWidth() + 1,
                     std::plus<APSInt>(), Result);
// BO_Mul:
CheckedIntArithmetic(Info, E, LHS, RHS, LHS.getBitWidth() * 2,
                     std::multiplies<APSInt>(), Result);
```

`CheckedIntArithmetic` performs the operation at the widened precision, then checks whether the result fits in the original bit-width. Bitwise operations (`&`, `|`, `^`) never overflow and return directly. Division and remainder check for zero divisor (`note_expr_divide_by_zero`) and for the `INT_MIN / -1` overflow case. Note that C++20 changed left-shift to be well-defined for signed types (wrapping is no longer UB), so the `note_constexpr_lshift_of_negative` and `note_constexpr_lshift_discards` warnings are only emitted in pre-C++20 modes.

### 35.5.3 `sizeof` and `alignof`

`sizeof` expressions are handled as `UnaryExprOrTypeTraitExpr` with `UETT_SizeOf`. The evaluator calls `ASTContext::getTypeSize()` or `getTypeSizeInChars()` and returns the result as an `APSInt` with type `size_t`. `alignof` is analogous via `ASTContext::getTypeAlign()`.

### 35.5.4 Conditional Operators, Comma Operators, and Short-Circuit Evaluation

The base class `ExprEvaluatorBase::HandleConditionalOperator` handles `? :` and `if constexpr`:

1. Evaluate the condition via `EvaluateAsBooleanCondition`.
2. If the condition is constant, visit only the selected branch.
3. If the condition is not constant in `CheckingPotentialConstantExpression` mode, speculatively evaluate *both* branches using `SpeculativeEvaluationRAII` to determine whether either branch could contribute a failure.

Short-circuit `&&` and `||` in boolean contexts are handled by `IntExprEvaluator::VisitBinaryOperator` via early-return on the first known operand. The comma operator evaluates the left operand via `EvaluateIgnoredValue` (which runs the evaluator purely for its side-effects on `EvalInfo`) then visits the right operand for the result.

### 35.5.5 Cast Handling: `HandleIntToIntCast` and `HandleFloatToIntCast`

```cpp
static APSInt HandleIntToIntCast(EvalInfo &Info, const Expr *E,
                                  QualType DestType, QualType SrcType,
                                  APSInt &Value);

static bool HandleFloatToIntCast(EvalInfo &Info, const Expr *E,
                                  QualType SrcType, const APFloat &Value,
                                  QualType DestType, APSInt &Result);
```

`HandleIntToIntCast` performs sign-extension, zero-extension, or truncation using `APSInt::extOrTrunc` and `setIsUnsigned`. `HandleFloatToIntCast` uses `APFloat::convertToInteger`; out-of-range truncation sets `HasUndefinedBehavior` in `EvalStatus` rather than silently wrapping.

### 35.5.6 `OffsetOfExpr`

`__builtin_offsetof(T, member)` is evaluated in `IntExprEvaluator::VisitOffsetOfExpr`. The evaluator iterates over the offset-of components (array indices, field accesses, base class steps), calling `ASTContext::getASTRecordLayout()` and summing the byte offsets:

```cpp
bool IntExprEvaluator::VisitOffsetOfExpr(const OffsetOfExpr *OOE) {
  CharUnits Result;
  for (unsigned i = 0, n = OOE->getNumComponents(); i != n; ++i) {
    // Accumulate field offsets and array element sizes via RecordLayout
  }
  return Success(Result, OOE);
}
```

---

## 35.6 `constexpr` Function Evaluation

### 35.6.1 `HandleFunctionCall`

The primary workhorse for evaluating a call to a `constexpr` function is `HandleFunctionCall`:

```cpp
static bool HandleFunctionCall(
    SourceLocation CallLoc,
    const FunctionDecl *Callee,
    const LValue *This,
    const Expr *E,
    ArrayRef<const Expr *> Args,
    CallRef Call,
    const Stmt *Body,
    EvalInfo &Info,
    APValue &Result,
    const LValue *ResultSlot);
```

It first checks the call limit (`Info.CheckCallLimit`), then creates a `CallStackFrame` on the stack. Argument evaluation (`EvaluateArgs`) iterates over the `ParmVarDecl`s and stores their evaluated `APValue`s in the new frame's `Temporaries` map. The body is then executed by visiting each `Stmt` in the function body through the statement visitor.

### 35.6.2 Constructor and Destructor Calls

Constructor evaluation is `HandleConstructorCall`. It distinguishes between trivial and non-trivial constructors: for a trivial copy or move constructor, it performs a direct `APValue` copy. For a non-trivial constructor, it evaluates member initializers in declaration order, storing each into the appropriate field slot of the `Struct`-kind `APValue` that represents the object being constructed.

`HandleDestructorCall` operates symmetrically, running `~T()` on the `APValue` and checking that the destruction has no externally visible side-effects (required for `HasConstantDestruction`).

### 35.6.3 Local State and Mutation

Within a constexpr function, local variables are `APValue` entries in `CallStackFrame::Temporaries`. Assignments to local variables are handled by `handleAssignment`, which traverses the `LValue` designator path to locate the correct `APValue` slot and overwrites it. The key constraint is that the evaluator tracks which base object is being initialized (`EvaluatingDecl`): a constexpr function may freely mutate local variables and formal arguments but may not write to global or static storage that was already initialized before the evaluation began (because such writes are not modeled as compile-time constants).

### 35.6.4 `consteval` and Immediate Invocations

`consteval` functions must be called in a manifestly constant-evaluated context. Clang marks these calls as `ImmediateInvocation` via `ConstantExprKind::ImmediateInvocation`. The `ActOnCallExpr` Sema path identifies such calls and wraps them in `ConstantExpr` nodes during semantic analysis. The constant evaluator then runs on these wrapped expressions with the `ImmediateInvocation` kind, which causes all temporaries created during evaluation to be destroyed as part of the evaluation (since the standard requires the call to be evaluated to a prvalue with no surviving temporary).

### 35.6.5 C++14 Multi-Statement Bodies

C++14 lifted the restriction that a `constexpr` function body must be a single `return` statement. The evaluator handles `IfStmt`, `ForStmt`, `WhileStmt`, `DoStmt`, `SwitchStmt`, `BreakStmt`, `ContinueStmt`, `ReturnStmt`, `DeclStmt`, `NullStmt`, and `CompoundStmt` via a statement visitor rooted in `EvaluateStmt`. Each iteration of a loop decrements `StepsLeft` in `EvalInfo`; when `StepsLeft` reaches zero the evaluator emits `note_constexpr_step_limit_exceeded` and aborts.

`DeclStmt` evaluates the initializer of each local variable and stores the result in `CallStackFrame::Temporaries` keyed by the `VarDecl*` and the current version. `ReturnStmt` evaluates the return expression and sets a `ReturnValue` slot in the frame; `EvaluateStmt` detects this slot being set and unwinds the statement visitor. `SwitchStmt` evaluates its condition to an integer and finds the matching `CaseStmt` by walking the case label list, then jumps to it — the evaluator does not use computed `goto` but instead walks the statement tree looking for the matching case label, respecting the `StepsLeft` countdown.

### 35.6.6 Dynamic Memory Allocation in `constexpr` (C++20)

C++20 permits `new` and `delete` in `constexpr` functions provided all dynamic allocations are deallocated before the constant expression evaluation completes. The evaluator tracks live heap allocations in `EvalInfo::HeapAllocs` (a `std::map<DynamicAllocLValue, DynAlloc>`) and their `APValue` storage. An expression that ends evaluation with outstanding heap allocations fails with `note_constexpr_heap_alloc_limit_exceeded` or with a leak diagnostic if `-Wcontexpr-heap-leaks` is enabled. The `DynamicAllocLValue` type (Section 35.3.1) is the base of any `APValue::LValue` that refers to a constexpr-new'd object.

---

## 35.7 LValue Evaluation and Pointer Arithmetic

### 35.7.1 `LValueExprEvaluator`

The `LValueExprEvaluator` (deriving from `LValueExprEvaluatorBase<LValueExprEvaluator>`) evaluates expressions whose value category is glvalue. Its result is an internal `LValue` struct (distinct from `APValue` with kind `LValue`) that carries a base, a `CharUnits` byte offset, and a `SubobjectDesignator` — the evaluator's richer representation of an lvalue path before it is serialized into the `APValue::LValuePathEntry` array.

The key helper functions are:

```cpp
// Navigate into a member field:
static bool HandleLValueMember(EvalInfo &Info, const Expr *E, LValue &LVal,
                                const FieldDecl *FD,
                                const ASTRecordLayout *RL = nullptr);

// Navigate into a base class subobject:
static bool HandleLValueBase(EvalInfo &Info, const Expr *E, LValue &Obj,
                              const CXXRecordDecl *DerivedDecl,
                              const CXXBaseSpecifier *Base);

// Advance an array pointer by N elements:
static bool HandleLValueArrayAdjustment(EvalInfo &Info, const Expr *E,
                                         LValue &LVal, QualType EltTy,
                                         APSInt Adjustment);
```

`HandleLValueMember` calls `ASTContext::getASTRecordLayout()` to obtain the byte offset of the field and appends a `LValuePathEntry` built from a `BaseOrMemberType(FD, false)` to the designator.

### 35.7.2 One-Past-The-End Pointer Legality

The evaluator tracks the `IsOnePastTheEnd` flag in `SubobjectDesignator`. A one-past-the-end pointer is legal to form (via `&arr[N]` or pointer arithmetic) but illegal to dereference. `CheckLValueConstantExpression` verifies that a pointer intended for use as a constant is either null, points to a valid object (including a complete object via its designator), or is a one-past-the-end pointer to an array. The diagnostics `note_constexpr_past_end_subobject` and `note_constexpr_array_index` describe violations.

### 35.7.3 Pointer Arithmetic: `HandleLValueArrayAdjustment`

Array subscripting and pointer arithmetic both funnel through `HandleLValueArrayAdjustment`:

```cpp
static bool HandleLValueArrayAdjustment(EvalInfo &Info, const Expr *E,
                                         LValue &LVal, QualType EltTy,
                                         APSInt Adjustment) {
  CharUnits EltSize;
  if (!HandleSizeof(Info, E->getExprLoc(), EltTy, EltSize))
    return false;
  LVal.Offset += EltSize * Adjustment.getExtValue();
  // Update the SubobjectDesignator to reflect the new array index
  LVal.Designator.adjustIndex(Info, E, Adjustment);
  return true;
}
```

The function updates both the byte-level `CharUnits` offset (which is what ends up in the final `APValue::getLValueOffset()`) and the logical `SubobjectDesignator` (which tracks the index within the array for bounds checking). When the adjusted index falls outside `[0, array_size]` (the closed interval including one-past-the-end), the designator is marked invalid with `note_constexpr_array_index`.

### 35.7.4 `CheckLValueConstantExpression`

Before an `LValue` can be stored as a constant, `CheckLValueConstantExpression` validates it:

- The base must be a global variable, a string literal, a `typeid` expression, or a dynamic allocation (in contexts where `new` in constexpr is permitted).
- If the base is a `DeclRefExpr` to a variable, that variable must have `constexpr` or `const` with a constant initializer.
- The path must not pass through a `mutable` subobject.
- References to temporaries are only valid within the expression that created the temporary.

### 35.7.5 `std::is_constant_evaluated` / `__builtin_is_constant_evaluated`

Both forms are handled in `IntExprEvaluator::VisitCallExpr` at the `Builtin::BI__builtin_is_constant_evaluated` case:

```cpp
case Builtin::BI__builtin_is_constant_evaluated: {
  return Success(Info.InConstantContext, E);
}
```

`Info.InConstantContext` is set to `true` whenever the evaluator is running in `EM_ConstantExpression` or `EM_ConstantExpressionUnevaluated` mode. The evaluator also emits `diag::warn_is_constant_evaluated_always_true_constexpr` if the call occurs at call-stack depth 1 in a constant context — a common programming mistake where the programmer tests `std::is_constant_evaluated()` inside a `constexpr` function but the call is actually always constant.

---

## 35.8 Floating-Point Constant Evaluation

### 35.8.1 `FloatExprEvaluator`

`FloatExprEvaluator` extends `ExprEvaluatorBase<FloatExprEvaluator>` and evaluates expressions of floating-point type into an `APFloat`:

```cpp
class FloatExprEvaluator : public ExprEvaluatorBase<FloatExprEvaluator> {
  APValue &Result;
  bool VisitFloatingLiteral(const FloatingLiteral *E);
  bool VisitBinaryOperator(const BinaryOperator *E);
  bool VisitCastExpr(const CastExpr *E);
  bool VisitCallExpr(const CallExpr *E);
};
```

`FloatExprEvaluator::VisitBinaryOperator` dispatches on the opcode. For `+`, `-`, `*`, `/` it calls the corresponding `APFloat` method (`add`, `subtract`, `multiply`, `divide`) with the `roundingMode` obtained from the `FPOptions` attached to the binary operator node. In C++ constant expressions, `FPOptions` always uses `rmNearestTiesToEven` regardless of any `-ffast-math` or `#pragma STDC FENV_ROUND` settings — the standard mandates that floating-point constant evaluation is exact per IEEE 754 defaults.

### 35.8.2 Cast Handling

```cpp
static bool HandleFloatToFloatCast(EvalInfo &Info, const Expr *E,
                                    QualType SrcType, QualType DestType,
                                    APFloat &Result) {
  bool Ignored;
  Result.convert(Ctx.getFloatTypeSemantics(DestType),
                 rmNearestTiesToEven, &Ignored);
  return true;
}
```

`APFloat::convert` takes an `fltSemantics` reference — `APFloat::IEEEhalf()`, `IEEEsingle()`, `IEEEdouble()`, `IEEEquad()`, or `x87DoubleExtended()` — matching the LLVM type semantics for the target's `half`, `float`, `double`, `long double`. The `Ignored` bool captures whether the conversion was inexact; the evaluator does not treat inexactness as a failure because IEEE 754 rounding is part of the abstract machine semantics.

### 35.8.3 `__builtin_*` Math Functions in `constexpr`

The C++ standard library `<cmath>` functions like `std::sqrt`, `std::abs`, `std::floor` are `constexpr` in C++23 when the argument is a constant. Clang implements these by mapping them to `__builtin_sqrtf`, `__builtin_sqrt`, etc. in the headers. The evaluator handles these builtins explicitly:

```cpp
case Builtin::BI__builtin_fabs:
case Builtin::BI__builtin_fabsf:
case Builtin::BI__builtin_fabsl:
  // Result.makeAbsolute();
  return true;

case Builtin::BI__builtin_sqrt:
case Builtin::BI__builtin_sqrtf:
  // Result.convert(), then APFloat::sqrt approximation
  return true;
```

The `APFloat::fusedMultiplyAdd` method implements `std::fma` exactly per IEEE 754-2008 clause 5.4.1, which is why `std::fma` in a `constexpr` context can produce results that differ from `a * b + c` (which rounds twice).

### 35.8.4 `-ffast-math` Is Never Applied in `constexpr` Contexts

The `FPOptions` node attached to each floating-point binary operator records the fast-math flags in effect at that source location. However, the evaluator ignores all relaxation flags when running in `EM_ConstantExpression` mode. The C++ standard requires that floating-point constant expressions be evaluated as if by the abstract machine with IEEE 754 round-to-nearest-ties-to-even, so `clang -ffast-math` cannot change the value of a `constexpr float` computation.

---

## 35.9 Aggregate and Composite Initialization

### 35.9.1 `ArrayExprEvaluator`

Arrays are evaluated by `ArrayExprEvaluator`, which inherits `ExprEvaluatorBase<ArrayExprEvaluator>`. The result is an `APValue` with kind `Array`:

```cpp
class ArrayExprEvaluator : public ExprEvaluatorBase<ArrayExprEvaluator> {
  const LValue &This;
  APValue &Result;
  bool VisitInitListExpr(const InitListExpr *E);
  bool VisitCXXConstructExpr(const CXXConstructExpr *E);
  bool VisitCallExpr(const CallExpr *E);
};
```

For an `InitListExpr`, the evaluator first creates an `APValue(UninitArray{}, NumInit, ArrSize)`. It then iterates over the explicitly provided initializers, evaluating each into `Result.getArrayInitializedElt(I)`. If fewer initializers are given than the array size, the remaining slots are covered by the *filler*, which is evaluated from the last initializer if present (for `= {0}` style), or zero-initialized. This avoids storing millions of `APValue(APSInt(0))` entries for large zero-initialized arrays.

### 35.9.2 `RecordExprEvaluator`

Struct and class objects are evaluated by `RecordExprEvaluator`. For aggregate initialization from an `InitListExpr`, it creates an `APValue(UninitStruct{}, NumBases, NumFields)` and iterates over fields:

```cpp
bool RecordExprEvaluator::VisitInitListExpr(const InitListExpr *E) {
  // ...
  for (FieldDecl *FD : RD->fields()) {
    LValue Subobject = This;
    HandleLValueMember(Info, InitExpr, Subobject, FD, &Layout);
    // Evaluate initializer into Result.getStructField(FI)
  }
}
```

Virtual and non-virtual base class subobjects are initialized before fields; `HandleConstructorCall` handles this ordering when a non-aggregate constructor is present.

### 35.9.3 `std::array` and Standard Library Types

`std::array<T, N>` is a struct wrapping a C array member. In `constexpr` contexts, Clang evaluates it as a `Struct` with one field whose value is an `Array`. Because `std::array` has a trivial aggregate structure, no special treatment is required: the generic `RecordExprEvaluator` and `ArrayExprEvaluator` paths handle it naturally. The same applies to `std::pair`, `std::tuple` (in recent standard library versions with `constexpr` constructors), and `std::optional`.

### 35.9.4 Designated Initializers (C99 / C++20)

C99 designated initializers and C++20 designated initializers in aggregate classes are desugared by `Sema` into `InitListExpr` nodes with the designations recorded as field indices. By the time the constant evaluator sees the tree, the `InitListExpr` is already ordered to match the record's field declaration order, so no special handling is required in the evaluator.

---

## 35.10 `static_assert` and Concept Constraint Satisfaction

### 35.10.1 `static_assert` Evaluation

`Sema::ActOnStaticAssertDeclaration` is the entry point for `static_assert`:

```cpp
Decl *Sema::ActOnStaticAssertDeclaration(
    SourceLocation StaticAssertLoc,
    Expr *AssertExpr,
    Expr *AssertMessageExpr,
    SourceLocation RParenLoc);
```

Inside this function, Clang calls `EvaluateAsRValue` on the condition expression. If evaluation succeeds and the result is integer zero (false), the static assert fires. The message expression — a string literal in C++11/14/17, or (since C++26) an expression whose `.data()` and `.size()` are accessible as constant strings — is evaluated separately through `EvaluateAsConstantExpr` and stored in the `StaticAssertDecl` AST node. The diagnostic `diag::err_static_assert_expression_is_not_constant` is emitted via `VerifyIntegerConstantExpression` when the condition itself is not a valid constant expression.

### 35.10.2 `CheckConstraintSatisfaction`

Concept satisfaction is checked in `Sema::CheckConstraintSatisfaction`. The function decomposes a conjunction or disjunction of *atomic constraints* and evaluates each independently (the standard mandates that atomic constraints be checked in isolation to prevent short-circuit evaluation from masking errors):

```cpp
bool Sema::CheckConstraintSatisfaction(
    ConstrainedDeclOrNestedRequirement Entity,
    ArrayRef<AssociatedConstraint> AssociatedConstraints,
    const MultiLevelTemplateArgumentList &TemplateArgLists,
    SourceRange TemplateIDRange,
    ConstraintSatisfaction &Satisfaction,
    const ConceptReference *TopLevelConceptId = nullptr,
    Expr **ConvertedExpr = nullptr);
```

For each atomic constraint, after template argument substitution, the evaluator calls `EvaluateAsConstantExpr` with `EM_ConstantExpression`. The result is stored in `ConstraintSatisfaction::IsSatisfied`. Unsatisfied atomic constraints are recorded in `ConstraintSatisfaction::Details` as `UnsatisfiedConstraintRecord` entries (a variant of either a substitution failure diagnostic or the substituted expression alongside its evaluated false value). This record is what Clang uses to produce the verbose concept-not-satisfied diagnostics with the expression shown in the note.

---

## 35.11 `__builtin_*` Constant Evaluation

Clang's constant evaluator handles over 200 `__builtin_*` intrinsics directly, without generating a call or passing through the function call machinery. They are dispatched in `IntExprEvaluator::VisitCallExpr`, `FloatExprEvaluator::VisitCallExpr`, and the `LValueExprEvaluator`/`PointerExprEvaluator` via the `getBuiltinCallee()` switch.

### 35.11.1 `__builtin_constant_p`

```cpp
case Builtin::BI__builtin_constant_p: {
  if (EvaluateBuiltinConstantP(Info, Arg))
    return Success(true, E);
  if (Info.InConstantContext || Arg->HasSideEffects(Info.Ctx))
    return Success(false, E);
  Info.FFDiag(E, diag::note_invalid_subexpr_in_const_expr);
  return false;
}
```

`EvaluateBuiltinConstantP` attempts to evaluate the argument using `EM_ConstantFold`. If it succeeds, the builtin returns 1. The special case for `Info.InConstantContext` ensures that calling `__builtin_constant_p` from within a `constexpr` function during template instantiation returns false (since the optimizer might not yet have context to fold the argument).

### 35.11.2 `__builtin_bit_cast`

`__builtin_bit_cast(DestType, Src)` is a type-pun with defined semantics in C++20. The evaluator handles it in `handleRValueToRValueBitCast` by serializing the source `APValue` to its raw byte representation and then deserializing into the destination type. This works for aggregate types including `std::bit_cast<int>(3.14f)`.

### 35.11.3 `__builtin_memcpy`, `__builtin_memmove`, `__builtin_memcmp`

These are handled in the `PointerExprEvaluator` (for the return value) and directly in `IntExprEvaluator` (for `memcmp`). The evaluator reads the source bytes into a buffer by iterating through the `APValue` representation character-by-character, respecting the `LValuePathEntry` designator path to find each byte's value. The destination is written similarly. Out-of-bounds access, misaligned access, and overlap detection are all checked by the evaluator, producing diagnostics in constant-expression contexts.

### 35.11.4 `__builtin_strlen` and `__builtin_strcmp`

`__builtin_strlen` evaluates by navigating the `LValue` base to the string literal `APValue` (stored as `Array` of characters) and scanning until a null character is found:

```cpp
case Builtin::BI__builtin_strlen:
  // Scan string literal APValue for '\0'
  return Success(Length, E);
```

`__builtin_strcmp` similarly reads two character arrays elementwise, comparing `APSInt` values at each position until it finds a difference or a null terminator in both strings.

### 35.11.5 `__builtin_add_overflow`, `__builtin_sub_overflow`, `__builtin_mul_overflow`

These are handled in `IntExprEvaluator::VisitCallExpr` by performing the arithmetic in a larger integer type, then checking whether the result fits in the destination type:

```cpp
case Builtin::BI__builtin_add_overflow:
  // Evaluate LHS and RHS as APSInt
  // Compute result in widened type
  // Store result via the pointer argument using handleAssignment
  // Return Success(overflow_occurred, E)
```

The pointer argument is evaluated as an `LValue` and written through `handleAssignment`. The overflow flag is a `bool` stored as a 1-bit integer `APSInt`. All three variants (`add`, `sub`, `mul`) use the same dispatch path; `mul` uses a `2*bitwidth` intermediate.

### 35.11.6 `__builtin_expect` and `__builtin_unreachable`

`__builtin_expect(expr, expected_value)` evaluates in a constant context by simply returning the first argument's value; the hint is irrelevant at compile time. `__builtin_unreachable()` causes `FFDiag` with `note_constexpr_builtin_unreachable` — reaching it in a constant expression is undefined behavior and the evaluator must fail.

### 35.11.7 `__builtin_popcount`, `__builtin_clz`, `__builtin_ctz`

The bit-count and bit-scan builtins are handled directly:

```cpp
case Builtin::BI__builtin_popcount:
  return Success(Val.countPopulation(), E);

case Builtin::BI__builtin_clz:
  if (!Val) return Error(E);  // UB: clz(0)
  return Success(Val.countl_zero(), E);

case Builtin::BI__builtin_ctz:
  if (!Val) return Error(E);  // UB: ctz(0)
  return Success(Val.countr_zero(), E);
```

The C++23 `__builtin_ctzg` variant takes an optional fallback argument for the zero case:

```cpp
case Builtin::BI__builtin_ctzg:
  if (!Val) {
    if (Fallback) return Success(*Fallback, E);
    return Error(E);
  }
  return Success(Val.countr_zero(), E);
```

---

## 35.12 Calling the Evaluator from a Clang Tool

Tool authors working with `clang::ASTConsumer` or `clang::RecursiveASTVisitor` can invoke the constant evaluator directly on any `Expr*` node. The typical pattern for extracting an integer constant is:

```cpp
#include "clang/AST/Expr.h"
#include "clang/AST/ASTContext.h"

bool tryGetIntValue(const clang::Expr *E, const clang::ASTContext &Ctx,
                    llvm::APSInt &Out) {
  clang::Expr::EvalResult Result;
  SmallVector<PartialDiagnosticAt, 4> Notes;
  Result.Diag = &Notes;

  if (!E->EvaluateAsInt(Result, Ctx, clang::Expr::SE_NoSideEffects,
                         /*InConstantContext=*/false))
    return false;

  if (Result.HasSideEffects || Result.HasUndefinedBehavior)
    return false;  // folded but not a clean constant

  Out = Result.Val.getInt();
  return true;
}
```

For constexpr initializer evaluation (e.g., to determine the value of a `constexpr` variable for a static analysis check), use `VarDecl::evaluateValue()`:

```cpp
bool tryGetVarConstValue(const clang::VarDecl *VD, clang::APValue &Out) {
  if (!VD->isConstexpr() && !VD->hasAttr<clang::ConstInitAttr>())
    return false;
  const clang::APValue *V = VD->evaluateValue();
  if (!V || V->isAbsent() || V->isIndeterminate())
    return false;
  Out = *V;
  return true;
}
```

`evaluateValue()` is safe to call multiple times; it caches the result in the `EvaluatedStmt` attached to the `VarDecl` (checking `WasEvaluated` before re-running). The function returns `nullptr` if evaluation failed, and a pointer to an `APValue(None)` if evaluation succeeded but produced a value Clang chose not to store (this distinction matters for diagnostics).

For tools that need to check whether an arbitrary expression is a valid core constant expression under the active language standard (for example, to verify user annotations), `EvaluateAsConstantExpr` provides conforming behavior:

```cpp
bool isConstantExpr(const clang::Expr *E, const clang::ASTContext &Ctx) {
  clang::Expr::EvalResult Result;
  return E->EvaluateAsConstantExpr(Result, Ctx,
                                    clang::ConstantExprKind::Normal);
}
```

This will return `false` for any expression that contains a read of a non-const variable, a call to a non-`constexpr` function, or anything else that the language standard declares non-constant, matching the diagnostics Sema would emit.

---

## 35.13 Diagnosing Non-Constant Expressions

The evaluator's diagnostic infrastructure distinguishes between two failure modes:

1. **Fold failure** (`FFDiag`): the expression cannot be folded at all. Examples: calling a non-`constexpr` function, reading a non-const variable, encountering a `goto`, calling a virtual function through a virtual dispatch that cannot be resolved.
2. **Core-constant-expression failure** (`CCEDiag`): the expression can be folded but the result is not a valid constant expression under language rules. Examples: modifying a global object, using a pointer to a string literal as an integer, reading an indeterminate value.

In `EM_ConstantFold` and `EM_IgnoreSideEffects` modes, `CCEDiag` produces no output (it is absorbed by the `OptionalDiagnostic` mechanism). In `EM_ConstantExpression` mode, both kinds are fatal and produce the note chain.

### 35.13.1 `EvalStatus::HasSideEffects` and `HasUndefinedBehavior`

`HasSideEffects` is set by `EvalInfo::noteSideEffect()` whenever the evaluator encounters an expression with a side-effect that it is folding past (via `SE_AllowSideEffects`). Callers that need a clean constant check `HasSideEffects` after evaluation and reject the result.

`HasUndefinedBehavior` is set when the evaluation path passes through an operation that is undefined behavior under the standard, but for which the evaluator has a policy of producing *some* result (for example, truncating `INT_MAX + 1` to `INT_MIN` on a two's-complement machine). Users of `EvaluateAsRValue` with the default `SE_NoSideEffects` policy will see `HasUndefinedBehavior = true` and can choose to discard the result.

### 35.13.2 C++20 Behaviors That Were Previously Silent

C++20 made several previously-silent undefined behaviors into explicitly diagnosed failures in `constexpr` contexts:

- **Reading an indeterminate value**: `APValue::Kind::Indeterminate` is now fatal. The diagnostic is `note_constexpr_access_uninit`.
- **Out-of-bounds array access**: previously silently produced a wrong result; now diagnosed via `note_constexpr_array_index`.
- **`reinterpret_cast`** in constant expressions: previously silently failed to fold; now produces `note_constexpr_invalid_cast` with a C++20-specific message.
- **Accessing an inactive union member**: `note_constexpr_access_inactive_union_member`.
- **Reading a `mutable` member**: `note_constexpr_access_mutable`.

---

## 35.14 The `ConstantExpr` AST Node

When the evaluator successfully evaluates an expression in a context that requires it to be constant, Clang can cache the result in a `ConstantExpr` wrapper node to avoid re-evaluation:

```cpp
class ConstantExpr final
    : public FullExpr,
      private llvm::TrailingObjects<ConstantExpr, APValue, uint64_t> {
public:
  static ConstantExpr *Create(const ASTContext &Context, Expr *E,
                               const APValue &Result);
  static ConstantExpr *Create(const ASTContext &Context, Expr *E,
                               ConstantResultStorageKind Storage =
                                   ConstantResultStorageKind::None,
                               bool IsImmediateInvocation = false);

  APValue::ValueKind getResultAPValueKind() const;
  ConstantResultStorageKind getResultStorageKind() const;
  bool isImmediateInvocation() const;
  bool hasAPValueResult() const;
  APValue getAPValueResult() const;
  llvm::APSInt getResultAsAPSInt() const;
};
```

`ConstantExpr` stores the result in one of three ways, controlled by `ConstantResultStorageKind`:
- `None`: no result stored (the node is a marker that the subexpression is constant, but the value is not cached).
- `Int64`: the result fits in a `uint64_t` and is stored inline as a trailing object.
- `APValue`: a full `APValue` trailing object for complex results (floats, lvalues, aggregates).

`ConstantExpr` nodes appear in the AST wherever a constant expression has been evaluated:
- Template non-type arguments (all three storage kinds appear depending on argument type).
- Immediate invocations (`consteval` calls).
- Enum member definitions after C++17 (where the evaluated value is preserved).
- Case expressions whose value has been verified.

The `ConstantResultStorageKind::Int64` optimization matters for performance: the majority of constant expressions (array bounds, enum values, case labels) fit in 64 bits, avoiding heap allocation of a full `APValue`.

---

## Chapter Summary

- Clang's constant evaluator lives entirely in `clang/lib/AST/ExprConstant.cpp` and is exposed through methods on `Expr`: `EvaluateAsRValue`, `EvaluateAsInt`, `EvaluateAsFloat`, `EvaluateAsLValue`, `EvaluateAsInitializer`, and `EvaluateAsConstantExpr`.
- All evaluated constants are represented as `APValue`, a discriminated union over `None`, `Indeterminate`, `Int`, `Float`, `FixedPoint`, `ComplexInt`, `ComplexFloat`, `LValue`, `Vector`, `Array`, `Struct`, `Union`, `MemberPointer`, and `AddrLabelDiff`.
- `LValue`-kind `APValues` use `LValueBase` (a `PointerUnion` over `ValueDecl*`, `Expr*`, `TypeInfoLValue`, and `DynamicAllocLValue`) plus an `LValuePathEntry` array as a subobject designator.
- `EvalInfo` is the internal context object carrying the call stack (`CallStackFrame` linked list with local variables in `std::map<MapKeyTy, APValue> Temporaries`), the evaluation mode (`EM_ConstantExpression`, `EM_ConstantFold`, `EM_IgnoreSideEffects`, `EM_ConstantExpressionUnevaluated`), and diagnostic state.
- The CRTP-based evaluator hierarchy (`ExprEvaluatorBase<Derived>`) has concrete subclasses for integers (`IntExprEvaluator`), floats (`FloatExprEvaluator`), lvalues (`LValueExprEvaluator`), arrays (`ArrayExprEvaluator`), and records (`RecordExprEvaluator`).
- `constexpr` function calls go through `HandleFunctionCall` → `CallStackFrame` → statement visitor; `consteval` calls use `ConstantExprKind::ImmediateInvocation`.
- Floating-point evaluation always uses IEEE 754 default rounding regardless of `-ffast-math`; `APFloat::convert()` handles cross-format casts.
- `static_assert` is backed by `EvaluateAsRValue`; concept satisfaction uses `EvaluateAsConstantExpr` per atomic constraint with results in `ConstraintSatisfaction::Details`.
- Over 200 `__builtin_*` functions receive specialized evaluator treatment; `__builtin_constant_p` and `__builtin_is_constant_evaluated` are the two that change the evaluation's own behavior.
- `ConstantExpr` AST nodes cache evaluated results as trailing `Int64` or `APValue` objects to avoid re-evaluation in template instantiation and code generation.

---

*Cross-references: [Chapter 33 — Semantic Analysis and the Type Checker](../part-05-clang-frontend/ch33-semantic-analysis.md) — Sema uses `VerifyIntegerConstantExpression` to validate ICEs; [Chapter 34 — Templates and Template Instantiation](../part-05-clang-frontend/ch34-templates.md) — concept satisfaction calls `CheckConstraintSatisfaction`; [Chapter 36 — The AST: Expressions and Statements](../part-05-clang-frontend/ch36-ast-expressions.md) — `ConstantExpr` node placement in the AST; [Chapter 15 — Dependent Types and Type Deduction](../part-03-type-theory/ch15-dependent-types.md) — type-level constant expressions in template arguments.*

**Reference links:**
- [`clang/lib/AST/ExprConstant.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/AST/ExprConstant.cpp) — the entire evaluator implementation (~17,000 lines)
- [`clang/include/clang/AST/APValue.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/APValue.h) — `APValue` class and `LValueBase`
- [`clang/include/clang/AST/Expr.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Expr.h#L608) — `EvalStatus`, `EvalResult`, `SideEffectsKind`, `ConstantExprKind`, `ConstantExpr`
- [`clang/include/clang/AST/Decl.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Decl.h#L887) — `EvaluatedStmt`, `VarDecl::evaluateValue`
- [`clang/include/clang/Sema/Sema.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/Sema.h#L7727) — `Sema::VerifyIntegerConstantExpression`, `CheckConstraintSatisfaction`
- [`clang/include/clang/AST/ASTConcept.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/ASTConcept.h#L47) — `ConstraintSatisfaction`
- [`clang/include/clang/Basic/DiagnosticASTKinds.inc`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/DiagnosticASTKinds.inc) — all `note_constexpr_*` diagnostic IDs
- [`llvm/include/llvm/ADT/APFloat.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ADT/APFloat.h) — `APFloat`, `fltSemantics`, `fusedMultiplyAdd`


---

@copyright jreuben11
