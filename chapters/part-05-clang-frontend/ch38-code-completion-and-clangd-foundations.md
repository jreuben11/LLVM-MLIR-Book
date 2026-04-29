# Chapter 38 — Code Completion and clangd Foundations

*Part V — Clang Internals: Frontend Pipeline*

Code completion sits at the intersection of parsing, semantic analysis, and interactive tooling. Clang's completion infrastructure, built into the compiler's core, enables editors and IDEs to offer context-aware suggestions with full semantic fidelity — not lexical heuristics, but the same type-checking and name-lookup machinery that the compiler uses for full builds. On top of this foundation, clangd extends Clang into a fully-featured language server, adding persistent parse caching, asynchronous scheduling, an in-process symbol index, and the complete Language Server Protocol (LSP) feature set. This chapter traces the path from the `tok::code_completion` token in the lexer through `Sema`'s completion callbacks, the `CodeCompletionResult` data model, the `PrecompiledPreamble` that makes interactive performance tractable, and the layered clangd architecture that delivers hover, navigation, diagnostics, and semantic tokens to any LSP-capable editor.

---

## 38.1 The Code-Completion Token and Parser Integration

### 38.1.1 How Completion Is Triggered

Clang's code-completion mechanism is activated by a specially crafted invocation of the compiler with the `-code-completion-at=file:line:col` flag (the cc1-level flag, passed through `-Xclang`). When the lexer reaches the target position, it emits a synthetic `tok::code_completion` token instead of the real token at that location. The parser sees this token and branches into one of dozens of completion entry points in `Sema`.

The entry point on the command line looks like:

```bash
clang++ -fsyntax-only \
  -Xclang -code-completion-at=/path/to/file.cpp:42:10 \
  /path/to/file.cpp
```

Internally, `FrontendOptions::CodeCompletionAt` is set to a `ParsedSourceLocation`, and `CompilerInstance::createCodeCompletionConsumer()` instantiates a `PrintingCodeCompleteConsumer` that writes results to standard output. For programmatic use — as in clangd — a custom `CodeCompleteConsumer` subclass is installed instead.

The lexer's position-tracking logic is in `clang/lib/Lex/Lexer.cpp`. When `Lexer::LexTokenInternal()` detects that the current character position coincides with the completion point (by comparing against `SourceManager::getCodeCompletionLoc()`), it returns `tok::code_completion`. This token propagates through the token stream to the parser exactly once; subsequent tokens after the completion point are suppressed.

### 38.1.2 Parser Dispatch to Sema Completion Methods

The parser recognizes `tok::code_completion` at dozens of grammar positions and calls the corresponding `SemaCodeCompletion` method. The dispatch is straightforward: wherever the grammar has a production that could consume an identifier or keyword, the parser checks whether the current token is `tok::code_completion` and routes control accordingly. For example, in `clang/lib/Parse/ParseExpr.cpp`:

```cpp
// Inside ParsePostfixExpression, after parsing '.'
if (Tok.is(tok::code_completion)) {
  cutOffParsing();
  Actions.CodeCompletion().CodeCompleteMemberReferenceExpr(
      getCurScope(), Base.get(), /*OtherOpBase=*/nullptr,
      OpLoc, /*IsArrow=*/false, IsBaseExprStatement, PreferredType);
  return ExprError();
}
```

`cutOffParsing()` sets an internal flag that stops further token consumption, preventing spurious diagnostics from incomplete syntax after the completion point. Control returns to `SemaCodeCompletion::CodeCompleteMemberReferenceExpr()`, which performs the actual member lookup and populates results.

The full set of `SemaCodeCompletion` entry points covers every significant grammar position. A partial list of the most important methods illustrates the scope:

- `CodeCompleteOrdinaryName(Scope *, ParserCompletionContext)` — general name completion at statement, expression, or namespace scope
- `CodeCompleteMemberReferenceExpr(Scope *, Expr *, Expr *, SourceLocation, bool IsArrow, bool IsBaseExprStatement, QualType)` — member access via `.` or `->`
- `CodeCompleteCase(Scope *)` — `case` label inside a `switch`; enumerators of the switch's type are offered with priority `CCP_EnumInCase = 7`
- `CodeCompleteTag(Scope *, unsigned TagSpec)` — after `struct`, `class`, `union`, or `enum` keywords
- `CodeCompleteNamespaceAliasDecl(Scope *)` — in a `namespace X = ^` declaration
- `CodeCompleteConstructorInitializer(Decl *, ArrayRef<CXXCtorInitializer *>)` — base classes and members in a constructor initializer list
- `CodeCompleteQualifiedId(Scope *, CXXScopeSpec &, bool, bool, QualType, QualType)` — after `X::` qualified names
- `CodeCompleteUsing(Scope *)` / `CodeCompleteUsingDirective(Scope *)` — `using` declarations
- `CodeCompleteLambdaIntroducer(Scope *, LambdaIntroducer &, bool)` — capture lists in lambdas
- `CodeCompleteAttribute(AttributeSyntax, AttributeCompletion, IdentifierInfo *)` — `[[` attribute names and arguments
- `CodeCompletePreprocessorDirective(bool)` / `CodeCompleteIncludedFile(StringRef Dir, bool IsAngled)` — preprocessor directives and `#include` paths
- `CodeCompleteInitializer(Scope *, Decl *)` — aggregate initializers, with `CodeCompleteDesignator()` for designated initializers

Each of these methods collects results using the internal `ResultBuilder` class, which tracks which declarations have already been added (to avoid duplicates from inherited scopes), applies visibility filtering (access specifiers), priority adjustments, and calls `ProcessCodeCompleteResults()` on the installed consumer.

### 38.1.3 `SemaCodeCompletion`'s Internal `ResultBuilder`

The `ResultBuilder` (defined in `clang/lib/Sema/SemaCodeCompletion.cpp`) is a helper that accumulates `CodeCompletionResult` objects. It maintains a `VisitedContextSet` to deduplicate declarations seen during lookup across base-class hierarchies, and it applies the `IsAccessible` check for each declaration. When constructing results for `RK_Declaration` candidates, it calls `getCompletionComment()` to extract documentation comments attached to the declaration, which are embedded into the `CodeCompletionString` as the `BriefComment` field.

Sema's name-lookup infrastructure (`Sema::LookupName()`, `Sema::LookupQualifiedName()`) feeds declarations into the `ResultBuilder`. For global completion contexts, `Sema::GatherGlobalCodeCompletions()` performs a TU-wide scan of all visible declarations, which is the most expensive code path — this is one reason why preamble caching is critical for interactive performance.

### 38.1.4 The `PrintingCodeCompleteConsumer`

