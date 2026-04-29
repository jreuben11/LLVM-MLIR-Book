# Chapter 104 — The SPIR-V Backend

*Part XV — Targets*

SPIR-V occupies a unique position in the GPU programming landscape: it is the portable binary intermediate language mandated by both the Vulkan and OpenCL 2.x specifications, and it is the consumption format of every Vulkan driver, every modern OpenCL implementation, and most SYCL runtimes. Unlike PTX or GCN ISA, SPIR-V is logical — it carries no register allocation, no instruction scheduling, and no hardware-specific encoding. Its type system is explicit and hierarchical, its module structure is canonically ordered, and its capability system tracks which language features are used. LLVM's in-tree SPIR-V backend (`llvm/lib/Target/SPIRV/`), introduced in LLVM 15 and substantially matured through LLVM 22, translates LLVM IR into SPIR-V binary using GlobalISel — a deliberate architectural choice that avoids the structural impedance mismatch that SelectionDAG would impose on a logical-addressing target. This chapter explains the backend's design, the GlobalISel pipeline it uses, the SPIR-V type system reconstruction problem, capability tracking, and binary emission.

---

## 104.1 SPIR-V Overview

### 104.1.1 SPIR-V as a Binary Intermediate Language

SPIR-V (Standard Portable Intermediate Representation V) is specified by the Khronos Group. It defines a binary module format consisting of 32-bit words encoding typed instructions with explicit SSA IDs. There are no registers, no physical addresses, no calling conventions in the hardware sense — SPIR-V specifies computation at the abstract graph level, leaving all physical implementation to driver compilers (e.g., AMD's RDNA compiler stack, Intel's IGC, NVIDIA's nvk).

Key properties that distinguish SPIR-V from PTX or GCN ISA:

| Property | SPIR-V | PTX | GCN ISA |
|----------|--------|-----|---------|
| Addressing | Logical (Vulkan) / Physical (OpenCL) | Physical 64-bit | Physical 64-bit |
| Register allocation | None — SSA IDs only | Virtual registers | Hardware registers |
| Type system | Explicit OpType instructions | Implicit (embedded in opcode) | None |
| Module ordering | Canonical (capabilities → types → globals → functions) | None enforced | None enforced |
| Capability declarations | Mandatory | N/A | N/A |
| Extension mechanism | `OpExtension`, `OpExtInstImport` | Inline PTX extensions | Feature bits |

### 104.1.2 SPIR-V Versions and Profiles

| SPIR-V Version | Key additions |
|---------------|---------------|
| 1.0 (2015) | Vulkan 1.0, OpenCL 2.1 |
| 1.1 (2016) | SubgroupBallot, Subgroups |
| 1.3 (2018) | Vulkan 1.1, SubgroupBroadcast |
| 1.4 (2019) | Vulkan 1.2, EntryPoint globals list |
| 1.5 (2020) | VulkanMemoryModel, SPIR-V/OpenCL alignment |
| 1.6 (2021) | Vulkan 1.3, removal of old features |

LLVM 22's SPIR-V backend emits SPIR-V 1.4 by default for the OpenCL profile:

```
Magic:     0x07230203
Version:   0x00010400  (SPIR-V 1.4)
Generator: 0x002B0016  (LLVM SPIR-V backend)
Bound:     19          (max SSA ID + 1)
Schema:    0
```

### 104.1.3 Execution Models

SPIR-V's `OpEntryPoint` instruction names the execution model alongside the entry function:

| Execution model | Used for |
|----------------|---------|
| `Vertex` | Vertex shaders (Vulkan) |
| `Fragment` | Fragment/pixel shaders (Vulkan) |
| `GLCompute` | Compute shaders (Vulkan GLSL) |
| `Kernel` | OpenCL kernels (OpenCL profile) |
| `RayGenerationKHR` | Ray tracing (Vulkan KHR_ray_tracing) |
| `MeshEXT` | Mesh shaders (Vulkan EXT_mesh_shader) |

The LLVM SPIR-V backend emits `Kernel` for the `spirv64-unknown-unknown` triple and `GLCompute`/`Vertex`/etc. for the `spirv-unknown-vulkan*` family.

---

## 104.2 The LLVM SPIR-V Backend History

### 104.2.1 Pre-existing Path: Khronos Translator

Before LLVM 15, the dominant tool for generating SPIR-V from LLVM IR was the [Khronos `llvm-spirv` translator](https://github.com/KhronosGroup/SPIRV-LLVM-Translator) — an out-of-tree project. It translates LLVM IR to SPIR-V using high-level pattern matching: it recognizes OpenCL built-in function call signatures, LLVM metadata conventions, and C++ mangling patterns. This translator is used by Intel oneAPI's DPC++, AMD's ROCm OpenCL path, and several SYCL toolchains.

### 104.2.2 The In-Tree SPIRV Backend

LLVM 15 added `llvm/lib/Target/SPIRV/` — an in-tree backend designed to translate arbitrary LLVM IR to SPIR-V without requiring OpenCL-specific function naming conventions or metadata. The design goal is structural correctness: the backend should produce valid SPIR-V for any well-formed LLVM IR module targeting a SPIR-V triple.

Target triples supported:
- `spirv32-unknown-unknown` — SPIR-V 32-bit (Physical32 addressing, for 32-bit OpenCL)
- `spirv64-unknown-unknown` — SPIR-V 64-bit (Physical64 addressing, for 64-bit OpenCL)
- `spirv-unknown-vulkan1.3` — SPIR-V for Vulkan 1.3 (Logical addressing, GLSL450 memory model)

### 104.2.3 Design Philosophy

The in-tree backend deliberately uses GlobalISel instead of SelectionDAG. The rationale: SelectionDAG operates on DAGs of target-typed nodes, requires a register allocator, and assumes physical addressing — none of which fits SPIR-V's logical SSA model. GlobalISel's `G_*` generic instructions map more naturally to SPIR-V opcodes, and the GlobalISel legalizer can handle SPIR-V's strict type constraints without fighting SelectionDAG's assumptions.

---

## 104.2b SPIR-V Module Structure Reference

### 104.2b.1 A Complete Annotated Module

A complete, minimal, valid SPIR-V module (OpenCL profile, 32-bit) for the `simple(ptr out, i32 n)` kernel — matching the actual LLVM-generated binary decoded earlier:

```
; ─── Module Header ────────────────────────────────────────────────
Magic:     0x07230203
Version:   0x00010400   (SPIR-V 1.4)
Generator: 0x002B0016   (LLVM SPIRV backend, version 22)
Bound:     19           (IDs 1..18 are used)
Schema:    0

; ─── Section 1: Capabilities ──────────────────────────────────────
OpCapability Kernel         ; OpenCL execution model
OpCapability Addresses      ; Physical32/64 addressing
; (no Int64 needed: no 64-bit values in this kernel)

; ─── Section 3: Extended Instruction Sets ─────────────────────────
%1 = OpExtInstImport "OpenCL.std"    ; built-in math, sync, etc.

; ─── Section 4: Memory Model ──────────────────────────────────────
OpMemoryModel Physical32 OpenCL      ; 32-bit addresses, OpenCL semantics

; ─── Section 5: Entry Points ──────────────────────────────────────
OpEntryPoint Kernel %8 "simple"      ; execution model, fn ID, name

; ─── Section 6: Execution Modes ───────────────────────────────────
OpExecutionMode %8 ContractionOff    ; disable FMA contraction for OpenCL conformance

; ─── Section 7: Debug Names ───────────────────────────────────────
OpName %6 "out"
OpName %7 "n"
OpName %8 "simple"

; ─── Section 9: Types ─────────────────────────────────────────────
%2 = OpTypeInt 32 0                  ; i32 (unsigned)
%3 = OpTypePointer CrossWorkgroup %2 ; i32 addrspace(1)*  → global int*
%4 = OpTypeVoid
%5 = OpTypeFunction %4 %3 %2         ; (ptr, i32) → void

; ─── Section 12: Functions ────────────────────────────────────────
%8 = OpFunction %4 None %5
%6 = OpFunctionParameter %3          ; out: global int*
%7 = OpFunctionParameter %2          ; n: i32
%9 = OpLabel                         ; entry basic block
OpStore %6 %7 Aligned 4             ; *out = n
OpReturn
OpFunctionEnd
```

This is the canonical form: every type appears exactly once, function parameters follow `OpFunction` before the first `OpLabel`, and the type sections appear before any function definition.

---

## 104.3 Target Machine and Configuration

### 104.3.1 SPIRVTargetMachine

[`SPIRVTargetMachine.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/SPIRV/SPIRVTargetMachine.cpp) defines the SPIR-V target machine. Unlike GPU backends that configure register allocators and instruction schedulers, `SPIRVTargetMachine` disables most physical-machine passes:

```cpp
class SPIRVTargetMachine : public LLVMTargetMachine {
public:
  SPIRVTargetMachine(const Target &T, const Triple &TT,
                     StringRef CPU, StringRef FS,
                     const TargetOptions &Options,
                     std::optional<Reloc::Model> RM,
                     std::optional<CodeModel::Model> CM,
                     CodeGenOptLevel OL, bool JIT);

  const SPIRVSubtarget *getSubtargetImpl(const Function &F) const override;
  TargetPassConfig *createPassConfig(PassManagerBase &PM) override;
};
```

The pass pipeline from `createPassConfig` (verified from `-debug-pass=Structure` output):

```
Pre-ISel Intrinsic Lowering
SPIRV prepare functions
SPIRV prepare global variables
  SPIRV strip convergent intrinsics
SPIRV Legalize Implicit Binding
SPIRV Legalize Zero-Size Arrays
SPIRV CBuffer Access          [Vulkan path: constant buffer access]
SPIRV push constant Access    [Vulkan path]
SPIRV emit intrinsics
  SPIRVPreLegalizerCombiner
  SPIRV pre legalizer
  Legalizer
  SPIRV post legalizer
  Finalize ISel and expand pseudo-instructions
SPIRV module analysis
  SPIRV Assembly Printer
```

### 104.3.2 SPIRVSubtarget

`SPIRVSubtarget` tracks the target environment (OpenCL vs Vulkan), the SPIR-V version to emit, and available capabilities:

```cpp
class SPIRVSubtarget : public SPIRVGenSubtargetInfo {
  SPIRVTargetEnv TargetEnv;  // OpenCL, Vulkan, etc.
  unsigned SPIRVVersion;     // e.g., 0x00010400 for 1.4
  bool isVulkanEnv() const;
  bool isOpenCLEnv() const;
  unsigned getMemoryModel() const; // GLSL450 or OpenCL
};
```

### 104.3.3 Address Spaces in SPIR-V

SPIR-V uses storage classes (analogous to address spaces) as part of its pointer type system. LLVM `addrspace(N)` values map to SPIR-V storage classes differently between OpenCL and Vulkan profiles:

**OpenCL profile** (`spirv64-unknown-unknown`):

| `addrspace(N)` | SPIR-V Storage Class | Description |
|---------------|---------------------|-------------|
| 0 | `Function` | Function-local variables |
| 1 | `CrossWorkgroup` | Global memory |
| 2 | `UniformConstant` | Constant memory / images / samplers |
| 3 | `Workgroup` | Local (shared) memory |
| 4 | `Input` | Input builtins (readonly) |

**Vulkan profile** (`spirv-unknown-vulkan*`):

| `addrspace(N)` | SPIR-V Storage Class | Description |
|---------------|---------------------|-------------|
| 0 | `Function` | Function-local variables |
| 1 | `StorageBuffer` | Shader storage buffer (SSBO) |
| 2 | `Uniform` | Uniform buffer (UBO) |
| 3 | `Workgroup` | Shared memory |
| 4 | `Input` | Stage input |
| 5 | `Output` | Stage output |
| 6 | `Private` | Thread-private |

---

## 104.4 GlobalISel-Based Instruction Selection

### 104.4.1 The SPIR-V IRTranslator

GlobalISel's first phase, `IRTranslator`, translates LLVM IR instructions to generic `G_*` machine instructions. For SPIR-V, the custom `SPIRVIRTranslator` ([`SPIRVIRTranslator.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/SPIRV/SPIRVIRTranslator.cpp)) extends this with SPIR-V-specific handling:

- `alloca` → `G_ALLOCA` with `Function` storage class tracking
- `load`/`store` from specific address spaces → tagged with storage class
- `call @__spirv_*` → SPIR-V extended instruction set calls
- `addrspacecast` → `OpPtrCastToGeneric` / `OpGenericCastToPtr`

### 104.4.2 SPIRVLegalizer

The legalizer enforces SPIR-V's type constraints. SPIR-V has stricter rules than LLVM IR:

- Integer types must be `OpTypeInt 8/16/32/64` — arbitrary-width integers are illegal
- Pointer types must carry a storage class — opaque `ptr` without `addrspace` is illegal
- Vector types must use supported element counts: 2, 3, 4, 8, 16

The legalizer expands illegal types (e.g., `i1` vectors) and inserts conversions where LLVM IR is more permissive than SPIR-V.

### 104.4.3 SPIRVInstructionSelector

[`SPIRVInstructionSelector.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/SPIRV/SPIRVInstructionSelector.cpp) maps legalized `G_*` opcodes to concrete SPIR-V opcodes:

| LLVM `G_*` opcode | SPIR-V opcode |
|-------------------|--------------|
| `G_ADD` (integer) | `OpIAdd` |
| `G_FADD` | `OpFAdd` |
| `G_FMUL` | `OpFMul` |
| `G_LOAD` | `OpLoad` |
| `G_STORE` | `OpStore` |
| `G_GEP` / `G_PTR_ADD` | `OpPtrAccessChain` |
| `G_ICMP` | `OpIEqual`, `OpSLessThan`, etc. |
| `G_BR` / `G_BRCOND` | `OpBranch`, `OpBranchConditional` |
| `G_PHI` | `OpPhi` |
| `G_SELECT` | `OpSelect` |
| `G_INTRINSIC` | SPIR-V extended instructions |
| `G_INSERT_VECTOR_ELT` | `OpCompositeInsert` |
| `G_EXTRACT_VECTOR_ELT` | `OpCompositeExtract` |

### 104.4.3b MIR Dump After IRTranslator

The `--stop-after=irtranslator` stage reveals how LLVM IR translates to generic machine instructions before legalization. For the simple `add_one` kernel:

```
; Input LLVM IR:
define spir_kernel void @add_one(ptr addrspace(1) %out, ptr addrspace(1) %in) {
  %gid = call i64 @_Z13get_global_idj(i32 0)
  %in_ptr = getelementptr i32, ptr addrspace(1) %in, i64 %gid
  %v = load i32, ptr addrspace(1) %in_ptr
  %r = add i32 %v, 1
  %out_ptr = getelementptr i32, ptr addrspace(1) %out, i64 %gid
  store i32 %r, ptr addrspace(1) %out_ptr
  ret void
}
```

After IRTranslator (key instructions from `--stop-after=irtranslator` MIR output):

```
; Type declarations (emitted as pseudo-instructions before the function body)
%type_i32  : type(s64) = OpTypeInt 32, 0
%type_ptr  : type(s64) = OpTypePointer 5, %type_i32   ; 5 = CrossWorkgroup
%type_void : type(s64) = OpTypeVoid
%type_fn   : type(s64) = OpTypeFunction %type_void, %type_ptr, %type_ptr

; In the basic block:
%8  : iid(s32) = G_CONSTANT i32 0         ; arg to get_global_id
%16 : _(s32) = G_CONSTANT i32 1           ; the "+1"

; The @_Z13get_global_idj call → load from __spirv_BuiltInGlobalInvocationId:
%11 : pid(p0) = OpVariable %gid_type, 1   ; BuiltIn variable (Input storage class)
%13 : vid(<3 x s64>) = G_LOAD %11 :: (load <3 x i64>, align 1)
%7  : iid(s64) = G_INTRINSIC @llvm.spv.extractelt, %13, %8  ; extract x component

; GEP → llvm.spv.gep:
%14 : _(p1) = G_INTRINSIC @llvm.spv.gep, 0, %1, %7   ; in + gid

; Load, add, GEP, store:
%15 : _(s32) = G_LOAD %14 :: (load s32 from %in_ptr, addrspace 1)
%17 : _(s32) = G_ADD %15, %16
%18 : _(p1) = G_INTRINSIC @llvm.spv.gep, 0, %0, %7   ; out + gid
G_STORE %17, %18 :: (store s32 into %out_ptr, addrspace 1)
OpReturn
```

The `llvm.spv.assign.type` and `llvm.spv.assign.ptr.type` intrinsics are injected by the pre-ISel `SPIRVEmitIntrinsics` pass to carry type information that survives to the legalizer — this is the SPIR-V backend's solution to opaque pointer type reconstruction.

### 104.4.4 SPIRVGlobalRegistry

The `SPIRVGlobalRegistry` ([`SPIRVGlobalRegistry.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/SPIRV/SPIRVGlobalRegistry.h)) is a module-scoped registry that assigns SPIR-V result IDs to types, constants, and global variables, ensuring each unique type or constant receives exactly one ID across the entire module:

```cpp
class SPIRVGlobalRegistry {
  // Map from LLVM type → SPIR-V type ID
  DenseMap<const Type *, Register> LLVMTypeToSPIRV;
  // Map from constant value → SPIR-V constant ID
  DenseMap<const Constant *, Register> LLVMConstToSPIRV;
  // Required capabilities (populated as instructions are selected)
  SmallSet<Capability::Capability, 8> RequiredCapabilities;
  // Required extensions
  SmallSet<Extension::Extension, 4> RequiredExtensions;

  Register getOrCreateIntType(unsigned BitWidth, bool IsSigned);
  Register getOrCreateFloatType(unsigned BitWidth);
  Register getOrCreatePointerType(Register ElemType, StorageClass::StorageClass SC);
  Register getOrCreateVectorType(Register ElemType, unsigned NumElems);
};
```

This registry is the solution to the fundamental tension between LLVM IR's type-erase-at-pointer-level model and SPIR-V's requirement that every pointer carry its pointee type and storage class.

### 104.4.5 Type Annotation Intrinsics

Because LLVM opaque pointers carry no pointee type information, the SPIR-V backend injects special intrinsics during the pre-ISel `SPIRVEmitIntrinsics` pass to preserve type context through the lowering pipeline:

```llvm
; These intrinsics are injected by SPIRVEmitIntrinsics — they do not appear
; in the user's LLVM IR but are added during codegen preparation:

; Annotate an integer value's SPIR-V type:
declare void @llvm.spv.assign.type.i32(i32 %val, metadata %ty_md)
declare void @llvm.spv.assign.type.i64(i64 %val, metadata %ty_md)

; Annotate a pointer value's pointee type and storage class:
declare void @llvm.spv.assign.ptr.type.p1(ptr addrspace(1) %ptr,
                                           metadata %elem_ty_md,
                                           i32 immarg %storage_class)

; SPIR-V-aware GEP (replaces regular getelementptr in SPIR-V path):
declare ptr addrspace(1) @llvm.spv.gep.p1.p1(i1 immarg %in_bounds,
                                               ptr addrspace(1) %base,
                                               ...) ; variadic indices
```

