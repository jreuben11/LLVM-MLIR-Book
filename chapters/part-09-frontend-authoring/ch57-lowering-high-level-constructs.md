# Chapter 57 — Lowering High-Level Constructs

*Part IX — Frontend Authoring (Building Your Own)*

A real language frontend must lower constructs that have no direct LLVM counterpart: aggregates, dynamic dispatch, closures, tagged unions, coroutines, string literals, and runtime type information. Each requires a deliberate mapping from the source-level abstraction to sequences of LLVM IR instructions. This chapter documents the canonical lowering strategies for each, with worked code examples that extend the Cal emitter from Chapter 56.

## 57.1 Aggregates

### 57.1.1 Struct Types

A C-like struct is lowered to an LLVM `StructType`. Field layout uses the same rules as the C ABI: natural alignment for each field, with padding inserted to maintain it:

```cpp
llvm::StructType *emitRecordType(RecordDecl *decl, llvm::Module &mod) {
  llvm::SmallVector<llvm::Type*> fieldTys;
  for (auto *field : decl->fields())
    fieldTys.push_back(toLLVM(field->getType()));
  auto *sty = llvm::StructType::create(mod.getContext(), fieldTys, decl->name());
  return sty;
}
```

LLVM's `StructType::create` produces an *identified* struct (named, forward-declarable, suitable for recursive types). `StructType::get` produces a *literal* struct (anonymous, structurally typed — no forward references).

**Field access** via `getelementptr`:

```cpp
// struct Point { int x; int y; };
// Point p; int px = p.x;
//
// p is an alloca:  %p.addr = alloca %Point
// load x:
//   %px.addr = getelementptr inbounds %Point, ptr %p.addr, i32 0, i32 0
//   %px = load i32, ptr %px.addr

llvm::Value *IREmitter::emitFieldAccess(llvm::Value *structPtr,
                                         unsigned fieldIdx,
                                         llvm::StructType *sty) {
  return builder.CreateStructGEP(sty, structPtr, fieldIdx, "field");
}
```

`CreateStructGEP` is shorthand for `CreateGEP` with indices `{0, fieldIdx}`. It produces an inbounds GEP pointing to the field within the struct.

### 57.1.2 Arrays

Fixed-size arrays:

```cpp
// int arr[4];  →  alloca [4 x i32]
llvm::Type *elemTy = builder.getInt32Ty();
llvm::ArrayType *aty = llvm::ArrayType::get(elemTy, 4);
llvm::AllocaInst *arrAlloca = builder.CreateAlloca(aty, nullptr, "arr");

// arr[i]:
//   %p = getelementptr inbounds [4 x i32], ptr %arr, i64 0, i64 %i
llvm::Value *idx = builder.CreateSExt(indexVal, builder.getInt64Ty(), "idx");
llvm::Value *elemPtr = builder.CreateInBoundsGEP(aty, arrAlloca,
                         {builder.getInt64(0), idx}, "elem");
```

**Bounds checking**: if the language guarantees bounds safety, emit a check before the GEP:

```cpp
void IREmitter::emitBoundsCheck(llvm::Value *idx, int64_t len, AstNode *loc) {
  // Check 0 <= idx < len
  llvm::Value *loOk = builder.CreateICmpSGE(idx, builder.getInt64(0), "lo");
  llvm::Value *hiOk = builder.CreateICmpSLT(idx, builder.getInt64(len), "hi");
  llvm::Value *ok   = builder.CreateAnd(loOk, hiOk, "inbounds");

  auto *fn = builder.GetInsertBlock()->getParent();
  auto *failBB = llvm::BasicBlock::Create(ctx, "bounds.fail", fn);
  auto *okBB   = llvm::BasicBlock::Create(ctx, "bounds.ok");
  builder.CreateCondBr(ok, okBB, failBB);

  // Emit panic in fail block
  builder.SetInsertPoint(failBB);
  auto *panic = mod->getOrInsertFunction("__bounds_fail",
    llvm::FunctionType::get(builder.getVoidTy(), {}, false));
  auto *call = builder.CreateCall(panic, {});
  cast<llvm::CallInst>(call)->setDoesNotReturn();
  builder.CreateUnreachable();

  fn->insert(fn->end(), okBB);
  builder.SetInsertPoint(okBB);
}
```

