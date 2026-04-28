# Chapter 39 — CodeGenModule and CodeGenFunction

*Part VI — Clang Internals: Codegen and ABI*

The codegen layer is where Clang's carefully preserved AST meets the LLVM IR type system. Two classes dominate this translation: `CodeGenModule` manages the lifetime of a single `llvm::Module` and coordinates all module-level emissions, while `CodeGenFunction` handles the per-function state machine that converts Clang `Stmt` and `Expr` trees into a sequence of LLVM basic blocks and instructions. Understanding their architectures — their data members, their call sequences, their ownership model, and the invariants each maintains — is the prerequisite to understanding every subsequent topic in Part VI: statement lowering, the ABI boundary, C++ special members, and coroutine transformation. This chapter covers both classes in full, from the `ASTConsumer` entry point through the `llvm::Module` exit, with detours into `CodeGenTypes`, `CGBuilderTy`, alloca management, debug-info integration, and the deferred-emission machinery that keeps compile times tractable.

---

## 39.1 The Codegen Pipeline Entry Point

### 39.1.1 `CodeGenAction` and `BackendConsumer`

Clang's compilation pipeline is organized as a chain of `FrontendAction` objects. Code generation uses `CodeGenAction`, declared in [`clang/include/clang/CodeGen/CodeGenAction.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CodeGen/CodeGenAction.h). Concrete subclasses — `EmitObjAction`, `EmitLLVMAction`, `EmitAssemblyAction`, `EmitBCAction` — differ only in the `BackendAction` enum value they pass to the base constructor:

```cpp
// Concrete action that emits a native object file
class EmitObjAction : public CodeGenAction {
public:
  EmitObjAction(llvm::LLVMContext *VMContext = nullptr)
    : CodeGenAction(Backend_EmitObj, VMContext) {}
};
```

When the action executes, `CodeGenAction::CreateASTConsumer()` instantiates a `BackendConsumer`. The `BackendConsumer` is the pivot between Sema output and LLVM input: it implements `ASTConsumer` and holds a pointer to both the `CodeGenerator` (the IR producer) and the output stream. `BackendConsumer::HandleTranslationUnit()` is called after Sema has fully parsed the translation unit. Its body is straightforward:

```cpp
// Simplified from clang/lib/CodeGen/CodeGenAction.cpp
void BackendConsumer::HandleTranslationUnit(ASTContext &C) {
  // Forward to the CodeGenerator first so all top-level decls are emitted
  Gen->HandleTranslationUnit(C);
  // If IR generation succeeded, run the LLVM backend passes
  if (!Diags.hasErrorOccurred()) {
    llvm::Module *M = Gen->GetModule();
    emitBackendOutput(CI, CGOpts, M->getTargetTriple(), M, Action,
                      FS, std::move(AsmOutStream), this);
  }
}
```

`emitBackendOutput()`, declared in [`clang/include/clang/CodeGen/BackendUtil.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CodeGen/BackendUtil.h), constructs the LLVM pass manager pipeline and drives it to produce the final output — assembly, bitcode, or machine code depending on the `BackendAction` variant. The critical path from the compiler driver's perspective is therefore:

```
Driver → CompilerInvocation → CompilerInstance::ExecuteAction()
  → CodeGenAction::ExecuteAction()
    → ParseAST() → (per-decl) ASTConsumer::HandleTopLevelDecl()
    → BackendConsumer::HandleTranslationUnit()
      → CodeGenerator::HandleTranslationUnit()  [IR emission complete]
      → emitBackendOutput()                      [LLVM passes + output]
```

### 39.1.2 `CodeGenerator` ASTConsumer

`CodeGenerator`, declared in [`clang/include/clang/CodeGen/ModuleBuilder.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CodeGen/ModuleBuilder.h), is the public interface to Clang's IR generator. It extends `ASTConsumer` and exposes three key methods beyond the consumer protocol:

```cpp
class CodeGenerator : public ASTConsumer {
public:
  CodeGen::CodeGenModule &CGM();           // access module-level state
  llvm::Module *GetModule();               // owned until ReleaseModule()
  std::unique_ptr<llvm::Module> ReleaseModule();
  CodeGen::CGDebugInfo *getCGDebugInfo();
  const Decl *GetDeclForMangledName(StringRef MangledName);
  llvm::StringRef GetMangledName(GlobalDecl GD);
  llvm::Constant *GetAddrOfGlobal(GlobalDecl decl, bool isForDefinition);
};

std::unique_ptr<CodeGenerator>
CreateLLVMCodeGen(const CompilerInstance &CI, StringRef ModuleName,
                  llvm::LLVMContext &C,
                  CoverageSourceInfo *CoverageInfo = nullptr);
```

The factory function `CreateLLVMCodeGen()` is the preferred way to obtain a `CodeGenerator`. It returns a concrete `CodeGeneratorImpl` whose construction also instantiates the `CodeGenModule`. After the `HandleTranslationUnit()` call returns, `ReleaseModule()` transfers unique ownership of the fully-formed `llvm::Module` to the caller; subsequent calls to `GetModule()` return null.

The `IRGenFinished` flag (visible in the `ModuleBuilder.h` header) prevents duplicate emission if an AST plugin triggers further deserialization after the IR generation phase has completed — a scenario that arises in interactive REPL environments such as Clang Interpreter.

### 39.1.3 `CodeGenModule` vs `CodeGenFunction`: Ownership and Split

The two core classes implement a clear ownership hierarchy. `CodeGenModule` (defined in `clang/lib/CodeGen/CodeGenModule.h`) is created once per translation unit and survives until the `llvm::Module` is released. `CodeGenFunction` (defined in `clang/lib/CodeGen/CodeGenFunction.h`) is instantiated once per function body being compiled and destroyed when `FinishFunction()` returns. A single `CodeGenModule` may create thousands of `CodeGenFunction` instances over the course of compiling one translation unit, each a separate stack-allocated object.

Module-level state — global variable declarations, VTables, TBAA metadata trees, the mangling cache, the deferred-declaration queue — lives in `CodeGenModule`. Function-level state — the current `IRBuilder` insertion point, the alloca insertion point, the cleanup stack, the local variable map, the return block — lives in `CodeGenFunction`. Each `CodeGenFunction` holds a non-owning reference back to its parent `CodeGenModule` via a `CodeGenModule &CGM` field.

---

## 39.2 `CodeGenModule` Architecture

### 39.2.1 Core Data Members

`CodeGenModule` is a large class with roughly one hundred data members. The essential ones form three groups.

**LLVM infrastructure:**

| Field | Type | Purpose |
|-------|------|---------|
| `TheModule` | `llvm::Module &` | The module being constructed; owned externally |
| `VMContext` | `llvm::LLVMContext &` | The context that owns all LLVM types and constants |
| `Int8Ty`, `Int32Ty`, … | `llvm::IntegerType *` | Commonly used LLVM integer types, cached to avoid repeated `Type::getInt32Ty()` calls |
| `PtrTy` | `llvm::PointerType *` | The target's pointer type (opaque by default in LLVM 17+) |
| `SizeTy` | `llvm::IntegerType *` | `size_t`-width integer type for the target |

