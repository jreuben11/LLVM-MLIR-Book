# Appendix A — LLVM IR Quick Reference

*Quick Reference | LLVM 22.1.x*

This appendix is a dense quick-reference for LLVM IR syntax and semantics as of LLVM 22.1.x. It is not a tutorial — consult [Chapter 16 — IR Structure](../chapters/part-04-llvm-ir/ch16-ir-structure.md) through [Chapter 27 — Coroutines and Atomics](../chapters/part-04-llvm-ir/ch27-coroutines-and-atomics.md) for explanations. Canonical source: [LangRef.rst](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/docs/LangRef.rst).

---

## A.1 Module-Level Declarations

```llvm
source_filename = "foo.c"
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"

@g = global i32 42, align 4
@c = constant [5 x i8] c"hello"
@w = weak global i32 0
@e = external global i32
@t = thread_local global i32 0
@a = alias i32, ptr @g

declare i32 @printf(ptr nocapture readonly, ...)
define i32 @main(i32 %argc, ptr %argv) #0 { ... }

module asm "nop"

!llvm.module.flags = !{!0, !1}
!llvm.dbg.cu = !{!2}
!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{i32 7, !"uwtable", i32 2}
```

| Declaration | Syntax | Notes |
|---|---|---|
| `target datalayout` | `target datalayout = "..."` | Specifies endianness, pointer size, alignment. Required for correct lowering |
| `target triple` | `target triple = "arch-vendor-os-env"` | e.g. `x86_64-unknown-linux-gnu`, `aarch64-apple-macosx14.0` |
| `source_filename` | `source_filename = "path"` | Used for profile instrumentation and debug info |
| Global variable | `@g = [linkage] [preemption] [visibility] [dll] [thread_local] [addrspace] [unnamed_addr] [constant/global] <type> <init> [, align N]` | `global` = mutable, `constant` = read-only |
| Function declaration | `declare [linkage] <retty> @name(<argtys>) [attrs]` | External function with no body |
| Function definition | `define [linkage] <retty> @name(<args>) [attrs] { <BBs> }` | Must have at least one basic block |
| Alias | `@a = [linkage] alias <ty>, <ty>* @target` | Symbolic alias; must refer to defined global or function |
| Module asm | `module asm "..."` | File-scope inline assembly |
| `!llvm.module.flags` | Module-level metadata controlling codegen features | Key: `wchar_size`, `uwtable`, `PIC Level`, `PIE Level`, `frame-pointer` |
| `!llvm.dbg.cu` | Reference to compile unit DIE | Produced by `-g`; see [Chapter 22](../chapters/part-04-llvm-ir/ch22-metadata-and-debug-info.md) |

### Linkage Types

| Linkage | Meaning |
|---|---|
| `private` | Local to module; symbol not emitted |
| `internal` | Local to module; symbol emitted |
| `available_externally` | Inlineable copy; not emitted for linking |
| `linkonce` | Merged with identical definitions |
| `weak` | Mergeable; may be overridden |
| `common` | Uninitialized global (like C tentative definition) |
| `appending` | Concatenated at link time (used for `llvm.global_ctors`) |
| `extern_weak` | External weak reference |
| `linkonce_odr` / `weak_odr` | ODR-compliant mergeable |
| `external` | Default; visible externally (default for `define`/`declare`) |

---

## A.2 Type System

### Primitive Types

| Category | Types | Notes |
|---|---|---|
| Integer | `i1`, `i8`, `i16`, `i32`, `i64`, `i128`, `iN` (any N) | No signedness in type; signedness encoded in instruction |
| Float | `half` (16-bit IEEE), `bfloat` (Brain float), `float` (32-bit IEEE), `double` (64-bit IEEE), `fp128` (128-bit IEEE), `x86_fp80` (80-bit extended), `ppc_fp128` | |
| Pointer | `ptr` | Opaque since LLVM 15; address space variant: `ptr addrspace(N)` |
| Void | `void` | Only as function return type |
| Label | `label` | Basic block type (used in `phi`, `br`) |
| Token | `token` | Used for coroutine/exception tokens |
| Metadata | `metadata` | Intrinsic argument type |

### Aggregate and Derived Types

