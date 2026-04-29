# Chapter 125 — Flang Architecture and Driver

*Part XVIII — Flang*

Fortran may be half a century old, but it remains the dominant language for high-performance scientific computing: weather models, fluid dynamics solvers, nuclear physics codes, and the dense linear-algebra libraries that underpin modern machine learning all rely on it. For years the LLVM ecosystem lacked a production-quality Fortran compiler, leaving projects to depend on GFortran or proprietary alternatives. Flang, LLVM's native Fortran compiler, changes that. Built from the ground up as a modern, modular compiler within the LLVM monorepo, Flang covers Fortran 77 through 2018, leverages MLIR as its mid-level IR, and produces code via the same LLVM backend that powers Clang. This chapter traces Flang from its origins through its full architecture: the driver, parser, semantic analysis engine, and the lowering bridge that hands control to MLIR.

---

## 125.1 Flang Overview

Flang occupies the `flang/` subtree of the LLVM monorepo. Its immediate predecessor, known informally as **f18**, was a clean-room reimplementation begun at NVIDIA in 2017 and donated to the LLVM Foundation in 2019. The original LLVM Flang (sometimes called "classic Flang," hosted under the `flang-legacy` organization) was a separate project based on PGI's Fortran front end; it is no longer actively developed. Every reference in this book to "Flang" means the in-tree implementation at `flang/`.

### Language Coverage

Flang targets:

| Standard | Status |
|----------|--------|
| Fortran 77 | Complete |
| Fortran 90 | Complete |
| Fortran 95 | Complete |
| Fortran 2003 | Mostly complete; polymorphism stable |
| Fortran 2008 | Mostly complete; coarrays basic support |
| Fortran 2018 | Partially complete; teams, events in progress |
| OpenMP 5.1 | CPU complete; GPU offload production-ready |
| OpenACC 3.x | Supported via `acc.*` dialect |

### Major Components

```
flang/
├── include/flang/
│   ├── Parser/           -- ParseTree node types, Provenance
│   ├── Semantics/        -- Symbol, Scope, Evaluate::Expr<T>
│   ├── Lower/            -- AbstractConverter, Bridge
│   └── Optimizer/        -- FIR/HLFIR dialect headers, passes
├── lib/
│   ├── Parser/           -- Prescanner, Tokenizer, recursive-descent parser
│   ├── Semantics/        -- ResolveNames, CheckDeclarations, evaluate/
│   ├── Lower/            -- Bridge.cpp, OpenMP.cpp, OpenACC.cpp, ...
│   └── Optimizer/        -- FIR passes, HLFIR passes, CodeGen/
├── runtime/              -- flang-rt: I/O, intrinsics, type info
└── tools/
    ├── flang-driver/     -- flang-new binary
    └── bbc/              -- bbc (Flang's mlir-opt stand-in)
```

The compilation pipeline is:

```
Fortran source
    │
    ▼  Prescanner + Tokenizer (fixed/free-form normalization)
    │
    ▼  Recursive-descent Parser → ParseTree
    │
    ▼  Semantic Analysis (ResolveNames, type-checking, expression evaluation)
    │
    ▼  Lower::Bridge → HLFIR (mlir::ModuleOp)
    │
    ▼  HLFIR simplification + bufferization passes
    │
    ▼  HLFIR → FIR lowering passes
    │
    ▼  FIR optimization passes (inlining, array copy elision, etc.)
    │
    ▼  FIR → LLVM dialect conversion
    │
    ▼  ConvertMLIRToLLVMIR → LLVM IR Module
    │
    ▼  LLVM backend → object file / assembly
```

---

## 125.2 The Driver

### flang-new and the Clang Driver Framework

The `flang-new` binary (defined in `flang/tools/flang-driver/driver.cpp`) is built on the **Clang Driver framework** rather than rolling its own. This reuse gives Flang toolchain integration (target triples, sysroots, multilibs, response files) essentially for free. The driver delegates to Clang for the final link step, invoking `clang` or the platform linker with the appropriate runtime and intrinsic libraries.

The Flang-specific tool chain classes live in:
- [`clang/lib/Driver/ToolChains/Flang.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Driver/ToolChains/Flang.cpp)
- [`clang/lib/Driver/ToolChains/Flang.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Driver/ToolChains/Flang.h)

