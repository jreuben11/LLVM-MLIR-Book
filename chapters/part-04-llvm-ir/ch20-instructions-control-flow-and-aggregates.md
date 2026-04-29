# Chapter 20 — Instructions II — Control Flow and Aggregates

*Part IV — LLVM IR*

Every instruction covered in Chapter 19 produces a value without altering the thread of execution. Control-flow instructions are different: their job is to decide *what executes next*. That decision carries profound consequences for every analysis that follows — alias analysis, value ranges, loop structure, exception safety — because the shape of the control-flow graph (CFG) determines what properties can be proven about program state at each point. Aggregate and vector operations occupy the other end of the spectrum: they manipulate composite values entirely inside the SSA value graph, with no branches and no side effects, yet they are the substrate on which vectorization and struct lowering stand.

This chapter is a systematic treatment of every instruction not covered in Chapter 19. It begins with the four terminator instructions that transfer control (`br`, `switch`, `indirectbr`, `callbr`), continues through `call` and `invoke` and their tail-call attributes, then addresses the cornerstone SSA instruction `phi` and its branchless companion `select`. The aggregate section covers `extractvalue` / `insertvalue` for structs and arrays, and `extractelement` / `insertelement` / `shufflevector` for vectors. The chapter closes with the exception-handling instruction set — both the Itanium `landingpad` / `resume` model and the Windows funclet model (`catchswitch`, `catchpad`, `cleanuppad`) — and the three remaining terminators `ret`, `resume`, and `unreachable`.

All LLVM IR examples are verified against LLVM 22.1.3 (`llvm-as` at `/usr/lib/llvm-22/bin/llvm-as`). All IR uses opaque `ptr` throughout.

---

## 20.1 Control Flow Terminators: `br`, `switch`, `indirectbr`, `callbr`

Every basic block must end with exactly one terminator instruction. Terminators are the only instructions that transfer control to another block; they are also the only instructions that define the CFG successor relation used by all analyses.

### 20.1.1 `br` — Conditional and Unconditional Branch

The simplest terminator comes in two forms:

```llvm
; Unconditional: always transfers to %target
br label %target

; Conditional: transfers to %true_bb if %cond is 1, else %false_bb
br i1 %cond, label %true_bb, label %false_bb
```

The condition operand is always `i1` — there is no implicit widening from `i8` or `i32`. Clang generates an `icmp` or `fcmp` result and feeds it directly into `br`. The backend lowers a conditional branch to a single machine compare-and-branch, or, when the condition was produced by a flag-setting instruction, it may fold the compare into the branch and eliminate the intermediate register.

*Why both successors can be the same block.* A conditional branch where both targets are identical is legal IR (`br i1 %c, label %L, label %L`). It is semantically equivalent to an unconditional branch to `%L` but retains the condition operand as a use — which matters when the condition has side effects through `poison` propagation (Chapter 19 §19.8). `SimplifyCFG` converts it to `br label %L` when the condition is provably dead.

The canonical CFG structure for `if-then-else` in C is:

```llvm
define i32 @max(i32 %a, i32 %b) {
entry:
  %cmp = icmp sgt i32 %a, %b
  br i1 %cmp, label %return_a, label %return_b
return_a:
  ret i32 %a
return_b:
  ret i32 %b
}
```

This verifies cleanly and represents the straightforward implementation without `phi`. A `phi`-based version, better suited to subsequent optimization, appears in §20.3.

### 20.1.2 `switch` — Multi-Way Branch

`switch` provides a compact representation of multi-way dispatch:

```llvm
switch i32 %val, label %default [
  i32 0, label %case0
  i32 1, label %case1
  i32 2, label %case2
]
```

The first operand is the integer value being compared. The second operand is the default label — reached when none of the listed cases match. The bracketed table lists (constant, label) pairs in any order; duplicate constants are illegal. The type of the switched-on value may be any integer type (`i8`, `i16`, `i32`, `i64`), but the constants in the table must have exactly the same type.

*Semantics.* The default label is not optional; it must be present even if the programmer has covered every possible value. This mirrors C's `default:` case. If the programmer asserts all values are covered (for example, an enum switch), the default label is typically a basic block containing `unreachable` (§20.8.2), which communicates to the optimizer that the default path is dead.

*Backend lowering strategies.* The backend has considerable latitude in how it lowers `switch`. Three main strategies exist, and the backend chooses among them based on the density and range of case values:

1. **Jump table.** When cases form a dense range (e.g., values 0–15 with few gaps), the backend emits a range check followed by an indexed load from a table of target addresses. This is O(1) dispatch. On x86-64 the sequence is: `sub`, bounds check (`ja default`), `movsxd` from the table, `jmp *rax`.

2. **Binary search tree.** When cases are sparse, the backend emits a balanced binary search over the literal values. This is O(log N) dispatch and requires no data table. It is the fallback for sparse enumerations.

3. **Bit test.** When the case range is narrow but sparse (e.g., a few booleans in a 32-bit range), the backend can check membership using a bit test: build a bitmask of all case values, shift `1 << val`, and test against the mask. A single branch handles all cases.

The backend's choice can be influenced by `!prof` metadata attached to the `switch` terminator. The metadata node specifies branch weights — the expected execution frequency of each arm relative to the others:

```llvm
switch i32 %val, label %default [
  i32 0, label %case0
  i32 1, label %case1
  i32 2, label %case2
], !prof !0

!0 = !{!"branch_weights", i32 1, i32 10, i32 5, i32 2}
```

