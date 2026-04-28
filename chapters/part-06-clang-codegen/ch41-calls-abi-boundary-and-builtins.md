# Chapter 41 — Calls, the ABI Boundary, and Builtins

*Part VI — Clang Internals: Codegen and ABI*

Every function call in a C or C++ program crosses two distinct frontiers: the language-level calling convention specified by the programmer (or implied by default), and the ABI-level calling convention imposed by the platform. These frontiers are not the same. A `struct { int x; int y; }` appears in the source as a single aggregate argument, but on x86-64 System V the two integer fields are packed into one 64-bit register and passed as `i64` in LLVM IR. A 40-byte return type that fits into two SSE registers in the caller's perspective becomes an invisible first `sret` pointer parameter at the LLVM IR level. Clang's codegen layer mediates this translation entirely, and understanding it requires mastering three interlocking layers: the `ABIArgInfo` classification system, the `CGFunctionInfo` ABI-lowered signature descriptor, and the target-specific `ABIInfo` subclasses that embody each platform's rules. Above those sits the call-emission machinery in `CodeGenFunction`, which marshals arguments, inserts sret pointers, coerces types, and ultimately emits a `call` or `invoke` instruction. Below the call level, the builtin-lowering subsystem (`EmitBuiltinExpr`) translates `__builtin_*` intrinsics, C11 atomics, and `__sync_*` operations into their LLVM counterparts. This chapter covers the complete path from source-level call expression to emitted IR, including variadic function lowering, function pointer handling, and the block/ObjC message dispatch model.

---

## 41.1 The ABI Abstraction Layer

### 41.1.1 Source-Level Declaration vs ABI-Lowered Form

A Clang `FunctionDecl` carries the source-level type: parameter types as the programmer wrote them, return type as declared. When Clang must emit or call this function, it first transforms this source signature into an ABI-lowered signature — one where aggregates may be split into scalar components, small structs may be coerced to integer or vector types, large return types gain an implicit first pointer parameter, and integer arguments may acquire zero- or sign-extend attributes. The resulting descriptor is a `CGFunctionInfo`, and the transformation is performed by target-specific logic encapsulated in the `ABIInfo` class hierarchy.

The key insight is that LLVM IR knows nothing about platform ABIs. LLVM IR calling conventions (C, Fast, Cold, X86_64SysV, Win64, AAPCS, etc.) determine register allocation policy for the backend register allocator, but they do not encode which fields of a struct land in which registers. That knowledge lives entirely in Clang's codegen layer and is expressed through type coercion: Clang deliberately emits function signatures using IR types that differ from the Clang-type-to-IR-type mapping, and the backend then assigns those IR types to registers according to the IR calling convention.

### 41.1.2 `ABIInfo` Base Class

`ABIInfo`, declared in [`clang/lib/CodeGen/ABIInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/ABIInfo.h), is the abstract interface that all target ABI implementations satisfy. Its primary virtual method is:

```cpp
class ABIInfo {
public:
  CodeGen::CodeGenTypes &CGT;
  ABIInfo(CodeGen::CodeGenTypes &cgt) : CGT(cgt) {}
  virtual ~ABIInfo() = default;

  // Classify how a given argument or return type is passed.
  virtual void computeInfo(CGFunctionInfo &FI) const = 0;

  // Emit target-specific code to load a va_arg from a va_list.
  virtual RValue EmitVAArg(CodeGenFunction &CGF, Address VAListAddr,
                           QualType Ty, AggValueSlot Slot) const = 0;
};
```

`computeInfo()` receives a mutable `CGFunctionInfo` reference and fills in the `ABIArgInfo` descriptor for each argument and the return type. Every concrete `ABIInfo` subclass overrides this method to implement its platform's argument classification algorithm. The second virtual method, `EmitVAArg()`, handles variadic argument extraction and is called by the `va_arg` lowering path.

### 41.1.3 `TargetCodeGenInfo`

`TargetCodeGenInfo`, declared in [`clang/lib/CodeGen/TargetInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/TargetInfo.h), is a broader per-target customization point that owns the `ABIInfo` subclass instance and adds hooks for stack probe generation, exception table emission, OpenCL address space mapping, and platform-specific function attributes. `CodeGenModule` holds a single `TargetCodeGenInfo` instance obtained from `createTargetCodeGenInfo()`:

```cpp
// clang/lib/CodeGen/TargetInfo.cpp
std::unique_ptr<TargetCodeGenInfo>
createTargetCodeGenInfo(CodeGenModule &CGM) {
  const TargetInfo &Target = CGM.getTarget();
  const llvm::Triple &Triple = Target.getTriple();
  switch (Triple.getArch()) {
  case llvm::Triple::x86_64:
    // Choose between SysV AMD64 and Win64 depending on the OS
    if (Triple.isOSWindows() && Triple.isWindowsMSVCEnvironment())
      return createWinX86_64TargetCodeGenInfo(CGM, X86AVXABILevel::None);
    return createX86_64TargetCodeGenInfo(CGM, X86AVXABILevel::None);
  case llvm::Triple::aarch64:
    return createAArch64TargetCodeGenInfo(CGM, AArch64ABIKind::AAPCS);
  case llvm::Triple::riscv64:
    return createRISCVTargetCodeGenInfo(CGM, 64, /*FLen=*/64);
  // ... many more targets
  }
}
```

---

## 41.2 `ABIArgInfo` — Classification Outcomes

