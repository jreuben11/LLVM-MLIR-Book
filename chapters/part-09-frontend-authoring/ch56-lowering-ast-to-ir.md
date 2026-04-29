# Chapter 56 — Lowering AST to IR

*Part IX — Frontend Authoring (Building Your Own)*

With a typed AST in hand (Chapter 55), the frontend's next step is generating LLVM IR. `IRBuilder<>` is the primary tool: a typed instruction factory that maintains an insertion point within a function and issues correctly-formed LLVM instructions. This chapter builds an IR emitter for the Cal expression language, covering every major pattern: expression lowering, control flow with phi nodes, the alloca-then-mem2reg idiom for mutable variables, and debug information emission.

## 56.1 IRBuilder Fundamentals

`IRBuilder<>` is defined in
[`llvm/include/llvm/IR/IRBuilder.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/IRBuilder.h). It is a template over a *folder* (for constant folding during construction) and an *inserter* (for post-insertion callbacks). The default specialization is:

```cpp
using IRBuilder = IRBuilder<ConstantFolder, IRBuilderDefaultInserter>;
```

The folder attempts to constant-fold each instruction as it is built. `CreateAdd(ConstantInt::get(i32, 3), ConstantInt::get(i32, 4))` returns `ConstantInt::get(i32, 7)` directly — no instruction is emitted. This removes a separate constant-folding pass for simple cases.

### 56.1.1 Construction and State

```cpp
llvm::LLVMContext ctx;
auto mod = std::make_unique<llvm::Module>("cal", ctx);
llvm::IRBuilder<> builder(ctx);
```

The builder's key state is its *insertion point*: a basic block and an iterator within it. All `Create*` calls insert at the current insertion point:

```cpp
// Move insertion to the end of a basic block
builder.SetInsertPoint(bb);

// Move insertion to a specific instruction
builder.SetInsertPoint(someInst);

// Move insertion past all allocas in a function (preferred for new allocas)
builder.SetInsertPointPastAllocas(fn);

// Query current position
llvm::BasicBlock *curBB  = builder.GetInsertBlock();
llvm::Function   *curFn  = curBB->getParent();
```

### 56.1.2 Type Helpers

The builder provides shorthand type constructors:

```cpp
llvm::Type *i1   = builder.getInt1Ty();   // bool
llvm::Type *i8   = builder.getInt8Ty();
llvm::Type *i32  = builder.getInt32Ty();
llvm::Type *i64  = builder.getInt64Ty();
llvm::Type *f64  = builder.getDoubleTy();
llvm::Type *voidT = builder.getVoidTy();
llvm::Type *ptrT  = builder.getPtrTy();   // opaque pointer (LLVM 15+)

// Constants
llvm::Value *zero32 = builder.getInt32(0);
llvm::Value *one64  = builder.getInt64(1);
llvm::Value *trueV  = builder.getTrue();
llvm::Value *falseV = builder.getFalse();
```

In LLVM 15+, all pointer types are opaque (`ptr` / `ptr addrspace(N)`); pointee types are tracked separately by the IR consumer, not stored in the pointer type itself.

## 56.2 The IR Emitter Structure

```cpp
class IREmitter {
  llvm::LLVMContext &ctx;
  std::unique_ptr<llvm::Module> mod;
  llvm::IRBuilder<> builder;

  // Per-function state
  llvm::Function *curFn   = nullptr;
  // Variable map: VarDecl name → alloca address
  llvm::StringMap<llvm::AllocaInst*> namedValues;

  llvm::Type *cirTypeToLLVM(TypeNode *ty);
  llvm::FunctionType *fnTypeToLLVM(FnDecl *fn);

public:
  IREmitter(llvm::LLVMContext &ctx, llvm::StringRef name)
      : ctx(ctx), mod(std::make_unique<llvm::Module>(name, ctx)),
        builder(ctx) {}

  std::unique_ptr<llvm::Module> emit(Module &ast);
  llvm::Function *emitFnDecl(FnDecl *fn);
  llvm::Value *emitExpr(ExprNode *expr);
  bool emitStmt(StmtNode *stmt);
};
```

### 56.2.1 Type Mapping

```cpp
llvm::Type *IREmitter::cirTypeToLLVM(TypeNode *ty) {
  if (auto *named = llvm::dyn_cast<NamedTypeNode>(ty)) {
    if (named->name == "int")   return builder.getInt64Ty();
    if (named->name == "float") return builder.getDoubleTy();
    if (named->name == "bool")  return builder.getInt1Ty();
    if (named->name == "void")  return builder.getVoidTy();
    llvm_unreachable("unknown named type");
  }
  if (auto *ptr = llvm::dyn_cast<PtrTypeNode>(ty))
    return builder.getPtrTy();
  llvm_unreachable("unknown type node kind");
}
```

## 56.3 Emitting Expressions

### 56.3.1 Literals

```cpp
llvm::Value *IREmitter::emitExpr(ExprNode *expr) {
  if (auto *lit = llvm::dyn_cast<IntLitExpr>(expr))
    return llvm::ConstantInt::get(builder.getInt64Ty(), lit->value,
                                   /*isSigned=*/true);

  if (auto *lit = llvm::dyn_cast<FpLitExpr>(expr))
    return llvm::ConstantFP::get(builder.getDoubleTy(), lit->value);

  if (auto *lit = llvm::dyn_cast<BoolLitExpr>(expr))
    return builder.getInt1(lit->value ? 1 : 0);
```

### 56.3.2 Variable References

Variables are stored on the stack as allocas. A reference loads the value:

```cpp
  if (auto *id = llvm::dyn_cast<IdentExpr>(expr)) {
    auto it = namedValues.find(id->name);
    assert(it != namedValues.end() && "undefined variable");
    llvm::AllocaInst *alloca = it->second;
    return builder.CreateLoad(alloca->getAllocatedType(), alloca, id->name);
  }
```

### 56.3.3 Binary Operations

```cpp
  if (auto *binop = llvm::dyn_cast<BinopExpr>(expr)) {
    llvm::Value *lhs = emitExpr(binop->lhs.get());
    llvm::Value *rhs = emitExpr(binop->rhs.get());
    bool isInt   = lhs->getType()->isIntegerTy();
    bool isBool  = lhs->getType()->isIntegerTy(1);

    switch (binop->op) {
    case TokenKind::Plus:
      return isInt ? builder.CreateAdd(lhs, rhs, "add")
                   : builder.CreateFAdd(lhs, rhs, "fadd");
    case TokenKind::Minus:
      return isInt ? builder.CreateSub(lhs, rhs, "sub")
                   : builder.CreateFSub(lhs, rhs, "fsub");
    case TokenKind::Star:
      return isInt ? builder.CreateMul(lhs, rhs, "mul")
                   : builder.CreateFMul(lhs, rhs, "fmul");
    case TokenKind::Slash:
      return isInt ? builder.CreateSDiv(lhs, rhs, "div")
                   : builder.CreateFDiv(lhs, rhs, "fdiv");
    case TokenKind::Percent:
      return builder.CreateSRem(lhs, rhs, "rem");
    case TokenKind::EqEq:
      return isInt ? builder.CreateICmpEQ(lhs, rhs, "eq")
                   : builder.CreateFCmpOEQ(lhs, rhs, "feq");
    case TokenKind::BangEq:
      return isInt ? builder.CreateICmpNE(lhs, rhs, "ne")
                   : builder.CreateFCmpONE(lhs, rhs, "fne");
    case TokenKind::Lt:
      return isInt ? builder.CreateICmpSLT(lhs, rhs, "lt")
                   : builder.CreateFCmpOLT(lhs, rhs, "flt");
    case TokenKind::LtEq:
      return isInt ? builder.CreateICmpSLE(lhs, rhs, "le")
                   : builder.CreateFCmpOLE(lhs, rhs, "fle");
    case TokenKind::Gt:
      return isInt ? builder.CreateICmpSGT(lhs, rhs, "gt")
                   : builder.CreateFCmpOGT(lhs, rhs, "fgt");
    case TokenKind::GtEq:
      return isInt ? builder.CreateICmpSGE(lhs, rhs, "ge")
                   : builder.CreateFCmpOGE(lhs, rhs, "fge");
    default: llvm_unreachable("unknown binop");
    }
  }
```

**Naming values**: every `Create*` call accepts an optional `const Twine &Name` argument. Names appear in the IR text (`%add`, `%lt`) and survive until module printing. They do not affect semantics but are invaluable for debugging. The name is unique-ified automatically if it conflicts with an existing value name.

### 56.3.4 Unary Operations

```cpp
  if (auto *unary = llvm::dyn_cast<UnaryExpr>(expr)) {
    llvm::Value *operand = emitExpr(unary->operand.get());
    switch (unary->op) {
    case TokenKind::Minus:
      return operand->getType()->isFloatingPointTy()
             ? builder.CreateFNeg(operand, "fneg")
             : builder.CreateNeg(operand, "neg");
    case TokenKind::Bang:
      // !bool: XOR with 1
      return builder.CreateXor(operand, builder.getTrue(), "not");
    default: llvm_unreachable("unknown unary op");
    }
  }
```

### 56.3.5 Function Calls

```cpp
  if (auto *call = llvm::dyn_cast<CallExpr>(expr)) {
    llvm::Function *callee = mod->getFunction(call->callee);
    assert(callee && "undefined function");

    std::vector<llvm::Value*> args;
    for (auto &argExpr : call->args)
      args.push_back(emitExpr(argExpr.get()));

    return builder.CreateCall(callee, args,
                               callee->getReturnType()->isVoidTy() ? "" : "call");
  }
```

`CreateCall` receives a `FunctionCallee` (function type + value) or a `Function*` directly. For indirect calls through a function pointer, pass the pointer value and the explicit `FunctionType`:

```cpp
llvm::FunctionType *fty = llvm::FunctionType::get(retTy, paramTys, /*vararg=*/false);
builder.CreateCall(fty, fnPtrVal, args, "indirect");
```

## 56.4 Control Flow with Phi Nodes

Control flow in LLVM IR requires explicit basic blocks and branch instructions. Consider an `if` expression that produces a value:

```cpp
// Source: let x = if cond { 10 } else { 20 };
// IR pattern with phi:
//
//   br i1 %cond, %then, %else
// then:
//   br %merge
// else:
//   br %merge
// merge:
//   %x = phi i64 [ 10, %then ], [ 20, %else ]
```

Implementation:

```cpp
llvm::Value *IREmitter::emitIfExpr(IfExpr *ifExpr) {
  llvm::Value *cond = emitExpr(ifExpr->cond.get());
  // Ensure cond is i1
  if (!cond->getType()->isIntegerTy(1))
    cond = builder.CreateICmpNE(cond, llvm::ConstantInt::get(cond->getType(), 0));

  llvm::Function *fn = builder.GetInsertBlock()->getParent();
  llvm::BasicBlock *thenBB  = llvm::BasicBlock::Create(ctx, "then",  fn);
  llvm::BasicBlock *elseBB  = llvm::BasicBlock::Create(ctx, "else");
  llvm::BasicBlock *mergeBB = llvm::BasicBlock::Create(ctx, "merge");

  builder.CreateCondBr(cond, thenBB, elseBB);

  // Emit then
  builder.SetInsertPoint(thenBB);
  llvm::Value *thenVal = emitExpr(ifExpr->thenBody.get());
  builder.CreateBr(mergeBB);
  thenBB = builder.GetInsertBlock(); // may have changed if nested branches

  // Emit else
  fn->insert(fn->end(), elseBB);
  builder.SetInsertPoint(elseBB);
  llvm::Value *elseVal = emitExpr(ifExpr->elseBody.get());
  builder.CreateBr(mergeBB);
  elseBB = builder.GetInsertBlock();

  // Emit phi in merge
  fn->insert(fn->end(), mergeBB);
  builder.SetInsertPoint(mergeBB);
  llvm::PHINode *phi = builder.CreatePHI(thenVal->getType(), 2, "iftmp");
  phi->addIncoming(thenVal, thenBB);
  phi->addIncoming(elseVal, elseBB);
  return phi;
}
```

Key subtlety: after emitting a nested region, re-capture the insertion block (`thenBB = builder.GetInsertBlock()`) because nested branches may have changed the active block. `PHINode::addIncoming` requires the **actual** predecessor block, not the block that was current when you began emitting the region.

### 56.4.1 Loop Emission

```cpp
void IREmitter::emitWhileStmt(WhileStmt *ws) {
  llvm::Function *fn = builder.GetInsertBlock()->getParent();
  llvm::BasicBlock *condBB = llvm::BasicBlock::Create(ctx, "while.cond", fn);
  llvm::BasicBlock *bodyBB = llvm::BasicBlock::Create(ctx, "while.body");
  llvm::BasicBlock *exitBB = llvm::BasicBlock::Create(ctx, "while.exit");

  builder.CreateBr(condBB);

  // Condition
  builder.SetInsertPoint(condBB);
  llvm::Value *cond = emitExpr(ws->cond.get());
  builder.CreateCondBr(cond, bodyBB, exitBB);

  // Body
  fn->insert(fn->end(), bodyBB);
  builder.SetInsertPoint(bodyBB);
  emitStmt(ws->body.get());
  if (!builder.GetInsertBlock()->getTerminator())
    builder.CreateBr(condBB); // back-edge

  // Exit
  fn->insert(fn->end(), exitBB);
  builder.SetInsertPoint(exitBB);
}
```

The back-edge from `bodyBB` to `condBB` creates the loop. The `break`/`continue` statements are handled by maintaining a stack of `{condBB, exitBB}` pairs that `BreakStmt` and `ContinueStmt` branch to.

## 56.5 The Alloca-then-mem2reg Idiom

Mutable local variables are implemented with explicit allocas rather than phi nodes. This separates concerns: the frontend emits simple load/store sequences, and the `mem2reg` pass promotes them to SSA phi nodes automatically.

### 56.5.1 Creating Allocas at Function Entry

All allocas must appear in the entry block of the function (before any branch) for `mem2reg` to promote them. The `SetInsertPointPastAllocas` helper positions the builder after existing allocas:

```cpp
llvm::AllocaInst *IREmitter::createEntryAlloca(llvm::Function *fn,
                                                llvm::Type *ty,
                                                const llvm::Twine &name) {
  llvm::IRBuilder<> entry(&fn->getEntryBlock(),
                            fn->getEntryBlock().begin());
  return entry.CreateAlloca(ty, nullptr, name);
}
```

A separate builder object positioned at the function entry block ensures allocas land before any other instructions, regardless of the emission order of statements:

```cpp
// Emit a let statement
bool IREmitter::emitLetStmt(LetStmt *ls) {
  llvm::Type *ty = cirTypeToLLVM(ls->type.get());
  llvm::AllocaInst *alloca = createEntryAlloca(curFn, ty, ls->name);
  namedValues[ls->name] = alloca;

  if (ls->init) {
    llvm::Value *initVal = emitExpr(ls->init.get());
    builder.CreateStore(initVal, alloca);
  }
  return true;
}
```

### 56.5.2 Running mem2reg

After emitting the entire module, `mem2reg` (the `PromoteMemoryToRegisterPass`) promotes all alloca/load/store triples into phi-form SSA:

```cpp
// Using the new pass manager
llvm::LoopAnalysisManager LAM;
llvm::FunctionAnalysisManager FAM;
llvm::CGSCCAnalysisManager CGAM;
llvm::ModuleAnalysisManager MAM;

llvm::PassBuilder PB;
PB.registerModuleAnalyses(MAM);
PB.registerCGSCCAnalyses(CGAM);
PB.registerFunctionAnalyses(FAM);
PB.registerLoopAnalyses(LAM);
PB.crossRegisterProxies(LAM, FAM, CGAM, MAM);

llvm::FunctionPassManager FPM;
FPM.addPass(llvm::PromotePass()); // mem2reg

llvm::ModulePassManager MPM;
MPM.addPass(llvm::createModuleToFunctionPassAdaptor(std::move(FPM)));
MPM.run(*mod, MAM);
```

After `mem2reg`, all alloca/load/store patterns that can be promoted are in SSA phi-form — identical to what a direct phi-emitting frontend produces, but with far simpler frontend code.

### 56.5.3 When mem2reg Cannot Promote

`mem2reg` cannot promote an alloca if:
- Its address is taken and passed to a function (the alloca escapes to the heap).
- It is indexed with a non-constant GEP (array element by computed index).
- It is a `volatile` access.

In these cases the alloca remains as-is. Alias analysis and subsequent optimization passes handle it.

## 56.6 Function Emission

### 56.6.1 Function Declaration

```cpp
llvm::Function *IREmitter::emitFnDecl(FnDecl *fn) {
  // Build LLVM function type
  std::vector<llvm::Type*> paramTys;
  for (auto &[name, type] : fn->params)
    paramTys.push_back(cirTypeToLLVM(type.get()));
  llvm::Type *retTy = fn->retType ? cirTypeToLLVM(fn->retType.get())
                                  : builder.getVoidTy();
  auto *fty = llvm::FunctionType::get(retTy, paramTys, /*vararg=*/false);

  auto linkage = fn->body ? llvm::Function::ExternalLinkage
                           : llvm::Function::ExternalLinkage; // extern
  auto *func = llvm::Function::Create(fty, linkage, fn->name, mod.get());

  // Name parameters
  unsigned i = 0;
  for (auto &arg : func->args())
    arg.setName(fn->params[i++].first);

  if (!fn->body) return func; // extern declaration done

  // Create entry block and emit body
  auto *entry = llvm::BasicBlock::Create(ctx, "entry", func);
  builder.SetInsertPoint(entry);
  curFn = func;
  namedValues.clear();

  // Materialize parameters as allocas (for mutability)
  i = 0;
  for (auto &arg : func->args()) {
    auto *alloca = createEntryAlloca(func, arg.getType(), arg.getName());
    builder.CreateStore(&arg, alloca);
    namedValues[std::string(arg.getName())] = alloca;
    ++i;
  }

  emitStmt(fn->body.get());

  // Append implicit return if needed
  if (!builder.GetInsertBlock()->getTerminator()) {
    if (retTy->isVoidTy())
      builder.CreateRetVoid();
    else
      builder.CreateRet(llvm::UndefValue::get(retTy)); // unreachable path
  }

  llvm::verifyFunction(*func, &llvm::errs());
  return func;
}
```

`llvm::verifyFunction` checks the function for IR well-formedness (every phi has an entry for every predecessor, every branch terminates a block, types are consistent). It is essential to call this after emitting each function during development.

## 56.7 Debug Information Emission

Debug info in LLVM uses the `DIBuilder` API (declared in
[`llvm/include/llvm/IR/DIBuilder.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/DIBuilder.h)). It attaches DWARF metadata to functions, variables, and instructions.