After `SPIRVEmitIntrinsics`, every `ptr addrspace(1)` in the function has at least one `@llvm.spv.assign.ptr.type.p1` annotation that the legalizer uses to determine the SPIR-V `OpTypePointer` storage class and element type. This approach threads type information through the GlobalISel pipeline without modifying LLVM's core type system.

---

## 104.5 The SPIR-V Type System

### 104.5.1 Explicit Type Instructions

SPIR-V encodes its entire type system as instructions that appear before function definitions. Each type instruction produces a result ID that subsequent instructions reference:

```
; Complete SPIR-V type hierarchy for a simple kernel:
%void     = OpTypeVoid
%uint     = OpTypeInt 32 0         ; 32-bit unsigned int
%int      = OpTypeInt 32 1         ; 32-bit signed int
%ulong    = OpTypeInt 64 0         ; 64-bit unsigned int
%float    = OpTypeFloat 32
%double   = OpTypeFloat 64
%half     = OpTypeFloat 16

; Pointer types (storage class is part of the type):
%ptr_cs   = OpTypePointer CrossWorkgroup %float    ; global float*
%ptr_wg   = OpTypePointer Workgroup %float         ; shared float*
%ptr_fn   = OpTypePointer Function %float          ; local float*

; Vector and aggregate types:
%v4float  = OpTypeVector %float 4
%v3ulong  = OpTypeVector %ulong 3  ; used for built-in IDs

; Function type:
%fn_type  = OpTypeFunction %void %ptr_cs %ptr_cs
```

The LLVM SPIR-V backend materializes all required types at the beginning of the module, ordered by dependency (base types before derived types). This is enforced by `SPIRVModuleSectionPrinter`.

### 104.5.2 The Opaque Pointer Problem

LLVM 15+ uses opaque pointers — `ptr addrspace(N)` carries no pointee type information in the IR. SPIR-V requires pointee types for `OpTypePointer`. The SPIR-V backend must reconstruct pointee types by analyzing how each pointer is used: a load of `ptr addrspace(1)` producing an `i32` implies `OpTypePointer CrossWorkgroup %int`.

`SPIRVGlobalRegistry::getOrCreatePointerType` performs this reconstruction by walking def-use chains. When a pointer has multiple uses that imply different types (can happen with bitcast-elided opaque pointers), the backend inserts `OpBitcast` to reconcile the types.

### 104.5.3 Type Deduplication

SPIR-V forbids duplicate type declarations — two `OpTypeInt 32 0` would be invalid. `SPIRVGlobalRegistry` deduplicates by maintaining a map from type descriptor to ID. The exact deduplication rules follow the SPIR-V spec: two `OpTypeVector` instructions with the same element type and count are the same type; two `OpTypePointer` instructions with different storage classes but the same element type are different types (even if LLVM sees them as the same pointer type with different `addrspace`).