Note that `CodeGenModule` does *not* carry an `IRBuilder` or `CGBuilderTy` field — that builder lives exclusively on `CodeGenFunction`, one per active function. Module-level constant initialization is handled on demand via `ConstantInitBuilder(CGM)` (a stack-allocated RAII object that takes a `CodeGenModule&` reference), not by a persistent `Builder` field in the module object itself.

**Clang context:**

| Field | Type | Purpose |
|-------|------|---------|
| `Context` | `ASTContext &` | Source type system, constants, target info |
| `LangOpts` | `const LangOptions &` | Language dialect flags |
| `CodeGenOpts` | `const CodeGenOptions &` | `-O`, `-g`, sanitizer, PIC/PIE, etc. |
| `Target` | `const TargetInfo &` | Data layout, calling convention, TLS models |
| `Diags` | `DiagnosticsEngine &` | For emitting codegen-phase diagnostics |

**Codegen subsystems:**

| Field | Type | Purpose |
|-------|------|---------|
| `Types` | `CodeGenTypes` | `QualType` → `llvm::Type*` conversion |
| `VTables` | `CodeGenVTables` | VTable/VTT/rtti-object emission |
| `ABI` | `std::unique_ptr<CGCXXABI>` | C++ ABI dispatch (Itanium vs Microsoft) |
| `TBAA` | `std::unique_ptr<CodeGenTBAA>` | TBAA type tree builder and access-tag cache |
| `DebugInfo` | `std::unique_ptr<CGDebugInfo>` | DWARF/CodeView debug-info generation |
| `ObjCRuntime` | `std::unique_ptr<CGObjCRuntime>` | Objective-C runtime calls (if any) |
| `SanitizerMD` | `std::unique_ptr<SanitizerMetadata>` | Per-global sanitizer metadata |

### 39.2.2 `EmitGlobal()` Dispatch

`EmitGlobal()` is the central dispatch point for all top-level declarations. It receives a `GlobalDecl` — a `(Decl*, optional CXXCtorType/DtorType)` pair — and decides whether immediate emission is appropriate or the declaration should be placed in the deferred queue. The rough shape of its logic:

```cpp
void CodeGenModule::EmitGlobal(GlobalDecl GD) {
  const auto *Global = cast<ValueDecl>(GD.getDecl());

  // Skip declarations with no definition in this TU
  if (const auto *FD = dyn_cast<FunctionDecl>(Global)) {
    if (!FD->doesThisDeclarationHaveABody() && !FD->isImplicit())
      return; // external declaration; only emit a prototype on demand
    if (getLangOpts().CUDA && /* device/host filtering */ ...)
      return;
    EmitGlobalFunctionDefinition(GD, /*GV=*/nullptr);
  } else if (const auto *VD = dyn_cast<VarDecl>(Global)) {
    EmitGlobalVarDefinition(VD);
  }
}
```

`EmitGlobalFunctionDefinition()` creates an `llvm::Function` with the right linkage, then constructs a `CodeGenFunction` on the stack and calls `CGF.GenerateCode(GD, Fn, FnInfo)` to fill in the body. `EmitGlobalVarDefinition()` handles static storage, initializer evaluation, TLS flags, COMDAT membership, and section placement.

### 39.2.3 `DeferredDeclsToEmit` Queue

Not every reachable function is emitted immediately. Functions with internal linkage that have not been referenced yet, inline functions in header-only libraries, and implicit special member functions generated on demand are all placed in `DeferredDecls` (a `StringMap<GlobalDecl>`) when they are first mentioned. When codegen for a function body calls `GetAddrOfFunction()` on a not-yet-emitted callee, the callee is moved from `DeferredDecls` into `DeferredDeclsToEmit` (a `SmallVector<GlobalDecl>`). At the end of translation-unit processing, `EmitDeferred()` drains this queue:

```cpp
void CodeGenModule::EmitDeferred() {
  // DeferredDeclsToEmit grows as emission of each decl references more.
  for (size_t I = 0; I < DeferredDeclsToEmit.size(); ++I) {
    GlobalDecl D = DeferredDeclsToEmit[I];
    EmitGlobalDefinition(D);
  }
}
```

The loop index — not an iterator — handles the fact that `EmitGlobalDefinition()` may push additional entries into `DeferredDeclsToEmit`. This fixed-point iteration continues until no new declarations are enqueued. A parallel `DeferredVTables` queue collects VTable definitions deferred because the key function of their class was not yet seen.

The interaction with optimization is intentional: at `-O0`, Clang emits every reachable function eagerly. At higher optimization levels, the deferred model avoids emitting bodies that will ultimately be DCE'd, giving the pass manager a smaller initial module and reducing compile time.

### 39.2.4 `MangledDeclNames` and the Mangling Cache

Every `GlobalDecl` has a mangled name, computed by `getCXXABI().getMangleContext().mangleName()`. Because mangling is expensive — it involves template argument substitution, nested-name rendering, and target-specific encodings — `CodeGenModule` caches results in:

```cpp
llvm::StringMap<GlobalDecl, llvm::BumpPtrAllocator> MangledDeclNames;
llvm::StringMap<GlobalDecl, llvm::BumpPtrAllocator> Manglings;
```

`getMangledName(GlobalDecl GD)` returns a `StringRef` into the bump-allocated storage. The reverse mapping (`GetDeclForMangledName()`) supports the `CodeGenerator` public API used by LLDB and similar tools that need to look up declarations by the symbol names they observe in the object file.

### 39.2.5 Linkage Determination

`getLLVMLinkageForDecl()` maps from Clang's `Linkage` enum (computed by Sema) to LLVM's `GlobalValue::LinkageTypes`. The mapping is not one-to-one because LLVM encodes more information in linkage than C/C++ source semantics require:

| Clang Linkage | LLVM Linkage | Condition |
|---|---|---|
| `ExternalLinkage` + definition | `ExternalLinkage` | Most functions/variables |
| `ExternalLinkage` + inline | `LinkOnceODRLinkage` | C++ `inline` functions |
| `InternalLinkage` | `InternalLinkage` | `static` file-scope |
| `ExternalLinkage` + `weak` attribute | `WeakAnyLinkage` | `__attribute__((weak))` |
| `ExternalLinkage` + `selectany` | `WeakODRLinkage` | MSVC `__declspec(selectany)` |
| `VisibilityHidden` | `ExternalLinkage` + `hidden` visibility | `-fvisibility=hidden` |

COMDAT groups are created for `LinkOnceODR` and `WeakODR` symbols on ELF and COFF targets using `getOrInsertComdat()`, enabling the linker to deduplicate across translation units while guaranteeing one survivor.

---

## 39.3 `CodeGenTypes`: Type Conversion

### 39.3.1 `ConvertType()` Pipeline

`CodeGenTypes` (defined in `clang/lib/CodeGen/CodeGenTypes.h`) performs the bidirectional translation between Clang's `QualType` and LLVM's `llvm::Type*`. The entry point is:

```cpp
llvm::Type *CodeGenTypes::ConvertType(QualType T);
llvm::Type *CodeGenTypes::ConvertTypeForMem(QualType T);
```

`ConvertType()` returns the LLVM type appropriate for holding a value of that type in a register (a "value type"). `ConvertTypeForMem()` returns the type appropriate for an in-memory representation — the difference matters for bit-fields and `_Bool`, which are stored in more bytes than they logically occupy.

