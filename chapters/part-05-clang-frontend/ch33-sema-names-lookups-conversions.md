# Chapter 33 — Sema I: Names, Lookups, and Conversions

*Part V — The Clang Frontend*

Semantic analysis is where a stream of syntactic constructs acquires meaning: types are resolved, names are bound to declarations, expressions are checked for well-formedness, and implicit conversions are inserted wherever the language mandates them. In Clang, all of this machinery lives in a single class, `Sema`, which serves as the "actions" object driven by the Parser. This chapter dissects the name-resolution and conversion half of `Sema`'s responsibilities — the subsystems a compiler writer touches first when adding a new language construct or diagnosing a subtle type mismatch. Templates and overload deduction are deferred to Chapter 34; the persistent AST node hierarchy is examined in Chapter 36.

## The `Sema` Class

`Sema` is defined in [`clang/include/clang/Sema/Sema.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/Sema.h) and implemented across roughly twenty translation units under `clang/lib/Sema/`. The header alone exceeds 14,000 lines. Rather than a monolithic object file, the implementation is partitioned by concern:

| Source file | Primary responsibility |
|---|---|
| `SemaDecl.cpp` | Variable, function, and tag declarations |
| `SemaExpr.cpp` | Expression semantic checks |
| `SemaExprCXX.cpp` | C++-specific expressions (casts, `new`, `delete`, `typeid`) |
| `SemaLookup.cpp` | Name lookup, ADL, typo correction |
| `SemaOverload.cpp` | Overload resolution, conversion sequences |
| `SemaTemplate.cpp` | Template instantiation and deduction |
| `SemaDeclCXX.cpp` | Class member declarations, special members |
| `SemaInit.cpp` | Initialization sequences |
| `SemaType.cpp` | Type specifier processing |

`Sema` is constructed once per translation unit by `CompilerInstance::createSema()` and lives for the translation unit's lifetime. Its key fields set the context for every action callback from the Parser:

```cpp
// clang/include/clang/Sema/Sema.h (selected fields)
class Sema : public SemaBase {
public:
  Preprocessor &PP;               // token stream, macro expansion
  ASTContext &Context;            // type uniquing, memory, node allocation
  ASTConsumer &Consumer;          // receives completed top-level decls
  DiagnosticsEngine &Diags;
  SourceManager &SourceMgr;

  DeclContext *CurContext;         // the decl context currently being parsed
  Scope *TUScope;                  // the translation-unit scope (depth 0)
  IdentifierResolver IdResolver;   // lexical scope shadowing chains
  // ...
};
```

`CurContext` tracks the `DeclContext` whose body is currently under construction — a `TranslationUnitDecl` at file scope, a `FunctionDecl` inside a function body, or a `CXXRecordDecl` inside a class definition. Every time the Parser enters a declarative region, it calls back into `Sema` to push `CurContext`; every time it leaves, Sema restores the previous context. The `ContextRAII` inner class provides a scoped guard:

```cpp
struct ContextRAII {
  Sema &S;
  DeclContext *SavedContext;
  ContextRAII(Sema &S, DeclContext *ContextToPush, bool NewThisContext = true)
      : S(S), SavedContext(S.CurContext) {
    S.CurContext = ContextToPush;
  }
  ~ContextRAII() { S.CurContext = SavedContext; }
};
```

The `TUScope` pointer is the root of the `Scope` chain. It is needed in Objective-C contexts and by any lookup that must restart from file scope. The `IdentifierResolver` (`clang/include/clang/Sema/IdentifierResolver.h`) is the shadow-chain store for unqualified lookup: it maps declaration names to a linked sequence of `NamedDecl*` entries representing all declarations currently on the lexical scope stack, without consulting the `DeclContext` hierarchy at all.

### Decomposition into Sub-Objects

In Clang 22, large target-specific and language-specific subsets of the `Sema` API have been extracted into semi-independent sub-objects accessible as member objects:

```cpp
SemaARM ARM;
SemaX86 X86;
SemaOpenMP OpenMP;
SemaObjC ObjC;
SemaCUDA CUDA;
SemaHLSL HLSL;
SemaSPIRV SPIRV;
SemaSYCL SYCL;
// etc.
```

Each sub-object inherits from `SemaBase` (giving it access to `Sema &SemaRef`) and is initialized by `Sema`'s constructor. The Parser still invokes methods directly on `Sema` for language-neutral actions, but target-specific builtins, pragma handling, and language extensions are dispatched to the appropriate sub-object. This decomposition reduces include-time coupling and compilation times without changing the visible calling conventions for callers of the main `Sema` object.

`SemaBase` provides:

```cpp
class SemaBase {
protected:
  Sema &SemaRef;
  ASTContext &getASTContext() const;
  DiagnosticsEngine &getDiagnostics() const;
  DiagnosticBuilder Diag(SourceLocation Loc, unsigned DiagID);
  DiagnosticBuilder Diag(SourceLocation Loc, const PartialDiagnostic &PD);
};
```

The `Diag()` convenience method is heavily used throughout Sema's implementation; it issues a diagnostic at a given source location. The `PartialDiagnostic` variant (used extensively in overload-resolution error paths) delays formatting until it is determined whether the diagnostic will actually be emitted — avoiding expensive string operations when no error occurs.

## Scopes and Scope Management

### The `Scope` Type

`clang::Scope` (`clang/include/clang/Sema/Scope.h`) is a transient data structure created during parsing and discarded after it. It is *not* the same as a `DeclContext`: a `DeclContext` is persistent AST, while a `Scope` is a scratch pad for name lookup and break/continue target resolution. The two are linked — each `Scope` optionally stores a `DeclContext *Entity` field pointing to the AST node it represents — but many scopes are pure control-flow constructs with no associated `DeclContext`.

Each scope carries an unsigned `Flags` field that is a bitwise OR of values from the `ScopeFlags` enumeration:

```cpp
enum ScopeFlags : unsigned {
  FnScope                  = 0x01,  // labels resolved at this boundary
  BreakScope               = 0x02,  // break statement exits here
  ContinueScope            = 0x04,  // continue statement targets here
  DeclScope                = 0x08,  // can contain declarations
  ControlScope             = 0x10,  // if/switch/while/for condition
  ClassScope               = 0x20,  // struct/union/class body
  BlockScope               = 0x40,  // ObjC ^block or lambda body
  TemplateParamScope       = 0x80,  // C++ template parameter list
  FunctionPrototypeScope   = 0x100, // function declarator parameter list
  FunctionDeclarationScope = 0x200, // function declaration (not definition)
  AtCatchScope             = 0x400, // ObjC @catch
  ObjCMethodScope          = 0x800, // ObjC method body
  SwitchScope              = 0x1000,
  TryScope                 = 0x2000,
  FnTryCatchScope          = 0x4000,
  EnumScope                = 0x40000,
  SEHTryScope              = 0x80000,
  CompoundStmtScope        = 0x400000,
  ClassInheritanceScope    = 0x800000, // between ':' and '{' in class head
  CatchScope               = 0x1000000,
  TypeAliasScope           = 0x40000000,
  FriendScope              = 0x80000000,
  // Plus OpenMP, OpenACC, lambda scopes
};
```

A compound-statement scope inside a function body will have `FnScope | DeclScope | CompoundStmtScope`. A for-loop's body adds `BreakScope | ContinueScope`. A template parameter list creates a scope with only `TemplateParamScope | DeclScope`.

### Scope Chains and Shortcut Pointers

`Scope` objects form a singly-linked parent chain via `AnyParent`. Several shortcut pointers accelerate common traversals without a linear parent walk:

- `FnParent` — nearest enclosing function scope (label lookup needs this)
- `BreakParent` / `ContinueParent` — nearest break/continue target
- `BlockParent` — nearest block (lambda/Objective-C block) scope
- `TemplateParamParent` — nearest template-parameter scope
- `DeclParent` — nearest scope whose `DeclScope` bit is set

The scope depth `Depth` (0 = TU scope) is used by `Scope::Contains()` to compare scope ancestry without walking the chain. The `isDeclScope(const Decl *D)` query delegates to `DeclsInScope.contains(D)`.

The parser allocates and deallocates `Scope` objects via its own `EnterScope` / `ExitScope` helpers; on scope exit the Parser calls `Sema::ActOnPopScope(SourceLocation Loc, Scope *S)`, which removes that scope's declarations from the `IdentifierResolver` shadow chains so they no longer appear in unqualified lookup. Each `Scope` also records `using`-directive declarations in a `SmallVector<UsingDirectiveDecl*, 2> UsingDirectives`; `ActOnPopScope` propagates these upward to the enclosing declaration scope.

## `DeclContext` and Declaration Containers

`DeclContext` is the base class of every AST node that can own named declarations. It provides a linked list of `Decl*` (maintained in source order) and a lazily-built hash map keyed by `DeclarationName` for O(1) lookup by name.

The primary concrete subclasses:

| Class | Role |
|---|---|
| `TranslationUnitDecl` | Root of every TU; represents the global namespace |
| `NamespaceDecl` | Named or anonymous namespace; may be inline |
| `CXXRecordDecl` | `struct`, `class`, or `union` body |
| `FunctionDecl` | Function body (contains `ParmVarDecl`, local types, nested functions) |
| `ObjCMethodDecl` | Objective-C method body |
| `BlockDecl` | Clang extension `^block` body |
| `LinkageSpecDecl` | `extern "C"` / `extern "C++"` region |
| `EnumDecl` | Enumeration body (enumerators are `EnumConstantDecl`) |
| `ObjCContainerDecl` | ObjC class/category/extension/protocol |

### Iterating and Searching

Iterating all declarations in a context:

```cpp
DeclContext *DC = /* some DeclContext */;
for (Decl *D : DC->decls()) {
  if (auto *ND = dyn_cast<NamedDecl>(D))
    llvm::outs() << ND->getNameAsString() << "\n";
}
```

Name lookup within a specific context uses `DeclContext::lookup(DeclarationName)`, which returns a `DeclContextLookupResult` (a range over `NamedDecl*`). This method builds the lookup table on first call by scanning `decls()` and building a `StoredDeclsMap`. For declarations added after the table was built, `DeclContext::localUncachedLookup()` does a linear scan of the raw decl list.

### `CurContext` Management

`Sema::EnterDeclaratorContext(Scope *S, DeclContext *DC)` pushes a new `DeclContext` by setting `CurContext = DC` and associating it with scope `S` via `S->setEntity(DC)`. The inverse `Sema::ExitDeclaratorContext(Scope *S)` pops to the saved context. These are called from the parser's class/namespace/function entry/exit hooks. For inline member function definitions that appear outside the class body (a C++ corner case), `Sema::EnterTemplatedContext()` re-enters template parameter scopes in the correct order first.

`Sema::getContainingDC(DeclContext *DC)` walks the DeclContext parent chain skipping `LinkageSpecDecl` wrappers, because `extern "C"` does not create a new semantic scope for the purpose of qualified name resolution.

## Name Lookup Machinery

### `LookupResult`

The central data structure for any name lookup is `LookupResult`, declared in [`clang/include/clang/Sema/Lookup.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/Lookup.h). It is constructed by the caller with the name to search, a `LookupNameKind`, and optionally a `RedeclarationKind`:

```cpp
LookupResult R(SemaRef, DeclarationName(&II), NameLoc,
               Sema::LookupOrdinaryName);
if (SemaRef.LookupName(R, CurScope)) {
  switch (R.getResultKind()) {
  case LookupResultKind::Found:
    NamedDecl *D = R.getFoundDecl();
    break;
  case LookupResultKind::FoundOverloaded:
    for (NamedDecl *D : R)
      processOverload(D);
    break;
  case LookupResultKind::Ambiguous:
    // R.getAmbiguityKind() reveals AmbiguousBaseSubobjectTypes, etc.
    break;
  case LookupResultKind::NotFound:
    // consider typo correction
    break;
  }
}
```

The `ResultKind` enumeration has six values: `NotFound`, `NotFoundInCurrentInstantiation` (there are dependent bases that could not be searched), `Found` (single declaration), `FoundOverloaded` (multiple declarations with the same name forming an overload set), `FoundUnresolvedValue` (dependent using declaration), and `Ambiguous`.

`LookupResult` carries an `UnresolvedSet<8>` for the found declarations, a `CXXBasePaths*` for base-class ambiguity information, a `NamingClass` (the class through which member lookup was performed), and flags controlling whether tags are hidden and whether hidden module-private declarations are visible.

The three `LookupAmbiguityKind` values signal distinct ambiguity patterns: `AmbiguousBaseSubobjectTypes` (same name found in two distinct base class types), `AmbiguousBaseSubobjects` (same name found in two subobjects of the same base type — the virtual/non-virtual diamond), `AmbiguousReference` (same name found through multiple `using`-directives in the same scope), and `AmbiguousTagHiding` (a tag hidden by a non-tag from a different namespace).

### `LookupNameKind`

The lookup kind determines which identifier namespace (IDNS) bits are searched. The enumeration from `Sema.h`:

```cpp
enum LookupNameKind {
  LookupOrdinaryName = 0,          // functions, vars, typedefs — most lookups
  LookupTagName,                   // enums, structs, classes, unions
  LookupLabel,                     // goto targets
  LookupMemberName,                // class/struct/union members
  LookupOperatorName,              // operator+, operator[], etc.
  LookupDestructorName,            // ~T (prefers tags over typedefs)
  LookupNestedNameSpecifierName,   // N in N::x (namespaces, types only)
  LookupNamespaceName,             // using namespace N (namespaces only)
  LookupUsingDeclName,             // redecl checks for using-declarations
  LookupRedeclarationWithLinkage,  // C99 6.2.2p4-5; ignores non-linkage decls
  LookupLocalFriendName,           // C++11 [class.friend]p11
  LookupObjCProtocolName,
  LookupObjCImplicitSelfParam,
  LookupOMPReductionName,
  LookupOMPMapperName,
  LookupAnyName                    // any declaration (for code completion)
};
```

`LookupOrdinaryName` in C++ searches `IDNS_Ordinary | IDNS_Tag | IDNS_Member | IDNS_Namespace`. `LookupTagName` uses `IDNS_Tag | IDNS_Type`. `LookupNestedNameSpecifierName` excludes ordinary names, permitting only namespaces, types, and namespace aliases. The mapping is performed by the private `getIDNS()` helper in `SemaLookup.cpp`.

### Unqualified Lookup

`Sema::LookupName(LookupResult &R, Scope *S, bool AllowBuiltinCreation, bool ForceNoCPlusPlus)` performs unqualified lookup starting from scope `S`. The C++ path is `Sema::CppLookupName()`. The algorithm:

1. **`IdentifierResolver` chain**: Walk the shadow chain for `R.getLookupName()`. This chain is ordered most-recently-declared first. For each `NamedDecl *D` in the chain, `IdentifierResolver::isDeclInScope(D, CurContext, S)` determines whether `D` belongs to a scope at or above `S`. When results are found at a given scope level, tag hiding is applied: if a non-tag decl and a tag decl of the same name exist at the same level, the non-tag wins. Further scopes are not searched if a non-tag result was already found (with some nuances for `using`-declarations, which continue searching for their overload set).

2. **DeclContext parent walk**: If the `IdentifierResolver` chain yields no result, `CppLookupName()` walks up the `DeclContext` parent chain (using `Sema::getContainingDC()`). For each namespace `DC`, it calls `DC->lookup(Name)`. Inline namespaces are searched transparently. This second phase finds declarations that were introduced into a `DeclContext` but not registered in the `IdentifierResolver` — primarily declarations from deserialized PCH/module files, for which the resolver chain was not populated.

3. **`using`-directives**: For each `using namespace N` recorded on the traversed scopes, the transitively-reachable namespaces are added to the search set. Declarations found via `using`-directives are checked for conflicts; if a direct declaration in any scope was found, the directive results are suppressed in that scope.

In C (or with `ForceNoCPlusPlus`), the algorithm is simply a linear walk of the `IdentifierResolver` chain without the DeclContext fallback.

`Sema::LookupSingleName()` is a convenience wrapper returning `nullptr` if the result is absent, ambiguous, or overloaded — appropriate when precisely one declaration is expected:

```cpp
NamedDecl *D = S.LookupSingleName(CurScope, &II, Loc,
                                   Sema::LookupOrdinaryName);
```

`Sema::LookupParsedName()` dispatches between unqualified and qualified lookup based on whether a `CXXScopeSpec` is present:

```cpp
bool Sema::LookupParsedName(LookupResult &R, Scope *S, CXXScopeSpec *SS,
                            QualType ObjectType,
                            bool AllowBuiltinCreation, bool EnteringContext);
```

When `SS` is non-empty (e.g., `N::foo`), it calls `LookupQualifiedName` on the `DeclContext` named by `SS`. Otherwise it calls `LookupName`.

### Qualified Lookup

`Sema::LookupQualifiedName(LookupResult &R, DeclContext *LookupCtx, bool InUnqualifiedLookup)` searches a specific `DeclContext` and, for class types, its base classes:

1. Call `LookupCtx->lookup(Name)` to check the direct context.
2. If `LookupCtx` is a `CXXRecordDecl` and no result was found, invoke `CXXRecordDecl::lookupInBases()` with a lookup predicate that searches each base class context.
3. Base-class lookup fills a `CXXBasePaths` object with all paths through the inheritance graph that reach the name. If multiple paths reach different declarations, the result is ambiguous.

The `CXXBasePaths` object enables detection of the ambiguity patterns:

- **`AmbiguousBaseSubobjectTypes`**: `struct A { void f(); }; struct B { void f(); }; struct C : A, B {};` — `c.f()` is ambiguous because `A::f` and `B::f` exist in distinct base class subobjects with different types. Overload resolution is not attempted.
- **`AmbiguousBaseSubobjects`**: `struct A { int x; }; struct B : A {}; struct C : A {}; struct D : B, C {};` — `d.x` finds two subobjects of type `A`, both containing `x`. Resolution requires explicit qualification.
- **`AmbiguousReference`** (via `using`-directives): same name found via two different `using namespace` directives in the same scope.

`LookupQualifiedName` also handles base-class lookup for `using` declarations inside a class, checking that each brought-in declaration satisfies access restrictions.

### The `IdentifierResolver` Shadow Chain

`IdentifierResolver` (`clang/include/clang/Sema/IdentifierResolver.h`) manages per-name chains of `NamedDecl*` entries representing all declarations currently in scope. The storage is embedded directly in `IdentifierInfo::FETokenInfo`:

- **Single declaration** (most common): `FETokenInfo` stores a `NamedDecl*` with the low bit clear.
- **Multiple declarations**: `FETokenInfo` stores an `IdDeclInfo*` with the low bit set. `IdDeclInfo` holds a `SmallVector<NamedDecl*, 2>` with the chain.

This representation ensures zero allocation overhead for the single-declaration case — which covers the vast majority of identifiers.

Insertion is via `IdentifierResolver::AddDecl(NamedDecl *D)`, which prepends `D` to the chain, placing it before older declarations. Removal via `RemoveDecl()` is O(n) but occurs only at scope pop time, which is infrequent. Iteration:

```cpp
for (auto I = IdResolver.begin(Name), E = IdResolver.end(); I != E; ++I) {
  NamedDecl *D = *I;
  if (IdResolver.isDeclInScope(D, CurContext, S)) {
    R.addDecl(D);
    // potentially break after finding one scope level's worth
  }
}
```

`isDeclInScope` checks whether `D` belongs to the passed `DeclContext` (for namespace-scope declarations) or is literally in scope `S` (for block-scope declarations where scope identity matters for C99 6.2.1 semantics).

`Sema::PushOnScopeChains(NamedDecl *D, Scope *S, bool AddToContext)` is the primary insertion interface when a new declaration is created:

```cpp
void Sema::PushOnScopeChains(NamedDecl *D, Scope *S, bool AddToContext) {
  IdResolver.AddDecl(D);  // registers in IdentifierResolver chain
  S->AddDecl(D);          // registers in Scope's DeclsInScope set
  if (AddToContext)
    CurContext->addDecl(D); // inserts into DeclContext's linked list
}
```

The `AddToContext` flag is `false` for template parameter declarations, which live in the `TemplateParamScope` but should not appear in the enclosing `DeclContext`'s own lookup table.

## Argument-Dependent Lookup

Argument-dependent lookup (ADL), specified in C++ [basic.lookup.argdep], extends the search for unqualified function names by examining namespaces associated with the types of the call arguments.

`Sema::ArgumentDependentLookup(DeclarationName Name, SourceLocation Loc, ArrayRef<Expr*> Args, ADLResult &Functions)` computes the associated namespace and class sets, then searches each:

```cpp
// Conceptual sketch from SemaLookup.cpp
void Sema::ArgumentDependentLookup(DeclarationName Name,
                                   SourceLocation Loc,
                                   ArrayRef<Expr *> Args,
                                   ADLResult &Functions) {
  AssociatedNamespaceSet AssociatedNamespaces;
  AssociatedClassSet AssociatedClasses;
  computeAssociatedDeclsForADL(Args, AssociatedNamespaces, AssociatedClasses);
  for (auto *NS : AssociatedNamespaces) {
    LookupResult R(*this, Name, Loc, LookupOrdinaryName);
    LookupQualifiedName(R, NS);
    for (auto *D : R)
      Functions.insert(D->getUnderlyingDecl());
  }
}
```

The associated namespace and class sets are computed per argument type according to C++ [basic.lookup.argdep]p2:

- **Fundamental type**: empty.
- **Class type `T`**: the innermost enclosing namespace of `T`; the namespaces and classes of all direct and indirect base classes of `T`; if `T` is a class template specialization, the namespaces of the template and all template arguments.
- **Pointer/reference to `T`**: the associated set of `T`.
- **Pointer-to-member of class `C` with member type `M`**: the associated sets of both `C` and `M`.
- **Function type**: union of the associated sets of all parameter types and the return type.
- **Enumeration type `E`**: the innermost enclosing namespace of `E`.

ADL fires only when the function call is *unqualified* (no `::` or scope specifier) and none of the following inhibition conditions hold: (1) a function with the same name is found in a scope that hides the namespace lookup (e.g., a local declaration), or (2) the call is to a named function found by normal lookup in block scope that is not a function or function template declaration.

The `Sema::UseArgumentDependentLookup()` predicate returns `false` — inhibiting ADL — when the callee has an explicit scope specifier or when the ordinary lookup result is a non-function declaration (a local variable named `swap` would prevent `std::swap` from being found by ADL).

The canonical examples:

```cpp
// ADL for std::swap: operator found in namespace std via
// the associated class std::vector<int>
std::vector<int> a, b;
swap(a, b);  // => std::swap(a, b) via ADL

// Stream insertion: operator<< is found in namespace std
// via ADL on std::ostream (type of std::cout)
std::cout << 42;  // => std::operator<<(std::cout, 42)

// Customization point pattern:
namespace mylib { struct S {}; void process(S); }
void f(mylib::S s) { process(s); }  // finds mylib::process via ADL
```

## Overload Resolution

### Building the Candidate Set

Overload resolution operates on an `OverloadCandidateSet` (`clang/include/clang/Sema/Overload.h`). The set is parameterized by a `CandidateSetKind`:

```cpp
enum CandidateSetKind {
  CSK_Normal,                   // regular function call
  CSK_Operator,                 // operator syntax
  CSK_InitByUserDefinedConversion, // copy-init from class type
  CSK_InitByConstructor,        // direct-init or list-init by constructor
  CSK_AddressOfOverloadSet,     // &overloaded_function
  CSK_CodeCompletion,           // show all candidates including deferred
};
```

The set is populated by:

- `Sema::AddFunctionCandidates(UnresolvedSetImpl &Fns, ArrayRef<Expr*> Args, OverloadCandidateSet &CS)` — iterates a pre-built function set and calls `AddOverloadCandidate()` for each `FunctionDecl`, and `AddTemplateOverloadCandidate()` for each `FunctionTemplateDecl`.
- `Sema::AddMethodCandidates()` — for member calls; sets up the implicit object parameter.
- `Sema::AddBuiltinOperatorCandidates(OverloadedOperatorKind Op, SourceLocation OpLoc, ArrayRef<Expr*> Args, OverloadCandidateSet &CS)` — synthesizes pseudo-candidates for built-in arithmetic, relational, and pointer operators. These candidates have `OverloadCandidate::Function == nullptr` and store parameter types in `BuiltinParamTypes[3]`.
- `Sema::AddArgumentDependentLookupCandidates()` — folds the ADL result set into the candidate set.

Each `OverloadCandidate` records:

```cpp
struct OverloadCandidate {
  FunctionDecl *Function;           // null for built-in pseudo-candidates
  DeclAccessPair FoundDecl;         // original lookup result (may be UsingShadowDecl)
  CXXConversionDecl *Surrogate;     // for conversion-to-function-pointer surrogates
  ConversionSequenceList Conversions; // ICS per argument
  bool Viable;                      // set false on any argument conversion failure
  bool Best;                        // set true if tied for best by BestViableFunction
  bool IsSurrogate;
  bool IgnoreObjectArgument;        // static member: skip implicit this conversion
  bool FoundByADL;
  // ...
};
```

### Implicit Conversion Sequences

For each candidate/argument pair, `Sema::TryCopyInitialization()` or `Sema::TryImplicitConversion()` computes an `ImplicitConversionSequence` (ICS). An ICS is a discriminated union over four alternatives, queryable via `getKind()`:

| `Kind` | Meaning |
|---|---|
| `StandardConversion` | Up to three chained standard conversion steps |
| `UserDefinedConversion` | Exactly one user-defined conversion wrapped by standard conversions |
| `EllipsisConversion` | Argument passed to `...` variadic parameter |
| `BadConversion` | No valid conversion exists |

A `StandardConversionSequence` records three slots:

- `First` — lvalue-to-rvalue, array-to-pointer, or function-to-pointer (or `ICK_Identity`)
- `Second` — the "value conversion": integral promotion, floating-point promotion, integral conversion, floating-point conversion, floating-integral, pointer conversion, pointer-to-member, boolean conversion, or derived-to-base
- `Third` — qualification conversion (adding `const`/`volatile`) or function pointer conversion

The `ImplicitConversionRank` enum maps conversion kinds to their preference rank:

```
ICR_Exact_Match = 0   (identity, lvalue-to-rvalue, array-to-pointer, ...)
ICR_Promotion   = 2   (integral promotion, floating promotion)
ICR_Conversion  = 4   (integral conversion, floating conversion, pointer, boolean, ...)
```