The weight order matches the operand order: first weight is for the default, then case 0, case 1, case 2. Hot paths receive higher weights; the backend and block placement passes use these to order the case checks and decide whether to use a jump table (which pays the table load unconditionally) versus an inlined chain of comparisons (which short-circuits on a hot case). See [Chapter 22 — Metadata and Debug Info](../part-04-llvm-ir/ch22-metadata-and-debug-info.md) for the full metadata grammar.

### 20.1.3 `indirectbr` — Computed Goto

`indirectbr` implements computed goto — the C extension `&&label` that yields a label address as a `void *`, which can later be jumped to:

```llvm
; Table of target addresses (blockaddress constants from Chapter 18)
@dispatch_table = global [3 x ptr] [
  ptr blockaddress(@dispatch, %label0),
  ptr blockaddress(@dispatch, %label1),
  ptr blockaddress(@dispatch, %label2)
]

define void @dispatch(i32 %idx) {
entry:
  %slot = getelementptr [3 x ptr], ptr @dispatch_table, i64 0, i32 %idx
  %addr = load ptr, ptr %slot
  indirectbr ptr %addr, [label %label0, label %label1, label %label2]
label0:
  br label %exit
label1:
  br label %exit
label2:
  br label %exit
exit:
  ret void
}
```

The first operand to `indirectbr` is the runtime target address. The bracketed list that follows is the **complete set of possible target basic blocks**. This list is not optional and is not a hint: it defines the CFG successor set used by every analysis. If a block is reachable at runtime but absent from the list, analyses may produce incorrect results because they will never consider that edge. The list must be drawn exclusively from `blockaddress` constants within the same function — you cannot branch to a block in a different function.

*Use case: threaded interpreters.* The canonical application is a direct-threaded bytecode interpreter where each handler dispatches to the next handler without returning to a central fetch loop. With `indirectbr`, the dispatch cost is a single load and an indirect jump; the CPU's branch target predictor learns the (opcode → handler) mapping over the course of execution. The GCC `&&label` extension has been used for this purpose in CPython, MicroPython, and numerous embedded VMs. LLVM IR's `indirectbr` preserves the necessary CFG structure for optimizations (like inlining the interpreter loop) to remain sound.

*Analysis conservatism.* Because the target is a runtime value, static analyses must consider all listed successors as possible targets on every execution of the instruction. This prevents most loop-invariant code motion and CSE across `indirectbr`; it is the price of the generality. For most LLVM passes, `indirectbr` is treated like a switch with one case per listed label.

### 20.1.4 `callbr` — Inline Assembly with Jump Targets

`callbr` is the instruction backing GCC's `asm goto` extension — inline assembly that may branch to one or more C labels in addition to falling through:

```llvm
; An x86 inline asm that jumps to %zero if the input is zero
define i32 @test_callbr(i32 %x) {
entry:
  %res = callbr i32 asm "testl $1, $1; jz ${2:l}", "=r,r,!i,~{cc}"(i32 %x)
    to label %normal [label %zero]
normal:
  ret i32 %res
zero:
  ret i32 0
}
```

The syntax mirrors `invoke`: a fall-through successor (`to label %normal`) plus a bracketed list of possible indirect successors (`[label %zero]`). The constraint string uses `!i` to mark the label operand as an indirect output — the asm may or may not branch to it. All listed targets are real CFG edges.

*Motivation: `__builtin_expect` and static branch prediction.* Before `callbr` existed, `asm goto` was not representable in LLVM IR and Clang had to lower it with special handling. Today, `callbr` provides a first-class representation. It is also used in Linux kernel code that uses `asm goto` for static keys — the kernel replaces `nop` instructions with jumps at runtime (patching the `.text` section) for low-overhead tracepoints. The IR-level CFG edge from `callbr` to the indirect label is what allows LLVM to reason about the cold path without needing to mark it dead.

*Restrictions.* Unlike `invoke`, `callbr` cannot appear inside an EH region — it cannot have an `unwind` path. The values produced by `callbr` are only valid on the fall-through path; uses on the indirect successor paths must treat the result as `poison`.

---

## 20.2 Function Calls: `call`, `invoke`, and Tail Calls

### 20.2.1 `call` — Normal Function Call

```llvm
; Direct call to a named function
%ret = call i32 @function_name(i32 %arg0, float %arg1)

; Indirect call through a function pointer
%ret2 = call i32 %funcptr(i32 %arg0, float %arg1)

; Call with no return value
call void @log_event(ptr %msg)

; Call with multiple return values (struct return)
%pair = call { i32, i32 } @divmod(i32 %a, i32 %b)
```

The calling convention appears between `call` and the return type when non-default: `call fastcc i32 @fast_fn(...)`. Attributes on the call site (`nounwind`, `readonly`, `nonnull`, etc.) narrow the optimizer's view of the call's effects. A `nounwind` call site that reaches a `landingpad` triggers a verification error; a `readonly` call is treated as a pure function with no store effects for alias analysis purposes.

*Indirect calls and devirtualization.* When the function pointer is a `phi` over a set of possible callees, or when it is loaded from a vtable, the optimizer's [devirtualization pass](../part-10-analysis-middle-end/ch69-whole-program-devirtualization.md) attempts to convert the indirect call to a direct call (or a series of guarded direct calls). The pattern for a C++ virtual call generated by Clang is:

```llvm
; Load vtable pointer from object
%vptr = load ptr, ptr %obj
; Load the function pointer at virtual slot 2 (slot = 2 * pointer size)
%slot = getelementptr ptr, ptr %vptr, i64 2
%fn   = load ptr, ptr %slot
; Indirect call
%ret  = call i32 %fn(ptr %obj, i32 %arg)
```