| Type | Syntax | Example | Notes |
|---|---|---|---|
| Array | `[N x T]` | `[10 x i32]` | Fixed-size; zero-indexed |
| Vector | `<N x T>` | `<4 x float>` | SIMD; N must be power of 2 for most targets. Scalable: `<vscale x N x T>` |
| Struct (literal) | `{T1, T2, ...}` | `{i32, float, ptr}` | Packed: `<{T1, T2}>` |
| Struct (named) | `%S = type {T1, T2}` | `%Foo = type {i32, i8}` | Identified struct; can be recursive |
| Opaque struct | `%S = type opaque` | `%Foo = type opaque` | Forward declaration |
| Function type | `retty (T1, T2, ...)` | `i32 (ptr, i32)` | Used in function pointer types |

### DataLayout Specifiers (key)

| Specifier | Meaning |
|---|---|
| `e` / `E` | Little-endian / Big-endian |
| `m:e` / `m:o` / `m:w` | Mangling: ELF / MachO / COFF |
| `pN:S:ABI[:PF]` | Pointer for address space N: size S bits, ABI align, pref align |
| `iN:ABI[:PF]` | Integer type alignment |
| `fN:ABI[:PF]` | Float type alignment |
| `vN:ABI[:PF]` | Vector type alignment |
| `nN:M:...` | Native integer widths (e.g. `n8:16:32:64`) |
| `SN` | Stack natural alignment in bits |

---

## A.3 Instruction Reference

### Arithmetic (Integer)

| Instruction | Syntax | Flags | Semantics |
|---|---|---|---|
| `add` | `%r = add [nsw\|nuw] i32 %a, %b` | `nsw`=no signed wrap, `nuw`=no unsigned wrap | Integer addition; poison if flag violated |
| `sub` | `%r = sub [nsw\|nuw] i32 %a, %b` | same | Integer subtraction |
| `mul` | `%r = mul [nsw\|nuw] i32 %a, %b` | same | Integer multiplication |
| `udiv` | `%r = udiv [exact] i32 %a, %b` | `exact`=no remainder | Unsigned division; UB if %b=0 |
| `sdiv` | `%r = sdiv [exact] i32 %a, %b` | `exact`=no remainder | Signed division; UB if %b=0 or INT_MIN/-1 |
| `urem` | `%r = urem i32 %a, %b` | — | Unsigned remainder; UB if %b=0 |
| `srem` | `%r = srem i32 %a, %b` | — | Signed remainder; sign matches dividend |
| `shl` | `%r = shl [nsw\|nuw] i32 %a, %b` | same | Shift left; UB if %b >= bitwidth |
| `lshr` | `%r = lshr [exact] i32 %a, %b` | `exact`=top bits zero | Logical shift right |
| `ashr` | `%r = ashr [exact] i32 %a, %b` | `exact`=top bits zero | Arithmetic shift right |
| `and` | `%r = and i32 %a, %b` | — | Bitwise AND |
| `or` | `%r = or [disjoint] i32 %a, %b` | `disjoint`=no shared bits | Bitwise OR; `disjoint` enables add-like optimization |
| `xor` | `%r = xor i32 %a, %b` | — | Bitwise XOR; `xor %a, -1` = bitwise NOT |

### Arithmetic (Floating-Point)

Fast-math flags (combinable): `nnan` (no NaN), `ninf` (no Inf), `nsz` (no signed zero), `arcp` (allow reciprocal), `contract` (allow FMA contraction), `afn` (approximate functions), `reassoc` (allow reassociation), `fast` (all of the above).

| Instruction | Syntax | Notes |
|---|---|---|
| `fadd` | `%r = fadd [flags] float %a, %b` | IEEE 754 add |
| `fsub` | `%r = fsub [flags] float %a, %b` | IEEE 754 subtract |
| `fmul` | `%r = fmul [flags] float %a, %b` | IEEE 754 multiply |
| `fdiv` | `%r = fdiv [flags] float %a, %b` | IEEE 754 divide |
| `frem` | `%r = frem [flags] float %a, %b` | IEEE 754 remainder |
| `fneg` | `%r = fneg [flags] float %a` | Negate; equivalent to `fsub -0.0, %a` |

### Comparison

| Instruction | Syntax | Predicates |
|---|---|---|
| `icmp` | `%r = icmp <pred> i32 %a, %b` | `eq`, `ne`, `ugt`, `uge`, `ult`, `ule`, `sgt`, `sge`, `slt`, `sle` |
| `fcmp` | `%r = fcmp [flags] <pred> float %a, %b` | `oeq`, `ogt`, `oge`, `olt`, `ole`, `one`, `ord` (ordered); `ueq`, `ugt`, `uge`, `ult`, `ule`, `une`, `uno` (unordered); `true`, `false` |

Ordered predicates return false if either operand is NaN. Unordered predicates return true if either operand is NaN.

