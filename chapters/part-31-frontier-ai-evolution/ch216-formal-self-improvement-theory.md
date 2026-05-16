# Chapter 216 — Formal Self-Improvement Theory

*Part XXXI — Frontier AI Evolution*

Every practical self-improvement mechanism surveyed in Chapters 212–215 rests on an implicit promise: after the modification, the system is better. That promise is surprisingly hard to formalise. Better at what? Better by whose measure? Better enough to justify the cost? These questions are not merely philosophical friction — they determine whether a self-modifying system converges, diverges, or silently regresses. The Gödel Machine, AIXI, and their surrounding framework of algorithmic information theory are attempts to turn that promise into a mathematical guarantee. This chapter develops those theoretical foundations, traces what they can and cannot deliver in practice, and connects them to the verified compilation tradition of Part XXIV — which offers the most rigorous partial answers currently available about what "certified improvement" can mean for a computable artifact. The goal is not a research survey but a precise exposition of the theory that gives the practical methods of this part their rigorous interpretation.

The chapter is structured as a descent from the theoretically ideal to the practically achievable. Section 216.1 establishes the mathematical substrate: Kolmogorov complexity, the universal semimeasure, and the fundamental limits imposed by incompleteness and undecidability. Sections 216.2–216.4 develop the three canonical theoretical constructs — the Gödel Machine, AIXI, and Levin search / Solomonoff induction — with precise formal definitions and explicit statements of what each achieves and what obstacles it cannot overcome. Sections 216.5–216.6 bridge theory to practice: the Darwin Gödel Machine as the best computable approximation to the Gödel Machine's ideal, and the verified compilation methods of Part XXIV as the best formal certification tools currently available for program-level self-rewrites. Sections 216.7–216.8 confront the specification problem — the deepest obstacle — and situate the full theoretical framework in relation to the practical methods surveyed throughout Part XXXI.

Cross-references: [Chapter 207 — Reflective Code, Open Problems, and Build Roadmap](../part-30-AI-first-PL-design/ch207-reflective-code-open-problems-roadmap.md) · [Chapter 212 — Weights as a Programming Substrate](ch212-weights-as-programming-substrate.md) · [Chapter 213 — Mechanistic Interpretability Infrastructure](ch213-mechanistic-interpretability.md) · [Chapter 214 — Gradient-Based Self-Modification](ch214-gradient-based-self-modification.md) · [Chapter 215 — Evolutionary Architecture Search](ch215-evolutionary-architecture-search.md) · [Chapter 217 — Self-Reflective Inference](ch217-self-reflective-inference.md) · [Chapter 218 — Self-Improvement Fitness Functions](ch218-self-improvement-fitness-functions.md) · [Chapter 168 — CompCert](../part-24-verified-compilation/ch168-compcert.md) · [Chapter 169 — Vellvm and Formalizing LLVM IR](../part-24-verified-compilation/ch169-vellvm-and-formalizing-llvm-ir.md) · [Chapter 170 — Alive2 and Translation Validation](../part-24-verified-compilation/ch170-alive2-and-translation-validation.md)

---

## 216.1 Theoretical Foundations

### Kolmogorov Complexity

The formal theory of self-improvement requires a substrate for measuring the intrinsic information content of programs, environments, and policies. Kolmogorov complexity provides that substrate.

Fix a universal Turing machine U. The Kolmogorov complexity of a binary string x is:

```
K(x) = min { |p| : U(p) = x }
```

where |p| is the length of program p in bits and the minimum ranges over all programs that cause U to halt and output x. K(x) is the length of the shortest self-delimiting description of x relative to U. It formalises minimum description length: the most compact lossless encoding achievable in principle.

Three properties govern how Kolmogorov complexity interacts with self-improvement theory. First, the **invariance theorem**: for any two universal Turing machines U and V, |K_U(x) − K_V(x)| ≤ c_{UV} where the constant c_{UV} depends only on the two machines, not on x. Complexity is machine-independent up to a fixed additive constant — a fact that permits the subscript U to be suppressed when asymptotics dominate. Second, K is **upper-semicomputable**: there exists an algorithm that, given x, enumerates programs in length order and halts when it finds a short one, but no algorithm can compute K(x) exactly (that would solve the halting problem). Third, **string incompressibility**: for any length n, most strings x of length n satisfy K(x) ≥ n − O(1) — most strings have no compact description, which is the formal statement that random data cannot be compressed.

The **universal probability** m(x) associated with algorithmic information theory is:

```
m(x) = 2^{-K(x)}
```

This assigns high probability to strings with short descriptions and low probability to strings that are algorithmically complex. It is not a normalised probability distribution in the classical sense, but a semimeasure: Σ_x m(x) ≤ 1, with the inequality arising because some programs diverge. The semimeasure m dominates every computable probability measure ν in the sense that m(x) ≥ c_ν · ν(x) for a constant c_ν depending only on ν. This is Solomonoff's key theorem: m is a universal prior that is, up to a multiplicative constant, at least as good as any computable probabilistic predictor [Solomonoff 1964, *Information and Control* 7(1) and 7(2)].

### Self-Reference and the Halting Problem

The formal obstacles to self-improvement are concentrated in two closely related results: Gödel's incompleteness theorems (1931) and the halting problem (Turing 1936/Church 1936). Both concern the limits of self-referential formal systems, and both constrain what a self-improving agent can prove about itself.