### 56.7.1 Setting Up DIBuilder

```cpp
class IREmitter {
  // …
  llvm::DIBuilder *diBuilder = nullptr;
  llvm::DICompileUnit *diCU  = nullptr;
  llvm::DIFile        *diFile = nullptr;

  void initDebugInfo(llvm::StringRef filename, llvm::StringRef dir) {
    diBuilder = new llvm::DIBuilder(*mod);
    diFile = diBuilder->createFile(filename, dir);
    diCU   = diBuilder->createCompileUnit(
      llvm::dwarf::DW_LANG_C,       // language tag (use custom for Cal)
      diFile,
      "Cal Compiler 1.0",           // producer string
      /*isOptimized=*/false,
      /*Flags=*/"",
      /*RuntimeVersion=*/0);
    mod->addModuleFlag(llvm::Module::Warning, "Dwarf Version",
                       llvm::dwarf::DWARF_VERSION);
    mod->addModuleFlag(llvm::Module::Warning, "Debug Info Version",
                       llvm::DEBUG_METADATA_VERSION);
  }
};
```

### 56.7.2 Function Debug Info

```cpp
llvm::Function *IREmitter::emitFnDeclWithDebug(FnDecl *fn) {
  auto *func = emitFnDecl(fn); // existing emission

  // Create subroutine type
  llvm::SmallVector<llvm::Metadata*> paramDITypes;
  paramDITypes.push_back(diTypeFor(fn->retType.get()));
  for (auto &[name, type] : fn->params)
    paramDITypes.push_back(diTypeFor(type.get()));

  auto *diSubrType = diBuilder->createSubroutineType(
    diBuilder->getOrCreateTypeArray(paramDITypes));

  auto *diSub = diBuilder->createFunction(
    diCU, fn->name, llvm::StringRef(), diFile,
    fn->body ? fn->body->line : 0,  // line number
    diSubrType,
    /*ScopeLine=*/fn->body ? fn->body->line : 0,
    llvm::DINode::FlagPrototyped,
    llvm::DISubprogram::SPFlagDefinition);

  func->setSubprogram(diSub);
  return func;
}
```