### Memory

| Instruction | Syntax | Notes |
|---|---|---|
| `alloca` | `%p = alloca <ty> [, <ty> <n>] [, align N] [, addrspace(A)]` | Stack allocation; returns `ptr`; freed at function exit |
| `load` | `%v = load [volatile] <ty>, ptr %p [, align N] [, !metadata]` | `volatile` prevents elimination; `atomic` variant: `load atomic <order> <ty>, ptr %p` |
| `store` | `store [volatile] <ty> %v, ptr %p [, align N]` | `atomic` variant: `store atomic <order> <ty> %v, ptr %p` |
| `fence` | `fence [syncscope("...")] <order>` | Orders: `acquire`, `release`, `acq_rel`, `seq_cst` |
| `cmpxchg` | `%r = cmpxchg [weak] ptr %p, <ty> %cmp, <ty> %new <success-ord> <fail-ord>` | Returns `{ty, i1}`; `i1` = success |
| `atomicrmw` | `%r = atomicrmw <op> ptr %p, <ty> %v <order>` | ops: `xchg`, `add`, `sub`, `and`, `nand`, `or`, `xor`, `max`, `min`, `umax`, `umin`, `fadd`, `fsub`, `fmax`, `fmin` |
| `getelementptr` | `%p = getelementptr [inbounds] <ty>, ptr %base, <idx-list>` | `inbounds`=poison if out-of-bounds; does not dereference |

### Cast Instructions

| Instruction | From → To | Notes |
|---|---|---|
| `trunc` | `iN → iM` (N > M) | Truncates to smaller int |
| `zext` | `iN → iM` (N < M) | Zero-extends |
| `sext` | `iN → iM` (N < M) | Sign-extends |
| `fptrunc` | `F → f` (wider → narrower float) | May lose precision |
| `fpext` | `f → F` (narrower → wider float) | Exact |
| `fptoui` | `float → iN` | UB if value not representable |
| `fptosi` | `float → iN` | UB if value not representable |
| `uitofp` | `iN → float` | Unsigned int to float |
| `sitofp` | `iN → float` | Signed int to float |
| `ptrtoint` | `ptr → iN` | Pointer to integer |
| `inttoptr` | `iN → ptr` | Integer to pointer |
| `bitcast` | `T → U` (same bit width) | Reinterpret bits; no-op for pointers |
| `addrspacecast` | `ptr addrspace(A) → ptr addrspace(B)` | Cast between address spaces |
| `freeze` | `T → T` | Converts `poison`/`undef` to a fixed but arbitrary value |

### Control Flow

| Instruction | Syntax | Notes |
|---|---|---|
| `ret` | `ret void` / `ret <ty> %v` | Return from function |
| `br` | `br label %L` / `br i1 %c, label %T, label %F` | Unconditional / conditional branch |
| `switch` | `switch i32 %v, label %default [ i32 0, label %L0 ... ]` | Multi-way branch |
| `indirectbr` | `indirectbr ptr %addr, [ label %L0, ... ]` | Jump to address in table |
| `invoke` | `%r = invoke <retty> @f(<args>) to label %ok unwind label %exc` | Call with landing pad on exception |
| `callbr` | `callbr <retty> asm "..." ... to label %fallthrough [label %indirect]` | For GCC inline asm goto |
| `resume` | `resume { ptr, i32 } %exc` | Re-throw exception caught by `landingpad` |
| `unreachable` | `unreachable` | UB if reached |
| `landingpad` | `%r = landingpad {ptr, i32} [cleanup] [catch ptr @t] [filter <ty> [<ty> @f ...]]` | Exception dispatch |
| `cleanuppad` | `%r = cleanuppad within %tok []` | C++ destructor scope |
| `catchpad` | `%r = catchpad within %switch [args]` | C++ catch clause |
| `catchswitch` | `%r = catchswitch within %tok [label %h ...] unwind (to caller\|label %u)` | Exception dispatch |

### Other

