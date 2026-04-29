# Chapter 99 — The RISC-V Vector Extension (RVV)

*Part XV — Targets*

The RISC-V Vector Extension (RVV 1.0, ratified November 2021) represents a fundamental departure from fixed-width SIMD architectures. Rather than defining a fixed vector length, RVV allows the hardware to implement any vector length that is a power-of-two multiple of 64 bits, from 128 bits up to 65536 bits. Programs written against the RVV ISA automatically scale to utilize wider hardware without recompilation. This scalable-vector model maps directly to LLVM's `<vscale x N x T>` type system, and the integration reaches from the low-level V extension instruction selection all the way to the Loop Vectorizer's EVL-based tail folding. This chapter covers the complete RVV software stack as implemented in LLVM 22.1.x: the architecture, LLVM type model, loop vectorization, intrinsic API families, masking semantics, and tuple types for segmented memory operations.

---

## 99.1 V Extension Architecture

### 99.1.1 Vector Registers and VLENB

RVV provides 32 vector registers (v0–v31). Each register holds VLEN bits, where VLEN is implementation-defined and is a power-of-two multiple of 64 bits (128, 256, 512, 1024, or 2048 bits are common commercial implementations). `VLENB` (vector length in bytes = VLEN/8) is readable at runtime from the `vlenb` CSR, enabling programs to adapt without recompilation:

```asm
; Query VLENB (bytes per vector register)
csrr a0, vlenb       ; a0 = VLEN/8
slli a0, a0, 3       ; a0 = VLEN (in bits)

; Example: VLEN=256 → vlenb=32, loop stride = 32/4 = 8 float32s per register
```

### 99.1.2 The `vtype` CSR

Before any vector operation, software configures the vector type register (`vtype`) via `VSETVLI`, `VSETVL`, or `VSETIVLI`. The `vtype` CSR encodes:

- **SEW** (Selected Element Width): 8, 16, 32, or 64 bits per element
- **LMUL** (Length MULtiplier): how many physical registers form one logical vector register
- **vta** (vector tail agnostic): 1 = processor may write anything to tail elements, 0 = tail elements retain old value (undisturbed)
- **vma** (vector mask agnostic): 1 = processor may write anything to masked-off elements, 0 = undisturbed

```asm
; Set SEW=32, LMUL=1, tail-agnostic, mask-agnostic
; Set vl = avl (application vector length requested in a0)
; Result vl (actual length granted) written to t0
vsetvli t0, a0, e32, m1, ta, ma

; Set SEW=8, LMUL=4 (4 registers grouped as one logical register)
vsetvli t0, a0, e8, m4, ta, ma

; Immediate avl form (for loop headers where avl ≤ 31 and LMUL/SEW are fixed)
vsetivli t0, 8, e32, m1, ta, ma  ; avl=8 as immediate
```

### 99.1.3 LMUL: Fractions and Multiples

LMUL controls how many physical vector registers form one logical vector register:

| LMUL encoding | LMUL value | Registers grouped | Max elements (VLEN=256, SEW=32) | LLVM type (SEW=32) |
|--------------|-----------|------------------|-------------------------------|-------------------|
| `mf8` | 1/8 | 1/8 of 1 register | 1 | `<vscale x 1 x i32>` (special) |
| `mf4` | 1/4 | 1/4 register | 2 | `<vscale x 1 x i32>` |
| `mf2` | 1/2 | 1/2 register | 4 | `<vscale x 2 x i32>` |
| `m1` | 1 | 1 register | 8 | `<vscale x 4 x i32>` |
| `m2` | 2 | 2 registers (even+odd pair) | 16 | `<vscale x 8 x i32>` |
| `m4` | 4 | 4 registers (aligned group of 4) | 32 | `<vscale x 16 x i32>` |
| `m8` | 8 | 8 registers (aligned group of 8) | 64 | `<vscale x 32 x i32>` |

Fractional LMUL (mf2, mf4, mf8) reduces the number of elements per operation below one register's full capacity. This is useful when mixing element widths — for example, widening a `uint8` LMUL=1 result into `uint16` LMUL=2 requires LMUL=1 on the input.

