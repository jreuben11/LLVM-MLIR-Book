# Chapter 34 — Sema II: Templates, Concepts, and Constraints

*Part V — Clang Internals: Frontend Pipeline*

C++ templates are the most semantically complex feature in the language standard, and Clang's implementation reflects that complexity in roughly 50,000 lines of code spread across `SemaTemplate.cpp`, `SemaTemplateInstantiate.cpp`, `SemaTemplateDeduction.cpp`, `SemaTemplateVariadic.cpp`, and `SemaConcept.cpp`. This chapter dissects the entire pipeline: how template declarations are represented in the AST, how the parser feeds actions into Sema, how argument deduction and substitution work, how instantiation proceeds lazily through a pending-work queue, and how the C++20 constraint system is evaluated. Understanding this machinery is prerequisite for anyone writing tools that analyze C++ templates, extending Clang with new constructs, or diagnosing instantiation failures.

The sheer breadth of the implementation reflects the standard's complexity: template argument deduction alone specifies over 20 distinct failure modes, SFINAE interacts with every overload-resolution candidate, constraint normalization introduces a non-trivial equivalence relation over syntactic expressions, and variadic templates require every transformation pass to be extended to handle pack expansions. The following sections treat each subsystem in depth, with concrete API signatures drawn from Clang 22.1.x headers and internal source files.

---

## Template Representation in the AST

### The TemplateDecl Hierarchy

Every template declaration in Clang is represented by a node that inherits from `TemplateDecl`, declared in [`clang/AST/DeclTemplate.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/DeclTemplate.h). `TemplateDecl` is a `NamedDecl` that owns a `TemplateParameterList*` and a `NamedDecl*` templated declaration:

```cpp
class TemplateDecl : public NamedDecl {
  TemplateParameterList *TemplateParams;
  NamedDecl            *TemplatedDecl;
public:
  TemplateParameterList *getTemplateParameters() const { return TemplateParams; }
  NamedDecl             *getTemplatedDecl()       const { return TemplatedDecl; }
};
```

The concrete subclasses used throughout compilation are:

| Class | Template form | Redeclarable? |
|---|---|---|
| `FunctionTemplateDecl` | `template<…> void f(…)` | Yes |
| `ClassTemplateDecl` | `template<…> class C { }` | Yes |
| `VarTemplateDecl` | `template<…> T x = …` | Yes |
| `TypeAliasTemplateDecl` | `template<…> using A = …` | Yes |
| `ConceptDecl` | `template<…> concept C = …` | No |

All except `ConceptDecl` inherit from `RedeclarableTemplateDecl`, which provides the standard redeclaration chain (`getPreviousDecl()`, `getMostRecentDecl()`, `getFirstDecl()`) and maintains a `FoldingSet` of specializations keyed on template arguments.

`ClassTemplateDecl` and `VarTemplateDecl` return `CXXRecordDecl` and `VarDecl` respectively from `getTemplatedDecl()`. A `FunctionTemplateDecl` wraps a `FunctionDecl`. These `Decl` nodes retain a back-pointer: `FunctionDecl::getDescribedFunctionTemplate()` returns the owning `FunctionTemplateDecl`, and `CXXRecordDecl::getDescribedClassTemplate()` the owning `ClassTemplateDecl`.

### TemplateParameterList and Parameter Kinds

`TemplateParameterList` is a fixed-size trailing-objects structure that stores an array of `NamedDecl*` parameters plus an optional `requires`-clause expression:

```cpp
class TemplateParameterList final
    : private llvm::TrailingObjects<TemplateParameterList,
                                    NamedDecl *, Expr *> {
  SourceLocation TemplateLoc, LAngleLoc, RAngleLoc;
  unsigned NumParams : 29;
  unsigned HasRequiresClause : 1;
  unsigned HasConstrainedParameters : 1;
  // ...
public:
  static TemplateParameterList *Create(const ASTContext &C,
                                        SourceLocation TemplateLoc,
                                        SourceLocation LAngleLoc,
                                        ArrayRef<NamedDecl *> Params,
                                        SourceLocation RAngleLoc,
                                        Expr *RequiresClause);
  Expr *getRequiresClause() const;
  ArrayRef<NamedDecl *> asArray() const;
};
```

Each parameter occupies one of three leaf classes:

- **`TemplateTypeParmDecl`** — represents `typename T` or `class T`. Carries a depth/index pair and, for C++20, a `TypeConstraint*` for constrained auto parameters.
- **`NonTypeTemplateParmDecl`** — represents `int N`, `auto V` (C++17), or `T* P`. The parameter type may itself be dependent when an earlier type parameter appears in it.
- **`TemplateTemplateParmDecl`** — represents `template<…> class TT`. The nested `TemplateParameterList` mirrors the structure expected of any template argument bound to this parameter.

Parameter packs are signalled by `isParameterPack()` on any of these three classes. The depth (number of enclosing template scopes) and index within the innermost scope together uniquely identify each parameter and appear in `TemplateTypeParmType`, `SubstTemplateTypeParmType`, and `SubstTemplateTypeParmPackType` nodes in the type system.

### TemplateArgument — The Discriminated Union

`TemplateArgument`, in [`clang/AST/TemplateBase.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/TemplateBase.h), is a value-type discriminated union over nine possible kinds:

```
Null              — uninitialized / error sentinel
Type              — QualType; used for type parameters
Integral          — APSInt + type; used for NTTP integral values
Declaration       — pointer/reference NTTP bound to a ValueDecl
NullPtr           — null-pointer NTTP
StructuralValue   — structural value (C++20) via APValue
Template          — TemplateName; for template template parameters
TemplateExpansion — pack expansion of a TemplateName
Expression        — Expr*; for value-dependent NTTPs
Pack              — ArrayRef<TemplateArgument>; for parameter packs
```

Accessing any field on the wrong kind is a contract violation; the union is manipulated exclusively through typed constructors (`TemplateArgument(QualType)`, `TemplateArgument(const ASTContext&, QualType, const APValue&, bool)`, etc.) and kind-checked accessors (`getAsType()`, `getAsIntegral()`, `pack_begin()`/`pack_end()`).

`TemplateArgumentList` is a separate class that owns a heap-allocated array of `TemplateArgument` values. It is created via `TemplateArgumentList::CreateCopy()` and is attached to specialization nodes such as `ClassTemplateSpecializationDecl` and `FunctionTemplateSpecializationInfo`.

---

## Template Parsing and Declaration

### ParseTemplateDeclaration

When the parser encounters `template`, `Parser::ParseTemplateDeclaration()` in `ParseTemplate.cpp` takes control. It calls `ParseTemplateParameters()` to consume the `<` … `>` and construct `TemplateParameterList` nodes via `Actions.ActOnTemplateParameterList()`. Each parameter kind has its own parse entry:

- `ParseTypeParameter()` → `Sema::ActOnTypeParameter()`
- `ParseNonTypeTemplateParameter()` → `Sema::ActOnNonTypeTemplateParameter()`
- `ParseTemplateTemplateParameter()` → `Sema::ActOnTemplateTemplateParameter()`

