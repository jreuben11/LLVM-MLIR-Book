# Chapter 55 — Building a Frontend

*Part IX — Frontend Authoring (Building Your Own)*

Every language that compiles through LLVM needs a frontend: a pipeline that transforms source text into LLVM IR. Clang is one such frontend, but nothing about LLVM requires it — Rust, Swift, Julia, Zig, Flang, and dozens of domain-specific languages all share the same LLVM backend via independently authored frontends. This chapter builds the skeleton of a new language frontend from scratch, covering lexical analysis, parsing, AST design, and the driver glue that connects them. The running example is a small expression language called *Cal*, demonstrating every step with compilable code.

## 55.1 Frontend Architecture

A minimal LLVM frontend has four components:

```
Source text
    │
    ▼  Lexer (tokenizer)
Token stream
    │
    ▼  Parser
AST (Abstract Syntax Tree)
    │
    ▼  Semantic analysis / type checking
Typed AST
    │
    ▼  IR emission (Chapter 56)
llvm::Module (LLVM IR)
```

Each component is independent and testable in isolation. The canonical reference is the Kaleidoscope tutorial distributed with LLVM — a small floating-point expression language with function definitions, binary operators, and `if`/`then`/`else`. Cal extends Kaleidoscope with integer types and explicit type annotations, providing a more realistic starting point.

