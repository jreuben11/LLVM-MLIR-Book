# Chapter 130 — MLIR IR Structure

*Part XIX — MLIR Foundations*

Understanding MLIR's IR structure is prerequisite to everything else in this part of the book. Unlike LLVM IR, which has a fixed three-level hierarchy (Module → Function → BasicBlock), MLIR's structure is recursive: operations contain regions, regions contain blocks, and blocks contain operations. This generality is not accidental — it is what allows a single IR representation to express function bodies, loop nests, GPU kernels, hardware module hierarchies, and dataflow graphs using the same building blocks. This chapter dissects each layer of the MLIR hierarchy, from the highest-level `mlir::ModuleOp` down to individual `Value`s and types, with a focus on the C++ API and the textual syntax that appears in `.mlir` files.

---

## 130.1 The Region-Block-Op Hierarchy

### Three-Level Nesting

MLIR's fundamental structure is:

```
Operation
  └─ Region (zero or more)
       └─ Block (one or more)
            └─ Operation
                 └─ Region
                      └─ ...
```

This differs fundamentally from LLVM IR:

| Aspect | LLVM IR | MLIR |
|--------|---------|------|
| Top level | Module (flat list of functions) | `ModuleOp` (a regular op with a body region) |
| Function | `Function` (special global object) | `func.func` (a regular op with one region) |
| Basic block | `BasicBlock` (list of instructions) | `Block` (list of ops) |
| Instruction | `Instruction` (typed, fixed opcode) | `Operation` (typed, dialect-defined) |
| Nesting | Functions do not nest | Ops can nest to arbitrary depth |

### ModuleOp

Every MLIR program is wrapped in a `builtin.module`:

```mlir
module {
  func.func @foo() {
    ...
  }
  // Other top-level ops: globals, type aliases, ...
}
```

`ModuleOp` is a regular operation defined in the builtin dialect. It has one region (the module body) which is a **graph region** — ops inside are not in SSA dominance order; they can reference each other out of order (e.g., function calls to functions defined later).

### func.func and SSA Regions

`func.func` has one region with multiple blocks in SSA form:

```mlir
func.func @example(%arg0: i32, %arg1: i32) -> i32 {
^entry(%a: i32, %b: i32):    // block arguments
  %sum = arith.addi %a, %b : i32
  cf.cond_br %cond, ^then(%sum : i32), ^else(%b : i32)
^then(%x: i32):
  return %x : i32
^else(%y: i32):
  return %y : i32
}
```

The function's entry block arguments correspond to the function parameters. The region is an **SSA region**: values must dominate their uses, and blocks are reached via branch terminators.

### scf.for — Nested Region Example

`scf.for` is a structured loop op with one region:

```mlir
%result = scf.for %i = %lb to %ub step %step
    iter_args(%acc = %init) -> (f64) {
  %elem = memref.load %arr[%i] : memref<?xf64>
  %new_acc = arith.addf %acc, %elem : f64
  scf.yield %new_acc : f64
}
```

The loop body region has one block with two block arguments: the induction variable `%i` and the loop-carried variable `%acc`. `scf.yield` is the region terminator.

---

## 130.2 Operations

### The Operation Class

