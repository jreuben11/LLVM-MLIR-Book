# Part II — Compiler Theory — Part Summary

*This part develops the complete theoretical foundation of compilation — from character streams to optimized intermediate code — equipping the reader to understand, extend, and reason about every phase of a production compiler such as Clang.*

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 6 | Lexical Analysis | Regular languages, finite automata, DFA minimization, hand-written lexers |
| 7 | Parsing Theory | CFGs, LL/LR/LALR, Pratt, Earley, recursive descent |
| 8 | Semantic Analysis Foundations | Symbol tables, scoping, attribute grammars, type checking, name resolution |
| 9 | Intermediate Representations and SSA Construction | IR taxonomy, dominator trees, Cytron algorithm, CPS–SSA equivalence |
| 10 | Dataflow Analysis: The Lattice Framework | Complete lattices, Tarski fixed points, worklist algorithms, abstract interpretation |
| 11 | Classical Optimization Theory | GVN, LICM, PRE, register allocation, instruction scheduling |

## Part Overview

Part II answers the question every compiler writer faces: what does each phase of a compiler actually compute, and why? The six chapters form a strict pipeline — each chapter's output is the next chapter's input — mirroring the structure of a real compilation pipeline. The treatment is simultaneously mathematically rigorous and grounded in the concrete designs of Clang, GCC, LLVM, and Tree-sitter, so the theory never floats free of implementation.

Chapter 6 establishes that the character-to-token boundary is justified by the Chomsky hierarchy: regular languages (Type-3) support O(n) recognition via DFA, and the Myhill–Nerode theorem guarantees a unique minimal DFA for any such language. Thompson's NFA construction, the subset construction, and Hopcroft's partition-refinement minimization form the complete algorithmic pipeline from regular expression to minimal DFA. The chapter then explains why every major production compiler — Clang, GCC, rustc — has abandoned lexer generators in favor of hand-written tight loops: the formal approach cannot express context-sensitive tokenization (C++ angle brackets, raw string literals, Python INDENT/DEDENT) and is an order of magnitude slower than SIMD-accelerated hand-written code. Chapter 7 extends the grammar hierarchy one step: context-free grammars (Type-2) recognized by pushdown automata are the right model for program syntax. The chapter develops LL(1) FIRST/FOLLOW table construction, the full LR(0)/SLR/LALR/LR(1) hierarchy with worked item-set automata, Pratt's top-down operator-precedence algorithm, Earley parsing, PEG and packrat linear-time parsing, and parser combinators. It closes by explaining why hand-written recursive descent dominates production compilers: C++ is not context-free, the grammar–action interleaving enables semantic disambiguation during parsing, and the call stack provides the syntactic context needed for precise error messages.

Chapter 8 covers semantic analysis — the phase that enforces meaning rather than structure. Three symbol-table designs (chained, flat-hash with scope stacks, persistent HAMT-based) are analyzed for their complexity trade-offs. The chapter formalizes Knuth's attribute grammar framework (synthesised attributes flow bottom-up; inherited attributes flow top-down; L-attributed grammars map onto recursive-descent evaluation), derives the backpatching SDT for if-else and while control flow, and develops type-checking as a structural recursion implementing the typing judgment Γ ⊢ e : τ. The critical C++ mechanisms — two-phase template name lookup, argument-dependent lookup (Koenig lookup), and overload resolution via conversion-sequence ranking — are derived from first principles and connected to Clang's `Sema` implementation. The Visitor pattern is revealed as a direct compilation of an attribute grammar evaluator. Chapter 9 introduces intermediate representations as the bridge between the decorated AST and machine code, motivating them through the N×M compiler architecture problem. A taxonomy of IRs (AST, three-address code, CPS, ANF, SSA) explains the design trade-offs, followed by a rigorous treatment of SSA construction: dominator trees (Cooper–Harvey–Kennedy and Lengauer–Tarjan algorithms), dominance frontiers, the Cytron–Ferrante–Rosen–Wegman–Zadeck iterated-dominance-frontier algorithm, and the renaming pass. A fully worked six-block loop CFG carries every step to completion. The chapter proves Appel's CPS–SSA isomorphism — each SSA basic block is a CPS continuation, each φ-function is a formal parameter — and shows that MLIR makes this isomorphism syntactically explicit through block arguments.