The conversion is recursive and uses a `SmallSet<const Type*> RecordsBeingLaidOut` set to detect recursive struct layout. When a struct is encountered mid-conversion, an opaque named struct is used as a forward reference and patched once the real layout is available. Without this guard, recursive structures such as linked lists would produce infinite recursion in the type converter.

Scalar conversions are straightforward:

```cpp
// Inside CodeGenTypes::ConvertType (simplified)
switch (Ty->getTypeClass()) {
case Type::Builtin:
  switch (BT->getKind()) {
  case BuiltinType::Bool:    return Int8Ty;   // stored as i8 in memory
  case BuiltinType::Char_S:  return Int8Ty;
  case BuiltinType::Int:     return Int32Ty;
  case BuiltinType::Long:    return Int64Ty;  // on LP64
  case BuiltinType::Float:   return FloatTy;
  case BuiltinType::Double:  return DoubleTy;
  // ...
  }
case Type::Pointer:
case Type::LValueReference:
case Type::RValueReference:
  return llvm::PointerType::get(VMContext, /*AS=*/0);
case Type::Record:
  return ConvertRecordDeclType(cast<RecordType>(Ty)->getDecl());
}
```

### 39.3.2 `CGRecordLayout` and Struct Lowering

Record types require layout computation. `CodeGenTypes::ComputeRecordLayout()` uses `ASTContext::getASTRecordLayout()` — which applies the C/C++ layout rules including tail padding, virtual base adjustments, and bit-field packing — and then produces both an `llvm::StructType` and a `CGRecordLayout` object.

`CGRecordLayout` stores three things:

1. The `llvm::StructType*` for the complete object.
2. A mapping from `FieldDecl*` to LLVM field index for non-bit-field members.
3. A per-bit-field descriptor encoding the field index in the backing LLVM integer, the shift amount, and the mask required to extract or insert the bit-field value.

Clang merges adjacent bit-fields of the same underlying integer type into a single LLVM integer field. The generated code for a bit-field load of `struct S { int x:3; int y:5; }` at `O0` looks like:

```llvm
; Read field 'y' (bits 3..7 of the backing i8)
%1 = load i8, ptr %s_addr, align 1
%2 = lshr i8 %1, 3        ; shift out 'x' bits
%3 = and i8 %2, 31        ; mask to 5 bits (0x1F)
%4 = sext i8 %3 to i32    ; sign-extend to int
```

The mask and shift constants are computed by `CodeGenTypes` at layout time and stored in `CGBitFieldInfo::Offset` and `CGBitFieldInfo::Size`.

### 39.3.3 `GetFunctionType()` and ABI-Adjusted Signatures

`CodeGenTypes::GetFunctionType(const CGFunctionInfo &FI)` converts an ABI-annotated function signature into an `llvm::FunctionType*`. The `CGFunctionInfo` carries `ABIArgInfo` for the return value and each parameter, computed by the target-specific `ABIInfo::computeInfo()` method (covered in depth in Chapter 41). The conversion maps each `ABIArgInfo` to zero, one, or multiple LLVM parameter types:

| `ABIArgInfo::Kind` | Effect on LLVM signature |
|---|---|
| `Direct` | Parameter present as its `CoerceToType` (or the natural LLVM type) |
| `Extend` | As `Direct` plus `zeroext`/`signext` attribute |
| `Indirect` | Replaced by a pointer; the original type disappears from the IR |
| `Ignore` | Removed entirely (empty structs, `void`) |
| `Expand` | Struct split into its individual scalar fields |
| `CoerceAndExpand` | Struct expanded into typed fields with padding |
| `InAlloca` | Passed via a shared `inalloca` struct pointer |

`ABIArgInfo` is defined in `clang/include/clang/CodeGen/CGFunctionInfo.h` with factory constructors `getDirect()`, `getIndirect()`, `getIgnore()`, `getExtend()`, etc. The distinction between `Direct` (pass as-is) and `Indirect` (pass a hidden pointer) is the central mechanism by which aggregate arguments cross the C ABI boundary — a topic developed fully in Chapter 41.

---

## 39.4 `CodeGenFunction` Lifecycle

### 39.4.1 Construction

`CodeGenFunction` is stack-allocated within `CodeGenModule::EmitGlobalFunctionDefinition()`. Its constructor takes a `CodeGenModule &` and optionally a `bool SupportsFirstClassAggregates` flag. It initializes a `CGBuilderTy` (described in Section 39.8) and the `EHStack` (the cleanup scope stack). No IR is emitted during construction.

### 39.4.2 `StartFunction()`

`StartFunction()` performs all per-function setup before the body is lowered:

```cpp
void CodeGenFunction::StartFunction(GlobalDecl GD,
                                    QualType RetTy,
                                    llvm::Function *Fn,
                                    const CGFunctionInfo &FnInfo,
                                    const FunctionArgList &Args,
                                    SourceLocation Loc,
                                    SourceLocation StartLoc);
```

Its steps, in order:

1. **Set `CurFn`** to the `llvm::Function*` being filled in.
2. **Create the entry basic block** (`createBasicBlock("entry")`) and position the `Builder` at its start.
3. **Emit the alloca for the return value** (`ReturnValue`) if the function returns by value into a caller-allocated slot (`sret`).
4. **Emit `alloca` instructions for all parameters** in the entry block — each parameter gets a local copy so it is addressable. The stores into those allocas come immediately after.
5. **Set `AllocaInsertPt`** — a `llvm::Instruction*` marking the end of the alloca region. All subsequent `CreateTempAlloca()` calls insert before this point, ensuring the alloca block stays at the top of the entry BB and enabling mem2reg to promote them all.
6. **Emit prologue code**: sanitizer prolog, stack protection canary setup, `__cyg_profile_func_enter()` if `-finstrument-functions`.
7. **Call `CGDebugInfo::EmitFunctionStart()`** if debug info is requested.

After `StartFunction()`, the `Builder` is positioned at the first instruction slot after the alloca block, ready to receive the translated body.

### 39.4.3 `FinishFunction()`

`FinishFunction()` is called after the entire function body has been lowered. It:

1. **Emits the `ReturnBlock`** — a branch target for all `return` paths that want to share a single epilogue. Only one physical ret instruction is emitted (at `-O0`; optimizers may introduce more).
2. **Runs cleanup emissions** via `PopCleanupBlocks()`, which processes deferred destructors, scope-exit actions, and EH cleanup frames.
3. **Calls `CGDebugInfo::EmitFunctionEnd()`**.
4. **Removes the `AllocaInsertPt` placeholder** instruction.
5. **Simplifies the CFG**: unreachable blocks are pruned, empty single-predecessor blocks may be merged.
6. **Calls `llvm::verifyFunction()`** in debug builds to assert that the emitted IR is well-formed.

### 39.4.4 Key Per-Function Fields