### 99.1.4 SEW × LMUL Combinations and Element Count

The number of elements per operation = LMUL × VLEN / SEW (must be ≥ 1):

| SEW | VLEN=128 LMUL=1 | VLEN=256 LMUL=1 | VLEN=256 LMUL=4 | VLEN=512 LMUL=8 |
|-----|----------------|----------------|----------------|----------------|
| 8 | 16 | 32 | 128 | 512 |
| 16 | 8 | 16 | 64 | 256 |
| 32 | 4 | 8 | 32 | 128 |
| 64 | 2 | 4 | 16 | 64 |

The maximum elements per operation (VLMAX) is what `vsetvli` with `x0` as `avl` returns:

```asm
vsetvli t0, x0, e32, m4, ta, ma  ; t0 = VLMAX = 4 * VLEN/32
```

### 99.1.5 Tail and Mask Policies

- **Tail-Undisturbed (tu)**: Elements beyond `vl` retain their previous values from the destination register
- **Tail-Agnostic (ta)**: Hardware may write 1s or previous values to tail elements (permits optimization)
- **Mask-Undisturbed (mu)**: Masked-off elements retain their previous values
- **Mask-Agnostic (ma)**: Hardware may write 1s or previous values to masked-off elements

Using `ta, ma` (the most common production setting) gives the processor the most freedom to optimize.

---

## 99.2 LLVM's Scalable-Vector Type Model

### 99.2.1 `<vscale x N x T>` Types

LLVM represents RVV types using scalable vectors. The `vscale` runtime multiplier equals `VLENB / 8` (the number of 64-bit chunks in one vector register):

```llvm
; vscale = VLENB / 8 at runtime
; <vscale x 4 x i32> has 4 * vscale elements
;   VLEN=128: vscale=2, 8 elements
;   VLEN=256: vscale=4, 16 elements
;   VLEN=512: vscale=8, 32 elements

%v = alloca <vscale x 4 x i32>
```

The relationship between RVV type names and LLVM types:

| C type (riscv_vector.h) | SEW | LMUL | LLVM type |
|------------------------|-----|------|-----------|
| `vint8m1_t` | 8 | 1 | `<vscale x 8 x i8>` |
| `vint16m1_t` | 16 | 1 | `<vscale x 4 x i16>` |
| `vint32m1_t` | 32 | 1 | `<vscale x 4 x i32>` |
| `vint64m1_t` | 64 | 1 | `<vscale x 2 x i64>` |
| `vuint8m2_t` | 8 | 2 | `<vscale x 16 x i8>` |
| `vfloat32m4_t` | 32 | 4 | `<vscale x 16 x f32>` |
| `vfloat64m8_t` | 64 | 8 | `<vscale x 8 x f64>` |
| `vbool1_t` | – | m1 mask | `<vscale x 64 x i1>` |
| `vbool4_t` | – | m4 mask | `<vscale x 16 x i1>` |
| `vbool32_t` | – | 1/(32/SEW) mask | `<vscale x 2 x i1>` |

### 99.2.2 `vscale` Intrinsic

```llvm
; Query vscale at runtime in LLVM IR
%vs = call i64 @llvm.vscale.i64()
; For VLEN=512: vs = 512/(8*8) = 8
; <vscale x 4 x i32> has 4 * 8 = 32 elements

; LLVM constant expressions using vscale
; sizeof(<vscale x 4 x i32>) = vscale * 4 * 4 bytes
%size = mul i64 %vs, 16   ; 4 elements * 4 bytes * vscale
```

### 99.2.3 Scalable Types in Memory

Alloca for scalable-vector types uses the `llvm.stacksave`/`llvm.stackrestore` mechanism internally, and the frame size computation includes the `vscale` factor:

```llvm
; Dynamic alloca for scalable vector
%ptr = alloca <vscale x 4 x i32>  ; size = vscale * 16 bytes
; Frame lowering generates: sub sp, sp, (vscale * 16)
; via: csrr t0, vlenb; slli t0, t0, 1; sub sp, sp, t0  (for LMUL=1/4 ratio)
```