### 56.7.3 Variable Debug Info

For each local variable, emit a `DILocalVariable` and a `dbg.declare` intrinsic:

```cpp
bool IREmitter::emitLetStmtWithDebug(LetStmt *ls) {
  emitLetStmt(ls); // creates the alloca

  llvm::AllocaInst *alloca = namedValues[ls->name];
  auto *diVarType = diTypeFor(ls->type.get());
  auto *diVar = diBuilder->createAutoVariable(
    diBuilder->createLexicalBlock(curDiScope, diFile, ls->line, ls->col),
    ls->name, diFile, ls->line, diVarType);

  diBuilder->insertDeclare(alloca, diVar,
    diBuilder->createExpression(),
    llvm::DILocation::get(ctx, ls->line, ls->col, curDiScope),
    builder.GetInsertBlock());
  return true;
}
```

`dbg.declare` links the alloca to the debug variable. After `mem2reg`, it becomes `dbg.value` intrinsics at each assignment point.

### 56.7.4 Instruction Locations

Every emitted instruction should carry a `!dbg` location metadata:

```cpp
builder.SetCurrentDebugLocation(
  llvm::DILocation::get(ctx, expr->line, expr->col, curDiScope));
llvm::Value *v = builder.CreateAdd(lhs, rhs, "add");
// The add instruction now has !dbg metadata
```

