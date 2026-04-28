# Chapter 16 — IR Structure

*Part IV — LLVM IR*

The previous three parts built the theoretical scaffolding: compiler front-end stages (Chapters 6–8), SSA construction and dataflow (Chapters 9–11), and the full arc of type theory culminating in Chapter 15's mapping of formal systems to LLVM IR and MLIR. Part IV begins the engineering complement to that theory. Chapters 16–27 cover LLVM IR from the ground up — its data model, type system, instruction set, metadata, and the tools built around it.

This chapter is the entry point. It establishes the four-level hierarchy that every LLVM IR object lives within, explains the `Value`/`User`/`Use` triad that underpins every SSA operation, surveys the textual `.ll` and binary `.bc` formats and how to move between them, and covers the module-level annotations — inline assembly and module flags — that LLVM uses to carry platform and ABI metadata. Every subsequent chapter in Part IV builds on the concepts introduced here.

If you have read Chapter 9's treatment of SSA theory, you already understand why every instruction defines exactly one name, why phi functions appear at join points, and why dominance is the key to reasoning about use-def chains. This chapter shows how that theory is materialized in C++ objects and in the textual grammar of LLVM IR.

---

## 1. The Module Hierarchy: Module → Function → BasicBlock → Instruction

LLVM IR is organized as a strict four-level containment hierarchy. Every object in a compiled program belongs to exactly one level, and each level is an iterable container for the level below it. The design is not accidental: the hierarchy maps cleanly onto the unit-of-compilation abstraction (module), the unit-of-linkage abstraction (function), the unit-of-control-flow abstraction (basic block), and the unit-of-computation abstraction (instruction). Understanding which level owns which state is the prerequisite for working with the C++ API.

### 1.1 `Module`: The Top-Level Container

A `Module` (declared in `llvm/IR/Module.h`) is the LLVM counterpart of a translation unit. It holds everything that a single compilation unit can contain: function definitions and declarations, global variables, aliases, indirect function stubs (ifuncs), named metadata nodes, and module-level inline assembly. It also carries two critical annotations: the target data layout string and the target triple.

**The data layout string** encodes the memory model: byte order (`e` for little-endian, `E` for big-endian), mangling convention, natural type alignment for every scalar type, pointer sizes and alignments for each address space, vector alignment, and the stack alignment. For x86-64 Linux under LLVM 22 the string is:

```
e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128
```

Reading this left to right: `e` (little-endian), `m:e` (ELF mangling), `p270:32:32` (pointer-size-address-space-270 is 32 bits with 32-bit alignment — the x32 ILP32 pointer), `p271:32:32` (non-integrable pointer in address space 271), `p272:64:64` (64-bit non-integrable pointer in address space 272), `i64:64` (64-bit integers are 64-bit aligned), `i128:128` (128-bit integers are 128-bit aligned), `f80:128` (x87 80-bit floats are stored 128-bit padded), `n8:16:32:64` (native integer widths supported by the target), `S128` (128-bit stack alignment). The backend and aliasing analyses both rely on this string for layout computations.

**The target triple** follows the `arch-vendor-os-environment` convention: `x86_64-pc-linux-gnu` for a typical x86-64 Linux build, `aarch64-apple-macosx14.0` for Apple Silicon macOS, `wasm32-unknown-emscripten` for WebAssembly via Emscripten. Clang deduces the triple from the host system unless `-target` is supplied.

The C++ API for the module-level fields:

```cpp
#include "llvm/IR/Module.h"

// Reading module identity
llvm::StringRef ModID     = M->getName();            // module identifier
std::string     SrcFile   = M->getSourceFileName();  // original source path
std::string     DLStr     = M->getDataLayoutStr();   // data layout string
const llvm::Triple &TT   = M->getTargetTriple();    // target triple

// Accessing module contents
for (llvm::Function &F : *M)           { /* all functions   */ }
for (llvm::GlobalVariable &GV : M->globals()) { /* globals */ }
for (llvm::GlobalAlias &GA : M->aliases())    { /* aliases */ }
for (llvm::GlobalIFunc &GI : M->ifuncs())     { /* ifuncs  */ }
for (llvm::NamedMDNode &NMD : M->named_metadata()) { /* named metadata */ }

// Serializing to textual IR
M->print(llvm::outs(), /*AAW=*/nullptr);
```

The `Module::print(raw_ostream&, AssemblyAnnotationWriter*)` method drives the `AssemblyWriter`, LLVM's internal IR printer. Passing a non-null `AssemblyAnnotationWriter` lets you inject per-instruction annotations into the output — useful for pass debugging tools that want to mark instructions with analysis results.

### 1.2 `Function`: A List of Basic Blocks

A `Function` (declared in `llvm/IR/Function.h`) contains the complete IR of one function: its type signature, attributes, linkage, alignment, personality function for exception handling, and — most importantly — an ordered list of `BasicBlock`s.

**Function type.** The first basic block in the list is the *entry block*; LLVM requires it to have no predecessors in the CFG. The `Function::getFunctionType()` method returns a `FunctionType*` encoding the return type and a list of parameter types. Parameters are accessed as `Function::arg_iterator` objects (also `Function::args()`), which are themselves `Argument` values with names and types.

**Attributes.** Functions carry two attribute lists: one for the function itself (`AttributeList::FunctionIndex`) and one per parameter. These encode ABI and optimization hints: `noinline`, `nounwind`, `noalias`, `readonly`, `mustprogress`, `uwtable`, `sanitize_address`, etc. Attributes are discussed in depth in Chapter 23.

