# Chapter 36 — The Clang AST in Depth

*Part V — Clang Internals: Frontend Pipeline*

The Clang Abstract Syntax Tree is simultaneously the most distinctive and the most load-bearing component of the entire Clang architecture. Unlike compilers that immediately lower source code to a normalized intermediate representation, Clang preserves the tree in a form that stays close to what the programmer wrote. Every implicit conversion, every synthesized special-member function, every default argument substitution, every temporary materialization is encoded as a first-class AST node with source locations. That fidelity makes the same tree serve diagnostics, refactoring tools, static analyzers, and code generation without any intermediate re-elaboration. This chapter dissects the AST from first principles: the arena allocator and type-uniquing context, the full `Decl`/`Type`/`Stmt`/`Expr` hierarchies, the value-category system, the traversal and visitor infrastructure, the matcher DSL, and the mechanics of implicit node synthesis.

---

## 36.1 AST Philosophy: Staying Close to the Source

Most compiler IRs are normalized: they fold away syntactic sugar, collapse type aliases, canonicalize implicit conversions into explicit operations, and erase source positions early. Clang takes the opposite position. The AST it produces is intentionally redundant, verbose, and source-faithful. Several design decisions follow from this commitment.

**No separate IL before codegen.** Sema elaborates the AST in place, annotating nodes with types, overload resolution results, and implicit conversion sequences. CodeGen walks the annotated AST directly to produce LLVM IR. There is no separate "HIR" stage in between. The benefit is that Clang tools — static analyzers, refactoring engines, IDE services, code formatters — all work on the same representation. Compare this to GCC, which has GIMPLE as a mandatory lowered IR, or Swift, which uses SIL between parse and codegen. Clang's choice means each tool that wants source-level precision can obtain it without reconstruction.

**Implicit nodes are explicit.** When the programmer writes `foo(p)` and `foo` expects a `double` but `p` is `int`, the AST contains an `ImplicitCastExpr` of kind `CK_IntegralToFloating` wrapping the `DeclRefExpr` for `p`. The programmer never wrote that cast, but it is a node with a source location (the position of `p`) so diagnostics can point to exactly the right place. Similarly, a call to `bar(x)` where `bar(int x, int y = 42)` has a default second argument materializes a `CXXDefaultArgExpr` node in the call site's argument list. The source location of that node is `<invalid sloc>` (no corresponding text), but the node carries the default-argument expression pointer.

**Sugar is preserved.** `typedef Foo Bar; Bar b;` — the declared type of `b` is a `TypedefType` whose underlying type is `Foo`. Canonical type comparison strips that sugar, but the AST retains it so that diagnostics print `Bar` and not `Foo` when the programmer used `Bar`. This principle extends to elaborated type specifiers (`struct Point p`) and `decltype` expressions.

**Value categories are tracked per-node.** Every `Expr` carries an `ExprValueKind` field encoding whether it produces an lvalue, an xvalue, or a prvalue. This is not derived from the type — it is an independent annotation assigned by Sema and consumed by CodeGen.

**Each decl is represented once per declaration, not once per definition.** Clang distinguishes `isThisDeclarationADefinition()` (true only for the one `FunctionDecl` that has a body) from `isDefined()` (true if any redeclaration is a definition). Forward declarations, extern declarations, friend declarations, and the defining declaration all have their own `Decl` nodes linked via `getPreviousDecl()` / `getMostRecentDecl()` redecl chains. This models the C++ concept of declarations and definitions faithfully.

**Ownership via `DeclContext`.** Every `Decl` is owned by exactly one `DeclContext`. `DeclContext` is a parallel mixin hierarchy — `TranslationUnitDecl`, `NamespaceDecl`, `FunctionDecl`, `RecordDecl`, `LinkageSpecDecl`, `BlockDecl`, and others are both `Decl` nodes and `DeclContext` parents. A `DeclContext` supports lexical traversal (`decls()`) for all declarations textually present, and semantic lookup (`lookup()`, `localUncachedLookup()`) for name-based queries.

---

## 36.2 `ASTContext`: The Allocator and Type-Uniquing Table

[`clang/include/clang/AST/ASTContext.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/ASTContext.h) defines the `ASTContext` class, which owns all AST memory and serves as the canonical registry for types.

### 36.2.1 Arena Allocation

Every AST node — every `Decl`, every `Stmt`, every `Type` — is allocated from a single bump-pointer arena owned by `ASTContext`:

```cpp
class ASTContext : public RefCountedBase<ASTContext> {
  mutable llvm::BumpPtrAllocator BumpAlloc;
public:
  void *Allocate(size_t Size, unsigned Align = 8) const {
    return BumpAlloc.Allocate(Size, Align);
  }
  template <typename T> T *Allocate(size_t Num = 1) const {
    return static_cast<T *>(Allocate(Num * sizeof(T), alignof(T)));
  }
  size_t getASTAllocatedMemory() const { return BumpAlloc.getTotalMemory(); }
};
```

The placement-`new` overloads on `Decl` and `Stmt` route through `ASTContext::Allocate()`, so every `new (Context) FunctionDecl(...)` call goes to the bump allocator. No AST node is individually freed; the entire arena is released when `ASTContext` is destroyed. This gives allocation latency of a few nanoseconds per node — important for large translation units with millions of nodes.

### 36.2.2 Type Uniquing

Types in Clang are immutable and canonically deduplicated. `ASTContext` maintains a folding-set of every type ever constructed:

```cpp
QualType ASTContext::getPointerType(QualType T) const;
QualType ASTContext::getFunctionType(QualType ResultTy,
                                     ArrayRef<QualType> Args,
                                     const FunctionProtoType::ExtProtoInfo &EPI);
QualType ASTContext::getIntegerTypeOrder(QualType LHS, QualType RHS) const;
```

Two calls to `getPointerType(IntTy)` with the same underlying `QualType` return the same `PointerType*`. Type pointer equality therefore implies structural equality — which makes type checking and overload resolution cheap.

`QualType` is a lightweight value type: a tagged pointer encoding `const`/`volatile`/`restrict` qualifiers in the three low bits alongside a `Type*`. The `Type*` always points into arena memory.

### 36.2.3 Canonical Types

Every type has a *canonical form* — the type with all sugar stripped. `QualType::getCanonicalType()` and `ASTContext::getCanonicalType(QualType)` return the canonical version. `ASTContext` stores both the sugared and the canonical pointer for each created type so that:

```cpp
QualType T1 = Context.getTypedefType(FooDecl);   // 'Foo' — sugared
QualType T2 = Context.getIntType();               // 'int' — canonical
bool same = Context.hasSameType(T1, T2);          // true iff Foo = int
```

`ASTContext::hasSameType()` compares canonical pointers. `ASTContext::hasSameUnqualifiedType()` additionally strips top-level qualifiers before comparing.

### 36.2.4 Size, Alignment, and Layout

The target-dependent size and alignment of any type are obtained through:

```cpp
uint64_t ASTContext::getTypeSize(QualType T) const;   // bits
uint64_t ASTContext::getTypeAlign(QualType T) const;  // bits
CharUnits ASTContext::getTypeSizeInChars(QualType T) const;
CharUnits ASTContext::getTypeAlignInChars(QualType T) const;
```

These delegate to `ASTContext::getTypeInfo(QualType)`, which dispatches on `TypeClass` and consults the `TargetInfo` for fundamental types. Record layout queries go through `ASTContext::getASTRecordLayout(const RecordDecl*)`, which caches the result in a side table.

### 36.2.5 Root of the Decl Tree

```cpp
TranslationUnitDecl *ASTContext::getTranslationUnitDecl() const;
```

This returns the single `TranslationUnitDecl` that is the parent of all top-level declarations. The Parser creates it once and Sema populates it incrementally as each declaration is processed. It is a `DeclContext` as well as a `Decl`, so it supports `decls()` iteration, lookup tables, and `localUncachedLookup()`.

---

## 36.3 The `Decl` Hierarchy

All declarations in a Clang AST inherit from [`clang/include/clang/AST/DeclBase.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/DeclBase.h)'s `Decl` base class.