**Gödel's first incompleteness theorem** (stated informally for the context): any consistent formal system F that is strong enough to encode arithmetic contains a sentence G_F that is true but not provable in F. The sentence G_F says, in effect, "I am not provable in F". If F could prove G_F, F would prove something false (its own consistency implies G_F is true, so if F proves G_F it would also prove its own consistency — ruling out provability by the second theorem for any sufficiently strong system). The operational consequence for self-improvement: a self-modifying agent whose goal is to prove that its rewrites are beneficial cannot, in general, prove this within any fixed formal system that it inhabits. Extending the proof system is always possible, but correctness of the extension is itself unprovable in the old system.

**The halting problem** is directly encoded in any program search process: to determine whether program p improves on baseline, one must in general evaluate p to completion — but whether p halts is undecidable. Proof search over programs is therefore not just computationally expensive but fundamentally undecidable as a decision procedure. Any computable self-improvement mechanism is necessarily incomplete: it will fail to recognise some genuinely beneficial rewrites.

These are not engineering inconveniences to be addressed by better hardware. They are structural limits that force any formal theory of self-improvement to choose between completeness (finding all beneficial rewrites) and decidability (being able to determine in finite time whether a rewrite is beneficial). No computable system achieves both.

### Algorithmic Information Theory Basics

The further concepts needed for the Gödel Machine and AIXI formulations are the universal distribution M (capital letter, the normalised Solomonoff mixture) and prefix-free Kolmogorov complexity K. A **prefix-free** Turing machine is one where no valid program is a prefix of another — the set of valid programs forms a prefix-free code and the Kraft inequality holds with equality: Σ_p 2^{-|p|} = 1. The complexity K is defined relative to such machines.

The **algorithmic probability** of a string x is:

```
M(x) = Σ_{p : U(p) starts with x} 2^{-|p|}
```

This sums over all programs whose output begins with x, weighting each by 2^{-|p|}. Solomonoff induction uses M as an online sequence predictor: given observations x_1, …, x_n, the predicted probability of the next symbol is M(x_{1:n}1) / M(x_{1:n}). This predictor converges to the true measure of any computable environment in the sense that the total expected squared prediction error is bounded by (ln 2/2) K(ρ), where ρ is the true environment's generating measure [Solomonoff 1978, *IEEE Trans. Inf. Theory* 24(4)]. It is optimal among all computable predictors, dominated by no other.

---

## 216.2 The Gödel Machine

The Gödel Machine, introduced by Schmidhuber in 2003 and formalised in the 2007 AGI compendium chapter, is the conceptual apex of formal self-improvement theory: a self-referential universal problem solver that provably only rewrite itself when a proof exists that the rewrite yields expected future reward at least as high as doing nothing [J. Schmidhuber, "Gödel Machines: Self-Referential Universal Problem Solvers Making Provably Optimal Self-Improvements", arXiv cs/0309048, 2003; "Gödel Machines: Fully Self-Referential Optimal Universal Self-Improvers", in Goertzel & Pennachin (eds.) *Artificial General Intelligence*, Springer 2007].

### Formal Definition

A Gödel Machine is a tuple (S, A, P, V, π, PS, F) where:

- **S** is the full system state, including the current policy code, the proof searcher code, the utility function, all memory, and the history of interactions with the environment.
- **A** is the action space, partitioned into environment actions a_e (affecting the external world) and self-modification actions a_r (replacing executable code within S).
- **P** is a probabilistic model of the environment and all future states, specified as part of the axioms of F.
- **V : S × ℕ → ℝ** is the utility function, representing expected cumulative reward from time t onwards. The utility function must be formally specified — not learned — and is part of the axioms of F.
- **π** is the current policy, a mapping from states to actions, encoded as executable code within S.
- **PS** is the proof searcher, a procedure that enumerates proofs within F in order of proof length.
- **F** is the formal axiomatic system: it encodes the environment model P, the utility function V, and the transition rules of the universal Turing machine that executes the agent. All proofs about self-rewrites are evaluated within F.

### The Self-Rewrite Condition

At each time step t, the proof searcher PS searches for a proof within F of a theorem of the form:

```
V(t, apply(r)) > V(t, stop)
```

where `apply(r)` denotes the state resulting from executing rewrite r on the current machine code, and `stop` denotes the state resulting from leaving the code unchanged and continuing with the current policy π. Both sides of the inequality are evaluated within the formal system F over the distribution of all future environmental interactions.

When the proof searcher finds such a proof for some rewrite r, the machine immediately applies r to itself — replacing its own policy, proof searcher, or any other component. The rewrite is not merely a local patch to a subroutine; F permits replacing the entire inference engine if a proof can be found that doing so raises expected utility.

The **globally optimal self-rewriter** property follows directly: conditioned on F being a sound proof system, every executed rewrite is provably beneficial. The machine never executes a rewrite that a formal proof shows to be harmful. This is the strongest form of self-improvement guarantee achievable — "improvement" here means improvement in the formal utility function's expectation, not in any informal or empirical sense.

### Pseudocode of the Gödel Machine Inner Loop

```pseudocode
state S    := (code_π, code_PS, code_V, memory, history)
formal_sys F := axioms over S, environment model, utility function spec

loop forever:
    proof := PS.search(F, goal="∃r. V(t, apply(r)) > V(t, stop)")
    if proof found for rewrite r:
        self.apply(r)           -- rewrite own code; S updated atomically
        S := post_rewrite_state
    else:
        a := π(S)               -- current policy generates action
        execute(a)
        observe(environment_response)
        update(S, history)
        t := t + 1
```