**Linkage.** A function's linkage type governs visibility at link time: `external`, `internal`, `linkonce_odr`, `weak`, `private`, `available_externally`, and others. Linkage interacts with LTO (Chapter 77) and with the linker's dead-stripping rules.

**The entry block requirement.** The verifier enforces that the entry basic block — `Function::getEntryBlock()` — has no predecessors. A phi instruction at the top of the entry block is a verifier error. This invariant simplifies dominance computation and enables the simplest form of register allocation to proceed without special-casing the entry.

```cpp
#include "llvm/IR/Function.h"

llvm::FunctionType *FTy = F.getFunctionType();
llvm::Type         *RetTy = FTy->getReturnType();
unsigned            NumParams = FTy->getNumParams();

// Iterate arguments
for (llvm::Argument &Arg : F.args())
  llvm::outs() << "  arg " << Arg.getName() << ": " << *Arg.getType() << "\n";

// Access the entry block
llvm::BasicBlock &Entry = F.getEntryBlock();

// Count instructions across the function
unsigned Total = 0;
for (const llvm::BasicBlock &BB : F)
  Total += BB.size();
```

### 1.3 `BasicBlock`: A Linear Sequence with Exactly One Terminator

A `BasicBlock` (declared in `llvm/IR/BasicBlock.h`) is a maximal straight-line sequence of instructions ending in a *terminator*. LLVM's IR verifier enforces two structural invariants on every basic block: (1) the last instruction must be a terminator (`ret`, `br`, `switch`, `invoke`, `unreachable`, etc.); (2) no instruction other than the terminator may transfer control. A phi instruction, which is not a terminator, must appear before any non-phi instruction but after no non-phi instruction — that is, all phis in a block appear together at the top.

**Naming.** In the textual IR a block may be named explicitly (`entry:`, `loop_header:`, `bb.17:`) or assigned an implicit integer name by the slot numbering algorithm. The name is a local identifier prefixed with `%` when referenced as a branch target: `br label %entry`. The block's name can be queried with `BasicBlock::getName()`.

**CFG edges.** Predecessors and successors are derived from the block's terminator instruction via the `llvm::pred_begin`/`llvm::pred_end` and `llvm::succ_begin`/`llvm::succ_end` range functions (from `llvm/IR/CFG.h`). The `llvm::predecessors(BB)` and `llvm::successors(BB)` range wrappers are the idiomatic way to iterate them.

**The phi-use relationship.** A basic block appears as a *value* in phi instructions of its successors: `phi i32 [ %x, %entry ]`. This means the block itself is a `Value` subclass and has a use-list — every phi that names the block as an incoming predecessor contributes a `Use`. This is discussed in depth in Section 2. It matters for transformations that split edges or redirect branches: `replaceAllUsesWith()` on a `BasicBlock` value updates all phi references to it.

```cpp
#include "llvm/IR/BasicBlock.h"
#include "llvm/IR/CFG.h"

// Predecessors and successors
for (llvm::BasicBlock *Pred : llvm::predecessors(&BB))
  llvm::outs() << "  pred: " << Pred->getName() << "\n";
for (llvm::BasicBlock *Succ : llvm::successors(&BB))
  llvm::outs() << "  succ: " << Succ->getName() << "\n";

// Terminator instruction
llvm::Instruction *Term = BB.getTerminator();

// All phis at top of block
for (llvm::PHINode &PHI : BB.phis())
  llvm::outs() << "  phi: " << PHI << "\n";
```

### 1.4 `Instruction`: An SSA Value

An `Instruction` (declared in `llvm/IR/Instruction.h`) is the leaf of the hierarchy. Every instruction is simultaneously an SSA *definition* (a `Value` that may be used by later instructions) and an SSA *use-site* (a `User` that references zero or more prior values as operands). This dual identity is the fundamental data structure of LLVM IR, examined in Section 2.

Each instruction carries:

- An *opcode* (`Instruction::getOpcode()`), a small unsigned integer identifying the operation. Opcodes are defined in `llvm/IR/Instruction.def` and grouped into families: terminators, unary operators, binary operators, memory operations, casts, and so on. The grouping enables range-check `classof` as described in Chapter 4.
- Zero or more *operands* (`Instruction::getOperand(i)`, `Instruction::getNumOperands()`), each a pointer to a `Value`. Operands are stored in a compact, inline-allocated array managed by `User`.
- A *result type* (`Instruction::getType()`), which is `void` for instructions that do not produce a value (`ret`, `store`, `br`).
- A pointer to the *parent basic block* (`Instruction::getParent()`).
- Optional *metadata* attachments — debug location (`!dbg`), TBAA (`!tbaa`), loop membership (`!llvm.loop`), and so on. Covered in Chapter 22.

The three-level nested loop that traverses an entire module visits every instruction exactly once:

```cpp
#include "llvm/IR/Module.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/BasicBlock.h"
#include "llvm/IR/Instruction.h"

for (llvm::Function &F : *M) {
  for (llvm::BasicBlock &BB : F) {
    for (llvm::Instruction &I : BB) {
      llvm::outs() << I.getOpcodeName()
                   << "  result type: " << *I.getType() << "\n";
    }
  }
}
```

This pattern appears countless times in pass infrastructure. The iterators are bidirectional; `std::prev(BB.end())` gives the terminator without a separate call.

---

## 2. The Value/User/Use Triad