| Field | Type | Meaning |
|-------|------|---------|
| `CurFn` | `llvm::Function *` | The function currently being lowered |
| `Builder` | `CGBuilderTy` | IRBuilder positioned at current insertion point |
| `CurFuncDecl` | `const Decl *` | AST declaration for the function (may be `FunctionDecl` or `BlockDecl`) |
| `CurCodeDecl` | `const Decl *` | Like `CurFuncDecl` but adjusted for lambdas |
| `ReturnValue` | `Address` | Address of the return-value slot (sret slot or alloca) |
| `ReturnBlock` | `JumpDest` | Unified function exit point |
| `EHStack` | `EHScopeStack` | Cleanup/EH frames stack |
| `NormalCleanupDest` | `llvm::AllocaInst *` | Switch slot for multi-destination cleanups |
| `BreakContinueStack` | `SmallVector<BreakContinue>` | Targets for `break`/`continue` in loops |
| `AllocaInsertPt` | `llvm::AssertingVH<llvm::Instruction>` | Insertion point for new allocas |
| `CurLexicalScope` | `LexicalScope *` | Innermost active lexical scope for debug info |

---

## 39.5 Basic Block and Control Flow Management

### 39.5.1 `createBasicBlock()` and `EmitBlock()`

```cpp
llvm::BasicBlock *CodeGenFunction::createBasicBlock(const Twine &Name,
                                                     llvm::Function *Parent,
                                                     llvm::BasicBlock *Before);
void CodeGenFunction::EmitBlock(llvm::BasicBlock *BB, bool IsFinished = false);
void CodeGenFunction::EmitBranch(llvm::BasicBlock *Target);
```

`createBasicBlock()` allocates a new `llvm::BasicBlock` but does not attach it to the function or set it as the current insertion point — it returns an un-inserted block. `EmitBlock()` attaches the block, terminates the current block with an unconditional branch if it lacks a terminator, and repositions the `Builder`. `EmitBranch()` emits the unconditional branch then leaves the old block — the `Builder`'s insertion point is then in a logically dead position until the next `EmitBlock()`.

The idiom for conditional branching to `if`/`else` blocks is:

```cpp
llvm::BasicBlock *ThenBB = createBasicBlock("if.then");
llvm::BasicBlock *ElseBB = createBasicBlock("if.else");
llvm::BasicBlock *MergeBB = createBasicBlock("if.end");

Builder.CreateCondBr(CondV, ThenBB, ElseBB);
EmitBlock(ThenBB);
// ... emit then-body ...
EmitBranch(MergeBB);
EmitBlock(ElseBB);
// ... emit else-body ...
EmitBranch(MergeBB);
EmitBlock(MergeBB);
```

### 39.5.2 `JumpDest` and Cleanup Depth

`JumpDest` encodes a destination for structured jumps such as `break`, `continue`, `goto`, and `return`. It bundles an `llvm::BasicBlock *` with the `EHScopeStack::stable_iterator` representing the cleanup depth at the destination. When emitting a `break` that exits two loops, `EmitBranchThroughCleanup()` detects the scope depth difference and inserts the appropriate cleanup code between the source block and the target block.

```cpp
struct JumpDest {
  llvm::BasicBlock *Block;               // target BB
  EHScopeStack::stable_iterator ScopeDepth;  // cleanup level
  unsigned Index;                        // for switch-dispatch cleanups
};
```

The `BreakContinueStack` is a `SmallVector<BreakContinue, 8>` where `BreakContinue` is a pair of `JumpDest` — the `break` target and the `continue` target for the enclosing loop or `switch`. `EmitLoopBody()` pushes onto this stack before lowering the loop body and pops after. The `ReturnBlock` field is a `JumpDest` used by all `EmitReturnStmt()` calls; a single `ret` instruction at the end of the function processes any return-value alloca.

### 39.5.3 `EHStack` and Cleanup Scopes

`EHScopeStack` is a linked stack of `EHScope` frames. Each frame is one of:

- `EHCleanupScope` — runs deferred destructors or custom cleanup code on normal and/or exceptional exits.
- `EHCatchScope` — implements a `catch` block.
- `EHFilterScope` — implements `throw()` / `noexcept` (MS EH filter).
- `EHTerminateScope` — terminates the program on EH entry (for `noexcept` functions).

Cleanup scopes are pushed by `CGF.pushFullExprCleanup<T>(...)` and related methods and popped by `PopCleanupBlock()`. Clang uses lazy cleanup emission: the cleanup code is not emitted until a branch through the cleanup scope is actually taken. This avoids code bloat in the common case where no exception is thrown.

---

## 39.6 `Address` and `LValue`

### 39.6.1 The `Address` Type

`Address` is Clang's typed-pointer abstraction. Where LLVM IR uses opaque `ptr` for all pointer values (since LLVM 15's opaque-pointer migration), Clang's codegen needs to track the *pointee type* for GEP calculations, load sizes, and debug info. `Address` encodes this as a triple:

```cpp
class Address {
  llvm::Value *Pointer;        // the LLVM pointer value
  llvm::Type  *ElementType;    // pointee type (used for GEPs and loads)
  CharUnits    Alignment;      // alignment in bytes
  // Optional: KnownNonNull, AddressSpace
};
```

`CharUnits` is a 64-bit integer counting bytes, used throughout Clang's ABI machinery to represent alignments and offsets. `Address::invalid()` is a sentinel for uninitialized addresses.

### 39.6.2 `LValue` — Memory Location with Metadata

An `LValue` extends `Address` with metadata needed for load/store correctness:

```cpp
class LValue {
  Address Addr;
  LValueBaseInfo BaseInfo;     // alignment source (declared vs inferred)
  TBAAAccessInfo TBAAInfo;     // TBAA tag for this access
  bool Volatile : 1;
  bool ObjCStrong : 1;
  bool ObjCWeak : 1;
  // bit-field descriptor when isBitField()
};
```

`TBAAAccessInfo` carries the access tag that `CGM.getTBAAAccessInfo(QualType)` computes. The tag encodes the access path in the TBAA type hierarchy — for example, an access to `Foo::x` carries the tag `{!"Foo", !"int", offset=0}` which LLVM uses to prove the access does not alias with a `float*` access.

### 39.6.3 `EmitLValue()` and the Load/Store Protocol

The central function for expression codegen is:

```cpp
LValue CodeGenFunction::EmitLValue(const Expr *E);
```

`EmitLValue()` dispatches on the expression class to produce an `LValue` representing the memory location of the expression. For a `DeclRefExpr` referencing a local variable, it looks up the alloca in `LocalDeclMap`. For a `MemberExpr`, it calls `EmitLValueForField()` which applies the field's byte offset as a GEP. For an array subscript `a[i]`, it emits the index, scales by element size, and produces a GEP-based `LValue`.

Loading and storing through an `LValue`:

```cpp
RValue CodeGenFunction::EmitLoadOfLValue(LValue LV, SourceLocation Loc);
void CodeGenFunction::EmitStoreThroughLValue(RValue Src, LValue Dst);
```

These functions use the `Address`, `Alignment`, `Volatile`, and `TBAAInfo` fields to generate the correct `load`/`store` instruction with the right IR metadata:

```cpp
// Simplified EmitLoadOfLValue for a scalar
llvm::Value *CodeGenFunction::EmitLoadOfScalar(Address Addr, bool Volatile,
                                                QualType Ty,
                                                SourceLocation Loc,
                                                LValueBaseInfo BaseInfo,
                                                TBAAAccessInfo TBAAInfo) {
  llvm::LoadInst *Load = Builder.CreateLoad(Addr, Volatile, "");
  Load->setAlignment(llvm::Align(Addr.getAlignment().getQuantity()));
  if (TBAAInfo)
    CGM.DecorateInstructionWithTBAA(Load, TBAAInfo);
  return Load;
}
```

