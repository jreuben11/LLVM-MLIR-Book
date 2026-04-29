# Chapter 54 — CIR Lowering and Analysis

*Part VIII — ClangIR (CIR)*

CIR generation (Chapter 53) produces a structured, type-rich MLIR module. That module must ultimately become LLVM IR — but the journey from CIR to LLVM IR is where CIR's value is extracted. This chapter covers the two lowering paths in detail, the CIR-to-CIR passes that run before lowering, and the analysis capabilities — lifetime checking, idiom recognition, ABI handling — that justify CIR's existence in the compilation pipeline.

## 54.1 The Pre-Lowering Pass Pipeline

Before lowering to LLVM IR, a sequence of CIR-to-CIR passes refines the module. These passes operate entirely within the CIR dialect, using MLIR's pass infrastructure. The sequence is assembled by `populateCIRPreLoweringPasses` and run by `runCIRToCIRPasses` (declared in
[`clang/include/clang/CIR/CIRToCIRPasses.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CIR/CIRToCIRPasses.h)):

```cpp
mlir::LogicalResult runCIRToCIRPasses(mlir::ModuleOp theModule,
                                       mlir::MLIRContext &mlirCtx,
                                       clang::ASTContext &astCtx,
                                       bool enableVerifier,
                                       bool enableCIRSimplify);
```

The passes run in order:

```
CIRCanonicalizePass
  └─ CIRSimplifyPass (if enabled)
CXXABILoweringPass
HoistAllocasPass
LoweringPreparePass
GotoSolverPass
CIRFlattenCFGPass  ← last; converts structured ops to basic-block form
```

### 54.1.1 CIRCanonicalizePass

This is a standard MLIR canonicalization pass scoped to CIR operations. It drives the `getCanonicalizationPatterns` method on each CIR op, applying:

- Constant folding: `cir.binop(add, #cir.int<3>, #cir.int<4>)` → `cir.const #cir.int<7>`
- Dead code elimination: unreachable `cir.scope` regions, unused `cir.alloca` ops
- Identity elimination: `cir.cast(integral, %x : !cir.int<s,32>) : !cir.int<s,32>` → `%x`
- Algebraic simplifications: `a * 1 → a`, `a + 0 → a`, `a & -1 → a`

These patterns are registered via MLIR's pattern rewriter framework — each CIR op class can declare its canonical rewrites in its `getCanonicalizationPatterns` static method.

### 54.1.2 CIRSimplifyPass

`CIRSimplifyPass` performs higher-level structural simplifications:

- **Empty scope removal**: collapses `cir.scope { cir.yield }` with no allocas or cleanup.
- **Redundant fallthrough removal**: `cir.switch` cases that fall through to `cir.break` are normalized.
- **Constant condition hoisting**: `cir.if #cir.bool<true> { A } else { B }` → region A, region B dead.
- **Loop-invariant detection**: prepares hoisting opportunities for `LoweringPreparePass`.

Enabled by default; suppressed with `-clangir-disable-passes`.

### 54.1.3 CXXABILoweringPass

This is the most complex pre-lowering pass. It handles C++ constructs that have no direct CIR expression because their representation depends on the target ABI (Itanium or Microsoft):

**Virtual dispatch lowering:**
```
// Before CXXABILoweringPass — a conceptual placeholder
cir.virtual_call %obj->method(%arg) : (…)

// After — Itanium path
%vptr_addr = cir.get_member %obj[0] : !cir.ptr<!cir.record<…>>
             -> !cir.ptr<!cir.ptr<!cir.ptr<!cir.func<…>>>>
%vptr = cir.load %vptr_addr : …
%slot_addr = cir.ptr_stride %vptr, %slot_idx : …
%slot = cir.load %slot_addr : …
%result = cir.call %slot(%obj, %arg) : …
```

**Constructor and destructor calls**: the pass inserts calls to the appropriate complete-object or base-object constructor/destructor variants, folding the Itanium `C1`/`C2`/`D0`/`D1`/`D2` structure into explicit `cir.call` ops.

**RTTI and type-info**: for `dynamic_cast` and `typeid`, the pass emits loads of vtable RTTI pointers and calls to `__cxa_dynamic_cast`.

**EH personality**: the pass annotates `cir.func` ops that contain exception-handling regions with the platform-specific EH personality attribute (`__gxx_personality_v0` on Itanium, `__CxxFrameHandler3` on Microsoft).

The pass accesses the live `clang::ASTContext&` passed through `runCIRToCIRPasses`, enabling it to query `RecordDecl` layout, virtual base information, and C++ ABI details without re-deriving them from CIR.

### 54.1.4 HoistAllocasPass

This pass lifts all `cir.alloca` ops to the function entry region, matching the behavior of classic CodeGen. After hoisting:

1. All allocas are at the entry region regardless of their lexical scope in the source.
2. `mem2reg`-style promotion is safe on all allocas (since all uses are post-domination of the allocation).
3. The lowering pass can emit LLVM `alloca` at the entry block without further transformation.

The pass respects `cir.scope` semantics: lifetime intrinsics (`cir.lifetime.start`/`cir.lifetime.end`) are inserted at the original scope boundaries, mirroring LLVM's `@llvm.lifetime.start`/`@llvm.lifetime.end` semantics.

### 54.1.5 LoweringPreparePass

`LoweringPreparePass` handles CIR constructs that require ASTContext information to lower correctly but do not yet generate final LLVM ops:

- **Complex arithmetic**: `cir.binop` on `cir.complex<T>` is split into real and imaginary component ops using `cir.complex.real`/`cir.complex.imag`.
- **Vector shuffles**: `cir.vec_shuffle` ops are normalized to forms that the LLVM dialect can represent.
- **Aggregate return**: functions returning `cir.record` types are transformed to use a hidden sret pointer argument if the target ABI requires it, using `ABIArgInfo` classification.
- **Variadic calls**: `cir.call` ops to variadic functions receive a `cir.va_list` argument annotation.
- **`__builtin_` calls**: CIR-level builtin placeholders are replaced with the appropriate CIR intrinsic ops.

The pass receives `clang::ASTContext*` to access target ABI classification (x86-64 SystemV, ARM AAPCS, etc.) for aggregate argument passing.

### 54.1.6 GotoSolverPass

C's `goto` statement creates jumps that may cross `cir.scope` boundaries, complicating cleanup insertion. `GotoSolverPass`:

1. Identifies all `cir.goto`/`cir.label` pairs in a function.
2. Computes the scope depth difference between the `goto` site and the `label` site.
3. Inserts any skipped cleanup calls (destructors) on the path from goto to label.
4. Lowers `cir.goto` and `cir.label` to `cir.br` targeting the appropriate MLIR block.

For `goto` jumps that enter a scope (undefined behavior in C++, valid in C), the pass can optionally warn or insert explicit skipped-initialization handling.

### 54.1.7 CIRFlattenCFGPass

The final pre-lowering pass converts all structured control-flow ops to basic-block form. After this pass:

- `cir.if` → two `cir.br` successors with `cir.brcond`
- `cir.for`, `cir.while`, `cir.do` → loop header/body/exit blocks with `cir.br`/`cir.brcond`
- `cir.switch` → a dispatch block with `cir.switch_flat` or a chain of `cir.brcond`
- `cir.scope` → inlined into the enclosing block sequence

The output is a CIR module in CFG form — no structured ops remain. This is the form consumed by both lowering paths.

## 54.2 Direct Lowering: CIR → LLVM Dialect

The direct lowering path (in `cir::direct`) is a single MLIR dialect conversion pass:

```cpp
namespace cir::direct {
  std::unique_ptr<mlir::Pass> createConvertCIRToLLVMPass();
  void populateCIRToLLVMPasses(mlir::OpPassManager &pm);
}
```

It uses MLIR's `ConversionTarget` and `TypeConverter` infrastructure:

```cpp
// Type conversion: CIR types → LLVM dialect types
mlir::LLVMTypeConverter typeConverter(&context);
typeConverter.addConversion([](cir::IntType t) -> mlir::Type {
  return mlir::IntegerType::get(t.getContext(), t.getWidth());
});
typeConverter.addConversion([](cir::PointerType t, ...) -> mlir::Type {
  return mlir::LLVM::LLVMPointerType::get(t.getContext(), /*addrspace=*/0);
});
// … more type rules …

// Op conversion patterns
mlir::RewritePatternSet patterns(&context);
cir::populateCIRToLLVMConversionPatterns(patterns, typeConverter, dataLayout);
```

Key op lowering patterns:

| CIR op | LLVM dialect op |
|--------|----------------|
| `cir.alloca` | `llvm.alloca` |
| `cir.load` | `llvm.load` |
| `cir.store` | `llvm.store` |
| `cir.binop(add)` | `llvm.add` / `llvm.fadd` |
| `cir.binop(mul) nsw` | `llvm.mul` with `nsw` flag |
| `cir.call` | `llvm.call` |
| `cir.br` | `llvm.br` |
| `cir.brcond` | `llvm.cond_br` |
| `cir.return` | `llvm.return` |
| `cir.func` | `llvm.func` |
| `cir.global` | `llvm.mlir.global` |
| `cir.get_member` | `llvm.getelementptr` |
| `cir.ptr_stride` | `llvm.getelementptr` (base+index) |
| `cir.cast(integral)` | `llvm.sext` / `llvm.zext` / `llvm.trunc` |
| `cir.cast(int_to_bool)` | `llvm.icmp ne …, 0` |

After the conversion pass, the module contains only MLIR LLVM dialect ops and builtin types. The final step is `mlir::translateModuleToLLVMIR`, which produces an `llvm::Module` that enters the standard optimization and backend pipeline:

```cpp
// Entry point declared in LowerToLLVM.h
namespace cir::direct {
  std::unique_ptr<llvm::Module>
  lowerDirectlyFromCIRToLLVMIR(mlir::ModuleOp mlirModule,
                                llvm::LLVMContext &llvmCtx);
}
```

The resulting `llvm::Module` is semantically equivalent to what the classic CodeGenModule would have produced — enabling CIR to reuse the entire existing LLVM optimization and backend infrastructure unchanged.

## 54.3 Through-MLIR Lowering

The direct path translates CIR → LLVM dialect in one step. The through-MLIR path progressively lowers CIR through intermediate dialects, enabling the MLIR ecosystem to act on C/C++ code:

```
CIR (post-flatten)
  │
  ▼  CIR → affine/scf/memref
affine.for / scf.for / memref.alloc
  │
  ▼  affine optimizations (Polly-equivalent, tiling, fusion)
  │
  ▼  affine/scf → LLVM dialect
  │
  ▼  LLVM dialect → llvm::Module
```

As of LLVM 22.1, the through-MLIR path is experimental and covers a subset of CIR constructs. Its primary motivation is enabling polyhedral-style loop optimization (tiling, interchange, fusion) on C/C++ programs without Polly's `Scop` extraction phase — instead, the structured CIR loop ops are directly lowerable to `affine.for` when the bounds are affine expressions.

A secondary motivation is GPU offloading: structured CIR loops can be lowered to the `gpu.launch` op in the MLIR GPU dialect, providing a path from C++ loops annotated with `[[clang::gpu_offload]]` (a CIR extension in prototyping) to NVPTX or AMDGPU kernels.

## 54.4 The Lifetime Checker

One of the analyses enabled by CIR's scope-aware representation is a **C++ lifetime safety checker**. This pass runs on the pre-flatten CIR, where `cir.scope` regions are still intact:

**Problem**: dangling references are a pervasive C++ bug class:
```cpp
std::string_view sv;
{
  std::string s = "hello";
  sv = s;           // sv borrows from s
}                   // s destroyed here
use(sv);            // sv is dangling
```

In LLVM IR, both `s` and `sv` are just pointers; the scope boundary has been erased. Reconstructing it requires complex alias analysis.

In CIR, `s`'s `cir.alloca` is nested inside the inner `cir.scope`, and `sv`'s `cir.alloca` is in the outer scope. The lifetime checker can statically detect that the value assigned to `sv` has shorter lifetime than `sv` itself.

The checker implements a dataflow analysis over CIR regions:

1. For each `cir.alloca`, record its containing `cir.scope` as its **lifetime region**.
2. Track pointer provenance: when `cir.store` writes a pointer derived from one alloca into another alloca, record the provenance edge.
3. At the end of each `cir.scope`, check whether any surviving outer-scope pointer has provenance that terminates within this scope.
4. Report a diagnostic if so.

This is the CIR realization of the Lifetime Safety for C++ proposal (Herb Sutter, P1179, 2019) — previously only implemented as a Clang static analyzer plugin, now expressible as an MLIR pass on CIR.

## 54.5 Idiom Recognition

CIR's structured representation enables **idiom recognition passes** that produce more efficient LLVM IR than the direct lowering path:

### 54.5.1 Loop Idiom Recognition

Before CIRFlattenCFGPass converts loops to basic blocks, an idiom recognition pass can match structured patterns:

```
// CIR: memset idiom
cir.for : {
  cir.condition(%i_lt_n)
} body {
  cir.store %zero, %arr_elem_addr  // store zero into array element
  cir.yield continue
} step {
  cir.yield  // i++
}
```

When the body contains only a `cir.store` of a constant to a linearly advancing address, the pass replaces the entire `cir.for` with a `cir.memset` op, which lowers to `@llvm.memset`. This is the same recognition that LLVM's LoopIdiomRecognize pass performs, but at the source level where the loop bounds are exact and the aliasing is trivially known.

### 54.5.2 Range-Based For Simplification

C++ range-based `for` loops are desugared by Sema into calls to `begin()`/`end()` and `!=`/`++` operators. When the container is a known CIR record type with inline storage (e.g., `std::array`), the CIR simplify pass can:

1. Replace the `begin()`/`end()` calls with `cir.get_member` + `cir.ptr_stride` GEPs.
2. Replace the `!=` comparison with a pointer comparison.
3. Simplify the loop body to direct element access, eliminating the iterator indirection.

This optimization is possible in CIR because the record type's members are still visible; after lowering to LLVM IR, the record has been decomposed into raw offsets.

### 54.5.3 Copy Elision (NRVO)

Named Return Value Optimization is implemented in CIR by tracking `cir.alloca` ops that are eligible for NRVO: a local variable of the function's return type that is always returned. When NRVO applies, the alloca is replaced with the hidden `sret` parameter address, eliminating the copy:

```
// Before NRVO
%result = cir.alloca !cir.record<struct "Foo" {…}>, ["result"]
// … build result …
%ret_copy = cir.copy %result : !cir.ptr<…>
cir.return %ret_copy

// After NRVO
// result directly written to sret parameter, no copy
```

## 54.6 ABI Lowering Details

`CXXABILoweringPass` handles the full Itanium C++ ABI surface that is visible at the CIR level:

### 54.6.1 vtable Emission

```
// CIR vtable for class Base
cir.global @_ZTV4Base : !cir.ptr<!cir.func<() -> !cir.void>> = {
  #cir.null_ptr,   // RTTI offset
  @_ZTI4Base,      // RTTI pointer
  @_ZN4Base3fooEv  // virtual function 0
}
```

The vtable layout follows the Itanium ABI: two padding slots (offset-to-top, RTTI), then virtual function pointers in declaration order.

### 54.6.2 `dynamic_cast` Lowering

```
// Source: B* b = dynamic_cast<B*>(a);
// CIR after CXXABILoweringPass:
%vptr = cir.load (cir.get_member %a_ptr[0] → !cir.ptr<!cir.ptr<…>>) : …
%rtti_base = cir.ptr_stride %vptr, -1
%src_type = @_ZTI1A
%dst_type = @_ZTI1B
%result = cir.call @__cxa_dynamic_cast(%a_ptr, %src_type,
                                        %dst_type, %offset) : …
```

### 54.6.3 Exception Lowering

`cir.try`/`cir.catch` → `invoke`/`landingpad`:

```
// After CIRFlattenCFGPass + direct lowering:
%r = llvm.invoke @may_throw() to ^normal unwind ^lpad : …
^lpad:
  %lp = llvm.landingpad catch @_ZTIi : …
  llvm.call @__cxa_begin_catch(%lp) : …
  // catch body
  llvm.call @__cxa_end_catch() : …
  llvm.br ^after
```

## 54.7 Integrating CIR into a Build

Since CIR requires `-DCLANG_ENABLE_CIR=ON` at CMake time and is not enabled in distribution packages as of LLVM 22.1, using CIR in production requires building LLVM from source. A minimal CMake invocation:

```bash
cmake -G Ninja ../llvm-project/llvm \
  -DLLVM_ENABLE_PROJECTS="clang" \
  -DCLANG_ENABLE_CIR=ON \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_TARGETS_TO_BUILD=X86

ninja clang
```

With a CIR-enabled build, the usage is:

```bash
# Emit CIR text
./bin/clang -fclangir -emit-cir -o foo.cir foo.cpp

# Compile through CIR
./bin/clang -fclangir -O2 -o foo foo.cpp

# Disable CIR-level simplifications (debugging)
./bin/clang -fclangir -mllvm -clangir-disable-passes -S foo.cpp
```

The `-clangir-disable-verifier` flag suppresses MLIR module verification, useful when working on a CIR feature that produces temporarily ill-formed IR.

## 54.8 Future Directions

Several directions are being pursued post-LLVM 22.1:

**Lifetime safety as a default analysis**: the scope-aware lifetime checker described in §54.4 is being developed toward production quality, with the goal of making it a `-Wlifetime` diagnostic in standard builds.

**Through-MLIR loop optimizations**: the affine lowering path (§54.3) is being extended to cover all CIR loop forms, enabling polyhedral-style tiling and vectorization as MLIR passes on C/C++ code.

**Incremental CIR serialization**: a CIR-level module format for precompiled headers, replacing the AST-level `.pch` format with a partially-lowered CIR form that can be linked and further processed.

**CIR for GPU offloading**: structured CIR loops lowering directly to `gpu.launch` without the intermediate CPU-side pass, enabling a unified compilation path for CPU and GPU code from standard C++.

**Lifetime and borrow checking**: a prototype CIR pass implementing the C++ Contracts memory-safety profile, consuming CIR with its rich provenance information to emit diagnostics equivalent to those produced by Rust's borrow checker for a subset of C++ programs.

---

## Chapter Summary

- Pre-lowering CIR-to-CIR passes run in order: canonicalize → simplify → CXXABILowering → hoist-allocas → lowering-prepare → goto-solver → flatten-cfg.
- `CXXABILoweringPass` handles vtable dispatch, RTTI, EH personality, and constructor/destructor variants, consulting `clang::ASTContext` for ABI details.
- `CIRFlattenCFGPass` is the final pre-lowering step, converting structured ops to basic-block form.
- Direct lowering (`cir::direct`) uses a single `ConversionTarget` pass to translate CIR ops to the MLIR LLVM dialect, producing an `llvm::Module` via `translateModuleToLLVMIR`.
- Through-MLIR lowering progressively lowers through `affine`/`scf`/`memref` dialects, enabling the MLIR optimization ecosystem on C/C++ code (experimental as of LLVM 22.1).
- The lifetime checker exploits `cir.scope` nesting to detect dangling pointer/reference uses at the source level — an analysis impossible in flat LLVM IR.
- Idiom recognition (memset loops, NRVO, range-for simplification) benefits from CIR's structured representation before CFG flattening.
- Using CIR in a build requires `-DCLANG_ENABLE_CIR=ON` at LLVM CMake time; the driver flag is `-fclangir`.