The Kaleidoscope tutorial source is maintained in
[`llvm/examples/Kaleidoscope/`](https://github.com/llvm/llvm-project/tree/llvmorg-22.1.0/llvm/examples/Kaleidoscope/)
across chapters 1–9.

## 55.2 The Lexer

### 55.2.1 Design Principles

A hand-written lexer is the standard choice for production compilers. It operates as a generator: each call to `Lexer::lex()` consumes characters from a buffer and returns the next token. The design invariants are:

- **No backtracking**: consume each character exactly once.
- **Position tracking**: maintain file, line, column for every token.
- **Keyword disambiguation**: identifiers that match keywords return a keyword token, not `TOK_IDENT`.
- **Number literals**: parse both integer and floating-point forms in the lexer, not the parser.

### 55.2.2 Token Representation

```cpp
enum class TokenKind {
  // Literals
  Ident,    // identifier
  IntLit,   // 42
  FpLit,    // 3.14
  StrLit,   // "hello"

  // Keywords
  KwFn, KwLet, KwIf, KwElse, KwReturn,
  KwInt, KwFloat, KwBool, KwVoid,

  // Punctuation
  LParen, RParen, LBrace, RBrace, LBracket, RBracket,
  Comma, Semicolon, Colon, Arrow,

  // Operators
  Plus, Minus, Star, Slash, Percent,
  Eq, EqEq, Bang, BangEq,
  Lt, LtEq, Gt, GtEq,
  Amp, AmpAmp, Pipe, PipePipe,

  // Misc
  Eof, Error,
};

struct Token {
  TokenKind  kind;
  std::string_view text;  // slice into source buffer
  unsigned   line;
  unsigned   col;

  // Semantic values (set by lexer, not parser)
  std::variant<std::monostate, int64_t, double, std::string> value;
};
```

Using `std::string_view` slices into the source buffer avoids allocations for most tokens. Only string literals (which require escape processing) allocate into `value`.

### 55.2.3 Implementation

```cpp
class Lexer {
  const char *cur;    // current position in buffer
  const char *end;    // one-past-end of buffer
  unsigned line = 1;
  unsigned col  = 1;

  char peek()  const { return cur < end ? *cur : '\0'; }
  char advance() {
    char c = *cur++;
    if (c == '\n') { ++line; col = 1; } else ++col;
    return c;
  }

public:
  Lexer(std::string_view src) : cur(src.data()), end(src.data() + src.size()) {}

  Token lex() {
    // Skip whitespace and comments
    while (cur < end) {
      if (std::isspace(peek())) { advance(); continue; }
      if (peek() == '/' && cur+1 < end && cur[1] == '/') {
        while (cur < end && peek() != '\n') advance();
        continue;
      }
      break;
    }
    if (cur >= end) return makeToken(TokenKind::Eof);

    unsigned startLine = line, startCol = col;
    const char *start = cur;
    char c = advance();

    // Identifiers and keywords
    if (std::isalpha(c) || c == '_') {
      while (std::isalnum(peek()) || peek() == '_') advance();
      std::string_view text(start, cur - start);
      return makeIdent(text, startLine, startCol);
    }

    // Integer and float literals
    if (std::isdigit(c)) {
      while (std::isdigit(peek())) advance();
      if (peek() == '.' && std::isdigit(cur[1])) {
        advance(); // consume '.'
        while (std::isdigit(peek())) advance();
        double val = std::stod(std::string(start, cur));
        Token tok; tok.kind = TokenKind::FpLit;
        tok.text = {start, (size_t)(cur-start)};
        tok.line = startLine; tok.col = startCol;
        tok.value = val;
        return tok;
      }
      int64_t val = std::stoll(std::string(start, cur));
      Token tok; tok.kind = TokenKind::IntLit;
      tok.text = {start, (size_t)(cur-start)};
      tok.line = startLine; tok.col = startCol;
      tok.value = val;
      return tok;
    }

    // Single-character and two-character operators
    switch (c) {
      case '+': return makeTok(TokenKind::Plus, start, startLine, startCol);
      case '-':
        if (peek() == '>') { advance(); return makeTok(TokenKind::Arrow, start, startLine, startCol); }
        return makeTok(TokenKind::Minus, start, startLine, startCol);
      case '*': return makeTok(TokenKind::Star, start, startLine, startCol);
      case '/': return makeTok(TokenKind::Slash, start, startLine, startCol);
      case '=':
        if (peek() == '=') { advance(); return makeTok(TokenKind::EqEq, start, startLine, startCol); }
        return makeTok(TokenKind::Eq, start, startLine, startCol);
      case '!':
        if (peek() == '=') { advance(); return makeTok(TokenKind::BangEq, start, startLine, startCol); }
        return makeTok(TokenKind::Bang, start, startLine, startCol);
      case '<':
        if (peek() == '=') { advance(); return makeTok(TokenKind::LtEq, start, startLine, startCol); }
        return makeTok(TokenKind::Lt, start, startLine, startCol);
      case '>':
        if (peek() == '=') { advance(); return makeTok(TokenKind::GtEq, start, startLine, startCol); }
        return makeTok(TokenKind::Gt, start, startLine, startCol);
      case '(': return makeTok(TokenKind::LParen, start, startLine, startCol);
      case ')': return makeTok(TokenKind::RParen, start, startLine, startCol);
      case '{': return makeTok(TokenKind::LBrace, start, startLine, startCol);
      case '}': return makeTok(TokenKind::RBrace, start, startLine, startCol);
      case ',': return makeTok(TokenKind::Comma, start, startLine, startCol);
      case ';': return makeTok(TokenKind::Semicolon, start, startLine, startCol);
      case ':': return makeTok(TokenKind::Colon, start, startLine, startCol);
      default:  return makeError(start, startLine, startCol);
    }
  }
};
```

Keyword disambiguation can be a lookup table or a sorted array:

```cpp
Token Lexer::makeIdent(std::string_view text, unsigned line, unsigned col) {
  static const std::pair<std::string_view, TokenKind> keywords[] = {
    {"fn",     TokenKind::KwFn},
    {"let",    TokenKind::KwLet},
    {"if",     TokenKind::KwIf},
    {"else",   TokenKind::KwElse},
    {"return", TokenKind::KwReturn},
    {"int",    TokenKind::KwInt},
    {"float",  TokenKind::KwFloat},
    {"bool",   TokenKind::KwBool},
    {"void",   TokenKind::KwVoid},
  };
  for (auto &[kw, kind] : keywords)
    if (text == kw) return makeTok(kind, text.data(), line, col);
  Token tok; tok.kind = TokenKind::Ident; tok.text = text;
  tok.line = line; tok.col = col;
  return tok;
}
```

For performance, an `llvm::StringMap<TokenKind>` or a perfect hash (generated by `gperf` or `llvm::StringSwitch`) replaces the linear scan in production:

```cpp
TokenKind Lexer::classifyIdent(std::string_view text) {
  return llvm::StringSwitch<TokenKind>(text)
    .Case("fn",     TokenKind::KwFn)
    .Case("let",    TokenKind::KwLet)
    .Case("if",     TokenKind::KwIf)
    .Case("else",   TokenKind::KwElse)
    .Case("return", TokenKind::KwReturn)
    .Default(TokenKind::Ident);
}
```

`llvm::StringSwitch` is defined in
[`llvm/include/llvm/ADT/StringSwitch.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ADT/StringSwitch.h) and generates efficient dispatch.

## 55.3 AST Design

### 55.3.1 Node Hierarchy

The AST represents the parsed program structure before semantic analysis. Nodes are heap-allocated (typically arena-allocated in production) and linked by raw or owning pointers:

```cpp
// Base node — carries source location
struct AstNode {
  unsigned line, col;
  virtual ~AstNode() = default;
};

// Types
struct TypeNode : AstNode {};
struct NamedTypeNode : TypeNode { std::string name; };
struct PtrTypeNode   : TypeNode { std::unique_ptr<TypeNode> pointee; };

// Expressions
struct ExprNode : AstNode {};
struct IntLitExpr   : ExprNode { int64_t value; };
struct FpLitExpr    : ExprNode { double  value; };
struct BoolLitExpr  : ExprNode { bool    value; };
struct IdentExpr    : ExprNode { std::string name; };

struct BinopExpr : ExprNode {
  TokenKind op;
  std::unique_ptr<ExprNode> lhs, rhs;
};

struct UnaryExpr : ExprNode {
  TokenKind op;
  std::unique_ptr<ExprNode> operand;
};

struct CallExpr : ExprNode {
  std::string callee;
  std::vector<std::unique_ptr<ExprNode>> args;
};

struct IfExpr : ExprNode {
  std::unique_ptr<ExprNode>  cond;
  std::unique_ptr<StmtNode>  thenBlock, elseBlock; // optional else
};

// Statements
struct StmtNode : AstNode {};

struct ExprStmt : StmtNode { std::unique_ptr<ExprNode> expr; };

struct LetStmt : StmtNode {
  std::string name;
  std::unique_ptr<TypeNode> type;  // optional
  std::unique_ptr<ExprNode> init;
};

struct ReturnStmt : StmtNode {
  std::unique_ptr<ExprNode> value; // nullable for void
};

struct BlockStmt : StmtNode {
  std::vector<std::unique_ptr<StmtNode>> stmts;
};

struct IfStmt : StmtNode {
  std::unique_ptr<ExprNode> cond;
  std::unique_ptr<BlockStmt> then_, else_; // else_ nullable
};

struct WhileStmt : StmtNode {
  std::unique_ptr<ExprNode> cond;
  std::unique_ptr<BlockStmt> body;
};

// Top-level declarations
struct FnDecl : AstNode {
  std::string name;
  std::vector<std::pair<std::string, std::unique_ptr<TypeNode>>> params;
  std::unique_ptr<TypeNode> retType;
  std::unique_ptr<BlockStmt> body;   // null for extern declarations
};

struct Module {
  std::vector<std::unique_ptr<FnDecl>> decls;
};
```

### 55.3.2 Arena Allocation

For production frontends, heap-allocating each node with `std::unique_ptr` is too slow. The standard alternative is an arena (bump allocator) that allocates all nodes from a large block:

```cpp
class AstArena {
  std::vector<std::vector<char>> blocks;
  char *cur, *end;
  static constexpr size_t blockSize = 1 << 20; // 1 MiB

  void newBlock() {
    blocks.emplace_back(blockSize);
    cur = blocks.back().data();
    end = cur + blockSize;
  }

public:
  AstArena() { newBlock(); }

  template<typename T, typename... Args>
  T* make(Args&&... args) {
    size_t align = alignof(T);
    size_t space = sizeof(T) + align - 1;
    char *p = (char*)(((uintptr_t)cur + align - 1) & ~(align - 1));
    if (p + sizeof(T) > end) { newBlock(); return make<T>(std::forward<Args>(args)...); }
    cur = p + sizeof(T);
    return new(p) T(std::forward<Args>(args)...);
  }
};
```

With an arena, AST pointers become raw pointers (`T*`) instead of `unique_ptr`. The entire arena is freed at the end of compilation, amortizing allocation and deallocation to a single `free` per block.

LLVM's `BumpPtrAllocator` (in
[`llvm/include/llvm/Support/Allocator.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/Allocator.h)) provides a production-quality bump allocator; Clang's `ASTContext` uses it for all AST nodes.