For `EmitScalarExpr(const Expr *E)`, the result is an `llvm::Value*` (not an `LValue`). `EmitScalarExpr()` calls `EmitLValue()` when the expression is an lvalue, then loads through it; for rvalue expressions it directly emits the computation.

---

## 39.7 Alloca Management

### 39.7.1 The `AllocaInsertPt` Invariant

The `mem2reg` LLVM pass promotes alloca/load/store chains to SSA values, enabling all of scalar optimization. For `mem2reg` to work, allocas must be in the function entry block with no predecessors. Clang maintains this invariant by inserting all allocas at `AllocaInsertPt` — an `llvm::Instruction` placed at the top of the entry block during `StartFunction()`. All subsequent `CreateTempAlloca()` calls insert before this point:

```cpp
Address CodeGenFunction::CreateTempAlloca(llvm::Type *Ty, CharUnits Align,
                                          const Twine &Name,
                                          llvm::Value *ArraySize,
                                          Address *AllocaAddr) {
  llvm::AllocaInst *Alloca = new llvm::AllocaInst(Ty, /*AddrSpace=*/0,
                                                   ArraySize, Name,
                                                   AllocaInsertPt->getIterator());
  Alloca->setAlignment(llvm::Align(Align.getQuantity()));
  return Address(Alloca, Ty, Align);
}
```

By inserting at `AllocaInsertPt`, all allocas appear at the top of the entry block regardless of when in the AST traversal they were requested, satisfying `mem2reg`'s requirement.

### 39.7.2 `EmitAutoVarAlloca()` and Local Variables

When `CodeGenFunction::EmitAutoVarDecl()` encounters a `VarDecl` with automatic storage, it calls `EmitAutoVarAlloca()`:

```cpp
CodeGenFunction::AutoVarEmission
CodeGenFunction::EmitAutoVarAlloca(const VarDecl &D);
```

This function:

1. Computes the LLVM type and required alignment via `CGM.Types.ConvertTypeForMem(D.getType())`.
2. If the variable is a fixed-size POD type, calls `CreateTempAlloca()` to get an entry-block alloca.
3. If the variable is a variable-length array (VLA), emits a runtime `alloca` with the computed size in the current block (VLAs cannot be promoted by `mem2reg` but are still stack-allocated).
4. Records the mapping `D → Address` in `LocalDeclMap` so subsequent `EmitLValue(DeclRefExpr)` calls can find it.
5. Optionally emits `llvm.lifetime.start` markers if `-O1` or higher and the variable is in a nested scope.

### 39.7.3 Lifetime Markers

`llvm.lifetime.start(size, ptr)` and `llvm.lifetime.end(size, ptr)` are intrinsics that inform the middle-end that a stack slot is live only within a bounded range. At `-O0`, Clang does not emit them; at `-O1` and above, `emitLifetimeMarkers()` is called around the scope of each local variable. The optimization benefit is twofold: the stack coloring pass can overlap the stack slots of non-overlapping variables, reducing frame size; and address sanitizer can detect use-after-scope bugs.

```llvm
; Lifetime markers for two non-overlapping arrays in nested scopes
  %arr1 = alloca [100 x i32], align 16
  %arr2 = alloca [50 x i32], align 16
  call void @llvm.lifetime.start.p0(ptr nonnull %arr1)
  ; ... use arr1 ...
  call void @llvm.lifetime.end.p0(ptr nonnull %arr1)
  call void @llvm.lifetime.start.p0(ptr nonnull %arr2)
  ; ... use arr2 ...
  call void @llvm.lifetime.end.p0(ptr nonnull %arr2)
```

The actual LLVM IR Clang 22 produces for the pattern above (verified):

```llvm
define dso_local void @_Z13lifetime_testv() local_unnamed_addr #0 {
  %1 = alloca [100 x i32], align 16
  %2 = alloca [50 x i32], align 16
  call void @llvm.lifetime.start.p0(ptr nonnull %1) #3
  call void @_Z7processPii(ptr noundef nonnull %1, i32 noundef 100)
  call void @llvm.lifetime.end.p0(ptr nonnull %1) #3
  call void @llvm.lifetime.start.p0(ptr nonnull %2) #3
  call void @_Z7processPii(ptr noundef nonnull %2, i32 noundef 50)
  call void @llvm.lifetime.end.p0(ptr nonnull %2) #3
  ret void
}
```

---

## 39.8 `CGBuilderTy` and IR Generation

### 39.8.1 Architecture of `CGBuilderTy`

`CGBuilderTy` is a type alias for an `llvm::IRBuilder<>` variant that has been specialized with address-aware wrappers. Its definition in `clang/lib/CodeGen/CGBuilder.h`:

```cpp
class CGBuilderTy : public CGBuilderBaseTy {
  // CGBuilderBaseTy = llvm::IRBuilder<llvm::ConstantFolder, CGBuilderInserterTy>
  CodeGenFunction *CGF;  // back-pointer to owning function
public:
  Address CreateLoad(Address Addr, const llvm::Twine &Name = "");
  llvm::StoreInst *CreateStore(llvm::Value *Val, Address Addr,
                                bool IsVolatile = false);
  Address CreateGEP(CodeGenFunction &CGF, Address Addr,
                    ArrayRef<llvm::Value*> IdxList,
                    const llvm::Twine &Name = "");
  Address CreateStructGEP(Address Addr, unsigned Index,
                          const llvm::Twine &Name = "");
  Address CreateConstArrayGEP(Address Addr, uint64_t Index,
                               const llvm::Twine &Name = "");
  Address CreateConstInBoundsGEP(Address Addr, uint64_t Index,
                                  const llvm::Twine &Name = "");
};
```

Each wrapper accepts `Address` (typed pointer + alignment) instead of a raw `llvm::Value*`, threads the element type through the GEP calculation, and propagates the correct result alignment. This is essential under opaque pointers where the LLVM IR does not encode the pointee type — Clang's codegen layer must track it internally.

### 39.8.2 TBAA Metadata Attachment

TBAA (Type-Based Alias Analysis) metadata enables the optimizer to prove that two pointer dereferences cannot alias based on their types. `CGM.DecorateInstructionWithTBAA(I, TBAAInfo)` attaches a `!tbaa` metadata node to a load or store instruction. The metadata tree is built lazily by `CGM.getTBAAAccessInfo(QualType)` and `CGM.getTBAABaseTypeInfo(QualType)`, which walk the type hierarchy to produce the TBAA path descriptor.

For the struct `Foo { int x; float y; }`, the TBAA metadata for the store `f->y = 1.0f` looks like (from actual Clang 22 output):

```llvm
store float 1.000000e+00, ptr %3, align 4, !tbaa !12
; where:
!10 = {!"_ZTS3Foo", !6, i64 0, !11, i64 4}   ; struct Foo descriptor
!11 = {!"float", !7, i64 0}                   ; float scalar node
!12 = {!10, !11, i64 4}                       ; access: Foo.y at offset 4
```

The three-element form `{base_type, access_type, offset}` is the LLVM TBAA access-tag format. When the optimizer encounters two memory accesses with disjoint access tags in the TBAA tree, it concludes they cannot alias.

