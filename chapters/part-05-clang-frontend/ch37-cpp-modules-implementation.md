# Chapter 37 — C++ Modules Implementation

*Part V — Clang Internals: Frontend Pipeline*

C++20 modules represent the most significant structural change to C++ compilation since the introduction of templates. Where `#include` pastes preprocessed text into every translation unit, modules define a named, typed interface that the compiler serializes once and loads deterministically. The result is faster builds, fewer header-order bugs, and a vocabulary for expressing what is public versus private in a library. Clang's implementation spans three subsystems: the preprocessor and parser (which recognize module syntax), the serialization layer (`ASTWriter`/`ASTReader`, which produces and consumes Binary Module Interface files), and the visibility/reachability model woven through `Sema`. This chapter walks every layer of that stack from the user-facing grammar down to on-disk bytes.

---

## 37.1 C++20 Module Grammar and Concepts

### 37.1.1 Module Interface Units

A module interface unit begins with the declaration `export module M;` as the first non-whitespace token after any global module fragment. The file conventionally carries a `.cppm` extension, though Clang identifies it by the `-x c++-module` language mode or the `--precompile` driver command.

```cpp
// math.cppm — module interface unit for "Math"
export module Math;

export int add(int a, int b) { return a + b; }
export constexpr double pi = 3.14159265358979323846;

// Not exported; internal to the module
static int impl_helper(int x) { return x * 2; }
```

The `export module M;` declaration marks the beginning of the module purview. Everything declared before it (but after an optional global module fragment) and not inside an `export` block is attached to the module but not exported.

### 37.1.2 Module Implementation Units

A module implementation unit uses `module M;` without `export`. It may access all names declared in the interface — including non-exported ones — but cannot introduce new exported names.

```cpp
// math_impl.cpp — implementation unit
module Math;

// Can use impl_helper from the interface TU
int large_add(int a, int b) { return impl_helper(a) + impl_helper(b); }
```

Implementation units are compiled independently: they import the BMI of their own module interface and produce a normal object file. They do not produce a BMI themselves.

### 37.1.3 Module Partitions

A module partition is a fragment of a module, identified by the colon syntax `module M:P`. Partitions can be interface partitions (`export module M:P;`) or implementation partitions (`module M:P;`). The primary interface must re-export each partition it wants external consumers to see via `export import :P;`.

```cpp
// math-core.cppm
export module Math:Core;
export int core_add(int a, int b) { return a + b; }

// math.cppm (primary interface)
export module Math;
export import :Core;          // re-exports everything from Math:Core
export double pi = 3.14159;
```

Clang assigns the `ModulePartitionInterface` kind to interface partitions and `ModulePartitionImplementation` to implementation partitions. The partition name is stored in `Module::Name` as `"Math:Core"`, with the colon-delimited form used consistently throughout the serialization layer.

### 37.1.4 Global Module Fragment

The global module fragment is the region between `module;` (a bare module declaration) and `export module M;`. It exists solely to allow `#include` of legacy headers whose macros must be visible during parsing of the module purview, without making those headers part of the module's interface.

```cpp
module;              // begins the global module fragment
#include <cstdio>    // text include: macros and declarations enter GMF
#include <cstring>

export module GMFTest;

export void greet(const char* name) {
    std::printf("Hello, %s! (len=%zu)\n", name, std::strlen(name));
}
```

Declarations in the global module fragment have no module attachment; they are effectively in the global module. In Clang 22, the flag `-fskip-odr-check-in-gmf` (emitted automatically by the driver) suppresses ODR checks for declarations included through the GMF, avoiding false positives when the same C header appears in multiple BMIs.

### 37.1.5 Private Module Fragment

The private module fragment is introduced by `module :private;` inside a module interface unit. All declarations after this marker are visible only within the same TU and do not appear in the BMI. This allows a module to be fully self-contained in a single `.cppm` file while keeping implementation details private.

```cpp
export module PMFTest;

export int public_api();     // declared, exported

module :private;

static int hidden_state = 100;
int public_api() { return hidden_state; }   // defined, never emitted to BMI
```

Clang assigns `PrivateModuleFragment` kind to the `Module` node representing this region. The `ASTWriter` omits declarations with this owning module from the exported declaration table.

### 37.1.6 The `std` Named Module (C++23)

P2465 introduces `import std;` and `import std.compat;` as named modules shipping with the standard library. libc++ 22 ships a `std.cppm` and `std.compat.cppm` under `<libc++ install>/share/libc++/v1/`. Building them:

```bash
clang++ -std=c++23 -x c++-module --precompile \
    $(clang++ --print-file-name=std.cppm) -o std.pcm
clang++ -std=c++23 -fmodule-file=std=/path/to/std.pcm main.cpp std.pcm
```

Support is experimental in Clang 22 — not all standard library headers are modularized, and header units remain the recommended interop path until the libc++ module is declared stable.

### 37.1.7 Modules vs. Clang Header Modules vs. PCH

| Feature | PCH (`-emit-pch`) | Clang Header Modules (`-fmodules`) | C++20 Named Modules |
|---------|-------------------|-------------------------------------|---------------------|
| Grammar | `#include` | `#include` / `@import` | `import M;` |
| Spec | Clang extension | Clang extension | ISO C++20 |
| File | `.pch` | `.pcm` (via module map) | `.pcm` (BMI) |
| Driver key | `-emit-pch` / `-include-pch` | `-fmodules`, `-fmodule-cache-path` | `--precompile`, `-fmodule-file=` |
| Macro export | Yes | Yes | No (macros do not cross module boundaries) |
| ODR isolation | Partial | Partial | Strict |
| Incremental deps | Whole-preamble | Per-module | Per-BMI |

PCH serializes the state of the preprocessor and AST at a single `#include` boundary and is primarily a build-speedup device. Clang header modules use the same `.pcm` on-disk format but are driven by `module.modulemap` files and are transparent to the C++ grammar. C++20 named modules are a language feature with strict attachment semantics and no macro leakage.

---

## 37.2 Module File Formats: The BMI

### 37.2.1 Producing a BMI

The Clang driver translates `clang++ -std=c++20 --precompile foo.cppm -o foo.pcm` into a `clang -cc1` invocation with `-emit-module-interface`:

