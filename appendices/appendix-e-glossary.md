# Appendix E — Glossary

*Quick Reference | LLVM 22.1.x / MLIR in-tree*

Alphabetical glossary of technical terms used throughout this book. Entries are concise definitions; for explanations and context, see the referenced chapter. Terms in **bold** within definitions have their own entries.

---

## A

**Abstract interpretation** — A formal framework for static analysis in which program semantics are approximated by computing over an abstract domain (e.g., sign, interval, pointer points-to sets). Soundness requires the abstraction to be a Galois connection with the concrete semantics. See [Chapter 10 — Dataflow Analysis](../chapters/part-02-compiler-theory/ch10-dataflow-analysis.md).

**ADT** (Abstract Data Type) — In LLVM context, refers to the `llvm/include/llvm/ADT/` family of containers: `SmallVector`, `DenseMap`, `StringRef`, `ArrayRef`, `StringMap`, `SetVector`, `BitVector`, etc. These are performance-tuned alternatives to STL containers.

**Affine map** — An MLIR first-class value describing a linear function from a set of dimension and symbol variables to a result: `(d0, d1)[s0] -> (d0 + s0, d1 * 2)`. Used in `affine` dialect and `linalg` indexing maps to express polyhedral iteration spaces. See [Chapter 140](../chapters/part-20-in-tree-dialects/ch140-affine-and-scf.md).

**Alias analysis** — Analysis determining whether two memory accesses may or must refer to the same location. LLVM's alias analysis infrastructure: `BasicAA`, `TBAA`, `GlobalsAA`, `SCEVAliasAnalysis`. See [Chapter 61 — Foundational Analyses](../chapters/part-10-analysis-middle-end/ch61-foundational-analyses.md).

**Alloca** — An LLVM IR instruction (`alloca <ty>`) that allocates stack space. In Clang, all local variables initially become allocas before **mem2reg** converts them to SSA **phi** nodes. Also an MLIR `memref.alloca` op.

**ARC** (Automatic Reference Counting) — A memory management strategy (used in Objective-C and Swift) where retain/release operations are inserted automatically. Clang implements ARC for Objective-C via `CGObjC*` in CodeGen.

**AsmPrinter** — LLVM backend component that converts **MachineInstr** sequences to assembly text or MC encoding. Implemented in `llvm/lib/CodeGen/AsmPrinter/`. Each target provides `AsmPrinterImpl`.