The critical subtlety: the proof goal quantifies over the *post-rewrite* state of the machine, including any subsequent rewrites the post-rewrite machine might itself perform. This is not a greedy one-step lookahead but a proof about the entire future trajectory under the rewritten policy. The proof must hold in F, which encodes the probabilistic environment model and the expected utility computation.

### Guaranteed Improvement and Its Caveats

The improvement guarantee is conditional on three assumptions, each of which fails in practice in a different way.

**Assumption 1: Soundness of F.** The formal system F must be sound — it must not prove false theorems. If F is inconsistent (e.g., the utility function spec contains a contradiction), any theorem is provable, including `V(t, apply(r)) > V(t, stop)` for a catastrophically harmful r. Soundness of F cannot be established from within F itself: Gödel's second incompleteness theorem proves that any consistent formal system strong enough to encode arithmetic cannot prove its own consistency.

**Assumption 2: Computability of the proof search.** Proof search is undecidable in general. The machine may search forever without finding a proof for a rewrite that is in fact beneficial — the proof simply exceeds whatever resource bound the proof searcher can allocate. Schmidhuber addresses this with a time-bounded variant that interleaves proof search with policy execution, but this removes the completeness guarantee.

**Assumption 3: Formalisation of the utility function.** The utility function V must be written down formally. For any real-world agent with complex, partially observable objectives — "be helpful", "avoid harm", "preserve capabilities" — formalising V completely is itself an open research problem. The specification problem (§216.7) is not peripheral to the Gödel Machine; it is prerequisite to its operation.

Despite these caveats, the Gödel Machine establishes the correct theoretical reference point: self-improvement should be guided by a formal criterion, executed only when a proof of benefit exists, and the proof should range over the entire future trajectory rather than a one-step heuristic. Every practical approximation (DGM §216.5, gradient-based modification Ch214, evolutionary search Ch215) sacrifices one or more of these properties and should be understood as a trade-off against this ideal.

---

## 216.3 AIXI: Bayesian-Optimal Universal Agent

AIXI, introduced by Hutter in 2000 and comprehensively developed in [M. Hutter, *Universal Artificial Intelligence: Sequential Decisions Based on Algorithmic Probability*, Springer 2005], is the Bayesian-optimal agent for any computable environment, with no domain knowledge assumed beyond a prior over all computable environment models.

### The Universal Semimeasure Prior

Fix a class M of all computable probability distributions ρ on environment interaction sequences. Each ρ assigns probability ρ(e_{1:t} | a_{1:t}) to seeing the sequence of perceptions e_{1:t} given that the agent took actions a_{1:t}. The Solomonoff–Hutter prior ξ is the mixture:

```
ξ(e_{1:t} | a_{1:t}) = Σ_{ρ ∈ M} 2^{-K(ρ)} · ρ(e_{1:t} | a_{1:t})
```

Each environment model ρ is weighted by 2^{-K(ρ)}: simpler environments (shorter program descriptions) receive exponentially higher prior weight. The mixture ξ is a universal semimeasure that dominates every computable measure, placing it in the role of the Bayesian universal prior — the "correct" prior from the standpoint of algorithmic information theory.

### The AIXI Policy

Given the universal prior ξ, AIXI computes the action at time t as the one that maximises expected total future reward under ξ:

```
π^AIXI(h_t) = argmax_{a_t} Σ_{e_t} max_{a_{t+1}} Σ_{e_{t+1}} · · ·
              max_{a_T} Σ_{e_T} ( Σ_{k=t}^{T} r_k ) · ξ(e_{t:T} | h_t a_{t:T})
```

where h_t = a_1 e_1 r_1 … a_{t-1} e_{t-1} r_{t-1} is the full interaction history, the outer summations are over perception-reward pairs e_k = (o_k, r_k), and T is a horizon (finite or infinite). In the infinite-horizon discounted form, the reward sum becomes Σ_{k≥t} γ^{k-t} r_k for discount factor γ < 1.

The expectimax tree this induces has depth proportional to the horizon, and at each node the maximisation is over the agent's actions while the summation (under ξ) is over environment responses. AIXI plays the optimal policy in every computable environment simultaneously, because ξ assigns positive weight to every computable environment — the optimal action for the mixture is Bayes-optimal for any specific computable environment weighted by its posterior under ξ.

**Optimality theorem** (Hutter 2005, Theorem 5.36): AIXI is Pareto-optimal in the class of all policies. No policy achieves higher expected reward in all computable environments simultaneously. Simpler environments (lower K(ρ)) receive higher prior weight under ξ, so AIXI's effective performance is best in the environments that the universal prior regards as most probable — a precise formalisation of Occam's razor. Hutter (2005, Theorem 5.40) also establishes that AIXI self-optimises in ergodic environments: its per-step expected reward converges to that of the optimal policy as the interaction count grows. However, AIXI is not asymptotically optimal in all computable environments — Lattimore and Hutter (2011, *ALT*) showed that arbitrary environment classes can defeat any single policy — so Pareto-optimality should not be confused with universally bounded regret.

### Incomputability and Approximations

AIXI is not computable. Two separate obstacles prevent its implementation:

**Obstacle 1: Computing ξ.** The mixture over all computable measures requires summing over infinitely many programs, weighted by 2^{-K(ρ)}. Approximating this sum requires enumerating and running all programs up to some length bound, but programs may not halt within any fixed time bound. No finite computation implements ξ exactly.

