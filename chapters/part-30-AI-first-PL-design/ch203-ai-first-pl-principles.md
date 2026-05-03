# Chapter 203 — AI-First Programming Language Design: Principles and Landscape

*Part XXX — AI-First Programming Language Design*

Human programming languages optimise for *writing and reading by people*. When an LLM generates code, this optimisation target inverts: the author is a machine that does not tire, has no working-memory limit, and can be guided token by token by a semantic sampler. This chapter establishes the seven design pressures that an AI-first programming language must satisfy, surveys the nine-tier capability maturity matrix for a unified system spanning general-purpose coding through transformer development and multi-agent coordination, and catalogues the languages — purpose-built and co-opted — that implement identifiable subsets of those properties today. The probabilistic programming tradition (Stan, Pyro/NumPyro, Gen) contributes five foundational paradigms that ground stochastic AI behaviour in a formal type system. WebAssembly emerges as the natural deployment substrate that makes capability safety and hardware portability compiler outputs rather than runtime add-ons. Subsequent chapters treat the formal language specification (Chapter 204), transformer model development PLs (Chapter 205), multi-agent coordination and SDLC (Chapter 206), and reflective self-improving code with a build roadmap (Chapter 207).

---

## 203.1 The AI-First Inversion

Seven design pressures distinguish an AI-first PL from a human-first one. Each inverts a property that human languages treat as a virtue.

**Formal semantics over convention.** Human PLs accumulate idioms: Python's `__dunder__` protocol, C's undefined behaviour that compilers exploit, JavaScript's coercive equality. Idioms work because human readers apply background knowledge to resolve ambiguity. An LLM generating code must either be trained on those idioms at enormous cost or be given a language where the semantics are fully determined by syntax alone. An AI-first PL has no "works by convention" behaviour — every construct has a formal operational rule (see Chapter 204 §204.5).

**Rich type systems as machine-readable specs.** Dependent types, refinement types, and algebraic effect rows are not academic luxuries in an AI-first context — they are the primary channel through which the programmer communicates intent to the generating agent. A postcondition `ensures result >= 0` is a specification that the LLM can generate against and the verifier can check. A bare `int` return type communicates nothing. The type system is the spec language.

**Explicit effects.** A function whose type is `A → B` might read a file, launch a network request, mutate global state, or sample a random number — nothing in its type says which. An AI generating call sites cannot reason about safety or correctness without effect information. Row-polymorphic algebraic effects (§203.5, Koka) give each function an effect signature as part of its type: `A →<IO, Stochastic> B` is statically different from `A → B`. The generating agent always has full effect context.

**Homoiconicity and AST-as-data.** Code generation requires manipulating code. If code is text, the agent must parse it; if code is a typed AST value, the agent operates on it directly. Lean 4's `quote`/`unquote` operators (Chapter 184 §184.1.6), Pel's Lisp-style AST structure, and Unison's content-addressed codebase all instantiate this property in different ways. Homoiconic languages make program transformation a typed function, not a string operation.

**Canonical single representation.** LLM-generated code and human-edited code must produce identical output after formatting — no diff noise from style variation. Go's `gofmt` demonstrated that a canonical formatter with no options eliminates an entire class of code review friction. MoonBit enforces this at the language level (§203.3). An AI-first PL embeds the formatter in the compiler toolchain.

**Structured typed diagnostics.** Human compilers produce prose error messages optimised for human reading: "expected `;` before `}`". An AI author needs structured, machine-parseable diagnostics: `TypeMismatch { expected: Tensor[Batch, DModel], got: Tensor[DModel, Batch], location: ... }`. The diagnostic is itself a typed value the agent can pattern-match on and use to drive repair.

**Edit locality with bounded blast radius.** Module boundary guarantees — that a change inside a module cannot change the observable type of its exported API — let an agent edit code with a predictable scope of effect. Unison's content-addressing makes this structural: a function's hash encodes its semantics, and dependent functions reference the hash, not the name. An AI can rename or refactor inside a module without triggering downstream recompilation cascades.

**Property specs as first-class syntax.** Preconditions, postconditions, loop invariants, and termination witnesses are not comments or assertions — they are syntactically distinct elements of the function declaration, checked by the compiler. Dafny's `requires`/`ensures` (§203.4) and Verus's attribute macros are the reference implementations. An agent generating a function generates its spec simultaneously; both are checked at the same time.

### 203.1.1 The Type Inference Strategy (G2)

A first-order design decision for any AI-first PL is where to draw the line between inference and mandatory annotation. Full Hindley-Milner inference hides type context from the agent; full explicit annotation is verbose but maximally informative.

The right split: **infer** shapes (tensor dimensions, derivable from construction sites), **infer** effects (row inference — what effects a function has, without annotation, exactly as Koka does), and **infer** capabilities (Pony-style reference capability inference from usage pattern). But **require explicit annotation** for specifications and invariants (Dafny postconditions — these encode *intent*, not derivable from usage), convergence bounds (probabilistic paradigm 5, §203.7), and security labels (information-flow labels, §206.3).