With type-based devirtualization (`-fwhole-program-vtables`), LLVM may replace the indirect call with a conditional direct call when it can prove only one concrete type is possible. See [Chapter 69 — Whole-Program Devirtualization](../part-10-analysis-middle-end/ch69-whole-program-devirtualization.md).

### 20.2.2 `invoke` — Call with Exception Unwind

`invoke` behaves identically to `call` for the normal path, but adds an unwind path for exception handling:

```llvm
define i32 @try_catch() personality ptr @__gxx_personality_v0 {
entry:
  %ret = invoke i32 @might_throw()
      to label %normal unwind label %landing_pad
normal:
  ret i32 %ret
landing_pad:
  %lp = landingpad { ptr, i32 }
    cleanup
  resume { ptr, i32 } %lp
}
```

The `to label` target is the normal successor — reached when the callee returns normally. The `unwind label` target is the *landing pad* — a basic block that must begin with a `landingpad` instruction (§20.7.2). When the callee (or anything it calls) throws an exception and the unwinder selects this frame for cleanup or catch, control arrives at the landing pad.

*When to use `invoke` vs. `call`.* The choice is made by Clang's `CodeGenFunction`. A function call is lowered to `invoke` if and only if both conditions hold: (1) the call might throw — the callee is not marked `nounwind` — and (2) there is an active cleanup or catch scope at the call site. If the function is declared `noexcept` or marked `nounwind`, Clang emits `call`. If the function might throw but there is nothing to unwind (no destructors, no catch), Clang also emits `call` to avoid the overhead of registering unwind info for that frame. This optimization is explicitly permitted by the C++ standard: if an exception would escape a `noexcept` function, `std::terminate` is called rather than unwinding — the runtime handles this without needing a landing pad.

*ABI overhead.* Every `invoke` implies a nonzero overhead at the ABI level (LSDA tables in the `.gcc_except_table` section, unwind info in `.eh_frame`) even when no exception is thrown. This is the zero-cost EH model: no overhead on the fast path, but a page of metadata tables per function. See [Chapter 26 — Exception Handling](../part-04-llvm-ir/ch26-exception-handling.md) for the LSDA format.

### 20.2.3 Tail Call Attributes: `tail`, `musttail`, `notail`

The three tail-call qualifiers modify the optimizer's latitude in eliminating the call frame:

```llvm
; tail: a hint that TCO is beneficial but not required
%r1 = tail call i32 @external(i32 %x)

; musttail: a hard requirement — error if TCO cannot be performed
%r2 = musttail call i32 @musttail_example(i32 %x)
ret i32 %r2

; notail: suppress TCO even if the backend would otherwise apply it
%r3 = notail call i32 @external(i32 %x)
```

**`tail`** is a compiler hint. If the backend determines that tail-call optimization (TCO) is safe — the call is in tail position, no live stack values are needed after it, the calling conventions are compatible — it may convert the call to a jump. It is not an error if TCO is not performed.

**`musttail`** is a hard guarantee. If LLVM cannot lower the call as a tail call, it must emit a diagnostic rather than produce incorrect code. For `musttail` to be valid, all of the following must hold:
- The call is in tail position (the very next instruction is `ret`).
- The called function has exactly the same prototype as the caller (same parameter types, same return type, same calling convention, same `byval`/`sret` attributes).
- No allocas with dynamic lifetime overlap the call.
- No `inalloca` or `preallocated` arguments are present.

This strictness is intentional: `musttail` is the primitive needed for guaranteed tail-call elimination in functional language runtimes (Scheme, Haskell, Erlang) and for continuation-passing style (CPS) transforms. If a frontend emits `musttail` and the call cannot be eliminated, it indicates a bug in either the frontend or the pass that transformed the IR. Clang emits `musttail` for C23 `[[musttail]]` attribute and for Objective-C message sends that have been marked for mandatory tail calls.

**`notail`** explicitly suppresses TCO. It is used when a call must not be a tail call for correctness — for example, when a sanitizer needs the current frame to remain on the stack for accurate backtraces, or when the callee relies on reading caller-frame data after the call completes.

---

## 20.3 The `phi` Instruction

### 20.3.1 Syntax and Semantics

`phi` is the SSA join point. It selects a value based on which predecessor block transferred control to the current block:

```llvm
; Select %a if arriving from %then, or %b if arriving from %else
%result = phi i32 [ %a, %then ], [ %b, %else ]
```

Every `phi` must have exactly one `[value, label]` pair for every predecessor of the block. No more, no fewer. The value in each pair must be defined in the corresponding predecessor — or be a constant — and must be of the same type as the `phi` itself. The `phi` is conceptually evaluated at the *end* of the predecessor block, just before control transfers; this is why multiple `phi` instructions in the same block can refer to each other's pre-transfer values without circularity.

`phi` instructions must appear as a contiguous group at the beginning of a basic block, before any non-`phi` instruction. This is the canonical SSA structure: all join-point selections happen first, then computation begins.

### 20.3.2 The phi-Based Implementation of `max`

Rewriting the `max` example from §20.1.1 with `phi`:

```llvm
define i32 @max_phi(i32 %a, i32 %b) {
entry:
  %cmp = icmp sgt i32 %a, %b
  br i1 %cmp, label %then, label %else
then:
  br label %merge
else:
  br label %merge
merge:
  %result = phi i32 [ %a, %then ], [ %b, %else ]
  ret i32 %result
}
```

