# Chapter 181 — Formal Verification in Practice

*Part XXVI — Ecosystem and Frontiers*

Part XXIV of this book examined what happens when formal methods are turned on the compiler itself: CompCert proves that its compilation passes preserve semantics ([Chapter 168](../part-24-verified-compilation/ch168-compcert.md)), Vellvm formalises the LLVM IR semantics in Coq ([Chapter 169](../part-24-verified-compilation/ch169-vellvm-and-formalizing-llvm-ir.md)), and Alive2 checks that each peephole rewrite LLVM performs is semantics-preserving under the undef/poison model ([Chapter 170](../part-24-verified-compilation/ch170-alive2-and-translation-validation.md)). This chapter addresses a complementary question: once LLVM has compiled your program, how do practitioners verify that the program itself is correct? The target of verification here is not the compiler but the software the compiler translates — the cryptographic libraries, operating system components, network protocols, and safety-critical applications that developers write. The tools range from push-button verifiers that discharge proof obligations to an SMT solver automatically, to interactive proof assistants in which the developer constructs a formal mathematical argument, to bounded model checkers that exhaustively explore all executions up to a given depth. Understanding where each fits, and which concrete tools have reached production maturity as of April 2026, is the practical knowledge this chapter conveys.

## 181.1 The Verification Landscape

Formal verification of software can be organised along two axes that determine the effort developers must invest and the class of properties that can be proved.

### 181.1.1 Two Axes: Automation and Scope

The first axis separates **push-button** tools from **interactive** tools. Push-button verifiers — Dafny, Verus, F*, CBMC, Kani — require the developer to annotate the program with specifications (preconditions, postconditions, loop invariants), but once annotations are in place the verification engine runs automatically. The developer does not write proofs; the underlying SMT solver or bounded model checker discharges the proof obligations mechanically. Interactive tools — Coq, Lean 4, Isabelle/HOL — require the developer to construct an explicit proof term or tactic script. The payoff is expressiveness: interactive provers can establish properties that no automated technique can handle, including theorems about infinite families of programs, semantic equivalences across language boundaries, and mathematical facts about algorithmic correctness. The cost is expertise and time.

The second axis separates **code verification** from **protocol or specification verification**. Code verifiers (Dafny, Verus, Kani, Flux) work directly on the source program and produce a verdict about that specific implementation. Protocol verifiers (TLA+, SPIN, nuXmv) work on an abstract state-machine model of a system and prove invariants about its reachable states, regardless of the implementation language.

Placing representative tools in the resulting four quadrants:

```
                  Push-button              Interactive
               ┌───────────────────────┬───────────────────────┐
Code           │ Dafny, Verus, Kani,   │ Coq, Lean 4,          │
               │ CBMC, Flux, F*        │ Isabelle/HOL          │
               ├───────────────────────┼───────────────────────┤
Protocol/Spec  │ TLC, Apalache, SPIN,  │ Isabelle/HOL (process │
               │ nuXmv, Alloy          │ algebra), mCRL2       │
               └───────────────────────┴───────────────────────┘
```

Most tools span part of the boundary — F* sits close to the push-button/interactive boundary because it automates many proof obligations via Z3 while still requiring the developer to write lemmas for non-trivial properties. Dafny is firmly push-button for verification but requires an interactive developer to write the specifications; it does not infer loop invariants automatically (though LLM assistance has substantially lowered this burden as of 2026). Lean 4 is firmly interactive but has an expanding library of powerful tactics (`decide`, `omega`, `norm_num`, `ring`, `simp`) that discharge entire classes of goals automatically, blurring the boundary.

The quadrant framing also maps to typical cost curves. A push-button tool typically requires 5–20% annotation overhead relative to unverified code for simple properties; interactive proof typically requires 3–10× the code size of the verified implementation (the Coq proof of the CompCert compiler is roughly 3× the size of the CompCert C source). These ratios have been improving as tactic automation improves, but the fundamental difference in effort remains: push-button tools commoditise a specific class of properties while interactive tools can prove anything that can be stated in the logic.

### 181.1.2 The Practical Threshold in April 2026

Two developments have moved push-button verification from an academic exercise into a plausible daily workflow. First, LLM-assisted Dafny specification generation has reached approximately 96% pass rate on standard vericoding benchmarks, meaning that for a wide class of algorithmic functions the developer supplies the code and an LLM generates a correct annotated specification that Dafny verifies. Second, AWS has published its deployment of Kani on s2n-tls, Firecracker, and tokio, demonstrating that bounded model checking of Rust production code is operational at scale.

A third less-visible shift is the maturation of refinement-type checkers embedded directly into existing language toolchains. Flux runs as a `rustc` plugin (Section 181.9); Liquid Haskell runs inside GHC; Stainless runs inside the Scala compiler. This embedding strategy reduces the entry cost for developers who work in those languages: rather than adopting a new language or tool, they add type annotations and gain incremental verification guarantees within their existing build.

### 181.1.3 Why LLVM Developers Should Care

The connection between this chapter and the preceding parts is not incidental. LLVM compiles the verified programs. The trusted base of any verification result must include the compiler; CompCert sidesteps the compiler from the trusted base entirely, but tools that verify C or Rust and extract to those languages trust that Clang or rustc preserves semantics. HACL* (Section 181.4) ships its verified C output directly into Firefox NSS and the Linux kernel WireGuard implementation, where it is compiled by Clang. The chain of trust from the F* proof to the executing binary includes Clang — which is why the Alive2 work of Chapter 170 and the translation-validation infrastructure of the LLVM community matters for program verification as much as for compiler correctness.

The same reasoning applies in reverse: every improvement to LLVM's correctness — Alive2 catching a misoptimisation, the sanitizer infrastructure (Chapter 110) catching a runtime bug — strengthens the argument that the compiled binary faithfully represents the verified source. Verification practitioners who understand the compiler's transformation semantics are better positioned to reason about the remaining gap between the high-assurance source proof and the binary that actually executes on hardware.

---

## 181.2 Dafny: Push-Button Code Verification

