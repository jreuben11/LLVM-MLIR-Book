# Chapter 30 — The Diagnostic Engine

*Part V: The Clang Frontend*

Clang's reputation for excellent diagnostics is not an accident. Behind every caret, every fix-it suggestion, and every color-coded message lies a carefully engineered subsystem that separates diagnostic *definition*, *routing*, and *rendering* into orthogonal components. Understanding this architecture is essential for anyone writing Clang plugins, extending Sema, integrating libclang into an IDE, or building tools that consume or produce structured diagnostics. This chapter dissects `DiagnosticsEngine`, `DiagnosticIDs`, `DiagnosticConsumer`, `FixItHint`, and the full rendering pipeline from a diagnostic site in the AST to formatted text on a terminal or a SARIF file on disk.

---

## 30.1 Architecture: Three-Component Decoupling

The diagnostic subsystem rests on three collaborating classes defined in
[`clang/include/clang/Basic/Diagnostic.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/Diagnostic.h)
and
[`clang/include/clang/Basic/DiagnosticIDs.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/DiagnosticIDs.h):

```
DiagnosticIDs        DiagnosticOptions        DiagnosticConsumer
     |                      |                        |
     +----------+           |                        |
                |           |                        |
           DiagnosticsEngine (hub)
                |
           DiagnosticBuilder (RAII, in-flight)
                |
           Diagnostic (read-only view passed to consumer)
```

**`DiagnosticIDs`** is a reference-counted, translation-unit-independent object that owns the static per-diagnostic metadata database: format strings, default severity, warning group membership, SFINAE response classification, and category numbers. Because it is `IntrusiveRefCntPtr`-managed it can be shared across multiple `DiagnosticsEngine` instances (useful when a library produces diagnostics independently from the driver).

**`DiagnosticsEngine`** is the central hub. It holds a `DiagnosticIDs`, a `DiagnosticOptions&`, a pointer to the active `DiagnosticConsumer`, and the per-source-location `DiagStateMap` that tracks all pragma-induced severity overrides. It is tied to exactly one `SourceManager`. The engine processes every in-flight diagnostic: it maps severity, checks suppression conditions, increments counters, and finally dispatches to the consumer.

**`DiagnosticConsumer`** is the abstract output interface. Concrete subclasses include `TextDiagnosticPrinter` for human-readable terminal output, `TextDiagnosticBuffer` for accumulate-and-replay, `IgnoringDiagConsumer` for silencing, and `SARIFDiagnosticPrinter` for structured JSON output. The consumer receives a const `Diagnostic&` object — a read-only view into the engine's in-flight state — and is responsible for formatting, coloring, and writing.

The decoupling is intentional: the caller that emits `diag::err_undeclared_var_use` knows nothing about terminal colors or SARIF schemas. The consumer knows nothing about C++ grammar. The engine enforces the mapping policy (is this diagnostic suppressed? upgraded to error? already hit the error limit?) as a single, auditable choke point.

The constructor for `DiagnosticsEngine`:

```cpp
explicit DiagnosticsEngine(
    IntrusiveRefCntPtr<DiagnosticIDs> Diags,
    DiagnosticOptions &DiagOpts,
    DiagnosticConsumer *client = nullptr,
    bool ShouldOwnClient = true);
```

Ownership of the consumer is optional; pass `ShouldOwnClient = false` when the caller keeps the consumer alive (e.g., when wrapping an existing `TextDiagnosticPrinter` in a plugin).

### 30.1.1 `DiagnosticStorage` and the Allocator

In-flight diagnostics accumulate their arguments, source ranges, and fix-its in a `DiagnosticStorage` struct. The struct holds up to `MaxArguments = 10` positional arguments (each stored as a `(ArgumentKind, uint64_t)` pair for non-string kinds, or a `std::string` for string kinds), a `SmallVector<CharSourceRange, 8>` for source ranges, and a `SmallVector<FixItHint, 6>` for fix-its.

Rather than heap-allocating a `DiagnosticStorage` for every diagnostic, `DiagnosticsEngine` maintains a `DiagStorageAllocator` that caches up to 16 storage objects on a free list. Most diagnostics allocate lazily (only when the first argument is streamed), so completely unconditional `isIgnored()` checks that short-circuit before adding any argument produce zero allocation.

The `DiagStorageAllocator` is held directly inside `DiagnosticsEngine` (as `DiagAllocator`), not heap-allocated, keeping it cache-local. `DiagnosticBuilder` holds a pointer to this allocator and uses it when `getStorage()` is called for the first time on a builder.

This design is performance-critical: Sema calls `Diag()` many millions of times across a large C++ codebase, and the per-diagnostic allocation cost matters at scale. The combination of the free list allocator, lazy allocation, and the early-exit path through `isIgnored()` for suppressed diagnostics ensures that the common case (diagnostic is ignored) has near-zero overhead.

---

## 30.2 `DiagnosticIDs` and the Metadata Database

### 30.2.1 Generated Numeric IDs

Every built-in diagnostic is represented by an unsigned integer in the `diag::kind` typedef. The IDs are partitioned into contiguous ranges by subsystem:

| Range constant | Size | Subsystem |
|---|---|---|
| `DIAG_START_COMMON` | 300 | Common (shared across subsystems) |
| `DIAG_START_DRIVER` | 400 | Driver |
| `DIAG_START_FRONTEND` | 200 | Frontend actions |
| `DIAG_START_SERIALIZATION` | 120 | PCH/module serialization |
| `DIAG_START_LEX` | 500 | Lexer |
| `DIAG_START_PARSE` | 800 | Parser |
| `DIAG_START_AST` | 300 | AST |
| `DIAG_START_SEMA` | 5000 | Semantic analysis |
| `DIAG_START_ANALYSIS` | 100 | Static analysis |
| `DIAG_UPPER_LIMIT` | — | Upper bound; custom IDs start here |

These ranges are defined at the top of `DiagnosticIDs.h` and exist to let future additions avoid renumbering. Any ID `>= diag::DIAG_UPPER_LIMIT` is a custom diagnostic created at runtime.