**Obstacle 2: Solving the expectimax tree.** Even with an oracle for ξ, the expectimax tree has branching factor |A| × |E| at every node and depth equal to the horizon. For horizon T and action/perception spaces of size k, the tree has k^{2T} leaves — growing doubly exponentially in T.

The two principal approximations that trade off these obstacles against computability are:

**MC-AIXI** [Veness, Ng, Hutter, Silver, 2011, *JAIR* 40]: replaces the full ξ mixture with a context-tree weighting (CTW) model — a computationally efficient approximation to Bayesian mixture models over finite-memory Markov chains — and solves the expectimax tree with UCT (Monte Carlo tree search). CTW maintains a binary tree of depth equal to the context window, with each node storing sufficient statistics for a Bernoulli/KT estimator; the weighted probability of a symbol given all prior context is computed in O(n log n) time per symbol, where n is the context length. On several partially observable environments including Pac-Man and Kuhn Poker, MC-AIXI achieves near-optimal performance, demonstrating that the theoretical framework produces practical algorithms at finite scale. The regret bound for MC-AIXI degrades gracefully with the approximation quality of CTW relative to ξ — the error introduced by restricting to finite-depth context trees is bounded by the Kullback-Leibler divergence between ξ and the CTW model, which decreases as context depth grows.

**AIXI-tl** [Hutter 2005, §7.4]: restricts both the mixture class (to programs running in time ≤ t and space ≤ l) and the expectimax tree (solved with Levin search, see §216.4). AIXI-tl is computable and achieves near-AIXI performance in environments whose shortest description fits within the resource bounds. As t and l grow, AIXI-tl converges to AIXI in the limit — a theoretically satisfying but computationally impractical guarantee. The convergence rate is governed by the fraction of environments in the full AIXI mixture that are excluded by the (t, l) resource bounds: an environment of complexity K(ρ) = k that runs within resource bounds receives full AIXI weight 2^{-k}; excluded environments are effectively assigned weight zero, distorting the posterior toward simpler but resource-bounded models.

### AIXI and Self-Improvement

AIXI does not explicitly model self-improvement, but it subsumes it: the action space A includes self-modification actions as a special case, and ξ assigns positive prior weight to environment models where the agent's own code is a causal factor in subsequent rewards. An AIXI agent that discovers a self-modification producing higher expected reward under ξ will select it as an action. The self-improvement criterion is automatically derived from the utility function — no separate self-rewrite rule is needed as in the Gödel Machine.

This makes AIXI the more elegant theoretical object, but does not remove the specification problem. The reward function r_k still requires formalisation. And because AIXI is incomputable, its role is primarily as a **yardstick** — a reference point against which the suboptimality of any computable policy can be measured.

---

## 216.4 Levin Search and Solomonoff Induction

### Levin Search

Levin search (Leonid Levin, 1973, "Universal Sequential Search Problems", *Problemy Peredachi Informatsii* 9(3)) addresses the program search problem: given a predicate Q, find a program p such that U(p) satisfies Q. Naively, one could enumerate programs in length order — but a short program that runs for an exponential number of steps can dominate the search. Levin search interleaves length with runtime cost to find the optimal trade-off.

The Levin complexity (Kt complexity) of finding a solution is:

```
Kt(x) = min_p { |p| + log(t_p) : U(p) = x in exactly t_p steps }
```

Levin search allocates computational resources proportional to 2^{|p|+log(t_p)} = 2^{|p|} · t_p to each candidate program p, interleaving all programs simultaneously. The optimal program — the one minimising Kt — is found in time at most 2^{Kt(x)+O(1)}, at most a constant factor more expensive than any other systematic search strategy. No other search strategy is more than a constant factor faster, in the worst case over all problem instances [Levin 1973].

**Connection to self-improvement:** searching for a better policy is a program search problem. Given the current policy π_t, the search for a policy π_{t+1} that improves expected reward is exactly the problem of finding a program satisfying the predicate "runs better than π_t on the agent's task distribution". Levin search is the provably optimal strategy for this search, in the sense that no alternative program search procedure is asymptotically faster by more than a constant factor. AIXI-tl's expectimax solver uses Levin search precisely for this reason.

The practical difficulty is that the Levin complexity Kt of finding a better policy is typically enormous: a policy for a complex natural-language task is a long program, and running it to evaluate its quality on a benchmark suite may require millions of forward passes. Levin search allocates time proportional to 2^{|p|} · t_p per candidate, which for neural-network-scale policies is computationally prohibitive. The evolutionary methods of Chapter 215 are heuristic approximations to Levin search that sacrifice the worst-case guarantee in exchange for practical per-iteration cost. DGM's QD archive (§216.5) implicitly maintains a Levin-like priority: simpler, more general policies (shorter description length) are preferred when fitness is equal, because they occupy a larger basin in the archive's behavioural space.

```pseudocode
levin_search(predicate Q, time_budget T):
    allocate_map := {}
    for each program p in enumeration order:
        allocate_map[p] := 2^(-|p|) * T
    for phase k = 1, 2, 3, ...:
        for each p with 2^(-|p|) > 2^(-k):
            run U(p) for one additional step
            if Q(U(p)) satisfied:
                return p
```

The key property: program p of length |p| whose solution requires t_p steps is found by time O(2^{|p|} · t_p) — polynomial overhead over the intrinsic search cost.

### Solomonoff Induction