### 36.3.1 `Decl` Base

`Decl` stores:
- `DeclKind Kind` — an enum value from `DeclNodes.inc`, identifies the concrete class.
- `SourceLocation Loc` — the name position (most-specific location, e.g., identifier token).
- A pointer to the `DeclContext` owning this declaration (packed with two semantic bits).
- A `NextDeclInContext` pointer forming a linked list of declarations in the same context.
- Module ownership, access specifier, and various flags.

The `Decl::dump()` and `Decl::print()` methods provide debug and pretty output:

```cpp
FunctionDecl *FD = /* ... */;
FD->dump();          // textual AST dump to llvm::errs()
FD->print(llvm::outs());  // source-like pretty-print
```

### 36.3.2 `NamedDecl` and `ValueDecl`

`NamedDecl` adds a `DeclarationName` (which can be a simple identifier, an operator name, a constructor name, a conversion function name, etc.). `DeclarationName` is a discriminated union encoding the name kind and the associated data (an `IdentifierInfo*` for ordinary names, a `QualType` for constructor/destructor/conversion names, an operator enum for operator overloads). Key queries:

```cpp
NamedDecl *ND = /* ... */;
DeclarationName N = ND->getDeclName();
bool isId = N.isIdentifier();
IdentifierInfo *II = N.getAsIdentifierInfo(); // null if not an identifier
std::string S = ND->getNameAsString();        // always works
StringRef SR = ND->getName();                 // works for identifiers only
bool hasLinkage = ND->hasLinkage();
bool isHidden = ND->isHidden();               // module-private
```

`ValueDecl` further adds a `QualType` for the declared type of the value, accessible via `getType()`. `DeclaratorDecl` wraps a `TypeSourceInfo*`, giving access to the source-level type including source locations for each type component. `TypeSourceInfo` carries both the `QualType` and a `TypeLoc` tree that attaches `SourceLocation` information to every component of a complex type like `const int * const * volatile` — enabling refactoring tools to find and modify individual type qualifiers.

### 36.3.3 `VarDecl` and `FieldDecl`

`VarDecl` (defined in [`clang/include/clang/AST/Decl.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Decl.h)) covers local variables, parameters (via `ParmVarDecl`), and file-scope variables. Key accessors:

```cpp
bool VarDecl::isLocalVarDecl() const;
bool VarDecl::hasInit() const;
Expr *VarDecl::getInit();
StorageDuration VarDecl::getStorageDuration() const; // SD_Automatic, SD_Static, etc.
bool VarDecl::isConstexpr() const;
```

`FieldDecl` represents a non-static data member. It carries a bit-width expression when the field is a bit-field and an in-class initializer expression accessible via `FieldDecl::getInClassInitializer()`.

### 36.3.4 `FunctionDecl`

`FunctionDecl` is the most structurally complex declaration class. It stores:
- A `DeclarationNameInfo` with source location.
- A list of `ParmVarDecl*` parameters via `parameters()`.
- A `CompoundStmt*` body when the function is a definition, accessible via `getBody()`.
- Template information (`FunctionTemplateDecl*` if it is a template, or a `FunctionTemplateSpecializationInfo*` if it is a specialization).
- Linkage, inline, constexpr, and exception-specification flags.

```cpp
bool FunctionDecl::isThisDeclarationADefinition() const;
bool FunctionDecl::isDefined() const;    // true if any redecl is a def
unsigned FunctionDecl::getNumParams() const;
ParmVarDecl *FunctionDecl::getParamDecl(unsigned i) const;
QualType FunctionDecl::getReturnType() const;
bool FunctionDecl::isVariadic() const;
bool FunctionDecl::isInlineSpecified() const;
bool FunctionDecl::isConstexpr() const;
bool FunctionDecl::hasBody() const;
```

`ParmVarDecl` inherits `VarDecl` and adds `hasDefaultArg()`, `getDefaultArg()` (the expression), `hasUninstantiatedDefaultArg()` (for template parameters), and `getOriginalType()` (the type before array-to-pointer decay for function parameter types). A `ParmVarDecl` without a name is valid (`int foo(int, int)`) and `getName()` returns an empty string.

Redeclarations are linked via the `Redeclarable<FunctionDecl>` mixin. Given any `FunctionDecl *FD`:

```cpp
FunctionDecl *First = FD->getFirstDecl();     // earliest redecl
FunctionDecl *Prev  = FD->getPreviousDecl();  // nullptr for first
FunctionDecl *Recent = FD->getMostRecentDecl();
for (FunctionDecl *RD : FD->redecls())
    process(RD);
```

### 36.3.5 CXX Method Hierarchy

`CXXMethodDecl` inherits `FunctionDecl` and adds `isConst()`, `isVirtual()`, `isStatic()`, `getParent()` (returns the `CXXRecordDecl`), and `getThisType()` (the implicit `this` parameter type). The `overridden_methods()` range iterates base-class virtual functions that this method overrides, enabling call-graph analysis without consulting the vtable.

Constructors are represented by `CXXConstructorDecl`, which records whether it is a copy/move constructor via `isCopyConstructor()`/`isMoveConstructor()`, and carries a list of `CXXCtorInitializer*` member-initializers. Each `CXXCtorInitializer` records either a base-class initializer or a member initializer, with the initializer expression and source location of the `:` list. `CXXDestructorDecl` adds `getOperatorDelete()` for the deallocation function linked to an `operator delete` call in the destructor body. `CXXConversionDecl` extends the hierarchy for user-defined conversion operators.

### 36.3.6 `TagDecl`, `RecordDecl`, `CXXRecordDecl`, `EnumDecl`

`TagDecl` is the common base for `struct`/`class`/`union`/`enum`. It carries a `TagKind` (TTK_Struct, TTK_Class, TTK_Union, TTK_Enum) and the notion of definition vs forward declaration. `CXXRecordDecl` ([`clang/include/clang/AST/DeclCXX.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/DeclCXX.h)) is the C++ version, adding:
- Base-class specifications (`bases()`, `vbases()`).
- Iteration over constructors, methods, and friends.
- The `DefinitionData` struct tracking triviality, standard-layout, and implicit-member state.

```cpp
bool CXXRecordDecl::needsImplicitDefaultConstructor() const;
bool CXXRecordDecl::hasTrivialDefaultConstructor() const;
bool CXXRecordDecl::isPolymorphic() const;
bool CXXRecordDecl::isStandardLayout() const;
```

### 36.3.7 Other Notable Decl Classes

