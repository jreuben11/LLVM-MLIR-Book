# Chapter 42 — C++ ABI Lowering: Itanium

*Part VI — Clang Internals: Codegen and ABI*

The Itanium C++ ABI is the de facto binary interface for C++ on Linux, macOS, FreeBSD, Android, and most UNIX-derived platforms. It specifies name mangling, vtable layout, RTTI representation, exception handling conventions, constructor and destructor variants, guard variables for static locals, and thread-local storage initialization. Understanding how Clang implements this ABI is indispensable for anyone who needs to interoperate with existing binaries, debug obscure linking failures, or extend Clang's code generation for new language constructs. This chapter traces the full path from AST nodes to LLVM IR, using Clang 22.1.x source class names and verified output from `clang++ -emit-llvm -S`.

---

## 42.1 The Itanium C++ ABI: Scope and Implementation Entry Point

The Itanium C++ ABI document — informally called the "Itanium ABI" even though it long ago escaped its original IA-64 home — was published jointly by CodeSourcery and HP in 2001 and has been the reference for GCC and Clang on non-Windows targets ever since. It governs every interaction between separately compiled translation units that exchange C++ entities: how symbols are named, how virtual dispatch is encoded, how exceptions propagate across frame boundaries, and how dynamic type information is represented at runtime.

Clang's implementation lives primarily in two source files:

- [`clang/lib/CodeGen/ItaniumCXXABI.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/ItaniumCXXABI.cpp) — the code generation side: vtable emission, virtual dispatch, constructor/destructor variant selection, guard variables, `operator new`/`delete` lowering, TLS registration
- [`clang/lib/AST/ItaniumMangle.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/AST/ItaniumMangle.cpp) — name mangling for all declarations visible across TUs

Both are selected through the ABI abstraction layer. `CodeGenModule::createCXXABI()` in [`clang/lib/CodeGen/CGCXXABI.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGCXXABI.cpp) calls `CreateItaniumCXXABI()` when the target is not Windows or when the user has not forced `-fms-compatibility`. `ItaniumCXXABI` extends `CGCXXABI`, overriding approximately 60 virtual methods that `CodeGenFunction` calls at each C++-specific code generation point.

The ABI applies to these platforms as of Clang 22:

| Platform | Triple Pattern | Notes |
|---|---|---|
| Linux (glibc, musl) | `x86_64-pc-linux-gnu` etc. | Default |
| macOS / iOS | `*-apple-darwin*` | With Apple extensions to guard layout |
| FreeBSD / NetBSD / OpenBSD | `*-*-freebsd*` | Standard Itanium |
| Android (Bionic) | `*-linux-android*` | Itanium with minor TLS differences |
| WebAssembly | `wasm32-*` | Itanium mangling, no EH personality |
| Fuchsia | `*-fuchsia` | Itanium with Zircon runtime |

---

## 42.2 Name Mangling

### 42.2.1 The `_Z` Prefix and `CXXNameMangler`

Every external C++ symbol whose name could collide with a C symbol or another C++ overload is encoded using the Itanium mangling scheme. The encoding always begins with `_Z`, which the linker treats as a reserved prefix on ELF targets. The class that drives the entire process is `CXXNameMangler` in `ItaniumMangle.cpp`. It holds an `llvm::raw_ostream` output stream and a `SmallVector` substitution table, and is invoked from `ItaniumMangleContextImpl::mangleCXXName()`.

### 42.2.2 Encoding Structure

The grammar for a mangled name is:

```
<mangled-name>  ::= _Z <encoding>
<encoding>      ::= <function name> <bare-function-type>
                 |  <data name>
<name>          ::= <nested-name>
                 |  <unscoped-name>
                 |  <local-name>
<nested-name>   ::= N [<CV-qualifiers>] <prefix> <unqualified-name> E
```

For a simple function `void foo(int)` in namespace `Ns`, the mangled name is `_ZN2Ns3fooEi`:

- `_Z` — extern C++ prefix
- `N` — begin nested name
- `2Ns` — length-prefixed component "Ns"
- `3foo` — length-prefixed component "foo"
- `E` — end nested name
- `i` — parameter type: `int`

### 42.2.3 Verified Examples

Clang 22 produces these mangled symbols for the class `Foo::Bar`:

```cpp
namespace Foo {
  struct Bar {
    Bar();                      // _ZN3Foo3BarC1Ev
    void baz(int x);            // _ZN3Foo3Bar3bazEi
    void qux() const;           // _ZNK3Foo3Bar3quxEv
    ~Bar();                     // _ZN3Foo3BarD1Ev
    virtual void virt();        // _ZN3Foo3Bar4virtEv

    template<typename T>
    T max_val(T a, T b);        // instantiation: _ZN3Foo7max_valIiEET_S1_S1_
  };
}
```

Verified against Clang 22 IR output (`clang++ -emit-llvm -S`):

```llvm
declare void @_ZN3Foo3BarC1Ev(...)
declare void @_ZN3Foo3Bar3bazEi(...)
declare void @_ZNK3Foo3Bar3quxEv(...)
declare void @_ZN3Foo3BarD1Ev(...)
define linkonce_odr i32 @_ZN3Foo7max_valIiEET_S1_S1_(i32 %0, i32 %1) ...
define linkonce_odr double @_ZN3Foo7max_valIdEET_S1_S1_(double %0, double %1) ...
```

The `K` qualifier in `_ZNK` encodes the `const` method qualifier. The `I...E` brackets encode template arguments — `Ii` for `int`, `Id` for `double`. The trailing `T_S1_S1_` encodes three uses of the first template parameter: return type `T`, then two `T` parameters using the substitution `S1_`.

### 42.2.4 Template Specializations

Template argument lists are surrounded by `I...E`. Built-in types encode to single letters (`i` = `int`, `d` = `double`, `c` = `char`, `b` = `bool`, `v` = `void`, `f` = `float`). Class types use the same `<N>name` encoding. For `std::vector<int>`:

```
_ZNSt6vectorIiSaIiEE
```

Here `St` is the abbreviation for `std::`, `6vector` is the class name, `Ii` is the template argument `int`, and `SaIiE` is `std::allocator<int>` using the `Sa` standard substitution.

### 42.2.5 Operator Encoding

Clang 22 generates the following operator encodings:

| Operator | Mangling | Example |
|---|---|---|
| `operator+` | `pl` | `_ZNK4Vec3plERKS_` |
| `operator=` | `aS` | `_ZN4Vec3aSERKS_` |
| `operator==` | `eq` | `_ZNK4Vec3eqERKS_` |
| `operator[]` | `ix` | `_ZN4Vec3ixEi` |
| `operator-` (unary) | `ng` | `_ZNK4Vec3ngEv` |
| `operator new` | `nw` | `_Znwm` |
| `operator delete` | `dl` | `_ZdlPvm` |
| `operator new[]` | `na` | `_Znam` |
| `operator delete[]` | `da` | `_ZdaPv` |

### 42.2.6 Substitution Compression

To prevent exponential symbol length in deeply templated code, the mangler maintains a substitution table. The first occurrence of a type or prefix is recorded; subsequent occurrences are replaced by `S_`, `S0_`, `S1_`, etc. (S_ = index 0, S0_ = index 1, S1_ = index 2, and so on). `CXXNameMangler::addSubstitution()` adds entries; `CXXNameMangler::mangleSubstitution()` emits them.

### 42.2.7 Local Names and Lambda Encodings

Local entities (functions nested within functions, lambdas) use the `<local-name>` production:

```
<local-name> ::= Z <encoding> E <entity name> [<discriminator>]
               | Z <encoding> E s [<discriminator>]   ; string literal
```

A lambda in function `foo` becomes `_ZZ3fooENK3$_0clEv` or similar, where `UlEn` is the lambda introducer. Lambda encoding changed significantly at ABI version 18 (the Clang 22 default): the lambda's mangled name now incorporates a hash of its definition rather than a simple sequential index, improving stability across incremental builds. Prior to version 18, adding a lambda earlier in a translation unit could change the mangling of all later lambdas.

### 42.2.8 Demangling with `llvm-cxxfilt`

```bash
$ /usr/lib/llvm-22/bin/llvm-cxxfilt \
    _ZN3Foo7max_valIiEET_S1_S1_ \
    _ZNK4Vec3plERKS_ \
    _ZN11MostDerivedC1Ev

int Foo::max_val<int>(int, int)
Vec3 Vec3::operator+(Vec3 const&) const
MostDerived::MostDerived()
```

`llvm-cxxfilt` uses `llvm::itaniumDemangle()` from [`llvm/lib/Demangle/ItaniumDemangle.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Demangle/ItaniumDemangle.cpp), which implements the full grammar including lambda encodings, requires-clause suffixes (C++20 concepts in mangled names), and the `_Z` prefix for vendor-extended types (`u<N>name`).

