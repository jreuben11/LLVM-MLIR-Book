# Chapter 40 — Lowering Statements and Expressions

*Part VI — Clang Internals: Codegen and ABI*

Every C and C++ program is a tree of statements containing expressions. The Clang code generator walks that tree and emits LLVM IR instruction-by-instruction. This chapter traces that walk from the top-level statement dispatcher down to the machinery that converts a single integer addition into a pair of `load` instructions and an `add nsw` instruction. Understanding this machinery is prerequisite to everything in the code generation pipeline: writing custom codegen passes, diagnosing miscompilations, implementing new language constructs in Clang, and understanding how sanitizers and undefined-behavior checks are woven into normal arithmetic.

The chapter follows the organization of `clang/lib/CodeGen/`: `CGStmt.cpp` for statements, `CGExprScalar.cpp` / `CGExprComplex.cpp` / `CGExprAgg.cpp` for the three expression families, `CGExpr.cpp` for LValue emission, and `CGCleanup.cpp` for cleanup scopes.

---

## Statement Visitor Dispatch

### EmitStmt and the Central Switch

`CodeGenFunction::EmitStmt()` in [`clang/lib/CodeGen/CGStmt.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGStmt.cpp) is the master dispatcher for the `Stmt` hierarchy. It examines the `Stmt::StmtClass` tag and forwards to a dedicated `Emit*` method. The function is deliberately a flat switch rather than a virtual dispatch: the `Stmt` hierarchy uses a non-virtual visitor pattern, and the switch lets the compiler optimize it as a jump table while keeping the codegen logic in one translation unit.

For a compound statement `{ s1; s2; ... }`, `EmitCompoundStmt()` opens a `LexicalScope` RAII object that records the source range for debug information, then iterates over child statements calling `EmitStmt` for each. The scope ensures that any variables declared inside the block are registered and destroyed at the closing brace. `LexicalScope` calls `DI->EmitLexicalBlockStart()` and `DI->EmitLexicalBlockEnd()` on the `CGDebugInfo` object to bracket the `DILexicalBlock` metadata.

### Conditional Statements

`EmitIfStmt()` handles `IfStmt` nodes, including `if consteval`, `if constexpr`, and the C++17 `if` with initializer. For a regular runtime condition:

1. Call `EmitBranchOnBoolExpr()` which evaluates the condition to an `i1` and emits a conditional branch.
2. Create two successor basic blocks (`then_bb`, `else_bb`) and a merge block (`cont_bb`).
3. Emit the then-branch into `then_bb`, conclude with an unconditional branch to `cont_bb`.
4. Emit the else-branch (or skip) into `else_bb`, conclude with an unconditional branch to `cont_bb`.
5. Position the builder at `cont_bb`.

```cpp
// Source
int test_if(int x) {
    if (x > 0) { return x * 2; } else { return -x; }
}
```

```llvm
define dso_local noundef i32 @_Z7test_ifi(i32 noundef %0) #0 {
  %2 = alloca i32, align 4          ; retval slot
  %3 = alloca i32, align 4          ; x
  store i32 %0, ptr %3, align 4
  %4 = load i32, ptr %3, align 4
  %5 = icmp sgt i32 %4, 0           ; x > 0
  br i1 %5, label %6, label %9      ; conditional branch

6:                                  ; then block
  %7 = load i32, ptr %3, align 4
  %8 = mul nsw i32 %7, 2
  store i32 %8, ptr %2, align 4
  br label %12                      ; jump to merge

9:                                  ; else block
  %10 = load i32, ptr %3, align 4
  %11 = sub nsw i32 0, %10
  store i32 %11, ptr %2, align 4
  br label %12                      ; jump to merge

12:                                 ; merge / continuation
  %13 = load i32, ptr %2, align 4
  ret i32 %13
}
```

The `nsw` flag on `mul` and `sub` reflects that signed overflow is undefined behavior in C — Clang annotates results of signed arithmetic with `nsw` unless `-fwrapv` is active.

### Loop Statements

All three loop forms follow the same four-block structure: **preheader**, **header** (condition test), **body**, **latch** (increment/continue target), **exit**. For a `while` loop the latch and header collapse: the latch unconditionally branches back to the header.

`EmitWhileStmt()` emits:
- An unconditional branch from the current block into a `while.cond` block.
- The `while.cond` block evaluates the condition and branches to `while.body` or `while.end`.
- The `while.body` block calls `EmitStmt` on the loop body.
- At the end of `while.body`, branch back to `while.cond` with `!llvm.loop` metadata attached.

`EmitForStmt()` adds a separate **increment block** (`for.inc`) between body and the branch back to `for.cond`. The latch is this increment block. Break edges target `for.end`; continue edges target `for.inc`.

```llvm
; for (int i = 0; i < n; i++) { result += i; }
; %5 = for.cond, %9 = for.body, %13 = for.inc, %16 = for.end
5:
  %6 = load i32, ptr %4, align 4    ; i
  %7 = load i32, ptr %2, align 4    ; n
  %8 = icmp slt i32 %6, %7
  br i1 %8, label %9, label %16

9:                                  ; body
  ...
  br label %13

13:                                 ; latch / increment
  %14 = load i32, ptr %4, align 4
  %15 = add nsw i32 %14, 1
  store i32 %15, ptr %4, align 4
  br label %5, !llvm.loop !8        ; back-edge with loop metadata

16:                                 ; exit
```

The `!llvm.loop` metadata carries loop ID used by loop transformation passes. `EmitDoStmt()` is symmetric: the condition appears in a `do.cond` block after the body rather than before it.

The `!llvm.loop` metadata node is a self-referential MDNode that carries sub-nodes describing loop properties. When `EmitForStmt()` or `EmitWhileStmt()` emits the back-edge branch, it annotates it with this metadata via `LoopInfo::getLoopID()`. The loop ID enables the `LoopVectorize`, `LoopUnroll`, and `LoopInterchange` passes to associate pragmas and hints with the correct IR loop. Clang also honors `#pragma clang loop` by setting loop attribute sub-nodes (`llvm.loop.unroll.count`, `llvm.loop.vectorize.width`, etc.) during `EmitLoopHeader()`.

### Switch Statements

`EmitSwitchStmt()` creates an `llvm::SwitchInst` with the discriminant value and populates it by calling `EmitSwitchCase()` for each `CaseStmt`. The default label is attached as the `SwitchInst`'s default destination. Case ranges (GNU extension `case 1 ... 5:`) are broken out into a separate linear search sequence since `SwitchInst` requires single-value cases.