The three-way relationship between `Value`, `User`, and `Use` is the structural heart of LLVM IR. Every data-flow edge in the SSA graph is represented by an instance of `Use`. Understanding this triad is prerequisite to writing any non-trivial IR transformation.

### 2.1 Class Roles

**`Value`** (declared in `llvm/IR/Value.h`) is the base class of every entity that can appear on the right-hand side of an SSA instruction — instructions, constants, globals, function arguments, basic blocks, metadata nodes, and inline asm. The key field is a singly-linked intrusive list of `Use` objects: the *use-list*. Each node in this list represents one place where the value is consumed.

**`User`** (declared in `llvm/IR/User.h`) is the base class of every entity that *consumes* values: instructions, constant expressions, and globals. A `User` owns an array of `Use` objects — one per operand — each of which holds a pointer to the `Value` operand and a pointer back to the owning `User`. Note that `Instruction` inherits from both `Value` (it produces a value) and `User` (it consumes values).

**`Use`** (declared in `llvm/IR/Use.h`) is the edge in the use-def graph. It stores:
- A pointer to the `Value` being used.
- A pointer to the `User` that owns this `Use` (i.e., the instruction that consumes the value).
- A `Use*` pointer to the next `Use` in the value's use-list.

The resulting data structure is a set of linked lists — one per `Value` — threading through the `Use` objects owned by every `User` that references that value. Inserting or removing an operand updates the linked list in O(1) time.

### 2.2 The Use-List as an Intrusive Linked List

When an instruction `%sum = add i32 %a, %b` is created, the `add` instruction's `User` portion allocates two `Use` slots (one for `%a`, one for `%b`). Each `Use` is then spliced into the use-list of its corresponding `Value`:

```
%a (Value)  ──UseList──►  Use{val=%a, user=add, next=...}  ──► ...
%b (Value)  ──UseList──►  Use{val=%b, user=add, next=...}  ──► ...
add (Value) ──UseList──►  (initially empty; filled when add itself is used)
```

If `%sum` is subsequently used by `%doubled = mul i32 %sum, 2`, the `mul`'s first `Use` is spliced into `%sum`'s use-list:

```
%sum (Value) ──UseList──► Use{val=%sum, user=mul, next=...}
```

Iterating the use-list of a `Value` thus yields all instructions (and constant expressions) that reference it:

```cpp
// For every instruction I, find all users of its result
for (llvm::Use &U : I.uses()) {
  llvm::User *TheUser = U.getUser();
  llvm::outs() << "  used by: " << *TheUser << "\n";
}

// Equivalently, iterate over User* objects directly
for (llvm::User *U : I.users()) {
  if (auto *UserInst = llvm::dyn_cast<llvm::Instruction>(U))
    llvm::outs() << "  in block: " << UserInst->getParent()->getName() << "\n";
}
```

### 2.3 `replaceAllUsesWith`

The most important operation on the use-list is `Value::replaceAllUsesWith(Value *New)`. It walks the use-list of the current value and, for every `Use`, replaces the value pointer with `New` and re-splices the `Use` into `New`'s use-list. The result is that every consumer of the old value now consumes the new value — the entire use-list is transferred. After this call, the old value's use-list is empty.

```cpp
// Classic pattern: replace a computation with a simpler value
llvm::Value *Orig = someInstruction;
llvm::Value *SimplerVal = llvm::ConstantInt::get(Orig->getType(), 0);
Orig->replaceAllUsesWith(SimplerVal);
// Orig now has an empty use-list and can be erased
Orig->eraseFromParent();
```

`replaceAllUsesWith` handles the recursive case — if `New` has a use of `Orig` in its own definition, that use is also updated, which could create a cycle in constant expressions. The implementation is careful to handle this correctly. For basic blocks (which appear in phi instructions), a specialized variant `replaceSuccessorsPhiUsesWith` is available.

### 2.4 `Value` Subclass Hierarchy

The primary `Value` subclasses relevant to IR manipulation are:

| Class | What it represents |
|---|---|
| `Instruction` | An SSA instruction in a basic block |
| `Argument` | A formal parameter of a `Function` |
| `BasicBlock` | A labeled sequence of instructions (used by branch targets in phis) |
| `Constant` | A compile-time-known value: `ConstantInt`, `ConstantFP`, `ConstantExpr`, etc. |
| `GlobalValue` | A named entity at module scope (`GlobalVariable`, `Function`, `GlobalAlias`, `GlobalIFunc`) |
| `InlineAsm` | An inline assembly fragment (not module-level asm) |
| `MetadataAsValue` | A metadata operand threaded through the value system |

The class hierarchy uses LLVM's custom RTTI (Chapter 4): `isa<ConstantInt>(V)`, `dyn_cast<Instruction>(V)`, etc. Chapter 17 covers the constant value hierarchy in detail; Chapter 18 covers `GlobalValue`.

---

## 3. Textual `.ll` Format: Syntax and Round-Tripping

LLVM IR has a human-readable textual representation with the `.ll` file extension. It is the format you inspect when debugging a pass, the format Clang emits with `-emit-llvm -S`, and the format used in LIT regression tests throughout the LLVM tree. Understanding the syntax and the round-trip guarantee are both essential for effective development.

### 3.1 Syntax Conventions

A complete minimal `.ll` module:

```llvm
; ModuleID = 'example.ll'
source_filename = "example.c"
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"

define i32 @add(i32 %a, i32 %b) {
entry:
  %result = add i32 %a, %b
  ret i32 %result
}
```

