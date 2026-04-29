# Chapter 69 — Whole-Program Devirtualization

*Part X — Analysis and the Middle-End*

Virtual dispatch is the mechanism by which C++ programs select a function implementation at runtime based on an object's dynamic type. This flexibility has a cost: each virtual call requires a vtable pointer load, a function-pointer load from the vtable, an indirect call, and — critically — it blocks the inliner from seeing inside the callee. Whole-Program Devirtualization (WPD) eliminates these costs when the global view afforded by link-time optimization reveals that a virtual call can only reach one (or a small set of) implementation(s). This chapter covers the `WholeProgramDevirtPass`, its analysis of `@llvm.type.checked.load` and `@llvm.type.test` intrinsics, the five devirtualization strategies it applies, how it shares infrastructure with CFI's `LowerTypeTestsPass`, and how it integrates with both full LTO and ThinLTO.

## 69.1 The Problem: Virtual Call Overhead and Opacity

A virtual call in C++ compiles to a three-instruction sequence at the machine level:

```llvm
; Typical virtual call before WPD:
%vtable_ptr = load ptr, ptr %obj             ; load vtable pointer from object
%fn_ptr_slot = getelementptr ptr, ptr %vtable_ptr, i64 2  ; select method slot
%fn_ptr = load ptr, ptr %fn_ptr_slot         ; load function pointer from vtable
call void %fn_ptr(ptr %obj, ...)             ; indirect call
```

The indirect call cannot be inlined, cannot be speculated away by branch prediction analysis, and prevents many classical optimizations (constant propagation, dead code elimination across the call boundary). If the program has a closed world — all implementations of the virtual method are known at link time — the indirect call may be replaceable by a direct call or even an inlined body.

WPD requires LTO (full or thin) because it must see *all* vtable definitions in the program to reason about which implementations a given call can reach. It runs after the IR merge (LTO) or summary propagation (ThinLTO) step.

## 69.2 Type Test Infrastructure: The Foundation of WPD

WPD and CFI share a common IR-level abstraction for expressing type constraints on pointers: the `@llvm.type.test` and `@llvm.type.checked.load` intrinsics (introduced in [Chapter 68 — Hardening and Mitigations](ch68-hardening-and-mitigations.md)).

### 69.2.1 @llvm.type.checked.load

For virtual calls, Clang emits `@llvm.type.checked.load` instead of a plain load from the vtable:

```llvm
; Clang-generated CFI+WPD virtual call:
%pair = call { ptr, i1 } @llvm.type.checked.load(
    ptr %vtable_ptr,
    i32 16,                          ; byte offset in vtable (method slot)
    metadata !"_ZTS3Foo")            ; type identifier (mangled class name)
%fn_ptr = extractvalue { ptr, i1 } %pair, 0   ; function pointer
%type_ok = extractvalue { ptr, i1 } %pair, 1  ; type check result (for CFI)
call void @llvm.assume(i1 %type_ok)           ; abort if type check fails (CFI)
call void %fn_ptr(ptr %obj, ...)              ; virtual call
```

The intrinsic bundles the vtable load with a type assertion in a single operation. This encodes both the CFI check (is `%vtable_ptr` a valid vtable for `_ZTS3Foo`?) and the WPD opportunity (what implementations exist for this slot offset in all vtables of type `_ZTS3Foo`?).

### 69.2.2 vcall_visibility Attribute

For WPD to analyze a vtable, the linker must have enough information to know whether all implementations are visible. Clang emits `!vcall_visibility` metadata on global vtable variables:

```llvm
@_ZTV3Foo = constant [...], !vcall_visibility !{i64 2}
; visibility values:
; 0 = public (any translation unit may define new virtual functions)
; 1 = linkage-unit (WPD can see all implementations within the LTO unit)
; 2 = translation-unit (all implementations in this single TU)
```

WPD only devirtualizes calls on vtables with `vcall_visibility` ≥ 1 (linkage-unit or translation-unit), because only then can it enumerate all possible implementations.

## 69.3 The WholeProgramDevirtPass