### 39.8.3 Range and Loop Metadata

Beyond TBAA, `CGBuilderTy` and the surrounding codegen infrastructure attach several other metadata kinds to IR instructions.

**`!range` metadata** is attached by `EmitLoadOfScalar()` when the type has a bounded value domain. For `bool`, the only representable values are 0 and 1, so every `bool` load carries `!range !{i8 0, i8 2}` (an exclusive upper bound). This is exactly what Clang 22 produces (verified):

```llvm
; bool* p → *p
%2 = load i8, ptr %0, align 1, !tbaa !9, !range !11, !noundef !12
; where !11 = !{i8 0, i8 2}  — values in [0, 2) i.e. {0, 1}
```

The optimizer uses the `!range` annotation to prove that `(bool)*p != 0` is equivalent to `(bool)*p == 1`, eliminating a redundant comparison. Similarly, loads from `enum` types with a closed enumerator range can carry `!range` bounds when `-fstrict-enums` is in effect.

**`!llvm.loop` metadata** is attached to the back-edge branch of a loop by `CodeGenFunction::EmitCondBrHints()`. The metadata is a distinct node whose first element is a self-reference (establishing uniqueness) followed by key-value pairs. `#pragma clang loop unroll_count(4)` produces:

```llvm
br label %6, !llvm.loop !6
; where:
!6 = distinct !{!6, !7, !8}
!7 = !{!"llvm.loop.mustprogress"}
!8 = !{!"llvm.loop.unroll.count", i32 4}
```

Additional supported loop hints include `llvm.loop.vectorize.enable`, `llvm.loop.vectorize.width`, `llvm.loop.interleave.count`, `llvm.loop.unroll.full`, and `llvm.loop.distribute.enable`. Profile-guided optimization attaches `llvm.loop.vectorize.predicate.enable` and branch-weight metadata (`!prof`) on the back edge when hot loop trip counts are available from instrumentation data.

**`!nonnull` and `!noundef` metadata** are attached to loads from non-nullable source expressions. References in C++ are guaranteed non-null by the language standard, and the `!nonnull` metadata on pointer loads from a reference permits the optimizer to fold null checks away. The `!noundef` metadata (visible on the `bool` example above) signals that the loaded value is not `undef`/`poison` — it has a well-defined bit representation — which unlocks additional strength-reduction transforms in `InstCombine`.

**`!invariant.load`** is attached to loads from `const` global objects and `constexpr` variables whose address is known to be immutable after initialization. This allows the optimizer to hoist the load out of loops even in the presence of aliasing stores to other objects.

---

## 39.9 Function Attributes and Prologue

### 39.9.1 `SetLLVMFunctionAttributes()` and `SetLLVMFunctionAttributesForDefinition()`

Two functions apply LLVM IR attributes to a freshly created `llvm::Function`:

```cpp
void CodeGenModule::SetLLVMFunctionAttributes(GlobalDecl GD,
                                               const CGFunctionInfo &Info,
                                               llvm::Function *F,
                                               bool IsThunk);
void CodeGenModule::SetLLVMFunctionAttributesForDefinition(const Decl *D,
                                                            llvm::Function *F);
```

`SetLLVMFunctionAttributes()` applies attributes that can be derived from the *signature* alone — calling convention, `noreturn`, `returns_twice`, `nounwind`, parameter attributes (`byval`, `sret`, `zeroext`, etc.). `SetLLVMFunctionAttributesForDefinition()` applies attributes that require seeing the *definition* — `noinline`, `alwaysinline`, `hot`, `cold`, `optnone`, stack protector, sanitizer flags, and target-feature overrides.

### 39.9.2 Mapping Clang Attributes to LLVM Attributes

| Clang Attribute | LLVM IR Attribute |
|---|---|
| `__attribute__((noinline))` | `noinline` |
| `__attribute__((always_inline))` | `alwaysinline` |
| `__attribute__((hot))` | `hot` |
| `__attribute__((cold))` | `cold` |
| `__attribute__((noreturn))` | `noreturn` |
| `[[likely]]`/`[[unlikely]]` on calls | `!prof` branch weights |
| `__attribute__((visibility("hidden")))` | `hidden` visibility |
| `__attribute__((visibility("protected")))` | `protected` visibility |
| `__attribute__((naked))` | `naked` |
| `-fstack-protector-all` | `sspreq` attribute on functions |
| `-fsanitize=address` | `sanitize_address` function attribute |
| `-fsanitize=thread` | `sanitize_thread` |
| `-fsanitize=memory` | `sanitize_memory` |

The `optnone` attribute — emitted when `-O0` is in effect or when `__attribute__((optnone))` is specified — disables almost all optimization passes for the function. `noinline` prevents the inliner but allows other passes. The two are orthogonal: `noinline optnone` is valid.

### 39.9.3 Target-Feature Attributes

`SetLLVMFunctionAttributes()` also attaches target-specific string attributes needed by the backend:

```llvm
attributes #0 = { mustprogress noinline nounwind optnone uwtable
  "target-cpu"="x86-64"
  "target-features"="+cmov,+cx8,+fxsr,+mmx,+sse,+sse2,+x87"
  "tune-cpu"="generic"
  "stack-protector-buffer-size"="8"
  "min-legal-vector-width"="0" }
```

`"target-cpu"` and `"target-features"` come from `TargetInfo::getTargetID()` and the `-march=`/`-mcpu=`/`-mattr=` flags. They allow per-function target overrides from `__attribute__((target("avx2")))` to coexist in the same module — the backend selects the ISA for each function independently.

---

## 39.10 Global Variable Emission

### 39.10.1 `EmitGlobalVarDefinition()`

`EmitGlobalVarDefinition()` converts a C/C++ `VarDecl` with static storage into an `llvm::GlobalVariable`. Its key steps:

1. **Type conversion**: `CGM.Types.ConvertTypeForMem(D->getType())` produces the LLVM element type.
2. **Linkage determination**: `getLLVMLinkageVarDefinition(D, isConstant)` applies the rules from Section 39.2.5.
3. **Initializer evaluation**: For variables with constant initializers, `ConstExprEmitter` evaluates the Clang `APValue` to an `llvm::Constant*`. For variables with dynamic initializers, a placeholder `undef` initializer is placed and a constructor function is scheduled.
4. **Section assignment**: `__attribute__((section("...")))` sets `GV->setSection()`.
5. **Alignment**: `GV->setAlignment(CGM.getContext().getDeclAlign(D))`.
6. **COMDAT**: For `linkonce_odr` globals (e.g., inline variable definitions), `GV->setComdat(M.getOrInsertComdat(GV->getName()))`.

### 39.10.2 `AppendToGlobalCtors()` and Static Initializers

C++ requires that objects with dynamic initialization be initialized in declaration order before `main()`. Clang implements this via the `@llvm.global_ctors` and `@llvm.global_dtors` appending globals. Each entry is a `{ i32 priority, ptr fn, ptr data }` struct:

```llvm
@llvm.global_ctors = appending global
  [1 x { i32, ptr, ptr }]
  [{ i32, ptr, ptr } { i32 65535, ptr @_GLOBAL__sub_I_foo.cpp, ptr null }]
```

