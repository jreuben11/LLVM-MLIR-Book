# Ch197 — Clang Plugin System

*Part XXIX — Compiler Tooling, Kernel Integration, and Binary Analysis*

Clang's plugin system lets you extend the compiler itself without forking the source tree. A plugin runs inside the compiler process, shares its AST, diagnostic engine, preprocessor, and code-generation pipeline. The result is richer than anything a separate libtooling binary can achieve: warnings integrated into a normal compilation, custom pragmas that influence code generation, IR-level instrumentation injected automatically during every build. This chapter covers the complete plugin surface area—from the self-registration macro to IR-level hooks—and delivers two fully worked examples: a null-safety checker and a function-entry instrumentation pass.

---

## 1. The Plugin Loading Mechanism

### 1.1 `-fplugin` versus `-Xclang -load`

Clang exposes two command-line paths for plugin loading.

The **driver-facing** form is:

```bash
clang -fplugin=./libmyplugin.so -fplugin-arg-myplugin-some-arg source.cpp
```

The driver translates `-fplugin=` into `-Xclang -load` followed by `-Xclang -plugin` and forwards any `-fplugin-arg-NAME-VALUE` arguments as `-Xclang -plugin-arg-NAME VALUE`. This path is available in every driver invocation.

The **cc1-facing** form bypasses the driver entirely:

```bash
clang -cc1 -load ./libmyplugin.so -plugin myplugin source.cpp
```

Use `-cc1 -load` during development and testing; it exercises the plugin in the same way the driver does but without driver-side argument transformation. In production build systems, prefer `-fplugin` so the flag survives driver invocations that rearrange cc1 arguments.

Both forms call `llvm::sys::DynamicLibrary::LoadLibraryPermanently` at startup. The shared library's global constructors run immediately, invoking the `FrontendPluginRegistry::Add<>` self-registration described in Section 4.

### 1.2 Multiple Plugins and Ordering

Multiple `-fplugin` arguments are accepted; plugins are loaded in order and all of them are active simultaneously. There is no isolation between plugins: they share the same `CompilerInstance`, `ASTContext`, and `DiagnosticsEngine`. A plugin that calls `CompilerInstance::getDiagnostics().setClient(...)` replaces the diagnostic consumer for every subsequent plugin.

---

## 2. `PluginASTAction` and the Action Type System

### 2.1 Class Hierarchy