```llvm
switch i32 %4, label %8 [          ; %8 = default
  i32 1, label %5
  i32 2, label %6
  i32 3, label %7
]
```

After constructing the `SwitchInst`, `EmitSwitchStmt` pushes the switch instruction onto a stack so that nested `break` statements can find the exit block. Fallthrough between cases is the default: each case block ends with an unconditional branch to the next case label unless the case body ends with a `break`.

`EmitSwitchStmt` also handles range-based case clusters. When the optimizer (particularly `SimplifyCFG`) processes the IR, it may transform a dense set of case values into a lookup table (`switch` → array index → load) or a bit-test sequence, but this is an IR transformation not a Clang codegen transformation. Clang simply emits one entry per case value.

For a `switch` on a C++ class type with a user-defined conversion to integer, `EmitSwitchStmt` first emits the conversion call via `EmitScalarExpr` and uses the resulting integer as the discriminant. For `switch` on an enumeration type, the discriminant is loaded and possibly extended to the switch type; `SwitchInst` requires the type of all case values and the discriminant to match exactly.

### Return Statements

`EmitReturnStmt()` does not directly emit a `ret` instruction. Instead it:
1. Stores the return value into the return-value alloca (the `%retval` slot established by `StartFunction()`).
2. Calls `EmitBranchThroughCleanup()` targeting the return block, which triggers any active cleanup scopes.
3. The actual `ret` instruction is emitted at the function epilogue in a dedicated `return_block`.

This design allows every early-return path to share cleanup code rather than duplicating destructor calls inline.

For functions that return aggregates, the return value slot is an `AggValueSlot` pointing at the sret parameter (the hidden first argument carrying the address of the caller-provided return buffer). `EmitReturnStmt` calls `EmitAggExpr` with that slot as the destination, so the return expression is constructed directly in the caller's buffer. If the compiler can prove the return expression is a named local variable (NRVO candidate), it marks the local's alloca as the return slot itself, avoiding the intermediate copy entirely.

Void functions call `EmitReturnStmt` with a null expression, causing a direct branch to the return block where only cleanup runs.

---

## Break, Continue, and Goto

### BreakContinueStack

`CodeGenFunction` maintains a `SmallVector<BreakContinue, 8> BreakContinueStack`. Each entry holds two `JumpDest` values: the break destination (exit of the current loop or switch) and the continue destination (loop latch or increment block). `EmitBreakStmt()` and `EmitContinueStmt()` read the top of this stack and call `EmitBranchThroughCleanup()` to reach the target, unwinding any cleanup scopes entered since the loop began.

`JumpDest` encapsulates an `llvm::BasicBlock*` together with a `EHScopeStack::stable_iterator` marking the innermost scope that must be exited when taking this branch. `EmitBranchThroughCleanup()` compares the current scope depth against the destination scope depth and emits cleanup calls for each intermediate scope.

Consider a `for` loop where the body declares a `std::string s` before a `continue`:

```cpp
for (int i = 0; i < n; i++) {
    std::string s = compute(i);
    if (skip(i)) continue;    // must destroy s first
    use(s);
}
```

`EmitContinueStmt()` does not branch directly to `for.inc`. Instead `EmitBranchThroughCleanup` emits the destructor call for `s` (a `NormalCleanup` scope pushed by `EmitAutoVarDecl`) then branches to `for.inc`. The same cleanup-through mechanism applies to `break` out of a loop that declares locals with destructors, and to `goto` that crosses destructor boundaries — Clang enforces that gotos cannot jump into the scope of a variable with a non-trivial constructor.

### Forward Goto and Label Resolution

`EmitGotoStmt()` calls `GetAddrOfLabel()` to obtain the `llvm::BasicBlock*` for the target label. If the label has not been encountered yet (a forward goto), `GetAddrOfLabel()` creates a placeholder block and registers it in `LocalLabelMap`. When the label statement is subsequently visited by `EmitLabelStmt()`, the placeholder block is completed. This design handles all permutations of forward and backward goto without requiring a two-pass approach.

`EmitBranchThroughCleanup` must also interoperate with `goto`. If a `goto` jumps out of a scope containing C++ objects with destructors, the branch path must execute those destructors. Clang emits a "cleanup block" on the goto path in the same way it does for `return`; the `goto` branches to the cleanup block, which falls through to the label. The resulting IR structure shares cleanup blocks between `goto`, `return`, and `break` statements that cross the same scope boundaries, controlled by the `EHScopeStack::stable_iterator` embedded in each `JumpDest`.

### Indirect Goto and BlockAddress

GNU C's `&&label` extension obtains the runtime address of a label. Clang lowers `&&label_a` to `llvm::BlockAddress::get(CurrentFunction, label_a_block)`. The global constant table receives these `blockaddress` values:

```llvm
@table = internal global [3 x ptr] [
  ptr blockaddress(@_Z13indirect_gotoi, %8),
  ptr blockaddress(@_Z13indirect_gotoi, %9),
  ptr blockaddress(@_Z13indirect_gotoi, %10)
], align 16
```

`EmitIndirectGotoStmt()` loads the target address and emits `indirectbr`:

```llvm
indirectbr ptr %13, [label %8, label %9, label %10]
```

The bracket lists all possible successor labels; the optimizer and verifier require this list to be complete. Clang collects these labels by tracking all `&&label` expressions seen in the function.

---

## Expression Codegen Visitors

### Three Scalar Families

Clang partitions expressions into three value categories by representation:

| Family | Handles | Emitter class |
|--------|---------|---------------|
| Scalar | `bool`, integers, floats, pointers, references | `ScalarExprEmitter` |
| Complex | `_Complex float`, `_Complex double`, `_Complex long double` | `ComplexExprEmitter` |
| Aggregate | structs, unions, arrays, non-trivial class types | `AggExprEmitter` |

`EmitScalarExpr()` instantiates `ScalarExprEmitter` (defined in `CGExprScalar.cpp`) and calls `Visit(E)`, which dispatches to typed `Visit*` methods via the recursive `StmtVisitor` base class. The result is a single `llvm::Value*`.

`EmitComplexExpr()` returns a `ComplexPairTy` — a pair of `llvm::Value*` holding the real and imaginary components separately, not as a packed struct.

`EmitAggExpr()` takes an `AggValueSlot` that specifies the destination memory: where the aggregate value should be constructed in place. This avoids unnecessary copies.