The `setDoesNotReturn()` attribute tells LLVM that execution never returns from this call, enabling better dead-code elimination in the success path.

### 57.1.3 Passing Aggregates

Structs may be passed by value or by hidden pointer, depending on the platform ABI:

- **Small structs** (≤2 machine words on x86-64): passed in registers as `{ i64, i64 }` (coercion).
- **Large structs**: passed via a hidden `sret` parameter (a pointer to caller-allocated storage).

Frontends that target multiple platforms should query the ABI via `llvm::TargetData` / `llvm::DataLayout`:

```cpp
const llvm::DataLayout &DL = mod->getDataLayout();
llvm::TypeSize size = DL.getTypeAllocSize(structTy);
if (size <= 16) { /* pass by value coerced */ }
else            { /* pass by sret pointer */ }
```

## 57.2 Vtables and Dynamic Dispatch

Vtable-based polymorphism maps to LLVM IR as global arrays of function pointers.

### 57.2.1 Vtable Layout

```cpp
// class Base { virtual int foo(int); virtual void bar(); };
//
// Vtable for Base:
//   @_ZTV4Base = global [4 x ptr] [
//     ptr null,           ; offset-to-top (Itanium: 0 for primary)
//     ptr @_ZTI4Base,     ; RTTI pointer
//     ptr @_ZN4Base3fooEi,; slot 0 → foo
//     ptr @_ZN4Base3barEv  ; slot 1 → bar
//   ]

llvm::Constant *IREmitter::buildVtable(ClassDecl *cls) {
  llvm::SmallVector<llvm::Constant*> entries;
  // Itanium mandatory slots
  entries.push_back(llvm::Constant::getNullValue(builder.getPtrTy())); // offset-to-top
  entries.push_back(buildRTTI(cls)); // RTTI pointer

  // Virtual function pointers
  for (VirtualMethod *vm : cls->virtualMethods()) {
    llvm::Function *fn = mod->getFunction(vm->mangledName());
    entries.push_back(fn);
  }

  auto *arrTy = llvm::ArrayType::get(builder.getPtrTy(), entries.size());
  auto *vtable = new llvm::GlobalVariable(*mod, arrTy, /*isConstant=*/true,
                   llvm::GlobalValue::ExternalLinkage,
                   llvm::ConstantArray::get(arrTy, entries),
                   "_ZTV" + cls->mangledName());
  return vtable;
}
```

### 57.2.2 Object Layout

Every polymorphic object has a vtable pointer as its first field:

```cpp
// struct Base { ptr vtptr; /* other fields */ }
llvm::StructType *objectTy = llvm::StructType::create(
  ctx, {builder.getPtrTy(), /* other fields */}, cls->name());
```

The vtable pointer is stored at offset 0, so loading it requires a single GEP + load:

```cpp
llvm::Value *IREmitter::loadVtptr(llvm::Value *objPtr) {
  return builder.CreateLoad(builder.getPtrTy(), objPtr, "vtptr");
}
```

### 57.2.3 Virtual Dispatch

```cpp
llvm::Value *IREmitter::emitVirtualCall(llvm::Value *objPtr,
                                         VirtualMethod *vm,
                                         ArrayRef<llvm::Value*> args) {
  // 1. Load vtable pointer from object
  llvm::Value *vtptr = loadVtptr(objPtr);

  // 2. Load function pointer from vtable slot
  //    slot index accounts for the 2 Itanium prefix entries
  int slotIdx = vm->vtableIndex() + 2; // +2 for offset-to-top and RTTI
  llvm::Value *slotPtr = builder.CreateConstInBoundsGEP1_64(
    builder.getPtrTy(), vtptr, slotIdx, "vslot.ptr");
  llvm::Value *fnPtr = builder.CreateLoad(builder.getPtrTy(), slotPtr, "vfn");

  // 3. Call through the pointer
  auto *fty = toLLVMFuncType(vm->funcType());
  llvm::SmallVector<llvm::Value*> allArgs = {objPtr}; // implicit this
  allArgs.append(args.begin(), args.end());
  return builder.CreateCall(fty, fnPtr, allArgs, "vcall");
}
```