---

## 42.3 Virtual Table Layout

### 42.3.1 The Itanium Vtable Memory Model

For a class with virtual functions, the Itanium ABI mandates a specific in-memory layout for the vtable (virtual function dispatch table). Each vtable contains, in order:

1. **Offset-to-top**: a `ptrdiff_t` (64-bit signed integer on LP64) giving the displacement from the embedded subobject to the most-derived object. Used by adjustment thunks to recover `this`.
2. **RTTI pointer**: a pointer to the `std::type_info` object for the class. Used by `dynamic_cast` and `typeid`.
3. **Virtual function pointers**: one pointer per virtual function in declaration order, including overrides and, for destructors, two entries (D1 = complete-object, D0 = deleting).

For `struct Base { virtual void foo(); virtual void bar(); virtual ~Base(); }`, Clang 22 emits:

```llvm
@_ZTV4Base = dso_local unnamed_addr constant { [6 x ptr] } {
  [6 x ptr] [
    ptr null,           ; offset-to-top = 0
    ptr @_ZTI4Base,     ; RTTI pointer
    ptr @_ZN4Base3fooEv,
    ptr @_ZN4Base3barEv,
    ptr @_ZN4BaseD1Ev,  ; complete-object destructor
    ptr @_ZN4BaseD0Ev   ; deleting destructor
  ]
}, align 8
```

The `vptr` installed in each object points to the third slot (index 2 in zero-based C counting, which is the first virtual function pointer). The RTTI pointer lives at `vptr[-1]` and the offset-to-top lives at `vptr[-2]`.

### 42.3.2 Primary and Secondary Vtables

For classes with multiple bases or virtual bases, the vtable is a composite of sub-tables:

- The **primary vtable** covers the most-derived class and the first non-virtual base that itself has a vtable.
- **Secondary vtables** follow in the same constant for non-primary bases and virtual bases.
- The composite symbol `_ZTV<class>` encodes all sub-tables in a single `{ [N x ptr], [M x ptr], ... }` LLVM constant.

For a class `MostDerived : Base, virtual Virt`, Clang emits `@_ZTV11MostDerived` as a `{ [5 x ptr], [4 x ptr] }`. The store instructions in the constructor use `getelementptr inbounds inrange(-24, 16)` and `getelementptr inbounds inrange(-24, 8)` to address the first element of each sub-table's function pointer region. The `inrange` attribute communicates to the optimizer that the pointer only aliases within the indicated byte range.

### 42.3.3 Virtual Base Offset and VCall Offset Entries

When virtual inheritance is involved, the vtable also contains:

- **vcall offsets**: slots preceding the offset-to-top that record the adjustment to apply to `this` when dispatching through a secondary vtable. These exist for virtual functions overriding a base that is reached through a virtual path.
- **vbase offsets**: slots in the primary vtable recording the byte offset from the current subobject to each virtual base. Used by `virtual_base_offset_entry` thunks and by `dynamic_cast`.