After the parameter list, `ParseTemplateDeclaration()` hands off to `ParseDeclaration()`, which in turn calls `ParseClassTemplate()` or simply proceeds with the function/variable declaration. Once the full declarator is available, `Sema::ActOnTemplateDeclarator()` is called:

```cpp
Decl *Sema::ActOnTemplateDeclarator(Scope *S,
                                    MultiTemplateParamsArg TemplateParameterLists,
                                    Declarator &D);
```

This method matches the incoming parameter lists against the declaration nesting, creates the appropriate `TemplateDecl` node, and performs initial constraint checking. For class templates specifically, `Sema::ActOnClassTemplate()` creates a `ClassTemplateDecl` and attaches the injected class name specialization via `getInjectedClassNameSpecialization()`, which returns the `ClassTemplateSpecializationDecl` that represents `C<T1, T2, …>` inside the template body.

### Member Templates and Local Classes

A member template is parsed as a nested declaration of a class template body. The `TemplateParameterList` is pushed onto `Sema::TemplateParamLists`, and when the enclosing class is later instantiated, the member template's parameters are instantiated separately — the class template arguments only flow in through the multi-level argument list, not through the member template's own parameters.

Local classes inside function templates present a subtlety: they capture the function template's parameter pack state and must be re-instantiated for each specialization of the enclosing function. Clang tracks local-class templates as `CXXRecordDecl` nodes whose `isLocalClass()` returns true.

---

## Template Instantiation — Core Mechanism

### The Instantiation Entry Points

Clang instantiates function templates through `Sema::InstantiateFunctionDefinition()` (declared in `Sema.h`, implemented in `SemaTemplateInstantiate.cpp`):

```cpp
void Sema::InstantiateFunctionDefinition(
    SourceLocation PointOfInstantiation,
    FunctionDecl *Function,
    bool Recursive = false,
    bool DefinitionRequired = false,
    bool AtEndOfTU = false);
```

This builds the `MultiLevelTemplateArgumentList` from the specialization's argument list, constructs a `LocalInstantiationScope`, and calls `SubstFunctionBody()` which drives the `TemplateInstantiator` tree transform. For class templates, `Sema::InstantiateClass()` (called by `Sema::RequireCompleteType()` when the instantiation is needed) iterates over the pattern class's members and instantiates each via `Sema::InstantiateClassMembers()`.

### MultiLevelTemplateArgumentList

Instantiating a member of a class template inside another class template requires arguments at multiple depths simultaneously. `MultiLevelTemplateArgumentList` in [`clang/Sema/Template.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/Template.h) holds a `SmallVector<ArgumentListLevel, 4>` stored innermost-first:

```cpp
class MultiLevelTemplateArgumentList {
  SmallVector<ArgumentListLevel, 4> TemplateArgumentLists;
  unsigned NumRetainedOuterLevels = 0;
  TemplateSubstitutionKind Kind = TemplateSubstitutionKind::Specialization;
public:
  unsigned getNumLevels() const;
  const TemplateArgument &operator()(unsigned Depth, unsigned Index) const;
  void addOuterTemplateArguments(Decl *D, ArgList Args, bool Final);
  void setKind(TemplateSubstitutionKind K);
};
```

When instantiating `X<int>::Y<17>::f()`, depth 0 gets `{int}` and depth 1 gets `{17}`. `getNewDepth()` computes how depths shift as the retained outer levels absorb argument lists.

### InstantiatingTemplate RAII Guard and the CodeSynthesisContext Stack

Every recursive instantiation entry is bracketed by an `InstantiatingTemplate` RAII object:

```cpp
Sema::InstantiatingTemplate Inst(
    *this, PointOfInstantiation, FunctionDecl);
if (Inst.isInvalid())
    return; // depth limit exceeded
```

Construction pushes a `CodeSynthesisContext` onto `Sema::CodeSynthesisContexts`, a `SmallVector<CodeSynthesisContext, 16>`. If the stack depth exceeds `LangOptions::InstantiationDepth` (defaulting to 1024, overridable via `-ftemplate-depth=N`), construction emits a fatal diagnostic and sets `Invalid = true`. The destructor calls `Clear()`, which pops the context. This mechanism detects infinite recursive instantiation (e.g., `template<int N> struct F : F<N-1>`) before the compiler runs out of native stack space.

`CodeSynthesisContext::SynthesisKind` enumerates every reason Clang might be synthesizing code. The key values include:

| Kind | Triggered by |
|---|---|
| `TemplateInstantiation` | Normal instantiation of a class/function/var template |
| `DefaultTemplateArgumentInstantiation` | Evaluating a default template argument |
| `ExplicitTemplateArgumentSubstitution` | Substituting explicit template arguments |
| `DeducedTemplateArgumentSubstitution` | Substituting deduced arguments after deduction succeeds |
| `LambdaExpressionSubstitution` | Substituting into a lambda appearing inside a template |
| `RequirementInstantiation` | Instantiating a requirement inside a `requires { }` body |
| `NestedRequirementConstraintsCheck` | Checking satisfaction of a nested requirement |
| `ConstraintsCheck` | Checking constraints of a template or concept |
| `ConstraintSubstitution` | Substituting template args into a constraint expression |
| `ConstraintNormalization` | Normalizing a constraint into ANF/DNF |
| `ParameterMappingSubstitution` | Substituting a parameter mapping for an atomic constraint |
| `BuildingDeductionGuides` | Synthesizing CTAD deduction guides from constructors |
| `PartialOrderingTTP` | Partial ordering template template parameters |

Each `CodeSynthesisContext` records the `SynthesisKind`, `SourceLocation PointOfInstantiation`, `Decl *Entity`, and the template arguments in effect. The stack is traversed by `Sema::PrintInstantiationStack()` to emit the "in instantiation of…" note chain, and by tooling via the `TemplateInstantiationCallback` interface.

The `InstantiatingTemplate` struct provides a constructor overload for each kind. For example, recording a constraint check:

```cpp
InstantiatingTemplate(Sema &SemaRef, SourceLocation PointOfInstantiation,
                      ConstraintsCheck, NamedDecl *Template,
                      ArrayRef<TemplateArgument> TemplateArgs,
                      SourceRange InstantiationRange);
```

All overloads share the same push/pop logic; the kind tag determines how the context appears in the diagnostic note chain.

---

## Template Argument Deduction

### DeduceTemplateArguments for Function Templates

The primary entry point for function template argument deduction is:

```cpp
TemplateDeductionResult Sema::DeduceTemplateArguments(
    FunctionTemplateDecl *FunctionTemplate,
    TemplateArgumentListInfo *ExplicitTemplateArgs,
    ArrayRef<Expr *> Args,
    FunctionDecl *&Specialization,
    sema::TemplateDeductionInfo &Info,
    bool PartialOverloading,
    bool PartialOrdering,
    QualType ObjectType,
    Expr::Classification ObjectClassification,
    llvm::function_ref<bool(ArrayRef<QualType>)> CheckNonDependent = {});
```

This function sets up a `SFINAETrap` to capture substitution failures, calls `DeduceTemplateArgumentsFromCallArguments()` for each function argument, then invokes `FinishTemplateArgumentDeduction()` to substitute and check constraints. On success, `Specialization` is set to the instantiated or found `FunctionDecl`.

Internal deduction walks type-pairs recursively through `DeduceTemplateArgumentsByTypeMatch()`, which handles `TemplateTypeParmType` (direct substitution), `PointerType`, `ReferenceType`, `FunctionProtoType`, `TemplateSpecializationType`, `PackExpansionType`, and every other composite type. The deduction result at each step is accumulated in a `SmallVector<DeducedTemplateArgument>` indexed by parameter index.

### TemplateDeductionResult

`Sema::DeduceTemplateArguments()` returns a `TemplateDeductionResult` enum value indicating the precise failure mode. The full enumeration, declared in `clang/Sema/Sema.h`, distinguishes cases that callers must handle differently:

| Value | Meaning |
|---|---|
| `Success` | All parameters deduced; specialization is valid |
| `Incomplete` | At least one template parameter has no deduced value |
| `IncompletePack` | A pack expansion has an undetermined element count |
| `Inconsistent` | Two arguments produced conflicting deductions for the same parameter |
| `Underqualified` | cv-qualifiers mismatch during type deduction |
| `SubstitutionFailure` | Substituting deduced args produced an invalid type/expr |
| `DeducedMismatch` | Deduced args applied to a dependent parameter do not match the argument |
| `NonDeducedMismatch` | A non-dependent component of the parameter mismatches the argument |
| `TooManyArguments` / `TooFewArguments` | Wrong number of call arguments for function template |
| `InvalidExplicitArguments` | Explicit template arguments not valid for the template |
| `ConstraintsNotSatisfied` | Deduced args satisfy the type but not the associated constraints |
| `MiscellaneousDeductionFailure` | Catch-all for unlisted failures |

Distinguishing `SubstitutionFailure` from `ConstraintsNotSatisfied` matters for SFINAE: the former is a classic soft error (drops the candidate silently); the latter is also silent during overload resolution but produces a better-quality diagnostic when no other candidates succeed.

### TemplateDeductionInfo

`sema::TemplateDeductionInfo` (in [`clang/Sema/TemplateDeduction.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/TemplateDeduction.h)) accompanies every deduction attempt and records:

- The deduced argument lists (sugared and canonical), stored as `TemplateArgumentList *DeducedSugared` and `TemplateArgumentList *DeducedCanonical`.
- Whether an SFINAE diagnostic was suppressed (`HasSFINAEDiagnostic`).
- The `SmallVector<PartialDiagnosticAt>` of suppressed diagnostics that would be emitted if this candidate eventually is selected.
- A `SourceLocation` of the deduction point and `unsigned DeducedDepth` indicating which template parameter depth is being deduced.
- `unsigned ExplicitArgs`, the count of explicitly-specified arguments that bypass deduction.
- A `bool StrictPackMatch` flag set when a pack on the parameter side matched non-packs on the argument side.

When deduction fails, the caller can query `Info.getDeducedSugared()`, `Info.getExplicitArgs()`, and the suppressed diagnostics to produce a detailed note explaining the failure. The `TemplateDeductionInfo::ForBase` constructor tag creates a temporary copy for speculative deduction against base classes of an argument type, keeping the primary info clean.

### Class Template Argument Deduction (CTAD, C++17)

For `std::vector v{1, 2, 3};`, Clang invokes:

```cpp
QualType Sema::DeduceTemplateSpecializationFromInitializer(
    TypeSourceInfo *TInfo,
    const InitializedEntity &Entity,
    const InitializationKind &Kind,
    MultiExprArg Inits);
```

This method collects all deduction guides for the class template (user-written guides appear as `FunctionTemplateDecl` nodes in the class scope; Clang synthesizes implicit guides from constructors and, in C++23, from aggregate member types). Each guide is a function template whose return type is the specialization. The best matching guide is selected through the normal overload resolution machinery, and the winning return type gives the deduced specialization. Deduction guides are built (or regenerated from constructors) by `Sema::DeclareImplicitDeductionGuides()`.

### Class Template Argument Deduction — Deduction Guides

The deduction guide mechanism deserves further detail. Clang synthesizes implicit deduction guides from each constructor of the primary template and from any inherited constructors. The synthetic guide is a `FunctionTemplateDecl` whose return type is the specialization `C<deduced-args>` and whose parameter types mirror the constructor's. User-written deduction guides are explicit `FunctionTemplateDecl` nodes in the class's enclosing namespace.

For `template<typename T> struct Pair { Pair(T, T); };` Clang synthesizes:

```cpp
// Implicit guide generated by Sema::DeclareImplicitDeductionGuides():
template<typename T>
Pair<T> __deduction_guide(T, T);
```

When the user writes `Pair p{1, 2.0};`, CTAD invokes overload resolution over all deduction guides with `{1, 2.0}` as arguments. If multiple guides match, the most constrained or most specialized wins via the same partial ordering used for function templates. In C++23, aggregate deduction guides further extend this by generating guides from aggregate member types.

### Partial Ordering of Function Templates

When overload resolution finds multiple viable function template specializations, Clang calls `Sema::getMoreSpecializedTemplate()` which drives partial ordering by deducing one template's parameter types from the other's argument types. The two deductions are:

1. Deduce the parameter types of `F2` from the argument types of `F1`.
2. Deduce the parameter types of `F1` from the argument types of `F2`.

If deduction succeeds in one direction but not the other, the template that succeeded in the forward direction is more specialized. `isMoreSpecializedThanPrimary()` handles the related but distinct question of whether a partial specialization is more specialized than the primary template. The outcome feeds back into overload ranking via `FunctionTemplateDecl`'s specialization info stored in `FunctionTemplateSpecializationInfo`.

---

## Substitution and SFINAE

### The Substitution Entry Points

Template substitution is performed by a family of `Subst*` methods on `Sema`:

```cpp
TypeSourceInfo *SubstType(TypeSourceInfo *T,
                           const MultiLevelTemplateArgumentList &Args,
                           SourceLocation Loc,
                           DeclarationName Entity);

QualType SubstType(QualType T,
                   const MultiLevelTemplateArgumentList &Args,
                   SourceLocation Loc,
                   DeclarationName Entity);

ExprResult SubstExpr(Expr *E,
                     const MultiLevelTemplateArgumentList &Args);

Decl *SubstDecl(Decl *D, DeclContext *Owner,
                const MultiLevelTemplateArgumentList &Args);
```

These are thin wrappers that create a `TemplateInstantiator` (a `TreeTransform<TemplateInstantiator>` subclass) and call the appropriate `Transform*` method. The multi-level argument list tells the transform how to replace `TemplateTypeParmType` at each (depth, index) with the concrete argument.

### SFINAE Mechanics

Substitution Failure Is Not An Error (SFINAE) is implemented by wrapping deduction in a `SFINAETrap`:

```cpp
class SFINAETrap : SFINAEContextBase {
  bool HasErrorOcurred   = false;
  bool WithAccessChecking = false;
  sema::TemplateDeductionInfo *DeductionInfo = nullptr;
  // ...
public:
  SFINAETrap(Sema &S, sema::TemplateDeductionInfo &Info);
  bool hasErrorOccurred() const { return HasErrorOcurred; }
};
```