## 57.3 Closures

A closure is a function that captures values from its enclosing scope. The canonical IR lowering converts a closure to a pair: a function pointer that accepts an explicit *environment pointer*, and a heap-allocated environment record containing the captured values.

### 57.3.1 Closure Type

```
; Closure = { ptr fn_ptr, ptr env_ptr }
%closure_t = type { ptr, ptr }
```

### 57.3.2 Environment Record

```cpp
llvm::StructType *IREmitter::buildEnvType(ClosureExpr *closure) {
  llvm::SmallVector<llvm::Type*> captureTypes;
  for (auto *capture : closure->captures())
    captureTypes.push_back(toLLVM(capture->type()));
  return llvm::StructType::get(ctx, captureTypes);
}

// Heap-allocate environment
llvm::Value *IREmitter::buildEnv(ClosureExpr *closure) {
  auto *envTy = buildEnvType(closure);
  auto *mallocFn = mod->getOrInsertFunction("malloc",
    llvm::FunctionType::get(builder.getPtrTy(),
                             {builder.getInt64Ty()}, false));
  const llvm::DataLayout &DL = mod->getDataLayout();
  uint64_t size = DL.getTypeAllocSize(envTy);
  auto *env = builder.CreateCall(mallocFn, {builder.getInt64(size)}, "env");

  // Store captured values into environment
  unsigned i = 0;
  for (auto *capture : closure->captures()) {
    llvm::Value *capturedVal = emitExpr(capture->sourceExpr());
    llvm::Value *fieldPtr    = builder.CreateStructGEP(envTy, env, i++);
    builder.CreateStore(capturedVal, fieldPtr);
  }
  return env;
}
```

### 57.3.3 Closure Function

The lifted function takes an extra `ptr` argument (the environment):

```cpp
llvm::Function *IREmitter::buildClosureFn(ClosureExpr *closure) {
  // Build parameter types: env_ptr first, then declared params
  llvm::SmallVector<llvm::Type*> paramTypes = {builder.getPtrTy()};
  for (auto &param : closure->params())
    paramTypes.push_back(toLLVM(param.type()));

  auto *fty = llvm::FunctionType::get(toLLVM(closure->retType()), paramTypes, false);
  auto *fn  = llvm::Function::Create(fty, llvm::Function::InternalLinkage,
                                      "closure", mod.get());
  fn->setName(closure->uniqueName()); // unique per-closure name

  auto *entry = llvm::BasicBlock::Create(ctx, "entry", fn);
  builder.SetInsertPoint(entry);

  // Unpack captures from environment
  llvm::Value *envPtr = fn->arg_begin(); // first argument
  auto *envTy = buildEnvType(closure);
  unsigned i = 0;
  for (auto *capture : closure->captures()) {
    llvm::Value *fieldPtr = builder.CreateStructGEP(envTy, envPtr, i++);
    llvm::Value *val      = builder.CreateLoad(toLLVM(capture->type()), fieldPtr);
    namedValues[capture->name()] = /* store in alloca for mutability */ ...;
  }

  // Emit body
  emitStmt(closure->body());
  return fn;
}
```

### 57.3.4 Calling a Closure

```cpp
// Given closure value %cl = { ptr fn_ptr, ptr env_ptr }:
llvm::Value *fnPtr  = builder.CreateExtractValue(closureVal, {0}, "fn");
llvm::Value *envPtr = builder.CreateExtractValue(closureVal, {1}, "env");
// Call: fn_ptr(env_ptr, arg1, arg2, ...)
llvm::SmallVector<llvm::Value*> args = {envPtr};
args.append(userArgs);
builder.CreateCall(closureFTy, fnPtr, args, "closure.call");
```

## 57.4 Tagged Unions (Sum Types)

A tagged union (Rust `enum`, Haskell ADT, ML sum type) maps to an LLVM struct containing:
1. A **tag** field (integer discriminant identifying the variant).
2. A **payload** field (a union-sized byte array large enough for the largest variant).