The classification is performed by `CodeGenFunction::getEvaluationKind()` (delegating to `CodeGenTypes::getTypeForMem()`), which examines the `QualType` of the expression. A type is *scalar* if it is an integer, floating-point, pointer, member-pointer, or reference type. A type is *complex* if it has the `_Complex` specifier. All other record and array types are *aggregate*. This classification determines which `Emit*Expr` entry point is used and therefore which emitter class is instantiated.

The `StmtVisitor` pattern used by all three emitters relies on the CRTP visitor base defined in `clang/include/clang/AST/StmtVisitor.h`. Each `Visit*` override returns the appropriate value type (`llvm::Value*` for scalars, `ComplexPairTy` for complex, `void` for aggregates since the result is written to the slot). The `StmtVisitor::Visit()` method dispatches on `Stmt::getStmtClass()` through a generated switch table, achieving O(1) dispatch without virtual functions.

### AggValueSlot

`AggValueSlot` carries:
- The destination address and alignment.
- Whether the slot is the object's final location (affects when destructors are registered).
- Whether the memory is considered externally destructed (the caller handles cleanup).
- Zero-initialization preference.

When a caller needs an aggregate return value, it passes an `AggValueSlot` pointing at the return-value alloca. If the aggregate is being passed by value into a function call, the slot points at the argument temporary. This design is crucial for copy-elision: when the slot is the final location, the emitter can construct directly there.

---

## Arithmetic and Bitwise Operations

### ScalarExprEmitter::VisitBinaryOperator

Arithmetic operators dispatch through `ScalarExprEmitter::VisitBinaryOperator()`. For integer arithmetic, the emitter checks whether the operation can carry `nsw` or `nuw` flags:

- `nsw` (no signed wrap) is set for signed integer operations when `-fwrapv` is not active. Signed overflow is undefined behavior in C and C++, so Clang asserts the invariant in the IR.
- `nuw` (no unsigned wrap) is set only when the compiler can prove unsigned overflow cannot occur; unsigned wraparound is defined behavior in C so `nuw` is conservative.

The flag selection logic is in `CodeGenFunction::HasSignedOverflowBehavior()` and `BinOpInfo::canOverflow()`. Key cases:
- `a + b` where both `a` and `b` are `int`: `nsw` set unless `-fwrapv`.
- `a + b` where both are `unsigned int`: no `nuw` (wraparound is defined).
- `a + b` where one side is a pointer: no overflow flag; pointer arithmetic uses GEP semantics.
- `a + 1` where `a` is a result of an `__int128` signed expression: `nsw` applies.

The `-fwrapv` flag sets `LangOptions::OverflowWraps = true`, which suppresses all `nsw`/`nuw` flags on integer arithmetic. The `-fstrict-overflow` (default) keeps them. The `-fno-strict-overflow` alias has the same effect as `-fwrapv`.

```llvm
; int add(int a, int b) { return a + b; }
%7 = add nsw i32 %5, %6

; unsigned add(unsigned a, unsigned b) { return a + b; }
%7 = add i32 %5, %6               ; no nuw without proof
```

### Sanitizer-Instrumented Arithmetic

When `-fsanitize=signed-integer-overflow` is active, `EmitOverflowCheckedBinOp()` replaces the bare `add` with the `llvm.sadd.with.overflow` intrinsic. The intrinsic returns a `{i32, i1}` struct; the `i1` overflow bit triggers a call to `__ubsan_handle_add_overflow` on the slow path:

```llvm
%7 = call { i32, i1 } @llvm.sadd.with.overflow.i32(i32 %5, i32 %6)
%8 = extractvalue { i32, i1 } %7, 0   ; result
%9 = extractvalue { i32, i1 } %7, 1   ; overflow flag
%10 = xor i1 %9, true                 ; !overflow
br i1 %10, label %14, label %11       ; likely branch: no overflow

11:                                   ; overflow handler
  call void @__ubsan_handle_add_overflow(ptr @.src, i64 %12, i64 %13)
  br label %14

14:                                   ; continue with %8
```

The `!prof !{!"branch_weights", 2000, 1}` branch weight metadata on the check branch tells the optimizer the fast path (no overflow) is overwhelmingly likely.

Analogous intrinsics exist for `llvm.ssub.with.overflow`, `llvm.smul.with.overflow`, and their unsigned variants.

### Division and Remainder

`EmitDiv()` and `EmitRem()` handle `/` and `%`. For integer division, two checks may be necessary under `-fsanitize=undefined`:
1. Division-by-zero: `icmp eq divisor, 0` → call `__ubsan_handle_divrem_overflow`.
2. Signed overflow: the case `INT_MIN / -1` (the only signed division that overflows) is checked separately.

Without sanitizers, a plain `sdiv` or `udiv` (and `srem`/`urem`) is emitted. Integer division in LLVM has defined behavior for non-zero divisors; the undefined behavior at divisor zero comes from the source language, not LLVM.

For floating point, `fdiv` is emitted. Floating-point division by zero produces an IEEE infinity, not a trap.

### Shift Operations

`EmitShl()` and `EmitShr()` emit `shl` and `ashr`/`lshr`. Under `-fsanitize=shift`, two checks are injected:
- Shift amount must be in `[0, bit_width - 1]`.
- For left shifts of signed types, the shifted value must not overflow (C++14 and later: `E1 << E2` is undefined if the result cannot be represented in the result type).

Without sanitizers, shifts emit directly to `shl`/`ashr`/`lshr` with the implicit LLVM poison-on-excessive-shift semantics matching C's undefined behavior.

---

## Comparison and Logical Operators

### Integer and Floating-Point Comparison

`VisitBinaryOperator` routes comparison operators to `EmitCmp()`. Integer comparisons become `ICmpInst` with the appropriate predicate:

| C operator | Signed | Unsigned |
|------------|--------|----------|
| `<` | `icmp slt` | `icmp ult` |
| `<=` | `icmp sle` | `icmp ule` |
| `>` | `icmp sgt` | `icmp ugt` |
| `>=` | `icmp sge` | `icmp uge` |
| `==` | `icmp eq` | `icmp eq` |
| `!=` | `icmp ne` | `icmp ne` |

Floating-point comparisons use `FCmpInst` with ordered predicates (`oeq`, `olt`, etc.) when the types are not `__attribute__((nnan))` annotated. Unordered predicates (`ugt`, `ult`) are used when NaN semantics require it.

### Short-Circuit Logical Operators