The lexical conventions:

- **Comments** begin with `;` and run to end of line.
- **`@name`** sigil denotes global values: function names, global variable names, aliases. The name is the linkage name (mangled C++ name in practice).
- **`%name`** sigil denotes local SSA values: instruction results and function arguments. Names may be identifiers (`%result`, `%entry`) or decimal numbers (`%0`, `%1`) assigned by the slot allocator.
- **Types precede operands** everywhere. There is no type inference in the textual format; every value is spelled with its type at its definition point and every operand includes its type: `add i32 %a, %b`.
- **Labels** end with `:` and introduce basic blocks.
- **Metadata** nodes appear at the bottom of the file as numbered entries (`!0 = !{...}`) and are referenced inline (`!dbg !17`, `!tbaa !5`).
- **Attributes** may be grouped (`attributes #0 = { ... }`) and referenced by number (`define ... #0`).
- **Named metadata** appears as `!llvm.module.flags = !{!0, !1}`.

LLVM 22 uses *opaque pointers* exclusively. Since LLVM 17 the old typed-pointer syntax (`i8*`, `i32**`, `%struct.Foo*`) is gone. All pointers are spelled `ptr`, regardless of the pointee type. Code that worked in LLVM 14 with `i8*` operands must use `ptr` in LLVM 22.

### 3.2 Parsing `.ll` Files

Two primary parsing entry points exist in LLVM 22:

```cpp
#include "llvm/IRReader/IRReader.h"
#include "llvm/AsmParser/Parser.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/Support/SourceMgr.h"

llvm::LLVMContext Ctx;
llvm::SMDiagnostic Err;

// From a file (auto-detects .ll vs .bc from magic bytes)
std::unique_ptr<llvm::Module> M = llvm::parseIRFile("input.ll", Err, Ctx);
if (!M) {
  Err.print("my-tool", llvm::errs());
  return 1;
}

// From an in-memory string (useful in tests)
llvm::StringRef AsmText = R"ir(
  define i32 @id(i32 %x) { ret i32 %x }
)ir";
std::unique_ptr<llvm::Module> M2 =
    llvm::parseAssemblyString(AsmText, Err, Ctx);
```

`parseIRFile` dispatches on the file's magic bytes: if the first four bytes match the LLVM bitcode magic (`\xde\xc0\x17\x0b`), it calls the bitcode reader; otherwise it calls the assembly parser. The `SMDiagnostic` carries the filename, line number, column, and message of any parse error, suitable for user-facing diagnostics.

### 3.3 The Round-Trip Invariant

Serializing a module to textual IR and re-parsing it must produce a module semantically identical to the original. This round-trip invariant is the foundation of LLVM's IR regression test infrastructure: lit tests typically pass textual IR to a tool (opt, llc, clang), then check the output with `FileCheck`. The invariant guarantees that the form used in tests is always parseable and that text-format test files remain stable across LLVM tool updates.

Practically:

```bash
# Emit textual IR from a source file
clang -O2 -emit-llvm -S foo.c -o foo.ll

# Assemble back to bitcode
llvm-as foo.ll -o foo.bc

# Disassemble again and diff — must be whitespace-equivalent
llvm-dis foo.bc -o foo_rt.ll
diff foo.ll foo_rt.ll   # should differ only in whitespace/ordering of metadata
```

There is a subtle caveat: unnamed values (instructions and basic blocks assigned numeric names by the slot allocator) may be assigned different numbers after a round-trip if the slot numbering order changes. This is why IR tests use `; CHECK:` lines that match patterns rather than literal instruction names.

### 3.4 The IR Verifier

After parsing (or after any IR transformation), it is good practice to run the LLVM IR verifier:

```cpp
#include "llvm/IR/Verifier.h"

// Returns true if errors were found; writes diagnostics to OS
if (llvm::verifyModule(*M, &llvm::errs())) {
  llvm::errs() << "Module is broken!\n";
  return 1;
}

// Per-function variant (subset of module-level checks)
if (llvm::verifyFunction(F, &llvm::errs()))
  llvm::errs() << "Function " << F.getName() << " is broken!\n";
```

Note the convention: `verifyModule` returns `true` on *failure*, which is the opposite of most boolean functions in the LLVM API. The verifier checks a comprehensive set of structural invariants:

- **SSA dominance**: every use of an SSA value must be dominated by its definition (in the same function). Chapter 21 covers the dominance relation in depth.
- **Type consistency**: every instruction's operand types must match what the instruction expects. A `add i32` with an `i64` operand is a verifier error.
- **Terminator rules**: every basic block must end in exactly one terminator; no non-terminator may appear after a terminator.
- **Entry block**: the entry block may have no predecessors.
- **Phi placement**: phi instructions must appear at the top of their block, before any non-phi instruction.
- **Function attribute consistency**: mutually exclusive attributes (`readnone`/`writeonly`) must not both be present.
- **Metadata well-formedness**: `!dbg` attachments must reference valid `DILocation` nodes; TBAA nodes must form a valid tree.

Running the verifier after every pass during debugging (controlled by `-verify-each` in `opt`) catches IR corruption at the pass that introduces it rather than at a later crash.

---

## 4. Bitcode `.bc`: Format and Trade-offs

The binary LLVM bitcode format (`.bc`) is the second serialization form. While the textual format is optimized for human inspection and testing, bitcode is optimized for fast loading and compact on-disk representation. Both formats encode the same IR; which you use depends on the use case.

### 4.1 The Bitstream Container