This form is better for optimization than the two-`ret` version because: (1) the `select` instruction rewrite (§20.4) is directly applicable — `SimplifyCFG` will convert this pattern to a single `select`; (2) value tracking can propagate range information through the single definition `%result`; (3) downstream uses see a single SSA value rather than two uses of `%a` and `%b` from different paths.

### 20.3.3 Domination Requirement

The correctness requirement for `phi` operands is stricter than it first appears: each value operand `%v` paired with predecessor `%pred` must *dominate the edge from* `%pred` *to the current block* — not merely dominate the current block. For most CFGs these are equivalent, but when a `phi` value is defined inside `%pred` itself and `%pred` has multiple successors, the value must be available on every path from `%pred` to the `phi`'s block. Formally: `%v` must dominate the terminator of `%pred`.

This distinction matters when critical edges exist (an edge from a block with multiple successors to a block with multiple predecessors). SSA construction algorithms — including Clang's `mem2reg` and LLVM's `SROAPass` — automatically split critical edges when necessary to maintain this invariant. The domination requirement is detailed in [Chapter 21 — SSA, Dominance, and Loops](../part-04-llvm-ir/ch21-ssa-dominance-and-loops.md).

### 20.3.4 Phi Cycles: Mutually Recursive Phis

Phi instructions may refer to each other, creating cycles in the use-def graph. This is not a circular definition; it models the behavior of loop-carried values:

```llvm
; Fibonacci: compute fib(n) iteratively using mutually dependent phi values
define i64 @fib(i64 %n) {
entry:
  br label %loop
loop:
  %i         = phi i64 [ 0,   %entry ], [ %i_next,   %loop ]
  %a         = phi i64 [ 0,   %entry ], [ %b,         %loop ]
  %b         = phi i64 [ 1,   %entry ], [ %a_plus_b,  %loop ]
  %a_plus_b  = add i64 %a, %b
  %i_next    = add i64 %i, 1
  %done      = icmp eq i64 %i_next, %n
  br i1 %done, label %exit, label %loop
exit:
  ret i64 %a
}
```

`%a` uses `%b` and `%b` uses `%a_plus_b` which uses `%a` — a cycle that is perfectly legal because all uses are mediated through loop-back edges. The SSA renaming algorithm (described in Chapter 9 of the companion theory coverage and [Chapter 21](../part-04-llvm-ir/ch21-ssa-dominance-and-loops.md)) handles this by processing dominators first, then back edges. The key insight is that the `phi` at loop entry captures the value *arriving on the back edge* from the previous iteration, which is always dominated by the prior iteration's computation.

### 20.3.5 The Alternative Without `phi`: `alloca` / `load` / `store`

Before `mem2reg` runs, Clang generates mutable variables as stack allocations:

```llvm
; Clang's initial output for 'int max(int a, int b)'
define i32 @max_alloca(i32 %a, i32 %b) {
entry:
  %a.addr = alloca i32
  %b.addr = alloca i32
  %result = alloca i32
  store i32 %a, ptr %a.addr
  store i32 %b, ptr %b.addr
  %av = load i32, ptr %a.addr
  %bv = load i32, ptr %b.addr
  %cmp = icmp sgt i32 %av, %bv
  br i1 %cmp, label %then, label %else
then:
  %av2 = load i32, ptr %a.addr
  store i32 %av2, ptr %result
  br label %end
else:
  %bv2 = load i32, ptr %b.addr
  store i32 %bv2, ptr %result
  br label %end
end:
  %r = load i32, ptr %result
  ret i32 %r
}
```

The `mem2reg` pass (part of the `-O1` pipeline) converts every `alloca` that does not have its address taken into SSA `phi` nodes, yielding the `phi`-based form from §20.3.2. This is why Clang does not need to implement SSA construction directly: it emits `alloca`-based IR and lets `mem2reg` + `SROA` build the SSA form.

### 20.3.6 MLIR Block Arguments

MLIR uses *block arguments* rather than `phi` instructions to express SSA join points. A block declares typed parameters, and each branch to that block passes a corresponding argument list — resembling a function call rather than a phi:

```mlir
// MLIR equivalent of max_phi
func.func @max(%a: i32, %b: i32) -> i32 {
  %cmp = arith.cmpi sgt, %a, %b : i32
  cf.cond_br %cmp, ^then(%a : i32), ^else_bb(%b : i32)
^then(%r0: i32):
  cf.br ^merge(%r0 : i32)
^else_bb(%r1: i32):
  cf.br ^merge(%r1 : i32)
^merge(%result: i32):
  return %result : i32
}
```

Block arguments are semantically equivalent to `phi` instructions but make the data-flow explicit at the branch site rather than at the join point. The LLVM dialect inside MLIR (`llvm.mlir.constant`, etc.) uses the same block-argument model; conversion to `phi` happens during lowering to LLVM IR. See [Chapter 130 — MLIR IR Structure](../part-19-mlir-foundations/ch130-mlir-ir-structure.md) for the full treatment.

---

## 20.4 `select`: Branchless Conditionals

### 20.4.1 Syntax and Semantics

`select` produces one of two values based on a condition, without creating any control-flow edges:

```llvm
; Scalar select: choose %a if %cond is 1, else %b
%r = select i1 %cond, i32 %a, i32 %b

; Vector select: element-wise, each lane independently controlled
%vr = select <4 x i1> %vmask, <4 x i32> %va, <4 x i32> %vb
```