`&&` and `||` are not dispatched through `VisitBinaryOperator`; they receive special treatment through `EmitBranchOnBoolExpr()` to ensure short-circuit evaluation with side effects.

For `a && b`:

```llvm
; bool and_test(bool a, bool b) { return a && b; }
  %8 = trunc i8 %7 to i1
  br i1 %8, label %9, label %12   ; if !a, skip to merge with false

9:                                ; a was true, evaluate b
  %10 = load i8, ptr %4, align 1
  %11 = trunc i8 %10 to i1
  br label %12

12:                               ; merge
  %13 = phi i1 [ false, %2 ], [ %11, %9 ]   ; phi for result
  ret i1 %13
```

For `a || b`, the structure is symmetric: if `a` is true, skip to the merge with `true`; otherwise evaluate `b`.

The `phi` node at the merge block is the classical SSA representation of the conditional value. This differs from a hardware `and`/`or` instruction because the right operand may have side effects.

### Ternary Operator

For `cond ? a : b`, `EmitScalarExpr` calls `VisitConditionalOperator()`. When both operands are simple (no side effects, no volatile loads), the optimizer can later transform the branch-based pattern to `select`. At `-O0`, Clang always emits the branch pattern:

```llvm
; int ternary(int a, int b, bool cond) { return cond ? a : b; }
  br i1 %9, label %10, label %12

10:
  %11 = load i32, ptr %4, align 4   ; a
  br label %14

12:
  %13 = load i32, ptr %5, align 4   ; b
  br label %14

14:
  %15 = phi i32 [ %11, %10 ], [ %13, %12 ]
  ret i32 %15
```

The GNU omitted-middle ternary `x ?: fallback` avoids re-evaluating `x`. Clang evaluates `x` once, checks it for truth with `icmp ne`, and uses the original value for the true-branch phi argument:

```llvm
; int gnu_ternary(int x) { return x ?: 42; }
  %3 = load i32, ptr %2, align 4    ; x (evaluated once)
  %4 = icmp ne i32 %3, 0
  br i1 %4, label %5, label %6

5: br label %7
6: br label %7

7:
  %8 = phi i32 [ %3, %5 ], [ 42, %6 ]   ; use original %3, not a re-load
```

### Spaceship Operator

The C++20 three-way comparison `<=>` is lowered by `VisitBinaryOperator` for built-in types. For integer operands, Clang emits a sequence of comparisons that computes the `std::strong_ordering` value as a `i8` (`-1`, `0`, or `1`). The `std::strong_ordering` type is defined with a single `i8` member tagged with the comparison category. The lowering uses `icmp slt` and `icmp sgt` in sequence, materializing `-1`, `0`, or `1` via GEP into the ordering struct.

---

## Cast Lowering

### ScalarExprEmitter::VisitCastExpr

Cast expressions are dispatched via `VisitCastExpr()` in `CGExprScalar.cpp`. The `CastKind` enum selects the appropriate IR instruction.

**`CK_LValueToRValue`** materializes an lvalue: calls `EmitLoadOfScalar()` which calls `Builder.CreateLoad()` with the appropriate type and alignment, plus any volatile or atomic load flags.

**Integral casts** (`CK_IntegralCast`): `Builder.CreateIntCast()` selects `sext` (sign extend), `zext` (zero extend), or `trunc` based on whether the cast is widening to a signed type, widening to an unsigned type, or narrowing. Narrowing casts are undefined behavior for out-of-range values in C++ and generate a direct `trunc`.

**Floating-point casts** (`CK_FloatingCast`): `Builder.CreateFPCast()` selects `fpext` (float → double) or `fptrunc` (double → float).

**`CK_IntegralToFloating`** / **`CK_FloatingToIntegral`**: `sitofp`/`uitofp` and `fptosi`/`fptoui` respectively. Floating-to-integer conversion for out-of-range values is undefined behavior, and without `-fsanitize=float-cast-overflow` no check is emitted.

**Pointer-integer interconversion**: `CK_PointerToIntegral` → `ptrtoint`; `CK_IntegralToPointer` → `inttoptr`. Both require that the integer type is the same width as a pointer on the target.

```llvm
; void* vp = &i;
store ptr %1, ptr %5, align 8    ; CK_BitCast (identity, ptr stays ptr)

; long li = (long)vp;
%15 = load ptr, ptr %5, align 8
%16 = ptrtoint ptr %15 to i64    ; CK_PointerToIntegral

; int* ip = (int*)li;
%17 = load i64, ptr %6, align 8
%18 = inttoptr i64 %17 to ptr   ; CK_IntegralToPointer
```

**`CK_BitCast`**: `Builder.CreateBitCast()` or (in opaque-pointer mode) a GEP or identity pointer. In LLVM 22's opaque-pointer IR, `bitcast ptr to ptr` is a no-op and is elided; type information lives in load/store operands rather than pointer types.

**`CK_DerivedToBase`** and **`CK_BaseToDerived`**: Both use `ScalarExprEmitter::EmitPointerWithAlignment()` followed by a GEP to adjust the pointer by the base class offset within the derived layout. The offset is obtained from `CodeGenTypes::getCXXRecordLayout()`.

**`CK_Dynamic`**: Dynamic casts emit a call to `__dynamic_cast()` (the Itanium ABI runtime function). For `dynamic_cast<Derived*>(base_ptr)`, Clang passes the source pointer, the source `std::type_info`, the destination `std::type_info`, and the hint offset. A null return triggers UB or exception depending on whether the cast is to a pointer or reference.

---

## LValue Emission

### EmitLValue Dispatch

`EmitLValue()` in `CGExpr.cpp` produces an `LValue` — a wrapper around an address plus type, alignment, TBAA tags, and volatile/restrict qualifiers. It dispatches on `Expr::getStmtClass()`.

The `LValue` type carries several properties beyond a bare address:
- **BaseInfo**: TBAA base type information for alias analysis. Every field access appends a TBAA access node.
- **Alignment**: the known alignment at the use site. A `char*` derived from a `struct` pointer retains the struct's alignment.
- **Volatile**: if the underlying type is `volatile`, loads and stores gain `volatile` semantics in LLVM IR.
- **GlobalReg**: for global register variables (`register int x asm("rsp")`), the lvalue is actually a call to `llvm.read_register` / `llvm.write_register` rather than a memory address.

**`DeclRefExpr`**: Looks up the declaration in `LocalDeclMap` (for locals/parameters) or calls `CGM.GetAddrOfGlobalVar()` (for file-scope variables). For reference types, the lvalue is the address stored in the reference variable, then loaded.

