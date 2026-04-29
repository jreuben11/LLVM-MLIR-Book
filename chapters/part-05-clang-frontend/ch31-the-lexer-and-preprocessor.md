# Chapter 31 — The Lexer and Preprocessor

*Part V — Clang Internals: Frontend Pipeline*

The lexer and preprocessor form the first active phase of Clang's compilation pipeline: they transform a raw byte stream into a sequence of preprocessed tokens that the parser can consume. Unlike the conceptual separation taught in compiler courses, Clang's architecture tightly couples lexing and preprocessing — the `Preprocessor` owns the `Lexer`, drives it token by token, and interleaves macro expansion, directive handling, and header inclusion into a unified token pump. Understanding this layer is prerequisite to writing Clang plugins, PPCallbacks observers, pragma handlers, or any tool that must reason about source text before AST construction.

This chapter covers every major component: the `Lexer` class and its buffer cursor model, the `Token` struct and the `tok::TokenKind` enumeration, identifier interning via `IdentifierInfo` and `IdentifierTable`, the full range of literal lexing, the `Preprocessor` class as a token dispatcher, macro expansion mechanics, directive processing, conditional compilation, pragma handling, module-level interactions, `PPCallbacks` for non-invasive observation, and the `-E` preprocessing pipeline.

---

## 31.1 Lexer Architecture

### 31.1.1 The `clang::Lexer` Class