`ABIArgInfo`, defined in [`clang/include/clang/CodeGen/CGFunctionInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CodeGen/CGFunctionInfo.h), encodes what the ABI classifier decided to do with a single argument or return value. Its `Kind` enum is the central vocabulary:

### 41.2.1 Kind Taxonomy

| Kind | Meaning |
|------|---------|
| `Direct` | Pass using the normal converted LLVM type, optionally coerced to a different IR type via `CoerceToType` |
| `Extend` | Like `Direct` but also attach `zeroext` or `signext` to the IR parameter |
| `Indirect` | Pass via a caller-allocated pointer; corresponds to `byval` attribute in IR when `getIndirectByVal()` is true |
| `IndirectAliased` | Like `Indirect` but the caller vouches the pointed-to object is not otherwise modified during the call; emits `byref` |
| `Ignore` | The argument carries no information (empty structs, `void`); emit nothing |
| `Expand` | Aggregate expanded field-by-field into consecutive IR parameters |
| `CoerceAndExpand` | Aggregate expanded into the non-padding fields of the struct type stored in `CoerceToType`, one IR param per non-padding element |
| `InAlloca` | Argument passed via the LLVM `inalloca` mechanism — lives in a caller-allocated memory block pointed to by the last parameter |
| `TargetSpecific` | RISC-V tuple types and other architecture-specific argument handling |

### 41.2.2 Key Accessors

The `ABIArgInfo` accessors are kind-gated and assert on misuse:

- `getCoerceToType()` / `setCoerceToType(llvm::Type*)` — valid for `Direct`, `Extend`, `CoerceAndExpand`, `TargetSpecific`. Returns the IR type to use in place of the default lowered type.
- `getIndirectAlign()` — valid for `Indirect`, `IndirectAliased`. Returns the alignment as `CharUnits`.
- `getIndirectByVal()` — valid for `Indirect`. True means emit the `byval` attribute on the pointer parameter.
- `getPaddingType()` — valid for most kinds. An IR type inserted as a dummy parameter preceding the real argument to achieve register-file alignment. Typically used on x86-32 for x87 FP return alignment.
- `getDirectOffset()` — valid for `Direct`, `Extend`, `TargetSpecific`. A byte offset into a coerced struct type at which the actual data begins; used when a small struct is coerced to a larger integer but only the low bytes carry data.
- `getCoerceAndExpandType()` / `getUnpaddedCoerceAndExpandType()` — valid for `CoerceAndExpand`. The former gives the full struct IR type with padding; the latter gives the flat (padding-stripped) type sequence.

Factory methods follow the same kind-naming pattern:

```cpp
// Clang source: clang/include/clang/CodeGen/CGFunctionInfo.h
static ABIArgInfo getDirect(llvm::Type *T = nullptr,
                            unsigned Offset = 0,
                            llvm::Type *Padding = nullptr,
                            bool CanBeFlattened = true);
static ABIArgInfo getIndirect(CharUnits Alignment, unsigned AddrSpace,
                               bool ByVal = true, bool Realign = false,
                               llvm::Type *Padding = nullptr);
static ABIArgInfo getIgnore();
static ABIArgInfo getExpand();
static ABIArgInfo getCoerceAndExpand(llvm::StructType *coerceToType,
                                     llvm::Type *unpaddedCoerceToType);
static ABIArgInfo getInAlloca(unsigned FieldIndex, bool Indirect = false);
```

---

## 41.3 `CGFunctionInfo` — The ABI-Lowered Signature

`CGFunctionInfo`, defined alongside `ABIArgInfo` in `CGFunctionInfo.h`, is the complete ABI description of one function signature. It is a trailing-objects node stored in a `llvm::FoldingSet` (keyed by the profile of the canonical argument types, calling convention, and a set of boolean flags), so identical signatures are deduplicated and the same `CGFunctionInfo` is reused for all calls to functions with that signature.

### 41.3.1 Structure

`CGFunctionInfo` stores, as trailing objects, an array of `CGFunctionInfoArgInfo` structs:

```cpp
struct CGFunctionInfoArgInfo {
  CanQualType type;     // Clang canonical type
  ABIArgInfo info;      // ABI classification outcome
};
```

The zeroth slot holds the return type; slots 1 through `NumArgs` hold parameters. The `arg_begin()` / `arg_end()` iterators return the parameters only (skipping slot 0), while `getReturnInfo()` returns a reference to slot 0's `ABIArgInfo` and `getReturnType()` returns the corresponding `CanQualType`.

### 41.3.2 Calling Convention Fields

`CGFunctionInfo` tracks two calling conventions:

- `CallingConvention` — the `llvm::CallingConv::ID` value derived from the AST-level `CallingConv` attribute (e.g., `CC_C`, `CC_X86StdCall`, `CC_Win64`).
- `EffectiveCallingConvention` — the actual LLVM CC to use after ABI transformation. For most System V targets this equals `CallingConvention`, but for functions that must use the Windows calling convention on non-Windows targets (e.g., an extern "C" function attributed `__declspec(dllexport)` in a cross-compilation context) they may differ. The `setEffectiveCallingConvention()` method allows `ABIInfo` subclasses to override the default.

Additional boolean flags encoded in bit-fields cover `NoReturn`, `ReturnsRetained` (ObjC ARC), `NoCallerSavedRegs`, `HasRegParm`, `NoCfCheck`, `CmseNSCall` (ARM Cortex-M security extension), and `DelegateCall` (used for thunk forwarding).

### 41.3.3 Arrangement Entry Points

`CodeGenTypes`, the class that drives type-lowering and ABI classification, provides multiple overloads for constructing `CGFunctionInfo` instances. The most commonly invoked are:

```cpp
// clang/lib/CodeGen/CodeGenTypes.h
const CGFunctionInfo &arrangeFunctionDeclaration(const FunctionDecl *FD);
const CGFunctionInfo &arrangeCXXMethodDeclaration(const CXXMethodDecl *MD);
const CGFunctionInfo &arrangeFreeFunctionCall(CanQualType returnType,
                                              ArrayRef<CanQualType> argTypes,
                                              FunctionType::ExtInfo info,
                                              RequiredArgs args);
const CGFunctionInfo &arrangeBuiltinFunctionCall(CanQualType resultType,
                                                 const CallArgList &args);
const CGFunctionInfo &arrangeObjCMessageSendSignature(
    const ObjCMethodDecl *MD, QualType receiverType);
```

Each variant constructs a canonical `CGFunctionInfo` key, looks it up in the folding set, and if not found calls `ABIInfo::computeInfo()` to fill in all the `ABIArgInfo` slots. The result is then cached indefinitely until the `CodeGenTypes` object is destroyed at the end of the translation unit.

---

## 41.4 Target-Specific ABI Implementations

### 41.4.1 x86-64 System V (`X86_64ABIInfo`)

The System V AMD64 ABI, implemented in `X86_64ABIInfo` ([`clang/lib/CodeGen/Targets/X86.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/Targets/X86.cpp)), classifies each eight-byte unit of a type into one of several machine classes: INTEGER (general-purpose registers), SSE (XMM registers), SSEUP (upper half of YMM/ZMM), X87/X87UP (x87 FPU), COMPLEX_X87, NO_CLASS, or MEMORY.

`X86_64ABIInfo::classify()` is the recursive classification function. For primitive types: integers and pointers fall into INTEGER; `float`, `double`, and `_Float16` fall into SSE; `long double` gets X87 + X87UP; `__m128`/`__m256`/`__m512` get SSE/SSEUP. For aggregates: each eight-byte unit is assigned the merge of all field classes overlapping it, with the following combination rules: MEMORY dominates everything; X87/X87UP/COMPLEX_X87 cause the whole aggregate to be MEMORY if paired with any other non-empty class; SSE + INTEGER yields SSE; otherwise the higher-priority class wins.

After classification:

- Aggregates with total size > 2 × eight-bytes (16 bytes) are MEMORY → `Indirect` with `byval`.
- Aggregates fitting in one or two eight-byte units use `Direct` with a coerce type: if both units are INTEGER, the coerce type is `{ i64 }` or `{ i64, i64 }`; if the first is SSE and second INTEGER, it is `{ double, i64 }`, etc.
- Return types follow similar rules but MEMORY returns use `sret` (an implicit first pointer parameter with `sret` attribute and `dead_on_unwind noalias writable` IR attributes).

The concrete IR produced for a two-field int struct:

