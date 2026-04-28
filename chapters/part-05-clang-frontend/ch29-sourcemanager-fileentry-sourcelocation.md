# Chapter 29 — SourceManager, FileEntry, and SourceLocation

*Part V — Clang Internals: Frontend Pipeline*

A C++ translation unit is not a single text file. Before a single token has been parsed, the preprocessor has concatenated headers pulled in by `#include`, spliced text injected by `-include` command-line flags, replaced macro invocations with their expansions, and annotated the resulting character stream with synthetic line-number directives. The compiler's diagnostic, debug-info, and code-coverage infrastructure must be able to answer one fundamental question at any moment: for this particular byte in the input stream, what was the original file, line, and column? Clang's answer is a three-component system consisting of `SourceLocation`, `FileEntry`/`FileManager`, and `SourceManager`. Understanding these three types in depth is a prerequisite for writing any Clang library component, tool, or plugin that touches source positions, and it is the foundation on which every subsequent chapter of Part V is built.

---

## The Fundamental Problem: Virtual vs. Physical Character Stream

The C preprocessor model specifies that translation proceeds on a "translation unit" that is conceptually a flat sequence of characters. In reality that sequence is produced dynamically: `#include` directives splice file contents inline, macro invocations replace identifier tokens with potentially many expansion tokens, and `#line` directives can rewrite the apparent file name and line number for any subsequent text. A naive compiler that records only a byte offset into this virtual stream cannot reconstruct the original source position from that offset alone.

Clang solves this by treating the virtual character stream as a concatenation of numbered *source location entries* (SLocEntries). Each entry covers a contiguous range of the virtual address space and records whether its bytes come from a physical file or from a macro expansion. The 32-bit `SourceLocation` is simply an integer offset into this virtual address space. Given a `SourceLocation`, the `SourceManager` can, in O(log n) time, find the owning SLocEntry and from it reconstruct the file path, line number, column number, and the full include/expansion stack.

This design means the compiler can pass source positions cheaply (a single 32-bit word) and pay the cost of decoding them only when producing diagnostics, writing debug info, or responding to an IDE query — which is exactly the right trade-off for a compiler.

### Concrete SLocEntry Layout: A Worked Example

Consider a minimal translation unit with two files and one macro:

```
// foo.h  (23 bytes, including newline)
#define DOUBLE(x) ((x)+(x))

// main.cpp  (45 bytes)
#include "foo.h"
int main() { return DOUBLE(42); }
```

When `SourceManager` processes this translation unit, it builds the following `LocalSLocEntryTable`:

| Index | FileID | Offset  | Kind      | Details |
|-------|--------|---------|-----------|---------|
| 0     | —      | 0       | invalid   | sentinel entry, never used |
| 1     | FID 1  | 1       | File      | `main.cpp`, IncludeLoc=invalid (main file) |
| 2     | FID 2  | 47      | File      | `foo.h`, IncludeLoc=FID1:offset 1 (at `#include`) |
| 3     | FID 3  | 71      | Expansion | SpellingLoc=FID2:offset 9 (inside macro def), ExpansionStart=FID1:offset 32 (at `DOUBLE`), ExpansionEnd=FID1:offset 37 |

Here offset 1 is the start of `main.cpp` in the virtual space (the source manager starts at 1 to keep 0 as the invalid sentinel). Offset 47 = 1 + 45 (main.cpp) + 1 (separation guard byte). Offset 71 = 47 + 23 (foo.h) + 1.

A `SourceLocation` with raw encoding 71 + 4 = 75 has its macro bit clear (it is less than 2^31) so it is a `FileID` location. `getFileID(75)` binary-searches `LocalLocOffsetTable` — a sorted copy of just the offsets — and finds that 75 falls inside the range [71, 71+23), returning FID 3. But FID 3 is an expansion entry, so the source manager must use it as a macro expansion record rather than a file record. If you then call `getSpellingLoc(Loc)`, it returns a location in FID 2 (inside the macro definition in `foo.h`). If you call `getExpansionLoc(Loc)`, it returns a location in FID 1 (inside `main.cpp` at the `DOUBLE` call site).

This layout is why macro-location arithmetic works: the virtual address for each expansion token is derived from the expansion entry's offset plus the token's position within the macro body, encoding the semantic relationship between spelling and expansion in the numeric value of the `SourceLocation` itself.

---

## `SourceLocation`: A 32-Bit Address in the Virtual Stream

