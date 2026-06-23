# Chapter 53 — CIR Generation from AST

*Part VIII — ClangIR (CIR)*

CIR generation follows the same producer-consumer contract as classic CodeGen: the compiler driver installs an `ASTConsumer`, and the Sema/Parser pipeline pushes completed declarations to it. But where `CodeGenModule` immediately lowers each construct to LLVM IR, `CIRGenModule` builds an MLIR module filled with CIR operations, deferring all LLVM knowledge to the lowering pass. This chapter dissects the generator, shows how C and C++ constructs map to CIR operations, and explains the back-references to the AST that give CIR its analysis advantage.

## Table of Contents

- [53.1 Entry Points](#531-entry-points)
  - [53.1.1 CIRGenerator — the ASTConsumer](#5311-cirgenerator-the-astconsumer)
  - [53.1.2 CIRGenModule — the Module Builder](#5312-cirgenmodule-the-module-builder)
  - [53.1.3 CIRGenFunction — the Function Builder](#5313-cirgenfunction-the-function-builder)
- [53.2 Type Mapping: AST Types to CIR Types](#532-type-mapping-ast-types-to-cir-types)
- [53.3 Function Emission](#533-function-emission)
  - [53.3.1 `emitFunctionBody`](#5331-emitfunctionbody)
  - [53.3.2 Statement Emission](#5332-statement-emission)
  - [53.3.3 Expression Emission](#5333-expression-emission)
- [53.4 Variable Handling](#534-variable-handling)
  - [53.4.1 Locals and Automatic Storage](#5341-locals-and-automatic-storage)
  - [53.4.2 Globals](#5342-globals)
  - [53.4.3 The Local Variable Map](#5343-the-local-variable-map)
- [53.5 Function Calls](#535-function-calls)
  - [53.5.1 Direct Calls](#5351-direct-calls)
  - [53.5.2 Indirect Calls](#5352-indirect-calls)
  - [53.5.3 Virtual Calls](#5353-virtual-calls)
  - [53.5.4 ABI Argument Handling](#5354-abi-argument-handling)
- [53.6 Record (Struct/Union) Emission](#536-record-structunion-emission)
  - [53.6.1 Record Type Creation](#5361-record-type-creation)
  - [53.6.2 Member Access](#5362-member-access)
- [53.7 Back-References to the AST](#537-back-references-to-the-ast)
- [53.8 Exception Handling Emission](#538-exception-handling-emission)
- [53.9 Constants and Initializers](#539-constants-and-initializers)
- [53.10 Comparison with Classic CodeGenModule](#5310-comparison-with-classic-codegenmodule)
- [Chapter Summary](#chapter-summary)

---

## 53.1 Entry Points

### 53.1.1 CIRGenerator — the ASTConsumer

`CIRGenerator` (declared in
[`clang/include/clang/CIR/CIRGenerator.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CIR/CIRGenerator.h)) is a subclass of `clang::ASTConsumer`. It mirrors `CodeGenerator` in the classic path:

```cpp
class CIRGenerator : public clang::ASTConsumer {
protected:
  std::unique_ptr<mlir::MLIRContext>         mlirContext;
  std::unique_ptr<clang::CIRGen::CIRGenModule> cgm;
  // …
public:
  void Initialize(clang::ASTContext &astContext) override;
  bool HandleTopLevelDecl(clang::DeclGroupRef group) override;
  void HandleTranslationUnit(clang::ASTContext &astContext) override;
  void HandleInlineFunctionDefinition(clang::FunctionDecl *d) override;
  void HandleTagDeclDefinition(clang::TagDecl *d) override;
  void HandleCXXStaticMemberVarInstantiation(clang::VarDecl *d) override;
  void CompleteTentativeDefinition(clang::VarDecl *d) override;
  void HandleVTable(clang::CXXRecordDecl *rd) override;
  mlir::ModuleOp getModule() const;
};
```

`Initialize` creates the `mlir::MLIRContext`, registers the `cir` dialect and all its dependencies, and constructs `CIRGenModule`. `HandleTopLevelDecl` drives per-declaration generation; `HandleTranslationUnit` flushes deferred declarations and finalizes the module.

The `CIRGenAction` frontend action orchestrates this consumer alongside the MLIR context:

```cpp
class CIRGenAction : public clang::ASTFrontendAction {
  mlir::OwningOpRef<mlir::ModuleOp> MLIRMod;
  mlir::MLIRContext *MLIRCtx;
  // …
  std::unique_ptr<ASTConsumer>
  CreateASTConsumer(CompilerInstance &CI, StringRef InFile) override;
};
```

### 53.1.2 CIRGenModule — the Module Builder

`CIRGenModule` (abbreviated **CGM** in comments) is the analogue of `CodeGenModule`. It owns the MLIR module op and the `mlir::OpBuilder`, and it provides helpers for emitting functions, globals, types, and attributes. CGM's constructor builds the top-level `mlir::ModuleOp` and attaches data-layout and target-triple attributes, just as the classic CodeGenModule does.

Key CGM members:
```cpp
class CIRGenModule {
  mlir::ModuleOp   theModule;   // the top-level MLIR container
  mlir::OpBuilder  builder;     // insertion-point manager
  clang::ASTContext  &astCtx;
  const clang::CodeGenOptions &codeGenOpts;
  const clang::TargetInfo     &target;
  // type cache, deferred list, vtable builder, …
};
```

### 53.1.3 CIRGenFunction — the Function Builder

`CIRGenFunction` (**CGF**) is instantiated once per function body and owns the insertion state for that function. Its structure parallels `CodeGenFunction`:

```cpp
class CIRGenFunction {
  CIRGenModule &cgm;
  cir::FuncOp  curFn;        // the function op being built
  // local variable map: VarDecl* → cir.alloca address
  // label map, cleanup stack, EH scope, …
};
```

## 53.2 Type Mapping: AST Types to CIR Types

CIRGen maintains a type cache that translates `clang::QualType` values to MLIR types in the CIR dialect. The mapping is more conservative than the classic path — it preserves signedness, exact widths, and record structure:

| Clang type | CIR type |
|------------|----------|
| `int` (LP64) | `cir.int<s, 32>` |
| `unsigned int` | `cir.int<u, 32>` |
| `long` (LP64) | `cir.int<s, 64>` |
| `__int128` | `cir.int<s, 128>` |
| `bool` | `cir.bool` |
| `float` | `cir.float` |
| `double` | `cir.double` |
| `long double` | `cir.long_double` |
| `T*` | `cir.ptr<!cir.T>` |
| `T[N]` | `cir.array<!cir.T x N>` |
| `struct S { … }` | `cir.record<struct "S" {…}>` |
| `void` | `cir.void` |
| function `R(A, B)` | `cir.func<(!cir.A, !cir.B) -> !cir.R>` |

Notably, `bool` is a distinct CIR type rather than `i1`. The translation preserves `signed`/`unsigned` qualifiers in `cir.int`, enabling passes to reason about overflow behavior without querying a side table.

For `cir.record`, the generation is *lazy*: the record type is created when the `RecordDecl` is first seen, with its field types populated later if the definition is available. This supports forward declarations (opaque pointers) and recursive types (linked lists) without special-casing.

## 53.3 Function Emission

### 53.3.1 `emitFunctionBody`

When CGM handles a `FunctionDecl` with a body, it calls `CIRGenFunction::emitFunctionBody`:

1. **Create `cir.func` op** with the mangled name, CIR function type, and linkage attribute.
2. **Emit parameter allocas**: for each parameter, emit `cir.alloca` and `cir.store` to create a mutable stack slot (the *alloca-then-store* pattern, matching classic CodeGen).
3. **Emit body**: `emitCompoundStmt` recursively emits the function body.
4. **Append implicit return**: if control can fall off the end, emit `cir.return` with a zero-initialized value.

```
// Source: int add(int a, int b) { return a + b; }
cir.func @add(%arg0: !cir.int<s,32>, %arg1: !cir.int<s,32>) -> !cir.int<s,32> {
  %a_addr = cir.alloca !cir.int<s,32>, ["a"] {alignment = 4 : i64}
  %b_addr = cir.alloca !cir.int<s,32>, ["b"] {alignment = 4 : i64}
  cir.store %arg0, %a_addr : !cir.int<s,32>, !cir.ptr<!cir.int<s,32>>
  cir.store %arg1, %b_addr : !cir.int<s,32>, !cir.ptr<!cir.int<s,32>>
  %a = cir.load %a_addr : !cir.ptr<!cir.int<s,32>>, !cir.int<s,32>
  %b = cir.load %b_addr : !cir.ptr<!cir.int<s,32>>, !cir.int<s,32>
  %sum = cir.binop(add, %a, %b) nsw : !cir.int<s,32>
  cir.return %sum : !cir.int<s,32>
}
```

Note that `cir.alloca` carries the source name (`"a"`) — this is preserved through lowering into LLVM IR's `alloca` name, improving debuggability.

### 53.3.2 Statement Emission

Each C/C++ statement class is handled by a visitor in `CIRGenFunction`:

**`ReturnStmt`:**
```
cir.return %val : !cir.int<s,32>
```

**`IfStmt`:**
```
cir.if %cond : !cir.bool {
  // then region
} else {
  // else region (optional)
}
```
The `cir.if` op has two optional regions — CIR preserves the structured nature rather than flattening to basic blocks immediately.

**`ForStmt`:**
```
cir.for : {
  // condition region — must terminate with cir.condition
  %cond = ...
  cir.condition(%cond)
} body {
  // body region
  cir.yield continue
} step {
  // increment region
  cir.yield
}
```

**`WhileStmt`:**
```
cir.while : {
  %cond = ...
  cir.condition(%cond)
} do {
  // body
  cir.yield
}
```

**`DoStmt`:**
```
cir.do {
  // body
  cir.yield
} while : {
  %cond = ...
  cir.condition(%cond)
}
```

**`SwitchStmt`:**
```
cir.switch (%n : !cir.int<s,32>) {
  case (equal, 0) {
    // case 0
    cir.yield fallthrough
  }
  case (equal, 1) {
    // case 1
    cir.break
  }
  case (default) {
    // default
    cir.break
  }
}
```

The `fallthrough` yield models C switch fall-through explicitly, enabling analysis passes to detect or eliminate it without reconstructing the control flow.

### 53.3.3 Expression Emission

Expression emission follows the same LValue/RValue split as the classic path:

- **`emitLValue(Expr *)`**: produces a `cir.ptr<T>` pointing to the storage location.
- **`emitRValue(Expr *)`**: produces a scalar value of the CIR type.

**Binary arithmetic** (`BinaryOperator`):

```cpp
// Source: a + b (signed int)
auto lhs = emitRValue(binop->getLHS());
auto rhs = emitRValue(binop->getRHS());
return builder.create<cir::BinOp>(loc, cir::BinOpKind::Add,
                                  lhs, rhs, overflowBehavior);
```

The `overflowBehavior` encodes `nsw`/`nuw`/`saturated` based on the C++ expression type. `unsigned int` addition emits `nuw`; `signed int` addition in C++ emits `nsw` (since signed overflow is UB).

**Comparison operators** produce `cir.bool`:

```
%cmp = cir.cmp(lt, %a, %b) : !cir.int<s,32>, !cir.bool
```

**Unary operators:**
```
%neg = cir.unary(minus, %a) nsw : !cir.int<s,32>
%not = cir.unary(not,   %b) : !cir.bool
%inc = cir.unary(inc,   %c) nsw : !cir.int<s,32>
```

**Cast expressions:**
```
// (long)i — integral widening
%wide = cir.cast(integral, %i : !cir.int<s,32>) : !cir.int<s,64>

// (bool)n — integer to bool
%b = cir.cast(int_to_bool, %n : !cir.int<s,32>) : !cir.bool

// Array decay: int arr[4] → int*
%p = cir.cast(array_to_ptrdecay, %arr : !cir.ptr<!cir.array<!cir.int<s,32> x 4>>)
           : !cir.ptr<!cir.int<s,32>>
```

## 53.4 Variable Handling

### 53.4.1 Locals and Automatic Storage

Every local variable with automatic storage duration is materialized as a `cir.alloca`. The alloca is placed in the *current scope*, not necessarily at function entry — this is a key difference from classic CodeGen, which always emits allocas at the function entry block.

```
cir.scope {
  // Block: { int x = 5; }
  %x_addr = cir.alloca !cir.int<s,32>, init, ["x"] {alignment = 4 : i64}
  %five = cir.const #cir.int<5> : !cir.int<s,32>
  cir.store %five, %x_addr : !cir.int<s,32>, !cir.ptr<!cir.int<s,32>>
  // … x used here …
}
// x is out of scope; destructor call emitted before cir.scope exit (C++)
```

The `cir.scope` op models a C++ block scope. Destructors for objects with non-trivial destructors are inserted as `cir.call` ops just before the `cir.scope` terminator, in declaration-reverse order. This mirrors the `RunCleanupsScope` mechanism in classic CodeGen but at a higher level of abstraction.

### 53.4.2 Globals

Global variables are emitted as `cir.global` ops on the module:

```
cir.global external @g_counter : !cir.int<s,32> = #cir.int<0> : !cir.int<s,32>
cir.global internal @s_local : !cir.int<s,32>
cir.global external constant @kPi : !cir.double = #cir.fp<3.14159> : !cir.double
```

Thread-local variables and globals with non-trivial initializers (C++ `__cxa_atexit`-registered objects) are among the `MissingFeatures` entries, reflecting in-progress upstreaming.

### 53.4.3 The Local Variable Map

`CIRGenFunction` maintains a `DeclMap` (conceptually `DenseMap<const VarDecl*, mlir::Value>`) mapping each `VarDecl` to its `cir.alloca` result. This mirrors `CodeGenFunction::LocalDeclMap`. When a variable is referenced, CIRGen looks up its alloca address and emits a `cir.load`; assignment emits a `cir.store`.

## 53.5 Function Calls

### 53.5.1 Direct Calls

```
// Source: int r = add(a, b);
%r = cir.call @add(%a, %b) : (!cir.int<s,32>, !cir.int<s,32>) -> !cir.int<s,32>
```

`cir.call` carries the callee symbol reference or a function pointer value, the argument list, and the calling convention. For C++ member functions, the implicit `this` pointer is the first argument.

### 53.5.2 Indirect Calls

```
// Source: int (*fp)(int, int) = add; int r = fp(a, b);
%fp_val = cir.load %fp_addr : !cir.ptr<!cir.func<(!cir.int<s,32>, !cir.int<s,32>) -> !cir.int<s,32>>>,
                               !cir.func<(!cir.int<s,32>, !cir.int<s,32>) -> !cir.int<s,32>>
%r = cir.call %fp_val(%a, %b) : (!cir.int<s,32>, !cir.int<s,32>) -> !cir.int<s,32>
```

### 53.5.3 Virtual Calls

Virtual dispatch is handled with `cir.vtable_ptr` and `cir.virtual_call` ops (emitted by `CXXABILoweringPass` post-generation). At generation time, a virtual call emits a placeholder that records which virtual function is called and the object pointer; the ABI-lowering pass replaces it with the vtable load and indirect call sequence.

### 53.5.4 ABI Argument Handling

`ABIArgInfo` (declared in
[`clang/include/clang/CIR/ABIArgInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CIR/ABIArgInfo.h)) tracks how each argument is passed: directly in a CIR type, coerced to a different type, or ignored. This parallels the classic `CodeGen::ABIArgInfo`. As of LLVM 22.1, `Direct` and `Ignore` are implemented; the full set of ABI classification kinds (`Indirect`, `Expand`, `CoerceAndExpand`, `InAlloca`) are `MissingFeatures` entries.

## 53.6 Record (Struct/Union) Emission

### 53.6.1 Record Type Creation

When `CIRGenModule` encounters a `RecordDecl` (struct or union), it creates a `cir.record` type:

```
// Source: struct Point { int x, y; };
cir.record<struct "Point" {!cir.int<s,32>, !cir.int<s,32>}>
```

Unnamed bit-fields are preserved as `cir.int<u, N>` padding fields. The CIR record type tracks whether the source is a struct, union, or class.

### 53.6.2 Member Access

Struct member access (`MemberExpr`) is lowered via `cir.get_member`:

```
// Source: p.x (where p : struct Point)
%x_addr = cir.get_member %p_addr[0] : !cir.ptr<!cir.record<struct "Point" {...}>>
                                     -> !cir.ptr<!cir.int<s,32>>
%x = cir.load %x_addr : !cir.ptr<!cir.int<s,32>>, !cir.int<s,32>
```

The field index (0 for `x`, 1 for `y`) comes from the `FieldDecl`'s position in the `RecordDecl`. CIR's `cir.get_member` is analogous to LLVM's `getelementptr` but carries the record type, enabling analyses to reason about which field is accessed without walking GEP indices.

## 53.7 Back-References to the AST

One of CIR's differentiating properties is that operation attributes can hold back-references to AST nodes. For example:

- `cir.call` can carry a `FunctionDecl*` pointer, enabling later passes to query Clang attributes (`[[clang::noinline]]`, `[[nodiscard]]`) without re-examining the CIR text.
- `cir.alloca` carries the `VarDecl*` for local variables, enabling lifetime analysis to correlate allocas with declared variable scopes.
- `cir.record` types are linked to their `RecordDecl*`, allowing ABI classification to access field offsets and sizes through `ASTContext` rather than recomputing them from CIR.

These AST pointers are non-serializable — they exist only during in-process compilation. When CIR is serialized to a `.cir` file (via `-emit-cir`), the pointers are replaced with source-location annotations and mangled names.

## 53.8 Exception Handling Emission

When exceptions are enabled (`-fexceptions`), `try/catch` blocks are emitted as:

```
cir.try {
  // try body
  %r = cir.call @may_throw() : () -> !cir.int<s,32>
  cir.yield
} catch [
  #cir.catch_info<type_info @_ZTIi, "e_int">,
  #cir.catch_info<all, "e_all">
] {
  ^bb_int_handler:
    // catch (int e_int) { … }
    cir.yield
  ^bb_all_handler:
    // catch (...) { … }
    cir.yield
}
```

The `cir.try` op preserves the `try` region and the catch-type list as CIR-level attributes. The `CXXABILoweringPass` later emits the `__cxa_begin_catch`/`__cxa_end_catch` calls and the `landingpad` machinery, using the AST type information to generate correct type-info references.

## 53.9 Constants and Initializers

Constant folding at the CIR level uses the same `ConstantEmitter` concept as classic CodeGen, but produces MLIR attributes instead of `llvm::Constant*`:

```
// int arr[4] = {1, 2, 3, 4};
cir.global external @arr : !cir.array<!cir.int<s,32> x 4> =
  #cir.const_array<[#cir.int<1> : !cir.int<s,32>,
                    #cir.int<2> : !cir.int<s,32>,
                    #cir.int<3> : !cir.int<s,32>,
                    #cir.int<4> : !cir.int<s,32>]>
  : !cir.array<!cir.int<s,32> x 4>
```

String literals are `cir.const_array` of `cir.int<s,8>` (or `u,8`), with the null terminator included.

For complex numbers, CIR uses `cir.complex<cir.float>` types and `cir.complex.create`/`cir.complex.real`/`cir.complex.imag` ops, preserving the complex structure rather than immediately splitting into real and imaginary SSA values.

## 53.10 Comparison with Classic CodeGenModule

| Concern | Classic CodeGen | CIR Gen |
|---------|-----------------|---------|
| Module container | `llvm::Module` | `mlir::ModuleOp` |
| Builder | `llvm::IRBuilder<>` | `mlir::OpBuilder` (CIRBaseBuilderTy) |
| Function container | `llvm::Function` | `cir.func` |
| Basic block | `llvm::BasicBlock` | MLIR region/block (or structured ops) |
| Local var storage | `alloca` at entry | `cir.alloca` at scope |
| Struct field access | `getelementptr` | `cir.get_member` |
| Type representation | `llvm::Type*` | `mlir::Type` (CIR types) |
| Control flow | Immediate BB lowering | Structured ops, flattened later |
| Destructor calls | `RunCleanupsScope` | `cir.scope` cleanup region |
| AST back-refs | None (discarded) | Preserved as op attributes |

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **Complete ABI argument-passing coverage**: The remaining `ABIArgInfo` kinds — `Indirect`, `Expand`, `CoerceAndExpand`, and `InAlloca` — are tracked as `MissingFeatures` entries in `CIRGenCall.cpp`; completing them is a prerequisite for upstreaming non-trivial C ABI call conventions (e.g., SysV AMD64 struct splitting, AAPCS homogeneous aggregates). Active patch series are being reviewed on [LLVM Discourse / ClangIR Working Group](https://discourse.llvm.org/c/clangir/58).
- **Thread-local storage (TLS) emission**: `cir.global` with TLS models (`__thread`, `thread_local`) is listed as a near-term milestone in the ClangIR upstreaming plan. Emission must integrate with ELF `.tbss`/`.tdata` section handling through `CXXABILoweringPass`.
- **Non-trivial global initializers and `__cxa_atexit`**: C++ globals with non-trivial constructors require emitting `cir.global_ctor` regions and registering teardown via `__cxa_atexit`; this is the primary remaining gap blocking real-world C++ translation-unit parity.
- **`cir.switch` range-case and string-switch lowering**: Extending `cir.switch` to model compiler-synthesized range cases (used by Clang's `EmitSwitchStmt` for dense integer sets) and `llvm.memcmp`-based string switches, enabling CIR-level jump-table optimizations before LLVM lowering.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **Full C++23/C++26 AST → CIR coverage**: Emission for C++23 features (explicit object parameters, `std::expected`, `if consteval`) and prospective C++26 features (pattern matching via `inspect`, pack indexing) must be added to `CIRGenFunction`; the structured-op model of CIR is well-suited to express `inspect` alternatives as multi-region ops analogous to `cir.switch`.
- **CIR serialization round-trip with AST cross-references**: The current in-process AST pointer scheme (§53.7) precludes offline/distributed compilation. A planned attribute scheme using `SourceLocation` + mangled-name tokens as a serializable surrogate, combined with re-parsed AST stubs, would enable distributing `.cir` artifacts across build clusters.
- **Lifetime analysis and borrow-checking at CIR level**: Because `cir.alloca` retains `VarDecl*` links and `cir.scope` models lexical scopes, CIR is a candidate for a C++ lifetime-safety analysis layer analogous to the Rust borrow checker — tracked in the `[[clang::lifetime_capture_by]]`/Lifetime Safety Profile work originated by Herb Sutter and now revived in the C++ Safety Study Group (SG23).
- **CIR-to-CIR inlining and interprocedural passes**: Implementing an inliner operating on `cir.func` ops before LLVM lowering could expose C++-aware optimizations (e.g., `[[likely]]`-annotated branch specialization, `nodiscard` devirtualization) that are currently lost once the AST is discarded.

### 5-Year Horizon (Long-Term, by ~2031)

- **Verified lowering from CIR to LLVM IR via Vellvm/Alive2 semantics**: The AST back-reference model in CIR (§53.7) provides a natural proof anchor for mechanized correctness arguments; a long-term research direction is connecting `CIRGenFunction` output to Alive2-style refinement checks, verifying that each `cir.binop`/`cir.cast`/`cir.call` emission preserves source-level C++ semantics under the ISO memory model.
- **ClangIR as the universal Clang IR for all targets and language frontends**: The upstreaming roadmap envisions CIR replacing the `-emit-llvm` direct path as the default Clang IR, with targets (SPIR-V, AMDGPU, NVPTX, RISC-V) consuming CIR directly through target-specific lowering passes rather than routing through LLVM IR — reducing target-specific hacks in `CodeGenFunction` by centralizing them in CIR passes.
- **Integration with C++ reflection (P2996) and CIR attribute emission**: C++26/27 static reflection (P2996/P3395) exposes AST-level metadata at compile time; CIR's AST back-reference architecture positions it to emit reflection-query results as CIR attributes evaluated at the `CIRGenModule` phase, making reflected type information first-class in the IR and enabling specialization strategies not expressible in LLVM IR.

---

## Chapter Summary

- `CIRGenerator` is an `ASTConsumer` that drives `CIRGenModule` and `CIRGenFunction` to produce an MLIR module.
- CIR types map C/C++ types with signedness and structure preserved: `cir.int<s,32>`, `cir.record`, `cir.ptr<T>`.
- Function emission follows alloca-then-store for parameters; statements map to structured ops (`cir.if`, `cir.for`, `cir.while`, `cir.switch`).
- Expressions emit `cir.binop`, `cir.cmp`, `cir.cast`, `cir.call`, preserving overflow annotations (`nsw`/`nuw`).
- Local variables are `cir.alloca` ops tied to `cir.scope` regions; destructors are emitted at scope exits.
- C++ virtual calls, exceptions, and ABI detail are handled by `CXXABILoweringPass` post-generation, with generation-time ops as placeholders.
- AST back-references (pointers to `VarDecl*`, `FunctionDecl*`, `RecordDecl*`) survive in CIR during in-process compilation, enabling analysis passes to query Clang AST semantics.


---

@copyright jreuben11
