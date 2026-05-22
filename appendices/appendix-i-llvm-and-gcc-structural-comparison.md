# Appendix I — LLVM and GCC: A Structural Comparison

Chapter 1 introduced the LLVM project with a brief comparison to GCC. This appendix is the full treatment: a section-by-section examination of how the two compiler infrastructures differ in architecture, intermediate representation, pass management, backend code generation, frontend design, toolchain ecosystem, optimization capability, language support, and ABI interoperability. The goal is not advocacy but orientation — a compiler engineer who knows one system well should be able to read this appendix and understand precisely where the other system diverges, and why.

---

## Table of Contents

- [I.1 Design Philosophy and Architecture](#i1-design-philosophy-and-architecture)
  - [The Library Model versus the Monolithic Model](#the-library-model-versus-the-monolithic-model)
  - [API Stability](#api-stability)
  - [Licensing: Apache 2.0 with LLVM Exception versus GPLv3](#licensing-apache-20-with-llvm-exception-versus-gplv3)
  - [Monorepo versus Multi-Repo](#monorepo-versus-multi-repo)
- [I.2 Intermediate Representation Comparison](#i2-intermediate-representation-comparison)
  - [GCC's Four-Level IR Stack](#gccs-four-level-ir-stack)
  - [LLVM's Single Typed SSA IR](#llvms-single-typed-ssa-ir)
  - [GIMPLE versus LLVM IR: Key Differences](#gimple-versus-llvm-ir-key-differences)
  - [RTL versus MachineIR](#rtl-versus-machineir)
- [I.3 Pass Infrastructure](#i3-pass-infrastructure)
  - [GCC's Pass Registry](#gccs-pass-registry)
  - [LLVM's PassManager](#llvms-passmanager)
  - [Analysis Caching and Invalidation](#analysis-caching-and-invalidation)
- [I.4 Backend and Target Description](#i4-backend-and-target-description)
  - [Machine Description: `.md` Files versus TableGen](#machine-description-md-files-versus-tablegen)
  - [Register Allocation](#register-allocation)
  - [Instruction Selection](#instruction-selection)
- [I.5 Frontend and Diagnostic Quality](#i5-frontend-and-diagnostic-quality)
  - [AST Architecture](#ast-architecture)
  - [C++ Template Instantiation](#c-template-instantiation)
  - [Diagnostic Quality](#diagnostic-quality)
  - [Language Server Protocol](#language-server-protocol)
  - [GNU Extension Compatibility](#gnu-extension-compatibility)
- [I.6 Toolchain Ecosystem](#i6-toolchain-ecosystem)
  - [Linker Comparison: GNU ld versus LLD](#linker-comparison-gnu-ld-versus-lld)
  - [Sanitizer Comparison](#sanitizer-comparison)
  - [PGO and BOLT](#pgo-and-bolt)
- [I.7 Optimization Capability Comparison](#i7-optimization-capability-comparison)
  - [Shared Optimizations](#shared-optimizations)
  - [GCC-Specific Strengths](#gcc-specific-strengths)
  - [LLVM-Specific Strengths](#llvm-specific-strengths)
- [I.8 Language and Target Support Matrix](#i8-language-and-target-support-matrix)
  - [Language Support](#language-support)
  - [Target Architecture Support](#target-architecture-support)
- [I.9 ABI Interoperability](#i9-abi-interoperability)
  - [Linking GCC and Clang Objects](#linking-gcc-and-clang-objects)
  - [Known ABI Divergences](#known-abi-divergences)
  - [Mixed Toolchain in Practice: ClangBuiltLinux](#mixed-toolchain-in-practice-clangbuiltlinux)
  - [`rustc_codegen_gcc` and `libgccjit`](#rustccodegengcc-and-libgccjit)
- [Summary](#summary)

---

## I.1 Design Philosophy and Architecture

### The Library Model versus the Monolithic Model

The most fundamental difference between LLVM and GCC is architectural: LLVM is a collection of independent libraries; GCC is an integrated application.

LLVM exposes no binary called `llvm`. The tools you invoke — `clang`, `opt`, `llc`, `lld`, `llvm-ar` — are thin driver programs that link against LLVM's libraries. A downstream project can pick up exactly the libraries it needs: `LLVMCore` for the IR data model, `LLVMPasses` for the optimization pipeline, `LLVMCodeGen` for the backend, or `LLVMMC` for the assembler layer. Each library has a defined C++ API, a C API wrapper in `llvm-c/`, and independently versioned headers.

GCC presents a different abstraction. The `gcc` binary is a driver that invokes `cc1` (the C compiler), `cc1plus` (the C++ compiler), `f951` (Fortran), and so on — but these are executables, not libraries you link against. The internal representations — `GENERIC`, `GIMPLE`, `RTL` — are defined in C headers scattered across the GCC source tree and are explicitly not a public API. The GCC internals manual states this directly: "No part of GCC is guaranteed to be stable from one release to the next."

The practical consequence: you can write a program that uses LLVM to compile a function at runtime and execute it. You cannot do that with GCC in the same way; `libgccjit` provides a narrow JIT interface, but it re-exposes a small surface area rather than giving you access to the full compiler internals.

### API Stability

LLVM IR is a stable, versioned format. Programs that emit or consume LLVM IR (`.ll` files or `.bc` bitcode) have a reasonable expectation of forward compatibility, and the LLVM release notes document breaking changes. The `LLVMContext`, `Module`, `Function`, `BasicBlock`, and `Instruction` C++ types have been stable enough that downstream projects track LLVM releases with relatively small porting effort.

LLVM does not provide source-level ABI stability across releases — the C++ API changes every release — but the semantic model of LLVM IR is stable. A pass written against LLVM 18 needs mechanical updates to compile against LLVM 22, but its conceptual model transfers directly.

GCC's plugin API (`gcc-plugin.h`) exposes callback hooks into the compilation process but not the full IR. Plugins can observe and modify GIMPLE through a limited interface, but they cannot inspect RTL in the same way. The plugin API has evolved across releases in breaking ways. GCC's interprocedural analysis infrastructure, its loop representation, and its alias analysis interfaces are internal to the compiler and unavailable to plugin authors in their full form.

### Licensing: Apache 2.0 with LLVM Exception versus GPLv3

LLVM is distributed under the Apache License 2.0 with the LLVM exception. The exception matters: it removes the Apache 2.0 requirement that modified source be marked as changed when distributing in binary form, and it explicitly allows LLVM-compiled binaries to be linked against code under licenses that are incompatible with Apache 2.0 (including MIT, BSD, and proprietary licenses). The net effect is that embedding LLVM in a proprietary product — compiling your DSL with it, shipping your custom backend — imposes no copyleft obligations on your product's source code.

GCC is distributed under GPLv3 with the GCC Runtime Library Exception. The runtime exception covers the output of GCC: programs compiled by GCC are not considered derivative works of GCC and can be distributed under any license. But the compiler itself — the tools, the libraries, any modifications you make — must be distributed under GPLv3 if distributed at all. This is why Apple funded Clang: shipping Xcode requires shipping a C++ compiler, and distributing a modified GCC in Xcode would have obligated Apple to release the source of any modifications. Apache 2.0 imposed no such requirement.

For organizations building commercial compiler products — hardware vendor toolchains, embedded systems IDEs, language-specific compilers — LLVM's license is the decisive technical factor. The GPL is not a defect in GCC; it is a deliberate design choice that maximizes GCC's role as shared community infrastructure. But it explains the pattern of commercial compiler infrastructure migrating to LLVM.

### Monorepo versus Multi-Repo

Since 2019, LLVM uses a single monorepo (`llvm/llvm-project` on GitHub) containing all subprojects: llvm, clang, lld, lldb, mlir, flang, compiler-rt, libc++, libc, openmp, and more. This means a single `git clone` gives you the full ecosystem, and cross-component changes land in a single commit. CI covers the full matrix.

GCC is distributed as a single tarball that includes the compiler itself but not binutils (the GNU assembler and linker), glibc, or GDB. These are separate projects with separate release cycles. Building a full GCC toolchain requires coordinating versions across GCC, binutils, and the target sysroot — a task that cross-compilation toolchain generators like `crosstool-NG` and `buildroot` automate but cannot eliminate.

---

## I.2 Intermediate Representation Comparison

### GCC's Four-Level IR Stack

GCC compiles through four distinct intermediate representations, each serving a different phase of compilation:

```
Source (C / C++ / Fortran / Ada / ...)
    ↓  front end
GENERIC      — language-independent AST; tree nodes; no control flow normalization
    ↓  gimplification
GIMPLE       — three-address statements; calls normalized; no side effects in expressions
    ↓  SSA construction
SSA-GIMPLE   — φ-functions at join points; use-def chains; primary optimization form
    ↓  RTL expansion (expand_expr)
RTL          — Register Transfer Language; near-assembly; register allocation target
    ↓  reload / LRA
Machine code
```

**GENERIC** is the output of GCC's language front ends. It is a tree representation (`tree` nodes, defined in `gcc/tree.h`) that is language-independent but retains high-level structure: function definitions, loops, conditionals, and expressions are all represented as trees. GENERIC is the point where different language front ends converge. C, C++, Fortran, and Ada all produce GENERIC, though the specific tree-node variants differ by language.

**GIMPLE** is produced by *gimplification* — the pass that normalizes GENERIC into a simpler form. GIMPLE enforces that:
- Every expression is at most a single operation (three-address form).
- Every call is a top-level statement, not nested in an expression.
- There are no side effects in subexpressions.

GIMPLE uses the same `tree` node types as GENERIC but restricts which combinations are permitted. It is not a new data structure; it is a restricted subset of the `tree` grammar. This is a source of both strength (no translation cost) and confusion (GENERIC and GIMPLE share types, requiring discipline to distinguish them programmatically).

**SSA-GIMPLE** is produced by building Static Single Assignment form over GIMPLE. GCC's SSA construction follows the standard dominance-frontier algorithm (see [Chapter 9 — Intermediate Representations and SSA Construction](../part-02-compiler-theory/ch09-intermediate-representations-and-ssa-construction.md)). φ-functions are represented as `GIMPLE_PHI` statements. This is the form used by essentially all of GCC's middle-end optimizers: SCCP, GVN, PRE, DCE, alias analysis, loop transformations.

**RTL** (Register Transfer Language) is the output of the *RTL expander*, which translates SSA-GIMPLE function by function into a near-assembly representation. RTL uses pseudo-registers (unlimited virtual registers, later allocated to physical registers by the register allocator). RTL expressions encode machine operations at a level close to the target ISA: `(set (reg:SI 0) (plus:SI (reg:SI 1) (const_int 4)))`. RTL is defined in `gcc/rtl.h` and processed by a sequence of passes that perform instruction selection, register allocation (IRA or LRA), and instruction scheduling before final assembly emission.

### LLVM's Single Typed SSA IR

LLVM uses a single intermediate representation from the output of the front end through all middle-end optimization and into the early stages of the backend:

```
Source
    ↓  front end (Clang / Flang / ...)
LLVM IR      — typed SSA; infinite virtual registers; explicit memory model
    ↓  optimization passes (PassManager)
LLVM IR      — same representation, progressively transformed
    ↓  SelectionDAG / GlobalISel
MachineIR    — target-specific; virtual registers; pre-regalloc
    ↓  register allocation (Greedy / PBQP)
MachineIR    — physical registers; post-regalloc
    ↓  MC layer (MCInst emission)
Machine code
```

The key property: there is no representation change between "high-level middle end" and "low-level middle end." The function that enters the inliner and the function that enters the vectorizer are the same type of object. LLVM's `Value`/`User`/`Use` triad (described in [Chapter 16 — IR Structure](../part-04-llvm-ir/ch16-ir-structure.md)) is uniform across all optimization passes.

### GIMPLE versus LLVM IR: Key Differences

**Type system.** GIMPLE operates over GCC's `tree` type system, which is rooted in the C type model: `INTEGER_TYPE`, `REAL_TYPE`, `POINTER_TYPE`, `ARRAY_TYPE`, `RECORD_TYPE`, `UNION_TYPE`. Types are represented as `tree` nodes and canonicalized by GCC's type comparison logic, which handles C-level structural equivalence.

LLVM IR has a separate, explicit type system (`llvm::Type` hierarchy): `IntegerType` (parameterized by bit width), `FloatType`, `DoubleType`, `PointerType` (opaque since LLVM 15), `ArrayType`, `StructType`, `FunctionType`, `VectorType`. LLVM types are interned by `LLVMContext` and compared by pointer identity. Notably, LLVM IR does not have a notion of signed versus unsigned integer at the type level — signedness is an attribute of operations (`add nsw`, `sdiv` vs `udiv`, `icmp slt` vs `icmp ult`).

**Memory model.** GIMPLE uses `MEM_REF` tree nodes to represent memory accesses, with TBAA (Type-Based Alias Analysis) information attached as tree attributes. The C/C++ type system drives aliasing: an `int*` and a `float*` are assumed non-aliasing by default.

LLVM IR uses `load` and `store` instructions with explicit pointer operands and optional metadata: `!tbaa` for type-based alias analysis, `!alias.scope` and `!noalias` for scoped alias sets, `align` and `dereferenceable` attributes. Aliasing is opt-in at the IR level; the optimizer uses metadata to assert non-aliasing rather than inferring it solely from types.

**Control flow.** GIMPLE basic blocks are sequences of `gimple` statements terminated by a `GIMPLE_COND`, `GIMPLE_GOTO`, `GIMPLE_SWITCH`, or `GIMPLE_RETURN`. The CFG is maintained as a separate `basic_block` data structure with predecessor/successor edge lists.

LLVM IR basic blocks are `BasicBlock` objects (a subclass of `Value`) containing a list of `Instruction` objects, terminated by a terminator instruction (`br`, `switch`, `ret`, `invoke`, `indirectbr`, etc.). The CFG is implicit in the terminators; there is no separate edge data structure for the non-exceptional CFG (though LLVM maintains predecessor lists for efficiency).

**φ-functions.** GCC's `GIMPLE_PHI` statements are separate from ordinary GIMPLE statements and stored in a parallel list per basic block. They are not `tree` expressions and cannot appear in expression positions.

LLVM IR `phi` instructions are ordinary `Instruction` subclasses. They appear at the top of basic blocks and are values — their result can be used as an operand by any subsequent instruction. This uniformity simplifies passes: a pass that walks instructions sees φ-nodes and arithmetic through the same interface.

### RTL versus MachineIR

GCC's RTL is an s-expression tree: `(set (reg:SI %eax) (plus:SI (reg:SI %ecx) (const_int 4)))`. Every RTL expression is typed by machine mode (`QI`, `HI`, `SI`, `DI`, `SF`, `DF` for 8/16/32/64-bit integers and single/double floats). Patterns in the `.md` machine description file are matched against RTL expressions using a tree-matching algorithm.

LLVM's `MachineInstr` is a flat list of `MachineOperand` objects: register, immediate, memory reference, or symbol. Machine instructions are identified by opcode (an integer defined by TableGen). The operand list is positional: for a given opcode, operand 0 is the destination register, operands 1..N are sources, per the TableGen definition. This flat structure is easier to serialize and inspect but requires opcode-specific knowledge to interpret operands.

Both representations support virtual registers (pre-allocation) and physical registers (post-allocation). GCC's virtual registers are `PSEUDO_REGNO` values; LLVM's are `Register` values with index ≥ `FirstVirtualRegister`. Both systems provide a register class hierarchy describing which physical registers can satisfy which virtual register uses.

---

## I.3 Pass Infrastructure

### GCC's Pass Registry

GCC organizes its passes in `gcc/passes.def`, a file of macros that define the pass pipeline as a nested structure:

```c
/* gcc/passes.def (simplified) */
NEXT_PASS (pass_build_cfg);
NEXT_PASS (pass_early_tree_profile);
NEXT_PASS (pass_cleanup_cfg);
/* ... many more ... */
PUSH_INSERT_PASSES_WITHIN (pass_all_optimizations)
  NEXT_PASS (pass_remove_cgraph_callee_edges);
  NEXT_PASS (pass_early_inline);
  NEXT_PASS (pass_ipa_free_lang_data);
  /* ... */
POP_INSERT_PASSES ()
```

Each pass is a subclass of `opt_pass` (defined in `gcc/tree-pass.h`), with virtual methods `execute()` and `gate()`. The `gate()` method returns a boolean indicating whether the pass should run given the current optimization level and flags. Passes are registered with the pass manager at startup and form a static pipeline.

The static nature of the pipeline is a design constraint: adding a new pass requires modifying GCC's source. The plugin API allows inserting passes at specific named positions (`PLUGIN_PASS_MANAGER_SETUP`), but the positions available are predefined. A plugin can insert a GIMPLE pass after `pass_build_cfg` or before `pass_all_optimizations`, but it cannot restructure the interprocedural analysis pipeline.

GCC's interprocedural passes run as part of the link-time optimization (LTO) pipeline, where the full call graph is available. The `ipa_pass` subclass represents passes that require the full call graph. IPA passes run in a fixed order determined by their `pos_summary_id` field.

### LLVM's PassManager

LLVM's new PassManager (introduced in LLVM 4, fully replacing the legacy PM in LLVM 13) is a composable, type-safe infrastructure. The three pass levels — module, CGSCC (Call Graph Strongly Connected Component), and function — are explicit:

```cpp
PassBuilder PB;
LoopAnalysisManager LAM;
FunctionAnalysisManager FAM;
CGSCCAnalysisManager CGAM;
ModuleAnalysisManager MAM;

PB.registerModuleAnalyses(MAM);
PB.registerCGSCCAnalyses(CGAM);
PB.registerFunctionAnalyses(FAM);
PB.registerLoopAnalyses(LAM);
PB.crossRegisterProxies(LAM, FAM, CGAM, MAM);

ModulePassManager MPM = PB.buildPerModuleDefaultPipeline(OptimizationLevel::O2);
MPM.run(*M, MAM);
```

Passes are typed by the IR unit they operate on: `PassInfoMixin<MyFunctionPass>` for function passes, `PassInfoMixin<MyModulePass>` for module passes. The pass manager enforces that a function pass cannot access module-level state directly — it must go through the `ModuleAnalysisManagerFunctionProxy`.

Out-of-tree passes load as shared libraries via the `PassPlugin` mechanism:

```cpp
// MyPass.cpp — out-of-tree plugin
extern "C" ::llvm::PassPluginLibraryInfo LLVM_ATTRIBUTE_WEAK
llvmGetPassPluginInfo() {
  return {LLVM_PLUGIN_API_VERSION, "MyPass", LLVM_VERSION_STRING,
          [](PassBuilder &PB) {
            PB.registerPipelineParsingCallback(
                [](StringRef Name, FunctionPassManager &FPM, ...) {
                  if (Name == "my-pass") { FPM.addPass(MyPass()); return true; }
                  return false;
                });
          }};
}
```

```bash
opt -load-pass-plugin=MyPass.so -passes="my-pass" input.ll -o output.ll
```

No GCC source modification is required. The plugin registers its passes with the `PassBuilder` at load time and can insert itself at any extension point: `EP_EarlyAsPossible`, `EP_ModuleOptimizerEarly`, `EP_VectorizerStart`, `EP_OptimizerLast`, `EP_CGSCCOptimizerLate`, etc.

### Analysis Caching and Invalidation

Both systems cache analysis results to avoid recomputation. GCC uses a per-function `struct function` to store cached analyses (`df_info` for dataflow, `dominance_info` for dominators, `loop_info` for loop structure). Analyses are invalidated by calling `free_dominance_info()`, `loop_optimizer_finalize()`, etc. The invalidation is manual: pass authors must know which analyses their pass preserves and explicitly free those it does not.

LLVM's `AnalysisManager` provides automatic invalidation through the `PreservedAnalyses` return value. Every pass returns a `PreservedAnalyses` object indicating which analyses are still valid. The `AnalysisManager` consults this to decide whether cached results can be reused:

```cpp
PreservedAnalyses MyPass::run(Function &F, FunctionAnalysisManager &AM) {
  auto &DT = AM.getResult<DominatorTreeAnalysis>(F);
  // ... modify F ...
  PreservedAnalyses PA;
  PA.preserve<DominatorTreeAnalysis>();  // we maintained the domtree
  return PA;
}
```

If a pass returns `PreservedAnalyses::none()`, all cached analyses for that IR unit are discarded.

---

## I.4 Backend and Target Description

### Machine Description: `.md` Files versus TableGen

GCC describes its target ISA in machine description `.md` files, one per target (e.g., `gcc/config/i386/i386.md`, `gcc/config/aarch64/aarch64.md`). An `.md` file contains:

- **`define_insn`**: a pattern matching a specific RTL expression tree, producing assembly output. The pattern is an RTL template; the constraint specifies valid register classes and memory addressing modes.
- **`define_expand`**: a macro-like expander that generates a sequence of RTL from a higher-level operation.
- **`define_split`**: splits one RTL pattern into several simpler ones, used for instruction splitting during the backend pipeline.
- **`define_peephole2`**: a peephole optimizer pattern.

Example (simplified x86-64 add):
```scheme
(define_insn "addsi3"
  [(set (match_operand:SI 0 "nonimmediate_operand" "=r,rm")
        (plus:SI (match_operand:SI 1 "nonimmediate_operand" "%0,0")
                 (match_operand:SI 2 "general_operand" "rmn,rn")))]
  ""
  "add{l}\t{%2, %0|%0, %2}"
  [(set_attr "type" "alu")])
```

Pattern matching is tree-based: the GCC backend walks the RTL tree and tries each `define_insn` pattern in priority order until one matches. The constraint strings (`"=r,rm"`) encode register/memory alternatives for the instruction selector and register allocator.

LLVM uses TableGen (`.td` files) to describe targets. TableGen is a domain-specific language that generates C++ code (instruction definitions, register descriptions, scheduling models) at build time. An LLVM instruction definition:

```tablegen
// llvm/lib/Target/X86/X86InstrArithmetic.td (simplified)
def ADD32rr : I<0x01, MRMDestReg, (outs GR32:$dst), (ins GR32:$src1, GR32:$src2),
               "add{l}\t{$src2, $dst|$dst, $src2}",
               [(set GR32:$dst, EFLAGS, (X86add_flag GR32:$src1, GR32:$src2))]>;
```

The `(outs)`/`(ins)` operand lists define the instruction's register operands with explicit register class constraints (`GR32` = 32-bit general registers). The DAG pattern `[(set GR32:$dst, ...)]` is matched by SelectionDAG's instruction selector. TableGen generates a `X86GenInstrInfo.inc` file containing the opcode enum, operand descriptors, and pattern tables consumed by the `X86InstrInfo` C++ class.

The practical difference: GCC `.md` files are interpreted at runtime by a pattern matcher; LLVM TableGen generates C++ code compiled into the compiler. LLVM's approach adds a build step but produces a faster instruction selector (no runtime pattern interpretation) and enables compile-time checking of the target description.

### Register Allocation

**GCC IRA (Integrated Register Allocator)**, introduced in GCC 4.4 and described in `gcc/ira.h`, uses a graph-coloring approach. IRA builds a conflict graph (interference graph) over live ranges, then colors it by assigning physical registers to pseudo-registers. IRA is "integrated" in the sense that it interleaves register class preference propagation with coloring, rather than running them as separate phases. When IRA cannot find a valid coloring, it triggers **spilling** — inserting loads and stores around uses of spilled pseudo-registers. GCC's older reload pass (`gcc/reload.cc`) has been largely superseded by LRA (Local Register Allocator), which handles complex addressing modes that IRA cannot directly handle.

**LLVM Greedy Register Allocator** (`llvm/lib/CodeGen/RegAllocGreedy.cpp`) is LLVM's default. It uses a priority queue of live ranges ordered by spill weight and assigns physical registers iteratively. When a live range cannot be assigned, it attempts **splitting** (splitting the live range into shorter intervals that can be individually colored) before falling back to spilling. The splitting heuristics — region splitting, per-block splitting, local splitting — are the key differentiator from simple graph coloring. LLVM also provides PBQP (Partitioned Boolean Quadratic Programming) allocation for targets with complex instruction-level register constraints (used on some embedded targets).

Both allocators support register coalescing (eliminating copy instructions by assigning source and destination the same physical register) and caller/callee-saved register conventions. LLVM's `LiveRangeEdit` infrastructure makes splitting and rematerialization first-class operations; GCC's analogous infrastructure is more tightly coupled to the reload framework.

### Instruction Selection

**GCC** selects instructions via the RTL expander (`expand_expr` in `gcc/expr.cc`), which translates GIMPLE statements to RTL using the `define_expand` and `define_insn` patterns. A second instruction selection pass — the `combiner` — performs local peephole-style optimizations by combining multiple RTL instructions into a single one that matches a `define_insn` pattern. This two-phase approach (expand then combine) means GCC's instruction selection is inherently local.

**LLVM** offers two instruction selection frameworks:

**SelectionDAG** (the original, `llvm/lib/CodeGen/SelectionDAG/`) converts LLVM IR into a directed acyclic graph (DAG) of `SDNode` objects, one basic block at a time. The DAG is then matched against the target's instruction patterns using a generated table-driven selector (`X86ISelDAGToDAG.cpp`). SelectionDAG enables cross-instruction optimizations within a basic block (DAG combining) and exposes the full structure of complex operations to the selector.

**GlobalISel** (`llvm/lib/CodeGen/GlobalISel/`) is the newer framework, working directly on `MachineInstr` without an intermediate DAG. GlobalISel operates on a generic IR (`G_ADD`, `G_LOAD`, `G_STORE`, etc.) that is progressively lowered to target-specific instructions through `Legalizer`, `RegBankSelect`, and `InstructionSelect` passes. GlobalISel enables interprocedural instruction selection (the name refers to global scope, not global in the LTO sense) and is already the default for AArch64 at `-O0`.

---

## I.5 Frontend and Diagnostic Quality

### AST Architecture

Clang builds an immutable Abstract Syntax Tree. Every `Decl`, `Stmt`, and `Expr` node is allocated in a `ASTContext`-owned arena and never modified after construction. This immutability enables:
- **Serialization**: the full AST can be written to a PCH (precompiled header) or module file and read back without reconstruction. The `ASTWriter`/`ASTReader` pair in Clang handles this.
- **Parallel use**: multiple threads can traverse the AST simultaneously without synchronization.
- **Source fidelity**: every node retains its source location (`SourceLocation`, `SourceRange`), enabling precise diagnostics and refactoring tools.

GCC builds a mutable `tree` node graph. `tree` nodes (`gcc/tree.h`) are tagged union structures with a type code (e.g., `FUNCTION_DECL`, `CALL_EXPR`, `PLUS_EXPR`) and a fixed set of operand slots accessed via macros (`TREE_OPERAND(t, 0)`, `DECL_NAME(t)`, etc.). The graph is modified in place throughout compilation: optimization passes rewrite tree nodes, fold constants, and substitute expressions. GCC does not serialize its GENERIC/GIMPLE representation in a way that enables precompiled headers from the same IR (GCC's PCH format serializes the preprocessor state and some symbol table information, not the full GIMPLE).

### C++ Template Instantiation

Both compilers implement C++ template instantiation, but their approaches differ in observable ways.

Clang eagerly builds a `TemplateArgumentList` and instantiates templates on demand, caching instantiated specializations in the `ASTContext`. Template specialization decisions are made during semantic analysis; the instantiated AST nodes flow through the same pipeline as non-template code. Clang's `CXXRecordDecl` for a class template specialization preserves the template arguments and the point of instantiation, which is why Clang can produce diagnostics that point into template argument lists with precise source locations.

GCC instantiates templates during the GIMPLE phase. Template specializations produce `GENERIC` trees that flow through the same `gimplify` pipeline as non-template code. GCC's diagnostics for template errors tend to show the internal form of the tree rather than the source-level expression, which is why Clang's template error messages are generally considered more readable.

### Diagnostic Quality

Clang's diagnostic infrastructure (`clang/lib/Frontend/DiagnosticRenderer.cpp`) was explicitly designed to produce human-oriented error messages. Key features:
- **Fix-it hints**: `diag.Note(Loc) << FixItHint::CreateReplacement(Range, NewText)` — many Clang diagnostics suggest the corrected code inline.
- **Caret diagnostics**: the error is underlined with `^~~~` indicating the precise token range, not just the line.
- **Include stack**: for errors in included headers, Clang shows the inclusion chain with source locations at each level.
- **Template instantiation notes**: `note: in instantiation of function template specialization 'foo<int>' requested here` chains are shown with the full instantiation context.

GCC has improved its diagnostics substantially from GCC 7 onward, including caret-style output and some fix-it suggestions. The `gcc.gnu.org/bugzilla/` tracker shows active work on diagnostic quality. As of GCC 14, GCC's diagnostics are competitive for common errors but still generally behind Clang on template-heavy C++ code.

### Language Server Protocol

Clang's language server, `clangd` (`clang-tools-extra/clangd/`), is a production-quality LSP server built directly on Clang's AST and indexing infrastructure. It provides:
- Code completion backed by actual semantic analysis
- Go-to-definition for macros, templates, and overload sets
- Find-references using an AST-based index
- Refactoring operations (rename, extract function) via Clang's `Tooling` library
- Real-time diagnostics using the same pipeline as `clang -fsyntax-only`

GCC has `ccls` (an LSP server using the GCC plugin API or libclang) and experimental in-tree LSP work, but there is no production-quality GCC-native language server comparable to `clangd`. Most IDEs that target GCC use `clangd` with a compile-commands database generated from GCC builds.

### GNU Extension Compatibility

Clang implements a large subset of GCC's extension vocabulary:
- `__attribute__((packed))`, `__attribute__((aligned(N)))`, `__attribute__((visibility("hidden")))`, `__attribute__((noreturn))`, and hundreds more
- `__builtin_expect`, `__builtin_unreachable`, `__builtin_popcount`, `__builtin_clz`, and the full `__builtin_*` namespace
- Statement expressions: `({ int x = f(); x * 2; })`
- Nested functions (Clang rejects these; GCC supports them via trampolines on the stack)
- `__label__` for explicitly declared local labels
- Designated initializers (now standard in C99/C++20 but GCC had them earlier)
- Zero-length arrays and flexible array members
- `typeof(expr)` / `__typeof__(expr)`

Clang's `-fgnuc-version=4.2.1` flag (set by default on Linux) makes Clang advertise itself as GCC 4.2.1 to preprocessor conditionals that check `__GNUC__`. This allows system headers written for GCC to accept Clang without modification.

Extensions that Clang does not support:
- GCC nested functions with upward funarg closures (stack-allocated trampolines)
- Some GCC-specific OpenMP extensions
- GNAT-specific Ada extensions
- Some GCC Fortran extensions used in legacy HPC code

---

## I.6 Toolchain Ecosystem

| Component | GCC Ecosystem | LLVM Ecosystem |
|-----------|---------------|----------------|
| **C/C++ compiler** | `gcc` / `g++` | `clang` / `clang++` |
| **Fortran** | `gfortran` | `flang` |
| **Ada** | GNAT (`gnat`) | None (some experimental work) |
| **Go** | `gccgo` | `gollvm` (experimental); most Go uses `gc` |
| **Assembler** | GNU as (GAS, `gas`) | LLVM integrated assembler (`clang -integrated-as`) |
| **Linker** | GNU ld; gold (`ld.gold`) | LLD (`ld.lld`); mold (third-party) |
| **Debugger** | GDB | LLDB |
| **C++ runtime library** | libstdc++ | libc++ |
| **C++ ABI runtime** | `libsupc++` (part of libstdc++) | libc++abi |
| **Unwinder** | `libgcc_s` / GNU libunwind | `libunwind` (LLVM) |
| **Compiler runtime** | `libgcc` | `compiler-rt` (`builtins/`) |
| **Sanitizers** | ASan, UBSan (shared code with LLVM) | ASan, MSan, TSan, UBSan, HWASan, LSan, XRay |
| **Coverage** | `gcov`; `lcov` for HTML | `llvm-cov`; `llvm-profdata` |
| **PGO** | `-fprofile-generate` / `-fprofile-use` | `-fprofile-generate` / `-fprofile-use`; AutoFDO; BOLT |
| **JIT interface** | `libgccjit` | OrcJIT; MCJIT (deprecated); LLVM-C JIT API |
| **Static analyzer** | none in-tree; Frama-C (external) | `clang -analyze`; `clang-sa`; CodeChecker |
| **Linter / style** | none in-tree | `clang-tidy`; `clang-format` |
| **Refactoring** | none in-tree | `clang-tools-extra`; `clangd` |
| **Archiver** | GNU `ar` | `llvm-ar` |
| **Object dump** | `objdump` (binutils) | `llvm-objdump` |
| **Symbol table** | `nm` (binutils) | `llvm-nm` |
| **Strip** | `strip` (binutils) | `llvm-strip` |
| **MLIR** | none | `mlir-opt`; full dialect ecosystem (Part XIX–XXI) |

### Linker Comparison: GNU ld versus LLD

GNU ld processes object files sequentially in a single-threaded link. For large projects, link time dominates the build. `gold` (an alternative GNU linker) added threading for some phases and improved incremental linking, but gold is no longer actively developed.

LLD (`lld`) is fully parallel. Input object files are parsed in parallel; symbol table construction and relocation processing are lock-free where possible. For large C++ projects with thousands of objects, LLD links 2–5× faster than GNU ld. LLD supports ELF (Linux), Mach-O (macOS), COFF/PE (Windows), and WebAssembly in a single binary.

LLD also supports ThinLTO natively: during a ThinLTO link, LLD reads the LLVM bitcode summary from each `.o` file, constructs the import/export plan, and launches per-module optimization in parallel worker threads — no separate LTO runner is needed. GNU ld requires the `ld.bfd --plugin` interface and a separate `liblto_plugin.so` to handle LTO, which serializes the cross-module analysis.

### Sanitizer Comparison

Both GCC and Clang implement ASan (AddressSanitizer) and UBSan (UndefinedBehaviorSanitizer) using shared runtime code from the LLVM `compiler-rt` project. GCC adopted the compiler-rt sanitizer runtimes, so the instrumentation logic and runtime behavior of ASan and UBSan are effectively identical between the two compilers.

Sanitizers available only in LLVM:
- **MemorySanitizer (MSan)**: detects reads from uninitialized memory using shadow memory. GCC does not implement MSan.
- **HWAddressSanitizer (HWASan)**: uses hardware top-byte ignore (TBI) on AArch64 for lower-overhead heap and stack safety. GCC does not implement HWASan.
- **LeakSanitizer (LSan)**: heap leak detection. Integrated into ASan but also standalone. GCC does not implement standalone LSan.
- **ThreadSanitizer (TSan)**: data race detection. Available in both, but LLVM's version is more actively maintained.
- **XRay**: low-overhead function tracing with a runtime-switchable sled mechanism. GCC does not implement XRay.

### PGO and BOLT

Both compilers support profile-guided optimization via instrumentation (`-fprofile-generate` / `-fprofile-use`). The profiling data format differs: GCC uses GCDA files; LLVM uses raw profile files processed by `llvm-profdata merge`.

LLVM additionally supports:
- **AutoFDO**: profile from Linux `perf` or `simpleperf` without recompilation for instrumentation. GCC supports this as well via `perf2bolt` or `create_gcov`.
- **BOLT** (Binary Optimization and Layout Tool): a post-link binary optimizer that reorders functions and basic blocks based on profiles, reducing instruction cache misses. BOLT operates on the final linked binary; it is independent of which compiler produced it. GCC has no equivalent post-link optimizer in the standard toolchain.

---

## I.7 Optimization Capability Comparison

### Shared Optimizations

Both compilers implement the same core middle-end optimization repertoire:

| Optimization | GCC pass | LLVM pass |
|-------------|----------|-----------|
| Constant folding | `fold_stmt` (gimple-fold.cc) | `ConstantFolding.cpp`, `InstructionSimplify.cpp` |
| Global value numbering | `pass_fre` (tree-ssa-sccvn.cc) | `GVN.cpp`, `NewGVN.cpp` |
| Sparse conditional constant propagation | `pass_ccp` | `SCCP.cpp` |
| Dead code elimination | `pass_dce` | `DCE.cpp`, `ADCE.cpp` |
| Function inlining | `pass_early_inline`, `pass_ipa_inline` | `Inliner.cpp`, `AlwaysInliner.cpp` |
| Loop invariant code motion | `pass_lim` | `LICM.cpp` |
| Strength reduction | `pass_strength_reduce` | `LoopStrengthReduce.cpp` |
| Auto-vectorization | `pass_vectorize` (tree-vect-*.cc) | `LoopVectorize.cpp`, `SLPVectorize.cpp` |
| Alias analysis | `gcc_nonstandard_alias` + TBAA | `AliasAnalysis.cpp`, TBAA, BasicAA, SCEVAA |
| Tail call optimization | `pass_tail_calls` | InstCombine + `tailcallelim` pass |
| Devirtualization | `pass_ipa_devirt` | `DevirtUtils.cpp` + speculative devirt |

### GCC-Specific Strengths

**Graphite polyhedral loop optimizer.** GCC's Graphite (`gcc/graphite*.cc`) applies the polyhedral model (see [Part XI — Polyhedral Theory](../part-11-polyhedral-theory/)) to loop nests, using the ISL library for integer set arithmetic. Graphite can perform loop interchange, loop tiling, and loop fusion automatically for affine loop nests. LLVM's Polly is a polyhedral optimizer for LLVM IR, but it is an out-of-tree project (not built by default) with less integration into the standard optimization pipeline. For HPC workloads with deep loop nests, GCC's Graphite is more accessible than Polly.

**Ada (GNAT).** GNAT (GNU Ada) is the reference implementation of Ada 2022, developed by AdaCore and contributed to GCC. GNAT is deeply integrated with GCC's middle-end and benefits from decades of production use. LLVM has no Ada frontend; the closest is the GNAT LLVM project (developed by AdaCore as `gnat-llvm`), which translates GNAT's GENERIC output to LLVM IR, but it is not part of the LLVM monorepo and trails the GNAT release cadence.

**Fortran (gfortran).** GCC's `gfortran` is a mature, production-quality Fortran 2018 compiler. LLVM's Flang (Part XVIII) has reached production status for Fortran 2018 and is included in the LLVM monorepo, but `gfortran` has a larger installed base and more complete coverage of legacy Fortran dialects used in HPC.

**Go (gccgo).** GCC's `gccgo` front end compiles Go using GCC's middle-end and backends, producing generally faster code than the reference `gc` compiler for compute-intensive code. LLVM's `gollvm` is experimental and not actively developed. Most Go programmers use `gc` regardless of compiler.

### LLVM-Specific Strengths

**ThinLTO at scale.** LLVM's ThinLTO ([Chapter 77 — ThinLTO and Distributed Compilation](../part-13-lto-whole-program/ch77-thinlto-and-distributed-compilation.md)) performs whole-program optimization with parallelism proportional to the number of modules. The two-phase design (summary phase + parallel per-module optimization with cross-module imports) makes ThinLTO practical for codebases with hundreds of thousands of source files. GCC's LTO is monolithic by default; its `-flto=N` parallelism option runs N linker plugin processes, but the whole-program analysis (IPA) is serial.

**MLIR.** LLVM's Multi-Level Intermediate Representation ([Part XIX–XXI](../part-19-mlir-foundations/)) is a compiler infrastructure framework with no equivalent in GCC. ML frameworks (JAX, IREE, Torch-MLIR), hardware compilers (CIRCT), and HPC compilers (Flang's HLFIR) all use MLIR. GCC has no extensible dialect system.

**JIT compilation.** OrcJIT ([Chapter 108 — OrcJIT and the MCJIT Interface](../part-16-jit-sanitizers/ch108-orcjit-and-the-mcjit-interface.md)) provides a full, production-grade JIT infrastructure with lazy compilation, resource tracking, custom memory managers, and remote JIT execution. `libgccjit` exposes a JIT API, but it does not support lazy compilation, remote execution, or the level of IR-level control available through OrcJIT.

**Sanitizer breadth.** LLVM's compiler-rt provides MSan, HWASan, XRay, and other sanitizers unavailable in GCC (as discussed in §I.6).

**BOLT.** Post-link binary optimization with no GCC equivalent (§I.6).

**Clang Static Analyzer.** The Clang Static Analyzer (`clang -analyze`) performs path-sensitive, interprocedural analysis using a symbolic execution engine (the `ExplodedGraph`). It detects null dereferences, use-after-free, memory leaks, and API misuse bugs that escape compile-time checking. GCC has no in-tree equivalent; external tools like Frama-C provide similar analysis but are not integrated into the compiler driver.

---

## I.8 Language and Target Support Matrix

### Language Support

| Language | GCC | LLVM/Clang |
|----------|-----|------------|
| C (C23) | Full | Full |
| C++ (C++23) | Full | Full |
| C++ (C++26) | Partial | Partial |
| Objective-C | Supported | Full (Apple's reference) |
| Objective-C++ | Supported | Full |
| Fortran (F2018) | Full (`gfortran`) | Production (`flang`) |
| Ada (Ada 2022) | Full (GNAT) | Experimental (`gnat-llvm`) |
| Go | Supported (`gccgo`) | Experimental (`gollvm`) |
| D | Supported (`gdc`) | Supported (`ldc2`, third-party) |
| Rust | Backend (`rustc_codegen_gcc`) | Production backend (`rustc`) |
| Swift | None | Full (Apple's implementation) |
| Julia | None | Full (uses LLVM directly) |
| Kotlin/Native | None | Full (uses LLVM) |
| CUDA | None | Full (`clang -cuda-gpu-arch=`) |
| HIP (AMD GPU) | Experimental | Full (`clang --offload-arch=`) |
| OpenCL | Supported | Full |
| HLSL | None | Full (`clang -x hlsl`) |
| Zig | None | Full (uses LLVM directly) |

### Target Architecture Support

| Architecture | GCC | LLVM |
|-------------|-----|------|
| x86 / x86-64 | Mature | Mature |
| AArch64 / ARM | Mature | Mature |
| RISC-V | Mature | Mature |
| PowerPC / PowerPC64 | Mature | Mature |
| MIPS / MIPS64 | Mature | Maintained |
| SPARC / SPARC64 | Mature | Maintained |
| s390 / z Architecture | Mature | Maintained |
| AVR (microcontroller) | Mature | Supported |
| MSP430 | Mature | Supported |
| NVPTX (NVIDIA GPU) | Supported (`nvptx-none`) | Full |
| AMDGPU (AMD GCN) | Supported | Full |
| SPIR-V | Partial | Full (`spirv-unknown-unknown`) |
| WebAssembly | Supported | Mature |
| eBPF | Supported | Mature |
| Hexagon (Qualcomm) | None | Supported (Qualcomm develops) |
| Lanai | None | Supported (tutorial target) |
| SystemZ (IBM mainframe) | Mature | Mature |
| XCore | None | Supported |

**Cross-compilation model.** GCC requires a separate toolchain built for each `host-build-target` triple. Compiling C for `aarch64-linux-gnu` on an `x86_64-linux-gnu` host requires a separately built `aarch64-linux-gnu-gcc` binary with a sysroot containing AArch64 libraries and headers. This is manageable but requires toolchain generation infrastructure (`crosstool-NG`, buildroot, Yocto).

LLVM's model is different: a single `clang` binary built for `x86_64-linux-gnu` can target any architecture for which LLVM has a backend, provided the appropriate sysroot is available. `clang --target=aarch64-linux-gnu --sysroot=/path/to/sysroot` works without rebuilding the compiler. This is possible because LLVM backends are compiled into the `clang` binary via `LLVMInitialize*Target()` calls, not separate executables. The single-binary cross-compilation model has made LLVM the default for embedded systems development and Android NDK cross-compilation.

---

## I.9 ABI Interoperability

### Linking GCC and Clang Objects

For C code, object files produced by GCC and Clang are freely interchangeable on the same platform, provided both compilers agree on the target triple and ABI. Both compilers implement the platform ABI (System V AMD64 ABI on Linux x86-64, AAPCS64 on AArch64, etc.) from the same specification, and the generated machine code is indistinguishable at the object file level.

For C++ code, interoperability requires matching:

1. **Name mangling scheme.** Both Clang and GCC implement the Itanium C++ ABI name mangling on Linux and most UNIX platforms (see [Appendix H — The C++ ABI: Itanium and Microsoft](appendix-h-the-cpp-abi-itanium-and-microsoft.md)). A `std::vector<int>` mangled by GCC and by Clang produces the same symbol name on Linux. On Windows, both use the MSVC mangling scheme when targeting MSVC-compatible mode.

2. **C++ standard library.** The most common source of GCC/Clang link failures is mixing libstdc++ and libc++. These are two separate C++ standard library implementations with incompatible ABI:
   - `std::string` (libstdc++ pre-C++11) uses a reference-counted copy-on-write representation.
   - `std::string` (libstdc++ C++11 and later) uses the SSO (Small String Optimization) layout.
   - `std::string` (libc++) uses its own SSO layout, incompatible with libstdc++.
   
   Passing a `std::string` across a link boundary where one translation unit was compiled against libstdc++ and another against libc++ produces undefined behavior. The fix is to ensure all translation units in a link use the same standard library.

3. **Exception handling tables.** Both compilers use the Itanium zero-cost exception ABI (`__cxa_throw`, `__cxa_begin_catch`, personality routines) on Linux. Object files can be mixed freely in the presence of exceptions, provided the same `libunwind` / `libgcc_s` implementation handles stack unwinding. Mixing LLVM `libunwind` with code compiled against `libgcc_s` exception tables is generally safe because both implement the same Itanium ABI unwind protocol, but edge cases exist with `.eh_frame` section format differences.

### Known ABI Divergences

**Bit-field layout.** GCC and Clang have occasional differences in how they lay out bit-fields at the boundaries of `packed` structures or when mixing bit-field types. The C standard leaves bit-field layout implementation-defined; both compilers follow the platform ABI where it is specified (ARM PCS, System V ABI) but may diverge on edge cases. The `offsetof` macro and `static_assert` can catch these at compile time.

**`__attribute__((packed))` and alignment.** GCC's `packed` attribute unconditionally removes alignment padding. Clang matches this behavior. However, GCC historically allowed unaligned `int` accesses within packed structs on targets where the hardware supports it (x86); Clang may emit different load/store sequences for the same structure. This is not an ABI issue (the struct layout is identical) but can produce different performance characteristics.

**VLAs (Variable-Length Arrays).** C99 VLAs are handled identically in GCC and Clang on the stack. C++ VLAs are a GCC extension; Clang supports them as an extension with `-Wvla` but may warn or differ in edge cases.

**`__int128` and `long double`.** Both compilers agree on `__int128` layout (128-bit, 16-byte aligned on x86-64). `long double` is platform-dependent: 80-bit extended precision on x86 (both compilers agree), 128-bit quad precision on SPARC and PowerPC (both agree), 64-bit (same as `double`) on ARM and AArch64 (both agree).

### Mixed Toolchain in Practice: ClangBuiltLinux

The ClangBuiltLinux project (covered in [Chapter 200 — Linux Kernel Compilation with LLVM/Clang](../part-29-tooling-kernel-binary/ch200-linux-kernel-compilation-with-llvm-clang.md)) demonstrates the largest-scale mixed-toolchain interoperability deployment in practice. The Linux kernel was historically built exclusively with GCC; ClangBuiltLinux achieved full kernel compilation with Clang, LLD, and the LLVM integrated assembler.

Key compatibility requirements for the transition:
- All `__attribute__` extensions used by the kernel must be supported by Clang.
- Inline assembly syntax must be acceptable to both GAS and LLVM's IAS.
- The kernel's use of GCC-specific sanitizer callbacks (`__sanitizer_cov_trace_pc_guard`) must map to LLVM's sanitizer ABI.
- LLD must handle all linker scripts, section ordering requirements, and relocation types used by the kernel's architecture-specific link scripts.

The ClangBuiltLinux experience shows that GCC-to-Clang migration in a large C codebase is achievable, but requires systematic attention to extension compatibility, inline assembly differences, and sanitizer ABI alignment.

### `rustc_codegen_gcc` and `libgccjit`

The experimental `rustc_codegen_gcc` backend (`rustc`'s alternative to `rustc_codegen_llvm`) uses GCC's `libgccjit` API to generate code from Rust's MIR. The motivation is to gain GCC's target support for architectures where LLVM has no backend (some older embedded targets, OpenRISC, etc.) and to benefit from GCC's optimizations on platforms where GCC historically produces better code.

`libgccjit` exposes a C API (`libgccjit.h`) that allows programs to construct GCC's GENERIC representation programmatically, then compile and optionally execute it. The API surface is deliberately narrow — it exposes types, locations, functions, basic blocks, and a limited set of expressions — which is both a strength (stability) and a limitation (cannot access the full IPA optimization pipeline programmatically).

The rustc/GCC interoperability path through `libgccjit` is one of the few examples of a compiler using GCC as a library in the way that projects routinely use LLVM. Its narrow API compared to LLVM's full C++ API reflects GCC's design choice not to expose its internals, and `rustc_codegen_gcc` must work within those constraints.

---

## Summary

The table below distills the key structural differences:

| Dimension | GCC | LLVM |
|-----------|-----|------|
| Architecture | Monolithic application | Library collection |
| IR stability | GIMPLE/RTL are private internals | LLVM IR is a public API |
| License | GPLv3 | Apache 2.0 with LLVM exception |
| IR levels | 4 (GENERIC→GIMPLE→SSA-GIMPLE→RTL) | 2 (LLVM IR → MachineIR) |
| Type system | C-rooted `tree` nodes | Typed value model, signedness-free integers |
| Pass extension | Plugin API with fixed hooks | Out-of-tree plugins at any extension point |
| Target description | `.md` pattern files (runtime-interpreted) | TableGen `.td` (compile-time generated C++) |
| Instruction selection | RTL expander + combiner | SelectionDAG or GlobalISel |
| Register allocation | IRA / LRA graph coloring | Greedy (splitting) / PBQP |
| Polyhedral optimization | Graphite (in-tree, on by default) | Polly (out-of-tree) |
| MLIR / multi-level IR | None | Full ecosystem |
| JIT | `libgccjit` (narrow API) | OrcJIT (full IR access) |
| Language server | None in-tree | `clangd` (production quality) |
| Sanitizers | ASan, UBSan, TSan | ASan, MSan, HWASan, TSan, UBSan, LSan, XRay |
| Post-link optimization | None | BOLT |
| Cross-compilation | Separate per-target toolchain build | Single binary, `--target=` flag |
| LTO parallelism | `-flto=N` (serial IPA, parallel codegen) | ThinLTO (parallel IPA + codegen) |
| C++ runtime | libstdc++ | libc++ |
| Linker | GNU ld / gold | LLD |

Neither compiler is unconditionally superior. GCC remains the reference for Ada, the mature choice for Fortran, and the deeper choice on some embedded targets. LLVM is the infrastructure of choice for new compiler projects, ML compilers, hardware design tools, language runtimes, and any application that needs to embed compilation as a library. For C and C++ production workloads, both produce competitive code, and most large projects build and test against both.