```c
// Source: struct Small { int x; int y; };
// struct Small add_small(struct Small a, struct Small b);

// IR (x86-64 SysV): both struct args coerced to i64
define dso_local i64 @add_small(i64 %0, i64 %1) {
  ; alloca %struct.Small for each, store i64 args, operate field-by-field
  store i64 %0, ptr %4, align 4    ; unpack first arg from i64 register
  store i64 %1, ptr %5, align 4    ; unpack second arg
  ; ... field operations ...
  %18 = load i64, ptr %3, align 4  ; repack result into i64
  ret i64 %18
}
```

A 40-byte struct (five doubles) exceeds the two-eight-byte-unit limit and triggers sret:

```c
// Source: struct BigResult make_big(double x);
// IR: sret pointer added as first parameter
define dso_local void @make_big(
    ptr dead_on_unwind noalias writable sret(%struct.BigResult) align 8 %0,
    double noundef %1) {
  // writes through %0 directly; no return value
  ret void
}
// Call site:
call void @make_big(ptr dead_on_unwind writable sret(%struct.BigResult) align 8 %3,
                    double noundef %4)
```

A struct passed with `byval` (too large for register passing, used as by-value argument):

```c
// Source: void take_large(struct Large l);  // Large = { int[10] }
// IR: byval pointer
define dso_local void @take_large(
    ptr noundef byval(%struct.Large) align 8 %0) {
  ret void
}
```

### 41.4.2 Windows x86-64 (`WinX86_64ABIInfo`)

`WinX86_64ABIInfo`, also in `Targets/X86.cpp`, implements the Microsoft x64 calling convention. The classification is simpler: each argument occupies exactly one register or stack slot regardless of structure size. Structs of size 1, 2, 4, or 8 bytes are passed by value in a general-purpose register (coerced to the appropriately-sized integer type); all other structs are passed as a pointer to a caller-allocated copy. Return values follow the same rule. The main consequential difference from SysV is that XMM registers and GP registers are not combined for struct passing — a struct that SysV would split across an INTEGER and an SSE unit is passed by pointer in Win64.

### 41.4.3 AArch64 AAPCS64 (`AArch64ABIInfo`)

`AArch64ABIInfo`, in [`clang/lib/CodeGen/Targets/AArch64.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/Targets/AArch64.cpp), implements AAPCS64. Its key specialties:

- **Homogeneous Floating-Point Aggregates (HFAs)**: A struct or union consisting of 1–4 identical floating-point or short-vector members is an HFA. HFAs are passed with each member in a separate `v` register, using `CoerceAndExpand` with a struct type whose elements are the member types.
- **Composites**: Non-HFA aggregates up to 16 bytes are passed in 1–2 general-purpose registers (coerced to `i64` or `{ i64, i64 }`). Larger composites use `Indirect` without `byval` — the caller passes a pointer but AAPCS64 does not require byval semantics; instead the callee is responsible for not modifying the pointed-to memory (this corresponds to `IndirectAliased` in Clang's classification).
- **Pointer authentication**: On platforms with ARMv8.3-A PAC (Apple Silicon, etc.), `AArch64ABIInfo` and `TargetCodeGenInfo` cooperate to attach `ptrauth` bundle operands to function-pointer call sites. This is handled through `CGPointerAuthInfo` which encodes the key, discriminator type (address-discriminated vs. constant), and the discriminator value.

### 41.4.4 ARM 32-bit (`ARMABIInfo`)

`ARMABIInfo`, in [`clang/lib/CodeGen/Targets/ARM.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/Targets/ARM.cpp), selects among APCS, AAPCS, and AAPCS-VFP sub-ABIs (the `-mfloat-abi=soft/softfp/hard` axis). Under AAPCS-VFP, HFAs of up to four floats or doubles use the VFP co-processor registers (via `CoerceAndExpand`). Under soft-float, all FP values are passed in core integer registers by coercing to the corresponding integer width. ARM's ABI also defines the `__va_list` type layout differently from x86-64.

### 41.4.5 RISC-V (`RISCVABIInfo`)

