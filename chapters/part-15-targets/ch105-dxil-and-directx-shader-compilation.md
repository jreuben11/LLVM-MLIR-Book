# Chapter 105 — DXIL and DirectX Shader Compilation

*Part XV — Targets*

The DirectX shader pipeline occupies a unique position in the LLVM ecosystem: it is simultaneously the most widely deployed LLVM-derived compiler in consumer hardware (running inside every DirectX 12 application on Windows) and the least visible to systems programmers accustomed to CPU targets. DXIL — DirectX Intermediate Language — is LLVM IR restricted to a carefully constrained subset, serialized as bitcode, validated by a driver-level verifier, and then compiled to proprietary GPU ISAs by D3D12 runtime shader compilers. Understanding the DXIL backend illuminates how LLVM IR can be repurposed as a GPU portability layer, why structural restrictions (no indirect calls, no recursion, no pointer arithmetic on resources) exist by design rather than oversight, and how the in-tree LLVM DirectX backend relates to but differs from Microsoft's standalone `dxc` tool. This chapter covers the full path from HLSL source through the Clang HLSL frontend, DXIL operation lowering, resource handling, container format, validation, and Shader Model feature progression.

---

## 105.1 DXIL Overview

### 105.1.1 DXIL as Restricted LLVM IR

DXIL is LLVM 3.7-era bitcode with additional structural constraints imposed by the DirectX specification. Every DXIL file is a valid LLVM module, but not every LLVM module is valid DXIL. The key restrictions are:

- No indirect function calls (call target must be statically known)
- No recursion (direct or indirect)
- No use of `alloca` outside entry function prologues
- No pointer arithmetic on resource handle types
- Floating-point operations must use IEEE semantics (no fast-math flags that violate IEEE)
- All resource accesses must be through `@dx.op.*` intrinsic calls with typed handles
- The module must contain exactly one entry point per shader stage, annotated via metadata

The LLVM IR subset used is intentionally small: only arithmetic, loads/stores from typed buffers via `dx.op`, branches, `phi` nodes, and `call` to `@dx.op.*` functions. This restriction enables aggressive driver-side JIT compilation with guaranteed termination properties.

### 105.1.2 Shader Model Versions and DXIL Versions

DXIL versioning tracks the DirectX Shader Model (SM) version:

| DXIL Version | Shader Model | Key Features |
|-------------|-------------|--------------|
| 1.0 | SM 6.0 | Wave intrinsics, 64-bit types, basic 16-bit |
| 1.1 | SM 6.1 | SV_ViewID, barycentric coordinates |
| 1.2 | SM 6.2 | Full 16-bit shader types, denorm modes |
| 1.3 | SM 6.3 | DXR 1.0 (ray tracing), WriteMSAADepth |
| 1.4 | SM 6.4 | Integer dot products, packed/unpacked i8 math |
| 1.5 | SM 6.5 | DXR 1.1, mesh/amplification shaders |
| 1.6 | SM 6.6 | Payload access qualifiers, raw/structured buffer atomics |
| 1.7 | SM 6.7 | QuadAny/QuadAll, feedback texture sampling |
| 1.8 | SM 6.8 | Work graphs, shader execution reordering |

The target triple encodes the shader model: `dxil-ms-dx` in the in-tree backend, or historically `dxil-unknown-shadermodel6.3-` for DXR shaders.

### 105.1.3 DXIL Container Format

A compiled DXIL shader is not a bare bitcode file; it is a **DXContainer** — a structured binary container with multiple chunks. The container format is also known as DXBC (DirectX Bytecode) from the D3D11 era, repurposed for DXIL in D3D12:

```
DXContainer binary layout:
  Offset 0:  DXContainerHeader
    [0..3]   magic = 'DXBC'   (0x43425844)
    [4..19]  hash[16]         -- MD5 over entire file (for driver caching)
    [20..23] major=1, minor=0 -- container format version
    [24..27] fileSize         -- total file size in bytes
    [28..31] chunkCount       -- number of chunks
  Offset 32: uint32_t chunkOffsets[chunkCount]
  Chunks (each at its offset):
    ChunkHeader: fourCC[4], chunkSize(uint32_t)
    Chunk data...

Chunk types:
  'DXIL' (0x4C495844) -- DXIL program: DxilProgramHeader + LLVM bitcode
  'RDAT' (0x54414452) -- Runtime Data: resource bindings, entry metadata
  'RTS0' (0x30535452) -- Root Signature version 1.0 (serialized D3D12_ROOT_SIGNATURE_DESC)
  'RTS1' (0x31535452) -- Root Signature version 1.1
  'PSV0' (0x30565350) -- Pipeline State Validation data
  'ISGN' (0x4E475349) -- Input Signature (vertex shader inputs)
  'OSGN' (0x4E47534F) -- Output Signature (shader outputs)
  'ISG1' (0x31475349) -- Extended Input Signature (SM 5.1+)
  'OSG1' (0x3147534F) -- Extended Output Signature
  'STAT' (0x54415453) -- Shader Statistics (instruction counts by category)
  'HASH' (0x48534148) -- Shader hash (DXIL-specific hash for driver caching)
  'SFI0' (0x30494653) -- Shader Feature Info (feature flags bitmask)
  'PRIV' (0x56495250) -- Private data (arbitrary blob, debug use)
  'ILDB' (0x42444C49) -- Debug info (when compiled with -Zi, PDB-like)
```

The `DXIL` chunk contains a nested `DxilProgramHeader`:
```
DxilProgramHeader:
  [0..3]   dxilMagic = 0x4C495844  ('DXIL')
  [4..7]   dxilVersion: minor(16), major(16) -- DXIL version (e.g., 1.0 = SM 6.0)
  [8..11]  bitcodeOffset -- offset from start of chunk data to bitcode
  [12..15] bitcodeSize   -- size of the LLVM bitcode blob
  [16..]   LLVM bitcode (LLVM 3.7 bitcode format, frozen)
```

The bitcode uses LLVM's standard bitstream encoding but only opcodes present in LLVM 3.7 are valid (the format was frozen at that version to ensure driver compatibility). The RDAT chunk deserves special attention: it encodes all reflection information needed by the D3D12 runtime without requiring the driver to parse LLVM bitcode, including resource binding records (type, register space, register index, count), entry point metadata (shader stage, numthreads for compute), and function dependency information for library targets.

### 105.1.4 DXIL vs SPIR-V

| Aspect | DXIL | SPIR-V |
|--------|------|--------|
| Base format | LLVM 3.7 bitcode | Custom binary format |
| Operations | `@dx.op.*` calls | SPIR-V OpCode instructions |
| Source language | HLSL (primarily) | GLSL, HLSL, OpenCL C, SYCL |
| Validation | `dxil.dll` or in-driver | `spirv-val` toolchain |
| Driver target | D3D12 (Windows) | Vulkan, OpenGL, OpenCL |
| Reflection | Embedded RDAT chunk | Decoration metadata |
| Type system | Opaque handles | Explicit pointer types |

SPIR-V chose a compact custom encoding optimized for parsing speed; DXIL chose to reuse LLVM's bitcode machinery because the DXC team already had an LLVM-based compiler infrastructure. Both serve as stable intermediate layers insulating driver compilers from source language evolution.

---

## 105.2 The DirectX Backend in LLVM

### 105.2.1 Backend Structure

