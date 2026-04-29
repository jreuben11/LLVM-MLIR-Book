# Chapter 27 — Coroutines and Atomics

*Part IV — LLVM IR*

LLVM IR must represent two seemingly unrelated but architecturally analogous problems: the ability for a function to pause mid-execution and resume later, and the ability to issue memory operations whose ordering guarantees survive aggressive reordering by both the compiler and the hardware. Both features require semantic information that plain load/store and call/ret cannot convey. Coroutines demand a way to split a single function body across multiple activation records connected by explicit resume and destroy paths. Atomics demand a way to annotate memory operations with ordering constraints that the backend can map to hardware-specific fence and instruction sequences. LLVM's solutions are the coroutine intrinsics (a family of `llvm.coro.*` tokens, frame allocation markers, and suspend points) and the IR-level `load atomic`, `store atomic`, `cmpxchg`, `atomicrmw`, and `fence` instructions with attached `AtomicOrdering` and `SyncScope::ID` qualifiers.

---

## Part A — Coroutines

### 27.1 Why Coroutines Require Compiler Support

A coroutine is a function that can yield control to its caller and be resumed from the point of suspension. At the machine level this requires three capabilities that C's call-return model cannot express without compiler intervention: saving all values that are live across a suspension point, restoring those values on resume, and exposing a stable opaque handle that the caller can hold between calls.

`setjmp`/`longjmp` can save and restore the program counter and stack pointer, but they provide no safety across C++ destructors (RAII cleanup is bypassed), give no type safety for the saved state, and expose no promise object for communicating results. Cooperative threads (green threads, fibres) are an alternative but impose a minimum stack size of tens of kilobytes per suspended unit and require a scheduler even when the concurrency is purely sequential.

Compiler-supported coroutines avoid both shortfalls. The compiler performs a *split-function transformation*: it identifies every value that is alive across at least one suspension point, allocates a *coroutine frame* on the heap (or, when the frame lifetime is proven to be scoped, on the stack), stores live-across-suspension values into frame fields, and creates independent *resume* and *destroy* funclets. The resume funclet is indexed by the suspension point last executed, entered through a switch on an integer field in the frame, and eventually branches back to one of the suspension points. The destroy funclet runs cleanup code and frees the frame. The function's stack frame is de-allocated at the first suspension and reused for nothing; all persistent state lives in the heap frame.

LLVM represents this transformation via a family of `llvm.coro.*` intrinsics that arrive in IR before CoroSplit has run. The frontend emits them; the coroutine passes lower them.

### 27.2 The LLVM Coroutine Intrinsics

The full intrinsic set provides identity tokens, frame lifetime markers, allocation size queries, suspension points, and accessor operations. The verifier enforces structural constraints: a function bearing `presplitcoroutine` must contain exactly one `llvm.coro.begin` whose first operand is the token returned by `llvm.coro.id` or one of its variants.

#### 27.2.1 Identity and Shape Tokens

**`llvm.coro.id(i32 align, ptr readnone promise, ptr nocapture readonly self, ptr info) → token`**

Establishes the *switch ABI* identity for a coroutine (`coro::ABI::Switch`). The `align` operand is the alignment of the coroutine frame (0 means natural alignment). The `promise` operand points to the alloca that holds the promise object; passing `null` means no promise. The `self` operand is the function itself (used for devirtualisation of sub-function calls). The `info` operand is initially `null`; CoroSplit replaces it with a global constant array of `[resume, destroy, cleanup]` function pointers after splitting. The `CoroIdInst::getInfo()` accessor returns the post-split state.

**`llvm.coro.id.retcon(i64 size, i32 align, ptr storage, ptr prototype, ptr alloc, ptr dealloc) → token`**

Selects `coro::ABI::Retcon`. Each suspension point produces a distinct continuation function (a funclet) rather than a shared resume/destroy pair. The `prototype` is a declaration whose type, attributes, and calling convention are copied to every continuation. `alloc` and `dealloc` are called for frame memory unless the frame fits inline in the `storage` buffer. `AnyCoroIdRetconInst` wraps both the `retcon` and `retcon.once` variants; `getStorageSize()`, `getStorageAlignment()`, `getPrototype()`, `getAllocFunction()`, and `getDeallocFunction()` are the accessors.

**`llvm.coro.id.retcon.once(i64 size, i32 align, ptr storage, ptr prototype, ptr alloc, ptr dealloc) → token`**

Selects `coro::ABI::RetconOnce`. Identical to `retcon` except the coroutine is known to suspend at most once, so the continuation returns `void` and no destroy funclet is needed. Used by async-swift's oldest lowering and by some `co_await`-once idioms.

**`llvm.coro.id.async(i32 size, i32 align, i32 storage_arg_index, ptr async_func_ptr) → token`**

Selects `coro::ABI::Async`. The coroutine frame is allocated as a suffix of an *async context* that the caller provides (argument index `storage_arg_index`). `async_func_ptr` points to a `{ uint32_t context_size; uint32_t relative_fn_ptr; }` global that the Swift runtime populates.

#### 27.2.2 Frame Size and Allocation

**`llvm.coro.size.i32() → i32` / `llvm.coro.size.i64() → i64`**

Returns the size in bytes that must be allocated for the coroutine frame. The actual value is not known until CoroSplit computes the frame layout; before splitting it is an opaque integer that the caller passes to its allocator.

**`llvm.coro.align.i32() → i32` / `llvm.coro.align.i64() → i64`**

Returns the required alignment of the allocated frame block.