The driver constructs a job list from source file suffixes (`.f`, `.f90`, `.F90`, `.f95`, `.f03`, `.f08`, `.f18`, `.fpp`). Each Fortran source goes through the `Flang` tool (the `-fc1` front end), then through `clang` for assembly and linking.

### Compilation Steps

Invoking `flang-new -c hello.f90` creates a job chain:

1. **Preprocess** (optional): `-cpp` / `-nocpp` controls the C preprocessor. Flang has its own prescanner that handles `#include`, `#define`, and `!DIR$` directives without invoking the C preprocessor by default.
2. **Compile** (`-fc1`): drives the full front end + MLIR pipeline + LLVM backend to produce an object file or intermediate IR.
3. **Assemble**: not typically separate for Fortran; `-fc1` emits object directly.
4. **Link**: delegated to `clang`, which links `flang-rt`, `libm`, and the system CRT.

Key driver flags:

| Flag | Effect |
|------|--------|
| `-fc1` | Invoke the compiler front end directly |
| `-emit-fir` | Stop after FIR generation; emit `.fir` text |
| `-emit-hlfir` | Stop after HLFIR generation |
| `-emit-llvm` | Emit LLVM IR text (`.ll`) |
| `-emit-obj` | Emit object file (default with `-c`) |
| `-fsyntax-only` | Parse and check semantics; no code generation |
| `-fopenacc` | Enable OpenACC processing |
| `-fopenmp` | Enable OpenMP processing |
| `-fopenmp-targets=nvptx64-nvidia-cuda` | Enable GPU offload |
| `-mmlir <pass-option>` | Pass an option to the MLIR pipeline |
| `-O0` / `-O2` / `-O3` | Optimization level |

### FrontendAction Hierarchy

The front end uses a `FrontendAction` hierarchy to dispatch to different pipeline endpoints:

```cpp
// flang/include/flang/Frontend/FrontendActions.h
class FrontendAction { virtual void executeAction() = 0; };

class InitOnlyAction       : public FrontendAction { ... };
class PrescanAction        : public FrontendAction { ... };
class ParseSyntaxOnlyAction: public FrontendAction { ... };
class EmitHLFIRAction      : public CodeGenAction   { ... };
class EmitFIRAction        : public CodeGenAction   { ... };
class EmitLLVMAction       : public CodeGenAction   { ... };
class EmitObjAction        : public CodeGenAction   { ... };
```

[`flang/include/flang/Frontend/FrontendActions.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/include/flang/Frontend/FrontendActions.h)

`CodeGenAction::executeAction()` is the common entry point; it invokes the parser, semantics, and then the appropriate lowering endpoint depending on the action type.

---

## 125.3 The Parser

### Design Philosophy

Flang's parser is a **hand-written recursive-descent parser** — no parser generator is used. This choice reflects the complexity of Fortran's grammar: fixed-form Fortran has notoriously ambiguous tokenization (spaces are insignificant inside tokens in some contexts), and the language is not context-free in several ways. A hand-written parser can carry semantic context during parsing where needed.

The parser lives in [`flang/lib/Parser/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Parser/) and produces a **ParseTree** — a typed tree of sum types and sequence types representing the entire program.

### Prescanner and Tokenizer

Before the recursive-descent parser runs, two pre-stages normalize the source:

**Prescanner** ([`flang/lib/Parser/prescan.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Parser/prescan.cpp)):
- Handles both **free-form** (Fortran 90+) and **fixed-form** (Fortran 77) source layouts
- Strips comments, handles line continuation, processes `INCLUDE` lines
- Expands preprocessor macros (when `-cpp` is active or implicit)
- Produces a flat `CookedSource` — a contiguous normalized character stream with a provenance map back to the original source locations

**Tokenizer**: operates on the cooked source; feeds character sequences to the recursive-descent parser functions.

### ParseTree Nodes

The ParseTree is defined in [`flang/include/flang/Parser/parse-tree.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/include/flang/Parser/parse-tree.h). It uses a distinctive C++ style: every grammar production is a `struct` with `std::tuple` or `std::variant` members. For example:

```cpp
// Excerpt: Fortran::parser parse-tree types
struct Program {
  std::list<ProgramUnit> v;
};

struct SubroutineSubprogram {
  std::tuple<Statement<SubroutineStmt>,
             ImplicitPart,
             std::list<DeclarationConstruct>,
             ExecutionPart,
             std::optional<ContainsStmt>,
             std::list<InternalSubprogram>,
             Statement<EndSubroutineStmt>> t;
};

struct FunctionSubprogram {
  std::tuple<Statement<FunctionStmt>,
             ImplicitPart,
             std::list<DeclarationConstruct>,
             ExecutionPart,
             std::optional<ContainsStmt>,
             std::list<InternalSubprogram>,
             Statement<EndFunctionStmt>> t;
};
```

The struct-per-production design gives strong typing across the entire syntax tree with no unsafe casts.

### The Parsing Class

[`flang/lib/Parser/parsing.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Parser/parsing.cpp) defines the `Parsing` class that orchestrates scanning and parsing:

```cpp
class Parsing {
public:
  void Prescan(const std::string &path, Options);
  void Parse(llvm::raw_ostream *);
  bool ParsedOk() const;
  const Program &program() const;
  ...
};
```

### Unparse Round-Trip

A notable feature is the `Unparse()` function ([`flang/lib/Parser/unparse.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Parser/unparse.cpp)), which regenerates valid Fortran source from the ParseTree. This serves as a correctness check (round-trip test) and is used in regression testing:

```bash
flang-new -fc1 -fdebug-unparse-no-sema hello.f90
```

---

## 125.4 Semantic Analysis

### Overview

Semantic analysis runs in `flang/lib/Semantics/`. Its job is to:
1. Build the **symbol table** (resolve every name to a `Symbol`)
2. Perform **type assignment** and **implicit typing** rules
3. Check **semantic constraints** from the Fortran standard
4. Evaluate **constant expressions** at compile time
5. Produce a decorated ParseTree ready for lowering

### Symbols and Scopes

The symbol table is organized into nested `Scope` objects ([`flang/include/flang/Semantics/scope.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/include/flang/Semantics/scope.h)):

```
GlobalScope
├── ModuleScope (for each MODULE)
│   └── SubprogramScope (for contained procedures)
├── SubprogramScope (for main program, external subprograms)
│   └── SubprogramScope (for internal subprograms)
└── BlockScope (for BLOCK constructs)
```

A `Symbol` ([`flang/include/flang/Semantics/symbol.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/include/flang/Semantics/symbol.h)) holds:
- Name (canonicalized to uppercase for case-insensitivity)
- Owning `Scope *`
- A `SymbolDetails` variant: `ObjectEntityDetails`, `ProcEntityDetails`, `ModuleDetails`, `DerivedTypeDetails`, `SubprogramDetails`, etc.
- Attributes: `INTENT(IN/OUT/INOUT)`, `ALLOCATABLE`, `POINTER`, `TARGET`, `SAVE`, etc.

```cpp
// Simplified Symbol usage:
const Symbol *sym = scope.FindSymbol(name);
if (const auto *obj = sym->detailsIf<ObjectEntityDetails>()) {
  const DeclTypeSpec *type = obj->type();  // INTEGER, REAL, derived type, etc.
  if (obj->IsArray()) {
    const ArraySpec &shape = obj->shape();
  }
}
```

### ResolveNames Pass

The `ResolveNames` visitor ([`flang/lib/Semantics/resolve-names.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Semantics/resolve-names.cpp)) is the largest single file in Flang (~8,000 lines). It walks the ParseTree, creates scopes, and populates each scope's symbol map. Crucially, it handles:

- Fortran's **implicit typing** rules (names starting with I–N are INTEGER by default unless overridden)
- `USE` statements: imports symbols from modules
- `INTERFACE` blocks: establishes explicit interfaces for external procedures
- Overloaded operators and defined assignments

### Expression Evaluation

After name resolution, expressions are **evaluated** into typed `Evaluate::Expr<T>` trees ([`flang/include/flang/Evaluate/expression.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/include/flang/Evaluate/expression.h)). The evaluate library is a separate compile-time expression evaluator capable of folding constant expressions:

```cpp
// Evaluate::Expr<T> is a typed expression tree:
// Expr<SomeType> = variant<Designator<SomeType>, FunctionRef<SomeType>,
//                          Add<SomeType>, Multiply<SomeType>, ...>
using IntExpr = Expr<SomeKindInteger<4>>;
```

Constant folding here handles `PARAMETER` variables, array bounds, and `DATA` initializers at semantic analysis time — a requirement of the Fortran standard.

### CheckDeclarations and CheckExpressions

Separate checker passes validate constraints:
- [`flang/lib/Semantics/check-declarations.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Semantics/check-declarations.cpp): checks attribute compatibility, module coherence, derived type rules
- [`flang/lib/Semantics/check-expressions.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Semantics/check-expressions.cpp): validates expression legality in different contexts (e.g., constant expressions in PARAMETER statements)
- [`flang/lib/Semantics/check-do-forall.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Semantics/check-do-forall.cpp): DO loop constraint checking (index variable type, modification restrictions)
- [`flang/lib/Semantics/check-omp-structure.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Semantics/check-omp-structure.cpp): OpenMP semantic constraints

---

## 125.5 Lower-Level Internals: The Lowering Bridge

### AbstractConverter

The interface between semantics and MLIR code generation is the `AbstractConverter` class ([`flang/include/flang/Lower/AbstractConverter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/include/flang/Lower/AbstractConverter.h)):

```cpp
class AbstractConverter {
public:
  // Translate a Fortran expression to MLIR value:
  virtual mlir::Value genExprValue(const Evaluate::Expr<SomeType> &,
                                   StatementContext &) = 0;
  // Translate an address:
  virtual mlir::Value genExprAddr(const Evaluate::Expr<SomeType> &,
                                  StatementContext &) = 0;
  // Generate box for descriptor-based objects:
  virtual mlir::Value genExprBox(mlir::Location, const Evaluate::Expr<SomeType> &,
                                 StatementContext &) = 0;
  virtual mlir::OpBuilder &getFirOpBuilder() = 0;
  virtual mlir::Location getCurrentLocation() = 0;
  ...
};
```

### Bridge Class

`Bridge` ([`flang/lib/Lower/Bridge.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Lower/Bridge.cpp)) is the top-level lowering driver. It:

1. Creates an `mlir::ModuleOp`
2. Iterates over `Program` top-level units
3. For each subprogram, calls `FirConverter::run()` which walks the semantic tree
4. `FirConverter` inherits `AbstractConverter` and uses an `mlir::OpBuilder` to emit HLFIR ops

```cpp
// Simplified Bridge usage:
Fortran::lower::LoweringBridge bridge =
    Fortran::lower::LoweringBridge::create(
        mlirCtx, semantics, defaultKinds, /*...*/);
bridge.lower(parseTree, semantics);
mlir::ModuleOp module = bridge.getModule();
```

### OpenMP and OpenACC Lowering

Directive lowering is handled by dedicated files:
- [`flang/lib/Lower/OpenMP/OpenMP.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Lower/OpenMP/OpenMP.cpp): translates `!$omp` constructs to `omp.*` MLIR dialect ops
- [`flang/lib/Lower/OpenACC.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Lower/OpenACC.cpp): translates `!$acc` constructs to `acc.*` ops

These are covered in depth in [Chapter 127 — Flang OpenMP and OpenACC](ch127-flang-openmp-and-openacc.md).

---

## 125.6 flang-new vs f18

### Historical Context

The original LLVM Fortran effort predates 2017. PGI (now part of NVIDIA) maintained "classic Flang" based on their commercial compiler; it used a different IR strategy and is no longer the upstream. The **f18** name referred to the clean-room rewrite begun at NVIDIA, targeting Fortran 2018. When the project was accepted into the LLVM monorepo (landing in 2021 as `flang/`), the binary was named `flang-new` to avoid confusion with the classic Flang installations on many systems. The plan, now substantially realized, is to eventually rename `flang-new` to `flang` as it achieves feature parity.

### Production Readiness