`AppendToGlobalCtors(M, InitFn, Priority, AssocData)` in `clang/lib/CodeGen/CGDeclCXX.cpp` appends to this array. The `_GLOBAL__sub_I_<filename>` function aggregates all per-TU dynamic initializers into a single function called at startup.

`__attribute__((constructor(N)))` and `__attribute__((destructor(N)))` with explicit priority `N` emit entries with that priority. The runtime (typically `crt1.o` / `__libc_csu_init`) traverses `@llvm.global_ctors` in priority order.

### 39.10.3 Thread-Local Storage

`thread_local` variables use the `thread_local` attribute on `llvm::GlobalVariable`. The TLS model (`general-dynamic`, `local-dynamic`, `initial-exec`, `local-exec`) is encoded as a `llvm::GlobalValue::ThreadLocalMode` enum value:

```cpp
// TLS access in generated IR (initial-exec model):
@tls_ie = dso_local thread_local(initialexec) global i32 0, align 4

// Access via llvm.threadlocal.address intrinsic:
%1 = call align 4 ptr @llvm.threadlocal.address.p0(ptr align 4 @tls_ie)
%2 = load i32, ptr %1, align 4
```

The `llvm.threadlocal.address` intrinsic was introduced in LLVM 15 to replace the `@llvm.tls_decl` pattern and to give the middle-end a typed representation of TLS accesses that supports opaque pointers.

`__attribute__((tls_model("initial-exec")))` maps to `llvm::GlobalValue::InitialExecTLSModel`. Without the attribute, Clang uses the TLS model from the `-ftls-model=` flag, defaulting to `general-dynamic` for position-independent code and `local-exec` for executables.

---

## 39.11 `CGDebugInfo`

### 39.11.1 Debug Verbosity Levels

`CGDebugInfo` is instantiated when any of `-g`, `-gline-tables-only`, or `-gmlt` is passed. The `CodeGenOptions::DebugInfoKind` field controls verbosity:

| Flag | `DebugInfoKind` | Content |
|---|---|---|
| `-g0` / no flag | `NoDebugInfo` | No debug metadata |
| `-gmlt` | `DebugLineTablesOnly` | Only `!DISubprogram` with `isDefinition=true`; no variables |
| `-gline-tables-only` | `DebugLineTablesOnly` | Line-table entries only |
| `-g1` | `DebugInfoConstructor` | Line tables + type skeletons |
| `-g` / `-g2` | `FullDebugInfo` | Full DWARF with variable locations |

At `FullDebugInfo`, `CGDebugInfo` emits a `!DICompileUnit`, `!DISubprogram` per function, `!DILexicalBlock` per scope, `!DILocalVariable` per local variable, and `!DIType` nodes for every type that appears in a local variable declaration.

### 39.11.2 Debug Intrinsics: RemoveDIs (LLVM 19+)

Prior to LLVM 19, debug variable locations were represented as calls to `llvm.dbg.declare` and `llvm.dbg.value` intrinsics. LLVM 19 introduced the "RemoveDIs" project, which replaces these intrinsic calls with `DbgVariableRecord` objects — non-instruction metadata nodes attached directly to the surrounding instruction. The textual IR representation uses the `#dbg_declare` directive:

```llvm
; LLVM 19+ format (RemoveDIs)
store i32 %0, ptr %a.addr, align 4
  #dbg_declare(ptr %a.addr, !16, !DIExpression(), !17)

; Legacy format (before LLVM 19)
call void @llvm.dbg.declare(metadata ptr %a.addr,
                             metadata !16,
                             metadata !DIExpression()), !dbg !17
```

In Clang 22, `CGDebugInfo::EmitDeclare()` unconditionally emits `DbgVariableRecord` objects rather than intrinsic calls. The `IRBuilderBase::insertDbgValueBefore()` and `insertDbgVariableRecord()` APIs are used internally. This change eliminates the overhead of treating debug intrinsics as regular instructions in the IR analysis infrastructure while preserving full DWARF-level variable location information.

### 39.11.3 `EmitFunctionStart()` and Lexical Blocks

`CGDebugInfo::EmitFunctionStart()` is called from `CodeGenFunction::StartFunction()`. It creates a `!DISubprogram` node and calls `Builder.SetCurrentDebugLocation()` to attach the function entry source location to all subsequent instructions. `EmitLexicalBlockStart()` / `EmitLexicalBlockEnd()` bracket compound statements, creating `!DILexicalBlock` nodes that give nested scopes their own source ranges.

`EmitLocation()` is called by expression emitters to attach `!dbg` location metadata to individual instructions. The implementation checks whether location tracking is active (it may be suppressed for compiler-synthesized code) before setting the `Builder`'s `DebugLoc`.

---

## 39.12 Deferred and Lazy Emission

### 39.12.1 `EmitTopLevelDecl()` Entry

`CodeGenerator::HandleTopLevelDecl()` calls `CGM.EmitTopLevelDecl(D)` for each top-level declaration delivered by the parser. This is the first entry point for global functions and variables. For a function `FunctionDecl`, `EmitTopLevelDecl()` may either call `EmitGlobal(GD)` immediately or record the decl for later emission.

The decision depends on whether the function is externally visible and whether it has been referenced yet. Clang follows the rule: *emit immediately if the function is external and has a definition; defer if it might be dead-stripped*. This is a heuristic rather than an LTO-precise analysis, erring on the side of emitting more at `-O0` for predictable debugging behavior.

### 39.12.2 Interaction with `-O0` vs `-O2`

At `-O0` (`optnone` on every function), `mem2reg` does not run, so alloca/load/store patterns remain in the IR. The optimized-variable problem — a variable optimized away, invisible to the debugger — does not arise. Clang emits an alloca for every local variable and a `#dbg_declare` pointing to it; the debugger can always find the variable's value by reading the stack slot.

At `-O2`, most allocas are promoted away by `mem2reg` before the function even enters the main optimization pipeline. Debug locations become `!DIExpression(DW_OP_deref)` and `DW_OP_LLVM_fragment` expressions that tell the debugger how to reconstruct the variable from registers. Deferred emission becomes more important: inline functions that end up dead after inlining and DCE are never emitted at all, reducing IR size and improving optimization quality.

### 39.12.3 `EmitDeferredDecls()` Final Drain

At `HandleTranslationUnit()` time, after the last AST declaration has been processed:

```cpp
void CodeGenModule::Release() {
  EmitDeferred();              // drain DeferredDeclsToEmit
  EmitVTablesOpportunistically(); // emit VTables for fully visible classes
  applyGlobalValReplacements();
  checkAliases();
  EmitCXXGlobalInitFunc();     // emit _GLOBAL__sub_I_ aggregator
  EmitCXXGlobalCleanUpFunc();
  // finalize module: set data layout, target triple, module flags, ident
}
```

`EmitDeferred()` iterates until the deferred queue is empty, emitting each deferred declaration which may in turn discover more references and grow the queue. The fixed-point loop terminates because each emission permanently removes a declaration from the deferred set and the total set of possible declarations is bounded by the TU.

---

## 39.13 A Complete Emission Trace

To ground the mechanisms described above, consider the emission trace for this minimal function:

```cpp
// Source
int compute(int a, int b) {
    int result = a + b;
    if (result > 10) result *= 2;
    return result;
}
```