Setting the debug location on the builder before each instruction attaches it automatically to all subsequently created instructions.

### 56.7.5 Finalizing Debug Info

```cpp
void IREmitter::finalizeDebugInfo() {
  diBuilder->finalize(); // writes all debug info to the module
}
```

`diBuilder->finalize()` must be called after all functions are emitted; it resolves forward references in the debug info metadata graph.

## 56.8 Module Verification and Output

After emitting the entire module:

```cpp
// Verify IR well-formedness
if (llvm::verifyModule(*mod, &llvm::errs())) {
  llvm::errs() << "module verification failed\n";
  return nullptr;
}

// Print textual IR
mod->print(llvm::outs(), nullptr);

// Or write bitcode
llvm::WriteBitcodeToFile(*mod, os);
```

`llvm::verifyModule` checks global-level invariants: every global value is internally consistent, all functions verified, all metadata self-consistent.

## 56.9 End-to-End Example

Given this Cal source:

```
fn fib(n: int) -> int {
  if n <= 1 {
    return n;
  }
  return fib(n - 1) + fib(n - 2);
}
```

The emitter produces (before optimization):

```llvm
define i64 @fib(i64 %n) {
entry:
  %n.addr = alloca i64
  store i64 %n, ptr %n.addr
  %n.1 = load i64, ptr %n.addr
  %le = icmp sle i64 %n.1, 1
  br i1 %le, label %then, label %else

then:
  %n.2 = load i64, ptr %n.addr
  ret i64 %n.2

else:
  %n.3  = load i64, ptr %n.addr
  %sub1 = sub nsw i64 %n.3, 1
  %r1   = call i64 @fib(i64 %sub1)
  %n.4  = load i64, ptr %n.addr
  %sub2 = sub nsw i64 %n.4, 2
  %r2   = call i64 @fib(i64 %sub2)
  %add  = add nsw i64 %r1, %r2
  ret i64 %add
}
```

