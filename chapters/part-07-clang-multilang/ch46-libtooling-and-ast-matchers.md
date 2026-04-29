# Chapter 46 — libtooling and AST Matchers

*Part VII — Clang as a Multi-Language Compiler*

The Clang tooling infrastructure exposes the compiler's full semantic understanding of C++ source code to external programs. Where compiler plugins must be compiled in-process and loaded as shared libraries, libtooling builds standalone executables that drive Clang as a library, accessing the parsed AST, manipulating source text, and running arbitrary analysis passes. The AST matcher DSL sits on top of that infrastructure, providing a declarative query language for locating specific patterns in C++ code with the same precision the compiler itself uses. Together, these two layers underpin every significant Clang-based tooling product: clang-tidy, clang-rename, clangd's indexing engine, and thousands of internal refactoring tools deployed in large C++ codebases. This chapter covers both layers systematically — the execution framework that drives Clang over a set of source files, and the pattern-matching and source-rewriting machinery that tools use once they have the AST.

---

## 46.1 The libtooling Execution Framework

### 46.1.1 Core Abstractions

libtooling lives in `clang/include/clang/Tooling/Tooling.h` and `clang/lib/Tooling/Tooling.cpp`. Three classes form the execution spine:

**`ToolAction`** is the abstract interface for anything a tool wants to do per translation unit. It declares a single virtual method:

```cpp
class ToolAction {
public:
  virtual ~ToolAction();
  virtual bool runInvocation(
      std::shared_ptr<CompilerInvocation> Invocation,
      FileManager *Files,
      std::shared_ptr<PCHContainerOperations> PCHContainerOps,
      DiagnosticConsumer *DiagConsumer) = 0;
  virtual void notifySkippedFile(StringRef FilePath) {}
};
```

Most tools never implement `ToolAction` directly. They subclass the higher-level `FrontendActionFactory` instead.

**`FrontendActionFactory`** wraps `ToolAction` and produces one `FrontendAction` instance per translation unit:

```cpp
class FrontendActionFactory : public ToolAction {
public:
  ~FrontendActionFactory() override;
  virtual std::unique_ptr<FrontendAction> create() = 0;
};
```

The template helper `newFrontendActionFactory<T>()` (defined inline in `Tooling.h`) generates a factory for any concrete `FrontendAction` subclass `T`:

```cpp
template <typename T>
std::unique_ptr<FrontendActionFactory> newFrontendActionFactory() {
  class SimpleFrontendActionFactory : public FrontendActionFactory {
  public:
    std::unique_ptr<FrontendAction> create() override {
      return std::make_unique<T>();
    }
  };
  return std::make_unique<SimpleFrontendActionFactory>();
}
```

The overloaded form `newFrontendActionFactory(FactoryT*, SourceFileCallbacks*)` accepts an `ASTConsumer` factory so that the tool can inject consumers without writing a full `FrontendAction` subclass.

**`ClangTool`** orchestrates execution over a list of source files:

```cpp
class ClangTool {
public:
  ClangTool(const CompilationDatabase &Compilations,
            ArrayRef<std::string> SourcePaths,
            std::shared_ptr<PCHContainerOperations> PCHContainerOps = ...,
            IntrusiveRefCntPtr<llvm::vfs::FileSystem> BaseFS = ...,
            IntrusiveRefCntPtr<FileManager> Files = nullptr);

  int run(ToolAction *Action);
  int buildASTs(std::vector<std::unique_ptr<ASTUnit>> &ASTs);
  void appendArgumentsAdjuster(ArgumentsAdjuster Adjuster);
  void mapVirtualFile(StringRef FilePath, StringRef Content);
};
```

`ClangTool::run()` iterates over each source path, looks up its `CompileCommand` in the database, applies any registered `ArgumentsAdjuster` chain, constructs a `CompilerInvocation`, and calls `ToolAction::runInvocation()`. The return value is 0 on full success, 1 if any action failed, and 2 if some files lacked compile commands.

### 46.1.2 Minimal Tool Template

The canonical libtooling tool follows this skeleton:

```cpp
#include "clang/Tooling/CommonOptionsParser.h"
#include "clang/Tooling/Tooling.h"
#include "clang/Frontend/FrontendActions.h"
#include "llvm/Support/CommandLine.h"

using namespace clang::tooling;
using namespace llvm;

static cl::OptionCategory MyToolCategory("my-tool options");
static cl::extrahelp CommonHelp(CommonOptionsParser::HelpMessage);

int main(int argc, const char **argv) {
  auto ExpectedParser =
      CommonOptionsParser::create(argc, argv, MyToolCategory);
  if (!ExpectedParser) {
    llvm::errs() << ExpectedParser.takeError();
    return 1;
  }
  CommonOptionsParser &OptionsParser = ExpectedParser.get();
  ClangTool Tool(OptionsParser.getCompilations(),
                 OptionsParser.getSourcePathList());
  return Tool.run(newFrontendActionFactory<SyntaxOnlyAction>().get());
}
```

`CommonOptionsParser` handles `--` separator syntax (`my-tool source.cpp -- -std=c++20`) and calls `CompilationDatabase::autoDetectFromDirectory()` when no `compile_commands.json` is specified.

### 46.1.3 Testing with `runToolOnCode`

For unit tests, materializing a real compilation database is impractical. `runToolOnCode()` and `runToolOnCodeWithArgs()` run a `FrontendAction` directly over an in-memory string:

```cpp
bool runToolOnCode(std::unique_ptr<FrontendAction> ToolAction,
                  const Twine &Code,
                  const Twine &FileName = "input.cc",
                  std::shared_ptr<PCHContainerOperations> PCHContainerOps = ...);

bool runToolOnCodeWithArgs(
    std::unique_ptr<FrontendAction> ToolAction,
    const Twine &Code,
    const std::vector<std::string> &Args,
    const Twine &FileName = "input.cc",
    const Twine &ToolName = "clang-tool",
    std::shared_ptr<PCHContainerOperations> PCHContainerOps = ...,
    const FileContentMappings &VirtualMappedFiles = {});
```

These are the standard testing helpers in clang-tidy's test infrastructure. Every clang-tidy check is exercised via `runToolOnCodeWithArgs` in its unit tests rather than through the full driver pipeline.

### 46.1.4 Flag Injection with `ArgumentsAdjuster`

`ArgumentsAdjuster` is a `std::function<CommandLineArguments(CommandLineArguments, StringRef)>`. Several factories exist in `clang/Tooling/ArgumentsAdjusters.h`:

| Factory | Effect |
|---------|--------|
| `getClangSyntaxOnlyAdjuster()` | Adds `-fsyntax-only`, removes link flags |
| `getClangStripOutputAdjuster()` | Removes `-o output` |
| `getClangStripDependencyFileAdjuster()` | Removes `-MF`, `-MD` flags |
| `getInsertArgumentAdjuster(args, pos)` | Prepends or appends arbitrary flags |
| `getStripPluginsAdjuster()` | Removes `-fplugin=` arguments |
| `combineAdjusters(first, second)` | Chains two adjusters |

```cpp
Tool.appendArgumentsAdjuster(
    combineAdjusters(
        getClangSyntaxOnlyAdjuster(),
        getInsertArgumentAdjuster("-std=c++20",
                                  ArgumentInsertPosition::BEGIN)));
```

`ClangTool` maintains an `ArgumentsAdjuster` chain that is applied in registration order each time it builds a `CompilerInvocation`. Adjusters are the correct mechanism for overriding language standard, include paths, or feature macros without modifying the stored compilation database.

---

## 46.2 The Compilation Database

### 46.2.1 `CompilationDatabase` and `CompileCommand`

`clang/include/clang/Tooling/CompilationDatabase.h` declares the abstract base and the fundamental data struct:

```cpp
struct CompileCommand {
  std::string Directory;   // working directory for the command
  std::string Filename;    // canonical file path
  std::vector<std::string> CommandLine;  // argv including compiler name
  std::string Output;      // -o target if present
  std::string Heuristic;   // diagnostic string for interpolated entries
};

class CompilationDatabase {
public:
  virtual ~CompilationDatabase();
  virtual std::vector<CompileCommand>
  getCompileCommands(StringRef FilePath) const = 0;
  virtual std::vector<std::string> getAllFiles() const { return {}; }
  virtual std::vector<CompileCommand> getAllCompileCommands() const { return {}; }

  static std::unique_ptr<CompilationDatabase>
  autoDetectFromSource(StringRef SourceFile, std::string &ErrorMessage);
  static std::unique_ptr<CompilationDatabase>
  autoDetectFromDirectory(StringRef SourceDir, std::string &ErrorMessage);
  static std::unique_ptr<CompilationDatabase>
  loadFromDirectory(StringRef BuildDirectory, std::string &ErrorMessage);
};
```

`autoDetectFromSource()` walks the file's parent directories searching for `compile_commands.json` or a `CMakeCache.txt`, then falls back to returning a fixed database if none is found. This is the path `CommonOptionsParser` takes when no explicit database is specified.

### 46.2.2 `JSONCompilationDatabase`