Both the true and false operands are evaluated regardless of the condition — `select` has no short-circuit semantics. The type may be any non-void type including vectors, pointers, and floating-point types.

### 20.4.2 Difference from `br` + `phi`

The key distinction from a branch is that `select` does not create CFG edges. It is a single instruction that the register allocator sees as a data-flow node, not as control flow. On targets with a conditional move instruction (`cmov` on x86, `csel` on AArch64), the backend can lower `select` directly to a single machine instruction with no branch misprediction. On targets without `cmov`, the backend may insert a conditional branch — but that is an implementation detail of the backend, not a property of the IR.

The IR-level significance is that analyses cannot distinguish the two operands based on control flow. A branch `br i1 %c, label %t, label %f` creates two separate edges, and a value-range analysis can specialize each path. A `select i1 %c, i32 %a, i32 %b` has no such specialization: the optimizer must conservatively consider either value possible for `%r` after the `select`.

*When to use `select`.* Prefer `select` for simple conditional expressions where both operands are cheap to compute and no UB guard is needed. Use `br` + `phi` when either operand is expensive (a function call, a load with side effects) or when one operand would cause UB if unconditionally evaluated.

### 20.4.3 Interaction with `freeze` and Poison

If either operand of `select` is `poison`, the result is `poison` regardless of the condition. More subtly, if the *condition* is `poison`, the result is also `poison` — even if both operands are the same value. This asymmetry with respect to human intuition ("if both arms are 42, the result is always 42") is intentional: the optimizer may not convert `select i1 poison, i32 42, i32 42` to `i32 42` unless it can prove the operands are equal and freeze-clean.

The remedy is `freeze`:

```llvm
; If %cond might be poison, freeze it first
%safe_cond = freeze i1 %cond
%r = select i1 %safe_cond, i32 %a, i32 %b
```

`freeze` converts poison to an arbitrary but fixed value (0 or 1 for `i1`). The full semantics of `freeze` and its role in guarding speculative execution are covered in [Chapter 19 — Instructions I](../part-04-llvm-ir/ch19-instructions-arithmetic-and-memory.md) §19.8 and [Chapter 171 — The Undef/Poison Story Formally](../part-24-verified-compilation/ch171-the-undef-poison-story-formally.md).

---

## 20.5 Aggregate Operations: `extractvalue` and `insertvalue`

### 20.5.1 `extractvalue` — Reading a Field

`extractvalue` pulls one field out of a struct or array type value:

```llvm
%MyStruct = type { i32, float, i64 }

; Extract field 0 (the i32)
define i32 @get_field(%MyStruct %s) {
  %v = extractvalue %MyStruct %s, 0
  ret i32 %v
}

; Extract a nested field: field 1 of field 2
%Nested = type { i32, { float, i64 } }
define float @get_nested(%Nested %n) {
  %v = extractvalue %Nested %n, 1, 0
  ret float %v
}
```

The index list is a sequence of *literal integer constants* — not SSA values. This is the key difference from `extractelement` (§20.6.1): aggregate indices must be known at compile time, which means the result type is determined statically. The result type is the type of the field at the given index path — LLVM computes it mechanically from the aggregate type and the indices.

*No memory access.* `extractvalue` operates entirely within the SSA value graph. The aggregate it operates on is a value, not a pointer. This contrasts with `getelementptr` + `load`, which addresses into memory. If a struct was stored in an `alloca` and then loaded into a value, `extractvalue` and `getelementptr` + `load` produce equivalent results, but `extractvalue` carries no aliasing implications.

### 20.5.2 `insertvalue` — Producing a Modified Aggregate

`insertvalue` produces a new aggregate with one field replaced:

```llvm
; Produce a new MyStruct with field 0 replaced by %val
define %MyStruct @set_field(%MyStruct %s, i32 %val) {
  %new = insertvalue %MyStruct %s, i32 %val, 0
  ret %MyStruct %new
}
```

The original `%s` is not modified — SSA guarantees immutability. `insertvalue` is purely functional: it produces a fresh value. Building a struct field by field therefore requires chaining `insertvalue` instructions:

```llvm
; Construct %MyStruct { i32 1, float 2.0, i64 3 } from scratch
%s0 = insertvalue %MyStruct undef,    i32   1,   0
%s1 = insertvalue %MyStruct %s0,      float 2.0, 1
%s2 = insertvalue %MyStruct %s1,      i64   3,   2
; %s2 is now { 1, 2.0, 3 }
```

*Use in function return.* When a function returns a struct by value, Clang first calls `insertvalue` for each field (building up the struct in SSA registers), then `ret`s the complete struct. The ABI lowering in [Chapter 41 — Calls, the ABI Boundary, and Builtins](../part-06-clang-codegen/ch41-calls-the-abi-boundary-and-builtins.md) then decides whether to pass the struct in registers or via a hidden pointer argument.

---

## 20.6 Vector Operations: `extractelement`, `insertelement`, `shufflevector`

### 20.6.1 `extractelement` — Extracting a Lane

```llvm
; Extract lane 2 from a 4-element float vector
define float @get_elem(<4 x float> %v) {
  %e = extractelement <4 x float> %v, i64 2
  ret float %e
}
```

Unlike `extractvalue`, the index of `extractelement` is an SSA value — a runtime integer. This means the result type is always the element type of the vector (statically known), but *which element* is selected may be runtime-determined. If the index is out of range (`>= N` for `<N x T>`), the result is `poison`. The index type is typically `i64` or `i32`; both are accepted.

### 20.6.2 `insertelement` — Setting a Lane

