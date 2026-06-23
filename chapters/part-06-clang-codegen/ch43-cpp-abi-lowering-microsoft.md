# Chapter 43 — C++ ABI Lowering: Microsoft

*Part VI — Clang Internals: Codegen and ABI*

The Microsoft Visual C++ ABI is the dominant application binary interface on Windows and constitutes a substantial fraction of all real-world C++ binaries. Clang's implementation of this ABI — activated whenever the target triple contains `windows-msvc` or when `clang-cl` is invoked — diverges from the Itanium ABI at nearly every level: name mangling, vtable structure, RTTI layout, exception handling machinery, constructor/destructor variants, and member pointer representation. Understanding these divergences is essential for anyone writing compiler infrastructure that must interoperate with MSVC-compiled libraries, generating Windows PE binaries from a non-Windows host, or extending Clang's codegen for Windows-specific language features.

This chapter examines each divergence concretely, grounding every claim in actual IR emitted by `clang 22.1.3` targeting `x86_64-pc-windows-msvc`.

---

## Table of Contents

- [The `MicrosoftCXXABI` Class](#the-microsoftcxxabi-class)
  - [Scope of Differences from Itanium](#scope-of-differences-from-itanium)
- [MSVC Name Mangling](#msvc-name-mangling)
  - [The `MicrosoftCXXNameMangler`](#the-microsoftcxxnamemangler)
  - [Constructor and Destructor Encodings](#constructor-and-destructor-encodings)
  - [Calling Convention Codes in Mangled Names](#calling-convention-codes-in-mangled-names)
  - [Operator Encoding](#operator-encoding)
  - [Template and Namespace Mangling](#template-and-namespace-mangling)
  - [`__declspec(dllimport)` and the `__imp_` Prefix](#declspecdllimport-and-the-imp-prefix)
- [Virtual Table Layout](#virtual-table-layout)
  - [Structural Differences from Itanium](#structural-differences-from-itanium)
  - [Virtual Base Tables (`vbtable`)](#virtual-base-tables-vbtable)
  - [Multiple vftables per Object](#multiple-vftables-per-object)
  - [vftable Naming Convention](#vftable-naming-convention)
- [`vptr` Initialization in Constructors](#vptr-initialization-in-constructors)
  - [`initializeVTablePointers` and `EmitVBPtrInit`](#initializevtablepointers-and-emitvbptrinit)
  - [Constructor with `should_call_vbase_constructors` Bool](#constructor-with-shouldcallvbaseconstructors-bool)
- [Virtual Dispatch and Thunks](#virtual-dispatch-and-thunks)
  - [`__vfptr` Load Pattern](#vfptr-load-pattern)
  - [Thunks](#thunks)
  - [Virtual Function Dispatch Through Abstract Base](#virtual-function-dispatch-through-abstract-base)
- [RTTI and `dynamic_cast`](#rtti-and-dynamiccast)
  - [The RTTI Data Structures](#the-rtti-data-structures)
  - [`__RTDynamicCast`](#rtdynamiccast)
  - [`typeid()`](#typeid)
  - [Image-Relative Offsets and ASLR](#image-relative-offsets-and-aslr)
- [Constructor and Destructor Variants](#constructor-and-destructor-variants)
  - [Single Constructor vs. Itanium's C1/C2 Split](#single-constructor-vs-itaniums-c1c2-split)
  - [Destructor Variants: `??1` and `??_G`/`??_E`](#destructor-variants-1-and-ge)
  - [The `??_D` Complete Destructor Helper](#the-d-complete-destructor-helper)
  - [RAII and `cleanuppad` in Practice](#raii-and-cleanuppad-in-practice)
- [Member Pointers](#member-pointers)
  - [Inheritance Model Determines Representation Size](#inheritance-model-determines-representation-size)
  - [`MicrosoftMemberPointerInfo`](#microsoftmemberpointerinfo)
  - [Member Pointer Indirection and Calling Through a Member Pointer](#member-pointer-indirection-and-calling-through-a-member-pointer)
  - [Forward-Declared Classes and `__unspecified_inheritance`](#forward-declared-classes-and-unspecifiedinheritance)
- [Windows SEH and C++ EH](#windows-seh-and-c-eh)
  - [`__CxxFrameHandler3` Personality](#cxxframehandler3-personality)
  - [WinEH Funclet IR (catchpad / cleanuppad)](#wineh-funclet-ir-catchpad-cleanuppad)
  - [`_CxxThrowException` vs. `__cxa_throw`](#cxxthrowexception-vs-cxathrow)
  - [x86 32-bit EH Chain](#x86-32-bit-eh-chain)
  - [`llvm.eh.exceptionpointer` and `llvm.eh.exceptioncode`](#llvmehexceptionpointer-and-llvmehexceptioncode)
- [AArch64 Windows Structured Exception Handling](#aarch64-windows-structured-exception-handling)
  - [Maturity Status and Scope](#maturity-status-and-scope)
  - [The Windows SEH Model on AArch64](#the-windows-seh-model-on-aarch64)
  - [ARM64 `.pdata` Layout: Packed vs. Unpacked](#arm64-pdata-layout-packed-vs-unpacked)
  - [Key Differences from x64 SEH](#key-differences-from-x64-seh)
  - [`__try` / `__except` / `__finally` Lowering on AArch64](#try--except--finally-lowering-on-aarch64)
  - [EH Personality: `__C_specific_handler` vs. `__CxxFrameHandler3`](#eh-personality-c_specific_handler-vs-cxxframehandler3)
  - [PAC-RET and the `UOP_PACSignLR` Unwind Opcode](#pac-ret-and-the-uop_pacsignlr-unwind-opcode)
- [`__declspec` Extensions](#declspec-extensions)
  - [`dllimport` / `dllexport`](#dllimport-dllexport)
  - [`selectany` → `linkonce_odr` + comdat](#selectany-linkonceodr-comdat)
  - [`thread` → TLS via `thread_local`](#thread-tls-via-threadlocal)
  - [`align(N)` → IR Alignment](#alignn-ir-alignment)
  - [`naked` → IR `naked` Attribute](#naked-ir-naked-attribute)
  - [`restrict` → `noalias`](#restrict-noalias)
- [MSVC `operator new` and `operator delete`](#msvc-operator-new-and-operator-delete)
  - [Global Allocator Mangling](#global-allocator-mangling)
  - [Array `new` Cookie](#array-new-cookie)
- [clang-cl Compatibility and Divergences](#clang-cl-compatibility-and-divergences)
  - [Areas of Exact Compatibility](#areas-of-exact-compatibility)
  - [Intentional Divergences](#intentional-divergences)
  - [`-fms-compatibility-version`](#fms-compatibility-version)
  - [`__declspec(novtable)` Optimization](#declspecnovtable-optimization)
  - [`/EHsc`, `/EHs`, `/EHa`](#ehsc-ehs-eha)
  - [ARM64 Windows](#arm64-windows)
  - [Guard Variables for Static Local Initialization](#guard-variables-for-static-local-initialization)
  - [`__declspec(uuid)` and COM GUIDs](#declspecuuid-and-com-guids)
- [Summary](#summary)
- [Cross-References](#cross-references)
- [Reference Links](#reference-links)

---

## The `MicrosoftCXXABI` Class

Clang selects its C++ ABI implementation through the factory function `CreateMicrosoftCXXABI()` in
[`clang/lib/CodeGen/MicrosoftCXXABI.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/MicrosoftCXXABI.cpp).
The returned object implements the abstract `CGCXXABI` interface; all virtual-function dispatch, constructor/destructor emission, RTTI generation, member pointer operations, and EH setup are routed through this object.

The class hierarchy is:

```
CGCXXABI                         (clang/lib/CodeGen/CGCXXABI.h)
  └── MicrosoftCXXABI            (clang/lib/CodeGen/MicrosoftCXXABI.cpp)
```

The Microsoft ABI is activated by `TargetInfo::getCXXABI()` returning `TargetCXXABI::Microsoft`. The `ASTContext` mirrors this through `ASTContext::getCXXABIKind()`. Name mangling is handled by a parallel hierarchy rooted in `MangleContext`, with the Microsoft-specific implementation in `MicrosoftMangleContextImpl` inside
[`clang/lib/AST/MicrosoftMangle.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/AST/MicrosoftMangle.cpp).

### Scope of Differences from Itanium

| Category | Itanium | Microsoft |
|---|---|---|
| Name mangling | `_Z` prefix | `?` prefix |
| vtable layout | offset-to-top + RTTI ptr | RTTI ptr only (no offset-to-top) |
| Virtual bases | single `__vptr` | separate `__vfptr` / `__vbptr` |
| RTTI | `std::type_info` subclass | `TypeDescriptor` + locator chain |
| EH personality | `__gxx_personality_v0` | `__CxxFrameHandler3` |
| Constructor variants | C1 / C2 | single with bool |
| Destructor variants | D0 / D1 / D2 | `??_G` (deleting) + `??1` (base) |
| Member pointers | uniform struct | size varies by inheritance model |
| `throw` runtime | `__cxa_throw` | `_CxxThrowException` |

---

## MSVC Name Mangling

### The `MicrosoftCXXNameMangler`

MSVC decorated names begin with `?` followed by a hierarchical encoding of the fully-qualified name. The core class is `MicrosoftCXXNameMangler` in `MicrosoftMangle.cpp`, which builds a substitution table (`BackReferences`) distinct from Itanium's `<substitution>` grammar. The MSVC scheme encodes names in reverse order (innermost scope first), separated by `@`, and terminated by `@@`.

The mangling of a member function encodes, in order: function name, enclosing class scopes (innermost-first), function access/type qualifiers, calling convention, return type, parameter types, and a trailing `@Z` terminator.

### Constructor and Destructor Encodings

MSVC uses fixed numeric codes for special member functions:

| Member | Encoded name fragment |
|---|---|
| Constructor | `??0` |
| Destructor | `??1` |
| Scalar delete destructor | `??_G` |
| Vector delete destructor | `??_E` |

Examples from Clang 22 output when compiling for `x86_64-pc-windows-msvc`:

```
??0Foo@NS@@QEAA@XZ        → public: __cdecl NS::Foo::Foo(void)
??1Foo@NS@@QEAA@XZ        → public: __cdecl NS::Foo::~Foo(void)
?bar@Foo@NS@@QEAAHI@Z     → public: int __cdecl NS::Foo::bar(unsigned int)
?staticMethod@Foo@NS@@SAXXZ → public: static void __cdecl NS::Foo::staticMethod(void)
??_GDerived@@UEAAPEAXI@Z  → scalar delete destructor
??_DDerived@@QEAAXXZ      → complete destructor helper
```

Verified via `llvm-undname`:

```bash
$ llvm-undname "??0Foo@NS@@QEAA@XZ"
public: __cdecl NS::Foo::Foo(void)
```

### Calling Convention Codes in Mangled Names

The character after the qualifier string encodes calling convention:

| Code | Convention |
|---|---|
| `A` | `__cdecl` |
| `E` | `__thiscall` |
| `G` | `__stdcall` |
| `I` | `__fastcall` |
| `Q` | `__vectorcall` |
| `S` | static `__cdecl` |

On x86-64 all member functions use `__cdecl` (the `QEAA` pattern seen above: `Q` = public, `E` = 64-bit `this` pointer qualifier, `A` = `__cdecl`, `A` = no `const`).

### Operator Encoding

Operators are encoded with `??` followed by a positional code:

| Operator | Code |
|---|---|
| `operator new` | `??2` |
| `operator delete` | `??3` |
| `operator=` | `??4` |
| `operator>>` | `??5` |
| `operator<<` | `??6` |
| `operator+` | `??H` |
| `operator==` | `??8` |
| `operator[]` | `??A` |
| `operator()` | `??R` |

Global `operator new` mangles to `??2@YAPEAX_K@Z` on 64-bit Windows (return `void*`, size_t parameter). Global `operator delete` with size mangles to `??3@YAXPEAX_K@Z`.

### Template and Namespace Mangling

Templates introduce angle brackets into the mangled name via `?$` notation. A class template `Container<double>` becomes `?$Container@N`, where `N` is the type code for `double`. Nested template arguments chain using the same `@`-separator rule. From Clang 22, `Container<double>::Container()` mangles as:

```
??0?$Container@N@@QEAA@XZ
```

Where `?$Container@N` is the template class name, `@@` closes the scope chain, and the remainder follows the standard member function encoding. `MicrosoftCXXNameMangler::mangleTemplateArgs()` handles the argument encoding, using a separate back-reference table for template arguments (`TemplateArgBackReferences`).

Namespaces are encoded identically to classes in the scope chain — no structural distinction exists between `NS::Foo` and a class `Foo` nested inside class `NS`. This means that a namespace-scope function `NS::bar()` and a static member function `Foo::bar()` (where `Foo` is a class also named `NS`) would mangle identically if their signatures match, a known MSVC ABI quirk that clang-cl preserves.

### `__declspec(dllimport)` and the `__imp_` Prefix

When a function is declared with `__declspec(dllimport)`, the linker stub is accessed via an import thunk. Clang emits the declaration with `dllimport` linkage in LLVM IR, and the linker generates a `__imp_`-prefixed entry in the import table. Internally in the IR, the function appears with its normal mangled name but carries the `dllimport` attribute, directing the backend to generate an indirect call through the IAT (Import Address Table).

The `__imp_` prefix is applied at the machine-code level: the backend emits an indirect `CALL [__imp_FunctionName]` instruction that dereferences the IAT slot rather than a direct call. For data symbols declared `dllimport`, Clang similarly emits an `__imp_`-prefixed load. This pattern enables DLL delay-loading and forward exports without recompiling callers.

---

## Virtual Table Layout

### Structural Differences from Itanium

Itanium vtables contain a negative-offset region with `offset-to-top` and a type-info pointer, followed by the function pointer array. The MSVC vtable (called a `vftable`) contains only function pointers, with one exception: the first slot is occupied by the `_RTTICompleteObjectLocator` pointer — but this pointer lives at `vtable[-1]` (one word before the published vtable address), not in-band among function slots.

The MSVC vtable for a simple class with two virtual functions and a destructor:

```
vftable layout for Derived (64-bit MSVC):
  [-1]  ptr to _RTTICompleteObjectLocator (??_R4Derived@@6B@)
  [0]   ptr to scalar delete destructor   (?_GDerived...)
  [1]   ptr to foo                        (?foo@Derived...)
```

In IR, this is emitted as:

```llvm
@0 = private unnamed_addr constant { [3 x ptr] } {
  [3 x ptr] [
    ptr @"??_R4Derived@@6B@",      ; RTTI locator at slot -1
    ptr @"??_EDerived@@UEAAPEAXI@Z",  ; vector delete destructor
    ptr @"?foo@Derived@@UEAAXXZ"   ; virtual function
  ]
}, comdat($"??_7Derived@@6B@")

; The published vtable pointer points one slot past the locator:
@"??_7Derived@@6B@" = unnamed_addr alias ptr,
    getelementptr inbounds ({ [3 x ptr] }, ptr @0, i32 0, i32 0, i32 1)
```

The `getelementptr` offset of `i32 1` places `??_7Derived@@6B@` at the address of slot 0 (the first function pointer), skipping the locator. This is the pointer stored in `this->__vfptr`.

### Virtual Base Tables (`vbtable`)

When virtual inheritance is present, MSVC introduces a separate `vbtable` (virtual base table) alongside the `vftable`. The `vbtable` encodes the byte offsets from the `vbptr` field to each virtual base subobject. The structure used in IR is a plain array of `i32` values, where index 0 holds a self-relative offset (always negative — the offset from the vbptr back to the beginning of the derived object) and subsequent indices hold offsets to each virtual base.

For the diamond hierarchy `Diamond : A, B` (both virtually inheriting `VBase`), Clang 22 emits:

```llvm
@"??_8Diamond@@7BA@@@" = linkonce_odr unnamed_addr constant [2 x i32]
    [i32 -8, i32 48], comdat, align 4

@"??_8Diamond@@7BB@@@" = linkonce_odr unnamed_addr constant [2 x i32]
    [i32 -8, i32 24], comdat, align 4
```

The naming convention `??_8ClassName@@7BBaseName@@@` identifies the vbtable used when accessing a specific base's virtual base pointer field. The first `i32` (-8) is the offset from the `vbptr` field back to the object's start; the second is the offset from the `vbptr` to the virtual base.

### Multiple vftables per Object

Unlike Itanium (one `__vptr` per polymorphic object), MSVC places a separate `__vfptr` for each base subobject that contributes virtual functions independently. In the diamond example, `Diamond` carries three vtable pointers: one from `A`'s perspective, one from `B`'s perspective, and one from `VBase`'s perspective, each with a corresponding `vftable`:

```llvm
@"??_7Diamond@@6BA@@@"   ; vftable for Diamond seen as A
@"??_7Diamond@@6BB@@@"   ; vftable for Diamond seen as B
@"??_7Diamond@@6BVBase@@@"  ; vftable for Diamond seen as VBase
```

The layout of a `Diamond` object on 64-bit MSVC (verified from Clang 22 output, total size 72 bytes) is:

```
Offset  Field
  0     __vfptr (ptr to ??_7Diamond@@6BA@@@)    — A's vftable
  8     __vbptr (ptr to ??_8Diamond@@7BA@@@)    — A's vbtable ptr
 16     a_data (int, 4B) + padding
 24     __vfptr (ptr to ??_7Diamond@@6BB@@@)    — B's vftable
 32     __vbptr (ptr to ??_8Diamond@@7BB@@@)    — B's vbtable ptr
 40     b_data (int, 4B) + padding
 48     d_data (int, 4B) + padding
 56     VBase subobject:
   56    __vfptr (ptr to ??_7Diamond@@6BVBase@@@) — VBase's vftable
   64    vb_data (int, 4B) + padding
```

This contrasts with Itanium, where the single virtual base subobject is reached via one `__vptr` and a compile-time offset; the MSVC layout requires runtime indirection through the vbtable even for non-virtual function dispatch when the dynamic type is unknown.

### vftable Naming Convention

MSVC vftable symbol names follow the pattern `??_7ClassName@@6BBaseContext@@@`. The `7` indicates a vftable (vs `8` for vbtable). The `BBaseContext@@` suffix qualifies which base class's virtual function set the vftable satisfies. For a class with no multiple-inheritance complication, the suffix is simply `B@` (empty base context). For each additional base providing an independent vtable pointer, the base class name is encoded.

---

## `vptr` Initialization in Constructors

### `initializeVTablePointers` and `EmitVBPtrInit`

`MicrosoftCXXABI::initializeVTablePointers()` is called from `CodeGenFunction::EmitConstructorBody()`. It iterates the class's `VBPtrInfo` list to initialize all `__vbptr` fields and then assigns all `__vfptr` fields to the appropriate vftable addresses for the complete type being constructed.

`EmitVBPtrInit()` handles virtual base pointer initialization. For a class with virtual bases, the `__vbptr` field is set to the appropriate vbtable's address.

### Constructor with `should_call_vbase_constructors` Bool

The most visible MSVC ABI difference in constructor codegen is the boolean parameter. The Itanium ABI uses separate C1 (complete-object) and C2 (base-object) constructor variants. The MSVC ABI uses a single constructor with an additional `i32` parameter (the "most derived" flag) that, when nonzero, directs the constructor to initialize virtual base subobjects and set `vbptr` fields.

From the Clang 22 IR for `Derived::Derived()` with virtual base `VBase`:

```llvm
define linkonce_odr dso_local noundef ptr @"??0Derived@@QEAA@XZ"(
    ptr noundef nonnull returned align 8 dereferenceable(16) %0,
    i32 noundef %1)   ; %1 = should_call_vbase_constructors
unnamed_addr #0 comdat align 2 {
  ...
  %7 = load i32, ptr %4, align 4     ; load bool param
  %8 = icmp ne i32 %7, 0
  br i1 %8, label %9, label %13      ; branch on bool

9:  ; most-derived path: set vbptr, construct VBase
  store ptr @"??_8Derived@@7B@", ptr %10, align 8  ; vbptr = vbtable
  call ptr @"??0VBase@@QEAA@XZ"(ptr %11)           ; construct VBase
  br label %13

13: ; both paths: set own vfptr, initialize data members
  store ptr @"??_7Derived@@6B@", ptr %20, align 8  ; vfptr = vftable
  store i32 0, ptr %21, align 8                    ; d = 0
  ...
}
```

When `Derived` is constructed as a complete object, the caller passes `i32 1`. When it is constructed as a base subobject within a further-derived class's constructor, the caller passes `i32 0`, suppressing virtual base initialization (which the most-derived constructor handles).

---

## Virtual Dispatch and Thunks

### `__vfptr` Load Pattern

Virtual dispatch in MSVC ABI IR follows the same load-GEP-call pattern as Itanium, but without the `offset-to-top` complexity in the common case. For a simple virtual call `d->vf()` through a `VBase*` obtained via vbtable lookup:

```llvm
; Virtual call through vbtable (from call_virtual in Diamond example)
%4 = getelementptr inbounds i8, ptr %3, i64 8        ; load vbptr field
%5 = load ptr, ptr %4, align 8                       ; vbptr
%6 = getelementptr inbounds i32, ptr %5, i32 1       ; vbtable[1]
%7 = load i32, ptr %6, align 4                       ; offset to VBase
%8 = sext i32 %7 to i64
%9 = add nsw i64 8, %8                               ; vbptr_offset + delta
%10 = getelementptr inbounds i8, ptr %3, i64 %9      ; &VBase subobject
%11 = load ptr, ptr %10, align 8                     ; load vfptr
%12 = getelementptr inbounds ptr, ptr %11, i64 0     ; vftable[0]
%13 = load ptr, ptr %12, align 8                     ; function pointer
call void %13(ptr %10)                               ; dispatch
```

The indirection through the vbtable is required because the offset from a `Diamond*` to its `VBase` subobject is not a compile-time constant; it depends on which concrete class owns the `Diamond`.

### Thunks

MSVC vcall thunks differ architecturally from Itanium. In Itanium, a virtual function that overrides with a covariant return type requires an out-of-line return-adjusting thunk that performs pointer arithmetic on the return value. In MSVC, thunks for base-subobject vftable slots adjust `this` rather than the return value in the common case. Clang emits `ReturnAdjustingThunk` entries when a derived class overrides a function whose base-subobject vftable entry requires a different `this` offset. These thunks are emitted as normal IR functions with the base class's mangled name suffix and stored inline in the vtable.

For the `ConcreteFactory::create()` override with covariant return (`Derived*` overriding `Base*`), Clang generates a thunk that calls the actual implementation and then adjusts the return pointer. The MSVC convention is that vtable slots always carry the type of the base class's virtual function; a `ConcreteFactory` seen as a `Factory` has its `create()` slot point to a thunk that calls the real implementation and casts the return. The backend can sometimes collapse this into a tail call optimization.

`this`-adjusting thunks appear in multiple-inheritance scenarios when a base-subobject vftable slot must forward a call to an overrider that lives at a different `this` offset. For example, if `B::f()` is overridden in `Diamond` but the `B` vftable slot carries `Diamond::f()`, the thunk must subtract the `B`-to-`Diamond` offset from `this` before dispatching. The `VTableThunk` structure in `clang/lib/CodeGen/MicrosoftCXXABI.cpp` records the `ThisAdjustment` as an `i64` byte delta applied before the real function is entered.

### Virtual Function Dispatch Through Abstract Base

An abstract class — one with pure virtual functions — uses `_purecall` in vtable slots for the pure virtual entries:

```llvm
; Pure virtual slot in vtable
ptr @_purecall   ; ← placeholder in vtable for pure virtual
```

`_purecall` is a CRT function that immediately aborts with a specific error code. This is unlike Itanium, which also uses a `__cxa_pure_virtual` placeholder. In MSVC, calling a pure virtual function through a partially-constructed object (e.g., from a base class constructor) triggers `_purecall` rather than undefined behavior silently.

---

## RTTI and `dynamic_cast`

### The RTTI Data Structures

The MSVC RTTI system uses four distinct data structures, all emitted as LLVM IR globals with `linkonce_odr` linkage and placed in the `.xdata` section. Clang 22 generates the following LLVM struct types:

```llvm
%rtti.TypeDescriptor13   = type { ptr, ptr, [14 x i8] }
%rtti.BaseClassDescriptor = type { i32, i32, i32, i32, i32, i32, i32 }
%rtti.ClassHierarchyDescriptor = type { i32, i32, i32, i32 }
%rtti.CompleteObjectLocator    = type { i32, i32, i32, i32, i32, i32 }
```

**`TypeDescriptor`** (mapped to `rtti.TypeDescriptorN`) holds a pointer to the `type_info` vtable, a runtime-allocated demangled-name cache pointer (initially null), and the mangled type name as a null-terminated string. The `N` suffix in the type name reflects the string length, making each distinct type have a uniquely-sized struct.

**`BaseClassDescriptor`** (mapped to `rtti.BaseClassDescriptor`) describes one base class in the hierarchy. Its fields are:
1. Image-relative offset to the base's `TypeDescriptor`
2. Number of direct bases of this base (used for hierarchy walk optimization)
3. `mdisp` — offset of the base within the object (member displacement)
4. `pdisp` — vbtable offset (-1 if not accessed via virtual base)
5. `vdisp` — offset within the vbtable to the vbase offset
6. Attributes flags (0x40 = `CHD_Ambiguous`, 0x20 = `CHD_Offset_Is_Virtual`, etc.)
7. Image-relative offset to the base's `ClassHierarchyDescriptor`

**`ClassHierarchyDescriptor`** records the total number of base classes (including indirect) and points to an array of image-relative offsets to `BaseClassDescriptor` entries.

**`CompleteObjectLocator`** sits at `vtable[-1]` (one pointer before the published vtable address) and contains:
1. Signature (1 on 64-bit, 0 on 32-bit — indicates image-relative offsets)
2. Offset of this vftable's object subobject from the complete object
3. Constructor displacement (for virtual base vtables)
4. Image-relative offset to `TypeDescriptor`
5. Image-relative offset to `ClassHierarchyDescriptor`
6. Image-relative offset to self (used for image-relative recovery)

For the `Derived : Base` example from Clang 22:

```llvm
@"??_R4Derived@@6B@" = linkonce_odr constant %rtti.CompleteObjectLocator {
  i32 1,          ; 64-bit image-relative mode
  i32 0,          ; offset of this subobject from complete object = 0
  i32 0,          ; ctor displacement = 0
  i32 trunc(TypeDescriptor - ImageBase),
  i32 trunc(ClassHierarchyDescriptor - ImageBase),
  i32 trunc(Self - ImageBase)  ; self-relative ptr for image recovery
}, comdat

@"??_R0?AUDerived@@@8" = linkonce_odr global %rtti.TypeDescriptor13 {
  ptr @"??_7type_info@@6B@",   ; type_info vtable
  ptr null,                    ; runtime cache (initially null)
  [14 x i8] c".?AUDerived@@\00"  ; mangled name with .?AU prefix
}, comdat
```

The `.?AU` prefix on the name string is the MSVC decoration for `struct`; classes use `.?AV`, and fundamental types use different schemes. The runtime (`msvcrt.dll`) uses this string for `typeid().name()` and `typeid().raw_name()`.

### `__RTDynamicCast`

`dynamic_cast` on Windows calls `__RTDynamicCast` (exported from `vcruntime140.dll`), which walks the `ClassHierarchyDescriptor` tree. Clang emits the call as:

```llvm
call ptr @__RTDynamicCast(
  ptr %obj,                    ; source pointer
  i32 %src_offset,             ; offset from source ptr to complete object
  ptr @TypeDescriptor_src,     ; source type
  ptr @TypeDescriptor_dst,     ; destination type
  i32 %is_reference_cast       ; 0 = pointer cast, 1 = reference cast
)
```

The `is_reference_cast` flag controls whether failure throws `std::bad_cast` (1) or returns null (0).

### `typeid()`

`typeid()` is implemented by loading the `CompleteObjectLocator` from `vtable[-1]`, then following the image-relative offset to the `TypeDescriptor`. The `std::type_info` object *is* the `TypeDescriptor` (the first two pointer fields — vtable ptr and cache — constitute the `type_info` object layout). The `name()` method returns `raw_name()` with the leading `.` stripped and the cache used for demangling.

### Image-Relative Offsets and ASLR

A critical aspect of the 64-bit MSVC RTTI layout is the pervasive use of **image-relative offsets** (RVAs — Relative Virtual Addresses) rather than absolute pointers. In IR, these are expressed as `i32 trunc (i64 sub nuw nsw (i64 ptrtoint (...) to i64), i64 ptrtoint (ptr @__ImageBase to i64)) to i32)`.

This encoding constrains all RTTI data to fit within a 4 GB window relative to the module's base address, which is always satisfied for PE images. The benefit is that RTTI structures in `.rdata`/`.xdata` need no runtime relocations: the `__ImageBase` symbol is defined at link time, and the arithmetic folds into constants. When the PE loader maps the image at a different base address (ASLR), `__ImageBase` moves with it and all image-relative addresses remain valid without fixups.

Clang emits `@__ImageBase = external dso_local constant i8` as a zero-sized external symbol whose address is the PE image base. All RVA calculations are then `ptrtoint(target) - ptrtoint(__ImageBase)`, truncated to `i32`.

This design means RTTI structures are not valid across DLL boundaries without additional indirection — `CompleteObjectLocator` and `TypeDescriptor` pointers within a DLL are only meaningful relative to that DLL's `__ImageBase`. Cross-DLL `dynamic_cast` works because `__RTDynamicCast` uses the locator's self-relative field to recover the image base at runtime.

---

## Constructor and Destructor Variants

### Single Constructor vs. Itanium's C1/C2 Split

As demonstrated above, MSVC uses one constructor function signature where the second parameter is `i32 %should_call_vbase_constructors`. The caller is responsible for passing 1 (complete-object construction) or 0 (base-subobject construction from a more-derived constructor). In contrast, Itanium compiles two distinct functions: `C1` initializes virtual bases and `C2` does not.

The `MicrosoftCXXABI::EmitCXXConstructors()` method generates only one variant (`Ctor_Complete` in Clang's enum, mapped to the single MSVC constructor).

### Destructor Variants: `??1` and `??_G`/`??_E`

MSVC destructors are split by whether they also `delete` the storage:

- `??1ClassName@@...` — the **base destructor** (`Dtor_Base` in Clang): runs the C++ destructor body without freeing heap storage. Called when destroying stack objects, base subobjects, or manually when managing memory separately.
- `??_GClassName@@...` — the **scalar deleting destructor** (`Dtor_Deleting`): calls `??1`, then calls `operator delete`. Stored in vtable slot 0.
- `??_EClassName@@...` — the **vector deleting destructor**: handles `delete[]`. Stored in the vtable as a weak alias to `??_G` if the class doesn't override it.

```llvm
; Scalar deleting destructor (weak alias to base destructor + delete)
@"??_EBase@@UEAAPEAXI@Z" = weak dso_local unnamed_addr alias
  ptr (ptr, i32), ptr @"??_GBase@@UEAAPEAXI@Z"
```

The second `i32` parameter to the deleting destructors is a flags field: bit 0 controls whether to call `operator delete`, enabling the same function to serve as both a destructor-only call (flag=0) and a deleting call (flag=1). This is how `MicrosoftCXXABI::EmitDestructorCall()` generates calls: passing `i32 1` when destruction includes deletion, `i32 0` for base-subobject destruction.

There is no MSVC equivalent of Itanium's `D0` (deleting) vs `D1` (complete) vs `D2` (base) three-way split.

### The `??_D` Complete Destructor Helper

The `??_D` symbol (`??_DDerived@@QEAAXXZ`) is a non-virtual destructor that sequences destruction of base subobjects and data members in the correct order, without involving `operator delete`. It is generated when a class has virtual bases: it first destroys data members, then calls the base subobject destructors (including virtual base destructors), and finally — unlike the Itanium D1 — does not set vptrs back (since MSVC skips the D2 path entirely). The complete destructor helper is called from the destructor body when the class has virtual bases and from cleanup funclets during EH unwind.

For the diamond example's `Derived` class:

```llvm
define linkonce_odr dso_local void @"??_DDerived@@QEAAXXZ"(
    ptr noundef nonnull align 8 dereferenceable(16) %0)
unnamed_addr #3 comdat align 2 {
  ; Call ??1Derived (base destructor body)
  ; Call ??1VBase (virtual base destructor)
  ret void
}
```

### RAII and `cleanuppad` in Practice

The `cleanuppad`/`cleanupret` pair in funclet IR directly encodes RAII destruction. For the `Guard` RAII wrapper example:

```llvm
define dso_local noundef i32 @"?with_raii@@YAHH@Z"(i32 %0) #0
    personality ptr @__CxxFrameHandler3 {
  ; ... normal path ...
  invoke void @_CxxThrowException(ptr %5, ptr @"_TI1?AUErr@@") #2
          to label %18 unwind label %16

16:  ; cleanup funclet
  %17 = cleanuppad within none []
  call void @"??1Guard@@QEAA@XZ"(ptr nonnull align 8 %4) #3
        [ "funclet"(token %17) ]
  cleanupret from %17 unwind to caller
}
```

The `cleanuppad within none []` token bounds the cleanup. The `[ "funclet"(token %17) ]` operand bundle on the destructor call is mandatory — without it, the WinEH preparation pass cannot associate the call with its enclosing funclet, and the backend will misgenerate the unwind table. Clang ensures this bundle is attached by `CodeGenFunction::EmitCallOrInvoke()` when inside a funclet scope tracked through `EHStack::getInnermostEHScope()`.

---

## Member Pointers

### Inheritance Model Determines Representation Size

The MSVC ABI's member pointer layout is uniquely complex: the size and layout of a member pointer `T C::*` depends on the **inheritance model** of `C`, which in turn depends on whether the compiler has seen the full class definition at the point of use. Clang, following MSVC, resolves this at the point of use and may need to annotate class declarations with `__single_inheritance`, `__multiple_inheritance`, or `__virtual_inheritance` pragmas when the definition is not yet available.

| Inheritance model | Data member pointer | Function member pointer |
|---|---|---|
| `__single_inheritance` | 4 bytes (offset) | 4 bytes (fn ptr) |
| `__multiple_inheritance` | 4 bytes (offset) | 8 bytes: {fn ptr, this-adj} |
| `__virtual_inheritance` | 4 bytes (offset) | 12 bytes: {fn ptr, vbtable-offset, this-adj} |
| Unknown / `__unspecified_inheritance` | 16 bytes | 16 bytes |

From Clang 22 IR for the member pointer test:

```llvm
; Single inheritance function member pointer — just a ptr
@"?pf1@@3P8Single@@EAAXXZEQ1@" = dso_local global
  ptr @"?foo@Single@@QEAAXXZ", align 8

; Multiple inheritance function member pointer — {ptr, i32}
@"?pf2@@3P8Multi@@EAAXXZEQ1@" = dso_local global
  { ptr, i32 } { ptr @"?f3@Multi@@QEAAXXZ", i32 0 }, align 8

; Virtual inheritance function member pointer — {ptr, i32, i32}
@"?pf3@@3P8Derived@@EAAXXZEQ1@" = dso_local global
  { ptr, i32, i32 } { ptr @"?df@Derived@@QEAAXXZ", i32 0, i32 0 }, align 8
```

For the virtual inheritance case, the three fields are:
1. Function pointer
2. vbtable offset (which entry in the vbtable gives the virtual base offset)
3. Non-virtual `this` adjustment applied *after* virtual base lookup

### `MicrosoftMemberPointerInfo`

The `MicrosoftCXXABI` implementation queries `MicrosoftMemberPointerInfo` to determine the representation for a given `MemberPointerType`. The key methods are:

- `getCXXMemberPointerType(const MemberPointerType*)` — returns the LLVM type for the representation
- `EmitMemberPointerConversion()` — handles upcasting and null/non-null conversions between member pointer types with different inheritance models
- `EmitMemberPointerComparison()` — generates the field-by-field comparison required for member pointer equality

Null member pointers use representation-dependent null values: for single-inheritance data pointers, the null is the integer -1 (since offset 0 is a valid data member offset). For function pointers, null is represented by a null function pointer with zero adjustments.

### Member Pointer Indirection and Calling Through a Member Pointer

Calling through a function member pointer requires code that branches on the inheritance model. For a `void (Multi::*pmf)()` (multiple-inheritance model, `{ptr, i32}` representation), the call sequence:

1. Load the function pointer from field 0 of the member pointer struct.
2. Load the `this`-adjustment from field 1 (`i32` byte delta).
3. Sign-extend the adjustment to pointer width and add it to `this`.
4. Call the function pointer with the adjusted `this`.

For virtual-inheritance member pointers with the `{ptr, i32, i32}` layout:

1. Load function pointer.
2. Load vbtable-entry index from field 1.
3. If the function pointer is non-null (not null member pointer), load `this->__vbptr` at the vbtable-entry index to get the virtual base offset.
4. Add the virtual base offset plus the non-virtual `this`-adjustment (field 2) to `this`.
5. Call the function pointer.

`MicrosoftCXXABI::EmitMemberPointerIndirection()` generates this sequence, emitting a `br` on whether the function pointer is null or the vbtable index is meaningful.

### Forward-Declared Classes and `__unspecified_inheritance`

When a member pointer to `C` is used before `C`'s full definition is seen, the MSVC ABI cannot determine the inheritance model. Clang tracks this as the `__unspecified_inheritance` model and uses 16-byte representation for both data and function member pointers — the maximum possible size, ensuring the representation is compatible with whatever model `C` turns out to have. A subsequent full definition may shrink the effective model; the `MicrosoftCXXABI` implementation must then emit conversion code (handled via `EmitMemberPointerConversion()`) if a wider representation is assigned to a narrower-typed variable.

---

## Windows SEH and C++ EH

### `__CxxFrameHandler3` Personality

Functions that contain `try`/`catch` or use RAII under the MSVC ABI carry the `__CxxFrameHandler3` personality function:

```llvm
define dso_local noundef i32 @"?safe_op@@YAHH@Z"(i32 %0)
    personality ptr @__CxxFrameHandler3 {
```

This is in contrast to Itanium's `__gxx_personality_v0`. The `__CxxFrameHandler3` implementation lives in `vcruntime140.dll` and interprets `.xdata` unwind tables that Clang's backend generates for Windows targets.

### WinEH Funclet IR (catchpad / cleanuppad)

Windows EH abandons the Itanium `landingpad` model entirely. Instead, each catch block and cleanup block becomes a **funclet** — a function-like IR region entered via `catchpad` or `cleanuppad` tokens. The complete IR pattern from the EH test with two catch clauses:

```llvm
; Unwind dispatch
%13 = catchswitch within none [label %14, label %19] unwind to caller

; First catch clause: catch (const MyError& e)
14:
  %15 = catchpad within %13 [
    ptr @"??_R0?AUMyError@@@8",   ; TypeDescriptor for catch type
    i32 8,                         ; flags: 0x8 = catch by reference
    ptr %5                         ; pointer-to-pointer (address of catch var)
  ]
  %16 = load ptr, ptr %5, align 8, !nonnull !8
  ; use exception object via %16 ...
  catchret from %15 to label %25

; Second catch clause: catch (...)
19:
  %20 = catchpad within %13 [
    ptr null,     ; null TypeDescriptor = catch-all
    i32 64,       ; 0x40 = catch-all flag
    ptr null
  ]
  store i32 -999, ptr %2, align 4
  catchret from %20 to label %24
```

The `catchpad` operands form a triple that `__CxxFrameHandler3` uses to match exception types at runtime:
- Operand 0: pointer to `TypeDescriptor` (null for `catch(...)`)
- Operand 1: flags (`0x1` = const, `0x2` = volatile, `0x8` = by reference, `0x40` = catch-all)
- Operand 2: address where the caught exception pointer/reference is stored

`cleanuppad` is used for RAII destructors and stack unwinding:

```llvm
%5 = cleanuppad within none []
call void @"??3@YAXPEAX_K@Z"(ptr %storage, i64 32) #5 [ "funclet"(token %5) ]
cleanupret from %5 unwind to caller
```

The `[ "funclet"(token %5) ]` operand bundle on every call within a funclet is mandatory — it establishes the funclet token chain that the backend uses to build the `.xdata` unwind table. Instructions inside funclets are not dominated by the funclet entry in the CFG sense; they form a separate "funclet scope" that the WinEH preparation pass validates.

### `_CxxThrowException` vs. `__cxa_throw`

Throwing an exception calls `_CxxThrowException` (not `__cxa_throw`):

```llvm
call void @_CxxThrowException(
  ptr %exception_object,        ; pointer to exception object (stack-allocated)
  ptr @"_TI1?AUMyError@@"       ; ThrowInfo describing the type
) #3  ; noreturn
```

The `ThrowInfo` structure (`_TI1?AUMyError@@` in the IR) contains:
1. Attributes (const/volatile of thrown object)
2. Image-relative offset to destructor (for thrown object cleanup on catch)
3. Image-relative offset to forward-compat data (unused)
4. Image-relative offset to `CatchableTypeArray`

The `CatchableTypeArray` lists all types that can match this throw, ordered from most-derived to least-derived. Each entry is a `CatchableType` that describes the conversion from the thrown type to one of the base types. Both `ThrowInfo` and `CatchableTypeArray` are emitted to the `.xdata` section with `linkonce_odr` + comdat, so they collapse across translation units.

### x86 32-bit EH Chain

On 32-bit x86 Windows, `__CxxFrameHandler3` uses a stack-based `EXCEPTION_REGISTRATION_RECORD` linked list rooted at `fs:[0]`. Each function with EH establishes a frame record on the stack, chaining them in a linked list. The compiler generates a state table (negative stack slots) and a handler table. Clang's 32-bit MSVC target generates compatible structures. This mechanism is categorically different from the 64-bit `RUNTIME_FUNCTION`/`.pdata`/`.xdata` table-driven unwinding.

The 64-bit model (x86-64, ARM64) uses structured exception handling (SEH) with image-relative unwind tables registered in `.pdata`, requiring no per-frame linked list. The WinEH funclet IR described above is the common abstraction for both.

### `llvm.eh.exceptionpointer` and `llvm.eh.exceptioncode`

Two LLVM intrinsics expose Windows EH metadata:

- `llvm.eh.exceptionpointer(token)` — inside a `catchpad`, returns the exception object pointer (the same address that `__CxxFrameHandler3` deposited into the catch variable slot)
- `llvm.eh.exceptioncode()` — inside a `catchpad` for SEH (`__except` filters), returns the Win32 exception code as an `i32`

These appear in the IR when compiling structured SEH (`__try`/`__except`/`__finally`) under clang-cl, which maps to `catchpad`/`cleanuppad` in the same way as C++ EH.

---

## AArch64 Windows Structured Exception Handling

### Maturity Status and Scope

Windows on ARM64 (`aarch64-pc-windows-msvc`) has been a supported Clang/LLVM target for several years; the core SEH machinery — `.pdata`, `.xdata`, and frame-based unwinding — has been production-quality for native ARM64 Windows development for several release cycles. What has evolved more recently is support for Pointer Authentication Code return-address signing (`UOP_PACSignLR`) in the unwind table, which LLVM 22 fully handles as described later in this section.

This section focuses on the AArch64-specific aspects of Windows structured exception handling as emitted by `clang 22.1.6` targeting `aarch64-pc-windows-msvc`. The funclet IR model (`catchpad`, `cleanuppad`, `catchswitch`) and the `MicrosoftCXXABI` codegen are shared with x64; the differences lie in the unwind table format emitted to `.pdata`/`.xdata` and in the calling constraints that the prologue model imposes.

### The Windows SEH Model on AArch64

Windows SEH on both x64 and AArch64 uses table-driven frame unwinding. Every function with non-trivial prologue/epilogue emits a `RUNTIME_FUNCTION` record into the `.pdata` section. The record stores a `BeginAddress` RVA and either packed unwind data or a pointer to a full `.xdata` `ExceptionDataRecord`. The operating system's unwind engine reads these tables during stack walking — no per-frame linked list exists, in contrast to the 32-bit x86 model described in the previous section.

On AArch64, the `RUNTIME_FUNCTION` is defined by `ARM::WinEH::RuntimeFunctionARM64` in
[`llvm/include/llvm/Support/ARMWinEH.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/ARMWinEH.h):

```
 31                   16 15          13 12   10  9  8         2  1 0
+---------------------+---+---+-------+---+-------+----------+---+
|     Frame Size      |CR | H | RegI  |RegF| Fn Len |  Flags  |
+---------------------+---+---+-------+---+--------+----------+---+
```

The two-bit `Flags` field at bits [1:0] determines the form:
- `00` (RFF_Unpacked): bits [31:2] are an image-relative offset to a full `.xdata` `ExceptionDataRecord`
- `01` (RFF_Packed): the remaining 30 bits encode the unwind information inline — no `.xdata` record is needed
- `10` (RFF_PackedFragment): packed form for a function fragment (no prologue)

For functions with SEH handlers (`__try`/`__except`/`__finally`) or C++ EH, the packed form cannot be used because the handler address must appear in the `.xdata` extension. Clang always emits the unpacked form for such functions.

When the X bit in the `ExceptionDataRecord` header is set, the unwind byte code is followed by an image-relative exception handler RVA and optional handler data. This is where `__C_specific_handler` or `__CxxFrameHandler3` appears in the binary.

The CONTEXT record used during AArch64 Windows exception dispatch is the standard `ARM64_NT_CONTEXT` structure, larger than x64's `CONTEXT` due to the additional NEON register state (32 Q-registers). The unwinder populates this structure from the `.xdata` unwind codes before calling into the exception filter.

### ARM64 `.pdata` Layout: Packed vs. Unpacked

For a canonical leaf function or a simple frame-saving non-EH function, Clang emits packed unwind data. The `RuntimeFunctionARM64` fields encode:

| Field | Bits | Meaning |
|---|---|---|
| `Flags` | [1:0] | 01 = packed |
| `FunctionLength` | [12:2] | Function byte-length / 4 |
| `RegF` | [15:13] | Number of saved SIMD registers (d8–dN) minus 1 |
| `RegI` | [19:16] | Number of saved integer registers (x19–xN) minus 1 |
| `H` | [20] | Home-parameter-registers flag |
| `CR` | [22:21] | Chain/return: 00=no frame chain; 11=frame chain (x29 set) |
| `FrameSize` | [31:23] | Total frame size in 16-byte units |

A simple function that saves `{x29, x30}` and allocates 32 bytes on the stack can be represented entirely in the packed word. Functions with SEH require the unpacked form, which references a full `ExceptionDataRecord` in `.xdata`.

The `ExceptionDataRecord` on AArch64 uses ARM64-specific unwind byte opcodes from the `Win64EH::UnwindOpcodes` enum. The ARM64-specific opcodes include `UOP_AllocMedium`, `UOP_SaveR19R20X`, `UOP_SaveFPLR`, `UOP_SaveFPLRX`, `UOP_SaveReg`, `UOP_SaveRegX`, `UOP_SaveRegP`, `UOP_SaveRegPX`, `UOP_SetFP`, `UOP_AddFP`, `UOP_Nop`, `UOP_End`, and `UOP_PACSignLR` — all defined in
[`llvm/include/llvm/Support/Win64EH.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/Win64EH.h).

### Key Differences from x64 SEH

The fundamental architecture of Windows SEH is the same across x64 and AArch64: table-driven unwinding, `RUNTIME_FUNCTION` entries in `.pdata`, `UnwindInfo`/`ExceptionDataRecord` in `.xdata`, and the same personality function ABI. The differences are in the unwind opcode set and the packed-form optimization:

| Aspect | x64 | AArch64 |
|---|---|---|
| `.pdata` entry | `{BeginAddress, EndAddress, UnwindInfoOffset}` (3 × u32) | `{BeginAddress, UnwindData}` (2 × u32) |
| Packed unwind form | Not available; always references `.xdata` | Available for canonical prologues |
| Unwind opcode set | `UOP_PushNonVol`, `UOP_AllocLarge`, `UOP_SaveXMM128`, etc. | ARM64-specific opcodes (`UOP_SaveFPLR`, `UOP_SaveReg`, `UOP_SaveRegP`, etc.) |
| Frame pointer | Optional; `rbp` via `UOP_SetFPReg` | `x29` via `UOP_SetFP` / `UOP_AddFP`; `frame-pointer=reserved` by default |
| PAC signing | Not applicable | `UOP_PACSignLR` for PAC-RET |
| Return-address register | `rsp[0]` at function entry | `x30` (link register, LR) |

A key prologue difference is that `frame-pointer=reserved` is the default on AArch64 Windows (visible in the `attributes` section of the IR generated above). This means `x29` is always used as a frame pointer, which simplifies the unwinder's job: it can always trust `x29` to chain to the previous frame when `.pdata` data is absent or corrupt.

x64 always uses three-field `RUNTIME_FUNCTION` structs (begin, end, offset), never packing unwind data inline. AArch64 uses two-field structs where `Flags` selects packed or unpacked mode. The two-field `.pdata` entry is always 8 bytes vs. x64's 12 bytes — a saving that is independent of whether the packed form is used. When the packed form is used for qualifying functions, there is an additional saving: no `.xdata` record is emitted at all.

The unwind code array in x64 `.xdata` `UNWIND_INFO` is an array of `UnwindCode` unions (each 2 bytes). The AArch64 `.xdata` `ExceptionDataRecord` uses a compact byte-stream of opcodes — up to four opcodes per 32-bit word — with a different encoding table entirely. Both formats encode prologue operations in reverse order (epilogue-to-prologue) so the unwinder can apply them in reverse during stack walking.

### `__try` / `__except` / `__finally` Lowering on AArch64

The Clang AST nodes for `__try`, `__except`, and `__finally` are identical regardless of target. `MicrosoftCXXABI` handles their lowering; only the target-specific backend differs.

For `__finally`, Clang on all Windows targets uses `llvm.localescape` / `llvm.localaddress` / `llvm.localrecover` rather than funclet IR. The `__finally` block is outlined into a separate function (mangled as `?fin$N@M@enclosing_function@@`) that receives an `i8` flag (0 = normal exit, 1 = exception) and a frame pointer. The parent function calls the outlined block directly on the normal path:

```llvm
; IR for __finally on AArch64 (clang 22.1.6, aarch64-pc-windows-msvc)
define dso_local i32 @test_try_finally(i32 noundef %0) #0 {
  %3 = alloca i32, align 4               ; result variable
  call void (...) @llvm.localescape(ptr %3)
  ; ... __try body: result = x * 2 ...
  %6 = call ptr @llvm.localaddress()
  call void @"?fin$0@0@test_try_finally@@"(i8 noundef 0, ptr noundef %6)
  ; ^ normal-exit call: flag=0, frame pointer passed as 2nd arg
  %7 = load i32, ptr %3, align 4
  ret i32 %7
}

define internal void @"?fin$0@0@test_try_finally@@"(i8 noundef %0, ptr noundef %1) #1 {
  ; Recover the escaped variable via llvm.localrecover
  %5 = call ptr @llvm.localrecover(ptr @test_try_finally, ptr %1, i32 0)
  ; ... __finally body: result += 100 ...
  ret void
}
```

The `?fin$N@M@enclosing@@` naming follows MSVC's convention: `N` is the `__finally` block index within the function, `M` is a nesting depth. On AArch64 the outlined finalizer function accesses the escaped variable via the `?frame_escape_N` local symbol as a signed offset from the frame pointer:

```asm
; AArch64 assembly: finalizer accessing escaped variable (clang 22.1.6)
"?fin$0@0@test_try_finally@@":
.seh_proc "?fin$0@0@test_try_finally@@"
    sub  sp, sp, #16
    .seh_stackalloc 16
    .seh_endprologue
    ; Frame-escape offset encoded as a pair of movz/movk:
    movz  x9, #:abs_g1_s:.Ltest_try_finally$frame_escape_0
    movk  x9, #:abs_g0_nc:.Ltest_try_finally$frame_escape_0
    ; x1 = parent frame pointer passed as argument
    ldr  w8, [x1, x9]       ; load result from parent frame
    add  w8, w8, #100
    str  w8, [x1, x9]       ; store updated value
    .seh_startepilogue
    add  sp, sp, #16
    .seh_stackalloc 16
    .seh_endepilogue
    ret
    .seh_endfunclet
    .seh_endproc
```

The `?frame_escape_0` label has the value of the stack slot's offset within the parent frame. LLVM generates it as a relocatable expression, which the assembler resolves at link time.

For `__except`, with `-fasync-exceptions` (`/EHa`) the IR uses `llvm.seh.try.begin` / `llvm.seh.try.end` to delimit the guarded region, then a `catchswitch`/`catchpad` for the filter. The personality is `__C_specific_handler`:

```llvm
; IR for __except on AArch64 (-fasync-exceptions, clang 22.1.6)
define dso_local noundef i32 @"?safe_divide@@YAHHH@Z"(i32 noundef %0, i32 noundef %1)
    personality ptr @__C_specific_handler {
  invoke void @llvm.seh.try.begin()
          to label %7 unwind label %11
  ; ... __try body: result = a / b ...
  invoke void @llvm.seh.try.end()
          to label %19 unwind label %11

11:
  %12 = catchswitch within none [label %13] unwind to caller

13:
  %14 = catchpad within %12 [ptr null]   ; null = catch-all filter
  catchret from %14 to label %15

15:
  %16 = call i32 @llvm.eh.exceptioncode(token %14)
  ; __except body: result = 0
  ret i32 %18
}
```

Without `-fasync-exceptions`, the compiler treats asynchronous hardware faults (divide-by-zero, access violation) as unhandled C++ termination — the `__try`/`__except` wrapping has no effect on hardware exceptions from the optimizer's perspective. `-fasync-exceptions` forces `invoke` + `llvm.seh.try.begin/end` even for ordinary arithmetic so that the hardware exception dispatched by the OS unwind engine reaches the `__except` filter.

The resulting AArch64 assembly shows the `.seh_handler` directive followed by `.seh_handlerdata` containing the call-site table:

```asm
; AArch64 assembly with __C_specific_handler (clang 22.1.6)
"?safe_divide@@YAHHH@Z":
.seh_proc "?safe_divide@@YAHHH@Z"
    .seh_handler __C_specific_handler, @unwind, @except
    sub  sp, sp, #64
    .seh_stackalloc  64
    stp  x29, x30, [sp, #32]
    .seh_save_fplr   32
    add  x29, sp, #32
    .seh_add_fp      32
    .seh_endprologue
    ; ... __try body ...
    .seh_startepilogue
    ldp  x29, x30, [sp, #32]
    .seh_save_fplr   32
    add  sp, sp, #64
    .seh_stackalloc  64
    .seh_endepilogue
    ret
.seh_endfunclet
.seh_handlerdata
    .word  (.Llsda_end0-.Llsda_begin0)/16   ; number of call sites
.Llsda_begin0:
    .word  .Ltmp2@IMGREL    ; LabelStart  (start of __try body)
    .word  .Ltmp3@IMGREL    ; LabelEnd    (end of __try body)
    .word  1                ; CatchAll (constant filter = 1 means always handle)
    .word  .LBB0_2@IMGREL   ; ExceptionHandler (address of __except block)
.Llsda_end0:
    .text
    .seh_endproc
```

The `.seh_handlerdata` block is the language-specific data (`ExceptionData`) that `__C_specific_handler` reads at runtime. Each 16-byte entry is a scope record: guarded-region start RVA, guarded-region end RVA, filter expression (or constant 1 for `EXCEPTION_EXECUTE_HANDLER`), and handler target RVA. This format is identical between x64 and AArch64 — it is consumed by `__C_specific_handler` in the CRT, not by the OS unwinder directly.

### EH Personality: `__C_specific_handler` vs. `__CxxFrameHandler3`

The choice of personality function follows the same rule as on x64:

- `__C_specific_handler` — used for C-language SEH (`__try`/`__except`/`__finally`) with `-fasync-exceptions`. Also used when a C++ function contains only `__try`/`__except` without any C++ exception handling. The personality reads the handler table in `.seh_handlerdata` to match hardware and software exceptions.
- `__CxxFrameHandler3` — used for C++ `try`/`catch` blocks and RAII unwinding. The `catchpad` operands carry `TypeDescriptor` pointers rather than constant filter values; type matching is performed by `__CxxFrameHandler3`.

In Clang's implementation, the personality function is selected in `CGException.cpp` based on the active EH model and whether the function body contains C++ EH or only SEH constructs. The selection is target-independent; the AArch64 backend simply references whichever symbol Clang selected.

When a function mixes C++ `try`/`catch` with `__try`/`__except` — which MSVC permits under `/EHa` semantics — `__CxxFrameHandler3` is used for the entire function and is expected to dispatch SEH exceptions to C++ handlers where the filter expression permits.

### PAC-RET and the `UOP_PACSignLR` Unwind Opcode

Pointer Authentication Code return-address signing (`-mbranch-protection=pac-ret`) adds `PACIASP` (sign LR with SP) at the start of the prologue and `AUTIASP` (authenticate LR with SP) before `RET`. These instructions use `hint #25` and `hint #29` respectively on hardware that supports PAC; on hardware without PAC support the hints are NOPs, preserving backward compatibility.

The Windows unwind table must record PAC signing so that the unwinder can account for the signed return address when recovering the link register. LLVM 22 implements this via the `UOP_PACSignLR` opcode in the unwind byte stream and emits the corresponding `.seh_pac_sign_lr` assembler directive:

```asm
; AArch64 with -mbranch-protection=pac-ret (clang 22.1.6)
"?safe_divide@@YAHHH@Z":
.seh_proc "?safe_divide@@YAHHH@Z"
    .seh_handler __C_specific_handler, @unwind, @except
    hint  #25               ; PACIASP — sign LR with SP
    .seh_pac_sign_lr        ; → UOP_PACSignLR in unwind byte stream
    sub   sp, sp, #64
    .seh_stackalloc 64
    stp   x29, x30, [sp, #32]
    .seh_save_fplr  32
    add   x29, sp, #32
    .seh_add_fp     32
    .seh_endprologue
    ; ... function body ...
    .seh_startepilogue
    ldp   x29, x30, [sp, #32]
    .seh_save_fplr  32
    add   sp, sp, #64
    .seh_stackalloc 64
    hint  #29               ; AUTIASP — authenticate LR before return
    .seh_pac_sign_lr        ; epilogue copy of UOP_PACSignLR
    .seh_endepilogue
    ret
.seh_endfunclet
.seh_handlerdata
    ; ... call-site table ...
    .seh_endproc
```

The `UOP_PACSignLR` opcode is defined in `Win64EH::UnwindOpcodes` (the same enum used for all Windows ARM64 opcodes) in
[`llvm/include/llvm/Support/Win64EH.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/Win64EH.h#L65).
When the OS unwinder encounters `UOP_PACSignLR` while replaying the prologue in reverse, it knows that the return address stored in the frame was signed with PAC and must authenticate it before using it as a return address. This allows the unwinder to correctly handle the saved LR value during stack walking even when the hardware has not yet executed `AUTIASP`.

BTI (Branch Target Identification) landing pads (`hint #34` = `BTI c`) are inserted by the backend at function entry when `-mbranch-protection=bti` is active, but BTI is orthogonal to unwind data — the OS unwinder does not need to record or replay BTI hints. The unwind table change for PAC (`UOP_PACSignLR`) is the only security-feature-specific unwind opcode in LLVM 22's AArch64 Windows support.

`ARM64EC` (the x64-emulation-compatible ABI, handled by `ARM64ECCXXABIInfo` in `clang/lib/CodeGen/TargetInfo.cpp`) uses a modified unwind format compatible with both the ARM64 and x64 unwinders. BTI landing pads (`hint #34` = `BTI c`) and PAC signing interact with ARM64EC's dual-unwinder requirements, though the details depend on function classification (EC entry thunk vs. native AArch64 function) and are outside the scope of this section.

---

## `__declspec` Extensions

### `dllimport` / `dllexport`

`__declspec(dllexport)` causes Clang to emit the function with `dllexport` linkage in IR:

```llvm
define dso_local dllexport noundef i32 @"?exported_func@@YAHH@Z"(i32 %0) ...
```

`__declspec(dllimport)` emits the declaration with `dllimport`:

```llvm
declare dllimport noundef i32 @"?imported_func@@YAHH@Z"(i32 noundef) ...
```

The x86-64 backend translates `dllimport` into an `__imp_`-prefixed indirect load from the Import Address Table. The linker resolves the IAT entry at link time.

### `selectany` → `linkonce_odr` + comdat

`__declspec(selectany)` on a variable allows multiple definitions to coexist; the linker picks one. Clang maps this to `weak_odr` linkage with a `comdat any` entry:

```llvm
$"?shared_var@@3HA" = comdat any

@"?shared_var@@3HA" = weak_odr dso_local global i32 42, comdat, align 4
```

The `comdat any` directive instructs the linker to retain exactly one copy across all object files. This is the mechanism underlying C++ inline variables and template static data members on Windows.

### `thread` → TLS via `thread_local`

`__declspec(thread)` maps to LLVM's `thread_local` storage class:

```llvm
@"?tls_var@@3HA" = dso_local thread_local global i32 0, align 4
```

Access uses the `llvm.threadlocal.address.p0` intrinsic to obtain the thread-local pointer:

```llvm
%5 = call align 4 ptr @llvm.threadlocal.address.p0(
    ptr align 4 @"?tls_var@@3HA")
store i32 %4, ptr %5, align 4
```

On Windows, this lowers to TEB-based TLS slot access through the `_tls_index` mechanism or `__declspec(thread)` static TLS.

### `align(N)` → IR Alignment

`__declspec(align(N))` maps to `align N` on the alloca:

```llvm
%3 = alloca %struct.AlignedStruct, align 64
```

This propagates through to backend alignment requirements without any special MSVC ABI machinery.

### `naked` → IR `naked` Attribute

`__declspec(naked)` suppresses prologue/epilogue generation. Clang emits the `naked` function attribute in LLVM IR, which the backend honors by omitting all frame setup and teardown code.

### `restrict` → `noalias`

`__declspec(restrict)` on a function return value (like `malloc`-family functions) maps to `noalias` on the return value in IR, enabling alias analysis to treat the result as a fresh allocation.

---

## MSVC `operator new` and `operator delete`

### Global Allocator Mangling

On 64-bit Windows, the global placement operators mangle as:

```
??2@YAPEAX_K@Z    → void* operator new(size_t)
??3@YAXPEAX_K@Z   → void operator delete(void*, size_t)
??2@YAPEAX_KAEBUnothrow_t@std@@@Z  → placement new (nothrow)
```

The `_K` encodes `size_t` (which is 64-bit on Win64). Clang emits these as declarations and calls them directly:

```llvm
declare dso_local noundef nonnull ptr @"??2@YAPEAX_K@Z"(i64 noundef) #1
declare dso_local void @"??3@YAXPEAX_K@Z"(ptr noundef, i64 noundef) #3

; In create_widget:
%5 = call noalias noundef nonnull ptr @"??2@YAPEAX_K@Z"(i64 noundef 16) #5
```

Note that on Windows, the sized `delete` (`operator delete(void*, size_t)`) is the default rather than the unsized variant — this is the `??3@YAXPEAX_K@Z` signature with two parameters.

### Array `new` Cookie

Windows array allocation differs from Itanium's hidden cookie placed before the array elements. On MSVC, array `new` places an `i64` cookie containing the element count at `ptr - 8`, before the first element. The cookie is written by the compiler-generated array construction loop and read by `operator delete[]` to determine how many destructors to call. `MicrosoftCXXABI::EmitCXXDeleteExpr()` generates the back-step to find the cookie and the loop that calls destructors before forwarding to `operator delete`.

The cookie stores the raw element count (not a byte size), since MSVC uses the vector deleting destructor (`??_E`) which already knows the element size via the destructor's knowledge of its class. The cookie layout:

```
ptr - 8:   i64 element_count   ← cookie
ptr + 0:   Widget[0]           ← first element
ptr + S:   Widget[1]
...
```

Where `S` is `sizeof(Widget)`. The `operator delete[]` call path is:

```llvm
; array delete reads cookie:
%cookie_ptr = getelementptr inbounds i8, ptr %arr_ptr, i64 -8
%count = load i64, ptr %cookie_ptr
; loop: call destructor for each element in reverse order
; then call operator delete on (arr_ptr - 8)
call void @"??3@YAXPEAX_K@Z"(ptr %alloc_base, i64 %total_size)
```

For trivially-destructible types, no cookie is written, and `operator delete[]` is called directly on the original pointer. This differs from Itanium, which always writes a cookie for non-trivially-destructible arrays regardless of whether it is needed.

---

## clang-cl Compatibility and Divergences

### Areas of Exact Compatibility

Clang-cl (invoked as `clang --driver-mode=cl` or the `clang-cl` binary) reproduces MSVC behavior precisely in:

- Name mangling (all mangled symbols are binary-compatible with MSVC-compiled objects)
- vtable layout and RTTI data structures (cross-linking MSVC and Clang objects in the same binary works)
- Member pointer representation (including the inheritance-model-dependent sizing)
- `__CxxFrameHandler3` usage and `.xdata` unwind table format
- `_CxxThrowException` / `__RTDynamicCast` external calls
- `__declspec` attribute semantics

### Intentional Divergences

Several areas where clang-cl intentionally differs from MSVC:

1. **Diagnostic quality**: clang-cl produces superior error messages and warnings.
2. **C++ standard conformance**: Clang tracks the C++ standard more closely; `-fms-extensions` enables most MSVC non-standard extensions but some edge cases differ.
3. **`__forceinline`**: Treated as `__attribute__((always_inline))`; Clang may not inline in all cases where MSVC does.
4. **Integer overflow**: MSVC historically treated signed overflow as defined in some optimization contexts; Clang follows the C++ standard (UB).
5. **Template instantiation order**: Minor differences in when implicit instantiations occur.

### `-fms-compatibility-version`

The flag `-fms-compatibility-version=X.Y.Z` sets the `_MSC_VER` and `_MSC_FULL_VER` macros and adjusts ABI compatibility for that MSVC version. Clang 22 defaults to emulating MSVC 19.33.0 (Visual Studio 2022 17.3). This version number affects:

- The target triple (`x86_64-pc-windows-msvc19.33.0`)
- Predefined macros consumed by Windows SDK headers
- Layout decisions that changed between MSVC versions (empty base optimization variants, `__declspec(novtable)` interactions)

### `__declspec(novtable)` Optimization

`__declspec(novtable)` is a Microsoft extension that suppresses vtable pointer initialization in the constructor of an abstract class. Since the class is never instantiated directly (only via derived classes), storing a vfptr pointing to a vtable full of pure virtual slots wastes cycles. Clang honors this annotation by omitting the `store ptr @"??_7Abstract@@6B@", ptr %vfptr_field` instruction from the abstract class's constructor body. The derived class constructor still initializes the vfptr to the derived class's vtable. This is a pure codegen optimization with no ABI impact — the vtable data still exists because it may be needed for partial construction, but the abstract class constructor just doesn't write it.

### `/EHsc`, `/EHs`, `/EHa`

The EH mode flag affects what Clang generates for exception handling:

| Flag | Clang behavior |
|---|---|
| `/EHsc` | C++ EH only; `extern "C"` functions are `nounwind`; SEH not propagated through C++ destructors |
| `/EHs` | C++ EH; `extern "C"` may throw (no `nounwind`); conservative |
| `/EHa` | Asynchronous EH; hardware exceptions (access violations, etc.) can trigger C++ destructors; generates `uwtable` on every function |
| `/EHs-` or no `/EH` | No EH; `noexcept` on all functions |

Clang maps these through `LangOptions::ExceptionHandling` (set in `clang/lib/Frontend/CompilerInvocation.cpp`) to the appropriate IR attributes and personality function selection.

### ARM64 Windows

The `ARM64ECCXXABIInfo` class (in `clang/lib/CodeGen/TargetInfo.cpp`) handles the ARM64EC (Emulation Compatible) variant used for x64 code running on ARM64 Windows. ARM64EC uses MSVC ABI name mangling and calling conventions but targets ARM64 instruction encoding. The regular `ARM64` Windows target uses standard ARM64 calling conventions with MSVC C++ ABI semantics. `MicrosoftCXXABI` is shared across all Windows targets; only the `TargetInfo` ABI info layer differs.

### Guard Variables for Static Local Initialization

MSVC uses a different static-local initialization guard mechanism from Itanium. In Itanium, `__cxa_guard_acquire`/`__cxa_guard_release` protect function-local statics. In MSVC, the guard variable is a `i32` flag where the runtime calls `_Init_thread_header`, `_Init_thread_footer`, and `_Init_thread_abort` (thread-safe static initialization, introduced in VS 2015) or uses simple `i32` flag checks for pre-2015 compatibility mode.

For `-fms-compatibility-version=19.00` and newer (the default in Clang 22), Clang generates:

```llvm
; Static local: first call initializes
@"?val@?1??get_static@@YAHXZ@4HA" = internal global i32 0, align 4
@"??_B?1??get_static@@YAHXZ@5" = internal global i32 0, align 4  ; guard

; On first call:
%g = load i32, ptr @"??_B?1??get_static@@YAHXZ@5"
%done = icmp ne i32 %g, 0
br i1 %done, label %initialized, label %need_init
need_init:
  call void @_Init_thread_header(ptr @"??_B?1??get_static@@YAHXZ@5")
  ; if still need init (thread won the race):
  ; ... initialize val ...
  call void @_Init_thread_footer(ptr @"??_B?1??get_static@@YAHXZ@5")
```

The guard symbol `??_B` encoding is distinct from Itanium's `_ZGVN` encoding. The `5` suffix in the guard name is a static count discriminator to distinguish multiple statics within the same function.

### `__declspec(uuid)` and COM GUIDs

`__declspec(uuid("..."))` attaches a GUID to a class, enabling COM interoperability via `__uuidof(T)`. Clang emits the GUID as LLVM metadata attached to the LLVM type representing the C++ class:

```llvm
!0 = !{!"Foo", !"12345678-1234-1234-1234-1234567890AB"}
```

The `MicrosoftCXXABI` implements `GetAddrOfUUID()` to emit a global constant `{i32, i16, i16, [8 x i8]}` containing the GUID bytes, and `__uuidof(T)` is lowered to a reference to this constant. The GUID constant uses `linkonce_odr` linkage so multiple translation units referencing `__uuidof(Foo)` coalesce.

---

## Research and Development Roadmap

> *Horizon dates are relative to April 2026.*

### 6-Month Horizon (Near-Term, by ~October 2026)

- **ARM64EC thunk generation improvements**: Active work on `ARM64ECCXXABIInfo` to reduce thunk overhead at the x64↔AArch64 boundary; patches under review on LLVM Discourse to unify the EC entry-thunk and exit-thunk emission paths in `MicrosoftCXXABI` and `TargetInfo.cpp` ([llvm-dev thread, March 2026](https://discourse.llvm.org/)).
- **`__CxxFrameHandler4` personality adoption**: MSVC shipped `__CxxFrameHandler4` (a compressed `.xdata` format) in VS 2019 16.x; clang-cl currently defaults to `__CxxFrameHandler3` for compatibility. Active Discourse discussion on when clang-cl should switch defaults, contingent on `vcruntime140_1.dll` availability checks in CMake and `-fms-compatibility-version` gating.
- **`UOP_PACSignLR` test coverage and CI hardening**: Follow-on patches to the LLVM 22 `UOP_PACSignLR` implementation are closing gaps in the Windows-on-AArch64 EH test matrix; the `llvm/test/CodeGen/AArch64/wineh-*.ll` suite is being expanded to cover mixed PAC-RET + SEH filter scenarios.
- **`_Init_thread_header` fast-path elimination**: Work in progress to switch thread-safe static-local initialization from `_Init_thread_header`/`_Init_thread_footer` to the faster `std::once`-style futex path available in newer Windows CRT builds, gated behind a new `-fms-compatibility-version` threshold.

### 2.5-Year Horizon (Mid-Term, by ~October 2028)

- **`MicrosoftMangle.cpp` C++26 conformance**: C++26 contracts, reflection (`std::meta`), and deducing `this` introduce new mangling requirements that MSVC is specifying in its C++ ABI extension documents; `MicrosoftCXXNameMangler` will need new encoding rules for `auto this`, explicit object parameters, and splice expressions — a significant extension to `mangleTemplateArgs()` and the back-reference table logic.
- **Windows CHPE (Compiled Hybrid Portable Executable) full codegen support**: CHPE binaries contain both x64 and ARM64 code sections with metadata tables directing the OS loader; `ARM64ECCXXABIInfo` currently generates one code flavor per compilation. Full CHPE codegen — emitting both flavors in one compilation unit with the required `HYBRID_PE_METADATA` section — is on the LLVM roadmap, requiring new linker plugin hooks and `lld-link` CHPE layout passes.
- **Refined `__unspecified_inheritance` member pointer narrowing**: When a forward-declared class later gains a definition revealing a narrower inheritance model, clang-cl emits suboptimal conversion thunks at every use site. Planned work (tracked in LLVM GitHub issue #65412 area) aims to narrow already-emitted member pointer uses via LTO-time specialization of the 16-byte catch-all representation to the exact width demanded by the resolved inheritance model.
- **`__CxxFrameHandler3` personality elision under LLVM 22 middle-end**: Ongoing collaboration between the WinEH and middle-end teams to let `SimplifyCFG` and `EarlyCSE` fold trivially-empty `cleanuppad` funclets (those containing only a single destructor call on a trivially-destructible type), replacing them with `nounwind` attribute propagation and reducing `.xdata` table size in release builds.

### 5-Year Horizon (Long-Term, by ~2031)

- **MSVC ABI stabilization for C++ modules (named module interface units)**: With C++20 modules now shipping in MSVC and clang-cl, the binary module interface (BMI) format still uses MSVC-proprietary serialization. Long-term, an interoperable BMI format negotiated between Clang, MSVC, and ICC may require extending `MicrosoftMangle.cpp` with module-unit scoping encodings and a new template-argument deduplication scheme across BMI boundaries.
- **Verified ABI compatibility via formal specification**: Research efforts (e.g., building on the Alive2 / CompCert methodology applied to ABI lowering) aim to formally verify that `MicrosoftCXXABI`'s member pointer arithmetic, vtable pointer stores, and EH table generation are observationally equivalent to MSVC output across all supported inheritance models; this would provide a mechanized proof target for the `EmitMemberPointerConversion()` and `EmitVBPtrInit()` code paths.
- **Hardware-enforced stack isolation under Windows CET + PAC**: Intel CET (`ENDBR64`) is already handled by LLVM's x86 backend; combining CET with ARM64 PAC-RET and BTI under Windows' Hardware-Enforced Stack Protection (`/CETCOMPAT`) will require coordinated changes in `MicrosoftCXXABI` (for `cleanuppad` funclet return-address handling), the WinEH preparation pass, and the AArch64 prologue emitter to ensure authenticated return addresses remain valid across funclet boundaries without disabling PAC on the funclet entry paths.

---

## Summary

- The `MicrosoftCXXABI` class in `clang/lib/CodeGen/MicrosoftCXXABI.cpp` implements all Windows-specific C++ ABI lowering; name mangling is handled separately by `MicrosoftCXXNameMangler` in `clang/lib/AST/MicrosoftMangle.cpp`.
- MSVC name mangling uses `?`-prefix encoding with reverse-scope ordering; constructors encode as `??0`, destructors as `??1`, with operator-specific codes for `new`/`delete` and overloaded operators.
- MSVC vtables (`vftables`) contain no offset-to-top; the `CompleteObjectLocator` lives at `vtable[-1]`; virtual bases require separate `vbtables` and additional `vbptr` fields in each object.
- Constructors take a `should_call_vbase_constructors` `i32` parameter rather than compiling separate C1/C2 variants; destructors split into `??1` (base) and `??_G`/`??_E` (deleting) variants.
- Windows EH uses the `__CxxFrameHandler3` personality with `catchpad`/`cleanuppad`/`catchswitch` funclet IR; throwing calls `_CxxThrowException` and the `ThrowInfo`/`CatchableTypeArray` structures live in `.xdata`.
- RTTI uses `TypeDescriptor`, `BaseClassDescriptor`, `ClassHierarchyDescriptor`, and `CompleteObjectLocator`; `dynamic_cast` calls `__RTDynamicCast`.
- Member pointer size varies by inheritance model: 4 bytes (single), 8 bytes (multiple), 12 bytes (virtual), 16 bytes (unknown).
- `__declspec` attributes map directly to IR attributes: `dllexport`/`dllimport` → IR visibility; `selectany` → `weak_odr` + `comdat any`; `thread` → `thread_local` + `llvm.threadlocal.address`.
- clang-cl is binary-compatible with MSVC at the ABI level; the `-fms-compatibility-version` flag controls `_MSC_VER` and per-version layout choices.

---

## Cross-References

- [Chapter 26 — Exception Handling](../part-04-llvm-ir/ch26-exception-handling.md) — WinEH funclet IR: `catchpad`, `cleanuppad`, `catchswitch`, `catchret`, `cleanupret`; `llvm.localescape` / `llvm.localrecover` for `__finally` outlined blocks
- [Chapter 28 — The Clang Driver](../part-05-clang-frontend/ch28-clang-driver.md) — clang-cl driver mode, `/EHsc` flag processing, target triple selection
- [Chapter 41 — Calls, the ABI Boundary, and Builtins](../part-06-clang-codegen/ch41-calls-abi-builtins.md) — MSVC calling conventions, `__thiscall`, `__vectorcall`, parameter passing
- [Chapter 42 — C++ ABI Lowering: Itanium](../part-06-clang-codegen/ch42-cpp-abi-lowering-itanium.md) — Itanium ABI for contrast: C1/C2 constructors, D0/D1/D2 destructors, `landingpad`, `__gxx_personality_v0`
- [Chapter 96 — The AArch64 Backend](../part-15-targets/ch96-the-aarch64-backend.md) — AArch64 frame lowering, prologue/epilogue emission, PAC-RET and BTI code generation, `frame-pointer=reserved` semantics

## Reference Links

- [`clang/lib/CodeGen/MicrosoftCXXABI.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/MicrosoftCXXABI.cpp) — primary MSVC ABI codegen implementation
- [`clang/lib/AST/MicrosoftMangle.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/AST/MicrosoftMangle.cpp) — `MicrosoftCXXNameMangler`, `MicrosoftMangleContextImpl`
- [`clang/lib/CodeGen/CGCXXABI.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGCXXABI.h) — `CGCXXABI` abstract interface
- [`llvm/lib/CodeGen/WinEHPrepare.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/WinEHPrepare.cpp) — WinEH funclet preparation pass
- [`llvm/lib/Target/X86/X86WinEHState.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/X86/X86WinEHState.cpp) — x86 32-bit EH state table generation
- [`llvm/include/llvm/Support/ARMWinEH.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/ARMWinEH.h) — `RuntimeFunctionARM64`, `ExceptionDataRecord`, `EpilogueScope` structs for ARM/AArch64 Windows EH
- [`llvm/include/llvm/Support/Win64EH.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/Win64EH.h) — `UnwindOpcodes` enum including `UOP_PACSignLR` and all AArch64 opcodes
- [Microsoft ARM64 exception handling documentation](https://docs.microsoft.com/en-us/cpp/build/arm64-exception-handling) — `.pdata` packed format, `.xdata` unwind byte codes, epilogue scope encoding
- [Microsoft C++ ABI documentation (Itanium-to-MSVC comparison)](https://github.com/MicrosoftDocs/cpp-docs/blob/main/docs/cpp/exception-specifications-throw-cpp.md)
- [MSVC Decorated Names specification](https://learn.microsoft.com/en-us/cpp/build/reference/decorated-names)
- [clang/docs/MSVCCompatibility.rst](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/docs/MSVCCompatibility.rst) — official documentation of clang-cl divergences from MSVC


---

@copyright jreuben11