| Instruction | Syntax | Notes |
|---|---|---|
| `phi` | `%r = phi <ty> [%v1, %L1], [%v2, %L2], ...` | SSA merge; one entry per predecessor |
| `select` | `%r = select i1 %c, <ty> %t, <ty> %f` | Ternary; both branches may be evaluated (not short-circuit) |
| `call` | `%r = [tail\|musttail\|notail] call [fastcc] <retty> @f(<args>) [#attrs]` | Regular function call |
| `va_arg` | `%v = va_arg ptr %ap, <ty>` | Advance vararg list |
| `extractelement` | `%v = extractelement <N x T> %vec, i32 %idx` | Extract scalar from vector |
| `insertelement` | `%r = insertelement <N x T> %vec, T %v, i32 %idx` | Insert scalar into vector |
| `shufflevector` | `%r = shufflevector <N x T> %a, <N x T> %b, <M x i32> <mask>` | Permute/blend vectors; `undef` in mask = don't-care |
| `extractvalue` | `%v = extractvalue {T1,T2} %s, 0` | Extract from aggregate |
| `insertvalue` | `%r = insertvalue {T1,T2} %s, T1 %v, 0` | Insert into aggregate |

---

## A.4 Attributes

### Function Attributes

| Attribute | Meaning |
|---|---|
| `noreturn` | Function never returns normally |
| `nounwind` | Function never throws/unwinds |
| `readonly` | Only reads memory; no writes |
| `writeonly` | Only writes memory; no reads |
| `readnone` | Neither reads nor writes memory observable to caller |
| `argmemonly` | Only accesses memory through pointer arguments |
| `inaccessiblememonly` | Only accesses inaccessible memory (e.g. allocator) |
| `speculatable` | Safe to speculate (no UB, no side effects that matter) |
| `willreturn` | Function always returns (no infinite loop) |
| `nocallback` | Not a callback; optimizer may not insert calls around it |
| `nofree` | Does not free memory |
| `nosync` | Does not synchronize with other threads |
| `noinline` | Never inline |
| `alwaysinline` | Always inline if possible |
| `optnone` | Do not optimize |
| `naked` | No prologue/epilogue generated |
| `cold` / `hot` | Hint: rarely / frequently called |
| `nobuiltin` | Do not optimize as builtin |
| `sanitize_address` / `sanitize_thread` / `sanitize_memory` | Enable sanitizer instrumentation |
| `uwtable` | Must have an unwind table entry (for C++ exceptions) |
| `frame-pointer` | Retain frame pointer: `all`, `non-leaf`, `none` |

### Parameter Attributes

| Attribute | Meaning |
|---|---|
| `noalias` | Pointer does not alias any other visible pointer |
| `nonnull` | Pointer is never null |
| `dereferenceable(N)` | Pointer dereferenceable for at least N bytes |
| `dereferenceable_or_null(N)` | As above, or null |
| `byval(<ty>)` | Pass struct by value on stack; callee copy |
| `byref(<ty>)` | Pass by reference (Fortran-style) |
| `inalloca(<ty>)` | Already allocated on callee stack |
| `inreg` | Pass in register (target-specific) |
| `sret(<ty>)` | Pointer to struct return value |
| `zeroext` / `signext` | Zero/sign-extend to register width |
| `align(N)` | Pointer aligned to N bytes |
| `captures(none)` | Pointer is not captured by the callee |
| `readonly` / `writeonly` | Memory access restriction on this parameter |
| `noundef` | Value is never undef/poison |

### Return Attributes

`noalias`, `nonnull`, `dereferenceable(N)`, `zeroext`, `signext`, `noundef` apply to return value with same semantics.

---

## A.5 Calling Conventions

| CC Token | Value | Description |
|---|---|---|
| `ccc` | 0 | C calling convention (platform default) |
| `fastcc` | 8 | Fast; optimizer may modify; no varargs |
| `coldcc` | 9 | Cold; minimizes caller-side overhead |
| `webkit_jscc` | 12 | WebKit JavaScript ABI |
| `anyregcc` | 13 | Dynamic register allocation (deopt) |
| `preserve_mostcc` | 14 | Preserve most registers (Objective-C msgSend) |
| `preserve_allcc` | 15 | Preserve all registers |
| `cxx_fast_tlscc` | 17 | C++ fast TLS |
| `swiftcc` | 16 | Swift calling convention |
| `swifttailcc` | 20 | Swift tail call |
| `ghccc` | 10 | Glasgow Haskell Compiler |
| `aarch64_vector_pcs` | 97 | AArch64 SVE/NEON vector PCS |
| `aarch64_sme_preservemost_from_x0` | 102 | SME streaming mode |
| `x86_vectorcallcc` | 80 | x86/x64 vectorcall |
| `win64cc` | 79 | Windows x64 ABI |

**Tail call requirements**: `musttail` enforces tail call; requires: same function signature, no alloca between call and ret, callee does not capture args, `noalias` return if applicable. `tail` is a hint only.

---

## A.6 Metadata

### Named Metadata