Every MLIR construct is an `mlir::Operation` ([`mlir/include/mlir/IR/Operation.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/Operation.h)). An `Operation` is a variable-size object containing:

```cpp
class Operation final : public llvm::ilist_node_with_parent<Operation, Block> {
public:
  // Identity:
  StringAttr getName() const;          // e.g., "arith.addi"
  Dialect *getDialect() const;

  // Operands (inputs):
  unsigned getNumOperands() const;
  Value getOperand(unsigned idx) const;
  MutableOperandRange getOperands();

  // Results (outputs):
  unsigned getNumResults() const;
  OpResult getResult(unsigned idx) const;

  // Attributes (compile-time constants):
  DictionaryAttr getAttrDictionary() const;
  Attribute getAttr(StringAttr name) const;
  void setAttr(StringAttr name, Attribute value);

  // Regions (nested structure):
  unsigned getNumRegions() const;
  Region &getRegion(unsigned idx);

  // Successors (for control flow ops with multiple targets):
  unsigned getNumSuccessors() const;
  Block *getSuccessor(unsigned idx) const;

  // Location (source provenance):
  Location getLoc() const;
  void setLoc(Location loc);

  // Parent structure:
  Block *getBlock() const;
  Region *getParentRegion() const;
  Operation *getParentOp() const;
};
```

### OpState: The User-Facing Wrapper

Direct use of `Operation *` is the low-level interface. Most code works with **concrete op types** generated from ODS (see [Chapter 132](ch132-defining-dialects-with-ods.md)). These are thin wrappers that hold an `Operation *` and provide named accessor methods:

```cpp
// Using the concrete type:
auto addOp = dyn_cast<arith::AddIOp>(op);
Value lhs = addOp.getLhs();       // operand 0
Value rhs = addOp.getRhs();       // operand 1
Value result = addOp.getResult(); // result 0

// Equivalent low-level:
Value lhs = op->getOperand(0);
Value result = op->getResult(0);
```

`OpState` is the base class for all ODS-generated op wrappers. It is a value type (holds just an `Operation *`) so it can be freely passed by value.

### Op Name Anatomy

```mlir
%result = arith.addi %lhs, %rhs : i32
```

- `%result`: the SSA result name (local to the enclosing block's scope)
- `arith`: dialect name
- `addi`: op name within the dialect
- `%lhs`, `%rhs`: operand SSA values
- `i32`: type of operands and result (here specified once because they are all the same)

Multi-result ops use:
```mlir
%q, %r = arith.divui %a, %b : i32
```

Ops with no results omit the `%name =` prefix:
```mlir
memref.store %val, %arr[%i] : memref<?xf32>, index
```

---

## 130.3 Values and Types

### The Value Class

An SSA value is represented by `mlir::Value` ([`mlir/include/mlir/IR/Value.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/Value.h)). It is a pointer-like wrapper that refers to either:

- **`OpResult`**: the result of an `Operation` (operation defines the value)
- **`BlockArgument`**: a block argument (the value is defined at block entry)

```cpp
Value val = ...;

// Query what defines this value:
if (auto opResult = dyn_cast<OpResult>(val)) {
  Operation *defOp = opResult.getOwner();
  unsigned resultIdx = opResult.getResultNumber();
}
if (auto blockArg = dyn_cast<BlockArgument>(val)) {
  Block *parentBlock = blockArg.getOwner();
  unsigned argIdx = blockArg.getArgNumber();
}

// Get the type:
Type ty = val.getType();

// Iterate uses:
for (OpOperand &use : val.getUses()) {
  Operation *user = use.getOwner();
  unsigned operandIdx = use.getOperandNumber();
}
```

### Types

All MLIR types inherit from `mlir::Type` ([`mlir/include/mlir/IR/Types.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/Types.h)). `Type` is a lightweight wrapper (pointer-sized) around a uniqued `TypeStorage *`. Two values with the same logical type share the same `TypeStorage *`, enabling `O(1)` type equality checks.

**Builtin integer types**:
```mlir
i1    // 1-bit integer (boolean)
i8    // 8-bit integer
i32   // 32-bit integer
i64   // 64-bit integer
i1024 // arbitrary-width supported
// Note: MLIR integers are signless by default; signedness is in op semantics
```

**Builtin float types**:
```mlir
f16     // IEEE 16-bit float
bf16    // bfloat16 (Google Brain float)
f32     // IEEE 32-bit float
f64     // IEEE 64-bit float
f80     // x87 80-bit extended precision
f128    // IEEE 128-bit float
tf32    // TensorFloat32 (NVIDIA A100)
```

**Index type**:
```mlir
index  // machine-word-sized integer for array indices
```

**Container types**:
```mlir
memref<4x4xf32>           // static-shape memory reference
memref<?x?xf64>           // dynamic-shape memory reference
memref<4xf32, strided<[1], offset: ?>>  // strided layout
tensor<4x4xf32>           // static-shape immutable tensor value
tensor<?xi64>             // dynamic-shape tensor
vector<4xf32>             // fixed-length SIMD vector
vector<[4]xf32>           // scalable vector (for SVE/RISC-V V)
```

**Function type**:
```mlir
(i32, f64) -> (i32, f64)
```

**None type** (unit/void):
```mlir
none
```

### TypeRange and ValueRange

`TypeRange` and `ValueRange` are lightweight non-owning spans over sequences of types/values, analogous to `ArrayRef<T>` in LLVM:

```cpp
// Constructing:
ValueRange operands = op->getOperands();          // all operands
TypeRange resultTypes = op->getResultTypes();     // all result types

// Usage in builders:
auto newOp = builder.create<SomeOp>(loc,
    /*resultTypes=*/TypeRange{i32Ty, f64Ty},
    /*operands=*/ValueRange{lhs, rhs});
```

---

## 130.4 Blocks and Control Flow

### Block Structure

A `Block` ([`mlir/include/mlir/IR/Block.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/Block.h)) is:

```cpp
class Block : public llvm::ilist_node_with_parent<Block, Region> {
public:
  // Block arguments (SSA values defined at entry):
  BlockArgListType getArguments();
  BlockArgument addArgument(Type type, Location loc);
  unsigned getNumArguments() const;

  // Operations:
  OpListType &getOperations();
  Operation &front();   // first op in block
  Operation &back();    // last op (usually terminator)
  bool empty() const;

  // Terminator:
  Operation *getTerminator();  // last op if it's a terminator

  // CFG successors:
  SuccessorRange getSuccessors();  // blocks this block jumps to
  PredecessorRange getPredecessors();  // blocks that jump to this

  // Parent:
  Region *getParent() const;
};
```

### Block Arguments vs PHI Nodes

LLVM IR uses **PHI nodes** to merge values at control flow join points. MLIR uses **block arguments** instead. The two representations are equivalent; block arguments are considered more readable and easier to transform.

```llvm
; LLVM IR PHI:
%result = phi i32 [ %x, %then_bb ], [ %y, %else_bb ]
```

```mlir
// MLIR block argument:
^merge(%result: i32):
  // %result is the block argument; set by branch ops
```

The branches that reach `^merge` pass the value as a branch argument:
```mlir
cf.br ^merge(%x : i32)    // from then block
cf.br ^merge(%y : i32)    // from else block
```

### Terminator Operations

Every non-empty block must end with a **terminator** — an op with the `Terminator` trait. Common terminators:

```mlir
return %val : i32                       // func.return
cf.br ^target_block(%arg : i32)         // unconditional branch
cf.cond_br %cond, ^true_bb, ^false_bb   // conditional branch
scf.yield %val : f64                    // structured loop yield
```

### CFG Operations

The `cf` dialect provides unstructured control flow for when structured ops are insufficient:

```mlir
// Conditional branch with block arguments:
cf.cond_br %cond, ^bb1(%a : i32), ^bb2(%b : i32, %c : f64)

// Switch (multi-way branch):
cf.switch %case : i32,
  default: ^default_bb,
  0: ^zero_bb,
  1: ^one_bb(%arg : i32)
```

---

## 130.5 Regions and Structural Constraints

### Region Class

A `Region` ([`mlir/include/mlir/IR/Region.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/Region.h)) is an ordered list of blocks:

```cpp
class Region {
public:
  BlockListType &getBlocks();
  Block &front();   // entry block
  Block &back();    // last block
  bool empty() const;
  unsigned getNumArguments() const;  // entry block argument count
  BlockArgListType getArguments();   // entry block arguments

  // Parent op:
  Operation *getParentOp() const;
};
```

### SSA Regions vs Graph Regions

MLIR distinguishes two region kinds:

**SSA region** (most common): blocks are ordered; each value dominates its uses; control flow flows from entry block. Used for function bodies, loop bodies, if/then/else branches.

**Graph region**: no dominance requirement; ops form a graph (think dataflow). Used at module level (functions can reference each other), in `func.graph_func`, and in some hardware dialects (CIRCT).

The `RegionKindInterface` lets ops declare which kind of region they contain:
```cpp
RegionKind getRegionKind(unsigned index) {
  return RegionKind::Graph;  // or RegionKind::SSACFG
}
```

### IsolatedFromAbove

An operation with the `IsolatedFromAbove` trait ([`mlir/include/mlir/IR/OpDefinition.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/OpDefinition.h)) declares that its regions cannot use SSA values defined outside those regions. `func.func` is the canonical example: a Fortran function cannot directly access variables from an enclosing function in the SSA sense (it can only see them through explicit arguments or globals).

```mlir
func.func @outer() {
  %x = arith.constant 42 : i32
  func.func @inner() {   // ERROR if inner tried to use %x directly
    // %x is not visible here — func.func is IsolatedFromAbove
  }
  return
}
```

