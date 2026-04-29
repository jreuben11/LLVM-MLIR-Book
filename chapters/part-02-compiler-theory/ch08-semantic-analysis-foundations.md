# Chapter 8 — Semantic Analysis Foundations

*Part II — Compiler Theory and Foundations*

Chapter 7 established that parsing is the process of recognising structure: given a stream of tokens the parser constructs a parse tree (or directly an abstract syntax tree) that encodes the hierarchical syntactic organisation of the input. Parsing is indifferent to *meaning*. The expression `true + 1`, the call `sqrt("hello")`, and the statement `int x; int x;` are perfectly grammatical in a typical C-family grammar, yet each violates the language's semantic rules. **Semantic analysis** is the compiler phase that enforces meaning. It answers the questions parsing cannot: does every name refer to a visible declaration? Are operand types compatible? Do overloaded functions resolve unambiguously? Do subexpressions coerce correctly?

This chapter develops the theoretical foundations that underpin every production semantic analyser. Section 8.1 covers symbol tables — the core data structure for binding names to declarations — in three flavours: chained (scope-stack), hash-based, and persistent functional. Section 8.2 formalises the scoping models that determine which declarations are visible where. Section 8.3 introduces Knuth's attribute grammar formalism, the cleanest theoretical account of how semantic information flows through a parse tree, distinguishing synthesised from inherited attributes and characterising the evaluator strategies that each class enables. Section 8.4 extends the formalism to syntax-directed translation schemes (SDTs) and works through the canonical backpatching example. Section 8.5 treats type checking as a tree walk implementing a type-assignment judgment. Section 8.6 develops name resolution algorithms for unqualified lookup, qualified lookup, and the two-phase template lookup and argument-dependent lookup (ADL) rules of C++. Section 8.7 connects the theory back to implementation by showing how the Gang of Four Visitor pattern is a direct compilation of the attribute grammar evaluator.

Chapter 9 continues from where this chapter ends: once the AST has been name-resolved and type-checked, it is lowered to an intermediate representation and SSA form is constructed.

---

## 8.1 Symbol Tables: Chained, Hash-Based, and Persistent

### The purpose of a symbol table

A **symbol table** (also called an *environment* or *name table*) is a mapping from identifiers to **attributes**: declarations, types, storage locations, linkage information, and any other semantic properties the compiler needs to associate with a name. The key operations are [Dragon §6.1, EaC §5.7]:

- **insert(name, attributes)** — record a new binding at the current scope. An error is raised if the name is already declared in the *same* scope (redeclaration).
- **lookup(name)** — return the attributes of the nearest enclosing binding for *name*, or ⊥ if no binding is visible.
- **enter\_scope()** — push a new scope frame onto the scope stack.
- **leave\_scope()** — pop the current scope frame, retiring all bindings introduced within it.

The abstract interface hides a spectrum of concrete implementations. The three principal designs — chained, hash-based, and persistent — differ in their asymptotic complexity, their treatment of scope, and their suitability for multi-pass or parallel compilation.

### 8.1.1 Chained (linked-list) symbol tables

The simplest design represents the scope stack as a linked list of *scope frames*, where each frame is a small association list (or array) of (name, attributes) pairs. The chain is traversed from front to back during lookup, so the innermost scope is found first — implementing the scoping rule that inner declarations shadow outer ones.

```
-- Pseudocode: chained symbol table

type Binding = { name: String, attrs: Attributes }
type Scope   = List Binding
type SymTab  = List Scope        -- head = innermost scope

enter_scope(T: SymTab) -> SymTab:
    return [] :: T               -- prepend empty scope

leave_scope(T: SymTab) -> SymTab:
    assert T ≠ []
    return tail(T)               -- discard innermost scope

insert(T: SymTab, name, attrs) -> SymTab:
    innerScope = head(T) ++ [(name, attrs)]
    return innerScope :: tail(T)

lookup(T: SymTab, name) -> Attributes | ⊥:
    for scope in T:              -- traverse from innermost outward
        for (n, a) in scope:
            if n == name: return a
    return ⊥
```

**Complexity.** Let *d* be the nesting depth and *s* be the average number of bindings per scope. Lookup visits at most *d* scopes and scans at most *s* bindings per scope, giving O(d × s) worst-case. For typical programs with small scopes and moderate nesting depth, lookup is effectively O(1) amortised because the innermost scope contains the binding for most local-variable lookups. Enter and leave are both O(1). Insert is O(1) (prepend to head scope).

**Scope entry and exit** are trivially managed: entering a new scope prepends a fresh frame; leaving discards the head. No hash table rebuild is required.

The chained design is the natural choice for teaching compilers and for interpreters. GCC's `cp_binding_level` is a production-grade variant: each scope level is a heap-allocated `cp_binding_level` struct containing a hash table of bindings for that level, with a `level_chain` pointer to the enclosing scope [GCC source, `cp/name-lookup.c`]. The "chaining" is the `level_chain` linked list; the individual levels use hashing for O(1) intra-scope lookup.

### 8.1.2 Hash-based symbol tables

For languages with large translation units and deep include hierarchies — C++ headers in particular — a single flat hash table with scope management via *decoration* or *versioning* can be faster than traversing a scope chain.

**Open-addressing hash table with scope versioning.** Each entry in the table stores a name, a *version counter*, and a stack of (version, attributes) pairs. The current scope's version is a monotonically increasing integer.

```
-- Pseudocode: flat hash table with per-entry scope stacks

type Entry = { name: String, bindings: Stack(version × Attributes) }
type SymTab = { table: HashTable(String → Entry),
                scope_stack: Stack(Set(String)),
                version: Int }

enter_scope(T: SymTab) -> SymTab:
    T.scope_stack.push({})       -- empty set of names introduced here
    T.version += 1
    return T

leave_scope(T: SymTab) -> SymTab:
    introduced = T.scope_stack.pop()
    for name in introduced:
        entry = T.table[name]
        entry.bindings.pop()     -- retire the innermost binding
        if entry.bindings.empty(): T.table.remove(name)
    return T

insert(T: SymTab, name, attrs) -> SymTab:
    if name not in T.table: T.table[name] = Entry(name, [])
    T.table[name].bindings.push((T.version, attrs))
    T.scope_stack.top().add(name)
    return T

lookup(T: SymTab, name) -> Attributes | ⊥:
    if name not in T.table: return ⊥
    stack = T.table[name].bindings
    if stack.empty(): return ⊥
    return top(stack).attrs      -- O(1): innermost binding is stack top
```

**Complexity.** Lookup is O(1) expected (one hash probe, one stack peek). Insert is O(1) expected. Leave\_scope is O(|introduced|) — proportional to the number of names introduced in the scope being left, not to the total table size.

**Open addressing vs. chaining.** Open addressing (linear probing, quadratic probing, Robin Hood hashing) achieves better cache performance when load factors remain low because entries are stored contiguously. Chaining (each bucket holds a linked list of entries) handles high load factors gracefully at the cost of pointer-chasing. The scope-management overhead (the per-entry binding stack) is identical in either variant.

**Shadowing via stack-per-bucket.** The per-entry binding stack implements shadowing naturally: when a new scope declares `x`, a new (version, attrs) pair is pushed onto `x`'s binding stack. The previous binding remains on the stack but is inaccessible until `x`'s new binding is popped. The version counter allows distinguishing "same scope, multiply declared" (error) from "different scope, legal shadowing."

### 8.1.3 Persistent (functional) symbol tables

Imperative symbol tables manage scope by mutation: enter pushes, leave pops. A **persistent** (or *functional*) symbol table uses immutable data structures where each scope operation produces a *new* symbol table without destroying the old one [Appel, §1.2].

The canonical implementation uses either a persistent balanced BST (e.g., a red-black tree or AVL tree with path-copying) or a **Hash Array Mapped Trie (HAMT)**. HAMTs — introduced by Bagwell (2001) and used pervasively in Clojure, Scala, and Haskell — provide O(log₃₂ n) ≈ O(1) practical lookup and update by storing a 32-way branching trie indexed by successive 5-bit chunks of the key's hash, with structural sharing at every level.

```
-- Pseudocode: persistent symbol table (functional)

type SymTab = ImmutableMap(String → Attributes)
             -- backed by HAMT or persistent BST

empty_symtab : SymTab = EmptyMap

-- "Entering a scope" means passing the current table to a sub-computation.
-- The scope IS the continuation with the extended table.
-- No explicit enter_scope/leave_scope needed.

insert(T: SymTab, name, attrs) -> SymTab:
    return T.with(name → attrs)  -- structural sharing; O(log n) or O(1) amortised

lookup(T: SymTab, name) -> Attributes | ⊥:
    return T.get(name)           -- O(log n) or O(1) amortised

-- Scope management:
-- To open a scope, pass T to the sub-computation.
-- The sub-computation inserts new bindings, producing T'.
-- To leave the scope, simply discard T' and reuse T.
-- Example:
compile_block(stmts, T: SymTab) -> IR:
    T' = T
    for stmt in stmts:
        (code, T') = compile_stmt(stmt, T')
    return code
    -- T' goes out of scope; T is the environment after the block
```

**Advantages.** A persistent table allows the *same* scope to be used as the starting point for multiple branches — for instance, when type-checking both arms of an `if` statement, or when processing multiple function bodies from the same set of top-level declarations in parallel. There is no need for an explicit "restore" or "pop"; the old table is simply re-used from the call stack or from wherever it was stored.