`RISCVABIInfo`, in [`clang/lib/CodeGen/Targets/RISCV.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/Targets/RISCV.cpp), implements the RISC-V psABI. Key rules: scalars up to 2×XLEN bits fit in one or two integer registers; structs with a single floating-point member and at most one integral member may use the FP calling convention (one FP register + optionally one integer register). Structs larger than 2×XLEN are passed via pointer. RISC-V vector types use `TargetSpecific` to handle the scalable-vector tuple types (e.g., `vint32m2_t`) which have no fixed-size IR representation.

`RISCVABIInfo::classifyArgumentType()` first checks for scalable vector types (RVV registers) and returns `TargetSpecific`. For non-vector types it applies the LP64/LP64F/LP64D sub-ABI rules: if the total size fits within 2×XLEN and the type is "trivially passable" (no non-trivial constructors in C++), it is `Direct` with a coerce type; otherwise `Indirect` with byval and the appropriate alignment. The FP calling convention (LP64F, LP64D, LP64Q) additionally considers single-float and double-float fields for split register pairs using `CoerceAndExpand`.

### 41.4.6 Calling Convention Table

| Platform | Struct ≤ 2 regs | Struct > 2 regs | Float HFA | Return > 2 regs |
|----------|----------------|-----------------|-----------|-----------------|
| x86-64 SysV | `Direct` (coerce to `i64` / `{i64,i64}`) | `Indirect` byval | `Direct` (SSE) | sret |
| Win64 | `Direct` if ≤ 8B, else `Indirect` | `Indirect` (no byval) | N/A | sret if > 8B |
| AArch64 AAPCS64 | `Direct` (`i64`/`{i64,i64}`) | `IndirectAliased` | `CoerceAndExpand` | sret |
| ARM AAPCS-VFP | `Direct` / `CoerceAndExpand` | `Indirect` byval | `CoerceAndExpand` | sret |
| RISC-V LP64D | `Direct` / `CoerceAndExpand` | `Indirect` byval | `CoerceAndExpand` | sret |

---

## 41.5 `EmitCall()` and Argument Marshaling

### 41.5.1 `CallArgList` Assembly

Before calling `EmitCall()`, the codegen for a `CallExpr` builds a `CallArgList` — a `SmallVector<CallArg, 16>` where each `CallArg` holds either an `RValue` (already-evaluated scalar or aggregate address) or a `LValue` with a copy-requirement flag. Arguments are appended by `CodeGenFunction::EmitCallArg()`:

```cpp
// clang/lib/CodeGen/CGCall.cpp
void CodeGenFunction::EmitCallArg(CallArgList &Args, const Expr *E,
                                  QualType type) {
  if (type->isReferenceType()) {
    // pass address of lvalue
    Args.add(RValue::get(EmitLValue(E).getPointer(*this)), type);
    return;
  }
  // For non-trivial C++ types, may need copy construction
  if (hasAggregateEvaluationKind(type)) {
    Args.addUncopiedAggregate(EmitLValue(E), type);
    return;
  }
  Args.add(EmitAnyExprToTemp(E), type);
}
```

Special handling applies for `__pass_object_size` parameters, ARC-managed ObjC object parameters (which may need `objc_retain`), and `_Nonnull` pointer parameters that trigger sanitizer checks.

### 41.5.2 `EmitCall()` Signature

The central call-emission entry point is:

```cpp
// clang/lib/CodeGen/CGCall.h
RValue CodeGenFunction::EmitCall(const CGFunctionInfo &CallInfo,
                                 const CGCalleeInfo &Callee,
                                 ReturnValueSlot ReturnValue,
                                 const CallArgList &Args,
                                 llvm::CallBase **callOrInvoke = nullptr,
                                 bool IsMustTail = false,
                                 SourceLocation Loc = SourceLocation(),
                                 bool IsVirtualFunctionPointerThunk = false);
```

`CGCalleeInfo` wraps the callee — either a `llvm::Value*` (for indirect calls and function pointers) or a `GlobalDecl` (for direct calls where the function is known). `ReturnValueSlot` tells `EmitCall()` where to materialize an aggregate return value in the caller's stack frame.

### 41.5.3 Argument Coercion

For each argument, `EmitCall()` consults the corresponding `ABIArgInfo`:

**Direct with coerce type**: If the source type's IR representation differs from `CoerceToType`, a bitcast or sequence of loads/stores converts between the two. For example, passing a `{ float, float }` struct as an `i64` involves storing the struct to a 4-byte-aligned alloca, then loading 8 bytes as `i64`. The helper `CoerceIntOrPtrToIntOrPtr()` handles the common case of reinterpreting between integer types and pointer types.

**Indirect (byval)**: The argument value is materialized in memory (if it is not already an lvalue), and its address is passed. If `getIndirectByVal()` is false, the pointer is passed without byval, relying on caller-side no-alias analysis instead.

**Expand**: The aggregate is recursively expanded by `expandTypeToArgs()`, which recursively visits fields and appends each scalar field as a separate IR argument.

**CoerceAndExpand**: The aggregate is stored to an alloca, then each non-padding element is loaded individually and appended to the IR argument list.

**InAlloca**: The argument is not appended individually; it was pre-placed in the inalloca struct by `EmitCXXMemberCallExpr()` or similar C++ emission paths.

**Ignore**: Nothing is appended.

### 41.5.4 sret Insertion

If `getReturnInfo().isIndirect()`, `EmitCall()` prepends the sret pointer to the argument list. If there is an explicit `this` pointer (C++ member call), the sret is inserted after `this` per the Itanium ABI rule that sret comes second in member functions. The sret address is either the `ReturnValueSlot` address (if the caller provided a slot) or a newly allocated alloca.

### 41.5.5 `EmitCallOrInvoke()`

With the argument list assembled, `EmitCallOrInvoke()` decides between `llvm::CallInst` and `llvm::InvokeInst`. A `call` is emitted if there are no active `EHCleanup` scopes or the callee is declared `noexcept`. An `invoke` is emitted otherwise, with the unwind destination being the innermost cleanup or catch handler's landing pad. The `musttail` keyword, when present (from `[[clang::musttail]]`), is set on the `CallInst` via `CallInst::setTailCallKind(llvm::CallInst::TCK_MustTail)`.

After emission, `EmitCallOrInvoke()` runs the `CGM.getTargetCodeGenInfo().setTargetAttributes()` hook to attach any final target-specific function call attributes. It also sets `callOrInvoke` (the out-parameter) so callers can attach additional metadata (profiling weights, `!nosanitize`, `!callsite` metadata for profile instrumentation).

### 41.5.6 Argument Evaluation Order and Side Effects

`CallArgList` assembly interacts with C/C++ sequencing rules. For C, arguments have unsequenced evaluation; for C++17, arguments are indeterminately sequenced (not unsequenced, so each argument is fully evaluated before the next, but order is unspecified). `CodeGenFunction::EmitCallArgs()` evaluates all non-variadic arguments left-to-right in practice, since each evaluation produces an `RValue` or `LValue` which is stable (stored in a temp) before the next argument begins. Variadic arguments require `EmitVariadicCallArgs()`, which additionally ensures that `va_list`-consuming arguments are evaluated after all fixed arguments, matching `RequiredArgs::getNumRequiredArgs()`.

---

## 41.6 Return Value Handling

### 41.6.1 sret: Large Aggregate Returns

When the return `ABIArgInfo` is `Indirect`, the return value resides at the sret address. After `EmitCallOrInvoke()` returns, the `call` instruction has type `void` and the aggregate is already written to the sret alloca. `EmitCall()` then returns an `RValue` wrapping the sret address.

If `ReturnValueSlot` provided a pre-existing address (e.g., the destination of an assignment expression or a named return value optimization (NRVO) candidate), the sret is pointed directly at it, eliminating a copy. The NRVO path is specifically gated by `ReturnValueSlot::isExternallyDestructed()` checking whether the slot can be used for direct construction.

### 41.6.2 Direct Return with Coercion

When the return `ABIArgInfo` is `Direct` with a non-null `CoerceToType`, the IR function returns the coerce type, not the source type. `EmitCall()` stores the returned IR value to a typed alloca and loads it back as the source type:

```cpp
// Pseudo-code: recovering a {float, float} from an i64 return
llvm::Value *RetCI = Builder.CreateCall(Fn, IRArgs);  // returns i64
llvm::AllocaInst *Alloca = CreateMemTemp(RetTy);      // { float, float } alloca
Builder.CreateStore(RetCI, Alloca);                   // store i64 → reinterpret
// load field by field or as aggregate RValue
```

The `CreateCoercedLoad()` helper in `CGCall.cpp` performs this pattern generically, handling both same-size and different-size coercions by first storing the IR return value to a correctly-aligned alloca and then loading the desired type.

### 41.6.3 Scalar and Vector Returns

For `Extend` returns, Clang attaches `zeroext` or `signext` to the return value attribute, instructing the backend to guarantee that the upper bits of the return register contain the proper extension. The `EmitCall()` epilogue wraps such returns in a `trunc` instruction to recover the declared width.

---

## 41.7 `EmitFunctionProlog()` and `EmitFunctionEpilog()`

### 41.7.1 Parameter Binding in `EmitFunctionProlog()`

`CodeGenFunction::EmitFunctionProlog()`, in `CGCall.cpp`, processes the incoming IR parameters against the `CGFunctionInfo` to bind each source parameter name to an `llvm::Value` or `Address` in `LocalDeclMap`.

For `Direct` or `Extend` parameters where the IR type matches the Clang type exactly (no coercion), the IR parameter is stored directly or used as-is. For `Direct` with coercion (`CoerceToType != nullptr`), the code:

1. Allocates an appropriately-typed alloca for the source-level parameter.
2. Stores the incoming coerced IR value to the alloca.
3. Optionally bitcasts the alloca pointer and loads the source type.

For `Indirect` parameters, the IR parameter is already a pointer to the source type in the caller's stack frame. Clang binds the source parameter to that pointer directly (no copy) unless the parameter has a declared alignment that exceeds what byval guarantees, in which case it copies to a new alloca via `EmitAggregateCopy()`.

For `Indirect` parameters with `getIndirectRealign()` set, an aligned alloca is allocated and the incoming data is copied into it:

```cpp
if (Arg.info.isIndirect() && Arg.info.getIndirectRealign()) {
  llvm::AllocaInst *AlignedTemp = CreateMemTemp(Ty, "coerce");
  Builder.CreateMemCpy(AlignedTemp, /*src=*/V, Size, /*isVolatile=*/false);
  V = AlignedTemp;
}
```

For `Expand` parameters, `EmitFunctionProlog()` reverses the expansion: it allocates an alloca for the aggregate and stores the consecutive IR parameters into the corresponding field offsets.

The sret parameter, if present, is extracted as the first (or second, after `this`) IR parameter and stored into `CurFnInfo->getReturnType()`'s alloca. The `ReturnValue` address member of `CodeGenFunction` is set to point to it, so `EmitReturnStmt()` can locate it later.

### 41.7.2 Return in `EmitFunctionEpilog()`

`CodeGenFunction::EmitFunctionEpilog()`, also in `CGCall.cpp`, closes the function:

- **sret return**: The return value has already been constructed at the `ReturnValue` address. `EmitFunctionEpilog()` emits a bare `ret void`.
- **Direct return with coercion**: Loads the source-type value from `ReturnValue`, coerces it to `CoerceToType` (using `CreateCoercedLoad()`), and emits `ret <coerced-value>`.
- **Scalar Direct/Extend**: Loads the value, optionally extends it, and emits `ret <value>`.
- **Indirect (non-sret)**: Should not arise for return values in standard C/C++ — `Indirect` for return means sret, which is handled above.

---

## 41.8 `__builtin_*` Lowering

### 41.8.1 `EmitBuiltinExpr()` Dispatch

`CodeGenFunction::EmitBuiltinExpr()`, in [`clang/lib/CodeGen/CGBuiltin.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGBuiltin.cpp), is an enormous switch over `BuiltinID` values defined in `clang/include/clang/Basic/Builtins.td`. The switch covers several hundred cases. When called for a `CallExpr` whose callee is a builtin, the function returns an `RValue`; if it returns `std::nullopt`, the call falls through to normal function-call lowering (used for builtins that have a fallback library implementation).

