# Chapter 47 — clangd, clang-tidy, clang-format, and clang-refactor

*Part VII — Clang as a Multi-Language Compiler*

The LLVM project's developer tooling ecosystem spans far beyond the compiler itself. clangd transforms Clang into a persistent language server that answers real-time LSP queries. clang-tidy provides an extensible, AST-matcher-driven linting and automated fix-it framework. clang-format applies a penalty-based formatting engine capable of handling any style guide with submillisecond latency on most files. clang-refactor exposes programmatic source transformations — extraction and renaming — through a rule-based refactoring API. Together these four tools share Clang's full semantic model: they parse with the same `Sema`, analyze with the same AST, and express edits with the same `tooling::Replacement` type. Understanding their internal architecture enables both effective use and meaningful extension.

---

## 47.1 clangd: The LSP Server

### 47.1.1 Server Architecture: ClangdLSPServer, ClangdServer, TUScheduler

clangd lives in `clang-tools-extra/clangd/` in the LLVM monorepo. Its public entry point is `ClangdLSPServer` ([ClangdLSPServer.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/ClangdLSPServer.h)), which owns the JSON-RPC transport layer — reading `Content-Length`-prefixed messages from stdin and dispatching them by method name. `ClangdLSPServer` delegates semantic work downward to `ClangdServer` ([ClangdServer.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/ClangdServer.h)), which coordinates the index, the preamble cache, and the translation-unit scheduler.

`TUScheduler` ([TUScheduler.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/TUScheduler.h)) is the core concurrency engine. It maintains a per-file worker thread pool and a queue of pending AST operations. Each file has an `ASTWorker` object that owns:

- The latest preamble (`PrecompiledPreamble`), rebuilt asynchronously when included headers change
- A cached `ParsedAST` for the current file contents
- A `CompletionRequest` queue for latency-sensitive operations that skip the full AST rebuild

Scheduling priorities distinguish *urgent* requests (completion, hover, diagnostics on keystroke) from *non-urgent* background tasks (index update, semantic token push). The scheduler uses a `Semaphore`-based throttle to avoid spawning more concurrent parses than there are hardware threads.

```
Editor keystroke
      │
      ▼
ClangdLSPServer::onNotification("textDocument/didChange")
      │
      ▼
TUScheduler::update(path, contents, WantDiagnostics)
      │ enqueues on ASTWorker thread
      ▼
ASTWorker::rebuild()
      │ reuse preamble if headers unchanged
      ▼
ParsedAST::build(Inputs, Preamble, &Diags)
      │
      ▼
ClangdServer::onMainAST(path, AST, publish diagnostics)
```

`ParsedAST` wraps a `CompilerInstance` that has completed parsing and semantic analysis. It stores the `ASTContext`, `Preprocessor`, a list of `Diag`, and a `MainFileMacros` table. All higher-level clangd features — hover, go-to-definition, inlay hints — receive a `ParsedAST &` reference and operate on this fully-built AST.

### 47.1.1b Hover, Go-to-Definition, and Semantic Tokens

Hover (`textDocument/hover`) calls `getHover(ParsedAST, Position)` in `clang-tools-extra/clangd/Hover.cpp`. The function calls `getDeclAtPosition()` to find the `NamedDecl` or expression under the cursor, then constructs a `HoverInfo` struct containing:

- The declaration's pretty-printed signature (via `PrintingPolicy`)
- Documentation extracted from the associated Doxygen comment by `RawComment::getFormattedText()`
- For template instantiations, the template arguments substituted in
- For `auto` variables, the deduced type expanded

Go-to-definition (`textDocument/definition`) and go-to-declaration use `locateCursorAt()`, which returns a list of `LocatedSymbol` objects. For virtual method calls, clangd returns all overriding definitions, not just the static type's definition. For `#include` directives, it navigates to the included file.

Semantic tokens (`textDocument/semanticTokens/full`) are computed by `getSemanticHighlights(ParsedAST)` in `clang-tools-extra/clangd/SemanticHighlighting.cpp`. An `HighlightingVisitor` traverses the AST and assigns token kinds (namespace, type, typeParameter, function, method, variable, parameter, macro, modifier) to each token position. The LSP response encodes tokens as a delta-encoded array of `[deltaLine, deltaStartChar, length, tokenType, tokenModifiers]` tuples for compact transmission.

Diagnostics are published via `textDocument/publishDiagnostics`. clangd collects diagnostics from three sources: Clang's own `DiagnosticsEngine` (syntax and semantic errors), the `ClangTidyProvider` (clang-tidy checks configured in `~/.config/clangd/config.yaml` or `.clang-tidy` files), and `IncludeCleanerProvider` (unused `#include` warnings from `clang-include-cleaner`). These are merged in `ClangdDiagnosticConsumer` and sent as a single `publishDiagnostics` notification per file per parse.

### 47.1.2 Inlay Hints

Inlay hints (`textDocument/inlayHint`) are implemented in `clang-tools-extra/clangd/InlayHints.cpp`. The `inlayHints()` function constructs an `InlayHintVisitor` — an `RecursiveASTVisitor` subclass — that traverses the AST and emits `InlayHint` structs:

- **Parameter names**: When a call expression passes unnamed arguments, the visitor checks whether the callee's `ParmVarDecl` names are meaningful (length > 1, not a single character) and emits a label like `bufSize:` just before each argument token.
- **Deduced types**: For `auto`-declared variables, the hint displays the deduced type via `QualType::getAsString()` with a custom print policy that suppresses elaborated-type prefixes.
- **Trailing return types**: When a lambda has an inferred return type, a hint showing `-> T` appears at the lambda body opening brace.

The response encodes each hint as:

```json
{
  "position": { "line": 12, "character": 30 },
  "label": "bufSize:",
  "kind": 2,
  "paddingRight": true
}
```

Kind 1 is `Type`, kind 2 is `Parameter`. The `paddingRight`/`paddingLeft` booleans control whether the editor inserts spacing around the label.

### 47.1.3 Call Hierarchy and Type Hierarchy

Call hierarchy (`textDocument/prepareCallHierarchy`, `callHierarchy/incomingCalls`, `callHierarchy/outgoingCalls`) is implemented in `clang-tools-extra/clangd/CallHierarchy.cpp`. `prepareCallHierarchy` takes a cursor position, resolves it to a `NamedDecl` via `getDeclAtPosition()`, and returns a `CallHierarchyItem` containing the decl's name, kind, URI, and selection range.

`callHierarchy/incomingCalls` queries the `SymbolIndex` for all references to the item's canonical `SymbolID`, then groups them by enclosing function. Each caller becomes a `CallHierarchyIncomingCall` with the call site ranges. The implementation uses `index::indexTopLevelDecls()` on background-indexed TUs to build reference lists.

Type hierarchy (`textDocument/prepareTypeHierarchy`) resolves to a `CXXRecordDecl`, then walks `bases()` for supertypes and the index for `isDerivedFrom` queries for subtypes. The `TypeHierarchyItem` carries a `data` field (a `TypeHierarchyData` serialized to JSON) that encodes the canonical symbol ID, enabling the client to retrieve hierarchies lazily.

### 47.1.4 Rename

Rename (`textDocument/rename`) is implemented via `PrepareRenameAction` and `RenameAction` in `clang-tools-extra/clangd/refactor/Rename.cpp`. The flow:

1. `prepareRename(position)` calls `locateCursorAt()` to find the `NamedDecl` under the cursor. It checks for rename blockers: built-in types, anonymous structs, and names used in macro expansions (which are disallowed in single-TU mode).
2. `rename(position, newName)` calls `findOccurrences()`, which enumerates all references within the current TU using `index::indexTopLevelDecls()` plus preprocessor macro token locations. For cross-TU renames, it queries the `SymbolIndex::refs()` method, which returns references from the background index.
3. Each found location is converted to a `tooling::Replacement(SourceManager, CharSourceRange, newName)`. The replacements are grouped by file URI and returned as a `WorkspaceEdit`.

Multi-file rename depends on index completeness. If a symbol appears in a TU not yet background-indexed, clangd warns that the rename may be incomplete. Cross-TU rename through macro expansions is explicitly blocked because the expanded token extent cannot be reliably mapped back to a replacement.

### 47.1.5 Workspace Symbol and BackgroundIndex

`workspace/symbol` queries are served by `BackgroundIndex` ([BackgroundIndex.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/index/Background.h)). The background index runs a thread pool that parses each TU in the `compile_commands.json` database, extracts `Symbol` and `Ref` records, serializes them to `*.idx` shard files in a configurable storage directory (default: `~/.cache/clangd/index/`), and merges them into an in-memory `MemIndex`.

The shard format uses LLVM's bitstream serialization. Each shard records:
- Symbol table: name, qualified name, kind, declaration location, definition location, signature, documentation
- Reference table: `SymbolID` → list of `(fileURI, range, kind)` tuples
- Relation table: base/derived type relations

`workspace/symbol` calls `MemIndex::fuzzyFind(FuzzyFindRequest)`, which uses trigram-based inverted indexes for sub-linear lookup. The `FuzzyMatcher` scores results by edit distance between the query and symbol name, breaking ties by symbol kind and definition availability.

### 47.1.6 Offline Diagnosis and Remote Index

`clangd --check=file.cpp` runs a single-file parse without a language-server transport, printing all diagnostics and exiting. It is useful for CI verification that a file parses cleanly under clangd's interpretation. The flag accepts an optional `--check-lines=N-M` to restrict diagnostics to a range.

`--remote-index-address=host:port` connects clangd to a gRPC-based remote index server (implemented in `clang-tools-extra/clangd/index/remote/`). The remote protocol (`remote.proto`) exposes `LookupSymbols`, `FuzzyFind`, and `Refs` RPCs. This enables large codebases to pre-build a shared index on a build server, letting each developer's clangd skip local background indexing entirely.

### 47.1.7 Completion Engine in clangd

While [Chapter 38](../part-05-clang-frontend/ch38-code-completion-and-clangd-foundations.md) covered the Clang completion engine, clangd adds a significant post-processing layer in `clang-tools-extra/clangd/CodeComplete.cpp`. The `codeComplete()` function invokes Clang's completion via `SemaCodeCompletion`, collects `CodeCompletionResult` items, and passes them through several enhancement stages:

1. **Index augmentation**: `ClangdServer::codeComplete()` merges AST-based results with results from `SymbolIndex::fuzzyFind()`, adding global symbols not visible in the current scope (e.g., functions defined in other TUs that would require a new `#include`).
2. **Fuzzy scoring**: `FuzzyMatcher` scores each candidate against the typed prefix using a DFA-based substring matching algorithm. The score accounts for case, word boundaries, and prefix matching.
3. **Include insertion**: For index-sourced completions, clangd computes the `#include` directive needed to make the symbol visible and embeds it as an `additionalTextEdit` in the LSP `CompletionItem`.
4. **Snippet generation**: Function completions generate LSP snippet syntax (`${1:paramName}`) using the callee's `ParmVarDecl` names when available.

The result is ranked by a linear combination of relevance (how well the symbol matches the query) and quality (symbol kind, definition availability, usage frequency from index statistics). The top `clangd.completion.limit` (default: 100) items are returned.

---

## 47.2 clang-tidy: Architecture and Configuration

### 47.2.1 Core Classes

clang-tidy lives in `clang-tools-extra/clang-tidy/`. The central class is `ClangTidyCheck` ([ClangTidyCheck.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clang-tidy/ClangTidyCheck.h)), an abstract base that implements `ast_matchers::MatchFinder::MatchCallback`. Subclasses override three methods:

```cpp
void registerMatchers(ast_matchers::MatchFinder *Finder) override;
void check(const ast_matchers::MatchFinder::MatchResult &Result) override;
void registerPPCallbacks(const SourceManager &SM,
                         Preprocessor *PP,
                         Preprocessor *ModuleExpanderPP) override;
```

`registerMatchers` is called once per TU to add AST matchers. `check` is called for each match. `registerPPCallbacks` is used by checks that need to observe macro expansions or include directives before the AST is built.

`ClangTidyContext` ([ClangTidyDiagnosticConsumer.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clang-tidy/ClangTidyDiagnosticConsumer.h)) holds the `DiagnosticsEngine` and collects `ClangTidyError` objects. It provides the `diag()` entry point used by checks and manages diagnostic deduplication and suppression (via `NOLINT` comments). It also exposes `getCurrentFile()` and `getLangOpts()` for checks that need file or language context.

`ClangTidyASTConsumerFactory` ([ClangTidy.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clang-tidy/ClangTidy.h)) is the libtooling `FrontendAction`-side entry point. It creates a `MultiplexConsumer` that combines the `MatchFinder`'s `ASTConsumer` with each registered check's consumer. `ClangTidyCheckFactories` maintains the registry mapping check names to factory functions.

### 47.2.2 Configuration and Options

clang-tidy reads configuration from `.clang-tidy` YAML files, discovered by walking from the file's directory toward the filesystem root. The `OptionsProvider` hierarchy has three levels: `DefaultOptionsProvider`, `FileOptionsProvider` (reading `.clang-tidy`), and `OverrideOptionsProvider` (for command-line `--config=` JSON blobs). Options are merged with lowest-level `.clang-tidy` files winning.

A `.clang-tidy` file example:

```yaml
---
Checks: '-*,modernize-use-override,readability-identifier-naming,performance-*'
WarningsAsErrors: 'modernize-*'
HeaderFilterRegex: '.*'
CheckOptions:
  - key:   readability-identifier-naming.ClassCase
    value: CamelCase
  - key:   readability-identifier-naming.VariableCase
    value: camelBack
  - key:   modernize-use-default-member-init.UseAssignment
    value: '1'
```

`HeaderFilterRegex` is a POSIX regex. Diagnostics whose file path does not match are suppressed. Setting it to `'.*'` enables diagnostics in all headers, which is appropriate for projects that own all their headers.

The `--checks` command-line flag uses the same glob syntax as `Checks` in the YAML: a comma-separated list where a leading `-` removes a check. Glob wildcards match module names. `'-*,modernize-use-override'` first disables all checks then enables only the named one.

### 47.2.3 Check Module Registration

Checks are grouped into modules (subdirectories of `clang-tidy/`): `modernize/`, `readability/`, `performance/`, `bugprone/`, `cppcoreguidelines/`, `clang-analyzer/`, and more. Each module provides a `ClangTidyModule` subclass that overrides `addCheckFactories(ClangTidyCheckFactories &)`:

```cpp
class ModernizeModule : public ClangTidyModule {
public:
  void addCheckFactories(ClangTidyCheckFactories &Factories) override {
    Factories.registerCheck<UseOverrideCheck>("modernize-use-override");
    Factories.registerCheck<UseNullptrCheck>("modernize-use-nullptr");
    // ...
  }
};
static ClangTidyModuleRegistry::Add<ModernizeModule>
    X("modernize", "Adds C++ modernization checks.");
```

`ClangTidyModuleRegistry` uses LLVM's `Registry<ClangTidyModule>` plugin mechanism, so out-of-tree modules loaded via `-load=` can register new checks without recompiling clang-tidy. Check names follow the `module-CheckName` convention: the module prefix is separated from the check name by a hyphen, with subsequent words also hyphen-separated (`modernize-use-override`, not `modernize-useOverride`).

### 47.2.4 Execution Pipeline

When `clang-tidy` is invoked on a single file, the execution path is:

1. `ClangTidyMain()` in `clang-tools-extra/clang-tidy/tool/ClangTidyMain.cpp` parses command-line arguments and builds a `ClangTidyOptions` by merging command-line overrides with file-discovered `.clang-tidy` options via `ConfigOptionsProvider`.
2. `runClangTidy(Context, Options, Files, CompileDB, ApplyFixes, EnableCheckProfile)` constructs a `ClangTool` ([Chapter 46](../part-07-clang-multilang/ch46-libtooling-and-ast-matchers.md)) with the compilation database and invokes it with a `ClangTidyASTConsumerFactory` as the action.
3. For each TU, `ClangTidyASTConsumerFactory::newASTConsumer()` instantiates all enabled checks (filtered by the glob expression), calls `registerMatchers()` and `registerPPCallbacks()` on each, and returns a `MultiplexConsumer`.
4. After the AST is built, the `MatchFinder` fires matches in AST traversal order, invoking each check's `check()` callback.
5. After the full TU is processed, `ClangTidyDiagnosticConsumer::finish()` deduplicates diagnostics by location and suppresses those matching `NOLINT` or `NOLINTBEGIN`/`NOLINTEND` comment regions.
6. If `--fix` is specified, `tooling::applyAllReplacements()` applies the collected fix-its. `clang-apply-replacements` is used when running in parallel mode to merge replacements from multiple workers before applying.

Profiling per-check performance is available via `--enable-check-profile`, which reports CPU time per check after each TU.

---

## 47.3 Key Check Families

### 47.3.1 modernize-\*

The `modernize` module targets C++11/14/17 migration:

| Check | Action |
|-------|--------|
| `modernize-use-override` | Adds `override`/`final` to overriding virtual functions |
| `modernize-use-nullptr` | Replaces `0` and `NULL` with `nullptr` in pointer contexts |
| `modernize-avoid-bind` | Replaces `std::bind` with lambdas |
| `modernize-use-auto` | Infers `auto` for iterator declarations and `new` expressions |
| `modernize-use-emplace` | Replaces `push_back(T(...))` with `emplace_back(...)` |
| `modernize-loop-convert` | Converts index-based `for` loops to range-for |
| `modernize-use-default-member-init` | Moves constructor member init to in-class init |
| `modernize-use-nodiscard` | Adds `[[nodiscard]]` to functions returning non-void values |

`modernize-loop-convert` in `clang-tools-extra/clang-tidy/modernize/LoopConvertCheck.cpp` is instructive: it matches `for` loops with a three-part structure whose condition compares an index variable against `container.end()` or `container.size()`, verifies that the loop body only accesses the container through the index (using a `DeclUsageVisitor`), and emits a replacement that rewrites the loop header and all index accesses.

### 47.3.2 readability-\*

`readability-identifier-naming` is the most configurable check in clang-tidy. It accepts per-entity-kind case style options (`CamelCase`, `camelBack`, `lower_case`, `UPPER_CASE`, `CamelBack_Case`, etc.) and validates every name declaration against the configured style, emitting a fix-it that renames the identifier everywhere in the TU.

`readability-const-return-type` removes spurious `const` qualifiers from non-reference, non-pointer return types (e.g., `const int foo()` → `int foo()`). `readability-avoid-const-params-in-decls` removes top-level `const` from function declaration parameter types, since they have no semantic effect in declarations (only in definitions).

### 47.3.3 performance-\*

`performance-avoid-unnecessary-copy-initialization` detects `auto x = container[i];` where `container[i]` returns a reference and emits `const auto& x = container[i];`. `performance-move-const-arg` detects `std::move` applied to a const expression or a trivially-copyable type and removes the move. `performance-unnecessary-value-param` detects by-value function parameters that are never moved or modified and suggests converting them to const references.

### 47.3.4 bugprone-\*

`bugprone-use-after-move` is among the most sophisticated checks. It uses a data-flow analysis (`UseAfterMoveFinder` in `clang-tools-extra/clang-tidy/bugprone/UseAfterMoveCheck.cpp`) that traverses the CFG of each function body, tracking which variables have been moved-from and warning when a moved-from variable is accessed on any reachable path before reinitialization.

`bugprone-dangling-handle` detects `std::string_view` (and similar handle types) constructed from temporaries: `std::string_view sv = std::string("hello");` is caught because the temporary `std::string` is destroyed at the end of the expression.

`bugprone-suspicious-memset-usage` catches `memset(buf, sizeof(buf), 0)` — transposed argument order — and similar common memset mistakes.

### 47.3.5 cppcoreguidelines-\* and clang-analyzer-\*

The `cppcoreguidelines` module implements checks from the C++ Core Guidelines. Several are aliases of other checks: `cppcoreguidelines-avoid-magic-numbers` aliases `readability-magic-numbers`. `cppcoreguidelines-pro-type-reinterpret-cast` simply warns on any use of `reinterpret_cast`.

`clang-analyzer-*` checks are wrappers around the Clang Static Analyzer's checker infrastructure (see [Chapter 45](../part-07-clang-multilang/ch45-the-static-analyzer.md)). Running `clang-tidy` with `clang-analyzer-core.DivideZero` is equivalent to running the static analyzer's `core.DivideZero` checker, but the results are reported through clang-tidy's diagnostic pipeline, enabling YAML configuration and suppression via `NOLINT`.

---

## 47.4 Writing a clang-tidy Check

### 47.4.1 Skeleton

A new check in module `mymodule` named `mymodule-no-raw-new` requires three files:

```
clang-tools-extra/clang-tidy/mymodule/
  NoRawNewCheck.h
  NoRawNewCheck.cpp
  MymoduleModule.cpp
```

The header:

```cpp
#include "clang-tidy/ClangTidyCheck.h"

namespace clang::tidy::mymodule {

class NoRawNewCheck : public ClangTidyCheck {
public:
  NoRawNewCheck(StringRef Name, ClangTidyContext *Context)
      : ClangTidyCheck(Name, Context),
        AllowPlacementNew(Options.get("AllowPlacementNew", false)) {}

  void registerMatchers(ast_matchers::MatchFinder *Finder) override;
  void check(const ast_matchers::MatchFinder::MatchResult &Result) override;
  void storeOptions(ClangTidyOptions::OptionMap &Options) override;

private:
  bool AllowPlacementNew;
};

} // namespace clang::tidy::mymodule
```

The implementation:

```cpp
#include "NoRawNewCheck.h"
#include "clang/ASTMatchers/ASTMatchFinder.h"

using namespace clang::ast_matchers;

namespace clang::tidy::mymodule {

void NoRawNewCheck::registerMatchers(MatchFinder *Finder) {
  // Match CXXNewExpr that is not placement-new
  Finder->addMatcher(
      cxxNewExpr(unless(hasAnyPlacementArg(anything()))).bind("newExpr"),
      this);
}

void NoRawNewCheck::check(const MatchFinder::MatchResult &Result) {
  const auto *NewExpr = Result.Nodes.getNodeAs<CXXNewExpr>("newExpr");
  if (!NewExpr)
    return;
  // Emit a warning with no automatic fix
  diag(NewExpr->getBeginLoc(),
       "avoid raw 'new'; prefer std::make_unique or std::make_shared")
      << FixItHint::CreateInsertion(NewExpr->getBeginLoc(), "/* FIXME */ ");
}

void NoRawNewCheck::storeOptions(ClangTidyOptions::OptionMap &Opts) {
  Options.store(Opts, "AllowPlacementNew", AllowPlacementNew);
}

} // namespace clang::tidy::mymodule
```

### 47.4.2 Per-Check Options

Options are read in the constructor via `Options.get("Key", DefaultValue)`. Supported types are `StringRef`, integral types, and enums with a specialized `OptionEnumMapping`. Options are written back via `storeOptions()` so that `clang-tidy --dump-config` can display current values:

```cpp
void NoRawNewCheck::storeOptions(ClangTidyOptions::OptionMap &Opts) {
  Options.store(Opts, "AllowPlacementNew", AllowPlacementNew);
}
```

In the `.clang-tidy` file, the option key is `mymodule-no-raw-new.AllowPlacementNew`.

### 47.4.3 Testing

clang-tidy checks use LLVM's lit-based testing framework. Test files live in `clang-tools-extra/test/clang-tidy/` and use `// CHECK-MESSAGES:` and `// CHECK-FIXES:` directives:

```cpp
// RUN: %check_clang_tidy %s mymodule-no-raw-new %t

void test() {
  int *p = new int(42);
  // CHECK-MESSAGES: :[[@LINE-1]]:12: warning: avoid raw 'new' [mymodule-no-raw-new]
  // CHECK-FIXES: /* FIXME */ new int(42);
}
```

`%check_clang_tidy` is a shell script that runs clang-tidy on the input, applies fixes to `%t`, and then runs FileCheck on both the diagnostic output and the fixed file. The `[[@LINE-1]]` substitution is FileCheck's line-relative directive.

---

## 47.5 clang-format: Style Configuration and API

### 47.5.1 FormatStyle

`FormatStyle` ([Format.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Format/Format.h)) is a large struct — over 400 fields — that controls every aspect of formatting. Key fields:

| Field | Effect |
|-------|--------|
| `BasedOnStyle` | Inheritance: `"LLVM"`, `"Google"`, `"Chromium"`, `"Mozilla"`, `"WebKit"`, `"GNU"`, `"Microsoft"` |
| `IndentWidth` | Spaces per indentation level (default: 2 for LLVM) |
| `ColumnLimit` | Maximum line width in columns (80 for LLVM, 100 for Google) |
| `BreakBeforeBraces` | Brace placement: `Attach`, `Linux`, `Allman`, `Stroustrup`, `Whitesmiths`, `Custom` |
| `AllowShortFunctionsOnASingleLine` | `None`, `Empty`, `Inline`, `All` |
| `SortIncludes` | `Never`, `CaseSensitive`, `CaseInsensitive` |
| `ReflowComments` | Reformat block and line comments to fit within `ColumnLimit` |
| `IncludeCategories` | `vector<IncludeCategory>` with priority regex and sort priority |
| `PenaltyBreakBeforeFirstCallParameter` | Penalty in the line-break dynamic-programming problem |
| `AttributeMacros` | List of macros to treat as C++ attributes (no space inserted before `(`) |

`FormatStyle` is loaded from a `.clang-format` file via `getStyleForFile(FileName, FallbackStyle, AllowUnknownOptions, FS)`. The YAML serialization is auto-generated from `Format.h` comments using `clang-format-update-docs`.

Built-in styles are retrieved via `getPredefinedStyle(Name, Language, Style)`. Each predefined style is a delta applied to the LLVM base style using `FormatStyle::merge()`.

### 47.5.2 The Programmatic API

The core formatting function is:

```cpp
namespace clang::format {

tooling::Replacements reformat(const FormatStyle &Style,
                               StringRef Code,
                               ArrayRef<tooling::Range> Ranges,
                               StringRef FileName = "<stdin>",
                               FormattingAttemptStatus *Status = nullptr);
} // namespace clang::format
```

`Ranges` specifies which byte ranges of `Code` to format; passing `{{0, Code.size()}}` formats the entire file. The returned `tooling::Replacements` is a set of non-overlapping `Replacement` objects. Applying them with `tooling::applyAllReplacements()` yields the formatted text.