Bitcode is built on a generic container format called *LLVM Bitstream*. The bitstream is a sequence of variable-width integers partitioned into *blocks* and *records*:

- A **block** demarcates a logical unit of data (analogous to an XML element). Blocks have a block ID and can be nested.
- A **record** is a sequence of integers within a block — the actual data.
- **Abbreviations** are user-defined encodings that replace common record patterns with shorter bit sequences, achieving compression without a general-purpose codec.

The LLVM IR bitcode file structure at the top level consists of:
1. A 32-bit magic number (`\xde\xc0\x17\x0b`) identifying the format.
2. A version and wrapper header (for wrapped bitcode, as produced by `clang -fembed-bitcode`).
3. The module block, which contains:
   - Type table block (all types used in the module).
   - Constants block (global constants).
   - Metadata block (all metadata nodes).
   - Function blocks (one per function definition, containing per-block instruction records).
   - Value symbol table block (name-to-value mappings).
   - Summary block (optionally, for ThinLTO; see Chapter 77).

### 4.2 The `use-list` Order Block

One subtlety of bitcode serialization: LLVM's `use-list` order — the order in which users appear in a value's use-list — is not determined by any semantic rule. It depends on the order instructions were created, which varies between runs. For reproducible builds (Chapter 174) LLVM emits a `use-list` order block in bitcode that explicitly records the permutation, allowing the use-list order to be reconstructed on load. The textual format has no such mechanism (use-list order in `.ll` is implicit); the `llvm-dis` `--show-annotations` flag can reveal the use-list permutation for debugging.

### 4.3 Performance Trade-offs

| Property | Textual `.ll` | Bitcode `.bc` |
|---|---|---|
| Human readable | Yes | No |
| Load speed | Slow (full parse) | 5–10× faster |
| File size | Large | ~40% smaller |
| Reproducibility | Stable | Stable (+ use-list block) |
| API stability | Stable | Stable (LLVM guarantees this; the C++ API does not) |
| LTO intermediate | No | Yes (standard format) |
| Debug inspection | Direct | Requires `llvm-dis` |

The bitcode format's API stability guarantee is notable: LLVM promises forward compatibility for bitcode produced by recent releases, meaning build caches, distributed compilation systems, and LTO artifacts remain loadable across minor LLVM version bumps. The C++ API has no such guarantee; it changes with every major release.

Bitcode is the format used internally by ThinLTO (Chapter 77): each translation unit produces a `.bc` file containing both the full IR and a compact per-function summary block. The linker reads only the summaries to build the global call graph, then lazily materializes only the functions it needs to inline across module boundaries.

---

## 5. `llvm-as`, `llvm-dis`, and `llvm-link`

Three command-line tools in the LLVM distribution deal directly with module serialization. They are essential for examining compiler output, writing regression tests, and diagnosing LTO issues.

### 5.1 `llvm-as`: Assembling `.ll` to `.bc`

`llvm-as` reads a textual `.ll` file and writes bitcode. It validates the assembly during parsing; a malformed module causes it to exit with a diagnostic. It does not run any optimization passes.

```bash
# Basic usage
llvm-as input.ll -o output.bc

# Validate but discard output (useful in CI to check IR well-formedness)
llvm-as input.ll -o /dev/null
```

`llvm-as` is the tool to reach for when you have hand-written IR that you want to compile with `llc` or feed to `opt`. It is also useful to verify that IR produced by a test generator is syntactically valid before the test reaches any pass infrastructure.

### 5.2 `llvm-dis`: Disassembling `.bc` to `.ll`

`llvm-dis` reads bitcode and writes textual IR. It is the workhorse for examining compiler output:

```bash
# Disassemble a library's LTO bitcode
llvm-dis libfoo.a.bc -o libfoo.ll

# Use --show-annotations to see use-list order and block metadata
llvm-dis --show-annotations input.bc -o annotated.ll

# Materialize-metadata: load without parsing full metadata, then dump
llvm-dis --materialize-metadata input.bc -o output.ll
```

The `--materialize-metadata` flag is useful for very large bitcode files where the metadata block dominates load time — it avoids fully materializing the potentially massive debug info tree until needed.

### 5.3 Practical Workflow: Examining Optimized IR

The most common workflow for examining what LLVM IR a build produces:

```bash
# Step 1: compile to textual IR with clang
clang -O2 -emit-llvm -S foo.c -o foo_O2.ll

# Step 2: or compile to bitcode, then disassemble (bitcode is smaller on disk)
clang -O2 -emit-llvm -c foo.c -o foo_O2.bc
llvm-dis foo_O2.bc -o foo_O2.ll

# Step 3: run a specific optimization and see the result
opt --passes="gvn,instcombine" -S foo_O2.ll -o foo_O2_opt.ll

# Step 4: compile to assembly for final confirmation
llc -O2 foo_O2.ll -o foo_O2.s
```

The `-S` flag to `clang` and `opt` produces textual IR rather than bitcode. The `-emit-llvm` flag to `clang` selects LLVM IR (rather than the default object or assembly output). These two flags compose: `-S -emit-llvm` gives textual IR from a C/C++ source file.

### 5.4 `llvm-link`: Linking Multiple Modules

`llvm-link` takes multiple bitcode or textual IR files and merges them into one module. It resolves undefined references across modules and applies the module-level merge semantics for global variables and module flags.