## 55.4 The Parser

### 55.4.1 Recursive Descent

Recursive descent is the standard parser style for hand-written parsers. Each grammar rule becomes a function; the parser maintains a one-token lookahead:

```cpp
class Parser {
  Lexer &lexer;
  Token cur;

  void advance() { cur = lexer.lex(); }
  bool check(TokenKind k) const { return cur.kind == k; }
  bool eat(TokenKind k) {
    if (check(k)) { advance(); return true; }
    return false;
  }
  Token expect(TokenKind k) {
    if (!check(k)) error("expected ", tokenKindName(k), " got ", cur.text);
    Token t = cur; advance(); return t;
  }

public:
  Parser(Lexer &lexer) : lexer(lexer) { advance(); }
  std::unique_ptr<Module> parseModule();
};
```

### 55.4.2 Parsing Declarations

```cpp
std::unique_ptr<FnDecl> Parser::parseFnDecl() {
  expect(TokenKind::KwFn);
  auto name = expect(TokenKind::Ident).text;
  expect(TokenKind::LParen);

  std::vector<std::pair<std::string, std::unique_ptr<TypeNode>>> params;
  while (!check(TokenKind::RParen)) {
    auto paramName = expect(TokenKind::Ident).text;
    expect(TokenKind::Colon);
    auto paramType = parseType();
    params.emplace_back(std::string(paramName), std::move(paramType));
    if (!eat(TokenKind::Comma)) break;
  }
  expect(TokenKind::RParen);

  std::unique_ptr<TypeNode> retType;
  if (eat(TokenKind::Arrow))
    retType = parseType();

  std::unique_ptr<BlockStmt> body;
  if (check(TokenKind::LBrace))
    body = parseBlock();
  else
    expect(TokenKind::Semicolon); // extern declaration

  auto decl = std::make_unique<FnDecl>();
  decl->name    = std::string(name);
  decl->params  = std::move(params);
  decl->retType = std::move(retType);
  decl->body    = std::move(body);
  return decl;
}
```