`clang::Lexer` is defined in [`clang/include/clang/Lex/Lexer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/Lexer.h) and inherits from `PreprocessorLexer`. It implements a hand-written, table-driven lexer over a contiguous memory buffer. The class is designed for maximal throughput: the common case for every character category is resolved in a few instructions without heap allocation.

The three central pointer fields are:

```cpp
const char *BufferStart; // First byte of the source file buffer
const char *BufferPtr;   // Current scan position (next character to read)
const char *BufferEnd;   // One past the last byte (always '\0' sentinel)
```

`BufferStart` and `BufferEnd` are set at construction and never change during lexing. `BufferPtr` advances as tokens are recognized. The buffer is always null-terminated so that single-character look-ahead can read `*BufferPtr` without a bounds check.

Two construction paths exist. The first creates a standalone lexer for raw lexing (no preprocessor) from a buffer range:

```cpp
Lexer(SourceLocation FileLoc, const LangOptions &LangOpts,
      const char *BufStart, const char *BufPtr, const char *BufEnd,
      bool IsFirstIncludeOfMainFile = false);
```

The second is the normal path used during compilation: it takes a `FileID` and `SourceManager` reference so that every token's `SourceLocation` can be computed by offset arithmetic against the buffer:

```cpp
Lexer(FileID FID, const llvm::MemoryBufferRef &InputBuffer,
      const SourceManager &SM, const LangOptions &LangOpts,
      bool IsFirstIncludeOfMainFile = false);
```

### 31.1.2 Raw Lexer Mode vs Preprocessor-Driven Mode

`LexingRawMode` is a boolean flag on the lexer that, when set, makes `Lex()` return tokens without keyword recognition, identifier resolution, or any interaction with a `Preprocessor`. All identifiers are returned as `tok::raw_identifier` rather than `tok::identifier`, and no directive processing occurs. This mode is used by tools that need to scan source text at high speed — for example, the dependency scanner (`DependencyDirectivesScanner`), clangd's background indexer, and `Lexer::Create_PragmaLexer()` which creates a temporary lexer for pragma argument parsing.

In preprocessor-driven mode (the default), `Lexer::Lex(Token &Result)` sets `Result`'s fields and returns `true` if the caller should stop (end of file). The `Preprocessor::Lex()` method wraps this and loops internally until it has a fully-preprocessed token.

### 31.1.3 `LangOptions` Governing Lexer Behavior

`LangOptions` is passed by const reference to every `Lexer` and controls which syntactic constructs are recognized. Relevant flags (defined in [`clang/include/clang/Basic/LangOptions.def`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/LangOptions.def)):

| Flag | Effect on lexer |
|------|----------------|
| `CPlusPlus` | Enables `//` comments, C++ keywords, `0b` binary literals |
| `CPlusPlus11`–`CPlusPlus26` | Enables `nullptr`, `constexpr`, `char8_t`, modules keywords |
| `C99` | Enables `//` comments in C mode, `_Bool`, `_Complex` |
| `C23` | Enables `true`/`false` as keywords, `_BitInt`, `typeof` |
| `MicrosoftExt` | Accepts `__int64`, `__cdecl`, `__declspec`, `#pragma comment` |
| `Trigraphs` | Translates `??=`, `??(`, etc. in phase 1 |
| `Digraphs` | Allows `<:`, `:>`, `<%`, `%>`, `%:`, `%:%:` |
| `HexFloats` | Accepts `0x1.8p+1` style hex-float constants |
| `CPlusPlusModules` | Enables `module`, `import`, `export` as context-sensitive keywords |

The `LangOptions` struct is populated by `CompilerInvocation::CreateFromArgs()` based on driver flags; the lexer reads it read-only throughout compilation.

### 31.1.4 `Lexer::Lex(Token&)` — The Main Entry Point

```cpp
bool Lexer::Lex(Token &Result);
```

On entry, `BufferPtr` points to the next unscanned character. `Lex()` performs the following:

1. Skip whitespace (spaces, tabs, form-feeds), setting `Token::StartOfLine` and `Token::LeadingSpace` flags.
2. Mark the token's start location via `getSourceLocation(BufferPtr)`.
3. Dispatch on `*BufferPtr` through a large `switch` statement that covers every ASCII value. Each case fast-paths common characters and falls back to a slow helper for multi-character sequences.
4. For identifiers: scan alphanumeric/underscore bytes, look up the spelling in `IdentifierTable`, and classify as identifier or keyword.
5. For `#` at the start of a line: if not in raw mode, return `tok::hash` and let the `Preprocessor` handle the directive.
6. Return `false` to indicate more tokens remain, or `true` at `tok::eof`.

The inner `getCharAndSize()` / `ConsumeChar()` pair handles phase 1 and phase 2 translation: trigraph expansion (if `LangOpts.Trigraphs`) and `\<newline>` line splicing, both implemented in `getCharAndSizeSlow()`.

### 31.1.5 `getSpelling()` and the Scratch Buffer

After a token is produced, consumers sometimes need the logical spelling — the characters as they appear after all phase 1/2 transformations have been applied. `Lexer` provides several static utilities:

```cpp
// Return the spelling as a std::string (may allocate for cleaned tokens)
static std::string Lexer::getSpelling(const Token &Tok,
                                       const SourceManager &SM,
                                       const LangOptions &LangOpts,
                                       bool *Invalid = nullptr);

// Copy spelling into buffer; returns length
static unsigned Lexer::getSpelling(const Token &Tok,
                                    const char *&Buffer,
                                    const SourceManager &SM,
                                    const LangOptions &LangOpts,
                                    bool *Invalid = nullptr);
```

If `Token::NeedsCleaning` is not set, `getSpelling()` simply returns a `StringRef` into the source buffer at `[Loc, Loc+Length)`, with zero allocation. When `NeedsCleaning` is set, the function re-runs the character-reading logic to produce the cleaned form. The `ScratchBuffer` allocator (owned by the `Preprocessor`) provides memory for synthetic tokens (stringified macro arguments, token-paste results) that have no corresponding source range.

### 31.1.6 Static Utilities: Source-Level Queries

`Lexer` also provides a collection of static source-level utilities useful to AST consumers and refactoring tools:

```cpp
// Compute source location at end of a token
static SourceLocation Lexer::getLocForEndOfToken(
    SourceLocation Loc, unsigned Offset,
    const SourceManager &SM, const LangOptions &LangOpts);

// Advance past `Characters` characters from a token start
static SourceLocation Lexer::getLocForStartOfToken(
    SourceLocation Loc, const SourceManager &SM,
    const LangOptions &LangOpts);

// Return the source range for a token
static CharSourceRange Lexer::getAsCharRange(
    SourceRange Range,
    const SourceManager &SM, const LangOptions &LangOpts);

// Find the token at a given source location (re-lexes the buffer)
static tok::TokenKind Lexer::getRawToken(
    SourceLocation Loc, Token &Result,
    const SourceManager &SM, const LangOptions &LangOpts,
    bool IgnoreWhiteSpace = false);
```

These static methods create temporary raw lexers over the relevant buffer region to answer position queries without maintaining persistent lexer state. They are extensively used by clang-tidy fix-it generation and source rewriting via `Rewriter`.

---

## 31.2 The `Token` Struct

### 31.2.1 Fields

`clang::Token` (in [`clang/include/clang/Lex/Token.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/Token.h)) is a 16-byte struct that carries every piece of information the parser needs about a single token:

```cpp
class Token {
  SourceLocation::UIntTy Loc;      // Encoded SourceLocation of token start
  SourceLocation::UIntTy UintData; // Length (identifier/literal) or annot end
  void *PtrData;                   // IdentifierInfo*, literal data, or annot ptr
  tok::TokenKind Kind;             // The token kind (9-bit enum)
  unsigned short Flags;            // TokenFlags bitmask
};
```

`Loc` is a raw-encoded `SourceLocation` (see [Chapter 29](../part-05-clang-frontend/ch29-sourcemanager-fileentry-sourcelocation.md)). `UintData` stores the byte-length of the token's spelling for identifier and literal tokens; for annotation tokens it stores the raw encoding of the annotation end location. `PtrData` is a type-punned union used as:

- `IdentifierInfo *` for `tok::identifier` and keywords
- `const char *` or `void *` for literal tokens (pointing into the buffer or scratch space)
- `void *` for annotation tokens storing parsed data

### 31.2.2 `tok::TokenKind` Enumeration

`tok::TokenKind` is generated from [`clang/include/clang/Basic/TokenKinds.def`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/TokenKinds.def) and contains over 300 values across several categories:

**Structural:**
- `tok::eof` — end of file sentinel
- `tok::unknown` — unrecognized character
- `tok::comment` — (only emitted in special modes)

**Identifiers and literals:**
- `tok::identifier` — any non-keyword identifier
- `tok::raw_identifier` — identifier in raw mode (before keyword mapping)
- `tok::numeric_constant` — `0x123`, `3.14f`, `0b1010`, `1_000`
- `tok::char_constant`, `tok::wide_char_constant`, `tok::utf8_char_constant`, `tok::utf16_char_constant`, `tok::utf32_char_constant`
- `tok::string_literal`, `tok::wide_string_literal`, `tok::utf8_string_literal`, `tok::utf16_string_literal`, `tok::utf32_string_literal`

**Punctuators** (generated by `PUNCTUATOR` macro):
- `tok::amp` (`&`), `tok::ampamp` (`&&`), `tok::ampequal` (`&=`)
- `tok::l_paren`, `tok::r_paren`, `tok::l_brace`, `tok::r_brace`
- `tok::arrow` (`->`), `tok::coloncolon` (`::`), `tok::ellipsis` (`...`)

**Keywords** (generated by `KEYWORD`, `CXX11_KEYWORD`, etc.):
- `tok::kw_if`, `tok::kw_for`, `tok::kw_while`, `tok::kw_return`, `tok::kw_class`
- `tok::kw___attribute__`, `tok::kw___declspec`, `tok::kw___asm`

**Annotations** (synthetic tokens injected by the parser):
- `tok::annot_typename`, `tok::annot_cxxscope`, `tok::annot_template_id`, `tok::annot_pragma_openmp`

### 31.2.3 `TokenFlags` Bitmask

```cpp
enum TokenFlags {
  StartOfLine        = 0x001, // Token is at start of logical line
  LeadingSpace       = 0x002, // Whitespace precedes this token
  DisableExpand      = 0x004, // Identifier: macro expansion disabled here
  NeedsCleaning      = 0x008, // Spelling requires trigraph/UCN/splicing cleanup
  LeadingEmptyMacro  = 0x010, // An empty macro expands before this token
  HasUDSuffix        = 0x020, // String/char literal has a UDL suffix
  HasUCN             = 0x040, // Identifier contains a universal character name
  IsReinjected       = 0x800, // Token was previously produced and re-injected
};
```

`DisableExpand` is set during macro argument pre-expansion and token-paste operations to prevent recursive expansion. `NeedsCleaning` tells downstream consumers that `Lexer::getSpelling()` must re-scan the raw buffer rather than using the stored pointer.

### 31.2.4 Querying Tokens

```cpp
bool Token::is(tok::TokenKind K) const;
bool Token::isNot(tok::TokenKind K) const;
template <typename... Ts> bool Token::isOneOf(Ts... Ks) const;
bool Token::isAnyIdentifier() const;      // tok::identifier or tok::raw_identifier
bool Token::isLiteral() const;            // numeric_constant through utf32_string_literal
bool Token::isAtStartOfLine() const;
bool Token::hasLeadingSpace() const;
bool Token::hasUDSuffix() const;
IdentifierInfo *Token::getIdentifierInfo() const; // asserts isAnyIdentifier()
SourceLocation Token::getLocation() const;
unsigned Token::getLength() const;
```

The parser and Sema use `is()` / `isOneOf()` extensively in preference to direct `Kind` comparisons because it reads more clearly and compiles to equivalent code.

---

## 31.3 `IdentifierInfo` and `IdentifierTable`

### 31.3.1 The String Intern Table

`IdentifierTable` (in [`clang/include/clang/Basic/IdentifierTable.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/IdentifierTable.h)) is a `llvm::StringMap<IdentifierInfo>` that ensures each distinct identifier string has exactly one `IdentifierInfo` object. Comparison of two identifiers reduces to a pointer comparison. The table is owned by `Preprocessor` and lives for the entire translation unit.

```cpp
IdentifierInfo &IdentifierTable::get(StringRef Name);
IdentifierInfo &IdentifierTable::get(StringRef Name, tok::TokenKind TokenCode);
```

The second overload registers a new keyword: it calls `get(Name)` and then stamps `II.TokenID = TokenCode`. All C and C++ keywords are registered during `Preprocessor` construction by iterating `TokenKinds.def`.

### 31.3.2 `IdentifierInfo` Fields

```cpp
class IdentifierInfo {
  unsigned TokenID : 9;              // tok::TokenKind (tok::identifier, or kw_*)
  unsigned InterestingIdentifierID;  // ObjC selector or builtin ID
  unsigned ChangedAfterLoad : 1;     // Macro redefinition since last deserialization
  unsigned FEChangedAfterLoad : 1;   // FETokenInfo changed since deserialization
  unsigned RevertedTokenID : 1;      // revertTokenIDToIdentifier() was called
  unsigned OutOfDate : 1;            // Needs update from external source
  unsigned IsModulesImport : 1;      // Is an import keyword in current context
  void *FETokenInfo;                 // Frontend-owned per-identifier payload
  llvm::StringMapEntry<IdentifierInfo *> *Entry; // back-pointer for getName()
};
```

The most important field is `TokenID`. When the lexer produces an identifier token, it looks up the spelling in `IdentifierTable` and reads `II->getTokenID()`. If the result is not `tok::identifier`, the lexer replaces `tok::identifier` with the keyword kind. This is how keywords are recognized: there is no trie or special table — just a hash-map lookup with a flag check.

`FETokenInfo` is an untyped pointer for exclusive use by Sema and other frontend consumers. Sema casts it to `NamedDecl *` or `Scope *` during name resolution (see [Chapter 33](../part-05-clang-frontend/ch33-sema-i-names-lookups-conversions.md)).

### 31.3.3 Keyword Recognition Sequence

When `Lexer::Lex()` scans an identifier, the path is:

1. Scan contiguous identifier bytes into a `StringRef` backed by the source buffer.
2. Call `PP->getIdentifierInfo(Spelling)` which calls `IdentifierTable::get(Spelling)`.
3. Store the resulting `IdentifierInfo *` in `Token::PtrData`.
4. Call `II->getTokenID()` — if not `tok::identifier`, update `Token::Kind`.
5. Check `II->isExtensionToken()` or test whether the keyword is enabled in the current `LangOptions` (e.g., `tok::kw_nullptr` requires `CPlusPlus11`).

Language extensions that define new keywords (e.g., OpenMP, HLSL, Objective-C) follow the same pattern: they register their identifiers with the appropriate `tok::TokenKind` via `IdentifierTable::get(name, kw_X)` at startup.

### 31.3.4 `revertTokenIDToIdentifier()`

Sometimes a keyword must be treated as a plain identifier — notably, GNU libstdc++ 4.2 uses `__is_empty` as a non-keyword. Clang handles this via:

```cpp
void IdentifierInfo::revertTokenIDToIdentifier();
void IdentifierInfo::revertIdentifierToTokenID(tok::TokenKind TK);
```

The `RevertedTokenID` flag records that the reversal happened so deserialization from a PCH can preserve the state.

---

## 31.4 Character Encoding and Universal Characters

### 31.4.1 UTF-8 Source Files

Clang assumes source files are UTF-8 encoded. Non-ASCII bytes are permitted in string and character literals, in comments, and (since C++23/C23) in identifiers. The `Lexer` does not transcode: it passes UTF-8 bytes through to string literal content unmodified. The `SourceManager` treats all source as a byte array; `SourceLocation` offsets are byte offsets, not Unicode code-point offsets.

### 31.4.2 Universal Character Names

UCNs (`\uXXXX` and `\UXXXXXXXX`) allow non-ASCII characters in identifiers and string literals at the source level regardless of file encoding. The lexer handles them in `getCharAndSizeSlow()` and the UCN-specific helpers:

```cpp
std::optional<uint32_t> Lexer::tryReadNumericUCN(
    const char *&StartPtr, const char *SlashLoc, Token *Result);
std::optional<uint32_t> Lexer::tryReadNamedUCN(
    const char *&StartPtr, const char *SlashLoc, Token *Result);
```

`tryReadNumericUCN()` parses `\uXXXX` / `\UXXXXXXXX` and validates that the code point is not a surrogate (0xD800–0xDFFF) and not in the basic source character set. When a UCN is found in an identifier, the `HasUCN` flag is set on the token; `Lexer::getSpelling()` is then aware it must perform UCN-to-UTF-8 conversion to produce the "clean" spelling.

### 31.4.3 Named UCNs (C23/C++23)

C23 and C++23 add support for named universal characters such as `\N{LATIN SMALL LETTER A}`. `tryReadNamedUCN()` handles these by consulting a compile-time lookup table of Unicode character names.

### 31.4.4 Line Splicing and Trigraphs

`getCharAndSizeSlow()` implements C translation phase 2: a backslash followed immediately by a newline is spliced, making the two physical lines appear as one logical line. The `NeedsCleaning` flag on a token tells consumers that the raw buffer bytes are not the true spelling.

Trigraph sequences (`??=` → `#`, `??(` → `[`, `??)` → `]`, etc.) are translated only when `LangOpts.Trigraphs` is true, which requires explicit `-ftrigraphs`. Clang 22 defaults to no trigraph support. The static helper `Lexer::getCharAndSizeNoWarn()` performs the same translation without emitting diagnostics, used when the lexer needs to peek ahead silently.

---

## 31.5 Literal Lexing

### 31.5.1 Integer Literals: `NumericLiteralParser`

The lexer classifies any token starting with a digit (or `0b`/`0x` prefix) as `tok::numeric_constant`. The raw spelling — e.g., `0xDEAD'BEEFull` — is stored as a `StringRef` into the source buffer. The semantic content is extracted later by `NumericLiteralParser` (defined in [`clang/include/clang/Lex/LiteralSupport.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/LiteralSupport.h)):

```cpp
NumericLiteralParser NLP(TokSpelling, TokLoc,
                         PP.getSourceManager(), PP.getLangOpts(),
                         PP.getTargetInfo(), PP.getDiagnostics());
if (NLP.isIntegerLiteral()) {
    llvm::APInt ResultVal(64, 0);
    NLP.GetIntegerValue(ResultVal);
    bool IsUnsigned = NLP.isUnsigned;
    bool IsLong = NLP.isLong;
    bool IsLongLong = NLP.isLongLong;
}
```

Supported integer forms:
- Decimal: `42`, `1'000'000` (C++14 digit separators)
- Hexadecimal: `0xFF`, `0XFF`
- Octal: `0755`
- Binary: `0b1010`, `0B1010` (C++14 / C23)
- `_BitInt`/`wb`/`WB` suffix (C23): `42wb`, `42WB`
- Unsigned suffix: `u`, `U`; long suffix: `l`, `L`; long-long: `ll`, `LL`
- C++23 `z`/`Z` suffix for `std::size_t` type

`NumericLiteralParser` also recognizes invalid suffixes and emits diagnostics via the `DiagnosticsEngine` ([Chapter 30](../part-05-clang-frontend/ch30-the-diagnostic-engine.md)).

### 31.5.2 Floating-Point Literals

Float literals are also tokenized as `tok::numeric_constant`. `NumericLiteralParser` fields `isFloat`, `isDouble`, `isLongDouble`, `isFloat16`, `isBitInt` distinguish them from integer literals. Hex floats (`0x1.8p+1`) require `LangOpts.HexFloats` (automatically set for C99/C++17 and later).

Suffixes: `f`/`F` (float), `l`/`L` (long double), `f16`/`F16` (IEEE 754 half), `bf16`/`BF16` (bfloat16). Clang 22 accepts these via `__attribute__((mode(BF)))` or native `_Float16` / `__bf16` types.

### 31.5.3 String Literals: `StringLiteralParser`

String tokens arrive as one or more consecutive `tok::string_literal` (or wide/UTF-8/16/32 variant) tokens. Adjacent literals are concatenated by the parser, not the lexer. `StringLiteralParser` handles the full transformation from raw token spellings to a null-terminated, decoded character sequence:

```cpp
StringLiteralParser SLP(StringToks, PP);
if (!SLP.hadError) {
    StringRef Result = SLP.GetString(); // decoded bytes
    unsigned CharByteWidth = SLP.getCharByteWidth();
}
```

Escape sequences decoded: `\n`, `\t`, `\r`, `\\`, `\"`, `\'`, `\0`, `\xHH` (hex), `\OOO` (octal), `\uXXXX`, `\UXXXXXXXX`, `\N{name}`.

Raw string literals (C++11): `R"tag(...)tag"`. The tag can be any sequence of up to 16 characters excluding parentheses, backslash, and whitespace. The lexer identifies the delimiter tag and searches for the matching closing sequence. The content between the parentheses is taken verbatim — no escape processing.

**Encoding prefixes and their token kinds:**

| Source | Token kind |
|--------|-----------|
| `"..."` | `tok::string_literal` |
| `L"..."` | `tok::wide_string_literal` |
| `u8"..."` | `tok::utf8_string_literal` |
| `u"..."` | `tok::utf16_string_literal` |
| `U"..."` | `tok::utf32_string_literal` |

### 31.5.4 Character Literals: `CharLiteralParser`

`CharLiteralParser` handles `'x'`, `L'x'`, `u8'x'`, `u'x'`, `U'x'`. Multi-character literals (e.g., `'AB'`) are accepted with a warning. The parser computes the integer value and stores it in an `unsigned long long`.

### 31.5.5 User-Defined Literals

A UDL is a literal immediately followed by an identifier suffix with no intervening whitespace: `42_km`, `3.14_v`, `"hello"_s`. The lexer detects this when scanning a literal token and, if the next character is an identifier start (`_` or letter), extends the scan to include the suffix. The `HasUDSuffix` flag is set. The raw suffix spelling is stored alongside the literal and `Token::getUDSuffix()` returns it as a `StringRef`. The parser routes UDL tokens to Sema for `operator""` overload resolution.

---

## 31.6 The `Preprocessor` Class

### 31.6.1 Overview

`clang::Preprocessor` (in [`clang/include/clang/Lex/Preprocessor.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/Preprocessor.h)) is the central coordinator of the preprocessor layer. It owns:

- The current `Lexer` (`CurLexer`), held as `std::unique_ptr<Lexer>`
- The include/macro stack (`IncludeMacroStack`), a `std::vector<IncludeStackInfo>`
- The macro definition map (`Macros`), a `DenseMap<const IdentifierInfo *, MacroState>` within the current `SubmoduleState`
- The `HeaderSearch` instance for `#include` resolution
- The `IdentifierTable`
- The `PPCallbacks` chain

### 31.6.2 The Include/Macro Stack

`IncludeMacroStack` is the call stack of the preprocessor. Each entry (`IncludeStackInfo`) stores the saved `CurLexer`, `CurTokenLexer`, the current directory iterator, and the submodule state, so that returning from an `#include` or a macro expansion restores the correct lexer context.

```cpp
struct IncludeStackInfo {
  LexerCallback    CurLexerCallback;  // which Lex() variant to call
  Module          *TheSubmodule;
  std::unique_ptr<Lexer> TheLexer;
  PreprocessorLexer     *ThePPLexer;
  std::unique_ptr<TokenLexer> TheTokenLexer;
  ConstSearchDirIterator TheDirLookup;
};
```

When the preprocessor `EnterSourceFile()` or `EnterMacro()` is called, the current state is pushed and a new context is installed. `PopIncludeMacroStack()` restores it.

### 31.6.3 `Preprocessor::Lex(Token&)` — The Token Pump

```cpp
void Preprocessor::Lex(Token &Result);
```

This is the primary interface consumed by the `Parser`. It calls `CurLexer->Lex(Result)` (or `CurTokenLexer->Lex(Result)` for token streams from macro expansion) and then:

1. If the result is a `tok::hash` at the start of a line, calls `HandleDirective(Result)` to process a preprocessor directive.
2. If the result is `tok::identifier` and the identifier has a macro definition and `DisableExpand` is not set, calls into the macro expansion engine.
3. If the lexer returns `tok::eof`, pops the include stack to resume the parent file (or returns `tok::eof` if at the top level).
4. Invokes `PPCallbacks` at appropriate points.

The loop terminates when a fully preprocessed, non-comment, non-directive token has been produced.

### 31.6.4 `EnterSourceFile()`, `EnterMacro()`, `EnterTokenStream()`

```cpp
bool Preprocessor::EnterSourceFile(
    FileID FID, ConstSearchDirIterator Dir, SourceLocation Loc,
    bool IsFirstIncludeOfMainFile = false);

void Preprocessor::EnterMacro(Token &Tok, SourceLocation ILEnd,
                               MacroInfo *Macro, MacroArgs *Args);

void Preprocessor::EnterTokenStream(const Token *Toks, unsigned NumToks,
                                    bool DisableMacroExpansion,
                                    bool OwnsTokens,
                                    bool IsReinject);
```

`EnterSourceFile()` pushes the current state, creates a new `Lexer` for `FID`, and sets it as `CurLexer`. `EnterMacro()` creates a `TokenLexer` (which plays back the macro's replacement token list) and installs it as `CurTokenLexer`. `EnterTokenStream()` is used to inject arbitrary token arrays — for example, the result of `_Pragma("...")` is converted to a token stream and re-lexed via this mechanism.

### 31.6.5 `LexUnexpandedToken()` and `LexNonComment()`

Two specialized variants of `Lex()` are available:

```cpp
void Preprocessor::LexUnexpandedToken(Token &Result);
void Preprocessor::LexNonComment(Token &Result);
```

`LexUnexpandedToken()` temporarily disables macro expansion by setting `DisableExpand` on the returned identifier token and bypassing the expansion check, but still processes directives. It is used when the preprocessor needs to peek at a macro's argument list without triggering expansion — the most important consumer is `ReadFunctionLikeMacroArgs()` itself. `LexNonComment()` loops internally over `tok::comment` tokens, useful in modes where comments are preserved (e.g., `-C` flag).

The `PreprocessingRecord` (in `clang/include/clang/Lex/PreprocessingRecord.h`) is an optional `PPCallbacks` subclass that builds a complete event log of all macro expansions and inclusion directives for the TU, consumed by libclang's `CXTranslationUnit` query API.

---

## 31.7 Macro Expansion

### 31.7.1 `MacroInfo` and `MacroDirective`

`MacroInfo` (in [`clang/include/clang/Lex/MacroInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/MacroInfo.h)) stores a macro's definition:

```cpp
class MacroInfo {
  SourceLocation Location;              // Point of #define
  SourceLocation EndLocation;           // End of replacement list
  Token *ReplacementTokens;             // Replacement token array
  unsigned NumTokens;                   // Length of replacement list
  IdentifierInfo **ParameterList;       // Formal parameter names
  unsigned NumParameters;               // Arity
  bool IsFunctionLike : 1;
  bool IsVariadic : 1;                  // Uses ...
  bool IsBuiltinMacro : 1;
  bool HasCommaPasting : 1;
};
```

`MacroDirective` is a linked list node representing the history of a single identifier's macro bindings across `#define` / `#undef` operations in the translation unit. Each `MacroDirective` points to either a `DefMacroDirective` (which holds a `MacroInfo *`) or an `UndefMacroDirective`.

### 31.7.2 Object-Like Macro Expansion

When `Preprocessor::Lex()` sees that the current identifier has a macro definition and `DisableExpand` is not set, it calls the expansion path. For an object-like macro:

1. The replacement token list is copied from `MacroInfo::ReplacementTokens`.
2. Each token's source location is remapped to the macro expansion's `SLocEntry` in the `SourceManager` — creating the `MacroExpansionLoc` chain that allows diagnostics to print both the expansion site and the definition site.
3. The tokens are pushed via `EnterTokenStream()` with `DisableMacroExpansion=false` so nested macros still expand.

### 31.7.3 Function-Like Macro Expansion

Function-like macros require argument collection before expansion. The preprocessor calls `ReadFunctionLikeMacroArgs()` which:

1. Scans past `(`, collecting comma-separated argument token sequences. Nested `(...)` are balanced.
2. Stores each argument as a range in a `MacroArgs` object: one "unexpanded" copy (raw tokens) and one "pre-expanded" copy (result of running `Lex()` on the argument tokens with a fresh `TokenLexer`).

Argument pre-expansion is deferred: `MacroArgs::getPreExpArgument()` runs the inner `Preprocessor::Lex()` loop on the argument's token range when the expanded form is first needed.

### 31.7.4 Token Pasting (`##`)

Token pasting is implemented in the `TokenLexer::PasteTokens()` method. When `TokenLexer::Lex()` encounters `tok::hashhash` in the replacement list:

1. The left-hand side token (already produced) and the right-hand side token (next in the list) are obtained.
2. Their spellings are concatenated into a `ScratchBuffer`.
3. A temporary `Lexer` re-lexes the concatenated string.
4. If lexing produces exactly one token and consumes all characters, that token is the pasted result; otherwise a diagnostic is emitted.
5. The pasted token has `NeedsCleaning` cleared (it is already clean).

### 31.7.5 Stringification (`#`)

`#` before a macro parameter is handled by `MacroArgs::StringifyArgument()`:

```cpp
static Token MacroArgs::StringifyArgument(
    const Token *ArgToks, Preprocessor &PP,
    bool Charify, SourceLocation ExpansionLocStart,
    SourceLocation ExpansionLocEnd);
```

The function iterates the unexpanded argument token list, builds a string with proper escaping (`\"` for quotes, `\\` for backslashes), wraps it in double quotes, and returns a single `tok::string_literal` token allocated in the `ScratchBuffer`.

### 31.7.6 `__VA_ARGS__` and `__VA_OPT__`

Variadic macros are declared with `...` as the last parameter. `__VA_ARGS__` in the replacement list expands to all actual arguments beyond the last named parameter, joined by commas. `__VA_OPT__(tokens)` (C++20 / C23) expands to `tokens` if `__VA_ARGS__` is non-empty and to nothing otherwise. Clang implements `__VA_OPT__` in `TokenLexer` by checking whether the variadic argument is empty before emitting the optional token sequence.

### 31.7.7 Built-In Macros

Several identifiers are handled as built-in macros that cannot be `#undef`ed:

| Macro | Behavior |
|-------|---------|
| `__FILE__` | Current source file path as string literal |
| `__LINE__` | Current logical line number as integer literal |
| `__DATE__`, `__TIME__` | Compilation date/time strings |
| `__COUNTER__` | Monotonically increasing integer per TU |
| `__has_feature(X)` | 1 if compiler feature X is present |
| `__has_extension(X)` | 1 if X is available as an extension |
| `__has_include(<H>)` | 1 if H can be found by the include search |
| `__has_include_next(<H>)` | 1 if H is findable after the current directory |
| `__has_attribute(X)` | 1 if `__attribute__((X))` is recognized |
| `__has_cpp_attribute(X)` | 1 if `[[X]]` C++ attribute is recognized |
| `__has_builtin(X)` | 1 if `__builtin_X` is a known builtin |

These are recognized by `Preprocessor::ExpandBuiltinMacro()` which does not use `MacroInfo` but directly manufactures the result token.

---

## 31.8 Directive Processing

### 31.8.1 `Preprocessor::HandleDirective()`

When `Preprocessor::Lex()` receives `tok::hash` with `Token::isAtStartOfLine()` true, it calls `HandleDirective(Token &Result)`. This function:

1. Reads the next token to identify the directive keyword.
2. Dispatches to the appropriate handler based on the `tok::PPKeyword` kind of that token (`tok::pp_define`, `tok::pp_include`, `tok::pp_if`, etc.).

```cpp
void Preprocessor::HandleDirective(Token &Result) {
    // Skips if we're in an excluded conditional block
    // (except for #if/#ifdef/#ifndef/#elif/#else/#endif)
    Token Tok;
    Lex(Tok); // read the directive keyword
    switch (Tok.getIdentifierInfo()->getPPKeywordID()) {
    case tok::pp_define:   return HandleDefineDirective(Tok, false);
    case tok::pp_include:  return HandleIncludeDirective(Result.getLocation(), Tok);
    case tok::pp_if:       return HandleIfDirective(Tok, Result, false);
    case tok::pp_ifdef:    return HandleIfdefDirective(Tok, Result, false, true);
    // ...
    }
}
```

### 31.8.2 `#include` and `HeaderSearch`

`HandleIncludeDirective()` parses the header name token (angle-bracket or quoted form), then delegates to `HeaderSearch::LookupFile()`:

```cpp
OptionalFileEntryRef HeaderSearch::LookupFile(
    StringRef Filename, SourceLocation IncludeLoc,
    bool isAngled,
    ConstSearchDirIterator FromDir,
    ConstSearchDirIterator *CurDir,
    ArrayRef<std::pair<const FileEntry *, const DirectoryEntry *>> Includers,
    SmallVectorImpl<char> *SearchPath,
    SmallVectorImpl<char> *RelativePath,
    ModuleMap::KnownHeader *SuggestedModule,
    bool *IsMapped, bool *IsFrameworkFound,
    bool SkipCache = false);
```

`isAngled` is `true` for `<header>` and `false` for `"header"`. The search strategy:

1. For `"header"`: search relative to the including file's directory first.
2. Apply the `-I` (normal search), `-isystem` (system search), and `-iwithprefix` paths in order.
3. For `<header>`: skip the local directory, start directly with the `-I` list.
4. Framework search (macOS): check `.framework/Headers/` paths.

`HeaderMap` is a binary-format hash map (`.hmap` files) that maps filename strings to actual paths, used by Xcode for framework headers.

`#include_next` skips all directories up to and including the one from which the current file was included, then continues the search from the next directory. This mechanism supports the "wrapper header" pattern (e.g., a portability header that includes the system header with the same name).

### 31.8.3 `HandleDefineDirective()`

Parsing a `#define` tokenizes the replacement list by calling `Lex()` until `tok::eod` (end-of-directive). If a `(` immediately follows the macro name without whitespace, the macro is function-like and parameters are collected before the replacement list. The resulting `MacroInfo` is allocated from a bump allocator owned by the `Preprocessor`, stored in the `MacroMap`, and a `DefMacroDirective` is prepended to the identifier's directive chain.

Re-definition of an already-defined macro with a different body emits a `warn_pp_macro_redef` diagnostic unless the body is token-for-token identical (the C standard permits identical re-definitions without a preceding `#undef`).

### 31.8.4 `#undef`, `#line`, and `#error`

`HandleUndefDirective()` looks up the identifier in `MacroMap` and, if a definition exists, appends an `UndefMacroDirective` to the directive chain. The `MacroInfo` itself is not freed immediately; it remains allocated for the lifetime of the `Preprocessor` so that `MacroDirective` history traversal remains valid.

`HandleLineDirective()` parses `#line N "file"` and calls `SourceManager::AddLineNote()` to record a synthetic line/file remapping. This is how compilers that generate C code (yacc/bison, Cython) annotate the output so diagnostics reference the original source.

`HandleErrorDirective()` and `HandleWarningDirective()` emit `err_pp_error_directive` and `warn_pp_warning_directive` diagnostics with the remainder of the directive line as the message string.

### 31.8.5 `#embed` (C23)

C23 adds `#embed "file"` which splices the binary content of a file as a comma-separated integer sequence directly into the translation unit. Clang implements this as a special lexer path in `HandleEmbedDirective()` which reads the target file via `SourceManager::getFileManager()`, checks `PPEmbedParameters` (limit, prefix, suffix, if-empty clauses), and injects the byte sequence as a sequence of `tok::numeric_constant` tokens via `EnterTokenStream()`. The `tok::annot_embed` annotation token is used internally to represent the embedded data before it is expanded by the token stream.

---

## 31.9 Conditional Compilation

### 31.9.1 The Conditional Stack

The preprocessor maintains a per-lexer stack of `PPConditionalInfo` structures:

```cpp
// In PreprocessorLexer.h:
struct PPConditionalInfo {
    SourceLocation IfLoc;       // Location of the opening #if
    bool WasSkipping;           // Were we skipping before this level?
    bool FoundNonSkip;          // Have we seen a non-skipped branch at this level?
    bool FoundElse;             // Have we seen #else at this level?
};
SmallVector<PPConditionalInfo, 4> ConditionalStack;
```

`WasSkipping` records whether the enclosing scope was already in a skip state, so that when the inner `#endif` is reached, the outer skip state can be restored. `FoundNonSkip` becomes true when any branch at this level has been taken; subsequent `#elif`/`#else` branches are all skipped. `FoundElse` prevents a second `#else` from being accepted.

### 31.9.2 `SkipExcludedConditionalBlock()`

When the condition of an `#if`/`#ifdef`/`#elif` evaluates to false (or is in a skip context), `SkipExcludedConditionalBlock()` is called to fast-forward to the matching `#endif`, `#else`, or `#elif`. It does this by calling `CurLexer->LexDependencyDirectiveTokenWhileSkipping()` — a stripped-down lexer pass that recognizes only `#if*`/`#endif` nesting for bookkeeping. This avoids the overhead of full tokenization during excluded regions and is a significant performance optimization for large conditionally-compiled sections.

The `PPConditionalInfo::WasSkipping` / `FoundNonSkipPortion` / `FoundElse` fields stored in `IncludeMacroStack` allow re-entry when a relevant directive (`#elif`, `#elifdef`, `#elifndef`, `#else`, `#endif`) is encountered inside the fast-forward.

### 31.9.3 `#if` Expression Evaluation

`#if` and `#elif` expressions are evaluated by `Preprocessor::EvaluateDirectiveExpression()`. The expression is parsed by a recursive-descent constant-expression evaluator that:

- Handles integer arithmetic, bitwise/logical operators, and the ternary operator.
- Recognizes `defined(X)` and `!defined(X)` as special pseudo-functions (the `defined` identifier is not a macro).
- Evaluates `__has_include(...)`, `__has_cpp_attribute(...)`, `__has_attribute(...)`.
- All identifiers in a `#if` expression that are not defined as macros expand to `0` (per the C standard).

C++23 adds `#elifdef`/`#elifndef` as shorthand for `#elif defined(...)` / `#elif !defined(...)`. Clang implements these in `HandleElsedirective()` with the same `IfdefDirective` logic but under the `#elif` umbrella.

---

## 31.10 Pragma Handling

### 31.10.1 `PragmaHandler`

`PragmaHandler` (in [`clang/include/clang/Lex/Pragma.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/Pragma.h)) is the base class for pragma implementations:

```cpp
class PragmaHandler {
  StringRef Name; // The pragma name (e.g., "once", "GCC", "clang")
public:
  explicit PragmaHandler(StringRef Name);
  virtual ~PragmaHandler();
  StringRef getName() const { return Name; }
  virtual void HandlePragma(Preprocessor &PP,
                             PragmaIntroducer Introducer,
                             Token &FirstToken) = 0;
};
```

`PragmaIntroducer` records whether the pragma arrived via `#pragma` or `_Pragma(...)`. Custom handlers are registered:

```cpp
PP.addPragmaHandler("my_namespace", new MyPragmaHandler("directive"));
// Or at top-level (no namespace):
PP.addPragmaHandler(new MyTopLevelPragmaHandler("directive"));
```

Removal:
```cpp
PP.RemovePragmaHandler("my_namespace", Handler);
```

Namespace-qualified pragmas (`#pragma my_namespace directive`) are handled by a `PragmaNamespace` dispatcher that holds a child handler map.

### 31.10.2 Built-In Pragma Handlers

Clang registers all standard pragma handlers in `Preprocessor::RegisterBuiltinPragmas()`:

| Pragma | Handler class | Effect |
|--------|--------------|--------|
| `#pragma once` | `PragmaOnceHandler` | Mark file for single-inclusion |
| `#pragma GCC system_header` | `PragmaGCCSystemHeader` | Treat current file as system header |
| `#pragma GCC diagnostic push/pop/error/warning/ignored` | `PragmaDiagnosticHandler` | Modify diagnostic state |
| `#pragma clang diagnostic push/pop/...` | (same handler) | Clang alias for GCC diagnostic |
| `#pragma clang attribute push/pop` | `PragmaAttributeHandler` | Apply attribute to all subsequent decls |
| `#pragma push_macro` / `pop_macro` | `PragmaPushMacroHandler` | Save/restore macro definition |
| `#pragma comment(lib, "...")` | `PragmaCommentHandler` | Generate `ModuleRef` or linker flags |
| `#pragma STDC FP_CONTRACT` | `PragmaFPContractHandler` | Control floating-point contraction |
| `#pragma ms_struct on/off` | `PragmaMSStructHandler` | Microsoft struct layout compatibility |

OpenMP pragmas (`#pragma omp parallel`, etc.) are handled by a dedicated `PragmaOpenMPHandler` that produces `tok::annot_pragma_openmp` annotation tokens consumed by the parser.

### 31.10.3 `_Pragma` Operator

`_Pragma("string")` is C99's operator-form of `#pragma`. When the preprocessor encounters the `_Pragma` identifier, it evaluates the string operand, removes the outer quotes, unescapes `\\` and `\"`, and re-lexes the result as though it were a `#pragma` line by calling `EnterTokenStream()` with the re-lexed tokens.

### 31.10.4 Writing a Custom Pragma Handler Plugin

A Clang plugin can register a custom pragma handler via `FrontendPluginRegistry` or directly on the `Preprocessor` from within `ASTFrontendAction::ExecuteAction()`:

```cpp
class MyPragmaHandler : public PragmaHandler {
public:
  MyPragmaHandler() : PragmaHandler("mypragma") {}
  void HandlePragma(Preprocessor &PP, PragmaIntroducer Introducer,
                    Token &PragmaTok) override {
    Token Tok;
    PP.Lex(Tok); // consume the first pragma argument token
    if (Tok.is(tok::identifier)) {
      // process Tok.getIdentifierInfo()->getName()
    }
    // consume until tok::eod
    while (!Tok.is(tok::eod))
      PP.Lex(Tok);
  }
};

// In your FrontendAction::CreateASTConsumer override:
PP.addPragmaHandler(new MyPragmaHandler());
```

---

## 31.11 Precompiled Headers and Modules at the PP Level

### 31.11.1 `ExternalPreprocessorSource`

`ExternalPreprocessorSource` (in [`clang/include/clang/Lex/ExternalPreprocessorSource.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/ExternalPreprocessorSource.h)) is a pure interface for loading preprocessor state lazily:

```cpp
class ExternalPreprocessorSource {
public:
  virtual void ReadDefinedMacros() = 0;
  virtual void updateOutOfDateIdentifier(IdentifierInfo &II) = 0;
  virtual void ReadIdentifiers() {}
};
```

The PCH reader (`ASTReader`) implements this interface. When the preprocessor encounters an identifier with `II.isOutOfDate() == true`, it calls `ExternalPreprocessorSource::updateOutOfDateIdentifier()` which triggers deserializing the macro history and identifier metadata from the PCH file. This lazy-loading strategy means only identifiers actually used in the TU are deserialized.

### 31.11.2 Precompiled Headers

When compiling with `-include-pch foo.pch`, `ASTReader::SetGloballyVisibleDecls()` pre-populates the `IdentifierTable` with stubs marked `OutOfDate = true`. The PCH also stores the state of the `HeaderSearch` so that previously-included files are not re-included. The `MultipleIncludeOpt` mechanism in each `Lexer` detects header guards (`#ifndef FOO_H` / `#define FOO_H` / `#endif` patterns) and records them so that subsequent `#include` of a guarded header can be short-circuited at the `HeaderSearch` level without re-lexing.

### 31.11.3 C++ Named Modules

Under C++ named modules (enabled by `LangOpts.CPlusPlusModules`), `import` and `module` are context-sensitive keywords: they are keywords only at the start of a logical line (i.e., `Token::isAtStartOfLine()` is true). The lexer does not recognize them as keywords unconditionally; instead, `Preprocessor::Lex()` has special handling via `ModuleImportState` that switches `tok::identifier` to `tok::kw_import` or `tok::kw_module` when the context is appropriate.

`ModuleLoader` (in [`clang/include/clang/Lex/ModuleLoader.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/ModuleLoader.h)) is the interface through which the preprocessor requests that a module be loaded:

```cpp
class ModuleLoader {
public:
  virtual ModuleLoadResult loadModule(
      SourceLocation ImportLoc, ModuleIdPath Path,
      Module::NameVisibilityKind Visibility,
      bool IsInclusionDirective) = 0;
  virtual void makeModuleVisible(Module *Mod,
                                 Module::NameVisibilityKind Visibility,
                                 SourceLocation ImportLoc) = 0;
};
```

`CompilerInstance` provides the implementation that calls into the module manager and `ASTReader`. When an import is recognized, the preprocessor calls `EnterSubmodule()` / `LeaveSubmodule()` which swap the `CurSubmoduleState` (the `MacroMap` active for the current module scope).

### 31.11.4 `#pragma clang module import`

`#pragma clang module import Module.SubModule` is the `#pragma` spelling of a module import. The `PragmaModuleImportHandler` resolves the dotted name into a `Module *` and calls `Preprocessor::makeModuleVisible()`.

---

## 31.12 `PPCallbacks` — Observing the Preprocessor

### 31.12.1 The `PPCallbacks` Interface

`PPCallbacks` (in [`clang/include/clang/Lex/PPCallbacks.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/PPCallbacks.h)) defines a pure-virtual observer interface fired by the `Preprocessor` at key events. All methods have empty default implementations, so consumers override only what they need:

```cpp
class PPCallbacks {
public:
  // File inclusion events
  virtual void FileChanged(SourceLocation Loc,
                           FileChangeReason Reason,
                           SrcMgr::CharacteristicKind FileType,
                           FileID PrevFID = FileID{});
  virtual void InclusionDirective(SourceLocation HashLoc,
                                  const Token &IncludeTok,
                                  StringRef FileName,
                                  bool IsAngled,
                                  CharSourceRange FilenameRange,
                                  OptionalFileEntryRef File,
                                  StringRef SearchPath,
                                  StringRef RelativePath,
                                  const Module *SuggestedModule,
                                  bool ModuleImported,
                                  SrcMgr::CharacteristicKind FileType);
  // Macro events
  virtual void MacroDefined(const Token &MacroNameTok,
                             const MacroDirective *MD);
  virtual void MacroUndefined(const Token &MacroNameTok,
                              const MacroDefinition &MD,
                              const MacroDirective *Undef);
  virtual void MacroExpands(const Token &MacroNameTok,
                            const MacroDefinition &MD,
                            SourceRange Range,
                            const MacroArgs *Args);
  // Conditional compilation events
  virtual void If(SourceLocation Loc, SourceRange ConditionRange,
                  ConditionValueKind ConditionValue);
  virtual void Elif(SourceLocation Loc, SourceRange ConditionRange,
                   ConditionValueKind ConditionValue, SourceLocation IfLoc);
  virtual void Ifdef(SourceLocation Loc, const Token &MacroNameTok,
                    const MacroDefinition &MD);
  virtual void Ifndef(SourceLocation Loc, const Token &MacroNameTok,
                     const MacroDefinition &MD);
  virtual void Else(SourceLocation Loc, SourceLocation IfLoc);
  virtual void Endif(SourceLocation Loc, SourceLocation IfLoc);
  virtual void Defined(const Token &MacroNameTok,
                       const MacroDefinition &MD, SourceRange Range);
};
```

### 31.12.2 Registration

```cpp
PP.addPPCallbacks(std::make_unique<MyCallbacks>());
```

Multiple callbacks can be registered; each call to `addPPCallbacks()` wraps the new callback and the existing one in a `PPChainedCallbacks` object that forwards all virtual calls to both. Order of callbacks within the chain is last-registered, first-called.

### 31.12.3 Applications

**clang-tidy** uses `PPCallbacks` to observe macro expansions, `#include` directives, and conditional branches for checks such as `modernize-macro-to-enum` and `cppcoreguidelines-macro-usage`.

**include-what-you-use** registers an `InclusionDirective` callback to record which headers are included and correlates them against which declarations are actually used (obtained from AST traversal).

**clangd** uses `PPCallbacks` to build the include graph and macro cross-reference index needed for hover information and go-to-definition on macro names.

**Dependency scanning** (`DependencyScanningWorker`) uses `PPCallbacks` exclusively — it runs the preprocessor in a stripped-down mode where `InclusionDirective` and `MacroDefined` callbacks populate the dependency graph without constructing an AST.

A minimal tracing example:

```cpp
class IncludeTracer : public PPCallbacks {
  SourceManager &SM;
public:
  explicit IncludeTracer(SourceManager &SM) : SM(SM) {}

  void InclusionDirective(SourceLocation HashLoc,
                          const Token &IncludeTok,
                          StringRef FileName, bool IsAngled,
                          CharSourceRange, OptionalFileEntryRef File,
                          StringRef SearchPath, StringRef,
                          const Module *, bool,
                          SrcMgr::CharacteristicKind) override {
    llvm::outs() << SM.getFilename(HashLoc)
                 << ":" << SM.getSpellingLineNumber(HashLoc)
                 << " includes " << FileName << "\n";
  }
};

// Installed in ExecuteAction():
CI.getPreprocessor().addPPCallbacks(
    std::make_unique<IncludeTracer>(CI.getSourceManager()));
```

---

## 31.13 The `clang -E` Pipeline

### 31.13.1 `PrintPreprocessedOutput`

When Clang is invoked with `-E`, the frontend action is `PrintPPOutputAction`, which calls `PrintPreprocessedOutput()` from [`clang/lib/Frontend/PrintPreprocessedOutput.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Frontend/PrintPreprocessedOutput.cpp). This function registers a `PrintPPOutputPPCallbacks` — a subclass of `PPCallbacks` — and then drives the preprocessor token pump in a loop:

```cpp
void clang::DoPrintPreprocessedInput(Preprocessor &PP,
                                     raw_ostream *OS,
                                     const PreprocessorOutputOptions &Opts) {
  // Install the callback
  PP.addPPCallbacks(std::make_unique<PrintPPOutputPPCallbacks>(PP, *OS, ...));
  // Run the token pump
  Token Tok;
  PP.EnterMainSourceFile();
  do {
    PP.Lex(Tok);
    // The callback writes tokens to OS as each is consumed
  } while (Tok.isNot(tok::eof));
}
```

The `PrintPPOutputPPCallbacks` class implements `FileChanged()` to emit linemarker lines (`# N "file" flags`) and `MacroDefined()` / `MacroUndefined()` to optionally emit macro definitions.

### 31.13.2 Linemarker Format

Linemarkers have the form:
```
# linenum "filename" flags
```
where `flags` is a space-separated list of:
- `1` — entering a new file
- `2` — returning to a file
- `3` — this is a system header
- `4` — extern "C" should be implicit

The `-P` flag suppresses all linemarker output. Linemarkers are generated both by the file-change callbacks and by `#line` directive processing.

### 31.13.3 Macro Dump Flags

- `-dD`: emit `#define` lines for all macros alongside the preprocessed output.
- `-dM`: emit only `#define` lines for all macros at the end; suppress all other output. Implemented by running the full preprocessor but printing only macro definitions in `FileChanged()` at the `ExitFile` transition for the main file.
- `-dI`: emit `#include` directives as encountered (in addition to the included content).
- `-dU`: emit `#undef` for all macros that are undefined during preprocessing.

These modes are controlled by `PreprocessorOutputOptions` fields (`ShowMacros`, `ShowMacroComments`, `ShowIncludeDirectives`, `ShowLineMarkers`).

### 31.13.4 Interaction with `TokenWriter`

The token serialization path (for PTH generation, now largely deprecated) used a `TokenWriter` that stored the token stream in a binary format. Modern Clang uses PCH (AST-based) and named modules exclusively; PTH is retained only for compatibility.

---

## Summary

- `Lexer` operates over a `[BufferStart, BufferEnd)` byte range with `BufferPtr` as the cursor; `LangOptions` selects which syntactic forms are recognized.
- `Token` is a 16-byte struct carrying `Kind`, `Loc`, `UintData` (length), `PtrData` (IdentifierInfo or literal pointer), and a `Flags` bitmask; `tok::TokenKind` enumerates 300+ values.
- `IdentifierTable` interns all identifier strings; `IdentifierInfo::TokenID` maps identifiers to keyword kinds at hash-map lookup time, avoiding any separate keyword trie.
- The lexer handles UTF-8 source, UCNs (`\uXXXX` / `\UXXXXXXXX` / `\N{name}`), trigraphs, and line splicing in `getCharAndSizeSlow()`.
- `NumericLiteralParser` and `StringLiteralParser` decode literal tokens after lexing; UDL suffixes are detected by the `HasUDSuffix` token flag.
- `Preprocessor` owns the lexer stack (`IncludeMacroStack`) and the macro map; `Preprocessor::Lex()` is the main token pump that interleaves directive handling and macro expansion.
- Macro expansion uses `MacroInfo` for the definition, `MacroArgs` for argument collection, `TokenLexer` for playback, `PasteTokens()` for `##`, and `StringifyArgument()` for `#`.
- `HandleDirective()` dispatches `#include` → `HeaderSearch`, `#define` → `MacroInfo` construction, `#if*` → the conditional stack with `SkipExcludedConditionalBlock()` fast-path.
- `PragmaHandler::HandlePragma()` is the extension point for pragma-based DSLs; clang-diagnostic, clang-attribute, OpenMP, and custom plugin pragmas all use this mechanism.
- `PPCallbacks` provides a zero-overhead observer hook for tools (clang-tidy, include-what-you-use, clangd) without modifying the preprocessor core.
- `clang -E` drives a `PrintPPOutputPPCallbacks` that writes linemarkers, preprocessed text, and optionally macro definitions; `-P`, `-dM`, and `-dD` modify the output.

---

## Cross-References

- [Chapter 29 — SourceManager, FileEntry, and SourceLocation](../part-05-clang-frontend/ch29-sourcemanager-fileentry-sourcelocation.md) — `SourceLocation` encoding and `SourceManager` API
- [Chapter 30 — The Diagnostic Engine](../part-05-clang-frontend/ch30-the-diagnostic-engine.md) — `DiagnosticsEngine` used by the lexer and `NumericLiteralParser`
- [Chapter 32 — The Parser](../part-05-clang-frontend/ch32-the-parser.md) — consumes the token stream produced by `Preprocessor::Lex()`
- [Chapter 36 — The Clang AST in Depth](../part-05-clang-frontend/ch36-the-clang-ast-in-depth.md) — `IntegerLiteral`, `StringLiteral`, and `CharacterLiteral` AST nodes constructed from parsed literal tokens
- [Chapter 37 — C++ Modules Implementation](../part-05-clang-frontend/ch37-cpp-modules-implementation.md) — `ModuleLoader`, `SubmoduleState`, `EnterSubmodule()`

## Reference Links

- [`clang/include/clang/Lex/Lexer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/Lexer.h)
- [`clang/include/clang/Lex/Token.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/Token.h)
- [`clang/include/clang/Basic/TokenKinds.def`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/TokenKinds.def)
- [`clang/include/clang/Basic/IdentifierTable.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/IdentifierTable.h)
- [`clang/include/clang/Basic/LangOptions.def`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/LangOptions.def)
- [`clang/include/clang/Lex/Preprocessor.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/Preprocessor.h)
- [`clang/include/clang/Lex/MacroInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/MacroInfo.h)
- [`clang/include/clang/Lex/MacroArgs.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/MacroArgs.h)
- [`clang/include/clang/Lex/LiteralSupport.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/LiteralSupport.h)
- [`clang/include/clang/Lex/HeaderSearch.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/HeaderSearch.h)
- [`clang/include/clang/Lex/PPCallbacks.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/PPCallbacks.h)
- [`clang/include/clang/Lex/Pragma.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/Pragma.h)
- [`clang/include/clang/Lex/ModuleLoader.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/ModuleLoader.h)
- [`clang/include/clang/Lex/ExternalPreprocessorSource.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/ExternalPreprocessorSource.h)
- [`clang/lib/Frontend/PrintPreprocessedOutput.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Frontend/PrintPreprocessedOutput.cpp)
- [`clang/lib/Lex/Lexer.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Lex/Lexer.cpp)
- [`clang/lib/Lex/Preprocessor.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Lex/Preprocessor.cpp)
- [`clang/lib/Lex/MacroExpansion.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Lex/MacroExpansion.cpp)
- [`clang/lib/Lex/TokenLexer.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Lex/TokenLexer.cpp)
- [`clang/lib/Lex/PPDirectives.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Lex/PPDirectives.cpp)
- [`clang/lib/Lex/PPConditionalDirectiveRecord.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Lex/PPConditionalDirectiveRecord.cpp)


---

@copyright jreuben11
