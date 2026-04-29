# Chapter 52 — ClangIR Architecture

*Part VIII — ClangIR (CIR)*

ClangIR (CIR) is a high-level MLIR dialect that interposes between the Clang AST and LLVM IR. Where CodeGenModule collapses C++ abstractions into flat SSA immediately, CIR keeps them: stack slots remain named allocas, structured loops remain loop ops, exceptions remain try/catch regions. The payoff is a level of representation that supports source-level analyses—lifetime checking, idiom recognition, C++ ABI-independent optimizations—without re-implementing them on top of the already-lowered LLVM IR. This chapter explains why CIR exists, how it slots into the compilation pipeline, and how the MLIR foundation enables analyses that the classic Clang-to-LLVM path cannot easily express.

## 52.1 Motivation and Design Goals

The traditional Clang pipeline has a representation gap. Between the typed, structured, C++-aware AST and the untyped, flat LLVM IR there is no persistent intermediate form. Any analysis that needs source structure—e.g., "does this pointer escape its lexical scope?"—either runs entirely on the AST (where all implicit code is not yet emitted) or on LLVM IR (where scopes, names, and C++ semantics have been erased). Many important transformations fall into the gap.

CIR closes it by providing an MLIR-based IR at the C/C++ source level:

- **Structured types**: `cir.int<s, 32>`, `cir.ptr<cir.int<s, 32>>`, `cir.record<…>` correspond directly to C/C++ types. No `i32` vs `i64` confusion across platforms.
- **Structured control flow**: `cir.if`, `cir.for`, `cir.while`, `cir.switch` mirror the source. Basic-block lowering happens at a later stage.
- **Source locations**: MLIR's `Location` infrastructure attaches exact `FileLineColLoc` to every operation.
- **C++-aware ops**: `cir.call` tracks whether a call uses the virtual-dispatch path. `cir.scope` models a C++ block scope. `cir.try`/`cir.catch` model exception handling at source fidelity.

### 52.1.1 Comparison with Classic CodeGen

| Dimension | Classic CodeGen (Ch39–44) | CIR |
|-----------|--------------------------|-----|
| Control flow | Immediate lowering to basic blocks | Structured ops (`cir.for`, `cir.while`, `cir.if`) |
| Types | LLVM `i32`, `ptr` | `cir.int<s,32>`, `cir.ptr<T>` |
| Stack allocation | `alloca` at entry block | `cir.alloca` tied to lexical scope |
| Exception handling | `landingpad`/`invoke` | `cir.try`/`cir.catch` |
| Analysis opportunity | Scalar opts, alias analysis | Lifetime, escape, idiom recognition |
| Persistence | Discarded after codegen | Serializable MLIR module |

### 52.1.2 Origin and Upstreaming

CIR was initiated by Bruno Cardoso Lopes and Nathan Lanza at Meta in 2021. The initial out-of-tree development established the dialect, a partial CIRGenModule/CIRGenFunction covering C and basic C++, and a CIR-to-LLVM lowering path. Upstreaming into the LLVM monorepo began in earnest in 2024. As of LLVM 22.1, CIR is compiled into the in-tree Clang when `CLANG_ENABLE_CIR=ON` is set at CMake time, but is **not** enabled in distribution builds by default. The flag `-fclangir` activates the CIR pipeline; `-fno-clangir` explicitly requests the classic path.

```bash
# Requires a -DCLANG_ENABLE_CIR=ON build
clang -fclangir -emit-cir -o hello.cir hello.c
clang -fclangir -S -o hello.s hello.c
clang -fno-clangir -S -o hello_classic.s hello.c
```

## 52.2 The CIR Dialect in MLIR Terms

CIR is a first-class MLIR dialect registered with the `cir` namespace. Its definition lives in:

- [`clang/include/clang/CIR/Dialect/IR/CIRDialect.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CIR/Dialect/IR/CIRDialect.h)
- TableGen ODS sources in `clang/include/clang/CIR/Dialect/IR/CIROps.td`, `CIRTypes.td`, `CIRAttrs.td`

The dialect implements `mlir::DialectInterface` and registers its types, attributes, operations, interfaces, and passes through the standard MLIR registration mechanisms.

### 52.2.1 Types

CIR defines its own type hierarchy rather than reusing MLIR builtin types, because it must carry C/C++ semantics:

```
// Primitive integer types — track signedness and exact width
cir.int<s, 8>    // signed 8-bit   (char on LP64)
cir.int<u, 8>    // unsigned 8-bit
cir.int<s, 32>   // signed 32-bit  (int on LP64)
cir.int<s, 64>   // signed 64-bit  (long on LP64)
cir.int<s, 128>  // signed 128-bit (__int128)

// IEEE floating-point types
cir.float         // float
cir.double        // double
cir.long_double   // x86 80-bit or quad, target-dependent

// Pointer — carries pointee type
cir.ptr<cir.int<s, 32>>

// Void type
cir.void

// Boolean
cir.bool

// Struct/union — identified by record name, members lazily resolved
cir.record<struct "Point" {!cir.int<s,32>, !cir.int<s,32>}>

// Array
cir.array<!cir.int<s,32> x 4>

// Function type
cir.func<(!cir.int<s,32>, !cir.int<s,32>) -> !cir.int<s,32>>
```

These types are TableGen-defined in `CIRTypes.td` and generated through the MLIR `TypeDef` mechanism. The C++ classes live in
[`clang/include/clang/CIR/Dialect/IR/CIRTypes.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CIR/Dialect/IR/CIRTypes.h).

`cir.int<s,32>` is importantly **different** from MLIR's `si32`: it carries the C `signed` qualifier and integrates with CIR's type inferencing. `cir.record` uses identified-struct semantics, allowing recursive types (linked lists, trees) without inlining.

### 52.2.2 Operations

CIR operations are grouped by category. The full set is defined in `CIROps.td`; key representatives are:

**Module-level ops:**
```
cir.func @add(%a: !cir.int<s,32>, %b: !cir.int<s,32>) -> !cir.int<s,32> {
  %result = cir.binop(add, %a, %b) : !cir.int<s,32>
  cir.return %result : !cir.int<s,32>
}

cir.global external @g_counter : !cir.int<s,32>
```

**Control flow ops (structured):**
```
cir.if %cond : !cir.bool {
  // then region
} else {
  // else region
}

cir.for : {
  // condition region
  cir.condition(%cond)
} body {
  // body region
  cir.yield continue
} step {
  cir.yield
}

cir.while : {
  cir.condition(%cond)
} do {
  cir.yield
}

cir.switch (%val : !cir.int<s,32>) {
  case (equal, 0)   { cir.yield fallthrough }
  case (equal, 1)   { cir.return %one : !cir.int<s,32> }
  case (default)    { cir.break }
}
```

**Memory ops:**
```
// Alloca — typed, carries source name for debug fidelity
%p = cir.alloca !cir.int<s,32>, init, ["x"] {alignment = 4 : i64}

// Load/store — typed
%val = cir.load %p : !cir.ptr<!cir.int<s,32>>, !cir.int<s,32>
cir.store %val, %p : !cir.int<s,32>, !cir.ptr<!cir.int<s,32>>
```

**Arithmetic ops:**
```
%sum  = cir.binop(add,  %a, %b) nsw : !cir.int<s,32>
%diff = cir.binop(sub,  %a, %b) nsw : !cir.int<s,32>
%prod = cir.binop(mul,  %a, %b) nsw : !cir.int<s,32>
%quot = cir.binop(div,  %a, %b) : !cir.int<s,32>
%rem  = cir.binop(rem,  %a, %b) : !cir.int<s,32>
%shl  = cir.shift(left, %a : !cir.int<s,32>, %b : !cir.int<u,32>) -> !cir.int<s,32>
```

The `nsw`/`nuw` flags on `cir.binop` carry C++ undefined-behavior semantics, allowing downstream passes to apply the same optimizations that LLVM performs on `add nsw`.

**Cast ops:**
```
%u = cir.cast(int_to_bool, %n : !cir.int<s,32>) : !cir.bool
%l = cir.cast(integral, %i : !cir.int<s,32>) : !cir.int<s,64>
%p = cir.cast(array_to_ptrdecay, %a : !cir.ptr<!cir.array<!cir.int<s,8> x 10>>) : !cir.ptr<!cir.int<s,8>>
```

**Exception handling ops:**
```
cir.try {
  // try body
  cir.yield
} catch [#cir.catch_info<type_info @_ZTIi, "e">, ...] {
  // catch handler
}
```

### 52.2.3 Attributes

CIR attributes carry C/C++ metadata that does not fit in types or operations:

- `cir.GlobalLinkageKind` — mirrors `llvm::GlobalValue::LinkageTypes`
- `cir.CallingConv` — C, x86_fastcall, Win64, etc.
- `cir.IntAttr` — integer constant (APInt backed)
- `cir.FPAttr` — floating-point constant (APFloat backed)
- `cir.ZeroAttr` — zero-initialization for any CIR type

