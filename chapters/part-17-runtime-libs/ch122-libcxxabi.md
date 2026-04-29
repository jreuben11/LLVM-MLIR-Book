# Chapter 122 — libc++abi

*Part XVII — Runtime Libraries*

libc++abi is LLVM's implementation of the Itanium C++ ABI runtime: the layer between libunwind's `_Unwind_*` primitives and the C++ exception model, plus RTTI, thread-safe static initialization, demangling, and `dynamic_cast`. Every C++ exception thrown in an LLVM-based toolchain passes through libc++abi on its way from `throw` to a `catch` clause. This chapter covers the `__cxa_*` ABI, the personality function, type info matching, thread-safe statics, the demangler, and `__dynamic_cast`. Cross-references: [Chapter 26 — Exception Handling](../part-04-llvm-ir/ch26-exception-handling.md) for the LLVM IR model, [Chapter 42 — C++ ABI Lowering: Itanium](../part-06-clang-codegen/ch42-cpp-abi-lowering-itanium.md) for how Clang lowers C++ to these calls, [Chapter 120 — libunwind](ch120-libunwind.md) for the `_Unwind_*` layer underneath.

---

## 22.1 Repository Layout

```
libcxxabi/
├── include/
│   ├── cxxabi.h          # Public __cxa_* API
│   └── __cxxabi_config.h
├── src/
│   ├── cxa_exception.cpp  # throw / catch machinery
│   ├── cxa_personality.cpp # __gxx_personality_v0
│   ├── cxa_default_handlers.cpp
│   ├── cxa_demangle.cpp   # Demangler (also in LLVM Support)
│   ├── cxa_dynamic_cast.cpp
│   ├── cxa_guard.cpp      # Thread-safe statics
│   ├── cxa_handlers.cpp   # terminate/unexpected
│   ├── cxa_vector.cpp     # Operator new[]/delete[] helpers
│   ├── cxa_thread_atexit.cpp
│   └── private_typeinfo.cpp
└── test/
```

---

## 22.2 The __cxa_* Exception ABI

### 22.2.1 Exception Object Layout

The Itanium ABI wraps the user's exception object in a `__cxa_exception` header:

```cpp
// cxa_exception.h (simplified)
struct __cxa_exception {
    // Note: header grows downward from the thrown object pointer
    size_t referenceCount;         // for shared catch-by-value
    const std::type_info *exceptionType;
    void (*exceptionDestructor)(void*);
    std::unexpected_handler unexpectedHandler;
    std::terminate_handler  terminateHandler;
    __cxa_exception        *nextException; // linked list per thread
    int                     handlerCount;
    __cxa_eh_globals       *globals;
    void                   *adjustedPtr;   // for virtual base catch
    _Unwind_Exception       unwindHeader;  // must be last before object
    // User exception object follows immediately
};
```

Source: [cxa_exception.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/libcxxabi/src/cxa_exception.h)

The crucial point is that `unwindHeader` must immediately precede the user object: `_Unwind_RaiseException` takes a pointer to `_Unwind_Exception`, and libunwind/libc++abi agree to cast between `__cxa_exception*` and the offset-adjusted `_Unwind_Exception*`.

### 22.2.2 __cxa_throw

```cpp
// src/cxa_exception.cpp
void __cxa_throw(void *thrownObject,
                 std::type_info *tinfo,
                 void (*dest)(void*)) {
    // Get the __cxa_exception header behind the thrown object
    __cxa_exception *header = fromThrownObjectToHeader(thrownObject);
    header->referenceCount    = 1;
    header->exceptionType     = tinfo;
    header->exceptionDestructor = dest;
    header->unexpectedHandler = std::get_unexpected();
    header->terminateHandler  = std::get_terminate();
    __cxa_get_globals()->uncaughtExceptions++;
    _Unwind_RaiseException(&header->unwindHeader);
    // If we reach here, no handler found: call terminate
    __cxa_begin_catch(&header->unwindHeader);
    std::terminate();
}
```

`_Unwind_RaiseException` does the two-phase walk (see [Chapter 120](ch120-libunwind.md)). Each frame's personality routine is called during both phases.