```
clang -cc1 -emit-module-interface -std=c++20 \
      -fmodules-reduced-bmi -fno-implicit-modules \
      foo.cppm -o foo.pcm
```

The `-fmodules-reduced-bmi` flag (on by default in Clang 22 when using `--precompile`) strips unreachable implementation-only declarations from the BMI, producing a smaller file that speeds subsequent imports. Without it, the full AST is serialized, which enables more complete cross-module debug info but increases BMI size.

Alternatively, a single compilation can emit both an object file and a BMI side-channel with `-fmodule-output`:

```bash
clang++ -std=c++20 -fmodule-output=foo.pcm foo.cppm -c -o foo.o
```

This is the preferred pattern for build systems that want to avoid a two-phase compilation.

### 37.2.2 On-Disk Binary Format

The BMI reuses the same LLVM bitstream container as PCH files, defined in [`clang/include/clang/Serialization/ASTBitCodes.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Serialization/ASTBitCodes.h). The outer container is an LLVM bitcode file with a two-word magic number (`CPCH` for C++/PCH). Inside, named blocks carry typed records:

- `CONTROL_BLOCK_ID` — version metadata, source file list, command-line options, module name (`MODULE_NAME` record), module directory (`MODULE_DIRECTORY` record)
- `UNHASHED_CONTROL_BLOCK_ID` — the `SIGNATURE` record (a 64-bit hash of the module interface content, used for dependency checking), and the `AST_BLOCK_HASH`
- `AST_BLOCK_ID` — the main payload: types, declarations, identifiers, macros, source locations
- `SUBMODULE_BLOCK_ID` — the submodule tree for header-module BMIs (less used for named modules)

The `SIGNATURE` field is an `ASTFileSignature`, a 160-bit (5×32-bit) value computed from the content hash of the serialized AST. Build systems can compare signatures without reparsing: if the signature of a recompiled BMI matches the cached value, downstream consumers need not be rebuilt.

### 37.2.3 The `ModuleFile` Struct

[`clang/include/clang/Serialization/ModuleFile.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Serialization/ModuleFile.h) holds all per-BMI state that `ASTReader` maintains while loading a module chain:

```cpp
class ModuleFile {
  ModuleKind Kind;          // MK_ImplicitModule, MK_ExplicitModule, MK_PCH, …
  std::string FileName;     // absolute path to the .pcm
  FileEntryRef File;        // VFS entry for the file
  ASTFileSignature Signature;
  unsigned Generation;      // load order counter
  uint64_t SizeInBits;      // bitstream size for seeking

  // Source location remapping
  int SLocEntryBaseID;
  SourceLocation::UIntTy SLocEntryBaseOffset;

  // Declaration ID remapping
  unsigned LocalNumDecls;   // number of decls in this file
  // plus type, macro, identifier tables …

  llvm::SetVector<ModuleFile *> Imports;    // direct imports
  llvm::SetVector<ModuleFile *> ImportedBy; // reverse edges
  llvm::SmallVector<ModuleFile *, 16> TransitiveImports;
};
```

When `ASTReader` loads a chain of BMIs (A imports B which imports C), each `ModuleFile` node tracks the offset bases needed to remap `LocalDeclID` values — which are file-local — to `GlobalDeclID` values unique across the entire compilation. This remapping is the core bookkeeping that makes multi-module builds coherent.