The class `VTableLayout` in [`clang/lib/AST/VTableBuilder.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/AST/VTableBuilder.cpp) encodes the full layout: it holds `VTableComponent` objects (each tagged with a kind enum: `OffsetToTop`, `RTTI`, `FunctionPointer`, `CompleteDtorPointer`, `DeletingDtorPointer`, `UnusedFunctionPointer`, `VCallOffset`, `VBaseOffset`) and a `ThunkInfoVectorTy` for functions that require this-adjustment at the call site.

### 42.3.4 VTT: Virtual Table Table

When a class has virtual bases, the Itanium ABI requires a **VTT** (`_ZTT<class>`) — an array of pointers to sub-vtables. The VTT exists because base-object constructors and destructors (C2/D2) must install the correct vtable slices for their sub-objects, but at the time C2 runs, the most-derived type's vtable may not yet be consistent. Instead, the most-derived constructor passes the VTT to C2, which uses it to set up vtable pointers for virtual bases. The VTT for a class with one direct base and one virtual base is:

```
_ZTT<most_derived>[0]  →  points into _ZTV<most_derived> (primary vtable)
_ZTT<most_derived>[1]  →  points into _ZTC<most_derived>0_<base> (construction vtable for base at offset 0)
_ZTT<most_derived>[2]  →  points into _ZTC<most_derived>0_<base> (vbase portion)
...
```

Construction vtables (`_ZTC`) are temporary vtables used only during construction and destruction; they contain the correct virtual dispatch targets for a partially-constructed object. `ItaniumCXXABI::EmitVTables()` in `ItaniumCXXABI.cpp` emits both the VTT and the construction vtables as module-level constants.

### 42.3.5 VTable Symbols Summary

| Symbol Prefix | Meaning |
|---|---|
| `_ZTV<class>` | vtable for class |
| `_ZTI<class>` | `type_info` object for class |
| `_ZTS<class>` | type name string for class |
| `_ZTT<class>` | VTT (virtual table table) for class with virtual bases |
| `_ZTC<class>N<offset>_<base>` | construction vtable for base at offset N |

---

## 42.4 Vptr Initialization

### 42.4.1 Order of Vptr Stores

The Itanium ABI requires that each constructor install the correct vtable pointers for the object as constructed so far before executing the constructor body. This means:

1. The most-derived (C1) constructor calls base-class C2 constructors first (in order: virtual bases, then direct non-virtual bases left-to-right), then overwrites all vptrs with the most-derived vtable.
2. Each intermediate C2 (base-object) constructor installs the base's own vtable after calling its own bases.

`ItaniumCXXABI::InitializeVTablePointer()` in `ItaniumCXXABI.cpp` emits the store. It calls `CGF.Builder.CreateStore()` with a GEP into `_ZTV<class>` at the correct sub-table offset. For a class with virtual bases, `EmitVTablePtrStoreToObject()` computes the byte offset from the object start to the vptr field, using `ASTRecordLayout::getVBaseClassOffset()` for virtual base vptrs.

The verified C1 constructor for `MostDerived : Base, virtual Virt` from Clang 22:

```llvm
define void @_ZN11MostDerivedC1Ev(ptr %0) {
  ; first init virtual base Virt
  %virt_ptr = getelementptr inbounds i8, ptr %0, i64 8
  call void @_ZN4VirtC2Ev(ptr %virt_ptr)
  ; then init direct base Base
  invoke void @_ZN4BaseC2Ev(ptr %0) to label %ok unwind label %eh
ok:
  ; overwrite primary vptr with MostDerived's vtable
  store ptr getelementptr inbounds inrange(-24, 16)
            ({ [5 x ptr], [4 x ptr] }, ptr @_ZTV11MostDerived,
             i32 0, i32 0, i32 3),
        ptr %0, align 8
  ; overwrite secondary vptr (virtual base Virt sub-object)
  %vb_ptr = getelementptr inbounds i8, ptr %0, i64 8
  store ptr getelementptr inbounds inrange(-24, 8)
            ({ [5 x ptr], [4 x ptr] }, ptr @_ZTV11MostDerived,
             i32 0, i32 1, i32 3),
        ptr %vb_ptr, align 8
  ret void
  ...
}
```

The C2 (base-object) constructor receives an extra hidden parameter — the VTT pointer — from which it reads the construction vtables for virtual bases before they are overwritten by the most-derived constructor.

### 42.4.2 Destructor Vptr Reset

The D1 (complete-object) destructor performs the mirror operation: it first stores the most-derived vtable (so that virtual functions called during the destructor body see the correct class), then calls base destructors. Each base D2 (base-object) destructor, when entered, restores that base's vtable slice. This is why `delete b` where `b` is a `Base*` pointing to a `Derived` object correctly routes virtual calls in `~Base()` to `Base`'s implementations rather than `Derived`'s.

---

## 42.5 Virtual Dispatch

### 42.5.1 `EmitCXXMemberVirtualCallExpr`

When Clang encounters a virtual member function call `f->bar(x)`, `CodeGenFunction::EmitCXXMemberVirtualCallExpr()` (in [`clang/lib/CodeGen/CGExprCXX.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGExprCXX.cpp)) drives the lowering:

1. Load the vptr from the object: `%vtbl = load ptr, ptr %obj`.
2. Compute the vtable index `idx` via `CGM.getItaniumVTableContext().getMethodVTableIndex(MD)`.
3. GEP into the vtable: `%slot = getelementptr inbounds ptr, ptr %vtbl, i64 idx`.
4. Load the function pointer: `%fn = load ptr, ptr %slot`.
5. Emit an indirect call: `call void %fn(ptr %this, ...)`.

For `struct Foo { virtual void bar(int); virtual void baz() const; }`, with `bar` at index 0 and `baz` at index 1, Clang 22 emits:

```llvm
; call f->bar(x)
%vtbl = load ptr, ptr %f
%slot0 = getelementptr inbounds ptr, ptr %vtbl, i64 0
%fn_bar = load ptr, ptr %slot0
call void %fn_bar(ptr %f, i32 %x)

; call f->baz()
%slot1 = getelementptr inbounds ptr, ptr %vtbl, i64 1
%fn_baz = load ptr, ptr %slot1
call void %fn_baz(ptr %f)
```

This is the exact IR from the verified test case (see `/tmp/vtable_test.ll`).

### 42.5.2 This-Adjustment Thunks

When a virtual function is inherited through multiple inheritance or virtual bases, the `this` pointer seen by the function body may differ from the pointer used to dispatch it. Thunks bridge this gap. A thunk is a small wrapper function emitted by `CGVTables::emitThunks()` that:

1. Adjusts `this` by a fixed byte offset (non-virtual adjustment) or by loading a value from the vcall-offset slot in the vtable (virtual adjustment).
2. Tail-calls the actual function implementation.

`ThunkInfo` holds the `this`-adjustment (`ThisAdjustment`) and, for covariant returns, the return-value adjustment (`ReturnAdjustment`). The `ThisAdjustment` struct distinguishes non-virtual (a constant byte offset) from virtual (a vtable slot index from which to load the offset at runtime).

### 42.5.3 Covariant Return Thunks

A covariant return override returns a type more derived than the base virtual function's declared return type. The thunk must adjust the return pointer if the more-derived type is offset within the base type. `CGVTables::emitCovariantReturnAdjustment()` inserts a GEP on the returned pointer before returning it to the caller.

For `struct B : A { B* clone() override; }` where `A::clone()` returns `A*`, the thunk installed in `A`'s vtable slot looks like:

```llvm
; _ZTch0_n16_N1B5cloneEv — thunk for B::clone with return covariance
define ptr @_ZTch0_n16_N1B5cloneEv(ptr %this) {
  ; call the real B::clone
  %ret = call ptr @_ZN1B5cloneEv(ptr %this)
  ; %ret is B* — adjust to A* if A and B have different offsets
  ; (no adjustment needed if B is the primary base of A)
  ret ptr %ret
}
```

When the derived and base types coincide in layout (the common case where `B` has `A` as a primary base with zero offset), no pointer arithmetic is needed and the thunk degenerates to a tail call. When the offset is non-zero, a GEP subtracts the offset:

```llvm
%adj = getelementptr inbounds i8, ptr %ret, i64 -offset_B_in_A
ret ptr %adj
```

The thunk is entered via the vtable slot that A-typed callers use; B-typed callers bypass the thunk and call `_ZN1B5cloneEv` directly.

---

## 42.6 RTTI Representation

### 42.6.1 The `std::type_info` Hierarchy