`scf.for`, `scf.if` are NOT isolated-from-above; their bodies capture values from the enclosing function:
```mlir
func.func @f(%arr: memref<10xf32>, %n: index) -> f32 {
  %sum = scf.for %i = %c0 to %n step %c1 iter_args(%acc = %zero) -> f32 {
    %v = memref.load %arr[%i] : memref<10xf32>  // %arr captured from outer scope
    %new = arith.addf %acc, %v : f32
    scf.yield %new : f32
  }
  return %sum : f32
}
```

---

## 130.6 Attributes

### Attribute Basics

Attributes are **compile-time constant data** attached to operations or types. Like types, they are uniqued and immutable:

```cpp
class Attribute {
  // Wrapper around AttributeStorage *
  // Equality: same AttributeStorage * → same attribute
};
```

Attributes appear in the textual IR in braces after the op name:
```mlir
// Op with attributes:
%c42 = arith.constant 42 : i32  // the '42' is an IntegerAttr

// Explicit attribute dictionary:
func.func @foo() attributes {sym_visibility = "private", noinline}
```

### Built-in Attributes

| Attribute Type | Example Textual Form | C++ Type |
|---------------|---------------------|----------|
| `IntegerAttr` | `42 : i32` | `IntegerAttr` |
| `FloatAttr` | `3.14 : f64` | `FloatAttr` |
| `StringAttr` | `"hello"` | `StringAttr` |
| `ArrayAttr` | `[1, 2, 3]` | `ArrayAttr` |
| `DictionaryAttr` | `{a = 1, b = "x"}` | `DictionaryAttr` |
| `TypeAttr` | `i32` | `TypeAttr` |
| `UnitAttr` | `unit` (or just presence in dict) | `UnitAttr` |
| `AffineMapAttr` | `affine_map<(d0) -> (d0 * 2)>` | `AffineMapAttr` |
| `DenseIntOrFPElementsAttr` | `dense<[1, 2, 3]> : tensor<3xi32>` | `DenseIntOrFPElementsAttr` |
| `SparseElementsAttr` | (sparse tensor constant) | `SparseElementsAttr` |
| `SymbolRefAttr` | `@func_name` | `FlatSymbolRefAttr` |

### Accessing Attributes

```cpp
// Get an attribute by name (from op's attribute dictionary):
Attribute val = op->getAttr("sym_name");

// Cast to a known type:
if (auto intAttr = dyn_cast<IntegerAttr>(val))
  int64_t n = intAttr.getInt();

// Typed accessor (generated by ODS):
auto funcOp = cast<func::FuncOp>(op);
StringRef name = funcOp.getSymName();  // accesses "sym_name" attribute
```

### Discardable vs Non-Discardable Attributes

**Non-discardable attributes**: defined by the op's ODS spec; the op verifier checks them. They carry semantic meaning.

**Discardable attributes**: extra attributes not in the ODS spec; passes can add/remove them freely. They have a dialect prefix: `{dialect.key = value}`. Example: `{llvm.align = 8}` added by an alignment optimization pass.

---

## 130.7 Affine Maps

### What Is an AffineMap?