### 41.8.2 Memory Builtins

```c
// Source
void *__builtin_memcpy(void *dst, const void *src, size_t n);
void *__builtin_memset(void *dst, int c, size_t n);
void *__builtin_memmove(void *dst, const void *src, size_t n);
```

These lower to the corresponding LLVM memory intrinsics:

```llvm
; __builtin_memcpy(dst, src, n)  →
call void @llvm.memcpy.p0.p0.i64(ptr align 1 %dst, ptr align 1 %src,
                                  i64 %n, i1 false)

; __builtin_memset(dst, c, n)  →
call void @llvm.memset.p0.i64(ptr align 1 %dst, i8 %c_trunc, i64 %n, i1 false)
```

The alignment and `isVolatile` arguments use the type's known alignment when available (extracted from `CallExpr` argument types through `getBuiltinAlignmentForType()`) or `align 1` as a conservative fallback. For constant-size copies, the LLVM optimizer later lowers `llvm.memcpy` to register moves or store sequences.

### 41.8.3 Math Builtins

```c
double __builtin_sqrt(double x);    // → llvm.sqrt.f64
float  __builtin_sqrtf(float x);    // → llvm.sqrt.f32
double __builtin_fma(double a, double b, double c);  // → llvm.fma.f64
double __builtin_fabs(double x);    // → llvm.fabs.f64
double __builtin_ceil(double x);    // → llvm.ceil.f64
double __builtin_floor(double x);   // → llvm.floor.f64
double __builtin_round(double x);   // → llvm.round.f64
double __builtin_trunc(double x);   // → llvm.trunc.f64
```

The mapping is direct — `EmitBuiltinExpr()` calls `EmitNounwindRuntimeCall()` with the corresponding LLVM intrinsic name and the evaluated arguments. The `nounwind` marker ensures these intrinsics do not introduce exception edges. In `-ffast-math` mode, an `!fpmath` metadata node or function-level attributes such as `reassoc` and `contract` may be added to allow further floating-point transformations.

```llvm
; __builtin_fma(a, b, c)  →
%result = call double @llvm.fma.f64(double %a, double %b, double %c)
```

### 41.8.4 Overflow Builtins

The GNU overflow builtins map to LLVM's `with.overflow` intrinsic family:

```c
bool __builtin_add_overflow(T a, T b, T *result);  // signed or unsigned
bool __builtin_sub_overflow(T a, T b, T *result);
bool __builtin_mul_overflow(T a, T b, T *result);
```

Selection of signed vs unsigned intrinsic depends on whether `T` is a signed type:

```llvm
; int overflow: __builtin_add_overflow(a, b, result)
%pair   = call { i32, i1 } @llvm.sadd.with.overflow.i32(i32 %a, i32 %b)
%ovf    = extractvalue { i32, i1 } %pair, 1
%sum    = extractvalue { i32, i1 } %pair, 0
store i32 %sum, ptr %result
%ret    = zext i1 %ovf to i32
```

For unsigned types, `@llvm.uadd.with.overflow.*` is selected. For mixed-sign cases (e.g., `bool __builtin_add_overflow(unsigned a, int b, long *result)`), `EmitBuiltinExpr()` converts all operands to the widest unsigned type, uses the unsigned intrinsic, and truncates/checks manually.

### 41.8.5 `__builtin_object_size` and `__builtin_dynamic_object_size`

```c
size_t __builtin_object_size(const void *ptr, int type);
```

This lowers to `@llvm.objectsize.i64.p0` with the `min`/`max` and `NullIsUnknown` immarg arguments derived from the `type` argument:

```llvm
%sz = call i64 @llvm.objectsize.i64.p0(
    ptr %ptr,
    i1 false,   ; false = max (type & 2 == 0)
    i1 true,    ; NullIsUnknown
    i1 false)   ; dynamic = false (builtin_object_size vs dynamic variant)
```

The optimizer's `ObjectSizeOffsetVisitor` (in `llvm/lib/Analysis/MemoryBuiltins.cpp`) attempts to fold this to a constant at optimization time. If folding fails, the result is conservatively `-1` (max of `size_t`) for `type == 0` or `0` for `type == 1`.

### 41.8.6 Architecture-Specific Builtins

For x86, `__builtin_ia32_*` builtins are generated by `clang/include/clang/Basic/BuiltinsX86.def`. Each entry maps a builtin name to an LLVM intrinsic (`__builtin_ia32_addps` → `@llvm.x86.sse.add.ps`) or to a code sequence using `CreateCall` with a hand-constructed signature. The `EmitX86BuiltinExpr()` function in `CGBuiltin.cpp` handles the x86-specific dispatch.

AArch64 SVE/NEON builtins (`__builtin_neon_*`, `__builtin_sve_*`) follow the same pattern via `EmitAArch64BuiltinExpr()`. RISC-V vector intrinsics use a generated dispatch table from `clang/include/clang/Basic/riscv_vector.td`.