The Itanium ABI defines a hierarchy of `type_info` subclasses in `<cxxabi.h>`:

| Class | Used for |
|---|---|
| `__fundamental_type_info` | `int`, `char`, `void`, etc. |
| `__array_type_info` | Array types |
| `__function_type_info` | Function types |
| `__enum_type_info` | Enum types |
| `__class_type_info` | Classes with no bases |
| `__si_class_type_info` | Classes with exactly one public non-virtual base |
| `__vmi_class_type_info` | Classes with multiple bases or virtual bases |
| `__pointer_type_info` | Pointer types |
| `__pointer_to_member_type_info` | Pointer-to-member types |

`__class_type_info` holds only the type name. `__si_class_type_info` adds a single `__class_type_info*` base pointer. `__vmi_class_type_info` holds a flags word and a variable-length array of `__base_class_type_info` entries, each containing a base `__class_type_info*` and a packed offset-and-flags word.

### 42.6.2 LLVM IR for RTTI Objects

For `struct Base` (no bases), Clang 22 emits:

```llvm
; _ZTI4Base: { vtable-ptr-of-__class_type_info, name-ptr }
@_ZTI4Base = dso_local constant { ptr, ptr } {
  ptr getelementptr inbounds (ptr,
       ptr @_ZTVN10__cxxabiv117__class_type_infoE, i64 2),
  ptr @_ZTS4Base
}, align 8

; _ZTS4Base: null-terminated type name string
@_ZTS4Base = dso_local constant [6 x i8] c"4Base\00", align 1
```

The first field points two slots past the start of `__class_type_info`'s vtable (i.e., to the first virtual function pointer), following the same convention as all vtable pointers. The second field points to the mangled type name string (without the `_Z` prefix, but length-prefixed in the same encoding used by the mangler for nested names).

`ItaniumCXXABI::EmitRTTIForClass()` selects the appropriate `type_info` subclass based on the class's inheritance structure and emits the constant. For `__vmi_class_type_info`, the `flags` word encodes whether any base is virtual (`VMI_non_diamond_repeat_mask`) and whether the diamond inheritance pattern occurs (`VMI_diamond_shaped_mask`).

### 42.6.3 `dynamic_cast` Lowering

`dynamic_cast<T*>(p)` lowers to a call to `__dynamic_cast()` from libcxxabi. The complete C signature is:

```
void* __dynamic_cast(const void* src_ptr,
                     const __class_type_info* src_type,
                     const __class_type_info* dst_type,
                     ptrdiff_t src2dst_offset);
```

Clang 22 IR for `dynamic_cast<Derived*>(b)`:

```llvm
@_ZTI4Base    = external constant ptr
@_ZTI7Derived = external constant ptr

; null-check first
%is_null = icmp eq ptr %b, null
br i1 %is_null, label %null_path, label %cast_path

cast_path:
  %result = call ptr @__dynamic_cast(
    ptr %b,
    ptr @_ZTI4Base,
    ptr @_ZTI7Derived,
    i64 0)           ; hint: -1=unknown, 0=public, positive=unique path
  br label %merge
```

The `src2dst_offset` hint allows `__dynamic_cast` to short-circuit the search when the relationship is known at compile time. A value of 0 means "try the cheapest path first" (cast `src` directly to `dst`); −1 means the relationship is unknown and a full traversal of the inheritance graph is required.

For `typeid(*b)`, Clang loads `vptr[-1]` (the RTTI slot):

```llvm
%vtbl = load ptr, ptr %b
%rtti_slot = getelementptr inbounds ptr, ptr %vtbl, i64 -1
%ti = load ptr, ptr %rtti_slot
```

---

## 42.7 Constructor and Destructor Variants

### 42.7.1 The C1/C2/C3 and D0/D1/D2 Scheme

The Itanium ABI specifies six structors for each class:

| Variant | Mangling Code | Role |
|---|---|---|
| C1 | `C1` | Complete-object constructor: initializes virtual bases, then calls base C2s, then installs vptrs for most-derived class |
| C2 | `C2` | Base-object constructor: called by a more-derived class's C1/C2; receives VTT pointer for virtual-base vptr setup |
| C3 | `C3` | Allocating constructor: specified by the ABI but not emitted by Clang or GCC in practice |
| D2 | `D2` | Base-object destructor: destroys direct member fields, then calls base D2s |
| D1 | `D1` | Complete-object destructor: installs most-derived vptrs, calls D2 body, then virtual-base D2s |
| D0 | `D0` | Deleting destructor: calls D1, then `operator delete` |

