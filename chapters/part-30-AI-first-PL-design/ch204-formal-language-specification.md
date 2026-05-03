# Chapter 204 — Formal Language Specification for AI-First PLs

*Part XXX — AI-First Programming Language Design*

The pseudo-code blocks in Chapters 205–207 invoke a common notation — effect rows like `<Stochastic, Gradient>`, security labels like `String{Untrusted}`, tensor types like `Tensor[Batch, SeqLen, DModel, Float32]`, and homoiconic operators like `` `(e) `` and `$(e)` — without a unified definition of their syntax or type-theoretic semantics. This chapter provides that definition. It specifies the unified AI-first PL across six layers: lexical structure, concrete syntax (EBNF), abstract syntax (AST), type rules (typing judgements), operational and denotational semantics, and an implementation plan mapping each layer to a compiler phase. The specification is prescriptive, not descriptive — it defines the language that Parts 1–7 of the AI-first PL vision require, drawing on Koka's row-polymorphic effects (Chapter 14 §14.3), Idris 2's Quantitative Type Theory (§203.5.3), Lean 4's dependent type kernel (Chapter 184 §184.1), Z3's SMT discharge of refinement predicates (Chapter 185 §185.4), and the Unison content-addressed identity model (§203.5.1). The implementation plan in §204.7 corresponds to Phase 0 through Phase 6 of the Chapter 207 build roadmap.

---

## 204.1 Specification Architecture

A six-layer stack is needed because the AI-first design pressures from Chapter 203 §203.1 impose requirements at every level of a language implementation:

- **Lexical level:** No indentation sensitivity (LLMs cannot reliably control whitespace); mandatory top-level type annotations enforced by grammar, not convention; canonical formatter with no configuration options.
- **Syntactic level:** Effect rows, security labels, tensor dimension variables, and linearity annotations must be syntactically distinct — the parser distinguishes them without semantic lookahead.
- **AST level:** Content-addressed identity (Unison model) — reformatting and renaming cannot change a definition's identity. Homoiconic quote/unquote constructors are AST nodes, not a separate reflection API.
- **Type system level:** Seven distinct type constructors address the seven AI-first concerns: `Tensor[...]` for named dimensions, `Dist[A]` for probabilistic types, `Code[T]` for homoiconic code, `T^q` for linear resource quantities, `T{L}` for information-flow labels, `{x:T|P}` for refinement types, and `Σ`/`Π` for dependent types.
- **Semantics level:** `{Stochastic}` has a measure-theoretic denotation; `{Gradient}` has a reverse-mode VJP denotation. Both are algebraic effects with handler-swappable implementations.
- **Implementation level:** The type checker (Layer 4) is the adversarial oracle that LLM code generation agents interact with — exposing it as an MCP tool is a first-order design decision, not an afterthought.

---

## 204.2 Layer 1 — Lexical Structure

The lexical design decisions are hardcoded at the token level to eliminate ambiguity that would prevent reliable LLM generation.

**No indentation sensitivity.** All structure is delimited by `{}`. Python-style significant indentation creates generation-time ambiguity that a semantic sampler cannot resolve because it requires tracking column offsets across arbitrary generation steps.

**Mandatory annotations by grammar.** The lexer produces a `REQUIRES_TYPE_ANN` token after every function name; the parser rejects definitions without annotations at parse time, not at a later semantic pass.

**Canonical formatter.** One fixed pretty-printer with no options — the `gofmt` discipline. LLM-generated code and human-edited code produce identical output after formatting. Code identity is therefore independent of formatting choices.

**Unicode operators with ASCII aliases.** Mathematical notation (`→`, `⊢`, `Σ`, `Π`) is first-class but has ASCII equivalents for codegen contexts.

```ebnf
(* Whitespace and comments *)
Whitespace   ::= (' ' | '\t' | '\n' | '\r')+
LineComment  ::= '--' Char* '\n'
BlockComment ::= '{-' (BlockComment | Char)* '-}'

(* Identifiers — naming conventions are grammar-enforced *)
Ident        ::= (Lower | '_') (Alnum | '_' | '\'')*   (* terms, functions *)
TypeIdent    ::= Upper Alnum*                           (* types, effects, modules *)
DimIdent     ::= Upper Alnum*                           (* tensor dimension variables *)