A standard conversion sequence is ranked by the worst of its three steps. Among two standard conversion sequences where the worse steps tie, a tiebreaker from [over.ics.rank] applies: e.g., `T*` → `const T*` is preferred over `T*` → `void*`.

### Best Viable Function Selection

`OverloadCandidateSet::BestViableFunction(Sema &S, SourceLocation Loc, OverloadCandidateSet::iterator &Best)` implements the partial order on viable candidates defined in [over.match.best]:

1. First, filter out non-viable candidates (Viable == false): those with a `BadConversion` ICS on any argument, wrong number of arguments after default-argument expansion, unsatisfied constraints (C++20), or CUDA target mismatch.
2. For each pair of viable candidates (F1, F2), F1 is *better* than F2 if:
   - For every argument i, ICS(F1, i) is not worse than ICS(F2, i); and
   - For at least one argument i, ICS(F1, i) is strictly better than ICS(F2, i).
   - Tiebreakers that F1 may win even with identical ICSes: F1 is a non-template where F2 is a template; F1 is more specialized than F2 by partial ordering; F1 has a non-reversed parameter order (C++20 rewritten operators); F1's constraints are more constrained.
3. If exactly one candidate beats all others, it is the best viable function.
4. If no such unique best exists, the result is `OR_Ambiguous`.

`Sema::CompareImplicitConversionSequences()` implements the ICS comparison, returning `ImplicitConversionSequence::Better`, `Indistinguishable`, or `Worse`.

### Error Reporting

When overload resolution fails with `OR_No_Viable_Function` or `OR_Ambiguous`, `OverloadCandidateSet::NoteCandidates()` emits a note for each rejected candidate:

```cpp
CandidateSet.NoteCandidates(
    PartialDiagnosticAt(CallLoc,
                        PDiag(diag::err_ovl_no_viable_function_in_call)
                            << FnName),
    SemaRef, OCD_AllCandidates, Args);
```

Each note includes the candidate's parameter types and the specific reason for rejection, drawn from `OverloadFailureKind`:

| `ovl_fail_*` | Reason |
|---|---|
| `too_many_arguments` / `too_few_arguments` | Arity mismatch |
| `bad_conversion` | ICS for at least one argument was BadConversion |
| `bad_deduction` | Template argument deduction failed |
| `explicit` | Constructor/conversion marked `explicit` in copy-init context |
| `constraints_not_satisfied` | C++20 requires-clause not satisfied |
| `enable_if` | `[[enable_if]]` attribute condition false |
| `bad_target` | CUDA host/device mismatch |

## Implicit Conversions

### Standard Conversion Sequences

`Sema::TryImplicitConversion()` checks whether a standard or user-defined implicit conversion sequence exists from `From->getType()` to `ToType`. When one is needed at a call site or assignment context, `Sema::PerformImplicitConversion()` applies it, wrapping the expression in the appropriate `ImplicitCastExpr` nodes.

The standard conversions, their `ImplicitConversionKind` enumerators, and the `CastKind` values inserted into the AST:

| Conversion | `ICK_*` | `CK_*` |
|---|---|---|
| Lvalue-to-rvalue | `ICK_Lvalue_To_Rvalue` | `CK_LValueToRValue` |
| Array-to-pointer | `ICK_Array_To_Pointer` | `CK_ArrayToPointerDecay` |
| Function-to-pointer | `ICK_Function_To_Pointer` | `CK_FunctionToPointerDecay` |
| Function pointer (noexcept strip) | `ICK_Function_Conversion` | `CK_NoOp` |
| Integral promotion | `ICK_Integral_Promotion` | `CK_IntegralCast` |
| Floating-point promotion | `ICK_Floating_Promotion` | `CK_FloatingCast` |
| Integral conversion | `ICK_Integral_Conversion` | `CK_IntegralCast` |
| Floating-point conversion | `ICK_Floating_Conversion` | `CK_FloatingCast` |
| Floating-integral | `ICK_Floating_Integral` | `CK_FloatingToIntegral` or `CK_IntegralToFloating` |
| Pointer conversion | `ICK_Pointer_Conversion` | `CK_NullToPointer`, `CK_DerivedToBase`, `CK_BitCast` |
| Pointer-to-member | `ICK_Pointer_Member` | `CK_NullToMemberPointer`, `CK_DerivedToBaseMemberPointer` |
| Boolean conversion | `ICK_Boolean_Conversion` | `CK_PointerToBoolean`, `CK_IntegralToBoolean`, etc. |
| Qualification | `ICK_Qualification` | `CK_NoOp` (const added is type-only) |
| Derived-to-base | `ICK_Derived_To_Base` | `CK_DerivedToBase` |

`PerformImplicitConversion(Expr*, QualType, ImplicitConversionSequence&, AssignmentAction)` processes the three standard conversion steps sequentially, each time calling `ImpCastExprToType()` which creates a new `ImplicitCastExpr` wrapping the previous result.

### Qualification Conversions

A qualification conversion adds `const` and/or `volatile` qualifiers to a pointer-to-T to produce a pointer-to-cv-T (or multi-level pointer where each level may gain qualifiers, subject to the "const bubble" rule: adding a qualifier at level k requires that all levels 1 through k-1 already carry `const`). Sema implements this check in `Sema::IsQualificationConversion()`.

The `CK_NoOp` cast kind is produced because no code needs to be generated — the operation is purely a type annotation. The pointer value itself is unchanged.

## Usual Arithmetic Conversions

`Sema::UsualArithmeticConversions(ExprResult &LHS, ExprResult &RHS, SourceLocation Loc, ArithConvKind ACK)` applies the "usual arithmetic conversions" of C++ [expr.arith.conv] and C [6.3.1.8] to normalize binary expression operands. The function returns the common `QualType` and modifies `LHS`/`RHS` by wrapping them in `ImplicitCastExpr` nodes.

The algorithm, in precedence order:

1. Apply `Sema::DefaultFunctionArrayLvalueConversion()` to both sides (lvalue-to-rvalue, array and function decay).
2. If either operand is a `long double`, convert the other to `long double`.
3. Else if either is `double`, convert the other to `double`.
4. Else if either is `float`, convert the other to `float`.
5. Otherwise, apply the integer promotions (converting any type narrower than `int` to `int` or `unsigned int`), then compare integer ranks.
6. If both are signed or both unsigned: the lower-rank operand is converted to the higher-rank type.
7. If the unsigned type has rank >= the signed type's rank: convert signed to unsigned.
8. If the signed type can represent all values of the unsigned type: convert unsigned to signed.
9. Otherwise: convert both to the unsigned counterpart of the signed type.

