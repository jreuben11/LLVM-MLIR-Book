# Chapter 82 — TableGen Deep Dive

*Part XIV — The Backend*

TableGen is LLVM's domain-specific language for describing hardware targets. Rather than writing thousands of lines of C++ that encode instruction formats, register classes, calling conventions, and instruction selection patterns, target authors write TableGen (`.td`) files that are processed by `llvm-tblgen` to generate the corresponding C++ code. The generated code is included into the backend via `#include "X86GenInstrInfo.inc"` and similar files. This chapter covers TableGen syntax from basics through advanced features (multiclass, foreach, defm, defvar, `!`-operators) and the specific TableGen backends that generate each piece of the LLVM backend infrastructure.

## 82.1 TableGen Language Fundamentals

### 82.1.1 Records

The fundamental unit in TableGen is a **record** — a named collection of fields with typed values:

```tablegen
// A simple record:
def MyReg {
  string Name = "rax";
  int HWEncoding = 0;
  bit IsCallerSaved = 1;
}
```

Records are accessed in generated C++ via the record name:
```cpp
const RecordKeeper &RK = /* from tblgen invocation */;
Record *MyReg = RK.getDef("MyReg");
int HWEnc = MyReg->getValueAsInt("HWEncoding");
```

### 82.1.2 Classes

A **class** is a record template with parameters:

```tablegen
class Register<string n, int enc> {
  string Name = n;
  int HWEncoding = enc;
  list<string> AltNames = [];
}

def RAX : Register<"rax", 0>;
def RCX : Register<"rcx", 1>;
def RDX : Register<"rdx", 2>;
```

Classes can inherit from other classes:

```tablegen
class GPR<string n, int enc> : Register<n, enc> {
  bit IsGPR = 1;
}

def RBX : GPR<"rbx", 3>;
```

### 82.1.3 `def`, `let`, and `defvar`

- **`def`**: defines a concrete record (no template parameters).
- **`let ... in`**: overrides field values in a scoped block.
- **`defvar`**: defines a variable that can be used in expressions.

```tablegen
defvar SmallInt = !cast<int>("8");  // integer variable

let isTerminator = 1, isReturn = 1 in {
  def RET : Instruction;     // isTerminator and isReturn are set for all defs here
  def IRET : Instruction;
}
```

### 82.1.4 `!`-Operators

TableGen's expression language uses `!`-prefixed operators:

| Operator | Type | Meaning |
|----------|------|---------|
| `!add(a, b)` | int → int | Integer addition |
| `!sub(a, b)` | int → int | Integer subtraction |
| `!and(a, b)` | int → int | Bitwise AND |
| `!or(a, b)` | int → int | Bitwise OR |
| `!xor(a, b)` | int → int | Bitwise XOR |
| `!shl(a, b)` | int → int | Left shift |
| `!eq(a, b)` | any → bit | Equality test |
| `!ne(a, b)` | any → bit | Inequality test |
| `!lt(a, b)` | int → bit | Less than |
| `!if(c, t, f)` | bit, T, T → T | Conditional |
| `!listconcat(a, b)` | list, list → list | Concatenate lists |
| `!size(list)` | list → int | List length |
| `!foreach(var, list, expr)` | list → list | Map over list |
| `!foldl(init, list, acc, var, expr)` | T, list → T | Fold over list |
| `!cast<T>(x)` | any → T | Type cast |
| `!strconcat(a, b)` | string → string | Concatenate strings |
| `!subst(from, to, val)` | string → string | String substitution |

```tablegen
defvar Width = 32;
defvar TypeName = !strconcat("i", !cast<string>(Width));  // "i32"
def MyField { int Bits = !add(Width, 8); }                // Bits = 40
```

## 82.2 Multiclass and defm

### 82.2.1 The Problem: Code Duplication

A common pattern in target descriptions: the same instruction type (add, sub, mul, etc.) exists for multiple sizes (8, 16, 32, 64 bit) or variants (register-register, register-immediate). Without a way to factor out common structure, this leads to massive duplication.