(* Literals *)
IntLit       ::= '-'? Digit+
FloatLit     ::= '-'? Digit+ '.' Digit+ ('e' '-'? Digit+)?
StringLit    ::= '"' (EscChar | [^"\\])* '"'
BoolLit      ::= 'true' | 'false'

(* Operators — all explicit, no precedence ambiguity *)
Arrow        ::= '->' | '→'
FatArrow     ::= '=>' | '⇒'
Turnstile    ::= '|-' | '⊢'
FlowLeq      ::= '<=' | '⊑'       (* IFT label ordering — Principle B §206.3 *)
Pipe         ::= '|'
Caret        ::= '^'               (* linearity annotation — Idris 2 QTT *)

(* Keywords — complete reserved set *)
Keywords     ::= 'fn' | 'let' | 'in' | 'type' | 'data' | 'effect' | 'handler'
               | 'with' | 'match' | 'case' | 'do' | 'return' | 'module' | 'import'
               | 'requires' | 'ensures' | 'invariant' | 'forall' | 'exists'
               | 'sample' | 'observe' | 'staged'
               | 'spec' | 'proof' | 'sorry' | 'decide' | 'native_decide'
               | 'Tensor' | 'Dist' | 'Code' | 'Σ' | 'Π'
               | 'Trusted' | 'Untrusted' | 'HighIntegrity' | 'Public' | 'Secret'
```

**Tokeniser note.** The lexer is maximal-munch with a reserved-word table. Effect rows `{Stochastic, IO}` tokenise as `LBRACE IDENT COMMA IDENT RBRACE` — effect row structure is grammatical, not lexical. Security labels `String{Untrusted}` tokenise as `TYPE_IDENT LBRACE IDENT RBRACE` — the grammar distinguishes label application from set literals by production context.

---

## 204.3 Layer 2 — Concrete Syntax (EBNF)

The concrete syntax makes every AI-first property syntactically manifest. Effect annotations, contracts, security labels, and linearity annotations are parsed by distinct grammatical productions, not by convention or comment parsing.

```ebnf
(* ══ Module structure ══════════════════════════════════════════ *)
Module      ::= 'module' TypeIdent '{' TopDecl* '}'
TopDecl     ::= FnDecl | TypeDecl | EffectDecl | HandlerDecl
              | SpecDecl | ImportDecl

ImportDecl  ::= 'import' TypeIdent ('.' TypeIdent)* ('{' Ident (',' Ident)* '}')?

(* ══ Top-level function — type annotation is MANDATORY ════════ *)
FnDecl      ::= 'fn' Ident TypeParams? '(' Params? ')' ':' Type EffectAnn?
                Contracts?
                ('{' Expr '}' | ';')     (* ';' = forward declaration *)

TypeParams  ::= '[' TypeParam (',' TypeParam)* ']'
TypeParam   ::= TypeIdent (':' Kind)?
Params      ::= Param (',' Param)*
Param       ::= Ident ':' Type

EffectAnn   ::= '<' EffectList '>'
EffectList  ::= Effect (',' Effect)*
Effect      ::= TypeIdent                              (* named effect *)

Contracts   ::= ContractClause*
ContractClause ::= ('requires' | 'ensures' | 'invariant') Expr

(* ══ Type declarations ════════════════════════════════════════ *)
TypeDecl    ::= 'type' TypeIdent TypeParams? '=' Type
              | 'data' TypeIdent TypeParams? '{' DataCon* '}'
DataCon     ::= TypeIdent ':' Type

(* ══ Effect and handler declarations ═════════════════════════ *)
EffectDecl  ::= 'effect' TypeIdent '{' EffectOp* '}'
EffectOp    ::= Ident ':' Type

HandlerDecl ::= 'handler' Ident 'for' TypeIdent '(' Params? ')' '{' HandlerCase+ '}'
HandlerCase ::= Ident '(' Params? ',' 'resume' ':' Type ')' '=>' Expr
              | 'return' '(' Ident ':' Type ')' '=>' Expr

(* ══ Spec declarations — dual-representation specs (§206.4) ══ *)
SpecDecl    ::= 'spec' Ident TypeParams? '{' SpecClause* '}'
SpecClause  ::= ('intent' ':' StringLit)               (* natural language — human layer *)
              | ('formal' ':' Type)                    (* machine-checkable type *)
              | ('requires' ':' Expr)
              | ('ensures' ':' Expr)
              | ('accepts' ':' Expr)                   (* test oracle *)

(* ══ Types ════════════════════════════════════════════════════ *)
Type        ::= BaseType | TypeApp | FuncType | EffectfulType
              | DepFuncType | TensorType | RefinedType
              | SigmaType | PiType | DistType | CodeType
              | LinearType | SecurityType | '(' Type ')'

BaseType    ::= 'Int' | 'Nat' | 'Float' | 'Bool' | 'String' | 'Unit' | TypeIdent
TypeApp     ::= TypeIdent '[' Type (',' Type)* ']'
FuncType    ::= Type Arrow Type

(* Effectful function: T₁ →<E₁,E₂> T₂ — Koka row syntax *)
EffectfulType ::= Type Arrow '<' EffectList '>' Type

(* Dependent function: (x : T) → U — where U may mention x *)
DepFuncType ::= '(' Ident ':' Type ')' Arrow Type

(* Tensor with named dimension variables — Chapter 205 §205.3.1 *)
TensorType  ::= 'Tensor' '[' DimList ',' BaseType ']'
DimList     ::= DimExpr (',' DimExpr)*
DimExpr     ::= DimIdent | IntLit
              | DimExpr ('*' | '+' | '/') DimExpr | '(' DimExpr ')'

(* Refinement: { x : T | P(x) } — P discharged by Z3 (Ch185 §185.4) *)
RefinedType ::= '{' Ident ':' Type '|' Expr '}'

(* Dependent Σ and Π types — Lean 4 kernel (Ch184 §184.1) *)
SigmaType   ::= ('Σ' | 'Sigma') '(' Ident ':' Type ',' Expr ')'
PiType      ::= ('Π' | 'Pi')   '(' Ident ':' Type ',' Type ')'

(* Probabilistic — PPL Bridge §203.7 *)
DistType    ::= 'Dist' '[' Type ']'

(* Homoiconic: a quoted, typed expression — code as data *)
CodeType    ::= 'Code' '[' Type ']'

(* Linearity annotation: T^1 = linear, T^ω = unrestricted — Idris 2 QTT *)
LinearType  ::= Type Caret ('1' | 'ω' | Ident)

(* Security label: T{L} — Jif Principle B (§206.3) *)
SecurityType  ::= Type '{' SecurityLabel '}'
SecurityLabel ::= 'Trusted' | 'Untrusted' | 'HighIntegrity' | 'Public' | 'Secret' | Ident

(* ══ Expressions ══════════════════════════════════════════════ *)
Expr        ::= Lit | Var | App | Lam | LetExpr | MatchExpr | DoExpr
              | EffectInvoke | HandlerExpr
              | SampleExpr | ObserveExpr
              | QuoteExpr | UnquoteExpr | StagedExpr
              | SigmaIntro | SigmaElim
              | Ascription | BinOp | IfExpr | '(' Expr ')'

App         ::= Expr '(' (Expr (',' Expr)*)? ')'
Lam         ::= 'fn' '(' Params? ')' FatArrow Expr
              | 'fn' '(' Params? ')' '{' Expr '}'
LetExpr     ::= 'let' Ident (':' Type)? '=' Expr (';' | 'in') Expr

MatchExpr   ::= 'match' Expr '{' MatchArm+ '}'
MatchArm    ::= 'case' Pattern Arrow Expr

(* Monadic do-notation for sequential effect composition *)
DoExpr      ::= 'do' '{' DoStmt+ '}'
DoStmt      ::= (Ident '<-')? Expr ';'
              | 'let' Ident '=' Expr ';'

EffectInvoke ::= '!' Ident '(' (Expr (',' Expr)*)? ')'  (* ! prefix *)
HandlerExpr  ::= 'with' Expr '{' Expr '}'

(* Probabilistic — PPL Bridge Paradigms 1 and 2 (§203.7) *)
SampleExpr   ::= 'sample' '(' Expr ')'
ObserveExpr  ::= 'observe' '(' Expr ',' Expr ')'

(* Homoiconic — rho-calculus and Lean 4 quote/unquote (Ch207 §207.1.2) *)
QuoteExpr    ::= '`(' Expr ')'
UnquoteExpr  ::= '$(' Expr ')'

(* Staged computation — Julia @generated paradigm (Ch207 §207.1.3) *)
StagedExpr   ::= 'staged' '{' Expr '}'

(* Sigma type intro and elim *)
SigmaIntro   ::= '⟨' Expr ',' Expr '⟩'
SigmaElim    ::= 'let' '⟨' Ident ',' Ident '⟩' '=' Expr 'in' Expr

Ascription   ::= Expr ':' Type
BinOp        ::= Expr Op Expr
IfExpr       ::= 'if' Expr '{' Expr '}' 'else' '{' Expr '}'

Pattern     ::= Ident | Lit
              | TypeIdent '(' (Pattern (',' Pattern)*)? ')'  (* constructor *)
              | '⟨' Pattern ',' Pattern '⟩'                  (* sigma pair *)
              | '_'                                           (* wildcard *)
```

---

## 204.4 Layer 3 — Abstract Syntax (AST)

The AST is the canonical internal representation. **Identity is the SHA-256 hash of the normalised, α-reduced AST** — the Unison model (§203.5.1) applied to the full unified PL. Reformatting, renaming, and import reordering cannot change the identity of a definition. This enables content-addressed dependency graphs and reproducible compilation.

```pseudo
(* Core AST — typed algebraic representation *)

data Expr
  = Lit    Literal
  | Var    Name
  | App    Expr Expr
  | Lam    Name Type Expr
  | Let    Name Type Expr Expr
  | Match  Expr (List Arm)

  (* Effects — Koka handler model *)
  | Invoke EffectName (List Expr)           -- !op(args)
  | Handle Expr Handler                     -- with h { e }
  | Do     (List DoStmt)                    -- monadic do

  (* Probabilistic — PPL Bridge Paradigms 1 and 2 *)
  | Sample  Expr                            -- sample(dist : Dist[A]) : A
  | Observe Expr Expr                       -- observe(dist, value)

  (* Homoiconic — rho-calculus / Lean 4 MetaM *)
  | Quote   Expr                            -- `(e) : Code[T]
  | Unquote Expr                            -- $(e) : T
  | Staged  Expr                            -- compile-time evaluation

  (* Dependent types — Lean 4 kernel *)
  | SigmaIntro Expr Expr                    -- ⟨value, proof⟩
  | SigmaElim  Name Name Expr Expr          -- let ⟨x, p⟩ = e in e'

  (* Proof terms — Lean 4 sorry/decide *)
  | Sorry                                   -- deferred obligation (open G2)
  | Decide  Expr                            -- proof by SMT computation
  | Proof   ProofTerm                       -- explicit proof term


data Type
  = TBase   BaseType
  | TApp    TypeName (List Type)
  | TFun    Type EffectRow Type             -- T₁ →<E> T₂
  | TDepFun Name Type Type                  -- (x : T) → U(x)
  | TTensor (List DimExpr) BaseType         -- Tensor[D₁..Dₙ, Base]
  | TRefine Name Type Expr                  -- {x : T | P}
  | TSigma  Name Type Expr                  -- Σ(x : T, P(x))
  | TPi     Name Type Type                  -- Π(x : T, U(x))
  | TDist   Type                            -- Dist[A]
  | TCode   Type                            -- Code[T]
  | TLinear Type Quantity                   -- T^q  (Idris 2 QTT)
  | TLabel  Type SecurityLabel             -- T{L} (Jif IFT)


data EffectRow = EffectRow (Set Effect)    -- {Stochastic, IO, Gradient, ...}

data Quantity  = Linear | Affine | Unrestricted   -- from QTT

(* Content-addressed identity — Unison model *)
type ASTHash = SHA256Bytes
identity : Expr → ASTHash
identity = sha256 ∘ serialise ∘ alpha_normalise

(* Homoiconicity: Code[T] wraps an Expr node — code IS data *)
(* An AI agent generating Code[T] manipulates Expr values directly *)
(* No separate reflection API — the type system IS the reflection system *)
```

Four properties are critical for AI generation:
- Every `Lam` node carries an explicit `Type` — the grammar enforces this so no LLM-generated AST lacks type information at any node.
- `EffectRow` is a `Set`, not a list — order-independent composition is structural, not derived from a separate commutativity proof.
- `Sorry` is a first-class AST node, enabling incremental generation with open obligations. An LLM can generate a partial proof and mark deferred obligations explicitly; the verifier counts open `Sorry` nodes as unverified obligations, not errors.
- `Quote`/`Unquote` constructors make the homoiconic interface explicit in the type of the AST itself — reflection is not a separate meta-API but a first-class constructor.

---

## 204.5 Layer 4 — Type Rules

The typing judgement form is:

```text
Γ ; Φ ⊢ e : T ! E
```

where `Γ` is the term environment, `Φ` is the proof context (dischargeable by Z3 or Lean 4 kernel), `T` is the value type, and `E` is the effect row. The `!` separates value type from computational effects, following Koka's separation convention (Chapter 14 §14.3).

```text
(* ── Variables ────────────────────────────────────────────────── *)

        Γ(x) = T
─────────────────────────────    (Var)
  Γ ; Φ ⊢ x : T ! {}


(* ── Function abstraction and application ─────────────────────── *)

    Γ, x:T₁ ; Φ ⊢ e : T₂ ! E
──────────────────────────────────────────    (Lam)
  Γ ; Φ ⊢ fn(x:T₁) => e : T₁ →<E> T₂ ! {}


  Γ ; Φ ⊢ f : T₁ →<E> T₂     Γ ; Φ ⊢ a : T₁
─────────────────────────────────────────────────    (App)
         Γ ; Φ ⊢ f(a) : T₂ ! E


(* ── Effects — Koka row model ────────────────────────────────── *)

      op : T₁ → T₂  ∈  effect Eff
─────────────────────────────────────────────    (Invoke)
  Γ ; Φ ⊢ !op(v) : T₂ ! {Eff}


  Γ ; Φ ⊢ e : T ! (E₁ ∪ E₂)     h handles E₁
──────────────────────────────────────────────────────    (Handle)
       Γ ; Φ ⊢ with h { e } : T ! E₂
  (* handler removes handled effects from the row *)


(* ── Probabilistic — PPL Bridge (§203.7) ─────────────────────── *)

       Γ ; Φ ⊢ d : Dist[A]
───────────────────────────────────────────────    (Sample)
  Γ ; Φ ⊢ sample(d) : A ! {Stochastic}


  Γ ; Φ ⊢ d : Dist[A]     Γ ; Φ ⊢ v : A
──────────────────────────────────────────────────    (Observe)
  Γ ; Φ ⊢ observe(d, v) : Unit ! {Conditioning}


(* ── Tensor types — shape constraints via Z3 (Ch185 §185.4) ───── *)

  Γ ; Φ ⊢ e : Array[D₁×…×Dₙ, Base]
  Z3 ⊢ (shape(e) = [D₁,…,Dₙ])  under Φ
────────────────────────────────────────────────    (Tensor-Intro)
  Γ ; Φ ⊢ e : Tensor[D₁,…,Dₙ, Base] ! {}


  Γ ; Φ ⊢ a : Tensor[B,M,K, F]
  Γ ; Φ ⊢ b : Tensor[B,K,N, F]
  Z3 ⊢ (B > 0 ∧ M > 0 ∧ N > 0 ∧ K > 0)
────────────────────────────────────────────────    (Matmul)
  Γ ; Φ ⊢ matmul(a,b) : Tensor[B,M,N, F] ! {}


(* ── Refinement types — Z3-discharged invariants ─────────────── *)

  Γ ; Φ ⊢ e : T     Z3 ⊢ P(e) under Φ
──────────────────────────────────────────────────    (Refine-Intro)
  Γ ; Φ ⊢ e : {x:T | P(x)} ! {}


  Γ ; Φ ⊢ e : {x:T | P(x)}
──────────────────────────────────────────────────    (Refine-Elim)
  Γ ; Φ, P(e) ⊢ e : T ! {}     (* P flows into proof context *)


(* ── Dependent types — Lean 4 kernel (Ch184 §184.1) ──────────── *)

  Γ, x:T ; Φ ⊢ body : U(x)
──────────────────────────────────────────────────    (Π-Intro)
  Γ ; Φ ⊢ fn(x:T) => body : Π(x:T, U(x)) ! {}


  Γ ; Φ ⊢ a : T     Γ ; Φ ⊢ p : P(a)
──────────────────────────────────────────────────    (Σ-Intro)
  Γ ; Φ ⊢ ⟨a, p⟩ : Σ(x:T, P(x)) ! {}

  (* The patch signature from Ch207 §207.1 is well-typed in this system: *)
  (* patch : Π(P: Program, M: MetaModel P) → Σ(Q: Program, Q satisfies spec P) *)


(* ── Linear types — Idris 2 QTT (§203.5.3) ───────────────────── *)

  Γ, x:^1 T ; Φ ⊢ e : U     x ∈ fv(e) (exactly once)
──────────────────────────────────────────────────────────────    (Linear-Use)
  Γ ; Φ ⊢ let x:^1 = v in e : U ! {}

  (* Budget[N] in Ch206 §206.1 is T^1 — consumed, not copied *)
  (* iso in Pony is T^1 — matches the capability safety linear resource model *)


(* ── Information-flow labels — Jif Principle B (Ch206 §206.3) ── *)

  Γ ; Φ ⊢ e : T{L₁}     L₁ ⊑ L₂
──────────────────────────────────────────────────    (IFT-Widen)
  Γ ; Φ ⊢ e : T{L₂} ! {IFT}

  (* Principle B: Untrusted ⋢ Trusted — flow is REJECTED without sanitise *)
  (* sanitise : String{Untrusted} →<IFT> String{Trusted} — requires handler *)


(* ── Homoiconic quote/unquote — capability-checked (Ch207 §207.1) *)

      Γ ; Φ ⊢ e : T
─────────────────────────────────────────    (Quote)
  Γ ; Φ ⊢ `(e) : Code[T] ! {}


  Γ ; Φ ⊢ e : Code[T]
  caps(T) ⊆ Γ.capabilities           (* Principle A: no ambient authority *)
──────────────────────────────────────────────────────    (Unquote)
  Γ ; Φ ⊢ $(e) : T ! caps(T)
  (* executing quoted code requires holding its capability requirements *)


(* ── Inference strategy summary — G2 open problem (Ch207 §207.2) *)
(*   INFER automatically:                                           *)
(*     tensor shape constraints (Z3 linear arithmetic)            *)
(*     effect rows (Koka row-polymorphic inference)                *)
(*     capability requirements (Pony ref-cap inference from usage) *)
(*   REQUIRE explicit annotation:                                  *)
(*     spec postconditions (ensures/requires)                      *)
(*     convergence bounds (PPL Paradigm 5, §203.7.5)              *)
(*     security labels (IFT — intent cannot be derived from usage) *)
```

---

## 204.6 Layer 5 — Semantics

### 204.6.1 Operational Semantics (Small-Step)

`e → e'` is a call-by-value small-step reduction. `E[_]` denotes an evaluation context.

```text
(* ── β-reductions ─────────────────────────────────────────────── *)

  (fn(x:T) => e)(v)              →  e[v/x]                  (β)
  let x = v; e                   →  e[v/x]                  (ζ)
  let ⟨x, p⟩ = ⟨v, proof⟩ in e  →  e[v/x, proof/p]         (Σ-β)


(* ── Effect handler reductions ────────────────────────────────── *)

  (* invoke inside a handler for the same effect *)
  with h { E[!op(v)] }
    →  h.op(v, fn(r) => with h { E[r] })                    (Handle-Op)

  (* value reaches the handler — return case *)
  with h { v }  →  h.return(v)                               (Handle-Return)


(* ── Probabilistic — handler-swappable (PPL Paradigm 2) ───────── *)

  with Deterministic(seed) { sample(d) }  →  d.point_sample(seed)   (Det)
  with MC(n)               { sample(d) }  →  d.mcmc_sample(n)        (MC)
  with VI(guide)           { e }          →  vi_approximate(e, guide) (VI)


(* ── Homoiconic ───────────────────────────────────────────────── *)

  $(`(e))        →  e                                        (Quote-Eval)
  staged { v }   →  v                                        (Stage)
  (* staged reduces at compile time — result substitutes into the AST *)


(* ── Linear ───────────────────────────────────────────────────── *)

  let x:^1 = v in e[x]  →  e[v/x]                           (Linear-β)
  (* x used exactly once — statically checked, dynamically unremarkable *)


(* ── Proof terms ──────────────────────────────────────────────── *)

  decide(P)         →  proof_term  if Z3 ⊢ P                 (Decide)
  native_decide(P)  →  proof_term  if compile_and_run(P) = true  (Native)
```

### 204.6.2 Denotational Semantics for `{Stochastic}` and `{Gradient}`

These two effects require measure-theoretic and calculus-theoretic denotations beyond simple operational rules.

```text
(* {Stochastic} — probability kernel semantics *)

⟦ e : A ! {Stochastic} ⟧ : Input → Meas(A)
  (* e denotes a measurable function to a probability measure over A *)

⟦ sample(d) ⟧(ω)                     = d(ω)
⟦ observe(d, v) ⟧                    = conditioning update
⟦ with Deterministic(s) { e } ⟧      = δ(⟦e⟧(s))     (* Dirac delta *)
⟦ with HMC(n)            { e } ⟧     = MCMC_approx(⟦e⟧, n)
⟦ with VI(q)             { e } ⟧     = VI_approx(⟦e⟧, q)


(* {Gradient} — reverse-mode AD semantics *)

⟦ e : (A →<Gradient> B) ⟧ : A → (B × (B → A))
  (* e denotes a differentiable function AND its compiler-derived VJP *)

⟦ grad(f) ⟧(x) = (f(x), vjp(f, x))
  (* differentiating a non-{Gradient} function is a type error *)
  (* the {Gradient} effect propagates through the call graph    *)
```

### 204.6.3 Type Soundness

```text
Theorem (Progress)
  If ⊢ e : T ! E and e is not a value,
  then either (a) ∃ e'. e → e', or
  (b) e = E[!op(v)] for some unhandled op ∈ E.
  Case (b) is stuck only if E is non-empty — a type error to evaluate
  a term with unhandled effects in a total context.

Theorem (Preservation)
  If Γ ; Φ ⊢ e : T ! E and e → e',
  then Γ ; Φ ⊢ e' : T ! E.

Corollary (Effect Safety)
  If ⊢ e : T ! {}, evaluation of e invokes no effects.
  Determinism, purity, and lack of I/O are compiler-certified.

Corollary (Stochastic Safety)
  If ⊢ e : T ! {Stochastic}, then ⟦e⟧ is a well-formed probability measure over T.

Corollary (IFT Non-Leakage — Principle B)
  No well-typed program flows String{Untrusted} to String{Trusted}
  without passing through an explicit sanitise handler.
  Prompt injection is structurally impossible in well-typed programs.

Corollary (Linear Conservation)
  If ⊢ let x:^1 = v in e : U, then x occurs in e exactly once.
  Token budgets (Budget[N]:^1) and GPU tensor ownership cannot be aliased.

Corollary (Tensor Shape Safety)
  If ⊢ matmul(a, b) : Tensor[B,M,N, F] ! {}, the Z3 solver has
  discharged all shape constraints at compile time.
  Shape mismatches are type errors, not runtime exceptions.

Corollary (Capability Safety — Principle A)
  If ⊢ $(e) : T ! caps(T) and caps(T) ⊈ Γ.capabilities,
  the Unquote rule does not apply — the program is rejected.
  Executing quoted code without holding its capabilities is a type error.
```

**The three hardest type-system interactions** (Phase 0 deliverable — must be proved before production implementation begins):

1. `{Gradient}` effect inside a `{Stochastic}` context — differentiating a stochastic function. Requires the denotational semantics to compose probability measures with VJPs (a differentiable probability kernel).
2. `Σ`-types (proof obligations) inside `{IFT}`-labelled computations — security labels on proof terms. Requires the label lattice to be compatible with the dependent type universe hierarchy.
3. `T^1` (linear) resources passed to a handler's continuation `resume : T → U`. The `resume` continuation must consume the linear resource exactly once — a constraint on handler definition that is not captured by the standard algebraic effects theory and requires a linear handler calculus extension.

---

## 204.7 Layer 6 — Implementation Plan

Each layer maps to a concrete compiler phase. The Chapter 207 §207.3 build roadmap covers the transformer development tier (Phases 0–5); this table covers the full language stack.

| Phase | Duration | Team | Layers | Deliverable |
|---|---|---|---|---|
| **0 — Reference interpreter** | 3–6 mo | 2–3 | 1–5 | Haskell/Agda for theory validation — prove the three hardest type interactions; write soundness argument. Rust reference impl for Phase 1 pipeline. Both target this spec; Haskell/Agda is the proof vehicle, Rust is the build vehicle. |
| **1 — Production lexer/parser** | 3–6 mo | 2–3 | 1–2 | Rust: `logos` lexer ([Ch182](../part-26-ecosystem-frontiers/ch182-language-tooling-parsers-lexers-syntax-trees.md) §182.2), `chumsky` PEG parser ([Ch182](../part-26-ecosystem-frontiers/ch182-language-tooling-parsers-lexers-syntax-trees.md) §182.4), typed AST (Layer 3). Canonical formatter. Structured JSON parse errors with span and expected-token trace. |
| **2 — Type checker + SMT** | 12–18 mo | 4–5 | 3–4 | Full Layer 4 typing rules. Z3 backend ([Ch185](../part-27-mathematical-foundations/ch185-mathematical-logic-model-theory.md) §185.4) for refinement types and tensor shape arithmetic. Lean 4-style elaborator for `Σ`/`Π` types ([Ch184](../part-27-mathematical-foundations/ch184-proof-assistant-internals-lean4-coq-isabelle.md) §184.1). IFT label checker. Linear type checker (usage counting via QTT). Expose type checker as MCP tool from day one. |
| **3 — MLIR backend** | 12–18 mo | 4–5 | 6 | AST → MLIR ([Ch179](../part-26-ecosystem-frontiers/ch179-llvm-mlir-for-ai.md)). `tensor_named` dialect (named dims through lowering). `effects` dialect (PRNGKey threading for `{Stochastic}`, VJP insertion for `{Gradient}`). Lowering: `tensor_named` → `linalg` → StableHLO → hardware ([Ch205](ch205-transformer-model-development-pls.md) §205.5). |
| **4 — Parallelism types** | 18–24 mo | 3–4 | 4+6 | Shard types. Data/tensor parallelism annotations. Collective communication insertion. Interaction between `{Gradient}` effect and shard types (hardest compiler interaction in the entire Phase 0–5 roadmap). |
| **5 — Standard library + tooling** | 12 mo | 3–4 | 6 | MCP server: type checker, shape inferencer, dependency graph as queryable AI tools. ONNX import bridge. Standard kernels via agentic generation ([Ch205](ch205-transformer-model-development-pls.md) §205.6). Benchmark suite. |
| **6+ — Multi-agent + SDLC** | Research | 5+ | 4–5 | Probabilistic session types ([Ch206](ch206-multi-agent-pls-sdlc-security.md) §206.1) — new type theory required. Type-context-delta VCS ([Ch206](ch206-multi-agent-pls-sdlc-security.md) §206.2) — dependent types in the VCS substrate. Certified `patch` ([Ch207](ch207-reflective-code-open-problems-roadmap.md) §207.1) — `Σ`-type-returning self-modification. |

**Critical path.** Phases 0–2 are purely language-level and can proceed in parallel with MLIR exploration. The type checker (Phase 2) is the dependency for everything downstream — it is the adversarial oracle that AlphaProof and LeanCopilot demonstrate is the key AI-agent interface (Chapter 207 §207.1.1). Expose it as an MCP tool from Phase 2 day one.

**Phase 0 gate.** The reference interpreter must handle all three hardest type interactions before Phase 1 begins. Swift for TensorFlow skipped this gate and accumulated type-system inconsistencies that could not be fixed without breaking API compatibility. The soundness argument is not optional.

---

## Chapter 204 Summary

- The unified AI-first PL requires a six-layer specification because each AI-first design pressure from Chapter 203 §203.1 imposes requirements at a different level of language implementation.
- Layer 1 (lexical): no indentation sensitivity, mandatory type annotations enforced by grammar, Unicode operators with ASCII aliases, canonical formatter with no options.
- Layer 2 (concrete syntax): effect rows `<E₁,...,Eₙ>`, security labels `T{L}`, tensor dimension variables `DimIdent`, linearity annotations `T^q`, and spec contracts `requires`/`ensures` are syntactically distinct productions — not convention.
- Layer 3 (AST): content-addressed by SHA-256 of normalised AST (Unison model); `Quote`/`Unquote` constructors are first-class AST nodes; `Sorry` is a first-class deferred obligation, not a hack.
- Layer 4 (type rules): seven type constructors address the seven AI-first concerns; the judgement form `Γ ; Φ ⊢ e : T ! E` separates value type from effect row following Koka's convention; inference strategy (G2) infers shapes/effects/capabilities, requires annotation for specs/labels/convergence.
- Layer 5 (semantics): `{Stochastic}` denotes a probability kernel (measure-theoretic); `{Gradient}` denotes a function paired with its compiler-derived VJP; six type soundness corollaries guarantee effect safety, stochastic safety, prompt-injection non-leakage, linear conservation, tensor shape safety, and capability safety.
- The three hardest type-system interactions — differentiating a stochastic function, security-labelling proof terms, and linear resources in handler continuations — must be proved correct in Phase 0 before production implementation begins.
- Layer 6 (implementation plan): seven phases from reference interpreter through multi-agent type theory; the type checker (Phase 2) is the adversarial oracle and must be exposed as an MCP tool from day one.