### 41.8.7 `__builtin_expect` and `__builtin_unpredictable`

`__builtin_expect(expr, expected)` lowers by evaluating `expr` normally and attaching `!prof` branch weight metadata to any conditional branch that is driven by the result. The actual lowering happens lazily: `EmitBuiltinExpr()` calls `EmitBoolBuiltinExpr()` which returns the evaluated boolean with an attached `BranchWeights` annotation. The annotation propagates through `EmitBranchOnBoolExpr()` which attaches `!prof !{!"branch_weights", i32 likely_weight, i32 unlikely_weight}` to the generated `br` instruction.

`__builtin_assume(cond)` lowers to `@llvm.assume` — a zero-cost hint that the optimizer can use to prune unreachable paths:

```llvm
call void @llvm.assume(i1 %cond)
```

`__builtin_unreachable()` lowers to `unreachable` — an IR terminator that tells the optimizer the point is truly unreachable, enabling subsequent dead-code elimination.

### 41.8.8 `__builtin_prefetch`

`__builtin_prefetch(addr, rw, locality)` lowers to `@llvm.prefetch`:

```llvm
call void @llvm.prefetch.p0(ptr %addr, i32 %rw, i32 %locality, i32 1)
```

The fourth argument is the cache type: 1 for data cache, 0 for instruction cache. The optimizer is free to drop prefetch hints if they are deemed unprofitable, since `@llvm.prefetch` has no observable side effect.

---

## 41.9 `__sync_*` and `__atomic_*` Lowering

### 41.9.1 GCC `__sync_*` Builtins

The `__sync_*` family uses sequentially consistent (`seq_cst`) ordering for all operations. Clang lowers them to atomic IR instructions:

```c
// __sync_fetch_and_add(ptr, val) → atomicrmw add
int prev = __sync_fetch_and_add(ptr, val);
```

```llvm
%prev = atomicrmw add ptr %ptr, i32 %val seq_cst, align 4
```

`__sync_bool_compare_and_swap(ptr, expected, desired)` returns 1 if the swap occurred, 0 otherwise. It lowers to `cmpxchg` with extraction of the success bit:

```llvm
%pair   = cmpxchg ptr %ptr, i32 %expected, i32 %desired seq_cst seq_cst, align 4
%success = extractvalue { i32, i1 } %pair, 1
%ret    = zext i1 %success to i32
```

`__sync_val_compare_and_swap` returns the old value; it extracts element 0 instead of element 1.

The full `atomicrmw` operation set covers `add`, `sub`, `or`, `and`, `xor`, `nand`, `max`, `min`, `umax`, `umin`. `__sync_lock_test_and_set` lowers to `atomicrmw xchg`.

### 41.9.2 C11 `_Atomic` and `<stdatomic.h>`

The C11 and C++11 atomic operations use explicit memory-order arguments. Clang translates `memory_order_relaxed/consume/acquire/release/acq_rel/seq_cst` to LLVM's `monotonic/monotonic/acquire/release/acq_rel/seq_cst` ordering values. The primary emission functions are:

- `EmitAtomicLoad()` — emits a `load atomic` instruction with the specified ordering.
- `EmitAtomicStore()` — emits a `store atomic` instruction.
- `EmitAtomicRMW()` — emits an `atomicrmw` instruction.
- `EmitAtomicCmpXchgInstruction()` — emits a `cmpxchg` with two orderings (success and failure).

`AtomicInfo`, a helper class within `CGAtomic.cpp`, encapsulates alignment requirements and lock-free determination. For types wider than the platform's native lock-free width, `AtomicInfo::shouldUseLibcall()` returns true and the operation is lowered to a call to `__atomic_load_*` / `__atomic_store_*` / `__atomic_compare_exchange_*` from the GCC-compatible atomic runtime (provided by compiler-rt or libatomic).

```c
// C11 example:
// _Atomic int *p; atomic_load_explicit(p, memory_order_acquire);
```

```llvm
%val = load atomic i32, ptr %p acquire, align 4
```

```c
// atomic_store_explicit(p, v, memory_order_release);
```

```llvm
store atomic i32 %v, ptr %p release, align 4
```

For 128-bit compare-exchange on x86-64 (which requires the `cmpxchg16b` instruction), the LLVM backend lowers a 128-bit `cmpxchg` to `cmpxchg16b` if the target feature is available; otherwise it falls back to a lock-based library call.

### 41.9.3 `AtomicInfo` and Lock-Free Width

`AtomicInfo`, the helper class in `CGAtomic.cpp`, is constructed for every atomic operation site:

```cpp
// clang/lib/CodeGen/CGAtomic.cpp
class AtomicInfo {
  CodeGenFunction &CGF;
  QualType         AtomicTy;     // The _Atomic type
  QualType         ValueTy;      // The underlying value type
  uint64_t         AtomicSizeInBits;
  uint64_t         ValueSizeInBits;
  CharUnits        AtomicAlignment;
  CharUnits        ValueAlignment;
  TypeEvaluationKind EvaluationKind;
  bool             UseLibcall;
};
```

`AtomicInfo::shouldUseLibcall()` returns true when `AtomicSizeInBits > CGF.CGM.getContext().getMaxAtomicInlineWidth()`. The inline width is typically 64 on 32-bit targets and 128 on 64-bit targets with `cmpxchg16b`/`lse2`. When libcalls are needed, `CGAtomic.cpp` emits calls to `__atomic_load_N`, `__atomic_store_N`, `__atomic_exchange_N`, `__atomic_compare_exchange_N` — the compiler-rt / libatomic runtime functions. The `N` suffix is the byte width; memory order integers (0=relaxed, 2=acquire, 3=release, 4=acq_rel, 5=seq_cst) are passed as the final integer argument.

---

## 41.10 Variadic Functions

### 41.10.1 `va_list` Representation on x86-64

The System V AMD64 ABI defines `va_list` as:

```c
typedef struct __va_list_tag {
  unsigned gp_offset;          // bytes consumed from GP register save area (max 48)
  unsigned fp_offset;          // bytes consumed from FP register save area (max 176)
  void    *overflow_arg_area;  // pointer to next overflow argument on stack
  void    *reg_save_area;      // pointer to register save area (allocated by callee)
} va_list[1];
```

In LLVM IR this appears as:

```llvm
%struct.__va_list_tag = type { i32, i32, ptr, ptr }
```

### 41.10.2 `__builtin_va_start` and `__builtin_va_end`

`__builtin_va_start(ap, last)` lowers to:

```llvm
call void @llvm.va_start.p0(ptr %ap)
```

`@llvm.va_start` is a backend-known intrinsic; the backend fills the `reg_save_area` field with a pointer into the function's frame save area and initializes `gp_offset` and `fp_offset` to 0 (no registers consumed yet). `__builtin_va_end` similarly lowers to `@llvm.va_end.p0`.

### 41.10.3 `va_arg` Lowering via `X86_64ABIInfo::EmitVAArg()`

