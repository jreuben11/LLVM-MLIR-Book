# Chapter 171 — The Undef/Poison Story Formally

*Part XXIV — Verified Compilation*

No aspect of LLVM IR semantics has generated more confusion, more bugs, and more heated debate than the semantics of `undef` and `poison`. These are not mere implementation details: they are the interface between the optimizer's freedom to assume favorable conditions and the obligation to produce code that behaves correctly for every valid input. Getting this interface wrong — and LLVM got it wrong in multiple ways for years — leads to miscompilations that are subtle, hard to reproduce, and potentially catastrophic in security-sensitive code. This chapter gives a precise, formal account of the historical design, the problems discovered, the introduction of `freeze` and `poison`, and the current formal model. Understanding this story is essential for every LLVM pass author: it explains why certain optimizations are valid or invalid, why `freeze` is necessary before certain rewrites, and how Alive2 encodes these concepts to automatically catch violations.

---

## 171.1 Why Undef Exists

### The Optimization Motivation

Optimizing compilers need the freedom to make choices that simplify generated code. Two canonical scenarios:

**Uninitialized variables**: when a variable is allocated but not initialized before use, the value is unspecified — the program reads whatever was in that memory. The optimizer should be free to treat this value as "anything convenient": if a variable is uninitialized, the compiler can represent it as any value that makes surrounding code simplest to optimize. For example, if uninitialized `%x` is used as a select condition (`select i1 %x, 0, 0`), the optimizer can conclude the result is 0 regardless of `%x`'s actual value.

**Register allocation and spilling**: before a variable is assigned a physical register, it may logically exist but have no concrete representation. The optimizer may need to move, coalesce, or eliminate such "unassigned" variables; treating them as "any value" simplifies the analysis.

**Dead stores**: if a stored value is immediately overwritten before any load, the original store is dead. The new state of the memory cell after the first store and before the second can be treated as "any value" (the undef concept).

### Pre-LLVM-10 Design

In LLVM IR before version 10, a single concept `undef` served all these purposes. The LangRef description:

> `undef` can be used anywhere a constant is expected, and indicates that the user of the value may receive an unspecified bit-pattern.

The key property: `undef` is "fresh" at each use. If `%x = undef`, and `%x` is used in two places, each use may see a different bit pattern. This models the most permissive possible behavior: the optimizer can choose the bit pattern for each use independently to make the code as simple as possible.

Example:
```llvm
%x = undef i32
%a = add i32 %x, %x      ; could be 0, 2, 4, ... (two different undef values)
%b = select i1 undef, i32 1, i32 1  ; = 1 regardless (both branches same)
```

---

## 171.2 The `undef` Semantics

### The Formal Model

`undef` of type τ is formally modeled as an existentially quantified value: at each use site, a fresh arbitrary value of type τ is chosen. In terms of operational semantics, evaluating `undef` triggers the `PickE` effect (Chapter 169):

```coq
(* Vellvm's semantics for undef *)
| INSTR_Op (EXP_Undef) => trigger (Pick (fun _ => True))   (* any value *)
```

The "any value" interpretation means different uses of the same `undef` SSA value are independent:

```llvm
%x = undef i1
%result = select i1 %x, i32 42, i32 0
```

Here, `%x` is evaluated once to determine the branch. If the evaluation gives `false`, the result is 0; if `true`, 42. Since `undef` can be either, `%result` can be either. But within a single evaluation, `%x` takes one specific value.

The confusing case:
```llvm
%y = undef i32
%z = add i32 %y, %y
```

If `%y` is an `undef`, can the two uses of `%y` see different values? Under the "fresh at each use" model, yes. But this is not how LLVM actually implements it: in practice, `%y = undef i32` means the virtual register `%y` holds some unspecified value, and both uses of `%y` see the *same* value (whatever the register contains). The formal "fresh at each use" model is not what LLVM's semantics manual intended — it was a misunderstanding propagated by imprecise documentation.

### The Core Problem: Non-Compositionality

The "fresh at each use" semantics is **non-compositional**: the meaning of a use depends not just on the expression `undef` but on how many times and in what context it is used. This causes problems for optimizations that perform equational rewriting.

Consider the identity `select %cond, %x, %x → %x`. This is always correct when `%x` is a concrete value: regardless of `%cond`, both branches return `%x`, so the select is redundant. But if `%x = undef`:
- Source: `select %cond, undef, undef` — two uses of `undef`, potentially different values; result could be 0 or 1.
- Target: `undef` — one use, potentially a different value; result could be 0 or 1.