Solomonoff induction is the Bayesian learning counterpart to Levin search: instead of searching for a program satisfying a predicate, it predicts the continuation of an observed sequence. The predictor uses M (the algorithmic probability semimeasure) directly:

```
P_M(x_{n+1} = 1 | x_{1:n}) = M(x_{1:n} 1) / M(x_{1:n})
```

**Convergence theorem** (Solomonoff 1978): if the true sequence x is generated by any computable probability measure μ, the total squared prediction error of Solomonoff induction relative to the Bayesian-optimal predictor for μ is bounded in expectation by (ln 2) K(μ) / 2. In particular, for sequences generated by simple (low-K) environments, convergence is rapid. The bound is independent of n — it holds for the total error over the entire infinite sequence.

**Connection to policy learning:** a policy for a reinforcement learning agent is a sequential predictor of action-optimal responses. Solomonoff induction applied to the policy learning problem produces the Bayesian-optimal policy for any computable environment — which is AIXI. The conceptual chain is: Solomonoff induction (optimal prediction) → AIXI (optimal acting) → Gödel Machine (optimal self-modifying acting) — each step adding one more layer of self-reference.

An instructive analogy connects Solomonoff induction to transformer language models: a model trained by next-token prediction minimises cross-entropy — the KL divergence between the model's predictive distribution and the empirical data distribution — which approximates the Solomonoff predictor's goal of minimising expected squared error relative to the true distribution. Pre-training on large corpora is, in algorithmic information theory terms, building an approximation to the universal prior M by compressing a large and diverse dataset. By analogy with the convergence theorem, models with lower perplexity on held-out data are closer approximations to the Solomonoff predictor and should generalise better to novel environments — which is consistent with empirical scaling laws [Hoffmann et al. 2022, arXiv 2203.15556]. The analogy is not a theorem: Solomonoff induction has no compute-budget structure, and Chinchilla scaling laws are derived from empirical loss curves rather than algorithmic probability theory. The connection is heuristic but productive — it suggests that the theoretical optimality of Solomonoff induction explains, at a high level, why language model pre-training generalises as well as it does, and why richer and more diverse training data consistently helps.

---

## 216.5 Darwin Gödel Machine as Evolutionary Relaxation

The Darwin Gödel Machine (DGM) [arXiv 2505.22954, 2025] replaces the Gödel Machine's proof requirement with empirical fitness evaluation, converting an uncomputable theorem-prover into a computable evolutionary search. The formal relationship between DGM and the Gödel Machine is precisely the relationship between a statistical test and a formal proof: the former is computable and probabilistically reliable; the latter is incomputable and certain.

### The Core Substitution

The Gödel Machine's self-rewrite condition:

```
-- Gödel Machine: execute r iff F ⊢ V(t, apply(r)) > V(t, stop)
proof_condition(r) = F.prove("V(t, apply(r)) > V(t, stop)")
```

becomes in the DGM:

```
-- Darwin Gödel Machine: execute r iff fitness(apply(r)) > fitness(current)
fitness_condition(r) = evaluate_on_benchmark_suite(apply(r)) > evaluate_on_benchmark_suite(current)
```

The proof is replaced by empirical measurement on a benchmark suite. This is computationally feasible: no undecidable proof search is required. But the guarantee is probabilistic and dependent on benchmark coverage — a rewrite that passes all benchmarks may still regress on out-of-distribution inputs.

### Archive-Based Quality-Diversity

DGM maintains an archive of agent variants (corresponding to different rewrite histories) and applies Quality-Diversity (QD) search (see Chapter 215) to explore the space of possible self-rewrites:

```python
archive = {}                         # variant_id -> (policy_code, fitness_score, capability_vector)

def dgm_loop(seed_agent, n_iterations):
    archive[seed_id] = (seed_agent, evaluate(seed_agent), capabilities(seed_agent))
    for _ in range(n_iterations):
        parent = sample_from_archive(archive)       # QD selection
        child = self_rewrite(parent)                # LLM-generated rewrite of agent code
        child_fitness = evaluate(child)
        child_capabilities = capabilities(child)
        if dominates_or_fills_niche(child, archive):
            archive[child_id] = (child, child_fitness, child_capabilities)
    return best(archive)
```

The QD objective — maximise fitness while maintaining diversity in capability space — approximates the Gödel Machine's global optimisation criterion. The archive prevents the system from collapsing to a single local optimum, analogous to how the Gödel Machine's proof quantifies over all future trajectories rather than just the next step.

**The fitness function is the specification problem in disguise.** QD search maximises whatever `evaluate` measures. If `evaluate` captures the true intended capability well, DGM produces genuine improvements. If it captures a proxy metric, DGM optimises the proxy — Goodhart's Law in the evolutionary setting. This makes fitness function design (Chapter 218) the central engineering challenge for DGM, corresponding precisely to the utility function formalisation problem for the Gödel Machine.

### The Trade-off: Soundness for Computability

The DGM relaxation is a necessary engineering choice, not a flaw. A formal proof that a large language model policy improves expected cumulative reward across all environments would require a complete formal semantics of natural language interaction — an unsolved problem. DGM replaces this with a statistical test on a well-chosen evaluation suite. The trade-off is:

| Property | Gödel Machine | Darwin Gödel Machine |
|---|---|---|
| Completeness (finds all beneficial rewrites) | No (proof search may fail) | No (benchmark coverage incomplete) |
| Soundness (never executes a harmful rewrite) | Yes (conditional on F sound) | No (statistical guarantee only) |
| Computability | No (proof search undecidable) | Yes (benchmark eval is computable) |
| Applicable to LLM-scale agents | No | Yes |