### 22.2.3 __cxa_begin_catch / __cxa_end_catch

When libunwind lands in a catch handler via `_Unwind_SetIP`, the generated code calls `__cxa_begin_catch` first:

```cpp
void *__cxa_begin_catch(void *exceptionObject) noexcept {
    _Unwind_Exception *unwindHeader = (_Unwind_Exception*)exceptionObject;
    __cxa_exception *header = toHeader(unwindHeader);
    __cxa_eh_globals *globals = __cxa_get_globals();
    header->handlerCount++;
    // Push onto the caught stack (for re-throw support)
    header->nextException = globals->caughtExceptions;
    globals->caughtExceptions = header;
    globals->uncaughtExceptions--;
    return adjustedThrowObject(header);
}
```

`adjustedThrowObject` returns the pointer to the actual user exception object (possibly adjusted for virtual base classes), which the catch variable then aliases.

`__cxa_end_catch` pops the caught exception from the thread-local stack and calls the destructor if no re-throw occurred.

### 22.2.4 __cxa_rethrow

`throw;` inside a catch clause calls `__cxa_rethrow`, which finds the current exception in the thread-local caught stack and calls `_Unwind_Resume_or_Rethrow` to continue unwinding.

---

## 22.3 The Personality Function: __gxx_personality_v0

The personality function is the callback that libunwind invokes per frame during exception unwinding. Its job is to:
1. (Search phase) Determine whether this frame has a handler for the current exception.
2. (Cleanup phase) Execute cleanups and, at the handler frame, transfer control.

### 22.3.1 Language-Specific Data Area (LSDA)

The LSDA is a per-function table embedded in `.gcc_except_table` (ELF) or `__gcc_except_tab` (Mach-O). It encodes the call-site table, action table, and type table:

```
LSDA structure:
  lpstart encoding (1 byte) + lpstart (landing-pad base)
  ttype encoding  (1 byte)  + ttype base (type table base)
  call-site encoding (1 byte)

  Call-site table:
    [cs_start, cs_len, cs_lp, cs_action]*
    ; cs_start: offset of region start from function start
    ; cs_len:   region length
    ; cs_lp:    landing pad offset (0 = no LP, just cleanup)
    ; cs_action: index into action table (0 = cleanup only)

  Action table:
    [action_filter, action_next]*
    ; action_filter > 0: catch type at types[-action_filter]
    ; action_filter < 0: exception spec
    ; action_filter = 0: cleanup

  Type table:
    [type_info_ptr]*  (grows downward from ttype base)
```

### 22.3.2 Personality Function Flow

```cpp
_Unwind_Reason_Code __gxx_personality_v0(
    int version,
    _Unwind_Action actions,
    uint64_t exceptionClass,
    _Unwind_Exception *unwindException,
    _Unwind_Context *context)
{
    bool isSearchPhase  = actions & _UA_SEARCH_PHASE;
    bool isCleanupPhase = actions & _UA_CLEANUP_PHASE;
    bool isHandler      = actions & _UA_HANDLER_FRAME;

    // 1. Get LSDA pointer and current IP
    const uint8_t *lsda = _Unwind_GetLanguageSpecificData(context);
    uintptr_t pc = _Unwind_GetIP(context) - 1; // PC points past call

    // 2. Parse LSDA to find matching call-site entry
    // 3. Walk action table to find matching catch type
    // 4. Type-match using __cxa_exception::exceptionType
    if (found_handler) {
        if (isSearchPhase) return _URC_HANDLER_FOUND;
        // Cleanup phase at handler frame:
        _Unwind_SetIP(context, landing_pad_addr);
        _Unwind_SetGR(context, __builtin_eh_return_data_regno(0),
                     (uintptr_t)unwindException);
        _Unwind_SetGR(context, __builtin_eh_return_data_regno(1),
                     typeIndex);
        return _URC_INSTALL_CONTEXT;
    }
    if (found_cleanup) {
        if (isSearchPhase) return _URC_CONTINUE_UNWIND;
        _Unwind_SetIP(context, cleanup_landing_pad);
        return _URC_INSTALL_CONTEXT;
    }
    return _URC_CONTINUE_UNWIND;
}
```

