# Chapter 32 — The Parser

*Part V: The Clang Frontend*

Clang's parser is the engine that transforms a token stream—delivered by the preprocessor—into a structured Abstract Syntax Tree, mediated at every node-creation point by the semantic analysis layer. Unlike most production parsers that separate the grammar from its actions, Clang's `Parser` is inseparably joined to `Sema`: every syntactic production that creates or modifies a declaration, type, or expression immediately calls a `Sema::Act*` method, allowing semantic constraints to feed back into parsing decisions. The result is a hand-written recursive-descent parser of roughly 40,000 lines spread across sixteen source files, capable of handling every dialect of C, C++23, Objective-C, OpenMP, OpenACC, HLSL, and more—while providing industrial-strength error recovery and incremental re-parsing for interactive tools. This chapter traces the complete lifecycle from `ParseAST()` entry through token management, disambiguation, declaration and expression parsing, statement dispatch, attribute processing, error recovery, and finally the incremental-parsing extension used by `clang-repl`.

---

## 32.1 Parser Architecture

### 32.1.1 The `clang::Parser` Class

`Parser` is declared in [`clang/include/clang/Parse/Parser.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Parse/Parser.h) and inherits from `CodeCompletionHandler` to participate in IDE completion callbacks. The class header enumerates sixteen implementation files in its table of contents:

```
// 1. Parsing        — Parser.cpp
// 2. C++ Inline Methods — ParseCXXInlineMethods.cpp
// 3. Declarations   — ParseDecl.cpp
// 4. C++ Declarations — ParseDeclCXX.cpp
// 5. Expressions    — ParseExpr.cpp
// 6. C++ Expressions — ParseExprCXX.cpp
// 7. HLSL           — ParseHLSL.cpp
// 8. Initializers   — ParseInit.cpp
// 9. Objective-C    — ParseObjc.cpp
// 10. OpenACC       — ParseOpenACC.cpp
// 11. OpenMP        — ParseOpenMP.cpp
// 12. Pragmas       — ParsePragma.cpp
// 13. Statements    — ParseStmt.cpp
// 14. Inline asm    — ParseStmtAsm.cpp
// 15. C++ Templates — ParseTemplate.cpp
// 16. Tentative Parsing — ParseTentative.cpp
```

The constructor signature is:

```cpp
Parser(Preprocessor &PP, Sema &Actions, bool SkipFunctionBodies);
```

Three private members anchor every parsing operation: `PP` (the `Preprocessor&` that delivers tokens), `Actions` (the `Sema&` through which every AST node is created), and `Tok` (the one-token lookahead buffer).

### 32.1.2 The `Actions` Member and the Sema Contract

Every non-trivial parsing method calls one or more `Sema::Act*` methods to create AST nodes and perform semantic validation. This design separates *syntactic recognition* (the parser's job) from *semantic construction* (Sema's job) while keeping them in lock-step:

```cpp
// From ParseDecl.cpp — after parsing a full declaration
Decl *D = Actions.ActOnDeclarator(getCurScope(), DeclaratorInfo);
```

The `Actions` member is typed as `Sema&`, so there is no virtual dispatch and no separate "action class" layer. Earlier LLVM design notes show the parameter was originally `ASTContext &Ctx` but was unified through `Sema`. The `getActions()` accessor exposes it to friend classes.

### 32.1.3 `ParseAST()` — The Entry Point

The public API in [`clang/include/clang/Parse/ParseAST.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Parse/ParseAST.h) provides two overloads:

```cpp
void ParseAST(Preprocessor &PP, ASTConsumer *C, ASTContext &Ctx,
              bool PrintStats = false,
              TranslationUnitKind TUKind = TU_Complete,
              CodeCompleteConsumer *CompletionConsumer = nullptr,
              bool SkipFunctionBodies = false);

void ParseAST(Sema &S, bool PrintStats = false,
              bool SkipFunctionBodies = false);
```

The first overload creates a `Sema` internally; the second accepts an externally constructed `Sema`. Internally `ParseAST` constructs a `Parser`, calls `Parser::Initialize()` (which primes the lookahead by calling `PP.Lex(Tok)` once), then drives the top-level loop:

```cpp
// Simplified from clang/lib/Parse/ParseAST.cpp
Parser P(PP, *S, SkipFunctionBodies);
P.Initialize();
Sema::ModuleImportState ImportState;
for (bool AtEOF = P.ParseFirstTopLevelDecl(ADecl, ImportState);
     !AtEOF; AtEOF = P.ParseTopLevelDecl(ADecl, ImportState)) {
  if (ADecl && !Consumer->HandleTopLevelDecl(ADecl.get()))
    return;
}
Consumer->HandleTranslationUnit(S->getASTContext());
S->ActOnEndOfTranslationUnit();
```

`FrontendAction::Execute()` reaches `ParseAST` through `ASTFrontendAction::ExecuteAction()`, which calls `ParseAST(*CI.getSema(), ...)` after setting up the `Sema` instance.

### 32.1.4 `ParseTopLevelDecl()` and the Translation Unit Loop

```cpp
bool Parser::ParseTopLevelDecl(DeclGroupPtrTy &Result,
                                Sema::ModuleImportState &ImportState);
```

This returns `true` when `tok::eof` is reached. Each iteration recognises one top-level production: a declaration, a function definition, a C++20 module declaration, or a module-import declaration. For C++20 modules the `ImportState` tracks whether the parser has seen the global module fragment preamble, the module declaration, and optionally the private module fragment.

### 32.1.5 `ParseScope` — RAII Scope Management

Every construct that introduces a new scope (function bodies, class bodies, namespaces, `if`/`for`/`while` blocks, template parameter lists) uses `ParseScope`:

```cpp
class ParseScope {
  Parser *Self;
public:
  ParseScope(Parser *Self, unsigned ScopeFlags,
             bool EnteredScope = true,
             bool BeforeCompoundStmt = false);
  void Exit();
  ~ParseScope() { Exit(); }
};
```

On construction `EnterScope(ScopeFlags)` is called; on destruction `ExitScope()` is called. The `ScopeFlags` bitmask—drawn from `Scope::ScopeFlags`—marks the scope as a function scope, a class scope, a template parameter scope, a control-flow scope, and so on. `MultiParseScope` manages a dynamically determined number of nested scopes, used when re-entering template parameter scopes during out-of-line template member parsing.

---

## 32.2 Token Management

### 32.2.1 The Lookahead Token

`Parser::Tok` is a single `Token` object representing the next unprocessed token. All parsing predicates examine it; all consumption methods advance it by calling `PP.Lex(Tok)`. There is no internal token queue: the preprocessor is the source of truth.

The `GetLookAheadToken(N)` method provides additional lookahead without consuming:

```cpp
const Token &GetLookAheadToken(unsigned N) {
  if (N == 0 || Tok.is(tok::eof)) return Tok;
  return PP.LookAhead(N - 1);   // PP buffers N-1 extra tokens
}
const Token &NextToken() { return PP.LookAhead(0); }
```

The preprocessor's `LookAhead(N)` triggers lexing of additional tokens into an internal cache, so arbitrary lookahead is possible but each additional token incurs a lex call.

### 32.2.2 Consumption Methods

The parser maintains three nesting counters—`ParenCount`, `BracketCount`, `BraceCount`—to enable balanced-delimiter tracking during error recovery. Dedicated consume methods maintain these counts and fire assertions if misused:

```cpp
// Normal tokens — must not be special
SourceLocation ConsumeToken();   // asserts !isTokenSpecial()

// Type-specialised consumers
SourceLocation ConsumeParen();   // increments / decrements ParenCount
SourceLocation ConsumeBracket(); // increments / decrements BracketCount
SourceLocation ConsumeBrace();   // increments / decrements BraceCount
SourceLocation ConsumeStringToken();

// Dispatch variant for unknown token type
SourceLocation ConsumeAnyToken(bool ConsumeCodeCompletionTok = false);
```

`ConsumeAnyToken` is used only in error recovery paths where the token kind is genuinely unknown.

### 32.2.3 Conditional Consumption