Defined in [`CIRAttrs.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CIR/Dialect/IR/CIRAttrs.h).

## 52.3 Pipeline Position

CIR slots between the Clang AST consumer and the LLVM backend:

```
Source
  │
  ▼
Clang Frontend (Lexer → Parser → Sema)
  │
  ▼
Clang AST
  │
  ▼  CIRGenerator (ASTConsumer)
CIR Module (MLIR)
  │
  ├─ CIR-to-CIR passes (canonicalize, flatten-cfg, simplify,
  │                      cxx-abi-lowering, lowering-prepare)
  │
  ▼
CIR Module (lowered)
  │
  ├─ Direct: cir.direct.convertCIRToLLVM → LLVM dialect → LLVM IR
  │
  └─ Through-MLIR: CIR → (other MLIR dialects) → LLVM dialect → LLVM IR
       │
       ▼
LLVM IR → optimization → MC → binary
```

The `CIRGenerator` class (an `ASTConsumer`) replaces `CodeGenerator` when `-fclangir` is active. It drives `CIRGenModule`, which walks top-level declarations and produces a `mlir::ModuleOp` populated with CIR ops. After generation, the CIR-to-CIR pass pipeline runs, followed by lowering.

The `CIRGenAction` frontend action (see [`CIRGenAction.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CIR/FrontendAction/CIRGenAction.h)) provides five output modes:

```cpp
enum class OutputType {
  EmitAssembly,   // -S
  EmitCIR,        // -emit-cir
  EmitLLVM,       // -emit-llvm
  EmitBC,         // -emit-llvm -x ir
  EmitObj,        // -c (default)
};
```

The `-emit-cir` flag writes the CIR module in MLIR's text format, which is human-readable and round-trippable with `mlir-opt`.

## 52.4 The CIR Pass Pipeline

CIR-to-CIR passes run before lowering and are registered through MLIR's pass infrastructure. The primary pre-lowering passes are declared in
[`clang/include/clang/CIR/Dialect/Passes.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CIR/Dialect/Passes.h):

```cpp
std::unique_ptr<Pass> createCIRCanonicalizePass();
std::unique_ptr<Pass> createCIRFlattenCFGPass();
std::unique_ptr<Pass> createCIRSimplifyPass();
std::unique_ptr<Pass> createCXXABILoweringPass();
std::unique_ptr<Pass> createHoistAllocasPass();
std::unique_ptr<Pass> createLoweringPreparePass();
std::unique_ptr<Pass> createGotoSolverPass();

void populateCIRPreLoweringPasses(mlir::OpPassManager &pm);
```

These run collectively via `runCIRToCIRPasses()` from
[`CIRToCIRPasses.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CIR/CIRToCIRPasses.h):

```cpp
mlir::LogicalResult runCIRToCIRPasses(mlir::ModuleOp theModule,
                                       mlir::MLIRContext &mlirCtx,
                                       clang::ASTContext &astCtx,
                                       bool enableVerifier,
                                       bool enableCIRSimplify);
```

**`CIRCanonicalizePass`** applies MLIR canonicalization patterns: dead-code elimination, constant folding, algebraic simplifications.

**`CIRFlattenCFGPass`** converts structured control flow ops (`cir.if`, `cir.for`, `cir.while`) to basic-block form with `cir.br`/`cir.brcond`, producing a CFG-oriented CIR that is easier to lower to LLVM IR.

**`CIRSimplifyPass`** performs source-level simplifications—e.g., collapsing adjacent `cir.scope` regions, removing empty catch blocks.

**`CXXABILoweringPass`** handles C++ ABI specifics: vtable emission, RTTI construction, `__cxa_` runtime call insertion. This pass knows about C++ semantics without yet emitting LLVM IR.

**`HoistAllocasPass`** moves `cir.alloca` ops to function entry, mirroring what classic CodeGen does by default, in preparation for mem2reg.

**`LoweringPreparePass`** finalizes CIR-level constructs that have no direct LLVM equivalent: `cir.complex` arithmetic, `cir.vec_shuffle`, type conversions for aggregates.

**`GotoSolverPass`** converts `cir.goto` / `cir.label` pairs to structured jumps where possible, and emits MLIR-level branch ops otherwise.

Passes can be disabled with `-clangir-disable-passes` for debugging; verification can be disabled with `-clangir-disable-verifier`.

## 52.5 Lowering to LLVM