The integer rank ordering: `bool` < `char`/`signed char`/`unsigned char` < `short` < `int` < `long` < `long long` < `__int128`. `_Float128` / `__float128` participate as floating types above `long double` in certain configurations.

The `ArithConvKind` parameter controls diagnostic behavior. `ACK_Comparison` triggers the signed/unsigned comparison warning (`-Wsign-compare`). `ACK_CompAssign` suppresses conversions on the LHS since the result must be assignable back.

A subtle `__int128` edge case: `unsigned long long` has rank below `__int128`, so the common type of `unsigned long long` and `__int128` is `__int128` (the signed `__int128` can represent all `unsigned long long` values). Mixing `unsigned __int128` with `__int128` produces `unsigned __int128`.

The `Sema::DefaultFunctionArrayLvalueConversion(Expr *E)` helper applies three conversions that must precede almost every expression evaluation: function-to-pointer (a function name decays to a function pointer), array-to-pointer (an array name decays to a pointer to its first element), and lvalue-to-rvalue (an lvalue is read to produce an rvalue). These are applied inside `UsualArithmeticConversions` before the type comparisons, and also at most call sites that pass expression arguments to operators.

`Sema::UsualUnaryConversions(Expr *E)` applies only the integer promotions to a single expression — used for unary `+`, `-`, `~` and the shift-count operand — without the floating-type or signed/unsigned merging steps of `UsualArithmeticConversions`.

## User-Defined Conversions

### Conversion Constructors

A constructor of class `T` that is callable with a single argument defines a user-defined conversion from the argument type to `T`. Unless marked `explicit`, it participates in implicit conversion sequences. During overload resolution, `Sema::IsUserDefinedConversion()` (in `SemaOverload.cpp`) looks up constructors of `T` via `LookupConstructors()` and performs a nested overload resolution to find the best converting constructor.

An `explicit` conversion constructor (C++11) is excluded from implicit conversions but is considered during direct initialization and in explicit cast expressions. The `AllowedExplicit` enum controls whether explicit conversions are included in the ICS computation:

```cpp
enum class AllowedExplicit {
  None,       // suppress explicit conversions
  Conversions, // allow explicit conversion functions, not constructors
  All,         // allow all explicit conversions (C-style cast contexts)
};
```

### Conversion Functions

A member function declared `operator T()` defines a user-defined conversion from the class type to `T`. During overload resolution, `Sema::IsUserDefinedConversion()` also looks up conversion functions in the source class using `CXXRecordDecl::getVisibleConversionFunctions()`, which returns all conversion functions including those inherited via `using` declarations.

The `UserDefinedConversionSequence` structure captures the full chain:

```cpp
struct UserDefinedConversionSequence {
  StandardConversionSequence Before; // source type → constructor/function arg
  StandardConversionSequence After;  // conversion result → target parameter type
  FunctionDecl *ConversionFunction;  // the constructor or operator T()
  DeclAccessPair FoundConversionFunction; // for access checking and notes
  bool EllipsisConversion;
  bool HadMultipleCandidates;
};
```

The "Before" standard conversion sequence transforms the source type to match the constructor argument or the implicit object parameter of the conversion function. The "After" sequence handles any remaining adaptation (e.g., derived-to-base or qualification) from the conversion result to the final parameter type.

### The One User-Defined Conversion Rule

The C++ standard ([over.best.ics]p4) forbids applying two user-defined conversions implicitly: if converting from `A` to `C` requires `A -> B` via a conversion function and `B -> C` via a conversion constructor, the sequence `A -> C` is not considered an implicit conversion sequence for overload resolution. This prevents combinatorial explosion in the candidate set and is enforced by the `SuppressUserConversions` flag in `TryImplicitConversion`.

## Typo Correction

### Architecture

When name lookup returns `NotFound` and the identifier is not a keyword, Sema may attempt typo correction via:

```cpp
TypoCorrection Sema::CorrectTypo(
    const DeclarationNameInfo &Typo,
    Sema::LookupNameKind LookupKind,
    Scope *S,
    CXXScopeSpec *SS,
    CorrectionCandidateCallback &CCC,
    CorrectTypoKind Mode,           // NonError or ErrorRecovery
    DeclContext *MemberContext,
    bool EnteringContext,
    const ObjCObjectPointerType *OPT,
    bool RecordFailure);
```

`CorrectTypo` first consults `TypoCorrectionFailures` — a `DenseMap<IdentifierInfo*, SmallSet<SourceLocation>>` — to avoid redundant correction attempts at the same location. It then delegates to `makeTypoCorrectionConsumer()`, which constructs a `TypoCorrectionConsumer` and populates it by calling `LookupVisibleDecls()`.

### Candidate Collection and Scoring

`TypoCorrectionConsumer` collects candidate declarations from: the `IdentifierResolver` chain, `DeclContext::decls()` iteration for the relevant scope, and tag declarations reachable through base class lookup. For each candidate, the edit distance is computed as a weighted sum:

```cpp
// From TypoCorrection.h
static const unsigned CharDistanceWeight     = 100U;
static const unsigned QualifierDistanceWeight = 110U;
static const unsigned CallbackDistanceWeight  = 150U;
```

- `CharDistance`: standard Levenshtein edit distance on the identifier string (insertions, deletions, substitutions), scaled by `CharDistanceWeight`.
- `QualifierDistance`: difference in nested-name-specifier depth, scaled by `QualifierDistanceWeight`.
- Callback penalty: the `CorrectionCandidateCallback` (supplied by the caller) can reject candidates or add additional cost, scaled by `CallbackDistanceWeight`.

The total is mapped to `InvalidDistance = UINT_MAX` if it exceeds `MaximumDistance = 10000`. Only candidates within a bounded multiple of the best distance seen are retained. The `TypoCorrection` result:

```cpp
class TypoCorrection {
  DeclarationName CorrectionName;
  NestedNameSpecifier CorrectionNameSpec; // may be empty
  SmallVector<NamedDecl *, 1> CorrectionDecls;
  unsigned CharDistance;
  unsigned QualifierDistance;
  // ...
public:
  NamedDecl *getCorrectionDecl() const;
  template <class DeclClass>
  DeclClass *getCorrectionDeclAs() const { return dyn_cast_or_null<DeclClass>(...); }
  SourceRange getCorrectionRange() const;
  unsigned getEditDistance(bool Normalized = true) const;
};
```

When a valid correction is found, Sema emits a `note_did_you_mean` diagnostic with the corrected name and, for high-confidence corrections, creates a `TypoExpr` in the AST that is later resolved to the corrected `DeclRefExpr`. `CorrectTypoDelayed()` returns a `TypoExprState` for cases where the correction must wait until a full expression context is established (inside dependent template contexts, for example).

Recursive correction is prevented by the failure cache: if typo correction for `foo` resolves to `bar`, and `bar` is subsequently not found, `bar` is recorded in `TypoCorrectionFailures` and is not itself corrected.