### 55.4.3 Pratt Parsing for Expressions

Recursive descent handles statements cleanly, but expressions with multi-level precedence require care. The **Pratt parser** (also called top-down operator precedence, TDOP) is the canonical solution — it handles left-recursion and precedence elegantly without grammar transforms:

```cpp
int precedence(TokenKind k) {
  switch (k) {
    case TokenKind::PipePipe: return 10;
    case TokenKind::AmpAmp:   return 20;
    case TokenKind::EqEq:
    case TokenKind::BangEq:   return 30;
    case TokenKind::Lt: case TokenKind::LtEq:
    case TokenKind::Gt: case TokenKind::GtEq: return 40;
    case TokenKind::Plus:
    case TokenKind::Minus:    return 50;
    case TokenKind::Star:
    case TokenKind::Slash:
    case TokenKind::Percent:  return 60;
    default:                  return -1;
  }
}

std::unique_ptr<ExprNode> Parser::parseExpr(int minPrec) {
  auto lhs = parseUnary();

  while (true) {
    int prec = precedence(cur.kind);
    if (prec < minPrec) break;

    TokenKind op = cur.kind;
    advance();
    auto rhs = parseExpr(prec + 1); // left-associative: prec + 1
    // prec for right-assoc (e.g. '='): parseExpr(prec)

    auto binop = std::make_unique<BinopExpr>();
    binop->op  = op;
    binop->lhs = std::move(lhs);
    binop->rhs = std::move(rhs);
    lhs = std::move(binop);
  }
  return lhs;
}

std::unique_ptr<ExprNode> Parser::parseUnary() {
  if (check(TokenKind::Minus) || check(TokenKind::Bang)) {
    TokenKind op = cur.kind; advance();
    auto operand = parseUnary();
    auto unary = std::make_unique<UnaryExpr>();
    unary->op = op; unary->operand = std::move(operand);
    return unary;
  }
  return parsePostfix();
}

std::unique_ptr<ExprNode> Parser::parsePrimary() {
  if (check(TokenKind::IntLit)) {
    auto e = std::make_unique<IntLitExpr>();
    e->value = std::get<int64_t>(cur.value);
    advance(); return e;
  }
  if (check(TokenKind::FpLit)) {
    auto e = std::make_unique<FpLitExpr>();
    e->value = std::get<double>(cur.value);
    advance(); return e;
  }
  if (check(TokenKind::Ident)) {
    std::string name(cur.text);
    advance();
    if (eat(TokenKind::LParen)) { // function call
      auto call = std::make_unique<CallExpr>();
      call->callee = name;
      while (!check(TokenKind::RParen)) {
        call->args.push_back(parseExpr(0));
        if (!eat(TokenKind::Comma)) break;
      }
      expect(TokenKind::RParen);
      return call;
    }
    auto id = std::make_unique<IdentExpr>();
    id->name = name;
    return id;
  }
  if (eat(TokenKind::LParen)) {
    auto e = parseExpr(0);
    expect(TokenKind::RParen);
    return e;
  }
  error("unexpected token in expression: ", cur.text);
}
```