The in-tree DirectX backend lives in [`llvm/lib/Target/DirectX/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/DirectX/) and was introduced in LLVM 15. Unlike traditional targets that produce native machine code, the DirectX backend's job is to:

1. Lower LLVM IR operations to DXIL-legal IR (replacing intrinsics, validating constraints)
2. Emit the DXIL container (bitcode + metadata chunks)

The backend registers as a standard `TargetMachine`:

```cpp
// DirectXTargetMachine.h
class DirectXTargetMachine : public LLVMTargetMachine {
public:
  DirectXTargetMachine(const Target &T, const Triple &TT,
                       StringRef CPU, StringRef FS,
                       const TargetOptions &Options,
                       std::optional<Reloc::Model> RM,
                       std::optional<CodeModel::Model> CM,
                       CodeGenOptLevel OL, bool JIT);

  TargetPassConfig *createPassConfig(PassManagerBase &PM) override;
  const DirectXSubtarget *getSubtargetImpl(const Function &F) const override;
};
```

The target triple used is `dxil-ms-dx`. The `DirectXSubtarget` holds the target shader model version extracted from the triple.

### 105.2.2 Target-Specific Pass Pipeline

The DirectX backend adds a sequence of IR-level passes before bitcode emission. Unlike most targets where passes operate on MachineIR, the DirectX backend works entirely at the LLVM IR level:

```
DirectXTargetMachine::createPassConfig():
  ── IR-level preparation ──────────────────────────────────
  DXILPrepareModule           -- normalize for DXIL: fix undef args,
                                 remove opaque pointer metadata, clean attrs
  DXILTranslateMetadata       -- emit dx.shaderModel, dx.resources,
                                 dx.entryPoints, dx.typeAnnotations metadata
  DXILFlattenArrays           -- flatten multi-dimensional LLVM arrays to 1D
                                 (DXIL only allows 1D arrays in resource types)
  DXILResourceAccess          -- lower typed resource load/store to dx.op calls
                                 (handles target("dx.Texture",...) intrinsic ops)
  DXILOpLowering              -- lower remaining LLVM intrinsics (@llvm.fabs,
                                 @llvm.sqrt, etc.) to @dx.op.* calls
  DXILIntrinsicExpansion      -- expand some intrinsics not 1:1 with dx.op
  ── Analysis ──────────────────────────────────────────────
  DXILResourceBindingAnalysis -- assign register/space bindings to resources
  DXILResourceTypeAnalysis    -- resolve target extension types to DXIL types
  ── Validation ────────────────────────────────────────────
  DXILPostOptimizationValidation -- verify structural constraints hold after opts
  ── Emission ──────────────────────────────────────────────
  DXILBitcodeWriter           -- emit DXIL container (not standard LLVM bitcoder)
```

Key pass source files:

| Pass | Source |
|------|--------|
| `DXILPrepareModule` | [`DXILPrepareModule.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/DirectX/DXILPrepareModule.cpp) |
| `DXILOpLowering` | [`DXILOpLowering.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/DirectX/DXILOpLowering.cpp) |
| `DXILTranslateMetadata` | [`DXILTranslateMetadata.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/DirectX/DXILTranslateMetadata.cpp) |
| `DXILFlattenArrays` | [`DXILFlattenArrays.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/DirectX/DXILFlattenArrays.cpp) |
| `DXILResourceAccess` | [`DXILResourceAccess.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/DirectX/DXILResourceAccess.cpp) |

### 105.2.3 DXILPrepareModule Pass Details

[`DXILPrepareModule`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/DirectX/DXILPrepareModule.cpp) performs normalization steps that DXIL requires but LLVM IR does not:

1. **Pointer mode normalization**: DXIL uses typed pointers (pre-opaque-pointer style). The pass ensures that all `getelementptr` instructions have explicit type annotations matching DXIL's expected format.
2. **`undef` argument elimination**: DXIL validators reject `undef` in certain positions. The pass replaces `undef` function arguments with zero values of the appropriate type.
3. **Attribute stripping**: LLVM IR attributes like `speculatable`, `willreturn`, `nofree` are not valid in DXIL; they are stripped from `@dx.op.*` function declarations.
4. **DXIL-incompatible instruction lowering**: Instructions like `freeze` (introduced in LLVM 10) have no DXIL equivalent; they are lowered to their operand.

### 105.2.4 DXIL Metadata Format

DXIL uses LLVM named metadata extensively for reflection and configuration. The key metadata nodes written by `DXILTranslateMetadata`:

```llvm
; Shader model
!dx.shaderModel = !{!0}
!0 = !{!"cs", i32 6, i32 0}   ; compute shader, SM 6.0

; Resources (SRVs, UAVs, CBVs, Samplers) — four nullable lists
!dx.resources = !{!1, !2, !3, null}
;                 ^    ^    ^    ^-- Sampler list (null = none)
;                 |    |    +------- CBV list
;                 |    +------------ UAV list
;                 +----------------- SRV list

; SRV list: each element is a metadata node per resource
!1 = !{!10}
!10 = !{i32 0,                    ; ID (index in binding table)
        %dx.types.Handle* undef,  ; placeholder (for typed info)
        !"InputBuffer",           ; variable name
        i32 0,                    ; shader register
        i32 0,                    ; register space
        i32 1,                    ; range size (1 = non-array)
        i32 12,                   ; resource kind: StructuredBuffer=12
        i32 4,                    ; element stride in bytes
        i1 false,                 ; globally coherent
        i1 false,                 ; HasCounter
        i32 0}                    ; sample count (0 for non-MS)

; UAV list
!2 = !{!20}
!20 = !{i32 0, %dx.types.Handle* undef, !"OutputBuffer",
        i32 0, i32 0, i32 1, i32 12, i32 4, i1 false, i1 false, i32 0}

; CBV list
!3 = !{!30}
!30 = !{i32 0, %dx.types.Handle* undef, !"Constants",
        i32 0, i32 0, i32 1,
        i32 8}    ; cbuffer size in bytes (must be multiple of 16)

; Entry point
!dx.entryPoints = !{!100}
!100 = !{void ()* @main,  ; function pointer
         !"main",          ; export name
         !101,             ; signature (null for compute)
         !102,             ; resources used by this entry
         !103}             ; shader-specific properties

!102 = !{!1, !2, !3, null}  ; resources used = same as dx.resources

; Compute shader properties
!103 = !{i32 0,      ; tag: 0 = NumThreads
          i32 64,    ; numthreads X
          i32 1,     ; numthreads Y
          i32 1}     ; numthreads Z

; dx.typeAnnotations: DXIL type info for structs (needed for structured buffers)
!dx.typeAnnotations = !{!200}
!200 = !{i32 1,          ; tag: struct annotation
         %MyStruct undef, ; representative value of the struct type
         i32 16,          ; struct byte size
         !201}            ; field annotations
!201 = !{i32 0, !"x", i32 4, i32 0, i32 0,  ; field 0: float x
         i32 4, !"y", i32 4, i32 0, i32 0,  ; field 1: float y
         i32 8, !"z", i32 4, i32 0, i32 0,  ; field 2: float z
         i32 12, !"w", i32 4, i32 0, i32 0} ; field 3: float w
```

### 105.2.5 Relationship to Standalone DXC