The intuition: inference is appropriate where the compiler has complete information and uniquely determines the type from usage. Annotation is required where the intent is the spec, not recoverable from usage alone. A function that returns a non-negative integer has `ensures result >= 0` as intent; no inference algorithm derives that from the implementation. This is the G2 open problem — no published treatment in the AI-first PL literature addresses it fully. Chapter 207 §207.2 gives the complete formulation.

---

## 203.2 Capability Tier Maturity Matrix

The unified AI-first PL is a stack of nine capability tiers. Each tier is assessed for theory maturity and implementation maturity on a scale of 1 (open research) to 5 (production). The prototype horizon estimates when a credible research prototype at that tier becomes achievable.

| Tier | Theory maturity | Impl maturity | Nearest existing system | Prototype horizon |
|---|---|---|---|---|
| Coding agent PL (Parts 1–2) | 4 | 3 | MoonBit (semantic sampler), Dafny (82% vericoding) | 1–2 yrs |
| Algebraic effects (Part 3 — Koka/Unison) | 5 | 3 | Koka (row-polymorphic effects), Unison (content-addressing) | 1–2 yrs |
| LLM interaction layer (Part 4) | 3 | 5 | Outlines (FSM sampling), DSPy (typed I/O signatures) | Now |
| Transformer dev PL (Part 5) | 3 | 3 | Mojo (MLIR backend) + Dex (type theory) — not yet unified | 2–3 yrs |
| Multi-agent coordination (Part 6) | 1 | 1 | Choral (deterministic only), ABC (runtime contracts) | 5–8 yrs |
| SDLC meta-PL (Part 7) | 1 | 2 | Dagger (CI as code), Pijul (patch theory) — not unified | 5–8 yrs |
| Security / capability safety (cross-cut) | 4 | 2 | Pony (ref-caps), Jif (IFT) — not yet in AI-facing PLs | 2–4 yrs |
| PPL bridge — `{Stochastic}` effect (cross-cut) | 4 | 4 | NumPyro (inference-as-effect-handler) — effect row unification open | 2–3 yrs |
| Reflective / self-improving (cross-cut) | 2 | 2 | Lean 4 MetaM (substrate), AlphaProof (RL oracle) — `patch` primitive open | Research frontier |

Tiers 1–4 and the PPL bridge are buildable now with known techniques. Tier 5 (transformer dev PL) is an engineering challenge with a largely understood type theory. Tiers 6 and 7 contain open type-theoretic problems; their pseudo-code blocks in Chapter 206 are research agendas, not near-term blueprints. The reflective tier requires a `patch` primitive — self-modification with invariant preservation — that has no published solution.

---

## 203.3 Languages Designed for AI as Primary Author

### 203.3.1 MoonBit

**Status:** Production (v0.9.0, 2025) — the most complete general-purpose AI-native PL.

MoonBit was initiated in late 2022, explicitly designed for cloud and edge computing (WebAssembly and native targets). Influenced by Go and Rust, its design was presented at ICSE 2024 (LLM4Code workshop) with measurable AI-specific rationale. Five properties differentiate it from languages that merely support LLM tooling:

**Flat module structure.** Interface implementations live at the top level, not nested inside class bodies. MoonBit measures this as reducing KV cache misses during LLM inference — the type context for any definition is always retrievable from adjacent lines, not from an enclosing scope hierarchy.

**Mandatory top-level type annotations.** Every top-level function declaration requires an explicit type signature; the parser rejects unannotated definitions. The agent always has full type context in its generation window. This is the §203.1 inference split applied at the lexer/parser level.

**Semantic sampler.** Real-time syntactic and semantic constraint enforcement during token generation: the language toolchain actively guides the LLM by masking tokens that would produce syntactically or semantically invalid continuations. This is the implementation-level realisation of Outlines' FSM intersection (§203.6) applied to a full programming language.

**MoonBit Pilot.** A natively integrated code agent with async cloud execution, tightly coupled to the compiler and build system. The agent has direct access to the type-checker and can invoke it between generation steps.

**First-class MCP support.** Model Context Protocol integration built into the language toolchain, not bolted on as a library. The agent's tool handles are typed capability references, not ambient globals (anticipating Principle A, §206.3).