For a function parameter of type `int`, the parameter's alloca address is in `LocalDeclMap`. For a function parameter of type `const int&`, `LocalDeclMap` maps to the alloca that holds the reference value (a pointer), and `EmitLValue` loads that pointer to get the address of the referenced integer.

**`MemberExpr`**: `EmitLValueForField()` computes the field's GEP from the struct base. For a non-bitfield at offset 32 bytes in a struct, this becomes `getelementptr inbounds %S, ptr %base, i32 0, i32 N` where `N` is the field index in the struct layout.

**Bit-field members**: `EmitLValueForBitField()` returns a special `LValue` with a `BitFieldInfo` payload. Subsequent loads and stores use `EmitLoadOfBitfieldLValue()` and `EmitStoreToBitfieldLValue()`, which emit shift-and-mask sequences:

```llvm
; int get_bitfield(Bits& b) { return b.a; }  // a: bits [0:2]
%4 = load i16, ptr %3, align 4
%5 = shl i16 %4, 13          ; shift left to discard upper bits
%6 = ashr i16 %5, 13         ; arithmetic shift right to sign-extend
%7 = sext i16 %6 to i32
```

For a store to field `b` (bits [3:7]):
```llvm
%9 = and i16 %7, 31           ; mask to 5 bits
%10 = shl i16 %9, 3           ; position at bit offset 3
%11 = and i16 %8, -249        ; clear field in existing word
%12 = or i16 %11, %10         ; merge
store i16 %12, ptr %6, align 4
```

**`ArraySubscriptExpr`**: `EmitArraySubscriptExpr()` calls `EmitLValue` on the base and `EmitScalarExpr` on the index, then emits a GEP: `getelementptr inbounds T, ptr %base, i64 %idx`.

**`UnaryOp(Deref)`**: A dereference `*p` is an lvalue — the address is the value of `p` itself. `EmitLValue` returns an `LValue` whose address is the scalar value of `p`, without emitting a load.

**`StringLiteral`**: Creates or reuses a `@.str` constant global with `private unnamed_addr` linkage. The lvalue address is the global's address.

**`CompoundLiteralExpr`**: For function scope, emits an `alloca` and initializes it; the lvalue is the alloca address.

---

## Aggregate Expressions

### AggExprEmitter::Visit

`AggExprEmitter` handles any expression whose type requires memory to represent: structs, unions, arrays, and non-trivially copyable class types. It is constructed with an `AggValueSlot` specifying the destination.

**Trivial POD initialization**: For zero initialization, emit `Builder.CreateMemSet(dest, 0, size)`. For constant aggregate initialization, Clang often creates a private constant global and emits `memcpy` into the destination:

```llvm
; Point p = {1, 2, 3};
@__const._Z10make_pointv.p = private unnamed_addr constant
    %struct.Point { i32 1, i32 2, i32 3 }, align 4

call void @llvm.memcpy.p0.p0.i64(
    ptr align 4 %1, ptr align 4 @__const..., i64 12, i1 false)
```

This is `EmitAggCopy()` — for trivially-copyable types, a `memcpy` intrinsic call is emitted rather than field-by-field loads and stores.

**Non-trivial copy**: When the type has a non-trivial copy constructor, `AggExprEmitter::VisitCXXConstructExpr()` calls `EmitCXXConstructorCall()` which routes to the copy constructor. No `memcpy` is emitted; the constructor handles the copy semantics.

**InitListExpr**: `VisitInitListExpr()` iterates over the initializer list. For each element, it obtains a GEP into the destination and recurses, calling `EmitAggExpr` or `EmitStoreThroughLValue` as appropriate.

**Designated initializers** (C99/C++20): Clang resolves designators during Sema to produce an `InitListExpr` with elements in declaration order, with zero-initializers inserted for unspecified fields. The codegen then proceeds as a regular `InitListExpr`:

```llvm
; return {.y = 5, .x = 3, .z = 7};
%3 = getelementptr inbounds nuw %struct.Point, ptr %1, i32 0, i32 0
store i32 3, ptr %3, align 4    ; x (declaration order)
%4 = getelementptr inbounds nuw %struct.Point, ptr %1, i32 0, i32 1
store i32 5, ptr %4, align 4    ; y
%5 = getelementptr inbounds nuw %struct.Point, ptr %1, i32 0, i32 2
store i32 7, ptr %5, align 4    ; z
```

**`CXXParenListInitExpr`**: The C++20 parenthesized initialization `T(a, b, c)` of aggregates is represented as a `CXXParenListInitExpr`. Codegen dispatches to `VisitCXXParenListInitExpr()`, which mirrors the behavior of `VisitInitListExpr` but allows conversion sequences for individual initializers.

### Union Initialization

For a union initialized to a specific member, `AggExprEmitter::VisitInitListExpr` selects the designated field (or the first named field for default initialization), zeroes the entire union storage, then overlays the selected member's value. Zero-initialization of the padding is important for reliable equality comparison of union values. If the member being initialized is a non-trivial class type, the appropriate constructor is called instead of a store.

### Trivial vs Non-Trivial Aggregates

The threshold between `memcpy` and constructor-call is `QualType::isNonTrivialToPrimitiveCopy()`. Types that are trivially copyable (no user-defined copy constructor, no virtual functions, no base class copy constructors) use `memcpy` via `EmitAggregateCopy()`. The `llvm.memcpy` is emitted with the aggregate's alignment. When the source and destination may alias, `EmitAggregateCopy` calls `memmove` (`llvm.memmove`) instead.

---

## Complex Numbers

### ComplexExprEmitter Layout and Arithmetic

C99 complex numbers are represented as a pair of real and imaginary floating-point values. Clang passes and returns `_Complex double` as `{ double, double }` in the IR at `-O0`. The real part occupies field 0 and the imaginary part field 1, reflecting the `{real, imag}` layout in memory.

`ComplexExprEmitter` maintains a `ComplexPairTy` typedef `std::pair<llvm::Value*, llvm::Value*>`. All operations produce and consume this pair directly.

**Addition/subtraction**: Component-wise `fadd`/`fsub`:

```llvm
; _Complex double add_complex(a, b)
%20 = fadd double %13, %17   ; real parts
%21 = fadd double %15, %19   ; imaginary parts
```

**Multiplication**: Standard complex multiplication `(a+bi)(c+di) = (ac-bd) + (ad+bc)i` emits four multiplications and two additions/subtractions at `-O0`. No attempt is made to use FMA unless the optimizer applies it.