## Name Hiding and Redeclaration

### Inner-Scope Hiding

When a declaration `D` is added to an inner scope and shares a name with a declaration `D'` in an outer scope, `D` hides `D'` from that point on within the inner scope. Hiding is implemented automatically by the `IdentifierResolver` ordering: `D` is prepended to the chain, so the linear walk finds it before `D'`. No explicit hide operation is required.

The one nuanced case is tag-name hiding in C++. If a typedef `struct Foo {}; typedef ... Foo;` exists in the same scope, the non-tag hides the tag. `LookupResult` respects the `HideTags` flag (default `true`): when a non-tag declaration is found in a scope, any tag with the same name in the same scope is removed from the result.

### Redeclaration Merging

When a new declaration `New` is introduced and lookup finds a prior `Old` with the same name in the same scope, `Sema` must decide whether to merge them or emit a redeclaration error. The merging functions:

**`Sema::MergeVarDecl(VarDecl *New, LookupResult &Previous)`** handles:
- `extern` redeclarations: checks type compatibility, merges visibility.
- C tentative definitions: multiple definitions of the same external variable are allowed in C.
- C++ variable redeclarations: diagnosed as errors unless they are out-of-line definitions of a previously-declared variable.

**`Sema::MergeFunctionDecl(FunctionDecl *New, NamedDecl *&Old, Scope *S, bool MergeTypeWithOld)`** handles:
- Multiple prototypes: checks that parameter types are compatible.
- Prototype vs. definition: the definition merges the declaration.
- `inline` merging: a function declared `inline` in some declarations and not in others merges to `inline` if any declaration was inline.
- `noexcept` merging: mismatched exception specifications are diagnosed.
- `[[nodiscard]]` / `[[deprecated]]` attribute propagation from old to new.

Returns `true` on hard type incompatibility.

**`Sema::MergeTypedefNameDecl(Scope *S, TypedefNameDecl *New, LookupResult &OldDecls)`**: typedef redeclarations are allowed in C (if the underlying types are identical) but trigger an error in C++ (with the exception of C++11 where a typedef that exactly matches a previous typedef is permitted as an extension in some implementations).

The `extern "C"` merging path in `MergeFunctionDecl` uses `Sema::IsOverload()` to distinguish overloading from redeclaration, with special handling for the case where the new declaration lacks a language linkage specification but the old one had `extern "C"`.

Clang 22 adds `Sema::CheckRedeclarationModuleOwnership()` and `Sema::CheckRedeclarationExported()` for C++20 modules: if a declaration in a module interface unit is re-exported from multiple partitions, exactly one partition must supply the definition.

### `CheckRedeclaration` Infrastructure

The `Sema::forRedeclarationInCurContext()` helper returns the appropriate `RedeclarationKind` enum value (`NotForRedeclaration`, `ForVisibleRedeclaration`, or `ForExternalRedeclaration`) based on whether the current context is module-private, exported, or an `extern "C"` linkage group. This value is passed to `LookupResult` constructors when looking for prior declarations during `ActOnVariableDeclarator()` and `ActOnFunctionDeclarator()`.

After merging, `Sema::mergeDeclAttributes()` copies attributes from `Old` to `New` using `Sema::mergeAlignedAttr()`, `Sema::mergeVisibilityAttr()`, and similar per-attribute helpers. This ensures that a definition without an explicit `[[nodiscard]]` inherits the attribute from the prior forward declaration.

## `using` Declarations and `using`-Directives

### `UsingDecl` and Shadow Decls

A `using` declaration (`using N::foo;`) creates a `UsingDecl` node in the current `DeclContext`. For each declaration named `foo` found in namespace `N`, Sema creates a `UsingShadowDecl` that wraps the target and is itself inserted into the current scope and context:

```
UsingDecl  [using N::foo; at current scope]
  UsingShadowDecl ──> N::foo (FunctionDecl, overload 1)
  UsingShadowDecl ──> N::foo (FunctionDecl, overload 2)
  UsingShadowDecl ──> N::foo (FunctionTemplateDecl)
```

A `UsingShadowDecl` derives from `NamedDecl` and returns the shadow's own name (same as the target's) from `getName()`. `UsingShadowDecl::getTargetDecl()` retrieves the actual brought-in declaration. During overload resolution, when a `UsingShadowDecl` is found, `NamedDecl::getUnderlyingDecl()` unwraps it to the target, but `OverloadCandidate::FoundDecl` preserves the shadow for access-control purposes.

`Sema::ActOnUsingDeclaration()` calls `Sema::BuildUsingDeclaration()`, which:
1. Resolves the nested-name-specifier to a `DeclContext`.
2. Performs `LookupQualifiedName` in that context.
3. Checks for ill-formed uses (e.g., `using ::` without a name, using a constructor name in a non-derived class).
4. Calls `Sema::BuildUsingShadowDecl()` for each found declaration.
5. Calls `Sema::CheckUsingShadowDecl()` to verify that each shadow does not conflict with existing declarations in the current scope.

For member `using` declarations that bring in base-class constructors (`using Base::Base;`), `ConstructorUsingShadowDecl` (a subclass of `UsingShadowDecl`) is created. The inheriting constructor machinery then synthesizes actual `CXXConstructorDecl` nodes as needed during template instantiation.

### `UsingDirectiveDecl`

A `using`-directive (`using namespace N;`) creates a `UsingDirectiveDecl` and records it on the enclosing `Scope` via `Scope::PushUsingDirective()`. Unlike a `using` declaration, it does *not* introduce shadow decls. Instead, the DeclContext-walking phase of `CppLookupName` reads the `UsingDirectives` vector from each scope in the walk and adds the named namespace to the transitive search set.

The semantic distinction is fundamental to C++ name lookup:
- `using N::foo;` makes `foo` a member of the current scope — it participates in overload resolution just like a locally-declared function.
- `using namespace N;` adds `N` to the set of namespaces searched during unqualified lookup for the enclosing namespace scope — it does not introduce names directly, and can therefore be outcompeted by ordinary declarations.

### Namespace Aliases and `using enum`

`Sema::ActOnNamespaceAliasDef(Scope *S, SourceLocation NamespaceLoc, SourceLocation AliasLoc, IdentifierInfo *Alias, CXXScopeSpec &SS, SourceLocation IdentLoc, IdentifierInfo *Ident)` creates a `NamespaceAliasDecl` mapping the alias to the target `NamespaceDecl*`. Subsequent qualified lookups of `Alias::x` resolve `Alias` through the alias chain to the canonical namespace.