Source: [cxa_personality.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/libcxxabi/src/cxa_personality.cpp)

### 22.3.3 Type Matching

Type matching must handle inheritance and pointer adjustment. The matching logic calls `__cxa_exception_type->__do_catch(catch_type, &adjustedPtr, ...)` which dispatches through the RTTI virtual dispatch to handle:

- Exact match (trivially true)
- Public base class (via `dynamic_cast` emulation in type info)
- Pointer-to-derived caught by pointer-to-base
- Reference catch (allows catch by base reference)

---

## 22.4 RTTI: Type Info Hierarchy

### 22.4.1 std::type_info and Derived Classes

libc++abi implements a hierarchy of type info classes in `private_typeinfo.h`:

```
__shim_type_info     (base: vtable with __do_catch, __do_upcast)
    __fundamental_type_info   (int, float, etc.)
    __array_type_info
    __function_type_info
    __enum_type_info
    __class_type_info         (non-polymorphic struct/class)
        __si_class_type_info  (single-inheritance class)
        __vmi_class_type_info (multiple/virtual inheritance)
    __pbase_type_info         (pointer base)
        __pointer_type_info   (pointer-to-T)
        __pointer_to_member_type_info
```

The `__do_catch` virtual function implements the type-matching logic for catch clauses. `__do_upcast` supports `dynamic_cast` via `__class_type_info::__do_dyncast`.

### 22.4.2 Emitted RTTI

Clang emits type info objects at link time. For a class `Foo : public Bar`:

```llvm
@_ZTI3Foo = constant { ptr, ptr, ptr }
  { ptr @_ZTVN10__cxxabiv120__si_class_type_infoE + 16,  ; vtable ptr
    ptr @_ZTS3Foo,        ; name string "3Foo"
    ptr @_ZTI3Bar }       ; base class type info
```

The `@_ZTS` string is the mangled name used for RTTI queries. Two type infos are considered equal if their `name()` pointers compare equal — this is why ODR violations across DSOs can cause silent type mismatch in `catch` clauses.

---

## 22.5 Thread-Safe Static Initialization

### 22.5.1 __cxa_guard_acquire / __cxa_guard_release

C++11 requires that local static initialization is thread-safe. Clang emits:

```llvm
define void @foo() {
  %guard = load i8, ptr @_ZGV..., align 8
  %done = icmp ne i8 %guard, 0
  br i1 %done, label %done_label, label %init_label
init_label:
  call i32 @__cxa_guard_acquire(ptr @_ZGV...)
  ; test result: 0 → another thread initialized, 1 → we must initialize
  ; ... initialize static ...
  call void @__cxa_guard_release(ptr @_ZGV...)
  br label %done_label
done_label:
  ; use static
}
```

The guard variable layout (64-bit):

```
bits 63:8  = implementation-defined
bit 7      = initialized flag (set after __cxa_guard_release)
bit 0      = "in progress" flag (set during __cxa_guard_acquire)
```

`__cxa_guard_acquire` uses a mutex (or a futex on Linux) to serialize concurrent initializations and spin-waits if another thread holds the in-progress flag. `__cxa_guard_abort` handles exceptions thrown during initialization.

Source: [cxa_guard.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/libcxxabi/src/cxa_guard.cpp)

### 22.5.2 Guard Variable Size

On 64-bit platforms, guard variables are 64 bits. On 32-bit platforms (ARM, x86), they are 32 bits. The Itanium ABI specifies that only the first byte is used for the initialized flag; libc++abi uses additional bits for the mutex state.

---

## 22.6 Demangling

### 22.6.1 __cxa_demangle

```cpp
// cxxabi.h
char* __cxa_demangle(const char *mangled_name,
                     char *output_buffer,
                     size_t *length,
                     int *status);
```

`status` codes:
- `0`: Success
- `-1`: Memory allocation failure
- `-2`: Invalid mangled name
- `-3`: Null pointer argument