The JSON Compilation Database format (defined at [https://clang.llvm.org/docs/JSONCompilationDatabase.html](https://clang.llvm.org/docs/JSONCompilationDatabase.html)) is the standard output format for CMake (`-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`), Bazel, and Meson. `JSONCompilationDatabase` in `clang/Tooling/JSONCompilationDatabase.h` implements the loader:

```cpp
class JSONCompilationDatabase : public CompilationDatabase {
public:
  static std::unique_ptr<JSONCompilationDatabase>
  loadFromFile(StringRef FilePath, std::string &ErrorMessage,
               JSONCommandLineSyntax Syntax = JSONCommandLineSyntax::AutoDetect);
  static std::unique_ptr<JSONCompilationDatabase>
  loadFromBuffer(StringRef DatabaseString, std::string &ErrorMessage,
                 JSONCommandLineSyntax Syntax);
};
```

Each JSON entry may use either `"command"` (shell-escaped string) or `"arguments"` (pre-split array). The `JSONCommandLineSyntax` enum controls which format to expect; `AutoDetect` handles both.

### 46.2.3 `FixedCompilationDatabase` and `InterpolatingCompilationDatabase`

`FixedCompilationDatabase` supplies a single command line to every queried file. It is constructed either programmatically or from a `--` separator on the command line:

```cpp
// Equivalent to: my-tool a.cpp b.cpp -- -std=c++20 -I/usr/local/include
auto DB = std::make_unique<FixedCompilationDatabase>(
    Twine("."), std::vector<std::string>{"-std=c++20", "-I/usr/local/include"});
```

`InterpolatingCompilationDatabase` wraps another `CompilationDatabase` and synthesizes entries for files that have no direct entry. When `getCompileCommands()` finds no exact match, it searches for the closest file in the same directory tree with an entry, strips the filename from the original command, and substitutes the requested file. This matters when header files are queried directly (as clangd does): headers rarely have dedicated compilation database entries, so interpolation from a nearby translation unit's entry approximates the correct include paths and macros.

### 46.2.4 `CompilationDatabasePluginRegistry`

External compilation database formats register themselves through `CompilationDatabasePluginRegistry`, an `llvm::Registry` instantiation:

```cpp
// clang/include/clang/Tooling/CompilationDatabasePluginRegistry.h
using CompilationDatabasePluginRegistry =
    llvm::Registry<CompilationDatabasePlugin>;
```

A plugin provides a `loadFromDirectory()` implementation. The `autoDetectFromDirectory()` function queries all registered plugins in addition to the built-in JSON loader.

---

## 46.3 `FrontendAction` and `ASTConsumer`

### 46.3.1 The Action Hierarchy

`ASTFrontendAction` is the base for any action that parses source and produces an AST. Subclasses override `CreateASTConsumer()`:

```cpp
class ASTFrontendAction : public FrontendAction {
protected:
  virtual std::unique_ptr<ASTConsumer>
  CreateASTConsumer(CompilerInstance &CI, StringRef InFile) = 0;
};
```

`SyntaxOnlyAction` (in `clang/Frontend/FrontendActions.h`) implements `CreateASTConsumer()` to return an empty consumer that discards the AST. It is suitable as a base for tools that only need type-checking side effects, but most tools provide their own consumer.

`CompilerInstance` passed to `CreateASTConsumer()` is fully initialized at that point, providing:
- `CI.getASTContext()` — the `ASTContext` for the TU
- `CI.getPreprocessor()` — the `Preprocessor`, needed for PP callbacks
- `CI.getSourceManager()` — the `SourceManager`
- `CI.getDiagnostics()` — the `DiagnosticsEngine`
- `CI.getFileManager()` — the `FileManager`

### 46.3.2 `ASTConsumer` Callbacks

`ASTConsumer` (in `clang/AST/ASTConsumer.h`) has several virtual callbacks:

```cpp
class ASTConsumer {
public:
  virtual ~ASTConsumer();
  virtual void Initialize(ASTContext &Context) {}
  virtual bool HandleTopLevelDecl(DeclGroupRef D);
  virtual void HandleTranslationUnit(ASTContext &Ctx) {}
  virtual void HandleTagDeclDefinition(TagDecl *D) {}
  virtual void HandleCXXStaticMemberVarInstantiation(VarDecl *D) {}
  virtual void HandleTopLevelDeclInObjCContainer(DeclGroupRef D) {}
  virtual void HandleImplicitImportDecl(ImportDecl *D) {}
  virtual ASTMutationListener *GetASTMutationListener() { return nullptr; }
  virtual ASTDeserializationListener *GetASTDeserializationListener() {
    return nullptr;
  }
  virtual void PrintStats() {}
  virtual bool shouldSkipFunctionBody(Decl *D) { return false; }
};
```

`HandleTopLevelDecl()` is invoked incrementally as declarations are parsed — this is the only callback called during parsing rather than after. `HandleTranslationUnit()` fires once parsing and semantic analysis are complete; it is the primary callback for tools that need the full AST.

`HandleTagDeclDefinition()` fires each time a `struct`, `class`, `union`, or `enum` definition is completed during parsing. This is more targeted than `HandleTopLevelDecl()` for tools that only care about type definitions.

`shouldSkipFunctionBody()` enables lazy parsing of function bodies. When a consumer returns `true` for a given `Decl`, the parser skips tokenizing the function body entirely and inserts a `CompoundStmt` stub. This is the basis for the `clang -fsyntax-only` fast path used in header-only analysis: Sema processes declarations and types without building AST nodes for every function body, dramatically reducing memory usage.

A production-grade consumer that populates a symbol index typically uses `HandleTopLevelDecl()` for fast incremental indexing and `HandleTranslationUnit()` for a final consistency pass. The incremental callback allows the consumer to pipeline work (emitting indexed symbols as they arrive) rather than buffering the entire AST.

### 46.3.3 `MultiplexConsumer`

When multiple consumers must observe the same TU, `MultiplexConsumer` (in `clang/Frontend/MultiplexConsumer.h`) fans out all callbacks:

```cpp
class MultiplexConsumer : public SemaConsumer {
public:
  explicit MultiplexConsumer(
      std::vector<std::unique_ptr<ASTConsumer>> C);
  void Initialize(ASTContext &Context) override;
  bool HandleTopLevelDecl(DeclGroupRef D) override;
  void HandleTranslationUnit(ASTContext &Ctx) override;
  // ... all other ASTConsumer virtuals
};
```

This is how clang-tidy runs dozens of independent checks over a single parse: each check provides its own `ASTConsumer` (usually a `MatchFinder::newASTConsumer()` wrapping an internal matcher), and the driver wraps them all in a `MultiplexConsumer`.

---

## 46.4 `ASTUnit`

### 46.4.1 In-Process Compilation

`ASTUnit` (in `clang/Frontend/ASTUnit.h`) represents a fully parsed translation unit held in memory. Unlike `ClangTool`, which runs and discards each TU, `ASTUnit` retains the `ASTContext`, `Preprocessor`, and `SourceManager` for subsequent query or reparse:

```cpp
class ASTUnit {
public:
  static std::unique_ptr<ASTUnit>
  LoadFromCompilerInvocation(
      std::shared_ptr<CompilerInvocation> CI,
      std::shared_ptr<PCHContainerOperations> PCHContainerOps,
      IntrusiveRefCntPtr<DiagnosticsEngine> Diags,
      FileManager *FileMgr,
      bool OnlyLocalDecls = false,
      CaptureDiagsKind CaptureDiagnostics = CaptureDiagsKind::None,
      bool PrecompilePreamble = false,
      TranslationUnitKind TUKind = TU_Complete,
      bool CacheCodeCompletionResults = false,
      bool IncludeBriefCommentsInCodeCompletion = false,
      bool UserFilesAreVolatile = false);

  static std::unique_ptr<ASTUnit>
  LoadFromCommandLine(
      const char **ArgBegin, const char **ArgEnd,
      std::shared_ptr<PCHContainerOperations> PCHContainerOps,
      IntrusiveRefCntPtr<DiagnosticsEngine> Diags,
      StringRef ResourceFilesPath,
      bool OnlyLocalDecls = false,
      ...);

  bool Reparse(
      std::shared_ptr<PCHContainerOperations> PCHContainerOps,
      ArrayRef<RemappedFile> RemappedFiles = {},
      IntrusiveRefCntPtr<vfs::FileSystem> VFS = nullptr);
};
```

`LoadFromCommandLine()` accepts a raw `argv` array, making it usable when the compiler invocation is expressed as a shell command line rather than a structured `CompilerInvocation`. This is the path libclang and clangd use internally.

### 46.4.2 Preamble and Incremental Reparse

`ASTUnit` implements preamble precompilation for fast re-parsing. When `PrecompilePreamble = true`, the unchanged prefix of the file (typically all `#include` directives and their transitive content) is serialized to a PCH buffer on first parse. Subsequent `Reparse()` calls deserialize the preamble and only re-parse the changed suffix. clangd relies on this to achieve sub-second reparse latency after edits.

`Reparse()` accepts `RemappedFile` pairs mapping a filesystem path to an in-memory buffer, enabling the caller to supply unsaved file contents without writing to disk — the standard pattern for IDE integration where the user's editor holds buffers not yet persisted.

### 46.4.3 Querying `ASTUnit`

```cpp
ASTContext &getASTContext();
Preprocessor &getPreprocessor();
SourceManager &getSourceManager();
bool getLocalTopLevelDecls(SmallVectorImpl<Decl *> &Results);
const FileEntry *getFile(StringRef Filename);
```

`getLocalTopLevelDecls()` returns declarations defined in the main file (excluding those from included headers), which is useful when a tool only processes code the user wrote.

---

## 46.5 The AST Matcher DSL

### 46.5.1 Matcher Types

The `clang::ast_matchers` namespace (declared in `clang/ASTMatchers/ASTMatchers.h`) defines the matcher DSL. The foundational matcher types are type aliases over `internal::Matcher<NodeType>`:

```cpp
using DeclarationMatcher        = internal::Matcher<Decl>;
using StatementMatcher          = internal::Matcher<Stmt>;
using TypeMatcher               = internal::Matcher<QualType>;
using TypeLocMatcher            = internal::Matcher<TypeLoc>;
using NestedNameSpecifierMatcher = internal::Matcher<NestedNameSpecifier>;
using NestedNameSpecifierLocMatcher = internal::Matcher<NestedNameSpecifierLoc>;
using CXXCtorInitializerMatcher = internal::Matcher<CXXCtorInitializer>;
using TemplateArgumentLocMatcher = internal::Matcher<TemplateArgumentLoc>;
using AttrMatcher               = internal::Matcher<Attr>;
```

Internally every matcher is a polymorphic callable: `internal::Matcher<T>` wraps a `std::shared_ptr<internal::MatcherInterface<T>>` that defines `bool matches(const T &, ASTMatchFinder *, BoundNodesTreeBuilder *) const`. The DSL's type safety comes from template instantiation: a matcher that accepts a `FunctionDecl` cannot be used where a `Stmt` is expected.

### 46.5.1a `DynTypedMatcher` and the Dynamic Interface

The `internal::DynTypedMatcher` class in `clang/ASTMatchers/ASTMatchersInternal.h` is the type-erased backbone that allows the matcher DSL to function at runtime. It stores matchers as `llvm::IntrusiveRefCntPtr<MatcherInterface>` together with the `ASTNodeKind` of the accepted node type:

```cpp
class DynTypedMatcher {
public:
  static DynTypedMatcher constructVariadic(
      DynTypedMatcher::VariadicOperator Op,
      ASTNodeKind SupportedKind,
      std::vector<DynTypedMatcher> InnerMatchers);

  bool matches(const DynTypedNode &DynNode,
               ASTMatchFinder *Finder,
               BoundNodesTreeBuilder *Builder) const;
  bool matchesNoKindCheck(const DynTypedNode &DynNode,
                          ASTMatchFinder *Finder,
                          BoundNodesTreeBuilder *Builder) const;

  ASTNodeKind getSupportedKind() const;
  bool canConvertTo(ASTNodeKind Kind) const;
  DynTypedMatcher dynCastTo(ASTNodeKind Kind) const;
};
```

`DynTypedMatcher` is what `clang-query` manipulates — it receives matcher expressions parsed from strings and stores them as `DynTypedMatcher` instances. The typed `internal::Matcher<T>` is a thin wrapper that carries the concrete `T` type at the C++ level for compile-time safety. Both ultimately delegate to the same `MatcherInterface`.

`ASTNodeKind` (in `clang/AST/ASTTypeTraits.h`) encodes the class hierarchy of AST nodes as runtime values. `DynTypedMatcher::canConvertTo()` checks whether a matcher for a base class (`Decl`) is compatible with a derived class context (`FunctionDecl`) — the underpinning of the DSL's implicit narrowing from base to derived types.

The `BoundNodesTreeBuilder` is the mutable accumulator passed through matcher evaluation. When `.bind("label")` is encountered, the matching node is inserted into the builder under the given key. At the end of evaluation, `BoundNodesTreeBuilder` is converted into a `BoundNodes` object that the callback receives via `MatchResult::Nodes`.

### 46.5.2 Node Matchers

Node matchers select a particular AST node class. They are the outermost call in every matcher expression:

| Matcher | AST Node |
|---------|----------|
| `functionDecl(...)` | `FunctionDecl` |
| `cxxMethodDecl(...)` | `CXXMethodDecl` |
| `cxxRecordDecl(...)` | `CXXRecordDecl` |
| `varDecl(...)` | `VarDecl` |
| `fieldDecl(...)` | `FieldDecl` |
| `callExpr(...)` | `CallExpr` |
| `cxxMemberCallExpr(...)` | `CXXMemberCallExpr` |
| `binaryOperator(...)` | `BinaryOperator` |
| `ifStmt(...)` | `IfStmt` |
| `forStmt(...)` | `ForStmt` |
| `declRefExpr(...)` | `DeclRefExpr` |
| `memberExpr(...)` | `MemberExpr` |
| `typeAliasDecl(...)` | `TypeAliasDecl` |
| `namespaceDecl(...)` | `NamespaceDecl` |

### 46.5.3 Narrowing Matchers

Narrowing matchers filter nodes matching the structural type. The most frequently used:

```cpp
// Name predicates
hasName("foo")          // exact unqualified name
hasAnyName("a", "b")   // any of several names
matchesName("::std::.*") // regex on fully qualified name

// Access and linkage
isPublic(), isProtected(), isPrivate()
isStatic()
isInline()
isConstexpr()
isVirtual()
isPure()
isOverride()
isFinal()
isDeleted()
isDefaulted()
isExplicit()

// Type predicates
hasType(QualType matcher)
returns(QualType matcher)
hasParameter(unsigned N, matcher)
hasAnyParameter(matcher)
parameterCountIs(N)

// Source location
isExpansionInMainFile()
isExpansionInFileMatching("pattern")
isInStdNamespace()
isDefinition()
```

### 46.5.4 Structural (Traversal) Matchers

Structural matchers navigate the AST tree:

```cpp
// Single-child: succeeds if inner matcher matches exactly one child
has(matcher)

// Descendant: succeeds if inner matcher matches any node in the subtree
hasDescendant(matcher)

// forEach: like has(), but generates one result per matching child
forEach(matcher)

// forEachDescendant: generates one result per matching descendant
forEachDescendant(matcher)

// Relationship to declaration
hasDeclaration(matcher)   // for types, refs: points to a decl matching
to(matcher)               // for DeclRefExpr: refers to matching decl
```

The distinction between `has`/`hasDescendant` (short-circuit, return one match) and `forEach`/`forEachDescendant` (enumerate all matches) is critical for callback frequency: a node with three matching children fires one callback with `forEach` for each, but only one with `has`.

### 46.5.5 Logical Combinators

```cpp
allOf(m1, m2, m3, ...)   // all must match (AND)
anyOf(m1, m2, m3, ...)   // at least one must match (OR)
unless(m)                 // negation
eachOf(m1, m2, ...)       // fires a result for each sub-matcher that matches
```

`allOf` with one argument is a no-op. `anyOf` is evaluated left-to-right and short-circuits. `unless` wraps any matcher into its complement; `unless(isPublic())` matches non-public declarations.

### 46.5.6 Binding

Binding attaches a user-defined label to any matched node so it can be retrieved from `MatchResult::Nodes`:

```cpp
DeclarationMatcher M =
    functionDecl(
        hasName("process"),
        hasParameter(0, varDecl(hasType(pointerType())).bind("param")))
    .bind("fn");
```

The `.bind("label")` call is available on any `internal::Matcher<T>`. `MatchResult::Nodes` is a `BoundNodes` map:

```cpp
const FunctionDecl *FD = Result.Nodes.getNodeAs<FunctionDecl>("fn");
const VarDecl      *Param = Result.Nodes.getNodeAs<VarDecl>("param");
```

`getNodeAs<T>()` returns `nullptr` if the label was not bound (possible with `anyOf` branches where one alternative doesn't set all bindings).

### 46.5.7 Matcher Performance Considerations

The AST matcher engine traverses the entire translation unit once per registered matcher set, not once per individual matcher. `MatchFinder` internally builds a `MatchASTVisitor` (a `RecursiveASTVisitor` subclass) that visits each node once and tests all registered matchers at that node kind. This means that registering 50 matchers does not cause 50 traversals — all 50 run during the single traversal pass.

However, matcher complexity still matters. A matcher with a deeply nested `forEachDescendant` at the root forces the evaluator to recursively explore the entire sub-tree for every node of the outermost kind. Prefer `has()` when exactly one child should match; prefer `forEachDescendant` only when you need every matching descendant and can afford the combinatorial search.

The `setTraversalKind(TraversalKind TK)` method on `MatchFinder` (added in LLVM 12) controls how implicit nodes are treated:

```cpp
enum class TraversalKind {
  TK_AsIs,                        // visit all nodes including implicit
  TK_IgnoreUnlessSpelledInSource, // skip compiler-synthesized nodes
};
```

`TK_IgnoreUnlessSpelledInSource` is the default in `clang-tidy` checks and `clang-query`. It prevents matchers from firing on implicit constructor calls, cast expressions inserted by the compiler, and other AST nodes that have no textual representation in the source file. Analysis tools that want to see implicit nodes (e.g., a checker that tracks all `ImplicitCastExpr` nodes) must explicitly set `TK_AsIs`.

Attribute matchers and `TypeLoc` matchers are inherently more expensive than declaration or statement matchers because the AST matcher evaluator must enter separate traversal contexts to access them. When writing checks that only need declaration-level information, avoid adding `TypeLoc` matchers to the same `MatchFinder` pass.

---

## 46.6 `MatchFinder` and `MatchCallback`

### 46.6.1 Registration and Execution

`MatchFinder` (in `clang/ASTMatchers/ASTMatchFinder.h`) is the engine that applies registered matchers against an AST:

```cpp
class MatchFinder {
public:
  struct MatchResult {
    MatchResult(const BoundNodes &Nodes, clang::ASTContext *Context);
    const BoundNodes Nodes;
    clang::ASTContext * const Context;
    clang::SourceManager * const SourceManager;
  };

  class MatchCallback {
  public:
    virtual ~MatchCallback();
    virtual void run(const MatchResult &Result) = 0;
    virtual void onStartOfTranslationUnit() {}
    virtual void onEndOfTranslationUnit() {}
    virtual StringRef getID() const { return ""; }
  };

  void addMatcher(const DeclarationMatcher &, MatchCallback *);
  void addMatcher(const StatementMatcher &, MatchCallback *);
  void addMatcher(const TypeMatcher &, MatchCallback *);
  void addMatcher(const NestedNameSpecifierMatcher &, MatchCallback *);
  void addMatcher(const NestedNameSpecifierLocMatcher &, MatchCallback *);
  void addMatcher(const TypeLocMatcher &, MatchCallback *);
  void addMatcher(const CXXCtorInitializerMatcher &, MatchCallback *);
  void addMatcher(const TemplateArgumentLocMatcher &, MatchCallback *);
  void addMatcher(const AttrMatcher &, MatchCallback *);

  std::unique_ptr<clang::ASTConsumer> newASTConsumer();
  void matchAST(ASTContext &Context);
};
```

`newASTConsumer()` wraps the finder in an `ASTConsumer` appropriate for use with `FrontendActionFactory`. `matchAST()` is the direct entry point when you already have an `ASTContext` — as when operating on a cached `ASTUnit`.

### 46.6.2 A Complete Checker: `printf` Non-Literal Format Strings

The following tool detects calls to `printf` where the format argument is not a string literal — a common security anti-pattern:

```cpp
#include "clang/ASTMatchers/ASTMatchers.h"
#include "clang/ASTMatchers/ASTMatchFinder.h"
#include "clang/Tooling/CommonOptionsParser.h"
#include "clang/Tooling/Tooling.h"
#include "llvm/Support/raw_ostream.h"

using namespace clang;
using namespace clang::ast_matchers;
using namespace clang::tooling;

// Match: printf(X) where X is not a string literal
// Also catches fprintf(stream, X) via argument index 1
static const auto PrintfFormatMatcher =
    callExpr(
        callee(functionDecl(hasAnyName("printf", "fprintf", "sprintf"))),
        unless(hasArgument(0, ignoringParenImpCasts(stringLiteral()))),
        unless(hasArgument(1, ignoringParenImpCasts(stringLiteral()))))
    .bind("call");

class PrintfFormatChecker : public MatchFinder::MatchCallback {
public:
  void run(const MatchFinder::MatchResult &Result) override {
    const auto *Call = Result.Nodes.getNodeAs<CallExpr>("call");
    if (!Call) return;
    SourceManager &SM = *Result.SourceManager;
    SourceLocation Loc = Call->getBeginLoc();
    llvm::errs() << SM.getFilename(Loc) << ":"
                 << SM.getSpellingLineNumber(Loc) << ": "
                 << "warning: printf-style call with non-literal format\n";
  }
  void onEndOfTranslationUnit() override {
    llvm::errs() << "[done]\n";
  }
};

int main(int argc, const char **argv) {
  auto ExpectedParser =
      CommonOptionsParser::create(argc, argv, llvm::cl::getGeneralCategory());
  if (!ExpectedParser) { return 1; }
  ClangTool Tool(ExpectedParser->getCompilations(),
                 ExpectedParser->getSourcePathList());

  PrintfFormatChecker Checker;
  MatchFinder Finder;
  Finder.addMatcher(PrintfFormatMatcher, &Checker);
  return Tool.run(newFrontendActionFactory(&Finder).get());
}
```

The `ignoringParenImpCasts()` wrapper is essential: the format argument is often cast to `const char *` via an implicit `ImplicitCastExpr`, and without unwrapping it the matcher would never see the underlying `StringLiteral`.

The logic for `fprintf` illustrates a subtlety: the format string argument is at index 1 (after the `FILE *`), while for `printf` it is at index 0. The `unless(hasArgument(0, ...))` combined with `unless(hasArgument(1, ...))` works only because `printf` never has a `stringLiteral` at index 1 (its second argument, if present, is the vararg) and `fprintf` never has a `stringLiteral` at index 0 (which is the `FILE *`). A production checker would instead identify the format index per function via a compile-time table or the `format` attribute.

The `callee(functionDecl(...))` sub-matcher is important: it restricts the match to direct calls to named free functions. Without it, any expression of the right name appearing as a callee would match, including function pointer variables that happen to be stored in a variable named `printf`.

### 46.6.3 Lifetime Considerations

`MatchFinder` does not own `MatchCallback` pointers. The callbacks must outlive all `run()` calls. Since `ClangTool::run()` is synchronous and returns only after all TUs are processed, stack-allocating the callbacks is safe in the pattern above.

---

## 46.7 `Rewriter` and Source Transformations

### 46.7.1 `clang::Rewriter`

`clang::Rewriter` (in `clang/Rewrite/Core/Rewriter.h`) maintains a buffer per source file that records pending edits:

```cpp
class Rewriter {
public:
  explicit Rewriter(SourceManager &SM, const LangOptions &LO);

  bool InsertText(SourceLocation Loc, StringRef Str,
                  bool InsertAfter = true, bool indentNewLines = false);
  bool InsertTextBefore(SourceLocation Loc, StringRef Str);
  bool InsertTextAfter(SourceLocation Loc, StringRef Str);
  bool InsertTextAfterToken(SourceLocation Loc, StringRef Str);

  bool RemoveText(SourceLocation Start, unsigned Length,
                  RewriteOptions opts = RewriteOptions());
  bool RemoveText(CharSourceRange range, RewriteOptions opts = {});
  bool RemoveText(SourceRange range, RewriteOptions opts = {});

  bool ReplaceText(SourceLocation Start, unsigned OrigLength, StringRef NewStr);
  bool ReplaceText(CharSourceRange range, StringRef NewStr);
  bool ReplaceText(SourceRange range, StringRef NewStr);

  std::string getRewrittenText(CharSourceRange Range) const;
  std::string getRewrittenText(SourceRange Range) const;

  bool overwriteChangedFiles();
  int  getRangeSize(SourceRange SR, RewriteOptions opts = {}) const;
};
```

`overwriteChangedFiles()` writes the modified buffers to disk, creating backup files if configured. `getRewrittenText()` extracts the post-edit content of a source range without flushing to disk — useful for testing the effect of a transformation.

`Rewriter` maintains one `RewriteBuffer` per `FileID`. Each `RewriteBuffer` is a delta from the original file content: it stores a list of text segments (original ranges and inserted strings) in offset order. When `getRewrittenText()` or `overwriteChangedFiles()` is called, the buffer linearizes these segments into the final text. This design means that non-overlapping edits to the same file can be accumulated across multiple `MatchCallback::run()` invocations without conflict.

The `SourceRange` and `CharSourceRange` distinction is important for `Rewriter` use. `SourceRange` endpoints are token locations; the range extends to the end of the final token (end of identifier or keyword, not end of the following whitespace). `CharSourceRange::getTokenRange(SR)` converts a `SourceRange` to a character range that includes the last token's text. `CharSourceRange::getCharRange(begin, end)` treats `end` as a raw character offset (exclusive). Using the wrong range type leads to off-by-one errors where the trailing character of a replaced token is left in or an extra space is deleted.

For macro-expanded code, `Rewriter` operations require that the source range falls within a single `FileID` — you cannot replace a range that spans a macro boundary. `SourceManager::isMacroBodyExpansion()` and `SourceManager::isMacroArgExpansion()` let tools detect such ranges and skip them or expand them to the macro call site using `SourceManager::getExpansionRange()`.

### 46.7.2 `tooling::Replacement` and `tooling::Replacements`

`Rewriter` is stateful and tied to a single `SourceManager` and `CompilerInvocation`. For tools that process multiple TUs and need to aggregate edits, the `tooling::Replacement` struct is the portable format:

```cpp
// clang/Tooling/Core/Replacement.h
class Replacement {
public:
  Replacement(StringRef FilePath, unsigned Offset, unsigned Length,
              StringRef ReplacementText);
  Replacement(const SourceManager &Sources, SourceLocation Start,
              unsigned Length, StringRef ReplacementText);
  Replacement(const SourceManager &Sources, const CharSourceRange &Range,
              StringRef ReplacementText, const LangOptions &LO = {});

  StringRef getFilePath() const;
  unsigned  getOffset() const;
  unsigned  getLength() const;
  StringRef getReplacementText() const;
};
```

`tooling::Replacements` (in the same header) is an ordered set that detects overlapping edits and merges insertions at the same offset:

```cpp
class Replacements {
public:
  llvm::Error add(const Replacement &R);
  Replacements merge(const Replacements &ReplacesToMerge) const;
  bool empty() const;
  unsigned size() const;
  // Range of edits sorted by offset:
  const_iterator begin() const;
  const_iterator end() const;
};
```

`Replacements::add()` returns an `llvm::Error` if the replacement overlaps an existing one. Well-written tools collect replacements via `add()` and check for conflicts before applying.

`tooling::applyAllReplacements()` applies a `Replacements` set to a `Rewriter`:

```cpp
bool applyAllReplacements(const Replacements &Replaces, Rewriter &Rewrite);
// In-memory variant — returns the modified file content as a string:
llvm::Expected<std::string> applyAllReplacements(StringRef Code,
                                                  const Replacements &Replaces);
```

### 46.7.3 `TranslationUnitReplacements` and YAML Export

When replacements must be persisted to disk for later application, `tooling::TranslationUnitReplacements` (in `clang/Tooling/Core/Replacement.h`) provides a serialisable container:

```cpp
struct TranslationUnitReplacements {
  std::string MainSourceFile;
  std::vector<Replacement> Replacements;
};
```

The `clang/Tooling/ReplacementsYaml.h` header provides YAML serialization via `llvm::yaml::MappingTraits`. Tools emit YAML files (one per TU), and `clang-apply-replacements` aggregates and applies them.

### 46.7.4 `RefactoringTool`

`RefactoringTool` in `clang/Tooling/Refactoring.h` combines `ClangTool` with a `Replacements` map keyed by filename:

```cpp
class RefactoringTool : public ClangTool {
public:
  RefactoringTool(const CompilationDatabase &Compilations,
                  ArrayRef<std::string> SourcePaths, ...);

  std::map<std::string, Replacements> &getReplacements();
  bool applyAllReplacements(Rewriter &Rewrite);
  int runAndSave(FrontendActionFactory *ActionFactory);
};
```

`runAndSave()` calls `run()`, then applies all accumulated replacements and writes changed files. Callbacks obtain the tool's `Replacements` map by reference and `add()` into it during `MatchCallback::run()`.

---

## 46.8 `RecursiveASTVisitor` vs. Matchers

### 46.8.1 When to Use Each

`RecursiveASTVisitor<Derived>` (in `clang/AST/RecursiveASTVisitor.h`) is the low-level traversal engine. It implements the CRTP visitor pattern and provides a `Visit*` callback for every AST node type. Matchers internally use `RecursiveASTVisitor` to drive their execution.

Choose `RecursiveASTVisitor` when:
- You need control over traversal order (e.g., post-order vs. pre-order).
- You need to abort traversal early (return `false` from `Traverse*`).
- You maintain significant stateful context that accumulates across nodes.
- You need to traverse partial sub-trees without scanning the whole TU.

Choose matchers when:
- The query is compositional and can be expressed declaratively.
- Multiple independent patterns must be checked on the same AST.
- The tool will be extended with new checks (matchers compose better than visitor code).
- You want `clang-query` exploration during development.

### 46.8.2 `RecursiveASTVisitor` Interface

```cpp
template <typename Derived>
class RecursiveASTVisitor {
public:
  // Override to visit each node type:
  bool VisitDecl(Decl *D) { return true; }
  bool VisitStmt(Stmt *S) { return true; }
  bool VisitType(Type *T) { return true; }
  bool VisitTypeLoc(TypeLoc TL) { return true; }
  bool VisitAttr(Attr *A) { return true; }

  // Control which nodes are visited:
  bool shouldVisitTemplateInstantiations() const { return false; }
  bool shouldVisitImplicitCode() const { return false; }
  bool shouldVisitLambdaBody() const { return true; }
  bool shouldWalkTypesOfTypeLocs() const { return true; }

  // Manual entry points:
  bool TraverseDecl(Decl *D);
  bool TraverseStmt(Stmt *S, DataRecursionQueue *Queue = nullptr);
  bool TraverseType(QualType T);
  bool TraverseTypeLoc(TypeLoc TL);
};
```

`shouldVisitTemplateInstantiations()` defaults to `false`. When set to `true`, the visitor visits instantiated function and class templates in addition to the uninstantiated template definitions. This dramatically increases traversal volume in heavily templated code.

`shouldVisitImplicitCode()` controls whether compiler-synthesized code (implicit constructors, destructor calls inserted by the compiler) is visited. Most analysis tools leave this `false` to avoid reporting diagnostics on code the user did not write.

### 46.8.3 `WalkUpFrom` vs. `Visit` vs. `Traverse`

The naming convention in `RecursiveASTVisitor` encodes three distinct traversal phases:

- `Traverse*` — the full traversal entry point for a node kind; calls children. Override to skip sub-trees or inject pre/post logic.
- `WalkUpFrom*` — walks up the inheritance chain calling `Visit*` at each level. For `FunctionDecl`, `WalkUpFromFunctionDecl()` calls `VisitFunctionDecl()` then `WalkUpFromDeclaratorDecl()`, etc.
- `Visit*` — the leaf handler at a specific inheritance level. Override `VisitFunctionDecl()` to handle only `FunctionDecl` nodes without firing for subclasses.

In practice, most tools override `Visit*` methods and return `true` to continue traversal. Returning `false` from any `Traverse*` or `Visit*` method aborts the entire traversal (not just the current branch), which is useful for finding the first occurrence of something.

To skip a sub-tree without aborting, override `TraverseDecl()` (or the specific variant like `TraverseFunctionDecl()`) and return `true` without calling `RecursiveASTVisitor::TraverseDecl()`:

```cpp
bool TraverseFunctionDecl(FunctionDecl *FD) {
  if (FD->isTemplateInstantiation())
    return true;  // skip body, don't recurse
  return RecursiveASTVisitor<MyVisitor>::TraverseFunctionDecl(FD);
}
```

### 46.8.4 Combining Both Approaches

A common pattern is to use matchers to locate interesting nodes and `RecursiveASTVisitor` to analyze their sub-trees:

```cpp
class BodyAnalyzer : public RecursiveASTVisitor<BodyAnalyzer> {
public:
  bool VisitCallExpr(CallExpr *CE) {
    // collect every call in the function body
    Calls.push_back(CE);
    return true;
  }
  std::vector<CallExpr *> Calls;
};

class FunctionChecker : public MatchFinder::MatchCallback {
public:
  void run(const MatchFinder::MatchResult &Result) override {
    const auto *FD = Result.Nodes.getNodeAs<FunctionDecl>("fn");
    if (!FD || !FD->hasBody()) return;
    BodyAnalyzer Analyzer;
    Analyzer.TraverseDecl(const_cast<FunctionDecl *>(FD));
    for (CallExpr *CE : Analyzer.Calls) {
      // process each call
    }
  }
};
```

The matcher selects functions of interest; the visitor does targeted sub-tree analysis. This avoids traversing the entire TU twice while preserving the declarative selection logic of the matcher.

A less obvious but equally useful hybrid: use `RecursiveASTVisitor` as the outer driver for its traversal-order control, but evaluate a `DynTypedMatcher` at specific nodes using `DynTypedMatcher::matches(DynTypedNode::create(Node), Finder, &Builder)`. This requires constructing a `MatchFinder::MatchASTVisitor` manually or building a lightweight traversal harness, and is appropriate only for highly performance-sensitive scenarios where the standard `MatchFinder` overhead is measurable.

---

## 46.9 The Refactoring Pipeline

### 46.9.1 `clang-apply-replacements`

`clang-apply-replacements` (in `clang/tools/clang-apply-replacements/`) reads YAML files containing `TranslationUnitReplacements` objects, merges them, detects conflicts, and applies surviving replacements to source files. The merge step is essential for whole-project refactoring: when a rename touches a header included by many TUs, each TU produces a replacement for the same location. `clang-apply-replacements` deduplicates identical replacements and reports conflicts when two TUs attempt different changes to the same byte range.

Invocation:

```bash
# Run the tool to produce YAML files in /tmp/replacements/
my-rename-tool --export-fixes=/tmp/replacements/ -- src/

# Apply all collected replacements
clang-apply-replacements /tmp/replacements/
```

### 46.9.2 `clang-rename` as a Worked Example

`clang-rename` demonstrates the complete pipeline:

1. **Locate the declaration** at a cursor offset using `USR` (Unified Symbol Resolution) via `clang::index::generateUSRForDecl()`.
2. **Find all references** using a `MatchFinder` over a `DeclarationMatcher` that calls `isDefinition()` and a `StatementMatcher` for `declRefExpr(to(decl(...)))`.
3. **Emit `Replacement` objects** for each reference location.
4. **Apply via `RefactoringTool::runAndSave()`** or export to YAML for `clang-apply-replacements`.

The tool correctly handles cross-file renames through the compilation database: it processes all files that include the header containing the renamed symbol, ensuring consistent renaming across the entire project.

### 46.9.3 Conflict Detection

`tooling::Replacements::add()` returns `llvm::Error` when a new replacement overlaps an existing entry. Well-written tools handle this explicitly:

```cpp
auto Err = FileReplacements.add(
    Replacement(*Result.SourceManager, Range, NewName));
if (Err) {
  llvm::errs() << "Conflict: " << llvm::toString(std::move(Err)) << "\n";
}
```

The `Replacements` set merges consecutive insertions at the same location and rejects overlapping deletions or replacements, preserving the invariant that the set represents a well-defined transformation applicable in a single pass.

---

## 46.10 Practical Tool Examples

### 46.10.1 Finding TODO Comments with `PPCallbacks`

The preprocessor fires callbacks for comments when the `Preprocessor` is configured with `SetCommentRetentionState(true)`. A tool that finds all `// TODO` comments:

```cpp
class TodoFinder : public PPCallbacks {
public:
  explicit TodoFinder(SourceManager &SM) : SM(SM) {}
  void FileChanged(SourceLocation Loc, FileChangeReason Reason,
                   SrcMgr::CharacteristicKind FileType,
                   FileID PrevFID) override {}
  // CommentHandler is the correct hook for raw comments:
private:
  SourceManager &SM;
};

// CommentHandler (not PPCallbacks):
class TodoCommentHandler : public CommentHandler {
public:
  bool HandleComment(Preprocessor &PP, SourceRange Comment) override {
    SourceManager &SM = PP.getSourceManager();
    StringRef Text = Lexer::getSourceText(
        CharSourceRange::getTokenRange(Comment), SM, PP.getLangOpts());
    if (Text.contains("TODO")) {
      FullSourceLoc Loc(Comment.getBegin(), SM);
      llvm::outs() << SM.getFilename(Loc) << ":"
                   << Loc.getSpellingLineNumber() << ": " << Text << "\n";
    }
    return false;  // false = don't suppress the comment
  }
};
```

Register the handler via `CompilerInstance::getPreprocessor().addCommentHandler(&Handler)` inside `CreateASTConsumer()`. The `CommentHandler::HandleComment()` callback fires for every comment token encountered during preprocessing, giving access to the raw text before it is stripped.

### 46.10.2 Include-Guard Checker with `PPCallbacks`

```cpp
class IncludeGuardChecker : public PPCallbacks {
  SourceManager &SM;
  llvm::StringSet<> GuardedFiles;
public:
  explicit IncludeGuardChecker(SourceManager &SM) : SM(SM) {}

  void FileGuarded(const FileEntry *GuardedFile,
                   const IdentifierInfo *IfDefMacro,
                   SourceLocation IfDefLoc,
                   const IdentifierInfo *DefMacro,
                   SourceLocation DefMacroLoc) override {
    GuardedFiles.insert(GuardedFile->getName());
  }

  void EndOfMainFile() override {
    // Cross-reference against all included files via SourceManager
  }
};
```

`PPCallbacks::FileGuarded()` fires when the preprocessor detects the `#ifndef FOO_H / #define FOO_H` idiom (or `#pragma once`). Files not reported through this callback after inclusion can be flagged as missing include guards.

### 46.10.3 Whole-Project Analysis with `AllTUsToolExecutor`

For project-wide analysis, `AllTUsToolExecutor` (in `clang/Tooling/AllTUsExecution.h`) parallelizes execution across all TUs in the compilation database:

```cpp
#include "clang/Tooling/AllTUsExecution.h"

AllTUsToolExecutor Executor(OptionsParser.getCompilations(), /*ThreadCount=*/0);
auto Err = Executor.execute(newFrontendActionFactory(&Finder));
if (Err) {
  llvm::errs() << toString(std::move(Err)) << "\n";
  return 1;
}
```

`ThreadCount=0` uses `std::thread::hardware_concurrency()`. Results are aggregated through thread-safe containers; `MatchCallback::run()` must synchronize any shared state.

`AllTUsToolExecutor` implements the `ToolExecutor` interface defined in `clang/Tooling/Execution.h`. A companion `ToolResults` object (`InMemoryToolResults` or file-backed) collects key-value pairs emitted by callbacks via `ExecutionContext::reportResult(Key, Value)`. This provides a structured, thread-safe output channel without requiring callbacks to manage their own mutexes.

For very large codebases where even the compilation database enumeration is expensive, `StandaloneToolExecutor` (in `clang/Tooling/StandaloneExecution.h`) processes a pre-filtered list of files and reuses file system state across TUs, minimizing repeated file system calls.

The `--filter` flag supported by `AllTUsToolExecutor` accepts a regex applied to source file paths before processing, enabling targeted runs over a subdirectory without modifying the compilation database.

---

## 46.11 `clang-query` REPL

`clang-query` (in `clang/tools/clang-query/`) is an interactive shell for developing and testing AST matcher expressions. It compiles a source file on startup and then accepts matcher commands:

```
$ clang-query --extra-arg=-std=c++20 myfile.cpp --
clang-query> match functionDecl(hasName("foo"))
```

Key commands:

| Command | Effect |
|---------|--------|
| `match <matcher>` | Execute matcher, print results |
| `let <name> <matcher>` | Bind matcher to a name for reuse |
| `set output dump` | Print full AST node dump for each match |
| `set output diag` | Print diagnostic-style location for each match |
| `set output print` | Print source text of each match |
| `set output detailed_ast` | Print full AST subtree with types |
| `set traversal IgnoreUnlessSpelledInSource` | Skip implicit nodes |
| `enable output dump` | Adds dump mode alongside current output |

`let` bindings allow incremental construction of complex matchers:

```
clang-query> let baseClass cxxRecordDecl(hasName("Base"))
clang-query> match cxxRecordDecl(isDerivedFrom(baseClass))
```

`clang-query` uses the same `DynTypedMatcher` infrastructure as the production matcher engine, making it a reliable predictor of production behavior. The `--extra-arg` flag injects additional compilation flags, essential when the source requires a specific language standard or project-specific macros.

The `set traversal IgnoreUnlessSpelledInSource` command in `clang-query` maps directly to `TraversalKind::TK_IgnoreUnlessSpelledInSource`. When developing a `clang-tidy` check (which uses this traversal mode by default), always set the same mode in `clang-query` to get matching behavior:

```
clang-query> set traversal IgnoreUnlessSpelledInSource
clang-query> match cxxConstructExpr(hasType(cxxRecordDecl(hasName("Widget"))))
```

Without this, `clang-query` might report matches on implicit constructor calls that the clang-tidy check would never see.

The `--extra-args-before` and `--extra-args` options serve different purposes: `--extra-args-before` prepends arguments before the compilation database entry (useful for overriding includes), while `--extra-args` appends them (useful for adding feature macros). The same distinction applies to `getInsertArgumentAdjuster()` with `ArgumentInsertPosition::BEGIN` vs. `ArgumentInsertPosition::END`.

For matcher development workflow, the recommended sequence is:

1. Start with `clang-query` to explore the AST: `set output dump` to see the full node structure, then narrow down the matcher expression.
2. Copy the validated matcher expression into a `clang-tidy` check or standalone tool.
3. Run `runToolOnCodeWithArgs` unit tests to verify edge cases.
4. Deploy with `ClangTool` over the full compilation database.

---

## 46.12 libclang C API

### 46.12.1 Design Philosophy

libclang (`clang/include/clang-c/Index.h`) exposes a C API over a subset of Clang's capabilities. It maintains a stable ABI across LLVM releases, making it the preferred foundation for language bindings: Python's `libclang` module (`clang.cindex`), Rust's `clang-sys` crate, and editors that embed Clang for syntax highlighting all use this interface.

The stable ABI comes at a cost: the API exposes an opaque `CXCursor` handle rather than typed C++ pointers, provides no direct access to the internal AST node hierarchy, and cannot be extended by users without rebuilding the library.

### 46.12.2 Core Types

```c
// Opaque handle to a parsed TU
typedef struct CXTranslationUnitImpl *CXTranslationUnit;

// Tagged union cursor: represents any AST node
typedef struct {
  enum CXCursorKind kind;
  int xdata;
  const void *data[3];
} CXCursor;

// Type information
typedef struct {
  enum CXTypeKind kind;
  void *data[2];
} CXType;
```

`CXCursor` is a value type (passed and returned by value) that identifies a specific node. `kind` is a `CXCursorKind` enum covering declarations, statements, expressions, and preprocessing tokens. The `data` fields hold internal pointers; their layout is not part of the public API.

### 46.12.3 Parsing and Traversal

```c
// Parse a translation unit from source files
CXTranslationUnit
clang_parseTranslationUnit(CXIndex CIdx,
                           const char *source_filename,
                           const char *const *command_line_args,
                           int num_command_line_args,
                           struct CXUnsavedFile *unsaved_files,
                           unsigned num_unsaved_files,
                           unsigned options);

// Visitor callback type
typedef enum CXChildVisitResult
(*CXCursorVisitor)(CXCursor cursor, CXCursor parent, CXClientData client_data);

// Traverse the cursor tree
unsigned clang_visitChildren(CXCursor parent,
                             CXCursorVisitor visitor,
                             CXClientData client_data);
```

`CXChildVisitResult` controls traversal:
- `CXChildVisit_Break` — stop traversal entirely
- `CXChildVisit_Continue` — skip children, continue at siblings
- `CXChildVisit_Recurse` — descend into children

A minimal cursor printer:

```c
#include <clang-c/Index.h>
#include <stdio.h>

enum CXChildVisitResult printCursor(CXCursor cursor, CXCursor parent,
                                    CXClientData data) {
  CXString spelling = clang_getCursorSpelling(cursor);
  CXSourceLocation loc = clang_getCursorLocation(cursor);
  unsigned line, col;
  CXFile file;
  clang_getSpellingLocation(loc, &file, &line, &col, NULL);
  CXString filename = clang_getFileName(file);
  printf("%s:%u:%u %s\n",
         clang_getCString(filename), line, col,
         clang_getCString(spelling));
  clang_disposeString(spelling);
  clang_disposeString(filename);
  return CXChildVisit_Recurse;
}

int main(void) {
  CXIndex index = clang_createIndex(0, 0);
  CXTranslationUnit TU = clang_parseTranslationUnit(
      index, "input.cpp", NULL, 0, NULL, 0,
      CXTranslationUnit_DetailedPreprocessingRecord);
  clang_visitChildren(clang_getTranslationUnitCursor(TU),
                      printCursor, NULL);
  clang_disposeTranslationUnit(TU);
  clang_disposeIndex(index);
}
```

`CXString` must be freed with `clang_disposeString()`. Memory management discipline is the primary burden of the C API.

### 46.12.4 Code Completion

```c
CXCodeCompleteResults *
clang_codeCompleteAt(CXTranslationUnit TU,
                     const char *complete_filename,
                     unsigned complete_line,
                     unsigned complete_column,
                     struct CXUnsavedFile *unsaved_files,
                     unsigned num_unsaved_files,
                     unsigned options);
```

`clang_codeCompleteAt()` re-parses the TU with a truncated source file and returns a `CXCodeCompleteResults` array containing completion candidates. Each result has a `CXCompletionString` with priority, availability, and typed chunks. This is the API that Vim's `YouCompleteMe` and similar plugins used historically; more recent IDE integrations prefer LSP (clangd), which internally calls the same infrastructure via C++.

### 46.12.5 libclang vs. libtooling

| Dimension | libclang | libtooling |
|-----------|----------|------------|
| ABI stability | Stable across LLVM releases | No guarantee |
| Language bindings | Python, Rust, OCaml, Go, ... | C++ only |
| AST fidelity | Partial (via `CXCursor`) | Full (typed C++ classes) |
| Matcher DSL | Not available | Full `ast_matchers` |
| Rewriting | Not available | `Rewriter`, `Replacements` |
| Performance | Extra overhead from opaque types | Direct C++ access |
| Use cases | IDE bindings, syntax highlighting, basic navigation | Analysis, refactoring, linting tools |

The choice is straightforward: use libclang for stable, language-binding-friendly integration; use libtooling for production-grade tools that require full AST access and source modification capabilities.

### 46.12.6 Python `libclang` Bindings

The Python `libclang` module (`clang.cindex`) wraps the C API with a Pythonic interface. The module ships in the LLVM source tree at `clang/bindings/python/clang/cindex.py`. Its `TranslationUnit`, `Cursor`, and `Type` classes correspond directly to `CXTranslationUnit`, `CXCursor`, and `CXType`:

```python
import clang.cindex as clang

index = clang.Index.create()
tu = index.parse('example.cpp', args=['-std=c++20'])

def find_functions(node):
    if node.kind == clang.CursorKind.FUNCTION_DECL:
        print(f"{node.spelling} at {node.location.file}:{node.location.line}")
    for child in node.get_children():
        find_functions(child)

find_functions(tu.cursor)
```

`Cursor.get_children()` returns an iterator over child cursors, the Python equivalent of `clang_visitChildren()`. The Python binding retains the tree structure as Python objects rather than requiring a C callback, making recursive traversal more natural. However, creating large numbers of `Cursor` Python objects has measurable overhead; performance-sensitive scripts should minimize cursor creation by filtering at the `kind` level before inspecting spelling or location.

The Python bindings require that `libclang.so` (or `libclang.dylib`) is loadable, configured via `Config.set_library_file()` or by setting `LD_LIBRARY_PATH` to point to the LLVM 22 library directory. On systems where multiple LLVM versions are installed, binding-version mismatches cause silent failures or undefined behavior; always match the Python binding file version to the shared library.

---

## Chapter Summary

- `ClangTool` drives Clang per-file over a `CompilationDatabase`, with `ArgumentsAdjuster` chains for flag injection; `runToolOnCode()` enables unit testing without a database.
- `JSONCompilationDatabase` reads `compile_commands.json`; `InterpolatingCompilationDatabase` synthesizes entries for headers; `FixedCompilationDatabase` provides a uniform command line for ad-hoc use.
- `ASTFrontendAction::CreateASTConsumer()` receives a fully initialized `CompilerInstance`; `MultiplexConsumer` fans callbacks to multiple independent consumers enabling clang-tidy's per-check architecture.
- `ASTUnit` retains the parsed AST with preamble precompilation for incremental reparse; clangd and libclang build on `ASTUnit::LoadFromCommandLine()`.
- The AST matcher DSL uses typed `internal::Matcher<T>` wrappers; `has` vs `forEach` and `hasDescendant` vs `forEachDescendant` differ in whether they produce one or many matches per structural node.
- `MatchFinder` drives matcher evaluation over an `ASTContext`; `MatchCallback::onEndOfTranslationUnit()` is the correct hook for per-TU aggregation.
- `clang::Rewriter` provides direct source editing; `tooling::Replacements` is the portable, mergeable format for multi-file refactoring; `tooling::applyAllReplacements()` bridges the two.
- `RecursiveASTVisitor` provides CRTP-based full traversal with `shouldVisitTemplateInstantiations()` and `shouldVisitImplicitCode()` guards; combining it with matchers for sub-tree analysis is a common and efficient pattern.
- `clang-apply-replacements` aggregates YAML replacement files from parallel tool runs and resolves conflicts; `clang-rename` demonstrates the complete pipeline.
- `clang-query` is the interactive development REPL for AST matcher expressions; `let` bindings enable incremental composition of complex queries.
- libclang provides a stable C ABI for language bindings at the cost of partial AST fidelity and no matcher or rewriting support.

---

*Cross-references:*
- [Chapter 36 — The Clang AST in Depth](../part-05-clang-frontend/ch36-clang-ast-in-depth.md) — `ASTContext`, `Decl`, `Stmt`, `QualType` node types used throughout this chapter
- [Chapter 38 — Code Completion and clangd Foundations](../part-05-clang-frontend/ch38-clangd-foundations.md) — clangd's use of `ASTUnit` and `InterpolatingCompilationDatabase`
- [Chapter 47 — clangd, clang-tidy, clang-format, clang-refactor](ch47-clangd-clang-tidy-clang-format.md) — production tools built on the libtooling stack

*Reference links:*
- [`clang/include/clang/Tooling/Tooling.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Tooling/Tooling.h)
- [`clang/include/clang/Tooling/CompilationDatabase.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Tooling/CompilationDatabase.h)
- [`clang/include/clang/ASTMatchers/ASTMatchers.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/ASTMatchers/ASTMatchers.h)
- [`clang/include/clang/ASTMatchers/ASTMatchFinder.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/ASTMatchers/ASTMatchFinder.h)
- [`clang/include/clang/Rewrite/Core/Rewriter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Rewrite/Core/Rewriter.h)
- [`clang/include/clang/Tooling/Core/Replacement.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Tooling/Core/Replacement.h)
- [`clang/include/clang/Tooling/Refactoring.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Tooling/Refactoring.h)
- [`clang/include/clang/AST/RecursiveASTVisitor.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/RecursiveASTVisitor.h)
- [`clang/include/clang/Frontend/ASTUnit.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/ASTUnit.h)
- [`clang/include/clang-c/Index.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang-c/Index.h)
- [JSON Compilation Database Format Specification](https://clang.llvm.org/docs/JSONCompilationDatabase.html)
- [AST Matcher Reference](https://clang.llvm.org/docs/LibASTMatchersReference.html)
- [Clang LibTooling tutorial](https://clang.llvm.org/docs/LibTooling.html)