**Division**: Complex division is not handled inline because of the numerical subtleties (avoiding overflow/underflow in the denominator). Clang emits a call to `__divdc3` (from `compiler-rt` or `libgcc`), which implements Smith's algorithm for numerically stable complex division:

```llvm
%20 = call noundef { double, double } @__divdc3(
    double noundef %13, double noundef %15,   ; numerator
    double noundef %17, double noundef %19)   ; denominator
```

Similarly `__divsc3` for `float` and `__divtc3` for `long double`.

**`__real__` and `__imag__`**: GNU extensions that extract components. `__real__ z` emits a GEP to field 0 followed by a load. `__imag__ z` uses field 1. When used as lvalues (`__real__ z = 3.0`), `EmitLValue` returns an `LValue` pointing directly at the component within the struct.

**`__builtin_complex(r, i)`**: Creates a complex value from two reals. Codegen assembles the pair without memory allocation.

### Complex Multiplication and Compiler Flags

Standard complex multiplication `(a+bi)(c+di) = (ac-bd) + (ad+bc)i` ignores the mathematical convention for handling infinity and NaN (for example, `(Inf + 0i) * (1 + 0i)` should yield `Inf + NaN*i` per the C standard, not `Inf + 0i` as the naive formula gives). At `-O0`, Clang uses the naive four-multiply formula. At `-O2` with `-fcomplex-arithmetic=improved` or `-fcx-limited-range`, Clang may call `__muldc3` (the `compiler-rt` complex multiplication function that handles infinities). The default at `-O2` without these flags uses the full `__muldc3` for multiplication as well, matching the C99 Annex G requirements for `CX_LIMITED_RANGE` pragma default-off behavior. This is an area where correctness and performance trade-offs are explicit in the compiler flags.

---

## C++ Expression Lowering

### CXXConstructExpr

`EmitCXXConstructExpr()` is called with an `AggValueSlot` pointing at the memory to construct. For a local variable `Widget w(42)`:

1. The alloca for `w` was created during `EmitAutoVarDecl`.
2. `EmitCXXConstructorCall()` is called with a `CXXConstructorDecl*` and the slot address.
3. A `call` (or `invoke` if inside an EH scope) to the mangled constructor name is emitted.

The Itanium ABI distinguishes three constructor variants: `C1` (complete object constructor, called when constructing the most-derived object), `C2` (base subobject constructor, called when constructing a base class subobject), and `C3` (complete object allocating constructor, rarely used). `EmitCXXConstructorCall` selects `Ctor_Complete` or `Ctor_Base` depending on whether the construction is for a most-derived object. At `-O0` these are separate mangled symbols; at `-O2` `C1` and `C2` are often identical and may be coalesced by the linker via comdat.

The `AggValueSlot::IsZeroed` flag allows `EmitCXXConstructExpr` to elide zero-initialization of POD members in the constructor body if the storage is already zero. This interacts with `calloc`-based allocation and static storage initialization.

### CXXNewExpr: operator new + constructor

`EmitCXXNewExpr()` handles `new Widget(v)`:

1. Call `_Znwm` (global `operator new(size_t)`) to allocate memory.
2. Emit the constructor via `invoke` since constructor exceptions must free the freshly allocated memory.
3. An `unwind` landing pad calls `_ZdlPvm` (sized `operator delete`) to release the memory before re-propagating the exception.

```llvm
%5 = call noalias noundef nonnull ptr @_Znwm(i64 noundef 4) #6
invoke void @_ZN6WidgetC2Ei(ptr %5, i32 %6)
    to label %7 unwind label %8

8:                                ; constructor threw
  %9 = landingpad { ptr, i32 } cleanup
  call void @_ZdlPvm(ptr %5, i64 4)   ; free before re-throw
  br label %12
```

### CXXDeleteExpr: destructor + operator delete

`EmitCXXDeleteExpr()` for `delete w` generates a null check (since `delete nullptr` is a no-op), then:

1. Call the destructor `_ZN6WidgetD2Ev` (base destructor variant).
2. Call `_ZdlPvm(ptr, size)` (sized deallocation, C++14).

```llvm
%4 = icmp eq ptr %3, null
br i1 %4, label %6, label %5

5:
  call void @_ZN6WidgetD2Ev(ptr %3)
  call void @_ZdlPvm(ptr %3, i64 4)
  br label %6
```

For arrays, `delete[]` calls the array destructor loop followed by `_ZdaPv`.

### CXXThrowExpr

`EmitCXXThrowExpr()` for `throw std::runtime_error("test")`:

1. `__cxa_allocate_exception(sizeof(T))` — allocates the exception object on the exception heap.
2. Construct the exception object via `invoke` of the exception type's constructor.
3. `__cxa_throw(exception_ptr, type_info_ptr, destructor_ptr)` — never returns; marked `noreturn`.

```llvm
%3 = call ptr @__cxa_allocate_exception(i64 16)
invoke void @_ZNSt13runtime_errorC1EPKc(ptr %3, ptr @.str)
    to label %4 unwind label %5

4:
  call void @__cxa_throw(ptr %3,
      ptr @_ZTISt13runtime_error,
      ptr @_ZNSt13runtime_errorD1Ev)   ; destructor for EH
  unreachable
```

### LambdaExpr: Capture Struct Construction

A lambda expression `[x, y](int z) { return x + y + z; }` is lowered by `EmitLambdaExpr()`:

1. An alloca is emitted for the closure type (a compiler-generated struct `%class.anon`).
2. For each captured variable by value, `EmitLValueForCapture()` stores the current value of `x` and `y` into the corresponding fields of the closure struct.
3. The result lvalue (or return value if the lambda is immediately used) is the closure's address.

```llvm
define dso_local i64 @_Z14capture_by_valii(i32 %0, i32 %1) {
  %3 = alloca %class.anon, align 4    ; closure struct
  %6 = getelementptr inbounds nuw %class.anon, ptr %3, i32 0, i32 0
  store i32 %7, ptr %6, align 4       ; capture x
  %8 = getelementptr inbounds nuw %class.anon, ptr %3, i32 0, i32 1
  store i32 %9, ptr %8, align 4       ; capture y
  %10 = load i64, ptr %3, align 4     ; return closure by value
  ret i64 %10
}
```

For by-reference capture `[&x]`, the closure struct stores a pointer to `x`, and `EmitLValueForCapture()` stores the address of `x` rather than loading its value.