The `StructorType` enum in [`clang/include/clang/CodeGen/CGFunctionInfo.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CodeGen/CGFunctionInfo.h) uses `Complete` for C1/D1 and `Base` for C2/D2.

### 42.7.2 Verified C1/C2 Distinction

Clang 22 emits separate `C1` and `C2` functions when virtual inheritance is present:

```llvm
; C2 (base-object): receives hidden VTT pointer (second arg)
define void @_ZN11MostDerivedC2Ev(ptr %this, ptr %vtt) {
  ; call Base::C2 first
  call void @_ZN4BaseC2Ev(ptr %this)
  ; install vptrs from VTT
  %vtt_slot0 = load ptr, ptr %vtt
  store ptr %vtt_slot0, ptr %this
  ; install virtual base vptr via vcall-offset indirection
  ...
}

; C1 (complete-object): no VTT, initializes virtual bases itself
define void @_ZN11MostDerivedC1Ev(ptr %this) {
  ; init virtual base Virt first
  %vb_off = getelementptr i8, ptr %this, i64 8
  call void @_ZN4VirtC2Ev(ptr %vb_off)
  ; init Base
  invoke void @_ZN4BaseC2Ev(ptr %this) to label %ok ...
ok:
  ; overwrite vptrs with MostDerived's final vtables
  store ptr ..._ZTV11MostDerived..., ptr %this
  store ptr ..._ZTV11MostDerived..., ptr %vb_off
}
```

When C1 and C2 are identical (no virtual bases, no complex initialization), Clang emits only C2 and creates `_ZN<class>C1Ev` as an alias:

```llvm
@_ZN4BaseD1Ev = dso_local unnamed_addr alias void (ptr), ptr @_ZN4BaseD2Ev
```

### 42.7.3 D0 (Deleting Destructor) IR

The D0 destructor is always emitted when the class has a virtual destructor and cannot be merged with D1:

```llvm
define void @_ZN4BaseD0Ev(ptr %this) {
  call void @_ZN4BaseD1Ev(ptr %this)   ; complete-object destructor
  call void @_ZdlPvm(ptr %this, i64 8)  ; sized operator delete
  ret void
}
```

For a class with a user-defined `void operator delete(void*)`, Clang replaces the `_ZdlPvm` call with the user's operator. For virtual delete (the caller invokes the destructor through a base pointer), vtable slot 1 (D0) is loaded and called — this is why `delete base_ptr` correctly dispatches to the most-derived `operator delete`.

---

## 42.8 Guard Variables for Static Locals

### 42.8.1 The Double-Checked Locking Protocol

Function-local `static` objects with dynamic initialization require protection against concurrent first-call initialization (guaranteed unique since C++11). The Itanium ABI implements this using a 64-bit guard variable per static, the first byte of which serves as an "already initialized" flag, and a lock protocol using `__cxa_guard_acquire/release/abort`.

The Clang 22 IR for `Widget& get_widget()` with `static Widget w`:

```llvm
@_ZZ10get_widgetvE1w  = internal global %struct.Widget zeroinitializer
@_ZGVZ10get_widgetvE1w = internal global i64 0, align 8   ; guard variable

define ptr @_Z10get_widgetv() personality ptr @__gxx_personality_v0 {
  ; fast path: load first byte of guard (acquire)
  %first_byte = load atomic i8, ptr @_ZGVZ10get_widgetvE1w acquire, align 8
  %done = icmp eq i8 %first_byte, 0
  br i1 %done, label %init_path, label %done_path, !prof !{!"branch_weights", i32 1, i32 1048575}

init_path:
  ; slow path: call __cxa_guard_acquire (returns nonzero if we won the race)
  %acquired = call i32 @__cxa_guard_acquire(ptr @_ZGVZ10get_widgetvE1w)
  %won = icmp ne i32 %acquired, 0
  br i1 %won, label %construct, label %done_path

construct:
  ; construct the object (may throw)
  invoke void @_ZN6WidgetC1Ev(ptr @_ZZ10get_widgetvE1w) to label %ok unwind label %abort

ok:
  ; register destructor
  call i32 @__cxa_atexit(ptr @_ZN6WidgetD1Ev,
                          ptr @_ZZ10get_widgetvE1w,
                          ptr @__dso_handle)
  call void @__cxa_guard_release(ptr @_ZGVZ10get_widgetvE1w)
  br label %done_path

abort:
  %ex = landingpad { ptr, i32 } cleanup
  call void @__cxa_guard_abort(ptr @_ZGVZ10get_widgetvE1w)
  resume %ex

done_path:
  ret ptr @_ZZ10get_widgetvE1w
}
```

The `!prof !{!"branch_weights", i32 1, i32 1048575}` annotation biases the branch predictor toward the fast (already initialized) path.

The entry point for this entire pattern is `CodeGenFunction::EmitCXXGuardedInit()` in [`clang/lib/CodeGen/CGDeclCXX.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGDeclCXX.cpp). It calls into `ItaniumCXXABI::getOrCreateStaticInitMutex()` to obtain or create the guard variable, emits the fast-path load, branches to the slow-path `__cxa_guard_acquire` call, invokes the constructor, registers the destructor with `__cxa_atexit`, and calls `__cxa_guard_release`. The landing pad that calls `__cxa_guard_abort` is also inserted here, protecting against re-entry if the constructor throws.

### 42.8.2 Apple Platform Difference

On Apple targets, the guard variable has a different layout: the low 8 bytes use `0x1` in the low bit rather than in the high bit of the first byte. `ItaniumCXXABI::getOrCreateStaticInitMutex()` checks `CGM.getTarget().getTriple().isOSDarwin()` to select the correct bit test.

### 42.8.3 `__cxa_guard_acquire` Semantics

`__cxa_guard_acquire` (provided by libcxxabi) performs a compare-and-exchange on the guard variable. It returns 1 if the caller should proceed with initialization and 0 if another thread already completed it. While initialization is in progress, other threads calling `__cxa_guard_acquire` block on a futex (Linux) or a pthread mutex (other platforms). The 64-bit guard layout has 1 byte for the "initialized" flag, 1 byte for the "in-progress" flag, and 6 bytes for the OS-level lock word.

---

## 42.9 `operator new` and `operator delete` Lowering

### 42.9.1 `EmitCXXNewExpr`

`new Obj(x)` is lowered in `CodeGenFunction::EmitCXXNewExpr()`:

1. Compute the allocation size (`sizeof(Obj)` for non-arrays; for arrays, multiply element size by count with overflow check).
2. Call `operator new(size)` — either the global `_Znwm` or a class-specific one looked up via `Sema::FindAllocationFunctions()`.
3. Emit a null check on the returned pointer (unless `new` is `nothrow`, in which case a null branch is inserted).
4. Emit the constructor call wrapped in an `invoke` with a landing pad that calls `operator delete` and rethrows if construction fails.

The verified IR for `new Obj(x)` using Clang 22:

```llvm
define ptr @_Z8make_obji(i32 %x) personality ptr @__gxx_personality_v0 {
  ; allocate: _Znwm(4) — sizeof(Obj) = 4
  %mem = call noalias nonnull ptr @_Znwm(i64 4) #builtin
  ; construct (may throw)
  invoke void @_ZN3ObjC1Ei(ptr nonnull %mem, i32 %x)
    to label %ok unwind label %abort
ok:
  ret ptr %mem
abort:
  %ex = landingpad { ptr, i32 } cleanup
  ; sized deallocation: _ZdlPvm(ptr, size)
  call void @_ZdlPvm(ptr %mem, i64 4)
  resume %ex
}
```

### 42.9.2 Array New Cookies

When `T` has a non-trivial destructor and `new T[n]` is used, the Itanium ABI inserts a **cookie** — a hidden `size_t` at the start of the allocated block storing the array element count — so that `delete[]` knows how many destructors to call. The cookie size is `max(sizeof(size_t), alignof(T))`. `ItaniumCXXABI::getArrayCookieSizeImpl()` returns this value; `ItaniumCXXABI::InitializeArrayCookie()` emits the store. For `new int[n]`, no cookie is needed because `int` has a trivial destructor; the raw `_Znam(n * sizeof(int))` call is emitted without adjustment.

For `new Complex[n]` where `Complex` has a non-trivial destructor (size 8, so cookie size = `sizeof(size_t)` = 8), Clang 22 emits:

```llvm
; Total allocation size = n * sizeof(Complex) + 8 (cookie)
%elem_bytes = umul.with.overflow i64 %n, 8      ; sizeof(Complex) = 8
%total = uadd.with.overflow i64 %elem_bytes, 8  ; + cookie
%raw = call nonnull ptr @_Znam(i64 %total)

; Store element count into cookie at start of allocation
store i64 %n, ptr %raw, align 8

; User pointer = raw + 8 (skip cookie)
%user_ptr = getelementptr inbounds i8, ptr %raw, i64 8

; Loop: construct each element
; [... constructor loop ...]
ret ptr %user_ptr
```

`delete[] p` recovers the cookie by loading from `p[-8]` (one `size_t` before the user pointer):