After `mem2reg`:

```llvm
define i64 @fib(i64 %n) {
entry:
  %le = icmp sle i64 %n, 1
  br i1 %le, label %then, label %else

then:
  ret i64 %n

else:
  %sub1 = sub nsw i64 %n, 1
  %r1   = call i64 @fib(i64 %sub1)
  %sub2 = sub nsw i64 %n, 2
  %r2   = call i64 @fib(i64 %sub2)
  %add  = add nsw i64 %r1, %r2
  ret i64 %add
}
```

The redundant loads and alloca for `n` are eliminated; the parameter is used directly.

---

## Chapter Summary

- `IRBuilder<>` is the core instruction factory; its insertion point determines where new instructions land.
- Expressions map directly to `Create*` calls: `CreateAdd`, `CreateICmpSLT`, `CreateFNeg`, `CreateCall`, etc.
- Control flow uses basic blocks with `CreateCondBr`/`CreateBr` and `CreatePHI` for value-producing branches.
- The alloca-then-mem2reg idiom keeps the frontend simple: emit alloca/load/store for mutable variables, then run `PromotePass` to convert them to SSA phi nodes.
- All allocas for a function should be emitted in the entry block; use a separate `IRBuilder` positioned at `fn->getEntryBlock().begin()` to guarantee this.
- `llvm::verifyFunction` / `llvm::verifyModule` should be called after emission during development to catch IR invariant violations early.
- `DIBuilder` attaches DWARF debug metadata to functions (via `createFunction`/`setSubprogram`), variables (via `createAutoVariable`/`insertDeclare`), and instructions (via `SetCurrentDebugLocation`).


---

@copyright jreuben11