---

## 99.3 VLS Optimizer: Fixed-Length Optimization

### 99.3.1 When VLEN Is Known

When the target guarantees a specific VLEN, either via hardware documentation or the `-mrvv-vector-bits=N` flag, the VLS (Vector Length Specific) optimizer converts scalable-vector operations to fixed-length equivalents:

```bash
# Tell LLVM that VLEN is guaranteed to be 256 bits
clang --target=riscv64-linux-gnu \
    -march=rv64gcv -mabi=lp64d \
    -mrvv-vector-bits=256 \
    -O3 foo.c -o foo_vlen256
```

With VLEN=256 specified, `<vscale x 4 x i32>` (= 4 × vscale = 4 × 4 = 16 elements) becomes equivalent to `<16 x i32>`, enabling fixed-width DAG combining.

### 99.3.2 VLS Register Classes

LLVM defines VLS register classes for common fixed-length configurations:

```tablegen
// RISCVRegisterInfo.td — fixed-length vector register classes
def VRM1    : VecRC<[VR], "v", [i32, i64, f32, f64, ...], 128>;  // LMUL=1
def VRM2    : VecRC<[VR], "v", [...], 256>;   // LMUL=2
def VRM4    : VecRC<[VR], "v", [...], 512>;   // LMUL=4
def VRM8    : VecRC<[VR], "v", [...], 1024>;  // LMUL=8
```

The VLS optimizer in `RISCVTargetLowering.cpp` rewrites scalable operations to fixed-width ones when `RVVBitsMin == RVVBitsMax`, allowing DAG combining to apply fixed-width patterns.

---

## 99.4 Loop Vectorizer Integration

### 99.4.1 VP Intrinsics and EVL Tail Folding

The Loop Vectorizer uses RVV's native variable-length mechanism for efficient tail folding. Instead of generating a scalar epilogue, it uses `@llvm.vp.*` (vector predication) intrinsics with an explicit vector length:

```llvm
; VP add: only processes evl elements
%res = call <vscale x 4 x float> @llvm.vp.fadd.nxv4f32(
    <vscale x 4 x float> %a,
    <vscale x 4 x float> %b,
    <vscale x 4 x i1> %mask,   ; all-true for non-tail iterations
    i32 %evl)                   ; actual element count this iteration
```

The vectorizer generates a loop structure:

```llvm
loop.header:
  %remaining = phi i64 [ %n, %entry ], [ %rem_next, %loop.latch ]
  %ptr_a = phi float* [ %a, %entry ], [ %ptr_a_next, %loop.latch ]

  ; Set VL = min(remaining, VLMAX)
  %vl = call i64 @llvm.riscv.vsetvli(i64 %remaining, i64 2, i64 0)
                                    ; e32=2, m1=0
  ; Load with VP semantics
  %va = call <vscale x 4 x float> @llvm.riscv.vle32.nxv4f32(
      <vscale x 4 x float> undef, float* %ptr_a, i64 %vl)
  ; ... compute ...
  ; Store
  call void @llvm.riscv.vse32.nxv4f32(
      <vscale x 4 x float> %result, float* %ptr_c, i64 %vl)

  %rem_next = sub i64 %remaining, %vl
  %ptr_a_next = getelementptr float, float* %ptr_a, i64 %vl
  %done = icmp eq i64 %rem_next, 0
  br i1 %done, %exit, %loop.header
```

### 99.4.2 RISCVInsertVSETVLI Optimization

The `RISCVInsertVSETVLI` pass (Phase 1 and Phase 2) optimizes the placement of `VSETVLI` instructions in the generated machine code:

**Phase 1 (backward dataflow)**: Compute the demanded vtype information for each basic block — what SEW/LMUL configuration must be set on entry.

**Phase 2 (forward insertion)**: Insert `VSETVLI` only where the current vtype differs from what the next vector instruction needs.

**Phase 3 (cleanup)**: Remove any `VSETVLI` instructions that have become redundant.