- `TypedefNameDecl` / `TypeAliasDecl` — typedef and `using` aliases. `TypedefNameDecl::getUnderlyingType()` gives the sugared target type; the canonical type can differ substantially.
- `TemplateDecl` — base for `FunctionTemplateDecl`, `ClassTemplateDecl`, `VarTemplateDecl`, `TypeAliasTemplateDecl`. Each wraps the templated decl via `getTemplatedDecl()`.
- `ClassTemplateSpecializationDecl` — a `CXXRecordDecl` that is also a template specialization. It carries `getTemplateArgs()` returning the argument list and `getSpecializationKind()` distinguishing explicit vs implicit instantiation.
- `UsingShadowDecl` — introduces a using-declaration's set of shadow declarations into a scope. Each shadow points to the underlying target declaration via `getTargetDecl()`.
- `BindingDecl` — each element of a structured binding (`auto [a, b] = p;`) is a `BindingDecl` with a `HoldingVar` and a `getBinding()` expression that computes the element.
- `BlockDecl` — Clang extension for Objective-C/C blocks (`^{ ... }`). It carries a `captures()` range of `BlockDecl::Capture` objects recording which variables are captured and how.
- `ConceptDecl` — a C++20 concept. It wraps the constraint expression accessible via `getConstraintExpr()`; `Sema` calls `CheckConstraintSatisfaction` to evaluate it during template argument deduction.
- `NamespaceDecl` — both a `NamedDecl` and a `DeclContext`. `isAnonymousNamespace()`, `isInlineNamespace()`, and `getOriginalNamespace()` provide the key queries. Multiple definitions of the same namespace accumulate via `getNextNamespace()` singly-linked list.

---

## 36.4 The `Type` Hierarchy and `QualType`

### 36.4.1 `QualType`

`QualType` is defined in [`clang/include/clang/AST/Type.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Type.h). It is a value type — a 64-bit word storing a `Type*` with qualifiers in the lowest three bits:

| Bit | Qualifier |
|-----|-----------|
| 0   | `const`   |
| 1   | `volatile`|
| 2   | `restrict`|

Key operations:

```cpp
QualType QT = /* ... */;
const Type *TP = QT.getTypePtr();
bool isC = QT.isConstQualified();
bool isV = QT.isVolatileQualified();
QualType CQ = QT.withConst();   // returns new QualType with const set
QualType UQ = QT.getUnqualifiedType(); // strip all cvr
Qualifiers Q = QT.getQualifiers();
bool isNull = QT.isNull();      // a null QualType (empty)
```

### 36.4.2 The `Type` Base Class and `TypeClass` Enum

`Type` is the polymorphic base. The concrete class is identified by a `TypeClass` enum value stored in the `Type::TypeBits` bitfield. The enum is generated from `clang/include/clang/AST/TypeNodes.inc` via the `TYPE(Class, Base)` macro. Key branches:

| `TypeClass` | Concrete class | Source form |
|-------------|----------------|-------------|
| `Builtin` | `BuiltinType` | `int`, `float`, `void`, `bool` |
| `Pointer` | `PointerType` | `T*` |
| `LValueReference` | `LValueReferenceType` | `T&` |
| `RValueReference` | `RValueReferenceType` | `T&&` |
| `ConstantArray` | `ConstantArrayType` | `T[N]` |
| `VariableArray` | `VariableArrayType` | `T[expr]` (C99 VLA) |
| `IncompleteArray` | `IncompleteArrayType` | `T[]` |
| `FunctionProto` | `FunctionProtoType` | `R(A1,...,AN)` with proto |
| `FunctionNoProto` | `FunctionNoProtoType` | `R()` K&R-style |
| `Record` | `RecordType` | struct/class/union |
| `Enum` | `EnumType` | enum |
| `TemplateSpecialization` | `TemplateSpecializationType` | `vector<int>` |
| `Auto` | `AutoType` | `auto`, `decltype(auto)` |
| `Decltype` | `DecltypeType` | `decltype(expr)` |
| `Elaborated` | `ElaboratedType` | `struct Foo`, `ns::Bar` |
| `PackExpansion` | `PackExpansionType` | `T...` |

### 36.4.3 Common Type Query Methods

```cpp
bool Type::isIntegerType() const;
bool Type::isSignedIntegerType() const;
bool Type::isFloatingType() const;
bool Type::isPointerType() const;
bool Type::isReferenceType() const;
bool Type::isArrayType() const;
bool Type::isFunctionType() const;
bool Type::isRecordType() const;
bool Type::isDependentType() const;
bool Type::isInstantiationDependentType() const;
bool Type::isVariablyModifiedType() const; // VLA or member of VLA
bool Type::isObjectType() const;
bool Type::isScalarType() const;
bool Type::isLiteralType(const ASTContext &Ctx) const;
CXXRecordDecl *Type::getAsCXXRecordDecl() const; // null for non-record types
const RecordType *Type::getAsStructureType() const;
```

Type queries operate on the *de-sugared* type — `isPointerType()` strips `TypedefType` and `ElaboratedType` wrappers automatically because the implementation calls `getUnqualifiedDesugaredType()`. The important distinction: `QualType::getTypePtr()` gives the *sugared* `Type*`, while `QualType::getCanonicalType().getTypePtr()` gives the fully desugared canonical `Type*`. Static analysis tools that want to discover the "real" type behind aliases should use canonical types; diagnostic tools that want to display what the programmer wrote should use the sugared type.

For pointer and array types, `Type::getPointeeType()` and `ArrayType::getElementType()` return `QualType` values that preserve qualifiers on the element type — critical for correctness when working with `const int *` vs `int *`.

### 36.4.4 `FunctionProtoType`

The most information-dense type node. It stores parameter types, variadic flag, exception specification (`noexcept`, `throw()`), ref-qualifier (`&`/`&&`), cv-qualifiers for the function type, calling convention, and the `ExtProtoInfo` aggregate:

```cpp
const FunctionProtoType *FPT = /* ... */;
unsigned N = FPT->getNumParams();
QualType Ret = FPT->getReturnType();
ArrayRef<QualType> Params = FPT->getParamTypes();
ExceptionSpecificationType EST = FPT->getExceptionSpecType();
bool isNoexcept = FPT->isNothrow();
RefQualifierKind RQ = FPT->getRefQualifier(); // RQ_None, RQ_LValue, RQ_RValue
```

---

## 36.5 The `Stmt` and `Expr` Hierarchy

### 36.5.1 `Stmt` Base

`Stmt` is defined in [`clang/include/clang/AST/Stmt.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Stmt.h). It is the base for all statements and all expressions (since `Expr` derives from `Stmt`). The `StmtClass` enum, generated from `StmtNodes.inc`, identifies concrete classes. Allocation uses the same `ASTContext` arena.

Every `Stmt` supports child iteration:

```cpp
for (Stmt *child : S->children())
    process(child);
```

`Stmt::getBeginLoc()` / `Stmt::getEndLoc()` return `SourceLocation` values for the source extent.

### 36.5.2 Statement Classes

The primary statement nodes mirror the C/C++ grammar:

- `CompoundStmt` — brace-delimited sequence, holds `body_begin()`/`body_end()` iterators.
- `IfStmt` — condition, then-branch, optional else-branch, optional init-statement, optional condition-variable.
- `ForStmt` — init, condition, increment, body; all optional.
- `CXXForRangeStmt` — range-based for; carries the range variable, begin/end iterators, and loop variable.
- `WhileStmt`, `DoStmt` — while and do-while.
- `SwitchStmt`, `CaseStmt`, `DefaultStmt` — switch and its branches.
- `ReturnStmt` — optional return expression.
- `DeclStmt` — wraps a `DeclGroupRef` of one or more declarations that appear as a statement.
- `GotoStmt`, `LabelStmt` — goto target and label.
- `BreakStmt`, `ContinueStmt`, `NullStmt` — simple control-flow.
- `CXXTryStmt`, `CXXCatchStmt` — try/catch blocks.

### 36.5.3 `Expr` Base

`Expr` inherits `Stmt` and adds:
- `QualType` — the type of the expression, accessible via `getType()`.
- `ExprValueKind` — encoded in `ExprBits.ValueKind`; see section 36.6.
- `ExprObjectKind` — encoded in `ExprBits.ObjectKind`.