The plugin entry point is
[`PluginASTAction`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/FrontendAction.h#L269),
which inherits from `ASTFrontendAction`. You must override two pure-virtual members:

```cpp
#include "clang/Frontend/FrontendAction.h"
#include "clang/Frontend/FrontendPluginRegistry.h"
#include "clang/AST/ASTConsumer.h"

namespace {

class MyAction : public clang::PluginASTAction {
public:
  // Return the consumer that processes the AST.
  std::unique_ptr<clang::ASTConsumer>
  CreateASTConsumer(clang::CompilerInstance &CI,
                    llvm::StringRef InFile) override;

  // Parse any -plugin-arg-NAME-... arguments.
  bool ParseArgs(const clang::CompilerInstance &CI,
                 const std::vector<std::string> &args) override {
    // Return false to abort; CI.getDiagnostics() is available for errors.
    return true;
  }

  // Control when this action fires.
  ActionType getActionType() override {
    return AddAfterMainAction;
  }
};

} // anonymous namespace
```

### 2.2 `ActionType` Enum

The `ActionType` enum controls the plugin's relationship to Clang's main action (normally `EmitObjAction` or `EmitLLVMOnlyAction`):

| `ActionType` value | Behaviour |
|---|---|
| `CmdlineBeforeMainAction` | Run before main action only when `-plugin` is on command line |
| `CmdlineAfterMainAction` | Run after main action only when `-plugin` is on command line (default) |
| `ReplaceAction` | Completely replace the main action |
| `AddBeforeMainAction` | Always run before main action, regardless of `-plugin` |
| `AddAfterMainAction` | Always run after main action, regardless of `-plugin` |

`AddBeforeMainAction` and `AddAfterMainAction` are useful for plugins delivered via `-fplugin` that should fire on every compilation without requiring an explicit `-plugin` argument. Analysis-only plugins that emit warnings should use `AddAfterMainAction` so the main compilation has already succeeded before extra diagnostics appear.

### 2.3 `CompilerInstance` Access

`CreateASTConsumer` receives a `CompilerInstance &CI`. Everything the plugin needs flows from this object:

```cpp
#include "clang/Frontend/CompilerInstance.h"

std::unique_ptr<clang::ASTConsumer>
MyAction::CreateASTConsumer(clang::CompilerInstance &CI,
                            llvm::StringRef InFile) {
  auto &Diag  = CI.getDiagnostics();        // DiagnosticsEngine
  auto &PP    = CI.getPreprocessor();       // Preprocessor
  auto &SM    = CI.getSourceManager();      // SourceManager
  auto &Ctx   = CI.getASTContext();         // ASTContext (valid after parsing)
  auto &LO    = CI.getLangOpts();           // LangOptions
  auto &TO    = CI.getTarget();             // TargetInfo
  return std::make_unique<MyConsumer>(CI);
}
```

`ASTContext` is not yet populated when `CreateASTConsumer` is called; it becomes valid only when `ASTConsumer::Initialize` fires. Store the `CompilerInstance` reference or the specific sub-objects you need as consumer member variables.

---

## 3. `ASTConsumer` Callbacks

### 3.1 Callback Lifecycle

[`ASTConsumer`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/ASTConsumer.h)
provides a set of virtual callbacks that Clang calls at defined points in the compilation:

| Callback | When called |
|---|---|
| `Initialize(ASTContext &)` | Once after parsing begins; `ASTContext` is valid here |
| `HandleTopLevelDecl(DeclGroupRef)` | After each top-level declaration group is parsed |
| `HandleInlineMethodDefinition(CXXMethodDecl *)` | When an inline method body is parsed |
| `HandleTranslationUnit(ASTContext &)` | After the entire translation unit has been parsed |
| `HandleTagDeclDefinition(TagDecl *)` | After a `struct`/`class`/`enum` definition |
| `HandleCXXStaticMemberVarInstantiation(VarDecl *)` | After a static data member is instantiated |

For whole-TU analysis, override `HandleTranslationUnit`. For streaming analysis that avoids buffering the entire AST, override `HandleTopLevelDecl` and return `false` to abort early.

```cpp
#include "clang/AST/ASTConsumer.h"
#include "clang/AST/ASTContext.h"

class MyConsumer : public clang::ASTConsumer {
  clang::CompilerInstance &CI;

public:
  explicit MyConsumer(clang::CompilerInstance &CI) : CI(CI) {}

  void Initialize(clang::ASTContext &Ctx) override {
    // Called before any decls arrive. Good place to register
    // custom diagnostic IDs.
  }

  bool HandleTopLevelDecl(clang::DeclGroupRef DG) override {
    for (clang::Decl *D : DG) {
      // Process D as it arrives, before the next function is parsed.
    }
    return true; // Return false to abort compilation.
  }

  void HandleTranslationUnit(clang::ASTContext &Ctx) override {
    // Process the complete AST. The RecursiveASTVisitor is
    // typically invoked from here.
  }
};
```

### 3.2 `RecursiveASTVisitor` Inside a Plugin

AST traversal works identically inside a plugin and inside a libtooling tool. The visitor is most commonly instantiated from `HandleTranslationUnit`:

```cpp
#include "clang/AST/RecursiveASTVisitor.h"

class MyVisitor : public clang::RecursiveASTVisitor<MyVisitor> {
  clang::DiagnosticsEngine &Diags;
public:
  explicit MyVisitor(clang::DiagnosticsEngine &D) : Diags(D) {}

  // Override Visit* to intercept specific node types.
  bool VisitFunctionDecl(clang::FunctionDecl *FD) {
    // Called for every function declaration in the TU.
    return true; // Return false to stop traversal.
  }
};

// In HandleTranslationUnit:
void HandleTranslationUnit(clang::ASTContext &Ctx) override {
  MyVisitor V(CI.getDiagnostics());
  V.TraverseDecl(Ctx.getTranslationUnitDecl());
}
```

For cross-reference analysis (e.g., following call targets), use `TraverseDecl` on the translation unit declaration rather than traversing individual top-level decls, so that template instantiations and implicit definitions are included.

---

## 4. Self-Registration: `FrontendPluginRegistry::Add<>`

The
[`FrontendPluginRegistry`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/FrontendPluginRegistry.h)
is a specialisation of `llvm::Registry<PluginASTAction>`. Registration occurs via a file-scope object whose constructor fires when the shared library is loaded:

```cpp
static clang::FrontendPluginRegistry::Add<MyAction>
    X("myplugin",        // the name passed to -plugin
      "description of what myplugin does");
```

The `Add<>` template instantiates a registry entry that points to a factory function returning `new MyAction()`. Clang iterates all registered plugins after loading; those whose names match the `-plugin` argument (or all `AddBefore/AfterMainAction` plugins) are instantiated.

Because the registry uses a linked-list of static nodes, multiple plugins can be registered in a single shared library—each with its own file-scope `Add<>` object.

---

## 5. Custom Diagnostics

### 5.1 `getCustomDiagID` and `Report`

Clang's diagnostic system does not require TableGen entries for plugin diagnostics. The `DiagnosticsEngine::getCustomDiagID` method allocates a new diagnostic ID at runtime:

```cpp
#include "clang/Basic/Diagnostic.h"

unsigned WarnNullArg;  // store as member variable

void Initialize(clang::ASTContext &Ctx) override {
  auto &Diags = CI.getDiagnostics();
  WarnNullArg = Diags.getCustomDiagID(
      clang::DiagnosticsEngine::Warning,
      "argument %0 passed to nonnull parameter '%1' may be null");
}
```

Emit the diagnostic with `Report`:

```cpp
auto &Diags = CI.getDiagnostics();
auto DB = Diags.Report(CallSite, WarnNullArg);
DB << ArgIndex << ParamName;
```

The `DiagnosticBuilder` returned by `Report` accepts `<<` to bind arguments in positional order (`%0`, `%1`, …). Arguments may be `int`, `llvm::StringRef`, `clang::QualType`, `clang::SourceRange`, and others.

### 5.2 `FixItHint`

Fix-it hints attach suggested edits to a diagnostic. The three factory methods cover the common cases:

```cpp
// Suggest inserting text before a source location.
DB << clang::FixItHint::CreateInsertion(Loc, "/* null guard */");

// Suggest removing a source range.
DB << clang::FixItHint::CreateRemoval(BadRange);

// Suggest replacing a source range with new text.
DB << clang::FixItHint::CreateReplacement(BadRange, ReplacementText);
```

Fix-it hints are displayed by clang-format's `-fix` mode and honoured by `clang-tidy --fix`. They are metadata only; the compiler does not apply them.

### 5.3 `DiagnosticConsumer` Replacement

A plugin can replace the diagnostic consumer entirely to redirect diagnostics to a custom sink (e.g., a JSON file or a CI service):

```cpp
#include "clang/Basic/DiagnosticOptions.h"

class JsonDiagConsumer : public clang::DiagnosticConsumer {
public:
  void HandleDiagnostic(clang::DiagnosticsEngine::Level Level,
                        const clang::Diagnostic &Info) override {
    // Serialise Info to JSON.
  }
};

// In ParseArgs or BeginSourceFileAction:
CI.getDiagnostics().setClient(new JsonDiagConsumer(),
                              /*ShouldOwnClient=*/true);
```

`setClient` replaces the current client; the previous client is deleted if Clang owns it. Call `setClient` before returning from `CreateASTConsumer`—or override `BeginSourceFileAction`—to capture all diagnostics from the compilation.

---

## 6. `PragmaHandler` for Custom Pragmas

### 6.1 Registering a Handler

Custom pragmas must be registered before parsing begins. Override `BeginSourceFileAction` and call `AddPragmaHandler`:

```cpp
#include "clang/Lex/Preprocessor.h"
#include "clang/Lex/Pragma.h"

class NoInstrumentPragma : public clang::PragmaHandler {
public:
  NoInstrumentPragma() : clang::PragmaHandler("no_instrument") {}

  void HandlePragma(clang::Preprocessor &PP,
                    clang::PragmaIntroducer Introducer,
                    clang::Token &FirstToken) override {
    // Consume tokens until end of line.
    // Set a flag that the next function should not be instrumented.
  }
};

class MyAction : public clang::PluginASTAction {
  NoInstrumentPragma *PH = nullptr;

  bool BeginSourceFileAction(clang::CompilerInstance &CI) override {
    PH = new NoInstrumentPragma();
    // Second arg is optional namespace; null = top-level pragma.
    CI.getPreprocessor().AddPragmaHandler(PH);
    return true;
  }

  void EndSourceFileAction() override {
    if (PH)
      CI.getPreprocessor().RemovePragmaHandler(PH);
  }
};
```

The namespace mechanism allows `#pragma myplugin no_instrument` syntax by registering under the `"myplugin"` namespace:

```cpp
CI.getPreprocessor().AddPragmaHandler("myplugin", PH);
```

### 6.2 Token Consumption in `HandlePragma`

The preprocessor cursor is positioned at the first token after the pragma name. Consume tokens with `PP.Lex`:

```cpp
void HandlePragma(clang::Preprocessor &PP,
                  clang::PragmaIntroducer Introducer,
                  clang::Token &Tok) override {
  PP.Lex(Tok);  // Move to first argument token.
  if (Tok.is(clang::tok::eod)) return;  // Empty pragma body.

  if (Tok.is(clang::tok::identifier)) {
    llvm::StringRef Name = Tok.getIdentifierInfo()->getName();
    // Act on Name.
  }

  // Consume until end-of-directive.
  while (Tok.isNot(clang::tok::eod))
    PP.Lex(Tok);
}
```

---

## 7. IR-Level Hooks: Accessing the `llvm::Module`

### 7.1 The `CodeGenerator` and Post-Codegen Processing

For IR-level work, the cleanest approach is a `WrapperFrontendAction` that wraps the standard codegen action and post-processes the resulting `llvm::Module`. An alternative used in several in-tree examples is writing a wrapper `ASTConsumer` that chains a `CodeGenerator`:

```cpp
#include "clang/CodeGen/ModuleBuilder.h"
#include "clang/CodeGen/CodeGenAction.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/IRBuilder.h"

class InstrumentConsumer : public clang::ASTConsumer {
  clang::CompilerInstance &CI;
  std::unique_ptr<clang::CodeGenerator> CG;

public:
  InstrumentConsumer(clang::CompilerInstance &CI, llvm::StringRef InFile)
      : CI(CI),
        CG(clang::CreateLLVMCodeGen(
               CI, InFile, CI.getASTContext().getLangOpts(),
               CI.getASTContext().getTargetInfo())) {}

  void Initialize(clang::ASTContext &Ctx) override {
    CG->Initialize(Ctx);
  }

  bool HandleTopLevelDecl(clang::DeclGroupRef DG) override {
    return CG->HandleTopLevelDecl(DG);
  }

  void HandleTranslationUnit(clang::ASTContext &Ctx) override {
    // Run normal codegen first.
    CG->HandleTranslationUnit(Ctx);

    // Now walk the generated module.
    llvm::Module *M = CG->GetModule();
    if (!M) return;
    instrumentModule(*M);
  }

private:
  void instrumentModule(llvm::Module &M);
};
```

`CodeGenerator::GetModule` returns the in-progress `llvm::Module`. The module is fully formed at this point: all functions have IR bodies and global variables have initialisers.

### 7.2 Walking and Modifying the Module

Once you hold the `llvm::Module *`, standard LLVM IR APIs apply without restriction:

```cpp
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/BasicBlock.h"

void InstrumentConsumer::instrumentModule(llvm::Module &M) {
  llvm::LLVMContext &Ctx = M.getContext();

  // Declare the runtime hook: void __my_instrument(const char *name)
  auto *VoidTy = llvm::Type::getVoidTy(Ctx);
  auto *I8PtrTy = llvm::PointerType::getUnqual(Ctx);
  auto *FTy = llvm::FunctionType::get(VoidTy, {I8PtrTy}, /*isVarArg=*/false);
  llvm::FunctionCallee Hook =
      M.getOrInsertFunction("__my_instrument", FTy);

  for (llvm::Function &F : M) {
    if (F.isDeclaration()) continue;  // Skip declarations.
    if (F.getName().starts_with("__my_")) continue;  // Avoid recursion.

    // Insert call at the very start of the entry block.
    llvm::BasicBlock &Entry = F.getEntryBlock();
    llvm::IRBuilder<> Builder(&*Entry.getFirstInsertionPt());

    llvm::Constant *FuncName =
        Builder.CreateGlobalStringPtr(F.getName(), ".instrument.name");
    Builder.CreateCall(Hook, {FuncName});
  }
}
```

This pattern gives you full access to every LLVM API: you can add globals, split blocks, clone functions, or run arbitrary analyses.

### 7.3 Using `BackendConsumer` and the Emit Hook

`CodeGenAction::BEConsumer` is a public pointer to the `BackendConsumer` object that manages the LLVM backend pipeline. For plugins delivered as `PluginASTAction` (rather than a standalone tool), the backend consumer is not directly accessible. The approach above—wrapping the `CodeGenerator` manually—is the standard pattern used by the in-tree `AnnotateFunctions` example in
[`clang/examples/AnnotateFunctions/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/examples/AnnotateFunctions/).

For pass-level (rather than module-level) IR work, the `-fplugin` path for LLVM pass plugins is separate and uses `llvm::PassPlugin`—see [Ch60 — Writing a Pass](../part-10-analysis-middle-end/ch60-writing-a-pass.md).

---

## 8. Custom Attribute Plugins

### 8.1 The `annotate` Attribute as a Simple Alternative

The path of least resistance for custom per-declaration metadata is the built-in `__attribute__((annotate("mykey")))`. No plugin is required: Clang parses it unconditionally, stores it as an `AnnotateAttr`, and a plugin's visitor can query it:

```cpp
if (auto *AA = FD->getAttr<clang::AnnotateAttr>()) {
  if (AA->getAnnotation() == "my_noinstrument")
    return; // Skip this function.
}
```

This approach covers 90% of practical attribute use cases with zero Clang patching.

### 8.2 `ParsedAttr` and Plugin Attribute Registration

For attributes that require argument parsing, spelling validation, or Sema-level checks, Clang 12+ supports attribute plugins via `ParsedAttrInfo`. Implement `clang::ParsedAttrInfo` and register it:

```cpp
#include "clang/Sema/ParsedAttr.h"
#include "clang/Sema/SemaDiagnostic.h"

namespace {

struct MyAttrInfo : public clang::ParsedAttrInfo {
  MyAttrInfo() {
    // Describe the attribute to the parser.
    static constexpr Spelling Spellings[] = {
        {clang::ParsedAttr::AS_GNU,  "my_attr"},
        {clang::ParsedAttr::AS_CXX11, "my_ns::my_attr"},
    };
    NumArgs = 0;
    OptArgs = 0;
    this->Spellings = Spellings;
  }

  bool diagAppertainsToDecl(clang::Sema &S,
                            const clang::ParsedAttr &Attr,
                            const clang::Decl *D) const override {
    if (!llvm::isa<clang::FunctionDecl>(D)) {
      S.Diag(Attr.getLoc(), clang::diag::warn_attribute_wrong_decl_type_str)
          << Attr << Attr.isRegularKeywordAttribute()
          << "functions";
      return false;
    }
    return true;
  }

  AttrHandling handleDeclAttribute(clang::Sema &S,
                                   clang::Decl *D,
                                   const clang::ParsedAttr &Attr) const override {
    D->addAttr(clang::AnnotateAttr::Create(
        S.Context, "my_attr", nullptr, 0, Attr));
    return AttributeApplied;
  }
};

} // anonymous namespace

static clang::ParsedAttrInfoRegistry::Add<MyAttrInfo>
    Y("my_attr", "My custom attribute");
```

The `ParsedAttrInfoRegistry` follows the same `llvm::Registry` self-registration pattern as `FrontendPluginRegistry`. At attribute parsing time Clang queries all registered `ParsedAttrInfo` objects; the matching one takes over argument validation and semantic handling.

---

## 9. PCH and Modules Interaction

### 9.1 Caveats with Precompiled Headers

When a translation unit is compiled with `-include-pch`, the PCH is deserialised before `HandleTopLevelDecl` fires for the main file. Declarations from the PCH are **not** re-delivered via `HandleTopLevelDecl`; they arrive lazily during `HandleTranslationUnit` traversal. A visitor traversing the full TU via `TraverseDecl(Ctx.getTranslationUnitDecl())` will see them, but a plugin that analyses only `HandleTopLevelDecl` calls will miss them.

### 9.2 Caveats with C++ Modules

With implicit module builds (`-fmodules`), individual module translation units are compiled separately and cached. A plugin active during the main TU compilation is **not** active when module interface units are built (unless `-fplugin` is also passed to those compilations). Plugin warnings will therefore be absent for declarations that arrive via module imports.

For explicit module builds (C++20 `import`), the same restriction applies: module units are compiled independently. To analyse module-imported declarations, traverse the full `ASTContext` in `HandleTranslationUnit` rather than relying on `HandleTopLevelDecl` streaming.

### 9.3 The `hasPCHSupport` Override

`FrontendAction::hasPCHSupport` returns `true` by default, meaning the action tolerates PCH input. If your plugin is incompatible with PCH (e.g., it requires full re-parsing of all declarations), override it:

```cpp
bool hasPCHSupport() const override { return false; }
```

Clang will then error if a PCH is present, rather than silently producing incomplete results.

---

## 10. Out-of-Tree CMake Setup

### 10.1 `find_package(Clang)` Configuration

An out-of-tree plugin links against the installed Clang libraries. The canonical `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.20)
project(MyClangPlugin)

# Locate the installed Clang and LLVM packages.
find_package(Clang REQUIRED CONFIG
    HINTS "/usr/lib/llvm-22/lib/cmake/clang")

message(STATUS "Clang version: ${LLVM_PACKAGE_VERSION}")
message(STATUS "Clang includes: ${CLANG_INCLUDE_DIRS}")

include(AddLLVM)
include(AddClang)

include_directories(${LLVM_INCLUDE_DIRS} ${CLANG_INCLUDE_DIRS})
add_definitions(${LLVM_DEFINITIONS})

# Build the plugin as a shared library (MODULE = no link to executables).
add_library(MyPlugin MODULE
    MyPlugin.cpp
)

# Use clang_target_link_libraries for correct dylib/static handling.
clang_target_link_libraries(MyPlugin PRIVATE
    clangAST
    clangBasic
    clangFrontend
    clangLex
    clangSema
)

# Do NOT link libclang-cpp or libclang here; the host clang binary
# already provides those symbols. Linking them again causes
# ODR violations at runtime.
set_target_properties(MyPlugin PROPERTIES
    PREFIX ""              # Produce MyPlugin.so, not libMyPlugin.so
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
)
```

The `clang_target_link_libraries` function (defined in
[`AddClang.cmake`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/cmake/modules/AddClang.cmake))
respects the `CLANG_LINK_CLANG_DYLIB` CMake variable. When set to `ON` (the default for installed packages), it links against the single `libclang-cpp.so` shared library rather than individual static archives. Since the host `clang` binary already loaded that dylib, the plugin must not link it again—`clang_target_link_libraries` handles this correctly.

### 10.2 Build and Test Invocation

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_CXX_COMPILER=/usr/lib/llvm-22/bin/clang++
cmake --build build

# Test with cc1 directly:
/usr/lib/llvm-22/bin/clang -cc1 \
    -load ./build/MyPlugin.so \
    -plugin myplugin \
    test.cpp

# Test via the driver:
/usr/lib/llvm-22/bin/clang \
    -fplugin=./build/MyPlugin.so \
    -fplugin-arg-myplugin-verbose \
    test.cpp
```

---

## 11. Testing Plugin Diagnostics with FileCheck

Plugin output integrates with LLVM's `FileCheck` test infrastructure. A typical test file:

```cpp
// RUN: clang -cc1 -load %S/../build/MyPlugin.so -plugin myplugin %s 2>&1 | FileCheck %s
// RUN: clang -cc1 -load %S/../build/MyPlugin.so -plugin myplugin -verify %s

void foo(int *p __attribute__((nonnull)));

void bar() {
  foo(nullptr); // CHECK: warning: argument 0 passed to nonnull parameter 'p' may be null
}
```

The `-verify` mode (flag `-verify` to cc1) uses `// expected-warning`, `// expected-error` comments in the source file and fails if the diagnostics do not match. This is preferred for regression tests because it ties the expected diagnostic to the source line:

```cpp
// RUN: clang -cc1 -load %S/../build/MyPlugin.so -plugin myplugin -verify %s

void foo(int * __attribute__((nonnull)) p);

void bar() {
  foo(nullptr); // expected-warning{{argument 0 passed to nonnull parameter}}
}
```

---

## 12. Plugins versus libtooling: When to Use Each

Both plugins and libtooling tools use the same `ASTConsumer`/`RecursiveASTVisitor` infrastructure. The decision criterion is **whether the analysis must run inside the build**:

| Factor | Plugin | libtooling tool |
|---|---|---|
| Integrated with build flags | Yes — `-fplugin` flows through Makefiles/CMake | No — separate invocation |
| Issues compiler warnings/errors | Yes — blocked build on error | Yes — but separate process |
| IR-level access | Yes — `CodeGenerator::GetModule()` | Yes — via `EmitLLVMOnlyAction` |
| Run in parallel across TUs | Shared library loaded per compiler invocation | Tool per TU, fully independent |
| ABI stability requirement | Must match compiler's ABI exactly | Same |
| Distribution | Shared library, version-locked to Clang | Standalone binary |
| Custom pragmas | Yes | No (no preprocessor access during tool run) |
| PCH/modules caveat | Yes (§9) | Yes (same limitations) |

Use a plugin when you need diagnostics that block the build, when you need custom pragmas, or when you need IR-level access to the just-generated module. Use a libtooling tool (see [Ch46 — libtooling and AST Matchers](../part-07-clang-multilang/ch46-libtooling-ast-matchers.md)) when you need a standalone refactoring tool, a code indexer, or an analysis that runs as a separate CI step without being on the critical build path.

---

## 13. Worked Example 1 — Nonnull Argument Checker

This plugin warns when a function argument is `nullptr` and the corresponding parameter carries `__attribute__((nonnull))` or the C++ `[[gnu::nonnull]]` attribute.

### 13.1 Plugin Source

```cpp
// NonnullChecker.cpp
//
// Clang plugin: warn when nullptr is passed to a nonnull parameter.
// Build: see CMakeLists.txt in §10.
//
// Usage:
//   clang -fplugin=./NonnullChecker.so source.cpp

#include "clang/AST/ASTConsumer.h"
#include "clang/AST/ASTContext.h"
#include "clang/AST/Decl.h"
#include "clang/AST/Expr.h"
#include "clang/AST/RecursiveASTVisitor.h"
#include "clang/Basic/Diagnostic.h"
#include "clang/Frontend/CompilerInstance.h"
#include "clang/Frontend/FrontendAction.h"
#include "clang/Frontend/FrontendPluginRegistry.h"
#include "llvm/ADT/StringRef.h"

using namespace clang;

namespace {

// ---------------------------------------------------------------
// Visitor: walk all CallExprs and check nonnull arguments.
// ---------------------------------------------------------------
class NonnullVisitor : public RecursiveASTVisitor<NonnullVisitor> {
  DiagnosticsEngine &Diags;
  unsigned WarnID;

public:
  NonnullVisitor(DiagnosticsEngine &D, unsigned WarnID)
      : Diags(D), WarnID(WarnID) {}

  bool VisitCallExpr(CallExpr *CE) {
    const FunctionDecl *FD = CE->getDirectCallee();
    if (!FD) return true;

    unsigned NumParams = FD->getNumParams();
    for (unsigned I = 0; I < NumParams && I < CE->getNumArgs(); ++I) {
      const ParmVarDecl *PVD = FD->getParamDecl(I);

      // Check for __attribute__((nonnull)) on the parameter itself.
      bool IsNonnull = PVD->hasAttr<NonNullAttr>();

      // Also check function-level nonnull(N) — these reference param
      // indices in the Attrs stored on the FunctionDecl.
      if (!IsNonnull) {
        for (const auto *FNA : FD->specific_attrs<NonNullAttr>()) {
          for (const ParamIdx &PI : FNA->args()) {
            if (PI.getASTIndex() == I) { IsNonnull = true; break; }
          }
          if (IsNonnull) break;
        }
      }

      if (!IsNonnull) continue;

      // Check if the argument expression is a null pointer constant.
      const Expr *Arg = CE->getArg(I)->IgnoreParenImpCasts();
      if (Arg->isNullPointerConstant(FD->getASTContext(),
                                     Expr::NPC_ValueDependentIsNull)
          == Expr::NPCK_ZeroLiteral ||
          Arg->isNullPointerConstant(FD->getASTContext(),
                                     Expr::NPC_ValueDependentIsNull)
          == Expr::NPCK_GNUNull) {
        auto DB = Diags.Report(CE->getArg(I)->getExprLoc(), WarnID);
        DB << (int)I << PVD->getName();
      }
    }
    return true;
  }
};

// ---------------------------------------------------------------
// Consumer: registers the diagnostic ID and runs the visitor.
// ---------------------------------------------------------------
class NonnullConsumer : public ASTConsumer {
  CompilerInstance &CI;
  unsigned WarnNullArg = 0;

public:
  explicit NonnullConsumer(CompilerInstance &CI) : CI(CI) {}

  void Initialize(ASTContext &) override {
    WarnNullArg = CI.getDiagnostics().getCustomDiagID(
        DiagnosticsEngine::Warning,
        "argument %0 passed to nonnull parameter '%1' may be null");
  }

  void HandleTranslationUnit(ASTContext &Ctx) override {
    NonnullVisitor V(CI.getDiagnostics(), WarnNullArg);
    V.TraverseDecl(Ctx.getTranslationUnitDecl());
  }
};

// ---------------------------------------------------------------
// Action: the entry point registered with the plugin registry.
// ---------------------------------------------------------------
class NonnullAction : public PluginASTAction {
public:
  std::unique_ptr<ASTConsumer>
  CreateASTConsumer(CompilerInstance &CI, llvm::StringRef) override {
    return std::make_unique<NonnullConsumer>(CI);
  }

  bool ParseArgs(const CompilerInstance &,
                 const std::vector<std::string> &) override {
    return true;
  }

  ActionType getActionType() override { return AddAfterMainAction; }
};

} // anonymous namespace

// Self-registration — fires when the shared library is dlopen'd.
static FrontendPluginRegistry::Add<NonnullAction>
    X("nonnull-checker", "Warn when nullptr is passed to nonnull parameters");
```

### 13.2 Test

```cpp
// test_nonnull.cpp
// RUN: clang -cc1 -load ./NonnullChecker.so -plugin nonnull-checker -verify %s

void process(int * __attribute__((nonnull)) ptr, int count);

void caller() {
  int x = 0;
  process(&x, 1);          // ok — address-of is not null
  process(nullptr, 2);     // expected-warning{{argument 0 passed to nonnull parameter 'ptr' may be null}}
  process((int*)0, 3);     // expected-warning{{argument 0 passed to nonnull parameter 'ptr' may be null}}
}
```

---

## 14. Worked Example 2 — IR-Level Function Entry Instrumentation

This plugin injects a call to `void __my_instrument(const char *funcname)` at the entry of every non-declaration function in the generated IR. It follows the `CodeGenerator` wrapping pattern from §7.

### 14.1 Plugin Source

```cpp
// EntryInstrument.cpp
//
// Clang plugin: inject __my_instrument(funcname) at every function entry.
// The runtime library must provide:
//   void __my_instrument(const char *funcname);

#include "clang/AST/ASTConsumer.h"
#include "clang/AST/ASTContext.h"
#include "clang/CodeGen/ModuleBuilder.h"
#include "clang/Frontend/CompilerInstance.h"
#include "clang/Frontend/FrontendAction.h"
#include "clang/Frontend/FrontendPluginRegistry.h"
#include "llvm/IR/BasicBlock.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Type.h"

using namespace clang;
using namespace llvm;

namespace {

// ---------------------------------------------------------------
// Consumer: chains a CodeGenerator and post-processes the module.
// ---------------------------------------------------------------
class InstrumentConsumer : public ASTConsumer {
  CompilerInstance &CI;
  std::unique_ptr<CodeGenerator> CG;

public:
  InstrumentConsumer(CompilerInstance &CI, StringRef InFile)
      : CI(CI), CG(CreateLLVMCodeGen(CI, InFile, CI.getASTContext())) {}

  void Initialize(ASTContext &Ctx) override { CG->Initialize(Ctx); }

  bool HandleTopLevelDecl(DeclGroupRef DG) override {
    return CG->HandleTopLevelDecl(DG);
  }

  void HandleInlineMethodDefinition(CXXMethodDecl *D) override {
    CG->HandleInlineMethodDefinition(D);
  }

  void HandleTagDeclDefinition(TagDecl *D) override {
    CG->HandleTagDeclDefinition(D);
  }

  void HandleTranslationUnit(ASTContext &Ctx) override {
    // 1. Complete normal IR generation.
    CG->HandleTranslationUnit(Ctx);

    // 2. Retrieve the generated module.
    llvm::Module *M = CG->GetModule();
    if (!M) return;

    // 3. Inject instrumentation calls.
    injectInstrumentation(*M);

    // 4. Emit the module through the backend.
    // (In practice the plugin action replaces EmitObjAction;
    //  if wrapping, emit via CompilerInstance's output streams.)
  }

private:
  void injectInstrumentation(llvm::Module &M) {
    LLVMContext &Ctx = M.getContext();

    // Declare the hook: void __my_instrument(ptr)
    auto *VoidTy   = Type::getVoidTy(Ctx);
    auto *PtrTy    = PointerType::getUnqual(Ctx);  // opaque ptr (LLVM 15+)
    auto *FTy      = FunctionType::get(VoidTy, {PtrTy}, /*isVarArg=*/false);
    FunctionCallee Hook = M.getOrInsertFunction("__my_instrument", FTy);

    for (Function &F : M) {
      if (F.isDeclaration())        continue;
      if (F.getName().starts_with("__my_")) continue;
      if (F.hasFnAttribute(Attribute::Naked)) continue;

      BasicBlock &Entry = F.getEntryBlock();
      // Insert before the first non-alloca instruction to keep
      // the alloca group at the top (required by mem2reg).
      Instruction *InsertPt = &*Entry.getFirstNonPHIOrDbgOrAlloca();
      IRBuilder<> Builder(InsertPt);

      // Create a module-level string constant for the function name.
      Constant *NameStr =
          Builder.CreateGlobalStringPtr(F.getName(), ".instr.name",
                                        /*AddressSpace=*/0, &M);
      Builder.CreateCall(Hook, {NameStr});
    }
  }
};

// ---------------------------------------------------------------
// Action.
// ---------------------------------------------------------------
class InstrumentAction : public PluginASTAction {
public:
  std::unique_ptr<ASTConsumer>
  CreateASTConsumer(CompilerInstance &CI, llvm::StringRef InFile) override {
    return std::make_unique<InstrumentConsumer>(CI, InFile);
  }

  bool ParseArgs(const CompilerInstance &,
                 const std::vector<std::string> &) override {
    return true;
  }

  // Replace normal codegen — this consumer drives codegen itself.
  ActionType getActionType() override { return ReplaceAction; }
};

} // anonymous namespace

static FrontendPluginRegistry::Add<InstrumentAction>
    Y("entry-instrument",
      "Inject __my_instrument() at every function entry");
```

### 14.2 Runtime Stub and Test

```cpp
// runtime.c — compile and link with instrumented objects.
#include <stdio.h>
void __my_instrument(const char *funcname) {
  fprintf(stderr, "[instrument] entering %s\n", funcname);
}
```

```bash
# Build the plugin.
cmake --build build

# Compile with instrumentation.
clang -fplugin=./build/EntryInstrument.so \
      -fplugin-arg-entry-instrument \
      -c source.cpp -o source.o

# Verify IR contains the hook call.
clang -cc1 -load ./build/EntryInstrument.so \
      -plugin entry-instrument \
      -emit-llvm -o - source.cpp | \
  grep -A2 "call.*__my_instrument"

# Link with runtime and execute.
clang source.o runtime.c -o prog && ./prog
```

FileCheck test pattern:

```
; RUN: clang -cc1 -load %S/../build/EntryInstrument.so \
; RUN:   -plugin entry-instrument -emit-llvm -o - %s | FileCheck %s
;
; CHECK-LABEL: define {{.*}}@_Z3foov
; CHECK:       call void @__my_instrument
```

---

## 15. Chapter Summary

- **`PluginASTAction`** is the plugin base class. Override `CreateASTConsumer`, `ParseArgs`, and optionally `getActionType`. Register with `FrontendPluginRegistry::Add<>`.
- **Loading**: `-fplugin=path.so` (driver-level) or `-cc1 -load path.so -plugin name` (cc1-level). Both call `DynamicLibrary::LoadLibraryPermanently`; global constructors fire the registry.
- **`ActionType`** controls whether the plugin fires alongside or instead of the main action. `AddAfterMainAction` is appropriate for analysis plugins; `ReplaceAction` is appropriate for plugins that take over codegen.
- **`ASTConsumer` callbacks**: `Initialize` for setup, `HandleTopLevelDecl` for streaming analysis, `HandleTranslationUnit` for whole-TU work with `RecursiveASTVisitor`.
- **Custom diagnostics** use `DiagnosticsEngine::getCustomDiagID` at runtime—no TableGen required. `FixItHint` attaches suggested edits to any diagnostic.
- **`PragmaHandler`** is registered via `Preprocessor::AddPragmaHandler` in `BeginSourceFileAction` and removed in `EndSourceFileAction`.
- **IR-level access**: wrap a `CodeGenerator` in your `ASTConsumer`; after `HandleTranslationUnit` calls `CG->HandleTranslationUnit`, retrieve the `llvm::Module` via `CG->GetModule()` and use standard IR APIs.
- **`DiagnosticConsumer` replacement**: call `CI.getDiagnostics().setClient(new MyConsumer(), true)` to intercept all diagnostics from the compilation.
- **Custom attributes**: `__attribute__((annotate(...)))` covers most use cases without any plugin code. Full custom attributes require implementing `ParsedAttrInfo` and registering with `ParsedAttrInfoRegistry::Add<>`.
- **PCH/modules caveats**: declarations arriving via PCH or imported modules are not re-delivered to `HandleTopLevelDecl`; traverse the full TU via `TraverseDecl(Ctx.getTranslationUnitDecl())` instead.
- **CMake**: use `find_package(Clang)` and `clang_target_link_libraries` with `MODULE` library type. Do not link `libclang-cpp.so` directly—it is already loaded by the host compiler binary.
- **Testing**: use `clang -cc1 -load ... -verify` with `// expected-warning` annotations for unit tests; use `FileCheck` on emitted IR for IR-level plugin tests.
- **Plugin vs libtooling**: prefer plugins for diagnostics that must block a build and for IR instrumentation; prefer libtooling (see [Ch46 — libtooling and AST Matchers](../part-07-clang-multilang/ch46-libtooling-ast-matchers.md)) for standalone refactoring tools and CI-only analyses. See also [Ch36 — The Clang AST in Depth](../part-05-clang-frontend/ch36-clang-ast-in-depth.md) for AST node structure and [Ch52 — ClangIR Architecture](../part-08-clangir/ch52-clangir-architecture.md) for the CIR alternative pipeline.