**Step 1** — `HandleTopLevelDecl` → `EmitGlobal(GD)` → `EmitGlobalFunctionDefinition()`. An `llvm::Function *Fn` is created with signature `i32(i32, i32)`.

**Step 2** — `SetLLVMFunctionAttributes()` attaches calling convention, `uwtable`, `noinline`, `optnone` (at `-O0`), and the target-cpu/features string.

**Step 3** — `CodeGenFunction CGF(CGM)` is constructed. `CGF.StartFunction()` creates the entry block, emits `%a.addr = alloca i32` and `%b.addr = alloca i32` at `AllocaInsertPt`, stores the incoming parameters, and sets `CurFn = Fn`.

**Step 4** — `CGF.EmitFunctionBody()` calls `CGF.EmitCompoundStmt()` which iterates over the `{` `}` block.

**Step 5** — `EmitDeclStmt` for `int result` calls `EmitAutoVarAlloca()` → `%result = alloca i32`.

**Step 6** — `EmitBinaryOperator(a+b)` calls `EmitScalarExpr()` twice to load `a` and `b`, then emits `%add = add nsw i32 %a, %b`, then calls `EmitStoreThroughLValue()` to store into `%result`.

**Step 7** — `EmitIfStmt` creates `if.then` and `if.end` basic blocks, emits `icmp sgt`, `br i1`, fills `if.then`, branches to `if.end`.

**Step 8** — `EmitReturnStmt` emits a load from `%result`, stores into `ReturnValue`, and branches to `ReturnBlock`.

**Step 9** — `FinishFunction()` emits the `ReturnBlock` BB, loads `ReturnValue`, emits `ret i32`, removes `AllocaInsertPt`, runs the CFG simplifier.

The resulting `-O0` IR (verified):

```llvm
define dso_local noundef i32 @_Z7computeii(i32 noundef %0, i32 noundef %1) #0 {
  %3 = alloca i32, align 4    ; a.addr
  %4 = alloca i32, align 4    ; b.addr
  %5 = alloca i32, align 4    ; result
  store i32 %0, ptr %3, align 4
  store i32 %1, ptr %4, align 4
  %6 = load i32, ptr %3, align 4
  %7 = load i32, ptr %4, align 4
  %8 = add nsw i32 %6, %7
  store i32 %8, ptr %5, align 4
  %9 = load i32, ptr %5, align 4
  %10 = icmp sgt i32 %9, 10
  br i1 %10, label %11, label %14
11:
  %12 = load i32, ptr %5, align 4
  %13 = mul nsw i32 %12, 2
  store i32 %13, ptr %5, align 4
  br label %14
14:
  %15 = load i32, ptr %5, align 4
  ret i32 %15
}
```

This one-to-one correspondence between AST nodes and IR instructions is exactly what makes `-O0` debugging reliable: every source variable lives at a predictable stack address throughout its scope.

---

## Chapter Summary

- `BackendConsumer::HandleTranslationUnit()` drives IR generation and then invokes `emitBackendOutput()` to run the LLVM pass pipeline; `CodeGenAction` concrete subclasses control the output format.
- `CodeGenerator` is the public `ASTConsumer` façade; it wraps a `CodeGenModule` that owns the `llvm::Module` for the entire translation unit.
- `CodeGenModule` holds all module-level state: the deferred-emission queue, the mangling cache, the `CodeGenTypes` subsystem, the `CodeGenVTables` subsystem, and the `CGCXXABI` dispatch object.
- `CodeGenTypes::ConvertType(QualType)` maps Clang types to LLVM types, using `CGRecordLayout` for structs and `CGFunctionInfo`/`ABIArgInfo` for function signatures.
- `CodeGenFunction` is stack-allocated per function body; `StartFunction()` sets up the alloca region and the entry block; `FinishFunction()` seals the function and verifies the IR.
- The `Address` type carries pointee type and alignment alongside the LLVM pointer value; `LValue` adds TBAA and volatile metadata for correct load/store emission.
- All allocas are inserted at `AllocaInsertPt` in the entry block to guarantee `mem2reg` promotability; lifetime markers bracket nested-scope variables at `-O1+`.
- `CGBuilderTy` wraps `llvm::IRBuilder<>` with address-aware `CreateLoad`, `CreateStore`, and `CreateGEP` overloads that propagate alignment and element type.
- Function attributes are applied in two passes: signature-derived attributes via `SetLLVMFunctionAttributes()` and definition-derived attributes via `SetLLVMFunctionAttributesForDefinition()`.
- Global variables carry linkage, COMDAT membership, section assignment, and TLS model derived from Clang attributes and language rules; static initializers are aggregated into `_GLOBAL__sub_I_` functions linked via `@llvm.global_ctors`.
- `CGDebugInfo` in LLVM 22 uses the RemoveDIs format (`DbgVariableRecord`, `#dbg_declare`) rather than intrinsic calls; verbosity is controlled by `-g0`/`-gmlt`/`-gline-tables-only`/`-g`.
- The deferred-emission fixed-point loop in `EmitDeferred()` ensures that all reachable declarations are eventually emitted without emitting dead code prematurely.

---

*Cross-references:*
- [Chapter 36 — The Clang AST in Depth](../part-05-clang-frontend/ch36-the-clang-ast-in-depth.md) — `QualType`, `ASTContext`, `Decl`/`Stmt`/`Expr` hierarchies consumed by codegen
- [Chapter 16 — IR Structure](../part-04-llvm-ir/ch16-ir-structure.md) — `llvm::Module`, `llvm::Function`, `llvm::BasicBlock`, `llvm::Instruction` produced by codegen
- [Chapter 40 — Lowering Statements and Expressions](ch40-lowering-statements-and-expressions.md) — `EmitStmt()`, `EmitScalarExpr()`, and the full expression/statement emission machinery
- [Chapter 41 — Calls, the ABI Boundary, and Builtins](ch41-calls-abi-boundary-builtins.md) — `ABIArgInfo`, `CGFunctionInfo`, and `ABIInfo::computeInfo()` for call lowering

*Reference links:*
- [`clang/lib/CodeGen/CodeGenModule.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CodeGenModule.h) — `CodeGenModule` class definition
- [`clang/lib/CodeGen/CodeGenFunction.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CodeGenFunction.h) — `CodeGenFunction` class definition
- [`clang/lib/CodeGen/CodeGenTypes.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CodeGenTypes.h) — `CodeGenTypes` and `CGRecordLayout`
- [`clang/lib/CodeGen/CGBuilder.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGBuilder.h) — `CGBuilderTy`
- [`clang/lib/CodeGen/CGDebugInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGDebugInfo.h) — `CGDebugInfo`
- [`clang/lib/CodeGen/ModuleBuilder.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/ModuleBuilder.cpp) — `CodeGeneratorImpl`, `CreateLLVMCodeGen()`
- [`clang/lib/CodeGen/CodeGenAction.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CodeGenAction.cpp) — `BackendConsumer`, `HandleTranslationUnit()`
- [`clang/include/clang/CodeGen/CGFunctionInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CodeGen/CGFunctionInfo.h) — `CGFunctionInfo`, `ABIArgInfo`, `RequiredArgs`
- [`clang/include/clang/CodeGen/ModuleBuilder.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CodeGen/ModuleBuilder.h) — `CodeGenerator` public interface