### 36.5.4 Literal Expressions

```
IntegerLiteral      — integer constant; carries APInt value
FloatingLiteral     — floating-point constant; carries APFloat
StringLiteral       — string or wide string; carries string data and byte width
CharacterLiteral    — 'x'; carries code point as unsigned
CXXBoolLiteralExpr  — true/false
CXXNullPtrLiteralExpr — nullptr
```

### 36.5.5 Reference and Member Expressions

`DeclRefExpr` refers to a named declaration (`VarDecl`, `FunctionDecl`, `EnumConstantDecl`, etc.). It carries a `ValueDecl*` for the referenced declaration and a `NamedDecl*` for the found declaration (which may differ for using-declarations).

```cpp
DeclRefExpr *DRE = /* ... */;
ValueDecl *VD = DRE->getDecl();
SourceLocation NameLoc = DRE->getLocation();
bool HasQualifier = DRE->hasQualifier();  // e.g., ns::x
```

`MemberExpr` represents `obj.field` or `obj->field`. It carries the base expression, the `ValueDecl*` member, and flags for arrow-vs-dot, explicit template arguments, and whether the member is a pointer-to-member dereference.

### 36.5.6 Call Expressions

`CallExpr` stores the callee expression, argument list, and result type. Subclasses refine the dispatch kind:
- `CXXMemberCallExpr` — member function call; `getImplicitObjectArgument()` returns the receiver.
- `CXXOperatorCallExpr` — overloaded operator call; `getOperator()` returns the `OverloadedOperatorKind`.
- `CUDAKernelCallExpr` — CUDA kernel launch with `<<<config>>>`.

### 36.5.7 Arithmetic and Logical Expressions

`UnaryOperator` stores an opcode (`UO_Minus`, `UO_Not`, `UO_Deref`, `UO_AddrOf`, `UO_PreInc`, `UO_PostInc`, `UO_PostDec`, `UO_PreDec`, `UO_LNot`, `UO_Real`, `UO_Imag`, etc.) and a subexpression. The opcode is checked via `UnaryOperator::getOpcode()` returning a `UnaryOperatorKind` enum. `UnaryOperator::isPrefix(Kind)` and `isPostfix(Kind)` distinguish pre- from post-increment.

`BinaryOperator` stores an opcode (`BO_Add`, `BO_Sub`, `BO_Mul`, `BO_Div`, `BO_Rem`, `BO_Shl`, `BO_Shr`, `BO_LT`, `BO_GT`, `BO_LE`, `BO_GE`, `BO_EQ`, `BO_NE`, `BO_And`, `BO_Or`, `BO_Xor`, `BO_LAnd`, `BO_LOr`, `BO_Assign`, `BO_Comma`, etc.) and left/right operands. Static helpers:

```cpp
bool BinaryOperator::isAssignmentOp(BinaryOperatorKind Op);
bool BinaryOperator::isComparisonOp(BinaryOperatorKind Op);
bool BinaryOperator::isRelationalOp(BinaryOperatorKind Op);
bool BinaryOperator::isShiftOp(BinaryOperatorKind Op);
```

`CompoundAssignOperator` extends `BinaryOperator` for `+=`/`-=`/`*=`/etc., storing the computed left-hand-side type and the computation result type separately. For pointer arithmetic (`p += n`), the computation LHS type is `ptrdiff_t` even though the declared type of `p` is `T*`. CodeGen uses `getComputationLHSType()` and `getComputationResultType()` to generate correct IR.

`ConditionalOperator` holds condition, LHS (true branch), and RHS (false branch). `BinaryConditionalOperator` is the GNU extension `x ?: y` — it evaluates `x` once, using the value as both the test and the true branch.

### 36.5.8 Cast Expressions

`ImplicitCastExpr` represents a compiler-inserted conversion. It is a separate node in the AST rather than being silent, which allows tools to observe every conversion path. `ExplicitCastExpr` is the base for user-written casts, with subclasses `CStyleCastExpr`, `CXXStaticCastExpr`, `CXXDynamicCastExpr`, `CXXReinterpretCastExpr`, `CXXConstCastExpr`. All cast classes store a `CastKind` enum and a path for base-to-derived conversions.

### 36.5.9 C++ Expression Nodes

```
CXXConstructExpr       — direct construction, T(a, b)
CXXTemporaryObjectExpr — T{...} or T(a) as a temporary (prvalue)
CXXNewExpr             — new T / new T[n]; carries array flag, placement args,
                         optional init expression
CXXDeleteExpr          — delete p / delete[] p; carries deallocation function
CXXThrowExpr           — throw e; optional subexpression (null for rethrow)
CXXDefaultArgExpr      — stands in for a missing default argument at call site
CXXDefaultInitExpr     — stands in for a field's in-class default initializer
CXXBindTemporaryExpr   — wraps a temporary that needs its destructor called
ExprWithCleanups       — scopes a set of cleanup objects (temporaries + EH cleanups)
MaterializeTemporaryExpr — makes a prvalue into a glvalue so it can be bound
LambdaExpr             — closure type + capture list + call operator body
CXXScalarValueInitExpr — value-initialization T{} for scalars
CXXFoldExpr            — C++17 fold expression: (args op ...) or (... op args)
CXXNoexceptExpr        — noexcept(expr) operator; carries constexpr bool result
SizeOfPackExpr         — sizeof...(Pack) for variadic templates
```

`LambdaExpr` is particularly rich: `capture_begin()`/`capture_end()` iterate `LambdaCapture` objects recording default capture mode, explicit captures with their kinds (`LCK_This`, `LCK_ByCopy`, `LCK_ByRef`, `LCK_VLAType`, `LCK_StarThis`), and the initializer for init-captures. `getCallOperator()` returns the `CXXMethodDecl` for `operator()`. Internally Sema creates a `CXXRecordDecl` for the closure type — accessible via `getLambdaClass()` — and populates it with data members for each captured entity.

`ParenExpr` and `ConstantExpr` are wrappers: `ParenExpr` preserves parentheses for diagnostics and for preventing some semantic rewritings; `ConstantExpr` caches a computed constant value alongside the expression subtree so subsequent evaluation queries (`EvaluateAsInt`, `EvaluateAsFloat`, `EvaluateAsRValue`) can short-circuit.

`InitListExpr` covers brace-initialization lists. For aggregates, Sema produces a *syntactic* form (preserving user-written elements) and a *semantic* form (filling in designated-initializer positions and implicit zero initializers) linked via `InitListExpr::getSyntacticForm()` / `getSemanticForm()`. Designated initializers are handled through `DesignatedInitExpr`, which carries the designator path (array indices or field chains) alongside the initializer value.

---

## 36.6 Value Categories and the Expression Value-Kind System

C++11 formalized a three-way classification of expression results: lvalue (a locatable object), prvalue (a pure rvalue with no identity), and xvalue (an expiring value, produced by `std::move` or a `&&` member access). Clang encodes this at every expression node.

### 36.6.1 `ExprValueKind` and `ExprObjectKind`

```cpp
enum ExprValueKind : unsigned {
    VK_PRValue,   // prvalue — no address, consumed directly
    VK_LValue,    // lvalue — has a persistent address
    VK_XValue,    // xvalue — has an address but is expiring
};

enum ExprObjectKind : unsigned {
    OK_Ordinary,        // normal object
    OK_BitField,        // non-addressable bit-field member
    OK_ObjCProperty,    // ObjC property access
    OK_ObjCSubscript,   // ObjC subscript
    OK_VectorComponent, // SIMD vector element
    OK_MatrixComponent, // matrix element (Clang extension)
};
```