`WholeProgramDevirtPass` is declared in [`llvm/Transforms/IPO/WholeProgramDevirt.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Transforms/IPO/WholeProgramDevirt.h) and implemented in [`llvm/lib/Transforms/IPO/WholeProgramDevirt.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Transforms/IPO/WholeProgramDevirt.cpp). It is a module pass that runs as part of the LTO pipeline.

```bash
opt -passes='wholeprogramdevirt' -S input.ll
```

In the standard Clang LTO pipeline, WPD runs after ThinLTO link-time optimization:

```bash
clang -O2 -flto -fwhole-program-vtables -fvisibility=hidden program.cpp -o program
```

`-fwhole-program-vtables` instructs Clang to emit `@llvm.type.checked.load` intrinsics and `!vcall_visibility` metadata; `-fvisibility=hidden` gives vtables linkage-unit visibility, enabling WPD to treat all vtables as known.

### 69.3.1 Pass Structure

The pass operates in four phases:

1. **Collection**: scan all modules for `GlobalVariable` objects with vtable metadata, and all call sites that use `@llvm.type.checked.load` or `@llvm.type.test`.
2. **Analysis**: for each `VTableSlot` (a `{type_id, byte_offset}` pair), enumerate all vtables with that type and extract the function pointer at that slot.
3. **Devirtualization**: apply the best strategy (see §69.4) to each call site.
4. **Lowering handoff**: update type test metadata so `LowerTypeTestsPass` can replace remaining `@llvm.type.test` intrinsics with range checks.

```cpp
// Simplified internal representation
struct VTableSlot {
  Metadata *TypeID;          // type identifier metadata
  uint64_t ByteOffset;       // offset within vtable (in bytes)
};

struct VTableSlotInfo {
  // All call sites using this slot
  SmallVector<VirtualCallSite, 4> CallSites;
  // All vtable definitions providing this slot
  SmallVector<VTableBits *, 4> VTables;
};
```

## 69.4 Devirtualization Strategies

WPD applies the most aggressive strategy that the evidence supports. The strategies, from most to least aggressive:

### 69.4.1 Single Implementation (Direct Call)

If exactly one function appears across all vtables at a given slot, every call through that slot must call that one function. WPD replaces the indirect call with a direct call:

```llvm
; Before WPD (indirect virtual call):
%fn_ptr = extractvalue { ptr, i1 } %pair, 0
call void %fn_ptr(ptr %obj, ...)

; After WPD (single-impl devirtualization):
call void @_ZN3Foo6methodEv(ptr %obj, ...)
```

The direct call can now be inlined by subsequent inlining passes, enabling constant propagation and dead-code elimination through the previously-opaque call boundary.

### 69.4.2 Branch Funnel (Multiple Implementations, Small Set)

If two or three implementations exist, WPD generates a branch funnel — a small dispatch function that performs a direct comparison on the vtable pointer and dispatches accordingly:

```llvm
; Branch funnel for two implementations:
%is_foo = icmp eq ptr %vtable_ptr, @_ZTV3Foo   ; is it Foo?
%result = select i1 %is_foo,
    ptr @_ZN3Foo6methodEv,
    ptr @_ZN3Bar6methodEv
call void %result(ptr %obj, ...)
```

The branch funnel replaces a generic indirect call with a direct comparison tree. At call sites with profile data indicating which branch is hot, the inliner can then inline the hot branch directly.

### 69.4.3 Uniform Return Value

If all implementations at a slot return the same constant value, WPD replaces the call with that constant (or marks the call with the known return value for constant propagation):

```cpp
// All three implementations return the same constant:
int Foo::getId() const { return 42; }
int Bar::getId() const { return 42; }
int Baz::getId() const { return 42; }
```

```llvm
; Before:
%id = call i32 %fn_ptr(ptr %obj)

; After WPD (uniform return value):
; The call is kept but annotated with !range metadata:
%id = call i32 %fn_ptr(ptr %obj), !range !{i32 42, i32 43}
; SCCP and InstCombine then fold to:
; %id = i32 42
```

### 69.4.4 Unique Return Value

If all implementations return *distinct* values (each subclass returns a unique constant), WPD transforms the call into a load from a type-specific constant:

```cpp
int Foo::typeTag() const { return 1; }
int Bar::typeTag() const { return 2; }
int Baz::typeTag() const { return 3; }
```

WPD synthesizes a "virtual constant" at a fixed offset relative to the vtable pointer:

```llvm
; WPD emits a virtual constant at vtable+offset:
; Effectively: store the return value *before* the function pointer in the vtable
%tag_slot = getelementptr i8, ptr %vtable_ptr, i64 -8  ; slot before method
%tag = load i32, ptr %tag_slot                          ; load stored constant
; The actual function call is eliminated
```

The function call is replaced with a memory load from a known vtable offset, which is typically cheaper than the full virtual dispatch and eliminates the callee's code entirely.

### 69.4.5 Virtual Constant Propagation

Generalization of unique return value: WPD propagates constants through vtable slots more aggressively, storing multiple precomputed values per vtable entry and replacing calls with table lookups:

```cpp
// Two different virtual methods that can both be replaced with constants:
class Base { virtual int a() = 0; virtual int b() = 0; };
class Foo : Base { int a() override { return 1; } int b() override { return 10; } };
class Bar : Base { int a() override { return 2; } int b() override { return 20; } };
```

WPD propagates both `a()` and `b()` as virtual constants, emitting two per-vtable constant slots that replace both virtual calls with loads.

## 69.5 CFI and WPD Shared Infrastructure

CFI (Chapter 68) and WPD share the same IR infrastructure: both use `@llvm.type.test` and `@llvm.type.checked.load`, and both are lowered by `LowerTypeTestsPass`. The two optimizations are designed to work together: Clang emits both CFI checks and WPD hints from a single `-fwhole-program-vtables` + `-fsanitize=cfi-vcall` invocation.

### 69.5.1 LowerTypeTestsPass

[`LowerTypeTestsPass`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Transforms/IPO/LowerTypeTests.h) runs after WPD and converts remaining `@llvm.type.test` intrinsics to concrete range checks:

```
WholeProgramDevirtPass
  → removes @llvm.type.checked.load for devirtualized calls
  → leaves @llvm.type.test for CFI checks that remain

LowerTypeTestsPass
  → groups vtables with the same type_id into contiguous memory regions
  → replaces @llvm.type.test with: (ptr - region_start) < region_size
```

After WPD replaces indirect calls with direct calls (or branch funnels), the associated `@llvm.assume(i1 %type_ok)` becomes trivially true (the direct call is always to a valid target) and is removed. The `LowerTypeTestsPass` only needs to handle CFI checks at call sites where WPD couldn't fully devirtualize (because the call site is truly polymorphic or `vcall_visibility` is public).

### 69.5.2 Interplay: CFI Precision After WPD

WPD and CFI are complementary: WPD is a performance optimization (remove virtual dispatch overhead), CFI is a security hardening (restrict indirect calls to valid targets). After WPD replaces a virtual call with a direct call, CFI's check becomes trivially satisfied — the direct call is always correct. WPD and CFI can both be enabled simultaneously:

```bash
clang -O2 -flto -fwhole-program-vtables \
    -fvisibility=hidden \
    -fsanitize=cfi-vcall \
    program.cpp -o program
```

In this configuration:
- WPD eliminates indirect calls where it has full visibility.
- CFI checks remain at call sites where WPD cannot fully devirtualize (public visibility vtables, external library calls).
- LowerTypeTestsPass converts remaining CFI checks to range checks.

## 69.6 ThinLTO Integration

ThinLTO (Chapter 77) does not perform a full IR merge — instead, each module is compiled independently using a shared module summary index. WPD must work within this constraint.

### 69.6.1 Summary-Based WPD

During the ThinLTO pre-link phase, LLVM emits vtable summary information into the module summary index:

```
Module Summary Index:
  GlobalValue: _ZTV3Foo (vtable)
    vcall_visibility: linkage_unit
    vcall_offset_16: @_ZN3Foo6methodEv   ; method at byte offset 16
  GlobalValue: _ZTV3Bar (vtable)
    vcall_visibility: linkage_unit
    vcall_offset_16: @_ZN3Bar6methodEv
```

