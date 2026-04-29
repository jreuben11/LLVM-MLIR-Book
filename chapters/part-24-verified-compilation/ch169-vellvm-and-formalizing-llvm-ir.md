# Chapter 169 — Vellvm and Formalizing LLVM IR

*Part XXIV — Verified Compilation*

LLVM IR is the pivot point of a compiler ecosystem used by billions of devices daily. Yet its semantics is specified only in the LLVM Language Reference Manual — English prose of approximately 150,000 words, full of precise intent but also ambiguity, underspecification, and occasional internal contradiction. Passes in `opt` rely on an informal understanding of what IR instructions mean; bugs arise when two developers' mental models diverge on edge cases involving undefined behavior, pointer provenance, or the interaction of `undef` with optimization. Vellvm (Verified LLVM) is the effort to give LLVM IR a precise, machine-checkable formal semantics — one that can serve as the foundation for proving optimization passes correct, for automatically discovering miscompilation bugs, and for understanding the true meaning of programs compiled through the LLVM pipeline. This chapter examines Vellvm's architecture, its use of interaction trees for effectful semantics, its memory model, verified transformations, and the broader landscape of LLVM IR formalization efforts.

---

## 169.1 The Need for LLVM IR Semantics

### The Specification Gap

The LLVM Language Reference Manual ([llvm.org/docs/LangRef.html](https://llvm.org/docs/LangRef.html)) specifies LLVM IR's behavior via English text. For most instructions, this is adequate: `add i32 %a, %b` adds two 32-bit integers with two's complement wrapping. But the specification gaps are numerous and consequential:

**Undefined behavior**: Operations like division by zero, shift by ≥ bitwidth, and signed overflow with `nsw` are all "undefined behavior" in LLVM IR. But what exactly does "undefined" mean? The LangRef says "this instruction has undefined behavior if..."; it does not say whether "undefined behavior" means the result is arbitrary, the program is allowed to terminate abnormally, or the compiler can assume the condition never occurs (the "value semantics" vs "trap semantics" question, relevant to miscompilation).

**`undef` and `poison`**: the LangRef describes `undef` as "a value that may evaluate to any bit pattern" and `poison` as a "stronger" form of `undef`. But the precise interaction rules (e.g., can the same `undef` value appear with different bit patterns in different uses of the same SSA value?) were not formally specified until the work around Alive2 and the Vellvm project forced the issue.

**Pointer semantics**: the LangRef describes `getelementptr` and memory accesses, but the question of when two pointers may alias (provenance rules), what happens when a pointer is cast to an integer and back, and whether pointer arithmetic that "wraps around" the address space is defined — these were subjects of ongoing debate in the LLVM community throughout the 2010s.

**Memory model for concurrent programs**: the LangRef added `atomic` instructions in LLVM 3.1 with a description based on the C11 memory model, but the interaction between atomic and non-atomic accesses in the presence of optimizations like code motion and load elimination was not formally verified.

### The Cost of Ambiguity

The cost of semantic ambiguity is real:
- Compiler bugs arise when a pass developer and an IR consumer have inconsistent understandings of an edge case.
- The CSmith experiment (Yang et al., 2011) found hundreds of miscompilations in LLVM; many were caused by passes making optimizations valid under one interpretation of UB but invalid under another.
- Security researchers have exploited UB-based miscompilations: an optimizer removes a null pointer check (reasoning: "if the pointer were null, the dereference two lines earlier would be UB, so we can assume it's non-null") that the programmer intended as a security guard.

A formal semantics resolves ambiguity by construction: it says exactly what each instruction means, for every input, including the boundary cases. Any optimization that preserves the formal semantics is correct; any that doesn't is a bug.

---

## 169.2 Vellvm Architecture

Vellvm (Zhao, Nagarakatte, Martin, Zdancewic; University of Pennsylvania; first published POPL 2012, significantly revised with interaction trees by Zakowski, Beck, et al., 2021) is a Coq formalization of LLVM IR.

### Repository Structure

Vellvm lives at [github.com/vellvm/vellvm](https://github.com/vellvm/vellvm). Key directories:

```
src/
  coq/
    Syntax/
      LLVMAst.v         -- algebraic data types for LLVM IR AST
      TypToDtyp.v       -- type translation to dynamic types
    Semantics/
      TopLevel.v        -- the main interpreter entry point
      IntrinsicsDefinitions.v -- built-in intrinsic semantics
      DynamicValues.v   -- runtime values (DVALUE, UVALUE)
      DynamicTypes.v    -- runtime type system
      GepM.v            -- getelementptr semantics
      Memory/
        MemoryModel.v   -- abstract memory model interface
        MemBytes.v      -- byte-level memory representation
        Sizeof.v        -- sizeof/alignof for types
    Theory/
      DenotationTheory.v -- correctness lemmas for the interpreter
```

### LLVM IR AST in Coq

`Syntax/LLVMAst.v` defines algebraic data types for every LLVM IR construct:

```coq
(* Types *)
Inductive typ : Type :=
  | TYPE_I           (sz : N)                (* i1, i8, i32, i64 *)
  | TYPE_Float
  | TYPE_Double
  | TYPE_Void
  | TYPE_Pointer     (t : typ)
  | TYPE_Array       (sz : N) (t : typ)
  | TYPE_Struct      (fields : list typ)
  | TYPE_Vector      (sz : N) (t : typ)
  | TYPE_Function    (ret : typ) (args : list typ) (vararg : bool)
  (* ... *)

(* Values / expressions *)
Inductive exp (T : Set) : Set :=
  | EXP_Ident      (id : ident)
  | EXP_Integer    (x : int64)
  | EXP_Float      (f : float32)
  | EXP_Double     (f : float)
  | EXP_Null
  | EXP_Undef
  | EXP_Poison
  | OP_IBinop      (iop : ibinop) (t : T) (v1 v2 : exp T)
  | OP_ICmp        (cmp : icmp)   (t : T) (v1 v2 : exp T)
  | OP_GetElementPtr (t : T) (ptrval : T * exp T) (idxs : list (T * exp T))
  (* ... *)

(* Instructions *)
Inductive instr (T : Set) : Set :=
  | INSTR_Op      (op : exp T)
  | INSTR_Load    (volatile : bool) (t : T) (ptr : T * exp T) (align : option int)
  | INSTR_Store   (volatile : bool) (val : T * exp T) (ptr : T * exp T) (align : option int)
  | INSTR_Alloca  (t : T) (nb : option (T * exp T)) (align : option int)
  | INSTR_Call    (fn : T * exp T) (args : list (T * exp T))
  (* ... *)
```

The `T` parameter allows the AST to be instantiated with either syntactic types (before type resolution) or dynamic types (after). This single-definition-multiple-use pattern is common in Coq mechanizations.

### The Interpreter

Vellvm's semantics is given by an interpreter in Coq. The top-level entry point (`Semantics/TopLevel.v`) takes a module (a list of function definitions, global variables, type declarations) and returns an interaction tree representing the program's computation:

```coq
Definition interpreter_top_level 
    (prog : list (toplevel_entity typ (block typ) * list (block typ)))
  : itree L0 uvalue :=
  ... (* detailed interpreter definition *)
```

The result type `itree L0 uvalue` is an interaction tree (§169.3) parameterized by an effect signature `L0` and returning a `uvalue` (a possibly-undefined value).

---

## 169.3 Interaction Trees

Interaction trees (ITrees), introduced by Xia et al. (POPL 2020), are the key technical device that allows Vellvm to handle the inherently effectful nature of LLVM IR — memory operations, nondeterminism, undefined behavior, and I/O — within Coq's purely functional framework.

### The ITree Type

```coq
CoInductive itree (E : Type → Type) (R : Type) : Type :=
  | Ret  (r : R)                              (* return a value *)
  | Tau  (t : itree E R)                      (* silent step *)
  | Vis  (A : Type) (e : E A) (k : A → itree E R)  (* observe effect e, continue *)
```

An `itree E R` is a (possibly infinite) tree where:
- `Ret r`: the computation terminates with result `r`.
- `Tau t`: a silent internal step (used to maintain productivity in corecursive definitions; allows delayed computation without changing the observable behavior).
- `Vis e k`: observe effect `e` of type `E A`, receive a response of type `A`, continue with `k`.

The `E` parameter is an effect signature — a type-indexed family describing the effects the computation can perform. Examples:

```coq
(* Memory effects *)
Inductive MemoryE : Type → Type :=
  | MemAlloca  : dtyp → MemoryE dvalue
  | MemLoad    : dtyp → dvalue → MemoryE uvalue
  | MemStore   : dvalue → dtyp → dvalue → MemoryE unit
  | MemFree    : dvalue → MemoryE unit

(* Undefined behavior *)
Inductive UBE : Type → Type :=
  | ThrowUB : string → UBE void

(* Nondeterminism (for undef) *)
Inductive PickE : Type → Type :=
  | Pick : forall (P : uvalue → Prop), PickE uvalue
```

### Interpretation

An ITree by itself is purely syntax — it describes a computation without running it. To give it meaning, we use **handlers**: functions from effects to ITrees that implement the effects:

```coq
Definition handle_memory_with_spec {E} `{FailureE -< E} `{UBE -< E}
    : MemoryE ~> stateT memory_stack (itree E) :=
  fun _ e =>
    match e with
    | MemAlloca t => ... (* allocate a new block *)
    | MemLoad t ptr => ... (* load from address, check validity *)
    | MemStore ptr t val => ... (* store to address, check validity *)
    | MemFree ptr => ... (* free block, check valid *)
    end.
```

The interpretation operator `interp h t` replaces every `Vis e k` in `t` with `h e >>= k`:

```coq
Definition interp {E F : Type → Type} (h : E ~> itree F) : itree E ~> itree F :=
  cofix interp_ t :=
    match observe t with
    | RetF r     => Ret r
    | TauF t'    => Tau (interp_ t')
    | VisF e k   => ITree.bind (h e) (fun x => interp_ (k x))
    end.
```

By composing different handlers, Vellvm can give different semantics to the same program:
- The **concrete interpreter** uses a specific memory model (block-based, byte-accurate).
- A **symbolic interpreter** leaves `PickE` effects unhandled (modeling nondeterminism as an unresolved choice).
- A **validated interpreter** uses a permissive memory model that tracks what accesses are valid without fully specifying the layout.

### Refinement via Bisimulation

Two ITrees are equivalent if they are **bisimilar** — they produce the same observable events in the same order:

```coq
Definition eutt {E R} (t1 t2 : itree E R) : Prop :=
  (* coinductive bisimulation, up to weak bisimulation (absorbs Tau steps) *)
```

`eutt` (short for "equivalence up to tau") is the standard notion of ITree equivalence. It is:
- **Reflexive**: every ITree is eutt to itself.
- **Symmetric**: the relation is bidirectional.
- **Transitive**: used for equational reasoning.
- **Congruent**: well-behaved with ITree operators (bind, map, etc.).

Optimization correctness in Vellvm is stated as: `eutt (interpret (transform P)) (interpret P)` — the optimized program's ITree is bisimilar to the original's. This is the precise formulation of "the transformation preserves behavior."

---

## 169.4 The Vellvm Memory Model

### Dynamic Values

Vellvm distinguishes two layers of values:

**`DVALUE` (Defined Value)**: a fully concrete value with no `undef` or `poison`.
```coq
Inductive dvalue : Type :=
  | DVALUE_Addr   (a : addr)           (* pointer: (block_id, offset) *)
  | DVALUE_I1     (x : int1)
  | DVALUE_I8     (x : int8)
  | DVALUE_I32    (x : int32)
  | DVALUE_I64    (x : int64)
  | DVALUE_Float  (f : float32)
  | DVALUE_Double (f : float)
  | DVALUE_Struct (fields : list dvalue)
  | DVALUE_Array  (elts : list dvalue)
  | DVALUE_Vector (elts : list dvalue)
  | DVALUE_None                         (* void *)
```

**`UVALUE` (Undetermined Value)**: may contain `undef`, `poison`, or expressions awaiting evaluation.
```coq
Inductive uvalue : Type :=
  | UVALUE_Addr   (a : addr)
  (* ... all DVALUE cases ... *)
  | UVALUE_Undef  (t : dtyp)            (* undef of type t *)
  | UVALUE_Poison (t : dtyp)            (* poison of type t *)
  | UVALUE_IBinop (iop : ibinop) (v1 v2 : uvalue)   (* delayed operation *)
  | UVALUE_ICmp   (cmp : icmp) (v1 v2 : uvalue)
  (* ... *)
```

`UVALUE_IBinop` represents a computation whose result is not yet fully determined because one or both operands contain `undef`. The "pick" interpretation of `undef` is deferred: when a `UVALUE` is ultimately used in a way that requires a concrete value (e.g., as a branch condition), the `PickE` effect is fired to nondeterministically choose a value.

This two-level value design cleanly separates:
- Fully concrete execution (only `DVALUE`): deterministic, fast, suitable for concrete test cases.
- Symbolic execution (with `UVALUE`): tracks undef/poison propagation through operations.

### Block-Based Memory

Vellvm's memory model (`Semantics/Memory/`) uses abstract memory blocks, similar to CompCert but tailored to LLVM IR's semantics:

```coq
(* A memory block: a contiguous range of allocated bytes *)
Record mem_block : Type := {
  mb_bytes    : IntMap.t SByte;    (* byte-level contents *)
  mb_alloc    : Z;                  (* allocation size *)
  mb_id       : block_id           (* abstract block identifier *)
}.

(* The full memory state: a map from block IDs to blocks *)
Definition memory_stack : Type := 
  list (IntMap.t mem_block * list block_id).   (* frames for alloca *)
```

The memory is a stack of frames (for handling `alloca` in function calls). Each frame contains a map from block IDs to blocks. When a function is called, a new frame is pushed; when it returns, the frame is popped and all `alloca`'d memory is freed.

**SByte** (symbolic byte): the byte-level representation that tracks provenance and `undef`:
```coq
Inductive SByte : Type :=
  | UByte  (uv : uvalue) (dt : dtyp) (idx : uvalue) (sid : store_id)
```
Each byte is tagged with the original `uvalue` that contained it, the type, the byte index within the type, and a store ID (to track aliasing of `undef` across different stores).

### `sizeof` and Alignment

`Semantics/Memory/Sizeof.v` provides the `sizeof` and `alignof` functions for LLVM types, following the ABI rules:

```coq
Fixpoint sizeof_dtyp (dt : dtyp) : N :=
  match dt with
  | DTYPE_I 1   => 1  (* i1: 1 byte *)
  | DTYPE_I 8   => 1
  | DTYPE_I 32  => 4
  | DTYPE_I 64  => 8
  | DTYPE_Float  => 4
  | DTYPE_Double => 8
  | DTYPE_Array sz t => sz * sizeof_dtyp t
  | DTYPE_Struct fields => fold_left (fun acc t => acc + sizeof_dtyp t) fields 0
  (* ... padding rules for alignment ... *)
  end.
```

The `GepM.v` module gives the semantics of `getelementptr`, computing byte offsets from type information and indices. This is one of the more complex parts of LLVM IR semantics: GEP with `inbounds` has different semantics from GEP without, and pointer arithmetic that steps outside the bounds of an allocation is undefined.

---

## 169.5 Verified Transformations in Vellvm

### Dead Code Elimination

Dead code elimination (DCE) removes instructions whose results are never used. In Vellvm, the DCE proof establishes:

```coq
Theorem dce_correct : forall p,
  eutt (⟦ dce p ⟧) (⟦ p ⟧)
```

where `⟦ · ⟧` denotes the ITree interpreter. The proof uses:

1. **Syntactic characterization of dead instructions**: an instruction `%x = op` is dead if `%x` does not appear in any later instruction or the return statement. This is computed by a liveness analysis over the CFG.

2. **Semantic lemma**: removing a dead instruction from a basic block does not change the ITree produced by executing the block. This is proved by induction on the block's instruction list: if `%x = op` is dead, then `exec(op)` never contributes to the output (all uses of `%x` are removed), so its execution is equivalent to `Tau` (a silent step) followed by the execution of the rest of the block.

3. **Global correctness**: extends the per-block lemma to whole programs by induction on the control-flow graph structure.

The main challenge is memory: an instruction like `%x = load i32, ptr %p` may be dead in terms of its result, but if `%p` is an invalid pointer, the load would fire the `UBE` effect (undefined behavior). The DCE proof must show that dead loads of valid pointers are safely removable, but cannot remove loads that could fire UB (since removing UB is not a safe transformation — it changes the set of possible behaviors).

This constraint is formalized: DCE only removes `%x = op` when `op` has no memory side effects (`op` is not a `load`, `store`, `call`, or `alloca`). For pure operations (arithmetic, bitwise operations, comparisons), the removal is always safe.

### Constant Propagation

Constant propagation replaces uses of SSA values that are known to be constant with the constant itself. The Vellvm proof:

```coq
Theorem const_prop_correct : forall p,
  eutt (⟦ const_prop p ⟧) (⟦ p ⟧)
```

The key invariant: if the dataflow analysis at a program point says `%x = c` (the value is constant `c`), then at runtime, the `uvalue` associated with `%x` is `UVALUE_I32 c` (or the appropriate type). The proof proceeds:

1. **Analysis soundness**: the constant propagation dataflow analysis overapproximates the set of possibly-constant values. If the analysis says "constant," the value is definitely that constant.

2. **Substitution lemma**: replacing a use of `%x` (where `%x = c`) with the literal `c` produces an eutt-equivalent ITree. This relies on the fact that `UVALUE_I32 c` and the literal `c` fire exactly the same effects.

3. **Interaction tree reasoning**: uses `eutt`'s congruence properties to conclude that substituting at every use site (via structural induction on the program) preserves the overall ITree.

### The Challenge of Memory Aliasing

Many optimizations in real LLVM deal with memory — load forwarding, store elimination, code motion across stores. These proofs are significantly harder in Vellvm because they require reasoning about pointer aliasing.

Consider load forwarding: given `store i32 %v, ptr %p` followed by `%r = load i32, ptr %q`, if the alias analysis determines that `%p` and `%q` must alias (point to the same address), the load can be replaced by `%r = %v`. The proof must establish:

1. The store and load access the same memory location (requires alias reasoning in the memory model).
2. No other store between them modifies the location.
3. The replacement does not change the program's ITree.

Obligation 1 requires reasoning about pointer arithmetic and block IDs — non-trivial in Vellvm's abstract memory model. Obligation 2 requires a reachability analysis in the CFG. Both are provable but require substantial proof infrastructure.

As of 2024, Vellvm's verified transformations are primarily non-memory ones (constant propagation, dead code elimination, instruction combining for pure operations). Memory-aliasing-dependent optimizations remain active research.

---

## 169.6 Other LLVM IR Formalization Efforts

### Crellvm (Seoul National University)

Crellvm (Kang et al., PLDI 2018) takes a different approach to LLVM IR verification. Rather than providing a complete semantics for all of LLVM IR, Crellvm focuses on verifying specific transformations by translating them to a CPS (continuation-passing style) IR and checking refinement via a custom logic. Crellvm verified several InstCombine patterns and part of the register allocator for a simplified LLVM backend.

Key differences from Vellvm:
- Crellvm uses a **validator** approach (like CompCert's regalloc): the transformation runs first; then a proof search checks that the transformation was valid.
- The logic is custom-designed for LLVM-style optimizations, rather than being a general-purpose proof assistant.
- Crellvm does not handle the full LLVM memory model.

### K-Framework Semantics for LLVM IR

The K framework (Rosu et al.) is a rewriting-based language definition framework that generates executable interpreters, type checkers, and program verifiers from a single semantic specification. A K semantics for LLVM IR was developed by Ellison and Rosu around 2012 and updated by subsequent work.

Key characteristics:
- **Executable**: the K semantics directly runs LLVM IR programs (unlike Coq-based semantics which require extraction or JIT).
- **Not machine-checked**: K's rewriting-based proofs are not as thoroughly audited as Coq proofs; the semantics may contain errors.
- **Partial**: does not handle all of LLVM IR (floating-point, intrinsics, and threading are incomplete).

The K LLVM semantics is useful as a **testing** reference: run a program in both LLVM and the K interpreter; differences reveal either bugs in LLVM or underdocumented behavior.

### LLVM Memory Model (Kang et al., PLDI 2015)

Kang, Hur, Mansky, Garbuzov, Zdancewic, and Vafeiadis (PLDI 2015) proposed a formal memory model for LLVM IR that resolves the `undef` vs `poison` vs UB ambiguity. Their model:

- Distinguishes **unspecified values** (undef-like: can be any bit pattern at each use) from **poison values** (single fixed-but-unknown value) from **undefined behavior** (the program has no defined behavior at all).
- Formalizes when an optimizer is allowed to replace a value with `undef` (when the original was unreachable), when it can propagate `poison` (when the original computed with overflow flags), and when UB is triggered (when `poison` is used in a way that requires a defined value).
- The model is not implemented in Coq but is expressed mathematically with sufficient precision to check specific examples.

This paper directly influenced the changes to LLVM IR in LLVM 10–13 (the `freeze` instruction, the migration from `undef` to `poison` for arithmetic flags).

### Alive2 as an Empirical Semantics Reference

Alive2 (Chapter 170) can be viewed as an operational formalization of LLVM IR semantics: its Z3 encoding defines precisely what each LLVM IR instruction means (in terms of bitvector arithmetic, memory model axioms, and poison/UB propagation). When Alive2 says "this transformation is incorrect," it is saying: "under this encoding of LLVM IR semantics, the transformation changes the observable behavior for some input."

This makes Alive2 a practical semantics reference even for people who never use it for formal verification: the source code of Alive2's Z3 encoding serves as a precise, executable specification of LLVM IR semantics that has been validated empirically against LLVM's actual behavior.

### Vellvm's Current Status (2024)

The Vellvm project has been actively developed since 2012. Current status:
- Full formalization of LLVM IR types, values, instructions, and basic blocks.
- Complete interpreter for sequential programs (no concurrency).
- Verified DCE and constant propagation.
- Active work on memory model refinements and more complex optimizations.
- Integration with the `llvm-torture` test suite: Vellvm's interpreter is run against thousands of test programs and compared to LLVM's output.

The gap between Vellvm and a "fully verified LLVM" is still substantial: LLVM's optimization pipeline contains hundreds of passes, each with complex interactions, and most are not yet verified in Vellvm. But Vellvm provides the foundational semantics that any future verification effort will build on.

---

## Chapter Summary

- **LLVM IR lacks a formal semantics**: the LangRef is English prose with ambiguities around UB, `undef`, `poison`, and pointer provenance — creating conditions for miscompilation bugs.
- **Vellvm** (Zhao, Zdancewic et al.) provides a Coq formalization of LLVM IR using algebraic data types for the AST (`LLVMAst.v`) and interaction trees for the semantics (`TopLevel.v`).
- **Interaction trees** (`ITree E R`) are coinductive structures that model effectful programs in Coq; effects include memory operations (`MemoryE`), undefined behavior (`UBE`), and nondeterminism (`PickE`). Handlers replace effects with concrete implementations.
- **`UVALUE`/`DVALUE`** separate fully-defined values from potentially-undefined ones; `undef` and `poison` are explicit constructors in `UVALUE`; their semantics is realized by a `PickE` handler.
- **Vellvm's memory model** uses abstract block-based memory with byte-level `SByte` representation that tracks provenance; `sizeof`/`alignof` follow ABI rules; GEP semantics handles pointer arithmetic precisely.
- **Verified transformations** include DCE and constant propagation; memory-dependent optimizations (load forwarding, alias-based elimination) remain harder and are active research.
- **Crellvm** uses a validator approach (similar to CompCert's regalloc); verified some InstCombine patterns; does not cover the full memory model.
- **LLVM Memory Model** (Kang et al., 2015) provided the formal basis for the `undef` → `poison` migration and the `freeze` instruction.
- **Alive2**'s Z3 encoding serves as an executable reference semantics for LLVM IR.

### References

- Zhao, J., Nagarakatte, S., Martin, M.M.K., Zdancewic, S. (2012). "Formalizing the LLVM Intermediate Representation for Verified Program Transformations." *POPL 2012*.
- Zakowski, Y., Beck, C., Yoon, I., Zaichuk, I., Zdancewic, S. (2021). "Modular, compositional, and executable formal semantics for LLVM IR." *ICFP 2021*.
- Xia, L.Y. et al. (2020). "Interaction Trees: Representing Recursive and Impure Programs in Coq." *POPL 2020*.
- Kang, J. et al. (2015). "Formal Verification of a realistic compiler: The CompCert experience." (Kang et al., separate from Leroy)
- Kang, J., Hur, C.K., Mansky, W., Garbuzov, D., Zdancewic, S., Vafeiadis, V. (2015). "A Formal C Memory Model Supporting Integer-Pointer Casts." *PLDI 2015*.
- Vellvm repository: [github.com/vellvm/vellvm](https://github.com/vellvm/vellvm)