**`llvm.coro.alloc(token id) → i1`**

Returns `true` if a heap allocation is required. The alloc/begin pattern:

```llvm
%id = call token @llvm.coro.id(i32 0, ptr null, ptr null, ptr null)
%need = call i1 @llvm.coro.alloc(token %id)
br i1 %need, label %alloc, label %begin

alloc:
  %sz  = call i64 @llvm.coro.size.i64()
  %mem = call ptr @malloc(i64 %sz)
  br label %begin

begin:
  %frame_mem = phi ptr [ null, %entry ], [ %mem, %alloc ]
  %hdl = call ptr @llvm.coro.begin(token %id, ptr %frame_mem)
```

When CoroElide proves that the frame lifetime is scoped (the handle never escapes the caller), it replaces the dynamic allocation with an `alloca` and rewrites `llvm.coro.alloc` to return `false`.

**`llvm.coro.begin(token id, ptr mem) → ptr`**

Marks the official start of the coroutine's execution and returns the *coroutine handle* — a pointer to the frame. All subsequent coroutine intrinsic operations use this pointer. The handle is the value that C++20 `coroutine_handle::address()` returns to user code.

**`llvm.coro.free(token id, ptr hdl) → ptr`**

Returns the pointer that must be freed when destroying the frame, or `null` if CoroElide has eliminated the heap allocation. CoroSplit inserts a call to the appropriate deallocation function based on this return value.

#### 27.2.3 Dynamic Allocas in the Frame

Three intrinsics allow allocas that cannot be statically sized at coroutine creation time to live inside the frame rather than on a temporary stack:

**`llvm.coro.alloca.alloc(i64 size, i32 align) → token`** — allocates `size` bytes with `align` alignment inside the frame, returns an opaque allocation token.

**`llvm.coro.alloca.get(token alloc) → ptr`** — returns the base address of the space allocated by `llvm.coro.alloca.alloc`.

**`llvm.coro.alloca.free(token alloc)`** — marks the space available for reuse (important for coroutines with variable-lifetime sub-objects).

#### 27.2.4 Suspension Points

**`llvm.coro.save(ptr hdl) → token`**

Emits a *save point*: writes the index of the next suspension into the frame's index field. The save must appear immediately before its corresponding `llvm.coro.suspend` call; together they ensure the resume switch reaches the correct point after the caller resumes the coroutine. If `token none` is passed to `llvm.coro.suspend`, the pass inserts an implicit save.

**`llvm.coro.suspend(token save, i1 final) → i8`**

The core suspension intrinsic for the Switch ABI. Returns `0` if the coroutine is being resumed, `1` if it is being destroyed, and `-1` (as an unsigned 255) to fall through to the `suspend` block (which returns the handle to the caller). The `final` flag is `true` for the final suspension point (the point after `co_return`). The pattern:

```llvm
%save = call token @llvm.coro.save(ptr %hdl)
%sw   = call i8 @llvm.coro.suspend(token %save, i1 false)
switch i8 %sw, label %suspend [
  i8 0, label %after_yield
  i8 1, label %cleanup
]
```

**`llvm.coro.suspend.async(i32 storage_arg_no, ptr resume_fn, ptr ctx_project, ptr must_tail_fn) → void*`**

Async-ABI suspension. Rather than returning an i8 dispatch value, the async suspend performs a musttail call to the function specified in the last operand, passing the continuation function returned by `llvm.coro.async.resume`. The storage argument index and context-projection function allow the lowering to patch the async context.

**`llvm.coro.suspend.retcon(values...) → values...`**

Retcon-ABI suspension. The operands are the return values the continuation will return to the caller; the intrinsic's result types are the arguments the continuation will receive when the coroutine is next resumed. This ABI avoids the shared resume/destroy function pointers entirely.

#### 27.2.5 Access and Termination Intrinsics

