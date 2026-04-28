# Chapter 22 — Metadata and Debug Info

*Part IV — LLVM IR*

Every instruction in LLVM IR has a fixed semantic meaning determined by its opcode, types, and operands. But the optimizer and the debugger need richer information that cannot be expressed in those three axes alone: the access type of a pointer load, the expected branch frequency derived from profiling, the original C source line for a store, the vectorization directives attached to a loop. LLVM attaches this supplementary information through a separate metadata system — a parallel universe of typed nodes that ride alongside the IR but are explicitly outside its value semantics. An optimizer may freely discard or modify metadata; doing so never changes the program's observable behavior (with one narrow exception). The debugger, by contrast, depends on the metadata surviving the entire compilation pipeline and being faithfully lowered to DWARF sections in the object file.

This chapter covers the entire metadata system: the node hierarchy and attachment APIs, the major optimization-guiding metadata kinds (TBAA, range, loop, profile, alias scope), the full DWARF debug-info family, the new debug-record format that decouples debug info from the SSA def-use graph, and the pseudo-probe infrastructure that enables context-sensitive sample profiling. All code examples are verified against LLVM 22.

---

## 22.1 Metadata Fundamentals: MDNode, MDString, ValueAsMetadata

### 22.1.1 What Metadata Is (and Is Not)

Metadata in LLVM is a first-class system that exists alongside IR values but is not part of the IR's operational semantics. An instruction's behavior is fully determined by its opcode, type, and operands. Metadata attached to that instruction is a side channel — a set of annotations that passes may read and act on, but may also ignore, remove, or replace. The only exception is `llvm.dbg.*` intrinsics in the old debug-info format (Section 22.7), which are technically instructions and therefore nominally in the SSA graph; but even they carry no semantic meaning that affects memory or registers, and the new debug-record format (Section 22.8) removes them from the instruction stream entirely.

The practical consequence is that metadata enables a clean separation of concerns: correctness-critical information lives in instructions; optimization hints, profiling data, and debugger annotations live in metadata. A pass that knows nothing about loop unroll hints will not miscompile the loop; it will simply not unroll it.

### 22.1.2 The MDNode Class Hierarchy