```bash
# Link two modules
llvm-link a.bc b.bc -o linked.bc

# Link textual IR modules
llvm-link a.ll b.ll -S -o linked.ll

# Override-on-conflict mode (useful for replacing function definitions)
llvm-link base.bc override.bc --override=override.bc -o result.bc
```

`llvm-link` is not a substitute for `lld` (Chapter 78) — it operates on LLVM IR, not object files. Its primary use is building a whole-program bitcode module for LTO analysis, or in tests that need to assemble a multi-file program's IR before feeding it to a tool.

When merging modules, `llvm-link` handles conflicts according to linkage types: two definitions with `linkonce_odr` linkage keep one; an external definition and a declaration merge by keeping the definition; conflicting strong definitions are a link-time error. Module flags are merged according to their declared behavior (Section 7).

---

## 6. Module-Level Inline Assembly

LLVM IR supports a form of raw assembly text that is emitted verbatim into the final object file, outside any function body. This is the *module-level inline assembly*, distinct from the per-instruction `asm ""` syntax covered in Chapter 25.

### 6.1 Syntax

In textual IR, module-level inline assembly is spelled with the `module asm` directive:

```llvm
module asm ".text"
module asm ".globl my_func"
module asm "my_func:"
module asm "  xor %eax, %eax"
module asm "  ret"
```

Multiple `module asm` strings are concatenated in order with newlines and placed at the beginning of the object file's text as a single blob of assembly. The assembler (LLVM's integrated assembler or an external `as`) processes this text after the compiler-generated code.

In C source code, the GCC extension `__asm__("...")` at file scope has the same effect; Clang translates it to module-level inline assembly in the IR.

### 6.2 C++ API

```cpp
#include "llvm/IR/Module.h"

// Read existing module asm
std::string ASM = M->getModuleInlineAsm();

// Replace entirely
M->setModuleInlineAsm(".section .note.GNU-stack,\"\",@progbits\n");

// Append (the more common operation)
M->appendModuleInlineAsm(".symver _ZN3foo3barEv, _ZN3foo3barEv@@MY_LIB_1.0\n");
```

### 6.3 Use Cases

**Symbol versioning** (`.symver`) is the canonical use case. Linux shared libraries use ELF symbol versioning to allow multiple versions of a symbol to coexist in one library. The `.symver` directive — not directly expressible in C — must be emitted as assembly text:

```llvm
; In versioned_lib.ll
module asm ".symver memcpy_impl, memcpy@@GLIBC_2.14"

define ptr @memcpy_impl(ptr %dst, ptr %src, i64 %n) {
entry:
  ; ... actual implementation ...
  ret ptr %dst
}
```

This tells the assembler that the `memcpy_impl` symbol should be published as `memcpy` at version `GLIBC_2.14` in the shared library's symbol table.

**GNU note sections.** The Linux kernel and some hardening tools require a `.note.GNU-stack` section with specific flags to communicate stack executability requirements:

```llvm
module asm ".section .note.GNU-stack,\22\22,@progbits"
```

Note the `\22` encoding: LLVM's textual IR string literals use hex escape sequences (`\XX`) for characters that would otherwise be syntactically ambiguous. `\22` is the ASCII code for a double-quote character, so `\22\22` produces the two-character `""` in the assembled output. Clang emits this directive automatically for C/C++ compilation units, but hand-written IR files typically omit it, which can cause linker warnings.

**CRT shims.** Embedded targets sometimes require CRT initialization functions (`_start`, `__libc_csu_init`) to be written in assembly. Wrapping them in module-level inline asm lets them be included in a bitcode compilation unit without a separate `.s` file.

**`.file` directives.** DWARF-based debuggers use `.file` directives to track source file paths. Some legacy toolchains require a specific `.file` to appear before any generated code; module asm can inject it.

Module-level inline asm is a sharp tool: it is completely opaque to LLVM's analysis and optimization passes. The optimizer cannot reason about its effects, cannot move it, and cannot eliminate it even if it appears dead. Code placed here bypasses aliasing analysis, escape analysis, and all other IR-level safety mechanisms. Use it only for assembler directives and low-level ABI shims that genuinely require it.

---

## 7. Module Flags

Module flags are a structured mechanism for attaching key-value metadata to a module — metadata that affects how modules are linked and how the backend generates code. Unlike function or instruction metadata, module flags are visible to the linker and may cause errors or behavioral changes when modules with conflicting flag values are combined.

### 7.1 The `!llvm.module.flags` Named Metadata Node

Module flags live in the `!llvm.module.flags` named metadata node. Each flag is a three-element tuple: `!{i32 behavior, !"key", value}`.

- `behavior` is an integer encoding how conflicts are resolved when two modules are linked (Table 7.1).
- `"key"` is a metadata string identifying the flag.
- `value` is the flag's value — an integer, a string, or a metadata tuple, depending on the flag.

**Table 7.1: Module flag behaviors**

| Behavior integer | Symbolic name | Semantics |
|---|---|---|
| 1 | `Error` | Linking two modules with different values for this key is a hard error |
| 2 | `Warning` | Linking two modules with different values emits a warning; the first value wins |
| 3 | `Require` | Requires another flag to have a specific value; used for dependency assertions |
| 4 | `Override` | The second module's value overrides the first (last-write-wins) |
| 5 | `Append` | Append the values from both modules into a metadata list |
| 6 | `AppendUnique` | Append only values not already present (set-union semantics) |
| 7 | `Max` | Take the maximum integer value of the two modules |
| 8 | `Min` | Take the minimum integer value of the two modules |