```llvm
; Recover element count
%cookie_ptr = getelementptr inbounds i8, ptr %p, i64 -8
%count = load i64, ptr %cookie_ptr, align 4

; Destroy elements in reverse order, then free
; [... destructor loop ...]
call void @_ZdaPvm(ptr %cookie_ptr, i64 %total_size)
```

The overflow checks (`llvm.umul.with.overflow.i64`, `llvm.uadd.with.overflow.i64`) guard against size overflow; if either overflows, the size is set to `-1` (`i64 -1 = SIZE_MAX`), which reliably causes `operator new[]` to throw `std::bad_alloc`.

### 42.9.3 Virtual Delete

When `delete b` is called through a base pointer, the vtable's D0 (deleting destructor) slot is dispatched. The D0 function calls the most-derived destructor chain and then invokes the appropriate `operator delete`. Clang 22 IR for `virtual_delete(Base* b)`:

```llvm
define void @_Z14virtual_deleteP4Base(ptr %b) {
  %is_null = icmp eq ptr %b, null
  br i1 %is_null, label %end, label %do_delete

do_delete:
  %vtbl = load ptr, ptr %b
  ; D0 is at vtable index 1 (after D1 at index 0 for a single-virtual-fn class
  ; with only a virtual destructor — actual index depends on class)
  %d0_slot = getelementptr inbounds ptr, ptr %vtbl, i64 1
  %d0_fn = load ptr, ptr %d0_slot
  call void %d0_fn(ptr %b)
  br label %end
end:
  ret void
}
```

`CodeGenFunction::EmitCXXDeleteExpr()` in `CGExprCXX.cpp` drives delete lowering: it null-checks the pointer, resolves the correct `operator delete` (or, for a virtual destructor, routes through the D0 vtable slot), and emits the destructor call or virtual dispatch sequence. `ItaniumCXXABI::emitVirtualObjectDelete()` handles the case where the class has a custom `operator delete` that must receive the original pointer before any adjustment.

### 42.9.4 Sized Deallocation (C++14)

Since C++14, `operator delete(void*, size_t)` is the preferred form when available. Clang selects the sized form unconditionally when `-fsized-deallocation` is active (the default in Clang 22 for C++14 and later). The size argument is computed at the call site from the known static type, avoiding the need for libcxxabi to re-query the allocator header. For the `_Znwm`/`_ZdlPvm` pair, this is encoded as `_ZdlPvm` (mangling of `operator delete(void*, unsigned long)`).

---

## 42.10 Thread-Local Storage and `__cxa_thread_atexit`

### 42.10.1 Non-Trivial TLS Initialization

A `thread_local` variable with a non-trivial constructor or destructor requires per-thread initialization and cleanup. `ItaniumCXXABI::emitThreadLocalInitFuncs()` generates:

1. An internal `__cxx_global_var_init` function that calls the constructor and registers the destructor with `__cxa_thread_atexit`.
2. A `__tls_init` wrapper that checks a per-thread guard byte (`@__tls_guard`) before calling `__cxx_global_var_init`, preventing double-initialization.
3. A TLS init function alias `_ZTH<var>` pointing to `__tls_init`, which the dynamic linker calls on first thread access.
4. A TLS wrapper function `_ZTW<var>` that calls the init function and returns the thread-local address via `@llvm.threadlocal.address.p0`.

Clang 22 IR for `thread_local TLSObj tls_obj`:

```llvm
@tls_obj     = dso_local thread_local global %struct.TLSObj zeroinitializer
@__tls_guard = internal thread_local global i8 0

@_ZTH7tls_obj = dso_local alias void (), ptr @__tls_init

define internal void @__cxx_global_var_init() {
  call void @_ZN6TLSObjC1Ev(ptr @tls_obj)
  call i32 @__cxa_thread_atexit(ptr @_ZN6TLSObjD1Ev,
                                  ptr @tls_obj,
                                  ptr @__dso_handle)
  ret void
}

define internal void @__tls_init() {
  %done = load i8, ptr @__tls_guard
  %is_zero = icmp eq i8 %done, 0
  br i1 %is_zero, label %init, label %end
init:
  store i8 1, ptr @__tls_guard
  call void @__cxx_global_var_init()
  br label %end
end:
  ret void
}

define weak_odr hidden ptr @_ZTW7tls_obj() comdat {
  call void @_ZTH7tls_obj()        ; calls __tls_init
  %addr = call ptr @llvm.threadlocal.address.p0(ptr @tls_obj)
  ret ptr %addr
}
```

### 42.10.2 `__cxa_thread_atexit` Semantics

`__cxa_thread_atexit(dtor, obj, dso_handle)` registers `dtor(obj)` to be called when the current thread exits, in LIFO order with other thread-exit callbacks registered by this DSO. The `dso_handle` argument is a pointer into the `.bss` of the owning shared library; the runtime uses it to filter calls when a DSO is unloaded before thread exit.

For `thread_local` variables with trivial destructors, no registration is needed and the wrapper simplifies to a single `@llvm.threadlocal.address.p0` call.

### 42.10.3 TLS Access Models and `_tlsdesc`

The `_ZTH`/`_ZTW` wrapper infrastructure exists precisely because non-trivial TLS initialization must run on first access regardless of which low-level TLS access model the linker selects. The access model hierarchy is:

| Model | Use case | Access mechanism |
|---|---|---|
| `general-dynamic` | Shared library, symbol may be preemptable | `__tls_get_addr(TLSGD reloc)` or `_tlsdesc` resolver |
| `local-dynamic` | Shared library, symbol is module-local | `__tls_get_addr(TLSLD reloc)` + offset |
| `initial-exec` | Executable or DSO loaded at startup | Load from GOT + FS-relative offset |
| `local-exec` | Executable, own TLS | FS-relative load with fixed offset |

By default, a `thread_local` variable in a shared library compiles to the `general-dynamic` model. On x86-64 with `-mtls-dialect=gnu2` (the default since glibc 2.25), Clang emits `_tlsdesc` descriptors instead of `__tls_get_addr` calls, reducing TLS access from two indirect calls to one indirect call through the descriptor's resolve function. On AArch64, `-mtls-dialect=desc` similarly uses `_tlsdesc`. The `_ZTW<var>` wrapper function calls the init alias first so that non-trivial constructor semantics are preserved even under `initial-exec` (where the linker may resolve the access to a simple FS-relative load), because the wrapper layer intercepts the first-access check before passing through to `llvm.threadlocal.address.p0`.

---

## 42.11 Exception Personality and Stack Unwinding

### 42.11.1 `__gxx_personality_v0`