Query methods on `Expr`:

```cpp
ExprValueKind getValueKind() const;
ExprObjectKind getObjectKind() const;
bool isLValue() const;   // VK_LValue
bool isPRValue() const;  // VK_PRValue
bool isXValue() const;   // VK_XValue
bool isGLValue() const;  // VK_LValue || VK_XValue
```

### 36.6.2 The lvalue-to-rvalue Conversion

When a named variable is read, Sema inserts an `ImplicitCastExpr` of kind `CK_LValueToRValue` above the `DeclRefExpr`. The `DeclRefExpr` is a VK_LValue (the variable has an address); the `ImplicitCastExpr` is a VK_PRValue (the loaded value). Inspecting the AST for `return i;` shows:

```
ReturnStmt
`-ImplicitCastExpr 'int' <LValueToRValue>
  `-DeclRefExpr 'int' lvalue Var 'i'
```

CodeGen checks for the `CK_LValueToRValue` cast kind to emit a load instruction. Static analyzers track the cast to know when values cross from address space to value space. This explicitness is what makes the "close to source" philosophy cost-neutral for downstream consumers.

### 36.6.3 xvalues and `std::move`

An xvalue is produced whenever a named rvalue reference is accessed or when `static_cast<T&&>(lvalue)` is applied. The standard library's `std::move` is effectively an alias for the latter, so:

```cpp
std::string s = "hello";
std::string t = std::move(s);
```

produces a call to `std::move<std::string&>` whose return type is `std::string&&` and whose `ExprValueKind` is `VK_XValue`. CodeGen sees VK_XValue and does not emit a copy; it instead passes the address of `s` to the move constructor.

`Expr::isGLValue()` tests for the union of lvalues and xvalues — any expression that can be bound to a reference. The C++ value-category taxonomy maps directly to these three enum values: glvalue = lvalue | xvalue, rvalue = prvalue | xvalue.

### 36.6.4 `OK_BitField` and Non-Ordinary Object Kinds

When `getObjectKind()` returns `OK_BitField`, the expression produces a bit-field lvalue. CodeGen emits a different code sequence: it must mask and shift to read or write individual bits within the containing storage unit. `Expr::refersToBitField()` is a shortcut for `getObjectKind() == OK_BitField`. 

`OK_VectorComponent` arises from SIMD vector element accesses like `vec.x` on Clang's `__attribute__((vector_size(...)))` types. `OK_MatrixComponent` arises from the Clang matrix extension's element access via `m[r][c]`. Both require special CodeGen handling (using `extractelement`/`insertelement` LLVM IR instructions for vector components).

---

## 36.7 `RecursiveASTVisitor`

[`clang/include/clang/AST/RecursiveASTVisitor.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/RecursiveASTVisitor.h) provides a CRTP visitor that traverses the entire AST calling user-supplied hooks.

### 36.7.1 Class Structure

```cpp
template <typename Derived>
class RecursiveASTVisitor {
public:
  bool TraverseDecl(Decl *D);
  bool TraverseStmt(Stmt *S, DataRecursionQueue *Queue = nullptr);
  bool TraverseType(QualType T, bool TraverseQualifier = true);
  bool TraverseTypeLoc(TypeLoc TL, bool TraverseQualifier = true);

  // Policy hooks (override to change traversal behaviour):
  bool shouldVisitTemplateInstantiations() const { return false; }
  bool shouldVisitImplicitCode() const { return false; }
  bool shouldTraversePostOrder() const { return false; }
};
```

The CRTP pattern means `Derived::VisitFunctionDecl(FunctionDecl*)` is called for every `FunctionDecl` node. The three-tier method hierarchy is:

1. **Traverse***: entry points. Call `WalkUpFrom*` then recurse into children. Override to skip subtrees.
2. **WalkUpFrom***: walk the class hierarchy calling `WalkUpFromBase` then `Visit*`. Override to add pre-visit logic.
3. **Visit***: the leaf hook. Override to act on a node without changing traversal.

### 36.7.2 Writing a Visitor

A visitor that counts function definitions:

```cpp
#include "clang/AST/RecursiveASTVisitor.h"
#include "clang/AST/Decl.h"

class FunctionCounter : public clang::RecursiveASTVisitor<FunctionCounter> {
public:
  unsigned Count = 0;

  bool VisitFunctionDecl(clang::FunctionDecl *FD) {
    if (FD->isThisDeclarationADefinition())
      ++Count;
    return true; // true = continue traversal
  }
};

// Usage:
FunctionCounter Visitor;
Visitor.TraverseDecl(Context.getTranslationUnitDecl());
llvm::outs() << "Definitions: " << Visitor.Count << "\n";
```

### 36.7.3 Controlling Traversal

By default `shouldVisitTemplateInstantiations()` returns `false`, so template specializations are skipped unless overridden. This means a `RecursiveASTVisitor` will see `FunctionDecl` nodes only for template patterns, not for each `FunctionDecl` that is a specialization. Set `shouldVisitTemplateInstantiations() { return true; }` to visit both.

`shouldVisitImplicitCode()` defaults to `false`, so synthesized bodies (default constructors, synthesized copy/move operators, etc.) are not traversed. Returning `true` enables visiting implicit function bodies and `CXXDefaultArgExpr` / `CXXDefaultInitExpr` inlinings.

Returning `false` from any `Traverse*` or `Visit*` method halts traversal immediately and propagates `false` up the call stack. Returning `true` continues. This allows early exit when the first match is sufficient.

`shouldTraversePostOrder()` controls whether `Visit*` is called on the way down (pre-order, default) or up (post-order). The `dataTraverseStmtPre(Stmt*)` / `dataTraverseStmtPost(Stmt*)` hooks enable lightweight pre/post callbacks for statement subtrees without overriding the full traversal logic.

A subtle issue arises with template argument expressions. When `shouldVisitTemplateInstantiations()` is `false`, `TraverseDecl` skips specialization bodies entirely — it does not recursively traverse the template arguments of a `TemplateSpecializationType`. To visit type template arguments, override `TraverseTemplateArgument` or `VisitTemplateArgument`.

For large translation units with deep ASTs, `RecursiveASTVisitor` uses a data-recursive algorithm for `Stmt*` traversal (via `DataRecursionQueue`) to avoid stack overflow on deeply nested expressions such as long initializer lists or heavily chained method calls.

---

## 36.8 `ASTConsumer` and `FrontendAction`

Tools that process an entire translation unit plug into Clang via the `ASTConsumer` / `FrontendAction` pair.

### 36.8.1 `ASTConsumer`

Defined in [`clang/include/clang/AST/ASTConsumer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/ASTConsumer.h):

```cpp
class ASTConsumer {
public:
  virtual bool HandleTopLevelDecl(DeclGroupRef D);
  virtual void HandleTranslationUnit(ASTContext &Ctx) {}
  virtual void HandleTagDeclDefinition(TagDecl *D) {}
  virtual void HandleInterestingDecl(DeclGroupRef D);
  // ...
};
```

`HandleTopLevelDecl` is called incrementally as each top-level declaration is parsed, enabling streaming processing. `HandleTranslationUnit` is called once after the entire file is parsed and all implicit definitions have been generated; at this point the AST is complete and stable.

### 36.8.2 `FrontendAction`

```cpp
class ASTFrontendAction : public FrontendAction {
protected:
  virtual std::unique_ptr<ASTConsumer>
  CreateASTConsumer(CompilerInstance &CI, StringRef InFile) = 0;
};
```