Chapter 10 provides the mathematical framework that gives all dataflow analyses a unified treatment. Complete lattices, Tarski's fixed-point theorem (guaranteeing existence of the least fixed point), and Kleene's ascending chain theorem (guaranteeing computability by finite iteration) are developed from first principles. The worklist algorithm and its bit-vector implementation are derived, and every classical analysis — liveness, available expressions, reaching definitions, very busy expressions, constant propagation via the flat lattice — is presented as an instantiation of the same monotone-framework template. The chapter extends to infinite-height lattices via Cousot–Cousot widening and narrowing operators, making abstract interpretation a first-class citizen of the theory, and closes with interprocedural extensions: call strings, k-CFA, summary-based IFDS analysis. Chapter 11 converts analysis results into transformations. Local optimizations (constant folding, peephole, dead code elimination) require no dataflow; global optimizations (GVN/NewGVN via value numbering on SSA, LICM via dominator-tree hoisting, strength reduction via ScalarEvolution, PRE via lazy code motion) consume the lattice fixed-points computed in Chapter 10. Register allocation theory is developed in depth — the interference graph, NP-completeness of graph coloring, Chaitin's algorithm, Chaitin–Briggs improvements, George–Appel iterated register coalescing, linear scan for JITs, and PBQP for architectures with complex constraints. Instruction scheduling (dependence DAG, list scheduling, software pipelining) and garbage collection theory (mark-sweep, Cheney copying, generational GC, compiler-supported statepoints) complete the middle-end survey.

After completing this part, the reader can derive the correctness conditions for any standard compiler analysis, implement a monotone dataflow framework from scratch, understand why SSA is the universal IR of production optimizers, and read the LLVM and GCC source code for passes such as `mem2reg`, `LICM`, `NewGVN`, `PHIElimination`, and the greedy register allocator with theoretical comprehension rather than pattern matching.

## Key Concepts Introduced

- **Chomsky Hierarchy** — Four nested language classes (Type-0 through Type-3); lexical analysis operates in Type-3 (regular), syntax in Type-2 (context-free), semantics in Type-1 and beyond.
- **Thompson's Construction** — Algorithm converting a regular expression of size |r| to an ε-NFA with at most 2|r| states, the foundation of every lexer generator.
- **Subset Construction and DFA Minimization** — NFA-to-DFA conversion (worst-case exponential, near-linear in practice); Hopcroft's O(n log n) partition-refinement algorithm produces the unique minimal DFA guaranteed by the Myhill–Nerode theorem.
- **Context-Free Grammar and Parse Tree** — CFG G = (V, T, P, S) generating a language via leftmost/rightmost derivations; the parse tree is the semantic object, the derivation the construction method.
- **LALR(1) Parsing** — The practical sweet spot of LR table generation: merge LR(1) states with identical cores, resolving conflicts via lookahead sets; the standard for yacc, bison, and most parser generators.
- **Pratt Parsing (Top-Down Operator Precedence)** — Encodes operator precedence as integer binding powers with per-token nud/led handlers; O(n), easily extensible to user-defined operators, widely used in production compilers for expression sub-grammars.
- **Attribute Grammars** — Knuth's formalism for semantic computation over parse trees; synthesised attributes flow bottom-up (S-attributed, evaluable by LR parsers), inherited attributes flow top-down (L-attributed, evaluable by recursive descent).
- **Symbol Table and Scoping** — The operational realisation of the type environment Γ; chained (scope stack), hash-based (per-entry binding stacks), and persistent (HAMT-based) designs; lexical vs. dynamic scoping; C++'s eight-level scope hierarchy.
- **Static Single Assignment (SSA) Form** — IR invariant requiring each variable to be defined exactly once; φ-functions at join points gather incoming values; enables O(1) use-def lookup, trivial dead-code detection, and efficient GVN.
- **Dominator Tree and Dominance Frontiers** — The dominator tree T_D encodes "A dom B iff A is an ancestor of B"; the dominance frontier DF(A) is the set of blocks where A's dominance stops; DF⁺(Defs(v)) gives the exact φ-placement sites for variable v.
- **Cytron–Ferrante–Rosen–Wegman–Zadeck Algorithm** — The standard SSA construction: compute IDF via worklist, insert φ-functions, rename via dominator-tree DFS maintaining per-variable stacks.
- **CPS–SSA Isomorphism** — Appel's theorem: each SSA basic block is a CPS continuation, φ-parameters are formal parameters, control-flow edges are tail calls; MLIR expresses this directly via block arguments.
- **Complete Lattice and Tarski Fixed-Point Theorem** — A complete lattice (S, ≤, ⊓, ⊔, ⊥, ⊤) admits a least fixed point for any monotone function (Tarski 1955); Kleene's ascending chain theorem guarantees computability when the lattice has finite height.
- **Abstract Interpretation and Galois Connections** — Cousot–Cousot framework formalizing the relationship between a concrete semantics and an abstract approximation via a Galois connection (α, γ); widening collapses infinite ascending chains for convergence; narrowing regains precision post-convergence.
- **Graph Coloring Register Allocation** — Assign physical registers to SSA values by coloring an interference graph (two values interfere if their live ranges overlap); Chaitin's algorithm, Chaitin–Briggs conservative coalescing, George–Appel iterated coalescing; graph coloring is NP-complete, handled via degree-k simplification heuristics and spilling.

