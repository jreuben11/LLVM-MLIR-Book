# Chapter 228 — Virtual Machine Design: Bytecode Interpreters, GC Strategies, and Object Models

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

A virtual machine is a software interpreter that executes programs expressed in a portable bytecode rather than native machine code. The design decisions made when building a VM — how bytecode is encoded, how the interpreter dispatches instructions, how objects are laid out in memory, how garbage is collected — determine both the performance ceiling achievable before JIT compilation and the ease with which a JIT can be added later. This chapter presents VM design from first principles using production VMs as concrete references, then shows how LLVM ORC connects a bytecode interpreter to native-code tiered compilation.

---

## Table of Contents

- [228.1 Stack VM vs Register VM](#2281-stack-vm-vs-register-vm)
  - [Stack-Based Architecture](#stack-based-architecture)
  - [Register-Based Architecture](#register-based-architecture)
  - [The Instruction Count Trade-off](#the-instruction-count-trade-off)
- [228.2 Bytecode Instruction Set Design](#2282-bytecode-instruction-set-design)
  - [Opcode Width and Encoding](#opcode-width-and-encoding)
  - [Constant Pool](#constant-pool)
  - [Superinstructions](#superinstructions)
- [228.3 Interpreter Dispatch Strategies](#2283-interpreter-dispatch-strategies)
  - [Switch Dispatch](#switch-dispatch)
  - [Computed Goto / Direct Threading](#computed-goto-direct-threading)
  - [Indirect Threading](#indirect-threading)
  - [Copy-and-Patch JIT (CPython 3.13)](#copy-and-patch-jit-cpython-313)
- [228.4 Object Representation and Memory Layout](#2284-object-representation-and-memory-layout)
  - [Tagged Pointers](#tagged-pointers)
  - [NaN-Boxing](#nan-boxing)
  - [Object Header Word](#object-header-word)
  - [Hidden Classes (Shapes)](#hidden-classes-shapes)
- [228.5 Garbage Collection Strategies](#2285-garbage-collection-strategies)
  - [Mark-Sweep](#mark-sweep)
  - [Semi-Space Copying](#semi-space-copying)
  - [Generational GC](#generational-gc)
  - [Tri-Color Concurrent Marking](#tri-color-concurrent-marking)
  - [Reference Counting with Cycle Collection](#reference-counting-with-cycle-collection)
- [228.6 GC Roots and Precise vs Conservative Scanning](#2286-gc-roots-and-precise-vs-conservative-scanning)
  - [Precise Scanning](#precise-scanning)
  - [Conservative Scanning](#conservative-scanning)
  - [Safepoints](#safepoints)
- [228.7 Method Dispatch and Polymorphic Inline Caches](#2287-method-dispatch-and-polymorphic-inline-caches)
  - [Virtual Dispatch via Vtable](#virtual-dispatch-via-vtable)
  - [Polymorphic Inline Cache](#polymorphic-inline-cache)
  - [`invokedynamic` and Method Handles (JVM)](#invokedynamic-and-method-handles-jvm)
- [228.8 Frame Layout and Calling Conventions for Interpreted Code](#2288-frame-layout-and-calling-conventions-for-interpreted-code)
  - [Interpreter Frame Structure](#interpreter-frame-structure)
  - [CPython Frame Evolution](#cpython-frame-evolution)
  - [Coexistence of JIT and Interpreted Frames](#coexistence-of-jit-and-interpreted-frames)
- [228.9 Connecting the Interpreter to ORC JIT for Tiered Compilation](#2289-connecting-the-interpreter-to-orc-jit-for-tiered-compilation)
  - [The Tiering Architecture](#the-tiering-architecture)
  - [Call Count Instrumentation](#call-count-instrumentation)
  - [Emitting LLVM IR from the Interpreter's IR](#emitting-llvm-ir-from-the-interpreters-ir)
  - [Loading into ORC and Redirecting](#loading-into-orc-and-redirecting)
  - [Per-Function vs Tracing JIT](#per-function-vs-tracing-jit)
- [228.10 Reference VM Architectures](#22810-reference-vm-architectures)
  - [CPython](#cpython)
  - [Lua 5.4](#lua-54)
  - [Ruby YARV](#ruby-yarv)
  - [HotSpot JVM](#hotspot-jvm)
  - [SpiderMonkey](#spidermonkey)
  - [Comparison Table](#comparison-table)
- [Chapter Summary](#chapter-summary)

---

## 228.1 Stack VM vs Register VM

The fundamental architectural choice is where operands live during evaluation.

### Stack-Based Architecture

Stack VMs keep operands on an implicit evaluation stack. Instructions pop operands and push results:

```
; Compute (a + b) * c on a stack VM
LOAD a      ; stack: [a]
LOAD b      ; stack: [a, b]
ADD         ; stack: [a+b]
LOAD c      ; stack: [a+b, c]
MUL         ; stack: [(a+b)*c]
STORE result
```

Instruction encoding is compact — operands are implicit — but evaluation requires more instructions than a register VM for the same expression. Stack VMs include: JVM (all arithmetic via operand stack), CPython (prior to 3.11), WebAssembly.

### Register-Based Architecture

Register VMs explicitly name operands as virtual register numbers in each instruction:

```
; Compute (a + b) * c on a register VM
ADD r1, a, b    ; r1 = a + b
MUL r2, r1, c   ; r2 = r1 * c
```

Each instruction is wider (must encode register numbers), but the same computation requires fewer instructions. Register VMs include: Lua 5.4 (`lvm.c`), Android Dalvik/ART, Ruby YARV 3.x.

### The Instruction Count Trade-off

| Metric | Stack VM | Register VM |
|--------|----------|-------------|
| Instructions per expression | Higher (~1.5–2×) | Lower |
| Instruction width | Narrower (1–3 bytes typical) | Wider (4–8 bytes typical) |
| Code size | Smaller | Larger |
| Interpreter complexity | Simpler | Moderate |
| JIT backend impedance | Higher (stack → SSA conversion) | Lower (near-SSA already) |

CPython 3.11+ introduced a specializing adaptive interpreter that rewrites hot bytecodes in-place (`SPECIALIZE_LOAD_ATTR`, `SPECIALIZE_CALL`) to avoid dispatch overhead on typed fast paths — a technique that bridges the stack/register gap without changing the fundamental architecture.

---

## 228.2 Bytecode Instruction Set Design

### Opcode Width and Encoding

Three common strategies:

**Fixed-width 1-byte opcodes**: CPython (one `uint8_t` opcode followed by one `uint8_t` argument). Simple dispatch table; instruction boundaries are predictable. Limits argument range to 256 values — `EXTENDED_ARG` prefix handles larger operands.

**Fixed-width 2-byte (word-code)**: CPython 3.6+ uses `(opcode, oparg)` pairs (2 bytes each); CPython 3.12 moved to truly word-aligned instruction pairs for better performance.

**Variable-length instructions**: JVM uses 1–5 byte encodings per instruction (`wide` prefix expands local variable and constant pool indices). WebAssembly uses LEB128-encoded immediates.

### Constant Pool

The constant pool (JVM: per-class `.class` file structure; Lua: per-function `Proto` struct) stores literals, method names, and type descriptors referenced by instructions. Instructions reference pool entries by index rather than embedding constants directly:

```c
// Lua 5.4: Proto structure (lobject.h)
typedef struct Proto {
    TValue *k;       /* constants used by the function */
    int sizek;       /* size of `k' */
    Instruction *code;  /* opcodes */
    int sizecode;
    // ...
} Proto;
```

### Superinstructions

Combining frequent instruction pairs into a single opcode reduces dispatch overhead. CPython 3.11 `LOAD_FAST` + `LOAD_FAST` → `LOAD_FAST_LOAD_FAST` superinstruction. Generated automatically by measuring instruction pair frequencies in production workloads.

---

## 228.3 Interpreter Dispatch Strategies

### Switch Dispatch

The simplest implementation: a `switch` over the opcode in a loop.

```c
// Simplified CPython-style switch dispatch
while (true) {
    uint8_t opcode = *pc++;
    switch (opcode) {
    case LOAD_FAST: {
        int arg = *pc++;
        PUSH(frame->localsplus[arg]);
        break;
    }
    case BINARY_ADD: {
        PyObject *right = POP();
        PyObject *left = POP();
        PUSH(PyNumber_Add(left, right));
        break;
    }
    // ...
    }
}
```

The CPU's branch predictor sees all opcodes as targets of a single indirect branch — no per-opcode specialization possible. Modern CPUs mispredict ~10–15% of opcode dispatches.

### Computed Goto / Direct Threading

The GNU C `&&label` extension returns the address of a label as a `void*`. Each opcode handler jumps directly to the address of the next opcode's handler:

```c
// Direct-threaded dispatch (CPython ceval.c fast path with COMPUTED_GOTO)
static void *dispatch_table[] = {
    [LOAD_FAST] = &&TARGET_LOAD_FAST,
    [BINARY_ADD] = &&TARGET_BINARY_ADD,
    // ...
};

#define DISPATCH() goto *dispatch_table[*pc++]

TARGET_LOAD_FAST: {
    int arg = *pc++;
    PUSH(frame->localsplus[arg]);
    DISPATCH();
}
TARGET_BINARY_ADD: {
    PyObject *right = POP();
    PyObject *left  = POP();
    PUSH(PyNumber_Add(left, right));
    DISPATCH();
}
```

Each jump from a handler targets a specific subsequent handler — the CPU's indirect branch predictor can learn per-dispatch-site patterns. CPython enables this path with `USE_COMPUTED_GOTOS` when compiled with GCC or Clang.

Measured improvement over switch dispatch: 15–20% on typical Python workloads.

### Indirect Threading

Store the handler address directly in the bytecode array instead of an opcode index. No dispatch table lookup needed — each instruction word is its own handler pointer. Used in some Forth implementations and research VMs. Eliminates one memory indirection but inflates bytecode size and complicates serialisation.

### Copy-and-Patch JIT (CPython 3.13)

Copy-and-patch is a novel tier-1 JIT technique introduced in CPython 3.13 (PEP 744, Mark Shannon and Guido van Rossum):

1. For each bytecode handler, a **template** of native code is compiled at build time with **holes** (relocatable slots) for operands.
2. At JIT time, the template is **copied** into a code buffer and the holes are **patched** with actual operand values.
3. No full JIT compiler infrastructure is needed — no SSA construction, no register allocation.

```python
# LOAD_FAST template (conceptual):
#   mov rax, [rbp + HOLE_0]  ; HOLE_0 = offset of local variable
#   mov [rsp + HOLE_1], rax  ; HOLE_1 = stack push offset

# Copy-and-patch for: LOAD_FAST 3
# HOLE_0 ← 3 * sizeof(PyObject*)
# HOLE_1 ← current_stack_depth * sizeof(PyObject*)
```

This produces native code with zero interpreter overhead for individual instructions while avoiding the complexity of a full optimising JIT. Provides ~30% speedup over the adaptive interpreter for CPU-bound pure Python.

---

## 228.4 Object Representation and Memory Layout

### Tagged Pointers

Low-order bits of a pointer are available (on aligned allocations) as tag bits. Common conventions:

| Tag | Meaning |
|-----|---------|
| `...xxx0` | Pointer to heap object |
| `...xxx1` | Immediate small integer (value in high bits) |
| `...x11` | Other immediate (float, symbol, etc.) |

Ruby uses the lowest 3 bits: `...001` = Fixnum (integer × 2 + 1), `...000` = object pointer, `...010` = `false`, `...100` = `true`, `...110` = nil.

### NaN-Boxing

IEEE 754 double-precision NaN has 51 free payload bits. LuaJIT and Lua 5.x encode all values in a 64-bit NaN:

```
sign exponent(11)      mantissa(52)
   0 11111111111   <type-tag(13)> <payload(47)>
```

A quiet NaN (`exponent = 0x7FF`, `mantissa != 0`) encodes a non-double value. The 13-bit type tag distinguishes integer, boolean, nil, string pointer, object pointer. All 64-bit reads yield a double if the bits are a valid double, or a tagged value otherwise. Avoids object allocation for numeric-heavy code.

### Object Header Word

Every heap object begins with a header encoding GC metadata:

```c
// Simplified object header (common pattern)
typedef struct {
    uintptr_t header;  // [GC-flags:8][type-tag:8][ref-count or size:16][hash:32]
} ObjHeader;
```

The JVM `oopDesc` header on 64-bit HotSpot:
```
markWord (64 bits):
  [age:4][biased-lock:1][lock-state:2][hash:31][unused:26]
  (or forwarding pointer during GC evacuation)
klass pointer (32 bits, compressed oops)
```

CPython `PyObject`:
```c
struct _object {
    Py_ssize_t ob_refcnt;  // reference count (+ GC header prepended for tracked objects)
    PyTypeObject *ob_type; // type pointer (hot path for isinstance checks)
};
```

### Hidden Classes (Shapes)

Dynamic languages allow arbitrary property addition at runtime. Tracking property names per-object wastes memory. **Hidden classes** (V8: Maps; SpiderMonkey: Shapes; PyPy: Maps) cache the property layout for objects with identical property sequences:

```
Object {x:1, y:2}  ←  HiddenClass C1 {layout: [x@0, y@8]}
Object {x:3, y:4}  ←  same HiddenClass C1 (shared!)

After obj.z = 5:
Object {x:1, y:2, z:5}  ←  HiddenClass C2 {layout: [x@0, y@8, z@16]}
                             (transition: C1 --[add z]--> C2)
```

When an inline cache sees the same hidden class repeatedly, it can use a fixed offset lookup without any dictionary traversal.

---

## 228.5 Garbage Collection Strategies

### Mark-Sweep

**Mark phase**: traverse all reachable objects from roots (globals, stack, registers), set a mark bit on each. **Sweep phase**: scan the entire heap, collect all unmarked objects.

Simple but causes stop-the-world pauses proportional to heap size. Used in CPython's cyclic GC component (handles reference cycles; the primary allocator is reference counting).

### Semi-Space Copying

Divide the heap into two equal halves. Allocation uses a bump pointer in the "from-space". At GC time, copy all live objects to "to-space" (compacts in the process), update all pointers, swap "to-space" becomes the new "from-space".

Advantages: allocation is O(1) bump pointer; automatically compacts; no fragmentation. Disadvantage: wastes half the heap. Used as the young-generation collector in many implementations.

### Generational GC

Most objects die young (the "generational hypothesis"). Divide the heap into generations:
- **Young generation** (nursery): small, collected frequently (minor GC), low latency
- **Old generation** (tenured): large, collected rarely (major GC)

Objects surviving N minor GCs are **promoted** to the old generation. A **write barrier** tracks old→young pointers (a pointer from old generation into young generation must be in the **remembered set** so the young-generation collector knows all roots):

```c
// Write barrier (simplified)
void writeBarrier(Object *dst, Object **field, Object *newVal) {
    *field = newVal;
    if (isOldGen(dst) && isYoungGen(newVal))
        rememberSet.add(dst);
}
```

JVM G1 and ZGC use generational collection; CPython 3.13 added experimental generational GC.

### Tri-Color Concurrent Marking

The **tri-color invariant** enables concurrent GC (collector runs alongside mutator threads):
- **White**: not yet visited — candidates for collection
- **Grey**: discovered but children not yet scanned
- **Black**: fully scanned — all children are grey or black

The invariant: no black object points directly to a white object (would allow premature collection). Write barriers maintain this invariant when the mutator creates new pointers:

**SATB (Snapshot-At-The-Beginning)**: record the old value of any overwritten pointer. Used by HotSpot G1.

**IUC (Incremental Update)**: grey any newly pointed-to white object. Used by CMS, Shenandoah.

Lua 5.4 uses an incremental tri-color mark-sweep (not concurrent, but interleaved with mutator steps to avoid long pauses): `lgc.c`, `luaC_step()`.

### Reference Counting with Cycle Collection

CPython's primary GC is reference counting: each `PyObject` has `ob_refcnt`; `Py_INCREF`/`Py_DECREF` adjust it; `ob_refcnt == 0` triggers immediate deallocation.

Reference counting cannot collect cycles (objects that mutually reference each other). CPython's cyclic GC scans for isolated reference cycles among "container" objects (lists, dicts, sets, user classes). Runs periodically (generation-0 threshold default: 700 allocations). Controlled via `gc.collect()` and `gc.set_threshold()`.

---

## 228.6 GC Roots and Precise vs Conservative Scanning

### Precise Scanning

The GC knows exactly which words are pointers. Requires a **root map** (also called stackmap or OopMap) telling the GC where live heap references are at every safepoint:

```
Function foo(), at bytecode 42:
  rax = pointer to PyObject (live)
  rbx = integer value (not a pointer)
  [rbp-8] = pointer to PyObject (live)
  [rbp-16] = double-precision float (not a pointer)
```

LLVM generates these maps via `RewriteStatepointsForGC` (cross-ref [Chapter 221](ch221-speculative-optimization-inline-caches.md)) — the pass inserts `gc.statepoint` intrinsics at call sites and `gc.relocate` after them, with `.llvm_stackmaps` encoding the precise root set.

JVM HotSpot: each compiled method has an `OopMap` per safepoint. At a safepoint, all threads reach a known-safe state (polling-based: check a flag in a loop back-edge; signal-based on some platforms) before the GC scans stacks.

### Conservative Scanning

The GC treats any word that looks like a valid heap pointer as a GC root, without knowing definitively. The Boehm-Demers-Weiser GC (used in GCC Java frontend, Mono/C# in conservative mode, some C programs) scans all stack words and globals:

```c
// Conservative: if this word is in [heap_start, heap_end], treat as pointer
for (void **p = stack_top; p < stack_bottom; p++) {
    if (isHeapPointer(*p))
        mark(*p);
}
```

Advantages: no stackmap required — works with any native code including hand-written assembly. Disadvantages: interior pointers (pointing into the middle of an array) confuse the scanner; cannot move objects (can't update all pointers since some roots may be false positives).

### Safepoints

A safepoint is a point in a program where the GC can safely inspect and modify all thread stacks. For stop-the-world collection, all threads must reach a safepoint before collection begins.

**Polling safepoints**: compiler inserts a load from a safepoint flag page at loop back-edges and method entries. When the GC needs threads to stop, it marks the flag page non-readable; the resulting page fault stops the thread at the next poll.

**Signal-based**: GC sends `SIGUSR2` to all threads; signal handlers cooperate with the GC. Less predictable latency.

---

## 228.7 Method Dispatch and Polymorphic Inline Caches

### Virtual Dispatch via Vtable

C++ virtual dispatch: each class has a vtable (array of function pointers); each object has a vtable pointer as its first word. Lookup: `(*obj->vtable[method_index])(obj, ...)`. Fast (one indirection) but monomorphic — no adaptation for call-site type history.

### Polymorphic Inline Cache

An inline cache (IC) is a call-site-specific cache of the dispatch resolution for recently observed receiver types:

```
call_site_42:
  ; Polymorphic IC stub (type-keyed chain)
  cmp [receiver+type_offset], Type_A
  je  handler_for_A
  cmp [receiver+type_offset], Type_B
  je  handler_for_B
  jmp megamorphic_fallback
```

States:
1. **Uninitialized**: first call — record type and resolved target
2. **Monomorphic**: one type seen — direct call (fastest)
3. **Polymorphic**: 2–8 types — type-check chain
4. **Megamorphic**: >8 types — fall through to dictionary lookup

From the VM designer's perspective (contrasting with the JIT-compiler perspective of [Chapter 221](ch221-speculative-optimization-inline-caches.md)): ICs are the primary mechanism for removing dictionary-lookup overhead from attribute access and method calls. V8's IC system in Ignition is implemented in Torque (a typed assembly-like DSL compiled to machine code); CPython 3.11+ has a simpler in-bytecode specialization that rewrites `LOAD_ATTR` → `LOAD_ATTR_INSTANCE_VALUE` when the object has the right hidden class.

### `invokedynamic` and Method Handles (JVM)

JVM 7+ `invokedynamic` allows language implementors to provide their own linkage mechanism via a **bootstrap method**. On first execution, the bootstrap method returns a `MethodHandle` (a strongly typed reference to a callable); the JVM caches it and uses it for all subsequent calls. Used by JRuby, Groovy, Scala closures, and Java lambdas.

---

## 228.8 Frame Layout and Calling Conventions for Interpreted Code

### Interpreter Frame Structure

A typical interpreter frame:

```
High addresses
+------------------------+
| previous frame pointer |  ← link to caller's frame
| return address         |  ← where to resume after RETURN
| code object pointer    |  ← bytecode array, constants, etc.
| local variable 0       |  ← function argument 0
| local variable 1       |
| ...                    |
| local variable N       |
| eval stack slot 0      |  ← operand stack (stack VMs only)
| eval stack slot 1      |
| ...                    |
Low addresses (stack grows down)
```

### CPython Frame Evolution

CPython 3.11 (`_PyEvalFrameDefault` in `ceval.c`) replaced heap-allocated `PyFrameObject` with a lighter `_PyInterpreterFrame` stored on the C stack or in a per-thread frame allocator:

```c
// cpython/Include/internal/pycore_frame.h
typedef struct _PyInterpreterFrame {
    PyFunctionObject *f_func;  // NULL for module/class frames
    PyObject **localsplus;     // locals + eval stack contiguous array
    PyObject *f_locals;        // NULL until materialised
    PyCodeObject *f_code;
    _Py_CODEUNIT *prev_instr;  // last executed instruction
    int stacktop;
    uint16_t return_offset;
    char owner;                // FRAME_OWNED_BY_CSTACK, _GENERATOR, etc.
} _PyInterpreterFrame;
```

This eliminated per-call-frame heap allocation, reducing function call overhead by ~30%.

### Coexistence of JIT and Interpreted Frames

When a JIT-compiled function deoptimizes (guard violation), it must reconstruct an interpreter frame from the JIT frame's live values. The `.llvm_stackmaps` section (v3 format, §221.5) maps each JIT deopt point to the set of live values that reconstruct the interpreter frame state. The deopt handler:

1. Reads the JIT frame's registers and stack slots according to the stackmap
2. Allocates an interpreter frame
3. Fills in local variable slots and eval stack from the live values
4. Resumes at the interpreter bytecode offset corresponding to the deopt point

Exception handling in the interpreter differs fundamentally from native code: rather than `.eh_frame` / DWARF unwinding, the interpreter uses a per-function **exception table** mapping bytecode ranges to handler offsets (JVM style) or a simpler try/except block list (CPython `PyTryBlock`).

---

## 228.9 Connecting the Interpreter to ORC JIT for Tiered Compilation

### The Tiering Architecture

A production VM collects type feedback and call-count profiles during interpretation, then promotes hot functions to native code:

```
[Interpreter] ──count threshold──→ [LLVM IR Generator] ──→ [ORC JIT] ──→ [Native Code]
      ↑                                                                           |
      └──────────────────── deopt on guard failure ──────────────────────────────┘
```

### Call Count Instrumentation

The interpreter increments a counter at each function entry (or in loop back-edges for loops):

```c
// Inside the interpreter's CALL handling
++callee_code->call_count;
if (callee_code->call_count > JIT_THRESHOLD) {
    jit_compile(callee_code);  // trigger JIT compilation
}
```

### Emitting LLVM IR from the Interpreter's IR

For each bytecode instruction, emit the corresponding LLVM IR using `IRBuilder`:

```cpp
// Simplified: lowering BINARY_ADD for integer fast path
Value* lhs = loadLocal(builder, frame, lhs_slot);
Value* rhs = loadLocal(builder, frame, rhs_slot);
// Emit type guard: assert both are tagged integers
Value* lhsTagged = builder.CreateAnd(lhs, TagMask);
builder.CreateCondBr(
    builder.CreateICmpEQ(lhsTagged, IntTag),
    fast_path_bb, deopt_bb);
// Fast path: untag, add, retag
Value* lhsInt = builder.CreateAShr(lhs, TagShift);
Value* rhsInt = builder.CreateAShr(rhs, TagShift);
Value* sum    = builder.CreateAdd(lhsInt, rhsInt);
Value* result = builder.CreateOr(builder.CreateShl(sum, TagShift), IntTag);
```

### Loading into ORC and Redirecting

```cpp
// Add the compiled module to an ExecutionSession
ExitOnError ExitOnErr;
auto ES = std::make_unique<ExecutionSession>();
auto& JD = ES->createBareJITDylib("function_foo");

ThreadSafeModule TSM(std::move(Module), std::move(Ctx));
ExitOnErr(IRLayer.add(JD, std::move(TSM)));

// Look up and redirect
auto FooAddr = ExitOnErr(ES->lookup({&JD}, "foo"));
Redirector.redirect(JD, {{"foo", FooAddr}});
// All future calls to foo() now go to native code
```

The `RedirectableSymbolManager` (`llvm/include/llvm/ExecutionEngine/Orc/RedirectableSymbolManager.h`) atomically patches the dispatch table stub — no locking required; in-flight calls through the old stub complete normally.

### Per-Function vs Tracing JIT

**Per-function JIT** (used by JVM HotSpot C1/C2, this chapter's design): compile entire functions. Predictable, good for loop-heavy code. Must handle deopt for all guarded optimisations.

**Tracing JIT** (LuaJIT, historic Firefox TraceMonkey): record the sequence of operations along a hot path (a trace), compile only the trace. Very fast for tight numeric loops; struggles with large functions and diverse call paths. LuaJIT 2.x uses a per-trace compilation model with `lj_trace.c` driving trace recording.

---

## 228.10 Reference VM Architectures

### CPython

Source tree: `Objects/` (type implementations), `Python/ceval.c` (main evaluation loop), `Python/compile.c` (AST → bytecode), `Include/internal/pycore_frame.h` (frame layout), `Modules/gc.c` (cyclic GC).

Key design choices: reference counting as primary GC; the Global Interpreter Lock (GIL) simplifies ref-count correctness; bytecode is a public, stable interface; the specialising adaptive interpreter (3.11+) avoids a full JIT.

### Lua 5.4

Source: `lvm.c` (register VM interpreter), `lgc.c` (incremental tri-color GC), `lobject.h` (tagged value representation), `lstate.h` (VM state), `ldo.c` (call/return/error handling).

Key design choices: register VM with 256 virtual registers per function; NaN-boxing (TValue as 64-bit); tri-color incremental GC with `luaC_step()` interleaved in allocation; integer and float as distinct types (Lua 5.3+); coroutine-first design (`lua_State` per coroutine).

### Ruby YARV

Source: `vm.c` (YARV bytecode interpreter), `compile.c` (AST → YARV), `gc.c` (incremental generational GC), `vm_exec.c` (dispatch core).

Ruby 3.x JIT options: MJIT (C-based compilation via system C compiler, deprecated in 3.3), RJIT (pure Ruby reimplementation), YJIT (Rust-based, shipping in 3.2+, the recommended JIT). YJIT uses a lazy basic block versioning approach — specialize basic blocks on observed types — rather than full function compilation.

### HotSpot JVM

Source: `src/hotspot/share/interpreter/` (bytecode interpreter, template interpreter), `share/gc/` (GC implementations: Serial, G1, ZGC, Shenandoah, Parallel), `share/oops/` (object layout, OopMap), `share/compiler/` (C1/C2 interface).

Key design choices: tiered compilation (C1 at 2K invocations, C2 at 10K); safepoints via polling; compressed ordinary object pointers (CompressedOops, 32-bit pointer on 64-bit heap ≤ 32 GB); JVM TI (Tool Interface) for debuggers/profilers.

### SpiderMonkey

Source: `js/src/vm/` (interpreter, frames), `js/src/gc/` (generational GC), `js/src/jit/` (Warp JIT compiler).

Key design choices: Warp JIT replaced IonMonkey in Firefox 83; Warp performs type inference and optimization using CacheIR (a bytecode IR for inline caches) as the type feedback mechanism; bailouts from Warp to the baseline interpreter on type mismatches.

### Comparison Table

| VM | Dispatch | GC Strategy | Object Rep. | JIT |
|----|----------|-------------|-------------|-----|
| CPython 3.13 | Direct threading + copy-and-patch | Ref-count + cyclic mark-sweep | Header word + type ptr | Copy-and-patch (tier 1) |
| Lua 5.4 | Switch + computed goto | Tri-color incremental mark-sweep | NaN-boxing (TValue) | LuaJIT (separate) |
| Ruby YARV | Computed goto | Generational mark-sweep | Tagged pointer | YJIT (lazy BBV) |
| HotSpot | Template interpreter | G1 / ZGC generational | Compressed oops + mark word | C1 + C2 tiered |
| SpiderMonkey | Switch | Generational | Tagged NaN-box | Warp (CacheIR-guided) |
| WAVM | N/A (compiled) | Host memory model | Linear memory | LLVM ORC |

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **CPython 3.14 JIT maturation**: The copy-and-patch tier-1 JIT (PEP 744) is being extended with a tier-2 "micro-op" optimizer that translates uop traces to optimized native code via LLVM or a custom backend; CPython 3.14 alpha cycles are refining the uop IR and the specialization pipeline. Follow discourse at [bugs.python.org](https://bugs.python.org) and [faster-cpython/ideas](https://github.com/faster-cpython/ideas).
- **YJIT Ruby 3.5 improvements**: Shopify's YJIT team is adding lazy compilation for blocks/procs, improved type-profiling for splats and keyword arguments, and ARM64 backend enhancements. The 3.5 development branch (tracked at [github.com/ruby/ruby](https://github.com/ruby/ruby)) targets a 5–10% throughput improvement over YJIT 3.4.
- **CPython free-threaded (PEP 703) GC**: The GIL-free CPython 3.13+ branch requires a thread-safe GC; the biased reference counting scheme (`_Py_INCREF_SPECIALIZED`) and per-thread arenas are being hardened; generational GC in free-threaded mode is tracked in [gh-116206](https://github.com/python/cpython/issues/116206).
- **LLVM ORC v2 `RedirectableSymbolManager` stabilization**: The `RedirectableSymbolManager` API landed in LLVM 19 and is being used by Julia's new JIT backend; LLVM 22.x adds atomic trampoline patching on AArch64 using BTI-compatible stubs — see llvm-dev RFC "ORC: AArch64 trampoline pool for indirect stubs."

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **GraalVM Native Image + Truffle AST interpreter convergence**: Oracle's roadmap includes shipping a fully AOT-compiled Truffle AST interpreter that still performs partial-evaluation specialization at load time, collapsing the distinction between AOT native images and JIT VMs. Relevant papers: Würthinger et al., "One VM to Rule Them All" (SPLASH 2013) and the ongoing Graal CE 24.x release notes.
- **WebAssembly GC proposal reaching broad toolchain support**: The Wasm GC proposal (MVP finalized in 2023) will be fully supported in all major browsers and V8/SpiderMonkey by 2027; LLVM will need a `wasm-gc` target backend that maps typed GC references (`anyref`, `structref`, `arrayref`) to LLVM's `gc` IR — an open RFC area as of early 2026. This will enable JVM, Dart, and Kotlin to target Wasm without a separate GC.
- **Concurrent GC in CPython**: A concurrent, parallel mark-sweep replacing the stop-the-world cyclic GC is under investigation in the [PEP 703 long-term roadmap](https://peps.python.org/pep-0703/); the key challenge is making `ob_refcnt` atomic and introducing tri-color write barriers without overhead for single-threaded code.
- **Generational Lua VM and LuaJIT 3.0**: LuaJIT has been effectively unmaintained since 2017; community fork efforts ([LuaJIT-remake](https://github.com/nicowillis/LuaJIT-Remake)) and Mike Pall's announced (but unreleased) LuaJIT 3.0 aim to add a generational GC with precise stack maps and a new code-gen backend targeting LLVM; the timeline is uncertain but the technical design documents are circulating.
- **Safepoint-free GC for latency-sensitive VMs**: Research into "ragnarok"-style GC (Pizlo, "Scalable Garbage Collection with Guaranteed MMU," ACM ISMM 2010 and follow-on work) that eliminates safepoint STW pauses is being evaluated for WebKit JSC and SpiderMonkey; achieving sub-100µs max pause without safepoints requires probabilistic handshake protocols and colored stacks.

### 5-Year Horizon (Long-Term, by ~2031)

- **Verified GC implementations**: Formal verification of GC correctness using Iris/Coq (cf. Sammler et al., "RefinedC," PLDI 2021 and the ongoing "Diaframe" work) is expected to produce production-grade verified collectors for languages that demand safety guarantees; integration with LLVM's `RewriteStatepointsForGC` stackmap generation would require a formally verified stackmap emitter.
- **Hardware-assisted GC tagging**: ARM's Memory Tagging Extension (MTE) and RISC-V pointer masking proposals will let VM runtime libraries tag heap objects at the hardware level, enabling zero-overhead write barriers and precise liveness tracking without compiler instrumentation; LLVM's address sanitizer infrastructure (`llvm/lib/Transforms/Instrumentation/`) will need corresponding MTE-aware stackmap and barrier passes.
- **Unified bytecode IR across VM tiers**: Research projects (cf. BOLT, Graal IR) suggest a single IR that serves as both an interpreter bytecode and an input to the optimizing compiler, eliminating the IR translation step that currently adds latency when a function crosses the interpretation/JIT threshold; LLVM's MLIR Bytecode dialect work (`mlir/lib/Bytecode/`) may provide the foundation for such a unified format.

---

## Chapter Summary

- Stack VMs (JVM, CPython, Wasm) use implicit operand stack; register VMs (Lua 5.4, Dalvik, YARV) encode operand registers explicitly — register VMs need fewer instructions but wider encoding
- Dispatch strategy determines interpreter throughput: switch dispatch is portable; computed goto (direct threading) reduces branch mispredictions by ~15–20%; copy-and-patch JIT (CPython 3.13) generates native stubs without a full compiler
- NaN-boxing (LuaJIT, Lua 5.x) encodes all values including object pointers in an IEEE 754 NaN; tagged pointers (Ruby, SBCL) use low-bit tags; hidden classes (V8 Maps, SpiderMonkey Shapes) cache property layouts for fast attribute access
- GC strategies: reference counting for immediate reclamation + cyclic GC (CPython); tri-color incremental mark-sweep (Lua 5.4); generational with write barriers (JVM G1/ZGC, HotSpot); concurrent marking via SATB or IUC write barriers eliminates most stop-the-world pauses
- Precise GC scanning requires a root map (JVM OopMap, LLVM `RewriteStatepointsForGC` stackmaps); conservative scanning (Boehm GC) works with any native code but cannot compact
- Safepoints bring all threads to a consistent state for GC: polling-based (load from page, mapped non-readable to halt) or signal-based
- Method dispatch: vtable for monomorphic; polymorphic inline cache (PIC) for 2–8 observed types; megamorphic fallback; ICs connect to JIT specialisation (cross-ref Ch221)
- CPython 3.12+ `_PyInterpreterFrame` (C-stack-resident, no heap allocation per call) reduced call overhead ~30%; when JIT frames deoptimize, the `.llvm_stackmaps` section encodes the live value set needed to reconstruct the interpreter frame
- Tiered compilation with LLVM ORC: interpreter collects call counts → threshold triggers IR emission → `IRLayer.add()` → `RedirectableSymbolManager::redirect()` atomically patches dispatch; per-function JIT (HotSpot style) vs tracing JIT (LuaJIT style) is the primary architectural split for the native tier

---

@copyright jreuben11
