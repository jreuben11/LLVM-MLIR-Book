# Chapter 44 — Coroutine Lowering in Clang

*Part VI — Clang Internals: Codegen and ABI*

C++20 coroutines introduce a suspension model that is fundamentally unlike any other C++ construct: a function that can suspend mid-execution, return a handle to the caller, and later be resumed at the suspension point with full access to its original local variables. Implementing this requires Clang to transform ordinary-looking function bodies into state machines that heap-allocate their stack frames, split into multiple executable fragments, and coordinate with a suite of LLVM coroutine intrinsics that are opaque to the optimizer until a dedicated pass suite materializes them. This chapter traces the complete compilation path from C++ source through Sema analysis, AST construction, codegen, and the LLVM coroutine passes that perform the final lowering to machine-ready IR.

---

## 44.1 The C++20 Coroutine Model

A function is a coroutine if its body contains at least one of `co_await`, `co_yield`, or `co_return`. The standard ([dcl.fct.def.coroutine]) specifies behavior through two interacting protocol families.

### 44.1.1 The Promise Protocol

Every coroutine has an associated *promise type*, obtained via `std::coroutine_traits<ReturnType, Params...>::promise_type`. The promise object is allocated as part of the coroutine frame and mediates all observable behavior:

```cpp
struct promise_type {
    ReturnObject get_return_object();   // called before initial_suspend
    Awaitable initial_suspend();        // suspends or runs at entry
    Awaitable final_suspend() noexcept; // suspends or runs at exit
    void return_value(T);               // or return_void()
    void unhandled_exception();         // called in catch block around body
    // optional:
    template<typename U>
    Awaitable yield_value(U&&);         // for co_yield
    template<typename U>
    decltype(auto) await_transform(U&&); // transforms co_await operand
};
```

The return object is constructed first—before the coroutine suspends at `initial_suspend`. This is the value returned to the caller and typically wraps a `std::coroutine_handle<promise_type>`. Heap allocation of the coroutine frame happens before `get_return_object()` is called; if allocation fails and the promise declares `get_return_object_on_allocation_failure()`, a nothrow `operator new` is used and the failure path returns that value without executing the body.

### 44.1.2 The Awaitable Protocol

An *awaitable* object is evaluated at each `co_await` expression through three member functions:

```cpp
struct MyAwaitable {
    bool await_ready();                  // skip suspension if true
    void await_suspend(coroutine_handle<P>); // or bool, or coroutine_handle<Q>
    T    await_resume();                 // result of the co_await expression
};
```

`await_ready()` is checked first; returning `true` means the awaitable is already complete and suspension is skipped. Otherwise `await_suspend` is called with the current coroutine's handle. The three return-type variants each affect control flow differently:

- **void**: control transfers to the caller/resumer unconditionally
- **bool**: `true` suspends, `false` immediately resumes the current coroutine
- **coroutine_handle<Q>**: transfers control to the target coroutine (symmetric transfer, discussed in §44.11)

`std::suspend_always` returns `false` from `await_ready` and `void` from `await_suspend`—always suspending. `std::suspend_never` returns `true` from `await_ready`—never suspending and never calling `await_suspend`.

### 44.1.3 The Coroutine Handle

`std::coroutine_handle<P>` is a thin wrapper around a `void*` pointing to the coroutine frame. The frame's first two words are function pointers to the resume function and the destroy function, so `handle.resume()` and `handle.destroy()` dereference those pointers and call them. `handle.done()` checks whether the resume pointer is null (set at `final_suspend`). `std::coroutine_handle<void>` (or `coroutine_handle<>`) is the type-erased variant. `std::noop_coroutine()` returns a handle to a statically allocated coroutine that does nothing when resumed.

The coroutine frame layout—established by `CoroSplitPass`—is:

```
struct coroutine_frame {
    void (*resume_fn)(frame*);   // offset 0 — set to null at final_suspend
    void (*destroy_fn)(frame*);  // offset 8
    promise_type promise;        // offset determined by alignment
    uint<N> index;               // suspension-point index (for resume switch)
    // spilled locals follow
};
```

`std::coroutine_handle<P>::from_promise(p)` computes `frame* = (char*)&p - offsetof(frame, promise)` using `llvm.coro.promise` semantics.

---

## 44.2 Clang's Coroutine AST Nodes

Clang represents a coroutine body as a specialized statement node that captures all the synthesized sub-expressions that constitute the lowered protocol.

### 44.2.1 CoroutineBodyStmt