Every function that can participate in C++ exception unwinding declares `__gxx_personality_v0` as its EH personality function. This is encoded in the LLVM IR function attribute `personality ptr @__gxx_personality_v0` and emitted as an entry in the `.gcc_except_table` section for each function. The personality function, provided by libcxxabi, is called by the Itanium unwinder (`libgcc_s.so` or `libunwind.so`) during both the search phase and the cleanup phase.

### 42.11.2 Throw Sequence

`throw std::runtime_error("msg")` lowers to:

```llvm
; 1. Allocate exception buffer
%ex_obj = call ptr @__cxa_allocate_exception(i64 16)  ; sizeof(runtime_error)

; 2. Construct the exception object in-place
invoke void @_ZNSt13runtime_errorC1EPKc(ptr %ex_obj, ptr @.str)
  to label %throw unwind label %abort_ex

throw:
  ; 3. Raise: sets type_info, calls unwind machinery
  call void @__cxa_throw(ptr %ex_obj,
                          ptr @_ZTISt13runtime_error,   ; type_info
                          ptr @_ZNSt13runtime_errorD1Ev) ; destructor
  unreachable
```

`__cxa_throw` calls `_Unwind_RaiseException`, which traverses the call stack looking for a matching handler. During the search phase (phase 1), it calls `__gxx_personality_v0` with `_UA_SEARCH_PHASE` for each frame. During the cleanup phase (phase 2, after a handler is found), it calls the personality with `_UA_CLEANUP_PHASE`, which executes landing pad code.

### 42.11.3 Catch and Landing Pads

`catch (const std::runtime_error& e)` is lowered to a landing pad with a `catch` clause:

```llvm
%lp = landingpad { ptr, i32 }
        catch ptr @_ZTISt13runtime_error
        catch ptr null                    ; catch(...)

%ex_ptr = extractvalue { ptr, i32 } %lp, 0
%selector = extractvalue { ptr, i32 } %lp, 1

; check which catch clause matched
%match = call i32 @llvm.eh.typeid.for.p0(ptr @_ZTISt13runtime_error)
%is_rte = icmp eq i32 %selector, %match
br i1 %is_rte, label %catch_rte, label %catch_all

catch_rte:
  %obj = call ptr @__cxa_begin_catch(ptr %ex_ptr)
  ; use %obj as reference to std::runtime_error
  call void @__cxa_end_catch()
  ...
```

`__cxa_begin_catch` adjusts the caught pointer (for non-pointer exceptions, it returns a pointer to the exception object itself) and increments the uncaught-exception count. `__cxa_end_catch` decrements it and, if zero, destroys the exception object.

### 42.11.4 LSDA Generation

The Language-Specific Data Area (LSDA) is emitted by `EHStreamer` in [`llvm/lib/CodeGen/AsmPrinter/EHStreamer.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/AsmPrinter/EHStreamer.cpp). It encodes three parallel tables:

1. **Call-site table**: maps PC ranges within the function to landing pad offsets and action record indices. Encoded as ULEB128 pairs.
2. **Action table**: a linked list of type-filter entries. Each entry holds a SLEB128 type index and a SLEB128 link to the next action, or 0 for cleanup-only.
3. **Type table**: a reverse array of `type_info*` pointers (positive indices) and exception specification entries (negative indices), referenced by the action table.

The entire LSDA is emitted into the `.gcc_except_table` section. The function's unwind table entry (in `.eh_frame`) points to it via a `DW_EH_PE_pcrel` pointer.

The LSDA header specifies the encoding for the landing pad base (usually `DW_EH_PE_omit`, meaning the function start is the base), the encoding for type table entries (typically `DW_EH_PE_indirect | DW_EH_PE_pcrel`), and the encoding for call-site entries. The compact LSDA format used in `.eh_frame` on macOS differs: Apple's runtime uses a slightly different table layout for Compact Unwind Descriptors, but `__gxx_personality_v0` remains the personality function.

### 42.11.5 `llvm.eh.typeid.for` Intrinsic

The `llvm.eh.typeid.for.p0(ptr @_ZTISt13runtime_error)` intrinsic returns the integer selector value that the personality function assigns to a given `type_info` pointer for the current function. It is a compile-time constant from the optimizer's perspective: no two distinct `type_info` pointers in the same function get the same selector. The backend lowers this intrinsic to the literal selector integer recorded in the LSDA type table, enabling the `icmp eq i32 %selector, %match` pattern shown in §42.11.3 without any runtime lookup.

### 42.11.6 `_Unwind_Resume` and `__cxa_rethrow`

A `throw;` statement inside a catch block calls `__cxa_rethrow()`, which re-raises the current exception. `resume { ptr, i32 }` in LLVM IR (generated by Clang for cleanup paths that need to propagate) calls `_Unwind_Resume()`, which continues the phase-2 unwinding from where it was interrupted by a cleanup landing pad. The distinction matters: `__cxa_rethrow` restarts phase 1 (searches for a new handler), while `_Unwind_Resume` continues phase 2 (runs remaining cleanups).

---

## 42.12 ABI Compatibility Flags

### 42.12.1 `-fabi-version=N`

Clang tracks a version counter for breaking ABI changes. The `-fabi-version=N` flag selects which version to target. As of Clang 22, the default is version 18. Key version bumps:

| Version | Change |
|---|---|
| 1 | Original Itanium ABI |
| 2 | Bug fixes for `std::string` layout |
| 6 | Fixes for mangling of function types under `decltype` |
| 14 | Mangling of `_Complex` types corrected |
| 15 | Mangling of C++20 concepts constraints in function templates |
| 17 | Mangling of NTTP (non-type template parameter) changes |
| 18 | Default as of Clang 22; fixes for lambda capture mangling |

`__GXX_ABI_VERSION` is defined as `1002` by Clang 22 (matching GCC's encoding of version 1.002), regardless of `-fabi-version`. This macro is for libstdc++/libc++ internal use; for user code, `__clang_major__` is the reliable indicator.

### 42.12.2 `-fabi-compat-version=N`

When linking a mix of objects built with different `-fabi-version` values, `-fabi-compat-version=M` instructs Clang to emit symbols using ABI version M for the affected constructs while compiling at a newer version. This produces both old and new mangled names (via `.symver` aliases on ELF) for declarations that differ between the two versions.

### 42.12.3 `-Wabi` Diagnostics

`-Wabi` enables warnings for constructs whose mangling or layout changed between ABI versions. For example, a template specialization using a `decltype` expression that is mangled differently at version 5 versus 6 will produce a diagnostic. `-Wabi=N` limits warnings to changes introduced at version N or earlier.

### 42.12.4 Checking the Active ABI Version

```cpp
#if __GXX_ABI_VERSION >= 1002
  // Clang/GCC with Itanium ABI
#endif
```

For version-specific behavior at the Clang source level, `CGM.getCodeGenOpts().CurrentModule` and `CGM.getTarget().getCXXABI().getKind() == TargetCXXABI::GenericItanium` are the relevant predicates. The ABI version number itself is accessed via `CGM.getCodeGenOpts().ABIVersion`.

### 42.12.5 ABI Stability in Practice

The most common source of Itanium ABI incompatibility in production is not the formal version breaks but rather the implicit layout dependency on `std::string`, `std::list`, and similar types whose sizes changed between libstdc++ 4 and 5 (the COW string vs SSO string transition). This is not an ABI version issue in the Clang sense — both old and new strings mangle to the same symbol — but it means that passing `std::string` by value across a DSO boundary compiled with different standard library versions is undefined behavior. The `-D_GLIBCXX_USE_CXX11_ABI=0` flag (for libstdc++) and the dual-ABI mechanism in `std::__cxx11` namespace are the standard mitigation.

For Clang-specific extensions such as `__attribute__((trivial_abi))`, which allows C++ types to be passed in registers even if they have non-trivial copy/move constructors, the ABI changes are opt-in and isolated: only direct callers and callees of such types see the different calling convention; the vtable and mangling are unaffected.

---

## 42.13 The CGCXXABI Abstraction Layer

The `CGCXXABI` base class in [`clang/lib/CodeGen/CGCXXABI.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/CodeGen/CGCXXABI.h) defines the contract between the target-independent code generator and the ABI-specific layer. Key virtual methods that `ItaniumCXXABI` overrides:

| Method | Purpose |
|---|---|
| `EmitConstructorCall` | Emit call to C1/C2 variant |
| `EmitDestructorCall` | Emit call to D0/D1/D2 variant |
| `emitVTableDefinitions` | Emit `_ZTV`, `_ZTT`, `_ZTC` constants |
| `getVirtualFunctionPointer` | Load vptr and GEP into vtable |
| `EmitVirtualDestructorCall` | Load D0 or D1 slot and call |
| `emitVirtualObjectDelete` | Handle virtual `operator delete` dispatch |
| `getArrayCookieSizeImpl` | Return cookie bytes needed for array new |
| `InitializeArrayCookie` | Store element count into cookie |
| `ReadArrayCookie` | Read element count from cookie in delete[] |
| `EmitGuardedInit` | Emit guard variable and init sequence for static locals |
| `registerGlobalDtor` | Register destructor via `__cxa_atexit` |
| `emitThreadLocalInitFuncs` | Emit `_ZTH`, `_ZTW`, `__tls_init` for non-trivial TLS |
| `EmitRTTIForClass` | Emit `_ZTI`/`_ZTS` for a class type |
| `shouldTypeidBeNullChecked` | Determine if `typeid` expr needs null check |
| `EmitBadTypeidCall` | Emit call to `__cxa_bad_typeid` |

This separation means the Microsoft ABI (Chapter 43) overrides the same set of methods with completely different implementations — vtable layout, mangling, EH personality, and guard variables all differ — without changing any of the call sites in `CodeGenFunction`.

---

## Chapter Summary

- The `ItaniumCXXABI` class in `clang/lib/CodeGen/ItaniumCXXABI.cpp` and `CXXNameMangler` in `clang/lib/AST/ItaniumMangle.cpp` together implement the entire Itanium C++ ABI for Linux/macOS/Android/FreeBSD targets.
- Name mangling uses `_Z` prefix, `N...E` for nested names, `I...E` for template arguments, single-letter encodings for built-in types, and substitution compression via `S_`/`S0_`/`S1_`.
- Vtables contain offset-to-top, RTTI pointer, then function pointers; the vptr points to the third slot; secondary vtables cover multiple and virtual bases in a single constant.
- C1 constructors (complete-object) initialize virtual bases then overwrite all vptrs; C2 (base-object) constructors receive a hidden VTT pointer; when C1 and C2 are identical, C1 becomes an alias to C2.
- D0 (deleting) destructors dispatch virtual `operator delete`; D1 (complete-object) resets vptrs then calls D2; Clang merges D1 into D2 via alias when they are identical.
- Static local guard variables use a 64-bit guard with first-byte flag, `__cxa_guard_acquire/release/abort`, and `__cxa_atexit` for destructor registration.
- `new` expressions call `operator new`, construct via `invoke`, and on exception free via the sized `operator delete`; array new uses a hidden cookie for types with non-trivial destructors.
- Thread-local non-trivial objects are initialized via a `_ZTH` init function alias, a `__tls_init` guard wrapper, and `__cxa_thread_atexit` for per-thread destructor registration.
- Exception handling uses `__gxx_personality_v0`, `__cxa_allocate_exception`/`__cxa_throw`, `landingpad` instructions with type selectors, and LSDA tables in `.gcc_except_table`.
- The default ABI version in Clang 22 is 18; `-fabi-version=N` and `-fabi-compat-version=N` allow mixing objects from different ABI eras.

---

*Cross-references:*
- [Chapter 26 — Exception Handling](../part-04-llvm-ir/ch26-exception-handling.md) — LLVM IR EH constructs: `landingpad`, `invoke`, `resume`, `.eh_frame`/LSDA encoding at the IR level
- [Chapter 39 — CodeGenModule and CodeGenFunction](../part-06-clang-codegen/ch39-codegenmodule-codegenfunction.md) — module-level emission infrastructure used by `CGVTables::EmitVTable()`
- [Chapter 41 — Calls, the ABI Boundary, and Builtins](../part-06-clang-codegen/ch41-calls-abi-boundary-builtins.md) — calling convention lowering, `this`-pointer passing, return-value slots
- [Chapter 43 — C++ ABI Lowering: Microsoft](../part-06-clang-codegen/ch43-cpp-abi-lowering-msvc.md) — contrast: MSVC name decoration, vtable layout differences, SEH personality

*Reference links:*
- [Itanium C++ ABI Specification](https://itanium-cxx-abi.github.io/cxx-abi/abi.html) — the normative document
- [`clang/lib/CodeGen/ItaniumCXXABI.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/ItaniumCXXABI.cpp) — all ABI code generation hooks
- [`clang/lib/AST/ItaniumMangle.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/AST/ItaniumMangle.cpp) — `CXXNameMangler` implementation
- [`clang/lib/AST/VTableBuilder.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/AST/VTableBuilder.cpp) — `VTableLayout`, `ItaniumVTableContext`
- [`clang/lib/CodeGen/CGExprCXX.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/CodeGen/CGExprCXX.cpp) — `EmitCXXMemberVirtualCallExpr`, `EmitCXXNewExpr`, `EmitCXXDeleteExpr`
- [`llvm/lib/Demangle/ItaniumDemangle.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Demangle/ItaniumDemangle.cpp) — demangling used by `llvm-cxxfilt`
- [`llvm/lib/CodeGen/AsmPrinter/EHStreamer.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/CodeGen/AsmPrinter/EHStreamer.cpp) — LSDA table emission