### 104.5.4 Verified Type Emission Example

The following LLVM IR:

```llvm
target triple = "spirv64-unknown-unknown"
target datalayout = "e-i64:64"

define spir_kernel void @vadd(ptr addrspace(1) %a,
                               ptr addrspace(1) %b,
                               ptr addrspace(1) %c) {
  %gid = call i64 @_Z13get_global_idj(i32 0)
  %ap = getelementptr float, ptr addrspace(1) %a, i64 %gid
  %bp = getelementptr float, ptr addrspace(1) %b, i64 %gid
  %cp = getelementptr float, ptr addrspace(1) %c, i64 %gid
  %av = load float, ptr addrspace(1) %ap
  %bv = load float, ptr addrspace(1) %bp
  %cv = fadd float %av, %bv
  store float %cv, ptr addrspace(1) %cp
  ret void
}

declare i64 @_Z13get_global_idj(i32)
```

Produces (verified with `llc -march=spirv64 --filetype=asm`):

```
    OpCapability Kernel
    OpCapability Addresses
    OpCapability Int64
    OpCapability Linkage
    %1 = OpExtInstImport "OpenCL.std"
    OpMemoryModel Physical64 OpenCL
    OpEntryPoint Kernel %13 "vadd" %9
    OpExecutionMode %13 ContractionOff
    OpSource OpenCL_CPP 100000
    OpDecorate %9 BuiltIn GlobalInvocationId
    ; Types:
    %2 = OpTypeFloat 32
    %3 = OpTypePointer CrossWorkgroup %2        ; float addrspace(1)*
    %4 = OpTypeVoid
    %5 = OpTypeFunction %4 %3 %3 %3            ; (ptr, ptr, ptr) → void
    %6 = OpTypeInt 64 0                         ; i64
    %7 = OpTypeVector %6 3                      ; <3 x i64> for BuiltIn
    %8 = OpTypePointer Input %7
    %9 = OpVariable %8 Input                    ; GlobalInvocationId builtin
    ; Function:
    %13 = OpFunction %4 None %5
    %10 = OpFunctionParameter %3               ; a
    %11 = OpFunctionParameter %3               ; b
    %12 = OpFunctionParameter %3               ; c
    %22 = OpLabel
    %14 = OpLoad %7 %9 Aligned 1              ; load builtin ID
    %15 = OpCompositeExtract %6 %14 0         ; gid = ID.x
    %16 = OpPtrAccessChain %3 %10 %15         ; a + gid
    %17 = OpPtrAccessChain %3 %11 %15         ; b + gid
    %18 = OpPtrAccessChain %3 %12 %15         ; c + gid
    %19 = OpLoad %2 %16 Aligned 4             ; load a[gid]
    %20 = OpLoad %2 %17 Aligned 4             ; load b[gid]
    %21 = OpFAdd %2 %19 %20                   ; a + b
    OpStore %18 %21 Aligned 4                  ; c[gid] = a + b
    OpReturn
    OpFunctionEnd
```

---

### 104.5.5 Image and Sampler Types

SPIR-V has a rich type system for GPU-specific objects not present in general-purpose IR:

```
; Texture / sampled image types (Vulkan):
%img_2d    = OpTypeImage %float 2D 0 0 0 1 Unknown  ; 2D, non-arrayed, non-MS, sampled
%samp_img  = OpTypeSampledImage %img_2d
%sampler   = OpTypeSampler

; Usage:
%loaded = OpImageSampleImplicitLod %v4float %sampled_img %coords
```

OpenCL's `image2d_t`, `sampler_t` types are similarly encoded. The SPIR-V backend maps Clang's `__read_only image2d_t` to the appropriate `OpTypeImage` with access qualifier encoding. These types are tracked by `SPIRVGlobalRegistry` along with scalar/pointer types and require the `ImageBasic` capability.

---

## 104.6 Capabilities and Extensions

### 104.6.1 Capability System

Every SPIR-V module must declare the capabilities it uses before employing instructions or types that require them. The SPIR-V specification defines a capability tree: `Shader` implies `Matrix`; `Float64` implies `Float16` is unnecessary to request separately.

The LLVM SPIR-V backend tracks required capabilities as instructions are selected. Some capability triggers:

| Trigger | Required Capability |
|---------|-------------------|
| Any float16 operation | `Float16Buffer`, `Float16` |
| Float64 operations | `Float64` |
| 64-bit integers | `Int64` |
| 8-bit integers | `Int8` |
| Atomic operations on 64-bit | `Int64Atomics` |
| `OpGroupNonUniformBallot` | `GroupNonUniformBallot` |
| Image types | `ImageBasic` |
| Sampled images | `Sampled1D`, etc. |
| Physical addressing (OpenCL) | `Addresses` |
| Kernel execution model | `Kernel` |
| Shader execution model | `Shader` |

Verified from actual SPIR-V output: an FP16 kernel triggers both `Float16Buffer` and `Float16`; an atomic i64 kernel triggers `Int64Atomics`:

```
; FP16 kernel output:
OpCapability Kernel
OpCapability Addresses
OpCapability Float16Buffer     ← triggered by half load
OpCapability Int64
OpCapability Float16           ← triggered by half arithmetic
OpCapability Linkage
```

### 104.6.2 Extensions

SPIR-V extensions augment capabilities for vendor-specific or provisional features:

```
; Extensions declared with OpExtension:
OpExtension "SPV_KHR_vulkan_memory_model"
OpExtension "SPV_EXT_shader_atomic_float_add"
OpExtension "SPV_KHR_cooperative_matrix"
OpExtension "SPV_INTEL_arbitrary_precision_integers"
```

