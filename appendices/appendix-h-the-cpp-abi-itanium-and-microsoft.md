# Appendix H — The C++ ABI: Itanium and Microsoft

*Quick Reference | LLVM 22.1.x / Clang 22*

Quick reference for the two major C++ ABIs supported by Clang. For full treatment see [Chapter 42 — C++ ABI Lowering: Itanium](../chapters/part-06-clang-codegen/ch42-cpp-abi-lowering-itanium.md) and [Chapter 43 — C++ ABI Lowering: Microsoft](../chapters/part-06-clang-codegen/ch43-cpp-abi-lowering-microsoft.md). Inspect mangled names with `c++filt` (Itanium) or `undname` (MSVC).

---

## H.1 Itanium C++ ABI

Used on Linux, macOS (up to Apple clang switch to Apple ABI), FreeBSD, Android, and all GCC-compatible targets. Specification: [Itanium C++ ABI](https://itanium-cxx-abi.github.io/cxx-abi/abi.html). Implemented in Clang at [clang/lib/CodeGen/CGCXXABI.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGCXXABI.cpp) and `ItaniumCXXABI.cpp`.

### Name Mangling Grammar

All mangled names start with `_Z`. Quick demangling: `echo '_ZN3Foo3barEv' | c++filt`.

| Pattern | Example | Demangled |
|---|---|---|
| `_Z<name>v` | `_Z3foov` | `foo()` |
| `_Z<name><type>` | `_Z3fooi` | `foo(int)` |
| `_Z<name>RKi` | `_Z3fooRKi` | `foo(int const&)` |
| `_Z<name>Ri` | `_Z3fooRi` | `foo(int&)` |
| `_Z<name>Oi` | `_Z3fooOi` | `foo(int&&)` |
| `_ZN<qual>E<type>` | `_ZN3Foo3barEv` | `Foo::bar()` |
| `_ZNK<qual>E<type>` | `_ZNK3Foo3barEv` | `Foo::bar() const` |
| `_ZN<qual>E<tmpl><S_>E<type>` | `_ZN3FooIiE3barEv` | `Foo<int>::bar()` |
| `_ZN<N>D0Ev` | `_ZN3FooD0Ev` | `Foo::~Foo()` (deleting destructor) |
| `_ZN<N>D1Ev` | `_ZN3FooD1Ev` | `Foo::~Foo()` (complete destructor) |
| `_ZN<N>D2Ev` | `_ZN3FooD2Ev` | `Foo::~Foo()` (base destructor) |
| `_ZN<N>C1E<type>` | `_ZN3FooC1Ev` | `Foo::Foo()` (complete constructor) |
| `_ZN<N>C2E<type>` | `_ZN3FooC2Ev` | `Foo::Foo()` (base constructor) |
| `_ZTV<N>` | `_ZTV3Foo` | vtable for `Foo` |
| `_ZTI<N>` | `_ZTI3Foo` | typeinfo for `Foo` |
| `_ZTS<N>` | `_ZTS3Foo` | typeinfo name string for `Foo` |
| `_ZTT<N>` | `_ZTT3Foo` | VTT (virtual table table) for `Foo` |
| `_ZTh<offset>_<name>` | | Thunk for covariant return / non-zero offset |
| `_ZTc<coff><voff>_<name>` | | Covariant return thunk |
| `_ZGV<scope><type><name>` | `_ZGV_ZN3FooC1Ev` | Guard variable for static local |
| `_ZZ<func>E<var>` | `_ZZ3fooEvE3bar` | Static local `bar` in `foo()` |
| `_ZN<anon>E` | `_ZN12_GLOBAL__N_1<name>E` | Anonymous namespace |

### Type Encoding in Mangling

| Type | Encoding | | Type | Encoding |
|---|---|---|---|---|
| `void` | `v` | | `signed char` | `a` |
| `bool` | `b` | | `unsigned char` | `h` |
| `char` | `c` | | `wchar_t` | `w` |
| `short` | `s` | | `unsigned short` | `t` |
| `int` | `i` | | `unsigned int` | `j` |
| `long` | `l` | | `unsigned long` | `m` |
| `long long` | `x` | | `unsigned long long` | `y` |
| `__int128` | `n` | | `unsigned __int128` | `o` |
| `float` | `f` | | `double` | `d` |
| `long double` | `e` | | `__float128` | `g` |
| `...` (varargs) | `z` | | pointer | `P<type>` |
| lvalue ref | `R<type>` | | rvalue ref | `O<type>` |
| const | `K<type>` | | volatile | `V<type>` |
| restrict | `r<type>` | | user type | `N<len><name>E` or `<len><name>` |
| template subst | `S_`, `S0_`, `S1_`... | | template arg | `I<type>E` |

### Virtual Table Layout (Single Inheritance)

```
Foo vtable at _ZTV3Foo:
  [0]  offset_to_top     (0 for primary base)
  [1]  typeinfo pointer  → _ZTI3Foo
  [2]  virtual function 0 (e.g. Foo::~Foo() D1)
  [3]  virtual function 1 (e.g. Foo::~Foo() D0)
  [4]  virtual function 2 (e.g. Foo::virtualMethod)
  ...
```

The vptr stored in an object of type `Foo` points to slot [2] (first function pointer). The `offset_to_top` and `typeinfo` are at negative offsets from the vptr.

### Multiple Inheritance vtable Layout

For a class `D` inheriting from `B1` (primary) and `B2` (secondary):

```
D vtable:
  [offset_to_top=0, typeinfo=_ZTID, D::f overrides B1::f, ...]    ← primary (B1)
  [offset_to_top=-sizeof(B1), typeinfo=_ZTID, thunk for B2::g, ...]  ← secondary (B2)
```

Object layout: `[D data][B1 data][B2 data]` — B1 vptr at offset 0, B2 vptr at `sizeof(B1)`.

### Virtual Inheritance and VTT

For virtual base `VB`:
- `_ZTT<D>`: Virtual Table Table — table of vtable pointers used during construction
- Construction vtables: `_ZTC<D><offset>_<VB>` for each virtual base
- At runtime, the vptr is adjusted to point at the correct construction/complete-object vtable

### `__cxa_exception` Layout

When an exception is thrown, `__cxa_throw` allocates:

```cpp
struct __cxa_exception {
    std::type_info *exceptionType;    // typeinfo of thrown type
    void (*exceptionDestructor)(void*);
    std::unexpected_handler unexpectedHandler;
    std::terminate_handler terminateHandler;
    __cxa_exception *nextException;   // linked list
    int handlerCount;                 // number of active handlers
    int handlerSwitchValue;           // used in landing pad dispatch
    const char *actionRecord;         // action table pointer
    const char *languageSpecificData; // LSDA pointer
    void *catchTemp;                  // landing pad address
    void *adjustedPtr;                // adjusted pointer to thrown object
    _Unwind_Exception unwindHeader;   // must be last (for pointer arithmetic)
};
```

The thrown object immediately follows `__cxa_exception` in memory.

---

## H.2 Calling Convention: Itanium / System V AMD64 ABI

### Integer/Pointer Arguments

| Argument order | Register |
|---|---|
| 1st | `rdi` |
| 2nd | `rsi` |
| 3rd | `rdx` |
| 4th | `rcx` |
| 5th | `r8` |
| 6th | `r9` |
| 7th+ | Stack (right to left, 8-byte aligned) |

### Floating-Point Arguments

| Argument order | Register |
|---|---|
| 1st | `xmm0` |
| 2nd–8th | `xmm1`–`xmm7` |
| 9th+ | Stack |

### Return Values

| Type | Register(s) |
|---|---|
| Integer ≤ 64 bits | `rax` |
| Integer 65–128 bits | `rax:rdx` |
| `float`/`double` | `xmm0` |
| `long double` (x87) | `st0` |
| Small structs (≤ 16 bytes) | `rax:rdx` / `xmm0:xmm1` (based on class) |
| Large structs | Hidden first arg `rdi`; callee stores; `rax` = pointer |

### Callee-Saved Registers

`rbp`, `rbx`, `r12`, `r13`, `r14`, `r15` must be preserved by the callee.

### Stack Frame Layout

```
High addresses
┌─────────────────┐
│ arg 7, 8, ...   │  ← caller pushes extra arguments
├─────────────────┤
│ return address  │  ← call instruction pushes this
├─────────────────┤  ← rsp on function entry
│ old rbp         │  ← push %rbp (if frame pointer enabled)
├─────────────────┤  ← rbp (frame pointer)
│ callee-saved    │  ← r12, r13, r14, r15, rbx, rbp
├─────────────────┤
│ local variables │
├─────────────────┤
│ alignment pad   │  ← rsp must be 16-byte aligned at call
├─────────────────┤  ← rsp
│ red zone        │  ← 128 bytes below rsp; signal-safe scratch
└─────────────────┘
Low addresses
```

---

## H.3 Microsoft C++ ABI

Used by MSVC and Clang with `-target x86_64-windows-msvc` or `clang-cl`. Specification: partially documented in [MSVC ABI](https://docs.microsoft.com/en-us/cpp/cpp/calling-conventions). Implemented in [clang/lib/CodeGen/MicrosoftCXXABI.cpp](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/MicrosoftCXXABI.cpp).

### Name Mangling

All mangled names start with `?`. Demangling: `undname.exe` on Windows or `llvm-undname` / `c++filt -n`.

| Pattern | Example | Demangled |
|---|---|---|
| `?<name>@@YA<ret><args>@Z` | `?foo@@YAXXZ` | `void foo()` |
| `?<name>@@YA<ret><arg1><arg2>@Z` | `?foo@@YAHH@Z` | `int foo(int, int)` |
| `?<member>@<class>@@QEA<ret><args>@Z` | `?bar@Foo@@QEAAXXZ` | `void Foo::bar()` |
| `?<member>@<class>@@QEB<ret><args>@Z` | `?bar@Foo@@QEBAXXZ` | `void Foo::bar() const` |
| `??0<class>@@QEA<...>@Z` | `??0Foo@@QEAA@XZ` | `Foo::Foo()` |
| `??1<class>@@QEA<...>@Z` | `??1Foo@@QEAA@XZ` | `Foo::~Foo()` |
| `??_7<class>@@6B@` | `??_7Foo@@6B@` | vtable for `Foo` |
| `??_R0?AV<class>@@@8` | `??_R0?AVFoo@@@8` | typeinfo for `Foo` |
| `??_R1A@?0A@EA@<class>@@8` | | `Foo::'RTTI Base Class Descriptor'` |
| `??_R2<class>@@8` | | RTTI Base Class Array |
| `??_R3<class>@@8` | | RTTI Class Hierarchy Descriptor |
| `??_R4<class>@@6B@` | | RTTI Complete Object Locator |

### MSVC Type Encoding in Mangling

| Type | Encoding |
|---|---|
| `void` | `X` |
| `int` | `H` |
| `unsigned int` | `I` |
| `long` | `J` |
| `unsigned long` | `K` |
| `__int64` | `_J` |
| `unsigned __int64` | `_K` |
| `float` | `M` |
| `double` | `N` |
| `long double` | `O` |
| `bool` | `_N` |
| `char` | `D` |
| `unsigned char` | `E` |
| `wchar_t` | `_W` |
| pointer | `PEA<type>` (64-bit), `PA<type>` (32-bit) |
| lvalue ref | `AEA<type>` |
| rvalue ref | `$$Q<type>` |
| const | adding `B` to base |
| volatile | adding `C` |
| user class | `V<name>@@` |
| function | `P6A<ret><args>@Z` |

### Virtual Table Layout (MSVC)

Unlike Itanium, MSVC vtables do **not** include `offset_to_top` or `typeinfo` at negative offsets. Instead, a separate `RTTI Complete Object Locator` structure holds the type information.

```
Foo vtable at ??_7Foo@@6B@:
  [0]  virtual function 0  (e.g. Foo::f)
  [1]  virtual function 1  (e.g. Foo::~Foo destructor)
  ...

Object layout for Foo:
  [0]  vfptr → Foo's vtable
  [4/8]  data members
```

For virtual inheritance, MSVC adds a `vbptr` (virtual base pointer) alongside the `vfptr`:

```
Object with virtual base:
  [0]  vfptr → primary vtable
  [8]  vbptr → virtual base table
  [16] data members
  ...  (virtual base data at end)
```

### RTTI Structures

```cpp
// MSVC RTTI Complete Object Locator
struct RTTICompleteObjectLocator {
    unsigned signature;    // 0 for x86, 1 for x64 (uses RVAs)
    unsigned offset;       // offset of this vftable in complete object
    unsigned cdOffset;     // offset of constructor displacement
    int pTypeDescriptor;   // RVA of TypeDescriptor
    int pClassHierarchyDescriptor; // RVA of ClassHierarchyDescriptor
    int pSelf;             // RVA of this locator (x64 only)
};

// TypeDescriptor
struct TypeDescriptor {
    void *pVFTable;        // pointer to type_info vtable
    void *spare;
    char name[];           // mangled type name starting with '.'
};
```

### `__declspec` Extensions

| Attribute | Description |
|---|---|
| `__declspec(dllexport)` | Mark symbol for export from DLL; generates `.def` entry |
| `__declspec(dllimport)` | Mark symbol as imported from DLL; generates `__imp_` prefix |
| `__declspec(noinline)` | Prevent inlining |
| `__declspec(noreturn)` | Function does not return |
| `__declspec(restrict)` | Return pointer does not alias |
| `__declspec(align(N))` | Force alignment |
| `__declspec(naked)` | No prologue/epilogue |
| `__declspec(thread)` | Thread-local variable (like `thread_local`) |
| `__declspec(safebuffers)` | Suppress buffer security checks |
| `__declspec(guard(cf))` | Enable Control Flow Guard for this function |

### SEH (Structured Exception Handling)

Windows uses SEH for both C++ exceptions and hardware exceptions on x64.

```cpp
__try {
    // guarded body
} __except (filter_expression) {
    // exception handler
}

__try {
    // guarded body
} __finally {
    // termination handler (always runs)
}
```

**x64 SEH**: Uses `.pdata` (array of `RUNTIME_FUNCTION` records) and `.xdata` (`UNWIND_INFO` structures) for table-based unwinding. No frame registration at runtime (unlike x86 `fs:[0]` chain).

```c
// RUNTIME_FUNCTION entry (in .pdata, 12 bytes)
struct RUNTIME_FUNCTION {
    ULONG BeginAddress;  // RVA of function start
    ULONG EndAddress;    // RVA of function end
    ULONG UnwindInfoAddress; // RVA of UNWIND_INFO
};

// UNWIND_INFO structure (in .xdata)
struct UNWIND_INFO {
    BYTE Version : 3;       // 1 or 2
    BYTE Flags : 5;         // UNW_FLAG_NHANDLER=0, EHANDLER=1, UHANDLER=2, CHAININFO=4
    BYTE SizeOfProlog;
    BYTE CountOfCodes;      // number of UNWIND_CODEs following
    BYTE FrameRegister : 4;
    BYTE FrameOffset : 4;
    UNWIND_CODE UnwindCode[]; // array of unwind codes
    // followed by handler RVA and language-specific data if EHANDLER/UHANDLER
};
```

**Handler personalities**: `__C_specific_handler` (C SEH), `__CxxFrameHandler3`/`__CxxFrameHandler4` (C++ EH), `_except_handler4` (x86 legacy).

---

## H.4 Calling Convention: Microsoft x64

### Integer/Pointer Arguments

Only 4 registers for all integer arguments (unlike System V's 6):

| Argument order | Register |
|---|---|
| 1st | `rcx` |
| 2nd | `rdx` |
| 3rd | `r8` |
| 4th | `r9` |
| 5th+ | Stack |

### Floating-Point Arguments

| Argument order | Register |
|---|---|
| 1st | `xmm0` |
| 2nd | `xmm1` |
| 3rd | `xmm2` |
| 4th | `xmm3` |
| 5th+ | Stack |

Note: Each argument slot corresponds to one argument position regardless of type — `rcx` and `xmm0` are **not** used simultaneously for the same argument slot (unlike System V).

### Return Values

| Type | Register |
|---|---|
| Integer ≤ 64 bits | `rax` |
| `float`/`double` | `xmm0` |
| Struct ≤ 8 bytes | `rax` |
| Struct > 8 bytes | Hidden first arg in `rcx`; `rax` = pointer |

### Shadow Space (Home Space)

The caller **must** allocate 32 bytes (4 × 8) of shadow space on the stack before the call. This space is above the return address and is available for the callee to spill its register arguments.

```
High addresses
┌─────────────────┐
│ arg 5, 6, ...   │  ← extra args beyond 4
├─────────────────┤
│ shadow space    │  ← 32 bytes (home space for rcx/rdx/r8/r9)
├─────────────────┤
│ return address  │
├─────────────────┤  ← rsp after call (must be 16-byte aligned)
│ callee locals   │
└─────────────────┘
```

### Callee-Saved Registers (Microsoft x64)

`rbp`, `rbx`, `rdi`, `rsi`, `r12`, `r13`, `r14`, `r15`, `xmm6`–`xmm15` must be preserved.

Note: Unlike System V, `rdi` and `rsi` are **callee-saved** on Windows.

---

## H.5 C++ ABI in LLVM / Clang

### `TargetCXXABI::Kind` Enumeration

| Kind | Description | Platforms |
|---|---|---|
| `GenericItanium` | Standard Itanium ABI | Linux x86-64, RISC-V, etc. |
| `GenericARM` | ARM Itanium variant | 32-bit ARM Linux |
| `iOS` | Apple iOS 32-bit | iOS (ARM) |
| `iOS64` | Apple iOS 64-bit / Apple Silicon | iOS/iPadOS (ARM64) |
| `WatchOS` | Apple watchOS | watchOS (ARM64) |
| `AppleARM64` | macOS ARM64 | macOS on M1/M2/M3 |
| `Fuchsia` | Fuchsia OS | Fuchsia (ARM64/x86-64) |
| `WebAssembly` | Wasm Itanium variant | WebAssembly |
| `XL` | IBM XL compiler | AIX (Power) |
| `GenericMIPS` | MIPS Itanium variant | MIPS Linux |
| `Microsoft` | MSVC ABI | Windows (x86/x86-64/ARM64) |

### Selecting ABI in Clang

```bash
# Itanium on Linux x86-64
clang -target x86_64-unknown-linux-gnu -c -emit-llvm foo.cpp -o foo.bc

# Apple ABI on macOS ARM64
clang -target arm64-apple-macosx14.0 -c -emit-llvm foo.cpp -o foo.bc

# Microsoft ABI on Windows x64
clang -target x86_64-pc-windows-msvc -c -emit-llvm foo.cpp -o foo.bc

# Cross-compile for Windows from Linux
clang-cl --target=x86_64-windows-msvc -c foo.cpp -o foo.obj
```

### Key ABI-Affecting Clang Flags

| Flag | Effect |
|---|---|
| `-fms-compatibility` | Enable MSVC compatibility mode |
| `-fms-extensions` | Enable MSVC language extensions |
| `-fdelayed-template-parsing` | MSVC-style template parsing |
| `-fexceptions` / `-fno-exceptions` | Enable/disable C++ exceptions |
| `-frtti` / `-fno-rtti` | Enable/disable RTTI (typeinfo) |
| `-fcxx-exceptions` | Enable C++ exception semantics |
| `-fthreadsafe-statics` | Emit guards for function-local static init (default on) |
| `-fvisibility-inlines-hidden` | Hide inline function symbols (breaks RTTI across DSOs if misused) |
| `-fuse-cxa-atexit` / `-fno-use-cxa-atexit` | Use `__cxa_atexit` for static destructor registration |
| `-mframe-pointer=all/non-leaf/none` | Frame pointer retention |
| `-mretpoline` | Retpoline mitigation for indirect calls (Spectre v2) |

---

## H.6 ABI Quick Comparison Table

| Feature | Itanium (System V AMD64) | Microsoft x64 |
|---|---|---|
| Integer arg registers | `rdi rsi rdx rcx r8 r9` (6) | `rcx rdx r8 r9` (4) |
| Float arg registers | `xmm0`–`xmm7` (8) | `xmm0`–`xmm3` (4) |
| Shadow space | None | 32 bytes required |
| Return int | `rax` | `rax` |
| Return float | `xmm0` | `xmm0` |
| Callee-saved | `rbp rbx r12–r15` | `rbp rbx rdi rsi r12–r15 xmm6–xmm15` |
| Exception model | DWARF `.eh_frame` + personality | Table-based SEH `.pdata`/`.xdata` |
| vtable layout | `[offset_to_top, typeinfo, vfuncs...]` | `[vfuncs...]` + separate COL |
| RTTI access | Negative offset from vptr | Separate `RTTICompleteObjectLocator` |
| Name mangling prefix | `_Z` | `?` |
| Destructor variants | D0 (deleting), D1 (complete), D2 (base) | scalar / vector `??_G`/`??_E` |
| Virtual base pointer | Thunks + VTT | Separate `vbptr` field |
| TLS access model | `fs:` relative (Linux) / `gs:` | `_tls_index` + TEB `gs:` |
| Stack alignment | 16-byte at call | 16-byte at call (after shadow space) |


---

@copyright jreuben11