Subclass `ASTFrontendAction`, implement `CreateASTConsumer()`, and register the action with a `FrontendActionFactory`. For unit testing, `clang::tooling::runToolOnCode()` takes an action and a code string:

```cpp
#include "clang/Tooling/Tooling.h"

struct MyAction : clang::ASTFrontendAction {
  std::unique_ptr<clang::ASTConsumer>
  CreateASTConsumer(clang::CompilerInstance &CI, llvm::StringRef) override {
    return std::make_unique<MyConsumer>(CI.getASTContext());
  }
};

bool ok = clang::tooling::runToolOnCode(
    std::make_unique<MyAction>(), "int main() { return 0; }");
```

### 36.8.3 `ASTUnit`

`clang::ASTUnit` ([`clang/include/clang/Frontend/ASTUnit.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/ASTUnit.h)) is a self-contained compilation unit suitable for in-process use by IDEs. It owns its `ASTContext`, `Preprocessor`, `Sema`, and the serialized preamble. `ASTUnit::LoadFromCompilerInvocation()` and `ASTUnit::LoadFromCommandLine()` build an AST and keep it resident, allowing incremental re-parsing on edit.

The preamble feature is critical for IDE performance: the initial compilation parses and serializes all headers (the "preamble") into a PCH-like blob. On subsequent edits to the main file, only the changed portion is re-parsed; the preamble is deserialized from the in-memory blob. `ASTUnit::Reparse()` drives this incremental path.

`ASTUnit` also aggregates `StoredDiagnostic` objects from previous compilations so that IDEs can display stale diagnostics until the next successful parse, rather than showing no diagnostics during a transient parse error.

### 36.8.4 `ClangTool` and Compilation Databases

For batch processing of multiple files, `clang::tooling::ClangTool` ([`clang/include/clang/Tooling/Tooling.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Tooling/Tooling.h)) drives a `FrontendActionFactory` over all files listed in a JSON compilation database:

```cpp
#include "clang/Tooling/CommonOptionsParser.h"
#include "clang/Tooling/Tooling.h"

int main(int argc, const char **argv) {
  auto ExpectedParser =
      clang::tooling::CommonOptionsParser::create(argc, argv, MyCategory);
  clang::tooling::ClangTool Tool(
      ExpectedParser->getCompilations(),
      ExpectedParser->getSourcePathList());
  return Tool.run(
      clang::tooling::newFrontendActionFactory<MyAction>().get());
}
```

`CommonOptionsParser` handles `--` and `-p` arguments automatically, loading `compile_commands.json` from the project directory. Each source file is processed in a separate `CompilerInstance` with the flags from the compilation database, ensuring that per-file macros and include paths are correct.

---

## 36.9 The `ASTMatcher` Library

