# Chapter 17 — The Type System

*Part IV — LLVM IR*

Every instruction in LLVM IR operates on typed values. Unlike most assembly languages, where bytes are bytes and the programmer keeps meaning in their head, LLVM IR requires every SSA value to carry a precise type. That requirement is not ceremonial: the type system drives code generation, informs alias analysis, enables bitcode verification, and determines the ABI-level representation that the backend must produce. Understanding the type system thoroughly is prerequisite to understanding everything that follows in Part IV — how instructions compute (Chapter 19 — Instructions I), how the calling convention maps types to registers and stack slots (Chapter 23 — Attributes, Calling Conventions, and the ABI), and how metadata attaches human-readable type information beyond what the IR itself encodes (Chapter 22 — Metadata and Debug Info).

This chapter works through every category of LLVM type in turn: primitive integers, floating-point types, vectors, arrays and structs, the modern opaque-pointer model, address spaces, target extension types, and the `DataLayout` string that ties abstract types to physical byte layouts. Each type is discussed in terms of its design rationale, its IR syntax, and its implications for the backend.

---

## 17.1 Primitive Integer Types: `iN`

### 17.1.1 The `iN` design

LLVM integers follow a single, uniform scheme: the type `iN` represents an N-bit integer for any positive integer N from 1 to 8,388,607 (2^23 − 1). There are no separate `signed int` and `unsigned int` types; signedness is a property of the *instruction*, not the *type*. The `add` instruction performs the same bit operation for signed and unsigned; `sdiv` performs signed division and `udiv` performs unsigned division; `icmp slt` is a signed comparison while `icmp ult` is unsigned. This cleanly separates the representation from the interpretation, which is the right split for an optimising compiler that may need to reason about both interpretations of the same bits simultaneously.

```llvm
; Signed and unsigned division of the same value — same type, different opcodes
define i32 @divtest(i32 %x, i32 %y) {
  %s = sdiv i32 %x, %y        ; signed: treats bits as two's complement
  %u = udiv i32 %x, %y        ; unsigned: treats bits as non-negative
  %r = add i32 %s, %u
  ret i32 %r
}
```

The most common widths map directly to C fundamental types: `i8` (char), `i16` (short), `i32` (int), `i64` (long/long long). However, `i1` plays a special role as LLVM's boolean type — the result of every comparison (`icmp`, `fcmp`) is `i1`, and `select`, `br`, and `zext i1 to i8` all consume `i1` values. On hardware, a `i1` is typically carried in a GP register, occupying a full word; the backend legalizes it appropriately.

### 17.1.2 Unusual widths

Non-power-of-two widths are first-class citizens. Practical examples:

| Width | Use case |
|-------|----------|
| `i1` | Boolean; result of `icmp`/`fcmp`; branch condition |
| `i24` | 24-bit audio PCM samples (common in audio DSP) |
| `i48` | MAC addresses, some packed timestamp formats |
| `i128` | 128-bit integers for 128-bit atomics, UUID, cryptographic state |
| `i256` | BigInteger arithmetic, cryptographic field elements (e.g., ECDSA curve points) |
| `i1024` | Arbitrary-precision intermediate values in crypto libraries |

The LLVM backend does not emit hardware instructions for unusual widths directly. Instead, it *legalizes* them: non-native widths are either promoted (widened to the next native width), expanded (split into multiple native-width values), or custom-lowered. An `i24` add on x86-64 will be widened to `i32` during SelectionDAG legalization. An `i256` add on a 64-bit target will be expanded to four 64-bit additions with carry propagation.

### 17.1.3 Widening, narrowing, and bitwise operations

Three instructions convert between integer widths, all of which are zero-cost in the sense that they generate no computation — they reinterpret bit patterns:

```llvm
define i32 @widen(i8 %a, i16 %b) {
  %se = sext i8  %a to i32    ; sign-extend: fills upper bits with sign bit
  %ze = zext i16 %b to i32    ; zero-extend: fills upper bits with 0
  %tr = trunc i32 %se to i8   ; truncate: drops upper bits
  ret i32 %ze
}
```

`sext` is used when the source is logically signed (e.g., promoting a C `char` to `int`); `zext` is used when it is logically unsigned (e.g., promoting a C `unsigned char`). The front end — not the IR — is responsible for choosing correctly.

### 17.1.4 Type legalization and backend behavior

When an LLVM backend encounters a type that its target does not natively support, the `TargetLowering` framework performs one of three legalizations:

- **Promote**: widen a narrow integer to the smallest supported native type. On most 64-bit targets, `i24` is promoted to `i32`. The backend synthesizes a mask or shift as needed to preserve semantics. For `zext i24 %x to i32`, the result is simply a 32-bit value with the upper byte zeroed — often free because the `i24` value was already in a 32-bit register.
- **Expand**: split a wide integer into multiple native-width values. An `i128` addition on a 64-bit target that does not have 128-bit arithmetic (e.g., RISC-V before the V or Zacas extensions) becomes two `add` instructions with carry propagation via `addc`/`adde` pseudo-instructions, or on architectures without carry flags, an explicit comparison-and-conditional-add sequence.
- **Custom**: target-specific lowering code handles the type. Some GPU targets define custom lowering for `i1` vectors (turning them into predicate registers) and for wide integers that match hardware capability.

The key takeaway for IR authors is that **the type system is target-agnostic**: writing `i24` in your IR is legal and correct even on x86-64. The backend will make it work. The choice of unusual widths should be driven by semantic correctness (representing a 24-bit PCM sample without phantom signedness from a wider representation) rather than fear of backend cost, which is usually zero or near-zero after legalization.

### 17.1.5 Integer types in the C++ API