C++20 `using enum E;` introduces a `UsingEnumDecl` (derived from `BaseUsingDecl`, a common base of `UsingDecl` and `UsingEnumDecl`). Sema creates one `UsingShadowDecl` per enumerator of `E`, inserting them into the current scope. This enables unqualified access to enum values within a block scope — the primary motivation is eliminating repeated `MyEnum::` qualifications inside a switch statement:

```cpp
switch (state) {
  using enum TrafficLight;
  case Red:    stop();  break;
  case Yellow: slow();  break;
  case Green:  go();    break;
}
```

The Sema entry point is `Sema::ActOnUsingEnumDeclaration(Scope *S, AccessSpecifier AS, SourceLocation UsingLoc, SourceLocation EnumLoc, SourceLocation IdentLoc, ParsedType T, CXXScopeSpec *SS)`.

## Putting It Together: A Name Lookup Trace

To make these subsystems concrete, consider resolving `swap(a, b)` in a function body where `a` and `b` are `std::vector<int>`:

1. The Parser calls `Sema::ActOnCallExpr()` with callee expression `swap` (an unresolved identifier) and two argument expressions.
2. `Sema::ClassifyName()` / `Sema::LookupName()` is called with `LookupOrdinaryName`. The `IdentifierResolver` chain for `swap` is empty (no local declaration). The DeclContext walk finds nothing in the function scope, enclosing namespace, or global scope.
3. ADL fires because ordinary lookup found nothing and the call is unqualified. The argument types `std::vector<int>` and `std::vector<int>` have associated namespace `std`. `Sema::ArgumentDependentLookup()` calls `LookupQualifiedName` in `std`, finding `std::swap` (a function template). The result is an `UnresolvedLookupExpr` noting that ADL was used.
4. Overload resolution: `AddArgumentDependentLookupCandidates()` adds the `FunctionTemplateDecl` for `std::swap`. `AddTemplateOverloadCandidate()` deduces `T = int`, yielding a viable candidate `std::swap<int>` with exact-match standard-conversion ICS on both `std::vector<int> &` parameters.
5. `BestViableFunction()` selects `std::swap<int>`.
6. `ActOnCallExpr()` builds a `CallExpr` with the resolved `FunctionDecl` as callee.

This trace shows all three major subsystems — unqualified lookup, ADL, and overload resolution — collaborating on a single three-token expression.

---

## Chapter Summary

- `Sema` is the Parser's "actions" object: instantiated once per TU, split across ~20 source files, with `CurContext` tracking the active `DeclContext` and `TUScope` anchoring the root scope. Clang 22 extracts target-specific functionality into sub-objects (`SemaARM`, `SemaX86`, etc.) without changing calling conventions.
- `clang::Scope` is a transient parse-time structure carrying `ScopeFlags` bits; it is distinct from (but associated with) the persistent `DeclContext`. Multiple shortcut parent pointers (`FnParent`, `BreakParent`, `TemplateParamParent`) accelerate common traversals.
- `DeclContext` provides `lookup()` for O(1) name-based searching and `decls()` for order-preserving iteration. `IdentifierResolver` provides lexical-scope lookup via shadow chains stored in `IdentifierInfo::FETokenInfo`.
- `LookupResult` encapsulates a name lookup query and result set; `LookupNameKind` selects identifier namespace bits. Unqualified lookup walks `IdentifierResolver`, then DeclContext parents, then `using`-directive transitive closures. Qualified lookup searches a specific context and its base classes.
- ADL extends function lookup to associated namespaces of argument types; it is inhibited by local declarations, qualified calls, and the `UseArgumentDependentLookup()` predicate.
- Overload resolution populates an `OverloadCandidateSet`, computes an `ImplicitConversionSequence` per candidate/argument pair, and selects the best viable function through the partial order on ICS rankings. Each `OverloadCandidate` stores `FoundDecl` (may be a `UsingShadowDecl`) alongside the resolved `FunctionDecl`.
- `StandardConversionSequence` records up to three conversion steps (First/Second/Third); `UserDefinedConversionSequence` wraps one user-defined conversion between two standard steps; at most one user-defined conversion is allowed per implicit conversion sequence.
- `UsualArithmeticConversions` normalizes binary expression operands to a common type following floating/integer rank rules; `__int128` and `unsigned __int128` extend the rank hierarchy above `long long`.
- `CorrectTypo` uses weighted Levenshtein edit distance over the visible declaration space to generate "did you mean X?" suggestions; the failure cache prevents recursive correction.
- `UsingDecl` + `UsingShadowDecl` introduces specific names into the current scope and participates in overload resolution; `UsingDirectiveDecl` adds a namespace to the transitive lookup set for the enclosing namespace scope. C++20 `using enum` follows the shadow-decl model.

---

## Cross-References

- [Chapter 32 — The Parser](ch32-the-parser.md): The Parser drives Sema via `Actions` callbacks; every `ParseXxx` function calls a corresponding `Sema::ActOnXxx` method that invokes the lookup and conversion machinery described here.
- [Chapter 34 — Sema II: Templates](ch34-sema-templates.md): Template argument deduction, `TemplateDeductionResult`, template instantiation, and dependent name resolution build on the name lookup infrastructure described in this chapter.
- [Chapter 36 — The Clang AST in Depth](ch36-the-clang-ast-in-depth.md): `NamedDecl`, `ValueDecl`, `FunctionDecl`, and their relationships are the declarations that `LookupResult` returns and `OverloadCandidateSet` operates on.
- [Chapter 08 — Semantic Analysis Theory](../part-02-compiler-theory/ch08-semantic-analysis.md): Type environments, symbol tables, and the formal treatment of name scoping that this chapter implements in Clang's `Sema`.

## Reference Links

- [`clang/include/clang/Sema/Sema.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/Sema.h)
- [`clang/include/clang/Sema/Scope.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/Scope.h)
- [`clang/include/clang/Sema/Lookup.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/Lookup.h)
- [`clang/include/clang/Sema/Overload.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/Overload.h)
- [`clang/include/clang/Sema/IdentifierResolver.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/IdentifierResolver.h)
- [`clang/include/clang/Sema/TypoCorrection.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/TypoCorrection.h)
- [`clang/lib/Sema/SemaLookup.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaLookup.cpp)
- [`clang/lib/Sema/SemaOverload.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaOverload.cpp)
- [`clang/lib/Sema/SemaExpr.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaExpr.cpp)
- [`clang/lib/Sema/SemaDecl.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaDecl.cpp)
- [`clang/lib/Sema/SemaDeclCXX.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaDeclCXX.cpp)
- [ISO C++ Standard, \[basic.lookup\]](https://eel.is/c++draft/basic.lookup)
- [ISO C++ Standard, \[over.match\]](https://eel.is/c++draft/over.match)
- [ISO C++ Standard, \[conv\]](https://eel.is/c++draft/conv)


---

@copyright jreuben11