### MaterializeTemporaryExpr and Lifetime Extension

`MaterializeTemporaryExpr` wraps an rvalue that must be given a stable address. `EmitMaterializeTemporaryExpr()` allocates a temporary on the stack (`CreateTempAlloca`) and initializes it. If the temporary is lifetime-extended (bound to a `const` reference or rvalue reference), it is placed in a named alloca for the reference's scope.

`ExprWithCleanups` wraps an expression that creates temporaries needing destruction. When `EmitScalarExpr` or `EmitAggExpr` encounters `ExprWithCleanups`, it pushes a cleanup scope via `RunCleanupsScope` before evaluating the sub-expression, so that temporaries' destructors are registered in order.

`CXXBindTemporaryExpr` registers a cleanup to call the temporary's destructor. The cleanup is pushed onto the `EHScopeStack` as a `DestroyTemporary` cleanup object, which will be popped and executed at the end of the full expression.

The combined effect for `RAII().get()`:

```llvm
define dso_local noundef i32 @_Z13use_temporaryv() personality ptr @__gxx_personality_v0 {
  %1 = alloca %struct.RAII, align 1   ; materialized temporary
  call void @_ZN4RAIIC2Ev(ptr %1)    ; construct
  %4 = invoke noundef i32 @_ZNK4RAII3getEv(ptr %1)
           to label %5 unwind label %6

5:
  call void @_ZN4RAIID2Ev(ptr %1)    ; destroy on normal path
  ret i32 %4

6:
  %7 = landingpad { ptr, i32 } cleanup
  call void @_ZN4RAIID2Ev(ptr %1)    ; destroy on exception path
  resume { ptr, i32 } ...
}
```

---

## Cleanups and Exception Handling in Codegen

### EHScopeStack Architecture