Two lowering paths exist, both declared in
[`clang/include/clang/CIR/Passes.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CIR/Passes.h):

```cpp
namespace cir::direct {
  std::unique_ptr<mlir::Pass> createConvertCIRToLLVMPass();
  void populateCIRToLLVMPasses(mlir::OpPassManager &pm);
}
```

**Direct lowering** (`cir.direct`): a single conversion pass translates CIR operations one-for-one into the MLIR LLVM dialect, then `mlir::translateModuleToLLVMIR` converts the MLIR LLVM dialect to an `llvm::Module`. The API entry point is:

```cpp
namespace cir::direct {
  std::unique_ptr<llvm::Module>
  lowerDirectlyFromCIRToLLVMIR(mlir::ModuleOp mlirModule,
                                llvm::LLVMContext &llvmCtx);
}
```

**Through-MLIR lowering**: CIR ops are progressively lowered through other MLIR dialects (e.g., `memref`, `scf`, `affine`) before reaching the LLVM dialect. This path enables the full MLIR ecosystem—polyhedral analysis, vectorization—to operate on C/C++ code without leaving the MLIR framework. This path is the longer-term direction; as of LLVM 22.1 it covers less of the CIR op surface than the direct path.

## 52.6 Tooling and Debugging

Because CIR is a standard MLIR dialect, the full MLIR toolchain applies:

```bash
# Emit CIR in text form
clang -fclangir -emit-cir -o hello.cir hello.c

# Round-trip and verify
mlir-opt hello.cir --verify-diagnostics

# Apply a specific pass
mlir-opt hello.cir --cir-canonicalize --cir-flatten-cfg -o hello_flat.cir

# Dump after each pass (MLIR debug flag)
clang -fclangir -mllvm -debug-only=cir-pipeline -S hello.c
```

The `-clangir-disable-verifier` flag suppresses MLIR module verification, useful when working on an incomplete CIR lowering of a new language feature.

The `MissingFeatures` utility in
[`clang/include/clang/CIR/MissingFeatures.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CIR/MissingFeatures.h) is CIR's "TODO tracker in code": each unimplemented feature is a static function returning `false`. When a feature lands, flipping it to `true` triggers compile-time failures at all call sites, ensuring completeness.

```cpp
struct MissingFeatures {
  static bool addressSpace()         { return false; }
  static bool opGlobalThreadLocal()  { return false; }
  static bool opLoadStoreAtomic()    { return false; }
  // … 100+ entries …
};
```

Call sites pattern-match against this:
```cpp
if (MissingFeatures::opLoadStoreAtomic())
  llvm_unreachable("atomic load/store not yet implemented in CIR");
```

## 52.7 Upstreaming Status (LLVM 22.1)

The following table summarizes CIR coverage as of LLVM 22.1.3:

| Feature | Status |
|---------|--------|
| C scalar types, arithmetic | Complete |
| C control flow (if/for/while/switch) | Complete |
| C functions, globals | Complete |
| C pointers, aggregates (struct/union) | Mostly complete |
| C++ virtual dispatch | Partial |
| C++ exceptions | Partial |
| C++ templates | Relies on instantiation before CIRGen |
| Atomics | Incomplete (MissingFeatures entry) |
| Thread-local variables | Incomplete |
| Inline assembly | Not implemented |
| OpenMP/OpenACC | Early work |
| Lifetime checker | Prototype |
| HLSL via CIR | Early work |

The project tracks missing coverage explicitly through `MissingFeatures` entries and a public tracking spreadsheet. Contributions to close these gaps are the primary call to action for CIR.

---

## Chapter Summary

- **CIR** is an MLIR dialect for C/C++ that preserves source structure between the Clang AST and LLVM IR.
- It provides typed ops (`cir.binop`, `cir.call`, `cir.try`), structured control flow (`cir.for`, `cir.while`, `cir.if`), and C++-aware types (`cir.int<s,32>`, `cir.record`, `cir.ptr`).
- Activated with `-fclangir`; requires a `CLANG_ENABLE_CIR=ON` build. `-emit-cir` dumps the CIR module in MLIR text form.
- The pipeline is: AST → `CIRGenerator` → CIR module → CIR-to-CIR passes → lowering (direct or through-MLIR) → LLVM IR.
- Pre-lowering passes include canonicalize, flatten-cfg, simplify, C++ ABI lowering, hoist-allocas, and goto-solver.
- Two lowering paths: direct CIR→LLVM-dialect conversion, and a through-MLIR path enabling the broader MLIR ecosystem.
- `MissingFeatures.h` provides a compile-time tracking mechanism for unimplemented features.
- As of LLVM 22.1, CIR covers C and basic C++ but has gaps in atomics, TLS, inline assembly, and OpenMP.


---

@copyright jreuben11