### 55.4.4 Error Recovery

Production parsers implement error recovery to report multiple errors per compilation. The standard strategies are:

**Panic-mode recovery**: when a parse error occurs, discard tokens until a synchronization point (`';'`, `'}'`, or a keyword that starts a statement):

```cpp
void Parser::synchronize() {
  while (!check(TokenKind::Eof)) {
    if (eat(TokenKind::Semicolon)) return;
    switch (cur.kind) {
      case TokenKind::KwFn: case TokenKind::KwLet:
      case TokenKind::KwReturn: case TokenKind::KwIf:
        return;
      default: advance();
    }
  }
}
```

**Structured error nodes**: parse errors produce special `ErrorExpr`/`ErrorStmt` AST nodes. Subsequent passes treat these as typed holes, allowing type checking to continue and collect further errors without crashing.

## 55.5 The Kaleidoscope Reference

The Kaleidoscope tutorial (chapters 1–9 in the LLVM documentation) provides the canonical reference for a minimal LLVM frontend. Its structure is:

| Tutorial chapter | Topic |
|-----------------|-------|
| 1 | Lexer |
| 2 | AST + parser |
| 3 | IR codegen with `IRBuilder<>` |
| 4 | JIT with ORC + optimizer |
| 5 | Control flow (`if`/`then`/`else`, `for`) |
| 6 | User-defined operators |
| 7 | Mutable variables (alloca + mem2reg) |
| 8 | Debug information (DIBuilder) |
| 9 | Ahead-of-time compilation |