Editor integrations (clangd's `textDocument/formatting`, `textDocument/rangeFormatting`, `textDocument/onTypeFormatting`) call `reformat` with the appropriate range and then convert the replacements to LSP `TextEdit` objects using `tooling::replacementToEdit()`.

### 47.5.3 Command-Line Usage

```bash
# Format in place
clang-format -i --style=file src/foo.cpp

# Dump the effective style as YAML
clang-format --dump-config

# Dry run: emit replacements without modifying files, error on any change
clang-format --dry-run --Werror src/foo.cpp

# Format only lines 10-25
clang-format --lines=10:25 src/foo.cpp
```

`git-clang-format` (a Python script in `clang/tools/clang-format/`) formats only the lines modified in the current working-tree diff:

```bash
git clang-format --diff HEAD~1
```

It calls `git diff-tree` to get changed ranges, then invokes `clang-format --lines=` for each changed hunk.

Escape hatches in source code:

```cpp
// clang-format off
Matrix m = { 1,0,0,
             0,1,0,
             0,0,1 };
// clang-format on
```

`DisableFormat: true` in `.clang-format` is the nuclear option; it passes the file through unchanged.

### 47.5.4 Style Inheritance and Custom Styles

`.clang-format` files support `BasedOnStyle: InheritParentConfig`, which merges the current file's settings atop the nearest parent-directory `.clang-format`. This is useful for monorepos where a root config sets global defaults and subdirectories override specific options:

```yaml
# root/.clang-format
BasedOnStyle: Google
ColumnLimit: 100

# root/legacy_code/.clang-format
BasedOnStyle: InheritParentConfig
ColumnLimit: 120
BreakBeforeBraces: Allman
```

The `IncludeCategories` field controls `#include` grouping and sorting. Each category is a regex with an integer priority; includes with lower priorities sort earlier within a group. A common Google-style setup:

```yaml
IncludeCategories:
  - Regex:           '^<.*\.h>'
    Priority:        1
    SortPriority:    0
    CaseSensitive:   false
  - Regex:           '^<.*>'
    Priority:        2
  - Regex:           '.*'
    Priority:        3
IncludeIsMainRegex: '([-_](test|unittest))?$'
```

`IncludeIsMainRegex` identifies which header is the "main" include for a source file (e.g., `foo.cpp`'s main header is `foo.h`); it is placed in priority group 0, ahead of all other includes.

---

## 47.6 clang-format Internals

### 47.6.1 UnwrappedLine and the Parser

The formatting pipeline begins by tokenizing the input using Clang's lexer. `UnwrappedLineParser` ([UnwrappedLineParser.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Format/UnwrappedLineParser.h)) builds a sequence of `UnwrappedLine` objects. An `UnwrappedLine` is a logical line: a maximal sequence of tokens that can be placed on a single physical line if there is enough column space, plus its indentation level. Statement bodies, preprocessor directives, and template parameter lists each produce their own `UnwrappedLine`.

Each `UnwrappedLine` contains a linked list of `FormatToken` nodes. `FormatToken` ([FormatToken.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Format/FormatToken.h)) augments the Clang `Token` with formatting metadata:

- `MustBreakBefore`: forced line break before this token
- `CanBreakBefore`: eligible break point
- `SpacesRequiredBefore`: mandatory space count
- `TotalLength`: sum of token text lengths from the line start to this token
- `MatchingParen`: pointer to the matching bracket token
- `Role`: contextual role (e.g., `CommaSeparatedList`, `OverloadedOperator`)

### 47.6.2 ContinuationIndenter and the Penalty Model

After parsing into `UnwrappedLine`s, `ContinuationIndenter` ([ContinuationIndenter.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Format/ContinuationIndenter.h)) computes the optimal sequence of line breaks. It models the formatting problem as a shortest-path search over a state space where each state is `(LineState, TotalPenalty)`. `LineState` records the current column position, the indentation stack, and a stack of `ParenState` for open brackets.

For each `FormatToken`, `ContinuationIndenter::addTokenToState()` either continues on the current line or breaks before the token, computing the penalty for each choice:

- Penalty for breaking (`PenaltyBreak`): proportional to how far the break point is from the column limit
- Penalty for exceeding column limit: `PenaltyExcessCharacter` per excess column
- `PenaltyBreakBeforeFirstCallParameter`: extra penalty for breaking immediately after `(`
- `PenaltyBreakComment`, `PenaltyBreakFirstLessLess`: special-case penalties

The dynamic programming is implemented in `UnwrappedLineFormatter::format()` ([UnwrappedLineFormatter.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Format/UnwrappedLineFormatter.cpp)) using a priority queue of `(penalty, state)` pairs — essentially Dijkstra's algorithm over the line-break state graph. This guarantees globally optimal line breaking within each `UnwrappedLine` in O(N × K) time, where N is the token count and K is a small branching factor (typically 2: break or don't break).

### 47.6.3 WhitespaceManager

`WhitespaceManager` ([WhitespaceManager.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Format/WhitespaceManager.h)) collects `Change` records — each specifying a token's preceding whitespace — and applies them in a post-processing step. It handles:

- Column alignment: `alignTrailingComments()`, `alignConsecutiveAssignments()`, `alignConsecutiveDeclarations()` scan groups of consecutive lines and insert padding spaces to vertically align tokens.
- Escaped newline alignment in preprocessor continuations.

The final output is produced by iterating the `Change` list in source order and emitting `tooling::Replacement` objects for each position where whitespace differs from the input.

### 47.6.4 Handling Special Constructs

Several C++ constructs require special-case logic in the formatter:

**Lambda formatting**: `LambdaBodyIndentation` (either `Signature` or `OuterScope`) controls whether the lambda body is indented relative to the lambda signature or the enclosing scope. When a lambda is passed as a function argument, `BraceWrappingAfterControl.AfterLambdaBody` controls whether the closing brace appears on a new line.

**Requires clauses**: In C++20 `requires` clauses, `RequiresClausePosition` (`OwnLine`, `WithPreceding`, `WithFollowing`, `SingleLine`) controls placement. `RequiresExpressionIndentation` (`OuterScope` or `Keyword`) controls indentation within a `requires` expression body.

**Macro formatting**: Preprocessor directives within macros are handled by `MacroBlockBegin`/`MacroBlockEnd` — regex strings identifying macros that should be treated as block openers/closers for indentation purposes. `IndentPPDirectives` (`None`, `AfterHash`, `BeforeHash`) controls indentation within `#if`/`#endif` chains.

**Trailing commas in initialization lists**: `InsertTrailingCommas` (`None` or `Wrapped`) automatically adds trailing commas to multi-line container initialization lists, easing diff-friendliness in version control.

The `TokenAnnotator` pass ([TokenAnnotator.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Format/TokenAnnotator.h)) runs before `ContinuationIndenter` to annotate each `FormatToken` with its `TokenType` (e.g., `TT_BinaryOperator`, `TT_TemplateOpener`, `TT_TrailingReturnArrow`) and `TokenRole` context. These annotations feed `ContinuationIndenter`'s break-penalty calculations, enabling correct handling of operator precedence in line-break decisions.

---

## 47.7 clang-refactor

### 47.7.1 Architecture

`clang-refactor` ([clang-tools-extra/clang-refactor/](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clang-refactor/)) is a command-line driver for the `clang/lib/Tooling/Refactoring/` library. It exposes two stable subcommands:

```bash
clang-refactor extract   [options] -selection=file:L1:C1-L2:C2 -- <compile args>
clang-refactor local-rename --new-name=foo -selection=file:L:C -- <compile args>
```

The `extract` subcommand is marked `(WIP action; use with caution!)` in Clang 22 — the interface is stable but edge-case handling for complex control flow (return statements, break/continue through the extracted range) is incomplete.

### 47.7.2 RefactoringAction and Rules

Each refactoring is implemented as a `RefactoringAction` ([RefactoringAction.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Tooling/Refactoring/RefactoringAction.h)) that provides:

```cpp
class ExtractFunction : public RefactoringAction {
public:
  StringRef getCommand() const override { return "extract"; }
  StringRef getDescription() const override { return "Extracts code..."; }
  SmallVector<std::unique_ptr<RefactoringActionRule>, 1>
  createActionRules() const override;
};
```

`createActionRules()` returns a list of `RefactoringActionRule` objects, each constraining when the action applies. For `extract`, the rule requires a `SourceRangeSelectionRequirement` — the user must specify a selection range. Rules can also require a `NamedDeclRefSelection` (for rename) or an `OccurrenceSelection` (for extract-repeating-expression).

`RefactoringResultConsumer` receives the computed `AtomicChange` list and decides how to apply them. The command-line driver uses `SourceChangeExecutor`, which applies changes to the physical filesystem using `tooling::applyAllReplacements()`.

### 47.7.3 Rename via clangd

clangd's rename pipeline (Section 47.1.4) builds on `tooling::SymbolOccurrences` and `tooling::createRenameReplacements()`. The `symbolOccurrences(Decl, ASTContext, SymbolIndex)` function returns all textual occurrences of a symbol across indexed files as `SymbolOccurrence` objects, each carrying the `MatchType` (declaration, definition, reference, selector) and source range.

`createRenameReplacements(Occurrences, SourceManager, NewName)` converts these to `tooling::Replacements`, handling the case where the old name is part of a qualified name (it replaces only the unqualified portion).

Limitations:
- **Macro-expanded names**: if a symbol name appears inside a macro expansion, the expansion site cannot be safely renamed because the replacement must be made in the macro definition, which may affect other call sites.
- **Multi-TU rename**: incomplete background indexing means references in unindexed TUs are silently missed. clangd warns about this via a `WorkDoneProgressReport`.
- **Computed names**: names generated by template or preprocessor string-pasting cannot be renamed.

### 47.7.4 Extract Function Internals

The `extract` subcommand's implementation in `clang/lib/Tooling/Refactoring/Extract/Extract.cpp` operates in several phases:

1. **Selection validation**: The selection range (from `--selection=file:L1:C1-L2:C2`) is mapped to a `CodeRangeASTSelection` by `CodeRangeASTSelection::create()`, which finds the outermost `Stmt` nodes fully enclosed in the range. Non-extractable selections include: ranges crossing statement boundaries without a common enclosing compound statement, ranges containing `break`/`continue` with targets outside the range, and ranges containing `return` statements (in the current implementation — this is a known WIP limitation).

2. **Variable analysis**: `ExtractedCodeVisitor` traverses the selected statements to classify every referenced `VarDecl` as:
   - *Input parameter*: used inside the selection but declared outside
   - *Output parameter*: assigned inside the selection and used outside (requires pass-by-pointer or return value)
   - *Local variable*: declared and used entirely within the selection

3. **Function signature generation**: The extracted function's signature is built from the input parameter list. If there is exactly one output variable, it becomes the return type. Multiple outputs require a `std::tuple` return or out-parameters.

4. **Replacement construction**: Two `AtomicChange` objects are created — one replacing the selected statements with a call to the new function, and one inserting the new function definition before the enclosing function.

---

## 47.8 include-what-you-use Integration

### 47.8.1 Architecture

include-what-you-use (IWYU) is a separate tool (not part of the LLVM monorepo) built on libtooling. `IWYUAstConsumer` traverses the AST and records which symbols are *used* (in declarations, definitions, and expressions) and which `#include` directives *provide* each symbol. It uses a `FullUseBuilder` to collect direct type uses (full definitions required) and a `FwdDeclUseBuilder` for contexts where a forward declaration suffices.

Mapping files (`.imp` files) express non-obvious symbol-to-header relationships in JSON:

```json
[
  { "symbol": ["std::vector", "private", "<vector>", "public"] },
  { "include": ["<bits/stdc++.h>", "private", "<iostream>", "public"] }
]
```

The mapping file tells IWYU that `std::vector` is publicly provided by `<vector>` even though the implementation includes it via `<bits/vector.h>`, and that `<bits/stdc++.h>` should never be suggested as a public include.

### 47.8.2 Applying Fixes

IWYU emits its analysis to stdout in a structured format. `fix_includes.py` parses this output and applies the suggested `#include` additions and removals to source files. The workflow:

```bash
# Run IWYU on one file
iwyu_tool.py -p build/ src/foo.cpp -- -Xiwyu --mapping_file=my.imp \
  > iwyu.out 2>&1

# Apply suggested fixes
fix_includes.py < iwyu.out
```

IWYU is deliberately separate from clang-tidy because its analysis requires a complete, cross-header understanding of which header provides each symbol — a global property that the per-TU model of clang-tidy checks cannot easily express.

---

## 47.9 pp-trace and clang-check

### 47.9.1 clang-check

`clang-check` ([clang-tools-extra/clang-check/](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clang-check/)) is the simplest libtooling-based frontend: it runs `SyntaxOnlyAction` (or `ASTDumpAction`, `ASTDeclListAction`, `ASTViewAction`) on source files. Its flags map directly to clang `-Xclang` diagnostic flags:

```bash
# Dump full AST
clang-check --ast-dump src/foo.cpp --

# Pretty-print AST (closer to C++ source)
clang-check --ast-print src/foo.cpp --

# List all declaration qualified names
clang-check --ast-list src/foo.cpp --

# Dump syntax tree (TreeSitter-style)
clang-check --syntax-tree-dump src/foo.cpp --

# Run static analyzer
clang-check --analyze src/foo.cpp --
```

`--ast-dump-filter=Foo` narrows the dump to declarations whose qualified name contains `Foo`, which is essential for navigating large AST dumps.

### 47.9.2 pp-trace

`pp-trace` ([clang-tools-extra/pp-trace/](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/pp-trace/)) installs a `PPCallbacks` subclass that logs every preprocessor event as YAML to stdout:

```bash
pp-trace --callbacks='MacroDefined,MacroExpanded,InclusionDirective' \
  src/foo.cpp --
```

Output excerpt:

```yaml
- Callback: InclusionDirective
  IncludeTok: <identifier>
  FileName: "vector"
  IsAngled: true
  FilenameRange: { FileId: 1, Offset: 42, Length: 8 }
  File: /usr/include/c++/14/vector
- Callback: MacroDefined
  MacroNameTok: NDEBUG
  MacroDirective: DefInfo
```

`pp-trace` is invaluable for debugging header guard issues, unexpected macro redefinitions, and the order of include processing. The `--callbacks` filter accepts globs; `--callbacks='-*,Macro*'` logs only macro-related events.

### 47.9.3 clang-extdef-mapping

`clang-extdef-mapping` prepares the cross-TU analysis index (Section 47.10). It runs `SyntaxOnlyAction` on each TU and emits a flat text file mapping each function/method `USR` to the source file that contains its definition:

```bash
# Run on all TUs in the build
clang-extdef-mapping -p build/ src/*.cpp > externalDefMap.txt
```

The output format is one `<USR> <absolute-path>` pair per line. Cross-TU analysis tools look up the definition file for a USR and import that AST.

---

## 47.10 Cross-TU Analysis

### 47.10.1 CrossTranslationUnitContext

Cross-TU analysis lives in `clang/lib/CrossTU/` ([CrossTranslationUnit.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CrossTU/CrossTranslationUnit.h)). `CrossTranslationUnitContext` owns:

- A map from function USR to the `ASTUnit` that has been imported so far
- A `CTUIndexMap` loaded from `externalDefMap.txt`
- An `ASTImporter` pool for each source-target TU pair

The central method is:

```cpp
llvm::Expected<const FunctionDecl *>
CrossTranslationUnitContext::getCrossTUDefinition(
    const FunctionDecl *FD,
    StringRef CrossTUDir,
    StringRef IndexName,
    bool DisplayCTUProgress);
```

It looks up `FD`'s USR in the index, loads the corresponding source file as a new `ASTUnit` (via `ASTUnit::LoadFromASTFile` if pre-built, or `ASTUnit::LoadFromCompilerInvocation` for fresh parsing), imports the definition into the current `ASTContext` using `ASTImporter::Import()`, and returns the imported `FunctionDecl *`.

### 47.10.2 Integration with the Static Analyzer

The static analyzer uses cross-TU analysis when launched with:

```bash
clang -cc1 -analyze \
  -analyzer-checker=core,unix \
  -analyzer-config experimental-enable-naive-ctu-analysis=true \
  -analyzer-config ctu-dir=/path/to/ctu-index \
  src/foo.cpp
```

During the analysis of `foo.cpp`, when the `ExprEngine` encounters a call to a function whose body is unavailable (i.e., defined in another TU), it calls `CrossTranslationUnitContext::getCrossTUDefinition()`. If the definition is found in the cross-TU index, the analyzer inlines the body and continues the path-sensitive analysis across the TU boundary. This substantially improves bug detection for callee-side issues that manifest only when considering the caller's argument invariants.

The "naive" qualifier in `experimental-enable-naive-ctu-analysis` indicates that the current implementation does not verify ABI compatibility between TUs — it imports definitions optimistically. More sophisticated implementations would check parameter types via the `ASTImporter`'s type-compatibility logic.

### 47.10.3 ASTImporter and Merging

When `CrossTranslationUnitContext::getCrossTUDefinition()` imports a `FunctionDecl` from a foreign `ASTUnit`, it calls `ASTImporter::Import(FD)`. `ASTImporter` ([ASTImporter.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/ASTImporter.h)) is a general-purpose AST transplant mechanism that recursively imports:

- The function's `QualType` (parameter types, return type), resolving type aliases in the destination `ASTContext`
- All `VarDecl`s and `TagDecl`s that the function's type system references
- Template specializations, if the function is an instantiation

The imported `FunctionDecl` is installed into the destination `TranslationUnitDecl`'s `DeclContext` under a synthetic `ExternalASTSource`. The `ASTImporterLookupTable` tracks which source `Decl` corresponds to which imported `Decl`, preventing double-import on repeated lookups.

A key challenge is ODR (One Definition Rule) violations: two TUs may define the same struct with incompatible layouts (e.g., due to differing compile flags). `ASTImporter` uses structural equivalence checking (`StructuralEquivalenceContext`) to detect obvious mismatches and emits an error rather than silently importing an incompatible definition. In the cross-TU analysis context, ODR violations cause the analyzer to skip inlining the offending function, falling back to conservative modeling.

---

## 47.11 Plugin Infrastructure

### 47.11.1 FrontendPlugin

Clang plugins live in `clang/include/clang/Frontend/FrontendPluginRegistry.h`. A plugin is a shared library that provides a `PluginASTAction` subclass registered via the `FrontendPluginRegistry`:

```cpp
class FindDeprecatedAPIAction : public PluginASTAction {
protected:
  std::unique_ptr<ASTConsumer>
  CreateASTConsumer(CompilerInstance &CI,
                    StringRef InFile) override {
    return std::make_unique<FindDeprecatedConsumer>(CI);
  }

  bool ParseArgs(const CompilerInstance &CI,
                 const std::vector<std::string> &Args) override {
    for (const auto &Arg : Args) {
      if (Arg == "-deprecated-header")
        DeprecatedHeader = Args[++i];
    }
    return true;
  }

  ActionType getActionType() override {
    return AddBeforeMainAction; // or AddAfterMainAction, ReplaceAction
  }
};

static FrontendPluginRegistry::Add<FindDeprecatedAPIAction>
    X("find-deprecated", "Finds usages of deprecated API headers.");
```

Load the plugin during compilation:

```bash
clang++ -fplugin=/path/to/myplugin.so \
        -Xclang -plugin-arg-find-deprecated \
        -Xclang -deprecated-header=legacy.h \
        src/foo.cpp
```

### 47.11.2 Attribute Plugins

Attribute plugins extend Clang's attribute grammar. Register via `ParsedAttrInfoRegistry`:

```cpp
struct MyAttrInfo : public ParsedAttrInfo {
  MyAttrInfo() {
    NumArgs = 0;
    OptArgs = 0;
    HasCustomParsing = false;
    IsType = false;
    AttrKind = clang::ParsedAttr::AT_Annotate; // reuse AnnotateAttr
  }

  bool diagAppertainsToDecl(Sema &S, const ParsedAttr &Attr,
                             const Decl *D) const override {
    if (!isa<FunctionDecl>(D)) {
      S.Diag(Attr.getLoc(), diag::warn_attribute_wrong_decl_type_str)
          << Attr << Attr.isRegularKeywordAttribute()
          << "functions";
      return false;
    }
    return true;
  }
};

static ParsedAttrInfoRegistry::Add<MyAttrInfo>
    Y("my_attr", "A custom attribute.");
```

Attribute plugins can inject `AnnotateAttr` nodes into the AST, which subsequent analysis passes (or clang-tidy checks) can query via `Decl::hasAttr<AnnotateAttr>()`.

### 47.11.3 Plugin vs. clang-tidy Check

| Criterion | Plugin | clang-tidy check |
|-----------|--------|-----------------|
| Build integration | `-fplugin=` during compilation | Separate `clang-tidy` invocation |
| Modify AST/IR | Yes (can inject attributes, run codegen) | No (read-only AST, emit fix-its) |
| Fix-it support | Limited | First-class via `DiagnosticBuilder` |
| Configuration | Custom CLI args | `.clang-tidy` YAML |
| Out-of-tree distribution | Single `.so` file | Requires recompiling clang-tidy or `--load` |
| Per-file suppression | `#pragma clang diagnostic` | `// NOLINT` comments |

Use a plugin when the action must occur during compilation — injecting attributes that affect codegen, enforcing ABI rules, or generating additional output files. Use a clang-tidy check when the goal is code quality analysis with suggested fixes that developers review before applying.

---

## 47.12 Performance and Large Codebases

### 47.12.1 AllTUsToolExecutor and Parallelism

The `ClangTool` API (Section [Chapter 46](../part-07-clang-multilang/ch46-libtooling-and-ast-matchers.md)) runs one TU at a time. For large codebases, `AllTUsToolExecutor` ([AllTUsToolExecutor.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Tooling/AllTUsToolExecutor.h)) runs a `ToolAction` on all TUs in a `CompilationDatabase` in parallel:

```cpp
auto Executor = std::make_unique<tooling::AllTUsToolExecutor>(
    *Compilations, /*ThreadCount=*/std::thread::hardware_concurrency());

auto Err = Executor->execute(
    std::make_unique<tooling::FrontendActionFactory<MyAction>>());
```

Each worker thread owns its own `DiagnosticsEngine` and `ASTContext`; results are collected via a thread-safe `ToolResults` map (string → vector of string). The executor handles errors per-TU: a parse failure in one TU does not abort others.

`clang-tidy` uses `AllTUsToolExecutor` when passed multiple files or `--header-filter` with a compilation database. The `--use-color` flag enables ANSI coloring of diagnostics, `--fix` applies suggested fix-its in place, and `--fix-errors` applies fixes even when a TU has compilation errors (useful for generated code).

### 47.12.2 compile_commands.json Generation

Every tool in this chapter requires accurate compiler flags, supplied via `compile_commands.json`. Generation methods:

- **CMake**: `cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON build/` creates `build/compile_commands.json`.
- **Bazel**: Use `bazel build --aspects=@bazel_compdb//:aspects.bzl%compilation_database_aspect --output_groups=compdb_files //...` or the `hedronvision/bazel-compile-commands-extractor` tool.
- **Bear**: `bear -- make` intercepts compiler invocations via `LD_PRELOAD` and records them.
- **compiledb**: `compiledb make` uses the `make -n` dry-run output to infer flags.

Symlink `compile_commands.json` into the source root: `ln -s build/compile_commands.json .`. clangd, clang-tidy, and clang-refactor all walk up from each source file to find it.

The `compile_commands.json` format is a JSON array of command objects:

```json
[
  {
    "directory": "/home/user/project/build",
    "command": "clang++ -std=c++20 -I/usr/include -DFOO=1 -o foo.o -c /home/user/project/src/foo.cpp",
    "file": "/home/user/project/src/foo.cpp"
  }
]
```

The `"arguments"` array form (splitting the command into individual tokens) is preferred over the `"command"` string form because it avoids shell quoting ambiguities. `CompilationDatabase::autoDetectFromDirectory()` in `clang/lib/Tooling/CompilationDatabase.cpp` searches for `compile_commands.json` by walking parent directories, which is the same discovery logic used by clangd, clang-tidy, and clang-refactor.

For files not present in the database (generated headers, third-party files), `JSONCompilationDatabase` returns a fallback set of flags via `getCompileCommands()` returning the database entry for the closest matching file. `FixedCompilationDatabase` is used when the entire project uses a uniform set of flags, specified via `--` on the command line.

### 47.12.3 Timeout and Error Handling

`AllTUsToolExecutor` runs each TU in a worker thread. If a TU parse hangs (e.g., due to a deeply recursive template instantiation), the executor's timeout mechanism (configurable via `--execution-timeout-seconds`) sends a `SIGALRM` to the worker, which is caught by Clang's `CrashRecoveryContext`. The TU is marked as failed and the worker continues with the next one. Failed TUs are reported via `ToolResults::addResult("failed-files", filename)`.

`clang-tidy`'s `--timeout` flag wraps each TU's processing in a `llvm::sys::ProcessTimeout`. The `--verify-config` flag validates the `.clang-tidy` file syntax and reports unknown keys without running any analysis.

For CI environments, `clang-tidy --format-style=file --export-fixes=fixes.yaml` collects all fix-its into a YAML file without modifying source. A separate CI step can review or apply these fixes with `clang-apply-replacements fixes.yaml`.

### 47.12.4 clangd Configuration File

Beyond `.clang-tidy` files, clangd reads `~/.config/clangd/config.yaml` (user-level) and `.clangd` files in the source tree (project-level). The configuration supports:

```yaml
CompileFlags:
  Add: [-DDEBUG_MODE, -Wno-unused-parameter]
  Remove: [-W*]

Diagnostics:
  ClangTidy:
    Add: modernize-use-override
    Remove: readability-*
    CheckOptions:
      modernize-use-override.IgnoreDestructors: true
  UnusedIncludes: Strict

Index:
  Background: Build

InlayHints:
  Enabled: Yes
  ParameterNames: Yes
  DeducedTypes: Yes

Hover:
  ShowAKA: Yes
```

`ShowAKA` in the `Hover` section controls whether clangd appends the canonical type alias expansion (the "AKA" or "also known as" clause) to hover types — useful when template instantiations produce complex type names.

### 47.12.5 clangd with Remote Index

For codebases with millions of lines, local background indexing takes hours. The remote index architecture (Section 47.1.6) requires:

1. A CI job that runs `clangd-indexer` on the full codebase, producing a merged `.idx` file.
2. A `clangd-index-server` process serving the index via gRPC.
3. Developer `~/.config/clangd/config.yaml` entries:

```yaml
Index:
  External:
    Server: grpc://index-server.internal:5900
    MountPoint: /home/user/src/myproject
```

`MountPoint` tells clangd which local path prefix corresponds to the server's root. The server handles `FuzzyFind` and `Refs` RPCs; local file edits are handled by an overlay in-memory index that takes precedence over remote results.

---

## 47.13 Summary

- `ClangdLSPServer` → `ClangdServer` → `TUScheduler` is the three-layer architecture that decouples transport, semantic coordination, and per-file async parsing. `ASTWorker` threads own preambles and `ParsedAST` caches, reusing unchanged preambles across edits.
- Inlay hints, call hierarchy, type hierarchy, rename, and workspace symbol are all implemented against `ParsedAST` and the `BackgroundIndex`; rename uses `tooling::Replacements` and is cross-TU only when the index is complete.
- `ClangTidyCheck` subclasses override `registerMatchers`, `check`, and optionally `registerPPCallbacks`; `ClangTidyContext` holds the diagnostics engine; `ClangTidyCheckFactories` is the plugin registry.
- The `modernize`, `readability`, `performance`, `bugprone`, `cppcoreguidelines`, and `clang-analyzer` modules collectively cover the most impactful C++ code-quality patterns.
- `FormatStyle` has 400+ fields; the formatting pipeline proceeds: tokenize → `UnwrappedLineParser` → `ContinuationIndenter` (Dijkstra penalty model) → `WhitespaceManager` (alignment, spacing) → `tooling::Replacements`.
- `clang-refactor`'s `extract` and `local-rename` subcommands are built on `RefactoringAction`/`RefactoringActionRule`; clangd's rename pipeline uses `symbolOccurrences()` and `createRenameReplacements()`.
- Cross-TU analysis uses `CrossTranslationUnitContext::getCrossTUDefinition()` to import function bodies from pre-built index files, enabling the static analyzer to inline across TU boundaries.
- Plugins (`FrontendPluginRegistry`) suit compile-time actions; clang-tidy checks suit code-quality analysis with fix-its. The two mechanisms share Clang's semantic model but differ in lifecycle and distribution.
- `AllTUsToolExecutor` parallelizes clang-tidy and other tools over entire compilation databases; `compile_commands.json` is the universal source of truth for compiler flags.

---

*Cross-references:*
- [Chapter 38 — Code Completion and clangd Foundations](../part-05-clang-frontend/ch38-code-completion-and-clangd-foundations.md) — `PrecompiledPreamble`, `CodeCompleteConsumer`, and LSP basics
- [Chapter 45 — The Static Analyzer](../part-07-clang-multilang/ch45-the-static-analyzer.md) — checker infrastructure wrapped by `clang-analyzer-*` checks and cross-TU analysis
- [Chapter 46 — libtooling and AST Matchers](../part-07-clang-multilang/ch46-libtooling-and-ast-matchers.md) — `ClangTool`, `ASTMatchFinder`, `MatchFinder::MatchCallback` — the substrate on which clang-tidy checks are built

*Reference links:*
- [`clang-tools-extra/clangd/ClangdLSPServer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/ClangdLSPServer.h)
- [`clang-tools-extra/clangd/TUScheduler.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/TUScheduler.h)
- [`clang-tools-extra/clangd/InlayHints.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/InlayHints.cpp)
- [`clang-tools-extra/clangd/index/Background.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clangd/index/Background.h)
- [`clang-tools-extra/clang-tidy/ClangTidyCheck.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clang-tidy/ClangTidyCheck.h)
- [`clang-tools-extra/clang-tidy/ClangTidyOptions.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang-tools-extra/clang-tidy/ClangTidyOptions.h)
- [`clang/include/clang/Format/Format.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Format/Format.h)
- [`clang/lib/Format/UnwrappedLineParser.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Format/UnwrappedLineParser.h)
- [`clang/lib/Format/ContinuationIndenter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Format/ContinuationIndenter.h)
- [`clang/lib/Format/WhitespaceManager.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Format/WhitespaceManager.h)
- [`clang/include/clang/CrossTU/CrossTranslationUnit.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CrossTU/CrossTranslationUnit.h)
- [`clang/include/clang/Frontend/FrontendPluginRegistry.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/FrontendPluginRegistry.h)
- [`clang/include/clang/Tooling/AllTUsToolExecutor.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Tooling/AllTUsToolExecutor.h)


---

@copyright jreuben11
