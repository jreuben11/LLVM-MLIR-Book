# Chapter 182 — Language Tooling: Parsers, Lexers, and Syntax Trees

*Part XXVI — Ecosystem and Frontiers*

Building a production compiler and building language tooling are activities that share surface-level machinery—lexing, parsing, tree construction—but diverge sharply in their requirements and therefore in their optimal implementations. A compiler must parse a valid program exactly once, produce maximally informative error messages for invalid programs, and sustain throughput across millions of lines per minute. An IDE plugin must parse programs that are perpetually half-formed, never block the keystroke loop, survive syntax errors that would halt a compiler entirely, and reparse only the region the user just edited. These two regimes do not admit the same tool.

This chapter maps the Rust-centric lexing and parsing ecosystem, alongside ANTLR4 and TreeSitter for polyglot tooling, against that axis. Where [Chapter 6 — Lexical Analysis](../part-02-compiler-theory/ch06-lexical-analysis.md) and [Chapter 7 — Parsing Theory](../part-02-compiler-theory/ch07-parsing-theory.md) developed the algorithmic foundations—NFA/DFA construction, LL/LR/GLR, PEG, Earley—this chapter treats those algorithms as selection criteria. Where [Chapter 32 — The Parser](../part-05-clang-frontend/ch32-the-parser.md) showed a production-grade hand-written recursive-descent parser and [Chapter 55 — Building a Frontend](../part-09-frontend-authoring/ch55-building-a-frontend.md) walked the Kaleidoscope pipeline, this chapter covers the tools that live between "write it entirely yourself" and "use the full LLVM machinery." The audience is an engineer choosing a lexer/parser library for a DSL, a language server, or a syntax-highlighting grammar.

---

## Table of Contents