```cpp
bool TryConsumeToken(tok::TokenKind Expected);
bool TryConsumeToken(tok::TokenKind Expected, SourceLocation &Loc);
```

Both return `false` without consuming if the current token is not `Expected`. `TryConsumeToken` is the preferred idiom for optional tokens such as trailing `>` in template argument lists.

### 32.2.4 `ExpectAndConsume` and `ExpectAndConsumeSemi`

```cpp
bool ExpectAndConsume(tok::TokenKind ExpectedTok,
                      unsigned Diag = diag::err_expected,
                      StringRef DiagMsg = "");

bool ExpectAndConsumeSemi(unsigned DiagID, StringRef TokenUsed = "");
```

`ExpectAndConsume` emits a diagnostic and returns `true` if the current token is not `ExpectedTok`; it also handles FixIt-recoverable misspellings (e.g., `=` for `==`). `ExpectAndConsumeSemi` also consumes stray closing delimiters that immediately precede the semicolon—a common programmer error.

### 32.2.5 Annotation Tokens

The preprocessor converts identifier tokens that resolve to types, scope specifiers, or template-ids into annotation tokens (`tok::annot_typename`, `tok::annot_cxxscope`, `tok::annot_template_id`). These are single tokens that carry structured payloads via `Token::getAnnotationValue()`. `TryAnnotateTypeOrScopeToken()` is the primary method that replaces an identifier-plus-optional-scope-spec sequence with an annotation:

```cpp
bool TryAnnotateTypeOrScopeToken(
    ImplicitTypenameContext AllowImplicitTypename = ImplicitTypenameContext::No);
```

Type annotations carry a `ParsedType` opaque pointer; scope annotations carry a `CXXScopeSpec`. Annotation conversion eliminates the need to re-look-up names during subsequent parsing passes.

---

## 32.3 Tentative Parsing and Disambiguation

### 32.3.1 The C++ Ambiguity Problem

C++ inherits from C the structural ambiguity between declarations and expressions. A token sequence such as `T * p` can be either a multiplication expression (`T` times `p`) or a pointer-to-`T` declarator (`T *p`). More complex is `A(B)`: a function call with argument `B`, or a variable `A` whose type is function `B()` (the "most vexing parse"). The grammar cannot be made unambiguous without semantic type information; `isCXXDeclarationSpecifier` queries Sema to determine whether `T` is a type name.

### 32.3.2 `TentativeParsingAction`

The parser implements speculative look-ahead via a token backtracking mechanism:

```cpp
class TentativeParsingAction {
  Parser &P;
  Token PrevTok;
  size_t PrevTentativelyDeclaredIdentifierCount;
  unsigned short PrevParenCount, PrevBracketCount, PrevBraceCount;
  bool isActive;
public:
  explicit TentativeParsingAction(Parser &p, bool Unannotated = false);
  void Commit();   // discard the saved state — keep consumed tokens
  void Revert();   // restore Tok, counts, and PP backtrack position
  ~TentativeParsingAction();  // asserts Commit/Revert was called
};
```

On construction `PP.EnableBacktrackAtThisPos(Unannotated)` saves the preprocessor's token position. `Revert()` calls `PP.Backtrack()` and restores `Parser::Tok` plus all delimiter counts. `RevertingTentativeParsingAction` automatically calls `Revert()` in its destructor, used for probes that always revert.

When `Unannotated` is `true`, any annotation tokens created during the tentative parse are also reverted (they cannot survive a backtrack if the parser decided the branch was wrong).

### 32.3.3 The `TPResult` Enum and Disambiguation Functions

The tentative parsing subsystem (implemented in `ParseTentative.cpp`) uses a four-valued return type:

```cpp
enum class TPResult { True, False, Ambiguous, Error };
```

The primary discriminator is:

```cpp
TPResult isCXXDeclarationSpecifier(
    ImplicitTypenameContext AllowImplicitTypename,
    TPResult BracedCastResult = TPResult::False,
    bool *InvalidAsDeclSpec = nullptr);
```

`TPResult::Ambiguous` is returned for cases like `(T)` where `T` is a type-name: it could be a cast expression or a function-style declaration. The caller then applies the "most vexing parse" rule—prefer declaration when ambiguous.

Supporting functions include:

- `isCXXSimpleDeclaration(bool AllowForRangeDecl)` — returns `true` if the lookahead unambiguously matches a declaration
- `isCXXTypeId(TentativeCXXTypeIdContext, bool &isAmbiguous)` — distinguishes `type-id` from `expression` in `sizeof(...)`, `typeid(...)`, and cast contexts
- `isDeclarationStatement(bool DisambiguatingWithExpression)` — the outer entry point for statement-level disambiguation

### 32.3.4 Resolution Rules

For `T * p` at statement level: if `T` is a known type name, `isCXXDeclarationSpecifier` returns `TPResult::True` and the statement is parsed as a pointer declaration. Otherwise it is parsed as a multiplicative expression. For `A(B)` at function scope: Clang follows the C++ standard's "most vexing parse" rule—if the parse as a declaration is syntactically valid, it is preferred. A variable `std::mutex m()` is parsed as a function declaration; `std::mutex m{}` uses brace-init to force variable initialization.

---

## 32.4 Annotation Tokens and Name Resolution

### 32.4.1 The `TryAnnotateName` Family

Before parsing ambiguous token sequences, the parser may convert identifiers into structured annotation tokens. Three primary annotation-conversion methods exist:

```cpp
AnnotatedNameKind TryAnnotateName(
    CorrectionCandidateCallback *CCC = nullptr,
    ImplicitTypenameContext AllowImplicitTypename =
        ImplicitTypenameContext::No);

bool TryAnnotateTypeOrScopeToken(
    ImplicitTypenameContext AllowImplicitTypename = ImplicitTypenameContext::No);

bool TryAnnotateCXXScopeToken(bool EnteringContext = false);
```

`TryAnnotateName` queries Sema for the meaning of the current identifier in the current scope. It returns one of the `AnnotatedNameKind` enumerators:

```cpp
enum class AnnotatedNameKind {
  Error,          // annotation failed with diagnostic
  TentativeDecl,  // identifier is a tentatively declared name
  TemplateName,   // identifier names a template
  Unresolved,     // cannot be resolved (dependent context)
  Success         // annotation token pushed onto stream
};
```

On `Success`, the identifier token (and any following `::` and identifier chain) has been replaced in the token stream by a `tok::annot_typename` token whose annotation value is a `ParsedType`, or by a `tok::annot_cxxscope` token whose annotation value is a `CXXScopeSpec*`.

`TryAnnotateTypeOrScopeToken` is the combined entry: if the current token begins a type name or nested-name-specifier, it produces an annotation. In dependent contexts (inside template bodies), the annotation may still represent an unresolved name that Sema must handle during instantiation.

### 32.4.2 Annotation Token Kinds

| Annotation kind | `tok::` value | Payload |
|-----------------|---------------|---------|
| Type name | `annot_typename` | `ParsedType` opaque ptr |
| Scope spec | `annot_cxxscope` | `CXXScopeSpec*` |
| Template-id | `annot_template_id` | `TemplateIdAnnotation*` |
| Repl EOF | `annot_repl_input_end` | (none) |
| Module begin | `annot_module_begin` | `Module*` |
| Module end | `annot_module_end` | `Module*` |

The parser reads annotation payloads through `Token::getAnnotationValue()` (returns `void*`) and casts it to the appropriate type. The `ConsumeAnnotationToken()` method is provided for consuming these special tokens without triggering the "use ConsumeToken for non-special" assertion.

---

## 32.5 Declarations and Declarators

### 32.5.1 `DeclSpec` — Accumulating Type Specifiers