```llvm
!llvm.module.flags = !{!0}     ; Module flags
!llvm.dbg.cu = !{!1}           ; Debug compile units
!llvm.ident = !{!2}            ; Compiler identification
```

### Key Instruction Metadata

| Metadata | Attachment | Meaning |
|---|---|---|
| `!tbaa` | `load`, `store` | Type-based alias analysis tree node |
| `!tbaa.struct` | `memcpy` | TBAA for struct copies |
| `!alias.scope` | `load`, `store`, `call` | NoAlias scope membership |
| `!noalias` | `load`, `store`, `call` | Not in these alias scopes |
| `!nontemporal` | `load`, `store` | Non-temporal (streaming) hint |
| `!invariant.load` | `load` | Value will not change after this load |
| `!invariant.group` | `load`, `store` | Group for invariant-across-group analysis |
| `!range` | `load`, `call` | Value range constraint: `!{i32 lo, i32 hi}` |
| `!align` | `load`, `store` | Additional alignment hint |
| `!prof` | `br`, `switch`, `call` | Branch weights / call-site profiles |
| `!dbg` | any instruction | Debug location: `!DILocation(line:, col:, scope:)` |
| `!llvm.loop` | back-edge of loop | Loop transformation hints (see below) |
| `!callees` | indirect `call` | Conservative set of possible callees |

### `!llvm.loop` Hints

```llvm
!{!"llvm.loop.unroll.count", i32 4}
!{!"llvm.loop.unroll.full"}
!{!"llvm.loop.unroll.disable"}
!{!"llvm.loop.vectorize.enable", i1 true}
!{!"llvm.loop.vectorize.width", i32 8}
!{!"llvm.loop.interleave.count", i32 2}
!{!"llvm.loop.distribute.enable", i1 true}
!{!"llvm.loop.pipeline.initiationinterval", i32 2}
```

---

## A.7 Intrinsics

### Memory Intrinsics

| Intrinsic | Signature | Notes |
|---|---|---|
| `llvm.memcpy` | `(ptr dst, ptr src, i64 len, i1 isvolatile)` | Copy memory; no overlap |
| `llvm.memcpy.inline` | same | Must be inlined by backend |
| `llvm.memmove` | `(ptr dst, ptr src, i64 len, i1 isvolatile)` | Copy memory; overlap OK |
| `llvm.memset` | `(ptr dst, i8 val, i64 len, i1 isvolatile)` | Fill memory |
| `llvm.memset.inline` | same | Must be inlined |
| `llvm.lifetime.start` | `(i64 size, ptr %p)` | Mark alloca alive; enables stack reuse |
| `llvm.lifetime.end` | `(i64 size, ptr %p)` | Mark alloca dead |
| `llvm.invariant.start` | `(i64 size, ptr %p) → ptr` | Region is invariant |
| `llvm.invariant.end` | `(ptr %start, i64 size, ptr %p)` | End invariant region |

### Hint Intrinsics

| Intrinsic | Signature | Notes |
|---|---|---|
| `llvm.assume` | `(i1 %cond)` | Assert cond true; UB if false at runtime |
| `llvm.expect` | `(iN %val, iN %expected) → iN` | Branch prediction hint; returns %val |
| `llvm.expect.with.probability` | `(iN %val, iN %expected, double %prob) → iN` | Weighted hint |
| `llvm.prefetch` | `(ptr %p, i32 rw, i32 locality, i32 cache)` | Prefetch hint |
| `llvm.donothing` | `()` | No-op; use as inlineasm substitute |
| `llvm.sideeffect` | `()` | Prevents optimization of surrounding code |
| `llvm.pseudoprobe` | `(i64 guid, i64 idx, i32 type, i64 attr)` | Pseudo-probe for PGO |

### Exception Handling

| Intrinsic | Signature | Notes |
|---|---|---|
| `llvm.eh.typeid.for` | `(ptr %typeinfo) → i32` | TypeInfo discriminator for `landingpad` |
| `llvm.eh.sjlj.setjmp` | `(ptr %buf) → i32` | setjmp for SJLJ EH |
| `llvm.eh.sjlj.longjmp` | `(ptr %buf)` | longjmp for SJLJ EH |
| `llvm.eh.exceptionpointer` | `(token %pad) → ptr` | Exception pointer from catchpad |
| `llvm.eh.exceptioncode` | `(token %pad) → i64` | Exception code (Windows SEH) |

### Frame/Stack