```llvm
; Produce a new vector with lane 1 replaced
define <4 x float> @set_elem(<4 x float> %v, float %val) {
  %new = insertelement <4 x float> %v, float %val, i64 1
  ret <4 x float> %new
}
```

As with `insertvalue`, the original vector is unchanged. Out-of-range indices yield `poison` for the entire result vector.

### 20.6.3 `shufflevector` — Permutation, Broadcast, and Interleave

`shufflevector` is the Swiss Army knife of vector operations. It selects elements from two input vectors (of the same element type but potentially different widths) into a result vector, using a compile-time constant mask:

```llvm
shufflevector <N x T> %v1, <N x T> %v2, <M x i32> mask
```

The result has `M` elements. Each mask entry selects one element: indices `0` through `N-1` select from `%v1`; indices `N` through `2N-1` select from `%v2`. A mask entry of `-1` (represented as `i32 undef` in some tools, but `poison` is the preferred form in LLVM 22) means the corresponding result lane is `poison`.

**Broadcast (splat) — spread one element to all lanes:**

```llvm
define <4 x float> @splat(float %x) {
  ; Insert scalar into lane 0, then broadcast to all lanes
  %vec = insertelement <4 x float> poison, float %x, i64 0
  %broadcast = shufflevector <4 x float> %vec, <4 x float> poison,
                 <4 x i32> zeroinitializer   ; mask = [0, 0, 0, 0]
  ret <4 x float> %broadcast
}
```

**Interleave — merge two vectors into alternating lanes:**

```llvm
; Produce [a0, b0, a1, b1, a2, b2, a3, b3] from [a0..a3] and [b0..b3]
define <8 x float> @interleave(<4 x float> %a, <4 x float> %b) {
  %r = shufflevector <4 x float> %a, <4 x float> %b,
         <8 x i32> <i32 0, i32 4, i32 1, i32 5, i32 2, i32 6, i32 3, i32 7>
  ret <8 x float> %r
}
```

**Deinterleave — extract every other lane (gather even elements):**

```llvm
; Extract lanes [0, 2, 4, 6] from an 8-element vector
define <4 x float> @deinterleave_even(<8 x float> %v) {
  %r = shufflevector <8 x float> %v, <8 x float> poison,
         <4 x i32> <i32 0, i32 2, i32 4, i32 6>
  ret <4 x float> %r
}
```

**Reverse — flip element order:**

```llvm
define <4 x float> @reverse(<4 x float> %v) {
  %r = shufflevector <4 x float> %v, <4 x float> poison,
         <4 x i32> <i32 3, i32 2, i32 1, i32 0>
  ret <4 x float> %r
}
```

*Why a constant mask.* Requiring the mask to be compile-time constant means the result type, the set of source lanes accessed, and the data-flow graph are all statically known. The vectorizer relies on this: when it generates shuffle instructions during auto-vectorization, loop vectorization, or SLP, it must prove that the shuffle pattern is a compile-time constant. Dynamic permutation (a runtime index into a vector) requires a gather/scatter operation, which is a different LLVM intrinsic family (`llvm.masked.gather` / `llvm.masked.scatter`).

*Backend lowering.* On x86 with AVX2, a broadcast shuffle lowers to `vbroadcastss`. An interleave of two 4-wide float vectors lowers to `vunpcklps` + `vunpckhps` + two `vperm2f128` or equivalent. The backend's shuffle canonicalization pass (`X86VectorCombine` and `DAGCombine`) rewrites complex shuffle sequences into fewer native instructions. See [Chapter 64 — Vectorization Deep Dive](../part-10-analysis-middle-end/ch64-vectorization-deep-dive.md).

---

## 20.7 Exception Handling Instructions

Exception handling in LLVM IR is divided into two models that reflect the two dominant C++ ABI families: the Itanium (Linux/macOS/BSD) model based on `landingpad` / `resume`, and the funclet model (Windows MSVC SEH / C++ EH) based on `catchswitch` / `catchpad` / `cleanuppad`. Both models interact with `invoke` to integrate exception dispatch with the normal call graph.

### 20.7.1 The Itanium EH Model Overview

The Itanium EH ABI ([Itanium C++ ABI §3.4](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html)) separates *unwinding* (walking the stack) from *exception handling* (deciding what to do). When a `throw` executes, the runtime calls `__cxa_throw`, which calls `_Unwind_RaiseException`. The unwinder walks the stack frame by frame, consulting `.eh_frame` to find the return address for each frame and `.gcc_except_table` (the Language-Specific Data Area, LSDA) to find landing pads and their associated type tables. When the unwinder finds a frame with a matching catch clause, or a cleanup, it transfers control to the landing pad. The *personality function* (`__gxx_personality_v0` for C++, `__gcc_personality_v0` for C) is called during this process to interpret the LSDA and decide whether each frame handles the exception.

At the IR level, every function that participates in EH must declare its personality function:

```llvm
define i32 @try_catch() personality ptr @__gxx_personality_v0 {
  ...
}
```

### 20.7.2 `landingpad` — Receiving an Exception

```llvm
define i32 @try_catch() personality ptr @__gxx_personality_v0 {
entry:
  %ret = invoke i32 @throwing_fn()
      to label %normal unwind label %landing_pad

normal:
  ret i32 %ret

landing_pad:
  %lp = landingpad { ptr, i32 }
    catch ptr @_ZTIi    ; catch int
    catch ptr @_ZTId    ; catch double
  %exc_ptr = extractvalue { ptr, i32 } %lp, 0
  %sel     = extractvalue { ptr, i32 } %lp, 1
  %caught  = call ptr @__cxa_begin_catch(ptr %exc_ptr)
  call void @__cxa_end_catch()
  ret i32 -1
}

@_ZTIi = external constant ptr
@_ZTId = external constant ptr
declare ptr @__cxa_begin_catch(ptr)
declare void @__cxa_end_catch()
declare i32 @throwing_fn()
declare i32 @__gxx_personality_v0(...)
```