For command-line use, `PrintingCodeCompleteConsumer` (declared in [`clang/include/clang/Sema/CodeCompleteConsumer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/CodeCompleteConsumer.h)) overrides `ProcessCodeCompleteResults()` and prints each result in the format:

```
COMPLETION: move : [#void#]move(<#int dx#>, <#int dy#>)
COMPLETION: x : [#int#]x
COMPLETION: y : [#int#]y
```

The brackets encode chunk types: `[#...#]` is `CK_ResultType`, `<#...#>` is `CK_Placeholder`, bare text is `CK_TypedText` or `CK_Text`. This textual format is mainly useful for testing and scripting; production tools always install a custom `CodeCompleteConsumer`.

---

## 38.2 `CodeCompletionResult` and the Result Data Model

### 38.2.1 `ResultKind` Enumeration

Every completion candidate is represented by a `CodeCompletionResult` (defined in [`clang/include/clang/Sema/CodeCompleteConsumer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/CodeCompleteConsumer.h#L761)). The `ResultKind` field classifies each result into one of four categories:

| `ResultKind`       | Payload             | Examples                          |
|--------------------|---------------------|-----------------------------------|
| `RK_Declaration`   | `const NamedDecl *` | functions, variables, types       |
| `RK_Keyword`       | `const char *`      | `return`, `nullptr`, `override`   |
| `RK_Macro`         | `IdentifierInfo *`  | `NULL`, `NDEBUG`, `assert`        |
| `RK_Pattern`       | `CodeCompletionString *` | `if (...)`, `for (...)`, `class` |

Declaration results carry the full `NamedDecl` pointer, giving downstream consumers access to the declaration's type, location, attributes, access specifier, and documentation comment. Pattern results carry a pre-built `CodeCompletionString` encoding structural syntax (for loops, class bodies, etc.).

### 38.2.2 Priority Scoring

Each result has an unsigned `Priority` field. Lower values indicate higher priority. The priority constants defined in `CodeCompleteConsumer.h` are:

```cpp
CCP_LocalDeclaration  = 34   // declared in current scope
CCP_MemberDeclaration = 35   // found via member lookup
CCP_Keyword           = 40   // language keyword
CCP_CodePattern       = 40   // structural pattern
CCP_Declaration       = 50   // non-local declaration
CCP_Constant          = 65   // enumerator
CCP_Macro             = 70   // preprocessor macro
CCP_NestedNameSpecifier = 75
CCP_Unlikely          = 80
```

Sema further adjusts priority at result-construction time with delta constants: `CCD_InBaseClass = +2`, `CCD_ObjectQualifierMatch = -1`. Type-matching multipliers divide priority: `CCF_ExactTypeMatch = 4` (divide by 4 when the result type matches the expected type exactly), `CCF_SimilarTypeMatch = 2`. For instance, if the context expects an `int` and a variable of type `int` is found, its priority is divided by 4, making it appear first among declarations of the same raw priority tier.

### 38.2.3 Availability

`CXAvailabilityKind` (from `clang-c/Index.h`) classifies the usability of a result:

- `CXAvailability_Available` — usable without restriction
- `CXAvailability_Deprecated` — carries a `deprecated` attribute; still callable but should be avoided
- `CXAvailability_NotAvailable` — access-controlled: private member, or marked `delete`d
- `CXAvailability_NotAccessible` — found by name lookup but not accessible in the current context (e.g., a protected member accessed from a non-derived class)

The availability is computed by `CodeCompletionResult::computeCursorKindAndAvailability()`, which inspects `NamedDecl` attributes and access specifiers.

### 38.2.4 `CodeCompletionString` and Its Chunks

A `CodeCompletionString` is an immutable, arena-allocated sequence of chunks that encodes how to present (and insert) a completion. The `ChunkKind` enum covers:

| `ChunkKind`         | Meaning |
|---------------------|---------|
| `CK_TypedText`      | The text the user types to match this completion |
| `CK_Text`           | Fixed text inserted as-is |
| `CK_Optional`       | A nested `CodeCompletionString` representing optional elements (default args) |
| `CK_Placeholder`    | A tab-stop position for argument filling |
| `CK_Informative`    | Descriptive text not inserted (e.g., parameter names in a declaration context) |
| `CK_ResultType`     | Return type or variable type, shown in the UI but not inserted |
| `CK_CurrentParameter` | Highlights the active argument during signature help |
| `CK_LeftParen`, `CK_RightParen`, `CK_Comma`, etc. | Punctuation |

A function `int foo(int x, double y)` produces a `CodeCompletionString` with chunks roughly as:
`CK_ResultType("int")`, `CK_TypedText("foo")`, `CK_LeftParen`, `CK_Placeholder("int x")`, `CK_Comma`, `CK_Placeholder("double y")`, `CK_RightParen`.

### 38.2.5 `CodeCompletionAllocator` and `CodeCompletionBuilder`

`CodeCompletionString` objects are never heap-allocated directly. They live in a `CodeCompletionAllocator`, which subclasses `llvm::BumpPtrAllocator` for O(1) allocation and bulk deallocation. The `CodeCompletionBuilder` RAII class accumulates chunks and metadata, then calls `TakeString()` to finalize and allocate the string from the arena:

```cpp
CodeCompletionBuilder Builder(Allocator, CCTUInfo,
                              CCP_Declaration,
                              CXAvailability_Available);
Builder.AddResultTypeChunk("int");
Builder.AddTypedTextChunk("myFunc");
Builder.AddChunk(CK_LeftParen);
Builder.AddPlaceholderChunk("int x");
Builder.AddChunk(CK_Comma);
Builder.AddPlaceholderChunk("double y");
Builder.AddChunk(CK_RightParen);
CodeCompletionString *CCS = Builder.TakeString();
```

`CodeCompletionTUInfo` maintains a per-translation-unit map from `DeclContext *` to the string representation of the parent scope name (e.g., `"std::"`), computed lazily and cached for the lifetime of the completion session.

---

## 38.3 Completion Contexts

### 38.3.1 `CodeCompletionContext::Kind`

`SemaCodeCompletion` determines the completion context from the parser state at the completion point. The `CodeCompletionContext::Kind` enum (in `CodeCompleteConsumer.h`) has more than 40 values. Key contexts:

| Context Kind | Where completion fires |
|---|---|
| `CCC_TopLevel` | Global or namespace scope |
| `CCC_Statement` | Inside a function body at statement position |
| `CCC_Expression` | Inside an expression (rvalue context) |
| `CCC_DotMemberAccess` | After `obj.` |
| `CCC_ArrowMemberAccess` | After `ptr->` |
| `CCC_Type` | Where a type-specifier is expected |
| `CCC_ClassOrStructTag` | After `struct`/`class` keyword |
| `CCC_EnumTag` | After `enum` keyword |
| `CCC_UnionTag` | After `union` keyword |
| `CCC_ClassInheritance` | In a base-class specifier list |
| `CCC_PreprocessorExpression` | Inside `#if` condition |
| `CCC_PreprocessorDirective` | After `#` at line start |
| `CCC_Namespace` | After `namespace` |
| `CCC_MemberAccess` | General member access (covers dot and arrow) |
| `CCC_IncludedFile` | Inside `#include "..."` or `#include <...>` |
| `CCC_Attribute` | Inside an attribute specifier `[[...]]` |
| `CCC_NaturalLanguage` | In comments or string literals |
| `CCC_Recovery` | Error recovery: unknown context |

The `CodeCompletionContext` object also carries a `PreferredType` (the expected type, used for priority adjustment) and a `BaseType` (the object type for member-access contexts). `ScopeSpecifier` captures the nested-name-specifier seen before the completion point (e.g., `std::vector<int>::`).

### 38.3.2 Mapping Parser State to Context

`SemaCodeCompletion` uses `ParserCompletionContext` (an internal enum in `SemaCodeCompletion.h`) as an intermediate representation of where the parser is. `CodeCompleteOrdinaryName()` maps `PCC_Statement` to `CCC_Statement`, `PCC_Expression` to `CCC_Expression`, `PCC_Namespace` to `CCC_TopLevel`, etc. The mapping filters which completion candidates are collected: top-level context may include macros; member-access context only includes the members of the base type; type context excludes non-type declarations.

The context also controls what kind of candidates are produced at all. The `CodeCompleteOptions` struct (in `clang/include/clang/Sema/CodeCompleteOptions.h`) carries flags that the consumer can set before installing itself:

```cpp
struct CodeCompleteOptions {
  bool IncludeMacros = false;         // offer macro names
  bool IncludeCodePatterns = false;   // offer structural patterns (if/for/etc.)
  bool IncludeGlobals = true;         // include declarations from global scope
  bool IncludeNamespaceLevelDecls = true;
  bool IncludeBriefComments = false;  // attach documentation to results
  bool IncludeFixIts = false;         // include . -> correction fix-its
  bool LoadExternal = true;           // load decls from PCH/preamble
};
```

clangd sets `IncludeMacros`, `IncludeGlobals`, `IncludeBriefComments`, and `IncludeFixIts` to true when performing interactive completion. For signature-help requests (which present overload candidates rather than name completions), `ProcessOverloadCandidates()` is called instead of `ProcessCodeCompleteResults()`, providing `OverloadCandidate` objects that describe function signature, parameter types, and the index of the current argument.

### 38.3.3 Fix-It Completions

`IncludeFixIts` enables a special category of completion results: those that carry mandatory `FixItHint` edits alongside the completion text itself. The canonical example is when a user writes `ptr.member` but `ptr` is a pointer type — the completion for `member` would be accompanied by a fix-it that replaces `.` with `->`. Similarly, `ptr->member` on a non-pointer value offers fix-its changing `->` to `.`. These are enabled only when the consumer explicitly requests them, as they involve additional lookup overhead (trying both access operators on the same expression). The `FixIts` field in `CodeCompletionResult` ensures the edit is applied atomically with the completion insertion, so the fix-it range never overlaps the completion point.

---

## 38.4 Preamble Compilation

### 38.4.1 What the Preamble Is

The preamble is the contiguous initial region of a translation unit consisting entirely of preprocessing directives (`#include`, `#define`, `#pragma`, `#if`/`#endif`). It is the portion that can be precompiled once and reused across multiple parses of the same file as the main body changes. `ComputePreambleBounds()` (declared in [`clang/include/clang/Frontend/PrecompiledPreamble.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/PrecompiledPreamble.h#L42)) lexes the file's beginning and returns a `PreambleBounds` struct with a byte count and a flag indicating whether the preamble ends at the start of a line:

```cpp
PreambleBounds Bounds = ComputePreambleBounds(
    LangOpts, MainFileBuffer->getMemBufferRef(), /*MaxLines=*/0);
```

`MaxLines = 0` means scan the entire file (useful for files with large preambles). The scanner stops when it sees the first non-preprocessing token. For a file that begins with fifty `#include` lines, the preamble may cover several hundred kilobytes.

### 38.4.2 `PrecompiledPreamble::Build()`

`PrecompiledPreamble::Build()` is a static factory that compiles the preamble region and serializes the resulting `ASTContext` fragment as a PCH:

```cpp
llvm::ErrorOr<PrecompiledPreamble> MaybePreamble =
    PrecompiledPreamble::Build(
        *Invocation, MainFileBuffer.get(), Bounds,
        Diagnostics, VFS, PCHContainerOps,
        /*StoreInMemory=*/true, /*StoragePath=*/"",
        Callbacks);
```

With `StoreInMemory = true`, the PCH bytes are kept in a `MemoryBuffer` held by the `PrecompiledPreamble` object. With `StoreInMemory = false`, the PCH is written to a temporary file under `StoragePath`. In-memory storage is preferred for interactive tools where disk I/O latency would be unacceptable.

`PreambleCallbacks` is a virtual interface allowing the consumer to observe the preamble build: `BeforeExecute()` fires before parsing begins, `AfterExecute()` after the frontend action finishes (but before PCH emission), `AfterPCHEmitted()` when the writer has finished, and `HandleTopLevelDecl()` for each top-level declaration found in the preamble region.

### 38.4.3 Reuse Validation and `AddImplicitPreamble()`

Before reusing a cached preamble for a new parse, `CanReuse()` validates that the preamble bytes still match the current file content and that none of the included header files have changed (checked by size + modification time, or MD5 for in-memory buffers). The set of file hashes is recorded in `FilesInPreamble`, a `StringMap<PreambleFileHash>`. If a newly-created file now satisfies a previously-missing `#include`, the `MissingFiles` set detects this.

When reuse is valid, `AddImplicitPreamble()` configures the `CompilerInvocation` to treat the cached PCH as an implicit prefix:

```cpp
if (Preamble && Preamble->CanReuse(*CI, MainFileBuffer->getMemBufferRef(),
                                   NewBounds, *VFS)) {
  Preamble->AddImplicitPreamble(*CI, VFS, MainFileBuffer.get());
}
```

The precompiled headers are loaded by the subsequent `CompilerInstance` as if they had been parsed fresh, but in a fraction of the time — typically 10–50× faster for large codebases.

---

## 38.5 clangd Architecture Overview

### 38.5.1 clangd as an LSP Server

clangd is a Language Server Protocol implementation built as a standalone binary on top of Clang's library APIs. Its source lives entirely in `clang-tools-extra/clangd/` within the LLVM monorepo. The server communicates with editors (VS Code, Neovim, Emacs, etc.) via JSON-RPC messages over standard input/output, or optionally over a Unix domain socket.

The LSP protocol defines a set of request/notification methods — `textDocument/completion`, `textDocument/hover`, `textDocument/definition`, `textDocument/references`, `textDocument/semanticTokens/full`, and so on — that the server implements. Each method corresponds to a handler in `ClangdLSPServer`.

### 38.5.2 The Layered Architecture

clangd's architecture separates concerns across five layers:

```
Editor (VS Code, Neovim, …)
    │  JSON-RPC over stdin/stdout
    ▼
Transport (JSONTransport.cpp)
    │  llvm::json::Value messages
    ▼
ClangdLSPServer (ClangdLSPServer.{h,cpp})
    │  Parses LSP method names, deserializes Protocol types
    ▼
ClangdServer (ClangdServer.{h,cpp})
    │  Public API: addDocument, codeComplete, hover, references, …
    ▼
TUScheduler (TUScheduler.{h,cpp})
    │  Per-file ASTWorker threads, preamble threads
    ▼
ParsedAST / PreambleData
```

`Transport` is an abstract interface (in `Transport.h`) with `notify()`, `call()`, and `reply()` virtuals. `JSONTransport` (in `JSONTransport.cpp`) implements the standard JSON-RPC framing with `Content-Length` headers.

`ClangdLSPServer` owns a `ClangdServer` and binds LSP method names to lambdas that dispatch to `ClangdServer`'s async methods. It also implements LSP lifecycle management (initialize/shutdown) and routes client notifications.

`ClangdServer` (in [`clang-tools-extra/clangd/ClangdServer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/ClangdServer.h)) is the public API. It owns a `TUScheduler`, a `FileIndex` (the dynamic per-file symbol index), a `BackgroundIndex` (the static workspace-wide index), and a `GlobalCompilationDatabase`. Key options in `ClangdServer::Options` include `AsyncThreadsCount`, `BuildDynamicSymbolIndex`, `BackgroundIndex`, `ClangTidyProvider`, and `StaticIndex`.

---

## 38.6 `TUScheduler` and Asynchronous Compilation

### 38.6.1 Per-File `ASTWorker` Threads

`TUScheduler` (in [`clang-tools-extra/clangd/TUScheduler.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/TUScheduler.h)) maintains a map from file paths to `FileData` structs, each owning an `ASTWorker` running on a dedicated background thread. The number of concurrent worker threads is bounded by `Options::AsyncThreadsCount` (default: `getDefaultAsyncThreadsCount()`, which returns the hardware thread count capped at a reasonable maximum).

The three principal operations on a file are:

- **`update(File, ParseInputs, WantDiagnostics)`**: Enqueues a new version of the file content. The worker will eventually rebuild the preamble (if the preamble region changed) and the full AST.
- **`runWithAST(Name, File, Callback)`**: Enqueues a read operation that will execute `Callback` when the current AST is ready.
- **`runWithPreamble(Name, File, Consistency, Callback)`**: Enqueues a preamble read. With `PreambleConsistency::Stale`, the callback may receive an older preamble if the current one is still building.

### 38.6.2 Debouncing and Rebuild Scheduling

`DebouncePolicy` controls how long the scheduler waits after receiving an `update()` before actually rebuilding. This prevents rebuilding on every keystroke:

```cpp
struct DebouncePolicy {
  clock::duration Min = {};
  clock::duration Max = {};
  float RebuildRatio = 1;  // multiple of last rebuild time
  clock::duration compute(ArrayRef<clock::duration> History) const;
};
```

The computed debounce interval is `clamp(LastRebuildTime * RebuildRatio, Min, Max)`. For a file that takes 200 ms to parse, with `RebuildRatio = 2`, clangd waits up to 400 ms before triggering a rebuild. This adapts to machine speed automatically.

The `WantDiagnostics` enum governs diagnostic delivery: `Yes` forces diagnostics for this exact version, `No` suppresses them, and `Auto` (the most common) means "deliver diagnostics for this version or a subsequent one, within a bounded delay".

### 38.6.3 `ASTRetentionPolicy` and Memory Management

`ASTRetentionPolicy::MaxRetainedASTs` (default: 3) bounds how many parsed ASTs are kept in memory when idle. When a file's AST is needed but was evicted, it must be rebuilt. The LRU cache in `TUScheduler::ASTCache` handles this. Preamble data is separately retained; since preambles are expensive to build (and shared across multiple re-parses of the same file), they are kept in memory until the preamble region changes.

The `PreambleThrottler` interface allows controlling which preambles may be built concurrently — useful in distributed environments or on machines with constrained memory.

### 38.6.4 `ASTActionInvalidation` and Request Cancellation

Requests enqueued via `runWithAST()` can be tagged with `ASTActionInvalidation::InvalidateOnUpdate`, which causes the request to be silently dropped (reported as `CancelledError` to the callback) if a new `update()` arrives before the request executes. This is appropriate for background operations like background-indexed symbol collection, where the result of an older version is worthless once the file changes. In contrast, user-initiated operations like `hover` or `definition` use `NoInvalidation` and proceed to completion even if the file has been edited meanwhile (the result is based on the version that was in scope when the request arrived).

The `Deadline` mechanism allows callers to express a latency budget: if a preamble is not yet available by the deadline, `runWithPreamble()` with `PreambleConsistency::StaleOrAbsent` allows proceeding with no preamble at all, trading completeness for responsiveness. This matters for the first completion request after opening a large file where the preamble build may take several seconds.

---

## 38.7 `ParsedAST`

### 38.7.1 `ParsedAST::build()`

`clangd::ParsedAST` wraps the result of a full parse of one translation unit, including the deserialized preamble portion. It is constructed by `ParsedAST::build()`:

```cpp
std::optional<ParsedAST> AST = ParsedAST::build(
    Filename, Inputs, std::move(CI),
    CompilerInvocationDiags, Preamble);
```

Internally, `build()` constructs a `CompilerInstance`, installs custom `DiagnosticConsumer` and `ASTConsumer` implementations, runs `FrontendAction::Execute()`, and collects the results. If the preamble is non-null and `CanReuse()` validates it, `AddImplicitPreamble()` injects the PCH before parsing begins.

### 38.7.2 Contents of `ParsedAST`

`ParsedAST` holds:

- `ASTContext &getASTContext()`: The full AST context, containing the type system, declaration table, and identifier table.
- `Preprocessor &getPreprocessor()`: The preprocessor state after parsing (macro definitions, included files).
- `getLocalTopLevelDecls()`: An `ArrayRef<Decl *>` of top-level declarations from the main file only (excludes preamble deserialized decls).
- `getDiagnostics()`: An `ArrayRef<Diag>` of all diagnostics collected during this parse.
- `getIncludeStructure()`: An `IncludeStructure` representing the include graph of the main file (header file names, inclusion ranges).
- `getTokens()`: A `syntax::TokenBuffer` from `clang::tooling::syntax`, containing all spelled and expanded tokens for the main file. This is used for semantic tokens and rename operations.
- `getMacros()`: A `MainFileMacros` structure cataloging macro definitions and expansions within the main file, used for highlighting and navigation.

`ParsedAST` holds the `CompilerInstance` and `FrontendAction` alive (with `EndSourceFile()` deliberately not called), so the AST data structures outlive the `build()` call. Destruction of `ParsedAST` calls `EndSourceFile()` and releases all compiler resources.

---

## 38.8 Symbol Index

### 38.8.1 `SymbolIndex` Interface

The `SymbolIndex` abstract base class (in [`clang-tools-extra/clangd/index/Index.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/index/Index.h)) defines four query operations:

```cpp
class SymbolIndex {
public:
  // Fuzzy-match by unqualified name. Callback receives Symbol objects.
  virtual bool fuzzyFind(const FuzzyFindRequest &Req,
                         llvm::function_ref<void(const Symbol &)> Callback) const = 0;
  // Exact lookup by SymbolID.
  virtual void lookup(const LookupRequest &Req,
                      llvm::function_ref<void(const Symbol &)> Callback) const = 0;
  // Find all references to a set of symbol IDs.
  virtual bool refs(const RefsRequest &Req,
                    llvm::function_ref<void(const Ref &)> Callback) const = 0;
  // Find symbol relations (subclass, override, …).
  virtual void relations(const RelationsRequest &Req,
                         llvm::function_ref<void(const SymbolID &, const Symbol &)>
                             Callback) const = 0;
};
```

`FuzzyFindRequest` carries a query string, an optional set of namespace scopes (to restrict results), a `Limit`, a `RestrictForCodeCompletion` flag, and `ProximityPaths` that bias results toward symbols defined in files closer to the current file. `RefsRequest` filters by `RefKind` (read, write, declaration, definition, etc.) and supports an optional `Limit`.

### 38.8.2 `MemIndex` and `DexIndex`

Two concrete implementations serve different performance tradeoffs:

**`MemIndex`** (`IndexType::Light`) is a flat in-memory index that performs sequential scan for `fuzzyFind`. It is fast to build — O(n) over the symbol set — making it appropriate for per-file dynamic indexing where the symbol count is small and index freshness is paramount.

**`DexIndex`** (`IndexType::Heavy`) uses trigram-based inverted indexing for `fuzzyFind`, achieving sublinear query time on large symbol sets. It is the implementation used for the background index across the whole workspace. Building a `DexIndex` from n symbols is O(n log n) and uses several times more memory than `MemIndex`, but `fuzzyFind` on large codebases is orders of magnitude faster.

### 38.8.3 `FileIndex` and the Dynamic/Static Split

`FileIndex` (in [`clang-tools-extra/clangd/index/FileIndex.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/index/FileIndex.h)) is a `MergedIndex` that combines a **dynamic** (per-open-file) index and a **static** (background) index. The dynamic index is updated whenever `TUScheduler` notifies `ParsingCallbacks::onMainAST()` that a new AST is ready — the `FileIndex` re-indexes the symbols from the main file's AST and stores them in a `MemIndex` shard. The static index is built by `BackgroundIndex`.

`BackgroundIndex` crawls the `CompilationDatabase`, compiles each file in the workspace on low-priority background threads, extracts symbols and references, and stores them as serialized shards on disk (in `.clangd/index/` under the project root). On startup, existing shards are loaded; only files that have changed since the last shard write need reindexing.

`MergedIndex` implements a merge strategy: for `fuzzyFind()`, it queries both the dynamic and static index and deduplicates by `SymbolID`, preferring the more recent dynamic entry. For `refs()`, it unions the results from both indexes. This ensures that freshly edited code (which updates the dynamic index immediately) always appears in completion results, even before the background index has been updated.

The `Symbol` data structure (in `index/Symbol.h`) captures the information needed for index-driven features: `Name` (unqualified), `Scope` (the namespace prefix), `SymbolID`, `CanonicalDeclaration` (file/offset), optional `Definition` (file/offset), `Origin` (AST, index file, or merged), `SymbolInfo` (kind, language), `Signature` (function parameters formatted as a string), and `CompletionSnippetSuffix` (for snippet insertion). The `Ref` structure captures a reference: a file location, `RefKind` bitmask, and optional container symbol ID.

---

## 38.9 Hover, Go-to-Definition, and Find-References

### 38.9.1 Hover via `getHover()`

`clangd::getHover()` (in [`clang-tools-extra/clangd/Hover.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/Hover.h)) takes a `ParsedAST`, a `Position`, and an optional `SymbolIndex`:

```cpp
std::optional<HoverInfo> getHover(ParsedAST &AST, Position Pos,
                                  const format::FormatStyle &Style,
                                  const SymbolIndex *Index);
```

The function locates the token under the cursor using `SourceCode::getSourceRange()`, then dispatches based on the AST node found:

- For a `DeclRefExpr` or `MemberExpr`, it finds the referenced `NamedDecl` and extracts type, definition location, access specifier, documentation, template parameters, and the pretty-printed definition.
- For a macro expansion, it finds the `MacroInfo` and formats the macro body.
- For an expression result, it formats the expression's `QualType`.

`HoverInfo` contains structured fields: `NamespaceScope`, `LocalScope`, `Name`, `Kind` (from `index::SymbolKind`), `Documentation`, `Definition`, optional `Type`, `ReturnType`, `Parameters`, `TemplateParameters`, `Value` (for constexpr), `Size`, `Offset`, and `Padding`. The LSP client receives this as a `Hover` response with a `MarkupContent` body.

### 38.9.2 Go-to-Definition via `locateSymbolAt()`

`locateSymbolAt()` (in [`clang-tools-extra/clangd/XRefs.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/XRefs.h)) returns a `std::vector<LocatedSymbol>`. Each `LocatedSymbol` has a `PreferredDeclaration` (usually the header declaration) and an optional `Definition` (the `.cpp` file definition). The function first resolves the cursor's AST node to a `NamedDecl` using `FindTarget.cpp`'s `findDecl()` logic, which handles `UsingShadowDecl` (alias declarations), `TypedefDecl`, and template instantiations (following them back to the template definition).

For symbols not found in the current AST (defined in files not yet parsed), clangd falls back to `SymbolIndex::lookup()` using the symbol's `SymbolID`. The `SymbolID` is a 160-bit hash of the symbol's canonical qualified name and location, stable across parses. The index returns `Symbol` objects containing `CanonicalDeclaration` and `Definition` file/line locations.

Template instantiations require special handling: a `CXXMemberCallExpr` on a `vector<int>` resolves to a `CXXMethodDecl` that is an explicit instantiation, not the template. `locateSymbolAt()` detects this via `clangd::getDeclAtPosition()` and explicitly follows the instantiation to the template definition using `FunctionDecl::getPrimaryTemplate()` and `ClassTemplateSpecializationDecl::getSpecializedTemplate()`. This ensures go-to-definition on `v.push_back(x)` navigates to the `push_back` template in `<vector>`, not an invisible instantiation.

Macro definitions are resolved separately: `locateSymbolAt()` checks whether the cursor is on a macro-expansion token via `Preprocessor::getMacroDefinition()` and returns the `MacroInfo`'s spelling location as the definition.

The `textDocument/declaration` vs `textDocument/definition` LSP methods share this infrastructure but differ in priority: `declaration` returns `PreferredDeclaration` (the `.h` file forward declaration), while `definition` returns the `Definition` location if available, falling back to `PreferredDeclaration` if the definition is in an unparsed file.

### 38.9.3 Find-References via `findReferences()`

```cpp
ReferencesResult findReferences(ParsedAST &AST, Position Pos,
                                uint32_t Limit,
                                const SymbolIndex *Index,
                                bool AddContext = false);
```

`findReferences()` collects references in two passes:

1. **AST pass**: Walks the main file's AST using a `RecursiveASTVisitor` to find all `DeclRefExpr`, `MemberExpr`, `TypeLoc`, and macro expansion references to the target symbol.
2. **Index pass**: Issues a `SymbolIndex::refs()` query to find cross-file references in the background index.

Results from both passes are merged and deduplicated (same file + line + column). The `ReferencesResult` structure carries a `std::vector<ReferencesResult::Reference>`, each with a `Location` (URI + range) and a `RefKind` bitmask indicating whether it is a read, write, declaration, or definition.

---

## 38.10 Diagnostics and Fixes in clangd

### 38.10.1 `StoreDiags`

`StoreDiags` (in [`clang-tools-extra/clangd/Diagnostics.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/Diagnostics.h#L138)) is a `DiagnosticConsumer` that collects all `StoredDiagnostic`s during parsing and attaches them as `clangd::Diag` objects. `Diag` extends `DiagBase` with:

- `Notes`: a list of attached notes (sub-diagnostics)
- `Fixes`: a list of `Fix` structs, each containing an `llvm::StringRef` message and a `std::vector<TextEdit>` LSP edits
- `InsideMainFile`: whether the diagnostic comes from the main file
- `OpaqueData`: a JSON object for feature-module use

`FixItHint`s from Clang's diagnostic engine are converted to LSP `TextEdit` structures by `clangdDiagnosticToLSP()`, which translates `SourceRange`-based edits into UTF-16 line/column ranges (as required by the LSP specification).

### 38.10.2 `ClangdDiagnosticOptions`

```cpp
struct ClangdDiagnosticOptions {
  bool EmbedFixesInDiagnostics = false;   // LSP extension
  bool EmitRelatedLocations = false;       // use relatedInformation field
  bool SendDiagnosticCategory = false;     // "Semantic Issue", "Parse Issue"
  bool DisplayFixesCount = true;
};
```

With `EmbedFixesInDiagnostics = true`, clangd sends quick-fixes inline in the `textDocument/publishDiagnostics` notification as a non-standard `clangd.fixes` extension field, reducing the need for a separate `textDocument/codeAction` round-trip for common fix-its.

### 38.10.3 clang-tidy Integration

clangd runs clang-tidy checks as an additional `ASTConsumer` during parsing. The integration path is:

1. `ClangdServer` holds a `TidyProviderRef` (`ClangTidyProvider` in `Options`).
2. When building `ParsedAST`, `ParsedAST::build()` queries the `TidyProvider` for the per-file options.
3. A `ClangTidyASTConsumer` is created and installed alongside the main `ASTConsumer`.
4. Tidy diagnostics are collected by `StoreDiags` in the same pipeline as compiler diagnostics.

`TidyProvider` is a `unique_function<void(ClangTidyOptions &, StringRef filename)>`. The standard implementation (`provideClangTidyFiles()`) walks up the directory tree from the source file, loading `.clang-tidy` YAML files and merging their check lists according to the standard override semantics. `disableUnusableChecks()` filters out checks known to be incompatible with clangd's incremental parsing model (e.g., checks that require the full TU to be syntactically complete).

The LSP `CodeAction` response for a tidy diagnostic includes the fix-it as a `WorkspaceEdit` with one or more `TextEdit` operations, enabling editors to apply the fix without user code changes.

---

## 38.11 Semantic Tokens

### 38.11.1 `getSemanticHighlightings()`

```cpp
std::vector<HighlightingToken>
getSemanticHighlightings(ParsedAST &AST, bool IncludeInactiveRegionTokens);
```

This function (declared in [`clang-tools-extra/clangd/SemanticHighlighting.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/SemanticHighlighting.h)) performs two passes:

1. **AST visitor pass**: A `HighlightingTokenCollector` traverses the `ASTContext`, visiting each declaration and expression node. For each token of interest (an identifier that refers to a declaration), it assigns a `HighlightingKind` and bitmask of `HighlightingModifier`s.
2. **Macro pass**: The `MainFileMacros` structure from `ParsedAST` provides the set of macro expansion sites. These are annotated as `HighlightingKind::Macro` tokens.

### 38.11.2 `HighlightingKind` and `HighlightingModifier`

`HighlightingKind` maps to LSP semantic token types:

| `HighlightingKind` | LSP token type |
|---|---|
| `Variable` | `variable` |
| `LocalVariable` | `variable` |
| `Parameter` | `parameter` |
| `Function` | `function` |
| `Method` | `method` |
| `StaticMethod` | `method` |
| `Field` | `property` |
| `Class` | `class` |
| `Enum` | `enum` |
| `EnumConstant` | `enumMember` |
| `Typedef` | `type` |
| `Namespace` | `namespace` |
| `TemplateParameter` | `typeParameter` |
| `Concept` | `concept` |
| `Macro` | `macro` |
| `InactiveCode` | `comment` (via modifier) |

`HighlightingModifier` values include: `Declaration`, `Definition`, `Deprecated`, `Deduced` (auto), `Readonly`, `Static`, `Abstract`, `Virtual`, `DependentName`, `DefaultLibrary`, `UsedAsMutableReference`, `UsedAsMutablePointer`.

### 38.11.3 Token Resolution and LSP Encoding

`HighlightingToken` carries a `Range` (line/column in the main file), a `Kind`, and a modifier bitmask. After collection, `toSemanticTokens()` converts this to the LSP `SemanticToken` array format, which encodes tokens as delta-encoded (line-delta, startChar-delta, length, tokenType, tokenModifiers) tuples. `diffTokens()` computes the minimal edit between two token arrays for the `textDocument/semanticTokens/delta` response.

Macro expansions require special care: a macro-expanded token's `SourceLocation` is a macro ID, not a file location. The `SourceManager::getSpellingLoc()` call maps it to the written position in the main file. This means tokens inside multi-line macro expansions may be reported at the expansion site rather than the spelled location, which is the most useful position for the editor.

The `InactiveCode` kind is handled differently from all others — it annotates entire line ranges between `#if 0` / `#else` / `#endif` blocks that are excluded from compilation. clangd emits these as semantic tokens with a line-granular range rather than individual token ranges, and the LSP client typically renders the inactive region in a dimmed color. The `IncludeInactiveRegionTokens` parameter to `getSemanticHighlightings()` controls whether these tokens are included (they are relevant for semantic tokens but not for other uses of the function).

Token ranges are computed from the `syntax::TokenBuffer`, which records both the spelling location (where the token was written in the source) and the expansion location (where it appears after macro substitution). For non-macro tokens, these are identical. The semantic token range always uses the spelling location so that edits in the source file correctly target the token's position.

---

## 38.12 Compilation Database

### 38.12.1 `CompilationDatabase` Interface

`clang::tooling::CompilationDatabase` (in [`clang/include/clang/Tooling/CompilationDatabase.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Tooling/CompilationDatabase.h)) is the abstract base class for compile-command discovery:

```cpp
class CompilationDatabase {
public:
  static std::unique_ptr<CompilationDatabase>
      autoDetectFromSource(StringRef SourceFile, std::string &ErrorMessage);
  static std::unique_ptr<CompilationDatabase>
      autoDetectFromDirectory(StringRef SourceDir, std::string &ErrorMessage);

  virtual std::vector<CompileCommand>
  getCompileCommands(StringRef FilePath) const = 0;
  virtual std::vector<std::string> getAllFiles() const = 0;
  virtual std::vector<CompileCommand> getAllCompileCommands() const = 0;
};
```

`CompileCommand` carries `Directory` (the working directory for the compilation), `Filename` (absolute path to the source file), `CommandLine` (the full argument vector including the compiler executable), and `Output` (the object file path if known).

### 38.12.2 Concrete Implementations

**`JSONCompilationDatabase`** reads `compile_commands.json` files, which conform to the standard format originally defined for CMake's `CMAKE_EXPORT_COMPILE_COMMANDS`. The JSON file is an array of objects, each with `"directory"`, `"command"` (or `"arguments"` array), and `"file"` fields. `JSONCompilationDatabase::loadFromFile()` parses this file and builds an internal map from file paths to command vectors.

**`FixedCompilationDatabase`** is constructed with a single set of flags applied to all files. It is commonly constructed by parsing the `--` separator in a command line:

```cpp
auto CDB = FixedCompilationDatabase::loadFromCommandLine(argc, argv, ErrorMsg);
```

**`InferredCompilationDatabase`** (in the `tooling` namespace) guesses flags from file extensions and common patterns when no explicit database exists.

### 38.12.3 `GlobalCompilationDatabase` in clangd

clangd wraps these into its own `GlobalCompilationDatabase` interface, with `DirectoryBasedGlobalCompilationDatabase` as the primary implementation. On each `getCompileCommand()` call, it searches for `compile_commands.json` in parent directories of the requested file. The discovery path in order:

1. Look for `compile_commands.json` in each ancestor directory of the source file.
2. Look for `compile_flags.txt` (a plain-text file with one flag per line) in the same set of directories.
3. Fall back to `getFallbackCommand()`, which constructs a minimal clang invocation for the file based on its extension and the system default include paths.

The `CommandChanged` event fires whenever the database changes (e.g., CMake re-runs and updates `compile_commands.json`), causing clangd to invalidate affected file's parse results and re-queue them for rebuilding.

### 38.12.4 `clang-scan-deps` and Modular Databases

For projects using C++20 modules, `clang-scan-deps` (a separate tool in `clang/tools/clang-scan-deps/`) is used to pre-compute module dependency graphs. clangd integrates this via `ScanningProjectModules`, which uses `clang-scan-deps`'s programmatic API to compute inter-module dependencies and feed them to `ModulesBuilder` for ordered preamble/module compilation.

---

## 38.13 The LSP Message Flow: A Complete Example

To ground the architecture, consider a `textDocument/completion` request. The LSP JSON-RPC request arrives:

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "method": "textDocument/completion",
  "params": {
    "textDocument": {"uri": "file:///home/user/project/main.cpp"},
    "position": {"line": 22, "character": 8}
  }
}
```

`JSONTransport` reads the Content-Length framed message, parses the JSON, and dispatches to `ClangdLSPServer::onCompletion()`. This calls `ClangdServer::codeComplete()`:

```cpp
void ClangdServer::codeComplete(PathRef File, Position Pos,
                                const clangd::CodeCompleteOptions &Opts,
                                Callback<CodeCompleteResult> CB) {
  // Queue a preamble read first (faster path)
  Scheduler.runWithPreamble("CodeCompletion", File,
                            TUScheduler::Stale,
                            [=](Expected<InputsAndPreamble> IP) {
    // ... then run full completion with AST
  });
}
```

The `TUScheduler` dispatches `runWithPreamble()` on the file's worker thread. `clangd::CodeComplete()` in `CodeComplete.cpp` sets up a `CompilerInvocation` with the completion position, installs a `CodeCompleteConsumer` subclass, runs the parse with the cached preamble via `AddImplicitPreamble()`, and collects results. The consumer combines:

- AST-based candidates from Sema's completion callbacks
- Index-based candidates from `SymbolIndex::fuzzyFind()` (for global symbols not yet visible in the partial parse)

Results are ranked by a learned scoring model (using a `DecisionForest` in `quality/CompletionModel.cpp`), then serialized to the LSP `CompletionList` format and returned via `JSONTransport::reply()`.

The LSP response for a completion takes the form:

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "result": {
    "isIncomplete": false,
    "items": [
      {
        "label": "push_back",
        "kind": 2,
        "detail": "void (const value_type &value)",
        "insertText": "push_back(${1:value})",
        "insertTextFormat": 2,
        "sortText": "3push_back",
        "filterText": "push_back"
      }
    ]
  }
}
```

The `insertTextFormat: 2` signals snippet syntax; `${1:...}` marks tab stops. `sortText` carries a priority-derived prefix so editors sort results correctly. `filterText` allows editors to filter the list as the user continues typing. Each `CompletionItem` may carry a `textEdit` field with the exact `TextEdit` to apply (including any fix-it edits for operator corrections), making the insertion unambiguous even when the cursor is in the middle of an existing identifier.

## 38.14 Code Completion Ranking and Quality Scoring

### 38.14.1 Heuristic vs. Learned Ranking

Clang's base priority system (the `CCP_*` constants) provides a heuristic ranking suitable for `PrintingCodeCompleteConsumer` and simple tooling. clangd supplements this with a richer scoring model. The `Quality` module in `clang-tools-extra/clangd/Quality.cpp` defines two complementary components:

**`SymbolQuality`** describes intrinsic properties of the symbol independent of context: whether it was referenced recently (from index data), the number of references in the codebase, whether it comes from the current project vs. a standard library, whether it is deprecated, and its symbol kind.

**`SymbolRelevance`** describes how well the symbol fits the current completion context: whether the symbol's name matches the query prefix, distance from the current file (symbols in the same directory score higher), whether the symbol's type matches the expected type, scope proximity (local > class member > global), and whether the symbol is already imported (or would require an `#include` to use).

### 38.14.2 Decision Forest and Feature Encoding

clangd uses a machine-learned `DecisionForest` (in `quality/CompletionModel.cpp`, auto-generated from training data) to combine the `SymbolQuality` and `SymbolRelevance` features into a single float score. The model takes about 30 features and produces a score in [0, 1], with higher scores indicating more relevant completions. The decision forest is evaluated at O(log n) depth per tree and is intentionally fast — it runs on potentially hundreds of candidates per completion request.

The features fed to the forest include: `SymbolKind` (encoded as an integer), `NumReferences` (log-transformed), `IsInMainFile`, `ScopeProximity` (the minimum number of namespace boundaries to cross), `FrecencyScore` (a time-decaying usage frequency from the index), and the string similarity score between the query text and the completion label. This scoring approach means that a rarely-used global function is automatically pushed below a frequently-used local variable of matching type, without the developer needing to set explicit priorities.

### 38.14.3 Header Insertion

When a completion candidate requires including a header not yet present in the current file, clangd generates an `additionalTextEdits` field in the `CompletionItem` with a `TextEdit` that inserts the `#include` directive at the appropriate location. The header path is determined by consulting `CanonicalIncludes` (which maps symbols to their canonical public headers, e.g., mapping `std::vector` to `<vector>` rather than implementation headers) and `IncludeInserter` (which finds the right insertion point respecting existing include ordering and grouping).

---

## Chapter Summary

- Clang code completion is driven by the `tok::code_completion` synthetic token injected by the lexer at the `-code-completion-at` position; the parser dispatches to over 80 `SemaCodeCompletion` methods based on grammar context.
- `CodeCompletionResult` classifies results as `RK_Declaration`, `RK_Keyword`, `RK_Macro`, or `RK_Pattern`; priority scoring uses a numeric scale where lower is better, with adjustments for scope locality, type matching, and deprecation.
- `CodeCompletionString` encodes insertion text as an ordered sequence of typed chunks; `CodeCompletionAllocator` (a bump allocator) and `CodeCompletionBuilder` handle arena allocation.
- `CodeCompletionContext::Kind` encodes the grammar position of the completion point; the `PreferredType` and `BaseType` fields enable type-directed priority adjustment.
- `PrecompiledPreamble` compiles and caches the `#include` header region as a PCH; `ComputePreambleBounds()` finds the boundary, `Build()` creates the PCH, `CanReuse()` validates staleness, and `AddImplicitPreamble()` injects it into subsequent compilations for 10–50× speedup.
- clangd's architecture layers `Transport` → `ClangdLSPServer` → `ClangdServer` → `TUScheduler` → `ParsedAST`; each layer has a clear responsibility boundary.
- `TUScheduler` provides per-file `ASTWorker` threads with `update()`, `runWithAST()`, and `runWithPreamble()` operations; `DebouncePolicy` prevents over-eager rebuilds; `ASTRetentionPolicy` bounds memory use.
- `ParsedAST` holds `ASTContext`, `Preprocessor`, `syntax::TokenBuffer`, collected `Diag`s, and `LocalTopLevelDecls`; it keeps the `CompilerInstance` alive.
- `SymbolIndex` provides `fuzzyFind()`, `lookup()`, `refs()`, and `relations()`; `MemIndex` (fast build) and `DexIndex` (trigram-indexed, fast query) are concrete implementations; `FileIndex` merges per-file dynamic and background-indexed static indexes.
- `getHover()` extracts `HoverInfo` from AST nodes and the symbol index; `locateSymbolAt()` handles `UsingShadowDecl` and template instantiations for go-to-definition; `findReferences()` combines AST and index passes.
- `StoreDiags` collects `clangd::Diag` with attached `Fix` structs; clang-tidy runs as an additional `ASTConsumer`; `TidyProvider` resolves per-file `.clang-tidy` configuration.
- `getSemanticHighlightings()` classifies tokens by `HighlightingKind` and `HighlightingModifier`; `toSemanticTokens()` delta-encodes for LSP; macro expansion sites are mapped to spelling locations via `SourceManager::getSpellingLoc()`.
- `CompilationDatabase` and `GlobalCompilationDatabase` abstract compile-flag discovery; `JSONCompilationDatabase` reads `compile_commands.json`; clangd searches ancestor directories and falls back to inferred flags.

---

## Cross-References

- [Chapter 29 — SourceManager, FileEntry, and SourceLocation](../part-05-clang-frontend/ch29-sourcemanager-fileentry-sourcelocation.md): preamble boundary computation uses `SourceManager` to track file positions.
- [Chapter 30 — The Diagnostic Engine](../part-05-clang-frontend/ch30-the-diagnostic-engine.md): `StoreDiags` is a `DiagnosticConsumer`; `FixItHint` conversion to LSP `TextEdit` builds on the engine's representation.
- [Chapter 31 — The Lexer and Preprocessor](../part-05-clang-frontend/ch31-the-lexer-and-preprocessor.md): `tok::code_completion` token generation and the lexer's position-matching logic.
- [Chapter 32 — The Parser](../part-05-clang-frontend/ch32-the-parser.md): parser dispatch to `SemaCodeCompletion` methods at completion token sites.
- [Chapter 36 — The Clang AST in Depth](../part-05-clang-frontend/ch36-the-clang-ast-in-depth.md): `ASTContext`, `NamedDecl`, `DeclContext`, and `QualType` used throughout code completion and clangd features.
- [Chapter 46 — libtooling and AST Matchers](../part-07-clang-multilang/ch46-libtooling-and-ast-matchers.md): `clang::tooling::CompilationDatabase` base class and the tooling infrastructure clangd builds on.
- [Chapter 47 — clangd, clang-tidy, clang-format, clang-refactor](../part-07-clang-multilang/ch47-clangd-tidy-format-refactor.md): the broader clangd feature set (inlay hints, rename, code actions) beyond the foundations covered here.

## Reference Links

- [`clang/include/clang/Sema/CodeCompleteConsumer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/CodeCompleteConsumer.h)
- [`clang/include/clang/Sema/SemaCodeCompletion.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/SemaCodeCompletion.h)
- [`clang/include/clang/Frontend/PrecompiledPreamble.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/PrecompiledPreamble.h)
- [`clang-tools-extra/clangd/ClangdServer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/ClangdServer.h)
- [`clang-tools-extra/clangd/TUScheduler.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/TUScheduler.h)
- [`clang-tools-extra/clangd/ParsedAST.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/ParsedAST.h)
- [`clang-tools-extra/clangd/index/Index.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/index/Index.h)
- [`clang-tools-extra/clangd/index/FileIndex.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/index/FileIndex.h)
- [`clang-tools-extra/clangd/index/Background.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/index/Background.h)
- [`clang-tools-extra/clangd/Hover.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/Hover.h)
- [`clang-tools-extra/clangd/XRefs.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/XRefs.h)
- [`clang-tools-extra/clangd/Diagnostics.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/Diagnostics.h)
- [`clang-tools-extra/clangd/SemanticHighlighting.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/SemanticHighlighting.h)
- [`clang-tools-extra/clangd/TidyProvider.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/TidyProvider.h)
- [`clang-tools-extra/clangd/Transport.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/Transport.h)
- [`clang/include/clang/Tooling/CompilationDatabase.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Tooling/CompilationDatabase.h)


---

@copyright jreuben11