The `Max` (7) and `Min` (8) behaviors appear in LLVM 22 in addition to the six described in older documentation. They are used for `PIC Level`, `PIE Level`, and `uwtable` flags where larger values indicate stricter requirements and the stricter requirement should win.

### 7.2 A Complete Example

The following is the `!llvm.module.flags` block from a C++ file compiled with `clang++ -O1 -g` on x86-64 Linux with LLVM 22:

```llvm
!llvm.module.flags = !{!17, !18, !19, !20, !21, !22, !23}

!17 = !{i32 7, !"Dwarf Version", i32 5}
!18 = !{i32 2, !"Debug Info Version", i32 3}
!19 = !{i32 1, !"wchar_size", i32 4}
!20 = !{i32 8, !"PIC Level", i32 2}
!21 = !{i32 7, !"PIE Level", i32 2}
!22 = !{i32 7, !"uwtable", i32 2}
!23 = !{i32 7, !"debug-info-assignment-tracking", i1 true}
```

Reading these:
- `"Dwarf Version"` with behavior 7 (`Max`): DWARF v5; if another module requests DWARF v4, v5 wins.
- `"Debug Info Version"` with behavior 2 (`Warning`): LLVM's internal debug info version is 3; a mismatch produces a warning and strips debug info.
- `"wchar_size"` with behavior 1 (`Error`): `wchar_t` is 4 bytes; linking a module with `wchar_size=2` (Windows MSVC) is a hard error.
- `"PIC Level"` with behavior 8 (`Min`): minimum PIC level among linked modules (actually uses `Max` per symbol; the real behavior here encodes the minimum useful); value 2 means PIC.
- `"PIE Level"` with behavior 7 (`Max`): value 2 means full PIE.
- `"uwtable"` with behavior 7 (`Max`): value 2 means async unwind tables are required for all functions.
- `"debug-info-assignment-tracking"` with behavior 7 (`Max`): LLVM 16+ assignment-tracking debug info feature is active.

### 7.3 Common Module Flags

The full catalog of standard flags used in LLVM 22:

**ABI and compatibility flags:**
- `"wchar_size"` (Error): Size of `wchar_t` in bytes. Mismatch across objects from different compilers is a fatal link error.
- `"PIC Level"` (Min): 0 = not PIC, 1 = small PIC, 2 = large PIC.
- `"PIE Level"` (Max): 1 = PIE, 2 = PIE with full ASLR.
- `"Code Model"` (Error): LLVM code model — `tiny`, `small`, `kernel`, `medium`, `large`.

**Control-flow integrity flags:**
- `"cf-protection-return"` (Max): Enable Intel CET `ENDBR` return protection.
- `"cf-protection-branch"` (Max): Enable Intel CET `ENDBR` branch protection.

**Unwind table flags:**
- `"uwtable"` (Max): 1 = synchronous unwind tables; 2 = async unwind tables. Async unwind tables are required for `libunwind` and `libgcc_s` to correctly walk the stack during signal handlers and profiling. Emitted for all C++ compilations on most platforms.
- `"frame-pointer"` (Max): 0 = omit, 1 = non-leaf functions keep frame pointer, 2 = all functions keep frame pointer.

**Debug info flags:**
- `"Dwarf Version"` (Max): DWARF version (2, 4, or 5).
- `"Debug Info Version"` (Warning): LLVM's internal debug info schema version. Currently 3. Mismatch causes debug info to be stripped at link time.
- `"debug-info-assignment-tracking"` (Max): Assignment tracking for improved `-O` debug info quality (LLVM 16+).

**Platform-specific flags:**
- `"Linker Options"` (Append): On MachO, used to specify additional linker flags embedded in the object file (equivalent to `#pragma comment(lib, "...")` on Windows).
- `"SDK Version"` (Warning): macOS/iOS SDK version used to compile the object.
- `"SemanticInterposition"` (Min): Whether ELF semantic interposition is in effect for this module.

**Profile flags:**
- `"ProfileSummary"` (Error): Embedding a summary of the PGO profile data used to optimize the module, so the backend can make budget decisions for hot-path code.

### 7.4 The C++ API for Module Flags

```cpp
#include "llvm/IR/Module.h"

// Add a module flag
M->addModuleFlag(llvm::Module::Error, "wchar_size",
                 llvm::ConstantInt::get(llvm::Type::getInt32Ty(Ctx), 4));

// Query a flag
if (auto *WcharFlag = M->getModuleFlag("wchar_size")) {
  auto *CI = llvm::mdconst::dyn_extract<llvm::ConstantInt>(WcharFlag);
  if (CI)
    llvm::outs() << "wchar_size = " << CI->getZExtValue() << "\n";
}

// Enumerate all flags
llvm::SmallVector<llvm::Module::ModuleFlagEntry, 8> Flags;
M->getModuleFlagsMetadata(Flags);
for (const auto &Flag : Flags) {
  llvm::outs() << "Flag: " << Flag.Key->getString()
               << "  behavior: " << Flag.Behavior << "\n";
}
```

Module flags are one of the few places where LLVM's IR carries platform-specific, target-specific, and ABI-specific information in a structured, queryable form. A correctly constructed module flags block ensures that when a mixed-compiler build (e.g., Clang for C, GCC for Fortran, MSVC-compiled bitcode for a Windows DLL) reaches the linker, ABI incompatibilities are caught at compile time rather than manifesting as mysterious runtime failures.

---

## 8. Putting It Together: A Fully Annotated Module