The demangler implementation (`cxa_demangle.cpp`, ~7000 lines) is also exposed as `llvm::demangle()` from LLVM's `Support` library and is used by LLDB, stack symbolizers, and sanitizer reports.

### 22.6.2 Itanium Mangling Grammar Reference

The demangler handles the full Itanium mangling grammar. Key productions:

| Construct | Mangling | Example |
|-----------|----------|---------|
| Top-level name | `_Z<name><type>` | `_Z3fooi` → `foo(int)` |
| Nested name | `N<qualifiers><name>E` | `_ZN3Foo3barEv` → `Foo::bar()` |
| Template instantiation | `I<type-args>E` | `_Z3fooIiEvi` → `foo<int>(int)` |
| Constructor | `C1/C2/C3` | `_ZN3FooC1Ev` → `Foo::Foo()` |
| Destructor | `D0/D1/D2` | `_ZN3FooD1Ev` → `Foo::~Foo()` |
| `const` qualifier | `K` before type | `_Z3fooRKi` → `foo(int const&)` |
| `volatile` qualifier | `V` before type | `_Z3fooRVi` → `foo(int volatile&)` |
| Pointer | `P<type>` | `_Z3fooPi` → `foo(int*)` |
| Reference | `R<type>` | `_Z3fooRi` → `foo(int&)` |
| Rvalue reference | `O<type>` | `_Z3fooOi` → `foo(int&&)` |
| `std::` | `St` | `StKi` → `std::...const int` |
| Substitution | `S<seq_id>_` | Back-references to earlier encodings |

```cpp
// Source: void foo(int, std::string const&)
// Mangled: _Z3fooiRKNSt3__212basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE

char* demangled = __cxa_demangle(
    "_Z3fooiRKNSt3__212basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE",
    nullptr, nullptr, &status);
// Result: "foo(int, std::__2::basic_string<char, ...> const&)"
free(demangled);
```

The `__2` in the demangled output is the inline namespace from libc++; the demangler preserves it so that error messages clearly identify the library version. Sanitizer reports and LLDB use `llvm::demangle()` (which calls the same implementation) to translate mangled names in backtraces.

---

## 22.7 __dynamic_cast

`dynamic_cast<Derived*>(base_ptr)` is lowered by Clang to `__dynamic_cast`:

```cpp
// cxa_dynamic_cast.cpp
void* __dynamic_cast(
    const void *src_ptr,
    const __class_type_info *src_type,
    const __class_type_info *dst_type,
    ptrdiff_t src2dst_offset)
```

The implementation traverses the object's type info hierarchy:

1. If `src2dst_offset >= 0`, it's a hint: try the static offset first.
2. Find the most-derived object using the vtable offset stored at `((void**)vptr)[-2]` (the `offset_to_top` field from the vtable).
3. Call `dst_type->__do_dyncast(src2dst_offset, ...)` to traverse the inheritance DAG.
4. Return the adjusted pointer, or null if the cast is invalid.

Source: [cxa_dynamic_cast.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/libcxxabi/src/cxa_dynamic_cast.cpp)

For diamond inheritance hierarchies, `__vmi_class_type_info::__do_dyncast` must traverse all paths and ensure exactly one unambiguous base subobject of the target type exists.

---

## 22.8 Array new/delete: __cxa_vec_new / __cxa_vec_delete

C++ `new T[n]` with a non-trivial `T` must construct each element and record the element count so that `delete[]` can call each destructor. libc++abi provides:

```cpp
// cxxabi.h
void* __cxa_vec_new(size_t element_count,
                    size_t element_size,
                    size_t padding_size,      // for array cookie
                    void (*constructor)(void*),
                    void (*destructor)(void*));

void  __cxa_vec_delete(void *array_address,
                        size_t element_size,
                        size_t padding_size,
                        void (*destructor)(void*));

// Three-argument variants for exceptions during construction
void* __cxa_vec_new2(size_t element_count,
                      size_t element_size,
                      size_t padding_size,
                      void (*constructor)(void*),
                      void (*destructor)(void*),
                      void* (*alloc)(size_t),
                      void  (*dealloc)(void*));
```