WPD runs on the merged summary index during ThinLTO's link step, collecting all vtable implementations across all modules. It then communicates devirtualization decisions back to each module via the summary: "devirtualize call site at function F, call site ID X, to direct call @_ZN3Foo6methodEv".

Each module then applies the WPD decisions during its per-module optimization pass without needing the full IR of other modules.

### 69.6.2 Summary Export and Import

The WPD pass in ThinLTO mode takes two additional parameters:

```cpp
// In ThinLTO pre-link phase:
WholeProgramDevirtPass WPD(
    /*ExportSummary=*/&ModuleSummaryIndex,   // write WPD results to index
    /*ImportSummary=*/nullptr);

// In ThinLTO per-module phase:
WholeProgramDevirtPass WPD(
    /*ExportSummary=*/nullptr,
    /*ImportSummary=*/&ModuleSummaryIndex);  // read WPD results from index
```

The export phase writes devirtualization decisions into the summary; the import phase reads them and applies them during the per-module compilation.

```bash
# ThinLTO + WPD:
clang -O2 -flto=thin -fwhole-program-vtables -fvisibility=hidden -c a.cpp -o a.o
clang -O2 -flto=thin -fwhole-program-vtables -fvisibility=hidden -c b.cpp -o b.o
clang -O2 -flto=thin a.o b.o -o program   # ThinLTO runs WPD on summary
```

## 69.7 Visibility Requirements

WPD requires that all vtable definitions be visible to the LTO unit. There are several ways to achieve this:

### 69.7.1 Hidden Visibility

`-fvisibility=hidden` marks all symbols (including vtables) as hidden by default. Hidden symbols cannot be overridden by shared libraries, giving the linker a closed world:

```bash
clang -O2 -flto -fwhole-program-vtables -fvisibility=hidden -fvisibility-inlines-hidden program.cpp
```

### 69.7.2 vcall_visibility Annotation

For mixed-visibility programs (some classes exported, some not), Clang annotates each vtable with `!vcall_visibility`:

```llvm
; Exported class (any DSO may derive from it):
@_ZTV11ExportedBase = constant [...], !vcall_visibility !{i64 0}  ; public

; Internal class (only this LTO unit derives from it):
@_ZTV13InternalDerived = constant [...], !vcall_visibility !{i64 1}  ; linkage_unit
```

WPD only devirtualizes calls to `linkage_unit` or `translation_unit` visibility vtables.

### 69.7.3 `__attribute__((visibility("hidden")))` Per-Class

Individual classes can be marked hidden even in a default-visible compilation:

```cpp
class __attribute__((visibility("hidden"))) InternalImpl : public PublicBase {
    int method() override { return 42; }
};
```

Clang emits `vcall_visibility = linkage_unit` for `InternalImpl`'s vtable, enabling WPD for calls to `InternalImpl` objects even when other classes in the program have public vtables.

## 69.8 Diagnostic and Debug Support

WPD emits optimization remarks when it devirtualizes a call site:

```bash
clang -O2 -flto -fwhole-program-vtables -fvisibility=hidden \
    -Rpass=wholeprogramdevirt program.cpp -o program
# Output:
# program.cpp:42:5: remark: single-implementation devirtualization of call to Base::method [-Rpass=wholeprogramdevirt]
# program.cpp:55:5: remark: virtual constant propagation of Base::typeTag [-Rpass=wholeprogramdevirt]
```

These remarks are emitted via `ORE` (OptimizationRemarkEmitter) and can also be captured in YAML format with `-fsave-optimization-record` for offline analysis.

## 69.9 A Complete WPD Example

```cpp
// program.cpp — closed class hierarchy, hidden visibility
class __attribute__((visibility("hidden"))) Base {
public:
    virtual int compute(int x) = 0;
    virtual int tag() const = 0;
};

class __attribute__((visibility("hidden"))) Foo : public Base {
public:
    int compute(int x) override { return x * 2; }
    int tag() const override { return 1; }
};

class __attribute__((visibility("hidden"))) Bar : public Base {
public:
    int compute(int x) override { return x + 10; }
    int tag() const override { return 2; }
};

int process(Base *b, int x) {
    return b->compute(x) + b->tag();  // two virtual calls
}
```