The standalone [`DirectXShaderCompiler (dxc)`](https://github.com/microsoft/DirectXShaderCompiler) is a Microsoft-maintained fork of LLVM 3.7 with extensive private Microsoft HLSL support layered on top. It predates the in-tree LLVM DirectX backend by several years and has a broader feature set including:

- Full HLSL language support (all SM 6.x language features)
- `dxil-val`: the DXIL validator (required for signing on Windows)
- Library targets (`.lib` DXIL), linking multiple DXIL modules
- SPIR-V backend (via the `dxc -spirv` path from a community contributor)

The in-tree LLVM backend aims for eventual convergence: Clang's HLSL frontend (covered in [Chapter 51 — Clang as an HLSL Compiler](../part-07-clang-multilang/ch51-clang-as-hlsl-compiler.md)) targets the in-tree backend, providing an open-source HLSL compilation path within the upstream LLVM project.

---

## 105.3 HLSL as a Source Language

### 105.3.1 HLSL Language Fundamentals

HLSL (High-Level Shader Language) is a C-like language with GPU-specific types and semantics. Clang's HLSL support uses `__attribute__((shader("compute")))` to mark entry points:

```hlsl
// compute.hlsl — simple image processing kernel
RWStructuredBuffer<float4> OutputBuffer : register(u0);
StructuredBuffer<float4>   InputBuffer  : register(t0);
cbuffer Constants : register(b0) {
    uint ElementCount;
    float Scale;
}

[numthreads(64, 1, 1)]
void main(uint3 DTid : SV_DispatchThreadID) {
    uint idx = DTid.x;
    if (idx >= ElementCount) return;
    OutputBuffer[idx] = InputBuffer[idx] * Scale;
}
```

Compiling with Clang:

```bash
clang -target dxil-ms-dx -fprofile-instr-generate \
      -D__HLSL_VERSION=2021 -fhlsl-entry=main \
      -hlsl-shader-model=cs_6_0 \
      -x hlsl compute.hlsl -o compute.dxil
```

The Clang HLSL frontend parses HLSL syntax including semantic annotations (`: SV_DispatchThreadID`), register bindings (`: register(u0)`), and shader stage attributes (`[numthreads]`). These are lowered to LLVM metadata in the IR.

### 105.3.2 HLSL Type System

HLSL's type system maps to LLVM IR types as follows:

| HLSL Type | LLVM IR Type |
|-----------|-------------|
| `float` | `float` |
| `float4` | `<4 x float>` |
| `half` / `float16_t` | `half` (i16 in DXIL) |
| `uint`, `int` | `i32` |
| `uint64_t` | `i64` |
| `bool` | `i1` / `i32` (context dependent) |
| `matrix<float,4,4>` | `[4 x <4 x float>]` (flattened) |
| `RWStructuredBuffer<T>` | `target("dx.RawBuffer", T, 1, 0)` |
| `StructuredBuffer<T>` | `target("dx.RawBuffer", T, 0, 0)` |
| `Texture2D<float4>` | `target("dx.Texture", float, 0, 0, 2)` |
| `SamplerState` | `target("dx.Sampler", 0)` |

The `target(...)` typed pointer representation was introduced in LLVM 15 as part of the opaque pointer transition, replacing the older `%dx.types.Handle` opaque struct type used in DXIL 1.0–1.4 bitcode.

### 105.3.3 Semantic Annotations and System Values

HLSL semantic strings (e.g., `SV_DispatchThreadID`) are system-value inputs from the GPU hardware. They are lowered through `@dx.op.*` calls in the generated DXIL:

| HLSL Semantic | DXIL Operation | OpCode |
|--------------|---------------|--------|
| `SV_DispatchThreadID` | `dx.op.threadId` | 93 |
| `SV_GroupIndex` | `dx.op.flattenedThreadIdInGroup` | 96 |
| `SV_GroupThreadID` | `dx.op.threadIdInGroup` | 95 |
| `SV_GroupID` | `dx.op.groupId` | 94 |
| `SV_Position` | Pixel shader input, `dx.op.loadInput` | 4 |
| `SV_Target` | Pixel shader output, `dx.op.storeOutput` | 5 |
| `SV_VertexID` | `dx.op.loadInput` with SV type | 4 |

---

## 105.4 DXIL Operations

### 105.4.0 Vertex and Pixel Shader Operation Patterns

Before examining the operation encoding, it is instructive to see complete DXIL IR for shader stages beyond compute. A minimal vertex shader passing position through:

```llvm
; Vertex shader: float4 main(float4 pos : POSITION) : SV_Position
define void @vs_main() {
entry:
  ; Load vertex input at index 0, component 0 (x)
  %x = call float @dx.op.loadInput.f32(
      i32 4,      ; opcode: LoadInput
      i32 0,      ; input element index (POSITION is element 0)
      i32 0,      ; row index (0 for non-array semantics)
      i8 0,       ; component: 0=x, 1=y, 2=z, 3=w
      i32 undef)  ; vertex/primitive ID (undef for VS inputs)
  %y = call float @dx.op.loadInput.f32(i32 4, i32 0, i32 0, i8 1, i32 undef)
  %z = call float @dx.op.loadInput.f32(i32 4, i32 0, i32 0, i8 2, i32 undef)
  %w = call float @dx.op.loadInput.f32(i32 4, i32 0, i32 0, i8 3, i32 undef)

  ; Store to SV_Position output (element 0)
  call void @dx.op.storeOutput.f32(
      i32 5,      ; opcode: StoreOutput
      i32 0,      ; output element index
      i32 0,      ; row index
      i8 0,       ; component x
      float %x)
  call void @dx.op.storeOutput.f32(i32 5, i32 0, i32 0, i8 1, float %y)
  call void @dx.op.storeOutput.f32(i32 5, i32 0, i32 0, i8 2, float %z)
  call void @dx.op.storeOutput.f32(i32 5, i32 0, i32 0, i8 3, float %w)
  ret void
}
```

A pixel shader sampling a texture with a sampler:

```llvm
; Pixel shader: float4 main(float2 uv : TEXCOORD) : SV_Target
define void @ps_main() {
entry:
  ; Load interpolated UV inputs
  %u = call float @dx.op.loadInput.f32(i32 4, i32 0, i32 0, i8 0, i32 undef)
  %v = call float @dx.op.loadInput.f32(i32 4, i32 0, i32 0, i8 1, i32 undef)

  ; Create texture handle (SRV slot t0)
  %texH = call %dx.types.Handle @dx.op.createHandle(
      i32 57,    ; CreateHandle
      i8 0,      ; SRV = 0
      i32 0,     ; resource meta-data index
      i32 0,     ; array index (non-array)
      i1 false)  ; non-uniform = false

  ; Create sampler handle (Sampler slot s0)
  %sampH = call %dx.types.Handle @dx.op.createHandle(
      i32 57, i8 3, i32 0, i32 0, i1 false)

  ; Sample the texture
  ; dx.op.sample args: (opcode, tex, samp, coord_x, coord_y, coord_z, coord_w,
  ;                     offset_x, offset_y, offset_z, clamp)
  %samp = call %dx.types.ResRet.f32 @dx.op.sample.f32(
      i32 60,         ; opcode: Sample
      %dx.types.Handle %texH,
      %dx.types.Handle %sampH,
      float %u,       ; s (U coordinate)
      float %v,       ; t (V coordinate)
      float undef,    ; r (unused for 2D)
      float undef,    ; clamp (unused)
      i32 0,          ; offset x
      i32 0,          ; offset y
      i32 undef,      ; offset z
      float undef)    ; LOD clamp

  ; Extract RGBA components
  %r = extractvalue %dx.types.ResRet.f32 %samp, 0
  %g = extractvalue %dx.types.ResRet.f32 %samp, 1
  %b = extractvalue %dx.types.ResRet.f32 %samp, 2
  %a = extractvalue %dx.types.ResRet.f32 %samp, 3

  ; Store to SV_Target (render target output)
  call void @dx.op.storeOutput.f32(i32 5, i32 0, i32 0, i8 0, float %r)
  call void @dx.op.storeOutput.f32(i32 5, i32 0, i32 0, i8 1, float %g)
  call void @dx.op.storeOutput.f32(i32 5, i32 0, i32 0, i8 2, float %b)
  call void @dx.op.storeOutput.f32(i32 5, i32 0, i32 0, i8 3, float %a)
  ret void
}
```

### 105.4.1 The dx.op Calling Convention

DXIL operations are not native LLVM opcodes. They are calls to external functions named `@dx.op.<name>.<type>` where the first argument is always an i32 opcode identifying the operation. This design means DXIL modules can be parsed by any LLVM 3.7-era parser without special-case logic — the operations look like ordinary function calls.

```llvm
; Thread ID retrieval — SV_DispatchThreadID.x
%tid.x = call i32 @dx.op.threadId.i32(i32 93, i32 0)
;                                         ^    ^
;                                    opcode   dimension (0=x,1=y,2=z)

; Flat thread index — SV_GroupIndex
%flat = call i32 @dx.op.flattenedThreadIdInGroup.i32(i32 96)

; Buffer load from RWStructuredBuffer<float4>
%ret = call %dx.types.ResRet.f32 @dx.op.rawBufferLoad.f32(
    i32 139,             ; opcode: RawBufferLoad (SM 6.2+)
    %dx.types.Handle %h, ; resource handle
    i32 %byteOffset,     ; byte offset in buffer
    i32 undef,           ; element offset (unused for raw)
    i8 15,               ; component mask (0b1111 = xyzw)
    i32 4)               ; alignment hint

; Extract component from ResRet
%x = extractvalue %dx.types.ResRet.f32 %ret, 0

; Buffer store
call void @dx.op.rawBufferStore.f32(
    i32 140,
    %dx.types.Handle %h,
    i32 %byteOffset,
    i32 undef,
    float %x, float %y, float %z, float %w,
    i8 15,
    i32 4)
```

The `%dx.types.ResRet.f32` aggregate type holds four values plus a status bit:
```llvm
%dx.types.ResRet.f32 = type { float, float, float, float, i32 }
;                                                          ^ status: 1=accessed valid data
```

### 105.4.2 Selected DXIL Opcode Table

| OpCode | Mnemonic | Description |
|--------|---------|-------------|
| 4 | `loadInput` | Load shader input (interpolated or flat) |
| 5 | `storeOutput` | Store shader output |
| 6 | `FAbs` | IEEE abs(float) |
| 13 | `Sqrt` | IEEE sqrt |
| 21 | `Dot2` | 2-component float dot product |
| 22 | `Dot3` | 3-component float dot product |
| 23 | `Dot4` | 4-component float dot product |
| 31 | `sample` | Texture sample |
| 32 | `sampleBias` | Texture sample with bias |
| 34 | `sampleCmp` | Texture shadow comparison sample |
| 60 | `bufferLoad` | Typed buffer load (SRV) |
| 67 | `bufferStore` | Typed buffer store (UAV) |
| 68 | `bufferLoad` | Buffer load (typed) |
| 93 | `threadId` | SV_DispatchThreadID component |
| 94 | `groupId` | SV_GroupID component |
| 95 | `threadIdInGroup` | SV_GroupThreadID component |
| 96 | `flattenedThreadIdInGroup` | SV_GroupIndex |
| 114 | `waveIsFirstLane` | Wave: is first active lane |
| 117 | `waveReadLaneFirst` | Broadcast first active lane's value |
| 120 | `waveActiveSum` | Wave: reduction sum |
| 139 | `rawBufferLoad` | Raw/structured buffer load (SM 6.2) |
| 140 | `rawBufferStore` | Raw/structured buffer store (SM 6.2) |

### 105.4.3 DXILOpLowering Pass

The [`DXILOpLowering`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/DirectX/DXILOpLowering.cpp) pass walks the module replacing LLVM intrinsic calls and HLSL-specific calls with the corresponding `@dx.op.*` calls. The mapping from LLVM intrinsics to DXIL opcodes is generated from a TableGen description in [`DXIL.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/Target/DirectX/DXIL.td). For example, `@llvm.fabs.f32` becomes `@dx.op.unary.f32(i32 6, ...)` (opcode 6 = FAbs), and `@llvm.sqrt.f32` becomes `@dx.op.unary.f32(i32 24, ...)` (opcode 24 = Sqrt). Standard LLVM `fadd`, `fmul`, etc. instructions are left as-is since they map directly to the DXIL subset of LLVM IR operations.

```tablegen
// DXIL.td (excerpt)
def ThreadId : DXILOp<93, threadId> {
  let Doc = "Reads the thread ID for the given dimension.";
  let arguments = [Int32Ty];  // dimension index
  let result = Int32Ty;
  let stages = [Stages<CS, [all_stages]>];
  let attributes = [Attributes<ReadNone>];
}

def Dot4 : DXILOp<23, dot4> {
  let Doc = "Dot product of 4-element floating-point vectors.";
  let arguments = [FloatTy, FloatTy, FloatTy, FloatTy,
                   FloatTy, FloatTy, FloatTy, FloatTy];
  let result = FloatTy;
  let attributes = [Attributes<ReadNone>];
}
```

The TableGen backend generates C++ tables used by `DXILOpLowering` to perform the replacement. This design means adding a new DXIL operation requires only a TableGen entry, not C++ changes to the pass itself.

---

## 105.5 Resource Handling

### 105.5.1 The DXIL Resource Model

DXIL uses four resource classes corresponding to D3D12 descriptor table concepts:

| Class | Description | Register Prefix |
|-------|-------------|----------------|
| SRV | Shader Resource View: read-only buffers and textures | `t` |
| UAV | Unordered Access View: read-write buffers and textures | `u` |
| CBV | Constant Buffer View: read-only constant data | `b` |
| Sampler | Texture sampler state | `s` |

Each resource binding has a register space, register index, and count. The binding is specified in HLSL via `: register(u0, space1)` syntax and embedded in DXIL as `!dx.resources` metadata.

### 105.5.2 Handle Creation and Usage

In DXIL, all resource accesses go through opaque handles. The handle creation chain:

```llvm
; Step 1: CreateHandle (DXIL 1.0–1.5)
; Arguments: (opcode, resource_class, resource_meta_idx, index, non_uniform_index)
%h = call %dx.types.Handle @dx.op.createHandle(
    i32 57,    ; opcode: CreateHandle
    i8 0,      ; resource class: SRV=0, UAV=1, CBV=2, Sampler=3
    i32 0,     ; resource meta-data index (into !dx.resources)
    i32 %idx,  ; array index
    i1 false)  ; non-uniform index flag

; Step 2: Use handle in buffer load
%ret = call %dx.types.ResRet.f32 @dx.op.bufferLoad.f32(
    i32 68, %dx.types.Handle %h, i32 %element, i32 undef)
```

For SM 6.6+ with bindless resources:
```llvm
; CreateHandleFromBinding (DXIL 1.6)
%h = call %dx.types.Handle @dx.op.createHandleFromBinding(
    i32 218,
    %dx.types.ResBind { i32 0, i32 0, i32 0, i8 1 }, ; binding: regLow=0, regHi=0, space=0, class=UAV
    i32 %dynamic_idx,
    i1 false)

; CreateHandleFromHeap (DXIL 1.6 — bindless)
%h = call %dx.types.Handle @dx.op.createHandleFromHeap(
    i32 219,
    i32 %heapIdx,   ; descriptor heap index
    i1 false,       ; sampler heap (vs CBV/SRV/UAV heap)
    i1 false)       ; non-uniform index
```

### 105.5.2b Annotate Handle (SM 6.6)

SM 6.6 added `AnnotateHandle` to associate a type annotation with a handle created via `CreateHandleFromBinding` or `CreateHandleFromHeap`. This is needed because binding-based handles carry only a register class; the element type is inferred from the annotation:

```llvm
; SM 6.6: annotate handle with resource type info
%annotH = call %dx.types.Handle @dx.op.annotateHandle(
    i32 216,                          ; opcode: AnnotateHandle
    %dx.types.Handle %rawH,           ; handle from CreateHandleFromBinding/Heap
    %dx.types.ResourceProperties {    ; type annotation struct
        i32 4106,  ; ResKind | flags: 4096 = UAV bit, 10 = StructuredBuffer kind
        i32 4})    ; element stride in bytes (sizeof(float) = 4)
```

The `AnnotateHandle` instruction provides the type-checking information that DXIL operations need to generate the correct opcode. Before SM 6.6, this type information was baked into the `CreateHandle` metadata index; after SM 6.6 with bindless, it must be explicitly attached to each handle.

### 105.5.3 Root Signatures

The root signature describes the layout of descriptor tables, root constants, and root descriptors passed to the shader. It is serialized into the DXIL container as the `RTS0` chunk or embedded inline as a string attribute in the HLSL:

```hlsl
#define RS "DescriptorTable(SRV(t0, numDescriptors=1), UAV(u0, numDescriptors=1)), \
             CBV(b0), \
             StaticSampler(s0)"
[RootSignature(RS)]
[numthreads(64,1,1)]
void main(uint3 tid : SV_DispatchThreadID) { /* ... */ }
```

The `DXILResourceBindingAnalysis` pass analyzes all resource uses in the module and validates that they match the declared root signature layout. This analysis feeds the `PSV0` chunk generation in the container.

---

## 105.6 DXIL Validation and the DXC Toolchain

### 105.6.0 PSV0 Chunk: Pipeline State Validation Data

The PSV0 (Pipeline State Validation) chunk contains precomputed data that lets the D3D12 runtime validate pipeline state objects without re-analyzing the DXIL bitcode. It is generated by the `DXILResourceBindingAnalysis` pass and the container writer:

```
PSV0 chunk binary layout:
  PSVRuntimeInfo0 {
    entryStageInfo:    -- depends on shader stage:
      For compute:   numthreads X, Y, Z (each uint32)
      For vertex:    outputPositionPresent flag
      For pixel:     depthOutput, sampleFreq, coverageOutput
      For geometry:  inputPrimitive, outputTopology, maxOutputVerts, instanceCount
    shaderFlags:       -- bitmask of SFI0-style features used
    sigInputElements:  -- count of input signature elements
    sigOutputElements: -- count of output signature elements
  }
  PSVRuntimeInfo1 {    -- SM 6.1+
    shaderStage,       -- shader model stage enum
    usesViewID,        -- uses SV_ViewID
    maxVertexCount,    -- mesh/hull specific
    sigPatchConstOrPrimCount
  }
  SignatureElements[]  -- descriptor for each in/out semantic
  ResourceBindings[]   -- one entry per SRV/UAV/CBV/Sampler
  ViewIDDependencyTable -- SM 6.1: which outputs depend on SV_ViewID
```

The PSV0 chunk allows the D3D12 runtime to validate, for example, that the vertex shader's output signature matches the pixel shader's input signature at pipeline creation time — a validation that occurs before any draw call and is essential for driver performance (avoiding per-draw revalidation).

### 105.6.1 Structural Validation Requirements

The DXIL validator (`dxil-val`) enforces a comprehensive set of structural rules beyond what the LLVM bitcode parser checks. Critical requirements:

**Control flow restrictions:**
- No irreducible control flow (the CFG must be reducible)
- No indirect branches (`indirectbr` instruction)
- No dynamic dispatch (no function pointers used as call targets)
- All loops must be detectable as loops (no goto-based unstructured loops)

**Resource restrictions (SM 6.0–6.3):**
- Resource handles may not be stored to memory (must remain as SSA values)
- Resource arrays may be indexed only with dynamically-uniform values unless the non-uniform flag is set
- No resource handles passed via phi nodes across loop back-edges (lifted in SM 6.6)

**Type restrictions:**
- No 8-bit integer types in SM 6.1 and below
- No double-precision (f64) in most shader stages (compute shaders allow it)
- Vectors restricted to sizes 2, 3, 4 components

### 105.6.2 DXC Toolchain Overview

The standalone `dxc` compiler provides a complete HLSL compilation pipeline:

```
HLSL Source
    │
    ▼
dxc Frontend (fork of Clang, HLSL AST)
    │
    ▼
HLSL CodeGen → LLVM IR (dxc's private IR fork)
    │
    ▼
DXILGeneration Passes (op lowering, metadata)
    │
    ▼
LLVM Passes (optimizations)
    │
    ▼
DXIL Bitcode
    │
    ▼
dxil-val (DXIL Validator)
    │
    ▼ [on Windows: signed by dxil.dll]
DXContainer (DXIL + RDAT + RTS0 + PSV0 chunks)
```

Basic `dxc` invocation:

```bash
# Compile HLSL compute shader to DXIL container
dxc -T cs_6_5 -E main -O3 compute.hlsl -Fo compute.dxo

# With debug info embedded
dxc -T cs_6_5 -E main -Zi -Od compute.hlsl -Fo compute_debug.dxo -Fd compute.pdb

# Emit DXIL text (LLVM IR disassembly)
dxc -T cs_6_5 -E main compute.hlsl -Fc compute_il.txt

# Validate an existing DXIL container
dxc -val compute.dxo
```

### 105.6.3 DXIL Signing

On Windows, `dxil.dll` (part of the Windows SDK) digitally signs DXIL containers. D3D12 drivers verify this signature before accepting shaders from user-space applications. The signing requirement:

- Exists as a security measure preventing injection of arbitrary GPU code
- Is enforced in D3D12's `CreateComputePipelineState` / `CreateGraphicsPipelineState`
- Can be bypassed in development mode via `D3D12_SHADER_DEBUG_FLAG_ALLOW_POLICY_BASED_VALIDATION`
- Does not apply to WARP (software rasterizer) or PIX debugging builds

The in-tree LLVM backend does not produce signed containers; a separate signing step using the Windows SDK is required for production deployment.

---

## 105.7 Shader Model 6.x Features

### 105.7.0 DXR Ray Tracing (SM 6.3 / 6.5)

DirectX Raytracing (DXR) introduced a set of new shader stages and operations in SM 6.3:

| Shader Stage | Role |
|-------------|------|
| `raygeneration` | Launches rays; root of the ray dispatch |
| `intersection` | Custom ray-primitive intersection test |
| `anyhit` | Called on potential intersections (can reject) |
| `closesthit` | Called on confirmed closest intersection |
| `miss` | Called when no geometry is hit |
| `callable` | General-purpose callable shader |

The core DXR DXIL operations:

```llvm
; TraceRay: launch a ray from a raygeneration or closesthit shader
call void @dx.op.traceRay.payload_type(
    i32 157,                        ; opcode: TraceRay
    %dx.types.Handle %accelStruct,  ; top-level acceleration structure
    i32 %rayFlags,                  ; RAY_FLAG_* bitmask
    i32 %instanceMask,
    i32 %hitGroupIndex,
    i32 %missShaderIndex,
    float %originX, float %originY, float %originZ,  ; ray origin
    float %tMin,                    ; parametric t minimum
    float %dirX,   float %dirY,    float %dirZ,      ; ray direction
    float %tMax,                    ; parametric t maximum
    %MyPayload* %payload)           ; in/out payload struct

; DispatchRaysIndex: get the current ray launch index
%raysIdx = call %dx.types.Dimensions @dx.op.dispatchRaysIndex(i32 145)
%x = extractvalue %dx.types.Dimensions %raysIdx, 0

; WorldRayOrigin / WorldRayDirection: in hit/miss shaders
%orig_x = call float @dx.op.worldRayOrigin.f32(i32 149, i32 0)

; ObjectRayOrigin: ray origin in object space (for intersection tests)
%obj_orig_x = call float @dx.op.objectRayOrigin.f32(i32 151, i32 0)

; ReportHit: in intersection shaders, report a hit
call void @dx.op.reportHit(
    i32 158,          ; opcode: ReportHit
    float %t,         ; hit t value
    i32 %hitKind,     ; 0 = front face, 1 = back face, 254/255 user-defined
    %MyAttr* %attr)   ; hit attributes struct
```

SM 6.5 added DXR 1.1 with inline raytracing (no separate shader stages):

```hlsl
// Inline ray tracing (SM 6.5) — within any shader stage
RaytracingAccelerationStructure TLAS : register(t0);

float4 trace_inline(float3 origin, float3 dir) {
    RayQuery<RAY_FLAG_CULL_BACK_FACING_TRIANGLES> q;
    q.TraceRayInline(TLAS, RAY_FLAG_NONE, 0xFF,
                     RayDesc{ origin, 0.001f, dir, 1000.0f });
    q.Proceed();
    if (q.CommittedStatus() == COMMITTED_TRIANGLE_HIT) {
        return float4(q.CommittedTriangleBarycentrics(), 0, 1);
    }
    return float4(0.2f, 0.2f, 0.8f, 1.0f);  // sky color
}
```

### 105.7.1 Wave Intrinsics (SM 6.0)

SM 6.0 introduced wave-level intrinsics — GPU SIMT operations across the warp/wavefront. In HLSL:

```hlsl
// Wave reduction
float sum = WaveActiveSum(myValue);
float prefixSum = WavePrefixSum(myValue);
bool allPositive = WaveActiveAllTrue(myValue > 0.0f);
uint ballot = WaveActiveBallot(myValue > threshold);

// Lane operations
uint laneIdx = WaveGetLaneIndex();
float firstVal = WaveReadLaneFirst(myValue);
float laneVal = WaveReadLaneAt(myValue, targetLane);
```

These lower to DXIL wave opcodes:

```llvm
; WaveGetLaneIndex → dx.op.waveGetLaneIndex
%lane = call i32 @dx.op.waveGetLaneIndex.i32(i32 111)

; WaveActiveSum → dx.op.waveActiveOp
%sum = call float @dx.op.waveActiveOp.f32(
    i32 119,  ; opcode
    float %v,
    i8 0,     ; op: sum=0, product=1, min=2, max=3, ...
    i8 0)     ; sign: unsigned=0, signed=1

; WaveActiveBallot → dx.op.waveActiveBallot
%ballot = call %dx.types.fouri32 @dx.op.waveActiveBallot.i1(
    i32 116, i1 %cond)
; returns 4×i32 bitmask (128-bit wave max)
```

### 105.7.2 Integer Dot Products (SM 6.4)

SM 6.4 added packed integer dot-product operations targeting machine learning inference workloads:

```llvm
; Dot2AddHalf: 2×fp16 dot product with fp32 accumulate
%acc = call float @dx.op.dot2AddHalf(
    i32 162,        ; opcode
    float %acc_in,  ; accumulator
    half %a0, half %a1,  ; first vector
    half %b0, half %b1)  ; second vector

; Dot4AddI8Packed: 4×i8 (packed in i32) dot product
%acc_i = call i32 @dx.op.dot4AddI8Packed(
    i32 163,
    i32 %acc_in,
    i32 %packed_a,  ; 4×i8 packed as i32
    i32 %packed_b)

; Dot4AddU8Packed: unsigned variant
%acc_u = call i32 @dx.op.dot4AddU8Packed(
    i32 164, i32 %acc_in, i32 %packed_a, i32 %packed_b)
```

### 105.7.3 Mesh and Amplification Shaders (SM 6.5)

The mesh shading pipeline replaces the traditional vertex/geometry pipeline with two new shader stages:

```hlsl
// Amplification (task) shader — controls mesh shader dispatch
[numthreads(32, 1, 1)]
void AmplificationMain(uint3 dtid : SV_DispatchThreadID) {
    MyPayload payload;
    payload.visible = ComputeVisibility(dtid.x);
    // Dispatch mesh shader group; payload forwarded to mesh shader
    DispatchMesh(1, 1, 1, payload);
}

// Mesh shader — outputs primitives directly
[outputtopology("triangle")]
[numthreads(128, 1, 1)]
void MeshMain(uint gtid : SV_GroupThreadID,
              uint gid  : SV_GroupID,
              in payload MyPayload pl,
              out vertices VertexOut verts[64],
              out indices uint3 tris[128]) {
    SetMeshOutputCounts(64, 128);
    // Emit vertices and triangles...
}
```

The `DispatchMesh` built-in lowers to `dx.op.dispatchMesh` (opcode 173), which encodes the group counts and payload into the DXIL container for the amplification shader stage.

### 105.7.3b Bindless Resources (SM 6.6)

SM 6.6 introduced bindless resource access through the descriptor heap directly, removing the root signature requirement for SRV/UAV binding:

```hlsl
// SM 6.6 bindless: access any descriptor heap slot at runtime
Texture2D<float4> GetTexture(uint heapIndex) {
    // ResourceDescriptorHeap[N] accesses the CBV/SRV/UAV heap at index N
    return ResourceDescriptorHeap[heapIndex];
}

[numthreads(8,8,1)]
void main(uint3 dtid : SV_DispatchThreadID,
          uint   texIdx : SV_GroupIndex) {
    Texture2D<float4> tex = GetTexture(texIdx);
    float4 color = tex.Sample(SamplerDescriptorHeap[0],
                              float2(dtid.xy) / float2(1920, 1080));
    OutputUAV[dtid.xy] = color;
}
```

In the generated DXIL, `ResourceDescriptorHeap[N]` lowers to `@dx.op.createHandleFromHeap` with `heapIdx = N`. This enables data-driven rendering where material systems pass texture indices via constant buffers, indexing into a bindless descriptor heap containing all scene textures — a key architectural pattern for modern D3D12 engines.

### 105.7.4 Work Graphs (SM 6.8)

SM 6.8 work graphs allow GPU nodes to launch other GPU nodes directly, enabling GPU-driven dispatch:

```hlsl
struct MyRecord { uint value; };

[Shader("node")]
[NodeDispatchGrid(4, 1, 1)]
[numthreads(64, 1, 1)]
void ProducerNode(uint3 tid : SV_DispatchThreadID,
                  [MaxRecords(16)] NodeOutput<MyRecord> Consumer) {
    ThreadNodeOutputRecords<MyRecord> rec = Consumer.GetThreadNodeOutputRecords(1);
    rec.Get().value = tid.x * 2;
    rec.OutputComplete();
}
```

Work graph nodes lower to new DXIL opcodes in the 200+ range for node handle creation, record allocation, and output finalization.

---

## 105.8 Debugging DXIL

### 105.8.1 DXIL Debug Info

DXIL embeds a modified form of LLVM debug info that is serialized alongside the bitcode. With `dxc -Zi -Od`:

- Source line info is preserved through optimization (with `-Od`, optimizations are disabled)
- A separate PDB-like blob is embedded in the container or written to a `.pdb` file (`-Fd`)
- The `ILDB` chunk (DXIL debug blob) contains stripped debug info referencing external source

```bash
# Compile with debug info; keep source embedded
dxc -T cs_6_0 -E main -Zi -Zss shader.hlsl -Fo shader.dxo

# Compile with external PDB
dxc -T cs_6_0 -E main -Zi -Fd shader.pdb shader.hlsl -Fo shader.dxo
```

### 105.8.2 Disassembling DXIL

The `dxc -dumpbin` flag and standalone LLVM tools both work:

```bash
# dxc: dump DXIL disassembly (HLSL-aware)
dxc -dumpbin shader.dxo

# Extract raw bitcode from DXContainer, then disassemble
python3 - <<'PY'
import struct, sys
data = open("shader.dxo","rb").read()
# Find DXIL chunk (offset 12 in chunk list)
# Parse DXContainerHeader + chunks to locate DXIL chunk
# Then skip DxilProgramHeader (20 bytes) to reach LLVM bitcode
PY

# llvm-dis on extracted bitcode
/usr/lib/llvm-22/bin/llvm-dis shader.bc -o shader.ll

# Use llc to emit DXIL from LLVM IR
/usr/lib/llvm-22/bin/llc -march=dxil \
    -mtriple=dxil-ms-dx \
    -filetype=obj \
    -o output.dxo shader.ll
```

### 105.8.2a SFI0 Shader Feature Flags

The `SFI0` chunk encodes a bitmask of D3D12 feature flags required by the shader. This allows the runtime to fail early if the driver does not support the required features:

| Bit | Feature | Shader Model |
|-----|---------|-------------|
| 0 | Doubles (64-bit float) | SM 6.0+ |
| 1 | ComputeShadersPlusRawAndStructuredBuffers | SM 5.0+ |
| 2 | UAVsAtEveryStage | SM 5.1+ |
| 3 | 64UAVs | SM 5.1+ |
| 4 | MinimumPrecision (min16float, min10float) | SM 5.0+ |
| 5 | 11_1_DoubleExtensions | SM 5.0+ |
| 6 | 11_1_ShaderExtensions | SM 5.0+ |
| 7 | Level9ComparisonFiltering | SM 4.x |
| 8 | TiledResources (sparse textures) | SM 5.0+ |
| 9 | StencilRef | SM 5.0+ |
| 10 | InnerCoverage | SM 5.0+ |
| 11 | TypedUAVLoadAdditionalFormats | SM 5.0+ |
| 12 | ROVs (Rasterizer Ordered Views) | SM 5.1+ |
| 13 | ViewportAndRTArrayIndexFromAnyShader | SM 6.1+ |
| 14 | WaveOps | SM 6.0+ |
| 15 | Int64Ops | SM 6.0+ |
| 16 | ViewID | SM 6.1+ |
| 17 | Barycentrics | SM 6.1+ |
| 18 | NativeLowPrecision | SM 6.2+ |
| 19 | ShadingRate | SM 6.4+ |
| 20 | Raytracing Tier 1.1 | SM 6.5+ |
| 21 | SamplerFeedback | SM 6.5+ |
| 22 | AtomicInt64OnTypedResource | SM 6.6+ |
| 23 | AtomicInt64OnGroupShared | SM 6.6+ |
| 24 | DerivativesInMeshAndAmplificationShaders | SM 6.6+ |
| 25 | ResourceDescriptorHeap | SM 6.6+ |
| 26 | SamplerDescriptorHeap | SM 6.6+ |
| 27 | MeshShaders | SM 6.5+ |
| 28 | WritableMSAATextures | SM 6.7+ |

### 105.8.2b Shader Hot Reload and Runtime Recompilation

Modern D3D12 engines often hot-reload shaders during development without restarting the application. The workflow:

1. File watcher detects `.hlsl` change
2. Background thread invokes `dxc` (or the in-tree Clang HLSL pipeline) to recompile
3. New `DXContainer` is loaded via `D3D12Device::CreateComputePipelineState`
4. Existing pipeline state object is destroyed at the end of the current frame
5. Rendering resumes with the new shader

The container hash (`HASH` chunk) enables shader caching: if the hash matches a previously compiled pipeline state, the driver can skip re-JITting the shader to native ISA. This is critical for reducing PSO (Pipeline State Object) creation stalls, which are a common performance problem in D3D12 applications.

### 105.8.3 PIX and RenderDoc

GPU-side shader debugging uses specialized tools:

**PIX** (Microsoft): Captures D3D12 command queues, steps through draw/dispatch calls, views DXIL disassembly alongside register state. Supports hardware-based shader debugging on Xbox and via Windows Debug Layer.

**RenderDoc**: Open-source GPU frame capture tool. Captures D3D12 (and Vulkan) frames, shows DXIL disassembly with source mapping, supports compute shader stepping on some vendor drivers.

Both tools rely on the debug information embedded by `dxc -Zi` to map DXIL instructions back to HLSL source lines.

---

## 105.9 DXIL Library Targets and Linking

### 105.9.1 DXIL Library Shaders

Beyond individual entry-point shaders, DXIL supports **library targets** (`lib_6_x`) that can contain multiple functions compiled into a single DXIL module, with individual functions exposed as exports. This enables callable shaders for DXR and modular shader development:

```hlsl
// shader_library.hlsl — compiled as lib_6_3
export float3 ComputePhong(float3 normal, float3 lightDir, float3 albedo) {
    float NdotL = saturate(dot(normal, lightDir));
    return albedo * NdotL;
}

[shader("closesthit")]
void ClosestHit(inout MyPayload payload, in BuiltInTriangleIntersectionAttributes attr) {
    float3 bary = float3(1 - attr.barycentrics.x - attr.barycentrics.y,
                         attr.barycentrics.x, attr.barycentrics.y);
    float3 normal = interpolate_normal(bary);
    payload.color = float4(ComputePhong(normal, SunDir, Albedo.Sample(S0, UV)), 1);
}
```

```bash
# Compile as DXR library
dxc -T lib_6_3 -exports ClosestHit shader_library.hlsl -Fo shaders.lib

# The resulting DXContainer has no ISGN/OSGN chunks;
# RDAT contains the function export table with callable shader records
```

Library DXIL modules contain an `RDAT` chunk with a `FunctionTable` that lists each exported function with its shader stage, resource dependencies, and payload/attribute types. The D3D12 state object creation API merges multiple DXIL libraries into a ray tracing pipeline state object, resolving cross-library callable shader references at pipeline creation time.

### 105.9.2 Specialization Constants and DXC Defines

HLSL supports preprocessor-driven specialization via `#define` and Clang's `-D` flags, analogous to GLSL specialization constants:

```bash
# Compile with specialization defines
dxc -T cs_6_0 -E main \
    -D WORKGROUP_SIZE=128 \
    -D ENABLE_DOUBLE_PRECISION=1 \
    shader.hlsl -Fo shader_128.dxo

# Compile a variant
dxc -T cs_6_0 -E main \
    -D WORKGROUP_SIZE=64 \
    -D ENABLE_DOUBLE_PRECISION=0 \
    shader.hlsl -Fo shader_64.dxo
```

The in-tree LLVM HLSL frontend handles these via Clang's preprocessor. The resulting DXIL containers are distinct binaries — there is no runtime specialization mechanism in DXIL (unlike SPIR-V's `OpSpecConstant`). Build systems for D3D12 engines typically maintain a shader permutation cache keyed on define combinations.

## 105.10 Invoking the In-Tree Backend via llc

The in-tree LLVM DirectX backend can compile LLVM IR directly to DXIL:

```bash
# Compile HLSL via Clang to LLVM IR
/usr/lib/llvm-22/bin/clang -target dxil-ms-dx \
    -x hlsl \
    -fhlsl-entry=main \
    -hlsl-shader-model=cs_6_0 \
    -emit-llvm -S \
    compute.hlsl -o compute.ll

# Inspect the generated IR before DXIL lowering
/usr/lib/llvm-22/bin/opt -passes="dxil-prepare,dxil-translate-metadata,dxil-op-lower" \
    compute.ll -S -o compute_dxil.ll

# Emit DXIL bitcode container via llc
/usr/lib/llvm-22/bin/llc \
    -mtriple=dxil-ms-dx \
    -filetype=obj \
    compute_dxil.ll \
    -o compute.dxo
```

The `-filetype=obj` output from the DirectX backend is a DXContainer (not an ELF object). The distinction matters for build system integration.

---

## 105.11 HLSL Namespaces, Overloads, and the Clang HLSL Frontend

### 105.11.1 HLSL as a Clang Language Mode

HLSL is supported in Clang as a first-class language mode, triggered by `-x hlsl` or the `.hlsl` file extension. The HLSL language mode enables:

- The `[numthreads]`, `[outputtopology]`, `[shader]` attributes
- HLSL built-in types: `float4`, `int2`, `uint3`, `half4x4`, `RWBuffer<T>`, etc.
- HLSL built-in functions: `dot()`, `cross()`, `normalize()`, `mul()`, `lerp()`, `saturate()`, etc.
- Semantic annotations (`: SV_Position`, `: TEXCOORD0`)
- Register bindings (`: register(t0, space1)`)
- `cbuffer`/`tbuffer` declarations
- The `in`/`out`/`inout` parameter qualifiers for shader signature elements

The Clang HLSL semantic analysis lives in [`clang/lib/Sema/SemaHLSL.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Sema/SemaHLSL.cpp) and [`clang/include/clang/Sema/SemaDirectX.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/SemaDirectX.h). The built-in functions are defined in [`clang/include/clang/Basic/BuiltinsDirectX.inc`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/BuiltinsDirectX.inc) (autogenerated from the HLSL spec) and [`clang/lib/Headers/hlsl/`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/lib/Headers/hlsl/).

### 105.11.2 Vector Type Swizzling

HLSL's vector swizzle notation (e.g., `float4 v; v.xzyw`) is a major source of Clang HLSL complexity. Swizzles are parsed as member access expressions on vector types and lowered to LLVM `shufflevector` instructions:

```hlsl
float4 a = float4(1,2,3,4);
float4 b = a.wzyx;          // reversed
float2 c = a.xy;            // truncation
float4 d = a.xxyy;          // lane duplication

// These lower to:
// b = shufflevector a, undef, <3,2,1,0>
// c = shufflevector a, undef, <0,1>
// d = shufflevector a, undef, <0,0,1,1>
```

Swizzle assignment (`a.yz = float2(5,6)`) lowers to an insert + shuffle combination:
```llvm
; a.yz = float2(5,6)  →  a = { a.x, 5, 6, a.w }
%a_new = shufflevector <4 x float> %a,
                        <4 x float> <float 5.0, float 6.0, float undef, float undef>,
                        <i32 0, i32 4, i32 5, i32 3>
```

### 105.11.3 HLSL Built-in Function Lowering

HLSL built-in functions are lowered in the Clang CodeGen through the HLSL built-in table in `BuiltinsDirectX.inc`. Each entry maps an HLSL function signature to either:
- A standard LLVM intrinsic (e.g., `abs(float x)` → `@llvm.fabs.f32`)
- A `dx.op.*` call directly (e.g., `IsInf(float x)` → `@dx.op.isSpecialFloat.i1(i32 9, float x)`)
- A compound expansion (e.g., `mul(float4x4 A, float4 v)` → four `dx.op.dot4` calls)

The `dot()` built-in demonstrates the element-count dispatch pattern:
```c
// In clang/lib/CodeGen/CGBuiltin.cpp, HLSL section:
// dot(float2, float2) → dx.op.dot2(f32, f32, f32, f32)
// dot(float3, float3) → dx.op.dot3(f32, f32, f32, f32, f32, f32)
// dot(float4, float4) → dx.op.dot4(...)
// dot(int2, int2) → integer dot (uses mul + add, no dx.op equivalent in SM6.3-)
```

The integer dot product operations (`dot4add_i8packed`) introduced in SM 6.4 are exposed as HLSL built-ins only when targeting `cs_6_4` or higher, enforced by the Clang HLSL sema layer through the `AvailableAtShaderModel` attribute on built-in declarations.

## Chapter 105 Summary

- **DXIL** is LLVM 3.7-era bitcode with structural restrictions enabling safe driver-side JIT compilation; the DXContainer wraps it alongside reflection, root signature, and validation metadata chunks.
- The **in-tree DirectX backend** (`llvm/lib/Target/DirectX/`) handles DXIL emission via a sequence of IR-level transformation passes; it is distinct from Microsoft's standalone `dxc` fork.
- **DXIL operations** are encoded as calls to `@dx.op.<name>.<type>` functions with an i32 opcode as the first argument; the `DXILOpLowering` pass converts LLVM intrinsics to these calls using a TableGen-generated mapping table.
- **Resource handles** (`%dx.types.Handle`) are SSA values created via `@dx.op.createHandle` and used only through `@dx.op.*` access operations; SM 6.6 relaxed some handle restrictions to enable bindless access.
- **Shader Model versions** track DXIL versions 1.0–1.8, each adding capabilities: wave intrinsics (6.0), integer dot products (6.4), mesh/amplification shaders (6.5), bindless (6.6), work graphs (6.8).
- **Validation** by `dxil-val` enforces no indirect calls, no recursion, reducible CFG, and resource-access constraints; on Windows, `dxil.dll` signs containers for driver acceptance.
- **Debugging** uses `dxc -Zi` for DXIL debug info, `llvm-dis` for bitcode disassembly, and PIX/RenderDoc for GPU-side debugging.
- Cross-references: [Chapter 51 — Clang as an HLSL Compiler](../part-07-clang-multilang/ch51-clang-as-hlsl-compiler.md) for the Clang HLSL frontend; [Chapter 104 — The SPIR-V Backend](ch104-spirv-backend.md) for the alternative GPU IR path; [Chapter 82 — TableGen Deep Dive](../part-14-backend/ch82-tablegen-deep-dive.md) for understanding DXIL.td generation.


---

@copyright jreuben11