**Disadvantages.** Persistent tables have higher constant factors than mutable hash tables for single-threaded sequential use. The allocation pressure from structural sharing can stress the garbage collector in long passes over large translation units.

**HAMT structure in brief.** A Hash Array Mapped Trie stores a mapping as a trie where each node covers 5 bits of the key's hash, providing a branching factor of 32. Each node stores a 32-bit bitmap indicating which of the 32 slots are occupied, and a compact array of the occupied slots. Lookup hashes the key, extracts 5 bits at the current depth, checks the bitmap to see if the slot is occupied, and recurses into the child. Structural sharing means that an `insert` operation copies only the path from root to the modified leaf — at most log₃₂(n) ≈ 7 nodes for a million-entry map — leaving the rest of the trie shared with the previous version. This makes the persistent-table model memory-efficient in practice even for large scopes.

### 8.1.4 Implementation in production compilers

**Clang's `DeclContext`.** Clang does not maintain a conventional symbol table as a separate data structure. Instead, each scope in the AST is represented by a `DeclContext` node (the base class for `NamespaceDecl`, `CXXRecordDecl`, `FunctionDecl`, etc.). The `DeclContext` stores its contained declarations in a `StoredDeclsMap`, a hash map from `DeclarationName` to a `SmallVector<NamedDecl*>` of all declarations with that name (since a name can be overloaded). Lookup proceeds by walking the `DeclContext` chain: block scope → function scope → class scope → namespace scope → global scope [Clang, `lib/AST/DeclBase.cpp`]. The `Sema` class maintains a `ScopeStack` of `Scope*` objects for the current parse context, but the authoritative bindings live in the `DeclContext` hierarchy. Chapter 33 covers Clang's `Sema` and lookup pipeline in full detail.

**GCC's `cp_binding_level`.** GCC's C++ frontend represents the scope chain as a linked list of `cp_binding_level` structures. Each level contains a hash table of `cxx_binding` objects, where each `cxx_binding` holds the current value and the previous (shadowed) value, implementing chained shadowing. The `current_binding_level` global pointer tracks the innermost scope; `pushlevel`/`poplevel` manage the chain [GCC source, `cp/name-lookup.c`].

---

## 8.2 Scoping Models: Lexical, Dynamic, and Hierarchical

### 8.2.1 Lexical (static) scoping

**Definition.** Under **lexical scoping** (also called *static scoping*), a free occurrence of a name *x* in an expression *e* refers to the binding of *x* that is *textually* innermost enclosing *e* in the source code [Dragon §1.6.3, EaC §5.1]. The binding of a name is fully determined by the static structure of the program text; no knowledge of the runtime call sequence is needed.

Lexical scoping is the foundation of ML, Scheme, C, C++, Java, Rust, Python, and essentially every language designed since 1975. Its key property is **referential transparency in the name-resolution sense**: if you can see the source text around a use of *x*, you can find *x*'s declaration without simulating program execution.

The formal rule is stated in terms of the *scoping function* σ: (occurrence × program) → declaration. For lexically scoped languages, σ is computable from the static text alone.

```
-- Example: lexical scoping in C

int x = 1;

int f() {
    return x;   // refers to global x = 1 (nearest enclosing textual binding)
}

int g() {
    int x = 2;
    return f(); // f() still returns 1, not 2,
                // because f's free reference to x is resolved
                // at f's *definition* site, not its call site
}
```

The function `f` closes over the *textual* binding of `x` at the point where `f` is defined. When `g` calls `f`, the fact that `g` has a local `x = 2` is irrelevant. This is the crucial difference from dynamic scoping.

### 8.2.2 Dynamic scoping

**Definition.** Under **dynamic scoping**, a free occurrence of *x* refers to the most recently *activated* binding of *x* in the runtime call stack — the "most dynamic" binding.

With dynamic scoping, `f()` above would return 2 when called from `g`, because `g`'s `x = 2` was the most recently activated binding of `x` at the moment of the call.

Dynamic scoping was the default in traditional Lisp (and remains the default in Emacs Lisp). Early BASIC used dynamic scoping for subroutines. It is implemented by maintaining a runtime stack or association list of (name, value) pairs, and looking up names by scanning from the top.

**Why dynamic scoping is rarely used today.**

1. **Unpredictability.** The meaning of a name at a call site depends on the entire call chain leading to that call. Code becomes difficult to reason about in isolation; a function's behaviour changes depending on the context in which it is called, even if the source text is not changed.
2. **Poor modularisation.** Two independently-developed modules that happen to use the same internal variable name can interfere when composed. Lexical scoping makes the module boundary a hard scope boundary.
3. **No support for higher-order functions.** If a function is passed as a value and called later, the binding environment at the call site may bear no relation to the environment the function was designed for.
4. **Compiler optimisation.** Dynamic lookup of names must proceed at runtime; the compiler cannot inline variable accesses as stack-frame offsets because the location of a binding depends on the dynamic call chain.

Limited forms of dynamic scoping survive in modern languages: Perl's `local`, Common Lisp's special variables, and Clojure's `binding` form all provide thread-local dynamic rebinding with a restricted scope. These are deliberate choices for specific use cases (thread-local overrides, configuration values), not the default scoping rule.

### 8.2.3 Hierarchical scoping in C++

C++ has one of the most elaborate scope hierarchies among production languages [Dragon §6.3, EaC §5.3]:

| Scope kind            | Introduced by                        | Example                                  |
|-----------------------|--------------------------------------|------------------------------------------|
| File (translation unit) | Top-level declarations              | `int global_x;`                          |
| Namespace             | `namespace N { … }`                  | `namespace std { … }`                    |
| Class                 | `class C { … }` / `struct`           | `class Foo { int x; };`                  |
| Function              | Function definition                  | `void f() { … }`                         |
| Function-parameter    | Parameter list                       | `void f(int x) { … }`                    |
| Block                 | `{ … }` compound statement           | `if (cond) { int tmp; … }`               |
| Template              | `template<typename T>` declaration   | template parameter `T`                   |
| Enumeration           | `enum class E { … }`                 | scoped enumerator names                  |

Scopes are nested: a block scope is enclosed by the function scope, which is enclosed by the class scope (for member functions), which is enclosed by the namespace scope, which is enclosed by the global scope. Lookup for unqualified names proceeds outward from the innermost scope.

**Name hiding and shadowing.** A declaration in an inner scope *hides* a declaration with the same name in an outer scope. The outer declaration remains valid and accessible via a qualified name:

```cpp
int x = 1;
namespace N {
    int x = 2;         // hides global x within N
    void f() {
        int x = 3;     // hides N::x within f
        {
            int x = 4; // hides f's x within this block
            // ::x == 1, N::x == 2, but unqualified x == 4
        }
    }
}
```

The **scope resolution operator** `::` provides qualified access: `::x` names the global `x`; `N::x` names namespace `N`'s `x`; `C::m` names class `C`'s member `m`.

**The two-pass approach and forward declarations.** C requires declarations to precede uses within a translation unit (top-down single-pass rule), though function prototypes allow forward reference. Pascal enforced strict order of declarations and required explicit `forward` annotations for mutually recursive procedures. C++, Java, and most modern languages adopt a **two-pass approach** for class and namespace scopes: the compiler performs an initial scan to collect all declarations in the scope before resolving any uses [EaC §5.5]. This allows members of a class to reference each other freely regardless of declaration order, and allows functions in a namespace to call each other without forward declarations. The two passes are typically implemented as a single recursive descent over the AST with a pre-pass that populates the `DeclContext` hash table before the main semantic-analysis traversal begins.

**Scope in for-loop initialisers.** C++11 introduced the range-based `for` loop with `for (auto x : container)`, where `x`'s scope is the entire loop body. The initialiser-list for `for (init; cond; incr)` creates a new scope for variables declared in `init` that extends to the end of the loop body. This "for-scope" is an additional scope level between the enclosing block and the loop body block, meaning a variable declared in `init` is in scope for `cond` and `incr` but not after the loop. Getting this scoping right requires careful management of the scope stack in the semantic analyser, which Clang handles in `Sema::ActOnForStmt`.

**The `this` pointer and implicit member access.** Inside a non-static member function, the name `this` is a prvalue of type `C*` (or `const C*` for const member functions). Unqualified lookup for a name inside a member function checks the class scope: if the name resolves to a non-static data member or member function, the compiler implicitly desugars the access to `this->member`. This is an example of scope-directed name transformation — the semantic analyser rewrites the AST node from a `VarRefExpr` or `FunctionRef` into a `MemberExpr` with `this` as the object expression.

---

## 8.3 Attribute Grammars: Synthesised and Inherited

### 8.3.1 Knuth's formalism

In 1968, Donald Knuth introduced **attribute grammars** as a formal framework for specifying the semantics of context-free languages [Knuth 1968]. An attribute grammar is a triple AG = (G, A, R) where:

- **G = (V, T, P, S)** is a context-free grammar.
- **A** is a function associating with each grammar symbol X ∈ V ∪ T a finite set A(X) of **attributes**. Attributes are partitioned into **synthesised** attributes (A_s(X)) and **inherited** attributes (A_i(X)). Terminals may have synthesised attributes (e.g., a token's lexical value) but no inherited attributes. The start symbol has no inherited attributes.
- **R** is a function associating with each production p ∈ P a set R(p) of **semantic rules** (also called *attribute equations*). Each rule in R(p) defines exactly one attribute occurrence on p's symbols as a function of other attribute occurrences on p's symbols.

For a production p: A → X₁ X₂ … Xₙ, a semantic rule has the form:

```
X₀.a := f(X₁.b₁, X₂.b₂, …, Xₙ.bₙ)
```

where X₀ is either A (the left-hand side) or one of the Xᵢ, and a, b₁, … are attribute names. The function f is an arbitrary semantic function (arithmetic expression, type-set union, offset accumulator, etc.).

### 8.3.2 Synthesised attributes

**Definition.** An attribute `X.a` is **synthesised** if the semantic rule that defines it appears in a production where `X` is the *left-hand side* nonterminal [Dragon §5.1.1]. Equivalently, X's synthesised attribute is computed from the attributes of X's *children* in the parse tree.

Synthesised attributes flow *bottom-up*: leaves first, then interior nodes, then the root. A grammar that uses *only* synthesised attributes is called **S-attributed** [Dragon §5.2.3]. An S-attributed grammar can be evaluated in a single bottom-up pass — exactly the order in which an LR parser reduces productions. This is why S-attributed grammars map directly onto yacc/bison actions.

**Worked example: Type computation for arithmetic expressions.**

Consider the grammar for typed arithmetic expressions with integer literals `inum` and floating-point literals `fnum`:

```
Production                          Semantic rule
──────────────────────────────────────────────────────────────
E → E₁ + T                         E.type := wider(E₁.type, T.type)
E → T                               E.type := T.type
T → T₁ * F                         T.type := wider(T₁.type, F.type)
T → F                               T.type := F.type
F → ( E )                           F.type := E.type
F → inum                            F.type := int
F → fnum                            F.type := float
```

The helper function `wider(t₁, t₂)` returns `float` if either argument is `float`, otherwise `int` — implementing C's usual arithmetic promotion.

Every attribute (`E.type`, `T.type`, `F.type`) is synthesised: it is defined by the left-hand side of each production as a function of child attributes. For the string `3.14 + 2`:

```
                E (type=float)
              / | \
       E(type=int) + T(type=float)
            |              |
       T(type=int)    F(type=float)
            |              |
       F(type=int)        fnum
            |
           inum
```

The `int` leaf (`inum`) propagates `int` up to its T and E nodes. The `fnum` leaf propagates `float` up to its F and T nodes. At the root, `wider(int, float) = float` gives the expression's type.

**Worked example: Offset computation for record fields.**

```
Production                          Semantic rules
──────────────────────────────────────────────────────────────
Record → 'record' Fields 'end'      Record.size := Fields.size
Fields → Field Fields₁              Fields.size := Field.size + Fields₁.size
                                    Field.offset := 0          [for first field]
Fields → ε                          Fields.size := 0
Field → T id ';'                    Field.size  := T.size
                                    -- id.offset is the field offset
                                    -- (inherited from parent — see §8.3.3)
```

For a record `{ int x; float y; double z; }` assuming `sizeof(int)=4, sizeof(float)=4, sizeof(double)=8`:

```
Fields.size = 4 + 4 + 8 = 16
x.offset = 0, y.offset = 4, z.offset = 8
```

The sizes are synthesised bottom-up. The offsets require inherited attributes (§8.3.3).

**Parse-tree annotation for the type computation example.** To make the evaluation order concrete, consider the string `3 * 2.0 + 1` under the typed expression grammar above. The parse tree and the attribute values annotated at each node are:

```
              E  [type=float]
            / | \
    E[type=float] +  T[type=int]
          |                |
    T[type=float]      F[type=int]
        / | \               |
  T[type=int] * F[type=float]  inum(1)
       |              |
  F[type=int]       fnum(2.0)
       |
   inum(3)
```

Evaluation proceeds post-order (leaves first):
1. `inum(3)` → `F.type = int`
2. `fnum(2.0)` → `F.type = float`
3. `T → T * F` with T.type=int, F.type=float → `T.type = wider(int,float) = float`
4. `inum(1)` → `F.type = int`
5. `T → F` with F.type=int → `T.type = int`
6. `E → E + T` with E.type=float, T.type=int → `E.type = wider(float,int) = float`

At step 6, the compiler knows it must widen the right operand from `int` to `float` before performing the addition. This triggers coercion-node insertion into the AST (§8.5.4), which replaces the T subtree with `CastExpr(T, int→float)`.

### 8.3.3 Inherited attributes

**Definition.** An attribute `Xᵢ.a` (for some Xᵢ on the right-hand side of a production A → X₁…Xₙ) is **inherited** if the semantic rule that defines it appears in the production for A — that is, the parent A computes a child's attribute from A's own attributes and the attributes of siblings of Xᵢ [Dragon §5.1.2]. Inherited attributes flow *top-down*: from the root toward the leaves.

**Worked example: Offset computation (continued).**

To distribute offsets to fields, we add an inherited attribute `offset_in` carrying the starting offset to each `Fields` and `Field` node:

```
Production                          Semantic rules
──────────────────────────────────────────────────────────────
Record → 'record' Fields 'end'      Fields.offset_in := 0
                                    Record.size      := Fields.size

Fields → Field Fields₁              Field.offset_in   := Fields.offset_in  [inherited]
                                    Fields₁.offset_in := Fields.offset_in + Field.size
                                    Fields.size       := Field.size + Fields₁.size

Fields → ε                          Fields.size := 0

Field → T id ';'                    id.offset := Field.offset_in           [inherited]
                                    Field.size := T.size
```

`Fields.offset_in` is inherited (the parent `Record` or the recursive `Fields` production computes it and passes it downward). `id.offset` is also inherited (it receives the accumulated offset from the parent `Field`).

**Worked example: Type coercion insertion (L-attributed).**

A common semantic requirement is to insert implicit coercion nodes when operands of mixed types are combined. We add an inherited attribute `type_ctx` that carries the *expected* type from context:

```
Production                          Semantic rules
──────────────────────────────────────────────────────────────
AssignStmt → id '=' E ';'           E.type_ctx := id.decl_type    [inherited: from id's declared type]
                                    code := assign(id.offset,
                                                  coerce(E.code, E.type, E.type_ctx))

E → E₁ + T                         E.type     := wider(E₁.type, T.type)
                                    E₁.type_ctx := E.type          [inherited: from result type]
                                    T.type_ctx  := E.type
                                    E.code      := add_with_coerce(E₁.code, T.code, E.type)

E → T                               E.type     := T.type
                                    T.type_ctx  := E.type_ctx       [pass inherited attr to child]
```

The `type_ctx` inherited attribute flows from the assignment statement downward into the expression tree, allowing each node to insert a coercion (e.g., `int → float`) at the point where the type mismatch occurs. This is an **L-attributed** grammar [Dragon §5.3]: each inherited attribute depends only on attributes of left siblings and the parent.

### 8.3.4 Well-definedness and the dependency graph

**Definition.** For a parse tree T, the **dependency graph** D(T) has one node for each attribute occurrence in T, and a directed edge from b to a whenever a semantic rule of the form `a := f(…, b, …)` is instantiated at some node in T. An attribute grammar is **well-defined** if every dependency graph D(T) for every parse tree T is acyclic — i.e., the attribute computation has no circular dependencies [Dragon §5.2.1, Knuth 1968].

Checking well-definedness in the general case requires examining all possible parse trees, which is undecidable for arbitrary grammars. However, for restricted classes of grammars the check can be done on the grammar rules alone:

- **S-attributed grammars** are trivially well-defined: synthesised attributes form a pure bottom-up data flow with no cycles possible.
- **L-attributed grammars** (inherited attributes depend only on left siblings and the parent) are well-defined and can be evaluated in a single left-to-right traversal of the parse tree, which corresponds to depth-first traversal in the standard preorder.

Knuth proved that determining if an arbitrary attribute grammar is well-defined is decidable but PSPACE-complete [Knuth 1968, Theorem 6]. In practice, attribute grammar tools (UUAG, Lrc, Silver) restrict the formalism to well-definedness-guaranteed subclasses.

**Evaluating an S-attributed grammar.** For an S-attributed grammar, evaluation is a single bottom-up pass: a post-order tree traversal. For each node visited, the semantic rules for its production are evaluated in any order that respects the dependency within the production (typically left-to-right since sibling dependencies in S-attributed rules go left-to-right). This exactly matches the reduce actions of a bottom-up (LR) parser: actions fire as each production is reduced, and by that point all children's attributes are available.

**Evaluating an L-attributed grammar.** For an L-attributed grammar, evaluation is a combined pre-order (for inherited) and post-order (for synthesised) traversal. As the tree traversal enters a node (pre-order), the inherited attributes of its children are computed from the parent's attributes and left siblings' synthesised attributes. As the traversal exits a node (post-order), the node's synthesised attributes are computed from its children's synthesised attributes. This corresponds to a recursive-descent (top-down) parser that threads the inherited attributes as parameters through mutually recursive parsing procedures.

**The dependency graph in detail.** For the L-attributed type-coercion grammar, the dependency graph for the assignment `x = 3 * 2.0 + 1;` (assuming `x` is declared `float`) contains:

```
AssignStmt.code ← E.code, E.type, E.type_ctx, id.offset
E.type_ctx ← id.decl_type                          [inherited: flows downward]
E.type ← E₁.type, T.type                           [synthesised: flows upward]
E₁.type_ctx ← E.type                               [inherited: flows downward]
T.type_ctx ← E.type_ctx                            [inherited: pass-through]
```

Topologically sorting this graph yields the evaluation order: (1) look up `id.decl_type` from the symbol table → (2) set `E.type_ctx` → (3) synthesise types bottom-up → (4) use `E.type_ctx` to decide coercions → (5) emit assignment code. The graph is acyclic, confirming the grammar is well-defined.

### 8.3.5 Limitations of literal attribute grammars

Attribute grammar tools that implement the formalism literally — generating evaluators from grammar + semantic-rule specifications — exist but are rarely used in production compilers. The reasons are pragmatic:

1. **Coupling to the grammar.** The grammar must be explicit and stable. Real language grammars evolve frequently, and rewriting semantic rules when grammar rules change is expensive.
2. **Ordering constraints.** Specifying the correct inherited/synthesised classification for every attribute and verifying well-definedness is labour-intensive.
3. **Imperative extension.** Many semantic operations — I/O, table updates, error reporting — are inherently side-effectful and do not fit the functional attribute-equation model.

Production compilers implement the *ideas* of attribute grammars — synthesised attributes as values computed during bottom-up traversal, inherited attributes as parameters threaded through recursive descent — but do so via direct C++ code over an AST, not via a separate attribute grammar specification language. The theoretical framework remains indispensable for understanding what the code is doing and for proving that the evaluation order is correct.

---

## 8.4 Syntax-Directed Translation Schemes

### 8.4.1 SDTs: extending CFGs with embedded actions

A **syntax-directed translation scheme (SDT)** is a CFG with **semantic actions** embedded at specific positions within the right-hand sides of productions [Dragon §5.4]. An action is a code fragment enclosed in braces `{ … }` that executes when the parser's "dot" reaches that position:

```
A  →  X₁ { action₁ } X₂ { action₂ } … Xₙ { actionₙ }
```

Actions to the *right* of a symbol execute *after* that symbol has been recognised (and, for a nonterminal, fully processed). This is the crucial distinction from attribute grammars, where semantic rules are associated with entire productions: SDTs make the *timing* of action execution explicit.

### 8.4.2 Postfix SDTs

A **postfix SDT** places all actions at the right end of productions. Because no action precedes any grammar symbol, postfix SDTs are implementable during LR (bottom-up) parsing: the action fires exactly when the production is reduced, at which point all right-hand side symbols have been processed and their attributes are available on the parsing stack.

```
-- Postfix SDT for arithmetic evaluation (calculator)
E  →  E₁ + T   { E.val := E₁.val + T.val }
E  →  T         { E.val := T.val }
T  →  T₁ * F   { T.val := T₁.val * F.val }
T  →  F         { T.val := F.val }
F  →  ( E )     { F.val := E.val }
F  →  num       { F.val := num.lexval }
```

This is the yacc/bison model: `$$` is the synthesised attribute of the left-hand side; `$1`, `$2`, … are the (synthesised) attributes of the right-hand side symbols in order. The braced action is placed at the right end of the production (before the closing semicolon). Internally, bison implements this by maintaining a **value stack** parallel to the parse state stack: when a production is reduced, the action fires, and the result replaces the *n* stack entries for the right-hand side with a single entry for the left-hand side.

### 8.4.3 L-attributed SDTs

An **L-attributed SDT** allows actions anywhere within a production, subject to the restriction that an action may only reference:

1. Attributes of grammar symbols to its *left* in the right-hand side (synthesised attributes of left siblings and inherited attributes of left siblings).
2. Inherited attributes of the left-hand side nonterminal (i.e., attributes from the parent).

This restriction ensures that all attributes needed by an action are available when the action fires during a left-to-right (top-down or bottom-up) parse [Dragon §5.4.2].

```
-- L-attributed SDT for type coercion in assignments
AssignStmt → id '=' { E.itype := lookup_type(id.name) } E ';'
             { emit_assign(id.offset, coerce(E.code, E.type, E.itype)) }
```

Here `{ E.itype := lookup_type(id.name) }` is an action embedded *before* `E`. It computes an inherited attribute of `E` from `id`, which appears to the left of the action. This requires L-attributed evaluation.

L-attributed SDTs can be implemented during recursive-descent (top-down) parsing. The inherited attributes of each nonterminal become **parameters** of its parsing procedure, and the synthesised attributes become **return values**:

```
-- Recursive descent implementing L-attributed SDT
parse_E(itype: Type) -> (code: Code, type: Type):
    (e1_code, e1_type) = parse_E(itype)    -- left recursion eliminated
    match_token('+')
    (t_code, t_type)   = parse_T(itype)
    result_type = wider(e1_type, t_type)
    return (add_with_coerce(e1_code, t_code, result_type), result_type)
```

### 8.4.4 Backpatching: `if-else` translation

A canonical application of SDTs is the translation of control flow with unresolved jump targets [Dragon §6.6]. When generating three-address code for an `if-else` statement, the compiler needs to emit conditional jumps to the `then`-branch and `else`-branch before the target addresses are known. **Backpatching** defers the resolution: emit jumps with *placeholder* targets, record where the placeholders are, and fill them in once the targets are known.

Each nonterminal in the translation carries two lists of instruction addresses:

- **truelist**: addresses of jump instructions that should jump to the statement's "true" destination (the fall-through into the then-branch).
- **falselist**: addresses of jump instructions that should jump to the statement's "false" destination (the else-branch or the code after the if).
- **nextlist**: addresses of unconditional jumps that should jump to the code *after* the entire statement.

The SDT uses a **marker nonterminal M** to capture the address of the next instruction to be emitted:

```
-- Marker nonterminal
M  →  ε  { M.quad := nextquad }
         -- nextquad is a global counter of the next instruction address

-- Boolean expression (generates truelist / falselist)
B  →  B₁ or M B₂    { backpatch(B₁.falselist, M.quad)
                       B.truelist  := merge(B₁.truelist, B₂.truelist)
                       B.falselist := B₂.falselist }

B  →  B₁ and M B₂   { backpatch(B₁.truelist, M.quad)
                       B.truelist  := B₂.truelist
                       B.falselist := merge(B₁.falselist, B₂.falselist) }

B  →  not B₁         { B.truelist  := B₁.falselist
                        B.falselist := B₁.truelist }

B  →  E₁ relop E₂   { B.truelist  := makelist(nextquad)
                       B.falselist := makelist(nextquad + 1)
                       emit("if" E₁.place relop.op E₂.place "goto _")
                       emit("goto _") }

-- Statement translation
S  →  if B then M₁ S₁ N else M₂ S₂
      { backpatch(B.truelist,  M₁.quad)   -- B's true jumps → then-branch
        backpatch(B.falselist, M₂.quad)   -- B's false jumps → else-branch
        S.nextlist := merge(S₁.nextlist, N.nextlist, S₂.nextlist) }

N  →  ε  { N.nextlist := makelist(nextquad)
            emit("goto _") }              -- unconditional jump to after-else

S  →  if B then M S₁
      { backpatch(B.truelist, M.quad)
        S.nextlist := merge(B.falselist, S₁.nextlist) }
```

**Walk-through for `if (a < b) { s1 } else { s2 }`:**

Assuming `nextquad = 100` at the start:

| quad | instruction                        | notes                                |
|------|------------------------------------|--------------------------------------|
| 100  | `if a < b goto _`                  | B.truelist = {100}                   |
| 101  | `goto _`                           | B.falselist = {101}                  |
|      | *(M₁ fires: M₁.quad = 102)*        |                                      |
| 102  | `… code for s1 …`                  | S₁ translated here                   |
| 103  | `goto _`                           | N emits unconditional jump; nextlist = {103} |
|      | *(M₂ fires: M₂.quad = 104)*        |                                      |
| 104  | `… code for s2 …`                  | S₂ translated here                   |

`backpatch({100}, 102)` → quad 100 becomes `if a < b goto 102`.
`backpatch({101}, 104)` → quad 101 becomes `goto 104`.
`S.nextlist = {103} ∪ S₁.nextlist ∪ S₂.nextlist` — all these get resolved to the instruction after the if-else when the enclosing statement is processed.

**Backpatching for `while` loops.** The same machinery generalises to while loops, extending the backpatching pattern:

```
-- While loop translation
S  →  while M₁ B do M₂ S₁
      { backpatch(B.truelist,  M₂.quad)   -- B's true jumps into loop body
        backpatch(S₁.nextlist, M₁.quad)   -- end of body jumps back to test
        S.nextlist := B.falselist
        emit("goto " M₁.quad) }           -- unconditional back-edge to test
```

After translation of `while (a < b) { s1 }` with `M₁.quad = 100`, `M₂.quad = 103`:

| quad | instruction              |
|------|--------------------------|
| 100  | `if a < b goto 103`      |  ← B.truelist={100} backpatched to M₂.quad
| 101  | `goto _`                 |  ← B.falselist={101} (left for enclosing stmt to backpatch)
| 102  | `goto 100`               |  ← unconditional back-edge to M₁.quad
| 103  | `… code for s1 …`        |  ← body code starts here
| ...  | `goto 100`               |  ← S₁.nextlist backpatched to M₁.quad

The `S.nextlist = {101}` is passed to the enclosing statement, which will backpatch quad 101 to the first instruction after the while loop. This chain of `nextlist` propagation continues until all jumps are resolved when their target scope exits.

### 8.4.5 Connection to yacc/bison

yacc and bison implement S-attributed and (with limitations) L-attributed SDTs:

- `$$` is the synthesised value of the left-hand side, assigned in the action.
- `$1`, `$2`, `$n` are the synthesised values of the *n*-th symbol on the right-hand side.
- Inherited attributes require embedded mid-rule actions. An embedded action `{ $$ = f($1); }` midway through a production is implemented by bison as a fresh unit production, creating a new nonterminal. The value of this nonterminal (`$$`) becomes a `$k` reference for subsequent symbols.
- The `%union` declaration specifies the type of `YYSTYPE`, the type of all semantic values. `%type <field>` specifies which union field each nonterminal and terminal uses.
- `yylex()` provides the token type (as the function return value) and the semantic value (via `yylval`), which corresponds to the terminal's synthesised attribute.

---

## 8.5 Type-Checking as a Tree Walk

### 8.5.1 The typing judgment

The central object of formal type theory is the **typing judgment** Γ ⊢ e : τ, read "under type environment Γ, expression e has type τ." (Chapter 12 derives the complete inference rules for the Simply Typed Lambda Calculus; Chapter 13 extends them to Hindley-Milner polymorphism.) For our purposes in this chapter, we treat the type-checking problem concretely: given an AST node, determine its type by recursing over the tree.

**The type environment Γ** is precisely the symbol table restricted to type bindings: Γ(x) = τ means that variable x has been declared with type τ. The connection to §8.1 is direct: a chained symbol table *is* an operational realisation of the mathematical type environment Γ.

### 8.5.2 Recursive tree walk for monomorphic type checking

For a monomorphically typed language (no parametric polymorphism, no type inference), type checking is a direct S-attributed evaluation: each AST node's type is synthesised from its children's types. The algorithm is a recursive descent over the AST [EaC §5.4]:

```
-- Type-checking a monomorphic expression language
typecheck(node: ASTNode, Γ: SymTab) -> Type:
    match node with
    | LitInt(_)          -> int
    | LitFloat(_)        -> float
    | LitBool(_)         -> bool

    | Var(name)          ->
        τ = Γ.lookup(name)
        if τ == ⊥: error("undeclared variable: " + name)
        return τ

    | BinOp(op, e1, e2)  ->
        τ1 = typecheck(e1, Γ)
        τ2 = typecheck(e2, Γ)
        match op with
        | Add | Sub | Mul | Div ->
            if τ1 == τ2 and τ1 ∈ {int, float}:
                return τ1
            if {τ1, τ2} == {int, float}:
                -- insert coercion node; return float
                coerce_node(e1, e2, op)
                return float
            error("type mismatch in arithmetic")
        | Eq | Lt | Gt ->
            if τ1 == τ2: return bool
            error("comparison of incompatible types")

    | If(cond, then_, else_) ->
        τc   = typecheck(cond, Γ)
        if τc ≠ bool: error("if condition must be bool")
        τt   = typecheck(then_, Γ)
        τe   = typecheck(else_, Γ)
        if τt ≠ τe: error("branches have incompatible types")
        return τt

    | Let(x, τ_decl, rhs, body) ->
        τ_rhs = typecheck(rhs, Γ)
        if τ_rhs ≠ τ_decl: error("let binding type mismatch")
        Γ' = Γ.insert(x, τ_decl)
        return typecheck(body, Γ')

    | App(f, arg)        ->
        τ_f   = typecheck(f, Γ)
        τ_arg = typecheck(arg, Γ)
        match τ_f with
        | Fun(τ_param, τ_ret) ->
            if τ_arg ≠ τ_param: error("argument type mismatch")
            return τ_ret
        | _ -> error("application of non-function")
```

This is a single bottom-up (post-order) pass over the AST: every recursive call returns a type, and the parent node's type is computed from the returned types. The type environment `Γ` is threaded as a parameter and is extended only for `Let` bindings, using the persistent-table style of §8.1.3.

**Error recovery in type checking.** A production compiler must continue after a type error to report as many further errors as possible. The standard technique is to introduce an **error type** ⊥ that propagates through all type-checking rules without triggering further errors. When `typecheck` detects a type mismatch, it emits a diagnostic and returns ⊥. All type-checking rules that receive ⊥ as an input silently return ⊥ without emitting additional diagnostics. This suppresses cascading errors that would otherwise overwhelm the user with hundreds of spurious messages from a single root cause.

```
-- Error-propagating type rule for BinOp
| BinOp(op, e1, e2) ->
    τ1 = typecheck(e1, Γ)
    τ2 = typecheck(e2, Γ)
    if τ1 == ⊥ or τ2 == ⊥: return ⊥   -- suppress cascading errors
    -- … normal type-checking logic …
```

Clang implements this with the `ExprError()` return value from `Sema::ActOnBinOp` and related methods. An invalid expression node (`ExprError`) carries the error type and is recognised by all subsequent semantic operations, which skip analysis and propagate the error.

### 8.5.3 Polymorphic type checking and interaction with type inference

For Hindley-Milner polymorphic type systems (ML, Haskell, OCaml, Rust), type checking and type inference are interleaved in a single pass using **Algorithm W** (Damas-Milner, 1982) [Damas-Milner 1982; Chapter 13]. The type environment maps names to *type schemes* ∀α₁…αₙ.τ rather than monotypes. The inference algorithm proceeds by:

1. Assigning fresh **unification variables** (also called *type holes* or *type metavariables*) α, β, … to subexpressions whose types are not yet known.
2. Generating **type constraints** from each constructor: for `App(f, arg)` with `f: α` and `arg: β`, emit the constraint `α = β → γ` (f is a function from β to some result type γ).
3. Solving the constraints via **unification** (Robinson 1965). Unification finds the most general type assignment; a failure indicates a type error.
4. **Generalisation** at `let`-bindings: free unification variables not constrained by the outer context are universally quantified to produce a type scheme.

The key insight for the tree walk is that Algorithm W is itself a structural recursion over the AST [Chapter 13, §13.3]. The "S-attributed" character is preserved: each node is processed after its children, and the type assigned to a node is a function of the types of its children. The difference from monomorphic checking is that unification may propagate type information both bottom-up (through return values) and effectively top-down (through the unification substitution), but this is handled by the mutable substitution environment, not by inherited attributes in the grammar sense.

### 8.5.4 Coercions and implicit conversions

Type checking in C and C++ must insert **implicit coercion nodes** into the AST to represent implicit conversions [Dragon §6.5.2]. The coercion-insertion is an attributed translation: as the type checker processes each node and determines the expected type (an inherited attribute, in the attribute-grammar sense), it compares the actual type of the subexpression with the expected type and inserts a coercion node when necessary.

**C's usual arithmetic conversions [C11 §6.3.1.8]** apply to operands of binary operators:
1. If either operand is `long double`, convert the other to `long double`.
2. If either operand is `double`, convert the other to `double`.
3. If either operand is `float`, convert the other to `float`.
4. Apply integer promotions (e.g., `short` → `int`, `char` → `int`).
5. If both operands are signed or both unsigned, convert the type of smaller rank to the type of larger rank.
6. Handle mixed signed/unsigned cases.

Each conversion rule corresponds to inserting a coercion node (a `CastExpr` in Clang's AST, typically with a `CastKind` of `IntegralToFloating`, `IntegralCast`, `FloatingCast`, etc.) between the expression and its parent.

**C++'s implicit constructors and conversion operators.** C++ extends implicit conversion with user-defined conversions: a single-argument constructor (`Foo::Foo(int)`) is an implicit converting constructor; a `Foo::operator int()` is a conversion operator. When type checking a function call `f(e)` where the argument type does not exactly match the parameter type, the compiler searches for a sequence of at most one standard conversion + at most one user-defined conversion + at most one standard conversion (the **three-tier model** of C++ conversion sequences) [C++17 §12.3].

### 8.5.5 Overload resolution as type-directed disambiguation

In C++, a function name may be **overloaded**: multiple function declarations share the same name but differ in parameter types. **Overload resolution** [C++17 §16.3] is performed at every call site and proceeds in three phases:

1. **Candidate set.** Collect all functions visible with the given name via unqualified lookup, qualified lookup, and ADL (§8.6.3). Add built-in operator candidates for operator expressions.
2. **Viable functions.** Filter the candidate set to those functions that can be called with the given arguments, considering implicit conversions, default arguments, and variadic functions.
3. **Best viable function.** Among viable functions, select the one that is *not worse* than every other viable function in every parameter position. The ordering is defined by comparing implicit conversion sequences for each argument. The **best implicit conversion sequence** for a given (argument, parameter) pair is the shortest sequence in the three-tier hierarchy.

If no single best viable function exists — either no viable functions (error: no match) or multiple equally ranked viable functions (error: ambiguous call) — overload resolution fails and the compiler emits a diagnostic.

Overload resolution is **type-directed disambiguation**: the types of the arguments guide the selection of the correct function body to compile. It is an instance of the general principle that semantic analysis resolves ambiguities that parsing leaves open.

**Ranking conversion sequences.** C++17 [§16.3.3.2] defines a partial order on implicit conversion sequences:

1. **Exact match** — no conversion, lvalue-to-rvalue conversion, array-to-pointer, function-to-pointer, cv-qualification adjustment.
2. **Promotion** — integral promotion (char→int, short→int), floating-point promotion (float→double).
3. **Conversion** — any standard conversion that is not an exact match or promotion: integral conversion (int→long), floating-to-integral, pointer conversion, etc.
4. **User-defined conversion** — one user-defined conversion function or constructor, surrounded by at most one standard conversion on each side.
5. **Ellipsis** — varargs `(...)` match; accepts any argument.

A conversion sequence S1 is *better* than S2 if S1 is an earlier category than S2, or if both are in the same category but S1's conversion is more specific. A function F1 is *better* than F2 if for every argument position, F1's conversion sequence is not worse than F2's and for at least one argument position it is strictly better. If neither F1 nor F2 is better, and both are templates, additional tiebreakers apply (partial ordering of templates, explicit specialisations).

**Worked example: overload resolution for `print(3.14)`.**

```cpp
void print(int x);            // candidate A
void print(double x);         // candidate B
void print(const std::string& s); // candidate C

print(3.14);  // argument type: double
```

- Candidate A: double→int is an *integral conversion* (category 3) with potential precision loss.
- Candidate B: double→double is an *exact match* (category 1).
- Candidate C: double→std::string would require a user-defined conversion; std::string has no constructor from double — not viable.

Candidate B is strictly better than A (exact match vs. conversion), and C is not viable. Overload resolution selects `print(double)`. Changing the call to `print(3)` would select `print(int)` (exact match for int) over `print(double)` (promotion from int to double).

---

## 8.6 Name Resolution and Argument-Dependent Lookup

### 8.6.1 The name resolution problem

**Name resolution** is the function σ: (NameUse × Program) → Declaration that maps every occurrence of a name in the source to the declaration it refers to [EaC §5.3]. For a program to be well-formed, σ must be total (every use refers to some declaration), consistent (each use refers to exactly one declaration), and temporally valid (declarations of certain forms must precede certain uses, as in C's use-before-declaration rule).

Name resolution is *not* fully compositional with context-free parsing: C++'s name lookup rules depend on types (ADL, dependent names), and Pascal's rule that declarations precede uses is a semantic constraint not enforceable by a CFG. The boundary between parsing and name resolution is fuzzy in C++ specifically because parsing of some constructs (the vexing parse, template disambiguation) depends on whether names denote types or values — precisely the question name resolution answers.

### 8.6.2 Unqualified name lookup

**Unqualified name lookup** resolves a name written without any qualifier (no `::`, no `.`, no `->`) by searching the scope chain outward from the innermost enclosing scope [C++17 §6.4.1]:

```
unqualified_lookup(name, scope):
    while scope ≠ null:
        decls = scope.lookup(name)
        if decls ≠ ∅:
            return decls
        scope = scope.enclosing_scope
    error("undeclared identifier: " + name)
```

For most name uses, unqualified lookup suffices. The result is the first scope (innermost) that contains a declaration for the name. In C++, the search halts at the first scope that contains *any* declaration for the name — even if the declaration is a forward declaration or an overload set. All overloads within that scope are returned; the selection among them is done by overload resolution.

**Class scope lookup.** For a name used inside a class member function, the lookup order is: local block scopes → function parameter scope → class scope → base class scopes (in inheritance order) → enclosing namespace scopes [C++17 §6.4.1 ¶6]. This means a member function can implicitly access all members and base-class members without qualification.

**Lookup across `using` declarations.** A `using` declaration (`using std::cout`) introduces a name from another namespace into the current scope. A `using-directive` (`using namespace std`) makes all names from a namespace visible for unqualified lookup from the current scope, but at a lower priority than explicitly declared names in enclosing scopes.

### 8.6.3 Qualified name lookup

**Qualified name lookup** starts from a specified scope, rather than the innermost enclosing scope [C++17 §6.4.3]:

- `N::x` looks up `x` in namespace `N`.
- `C::m` looks up `m` in class `C`, then in `C`'s base classes.
- `::x` looks up `x` in the global namespace.
- `expr.m` and `expr->m` look up `m` in the static type of `expr` (member access).

Qualified lookup does not search enclosing scopes: `N::x` finds `x` in `N` or reports an error; it does not fall back to the enclosing namespace.

### 8.6.4 Two-phase name lookup in C++ templates

Two-phase lookup is one of the most important and widely misunderstood aspects of C++ name resolution [C++17 §17.6.3, EaC §5.5]. When a function template or class template is *defined*, the compiler performs a **first-phase lookup** that resolves all names that are *not dependent* on a template parameter. When the template is *instantiated*, the compiler performs a **second-phase lookup** for names that *are* dependent.

```cpp
// Example demonstrating two-phase lookup
template <typename T>
void f(T x) {
    g(x);       // 'g' is a dependent name (argument type depends on T)
                // Looked up at instantiation time (phase 2) + ADL

    h();        // 'h' is NOT dependent (no template argument involved)
                // Looked up at definition time (phase 1)
                // If h is not visible here, error — even if h is defined
                // before instantiation.
}

void h() {}     // Too late for phase 1 — 'f' is already defined
```

**Phase 1 (definition time):** Non-dependent names are looked up using the normal unqualified lookup rules, with the enclosing scope at the point of template *definition*. If a non-dependent name is not visible at definition time, the program is ill-formed even if the name would be visible at instantiation time.

**Phase 2 (instantiation time):** Dependent names (names that depend on a template parameter) are looked up at the point of *instantiation*, in the instantiation context (the enclosing scope at the instantiation site) augmented with the results of **ADL** based on the template arguments.

This two-phase design has a critical consequence: code inside a template that calls a non-member function depending on a template type *must* rely on ADL to find that function. If the function is not in an associated namespace, it must be explicitly declared before the template definition.

```cpp
namespace N {
    struct Widget {};
    void serialize(Widget w);         // associated with Widget
}

template <typename T>
void save(T x) {
    serialize(x);   // resolved via ADL at instantiation:
                    // when T=N::Widget, N is an associated namespace
}

save(N::Widget{});  // OK: phase-2 + ADL finds N::serialize
```

**The `typename` and `template` disambiguators.** In phase 2, a dependent name may refer either to a type or to a value/function. The compiler cannot know which until instantiation, yet parsing requires knowing. C++17 resolves this with disambiguation keywords [C++17 §17.7]:

- `typename T::nested_type` — asserts that `nested_type` is a type, allowing it to appear in a declaration context.
- `obj.template member<int>()` — asserts that `member` is a template, allowing `<` to be parsed as a template argument list rather than a less-than operator.

Without these disambiguators, a parser seeing `T::foo` at phase 1 cannot determine if `foo` is a type or a value, and `T::foo<1>` is ambiguous between `(T::foo) < 1` and a template instantiation. The disambiguators shift the responsibility to the programmer to tell the parser what it cannot yet infer.

**GCC vs. MSVC divergence.** Historically, Microsoft's MSVC did not implement two-phase lookup; it effectively performed all name lookup at instantiation time (a single phase). This resulted in MSVC accepting programs that were ill-formed under the standard (non-dependent names not visible at definition time). MSVC began implementing conformant two-phase lookup starting with `/permissive-` mode in Visual Studio 2017 and made it the default in VS 2019.

The divergence created a class of programs that compiled with MSVC but not GCC/Clang and vice versa. The most common pattern: a non-dependent function call in a template body, where the function is defined *after* the template definition but *before* any instantiation. GCC/Clang correctly reject this (the name is not visible at definition time), while pre-conformant MSVC accepted it. Porting code between compilers requires understanding this difference.

**Clang's implementation.** Clang implements two-phase lookup in `Sema::LookupName`. Non-dependent names are looked up immediately via `LookupUnqualifiedName`; dependent names generate `UnresolvedLookupExpr` AST nodes that are resolved at instantiation via `Sema::SubstDecl` and the template instantiation machinery in `SemaTemplateInstantiate.cpp`. ADL for dependent calls is performed as part of the instantiation-time lookup [Chapter 33].

### 8.6.5 Argument-Dependent Lookup (ADL / Koenig lookup)

**Argument-Dependent Lookup** (ADL, named after Andrew Koenig) is the rule that, for a function call with at least one argument of class or enumeration type, the set of namespaces searched for the function name is augmented with the **associated namespaces** of the argument types [C++17 §6.4.2].

**The associated namespace/class set** for a type τ is computed as follows:

- For a class type `C`: the namespace containing `C`, plus the namespaces of `C`'s base classes, plus the namespace of the class template (if `C` is a template specialisation), recursively.
- For a fundamental type: no associated namespaces.
- For a pointer/reference type: the associated namespaces of the pointed-to type.
- For a function type: the associated namespaces of the parameter types and return type.
- For a template specialisation: the associated namespaces of the template arguments in addition to the template itself.

**Canonical example: `swap`.**

```cpp
#include <vector>
#include <algorithm>    // std::swap

std::vector<int> a, b;
swap(a, b);             // unqualified call
```

Without ADL, this call would require `std::swap(a, b)` because `swap` is in namespace `std`. With ADL, the associated namespace of `std::vector<int>` is `std`, so `std::swap` is included in the candidate set. The call resolves to `std::swap<std::vector<int>>`.

**Why ADL is necessary.** ADL is indispensable for operators. When writing `a + b` for user-defined types, the compiler desugars this to `operator+(a, b)`. If `a` and `b` are of type `N::T`, the `+` operator is naturally declared in namespace `N` alongside the type. Without ADL, every use of such an operator would require explicit qualification: `N::operator+(a, b)`. ADL makes the operator syntax usable.

```cpp
namespace Algebra {
    struct Matrix { … };
    Matrix operator+(const Matrix&, const Matrix&);
    Matrix operator*(const Matrix&, const Matrix&);
    std::ostream& operator<<(std::ostream&, const Matrix&);
}

Algebra::Matrix m1, m2;
auto m3 = m1 + m2;    // ADL finds Algebra::operator+ from Algebra::Matrix
std::cout << m3;      // ADL finds Algebra::operator<< (arg m3 → namespace Algebra)
```

**Friend declarations and ADL.** A `friend` function declared inside a class body is *injected* into the enclosing namespace but may not be directly accessible via unqualified lookup unless called with an argument of the class's type (via ADL). This is the *hidden-friend* idiom:

```cpp
namespace N {
    class C {
        friend C operator+(C, C);   // hidden: not visible without ADL
    };
}
N::C a, b;
auto c = a + b;   // OK: ADL on N::C finds the friend operator+
```

**ADL and overload resolution interaction.** The candidate set for overload resolution = (names from unqualified lookup) ∪ (names found by ADL). These sets are merged before any viability filtering. ADL can therefore introduce additional overload candidates that would not otherwise be visible, which can sometimes resolve an ambiguity or, in poorly designed interfaces, create one.

---

## 8.7 The Visitor Pattern as a Compiled Attribute Grammar

### 8.7.1 The Gang of Four Visitor pattern

The **Visitor pattern** [Gamma et al., *Design Patterns*, 1994] separates an algorithm from the object structure it operates on. In the context of ASTs, the pattern provides a type-safe way to define new tree traversals without modifying the AST node classes.

The protocol involves two hierarchies:

1. **Node hierarchy.** Each AST node class `ASTNode` has a virtual `accept(Visitor&)` method. The base class declares `accept` as pure virtual; each concrete subclass overrides it to call the visitor's `visit` method with itself as the argument.

2. **Visitor hierarchy.** The `Visitor` abstract base class declares a pure virtual `visit` overload for each concrete node type. Each concrete visitor subclass implements all `visit` overloads.

```cpp
// The double-dispatch protocol
struct Visitor;

struct ASTNode {
    virtual void accept(Visitor& v) = 0;
};

struct LitIntExpr  : ASTNode { int value;  void accept(Visitor& v) override; };
struct BinOpExpr   : ASTNode { Op op; ASTNode* lhs; ASTNode* rhs;
                                void accept(Visitor& v) override; };
struct VarRefExpr  : ASTNode { std::string name;
                                void accept(Visitor& v) override; };

struct Visitor {
    virtual void visit(LitIntExpr&)  = 0;
    virtual void visit(BinOpExpr&)   = 0;
    virtual void visit(VarRefExpr&)  = 0;
};

// Concrete node: accept dispatches to the right visit overload
void LitIntExpr::accept(Visitor& v) { v.visit(*this); }
void BinOpExpr::accept(Visitor& v)  { v.visit(*this); }
void VarRefExpr::accept(Visitor& v) { v.visit(*this); }
```

**Double dispatch.** The method `node->accept(v)` performs two virtual dispatch steps: one on the dynamic type of `node` (to reach the correct `accept` override) and one on the dynamic type of `v` (implicit in the `v.visit(...)` call, which selects the overloaded `visit` based on the concrete type passed). The combination achieves runtime dispatch on *both* the node type and the visitor type.

### 8.7.2 How a visitor implements attribute computation

A visitor corresponds directly to an attribute grammar evaluator. Each `visit` method computes the attribute for one concrete node type:

```cpp
// Type-checking visitor: computes the 'type' synthesised attribute
class TypeCheckVisitor : public Visitor {
    SymTab& symtab;
    Type    result;           // the synthesised 'type' attribute

public:
    TypeCheckVisitor(SymTab& s) : symtab(s) {}

    void visit(LitIntExpr& node) override {
        result = Type::Int;
    }

    void visit(VarRefExpr& node) override {
        auto t = symtab.lookup(node.name);
        if (!t) throw TypeError("undeclared: " + node.name);
        result = *t;
    }

    void visit(BinOpExpr& node) override {
        // Recurse into children (synthesise left and right types)
        node.lhs->accept(*this); Type tlhs = result;
        node.rhs->accept(*this); Type trhs = result;
        // Compute this node's type from children's types
        if (tlhs == trhs && isArithmetic(tlhs))
            result = tlhs;
        else if (isNumeric(tlhs) && isNumeric(trhs))
            result = wider(tlhs, trhs);
        else
            throw TypeError("type mismatch in binary op");
    }
};
```

The `result` field plays the role of the synthesised attribute: after `accept` returns, `result` holds the type of the visited subtree. The `visit(BinOpExpr&)` method first recurses into `lhs` (saving its type), then recurses into `rhs` (saving its type), then computes the current node's type — exactly the post-order S-attributed evaluation of §8.3.2.

### 8.7.3 S-attributed and L-attributed grammars in visitor form

**S-attributed grammar ≡ visitor with return value (or thread-via-field).** An S-attributed grammar's evaluator is a post-order tree traversal with no downward information flow. This maps exactly onto a visitor where `visit` methods only read from children (by recursing and reading `result`) and write to the parent (by setting `result` before returning). No state needs to be passed *into* a node; everything flows upward.

**L-attributed grammar ≡ visitor with inherited parameter threading.** An L-attributed grammar passes inherited attributes from parent to children. In the visitor pattern, inherited attributes are threaded either as (a) explicit parameters to a richer `visit` signature, (b) instance fields set by the parent before recursing into a child, or (c) a separate stack of "context" objects maintained alongside the traversal.

```cpp
// L-attributed threading: pass type context as a parameter
class CoercionInsertionVisitor : public Visitor {
    SymTab& symtab;
    Type    result;
    Type    expectedType;   // the inherited attribute: expected type from context

public:
    void visit(BinOpExpr& node) override {
        // Synthesise result type first (S-attributed sub-computation)
        node.lhs->accept(*this); Type tlhs = result;
        node.rhs->accept(*this); Type trhs = result;
        result = wider(tlhs, trhs);

        // Now thread expected type down into children (L-attributed)
        Type saved = expectedType;
        expectedType = result;          // pass wider type as context
        node.lhs->accept(*this);        // re-visit with inherited context
        if (result != expectedType)
            insert_coerce(node.lhs, result, expectedType);
        node.rhs->accept(*this);
        if (result != expectedType)
            insert_coerce(node.rhs, result, expectedType);
        expectedType = saved;
        result = expectedType;          // node's type is the expected type
    }
};
```

This double-traversal pattern (once to compute synthesised types, once to pass inherited context) is cumbersome. In practice, compilers often merge the two passes or use a different representation of inherited context (e.g., a scope-threaded environment in the recursive-descent style of §8.5.2).

### 8.7.4 LLVM and Clang realisations

**LLVM's `InstVisitor<Derived>`.** LLVM uses a Curiously Recurring Template Pattern (CRTP) variant of the Visitor pattern for instruction visitors [Chapter 4, §4.5]. The `InstVisitor<Derived>` template uses static dispatch via CRTP rather than virtual dispatch, eliminating virtual call overhead:

```cpp
template <typename Derived, typename RetTy = void>
class InstVisitor {
public:
    RetTy visit(Instruction& I) {
        // Dispatches to visitXXX in Derived based on instruction opcode
        switch (I.getOpcode()) {
        case Instruction::Add:  return static_cast<Derived*>(this)->visitAdd(cast<BinaryOperator>(I));
        // … all opcodes …
        }
    }
    // Default implementations (call visitInstruction)
    RetTy visitAdd(BinaryOperator& I)  { return visitBinaryOperator(I); }
    RetTy visitBinaryOperator(BinaryOperator& I) { return visitInstruction(I); }
    RetTy visitInstruction(Instruction& I) { return RetTy(); }
};
```

The CRTP visitor avoids the cost of two virtual dispatches per node, which matters when traversing millions of IR instructions. The `RetTy` template parameter allows the visitor to return a value, implementing the synthesised-attribute pattern directly.

**Clang's `RecursiveASTVisitor<Derived>`.** Clang provides `RecursiveASTVisitor<Derived>` (in `clang/AST/RecursiveASTVisitor.h`), another CRTP visitor that automatically recurses into children unless overridden [Chapter 46, §46.2]. The derived class overrides `VisitXXX` methods for the node types of interest; the base class generates the full traversal. This corresponds to an S-attributed evaluator with default "pass-through" rules for unrecognised node types.

### 8.7.5 Limitations: the expression problem

The Visitor pattern has a fundamental duality: it is easy to add new operations (new `Visitor` subclasses) but hard to add new node types (requires updating every existing `Visitor`). This is the **Expression Problem** [Wadler 1998]:

- **Object-oriented decomposition** (class hierarchy with virtual methods) makes adding new types easy (subclass) but adding new operations hard (modify all existing classes).
- **Functional decomposition** (ADTs with pattern matching) makes adding new operations easy (new function) but adding new types hard (modify all existing pattern matches).

The visitor pattern inverts the OO decomposition: by pulling operations out into visitor classes, it makes adding operations easy at the cost of making adding types hard.

Production compilers that change their AST infrequently (adding a new node type for each new language feature, which happens maybe once per language version) and change their analysis passes frequently (optimisations, checks, transforms) can live with this trade-off. Extensible AST designs — like MLIR's operation system where operations are defined by dialects and transformations are defined as passes, or like Roslyn's visitor infrastructure for C# — address the expression problem at the cost of more complex dispatch machinery.

**Pattern matching and algebraic data types** as an alternative. Functional languages (OCaml, Haskell, Standard ML, Rust, Scala) use algebraic data types and exhaustive pattern matching as the primary AST traversal mechanism. Pattern matching is syntactic sugar for a function over a disjoint union; the compiler checks exhaustiveness (every variant is handled) at compile time. This is equivalent to a visitor where every `visit` method is mandatory, but with no runtime dispatch overhead for the dispatch itself (the match is compiled to a decision tree).

The OCaml implementation of a type-checker for our expression grammar illustrates this clearly:

```ocaml
(* OCaml: pattern-matching replaces the visitor protocol *)
type expr =
  | LitInt  of int
  | LitFloat of float
  | Var     of string
  | BinOp   of op * expr * expr
  | Let     of string * typ * expr * expr

let rec typecheck (gamma : env) (e : expr) : typ =
  match e with
  | LitInt _   -> TInt
  | LitFloat _ -> TFloat
  | Var x      ->
      (match lookup gamma x with
       | Some t -> t
       | None   -> error ("undeclared: " ^ x))
  | BinOp (op, e1, e2) ->
      let t1 = typecheck gamma e1 in
      let t2 = typecheck gamma e2 in
      (match (t1, t2) with
       | (TInt,   TInt)   -> TInt
       | (TFloat, TFloat) -> TFloat
       | (TInt,   TFloat)
       | (TFloat, TInt)   -> TFloat   (* widening *)
       | _ -> error "type mismatch")
  | Let (x, t_decl, rhs, body) ->
      let t_rhs = typecheck gamma rhs in
      if t_rhs <> t_decl then error "let binding mismatch";
      typecheck (extend gamma x t_decl) body
```

The OCaml compiler statically verifies that the `match` covers all constructors of `expr`. If a new constructor is added to the ADT, every pattern-matching function becomes a compile-time warning (or error with `-warn-error`) until it handles the new case. This inverts the visitor trade-off: adding a node type requires updating all functions (easy to audit via exhaustiveness warnings), while adding a new traversal requires only a new function.

MLIR's operation system and Rust's `enum`-based IR representations (e.g., Cranelift's `Inst`) are practical realisations of the ADT approach at scale. MLIR additionally provides op interfaces (Chapter 133) as a mechanism to define operations that can participate in generic algorithms without requiring the algorithm to enumerate all concrete op types — a hybrid solution that partially addresses the expression problem.

---

## 8.8 Chapter Summary

- **Symbol tables** map names to their declaration attributes. Chained (linked-list) symbol tables implement scope as a stack of frames and provide O(depth × scope\_size) lookup with O(1) scope management. Hash-based tables with per-entry binding stacks provide O(1) expected lookup with O(|scope|) scope retirement. Persistent (functional) tables using HAMTs or balanced BSTs support O(1) amortised lookup with structural sharing, enabling multi-branch and parallel compilation without explicit scope reversal.

- **Lexical (static) scoping** ties each name use to the textually innermost enclosing declaration; it is the default for all modern languages and is the foundation for referential transparency at the name level. **Dynamic scoping** ties each use to the most recently activated runtime binding; it is unpredictable and incompatible with modular reasoning. C++ hierarchical scoping — file, namespace, class, function, block, template — adds the scope resolution operator and qualified lookup as explicit override mechanisms.

- **Attribute grammars** [Knuth 1968] formalise semantic computations as equations over parse-tree attributes. **Synthesised attributes** flow bottom-up (children → parent); an S-attributed grammar (all synthesised) evaluates in a single LR-compatible pass. **Inherited attributes** flow top-down (parent and siblings → child); an L-attributed grammar evaluates in a single left-to-right recursive-descent-compatible pass. Well-definedness requires an acyclic dependency graph on every parse tree; S-attributed and L-attributed grammars both guarantee this structurally.

- **Syntax-directed translation schemes** (SDTs) embed semantic actions at specific positions within production right-hand sides, making the evaluation timing explicit. Postfix SDTs (all actions at the right end) implement S-attributed semantics directly in LR parsers. L-attributed SDTs allow mid-production actions that reference only left siblings and the parent's inherited attributes. Backpatching — generating jumps with placeholder targets and filling them in when targets are known — is the canonical application for control-flow SDTs, using `truelist`/`falselist`/`nextlist` attributes and the marker nonterminal `M → ε { M.quad := nextquad }`.

- **Type checking as a tree walk** implements the judgment Γ ⊢ e : τ as a post-order AST traversal. The type environment Γ is operationally the symbol table restricted to type bindings. Monomorphic type checking is a single S-attributed pass. Polymorphic type checking (Hindley-Milner) is also a single structural recursion, but unification propagates information through a mutable substitution, giving the effect of bidirectional flow. Coercion insertion (C's arithmetic conversions, C++'s implicit constructors) requires threading an inherited type-context attribute. Overload resolution selects the best viable function from the candidate set by ranking implicit conversion sequences.

- **Name resolution** maps each name use to its declaration via the scoping rules. Unqualified lookup searches the scope chain outward; qualified lookup starts from a specified scope. **Two-phase name lookup in C++ templates** resolves non-dependent names at template definition time (phase 1) and dependent names at instantiation time (phase 2, augmented by ADL). **Argument-Dependent Lookup (ADL / Koenig lookup)** extends the candidate namespace for function calls to the associated namespaces and classes of the argument types; it is indispensable for user-defined operators and `swap`-style customisation points.

- **The Visitor pattern** is a direct implementation of the attribute grammar evaluator. An S-attributed grammar maps onto a visitor that computes synthesised attributes bottom-up via post-order traversal. An L-attributed grammar requires threading inherited attributes as extra parameters or instance fields. LLVM's `InstVisitor<Derived>` and Clang's `RecursiveASTVisitor<Derived>` are production CRTP realisations that eliminate virtual dispatch overhead. The expression problem — easy to add operations, hard to add types — limits extensibility; functional languages address this with ADTs and exhaustive pattern matching.

- **The theoretical framework of this chapter — attribute grammars, SDTs, and the typing judgment — underpins the design of every major component of Clang's Sema phase** (Chapters 33–34: name lookup, overload resolution, template instantiation), the AST visitor infrastructure (Chapter 46: `RecursiveASTVisitor`, AST matchers), and ultimately the way MLIR dialects define verification rules for their operations (Part XIX: MLIR dialect verification as an S-attributed walk over the IR tree).

---

*References:*

- Aho, Lam, Sethi, Ullman. *Compilers: Principles, Techniques, and Tools*, 2nd ed. Addison-Wesley, 2006. (Dragon Book) — §5.1–5.3 (attribute grammars, S-attributed, L-attributed), §5.4 (SDTs), §6.1 (symbol tables), §6.3 (scoping), §6.5–6.6 (type checking, backpatching)
- Cooper, Torczon. *Engineering a Compiler*, 3rd ed. Morgan Kaufmann, 2022. (EaC) — §5.1–5.5 (symbol tables, scoping, attribute grammars), §5.7 (type systems), §4.2 (SDTs)
- Appel, A.W. *Modern Compiler Implementation in ML*. Cambridge University Press, 1998. — Ch. 1 (symbol tables, environments), Ch. 4 (type checking)
- Knuth, D.E. "Semantics of Context-Free Languages." *Mathematical Systems Theory* 2(2), 1968. pp. 127–145. — Original attribute grammar paper; introduces synthesised and inherited attributes, well-definedness, and the dependency graph.
- Damas, L. and Milner, R. "Principal Type-Schemes for Functional Programs." *POPL 1982*. — Algorithm W for Hindley-Milner type inference.
- Gamma, E., Helm, R., Johnson, R., Vlissides, J. *Design Patterns: Elements of Reusable Object-Oriented Software*. Addison-Wesley, 1994. — Chapter on the Visitor pattern.
- Wadler, P. "The Expression Problem." Email to the Java Generics mailing list, 12 November 1998. — Formal statement of the OO vs. functional decomposition trade-off.
- ISO/IEC 14882:2017 (C++17 Standard) — §6.4.1 (unqualified lookup), §6.4.2 (ADL), §6.4.3 (qualified lookup), §17.6.3 (two-phase lookup), §12.3 (user-defined conversions), §16.3 (overload resolution).
- ISO/IEC 9899:2011 (C11 Standard) — §6.3.1.8 (usual arithmetic conversions).
- Bagwell, P. "Ideal Hash Trees." Technical Report, EPFL, 2001. — Original HAMT paper.
- Clang source code: `clang/lib/AST/DeclBase.cpp` (DeclContext), `clang/lib/Sema/SemaLookup.cpp` (lookup algorithms), `clang/include/clang/AST/RecursiveASTVisitor.h`.
- GCC source code: `gcc/cp/name-lookup.c` (cp_binding_level, pushlevel/poplevel).
- LLVM source code: `llvm/include/llvm/IR/InstVisitor.h` (InstVisitor CRTP).


---

@copyright jreuben11