### 82.2.2 multiclass

A **multiclass** defines a group of related `def`s simultaneously:

```tablegen
multiclass ALU_rr<string Mnemonic, SDNode OpNode> {
  def 8rr  : Instruction<Mnemonic # "8",  OpNode, i8,  GR8>;
  def 16rr : Instruction<Mnemonic # "16", OpNode, i16, GR16>;
  def 32rr : Instruction<Mnemonic # "32", OpNode, i32, GR32>;
  def 64rr : Instruction<Mnemonic # "64", OpNode, i64, GR64>;
}

defm ADD : ALU_rr<"add", add>;   // creates ADD8rr, ADD16rr, ADD32rr, ADD64rr
defm SUB : ALU_rr<"sub", sub>;   // creates SUB8rr, SUB16rr, SUB32rr, SUB64rr
```

The `#` operator concatenates strings in multiclass definitions; the prefix from `defm` is prepended to each def name.

### 82.2.3 defm Inheritance

A `defm` can specify multiple multiclasses:

```tablegen
multiclass ALU_ri<string Mnemonic, SDNode OpNode> {
  def 32ri : Instruction<Mnemonic # "32i", OpNode, i32, i32imm>;
  def 64ri : Instruction<Mnemonic # "64i", OpNode, i64, i64imm>;
}

defm ADD : ALU_rr<"add", add>, ALU_ri<"add", add>;
// Creates: ADD8rr, ADD16rr, ADD32rr, ADD64rr, ADD32ri, ADD64ri
```

### 82.2.4 foreach

`foreach` iterates over a list, creating defs or other constructs:

```tablegen
foreach Sz = [8, 16, 32, 64] in {
  def MOVr # Sz : Instruction<"mov", Sz>;
}
// Creates: MOVr8, MOVr16, MOVr32, MOVr64
```

### 82.2.5 The Record Graph

TableGen builds a **record graph** — a bipartite graph of Record nodes and Class nodes. Each record has class edges (the classes it instantiates) and field edges (values of its fields, which may be references to other records). The `-gen-*` backends traverse this graph to generate C++.

```bash
# Dump all records (for debugging):
llvm-tblgen -dump-json X86.td -I include/ > all_records.json

# Print a specific record:
llvm-tblgen -print-records X86.td -I include/ 2>&1 | grep -A 20 "def ADD64rr"
```

## 82.3 TableGen for Register Information (-gen-register-info)

The `-gen-register-info` backend generates `X86GenRegisterInfo.inc`, which defines:

- `X86::RAX`, `X86::RCX`, … register enum values.
- `X86RegisterInfo::DwarfFlavour` — DWARF register numbers.
- Register class tables (membership, allocation order, spill sizes).
- Sub-register and super-register relationships.

```tablegen
// X86 register definition (simplified):
class X86Reg<string n, bits<16> enc, list<Register> subregs = []>
    : Register<n> {
  let HWEncoding = enc;
  let SubRegs = subregs;
}

def AL  : X86Reg<"al",   0>;
def AH  : X86Reg<"ah",   4>;
def AX  : X86Reg<"ax",   0, [AL, AH]>;
def EAX : X86Reg<"eax",  0, [AX]>;
def RAX : X86Reg<"rax",  0, [EAX]>;

// Register class:
def GR64 : RegisterClass<"X86", [i64], 64, (add RAX, RCX, RDX, ...)> {
  let AllocationPriority = 1;
}
```

The `SubRegs` field defines the sub-register hierarchy: `RAX` contains `EAX` (low 32 bits), which contains `AX` (low 16 bits), which contains `AL` (low 8) and `AH` (bits 8-15). The register allocator uses this hierarchy to track which physical registers are partially clobbered.

## 82.4 TableGen for Instruction Info (-gen-instr-info)

The `-gen-instr-info` backend generates `X86GenInstrInfo.inc`:

```tablegen
// Instruction definition (simplified):
class Instruction {
  string AsmString;        // assembly mnemonic
  dag OutOperandList;      // output operands
  dag InOperandList;       // input operands
  list<dag> Pattern;       // isel patterns

  // Instruction properties:
  bit isTerminator = 0;
  bit isReturn = 0;
  bit isCall = 0;
  bit isBarrier = 0;
  bit hasDelaySlot = 0;
  bit mayLoad = 0;
  bit mayStore = 0;
  bit hasSideEffects = 0;
  bits<8> TSFlags = 0;     // target-specific flags
}

def ADD64rr : Instruction {
  let AsmString = "add\t{$src2, $dst|$dst, $src2}";
  let OutOperandList = (outs GR64:$dst);
  let InOperandList  = (ins GR64:$src1, GR64:$src2);
  let Pattern = [(set GR64:$dst, (add GR64:$src1, GR64:$src2))];
  let isCommutable = 1;
  let Constraints = "$src1 = $dst";   // two-address: dst aliases src1
}
```

The `Pattern` field is a DAG pattern that the instruction selection engine matches against the SelectionDAG. `(set GR64:$dst, (add GR64:$src1, GR64:$src2))` means "this instruction implements a 64-bit integer add of two GR64 registers."

## 82.5 TableGen for DAG Instruction Selection (-gen-dag-isel)

The `-gen-dag-isel` backend generates the `SelectCode` function in `X86GenDAGISel.inc`. This function is a giant switch statement that pattern-matches SelectionDAG nodes:

```tablegen
// Pattern matching:
def : Pat<(add GR32:$src, imm:$imm),
          (ADD32ri GR32:$src, imm:$imm)>;

// Pattern with predicates:
def : Pat<(X86cmov GR64:$t, GR64:$f, (X86cmp (setcc_carry GR64:$src, 0)), 4),
          (CMOVB64rr GR64:$t, GR64:$f)>;

// Complex pattern:
def addr : ComplexPattern<iPTR, 5, "SelectAddr", [frameindex], [SDNPWantParent]>;
def : Pat<(load addr:$src), (MOV64rm addr:$src)>;
```

The generated `SelectCode` function walks the SDNode tree and matches against these patterns using a decision tree generated by `llvm-tblgen`.

## 82.6 TableGen for GlobalISel (-gen-global-isel)

The `-gen-global-isel` backend generates combiners and instruction selectors for GlobalISel:

```tablegen
// GlobalISel combiner rule:
def AddOfVecSplat : GICombineRule<
  (defs root:$root, build_vector_all_same:$bvec),
  (match (wip_match_opcode G_ADD):$root,
         [{ return Helper.matchCombineAddP(...); }]),
  (apply [{ Helper.applyCombineAddP(...); }])>;

// GlobalISel instruction selection pattern:
def : GINodeEquiv<G_ADD, add>;  // map generic G_ADD to ISD::add for pattern matching
```

The GlobalISel selection infrastructure uses a separate pattern-matching engine (`InstructionSelector`) generated by `-gen-global-isel`, which is simpler and more regular than the SelectionDAG engine.

## 82.7 TableGen for Assembly (-gen-asm-matcher, -gen-asm-writer)

### 82.7.1 Assembly Writing (-gen-asm-writer)

The `-gen-asm-writer` backend generates `printInstruction()` for the `AsmPrinter`:

```tablegen
def ADD64rr {
  let AsmString = "addq\t{$src, $dst|$dst, $src}";
  // AT&T syntax: "addq %rcx, %rax"
  // Intel syntax: "add rax, rcx"
}
```

The `AsmString` uses `$operand_name` for operand substitution. The `{att|intel}` syntax specifies AT&T vs Intel syntax variants.

### 82.7.2 Assembly Matching (-gen-asm-matcher)

The `-gen-asm-matcher` backend generates `MatchInstructionImpl()` for the `MCAsmParser`:

```tablegen
// Mnemonic alias:
def : MnemonicAlias<"sal", "shl">;  // "sal" is an alias for "shl"

// Operand type:
class Operand<ValueType ty> {
  string PrintMethod = "printOperand";
  string ParserMatchClass = "AnyMemOperand";
}
```

The generated matcher converts assembly text → `MCInst` objects, used by the integrated assembler and `.s` file parsing.

## 82.8 TableGen for Calling Conventions (-gen-callingconv)

```tablegen
// Calling convention (x86-64 System V):
def CC_X86_64_C : CallingConv<[
  // Pass first 6 integer args in registers:
  CCIfType<[i32], CCAssignToReg<[EDI, ESI, EDX, ECX, R8D, R9D]>>,
  CCIfType<[i64], CCAssignToReg<[RDI, RSI, RDX, RCX, R8, R9]>>,
  // Pass first 8 FP args in XMM registers:
  CCIfType<[f32, f64], CCAssignToReg<[XMM0, XMM1, ..., XMM7]>>,
  // Remaining args go on the stack:
  CCIfType<[i32, i64], CCAssignToStack<4, 4>>,
  CCIfType<[f32, f64], CCAssignToStack<8, 8>>
]>;
```

The `-gen-callingconv` backend generates `CC_X86_64_C()` — a function that, given a list of argument types, determines where each argument is placed (register or stack offset).

## 82.9 TableGen for Subtarget (-gen-subtarget)

The `-gen-subtarget` backend generates feature flag parsing and CPU model definitions:

```tablegen
def FeatureAVX2 : SubtargetFeature<"avx2", "HasAVX2", "true",
    "Enable AVX2 instructions", [FeatureAVX]>;

def FeatureAVX512F : SubtargetFeature<"avx512f", "HasAVX512F", "true",
    "Enable AVX-512 instructions", [FeatureAVX2]>;

def : ProcessorModel<"skylake", SkylakeClientModel,
    [FeatureAVX2, FeatureBMI, FeatureBMI2, FeatureAES, ...]>;
```

This generates `X86::FeatureAVX2`, `X86Subtarget::HasAVX2`, and the `ParseSubtargetFeatures` function that parses `-mattr=+avx2` command-line flags.

## 82.10 TableGen for Scheduling Models (-gen-sched-models)

The `-gen-sched-models` backend generates the machine scheduling model — instruction latencies, resource usage, and throughput:

```tablegen
// Skylake scheduling model for integer adds:
def SKLWriteResGroup1 : SchedWriteRes<[SKLPort0156]> {
  let Latency = 1;
  let NumMicroOps = 1;
}

def : InstRW<[SKLWriteResGroup1], (instregex "ADD(8|16|32|64)rr")>;
```

This generates the `SchedModel` tables that `MachineScheduler` uses to determine instruction issue latencies and resource conflicts.

---

## Chapter Summary

- TableGen is a declarative language for hardware descriptions; **records** (named field collections) and **classes** (parameterized templates) are the fundamental building blocks.
- **`!`-operators** provide a rich expression language: arithmetic, list manipulation, conditionals, string operations, and type casts — all evaluated at TableGen processing time.
- **`multiclass` + `defm`** factor out repeated instruction families (e.g., ADD8, ADD16, ADD32, ADD64) into a single parameterized definition; **`foreach`** iterates a list to generate def groups.
- TableGen backends (`-gen-*` flags to `llvm-tblgen`) translate the record graph to C++ include files:
  - `-gen-register-info`: register enums, classes, sub-register hierarchies.
  - `-gen-instr-info`: instruction tables with properties and patterns.
  - `-gen-dag-isel`: `SelectCode()` — the SelectionDAG pattern-matching function.
  - `-gen-global-isel`: GlobalISel combiners and instruction selector tables.
  - `-gen-asm-writer`/`-gen-asm-matcher`: assembly printing and parsing.
  - `-gen-callingconv`: calling convention assignment functions.
  - `-gen-subtarget`: feature flags, CPU models, `ParseSubtargetFeatures`.
  - `-gen-sched-models`: instruction latency and throughput tables.