Defined in [`clang/AST/StmtCXX.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/StmtCXX.h), `CoroutineBodyStmt` uses trailing objects to store a fixed set of sub-statements enumerated by `SubStmt`:

| Field | SubStmt enum | Content |
|-------|-------------|---------|
| `Body` | `Body` | Original compound statement as written |
| `Promise` | `Promise` | `DeclStmt` declaring the promise variable |
| `InitSuspend` | `InitSuspend` | `co_await promise.initial_suspend()` |
| `FinalSuspend` | `FinalSuspend` | `co_await promise.final_suspend()` |
| `OnException` | `OnException` | Try/catch block wrapping body; calls `unhandled_exception()` |
| `OnFallthrough` | `OnFallthrough` | Implicit `co_return` if body falls off |
| `Allocate` | `Allocate` | Frame allocation expression |
| `Deallocate` | `Deallocate` | Frame deallocation expression |
| `ResultDecl` | `ResultDecl` | Variable holding `get_return_object()` result |
| `ReturnValue` | `ReturnValue` | Call `promise.get_return_object()` |
| `ReturnStmt` | `ReturnStmt` | Thunk return of `ResultDecl` |
| `ReturnStmtOnAllocFailure` | `ReturnStmtOnAllocFailure` | Return on allocation failure |
| Parameter moves | `FirstParamMove+i` | Copy-constructed parameter clones |

The parameter move nodes exist because coroutine parameters are immediately copy-constructed into the frame—the originals live in the caller's stack frame and become invalid on suspension.

`CoroutineBodyStmt::getPromiseDecl()` returns the `VarDecl` for the promise object. `hasDependentPromiseType()` identifies dependent coroutines in templates. Codegen consults `getBody()` for the original statements and the synthesized fields for the surrounding infrastructure.

### 44.2.2 CoroutineSuspendExpr and Its Subclasses

The base class `CoroutineSuspendExpr` stores five sub-expressions indexed by `SubExpr`:

```
SubExpr::Operand  — the expression written after co_await / co_yield
SubExpr::Common   — expression capturing the operand via OpaqueValueExpr
SubExpr::Ready    — awaitable.await_ready() call
SubExpr::Suspend  — awaitable.await_suspend(handle) call
SubExpr::Resume   — awaitable.await_resume() call (type of the whole expr)
```

`getOpaqueValue()` returns the `OpaqueValueExpr` that represents the single-evaluation binding of the awaitable. This ensures the awaitable is evaluated once and the same object is used for all three protocol calls.

`getSuspendReturnType()` inspects the type of `getSuspendExpr()` and classifies it as `SuspendVoid`, `SuspendBool`, or `SuspendHandle` (the last indicated by a `void*` return from the `await_suspend` wrapper). Codegen uses this classification to emit the appropriate three-way branch.

**`CoawaitExpr`** inherits from `CoroutineSuspendExpr` and adds `isImplicit()` to distinguish `co_await p.initial_suspend()` (implicit, synthesized by Sema) from explicit user-written `co_await expr`.

**`CoyieldExpr`** inherits from `CoroutineSuspendExpr` and wraps `co_await promise.yield_value(expr)`. By the time codegen runs, `CoyieldExpr` carries the same five sub-expressions as `CoawaitExpr`—the desugaring has occurred.

**`DependentCoawaitExpr`** is used when the promise type is dependent (inside a template): it stores the operand plus an `UnresolvedLookupExpr` for `operator co_await`. It is resolved to a `CoawaitExpr` during template instantiation.

### 44.2.3 CoreturnStmt

`CoreturnStmt` stores two sub-statements: `Operand` (the expression after `co_return`, or null for `co_return;`) and `PromiseCall` (the call to `promise.return_value(operand)` or `promise.return_void()`). The `isImplicit()` flag marks the synthesized fallthrough co-return.

### 44.2.4 Parameter Copies and Lifetime Safety

The trailing `ParamMoves` array in `CoroutineBodyStmt` encodes move-constructions of each parameter into the coroutine frame. This is a correctness requirement, not an optimization: after the coroutine first suspends, control returns to the caller's frame. The caller's frame holds the original parameters (as local variables or temporaries), and on resumption those locations may no longer be valid. The coroutine body must therefore operate exclusively on copies owned by the heap-allocated frame.

`buildCoroutineParameterMoves()` iterates over each non-reference parameter and synthesizes a `DeclStmt` that move-constructs or copy-constructs a frame-local variable from the original parameter. These move nodes execute before `get_return_object()` and the initial suspend so they are already in the frame when the first suspension occurs.

Reference parameters are a known hazard. If a coroutine takes a `const std::string&` and the caller passes a temporary, the reference is valid during the initial synchronous portion of the coroutine body (before the first suspension) but dangling after. Clang emits no automatic copying for reference parameters—they remain raw references pointing into the caller's frame. The `[[clang::coro_lifetimebound]]` attribute (available in Clang 22) can annotate a type's constructor or function parameter to trigger a lifetime analysis warning when such dangling is likely:

```cpp
// Warning: reference to temporary bound to coroutine parameter
Task<int> process(const std::vector<int>& data) {
    co_await step_one();
    co_return data[0];  // data is dangling if caller passed a temporary
}
```

With `[[clang::coro_lifetimebound]]` applied to the parameter, Clang's `-Wdangling` machinery warns at the call site. Without it, the dangling occurs silently at runtime—a class of bugs unique to coroutines and not present in regular functions.

---

## 44.3 Sema Coroutine Processing

Coroutine semantics analysis is concentrated in `clang/lib/Sema/SemaCoroutine.cpp`. The transformation proceeds incrementally as coroutine keywords are parsed.

### 44.3.1 Marking the Function as a Coroutine

`ActOnCoroutineBodyStart()` is called the first time any of `co_await`, `co_yield`, or `co_return` is parsed inside a function. It:

1. Checks that the enclosing function is not a constructor, destructor, `main`, constexpr function, consteval function, or varargs function—any of these triggers `err_coroutine_invalid_func_context`.
2. Creates the `FnScopeInfo` entry `CoroutineInfo` via `getCurFunction()->setCoroutine()`.
3. Calls `buildCoroutinePromise()` to look up and instantiate the promise type.

### 44.3.2 Promise Type Lookup

`buildCoroutinePromise()` instantiates `std::coroutine_traits<RetType, ParamTypes...>::promise_type`. The traits template lookup proceeds through the standard namespace; if it is missing, `err_implied_coroutine_type_not_found` is issued requiring `<coroutine>` inclusion. For lambda coroutines, the function type used for traits lookup replaces the closure type with its `operator()` types.

The promise variable (`__promise`) is added as a local `VarDecl` inside the coroutine's function scope and initialized with default construction (or the specific form required if `promise_type` has a constructor that accepts the function's parameter types).

### 44.3.3 Building the CoroutineBodyStmt

`CheckCompletedCoroutineBody()` is called when the function body is fully parsed. It invokes `BuildCoroutineBodyStmt()` which:

1. Calls `buildCoroutinePromise()` if not yet done.
2. Builds `InitSuspend`: synthesizes `co_await promise.initial_suspend()` as a `CoawaitExpr` with `isImplicit=true`.
3. Builds `FinalSuspend`: synthesizes `co_await promise.final_suspend()` and validates the `noexcept` requirement. `err_coroutine_promise_final_suspend_requires_nothrow` is issued if the expression can throw.
4. Constructs the exception handler: wraps the body in a try block with a `catch(...)` that calls `promise.unhandled_exception()`.
5. Constructs `OnFallthrough`: an implicit `co_return;` at end-of-body.
6. Builds the allocation: checks for `promise_type::operator new` overloads; if the promise declares `get_return_object_on_allocation_failure()`, uses `::operator new(size, nothrow)` or `::operator new(size, align_val_t, nothrow)` (with `-fcoro-aligned-allocation`).
7. Synthesizes parameter move nodes for each parameter that is used in the body.
8. Creates the `CoroutineBodyStmt` via `CoroutineBodyStmt::Create()`.

### 44.3.4 await_transform Support

When Sema builds a `CoawaitExpr` for a `co_await expr`, it first checks whether `promise_type` has a member `await_transform`. If so, it rewrites `co_await expr` to `co_await promise.await_transform(expr)`. This is the mechanism by which task frameworks intercept `co_await` and impose scheduling policy—for example, ensuring an awaitable is always scheduled on a particular executor.

The `operator co_await` lookup is also performed at this point: if the expression (after `await_transform`) provides an `operator co_await`, that is called to produce the actual awaitable object.

A concrete example demonstrates how `await_transform` can impose scheduling policy. A typical task runtime uses it to intercept every `co_await` and wrap the operand in a scheduler-aware awaitable:

```cpp
template<typename T>
struct Task {
    struct promise_type {
        Task get_return_object();
        std::suspend_always initial_suspend();
        std::suspend_always final_suspend() noexcept;
        void return_value(T);
        void unhandled_exception();

        // Intercept every co_await to ensure execution resumes on the thread pool
        template<typename Awaitable>
        auto await_transform(Awaitable&& a) {
            return ScheduledAwaitable<Awaitable>{
                std::forward<Awaitable>(a), current_executor()};
        }
    };
    // ...
};

Task<int> fetch_data() {
    auto result = co_await http_get("http://example.com");
    co_return process(result);
}
```

When Sema processes `co_await http_get(...)`, it rewrites the operand to `promise.await_transform(http_get(...))`, which returns a `ScheduledAwaitable<HttpFuture>`. Sema then applies the awaitable protocol to that wrapper type. The final `CoawaitExpr` has `Common` bound to the result of `await_transform`, `Ready` bound to `ScheduledAwaitable::await_ready()`, `Suspend` bound to `ScheduledAwaitable::await_suspend()`, and `Resume` bound to `ScheduledAwaitable::await_resume()`. The original `http_get` result is invisible to codegen—only the transformed awaitable's protocol methods are emitted.

### 44.3.5 The OpaqueValueExpr Binding

The `Common` sub-expression in `CoroutineSuspendExpr` evaluates the awaitable exactly once and binds the result to an `OpaqueValueExpr`. This is essential because `await_ready`, `await_suspend`, and `await_resume` must all operate on the *same* awaitable object, but the expression producing the awaitable might have side effects or produce an rvalue.

```cpp
// This must evaluate http_get() exactly once:
auto result = co_await http_get("...");
// Lowers to:
// %common = call @http_get("...")         ; OpaqueValueExpr binding
// %ready  = call @await_ready(%common)   ; ReadyExpr uses %common
// if (!%ready):
//   call @await_suspend(%common, handle)  ; SuspendExpr uses %common
// %result = call @await_resume(%common)  ; ResumeExpr uses %common
```

In the AST, `getOpaqueValue()` returns the `OpaqueValueExpr` that appears as the `this` pointer (or first argument) of each protocol call. During codegen, `EmitAnyExprToMem` stores the `Common` expression result into a temporary, and the `OpaqueValueExpr` is remapped to that storage. Codegen for all three protocol calls then uses the same temporary address.

---

## 44.4 Codegen Entry: EmitCoroutineBody()

The entry point in `clang/lib/CodeGen/CGCoroutine.cpp` is `CodeGenFunction::EmitCoroutineBody()`, which handles the `CoroutineBodyStmt`. The function operates in four distinct phases.

### 44.4.1 Phase 1: Prologue

The prologue establishes the coroutine frame and the return object. The coroutine identity is established with `llvm.coro.id`:

```llvm
%id = call token @llvm.coro.id(i32 <align>, ptr %promise_addr, ptr null, ptr null)
```

The first argument is the alignment requirement for the frame. The second is the address of the promise object within the (not-yet-allocated) frame—at this IR stage, a placeholder alloca is used and CoroSplit replaces it with the actual promise field address. The third and fourth arguments hold cleanup and metadata for the `RetconOnce` ABI; for the standard switch-resume ABI they are null.

Frame allocation follows immediately:

```llvm
%alloc_needed = call i1 @llvm.coro.alloc(token %id)
br i1 %alloc_needed, label %alloc_bb, label %no_alloc

alloc_bb:
  %frame_size = call i64 @llvm.coro.size.i64()
  %raw = call noalias ptr @_Znwm(i64 %frame_size)
  br label %merge

no_alloc:
  br label %merge

merge:
  %mem = phi ptr [null, %entry], [%raw, %alloc_bb]
  %frame = call ptr @llvm.coro.begin(token %id, ptr %mem)
```

`llvm.coro.alloc` returns `false` when CoroElide has already allocated the frame on the caller's stack—in that case no heap allocation occurs. `llvm.coro.begin` marks the point where the frame becomes live; `CoroSplitPass` uses it as the anchor for all frame analysis.

After `llvm.coro.begin`, the promise is initialized (default-constructed or copy-constructed from parameters) and `get_return_object()` is called. The return object is stored in the `ResultDecl` variable—a local that lives in the *caller's* stack frame because it is the actual return value of the ramp function.

### 44.4.2 Phase 2: Initial Suspend

`EmitCoroutineBody()` emits the `InitSuspend` expression (the synthesized `co_await promise.initial_suspend()`). For `std::suspend_always`, this always suspends, returning control to the caller with the return object. For `std::suspend_never`, it falls through into the body.

### 44.4.3 Phase 3: Body with Suspension Points

The original `getBody()` compound statement is emitted. `co_await` expressions within it produce suspension points (§44.5), `co_yield` expressions produce yields (§44.6), and `co_return` statements produce the terminal sequence (§44.7). The entire body is wrapped in a try block that catches exceptions and calls `promise.unhandled_exception()`.

### 44.4.4 Phase 4: Final Suspend

After normal completion (or after the exception handler), the `FinalSuspend` expression is emitted. The final suspend uses `llvm.coro.suspend(token, i1 true)` with the second argument `true` to indicate final suspension. At this point the resume pointer in the frame is set to null by CoroSplit, making `handle.done()` return true. After final suspend, `llvm.coro.free` retrieves the allocation pointer and `operator delete` frees the frame.

```llvm
%free_ptr = call ptr @llvm.coro.free(token %id, ptr %frame)
%need_free = icmp ne ptr %free_ptr, null
br i1 %need_free, label %free_bb, label %end

free_bb:
  %size = call i64 @llvm.coro.size.i64()
  call void @_ZdlPvm(ptr %free_ptr, i64 %size)
  br label %end
```

---

## 44.5 co_await Lowering

`CodeGenFunction::EmitCoawaitExpr()` handles both explicit user-written `co_await expr` and the synthesized initial/final suspends. The lowering always follows the pattern determined by the AST's pre-analyzed `ReadyExpr`, `SuspendExpr`, and `ResumeExpr` fields.

### 44.5.1 The Three-Way Branch

```llvm
; Evaluate the common expression (binds the awaitable into an OpaqueValueExpr)
; Then:
%ready = call i1 @awaitable.await_ready(...)
br i1 %ready, label %resume_bb, label %suspend_bb

suspend_bb:
  %save = call token @llvm.coro.save(ptr null)
  ; Dispatch to await_suspend wrapper:
  call void @func.__await_suspend_wrapper__X(ptr %awaiter, ptr %frame)
  %suspend_result = call i8 @llvm.coro.suspend(token %save, i1 false)
  switch i8 %suspend_result, label %unreachable [
    i8 0, label %resume_bb    ; resumed
    i8 1, label %cleanup_bb   ; destroy path
  ]

resume_bb:
  ; fall-through from ready=true or after resume
  %result = call T @awaitable.await_resume(...)
  ; %result is the value of the co_await expression

cleanup_bb:
  ; coroutine is being destroyed, run destructors
  call void @llvm.coro.end(ptr null, i1 false, token none)
  ret void
```

`llvm.coro.save` captures the current suspension-point index (incremented by CoroSplit), which the resume function uses to dispatch to the correct resume point. `llvm.coro.suspend` returns `i8 0` when the coroutine should resume immediately (e.g., `await_suspend` returned `false`), `i8 1` when the coroutine should take the destroy path (e.g., `handle.destroy()` was called before resumption), and the default switch edge (no match) when the coroutine is genuinely suspending—control exits the function. The `i1 false` second argument distinguishes non-final from final suspends; final suspend uses `i1 true`.

### 44.5.2 End-to-End: count_up Generator Walkthrough

Consider the `count_up(from, to)` generator from §44.6. Tracing from source to post-split IR clarifies exactly what each stage produces.

**Source:**
```cpp
Generator<int> count_up(int from, int to) {
    for (int i = from; i <= to; ++i)
        co_yield i;
}
```

**AST structure** (`CoroutineBodyStmt`):
- `Promise`: `DeclStmt` declaring `__promise` of type `Generator<int>::promise_type`
- `ReturnValue`: `get_return_object()` call → `Generator<int>` wrapper
- `InitSuspend`: `CoawaitExpr` wrapping `__promise.initial_suspend()` → `suspend_always`
- `Body`: the for-loop containing a `CoyieldExpr` for `co_yield i`
- `OnFallthrough`: implicit `CoreturnStmt` → `__promise.return_void()`
- `FinalSuspend`: `CoawaitExpr` wrapping `__promise.final_suspend()` → `suspend_always`
- `ParamMoves[0]`: `from_copy` ← move from `from`; `ParamMoves[1]`: `to_copy` ← move from `to`

**Pre-split IR** (simplified, with allocas before frame is materialized):
```llvm
define void @count_up(sret(%Generator) %ret, i32 %from, i32 %to) {
  %from_copy = alloca i32    ; ParamMove[0]
  %to_copy   = alloca i32    ; ParamMove[1]
  %promise   = alloca %promise_type, align 4
  %i         = alloca i32

  %id    = call token @llvm.coro.id(i32 4, ptr %promise, ...)
  %alloc = call i1   @llvm.coro.alloc(token %id)
  br i1 %alloc, %heap, %noheap
  ...
  %frame = call ptr @llvm.coro.begin(token %id, ptr %mem)

  store i32 %from, ptr %from_copy    ; parameter copy
  store i32 %to,   ptr %to_copy      ; parameter copy

  call void @get_return_object(%promise)     ; builds Generator wrapper
  ; initial_suspend:
  %tok0 = call token @llvm.coro.save(ptr null)
  call void @__await_suspend_wrapper__init(ptr %init_aw, ptr %frame)
  %s0 = call i8 @llvm.coro.suspend(token %tok0, i1 false)
  switch i8 %s0, label %coro_ret [i8 0, label %body  i8 1, label %cleanup]

body:
  store i32 %from_copy, ptr %i
loop:
  %cond = icmp sle i32 %i, %to_copy
  br i1 %cond, loop_body, loop_exit
loop_body:
  ; yield_value(i) stores i into promise.current_value
  call void @yield_value(%promise, %i)
  %tok1 = call token @llvm.coro.save(ptr null)
  call void @__await_suspend_wrapper__yield(ptr %yield_aw, ptr %frame)
  %s1 = call i8 @llvm.coro.suspend(token %tok1, i1 false)
  switch i8 %s1, label %coro_ret [i8 0, label %inc  i8 1, label %cleanup]
inc:
  %inext = add i32 %i, 1
  store i32 %inext, ptr %i
  br label %loop
loop_exit:
  call void @return_void(%promise)
  ; final_suspend:
  %tokN = call token @llvm.coro.save(ptr null)
  call void @__await_suspend_wrapper__final(ptr %final_aw, ptr %frame)
  %sN = call i8 @llvm.coro.suspend(token %tokN, i1 true)
  switch i8 %sN, label %coro_ret [i8 0, label %done  i8 1, label %cleanup]
...
coro_ret:
  call void @llvm.coro.end(ptr null, i1 false, token none)
  ret void
}
```

**Post-CoroSplit (O1)**: the ramp function executes through initial_suspend and returns. `count_up.resume` is:
```llvm
define internal fastcc void @count_up.resume(ptr %frame) {
  %index    = load i2,  ptr gep(%frame, 32)   ; suspension index field
  %from_val = load i32, ptr gep(%frame, 20)   ; from_copy field
  %to_val   = load i32, ptr gep(%frame, 24)   ; to_copy field
  switch i2 %index, label %unreachable [
    i2 0, label %after_init   ; first resume: run the loop
    i2 1, label %after_yield  ; subsequent resumes: increment and loop
  ]

after_init:
  ; check if from > to (empty range)
  %cmp = icmp sgt i32 %from_val, %to_val
  br i1 %cmp, %exit, %first_yield

first_yield:
  %cur = load i32, ptr gep(%frame, 28)  ; i field
  store i32 %from_val, ptr gep(%frame, 16) ; current_value = from
  store i32 %from_val, ptr gep(%frame, 28) ; i = from
  store i2 1, ptr gep(%frame, 32)          ; index = 1 (yield point)
  ret void

after_yield:
  %i = load i32, ptr gep(%frame, 28)
  %limit = load i32, ptr gep(%frame, 24)
  %next = add i32 %i, 1
  %done = icmp sgt i32 %next, %limit
  br i1 %done, %exit, %yield_next

yield_next:
  store i32 %next, ptr gep(%frame, 16)  ; current_value = next
  store i32 %next, ptr gep(%frame, 28)  ; i = next
  ret void

exit:
  store ptr null, ptr %frame   ; resume_fn = null → done()
  ret void
}
```

Each `ret void` in the resume function is a yield point—control returns to whatever code called `handle.resume()`. The caller reads `handle.promise().current_value` to get the yielded value. This pattern makes coroutine iteration a sequence of function calls with no per-iteration heap allocation.

### 44.5.3 The await_suspend Wrapper

Clang generates an `alwaysinline` wrapper function for each `await_suspend` call:

```
define internal void @func.__await_suspend_wrapper__init(ptr %awaiter, ptr %frame) {
  %handle = call ptr @coroutine_handle_from_address(ptr %frame)
  call void @MyAwaitable.await_suspend(ptr %awaiter, ptr %handle)
  ret void
}
```

This indirection is necessary because `await_suspend` takes a typed `coroutine_handle<P>` argument. By the time CoroSplit runs, the frame pointer is concrete, so it inlines these wrappers and eliminates the indirection.

For the `bool`-returning variant, the wrapper returns `i1` and the suspension branch is conditional. For the `coroutine_handle`-returning variant, the wrapper returns `ptr` (the `void*` of `handle.address()`) and symmetric transfer is performed (§44.11).

### 44.5.4 std::suspend_always and std::suspend_never

`std::suspend_always::await_ready()` returns `false` (never skips suspension) and `await_suspend` is a no-op. The IR for `co_await std::suspend_always{}` therefore always reaches the `llvm.coro.suspend` intrinsic. After inlining and optimization, the ready branch is eliminated.

`std::suspend_never::await_ready()` returns `true`, so the branch is immediately taken to `resume_bb` and `llvm.coro.save`/`llvm.coro.suspend` are on dead paths. After DCE, no suspension point remains—the coroutine executes without ever suspending at that point.

---

## 44.6 co_yield Lowering

`co_yield expr` is defined by the standard as `co_await promise.yield_value(expr)`. Sema builds a `CoyieldExpr` whose `Common` sub-expression is the call to `promise.yield_value(expr)`, and whose `Ready`/`Suspend`/`Resume` sub-expressions come from the awaitable returned by `yield_value`.

`CodeGenFunction::EmitCoyieldExpr()` delegates directly to `EmitCoawaitExpr()` on the `CoyieldExpr`'s sub-expressions. The emitted IR is identical in structure to a `co_await`: evaluate `yield_value(expr)` (which stores the yielded value into the promise), call `await_ready()` (returns `false` for `suspend_always`), call `llvm.coro.save`, invoke the `await_suspend` wrapper, call `llvm.coro.suspend`.

For a generator that yields integers, the compiled frame at optimization level O1 already has the loop body split across the ramp and resume functions, with `yield_value` materializing as a store into the `current_value` field of the promise:

```llvm
; In _Z8count_upii.resume:
%current_value_ptr = getelementptr inbounds i8, ptr %frame, i64 16
store i32 %i, ptr %current_value_ptr, align 8  ; promise.current_value = i
store i2 1, ptr %index_ptr, align 8            ; suspend-point index = 1
ret void                                        ; return to caller (yield)
```

The next call to `handle.resume()` loads the index, branches to the resume-point-1 continuation, increments `i`, and repeats.

---

## 44.7 co_return Lowering

`CodeGenFunction::EmitCoreturnStmt()` processes the `CoreturnStmt`. The Sema-built `PromiseCall` already encodes whether to call `return_value(expr)` or `return_void()`, so codegen simply emits that call and then jumps to the final-suspend cleanup block.

```cpp
// co_return expr;  →
promise.return_value(expr);  // emitted via PromiseCall
goto final_cleanup;          // jump to final_suspend emission
```

If the body falls off without a `co_return`, the synthesized `OnFallthrough` node, an implicit `CoreturnStmt` with null operand and a `return_void()` call, handles it.

### 44.7.1 Exception Handling Around the Body

The `OnException` handler wraps the body (including `co_await` expressions) in a try block. The catch clause catches all exceptions and calls `promise.unhandled_exception()`. This ensures that exceptions thrown during the coroutine body or at `await_resume()` are routed to the promise rather than propagating out of the resume function.

```llvm
invoke void @body_sequence(...)
    to label %body_ok unwind label %coro_lpad

coro_lpad:
  %lp = landingpad { ptr, i32 } catch ptr null
  %exn = extractvalue { ptr, i32 } %lp, 0
  call ptr @__cxa_begin_catch(ptr %exn)
  invoke void @Promise.unhandled_exception(ptr %promise)
      to label %catch_ok unwind label %terminate_lpad
  call void @__cxa_end_catch()
  br label %final_suspend
```

The final-suspend expression is outside the exception handler and is required to be `noexcept`. Sema validates this at `BuildCoroutineBodyStmt` time: the entire expression `co_await promise.final_suspend()` is checked for the noexcept property via `checkNoThrow`. If the check fails, `err_coroutine_promise_final_suspend_requires_nothrow` is issued.

---

## 44.8 The Coroutine Frame Before CoroSplit

Before `CoroSplitPass` runs, the coroutine is a single monolithic function with `alloca`s for all live values. The frame is not yet a struct—it is the collection of allocas that survive across suspension points. The IR at this stage is called the *pre-split* form and contains the coro intrinsics as markers:

```llvm
define void @count_up(sret Generator %0, i32 %from, i32 %to) {
  ; parameter copies into frame allocas
  %from_copy = alloca i32
  %to_copy   = alloca i32
  %promise   = alloca promise_type, align 4

  ; coroutine identity and frame allocation
  %id    = call token @llvm.coro.id(i32 4, ptr %promise, ptr null, ptr null)
  %alloc = call i1   @llvm.coro.alloc(token %id)
  br i1 %alloc, %heap_alloc, %skip_alloc
  ...
  %frame = call ptr @llvm.coro.begin(token %id, ptr %mem)

  ; body with suspension points
  %save0 = call token @llvm.coro.save(ptr null)
  call void @__await_suspend_wrapper__init(ptr %init_awaiter, ptr %frame)
  %s0    = call i8   @llvm.coro.suspend(token %save0, i1 false)
  switch i8 %s0, label %unreachable [i8 0, label %resume0  i8 1, label %cleanup]

  ; ... loop body, yield suspend points ...

  ; final suspend
  %saveN = call token @llvm.coro.save(ptr null)
  call void @__await_suspend_wrapper__final(ptr %final_awaiter, ptr %frame)
  %sN    = call i8   @llvm.coro.suspend(token %saveN, i1 true)

  ; frame deallocation
  %free_ptr = call ptr @llvm.coro.free(token %id, ptr %frame)
  call void @operator_delete(ptr %free_ptr, ...)
  call void @llvm.coro.end(ptr null, i1 false, token none)
  ret void
}
```

The `llvm.coro.id` alignment argument (16 in the simple task example, 4 in the integer generator) reflects the maximum alignment requirement of any value that will be spilled to the frame. With `-fcoro-aligned-allocation`, Clang uses `::operator new(size_t, std::align_val_t)` when the required alignment exceeds `__STDCPP_DEFAULT_NEW_ALIGNMENT__`.

---

## 44.9 CoroSplit and Resume/Destroy Functions

`CoroSplitPass` (in `llvm/lib/Transforms/Coroutines/CoroSplit.cpp`) is an CGSCC pass that transforms each coroutine function from its single-function pre-split form into three functions.

### 44.9.1 Pass Infrastructure

The pass implements the `Switch` ABI (for standard C++20 coroutines), `Retcon`/`RetconOnce` ABIs (for Swift-style coroutines), and the `Async` ABI (for Swift async functions). The ABI is determined from the `llvm.coro.id` variant. For C++ coroutines, `llvm.coro.id` selects `coro::ABI::Switch`.

`CoroShape::analyze()` scans the function and collects:
- The single `CoroBeginInst*` (`llvm.coro.begin`)
- All `AnyCoroEndInst*` (`llvm.coro.end`) — normal and unwind variants
- All `AnyCoroSuspendInst*` (`llvm.coro.suspend`) — one per suspension point
- All `CoroAwaitSuspendInst*` — wrappers for `await_suspend`
- `CoroSizeInst*` / `CoroAlignInst*` — for frame sizing
- Symmetric transfers (`SymmetricTransfers`)

`CoroShape::initABI()` materializes the frame struct type. Each `alloca` that has live values across at least one suspension point (identified by `SuspendCrossingInfo`) becomes a field in the frame.

`SuspendCrossingInfo` (declared in `llvm/include/llvm/Transforms/Coroutines/SuspendCrossingInfo.h`) implements the critical liveness analysis. It answers: for each value `v` and each use `u`, is there a path from `v`'s definition to `u` that passes through a `llvm.coro.suspend`? If so, `v` is a *suspend-crossing* value and must be spilled to the frame.

The analysis constructs a block-level dataflow problem. For each basic block `B`, it computes:
- `Kills[B]`: values defined in `B` (their previous frame values are overwritten)
- `Uses[B]`: values used in `B` before being killed
- `Suspends[B]`: whether `B` contains a `llvm.coro.suspend`

A reverse dataflow pass propagates "needs-to-be-live-at-suspend" information upward from each suspend block. For each use of a value `v` that is reachable from a suspend point on some path back to `v`'s definition, `v` is marked as suspend-crossing. The analysis also handles `phi` nodes: a `phi` whose incoming values include definitions from different sides of a suspend is itself suspend-crossing.

In practice, for the `count_up` generator: `from_copy`, `to_copy`, and `i` are all used after both the initial-suspend and yield-suspend, so all three are suspend-crossing. The `await_ready` return value and the `await_suspend` temporaries are *not* suspend-crossing—they are consumed within the same basic block cluster before the `llvm.coro.suspend` call—so they remain as ordinary IR values and are not spilled.

`CoroShape::FrameTy` is the resulting `StructType`. `CoroShape::FrameSize` is its allocation size. These replace `llvm.coro.size.i64()` during materialization. `CoroShape::FrameAlign` replaces `llvm.coro.align.i64()`.

### 44.9.2 Three Outlined Functions

After analysis, `CoroSplitPass` creates three functions:

**Ramp function** (the original function, renamed): executes from entry up to and including the first `llvm.coro.suspend`, then returns the return object to the caller. This is the function invoked when the coroutine is called.

**Resume function** (`funcname.resume`): an internal fastcc function that takes the frame pointer, dispatches on the suspension-point index via a switch, and jumps to the appropriate resume label. The body after each `llvm.coro.suspend(token %save, i1 false)` resumes at the corresponding label.

**Destroy function** (`funcname.destroy`): an internal fastcc function structured similarly to the resume function but taking the *destroy* path at each suspension point—running destructors for live objects at that suspension depth and eventually calling the deallocator.

The resume and destroy function pointers are stored at offsets 0 and 8 of the frame. `CoroEarlyPass` (a module pass that runs first) lowers `llvm.coro.resume` and `llvm.coro.destroy` calls to loads from the frame followed by indirect calls.

### 44.9.3 The Resume Switch

For a generator with two suspension points (initial_suspend and yield):

```llvm
define internal fastcc void @count_up.resume(ptr %frame) {
  %index = load i2, ptr %index_ptr
  switch i2 %index, label %unreachable [
    i2 0, label %resume_initial   ; after initial_suspend
    i2 1, label %resume_yield     ; after co_yield in loop
  ]

resume_initial:
  ; set up loop variable from frame
  br label %loop_head

loop_head:
  ; loop condition check
  br i1 %cond, label %loop_body, label %loop_exit

loop_body:
  %i = load i32, ptr %i_field
  ; call yield_value(i), store current_value
  store i2 1, ptr %index_ptr  ; update suspension index
  store ptr null, ptr %resume_fn_ptr  ; not final, but update
  ret void                     ; suspend = return to caller

resume_yield:
  %i = load i32, ptr %i_field
  %i2 = add i32 %i, 1
  store i32 %i2, ptr %i_field
  br label %loop_head

loop_exit:
  ; co_return / final_suspend
  store ptr null, ptr %resume_fn_ptr  ; mark done
  ret void
}
```

---

## 44.10 Heap Allocation Elision Optimization (HALO)

`CoroElidePass` eliminates the heap allocation of coroutine frames when the coroutine is scoped to the caller. This optimization, described in the Clang/LLVM design documents as HALO (Heap Allocation eLision Optimization), replaces `operator new` with a stack allocation in the caller.

### 44.10.1 Eligibility Conditions

`CoroElidePass` checks three conditions for each call to a coroutine function:

1. **Single `llvm.coro.begin`**: the callee has exactly one coroutine begin point (no multi-phase construction).
2. **Non-escaping frame**: no use of the frame pointer (`llvm.coro.begin` result or its derived `coroutine_handle::address()`) escapes to a memory location that outlives the caller's activation frame. The alias analysis used is conservative: if the handle is stored to memory not provably on the stack or returned through an output parameter, elision is rejected.
3. **Dominated destroy**: every `llvm.coro.destroy` (or `llvm.coro.resume` leading to final-suspend) is post-dominated by the call site.

When all three hold, `CoroElidePass` substitutes an `alloca` in the caller for the frame storage, patches the callee's `llvm.coro.alloc` to return `false`, and replaces `llvm.coro.free` with a no-op.

For the eager coroutine with `suspend_never` (no suspension at all), at O2 the entire coroutine collapses to a direct function call or is inlined away:

```llvm
; Before elision:
define void @eager_task() { ... allocate frame, run body, free frame ... }
define i32 @caller() { call void @eager_task(); ret i32 0 }

; After elision and O2 DCE:
define void @eager_task() { ret void }  ; frame elided, body was trivial
define i32 @caller() { ret i32 0 }
```

### 44.10.2 When Elision Fails

Several common patterns defeat `CoroElidePass`:

**Pattern 1 — Handle stored to a class member.** When the `coroutine_handle` returned from a coroutine is stored into a field of a class that outlives the caller's stack frame, the handle escapes:

```cpp
struct Executor {
    std::vector<std::coroutine_handle<>> queue;
    void submit(Task t) {
        queue.push_back(t.handle.release()); // handle escapes to member
    }
};
void f(Executor& ex) {
    auto t = my_coroutine();     // frame cannot be elided
    ex.submit(std::move(t));     // handle stored to ex.queue
}
```

The alias analysis detects that `t.handle` is stored to `ex.queue`, which is heap-allocated and outlives `f`. Elision is rejected.

**Pattern 2 — Handle returned from the caller.** If the caller returns a `Task` (which wraps the handle) to its own caller, the handle escapes by definition:

```cpp
Task outer() {
    auto inner_task = inner();   // inner frame cannot be stack-allocated
    co_await inner_task;         // because inner_task is returned
}
```

The outer coroutine's handle is itself managed by its caller—the handle to the inner coroutine escapes through the outer coroutine's frame.

**Pattern 3 — Handle captured by a lambda or stored in a coroutine frame field.** A lambda that captures the handle by value stores it to an implicit field in the lambda object. If the lambda outlives the call site, the handle escapes:

```cpp
void f() {
    auto task = my_coroutine();
    auto resume_later = [h = task.handle]() { h.resume(); };
    register_callback(std::move(resume_later));  // handle captured → escaped
}
```

In all three cases, the allocation stays on the heap. Programmers who need guaranteed stack allocation should use the `[[clang::coro_elide_safe]]` annotation pattern with explicit frame passing, accepting the responsibility for proving safety themselves.

### 44.10.3 The .noalloc Variant and CoroAnnotationElide

`CoroAnnotationElidePass` provides an alternative mechanism for guaranteed elision. Functions decorated with `[[clang::coro_elide_safe]]` are transformed to call a `.noalloc` variant of the coroutine that accepts the frame pointer as an explicit first argument. The frame is allocated in the caller via `alloca` before the call. This annotation-driven approach is designed for coroutine libraries that can statically guarantee safety—typically because the coroutine handle never outlives the caller's scope.

---

## 44.11 Symmetric Transfer and std::generator

Symmetric transfer, introduced in P0913R1 and merged for C++20, allows `await_suspend` to return a `coroutine_handle<Q>` to the current coroutine's resumer, which then immediately resumes the target coroutine. This achieves O(1) coroutine-to-coroutine transfers without growing the call stack.

### 44.11.1 Musttail Call Generation

When `getSuspendReturnType()` returns `SuspendHandle`, Clang emits an `__await_suspend_wrapper` that returns `void*` (the `handle.address()` of the returned handle). After CoroSplit, the resume/destroy function's dispatch to this wrapper produces a tail call:

```llvm
define internal fastcc void @inner.resume(ptr %frame) {
  %next_handle = call ptr @inner.__await_suspend_wrapper__final(ptr %final_awaiter, ptr %frame)
  ; symmetric transfer: call next coroutine's resume function
  %resume_fn = load ptr, ptr %next_handle   ; resume_fn_ptr from frame[0]
  musttail call fastcc void %resume_fn(ptr %next_handle)
  ret void
}
```

The `musttail` attribute (described in [Chapter 23 — Attributes, Calling Conventions, and the ABI](../part-04-llvm-ir/ch23-attributes-calling-conventions-abi.md)) guarantees that the call does not grow the stack. This is critical for recursive coroutine chains (e.g., deeply nested awaits) where non-tail dispatch would overflow the stack.

`CoroShape::SymmetricTransfers` collects the calls that need `musttail` treatment. CoroSplit marks them as musttail after splitting.

### 44.11.2 std::generator

`std::generator<T>` (C++23, P2502R2) is a pull-based range coroutine. Its `promise_type::yield_value` returns `suspend_always`, making every `co_yield` a suspension. Iteration proceeds by calling `handle.resume()` for each element:

```cpp
std::generator<int> fibonacci() {
    int a = 0, b = 1;
    while (true) {
        co_yield a;
        auto c = a + b;
        a = b;
        b = c;
    }
}

// Caller:
for (int x : fibonacci()) {
    if (x > 100) break;
    use(x);
}
```

The range-for loop calls `handle.resume()` at each iteration. In optimized code, CoroSplit fuses the loop body: after yield, the index is set, the function returns; on the next resume, the index dispatches to the continuation after the yield, which computes the next value and stores it. No intermediate heap traffic occurs beyond the initial frame allocation.

For recursive generators (yielding from sub-generators), symmetric transfer eliminates the per-recursive-call stack frame:

```cpp
std::generator<int> tree_values(Node* n) {
    if (!n) co_return;
    co_yield std::ranges::elements_of(tree_values(n->left));
    co_yield n->value;
    co_yield std::ranges::elements_of(tree_values(n->right));
}
```

`std::ranges::elements_of` uses a promise-level recursive generator protocol that issues symmetric transfer to the inner generator, removing the O(depth) stack requirement.

---

## 44.12 Diagnostics and Restrictions

### 44.12.1 Structural Restrictions

Clang enforces the following at `ActOnCoroutineBodyStart()`:

| Context | Diagnostic |
|---------|-----------|
| Constructor | `err_coroutine_invalid_func_context` (select 0) |
| Destructor | `err_coroutine_invalid_func_context` (select 1) |
| `main()` | `err_coroutine_invalid_func_context` (select 2) |
| Constexpr function | `err_coroutine_invalid_func_context` (select 3) |
| Deduced return type | `err_coroutine_invalid_func_context` (select 4) |
| Varargs | `err_coroutine_invalid_func_context` (select 5) |
| Consteval | `err_coroutine_invalid_func_context` (select 6) |
| Catch handler | `err_coroutine_within_handler` |
| ObjC method | `err_coroutine_objc_method` |
| GNU label address | `err_coro_invalid_addr_of_label` |

### 44.12.2 Promise Type Errors

`err_coroutine_type_missing_specialization` fires when the required `coroutine_traits` specialization is absent (missing `<coroutine>` header or missing user specialization). `err_coroutine_promise_type_incomplete` fires when the promise type is declared but incomplete at the point of use.

`err_coroutine_promise_incompatible_return_functions` fires when `promise_type` declares both `return_value` and `return_void`—the standard permits exactly one. `err_coroutine_promise_unhandled_exception_required` fires when `unhandled_exception()` is absent.

`err_coroutine_promise_get_return_object_on_allocation_failure` fires when `get_return_object_on_allocation_failure` is not a static member. `err_coroutine_promise_new_requires_nothrow` fires when this failure path is active but the selected `operator new` can throw.

### 44.12.3 The [[clang::coro_return_type]] Attribute

Functions whose return type is marked `[[clang::coro_return_type]]` must themselves be coroutines or be annotated `[[clang::coro_wrapper]]`. Any non-coroutine function returning such a type without the wrapper attribute triggers `err_coroutine_return_type`. This attribute pair allows library authors to enforce discipline on coroutine return types—preventing accidental use of `Task` as a plain return value rather than as a launched coroutine.

### 44.12.4 -fcoro-aligned-allocation

This flag (enabled by default in C++17 mode and later when the platform supports `align_val_t`) causes Clang to prefer `operator new(size_t, align_val_t)` for coroutine frame allocation when the frame's alignment requirement exceeds `__STDCPP_DEFAULT_NEW_ALIGNMENT__` (typically 16). The `std::nothrow` variant is used when `get_return_object_on_allocation_failure()` is present; if aligned nothrow allocation is unavailable, `err_coroutine_unfound_nothrow_new` fires. The flag can be disabled with `-fno-coro-aligned-allocation`.

### 44.12.5 Debugging Coroutines

The `__builtin_coro_frame()` builtin returns the current coroutine's frame pointer as a `void*`. This can be passed to LLDB's `frame variable` or used to compute the promise address via `llvm.coro.promise` semantics:

```cpp
// Inside a coroutine:
void* frame = __builtin_coro_frame();
// promise is at frame + promise_offset (ABI-dependent)
```

LLDB 22 includes coroutine-aware pretty printers that, given a `coroutine_handle<P>`, can walk the frame and display the promise and local variable state at each suspension point. The debug information for the coroutine frame uses the same DWARF variable descriptions as regular locals, since CoroSplit preserves the `DIVariable` metadata and attaches it to the appropriate subfields of the frame struct.

---

## 44.13 The Complete Pass Pipeline

The coroutine lowering passes are inserted at specific points in the LLVM optimization pipeline:

| Pass | Level | Purpose |
|------|-------|---------|
| `CoroEarlyPass` | Module, O0 and above | Lower `llvm.coro.resume` / `llvm.coro.destroy` to indirect calls through frame |
| `CoroSplitPass` | CGSCC, O0 and above | Split coroutine into ramp + resume + destroy; build frame struct; insert musttail for symmetric transfer |
| `CoroElidePass` | Function, O1+ | Replace heap allocation with stack allocation when safe |
| `CoroAnnotationElidePass` | CGSCC, O2+ | Call `.noalloc` variants for annotated coroutines |
| `CoroCleanupPass` | Function, O0 and above | Remove residual coro intrinsics after splitting |

At O0, only `CoroEarly` and `CoroSplit` are needed (no elision, no annotation processing). At O1+, `CoroElide` runs after inlining. At O2+, `CoroAnnotationElide` can transform library-annotated sites.

The pipeline order matters: `CoroSplit` must run before inlining of the resume/destroy functions, because inlining the unsplit coroutine would inline the entire state machine as monolithic code. After `CoroSplit`, the resume and destroy functions are small, and inlining them into call sites (e.g., an async runtime's dispatch loop) is beneficial.

---

## Chapter Summary

- **C++20 coroutines** are specified through two protocols: the promise (controlling frame lifetime, return, and exceptions) and the awaitable (controlling suspension readiness and transfer).
- **`CoroutineBodyStmt`** captures all synthesized sub-statements: promise declaration, initial/final suspend, exception handler, fallthrough handler, allocation, deallocation, return value, and parameter moves.
- **`CoroutineSuspendExpr`** (base of `CoawaitExpr` and `CoyieldExpr`) stores five sub-expressions evaluated at suspension: Operand, Common, Ready, Suspend, and Resume, together with an `OpaqueValueExpr` binding.
- **Sema** builds the promise via `std::coroutine_traits`, synthesizes the `CoroutineBodyStmt`, validates all protocol requirements, and applies `await_transform` hooks.
- **`EmitCoroutineBody()`** proceeds in four phases: prologue (frame allocation via `llvm.coro.id`/`llvm.coro.alloc`/`llvm.coro.begin`), initial suspend, body, final suspend.
- **`co_await` lowering** produces a three-way branch on `await_ready`, followed by `llvm.coro.save` + `await_suspend` wrapper + `llvm.coro.suspend` returning destroy/resume/suspend.
- **`co_yield`** is syntactic sugar; `EmitCoyieldExpr` delegates to `EmitCoawaitExpr` after the `yield_value` call.
- **`co_return`** invokes the Sema-built promise call and jumps to the final-suspend cleanup.
- **The coroutine frame** is a struct with resume/destroy function pointers, the promise, a suspension-point index, and all values that live across any suspension.
- **`CoroSplitPass`** splits the pre-split function into ramp, resume, and destroy; uses `SuspendCrossingInfo` to determine which values must be spilled; inserts `musttail` for symmetric transfer.
- **HALO** (`CoroElidePass`) eliminates heap allocation when the frame does not escape and all destroys are dominated.
- **Symmetric transfer** generates `musttail` calls in resume functions, enabling O(1) coroutine chaining without stack growth.
- **Diagnostics** enforce structural restrictions (no coroutines in constructors/destructors/main), promise protocol completeness, and the `[[clang::coro_return_type]]` discipline.

---

*Cross-references:*
- [Chapter 27 — Coroutines and Atomics](../part-04-llvm-ir/ch27-coroutines-and-atomics.md) — LLVM coroutine intrinsic reference and the full intrinsic ABI variants
- [Chapter 36 — The Clang AST in Depth](../part-05-clang-frontend/ch36-clang-ast-in-depth.md) — AST node hierarchy; how `CoroutineBodyStmt` fits in the statement tree
- [Chapter 40 — Lowering Statements and Expressions](../part-06-clang-codegen/ch40-lowering-statements-and-expressions.md) — General expression lowering infrastructure; `OpaqueValueExpr` binding
- [Chapter 41 — Calls, the ABI Boundary, and Builtins](../part-06-clang-codegen/ch41-calls-abi-boundary-builtins.md) — How `coroutine_handle` is passed as a parameter; ABI implications of sret returns for the ramp function

*Reference links:*
- [`clang/include/clang/AST/StmtCXX.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/StmtCXX.h#L318) — `CoroutineBodyStmt`, `CoreturnStmt`
- [`clang/include/clang/AST/ExprCXX.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/ExprCXX.h#L5255) — `CoroutineSuspendExpr`, `CoawaitExpr`, `CoyieldExpr`, `DependentCoawaitExpr`
- [`clang/lib/Sema/SemaCoroutine.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaCoroutine.cpp) — `ActOnCoroutineBodyStart`, `buildCoroutinePromise`, `BuildCoroutineBodyStmt`
- [`clang/lib/CodeGen/CGCoroutine.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGCoroutine.cpp) — `EmitCoroutineBody`, `EmitCoawaitExpr`, `EmitCoyieldExpr`, `EmitCoreturnStmt`
- [`llvm/include/llvm/Transforms/Coroutines/CoroShape.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Transforms/Coroutines/CoroShape.h) — `coro::Shape`, `coro::ABI`
- [`llvm/lib/Transforms/Coroutines/CoroSplit.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Coroutines/CoroSplit.cpp) — `CoroSplitPass::run`, frame materialization, symmetric transfer
- [`llvm/lib/Transforms/Coroutines/CoroElide.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/Coroutines/CoroElide.cpp) — `CoroElidePass::run`, HALO conditions
- [P0057R8](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0057r8.pdf) — Original C++ Coroutines TS proposal
- [P0913R1](https://wg21.link/P0913R1) — Symmetric transfer for coroutines
- [P2502R2](https://wg21.link/P2502R2) — `std::generator` specification


---

@copyright jreuben11