As of LLVM 22, Flang:
- Passes the **SPEC CPU 2017 Fortran benchmarks** (503.bwaves\_r, 507.cactuBSSN\_r, 521.wrf\_r, 527.cam4\_r, 628.pop2\_s, 654.roms\_s) with correct results
- Compiles the **ECMWF IFS model** (a 500k-line operational weather forecast code) successfully
- Is the default Fortran compiler in several HPC system distributions (Frontier, Aurora)
- Has official support from AMD (ROCm) and NVIDIA for GPU offload via OpenMP

### Known Limitations

- **IEEE exception handling**: `IEEE_SET_HALTING_MODE` and related Fortran 2003 IEEE intrinsics are partially implemented (see [Chapter 128](ch128-flang-codegen-and-runtime.md))
- **Coarrays** (Fortran 2008 distributed memory): basic support; full co-array teams (Fortran 2018) in progress
- **Fortran 2023 extensions**: ranked arrays as procedure arguments in some contexts still under development
- **COMPLEX arithmetic intrinsics**: some edge cases in edge KINDs (COMPLEX(16)) require verification

---

## 125.7 The bbc Tool

`bbc` (the "Flang MLIR compiler") is to Flang what `mlir-opt` is to MLIR: a testing and development tool that runs Fortran source through the front end and emits MLIR at various pipeline stages.

```bash
# Emit HLFIR from a Fortran file:
bbc --emit-hlfir -o output.hlfir hello.f90

# Emit FIR:
bbc --emit-fir -o output.fir hello.f90

# Run FIR passes:
bbc --emit-fir hello.f90 | \
  /usr/lib/llvm-22/bin/mlir-opt \
    --convert-hlfir-to-fir \
    --fir-to-llvm-ir \
    -o output.ll
```

Source: [`flang/tools/bbc/bbc.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/tools/bbc/bbc.cpp)

The `bbc` tool accepts most `-fc1` flags and is the primary vehicle for FIR/HLFIR FileCheck tests in `flang/test/`.

---

## 125.8 Testing Infrastructure

Flang's test suite is in `flang/test/` and uses LLVM's **lit** (LLVM Integrated Tester) framework:

```
flang/test/
├── Driver/          -- driver flag tests
├── Parser/          -- parse-only tests (--fdebug-unparse)
├── Semantics/       -- semantic error tests (CHECK-ERROR)
├── Lower/           -- bbc-based FIR/HLFIR output tests
├── HLFIR/           -- HLFIR pass tests
├── Fir/             -- FIR pass tests
├── Transforms/      -- optimization pass tests
└── Codegen/         -- FIR-to-LLVM-IR tests
```

Tests use the standard FileCheck pattern matching:

```fortran
! RUN: %flang_fc1 -emit-hlfir %s -o - | FileCheck %s
subroutine add(a, b, c)
  real :: a(10), b(10), c(10)
  c = a + b
end subroutine
! CHECK: hlfir.elemental
! CHECK: arith.addf
```

Semantic error tests use `%flang_fc1 -fsyntax-only` with `! CHECK-ERROR:` markers.

---

## Chapter Summary

- Flang is LLVM's in-tree Fortran compiler, descended from the f18 clean-room rewrite; the `flang/` subtree covers the full compilation pipeline
- The `flang-new` driver builds on the Clang Driver framework, delegating preprocessing, compilation (`-fc1`), and linking to appropriate tools
- The front end consists of a Prescanner (handles fixed- and free-form source, INCLUDE, macros), a hand-written recursive-descent Parser producing a typed `ParseTree`, and a semantic analysis engine that builds scoped symbol tables and constant-folding expression trees
- `ResolveNames`, `CheckDeclarations`, `CheckExpressions`, and directive-specific checkers form the semantic layer; `Evaluate::Expr<T>` is the compile-time expression representation
- The `Lower::Bridge` class drives the translation from semantic representation to MLIR HLFIR dialect ops inside an `mlir::ModuleOp`; `AbstractConverter` is the interface contract
- `bbc` is the testing/development tool equivalent to `mlir-opt`, used in the lit test suite via `%flang_fc1` substitution
- Flang is production-ready for SPEC CPU 2017 and large scientific codes; remaining gaps are in IEEE exception handling, coarray teams, and some Fortran 2023 features


---

@copyright jreuben11