[Dafny](https://github.com/dafny-lang/dafny) is a language and verifier developed originally at Microsoft Research by K. Rustan M. Leino. It compiles to multiple targets (C#, Java, Go, Python, JavaScript) and embeds a specification sublanguage directly in the source syntax. The verification engine generates verification conditions (VCs) per procedure and discharges them automatically via Z3.

### 181.2.1 The Specification Sublanguage

Dafny exposes five principal specification clauses:

- `requires P` — precondition; the caller must establish `P` before invoking the method
- `ensures Q` — postcondition; the method guarantees `Q` holds on return
- `invariant I` — loop invariant; `I` must hold at each iteration entry and exit
- `decreases E` — termination measure; `E` must strictly decrease on each iteration
- `reads S`, `modifies S` — frame conditions; the method may only read/modify the objects in `S`

Ghost variables declared with `ghost var` exist only during verification; they are erased before compilation. Ghost methods are similarly erased. This allows the developer to introduce proof-state variables that track logical history without bloating the executable.

### 181.2.2 Binary Search with Loop Invariant

The classic demonstration of loop-invariant verification is binary search. The postcondition must capture both the success case (the element was found at the returned index) and the failure case (the element is absent from the array):

```dafny
method BinarySearch(a: array<int>, key: int) returns (index: int)
  requires a != null
  requires forall i, j :: 0 <= i < j < a.Length ==> a[i] <= a[j]
  ensures 0 <= index ==> index < a.Length && a[index] == key
  ensures index < 0 ==> forall i :: 0 <= i < a.Length ==> a[i] != key
{
  var lo := 0;
  var hi := a.Length;
  while lo < hi
    invariant 0 <= lo <= hi <= a.Length
    invariant forall i :: 0 <= i < lo ==> a[i] < key
    invariant forall i :: hi <= i < a.Length ==> a[i] > key
    decreases hi - lo
  {
    var mid := lo + (hi - lo) / 2;
    if a[mid] < key {
      lo := mid + 1;
    } else if a[mid] > key {
      hi := mid;
    } else {
      return mid;
    }
  }
  return -1;
}
```

The three `invariant` clauses jointly imply the postcondition at loop exit: the first bounds `lo` and `hi`; the second establishes that every element before `lo` is strictly less than `key`; the third establishes that every element from `hi` onward is strictly greater. The `decreases` clause proves termination. Z3 discharges all obligations without further developer input.

The verification condition generated by Dafny for the `ensures index < 0 ==> ...` clause is approximately: `∀ i :: 0 ≤ i < lo ==> a[i] < key AND ∀ i :: hi ≤ i < a.Length ==> a[i] > key AND lo == hi ==> ∀ i :: 0 ≤ i < a.Length ==> a[i] != key`. This is a purely quantified formula over integers and arrays that Z3's array and quantifier theories handle directly. The developer need not understand how the VC is generated or what Z3 does with it; the Dafny verifier reports either `verified` or an error message with a counterexample if Z3 finds a valuation that violates any clause.

The strength of this workflow is that the developer thinks in terms of *what* the program does (the specification) and trusts Dafny to check *that* the implementation matches. The weakness is that the specification itself can be wrong — it can be too weak (not capturing an important property) or vacuously true (trivially satisfied regardless of the implementation). Writing precise, meaningful specifications is the human-expertise component that no tool can fully automate, even with LLM assistance.

### 181.2.3 Generics and Trait-Style Specifications

Dafny supports generic types and trait declarations that carry specification obligations. A sorted-collection trait can assert that any conforming datatype maintains the sorted invariant under all mutating operations, making the invariant part of the interface contract rather than a per-implementation detail.

```dafny
trait OrderedCollection<T(==)> {
  ghost var Contents: seq<T>
  ghost var Repr: set<object>
  predicate Valid() reads Repr
  method Insert(x: T)
    requires Valid()
    modifies Repr
    ensures Valid()
    ensures multiset(Contents) == multiset(old(Contents)) + multiset{x}
}
```

### 181.2.4 Ghost Code and Heap Reasoning

The `reads` and `modifies` clauses are essential for heap-allocated structures. A method that reads from a linked list must declare the entire list spine in its `reads` clause; Dafny then ensures that any assertion about the list is stable across calls to other methods that do not modify those objects. Ghost fields, such as a `ghost var Contents: seq<T>` field on a stack class, let the developer specify the abstract value of a data structure independently of its concrete representation.

A stack class illustrates the pattern: the concrete state is an array with a length counter, but the specification tracks an abstract sequence. Every mutating method — `Push`, `Pop` — carries `ensures Contents == old(Contents) + [x]` and `ensures Contents == old(Contents)[..|old(Contents)|-1]` respectively. The `ghost var Contents` field is only used in specifications; it vanishes at compilation. The result is a verified abstract data type whose clients need only understand the sequence-based specification, not the array-based implementation.

### 181.2.5 Limitations and LLM Integration

Dafny does not directly support fine-grained concurrent verification; the companion language [Armada](https://github.com/microsoft/armada) extends Dafny with rely-guarantee and atomic section reasoning for concurrent programs. Z3 timeouts are the most common practical obstacle: deeply nested quantifiers and large generics can exceed the default 10-second solver budget, requiring manual splitting or triggering hints. There is no direct C output path; for extraction to verified C, F* (Section 181.4) is the appropriate choice.

The Dafny verification condition generator translates each procedure into a single, large SMT-LIB formula. A procedure with N specification clauses and M branching points generates a VC whose size is roughly O(N × M). For functions with complex generics or deeply recursive data structures, this formula can contain thousands of quantified axioms, at which point Z3's quantifier instantiation heuristics become the bottleneck. The `{:fuel 3}` attribute on recursive functions controls how many unfoldings Z3 attempts; `{:vcs_split_on_every_assert}` splits the VC into one sub-query per assertion, allowing Z3 to focus on each independently.

The [Dafny VS Code extension](https://marketplace.visualstudio.com/items?itemName=dafny-lang.ide-vscode) integrates with GitHub Copilot so that LLM suggestions are verified incrementally as they are typed. The ~96% vericoding pass rate reflects this workflow on the standard SV-COMP-derived and HumanEval-derived benchmarks: the LLM generates both implementation and specification; Z3 verifies; the feedback loop corrects the rare failure. The remaining 4% require either manual invariant strengthening (when Z3 cannot close a loop invariant without additional auxiliary predicates) or solver budget increases (when a Z3 timeout masks a valid proof).

---

## 181.3 Verus: Dafny-Style Verification for Rust

[Verus](https://github.com/verus-lang/verus) brings push-button program verification to production Rust. Rather than defining a new language, Verus is a Rust dialect implemented as a procedural macro that wraps verified code, plus a custom verification back-end built on Z3. Verified Rust code coexists with ordinary Rust in the same crate, interoperating through the ownership system.

### 181.3.1 The `verus!` Macro and Function Modes

Verus distinguishes three function modes:

- `exec` functions are compiled to machine code and may have specifications
- `spec` functions are pure mathematical functions used inside specifications; they are erased at compile time
- `proof` functions are ghost computations that construct proof terms; they too are erased

The `verus!` macro delimits code that the Verus checker processes:

```rust
use vstd::prelude::*;

verus! {

spec fn square(n: u64) -> u64 {
    n * n
}

fn isqrt(n: u64) -> (result: u64)
    ensures
        square(result) <= n,
        n < square(result + 1),
{
    if n == 0 {
        return 0;
    }
    let mut x = n;
    let mut y = (x + 1) / 2;
    while y < x
        invariant
            1 <= y,
            square(y) <= n || square(x) <= n,
        decreases x - y,
    {
        x = y;
        y = (x + n / x) / 2;
    }
    proof {
        assert(square(x) <= n) by {
            // Verus discharges this with nonlinear arithmetic lemmas from vstd
        }
    }
    x
}

} // verus!
```

The `ensures` clause states both that `result * result <= n` and that `n < (result + 1) * (result + 1)`, pinning the result to the unique integer square root. The `spec fn square` provides a readable alias that Verus uses to reason about multiplication without exposing nonlinear arithmetic to Z3 directly.

The loop invariant `square(y) <= n || square(x) <= n` is weaker than one might expect: it says that at least one of `x` or `y` is a candidate square root. This is sufficient because the loop monotonically reduces `x - y` (the `decreases` term), and at termination `y >= x` so both are equal. The postcondition then follows from the invariant, the termination condition, and the nonlinear arithmetic lemma `square(x) <= n`. The `proof { assert(...) by { ... } }` block is a hint to Verus to call vstd's nonlinear arithmetic module, which internally invokes Z3 with a nonlinear arithmetic hint strategy that Z3's default mode would time out on.

### 181.3.2 Ghost State: `Ghost<T>` and `Tracked<T>`

Verus introduces two type wrappers for ghost values. `Ghost<T>` is an erased value; it exists only during verification and carries no runtime cost. `Tracked<T>` is a linear ghost resource: it must be consumed exactly once, matching Rust's ownership semantics and enabling ghost reasoning about resources such as permission tokens, which are used to model heap ownership and concurrency invariants.

The separation between `Ghost<T>` and `Tracked<T>` mirrors the separation between ordinary ghost variables and linear ghost resources in separation logic. A `Ghost<bool>` might track whether a flag has been set; it can be copied freely. A `Tracked<WritePermission>` models the exclusive right to write to a memory location; it must be given up (consumed) before the write proceeds, and the write produces a `Tracked<ProvenWritten>` receipt. This resource-passing style allows Verus to reason about protocol invariants and ownership transfer without requiring the language runtime to track any of it — it is all erased before compilation.

```rust
verus! {

proof fn use_token(tracked tok: PermissionToken)
    requires tok.valid(),
    ensures  /* resource consumed */,
{
    tok.consume();   // consumes tok; no second use possible
}

} // verus!
```

This linearity is the key distinction from Dafny's ghost variables, which are unrestricted. `Tracked<T>` enables Verus to encode separation-logic-style heap ownership proofs within the Rust type system.

### 181.3.3 The `cargo verus` Workflow

The recommended `Cargo.toml` for a Verus project targets Rust Edition 2024:

```toml
[package]
name    = "my-verified-crate"
version = "0.1.0"
edition = "2024"

[dependencies]
vstd = { git = "https://github.com/verus-lang/verus", features = ["alloc"] }

[build-dependencies]
# verus provides a cargo subcommand; install with:
# cargo install cargo-verus
```

Running `cargo verus` invokes the Verus checker on all `verus!` blocks, then falls through to the standard rustc build. The `vstd` library supplies verified standard-library equivalents — vectors, maps, sequences — each carrying their own Verus specifications.

### 181.3.4 AWS Deployments

AWS uses Verus to verify components of two production systems. The [s2n-tls](https://github.com/aws/s2n-tls) TLS library has Verus proofs covering handshake state-machine invariants and memory-safety properties of its certificate handling code. [Firecracker](https://github.com/firecracker-microvm/firecracker), the microVM monitor underlying Lambda and Fargate, has Verus proofs on selected virtual device model components. Both deployments target safe Rust and rely on `vstd` for verified containers.

The s2n-tls deployment is particularly instructive because s2n-tls is a security-critical library where correctness matters beyond the ordinary standard. The Verus proofs provide formal certificates that specific state machine transitions in the TLS handshake are unreachable from valid initial states — for example, that the server cannot proceed to the key-exchange step before receiving the client's `ClientHello`, regardless of the interleaving of calls from the caller's code. This kind of state-machine invariant is exactly what TLA+ would prove at the protocol level, and Verus proves it at the implementation level, in the actual Rust code, closing the gap between the abstract protocol and the concrete implementation.

### 181.3.5 Current Maturity and Limitations

Verus supports most of safe Rust including generics, trait bounds, iterators via `vstd` wrappers, and closures in exec mode. Limitations as of April 2026: `async` functions cannot be verified (the lowering to state machines is not yet modelled); full trait object dispatch (`dyn Trait`) is handled only in restricted cases; proc-macro-generated code is opaque to the checker. Despite these boundaries, the supported fragment covers the bulk of systems-level Rust.

The Verus verification process interleaves with the standard Rust compilation pipeline. When `cargo verus` is invoked, it first invokes `rustc` with a special Verus back-end that intercepts the MIR before codegen, extracts the `verus!` blocks, generates the Z3 verification queries, and calls Z3. If Z3 finds a verification failure, Verus reports it with Rust-level source locations. Only after verification succeeds does the pipeline continue with standard codegen. This means Verus proofs have zero runtime cost: the verified properties are guaranteed at compile time, and the compiled binary contains no trace of the proof infrastructure. The resulting binary is identical to one produced by ordinary `cargo build`.

---

## 181.4 F* and HACL*: Verified Cryptography in Production

[F*](https://www.fstar-lang.org/) is a proof-oriented programming language developed jointly by INRIA and Microsoft Research. It combines a dependently-typed functional language with an SMT-backed automated prover, and includes an extraction pipeline to OCaml and to a low-level C-targeting subset called Low* (the basis of HACL*). The combination makes F* uniquely suited to verified cryptographic library development: the same source file carries both the mathematical proof and the implementation that extracts to production C.

### 181.4.1 The Effect System

F* programs are typed with *effects* that track computational properties:

- `Tot` — total function; terminates and has no side effects; the default for pure mathematical specifications
- `Lemma` — a `Tot` unit function whose sole purpose is to establish a proposition; the body is a proof script combining SMT and explicit logical reasoning
- `ST` — stateful computation over a heap; models mutable memory
- `ML` — ML-style effectful computation; the extraction target; no verification guarantees

The effect of a function determines what the SMT solver and type checker jointly verify. A `Tot` function that adds two natural numbers carries a machine-checkable proof that it terminates. A `Lemma` carries a proof of its postcondition. A `Low*` function (which elaborates to a combination of `ST` and region-typed heap) extracts to C via the [KaRaMeL](https://github.com/FStarLang/karamel) compiler.

### 181.4.2 A Cryptographic Lemma in F*

The following illustrates the F* style for a basic XOR self-inverse property, a foundational building block in stream cipher proofs:

```lean
open FStar.UInt8
open FStar.BV

let xor_self_inverse (x: uint_t 8) :
    Lemma (ensures (logxor x x = UInt8.zero)) =
  (* Z3 discharges this via bitvector arithmetic *)
  ()

let xor_commutative (x y: uint_t 8) :
    Lemma (ensures (logxor x y = logxor y x)) =
  ()

let stream_cipher_roundtrip (key: uint_t 8) (plaintext: uint_t 8) :
    Lemma (ensures (logxor (logxor plaintext key) key = plaintext)) =
  (* Proof by rewriting with xor_self_inverse and associativity *)
  assert (logxor (logxor plaintext key) key
          = logxor plaintext (logxor key key));
  xor_self_inverse key
```

Z3 discharges the bitvector obligations behind the `xor_self_inverse` and `xor_commutative` lemmas automatically. The `stream_cipher_roundtrip` lemma chains them. In HACL*, similar proofs cover ChaCha20 keystream generation, Poly1305 field arithmetic, and Curve25519 point multiplication.

The `stream_cipher_roundtrip` lemma is representative of how HACL* uses F* proofs: the specification function `stream_cipher_roundtrip` lives in a proof-only module that is never extracted to C. Its sole purpose is to establish, as a machine-checked fact, that the mathematical function implemented by the HACL* stream cipher is its own inverse when called twice with the same key. The Low* implementation of the stream cipher is a separate function in a separate module; the correspondence between the Low* implementation and the mathematical specification is proved by a third module that states and proves a `correctness` lemma connecting the two. This three-layer structure — specification, implementation, correspondence proof — is the standard HACL* methodology and is reused across all algorithms in the library.

### 181.4.3 HACL*: High-Assurance Cryptographic Library

[HACL*](https://github.com/hacl-star/hacl-star) proves three properties for every implemented algorithm:

1. **Memory safety** — no buffer overreads, no use-after-free, no null dereferences (established by Low*'s region-typed heap model)
2. **Functional correctness** — the C output computes the same function as the mathematical specification (proved in F* and preserved by KaRaMeL extraction)
3. **Absence of secret-dependent branches** — the execution trace does not depend on secret inputs; this constant-time property is verified via the `Taint` effect in F* ([Vale/EverCrypt framework](https://project-everest.github.io/))

HACL* algorithms are deployed in production:

| Algorithm | Deployment |
|---|---|
| ChaCha20-Poly1305 | Firefox NSS (via mozilla-central) |
| Curve25519 (X25519) | Firefox NSS TLS 1.3 key exchange |
| Poly1305 | Linux kernel WireGuard (`crypto/poly1305_generic.c` replaced by HACL* output) |
| BLAKE2b/BLAKE2s | libsodium, selected builds |
| Ed25519 | libsodium |

The trusted base for any deployed HACL* algorithm consists of: the F* type checker, Z3 (for SMT-discharged obligations), KaRaMeL (the C extraction tool), and the C compiler (typically Clang). Clang's correctness here is not formally verified unless the Alive2 peephole-validation infrastructure catches any transformation bug along the way — the intersection of Chapters 170 and 181.

The trusted-base analysis is worth dwelling on because it reveals a hierarchy of assurance. At the top sits the F* proof, which relies on F*'s own type-checker — a relatively small, auditable core. One level down, the SMT back-end: Z3 is not formally verified, so any Z3 bug that causes an incorrect `unsat` result would propagate undetected. The EverCrypt team mitigates this by requiring proof certificates from CVC5 on selected lemmas and cross-checking against Alt-Ergo, reducing reliance on any single solver. Below Z3 sits KaRaMeL: its extraction correctness is partially verified in F* itself (a bootstrap argument). At the bottom, Clang: the Alive2 infrastructure of [Chapter 170](../part-24-verified-compilation/ch170-alive2-and-translation-validation.md) catches many potential miscompilations, but it is not a full semantics-preservation proof. CompCert would eliminate this last gap, but CompCert does not target the performance-critical SIMD intrinsics that HACL* exploits on x86 and ARM.

The practical implication: HACL* provides meaningfully stronger assurance than an unverified cryptographic implementation, and the remaining trusted-base gaps are auditable and actively mitigated, making the overall system defensible in regulatory and security-critical contexts.

### 181.4.4 Why3 as a Meta-Platform

[Why3](https://www.why3.org/) is an alternative framework that compiles WhyML (a pure functional language with mutable state) to multiple back-end solvers simultaneously: Z3, CVC5, Alt-Ergo, and Coq. This multi-prover dispatch is useful when a single solver times out on a given obligation; Why3 tries each in sequence. When Z3 returns `unknown` on a nonlinear arithmetic obligation, Why3 automatically tries CVC5's Simplex-based arithmetic prover, then Alt-Ergo's native arithmetic module, before falling back to Coq as a last resort — at which point the developer must write an explicit proof term. CompCert's verification infrastructure uses a Why3 back-end for certain arithmetic lemmas, establishing a direct connection between the verified-compiler work of Chapter 168 and the broader SMT ecosystem.

Why3 also serves as a compilation target for other languages. The Frama-C static analysis platform (widely used in the avionics and automotive safety communities) can emit Why3 proof obligations from annotated C programs, leveraging the same multi-prover dispatch for C-level verification with the DO-178C and ISO 26262 standards in mind. [Frama-C](https://frama-c.com/)'s WP (Weakest Precondition) plugin translates ACSL (ANSI/ISO C Specification Language) annotations into WhyML goals and dispatches them to Why3; the combination is used in avionics certification projects where the DO-178C standard requires a formal methods supplement for the highest safety levels (DAL-A). The ACSL annotation language for C parallels Dafny's specification language: `//@ requires n >= 0; ensures \result >= 0;` annotations on C functions generate WP goals that Why3 then tries to discharge. Where solvers fail, the developer writes explicit Coq proofs, invoked through Why3's Coq back-end.

### 181.4.5 Jasmin: Assembly-Level Verification for Cryptography

[Jasmin](https://github.com/jasmin-lang/jasmin) (Almeida, Barthe, Besson, Blot, Grégoire, Laporte, Loew, Moreau, Strub, Tillich; CCS 2017, PLDI 2022) is a verification-oriented imperative language for high-assurance cryptographic implementations. Its distinguishing design choice — which differentiates it from F*/HACL* — is that Jasmin compiles directly to **x86-64 assembly** and simultaneously generates **EasyCrypt proof obligations** relating the source program to the assembly output. The verification chain terminates at the binary, not at a C intermediate.

#### The Jasmin Compilation Model

Jasmin programs carry pre/postconditions (`requires`, `ensures`) and loop invariants, similar to Dafny:

```jasmin
/* Constant-time conditional move: result = cond ? src : dst */
fn cmov(reg u64 dst, reg u64 src, reg bool cond) -> reg u64 {
  requires { true }
  ensures  { res = if cond then src else dst }
  reg u64 result;
  result = dst;
  if (cond) { result = src; }
  return result;
}
```

`jasminc` compiles this to x86-64 assembly and emits an EasyCrypt theory containing: (1) a Jasmin semantics of the source function, (2) a compiled semantics of the assembly, and (3) a preservation theorem relating them. Every compiler pass is verified in Coq/Rocq for correctness, making `jasminc` a verified compiler for the Jasmin language — analogous to CompCert for C ([Chapter 168](../part-24-verified-compilation/ch168-compcert.md)) but targeting the cryptographic domain and producing native assembly directly.

#### EasyCrypt Integration

[EasyCrypt](https://www.easycrypt.info/) is a framework for proving *cryptographic security* properties — indistinguishability, semantic security, IND-CCA — in a probabilistic relational Hoare logic (pRHL). The Jasmin/EasyCrypt connection enables a two-layer proof strategy:

1. **Functional correctness** — proved in Jasmin (Coq backend): the implementation correctly computes the mathematical function (e.g., ChaCha20 key stream matches the RFC specification).
2. **Cryptographic security** — proved in EasyCrypt: the specification-level algorithm is cryptographically secure (e.g., ChaCha20-Poly1305 achieves IND-CPA given ChaCha20's PRF security).

The combination yields an end-to-end proof: from cryptographic assumption to assembly binary.

#### libjade: Open Library of Verified Assembly Cryptography

[libjade](https://github.com/formosa-crypto/libjade) is an open-source library of Jasmin-verified cryptographic implementations, analogous to HACL* for the F* ecosystem. As of April 2026, libjade includes:

| Algorithm | Standard | AVX2 variant |
|-----------|----------|--------------|
| ML-KEM (Kyber) | NIST FIPS 203 | Yes |
| ML-DSA (Dilithium) | NIST FIPS 204 | Yes |
| Falcon | NIST FIPS 206 | No |
| BLAKE2b | RFC 7693 | Yes |
| SHA-3 / SHAKE | NIST FIPS 202 | Yes |
| X25519 | RFC 7748 | Yes |
| ChaCha20-Poly1305 | RFC 8439 | Yes |

The ML-KEM and ML-DSA implementations are particularly significant since NIST standardised these post-quantum algorithms in August 2024; formally verified implementations are required for high-assurance cryptographic libraries targeting Common Criteria EAL6+ or NIST FIPS 140-3.

#### Comparison with F*/HACL*

| Property | F*/HACL* | Jasmin/libjade |
|----------|----------|----------------|
| Source language | F* (functional, dependently typed) | Jasmin (imperative, C-like) |
| Compilation target | C (then Clang → assembly) | x86-64 assembly directly |
| Verification level | Source (F*) to C | Source to assembly |
| Trusted base | F* type system + Clang + LLVM | `jasminc` Coq proof + assembler |
| Security proofs | F*/Low* effects | EasyCrypt pRHL |
| Constant-time | `constant_time_select` F* effects | Language-level `ct` annotations |
| Production deployment | Firefox NSS, Linux WireGuard | libsodium (in progress), PQC libraries |
| Primary algorithm focus | ChaCha20-Poly1305, Curve25519, AES | ML-KEM, ML-DSA, Falcon, BLAKE2, SHA-3 |

The structural difference: HACL* verifies at the C level, so its trusted base includes Clang and the LLVM optimisation pipeline. Any misoptimisation detected by Alive2 ([Chapter 170](../part-24-verified-compilation/ch170-alive2-and-translation-validation.md)) in the compilation of HACL* C code would in principle invalidate the end-to-end security guarantee. Jasmin's verified-compiler approach eliminates this gap by targeting assembly directly and making the compilation itself a formal artefact verified in Coq.

---

## 181.5 Bounded Model Checking: CBMC and Kani

Bounded model checking (BMC) translates a program with loops bounded to depth `N` into a propositional formula and asks a SAT or SMT solver whether any execution of the bounded program violates the checked property. Unlike deductive verification, BMC requires no specifications beyond the property of interest and no loop invariants; the cost is that it cannot prove properties for all depths, only certify absence of bugs up to the chosen bound.

### 181.5.1 CBMC: Bounded Model Checking for C

[CBMC](https://github.com/diffblue/cbmc) (C Bounded Model Checker) translates C and C++ programs, with the loop bound controlled by `--unwind N`. It checks:

- Memory safety: out-of-bounds array access, null pointer dereferences
- Arithmetic: signed/unsigned integer overflow under C11 rules
- Assertion violations: `__CPROVER_assert(P, "message")`
- User-supplied safety annotations via `__CPROVER_assume(P)` and `__CPROVER_assert(P, ...)`

A typical invocation for a memory-safety audit:

```bash
cbmc program.c \
  --unwind 10 \
  --unwinding-assertions \
  --bounds-check \
  --pointer-check \
  --overflow-check \
  --signed-overflow-check \
  --trace
```

The `--unwinding-assertions` flag inserts an assertion at each back-edge that fires if the actual execution would require more unwindings than `--unwind N` permits, ensuring that a `VERIFICATION SUCCESSFUL` result is not vacuously true because the program never loops. `--trace` emits a counterexample witness when a violation is found, showing the sequence of variable assignments and branch decisions that lead to the failing state.

CBMC's Goto-CC front-end can compile entire C projects (via `goto-cc` as a drop-in replacement for `gcc`) producing a goto-program that CBMC then analyses, enabling whole-program BMC of codebases built with autotools or CMake.

Internally, CBMC translates the goto-program into *single static assignment* (SSA) form for the bounded path, producing a formula in which each variable assignment is a fresh propositional variable. The resulting SAT formula is then handed to a CDCL solver. For a loop bounded to depth `N`, the SSA formula has `O(N × L)` variables where `L` is the number of variables in the loop body. For `N = 10` and a moderate loop body, the resulting formula is typically in the tens of thousands of variables, well within the capacity of modern SAT solvers. For `N = 100`, it can reach millions of variables, at which point selective unwinding (`--unwind-set function:N`) is necessary to apply large bounds only to the target loop and small bounds elsewhere.

CBMC also supports *incremental* verification via the `--incremental-BMC` flag, which starts at depth 1 and increases the unwind bound until either a violation is found or the unwinding assertions pass, confirming the bounded program is bug-free. This avoids the need to guess an appropriate bound upfront.

### 181.5.2 Kani: Bounded Model Checking for Rust

[Kani](https://github.com/model-checking/kani) is a Rust verification tool developed by AWS that wraps CBMC. The developer writes a *proof harness* — an ordinary Rust function marked `#[kani::proof]` that exercises the code under test with symbolic inputs:

```rust
// Cargo.toml
// [package]
// edition = "2024"
//
// [dev-dependencies]
// kani = "0.55"

fn safe_add(a: u32, b: u32) -> Option<u32> {
    a.checked_add(b)
}

#[cfg(kani)]
mod verification {
    use super::*;

    #[kani::proof]
    fn safe_add_never_panics() {
        let a: u32 = kani::any();
        let b: u32 = kani::any();
        // kani::any() produces an unconstrained symbolic value;
        // CBMC will explore all 2^64 (a, b) pairs symbolically
        let result = safe_add(a, b);
        match result {
            Some(sum) => {
                // If addition succeeded, sum must equal a + b in the mathematical sense
                kani::assert(
                    (sum as u64) == (a as u64) + (b as u64),
                    "checked_add must return exact sum when it succeeds",
                );
            }
            None => {
                // If addition overflowed, a + b must exceed u32::MAX
                kani::assert(
                    (a as u64) + (b as u64) > (u32::MAX as u64),
                    "checked_add must return None only on overflow",
                );
            }
        }
    }

    #[kani::proof]
    #[kani::unwind(1)]
    fn safe_add_no_panic_on_max() {
        kani::assert(safe_add(u32::MAX, 1).is_none(), "MAX + 1 must overflow");
        kani::assert(safe_add(u32::MAX, 0).is_some(), "MAX + 0 must not overflow");
    }
}
```

The `Cargo.toml` specifies `edition = "2024"` and lists `kani` as a dev-dependency. Running `cargo kani` compiles the crate to MIR, translates MIR to CBMC's goto-program representation, and invokes CBMC on the resulting formula. The `#[cfg(kani)]` guard ensures the verification harnesses are excluded from normal builds.

The MIR-to-goto-program translation is Kani's core engineering contribution. Rust MIR (Mid-level IR) is an SSA-like representation at roughly the level of C — after monomorphisation, borrow-check, and lifetime erasure, but before LLVM IR generation. Translating from MIR rather than from LLVM IR preserves Rust-level type information (integer widths, struct layout, enum discriminants) that LLVM IR has already lowered away, enabling Kani to report counterexamples with Rust-level variable names and types. It also means Kani can reason about Rust's `Option<T>` and `Result<T,E>` types natively: `kani::any::<Option<u32>>()` generates a symbolic value that is either `None` or `Some(v)` for symbolic `v`, without the developer needing to encode the discriminant manually.

The goto-program representation uses a flat list of statements with explicit `GOTO` jumps, which CBMC converts to a single-path unrolled SAT formula. For the `safe_add` harness, the formula encodes: all 2³² × 2³² possible (a, b) pairs as free variables; the `checked_add` implementation (essentially: compute `(a as u64) + (b as u64)`, return `None` if it exceeds `u32::MAX`); and the two `kani::assert` conditions as required clauses. CBMC checks whether the negation of the conjunct of all assertions is satisfiable; `unsat` means all assertions hold for all inputs.

### 181.5.3 Kani at AWS

AWS applies Kani systematically across its Rust production codebase. Documented use cases include:

- **s2n-tls**: memory safety of the TLS record layer's Rust bindings
- **Firecracker**: virtio device model state transitions; verified that the MMIO handler never writes outside the guest memory region
- **tokio**: selected proofs on `Arc<T>` reference count manipulation and channel send/receive ordering

The contract-based variant `#[kani::proof_for_contract(function_name)]` generates a harness automatically from `#[requires(...)]` and `#[ensures(...)]` attributes placed on the function under test, reducing harness boilerplate for simple pre/postcondition checking.

### 181.5.4 CBMC vs. Kani

Both tools use CBMC as the underlying SAT/SMT engine. Their differences:

| Dimension | CBMC | Kani |
|---|---|---|
| Input language | C, C++ | Rust (via MIR) |
| Ownership modelling | Not applicable | Full Rust borrow checker honoured |
| Unwinding | `--unwind N` CLI flag | `#[kani::unwind(N)]` attribute |
| Counterexample | `--trace` output, C-level | Rust-level variable names |
| Integration | Makefile / goto-cc | `cargo kani` |
| Concurrency | Limited (POSIX threads) | Partial (not async) |

For C codebases, CBMC remains the correct choice. For Rust, Kani's Rust-native harness language, cargo integration, and MIR-accurate ownership model make it substantially more productive.

---

## 181.6 Temporal Logic and Model Checking

Deductive and bounded verification of source code leaves a gap: the correctness of a distributed protocol cannot be expressed as a property of a single function. A TLS handshake, a distributed consensus algorithm, or a two-phase commit protocol has correctness conditions — safety invariants, liveness guarantees — that depend on the interleaving of actions across multiple participants. Model checking addresses this gap by exhaustively or symbolically exploring all reachable states of an abstract state-machine specification.

### 181.6.1 TLA+ and the TLC Model Checker

[TLA+](https://lamport.azurewebsites.net/tla/tla.html) (Temporal Logic of Actions, Leslie Lamport) specifies a system as a state machine. A state is an assignment of values to variables; an action is a relation between current state and next state. The conjunction of the initial predicate `Init` and the transition relation `Next` defines all reachable behaviors.

A minimal two-phase commit specification illustrates the idiom:

```tla
----- MODULE TwoPhaseCommit -----
EXTENDS Naturals, FiniteSets

CONSTANTS RM   \* set of resource managers, e.g. {r1, r2, r3}

VARIABLES
  rmState,     \* rmState[r] \in {"working", "prepared", "committed", "aborted"}
  tmState,     \* tmState \in {"init", "committed", "aborted"}
  tmPrepared   \* set of RMs that have sent Prepare

TypeOK ==
  /\ rmState  \in [RM -> {"working", "prepared", "committed", "aborted"}]
  /\ tmState  \in {"init", "committed", "aborted"}
  /\ tmPrepared \subseteq RM

Init ==
  /\ rmState    = [r \in RM |-> "working"]
  /\ tmState    = "init"
  /\ tmPrepared = {}

RMPrepare(r) ==
  /\ rmState[r] = "working"
  /\ rmState'   = [rmState EXCEPT ![r] = "prepared"]
  /\ tmPrepared' = tmPrepared \cup {r}
  /\ UNCHANGED tmState

TMCommit ==
  /\ tmState     = "init"
  /\ tmPrepared  = RM
  /\ tmState'    = "committed"
  /\ rmState'    = [r \in RM |-> "committed"]
  /\ UNCHANGED tmPrepared

TMAbort ==
  /\ tmState  = "init"
  /\ tmState' = "aborted"
  /\ rmState' = [r \in RM |-> "aborted"]
  /\ UNCHANGED tmPrepared

Next == \E r \in RM : RMPrepare(r) \/ TMCommit \/ TMAbort

Spec == Init /\ [][Next]_<<rmState, tmState, tmPrepared>>

Consistent == \A r1, r2 \in RM :
  ~(rmState[r1] = "committed" /\ rmState[r2] = "aborted")

THEOREM Spec => []Consistent
=====
```

TLC, the explicit-state model checker for TLA+, can verify `Consistent` exhaustively for small `RM` sets (three resource managers yields a manageable state space). TLC explores the state graph in breadth-first order, storing visited states in an in-memory hash set. It scales to approximately 10⁸ states on a single machine with the default 4GB heap; beyond that, the `tlc2` tool supports distributed model checking across multiple machines using a work-stealing protocol over a shared filesystem.

The TLA+ toolbox produces counterexample traces in a human-readable format — a sequence of state assignments that violates the checked property. For the `Consistent` invariant on two-phase commit, a counterexample would show the exact sequence of `RMPrepare`, `TMCommit`, and `TMAbort` steps that leads one resource manager to commit while another aborts, which should be impossible given the spec above. Finding such a trace in a 15-line spec would indicate a logic error: perhaps the `TMAbort` action was written to allow abort even after some RMs have prepared without checking whether any have already received a commit decision, or equivalently that the `TMCommit` condition was not guarded by an absence of prior abort.

AWS published a [technical report](https://lamport.azurewebsites.net/tla/amazon.html) documenting TLA+ use on DynamoDB, S3, EBS, and internal services, finding non-trivial bugs in drafts that code review had missed. The DynamoDB replication protocol specification discovered a liveness violation — a scenario where a network partition combined with a specific crash recovery sequence could stall the replication pipeline indefinitely. The violation was visible in the TLC counterexample trace but had not been detected in months of integration testing because the triggering interleaving requires a precise timing of three independent failure modes.

### 181.6.2 Apalache: Symbolic TLA+

[Apalache](https://github.com/informalsystems/apalache) applies SMT-based symbolic model checking to TLA+ specifications, making it possible to verify properties over infinite domains where TLC's explicit enumeration is infeasible. Apalache translates TLA+ actions into SMT constraints and calls Z3 or CVC5, producing bounded-inductive proofs or counterexamples. For the two-phase commit specification above, Apalache can verify `Consistent` for all `RM` sets of size up to a given bound without enumerating states.

Apalache's approach to TLA+ typechecking is a notable engineering contribution: it introduces an optional type system for TLA+ (via `@type` annotations in comments) that allows Apalache to emit typed SMT constraints rather than untyped set-theoretic ones, substantially reducing the solver load on queries with set-valued variables. The annotation syntax is non-intrusive — TLC ignores the comments, so a type-annotated spec works with both tools. For new specifications, the Informal Systems team recommends writing type annotations from the start; for the growing library of existing TLA+ specifications (the TLA+ Examples repository on GitHub), Apalache provides a `--infer-poly` flag that infers types automatically for many common patterns.

### 181.6.3 SPIN and nuXmv

[SPIN](https://spinroot.com/) is a model checker for concurrent systems described in the Promela process language. Promela models asynchronous processes communicating via channels; SPIN verifies LTL (Linear Temporal Logic) properties by product-constructing the model with a Büchi automaton derived from the negated property. Partial-order reduction and bitstate hashing make SPIN scale to systems with 10⁸ reachable states in practice. The `-DSAFETY` compile flag activates a specialised safety-property checker (no liveness) that is significantly faster.

A key SPIN concept is the *never claim* — a Promela construct that encodes the negated LTL property as a Büchi automaton directly in the model file. When SPIN encounters a never claim, it product-constructs it with the system model and searches for accepting cycles in the product. Partial-order reduction prunes states where different interleavings of independent process steps would lead to the same observable behaviour, often reducing the explored state space by 10× or more for message-passing systems.

[nuXmv](https://nuxmv.fbk.eu/) extends its predecessor NuSMV with BDD-based, SAT-based, and SMT-based engines, and adds support for timed automata and infinite-precision arithmetic. It is used in hardware design verification and for hybrid systems where continuous dynamics interact with discrete state machines. nuXmv's ability to reason about integer and real-valued variables (via the SMT back-end) distinguishes it from purely Boolean model checkers; hybrid systems in automotive and avionics contexts, where a discrete controller interacts with a continuous physical plant, can be modelled and verified without the state-space explosion that would result from discretising the continuous variables.

### 181.6.4 SV-COMP 2026

The International Competition on Software Verification ([SV-COMP 2026](https://sv-comp.sosy-lab.org/2026/)) ran at TACAS 2026 in Turin, Italy. It is the primary benchmark for automated software verification tools operating on C programs. The competition categories include:

| Category | Scope |
|---|---|
| ReachSafety | Absence of assertion violations (`__VERIFIER_error()`) |
| MemorySafety | No invalid dereferences, no memory leaks |
| Concurrency | Race-freedom and atomicity violations in pthreads programs |
| Termination | All paths terminate |
| SoftwareSystems | Real-world Linux kernel device drivers and daemons |

2026 category highlights: CPAchecker (Java, Apache-2.0) won the ReachSafety overall category; Ultimate Automizer (Java, LGPL) won the MemorySafety category; Symbiotic (Python+LLVM instrumentation, MIT) won the MemorySafety subcategory for heap properties. All three tools are freely licensed. [CPAchecker](https://cpachecker.sosy-lab.org/) uses configurable program analysis (CPA), a framework that unifies predicate abstraction, BDD-based analysis, and CEGAR. [ESBMC](https://esbmc.org/) (C++, Apache) placed competitively across multiple categories using SMT-based BMC with bitvector and heap theories.

SV-COMP is relevant to the LLVM community for two reasons. First, several SV-COMP tools use LLVM as an intermediate representation: Symbiotic instruments LLVM bitcode with shadow memory and assertion checks before executing it symbolically; SMACK translates LLVM IR to Boogie IVL (Intermediate Verification Language) and invokes Corral or Z3. Second, the SV-COMP benchmark suite (the `sv-benchmarks` GitHub repository) contains thousands of annotated C programs that serve as regression tests for static analysis tools, including LLVM's own `clang -analyze` and the Clang Static Analyzer (Chapter 45). Practitioners building LLVM-based static analysis tools can benchmark their tools against the SV-COMP suite to compare precision and recall against the competition's state-of-the-art tools.

The CPAchecker CPA framework deserves a closer look because it represents the state of the art in reachability analysis for C. A CPA is defined by a domain lattice, a transfer function (one per statement type), and a merge/stop operator. The strength of the framework is composability: the `CompositeCPA` product-combines multiple CPAs — a value analysis CPA, a shape-analysis CPA for heap, a predicate CPA — into a joint analysis whose precision is the conjunction of the individual analyses. When a counterexample is found by the coarser value analysis, it is cross-checked by predicate abstraction via CEGAR (Counterexample-Guided Abstraction Refinement); if the counterexample is spurious, the abstraction is refined and the analysis restarts. This loop converges for the large majority of Linux driver benchmarks in the SV-COMP suite within a few minutes.

---

## 181.7 SMT and SAT Engines

All push-button verification tools described in this chapter ultimately reduce to calls on SMT or SAT solvers. Understanding the solvers' strengths and failure modes is prerequisite knowledge for diagnosing verification timeouts and for choosing the right tool for a given proof obligation.

### 181.7.1 Z3

[Z3](https://github.com/Z3Prover/z3) (MIT License, Microsoft Research) is the dominant SMT solver in the verification toolchain. It supports all major SMT-LIB theories: linear and nonlinear integer/real arithmetic, bitvectors (QF_BV), arrays (QF_AX), uninterpreted functions, strings, sequences, and quantifiers with pattern-based instantiation.

Z3's tactic system composes specialized solvers. A tactic is a strategy that transforms the current goal; the `then` combinator sequences them:

```smt2
; Verify that (a + b) mod 2 = (a mod 2 + b mod 2) mod 2 for 32-bit integers
(declare-fun a () (_ BitVec 32))
(declare-fun b () (_ BitVec 32))

; Using 32-bit bitvector arithmetic throughout
(define-fun lhs () (_ BitVec 32)
  (bvurem (bvadd a b) #x00000002))
(define-fun rhs () (_ BitVec 32)
  (bvurem (bvadd (bvurem a #x00000002) (bvurem b #x00000002)) #x00000002))

(assert (not (= lhs rhs)))
(check-sat)
; Expected: unsat   (the two expressions are always equal)
```

Z3 returns `unsat` in milliseconds for this query. The Python API provides the same capability programmatically:

```python
from z3 import BitVec, BitVecVal, UDiv, URem, Solver, Not, ForAll, unsat

a = BitVec('a', 32)
b = BitVec('b', 32)

two = BitVecVal(2, 32)

lhs = URem(a + b, two)
rhs = URem(URem(a, two) + URem(b, two), two)

s = Solver()
s.add(Not(lhs == rhs))          # assert the negation of the property
result = s.check()

assert result == unsat, f"Unexpected: {result}"
print("Property verified: (a + b) % 2 == (a % 2 + b % 2) % 2 for all 32-bit a, b")
```

Z3 struggles with nonlinear integer arithmetic (products of variables), quantifier alternation (∀∃ queries), and deeply recursive definitions. For these cases, Dafny users add `{:nonlinear}` lemmas with hints; Verus users call `vstd::arithmetic::nonlinear` lemmas from the standard library; F* users fall back to interactive Coq proofs for obligations that Z3 cannot close automatically.

Z3's architecture is modular: it consists of a core SAT/DPLL(T) engine, theory solvers for each SMT-LIB theory, and a tactic language for combining them. The tactic `(using-params smt :max-conflicts 100000)` increases the conflict budget before timeout; `(check-sat-using (then simplify bit-blast sat))` forces full bit-blasting for QF_BV queries when Z3's native bit-vector solver is too slow. For LLVM-related verification work, the Z3 Python API is the most ergonomic interface; the SMT-LIB2 text format (`.smt2` files) is preferred for tool integration and for sharing reproducible benchmarks.

### 181.7.2 CVC5

[CVC5](https://cvc5.github.io/) (BSD License, Stanford/Iowa/NYU) is Z3's primary peer for SMT-LIB compliance. Its strengths relative to Z3 lie in string and sequence reasoning (the `QF_S` and `QF_SLIA` fragments), arithmetic reasoning via the Simplex algorithm with extended Farkas lemmas, and proof production: the `--produce-proofs` flag emits a machine-checkable proof certificate in the Alethe proof format, enabling independent certification of `unsat` results. CVC5 is used by Dafny as a secondary solver (alongside Z3) when Z3 times out on a particular obligation, and by Why3 as one of its back-end solvers.

The Alethe proof format produced by `--produce-proofs` is human-readable and can be checked independently by the [Isabelle Alethe checker](https://github.com/seL4/isabelle) and by `alechecker`, a standalone verifier. This makes CVC5 appropriate for high-assurance contexts where the solver result itself must be independently audited — for example, in regulatory submissions that require a tool qualification argument under DO-178C.

### 181.7.3 Bitwuzla

[Bitwuzla](https://bitwuzla.github.io/) (MIT License) is specialised for the bitvector and floating-point theories: QF_BV, QF_FP, and combinations with arrays and uninterpreted functions. On benchmarks involving bit-manipulation algorithms, cryptographic primitives, and hardware-software interface code, Bitwuzla outperforms both Z3 and CVC5. It is built on the CaDiCaL SAT solver and uses MCSAT (Model Constructing Satisfiability) for floating-point reasoning. Tools that verify bitwise invariants — e.g., that a bit-twiddling hack for integer square root is equivalent to the naive implementation — benefit from routing those queries to Bitwuzla rather than Z3.

Bitwuzla's command-line interface accepts SMT-LIB2 format and can be used as a drop-in replacement for Z3 on QF_BV benchmarks:

```bash
# Verify a bitvector property using Bitwuzla
bitwuzla --produce-models property.smt2
# Returns: sat / unsat / unknown
# With --produce-models: if sat, prints a model showing a counterexample
```

For the HACL* use case, Bitwuzla is particularly relevant because Poly1305 field arithmetic over GF(2¹³⁰−5) involves large bitvector multiplications that benefit from Bitwuzla's specialised field-arithmetic preprocessing.

### 181.7.4 Kissat

[Kissat](https://github.com/arminbiere/kissat) is a CDCL (Conflict-Driven Clause Learning) SAT solver written in C by Armin Biere. It won the SAT Competition in 2020, 2021, and 2022. SAT solvers are appropriate when the verification problem has been reduced to pure propositional logic: circuit equivalence checking, bounded hardware model checking, and certain combinatorial invariants. SMT solvers that use bitvector theories internally call SAT solvers as their back-ends; Bitwuzla uses CaDiCaL (Biere's earlier solver), while many others use Kissat or similar. CBMC, the bounded model checker underlying Kani, translates its SAT queries to DIMACS format and can use any external SAT solver; configuring Kissat as the back-end (`cbmc --sat-solver kissat`) often improves performance on benchmark programs with dense propositional structure.

The developer-facing rule of thumb: use Z3 via the tool's default interface for most verification tasks; reach for Bitwuzla when the properties involve bit-level or floating-point arithmetic; use CVC5 when Z3 times out on string or arithmetic obligations, or when proof certificates are required; use Kissat directly only when writing a custom SAT-based analysis pipeline or when configuring CBMC's solver back-end.

---

## 181.8 Neural Network Verification

Safety-critical deployments of machine learning models require formal guarantees that the model's output is bounded over a region of inputs. This is fundamentally a verification problem: given a network `f` and an input ball `‖x − x₀‖∞ ≤ ε`, prove that `f(x) ∈ S` for all `x` in the ball. The EU AI Act (Annex III, high-risk systems) mandates robustness guarantees for certain deployed classifiers; formal NN verification is the technical underpinning of such guarantees.

The connection to LLVM and MLIR is direct. Neural network inference is increasingly compiled through MLIR dialects (Chapter 139 — Linalg, Chapter 141 — Vector) or through XLA/OpenXLA (Part XXII) before being lowered to LLVM IR and emitted as machine code. Formal verification at the network level (proving that the mathematical function `f` satisfies a robustness property) says nothing about the compiled binary; a bug in the MLIR lowering pipeline could violate the property at runtime even if the network itself is provably robust. This is the NN-verification analogue of the HACL* trusted-base problem: formal proof at one level of abstraction does not automatically propagate through all compilation steps to the executing binary. The MLIR community is beginning to address this via translation-validation approaches modelled on Alive2 — checking that each MLIR lowering step preserves tensor semantics — but as of April 2026 this remains an active research direction rather than a deployed tool.

### 181.8.1 α,β-CROWN

[α,β-CROWN](https://github.com/Verified-Intelligence/alpha-beta-CROWN) (α,β-CROWN, BSD License) has won the VNN-COMP neural network verification competition every year from 2021 through 2024. Its approach combines two techniques:

**Linear relaxation**: each ReLU activation `max(0, x)` is over-approximated by a linear constraint `lx ≤ relu(x) ≤ ux`. The tightness of the linear bound is controlled by learnable slopes `α` (per-neuron, optimised by gradient descent). This yields an outer bound on the output set that can be certified without case analysis.

**Branch and bound**: when the linear relaxation bound is insufficiently tight to certify the property, α,β-CROWN splits on the sign of an unstable neuron (a neuron whose activation is unknown), creating two subproblems with tighter bounds. The `β` parameters control the incorporation of split constraints into the relaxation, enabling GPU-parallel bound tightening across the branch tree.

The result is a GPU-accelerated verifier capable of certifying robustness properties for ResNet-50 and VGG-16 scale models in minutes.

The [auto_LiRPA](https://github.com/Verified-Intelligence/auto_LiRPA) Python library provides the underlying bound propagation infrastructure:

```python
import torch
import torch.nn as nn
from auto_LiRPA import BoundedModule, BoundedTensor, PerturbationLpNorm

# Define a small two-layer network
class TwoLayerNet(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(4, 8)
        self.fc2 = nn.Linear(8, 2)

    def forward(self, x):
        return self.fc2(torch.relu(self.fc1(x)))

model = TwoLayerNet()
model.eval()

# Nominal input
x0 = torch.randn(1, 4)
epsilon = 0.05    # L∞ perturbation radius

# Wrap the model for bound computation
bounded_model = BoundedModule(model, x0)

# Define the perturbation set: all inputs within ε of x0 in L∞ norm
ptb = PerturbationLpNorm(norm=float('inf'), eps=epsilon)
x_bounded = BoundedTensor(x0, ptb)

# Compute lower and upper bounds on all output neurons
# method="CROWN-optimized" uses α-CROWN with gradient-optimised slopes
lb, ub = bounded_model.compute_bounds(x=(x_bounded,), method="CROWN-optimized")

print(f"Output lower bounds: {lb}")   # guaranteed lower bound on both logits
print(f"Output upper bounds: {ub}")   # guaranteed upper bound on both logits

# If lb[0, 0] > ub[0, 1] for all inputs in the ball, class 0 always wins:
if lb[0, 0].item() > ub[0, 1].item():
    print(f"Certified: class 0 is robust within L∞ ball of radius {epsilon}")
```

### 181.8.2 Marabou

[Marabou](https://github.com/NeuralNetworkVerification/Marabou) (NYU/Hebrew University, BSD License) uses the Simplex algorithm extended with case splits on ReLU neurons to solve NN verification queries exactly. Unlike α,β-CROWN's relaxation-based over-approximation, Marabou never loses precision: a `SAT` result from Marabou means the property is genuinely violated, not that the relaxation was too loose. This exactness makes Marabou better suited to small networks and equality properties (e.g., "the output of network A exactly equals the output of network B on these inputs") where a relaxation-based tool would return `UNKNOWN`.

Marabou uses an SMT-like query language:

```python
from maraboupy import Marabou
import numpy as np

# Load an ONNX network
network = Marabou.read_onnx("small_classifier.onnx")

# Set input bounds: all inputs in [0, 1]
for i in range(network.numInputVars):
    network.setLowerBound(network.inputVars[0][i], 0.0)
    network.setUpperBound(network.inputVars[0][i], 1.0)

# Assert that the first output neuron is always >= 0.5 (property to verify)
# addConstraint: output[0] >= 0.5 is encoded as -output[0] <= -0.5
output_var = network.outputVars[0][0]
network.setLowerBound(output_var, 0.5)

options = Marabou.createOptions(verbosity=0, timeoutInSeconds=120)
exit_code, vals, stats = network.solve(options=options)

if exit_code == "unsat":
    print("Property verified: output[0] >= 0.5 for all inputs in [0,1]^n")
elif exit_code == "sat":
    print("Counterexample found:", vals)
```

The trade-off between α,β-CROWN and Marabou is the standard approximation/exactness trade-off: α,β-CROWN scales to large networks and certifies most real-world robustness properties within minutes, but may return `UNKNOWN` on tight cases; Marabou is exact but becomes impractical for networks with thousands of ReLU neurons.

A third dimension of NN verification, beyond robustness and output bounding, is **equivalence checking**: proving that two networks (e.g., a full-precision model and its quantised version) produce identical classifications on a shared input domain. This problem maps to a bilinear constraint system; tools like [nneq](https://github.com/NeuralNetworkEquivalence/nneq) encode it as a QF_BV problem and invoke Bitwuzla, exploiting the fact that quantised networks operate over integer arithmetic. This connection between NN verification and SMT closes the loop with Section 181.7: the same Bitwuzla solver used for cryptographic proofs is applicable to quantised neural network equivalence.

The regulatory landscape is also worth noting. The EU AI Act (Article 10 and Annex III, effective August 2026) classifies certain ML systems as high-risk and requires providers to implement risk management systems that include accuracy and robustness testing. Formal NN verification provides a stronger guarantee than empirical robustness testing: a certification from α,β-CROWN that a medical image classifier's decision is unchanged for all L∞ perturbations of radius ε is a mathematically provable statement, not a statistical observation from a finite test set. Expect the intersection of formal verification and AI regulation to drive significant tooling investment in the 2026–2030 period.

---

## 181.9 Refinement Types for Rust: Flux

[Flux](https://flux-rs.github.io/flux/) is a refinement-type checker for Rust implemented as a `rustc` plugin. Rather than wrapping verified code in a macro (as Verus does) or requiring a separate language (as Dafny does), Flux annotates Rust function signatures with refinement predicates — logical formulas over value variables — and checks that the body satisfies the refined type statically.

### 181.9.1 The Specification Language

Flux annotations attach to function signatures via the `#[flux::sig(...)]` attribute:

```rust
#![feature(register_tool)]
#![register_tool(flux)]

// Cargo.toml:
// [package]
// edition = "2024"
//
// [dependencies]
// flux-rs = { git = "https://github.com/flux-rs/flux" }

#[flux::sig(fn(a: i32, b: i32{v: v != 0}) -> i32)]
fn divide(a: i32, b: i32) -> i32 {
    a / b
    // Flux checks that division by zero is impossible because b carries
    // the refinement predicate {v: v != 0}, ruling out b == 0 at the type level.
}

#[flux::sig(fn(x: i32{v: v > 0}) -> i32{v: v > 0})]
fn increment_positive(x: i32) -> i32 {
    x + 1
    // Flux must verify that x + 1 > 0 given x > 0.
    // It generates the VC: forall x. x > 0 => x + 1 > 0, which the
    // Fixpoint Horn-clause solver discharges automatically.
}

#[flux::sig(fn(n: i32{v: v >= 0}) -> i32{v: v >= 0})]
fn factorial(n: i32) -> i32 {
    if n == 0 { 1 } else { n * factorial(n - 1) }
}
```

The `i32{v: v != 0}` notation is a *base refinement type*: the base type is `i32` and the predicate `v != 0` must hold for any value inhabiting this type. The variable `v` is a logical variable ranging over values of the base type.

### 181.9.2 The `RefinedBy` Attribute

Refinement predicates can be attached to struct fields:

```rust
#[flux::refined_by(len: int)]
pub struct Buffer {
    #[flux::field(Vec<u8>[len])]
    data: Vec<u8>,
    #[flux::field(usize[len])]
    capacity: usize,
}

#[flux::sig(fn(buf: &Buffer[@n]) -> usize[n])]
fn buffer_len(buf: &Buffer) -> usize {
    buf.data.len()
}
```

The `@n` binder in `&Buffer[@n]` names the refinement index of the buffer — its logical length — within the function's scope, enabling reasoning that relates the return value to the structural invariant.

### 181.9.3 The Verification Architecture

Flux generates *Horn clause* verification conditions. A Horn clause is a logical formula of the form `P₁ ∧ P₂ ∧ … ∧ Pₙ ⇒ Q` where the `Pᵢ` and `Q` are atoms. Horn clauses are strictly easier to solve than general SMT; the [Fixpoint](https://github.com/ucsd-progsys/liquid-fixpoint) solver (used by LiquidHaskell and Flux) applies a greatest-fixpoint iteration on the Horn constraint system, converging rapidly for the predicates typical of systems code. For quantifier-free linear arithmetic and bitvectors, Fixpoint discharges the constraints without calling Z3 at all.

The Fixpoint solver uses the *liquid types* inference algorithm from Rondon, Kawaguchi, and Jhala (PLDI 2008): it infers refinement types for intermediate expressions by solving a system of subtyping constraints. The constraints have the form `[Γ ⊢ P]` meaning "under context Γ, the predicate P must hold"; these are reduced to Horn clauses and solved by greatest-fixpoint iteration over the predicate lattice. Convergence is guaranteed for the supported predicate domains (linear arithmetic, simple bitvector predicates) because these form finite-height lattices. The key advantage over full SMT verification is decidability and speed: Horn clause solving in a quantifier-free linear arithmetic domain runs in polynomial time, making Flux dramatically faster than Verus or Dafny for the class of properties it can express.

Flux integrates with rustc as a plugin via the `register_tool` mechanism. When rustc encounters `#[flux::sig(...)]`, the plugin intercepts the type-checking phase, generates Horn clause constraints from the annotated function, calls Fixpoint, and reports any subtyping violations as rustc diagnostics with source locations. The integration is transparent to the rest of the build: non-Flux code is compiled without overhead, and Flux-annotated code gets checked incrementally as part of the normal `cargo build` or `cargo check` invocation.

### 181.9.4 Comparison with Verus

| Dimension | Flux | Verus |
|---|---|---|
| Annotation style | Attribute macros on signatures | `verus!` macro blocks |
| Ghost code | No ghost functions | Full ghost/proof mode |
| Proof language | Implicit (Horn clause solving) | Explicit proof terms |
| Expressiveness | Refinement predicates, base types | Full first-order logic |
| Async support | No | No |
| Verification strength | Lighter (no heap reasoning) | Stronger (Tracked resources) |
| Maturity | Research prototype | AWS production use |

Flux is appropriate for catching value-range bugs (division by zero, array index out of range, negative-length buffers) in existing Rust code with minimal annotation overhead. Verus is appropriate when stronger correctness properties — full functional correctness, protocol invariants, resource ownership — must be proved.

The intellectual lineage of Flux runs through [LiquidHaskell](https://ucsd-progsys.github.io/liquidhaskell/) (the same UCSD group) and, further back, to Xi and Pfenning's Dependent ML (1999), which first showed that a restricted dependent type system could be implemented efficiently in a mainstream language via index refinements. Liquid Haskell applies the same idea to Haskell, where the pure functional type system makes refinement inference particularly tractable. Flux adapts the approach to Rust, where the borrow checker's ownership tracking provides a natural foundation for reasoning about aliasing — a notoriously difficult problem for refinement-type checkers on imperative languages.

### 181.9.5 Current Limitations

Flux does not support `async` functions, closures with captures, or full trait object dispatch. The refinement language is limited to quantifier-free linear arithmetic and bitvectors; recursive data structures require explicit loop or recursion invariants expressed as indexed types. Trait implementations can be annotated with Flux signatures, but the checker does not yet verify that all implementations satisfy the trait's refinement contract uniformly — the developer must annotate each implementation separately. As of April 2026, Flux is a research prototype from the UCSD Programming Systems group; it is not recommended for production deployment without an active research collaboration.

---

## 181.10 AI-Assisted Proof

The verification landscape in April 2026 is defined as much by what large language models can do as by what the solvers can do. The combination of LLMs with formal proof checkers has produced a qualitative shift in the accessibility of interactive verification.

### 181.10.1 The 2025–2026 Shift

Three developments converged: (1) LLMs pretrained on code were fine-tuned or RLHF-trained specifically on formal proof corpora — Lean 4's [Mathlib4](https://github.com/leanprover-community/mathlib4), Isabelle's Archive of Formal Proofs, Coq's [opam ecosystem](https://coq.inria.fr/opam/www/) — giving models a working understanding of tactic syntax and proof strategy; (2) Lean 4 tooling matured to provide a fast kernel that can provide token-level feedback during LLM generation, enabling reinforcement-learning-from-proof-checker-feedback loops; (3) the open-weight model ecosystem produced specialised models that outperform general-purpose LLMs on formal proof benchmarks.

The feedback loop enabled by (2) is qualitatively different from the feedback loop available for code generation. When a code LLM generates a function that fails tests, the error is a runtime value or a test failure message — an English-language signal that the model can interpret only indirectly. When a proof LLM generates a tactic that the Lean kernel rejects, the error is a structured proof-state mismatch: the current goal is `⊢ x + y = y + x` and the tactic `ring` either closes it or does not. The kernel's judgment is binary, unambiguous, and available in microseconds. This clean reward signal is why RL training on proof checkers has been so effective: the training signal has far lower variance than test-based code generation rewards.

### 181.10.2 Lean Copilot and LeanDojo

[Lean Copilot](https://github.com/lean-dojo/LeanCopilot) integrates LLM suggestions into Lean 4's interactive tactic mode. From within the VS Code Lean 4 extension, the developer can invoke `suggest_tactics` to get ranked tactic suggestions for the current proof state, or `search_proof` to attempt fully automated proof search. Lean Copilot queries a locally-served or remote-API language model and filters suggestions through the Lean 4 elaborator, discarding invalid tactics before presenting candidates. The filtering step is crucial: raw LLM output contains many syntactically plausible but type-incorrect tactics; running each candidate through the elaborator on the current proof state, and discarding those that fail, transforms a noisy suggestion stream into a ranked list of valid moves.

The workflow from the developer's perspective: open a Lean 4 file in VS Code, write a theorem statement, place the cursor inside the proof block where the cursor goal is displayed in the Lean infoview, and invoke `suggest_tactics`. Lean Copilot presents three to five ranked candidate tactics; the developer clicks one to insert it, sees the new goal in the infoview, and repeats. For simple goals (ring identities, simp-closable propositions, single-step rewrites), `search_proof` often finds a complete proof immediately.

[LeanDojo](https://github.com/lean-dojo/LeanDojo) provides the training infrastructure: it extracts training examples — (proof state, next tactic) pairs — from Mathlib4, and the [ReProver](https://github.com/lean-dojo/ReProver) model trained on this data achieves competitive performance on the miniF2F benchmark with retrieval-augmented tactic generation. The retrieval step indexes Mathlib4 theorems by their statement; when the model encounters a goal that resembles a known library theorem, retrieval brings the relevant lemmas into the generation context, substantially improving the probability of a correct tactic. This is the key architectural difference between ReProver and models that generate tactics purely from the local context.

### 181.10.3 Open-Weight Proof Models

Three open-weight models stand out as of April 2026:

**DeepSeek-Prover-V2** (DeepSeek-AI, MIT License): trained via reinforcement learning with the Lean 4 kernel as the reward signal. On the miniF2F benchmark (a set of 488 competition-mathematics problems formalised in Lean 4 and Isabelle) and the ProofNet benchmark (undergraduate mathematics), DeepSeek-Prover-V2 achieves state-of-the-art results among open-weight models. The RL training loop generates candidate proof scripts, runs them through the Lean 4 kernel, and assigns reward 1 for a closed proof and 0 for a failure, training on the resulting signal with PPO. The key insight is that the Lean 4 kernel provides a perfectly reliable and zero-latency reward signal for formal proof.

**Kimina-Prover** (Alibaba DAMO Academy): specialised for competition mathematics in Lean 4. It uses a hierarchical generation strategy: first generate a proof sketch (named lemmas, key steps), then fill in each lemma's proof independently. This reduces the generation depth required per proof, improving success rates on multi-step competition problems.

**Goedel-Prover** (open-source): targets both Lean 4 and Isabelle, with a focus on theorem discovery — generating conjectures that are worth proving rather than only proving given conjectures. It is trained on a corpus that mixes Lean 4 and Isabelle proofs, enabling transfer across the two systems.

### 181.10.4 The 2026 Vericoding Benchmarks

The vericoding benchmarks measure the end-to-end success rate of LLM-assisted program verification: given a natural-language problem description, generate both the implementation and the formal specification, and verify that the generated annotated program passes the verifier. Results as of early 2026:

| Tool | LLM pass rate | Primary bottleneck |
|---|---|---|
| Dafny | ~96% | Z3 timeout on complex specs |
| Verus | ~44% | Rust type system interaction |
| Lean 4 | ~27% | Deep mathematical reasoning |

The dramatic spread reflects the difficulty gradient. In Dafny, Z3 discharges most proof obligations automatically once the LLM has generated a plausible loop invariant and postcondition; the LLM's job is specification generation, not proof construction. The LLM therefore operates in a familiar regime — writing structured annotations that look like pre/postconditions in many languages — and the solver handles the mathematical detail. In Verus, the Rust type system interacts with the `verus!` macro in ways that require precise handling of lifetimes and trait bounds; current LLMs generate type-incorrect Verus more often than Dafny. Ghost-state reasoning in particular is a failure mode: models frequently generate `Tracked<T>` values that are consumed multiple times or `Ghost<T>` values that leak into exec-mode contexts, producing elaboration errors that are difficult to recover from without domain knowledge. In Lean 4, the proofs require not just syntactically correct Lean but mathematically valid arguments; without the right lemma imports and tactic choices, the Lean kernel rejects the proof.

The progression Dafny → Verus → Lean 4 also tracks the verification strength of each tool: Dafny checks lightweight properties (loop invariants, postconditions), Verus checks functional correctness with ownership reasoning, and Lean 4 can establish arbitrarily deep mathematical theorems. The 96%/44%/27% spread therefore reflects both LLM capability and the inherent difficulty of what each tool asks the developer (and by extension, the LLM) to specify.

### 181.10.5 Practical Recommendations

For practitioners choosing a verification tool in April 2026:

**For verified cryptography or security-critical C libraries**: F*/HACL* is the most mature path. The EverCrypt library demonstrates that production cryptographic code — ChaCha20, Poly1305, Curve25519 — can be specified, proved, and extracted to C that ships in Firefox and the Linux kernel. The learning curve is steep but the trusted base is smallest.

**For verified systems Rust**: Verus is production-ready for the verified portions of AWS's infrastructure. It covers safe Rust with full functional correctness; the cargo integration fits existing build workflows. Start with `ensures` postconditions on leaf functions before adding ghost state.

**For protocol correctness**: TLA+/TLC is the pragmatic choice for distributed protocol design; Apalache for protocols with large or infinite domains. Both tools have been deployed at AWS scale.

**For bounded bug-finding in C**: CBMC with `--bounds-check --overflow-check --pointer-check` is the fastest path to catching memory safety bugs without formal specifications.

**For bounded bug-finding in Rust**: `cargo kani` with symbolic `kani::any()` inputs; add `#[requires]`/`#[ensures]` contracts to grow into contract-based verification.

**For LLM-assisted verification today**: Dafny with the VS Code extension and Copilot integration offers the highest LLM success rate. Write the implementation, let the LLM generate the specification, review and adjust. The 96% success rate means one in twenty specifications needs manual correction.

**For interactive theorem proving**: Lean 4 with the Mathlib4 library is the most active community as of 2026. The LeanCopilot and LeanDojo tools lower the entry barrier substantially. For developers coming from a functional programming background (Haskell, OCaml), Lean 4's syntax and tactic mode will feel more natural than Coq's; for those already invested in Coq (e.g., the seL4 community), Isabelle2026 and Coq remain fully supported and well-maintained.

A note on maturity signals: the best indicator that a verification tool has reached production maturity is not academic publications but auditor acceptance. HACL* has been accepted as evidence in Mozilla security reviews. The AWS use of TLA+ and Kani has been cited in SOSP and OSDI papers documenting production deployment. seL4 (Isabelle) is certified under Common Criteria EAL7+ and used in aerospace and automotive systems. When choosing a tool for a compliance-relevant context, look for these external validation signals, not just benchmark numbers.

The underlying theme across all tools in this chapter is the *specification gap*: the difficulty of expressing what a program should do precisely enough that a mechanical checker can verify it. Push-button tools minimise the proof obligation but still require the developer to write `requires`, `ensures`, `invariant`, and `decreases` clauses that completely and correctly capture the intended behaviour. Interactive provers eliminate the specification gap by demanding total precision, at the cost of expert effort. Model checkers for protocols side-step code-level specifications entirely, working at the level of abstract state machines. Each approach makes a different trade-off, and the practitioner who understands all four quadrants is equipped to select the right tool for each situation — boundary conditions in a safety-critical function (Kani, Dafny), cryptographic algorithm correctness (HACL*), distributed protocol invariants (TLA+), and machine-learning model robustness (α,β-CROWN).

For the LLVM practitioner specifically, the most immediately actionable entry point is Kani: it requires no new language, integrates with cargo, and applies directly to the Rust systems code that the LLVM Rust frontend compiles. Adding `#[kani::proof]` harnesses to a Rust library that wraps an LLVM-based tool — say, a safe wrapper around `llvm-c` bindings — is a concrete, bounded verification task achievable in a day of work that immediately improves assurance of that interface.

---

## Chapter Summary

- Formal verification of programs divides into four quadrants along two axes: (push-button vs. interactive) × (code verification vs. protocol/specification verification). Each quadrant has mature tooling as of April 2026.

- **Dafny** is the most accessible push-button code verifier: `requires`/`ensures`/`invariant`/`decreases` annotations; Z3 backend; LLM-assisted specification generation reaches ~96% pass rate on vericoding benchmarks.

- **Verus** brings Dafny-style verification to production Rust (Edition 2024). `Tracked<T>` enables linear ghost state integrated with Rust's ownership model; `cargo verus` and `vstd` provide a complete workflow. AWS uses it on s2n-tls and Firecracker.

- **F*/HACL*** proves memory safety, functional correctness, and constant-time execution for cryptographic algorithms. ChaCha20-Poly1305 and Curve25519 verified in F* run in Firefox NSS; Poly1305 runs in Linux kernel WireGuard. The trusted base includes Clang, linking F* proofs to LLVM correctness.

- **Jasmin** (§181.4.5) is a verification-oriented imperative language that compiles directly to x86-64 assembly, simultaneously emitting EasyCrypt proof obligations relating source to assembly. `jasminc` is a Coq-verified compiler. The **libjade** library provides Jasmin-verified implementations of ML-KEM, ML-DSA, Falcon, BLAKE2b, SHA-3, X25519, and ChaCha20-Poly1305, targeting post-quantum cryptography libraries and FIPS 140-3 deployments. Compared to HACL*, Jasmin eliminates LLVM/Clang from the trusted base by targeting assembly directly; the two tools are complementary in coverage.

- **CBMC** performs bounded model checking of C programs via SAT/SMT; `--unwind N` bounds loops; `--bounds-check --pointer-check --overflow-check` catches common memory and arithmetic errors. **Kani** wraps CBMC for Rust with `#[kani::proof]` harnesses, `kani::any::<T>()` symbolic inputs, and `cargo kani` integration.

- **TLA+/TLC** and **Apalache** verify distributed protocol invariants via explicit-state and SMT-symbolic model checking respectively. AWS's documented use on DynamoDB, S3, and EBS establishes TLA+ as the industry standard for distributed protocol design. SPIN handles concurrent process models in Promela.

- **Z3** (MIT) is the primary SMT solver backend; **CVC5** (BSD) excels at string/arithmetic theories and proof production; **Bitwuzla** (MIT) outperforms both for bitvector and floating-point verification. **Kissat** is the leading open-source CDCL SAT solver.

- **α,β-CROWN** is the four-year VNN-COMP champion for neural network robustness verification, using optimised linear relaxation with branch-and-bound on a GPU. **Marabou** provides exact (non-approximate) NN verification for smaller networks.

- **Flux** is a refinement-type checker implemented as a rustc plugin, annotating function signatures with `#[flux::sig(...)]` predicates and generating Horn clause VCs discharged by Fixpoint. It is lighter-weight than Verus but less expressive and currently a research prototype.

- LLM-assisted proof is transforming the field: **Lean Copilot** integrates model suggestions into Lean 4 tactic mode; **DeepSeek-Prover-V2** (MIT, RL-trained on Lean 4 kernel feedback) achieves SOTA on miniF2F and ProofNet; **Kimina-Prover** and **Goedel-Prover** target competition mathematics and theorem discovery. The open-weight ecosystem for formal proof has matured from research novelty to practical infrastructure.

- The chain of trust for deployed verified software runs from the formal proof through extraction or compilation to the binary — and Clang/LLVM sits in that chain. The Alive2 translation-validation work of [Chapter 170](../part-24-verified-compilation/ch170-alive2-and-translation-validation.md) and the CompCert architecture of [Chapter 168](../part-24-verified-compilation/ch168-compcert.md) are therefore directly relevant to program verification practitioners, not only to compiler engineers.

---

@copyright jreuben11
