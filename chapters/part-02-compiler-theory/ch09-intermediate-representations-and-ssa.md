# Chapter 9 — Intermediate Representations and SSA Construction

*Part II — Compiler Theory and Foundations*

Chapter 8 completed the front-end pipeline: names are resolved, types are checked, and the abstract syntax tree is fully decorated with semantic information. The compiler now possesses a faithful, tree-structured account of what the program means. Yet that tree is a poor substrate for the transformations that make compiled code fast. Recursive tree structure does not expose dataflow, the ordering of computations is implicit in the tree topology, and many classical optimisation algorithms assume a flat, linear sequence of instructions with explicit use-def relationships. The gap between the AST and final machine code is bridged by one or more **intermediate representations** (IRs), which this chapter studies in depth.

The chapter's first half surveys the design space of IRs — from historical P-code to modern three-address code, continuation-passing style, and administrative normal form — and explains why Static Single Assignment form has become the industry standard. The second half is a rigorous development of SSA construction: dominator trees, dominance frontiers, the Cytron–Ferrante–Rosen–Wegman–Zadeck algorithm, φ-function placement, and the renaming pass. A fully worked six-block example carries every step through to completion. We then treat pruned and semi-pruned variants, the subtleties of out-of-SSA destruction, the deep equivalence between SSA and continuation-passing style, and the specific way LLVM IR and MLIR realise these ideas in practice.

Chapter 10 builds directly on this foundation: dataflow analysis — liveness, reaching definitions, available expressions — is most clearly expressed as a lattice-valued computation over SSA or its CFG, and the SSA structure dramatically speeds up every classical dataflow algorithm.

---

## 9.1 Why an IR? The N×M Problem and IR Properties

### 9.1.1 The multi-pass compilation model

A compiler is not a single function from source text to machine code. It is a pipeline of *passes*, each consuming a representation, performing an analysis or transformation, and producing a refined representation. This design is not accidental: separation of concerns is as valuable in compilers as in any software system. Type checking must precede code generation; loop optimisation must precede register allocation; instruction selection must precede scheduling. Each pass is smaller, more testable, and more portable than a monolithic translator would be.

The AST produced by parsing and semantic analysis (Chapters 7–8) is the natural output of the front end. It is, however, poorly suited to most subsequent phases:

- **Dataflow is implicit.** The edges of a tree record syntactic containment, not data dependencies. To compute that `x` is used after it is defined, the compiler must traverse the tree, maintaining auxiliary structures such as reaching-definition sets and use-def chains. A flat, explicitly-linked representation makes these edges first-class citizens of the IR.
- **The tree is non-linear.** Register allocation, instruction scheduling, and code emission all work on a linear ordering of instructions. Flattening the tree to obtain that ordering requires a non-trivial traversal (a post-order walk for most expression trees). This traversal is better performed once, at the point of lowering to IR, rather than repeated in every downstream pass.
- **High-level constructs have no machine analogue.** A function call in the AST may encode a complex overload-resolution result, default-argument expansion, and an ABI-specific calling convention. The IR should expose the already-resolved target, the stack frame layout, and the calling convention directly, so each backend does not re-implement this logic.
- **Loop and branch structure must be made explicit.** An `if` node in the AST is convenient for type checking but contains no information about which registers hold live values at the branch point. The CFG representation makes predecessor and successor blocks explicit.

The IR thus serves as the *lingua franca* between the front end and back end: it is expressive enough to faithfully represent source-level semantics (after lowering), yet structured enough to support analysis and transformation.

### 9.1.2 The N×M problem

The decisive industrial argument for IRs is the **N×M problem** [EaC §1.3]. Suppose a compiler project targets N source languages (C, C++, Fortran, Rust, …) and M machine architectures (x86-64, AArch64, RISC-V, WebAssembly, NVPTX, …). Without a shared IR, each (language, target) pair requires a dedicated translator: N×M translators in total. With a shared IR, each of the N front ends emits IR and each of the M back ends consumes IR: N+M components. More importantly, any improvement to the middle end (the optimiser that consumes and emits IR) benefits every language–target combination simultaneously, with no duplication of effort.

The LLVM project is the canonical realisation of this principle. Clang (C/C++/Objective-C), Rust's `rustc`, Swift's compiler, Zig, Fortran via Flang, Julia, and more than a dozen other front ends all emit LLVM IR. A single family of optimisation passes — inlining, loop optimisation, vectorisation, scalar replacement, dead code elimination — then transforms that IR before the target-specific backends lower it to machine code. The resulting ecosystem is as close to N+M as a production compiler can achieve.

### 9.1.3 Properties of a good IR

[Cooper/Torczon §5.2] identify four key properties that a good IR must satisfy:

**1. Expressiveness.** The IR must be able to represent every source concept that matters for the compiler's correctness and optimisation goals. A C compiler's IR must represent pointer arithmetic, volatile accesses, restrict annotations, atomic operations, and the undefined-behaviour assumptions that permit aggressive optimisation (e.g., that signed integer overflow cannot occur). An IR that elides any of these forces the optimiser to be overly conservative.

**2. Analysability.** The structure of the IR should make classical analyses — dominance, liveness, reaching definitions, available expressions — straightforward and efficient to implement. SSA form is the prime example: the unique-definition invariant transforms reaching-definitions analysis from a fixed-point iteration over bitsets into a trivial structural property (every use has a unique, syntactically-visible definition), reducing the analysis to a single scan rather than an iterative solve.

**3. Transformability.** Passes must be able to create, modify, and delete IR nodes cheaply and correctly. LLVM's `Value`/`Use` graph, in which every use of a value records a back-pointer to that value, makes replacing all uses of one value with another a single traversal of the use list — the `replaceAllUsesWith` operation. This enables clean implementation of transformations like constant folding, inlining, and loop transformations without fragile manual bookkeeping.

**4. Proximity to the machine.** An IR that is too high-level (retaining, say, object-oriented dispatch tables, garbage-collected heap references, or source-language exception semantics) forces every target backend to re-implement high-level lowering. An IR too close to a specific machine (encoding x86 condition codes, ARM predicated execution, or MIPS delay slots) cannot serve as an abstraction over multiple targets. The sweet spot — exemplified by LLVM IR — is a typed assembly language with an infinite set of virtual registers, explicit control flow via labeled basic blocks, no machine-specific details, and a type system rich enough to express pointer provenance but simple enough to lower to any ISA.

### 9.1.4 Historical IRs

The lineage of production IRs illuminates the design choices behind modern systems:

**P-code (UCSD Pascal, 1975).** The UCSD Pascal compiler generated P-code, a bytecode for a hypothetical "P-machine" — essentially a stack machine with opcodes like `LOAD`, `STORE`, `ADD`, and `CALL`. It was the first widely-deployed IR-based portable compiler, enabling Pascal to run on dozens of microprocessors with a single front end. Its stack-based semantics made code generation straightforward but analysis and optimisation difficult: there are no named values, only a stack depth and a position counter.

**RTL (GCC, 1984–).** GCC's Register Transfer Language encodes machine instructions in a LISP-like s-expression format at a level just above actual machine code. It exposes registers (initially virtual, later physical after register allocation), condition codes, memory operations, and calling conventions explicitly. RTL predates SSA and uses explicit liveness tracking via `REG_DEAD`/`REG_UNUSED` notes. RTL remains in GCC today for the final code-generation phases, but SSA-based GIMPLE handles most optimisations.

**GIMPLE (GCC, 2004–).** Introduced to give GCC a language-independent optimisation IR above RTL, GIMPLE is a three-address code (TAC) that restricts each statement to at most three operands and prohibits nested expressions. High-level GCC representations (GENERIC, from the front end) are lowered to GIMPLE before most optimisation passes. Modern GCC converts GIMPLE to SSA form (GIMPLE-SSA) for inlining, loop optimisation, and global optimisation, mirroring LLVM's approach [GCC internals manual §12]. GIMPLE-SSA uses the same φ-function placement algorithm as Cytron et al.

**CIL/.NET (2001–).** The Common Intermediate Language is a typed, stack-based bytecode for the .NET runtime. It is richer than JVM bytecode: it supports generics natively, has a structured exception model, and includes typed references. The .NET JIT (RyuJIT) lifts CIL to an SSA-based IR called MSIL-SSA for optimisation before native code generation.