```asm
; Without optimization (conceptual):
loop:
  vsetvli t0, a2, e32, m1, ta, ma  ; ← needed
  vle32.v v0, (a0)
  vsetvli t0, a2, e32, m1, ta, ma  ; ← REDUNDANT: same vtype
  vfadd.vv v0, v0, v1
  vsetvli t0, a2, e32, m1, ta, ma  ; ← REDUNDANT
  vse32.v v0, (a3)

; After RISCVInsertVSETVLI optimization:
loop:
  vsetvli t0, a2, e32, m1, ta, ma  ; ← only one needed
  vle32.v v0, (a0)
  vfadd.vv v0, v0, v1
  vse32.v v0, (a3)
```

### 99.4.3 Vectorization Flags for RVV

```bash
# Enable RVV auto-vectorization with tail folding
clang --target=riscv64-linux-gnu \
    -march=rv64gcv -mabi=lp64d \
    -O3 \
    -fno-vectorize-loops=false \
    -mllvm --riscv-v-vector-bits-min=128 \
    foo.c -o foo

# Inspect vectorized IR
/usr/lib/llvm-22/bin/opt \
    --passes="loop-vectorize,instcombine,simplifycfg" \
    -S foo.ll | grep "vscale"

# Verify machine-level vector instructions
clang --target=riscv64-linux-gnu \
    -march=rv64gcv -mabi=lp64d \
    -O3 -S foo.c -o foo.s
grep -E "vsetvl|vle|vse|vfadd" foo.s
```

---

## 99.5 RVV Intrinsic Families