The DGM is the most practical implementation of self-improvement theory available as of 2025, and its archive-based QD search gives it richer exploration properties than the gradient-based methods of Chapter 214. Its theoretical grounding in the Gödel Machine clarifies what it sacrifices.

---

## 216.6 Connection to Verified Compilation

Part XXIV established formal guarantees for compiler correctness: CompCert's verified semantics-preservation proof (Chapter 168), Vellvm's Coq formalisation of LLVM IR semantics (Chapter 169), and Alive2's translation validation for peephole rewrites (Chapter 170). These achievements raise an obvious question for self-improvement: can the verified compilation methodology certify self-rewrites of a learning agent?

### What Alive2 and Vellvm Can Certify

Alive2 [Lopes et al. 2021, *PLDI*] certifies LLVM IR peephole transformations by verifying that the transformed IR is a **refinement** of the source IR: every behaviour of the target is a possible behaviour of the source (under the LLVM IR semantics formalised in Vellvm). A peephole transform `src → tgt` is Alive2-certified if:

```
∀ state S, inputs I. behaviours(tgt(S, I)) ⊆ behaviours(src(S, I))
```

modulo the `undef`/`poison` semantics of Chapter 171. This is a formal proof of behavioural preservation for the specific class of IR-to-IR transformations expressible as pattern-match rewrites on LLVM IR.

Vellvm [Zhao et al., arXiv 1208.3717] provides the Coq semantics on which Alive2's refinement is defined. Together they constitute the most complete formal self-rewrite certification infrastructure available for compiler passes.

**The critical limitation for learning agents:** Alive2 and Vellvm operate on LLVM IR — a typed, explicitly structured intermediate representation with a precise operational semantics. Neural network weights are floating-point arrays. The "behaviour" of a weight update is not captured by IR semantics: the meaning of a weight tensor is its functional relationship to inputs and outputs across all possible inputs, which is not expressible as an LLVM IR refinement.