Every piece of metadata is an instance of `llvm::MDNode`, which lives in [`llvm/include/llvm/IR/Metadata.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/Metadata.h). The hierarchy is:

```
Metadata
├── MDNode                    (base for compound metadata)
│   ├── MDTuple               (generic heterogeneous tuple: !{...})
│   └── <DWARF specializations>
│       ├── DIScope           (abstract)
│       │   ├── DIFile
│       │   ├── DICompileUnit
│       │   ├── DISubprogram
│       │   ├── DILexicalBlock
│       │   └── DILexicalBlockFile
│       ├── DIType            (abstract)
│       │   ├── DIBasicType
│       │   ├── DIDerivedType
│       │   ├── DICompositeType
│       │   └── DISubroutineType
│       ├── DILocalVariable
│       ├── DILocation
│       └── DIExpression
├── MDString                  (leaf: !"string value")
└── ValueAsMetadata           (wraps an IR Value into metadata)
    ├── ConstantAsMetadata    (wraps a Constant*)
    └── LocalAsMetadata       (wraps a non-constant Value*)
```

`MDTuple` is the workhorse. Every `!{...}` literal in textual IR is an `MDTuple`. Its operands are `Metadata*` — they can be other `MDNode` subtypes, `MDString` leaves, or `ValueAsMetadata` wrappers. In binary bitcode, metadata nodes are numbered sequentially; in textual IR those numbers appear as `!42`. Named metadata nodes (e.g., `!llvm.dbg.cu`, `!llvm.module.flags`, `!llvm.ident`) are module-level named lists of `MDNode*` — they do not have numbers and are accessed by name.

### 22.1.3 The MDKind Registry

Attaching metadata to an instruction requires a kind ID: a small integer identifying what kind of metadata this is. The kind registry lives in `LLVMContext`; every context maintains its own mapping from kind name to kind ID. Standard well-known kinds have stable IDs by convention:

```cpp
// Well-known kind IDs from llvm/include/llvm/IR/FixedMetadataKinds.def (LLVM 22)
//   0  = MD_dbg             ("dbg")
//   1  = MD_tbaa            ("tbaa")
//   2  = MD_prof            ("prof")
//   3  = MD_fpmath          ("fpmath")
//   4  = MD_range           ("range")
//   5  = MD_tbaa_struct     ("tbaa.struct")
//   6  = MD_invariant_load  ("invariant.load")
//   7  = MD_alias_scope     ("alias.scope")
//   8  = MD_noalias         ("noalias")
//  15  = MD_unpredictable   ("unpredictable")
//  18  = MD_loop            ("llvm.loop")
//  25  = MD_access_group    ("llvm.access.group")
//  26  = MD_callback        ("callback")
```

User-defined metadata kinds are registered at runtime with `LLVMContext::getMDKindID("my.custom.tag")`, which returns the next available integer. This makes the kind registry extensible for downstream tools without requiring changes to the core IR.

### 22.1.4 Attaching and Retrieving Metadata

The `Instruction` API for metadata is symmetric:

```cpp
// Attach:
Instruction *I = ...;
MDNode *Node = MDNode::get(Ctx, {MDString::get(Ctx, "branch_weights"),
                                  ConstantAsMetadata::get(ConstantInt::get(i32, 99)),
                                  ConstantAsMetadata::get(ConstantInt::get(i32, 1))});
I->setMetadata(LLVMContext::MD_prof, Node);

// Retrieve by well-known kind:
MDNode *Prof = I->getMetadata(LLVMContext::MD_prof);

// Retrieve by name (for custom kinds):
unsigned KindID = Ctx.getMDKindID("my.annotation");
MDNode *Custom = I->getMetadata(KindID);
```

The `setMetadata` call does not copy the node — it increases a reference count. Metadata nodes are uniqued within a context: two `MDTuple` nodes with the same operands in the same `LLVMContext` are the same pointer. `distinct` metadata (written `distinct !{...}` in textual IR) bypasses uniquing; it is required for nodes that must have identity, such as the `DISubprogram` attached to a function (two functions cannot share the same program node even if their names happen to be identical).

### 22.1.5 Named Module-Level Metadata

Several important metadata lists are attached to the module rather than to individual instructions. The two most prominent:

- `!llvm.dbg.cu` — a list of `DICompileUnit` nodes; the starting point for iterating all debug information in the module. The bitcode writer uses this list to identify which compile-unit nodes to serialize.
- `!llvm.module.flags` — a list of `!{i32 behavior, !"key", value}` triples. The behavior field is one of: `1` (Error if conflicting), `2` (Warning if conflicting), `3` (Require: asserts a particular flag must be present), `4` (Override: later wins), `5` (Append), `6` (AppendUnique). Linkers and passes use these flags for ABI compatibility checks, e.g., `!{i32 1, !"wchar_size", i32 4}`.
- `!llvm.ident` — a single string identifying the producer (`!{!"clang version 22.1.3"}`).

---

## 22.2 MD_tbaa — Type-Based Alias Analysis Metadata

### 22.2.1 Strict Aliasing and Why It Matters

C and C++ impose the strict aliasing rule: an object of type `T` may only be accessed through a pointer to `T`, through a pointer to a compatible type (e.g., a signed/unsigned variant), or through a `char*` pointer. Accessing an `int` object through a `float*` pointer is undefined behavior. Because the front-end can rely on this rule, it emits metadata that tells the optimizer: *these two memory operations cannot alias because their types are incompatible*.

Without TBAA, a `load i32, ptr %p` and a `store float, ptr %q` in the same basic block would force the alias analysis to conservatively assume `%p` and `%q` might alias — blocking load-store reordering and store elimination. With TBAA, the optimizer can prove they access different type families and schedule them freely.

### 22.2.2 The TBAA Metadata Structure

TBAA metadata in LLVM uses a *type tree* rooted at an omnipotent root node. Each node in the tree represents a C/C++ type. A pointer to type `T` may alias a pointer to type `S` only if `T` and `S` are the same node, or one is an ancestor of the other in the type tree. `char` (and `std::byte`) is always a child of the root, meaning char pointers may alias any other pointer.

```llvm
; TBAA type tree for a simple C module:
!0 = !{!"Simple C/C++ TBAA"}          ; root (omnipotent)
!1 = !{!"omnipotent char", !0, i64 0} ; char — child of root; aliases everything
!2 = !{!"int", !1, i64 0}             ; int — child of char
!3 = !{!"float", !1, i64 0}           ; float — child of char

; Access tags on memory operations:
; Format: {base type node, access type node, offset in bytes}
!4 = !{!2, !2, i64 0}    ; access tag: int access at offset 0
!5 = !{!3, !3, i64 0}    ; access tag: float access at offset 0

define void @f(ptr %p, ptr %q) {
  store i32 42, ptr %p, !tbaa !4   ; writing int via %p
  %v = load float, ptr %q, !tbaa !5 ; reading float via %q — does NOT alias %p
  ret void
}
```

The access tag `!{base, access, offset}` states: "this instruction accesses an object of type `access` at byte offset `offset` within an enclosing object of type `base`." For scalar accesses `base == access` and `offset == 0`. For struct field accesses, `base` is the struct type and `offset` is the field's byte offset.

### 22.2.3 Struct-Path TBAA

*Struct-path TBAA* (also called *path-based TBAA*) is the refined variant that Clang emits to reason about struct field accesses precisely. The access tag `{struct_type, field_type, field_offset}` tells the optimizer which field is being accessed, enabling it to distinguish between accesses to different fields of the same struct even when both fields have compatible base types:

```llvm
; struct Point { int x; int y; };
; Two fields, both int, but different offsets — they don't alias each other.

!10 = !{!"Point", !2, i64 0, !2, i64 4}  ; struct descriptor: name, type0, offset0, type1, offset1
!11 = !{!10, !2, i64 0}   ; access tag for .x (offset 0)
!12 = !{!10, !2, i64 4}   ; access tag for .y (offset 4)

store i32 1, ptr %px, !tbaa !11   ; store to Point.x
%vy = load i32, ptr %py, !tbaa !12 ; load from Point.y — does NOT alias Point.x
```

This is the metadata Clang's `TBAAAccessInfo` and `CodeGenTBAA` emit for every struct field load and store. The relevant source file is [`clang/lib/CodeGen/CodeGenTBAA.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CodeGenTBAA.cpp).

### 22.2.4 The `-fno-strict-aliasing` Effect

When Clang is invoked with `-fno-strict-aliasing`, it does not omit TBAA metadata entirely — it emits all memory accesses with access tags pointing to the omnipotent root node. Since every type is a descendant of the root, every pair of accesses may alias, and the optimizer recovers the same conservatism as if no TBAA were present. The key invariant: *TBAA is always attached; its information content varies by flag*.

---

## 22.3 MD_range and MD_unpredictable

### 22.3.1 Value Range Metadata (`!range`)