```
; enum Shape { Circle { radius: f64 }, Rect { w: f64, h: f64 } }
;
; Largest payload = Rect = { f64, f64 } = 16 bytes, align 8
;
; %Shape = type { i32, [16 x i8] }
;                  ^tag  ^payload
```

### 57.4.1 Building the Type

```cpp
llvm::StructType *IREmitter::buildTaggedUnionType(EnumDecl *decl) {
  const llvm::DataLayout &DL = mod->getDataLayout();
  uint64_t maxSize = 0; uint64_t maxAlign = 1;
  for (auto *variant : decl->variants()) {
    auto *payTy = buildVariantPayloadType(variant);
    maxSize  = std::max(maxSize, DL.getTypeAllocSize(payTy));
    maxAlign = std::max(maxAlign, (uint64_t)DL.getABITypeAlign(payTy).value());
  }
  // Tag is i32; payload is [maxSize x i8] with explicit alignment
  llvm::Type *tagTy  = builder.getInt32Ty();
  llvm::Type *payTy  = llvm::ArrayType::get(builder.getInt8Ty(), maxSize);
  auto *sty = llvm::StructType::create(ctx, {tagTy, payTy}, decl->name());
  return sty;
}
```

### 57.4.2 Construction

```cpp
llvm::Value *IREmitter::emitVariantConstruct(VariantConstructExpr *expr) {
  auto *unionAlloca = builder.CreateAlloca(getTaggedUnionType(expr->enumDecl()));

  // Write tag
  llvm::Value *tagPtr = builder.CreateStructGEP(getTaggedUnionType(expr->enumDecl()),
                                                  unionAlloca, 0, "tag.ptr");
  builder.CreateStore(builder.getInt32(expr->variantIndex()), tagPtr);

  // Write payload
  llvm::Value *payPtr = builder.CreateStructGEP(getTaggedUnionType(expr->enumDecl()),
                                                  unionAlloca, 1, "pay.ptr");
  auto *payTy = buildVariantPayloadType(expr->variantDecl());
  // Bitcast payload region to variant struct type
  for (unsigned i = 0; i < expr->fields().size(); ++i) {
    llvm::Value *fieldPtr = builder.CreateStructGEP(payTy, payPtr, i);
    builder.CreateStore(emitExpr(expr->fields()[i].get()), fieldPtr);
  }
  return builder.CreateLoad(getTaggedUnionType(expr->enumDecl()), unionAlloca);
}
```

### 57.4.3 Pattern Matching (Match/Switch)

```cpp
void IREmitter::emitMatchStmt(MatchStmt *match) {
  llvm::Value *taggedVal = emitExpr(match->scrutinee());
  llvm::Value *tag = builder.CreateExtractValue(taggedVal, {0}, "tag");

  auto *fn      = builder.GetInsertBlock()->getParent();
  auto *exitBB  = llvm::BasicBlock::Create(ctx, "match.exit");
  auto *sw      = builder.CreateSwitch(tag, exitBB, match->arms().size());

  for (auto &arm : match->arms()) {
    auto *armBB = llvm::BasicBlock::Create(ctx, "arm." + arm.variantName, fn);
    sw->addCase(builder.getInt32(arm.variantIndex), armBB);

    builder.SetInsertPoint(armBB);
    // Extract payload and bind pattern variables
    llvm::Value *payPtr = builder.CreateExtractValue(taggedVal, {1}, "pay");
    auto *payTy = buildVariantPayloadType(arm.variantDecl);
    for (unsigned i = 0; i < arm.bindings.size(); ++i) {
      llvm::Value *fieldPtr = builder.CreateStructGEP(payTy, payPtr, i);
      llvm::Value *field    = builder.CreateLoad(payTy->getElementType(i), fieldPtr);
      auto *bindAlloca = createEntryAlloca(fn, field->getType(), arm.bindings[i]);
      builder.CreateStore(field, bindAlloca);
      namedValues[arm.bindings[i]] = bindAlloca;
    }
    emitStmt(arm.body.get());
    if (!builder.GetInsertBlock()->getTerminator())
      builder.CreateBr(exitBB);
  }

  fn->insert(fn->end(), exitBB);
  builder.SetInsertPoint(exitBB);
}
```