`AffineMap` ([`mlir/include/mlir/IR/AffineMap.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/IR/AffineMap.h)) represents a function from a tuple of dimension variables and symbol variables to a tuple of affine expressions:

```
AffineMap: (d0, d1, ...) [s0, s1, ...] -> (expr0, expr1, ...)
```

where each `exprK` is an affine combination of dimensions and symbols.

### Affine Map Syntax

```mlir
// Identity map: 1D
affine_map<(d0) -> (d0)>

// 2D transposition:
affine_map<(d0, d1) -> (d1, d0)>

// Tiled access: tile of size 4
affine_map<(d0)[s0] -> (d0 + s0 * 4)>

// Row-major to column-major:
affine_map<(d0, d1) -> (d1, d0)>

// Multi-result (for linalg.generic indexing maps):
affine_map<(i, j, k) -> (i, k)>   // access to A[i,k] in matmul
affine_map<(i, j, k) -> (k, j)>   // access to B[k,j]
affine_map<(i, j, k) -> (i, j)>   // access to C[i,j]
```

### AffineExpr Operations

`AffineExpr` supports:
- `+`, `-` (addition/subtraction)
- `*` by a constant (multiplication)
- `floordiv`, `ceildiv` by a positive constant
- `mod` by a positive constant
- Composition: applying one map to another

```cpp
// Building an affine map in C++:
AffineExpr d0 = getAffineDimExpr(0, &ctx);
AffineExpr d1 = getAffineDimExpr(1, &ctx);
AffineExpr s0 = getAffineSymbolExpr(0, &ctx);

// (d0, d1)[s0] -> (d0 + s0, d1 * 2):
AffineMap map = AffineMap::get(/*numDims=*/2, /*numSymbols=*/1,
                                {d0 + s0, d1 * 2}, &ctx);
```

### Uses of AffineMap

- `memref<4x4xf32, affine_map<(d0, d1) -> (d1, d0)>>`: column-major storage layout
- `affine.for %i = 0 to 10`: bounds expressed as affine maps
- `affine.load %arr[affine_map<...>(%idx)]`: affine memory access
- `linalg.generic {indexing_maps = [...]}`: tensor contraction indexing

---

## 130.8 Locations

Every `Operation` and `Value` has an associated `Location` — source provenance information:

```cpp
// File:line:column location:
Location loc = FileLineColLoc::get(&ctx, "file.mlir", 42, 7);

// Fused location (e.g., after inlining):
Location fused = FusedLoc::get(&ctx, {loc1, loc2});

// Named location (e.g., variable name):
Location named = NameLoc::get(StringAttr::get(&ctx, "var_x"), loc);

// Unknown location:
Location unk = UnknownLoc::get(&ctx);
```

Locations are printed in textual MLIR as `loc(...)` suffixes:
```mlir
%x = arith.addi %a, %b : i32 loc("source.f90":10:5)
```

They flow through transformations (MLIR's pass manager preserves them) and appear in diagnostic messages.

---

## 130.9 Walking and Visiting IR

### walk()

The `walk()` method on any MLIR entity traverses all nested ops:

```cpp
// Walk all ops in a module (post-order by default):
module.walk([](Operation *op) {
  llvm::outs() << op->getName() << "\n";
});

// Walk specific op type:
module.walk([](arith::AddIOp addOp) {
  // Only called for arith.addi ops
  llvm::outs() << "Found addi at " << addOp.getLoc() << "\n";
});

// Pre-order walk with early exit:
module.walk<WalkOrder::PreOrder>([](func::FuncOp func) -> WalkResult {
  if (func.isPrivate())
    return WalkResult::interrupt();  // stop walking into this function
  return WalkResult::advance();
});
```

### getParentOfType

```cpp
// Find nearest enclosing op of a given type:
auto funcOp = op->getParentOfType<func::FuncOp>();
auto moduleOp = op->getParentOfType<ModuleOp>();
```

### Iterating Blocks and Ops

```cpp
// Iterate blocks of a region:
for (Block &block : region) {
  // Iterate ops of a block:
  for (Operation &op : block) {
    // ...
  }
}

// Iterate operands:
for (Value operand : op.getOperands()) {
  // ...
}

// Iterate results:
for (OpResult result : op.getResults()) {
  // ...
}
```

---

## Chapter Summary

- MLIR's structure is recursive: `Operation` → `Region` → `Block` → `Operation`; this allows function bodies, loop nests, GPU kernels, and hardware hierarchies to be expressed with the same primitives
- `Operation` is the universal IR node: it holds a name (dialect + opname), operands (SSA `Value`s), results (SSA `Value`s), attributes (compile-time data), regions (nested structure), and successors (for CFG branches)
- `Value` is either an `OpResult` (defined by an op) or a `BlockArgument` (defined at block entry); block arguments replace LLVM IR's PHI nodes and serve the same purpose
- MLIR's builtin type system covers integers (`i1`..`i1024`), floats (`f16`, `bf16`, `f32`, `f64`, `f80`, `f128`, `tf32`), `index`, `memref`, `tensor`, `vector`, and function types; each dialect extends this with domain-specific types
- `Block` arguments serve as the SSA join-point mechanism; terminators pass values to successor blocks via branch arguments
- `IsolatedFromAbove` regions (e.g., `func.func`) prevent capturing outer SSA values; non-isolated regions (e.g., `scf.for`) can capture freely from enclosing scopes
- `AffineMap` is a first-class IR concept representing multi-dimensional affine index functions; it underpins memory layouts in `memref`, loop bounds in `affine.for`, and indexing in `linalg.generic`
- Locations track source provenance through transformations; `walk()` and `getParentOfType()` are the primary IR navigation APIs