| Intrinsic | Signature | Notes |
|---|---|---|
| `llvm.frameaddress` | `(i32 %level) → ptr` | Frame address at given call depth |
| `llvm.returnaddress` | `(i32 %level) → ptr` | Return address |
| `llvm.addressofreturnaddress` | `() → ptr` | Pointer to return address slot |
| `llvm.stacksave` | `() → ptr` | Save stack pointer |
| `llvm.stackrestore` | `(ptr %sp)` | Restore stack pointer |
| `llvm.get.dynamic.area.offset` | `() → iN` | Dynamic alloca area offset |
| `llvm.localescape` | `(ptr %alloca, ...)` | Expose allocas for Windows EH |
| `llvm.localrecover` | `(ptr %func, ptr %fp, i32 %idx) → ptr` | Recover escaped alloca |

### Register Access

| Intrinsic | Signature | Notes |
|---|---|---|
| `llvm.read_register` | `(metadata %reg) → iN` | Read machine register |
| `llvm.write_register` | `(metadata %reg, iN %val)` | Write machine register |
| `llvm.read_volatile_register` | `(metadata %reg) → iN` | Volatile read |

### Bit Manipulation

| Intrinsic | Signature | Notes |
|---|---|---|
| `llvm.ctpop` | `(iN %v) → iN` | Population count (number of 1 bits) |
| `llvm.ctlz` | `(iN %v, i1 %is_zero_undef) → iN` | Count leading zeros |
| `llvm.cttz` | `(iN %v, i1 %is_zero_undef) → iN` | Count trailing zeros |
| `llvm.bitreverse` | `(iN %v) → iN` | Reverse all bits |
| `llvm.bswap` | `(iN %v) → iN` | Byte-swap |
| `llvm.fshl` | `(iN %a, iN %b, iN %sh) → iN` | Funnel shift left: `(a:b) << sh` |
| `llvm.fshr` | `(iN %a, iN %b, iN %sh) → iN` | Funnel shift right: `(a:b) >> sh` |

### Arithmetic with Overflow

| Intrinsic | Signature | Notes |
|---|---|---|
| `llvm.sadd.with.overflow` | `(iN %a, iN %b) → {iN, i1}` | Signed add; i1=overflow |
| `llvm.uadd.with.overflow` | `(iN %a, iN %b) → {iN, i1}` | Unsigned add |
| `llvm.ssub.with.overflow` | `(iN %a, iN %b) → {iN, i1}` | Signed subtract |
| `llvm.usub.with.overflow` | `(iN %a, iN %b) → {iN, i1}` | Unsigned subtract |
| `llvm.smul.with.overflow` | `(iN %a, iN %b) → {iN, i1}` | Signed multiply |
| `llvm.umul.with.overflow` | `(iN %a, iN %b) → {iN, i1}` | Unsigned multiply |
| `llvm.sadd.sat` | `(iN %a, iN %b) → iN` | Saturating signed add |
| `llvm.uadd.sat` | `(iN %a, iN %b) → iN` | Saturating unsigned add |

### Vector Intrinsics

| Intrinsic | Signature | Notes |
|---|---|---|
| `llvm.fma` | `(float %a, %b, %c) → float` | Fused multiply-add: `a*b+c` |
| `llvm.fabs` | `(float %v) → float` | Absolute value |
| `llvm.sqrt` | `(float %v) → float` | Square root; returns NaN if %v < -0 |
| `llvm.powi` | `(float %v, i32 %exp) → float` | Float raised to integer power |
| `llvm.sin`, `llvm.cos` | `(float) → float` | Trig (approximate if `afn` flag) |
| `llvm.vector.reduce.add` | `(<N x iN>) → iN` | Horizontal add |
| `llvm.vector.reduce.fmul` | `(float acc, <N x float>) → float` | Horizontal multiply |
| `llvm.experimental.vector.insert` | `(<M x T> %src, <N x T> %sub, i64 %idx)` | Insert subvector |
| `llvm.experimental.vector.extract` | `(<N x T> %src, i64 %idx) → <M x T>` | Extract subvector |
| `llvm.masked.load` | `(ptr, i32 align, <N x i1> mask, <N x T> passthru)` | Predicated load |
| `llvm.masked.store` | `(<N x T> val, ptr, i32 align, <N x i1> mask)` | Predicated store |
| `llvm.masked.gather` | `(<N x ptr>, i32 align, <N x i1> mask, <N x T> passthru)` | Gather |
| `llvm.masked.scatter` | `(<N x T>, <N x ptr>, i32 align, <N x i1> mask)` | Scatter |


---

@copyright jreuben11