Concretely: a gradient descent step that updates W to W' in a linear layer produces a new function f_{W'} : ℝ^n → ℝ^m. Certifying that f_{W'} is a "refinement" of f_W would require:

```
∀ input x. |f_{W'}(x) - f_W(x)| < ε(x)
```

for some tolerance ε. This is a **numerical analysis** statement, not a type-theoretic refinement. No current tool (Alive2, Vellvm, or CompCert) addresses this class of statement. The gap is fundamental: verified compilation certifies discrete symbolic rewrites of structured programs; self-improvement in the neural network setting requires certifying continuous numerical modifications of unstructured parameter tensors.

### `drift` as a Practical Surrogate

The type `drift : Impl[T] → Spec[T] → DriftReport` from Chapter 207 (§207.3) provides the closest practical surrogate available. In that formalisation, `Impl[T]` is a program satisfying a formal specification `Spec[T]`, and `drift` produces a structured report of behavioural deviations detected between them. Applied to self-improvement:

```
-- Pre-modification specification
spec_before : Spec[Policy]

-- Post-modification implementation
impl_after  : Impl[Policy]

-- Check that post-modification policy still satisfies pre-modification spec
report : DriftReport = drift impl_after spec_before
```

A DGM self-rewrite is "Alive2-like certified" if `drift` returns an empty DriftReport — the post-rewrite implementation satisfies all obligations in the pre-rewrite specification. This is weaker than a formal proof (the spec must be machine-checkable, which requires formalising the agent's behaviour), but stronger than an empirical benchmark evaluation (the spec is persistent and cannot be gamed by optimising benchmark performance).

The formal analogy to the Gödel Machine's self-rewrite condition:

```
-- Gödel Machine criterion (incomputable)
F ⊢ V(t, apply(r)) > V(t, stop)

-- drift-based criterion (computable, incomplete)
fitness(apply(r)) > fitness(current)   AND   drift(apply(r), spec_current) = EmptyReport
```

The conjunction enforces both improvement (strictly higher fitness) and preservation (no spec violation). This is the strongest practically computable approximation to the Gödel Machine's self-rewrite condition currently available.

### Capability Regression Prevention as a Formal Constraint

Verified compilation provides one more tool: the refinement type. In CompCert's verification, every compilation pass is required to produce a program whose observable behaviours are a subset of the source program's observable behaviours — preservation is a type-level constraint that no pass is permitted to violate, checked by Coq proof. Translating this to self-improvement:

```
-- Capability regression constraint: a type-level invariant
CapabilityPreserving (r : Rewrite) : Prop :=
  ∀ (task : BenchmarkTask) (before after : Policy),
    after = apply(r, before) →
    score(after, task) ≥ score(before, task) - ε_regression
```

A self-improvement system that enforces `CapabilityPreserving` as a precondition for executing a rewrite is implementing the compiler-analogy: only refinement-typed rewrites may proceed. The challenge is that `score` for a neural network policy is an empirical quantity, making this a statistical constraint (with tolerance ε_regression > 0) rather than a formal proof. Continual learning methods (Chapter 214) such as EWC and GEM are engineering approximations to this constraint: they penalise parameter changes that increase the loss on previously mastered tasks, enforcing `CapabilityPreserving` approximately via gradient penalties.

---

## 216.7 The Specification Problem

### Goodhart's Law at the Foundation

Every formal self-improvement theory — Gödel Machine, AIXI, DGM — places the utility function or fitness measure at its foundation and assumes it is correctly specified. But specifying intelligence, capability, or helpfulness formally is an open problem that predates the modern AI era. Goodhart's Law (Goodhart 1975, reformulated by Strathern 1997) makes this precise: "When a measure becomes a target, it ceases to be a good measure." Optimising any proxy metric for intelligence will eventually diverge from optimising intelligence itself.

The formal statement in the self-improvement setting: let the true utility function be V^* (unspecified, representing what we actually want the agent to do) and let the formalised proxy be V (what we managed to write down). A Gödel Machine optimising V achieves:

```
V(t_∞) → maximum achievable under V           (by the optimality theorem)
V^*(t_∞) → unknown, possibly diverges         (Goodhart's Law)
```

The gap V^* − V measures specification error. Unlike approximation error (which decreases with more computation) or generalisation error (which decreases with more data), specification error does not decrease with either. It reflects a fundamental mismatch between the formalised utility function and the intended objective.

**Goodhart's Law is not a computational problem** — it is a specification problem. No amount of additional theorem-proving power in the Gödel Machine closes this gap.

### The Legg–Hutter Universal Intelligence Measure

The most ambitious formal specification of intelligence is the Legg–Hutter measure [S. Legg, M. Hutter, "Universal Intelligence: A Definition of Machine Intelligence", *Minds and Machines* 17(4), 2007]:

```
Υ(π) = Σ_{μ ∈ E} 2^{-K(μ)} · V_μ^π
```

where E is the space of all computable reward-generating environments, K(μ) is the Kolmogorov complexity of environment μ, and V_μ^π is the expected total reward of policy π in environment μ. The measure Υ(π) assigns a single real number to any policy, weighting performance across all computable environments by their prior probability under the universal semimeasure. By construction, AIXI maximises Υ: Υ(π^AIXI) ≥ Υ(π) for all policies π.

The Legg–Hutter measure makes precise what "general intelligence" means: performance across all computable environments, weighted by their description length. It avoids Goodhart's Law by ranging over all possible tasks rather than a fixed benchmark suite — there is no single proxy to overfit. However, it is incomputable (K(μ) is incomputable), so it functions as a theoretical specification rather than a practical fitness function.

**Implications for DGM fitness functions:** a DGM fitness function approximates Υ with a finite sample of environments from E. The quality of this approximation depends on how representatively the benchmark suite samples the full distribution over computable environments weighted by 2^{-K(μ)}. Since this distribution heavily favours simple environments, a benchmark suite biased toward complex, real-world tasks will systematically underweight performance on the simple, highly probable environments that dominate Υ. Calibrating practical benchmark suites against the Legg–Hutter ideal is an open research problem.

### Formalising Capability Preservation as a Typed Invariant

The most tractable partial solution to the specification problem is not formalising the full utility function V^* but formalising **preservation constraints** — types that rule out obvious regressions without requiring a complete specification of the optimisation target. The analogy from CompCert: the specification of a compilation pass is not "produce the most optimised output" but "preserve all observable behaviours of the source". The full target (optimal code) is hard to specify; the preservation constraint (refinement) is comparatively easy.

For a self-improving agent, the corresponding preservation constraints are:

```
-- Type-level capability preservation invariant
type CapabilitySet := List (Task × PerformanceThreshold)

PreservesCapabilities (r : SelfRewrite) (C : CapabilitySet) : Prop :=
  ∀ (task, threshold) ∈ C.
    score(apply(r, current_policy), task) ≥ threshold

-- Refinement constraint: self-rewrite is only executed if preservation holds
execute_rewrite (r : SelfRewrite) (C : CapabilitySet) : IO Unit :=
  if PreservesCapabilities r C then
    apply_rewrite r
  else
    reject_with_report r C
```

This type cannot be checked by a theorem prover (score is an empirical quantity), but it can be enforced by a test oracle that evaluates the proposed rewrite on the capability set C before execution. The formal connection to the Gödel Machine: `PreservesCapabilities` is a necessary condition for `V(t, apply(r)) > V(t, stop)` — any rewrite that fails capability preservation necessarily decreases expected utility. The drift-based check of §216.6 implements this as a continuous invariant check.

### Open Problems

The specification problem generates three clusters of open problems that the theory has not resolved and practical systems have not solved:

**1. Defining the capability set C.** A finite capability set C can always be exhausted — an agent can learn to ace every benchmark in C while failing on out-of-C inputs. The solution requires either (a) a continuously updated C that grows faster than the agent can overfit, or (b) a formal proof that C is representative of the true distribution — which requires knowing the true distribution.

**2. Aggregating preservation constraints with improvement criteria.** The self-rewrite condition requires simultaneously `score(apply(r)) > score(current)` (improvement) and `PreservesCapabilities r C` (no regression). These two objectives are in tension: an aggressive improvement-seeking rewrite may produce modest capability regression on the boundary of C. The formal trade-off requires specifying how much regression is acceptable for how much improvement — a utility function over the space of Pareto-optimal rewrites. The Gödel Machine defers this to V; DGM defers it to the fitness function design.

**3. The self-referential evaluation problem.** A self-improving agent that evaluates its own capability state is using itself to assess itself. Biases in the evaluator are inherited by the evaluation. This is structurally identical to the self-reference problem in Gödel's theorems: the formal system cannot fully characterise its own properties. The practical consequence (Chapter 218) is that external evaluators or adversarially generated test cases are required to avoid evaluation collapse.

---

## 216.8 Cognitive Angle

The Gödel Machine and AIXI are not algorithms. They are formalised visions of what optimal self-improvement looks like when all assumptions are satisfied. Their role in Part XXXI is the same role that the Chomsky hierarchy plays in compiler theory (Chapter 2) or that computable function theory plays in the study of optimising compilers: they define the theoretical upper bound, against which every practical implementation is measured as a calibrated approximation.

The practical methods surveyed in this part occupy specific positions in the space between theory and practice:

| Method | Proof Search | Utility | Soundness | Computability |
|---|---|---|---|---|
| Gödel Machine | Exhaustive formal search | Formalised V | Yes (if F sound) | No |
| AIXI | Expectimax under ξ | Formal reward signal | Yes (by construction) | No |
| AIXI-tl / MC-AIXI | Time/space bounded search | Formal reward signal | Approximately | Yes |
| DGM (Ch215) | QD evolutionary search | Empirical fitness | No | Yes |
| TTT / MAML (Ch214) | Gradient descent | Empirical loss | No | Yes |
| Self-reflective inference (Ch217) | Inference-time search | Implicit utility | No | Yes |

The theory establishes two results that have practical consequences for every row beneath the first two. First, **no computable policy is optimal in all computable environments** — any finite policy achieves suboptimal expected reward in environments that are complex relative to the policy's description length. This is not a deficiency to be engineered away; it is a theorem. Practical self-improvement systems should be understood as trading off environment complexity coverage for computability. Second, **the specification problem is irreducible** — no amount of optimisation power eliminates the gap between a formalised proxy and the intended objective. Capability preservation constraints are the principled engineering response: not formalising V^* completely, but ruling out the most obvious forms of divergence.

The deepest unresolved question in the field is whether the gap between the Gödel Machine's formal proof and the DGM's empirical fitness evaluation can be systematically narrowed using methods from verified compilation — whether `drift : Impl[T] → Spec[T] → DriftReport` can be made sufficiently rich that passing it constitutes a strong statistical proxy for the Gödel Machine's self-rewrite condition. Chapter 207's build roadmap (Phases 4 and 5) represents the current state of art for that direction; the theory of this chapter characterises how much farther there is to go.

One productive framing for future work is the **certification frontier**: the boundary in the space of possible self-rewrites separating those that can currently be certified (IR-to-IR peephole transforms via Alive2; test-passing criteria via DriftReport; capability preservation via held-out benchmarks) from those that cannot (weight-space modifications; capability generalisation beyond the benchmark distribution; self-referential evaluation). The certification frontier is not static — it advances as formal methods extend their reach (Lean 4 metaprogramming, arXiv 2403.14064), as mechanistic interpretability (Chapter 213) produces formal models of neural circuit behaviour that could in principle be certified, and as specification languages for neural network behaviours mature. The formal theory of this chapter is the map; the certification frontier is where the map currently runs out.

---

## Chapter Summary

- **Kolmogorov complexity** K(x) = min |p| such that U(p) = x formalises minimum description length; the universal probability m(x) = 2^{-K(x)} is the Bayesian prior that dominates all computable probability measures.
- **Self-reference limits** (Gödel 1931, Turing 1936) establish that no formal system can prove all truths about itself and no algorithm can decide all halting problems; any self-improvement system based on formal proof is necessarily incomplete.
- **The Gödel Machine** executes a self-rewrite r only when F proves `V(t, apply(r)) > V(t, stop)`, guaranteeing improvement conditional on formal system soundness; proof search is undecidable and the utility function requires full formalisation — both are fundamental obstacles.
- **AIXI** is the Bayesian-optimal agent under the Solomonoff–Hutter prior ξ = Σ_ρ 2^{-K(ρ)} ρ; it is Pareto-optimal in the class of all policies but incomputable; MC-AIXI (CTW + UCT) and AIXI-tl are the two principal computable approximations.
- **Solomonoff induction** is the optimal computable sequence predictor; its total squared error is bounded by (ln 2/2) K(μ) for any computable environment μ. **Levin search** is the optimal program search strategy with time allocation 2^{|p|} · t_p per candidate program.
- **Darwin Gödel Machine** replaces formal proof with empirical fitness evaluation and archive-based QD search; it is computable but sacrifices soundness — an empirically good rewrite may still regress on out-of-distribution inputs.
- **Alive2 and Vellvm** certify symbolic IR rewrites as behavioural refinements; they do not apply to weight-space self-modifications, which require numerical tolerance specifications rather than type-theoretic refinement.
- **`drift : Impl[T] → Spec[T] → DriftReport`** from Chapter 207 is the closest practical surrogate to the Gödel Machine's self-rewrite condition; combining a positive fitness check with an empty DriftReport enforces both improvement and preservation.
- **Goodhart's Law** (optimising a proxy diverges from optimising the true objective) is irreducible; capability preservation constraints — analogous to CompCert's refinement types — are the principled engineering response, ruling out regressions without requiring complete specification of V^*.
- **The Legg–Hutter measure** Υ(π) = Σ_μ 2^{-K(μ)} V_μ^π formalises universal intelligence; AIXI maximises it; practical benchmark suites approximate it at finite scale, systematically underweighting simple environments.
- The formal theory establishes the reference point for every practical method in Chapters 212–215 and 217–218: all are computable approximations that sacrifice some combination of soundness, completeness, and universality against computability.