In isolation, these seem equivalent. But the problem arises in context:

```llvm
; Source
%x = undef i32
%r = select i1 %cond, i32 %x, i32 %x  ; %r = %x (any value)
%s = add i32 %r, %x                    ; %r + %x (two uses, both same as %x)
; = 2 * some_value

; After simplification: select → %x
%x = undef i32
%r = %x
%s = add i32 %r, %x                    ; %r + %x = %x + %x = 2 * some_value
```

Actually this case is fine. But consider:

```llvm
; Source
%x = undef i1
%v = select i1 %x, i32 1, i32 2       ; 1 or 2

; After "optimization": select true, 1, 2 = 1
%v = i32 1
```

This optimization (concretizing `undef i1` to `true`) is valid under the "optimizer chooses the value" interpretation. But it fixes the value of `undef` at one use site, while `%x` might still appear elsewhere with a different value in the source semantics.

### What "Freeze-Free" Undef Enables and Breaks

The original `undef` was designed to enable:
1. **Combine identities**: `%x + 0 → %x` even when `%x = undef` (same undef on both sides).
2. **Register coalescing**: two virtual registers that are never simultaneously live can share a physical register; when one is "dead," its value is `undef`.
3. **SROA**: breaking a struct into fields may leave some fields as `undef` if they were never written.

But it breaks:
1. **Commutativity with optimization**: applying two different optimizations in different orders can produce different results, because each use of `undef` is independent.
2. **The frame rule**: knowing that `%x = undef` and `use_x_once(%x)` terminates normally does not tell you that `use_x_twice(%x)` terminates normally — the second use might see a different value.
3. **Branch-on-undef semantics**: if `%x = undef i1` is used as a branch condition, the branch can go either way. But if `%x` is used in two separate branch conditions, they are independent — a peculiarity that no real hardware can implement.

---

## 171.3 The `poison` Value

### The Design of Poison

`poison` was introduced (by the LLVM community, driven by the work of Lee, Liu, Hurd, Regehr, and the Alive team) to fix `undef`'s compositionality problem. `poison` represents a single "bad" value — not a fresh-choice-at-each-use, but a specific designated bad value. Its properties:

1. `poison` is a single fixed (but unspecified) value: all uses of a `poison` SSA value see the same "bad" value.
2. Any operation on `poison` produces `poison` (poison propagates).
3. Using `poison` where a defined value is required triggers **undefined behavior**.

The third property is the key: `poison` is not immediately UB — it becomes UB only when you actually *use* it in a way that requires a defined value. This deferred UB model is what distinguishes `poison` from C's immediate UB.

### When Poison Is Produced

The following instructions produce `poison` under specific conditions (not UB — poison):

```llvm
add nsw i32 %x, %y    ; poison if signed overflow
add nuw i32 %x, %y    ; poison if unsigned overflow
sub nsw i32 %x, %y    ; poison if signed overflow
mul nsw i32 %x, %y    ; poison if signed overflow
shl i32 %x, %y        ; poison if shift amount ≥ bitwidth
lshr exact i32 %x, %y ; poison if any shifted-out bit is non-zero
udiv exact i32 %x, %y ; poison if %x is not evenly divisible by %y
getelementptr inbounds ... ; poison if GEP result is outside allocated bounds
```

Previously (before LLVM 10), some of these produced `undef` rather than `poison`. The migration to `poison` was intentional: `poison` is the right model because overflow with `nsw` is a single "bad" result that propagates consistently through the program, not a fresh value at each use.

### When Poison Triggers UB

`poison` becomes UB when used in:

```llvm
; Branch on poison
br i1 %poison_val, ...                  ; UB

; Function call where argument is poison
call void @foo(i32 %poison_val)          ; UB if @foo's parameter is not marked noundef

; Memory access through poison address
load i32, ptr %poison_ptr                ; UB
store i32 %v, ptr %poison_ptr           ; UB

; Return poison from function
ret i32 %poison_val                      ; UB (if return type is not a "passthrough" type)

; Arithmetic where intermediate poison causes UB
%r = udiv i32 1, %poison                 ; UB (divisor is poison → division is UB)
```

The trigger is: wherever the semantics requires a specific bit pattern to determine the next action (branch direction, address to access, function to call), `poison` makes that determination undefined.