The `clang::ast_matchers` namespace ([`clang/include/clang/ASTMatchers/ASTMatchers.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/ASTMatchers/ASTMatchers.h)) provides a domain-specific embedded language for querying the AST.

### 36.9.1 Matcher Types

```cpp
using DeclarationMatcher = internal::Matcher<Decl>;
using StatementMatcher   = internal::Matcher<Stmt>;
using TypeMatcher        = internal::Matcher<QualType>;
```

Matchers compose:

```cpp
DeclarationMatcher M =
    functionDecl(
        hasName("memcpy"),
        hasParameter(0, hasType(pointerType())),
        isDefinition()
    );
```

Structural combinators include:
- `has(m)` — direct child matches `m`
- `hasDescendant(m)` — any descendant matches `m`
- `forEachDescendant(m)` — binds all matching descendants
- `unless(m)` — negation
- `anyOf(m1, m2, ...)` — disjunction
- `allOf(m1, m2, ...)` — conjunction (also written as argument list)
- `isExpansionInMainFile()` — location predicate

### 36.9.2 `MatchFinder` and Callbacks

```cpp
#include "clang/ASTMatchers/ASTMatchFinder.h"

struct Handler : clang::ast_matchers::MatchFinder::MatchCallback {
  void run(const clang::ast_matchers::MatchFinder::MatchResult &R) override {
    if (auto *FD = R.Nodes.getNodeAs<clang::FunctionDecl>("fn")) {
      llvm::outs() << "Matched: " << FD->getNameAsString() << "\n";
    }
  }
};

Handler H;
clang::ast_matchers::MatchFinder Finder;
Finder.addMatcher(
    functionDecl(hasName("foo")).bind("fn"), &H);

// Run over an ASTUnit:
Finder.matchAST(Context);
```

The `.bind("fn")` call associates a string key with the matched node; `R.Nodes.getNodeAs<FunctionDecl>("fn")` retrieves it. Matchers are the foundation of clang-tidy checks (Chapter 47) and clang-query, the interactive REPL for exploratory AST queries.

### 36.9.3 Composing Complex Queries

Matchers for C++-specific constructs use the `cxx*` naming convention:

```cpp
// Find all virtual function calls through a base-class pointer
StatementMatcher VirtualCall =
    cxxMemberCallExpr(
        on(hasType(pointsTo(cxxRecordDecl()))),
        callee(cxxMethodDecl(isVirtual()))
    ).bind("vcall");

// Find STL containers storing raw pointers
TypeMatcher RawPtrContainer =
    classTemplateSpecializationType(
        hasDeclaration(classTemplateDecl(
            hasName("vector"))),
        hasTemplateArgument(0,
            refersToType(pointerType()))
    );

// Find all functions that call malloc but never call free
DeclarationMatcher MallocWithoutFree =
    functionDecl(
        hasDescendant(callExpr(callee(functionDecl(hasName("malloc"))))),
        unless(hasDescendant(callExpr(callee(functionDecl(hasName("free"))))))
    ).bind("leaky");
```

The `eachOf` and `forEachDescendant` matchers are important for collecting all matches rather than just the first. `eachOf(m1, m2)` tries both matchers and triggers the callback for each that succeeds, which is useful for binding multiple node types. `forEachDescendant(m)` binds every node in the subtree matching `m` — note that it re-invokes the callback for each match, which can produce many callbacks for deeply nested structures.

Matchers can also be composed into named variables and reused:

```cpp
using namespace clang::ast_matchers;

// Named sub-matcher for reuse
auto IsInSystemHeader = isExpansionInSystemHeader();

DeclarationMatcher UserDefined =
    cxxRecordDecl(unless(IsInSystemHeader)).bind("rec");
```

The `hasAncestor` matcher enables upward traversal — unusual in ASTs, which are typically traversed downward. It is implemented by walking the parent map stored in `ASTContext`, which is lazily built on first use. The parent map is a `DynTypedNodeList` mapping from child nodes to sets of typed parents.

---

## 36.10 Implicit AST Nodes

Clang synthesizes a large number of AST nodes that the programmer did not write. These are real first-class nodes with full type information, though their source locations are typically `<invalid sloc>`.

### 36.10.1 Implicit Special Member Functions

When a `CXXRecordDecl` is completed, Sema evaluates whether each special member function is implicitly declared. The `CXXRecordDecl::DefinitionData` struct tracks:

```cpp
bool needsImplicitDefaultConstructor() const;
bool needsImplicitCopyConstructor() const;
bool needsImplicitMoveConstructor() const;
bool needsImplicitCopyAssignment() const;
bool needsImplicitMoveAssignment() const;
bool needsImplicitDestructor() const;
```

When implicit declaration is needed, Sema creates the declaration lazily:

```cpp
// clang/include/clang/Sema/Sema.h
CXXConstructorDecl *DeclareImplicitDefaultConstructor(CXXRecordDecl *ClassDecl);
CXXConstructorDecl *DeclareImplicitCopyConstructor(CXXRecordDecl *ClassDecl);
CXXConstructorDecl *DeclareImplicitMoveConstructor(CXXRecordDecl *ClassDecl);
CXXMethodDecl *DeclareImplicitCopyAssignment(CXXRecordDecl *ClassDecl);
CXXMethodDecl *DeclareImplicitMoveAssignment(CXXRecordDecl *ClassDecl);
CXXDestructorDecl *DeclareImplicitDestructor(CXXRecordDecl *ClassDecl);
```

The resulting `CXXConstructorDecl` has `isImplicit() == true` and, once *defined* (lazily on first ODR use), carries a synthesized `CompoundStmt` body. `RecursiveASTVisitor` will only visit these bodies if `shouldVisitImplicitCode()` returns `true`.

### 36.10.2 `ImplicitCastExpr` Nodes

Every standard conversion sequence is represented as a chain of `ImplicitCastExpr` wrappers. A call `double d = intVar;` wraps `DeclRefExpr(intVar)` in `ImplicitCastExpr<CK_LValueToRValue>` and then `ImplicitCastExpr<CK_IntegralToFloating>`. The AST dump shows:

```
ImplicitCastExpr 'double' <IntegralToFloating>
`-ImplicitCastExpr 'int' <LValueToRValue>
  `-DeclRefExpr 'int' lvalue Var 'intVar'
```

Other common cast kinds: `CK_FunctionToPointerDecay`, `CK_ArrayToPointerDecay`, `CK_DerivedToBase`, `CK_NullToPointer`, `CK_IntegralCast`.

### 36.10.3 `CXXDefaultArgExpr`

When a call site omits a defaulted argument, Sema inserts a `CXXDefaultArgExpr` into the call's argument list. The node carries a pointer to the `ParmVarDecl` whose default expression it represents. The source location is `<invalid sloc>` because the user wrote no argument there, but the expression's value is the default's constant-evaluated result. This is visible in the AST dump:

```
CallExpr 'void'
|-ImplicitCastExpr <FunctionToPointerDecay>
| `-DeclRefExpr 'foo'
|-IntegerLiteral 'int' 1
`-CXXDefaultArgExpr 'int'          <<invalid sloc>>
  `-IntegerLiteral 'int' 42
```

### 36.10.4 `MaterializeTemporaryExpr` and `ExprWithCleanups`

When a prvalue is bound to a `const T&` or `T&&`, C++ mandates that the temporary's lifetime be extended. Sema wraps the prvalue in a `MaterializeTemporaryExpr`, which gives it a glvalue identity for the duration of the reference:

```cpp
const std::string &r = std::string("hello"); // extends lifetime
```

The AST contains:
```
VarDecl r 'const std::string &' cinit
`-MaterializeTemporaryExpr 'const std::string' xvalue
  `-CXXTemporaryObjectExpr 'std::string' ...
```

`ExprWithCleanups` wraps a statement or expression that requires destructor calls for temporaries created within it. The cleanup objects (either `BlockDecl*` for block captures or `CompoundLiteralExpr*`) are stored alongside the wrapped expression.

`CXXBindTemporaryExpr` appears inside `ExprWithCleanups` to mark where a `CXXDestructorDecl` must be called at the end of the full expression.

---

## 36.11 Debugging and Dumping the AST

### 36.11.1 Command-Line Dumps

```bash
# Full AST in textual form
clang -Xclang -ast-dump -fsyntax-only foo.cpp

# Pretty-print source reconstruction
clang -Xclang -ast-print -fsyntax-only foo.cpp

# Restrict to a single JSON format
clang -Xclang -ast-dump=json -fsyntax-only foo.cpp
```

The `-ast-dump` output uses color coding when the terminal supports it: declarations in cyan, types in green, source locations in yellow.

### 36.11.2 Programmatic Dumps

```cpp
// From ASTContext
Context.getTranslationUnitDecl()->dump();

// Any Decl node
FD->dump();

// Any Stmt node
Body->dump();

// Any QualType
QT.dump();
```

These route through `clang::ASTDumper` ([`clang/include/clang/AST/ASTDumper.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/ASTDumper.h)), which inherits from `ASTNodeTraverser<ASTDumper, TextNodeDumper>`. `TextNodeDumper` handles the per-node line formatting while `ASTNodeTraverser` handles the traversal logic. To dump a single node without its subtree, use `TextNodeDumper` directly; to dump a full subtree, use `ASTDumper`.

### 36.11.3 `clang-query`

`clang-query` is an interactive REPL for exploring AST matchers against live source. To start:

```bash
clang-query foo.cpp --
```

At the prompt, type AST matcher expressions:

```
match functionDecl(hasName("main"))
set output dump
match returnStmt(hasAncestor(functionDecl(hasName("main"))))
set output print
match varDecl(hasType(pointerType()))
```

Output modes include `dump` (full textual AST dump of matching nodes), `print` (pretty-printed source reconstruction), `diag` (diagnostic-style caret output), and `detailed-ast` (dump with source range annotations). The `let` command binds named matchers for reuse:

```
let fn functionDecl(hasName("processData"))
match callExpr(callee(fn))
```

`clang-query` accepts a `-p <compile_commands.json>` argument to load compile commands from a build directory, enabling accurate preprocessing of headers and macros for real projects.

### 36.11.4 Reducing AST Noise

When dumping large files, the AST output can be overwhelming because it includes all implicit `TypedefDecl` nodes for built-in types and all header content. Several techniques reduce noise:

```bash
# Restrict dump to a specific location range
clang -Xclang -ast-dump -fsyntax-only foo.cpp 2>&1 | grep -A 20 "foo.cpp:10"

# Use -ast-dump-filter to match node names
clang -Xclang "-ast-dump-filter=main" -Xclang -ast-dump \
      -fsyntax-only foo.cpp

# JSON format for programmatic post-processing
clang -Xclang -ast-dump=json -fsyntax-only foo.cpp > ast.json
python3 -c "import json,sys; ast=json.load(open('ast.json')); ..."
```

The JSON dump includes all field data for every node and is the basis for tools that process ASTs outside the Clang process boundary — for example, cross-language bridges that consume Clang ASTs from Python or Rust.

---

## 36.12 Source Locations in the AST

Every AST node carries precise source location information, which is the foundation of Clang's diagnostic quality and refactoring correctness.

### 36.12.1 `Decl` Locations

```cpp
// The "name" location — position of the identifier token
SourceLocation Decl::getLocation() const;

// Full syntactic range including storage class, type, initializer
SourceRange Decl::getSourceRange() const;

// Convenience
SourceLocation Decl::getBeginLoc() const;
SourceLocation Decl::getEndLoc() const;
```

For a `FunctionDecl`, `getLocation()` returns the function name position; `getBeginLoc()` returns the return type; `getEndLoc()` returns the closing brace of the body.

### 36.12.2 `Stmt` and `Expr` Locations

```cpp
SourceLocation Stmt::getBeginLoc() const;
SourceLocation Stmt::getEndLoc() const;
SourceRange Stmt::getSourceRange() const;
```

Subclasses add specific locations:

```cpp
SourceLocation CallExpr::getRParenLoc() const;     // closing paren of call
SourceLocation MemberExpr::getMemberLoc() const;   // member name token
SourceLocation BinaryOperator::getOperatorLoc() const;
SourceLocation IfStmt::getIfLoc() const;
SourceLocation IfStmt::getElseLoc() const;
```

### 36.12.3 Extending Ranges

`SourceLocation` values mark the start of a token. To compute the character-level end position, use `Lexer::getLocForEndOfToken()`:

```cpp
#include "clang/Lex/Lexer.h"

SourceLocation End = clang::Lexer::getLocForEndOfToken(
    CallExpr->getRParenLoc(), 0, SM, LangOpts);
```

`CharSourceRange` represents a range in character units rather than token units, necessary for text replacement in refactoring:

```cpp
CharSourceRange Range = CharSourceRange::getCharRange(Begin, End);
// or
CharSourceRange Range = CharSourceRange::getTokenRange(Begin, End);
```

`CharSourceRange::getTokenRange(Begin, End)` expands `End` to include the last token; `getCharRange(Begin, End)` treats `End` as a raw character offset.

### 36.12.4 Macro Expansion Locations

`SourceManager::isMacroBodyExpansion(Loc)` and `isMacroArgExpansion(Loc)` distinguish body and argument expansion locations. `SourceManager::getSpellingLoc(Loc)` maps an expansion location back to where the token was physically written; `getExpansionLoc(Loc)` maps back to the macro call site. For refactoring tools that must not modify text inside a macro definition, this distinction is critical.

For compound macros (a macro expanding to another macro), `getImmediateSpellingLoc()` and `getImmediateMacroCallerLoc()` provide one level of unwinding rather than fully resolving the chain. `getFileLoc()` resolves any expansion to a file location (the spelling location of the innermost expansion).

### 36.12.5 Source Locations and Rewriting

The `clang::Rewriter` class ([`clang/include/clang/Rewrite/Core/Rewriter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Rewrite/Core/Rewriter.h)) builds on `SourceManager` to perform source-level text transformations guided by AST node locations. It accumulates insertions, replacements, and removals in a `RewriteBuffer` per file, then serializes the modified text:

```cpp
clang::Rewriter RW(SM, LangOpts);

// Replace a CallExpr with a different call
SourceRange SR = CallE->getSourceRange();
RW.ReplaceText(SR, "new_call(args)");

// Insert before a declaration
RW.InsertText(FD->getBeginLoc(), "[[nodiscard]] ", /*InsertAfter=*/false);

// Write output
const clang::RewriteBuffer *RBuf = RW.getRewriteBufferFor(SM.getMainFileID());
llvm::outs() << std::string(RBuf->begin(), RBuf->end());
```

`clang-apply-replacements` and `clang::tooling::Replacements` extend this to batch-apply replacements computed by multiple independent matcher callbacks, merging them and detecting conflicts before any file is modified.

---

## Chapter Summary

- Clang's AST is source-faithful, not normalized: it preserves sugar, implicit nodes, and source locations throughout, making it suitable simultaneously for diagnostics, static analysis, refactoring, and codegen.
- `ASTContext` owns all AST memory via a bump-pointer arena, performs type uniquing via folding sets, and provides size/alignment queries through target-aware type info.
- The `Decl` hierarchy roots at `Decl` → `NamedDecl` → `ValueDecl` → `DeclaratorDecl`, with `FunctionDecl`, `VarDecl`, `FieldDecl`, `CXXRecordDecl`, `EnumDecl`, `TemplateDecl`, and `BindingDecl` covering all C++ declaration forms.
- `QualType` is a tagged pointer combining `Type*` with three qualifier bits; canonical types enable O(1) type equality; sugar types preserve source spelling for diagnostics.
- The `Stmt`/`Expr` hierarchy encodes every grammatical construct; `ImplicitCastExpr`, `CXXDefaultArgExpr`, `MaterializeTemporaryExpr`, `ExprWithCleanups`, and `CXXBindTemporaryExpr` make every implicit semantic action explicit in the tree.
- Every `Expr` carries an `ExprValueKind` (VK_LValue / VK_PRValue / VK_XValue) and an `ExprObjectKind`; the lvalue-to-rvalue conversion is modeled as an `ImplicitCastExpr<CK_LValueToRValue>`.
- `RecursiveASTVisitor<Derived>` provides pre-order CRTP traversal with separate Traverse/WalkUpFrom/Visit tiers; template instantiations and implicit code are opt-in.
- `ASTConsumer` receives incremental and final callbacks; `ASTFrontendAction` wires an action into the compilation pipeline; `ASTUnit` supports resident in-process ASTs for IDE tools.
- The `clang::ast_matchers` DSL enables declarative structural queries; `MatchFinder` + `MatchCallback` feed matched nodes to user code.
- Implicit synthesized members are real `CXXMethodDecl` nodes, declared lazily by `Sema::DeclareImplicit*` and visible to `RecursiveASTVisitor` when `shouldVisitImplicitCode()` returns `true`.
- Source locations pervade every node; `Lexer::getLocForEndOfToken()` extends token positions to character offsets; `CharSourceRange` provides the character-level ranges needed for text manipulation.

---

## Cross-References

- [Chapter 29 — SourceManager, FileEntry, SourceLocation](../part-05-clang-frontend/ch29-sourcemanager-fileentry-sourcelocation.md) — in-depth coverage of `SourceLocation` encoding and `SourceManager` APIs used throughout the AST.
- [Chapter 32 — The Parser](../part-05-clang-frontend/ch32-the-parser.md) — how the recursive-descent parser creates `Decl` and `Stmt` nodes and invokes `Sema` actions.
- [Chapter 33 — Sema I: Names, Lookups, and Conversions](../part-05-clang-frontend/ch33-sema-names-lookups-conversions.md) — how `Sema` annotates the AST with types, resolves names, and inserts `ImplicitCastExpr` nodes.
- [Chapter 34 — Sema II: Templates, Concepts, and Constraints](../part-05-clang-frontend/ch34-sema-templates-concepts-constraints.md) — template instantiation and how `TemplateSpecializationType` and `FunctionTemplateSpecializationInfo` populate the AST.
- [Chapter 39 — CodeGenModule and CodeGenFunction](../part-06-clang-codegen/ch39-codegenmodule-codegenfunction.md) — how CodeGen walks the AST nodes described here to emit LLVM IR.
- [Chapter 46 — libtooling and AST Matchers](../part-07-clang-multilang/ch46-libtooling-ast-matchers.md) — end-to-end examples of building Clang tools with `RecursiveASTVisitor` and `ASTMatcher`.

## Reference Links

- [`clang/include/clang/AST/ASTContext.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/ASTContext.h)
- [`clang/include/clang/AST/DeclBase.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/DeclBase.h)
- [`clang/include/clang/AST/Decl.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Decl.h)
- [`clang/include/clang/AST/DeclCXX.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/DeclCXX.h)
- [`clang/include/clang/AST/Type.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Type.h)
- [`clang/include/clang/AST/Stmt.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Stmt.h)
- [`clang/include/clang/AST/Expr.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Expr.h)
- [`clang/include/clang/AST/ExprCXX.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/ExprCXX.h)
- [`clang/include/clang/AST/RecursiveASTVisitor.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/RecursiveASTVisitor.h)
- [`clang/include/clang/AST/ASTConsumer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/ASTConsumer.h)
- [`clang/include/clang/Frontend/FrontendAction.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/FrontendAction.h)
- [`clang/include/clang/ASTMatchers/ASTMatchers.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/ASTMatchers/ASTMatchers.h)
- [`clang/include/clang/ASTMatchers/ASTMatchFinder.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/ASTMatchers/ASTMatchFinder.h)
- [`clang/include/clang/AST/ASTDumper.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/ASTDumper.h)
- [`clang/include/clang/Tooling/Tooling.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Tooling/Tooling.h)
- [`clang/include/clang/Sema/Sema.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/Sema.h) — `DeclareImplicit*` methods