`SourceLocation` is defined in [`clang/include/clang/Basic/SourceLocation.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/SourceLocation.h). Its entire data representation is a single `uint32_t` named `ID`.

```cpp
class SourceLocation {
public:
  using UIntTy = uint32_t;
  using IntTy  = int32_t;
private:
  UIntTy ID = 0;
  enum : UIntTy { MacroIDBit = 1ULL << (8 * sizeof(UIntTy) - 1) };
  // ...
};
```

The high bit of `ID` is the *macro bit*. When it is zero, the location refers to a position in a physical file; when it is one, the location refers to a position inside a macro expansion record. The remaining 31 bits are the byte offset into the virtual address space. This gives a maximum translation-unit size of 2 GiB of source text — more than sufficient for any realistic input.

### Validity, File vs. Macro

The sentinel value zero is the invalid location:

```cpp
bool isValid()   const { return ID != 0; }
bool isInvalid() const { return ID == 0; }
bool isFileID()  const { return (ID & MacroIDBit) == 0; }
bool isMacroID() const { return (ID & MacroIDBit) != 0; }
```

An "invalid" `SourceLocation` is frequently used to indicate that no position is known — for example, a diagnostic originating from a command-line flag, or a synthesized AST node that has no corresponding source text.

### Serialization

Because `SourceLocation` is 32 bits, it fits in a pointer-sized slot on 64-bit platforms and is trivially passed by value. For contexts where even the type must be opaque (such as certain LLVM table-gen or hash-map uses), the raw encoding can be extracted and restored:

```cpp
SourceLocation::UIntTy raw = Loc.getRawEncoding();
// ... store in DB, send over IPC, etc.
SourceLocation restored = SourceLocation::getFromRawEncoding(raw);
```

The pointer-flavored variants (`getPtrEncoding` / `getFromPtrEncoding`) double-cast through `uintptr_t` to suppress compiler warnings on platforms where `sizeof(void*)` differs from `sizeof(uint32_t)`.

### `SourceRange` and `CharSourceRange`

A `SourceRange` pairs a begin and end `SourceLocation`. By convention, the end location points to the *start of the last token* of the range, not to the character past the end. This token-range convention means that expanding the range to character granularity requires knowing the token length, which in turn requires calling the lexer.

`CharSourceRange` wraps a `SourceRange` with a boolean `IsTokenRange` to distinguish the two conventions:

```cpp
// Token range: end is start of last token
CharSourceRange TR = CharSourceRange::getTokenRange(BeginLoc, EndLoc);

// Char range: end is one-past-last character
CharSourceRange CR = CharSourceRange::getCharRange(BeginLoc, EndLoc);
```

The distinction matters whenever source text is extracted: `Lexer::getSourceText(CharSourceRange, SM, LO)` handles both modes.

### `FullSourceLoc`

`FullSourceLoc` bundles a `SourceLocation` with a pointer to its owning `SourceManager`, enabling standalone use without always threading a manager reference through call chains:

```cpp
FullSourceLoc FSL(Loc, SM);
unsigned Line = FSL.getSpellingLineNumber();
unsigned Col  = FSL.getSpellingColumnNumber();
PresumedLoc PL = FSL.getPresumedLoc();
```

`FullSourceLoc` also provides `isBeforeInTranslationUnitThan()` and a `BeforeThanCompare` functor suitable for `std::sort` and sorted containers.

---

## `FileID` and `SLocEntry`: The Index Table

### `FileID`

`FileID` is a thin wrapper around a signed 32-bit integer. Positive values index into `LocalSLocEntryTable`; negative values (excluding -1) index into `LoadedSLocEntryTable`; zero is invalid; -1 is the sentinel. The sign convention is internal and clients should not depend on the numeric value, but it explains the common pattern:

```cpp
if (FID.getOpaqueValue() >= -1)
    // local or no entry
```

### `SLocEntry`

`SLocEntry` is the fundamental record type stored in the SLocEntry tables:

```cpp
class SLocEntry {
  SourceLocation::UIntTy Offset : 31;  // offset in virtual address space
  SourceLocation::UIntTy IsExpansion : 1;
  union {
    FileInfo      File;
    ExpansionInfo Expansion;
  };
};
```

Two subtypes discriminated by `IsExpansion`:

**`FileInfo`** represents a physical file or memory buffer. Its fields are:
- `IncludeLoc` — the `SourceLocation` of the `#include` directive that brought this file in (invalid for the main file)
- `NumCreatedFIDs` — the count of FileIDs allocated during preprocessing of this file, used to compute ranges
- `HasLineDirectives` — set when the file contains `#line` or GNU linemarker directives
- `ContentAndKind` — a `PointerIntPair<ContentCache*, 3, CharacteristicKind>` packing a pointer to the content cache alongside the `C_User`/`C_System`/`C_ExternCSystem` flag

**`ExpansionInfo`** represents one macro expansion. Its fields are:
- `SpellingLoc` — where the token characters physically reside (inside the macro definition, for a macro body expansion; at the call site argument, for a macro argument expansion)
- `ExpansionLocStart`, `ExpansionLocEnd` — the begin/end of the expansion site in the enclosing file
- `ExpansionIsTokenRange` — false only for token-split records (`>>` split into two `>` tokens)

A critical size constraint governs both types:

```cpp
static_assert(sizeof(FileInfo) <= sizeof(ExpansionInfo),
              "FileInfo must be no larger than ExpansionInfo.");
```

Every unloaded macro expansion (and there can be millions in large TUs) is stored in this union. Keeping `ExpansionInfo` small is a real memory concern; extra data for `FileInfo` goes in `ContentCache` which is allocated separately.

### Local vs. Loaded Tables

`SourceManager` maintains two distinct tables:

```cpp
SmallVector<SrcMgr::SLocEntry, 0> LocalSLocEntryTable;
llvm::PagedVector<SrcMgr::SLocEntry, 32> LoadedSLocEntryTable;
```

`LocalSLocEntryTable` holds entries created during the current compilation. Its entries are allocated at offsets starting from 1, growing upward. `LoadedSLocEntryTable` holds entries imported from precompiled headers or modules. Its entries are allocated at offsets starting from `MaxLoadedOffset` (= 2^31) and growing *downward*. The two spaces meet in the middle. Local offsets satisfy `offset < NextLocalOffset`; loaded offsets satisfy `offset >= CurrentLoadedOffset`. This split allows the source manager to be extended with module/PCH data without disturbing local offset assignments.

---

## `ContentCache`: Lazy Buffer Management

`ContentCache` (in `SourceManager.h`, namespace `SrcMgr`) is the object that actually owns the `MemoryBuffer` for a file. It is allocated from `SourceManager::ContentCacheAlloc` (a `BumpPtrAllocator`) and referenced by pointer from `FileInfo::ContentAndKind`. Key fields:

```cpp
class ContentCache {
  mutable std::unique_ptr<llvm::MemoryBuffer> Buffer;
public:
  OptionalFileEntryRef OrigEntry;     // entry as originally looked up
  OptionalFileEntryRef ContentsEntry; // entry whose bytes we actually read
  StringRef Filename;
  mutable LineOffsetMapping SourceLineCache; // lazily computed line offsets
  unsigned BufferOverridden : 1;
  unsigned IsFileVolatile  : 1;
  unsigned IsTransient     : 1;
  mutable unsigned IsBufferInvalid : 1;
  // ...
};
```

`Buffer` is loaded lazily: the first call to `getBufferOrNone()` opens the file, reads it, and stores the `MemoryBuffer`. Subsequent calls return the cached buffer. `SourceLineCache` is computed even more lazily — only when `getLineNumber()` is called — by scanning the buffer for `\n` characters and storing the resulting byte-offset array in the bump allocator. This means the first `getLineNumber()` call on a large file does O(N) work, but all subsequent calls on the same file are binary searches into the cached array.

`OrigEntry` vs. `ContentsEntry` handles `overrideFileContents()`: if the source manager is told to substitute file A's bytes with file B's bytes, `OrigEntry` records A and `ContentsEntry` records B. `OrigEntry` is still used for diagnostic messages (reporting the file name the user wrote), while `ContentsEntry` is used for reading data.

---

## `FileManager` and `FileEntry`

### `FileManager`

`FileManager` (defined in [`clang/include/clang/Basic/FileManager.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/FileManager.h)) is the layer between Clang's compilation logic and the operating system's filesystem. It wraps an `llvm::vfs::FileSystem` and provides:

1. **Uniquing** — two lookups of the same inode return the same `FileEntry*`, even if reached through different path spellings (symlinks, relative paths, etc.)
2. **Caching** — stat results and opened file descriptors are cached across the compilation
3. **Virtual filesystem overlay** — the underlying `llvm::vfs::FileSystem` can be a layered composition that remaps, overlays, or replaces files

Construction requires a `FileSystemOptions` and optionally an explicit VFS:

```cpp
// Real filesystem (default)
auto FM = llvm::makeIntrusiveRefCnt<FileManager>(FileSystemOptions{});

// In-memory overlay for testing or clangd
llvm::IntrusiveRefCntPtr<llvm::vfs::OverlayFileSystem> OverlayFS(
    new llvm::vfs::OverlayFileSystem(llvm::vfs::getRealFileSystem()));
auto InMemFS = llvm::makeIntrusiveRefCnt<llvm::vfs::InMemoryFileSystem>();
InMemFS->addFile("/virtual/foo.h", 0,
                 llvm::MemoryBuffer::getMemBuffer("#pragma once\n"));
OverlayFS->pushOverlay(InMemFS);
auto FM = llvm::makeIntrusiveRefCnt<FileManager>(FileSystemOptions{}, OverlayFS);
```

The primary lookup entry points are:

```cpp
llvm::Expected<FileEntryRef>     FM.getFileRef(StringRef Filename,
                                               bool OpenFile = false,
                                               bool CacheFailure = true);
OptionalFileEntryRef             FM.getOptionalFileRef(StringRef Filename);
llvm::Expected<DirectoryEntryRef> FM.getDirectoryRef(StringRef DirName);
```

### `FileEntry`

`FileEntry` holds the inode-level identity of a file. It is intentionally not constructible by clients (the constructor is `private`; `FileManager` is a friend). Its public API:

```cpp
class FileEntry {
public:
  StringRef  tryGetRealPathName()   const; // may be empty
  off_t      getSize()              const;
  unsigned   getUID()               const; // small integer, assigned by FileManager
  const llvm::sys::fs::UniqueID &getUniqueID() const; // (device, inode)
  time_t     getModificationTime() const;
  const DirectoryEntry *getDir()   const;
  bool isNamedPipe()               const;
  bool isDeviceFile()              const;
};
```

Two files are the "same" if and only if `&entry1 == &entry2` — pointer identity guarantees inode-level deduplication, which is what makes the `FileInfos` map in `SourceManager` work correctly.

### `FileEntryRef`: The Modern Reference Type

Since LLVM 16, the preferred type for passing file references is `FileEntryRef`, not raw `FileEntry*`. `FileEntryRef` is a thin wrapper around a pointer to a `StringMap` entry, where the key is the path name as presented to `FileManager`. This retains name information that `FileEntry` discards:

```cpp
// getName() returns the path used in this lookup
StringRef name = Ref.getName();
// getNameAsRequested() returns the name before VFS redirections
StringRef req  = Ref.getNameAsRequested();
// Identity comparison: same inode
bool sameFile = (Ref1 == Ref2);
// Same reference: identical path string
bool sameRef  = Ref1.isSameRef(Ref2);
```

`FileEntryRef` is the same size as `const FileEntry*` (asserted by a `static_assert`), trivially copyable, and implicitly convertible to `const FileEntry*` for backward compatibility during incremental adoption.

`OptionalFileEntryRef` is a specialized `Optional<FileEntryRef>` that avoids storage overhead by using a null pointer to represent the absent state — also guaranteed by `static_assert`.

---

## `SourceManager`: Construction and Core API

`SourceManager` is the central hub of all source-position bookkeeping. It is declared in [`clang/include/clang/Basic/SourceManager.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/SourceManager.h) and is reference-counted via `llvm::RefCountedBase<SourceManager>`.

### Construction

```cpp
DiagnosticsEngine Diags(DiagIDs, &DiagOpts, new TextDiagnosticPrinter(llvm::errs(), &DiagOpts));
FileManager FM(FileSystemOptions{});
SourceManager SM(Diags, FM);
```

The `bool UserFilesAreVolatile` parameter (default false) marks all non-system files as volatile, which tells the content cache to stat and re-open files on every access rather than caching the result — useful for language servers that run while the user is editing.

### Creating FileIDs

A `FileID` is created by registering a file or memory buffer with the source manager:

```cpp
// From a file on disk
auto FileOrErr = FM.getFileRef("foo.cpp");
if (auto FE = llvm::expectedToOptional(FileOrErr)) {
    FileID FID = SM.createFileID(*FE, SourceLocation(),
                                 SrcMgr::C_User);
    SM.setMainFileID(FID);
}

// From a memory buffer (e.g., for unit tests or generated code)
auto Buf = llvm::MemoryBuffer::getMemBuffer("int x = 0;\n", "<generated>");
FileID FID = SM.createFileID(std::move(Buf));
```

`createFileID` appends a new `SLocEntry` to `LocalSLocEntryTable`, assigns it an offset `NextLocalOffset`, advances `NextLocalOffset` by the file's size plus one (the +1 prevents adjacent files from sharing offsets), and returns the corresponding `FileID`.

For macro expansions, the preprocessor calls:

```cpp
// Body expansion: SpellingLoc points into the macro definition
SourceLocation ExpStart =
    SM.createExpansionLoc(SpellingLoc, ExpansionStart, ExpansionEnd, Length);

// Argument expansion: SpellingLoc points to the argument at the call site
SourceLocation ArgStart =
    SM.createMacroArgExpansionLoc(SpellingLoc, ExpansionLoc, Length);
```

Both allocate a new `SLocEntry` of type `ExpansionInfo` and return its start location.

### FileID Lookups

```cpp
FileID FID = SM.getMainFileID();
FileID FID = SM.getFileID(Loc);        // binary search by offset
OptionalFileEntryRef FE = SM.getFileEntryRefForID(FID);
StringRef bufName = SM.getBufferName(Loc);
```

`getFileID` is the hot path: it first checks a one-entry cache (`LastFileIDLookup` + offset range), then falls back to a binary search of `LocalLocOffsetTable`, an in-parallel offset array kept sorted to speed up lookups without requiring a full scan of `LocalSLocEntryTable`.

---

## Spelling vs. Expansion Locations: The Core Duality

Every macro-expanded token has two distinct positions:

- **Spelling location** — where the token's characters physically reside in the source file. For a macro body expansion of `X` in `#define DOUBLE(x) x + x`, the spelling location of the `+` is inside the macro definition text.
- **Expansion location** — where the macro was invoked in the user's source. For `DOUBLE(42)` on line 17, the expansion location of every token emitted is on line 17 at the `DOUBLE` identifier.

The `SourceManager` exposes these through symmetric API pairs:

```cpp
SourceLocation Spelling = SM.getSpellingLoc(Loc);
SourceLocation Expansion = SM.getExpansionLoc(Loc);
```

Both functions are optimized for the common case (a file location, which is already both its own spelling and expansion location) with an inline fast path:

```cpp
SourceLocation getSpellingLoc(SourceLocation Loc) const {
    if (Loc.isFileID()) return Loc;
    return getSpellingLocSlowCase(Loc);
}
```

### Expansion Range

For a macro invocation like `MAX(a, b)`, the expansion *range* covers the entire call from `M` to `)`. `getImmediateExpansionRange` returns this range for the innermost expansion:

```cpp
CharSourceRange Range = SM.getImmediateExpansionRange(Loc);
SourceLocation Start = Range.getBegin();
SourceLocation End   = Range.getEnd();
bool IsToken = Range.isTokenRange();
```

`getExpansionRange(Loc)` walks outward through all expansion levels until it finds a file location, returning the final range.

### Macro Argument vs. Macro Body

For function-like macros, a token can appear in two ways:

```cpp
#define F(x) f(x)
F(expr);
// 'expr' token: spelling = where 'expr' was written at the call site
//               expansion = where 'x' appears in the macro body
```

The distinction is encoded in `ExpansionInfo::isMacroArgExpansion()`, which checks whether `ExpansionLocEnd` is invalid. The `SourceManager` exposes:

```cpp
bool isMacroArg = SM.isMacroArgExpansion(Loc);
bool isMacroBody = SM.isMacroBodyExpansion(Loc);
```

`isMacroArgExpansion` also accepts an optional `SourceLocation *StartLoc` out-parameter that is set to the start of the macro-argument expansion if the function returns true.

### `getFileLoc`

When you need a single "best" physical location for a macro location — one that can be used for file-system operations or line-number lookup — `getFileLoc` walks through expansions: for a macro-argument expansion, it returns the spelling location (the call site); for a macro-body expansion, it returns the expansion location (also the call site, which is more useful for diagnostics):

```cpp
SourceLocation PhysLoc = SM.getFileLoc(Loc);
assert(PhysLoc.isFileID());
```

---

## Line/Column Decomposition

### `getDecomposedLoc`

The primitive decomposition operation returns the FileID and byte offset within that file:

```cpp
auto [FID, Offset] = SM.getDecomposedLoc(Loc);
// Offset = bytes from start of file buffer
```

For macro locations, this decomposes the location in its current form (expansion or spelling). Two convenience wrappers exist:

```cpp
auto [FID, Offset] = SM.getDecomposedSpellingLoc(Loc);
auto [FID, Offset] = SM.getDecomposedExpansionLoc(Loc);
```

### `getLineNumber` and `getColumnNumber`

```cpp
unsigned Line = SM.getLineNumber(FID, Offset);   // 1-based
unsigned Col  = SM.getColumnNumber(FID, Offset); // 1-based, byte offset
```

`getLineNumber` is not cheap on first call: it triggers `ContentCache::SourceLineCache` computation by scanning the buffer for `\n`. Subsequent calls binary-search the cached table. The method also maintains a locality cache (`LastLineNoFileIDQuery`, `LastLineNoFilePos`, `LastLineNoResult`) that handles the extremely common case of sequential queries into the same file.

`getColumnNumber` does not need the line table; it simply scans backward from the given offset to the preceding `\n`, making it O(column_number) rather than O(file_size).

The spelling/expansion variants compose both operations:

```cpp
unsigned SpellingLine = SM.getSpellingLineNumber(Loc);
unsigned SpellingCol  = SM.getSpellingColumnNumber(Loc);
unsigned ExpLine = SM.getExpansionLineNumber(Loc);
unsigned ExpCol  = SM.getExpansionColumnNumber(Loc);
```

### `getPresumedLoc` and the Line Table

`getPresumedLoc` is what diagnostics should use when reporting locations to the user. Unlike `getLineNumber`, it respects `#line` directives and GNU linemarker flags, returning the line number the programmer intended:

```cpp
PresumedLoc PL = SM.getPresumedLoc(Loc);
if (PL.isValid()) {
    llvm::errs() << PL.getFilename() << ':'
                 << PL.getLine()     << ':'
                 << PL.getColumn()   << '\n';
}
```

`PresumedLoc` also carries an `IncludeLoc` pointing to the `#include` site (modified by any enclosing `#line` directives), which the diagnostic engine uses to build the "In file included from …" chain.

When `UseLineDirectives` is false, `getPresumedLoc` ignores `#line` directives and reports the physical file and line.

### `LineTableInfo` and `#line` Directives

When the preprocessor encounters a `#line N "filename"` directive or a GNU linemarker of the form `# N "filename" flags`, it calls `SourceManager::AddLineNote`:

```cpp
void AddLineNote(SourceLocation Loc, unsigned LineNo, int FilenameID,
                 bool IsFileEntry, bool IsFileExit,
                 SrcMgr::CharacteristicKind FileKind);
```

This appends a `LineEntry` to the `LineTableInfo` object (stored in `SourceManager::LineTable`). A `LineEntry` records the file-offset at which the directive appears, the new presumed line number, a filename table ID (obtained from `getLineTableFilenameID`), and three boolean flags corresponding to the GNU linemarker flag values:

- Flag **1** (IsFileEntry): this is an include transition entering a new file
- Flag **2** (IsFileExit): this is a return from an included file
- Flag **3** (system header): subsequent lines should be treated as system-header content (`C_System`)
- Flag **4** (extern C): implies flag 3 plus `C_ExternCSystem` treatment

`getPresumedLoc` works by first resolving the expansion location (for macro IDs), then looking up whether the enclosing `FileInfo` has `HasLineDirectives` set. If it does, `LineTableInfo::FindNearestLineEntry` binary-searches the line entry table for the last entry at or before the current file offset. The resulting `LineEntry` overrides the physical filename and line number in the returned `PresumedLoc`.

`FileInfo::HasLineDirectives` is set by `SourceManager::AddLineNote` via `setHasLineDirectives()`. The flag allows `getPresumedLoc` to skip the `LineTableInfo` lookup entirely for the vast majority of files that contain no `#line` directives, keeping the common case fast.

---

## Practical Source Manager Queries

### Reverse Lookup: From File/Line/Column to SourceLocation

Tools that consume compiler output — clang-tidy, clangd, IDEs, fault-localizers — frequently need to convert a `file:line:col` triple into a `SourceLocation`. The canonical API for this is:

```cpp
// Look up the FileEntry, then translate
const FileEntry *FE = SM.getFileManager()
                       .getOptionalFileRef("path/to/file.cpp")
                       .value().getFileEntry();

// Translate file + line + col to SourceLocation
SourceLocation Loc = SM.translateFileLineCol(FE, Line, Col);

// If you already have the FileID:
FileID FID = SM.translateFile(FE);
SourceLocation Loc2 = SM.translateLineCol(FID, Line, Col);
```

`translateFileLineCol` looks up the first `FileID` for the given `FileEntry` (in case the same file was included multiple times, it returns the location in the first occurrence). `translateLineCol` is more precise when you already know the `FileID`, and is the form used internally by clangd when mapping LSP `TextDocumentPositionParams` to AST locations.

A related query, `getMacroArgExpandedLocation`, handles the case where you have a file location pointing into a macro argument and you want to find the expanded location inside the macro body — needed for macro-aware rename operations:

```cpp
// Loc points at 'expr' in F(expr); where F is a macro
SourceLocation ExpandedLoc = SM.getMacroArgExpandedLocation(Loc);
// ExpandedLoc now points into the macro body where the parameter was substituted
```

This function uses the per-FileID `MacroArgsMap` cache to avoid scanning expansion records on every call.

### Extracting Source Text

Given a `CharSourceRange`, the spell-checked source text is:

```cpp
#include "clang/Lex/Lexer.h"

CharSourceRange CSR = CharSourceRange::getTokenRange(Begin, End);
bool Invalid = false;
StringRef Text = Lexer::getSourceText(CSR, SM, LangOpts, &Invalid);
if (!Invalid)
    llvm::outs() << Text << '\n';
```

For token ranges, `Lexer::getSourceText` internally calls `Lexer::MeasureTokenLength` to find the end of the last token before computing the slice.

### Ordering Two Locations

The canonical ordering query is:

```cpp
bool LhsFirst = SM.isBeforeInTranslationUnit(LhsLoc, RhsLoc);
```

This is the most expensive source-manager query: when both locations are in the same FileID, it reduces to offset comparison. When they are in different FileIDs, the implementation walks the include/expansion chains to find the nearest common ancestor in the include tree, then compares offsets in that common file. An `InBeforeInTUCacheEntry` keyed by `(FileID, FileID)` is consulted first; on a cache miss the traversal is performed and the result cached. For hot loops over tokens, prefer decomposing locations with `getDecomposedLoc` and comparing raw `(FileID, offset)` pairs directly.

### Checking Same File

```cpp
// Fast: same FileID means same physical file (no include, no macro crossing)
bool same = SM.isWrittenInSameFile(Loc1, Loc2);

// Slightly more nuanced: accounts for #line directives
bool same = SM.getPresumedLoc(Loc1).getFileID() ==
            SM.getPresumedLoc(Loc2).getFileID();
```

### Walking the Include Chain

`getIncludeLoc(FID)` returns the source location of the `#include` directive that brought `FID` into the translation unit. Iterating the chain from a known `FileID`:

```cpp
void printIncludeStack(FileID FID, const SourceManager &SM) {
    while (FID.isValid()) {
        if (auto FE = SM.getFileEntryRefForID(FID))
            llvm::errs() << "  " << FE->getName() << '\n';
        SourceLocation IncLoc = SM.getIncludeLoc(FID);
        if (IncLoc.isInvalid()) break;
        FID = SM.getFileID(IncLoc);
    }
}
```

The diagnostic engine's "In file included from" notes are emitted by exactly this traversal; see `DiagnosticRenderer::emitIncludeStack` in `clang/lib/Frontend/DiagnosticRenderer.cpp`.

### Example: Printing Macro Expansion Locations in a Clang Tool

The following example uses `PPCallbacks::MacroExpands` — the real hook for intercepting macro uses — to print the file, line, and column of every macro call site. There is no per-token `PPCallbacks` hook; the preprocessor does not expose individual token events. For per-token processing, use `Preprocessor::Lex` directly in a loop (see Chapter 31).

```cpp
#include "clang/Basic/SourceManager.h"
#include "clang/Frontend/FrontendAction.h"
#include "clang/Lex/MacroInfo.h"
#include "clang/Lex/PPCallbacks.h"
#include "clang/Lex/Preprocessor.h"

struct MacroExpansionPrinter : public clang::PPCallbacks {
  const clang::SourceManager &SM;
  explicit MacroExpansionPrinter(const clang::SourceManager &SM) : SM(SM) {}

  void MacroExpands(const clang::Token &MacroNameTok,
                    const clang::MacroDefinition &MD,
                    clang::SourceRange Range,
                    const clang::MacroArgs *Args) override {
    clang::SourceLocation Loc = MacroNameTok.getLocation();
    // Expansion location: where the macro was written in source
    clang::PresumedLoc PL = SM.getPresumedLoc(SM.getExpansionLoc(Loc));
    // Spelling location: inside the macro definition (often different file)
    clang::SourceLocation SpelledLoc = SM.getSpellingLoc(Loc);

    if (PL.isValid()) {
      llvm::outs() << "Macro '" << MacroNameTok.getIdentifierInfo()->getName()
                   << "' expanded at " << PL.getFilename()
                   << ':' << PL.getLine()
                   << ':' << PL.getColumn() << '\n';
      // Also show the spelling (definition) location if it differs
      if (SpelledLoc != SM.getExpansionLoc(Loc)) {
        clang::PresumedLoc SPL = SM.getPresumedLoc(SpelledLoc);
        if (SPL.isValid())
          llvm::outs() << "  defined at " << SPL.getFilename()
                       << ':' << SPL.getLine() << '\n';
      }
    }
  }
};

// Registration in a FrontendAction:
//   PP.addPPCallbacks(
//       std::make_unique<MacroExpansionPrinter>(CI.getSourceManager()));
```

---

## The `#include` Chain and Macro Expansion Trace

### Reconstructing Include Depth

The include chain is encoded in the `FileInfo::IncludeLoc` field of every `SLocEntry`. The main file's entry has an invalid `IncludeLoc`. Walking up the chain:

```cpp
SourceLocation Loc = /* some location */;
FileID CurFID = SM.getFileID(SM.getSpellingLoc(Loc));

while (CurFID.isValid()) {
    SourceLocation IncLoc = SM.getIncludeLoc(CurFID);
    if (IncLoc.isInvalid()) break; // reached main file
    PresumedLoc PL = SM.getPresumedLoc(IncLoc);
    llvm::errs() << "In file included from "
                 << PL.getFilename() << ':'
                 << PL.getLine() << ":\n";
    CurFID = SM.getFileID(IncLoc);
}
```

### Tracing Macro Expansions

For a macro-expanded token, the expansion chain can be multiple levels deep when macros invoke other macros. `getImmediateMacroCallerLoc` steps one level up:

```cpp
SourceLocation getImmediateMacroCallerLoc(SourceLocation Loc) const;
```

This function returns: for a macro-argument expansion, the spelling location; for a macro-body expansion, the expansion location of the immediately enclosing macro. By calling it repeatedly until a file location is reached, the full expansion stack is traversed:

```cpp
void printMacroStack(SourceLocation Loc, const SourceManager &SM) {
    while (Loc.isMacroID()) {
        if (SM.isMacroArgExpansion(Loc)) {
            llvm::errs() << "  (macro argument)\n";
            Loc = SM.getSpellingLoc(Loc);
        } else {
            CharSourceRange R = SM.getImmediateExpansionRange(Loc);
            PresumedLoc PL = SM.getPresumedLoc(R.getBegin());
            llvm::errs() << "  expanded from macro at "
                         << PL.getFilename() << ':'
                         << PL.getLine() << '\n';
            Loc = SM.getImmediateMacroCallerLoc(Loc);
        }
    }
}
```

Note that `MacroInfo` — the struct that records a macro's parameter list and definition body — lives in the `Preprocessor`, not in `SourceManager`. The source manager records only positions; semantic macro metadata is orthogonal to location management.

---

## Token Splitting and `createTokenSplitLoc`

A special case that requires its own `ExpansionInfo` variant is *token splitting*. The canonical example is the `>>` token in `vector<vector<int>>`: the lexer produces a single `>>` token, but the C++11 parser needs two `>` tokens to close two nested template argument lists. Splitting a token changes its syntactic role without changing the source bytes.

`SourceManager::createTokenSplitLoc` creates an `ExpansionInfo` record with `ExpansionIsTokenRange = false`:

```cpp
SourceLocation createTokenSplitLoc(SourceLocation SpellingLoc,
                                   SourceLocation TokenStart,
                                   SourceLocation TokenEnd);
```

This calls `ExpansionInfo::createForTokenSplit(SpellingLoc, Start, End)` and allocates a new SLocEntry at `NextLocalOffset`. The resulting location has its macro bit set (it is technically an expansion record), but its expansion range spans only a *character* range (the `>>` bytes), not a token range. Diagnostics that call `getSpellingLoc` on a split-token location get the position of the original `>>`, which is exactly what you want for an "expected `>` to close template argument list" error.

The same mechanism is used in a few other places where the parser needs to split a token the lexer produced atomically: `->*` splitting, `<:` and other digraph handling, and UCN (universal character name) decomposition. In all cases, the underlying character data is unmodified; only the source-position record is split.

## Address Space Exhaustion

With 31 bits available for local offsets, the virtual address space for local SLocEntries is 2 GiB. In practice, the offset consumed per token is not 1 byte of source text but rather 1 byte per byte of the underlying MemoryBuffer — so macro expansion entries consume space proportional to the macro body size, not the invocation length. Template-heavy C++ code with deep macro instantiation chains can exhaust this space.

When `NextLocalOffset` approaches `CurrentLoadedOffset`, `SourceManager` emits a diagnostic and clamps further allocations. The diagnostic infrastructure includes:

```cpp
void noteSLocAddressSpaceUsage(DiagnosticsEngine &Diag,
                               std::optional<unsigned> MaxNotes = 32) const;
```

This function iterates the `LocalSLocEntryTable`, finds the entries that consumed the most address space, and emits notes identifying the heavy files or macro expansions. The `-fsource-location-overflow-test` internal flag artificially reduces the address space limit to trigger overflow testing.

In practice, standard library headers (especially `<type_traits>` and `<tuple>`) with their deep template expansions are the most common culprits. The fix is usually to reduce macro expansion depth or to use `extern template` declarations to avoid re-instantiating templates. Some code generators that emit synthetic C++ (protobuf, swig) have historically hit this limit.

## PCH and Modules Integration

### The Loaded SLocEntry Space

When a precompiled header or module is imported, `ASTReader` calls `SourceManager::createFileID` and `createExpansionLoc` with non-zero `LoadedID` and `LoadedOffset` parameters. These parameters direct the source manager to place the new entry at a specific offset in the *loaded* portion of the address space — starting at `MaxLoadedOffset` (= 2^31) and growing *downward* as successive modules are imported.

```cpp
static const SourceLocation::UIntTy MaxLoadedOffset =
    1ULL << (8 * sizeof(SourceLocation::UIntTy) - 1);
```

The loaded table is a `llvm::PagedVector<SLocEntry, 32>` — entries are allocated in 32-element pages, avoiding a large up-front heap allocation while still permitting O(1) random access by index. Indexing follows the convention: for a `FileID` with negative ID `N` (where `N < -1`), the index into `LoadedSLocEntryTable` is `(-N - 2)`.

Not all loaded entries are immediately materialized. The `SLocEntryLoaded` and `SLocEntryOffsetLoaded` bit vectors track per-entry load status. `SLocEntryOffsetLoaded[i]` is set as soon as the entry's offset is known (needed for binary search), while `SLocEntryLoaded[i]` is set only when the full `FileInfo` or `ExpansionInfo` is deserialized. The `ExternalSLocEntrySource` interface (implemented by `ASTReader`) is called on demand:

```cpp
class ExternalSLocEntrySource {
  // Called when a full entry needs to be materialized
  virtual bool ReadSLocEntry(int ID) = 0;
  // Called to map a loaded offset back to a FileID
  virtual int getSLocEntryID(SourceLocation::UIntTy SLocOffset) = 0;
  // Called to check if a location came from a module vs. a PCH
  virtual std::pair<SourceLocation, StringRef>
      getModuleImportLoc(int ID) = 0;
};
```

`ASTReader` populates `LoadedSLocEntryTable` in reverse order (highest offset first) when reading the AST file's SOURCE_LOCATION_OFFSETS block. Each AST file is assigned a contiguous range within the loaded half. The `LoadedSLocEntryAllocBegin` array records the first `FileID` of each such allocation, enabling `getUniqueLoadedASTFileID(Loc)` to identify which AST file a loaded location came from — important for module-cache invalidation.

The two-space design guarantees that source locations from a PCH and from the current translation unit never overlap. A location is known to be from a loaded AST if and only if its raw encoding is `>= CurrentLoadedOffset`.

### `MacroArgsMap` and `getMacroArgExpandedLocation`

For function-like macros, each parameter reference in the macro body corresponds to an argument expansion `SLocEntry`. When the preprocessor expands `F(expr)`, it creates one `SLocEntry` of type `isMacroArgExpansion()` for each use of the parameter `x` in the macro body, with `SpellingLoc` pointing to `expr` in the call site and `ExpansionLoc` pointing to the parameter reference in the body.

`getMacroArgExpandedLocation(Loc)` takes a spelling location pointing inside a macro argument (i.e., a file location within the bytes of `expr`) and returns the corresponding expansion location inside the macro body. The implementation uses a per-FileID cache:

```cpp
mutable llvm::DenseMap<FileID, std::unique_ptr<MacroArgsMap>>
    MacroArgsCacheMap;
```

where `MacroArgsMap` is `std::map<unsigned /*offset*/, SourceLocation>`. On first call for a given `FileID`, the source manager scans the `LocalSLocEntryTable` for all `isMacroArgExpansion()` entries whose spelling FileID matches, and builds the sorted map. Subsequent calls binary-search the cached map. This lazy construction trades startup cost for query speed — important for clangd rename operations that may call this function thousands of times.

### Preamble and `overrideFileContents`

clangd's preamble mechanism pre-compiles the stable prefix of a source file (the set of `#include` directives at the top that rarely change). The preamble is stored as a serialized AST in the module cache. When the user makes an edit that does not affect the preamble region, clangd reloads the preamble's SLocEntries into `LoadedSLocEntryTable` via the `ExternalSLocEntrySource` interface, then sets up a fresh `SourceManager` for the post-preamble body.

For in-memory edits, `SourceManager::overrideFileContents()` substitutes a file's bytes with a provided `MemoryBuffer` without changing the `FileEntry` identity:

```cpp
auto Buf = llvm::MemoryBuffer::getMemBufferCopy(NewContent, FileName);
SM.overrideFileContents(FileRef, llvm::MemoryBuffer::getMemBuffer(*Buf));
```

The `ContentCache` for the affected file will then return the override buffer instead of reading from disk. The `OrigEntry` field still points to the original file, so diagnostic messages report the correct file name. `isFileOverridden()` allows callers to check whether a specific file has been overridden, and `bypassFileContentsOverride()` creates a bypass entry that reads the disk content even when an override is in effect — used during module hash validation to check whether the original file has changed.

---

## Performance Characteristics

### SLocEntry Lookup

`getFileID(SourceLocation)` is one of the hottest operations in the compiler. The implementation uses a one-entry locality cache (`LastFileIDLookup` + offset bounds) that handles the common case of sequential token queries within a single file in O(1). On a cache miss, it binary-searches `LocalLocOffsetTable`, a sorted `SmallVector<UIntTy>` maintained in parallel with `LocalSLocEntryTable`. This binary search is O(log N) where N is the number of FileIDs — typically a few hundred to a few thousand, not the number of source bytes.

### Line Number Cache

`getLineNumber` is explicitly documented as expensive on first call. The comment in `SourceManager.h` says "this requires building and caching a table of line offsets for the MemoryBuffer, so this is not cheap: use only when about to emit a diagnostic." The `LineOffsetMapping` scan is O(file size); subsequent binary searches are O(log(line count)). The per-call locality cache (`LastLineNoFileIDQuery`, `LastLineNoFilePos`, `LastLineNoResult`) additionally avoids repeated binary searches for closely adjacent offsets in the same file.

### `isBeforeInTranslationUnit` Cost

`isBeforeInTranslationUnit` is O(include depth) in the worst case. For two locations in the same TU that share a common ancestor N levels up, the implementation walks both include/expansion chains to find the nearest common FileID, then compares the offsets of the `#include`/expansion points within that common file. The `IBTUCache` (a `DenseMap<pair<FileID, FileID>, InBeforeInTUCacheEntry>`) caches the result for each (L, R) FileID pair so that repeated comparisons between the same two files are O(1) after the first call.

The `InBeforeInTUCacheEntry` stores `CommonFID`, `LCommonOffset`, `RCommonOffset`, and `LChildBeforeRChild`. The `getCachedResult` method handles the subtle case where two macro expansions share the same expansion location (e.g., two tokens from the same macro invocation at the same call-site offset) by falling back to the FileID ordering: the macro whose FileID was created earlier comes first.

For performance-sensitive client code such as AST traversal or sort comparisons, prefer `isBeforeInSLocAddrSpace` which compares raw offsets without the include-chain traversal:

```cpp
// Fast, but only valid when comparing within the same "local" or "loaded" half
bool fast = SM.isBeforeInSLocAddrSpace(Loc1, Loc2);
```

When both locations are in the local half (`offset < NextLocalOffset`), this is a single integer comparison. Use `isBeforeInTranslationUnit` only when correctness across the full include tree is required.

### Memory Budget

A representative large translation unit with 10,000 SLocEntries (files + macro expansions) consumes:
- `LocalSLocEntryTable`: ~10,000 × ~40 bytes = ~400 KB
- `LocalLocOffsetTable`: ~10,000 × 4 bytes = ~40 KB
- `ContentCache` objects: one per unique file, ~64 bytes each (excluding buffers)
- Line offset tables: loaded lazily, proportional to source file sizes

`SourceManager::getDataStructureSizes()` and `getMemoryBufferSizes()` can be called to get exact figures at runtime. `getMemoryBufferSizes()` distinguishes malloc-backed buffers from mmap-backed ones, the latter being zero-cost in terms of RSS until pages are faulted in. The overall source manager metadata for a typical translation unit is well under 1 MB excluding raw file bytes.

---

## Integration with clangd

clangd uses all three systems described in this chapter in concert. At startup, `FileManager` is constructed with an `OverlayFileSystem` (from `llvm/Support/VirtualFileSystem.h`) that layers an `InMemoryFileSystem` over the real filesystem. When the user modifies a file in their editor, clangd calls `InMemoryFileSystem::addFile` with the new content and bumps the virtual modification time, ensuring that a subsequent `FileEntry::getModificationTime()` comparison detects the change.

The `PreambleData` struct (in `clang/lib/Frontend/PrecompiledPreamble.cpp`) stores the serialized AST of the preamble alongside the set of include files it depended on, tracked via `FileEntryRef` identity. On each edit, clangd calls `PreambleData::reuse` which constructs a new `CompilerInvocation`, attaches the preamble's serialized AST as the PCH, and sets up a `SourceManager` whose `LoadedSLocEntryTable` is populated from the preamble when `ASTReader` processes the PCH. The local SLocEntries (starting at `NextLocalOffset` = `PreambleData::EndOffset + 1`) are then allocated freshly for the post-preamble region. This arrangement makes every token in the preamble addressable by a stable `SourceLocation` across multiple reparses.

clangd's `IncludeStructure` (in `clang-tools-extra/clangd/IncludeStructure.h`) is built by the `CollectMainFileMacros` and `IncludeStructureCollector` `PPCallbacks` implementations. These record `FileChanged`, `InclusionDirective`, `MacroDefined`, and `MacroExpands` callbacks, extracting `FileEntryRef` and `SourceLocation` to build the include graph and macro-definition index. The `FileEntryRef::getName()` vs. `getNameAsRequested()` distinction is relevant here: clangd uses `getName()` (the VFS-canonical path, after redirections) when comparing against workspace paths, but uses the underlying `FileEntry*` for deduplication in `DenseMap` lookups.

clangd's `HeaderSearch` integration uses `FileManager::getOptionalFileRef` with a custom `OverlayFileSystem` to intercept all file system accesses during parsing, recording which headers were opened and what their modification times were. This data populates the `HeaderFileInfo` structures that later determine whether a header's contribution to a preamble has been invalidated. The tight coupling between `FileManager`, `SourceManager`, and the VFS is what makes clangd's incremental parsing both correct and efficient.

---

## Chapter Summary

- `SourceLocation` is a 32-bit integer encoding an offset into a virtual address space. Bit 31 distinguishes file locations (0) from macro expansion locations (1). Zero is the invalid sentinel. `getRawEncoding()`/`getFromRawEncoding()` provide the serialization interface.
- `FileID` is a signed 32-bit index: positive values index `LocalSLocEntryTable`, negative (< -1) values index `LoadedSLocEntryTable`. The value -1 is a sentinel; zero is invalid.
- `SLocEntry` is a discriminated union of `FileInfo` (physical file, include location, content cache pointer, characteristic kind) and `ExpansionInfo` (spelling location, expansion range, macro-arg vs. body flag). Both types are packed to fit in the same union to minimize memory for expansion-heavy TUs.
- `ContentCache` lazily loads the file buffer and lazily computes the line-offset table as a bump-allocated sorted array. `getLineNumber` is O(file size) on first call but O(log lines) afterward; `getPresumedLoc` is the correct API for user-facing output because it respects `#line` directives.
- `LineTableInfo` records `LineEntry` objects created by `AddLineNote` when the preprocessor encounters `#line` or GNU linemarker directives. `getPresumedLoc` consults this table only when `FileInfo::HasLineDirectives` is set, keeping the common case fast.
- `FileEntry` provides inode-level identity (device + inode via `UniqueID`); `FileEntryRef` adds the path name used during lookup; `FileManager` manages both, backed by `llvm::vfs::FileSystem` and a stat cache.
- The three location flavors — spelling (where bytes come from), expansion (where the macro was called), and presumed (what `#line` says) — are exposed through symmetric `SourceManager` API pairs. `getFileLoc` returns the best single physical location for a macro ID.
- `getDecomposedLoc` returns raw `(FileID, offset)` pairs for fast ordering; `isBeforeInSLocAddrSpace` is an O(1) comparison within the same address-space half; `isBeforeInTranslationUnit` handles cross-FileID ordering with a cached include-chain traversal.
- `createTokenSplitLoc` and `ExpansionInfo::createForTokenSplit` model token splitting (e.g., `>>` into two `>`), creating a macro-bit SLocEntry with `ExpansionIsTokenRange = false`.
- The `LocalSLocEntryTable` / `LoadedSLocEntryTable` split assigns local entries offsets starting at 1 (growing upward) and PCH/module entries offsets starting at 2^31 (growing downward), allowing modules to extend the address space without disturbing local assignments. `noteSLocAddressSpaceUsage` diagnoses exhaustion.
- `overrideFileContents` and `OverlayFileSystem` are the two mechanisms clangd uses to inject in-memory edits: the former substitutes bytes within an existing `ContentCache`, the latter intercepts `FileManager` lookups before files are opened.
- `translateFileLineCol`/`translateLineCol` convert `file:line:col` triples back to `SourceLocation` values; `getMacroArgExpandedLocation` maps a call-site spelling location into its macro-body expansion location, using a lazy per-FileID `MacroArgsMap` cache.

---

*Cross-references:*
- [Chapter 28 — The Clang Driver](ch28-clang-driver.md) — how the driver constructs `CompilerInstance` and initializes `FileManager` and `SourceManager`
- [Chapter 30 — The Diagnostic Engine](ch30-diagnostic-engine.md) — how `DiagnosticsEngine` consumes `SourceLocation` to emit caret diagnostics and "In file included from" notes
- [Chapter 31 — The Lexer and Preprocessor](ch31-lexer-preprocessor.md) — how `Preprocessor` calls `createExpansionLoc` and `createMacroArgExpansionLoc` as it expands macros
- [Chapter 36 — The Clang AST in Depth](ch36-clang-ast.md) — how `Stmt`, `Decl`, and `Expr` carry `SourceRange` and how the AST printer uses `SourceManager` to display source text
- [Chapter 38 — Code Completion and clangd Foundations](ch38-clangd-foundations.md) — how clangd's `TUScheduler` and `PreambleData` interact with `FileManager` and `SourceManager`

*Reference links:*
- [`clang/include/clang/Basic/SourceLocation.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/SourceLocation.h)
- [`clang/include/clang/Basic/SourceManager.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/SourceManager.h)
- [`clang/include/clang/Basic/FileEntry.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/FileEntry.h)
- [`clang/include/clang/Basic/FileManager.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/FileManager.h)
- [`clang/lib/Basic/SourceManager.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Basic/SourceManager.cpp)
- [`clang/lib/Basic/FileManager.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Basic/FileManager.cpp)
- [`llvm/include/llvm/Support/VirtualFileSystem.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/VirtualFileSystem.h)