### Poison Propagation

Poison propagates through computations:

```llvm
%a = add nsw i32 %x, 1     ; %a = poison if overflow
%b = mul i32 %a, 2         ; %b = poison (poison * 2 = poison)
%c = add i32 %b, 3         ; %c = poison
br i1 %icmp_using_c, ...   ; UB (branching on poison)
```

The poison "infects" a computation: once a value is poison, everything computed from it (without an intervening `freeze`) is also poison. This is the key difference from C's UB model where UB is "instantaneous" — in LLVM IR, poison carries the "poisonedness" forward until it is actually used destructively.

This model enables deferred UB: an optimization can produce a possibly-poison value early in a computation (e.g., computing with `nsw` flags), and the UB only manifests if the poison value eventually reaches a use that requires a defined value. If the value is discarded (dead code), no UB occurs.

### Compositionality of Poison

With `poison` (unlike `undef`), the identity `select %cond, %x, %x → %x` is valid even when `%x` is `poison`:
- Source: `select %cond, poison, poison` = `poison` (poison propagates through select).
- Target: `poison` (same).

These are the same. ✓

More generally, equational identities over concrete values hold over `poison` because `poison` propagates through all operations. The denotational semantics lifts cleanly: define `f(poison) = poison` for all operations `f`; then `f(x) = f(y)` implies `f(poison) = f(poison)`. The SSA use-def property is preserved: `%x = poison` means every use of `%x` sees the same `poison`.

---

## 171.4 The `freeze` Instruction

### The Problem with Undef/Poison in Loop Optimization

Consider a loop where the exit condition involves an uncertain value:

```llvm
loop:
  %i = phi i32 [0, %entry], [%i.next, %loop]
  %cond = icmp slt i32 %i, %n         ; n may be undef
  br i1 %cond, label %loop, label %exit
  %i.next = add nsw i32 %i, 1
```

If `%n = undef`, then at each iteration, `%cond` may be evaluated with a different value for `%n`. This means the loop might run 0, 1, 2, or any number of iterations — the semantics is non-deterministic in an uncontrolled way. An optimizer that tries to transform this loop (e.g., converting it to a counted loop) must handle this uncertainty.

The `freeze` instruction provides controlled resolution:

```llvm
%n.frozen = freeze i32 %n
loop:
  %i = phi i32 [0, %entry], [%i.next, %loop]
  %cond = icmp slt i32 %i, %n.frozen  ; use the frozen, stable value
  ...
```

After `freeze`, `%n.frozen` is a specific (but arbitrary) concrete value — no longer undef or poison. All uses of `%n.frozen` see the same value. This "freezes" the non-determinism: the loop now has a definite (if unspecified) iteration count.

### Freeze Semantics

Formally, `%y = freeze %x` is defined as:

```
freeze(v) =
  if v = undef or v = poison  then  arbitrary_but_fixed_value_of_type(typeof(v))
  else  v
```

The key properties:
1. `freeze(poison) ≠ poison`: `freeze` converts `poison` to a concrete value.
2. `freeze(undef) ≠ undef`: `freeze` converts `undef` to a concrete value.
3. `freeze(v) = v` when `v` is concrete.
4. Two separate `freeze(undef)` calls may return different values (each makes an independent arbitrary choice).
5. Within one `freeze` call, the result is stable: `%y = freeze %x` gives a single value; all uses of `%y` see the same value.

Property 5 is the critical one: `freeze` transforms the "fresh at each use" semantics of `undef` into the "once fixed, stable" semantics of a concrete value.

### freeze in Practice

The optimizer uses `freeze` when it needs to make a formerly-nondeterministic value stable for subsequent analysis. Common patterns:

**Loop induction variable upper bound**:
```llvm
; Before freeze
%n = some_potentially_undef_value
%trip.frozen = freeze i32 %n
; Use %trip.frozen as loop bound
```

**SROA cleanup**: when SROA breaks a struct into fields, some fields may be `undef`. If a field is used in a loop termination condition, it should be `freeze`'d before the loop.

**Select-to-branch lowering**: when lowering `select i1 %cond, %a, %b` to a branch (which reads `%cond` once), if `%cond = undef`, `freeze(%cond)` ensures the branch consistently goes one way.

### C++ API for freeze