`EmitVAArg()` in `X86_64ABIInfo` generates the type-dependent extraction protocol. For an integer variadic argument:

1. Load `gp_offset` from the `va_list`.
2. If `gp_offset <= 48 - 8` (i.e., at least one GP register remains), load from `reg_save_area + gp_offset` and increment `gp_offset` by 8.
3. Otherwise, load from `overflow_arg_area` and advance `overflow_arg_area` by 8.

This produces the branchy sequence visible in the IR earlier, with the PHI node merging the register-lane and stack-lane results. For FP arguments, the corresponding FP register save area protocol uses `fp_offset` and increments by 16 (each XMM register occupies 16 bytes in the save area).

For aggregates classified as SSE+INTEGER, `EmitVAArg()` loads from both register save areas and assembles the struct from two separate loads, taking care of the byval/overflow fallback if either register file is exhausted.

### 41.10.4 ARM and AArch64 `va_arg`

On 32-bit ARM (AAPCS), `va_list` is a simple `char *`; `va_arg` is a pointer load and increment with alignment rounding. `ARMABIInfo::EmitVAArg()` handles the alignment by rounding the `va_list` pointer up before loading. On AArch64 (AAPCS64), `va_list` has a similar five-field structure to x86-64's (`__stack`, `__gr_top`, `__vr_top`, `__gr_offs`, `__vr_offs`) and `AArch64ABIInfo::EmitVAArg()` follows the AAPCS64 variadic argument protocol with separate GP and VR (SIMD/FP) register lanes.

### 41.10.5 `__builtin_va_copy`

`__builtin_va_copy(dst, src)` copies the `va_list` struct. On x86-64 this is simply:

```llvm
call void @llvm.va_copy.p0(ptr %dst, ptr %src)
```

The backend lowers `@llvm.va_copy` to a `memcpy` of the four-field struct.

---

## 41.11 Function Pointers and Indirect Calls

### 41.11.1 `CGCalleeInfo` and Callee Resolution

`CGCalleeInfo` is the callee descriptor passed to `EmitCall()`. It has three forms:

1. **Direct known callee**: A `GlobalDecl` identifying the function; `EmitCall()` looks up or emits the `llvm::Function` constant.
2. **Virtual callee**: A `GlobalDecl` plus a vtable slot address; `EmitCall()` loads the function pointer from the vtable via `EmitVirtualFunctionPointer()` and then performs an indirect call.
3. **Indirect callee**: A raw `llvm::Value *` of function pointer type; `EmitCall()` calls through it directly.

A fourth constructor accepts both a `llvm::FunctionCallee` (combining an `llvm::FunctionType *` and a `llvm::Value *`) for cases where the precise LLVM function type is known in advance, avoiding the need to derive it from the `CGFunctionInfo`.

### 41.11.2 Indirect Calls via `EmitIndirectCallee()`

For a `CallExpr` whose callee is a pointer-to-function expression, `CodeGenFunction::EmitCallee()` dispatches to `EmitIndirectCallee()`, which evaluates the callee expression as an rvalue. If the source type is a function pointer (not a block pointer or member function pointer), the result is cast to the expected LLVM function type and wrapped in a `CGCalleeInfo`. The call site IR looks like:

```llvm
; int call_indirect(int (*f)(int,int), int a, int b)
%7 = load ptr, ptr %4, align 8          ; load function pointer
%10 = call noundef i32 %7(i32 noundef %8, i32 noundef %9)  ; indirect call
```

The IR type on the `call` instruction is derived by `CGFunctionInfo::getLLVMFunctionType()` for the callee's declared type — not the actual function that may be called at runtime. The backend trusts the IR function type annotation for code generation.

### 41.11.3 Pointer Authentication on AArch64

On AArch64 targets with ARMv8.3 PAC support (controlled by `-fptrauth-calls`), function pointer calls emit a `call` with a `ptrauth` operand bundle. The bundle encodes the authentication key (`ia` key = 0, `ib` = 1, `da` = 2, `db` = 3) and discriminator. `CGPointerAuthInfo`, constructed from `CGM.getFunctionPointerAuthInfo()`, provides the key and discriminator:

```llvm
; PAC-authenticated indirect call
call i32 %fn(i32 %arg) [ "ptrauth"(i32 0, i64 %discriminator) ]
```

The backend instruction selector expands the authenticated call to `blraaz`, `braa`, or equivalent instructions.

### 41.11.4 Virtual Function Pointers

For virtual calls, `CodeGenFunction::EmitCXXMemberOrOperatorCall()` delegates to `EmitVirtualFunctionPointer()`, which loads the vtable pointer from the object, advances to the appropriate vtable slot (offset computed by the ABI layer — see Chapter 42), and loads the function pointer. The resulting call is indirect. The vtable slot address is also the "vptr" source for `[[assume_aligned]]` and devirtualization analysis.

### 41.11.5 `musttail` Call Lowering

`[[clang::musttail]]` forces the emitted `call` instruction to carry the `musttail` marker. Clang validates at Sema time that the callee's parameter count and types are compatible with the caller (for indirect calls, the callee pointer's function type must match), and that the return type is identical. In `EmitCall()`, the `IsMustTail` flag set by `EmitCallExpr()` causes:

```cpp
if (IsMustTail) {
  CI->setTailCallKind(llvm::CallInst::TCK_MustTail);
}
```

The `musttail` marker in IR:

```llvm
; int trampoline(int a, int b) with [[clang::musttail]]
%7 = musttail call noundef i32 @_Z8dispatchii(i32 noundef %5, i32 noundef %6)
ret i32 %7
```

The LLVM backend verifies that `musttail` constraints (same calling convention, same return type, no allocas that outlive the call) are satisfied before instruction selection; violations become compile-time errors.

---

## 41.12 ObjC Message Sends and Block Invocations

### 41.12.1 Block Invocations

A block variable has type `void (^)(args)` and in memory is represented as a pointer to a block literal struct:

```c
struct __block_literal {
  void *isa;                   // &_NSConcreteStackBlock or &_NSConcreteGlobalBlock
  int   flags;
  int   reserved;
  void (*invoke)(void *, ...); // function pointer: the block body
  struct __block_descriptor *descriptor;
  // captured variables follow...
};
```

`call_block(b, x)` lowers to: load the `invoke` function pointer from field index 3 of the block struct, then call it with the block pointer as the implicit first argument:

```llvm
%6 = getelementptr inbounds nuw %struct.__block_literal_generic, ptr %5, i32 0, i32 3
%8 = load ptr, ptr %6, align 8        ; load invoke pointer
call void %8(ptr noundef %5, i32 noundef %7)  ; pass block as first arg
```

`__block` variables that need capture by reference use `__block_byref` structs with a forwarding pointer; the block's `copy_helper` and `dispose_helper` functions manage their lifetime when blocks escape to the heap.

### 41.12.2 ObjC `objc_msgSend`