The `landingpad` instruction must be the first non-`phi` instruction in its basic block — the *landing pad block*. Its result type is always `{ ptr, i32 }`: the first element is a pointer to the exception object, and the second is an integer *selector* value that identifies which `catch` clause matched (as determined by `llvm.eh.typeid.for`).

**Clauses.** A `landingpad` has one or more clauses:
- `catch ptr @typeinfo_ptr` — catch exceptions of the type described by the typeinfo. `catch ptr null` is the catch-all (`catch (...)`).
- `filter [N x ptr] [ptr @ti1, ...]` — a C++ exception specification; the exception should be caught if its type is *not* in the filter list. Rarely used in modern C++ (exception specifications were deprecated and removed).
- `cleanup` — this landing pad performs cleanup (runs destructors) and then re-throws. A cleanup landing pad must end with `resume` rather than with a `ret`.

A single `landingpad` may have multiple clauses, checked in order. The personality function evaluates them and populates the selector field.

**`cleanup` flag.** When a landing pad has the `cleanup` attribute (with no catch/filter clauses, or in addition to them), the unwinder will transfer control here even if no matching catch clause exists. The cleanup runs (e.g., calling destructors) and then `resume` re-propagates the exception:

```llvm
define void @cleanup_example() personality ptr @__gxx_personality_v0 {
entry:
  invoke void @cleanup_fn()
      to label %normal unwind label %lpad
normal:
  ret void
lpad:
  %lp = landingpad { ptr, i32 }
    cleanup
  resume { ptr, i32 } %lp
}

declare void @cleanup_fn()
declare i32 @__gxx_personality_v0(...)
```

### 20.7.3 `resume` — Re-propagating an Exception

`resume` is a terminator that re-throws the in-flight exception. Its operand must be the `{ ptr, i32 }` value produced by a `landingpad` (possibly after modification of the selector):

```llvm
resume { ptr, i32 } %lp
```

`resume` is only valid in functions that have a personality function. Its role is to hand control back to the runtime unwinder so it continues up the call stack looking for the next handler. A cleanup landing pad that does not itself handle the exception always ends in `resume`.

### 20.7.4 The Funclet EH Model: `catchswitch`, `catchpad`, `cleanuppad`

Windows C++ EH and Structured Exception Handling (SEH) use a different model: *funclets*. A funclet is a fragment of a function that the OS exception dispatcher can call directly — it is not a normal function call, but the unwinder passes it a "funclet token" that represents the nesting context. LLVM IR exposes this model through three instructions that explicitly form a *parent-funclet tree*.

**`catchswitch`** — the entry point for a set of catch handlers:

```llvm
define void @windows_eh() personality ptr @__CxxFrameHandler3 {
entry:
  invoke void @do_work()
    to label %normal unwind label %catchswitch_bb

catchswitch_bb:
  ; Dispatch to one of the listed catch handlers, or unwind to caller
  %cs = catchswitch within none [label %catch_handler] unwind to caller

catch_handler:
  ; Catch anything (ptr null = catch-all, 64 = catch-by-value, ptr null = type)
  %cp = catchpad within %cs [ptr null, i32 64, ptr null]
  ; Return normally from the catch handler
  catchret from %cp to label %normal

normal:
  ret void
}

declare void @do_work()
declare i32 @__CxxFrameHandler3(...)
```

`catchswitch within %parent` establishes the funclet nesting. `within none` means this is a top-level handler (no enclosing funclet). The `[label ...]` list enumerates the catch handler blocks. `unwind to caller` means the exception propagates to the caller if none of the handlers match; `unwind label %L` would unwind to another funclet.

**`catchpad`** — the actual handler entry:

```llvm
%cp = catchpad within %cs [ptr @type_descriptor, i32 flags, ptr @catch_var]
```

The argument list is personality-specific. For `__CxxFrameHandler3`, the three arguments are the type descriptor, adjective flags, and a pointer to where the caught object is stored. The `catchpad` instruction produces a *token* value used to close the handler with `catchret`.

**`cleanuppad`** — for destructors and `finally` blocks:

```llvm
define void @with_cleanup() personality ptr @__CxxFrameHandler3 {
entry:
  invoke void @do_work()
    to label %normal unwind label %cleanup_bb

cleanup_bb:
  %cp = cleanuppad within none []
  call void @cleanup_action()
  cleanupret from %cp unwind to caller

normal:
  ret void
}

declare void @do_work()
declare void @cleanup_action()
declare i32 @__CxxFrameHandler3(...)
```

`cleanuppad` executes a cleanup block (destructor body, `__finally` body) and then `cleanupret` either unwinds to the caller or to another funclet. The `within` chain builds the funclet tree that the Windows EH runtime needs to correctly determine which funclet to call and in what order.

*Itanium vs. funclet: the key differences.*