While a `SFINAETrap` is active (`Sema::isSFINAEContext()` returns true), the diagnostic engine does not emit errors; instead errors are classified by `Sema::isSFINAEContext()` and forwarded to `TemplateDeductionInfo::addSuppressedDiagnostic()`. The current trap is stored in `Sema::CurrentSFINAEContext`.

The distinction between "soft" and "hard" errors matters enormously:

- **Soft errors** (substitution failures proper): type mismatch during deduction, access violation in substitution, incomplete types in SFINAE context — these cause the overload candidate to be silently dropped.
- **Hard errors**: any diagnostic that would always be wrong regardless of the substitution context — undefined behavior, ill-formed program diagnostics — escape the trap and become real compiler errors.

`NonSFINAEContext` is the complementary RAII type that temporarily disables SFINAE checking even inside a deduction:

```cpp
struct NonSFINAEContext : SFINAEContextBase {
  NonSFINAEContext(Sema &S) : SFINAEContextBase(S, nullptr) {}
};
```

The canonical `enable_if<Cond, T>` idiom works because `enable_if<false, T>::type` does not exist, so `SubstType()` of a type depending on `enable_if<false,T>::type` fails silently and removes the overload. The `void_t` pattern works because `void_t<ill_formed_expr>` fails during substitution of `ill_formed_expr` into the template argument position. The detection idiom generalizes this to `std::is_detected`, where a primary template returns `std::false_type` and a specialization using `void_t<expr>` returns `std::true_type` only when the expression is valid.

An important SFINAE boundary: SFINAE applies only to the *immediate context* of the substitution — the function parameter and return types, the template parameter constraints, and the function's `noexcept` specifier. Errors that arise inside a function template's body (after substitution is complete) are hard errors, not SFINAE. This is why `std::enable_if` must appear in the function signature, not inside the body. Clang enforces this boundary through the `SFINAETrap` active-depth check: if `DeduceTemplateArguments()` completes and then a body instantiation fails, it is too late for SFINAE to suppress the error.

Substitution into constraints (C++20 `requires`-clauses) is *not* SFINAE in the C++17 sense, but it has a similar soft-error effect: a constraint that evaluates to `false` causes the template to be excluded from overload consideration without an error. A template with an unsatisfied constraint is removed from the candidate set exactly as a substitution failure would be, but the mechanism is the constraint satisfaction system rather than the SFINAE trap.

---

## Explicit and Partial Specializations

### ClassTemplateSpecializationDecl

Full explicit specializations of class templates create `ClassTemplateSpecializationDecl` nodes, which inherit from `CXXRecordDecl` and carry:

- `TemplateArgumentList *SpecializationArgs` — the template arguments identifying this specialization.
- `TemplateSpecializationKind SpecializationKind` — one of `TSK_Undeclared`, `TSK_ImplicitInstantiation`, `TSK_ExplicitSpecializationDeclaration`, `TSK_ExplicitSpecializationDefinition`, `TSK_ExplicitInstantiationDeclaration`, `TSK_ExplicitInstantiationDefinition`.