**`llvm.coro.done(ptr hdl) → i1`** — returns `true` if the coroutine has reached its final suspension point (the frame's index field equals the final-suspension index).

**`llvm.coro.resume(ptr hdl)`** — calls the resume function pointer stored in frame slot 0.

**`llvm.coro.destroy(ptr hdl)`** — calls the destroy function pointer stored in frame slot 1.

**`llvm.coro.promise(ptr hdl, i32 align, i1 from_promise) → ptr`** — translates between a frame pointer and the promise object pointer. When `from_promise` is `false`, navigates from handle to promise; when `true`, navigates from promise pointer back to frame. The `CoroPromiseInst` wrapper exposes `getAlignment()` and `isFromPromise()`.

**`llvm.coro.end(ptr hdl, i1 unwind, token results) → void`**

Marks the terminal point of the coroutine (both in normal and exceptional unwind paths). The `unwind` flag distinguishes cleanup landing-pad uses. The `results` token argument accepts an `llvm.end.results(...)` call for the retcon ABI to carry return values. This intrinsic returns `void`; user code falls through to `unreachable`.

**`llvm.coro.end.async(ptr hdl, i1 unwind, ptr must_tail_fn, ...) → void`**

Async-ABI terminal. The third operand is a function to call as a musttail at the terminal point; remaining operands are its arguments.

**`llvm.coro.subfn.addr(ptr hdl, i8 index) → ptr`**

Loads the resume (index 0) or destroy (index 1) function pointer directly from the frame, used by `llvm.coro.resume` and `llvm.coro.destroy` before those are lowered. `CoroSubFnInst::getIndex()` returns the `ResumeKind` enum: `ResumeIndex = 0`, `DestroyIndex = 1`.

**`llvm.coro.prepare.retcon(ptr fn) → ptr`**

Before CoroSplit runs, callers of a retcon coroutine emit calls through this intrinsic rather than calling the coroutine function directly. `CoroEarlyPass` inserts `llvm.coro.prepare.retcon` wrappers around every call site it identifies as a retcon-ABI coroutine invocation, passing the coroutine function pointer as its sole operand and returning a bitcast-compatible function pointer. This indirection gives `CoroSplitPass` a stable anchor point to find and rewrite every call site when it later splits the coroutine into per-suspension continuations. After `CoroSplitPass` has replaced the call sites with calls to the appropriate continuation functions, `CoroCleanupPass` removes the now-redundant `llvm.coro.prepare.retcon` wrappers.

**`llvm.coro.frame() → ptr`** — within the body of a presplit coroutine, returns the frame pointer alias (the value returned by `llvm.coro.begin`).

**`llvm.coro.await.suspend.void / .bool / .handle`**

Introduced in LLVM 22, these are the C++20 `co_await` protocol intrinsics. They wrap a call to the awaiter's `await_suspend()` method and communicate its return type (void, bool, or handle) to the coroutine lowering machinery. Because `await_suspend` can throw, they are modeled as `CallBase` (`CoroAwaitSuspendInst`) rather than `IntrinsicInst`, so they can appear as `invoke` targets. They carry three operands: the awaiter object, the frame pointer, and the wrapper function containing the actual `await_suspend` body.

### 27.3 The Coroutine Frame

When CoroSplit runs on a `presplitcoroutine` function, it constructs a `StructType` named `<functionname>.Frame` to represent the in-memory layout of a suspended coroutine. The structure's fields are determined by `CoroFrameBuilder` inside `llvm/lib/Transforms/Coroutines/CoroFrame.cpp`.

For the Switch ABI, `CoroShape::SwitchFieldIndex` defines two mandatory leading fields:

```
%MyFunc.Frame = type {
  ptr,        ; field 0: resume function pointer   (SwitchFieldIndex::Resume = 0)
  ptr,        ; field 1: destroy function pointer  (SwitchFieldIndex::Destroy = 1)
  <promise>,  ; field 2: promise object (if not null in coro.id)
  <index>,    ; field N: suspension index (i8, i16, or i32 depending on count)
  ...         ; spilled live values, one field per spilled alloca/SSA value
}
```

The resume and destroy function pointers at offsets 0 and 8 (on LP64 systems) allow `llvm.coro.resume` and `llvm.coro.destroy` to dereference the handle directly without knowing the concrete function type. This is the same slot structure that C++ `coroutine_handle<void>` and `coroutine_handle<P>` rely on: casting between specializations is valid precisely because the pointer fields are at fixed offsets.

CoroFrameBuilder determines what to spill by computing liveness across the entire function annotated with suspension points. A value must be spilled if it is defined in a basic block that dominates some suspension point and used in a basic block that is reachable from the same or a later suspension point. Stack-allocated values (`alloca`s) are also promoted into frame fields unless they bear the `!coro.outside.frame` metadata, which Clang attaches to allocas that are provably not live across any suspension (for example, loop induction variables whose scope ends before the yield).

The `CoroShape::FrameSize` field (set by the frame builder) becomes the value returned by `llvm.coro.size.*`. The `FrameAlign` field drives `llvm.coro.align.*`.

### 27.4 The Coroutine Transformation Passes

The four mandatory coroutine passes are registered by `llvm/lib/Transforms/Coroutines/CoroPassManager.cpp` and must appear in a specific order in the pipeline.

#### CoroEarlyPass (Module pass)

Runs before inlining and the main optimisation pipeline. For every coroutine it:
- validates structural constraints on the intrinsics;
- replaces `llvm.coro.resume` and `llvm.coro.destroy` with indirect calls through `llvm.coro.subfn.addr` (so inlining can devirtualise them later);
- and lowers `llvm.coro.promise` when the promise pointer is statically computable.

#### CoroSplitPass (CGSCC pass — `LazyCallGraph::SCC`)

The primary transformation pass. Operates over SCCs of the call graph, which allows it to handle recursive coroutines and mutual recursion.

For the Switch ABI it:
1. Builds the frame type and rewrites all uses of `alloca`s and SSA values that need spilling into GEP-based frame accesses.
2. Outlines the *ramp function* (the code from entry to first suspension), the *resume funclet* (`<name>.resume`), the *destroy funclet* (`<name>.destroy`), and the *cleanup funclet* (`<name>.cleanup`) as separate functions. Resume and destroy share a `switch` on the index field as their entry.
3. Inserts the resume-switch: the funclet entry loads the index field and dispatches to the basic block that corresponds to the suspension that was last saved.
4. Fills the frame's resume and destroy slots with the addresses of the new funclets.
5. Updates the `coro.id` info argument to a post-split `ConstantArray` of `[resume, destroy, cleanup]`.

For the Retcon and RetconOnce ABIs, CoroSplit instead outlines one continuation per suspension point; each continuation becomes its own function and there is no shared dispatch switch.

For the Async ABI, CoroSplit uses `musttail` calls to chain continuations via the Swift calling convention, and the frame is allocated as a suffix of the async context argument rather than as a separate heap allocation.

`CoroSplitPass::OptimizeFrame` (set true above O0) enables an additional pass that tries to reduce the frame size by proving that some spilled values are single-use and can be rematerialised on resume rather than stored.

#### CoroElidePass (Function pass)

Attempts to eliminate the heap allocation for coroutines whose handle does not escape the allocating function. Two paths:

1. **Lifetime analysis**: if the coroutine handle flows only to `llvm.coro.resume`, `llvm.coro.destroy`, and `llvm.coro.done` calls all within the same function, and the function's lifetime never exceeds its caller's, CoroElide replaces the `malloc`/`free` pair with an `alloca` and patches `llvm.coro.alloc` to return `false`.
2. **Annotation elision** (`CoroAnnotationElidePass`, CGSCC): if the callee has the `coro_elide_safe` attribute, Clang emits a `.noalloc` variant of the coroutine ramp whose first parameter accepts a caller-allocated buffer. CoroAnnotationElidePass replaces calls to the normal ramp with calls to the `.noalloc` variant and allocates the frame inline in the caller's own frame. This is how C++20 coroutine elision under `-O2` avoids all heap traffic when the compiler can prove the coroutine lifetime is properly nested.

#### CoroCleanupPass (Module pass)

Runs last in the pipeline. Lowers any remaining coroutine intrinsics that did not get removed by the earlier passes. In particular it removes `llvm.coro.subfn.addr` by replacing them with loads from known frame slot offsets, and removes any `llvm.coro.alloca.*` that were not converted.

### 27.5 C++20 `co_await`, `co_yield`, and `co_return` Lowering

Clang translates each C++20 coroutine keyword into a sequence of calls on the promise object plus coroutine intrinsics. The translation lives in `clang/lib/CodeGen/CGCoroutine.cpp`.

**Coroutine function entry**: Clang allocates the promise object as a local `alloca`, calls `promise_type::get_return_object()` to construct the returned object, and emits the `llvm.coro.id` + `llvm.coro.alloc` + `llvm.coro.begin` sequence. The ramp function is the combination of this preamble and all code up to the first suspension.

**`co_await expr`** expands to:
1. Evaluate `operator co_await(expr)` or use `expr` directly to get an awaitable object.
2. Call `awaitable.await_ready()`. If it returns `true`, skip suspension.
3. If suspension needed, call the `llvm.coro.await.suspend.{void,bool,handle}` intrinsic, which wraps `awaitable.await_suspend(coroutine_handle<promise_type>::from_promise(promise))`. The return type variant selects which continuation path runs: void always resumes, bool conditionally resumes, handle performs symmetric transfer to a different coroutine.
4. Emit `llvm.coro.save` + `llvm.coro.suspend(token, false)` to record the suspension index and yield control.
5. After resume: call `awaitable.await_resume()` for the awaited value.

**`co_yield value`** is syntactic sugar for `co_await promise.yield_value(value)`. The `yield_value()` call typically stores `value` into the promise and returns `suspend_always`, which has `await_ready()` returning `false` and `await_suspend()` doing nothing. The generated IR's structure around the co_yield is identical to a co_await that always suspends.

**`co_return value`** calls `promise.return_value(value)` (or `promise.return_void()`), falls through to the final-suspend block, emits `llvm.coro.save` + `llvm.coro.suspend(token, true)` with the `final` flag set, and after that places `llvm.coro.end` followed by `unreachable`.

**The post-split frame for the generator example** (`count_up(int start, int end)` from earlier) produces:

```llvm
%_Z8count_upii.Frame = type {
  ptr,                                       ; [0] resume fn ptr
  ptr,                                       ; [1] destroy fn ptr
  %"struct.Generator::promise_type",         ; [2] promise (holds current_value: i32)
  i32,                                       ; [3] start argument (spilled)
  i32,                                       ; [4] end argument (spilled)
  i32,                                       ; [5] loop variable i (spilled)
  i2,                                        ; [6] suspension index
  %"struct.std::__n4861::suspend_always",    ; [7] initial awaiter
  %"struct.std::__n4861::suspend_always",    ; [8] yield awaiter
  %"struct.std::__n4861::suspend_always"     ; [9] final awaiter
}
```

Fields [3]–[5] are live-across-suspension values: `start` and `end` parameters and the loop counter `i`. The `!coro.outside.frame` metadata on local temporaries instructs CoroFrameBuilder to omit them from the frame, reducing its size.

### 27.6 Swift Async Functions and `llvm.coro.id.async`

Swift's `async`/`await` model maps to the Async coroutine ABI (`coro::ABI::Async`). Instead of a heap-allocated frame associated with a handle, Swift uses an *async context*: a caller-allocated struct whose first fields are the task pointer, the executor pointer, and a relative function pointer to the resume function. The coroutine frame for the LLVM coroutine pass is appended as a suffix of this context.

`llvm.coro.id.async` receives the size and alignment of the *frontend-managed* prefix (the fields the Swift codegen fills before calling the coroutine), the index of the function argument that carries the context, and a pointer to an `async_function_pointer` global. The global contains two 32-bit fields: the total context size (including the LLVM frame suffix) and a relative pointer to the coroutine entry. The Swift runtime reads the total context size from this global when allocating the async context on a task-local heap.

The Async ABI uses `llvm.coro.suspend.async` rather than the switch-based suspend. When a Swift `async` function awaits another, it:
1. Stores a resume continuation pointer (the value from `llvm.coro.async.resume`) into the async context.
2. Issues a `musttail` call through `llvm.coro.suspend.async` to the awaited function, passing the current async context.
3. When the awaited function completes, it reads the resume continuation pointer from the context and issues its own `musttail` to it.

This *continuation-passing style* with musttail calls ensures no stack growth across await boundaries. Actor isolation (ensuring that code after a hop runs on the target actor's executor) is enforced by passing a different executor in the context before the musttail; the Swift runtime switches executors by calling `swift_task_switch` as the must-tail call in a `llvm.coro.suspend.async`.

---

## Part B — Atomics

### 27.7 The C11/C++11 Memory Model in LLVM IR

LLVM IR implements the C11/C++11 multi-copy atomicity model through six orderings defined in `llvm/Support/AtomicOrdering.h`:

| IR name | `AtomicOrdering` enumerator | C++ equivalent | Numeric ID |
|---|---|---|---|
| `unordered` | `Unordered = 1` | (Java volatile) | 1 |
| `monotonic` | `Monotonic = 2` | `memory_order_relaxed` | 2 |
| `acquire` | `Acquire = 4` | `memory_order_acquire` | 4 |
| `release` | `Release = 5` | `memory_order_release` | 5 |
| `acq_rel` | `AcquireRelease = 6` | `memory_order_acq_rel` | 6 |
| `seq_cst` | `SequentiallyConsistent = 7` | `memory_order_seq_cst` | 7 |

Note that 3 is permanently reserved for `consume` (`AtomicOrderingCABI::consume` exists for the C ABI mapping but has no corresponding LLVM IR ordering; `isValidAtomicOrdering` explicitly rejects value 3). Clang lowers `memory_order_consume` to `acquire` in IR.

The ordering determines what the back-end emits:

| Ordering | x86-64 load | x86-64 store | AArch64 load | AArch64 store | RISC-V load | RISC-V store |
|---|---|---|---|---|---|---|
| `unordered`/`monotonic` | `MOV` | `MOV` | `LDR` | `STR` | `LW` | `SW` |
| `acquire` | `MOV` | — | `LDAR` | — | `LW` + `fence r,rw` | — |
| `release` | — | `MOV` | — | `STLR` | — | `fence rw,w` + `SW` |
| `seq_cst` | `MOV`+`MFENCE` | `XCHG` | `LDAR` | `STLR`+`DMB ISH` | `LW`+`fence rw,rw` | `fence rw,rw`+`SW` |

x86 TSO provides acquire/release semantics for plain loads and stores; only `seq_cst` stores need the `XCHG` (or `MOV`+`MFENCE`) sequence. AArch64 uses the Load-Acquire (`LDAR`) and Store-Release (`STLR`) instructions. RISC-V relies on explicit `fence` instructions because neither load nor store has built-in ordering annotations.

`unordered` is below `monotonic` on the ordering lattice. It is the weakest meaningful atomic ordering: the compiler cannot tear the access and the hardware must not produce values out of thin air, but no happens-before edge is established. It is used for Java volatile semantics and for GC root accesses where the GC guarantees its own synchronisation.

### 27.8 `load atomic` and `store atomic`

The syntax for an atomic load is:

```llvm
%v = load atomic [volatile] i32, ptr %p <ordering>, align 4
```

The alignment operand is mandatory for atomic operations and must be at least the natural alignment of the type (for `i32`, at least 4 bytes; for `i64`, at least 8 bytes on most targets). The IR verifier rejects atomic loads and stores whose alignment is less than the type size in bytes.

`volatile` and `atomic` are orthogonal attributes. `volatile` suppresses reordering and elimination by the optimiser but makes no synchronisation promises. `atomic` makes synchronisation promises but allows reordering with non-atomic operations unless the ordering is `seq_cst`. `volatile atomic` combines both: the hardware-level fence implied by the ordering is emitted, and the compiler also treats the access as a side-effecting memory operation.

```llvm
; Plain load — may be reordered with other non-atomic ops
%a = load i32, ptr %p, align 4

; Atomic load — establishes happens-before with a store atomic release on %p
%b = load atomic i32, ptr %p acquire, align 4

; Atomic volatile load — additionally suppresses compiler reordering
%c = load atomic volatile i32, ptr %p seq_cst, align 4

; Unordered — used for GC roots; no synchronisation edge
%d = load atomic i32, ptr %p unordered, align 4
```

The valid orderings for `load atomic` are `unordered`, `monotonic`, `acquire`, and `seq_cst`. `release` and `acq_rel` are not valid on loads (the IR verifier rejects them). For `store atomic`, valid orderings are `unordered`, `monotonic`, `release`, and `seq_cst`.

### 27.9 `cmpxchg`

The compare-and-exchange instruction is the primitive from which lock-free algorithms are built:

```llvm
%result = cmpxchg [weak] ptr %p, i32 %expected, i32 %desired
               <success-ordering> <failure-ordering>
```

The instruction atomically: reads the value at `%p`; if it equals `%expected`, writes `%desired`; returns a `{ T, i1 }` aggregate where the first element is the original value and the second is `true` if the exchange occurred.

```llvm
; Strong cmpxchg — retries until successful
%cx     = cmpxchg ptr %p, i32 %old, i32 %new acq_rel acquire
%loaded = extractvalue { i32, i1 } %cx, 0
%succ   = extractvalue { i32, i1 } %cx, 1
br i1 %succ, label %done, label %retry

; Weak cmpxchg — may fail spuriously; appropriate for use in a loop
%cx2    = cmpxchg weak ptr %p, i64 %exp, i64 %des seq_cst acquire
```

The success ordering must be at least as strong as the failure ordering. The failure ordering cannot be `release` or `acq_rel`. Strong `cmpxchg` always completes the exchange when the comparison succeeds; weak `cmpxchg` may fail even when `*p == expected` (spurious failure), which is acceptable when the instruction is used inside a retry loop.

On x86, both strong and weak `cmpxchg` lower to `LOCK CMPXCHG` (an atomic compare-exchange that is always strong on this architecture). On AArch64, the lowering uses an `ldaxr` (load-acquire-exclusive) / `stlxr` (store-release-exclusive) retry loop for the standard lowering, or a single `CAS` instruction on systems with `feat_lse`. On RISC-V, the lowering uses `lr.{w,d}` (load-reserved) / `sc.{w,d}` (store-conditional) with appropriate fence annotations.

### 27.10 `atomicrmw`

The atomic read-modify-write instruction combines a load, an arithmetic or logical operation, and a store into a single atomic operation:

```llvm
%old = atomicrmw <op> ptr %p, <type> %val <ordering>
```

`%old` is the value read before modification. Supported operations:

| Operation | Description |
|---|---|
| `xchg` | Exchange: write `%val`, return old value |
| `add` / `sub` | Integer addition / subtraction |
| `and` / `nand` / `or` / `xor` | Bitwise operations; `nand` is `~(old & val)` |
| `max` / `min` | Signed maximum / minimum |
| `umax` / `umin` | Unsigned maximum / minimum |
| `fadd` / `fsub` | Floating-point add / subtract |
| `fmax` / `fmin` | Floating-point maximum / minimum (IEEE semantics) |
| `uinc_wrap` | Increment, wrapping to 0 when `old >= val` |
| `udec_wrap` | Decrement, wrapping to `val` when `old == 0 || old > val` |

`uinc_wrap` and `udec_wrap` are used to implement bounded atomic ring-buffer indices without branching. The verifier requires that `fadd`/`fsub`/`fmax`/`fmin` operate only on floating-point types and that integer operations operate on integer types.

The valid orderings are `monotonic`, `acquire`, `release`, `acq_rel`, and `seq_cst`. `unordered` is forbidden on RMW operations.

`volatile atomicrmw` is accepted; it carries the same compiler-barrier semantics as `volatile atomic` loads and stores.

Lowering strategies vary by target and operation:
- x86 provides native `LOCK XADD` (for `add`/`sub`), `LOCK XCHG` (for `xchg`), `LOCK AND`/`OR`/`XOR`, and `LOCK CMPXCHG`-based loops for the rest.
- AArch64 with LSE provides `LDADDA`, `SWPA`, `LDCLRA`, `LDSET`, etc. Without LSE the back-end emits `ldaxr`/`stlxr` loops.
- Operations without a native instruction (such as `fmax` on targets without a floating-point atomic) are lowered by `AtomicExpandPass` to a `cmpxchg` loop.

### 27.11 Fences

`fence` emits a standalone memory barrier that does not carry a memory address:

```llvm
fence acquire
fence release
fence acq_rel
fence seq_cst
fence syncscope("singlethread") seq_cst
```

A `fence acquire` prevents loads and stores after the fence from being reordered before any load preceding it. A `fence release` prevents loads and stores before the fence from being reordered after any store following it. `fence acq_rel` combines both effects. `fence seq_cst` additionally participates in the total order over all `seq_cst` operations in the program.

`fence` is not valid with `unordered` or `monotonic` ordering (the verifier rejects them). A fence without a `syncscope` modifier applies to the system-wide synchronisation scope (`SyncScope::System = 1`).

Hardware mappings:

| IR fence | x86-64 | AArch64 | RISC-V | PowerPC |
|---|---|---|---|---|
| `acquire` | (implicit: loads are acquire on TSO) | `DMB ISHLD` | `fence r,rw` | `lwsync` |
| `release` | (implicit: stores are release on TSO) | `DMB ISH` | `fence rw,w` | `lwsync` |
| `acq_rel` | (implicit on TSO) | `DMB ISH` | `fence rw,rw` | `lwsync` |
| `seq_cst` | `MFENCE` | `DMB ISH` | `fence rw,rw` | `sync` |

### 27.12 SyncScope: Extending Memory Model Scopes

The `syncscope` qualifier specifies the set of threads that a fence or atomic operation synchronises with. Two canonical scopes are predefined in `llvm/IR/LLVMContext.h`:

```cpp
namespace SyncScope {
  using ID = uint8_t;
  enum {
    SingleThread = 0,  // Synchronised with signal handlers in the same thread
    System       = 1   // Synchronised with all concurrently executing threads
  };
}
```

`SyncScope::SingleThread` is the scope used for `fence syncscope("singlethread")`. It is appropriate for single-threaded code where the only reordering hazard is signal handlers or compiler code motion. On x86 it emits a compiler barrier (`""` inline asm) rather than an `MFENCE`.

Additional scopes are registered at context creation time:

```cpp
SyncScope::ID id = ctx.getOrInsertSyncScopeID("warp");
```

GPU targets use this mechanism extensively. NVPTX defines `"warp"`, `"block"` (CTA), `"cluster"`, and `"device"` scopes. A `fence syncscope("block") acq_rel` instruction maps to `membar.cta` on PTX. A `fence syncscope("device") seq_cst` maps to `membar.gl`. Without a scope qualifier, NVPTX defaults to `membar.sys`.

AMDGPU defines `"workgroup"`, `"agent"`, and `"system"` scopes corresponding to the AMD HSA memory scope model.

`AtomicOrdering` and `SyncScope::ID` are independent dimensions. The combination determines both *what* ordering is guaranteed and *which* threads participate in the synchronisation. A target backend queries both when deciding which instruction sequence to emit.

New backends add scope identifiers by overriding `TargetLowering::getLLVMSyncScopeID`:

```cpp
SyncScope::ID MyTargetTLI::getLLVMSyncScopeID(
    const MachineFunction &MF, SyncScope::ID Scope,
    AtomicOrdering Ordering, LLVMContext &Ctx) const {
  // Map hardware-specific scope strings to IDs registered in context
  return Ctx.getOrInsertSyncScopeID("my_scope");
}
```

### 27.13 AtomicExpandPass

`AtomicExpandPass` is a function-level pass that runs early in the backend pipeline, after IR legalisation but before instruction selection. Its purpose is to lower atomic operations that the target cannot handle natively to sequences it can.

The pass queries the target through three virtual hooks:

```cpp
virtual AtomicExpansionKind shouldExpandAtomicLoadInIR(LoadInst *LI) const;
virtual AtomicExpansionKind shouldExpandAtomicStoreInIR(StoreInst *SI) const;
virtual AtomicExpansionKind shouldExpandAtomicCmpXchgInIR(
    AtomicCmpXchgInst *AI) const;
virtual AtomicExpansionKind shouldExpandAtomicRMWInIR(AtomicRMWInst *RMW) const;
```

The return value is one of the `AtomicExpansionKind` enumerators:

| Value | Meaning |
|---|---|
| `None` | Target handles natively; leave the operation in IR |
| `CastToInteger` | Rewrite float atomic to integer atomic of same width |
| `LLSC` | Expand to load-linked / store-conditional loop |
| `LLOnly` | Expand load to load-linked only (with stronger atomicity) |
| `CmpXChg` | Expand RMW to a `cmpxchg` loop |
| `MaskedIntrinsic` | Call a target-specific intrinsic for LL/SC |
| `BitTestIntrinsic` | Use a target-specific intrinsic for bit test operations (x86) |
| `CmpArithIntrinsic` | Use a target-specific intrinsic for compare-arithmetic (x86) |
| `Expand` | Generic expansion into smaller atomic operations |
| `CustomExpand` | Call `TLI.emitExpandAtomicRMW()` / `emitExpandAtomicCmpXchg()` |
| `NotAtomic` | Rewrite to non-atomic (for known-non-preemptible environments) |

When `CmpXChg` is selected for an `atomicrmw`, `expandAtomicRMWToCmpXchg` from `llvm/CodeGen/AtomicExpandUtils.h` rewrites the RMW into the standard LL/SC or CAS-loop pattern:

```llvm
; Before AtomicExpandPass: atomicrmw fmax ptr %p, float %val seq_cst
; After expansion (on targets without native fmax atomic):
entry:
  br label %loop
loop:
  %old    = load atomic i32, ptr %p seq_cst, align 4  ; read as integer
  %old_fp = bitcast i32 %old to float
  %new_fp = call float @llvm.maxnum.f32(float %old_fp, float %val)
  %new    = bitcast float %new_fp to i32
  %cx     = cmpxchg ptr %p, i32 %old, i32 %new seq_cst monotonic
  %succ   = extractvalue { i32, i1 } %cx, 1
  br i1 %succ, label %exit, label %loop
exit:
  ...
```

When no native operation exists and `shouldExpandAtomicRMWInIR` returns `Expand` (or the expansion itself is too wide for the target), AtomicExpandPass emits a `__atomic_fetch_add` or similar `__atomic_*` libcall. This is the path taken by RISC-V for 128-bit atomics and by any target without atomic instructions at all.

**AArch64 OUTLINE_ATOMIC**: AArch64 backends may select `MaskedIntrinsic` for certain orderings, causing the pass to emit a call to an outlined helper such as `__aarch64_cas4_acq_rel` (a 4-byte CAS with acq_rel ordering) that is resolved at link time to either a native LSE instruction or an LL/SC loop, depending on the CPU detected at runtime. The feature flag is controlled by `-moutline-atomics` and is enabled by default on targets where runtime CPU detection is available.

### 27.14 `freeze` and Undefined Behaviour in Atomic Contexts

The `freeze` instruction was added to LLVM IR to address a semantic gap: while poison propagation through arithmetic operations is well-defined in LLVM's undef/poison model, certain patterns in code that interacts with atomic operations can introduce subtly unsound transformations.

```llvm
%v = load atomic i32, ptr %p monotonic, align 4
%fv = freeze i32 %v
; %fv is guaranteed non-poison; %v may still be undef if the bits were torn
```

LLVM distinguishes three levels of definition for a value:
1. **Fully defined**: all bits have specific values observable by the program.
2. **Undef**: the bits are unspecified at this program point; the compiler may choose any value. Undef is *per-use*: different uses of the same undef SSA value may see different bit patterns.
3. **Poison**: a stronger undefined-behaviour marker; any instruction that observes poison produces poison, and eventually this must not propagate to a side-effecting operation (otherwise undefined behaviour).

For atomics, LLVM's model is:
- An `unordered` or `monotonic` load that races with a concurrent store may return *undef*, not poison. The difference matters: a value derived from undef via a condition (`select`, `br`) invokes undefined behaviour only if the derivation *produces* poison (e.g., `udiv` by zero when the divisor is undef), but undef itself does not.
- A `cmpxchg` or `atomicrmw` whose result flows into a branch without `freeze` can be silently miscompiled if the compiler later proves the RMW's address is unreachable. `freeze` breaks this derivation chain.

`volatile atomic` is the safe choice for memory-mapped I/O on embedded targets: it suppresses all compiler reordering and also prevents load/store optimisation from removing the operation. For pure concurrency control (thread synchronisation without MMIO), `atomic` without `volatile` is correct and allows the optimiser to remove redundant loads, hoist stores out of loops, and perform the standard atomic-load/store optimisations defined in the C++11 model.

The IR verifier enforces that `freeze` receives a single operand of any first-class type (including aggregate types) and that the result type matches the operand type. `freeze` is not a memory operation; it carries no ordering attribute. It is an IR-level instruction that becomes a no-op in the back-end because it only affects the optimiser's freedom to propagate undef/poison.

---

## Chapter Summary

- Coroutines require a *split-function transformation* that LLVM represents via `llvm.coro.*` intrinsics emitted by the frontend and lowered by four mandatory passes: CoroEarlyPass (Module), CoroSplitPass (CGSCC), CoroElidePass (Function), and CoroCleanupPass (Module).
- LLVM supports four coroutine ABIs: Switch (C++20 generator/task), Retcon and RetconOnce (per-suspend continuations), and Async (Swift). Each is selected by the identity intrinsic: `llvm.coro.id`, `llvm.coro.id.retcon`, `llvm.coro.id.retcon.once`, or `llvm.coro.id.async`.
- The coroutine frame for the Switch ABI begins with resume and destroy function pointers at fixed offsets 0 and 8, followed by the promise, index field, and spilled live-across-suspension values. Allocas with `!coro.outside.frame` metadata are excluded.
- CoroElide eliminates heap allocation when the frame lifetime is proven scoped; CoroAnnotationElidePass handles the `coro_elide_safe` annotation path introduced in LLVM 22.
- LLVM IR's six atomic orderings (`unordered`, `monotonic`, `acquire`, `release`, `acq_rel`, `seq_cst`) map to `AtomicOrdering` enumerators; `consume` has no usable IR representation.
- `load atomic` and `store atomic` require an explicit `align` operand at least as wide as the natural type alignment; `volatile` and `atomic` are orthogonal.
- `cmpxchg` returns `{ T, i1 }` and accepts separate success and failure orderings; the `weak` variant may fail spuriously.
- `atomicrmw` supports seventeen operations including the floating-point `fadd`/`fsub`/`fmax`/`fmin` and the wrapping-counter `uinc_wrap`/`udec_wrap` variants.
- `fence` and all atomic operations accept a `syncscope` identifier; GPU targets register scopes such as `"warp"`, `"block"`, and `"device"` via `LLVMContext::getOrInsertSyncScopeID`.
- `AtomicExpandPass` uses the `AtomicExpansionKind` dispatch table (`shouldExpandAtomicRMWInIR`, `shouldExpandAtomicCmpXchgInIR`) to lower unsupported operations to CAS loops, LL/SC sequences, or `__atomic_*` libcalls.
- `freeze` breaks poison-propagation chains from atomic loads; `unordered` and `monotonic` atomic loads that race may return `undef`, not poison.

---

**Cross-references**

- [Chapter 19 — Instructions I: Arithmetic and Memory](ch19-instructions-arithmetic-and-memory.md) — `load`, `store`, `volatile`, and the non-atomic memory model
- [Chapter 20 — Instructions II: Control Flow and Aggregates](ch20-instructions-control-flow-and-aggregates.md) — `phi` nodes across suspension-point basic blocks; `extractvalue` on `cmpxchg` results
- [Chapter 24 — Intrinsics](ch24-intrinsics.md) — the full LLVM intrinsic mechanism and `IntrinsicInst` class hierarchy
- [Chapter 44 — Coroutine Lowering in Clang](../part-06-clang-codegen/ch44-coroutine-lowering-in-clang.md) — `CGCoroutine.cpp`, promise-type lookup, `co_await` expression lowering, HALO optimisation
- [Chapter 90 — Register Allocation](../part-14-backend/ch90-register-allocation.md) — impact of `seq_cst` stores and fences on live-range computation

**Reference links**

- [`llvm/Transforms/Coroutines/CoroInstr.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Transforms/Coroutines/CoroInstr.h) — coroutine intrinsic wrapper classes
- [`llvm/Transforms/Coroutines/CoroShape.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Transforms/Coroutines/CoroShape.h) — `coro::Shape`, `coro::ABI`, frame layout fields
- [`llvm/Support/AtomicOrdering.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/AtomicOrdering.h) — `AtomicOrdering` and `AtomicOrderingCABI` enums
- [`llvm/CodeGen/AtomicExpand.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/AtomicExpand.h) — `AtomicExpandPass` declaration
- [`llvm/CodeGen/AtomicExpandUtils.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/CodeGen/AtomicExpandUtils.h) — `expandAtomicRMWToCmpXchg`
- [`llvm/IR/LLVMContext.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/LLVMContext.h) — `SyncScope` namespace, `getOrInsertSyncScopeID`
- [`llvm/lib/Transforms/Coroutines/CoroFrame.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Coroutines/CoroFrame.cpp) — `CoroFrameBuilder` implementation
- [`clang/lib/CodeGen/CGCoroutine.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGCoroutine.cpp) — Clang's coroutine codegen
- Lewandowski, Carruth, Kuperstein, "Coroutines in LLVM" (LLVM Developers' Meeting 2016) — original Switch ABI design
- Azuma, "LLVM Coroutines for Swift" (LLVM Developers' Meeting 2022) — Async ABI and Swift integration


---

@copyright jreuben11