The `ModuleManager` (declared in [`clang/include/clang/Serialization/ModuleManager.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Serialization/ModuleManager.h)) owns all live `ModuleFile` instances and maintains a topological ordering of the load sequence. It exposes `visit()` — a depth-first traversal used by `ASTReader` to propagate visibility changes and to run global-index queries across the full module graph. The `GlobalModuleIndex` (stored in `GlobalModuleIndex.pcm` in the module cache) is a pre-built lookup structure mapping identifier strings to the subset of cached modules that declare them, enabling `-fmodules-search-all` to find declarations without loading every module.

### 37.2.4 Module Signature and Dependency Checking

The `ASTFileSignature` is a 160-bit value (five 32-bit words) derived from a hash of the serialized AST block. The Clang 22 implementation computes this using a `llvm::SHA1` digest over the bytes of `AST_BLOCK_ID` after writing, then storing the result in the `SIGNATURE` record before closing the bitstream. This creates a content-addressed identity for each BMI: if the interface changes (new exports, changed signatures, modified inline bodies), the signature changes and all downstream consumers that cached the old signature are invalidated.

Build systems should compare `Signature` values rather than file modification timestamps when deciding whether a downstream BMI or object file is stale. The `clang-scan-deps` output does not directly expose signatures; build systems must read the PCM header (the first few hundred bytes of the LLVM bitstream) to extract the `SIGNATURE` record programmatically. The `llvm-bcanalyzer` tool can dump BMI structure for debugging:

```bash
/usr/lib/llvm-22/bin/llvm-bcanalyzer --dump /tmp/mymodule.pcm 2>&1 | head -40
```

---

## 37.3 The `Module` Class

### 37.3.1 `clang::Module` Identity and Fields

[`clang/include/clang/Basic/Module.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/Module.h) defines `clang::Module`, the in-memory representation of any module — whether a C++20 named module, a Clang header module, or a header unit.

Key fields:

```cpp
std::string Name;             // "Math" or "Math:Core" for partitions
SourceLocation DefinitionLoc; // location of the module declaration keyword
Module *Parent;               // nullptr for top-level modules
std::vector<Module *> SubModules;  // submodules (header modules) or partitions
ModuleKind Kind;
```

The `Name` for a partition includes the colon separator: `"Math:Core"`. `getPrimaryModuleInterfaceName()` strips the colon and suffix for the owning primary module name.

### 37.3.2 `ModuleKind` Enum

```cpp
enum ModuleKind {
  ModuleMapModule,              // driven by module.modulemap (Clang extension)
  ModuleHeaderUnit,             // C++20 header unit (import <vector>;)
  ModuleInterfaceUnit,          // export module M;
  ModuleImplementationUnit,     // module M;
  ModulePartitionInterface,     // export module M:P;
  ModulePartitionImplementation,// module M:P;
  ExplicitGlobalModuleFragment, // module; … export module M;
  PrivateModuleFragment,        // module :private;
  ImplicitGlobalModuleFragment, // synthetic GMF for declarations preceding any module decl
};
```

`isNamedModule()` returns true for `ModuleInterfaceUnit`, `ModuleImplementationUnit`, `ModulePartitionInterface`, and `ModulePartitionImplementation`. `isHeaderUnit()` is `Kind == ModuleHeaderUnit`. `isInterfaceOrPartition()` covers the two interface variants. `isModuleImplementation()` covers both implementation variants. `isModuleMapModule()` covers the older Clang header module system.

### 37.3.3 Module Membership Queries

The `Module` class provides several predicates for the common access patterns in Sema and Serialization:

```cpp
bool isNamedModule() const;        // C++20 named module unit
bool isHeaderUnit() const;         // import <header.h>; style header unit
bool isInterfaceOrPartition() const;
bool isModuleImplementation() const;
bool isModulePartition() const;
bool isNamedModuleInterfaceHasInit() const; // has initializers requiring runtime setup
StringRef getPrimaryModuleInterfaceName() const; // "Math" from "Math:Core"
Module *getGlobalModuleFragment() const;
Module *getPrivateModuleFragment() const;
```

The `getOwningModule()` method on `Decl` returns the `Module*` that owns a declaration. For deserialized declarations, it delegates to `getImportedOwningModule()`, which calls `getOwningModuleSlow()` to walk the module ownership ID stored in the `Decl` bitfield. For locally-parsed declarations, it uses `getLocalOwningModule()`.

### 37.3.4 Module Initialization and Runtime Support

Named modules that contain non-trivially-initializable static-duration variables require the importer to call the module's initializer before any module-scope variables are used. Clang 22 generates an `_GLOBAL__sub_I_<filename>` initializer function in the object file of each module implementation unit that touches such state, mirroring how translation unit static initializers work. The module interface unit's object file additionally defines a `_GLOBAL__sub_I_<modulename>` symbol that CMake-generated link steps reference explicitly.

`Module::isNamedModuleInterfaceHasInit()` returns `true` when Sema detects that the module interface has an `initializer for module M` requirement. The linker error `undefined reference to 'initializer for module MyMath'` seen when `import MyMath;` appears after a textual `#include` arises because the linker cannot find the object file containing that initializer — a reminder that module object files must always be linked alongside module consumers.

---

## 37.4 Module Map Files: The Older Header Module System

### 37.4.1 `module.modulemap` Syntax

Clang's header module system predates C++20 and is driven by `module.modulemap` files. These describe how filesystem headers map to named modules:

```
module POSIX [system] [extern_c] {
  header "sys/types.h"
  header "sys/stat.h"
  export *
}

framework module Foundation {
  umbrella header "Foundation/Foundation.h"
  export *
  module * { export * }
}

explicit module MyLib.Private {
  header "MyLib/Internal.h"
  requires !cplusplus
}
```

Keywords:
- `module` / `framework module` — top-level module declaration
- `header` / `umbrella` — associates specific or whole-directory headers
- `export *` — re-exports all imported names to consumers
- `use ModuleName` — restricts which modules may be imported (with `-fmodules-decluse`)
- `explicit` — module is not implicitly loaded by `#include`
- `requires feature` — conditional module activation

### 37.4.2 `ModuleMap` Class

[`clang/include/clang/Lex/ModuleMap.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/ModuleMap.h) defines `ModuleMap`, which parses and indexes module map files. `parseModuleMapFile()` reads a `module.modulemap`, creating `Module` nodes and populating `HeadersMap` — a `DenseMap<FileEntryRef, SmallVector<KnownHeader, 1>>` that maps each header `FileEntry` to its owning modules.

`KnownHeader` is a tagged pointer pairing a `Module*` with a `ModuleHeaderRole` (one of `NormalHeader`, `TextualHeader`, `PrivateHeader`, `ExcludedHeader`). `findModuleForHeader()` performs the lookup from `HeaderSearch` when the preprocessor encounters `#include` and needs to determine whether to load a module instead of raw text.

### 37.4.3 VFS Overlay and Darwin SDK Modules

Apple's SDK ships module maps under `$(SDKROOT)/usr/include/module.modulemap` and `$(SDKROOT)/System/Library/Frameworks/*/Headers/module.modulemap`. Clang uses a VFS overlay (JSON file passed via `-ivfsoverlay`) to map virtual paths to real SDK paths, enabling module caching without modifying the SDK tree. `-fimplicit-module-maps` instructs Clang to search for `module.modulemap` files automatically in all header search paths, the primary enabler for transparent PCM caching under `-fmodules`.

Non-modular headers — those not covered by any module map — remain as text includes. `-Wincomplete-umbrella` warns when an umbrella directory contains headers not covered by any module declaration.

---

## 37.5 `HeaderSearch` and Import Processing

### 37.5.1 `HeaderFileInfo`

[`clang/include/clang/Lex/HeaderSearch.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/HeaderSearch.h) defines `HeaderFileInfo`, a 16-byte per-file record cached at `HeaderSearch::FileInfo[FileID]`:

```cpp
struct HeaderFileInfo {
  unsigned DirInfo        : 3;  // SrcMgr::CharacteristicKind
  unsigned isModuleHeader : 1;  // header is owned by a module map module
  unsigned isTextualModuleHeader : 1;
  unsigned isCompilingModuleHeader : 1; // this is the header being compiled for a PCM
  // ... isImport, isPragmaOnce, NumIncludes, ...
};
```

When `HeaderSearch::findModuleForHeader()` locates a `KnownHeader` for a `FileEntry`, it marks `isModuleHeader = 1`. The flag `isCompilingModuleHeader` is set during the compilation of a header module's PCM to suppress recursive loading.

### 37.5.2 Processing `import` Directives

When the preprocessor encounters `import MyMath;`, control flows through:

1. **`Preprocessor::Lex()`** returns a `tok::kw_import` token from the lexer. The lexer's `HandleIdentifier` function sets `tok::annot_module_name` for the module-name token sequence.

2. **`Preprocessor::HandleImportDirective()`** — parses the module name tokens, calls `ModuleLoader::loadModule()` to request the BMI be loaded, and injects a `tok::annot_module_include` annotation token into the token stream.

3. **`CompilerInstance::loadModule()`** (which implements `ModuleLoader`) checks whether the module is already loaded (via the `ModuleManager`), then calls `ASTReader::ReadAST()` for the corresponding `.pcm` file. On success, it marks the `Module*` as loaded and returns it wrapped in `ModuleLoadResult`.

4. **`Sema::ActOnModuleImport()`** receives the `Module*` from the parser and records the import in the translation unit's import list, setting visibility for all exported declarations in the imported module.

The `-fprebuilt-module-path=DIR` flag tells `CompilerInstance::loadModule()` to search `DIR` for `ModuleName.pcm` files. The `-fmodule-file=Name=path.pcm` flag provides an explicit, per-name override. In Clang 22 the prebuilt path search does not automatically resolve partition names — each partition BMI must be provided explicitly with `-fmodule-file=M:P=path.pcm`.

### 37.5.3 `ModuleLoader` Interface

`ModuleLoader` is the pure virtual interface that `CompilerInstance` implements:

```cpp
class ModuleLoader {
public:
  virtual ModuleLoadResult
  loadModule(SourceLocation ImportLoc, ModuleIdPath Path,
             Module::NameVisibilityKind Visibility,
             bool IsInclusionDirective) = 0;
  virtual void makeModuleVisible(Module *Mod,
                                 Module::NameVisibilityKind Visibility,
                                 SourceLocation ImportLoc) = 0;
  virtual GlobalModuleIndex *loadGlobalModuleIndex(SourceLocation) = 0;
};
```

The `Visibility` parameter distinguishes `AllVisible` (a direct `import M;`) from `MacrosVisible` (used in some header module compatibility paths). `makeModuleVisible` propagates visibility to the `Sema` layer; the actual declaration-level visibility is managed by `Sema::VisibleModules`.

### 37.5.4 Token-Level Import Handling in the Preprocessor

The lexer in C++20 mode has special handling for `import` and `export` at the start of a logical line. Before C++20, these are ordinary identifiers. In C++20 mode, the preprocessor's `LexHeaderName()` path detects the `import` identifier followed by a module-name sequence and sets `tok::annot_module_name` on the composite token. The actual module-name parsing is delegated to the parser: `Parser::ParseModuleImport()` reconstructs the `ModuleIdPath` (a vector of `(IdentifierInfo*, SourceLocation)` pairs) from the annotation token payload.

The preprocessor callback `PPCallbacks::moduleImport()` fires for each `import` directive, enabling tools like `clang-tidy` and `clangd` to observe module dependency information during indexing without rerunning dependency scanning. The `InclusionDirective()` callback is also fired for compatibility with tools that monitor all include-like directives.

For `export import :P;` (re-exporting a partition), the parser calls `Sema::ActOnModuleImport()` with `IsExported=true`, which records the partition in the primary module's exported imports list. This information is serialized into the BMI so that consumers of the primary module implicitly receive all re-exported partition declarations.

---

## 37.6 Compiling Module Interface Units

### 37.6.1 Driver and cc1 Flags

The end-to-end command to produce a BMI:

```bash
clang++ -std=c++20 -x c++-module --precompile math.cppm -o math.pcm
```

The driver translates `--precompile` to `-emit-module-interface` in the `cc1` invocation. The language mode `-x c++-module` sets `InputKind::CXXModule`. With `-fmodule-output` instead of `--precompile`, a single cc1 run emits both the BMI and the object file:

```bash
clang++ -std=c++20 -fmodule-output=math.pcm math.cppm -c -o math.o
```

### 37.6.2 Parsing the Module Interface

At the cc1 level, compilation of a module interface unit proceeds identically to a normal TU through lexing and parsing. The difference surfaces when the parser sees `export module M;`:

1. `Parser::ParseTopLevelDecl()` calls `Parser::ParseModuleDecl()` which produces a `Module *` via `Sema::ActOnModuleDecl()`.
2. `Sema::ActOnModuleDecl()` creates or looks up the `Module` node in `ModuleMap`, sets its `Kind` to `ModuleInterfaceUnit`, and records the current compilation as building that module.
3. Subsequent exported declarations are parsed inside an `ExportDecl` node when they appear inside `export { … }` or are individually prefixed with `export`. `Sema::ActOnStartExportDecl()` opens an `ExportDecl`; `Sema::ActOnFinishExportDecl()` closes it.

### 37.6.3 `ExportDecl` and Export Semantics

`ExportDecl` (defined in [`clang/include/clang/AST/Decl.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Decl.h)) is a `DeclContext` that wraps exported declarations:

```cpp
class ExportDecl final : public Decl, public DeclContext {
  SourceLocation ExportLoc;
  // ...
};
```

All declarations lexically inside an `ExportDecl` receive module linkage attachment. Their visibility is marked as exported when the BMI is written. Declarations outside any `ExportDecl` but within the module purview are module-private: they are serialized into the BMI for use by implementation units of the same module but are not visible to external importers.

### 37.6.4 Header Unit Compilation

Header units are compiled with `-fmodule-header`:

```bash
clang++ -std=c++20 -fmodule-header myheader.h -o myheader.pcm
# or for system headers:
clang++ -std=c++20 -fmodule-header=system -xc++-system-header vector -o vector.pcm
```

The cc1 flag is `-emit-header-unit`. Header unit BMIs have `ModuleHeaderUnit` kind. All declarations in the header — including those from transitively included headers — are attached to the header unit and exported. Macros defined in the header are also exported, which is the key distinction from named module interfaces where macros never cross the boundary. This makes header units a lower-friction migration step: existing code using `#include <vector>` can be mechanically changed to `import <vector>;` without source modifications to the header itself.

### 37.6.5 End-of-Translation-Unit Actions

After parsing the entire module interface file, `Sema::ActOnEndOfTranslationUnit()` performs deferred instantiation of templates, checks for deferred diagnostics, and then transfers control to `ASTWriter::WriteAST()` which serializes the complete AST to the output `.pcm` file.

---

## 37.7 `ASTWriter` and `ASTReader`

### 37.7.1 `ASTWriter` Architecture

[`clang/lib/Serialization/ASTWriter.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Serialization/ASTWriter.cpp) implements module serialization. The top-level entry point is:

```cpp
ASTFileSignature ASTWriter::WriteAST(
    llvm::PointerUnion<Sema *, Preprocessor *> Subject,
    StringRef OutputFile,
    Module *WritingModule,
    StringRef isysroot,
    ArrayRef<std::shared_ptr<ModuleFileExtension>> Extensions,
    bool hasErrors);
```

`WriteAST` calls `WriteASTCore()`, which orchestrates:

1. **Control block** — version strings, source file list, options hash
2. **Unhashed control block** — signature (computed after writing everything else), AST block hash
3. **Preprocessor state** — macro definitions, identifier table (`WriteIdentifierTable()`), pragma records
4. **Types** — `WriteType(ASTContext&, QualType)` for each type node, with abbreviations for common cases (pointers, references, builtin types)
5. **Declarations** — `WriteDeclAndTypes()` drains the `DeclsToWrite` queue; each decl calls `WriteDecl(ASTContext&, Decl*)`, which dispatches by decl kind. Exported declarations are flagged so that `ASTReader` can reconstruct visibility on load.
6. **Identifier lookup tables** — `WriteIdentifierTable()` serializes the identifier hash table enabling fast lookup of exported names by identifier without loading every declaration.

### 37.7.2 Deferred Declaration Queue

`ASTWriter` does not serialize declarations eagerly. `GetDeclRef(const Decl *D)` assigns a `LocalDeclID` (a file-local integer) to `D` and enqueues it in `DeclsToWrite`; the actual bytes are written later in `WriteDeclAndTypes()`. This deferred model ensures that cross-references between declarations serialize consistently regardless of traversal order.

The mapping `DeclIDs: DenseMap<const Decl *, LocalDeclID>` is the canonical source of truth during writing. After `WriteAST` completes, the file offset of each declaration is recorded in the `DeclOffsets` table inside `AST_BLOCK_ID`, enabling O(1) seeking by `LocalDeclID`.

### 37.7.3 `ASTReader` Architecture

[`clang/lib/Serialization/ASTReader.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Serialization/ASTReader.cpp) is the largest source file in the Clang codebase (~10,000 lines). The entry point is:

```cpp
ASTReader::ASTReadResult ASTReader::ReadAST(
    StringRef FileName, ModuleKind Type,
    SourceLocation ImportLoc,
    unsigned ClientLoadCapabilities,
    SmallVectorImpl<ImportedSubmodule> *Imported = nullptr);
```

`ReadAST` calls `ReadASTCore()` which:

1. Opens the bitstream, validates the magic number and version.
2. Reads the `CONTROL_BLOCK_ID` to verify compatibility and populate `ModuleFile::FileName`, `Signature`, `Generation`.
3. Reads the `AST_BLOCK_ID` lazily: type and declaration offsets are read, but individual records are not deserialized until requested.
4. Calls `ReadASTBlock()` to populate the preprocessor state, identifier tables, source location base offsets, and macro tables.
5. Returns `Success`; declarations are deserialized on-demand via `GetDecl(GlobalDeclID)`.

### 37.7.4 `GlobalDeclID` and `LocalDeclID` Remapping

Each `ModuleFile` contributes a contiguous range of `GlobalDeclID` values. `ASTReader` tracks `ModuleFile::BaseDeclID` — the first `GlobalDeclID` assigned to declarations in that file. The mapping `LocalDeclID → GlobalDeclID` is a simple addition: `global = base + local`. The inverse (finding which `ModuleFile` owns a given `GlobalDeclID`) uses a sorted range table searched with `upper_bound`.

When multiple BMIs are loaded (e.g., `import A; import B;` where A also imports B), `ASTReader` detects that B's declarations already have `GlobalDeclID` assignments from the first load and merges the `ModuleFile` graphs without re-registering IDs.

### 37.7.5 Reading Types and Declarations

`ASTReader::ReadDecl(ModuleFile &F, const RecordDataImpl &R, unsigned &I)` reads a `LocalDeclID` from `R[I]` and maps it to a `GlobalDeclID` via the offset table, then calls `GetDecl(GlobalDeclID)` which deserializes on first access:

```cpp
Decl *ASTReader::ReadDeclRecord(GlobalDeclID ID) {
  // locate ModuleFile, seek to DeclOffset
  // call ASTDeclReader::Visit() dispatching on DeclCode
  // attach owning module, set visibility flags
}
```

`ReadType()` similarly maps a `LocalTypeID` via the type offset table and deserializes using `ASTTypeReader`.

### 37.7.6 Merging Declarations Across Modules

When the same entity — for example, a class template specialization or a `constexpr` function — is declared in multiple imported modules (e.g., both `import A;` and `import B;` where both A and B import C which defines the entity), `ASTReader` merges the declarations. `ASTDeclReader::mergeRedecl()` chains the `Decl` redeclaration list across `ModuleFile` boundaries. The `GlobalDeclID` from the first-loaded module becomes canonical; subsequent loads detect that the entity already has an ID and reuse it, updating the `Decl*` pointer table.

This merge logic is critical for templates: `ClassTemplateDecl` nodes from different modules that represent the same primary template must be merged so that instantiation decisions are shared. `ASTReader::CompleteRedeclChain()` rebuilds the chain after all modules are loaded, ensuring that `Decl::getMostRecentDecl()` and `Decl::getPreviousDecl()` traverse the correct history regardless of load order.

The `RelatedDeclsMap` in `ASTWriter` records which auxiliary declarations (e.g., implicit constructors, deduction guides) are associated with a given declaration, ensuring they are deserialized together and not orphaned when the owning decl is loaded lazily.

---

## 37.8 Name Visibility and Reachability

### 37.8.1 The Module Ownership Model

Every declaration is *attached* to exactly one module. The attachment is recorded in `Decl`'s module ID bitfield and retrieved via `Decl::getOwningModule()`. For declarations in a named module, `getOwningModuleForLinkage()` returns the primary interface module (stripping partition information) because linkage is per-module, not per-partition.

```cpp
Module *Decl::getOwningModuleForLinkage() const;
```

Declarations in the global module fragment have no owning module (`getOwningModule() == nullptr`). Declarations in the private module fragment are attached to the `PrivateModuleFragment` module node and are never exported.

### 37.8.2 Visible vs. Reachable

C++20 distinguishes two access levels:

- **Visible**: the name can be found by unqualified or qualified lookup. Only exported names of directly-imported modules are visible.
- **Reachable**: the semantic properties of a declaration are available (its type, members, template arguments) even if the name is not visible. A non-exported type used in the return type of an exported function is reachable.

`Sema::isVisible(const NamedDecl *D)` checks whether `D`'s owning module is in the set of visible modules (`Sema::VisibleModules`). `Sema::isReachable(const NamedDecl *D)` additionally checks whether `D` was introduced by a visible declaration (e.g., the return type of a visible function):

```cpp
bool Sema::isVisible(const NamedDecl *D) {
  return !D->isInvisibleOutsideTheOwningModule() ||
         isModuleVisible(D->getOwningModule());
}

bool Sema::isReachable(const NamedDecl *D) {
  return isVisible(D) || hasReachableDeclarationSlow(D, /*Modules=*/nullptr);
}
```

The practical consequence: callers can use an `InternalType` returned from an exported function (it is reachable), but they cannot name it directly in their own code (it is not visible).

### 37.8.3 `VisibleModules` and Transitive Visibility

`Sema` maintains a `VisibleModulesSet` that tracks which modules are currently visible. `Sema::VisibleModules` is populated by `makeModuleVisible()` calls and is consulted by `isVisible()`. Crucially, visibility is transitive for re-exported modules: if module A does `export import B;`, then importing A automatically makes B's exported declarations visible.

The transitivity is implemented in `CompilerInstance::loadModule()` → `Sema::ActOnModuleImport()` → `Sema::makeModuleVisible()`, which recursively calls `makeModuleVisible` for every module that the newly-imported module re-exports. `Module::Exports` (a `SmallVector<ExportDecl, 2>`) records explicit `export import` statements, and `Module::DirectUses` tracks `use` declarations from module maps. Both are traversed during visibility propagation.

### 37.8.4 ADL Across Module Boundaries

Argument-dependent lookup (ADL) finds functions in the namespaces of argument types regardless of module boundaries, but only if those functions are visible. If `Geometry::Point` is exported from `import Geometry` but `Geometry::distance()` is not exported, ADL will not find `distance` even if the call is `distance(p1, p2)`. This is a deliberate tightening of ADL semantics compared to the header world.

### 37.8.5 Inline Functions and Owning Modules

Inline functions defined in a module interface have their definitions serialized into the BMI and are visible to importers. The inline function is attached to the module interface unit. If the same inline function is also defined in the global module fragment (via an included header), Clang must reconcile the definitions; `-fskip-odr-check-in-gmf` suppresses the ODR comparison for GMF-sourced definitions.

### 37.8.6 `friend` Declarations Across Modules

A `friend` declaration inside a class template does not automatically export the friend function to external importers. The friend is reachable (the class layout depends on it for member-access purposes) but not visible. To make a friend function visible, it must be separately exported with its own `export` declaration:

```cpp
export module MyMod;

export class Widget {
    friend void swap(Widget &, Widget &); // friend — reachable, not visible
};

export void swap(Widget &, Widget &);     // makes swap visible to importers
```

---

## 37.9 Global Module Fragment and `extern "C"` Interaction

### 37.9.1 Header Units vs. Text Includes

A *header unit* (`import <cstdio>;` or `import "myheader.h";`) is a separately-compiled unit with `ModuleHeaderUnit` kind. The header's declarations become part of the header unit's interface and are subject to the same visibility rules as named modules. Macros defined in a header unit are exported.

A *text include* in the global module fragment (`#include <cstdio>` before `export module M;`) simply pastes the preprocessed text of the header into the GMF. Macro definitions flow into the module purview. Declarations are attached to the global module fragment (no module owner) and are thus universally accessible.

The distinction matters for ODR: using both `import <cstdio>` and `#include <cstdio>` in the same program risks double-definition if any declarations are non-inline. The recommended pattern is to choose one or the other per translation unit.

### 37.9.2 `extern "C"` and Module Attachment

`extern "C"` declarations are attached to the enclosing module if they appear in the module purview, or to the global module if in the GMF. C-linkage functions declared in the GMF have no owning module, making them globally accessible like traditional C headers — which is precisely why the GMF pattern is the correct bridge from legacy C headers.

When `-fmodule-file=POSIX=/path/to/posix.pcm` provides a modularized C header unit, `extern "C"` names like `printf` become part of the POSIX module's interface and are visible only after `import POSIX;`. Without module mapping, `#include <cstdio>` in the GMF continues to work unchanged.

### 37.9.3 Interaction with Implicit Modules and `-fimplicit-modules`

When `-fimplicit-modules` is active (the default for Clang header modules, not for C++20 named modules), an `import` directive for a name that has no pre-built BMI triggers on-demand compilation of the module from its module map sources. This implicit compilation runs in a subprocess (or in-process for certain build configurations) and caches the result under `-fmodules-cache-path`. The cache key is derived from the module map file content, target triple, language options, and preprocessor macro definitions.

C++20 named modules deliberately do not support implicit compilation — there is no discovery mechanism analogous to header file paths for finding a `.cppm` file given a module name. Every named module BMI must be provided explicitly through `-fmodule-file=Name=path` or `-fprebuilt-module-path=DIR`. This constraint enables reproducible, deterministic builds but shifts discovery responsibility entirely to the build system.

Mixed environments — where a codebase uses both `-fmodules` for system headers and C++20 named modules for project code — are supported in Clang 22 by using `-fimplicit-module-maps` (for the system side) alongside explicit `-fmodule-file=` flags (for project modules). The two systems use the same `.pcm` file format and the same `ASTReader`/`ASTWriter` machinery, but they are driven by different discovery mechanisms and their module objects carry different `ModuleKind` values.

---

## 37.10 Incremental Builds and Dependency Scanning

### 37.10.1 `clang-scan-deps` and P1689

[ISO P1689](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p1689r5.html) defines a JSON format for expressing C++20 module dependencies. `clang-scan-deps` implements this format:

```bash
clang-scan-deps -format=p1689 -- clang++ -std=c++20 math.cppm
```

Output:

```json
{
  "revision": 0,
  "rules": [
    {
      "primary-output": "a.out",
      "provides": [
        {
          "is-interface": true,
          "logical-name": "Math",
          "source-path": "/path/to/math.cppm"
        }
      ]
    }
  ],
  "version": 1
}
```

For a consumer TU:

```bash
clang-scan-deps -format=p1689 -- clang++ -std=c++20 main.cpp
# produces "requires": [{"logical-name": "Math"}]
```

The `provides` field names modules produced; `requires` names those consumed. Build systems use this to order BMI compilation before object compilation.

### 37.10.2 `DependencyScanningTool` API

For programmatic use, [`clang/include/clang/Tooling/DependencyScanningTool.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Tooling/DependencyScanningTool.h) provides:

```cpp
class DependencyScanningTool {
public:
  std::optional<P1689Rule>
  getP1689ModuleDependencyFile(const CompileCommand &Command,
                               StringRef CWD,
                               std::string &MakeformatOutput,
                               std::string &MakeformatErrorOutput);
};
```

`P1689Rule` holds the `primary-output`, optional `Provides` (`P1689ModuleInfo` with `logical-name` and `source-path`), and a vector of `Requires`. The underlying `DependencyScanningWorker` runs a lightweight compilation pipeline using `DependencyScanningFilesystem` — a caching overlay that avoids re-reading unchanged files — making repeated scans cheap.

### 37.10.3 CMake 3.28 and Ninja 1.11 Integration

CMake 3.28 added native C++20 module support via `target_sources(... FILE_SET CXX_MODULES FILES ...)`. During configuration, CMake runs `clang-scan-deps` in P1689 mode on each source file to extract the module dependency graph. At build time, Ninja 1.11's dynamic dependency support (the `dyndep` mechanism) reads the scanner output to schedule BMI compilation before object compilation, even within the same target.

The workflow:

```cmake
add_library(MathLib)
target_sources(MathLib
  PUBLIC FILE_SET CXX_MODULES FILES math.cppm math-core.cppm
  PRIVATE math_impl.cpp)
target_compile_features(MathLib PUBLIC cxx_std_20)
```

CMake generates a `CXXModules.json` scan database and passes `-fmodule-file=Math=/path/math.pcm` to each consumer compilation after `math.pcm` is up to date.

### 37.10.4 `-fprebuilt-module-path` and Explicit Module Files

For build systems that do not use CMake or ninja dyndep, the simplest strategy is:

1. Compile all BMIs first in dependency order.
2. Pass `-fprebuilt-module-path=DIR` to all consumer compilations, where `DIR` contains `ModuleName.pcm` files.
3. Or pass `-fmodule-file=Name=path.pcm` per module for precise control.

`-fprebuilt-module-path` probes `DIR/ModuleName.pcm` for each `import ModuleName;`. If the file exists and its signature is compatible, it is loaded. Mismatched signatures produce an error; `-fmodules-disable-diagnostic-validation` silences option-mismatch diagnostics (not recommended in production).

---

## 37.11 Clang Header Modules: The Older System

### 37.11.1 `-fmodules` Mode

Clang header modules predate C++20 by a decade. They are enabled by `-fmodules` (or `-fcxx-modules` for C++) and require either an implicit or explicit `module.modulemap` to describe header-to-module mappings. The module cache lives in `-fmodules-cache-path=DIR` (default: `~/.clang/ModuleCache`).

Under `-fimplicit-modules`, Clang automatically searches for `module.modulemap` in every header search path. When a `#include <vector>` is processed and `vector` is covered by a modulemap, Clang compiles a PCM for the owning module on-demand and caches it. Subsequent compilations with matching command-line options load the cached PCM.

### 37.11.2 PCM Validation

On each PCM load, `ASTReader` checks:

- **Signature match**: the `SIGNATURE` in the `UNHASHED_CONTROL_BLOCK_ID` must match what would be computed from current inputs.
- **Input file timestamps/sizes**: each `InputFile` record is re-stat'd. Mismatches trigger recompilation of the PCM unless `-fmodules-validate-once-per-build-session` is set.
- **Command-line options**: the serialized options in `CONTROL_BLOCK_ID` must match (modulo a whitelist of innocuous flags). `-fmodules-disable-diagnostic-validation` disables the diagnostic subset.

### 37.11.3 Darwin SDK Modules and `-fapinotes`

Apple's Darwin SDK ships with module maps for all system frameworks and libc. When Clang targets Apple platforms (or when `-isysroot` points to an Apple-style SDK), `-fimplicit-module-maps` is effectively enabled, and the system module map at `$(SDKROOT)/usr/include/module.modulemap` governs all C stdlib headers.

API Notes (`.apinotes` files, loaded via `-fapinotes`) extend module declarations with source-compatibility annotations (nullability, availability, swift rename), stored separately from module maps to avoid modifying vendor SDKs.

### 37.11.4 Migration Path to C++20 Named Modules

Header modules do not have a direct upgrade path to C++20 named modules; they serve different purposes. Header modules transparently accelerate `#include`-based code. Named modules require explicit `export module M;` authoring. The practical migration strategy:

1. Wrap existing public headers in a thin C++20 module interface that `#include`s them in the global module fragment and re-exports the desired names.
2. Progressively move implementation into module implementation units.
3. Once all public names are explicitly exported, remove the GMF includes.

---

## 37.12 Practical Issues and Diagnostics

### 37.12.1 Import Ordering Restrictions

The C++20 standard requires that all `import` declarations in a non-module TU appear before any non-import declarations (§10.3). Clang diagnoses violations:

```
error: 'import' declaration must precede all declarations in the global module fragment
```

In practice, mixing `#include` and `import` in the same TU is permitted — the preprocessor processes them in order — but `import` must appear before any non-preprocessing declarations that would be affected. The linker error about `initializer for module MyMath` appearing with `import` after `#include <cstdio>` reflects that the import initialization code is not correctly sequenced when a textual include precedes the import.

### 37.12.2 Partition Visibility Constraints

Partitions are not visible outside the primary module. External consumers `import Math;` (not `import Math:Core;`). Attempting to directly import a partition from outside the module:

```
error: module partition 'Math:Core' can only be imported from within module 'Math'
```

### 37.12.3 Private Module Fragment Rules

The private module fragment may only appear in a module interface unit that is not a partition. It must appear after all export declarations. Clang 22 diagnoses:

```
error: private module fragment cannot appear in a module that is not the primary module interface unit
```

Definitions in the PMF that are referenced by exported inline functions produce errors because the PMF definition would not be available to importers who instantiate the inline.

### 37.12.4 Deferred Instantiation Across Module Boundaries

Template instantiations triggered by an `import` are deferred to the importer's end-of-TU phase. If the imported module defines a `constexpr` function used in a `static_assert` in the importer, the assertion is evaluated during the importer's `Sema::ActOnEndOfTranslationUnit()`, not during the BMI load. This ordering is correct per standard but can cause confusion when instantiation errors appear far from the import site.

### 37.12.5 `[[clang::internal_linkage]]` and Module Boundaries

`[[clang::internal_linkage]]` (spelled `__attribute__((internal_linkage))` in pre-C++11 mode) forces a function or variable to have internal linkage even when declared in a named module interface. This is useful for module-private utility functions that must not be visible via symbol tables, complementing the language-level privacy of non-exported names.

The relationship between module linkage and C++ linkage deserves careful attention. A non-exported name in a module interface has *module linkage* — it is accessible to other translation units in the same named module (implementation units) but not to external consumers. This is stronger than internal linkage (translation-unit scope) but weaker than external linkage. `[[clang::internal_linkage]]` overrides this to true internal linkage, preventing the symbol from appearing in the object file's symbol table at all.

For exported inline functions, Clang emits the function body into both the BMI (for importers to instantiate/inline) and the object file of the module interface unit (as a weak linkage symbol for ODR compliance). Consumers that do not inline the function link against the module's object file. This explains why `clang++ -fmodule-file=M=M.pcm consumer.cpp M.pcm` must include `M.pcm`'s corresponding object file (`M.o`) in the link step — the inline function bodies must be available even when the compiler decides not to inline them.

### 37.12.6 Known Limitations in Clang 22

- **Standard library modules**: `import std;` is functional but requires a separately-compiled `std.pcm` from libc++; it is not automatically provided. The module compilation is slow on the first build.
- **Incomplete toolchain support**: some sanitizer instrumentation passes do not fully account for module ownership, producing spurious warnings with `-fsanitize=undefined` in modular builds.
- **Header unit macro export**: macros exported by header units are not propagated through re-imported chains (e.g., `export import <cassert>;` in a module interface does not propagate `assert` macro to consumers of the module).
- **Incremental compilation granularity**: changing a non-exported implementation detail in the module purview (but not the PMF) still rebuilds the BMI because the full module purview is serialized. The PMF exists precisely to avoid this; migrating implementation definitions to the PMF is the recommended pattern.
- **`-fmodule-output` interaction with LTO**: combined object + BMI output (`-fmodule-output`) is incompatible with `-flto=thin` in Clang 22; use the two-phase `--precompile` then compile approach for LTO builds.

---

## Chapter Summary

- C++20 modules define a strict interface/implementation split through `export module M;`, `module M;`, partitions (`module M:P;`), the global module fragment (`module;` preamble), and the private module fragment (`module :private;`).
- Clang produces BMIs with `-emit-module-interface` (driver flag `--precompile`); the BMI is an LLVM bitstream file reusing the PCH container format, stamped with a cryptographic signature for incremental build correctness.
- `clang::Module` represents every module kind through the `ModuleKind` enum; `Decl::getOwningModule()` tracks declaration attachment; `Sema::isVisible()` and `Sema::isReachable()` implement the two-tier access model.
- `ASTWriter::WriteAST()` serializes the AST lazily through a `DeclsToWrite` deferred queue; `ASTReader::ReadAST()` loads BMIs lazily, remapping `LocalDeclID` to `GlobalDeclID` through per-`ModuleFile` base offsets.
- The global module fragment bridges legacy `#include` headers into the modular world; `extern "C"` declarations in the GMF have no module attachment and remain globally accessible.
- `clang-scan-deps` with `-format=p1689` provides the dependency edges needed by build systems (CMake 3.28, Ninja 1.11) to correctly order BMI compilation.
- Clang's older header module system (`-fmodules`, `module.modulemap`) is distinct from C++20 named modules and remains the primary mechanism for transparently accelerating `#include`-heavy codebases on Apple platforms.
- Clang 22 has known gaps: standard library modules require explicit `std.pcm` preparation, header unit macro propagation through re-export chains is incomplete, and `-fmodule-output` is incompatible with ThinLTO.

---

*Cross-references:*
- [Chapter 29 — SourceManager, FileEntry, SourceLocation](ch29-sourcemanager-fileentry-sourcelocation.md) — `SourceLocation` remapping during BMI load; `SLocEntryBaseOffset` in `ModuleFile`
- [Chapter 31 — The Lexer and Preprocessor](ch31-the-lexer-and-preprocessor.md) — `HandleImportDirective()`, module-name annotation tokens, macro handling in the global module fragment
- [Chapter 32 — The Parser](ch32-the-parser.md) — `ParseModuleDecl()`, `ParseTopLevelDecl()` dispatch for module declarations
- [Chapter 36 — The Clang AST in Depth](ch36-the-clang-ast-in-depth.md) — `ExportDecl` node, `Decl::getOwningModule()`, visibility attributes

*Reference links:*
- [`clang/include/clang/Basic/Module.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/Module.h) — `clang::Module`, `ModuleKind`
- [`clang/include/clang/Serialization/ASTWriter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Serialization/ASTWriter.h) — `ASTWriter::WriteAST()`, `WriteDecl()`, `WriteType()`
- [`clang/include/clang/Serialization/ASTReader.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Serialization/ASTReader.h) — `ASTReader::ReadAST()`, `GlobalDeclID`/`LocalDeclID` remapping
- [`clang/include/clang/Serialization/ModuleFile.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Serialization/ModuleFile.h) — `ModuleFile`, `ASTFileSignature`
- [`clang/include/clang/Serialization/ASTBitCodes.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Serialization/ASTBitCodes.h) — on-disk block and record IDs
- [`clang/include/clang/Lex/HeaderSearch.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/HeaderSearch.h) — `HeaderSearch`, `HeaderFileInfo`
- [`clang/include/clang/Lex/ModuleMap.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/ModuleMap.h) — `ModuleMap`, `KnownHeader`
- [`clang/include/clang/Frontend/CompilerInstance.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Frontend/CompilerInstance.h) — `loadModule()`, `loadModuleFile()`
- [`clang/include/clang/Tooling/DependencyScanningTool.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Tooling/DependencyScanningTool.h) — `DependencyScanningTool`, `P1689Rule`
- [`clang/include/clang/AST/Decl.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/Decl.h#L5131) — `ExportDecl`
- [P1689R5 — Format for describing dependencies of source files](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p1689r5.html)