## How This Part Fits the Book

Part I — Foundations (Chapters 1–5) provides the LLVM/Clang ecosystem overview, build system, C++ prerequisite skills, and the compiler pipeline at a high level; Part II takes those prerequisites and derives the complete theoretical framework that underlies every phase. Parts III (Type Theory, Chapters 12–15) and IV (LLVM IR, Chapters 16–25) build directly on Part II: Part III extends the type-checking theory of Chapter 8 to the Simply Typed Lambda Calculus, Hindley–Milner inference, subtyping, and dependent types, while Part IV makes the SSA theory of Chapter 9 concrete in LLVM IR, covering LLVM's type system, instruction set, the pass infrastructure, and the alloca+mem2reg strategy that Chapter 9 §9.10 introduces. The optimization theory of Chapters 10–11 is then put into practice in Chapters 61–63 (LLVM's analysis infrastructure and scalar/loop optimization passes) and Chapters 89–91 (the backend lowering, register allocation, and instruction scheduling pipelines).

## Cross-Part Dependencies

- **Ch 12 (Part III) — Simply Typed Lambda Calculus** — builds on Ch 8's typing judgment Γ ⊢ e : τ and the type environment as a persistent symbol table.
- **Ch 13 (Part III) — Hindley–Milner Type Inference** — extends Ch 8's type-checking tree walk with unification variables and Algorithm W; the S-attributed evaluation structure of Ch 8 §8.3 carries forward directly.
- **Ch 16–17 (Part IV) — LLVM IR and its Type System** — concretize the SSA theory of Ch 9; the alloca/mem2reg strategy and the `Value`/`Use` graph architecture are direct implementations of Ch 9's SSA invariant and def-use chain discussion.
- **Ch 39–40 (Part VI) — Clang CodeGen** — the AST-to-IR lowering described there follows the N×M argument of Ch 9 §9.1.2 and the SDT backpatching of Ch 8 §8.4.4.
- **Ch 61 (Part X) — LLVM Analysis Infrastructure** — implements the lattice-theoretic analysis framework of Ch 10; the `AnalysisManager` is the caching mechanism for lattice fixed-point results.
- **Ch 62–63 (Part X) — Scalar and Loop Optimizations** — implement GVN (Ch 11 §11.2), LICM (Ch 11 §11.3), ScalarEvolution/strength reduction (Ch 11 §11.4), and PRE/lazy code motion (Ch 11 §11.5) in LLVM.
- **Ch 89–91 (Part XIV) — Backend Pipeline, Register Allocation, Scheduling** — implement Chaitin–Briggs and George–Appel coloring (Ch 11 §11.6), linear scan (Ch 11 §11.7), list scheduling, and software pipelining (Ch 11 §11.8) in LLVM's backend.
- **Ch 95 (Part XXIV) — Verified Compilation (Vellvm, Alive2)** — the formal SSA semantics developed in Ch 9 and the soundness conditions of Ch 10 §10.10 are the objects these verification tools reason about.
- **Ch 108 (Part XXVII) — Mathematical Foundations** — the lattice theory, fixed-point theorems, and abstract interpretation of Ch 10 receive full mathematical treatment (Coq/Lean4 mechanization, domain theory) in Part XXVII.

## Navigation

- ← Part I — Foundations
- → Part III — Type Theory

---

*@copyright jreuben11*