Integer types are singletons managed by `LLVMContext`. The `IntegerType::get(Context, N)` static method returns the unique `IntegerType *` for width N within that context — calling it twice for the same width returns the same pointer. This uniquing is cheap: LLVM maintains a dense map of `IntegerType *` indexed by width. Comparing two integer types for equality is therefore a pointer comparison, not a structural equality check.

```cpp
#include "llvm/IR/Type.h"
#include "llvm/IR/LLVMContext.h"

LLVMContext Ctx;
Type *I32  = Type::getInt32Ty(Ctx);    // shorthand for IntegerType::get(Ctx, 32)
Type *I1   = Type::getInt1Ty(Ctx);
Type *I8   = Type::getInt8Ty(Ctx);
Type *I24  = IntegerType::get(Ctx, 24);

// Equality is pointer equality — uniqued within a context
assert(I32 == IntegerType::get(Ctx, 32));

// Query width
unsigned w = cast<IntegerType>(I32)->getBitWidth();   // returns 32
```

The shorthand methods `getInt1Ty`, `getInt8Ty`, `getInt16Ty`, `getInt32Ty`, `getInt64Ty`, and `getInt128Ty` are defined in [`llvm/include/llvm/IR/Type.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/Type.h).

---

## 17.2 Floating-Point Types

LLVM provides six distinct floating-point types, each modelling a hardware format with different precision-range tradeoffs. All are IEEE 754 compliant to the degree the hardware supports, with specific exceptions noted below.

### 17.2.1 The standard types

| Type | Width | Significand | Exponent | Notes |
|------|-------|-------------|----------|-------|
| `half` | 16 bits | 10+1 bits | 5 bits | IEEE 754-2008 binary16 |
| `bfloat` | 16 bits | 7+1 bits | 8 bits | Brain Float 16; ML-oriented |
| `float` | 32 bits | 23+1 bits | 8 bits | IEEE 754 binary32; the default float |
| `double` | 64 bits | 52+1 bits | 11 bits | IEEE 754 binary64; the default double |
| `fp128` | 128 bits | 112+1 bits | 15 bits | IEEE 754 binary128; quad precision |
| `x86_fp80` | 80 bits | 64 bits | 15 bits | Intel 80-bit extended; x87 heritage |
| `ppc_fp128` | 128 bits | 2×53 bits | 2×11 bits | IBM double-double; PowerPC long double |

### 17.2.2 `bfloat`: the Machine-Learning type

`bfloat` (Brain Float 16) was first-class in LLVM from version 11 onward, driven by its adoption in machine-learning accelerators. Its key property is that it has the *same exponent range* as `float` (8-bit exponent, biased by 127, covering approximately 10^−38 to 10^38), with only 7 explicit mantissa bits instead of 23. In practice, `bfloat` can be thought of as a truncated `float`: the upper two bytes of a 32-bit float's IEEE representation *are* the bfloat16 representation of the same value.

This makes conversion between `bfloat` and `float` cheap on supported hardware — it is often a shift or a byte-lane selection — whereas `half` requires a full exponent re-encoding. The tradeoff is reduced precision: `bfloat` has about 2–3 decimal digits of precision versus 7 for `float`, which is sufficient for many neural-network weight representations and activations.

Hardware support for `bfloat` exists on Intel Sapphire Rapids (AMX and AVX-512 BF16 instructions), AMD's CDNA2/CDNA3 GPUs, ARM's Neoverse and recent Cortex-A cores (BF16 extension to NEON/SVE), and all Google TPUs.

In MLIR, `arith.extf` can promote a `bfloat` value to `float` (an extension operation from a narrower float type to a wider one). When lowering from the MLIR `arith` dialect to LLVM IR, this becomes an `fpext bfloat ... to float` instruction.

```llvm
; Widen bfloat to float, then operate, then narrow back
define bfloat @bfloat_accumulate(bfloat %a, bfloat %b) {
  %af = fpext bfloat %a to float
  %bf = fpext bfloat %b to float
  %sum = fadd float %af, %bf
  %result = fptrunc float %sum to bfloat
  ret bfloat %result
}
```

### 17.2.3 `fp128`: Quad Precision

`fp128` implements IEEE 754 binary128, providing approximately 34 decimal digits of precision. No mainstream general-purpose CPU has native `fp128` arithmetic in hardware (as of LLVM 22). Consequently, LLVM lowers `fp128` operations to `__float128` soft-float library calls on targets that do not support it natively. On x86-64 Linux, GCC's `libquadmath` or `compiler-rt` provides these routines. The ABI representation — 16 bytes, 16-byte aligned — is consistent across targets.

`fp128` is primarily used in numerical analysis software requiring high-precision intermediate results, in some PowerPC ABI contexts, and as a verification tool for checking the precision of lower-width implementations.

### 17.2.4 `x86_fp80`: The x87 Extended-Precision Format

`x86_fp80` is Intel's 80-bit extended-precision format, dating to the 8087 floating-point coprocessor (1980). It differs from all IEEE formats in one critical way: it has an *explicit integer bit* in the significand rather than an implicit leading 1. This gives it a 64-bit significand and a 15-bit exponent, supporting values from approximately 3.4×10^−4932 to 1.2×10^4932 with about 18–19 decimal digits of precision.

The primary consumer is the x87 FPU on x86 and x86-64 processors, which uses an 8-entry 80-bit register stack. The x87 unit has largely been superseded by SSE2/AVX for scalar and vector floating-point, but it remains the mandated representation for `long double` on x86-32 Linux/glibc and x86-64 Linux/glibc (where `long double` is 80-bit extended, stored in 12 or 16 bytes with padding).

ABI implications are significant: passing `x86_fp80` values on x86-64 Linux requires 16 bytes of stack space (ABI-mandated), even though the format is only 10 bytes. The `f80:128` field in the `DataLayout` string encodes the 128-bit ABI alignment, not a 128-bit value. Misunderstanding this is a common source of bugs when writing a new front end.

```llvm
; Long double multiplication on x86 Linux
define x86_fp80 @longdouble_mul(x86_fp80 %a, x86_fp80 %b) {
  %r = fmul x86_fp80 %a, %b
  ret x86_fp80 %r
}
```

### 17.2.5 `ppc_fp128`: IBM Double-Double

`ppc_fp128` is the PowerPC `long double` format, implemented as a pair of two `double` values with a defined mathematical relationship: the high `double` holds the leading-precision portion of the value, and the low `double` holds the error term (the residual that would otherwise be lost). The combined representation achieves approximately 31 decimal digits of precision by exploiting the 53-bit significand of each constituent `double`, but it is *not* IEEE 754 — it lacks a unique representation for many values and has undefined behavior for certain operations.

IBM and the GNU toolchain have been gradually transitioning PowerPC `long double` from double-double to IEEE 754 binary128 (`__float128`), driven by the interoperability problems that double-double's non-uniqueness creates. Existing code using `ppc_fp128` will remain in the wild for years. LLVM handles it as a distinct type with its own set of soft-float operations.

### 17.2.6 Floating-point semantics flags

The six floating-point types are just one dimension of floating-point behavior in LLVM IR. The other dimension is the set of *fast-math flags* that can be attached to individual FP instructions, modifying what IEEE 754 semantics the optimizer may assume:

| Flag | Meaning |
|------|---------|
| `nnan` | No NaN inputs or outputs; the optimizer may assume NaN does not occur |
| `ninf` | No Inf inputs or outputs |
| `nsz` | No signed zeros; `-0.0` and `+0.0` are interchangeable |
| `arcp` | Allow reciprocal: `x/y` may be replaced by `x * (1/y)` |
| `contract` | Allow FMA contraction: `a*b + c` may become a fused multiply-add |
| `afn` | Allow approximate functions: `sqrt`, `sin`, etc. may use lower-precision hardware intrinsics |
| `reassoc` | Allow reassociation: `(a+b)+c` may become `a+(b+c)` |
| `fast` | All of the above simultaneously |

These flags appear in the textual IR between the instruction keyword and the type:

```llvm
; Fast-math flags on individual instructions
define float @fast_dot(float %ax, float %ay, float %bx, float %by) {
  %t0 = fmul fast float %ax, %bx
  %t1 = fmul fast float %ay, %by
  %r  = fadd fast float %t0, %t1   ; may be contracted to FMA
  ret float %r
}

; nsz only: allows -0.0 / +0.0 folding but preserves NaN/Inf semantics
define float @nsz_add(float %a, float %b) {
  %r = fadd nsz float %a, %b
  ret float %r
}
```

Fast-math flags are an instruction-level annotation, not a type property. Two values of type `float` are the same type regardless of the flags on the instructions that computed them. The flags are recorded in `FPMathOperator` and accessible via `Instruction::getFastMathFlags()` in the C++ API. Chapter 19 covers the complete semantics of FP instructions in detail.

### 17.2.7 Special first-class types: `void`, `label`, `token`, `metadata`

Four types in LLVM IR serve special, non-computational roles.

**`void`** is the return type of functions that return nothing. It cannot be the type of any SSA value, cannot appear in aggregates, and cannot be used as a function argument type. Declaring a function as returning `void` is distinct from returning `i8*`/`ptr` null — a `void` function genuinely has no return value, and any attempt to `load` or `store` a `void` value is an IR verifier error.

```llvm
define void @nothing(i32 %x) {
  ; do some side effect
  store i32 %x, ptr @global, align 4
  ret void      ; required to end a void function
}
```

**`label`** is the type of basic-block identifiers. Labels are not values in the SSA sense — you cannot `add` two labels or store a label to memory. They appear only as operands of `br`, `switch`, and `indirectbr` instructions, and in `phi` node predecessor-block annotations. Internally, `LLVMContext` maintains a singleton `Type::LabelTyID` just as it does for `void`.

**`token`** is an opaque value type used by coroutine intrinsics (`@llvm.coro.id`, `@llvm.coro.save`) and certain exception-handling intrinsics (`@llvm.eh.padphi`). A token value represents an identity — it can be passed to other intrinsics that "consume" the same token — but it may not be inspected, duplicated, or converted to any other type. The optimizer must treat token-producing instructions as side-effecting and preserve their program-order position. If a token value is defined in a basic block but its consumer is not reachable, the token is dropped; but two distinct consumers of the same token would imply out-of-order execution of coroutine suspension points, which would be incorrect.

**`metadata`** appears exclusively as the type of arguments in certain intrinsic call sites, such as `@llvm.dbg.value(metadata i32 %x, metadata !DILocalVariable(...), metadata !DIExpression())`. It allows metadata nodes — which live outside the SSA value graph — to be passed alongside normal values to debug-info intrinsics. The metadata type is not a real computation type; you cannot have a `metadata`-typed SSA register, and it cannot appear in `load`/`store` instructions or user-defined functions.

---

## 17.3 Vectors: Fixed-Length and Scalable

LLVM's vector types model SIMD execution directly at the IR level. A vector type groups a fixed or runtime-variable number of scalar elements of the same type and exposes them to target-independent vectorization and to backend instruction selection.

### 17.3.1 Fixed-length vectors

The syntax `<N x T>` creates a vector of exactly N elements of scalar type T, where N must be a positive integer and T must be an integer, floating-point, or pointer type.

```llvm
; Common fixed-length vector types and operations
define <4 x float> @addvec(<4 x float> %a, <4 x float> %b) {
  ; Scalar opcodes applied element-wise to vectors
  %sum  = fadd <4 x float> %a, %b
  ; Lane extraction and insertion
  %lane = extractelement <4 x float> %sum, i32 0
  %v2   = insertelement  <4 x float> %sum, float %lane, i32 3
  ; Lane permutation
  %shuf = shufflevector <4 x float> %v2, <4 x float> zeroinitializer,
                        <4 x i32> <i32 3, i32 2, i32 1, i32 0>
  ret <4 x float> %shuf
}
```

Common fixed-vector types and their typical mappings:

| LLVM type | Hardware mapping |
|-----------|-----------------|
| `<4 x float>` | SSE `xmm` register (128-bit), ARM NEON `q` register |
| `<8 x float>` | AVX `ymm` register (256-bit) |
| `<16 x float>` | AVX-512 `zmm` register (512-bit) |
| `<8 x i32>` | AVX2 integer `ymm` register |
| `<16 x i8>` | SSE2 packed byte `xmm` register |

The correspondence is suggestive, not mandated. The backend's instruction selector and legalizer determine how a given vector type maps to hardware registers and instructions; a `<4 x float>` may be split into two `<2 x float>` operations or scalarized on a target without SIMD support.

### 17.3.2 Scalable vectors

Scalable vectors have the syntax `<vscale x N x T>`. At compile time, N is known; `vscale` is a positive integer that is determined at runtime by reading a hardware register (e.g., the `VL` register on RISC-V RVV, or the SVE vector length on AArch64). The total number of elements is `N × vscale`.

```llvm
; Scalable vector fadd — same opcode as fixed, different type
define <vscale x 4 x float> @scalable_add(<vscale x 4 x float> %a,
                                           <vscale x 4 x float> %b) {
  %r = fadd <vscale x 4 x float> %a, %b
  ret <vscale x 4 x float> %r
}

; Query the runtime scale factor
declare i64 @llvm.vscale.i64()

define i64 @get_element_count() {
  %vs    = call i64 @llvm.vscale.i64()
  %total = mul i64 %vs, 4       ; 4 * vscale = actual number of f32 lanes
  ret i64 %total
}
```

Scalable vectors are central to ARM SVE and RISC-V RVV — both ISAs define a vector length that varies across implementations (e.g., an SVE implementation may have 128-bit to 2048-bit vectors, always a multiple of 128 bits). The key invariant that LLVM's scalable-vector model enforces is that the element count is always a compile-time-unknown but runtime-constant multiple of N. This enables loop vectorization to generate code that is correct across the entire family of implementations without runtime branches.

The optimizer must conservatively assume that scalable-vector operations may touch more memory than fixed-vector operations of the same declared width; this affects alias analysis and memory dependency tracking.

### 17.3.3 Predicated vector operations

The `llvm.vp.*` intrinsic family provides masked (predicated) operations over both fixed and scalable vectors. A predicate mask of type `<N x i1>` (or `<vscale x N x i1>`) and an active element count determine which lanes participate.

```llvm
declare <4 x float> @llvm.vp.fadd.v4f32(<4 x float>, <4 x float>,
                                          <4 x i1>, i32)

define <4 x float> @masked_add(<4 x float> %a, <4 x float> %b,
                                <4 x i1> %mask) {
  ; Only add lanes where mask bit is 1; %evl = 4 means all 4 lanes eligible
  %r = call <4 x float> @llvm.vp.fadd.v4f32(<4 x float> %a, <4 x float> %b,
                                              <4 x i1> %mask, i32 4)
  ret <4 x float> %r
}
```

---

## 17.4 Arrays and Structs

### 17.4.1 Arrays

The array type `[N x T]` represents a contiguous sequence of N elements of type T, where N must be a non-negative integer. Zero-length arrays (`[0 x T]`) are permitted and are used to represent C99 flexible array members — the trailing `char data[]` field in a struct.

```llvm
; A fixed-size array and a struct with a flexible array member
%FixedBuffer = type { i32, [256 x i8] }
%FlexMsg     = type { i32, i32, [0 x i8] }   ; length, type, data[]

define void @array_access(ptr %p, i32 %idx) {
  ; Multi-level GEP: outer struct index 0 (first FixedBuffer), inner array index %idx
  %elem = getelementptr %FixedBuffer, ptr %p, i32 0, i32 1, i32 %idx
  store i8 0, ptr %elem, align 1
  ret void
}
```

Multidimensional arrays are represented by nesting: `[4 x [4 x float]]` is a 4×4 matrix of floats in row-major order. GEP navigates through the nesting level by level.

### 17.4.2 Named and anonymous structs

Struct types in LLVM IR have two flavors: **literal** (anonymous) structs and **named** structs.

A **literal struct** is written `{T1, T2, ...}` directly in the type position. Two literal structs with the same sequence of field types are the *same type* — structural equality applies. Literal structs cannot be recursive.

A **named struct** is declared at module scope with `%Name = type {...}` and referenced by name. Named structs are identified by name within a module: two structs with different names but identical fields are *different types*. This name-based identity is important for preserving type provenance through the optimizer — Clang names all C++ class types so that debug info and ABI-sensitive code generation can distinguish structurally identical but semantically different types.

```llvm
; Named struct declarations at module scope
%Point3D = type { float, float, float }
%RGB     = type { float, float, float }   ; structurally identical to Point3D but distinct

; Literal struct used inline — structural identity
define { i32, float } @make_pair(i32 %a, float %b) {
  %r = insertvalue { i32, float } undef, i32 %a, 0
  %s = insertvalue { i32, float } %r, float %b, 1
  ret { i32, float } %s
}
```

### 17.4.3 Packed structs

By default, structs include padding between fields to satisfy alignment requirements. The packed syntax `<{T1, T2, ...}>` eliminates all padding; fields are laid out at consecutive bytes regardless of alignment. This matches C's `__attribute__((packed))` and Rust's `#[repr(packed)]`.

```llvm
; Unpacked: i8 padded to 4-byte boundary before i32
%Unpacked = type { i8, i32 }         ; total size: 8 bytes on most targets

; Packed: no padding — i32 starts at byte offset 1
%Packed   = type <{ i8, i32 }>       ; total size: 5 bytes

define void @packed_write(ptr %p) {
  ; Packed field access — note the reduced alignment
  %field = getelementptr %Packed, ptr %p, i32 0, i32 1
  store i32 42, ptr %field, align 1  ; align 1 required for unaligned access
  ret void
}
```

Using align 1 on stores and loads into packed struct fields is critical. LLVM's memory access instructions require the alignment annotation to accurately reflect the actual memory alignment; omitting it or specifying too-large alignment triggers undefined behavior and can produce incorrect code on architectures that fault on unaligned accesses (e.g., strict-alignment ARM configurations, RISC-V without the "Zicclsm" relaxed misaligned access extension).

### 17.4.4 Opaque structs

An opaque struct declaration `%Name = type opaque` declares a named type without defining its body. This is LLVM IR's analogue of a C forward declaration: the type can be used as the target type of a `ptr` operand, but cannot be sized, loaded, or stored directly.

```llvm
%OpaqueHandle = type opaque
%Node         = type { i32, ptr }   ; recursive: next pointer as ptr

; Opaque types enable library handles and PIMPL patterns
declare ptr @create_handle()
declare void @use_handle(ptr %h)
```

Recursive types — a struct that contains a pointer to the same struct, as in a linked list — are expressed via named structs with `ptr` for the self-referential field. Because `ptr` is now opaque (Section 17.5), there is no cycle in the type graph: `%Node` has type `{ i32, ptr }`, not `{ i32, ptr %Node }`. The semantic link between the `ptr` field and the `%Node` type lives only in the frontend's bookkeeping and in debug metadata.

### 17.4.5 Struct types in the C++ API

The [`StructType`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/DerivedTypes.h) class represents both named and literal structs. Key factory methods:

```cpp
#include "llvm/IR/DerivedTypes.h"

LLVMContext &Ctx = ...;

// Literal struct: identified by structure, uniqued within a context
StructType *LitST = StructType::get(Ctx,
                      {Type::getInt32Ty(Ctx), Type::getFloatTy(Ctx)});

// Named struct: create empty, then set body (allows forward declarations)
StructType *NamedST = StructType::create(Ctx, "MyStruct");
NamedST->setBody({Type::getInt32Ty(Ctx), Type::getDoubleTy(Ctx)});

// Packed named struct
StructType *PackedST = StructType::create(Ctx, "PackedStruct");
PackedST->setBody({Type::getInt8Ty(Ctx), Type::getInt32Ty(Ctx)},
                  /*isPacked=*/true);

// Check if a struct has been given a body
if (NamedST->isOpaque())
  llvm::errs() << "No body defined yet\n";

// Iterate over fields
for (unsigned i = 0; i < NamedST->getNumElements(); ++i)
  llvm::errs() << "Field " << i << ": " << *NamedST->getElementType(i) << "\n";
```

Literal structs are uniqued by the `LLVMContext`: calling `StructType::get(Ctx, {i32, float})` twice returns the same pointer. Named structs are not uniqued by structure — two calls to `StructType::create(Ctx, "Foo")` with different bodies produce two distinct `StructType *` objects. This mirrors the textual IR semantics described above: named structs are identified by name within a module, and it is the front end's responsibility to ensure names are consistent.

---

## 17.5 Opaque Pointers: The Migration and the New Model

### 17.5.1 The old world: typed pointers

Prior to LLVM 15, every pointer type carried its pointee type explicitly: `i32*`, `float*`, `i8*`, `%MyStruct*`. The intent was to preserve type information so that downstream passes could determine what kind of object a pointer pointed to. In practice, this intent was never fully realised:

- **Alias analysis ignores pointer types.** The primary alias analysis algorithms — basic AA, TBAA (Type-Based AA), DSAA — use metadata annotations and argument attributes, not the pointer type itself, to determine aliasing. A `i32*` and a `float*` pointing to the same bytes are not proven non-aliasing by their types alone.
- **Bitcasts proliferated.** Whenever code needed to treat a `float*` as `i32*`, a `bitcast` was required. High-level code dealing with opaque library handles (e.g., `void*`-based C APIs) generated dense chains of `bitcast` instructions that cluttered the IR and made optimization harder.
- **Type identity was unstable.** Named struct types carried names through the module, but when linking two modules, identically-named structs needed renaming (`%MyStruct.0`, `%MyStruct.1`). This made linker-time optimization and LTO fragile.
- **No correctness guarantee was provided.** A front end could freely `bitcast` pointers to arbitrary types, making the type annotation unreliable for optimization purposes.

The conclusion, reached incrementally over LLVM 12–15 and completed in LLVM 15, was that typed pointers provided neither the safety guarantees of a true type-safe IR nor the flexibility of an untyped IR — they occupied an uncomfortable middle ground that imposed cost without benefit.

### 17.5.2 The new world: opaque `ptr`

As of LLVM 15, and exclusively so in LLVM 17+ (typed pointers fully removed), all pointer types are expressed as `ptr`. There is one pointer type per address space: `ptr` for address space 0, `ptr addrspace(1)` for address space 1, and so on. The pointee type is gone from the pointer itself.

The type information is not lost — it moves to the instruction. `load` and `store` carry an explicit type operand:

```llvm
; Old typed-pointer style (LLVM ≤ 14, no longer valid)
; %val = load i32, i32* %ptr, align 4
; store i32 %val, i32* %ptr2, align 4

; New opaque-pointer style (LLVM 15+, required in LLVM 22)
%val = load i32, ptr %ptr,  align 4
store i32 %val, ptr %ptr2, align 4
```

`getelementptr` similarly carries an explicit base type:

```llvm
%Point = type { float, float, float }

define float @get_z(ptr %p) {
  ; GEP carries the pointee type explicitly as first argument
  %zptr = getelementptr %Point, ptr %p, i32 0, i32 2
  %z    = load float, ptr %zptr, align 4
  ret float %z
}
```

The `bitcast` instruction for pointer-to-pointer casts has become a no-op (it still exists in the grammar for source compatibility but optimizes away immediately). `addrspacecast` remains meaningful for crossing address space boundaries.

### 17.5.3 Benefits of opaque pointers

**IR size reduction.** Without type decorations on pointers, `bitcast` chains vanish. In practice, migrated programs show 5–15% IR size reductions.

**Simpler alias analysis.** Alias analysis passes no longer need to strip `bitcast` chains to find the underlying pointer; they work directly on the base `ptr` values.

**Cleaner linker-time IR.** Struct type renaming on LTO link is no longer needed for pointer types. Named struct types still exist but appear only where explicitly referenced (in GEP and load/store type annotations), not embedded in every pointer type.

**Easier MLIR lowering.** The MLIR LLVM dialect maps `!llvm.ptr` directly to LLVM's `ptr`, simplifying the lowering pipeline.

### 17.5.4 Implications for reading existing documentation and code

A significant fraction of LLVM documentation, blog posts, and IR dumps written before LLVM 15 uses typed pointers. When reading such material, mentally substitute:

- `T*` → `ptr`
- `bitcast T* %p to U*` → no-op; just `%p`
- `load T, T* %p` → `load T, ptr %p`
- `getelementptr T, T* %p, ...` → `getelementptr T, ptr %p, ...`

The semantics are unchanged; only the syntactic decoration on the pointer is removed.

### 17.5.5 Migrating existing IR and passes

If you maintain code that emits LLVM IR or inspects pointer types in a pass, migration to opaque pointers requires:

1. **Remove uses of `PointerType::getElementType()`** — this method was removed in LLVM 17. Anywhere a pass used to call `PtrTy->getElementType()` to find out what type a pointer pointed to, it must instead use context from the instruction that *uses* the pointer. For a `load` instruction, the pointee type is `LI.getType()`. For a `store`, it is `SI.getValueOperand()->getType()`. For a `getelementptr`, it is `GEP.getSourceElementType()`.

2. **Replace `Type::getPointerTo()`** — instead of `T->getPointerTo(AS)`, use `PointerType::get(Ctx, AS)` or simply `PointerType::getUnqual(Ctx)` for address space 0.

3. **`bitcast` for pointer-to-pointer casts becomes a no-op** — two `ptr` values in the same address space are the same type; no cast is needed. Where the old code emitted `bitcast i8* %p to i32*`, simply pass `%p` directly.

4. **The `IRBuilder` API** updated all factory methods: `CreateGEP` now takes an explicit element type as its first argument; `CreateLoad` takes the loaded type; `CreateStore` infers it from the value.

```cpp
// Old (typed pointer) style — no longer compiles in LLVM 22
// Value *GEP = Builder.CreateGEP(PtrVal, Indices);
// Value *Load = Builder.CreateLoad(PtrVal);

// New (opaque pointer) style
Value *GEP  = Builder.CreateGEP(StructType, PtrVal, Indices);
Value *Load = Builder.CreateLoad(ElementType, PtrVal, "val");
```

The [`llvm/utils/update_test_checks.py`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/utils/update_test_checks.py) script automatically regenerates `CHECK` lines in lit tests with opaque-pointer-style IR, which handles the bulk of mechanical test migration.

---

## 17.6 Address Spaces

### 17.6.1 Motivation

Modern compute platforms expose multiple distinct memory regions with different addressing, caching, and coherence properties. A CUDA GPU has global DRAM, per-block shared memory, per-thread local memory, constant memory, and texture memory — each with different latency, bandwidth, and access rules. A CPU with memory-mapped I/O has regular RAM and device memory regions. Address spaces encode these distinctions in LLVM IR, enabling the backend to emit correct load and store instructions and preventing the optimizer from incorrectly reasoning that two pointers in different spaces may alias.

### 17.6.2 Syntax

`ptr addrspace(N)` for any non-negative integer N. `ptr` is shorthand for `ptr addrspace(0)`. The semantics of each address space are target-defined; the IR optimizer treats pointers in different address spaces as non-aliasing by default.

```llvm
; NVPTX (CUDA) address space usage
define void @cuda_kernel(ptr addrspace(1) %global,   ; global DRAM
                          ptr addrspace(3) %shared) { ; per-block shared
  %g_val = load i32, ptr addrspace(1) %global, align 4
  store i32 %g_val, ptr addrspace(3) %shared, align 4
  ; Cast from global to generic address space
  %generic = addrspacecast ptr addrspace(1) %global to ptr
  ret void
}
```

### 17.6.3 Target address space conventions

| Target | AS 0 | AS 1 | AS 2 | AS 3 | AS 4 | AS 5 |
|--------|------|------|------|------|------|------|
| NVPTX (CUDA) | Generic | Global | Internal | Shared | Constant | Local |
| AMDGPU | Flat/generic | Global | — | LDS (local) | Constant | Private |
| SPIR-V | — | Cross-workgroup | — | Workgroup | Uniform constant | — |
| x86 (segmented) | Default | — | — | — | — | — |

The `addrspacecast` instruction converts a pointer from one address space to another where the target defines a mapping. It is *not* a `bitcast` — the hardware may need to prepend an address-space segment register value or perform a pointer-compression/expansion step.

The `DataLayout` string's `p[N]:size:abi:pref` specifiers allow different address spaces to have different pointer sizes. On AMDGPU, `ptr addrspace(3)` (LDS) uses 32-bit pointers while the default (flat) address space uses 64-bit pointers.

---

## 17.7 Target Extension Types

### 17.7.1 Motivation

Some targets expose opaque hardware resources that have no natural representation in LLVM's type system: GPU texture objects, SVE predicate registers, SPIR-V image handles, AArch64 streaming-mode predicate-count registers. These resources must be passed through the IR from the frontend to the backend, but they cannot be stored in memory in the normal sense, may not be copied arbitrarily, and must not be inspected or transformed by the generic optimizer.

Target extension types, introduced in LLVM 15 and stabilized in LLVM 17, address this need. The syntax is:

```
target("<name>", <type-params...>, <int-params...>)
```

where `<name>` is a target-defined string, followed by optional type and integer parameters.

### 17.7.2 Examples

```llvm
; SPIR-V image type — a GPU texture handle
; target("spirv.Image", element_type, dimensionality, depth, arrayed, MS, sampled, format)
declare target("spirv.Image") @get_texture()

; AArch64 SVE predicate count register type
; target("aarch64.svcount")
declare target("aarch64.svcount") @get_svcount()
```

The target extension type is verified to be used consistently but otherwise treated as opaque by the generic optimizer. The backend's instruction selection machinery interprets it and maps it to the appropriate register class and instructions. Target extension types:

- Cannot be stored to or loaded from memory unless the target explicitly declares the `HasZeroInit` or `CanBeLocal` properties.
- Cannot be used as function return types or parameter types in the generic calling convention without a target-specific annotation.
- Are not subject to generic copy-propagation, constant folding, or dead-store elimination.

---

## 17.8 The `DataLayout` String

### 17.8.1 Purpose and location

LLVM's type system defines types *abstractly* — `i32` is a 32-bit integer, `%MyStruct` has a certain field sequence — but it does not specify how those types map to bytes, registers, and stack slots. That mapping is provided by the `target datalayout` string, which appears at the top of every LLVM IR module and looks like this:

```llvm
; x86-64 Linux (Clang 22, verified empirically)
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"
```

The string is a dash-separated sequence of specifiers. LLVM's `DataLayout` class (defined in [`llvm/lib/IR/DataLayout.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/IR/DataLayout.cpp)) parses this string and exposes a rich API for querying sizes and alignments.

### 17.8.2 Specifier reference

| Specifier | Meaning |
|-----------|---------|
| `e` | Little-endian byte order |
| `E` | Big-endian byte order |
| `m:e` | Symbol mangling: ELF (e=ELF, o=MachO, w=COFF, m=MIPS COFF) |
| `p[N]:size:abi[:pref[:idx]]` | Pointer in address space N: `size` bits, ABI alignment `abi` bits, preferred alignment `pref` bits |
| `p` (no N) | Shorthand for address space 0 |
| `i<n>:abi[:pref]` | Integer type `i<n>`: ABI alignment and preferred alignment |
| `f<n>:abi[:pref]` | Floating-point type of width `n` |
| `v<n>:abi[:pref]` | Vector type of width `n` |
| `a:abi[:pref]` | Aggregate (struct/array) ABI alignment |
| `n<n1>:<n2>:...` | Native integer widths supported by the CPU |
| `S<n>` | Stack natural alignment in bits |
| `A<n>` | Address space used by `alloca` instructions |
| `P<n>` | Address space for program addresses (code pointers) |
| `G<n>` | Address space for global variables |
| `Fn<n>` | Function pointer alignment |

### 17.8.3 Annotated real examples

**x86-64 Linux:**

```
"e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128"
```

Reading left to right:

- `e` — little-endian
- `m:e` — ELF symbol mangling
- `p270:32:32` — address space 270 (x86 code16 segment) uses 32-bit pointers, 32-bit ABI alignment
- `p271:32:32` — address space 271 (x86 code32 segment) — 32-bit pointers, 32-bit ABI alignment
- `p272:64:64` — address space 272 (x86 code64 segment) — 64-bit pointers, 64-bit ABI alignment
- `i64:64` — `i64` has 64-bit ABI alignment (in contrast to x86-32 where it is 32-bit)
- `i128:128` — `i128` has 128-bit ABI alignment
- `f80:128` — `x86_fp80` (80-bit float) has 128-bit ABI alignment (stored as 16 bytes with 6 bytes padding)
- `n8:16:32:64` — native integer widths are 8, 16, 32, 64 bits
- `S128` — stack is 128-bit (16-byte) aligned at call boundaries (the x86-64 ABI requirement)

**AArch64 Linux:**

```
"e-m:e-p270:32:32-p271:32:32-p272:64:64-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128-Fn32"
```

Notable differences from x86-64:

- `i8:8:32` — `i8` has 8-bit ABI alignment but 32-bit *preferred* alignment (ARM prefers 32-bit loads even for single bytes, to avoid partial-word memory ops)
- `i16:16:32` — similarly, `i16` prefers 32-bit alignment
- `n32:64` — native integer widths are 32 and 64 bits (no 8/16 native arithmetic)
- `Fn32` — function pointers have 32-bit alignment (relevant for Thumb interworking stubs on 32-bit ARM; included in AArch64 for toolchain uniformity)

**RISC-V 64-bit:**

```
"e-m:e-p:64:64-i64:64-i128:128-n32:64-S128"
```

The simplest of the three — no segment-register address spaces, no preferred-alignment overrides for narrow integers:

- `p:64:64` — default pointer is 64-bit, 64-bit ABI aligned
- `n32:64` — native widths are 32 and 64 bits
- `S128` — 16-byte stack alignment (RISC-V calling convention requirement)

### 17.8.4 The `DataLayout` C++ API

The [`DataLayout`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/IR/DataLayout.h) class exposes the following commonly-used methods:

| Method | Returns |
|--------|---------|
| `getTypeAllocSize(Type *Ty)` | Total allocation size in bytes, including any trailing padding |
| `getTypeSizeInBits(Type *Ty)` | Exact bit size (no trailing padding) |
| `getABITypeAlign(Type *Ty)` | ABI alignment in bytes (minimum required by the ABI) |
| `getPrefTypeAlign(Type *Ty)` | Preferred alignment in bytes (what the hardware prefers for performance) |
| `getPointerSize(unsigned AS)` | Pointer size in bytes for a given address space |
| `getPointerSizeInBits(unsigned AS)` | Pointer size in bits |
| `getIndexType(LLVMContext &C, unsigned AS)` | Integer type used for indexing (pointer-sized integer) |
| `getStructLayout(StructType *Ty)` | Returns a `StructLayout` with per-field offsets |

The `StructLayout` object (returned by `getStructLayout`) provides `getElementOffset(unsigned Idx)` for the byte offset of each field, and `getSizeInBytes()` for the total struct size. This is the authoritative source for struct layout in LLVM; front ends that compute struct layouts independently risk diverging from what the backend will produce.

A typical usage pattern in a pass:

```cpp
#include "llvm/IR/DataLayout.h"
#include "llvm/IR/Module.h"

void inspectType(const Module &M, StructType *ST) {
  const DataLayout &DL = M.getDataLayout();
  const StructLayout *SL = DL.getStructLayout(ST);

  for (unsigned i = 0; i < ST->getNumElements(); ++i) {
    Type *FieldTy = ST->getElementType(i);
    uint64_t offset = SL->getElementOffset(i);
    Align   abi    = DL.getABITypeAlign(FieldTy);
    llvm::errs() << "Field " << i << ": offset=" << offset
                 << " abi_align=" << abi.value() << "\n";
  }
  llvm::errs() << "Total size: " << SL->getSizeInBytes() << " bytes\n";
}
```

### 17.8.5 ABI alignment vs. preferred alignment

The distinction between ABI alignment (`getABITypeAlign`) and preferred alignment (`getPrefTypeAlign`) is subtle but important:

- **ABI alignment** is the minimum alignment that the platform ABI mandates for a type when it appears in a struct or on the stack. Violating ABI alignment causes undefined behavior (on strict-alignment targets) or a performance penalty (on tolerant targets like x86). The ABI alignment is what matters for struct layout, function argument passing, and `alloca`.
- **Preferred alignment** is the alignment at which the hardware performs most efficiently — usually the native word or cache-line size. On AArch64, an `i8` field has ABI alignment 1 but the backend may prefer to load it from a 4-byte-aligned address to avoid a read-modify-write cycle. The preferred alignment is used as a hint for `alloca` and global variable placement.

---

## 17.9 Chapter Summary

- **`iN` integers** encode any bit width from 1 to 8,388,607 without built-in signedness; signedness is a property of the instruction (`sdiv` vs. `udiv`, `sext` vs. `zext`). The backend legalizes non-native widths to the hardware's native word sizes.

- **Floating-point types** span six formats: `half` (IEEE binary16), `bfloat` (ML-oriented 8+7-bit format, same exponent as `float`), `float`, `double`, `fp128` (IEEE binary128, soft-float on most targets), `x86_fp80` (Intel x87 with explicit integer bit, significant ABI implications), and `ppc_fp128` (IBM double-double, not IEEE-compliant).

- **Fixed-length vectors** `<N x T>` group N scalar elements for SIMD operations; **scalable vectors** `<vscale x N x T>` add a runtime-determined scale factor for length-agnostic code targeting ARM SVE and RISC-V RVV. Scalar instruction opcodes (`fadd`, `mul`, etc.) apply element-wise to both vector flavors.

- **Arrays** `[N x T]` are contiguous, zero-padded (zero-length arrays model flexible array members). **Structs** come in named (module-scope, name-identity), anonymous (structural-identity), packed (`<{...}>`, no padding), and opaque (`type opaque`, body-less forward declaration) forms. Recursive types require named structs with `ptr` self-referential fields.

- **Opaque pointers** (`ptr`) replaced typed pointers (`T*`) starting in LLVM 15 and are the only pointer form in LLVM 22. The pointee type moved from the pointer to the `load`/`store`/`getelementptr` instructions. This eliminated pervasive `bitcast` chains, simplified alias analysis, and reduced IR size.

- **Address spaces** (`ptr addrspace(N)`) encode distinct memory regions on GPU and heterogeneous targets. The optimizer treats cross-address-space aliasing conservatively; the backend maps each address space to the correct hardware load/store variants. `addrspacecast` crosses address-space boundaries.

- **Target extension types** (`target("name", ...)`) carry opaque hardware-resource types — GPU texture handles, SVE predicate registers, SPIR-V images — through the IR to the backend without exposing their internal structure to the generic optimizer.

- **The `DataLayout` string** translates abstract LLVM types to physical byte layouts: endianness, pointer sizes per address space, ABI and preferred alignment for each type, native integer widths, and stack alignment. The `DataLayout` C++ API (`getTypeAllocSize`, `getABITypeAlign`, `getStructLayout`) is the authoritative source for layout decisions in both the frontend and backend.

---

*Cross-references: [Chapter 16 — IR Structure](../part-04-llvm-ir/ch16-ir-structure.md) introduced the module-level context in which type declarations appear. [Chapter 19 — Instructions I — Arithmetic and Memory](../part-04-llvm-ir/ch19-instructions-i-arithmetic-and-memory.md) uses every type covered here in the context of GEP, load, and store. [Chapter 23 — Attributes, Calling Conventions, and the ABI](../part-04-llvm-ir/ch23-attributes-calling-conventions-and-the-abi.md) shows how types interact with the platform ABI at call boundaries.*