**Attribute (LLVM IR)** — A property attached to functions, parameters, or return values in LLVM IR. Function attributes: `nounwind`, `readonly`, `noreturn`. Parameter attributes: `noalias`, `byval`, `nonnull`. Stored as `AttributeList` objects. See [Appendix A](#a4-attributes).

**Attribute (MLIR)** — An immutable compile-time value attached to MLIR ops as named properties. Distinct from **operands** (runtime values). Examples: `DenseElementsAttr`, `StringAttr`, `IntegerAttr`, `DictionaryAttr`. See [Chapter 131](../chapters/part-19-mlir-foundations/ch131-type-and-attribute-systems.md).

---

## B

**Backend** — The LLVM compiler phase that takes **LLVM IR** and produces target machine code. Consists of instruction selection (**SelectionDAG**, **GlobalISel**), **register allocation**, scheduling, and MC emission. See Part XIV.

**Basic block (BB)** — A maximal straight-line sequence of instructions with a single entry point (the label) and a single exit (a terminator instruction). In LLVM IR: `%label:` followed by instructions ending with `ret`, `br`, `switch`, `invoke`, etc.

**Bisimulation** — A relation between two labeled transition systems showing they behave identically from an external observer's perspective. Used in verified compilation to show that source and target programs are semantically equivalent. See [Chapter 167](../chapters/part-24-verified-compilation/ch167-operational-semantics.md).

**Block argument** — In MLIR, basic blocks take typed arguments (like function parameters) rather than using **phi** nodes. Conversion from LLVM IR phi nodes to block arguments is done during import. See [Chapter 130 — MLIR IR Structure](../chapters/part-19-mlir-foundations/ch130-mlir-ir-structure.md).

**BTI** (Branch Target Identification) — AArch64 security feature. `BTI` instruction marks valid indirect branch targets. Clang generates BTI landing pads when `-mbranch-protection=bti` is set. Enforced by the processor's Branch Target Exception mechanism.

**Bufferization** — The process of converting **tensor** (value-semantic) IR to **memref** (buffer-semantic) IR in MLIR. LLVM 22 uses One-Shot Bufferization (`one-shot-bufferize`). See [Chapter 151 — Bufferization Deep Dive](../chapters/part-21-mlir-transformations/ch151-bufferization-deep-dive.md).

---

## C

**Calling convention (CC)** — The ABI contract specifying which registers carry arguments/return values, how the stack is laid out, and who saves/restores callee-saved registers. LLVM IR encodes CC per call site: `ccc`, `fastcc`, `coldcc`, etc. See [Appendix A.5](#a5-calling-conventions) and [Chapter 23](../chapters/part-04-llvm-ir/ch23-attributes-calling-conventions-abi.md).

**CFA** (Canonical Frame Address) — In DWARF, the reference location for frame-relative variable locations. Defined by `DW_CFA_def_cfa*` rules in `.debug_frame`/`.eh_frame`. Typically the stack pointer value at function entry.

**CFI** (Control Flow Integrity) — A class of mitigations ensuring indirect control-flow transfers (calls, returns) target only valid locations. LLVM implements CFI via `-fsanitize=cfi-*` checks using shadow stacks (ShadowCallStack) and type-based dispatch checks. See [Chapter 68 — Hardening](../chapters/part-10-analysis-middle-end/ch68-hardening-and-mitigations.md).

**ClangIR (CIR)** — An in-tree MLIR dialect and pipeline that lowers the Clang AST to MLIR before reaching LLVM IR. Enables higher-level analysis and transformations. See Part VIII.

**Codegen** — LLVM's backend pipeline from optimized IR to machine code; also used colloquially for Clang's `CodeGen/` directory that lowers AST to LLVM IR.

**Conversion (MLIR)** — A transformation that changes the dialect of ops, either fully (full conversion: all ops must be converted) or partially (partial conversion). Implemented via `ConversionTarget` + `RewritePatternSet`. See [Chapter 148 — Dialect Conversion](../chapters/part-21-mlir-transformations/ch148-dialect-conversion.md).

**CPS** (Continuation-Passing Style) — A program transformation where every function takes an extra argument (the continuation) representing "what to do next." Used in compilation of functional languages and in formalizing exception handling.

**CU** (Compute Unit) — On AMD GPU (GCN/RDNA), the fundamental compute unit containing 4 SIMD32 units. Analogous to Streaming Multiprocessors (SMs) on NVIDIA. Each CU has its own L1 cache, register file (256 KB), and Local Data Share (LDS). See [Chapter 103 — AMDGPU](../chapters/part-15-targets/ch103-amdgpu-and-rocm.md).

---

## D

**DAG** (Directed Acyclic Graph) — Data structure used in **SelectionDAG** where nodes are `SDNode` objects and edges represent data or chain dependencies. Also used in LLVM scheduling.

**DAGCombiner** — A peephole optimizer operating on the **SelectionDAG** before and after legalization. Performs algebraic simplifications, combines patterns, and creates target-specific ops. See [Chapter 85 — SelectionDAG Combining](../chapters/part-14-backend/ch85-selectiondag-combining-and-selecting.md).

**DataLayout** — An LLVM module-level declaration specifying target data characteristics: endianness, pointer width, type alignments, stack alignment. Accessed via `Module::getDataLayout()` and `DataLayout` API. See [Appendix A.2](#a2-type-system).

**DCE** (Dead Code Elimination) — Removes instructions whose results are never used. LLVM performs it in `DCEPass`, `ADCEPass` (aggressive), and throughout many other passes.

**Definitional equality** — In type theory, two types or terms are definitionally equal if they reduce to the same normal form by computation rules. Distinct from **propositional equality** (proved by a term). Used in dependently-typed languages.

**Dialect (MLIR)** — A namespace grouping ops, types, and attributes that belong to the same abstraction level or domain. Examples: `arith`, `linalg`, `gpu`. See [Chapter 129 — MLIR Philosophy](../chapters/part-19-mlir-foundations/ch129-mlir-philosophy.md) and [Appendix B](#appendix-b--mlir-dialect-quick-reference).

**Dominance** — A **basic block** A *dominates* B if every path from the function entry to B passes through A. The *dominator tree* records these relationships. Required for SSA construction and many loop analyses. See [Chapter 21](../chapters/part-04-llvm-ir/ch21-ssa-dominance-and-loops.md).

---

## E

**ELF** (Executable and Linking Format) — The standard binary format on Linux and most UNIX-like systems. Comprises a header, segment headers (for linking), and section headers (for loading). See [Appendix F](#appendix-f--object-file-format-reference).

**EVT** (Extended Value Type) — Used in **SelectionDAG** to represent the type of an `SDNode` result. Extends the MVT table with custom/extended types. The main enumerator: `MVT::SimpleValueType` for common LLVM value types.

---

## F

**FastISel** — A fast instruction selector in the LLVM backend that quickly lowers IR to **MachineInstr** without building a full **SelectionDAG**. Used at `-O0` on some targets; being replaced by **GlobalISel**. See [Chapter 86 — GlobalISel](../chapters/part-14-backend/ch86-globalisel.md).

**FIR** (Fortran IR) — The MLIR dialect used by **Flang** to represent Fortran semantics before lowering to LLVM IR. Now largely superseded by **HLFIR**. See [Chapter 126](../chapters/part-18-flang/ch126-hlfir-and-fir-dialects.md).

**Freeze** — LLVM IR instruction (`freeze <ty> %val`) that converts a **poison** or **undef** value to an arbitrary but fixed value. Safe for optimization; does not propagate poison. Added in LLVM 12.

---

## G

**GlobalISel** (Global Instruction Selection) — LLVM backend framework that replaces **SelectionDAG** + **FastISel** with a unified pipeline using `MachineInstr`-based IR (`gMIR`) throughout. Key components: `IRTranslator`, `Legalizer`, `RegBankSelect`, `InstructionSelect`. See [Chapter 86](../chapters/part-14-backend/ch86-globalisel.md).

**GVN** (Global Value Numbering) — An optimization pass that detects and eliminates redundant computations across basic blocks. LLVM's `GVNPass` combines GVN with PRE (partial redundancy elimination). See [Chapter 62 — Scalar Optimizations](../chapters/part-10-analysis-middle-end/ch62-scalar-optimizations.md).

---

## H

**HLFIR** (High-Level Fortran IR) — The primary MLIR dialect for Flang, providing higher-level Fortran semantics (arrays, character operations, ASSOCIATE/FORALL) above the level of raw FIR. Default in LLVM 21+.

**Hoare triple** — `{P} C {Q}` — a formula in **Hoare logic** asserting that if precondition `P` holds before command `C` executes, then postcondition `Q` holds after. Foundation for program verification and separation logic.

---

## I

**ILP** (Integer Linear Programming) — An optimization problem where variables are integers and the objective and constraints are linear. Used in polyhedral scheduling to find optimal loop transformations. See Part XI.

**ILP** (Instruction-Level Parallelism) — The degree to which independent instructions can execute simultaneously. Measured by the machine's issue width and instruction latencies.

**Interface (MLIR)** — A polymorphic contract that an op, type, or attribute can implement to expose a uniform API. Defined in ODS: `OpInterface`, `TypeInterface`, `AttrInterface`. Examples: `CallOpInterface`, `InferTypeOpInterface`, `LoopLikeOpInterface`. See [Chapter 133](../chapters/part-19-mlir-foundations/ch133-op-interfaces-and-traits.md).

**IPO** (Inter-Procedural Optimization) — Optimizations that analyze and transform multiple functions simultaneously. Examples: inlining, **devirtualization**, argument promotion, dead argument elimination. See [Chapter 65 — IPO](../chapters/part-10-analysis-middle-end/ch65-inter-procedural-optimizations.md).

**IR** (Intermediate Representation) — LLVM's typed, **SSA**-form representation between the frontend and backend. Exists in text (`.ll`), binary bitcode (`.bc`), and in-memory (`Value`/`Instruction` objects) forms. See Part IV.

**Isel** (Instruction Selection) — The backend phase that maps IR instructions or **SelectionDAG** nodes to target machine instructions. Implemented via **SelectionDAG**, **GlobalISel**, or **FastISel**.

**ITrees** — Interaction Trees — a coinductive data structure for modeling potentially non-terminating, effectful computations in Coq. Used in **Vellvm** to formalize LLVM IR semantics. See [Chapter 169](../chapters/part-24-verified-compilation/ch169-vellvm.md).

---

## J

**JIT** (Just-In-Time compilation) — Compilation of code at runtime rather than ahead-of-time. LLVM's JIT frameworks: **MCJIT** (legacy), **ORC JIT** (current). See [Chapter 108 — The ORC JIT](../chapters/part-16-jit-sanitizers/ch108-the-orc-jit.md).

**JNI** (Java Native Interface) — The interface between Java (or JVM languages) and native C/C++ code. LLVM can target JNI-callable native libraries.

---

## L

**Lattice** — A partially ordered set where every pair of elements has a least upper bound (join, ⊔) and a greatest lower bound (meet, ⊓). Used in **dataflow analysis** to represent approximations of program states. See [Chapter 10](../chapters/part-02-compiler-theory/ch10-dataflow-analysis.md).

**LBR** (Last Branch Record) — Intel hardware feature recording the last N taken branches in an MSR ring buffer. Used by `llvm-profgen` for SPGO sample collection without instrumentation.

**LCSSA** (Loop-Closed SSA Form) — A variant of **SSA** where all values defined inside a loop that are used outside the loop are routed through **phi** nodes at the loop exit. Required by many loop transformation passes. See [Chapter 21](../chapters/part-04-llvm-ir/ch21-ssa-dominance-and-loops.md).

**LDS** (Local Data Share) — AMD GPU on-chip scratchpad memory shared among threads in a workgroup. 64 KB per CU on GCN; up to 128 KB on RDNA3. Equivalent to CUDA `__shared__` memory.

**Legalization** — Backend phase that converts operations that the target hardware does not support into sequences of supported operations. **SelectionDAG** legalizer, **GlobalISel** legalizer. See [Chapter 84](../chapters/part-14-backend/ch84-selectiondag-building-and-legalizing.md).

**LTO** (Link-Time Optimization) — Whole-program optimization performed by the linker using IR bitcode embedded in object files. Full LTO: single global module. **ThinLTO**: per-module summaries + parallel codegen. See [Chapter 77](../chapters/part-13-lto-whole-program/ch77-lto-and-thinlto.md).

**LSDA** (Language Specific Data Area) — Target-specific exception-handling data embedded in the binary, referenced by `.eh_frame`. Contains type filter tables and action tables used by `__gxx_personality_v0` during exception unwinding.

---

## M

**Machine function** — An `llvm::MachineFunction` object: the backend's representation of a function in terms of **MachineInstr** objects and **MachineBasicBlock** lists. See [Chapter 88 — Machine IR](../chapters/part-14-backend/ch88-machine-ir.md).

**mem2reg** — LLVM pass that converts `alloca`+`load`+`store` patterns into SSA **phi** nodes. The canonical way to lift C local variables into SSA form. Runs very early in the optimization pipeline.

**MIR** (Machine IR) — A serialization format for `MachineFunction` objects; also the general name for the IR used during backend codegen. Text format: `.mir` files with YAML header + MIR body. See [Chapter 88](../chapters/part-14-backend/ch88-machine-ir.md) and [Chapter 94](../chapters/part-14-backend/ch94-mc-layer-and-mir-test-infrastructure.md).

**MMIO** (Memory-Mapped I/O) — Peripheral device registers accessed by reading/writing specific memory addresses. LLVM backends must ensure `volatile` loads/stores are not reordered or eliminated for MMIO access.

**Module** — The top-level LLVM IR container. Contains globals, function declarations, function definitions, metadata, and data layout information. In-memory: `llvm::Module`. File: `.ll` (text) or `.bc` (bitcode).

**MVT** (Machine Value Type) — An enumeration used in **SelectionDAG** to represent the type of a node: `i8`, `i32`, `f32`, `v4f32`, `iPTR`, etc. Distinguished from `llvm::Type` (used in LLVM IR) and MLIR types.

---

## N

**NF** (Normal Form) — A term that cannot be reduced further by a set of reduction rules. In lambda calculus: beta-normal form. In type theory: a canonically reduced type or term used in type-checking.

**NVPTX** — LLVM's backend for NVIDIA PTX assembly, used in the CUDA compilation path. The NVPTX target generates PTX text which is then compiled by `ptxas` to CUBIN. See [Chapter 102](../chapters/part-15-targets/ch102-nvptx-and-cuda.md).

---

## O

**Opcode** — In the backend, a numeric identifier for a machine instruction within a target's `*InstrInfo.td` tablegen file. Accessed via `MachineInstr::getOpcode()`. Example: `X86::MOV64rr`.

**Op (MLIR)** — An operation in MLIR; the fundamental unit of computation. Described by a name (e.g., `arith.addi`), operands (SSA values), results, attributes, regions, and successors. Defined via ODS. See [Chapter 130](../chapters/part-19-mlir-foundations/ch130-mlir-ir-structure.md).

---

## P

**PAC** (Pointer Authentication Code) — AArch64 security feature. Embeds a cryptographic MAC in unused pointer bits using hardware keys (`PACIA`, `PACIB`, etc.). Defends against ROP/JOP by requiring valid PAC to call/return. LLVM supports via `-msign-return-address=all` and `-mbranch-protection`.

**Pass** — An LLVM or MLIR transformation or analysis unit. In LLVM: `FunctionPass`, `ModulePass`, etc. (legacy); `PassInfoMixin<>` + `run(Function&, FunctionAnalysisManager&)` (new PM). In MLIR: `OperationPass<>`. See [Chapter 59 — New Pass Manager](../chapters/part-10-analysis-middle-end/ch59-new-pass-manager.md).

**Pass manager** — Orchestrates the execution of passes and manages the caching/invalidation of analyses. LLVM: `PassBuilder`, `FunctionPassManager`, `ModulePassManager`. MLIR: `PassManager`. See [Chapter 59](../chapters/part-10-analysis-middle-end/ch59-new-pass-manager.md) and [Chapter 149](../chapters/part-21-mlir-transformations/ch149-pass-infrastructure.md).

**Pattern rewriting (MLIR)** — A mechanism for transforming MLIR ops by matching patterns and applying rewrites via `RewritePattern` / `ConversionPattern`. Core infrastructure: `PatternApplicator`, `GreedyPatternRewriteDriver`. See [Chapter 147](../chapters/part-21-mlir-transformations/ch147-pattern-rewriting.md).

**PGO** (Profile-Guided Optimization) — Using runtime profile data to guide compiler decisions (inlining, BB layout, branch prediction hints). LLVM supports instrumentation PGO (`-fprofile-generate/-fprofile-use`) and sampling PGO (`-fprofile-sample-use`). See [Chapter 67](../chapters/part-10-analysis-middle-end/ch67-profile-guided-optimization.md).

**PHI node** — In **SSA** form, a special pseudo-instruction at the start of a basic block that selects a value based on which predecessor was last executed: `%r = phi i32 [%a, %BB1], [%b, %BB2]`. MLIR replaces phi nodes with **block arguments**.

**PLT** (Procedure Linkage Table) — An ELF mechanism for lazy binding of dynamic library calls. The first call to an external function goes through a PLT stub that invokes `_dl_runtime_resolve`, which fills the GOT entry. See [Chapter 79](../chapters/part-13-lto-whole-program/ch79-linker-internals-got-plt-tls.md) and [Appendix F](#appendix-f--object-file-format-reference).

**Poison** — An LLVM IR special value that is neither `undef` nor a normal value; it propagates through operations and causes undefined behavior if used by certain operations (e.g., control flow). Distinct from `undef`. Related to `nsw`/`nuw` overflow semantics. See [Chapter 171](../chapters/part-24-verified-compilation/ch171-undef-poison-formally.md).

**Polyhedral model** — A mathematical framework for representing loop nests as **polyhedra** and applying affine transformations for parallelization and locality optimization. See Part XI and [Chapter 74 — Polly](../chapters/part-12-polly/ch74-polly-architecture.md).

**PTX** (Parallel Thread Execution) — NVIDIA's pseudo-assembly language for CUDA programs. An intermediate representation that NVIDIA's `ptxas` compiles to CUBIN for a specific GPU architecture.

---

## R

**Regalloc** (Register allocation) — Backend phase that maps virtual registers to physical machine registers, spilling to the stack when registers run out. LLVM allocators: `RegAllocGreedy` (default at `-O>0`), `RegAllocBasic`, `RegAllocFast`. See [Chapter 90](../chapters/part-14-backend/ch90-register-allocation.md).

**Region (MLIR)** — A container inside an MLIR op that holds a list of basic blocks. Regions provide structured control flow (unlike LLVM IR's flat CFG). An op may have zero or more regions. See [Chapter 130](../chapters/part-19-mlir-foundations/ch130-mlir-ir-structure.md).

**Relocation** — A directive in an object file indicating a location that must be patched at link or load time to reflect the actual runtime address of a symbol. See [Appendix F](#appendix-f--object-file-format-reference).

---

## S

**SCCP** (Sparse Conditional Constant Propagation) — An optimization that simultaneously propagates constant values and prunes unreachable code using the SSA graph structure. More powerful than simple constant folding. LLVM: `SCCPPass`.

**SCoP** (Static Control Part) — A maximal program region in which all loop bounds, conditionals, and array accesses are **affine** functions of loop induction variables and symbolic parameters. The unit of analysis in the **polyhedral model**.

**SDNode** — A node in LLVM's **SelectionDAG**. Has a list of input values (operands) and produces output values. Represents IR operations, memory chains, register copies, and target-specific operations.

**SelectionDAG** — LLVM's primary instruction selection framework. Builds a DAG from LLVM IR, legalizes types and operations, combines patterns, and selects machine instructions. See [Chapters 84–85](../chapters/part-14-backend/).

**Separation logic** — An extension of **Hoare logic** for reasoning about programs that manipulate heap memory. The separating conjunction `P * Q` asserts that P and Q hold on disjoint memory regions. Used in CompCert's memory model and Vellvm. See [Chapter 167](../chapters/part-24-verified-compilation/ch167-operational-semantics.md).

**SIMT** (Single Instruction Multiple Threads) — GPU execution model where many threads execute the same instruction stream. Threads are grouped into warps (NVIDIA) or wavefronts (AMD). Divergent branches cause serialization.

**Simulation** — A relation between transition systems showing that one simulates the behavior of another (in one direction). `Source ≼ Target`: every source step has a corresponding target step. Weaker than **bisimulation**. Used in compiler correctness proofs.

**SMT** (Satisfiability Modulo Theories) — A decision procedure extending SAT with theories (linear arithmetic, bitvectors, arrays). **Alive2** and translation validators use SMT (Z3) to verify IR transformations.

**SMT** (Simultaneous Multithreading) — Processor microarchitecture feature allowing multiple threads to share one physical core's execution units (Intel Hyper-Threading).

**SOS** (Structural Operational Semantics) — A style of defining programming language semantics via inference rules that describe how individual computation steps are taken. Used in **Vellvm** and **CompCert** formalization.

**SROA** (Scalar Replacement of Aggregates) — An LLVM pass that decomposes aggregate types (structs, arrays) accessed via `alloca` + GEP into scalar `alloca` objects, enabling **mem2reg** to convert them to SSA. See [Chapter 62](../chapters/part-10-analysis-middle-end/ch62-scalar-optimizations.md).

**SSA** (Static Single Assignment) — An IR property where each variable is defined exactly once. Control-flow merges use **phi** nodes. SSA enables efficient dataflow analysis. LLVM IR is always in SSA form. See [Chapter 21](../chapters/part-04-llvm-ir/ch21-ssa-dominance-and-loops.md).

**Subtarget** — LLVM's per-compilation-unit target configuration, capturing CPU, features, scheduling model, and ABI. Accessed via `TargetMachine::getSubtargetImpl()`. Derived from `TargetSubtargetInfo`.

---

## T

**TBI** (Top Byte Ignore) — AArch64 hardware feature that ignores the top 8 bits of a virtual address in hardware. Used by **HWASan** (memory safety) and pointer tagging. LLVM generates code respecting TBI when `-mno-implicit-float` is not set.

**TblGen** (TableGen) — LLVM's domain-specific language and tool for generating C++ code from declarative target descriptions. Files: `*.td`. Tools: `llvm-tblgen`, `clang-tblgen`, `mlir-tblgen`. See [Chapter 82 — TableGen Deep Dive](../chapters/part-14-backend/ch82-tablegen-deep-dive.md).

**ThinLTO** — A scalable **LTO** variant that avoids building a single merged module. Uses per-module summaries for cross-module analysis and performs parallel per-module codegen with thin link information. See [Chapter 77](../chapters/part-13-lto-whole-program/ch77-lto-and-thinlto.md).

**TLS** (Thread-Local Storage) — Per-thread variable storage. ELF TLS access models: local-exec, initial-exec, local-dynamic, global-dynamic. LLVM inserts appropriate TLS access sequences based on the model. See [Chapter 79](../chapters/part-13-lto-whole-program/ch79-linker-internals-got-plt-tls.md).

**Trait (MLIR)** — A compile-time property of an MLIR op that indicates structural or semantic invariants. Examples: `NoTerminator`, `IsTerminator`, `SameOperandsAndResultType`, `SingleBlock`, `RecursivelySpeculatable`. Enforced by the verifier. See [Chapter 133](../chapters/part-19-mlir-foundations/ch133-op-interfaces-and-traits.md).

**TTI** (Target Transform Info) — An LLVM analysis that provides target-specific cost models to middle-end passes: instruction costs, unrolling factors, vectorization widths, etc. Interface: `TargetTransformInfo`.

**Type constraint (MLIR)** — An ODS constraint on the type of an operand, result, or attribute. Examples: `AnyType`, `AnyInteger`, `F32`, `TensorOf<[F32, F64]>`. Checked by the op verifier.

**Type safety** — A language property guaranteeing that well-typed programs do not exhibit type errors at runtime. Proved as `progress + preservation` (or as soundness in a logical relation). Relevant to MLIR's typed ops.

---

## U

**Undef** — An LLVM IR special value indicating that any bit pattern is acceptable. Unlike **poison**, `undef` does not propagate and using it in control flow is defined (the choice is arbitrary but fixed). Increasingly deprecated in favor of `poison + freeze`.

---

## V

**Value (LLVM)** — The base class for all values in LLVM IR: `Instruction`, `Constant`, `GlobalVariable`, `Function`, `Argument`, etc. Every value has a `Type*` and a use-list (all `Use` edges pointing to this value).

**Value (MLIR)** — An SSA value produced by a block argument or op result. Has exactly one definition and zero or more uses. Typed; used as operands to other ops.

**VGPR/SGPR** — Vector/Scalar General-Purpose Registers on AMD GPU. VGPR: one lane per thread in a wavefront (e.g., 64 VGPRs × 64 threads). SGPR: single value per wavefront, used for uniform data. See [Chapter 103](../chapters/part-15-targets/ch103-amdgpu-and-rocm.md).

**Virtual register** — A register in LLVM's backend that has not yet been assigned to a physical register. Created by the `IRTranslator` / `SelectionDAG` and converted to physical registers by **register allocation**.

---

## W

**Wavefront** — AMD GPU execution unit: 64 threads (GCN) or 32 threads (RDNA) executing in lockstep. Analogous to a CUDA warp (32 threads). Divergent branches serialize execution.

**WGMMA** (Warpgroup Matrix Multiply-Accumulate) — NVIDIA Hopper (H100) instruction for 128-thread warpgroup-level matrix multiply. Accessed via `nvgpu.warpgroup.mma` in MLIR or `llvm.nvvm.wgmma.mma.async.*` intrinsics.

---

## Additional Abbreviations

| Abbreviation | Full Form |
|---|---|
| AAPCS | Procedure Call Standard for the Arm Architecture |
| ABI | Application Binary Interface |
| BOLT | Binary Optimization and Layout Tool |
| CU | Compile Unit (DWARF) / Compute Unit (GPU) |
| EXEC mask | Execution mask — per-thread enable bits in an AMD wavefront |
| HMAC | Hash-based Message Authentication Code (used in PAC) |
| ISA | Instruction Set Architecture |
| LSDA | Language-Specific Data Area |
| MC | Machine Code (LLVM MC layer) |
| ORC | On-Request Compilation (LLVM JIT framework) |
| PDL | Pattern Description Language (MLIR) |
| PDLL | PDL Language (higher-level MLIR pattern DSL) |
| PRE | Partial Redundancy Elimination |
| RISC-V | Reduced Instruction Set Computer V |
| ROCm | Radeon Open Compute (AMD GPU platform) |
| SBOM | Software Bill of Materials |
| TAPL | Types and Programming Languages (Pierce) |
| PFPL | Practical Foundations of Programming Languages (Harper) |