The following is a complete, valid LLVM 22 `.ll` module that demonstrates all the constructs introduced in this chapter. It includes module-level inline assembly, a global variable, a function with multiple basic blocks, a phi node, and a selection of module flags.

```llvm
; ModuleID = 'full_example.ll'
source_filename = "full_example.c"
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"

; Module-level inline assembly: mark stack as non-executable
module asm ".section .note.GNU-stack,\22\22,@progbits"

; A global variable with internal linkage and an initializer
@counter = internal global i32 0, align 4

; A function declaration (no body — external definition)
declare i32 @external_fn(i32) #1

; A function definition with multiple basic blocks and a phi
define i32 @abs_val(i32 %x) #0 {
entry:
  %is_neg = icmp slt i32 %x, 0
  br i1 %is_neg, label %negate, label %done

negate:
  %negated = sub nsw i32 0, %x
  br label %done

done:
  %result = phi i32 [ %x, %entry ], [ %negated, %negate ]
  ret i32 %result
}

; Function attributes group
attributes #0 = { nounwind "frame-pointer"="none" "no-trapping-math"="true" }
attributes #1 = { "frame-pointer"="none" }

; Named metadata: module flags
!llvm.module.flags = !{!0, !1, !2, !3}

; ABI flag: wchar_t is 4 bytes; Error on mismatch
!0 = !{i32 1, !"wchar_size", i32 4}
; PIC level: Min behavior, level 2
!1 = !{i32 8, !"PIC Level", i32 2}
; PIE level: Max behavior, level 2
!2 = !{i32 7, !"PIE Level", i32 2}
; Async unwind tables required
!3 = !{i32 7, !"uwtable", i32 2}
```

Walking through this module: the `target datalayout` and `target triple` set the platform context. The `module asm` directive injects a note section. `@counter` is an internal global — private to this compilation unit — with zero initialization and 4-byte alignment. `@external_fn` is a declaration with no body, representing a function defined elsewhere. `@abs_val` contains three basic blocks: `entry` (the mandatory first block with no predecessors), `negate`, and `done`. The phi in `done` selects between the original `%x` (if the branch from `entry` was taken) and the negated value (if the branch from `negate` was taken). The module flags block anchors ABI compatibility.

---

## 9. Chapter Summary

- **The four-level hierarchy** — `Module` → `Function` → `BasicBlock` → `Instruction` — partitions every IR object into a strict containment structure. Each level owns the level below it and exposes STL-compatible iterators. The three-level nested loop over `Module`/`Function`/`BasicBlock` is the fundamental traversal pattern in LLVM pass infrastructure.

- **The `Value`/`User`/`Use` triad** implements the SSA def-use graph as an intrusive linked list. Each `Value` owns a use-list of `Use` objects; each `Use` connects a `Value` to the `User` (instruction or constant expression) that consumes it. `replaceAllUsesWith` transfers the entire use-list in O(n) time, enabling clean IR transformation without dangling pointers.

- **The textual `.ll` format** uses `@` for globals, `%` for locals, and explicit type annotations on every operand. It is the primary format for tests, debugging, and human inspection. LLVM 22 uses opaque `ptr` exclusively; typed pointers from LLVM 14 and earlier will not parse. `parseIRFile` and `parseAssemblyString` are the two parsing entry points.

- **Bitcode `.bc`** is a compact binary format built on the LLVM Bitstream container. It loads 5–10× faster than textual IR and occupies ~40% less disk space. It is the stable interchange format for LTO and the format whose forward compatibility LLVM guarantees. The `use-list` order block preserves non-deterministic ordering across serialization cycles.

- **`llvm-as`**, **`llvm-dis`**, and **`llvm-link`** are the three core IR manipulation tools. `llvm-as` validates and assembles `.ll` to `.bc`; `llvm-dis` inverts the process with `--show-annotations` and `--materialize-metadata` options for large files; `llvm-link` merges multiple modules by applying linkage and module-flag merge semantics. The combination of `clang -emit-llvm -S` and `opt -S` forms the standard optimization inspection workflow.

- **Module-level inline assembly** (`module asm "..."`) injects verbatim assembly text at the top of the generated object file. It is completely opaque to the optimizer. Its primary uses are ELF symbol versioning (`.symver`), stack-executability note sections, and CRT shims that must be expressed at the assembler level.

- **Module flags** (`!llvm.module.flags`) carry structured key-value metadata with explicit merge behaviors — `Error`, `Warning`, `Override`, `Append`, `AppendUnique`, `Max`, `Min`. They encode ABI constraints (`wchar_size`), control-flow protection levels (`cf-protection-return`, `cf-protection-branch`), debug info format (`Dwarf Version`, `Debug Info Version`), unwind table requirements (`uwtable`), and PIC/PIE posture. Correct module flags are what allow a mixed-compiler build to detect ABI mismatches before they become runtime bugs.

- **The LLVM IR verifier** (`llvm::verifyModule`) is the enforcement point for all structural invariants: SSA dominance, type consistency, terminator placement, phi ordering, entry-block restrictions, and metadata well-formedness. Run it after every IR-modifying transformation during development; the `-verify-each` flag to `opt` automates this.

---

*Chapter 17 examines LLVM IR's type system in full — integer types, floating-point types, the opaque pointer model, aggregate types, and the first-class vector types that ground scalable vector architectures like SVE and RVV. Chapter 21 returns to the SSA dominance relationship that the verifier enforces here, presenting the dominator tree, dominance frontiers, and their role in SSA construction and destruction.*