The `padding_size` is typically `sizeof(size_t)` — the "array cookie" prepended to the allocation that stores the element count for `delete[]`. The cookie layout:

```
  allocated memory:
    [size_t element_count]   ← array cookie (padding_size bytes)
    [T element[0]]
    [T element[1]]
    ...
    [T element[n-1]]
    ← ptr returned to user points here
```

`__cxa_vec_new` calls `operator new[]`, writes the cookie, then calls `constructor` for each element in order. If the constructor throws at element `k`, it calls the destructor for elements `0` through `k-1` in reverse order, then rethrows.

---

## 22.9 Foreign Exception Interoperability

The Itanium ABI defines an exception class code: a 64-bit tag in `_Unwind_Exception::exception_class`. For C++ exceptions from libc++abi the class is `'CLNGC++\0'` (8 ASCII bytes). If the personality function encounters an exception with a different class code, it treats it as a "foreign exception" — it will not try to match catch handlers against it, only execute cleanups.

### 22.9.1 __cxa_get_exception_ptr

For code that wants to peek at a foreign exception during cleanup (e.g., `catch (...)` followed by a check), `__cxa_get_exception_ptr` returns the user object pointer:

```cpp
// Catch a C++ exception but also need the pointer during stack walk
catch (...) {
    void *exc_ptr = __cxa_get_exception_ptr(
        __cxa_current_exception_type());
    log_exception(exc_ptr);
    throw; // re-throw
}
```

### 22.9.2 Mixing with Objective-C Exceptions