`DeclSpec` (declared in [`clang/include/clang/Sema/DeclSpec.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/DeclSpec.h)) accumulates all specifiers on a declaration before the declarator:

```cpp
class DeclSpec {
public:
  enum SCS {          // storage-class specifier
    SCS_unspecified, SCS_typedef, SCS_extern, SCS_static,
    SCS_auto, SCS_register, SCS_private_extern, SCS_mutable
  };
  // TypeSpecType (TST) enumerates: void, char, int, float, double,
  // bool, struct, class, enum, typename, decltype, auto, ...
};
```

`ParseDeclarationSpecifiers(DeclSpec&, ...)` is a large loop that consumes type-specifier tokens and calls `DeclSpec::SetTypeSpecType()`, `SetStorageClassSpec()`, `SetTypeQual()`, etc. The loop terminates when the next token cannot be part of a decl-specifier:

```cpp
void Parser::ParseDeclarationSpecifiers(
    DeclSpec &DS, ParsedTemplateInfo &TemplateInfo,
    AccessSpecifier AS, DeclSpecContext DSC,
    ParsedAttributes &Attrs,
    LateParsedAttrList *LateAttrs = nullptr);
```

On encountering `struct`/`union`/`class` keywords, it calls `ParseClassSpecifier()`; on `enum`, it calls `ParseEnumSpecifier()`; on identifiers, it checks Sema to determine whether the identifier is a typedef name, class name, or enum name.

### 32.5.2 `Declarator` and `DeclaratorChunk`

`Declarator` (defined in `DeclSpec.h`) holds a reference to the already-parsed `DeclSpec` and accumulates a sequence of `DeclaratorChunk` entries representing the modifier layers of the type:

```cpp
struct DeclaratorChunk {
  enum {
    Pointer, Reference, Array, Function,
    BlockPointer, MemberPointer, Paren, Pipe
  } Kind;
  // Kind-specific sub-structs: PointerTypeInfo, ReferenceTypeInfo,
  // ArrayTypeInfo, FunctionTypeInfo, MemberPointerTypeInfo, ...
};
```

`ParseDeclarator(Declarator&)` builds the chunk list outward from the identifier:

```cpp
void Parser::ParseDeclarator(Declarator &D);
void Parser::ParseDeclaratorInternal(Declarator &D,
                                     DirectDeclParseFunction DirectDeclParser);
```

For `int (*fp)(double)`, parsing produces: `DeclSpec{int}`, then chunks in order: `Function(double)`, `Pointer`, yielding the type "pointer to function taking double returning int."

A `DecompositionDeclarator` (the C++17 structured binding `auto [a, b, c] = expr`) is stored as an alternative to the `Name` field within `Declarator`.

### 32.5.3 Function Declarator Parsing

`ParseFunctionDeclarator` handles the full complexity of C++ function signatures:

```cpp
void ParseFunctionDeclarator(Declarator &D,
                              ParsedAttributes &FirstArgAttrs,
                              BalancedDelimiterTracker &Tracker,
                              bool IsAmbiguous,
                              bool RequiresArg = false);
```

After the `(` of a function parameter list, it calls `ParseParameterDeclarationClause` (typed parameters) or `ParseFunctionDeclaratorIdentifierList` (K&R-style identifier list). For C++, it then processes:

1. **cv-qualifier-seq** — `const`, `volatile`, `restrict` on member functions
2. **ref-qualifier** — `&` or `&&` rvalue ref qualification
3. **exception-specification** — `noexcept` or `throw(...)`
4. **attribute-specifier-seq** — trailing attributes on the function type
5. **trailing-return-type** — `-> T` parsed by `ParseTrailingReturnType()`
6. **requires-clause** — C++20 `requires Constraint` on the function declarator

The `IsAmbiguous` flag is set when the function-declarator appears in a tentative-parse context; it suppresses certain diagnostics until the ambiguity is resolved.

### 32.5.4 `ParseDeclGroup()` — Init-Declarator Lists

After parsing the `DeclSpec`, the parser parses a comma-separated list of declarators and optional initialisers:

```cpp
// from ParseDecl.cpp
DeclGroupPtrTy Parser::ParseDeclGroup(ParsingDeclSpec &DS,
                                       DeclaratorContext Context,
                                       ParsedAttributes &DeclAttrs,
                                       ParsedAttributes &DeclSpecAttrs,
                                       SourceLocation *DeclEnd,
                                       ForRangeInit *FRI);
```

For each declarator, `Sema::ActOnDeclarator()` or `Sema::ActOnVariableDeclarator()` is called to produce a `Decl*`, which is returned as part of a `DeclGroupRef`. Function bodies are parsed immediately after the declarator when a `{` is seen.

`ParseFunctionDefinition` handles the case where the declarator is followed by a function body:

```cpp
Decl *ParseFunctionDefinition(ParsingDeclarator &D,
                               const ParsedTemplateInfo &TemplateInfo,
                               LateParsedAttrList *LateParsedAttrs);
```

For non-try function bodies it calls `ParseFunctionStatementBody(Decl*, ParseScope&)`. For function-try-block forms (`int f() try { } catch (...)`) it calls `ParseFunctionTryBlock`.

### 32.5.5 `ParsingDeclSpec` and `ParsingDeclarator` RAII Wrappers

`ParsingDeclSpec` extends `DeclSpec` with a `ParsingDeclRAIIObject` to track access-control diagnostics that may need to be delayed until the full declarator is known. Similarly, `ParsingDeclarator` wraps `Declarator` with delayed diagnostic infrastructure. These RAII objects interact with `Sema::PushParsingDeclaration()` and `Sema::PopParsingDeclaration()` to suppress premature access-control error emission during template-parameter-scope declarations.

---

## 32.6 Expression Parsing

### 32.6.1 Precedence Climbing

Clang parses expressions using a top-down operator-precedence (Pratt) algorithm. The `prec::Level` enum in [`clang/include/clang/Basic/OperatorPrecedence.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/OperatorPrecedence.h) assigns numeric levels:

| Level | Value | Operators |
|-------|-------|-----------|
| `Comma` | 1 | `,` |
| `Assignment` | 2 | `=` `+=` `-=` `*=` `/=` etc. |
| `Conditional` | 3 | `?:` |
| `LogicalOr` | 4 | `\|\|` |
| `LogicalAnd` | 5 | `&&` |
| `InclusiveOr` | 6 | `\|` |
| `ExclusiveOr` | 7 | `^` |
| `And` | 8 | `&` |
| `Equality` | 9 | `==` `!=` |
| `Relational` | 10 | `<` `>` `<=` `>=` |
| `Spaceship` | 11 | `<=>` |
| `Shift` | 12 | `<<` `>>` |
| `Additive` | 13 | `+` `-` |
| `Multiplicative` | 14 | `*` `/` `%` |
| `PointerToMember` | 15 | `.*` `->*` |

The entry points form a natural hierarchy:

```cpp
ExprResult ParseExpression(TypoCorrectionTypeBehavior = AllowNonTypes);
ExprResult ParseAssignmentExpression(TypoCorrectionTypeBehavior = AllowNonTypes);
ExprResult ParseRHSOfBinaryExpression(ExprResult LHS, prec::Level MinPrec);
ExprResult ParseCastExpression(CastParseKind ParseKind, bool isAddressOfOperand,
                               bool &NotCastExpr, TypeCastState isTypeCast,
                               bool isVectorLiteral, bool *NotPrimaryExpression);
```

`ParseExpression()` calls `ParseAssignmentExpression()`, which calls `ParseRHSOfBinaryExpression(..., prec::Comma)` after parsing the left-hand side. `ParseRHSOfBinaryExpression` loops as long as the current token has precedence ≥ `MinPrec`, consuming the operator and recursing with `MinPrec + 1` (left-associative) or `MinPrec` (right-associative).

### 32.6.2 Cast and Postfix Expressions

```cpp
ExprResult ParseCastExpression(CastParseKind ParseKind, ...);
ExprResult ParsePostfixExpressionSuffix(ExprResult LHS);
```

`ParseCastExpression` handles all unary prefix forms: `sizeof`, `alignof`, `&`, `*`, `+`, `-`, `!`, `~`, `++`, `--`, `co_await`, and GNU extensions. When `ParseKind == CastParseKind::AnyCastExpr` and the current token is `(`, it enters disambiguation logic to distinguish `(type)expr` from `(expr)`.

`ParsePostfixExpressionSuffix` handles suffix forms: function call `(args)`, array subscript `[expr]`, member access `.id`, pointer member access `->id`, `++`, `--`, and the `->*`/`.*` pointer-to-member operators.

### 32.6.3 Constant Expressions

```cpp
ExprResult ParseConstantExpression();
ExprResult ParseConstantExpressionInExprEvalContext(
    TypeCastState isTypeCast = NotTypeCast);
```

`ParseConstantExpression()` is used in contexts that require an integral constant expression (ICE): array bounds, bit-field widths, `case` labels, `static_assert` conditions. It calls `ParseAssignmentExpression` then `Actions.ActOnConstantExpression()`.

### 32.6.4 `ParseParenExpression` — The Type/Expression Crossroads

```cpp
ExprResult ParseParenExpression(ParenParseOption &ExprType,
                                 bool stopIfCastExpr,
                                 bool isTypeCast,
                                 ParsedType &CastTy,
                                 SourceLocation &RParenLoc);
```

`ParseParenExpression` is one of the most complex methods in the parser: a `(` can begin a parenthesised expression, a cast, a compound literal, a GNU statement expression `(stmt)`, a fold expression `(e op ... op e)`, or a parenthesised type-id. The `ParenParseOption` enum controls how much latitude the method has:

```cpp
enum class ParenParseOption {
  SimpleExpr,       // only '(' expression ')'
  FoldExpr,         // also fold-expression
  CompoundStmt,     // also GNU statement expression
  CompoundLiteral,  // also '(' type-name ')' '{' ... '}'
  CastExpr          // also '(' type-name ')' <anything>
};
```

The method uses a `TentativeParsingAction` to test whether the paren content is a type-id (indicating a cast); if `isCXXTypeId()` returns true, `CastTy` is set and `ExprType` is updated to `CastExpr`.

### 32.6.5 Built-In Primary Expressions

`ParseBuiltinPrimaryExpression()` handles the rich set of compiler built-ins that masquerade as expressions:

- `__builtin_va_arg(ap, T)` — variadic argument extraction
- `__builtin_offsetof(T, member)` — struct member offset
- `__builtin_choose_expr(cond, e1, e2)` — compile-time ternary
- `__builtin_convertvector(vec, T)` — vector type conversion
- `__builtin_shufflevector(...)` — SIMD lane shuffle
- `__builtin_FUNCTION()`, `__builtin_FILE()`, `__builtin_LINE()` — source location builtins

Each built-in has its own argument-count and argument-type rules enforced by `Actions.ActOnBuiltin*` callbacks.

### 32.6.6 Initializer Lists

Brace-enclosed initializer lists `{...}` are parsed by `ParseInitializerWithPotentialDesignator()` (in `ParseInit.cpp`). The parser must distinguish C99 designated initializers (`.field = val`, `[idx] = val`) from C++ aggregate initializers. A `BalancedDelimiterTracker` ensures the matching `}` is consumed or a diagnostic is emitted.

---

## 32.7 Statement Parsing

### 32.7.1 `ParseStatement()` Dispatch

```cpp
StmtResult ParseStatement(SourceLocation *TrailingElseLoc = nullptr,
                           ParsedStmtContext StmtCtx = ParsedStmtContext::SubStmt);
```

`ParseStatement` examines the current token and dispatches to a specific handler. The full dispatch table maps token kinds to methods:

| Token | Handler |
|-------|---------|
| `{` | `ParseCompoundStatement()` |
| `if` | `ParseIfStatement()` |
| `switch` | `ParseSwitchStatement()` |
| `while` | `ParseWhileStatement()` |
| `do` | `ParseDoStatement()` |
| `for` | `ParseForStatement()` |
| `goto` | `ParseGotoStatement()` |
| `return` | `ParseReturnStatement()` |
| `break` | `ParseBreakStatement()` |
| `continue` | `ParseContinueStatement()` |
| `case` / `default` | `ParseCaseStatement()` / `ParseDefaultStatement()` |
| `__asm__` / `asm` | `ParseAsmStatement()` |
| `try` | `ParseCXXTryBlock()` |
| identifier+`:` | `ParseLabeledStatement()` |
| anything else | `ParseExprStatement()` or declaration |

### 32.7.2 Compound Statements

```cpp
StmtResult ParseCompoundStatement(bool isStmtExpr = false);
StmtResult ParseCompoundStatement(bool isStmtExpr, unsigned ScopeFlags);
StmtResult ParseCompoundStatementBody(bool isStmtExpr = false);
```

`ParseCompoundStatement` creates a `ParseScope` with `Scope::CompoundStmtScope`, consumes `{`, calls `ParseCompoundStatementBody()` to process the statement sequence, then consumes `}`. Each statement is dispatched through `ParseStatementOrDeclaration()`, which tries the tentative declaration disambiguator first.

### 32.7.3 `for` and Range-Based `for`

```cpp
StmtResult ParseForStatement(SourceLocation *TrailingElseLoc,
                              LabelDecl *PrecedingLabel);
```

After consuming `for (`, the parser must distinguish four syntactic forms:
1. C-style `for (init; cond; incr)`
2. C++11 range-based `for (decl : range)`
3. OpenMP `for` (when inside an OpenMP context)
4. Coroutine-aware `for co_await` (C++ Coroutines TS extension)

The disambiguating heuristic: after parsing `for (`, if a complete declaration is seen followed by `:`, it is a range-`for`. The `ForRangeInit` struct carries the parsed range initialiser to `Actions.ActOnCXXForRangeDecl()`.

### 32.7.4 Coroutine Statements

Coroutine expressions appear as ordinary expressions within `ParseStatement`:
- `co_await expr` — triggers `ParseCastExpression` with the `co_await` prefix handler, calls `Actions.ActOnCoawaitExpr()`
- `co_yield expr` — parsed in `ParseExprCXX.cpp`, calls `Actions.ActOnCoyieldExpr()`
- `co_return [expr|braced-init]` — handled inside `ParseReturnStatement()`, which detects `co_return` and calls `Actions.ActOnCoreturnStmt()`

---

## 32.8 Late Parsing of Inline Method Bodies

### 32.8.1 `LateParsedDeclaration` and `LexedMethod`

Class member function bodies defined inline in the class body cannot be parsed at the point they appear, because they may reference class members declared later. Clang implements "late parsing" by storing the raw tokens for the body during the class parse, then replaying them after the entire class body has been processed.

```cpp
struct LexedMethod : public LateParsedDeclaration {
  Parser *Self;
  Decl *D;
  bool TemplateScope;
  CachedTokens Toks;    // raw token sequence for the body
  void ParseLexedMethodDefs() override;
};
```

When `ParseCXXClassMemberDeclaration` encounters a member function body (`{`), rather than calling `ParseFunctionStatementBody` immediately, it uses `ConsumeAndStoreUntil(tok::r_brace, LM.Toks)` to cache all tokens through the matching `}`. A `LexedMethod` object is registered on the active `ParsingClassDefinition`.

After `ParseCXXMemberSpecification` completes the class body, `ParseLexedMethodDefs()` iterates over all `LexedMethod` entries, replays their token sequences into the preprocessor via `PP.EnterTokenStream()`, and calls `ParseFunctionStatementBody` for each.

### 32.8.2 `LateParsedDefaultArgument`

Default argument expressions in function declarations face a similar problem: they can reference later-declared class members. The parser records a `LateParsedDefaultArgument`:

```cpp
struct LateParsedDefaultArgument {
  explicit LateParsedDefaultArgument(Decl *P,
      std::unique_ptr<CachedTokens> Toks = nullptr)
      : Param(P), Toks(std::move(Toks)) {}
  Decl *Param;
  std::unique_ptr<CachedTokens> Toks;
};
```

Parsing is deferred via `ParseLexedMethodDeclarations()`, which re-injects the stored tokens and calls `Sema::ActOnParamDefaultArgument()`.

### 32.8.3 `LateParsedAttribute`

GNU attributes on class members whose argument expressions refer to class-scope declarations are also stored for late parsing via `LateParsedAttribute`. The `ParseLexedAttributeList` and `ParseLexedCAttributeList` methods handle replay for `[[]]` and `__attribute__` forms respectively.

---

## 32.9 C++ Specific Declarations

### 32.9.1 Namespaces

```cpp
DeclGroupPtrTy ParseNamespace(DeclaratorContext Context,
                               SourceLocation &DeclEnd,
                               SourceLocation InlineLoc = SourceLocation());
```

`ParseNamespace` handles anonymous namespaces, named namespaces, inline namespaces, and nested namespace definitions (`namespace A::B::C {}`). It creates a `ParseScope` with `Scope::NamespaceScope`, calls `Actions.ActOnStartNamespaceDef()`, parses the body, then calls `Actions.ActOnFinishNamespaceDef()`.

### 32.9.2 Class and Struct Declarations

```cpp
void ParseClassSpecifier(tok::TokenKind TagTokKind, SourceLocation TagLoc,
                          DeclSpec &DS, ParsedTemplateInfo &TemplateInfo,
                          AccessSpecifier AS, bool EnteringContext,
                          DeclSpecContext DSC, ParsedAttributes &Attrs);

DeclGroupPtrTy ParseCXXClassMemberDeclaration(
    AccessSpecifier AS, ParsedAttributes &Attrs,
    ParsedTemplateInfo &TemplateInfo = ParsedTemplateInfo(),
    ParsingDeclRAIIObject *DiagsFromTParams = nullptr);
```

`ParseClassSpecifier` is called from `ParseDeclarationSpecifiers` when a `class`, `struct`, or `union` keyword is encountered. If a body (`{`) follows, it calls `ParseCXXMemberSpecification()`:

```cpp
void ParseCXXMemberSpecification(SourceLocation StartLoc,
                                  SourceLocation AttrFixitLoc,
                                  ParsedAttributes &Attrs,
                                  unsigned TagType,
                                  Decl *TagDecl);
```

`ParseCXXMemberSpecification` creates a `ParseScope` with `Scope::ClassScope | Scope::DeclScope`, calls `Actions.ActOnStartCXXMemberDeclarations()`, loops over member declarations using `ParseCXXClassMemberDeclaration()`, processes access specifiers (`public:`, `protected:`, `private:`) inline, and finally calls `Actions.ActOnFinishCXXMemberSpecification()`.

The `ParsingClassDefinition` RAII object tracks the state for nested class definitions and drives the late-parse phase after the outer `}` is consumed:

```cpp
class ParsingClassDefinition {
  Parser &P;
  bool Popped;
  Sema::ParsingClassState State;
public:
  ParsingClassDefinition(Parser &P, Decl *TagOrTemplate,
                          bool TopLevelClass, bool IsInterface);
  ~ParsingClassDefinition();
};
```

Member function bodies are lexed but not parsed during the class body; they are stored as `CachedTokens` and parsed afterwards via `ParseLexedMethodDefs()`. This "late parsing" allows member function bodies to reference members declared later in the class body.

### 32.9.3 Template Declarations

```cpp
bool ParseTemplateParameters(MultiParseScope &TemplateScopes, unsigned Depth,
                              SmallVectorImpl<NamedDecl *> &TemplateParams,
                              SourceLocation &LAngleLoc,
                              SourceLocation &RAngleLoc);

bool ParseTemplateParameterList(unsigned Depth,
                                 SmallVectorImpl<NamedDecl *> &TemplateParams);

NamedDecl *ParseTemplateParameter(unsigned Depth, unsigned Position);
```

`ParseDeclarationStartingWithTemplate` is the primary entry point, called from `ParseExternalDeclaration` when `template` is the current token:

```cpp
DeclGroupPtrTy ParseDeclarationStartingWithTemplate(
    DeclaratorContext Context, SourceLocation &DeclEnd,
    ParsedAttributes &DeclAttrs, AccessSpecifier AS);
```

`ParseTemplateDeclarationOrSpecialization` distinguishes between primary templates, explicit specialisations, and explicit instantiations based on the presence and content of the `template` keyword and angle brackets. The `ParsedTemplateKind` enum records which kind is being parsed:

```cpp
enum class ParsedTemplateKind {
  NonTemplate, Template, ExplicitSpecialization, ExplicitInstantiation
};
```

Template template parameters require `ParseTemplateTemplateParameter()`; type parameters use `ParseTypeParameter()`; non-type parameters fall through to the ordinary declarator parser. Explicit instantiations (`template class vector<int>;`) are handled by:

```cpp
DeclGroupPtrTy ParseExplicitInstantiation(DeclaratorContext Context,
                                           SourceLocation ExternLoc,
                                           SourceLocation TemplateLoc,
                                           SourceLocation &DeclEnd,
                                           ParsedAttributes &Attrs,
                                           AccessSpecifier AS);
```

The `Depth` parameter tracks nesting for variadic templates; `Position` identifies the parameter index for the associated `TemplateTypeParmDecl` or `NonTypeTemplateParmDecl`. `MultiParseScope` accumulates all nested `template<...>` scopes before the declaration body is parsed.

Template argument lists use `ParseTemplateArgumentList`:

```cpp
bool ParseTemplateArgumentList(TemplateArgList &TemplateArgs,
                                TemplateTy Template,
                                SourceLocation OpenLoc);
ParsedTemplateArgument ParseTemplateArgument();
```

Each argument is parsed as either a type-id (for type template parameters), an expression (for non-type parameters), or a template-name (for template template parameters). The disambiguation uses `isCXXTypeId(TentativeCXXTypeIdContext::AsTemplateArgument, isAmbiguous)`.

### 32.9.4 C++20 Modules

```cpp
Decl *ParseModuleImport(SourceLocation AtLoc,
                         Sema::ModuleImportState &ImportState);
```

C++20 module declarations (`module foo;`, `export module foo;`, `import foo;`) are parsed in `Parser.cpp`. The `Sema::ModuleImportState` tracks whether the parser is before or after the module declaration, which affects which declarations are legal. Global module fragment markers (`module;`) and private module fragment markers (`module : private;`) are handled inline in the `ParseTopLevelDecl` loop.

### 32.9.5 `using` Declarations and Aliases

```cpp
DeclGroupPtrTy ParseUsingDeclaration(DeclaratorContext Context,
                                      ParsedTemplateInfo &TemplateInfo,
                                      SourceLocation UsingLoc,
                                      SourceLocation &DeclEnd,
                                      ParsedAttributes &Attrs,
                                      AccessSpecifier AS = AS_none);
```

`ParseUsingDeclaration` handles `using Base::member;` (using-declaration), `using T = int;` (type alias), and `using enum E;` (C++20 using-enum). The `Actions.ActOnAliasDeclaration()` method is called for alias forms.

### 32.9.6 `static_assert`, Lambda, and Try-Block

```cpp
Decl *ParseStaticAssertDeclaration(SourceLocation &DeclEnd);
ExprResult ParseLambdaExpression();
StmtResult ParseCXXTryBlock();
```

`ParseStaticAssertDeclaration` parses `static_assert(expr)` (C++17 one-argument form) and `static_assert(expr, msg)`. `ParseLambdaExpression` first calls `ParseLambdaIntroducer()` to consume the `[...]` capture list, then `ParseLambdaExpressionAfterIntroducer()` for the parameter list, trailing return type, and body. `ParseCXXTryBlock` implements the full `try { } catch (decl) { }` grammar including multiple catch clauses and a catch-all (`catch (...)`).

---

## 32.10 C++ Specific Type and Expression Constructs

### 32.10.1 C++ Cast Expressions

```cpp
// In ParseExprCXX.cpp
ExprResult ParseCXXCastExpression(tok::TokenKind Kind);
```

`static_cast<T>(e)`, `dynamic_cast<T>(e)`, `reinterpret_cast<T>(e)`, and `const_cast<T>(e)` share a common parser since the syntactic structure is identical. The token kind selects which `Sema` action is called: `ActOnCXXNamedCast()`.

### 32.10.2 `new` and `delete`

```cpp
ExprResult ParseCXXNewExpression(bool UseGlobal, SourceLocation Start);
ExprResult ParseCXXDeleteExpression(bool UseGlobal, SourceLocation Start);
```

`ParseCXXNewExpression` handles `new (placement) T[n] (initializer)`. It must parse an optional placement argument list, a type-id that may include array dimensions, and an optional initializer. `ParseCXXDeleteExpression` handles `delete expr` and `delete[] expr`.

### 32.10.3 `if` with Initializer and `ParseCXXCondition`

C++17 introduced init-statements in `if` and `switch`:

```cpp
Sema::ConditionResult ParseCXXCondition(StmtResult *InitStmt,
                                         SourceLocation Loc,
                                         Sema::ConditionKind CK,
                                         bool MissingOK,
                                         ForRangeInit *FRI,
                                         bool EnterForConditionScope);
```

When `InitStmt` is non-null, the parser first attempts to parse an optional `decl ;` initializer before the condition expression or condition declaration. This handles `if (int x = f(); x > 0)`.

### 32.10.4 Structured Bindings

Structured bindings (`auto [a, b] = pair`) are represented by a `DecompositionDeclarator` inside the `Declarator` class. When `ParseDeclarator` encounters `[` after the leading `auto`/`const auto&`, it calls `ParseDecompositionDeclarator()`, which populates `Declarator::BindingGroup` with the identifier list. The `Actions.ActOnDecompositionDeclarator()` call produces the binding `Decl` hierarchy.

### 32.10.5 `throw`

```cpp
ExprResult ParseCXXThrowExpression();
```

A `throw;` (rethrow) produces a null expression operand; `throw expr` requires parsing an assignment expression. Both call `Actions.ActOnCXXThrow()`.

---

## 32.11 Attributes

### 32.11.1 Attribute Dispatch

The unified entry point:

```cpp
void ParseAttributes(unsigned WhichAttrKinds, ParsedAttributes &Attrs,
                     LateParsedAttrList *LateAttrs = nullptr);
```

`WhichAttrKinds` is a bitmask of:

```cpp
enum ParseAttrKindMask {
  PAKM_GNU      = 1 << 0,   // __attribute__((...))
  PAKM_Declspec = 1 << 1,   // __declspec(...)
  PAKM_CXX11    = 1 << 2,   // [[...]]
};
```

The `MaybeParseGNUAttributes`, `MaybeParseCXX11Attributes`, and `MaybeParseMicrosoftDeclSpecs` wrappers check the current token before dispatching.

### 32.11.2 GNU Attributes

```cpp
void ParseGNUAttributes(ParsedAttributes &Attrs,
                         LateParsedAttrList *LateAttrs = nullptr,
                         Declarator *D = nullptr);
bool ParseSingleGNUAttribute(ParsedAttributes &Attrs, SourceLocation &EndLoc,
                              LateParsedAttrList *LateAttrs, Declarator *D);
```

`ParseGNUAttributes` consumes the outer `__attribute__((...))` wrapper—which may contain multiple comma-separated attributes—and calls `ParseSingleGNUAttribute` for each. GNU attribute arguments conform to:

```
attrib-name '(' identifier ')'
attrib-name '(' identifier ',' expr-list ')'
attrib-name '(' expr-list ')'
```

Late-parsed attributes (those whose arguments require semantic context, such as `__attribute__((cleanup(...)))` with a function reference) are added to `LateAttrs` for deferred parsing after their declaration is complete.

### 32.11.3 C++11 / C23 Attributes

```cpp
void ParseCXX11AttributeSpecifierInternal(ParsedAttributes &Attrs,
                                           CachedTokens *OpenMPTokens,
                                           SourceLocation *EndLoc = nullptr);
void ParseCXX11AttributeSpecifier(ParsedAttributes &Attrs,
                                   SourceLocation *EndLoc = nullptr);
```

C++11 attributes use `[[` ... `]]` syntax. `isCXX11AttributeSpecifier()` distinguishes `[[` as an attribute start from `[[` as a double `[` (e.g., `matrix[i][j]`). The namespace-qualified form `[[vendor::attr(args)]]` is parsed by recognising the `::` between the namespace identifier and the attribute name.

### 32.11.4 Microsoft `__declspec`

```cpp
void ParseMicrosoftDeclSpecs(ParsedAttributes &Attrs);
```

Activated when `-fms-extensions` is enabled or when targeting Windows. `__declspec(...)` takes a comma-separated list of declarator-like specifiers: `__declspec(dllexport)`, `__declspec(align(16))`, `__declspec(uuid("..."))`, etc.

### 32.11.5 `ParsedAttr` and `ParseAttributeArgsCommon`

All three attribute syntaxes eventually produce `ParsedAttr` objects stored in `ParsedAttributes`. The attribute argument parser:

```cpp
unsigned ParseAttributeArgsCommon(
    IdentifierInfo *AttrName, SourceLocation AttrNameLoc,
    ParsedAttributes &Attrs, SourceLocation *EndLoc,
    IdentifierInfo *ScopeName, SourceLocation ScopeLoc,
    ParsedAttr::Form Form);
```

returns the count of parsed arguments. Arguments may be identifiers, integer literals, string literals, or full type expressions, depending on the attribute's `ParseKind` registered in `Attr.td`.

---

## 32.12 Pragma Handling

### 32.12.1 `PragmaHandler` Infrastructure

Pragmas are preprocessor events, but their semantic effects are felt during parsing. Clang registers a set of `PragmaHandler` objects with the `Preprocessor`; when the preprocessor encounters `#pragma foo`, it calls the registered handler's `HandlePragma()` method. The handler typically emits a synthetic annotation token (`tok::annot_pragma_*`) into the token stream; the parser then sees this annotation and calls the appropriate semantic action.

The `Parser` class owns a suite of pragma handlers as `unique_ptr` members:

```cpp
std::unique_ptr<PragmaHandler> AlignHandler;
std::unique_ptr<PragmaHandler> GCCVisibilityHandler;
std::unique_ptr<PragmaHandler> PackHandler;       // #pragma pack(...)
std::unique_ptr<PragmaHandler> OpenMPHandler;     // #pragma omp ...
std::unique_ptr<PragmaHandler> OpenACCHandler;    // #pragma acc ...
std::unique_ptr<PragmaHandler> MSCommentHandler;  // #pragma comment(...)
std::unique_ptr<PragmaHandler> FloatControlHandler; // #pragma float_control
// ... and many more
```

These are initialised in `Parser::Initialize()` and removed from the `Preprocessor` in `~Parser()`.

### 32.12.2 Pragma Annotation Token Processing

When the parser's statement dispatcher encounters a pragma annotation token, it calls methods in `ParsePragma.cpp`:

- `HandlePragmaAlign()` — `#pragma align(N)` and `#pragma options align=...`
- `HandlePragmaPack()` — `#pragma pack(push/pop/N)`
- `HandlePragmaVisibility()` — `#pragma GCC visibility push/pop`
- `HandlePragmaLoopHint()` — `#pragma clang loop vectorize(enable)` etc.
- `HandlePragmaOpenMP()` — delegates to `ParseOpenMPDeclarativeOrExecutableDirective()`

OpenMP pragmas undergo their own sub-parsing: `#pragma omp parallel for` causes the parser to enter `ParseOpenMPDeclarativeOrExecutableDirective`, which parses the directive name, associated clauses, and the governed statement, calling `Actions.ActOnOpenMPParallelDirective()` and similar.

---

## 32.13 Error Recovery

### 32.13.1 `SkipUntil`

```cpp
enum SkipUntilFlags {
  StopAtSemi          = 1 << 0,
  StopBeforeMatch     = 1 << 1,
  StopAtCodeCompletion= 1 << 2
};

bool SkipUntil(tok::TokenKind T, SkipUntilFlags Flags = ...);
bool SkipUntil(ArrayRef<tok::TokenKind> Toks, SkipUntilFlags Flags = ...);
```

`SkipUntil` reads and discards tokens until a target token kind is found (or `eof` is reached). It balances delimiter pairs—if an `(` is seen before the target, it skips until the matching `)`. The `StopAtSemi` flag is the most common: after a malformed expression, recovery skips to the next `;` so parsing can resume at the next statement. The `StopBeforeMatch` flag leaves the target token unconsumed, allowing the caller to decide whether to consume it.

Typical recovery patterns:

```cpp
// After a bad declaration specifier: skip to semicolon
SkipUntil(tok::semi);

// After a malformed expression inside braces: skip to closing brace
SkipUntil(tok::r_brace, StopAtSemi);

// Skip to either semicolon or closing brace
SkipUntil({tok::semi, tok::r_brace}, StopBeforeMatch);
```

### 32.13.2 `BalancedDelimiterTracker`

```cpp
class BalancedDelimiterTracker {
public:
  BalancedDelimiterTracker(Parser &P, tok::TokenKind Kind,
                            tok::TokenKind FinalToken = tok::unknown);
  bool expectAndConsume(unsigned DiagID = diag::err_expected,
                        const char *Msg = "",
                        tok::TokenKind SkipToTok = tok::unknown);
  bool consumeOpen();
  bool consumeClose();
  SourceLocation getOpenLocation() const;
  SourceLocation getCloseLocation() const;
  SourceRange getRange() const;
};
```

`BalancedDelimiterTracker` wraps a matched pair of delimiters—`()`, `[]`, or `{}`—and provides `consumeOpen()` / `consumeClose()`. If `consumeClose()` finds the wrong token, it emits an appropriate "expected `)'" diagnostic and either skips to the matching close or inserts a recovery point. The tracker records source locations for both delimiters, enabling precise diagnostic ranges.

Usage in `ParseCompoundStatement`:

```cpp
BalancedDelimiterTracker T(*this, tok::l_brace);
T.consumeOpen();
// ... parse body statements ...
T.consumeClose();
// T.getRange() covers the entire braced block
```

### 32.13.3 `DiagnosticErrorTrap`

```cpp
class DiagnosticErrorTrap {
  DiagnosticsEngine &Diags;
  unsigned NumErrors;
public:
  explicit DiagnosticErrorTrap(DiagnosticsEngine &Diags);
  bool hasErrorOccurred() const;
  bool hasUnrecoverableErrorOccurred() const;
  void reset();
};
```

`DiagnosticErrorTrap` records the error count at construction and provides `hasErrorOccurred()` to detect whether any new errors were emitted during a sub-parse. Recovery branches use this to avoid cascading diagnostics: if the speculative parse produced errors, the branch is abandoned.

### 32.13.4 Semicolon Insertion

`ExpectAndConsumeSemi` applies two recovery heuristics before emitting a diagnostic:
1. If the next token is `)`, `]`, or `}` at the same or outer nesting level, it is likely a stray closing delimiter—consume it and retry.
2. If the next token begins a new statement keyword (`if`, `for`, etc.), insert a virtual semicolon (do not consume any token) and proceed.

The "expected `;'" diagnostic includes a FixIt hint inserting `;` at the prior token's end location.

### 32.13.5 Typo Correction at Parse Time

The parser interacts with Sema's typo-correction infrastructure at key disambiguation points. `TryAnnotateName` accepts a `CorrectionCandidateCallback*` parameter; if name lookup fails but a similar name exists in scope, Sema may suggest the correction. The `TypoCorrectionTypeBehavior` enum controls whether corrections are restricted to type names, non-type names, or both:

```cpp
enum class TypoCorrectionTypeBehavior {
  AllowNonTypes,
  AllowTypes,
  AllowBoth,
};
```

`ParseExpression(TypoCorrectionTypeBehavior)` passes this enum to downstream disambiguation calls, enabling location-sensitive correction: in a declaration-specifier context only type corrections are offered, while in an expression context only non-type corrections apply.

### 32.13.6 Cascading Diagnostic Suppression

A key challenge in error recovery is preventing one syntactic error from generating dozens of follow-on diagnostics as the parser stumbles through corrupted state. The parser tracks whether the current scope has already seen an error via `DiagnosticErrorTrap` and suppresses subsequent diagnostics in the same syntactic region. The `cutOffParsing()` method takes the extreme approach of setting `Tok.setKind(tok::eof)`, immediately terminating all parsing—used only when the code-completion point has been reached or when a stack-exhaustion sentinel fires.

`StackExhaustionHandler` (stored as `Parser::StackHandler`) monitors recursion depth and emits a "stack exhaustion" fatal diagnostic when the remaining stack falls below a threshold. The check is inserted at the top of `ParseStatementOrDeclaration` and `ParseCastExpression`, the two deepest recursion points.

---

## 32.14 `ParseAST()` and the Full Compilation Pipeline

### 32.14.1 `FrontendAction` Integration

`FrontendAction` (in `clang/include/clang/Frontend/FrontendAction.h`) is the base class for all compiler front-end phases. `ASTFrontendAction::ExecuteAction()` calls:

```cpp
ParseAST(CI.getSema(), CI.getFrontendOpts().ShowStats,
         CI.getFrontendOpts().SkipFunctionBodies);
```

This is the path taken by `-Xclang -ast-dump`, `-emit-llvm`, and all standard compilation modes. `PreprocessorFrontendAction` (used for `-E`) never calls `ParseAST`.

### 32.14.2 `ASTConsumer` Interface

`ASTConsumer` provides two key callbacks invoked from `ParseAST`:

```cpp
class ASTConsumer {
public:
  virtual bool HandleTopLevelDecl(DeclGroupRef D);    // called per top-level decl
  virtual void HandleTranslationUnit(ASTContext &Ctx); // called at EOF
};
```

`HandleTopLevelDecl` returns `false` to abort early (used in `clang-repl` to stop on error). `HandleTranslationUnit` is where back-end code generation begins: `BackendConsumer::HandleTranslationUnit` calls `EmitBackendOutput()`, which drives the LLVM IR generation and optimisation pipeline.

### 32.14.3 `ActOnEndOfTranslationUnit`

```cpp
void Sema::ActOnEndOfTranslationUnit();
```

Called after `HandleTranslationUnit` in `ParseAST`, this performs TU-scope semantic work: implicit instantiation of pending template specialisations, checking for `main`'s presence in freestanding mode, deferred code analysis for constexpr functions, and ODR (One Definition Rule) violation detection. Only after this call is the AST fully complete.

### 32.14.4 `SkipFunctionBodies` Mode

When `ParseAST` is called with `SkipFunctionBodies = true` (exposed as `clang -Xclang -skip-function-bodies`), the parser skips the body of non-template function definitions by calling `SkipUntil(tok::r_brace)` after identifying the opening `{`, without constructing any `Stmt` or `Expr` nodes. Template function definitions must still be parsed to allow argument deduction and instantiation. This mode is used by indexing tools that only need declarations, not full ASTs, dramatically reducing parse time on large files.

---

## 32.15 Incremental Parsing (clang-repl)

### 32.15.1 Architecture

`clang-repl` provides an interactive C++ shell using incremental compilation. The `Interpreter` class (in [`clang/include/clang/Interpreter/Interpreter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Interpreter/Interpreter.h)) wraps an `IncrementalParser` and an `IncrementalExecutor`:

```cpp
class Interpreter {
  std::unique_ptr<IncrementalAction> Act;
  std::unique_ptr<IncrementalParser> IncrParser;
  std::unique_ptr<IncrementalExecutor> IncrExecutor;
  std::list<PartialTranslationUnit> PTUs;
  // ...
};
```

The key entry points:

```cpp
llvm::Expected<PartialTranslationUnit &> Parse(llvm::StringRef Code);
llvm::Error Execute(PartialTranslationUnit &T);
llvm::Error ParseAndExecute(llvm::StringRef Code, Value *V = nullptr);
```

### 32.15.2 `PartialTranslationUnit`

```cpp
struct PartialTranslationUnit {
  TranslationUnitDecl *TUPart = nullptr;
  std::unique_ptr<llvm::Module> TheModule;
};
```

Each user input corresponds to one `PartialTranslationUnit` appended to `Interpreter::PTUs`. `TUPart` points into the persistent `TranslationUnitDecl` (the root of the entire AST, which is never destroyed), and `TheModule` holds the LLVM IR generated for this input alone.

### 32.15.3 `IncrementalParser` Operation

`IncrementalParser::Parse(StringRef Code)` appends new source text to the single long-lived `CompilerInstance`. Rather than calling `ParseAST()` in its standard form, the incremental parser uses a `Parser` in a persistent state: the `Parser` object is constructed once and survives across multiple inputs. Each new chunk of text is appended as a virtual file and lexed by extending the `Preprocessor`'s token stream.

After parsing each input chunk, `IncrementalParser` calls `Sema::PerformPendingInstantiations()` to instantiate templates accumulated in this chunk, then drives LLVM IR generation for the new `DeclGroupRef`s streamed to the `ASTConsumer`. The resulting LLVM module is JIT-compiled by the `IncrementalExecutor` via ORC JIT.

### 32.15.4 `Undo` and Symbol Visibility

```cpp
llvm::Error Interpreter::Undo(unsigned N = 1);
```

`Undo` removes the last `N` partial translation units from the AST and unregisters their symbols from the JIT. This is implemented by popping `PTUs` from the tail and calling `Sema::ActOnTransactionCommit()` or `ActOnTransactionRollback()` equivalents. The persistent `TranslationUnitDecl` must correctly reflect the undone state for subsequent parses.

The `IncrementalCompilerBuilder` utility creates a pre-configured `CompilerInstance` suitable for incremental use, setting flags such as `-fno-delayed-template-parsing` and disabling PCH integration that would conflict with the append-only model.

### 32.15.5 Value Printing and `ParseAndExecute`

```cpp
llvm::Error Interpreter::ParseAndExecute(llvm::StringRef Code, Value *V = nullptr);
```

When `clang-repl` evaluates a bare expression (not a declaration or statement), the interpreter wraps it in a synthetic function and calls `ParseAndExecute`. If the expression has a non-void type, the result is stored in the `Value` output parameter. `Value` (declared in `clang/include/clang/Interpreter/Value.h`) is a discriminated union capable of holding integer, floating-point, pointer, and user-defined types. For non-trivial types the interpreter arranges storage via `CompileDtorCall` to ensure proper destructor invocation when the value goes out of scope.

This "expression result capture" requires the parser to recognise when an input is a statement vs. a value-producing expression—a distinction made by attempting to parse the input as an expression-statement and checking whether the result type is non-void.

---

## 32.16 Code Completion Integration

### 32.16.1 `CodeCompletionHandler` Inheritance

`Parser` inherits from `CodeCompletionHandler` to intercept code-completion tokens (`tok::code_completion`) generated by the preprocessor when the source position matches the `-code-completion-at` location. At each point in the grammar where completion is sensible, the parser calls a `CodeCompleteConsumer` method via `Actions.CodeComplete*()`:

```cpp
// Example from ParseDecl.cpp — completing a declaration specifier
if (Tok.is(tok::code_completion)) {
  Actions.CodeCompleteDeclSpec(getCurScope(), DS, ...);
  cutOffParsing();
  return;
}
```

`cutOffParsing()` sets `Tok` to `tok::eof`, short-circuiting all further parsing after the completion point is reached. The `CalledSignatureHelp` flag prevents multiple `ProduceSignatureHelp` calls at different nesting levels from all firing; only the deepest (innermost) function call reports.

### 32.16.2 Preferred Type Tracking

```cpp
PreferredTypeBuilder PreferredType;
```

`PreferredTypeBuilder` tracks the expected type for the current expression context, set by assignment, function-argument, and initializer contexts. This information is passed to `CodeCompleteExpression` to rank completion candidates by type match quality. The preferred type state is saved/restored by `TentativeParsingAction` so that speculative parses do not permanently modify it.

---

## Chapter Summary

- `clang::Parser` is a hand-written recursive-descent parser that couples directly to `Sema` via an `Actions` member, calling `Act*` methods at every AST-construction point.
- `ParseAST()` is the primary entry point, constructing `Parser`, driving the top-level declaration loop, and calling `ASTConsumer::HandleTopLevelDecl` and `HandleTranslationUnit` as output.
- Token management centres on the single `Tok` lookahead token; specialized `Consume*` methods maintain delimiter-balance counters used by error recovery.
- Tentative parsing via `TentativeParsingAction` enables C++ disambiguation (declarations vs. expressions, type-ids vs. cast expressions) without modifying the visible parse state.
- `DeclSpec` and `Declarator`/`DeclaratorChunk` form a two-phase declaration representation; `ParseDeclarationSpecifiers` fills `DeclSpec`, and `ParseDeclarator` builds the `DeclaratorChunk` stack.
- Expression parsing uses a Pratt (precedence-climbing) algorithm keyed on the `prec::Level` enum, with `ParseCastExpression` for unary prefix forms and `ParsePostfixExpressionSuffix` for suffix chains.
- Statement parsing dispatches on the current token to fifteen-plus dedicated handlers, with `BalancedDelimiterTracker` and `SkipUntil` providing structured error recovery.
- C++ specifics—templates, lambdas, structured bindings, coroutine keywords, C++ casts, `new`/`delete`—all live in dedicated parse files and converge on Sema action calls.
- Attributes from three syntaxes (GNU `__attribute__`, C++11 `[[]]`, MSVC `__declspec`) are normalised into `ParsedAttr` objects via `ParseAttributeArgsCommon`.
- `clang-repl` extends the parser via `IncrementalParser`, maintaining a persistent `Parser` and `Sema` state across user inputs, streaming `PartialTranslationUnit` results to an ORC JIT executor.

---

## Cross-References

- [Chapter 31 — The Preprocessor](ch31-the-preprocessor.md) — Token stream source; `PP.Lex()`, `PP.LookAhead()`, annotation tokens, `EnableBacktrackAtThisPos()`
- [Chapter 33 — Semantic Analysis (Sema)](ch33-sema.md) — `Sema::Act*` methods called at every parser decision point; scope management; type checking during parsing
- [Chapter 36 — The AST: Nodes and Representation](ch36-ast-nodes.md) — AST nodes (`Decl`, `Stmt`, `Expr` hierarchies) produced by Sema actions
- [Chapter 07 — Parsing Theory](../../part-02-compiler-theory/ch07-parsing-theory.md) — LL/LR grammars, recursive descent, operator-precedence algorithms

---

## Reference Links

| Symbol / File | Source |
|---------------|--------|
| `clang::Parser` | [`clang/include/clang/Parse/Parser.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Parse/Parser.h) |
| `ParseAST()` | [`clang/include/clang/Parse/ParseAST.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Parse/ParseAST.h) |
| `ParseAST()` implementation | [`clang/lib/Parse/ParseAST.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Parse/ParseAST.cpp) |
| `RAIIObjectsForParser.h` | [`clang/include/clang/Parse/RAIIObjectsForParser.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Parse/RAIIObjectsForParser.h) |
| `DeclSpec` / `Declarator` | [`clang/include/clang/Sema/DeclSpec.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/DeclSpec.h) |
| `ParsedAttr` | [`clang/include/clang/Sema/ParsedAttr.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/ParsedAttr.h) |
| `prec::Level` | [`clang/include/clang/Basic/OperatorPrecedence.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/OperatorPrecedence.h) |
| `Interpreter` | [`clang/include/clang/Interpreter/Interpreter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Interpreter/Interpreter.h) |
| `PartialTranslationUnit` | [`clang/include/clang/Interpreter/PartialTranslationUnit.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Interpreter/PartialTranslationUnit.h) |
| Tentative parsing | [`clang/lib/Parse/ParseTentative.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Parse/ParseTentative.cpp) |
| Expression parser | [`clang/lib/Parse/ParseExpr.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Parse/ParseExpr.cpp) |
| Statement parser | [`clang/lib/Parse/ParseStmt.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Parse/ParseStmt.cpp) |
| Template parser | [`clang/lib/Parse/ParseTemplate.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Parse/ParseTemplate.cpp) |


---

@copyright jreuben11