`CodeGenFunction::EHScopeStack` in [`clang/lib/CodeGen/EHScopeStack.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/EHScopeStack.h) is a stack of scopes, each scope a polymorphic `Cleanup` object. Scopes form a linked list; `stable_iterator` is a persistent position handle independent of stack reallocation.

Cleanup types:
- `NormalCleanup`: runs on normal (non-exception) exit from a scope.
- `EHCleanup`: runs when an exception unwinds through the scope.
- `NormalAndEHCleanup`: runs on both paths (the common case for destructors).

`pushCleanup<NormalAndEHCleanup>()` is a variadic template that constructs a cleanup object directly in the scope stack's arena. `pushDestroyCleanup()` is a convenience wrapper for destructors, pushing a `CallDestructorCleanup` object.

### RunCleanupsScope

`RunCleanupsScope` is an RAII guard. Its constructor calls `CGF.EHStack.stable_begin()` to record the current stack depth. Its destructor calls `CGF.PopCleanupBlocks(SavedDepth)`, which emits cleanup code for every scope pushed since the guard was created. This is used around the body of loop iterations, compound statements, and full expressions.

### invoke vs call Selection

When an expression or statement might throw — i.e., the `EHScopeStack` has any EH cleanup scopes active — `CodeGenFunction::EmitCallOrInvoke()` selects `invoke` instead of `call`. The `invoke` provides an `unwind` label pointing at the innermost EH landing pad. If no EH cleanups are active, a plain `call` is emitted.

The landing pad block is created lazily by `EmitLandingPad()`. When the landing pad is first needed, it emits `landingpad { ptr, i32 } cleanup` and chains cleanup calls. Each successive EH scope adds to the landing pad's list of clauses.

The selection logic is: `EHScopeStack.requiresLandingPad()` returns `true` if any scope with EH cleanups or catch clauses is active. When `true`, `EmitCallOrInvoke()` calls `EmitLandingPad()` to obtain the current landing pad block and uses `IRBuilder::CreateInvoke()` with that block as the unwind destination. When `false`, `IRBuilder::CreateCall()` is used directly.

A function compiled with `-fno-exceptions` never generates `invoke`; `EHScopeStack` still exists but `requiresLandingPad()` always returns `false`. Destructor cleanups for `-fno-exceptions` code still execute on normal exit paths via `NormalCleanup` scopes; they simply have no exception path. This is why destructors run correctly in `-fno-exceptions` code even without EH.

### EmitReturnStmt and Cleanup Chains

`EmitReturnStmt()` stores the return expression into the `ReturnValue` alloca, then calls `EmitBranchThroughCleanup(ReturnBlock)`. `EmitBranchThroughCleanup` walks from the current scope depth back to function scope, emitting each cleanup in sequence. Only one copy of each cleanup exists per scope exit path; multiple `return` statements that share a cleanup path branch to the same cleanup block.

This means the LLVM IR for a function with multiple early returns and a destructor contains one destructor-call block that is shared by all return paths:

```
entry → ... → %cleanup (destructor call) → %return_block → ret
                     ↑                           ↑
              return path 1            return path 2
```

---

## Vector and Matrix Expressions

### Vector Type Element Access

GCC/Clang vector types (`__attribute__((vector_size(N)))`) are lowered to LLVM `<N x T>` vector types. Element access `v[i]` maps to `extractelement` for reads and `insertelement` for writes:

```llvm
; int extract_elem(v4i32 v) { return v[2]; }
%4 = extractelement <4 x i32> %3, i32 2
```

### ExtVectorElementExpr and Swizzle

OpenCL-style `ext_vector_type` vectors support swizzle access (`v.wzyx`). `EmitExtVectorElementExpr()` translates the component selector string to a `shufflevector` mask:

```llvm
; float4 swizzle(float4 v) { return v.wzyx; }
%4 = shufflevector <4 x float> %3, <4 x float> poison,
     <4 x i32> <i32 3, i32 2, i32 1, i32 0>
```

The second operand is `poison` because no elements from a second vector are selected. Swizzles that select from two vectors (as in `__builtin_shufflevector(a, b, 0, 5, 2, 7)`) use both source operands:

```llvm
%7 = shufflevector <4 x float> %5, <4 x float> %6,
     <4 x i32> <i32 0, i32 5, i32 2, i32 7>
```

### Matrix Type Subscript

The `__attribute__((matrix_type(R, C)))` extension adds first-class matrix support. A matrix `float __attribute__((matrix_type(4, 4)))` is represented as `<16 x float>` in column-major order. Subscript `m[r][c]` lowers to an `extractelement` at index `r + c * num_rows`. Clang emits `llvm.matrix.transpose`, `llvm.matrix.multiply`, and `llvm.matrix.column.major.load`/`.store` intrinsics for the respective matrix operations.

For example, `matrix_multiply(A, B)` on two `4x4 float` matrices emits:

```llvm
%result = call <16 x float> @llvm.matrix.multiply.v16f32.v16f32.v16f32(
    <16 x float> %A, <16 x float> %B,
    i32 4,   ; lhs rows
    i32 4,   ; lhs columns / rhs rows
    i32 4)   ; rhs columns
```

Matrix load/store intrinsics carry the stride parameter to handle strided memory layouts:

```llvm
%m = call <16 x float> @llvm.matrix.column.major.load.v16f32.i64(
    ptr %src, i64 %stride, i1 false,   ; column stride, not transposed
    i32 4, i32 4)                      ; rows, columns
```

The `LowerMatrixIntrinsics` pass in the middle-end expands these intrinsics into actual vector operations: the multiply becomes a sequence of vector multiply-accumulate operations, exploiting the SIMD capabilities of the target. By representing matrix operations as intrinsics in the IR, passes like loop vectorization can see across the matrix boundary.

Vector element insertion uses `insertelement`. For `v[2] = 42` on a `v4i32`:

```llvm
%old = load <4 x i32>, ptr %v, align 16
%new = insertelement <4 x i32> %old, i32 42, i32 2
store <4 x i32> %new, ptr %v, align 16
```

The load-modify-store sequence is at `-O0`. Optimizers can simplify this when the vector is SSA — at `-O1` and above, `mem2reg` promotes the alloca to an SSA vector value, eliminating the load and store for many patterns.

---

## Chapter Summary

- `CodeGenFunction::EmitStmt()` is a flat switch dispatching to `EmitIfStmt`, `EmitWhileStmt`, `EmitForStmt`, `EmitSwitchStmt`, `EmitReturnStmt`, and their siblings in `CGStmt.cpp`.
- Loop codegen produces four-block structure: preheader → condition → body → latch/increment → exit; loops carry `!llvm.loop` metadata on the back edge.
- `BreakContinueStack` and `JumpDest` track break/continue targets through nested scopes; `EmitBranchThroughCleanup` handles the cleanup unwinding.
- Indirect goto uses `llvm::BlockAddress` constants and the `indirectbr` instruction with an exhaustive successor list.
- The three expression emitter classes — `ScalarExprEmitter`, `ComplexExprEmitter`, `AggExprEmitter` — each consume `llvm::Value*` and produce their respective result types; `AggValueSlot` carries the in-place construction destination.
- Signed integer arithmetic gains `nsw` flags by default; `-fsanitize=signed-integer-overflow` replaces bare arithmetic with `llvm.sadd.with.overflow` and a `__ubsan_handle_*` call on the slow path.
- `&&`/`||` short-circuit via conditional branches and `phi` nodes; ternary and GNU ternary also use branch+phi at `-O0`.
- Cast lowering maps `CastKind` variants to `sext`/`zext`/`trunc`, `sitofp`/`fpext`/`fptrunc`, `ptrtoint`/`inttoptr`, GEP base adjustments for class hierarchy casts, and a `__dynamic_cast` call for `CK_Dynamic`.
- Bit-field loads produce shift-and-mask sequences; stores read-modify-write the containing integer word.
- Complex division delegates to `__divdc3`/`__divsc3` for numerical stability; addition and multiplication are inline.
- `CXXNewExpr` wraps allocation and constructor in `invoke`+unwind-cleanup; `CXXThrowExpr` calls `__cxa_allocate_exception` then `__cxa_throw`.
- Lambda capture-by-value stores copies into closure struct fields; capture-by-reference stores addresses.
- `EHScopeStack` manages cleanup scopes; `invoke`/`call` selection depends on whether EH cleanups are active; `EmitReturnStmt` branches through cleanup before reaching `ret`.
- Vector swizzle uses `shufflevector`; matrix operations use `llvm.matrix.*` intrinsics on flat column-major vectors.

---

**Cross-references:**
- [Chapter 39 — CodeGenModule and CodeGenFunction](ch39-codegenmodule-and-codegenfunction.md) — establishes `CodeGenFunction`, `LocalDeclMap`, `EHScopeStack`, and `StartFunction` infrastructure used throughout this chapter.
- [Chapter 41 — Calls, the ABI Boundary, and Builtins](ch41-calls-abi-boundary-and-builtins.md) — covers how calls are lowered once argument marshalling is applied on top of the expression machinery shown here.
- [Chapter 19 — Instructions I: Arithmetic and Memory](../part-04-llvm-ir/ch19-instructions-arithmetic-memory.md) — defines the IR instructions (`add nsw`, `sdiv`, `load`, `store`, `getelementptr`) generated by the expression emitters.
- [Chapter 20 — Instructions II: Control Flow and Aggregates](../part-04-llvm-ir/ch20-instructions-control-flow-aggregates.md) — defines `br`, `switch`, `phi`, `select`, `indirectbr`, `landingpad`, and `resume` used by statement lowering.
- [Chapter 26 — Exception Handling](../part-04-llvm-ir/ch26-exception-handling.md) — the `landingpad`/`resume`/`invoke` IR model that EHScopeStack drives.

**Reference links:**
- [`clang/lib/CodeGen/CGStmt.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGStmt.cpp) — statement emission
- [`clang/lib/CodeGen/CGExprScalar.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGExprScalar.cpp) — `ScalarExprEmitter`
- [`clang/lib/CodeGen/CGExprComplex.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGExprComplex.cpp) — `ComplexExprEmitter`
- [`clang/lib/CodeGen/CGExprAgg.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGExprAgg.cpp) — `AggExprEmitter`
- [`clang/lib/CodeGen/CGExpr.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGExpr.cpp) — `EmitLValue`, `EmitLoadOfScalar`, `EmitStoreThroughLValue`
- [`clang/lib/CodeGen/CGCleanup.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGCleanup.cpp) — `EHScopeStack`, cleanup emission
- [`clang/lib/CodeGen/EHScopeStack.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/EHScopeStack.h) — `EHScopeStack` and `Cleanup` interface
- [`clang/lib/CodeGen/CGCXXExpr.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGCXXExpr.cpp) — C++ expression lowering
- [Smith, R.L. (1962). Algorithm 116: Complex division. *CACM* 5(8), 435](https://dl.acm.org/doi/10.1145/368637.368661) — the algorithm behind `__divdc3`


---

@copyright jreuben11