The IDs themselves are generated by TableGen from `.td` source files in
[`clang/include/clang/Basic/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/).
The relevant sources are:

- `DiagnosticCommonKinds.td` — cross-subsystem errors (file not found, module errors)
- `DiagnosticDriverKinds.td` — driver-level warnings and errors
- `DiagnosticFrontendKinds.td` — `-cc1`-level frontend actions
- `DiagnosticLexKinds.td` — lexer tokens, string literals, preprocessor
- `DiagnosticParseKinds.td` — parser syntax errors
- `DiagnosticSemaKinds.td` — the largest file; all semantic diagnostics
- `DiagnosticASTKinds.td` — AST-level diagnostics
- `DiagnosticGroups.td` — maps `-Wfoo` flag names to groups of diagnostic IDs

TableGen emits `Diagnostic*Kinds.inc` files. Each line has the form:

```
DIAG(err_undeclared_var_use,
     CLASS_ERROR,
     (unsigned)diag::Severity::Error,
     "use of undeclared identifier %0",
     0,                         // category
     SFINAE_SubstitutionFailure,
     false,                     // AccessControl
     true,                      // Warn in system headers
     true,                      // Show in system macro
     false,                     // deferrable
     3)                         // nonce
```

The enum value `diag::err_undeclared_var_use` is then accessible from `clang/include/clang/Basic/DiagnosticSema.h` (which `#include`s the generated `.inc`).

### 30.2.2 Diagnostic Classes

`DiagnosticIDs` defines six class enumerators in `DiagnosticIDs.h`:

```cpp
enum Class {
  CLASS_INVALID   = 0x00,
  CLASS_NOTE      = 0x01,  // diag::note_*
  CLASS_REMARK    = 0x02,  // diag::remark_*
  CLASS_WARNING   = 0x03,  // diag::warn_*
  CLASS_EXTENSION = 0x04,  // language extensions (pedantic)
  CLASS_ERROR     = 0x05,  // diag::err_*
  CLASS_TRAP      = 0x06,  // internal traps
};
```

The class is the *static* category of a diagnostic as defined in the `.td` file. The *dynamic* severity — what the engine actually routes to the consumer after applying command-line flags and pragmas — is one of `diag::Severity::Ignored`, `Remark`, `Warning`, `Error`, or `Fatal`. These can diverge: a `CLASS_WARNING` can be elevated to `Error` by `-Werror`, and a `CLASS_ERROR` can be downgraded to `Fatal` by `-ferror-limit`.

Key query methods on `DiagnosticIDs`:

```cpp
// Retrieve the format string for a built-in or custom diagnostic.
StringRef getDescription(unsigned DiagID) const;

// True if the diagnostic is a note.
bool isNote(unsigned DiagID) const;

// True if the diagnostic is a warning or extension.
bool isWarningOrExtension(unsigned DiagID) const;

// True if the diagnostic is an extension (CLASS_EXTENSION).
bool isExtensionDiag(unsigned DiagID) const;

// Return the lowest-level warning group that contains this diagnostic.
std::optional<diag::Group> getGroupForDiag(unsigned DiagID) const;

// Return the -Wfoo flag name for the group containing DiagID.
StringRef getWarningOptionForDiag(unsigned DiagID);

// Return the category number (for -fdiagnostics-show-category).
static unsigned getCategoryNumberForDiag(unsigned DiagID);
```

The private method `getDiagClass(unsigned DiagID)` returns the `Class` enum for a given ID; it is only accessible to `DiagnosticsEngine` via friendship.

---

## 30.3 Diagnostic Groups and `-W` Flags

### 30.3.1 Group Definitions

`DiagnosticGroups.td` maps human-readable flag names to sets of diagnostic IDs. TableGen generates `DiagnosticGroups.inc`, which populates static arrays `DiagArrays[]` and `DiagSubGroups[]`. Each entry in `DiagArrays` is a null-terminated list of `diag::kind` values that belong to that group; each entry in `DiagSubGroups` lists child groups (for hierarchical groups like `-Wall`).

For example, the `-Wunused-variable` group resolves to `DiagArray1120` containing `diag::warn_unused_variable`. The `-Wshadow` group resolves to `DiagArray910` containing `diag::warn_shadow_field` (among others). `-Wall` is a super-group whose `DiagSubGroups` entry lists dozens of sub-groups.

`diag::Group` is an enum class generated by TableGen, with one enumerator per named group, plus `NUM_GROUPS` as the sentinel. `DiagnosticIDs::GroupInfos` is a heap-allocated array of `GroupInfo` structures, one per group, tracking the current severity override and `HasNoWarningAsError` flag.

### 30.3.2 Severity Manipulation

`DiagnosticsEngine::setSeverityForGroup()` changes all diagnostics in a named group:

```cpp
bool setSeverityForGroup(diag::Flavor Flavor,
                         StringRef Group,
                         diag::Severity Map,
                         SourceLocation Loc = SourceLocation());
```

Passing `diag::Severity::Error` with `Group = "unused-variable"` is equivalent to `-Werror=unused-variable`. The `Loc` parameter records where in source the change takes effect (for pragma-scoped changes); passing `SourceLocation()` means a command-line-level change.

`diag::Flavor` distinguishes `-Wfoo` groups (`Flavor::WarningOrError`) from `-Rfoo` remark groups (`Flavor::Remark`). Setting the severity of a `-Wfoo` group with `Flavor::Remark` is a no-op, and vice versa.

Individual diagnostic severity is set via:

```cpp
void DiagnosticsEngine::setSeverity(diag::kind Diag,
                                    diag::Severity Map,
                                    SourceLocation Loc);
```

Global flags translate into engine state:

| Command-line flag | Engine call |
|---|---|
| `-w` | `setIgnoreAllWarnings(true)` |
| `-Werror` | `setWarningsAsErrors(true)` |
| `-Wno-error=foo` | `setDiagnosticGroupWarningAsError("foo", false)` |
| `-Weverything` | `setEnableAllWarnings(true)` |
| `-pedantic` | `setExtensionHandlingBehavior(diag::Severity::Warning)` |
| `-pedantic-errors` | `setExtensionHandlingBehavior(diag::Severity::Error)` |
| `-ferror-limit=N` | `setErrorLimit(N)` |

`DiagnosticOptions` (defined in
[`clang/include/clang/Basic/DiagnosticOptions.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/DiagnosticOptions.h))
holds the raw parsed command-line data: `DiagnosticOptions::Warnings` is a `std::vector<std::string>` populated with the `-Wfoo` flags (prefix removed). These are applied to the engine during `CompilerInvocation` initialization by `ProcessWarningOptions()`.

---

## 30.4 Emitting a Diagnostic: The `DiagnosticBuilder` RAII Pattern

### 30.4.1 The `Report()` + `<<` Pattern

All diagnostic emission goes through:

```cpp
DiagnosticBuilder DiagnosticsEngine::Report(SourceLocation Loc,
                                            unsigned DiagID);
```

The returned `DiagnosticBuilder` is a lightweight RAII object. Arguments, source ranges, and fix-it hints are attached by chaining `operator<<`. When the temporary is destroyed — typically at the semicolon of the statement — the destructor calls `Emit()`, which triggers `EmitDiagnostic()` and ultimately `ProcessDiag()`:

```cpp
// Inside Sema, using the convenience wrapper Sema::Diag():
Diag(NameLoc, diag::err_undeclared_var_use) << Name;

// With a source range highlighting the problematic expression:
Diag(ExprLoc, diag::warn_unused_value)
    << E->getSourceRange();

// With a fix-it hint:
Diag(SemiLoc, diag::warn_null_arg)
    << FixItHint::CreateInsertion(SemiLoc.getLocWithOffset(1), ";");
```

`Sema::Diag()` is a thin wrapper that calls `Diags.Report()` with the semantic analysis source manager. The `operator<<` overloads on `StreamingDiagnostic` (the base of `DiagnosticBuilder`) accept:

| C++ type | `ArgumentKind` | Format specifier |
|---|---|---|
| `std::string` / `StringRef` | `ak_std_string` | `%0`–`%9` |
| `const char *` | `ak_c_string` | `%0`–`%9` |
| `int` | `ak_sint` | `%0`–`%9` |
| `unsigned` | `ak_uint` | `%0`–`%9` |
| `tok::TokenKind` | `ak_tokenkind` | `%0`–`%9` |
| `IdentifierInfo *` | `ak_identifierinfo` | `%0`–`%9` |
| `QualType` | `ak_qualtype` | `%0`–`%9` |
| `DeclarationName` | `ak_declarationname` | `%0`–`%9` |
| `NamedDecl *` | `ak_nameddecl` | `%0`–`%9` |
| `SourceRange` | — | highlights code |
| `CharSourceRange` | — | highlights code |
| `FixItHint` | — | attached hint |

The format string in the `.td` file uses `%0`–`%9` as positional substitution points, and modifiers like `%select{opt1|opt2|opt3}0` (select from list by integer argument), `%plural{1:singular;:plural}0`, `%q0` (quote identifier), and `%diff{from $ to $|types}0,1` (template diff).

### 30.4.2 Inside `ProcessDiag()`

When `DiagnosticBuilder::Emit()` fires, it calls `DiagnosticsEngine::EmitDiagnostic()`, which calls `ProcessDiag()`. The processing pipeline:

1. Look up the current `DiagState` for the diagnostic's source location (pragma-aware).
2. Determine the dynamic severity via `DiagnosticIDs::getDiagnosticSeverity()`, applying `IgnoreAllWarnings`, `WarningsAsErrors`, `ErrorsAsFatal`, the per-diagnostic mapping, and system-header suppression.
3. If severity is `Ignored`, return false (suppressed).
4. If severity is `Fatal` and `FatalsAsError` is set, demote to `Error`.
5. Increment `NumErrors` or `NumWarnings`. If `ErrorLimit` is nonzero and `NumErrors >= ErrorLimit`, escalate to `Fatal`.
6. Set the sticky flags `ErrorOccurred`, `FatalErrorOccurred`, `UnrecoverableErrorOccurred` as appropriate.
7. Update `TrapNumErrorsOccurred` (used by `DiagnosticErrorTrap`).
8. Update `LastDiagLevel`.
9. Construct a `Diagnostic` view object and call `DiagnosticConsumer::HandleDiagnostic(Level, Info)`.

The `Diagnostic` object passed to the consumer is a const wrapper over the engine's in-flight state. It exposes:

```cpp
class Diagnostic {
public:
  const DiagnosticsEngine *getDiags() const;
  unsigned getID() const;
  SourceLocation getLocation() const;
  bool hasSourceManager() const;
  SourceManager &getSourceManager() const;
  unsigned getNumArgs() const;
  DiagnosticsEngine::ArgumentKind getArgKind(unsigned Idx) const;
  StringRef getArgStdStr(unsigned Idx) const;
  const char *getArgCStr(unsigned Idx) const;
  int64_t getArgSInt(unsigned Idx) const;
  uint64_t getArgUInt(unsigned Idx) const;
  ArrayRef<CharSourceRange> getRanges() const;
  ArrayRef<FixItHint> getFixItHints() const;
  void FormatDiagnostic(SmallVectorImpl<char> &OutStr) const;
};
```

`FormatDiagnostic()` performs the format-string substitution and produces the final human-readable message string.

### 30.4.3 Format String Syntax

The format strings embedded in `.td` files support a small domain-specific language richer than `printf`. All arguments are positional (`%0` through `%9`), with no implicit ordering. Available substitution forms:

| Syntax | Meaning |
|---|---|
| `%0` | Substitute argument 0 using its kind-specific formatter |
| `%select{a\|b\|c}0` | Select string by integer argument 0 (zero-based) |
| `%plural{1:thing;:things}0` | English plural by integer value |
| `%q0` | Quote an identifier argument with backticks |
| `%diff{from $ to $\|types differ}0,1` | Template diff between two `QualType` arguments |
| `%objcclass0` | Format as ObjC class name |
| `%ordinal0` | Format integer as ordinal (1st, 2nd, etc.) |
| `%%` | Literal percent sign |

The `%select` modifier is used pervasively in `DiagnosticSemaKinds.td` to avoid separate diagnostic IDs for grammatically similar messages:

```
"invalid %select{return|parameter|variable|field}0 type %1 is an abstract class"
```

This single format string replaces four separate diagnostics that would otherwise differ only in the subject noun. Integer argument 0 selects the noun, argument 1 carries the `QualType`.

The formatting engine in `DiagnosticsEngine::ConvertArgToString()` dispatches to `ArgToStringFn`, a function pointer set by Sema to `Sema::ConvertArgToStringImpl()`. This function handles the complex cases — `QualType` formatting with `PrintingPolicy`, `DeclarationName` pretty-printing, template difference trees — that cannot be computed below the Sema layer.

### 30.4.4 `PartialDiagnostic`: Deferred Argument Capture

`PartialDiagnostic` (in
[`clang/include/clang/Basic/PartialDiagnostic.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/PartialDiagnostic.h))
is a `StreamingDiagnostic` subclass that captures a `DiagID` and its arguments without a `SourceLocation` and without immediately firing `ProcessDiag()`. It exists to allow Sema to construct a diagnostic at one point in the code, pass it to another function that will supply the `SourceLocation`, and only then emit it:

```cpp
// In a function returning bool: true = error, with optional diagnostic.
bool checkConstraint(QualType T, PartialDiagnostic &PD) {
  if (!T->isIntegralType(Ctx)) {
    PD << T; // capture QualType argument
    return true;
  }
  return false;
}

// At the call site:
PartialDiagnostic PD = PDiag(diag::err_typecheck_constraint_violated);
if (checkConstraint(ArgType, PD))
  Diag(Loc, PD);  // emit at the actual location
```

`PartialDiagnostic` uses the `DiagStorageAllocator` from the `DiagnosticsEngine` to avoid per-diagnostic heap allocation when the diagnostic is not fired. A `PartialDiagnostic` carrying no arguments uses zero heap.

`PartialDiagnosticAt` (a `std::pair<SourceLocation, PartialDiagnostic>`) is the standard deferred representation used in Sema's constraint checking and overload resolution result accumulation.

### 30.4.5 `SemaDiagnosticBuilder` and GPU Deferred Diagnostics

`SemaDiagnosticBuilder` (defined in
[`clang/include/clang/Sema/SemaBase.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/SemaBase.h))
is Sema's replacement for `DiagnosticBuilder` in the GPU compilation path. It models four behaviors via a `Kind` enum:

```cpp
enum Kind {
  K_Nop,                  // emit nothing
  K_Immediate,            // fire now (normal path)
  K_ImmediateWithCallStack, // fire now + print call stack to device code
  K_Deferred              // store as PartialDiagnostic on the FunctionDecl
};
```

CUDA and HIP code can be compiled for both host and device simultaneously. A diagnostic in a `__host__ __device__` function may be valid on the host but invalid on the device, or vice versa. `SemaDiagnosticBuilder` with `K_Deferred` stores the diagnostic as a `PartialDiagnosticAt` in `Sema::DeviceDeferredDiags`, keyed by `CanonicalDeclPtr<const FunctionDecl>`. These deferred diagnostics are replayed during codegen when it is determined that the function actually needs device-side emission.

`Sema::Diag()` returns a `SemaDiagnosticBuilder`, not a raw `DiagnosticBuilder`, which is why `operator<<` on the Sema diagnostic path does not immediately fire but may be deferred. The `K_ImmediateWithCallStack` mode produces an extra note-chain showing how the device-incompatible function is transitively reachable from a known-device entrypoint, which is the mechanism behind CUDA's "called from device code" backtrace messages.

---

## 30.5 `DiagnosticConsumer` Implementations

The abstract interface:

```cpp
class DiagnosticConsumer {
public:
  virtual void BeginSourceFile(const LangOptions &LO,
                               const Preprocessor *PP = nullptr) {}
  virtual void EndSourceFile() {}
  virtual void finish() {}
  virtual bool IncludeInDiagnosticCounts() const;
  virtual void HandleDiagnostic(DiagnosticsEngine::Level Level,
                                const Diagnostic &Info);
};
```

`BeginSourceFile()` and `EndSourceFile()` bracket the processing of each translation unit. Consumers that need language options (e.g., to understand token kinds) or the preprocessor (e.g., to expand macro names) capture them here. `finish()` is called once at the end of all compilation, allowing consumers that buffer output (e.g., `LogDiagnosticPrinter`) to flush.

### 30.5.1 `TextDiagnosticPrinter`

Defined in
[`clang/include/clang/Frontend/TextDiagnosticPrinter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/TextDiagnosticPrinter.h),
this is the consumer that writes the familiar caret diagnostics to an `llvm::raw_ostream`. It delegates actual formatting to `TextDiagnostic` (in `clang/lib/Frontend/TextDiagnostic.cpp`), which handles source snippet extraction, caret placement, range highlighting, fix-it rendering, macro expansion backtraces, and include stacks.

```cpp
TextDiagnosticPrinter printer(llvm::errs(), DiagOpts);
auto DiagIDs = llvm::makeIntrusiveRefCnt<DiagnosticIDs>();
DiagnosticsEngine Diags(DiagIDs, DiagOpts, &printer,
                        /*ShouldOwnClient=*/false);
```

### 30.5.2 `TextDiagnosticBuffer`

Defined in
[`clang/include/clang/Frontend/TextDiagnosticBuffer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/TextDiagnosticBuffer.h),
this consumer accumulates diagnostics into four `std::vector<std::pair<SourceLocation, std::string>>` lists (`Errors`, `Warnings`, `Remarks`, `Notes`) without printing anything. After compilation, `FlushDiagnostics()` replays them through a second `DiagnosticsEngine`:

```cpp
auto Buffer = std::make_unique<TextDiagnosticBuffer>();
// ... compile with buffer as consumer ...
// Later:
Buffer->FlushDiagnostics(FinalDiags);
```

This pattern appears in the `CompilerInstance` two-phase approach used by PCH generation: diagnostics from parsing the PCH header are buffered and replayed against the actual source file's engine.

### 30.5.3 `IgnoringDiagConsumer`

A null consumer for cases where diagnostics must be suppressed entirely:

```cpp
class IgnoringDiagConsumer : public DiagnosticConsumer {
  void HandleDiagnostic(DiagnosticsEngine::Level DiagLevel,
                        const Diagnostic &Info) override {}
};
```

Used when probing the compiler (e.g., testing type compatibility without reporting errors) or inside `__extension__`-delimited blocks.

### 30.5.4 `ForwardingDiagnosticConsumer`

Wraps an existing consumer, forwarding all calls unchanged. Used when a layer needs to intercept `BeginSourceFile`/`EndSourceFile` callbacks without overriding `HandleDiagnostic`. `IncludeInDiagnosticCounts()` delegates to the target.

### 30.5.5 `LogDiagnosticPrinter`

Defined in
[`clang/include/clang/Frontend/LogDiagnosticPrinter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/LogDiagnosticPrinter.h),
this consumer collects `DiagEntry` structs (message, filename, line, column, diagnostic ID, warning option, level) and writes an XML log on `EndSourceFile()`. Activated by `-fdiagnostics-log-file=<file>`.

### 30.5.6 `SARIFDiagnosticPrinter`

Defined in
[`clang/include/clang/Frontend/SARIFDiagnosticPrinter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/SARIFDiagnosticPrinter.h),
this consumer writes Static Analysis Results Interchange Format (SARIF) JSON, used by GitHub Advanced Security, Azure DevOps, and VS Code's Problems panel. Activated by `-fdiagnostics-format=sarif`. It owns a `SarifDocumentWriter` and populates it with SARIF `result` objects for each diagnostic and SARIF `fix` objects for each fix-it hint. The writer flushes the complete SARIF document on `EndSourceFile()`.

### 30.5.7 Writing a Custom Consumer

Implementing `DiagnosticConsumer` requires only `HandleDiagnostic()`. A minimal example that collects error messages:

```cpp
class ErrorCollector : public DiagnosticConsumer {
  std::vector<std::string> &Errors;
public:
  ErrorCollector(std::vector<std::string> &E) : Errors(E) {}

  void HandleDiagnostic(DiagnosticsEngine::Level Level,
                        const Diagnostic &Info) override {
    DiagnosticConsumer::HandleDiagnostic(Level, Info); // update counts
    if (Level >= DiagnosticsEngine::Error) {
      SmallString<256> Msg;
      Info.FormatDiagnostic(Msg);
      Errors.push_back(std::string(Msg));
    }
  }
};
```

For source-location-aware consumers, check `Info.hasSourceManager()` before calling `Info.getSourceManager()`, as diagnostics emitted before the source manager is set (e.g., during option parsing) will have an invalid location.

---

## 30.6 Fix-It Hints

Fix-it hints are code modification suggestions attached to diagnostics. They model three primitive operations, all as static factory methods on `FixItHint` (defined inline in `Diagnostic.h`):

```cpp
// Insert Code before InsertionLoc.
static FixItHint CreateInsertion(SourceLocation InsertionLoc,
                                 StringRef Code,
                                 bool BeforePreviousInsertions = false);

// Remove the source range RemoveRange.
static FixItHint CreateRemoval(CharSourceRange RemoveRange);
static FixItHint CreateRemoval(SourceRange RemoveRange);

// Replace RemoveRange with Code.
static FixItHint CreateReplacement(CharSourceRange RemoveRange,
                                   StringRef Code);
static FixItHint CreateReplacement(SourceRange RemoveRange,
                                   StringRef Code);
```

Fix-it hints are attached to a diagnostic via `operator<<` on `DiagnosticBuilder`:

```cpp
Diag(Loc, diag::err_expected_semi_after_stmt)
    << FixItHint::CreateInsertion(Loc.getLocWithOffset(1), ";");
```

Multiple fix-it hints can be attached to a single diagnostic by chaining multiple `<<` insertions.

A `FixItHint` with a `RemoveRange` of zero length (start == end) is a pure insertion. A `FixItHint` with `CodeToInsert` empty is a pure removal. This is the internal representation; the public API enforces the distinction through the named factories but the underlying struct is unified.

The `BeforePreviousInsertions` field on `CreateInsertion` handles the case where multiple fix-its insert at the same location: if `BeforePreviousInsertions = true`, the new insertion is placed before any already-queued insertions at that point. This is used, for example, when both a `const` qualifier and a `*` need to be inserted at the same token boundary.

`FixItHint` objects are stored in `DiagnosticStorage::FixItHints` (capacity 6 on the stack before spilling to heap) and accessed by consumers via `Diagnostic::getFixItHints()`.

### 30.6.1 Parseable Fix-Its (`-fdiagnostics-parseable-fixits`)

When this flag is active, `TextDiagnosticPrinter` emits machine-readable fix-it output on a separate line after the human-readable message:

```
fix-it:"foo.cpp":{12:5-12:5}:";"
```

The format is: `fix-it:"<filename>":{<start-line>:<start-col>-<end-line>:<end-col>}:"<replacement>"`. An empty replacement string represents removal; start == end represents insertion. clangd uses this format when computing quick-fix code actions from compiler diagnostics.

### 30.6.2 `FixItRewriter` for In-Place Application

`FixItRewriter` (in
[`clang/include/clang/Rewrite/Frontend/FixItRewriter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Rewrite/Frontend/FixItRewriter.h))
is a `DiagnosticConsumer` that applies fix-it hints directly to the source buffers using a `Rewriter` and `edit::EditedSource`. It wraps another consumer for display purposes:

```cpp
FixItOptions Opts;
Opts.InPlace = true;
Opts.FixWhatYouCan = true;
FixItRewriter Rewriter(Diags, SrcMgr, LangOpts, &Opts);
Diags.setClient(&Rewriter, /*ShouldOwnClient=*/false);
```

After compilation, `WriteFixedFiles()` flushes all modified source buffers to disk. `clang -fixit` activates this path. `clang-apply-replacements` offers a standalone tool that applies replacements stored in YAML files (produced by `clang-tidy`'s `--export-fixes` option), which represent the same semantic operation as fix-it hints but in a file-based exchange format.

---

## 30.7 Notes and Continuation Diagnostics

### 30.7.1 Note Semantics

Notes (`CLASS_NOTE`, IDs in the `diag::note_*` namespace) are always associated with the most recently emitted non-note diagnostic. The engine tracks `LastDiagLevel`; if `LastDiagLevel == Ignored`, any subsequent notes are suppressed without being passed to the consumer. This ensures that silenced warnings do not produce orphaned notes.

Emitting a note uses the same `Report()` + `<<` pattern:

```cpp
// Error at the use site.
Diag(UseLoc, diag::err_use_of_default_argument_to_function_pack_expansion);
// Note at the declaration site.
Diag(DeclLoc, diag::note_template_param_here);
```

Notes carry a `SourceLocation` and format arguments just like other diagnostics. They may carry fix-it hints (uncommon but valid). Notes cannot be upgraded to errors or warnings; they are always rendered at note level regardless of `-Werror`.

### 30.7.2 Chaining Multiple Notes

It is valid and common to emit several consecutive notes for a single error. Sema does this extensively for template instantiation backtraces, overload resolution failures (each candidate produces a note), and multiple declaration sites:

```cpp
Diag(MainLoc, diag::err_ovl_no_viable_function_in_call) << FnName;
for (const auto &Cand : Candidates) {
  Diag(Cand.Decl->getLocation(),
       diag::note_ovl_candidate) << Cand.Decl;
}
```

The `DiagnosticsEngine` does not impose any limit on the number of notes per diagnostic. However, `setTemplateBacktraceLimit()` and `setConstexprBacktraceLimit()` control how many template/constexpr instantiation notes Sema emits before truncating with a summary note.

---

## 30.8 Diagnostic State and Suppression

### 30.8.1 The `DiagState` and `DiagStateMap`

`DiagnosticsEngine` maintains a linked list of `DiagState` objects. Each state is a `DenseMap<unsigned, DiagnosticMapping>` augmented with global flags:

```cpp
class DiagState {
  DenseMap<unsigned, DiagnosticMapping> DiagMap; // per-ID overrides
  unsigned IgnoreAllWarnings : 1;   // -w
  unsigned EnableAllWarnings : 1;   // -Weverything
  unsigned WarningsAsErrors : 1;    // -Werror
  unsigned ErrorsAsFatal : 1;       // -Werror (fatal)
  unsigned SuppressSystemWarnings : 1; // -Wno-system-headers
  diag::Severity ExtBehavior;       // -pedantic behavior
};
```

`DiagStateMap` maps `(FileID, offset)` pairs to `DiagState*`, forming a timeline of when the state changed within each file. `pushMappings()` / `popMappings()` create new state copies on a stack; the ASTReader and ASTWriter serialize and restore this map for PCH/module replay.

### 30.8.2 `#pragma clang diagnostic`

The preprocessor handles `#pragma clang diagnostic push/pop/ignored/warning/error` by calling `DiagnosticsEngine::pushMappings()` / `popMappings()` (for push/pop) or `setSeverity()` with the pragma source location (for ignored/warning/error). The source location parameter is critical: it tells the `DiagStateMap` at exactly which file offset the new state takes effect, ensuring that diagnostics before the pragma use the old state and diagnostics after use the new state, even within a single file being compiled:

```cpp
// Source: foo.cpp
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wunused-variable"
int unused = 42;   // suppressed
#pragma clang diagnostic pop
int unused2 = 43;  // visible (original state restored)
```

`_Pragma` (the C99 operator form) is expanded by the preprocessor into a pragma before the main token stream is processed, so it interacts with the diagnostic state identically to `#pragma`.

### 30.8.3 `DiagnosticMapping` Internals

`DiagnosticMapping` is an 8-bit packed struct that records the severity override for a single diagnostic ID within a given `DiagState`:

```cpp
class DiagnosticMapping {
  unsigned Severity           : 3;  // diag::Severity value
  unsigned IsUser             : 1;  // set by -Wfoo / #pragma
  unsigned IsPragma           : 1;  // set by #pragma (not command line)
  unsigned HasNoWarningAsError : 1; // -Wno-error=foo
  unsigned HasNoErrorAsFatal  : 1;  // -Wno-fatal-errors=foo
  unsigned WasUpgradedFromWarning : 1; // diagnostic was promoted by -Werror
};
```

A mapping with `Severity == 0` (unset) means the diagnostic uses its default severity from `DiagnosticIDs`. When `setSeverity()` is called, it creates a `DiagnosticMapping` with `IsUser = true` and inserts it into `DiagState::DiagMap` under the diagnostic ID. Subsequent calls to `getDiagnosticSeverity()` check this map first, then fall back to the `DiagnosticIDs` default.

The `IsPragma` flag distinguishes command-line overrides from `#pragma clang diagnostic` overrides. Pragma-sourced mappings have `HasNoWarningAsError = true` and `HasNoErrorAsFatal = true`, which means `-Werror` does not override a `#pragma clang diagnostic warning` that would otherwise be silenced by `-Werror`. This matches user expectations: an explicit `#pragma` is more specific than a global command-line flag.

`getDiagnosticMappings()` on `DiagnosticsEngine` returns an `iterator_range` over the current `DiagState`'s `DenseMap`, useful for tools that need to enumerate all active overrides.

### 30.8.4 Diagnostic Suppression Mapping Files

Clang 22 adds programmatic suppression via `DiagnosticOptions::DiagnosticSuppressionMappingsFile`. The file is a special-case list with section headers naming diagnostic groups and `src:` entries specifying file globs:

```
[unused]
src:clang/*
src:clang/foo/*=emit
```

This suppresses all `-Wunused` diagnostics in files under `clang/` except those under `clang/foo/`. The format is processed by `DiagnosticsEngine::setDiagSuppressionMapping()`, which populates a `llvm::unique_function` stored as `DiagSuppressionMapping`. During `ProcessDiag()`, the engine calls `isSuppressedViaMapping()` before dispatching to the consumer.

### 30.8.5 Per-Diagnostic and Error-Limit Controls

```cpp
Diags.setErrorLimit(20);         // stop after 20 errors (0 = unlimited)
Diags.setSuppressAllDiagnostics(true); // absolute silence
Diags.setFatalsAsError(true);    // treat fatals as errors (useful in tests)
bool hadErr = Diags.hasErrorOccurred();
bool hadFatal = Diags.hasFatalErrorOccurred();
bool hadUncompilable = Diags.hasUncompilableErrorOccurred(); // non-Werror
unsigned n = Diags.getNumErrors();
unsigned w = Diags.getNumWarnings();
Diags.Reset();                   // clear all state, reset counters
```

`Reset(bool soft)` with `soft = true` preserves the diagnostic mappings (used when reusing the engine across multiple files in a `CompilationDatabase` scan) but resets the error counters and sticky flags.

### 30.8.6 `DiagnosticErrorTrap`

`DiagnosticErrorTrap` is an RAII guard for scoped error detection:

```cpp
class DiagnosticErrorTrap {
  DiagnosticsEngine &Diag;
  unsigned NumErrors;
  unsigned NumUnrecoverableErrors;
public:
  explicit DiagnosticErrorTrap(DiagnosticsEngine &Diag);
  bool hasErrorOccurred() const;
  bool hasUnrecoverableErrorOccurred() const;
  void reset();
};
```

Sema uses `ErrorTrap` during, for example, parsing a function body to detect whether any errors occurred in that scope without checking global error state. This is important because errors from unrelated code paths may have already set the global `ErrorOccurred` flag.

---

## 30.9 Source Ranges, Carets, and Terminal Rendering

### 30.9.1 `DiagnosticRenderer` Architecture

The rendering pipeline inside `TextDiagnosticPrinter::HandleDiagnostic()` delegates to `TextDiagnostic`, a subclass of `DiagnosticNoteRenderer`, which is itself a subclass of `DiagnosticRenderer` (defined in
[`clang/include/clang/Frontend/DiagnosticRenderer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/DiagnosticRenderer.h)).

`DiagnosticRenderer` is an abstract class with pure virtual hooks:

```cpp
virtual void emitDiagnosticMessage(FullSourceLoc Loc, PresumedLoc PLoc,
                                   DiagnosticsEngine::Level Level,
                                   StringRef Message,
                                   ArrayRef<CharSourceRange> Ranges,
                                   DiagOrStoredDiag Info) = 0;
virtual void emitDiagnosticLoc(FullSourceLoc Loc, PresumedLoc PLoc,
                               DiagnosticsEngine::Level Level,
                               ArrayRef<CharSourceRange> Ranges) = 0;
virtual void emitCodeContext(FullSourceLoc Loc,
                             DiagnosticsEngine::Level Level,
                             SmallVectorImpl<CharSourceRange> &Ranges,
                             ArrayRef<FixItHint> Hints) = 0;
virtual void emitIncludeLocation(FullSourceLoc Loc, PresumedLoc PLoc) = 0;
```

The non-virtual entry point `DiagnosticRenderer::emitDiagnostic()` orchestrates the full output sequence: it calls `emitDiagnosticMessage()` for the header line, then `emitMacroExpansions()` if the location is in a macro, then `emitCodeContext()` for the source snippet. `emitMacroExpansions()` recursively unwinds the macro expansion stack, generating synthetic notes at each expansion level, until reaching the spelling location in the macro definition file.

`DiagnosticNoteRenderer` is an intermediate subclass that overrides `emitIncludeLocation()`, `emitImportLocation()`, and `emitBuildingModuleLocation()` to synthesize explicit notes ("in file included from ..."), relieving concrete subclasses of this responsibility.

### 30.9.2 Caret Placement

`TextDiagnostic` (in
[`clang/lib/Frontend/TextDiagnostic.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Frontend/TextDiagnostic.cpp))
extracts the source line containing the diagnostic location from the `SourceManager`, then computes the caret column. Column computation accounts for tab stops (`DiagnosticOptions::TabStop`, default 8) so that the caret aligns correctly for source with mixed tabs and spaces. Multi-column characters (CJK, emoji) are handled by `llvm::sys::locale::columnWidth()`.

When `ShowCarets` is set (default true via `DiagnosticOptions::ShowCarets`), the printer emits the source line, then a second line with `^` at the caret column and `~` under each highlighted range. Ranges are provided as `CharSourceRange` objects attached to the diagnostic.

Caret placement also accounts for multi-line diagnostics: when a source range spans multiple lines, `TextDiagnostic` limits the snippet to `DiagnosticOptions::SnippetLineLimit` lines (default 16) and marks elisions with `...`. The `DiagnosticOptions::ShowLineNumbers` flag (enabled by default) prepends line numbers to each snippet line for long-range highlights.

### 30.9.3 Macro Expansion Rendering

When a diagnostic location is inside a macro expansion, `TextDiagnostic` recursively emits expansion notes. The `DiagnosticRenderer` base class drives this via `emitMacroExpansions()` and `emitSingleMacroExpansion()`. The expansion chain walks from the expansion point back to the spelling point in the macro definition. For deep macro nesting, `DiagnosticOptions::MacroBacktraceLimit` (default 6) limits the depth.

A diagnostic emitted inside a macro shows both the expansion site (where the macro was invoked) and the spelling site (where the token appears in the macro body):

```
foo.cpp:10:5: warning: result unused [-Wunused-value]
    COMPUTE(x + y);
    ^~~~~~~~~~~~~~
foo.cpp:3:18: note: expanded from macro 'COMPUTE'
#define COMPUTE(e) (e)
                   ^
```

### 30.9.4 Display Flags

Key `DiagnosticOptions` boolean fields (all defined via `DIAGOPT` macros in `DiagnosticOptions.def`):

| Option field | Default | Flag |
|---|---|---|
| `ShowCarets` | 1 | `-fno-diagnostics-show-caret` |
| `ShowColumn` | 1 | `-fno-show-column` |
| `ShowLocation` | 1 | `-fno-show-source-location` |
| `ShowFixits` | 1 | `-fno-diagnostics-fixit-info` |
| `ShowColors` | 0 | `-fcolor-diagnostics` / `-fno-color-diagnostics` |
| `ShowParseableFixits` | 0 | `-fdiagnostics-parseable-fixits` |
| `ShowOptionNames` | 0 | `-fdiagnostics-show-option` |
| `ShowSourceRanges` | 0 | `-fdiagnostics-show-source-ranges-in-numeric-form` |
| `AbsolutePath` | 0 | `-fdiagnostics-absolute-paths` |

Color is controlled by `ShowColors` on the `DiagnosticsEngine` (set by `setShowColors()`). `TextDiagnostic` injects ANSI escape sequences for bold (errors), magenta (warnings), cyan (notes), and green (remarks), and for underlining source ranges. The `llvm::WithColor` utility class is the abstraction over ANSI escape sequences.

The `Format` enumeration in `DiagnosticOptions`:

```cpp
enum TextDiagnosticFormat { Clang, MSVC, Vi, SARIF };
```

- `Clang` (default): `file.cpp:10:5: error: message`
- `MSVC`: `file.cpp(10,5): error: message` — for Visual Studio integration
- `Vi`: `file.cpp +10: error: message` — for Vim quickfix list
- `SARIF`: activates `SARIFDiagnosticPrinter`

---

## 30.10 Structured Diagnostics and Tooling Integration

### 30.10.1 Serialized Diagnostics (`.dia` Files)

Clang can write binary-serialized diagnostics to a `.dia` file via `-serialize-diagnostics <file>`. The format is defined in `clang/include/clang/Frontend/SerializedDiagnostics.h` and is built on LLVM's bitstream container. The `clang::serialized_diags` namespace defines:

```cpp
enum BlockIDs { BLOCK_META, BLOCK_DIAG };
enum RecordIDs {
  RECORD_VERSION = 1, RECORD_DIAG,    RECORD_SOURCE_RANGE,
  RECORD_DIAG_FLAG,   RECORD_CATEGORY, RECORD_FILENAME,
  RECORD_FIXIT
};
enum Level { Ignored=0, Note, Warning, Error, Fatal, Remark };
```

`SerializedDiagnosticPrinter` (in
[`clang/include/clang/Frontend/SerializedDiagnosticPrinter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/SerializedDiagnosticPrinter.h))
is the consumer that writes this format. `SerializedDiagnosticReader` reads it back. Xcode uses `.dia` files to display inline diagnostics in the IDE after a build, reading them with `clang_loadDiagnostics()` from libclang.

The serialized format is stable across Clang versions (the version number is recorded in `RECORD_VERSION`; version 2 as of Clang 22) and survives removal of the source files, making it suitable for caching build results.

### 30.10.2 `VerifyDiagnosticConsumer` for Testing

`VerifyDiagnosticConsumer` (in
[`clang/include/clang/Frontend/VerifyDiagnosticConsumer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/VerifyDiagnosticConsumer.h))
is the backbone of Clang's compiler regression tests. It implements `DiagnosticConsumer` and `CommentHandler` simultaneously: as the preprocessor tokenizes the source, `VerifyDiagnosticConsumer` scans comments for `expected-*` directives, then after compilation it verifies that exactly the expected diagnostics were emitted:

```cpp
// Source test file:
int *p = 42;  // expected-error {{incompatible integer to pointer conversion}}
void f() { return 1; }  // expected-warning {{void function 'f' should not return a value}}
```

The directive syntax is `// expected-<level> {{<regex>}}` where `<level>` is `error`, `warning`, `note`, or `remark`. Regex directives use `// expected-error-re {{pattern}}`. The `{{}}` content is matched as a substring (non-regex) or full regex (with `-re`). Count ranges like `// expected-warning 2 {{...}}` assert that the diagnostic fires exactly twice.

The consumer is activated by `-verify` (passes to the frontend) or by calling `DiagnosticOptions::VerifyDiagnostics = true`. At `EndSourceFile()`, it compares the collected actual diagnostics against the parsed expected directives, and emits error diagnostics for any mismatch (unexpected diagnostics or expected diagnostics that were not fired). All LLVM test files under `clang/test/` that use `RUN: %clang_cc1 -verify` rely on this consumer.

### 30.10.3 libclang `CXDiagnostic` API

libclang exposes diagnostics through the `CXDiagnostic` type (opaque pointer). Key functions:

```c
CXDiagnosticSet clang_getDiagnosticSetFromTU(CXTranslationUnit TU);
unsigned        clang_getNumDiagnosticsInSet(CXDiagnosticSet);
CXDiagnostic    clang_getDiagnosticInSet(CXDiagnosticSet, unsigned idx);
CXString        clang_formatDiagnostic(CXDiagnostic, unsigned options);
enum CXDiagnosticSeverity clang_getDiagnosticSeverity(CXDiagnostic);
CXSourceLocation clang_getDiagnosticLocation(CXDiagnostic);
unsigned        clang_getDiagnosticNumFixIts(CXDiagnostic);
CXString        clang_getDiagnosticFixIt(CXDiagnostic, unsigned FixIt,
                                         CXSourceRange *ReplacementRange);
```

Internally, libclang stores diagnostics as `StoredDiagnostic` objects (each holds a `Level`, formatted message string, `FullSourceLoc`, source ranges, and fix-its). `CXDiagnostic` is a pointer to a `StoredDiagnostic`. This API powers IDE integrations that parse Clang diagnostics asynchronously.

`StoredDiagnostic` (declared in `clang/Basic/Diagnostic.h`) differs from the ephemeral `Diagnostic` view in that it owns its data: the formatted string is copied, `FullSourceLoc` retains a reference to the `SourceManager` so it can resolve file/line/column on demand, and ranges and fix-its are copied. When the `SourceManager` is destroyed (e.g., the `ASTUnit` is freed), `StandaloneDiagnostic` (in
[`clang/include/clang/Frontend/StandaloneDiagnostic.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/StandaloneDiagnostic.h))
provides a serialization layer that records file offsets instead of `SourceLocation` values, allowing the diagnostic to survive source manager teardown. `translateStandaloneDiag()` reconstructs a `StoredDiagnostic` from a `StandaloneDiagnostic` against a new `SourceManager`.

### 30.10.4 clangd Integration

clangd (the language server) uses `DiagnosticsEngine` directly through the `ClangdServer` pipeline. It registers a custom consumer that captures `StoredDiagnostic` objects and converts them to LSP `Diagnostic` JSON structures. Fix-it hints become LSP `CodeAction` items of kind `quickfix`. clangd also uses the `-fdiagnostics-parseable-fixits` format as a fallback when processing compile-command-based invocations via `CompilerInvocation::RunInvocation()`.

---

## 30.11 Custom Diagnostic IDs

### 30.11.1 `getCustomDiagID()`

Plugins, LibTooling tools, and clang-tidy checks that need diagnostics not in the built-in database can register them at runtime:

```cpp
// Legacy API (deprecated but widely used):
unsigned MyDiagID =
    Diags.getCustomDiagID(DiagnosticsEngine::Warning,
                          "my tool: %0 is problematic");

// Preferred: use CustomDiagDesc for group association:
using CDDesc = DiagnosticIDs::CustomDiagDesc;
unsigned MyDiagID2 = Diags.getDiagnosticIDs()->getCustomDiagID(
    CDDesc(diag::Severity::Warning,
           "my tool: %0 is problematic",
           DiagnosticIDs::CLASS_WARNING,
           /*ShowInSystemHeader=*/false,
           /*ShowInSystemMacro=*/false,
           /*Group=*/diag::Group::Unused));  // associate with -Wunused
```

Custom IDs are assigned values `>= diag::DIAG_UPPER_LIMIT`. `DiagnosticIDs::IsCustomDiag()` checks this boundary. Custom diagnostics registered with the same description string and severity get the same ID (they are deduplicated via a `std::map<CustomDiagDesc, unsigned>`).

The legacy API (`Level` overload) creates diagnostics without group association, meaning users cannot suppress them with `-Wno-foo`. The `CustomDiagDesc` API allows group association so the tool's diagnostics participate in the standard `-W` hierarchy.

The deprecation comment on `getCustomDiagID(Level, StringRef)` is intentional: the `Level`-based API cannot associate a custom diagnostic with a warning group, so users have no way to suppress it with `-Wno-foo` or elevate it with `-Werror=foo`. Every existing callsite in the Clang codebase is a technical debt item. New code should use `CustomDiagDesc` with an explicit `diag::Group` association, or omit the group to accept the tradeoff consciously.

When deduplicating custom diagnostics, `DiagnosticIDs` uses `std::map<CustomDiagDesc, unsigned>` ordered by `operator<` on the `CustomDiagDesc` tuple. Two descriptors with identical description strings, severity, class, system-header visibility, and group are assigned the same ID. This deduplication means that calling `getCustomDiagID` repeatedly with the same arguments (as clang-tidy does for each check invocation) is safe and efficient.

### 30.11.2 clang-tidy Pattern

clang-tidy checks emit diagnostics through `ClangTidyCheck::diag()`, which internally calls:

```cpp
DiagnosticBuilder ClangTidyCheck::diag(SourceLocation Loc,
                                       StringRef Message,
                                       DiagnosticIDs::Level Level) {
  unsigned ID = Context->getOrCreateDiagID(Level, Message, CheckName);
  return Context->DiagEngine->Report(Loc, ID);
}
```

The `ClangTidyContext::getOrCreateDiagID()` call maps each `(CheckName, Message)` pair to a stable custom ID. This means `-checks=-clang-analyzer-*` translates to suppressing all diagnostics whose custom IDs were registered by analysis-namespace checks. Fix-it hints returned by `registerFix()` are attached as `FixItHint` objects in the standard manner.

---

## 30.12 Diagnostics in the Compiler Pipeline

### 30.12.1 Error Gatekeeping

`DiagnosticsEngine` acts as a gate between pipeline stages. Each major phase checks `hasErrorOccurred()` or `hasUnrecoverableErrorOccurred()` before proceeding:

```cpp
// In CompilerInstance::ExecuteAction():
if (Diags.hasUnrecoverableErrorOccurred())
  return false;

// In Sema after parsing:
if (getDiagnostics().hasErrorOccurred())
  return;  // Skip codegen
```

`hasUncompilableErrorOccurred()` specifically checks for errors that were not `-Werror` upgrades — a distinction important for diagnostics-as-warnings modes where the tool is expected to continue despite upgraded warnings.

### 30.12.2 PCH Diagnostic State Replay

When a precompiled header is loaded, `ASTReader` (declared as a `friend` of `DiagnosticsEngine`) restores the serialized `DiagStateMap` via `DiagStatesByLoc`. This ensures that `#pragma clang diagnostic` directives that appeared in the PCH's source files are correctly honored when diagnosing code that uses the PCH. `ASTWriter` serializes the map by writing each `DiagStatePoint` (a `(FileID, offset, DiagState)` triple). After loading, `DiagnosticsEngine::ResetPragmas()` clears the FileID-to-state cache (because FileIDs from the PCH may conflict with FileIDs in the new translation unit).

### 30.12.3 `DiagnosticsEngine::Reset()`

```cpp
void DiagnosticsEngine::Reset(bool soft = false);
```

With `soft = false` (the default), `Reset()` clears the entire `DiagStatesByLoc` map, resets all error counters, clears the `DiagStateOnPushStack`, and recreates the initial `DiagState`. This is used between translation units in batch compilation.

With `soft = true`, the error counts and sticky flags are reset, but the diagnostic mappings in `DiagStatesByLoc` are preserved. This is used in incremental compilation scenarios (e.g., clangd's background index) where the same engine processes multiple slightly-different versions of a file.

### 30.12.4 Deferrable Diagnostics in Offloading

`DiagnosticIDs::isDeferrable(unsigned DiagID)` returns true for a small set of diagnostics tagged `deferrable` in their `.td` definition. These are diagnostics that occur inside `__host__ __device__` functions where it is not yet known at parse time whether the function will be emitted for the device side. Rather than firing immediately, the diagnostic is stored as a `PartialDiagnosticAt` on the `FunctionDecl` and replayed by `Sema::emitDeferredDiags()` during codegen if the function is actually instantiated for the device. The `isDeferrable` attribute in the generated `.inc` file is the sixth integer argument in each `DIAG(...)` macro row (value `true` vs. `false`).

### 30.12.5 SFINAE Interaction

During template argument deduction, errors in substituted expressions must not leak to the user; they must only cause deduction failure. `DiagnosticIDs::getDiagnosticSFINAEResponse()` classifies each built-in diagnostic as one of:

```cpp
enum SFINAEResponse {
  SFINAE_SubstitutionFailure, // suppress, cause deduction failure
  SFINAE_Suppress,            // suppress entirely (warnings)
  SFINAE_Report,              // always report (fatal errors)
  SFINAE_AccessControl        // report in non-SFINAE, suppress in SFINAE
};
```

Sema checks `DiagnosticIDs::getDiagnosticSFINAEResponse(DiagID)` before emitting diagnostics during substitution and uses `DiagnosticsEngine::setLastDiagnosticIgnored(true)` for suppressed diagnostics so that subsequent notes are also silenced.

---

## Chapter Summary

- `DiagnosticsEngine` is the central hub decoupled from definition (`DiagnosticIDs`), options (`DiagnosticOptions`), and rendering (`DiagnosticConsumer`).
- Built-in diagnostic IDs are generated by TableGen from `.td` files; each carries a class, default severity, format string, SFINAE classification, and group membership.
- Severity routing runs through a `DiagStateMap` that tracks all pragma-induced changes by source location, enabling correct replay after PCH loading.
- `DiagnosticBuilder` is a RAII type that accumulates arguments, ranges, and fix-it hints via `operator<<` and emits the diagnostic in its destructor.
- `FixItHint` models three operations (insert, remove, replace) expressed as `CharSourceRange`/`SourceLocation` pairs with a code string.
- `TextDiagnosticPrinter`, `TextDiagnosticBuffer`, `LogDiagnosticPrinter`, `SARIFDiagnosticPrinter`, and `IgnoringDiagConsumer` cover the standard output modes; custom consumers implement `HandleDiagnostic()`.
- `DiagnosticErrorTrap` provides scoped error detection; `Reset(soft=true)` preserves mappings while clearing counters between incremental compilations.
- Custom diagnostic IDs from plugins and tools are allocated above `diag::DIAG_UPPER_LIMIT`; `CustomDiagDesc` allows group association for user-controllable diagnostics.
- The engine gates pipeline progression via `hasErrorOccurred()` / `hasUnrecoverableErrorOccurred()`, and PCH loading restores serialized diagnostic state through `ASTReader` friendship.
- `PartialDiagnostic` and `SemaDiagnosticBuilder` extend the core builder model for deferred emission scenarios in Sema and GPU code compilation.
- `VerifyDiagnosticConsumer` enables test-driven diagnostic verification via `// expected-error`, `// expected-warning`, and regex variants.
- Diagnostic suppression mapping files allow glob-based suppression of named warning groups in specific source directories, complementing `#pragma clang diagnostic`.
- The `DiagStorageAllocator` free list and lazy allocation ensure near-zero overhead for suppressed diagnostics, which constitute the vast majority of `Diag()` calls in practice.

---

**Cross-references:**
- [Chapter 28 — The Driver](../part-05-clang-frontend/ch28-the-driver.md) — driver-level diagnostic IDs and `DiagnosticDriverKinds.td`
- [Chapter 29 — Source Locations and the SourceManager](../part-05-clang-frontend/ch29-source-locations.md) — `SourceLocation`, `SourceRange`, `CharSourceRange`, and `PresumedLoc` used in caret rendering
- [Chapter 31 — The Lexer and Preprocessor](../part-05-clang-frontend/ch31-lexer-preprocessor.md) — `DiagnosticLexKinds.td` and `#pragma clang diagnostic` handling in the preprocessor
- [Chapter 32 — Parsing](../part-05-clang-frontend/ch32-parsing.md) — `DiagnosticParseKinds.td` and parser error recovery via `DiagnosticErrorTrap`
- [Chapter 33 — Semantic Analysis](../part-05-clang-frontend/ch33-sema.md) — `DiagnosticSemaKinds.td`, SFINAE suppression, template backtrace limits

**Reference links:**
- [`clang/include/clang/Basic/Diagnostic.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/Diagnostic.h)
- [`clang/include/clang/Basic/DiagnosticIDs.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/DiagnosticIDs.h)
- [`clang/include/clang/Basic/DiagnosticOptions.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/DiagnosticOptions.h)
- [`clang/include/clang/Basic/DiagnosticOptions.def`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/DiagnosticOptions.def)
- [`clang/include/clang/Frontend/TextDiagnosticPrinter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/TextDiagnosticPrinter.h)
- [`clang/include/clang/Frontend/TextDiagnosticBuffer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/TextDiagnosticBuffer.h)
- [`clang/include/clang/Frontend/DiagnosticRenderer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/DiagnosticRenderer.h)
- [`clang/include/clang/Frontend/SARIFDiagnosticPrinter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/SARIFDiagnosticPrinter.h)
- [`clang/include/clang/Frontend/SerializedDiagnostics.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/SerializedDiagnostics.h)
- [`clang/include/clang/Frontend/LogDiagnosticPrinter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/LogDiagnosticPrinter.h)
- [`clang/include/clang/Rewrite/Frontend/FixItRewriter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Rewrite/Frontend/FixItRewriter.h)
- [`clang/lib/Frontend/TextDiagnostic.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Frontend/TextDiagnostic.cpp)
- [`clang/lib/Frontend/TextDiagnosticPrinter.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Frontend/TextDiagnosticPrinter.cpp)
- [`clang/include/clang/Basic/DiagnosticGroups.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/DiagnosticGroups.td)
- [`clang/include/clang/Basic/DiagnosticSemaKinds.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/DiagnosticSemaKinds.td)


---

@copyright jreuben11