## 57.5 Coroutines

LLVM has first-class coroutine support via the `@llvm.coro.*` intrinsic family. Frontends define coroutines (suspend/resume generators, `async`/`await` patterns) by annotating functions with these intrinsics; the `CoroSplit` and `CoroElide` passes handle the lowering.

### 57.5.1 Coroutine Lifecycle Intrinsics

```llvm
; Coroutine frame allocation and management
%id   = call token @llvm.coro.id(i32 0, ptr null, ptr null, ptr null)
%size = call i64 @llvm.coro.size.i64()
%alloc = call ptr @malloc(i64 %size)
%frame = call ptr @llvm.coro.begin(token %id, ptr %alloc)

; Suspend point
%tok = call token @llvm.coro.save(ptr null)
%suspend = call i8 @llvm.coro.suspend(token %tok, i1 false)  ; false = regular suspend
switch i8 %suspend, label %suspend [i8 0, label %resume
                                     i8 1, label %cleanup]

; Cleanup and final suspension
suspend:  %r = call i1 @llvm.coro.end(ptr null, i1 false, token none)
          unreachable
cleanup:  call void @llvm.coro.free(token %id, ptr %frame)
          ret void
```

A frontend that implements `yield`/`await` generates this scaffolding around the coroutine body. The `CoroSplitPass` splits the coroutine into the *ramp function* (allocates the frame, runs until first suspend) and *resume* and *destroy* functions. `CoroElidePass` eliminates heap allocation when the coroutine frame's lifetime is bounded (the common stack-allocated case).

### 57.5.2 Frontend Emission Pattern

```cpp
void IREmitter::emitCoroutineFn(FnDecl *fn) {
  // Emit the coroutine function with proper intrinsics
  auto *func = llvm::Function::Create(fty, llvm::Function::ExternalLinkage,
                                       fn->name, mod.get());
  auto *entry = llvm::BasicBlock::Create(ctx, "entry", func);
  builder.SetInsertPoint(entry);

  // Coroutine id and size intrinsics
  auto *coroId = llvm::Intrinsic::getOrInsertDeclaration(
    mod.get(), llvm::Intrinsic::coro_id,
    {builder.getInt32Ty(), builder.getPtrTy(),
     builder.getPtrTy(), builder.getPtrTy()});
  llvm::Value *id = builder.CreateCall(coroId,
    {builder.getInt32(0),
     llvm::ConstantPointerNull::get(builder.getPtrTy()),
     llvm::ConstantPointerNull::get(builder.getPtrTy()),
     llvm::ConstantPointerNull::get(builder.getPtrTy())},
    "coro.id");

  // … frame allocation, body emission with suspend points …
}
```

See the LLVM coroutine design document and `llvm/docs/Coroutines.rst` for the full lowering protocol.

## 57.6 String Literals

String literals are global constant byte arrays with a null terminator:

```cpp
llvm::Value *IREmitter::emitStringLiteral(std::string_view s) {
  // Build null-terminated array constant
  llvm::SmallVector<llvm::Constant*> chars;
  for (unsigned char c : s)
    chars.push_back(builder.getInt8(c));
  chars.push_back(builder.getInt8(0)); // null terminator

  auto *strTy   = llvm::ArrayType::get(builder.getInt8Ty(), chars.size());
  auto *strConst = llvm::ConstantArray::get(strTy, chars);

  // Emit as a private constant global
  auto *gv = new llvm::GlobalVariable(*mod, strTy, /*isConstant=*/true,
               llvm::GlobalValue::PrivateLinkage, strConst, ".str");
  gv->setUnnamedAddr(llvm::GlobalValue::UnnamedAddr::Global);
  gv->setAlignment(llvm::Align(1));

  // Decay to ptr (GEP with 0 offset)
  return llvm::ConstantExpr::getInBoundsGetElementPtr(
    strTy, gv, ArrayRef<llvm::Constant*>{builder.getInt64(0), builder.getInt64(0)});
}
```

`setUnnamedAddr(Global)` allows the linker to merge identical string constants across translation units (equivalent to `linkonce_odr` without the symbol). `PrivateLinkage` keeps the symbol file-local.