The backend adds extensions when it encounters intrinsics or metadata that map to extended SPIR-V operations. The `SPIRVCapabilityTracker` ([`SPIRVCapabilityTracker.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/SPIRV/SPIRVCapabilityTracker.cpp)) manages both capability and extension requirements.

### 104.6.3 Extended Instruction Sets

SPIR-V defines extended instruction sets imported via `OpExtInstImport`. LLVM emits these for built-in functions:

```
; OpenCL built-in import:
%1 = OpExtInstImport "OpenCL.std"

; GLSL built-in import (Vulkan):
%2 = OpExtInstImport "GLSL.std.450"

; Usage:
%r = OpExtInst %float %1 21 %x    ; OpenCL.std sin(x)
%r = OpExtInst %float %2 14 %x    ; GLSL.std.450 Sin(x)
```

The `@_Z3sinf` function call from the OpenCL runtime is recognized as an `OpExtInst OpenCL.std` call; the selector maps the demangled function name to the extended opcode.

---

### 104.6.4 Capability Dependencies

Capabilities form a directed acyclic graph of dependencies. Declaring `Shader` implies `Matrix` is unnecessary; declaring `Float64` does not imply `Float16` (they are orthogonal). The SPIR-V specification table defines which capabilities imply which others. The backend must emit a capability only once (duplicate `OpCapability` is valid in SPIR-V but redundant), and must not emit a capability that requires another undeclared capability.

Some common capability chains:

```
Kernel → (nothing additional; top-level for OpenCL compute)
Shader → Matrix (implied)
Shader → ImageBasic → Sampled1D, SampledRect, SampledBuffer
GroupNonUniformBallot → GroupNonUniform (implied)
SubgroupDispatch → DeviceEnqueue (implied)
```

When using `spirv-val`, missing capability errors appear as:
```
error: ID <N> is not within scope of capability X
```

This means an instruction that requires capability X was emitted without `OpCapability X` in the module. The fix is to track the instruction type (e.g., a `OpGroupNonUniformBallot` requires `GroupNonUniformBallot`) and ensure the capability tracker fires when that instruction is selected.

---

## 104.7 Decoration and Metadata

### 104.7.1 SPIR-V Decorations

Decorations annotate IDs (variables, types, struct members) with properties that driver compilers use for layout, binding, and semantic correctness:

| Decoration | Applied to | Meaning |
|-----------|-----------|---------|
| `Location N` | shader in/out variable | interface slot binding |
| `Binding N` | descriptor set variable | binding in descriptor set |
| `DescriptorSet N` | buffer/sampler | which descriptor set |
| `BuiltIn X` | variable | maps to hardware built-in (gl_Position, etc.) |
| `Constant` | variable | compile-time constant |
| `Offset N` | struct member | byte offset in buffer layout |
| `ArrayStride N` | array pointer | element stride |
| `MatrixStride N` | matrix member | column/row stride |
| `Flat` | interpolant | no interpolation |
| `Invariant` | output | position invariance |

The LLVM SPIR-V backend derives decorations from:
1. `!spirv.Decorations` metadata attached to global variables
2. `addrspace` values (which imply storage class)
3. Built-in variable recognition by name pattern (`__spirv_BuiltInGlobalInvocationId`)

### 104.7.2 Built-in Variables

Built-in variables in SPIR-V are `OpVariable` instructions with `BuiltIn` decoration:

```
; Global invocation ID (OpenCL get_global_id equivalent):
%gid_type = OpTypePointer Input (OpTypeVector (OpTypeInt 64 0) 3)
%GlobalInvocationId = OpVariable %gid_type Input
    OpDecorate %GlobalInvocationId BuiltIn GlobalInvocationId

; Used in a kernel:
%gid_vec = OpLoad (OpTypeVector i64 3) %GlobalInvocationId Aligned 1
%gid_x   = OpCompositeExtract i64 %gid_vec 0
```

This is exactly what the LLVM backend emits when it encounters `@_Z13get_global_idj` — it recognizes this OpenCL demangled name and routes it to the `GlobalInvocationId` built-in variable.

---

## 104.8 Emission: SPIR-V Binary Writer

### 104.8.1 Binary Word Format

Every SPIR-V instruction is one or more 32-bit words. The first word encodes the instruction's opcode in bits [15:0] and the total word count in bits [31:16]:

```python
# Word 0 of any SPIR-V instruction:
word0 = (word_count << 16) | opcode

# Example: OpCapability Shader (op=17, wc=2)
# word0 = 0x00020011
# word1 = 0x00000001  (Capability::Shader = 1)
```

Result-producing instructions have a result type ID and result ID as their second and third words (after word0):

```
; OpFAdd %result_type %result_id %operand1 %operand2
; word count = 5
op_word = (5 << 16) | 129  ; OpFAdd = 129
type_id = ...
result_id = ...
op1 = ...
op2 = ...
```

### 104.8.2 Module Section Order

SPIR-V requires a strict canonical ordering of instructions within a module. The `SPIRVModuleSectionPrinter` enforces this:

```
┌─────────────────────────────────────────────────────────┐
│  SPIR-V Module Canonical Section Order                   │
├──────┬──────────────────────────────────────────────────┤
│  §1  │ OpCapability (all required capabilities)         │
│  §2  │ OpExtension (string names of extensions)         │
│  §3  │ OpExtInstImport (e.g., "OpenCL.std", "GLSL.std") │
│  §4  │ OpMemoryModel (Logical/Physical + GLSL450/OpenCL) │
│  §5  │ OpEntryPoint (execution model, fn ID, name, intf)│
│  §6  │ OpExecutionMode / OpExecutionModeId               │
│  §7  │ Debug: OpString, OpName, OpMemberName             │
│  §8  │ Annotations: OpDecorate, OpMemberDecorate         │
│  §9  │ Type declarations (dependency order):            │
│      │   OpTypeVoid < OpTypeFloat < OpTypePointer        │
│      │   OpTypeVector < OpTypeFunction (no cycles)       │
│ §10  │ OpConstant, OpConstantComposite (module-scope)   │
│ §11  │ OpVariable (non-Function storage class)           │
│ §12  │ Function definitions: OpFunction…OpFunctionEnd   │
└──────┴──────────────────────────────────────────────────┘
```

Any violation of this order (e.g., an `OpDecorate` appearing after the first type declaration, or an `OpTypeFloat` appearing after `OpTypePointer` that references it) makes the SPIR-V module invalid. The `spirv-val` tool detects these violations.

Any violation of this order produces an invalid SPIR-V module that `spirv-val` will reject. The backend's module analysis pass (`SPIRV module analysis`) collects all instructions and reorders them into this canonical sequence before the printer emits words.

### 104.8.3 ID Numbering and Binary Decode

SPIR-V IDs are monotonically increasing positive integers. The `%bound` field in the module header is the maximum ID value used plus one. `SPIRVGlobalRegistry` allocates virtual register numbers during codegen, and the printer maps these to final SPIR-V IDs.

Verified from binary decode of a real LLVM-generated SPIR-V module (`simple` kernel with `ptr addrspace(1) %out, i32 %n` args, compiled with `llc -march=spirv32 -filetype=obj`):

```
; Binary decode (little-endian 32-bit words):
; [word 0] 0x07230203  ← Magic number (identifies SPIR-V binary)
; [word 1] 0x00010400  ← Version 1.4 (major=1, minor=4)
; [word 2] 0x002B0016  ← Generator ID (LLVM SPIR-V backend)
; [word 3] 0x00000013  ← Bound = 19 (IDs 1..18 used)
; [word 4] 0x00000000  ← Schema (always 0)
;
; First instructions:
; [word 5-6]  op=17 wc=2 → OpCapability; operand=6 (Kernel)
; [word 7-8]  op=17 wc=2 → OpCapability; operand=4 (Addresses)
; [word 9-13] op=11 wc=5 → OpExtInstImport %1 "OpenCL.std"
; [word 14-16] op=14 wc=3 → OpMemoryModel Physical32 OpenCL
; [word 17-21] op=15 wc=5 → OpEntryPoint Kernel %8 "simple"
; [word 22-24] op=16 wc=3 → OpExecutionMode %8 ContractionOff
; ...
; [word 53-57] op=54 wc=5 → OpFunction %void %8 None %fn_type
; [word 58-60] op=55 wc=3 → OpFunctionParameter %ptr %out_param
; [word 61-63] op=55 wc=3 → OpFunctionParameter %int %n_param
; [word 64-65] op=248 wc=2 → OpLabel %bb
; [word 66-70] op=62 wc=5 → OpStore %out_param, %n_param Aligned 4
; [word 71]    op=253 wc=1 → OpReturn
; [word 72]    op=56 wc=1  → OpFunctionEnd
; Total: 73 words = 292 bytes
```

The word count field `wc` in each instruction's word 0 tells the parser how many words to consume, enabling single-pass parsing of the instruction stream.

### 104.8.4 SPIRVAsmPrinter

[`SPIRVAsmPrinter.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/SPIRV/SPIRVAsmPrinter.cpp) extends `AsmPrinter`. For binary output (`-filetype=obj`), it writes raw 32-bit words using a `raw_pwrite_stream`. For text output (`-filetype=asm`), it emits the human-readable assembly format with TAB-indented opcodes and `%id = Op...` notation — readable but not a standard SPIR-V format (the text format is LLVM-specific and not accepted by `spirv-as`).

---

## 104.8b SPIR-V for Vulkan: Logical Addressing Model

### 104.8b.1 Logical vs Physical Addressing

The Vulkan SPIR-V profile uses *Logical* addressing: `OpTypePointer` types exist, but pointer arithmetic (`OpPtrAccessChain` is valid), and the module cannot use `OpConvertUToPtr`, `OpConvertPtrToU`, or `inttoptr`/`ptrtoint` in general. This contrasts with the OpenCL *Physical64* model where pointer-integer conversions are valid.

The key consequence: Vulkan SPIR-V cannot represent arbitrary pointer provenance. All pointers must derive from `OpVariable` declarations through `OpPtrAccessChain` chains. The LLVM SPIR-V backend enforces this by inserting `OpBitcast` to the nearest legal pointer type when encountering integer-to-pointer or opaque-cast patterns.

### 104.8b.2 Interface Variables in Vulkan SPIR-V

Vulkan shaders communicate with the outside world through interface variables — `OpVariable` instructions in specific storage classes with mandatory decorations:

```
; Vertex shader input: per-vertex position
%pos_type   = OpTypeVector %float 4
%pos_ptr    = OpTypePointer Input %pos_type
%in_pos     = OpVariable %pos_ptr Input
    OpDecorate %in_pos Location 0        ; corresponds to layout(location=0) in:

; Vertex shader output: transformed position
%out_ptr    = OpTypePointer Output %pos_type
%out_pos    = OpVariable %out_ptr Output
    OpDecorate %out_pos BuiltIn Position ; gl_Position

; Uniform buffer (UBO): read-only structured data
%UBO_t      = OpTypeStruct %mat4 %mat4  ; struct { mat4 model; mat4 view; };
    OpMemberDecorate %UBO_t 0 Offset 0
    OpMemberDecorate %UBO_t 1 Offset 64
    OpDecorate %UBO_t Block              ; declares it as a uniform block
%ubo_ptr    = OpTypePointer Uniform %UBO_t
%ubo_var    = OpVariable %ubo_ptr Uniform
    OpDecorate %ubo_var DescriptorSet 0
    OpDecorate %ubo_var Binding 0        ; layout(set=0, binding=0) uniform UBO {...}
```

The `OpEntryPoint` instruction's interface ID list must contain all `Input` and `Output` variables the entry point uses. LLVM's SPIR-V backend derives this list from the global variables and function parameters that flow through the IR module.

### 104.8b.3 Push Constants

Vulkan push constants are small values (typically ≤ 128 bytes) passed to shaders via a fast SGPR-like mechanism. In SPIR-V:

```
%PushConst_t = OpTypeStruct %uint %uint %float
    OpDecorate %PushConst_t Block
    OpMemberDecorate %PushConst_t 0 Offset 0
    OpMemberDecorate %PushConst_t 1 Offset 4
    OpMemberDecorate %PushConst_t 2 Offset 8
%pc_ptr = OpTypePointer PushConstant %PushConst_t
%pc_var = OpVariable %pc_ptr PushConstant   ; no DescriptorSet/Binding decorations
```

The `PushConstant` storage class carries no descriptor set binding — push constants are accessed via `s_load_dwordx*` from a special SGPR region on AMD hardware or via dedicated fast-path on NVIDIA.

---

## 104.9 The Khronos llvm-spirv Translator

### 104.9.1 Design and Scope

The [Khronos SPIRV-LLVM-Translator](https://github.com/KhronosGroup/SPIRV-LLVM-Translator) follows a different design philosophy than the in-tree LLVM backend:

| Dimension | In-tree LLVM SPIRV backend | Khronos Translator |
|-----------|---------------------------|-------------------|
| Location | `llvm/lib/Target/SPIRV/` (in-tree) | Out-of-tree repository |
| Approach | General LLVM IR → SPIR-V | OpenCL-specific IR → SPIR-V |
| Function naming | Works with any names | Expects OpenCL `_Z` demangled names |
| SPIR-V extensions | Growing coverage | Broader coverage, faster updates |
| Bidirectional | LLVM→SPIR-V only | LLVM→SPIR-V and SPIR-V→LLVM |
| Users | Clang OpenCL/SYCL path (in progress) | Intel oneAPI, AMD ROCm OpenCL |

The translator's reverse path (SPIR-V → LLVM IR) is unique and enables round-tripping — used by debuggers and optimization tools that consume SPIR-V.

### 104.9.2 llvm-spirv Usage

```bash
# Khronos translator usage (when installed):
# Compile OpenCL C to LLVM IR with Clang:
clang -x cl -cl-std=CL2.0 -target spir64-unknown-unknown \
      -emit-llvm -S kernel.cl -o kernel.ll

# Translate LLVM IR to SPIR-V:
llvm-spirv kernel.ll -o kernel.spv

# Reverse: SPIR-V to LLVM IR:
llvm-spirv -r kernel.spv -o kernel_roundtrip.ll
```

### 104.9.3 When to Use Which

The in-tree LLVM SPIR-V backend is the better choice when:
- Compiling through the standard `llc` or Clang `--target=spirv*` flow
- Full LLVM optimization pipeline integration is needed
- Debugging through LLVM's pass infrastructure is required

The Khronos translator is preferred when:
- Maximum SPIR-V extension coverage is required (e.g., `SPV_INTEL_*` extensions)
- SPIR-V → LLVM IR round-tripping is needed
- Targeting Intel oneAPI or AMD OpenCL runtimes that expect translator-compatible conventions

---

## 104.10 SPIR-V in Vulkan and OpenCL Toolchains

### 104.10.1 Clang as a SPIR-V Compiler

Clang can target SPIR-V directly for OpenCL kernels:

```bash
# Compile OpenCL C to SPIR-V (64-bit, OpenCL profile):
clang -x cl -cl-std=CL2.0 \
      -target spirv64-unknown-unknown \
      kernel.cl -o kernel.spv

# Compile OpenCL C++ to SPIR-V:
clang -x clcpp -target spirv64-unknown-unknown \
      kernel.clcpp -o kernel.spv

# View SPIR-V text (if SPIRV-Tools spirv-dis is available):
spirv-dis kernel.spv
```

The Vulkan compute path:

```bash
# Compile SYCL to SPIR-V for Vulkan via Clang:
clang++ -fsycl -target spirv-unknown-vulkan1.3 \
        -std=c++17 compute.cpp -o compute.spv
```

### 104.10.2 GLSL and HLSL to SPIR-V

GLSL shaders are compiled to SPIR-V by `glslangValidator` (Khronos reference compiler) or `glslc` (Google's wrapper). A minimal Vulkan compute shader in GLSL:

```glsl
// File: add_one.comp (GLSL compute shader)
#version 450
#extension GL_EXT_shader_explicit_arithmetic_types_int32 : require

layout(local_size_x = 64, local_size_y = 1, local_size_z = 1) in;

layout(set = 0, binding = 0) buffer DataBuffer {
    int data[];
};

void main() {
    uint gid = gl_GlobalInvocationID.x;
    data[gid] = data[gid] + 1;
}
```

Compiled to SPIR-V, the `layout(binding = 0)` annotation becomes:
```
OpDecorate %data DescriptorSet 0
OpDecorate %data Binding 0
```

The `gl_GlobalInvocationID` built-in becomes an input variable with `BuiltIn GlobalInvocationId` decoration, analogous to how the OpenCL path generates it from `_Z13get_global_idj`. The key SPIR-V difference from the OpenCL kernel above: the execution model is `GLCompute` instead of `Kernel`, and the memory model is `GLSL450` instead of `OpenCL`.

```bash
# Compile vertex shader:
glslc shader.vert -o vert.spv
# Or:
glslangValidator -V shader.vert -o vert.spv
```

HLSL can be compiled to SPIR-V by DirectXShaderCompiler (dxc) with `-spirv`:

```bash
dxc -spirv -T cs_6_0 compute.hlsl -Fo compute.spv
```

### 104.10.3 clspv: OpenCL via LLVM

`clspv` (Google's OpenCL to SPIR-V compiler, used by Vulkan compute over OpenCL kernels) uses LLVM's SPIR-V backend path internally:

```bash
# clspv compiles OpenCL C targeting Vulkan's SPIR-V dialect:
clspv kernel.cl -o kernel.spv

# It applies its own passes before invoking the SPIRV backend,
# including descriptor set layout analysis and push-constant inference.
```

### 104.10.4 Validation with SPIRV-Tools

All generated SPIR-V should be validated with `spirv-val` from the SPIRV-Tools suite before deployment:

```bash
# Validate a SPIR-V binary:
spirv-val kernel.spv

# Disassemble to text format:
spirv-dis kernel.spv

# Assemble from text format:
spirv-as kernel.spvasm -o kernel.spv

# Optimize:
spirv-opt -O kernel.spv -o kernel_opt.spv

# Show capabilities used:
spirv-reflect kernel.spv
```

Common validation errors from incorrect LLVM SPIR-V output include: duplicate type declarations, missing capability declarations, out-of-order module sections, and pointer type mismatches. These should be treated as backend bugs — the LLVM SPIR-V backend is expected to produce valid SPIR-V for any conforming LLVM IR input.

### 104.10.5 The MLIR SPIR-V Dialect

MLIR provides a high-level SPIR-V dialect (`mlir/lib/Dialect/SPIRV/`) that operates at the MLIR level rather than the LLVM IR level. It is covered in detail in [Chapter 143](../part-20-in-tree-dialects/ch143-spirv-dialect.md). The key difference: the MLIR SPIR-V dialect represents SPIR-V operations as MLIR ops with full type fidelity (SPIR-V types are first-class MLIR types), making round-trips and SPIR-V-specific optimizations easier. The LLVM SPIR-V backend, by contrast, receives opaque LLVM IR and must reconstruct SPIR-V type information. The two paths serve different use cases: MLIR's dialect for high-level GPU compute frameworks (XLA, TensorFlow, IREE), the LLVM backend for existing LLVM-based OpenCL/Vulkan toolchains.

---

## 104.11 Debugging the SPIR-V Backend

### 104.11.1 Generating Text vs Binary

```bash
# Text assembly (LLVM-specific format, not spirv-as compatible):
/usr/lib/llvm-22/bin/llc -march=spirv64 \
    --filetype=asm kernel.ll -o kernel.spvasm

# Binary output (valid SPIR-V):
/usr/lib/llvm-22/bin/llc -march=spirv64 \
    -filetype=obj kernel.ll -o kernel.spv

# Inspect binary header:
python3 -c "
import struct
data = open('kernel.spv','rb').read()
magic,ver,gen,bound,schema = struct.unpack('<5I',data[:20])
print(f'Magic: 0x{magic:08X}')
print(f'SPIR-V version: {(ver>>16)&0xff}.{(ver>>8)&0xff}')
print(f'Bound (max ID + 1): {bound}')
"
```

### 104.11.2 Pass-Level Inspection

```bash
# Print MIR after each SPIRV pass:
/usr/lib/llvm-22/bin/llc -march=spirv64 \
    -debug-pass=Structure \
    kernel.ll --filetype=asm -o /dev/null

# Stop after legalization:
/usr/lib/llvm-22/bin/llc -march=spirv64 \
    --stop-after=legalizer \
    kernel.ll -o kernel.mir

# Dump pre-legalization G_* MIR:
/usr/lib/llvm-22/bin/llc -march=spirv64 \
    --stop-after=irtranslator \
    kernel.ll -o kernel_gmir.mir
```

### 104.11.3 Validating Output

```bash
# If SPIRV-Tools is installed:
spirv-val --target-env opencl2.2 kernel.spv
spirv-dis kernel.spv | head -40

# Check capabilities emitted:
spirv-dis kernel.spv | grep OpCapability

# Validate Vulkan SPIR-V:
spirv-val --target-env vulkan1.3 shader.spv
```

### 104.11.4 Common SPIR-V Validation Errors from the LLVM Backend

Understanding which LLVM IR patterns produce which SPIR-V validation errors helps diagnose backend bugs and user input errors:

| LLVM IR pattern | Likely SPIR-V validation error |
|----------------|-------------------------------|
| Float16 arithmetic without `-target-feature +fp16` in OpenCL | `Float16Buffer` capability missing |
| `atomicrmw` on 64-bit without `-cl-std=CL2.0` | `Int64Atomics` capability missing |
| Generic `ptr` (no addrspace) passed to kernel | `OpTypePointer Function` as kernel parameter (invalid) |
| `getelementptr` through opaque struct | Type reconstruction fails → `OpPtrAccessChain` type mismatch |
| `select` on `ptr` type | `OpSelect` with pointer type requires `VariablePointers` capability |
| `alloca` with non-Function storage | `OpVariable` in Function storage class outside a function |

The backend aims to produce valid SPIR-V for any conformant input, but edge cases around opaque pointers, unsupported extension usage, and Vulkan-vs-OpenCL profile mismatches can produce invalid output. Always run `spirv-val` as part of the build pipeline.

### 104.11.5 SPIR-V Optimization with spirv-opt

The SPIRV-Tools optimizer (`spirv-opt`) can improve SPIR-V before it reaches the GPU driver. Since each driver's SPIR-V compiler re-does optimization from scratch, `spirv-opt` at the SPIR-V level primarily targets:
- Dead code elimination at the SPIR-V module level
- Constant folding and propagation
- Inlining of short functions
- Eliminating redundant `OpLoad` / `OpStore` sequences

```bash
# Apply standard optimization preset:
spirv-opt -O kernel.spv -o kernel_opt.spv

# Target-specific: optimize for Vulkan 1.3 environment:
spirv-opt --target-env vulkan1.3 -O shader.spv -o shader_opt.spv

# Individual passes:
spirv-opt \
    --eliminate-dead-code-aggressive \
    --eliminate-dead-functions \
    --inline-entry-points-exhaustive \
    kernel.spv -o kernel_opt.spv
```

Note that `spirv-opt` operates on the logical SPIR-V representation, not on PTX or GCN ISA — its optimizations are complementary to what a GPU driver's JIT compiler performs.

### 104.11.6 Subgroup and Cooperative Matrix Extensions

Modern compute workloads on Vulkan use subgroup operations (analogous to warp-level operations in CUDA) and cooperative matrix (analogous to tensor core / WMMA) instructions:

```llvm
; Subgroup ballot (Vulkan: SPV_KHR_shader_ballot → SPIR-V 1.3 SubgroupBallotKHR)
; In SPIR-V:
;   OpCapability GroupNonUniformBallot
;   %ballot = OpGroupNonUniformBallot %v4uint Subgroup %true
;
; In LLVM IR targeting spirv-unknown-vulkan:
%ballot = call <4 x i32> @llvm.spv.GroupNonUniformBallot(i32 3, i1 true)
;                                                         ^-- Subgroup scope = 3

; Cooperative matrix (Vulkan: SPV_KHR_cooperative_matrix):
;   OpCapability CooperativeMatrixKHR
;   %mat_a = OpCooperativeMatrixLoadKHR %MatType %ptr Aligned 16 ...
```

These map to SPIR-V opcodes from the corresponding extensions, which must be declared with `OpExtension "SPV_KHR_cooperative_matrix"` and `OpCapability CooperativeMatrixKHR`.

Cross-references: The NVPTX backend for CUDA/OpenCL is covered in [Chapter 102](../part-15-targets/ch102-nvptx-and-the-cuda-path.md). The AMDGPU backend and its ROCm OpenCL-SPIR-V path is [Chapter 103](../part-15-targets/ch103-amdgpu-and-the-rocm-path.md). The MLIR SPIR-V dialect with its structured representation is [Chapter 143](../part-20-in-tree-dialects/ch143-spirv-dialect.md). GlobalISel, which the SPIR-V backend uses exclusively, is covered in [Chapter 86](../part-14-backend/ch86-global-isel.md).

---

## 104.12 SPIR-V Backend Architecture: End-to-End Summary

### 104.12.1 Complete Data Flow

Tracing a single `fadd float` operation from LLVM IR to SPIR-V binary words:

```
LLVM IR:
  %cv = fadd float %av, %bv

SPIRVEmitIntrinsics (pre-ISel):
  → injects @llvm.spv.assign.type.f32(%av, metadata float), same for %bv, %cv
  → ensures type context is available for legalizer

SPIRVIRTranslator (GlobalISel IRTranslator):
  %cv:_(s32) = G_FADD %av:_(s32), %bv:_(s32)

SPIRVPreLegalizerCombiner:
  (no change for simple fadd)

Legalizer:
  G_FADD s32 is legal in SPIR-V → no change

SPIRVInstructionSelector:
  Maps G_FADD f32 → SPIRVAdd(OpFAdd, type=%float_id, result=%cv_id, op1=%av_id, op2=%bv_id)

SPIRVModuleAnalysis:
  Collects all OpType*, OpConstant*, OpDecorate in canonical order
  Ensures OpTypeFloat 32 is declared once (ID %float_id)

SPIRVAsmPrinter (text mode):
  emits: "%21 = OpFAdd %2 %19 %20"
    (where %2=OpTypeFloat32 ID, %19=av ID, %20=bv ID, %21=result ID)

SPIRVAsmPrinter (binary mode, -filetype=obj):
  word0 = (5 << 16) | 136   ; OpFAdd = 136, wc = 5
  word1 = float_type_id     ; result type
  word2 = result_id         ; %21
  word3 = av_id             ; %19
  word4 = bv_id             ; %20
```

### 104.12.2 Comparison: SelectionDAG vs GlobalISel for SPIR-V

The choice of GlobalISel over SelectionDAG is worth examining concretely:

**SelectionDAG would require:**
- Physical register classes (none exist in SPIR-V — all values are virtual IDs)
- A register allocator (SPIR-V has no registers to allocate)
- A target data layout that maps to physical byte offsets (SPIR-V uses logical types)
- Pattern matching tables that map DAG types to opcodes — workable but structurally awkward for a logical-addressing target

**GlobalISel allows:**
- `G_*` opcodes map 1:1 to SPIR-V opcodes without register allocation
- The legalizer can enforce SPIR-V type constraints (no i1 vectors, no arbitrary-width ints)
- The `SPIRVGlobalRegistry` acts as the "register allocator" — assigning monotonic IDs rather than hardware registers
- Type metadata can flow through `G_INTRINSIC` nodes naturally

This is why the SPIR-V backend passes list shows "SPIRV module analysis" immediately before "SPIRV Assembly Printer" with no register allocator or post-RA passes — the entire post-selection phase is replaced by the ID-assignment and module-ordering logic in `SPIRVGlobalRegistry` and `SPIRVModuleSectionPrinter`.

### 104.12.3 Portability Across GPU Targets

SPIR-V's logical addressing model makes it genuinely portable across GPU architectures in a way PTX and GCN ISA are not. A single SPIR-V module can be consumed by:

- **NVIDIA Vulkan drivers (nvk)**: SPIR-V → NVPTX-like ISA via NVIDIA's proprietary SPIR-V front-end
- **AMD RADV/Mesa**: SPIR-V → LLVM IR → AMDGPU backend → GCN ISA (ironically, re-entering LLVM)
- **Intel ANV/CRU**: SPIR-V → EU ISA via Intel's IGC (also LLVM-based internally)
- **ARM Mali**: SPIR-V → Bifrost/Valhall ISA
- **Qualcomm Adreno**: SPIR-V → proprietary backend

The LLVM SPIR-V backend thus occupies a dual role: it is a backend that *produces* the portable intermediate format that other backends *consume*. Understanding this position clarifies why getting the type system and capability declarations correct matters: driver SPIR-V compilers depend on them for correct code generation.

---

## Chapter Summary

- SPIR-V is a logical binary intermediate language: no register allocation, no physical addressing (Vulkan profile), mandatory capability declarations, and a canonical module ordering that all drivers rely on.
- The in-tree LLVM SPIR-V backend (`llvm/lib/Target/SPIRV/`) uses GlobalISel exclusively — `SPIRVIRTranslator`, `SPIRVLegalizer`, `SPIRVInstructionSelector` — because SelectionDAG's physical-machine assumptions conflict with SPIR-V's logical model.
- `SPIRVGlobalRegistry` solves the opaque pointer problem by tracking type assignments across the module and deduplicating `OpType*` instructions, ensuring each unique type gets exactly one SPIR-V ID.
- Capabilities are collected incrementally as instructions are selected; the `SPIRVCapabilityTracker` emits the complete set at module header time. Float16 arithmetic triggers `Float16Buffer` + `Float16`; 64-bit atomics trigger `Int64Atomics`.
- `SPIRVModuleSectionPrinter` enforces the canonical section order (capabilities → extensions → types → globals → functions) required by the SPIR-V specification.
- SPIR-V binary format is 32-bit words; the header carries magic `0x07230203`, version, generator, bound (max ID + 1), and schema. Each instruction encodes word count in bits [31:16] and opcode in bits [15:0].
- The Khronos `llvm-spirv` translator provides an alternative out-of-tree path with broader SPIR-V extension coverage and bidirectional translation; the in-tree backend offers better LLVM integration and debuggability.
- `spirv-val` from SPIRV-Tools should be run on all generated SPIR-V to catch capability omissions, type mismatches, and ordering violations before deployment to a Vulkan or OpenCL runtime.
- The MLIR SPIR-V dialect (`mlir/lib/Dialect/SPIRV/`) provides a higher-level alternative for frameworks that generate SPIR-V from MLIR, preserving type fidelity that the LLVM IR path must reconstruct.


---

@copyright jreuben11