**JVM bytecode (1995–).** Like P-code, stack-based and designed for portability and bytecode verification via type inference on the stack state. JIT compilers (HotSpot, GraalVM) lift JVM bytecode to SSA (HotSpot's "C2" IR uses an SSA sea-of-nodes representation; Graal uses an explicit SSA IR) for optimisation.

**LLVM IR (2003–).** The subject of Part IV of this book. A typed, infinite-register SSA in three-address code form with a textual assembly syntax, a binary bitcode encoding, and an in-memory C++ object model. LLVM IR is the most widely-deployed SSA IR in production use today and is the IR we connect all theory in this chapter back to.

---

## 9.2 IR Taxonomy: AST, Three-Address Code, CPS, ANF, SSA

### 9.2.1 Abstract Syntax Tree

An **AST** is a tree-structured representation of the program in which each interior node is an operator (addition, function call, if-then-else, dereference) and each leaf is a value (identifier, integer literal, string literal). The tree structure directly encodes the syntactic hierarchy of the source program, closely mirroring the grammar from which it was produced.

Advantages: the tree structure mirrors the grammar, making it easy to implement tree-walking analyses. Type checking (Chapter 8, §8.5) is a natural tree walk assigning types to nodes bottom-up. Name resolution is a tree walk that resolves identifier nodes to declarations. Constant folding can be applied during tree construction. The Visitor pattern [Chapter 8, §8.7] provides a clean, extensible traversal mechanism for implementing multiple analyses over the same tree.

Disadvantages: control flow is implicit. To determine the successors of an `if` node you must inspect its branches; there is no explicit control-flow graph. Dataflow analysis requires auxiliary bookkeeping — you must track which definitions reach which uses by correlating tree positions with block boundaries. The recursive structure is hard to emit directly as a flat instruction sequence without additional state, and is poorly suited to global optimisations that must reason about paths through the program.

Clang uses the AST through the entire front end (parsing, semantic analysis, constant evaluation, code completion, clang-tidy, and AST matchers) and lowers it to LLVM IR in the CodeGen phase (Chapters 39–40). Swift's SIL (Swift Intermediate Language) starts life close to the AST and is progressively lowered through multiple SIL stages before reaching LLVM IR. Clang's AST is unusually rich: it stores the original syntactic tokens alongside the semantic nodes, enabling high-fidelity source reconstruction needed for refactoring tools and diagnostics.

### 9.2.2 Three-Address Code

**Three-address code** (TAC) is a flat sequence of instructions, each of the form:

```
result := operand₁  op  operand₂
```

where *result*, *operand₁*, *operand₂* are either variable names, temporaries, or literals. The "three addresses" are the result and the two operands. Unary operations use two addresses; branches, copies, and jumps use fewer. No nesting is permitted: the expression `a + b * c` becomes two TAC instructions: `t1 := b * c` followed by `t2 := a + t1`. This flattening is the defining property: every intermediate result is explicitly named and stored.

The classic **quads** representation [Dragon §6.2] stores each TAC instruction as a record with four fields: (operator, arg1, arg2, result). Quads are easy to read, reorder, and transform; they trivially linearise as an array; and the fixed-width record format allows random access by instruction index (important for backpatching — see Chapter 8, §8.4).

TAC is very close to LLVM IR. The key differences:
- LLVM IR is **typed**: every value has a type, and the IR verifier enforces type correctness.
- LLVM IR is **in SSA form**: each name appears on the left-hand side exactly once.
- LLVM IR has **explicit basic blocks** with named labels and explicit terminator instructions (`br`, `ret`, `switch`).
- LLVM IR has a **rich instruction set** including `getelementptr` for pointer arithmetic, `select` for conditional moves, and `call`/`invoke` for function calls with exception-handling semantics.

Before SSA construction, a naive TAC allows re-assignment of variables (`x := 1; ... x := 2`), which is the common case when lowering from an AST. After SSA, each name appears on the left-hand side exactly once, and re-assignment is replaced by distinct versioned names.

### 9.2.3 Continuation-Passing Style

**Continuation-Passing Style** (CPS) is a program transformation in which every function takes an additional argument called the **continuation**: a function representing "what to do with the result". No function ever returns in the ordinary sense; instead, it calls its continuation with its result. Every call is a tail call, so the implicit call stack vanishes — the continuation chain encodes it explicitly.

```
-- Direct style:
let rec fact n =
  if n = 0 then 1 else n * fact (n-1)

-- CPS transformation:
let rec fact_k n k =
  if n = 0 then k 1
  else fact_k (n-1) (fun r -> k (n * r))
```

In the CPS version, `k` is the continuation. When `n = 0`, we call `k 1` — we pass 1 to whoever wanted the result of `fact`. For `n > 0`, we recurse with a new continuation `(fun r -> k (n * r))` that, when given the result `r` of `fact (n-1)`, multiplies it by `n` and passes the product to the outer continuation `k`.

CPS was pioneered for ML compilers by Appel [*Compiling with Continuations*, 1992] and for Scheme by Steele & Sussman. Its virtues for compilation:

- **Control flow is explicit.** Every jump is a tail call to a continuation; the "program counter" is the identity of the current continuation. Branch targets, exception handlers, and loop back-edges are all first-class values.
- **No implicit stack.** The call stack is encoded in the chain of continuation closures. This makes stack-manipulating optimisations (tail-call elimination, trampolining) trivial.
- **Optimisations are λ-calculus transformations.** Function inlining is β-reduction; dead code elimination is η-reduction; hoisting is an instance of λ-lifting. The well-developed theory of the λ-calculus applies directly.
- **Uniform representation of first-class functions and control flow.** CPS naturally handles coroutines, generators, and first-class continuations (i.e., call/cc) as ordinary functions.

CPS is expensive in space without aggressive optimisation: it allocates closure records for every continuation. Practical CPS compilers use escape analysis (to stack-allocate non-escaping closures), closure representation optimisation, and the CPS–direct-style retransformation to eliminate most allocation overhead. The deep connection between CPS and SSA is developed in Section 9.9.

### 9.2.4 Administrative Normal Form

**Administrative Normal Form** (ANF) was introduced by Flanagan, Sabry, Duba, and Felleisen [PLDI 1993] as a simplification of CPS that preserves evaluation order and naming of intermediate results without the syntactic overhead of explicit continuation arguments. In ANF, the arguments to every function call must be **values** (variables or literals, not compound expressions), and every intermediate computation is explicitly named via a `let`:

```
-- Direct style:
f (g x) + h y

-- ANF:
let t1 = g x in
let t2 = h y in
let t3 = f t1 in
t3 + t2
```

Every subexpression is bound to a name; no compound expression appears as an argument to a call or an operand of an operator. The operational semantics is simple: evaluation never needs to look inside a value, only to match names.

ANF is "flatter" than direct style (all subexpressions are named) but less syntactically noisy than full CPS (no explicit continuation arguments appear). It has been used as the IR in several Scheme and ML compilers (SML/NJ's closure-conversion IR, Rabbit, and others) and has directly influenced the design of LLVM IR, where the SSA property corresponds to the ANF naming discipline.

The key relationship: ANF is CPS without the explicit continuation argument. Every ANF term `let x = e₁ in e₂` corresponds to a CPS term `e₁_k (λx. e₂_k)` for some continuation `k`. The correspondence is exact: ANF can be mechanically derived from CPS by removing the continuation argument and making the sequential evaluation order implicit in the `let` nesting.

### 9.2.5 Static Single Assignment Form

**SSA form** [Cytron et al. 1991] is a TAC in which each variable name is **assigned exactly once** in the program text. When a variable's value differs along different control-flow paths and those paths merge at a join point, a special **φ-function** (phi-function) appears at the beginning of the join block to select the appropriate incoming value:

```
; Before SSA form:          ; After SSA construction:
  x = 0                       x0 = 0
  if p goto L                 if p goto L
  x = 1                       x1 = 1
L:                          L:
  y = x + 1                   x2 = φ(x0, x1)  ; from fall-through or branch
                               y0 = x2 + 1
```

The φ-function `x2 = φ(x0, x1)` is a pseudo-instruction at the join point L: it selects x0 if control reached L by falling through the `if` (the else-path) or x1 if control reached L via the `goto L` branch (the then-path). Precisely one φ-operand is "selected" at any given runtime — the one corresponding to the predecessor block that was actually executed.

SSA's benefits over plain TAC are profound and justify the construction cost entirely:

- **Reaching definitions become trivial.** Because each name is defined exactly once, the definition of any use is the unique point where that name appears on the left-hand side. There is no need for a fixed-point reaching-definitions computation. This eliminates an entire class of iterative dataflow algorithms — their results are directly readable from the SSA structure.
- **Dead code elimination is a single DFS.** A definition is dead iff its use list is empty. The use-def chains built into SSA (each value records all its uses) make this a linear-time single pass, not a fixed-point iteration.
- **Global Value Numbering is efficient.** Redundant computations can be detected by hashing SSA values: two instructions computing the same operation on the same SSA-versioned operands are identical and one can be eliminated. Without SSA, determining that two uses of `x` refer to the same definition requires reaching-definitions analysis.
- **Register allocation via graph coloring is precise.** Interference between two values exists only if their live ranges overlap. In SSA, each value's live range begins at its unique definition and ends at its last use; the ranges are immediately computable from the def-use chains and the dominance tree.
- **Many classical optimisation algorithms simplify dramatically on SSA.** Constant propagation (SCCP), loop-invariant code motion (LICM), partial redundancy elimination (PRE), and strength reduction all have cleaner, faster implementations on SSA than on raw TAC.

---

## 9.3 The SSA Invariant: Formal Definition and Intuition

### 9.3.1 Formal definition

Let P be a program represented as a control-flow graph G = (V, E, entry) where V is the set of basic blocks, E ⊆ V × V is the set of control-flow edges, and entry ∈ V is the entry block. Let Vars(P) be the set of program variables. For each variable v ∈ Vars(P), let Defs(v) ⊆ V be the set of basic blocks that contain an assignment to v (φ-functions count as assignments).

**Definition (SSA form) [Cytron et al. 1991, §1].** The program P is in *SSA form* iff for every variable v ∈ Vars(P), |Defs(v)| = 1 — every variable is assigned in exactly one basic block.

When a variable's value must merge from multiple predecessors at a join point B (a block with ≥2 predecessors), a **φ-function**

```
v_new = φ(v₁, v₂, ..., vₖ)
```

is inserted at the head of B, where k = |pred(B)| is the number of predecessors and v_i is the version of v contributed by the i-th predecessor B_i. The φ-function is itself a definition of v_new, and this definition is the only assignment to v_new in the entire program.

The semantics: the φ-function assigns v_new the value v_j where j is the index of the predecessor B_j that was the last block to execute before entering B. Since SSA is a static analysis tool, the φ-function does not need to be implemented as a machine instruction — it is resolved during out-of-SSA conversion (Section 9.8) by inserting copies in each predecessor block.

### 9.3.2 The dominance condition and strict SSA

**Definition (strict SSA form).** P is in *strict SSA form* iff it is in SSA form and additionally, for every variable v and every use of v at program point p, the unique definition of v *dominates* p.

The dominance condition (dominance is defined formally in Section 9.4) ensures that every use of a variable is always reached by a well-defined definition. It rules out uses of uninitialised variables and ensures that the SSA use-def chains are acyclic (no use precedes its definition on every execution path).

It is possible to construct SSA form without the dominance condition — this arises when variables are used before they are necessarily defined along all paths (e.g., `y = x + 1` when `x` might be uninitialised on one path). Such SSA programs are technically valid but semantically ill-formed. LLVM IR requires strict SSA: the verifier rejects any function in which a use is not dominated by its definition, treating such uses as an error rather than representing "potentially uninitialised."

The relationship between the two conditions:
- SSA form alone guarantees |Defs(v)| = 1.
- Strict SSA additionally guarantees that every use-def path is non-empty (the definition exists and dominates every use).

All SSA forms produced by the Cytron algorithm (Section 9.5) are strict, because the renaming pass traverses the dominator tree and only inserts definitions that dominate their uses by construction.

### 9.3.3 Static vs. dynamic assignment

The word "static" in SSA is important and often misunderstood. Consider a loop:

```
i0 = 0              ; i is "defined" once statically, before the loop
loop:
  i1 = φ(i0, i2)   ; join of pre-loop value and loop-back value
  ... use i1 ...
  i2 = i1 + 1       ; i is "incremented" here, once statically
  if i2 < N goto loop
```

There are exactly two static definitions: `i1 = φ(i0, i2)` and `i2 = i1 + 1`. Yet at runtime, if the loop executes N times, `i2` is assigned N times — once per iteration — to different *dynamic instances* of the same static definition site. SSA is a *static* property of the program text: each name is unique in the source, but multiple dynamic instances of the same static name may exist at runtime [Cytron et al. 1991, §1]. This is precisely why SSA is compatible with loops, recursion, and any iterative computation.

The distinction matters for register allocation: two live ranges of i2 that overlap in time (during pipelining or speculative execution) can still use the same register if the compiler can prove they do not simultaneously require distinct physical storage. The SSA static-assignment view gives a conservative but useful starting point for this analysis.

### 9.3.4 Use-def and def-use chains in SSA

In ordinary TAC, building **use-def chains** (for each use of a variable, pointing to all definitions that might reach it) requires a reaching-definitions dataflow analysis. In SSA, the use-def chain for any use of v is trivially the unique definition of v — found by following the variable's version subscript to its single definition site. This is an O(1) lookup per use.

**Def-use chains** (for each definition of a variable, the set of all uses of that definition) are explicitly maintained in LLVM's `Value`/`Use` data structure: each `Value` object holds a linked list of `Use` records, one per use. This makes "replace all uses of x with y" an O(number of uses) operation — no worklist, no recomputation.

The simplicity of use-def and def-use chains in SSA is the single most important reason why SSA-based optimisers are faster and simpler than their pre-SSA counterparts, and why SSA has completely displaced older IR forms in production compilers [EaC §9.1].

---

## 9.4 Dominator Trees and Dominance Frontiers

### 9.4.1 Dominators and the dominator tree

**Definition (dominance).** Block A *dominates* block B (written A dom B) iff every path from the entry block to B passes through A. Dominance is:
- *Reflexive*: A dom A (every block dominates itself).
- *Antisymmetric*: if A dom B and B dom A, then A = B.
- *Transitive*: if A dom B and B dom C, then A dom C.

Therefore dom is a partial order on V.

**Definition (strict dominance).** A *strictly dominates* B (written A sdom B) iff A dom B and A ≠ B.

**Definition (immediate dominator).** The *immediate dominator* of B, written idom(B), is the unique block A ≠ B such that A sdom B and there is no block C with A sdom C and C sdom B. Every block except the entry has a unique immediate dominator; the entry has none [Lengauer–Tarjan 1979].

**Definition (dominator tree).** The *dominator tree* T_D of G is the rooted tree with nodes V in which each node B has idom(B) as its parent (the entry is the root). T_D is a tree (not a DAG) because immediate dominators are unique. The key property: A dom B iff A is an ancestor of B in T_D.

The dominator tree is the central data structure for SSA construction and for many subsequent analyses. Its depth equals the maximum nesting depth of the control-flow graph, which is bounded by the number of natural loops plus one for typical programs — usually a small constant [EaC §9.2].

### 9.4.2 Computing dominator trees

Two algorithms are standard:

**Cooper–Harvey–Kennedy (2001).** A simple iterative algorithm [Cooper et al. 2001]:

```
algorithm compute_domtree(CFG):
  idom[entry] := entry
  changed := true
  while changed:
    changed := false
    for B in reverse-post-order(CFG) \ {entry}:
      new_idom := first processed predecessor of B
      for each other predecessor P of B that has been processed:
        new_idom := intersect(P, new_idom, idom)
      if idom[B] ≠ new_idom:
        idom[B] := new_idom
        changed := true

function intersect(b1, b2, idom):
  // climb both fingers up toward entry until they meet
  finger1 := b1
  finger2 := b2
  while finger1 ≠ finger2:
    while rpo_number[finger1] > rpo_number[finger2]:
      finger1 := idom[finger1]
    while rpo_number[finger2] > rpo_number[finger1]:
      finger2 := idom[finger2]
  return finger1
```

The `intersect` function computes the least common ancestor in the (partially-built) dominator tree by climbing both fingers simultaneously. The RPO numbering ensures that "earlier in RPO" means "closer to entry," so the finger-climbing terminates at the nearest common ancestor. The outer loop converges in at most two passes on reducible CFGs (CFGs without irreducible loops) [EaC §9.2], making the algorithm O(n²) worst-case but fast in practice.

**Lengauer–Tarjan (1979).** A theoretically superior algorithm [Lengauer–Tarjan 1979] that runs in nearly linear O(n α(n)) time, where α is the inverse Ackermann function. It uses DFS to compute a spanning tree and then computes *semi-dominators* (a weaker version of dominators along DFS tree paths) and uses them to derive immediate dominators. It is required for pathological CFGs with many irreducible loops or extreme depth, but benchmarks consistently show Cooper–Harvey–Kennedy to be faster in practice due to better cache behaviour.

LLVM uses a variant of Cooper–Harvey–Kennedy in `DominatorTree.cpp`, wrapped in the `DominatorTreeBase<BasicBlock>` class and computed in Reverse Post Order. The LLVM dominator tree is incrementally maintained: when a pass inserts or removes a CFG edge, it can call `DominatorTree::insertEdge` or `deleteEdge` to update the tree in O(n) time rather than recomputing from scratch.

### 9.4.3 Dominance frontiers

The **dominance frontier** of block A, DF(A), captures exactly where A's dominance "stops" — the set of blocks that A nearly dominates but does not strictly dominate:

**Definition [Cytron et al. 1991, §3].** DF(A) = { B ∈ V | ∃ predecessor P of B such that A dom P, and A ¬sdom B }.

In words: B is in DF(A) iff A dominates some predecessor of B (so A's influence "reaches" B) but A does not strictly dominate B itself (so A's dominance "stops" at B). The dominance frontier is the set of blocks that lie at the boundary of A's dominance.

**Intuition for SSA.** A variable v is defined in block A. Every block that A strictly dominates receives the value of v from A unambiguously — no alternative definition of v can reach those blocks without passing through A. But at a block B ∈ DF(A), there is a path to B that does *not* pass through A (arriving via a predecessor of B that A does not dominate). At B, two different versions of v — one from A's computation path and one from another definition — converge. This convergence is precisely the condition that requires a φ-function.

**Computing DF.** The standard O(|V| + |E|) algorithm processes every CFG edge:

```
algorithm compute_dominance_frontiers(CFG, idom):
  DF := {} for all blocks
  for each block B with |pred(B)| >= 2:   // B is a join point
    for each predecessor P of B:
      runner := P
      while runner ≠ idom(B):
        DF[runner].add(B)
        runner := idom[runner]
  return DF
```

This algorithm walks up the dominator tree from each join point: for each predecessor P of a join block B, every block on the dominator-tree path from P up to (but not including) idom(B) has B in its dominance frontier. The correctness is immediate from the definition: such a block "runner" dominates P (because P is a descendant of runner in T_D), and runner does not strictly dominate B (because idom(B) strictly dominates runner, and dominance is transitive — if runner sdom B, then idom(B) would be on the path from runner to B, but we stop before idom(B)).

The decomposition into *local* and *up* contributions [EaC §9.3] is useful for implementation:

```
DF_local(B) = { S | B→S ∈ E  and  B ¬sdom S }
DF_up(B)    = { S ∈ DF(C)  for some child C of B in T_D  |  B ¬sdom S }
DF(B)       = DF_local(B) ∪ DF_up(B)
```

Computing DF bottom-up in T_D: first compute DF_local for all blocks, then fold in DF_up from children. This bottom-up order ensures that a child's DF is complete before the parent's DF_up contribution is computed.

**Loop headers are in their own dominance frontier.** A loop header H that has a back-edge from some block B to H satisfies DF(H) ∋ H, because H dominates B (B is in the loop body) but H does not strictly dominate itself. This self-loop in the DF is what causes φ-functions for loop-carried variables to appear at loop headers — exactly the right place.

---

## 9.5 The Cytron Algorithm: IDF, φ-Placement, and Renaming

### 9.5.1 Why DF is not enough: the need for IDF

Placing φ-functions at DF(Defs(v)) is correct for variables defined in exactly two or more blocks when those definitions do not themselves create new join points. But consider: placing a φ-function at block B ∈ DF(Defs(v)) introduces a *new definition* of v at B. This new definition may require additional φ-functions at DF(B) — blocks where B's dominance meets paths from other definitions.

A simple example: suppose v is defined in blocks D1, D2, D3 arranged so that DF(D1) = {J1}, DF(D2) = {J1}, and DF(J1) = {J2}. After placing a φ at J1, we have a new definition of v at J1, so we must also check DF(J1) = {J2} for a required φ. This propagation may cascade.

The **iterated dominance frontier** (IDF) formalises this fixed-point:

**Definition.** For a set S ⊆ V:
```
DF¹(S)   = DF(S)  =  ⋃_{B ∈ S} DF(B)
DFⁿ⁺¹(S) = DF(DFⁿ(S) ∪ S)
DF⁺(S)   = ⋃_{n≥1} DFⁿ(S)   (the IDF of S)
```

The sequence DF¹(S) ⊆ DF²(S) ⊆ ... is monotone and bounded by V, so it stabilises — DF⁺(S) is the least fixed point of the operator T ↦ DF(S ∪ T).

**Theorem [Cytron et al. 1991, Theorem 3.1].** A variable v requires a φ-function at block B if and only if B ∈ DF⁺(Defs(v)).

*Proof sketch (necessity).* If v requires a φ-function at B, then there exist two definitions of v, say in blocks D_i and D_j, and two paths from D_i and D_j to B that do not share a definition of v. These paths "merge" at B, which by the structure of the CFG and dominance implies B ∈ DF(D_i) or B ∈ DF of some block reachable from D_i or D_j — i.e., B ∈ DF⁺(Defs(v)).

*Proof sketch (sufficiency).* If B ∈ DF⁺(Defs(v)), there is a chain of dominance-frontier steps from some definition of v to B. At each step, a new merge point is encountered where two paths with potentially distinct values of v converge. At B, the merging of these paths requires a φ-function to record which path delivered the current value.

**IDF computation with a worklist.** The following algorithm computes DF⁺(S) correctly:

```
algorithm IDF(S):
  // S is the initial set of definition blocks
  result   := {}
  worklist := copy of S
  in_worklist := {B | B ∈ S}

  while worklist ≠ {}:
    X := remove_any(worklist)
    for B in DF(X):
      if B ∉ result:
        result.add(B)
        // B is now a new definition site due to φ-placement;
        // its own DF must be explored
        if B ∉ in_worklist:
          worklist.add(B)
          in_worklist.add(B)

  return result
```

The `in_worklist` set prevents adding the same block to the worklist twice, ensuring termination. The algorithm visits each block in DF⁺(S) at most once, so its time complexity is O(|DF⁺(S)| × max_deg) where max_deg is the maximum number of entries in any single DF set — O(|V|²) worst case, O(|V| + |E|) in practice.

### 9.5.2 φ-function placement

Given the IDF, φ-function placement is straightforward:

```
algorithm place_phi_functions(CFG, variables):
  for each variable v:
    phi_blocks := IDF(Defs(v))
    for B in phi_blocks:
      if no φ for v already at head of B:
        insert at head of B: v = φ(v, v, ..., v)
        // one undifferentiated argument per predecessor; renaming fills them
```

Note that the φ-functions inserted here have *placeholder* arguments — all copies of the original name `v`. The renaming pass (Section 9.5.3) fills in the correct versioned names.

**The φ-function is its own definition.** When we place `v = φ(...)` at B, B itself becomes a member of Defs(v). The IDF worklist algorithm in Section 9.5.1 accounts for this by adding B to the worklist when it is first identified as a φ-placement site, so DF(B) is also explored.

### 9.5.3 Variable renaming

After φ-placement, every variable v may have multiple definitions (original assignments plus φ-functions inserted at IDF blocks) but they reside in different blocks. The **renaming pass** assigns a unique versioned name to every definition and updates all uses to refer to the correct version.

The algorithm performs a pre-order DFS of the dominator tree, maintaining a per-variable stack of "current version names." The invariant is: when the DFS visits block B, the top of Stack(v) holds the name of the definition of v that dominates the entry to B. This invariant is maintained by pushing new names when definitions are encountered and popping them when the DFS returns from a block.

The rename algorithm [Cytron et al. 1991, §4]:

```
algorithm rename(B):
  // Phase 1: rename definitions (left-hand sides)
  for each φ-function "v = φ(...)" in B, in program order:
    i := fresh_index(v)             // e.g., 3 if v has been defined 3 times so far
    rename lhs of φ to v_i
    Stack(v).push(v_i)

  // Phase 2: rename ordinary instructions
  for each non-φ instruction in B, in program order:
    // Rename uses first (right-hand side):
    for each use of variable v in instruction:
      replace v with Stack(v).top()
    // Then rename the definition (left-hand side), if any:
    if instruction defines variable v:
      i := fresh_index(v)
      rename lhs to v_i
      Stack(v).push(v_i)

  // Phase 3: fill in φ-arguments in CFG successors
  for each CFG successor S of B:
    j := index of B in predecessor_list(S)
    for each φ-function "x_? = φ(...)" in S:
      S.φ[x].args[j] := Stack(x).top()

  // Phase 4: recurse into dominator-tree children
  for each child C of B in dominator tree T_D:
    rename(C)

  // Phase 5: restore stacks (pop everything pushed in phases 1 and 2)
  for each push performed in phases 1 and 2:
    Stack(v).pop()
```

**Why the dominator-tree DFS works.** The dominator tree DFS visits block B before all blocks that B dominates. The invariant that Stack(v).top() holds the correct definition of v at the entry to B is maintained because:
- When we descend into B from B's dominator-tree parent, the parent has already updated the stacks (Phase 2 of the parent) so they reflect all definitions that dominate B.
- When we process B's instructions, we push new definitions, updating the stacks to reflect definitions within B for B's dominated children.
- Phase 5 restores the stacks after we return from B's subtree, so B's parent and B's siblings see a clean stack.

**Handling φ-function arguments (Phase 3).** The φ-function arguments are filled in during Phase 3, *after* the block's own instructions are processed, using the current stack tops. This is correct: the value passed to a φ-function via the edge B→S is the value of the variable at the end of B's execution, which is exactly Stack(v).top() after processing B's instructions.

**Complexity.** The placement step uses the IDF worklist and runs in O(|E| + |V| log |V|) time on typical programs. The renaming step performs one DFS of the dominator tree — O(|V| + |E|) — with O(|V|) total push/pop operations per variable. The total time for SSA construction is O(|V| + |E| + |Vars| × |V|) in the worst case but O(|V| + |E|) amortised on programs with few definition sites per variable.

---

## 9.6 Worked Example: SSA Construction for a Loop CFG

We work through the complete SSA construction for the following six-block CFG, which encodes a loop with a conditional increment — a representative and non-trivial example:

```
entry:
  x = 0
  y = 1
  goto header

header:                        (successors: body, exit)
  if x < 10 goto body else goto exit

body:                          (successors: then, merge)
  if y > 5 goto then else goto merge

then:                          (successor: merge)
  y = y + 2

merge:                         (successor: header)
  x = x + 1
  goto header

exit:
  return x, y
```

### 9.6.1 The control-flow graph

| Block  | Predecessors        | Successors      |
|--------|---------------------|-----------------|
| entry  | —                   | header          |
| header | entry, merge        | body, exit      |
| body   | header              | then, merge     |
| then   | body                | merge           |
| merge  | body, then          | header          |
| exit   | header              | —               |

Variables and their pre-SSA definition sites:

| Variable | Defs(v)           |
|----------|-------------------|
| x        | {entry, merge}    |
| y        | {entry, then}     |

### 9.6.2 Step 1: Dominator tree

Computing immediate dominators using Cooper–Harvey–Kennedy (RPO: entry=0, header=1, body=2, then=3, merge=4, exit=5):

After one pass of the iterative algorithm:

| Block  | idom(B) |
|--------|---------|
| entry  | entry   |
| header | entry   |
| body   | header  |
| then   | body    |
| merge  | body    |
| exit   | header  |

The dominator tree (parent → children in T_D):

```
entry
  └── header
        ├── body
        │     ├── then
        │     └── merge
        └── exit
```

**Verification.** Every path from entry to merge must pass through: entry (trivially), then header (both entry→header and any loop iteration goes through header before body), then body (header→body is the only path into body). So idom(merge) = body. ✓

### 9.6.3 Step 2: Dominance frontiers

Applying the edge-walking algorithm to each CFG edge:

| Edge            | runner path                     | Result                           |
|-----------------|---------------------------------|----------------------------------|
| entry→header    | runner=entry; idom(header)=entry; entry=entry → stop | DF(entry) += nothing |
| header→body     | runner=header; idom(body)=header; runner=idom(header) → stop | nothing |
| header→exit     | runner=header; idom(exit)=header → stop | nothing |
| body→then       | runner=body; idom(then)=body → stop | nothing |
| body→merge      | runner=body; idom(merge)=body → stop | nothing |
| then→merge      | runner=then; idom(merge)=body; then≠body → **DF(then)∋merge**; runner=idom(then)=body; body=body → stop | DF(then) = {merge} |
| merge→header    | runner=merge; idom(header)=entry; merge≠entry → **DF(merge)∋header**; runner=idom(merge)=body; body≠entry → **DF(body)∋header**; runner=idom(body)=header; header≠entry → **DF(header)∋header**; runner=idom(header)=entry; entry=entry → stop | DF(merge)={header}, DF(body)={header}, DF(header)={header} |

Final dominance frontiers:

| Block  | DF(B)          | Interpretation                                              |
|--------|----------------|-------------------------------------------------------------|
| entry  | {}             | Entry dominates everything; no boundary                    |
| header | {header}       | Loop header: back-edge creates self-loop in DF             |
| body   | {header}       | Body's code converges at the loop header                   |
| then   | {merge}        | Then-branch merges with fall-through at merge              |
| merge  | {header}       | After incrementing, merges back into loop at header        |
| exit   | {}             | Sink block; no successors                                  |

### 9.6.4 Step 3: IDF computation and φ-placement

**Variable x.** Defs(x) = {entry, merge}.

```
worklist = {entry, merge},  result = {}
Process entry:  DF(entry) = {} → nothing new
Process merge:  DF(merge) = {header} → header ∉ result
                              add header to result and worklist
                result = {header}
Process header: DF(header) = {header} → header ∈ result already → done
IDF(Defs(x)) = {header}
```

→ Insert `x = φ(x, x)` at **header** (one arg from entry, one from merge).

**Variable y.** Defs(y) = {entry, then}.

```
worklist = {entry, then},  result = {}
Process entry: DF(entry) = {} → nothing
Process then:  DF(then) = {merge} → merge ∉ result
                             add merge to result and worklist
               result = {merge}
Process merge: DF(merge) = {header} → header ∉ result
                             add header to result and worklist
               result = {merge, header}
Process header: DF(header) = {header} → header ∈ result → done
IDF(Defs(y)) = {merge, header}
```

→ Insert `y = φ(y, y)` at **merge** (one arg from body, one from then) and `y = φ(y, y)` at **header** (one arg from entry, one from merge).

φ-placement summary:

| Block  | φ-functions inserted                                     |
|--------|----------------------------------------------------------|
| header | `x = φ(_, _)` [entry, merge], `y = φ(_, _)` [entry, merge] |
| merge  | `y = φ(_, _)` [body, then]                               |

### 9.6.5 Step 4: Variable renaming

DFS pre-order of T_D: entry → header → body → then → merge → exit. Initial stacks empty.

**rename(entry):**
- Phase 1: no φ-functions.
- Phase 2: `x = 0` → def x: push **x₀**. Stack(x)=[x₀]. `y = 1` → def y: push **y₀**. Stack(y)=[y₀].
- Phase 3: successor header (position=0 from entry):
  - header's x-φ arg[0] = top(Stack(x)) = x₀
  - header's y-φ arg[0] = top(Stack(y)) = y₀
- Phase 4: recurse into header.

**rename(header):**
- Phase 1: φ `x = φ(x₀, _)` → push **x₁**; φ `y = φ(y₀, _)` → push **y₁**. Stack(x)=[x₀,x₁], Stack(y)=[y₀,y₁].
- Phase 2: `if x < 10` → use x → replace with x₁. No defs.
- Phase 3: successors body (no φ) and exit (no φ).
- Phase 4: recurse into body, then exit.

**rename(body):**
- Phase 1: no φ-functions at body.
- Phase 2: `if y > 5` → use y → replace with **y₁**. No defs.
- Phase 3: successor merge (position=0 from body): merge's y-φ arg[0] = y₁.
  Successor then: no φ.
- Phase 4: recurse into then, then merge.

**rename(then):**
- Phase 1: no φ-functions.
- Phase 2: `y = y + 2` → use y → replace with **y₁**; def y → push **y₂**. Stack(y)=[y₀,y₁,y₂].
- Phase 3: successor merge (position=1 from then): merge's y-φ arg[1] = y₂.
- Phase 4: no children.
- Phase 5: pop y₂. Stack(y)=[y₀,y₁].

**rename(merge):**
- Phase 1: φ `y = φ(y₁, y₂)` → push **y₃**. Stack(y)=[y₀,y₁,y₃].
- Phase 2: `x = x + 1` → use x → replace with x₁; def x → push **x₂**. Stack(x)=[x₀,x₁,x₂].
- Phase 3: successor header (position=1 from merge):
  - header's x-φ arg[1] = top(Stack(x)) = **x₂** → x₁ = φ(x₀, x₂) ✓
  - header's y-φ arg[1] = top(Stack(y)) = **y₃** → y₁ = φ(y₀, y₃) ✓
- Phase 4: no children.
- Phase 5: pop x₂, y₃. Stack(x)=[x₀,x₁], Stack(y)=[y₀,y₁].

**rename(exit):**
- Phase 2: `return x, y` → replace x with **x₁**, y with **y₁**.

After all recursive returns: pop header's pushes (x₁, y₁), then entry's pushes (x₀, y₀). Stacks empty. ✓

### 9.6.6 Final SSA form

```
entry:
  x0 = 0
  y0 = 1
  goto header

header:
  x1 = φ(x0, x2)         ; x0 from entry, x2 from merge
  y1 = φ(y0, y3)         ; y0 from entry, y3 from merge
  if x1 < 10 goto body else goto exit

body:
  if y1 > 5 goto then else goto merge

then:
  y2 = y1 + 2
  goto merge

merge:
  y3 = φ(y1, y2)         ; y1 from body (no increment), y2 from then (incremented)
  x2 = x1 + 1
  goto header

exit:
  return x1, y1
```

**Verification checklist:**
- Every variable (x0, x1, x2, y0, y1, y2, y3) appears on the LHS exactly once. ✓
- Every use is dominated by its definition:
  - x1 used in body (x1 defined at header; header dom body ✓)
  - x1 used in merge (header dom body dom merge ✓)
  - y1 used in body (header dom body ✓)
  - y3 used in header φ: y3 defined in merge; merge→header is the back-edge; the φ semantics of y1 = φ(y0, y3) ensure y3 is read only when control arrives from merge. ✓
- φ-function argument counts match predecessor counts:
  - header has 2 predecessors (entry, merge) → 2-argument φ. ✓
  - merge has 2 predecessors (body, then) → 2-argument φ. ✓

---

## 9.7 Pruned, Minimal, and Semi-Pruned SSA

### 9.7.1 Minimal SSA

The Cytron algorithm produces **minimal SSA**: φ-functions are placed at exactly IDF(Defs(v)) for each variable v, with no further filtering. "Minimal" here means: the fewest φ-functions consistent with the SSA invariant, given only the CFG structure. Minimal SSA ignores liveness.

A φ-function at block B for variable v is *useless* (or *dead*) if v is not live at the entry to B — if no path from B to any use of v exists without an intervening definition of v. Minimal SSA may contain such useless φ-functions. For example, if x is computed in entry and merge but is never used after header, the φ-function `x1 = φ(x0, x2)` at header is useless. Minimal SSA places it anyway because it cannot distinguish liveness from join-point structure.

Useless φ-functions waste space and slow down analyses that traverse all instructions. However, they are easily eliminated by a subsequent dead-code elimination pass, which is why minimal SSA is the standard starting point.

### 9.7.2 Pruned SSA

**Pruned SSA** [Briggs, Cooper, Harvey, Simpson 1998] eliminates useless φ-functions at construction time by consulting liveness:

```
Place v = φ(...) at B  iff  B ∈ IDF(Defs(v))  and  v ∈ LiveIn(B)
```

A variable v is *live at entry to B* (v ∈ LiveIn(B)) iff there is a path from B to some use of v that does not pass through any definition of v. Computing LiveIn requires a backward dataflow analysis over the CFG (the liveness equation: LiveIn(B) = Use(B) ∪ (LiveOut(B) \ Def(B))).

The chicken-and-egg tension: SSA was supposed to simplify liveness analysis, but pruned SSA requires liveness before SSA is built. The resolution is to run one pass of liveness analysis on the *pre-SSA* CFG (using original variable names, before renaming) to identify LiveIn sets. These approximate LiveIn sets may be slightly conservative (they do not benefit from SSA's precision) but are safe — they never wrongly exclude a φ-function that is needed.

Pruned SSA produces the smallest possible number of φ-functions consistent with correctness. It is the ideal form for register allocation because every live range corresponds to an actual use, and no phantom interference is introduced by dead φ-functions.

### 9.7.3 Semi-pruned SSA

**Semi-pruned SSA** [Briggs et al. 1998] provides a practical middle ground that avoids the full liveness computation. The observation: a variable v is *never live upon entry to any block other than its defining block* iff it is *purely local* — used only within the block in which it is defined. For such variables, φ-functions are never needed (no path from a definition to a use crosses a block boundary), so they can be excluded from IDF computation entirely.

```
-- Semi-pruned placement:
Place v = φ(...) at B  iff  B ∈ IDF(Defs(v))  and  v ∈ NonLocal(P)

-- Where NonLocal(P) is the set of variables that appear as uses
-- in some block before their definition in that block:
NonLocal(P) = { v | ∃ block B: v is used in B before any definition of v in B }
```

Computing NonLocal requires a single linear scan of each block — O(|V| + number of instructions) total — which is far cheaper than a full liveness analysis (which requires a fixed-point dataflow iteration).

Empirically, semi-pruned SSA eliminates the majority of useless φ-functions that minimal SSA would place, because a large fraction of compiler-generated temporaries (temporaries for sub-expressions, condition codes, address calculations) are purely local. The IR size and optimisation cost improvement over minimal SSA is substantial, while the construction overhead over minimal SSA is negligible [Briggs et al. 1998, §5].

### 9.7.4 Comparison

| Property                    | Minimal SSA          | Semi-Pruned SSA          | Pruned SSA                 |
|-----------------------------|----------------------|--------------------------|----------------------------|
| φ-function count            | Maximum              | Reduced                  | Minimum                    |
| Construction cost           | Lowest               | Very low (+linear scan)  | Medium (+ liveness pass)   |
| Requires liveness?          | No                   | Approximate only         | Yes (pre-SSA liveness)     |
| Useless φ removed           | None at construction | Most                     | All                        |
| Used by LLVM `mem2reg`      | Yes (placement step) | —                        | Approximated via DCE after |
| Practical recommendation    | + run ADCE after     | Good default             | Best for regalloc quality  |

LLVM's `mem2reg` pass (`PromoteMemToReg.cpp`) uses minimal SSA placement — the Cytron algorithm without liveness filtering — and then relies on subsequent DCE/ADCE passes to remove dead φ-functions. The net result approximates pruned SSA without a separate liveness pre-pass.

---

## 9.8 Out-of-SSA Conversion: Lost-Copy, Swap, and Correct Algorithms

At some point in the compilation pipeline — typically just before instruction selection, during register allocation, or explicitly in a separate lowering pass — SSA must be *destroyed*: φ-functions must be replaced with actual data-movement operations (copies) so the program can be executed on hardware that has no concept of φ-functions. This process is called **out-of-SSA conversion**, **SSA destruction**, or **φ-elimination**.

### 9.8.1 The naive approach and its correctness

The conceptually simplest approach: for each φ-function `v_new = φ(v₁, v₂, ..., vₖ)` at block B, insert a copy `v_new ← v_i` at the end of each predecessor block B_i (before B_i's terminator instruction). Then delete the φ-function. In isolation, this is semantically correct: the copy in B_i will assign v_new the value v_i just before jumping to B, ensuring that upon entering B, v_new holds the value v_i from predecessor B_i.

However, when multiple φ-functions coexist in the same block and their operands overlap, the naive sequential insertion of copies produces incorrect code.

### 9.8.2 The lost-copy problem

Suppose the loop in our worked example leads to the following SSA at the loop header:

```
header:
  a1 = φ(a0, b2)   ; a picks up b's old value on the back-edge
  b1 = φ(b0, a2)   ; b picks up a's old value on the back-edge
```

where a2 and b2 are defined in the loop body (say, `a2 = a1 + 1; b2 = b1 + 1` in a swap loop). On the back-edge from the loop body to header, the naive approach inserts:

```
loop-body (before the back-edge jump):
  a1 ← b2    ; copy for the first φ
  b1 ← a2    ; copy for the second φ
  goto header
```

This is wrong. After `a1 ← b2`, the location `a1` holds b2's value. The subsequent `b1 ← a2` reads a2 — but if a1 and a2 alias the same storage (or a1 and a2 are the same register), the first copy has already destroyed a1's value. The original value of a2 is lost. This is the **lost-copy problem**: a φ-operand that is also a destination in the same parallel copy set may be overwritten before it is read [Briggs et al. 1998, §3].

### 9.8.3 The swap problem

A symmetric issue arises when two φ-functions at the same block form a cyclic dependency. After certain IR transformations (e.g., inlining, loop peeling, or φ-function simplification), you can obtain:

```
B:
  a2 = φ(b2)    ; a2 should get the old value of b2
  b2 = φ(a2)    ; b2 should get the old value of a2
```

(Here a2 and b2 have only one predecessor, so this is a degenerate φ with one argument; it arises during loop-carry merging.) Naive copies:

```
predecessor of B:
  a2 ← b2    ; overwrites a2
  b2 ← a2    ; reads the *new* a2, not the original
```

The result is that both a2 and b2 get the original value of b2 — the swap is lost. The correct operation is a *parallel swap*: simultaneously read both old values and write both destinations. This is the **swap problem** [Briggs et al. 1998, §4].

**The general principle.** φ-functions at a block B must be treated as a **parallel assignment**: all right-hand-side operands are read simultaneously (using their pre-copy values) before any left-hand-side destination is written. Sequentialising a parallel assignment into a series of sequential moves requires care when there are read-write conflicts.

### 9.8.4 The Briggs–Cooper–Harvey–Reeves sequentialisation algorithm

The correct algorithm for sequentialising a set of parallel copies {dest_i ← src_i} is [Briggs et al. 1998, adapted]:

**Step 1: Build a dependency graph.** Create a directed graph G where each copy `dest ← src` is an edge dest ← src. A node `v` is a "source" if it appears as some src_i; it is a "destination" if it appears as some dest_i. An edge is a *conflict* if src_j = dest_i for some i ≠ j (i.e., a destination is also a source that would be overwritten before it is read).

**Step 2: Process non-conflicted copies first.** Any copy whose source is not overwritten (not a dest_j for any j) can be issued immediately: `dest ← src`. Remove it from the graph. Repeat until no more such copies exist.

**Step 3: Handle cycles.** The remaining copies form one or more cycles (since every remaining node is both a source and a destination). For each cycle `v₁ ← v₂ ← v₃ ← ... ← vₖ ← v₁`, emit:

```
t   ← vₖ           ; save the "tail" of the cycle in a temporary
vₖ  ← vₖ₋₁
vₖ₋₁ ← vₖ₋₂
...
v₂  ← v₁
v₁  ← t             ; complete the cycle using the saved temporary
```

This uses exactly one temporary per cycle and correctly implements the cyclic parallel assignment.

**Step 4: Repeat.** After resolving all cycles, resume Step 2 to process any copies that were previously blocked by now-resolved cycles.

The total number of copies issued is |copies| plus at most |cycles| extra copy-to-temporary instructions. For programs without circular swap patterns, no temporaries are needed and the algorithm degenerates to a simple topological sort.

### 9.8.5 Sreedhar's Method III: congruence class merging

Sreedhar and Gao [POPL 1995] proposed three progressively more powerful methods for out-of-SSA conversion:

**Method I** (naive): Insert copies without regard to conflicts. Produces incorrect code in the presence of lost-copy or swap problems. Useful only as a correctness baseline for testing.

**Method II** (correct sequentialisation): Insert parallel copies and sequentialise using the Briggs–Cooper algorithm above. Correct, but may insert copies that the register allocator could have coalesced.

**Method III** (congruence-class merging): The most powerful method. Before inserting any copies, compute *congruence classes* — groups of SSA names that are connected by φ-functions and have non-overlapping live ranges. Members of the same congruence class can be assigned the same machine register, making the copy unnecessary. Only when two congruence-class members have overlapping live ranges (they *interfere*) is a copy actually required; otherwise, the φ-function is destroyed "for free" by unifying the congruent names.

Method III is equivalent to performing register coalescing before register allocation: it identifies which φ-related variables can share a register without introducing spills, and inserts copies only where coalescing would cause interference. This produces minimal copy-insertion with no unnecessary copies, at the cost of computing an interference graph (which register allocation must do anyway).

LLVM does not implement a pure Method III out-of-SSA pass. Instead, LLVM's `PHIElimination` pass implements Method II (correct copy insertion), and the subsequent register allocator performs aggressive copy coalescing (via `RegisterCoalescer`) that achieves most of Method III's quality in practice.

---

## 9.9 CPS ↔ SSA Equivalence

### 9.9.1 Appel's observation

In 1998, Appel published a brief but profoundly influential note establishing that SSA form and continuation-passing style are *isomorphic* [Appel 1998]. The paper's title — "SSA is Functional Programming" — captures the claim directly.

The core mapping is:

> **SSA → CPS**: Each basic block B in SSA becomes a function `B` in CPS. The φ-function formal parameters at the head of B become the formal parameters of function `B`. Each control-flow edge from a predecessor A to B becomes a tail call `B(v₁, v₂, ...)` at the end of A, where v_i are the values contributed by A to B's φ-functions. Each non-φ instruction `x = op(y, z)` in B becomes a `let`-binding `let x = op(y, z)`.
>
> The resulting CPS program is in ANF (all arguments to calls are values, not compound expressions) and is semantically equivalent to the original SSA program.

> **CPS → SSA**: Each function `F(x₁, ..., xₖ) = body` in CPS (when F is called only at tail positions, i.e., F is known and non-escaping) becomes a basic block `F:` in SSA with φ-functions `x₁ = φ(...)`, ..., `xₖ = φ(...)` at its head. Each tail call `F(v₁, ..., vₖ)` in the CPS body becomes a conditional or unconditional jump to basic block `F` with appropriate assignments to the φ-operands.

The isomorphism is exact for programs where all functions are *known* (statically determined call targets) — precisely the condition under which CPS and SSA are equivalent. When unknown higher-order calls appear (true CPS with first-class continuations), they have no direct SSA counterpart without additional machinery (e.g., indirect branches or function pointer tables).

### 9.9.2 The formal mapping on our worked example

The header block from Section 9.6:

```
; SSA form:
header:
  x1 = φ(x0, x2)
  y1 = φ(y0, y3)
  if x1 < 10 goto body else goto exit
```

In CPS form becomes:

```
; CPS form (ANF):
let header (x1: int) (y1: int) =
  if x1 < 10
  then body ()            -- body takes no φ-parameters
  else exit x1 y1         -- exit receives the current loop values
```

The merge block:

```
; SSA:
merge:
  y3 = φ(y1, y2)
  x2 = x1 + 1
  goto header
```

becomes:

```
; CPS:
let merge (y3: int) =          -- y3 is merge's φ-parameter for y
  let x2 = x1 + 1 in           -- x1 is in scope from the enclosing function
  header x2 y3                  -- tail call, passing new values
```

Notice: `x1` is "free" in `merge`'s CPS form because it is not a φ-parameter of merge (merge does not have a φ-function for x in our example). In a full CPS program with closure conversion, x1 would either be a parameter or a free variable captured in merge's closure. In the SSA world, x1 is simply a live value from the dominating block header — the dominator-tree structure corresponds to the scope nesting in the CPS representation.

### 9.9.3 Implications of the isomorphism

The CPS–SSA isomorphism has far-reaching consequences for compiler theory and practice:

**Algorithms transfer between domains.** Any optimisation proved correct for SSA programs (GVN, SCCP, LICM, inlining) has a direct CPS analogue, and vice versa. Optimisations developed for functional-language CPS compilers can be mechanically translated into SSA transformations.

**Register allocation theory unifies.** Since CPS values correspond exactly to SSA values, and CPS tail calls correspond to SSA jumps, the live ranges of SSA values are exactly the live ranges of CPS values. Liveness analysis, interference graphs, and graph-coloring register allocation have identical formulations in both models. The "calling convention" of a CPS tail call is precisely the parallel copy of out-of-SSA conversion.

**Closure conversion informs out-of-SSA.** In CPS, when a continuation is a known function (not a general closure), the "call" is a direct jump and the "argument passing" is the parallel copy. Closure conversion techniques for eliminating heap allocation of known continuations directly correspond to register coalescing in SSA — both seek to assign the same storage to related values across a control-flow boundary.

**φ-functions are parameter passing.** The deep reason φ-functions have the structure they do — one argument per predecessor, all evaluated before any assignment — is that they model parameter passing to a continuation. The parallel-copy requirement in out-of-SSA (Section 9.8) follows immediately from the "all arguments are read before the function is called" semantics of function calls.

### 9.9.4 MLIR's block arguments as explicit CPS

MLIR makes the CPS–SSA equivalence *syntactically explicit* in the IR design. Where LLVM IR uses `phi` instructions to represent join-point merging:

```llvm
; LLVM IR (phi form):
%header:
  %x1 = phi i32 [ %x0, %entry ], [ %x2, %merge ]
  %y1 = phi i32 [ %y0, %entry ], [ %y3, %merge ]
  ...
```

MLIR uses *block arguments* — the block declares its parameters, and predecessor blocks pass values explicitly in their terminator:

```mlir
; MLIR (block argument form):
^header(%x1: i32, %y1: i32):
  ...
  cf.cond_br %cond, ^body, ^exit(%x1, %y1)

^entry:
  ...
  cf.br ^header(%x0, %y0)

^merge(%y3: i32):
  ...
  cf.br ^header(%x2, %y3)
```

The two representations are provably isomorphic. The MLIR form is in some ways more uniform:
- Every use of a value at a join point is syntactically visible at the branch site (in the terminator of the predecessor), not buried inside a φ-instruction at the target block.
- Adding or removing a φ-argument requires editing only the terminator instructions of predecessors and the block header, not scanning for `phi` instructions inside the block.
- The model generalises cleanly to nested regions: a region's entry block's arguments are the region's inputs, using the exact same syntax.

The block-argument design was a deliberate choice in the MLIR design process, explicitly acknowledging the CPS heritage and the SSA–CPS isomorphism [MLIR rationale §Basic Blocks, 2019]. Chapter 130 develops the MLIR block-argument model in full detail.

---

## 9.10 Connection to LLVM IR and MLIR

### 9.10.1 LLVM IR as infinite-register SSA

LLVM IR is a typed, infinite-register SSA in three-address code form. The `%name = instruction` pattern directly embodies the SSA invariant: the left-hand side name appears exactly once as a definition in the entire function, and subsequent instructions refer to it by name (Chapter 16).

```llvm
define i32 @example(i32 %a, i32 %b) {
entry:
  %sum   = add i32 %a, %b             ; %sum defined once
  %prod  = mul i32 %a, %b             ; %prod defined once
  %cond  = icmp slt i32 %sum, %prod   ; %cond defined once
  br i1 %cond, label %lhs, label %rhs

lhs:
  %val1  = add i32 %sum, 1
  br label %exit

rhs:
  %val2  = mul i32 %prod, 2
  br label %exit

exit:
  %result = phi i32 [ %val1, %lhs ], [ %val2, %rhs ]
  ret i32 %result
}
```

Every `%name` appears on the left-hand side of exactly one instruction. The `phi` instruction at the head of `exit` implements the φ-function, merging `%val1` from `%lhs` and `%val2` from `%rhs`.

LLVM IR is *strict SSA*: the LLVM verifier (`llvm/lib/IR/Verifier.cpp`) enforces that:
- Every value has exactly one definition.
- Every use is dominated by its definition.
- All `phi` instructions in a block appear before any non-`phi` instruction.
- The number of `phi` arguments matches the number of predecessors.

Violations result in a fatal verifier error during compilation. This strictness is fundamental to the correctness of LLVM's backend: the register allocator, instruction selector, and scheduler all rely on the invariant that every live value has a well-defined, dominating definition.

### 9.10.2 The alloca+mem2reg strategy in Clang

Clang's code generation (Chapter 39) uses a deliberately simple strategy that avoids SSA complexity in the front end. For every local variable in a C/C++ function, CodeGen emits an `alloca` instruction at the entry block (allocating a slot on the conceptual stack), and translates every read of the variable to a `load` from the alloca and every write to a `store` to the alloca:

```llvm
; Clang-generated IR for: int x = 0; if (p) x = 1; return x + 1;

entry:
  %x.addr = alloca i32, align 4    ; one alloca per local variable
  store i32 0, ptr %x.addr         ; x = 0
  %p.val = load i1, ptr %p.addr
  br i1 %p.val, label %if.then, label %if.end

if.then:
  store i32 1, ptr %x.addr         ; x = 1
  br label %if.end

if.end:
  %x.val = load i32, ptr %x.addr   ; read x
  %result = add i32 %x.val, 1
  ret i32 %result
```

This IR is technically valid (allocas and loads/stores are ordinary memory operations) but not in SSA form for the variable `x`: the value of `x` is accessed through memory, not through a direct use-def chain. The IR is correct but unoptimised — no optimiser can reason about the value of `%x.val` without alias analysis.

The `mem2reg` pass (`PromoteMemToReg.cpp`) converts this memory-based representation to proper SSA form:

1. **Identify promotable allocas.** An alloca is promotable iff all its uses are `load` and `store` instructions (no address taken, no `volatile` or `atomic` accesses, no casts to other pointer types). If these conditions hold, the alloca's lifetime is fully understood at the IR level.

2. **For each promotable alloca a, acting as a "variable v":**
   - Defs(v) = set of blocks containing a `store` to a.
   - Compute IDF(Defs(v)) using the Cytron worklist algorithm.
   - Insert `phi` instructions at each block in IDF(Defs(v)).

3. **Rename.** Run the dominator-tree DFS renaming pass, replacing each `load` from a with the current SSA value of v (from the top of Stack(v)) and recording each `store` to a as a new definition of v.

4. **Delete.** Remove all promotable allocas, loads, and stores.

After `mem2reg`, the function is in full SSA form. The pass is invoked at `-O0` (along with some other simple passes for debug info correctness) and universally at `-O1` and above [Chapter 59].

This two-phase strategy (alloca+mem2reg) is one of the cleverest design decisions in LLVM's architecture. It separates the concerns of "correct code generation" (CodeGen emits correct but unoptimised alloca-based code) from "efficient IR" (mem2reg promotes to SSA). It allows the CodeGen phase to be simpler and more trustworthy, and it means the SSA construction is done once, in a single well-tested pass, rather than being interleaved with the complex lowering logic of the code generator.

### 9.10.3 The LLVM dominance infrastructure

LLVM exposes the dominance infrastructure through its analysis manager (Chapter 59). Key classes:

**`DominatorTree`** (`llvm/include/llvm/Analysis/Dominators.h`): Implements the Cooper–Harvey–Kennedy algorithm. Provides:
- `DT.dominates(A, B)`: returns true iff A dom B.
- `DT.getNode(B)->getIDom()`: returns the idom(B) node.
- `DT.findNearestCommonDominator(A, B)`: the LCA in T_D of A and B.

**`PostDominatorTree`**: The reverse analogue — computed on the CFG with edges reversed, from exit to entry. A post-dominates B iff every path from B to any exit passes through A. Post-dominance is used in control-dependence analysis, which is needed for predicate analysis and for the weak-update rules in alias analysis.

**`DominanceFrontier`** (`llvm/include/llvm/Analysis/DominanceFrontier.h`): Computes DF(B) for all B using the Cytron edge-walking algorithm. Used directly by `mem2reg` for IDF computation.

**`LoopInfo`**: Identifies natural loops in the CFG using the back-edge characterisation: an edge B → H is a back-edge iff H dom B. The loop body is then computed as the set of blocks from which B is reachable without going through H. `LoopInfo` depends on `DominatorTree`.

When a pass modifies the CFG, it has two options:
1. *Invalidate*: mark the dominator tree as invalid; the analysis manager will recompute it before the next analysis that depends on it. O(|V| + |E|) at next use.
2. *Update incrementally*: call `DominatorTree::insertEdge(A, B)` and/or `deleteEdge(A, B)` to update the tree in O(depth of change) time. Preferred when only a few edges change.

The LLVM API for incremental updates is exposed through `DomTreeUpdater`, which batches insertions and deletions and applies them efficiently. Chapter 21 covers the LLVM dominance API in full detail.

### 9.10.4 SSA in the LLVM optimisation pipeline

The LLVM optimisation passes in the standard pipeline all assume and maintain SSA form. Key passes that interact with SSA structure:

- **`mem2reg`**: Constructs SSA from alloca-based code. Foundation of the optimisation pipeline.
- **`instcombine`**: Peephole optimisations on SSA values. Simplifies instruction sequences using algebraic identities.
- **`gvn`** (Global Value Numbering): Identifies and eliminates redundant computations using SSA value numbers. Requires SSA to be efficient.
- **`sccp`** (Sparse Conditional Constant Propagation): Propagates constants and unreachable code using SSA def-use chains. The "sparse" in the name refers to the SSA structure: only edges in the use-def graph need to be traversed, not the full CFG.
- **`licm`** (Loop-Invariant Code Motion): Hoists SSA values whose definitions dominate all loop exits out of the loop. Requires `LoopInfo` and `DominatorTree`.
- **`loop-rotate`**, **`loop-unroll`**, etc.: Transform loops in ways that preserve SSA invariants; each creates new φ-functions at new join points as needed.

The relationship between SSA and the pass pipeline is developed in Chapter 21 (SSA, Dominance, and Loops) and Chapter 59 (the pass manager).

### 9.10.5 MLIR's SSA semantics and regions

MLIR's value model is strictly SSA. Every `OpResult` (value produced by an operation) and every `BlockArgument` (a block's formal parameter) is defined exactly once and used zero or more times. The MLIR verifier enforces SSA well-formedness identically to LLVM's verifier, with one extension: *regions* can have non-standard dominance semantics based on their associated traits.

For a standard region (e.g., the body of `func.func`), the dominance rules are identical to LLVM IR: a value defined by an operation `op` in block `B` can be used only in operations in blocks dominated by `B`, or in `phi`-equivalent block arguments of successors of `B`. The MLIR `DominanceInfo` class implements this analysis.

MLIR's two region kinds and their dominance:
- **`IsolatedFromAbove`** (e.g., `func.func`, `gpu.func`): Values defined outside the region are not accessible inside. The region is "closed" from the surrounding value scope. This is the MLIR equivalent of function scope.
- **Standard (non-isolated) regions** (e.g., `scf.for`'s body, `affine.if`'s then/else regions): Values from the enclosing operation can be used freely inside the region, following the domination rule. This is used for structured control flow where the body is syntactically nested.

The choice to use block arguments over φ-instructions in MLIR enables cleaner transformations. Consider the common transformation of merging two blocks (when block A always jumps to block B and B has no other predecessors):
- In LLVM IR: we must collect all `phi` instructions in B and find the appropriate operands for A, then delete the phis.
- In MLIR: we simply redirect predecessors of B to A, re-route A's block arguments to B's, and inline B's body — no explicit phi-merging required.

Chapter 130 develops the full MLIR block-argument and region model.

---

## 9.11 Chapter Summary

- **The N×M problem** is the foundational motivation for IRs: with a shared IR, N front ends and M back ends require only N+M components (rather than N×M dedicated translators). Middle-end optimisations benefit all language–target combinations simultaneously. LLVM is the definitive realisation of this principle.

- **IR taxonomy** spans a spectrum from high-level to low-level: ASTs are tree-structured and suitable for type checking but poor for dataflow; three-address code is flat and explicit, close to LLVM IR; CPS makes all control flow and continuations explicit at the cost of closure allocation overhead; ANF is a simplification of CPS that retains direct-style evaluation order; SSA adds the unique-definition invariant to TAC, enabling dramatically simpler and faster analyses.

- **The SSA invariant** requires each variable to be assigned exactly once in the program text, with φ-functions at join points to merge differing incoming values. Strict SSA additionally requires every use to be dominated by its definition. The "static" in SSA distinguishes the one textual definition from the potentially many dynamic runtime instances.

- **Dominator trees** encode all dominance relationships compactly: A dominates B iff A is an ancestor of B in T_D. Computing T_D uses the Cooper–Harvey–Kennedy iterative algorithm (fast in practice, O(n²) worst case) or Lengauer–Tarjan (O(n α(n)) theoretically, but slower in practice). LLVM uses Cooper–Harvey–Kennedy with incremental update support.

- **Dominance frontiers** DF(A) capture where A's dominance stops — the join points just beyond A's reach. Loop headers are in their own DF due to back-edges. The **iterated dominance frontier** IDF(Defs(v)) is the fixed point of repeated DF application, and φ-functions for variable v are required at exactly IDF(Defs(v)) [Cytron et al. 1991].

- **The Cytron algorithm** constructs SSA in two phases: (1) φ-placement at IDF(Defs(v)) for each variable via a worklist computation; (2) variable renaming via a pre-order DFS of the dominator tree, maintaining per-variable stacks whose tops always hold the current dominating version. The worked example in Section 9.6 carries all four steps to completion on a representative loop CFG.

- **Minimal, pruned, and semi-pruned SSA** [Briggs et al. 1998] represent a trade-off between construction cost and φ-function count. Minimal SSA places φ-functions without liveness filtering; pruned SSA requires a pre-SSA liveness pass but places the fewest φ-functions; semi-pruned SSA eliminates purely-local variables at the cost of a single linear scan and is an excellent practical default.

- **Out-of-SSA conversion** requires sequentialising the parallel-copy semantics of φ-functions. The naive sequential approach fails due to the *lost-copy* and *swap* problems. The Briggs–Cooper–Harvey–Reeves algorithm correctly handles all cases by processing non-conflicted copies first and breaking cycles with temporaries. Sreedhar's Method III eliminates copies where congruent variables can share a register.

- **CPS and SSA are isomorphic** [Appel 1998]: each SSA basic block corresponds to a CPS function, φ-function parameters to function parameters, and SSA edges to tail calls. MLIR makes this isomorphism syntactically explicit via block arguments — predecessor terminators explicitly pass values to join-block parameters — providing a more uniform and transformation-friendly representation than LLVM's φ-instruction model.

- **LLVM generates SSA via alloca+mem2reg**: Clang emits simple alloca-based IR (local variables accessed through load/store), and the `mem2reg` pass promotes these to SSA values using the Cytron algorithm. This clean separation of concerns keeps the front-end code generator simple and concentrates SSA complexity in a single, well-tested pass. The LLVM dominance infrastructure (`DominatorTree`, `PostDominatorTree`, `DominanceFrontier`) is available to all LLVM passes via the analysis manager and supports incremental updates.

---

## References

- Cytron, R., Ferrante, J., Rosen, B.K., Wegman, M.N., Zadeck, F.K. "Efficiently Computing Static Single Assignment Form and the Control Dependence Graph." *ACM Trans. Program. Lang. Syst.* 13(4):451–490, 1991. [Cytron et al. 1991]
- Cooper, K.D., Harvey, T.J., Kennedy, K. "A Simple, Fast Dominance Algorithm." Rice University Computer Science Technical Report TR-06-33870, 2001. [Cooper–Harvey–Kennedy 2001]
- Lengauer, T., Tarjan, R.E. "A Fast Algorithm for Finding Dominators in a Flowgraph." *ACM Trans. Program. Lang. Syst.* 1(1):121–141, 1979. [Lengauer–Tarjan 1979]
- Briggs, P., Cooper, K.D., Harvey, T.J., Simpson, L.T. "Practical Improvements to the Construction and Destruction of Static Single Assignment Form." *Softw. Pract. Exper.* 28(8):859–881, 1998. [Briggs et al. 1998]
- Sreedhar, V.C., Gao, G.R. "A Linear Time Algorithm for Placing φ-Functions." In *Proc. POPL*, pp. 62–73, 1995. [Sreedhar–Gao 1995]
- Appel, A.W. "SSA is Functional Programming." *ACM SIGPLAN Notices* 33(4):17–20, 1998. [Appel 1998]
- Cooper, K.D., Torczon, L. *Engineering a Compiler*, 3rd ed. Morgan Kaufmann, 2022. Chapters 5, 9. [EaC]
- Appel, A.W. *Compiling with Continuations*. Cambridge University Press, 1992.
- Appel, A.W. *Modern Compiler Implementation in ML*. Cambridge University Press, 1998. [Appel MCI]
- Flanagan, C., Sabry, A., Duba, B.F., Felleisen, M. "The Essence of Compiling with Continuations." In *Proc. PLDI*, pp. 237–247, 1993.
- GCC Internals Manual, Chapter 12: GIMPLE. https://gcc.gnu.org/onlinedocs/gccint/GIMPLE.html