```cpp
// Create a FreezeInst in LLVM C++
Value *FrozenV = Builder.CreateFreeze(PotentiallyPoison);

// Check if a value needs freezing (may be undef/poison)
bool needsFreeze(const Value *V) {
  // Heuristic: if V was computed with nsw/nuw flags, may be poison
  if (auto *BO = dyn_cast<BinaryOperator>(V))
    return BO->hasNoSignedWrap() || BO->hasNoUnsignedWrap();
  return isa<UndefValue>(V) || isa<PoisonValue>(V);
}
```

---

## 171.5 The Historical Evolution

### Pre-LLVM-10: The Wild West

Before LLVM 10 (2020), the IR had only `undef`:
- Arithmetic with overflow flags (`nsw`, `nuw`, `exact`) produced `undef` on violation.
- `undef` had the "fresh at each use" semantics in theory, but passes often assumed it was "one value chosen arbitrarily at the start."
- This inconsistency was a source of many subtle bugs.

Key papers that identified the problems:
- **"Undefined Behavior: What Happened to My Code?"** (Regehr, LLVM Developers' Meeting 2013): survey of UB-related bugs in LLVM.
- **"Taming Undefined Behavior in LLVM"** (Lee et al., PLDI 2017): formal model proposing `poison` as the replacement for arithmetic-flag UB; introduces the formal distinctions between `undef`, `poison`, and UB.

### LLVM 10–12: The Migration

LLVM 10 (2020) introduced `poison` as a first-class value in LLVM IR and began migrating passes:
- Arithmetic with `nsw`/`nuw`/`exact` flags now produce `poison` (not `undef`) on violation.
- `GEP inbounds` produces `poison` (not `undef`) when out-of-bounds.
- `freeze` instruction added.

LLVM 11–12 continued the migration:
- `select` semantics updated: `select i1 undef, %a, %b` produces `undef` (not one of `%a`, `%b`).
- `phi` semantics: if an incoming value is `poison`, the phi's result is `poison`.
- Many InstCombine patterns rewritten to correctly handle `poison` (not `undef`) propagation.

### LLVM 13+: The Settled State

As of LLVM 13, the major migration is complete:
- `poison` is the right value to use for "this computation result is invalid."
- `undef` is retained for "this memory location has never been written" (e.g., new `alloca` bytes before initialization).
- Most passes have been updated to use `PoisonValue::get(T)` for new code.
- `freeze` is used in loop transforms and elsewhere where a stable arbitrary value is needed.

The remaining `undef` uses are intentional: memory allocation, `llvm.undef` intrinsic for compiler-internal purposes, some targets that require "ignore this byte" semantics.

---

## 171.6 Formal Models

### Lee et al. (PLDI 2017): Taming Undefined Behavior in LLVM

The Lee et al. paper provides the first rigorous formal model for LLVM IR's `undef`/`poison`/UB semantics. Their model defines:

**Values**: `v ::= n | undef | poison`
- `n`: a concrete integer (bitvector).
- `undef`: a non-deterministic value (fresh at each use).
- `poison`: a single "bad" value; stable.

**Operations** (in the paper's model):
```
eval_binop(op, undef, v)   = undef      (undef propagates)
eval_binop(op, v, undef)   = undef
eval_binop(op, poison, v)  = poison     (poison propagates)
eval_binop(op, v, poison)  = poison
eval_binop(add nsw, n1, n2) = if overflow(n1+n2) then poison else n1+n2
eval_binop(add, n1, n2)   = n1 + n2    (modular arithmetic, always concrete)
```

**UB Trigger**:
```
trigger_ub(poison)  = True   (using poison as a branch condition → UB)
trigger_ub(undef)   = False  (undef as branch condition: nondeterministic but not UB)
trigger_ub(n)       = False
```

**Undef as existential quantifier**: in the Lee et al. model, `undef` is an existential quantifier over all possible values. When evaluating a program with `undef` values, the semantics is a set of possible outcomes (one for each possible resolution of each `undef`). An optimizer can choose any element of this set.

This model is a clean formalization of the "optimizer may choose any value for `undef`" intuition. The paper shows that with this model, several important optimizations are valid:
- Dead code elimination (compute the value but throw it away → equivalent to not computing it).
- Constant folding (replace an expression known to evaluate to `c` with `c`).
- Reassociation (reorder additions: valid because integers form a commutative group over concrete values and `undef` propagates through them).

### Alive2 Encoding

Alive2's encoding (Chapter 170) formalizes the semantics as Z3 formulas:

```python
# Conceptual encoding of undef
class Undef:
    def __init__(self, type):
        # Each use creates a fresh existential variable
        self._vars = {}
    
    def get_value_at_use(self, use_id):
        if use_id not in self._vars:
            self._vars[use_id] = z3.BitVec(f"undef_{use_id}", width)
        return self._vars[use_id]

# poison: a separate boolean flag per value
def is_poison(expr, state):
    return state.get_poison_flag(expr)

# UB: triggered when poison is used in a control-flow-determining position
def check_ub(val, state):
    if is_poison(val, state):
        state.set_ub()
```

Each LLVM IR SSA value in Alive2 is represented as a pair `(Z3_expr, bool_poison_flag)`:
- `Z3_expr`: the bitvector expression.
- `bool_poison_flag`: a Z3 boolean expression that is `true` when the value is poison.

Operations on values update both components:
```
add nsw %a, %b:
  result.expr       = bvadd(a.expr, b.expr)
  result.is_poison  = a.is_poison ∨ b.is_poison ∨ bvoverflow_signed(a.expr, b.expr)
```

The `bvoverflow_signed` function is expressible in QF_BV: `bvsadd_no_overflow` is a standard predicate in Z3. This encoding precisely captures the semantics: the result is poison if either input is poison or if the operation overflows with the `nsw` flag.

### Vellvm's UVALUE/DVALUE

As discussed in Chapter 169, Vellvm's `UVALUE` type has explicit `UVALUE_Undef` and `UVALUE_Poison` constructors. The interpreter handles `undef` via `PickE`:

```coq
| UVALUE_Undef dt =>
    (* Undef: nondeterministically pick any value of type dt *)
    trigger (Pick dt (fun _ => True))

| UVALUE_Poison dt =>
    (* Poison: poison propagates; no pick needed (it's a specific bad value) *)
    ret (UVALUE_Poison dt)
```

The difference is clear: `undef` fires a `PickE` effect (nondeterminism), while `poison` is a deterministic "bad" value.

When `poison` is used in a context that triggers UB:
```coq
| INSTR_Br (t, exp, b1, b2) =>
    '(m, v) <- eval_exp exp ;;
    match v with
    | DVALUE_I1 bit => if bit then jump b1 else jump b2
    | UVALUE_Poison _ => trigger (ThrowUB "branch on poison")
    | _ => raise "type error"
    end
```

---

## 171.7 Practical Implications for Pass Authors

### Producing Poison vs. Undef

When writing LLVM IR transformations, use the right "bad value":

```cpp
// New LLVM code: use PoisonValue for arithmetic/flag violations
Value *Poison = PoisonValue::get(Type::getInt32Ty(Ctx));

// Legacy code may use UndefValue (avoid for new passes)
Value *Undef = UndefValue::get(Type::getInt32Ty(Ctx));
```

Guidelines:
- Use `PoisonValue` for results of arithmetic operations with violated flags (nsw, nuw, exact).
- Use `UndefValue` only for "this memory was never written" (e.g., initial value of an alloca before any store).
- When using `PoisonValue`, ensure downstream uses correctly propagate it (do not silently convert to undef).

### When to Use freeze

Use `freeze` before:
- Using a value as a loop trip count or bound.
- Using a value as an array index in a transformation that assumes the index is valid.
- Splitting a single use of a value into multiple uses (to ensure consistent results).

Example: converting a counted loop to a post-increment loop requires the trip count to be stable:
```llvm
; Bad: trip count may be undef/poison, different on each check
%n = load i32, ptr %p, !range !{i32 0, i32 100}
loop:
  %cond = icmp slt i32 %i, %n    ; %n re-evaluated each iteration? No, it's SSA.
                                   ; but if %n is undef, two checks of the same %n
                                   ; can theoretically give different answers... 
                                   ; under old undef semantics. Use freeze to be safe.

; Safe: freeze %n first
%n.frozen = freeze i32 %n
loop:
  %cond = icmp slt i32 %i, %n.frozen
```

### The `noundef` Attribute

The `noundef` attribute on function parameters and return values asserts that the value is not `undef` or `poison`. If a `poison` value is passed to a function with a `noundef` parameter, that's immediately UB at the call site (not deferred):

```llvm
declare void @foo(i32 noundef %x)
...
call void @foo(i32 poison)   ; UB: passing poison to noundef parameter
```

Passes that analyze function arguments can rely on `noundef` to avoid tracking poison through function calls. It simplifies analysis at the cost of requiring callers to prove non-poisonness.

### Freeze in the C++ API

```cpp
#include "llvm/IR/Instructions.h"
#include "llvm/IR/IRBuilder.h"

// Create a freeze instruction
FreezeInst *FI = new FreezeInst(V, V->getName() + ".frozen", InsertPt);

// Using IRBuilder
IRBuilder<> Builder(InsertPt);
Value *Frozen = Builder.CreateFreeze(V, V->getName() + ".frozen");

// Check if a freeze is needed
bool mayBePoison(Value *V) {
  return V->getType()->isIntegerTy() &&
         // check if there's a path from a poison-producing instruction to V
         llvm::impliesPoison(V);
}
```

The `llvm::impliesPoison(V)` function (in `llvm/include/llvm/Analysis/ValueTracking.h`) checks whether `V` is always poison if any of its inputs are poison. This is used to decide where `freeze` is necessary.

### Summary of UB Taxonomy in LLVM IR

| Situation | Value | When UB occurs |
|-----------|-------|----------------|
| Uninitialized alloca | `undef` | Never directly (only if used as branch cond etc.) |
| Overflow with `nsw`/`nuw` | `poison` | When used as branch cond, mem addr, etc. |
| GEP `inbounds` violation | `poison` | When the pointer is dereferenced |
| Null pointer dereference | UB immediately | On the load/store instruction |
| Division by zero | UB immediately | On the `udiv`/`sdiv` instruction |
| Shift by ≥ bitwidth | `poison` | When result is used as branch cond etc. |
| `freeze(undef)` | concrete (arbitrary) | Never |
| `freeze(poison)` | concrete (arbitrary) | Never |

---

## Chapter Summary

- **`undef`** represents a value that may be any bit pattern, with the "fresh at each use" interpretation; it enables optimizations that need to choose a convenient value for an unspecified result.
- The "fresh at each use" semantics of `undef` causes **compositionality problems**: the same `undef` SSA value may behave inconsistently across uses, breaking equational reasoning.
- **`poison`** was introduced to fix this: a single "bad" value that propagates through computations and triggers UB only when actually used in a control-flow-determining context (branch condition, memory address, etc.).
- Arithmetic flags (`nsw`, `nuw`, `exact`, `inbounds`) now produce `poison` on violation, not `undef`; this was the main LLVM 10 migration.
- **`freeze %x`** converts `undef`/`poison` to a stable, concrete, arbitrary value; all uses of the frozen value see the same value; used before loop bounds, index variables, and any context where stability is required.
- The Lee et al. (PLDI 2017) formal model distinguishes: `undef` (existential choice, fresh per use), `poison` (stable bad value), and UB (program has no defined meaning).
- Alive2 encodes the distinction via per-value poison flags (Z3 boolean) and per-use existential quantification for `undef`.
- Vellvm uses `UVALUE_Undef` (fires `PickE`) and `UVALUE_Poison` (deterministic bad value) as distinct constructors.
- Pass authors should use `PoisonValue` for arithmetic violations, `UndefValue` only for uninitialized memory, and `freeze` when a stable arbitrary value is needed.
- The `noundef` attribute asserts that a parameter/return value is not `undef` or `poison`; passing poison to a `noundef` parameter is immediately UB.

### References

- Lee, J., Kim, Y., Song, Y., Hur, C.K., Das, S., Majnemer, D., Regehr, J., Lopes, N.P. (2017). "Taming Undefined Behavior in LLVM." *PLDI 2017*.
- Memarian, K. et al. (2019). "Exploring C Semantics and Pointer Provenance." *POPL 2019*.
- Lopes, N.P. et al. (2021). "Alive2: Bounded Translation Validation for LLVM." *PLDI 2021*.
- LLVM Language Reference Manual: [llvm.org/docs/LangRef.html](https://llvm.org/docs/LangRef.html) (undef, poison, freeze sections)
- Regehr, J. (2013). "Undefined Behavior: What Happened to My Code?" *LLVM Developers' Meeting 2013*.
- `llvm/include/llvm/Analysis/ValueTracking.h`: `impliesPoison`, `isGuaranteedNotToBePoison`
- `llvm/lib/IR/Constants.cpp`: `PoisonValue::get()`, `UndefValue::get()`


---

@copyright jreuben11