The C intrinsic API is defined in the `riscv-v-intrinsic-doc` specification maintained at [`https://github.com/riscv-non-isa/rvv-intrinsic-doc`](https://github.com/riscv-non-isa/rvv-intrinsic-doc). LLVM implements these in `clang/include/clang/Basic/riscv_vector.td` and the generated header `riscv_vector.h`.

### 99.5.1 Load/Store Families

```c
#include <riscv_vector.h>

// ===== Unit-stride =====
// vle32_v_f32m1: unit-stride load, SEW=32, LMUL=1
// Assembly: vle32.v vd, (rs1), vm
vfloat32m1_t __riscv_vle32_v_f32m1(const float* base, size_t vl);

// vse32_v_f32m1: unit-stride store
void __riscv_vse32_v_f32m1(float* base, vfloat32m1_t value, size_t vl);

// ===== Strided =====
// vlse32_v_f32m1: strided load (stride in bytes)
// Assembly: vlse32.v vd, (rs1), rs2, vm
vfloat32m1_t __riscv_vlse32_v_f32m1(const float* base,
                                      ptrdiff_t bstride, size_t vl);

// vsse32_v_f32m1: strided store
void __riscv_vsse32_v_f32m1(float* base, ptrdiff_t bstride,
                              vfloat32m1_t value, size_t vl);

// ===== Indexed (gather/scatter) =====
// vluxei32_v_f32m1: indexed unordered load (gather)
// result[i] = *(base + byte_offset[i])
// Assembly: vluxei32.v vd, (rs1), vs2, vm
vfloat32m1_t __riscv_vluxei32_v_f32m1(const float* base,
                                        vuint32m1_t bindex, size_t vl);

// vsuxei32_v_f32m1: indexed unordered store (scatter)
void __riscv_vsuxei32_v_f32m1(float* base, vuint32m1_t bindex,
                                vfloat32m1_t value, size_t vl);

// vloxei32_v_f32m1: indexed ordered load (same address semantics, ordered)
vfloat32m1_t __riscv_vloxei32_v_f32m1(const float* base,
                                        vuint32m1_t bindex, size_t vl);
```

### 99.5.2 Arithmetic Operations

```c
// ===== Integer arithmetic =====
// vadd_vv: vector-vector add
vint32m1_t __riscv_vadd_vv_i32m1(vint32m1_t op1, vint32m1_t op2, size_t vl);

// vadd_vx: vector-scalar add
vint32m1_t __riscv_vadd_vx_i32m1(vint32m1_t op1, int32_t op2, size_t vl);

// vadd_vi: vector-immediate add (-16..15 immediate)
vint32m1_t __riscv_vadd_vi_i32m1(vint32m1_t op1, int8_t imm, size_t vl);

// vmul: vector multiply
vint32m1_t __riscv_vmul_vv_i32m1(vint32m1_t op1, vint32m1_t op2, size_t vl);

// vmacc: multiply-accumulate: vd = vd + vs1 * vs2
vint32m1_t __riscv_vmacc_vv_i32m1(vint32m1_t acc,
                                    vint32m1_t vs1, vint32m1_t vs2,
                                    size_t vl);

// ===== FP arithmetic =====
// vfadd: float add
vfloat32m1_t __riscv_vfadd_vv_f32m1(vfloat32m1_t op1, vfloat32m1_t op2,
                                      size_t vl);

// vfmacc: fused multiply-accumulate: vd = vd + vs1 * vs2
vfloat32m1_t __riscv_vfmacc_vv_f32m1(vfloat32m1_t acc,
                                       vfloat32m1_t vs1, vfloat32m1_t vs2,
                                       size_t vl);

// vfmacc scalar: vd = vd + rs1 * vs2 (broadcast scalar rs1)
vfloat32m1_t __riscv_vfmacc_vf_f32m1(vfloat32m1_t acc, float rs1,
                                       vfloat32m1_t vs2, size_t vl);

// vfnmsac: fused neg-multiply-subtract: vd = -vd + vs1 * vs2
vfloat32m1_t __riscv_vfnmsac_vv_f32m1(vfloat32m1_t acc,
                                        vfloat32m1_t vs1, vfloat32m1_t vs2,
                                        size_t vl);
```

### 99.5.3 Permute and Data-Reorganization

```c
// ===== Slides =====
// vslideup: shift vector elements up by offset positions
vint32m1_t __riscv_vslideup_vx_i32m1(vint32m1_t dst, vint32m1_t src,
                                       size_t offset, size_t vl);

// vslidedown: shift vector elements down
vint32m1_t __riscv_vslidedown_vx_i32m1(vint32m1_t dst, vint32m1_t src,
                                         size_t offset, size_t vl);

// vslide1up: shift up by 1, insert scalar at position 0
vint32m1_t __riscv_vslide1up_vx_i32m1(vint32m1_t src, int32_t val,
                                        size_t vl);

// ===== Gather =====
// vrgather: vd[i] = vs2[vs1[i]]  (within-vector gather)
vint32m1_t __riscv_vrgather_vv_i32m1(vint32m1_t src, vuint32m1_t index,
                                       size_t vl);

// vrgatherei16: gather with 16-bit indices (useful when indices < VLMAX)
vint32m1_t __riscv_vrgatherei16_vv_i32m1(vint32m1_t src, vuint16mf2_t index,
                                           size_t vl);

// ===== Compress =====
// vcompress: compact active (unmasked) elements to front
vint32m1_t __riscv_vcompress_vm_i32m1(vbool32_t mask, vint32m1_t dst,
                                        vint32m1_t src, size_t vl);

// ===== Reduction =====
// vredsum: reduce-add: result[0] = sum of vs2[0..vl-1] + vs1[0]
vint32m1_t __riscv_vredsum_vs_i32m1(vint32m1_t vs1, vint32m1_t vs2,
                                      size_t vl);
// vfredosum: ordered floating-point reduction (left-to-right)
vfloat32m1_t __riscv_vfredosum_vs_f32m1(vfloat32m1_t vs1, vfloat32m1_t vs2,
                                          size_t vl);
```

### 99.5.4 Mask Operations

```c
// ===== Mask logical =====
vbool32_t __riscv_vmand_mm_b32(vbool32_t op1, vbool32_t op2, size_t vl);
vbool32_t __riscv_vmor_mm_b32(vbool32_t op1, vbool32_t op2, size_t vl);
vbool32_t __riscv_vmxor_mm_b32(vbool32_t op1, vbool32_t op2, size_t vl);
vbool32_t __riscv_vmnot_m_b32(vbool32_t op, size_t vl);

// ===== Mask from comparison =====
// vmseq: mask = (vs1[i] == vs2[i])
vbool32_t __riscv_vmseq_vv_i32m1_b32(vint32m1_t op1, vint32m1_t op2,
                                       size_t vl);
vbool32_t __riscv_vmslt_vv_i32m1_b32(vint32m1_t op1, vint32m1_t op2,
                                       size_t vl);

// ===== Mask population / find =====
unsigned long __riscv_vcpop_m_b32(vbool32_t op, size_t vl);
long __riscv_vfirst_m_b32(vbool32_t op, size_t vl);  // first set bit index

// ===== Iota =====
// viota: result[i] = popcount(mask[0..i-1]) if mask[i]=1, else 0
vuint32m1_t __riscv_viota_m_u32m1(vbool32_t mask, size_t vl);

// ===== Index generation =====
// vid: result[i] = i (generate indices 0, 1, 2, ...)
vuint32m1_t __riscv_vid_v_u32m1(size_t vl);
```

---

## 99.6 LLVM Intrinsic Lowering for RVV

### 99.6.1 IntrinsicsRISCV.td

LLVM's RVV intrinsics are defined in `llvm/include/llvm/IR/IntrinsicsRISCV.td`:

```tablegen
// vle32.v (unit-stride load, no passthru)
def int_riscv_vle : DefaultAttrsIntrinsic<
    [llvm_anyvector_ty],                 // result: vector type
    [LLVMPointerType<LLVMMatchType<0>>,  // base pointer
     llvm_anyint_ty],                    // vl
    [IntrReadMem, IntrArgMemOnly,
     IntrWillReturn, NoCapture<ArgIndex<0>>]>;

// vle32.v with mask (masking intrinsic variant)
def int_riscv_vle_mask : DefaultAttrsIntrinsic<
    [llvm_anyvector_ty],
    [LLVMMatchType<0>,                   // passthru (merge value)
     LLVMPointerType<LLVMMatchType<0>>,  // base pointer
     LLVMScalarOrSameVectorWidth<0, llvm_i1_ty>,  // mask
     llvm_anyint_ty,                     // vl
     llvm_i64_ty],                       // policy (0=tu/mu, 3=ta/ma)
    [IntrReadMem, IntrArgMemOnly, IntrWillReturn]>;

// vfadd.vv
def int_riscv_vfadd : DefaultAttrsIntrinsic<
    [llvm_anyvector_ty],
    [LLVMMatchType<0>,  // vs1
     LLVMMatchType<0>,  // vs2
     llvm_anyint_ty],   // vl
    [IntrNoMem, IntrSpeculatable, IntrWillReturn]>;
```

### 99.6.2 Lowering Chain

```
Clang __riscv_vle32_v_f32m1()
  → LLVM IR call @llvm.riscv.vle.nxv4f32.p0f32(...)
    → RISCVISelLowering::LowerINTRINSIC_W_CHAIN()
      → RISCVISD::VLE node
        → RISCVInstrInfoV.td pattern: VLE32_V
          → MC: vle32.v v0, (a0)
```

The `RISCVISelLowering.cpp` file handles over 400 distinct intrinsic lowering cases for RVV, organized by instruction family.

---

## 99.7 Masking and Predication

### 99.7.1 Mask Register Architecture

Mask registers are stored in ordinary vector registers. Register `v0` is the implicit mask argument for most vector instructions. Each mask bit corresponds to one element at the current SEW. With SEW=32 and LMUL=1 at VLEN=256:
- 8 elements per vector register
- 8 bits of `v0` hold the 8 mask bits
- Bits 8–63 of v0 are inactive (ignored)

For LMUL=4, there would be 32 elements and 32 mask bits.

### 99.7.2 Masking Policy Combinations

RVV 1.0 defines 4 policy combinations via the intrinsic naming convention:

| Suffix | tail | mask | Passthru needed? |
|--------|------|------|-----------------|
| (none, default ta/ma) | agnostic | agnostic | No |
| `_tumu` | undisturbed | undisturbed | Yes |
| `_tamu` | agnostic | undisturbed | Yes |
| `_tuma` | undisturbed | agnostic | Yes |

```c
// Masked add with agnostic policy (most common in ML code)
vint32m1_t __riscv_vadd_vv_i32m1_m(vbool32_t mask,
                                     vint32m1_t op1,
                                     vint32m1_t op2,
                                     size_t vl);

// Masked add with tu/mu (merge into passthru)
vint32m1_t __riscv_vadd_vv_i32m1_tumu(vbool32_t mask,
                                        vint32m1_t maskedoff,
                                        vint32m1_t op1,
                                        vint32m1_t op2,
                                        size_t vl);
```

### 99.7.3 Mask Encoding in Instructions

In the assembly, masking is expressed with `.v` suffix and optional `vm` field:

```asm
; Unmasked (all elements active)
vadd.vv   v0, v1, v2         ; no mask

; Masked using v0.t (active where v0 bit=1)
vadd.vv   v3, v1, v2, v0.t  ; v0 is the implicit mask register

; Masked complement (active where v0 bit=0) — not directly in ISA,
; use vmnot first then mask
```

---

## 99.8 Tuple Types and Segmented Memory

### 99.8.1 Segmented Load/Store Architecture

RVV's segmented load/store instructions (VLSEG/VSSEG) load/store multiple "fields" interleaved in memory, as in structure-of-arrays data layouts:

```
Memory layout (RGB pixel data):
R[0] G[0] B[0] R[1] G[1] B[1] R[2] G[2] B[2] ...
```

`VLSEG3E32.V` loads this as three separate vector registers: all R values into v0, all G values into v1, all B values into v2. This is the SoA ↔ AoS transformation in hardware.

### 99.8.2 Tuple Types in LLVM

LLVM represents segmented load results as RVV tuple types:

```c
// Load 3-channel f32 data (RGB pixels)
// Returns a "tuple" of 3 LMUL=1 float vectors
vfloat32m1x3_t __riscv_vlseg3e32_v_f32m1x3(const float* base, size_t vl);

// Extract individual channel from tuple
vfloat32m1_t r = __riscv_vget_v_f32m1x3_f32m1(rgb_data, 0);
vfloat32m1_t g = __riscv_vget_v_f32m1x3_f32m1(rgb_data, 1);
vfloat32m1_t b = __riscv_vget_v_f32m1x3_f32m1(rgb_data, 2);

// Build tuple from individual vectors
vfloat32m1x3_t result = __riscv_vset_v_f32m1_f32m1x3(empty, 0, r_out);
result = __riscv_vset_v_f32m1_f32m1x3(result, 1, g_out);
result = __riscv_vset_v_f32m1_f32m1x3(result, 2, b_out);

// Store back as interleaved
__riscv_vsseg3e32_v_f32m1x3(float* out_base, result, vl);
```

LLVM enforces that each "segment" uses consecutive register numbers (v0, v1, v2 for the example above), and the register allocator allocates the tuple as an aligned group.

---

## 99.9 Complete Vectorized Loop Examples

### 99.9.1 SAXPY (Single-Precision A·X + Y)

```c
#include <riscv_vector.h>
#include <stddef.h>

// Vectorized SAXPY: y[i] = a * x[i] + y[i]
void saxpy(size_t n, float a, const float* restrict x, float* restrict y) {
    for (size_t vl; n > 0; n -= vl, x += vl, y += vl) {
        vl = __riscv_vsetvl_e32m4(n);     // LMUL=4 for 4× throughput
        vfloat32m4_t vx = __riscv_vle32_v_f32m4(x, vl);
        vfloat32m4_t vy = __riscv_vle32_v_f32m4(y, vl);
        // fused multiply-add: vy = a * vx + vy
        vy = __riscv_vfmacc_vf_f32m4(vy, a, vx, vl);
        __riscv_vse32_v_f32m4(y, vy, vl);
    }
}
```

### 99.9.2 Int8 Matrix-Vector Product

```c
// Matrix-vector product: y[i] = sum_j(A[i][j] * x[j])
// Uses LMUL=8 for maximum register utilization
void matvec_int8(int32_t* y, const int8_t* A,
                 const int8_t* x, int M, int N) {
    for (int i = 0; i < M; i++) {
        vint32m1_t vacc = __riscv_vmv_v_x_i32m1(0, 1);  // scalar 0
        const int8_t* row = A + i * N;
        for (size_t j = 0, vl; j < (size_t)N; j += vl) {
            vl = __riscv_vsetvl_e8m2((size_t)N - j);
            vint8m2_t vrow = __riscv_vle8_v_i8m2(row + j, vl);
            vint8m2_t vx   = __riscv_vle8_v_i8m2(x + j, vl);
            // Widening multiply: result is 16-bit
            vint16m4_t vprod = __riscv_vwmul_vv_i16m4(vrow, vx, vl);
            // Widening sum: accumulate into 32-bit
            vint32m8_t vprod32 = __riscv_vwadd_wv_i32m8(
                __riscv_vmv_v_x_i32m8(0, vl), vprod, vl);
            // Reduction into single value
            vint32m1_t vsum = __riscv_vredsum_vs_i32m8_i32m1(vacc, vprod32, vl);
            vacc = vsum;
        }
        y[i] = __riscv_vmv_x_s_i32m1_i32(vacc);
    }
}
```

### 99.9.3 Compilation and Verification

```bash
# Compile SAXPY with RVV LMUL=4
clang --target=riscv64-linux-gnu \
    -march=rv64gcv \
    -mabi=lp64d \
    -O2 -S saxpy.c -o saxpy.s

# Inspect the generated vector loop
grep -A 20 "vsetvl" saxpy.s | head -30

# Auto-vectorization check
clang --target=riscv64-linux-gnu \
    -march=rv64gcv -mabi=lp64d \
    -O3 -fopt-info-vec -S \
    auto_loop.c 2>&1 | grep "vectorized"
```

---

## Chapter Summary

- RVV's defining innovation is hardware-defined VLEN: software sets SEW (element width) and LMUL (register grouping) via `VSETVLI`, and the hardware provides as many elements as it can; programs auto-scale from 128-bit to 2048-bit implementations.
- LLVM represents RVV types as `<vscale x N x T>` scalable vectors where `vscale = VLENB/8`; a `<vscale x 4 x i32>` vector holds `4 × vscale` float32 elements, exactly one LMUL=1 RVV register.
- LMUL fractions (mf2/mf4/mf8) reduce elements per operation; LMUL multiples (m2/m4/m8) group 2/4/8 registers as one logical register, increasing throughput for wide operations.
- The VLS optimizer converts scalable vectors to fixed-length equivalents when `-mrvv-vector-bits=N` guarantees the VLEN, enabling additional fixed-width DAG combining.
- The Loop Vectorizer uses `@llvm.vp.*` intrinsics for EVL-based tail folding, and `RISCVInsertVSETVLI` performs multi-phase dataflow analysis to minimize redundant `VSETVLI` instructions between consecutive vector ops sharing the same vtype.
- The C intrinsic API from `riscv_vector.h` exposes unit-stride vle/vse, strided vlse/vsse, indexed vluxei/vsuxei, arithmetic (vadd/vmul/vfmacc), permute (vrgather/vslide/vcompress), mask (vmand/vmseq/vcpop), and reduction (vredsum/vfredosum) families.
- Masking policy (ta/ma vs tu/mu) controls tail and masked-off element treatment; `_tumu`/`_tama`/`_tuma` suffixed intrinsics expose merge semantics with explicit passthru operands.
- Segmented loads/stores (VLSEG/VSSEG) use RVV tuple types (`vfloat32m1x3_t`) to load/store interleaved N-field structures in a single operation, implementing the AoS↔SoA transformation in hardware with `__riscv_vlseg3e32_v_f32m1x3`.


---

@copyright jreuben11