On Apple platforms, Objective-C exceptions use a different exception class code (`'CLNGC++\0'` vs Objective-C's `'OBJC\0\0\0\0'`). The Apple libc++abi personality function handles both, but LLVM's libc++abi on non-Apple platforms does not include the Objective-C path.

---

## 22.10 __cxa_thread_atexit

C++11 `thread_local` variables with non-trivial destructors require per-thread cleanup. Clang emits a call to `__cxa_thread_atexit` to register the destructor:

```cpp
// Thread-local variable with destructor:
thread_local Foo tls_foo;
// Compiled as:
static Foo __tls_foo_storage; // per-thread via TLS
__cxa_thread_atexit(Foo::~Foo, &__tls_foo_storage, &__dso_handle);
```

The implementation uses `pthread_key_create` with a destructor that runs the chain of registered destructors when the thread exits. On Linux glibc, this maps to `__cxa_thread_atexit_impl` which integrates with the glibc thread-exit path for correct ordering.

---

## 22.11 Virtual Table Special Entries

### 22.11.1 __cxa_pure_virtual

A pure virtual function call — calling a virtual function on a partially-constructed or destroyed object where the function hasn't been overridden — is undefined behavior. libc++abi fills vtable slots for pure virtuals with a pointer to `__cxa_pure_virtual`:

```cpp
// src/cxa_default_handlers.cpp
void __cxa_pure_virtual() {
    // Called via vtable slot for pure virtual function
    // This represents a programming error:
    // calling virtual fn through base class during construction/destruction
    abort_message("Pure virtual function called!");
}
```

Clang emits vtable slots as:
```llvm
@_ZTV3Foo = constant [5 x ptr] [
    ptr null,                         ; offset_to_top
    ptr @_ZTI3Foo,                    ; type_info
    ptr @__cxa_pure_virtual,          ; pure virtual slot
    ptr @_ZN3Foo3barEv,               ; override of bar()
    ptr null                           ; end
]
```

### 22.11.2 __cxa_deleted_virtual

C++11 `= delete` on a virtual function similarly fills the vtable with `__cxa_deleted_virtual`:

```cpp
void __cxa_deleted_virtual() {
    abort_message("Deleted virtual function called!");
}
```

This fires if a subclass somehow inherits a deleted virtual (through an intermediate override), which the standard says is ill-formed but can arise from ODR violations or compiled-out checks.

### 22.11.3 offset_to_top and Virtual Base Adjustment

The vtable layout (Itanium ABI) for a class with virtual base:

```
vtable for Derived:
  offset[0]: offset-to-top (negative for derived)
  offset[1]: ptr to Derived's type_info
  offset[2]: Derived::vfunc1  (most-derived vptr)
  ...
  [virtual base subobject vtable]:
  offset[-2]: vbase_offset (distance from this to virtual base)
  offset[-1]: offset-to-top (for virtual base subobject)
  offset[0]:  ptr to VBase's type_info
  offset[1]:  VBase::vfunc1 (if not overridden)
```

`__dynamic_cast` reads `offset[-2]` (the `vbase_offset` entry) through the vptr to find virtual base subobjects during up-cast and cross-cast operations.

---

## 22.12 Interaction with libunwind

The relationship between libc++abi and libunwind is precisely defined:

```
User C++ code: throw Foo{};
    │
    ▼
__cxa_throw (libc++abi)
    │  Allocates __cxa_exception header
    │  Sets exceptionType, destructor
    │
    ▼
_Unwind_RaiseException (libunwind)
    │  Phase 1: calls personality per frame
    │           (__gxx_personality_v0 in libc++abi)
    │  Phase 2: calls personality to execute cleanups
    │           then calls __gxx_personality_v0 at handler frame
    │           → _URC_INSTALL_CONTEXT
    │
    ▼  (libunwind restores registers, jumps to landing pad)
Landing pad code (generated by Clang)
    │  Calls __cxa_begin_catch
    │  Executes catch body
    │  Calls __cxa_end_catch
    ▼
Execution continues past try-catch
```

libc++abi and libunwind must be built with the same `LIBUNWIND_ENABLE_SHARED` / `LIBCXXABI_USE_LLVM_UNWINDER` settings to guarantee they share the same `_Unwind_*` implementation.

---

## 22.13 Building libc++abi

### 22.10.1 Standard Build

libc++abi is always built together with libunwind and libc++:

```bash
cmake -G Ninja \
  -DLLVM_ENABLE_RUNTIMES="libunwind;libcxxabi;libcxx" \
  -DLIBCXXABI_USE_LLVM_UNWINDER=ON \
  -DLIBCXXABI_ENABLE_SHARED=ON \
  ../llvm
ninja cxxabi
```

`LIBCXXABI_USE_LLVM_UNWINDER=ON` links libc++abi against LLVM libunwind rather than the system libgcc_s.

### 22.10.2 Standalone Testing

```bash
# Run libc++abi tests
ninja check-cxxabi

# Run a specific test
llvm-lit libcxxabi/test/catch_class_01.pass.cpp
```

---

## Chapter Summary

- libc++abi implements the Itanium C++ ABI runtime: `__cxa_throw/begin_catch/end_catch/rethrow`, the `__gxx_personality_v0` personality function, RTTI type info, thread-safe statics, demangling, and `__dynamic_cast`.
- The `__cxa_exception` header is placed immediately before the user exception object in memory, with `_Unwind_Exception` as the final field to maintain the ABI contract with libunwind.
- `__gxx_personality_v0` interprets the per-function `.gcc_except_table` LSDA to determine whether a frame has a handler or cleanup, directing libunwind to install the corresponding landing pad context.
- Type matching is virtual-dispatch through the `__shim_type_info` hierarchy; `__vmi_class_type_info` handles multiple and virtual inheritance for catch and `dynamic_cast`.
- Thread-safe static initialization uses a guard variable with an in-progress flag and a mutex/futex to serialize concurrent initializations, plus `__cxa_guard_abort` for exception safety.
- `__cxa_demangle` exposes the Itanium demangler; the same implementation is used in LLDB and sanitizer symbolizers via `llvm::demangle()`.
- `__dynamic_cast` traverses the most-derived object's inheritance DAG to find an unambiguous target subobject, using the vtable's `offset_to_top` field to find the complete object.
- `LIBCXXABI_USE_LLVM_UNWINDER=ON` ensures libc++abi uses LLVM's libunwind rather than libgcc_s — required for a fully LLVM-based toolchain.