| Property | Itanium (`landingpad`) | Funclet (`catchswitch`/`catchpad`) |
|---|---|---|
| Model | Single function with EH tables | Per-funclet callees with tokens |
| Dispatch mechanism | Personality function called during unwind | OS-level dispatcher (Windows SEH) |
| Landing pad result | `{ ptr, i32 }` value | Token consumed by `catchret`/`cleanupret` |
| Re-throw | `resume` | `cleanupret unwind to caller` |
| Nesting | Flat (selector distinguishes handlers) | Tree (within-chain) |
| Personality functions | `__gxx_personality_v0` (Itanium EH) | `__CxxFrameHandler3` (MSVC), `__C_specific_handler` (SEH) |

The full ABI-level treatment — LSDA tables, `.eh_frame` generation, and the personality function protocol — is deferred to [Chapter 26 — Exception Handling](../part-04-llvm-ir/ch26-exception-handling.md) and [Chapter 42 — C++ ABI Lowering: Itanium](../part-06-clang-codegen/ch42-cxx-abi-lowering-itanium.md).

---

## 20.8 `ret`, `resume`, and `unreachable`

### 20.8.1 `ret` — Return from a Function

```llvm
ret void          ; return from a void function
ret i32 %val      ; return a value
ret { i32, float } %aggregate  ; return a struct by value
```

`ret` is the normal terminator for a function. It must appear in every basic block that exits the function without throwing. A function declared as returning `void` must use `ret void`. A function with a non-void return type must use `ret T %v` where `T` matches the function's declared return type exactly. Mismatches trigger IR verification errors.

*Tail call interaction.* When `musttail` is used, the `ret` immediately following it is not a "real" return — it is consumed by the tail call optimization and the callee's return is passed directly to the caller of the current function. The IR still requires the `ret` syntactically (to make the basic block a valid terminator), but the backend erases it.

### 20.8.2 `unreachable` — Dead Code Marker

```llvm
; After a noreturn call, the optimizer needs to know execution stops
declare void @noreturn_fn() noreturn

define i32 @test_unreachable(i1 %cond) {
entry:
  br i1 %cond, label %then, label %else
then:
  call void @noreturn_fn()
  unreachable       ; tells the optimizer this block has no successors
else:
  ret i32 42
}
```

`unreachable` is a terminator that tells the optimizer the basic block can never be reached at runtime. It has no operands and no successors — it terminates the CFG at that point. The optimizer is free to remove any code that can only reach `unreachable`, prune CFG edges that lead to `unreachable` blocks, and use the assumption of non-reachability to sharpen value ranges on paths that avoid the `unreachable`.

*Sources in practice.* Clang generates `unreachable` in several contexts:
- Immediately after a call to a `noreturn` function (`__builtin_unreachable()`, `exit()`, `abort()`).
- In the default arm of a switch over an enum where the programmer (or the front-end) asserts all enum values are covered.
- After an `assert(false)` that Clang evaluates at compile time as unreachable in release mode.
- In the body of a pure-virtual function stub that should never be called.

The `SimplifyCFG` pass routinely folds edges to `unreachable` blocks, propagating the assumption back through the dominator tree.

---

## 20.9 Chapter Summary

- **`br`** provides conditional and unconditional branches. The conditional form requires an `i1` condition; the backend lowers it to a single compare-and-branch sequence.

- **`switch`** implements multi-way dispatch with a required default label. The backend selects among jump tables (dense ranges), binary search trees (sparse values), and bit tests (narrow sparse ranges). The `!prof branch_weights` metadata influences the lowering choice.

- **`indirectbr`** implements computed goto; its complete successor list is mandatory for sound CFG analysis. Used for direct-threaded interpreters and Linux static-key dispatch.

- **`callbr`** supports `asm goto` — inline assembly that may branch to IR labels. The result value is only valid on the fall-through path.

- **`call`** is the normal function-call instruction. Indirect calls through `ptr`-typed function pointers enable virtual dispatch; devirtualization converts them to direct calls when type information is available.

- **`invoke`** extends `call` with an unwind path to a landing pad. It is emitted by Clang only when there is active cleanup or catch state at the call site, preserving the zero-cost EH model.

- **`tail`** is a TCO hint; **`musttail`** is a hard requirement enforced by a verification check with precise prototype-matching requirements; **`notail`** suppresses TCO.

- **`phi`** is the SSA join point. Its operand must dominate the incoming edge from each predecessor, not merely the current block. Phi cycles express loop-carried values. MLIR uses block arguments as the equivalent abstraction.

- **`select`** is a branchless conditional that produces one of two values without creating CFG edges. Both operands are always evaluated. Poison in the condition or either operand propagates to the result.

- **`extractvalue` / `insertvalue`** operate on struct and array types using compile-time constant indices. They work on SSA values directly with no memory access. `insertvalue` is purely functional — it produces a new aggregate.

- **`extractelement` / `insertelement`** operate on vector types using runtime integer indices. Out-of-range indices yield `poison`.

- **`shufflevector`** is the universal vector permutation instruction. Its constant mask enables broadcast, interleave, deinterleave, reverse, and arbitrary element selection from two input vectors.

- The **Itanium EH model** uses `landingpad` + `resume`. The `landingpad` instruction yields a `{ ptr, i32 }` pair; `catch`, `filter`, and `cleanup` clauses define handler conditions; `resume` re-propagates an unhandled exception.

- The **funclet EH model** (Windows) uses `catchswitch`, `catchpad`, `cleanuppad`, `catchret`, and `cleanupret`. The `within`-chain builds the funclet-parent tree required by the OS exception dispatcher.

- **`unreachable`** marks dead code. The optimizer uses it to prune CFG edges and tighten value ranges on live paths. It is generated after `noreturn` calls, exhaustive switch default arms, and `__builtin_unreachable()`.


---

@copyright jreuben11