The full source is in
[`llvm/examples/Kaleidoscope/`](https://github.com/llvm/llvm-project/tree/llvmorg-22.1.0/llvm/examples/Kaleidoscope/).

### 55.5.1 Kaleidoscope Grammar

```
toplevel ::= definition | extern | expression ';'
definition ::= 'def' prototype expression
extern     ::= 'extern' prototype
prototype  ::= id '(' id* ')'
expression ::= primary binop_rhs
primary    ::= id_expr | num_expr | paren_expr | if_expr | for_expr
binop_rhs  ::= (op primary)*
if_expr    ::= 'if' expression 'then' expression 'else' expression
for_expr   ::= 'for' id '=' expr ',' expr (',' expr)? 'in' expression
```

Kaleidoscope is purely expression-based — every construct produces a `double` value. This simplification allows IR emission (Chapter 56) to be presented without the statement/expression split.

### 55.5.2 Key Kaleidoscope Design Choices

Kaleidoscope makes several choices that differ from a production frontend:

1. **Single type** (`double`): eliminates the type system. Cal adds `int` and `bool`.
2. **No statements**: everything is an expression, returned from the enclosing function.
3. **No mutable locals** (until chapter 7): SSA values only, then alloca-based mutability.
4. **`-` for unknown token**: returns `tok_eof` on encountering `EOF`, `tok_number` for numbers.

These simplifications make Kaleidoscope appropriate for tutorial purposes; a real language requires the full statement/expression split and type system.

## 55.6 Putting the Pieces Together: the Cal Driver

```cpp
int main(int argc, char **argv) {
  if (argc < 2) { llvm::errs() << "usage: cal <file>\n"; return 1; }

  // 1. Read source
  auto bufOrErr = llvm::MemoryBuffer::getFile(argv[1]);
  if (!bufOrErr) { llvm::errs() << "cannot open file\n"; return 1; }
  auto &buf = *bufOrErr;

  // 2. Lex
  Lexer lexer(buf->getBuffer());

  // 3. Parse
  Parser parser(lexer);
  auto mod = parser.parseModule();
  if (!mod) return 1;  // parse errors already reported

  // 4. Semantic analysis (Chapter 56 adds IR emission here)
  TypeChecker checker;
  if (!checker.check(*mod)) return 1;

  // 5. IR emission (Chapter 56)
  llvm::LLVMContext ctx;
  IREmitter emitter(ctx, argv[1]);
  auto llvmMod = emitter.emit(*mod);

  // 6. Write IR
  llvmMod->print(llvm::outs(), nullptr);
  return 0;
}
```

The `llvm::MemoryBuffer::getFile` call (from
[`llvm/include/llvm/Support/MemoryBuffer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/MemoryBuffer.h)) maps the file into memory for zero-copy lexing. The `llvm::errs()` stream routes to stderr without buffering.

## 55.7 Type Checking

Before IR emission, a type-checking pass assigns types to every AST node and catches errors:

```cpp
class TypeChecker {
  // Symbol table: maps variable name to its type
  std::vector<llvm::StringMap<TypeNode*>> scopes;

  TypeNode *lookup(std::string_view name) {
    for (auto it = scopes.rbegin(); it != scopes.rend(); ++it)
      if (auto i = it->find(name); i != it->end()) return i->second;
    return nullptr;
  }

  void pushScope() { scopes.push_back({}); }
  void popScope()  { scopes.pop_back(); }
  void define(std::string_view name, TypeNode *type) { scopes.back()[name] = type; }

public:
  bool check(Module &mod);
  TypeNode *checkExpr(ExprNode *expr);
  bool checkStmt(StmtNode *stmt);
};
```

Type checking traverses the AST and:
1. **Resolves name references**: `IdentExpr` looks up its name in the scope stack.
2. **Infers types**: `BinopExpr` checks operand compatibility and returns the result type.
3. **Checks function calls**: verifies argument count and types against the callee's signature.
4. **Checks return types**: every `return` in a function must be compatible with the declared return type.

Type errors are reported through LLVM's diagnostic infrastructure (or a simpler `llvm::errs()` + error counter for toy frontends):

```cpp
bool TypeChecker::checkFnDecl(FnDecl *fn) {
  pushScope();
  for (auto &[name, type] : fn->params)
    define(name, type.get());

  bool ok = checkBlock(fn->body.get(), fn->retType.get());
  popScope();
  return ok;
}
```

## 55.8 Build System Integration

A standalone frontend links against LLVM's libraries. The minimal link set for a lexer/parser/IRBuilder frontend with no optimization:

```cmake
cmake_minimum_required(VERSION 3.20)
project(Cal)

find_package(LLVM REQUIRED CONFIG)
list(APPEND CMAKE_MODULE_PATH ${LLVM_CMAKE_DIR})

include(AddLLVM)
include(HandleLLVMOptions)

add_definitions(${LLVM_DEFINITIONS})
include_directories(${LLVM_INCLUDE_DIRS})

llvm_map_components_to_libnames(LLVM_LIBS
  core    # LLVMContext, Module, IRBuilder
  support # MemoryBuffer, raw_ostream
  target  # TargetMachine (for -emit-obj)
  X86     # X86 target (or native)
)

add_executable(cal main.cpp lexer.cpp parser.cpp irgen.cpp)
target_link_libraries(cal PRIVATE ${LLVM_LIBS})
```

`llvm_map_components_to_libnames` resolves LLVM component names to the actual library files, handling the difference between monolithic and split-library builds.

---

## Chapter Summary

- A minimal LLVM frontend comprises a lexer, parser, AST, type checker, and IR emitter; each is independently testable.
- Hand-written recursive-descent parsers with Pratt expression parsing are standard for production frontends.
- AST nodes should use arena allocation (`BumpPtrAllocator`) rather than per-node heap allocation for compile-time performance.
- `llvm::StringSwitch` provides efficient keyword dispatch; `llvm::MemoryBuffer::getFile` provides zero-copy file reading.
- The Kaleidoscope tutorial in `llvm/examples/Kaleidoscope/` is the canonical LLVM frontend reference; chapters 1–9 walk lexer → JIT → debug info.
- A type checker assigns types to every AST node before IR emission, catching errors in a single structured pass.
- CMake integration uses `find_package(LLVM)` + `llvm_map_components_to_libnames` to link the minimal required LLVM libraries.


---

@copyright jreuben11
