# Appendix J — Grammar Notation Reference: BNF, EBNF, PEG, and Modern Alternatives

Grammar notations are the language in which programming languages describe themselves. Every language specification, parser generator, and formal verification tool uses one: the LLVM LangRef uses informal EBNF, Python's grammar uses a PEG variant, TreeSitter grammars are written in JavaScript DSL, and the JVM spec uses a BNF-derived notation with explicit character encodings. This appendix surveys BNF, EBNF, and five modern successors, provides a comparison table, and gives a decision guide for choosing the right notation for a given use case.

---

## Table of Contents

- [J.1 Historical Background](#j1-historical-background)
  - [BNF (Backus-Naur Form)](#bnf-backus-naur-form)
  - [EBNF (Extended BNF)](#ebnf-extended-bnf)
  - [ISO/IEC 14977:1996](#isoiec-149771996)
- [J.2 BNF Core Constructs](#j2-bnf-core-constructs)
  - [Left vs Right Recursion](#left-vs-right-recursion)
- [J.3 EBNF Constructs and ISO 14977](#j3-ebnf-constructs-and-iso-14977)
  - [Wirth-Style EBNF (Pascal, Oberon, Modula)](#wirth-style-ebnf-pascal-oberon-modula)
  - [ISO 14977 Extensions](#iso-14977-extensions)
  - [W3C EBNF (XML, XPath, SPARQL)](#w3c-ebnf-xml-xpath-sparql)
- [J.4 Notation Variants in Practice](#j4-notation-variants-in-practice)
- [J.5 PEG — Parsing Expression Grammars](#j5-peg-parsing-expression-grammars)
  - [PEG Operators](#peg-operators)
  - [Packrat Parsing](#packrat-parsing)
  - [What PEG Cannot Express](#what-peg-cannot-express)
  - [PEG Tools](#peg-tools)
- [J.6 Ohm](#j6-ohm)
  - [Grammar Definition](#grammar-definition)
  - [Semantics Objects](#semantics-objects)
  - [Lexical vs Syntactic Rules](#lexical-vs-syntactic-rules)
  - [Rule Inheritance](#rule-inheritance)
- [J.7 ANTLR4 `.g4` Format](#j7-antlr4-g4-format)
  - [Grammar Structure](#grammar-structure)
  - [Key Features](#key-features)
- [J.8 TreeSitter `grammar.js`](#j8-treesitter-grammarjs)
  - [Grammar Definition](#grammar-definition)
  - [Key Constructs](#key-constructs)
  - [External Scanners](#external-scanners)
  - [Query Language](#query-language)
- [J.9 Railroad / Syntax Diagrams](#j9-railroad-syntax-diagrams)
  - [Structure](#structure)
  - [Generation Tools](#generation-tools)
- [J.10 Earley Parsing and Marpa](#j10-earley-parsing-and-marpa)
  - [Earley's Algorithm](#earleys-algorithm)
  - [Marpa](#marpa)
- [J.11 Notation Selection Guide](#j11-notation-selection-guide)
  - [Decision Table](#decision-table)
  - [Decision Flowchart](#decision-flowchart)
- [J.12 Converting Between Notations](#j12-converting-between-notations)
  - [EBNF → BNF](#ebnf-bnf)
  - [BNF → EBNF](#bnf-ebnf)
  - [EBNF → PEG](#ebnf-peg)
  - [Left-Recursion Elimination (for PEG and LL)](#left-recursion-elimination-for-peg-and-ll)
  - [Right-Factoring (for LL)](#right-factoring-for-ll)
  - [Common Pitfalls](#common-pitfalls)
- [Quick Reference: Notation Symbols](#quick-reference-notation-symbols)

---

## J.1 Historical Background

### BNF (Backus-Naur Form)

John Backus invented the notation in 1959 for the ALGOL 58 report. Peter Naur revised it for the ALGOL 60 report (1960), which first used the name "Backus Normal Form" (later renamed Backus-Naur Form at Knuth's suggestion). The two operators — `::=` for production and `|` for alternation — remain the core of every successor notation:

```bnf
<digit>  ::= "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9"
<number> ::= <digit> | <number> <digit>
```

BNF has no built-in repetition or optionality — both must be expressed via left or right recursion.

### EBNF (Extended BNF)

Niklaus Wirth extended BNF in 1977 for the Pascal Report. The three new constructs:
- `{ A }` — zero or more repetitions of A
- `[ A ]` — A is optional (zero or one occurrence)
- `( A | B )` — grouping for alternation

```ebnf
number = digit { digit }
digit  = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9"
```

Wirth used `=` instead of `::=` and plain names (no angle brackets) for nonterminals.

### ISO/IEC 14977:1996

The international standard for EBNF. Introduces explicit punctuation: concatenation via `,`, termination via `;`, exception via `-`, special sequence via `? ... ?`, comment via `(* ... *)`. Almost never used verbatim — most "EBNF" encountered in practice is informal Wirth-style, not ISO 14977.

```
(* ISO 14977 example *)
number = digit , { digit } ;
digit  = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;
```

---

## J.2 BNF Core Constructs

| Construct | Notation | Meaning |
|-----------|----------|---------|
| Production | `<A> ::= ...` | Defines nonterminal A |
| Alternation | `A \| B` | A or B |
| Concatenation | `A B` (juxtaposition) | A followed by B |
| Terminal | `"text"` or `'text'` | Literal string |
| Nonterminal | `<name>` or bare `name` | Reference to another rule |
| Empty / epsilon | (empty RHS, or explicit `ε`) | Matches the empty string |

### Left vs Right Recursion

```bnf
(* Left-recursive: natural for left-associative operators *)
expr ::= expr "+" term | term

(* Right-recursive: natural for right-associative operators *)
expr ::= term "+" expr | term
```

Left recursion is required for LR parsers and natural for left-associative expressions. LL parsers (including PEG) require left-recursion elimination (right-factoring) or special handling (ANTLR4 does this automatically).

---

## J.3 EBNF Constructs and ISO 14977

### Wirth-Style EBNF (Pascal, Oberon, Modula)

```
letter = "A" ... "Z" | "a" ... "z" .
ident  = letter { letter | digit } .
number = digit { digit } .
```

| Construct | Notation | Meaning |
|-----------|----------|---------|
| Optional | `[ E ]` | Zero or one E |
| Repetition | `{ E }` | Zero or more E |
| Grouping | `( E )` | Group for alternation |
| Alternation | `\|` | E1 or E2 |
| Concatenation | juxtaposition | E1 followed by E2 |
| Terminator | `.` | End of production |

### ISO 14977 Extensions

| Construct | Notation | Meaning |
|-----------|----------|---------|
| Concatenation | `,` | Explicit separator |
| Terminator | `;` | End of production |
| Exception | `A - B` | A except B |
| Special sequence | `? text ?` | Implementation-defined |
| Comment | `(* text *)` | Non-normative |

### W3C EBNF (XML, XPath, SPARQL)

Used in W3C specifications. Closer to regular-expression syntax:

```
NameStartChar ::= ":" | [A-Z] | "_" | [a-z] | [#xC0-#xD6]
Name          ::= NameStartChar (NameChar)*
```

| Construct | Notation |
|-----------|----------|
| Optional | `E?` |
| Zero or more | `E*` |
| One or more | `E+` |
| Character class | `[a-z]` |
| Hex character | `#xNN` |
| Complement class | `[^a-z]` |

---

## J.4 Notation Variants in Practice

| Feature | ISO 14977 | W3C EBNF | Python `.gram` | LLVM LangRef | Rust Reference | Java LangSpec |
|---------|-----------|----------|----------------|--------------|----------------|---------------|
| Production separator | `=` | `::=` | `:` | `::=` | `:` | `:` |
| Terminator | `;` | (none) | (newline) | (newline) | (newline) | (none) |
| Alternation | `\|` | `\|` | `\|` | `\|` | `\|` | `\|` |
| Optional | `[E]` | `E?` | `E?` | `[E]` | `E?` | `[E]` |
| Zero or more | `{E}` | `E*` | `E*` | `{E}` | `E*` | `{E}` |
| One or more | `{E}-` (one+) | `E+` | `E+` | (none, use A {A}) | `E+` | — |
| Character class | (quoted range) | `[a-z]` | `[a-z]` | (none) | `[a-z]` | `[a-z]` |
| Comment | `(*...*) ` | (none) | `#...` | (none) | `//...` | `/*...*/` |

**Python grammar** (`Grammar/python.gram`) uses a PEG-based notation processed by `pegen`:
```peg
expr:
    | expr '+' term
    | term
term:
    | NAME
    | NUMBER
```

**LLVM LangRef** uses informal EBNF with `::=` and `{}` repetition, no formal standard:
```
funcdef   ::= OptionalLinkage ... 'define' ... '@' GlobalId '(' ArgList ')' ...
```

**Rust Reference** uses a variant with regex-style quantifiers and named character classes:
```
INTEGER_LITERAL: ( DEC_LITERAL | BIN_LITERAL | OCT_LITERAL | HEX_LITERAL )
DEC_LITERAL:     DEC_DIGIT (DEC_DIGIT | _)*
```

---

## J.5 PEG — Parsing Expression Grammars

Bryan Ford introduced PEG at POPL 2004. The critical distinction from CFG-based notations is **ordered choice**: `e1 / e2` tries `e1` first; if `e1` fails, backtracks and tries `e2`. Unlike BNF's `|` (which can be ambiguous), `/` is deterministic.

### PEG Operators

| Operator | Notation | Meaning |
|----------|----------|---------|
| Sequence | `e1 e2` | e1 followed by e2 |
| Ordered choice | `e1 / e2` | Try e1; if fails, try e2 |
| Optional | `e?` | Zero or one e (greedy) |
| Zero or more | `e*` | Zero or more e (greedy) |
| One or more | `e+` | One or more e (greedy) |
| Positive lookahead | `&e` | Succeeds if e would match; consumes nothing |
| Negative lookahead | `!e` | Succeeds if e would not match; consumes nothing |
| Any character | `.` | Matches any single character |

### Packrat Parsing

PEG grammars can be parsed in guaranteed O(n) time using **packrat parsing** (memoization of intermediate results). Each `(rule, position)` pair is computed at most once and cached. Space cost: O(n × rules). For large inputs and many rules, the memory overhead can be prohibitive.

### What PEG Cannot Express

PEG cannot express **inherently ambiguous grammars** (where the same string has multiple distinct parse trees). Since `e1 / e2` always chooses `e1` when both would match, there is no way to say "either of these two parse trees is valid." This is a feature for programming language grammars (no ambiguity by construction) and a limitation for natural language grammars.

### PEG Tools

| Tool | Language | Notable features |
|------|----------|-----------------|
| **Pest** (pest.rs) | Rust | `.pest` files; atomic/silent rules; generated `pest::iterators::Pairs` |
| **Peggy** (formerly PEG.js) | JavaScript | Semantic actions inline; ES module output |
| **Lark** | Python | PEG mode (`parser="earley"` or `"lalr"` also available); standalone mode |
| **Ohm** | JavaScript | Separated semantics (see §J.6); rule inheritance |
| **PEGTL** | C++ | Header-only; zero-overhead grammar-to-template mapping |

---

## J.6 Ohm

[Ohm](https://ohmjs.org/) (Alexander Warth, CDG Labs / Viewpoints Research) is a PEG-based grammar formalism that enforces a strict separation between grammar definition and semantic actions. The key insight: mixing semantic code into grammar rules makes grammars harder to read, reuse, and extend.

### Grammar Definition

```javascript
const myGrammar = ohm.grammar(`
  Arithmetic {
    Exp     = AddExp
    AddExp  = AddExp "+" MulExp  -- add
            | AddExp "-" MulExp  -- sub
            | MulExp
    MulExp  = MulExp "*" PriExp  -- mul
            | PriExp
    PriExp  = "(" Exp ")"        -- paren
            | "+" PriExp         -- pos
            | "-" PriExp         -- neg
            | ident
            | number
    ident   (an identifier) = letter alnum*
    number  (a number)      = digit* "." digit+  -- fract
                            | digit+             -- whole
  }
`);
```

### Semantics Objects

```javascript
const semantics = myGrammar.createSemantics();
semantics.addOperation('eval', {
  AddExp_add(left, _op, right) { return left.eval() + right.eval(); },
  AddExp_sub(left, _op, right) { return left.eval() - right.eval(); },
  number_fract(_int, _dot, frac) { return parseFloat(this.sourceString); },
  number_whole(_)               { return parseInt(this.sourceString); },
  // ...
});

const result = semantics(myGrammar.match("1 + 2 * 3")).eval();  // 7
```

### Lexical vs Syntactic Rules

Ohm distinguishes:
- **Lexical rules** (lowercase start): no automatic whitespace skipping; character-level patterns
- **Syntactic rules** (uppercase start): automatically skip spaces between elements

This avoids the scanner/parser split without losing readability.

### Rule Inheritance

Ohm grammars can extend each other:

```javascript
const json5Grammar = ohm.grammar(`
  JSON5 <: JSON {
    value += comment
    comment = "//" (~"\n" any)*
  }
`, { JSON: jsonGrammar });
```

Used for teaching (CS courses), DSLs, and as a prototyping tool where excellent error messages matter.

---

## J.7 ANTLR4 `.g4` Format

ANTLR4 (Parr et al., "The Definitive ANTLR 4 Reference") uses an LL(*) adaptive prediction algorithm that handles most practical programming language grammars without explicit disambiguation.

### Grammar Structure

```antlr
grammar Expr;

// Parser rules (lowercase)
prog    : stat+ EOF ;
stat    : expr NEWLINE           # printExpr
        | ID '=' expr NEWLINE    # assign
        | NEWLINE                # blank
        ;
expr    : expr op=('*'|'/') expr # MulDiv
        | expr op=('+'|'-') expr # AddSub
        | INT                    # int
        | ID                     # id
        | '(' expr ')'           # parens
        ;

// Lexer rules (uppercase)
ID      : [a-zA-Z]+ ;
INT     : [0-9]+ ;
NEWLINE : '\r'? '\n' ;
WS      : [ \t]+ -> skip ;
```

### Key Features

**Left-recursion support**: ANTLR4 automatically rewrites left-recursive rules into iterative form. `expr : expr '+' expr | INT` is valid.

**Semantic predicates**: `{condition}?` gates a rule alternative on a runtime condition. Useful for context-sensitive syntax:
```antlr
expr : {isType()}? ID '<' typeList '>'   # genericExpr
     | expr '+' expr                      # addExpr
     ;
```

**Lexer modes**: handle context-sensitive tokenisation (e.g., inside string interpolation `${...}` where `$` switches to expression mode):
```antlr
STRING : '"' -> pushMode(INTERP_MODE) ;
mode INTERP_MODE;
INTERP_EXPR_START : '${' -> pushMode(DEFAULT_MODE) ;
INTERP_CLOSE      : '}' -> popMode ;
```

**Target languages**: Java (default), C++, Python 3, Go, Swift, Dart, TypeScript. Runtime library required per target.

**Reference repository**: [github.com/antlr/grammars-v4](https://github.com/antlr/grammars-v4) — battle-tested grammars for 200+ languages.

---

## J.8 TreeSitter `grammar.js`

[TreeSitter](https://tree-sitter.github.io/) (Nathan Sobo, GitHub) generates incremental LR(1) parsers with robust error recovery. It is the de facto standard for editor tooling: Neovim tree-sitter integration, Helix, VS Code syntax highlighting, GitHub semantic code navigation, Sourcegraph.

### Grammar Definition

```javascript
// grammar.js for a simple language
module.exports = grammar({
  name: 'simple',

  rules: {
    source_file: $ => repeat($._statement),

    _statement: $ => choice(
      $.assignment,
      $.expression_statement,
    ),

    assignment: $ => seq(
      field('left',  $.identifier),
      '=',
      field('right', $._expression),
      ';'
    ),

    _expression: $ => choice(
      $.binary_expression,
      $.number,
      $.identifier,
    ),

    binary_expression: $ => prec.left(1, seq(
      field('left',  $._expression),
      field('operator', choice('+', '-', '*', '/')),
      field('right', $._expression),
    )),

    number:     _ => /[0-9]+/,
    identifier: _ => /[a-zA-Z_][a-zA-Z0-9_]*/,
  }
});
```

### Key Constructs

| Function | Meaning |
|----------|---------|
| `grammar({ rules: {...} })` | Define grammar |
| `seq(a, b, ...)` | Sequence |
| `choice(a, b, ...)` | Alternation (unordered, generates LR states) |
| `repeat(e)` / `repeat1(e)` | Zero or more / one or more |
| `optional(e)` | Zero or one |
| `prec(n, e)` / `prec.left(n, e)` / `prec.right(n, e)` | Precedence / associativity |
| `field('name', e)` | Named field in AST node |
| `token(e)` | Treat e as a single lexer token (no whitespace inside) |
| `alias(e, name)` | Rename node in AST |
| `$._rule` | Hidden rule (underscore prefix = not in AST) |

### External Scanners

For context-sensitive lexical constructs (Python indentation, Rust lifetimes, heredocs), an external scanner written in C handles the lexer state machine while tree-sitter handles the parser:

```c
// scanner.c (external scanner API)
bool tree_sitter_python_external_scanner_scan(
    void *payload, TSLexer *lexer, const bool *valid_symbols) {
    if (valid_symbols[INDENT] && /* current indent > stack top */) {
        lexer->result_symbol = INDENT;
        return true;
    }
    // ...
}
```

### Query Language

TreeSitter's `.scm` (S-expression) query language allows pattern matching on the AST:

```scheme
; Find all function definitions with their names
(function_definition
  name: (identifier) @function.name
  body: (block) @function.body)

; Find TODO comments
(comment) @comment
(#match? @comment "TODO")
```

Used by Neovim's `nvim-treesitter` for syntax highlighting, code folding, and text objects.

---

## J.9 Railroad / Syntax Diagrams

Railroad diagrams (also called syntax diagrams) are a visual notation for grammar rules. Arrows flow left to right; boxes represent nonterminals; rounded boxes or ovals represent terminals. Used in the [SQLite documentation](https://www.sqlite.org/syntax.html), [Rust Reference](https://doc.rust-lang.org/reference/), [PostgreSQL documentation](https://www.postgresql.org/docs/current/sql.html), and the original Pascal User Manual.

### Structure

```
          ┌──────────┐
─┬─ "if" ─┤expression├─ "then" ─┬─ statement ─┬─
 │        └──────────┘          │             │
 │                               └─ "else" ───┘
```

Sequential steps are connected by arrows on the main rail; alternatives split the rail into parallel paths (like parallel tracks); optional elements are represented as a bypass arc; repetitions loop back.

### Generation Tools

| Tool | Language | Input |
|------|----------|-------|
| [tabatkins/railroad-diagrams](https://github.com/tabatkins/railroad-diagrams) | JavaScript/Python | API calls |
| [Bottlecaps.de RR](https://www.bottlecaps.de/rr/ui) | Web app | EBNF text |
| [railroad-diagrams](https://pypi.org/project/railroad-diagrams/) | Python | API |
| [Rr](https://github.com/nicowillis/rr) | JavaScript | EBNF/PEG |

Railroad diagrams are **not machine-parseable** — they are a presentation format for human readers, not a grammar interchange format. Generate them from EBNF/PEG source, not the reverse.

---

## J.10 Earley Parsing and Marpa

For grammars that are inherently ambiguous, or where the grammar structure is not known until runtime, neither LL nor LR nor PEG parsers suffice. Earley parsing handles any context-free grammar.

### Earley's Algorithm

Jay Earley (1968) presented a recognizer that operates in O(n³) for ambiguous grammars, O(n²) for unambiguous, and O(n) for most practical unambiguous grammars (including all LR(k) grammars).

The algorithm maintains a chart of **Earley items** `(rule, position, origin)` meaning "we are at position `position` in `rule` and started matching at input position `origin`". The chart has `n+1` sets (one per input position).

Three operations:
1. **Predict**: for item `(A → α • B β, j)`, add `(B → • γ, i)` for all rules `B → γ`
2. **Scan**: for item `(A → α • a β, j)` and next token `a`, advance to `(A → α a • β, j)`
3. **Complete**: for item `(B → γ •, j)`, advance all `(A → α • B β, k)` in set `j`

### Marpa

[Marpa::R2](https://metacpan.org/pod/Marpa::R2) (Jeffrey Kegler) implements Earley parsing with:
- **Joop Leo optimization**: O(n) time for right-recursive grammars (common in practice)
- **Scanless parsing**: grammar-driven lexer avoids lexer/parser coupling
- **Ambiguity**: returns all parses as a compact forest

```perl
use Marpa::R2;
my $grammar = Marpa::R2::Scanless::G->new({
    source => \q{
        :default ::= action => [values]
        :start ::= Expression
        Expression ::= Term
        Expression ::= Expression '+' Term action => do_add
        Term ::= Factor
        Term ::= Term '*' Factor       action => do_mul
        Factor ::= Number
        Number ~ [\d]+
    }
});
```

Marpa is particularly useful for natural language processing, DSLs with ambiguous syntax-by-design, or incremental parsing of partial inputs.

---

## J.11 Notation Selection Guide

### Decision Table

| Property | BNF | Wirth EBNF | W3C EBNF | PEG/Pest | Ohm | ANTLR4 | TreeSitter | Railroad |
|----------|-----|------------|----------|----------|-----|--------|------------|----------|
| Ambiguity handling | Possible | Possible | Possible | None | None | Possible | Possible | Visual |
| Left-recursion | Yes | Yes | Yes | Needs elim. | Needs elim. | Auto-rewrites | Conflict hints | Visual |
| Scannerless | No | No | No | Yes | Yes | No | Yes | N/A |
| Operator precedence | Manual | Manual | Manual | Via `prec` | Via labels | `prec()` | `prec.*()` | N/A |
| Semantic actions | No | No | No | Inline | Separated | Inline | Via query `.scm` | N/A |
| Error recovery | None | None | None | None | Good (built-in) | Good | Excellent | N/A |
| Machine-parseable | Yes | Yes | Yes | Yes | Yes | Yes | Yes | No |
| Editor tooling target | No | No | No | No | No | No | **Yes** | No |
| Human-readable spec | **Best** | **Best** | Good | Good | Good | Medium | Poor | **Best** |

### Decision Flowchart

```
Is the primary goal a human-readable spec?
  ├── Yes, for formal publication → EBNF (Wirth or ISO) or Railroad diagrams
  └── No, need a parser

  Is the grammar for editor tooling (syntax highlighting, code folding, AST queries)?
    ├── Yes → TreeSitter grammar.js
    └── No

    Does the grammar have context-sensitive lexical structure (indentation, heredocs)?
      ├── Yes → TreeSitter (external scanner) or ANTLR4 (lexer modes)
      └── No

      Is the grammar inherently ambiguous, or unknown at compile time?
        ├── Yes → Earley/Marpa
        └── No

        Do you need complex semantic actions, multiple target languages, or left-recursion?
          ├── Yes → ANTLR4
          └── No

            Are you prototyping with excellent error messages as a priority?
              ├── Yes → Ohm
              └── No → Pest (Rust) / PEG.js (JavaScript) / Lark (Python)
```

---

## J.12 Converting Between Notations

### EBNF → BNF

Introduce fresh nonterminals for each metasyntactic construct:

```
(* EBNF *)
list = "[" [ item { "," item } ] "]"

(* BNF equivalent *)
list    ::= "[" opt_items "]"
opt_items ::= items | ε
items   ::= item more_items
more_items ::= "," item more_items | ε
```

### BNF → EBNF

Collapse right-recursive list rules:

```bnf
(* BNF: right-recursive list *)
items ::= item | items "," item

(* EBNF: *)
items = item { "," item }
```

### EBNF → PEG

Replace `|` with `/` (ordered choice), verify that the ordering does not change the language for your intended use case:

```
(* EBNF *)
expr = term | expr "+" term

(* PEG -- must eliminate left-recursion first *)
expr = term ("+" term)*
```

The key difference: `a | b` in EBNF may match either `a` or `b` (both valid, grammar is ambiguous); `a / b` in PEG always prefers `a`. For unambiguous grammars, the semantics are identical.

### Left-Recursion Elimination (for PEG and LL)

The standard transformation for left-associative operators:

```
(* Original: left-recursive *)
E ::= E "+" T | T

(* Step 1: identify the base case (T) and the recursive suffix ("+T") *)
(* Step 2: rewrite as base case followed by zero or more suffixes *)
E  ::= T E'
E' ::= "+" T E' | ε

(* Or in EBNF / PEG: *)
E = T ("+" T)*
```

### Right-Factoring (for LL)

LL parsers require that each alternative begins with a distinct first token:

```bnf
(* Ambiguous first token: *)
stmt ::= "if" expr "then" stmt
       | "if" expr "then" stmt "else" stmt

(* Right-factored: *)
stmt    ::= "if" expr "then" stmt opt_else
opt_else ::= "else" stmt | ε
```

### Common Pitfalls

**EBNF `[E]` vs PEG `E?`**: identical semantics — both match zero or one E. Safe to translate directly.

**EBNF `{E}` vs PEG `E*`**: identical semantics — both are zero or more. Safe to translate.

**EBNF `A | B` vs PEG `A / B`**: identical for unambiguous grammars. For ambiguous grammars (where both `A` and `B` can match the same input), PEG commits to the first successful match; EBNF may report an ambiguity error or silently pick one.

**Token vs rule precedence**: in ANTLR4, lexer rules take precedence over parser rules; longer matches win; explicit ordering resolves ties. In Pest, rules are tried in declaration order when using `choice`. In TreeSitter, `prec` values resolve conflicts in the generated LR table.

**Dangling else**: the classic ambiguity in `if-then-else`. EBNF allows both parses; PEG's ordered choice resolves it deterministically (match `else` greedily). ANTLR4 uses implicit rule ordering (earliest rule wins). TreeSitter requires explicit `prec` to tell the LR table which reduction to prefer.

---

## Quick Reference: Notation Symbols

| Concept | BNF | Wirth EBNF | ISO 14977 | W3C EBNF | PEG | ANTLR4 |
|---------|-----|------------|-----------|----------|-----|--------|
| Production | `::=` | `=` | `=` | `::=` | `=` | `:` |
| Terminator | (none) | `.` | `;` | (none) | (newline) | `;` |
| Alternation | `\|` | `\|` | `\|` | `\|` | `/` | `\|` |
| Concatenation | space | space | `,` | space | space | space |
| Optional | `[A]` rec. | `[A]` | `[A]` | `A?` | `A?` | `A?` |
| Zero or more | recursion | `{A}` | `{A}` | `A*` | `A*` | `A*` |
| One or more | recursion | — | — | `A+` | `A+` | `A+` |
| Grouping | — | `(A)` | `(A)` | `(A)` | `(A)` | `(A)` |
| Positive lookahead | — | — | — | — | `&A` | — |
| Negative lookahead | — | — | — | — | `!A` | `~A` |
| Any character | — | — | — | — | `.` | `.` |
| Character class | — | — | — | `[a-z]` | `[a-z]` | `[a-z]` |
| Terminal | `"t"` | `"t"` | `"t"` | `"t"` | `"t"` | `'t'` |
| Nonterminal | `<name>` | `name` | `name` | `name` | `name` | `Name` |
| Lexer rule | — | — | — | — | — | `NAME` (upper) |
| Comment | — | — | `(*...*) ` | (none) | `//` | `//` |

---

@copyright jreuben11