- [182.1 The Tooling-vs-Compilation Axis](#1821-the-tooling-vs-compilation-axis)
- [182.2 Logos: Lexer Generation in Rust](#1822-logos-lexer-generation-in-rust)
- [182.3 PEG Parsers](#1823-peg-parsers)
  - [Pest: Grammar Files with a Rich Visitor API](#pest-grammar-files-with-a-rich-visitor-api)
  - [peg: Inline PEG via proc-macro](#peg-inline-peg-via-proc-macro)
  - [pom: Operator-Overloaded Combinators](#pom-operator-overloaded-combinators)
- [182.4 Parser Combinators](#1824-parser-combinators)
  - [Winnow: High-Performance Parsing with Structured Errors](#winnow-high-performance-parsing-with-structured-errors)
  - [combine: Parsec-Style Streaming Combinators](#combine-parsec-style-streaming-combinators)
  - [chumsky: Rich Error Recovery and Built-in Pratt Parsing](#chumsky-rich-error-recovery-and-built-in-pratt-parsing)
- [182.5 LR and LL Parser Generators](#1825-lr-and-ll-parser-generators)
- [182.5b Self-Referential Grammars, Metacompilation, and Compiler-Compilers](#1825b-self-referential-grammars-metacompilation-and-compiler-compilers)
  - [LALRPOP: Grammar-First LR Parsing in Rust](#lalrpop-grammar-first-lr-parsing-in-rust)
  - [Parol: LL(k)/LALR Hybrid](#parol-llklalr-hybrid)
  - [Grmtools: YACC-Compatible LR Parsing](#grmtools-yacc-compatible-lr-parsing)
- [182.6 Lossless Syntax Trees: Rowan](#1826-lossless-syntax-trees-rowan)
- [182.7 ANTLR4: Cross-Language Parser Generation](#1827-antlr4-cross-language-parser-generation)
- [182.8 TreeSitter: Error-Tolerant Incremental Parsing](#1828-treesitter-error-tolerant-incremental-parsing)
  - [Grammar Authoring](#grammar-authoring)
  - [Rust Bindings and Query API](#rust-bindings-and-query-api)
  - [Why TreeSitter Is Not a Compiler Front End](#why-treesitter-is-not-a-compiler-front-end)
- [182.9 The Decision Guide](#1829-the-decision-guide)
- [Chapter Summary](#chapter-summary)

---

## 182.1 The Tooling-vs-Compilation Axis

The spectrum from hand-written to generated spans four broad positions, each making different tradeoffs:

**Hand-written recursive descent** (Clang, rustc's early parser) gives complete control over error messages, no dependencies, and predictable performance, at the cost of significant engineering time and the risk of subtle inconsistency between what the grammar says and what the code accepts. It is the correct choice when the language evolves slowly and compiler-quality diagnostics are a first-class requirement.

**Parser combinators** (Winnow, combine, chumsky, pom) embed the grammar as Rust code. The grammar is readable, the library handles backtracking and memoization, and integration with surrounding code is seamless. Error recovery is possible but requires deliberate effort. Compile times increase with grammar size because the entire combinator graph is monomorphized.

**Generator-based parsers** (LALRPOP, grmtools, ANTLR4, Parol) take a declarative grammar file and emit code. The grammar document becomes the authoritative specification; conflicts surface at generation time rather than at runtime. Generated LALR parsers handle all deterministic context-free grammars and give O(n) guaranteed parse time. The cost is the build step, a learning curve for the grammar language, and sometimes frustrating conflict messages.

**Error-tolerant and incremental parsers** (TreeSitter, Rowan+hand-written) prioritize surviving broken input and updating cheaply on edits. They accept strictly more input than a correct grammar would allow, which makes them unusable as the front end of a compiler but ideal for syntax highlighting, symbol outline, and go-to-definition.

The table below summarizes the main tools covered in this chapter:

| Tool | Strategy | Error recovery | Incremental | Rust-native |
|------|-----------|---------------|-------------|-------------|
| Logos | DFA (compile-time) | Token::Error skip | No | Yes |
| Pest | PEG / packrat | Limited | No | Yes |
| peg | PEG / packrat | Limited | No | Yes |
| pom | PEG combinators | Limited | No | Yes |
| Winnow | PEG combinators | Context errors | No | Yes |
| combine | Parsec combinators | Limited | Partial | Yes |
| chumsky | PEG + Pratt | Rich recovery | No | Yes |
| LALRPOP | LALR(1) | Error token | No | Yes (gen) |
| Parol | LL(k)/LALR hybrid | Limited | No | Yes (gen) |
| grmtools | LALR(1) / Pager | YACC-compatible | No | Yes (gen) |
| Rowan | CST (manual parse) | Complete | Yes | Yes |
| ANTLR4 | ALL(*) LL(*) | Sync-and-return | No | No (C++ runtime) |
| TreeSitter | Incremental GLR | Full | Yes | Bindings |

---

## 182.2 Logos: Lexer Generation in Rust

[Logos](https://github.com/maciejhirsz/logos) constructs a DFA at compile time using a proc-macro derive. The DFA is embedded directly into the binary; tokenization at runtime is a tight dispatch loop with no heap allocation and no regex engine overhead. Performance is comparable to hand-written lexers.

The API derives `Logos` on a token enum. Each variant carries either a `#[token("...")]` attribute for exact string matches or `#[regex("...")]` for pattern matches. The generated `Lexer<'source, Token>` type implements `Iterator` yielding `Result<Token, ()>` per token (where `Err(())` signals an unrecognized byte). Spans are available via `lexer.span()`, which returns the byte range of the current token within the source slice. Because the lexer holds a `&'source str` slice, all string data is zero-copy.

A complete lexer for an expression language:

**`Cargo.toml`**
```toml
[package]
name = "expr-lexer"
version = "0.1.0"
edition = "2024"

[dependencies]
logos = "0.14"
```

```rust
use logos::Logos;

#[derive(Logos, Debug, PartialEq, Clone)]
#[logos(skip r"[ \t\r\n]+")] // skip whitespace
pub enum Token {
    // Literals
    #[regex(r"[0-9]+(\.[0-9]+)?", |lex| lex.slice().parse::<f64>().ok())]
    Number(f64),

    // Identifiers and keywords — keywords are matched first because
    // Logos resolves ties in declaration order for #[token] vs #[regex].
    #[token("let")]
    Let,
    #[token("fn")]
    Fn,
    #[token("if")]
    If,
    #[token("else")]
    Else,
    #[regex(r"[a-zA-Z_][a-zA-Z0-9_]*", |lex| lex.slice().to_owned())]
    Ident(String),

    // Operators
    #[token("+")]  Plus,
    #[token("-")]  Minus,
    #[token("*")]  Star,
    #[token("/")]  Slash,
    #[token("=")]  Eq,
    #[token("==")]  EqEq,
    #[token("(")]  LParen,
    #[token(")")]  RParen,
    #[token(";")]  Semi,
}

fn tokenize(src: &str) -> Vec<(Result<Token, ()>, std::ops::Range<usize>)> {
    Token::lexer(src)
        .spanned()
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn basic_expr() {
        let tokens = tokenize("let x = 3.14 + y;");
        let kinds: Vec<_> = tokens.iter()
            .map(|(t, _)| t.as_ref().ok().cloned())
            .flatten()
            .collect();
        assert!(matches!(kinds[0], Token::Let));
        assert!(matches!(kinds[3], Token::Number(n) if (n - 3.14).abs() < 1e-9));
    }
}
```

The `#[logos(skip ...)]` attribute suppresses whitespace tokens globally, eliminating the need for per-token filtering. When the iterator encounters a byte sequence matching no rule it yields `Err(())` and advances by one byte, giving a simple form of error recovery: the calling parser can collect or skip `Err` tokens without halting.

The `#[regex]` callback closure receives the `Lexer` and returns `Option<T>` (returning `None` skips the match as though it were an error). This is where conversions from string slices to typed values happen — no separate value-extraction pass is needed.

---

## 182.3 PEG Parsers

Three Rust libraries implement PEG parsing with distinct design philosophies.

### Pest: Grammar Files with a Rich Visitor API

[Pest](https://pest.rs/) separates grammar from code. A `.pest` file expresses the grammar; a `#[grammar = "grammar.pest"]` derive on a zero-sized struct triggers code generation at compile time. Pest grammars use `~` for sequence, `|` for ordered choice, `*`/`+`/`?` for repetition, and `&`/`!` for lookahead predicates. Built-in atomic rules like `ASCII_DIGIT` and `NEWLINE` cover common character classes. A special `WHITESPACE` rule, if defined, is inserted automatically between every sequence element.

**`arithmetic.pest`**
```pest
WHITESPACE = _{ " " | "\t" | "\r" | "\n" }

expr    =  { term ~ (("+" | "-") ~ term)* }
term    =  { factor ~ (("*" | "/") ~ factor)* }
factor  =  { number | "(" ~ expr ~ ")" | "-" ~ factor }
number  =  @{ ASCII_DIGIT+ ~ ("." ~ ASCII_DIGIT+)? }
```

The leading `_` on `WHITESPACE` makes it silent — it is consumed but not included in the parse tree. The `@` prefix on `number` makes it atomic (whitespace is not inserted internally).

```rust
use pest::Parser;
use pest_derive::Parser;

#[derive(Parser)]
#[grammar = "arithmetic.pest"]
struct ArithParser;

fn eval(pair: pest::iterators::Pair<Rule>) -> f64 {
    match pair.as_rule() {
        Rule::expr | Rule::term => {
            let mut inner = pair.into_inner();
            let mut result = eval(inner.next().unwrap());
            while let Some(op) = inner.next() {
                let rhs = eval(inner.next().unwrap());
                result = match op.as_str() {
                    "+" => result + rhs,
                    "-" => result - rhs,
                    "*" => result * rhs,
                    "/" => result / rhs,
                    _ => unreachable!(),
                };
            }
            result
        }
        Rule::factor => {
            let mut inner = pair.into_inner();
            let first = inner.next().unwrap();
            if first.as_rule() == Rule::number {
                first.as_str().parse().unwrap()
            } else {
                // parenthesised or negated
                let child = inner.next().unwrap_or(first);
                -eval(child)
            }
        }
        Rule::number => pair.as_str().parse().unwrap(),
        _ => unreachable!(),
    }
}
```

Pest's `Pair` API makes tree traversal straightforward: `pair.into_inner()` returns the children; `pair.as_rule()` discriminates on the grammar rule. Error messages include line/column information via `pest::error::Error<Rule>`.

### peg: Inline PEG via proc-macro

[peg](https://docs.rs/peg) embeds the grammar directly in Rust source using the `peg::parser!` macro, eliminating the separate file. Rules are written inside the macro and can freely call Rust expressions as semantic actions. Packrat memoization is available per-rule with the `#[cache]` attribute.

```rust
peg::parser! {
    pub grammar arithmetic() for str {
        pub rule expr() -> f64
            = e:term() rest:((op:("+"/"-") t:term() { (op, t) })*) {
                rest.iter().fold(e, |acc, (op, t)| {
                    if *op == "+" { acc + t } else { acc - t }
                })
            }

        rule term() -> f64
            = f:factor() rest:((op:("*"/"/") g:factor() { (op, g) })*) {
                rest.iter().fold(f, |acc, (op, g)| {
                    if *op == "*" { acc * g } else { acc / g }
                })
            }

        rule factor() -> f64
            = n:number() { n }
            / "(" e:expr() ")" { e }
            / "-" f:factor() { -f }

        rule number() -> f64
            = n:$(['0'..='9']+ ("." ['0'..='9']+)?) {
                n.parse().unwrap()
            }

        rule _() = [' ' | '\t' | '\n' | '\r']*
    }
}
```

The `$()` combinator captures the matched slice as a `&str`. Because the grammar lives inside the source file, IDE tooling sees it as ordinary Rust; no separate file synchronisation is needed.

### pom: Operator-Overloaded Combinators

[pom](https://docs.rs/pom) takes a third approach: no macros and no grammar files. Combinators are plain Rust values of type `Parser<'a, I, O>`, and sequencing is expressed via Rust operator overloading — `+` for sequence, `|` for choice, `*` and `+` are replaced by `.repeat(..)` and `.repeat(1..)`. This makes the grammar visible to the type system and easy to compose programmatically.

```rust
use pom::parser::*;

fn number<'a>() -> Parser<'a, u8, f64> {
    let integer = one_of(b"0123456789").repeat(1..);
    let frac = sym(b'.') + one_of(b"0123456789").repeat(1..);
    let number = integer + frac.opt();
    number.collect().convert(|s| {
        std::str::from_utf8(s).unwrap().parse::<f64>()
    })
}

fn expr<'a>() -> Parser<'a, u8, f64> {
    (term() + (one_of(b"+-").collect() + term()).repeat(0..))
    .map(|(first, rest)| {
        rest.iter().fold(first, |acc, (op, t)| {
            if op == b"+" { acc + t } else { acc - t }
        })
    })
}
```

pom's operator overloading is syntactically clean but requires all parsers to be written as functions to allow mutual recursion via `call(expr)`. It suits projects that prefer zero macro dependencies.

---

## 182.4 Parser Combinators

### Winnow: High-Performance Parsing with Structured Errors

[Winnow](https://docs.rs/winnow) is the actively maintained successor to nom. The central abstraction is the `Parser` trait: any value implementing `Parser<I, O, E>` can be composed with all built-in combinators. The key method is `parse_next(&mut input: I) -> PResult<O, E>`, where `PResult<O, E>` is an alias for `Result<O, ErrMode<E>>`. Inputs are stateful: after a partial match, the input slice has been advanced in place.

**`Cargo.toml`**
```toml
[package]
name = "winnow-json"
version = "0.1.0"
edition = "2024"

[dependencies]
winnow = "0.6"
```

```rust
use winnow::{
    PResult, Parser,
    ascii::{digit1, multispace0},
    combinator::{alt, delimited, preceded, repeat, separated, seq},
    error::ContextError,
    token::{any, none_of, take_while},
};

#[derive(Debug, Clone, PartialEq)]
pub enum JsonValue {
    Null,
    Bool(bool),
    Number(f64),
    Str(String),
    Array(Vec<JsonValue>),
    Object(Vec<(String, JsonValue)>),
}

fn ws<'s>(input: &mut &'s str) -> PResult<(), ContextError> {
    multispace0.void().parse_next(input)
}

fn null(input: &mut &str) -> PResult<JsonValue, ContextError> {
    "null".map(|_| JsonValue::Null).parse_next(input)
}

fn boolean(input: &mut &str) -> PResult<JsonValue, ContextError> {
    alt((
        "true".map(|_| JsonValue::Bool(true)),
        "false".map(|_| JsonValue::Bool(false)),
    )).parse_next(input)
}

fn number(input: &mut &str) -> PResult<JsonValue, ContextError> {
    // simplified: integer only for clarity
    digit1
        .parse_to::<f64>()
        .context("expected number")
        .map(JsonValue::Number)
        .parse_next(input)
}

fn string_inner(input: &mut &str) -> PResult<String, ContextError> {
    delimited(
        '"',
        repeat(0.., none_of('"')).map(|v: Vec<char>| v.into_iter().collect()),
        '"',
    )
    .context("expected string")
    .parse_next(input)
}

fn json_string(input: &mut &str) -> PResult<JsonValue, ContextError> {
    string_inner.map(JsonValue::Str).parse_next(input)
}

fn array(input: &mut &str) -> PResult<JsonValue, ContextError> {
    delimited(
        ('[', ws),
        separated(0.., value, (',', ws)),
        (ws, ']'),
    )
    .map(JsonValue::Array)
    .context("expected array")
    .parse_next(input)
}

fn key_value(input: &mut &str) -> PResult<(String, JsonValue), ContextError> {
    seq!(
        string_inner,
        _: (ws, ':', ws),
        value,
    )
    .parse_next(input)
}

fn object(input: &mut &str) -> PResult<JsonValue, ContextError> {
    delimited(
        ('{', ws),
        separated(0.., key_value, (',', ws)),
        (ws, '}'),
    )
    .map(JsonValue::Object)
    .context("expected object")
    .parse_next(input)
}

pub fn value(input: &mut &str) -> PResult<JsonValue, ContextError> {
    preceded(
        ws,
        alt((null, boolean, number, json_string, array, object))
            .context("expected JSON value"),
    )
    .parse_next(input)
}
```

The `.context("...")` call annotates the `ContextError` stack: when parsing fails, the error carries all context annotations accumulated along the call chain, enabling error messages of the form `at offset 42: expected JSON value > expected array > expected string`. This is the primary ergonomic advantage of Winnow over lower-level approaches.

Winnow's `Partial<&[u8]>` input wrapper enables streaming/partial parsing: if the parser returns `ErrMode::Incomplete`, the caller can extend the buffer and retry. This is the right model for protocol parsers reading from a socket.

### combine: Parsec-Style Streaming Combinators

[combine](https://docs.rs/combine) models the Haskell Parsec API: `satisfy()` for character predicates, `choice![]` for alternatives, `many()` for repetition, `sep_by()` for comma-separated lists. It operates zero-copy directly on `&str` slices and uses `RangeStream` for efficient backtracking without cloning. The `PartialState` mechanism allows resumable parsing of network streams, similar to Winnow's `Partial` mode. Developers coming from a Haskell or OCaml background typically find combine more natural than nom-derived libraries because the combinator names align with Parsec.

### chumsky: Rich Error Recovery and Built-in Pratt Parsing

[chumsky](https://codeberg.org/zesterer/chumsky) differentiates itself on two axes: rich error recovery and built-in operator precedence parsing. The `.recover_with(skip_then_retry_until(...))` and `nested_delimiters(...)` recovery strategies allow the parser to emit multiple errors per file—crucial for IDE-quality diagnostics. Every token carries a `SimpleSpan` (start + length), and the `MapExtra` mechanism passes spans through combinator chains without threading extra parameters manually.

The `pratt` combinator eliminates the need to encode precedence via grammar structure:

**`Cargo.toml`**
```toml
[package]
name = "chumsky-pratt"
version = "0.1.0"
edition = "2024"

[dependencies]
chumsky = "0.10"
```

```rust
use chumsky::prelude::*;
use chumsky::pratt::{infix, prefix, left, right};

#[derive(Debug, Clone, PartialEq)]
pub enum Expr {
    Num(f64),
    Neg(Box<Expr>),
    Add(Box<Expr>, Box<Expr>),
    Sub(Box<Expr>, Box<Expr>),
    Mul(Box<Expr>, Box<Expr>),
    Div(Box<Expr>, Box<Expr>),
}

pub fn parser<'src>() -> impl Parser<'src, &'src str, Expr, extra::Err<Rich<'src, char>>> {
    let num = text::digits(10)
        .then(just('.').then(text::digits(10)).or_not())
        .to_slice()
        .map(|s: &str| Expr::Num(s.parse().unwrap()))
        .padded();

    let atom = num.or(
        recursive(|expr| expr.delimited_by(just('('), just(')'))).padded()
    );

    atom.pratt((
        // Unary prefix: negation, highest-priority prefix
        prefix(5, just('-').padded(), |_, rhs, _| Expr::Neg(Box::new(rhs))),

        // Binary infix: multiplicative higher than additive
        infix(left(3), just('*').padded(), |lhs, _, rhs, _| {
            Expr::Mul(Box::new(lhs), Box::new(rhs))
        }),
        infix(left(3), just('/').padded(), |lhs, _, rhs, _| {
            Expr::Div(Box::new(lhs), Box::new(rhs))
        }),
        infix(left(2), just('+').padded(), |lhs, _, rhs, _| {
            Expr::Add(Box::new(lhs), Box::new(rhs))
        }),
        infix(left(2), just('-').padded(), |lhs, _, rhs, _| {
            Expr::Sub(Box::new(lhs), Box::new(rhs))
        }),
    ))
}
```

The `pratt(...)` combinator takes an iterable of `infix`, `prefix`, and `postfix` entries, each annotated with a binding power using `left(N)`, `right(N)`, or `none(N)`. This replaces the classical precedence-climbing machinery that would otherwise require separate grammar non-terminals for each precedence level. The parser correctly handles `1 + 2 * 3` as `1 + (2 * 3)`, unary `-` binds tighter than `*`, and right-associative exponentiation (were it added) would use `right(N)`.

Note: chumsky 0.10+ lives on [Codeberg](https://codeberg.org/zesterer/chumsky) after moving from GitHub. The `pratt` closure signatures target the 0.10 API: prefix closures receive `(op, rhs, extra)` (3 args) and infix closures receive `(lhs, op, rhs, extra)` (4 args). If a later release changes arity, the binding-power integers and fold structure remain the same; only the closure parameters change.

---

## 182.5 LR and LL Parser Generators

### LALRPOP: Grammar-First LR Parsing in Rust

[LALRPOP](https://lalrpop.github.io/lalrpop/) takes a `.lalrpop` grammar file and generates a Rust LALR(1) parser. Terminals are defined by inline regular expressions. Actions are Rust expressions embedded directly in the grammar. The build integration is a one-liner in `build.rs`:

```rust
// build.rs
fn main() {
    lalrpop::process_root().unwrap();
}
```

LALRPOP expresses precedence through grammar structure rather than through `%left`/`%right` declarations (those belong to YACC-family tools like grmtools). The standard approach is to use separate non-terminals for each precedence level:

**`src/grammar.lalrpop`**
```lalrpop
use crate::ast::Expr;

grammar;

pub Expr: Box<Expr> = {
    <l:Expr> "+" <r:Term>  => Box::new(Expr::Add(l, r)),
    <l:Expr> "-" <r:Term>  => Box::new(Expr::Sub(l, r)),
    Term,
};

Term: Box<Expr> = {
    <l:Term> "*" <r:Factor> => Box::new(Expr::Mul(l, r)),
    <l:Term> "/" <r:Factor> => Box::new(Expr::Div(l, r)),
    Factor,
};

Factor: Box<Expr> = {
    "(" <Expr> ")",
    "-" <f:Factor>          => Box::new(Expr::Neg(f)),
    <n:Num>                 => Box::new(Expr::Num(n)),
};

Num: f64 = {
    r"[0-9]+(\.[0-9]+)?" => <>.parse().unwrap(),
};
```

LALRPOP also supports an `#[LALR]` attribute and `#[precedence(level="N")]` / `#[assoc(side="left")]` annotations for resolving ambiguities in grammars that are intentionally written ambiguously. Error recovery is enabled by a special `!` token in a grammar rule, which catches parse errors and allows the parser to synchronise at a known delimiter. The generated parser is a plain Rust `struct` with a `parse` method; no runtime library is linked.

### Parol: LL(k)/LALR Hybrid

[Parol](https://jsinger67.github.io/) uses `.par` grammar files and a hybrid LL(k)/LALR(1) algorithm. Its distinguishing feature is `parol-ls`, a VS Code language server for `.par` files that provides syntax highlighting, go-to-definition, and inline grammar diagnostics — making the grammar file itself a first-class development artifact. The `parol_runtime` crate provides the runtime support for generated parsers. Parol suits teams that prefer top-down LL reasoning and want IDE support for grammar development itself.

### Grmtools: YACC-Compatible LR Parsing

[Grmtools](https://softdevteam.github.io/grmtools/) accepts YACC-syntax `.y` grammar files, making it easy to port existing YACC/Bison grammars to Rust. The `lrlex` crate handles the lexer side; `lrpar` handles the parser. Precedence and associativity use familiar YACC declarations:

```
%token INT PLUS TIMES LPAREN RPAREN
%left PLUS
%left TIMES

%%
expr : expr PLUS  expr { $1 + $3 }
     | expr TIMES expr { $1 * $3 }
     | LPAREN expr RPAREN { $2 }
     | INT { $1 }
     ;
```

The `%left`/`%right`/`%nonassoc` declarations are resolved at generation time; the generated parser handles both LALR(1) and Pager's PGM algorithm (which accepts a strictly larger set of grammars). Grmtools is the right choice when a pre-existing YACC grammar is the starting point or when the team has YACC/Bison expertise.

---

## 182.5b Self-Referential Grammars, Metacompilation, and Compiler-Compilers

The preceding sections treated parser generators as black boxes: you write a grammar, the tool emits a parser. This section asks the harder question that underlies all of them: *can a grammar describe itself?* And by extension: *can a compiler compile itself?* The answers connect parser theory to computability, bootstrapping, and the limits of formal verification.

### What Is a Compiler-Compiler?

A **compiler-compiler** (also called a *parser generator* at the grammar level, or a *metacompiler* at the full-language level) is a tool whose input is the description of a language and whose output is a compiler or parser for that language. The concept was introduced by Tony Brooker in 1960 (CPL compiler-compiler), and formalized through:

- **YACC** (Yet Another Compiler-Compiler, Stephen Johnson, Bell Labs 1973): the canonical LR(1) compiler-compiler; generated parsers for C, awk, eqn, and most Unix tools. YACC's grammar for C was used to generate the C parser, making it a historically significant instance of a compiler-compiler processing a production language.
- **Bison** (GNU, 1985–present): the YACC successor. Bison can parse its own grammar description using a parser that was itself generated by an earlier Bison. This is not circular: the *generator* (Bison the program) is a fixpoint; each version of Bison was compiled and tested by a prior version.
- **ANTLR** (Another Tool for Language Recognition, Parr, 1989–present): generates LL(\*) parsers; ANTLR 4's own grammar is written in ANTLR 4 and processed by an earlier ANTLR build — again a staged bootstrap.

The modern tools covered in this chapter (LALRPOP, Parol, grmtools) follow the same pattern. The key insight: the *grammar metalanguage* (the language used to write grammars) is itself a formal language, and a compiler-compiler must include a parser for that metalanguage. That parser is typically generated by an earlier version of the tool.

### Self-Describing Grammars

A grammar is **self-describing** (or *self-applicable*) if it can parse its own grammar notation. This is a fixpoint: the grammar *G* written in metalanguage *M* describes exactly those strings that are valid *M* grammars, and *G* is itself a valid *M* grammar, so *G* parses *G*.

Self-describing grammars exist for PEG, EBNF, and several other formalisms. The most famous example is the BNF/EBNF grammar for BNF itself, which appears in ISO/IEC 14977:

```ebnf
(* EBNF grammar for EBNF — ISO 14977 *)
syntax        = syntax rule, {syntax rule} ;
syntax rule   = meta identifier, '=', definitions list, ';' ;
definitions   = single definition, {'|', single definition} ;
single def    = term, {',', term} ;
term          = factor, ['-', factor] ;
factor        = [integer, '*'], primary ;
primary       = meta identifier | terminal string | group | option | repetition ;
group         = '(', definitions list, ')' ;
option        = '[', definitions list, ']' ;
repetition    = '{', definitions list, '}' ;
terminal str  = "'", {character - "'"}, "'" | '"', {character - '"'}, '"' ;
```

Parsing this grammar with a parser generated *from* this grammar produces a self-referential fixpoint. In practice, EBNF-based tools bootstrap by having a hand-written parser for the initial version of their metalanguage, then generating subsequent parsers.

### The Metacircular Interpreter and Metacompilation

A **metacircular interpreter** is an interpreter for language L written in L that interprets L programs by executing them. The canonical example is the Lisp metacircular evaluator from McCarthy's 1960 paper, reproduced in SICP. It is structurally a quine-like fixpoint: the evaluator applied to itself converges (it interprets itself interpreting programs).

**Metacompilation** extends this to compilers: a compiler for L written in L, compiled by an earlier version of itself. The stages are:

1. *Seed compiler*: a compiler for a restricted subset of L, written in assembly or C (the "bootstrap compiler").
2. *Stage 1*: the full L compiler *S* compiled by the seed. The output is a binary *B1* that compiles full L.
3. *Stage 2*: *S* compiled by *B1*, producing *B2*.
4. *Fixpoint check*: if *B1* compiled from *S* equals *B2* compiled from *S*, the compiler is self-consistent (a fixpoint). This is the same check performed by the LLVM bootstrap build (Ch80).

Languages where the compiler is written in itself: **Rust** (rustc is written in Rust), **Go** (gc is written in Go since Go 1.5), **Clang** (can compile itself), **GHC** (written in Haskell). The fixpoint property — that compiling the compiler with itself produces the same binary — is the practical definition of a self-hosting compiler.

### Quines as Compiler Correctness Tests

A **quine** is a program that outputs its own source. In the context of compiler testing, a quine that is compiled and executed produces a string that, when fed back to the compiler, produces the same binary. This gives a concrete correctness test:

```
compile(quine_source) → binary Q
execute(Q) → output_string
compile(output_string) → binary Q'
assert(Q == Q')
```

If the compiler is correct, Q == Q'. If the compiler introduces a mutation — even a semantics-preserving one like register renaming that changes binary layout — the test may detect it. Quine tests are used in research to detect *semantic-preserving but observably different* compiler transformations.

A **quine relay** (or *quine chain*) takes this further: a program in language A that outputs a program in language B that outputs a program in language C ... that eventually outputs the original program in language A. Quine relays up to 128 languages have been constructed. For compiler engineers, the interesting case is a two-language relay: a program in LLVM IR that outputs a Clang C program that compiles to the same LLVM IR — a fixpoint between the source and IR levels.

### Metacompilers and LLVM

LLVM is not itself a metacompiler, but several components exhibit the self-referential structure:

- **Clang self-hosting**: Clang can compile its own source. The LLVM release process verifies this via the stage1→stage2→stage3 bootstrap (Ch80). The stage2 == stage3 check is the fixpoint assertion.
- **MLIR ODS and TableGen**: ODS (Operation Definition Specification) is a DSL for defining MLIR operations, written using MLIR's own TableGen infrastructure. TableGen defines a language for describing TableGen backends. This is not strictly self-hosting (TableGen is compiled by Clang, not by itself), but the self-referential dependency graph is the same pattern.
- **The LLVM libc compiler-rt bootstrap**: compiler-rt (the runtime library) is required to compile certain programs, but compiler-rt itself requires a working compiler. The bootstrap sequence handles this circularity explicitly in CMake.
- **Clang-pseudo** (Ch47): a GLR parser for C++ written in C++ and using a C++ grammar encoded in a pseudo-grammar format. When applied to its own source, it parses the parser — the self-referential structure is present, though not exploited for bootstrapping.

The general lesson for compiler engineers: wherever a language processor is applied to its own description — grammar, specification, or source — the fixpoint structure of quines and metacircular interpreters appears. Recognizing this structure clarifies bootstrapping requirements, identifies circular dependencies in build systems, and motivates the diverse double compilation defense against supply-chain attacks.

---

## 182.6 Lossless Syntax Trees: Rowan

[Rowan](https://docs.rs/rowan) is the lossless CST library at the core of rust-analyzer. "Lossless" means that every byte of the source file—whitespace, comments, even malformed tokens—is represented in the tree and can be round-tripped back to the exact original string. This property is essential for formatting tools, automated refactoring, and diff-based code review, where mutating the AST and pretty-printing it would silently discard user formatting choices.

Rowan maintains two concurrent representations:

**Green tree** (immutable, persistent): `GreenNode` and `GreenToken` nodes contain a `SyntaxKind` (a newtype `u16` defined by the language using Rowan), a text length, and a list of child green nodes. Green nodes are reference-counted and deduplicated — two identical subtrees share a pointer. Edits create a new root with structural sharing of unchanged subtrees, making incremental reparsing efficient.

**Red tree** (ephemeral, computed on demand): `SyntaxNode` and `SyntaxToken` are thin wrappers computed from the green tree that add parent pointers and absolute text offsets. They are created on demand and not stored; traversal through `SyntaxNode::parent()` recomputes the path to root each time.

`SyntaxNode` provides `children()`, `children_with_tokens()`, `first_token()`, `last_token()`, and `text_range()` (a `TextRange` = `TextSize` start + `TextSize` end). Queries like "find all identifiers in this function" traverse the red layer; bulk tree operations work on the green layer.

```rust
use rowan::{GreenNodeBuilder, Language, SyntaxKind, SyntaxNode};

// Define SyntaxKinds for our language
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
#[repr(u16)]
enum SyntaxKindEnum {
    Root = 0,
    BinaryExpr,
    Number,
    Plus,
    Whitespace,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
struct ExprLang;

impl Language for ExprLang {
    type Kind = SyntaxKindEnum;

    fn kind_from_raw(raw: rowan::SyntaxKind) -> SyntaxKindEnum {
        // In a real implementation, use a match or enum conversion
        match raw.0 {
            0 => SyntaxKindEnum::Root,
            1 => SyntaxKindEnum::BinaryExpr,
            2 => SyntaxKindEnum::Number,
            3 => SyntaxKindEnum::Plus,
            4 => SyntaxKindEnum::Whitespace,
            _ => panic!("unknown kind"),
        }
    }

    fn kind_to_raw(kind: SyntaxKindEnum) -> rowan::SyntaxKind {
        rowan::SyntaxKind(kind as u16)
    }
}

type SyntaxNodeEx = SyntaxNode<ExprLang>;

fn build_tree_for_one_plus_two() -> SyntaxNodeEx {
    let mut builder = GreenNodeBuilder::new();

    builder.start_node(rowan::SyntaxKind(SyntaxKindEnum::Root as u16));
    builder.start_node(rowan::SyntaxKind(SyntaxKindEnum::BinaryExpr as u16));
    builder.token(rowan::SyntaxKind(SyntaxKindEnum::Number as u16), "1");
    builder.token(rowan::SyntaxKind(SyntaxKindEnum::Whitespace as u16), " ");
    builder.token(rowan::SyntaxKind(SyntaxKindEnum::Plus as u16), "+");
    builder.token(rowan::SyntaxKind(SyntaxKindEnum::Whitespace as u16), " ");
    builder.token(rowan::SyntaxKind(SyntaxKindEnum::Number as u16), "2");
    builder.finish_node();
    builder.finish_node();

    SyntaxNode::new_root(builder.finish())
}

fn main() {
    let root = build_tree_for_one_plus_two();
    println!("Root text: {:?}", root.text());      // "1 + 2"
    println!("Root range: {:?}", root.text_range()); // 0..5
    for child in root.children_with_tokens() {
        println!("  {:?} {:?}", child.kind(), child.text_range());
    }
}
```

The green tree correctly reflects the whitespace tokens; `root.text()` returns `"1 + 2"` byte-for-byte. When rust-analyzer reformats a function signature, it edits the green tree layer and then serialises back: no information is lost.

The `ra_ap_syntax` crate (the API-stable surface of rust-analyzer's syntax layer) wraps Rowan with hundreds of Rust-specific `SyntaxKind` variants and typed AST node wrappers generated from a grammar description. Application code rarely manipulates raw `SyntaxKind` values directly.

**When to use Rowan vs a traditional AST:** For IDE tools that must round-trip source and support incremental reparsing, Rowan's lossless CST is the right foundation. For a production compiler where the AST is only read (never printed back), a traditional Rust enum-based AST has simpler ownership semantics, smaller memory footprint, and faster traversal — there is no need to carry whitespace tokens through every optimization pass.

---

## 182.7 ANTLR4: Cross-Language Parser Generation

ANTLR4 uses the ALL(*) algorithm — an adaptive LL(*) algorithm that predicts alternatives using a dynamic analysis of the parse context. It accepts all LL grammars and a large class of ambiguous grammars (resolving ambiguity in favour of the first alternative, like PEG). The grammar syntax (`.g4`) uses uppercase rule names for lexer rules and lowercase for parser rules.

**`Arithmetic.g4`**
```antlr
grammar Arithmetic;

prog    : stat+ EOF ;
stat    : expr NEWLINE
        | NEWLINE
        ;
expr    : expr ('*'|'/') expr   # MulDiv
        | expr ('+'|'-') expr   # AddSub
        | '(' expr ')'          # Parens
        | INT                   # Int
        | FLOAT                 # Float
        ;

INT     : [0-9]+ ;
FLOAT   : [0-9]+ '.' [0-9]* ;
NEWLINE : [\r\n]+ ;
WS      : [ \t]+ -> skip ;
```

The `# Label` annotations on alternatives generate separate visitor methods, making each case of `expr` a distinct dispatch target. Ambiguity in `expr` (both `*` and `+` alternatives can match the same input) is resolved by ANTLR4's ALL(*) prediction engine using the rule-order precedence convention.

The C++ runtime walks the parse tree via either the Visitor or Listener pattern. In the Visitor pattern, the caller controls traversal and can return values; in the Listener pattern, ANTLR4's `ParseTreeWalker` drives traversal and fires `enter`/`exit` callbacks:

```cpp
#include "antlr4-runtime.h"
#include "ArithmeticLexer.h"
#include "ArithmeticParser.h"
#include "ArithmeticBaseListener.h"

class EvalListener : public ArithmeticBaseListener {
public:
    std::stack<double> stack;

    void exitInt(ArithmeticParser::IntContext* ctx) override {
        stack.push(std::stod(ctx->INT()->getText()));
    }
    void exitMulDiv(ArithmeticParser::MulDivContext* ctx) override {
        double rhs = stack.top(); stack.pop();
        double lhs = stack.top(); stack.pop();
        if (ctx->op->getType() == ArithmeticParser::T__0) // '*'
            stack.push(lhs * rhs);
        else
            stack.push(lhs / rhs);
    }
    void exitAddSub(ArithmeticParser::AddSubContext* ctx) override {
        double rhs = stack.top(); stack.pop();
        double lhs = stack.top(); stack.pop();
        if (ctx->op->getType() == ArithmeticParser::T__2) // '+'
            stack.push(lhs + rhs);
        else
            stack.push(lhs - rhs);
    }
};

int main() {
    antlr4::ANTLRInputStream input("3 + 4 * 2\n");
    ArithmeticLexer lexer(&input);
    antlr4::CommonTokenStream tokens(&lexer);
    ArithmeticParser parser(&tokens);
    auto* tree = parser.prog();

    EvalListener listener;
    antlr4::tree::ParseTreeWalker::DEFAULT.walk(&listener, tree);
    std::cout << listener.stack.top() << "\n"; // 11
}
```

ANTLR4's `DefaultErrorStrategy` implements sync-and-return recovery: on a syntax error it consumes tokens until a safe resynchronisation point, then continues parsing. `BailErrorStrategy` throws `ParseCancellationException` on the first error, appropriate when the caller wants to handle errors itself. Custom `ANTLRErrorListener` subclasses redirect error messages to diagnostics infrastructure.

Connecting ANTLR4 to LLVM IR generation follows the pattern established in [Chapter 55 — Building a Frontend](../part-09-frontend-authoring/ch55-building-a-frontend.md): the listener or visitor populates a Rust/C++ AST, then a separate codegen pass lowers the AST to LLVM IR via `inkwell` (as described in [Chapter 178 — The Rust Compiler Ecosystem](ch178-the-rust-compiler-ecosystem.md)). The parse tree itself is transient; it should be converted to a typed AST before semantic analysis.

ANTLR4's advantage over Rust-native generators is its polyglot runtime: the same `.g4` grammar can generate parsers for Java, Python, C++, C#, Go, and JavaScript, making it the right choice when a grammar must be shared across implementations in different languages.

---

## 182.8 TreeSitter: Error-Tolerant Incremental Parsing

TreeSitter is an incremental GLR parser with error recovery, designed exclusively for editors and other tools that must parse partial, syntactically invalid programs at interactive speed. It is architecturally incompatible with compilation: it accepts ambiguous and partially-matched input, emits nodes typed as `ERROR`, and provides no semantic information. Its value is in use cases where the alternative—not parsing at all—is worse than parsing incorrectly.

### Grammar Authoring

Grammars are written in `grammar.js`, a JavaScript DSL:

```javascript
// grammar.js (TreeSitter grammar for a simple function definition)
module.exports = grammar({
  name: 'simplelang',
  rules: {
    source_file: $ => repeat($._definition),
    _definition: $ => $.function_definition,

    function_definition: $ => seq(
      'fn',
      field('name', $.identifier),
      '(',
      field('params', optional($.param_list)),
      ')',
      field('body', $.block),
    ),

    param_list: $ => seq(
      $.identifier,
      repeat(seq(',', $.identifier)),
    ),

    block: $ => seq('{', repeat($._statement), '}'),
    _statement: $ => seq($.expression, ';'),
    expression: $ => choice($.identifier, $.number, $.binary_expr),

    binary_expr: $ => prec.left(1, seq(
      $.expression, choice('+', '-', '*', '/'), $.expression
    )),

    identifier: $ => /[a-zA-Z_][a-zA-Z0-9_]*/,
    number:     $ => /[0-9]+/,
  }
});
```

`tree-sitter generate` compiles this to a C parser. `tree-sitter test` runs corpus test cases defined in `test/corpus/` files that pair input strings with expected tree s-expressions. The generated C source is then included in a Rust crate via a `build.rs` that compiles it with the `cc` crate.

### Rust Bindings and Query API

```rust
use tree_sitter::{Language, Node, Parser, Query, QueryCursor};

extern "C" { fn tree_sitter_simplelang() -> Language; }

fn extract_function_names(source: &str) -> Vec<String> {
    let language = unsafe { tree_sitter_simplelang() };
    let mut parser = Parser::new();
    parser.set_language(&language).unwrap();

    let tree = parser.parse(source, None).unwrap();
    let root = tree.root_node();

    // Tree-sitter query pattern: capture function name identifiers
    let query_src = r#"(function_definition name: (identifier) @fn_name)"#;
    let query = Query::new(&language, query_src).unwrap();
    let mut cursor = QueryCursor::new();

    let mut names = Vec::new();
    for m in cursor.matches(&query, root, source.as_bytes()) {
        for capture in m.captures {
            let name = capture.node.utf8_text(source.as_bytes()).unwrap();
            names.push(name.to_owned());
        }
    }
    names
}

fn incremental_reparse_demo(source: &str, edit_source: &str) {
    let language = unsafe { tree_sitter_simplelang() };
    let mut parser = Parser::new();
    parser.set_language(&language).unwrap();

    // Initial parse
    let old_tree = parser.parse(source, None).unwrap();

    // Simulate an edit: insert a character at position 5
    let mut new_tree_holder = old_tree.clone();
    new_tree_holder.edit(&tree_sitter::InputEdit {
        start_byte: 5,
        old_end_byte: 5,
        new_end_byte: 6,
        start_position: tree_sitter::Point::new(0, 5),
        old_end_position: tree_sitter::Point::new(0, 5),
        new_end_position: tree_sitter::Point::new(0, 6),
    });

    // Incremental reparse: only affected subtrees are rebuilt
    let new_tree = parser.parse(edit_source, Some(&new_tree_holder)).unwrap();
    let _ = new_tree; // use the updated tree
}
```

The tree-sitter query language uses S-expression patterns that match against node kinds and field names. `(function_definition name: (identifier) @fn_name)` matches any `function_definition` node and captures its `name` field (an `identifier`) under the name `fn_name`. Multiple captures, predicates (`#eq?`, `#match?`), and alternation (`[(identifier)(type_identifier)]`) make queries expressive enough for symbol extraction and semantic token highlighting.

Incremental reparsing passes the old tree as the second argument to `parser.parse()`. TreeSitter identifies which subtrees are unaffected by the edit (using byte-range comparison) and reuses their cached parse results. In a 10,000-line file where the user types a single character, the parser touches only the subtrees whose byte ranges overlap the edit region, making latency independent of file size for local edits.

### Why TreeSitter Is Not a Compiler Front End

TreeSitter deliberately accepts ambiguous and malformed input. Its error recovery inserts `ERROR` nodes and continues; the resulting tree may have structurally incorrect subtrees. There is no type information, no scope resolution, and no semantic analysis. nvim-treesitter uses TreeSitter trees for syntax highlighting (assigning highlight groups to node kinds), indentation, and fold detection — tasks that tolerate structural approximations. clangd provides semantic intelligence (hover types, go-to-definition, cross-reference) via the LSP protocol and a full Clang parse, coexisting alongside nvim-treesitter in the same editor session: TreeSitter handles visual rendering at keystroke speed; clangd handles semantics asynchronously in a background process.

---

## 182.9 The Decision Guide

| Use case | Recommended tool | Why |
|----------|-----------------|-----|
| Production compiler for new language | Hand-written recursive descent or LALRPOP | Full control over error messages and recovery; LALRPOP generates correct LALR(1) without manual precedence climbing |
| DSL embedded in a Rust codebase | Winnow or chumsky | No build step, no separate grammar file, combinators compose with surrounding Rust naturally, chumsky adds recovery |
| IDE/editor syntax support | TreeSitter | Error-tolerant incremental GLR; polyglot; no semantic analysis required |
| IDE semantic intelligence | clangd/LSP or custom language server | Syntax alone is insufficient; needs full type resolution via Clang or equivalent |
| Grammar-first language design | ANTLR4 (polyglot) or LALRPOP (Rust-native) | Grammar file is the specification; conflicts surface at generation time rather than at test time |
| Lossless source manipulation (refactoring, formatting) | Rowan | Preserves all whitespace and comments; green-tree sharing enables cheap incremental edits |
| Quick expression language with operator precedence | chumsky (Pratt combinator) | Built-in precedence avoids separate grammar non-terminals per precedence level; composable |
| Lexer only | Logos | Fastest compile-time DFA; zero-copy; no runtime regex engine |
| Porting a YACC/Bison grammar to Rust | grmtools | YACC-syntax `.y` files; `%left`/`%right` declarations; minimal friction migration path |
| Streaming/protocol parser over a network connection | Winnow with `Partial<&[u8]>` | `ErrMode::Incomplete` signals the caller to extend the buffer; no restart from scratch |

**Production compiler (hand-written or LALRPOP).** When error messages are a product feature—users read and act on them—hand-writing the parser gives complete control over what is reported and how. LALRPOP is a practical middle ground: it guarantees LALR(1) correctness and eliminates the grammar/code synchronisation burden, while producing idiomatic Rust output with no runtime dependency.

**DSL in Rust (Winnow or chumsky).** When the language is small, the grammar evolves with the application, and the team does not want a separate build artifact, combinators beat generators. Winnow is the performance-first choice with structured error context; chumsky adds error recovery strategies and the Pratt parser for languages with rich operator precedence.

**IDE syntax (TreeSitter).** No alternative matches TreeSitter for editor integration: error recovery is first-class, incremental reparsing is O(edit size) for local changes, and the query language handles symbol extraction without writing a full pass. The grammar.js syntax requires a JavaScript build step but the resulting C parser is embeddable anywhere.

**Lossless manipulation (Rowan).** Formatters and refactoring tools that must not silently alter user formatting cannot use a lossy AST. Rowan's green-tree persistence means that a formatter can parse, modify a single node, and serialise back—touching only the changed region.

**Lexer only (Logos).** When the parser is hand-written but the lexer is mechanical—keywords, operators, string literals, numeric literals—Logos generates a DFA that matches the performance of a hand-optimised switch statement. The proc-macro runs once at compile time; there is no runtime startup cost.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **Logos 0.15 proc-macro rewrite**: the Logos maintainers have an open RFC to replace the legacy `logos-codegen` proc-macro backend with a new `logos-codegen2` crate that resolves long-standing issues with overlapping regex priorities and improves compile-time DFA minimisation diagnostics. Track at [github.com/maciejhirsz/logos/issues/](https://github.com/maciejhirsz/logos/issues/).
- **chumsky 0.10 stable API stabilisation**: chumsky 0.10 (moved to Codeberg) re-architected the `Parser` trait with GATs to eliminate `Clone` bounds and unify zero-copy and owned-output parsers. The `pratt` combinator closure arity stabilisation and `MapExtra` API finalisation are the remaining blocking items before a 1.0 release candidate.
- **TreeSitter 0.25 ABI freeze**: TreeSitter's C ABI has been informally stable but the project has signalled an intent to freeze the `TSLanguage` struct layout at 0.25 for embedding in editors without runtime version negotiation. This affects grammar authors linking against `tree-sitter-language` crates in Rust.
- **LALRPOP error recovery improvements**: LALRPOP issue [#678](https://github.com/lalrpop/lalrpop/issues/678) tracks extending the `!` error-recovery token with configurable synchronisation sets, similar to ANTLR4's `DefaultErrorStrategy`, enabling multi-error reporting in DSL toolchains.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **LLVM clang-pseudo reaching production readiness**: `clang-pseudo` ([llvm-project/clang-tools-extra/clang-pseudo/](https://github.com/llvm/llvm-project/tree/main/clang-tools-extra/clang-pseudo)) is an experimental GLR-based C++ parser intended for tooling use cases where Clang's full parser is too expensive. The project aims to support incremental reparsing on edit events similar to TreeSitter, closing the performance gap between `clangd` and TreeSitter-based editor plugins for large TUs.
- **Winnow streaming I/O integration with async Rust**: Winnow's `Partial<&[u8]>` streaming model is synchronous. A tracked RFC ([github.com/winnow-rs/winnow/discussions/](https://github.com/winnow-rs/winnow/discussions/)) explores an async `AsyncParser` trait that integrates with `tokio::io::AsyncRead`, enabling zero-copy protocol parsing without blocking executor threads — relevant for language servers that parse over stdin.
- **ANTLR5 ALL(*) successor grammar**: Terence Parr has described ANTLR5 as moving from the ALL(*) algorithm to a PEG-with-left-recursion algorithm (similar to the approach in `rats!`), which would give O(n) guaranteed parse time and eliminate the dynamic prediction overhead of ALL(*). This would make ANTLR5 competitive with LALRPOP for deterministic languages while retaining the polyglot runtime.
- **Rowan typed-CST code generation standardisation**: rust-analyzer's `ungrammar` format (which describes typed AST node wrappers over Rowan's raw tree) has been proposed as a shared standard for languages building on Rowan. A community effort to publish `ungrammar` as a standalone crate and toolchain (separate from `ra_ap_syntax`) would allow other language servers to generate Rowan wrappers without forking the rust-analyzer infrastructure.

### 5-Year Horizon (Long-Term, by ~2031)

- **Verified parser generators with Lean 4 or Coq backends**: research from the Iris/CompCert communities has explored mechanically verifying LALR and LL parser generators — ensuring that the generated parser is a correct implementation of the grammar. A practical tool (targeting Rust output with a Lean 4 proof certificate) would eliminate parser-generator bugs as a class of security vulnerability in language tooling, analogous to Alive2's role in verifying LLVM optimisations.
- **TreeSitter as a universal IDE protocol layer**: the Language Server Protocol currently requires each language server to implement its own syntax highlighting, folding, and breadcrumbs. An LSP extension that allows language servers to supply TreeSitter grammars and node-kind-to-semantic-token mappings would let editors use a single incremental parsing engine for all languages, reducing duplicated effort and enabling richer cross-language tooling (e.g., polyglot notebooks).
- **Incremental parser combinators for streaming source edits**: current combinator libraries (Winnow, chumsky) rebuild the entire parse from the changed token onward. Research into *online* or *reusable* parser combinators — where each combinator node caches its last parse result and invalidates only when its input span overlaps an edit — would bring Rowan-style incremental reparsing to combinator-based parsers, making them viable for IDE backends at the scale of rust-analyzer.

---

## Chapter Summary

- The tooling-vs-compilation axis separates two regimes with genuinely different requirements: production parsers optimise for correctness and diagnostics; IDE parsers optimise for error tolerance and incremental updates.
- **Logos** generates a compile-time DFA lexer via proc-macro derive; tokens carry zero-copy `&'source str` spans; errors are `Err(())` values that allow skip-recovery.
- **Pest** uses separate `.pest` grammar files with `~`, `|`, `@`, and `_` syntax; the `Pairs` API gives positional tree traversal; error messages include `LineColLocation`.
- **peg** and **pom** offer PEG parsing without a separate file — the former via `peg::parser!` macro, the latter via pure Rust combinator operator overloading.
- **Winnow** (nom successor) uses `Parser::parse_next` and stateful input slices; `.context(...)` chains produce structured error stacks; `Partial<&[u8]>` enables streaming.
- **chumsky**'s `pratt(...)` combinator replaces precedence-via-grammar-structure with explicit binding-power declarations; `.recover_with(...)` enables multiple-error reporting.
- **LALRPOP** generates LALR(1) parsers from `.lalrpop` files with inline Rust actions; precedence is expressed via grammar structure or `#[precedence]` / `#[assoc]` attributes, not `%left`/`%right` (those belong to YACC/grmtools).
- **grmtools** accepts YACC-syntax `.y` grammars with `%left`/`%right`/`%nonassoc`, making it the lowest-friction migration path for existing YACC grammars.
- **Rowan** maintains a lossless green/red tree pair; green trees are persistent and structurally shared; red trees add parent pointers on demand; `ra_ap_syntax` wraps Rowan for Rust-specific tooling.
- **ANTLR4** uses ALL(*) adaptive LL parsing; the same `.g4` grammar targets multiple runtime languages; visitor and listener traversal patterns connect to downstream IR generation.
- **TreeSitter** uses incremental GLR with full error recovery; query patterns match node kinds and field names; incremental reparsing is O(edit size); it is unsuitable as a compiler front end.
- Tool selection is driven by use case: Logos for lexers, Winnow/chumsky for embedded DSLs, LALRPOP/ANTLR4 for grammar-first design, Rowan for lossless manipulation, TreeSitter for editor integration.

---

@copyright jreuben11