*References: [MoonBit Documentation](https://docs.moonbitlang.com/) · [ICSE 2024: Design of an AI-Friendly PL](https://dl.acm.org/doi/abs/10.1145/3643795.3648376)*

### 203.3.2 Pel

**Status:** Research / pre-release (arXiv 2505.13453, May 2025, CMU Tepper).

Pel addresses a gap that function/tool calling does not fill: an LLM orchestrating complex agentic workflows needs to express control flow, partial application, conditional branching, and error recovery — none of which fit cleanly into a JSON tool-call schema. Pel is a language for LLMs to *write* agentic programs, not general software.

Inspired by Lisp, Elixir, Gleam, and Haskell. The key design decisions:

**Homoiconic (Lisp-style AST = data).** The code AST is a value in the language, manipulable by functions in the same language. Generation and manipulation of Pel programs are typed operations, not string parsing.

**Minimal unambiguous grammar.** Suitable for constrained LLM generation: the grammar is small enough that a semantic sampler can enforce it in real time. Capability control is at syntax level — dangerous operations are not part of the grammar, eliminating the need for sandboxing.

**Built-in natural language conditions.** Conditions evaluated by sub-LLMs inline: `if (llm-eval "is this response in English?") { ... }`. The PL and the LLM inference layer are unified.

**REPeL.** Read-Eval-Print Loop with Common Lisp-style restarts combined with LLM helper agents for automated error recovery. When a Pel program encounters an error, the recovery strategy is itself an LLM call with the error state as context.

**Static dependency analysis → automatic parallelisation.** Independent operations are automatically parallelised by the evaluator, without programmer annotation.

*Reference: [arXiv 2505.13453](https://arxiv.org/abs/2505.13453)*

### 203.3.3 Dana

**Status:** Early release, June 2025 (AI Alliance / Aitomatic).

Dana ("Domain-Aware Neurosymbolic Agent" language) takes the most radical stance: the developer describes *what*, Dana handles *how*. Released under the AI Alliance (IBM, Meta, et al.) and positioned as evolution from AI-assisted coding to AI-native programming.

**LLM + symbolic grounding.** Deterministic, reproducible outputs via neurosymbolic execution: the LLM handles ambiguous reasoning; a symbolic executor handles deterministic computation. The combination avoids hallucination in deterministic code paths.

**Intent-driven development.** The programmer describes functionality; the language handles implementation. The intent is the source of truth; the implementation is derived, not authored.

**`reason()` calls.** First-class calls that invoke LLM reasoning with typed output constraints: `reason("classify this document", output: Category)`. The output type is enforced by the runtime.

**Compositional `|` operators** for self-improving pipelines: `pipeline = step1 | step2 | step3`. Each stage has typed I/O; the pipeline type is the composition of stage types.

*References: [Dana Documentation](https://aitomatic.github.io/dana/) · [AI Alliance announcement](https://thealliance.ai/)*

---

## 203.4 Vericoding Languages

*Vericoding* (term coined September 2025, Beneficial AI Foundation) is the generation of formally verified code from formal specifications. It contrasts with *vibecoding* — generation without formal correctness guarantees. LLM success rates on a benchmark of 12,504 formal specifications:

| Language | LLM success rate | Notes |
|---|---|---|
| Dafny | **82%** | Highest; simpler spec language than dependent type theories |
| Verus/Rust | **44%** | Systems-level verification; Rust syntax overhead |
| Lean 4 | **27%** → rising fast | Theorem prover; 68%→96% overall improvement across ~1 year |

POPL 2026 includes a dedicated Dafny vericoding track. Gartner forecasts 60% of new code AI-generated by end 2026. AI-authored code carries a 2.74× higher security vulnerability rate than human-written code — vericoding directly addresses this gap by making correctness a compiler output, not a review artifact.

### 203.4.1 Dafny

**Status:** Mature (Microsoft Research); highest LLM success rates in vericoding benchmarks.

Dafny is an imperative and functional hybrid that compiles to C#, Java, JavaScript, Go, and Python. LLMs generate both the implementation and its specification (preconditions, postconditions, invariants); the Dafny verifier checks correctness statically using Z3 (Chapter 185 §185.4). The combination makes Dafny usable as a **verification-aware intermediate language**: the agent generates Dafny, the verifier confirms it, and the compiler translates to the target language.

The spec syntax is first-class: `requires`, `ensures`, and `invariant` are language keywords parsed by the front-end, not annotations processed by a separate tool. Loop invariants and termination witnesses (`decreases`) are mandatory for loops whose termination the verifier cannot prove automatically. The agent generating a loop generates its invariant simultaneously.

A representative vericoded function — both implementation and spec are LLM-generated, the verifier checks correctness:

```dafny
// Binary search: spec + implementation generated together
method BinarySearch(a: array<int>, key: int) returns (index: int)
  requires a != null && forall i, j :: 0 <= i < j < a.Length ==> a[i] <= a[j]
  ensures 0 <= index ==> index < a.Length && a[index] == key
  ensures index < 0 ==> forall i :: 0 <= i < a.Length ==> a[i] != key
{
  var lo, hi := 0, a.Length;
  while lo < hi
    invariant 0 <= lo <= hi <= a.Length
    invariant forall i :: 0 <= i < lo ==> a[i] < key
    invariant forall i :: hi <= i < a.Length ==> a[i] > key
    decreases hi - lo
  {
    var mid := lo + (hi - lo) / 2;
    if a[mid] < key      { lo := mid + 1; }
    else if a[mid] > key { hi := mid; }
    else                 { return mid; }
  }
  return -1;
}
```

The verifier discharges all annotations using Z3 (Chapter 185 §185.4). If the LLM generates an incorrect loop invariant — for instance, omitting `forall i :: hi <= i < a.Length ==> a[i] > key` — the verifier rejects the program and the agent repairs it in the next pass.

Practically relevant: 82% LLM success rate on formal specs means that for a large class of programming tasks, a single LLM pass produces a verified implementation without human intervention.

*References: [dafny.org](https://dafny.org/) · [Dafny Language Reference](https://dafny.org/dafny/DafnyRef/DafnyRef) · [Dafny as Verification-Aware IL (arXiv 2501.06283)](https://arxiv.org/pdf/2501.06283)*

### 203.4.2 Lean 4

**Status:** Mature theorem prover and dependently-typed PL; most active AI proof research in 2025–2026.

Lean 4 is simultaneously a programming language and an interactive theorem prover. The Mathlib library makes it the de facto standard for AI-assisted formal mathematics. The propositions-as-types correspondence (Curry-Howard) means that a Lean 4 proof of `P` is a program of type `P` — proof and code are syntactically identical.

For vericoding: LLMs write proof scripts alongside code, guided by the type-checker's feedback. The AlphaProof system (Google DeepMind, Nature 2025) demonstrated that Lean 4's kernel can serve as a zero-false-positive reward signal for RL — the type-checker either accepts or rejects a proof term with no intermediate "probably correct" state. This makes Lean 4 uniquely suited to bootstrapped AI self-improvement (Chapter 207 §207.1.1).

The internal architecture — MetaM monad, TacticM, `quote`/`unquote` homoiconic operators, and `decide`/`native_decide` proof-by-computation — is covered in detail in Chapter 184 §184.1. For the purposes of this chapter, the key vericoding property is the 27% LLM success rate, rising fast: the Lean 4 community's rate on Mathlib4 benchmarks improved from 68% to 96% across a single year of tooling and model improvements.

*References: [lean-lang.org](https://lean-lang.org/) · [Theorem Proving in Lean 4](https://leanprover.github.io/theorem_proving_in_lean4/) · [Lean4Lean (arXiv 2403.14064)](https://arxiv.org/abs/2403.14064)*

### 203.4.3 Verus

**Status:** Active research (CMU/MSR); only systems-level verification language with LLM traction.

Verus embeds verification in Rust syntax: preconditions, postconditions, and invariants are written with Verus macros that are syntactically valid Rust, checked by the Verus verifier at compile time. No separate spec language is needed; the same source file contains implementation and specification.

Verification is purely static with zero runtime overhead — verified Verus code is as fast as unverified Rust. The `verusfmt` autoformatter enforces canonical representation, and a Verus-aware fork of rust-analyzer provides IDE integration. For AI-first use, the most relevant property is that Verus specs are valid Rust syntax — an LLM fine-tuned on Rust has partial prior knowledge of the spec notation.

The 44% LLM success rate reflects the additional difficulty of systems-level verification: proving memory safety, arithmetic overflow absence, and pointer aliasing freedom requires more complex invariants than Dafny's typical use cases.

*References: [Verus Tutorial and Reference](https://verus-lang.github.io/verus/guide/) · [Verus Playground](https://play.verus-lang.org/)*

---

## 203.5 Languages with Structural AI Affinity

### 203.5.1 Unison

**Status:** 1.0 released November 2025.

Unison's content-addressed code model turns out to be exceptionally AI-friendly even though it was designed for distributed computing. In Unison, functions are not identified by name but by a cryptographic hash of their AST. The "text file" is a view generated by the Unison Codebase Manager; the actual codebase is a typed AST database.

**No rename cascades.** Refactoring has zero blast radius — a rename is a metadata change, not a search-and-replace across source files. An AI agent refactoring a Unison codebase can do so without triggering downstream recompilation cascades.

**MCP server.** AI agents typecheck code, browse documentation, and inspect dependencies through a typed interface without touching text files. The agent's view of the codebase is always type-consistent.

**Algebraic effects (abilities system).** Unison's effects system (`Ability`) is row-polymorphic in the same sense as Koka's. An agent generating Unison code has a typed effect signature for every function call.

**Distributed runtime built-in.** Code hashes are globally unique identifiers, enabling hash-stable distributed code execution — a function's identity does not change when deployed across machines.

The connection to the PPL bridge (§203.7): execution traces are content hashes of stochastic choices, making reproducibility-by-type achievable (paradigm 3). If `{Stochastic}` is absent from a function's effect row, the compiler guarantees its trace hash is constant across runs.

*References: [Unison Documentation](https://www.unison-lang.org/docs/) · [Tour of Unison](https://www.unison-lang.org/docs/tour/)*

### 203.5.2 Koka

**Status:** Active research (Microsoft Research, Koka 3.x, 2025).

Koka is not AI-first by design, but it is the *reference implementation* for the algebraic effects type system that all subsequent capability tiers in this chapter require. Every effect-row formula in this part — `{Stochastic}`, `{Gradient}`, `{RequiresHumanApproval}`, `{HighIntegrity}` — is Koka's type system applied to new domains.

**Row-polymorphic effects.** A function type `fun f() : <div,exn,state<int>> int` makes the effect row part of the function's type. Handlers compose polymorphically over effect rows: a handler that closes over `{Stochastic}` leaves other effects open. The effect row is additive — `{Stochastic, Gradient, IO}` is a valid compound row.

**Effect inference.** Koka infers effect rows without manual annotation. The compiler checks that all declared effects are covered by handlers in scope. An AI generating Koka code receives the inferred effect signature as part of the type-checker's feedback.

**First-class handlers.** A handler is an ordinary expression. A handler for `{Stochastic}` that seeds a fixed random number generator makes a stochastic computation fully deterministic — the same mechanism Pyro uses for reproducible variational inference (§203.7, paradigm 2).

The specific AI-first applications, shown in Koka syntax:

```koka
// Effect declarations for AI-first operations
effect stochastic
  fun sample(d: dist<a>) : a

effect gradient
  fun grad-of(f: () -> <gradient> float) : float  // reverse-mode

effect requires-human-approval
  fun request-approval(action: string) : bool

// A stochastic function: effect row is part of the type
fun dropout(x: vector<float>, p: float) : <stochastic> vector<float>
  x.map(fn(v) -> if sample(bernoulli(1.0 - p)) then v / (1.0 - p) else 0.0)

// Inference handler: closes over {stochastic}, makes dropout deterministic
fun with-fixed-seed(rng: rng, action: () -> <stochastic,div|e> a) : <div|e> a
  with handler
    fun sample(d) resume(rng.draw(d))
  action()

// Effect composition: both stochastic and gradient required
fun compute-elbo(model: () -> <stochastic,gradient> float) : <stochastic,gradient> float
  val sample-val = model()
  val log-q = grad-of(fn() -> model())
  sample-val - log-q

// {RequiresHumanApproval} propagates: every caller acquires the effect
fun deploy-model(spec: model-spec) : <requires-human-approval> ()
  val approved = request-approval("Deploy " ++ spec.name ++ " to production")
  if approved then do-deploy(spec) else throw("Deployment rejected")

// The handler gate: requires explicit human interaction
fun with-human-gate(action: () -> <requires-human-approval,io|e> a) : <io|e> a
  with handler
    fun request-approval(msg)
      println("APPROVAL REQUIRED: " ++ msg)
      val response = readline()
      resume(response == "yes")
  action()
```

The `{requires-human-approval}` effect propagates exactly as any Koka effect propagates: every function in the call graph that calls `deploy-model` acquires the `requires-human-approval` label in its own effect row. The compiler statically rejects programs that call `deploy-model` outside a `with-human-gate` handler. There is no runtime bypass — the type system enforces the human-in-the-loop requirement structurally.

The row-polymorphic composition property is critical for the AI-first use case: `<stochastic,gradient,requires-human-approval>` is a valid compound effect row. A model training loop that samples from distributions, computes gradients, and requires human approval for checkpointing is expressed with a single effect row that all three handlers address independently.

*References: [Koka Language (MSR)](https://koka-lang.github.io/) · [Algebraic Effects for Functional Programming (Leijen, 2016)](https://www.microsoft.com/en-us/research/publication/algebraic-effects-for-functional-programming/)*

### 203.5.3 Idris 2

**Status:** Active research/production (Idris 2.0+, 2024; Edwin Brady, University of St Andrews).

Idris 2 implements Quantitative Type Theory (QTT): every variable carries a *quantity* (0 = erased at runtime, 1 = used exactly once, ω = unrestricted). This is dependent types *and* linear types in one coherent system, not two separate mechanisms bolted together.

The AI-first applications are concrete:

**Token and cost budgets as linear types.** `Agent[Budget: 1 Token]` expresses that the token budget is consumed exactly once — the linear quantity enforced by the type system. The multi-agent PL's resource tracking for agent budgets is QTT in direct application (Chapter 206 §206.1).

**Device memory as quantities.** `Tensor[Batch, D]{1}` — a tensor consumed exactly once — prevents accidental aliasing of GPU memory buffers. The type system statically catches double-free and use-after-move errors in kernel code.

**Capability attenuation as erasure.** Capabilities with quantity 0 are statically erased — a zero-quantity capability is provably unused. The E language's "no ambient authority" model (Chapter 206 §206.3) is quantity-0 authority: code that does not receive a capability as an argument has quantity-0 authority for the corresponding resource.

**Self-describing types.** Idris 2's universe polymorphism combined with QTT allows writing types that describe their own resource usage: `MetaModel[P]` (resources consumed and produced by program `P`) is directly expressible in QTT.

*References: [Idris 2 Documentation](https://idris2.readthedocs.io/) · [QTT in Practice (arXiv 2104.00480)](https://arxiv.org/abs/2104.00480)*

---

## 203.6 LLM Interaction Languages

### 203.6.1 LMQL

**Status:** Active (ETH Zürich).

LMQL is a query language for LLMs: a Python-embedded DSL that provides declarative constraint syntax (regex, types, token budgets) to guide LLM token generation rather than post-processing output. The constraint is applied at generation time via logit masking, not as a filter on completed outputs.

The connection to the unified PL: LMQL's constraint syntax is an approximation of the `LLMCall` typed primitive (G1, Chapter 207 §207.2). An `LLMCall` in the unified PL has the same properties LMQL provides — schema constraints on output, budget tracking — plus the `{Stochastic}` algebraic effect and IFT security labels that LMQL does not yet support.

*References: [lmql.ai](https://lmql.ai/) · [LMQL Language Reference](https://lmql.ai/docs/language/reference.html)*

### 203.6.2 Guidance

**Status:** Production (Microsoft, Guidance 0.1.x, 2025).

Guidance uses Handlebars-style templates to specify exactly where LLM generation occurs within a structured output. Unlike LMQL's logit-masking approach, Guidance interleaves `{{gen}}` calls with static template structure, giving precise control over output shape via grammar-constrained generation (context-free grammar constraints on generated tokens).

**Structured output as a language primitive.** `{{gen 'field' regex=r'\d+'}}` generates a regex-constrained string — the template is the schema, not a post-processing filter. Grammar-constrained generation is the implementation-level realisation of Outlines' FSM intersection applied to the Guidance template evaluation model.

**Stateful generation.** Generation state persists across `{{gen}}` calls within a template — the template is a typed multi-turn conversation protocol.

*References: [Guidance GitHub](https://github.com/guidance-ai/guidance)*

### 203.6.3 Outlines

**Status:** Production (dottxt-ai, Outlines 0.1.x, 2025).

Outlines provides the strongest formal guarantee of the LLM interaction tools: it generates a finite-state machine (FSM) from a JSON schema, regex, or context-free grammar, then intersects it with the LLM's token vocabulary to produce a constrained logit mask at each generation step. The result is provably correct structured output — if the schema is satisfiable, the output satisfies it.

**FSM intersection as type checking.** The intersection of a CFG with a token vocabulary is equivalent to type-checking the generation against the schema *before* any token is emitted. Type errors are impossible at runtime: the token vocabulary mask ensures no invalid token can be chosen. This is the formal correctness property that LMQL approximates with logit masking but does not prove.

**AI-first significance.** MoonBit's semantic sampler (§203.3.1) is exactly Outlines' FSM intersection applied to a full programming language grammar. The conceptual connection: `@outlines.prompt(template, model=pydantic.Model)` makes the model's output schema a type annotation on the generation call — a step toward the `LLMCall[Out: Schema]` typed primitive of G1.

*References: [Outlines GitHub](https://github.com/outlines-dev/outlines)*

### 203.6.4 DSPy

**Status:** Active research/production (Stanford NLP, DSPy 2.x, 2025).

DSPy separates the *what* (I/O signature: typed input and output fields with constraints) from the *how* (prompting strategy, few-shot examples, chain-of-thought). The optimizer (`MIPROv2`, `BootstrapFewShot`) automatically finds the optimal prompting strategy for a given metric. The programmer declares intent; DSPy compiles it to the optimal prompt.

**Typed I/O signatures as machine-readable contracts.** `Signature("question: str -> answer: str, confidence: float")` is a typed specification of LLM I/O — the closest existing system to a formal contract for LLM behaviour. The signature is the type annotation; the optimizer is the semantic sampler for the prompt space.

**Composability.** DSPy modules compose like function calls: the pipeline type is the composition of I/O signatures. This is the right abstraction level for the multi-agent interaction layer: typed handoffs between agents are typed compositions of DSPy signatures.

**Evaluation as oracle.** DSPy's metric function is an executable correctness oracle — directly connecting to the `TestOracle[T]` in the AI-first SDLC meta-PL (Chapter 206 §206.2). A DSPy metric is a `Spec[T] → Bool` function that the optimizer uses to search the prompt space.

The G4 problem (Chapter 207 §207.2) — multimodal I/O types (`Image[W,H,Format]`, `Audio[SampleRate,Channels]`, `Document[Schema]`, `Video[FPS,W,H]`) as first-class type parameters on `LLMCall` — is the extension of DSPy's I/O signatures to multimodal inputs. No existing system formalises this.

*References: [DSPy Documentation](https://dspy.ai/) · [DSPy paper (arXiv 2310.03714)](https://arxiv.org/abs/2310.03714)*

---

## 203.7 The Probabilistic Programming Bridge

An LLM is a probabilistic program: its outputs are samples from `P(token | context)`. Any AI-first PL that reasons formally about LLM behaviour must have probabilistic semantics at its type-theoretic core. The probabilistic programming language (PPL) research tradition — Stan, Pyro/NumPyro, Gen — has worked out what those semantics look like at production scale. Five design paradigms emerge from that tradition, each mapping to a concrete requirement on the unified AI-first PL.

### 203.7.1 Paradigm 1 — Distributions as First-Class Types

Stan's contribution: parameter constraints are refinement types. `real<lower=0>` is `{r: Real | r > 0}` — a named type that carries a constraint the verifier checks. The unified PL generalises this: `Dist[A]` is a type constructor, and a stochastic expression has type `Dist[A]`, not `A`. Probabilistic correctness becomes machine-checkable: the analogue of a Dafny postcondition for a stochastic program is "the posterior satisfies its distributional specification."

```pseudo
-- Stochastic expressions have type Dist[A], not A
sample  : Dist[A] → <Stochastic> A        -- effect-producing elimination
observe : Dist[A] → A → <Conditioning> () -- condition on observed evidence

-- Stan-style refinement types for distribution parameters
type PositiveReal = { r : Float | r > 0.0 }
type Simplex[N]   = { v : Vec[N, Float] | sum(v) == 1.0 && all(v, fn(x) -> x >= 0.0) }

-- A Bayesian linear regression model with typed parameters
fn linear_regression(x: Vec[N, Float], y: Vec[N, Float]) : <Stochastic, Conditioning> Float
  let alpha : PositiveReal = sample(HalfNormal(0.0, 1.0))
  let beta  : Float        = sample(Normal(0.0, 1.0))
  let sigma : PositiveReal = sample(HalfNormal(0.0, 1.0))
  for i in 0..N do
    observe(Normal(alpha + beta * x[i], sigma), y[i])
  return alpha
```

The type-level distinction between `A` and `Dist[A]` prevents the most common class of probabilistic programming errors: treating a distribution object as a sample value. The refinement type `PositiveReal` is a compiler-checkable constraint that Stan enforces at the type level — the `sigma` variable can only be bound to a distribution whose support is the positive reals.

### 203.7.2 Paradigm 2 — Inference as Algebraic Effect Handler

Pyro's contribution: inference algorithms (variational inference, HMC, importance sampling, SMC) are effect handlers over stochastic programs. The same model code runs under different inference regimes by swapping the handler, not rewriting the model. In the unified PL:

```pseudo
-- Same model code; different handlers give different inference regimes
with HMC(warmup=500, samples=1000) { model(data) }    -- MCMC posterior
with VI(guide)                     { model(data) }    -- variational approximation
with Enumerate                     { model(data) }    -- exact (discrete-only models)
```

This is exactly the Koka row-polymorphic effect pattern applied to probabilistic computation. `{Stochastic}` is an open effect that any handler can close over. Composition with `{Gradient}` (automatic differentiation, Chapter 205 §205.3.2) and `{Parallel}` (data-parallel training) is natural because effect rows are additive: `<Stochastic, Gradient, Parallel>` is a valid compound row that a composed handler addresses.

### 203.7.3 Paradigm 3 — Trace as First-Class Typed Value

Gen's contribution: an execution trace — the record of all random choices made during a probabilistic program run — is a first-class typed value, not an opaque side effect.

In the unified PL this connects directly to Unison's content-addressing (§203.5.1): a program's execution trace is a content hash of its stochastic choices. Two runs with the same trace hash are semantically identical. Reproducibility becomes a type-level property: if `{Stochastic}` is absent from a function's effect row, the compiler guarantees that its trace hash is constant across all runs with the same inputs.

### 203.7.4 Paradigm 4 — Compiler-Derived Gradients

Stan's compiler derives all gradients from the model specification — no user-written gradient code. The programmer writes the probabilistic model; Stan generates the gradient computation needed for HMC.

This is the same principle the unified PL applies to transformer development (Chapter 205 §205.3.2): `grad` is a type-level transform over the elaborated AST, not a library call. Stan proves this is implementable at production scale for arbitrary probabilistic models. The unified PL generalises it: `∂/∂x f` is a compiler pass whose output is a gradient function whose type is verified against the original function's type by the type checker.

### 203.7.5 Paradigm 5 — Convergence as Refinement Types

Stan's parameter constraint types (`positive_real`, `simplex`, `correlation_matrix`) ensure that MCMC samplers only explore the valid parameter space, providing a constraint-satisfaction guarantee at compile time.

In the unified PL this becomes: posterior consistency bounds are compiler-checkable refinement types. A training loop with a learning rate schedule has a `converges_to(ε)` annotation; the type system checks whether the schedule satisfies the convergence preconditions. This is the G6 open problem (Chapter 207 §207.2) — it requires bridging continuous functional analysis (convergence theory) and discrete formal logic (type theory).

### 203.7.6 Reference Systems

| System | Paradigms contributed | Status |
|---|---|---|
| **Stan** (compiler-derived AD, constraint refinement types) | 1, 4, 5 | Production (2.36) |
| **Pyro / NumPyro** (inference-as-effect-handler, JAX composition) | 2 | Production (Pyro 1.9 / NumPyro 0.16) |
| **Gen** (trace as first-class value, composable generative functions) | 3 | Research / active |

| PPL paradigm | Unified PL requirement | Connects to |
|---|---|---|
| `Dist[A]` as type constructor | `sample : Dist[A] →<Stochastic> A` in type system | Ch205 §205.3.6 (stochasticity effect) |
| Inference-as-effect-handler | Swappable `{Stochastic}` handlers | §203.5.2 Koka effect rows |
| Trace-as-typed-value | Execution traces as content hashes; reproducibility by type | §203.5.1 Unison content-addressing |
| Compiler-derived gradients | `grad` as a type-level transform, not a library | Ch205 §205.3.2 (first-class AD) |
| Convergence as refinement | `converges_to(ε)` as a type annotation on training loops | G6 open problem (Ch207 §207.2) |

*References: [mc-stan.org](https://mc-stan.org/) · [pyro.ai](https://pyro.ai/) · [num.pyro.ai](https://num.pyro.ai/) · [gen.dev](https://www.gen.dev/)*

---

## 203.8 WebAssembly as AI-First Deployment Substrate

Three threads from the language survey converge on WebAssembly as the natural deployment substrate for an AI-first PL:

**MoonBit targets WASM/native.** KV-cache-aware syntax and MCP support are available in edge deployments without a Python runtime. The same compiler output runs in a browser, on a cloud function, or on an embedded device.

**Unison is content-addressed and WASM-deployable.** Hash-stable distributed code execution means a function's identity does not change when deployed from a development machine to a production cluster. The content hash is the deployment artifact.

**Wasm Component Model as object-capability system.** Components export and import capabilities as typed interfaces, directly implementing the E language's no-ambient-authority principle (Chapter 206 §206.3) at the module boundary. A component that does not import a `file-system` interface cannot access the filesystem — no sandbox policy engine required, because the capability model is enforced by the WASM runtime's type system. Chapter 106 covers the WASI preview 2 interface and WasmGC typed references in detail.

The implication: an AI-first PL that targets WASM gets capability safety and hardware portability as compiler outputs, not runtime add-ons. The semantic sampler (§203.3.1) generates WASM modules; the Wasm Component Model's capability interfaces are the type-level enforcement of §203.1's "explicit effects" and §206.3's security principles.

---

## Chapter 203 Summary

- An AI-first PL inverts seven human-PL design priorities: formal semantics over convention, rich type systems as machine-readable specs, explicit algebraic effects, homoiconicity, canonical single representation, structured typed diagnostics, and module-level edit locality.
- The type inference strategy (G2) draws the line at intent: infer shapes, effects, and capabilities; require explicit annotation for specifications, convergence bounds, and security labels.
- The nine-tier capability maturity matrix spans from production-ready (coding agent PLs, LLM interaction layer) to research frontiers (multi-agent coordination, reflective self-improving code).
- MoonBit (semantic sampler + MCP), Pel (homoiconic + REPeL), and Dana (intent-driven + neurosymbolic grounding) represent purpose-built AI-first PLs at different points on the formality spectrum.
- Dafny (82% LLM vericoding success), Lean 4 (propositions-as-types, AlphaProof RL oracle), and Verus (systems-level Rust verification) are the vericoding tier with the highest near-term impact.
- Unison (content-addressing, no rename cascades), Koka (row-polymorphic effects — the reference implementation for all effect-row formulas in this part), and Idris 2 (QTT — dependent types + linear resource tracking) provide structural AI affinity that purpose-built AI-first PLs should absorb.
- LMQL, Guidance, Outlines (FSM intersection = formal correctness), and DSPy (typed I/O signatures, automatic prompt optimisation) constitute the LLM interaction layer, already production-ready.
- Five PPL paradigms from Stan/Pyro/Gen ground probabilistic AI behaviour in the type system: `Dist[A]` as a type constructor, inference as an algebraic effect handler, traces as typed values, compiler-derived gradients, and convergence bounds as refinement types.
- WebAssembly combines MoonBit's edge deployment, Unison's content-addressing, and the Wasm Component Model's object-capability system into a substrate where capability safety is a compiler output.