The `!range` metadata on a `load` or `call` instruction states that the resulting value is guaranteed to fall within a specific set of half-open integer intervals. This enables the optimizer to remove range checks, fold comparisons, and propagate tighter bounds through subsequent computations.

The format is an MDTuple containing alternating lower/upper bounds: `!{i64 lo0, i64 hi0, i64 lo1, i64 hi1, ...}`. Each pair `[lo, hi)` is a half-open interval. If multiple pairs are present, the value is in the disjoint union of those intervals.

```llvm
; A load whose result is a valid byte value in [0, 256):
%byte = load i32, ptr %p, align 4, !range !{i32 0, i32 256}

; A call returns a value in {0..9} union {20..29}:
%v = call i32 @get_category(), !range !{i32 0, i32 10, i32 20, i32 30}

; Optimizer can fold: %cmp = icmp slt i32 %v, 30 → true (always)
; Optimizer can fold: %cmp2 = icmp sge i32 %v, 0  → true (always)
```

More commonly Clang uses `!range` on loads from bitfields. A 3-bit bitfield has values in `[0, 8)`, and after extraction Clang emits `!range !{i32 0, i32 8}` on the load, enabling comparison folding throughout the block. The relevant Clang logic is in [`clang/lib/CodeGen/CGExpr.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGExpr.cpp) in the `EmitLoadOfBitfieldLValue` path.

The constraint for validity: ranges must be non-empty, non-wrapping (except for a single pair that wraps and thus describes the full range), and cover all possible return values. Violating this constraint is undefined behavior — the optimizer may exploit it aggressively. As of LLVM 22, the `!range` metadata is checked by the verifier and rejected if the format is malformed.

### 22.3.2 `MD_unpredictable` — Unpredictable Branch Hint

The `!unpredictable` metadata attached to a branch instruction is a one-bit hint: this branch is architecturally unpredictable, and the backend should not attempt static branch-prediction optimizations. It appears as an empty MDTuple `!{}`.

```llvm
; Branch whose outcome cannot be predicted — disable prediction optimization.
br i1 %cond, label %true_bb, label %false_bb, !unpredictable !{}
```

The primary use case is security-sensitive code. Spectre-v1 mitigations in the Linux kernel and in Chromium's renderer processes tag certain security-critical branches with `!unpredictable` to prevent the CPU's branch predictor from speculating past them (in conjunction with `llvm.speculation.safe.load` intrinsics). A secondary use is code generated from hash tables or from random-number-based dispatch, where training the branch predictor is impossible and the hint avoids wasted prediction resources.

The backend translates `!unpredictable` into architecture-specific hints: on x86 it inserts `PAUSE` or uses the lfence barrier pattern; on ARM it may use `csdb` (Consumption of Speculative Data Barrier). The exact lowering is target-specific and is tracked through `MachineInstr::isUnpredictable()`.

---

## 22.4 MD_loop — Loop Transformation Hints

### 22.4.1 Structure of Loop Metadata

Loop metadata is attached to the *back-edge branch* of a loop — the conditional or unconditional branch at the end of the loop's latch block that jumps back to the header. This placement is architecturally deliberate: the loop header may have multiple predecessors (the entry edge and the back-edge), but the back-edge branch is unique and is owned by exactly one block.

The metadata node has a self-referential structure. The first operand of the `!llvm.loop` MDTuple is the node itself; this self-reference is the canary that LoopInfo uses to find and verify loop metadata. Subsequent operands are sub-nodes, each a tuple whose first element is a string identifying the transformation:

```llvm
; A loop that should be vectorized with width 4 and unrolled twice.
br i1 %exit_cond, label %loop_header, label %loop_exit, !llvm.loop !10

!10 = distinct !{!10,         ; self-reference (required)
                 !11,          ; vectorize sub-node
                 !12,          ; vectorize.width sub-node
                 !13}          ; unroll.count sub-node

!11 = !{!"llvm.loop.vectorize.enable", i1 true}
!12 = !{!"llvm.loop.vectorize.width", i32 4}
!13 = !{!"llvm.loop.unroll.count", i32 2}
```

The self-referential `distinct` node (not a uniqued node) is necessary because two different loops might request the same transformation but must not share the same metadata identity — LoopInfo uses pointer identity to associate a loop with its metadata.

### 22.4.2 Catalogue of Loop Sub-Nodes

| Sub-node key | Value type | Effect |
|---|---|---|
| `llvm.loop.unroll.count` | `i32 N` | Unroll exactly N times |
| `llvm.loop.unroll.full` | (none) | Unroll completely; only valid for loops with constant trip count |
| `llvm.loop.unroll.disable` | (none) | Suppress all unrolling for this loop |
| `llvm.loop.vectorize.enable` | `i1 true/false` | Force-enable or force-disable vectorization |
| `llvm.loop.vectorize.width` | `i32 N` | Request vector width N; overrides cost model |
| `llvm.loop.interleave.count` | `i32 N` | Interleave N copies of the loop body for pipeline throughput |
| `llvm.loop.distribute.enable` | `i1 true` | Enable loop distribution (splitting into independent sub-loops) |
| `llvm.loop.pipeline.initiationinterval` | `i32 N` | Target initiation interval for software pipelining |
| `llvm.loop.mustprogress` | (none) | The loop is guaranteed to make progress (terminate or have observable side effects per iteration); enables dead-loop elimination |
| `llvm.loop.parallel_accesses` | `!{!access_group, ...}` | Asserts that the listed memory accesses are independent across iterations; used by the vectorizer |

### 22.4.3 Pragma-to-Metadata Lowering in Clang

Clang translates `#pragma clang loop` and `#pragma unroll` directives into loop metadata during CodeGen. The relevant source is [`clang/lib/CodeGen/CGLoopInfo.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGLoopInfo.cpp).

```c
// C source with loop pragmas
#pragma clang loop vectorize(enable) vectorize_width(8) interleave_count(2)
for (int i = 0; i < n; i++) a[i] += b[i];

#pragma unroll 4
for (int i = 0; i < 16; i++) sum += arr[i];
```

This produces the following IR loop metadata (abridged):

```llvm
; Back-edge branch of the vectorize loop:
br i1 %exit, label %header, label %exit_bb, !llvm.loop !20

!20 = distinct !{!20, !21, !22, !23}
!21 = !{!"llvm.loop.vectorize.enable", i1 true}
!22 = !{!"llvm.loop.vectorize.width", i32 8}
!23 = !{!"llvm.loop.interleave.count", i32 2}

; Back-edge branch of the unroll loop:
br i1 %exit2, label %header2, label %exit2_bb, !llvm.loop !30

!30 = distinct !{!30, !31}
!31 = !{!"llvm.loop.unroll.count", i32 4}
```

The LoopVectorize pass reads `!22` and `!23`; the LoopUnroll pass reads `!31`. Each pass removes (or replaces with a `disable` marker) the metadata it has consumed, so downstream passes do not re-transform an already-transformed loop.

---

## 22.5 MD_prof — Profile Metadata

### 22.5.1 Branch Weights

The `!prof` metadata with a `branch_weights` tag provides the profile-guided optimization (PGO) subsystem with execution frequency information at branch sites. It is attached to `br`, `switch`, and `select` instructions.

```llvm
; A branch that is taken 99% of the time:
br i1 %cond, label %hot_path, label %cold_path, !prof !{!"branch_weights", i32 99, i32 1}

; A switch with per-case weights:
switch i32 %val, label %default [
  i32 0, label %case0
  i32 1, label %case1
  i32 2, label %case2
], !prof !{!"branch_weights", i32 5, i32 1000, i32 200, i32 3}
; Weight order: default, case0, case1, case2
```

The weights are relative, not absolute — only the ratios matter. The BranchProbabilityInfo pass converts them to `BranchProbability` fractions (numerator/denominator over 2^31). The BlockFrequencyInfo pass then propagates block execution frequencies through the CFG using a top-down frequency assignment. These frequencies feed into: inlining cost decisions (a hot call site gets more aggressive inlining), code layout (MachineBlockPlacement moves hot blocks to fall-through), loop optimization, and register allocation priority.

Clang emits branch weights from two sources:

1. **`__builtin_expect(expr, expected)`**: Clang translates this to a branch with `!prof !{!"branch_weights", i32 2000, i32 1}` for the expected case being strongly preferred.
2. **PGO instrumentation**: The `-fprofile-generate` / `-fprofile-use` pipeline instruments branch sites and loads per-branch counters into `!prof` nodes during the profile-use build.

### 22.5.2 Function Entry Counts

```llvm
; The function was called N times during the profiling run.
define i32 @hot_function(i32 %x) !prof !{!"function_entry_count", i64 8500000} {
  ...
}
```

The `function_entry_count` is attached to the function's IR definition (not to a specific instruction) as a function-level metadata entry. The inliner reads this to prioritize inlining of frequently-called functions. It also feeds into the per-function block frequency normalization: a function with a high entry count allocates more of the binary's "hot section" budget.

A synthetic profile for functions with zero count uses `!"synthetic_function_entry_count"` to distinguish instrumentation-absent cases from genuinely cold functions.

### 22.5.3 Value Profiles

Value profiles record not just *whether* a branch was taken but *which value* an expression took at runtime. The format is:

```llvm
; Indirect call value profile: VP kind=0, total=1000, [target_address, count] pairs
call void %fp(i32 %x), !prof !{!"VP", i32 0, i64 1000,
                                 i64 4198400, i64 980,   ; address 4198400, count 980
                                 i64 4201600, i64 20}     ; address 4201600, count 20
```

The `VP` tag supports three kinds:
- Kind 0: indirect call target profiling — the optimizer promotes the most frequent target to a direct call guarded by an `if` (indirect call promotion, ICP).
- Kind 1: memcpy/memset size profiling — enables size-specialized versions.
- Kind 2: GCD-based division profiling.

Value profiles are the backbone of the CSSPGO (Context-Sensitive Sample PGO) pipeline described in Chapter 67.

---

## 22.6 MD_alias_scope and MD_noalias

### 22.6.1 The Scope Domain Model

TBAA (Section 22.2) is a type-hierarchy-based approach to non-aliasing. Alias scope metadata is a *set-based* approach that is independent of types. It directly asserts: *this memory operation does not alias any operation in scope set S*. The two metadata kinds work together:

- `!alias.scope !{!scope0, !scope1, ...}` — this operation participates in these scopes.
- `!noalias !{!scope0, !scope1, ...}` — this operation does not alias any operation that participates in these scopes.

Scopes are organized into domains. A domain is simply a unique metadata node that names a region of non-aliasing guarantees:

```llvm
; Define a domain and two scopes within it:
!domain = distinct !{!"kernel loop domain"}
!scope0 = distinct !{!scope0, !domain, !"pointer A scope"}
!scope1 = distinct !{!scope1, !domain, !"pointer B scope"}
```

Two operations in different scopes within the same domain do not alias. An operation annotated with `!noalias !{!scope0}` asserts it does not alias any operation annotated with `!alias.scope !{!scope0}`.

### 22.6.2 Key Use Cases

**Vectorized loops.** When the loop vectorizer creates multiple iterations in a single vector pass, it marks loads from different loop iterations with distinct scopes to assert they access independent memory locations:

```llvm
; Two loads from different iterations of a vectorized loop:
%v0 = load float, ptr %ptr0, !alias.scope !{!iter0_scope}, !noalias !{!iter1_scope}
%v1 = load float, ptr %ptr1, !alias.scope !{!iter1_scope}, !noalias !{!iter0_scope}
```

**Function inlining.** When an outlined function is inlined, the inliner may add `!noalias` metadata to the inlined memory operations, asserting that parameters of the inlined callee do not alias each other (when the call site can statically prove this). This recovers non-aliasing information that the callee's body lost when reasoning purely about its own parameters.

**`__restrict__` pointers.** Clang lowers C99 `restrict` qualifiers and C++ `__restrict__` to `!noalias` metadata on all loads and stores through the restricted pointer. The C99 restrict promise — that no other accessible object aliases the restricted pointer during the function's execution — maps exactly onto the `!noalias` annotation.

```c
void vadd(float * __restrict__ a, const float * __restrict__ b, int n) {
    for (int i = 0; i < n; i++) a[i] += b[i];
}
```

Clang emits `!noalias` on every load from `b` and every store to `a`, asserting they operate in disjoint memory regions. Without `restrict`, the store to `a[i]` and the load from `b[i+1]` might alias, forcing the vectorizer to emit a scalar fallback.

---

## 22.7 DWARF Debug Info Metadata: DICompileUnit, DISubprogram, DILocalVariable

### 22.7.1 The DIScope Hierarchy

Debug information in LLVM IR is structured as a tree of `DIScope` nodes mirroring the source language's scoping rules. Every `DILocation` that annotates an instruction must reference a scope, and that scope must be reachable from one of the compile-unit roots in `!llvm.dbg.cu`.

```
DIScope (abstract)
├── DIFile                 — a source file name + directory
├── DICompileUnit          — top-level: language, file, producer, flags
├── DISubprogram           — a function definition or declaration
├── DILexicalBlock         — a {} block within a function
├── DILexicalBlockFile     — #include file boundary within a function
└── DINamespace            — C++ namespace
```

### 22.7.2 DICompileUnit

`DICompileUnit` is the root of all debug information for a single translation unit. It is always `distinct` (because two separate compilations of the same file produce two distinct compile units):

```llvm
!0 = distinct !DICompileUnit(
    language: DW_LANG_C_plus_plus_14,   ; DWARFv5 language code
    file: !1,                            ; DIFile reference
    producer: "clang version 22.1.3",
    isOptimized: false,
    runtimeVersion: 0,
    emissionKind: FullDebug,            ; FullDebug | LineTablesOnly | NoDebug
    enums: !2,                           ; list of DICompositeType (enums)
    globals: !3,                         ; list of DIGlobalVariableExpression
    splitDebugFilename: "",
    nameTableKind: Default)              ; None | Default | GNU

!1 = !DIFile(filename: "main.cpp", directory: "/home/user/proj")
```

The `emissionKind` field controls how much debug information is emitted. `FullDebug` produces complete DWARF including type information and variable locations. `LineTablesOnly` produces only line number tables (adequate for stack traces and profiling, but not for source-level debugging). `NoDebug` suppresses all debug info.

### 22.7.3 DISubprogram

`DISubprogram` describes a function. There are two forms: a *definition* (attached to the IR `Function`) and a *declaration* (referenced by the definition to capture the prototype):

```llvm
define i32 @add(i32 %a, i32 %b) !dbg !4 {
entry:
  %r = add nsw i32 %a, %b, !dbg !8
  ret i32 %r, !dbg !9
}

!4 = distinct !DISubprogram(
    name: "add",
    linkageName: "_Z3addii",             ; mangled name
    scope: !1,                           ; DIFile (file-level scope)
    file: !1,
    line: 1,
    type: !5,                            ; DISubroutineType
    scopeLine: 1,
    flags: DIFlagPrototyped,
    spFlags: DISPFlagDefinition | DISPFlagOptimized,
    unit: !0,                            ; DICompileUnit
    retainedNodes: !6)                   ; list of DILocalVariable

!5 = !DISubroutineType(types: !{i32, i32, i32})
                                         ; return type, then parameter types

!8 = !DILocation(line: 3, column: 14, scope: !4)
!9 = !DILocation(line: 3, column: 3,  scope: !4)
```

The `!dbg !4` on the function definition links the IR `Function` to its `DISubprogram`. The `!dbg !8` on the instruction links it to a source location (`DILocation`). This two-level indirection — instruction → DILocation → DISubprogram → DICompileUnit — is the chain that LLDB and GDB follow to map an instruction pointer to a source file and line number (Chapter 116).

### 22.7.4 The DIType Hierarchy

Types in LLVM debug info form their own hierarchy parallel to LLVM's IR type system:

| DWARF node | C/C++ equivalent | Key fields |
|---|---|---|
| `DIBasicType` | `int`, `float`, `bool` | `name`, `size` (bits), `encoding` (DW_ATE_*) |
| `DIDerivedType` | pointer, reference, typedef, `const`, `volatile`, `restrict` | `tag` (DW_TAG_pointer_type etc.), `baseType`, `size`, `offset` |
| `DICompositeType` | struct, union, class, array, enum | `tag`, `elements` (list of DIDerivedType/DISubprogram members), `size` |
| `DISubroutineType` | function type | `types` (return + params), `flags` |

A struct `Point { int x; int y; }` produces:

```llvm
!20 = distinct !DICompositeType(
    tag: DW_TAG_structure_type,
    name: "Point",
    file: !1,
    line: 5,
    size: 64,                ; 64 bits = 8 bytes
    elements: !21)

!21 = !{!22, !23}

!22 = !DIDerivedType(tag: DW_TAG_member, name: "x",
                     baseType: !24, size: 32, offset: 0)
!23 = !DIDerivedType(tag: DW_TAG_member, name: "y",
                     baseType: !24, size: 32, offset: 32)
!24 = !DIBasicType(name: "int", size: 32, encoding: DW_ATE_signed)
```

### 22.7.5 DILocalVariable and the `llvm.dbg` Intrinsics

Local variables are described by `DILocalVariable` nodes, each of which references the scope in which the variable is declared:

```llvm
!30 = !DILocalVariable(name: "x", scope: !4, file: !1, line: 2,
                       type: !24)   ; int x

; Old format: dbg.declare intrinsic (deprecated in LLVM 19+)
call void @llvm.dbg.declare(
    metadata ptr %x.addr,          ; the alloca where x lives
    metadata !30,                   ; DILocalVariable
    metadata !DIExpression())       ; no transformation (direct address)

; Old format: dbg.value intrinsic (deprecated in LLVM 19+)
call void @llvm.dbg.value(
    metadata i32 %x_val,           ; the SSA value of x at this point
    metadata !30,                   ; DILocalVariable
    metadata !DIExpression())
```

`llvm.dbg.declare` says: "the variable described by `!30` lives at the memory address stored in `%x.addr` for the duration of the scope." It is used for variables that have an alloca (address-taken variables, aggregates, variables whose address is passed to functions).

`llvm.dbg.value` says: "at this program point, the value of the variable described by `!30` is `%x_val`." It is used for promotable scalars where `mem2reg` has converted the alloca to an SSA value.

The `!DIExpression()` operand can encode arbitrary DWARF expression fragments: offsets, dereferences, bit extractions. For a simple variable, it is empty. For a field of a struct, it might be `!DIExpression(DW_OP_plus_uconst, 4)` to denote an 4-byte offset within the described object.

### 22.7.6 TBAA for Struct Fields: A Complete Example

Tying together the struct-path TBAA (Section 22.2) and the debug info type system (Section 22.7.4), here is the metadata a Clang compilation of a simple struct access produces:

```llvm
; C source: struct Pair { int a; float b; };
;           int read_a(struct Pair *p) { return p->a; }

define i32 @read_a(ptr %p) !dbg !50 {
entry:
  %a_ptr = getelementptr inbounds %struct.Pair, ptr %p, i32 0, i32 0
  %val = load i32, ptr %a_ptr, align 4,
             !tbaa !60,     ; struct-path TBAA: Pair::a
             !dbg !55       ; source location

  ret i32 %val, !dbg !56
}

; --- TBAA nodes ---
!tbaa_root  = !{!"Simple C/C++ TBAA"}
!char_node  = !{!"omnipotent char", !tbaa_root, i64 0}
!int_node   = !{!"int", !char_node, i64 0}
!pair_node  = !{!"Pair", !char_node, i64 0, !int_node, i64 0,
                                             !float_node, i64 4}
!60 = !{!pair_node, !int_node, i64 0}   ; access tag: Pair.a at offset 0

; --- Debug info nodes ---
!50 = distinct !DISubprogram(name: "read_a", ...)
!55 = !DILocation(line: 2, column: 30, scope: !50)
!56 = !DILocation(line: 2, column: 23, scope: !50)
```

---

## 22.8 The New Debug-Record Format (RemoveDIs)

### 22.8.1 The Problem with Intrinsic-Based Debug Info

In the old format, `llvm.dbg.declare` and `llvm.dbg.value` are full `CallInst` nodes in the IR. They sit in basic blocks alongside arithmetic and memory instructions, appear in the instruction iterator, occupy slots in the use-def graph (they use the alloca and SSA values as operands), and must be explicitly skipped by every pass that processes instructions. This creates several problems:

1. **Pass complexity.** Every transformation pass must handle the case that a `dbg.value` intrinsic intervenes between two instructions it wants to move or combine. The `isDbgInfoIntrinsic()` predicate appears in dozens of passes.
2. **Optimization inhibition.** Some transformations conservatively refuse to move code past debug intrinsics, leading to suboptimal IR in debug builds.
3. **Lost coverage.** Passes that were not written with debug intrinsics in mind may silently drop variable location information by failing to update `dbg.value` calls when they move or replace instructions.
4. **SROA/mem2reg interaction.** The SROA pass must specially-case `dbg.declare` intrinsics attached to alloca nodes it is splitting.

### 22.8.2 DbgVariableRecord and DbgLabelRecord

LLVM 18 introduced an experimental alternative: *debug records*, also known as the *RemoveDIs* format. LLVM 19 made it the default. The design separates debug info from the instruction stream entirely:

- **`DbgVariableRecord`**: encodes the information of a `llvm.dbg.declare` or `llvm.dbg.value` intrinsic, but is not an `Instruction`. It is attached to an `Instruction` as a pre-instruction record.
- **`DbgLabelRecord`**: encodes a `llvm.dbg.label` (for named labels in source, rare).

Both are instances of `DbgRecord`, declared in [`llvm/include/llvm/IR/DebugProgramInstruction.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/DebugProgramInstruction.h).

```
BasicBlock
└── Instruction (linked list, as before)
    ├── DbgRecord *DbgRecordList  (pre-attached records, new)
    │   ├── DbgVariableRecord(var: !30, value: %x, expr: !DIExpression())
    │   └── DbgVariableRecord(var: !31, value: %y, expr: !DIExpression())
    └── (next Instruction...)
```

The key property: `DbgRecord` nodes are not in the SSA def-use graph. They do not appear in `BasicBlock::iterator` when iterating instructions. They are accessed through a separate API:

```cpp
// Iterating debug records attached to an instruction:
for (DbgRecord &DR : I->getDbgRecordRange()) {
    if (auto *DVR = dyn_cast<DbgVariableRecord>(&DR)) {
        DILocalVariable *Var = DVR->getVariable();
        Value *Val = DVR->getValue();          // the SSA value or metadata
        DIExpression *Expr = DVR->getExpression();
        // ...
    }
}
```

A pass that moves an instruction is responsible for moving the associated debug records with it, but it never needs to skip them while iterating — they are invisible to `Instruction` iterators. The result is that the debug-intrinsic special-casing in hundreds of passes is removed, and the optimization pipeline can operate on clean instruction sequences.

### 22.8.3 Textual IR and Compatibility

In textual IR, the new format is distinguished by a module flag and the absence of `call void @llvm.dbg.*` instructions. Instead, each instruction may be followed by `  #dbg_value(...)` or `  #dbg_declare(...)` annotations:

```llvm
; New RemoveDIs textual format (LLVM 19+):
%x = alloca i32, align 4
    #dbg_declare(ptr %x, !30, !DIExpression(), !DILocation(line: 2, col: 7, scope: !4))
%y = add nsw i32 %a, %b, !dbg !8
    #dbg_value(i32 %y, !31, !DIExpression(), !DILocation(line: 3, col: 5, scope: !4))
```

The `--write-experimental-debuginfo=true` flag to `opt` and `llc` controls whether to emit in the new format. Passes that have not yet been updated to call `Instruction::getDbgRecordRange()` instead of using `dbg.value` intrinsics are automatically shimmed: a compatibility layer converts between the two representations at the pass boundary.

Passes are upgraded to the new format incrementally. The canonical tracking issue is in the LLVM GitHub issue tracker as "RemoveDIs migration." By LLVM 22, the majority of middle-end passes have been updated. The `DbgRecord::print()` method produces the `#dbg_*` textual annotations, and `VerifyFunctionDebugInfo` validates that no dangling variable references exist.

---

## 22.9 Pseudo-Probe Metadata

### 22.9.1 Motivation: Sample PGO Without Instrumentation Overhead

Traditional instrumentation-based PGO inserts counters at every basic block or edge. This produces high-quality profiles but incurs significant binary size and runtime overhead. Sample-based PGO uses hardware performance counters (e.g., Linux perf's `cycles` event) to periodically sample the instruction pointer and construct a histogram. The histogram is then mapped back to source locations to identify hot functions and callsites.

The difficulty is that the compiler transforms the IR significantly between front-end emission and final code generation: functions get inlined, blocks get merged, loops get unrolled. Mapping a sampled instruction address back to a source location through a fully-optimized binary is unreliable. Pseudo-probes solve this by embedding stable, lightweight markers into the IR that survive optimization (or can be explicitly preserved), and correlating sample counts with those markers rather than with raw instruction addresses.

### 22.9.2 The `llvm.pseudoprobe` Intrinsic

Each function is divided into *probes* at the start of each basic block and at each call site. A probe is emitted as:

```llvm
call void @llvm.pseudoprobe(
    i64 %guid,      ; function GUID (hash of the linkage name)
    i64 %index,     ; probe index within the function (0-based)
    i32 %type,      ; 0 = basic block probe, 1 = call site probe
    i64 %attr)      ; reserved attributes (currently 0)
```

These intrinsics are extremely lightweight: they compile to no instructions in the final binary. Instead, they are lowered to DWARF/ELF notes in a special `.pseudo_probe` section, which maps each probe to its GUID and index. When the Linux perf sampler fires, the sample address is looked up in this section to identify which probe fired, rather than requiring full debug-line-table resolution.

```llvm
; A function with two basic blocks, two probes:
define void @compute(i32 %n) {
entry:
  call void @llvm.pseudoprobe(i64 7234567890, i64 0, i32 0, i64 0)  ; probe 0
  %cmp = icmp sgt i32 %n, 0
  br i1 %cmp, label %loop_body, label %exit

loop_body:
  call void @llvm.pseudoprobe(i64 7234567890, i64 1, i32 0, i64 0)  ; probe 1
  ; ... loop body ...
  br i1 %next_cond, label %loop_body, label %exit

exit:
  ret void
}
```

The GUID `7234567890` is derived from hashing the function's linkage name, enabling the profiler to associate samples across separately compiled modules.

### 22.9.3 CSSPGO: Context-Sensitive Sample PGO

Standard sample PGO is *flat*: it records per-function call frequencies but not the calling context. `compute()` called from `hot_path()` and from `cold_path()` gets a single merged profile. CSSPGO (Context-Sensitive Sample PGO, described in Chapter 67) extends this by recording *call chains*: each sample carries a stack trace, and the profile records per-callsite frequency in addition to per-function frequency.

Pseudo-probes enable CSSPGO because each probe has a stable identity that survives inlining. When `compute()` is inlined into `hot_caller()`, the inlined pseudo-probe calls are annotated with the inline context (the callsite's GUID and probe index):

```llvm
; Inlined pseudo-probe: compute() inlined into hot_caller() at callsite probe 3
call void @llvm.pseudoprobe(i64 7234567890, i64 0, i32 0, i64 0),
    !dbg !{line: 15, col: 5, scope: !compute_subprogram,
           inlinedAt: !{line: 42, col: 10, scope: !hot_caller_subprogram}}
```

The `!dbg` attachment with `inlinedAt` provides the inline context that the CSSPGO profiling infrastructure uses to separate the "compute in hot context" profile from "compute in cold context." This enables the profile-use compilation to apply different inlining and optimization decisions depending on context — the single most powerful capability distinguishing CSSPGO from flat PGO.

The pseudo-probe infrastructure is implemented in [`llvm/lib/Transforms/IPO/SampleProfileProbe.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/IPO/SampleProfileProbe.cpp) and the profile reader in [`llvm/lib/ProfileData/SampleProf.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ProfileData/SampleProf.cpp).

---

## 22.10 Chapter Summary

- **Metadata is semantically inert** (with the narrow exception of `llvm.dbg.*` intrinsics in the old format). Metadata nodes (`MDTuple`, `MDString`, `ValueAsMetadata`) are typed side-channel annotations keyed by kind IDs registered in `LLVMContext`. Optimizers may freely discard or modify metadata without affecting program correctness; they may also *exploit* metadata to enable transformations that would otherwise require conservative assumptions.

- **TBAA metadata** enforces C/C++ strict aliasing rules by attaching type-tree nodes to every memory operation. Struct-path TBAA additionally encodes the field offset, enabling the optimizer to prove that `p->x` and `p->y` do not alias even when both are `int`. `-fno-strict-aliasing` emits omnipotent-root access tags that conservatively allow all aliasing.

- **`!range` metadata** on loads and calls provides constant-propagation-like information without requiring the values to be compile-time constants. It enables comparison folding and range-propagation through loaded values such as bitfield extractions. `!unpredictable` on branches disables branch-prediction optimization and is used for Spectre mitigations and security-critical conditional branches.

- **Loop metadata** (`!llvm.loop`) attached to back-edge branches carries vectorize, unroll, interleave, and pipeline hints from `#pragma clang loop` directives to the corresponding optimizer passes. The self-referential distinct node structure gives each loop a unique identity so that two loops requesting identical transformations remain independent.

- **Profile metadata** (`!prof`) carries branch weights (derived from PGO counters or `__builtin_expect`), function entry counts, and value profiles (indirect call targets, memcpy sizes). The BranchProbabilityInfo and BlockFrequencyInfo passes consume branch weights to drive code layout, inlining, and register allocation. `!alias.scope` and `!noalias` metadata express explicit non-aliasing contracts between specific memory operations, independent of the type hierarchy, and are used heavily by the vectorizer and by `__restrict__` lowering.

- **DWARF debug info metadata** (`DICompileUnit`, `DISubprogram`, `DILocalVariable`, `DIType` hierarchy, `DILocation`) forms a structured tree rooted at `!llvm.dbg.cu`. Every instruction annotated with `!dbg` carries a `DILocation` linking it to a source file, line, column, and scope chain. This is the information LLDB and GDB consume to provide source-level debugging (Chapter 116); the full DWARF encoding is covered in Chapter 117.

- **The RemoveDIs format** (default since LLVM 19) moves debug variable-location information out of the instruction stream and into `DbgVariableRecord` / `DbgLabelRecord` objects attached as pre-instruction records. Passes iterate instructions without encountering debug info; they access records through `Instruction::getDbgRecordRange()`. This removes hundreds of `isDbgInfoIntrinsic()` special cases and eliminates a class of optimization-inhibiting hazards.

- **Pseudo-probes** (`llvm.pseudoprobe`) embed stable function-level and callsite-level markers that survive optimization and enable sample-based PGO to attribute hardware performance counter samples to precise IR locations. Combined with inline-context `!dbg` annotations, pseudo-probes form the foundation of CSSPGO, which distinguishes hot and cold calling contexts and applies differentiated optimization strategies per context.

---

*Cross-references:* [Chapter 19 — Instructions I — Arithmetic and Memory](../part-04-llvm-ir/ch19-instructions-arithmetic-and-memory.md) · [Chapter 21 — SSA, Dominance, and Loops](../part-04-llvm-ir/ch21-ssa-dominance-and-loops.md) · [Chapter 67 — Profile-Guided Optimization](../part-10-analysis-middle-end/ch67-pgo.md) · [Chapter 116 — LLDB: Source-Level Debugging](../part-16-jit-sanitizers/ch116-lldb.md) · [Chapter 117 — DWARF in Depth](../part-16-jit-sanitizers/ch117-dwarf.md) · [Chapter 171 — The Undef/Poison Story Formally](../part-24-verified-compilation/ch171-the-undef-poison-story-formally.md)

*Reference links:* [LangRef — Metadata](https://llvm.org/docs/LangRef.html#metadata) · [LangRef — Debug Info](https://llvm.org/docs/LangRef.html#debug-info-metadata) · [LLVM TBAA Documentation](https://llvm.org/docs/AliasAnalysis.html#tbaa-metadata) · [RemoveDIs migration doc](https://llvm.org/docs/RemoveDIsDebugInfo.html) · [Pseudo-probes for sampling PGO](https://llvm.org/docs/PseudoProbeForSampling.html) · [llvm/include/llvm/IR/Metadata.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/Metadata.h) · [clang/lib/CodeGen/CodeGenTBAA.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CodeGenTBAA.cpp) · [clang/lib/CodeGen/CGLoopInfo.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGLoopInfo.cpp) · [llvm/lib/Transforms/IPO/SampleProfileProbe.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/IPO/SampleProfileProbe.cpp)