For wide strings (`wchar_t*`, `char16_t*`, `char32_t*`), use `getInt16Ty()` or `getInt32Ty()` elements and a 2- or 4-byte null terminator.

## 57.7 RTTI Emission

Runtime type information supports `dynamic_cast` and `typeid`. Under the Itanium C++ ABI, RTTI is a set of global structs defined in `<typeinfo>`:

```llvm
; __fundamental_type_info for int
@_ZTIi = external constant ptr  ; imported from libstdc++/libc++

; __class_type_info for a leaf class (no bases)
@_ZTI4Base = constant { ptr, ptr } {
  ptr @_ZTVN10__cxxabiv117__class_type_infoE + 16,  ; typeinfo vtable ptr
  ptr @_ZTS4Base                                      ; type name string ptr
}

; __si_class_type_info for single-inheritance derived class
@_ZTI7Derived = constant { ptr, ptr, ptr } {
  ptr @_ZTVN10__cxxabiv120__si_class_type_infoE + 16,
  ptr @_ZTS7Derived,
  ptr @_ZTI4Base                                     ; base class RTTI
}
```

A frontend emitting Itanium-ABI C++ must generate these globals for every class with virtual functions. For a language that does not use the C++ ABI, the RTTI format can be custom — any pointer-sized discriminator that the runtime `dynamic_cast` implementation understands.

```cpp
llvm::Constant *IREmitter::buildRTTI(ClassDecl *cls) {
  auto *name = emitStringLiteral(cls->mangledName()); // type name

  if (cls->baseClasses().empty()) {
    // __class_type_info
    auto *rttiTy = llvm::StructType::get(ctx, {builder.getPtrTy(), builder.getPtrTy()});
    return new llvm::GlobalVariable(*mod, rttiTy, true,
             llvm::GlobalValue::ExternalLinkage,
             llvm::ConstantStruct::get(rttiTy, {classTypeInfoVtable(cls), name}),
             "_ZTI" + cls->mangledName());
  }
  // … __si_class_type_info / __vmi_class_type_info for inheritance …
}
```

## 57.8 Putting It Together: Lowering Complex Types

The following table summarizes the IR lowering for each construct:

| Source construct | LLVM IR lowering |
|----------------|-----------------|
| Struct | `StructType` + `getelementptr` |
| Array access | GEP with `[0, idx]` + optional bounds check |
| Virtual call | Load vtable ptr → GEP to slot → load fn ptr → `call` |
| Closure | `{ ptr fn, ptr env }` + lifted function with env param |
| Tagged union | `{ i32 tag, [N x i8] payload }` + `extractvalue`/`insertvalue` |
| Pattern match | `extractvalue` tag → `switch` on tag → payload GEP per arm |
| Coroutine | `@llvm.coro.*` intrinsics + `CoroSplitPass` |
| String literal | `PrivateLinkage` constant `[N x i8]` + GEP decay to ptr |
| RTTI | Itanium `__*_type_info` structs + type name string |

---

## Chapter Summary

- Struct field access lowers to `getelementptr inbounds` with a struct type and a constant field index.
- Arrays use `getelementptr` with a dynamic index; bounds-safe languages should emit an `icmp`/`br` guard before each access.
- Virtual dispatch requires a vtable global (array of function pointers with Itanium prefix slots), vtable pointer stored at object offset 0, and a load-GEP-load-call sequence.
- Closures lift to an environment record (heap-allocated struct of captured values) plus a function pointer that takes the environment as its first argument.
- Tagged unions use a discriminant + byte-array payload, with pattern matching implemented as `switch` on the tag followed by typed GEPs into the payload region.
- Coroutines use the `@llvm.coro.*` intrinsic family; `CoroSplitPass` and `CoroElidePass` handle frame splitting and heap elision.
- String literals are `PrivateLinkage` constant `[N x i8]` globals, decayed to `ptr` via a zero-offset GEP.
- RTTI under the Itanium ABI is a hierarchy of `__*_type_info` structs; custom languages can use any pointer-sized discriminator their runtime understands.