Before WPD (with `-fwhole-program-vtables`), `process` compiles to two `@llvm.type.checked.load` sequences. After WPD with LTO:

- `b->compute(x)`: two implementations (Foo and Bar) → branch funnel
- `b->tag()`: two distinct constant returns (1 and 2) → virtual constant propagation (load from vtable slot)

```llvm
; After WPD — process():
define i32 @_Z7processP4Basei(ptr %b, i32 %x) {
  ; Devirtualize compute(): branch funnel
  %vtable = load ptr, ptr %b
  %is_foo = icmp eq ptr %vtable, @_ZTV3Foo
  br i1 %is_foo, %compute_foo, %compute_bar

compute_foo:
  %r1 = call i32 @_ZN3Foo7computeEi(ptr %b, i32 %x)  ; direct call (inlinable)
  br %merge1

compute_bar:
  %r2 = call i32 @_ZN3Bar7computeEi(ptr %b, i32 %x)  ; direct call (inlinable)
  br %merge1

merge1:
  %compute_result = phi i32 [%r1, %compute_foo], [%r2, %compute_bar]

  ; Devirtualize tag(): virtual constant propagation (load, not call)
  %tag_slot = getelementptr i8, ptr %vtable, i64 -8   ; virtual constant slot
  %tag_val = load i32, ptr %tag_slot                  ; = 1 for Foo, 2 for Bar

  %result = add i32 %compute_result, %tag_val
  ret i32 %result
}
```

After WPD, the inliner can inline both `Foo::compute` and `Bar::compute` into the branch funnel, resulting in:

```llvm
; After inlining post-WPD:
define i32 @_Z7processP4Basei(ptr %b, i32 %x) {
  %vtable = load ptr, ptr %b
  %is_foo = icmp eq ptr %vtable, @_ZTV3Foo
  %tag_slot = getelementptr i8, ptr %vtable, i64 -8
  %tag_val = load i32, ptr %tag_slot
  br i1 %is_foo, %is_foo_branch, %is_bar_branch

is_foo_branch:
  %foo_r = mul i32 %x, 2          ; Foo::compute inlined
  %foo_result = add i32 %foo_r, %tag_val
  ret i32 %foo_result

is_bar_branch:
  %bar_r = add i32 %x, 10         ; Bar::compute inlined
  %bar_result = add i32 %bar_r, %tag_val
  ret i32 %bar_result
}
```

The two virtual calls are now two branches, each with inlined implementations and a vtable-slot load — no indirect calls remain.

---

## Chapter Summary

- **Whole-Program Devirtualization** eliminates virtual call overhead at LTO time by replacing `@llvm.type.checked.load` intrinsics with direct calls or optimized alternatives when all vtable implementations are visible.
- **Preconditions**: requires `-fwhole-program-vtables` (to emit type.checked.load), `-fvisibility=hidden` or per-class hidden visibility (to give vtables linkage-unit visibility), and LTO or ThinLTO.
- **Five strategies**: single-implementation (direct call), branch funnel (small comparison tree), uniform return value (constant fold), unique return value (virtual constant), and virtual constant propagation (multi-slot table lookups replacing multiple calls).
- **CFI sharing**: WPD and CFI use the same `@llvm.type.checked.load` / `@llvm.type.test` infrastructure; `LowerTypeTestsPass` runs after WPD and converts remaining type tests to range checks for the CFI checks that survive devirtualization.
- **ThinLTO integration**: WPD runs in two phases — export (write decisions to module summary index) and import (apply decisions during per-module compilation); each module receives WPD decisions without seeing the full IR of other modules.
- **Diagnostics**: `-Rpass=wholeprogramdevirt` emits optimization remarks per devirtualized call site; `-fsave-optimization-record` captures them in YAML for offline analysis.
- **Combined pipeline**: `-O2 -flto=thin -fwhole-program-vtables -fvisibility=hidden -fsanitize=cfi-vcall` enables WPD for performance and CFI for security simultaneously; WPD eliminates indirect calls where possible, CFI guards the remaining ones.