ObjC message sends `[obj selector:arg]` are lowered by `CGObjCRuntime::GenerateMessageSend()`. The runtime-specific subclass (`CGObjCGNU` for GNUstep, `CGObjCMac` for Apple's runtime) emits a call to `objc_msgSend` (or `objc_msgSend_stret` for large struct returns, `objc_msgSend_fpret` for x87 FP returns on i386):

```llvm
; [obj compute:x]
%sel = load ptr, ptr @OBJC_SELECTOR_REFERENCES_compute:
%r   = call i32 (ptr, ptr, ...) @objc_msgSend(ptr %obj, ptr %sel, i32 %x)
```

The variadic prototype for `objc_msgSend` — `id objc_msgSend(id, SEL, ...)` — means LLVM IR must use a variadic call signature, with all arguments following the selector typed as their actual argument types via explicit casts.

### 41.12.3 ARC Runtime Calls

With `-fobjc-arc`, `CodeGenFunction` inserts ARC runtime calls automatically at certain AST boundaries:

| ARC Operation | Runtime Call |
|---------------|-------------|
| Strong variable assignment | `objc_retain` |
| Variable going out of scope | `objc_release` |
| Returning a retained value | `objc_autoreleaseReturnValue` |
| Receiving retained return | `objc_retainAutoreleasedReturnValue` |
| Weak load | `objc_loadWeakRetained` |
| Weak store | `objc_storeWeak` |

These calls are emitted as ordinary `call` instructions to externally-declared functions. The ARC optimizer pass (`ObjCARCOpt`) in the LLVM middle-end can eliminate redundant retain/release pairs once the call graph is visible.

---

## Summary

- `ABIArgInfo` encodes how each function parameter or return value crosses the source-to-IR boundary: `Direct` (possibly with type coercion), `Extend` (with zero/sign extension), `Indirect`/`IndirectAliased` (pass-by-pointer), `Ignore`, `Expand`, `CoerceAndExpand`, or `InAlloca`.
- `CGFunctionInfo` is the complete ABI-lowered signature, stored in a folding set for deduplication; it is constructed by `CodeGenTypes::arrange*()` entry points which invoke `ABIInfo::computeInfo()`.
- Each platform provides a concrete `ABIInfo` subclass: `X86_64ABIInfo` applies eight-byte classification for System V AMD64; `WinX86_64ABIInfo` passes non-trivial structs via pointer; `AArch64ABIInfo` supports HFAs and pointer authentication; `ARMABIInfo` manages VFP/soft-float sub-ABIs; `RISCVABIInfo` handles the LP64 psABI and scalable vector tuples.
- `EmitCall()` drives argument marshaling per `ABIArgInfo`: coercing direct args, inserting sret pointers, expanding aggregates, choosing `call` vs `invoke`.
- `EmitFunctionProlog()` reverses the ABI encoding to bind IR parameters to source-level parameter `Address` values; `EmitFunctionEpilog()` coerces the return value back to the IR return type.
- `EmitBuiltinExpr()` dispatches `__builtin_*` calls to LLVM intrinsics: `llvm.memcpy`, `llvm.memset`, `llvm.sqrt`, `llvm.fma`, `llvm.sadd.with.overflow`, `llvm.objectsize`, and hundreds of arch-specific intrinsics.
- `__sync_*` lowers to `atomicrmw`/`cmpxchg` with `seq_cst`; C11 atomics lower to `load atomic`/`store atomic`/`atomicrmw`/`cmpxchg` with explicit orderings; wide atomics fall back to runtime libcall via `AtomicInfo`.
- Variadic argument passing on x86-64 requires the `va_list` four-field structure and generates register-vs-stack branch sequences; `EmitVAArg()` is target-specific.
- Indirect calls use `CGCalleeInfo` with a raw `llvm::Value*`; virtual calls load through vtable slots; `musttail` calls set `TCK_MustTail` on the IR `call` instruction; PAC-authenticated calls add a `ptrauth` operand bundle.
- Block invocations load the `invoke` function pointer from field 3 of the block literal and pass the block pointer as the first argument; ObjC `[obj msg]` compiles to `objc_msgSend` calls; ARC inserts retain/release calls at lifetime boundaries.

---

## Cross-References

- [Chapter 23 — Attributes, Calling Conventions, and the ABI](../part-04-llvm-ir/ch23-attributes-calling-conventions-and-the-abi.md) — LLVM IR-level calling convention attributes and parameter attributes that Clang emits.
- [Chapter 39 — CodeGenModule and CodeGenFunction](../part-06-clang-codegen/ch39-codegenmodule-and-codegenfunction.md) — The infrastructure classes within which `EmitCall()` and `EmitBuiltinExpr()` execute.
- [Chapter 40 — Lowering Statements and Expressions](../part-06-clang-codegen/ch40-lowering-statements-and-expressions.md) — How `CallExpr` is dispatched to `EmitCallExpr()` which feeds into `EmitCall()`.
- [Chapter 42 — C++ ABI Lowering: Itanium](../part-06-clang-codegen/ch42-cpp-abi-lowering-itanium.md) — How the Itanium C++ ABI extends these call-lowering mechanisms for virtual dispatch, constructors, and exceptions.
- [Chapter 43 — C++ ABI Lowering: Microsoft](../part-06-clang-codegen/ch43-cpp-abi-lowering-microsoft.md) — How `WinX86_64ABIInfo` and the MSVC ABI differ in struct return, member pointers, and `inalloca` usage.

## Reference Links

- [`clang/include/clang/CodeGen/CGFunctionInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CodeGen/CGFunctionInfo.h) — `ABIArgInfo` and `CGFunctionInfo` definitions.
- [`clang/lib/CodeGen/CGCall.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGCall.cpp) — `EmitCall()`, `EmitFunctionProlog()`, `EmitFunctionEpilog()`, argument coercion logic.
- [`clang/lib/CodeGen/CGBuiltin.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGBuiltin.cpp) — `EmitBuiltinExpr()` and all `__builtin_*` lowering cases.
- [`clang/lib/CodeGen/CGAtomic.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGAtomic.cpp) — `AtomicInfo`, `EmitAtomicLoad()`, `EmitAtomicStore()`.
- [`clang/lib/CodeGen/Targets/X86.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/Targets/X86.cpp) — `X86_64ABIInfo::classify()`, `WinX86_64ABIInfo`, `EmitVAArg()` for x86-64.
- [`clang/lib/CodeGen/Targets/AArch64.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/Targets/AArch64.cpp) — `AArch64ABIInfo`, HFA classification, PAC call lowering.
- [`clang/lib/CodeGen/Targets/ARM.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/Targets/ARM.cpp) — `ARMABIInfo`, VFP/soft-float sub-ABI selection.
- [`clang/lib/CodeGen/Targets/RISCV.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/Targets/RISCV.cpp) — `RISCVABIInfo`, scalable vector tuple handling.
- [System V AMD64 ABI specification](https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf) — §3.2.3 Parameter Passing.
- [AAPCS64 specification](https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst) — §6.4–6.9 Parameter Passing.
- [RISC-V psABI](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/blob/master/riscv-cc.adoc) — §Calling Convention.