Partial specializations are `ClassTemplatePartialSpecializationDecl`, which additionally holds its own `TemplateParameterList` (the partial parameters, not the original template's parameters) and a `TemplateArgumentList` describing how the partial specialization's parameters map to the primary template's arguments.

### Specialization Registration

Every `ClassTemplateDecl` maintains a `FoldingSet<ClassTemplateSpecializationDecl>` in its common data. When `Sema::ActOnClassTemplateSpecialization()` fires, it looks up the argument list in that folding set. A match returns the existing node (which may be a forward declaration awaiting a definition); no match allocates a fresh specialization.

`FunctionTemplateSpecializationInfo` plays the analogous role for function templates:

```cpp
class FunctionTemplateSpecializationInfo final
    : private llvm::TrailingObjects<FunctionTemplateSpecializationInfo,
                                    MemberSpecializationInfo *,
                                    ASTTemplateArgumentListInfo *> {
  llvm::PointerIntPair<FunctionTemplateDecl *, 2> Template;
  TemplateArgumentList      *TemplateArguments;
  TemplateSpecializationKind SpecializationKind;
  // ...
};
```

### Explicit Instantiation

`extern template class vector<int>;` → `Sema::ActOnExplicitInstantiation()` with an extern `SourceLocation`. This creates or finds a `ClassTemplateSpecializationDecl` and sets its kind to `TSK_ExplicitInstantiationDeclaration`, suppressing implicit instantiation in this translation unit. The explicit instantiation definition (`template class vector<int>;` without `extern`) sets `TSK_ExplicitInstantiationDefinition` and triggers immediate instantiation via `Sema::InstantiateClassDefinition()`.

Out-of-line member definitions of specializations are handled in `Sema::CheckMemberSpecialization()`, which verifies that the out-of-line definition matches a previously declared explicit specialization and sets the definition pointer on the canonical specialization node.

---

## Variadic Templates

### Parameter Pack Representation

A type parameter declared `typename... Ts` sets `TemplateTypeParmDecl::isParameterPack()` to true. In the type system, unexpanded packs appear as `PackExpansionType`, which wraps a `Pattern` type and an optional expansion count (`std::optional<unsigned> NumExpansions`). The corresponding expression node is `PackExpansionExpr`.

`sizeof...(Pack)` lowers to a `SizeOfPackExpr` node that stores the pack declaration. During instantiation it is replaced by an `IntegerLiteral` equal to the pack's size.

### Pack Expansion Checking

When the parser encounters a `...` at an expression or type position, `Sema::CheckParameterPacksForExpansion()` verifies that the ellipsis expands at least one unexpanded pack. It takes the unexpanded packs found in the expression (via `Sema::collectUnexpandedParameterPacks()`), checks that all have consistent sizes if the sizes are known at parse time, and determines whether the pack size can be determined at this point or must wait until instantiation. If multiple packs appear in a single expansion, they must all have the same size — Clang diagnoses mismatched pack sizes during instantiation in `TreeTransform::TransformPackExpansionType()`.

`Sema::collectUnexpandedParameterPacks()` is a utility that recursively collects all unexpanded pack references within a type or expression subtree, returning them as an `SmallVector<UnexpandedParameterPack>`. Each `UnexpandedParameterPack` is a `PointerUnion<const TemplateTypeParmType *, SubstTemplateTypeParmPackType *, const SubstTemplateTypeParmPackType *, NamedDecl *>` identifying the pack declaration. This utility is used to validate that a fold expression or pack expansion actually mentions a pack, and to identify which packs must be carried through as unexpanded in template patterns.

### Fold Expressions (C++17)

`(args + ...)` and `(... + args)` and the binary forms parse into `CXXFoldExpr` nodes. The internal representation uses a four-element `SubExprs` array (Callee, LHS, RHS, and a count slot) where the callee is used for operator overload resolution during instantiation:

```cpp
class CXXFoldExpr : public Expr {
  SourceLocation LParenLoc, EllipsisLoc, RParenLoc;
  UnsignedOrNone NumExpansions; // nullopt if unknown
  Stmt *SubExprs[4];            // SubExpr::{Callee,LHS,RHS,Count}
public:
  Expr *getCallee() const;      // UnresolvedLookupExpr for the operator
  Expr *getLHS() const;         // null for right-fold unary
  Expr *getRHS() const;         // null for left-fold unary
  BinaryOperatorKind getOperator() const;
  bool isRightFold() const;
  bool isLeftFold() const { return !isRightFold(); }
  UnsignedOrNone getNumExpansions() const { return NumExpansions; }
};
```

During instantiation, `TreeTransform::TransformCXXFoldExpr()` expands the pack, generates a binary-operator tree over the expanded elements, and folds left or right according to `isRightFold()`. For a left fold `(... + args)` over `{a, b, c}`, the tree becomes `((a + b) + c)`. Empty unary folds are handled for `&&`, `||`, and `,` operators (yielding `true`, `false`, and `void()` respectively per [expr.prim.fold]), and are ill-formed for all other operators. Empty binary folds substitute the init operand directly.

The `Callee` sub-expression is an `UnresolvedLookupExpr` for the fold operator, ensuring that operator overload resolution occurs in the instantiation context where argument types are known, not at fold-expression parse time. This correctly handles cases like `(container << ... << elements)` where `operator<<` should be looked up for the actual element type.

### Pack Indexing (C++26)

C++26 introduces `Ts...[I]` syntax, selecting the I-th element of a pack. Clang 22 represents this as a `PackIndexingExpr` AST node. During instantiation, the index is evaluated as a constant expression and the appropriate pack element is extracted.

---

## Concepts and Constraints (C++20)

### ConceptDecl

A concept is declared as:

```cpp
template <typename T>
concept Sortable = requires(T t) { t < t; };
```

Clang creates a `ConceptDecl` node, which derives from `TemplateDecl` but not from `RedeclarableTemplateDecl` (concepts cannot be partially specialized). The single data member is `Expr *ConstraintExpr`, returned by `getConstraintExpr()`. Unlike function or class templates, a concept has no "templated decl" — the template parameters define the domain and the constraint expression defines the predicate.

```cpp
class ConceptDecl : public TemplateDecl, public Mergeable<ConceptDecl> {
protected:
  Expr *ConstraintExpr;
public:
  static ConceptDecl *Create(ASTContext &C, DeclContext *DC,
                              SourceLocation L, DeclarationName Name,
                              TemplateParameterList *Params,
                              Expr *ConstraintExpr = nullptr);
  Expr *getConstraintExpr() const { return ConstraintExpr; }
};
```

### ConceptSpecializationExpr

Whenever a concept is used in a constraint position — `Sortable<T>` in a `requires`-clause — the AST contains a `ConceptSpecializationExpr`:

```cpp
class ConceptSpecializationExpr final : public Expr {
  ConceptReference    *ConceptRef;
  ASTConstraintSatisfaction *Satisfaction; // null if dependent
  // ...
public:
  ConceptDecl *getNamedConcept() const;
  bool isSatisfied() const; // only when non-dependent
  const ASTConstraintSatisfaction &getSatisfaction() const;
};
```

The `ConceptReference` struct wraps the concept's nested name specifier, template keyword location, declaration name, and template argument list as written.

### RequiresExpr

`requires (T x) { x.begin(); }` parses into a `RequiresExpr`:

```cpp
class RequiresExpr final : public Expr,
    llvm::TrailingObjects<RequiresExpr, ParmVarDecl *,
                          concepts::Requirement *> {
  RequiresExprBodyDecl *Body;
  SourceLocation RequiresKWLoc, LParenLoc, RParenLoc, RBraceLoc;
  // Trailing: ParmVarDecl* parameters, Requirement* requirements
public:
  ArrayRef<ParmVarDecl *>          getLocalParameters() const;
  ArrayRef<concepts::Requirement*> getRequirements()    const;
  bool isSatisfied() const;
};
```

Each element of the requirement body is a subclass of `concepts::Requirement`:

- **`SimpleRequirement`** — `expression;` — the expression must be valid.
- **`TypeRequirement`** — `typename T::value_type;` — the type must be well-formed.
- **`CompoundRequirement`** — `{ expr } noexcept? -> ReturnTypeConstraint;` — the expression must be valid, optionally non-throwing, and its type must satisfy the return-type constraint.
- **`NestedRequirement`** — `requires constraint-expression;` — a nested constraint that must be satisfied.

`CompoundRequirement` carries a `ReturnTypeRequirement` which is either a `TemplateParameterList` (for `-> Concept<T>` syntax) or empty.

---

## Constraint Satisfaction Checking

### CheckConstraintSatisfaction

The central entry point for evaluating whether a constraint is satisfied is:

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

`AssociatedConstraint` pairs a constraint expression with the `NamedDecl` that owns it (needed for the parameter mapping). The result is stored in a `ConstraintSatisfaction` object which records the boolean verdict and, on failure, a list of unsatisfied atomic constraints with their witness sub-expressions.

Clang caches satisfaction results in `Sema::ConstraintSatisfactionCache` (a `ContextualFoldingSet<ConstraintSatisfaction, const ASTContext&>`) keyed on the constraint expression and the template arguments. This amortizes repeated checks in overload resolution where many candidates share constraints.

### EnsureTemplateArgumentListConstraints

At every point of instantiation where template arguments are supplied, Clang calls:

```cpp
bool Sema::EnsureTemplateArgumentListConstraints(
    TemplateDecl *Template,
    const MultiLevelTemplateArgumentList &TemplateArgs,
    SourceRange TemplateIDRange);
```

This calls `getAssociatedConstraints()` on the template, then `CheckConstraintSatisfaction()`, and if the result is false, calls `DiagnoseUnsatisfiedConstraint()` to emit a diagnostic with the failed atomic constraint highlighted.

---

## Constraint Normalization and Subsumption

### NormalizedConstraint

C++ [temp.constr.normal] defines normalization of constraint expressions into a flat conjunction/disjunction of atomic constraints. Clang implements this in `SemaConcept.cpp` through the `NormalizedConstraint` hierarchy defined in [`clang/Sema/SemaConcept.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/SemaConcept.h):

```cpp
struct NormalizedConstraint {
  enum class ConstraintKind : unsigned char {
    Atomic = 0, ConceptId, FoldExpanded, Compound
  };
  enum CompoundConstraintKind : unsigned char { CCK_Conjunction, CCK_Disjunction };
  // ...
};
```

A `NormalizedConstraint` with `ConstraintKind::Atomic` is a leaf node containing the original expression and its parameter mapping. `ConstraintKind::Compound` contains left and right sub-constraints linked by `CCK_Conjunction` (`&&`) or `CCK_Disjunction` (`||`). `ConstraintKind::ConceptId` expands a `ConceptSpecializationExpr` into the concept body's normal form.

Each atomic constraint carries an `OccurenceList` (a `SmallBitVector`) recording which template parameters appear in the parameter mapping. Two atomic constraints are *identical* if they arise from the same syntactic expression in the same template and their parameter mappings are identical after canonicalization. The identity check uses `hasMatchingParameterMapping()`.

### IsAtLeastAsConstrained / Subsumption

Constraint subsumption determines whether one declaration's constraints are at least as restrictive as another's, enabling partial ordering of constrained templates:

```cpp
bool Sema::IsAtLeastAsConstrained(
    const NamedDecl *D1,
    MutableArrayRef<AssociatedConstraint> AC1,
    const NamedDecl *D2,
    MutableArrayRef<AssociatedConstraint> AC2,
    bool &Result);
```

The algorithm, per [temp.constr.order], normalizes both constraint sets and checks whether every atomic constraint in D2's disjunctive normal form is subsumed by an identical atomic constraint in D1's conjunctive normal form. This is the mechanism behind "`T` satisfying `Sortable && Printable` is more constrained than `T` satisfying `Sortable`". When both `IsAtLeastAsConstrained(D1, D2)` and `IsAtLeastAsConstrained(D2, D1)` hold, the declarations are ambiguously constrained.

---

## requires-Clauses and Associated Constraints

### Associated Constraints

`TemplateDecl::getAssociatedConstraints()` collects constraints from three sources into an `AssociatedConstraint` array:

1. Constrained template parameters: each `TemplateTypeParmDecl` with a `TypeConstraint` contributes an atomic constraint.
2. The trailing `requires`-clause on the parameter list, obtained via `TemplateParameterList::getRequiresClause()`.
3. For function templates, the trailing `requires`-clause on the function declarator itself.

These are concatenated and treated as a conjunction. The three-source structure matters for subsumption: Clang must track which `NamedDecl` owns each component so that parameter mappings can be computed correctly.

### Abbreviated Function Templates

C++20 allows `auto` parameters in function declarations as shorthand for function templates:

```cpp
auto max(auto a, auto b) -> decltype(a < b ? a : b);
```

Clang synthesizes a `TemplateTypeParmDecl` for each `auto` parameter during `Sema::adjustCCAndNoReturn()` and `Sema::BuildAbbreviatedFunctionDecl()`. The resulting function is treated as a function template with an implied `TemplateParameterList`. Constrained auto (`Sortable auto x`) introduces both the synthetic type parameter and an atomic constraint from the concept name.

### Interaction with Implicit Member Synthesis

`Sema::AddImplicitlyDeclaredMembersToClass()` synthesizes copy constructors, move constructors, copy-assignment operators, and destructors. When a class has a constrained template member, the implicitly declared special members must not subsume constraints that do not apply to them. Clang handles this by synthesizing unconstrained special members and deferring constraint checking to overload resolution.

---

## Template Instantiation Internals — TreeTransform

### TreeTransform<Derived>

All AST transformations in Clang, including template instantiation, are implemented through the `TreeTransform<Derived>` CRTP template in the private implementation file [`clang/lib/Sema/TreeTransform.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/TreeTransform.h) (not installed at `/usr/lib/llvm-22/include`; it lives only in the source tree and is compiled as part of the `libclangSema` library). `TreeTransform` exposes a virtual-dispatch-like interface via static CRTP dispatch:

```cpp
template <typename Derived>
class TreeTransform {
protected:
  Sema &SemaRef;
  Derived &getDerived() { return static_cast<Derived&>(*this); }
public:
  QualType TransformType(QualType T);
  QualType TransformType(TypeLocBuilder &TLB, TypeLoc TL);
  ExprResult TransformExpr(Expr *E);
  StmtResult TransformStmt(Stmt *S);
  Decl *TransformDecl(SourceLocation Loc, Decl *D);

  // Leaf rebuild methods — override these in derived classes:
  QualType RebuildPointerType(QualType PointeeType, SourceLocation Sigil);
  QualType RebuildReferenceType(QualType ReferentType, bool LValueRef,
                                 SourceLocation Sigil);
  ExprResult RebuildCallExpr(Expr *Callee, SourceLocation LParenLoc,
                              MultiExprArg Args, SourceLocation RParenLoc,
                              Expr *ExecConfig = nullptr);
  // ... one RebuildXxx per AST node kind
};
```

The pattern is: `TransformFoo()` is the dispatch method that casts the node and calls `getDerived().RebuildFoo(transformed-children)`. The derived class overrides only the specific `RebuildXxx()` or `TransformXxx()` methods it needs; all other calls fall through to the base class implementations. This means a new pass inheriting from `TreeTransform` gets correct behavior for all unchanged node kinds for free.

`TemplateInstantiator` (private to `SemaTemplateInstantiate.cpp`) inherits from `TreeTransform<TemplateInstantiator>` and overrides:

- `TransformTemplateTypeParmType()` — looks up the (depth, index) in the `MultiLevelTemplateArgumentList`; returns the substituted `QualType` or a `SubstTemplateTypeParmType` wrapping it.
- `TransformSubstTemplateTypeParmType()` — handles previously substituted types from nested instantiations by re-substituting if the depth is still in the substitution range.
- `TransformDeclRefExpr()` — remaps `DeclRefExpr` nodes referencing template-local declarations to their substituted counterparts via `LocalInstantiationScope::findInstantiationOf()`.
- `TransformCXXFoldExpr()` — expands the pack argument to determine the element count, then rebuilds the binary-operator tree.
- `TransformLambdaExpr()` — re-enters lambda processing to rebuild the capture list and body with substituted types.

`TemplateDeclInstantiator` (also private, inherits from `DeclVisitor<TemplateDeclInstantiator, Decl*>`) handles instantiation of declaration nodes in the member list of a class template: `VisitFunctionDecl()`, `VisitCXXMethodDecl()`, `VisitFieldDecl()`, `VisitVarDecl()`, `VisitEnumDecl()`, and so on. It coordinates with `TemplateInstantiator` for the types and initializers of each member.

### Type Specialization Representations

After instantiation the type system reflects the substitution. Key type nodes:

- `SubstTemplateTypeParmType` — a type that resulted from substituting a `TemplateTypeParmType`. Carries both the original parameter and the replacement, enabling diagnostics like "deduced as `int` for parameter `T`".
- `TemplateSpecializationType` — `vector<int>` before a corresponding `ClassTemplateSpecializationDecl` exists, or in contexts where the specialization is still dependent. Carries a `TemplateName` and an array of `TemplateArgument`.
- `InjectedClassNameType` — the self-referential type `C<T>` that is visible inside the template body via the injected class name `C`. Needed so that out-of-class member definitions correctly refer to the template class.
- `DependentNameType` — `typename T::value_type` when `T` is a type parameter.
- `DependentTemplateSpecializationType` — `typename T::template Nested<Args>` when both the qualifier and the template arguments are dependent.

### LocalInstantiationScope

`LocalInstantiationScope` maintains the mapping from template-parameter `Decl*` nodes to their substituted forms:

```cpp
class LocalInstantiationScope {
  Sema &SemaRef;
  using LocalDeclsMap =
      llvm::SmallDenseMap<const Decl *,
                          llvm::PointerUnion<Decl *, DeclArgumentPack *>, 4>;
  LocalDeclsMap CurrentInstantiationScope;
  LocalInstantiationScope *Outer;
  bool MergeWithParentScope;
  // ...
public:
  void InstantiatedLocal(const Decl *D, Decl *Inst);
  void InstantiatedLocalPackArg(const Decl *D, VarDecl *Inst);
  llvm::PointerUnion<Decl *, DeclArgumentPack *>
      findInstantiationOf(const Decl *D);
};
```

When `TransformDeclRefExpr()` encounters a reference to a `ParmVarDecl` from the template pattern, it calls `LocalInstantiationScope::findInstantiationOf()` to obtain the instantiated `ParmVarDecl` for the current specialization. For pack parameters, `findInstantiationOf()` returns a `DeclArgumentPack*` containing all expanded elements.

---

## Lazy Instantiation and Deferred Diagnostics

### The PendingInstantiations Queue

Function template bodies are parsed but not semantically analyzed at the point of declaration. When a template specialization is needed (a call expression with known argument types, `sizeof` applied to a class template, or an `odr-use` of a variable template), Clang adds the specialization to `Sema::PendingInstantiations`:

```cpp
std::deque<PendingImplicitInstantiation> PendingInstantiations;
```

Each `PendingImplicitInstantiation` is a `(ValueDecl *, SourceLocation)` pair: the `FunctionDecl` or `VarDecl` to be instantiated and the source location that triggered the demand. These are drained at end-of-translation-unit by:

```cpp
void Sema::PerformPendingInstantiations(bool LocalOnly = false,
                                         bool Complain = true);
```

The `LocalOnly` flag, set true when leaving a function body (in `Sema::ActOnEndOfFunctionDef()`), handles templates used only within that function which must be instantiated before the function definition is complete — particularly relevant for `constexpr` functions and `static_assert` conditions that depend on template specializations. Setting `Complain = false` suppresses hard errors during speculative instantiation (used internally for tentative deduction).

`Sema::SavedPendingInstantiations` is a stack of deque snapshots that isolates instantiations triggered inside `constexpr` evaluation from the outer instantiation set:

```cpp
SmallVector<std::deque<PendingImplicitInstantiation>, 1>
    SavedPendingInstantiations;
```

The RAII type `Sema::GlobalEagerInstantiationScope` saves and restores the pending queue when entering a global scope that may trigger eager instantiations (such as the definition of a `constexpr` variable with a complex initializer). Conversely, `Sema::LocalEagerInstantiationScope` manages the local queue for function-scope instantiations and calls `PerformPendingInstantiations(/*LocalOnly=*/true)` on destruction.

### Deferred Access Checking

Access control (public/private/protected) is checked at the point of use, not at parse time for templates. Inside a template, names that are accessed through dependent types have their access validity unknown until instantiation. Clang's `DelayedDiagnostic` infrastructure stores access-control failures during template parsing and replays them at instantiation time. Each delayed diagnostic carries the access specifier, the source location, the context in which access was attempted, and the target declaration. `Sema::PerformDependentDiagnostics()` processes these after a class template is instantiated, re-checking each stored access check with the concrete template arguments substituted. Only failures that were truly access violations — not those suppressed by friendship or using-declarations — are emitted at this point.

### Lambda Captures in Templates

When a lambda appears inside a function template body, its capture list may reference dependent variables. The `TemplateInstantiator::TransformLambdaExpr()` override re-enters `Sema::ActOnLambdaExpr()` during instantiation, rebuilding the lambda with concrete types. Captures are re-added to the `LambdaScopeInfo` in `SemaLambda.cpp`. Generic lambdas (`[](auto x) { ... }`) generate implicit function templates inside the lambda's operator() and are re-instantiated each time the lambda is called with a new argument type.

---

## Friend Templates and Dependent Names

### FriendDecl with Templates

A friend declaration inside a class template may grant friendship to a function template or to a class template:

```cpp
template <typename T>
class Matrix {
  friend Matrix operator*(const Matrix&, const Matrix&);
  template <typename U>
  friend class Vector;
};
```

Clang represents these as `FriendDecl` nodes (in `DeclFriend.h`) which store either a `TypeSourceInfo` (for friend type declarations) or a `NamedDecl*` (for friend function or template declarations). `FriendDecl::getFriendDecl()` returns the `FunctionDecl`, `FunctionTemplateDecl`, or null (for type friends).

During class template instantiation, friend function declarations are instantiated and added to the enclosing namespace scope via `Sema::InstantiateFriendDeclaration()`.

### Dependent Name Representation

Inside a template body, names whose lookup depends on template parameters are "dependent". Clang represents them as special AST nodes:

- **`DependentScopeDeclRefExpr`** — `T::value` where `T` is a type parameter; the declaration cannot be resolved until instantiation.
- **`CXXDependentScopeMemberExpr`** — `obj.method()` where `obj` has a dependent type.

Both carry a `NestedNameSpecifier` for the qualifier and an optional `TemplateArgumentListInfo` for explicit template arguments.

### The `typename` Disambiguator

`T::value_type` in a dependent context is syntactically ambiguous: it could be a type or a value. The `typename` keyword asserts the type interpretation. Sema handles this in:

```cpp
TypeResult Sema::ActOnTypenameType(
    Scope *S, SourceLocation TypenameLoc,
    const CXXScopeSpec &SS, const IdentifierInfo &II,
    SourceLocation IdLoc,
    ImplicitTypenameContext IsImplicitTypename);
```

This produces an `ElaboratedType` wrapping a `DependentNameType` when the scope is dependent. At instantiation time, `TreeTransform::TransformDependentNameType()` resolves the name in the substituted scope and replaces the dependent type with the actual type.

### Two-Phase Name Lookup

C++ mandates two-phase name lookup for templates:

1. **Phase 1** (definition time): non-dependent names are looked up and bound immediately using the template's enclosing scope. An unqualified call `f(x)` where `x` is non-dependent binds to whatever `f` is visible at definition time.
2. **Phase 2** (instantiation time): dependent names are resolved in the instantiation context. Argument-dependent lookup (ADL) is deferred to phase 2 for function calls with dependent arguments.

Clang implements phase-2 lookup through the `DependentScopeDeclRefExpr` and `CXXDependentScopeMemberExpr` deferral mechanism: these nodes are not resolved at parse time; `TemplateInstantiator::TransformDependentScopeDeclRefExpr()` performs the actual lookup during instantiation. The `template` keyword disambiguator (`t.template foo<int>()`) sets `hasTemplateKWAndArgsInfo()` on the dependent member expression and causes the lookup to include template names.

A practical consequence is that ill-formed dependent calls may compile successfully if the template is never instantiated. Clang's `-fdelayed-template-parsing` flag (defaulted on for MSVC compatibility) further delays phase-1 parsing; the standard-conforming behavior is the default on non-Windows targets.

The `template` keyword disambiguator deserves particular attention. In `t.foo<int>()`, if the type of `t` is dependent, `foo` could be a template or a value: `t.foo < int > ()` could parse as two comparisons. The `template` keyword — `t.template foo<int>()` — tells the parser to treat `foo` as a template name. Clang represents this via the `hasTemplateKWAndArgsInfo()` flag on `CXXDependentScopeMemberExpr` and `DependentScopeDeclRefExpr`, which causes `TemplateInstantiator::TransformCXXDependentScopeMemberExpr()` to perform template-name lookup (not value lookup) when resolving the expression.

---

## Diagnostic Notes and Developer Patterns

### Reading Template Backtraces

When Clang emits "in instantiation of…" notes, each line corresponds to one `CodeSynthesisContext` on the `CodeSynthesisContexts` stack. The kind field distinguishes function template instantiation, class template instantiation, constraint checking, default argument substitution, and so on. Tooling that processes these diagnostics via `clang::DiagnosticConsumer` can access the context stack through `Sema::getTemplateInstantiationArgs()`.

### Examining Specializations Programmatically

To iterate all explicit or implicit specializations of a class template:

```cpp
ClassTemplateDecl *CTD = /* ... */;
for (auto *Spec : CTD->specializations()) {
    // Spec is ClassTemplateSpecializationDecl*
    auto Kind = Spec->getSpecializationKind();
    auto &Args = Spec->getTemplateArgs();
}
// Partial specializations are separate:
for (auto *Part : CTD->getPartialSpecializations()) {
    // Part is ClassTemplatePartialSpecializationDecl*
}
```

For function templates:

```cpp
FunctionTemplateDecl *FTD = /* ... */;
for (auto *Spec : FTD->specializations()) {
    // Spec is FunctionDecl* with getTemplateSpecializationInfo() non-null
}
```

### Triggering Constraint Checking in a Tool

```cpp
// Force constraint satisfaction check outside of overload resolution:
ConstraintSatisfaction CS;
bool Failed = SemaRef.CheckConstraintSatisfaction(
    ConceptDecl, ConceptDecl->getAssociatedConstraints(),
    MLTAL, ConceptRefRange, CS);
if (Failed || !CS.IsSatisfied) {
    SemaRef.DiagnoseUnsatisfiedConstraint(CS);
}
```

---

## Chapter Summary

- The `TemplateDecl` hierarchy (`FunctionTemplateDecl`, `ClassTemplateDecl`, `VarTemplateDecl`, `TypeAliasTemplateDecl`, `ConceptDecl`) stores parameters in a `TemplateParameterList` and specializations in a folding set.
- `TemplateArgument` is a discriminated union over nine kinds; `TemplateArgumentList` is its heap-allocated container attached to specialization nodes.
- Template instantiation is driven by `InstantiateFunctionDefinition()` / `InstantiateClass()`, protected by the `InstantiatingTemplate` RAII guard that enforces the instantiation depth limit.
- `MultiLevelTemplateArgumentList` resolves template parameters at multiple nesting depths simultaneously, essential for member templates of class templates.
- Argument deduction for function templates flows through `DeduceTemplateArguments()` → `DeduceTemplateArgumentsByTypeMatch()` → `FinishTemplateArgumentDeduction()`; CTAD adds deduction guides synthesized from constructors.
- SFINAE is implemented by `SFINAETrap` + `TemplateDeductionInfo`; soft errors (substitution failures) are silently swallowed; hard errors escape the trap.
- `ConceptDecl` owns a single constraint expression; `ConceptSpecializationExpr` appears wherever a concept is used as a constraint; `RequiresExpr` encodes `requires { }` bodies with `SimpleRequirement`, `TypeRequirement`, `CompoundRequirement`, and `NestedRequirement` sub-nodes.
- Constraint satisfaction is evaluated by `CheckConstraintSatisfaction()` with results cached; `IsAtLeastAsConstrained()` implements subsumption for partial ordering.
- `TreeTransform<TemplateInstantiator>` is the CRTP engine for all AST transformations; `LocalInstantiationScope` maps pattern `Decl*` nodes to their instantiated counterparts.
- Function template bodies are not instantiated until first use; `PendingInstantiations` accumulates deferred work and is drained by `PerformPendingInstantiations()` at end-of-TU.
- Two-phase name lookup defers dependent name resolution to instantiation time; `DependentScopeDeclRefExpr` and `CXXDependentScopeMemberExpr` carry unresolved dependent references.

---

## Cross-References

- [Chapter 33 — Sema I: Names, Lookups, and Conversions](../part-05-clang-frontend/ch33-sema-names-lookups-conversions.md) — overload resolution, ADL, and implicit conversions that interact with template deduction.
- [Chapter 35 — The Constant Evaluator](../part-05-clang-frontend/ch35-constant-evaluator.md) — `constexpr` evaluation used by `NonTypeTemplateParmDecl` initializers, `sizeof...`, and constraint satisfaction of `noexcept` specifiers.
- [Chapter 36 — The Clang AST in Depth](../part-05-clang-frontend/ch36-clang-ast.md) — comprehensive survey of AST node types including all template-related nodes.
- [Chapter 13 — Polymorphism and Type Inference](../part-03-type-theory/ch13-polymorphism-type-inference.md) — theoretical background for parametric polymorphism and Hindley-Milner type inference, whose concepts inform C++ template design.

## Reference Links

- [`clang/include/clang/AST/DeclTemplate.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/DeclTemplate.h) — `TemplateDecl` hierarchy, `TemplateParameterList`, `ClassTemplateSpecializationDecl`, `ConceptDecl`.
- [`clang/include/clang/AST/TemplateBase.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/TemplateBase.h) — `TemplateArgument`, `TemplateArgumentList`.
- [`clang/include/clang/AST/ExprConcepts.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/ExprConcepts.h) — `ConceptSpecializationExpr`, `RequiresExpr`, `SimpleRequirement`, `TypeRequirement`, `CompoundRequirement`, `NestedRequirement`.
- [`clang/include/clang/Sema/Sema.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/Sema.h) — `InstantiatingTemplate`, `SFINAETrap`, `DeduceTemplateArguments`, `CheckConstraintSatisfaction`, `EnsureTemplateArgumentListConstraints`, `IsAtLeastAsConstrained`.
- [`clang/include/clang/Sema/Template.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/Template.h) — `MultiLevelTemplateArgumentList`, `LocalInstantiationScope`, `TemplateSubstitutionKind`.
- [`clang/include/clang/Sema/SemaConcept.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/SemaConcept.h) — `NormalizedConstraint`, `AtomicConstraint`.
- [`clang/include/clang/Sema/TemplateDeduction.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/TemplateDeduction.h) — `TemplateDeductionInfo`.
- [`clang/lib/Sema/SemaTemplate.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaTemplate.cpp) — `ActOnClassTemplate`, `ActOnTemplateDeclarator`, `ActOnExplicitInstantiation`.
- [`clang/lib/Sema/SemaTemplateInstantiate.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaTemplateInstantiate.cpp) — `InstantiateFunctionDefinition`, `InstantiateClass`, `PerformPendingInstantiations`, `TemplateInstantiator`.
- [`clang/lib/Sema/SemaTemplateDeduction.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaTemplateDeduction.cpp) — `DeduceTemplateArguments`, `DeduceTemplateArgumentsByTypeMatch`, `FinishTemplateArgumentDeduction`.
- [`clang/lib/Sema/SemaConcept.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaConcept.cpp) — `CheckConstraintSatisfaction`, `IsAtLeastAsConstrained`, constraint normalization.
- [`clang/lib/Sema/TreeTransform.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/TreeTransform.h) — `TreeTransform<Derived>` CRTP base for all AST transformations.


---

@copyright jreuben11
