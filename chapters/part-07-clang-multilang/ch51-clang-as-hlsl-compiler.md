# Chapter 51 — Clang as an HLSL Compiler

*Part VII — Clang as a Multi-Language Compiler*

Shader programming has its own compiler ecosystem, historically separate from the C/C++ world even though shaders are written in C-like languages. HLSL (High-Level Shader Language) drives the DirectX graphics pipeline and has evolved from a simple vertex/pixel shader language into a full systems-programming language for massively parallel GPU workloads. Microsoft's shift to open-source shader tooling culminated in DirectX Shader Compiler (DXC), which wrapped an aging private LLVM fork; Clang 16 began absorbing that work into the LLVM monorepo and by Clang 22 the integration is deep and production-ready. This chapter dissects every layer of that integration: how the driver accepts HLSL inputs, how Sema handles `cbuffer` blocks and semantic annotations, how CodeGen lowers resource handles to typed LLVM IR, how the DirectX backend transforms that IR into DXIL bitcode, and how the resulting DX Container binary is validated by the runtime.

---

## 51.1 HLSL and the DirectX Compilation Chain

### 51.1.1 Historical Context: FXC, DXC, and Clang

The D3D9/D3D11 era was dominated by **FXC** (the Effect-File Compiler), a closed compiler shipping as `d3dcompiler_47.dll`. FXC compiled HLSL to **DXBC** (DirectX Bytecode), a proprietary instruction format unsuitable for aggressive optimization. When D3D12 and Shader Model 6.0 arrived in 2016, Microsoft introduced **DXC** (`dxcompiler.dll`) backed by a private LLVM 3.7 fork. DXC compiled HLSL to **DXIL** — DirectX Intermediate Language — which is LLVM IR 3.7 bitcode restricted to a specific subset: no opaque pointers, i1 booleans widened to i32, no scalable vectors, no tail calls, and no exceptions. DXC was open-sourced at [github.com/microsoft/DirectXShaderCompiler](https://github.com/microsoft/DirectXShaderCompiler) in 2019.

Maintaining a diverged LLVM fork became untenable. Starting in 2021, Microsoft engineers began upstreaming HLSL support into the LLVM monorepo. Clang 15 gained initial parsing, Clang 16 achieved a working compute shader, and Clang 22 ships a feature-complete implementation covering all shader stages, SM 6.0–6.8 semantics, wave operations, mesh/amplification shaders, and inline raytracing.

### 51.1.2 Shader Models and Stages

The shader model version appears in the target triple's environment component. DXIL targets follow the form:

```
dxil-pc-shadermodel<MAJOR>.<MINOR>-<stage>
```

| Triple environment | HLSL stage | SM introduced |
|---|---|---|
| `vertex` | Vertex shader | SM 1.0 |
| `pixel` | Pixel (fragment) shader | SM 1.0 |
| `geometry` | Geometry shader | SM 4.0 |
| `hull` | Hull (tessellation control) | SM 5.0 |
| `domain` | Domain (tessellation eval) | SM 5.0 |
| `compute` | Compute shader | SM 5.0 |
| `mesh` | Mesh shader | SM 6.5 |
| `amplification` | Amplification (task) shader | SM 6.5 |
| `raygeneration` | Ray generation | SM 6.3 |
| `intersection` | Intersection | SM 6.3 |
| `anyhit` | Any-hit | SM 6.3 |
| `closesthit` | Closest-hit | SM 6.3 |
| `miss` | Miss | SM 6.3 |
| `callable` | Callable | SM 6.3 |
| `library` | Library (linkable) | SM 6.3 |

A library target (`dxil-pc-shadermodel6.6-library`) allows multiple entry points to be compiled and linked before being specialized to a specific stage. This is the common mode for modern game engine pipelines.

### 51.1.3 The `clang-dxc` Driver Mode

When invoked as `clang-dxc` or with `--driver-mode=dxc`, the driver sets `Driver::DXCMode`. The `Driver::DriverMode` enum in [`clang/Driver/Driver.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Driver.h) declares:

```cpp
enum DriverMode {
  GCCMode,
  CLMode,
  FlangMode,
  DXCMode
};
bool IsDXCMode() const { return Mode == DXCMode; }
```

In DXC mode the driver accepts DXC-compatible flags:

| DXC flag | Meaning |
|---|---|
| `-T cs_6_6` | Target profile: stage + shader model |
| `-E CSMain` | Entry point function name |
| `-Fo output.dxil` | Output file |
| `-Zi` | Embed PDB-compatible debug info |
| `-Od` | Disable optimization |
| `-WX` | Treat warnings as errors |
| `-HV 2021` | HLSL language version (2021 features) |
| `-Qstrip_reflect` | Strip reflection metadata |
| `-rootsig-define RS` | Specify root signature macro |

The native Clang-GCC mode uses `-x hlsl --target=dxil-pc-shadermodel6.6-library` with standard `-O2`, `-g`, etc. flags. Both modes converge on the same compilation pipeline after argument translation.

---

## 51.2 HLSL Language Support in Clang

### 51.2.1 File Type and Language Options

The driver types table in [`clang/Driver/Types.def`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Types.def) registers HLSL as:

```
TYPE("hlsl", HLSL, PP_CXX, "hlsl",
     phases::Preprocess, phases::Compile, phases::Backend, phases::Assemble)
```

HLSL is preprocessed like C++ before entering the front end. The `LangOptions.def` entry:

```
LANGOPT(HLSL, 1, 0, NotCompatible, "HLSL")
ENUM_LANGOPT(HLSLVersion, HLSLLangStd, 16, HLSL_Unset, NotCompatible, "HLSL Version")
```

enables `LangOptions::HLSL` and tracks the language standard via the `HLSLLangStd` enum defined in `LangOptions.h`:

```cpp
enum HLSLLangStd {
  HLSL_Unset = 0,
  HLSL_2015 = 2015, HLSL_2016 = 2016, HLSL_2017 = 2017,
  HLSL_2018 = 2018, HLSL_2021 = 2021,
  HLSL_202x = 2028, HLSL_202y = 2029,
};
```

HLSL 2021 is the target for modern DirectX work and enables templates, operator overloading, bitfield qualifiers, and `[WaveSize(N)]`.

### 51.2.2 HLSL-Specific Attribute Family

HLSL attributes are declared as ordinary Clang attributes in `Attr.td` with `LangOpts<"HLSL">` guards. Key attributes:

**`HLSLNumThreadsAttr`** — maps `[numthreads(X,Y,Z)]` on compute, mesh, and amplification entry points. Stored on the `FunctionDecl` and later extracted by CodeGen to populate `dx.entryPoints` metadata.

**`HLSLShaderAttr`** — records the explicit shader stage for a function. Carries a `llvm::Triple::EnvironmentType` value. `SemaHLSL::mergeShaderAttr()` validates that a function is not decorated with conflicting stage annotations.

**`HLSLWaveSizeAttr`** — `[WaveSize(Min, Max, Preferred)]` introduced in SM 6.8. `SemaHLSL::mergeWaveSizeAttr()` enforces `Min <= Preferred <= Max` and that all values are powers of two in `[4, 128]`.

**`HLSLResourceBindingAttr`** — carries the register slot and register space from `register(b0, space1)` syntax. The `clang::hlsl::ResourceBindingAttrs` helper in [`clang/AST/HLSLResource.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/HLSLResource.h) abstracts over the DXIL register binding and the Vulkan-mode `HLSLVkBindingAttr`.

**`HLSLParamModifierAttr`** — distinguishes `in`, `out`, `inout`, and `in out` parameter directions, which differ from C++ reference semantics in that copies are made at call sites.

**`HLSLPayloadAttr`** — marks structs used as ray payload in raytracing shaders.

### 51.2.3 Semantic Annotations

HLSL semantics (`SV_Position`, `SV_DispatchThreadID`, etc.) are syntactic annotations on function parameters and return values that bind shader data to pipeline-defined registers. Clang 22 represents them via `HLSLParsedSemanticAttr` (attached during parsing) which is later resolved to `HLSLAppliedSemanticAttr` during Sema.

The canonical system-value semantics in DXIL:

| Semantic | Stage | Direction |
|---|---|---|
| `SV_Position` | VS/PS/GS/DS | Output from VS, input to PS |
| `SV_DispatchThreadID` | CS | Input: global thread ID |
| `SV_GroupThreadID` | CS | Input: thread ID within group |
| `SV_GroupID` | CS | Input: group ID |
| `SV_GroupIndex` | CS | Input: flat group index |
| `SV_VertexID` | VS | Input: vertex index |
| `SV_InstanceID` | VS | Input: instance index |
| `SV_IsFrontFace` | PS | Input: winding direction |
| `SV_Target[N]` | PS | Output: render target N |
| `SV_Depth` | PS | Output: depth value |
| `SV_PrimitiveID` | GS/HS/DS/PS | Input: primitive index |
| `SV_OutputControlPointID` | HS | Input: control point index |
| `SV_TessFactor` | HS | Output: tessellation factors |
| `SV_BarycentricCoords` | PS | Input: SM 6.1 barycentrics |

### 51.2.4 Builtin Types and the External Sema Source

HLSL's builtin types (`vector<T,N>`, `matrix<T,R,C>`, `RWBuffer<T>`, `Texture2D<T>`, `SamplerState`) are implemented as a lazy class hierarchy in the `HLSLExternalSemaSource`. The class, declared in [`clang/Sema/HLSLExternalSemaSource.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/HLSLExternalSemaSource.h), extends `ExternalSemaSource` and installs itself when `LangOpts.HLSL` is set. Its key methods:

```cpp
class HLSLExternalSemaSource : public ExternalSemaSource {
  void defineTrivialHLSLTypes();
  void defineHLSLVectorAlias();
  void defineHLSLMatrixAlias();
  void defineHLSLTypesWithForwardDeclarations();
  void CompleteType(TagDecl *Tag) override;
};
```

`defineTrivialHLSLTypes()` injects `SamplerState`, `SamplerComparisonState`, and the intangible `__hlsl_resource_t` into the `HLSLNamespace`. `defineHLSLVectorAlias()` creates a template alias `vector<T,N>` mapping to `__attribute__((ext_vector_type(N))) T`. `CompleteType()` is invoked lazily when the compiler needs the full definition of a forward-declared HLSL type, implementing the on-demand completion pattern used throughout Clang's external source infrastructure.

The `HLSLIntangibleTypes.def` file registers the handle base type:

```cpp
HLSL_INTANGIBLE_TYPE(__hlsl_resource_t, HLSLResource, HLSLResourceTy)
```

This single intangible type underlies all resource objects after `HLSLAttributedResourceType` applies its attributes.

The HLSL vector type alias deserves additional attention. HLSL's `float4` is syntactic sugar for `vector<float, 4>`, which in turn aliases to `__attribute__((ext_vector_type(4))) float`. This reuse of Clang's existing OpenCL/GLSL vector extension means vector swizzles (`v.xyzw`, `v.rgba`, `v.xxyy`) work through the same codepath as OpenCL vectors. Swizzle permutations are lowered to `shufflevector` instructions in LLVM IR. The matrix type `matrix<T,R,C>` (aliased as `float4x4`, etc.) is represented as a fixed-size array of row vectors; `SemaHLSL::handleInitialization()` handles the HLSL matrix initialization rules that differ from C-style array initialization.

The `HLSLAttributedResourceType` wraps `__hlsl_resource_t` with a set of resource attributes (class, kind, element type, writable, multisampled) that are resolved during `SemaHLSL::ProcessResourceTypeAttributes()`. This produces a `QualType` distinct for each (class, kind, element) combination, ensuring the type system properly separates `Texture2D<float>` from `Texture2D<float4>`.

---

## 51.3 HLSL Semantic Analysis

### 51.3.1 The `SemaHLSL` Subsystem

HLSL semantic analysis lives in the `SemaHLSL` class (a `SemaBase` subclass) defined in [`clang/Sema/SemaHLSL.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/SemaHLSL.h). It is instantiated as `Sema::HLSL` and called from the main `Sema` dispatch.

**Buffer declarations.** `cbuffer` and `tbuffer` keywords introduce constant and texture buffers. `SemaHLSL::ActOnStartBuffer()` creates a `HLSLBufferDecl` on the current scope:

```cpp
Decl *ActOnStartBuffer(Scope *BufferScope, bool CBuffer,
                       SourceLocation KwLoc,
                       IdentifierInfo *Ident,
                       SourceLocation IdentLoc,
                       SourceLocation LBrace);
void ActOnFinishBuffer(Decl *Dcl, SourceLocation RBrace);
```

Members declared inside a `cbuffer` block are hoisted to global scope at codegen time and share a 16-byte-aligned constant buffer packing layout. `ActOnFinishBuffer()` validates that member types are constant-buffer-compatible and records them on the buffer's `DeclContext`.

**Entry point validation.** `SemaHLSL::CheckEntryPoint()` is called from `ActOnTopLevelFunction()` for functions that bear `HLSLShaderAttr`. It checks:

- That the function signature matches the expected parameter types for the stage.
- That `HLSLNumThreadsAttr` is present on compute/mesh/amplification entries.
- That semantic annotations are valid for the stage via `checkSemanticAnnotation()`.
- That resource parameters are not passed by value.

**Stage mismatch diagnostics.** `diagnoseAttrStageMismatch()` is the workhorse for attribute–stage validity:

```cpp
void diagnoseAttrStageMismatch(
    const Attr *A, llvm::Triple::EnvironmentType Stage,
    std::initializer_list<llvm::Triple::EnvironmentType> AllowedStages);
```

For example, `HLSLNumThreadsAttr` is only valid on `Compute`, `Mesh`, and `Amplification` stages; applying it to a vertex shader triggers a `warn_hlsl_attr_invalid_stage` diagnostic.

### 51.3.2 Resource Binding Analysis

The `ResourceBindings` class in `SemaHLSL.h` tracks all resource declarations across the translation unit. Each `VarDecl` with a resource type gets one or more `DeclBindingInfo` entries (one per `ResourceClass`):

```cpp
struct DeclBindingInfo {
  const VarDecl *Decl;
  ResourceClass ResClass;       // SRV, UAV, CBuffer, Sampler
  const HLSLResourceBindingAttr *Attr;
  BindingType BindType;         // NotAssigned, Explicit, Implicit
};
```

`SemaHLSL::collectResourceBindingsOnVarDecl()` traverses globals and struct fields, recording binding info. `SemaHLSL::ActOnEndOfTranslationUnit()` drives implicit binding assignment: resources without explicit `register()` annotations are assigned the next available slot in their register class using `getNextImplicitBindingOrderID()`. This mirrors DXC's legacy binding model where register slots are assigned in declaration order per class.

### 51.3.3 Root Signature Parsing

The root signature is a D3D12 concept describing the mapping between shader resource bindings and hardware descriptor tables. HLSL 2021 allows encoding root signatures directly in shader source as attribute strings:

```hlsl
[RootSignature("CBV(b0), DescriptorTable(SRV(t0, numDescriptors=8))")]
float4 VSMain(float4 pos : SV_Position) : SV_Position { ... }
```

The string is lexed by `clang::hlsl::RootSignatureLexer` (declared in [`clang/Lex/LexHLSLRootSignature.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Lex/LexHLSLRootSignature.h)) and parsed by `clang::hlsl::RootSignatureParser` from [`clang/Parse/ParseHLSLRootSignature.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Parse/ParseHLSLRootSignature.h). The parser produces a vector of `llvm::hlsl::rootsig::RootElement` variants:

```cpp
using RootElement =
    std::variant<dxbc::RootFlags, RootConstants, RootDescriptor,
                 DescriptorTable, DescriptorTableClause, StaticSampler>;
```

`SemaHLSL::ActOnStartRootSignatureDecl()` deduplicates root signatures by content hash and `ActOnFinishRootSignatureDecl()` attaches the validated `HLSLRootSignatureDecl` to the AST. During CodeGen the root signature elements are serialized into the `RTS0` container part.

The `RootDescriptor` variant captures inline CBV/SRV/UAV root parameters:

```cpp
struct RootDescriptor {
  dxil::ResourceClass Type;   // CBuffer, SRV, or UAV
  Register Reg;                // e.g., { BReg, 0 } for b0
  uint32_t Space;
  dxbc::ShaderVisibility Visibility;
  dxbc::RootDescriptorFlags Flags;
};
```

Flags carry data volatility hints (`DataVolatile`, `DataStaticWhileSetAtExecute`, `DataStatic`) that enable driver optimizations. The default flags depend on the root signature version: version 1.0 uses `DataVolatile` everywhere; version 1.1 and 1.2 default to `DataStaticWhileSetAtExecute` for CBVs and SRVs, allowing the driver to cache descriptor reads.

`StaticSampler` elements embed sampler state directly into the root signature:

```cpp
struct StaticSampler {
  Register Reg;
  dxbc::SamplerFilter Filter;       // e.g., Anisotropic
  dxbc::TextureAddressMode AddressU, AddressV, AddressW; // Wrap, Clamp, etc.
  float MipLODBias;
  uint32_t MaxAnisotropy;
  dxbc::ComparisonFunc CompFunc;
  dxbc::StaticBorderColor BorderColor;
  float MinLOD, MaxLOD;
  uint32_t Space;
  dxbc::ShaderVisibility Visibility;
  dxbc::StaticSamplerFlags Flags;
};
```

Static samplers are more efficient than descriptor-table samplers because they are baked into the root signature and never change, allowing the driver to inline the sampler parameters into shader code.

### 51.3.4 Intrinsic Functions

HLSL intrinsics are exposed as Clang builtins. The `clang/Basic/BuiltinsHLSL.def` table maps names like `__builtin_hlsl_dot`, `__builtin_hlsl_cross`, `__builtin_hlsl_lerp` to overloaded builtin IDs. `SemaHLSL::CheckBuiltinFunctionCall()` validates:

- Vector dimension compatibility for `dot()` and `cross()`.
- That the scalar type is floating-point for transcendental functions.
- That `InterlockedAdd()` operands are integer-typed.
- That wave intrinsics are only called from wave-uniform control flow.

The `SemaDirectX` subsystem (a companion to `SemaHLSL`) handles DirectX-specific builtin checks via `SemaDirectX::CheckDirectXBuiltinFunctionCall()`.

---

## 51.4 HLSL CodeGen

### 51.4.1 `CGHLSLRuntime` Architecture

HLSL-specific code generation is encapsulated in `CGHLSLRuntime` defined in `clang/lib/CodeGen/CGHLSLRuntime.cpp`. It is instantiated on `CodeGenModule` when targeting DXIL or SPIR-V with HLSL input, and exposes hooks called at key points in `CodeGenModule::Release()`. The major responsibilities are:

- Emitting `!dx.resources` metadata for all resource handles.
- Emitting `!dx.entryPoints` metadata for all shader entry points.
- Wrapping entry functions with the DXIL entry-point calling convention.
- Lowering `cbuffer` members to constant buffer loads.

### 51.4.2 Resource Handle Lowering

HLSL resource types (`RWBuffer<float>`, `Texture2D<float4>`, `SamplerState`) are opaque handles in source code. Clang lowers them to LLVM **target extension types** introduced in LLVM 15. For the DXIL target, a `RWBuffer<float>` becomes:

```llvm
target("dx.RawBuffer", float, 1, 0)
```

The four parameters encode: element type, writable flag (1 = UAV), strided flag (0 = typed). A `Texture2D<float4>` becomes:

```llvm
target("dx.Texture", <4 x float>, 0, 0, 0)
; params: element type, writable, multisampled, array
```

A `SamplerState` maps to:

```llvm
target("dx.Sampler", 0)
; params: sampler kind (0 = default)
```

These target extension types are defined in the DirectX target's `TargetExtType` registration and are understood by the DXIL backend's resource lowering passes. They replace the earlier approach of using `addrspace(1)` typed pointers, which was fragile under pointer casting. The type parameters directly correspond to the `dxil::ResourceKind` and `dxil::ElementType` enums from `DXILABI.h`: a `Texture2D<float4>` has kind `Texture2D` and element type `F32` with four components.

Global resource declarations are represented as `@llvm.dx.handle.fromBinding` intrinsic calls:

```llvm
%buf = call target("dx.RawBuffer", float, 1, 0)
    @llvm.dx.handle.fromBinding.tdx.RawBuffer_f32_1_0t(
        i32 0,   ; register space
        i32 0,   ; lower bound
        i32 1,   ; range size
        i32 0,   ; index
        i1 false ; non-uniform index
    )
```

The `@llvm.dx.handle.fromBinding` family of intrinsics is the canonical interface between Clang CodeGen and the DirectX LLVM passes. They are defined in `llvm/include/llvm/IR/IntrinsicsDirectX.td`.

### 51.4.3 Constant Buffer Lowering

`cbuffer` blocks desugar to a collection of global variables that are associated with a constant buffer handle. `CGHLSLRuntime::addBuffer()` creates a `target("dx.CBuffer", ...)` handle global and uses `@llvm.dx.resource.handlefrombinding` to bind it to the declared register. Each member variable becomes a `@llvm.dx.resource.load.cbufferrow` intrinsic call that extracts a 16-byte row from the constant buffer and bitcasts individual fields out of it.

The `llvm::hlsl::CBufferMetadata` class in [`llvm/Frontend/HLSL/CBuffer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Frontend/HLSL/CBuffer.h) records `CBufferMapping` instances in `!hlsl.cbs` metadata:

```cpp
struct CBufferMember {
  GlobalVariable *GV;
  size_t Offset;        // byte offset into 16-byte row
};
struct CBufferMapping {
  GlobalVariable *Handle;
  SmallVector<CBufferMember> Members;
};
```

### 51.4.4 Entry Point Wrapping

DXIL entry functions must conform to a strict signature convention: all inputs and outputs are passed through system-value semantics encoded in the function's parameter list rather than through return values or reference parameters. `CGHLSLRuntime` wraps the user-written entry function in a trampoline that:

1. Reads input semantics from DXIL input intrinsics (`@llvm.dx.load.input.*`).
2. Calls the user function with those values.
3. Writes output semantics via DXIL output intrinsics (`@llvm.dx.store.output.*`).

The wrapping also handles `[numthreads]`-annotated compute entries: the three dimension values are embedded in the `!dx.entryPoints` metadata node so the DirectX runtime can launch the correct thread group geometry.

### 51.4.5 HLSL Built-in Lowering to `dx.op` Calls

HLSL intrinsics (`dot()`, `normalize()`, `WaveActiveSum()`, etc.) lower through two stages. Clang CodeGen emits `@llvm.dx.*` intrinsic calls, which the `DXILOpLowering` LLVM pass later converts to the actual DXIL opcode calls. The two-stage design allows Clang CodeGen to emit target-agnostic LLVM IR while the DXIL backend handles the DXIL-specific encoding. For example, `dot(float4, float4)` lowers through:

```llvm
; After Clang CodeGen — target-independent LLVM intrinsic:
%r = call float @llvm.dx.dot4(float %ax, float %ay, float %az, float %aw,
                              float %bx, float %by, float %bz, float %bw)

; After DXILOpLowering — DXIL opcode call:
%r = call float @dx.op.dot4.f32(i32 <opcode>, float %ax, ...)
```

The first argument to every `@dx.op.*` function is an integer literal encoding the DXIL opcode. The opcode table is generated from `llvm/lib/Target/DirectX/DXIL.td` and the exact values are specified in the [DXIL specification](https://github.com/Microsoft/DirectXShaderCompiler/blob/main/docs/DXIL.rst). This encoding is mandatory for DXIL validation: the GPU driver verifies that all `@dx.op.*` calls use the exact opcode integer literal matching the function name, with the correct overloaded type suffix.

The naming convention for `@dx.op.*` functions encodes the operation name and element type: `dx.op.<name>.<type>` where `<type>` is `f32`, `f16`, `i32`, `i64` etc. Overloaded intrinsics are duplicated per type in the final bitcode, which is why a `Dot4` on `half` vectors produces a different function symbol from a `Dot4` on `float` vectors.

---

## 51.5 The DXIL Target and Backend

### 51.5.1 Target Triple and Machine

The DirectX target resides in `llvm/lib/Target/DirectX/`. The `DXILTargetMachine` is registered for the `dxil-*` architecture and uses a custom `TargetPassConfig` that inserts several mandatory DXIL-specific passes. The target is deliberately simple at the machine level: DXIL does not model hardware register allocation, instruction scheduling, or addressing modes. Instead the "backend" is primarily a sequence of IR-level normalization passes followed by bitcode emission.

### 51.5.2 DXIL Normalization Passes

The `DXILPrepare` pass enforces DXIL's IR constraints:

- **No opaque pointer types**: DXIL was standardized before LLVM opaque pointers. `DXILPrepare` rewrites opaque pointer operations to typed pointer operations where required by validator versions older than DXIL 1.7.
- **i1 → i32 widening**: DXIL has no 1-bit integer type. All `i1` values are zero-extended to `i32` before comparison or storage.
- **No scalable vectors**: DXIL only supports fixed-width vectors up to 4 elements.
- **No musttail calls**: DXIL prohibits tail call optimization.
- **Freeze instruction removal**: DXIL validators do not understand `freeze`; these are replaced with `undef`.

The `DXILTranslateMetadata` pass constructs the DXIL-required module-level metadata nodes:

```llvm
!dx.version = !{!0}          ; DXIL version (e.g., 1.6 for SM 6.6)
!dx.valver  = !{!1}          ; Validator version
!dx.shaderModel = !{!2}      ; "cs", "6", "6"
!dx.typeAnnotations = !{!3}  ; cbuffer member types and offsets
!dx.resources = !{!4}        ; Resource table
!dx.entryPoints = !{!5}      ; Entry function + signature
```

The `dx.typeAnnotations` node encodes full type reflection data used by `ID3D12ShaderReflection` to enumerate cbuffer members at runtime — this is how a game engine discovers uniform variable names and offsets without hardcoding them.

### 51.5.3 `DXILOpLowering`

The `DXILOpLowering` pass replaces all `@llvm.dx.*` intrinsics with `@dx.op.*` calls according to a large opcode table generated from `DXIL.td`. The pass runs after `DXILPrepare` and before bitcode emission. Each DXIL opcode call must:

- Name the function exactly `dx.op.<overload>.<type>` (e.g., `dx.op.dot4.f32`).
- Pass the integer opcode literal as the first argument.
- Use the exact parameter types specified in the DXIL spec.

Violations of these invariants cause the DirectX validator to reject the shader. The LLVM DXIL validator (`llvm-dxil-val`) can be run offline to catch issues before the runtime sees the shader.

### 51.5.4 Resource Table Lowering

The `DXILResourceAccess` and `DXILResourceBindings` passes finalize the resource model. They:

1. Resolve `@llvm.dx.handle.fromBinding` intrinsics by assigning concrete register slots consistent with the `HLSLResourceBindingAttr` data passed down from Clang.
2. Emit the `!dx.resources` metadata node encoding SRV, UAV, cbuffer, and sampler tables.
3. Validate that no two resources share the same register slot within a space (overlapping bindings are rejected by the runtime).

The resource binding infrastructure is modeled by `llvm::hlsl::BindingInfo` in [`llvm/Frontend/HLSL/HLSLBinding.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Frontend/HLSL/HLSLBinding.h), which tracks `BindingSpaces` per `dxil::ResourceClass` and provides `findAvailableBinding()` for implicit register assignment.

---

## 51.6 The DX Container Binary Format

### 51.6.1 Container Layout

The output of DXIL compilation is a **DX Container** (historically called DXBC), a chunked binary format. The layout, defined in [`llvm/BinaryFormat/DXContainer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/BinaryFormat/DXContainer.h), is:

```
dxbc::Header      // Magic "DXBC", SHA-1 hash, file size, part count
uint32_t[]        // Part offsets array
dxbc::PartHeader  // name[4], size
uint8_t[]         // Part data
...               // Repeated for each part
```

The `dxbc::Header` structure:

```cpp
struct Header {
  uint8_t Magic[4];         // "DXBC"
  Hash FileHash;            // SHA-1 over the container content
  ContainerVersion Version; // major.minor
  uint32_t FileSize;
  uint32_t PartCount;
};
```

### 51.6.2 Container Parts

The `DXContainerConstants.def` file enumerates the standard part types:

| Part FourCC | Contents |
|---|---|
| `DXIL` | Raw LLVM bitcode wrapped in `BitcodeHeader` + `ProgramHeader` |
| `PSV0` | Pipeline State Validation data (SM 6.0+) |
| `ISG1` | Input signature (VS inputs, PS inputs) |
| `OSG1` | Output signature (VS outputs, PS outputs) |
| `PSG1` | Patch constant signature (hull/domain shaders) |
| `RTS0` | Serialized root signature |
| `SFI0` | Shader Feature Info flags (64-bit bitmask) |
| `HASH` | Shader hash for content tracking |

The `SFI0` part encodes feature requirements using the `dxbc::FeatureFlags` enum (a 64-bit bitmask defined in `DXContainerConstants.def`). Selected flags from the full table:

| Flag name | Bit | Meaning |
|---|---|---|
| `Doubles` | 0 | Double-precision FP operations |
| `UAVsAtEveryStage` | 2 | UAVs bound at all shader stages |
| `Max64UAVs` | 3 | More than 8 UAV slots |
| `MinimumPrecision` | 4 | `min16float`/`min16int` types |
| `TiledResources` | 8 | Tiled (sparse) resource access |
| `TypedUAVLoadAdditionalFormats` | 11 | UAV loads beyond RGBA32 |
| `ROVs` | 12 | Raster-ordered UAVs |
| `WaveOps` | 14 | SM 6.0 wave intrinsics |
| `Int64Ops` | 15 | 64-bit integer arithmetic |
| `ViewID` | 16 | View instancing |
| `Barycentrics` | 17 | SM 6.1 `SV_Barycentrics` |
| `ShadingRate` | 19 | Variable-rate shading |
| `Raytracing_Tier_1_1` | 20 | Inline raytracing (`RayQuery`) |
| `SamplerFeedback` | 21 | `FeedbackTexture2D` writes |
| `AtomicInt64OnTypedResource` | 22 | 64-bit atomics on typed buffers |
| `AtomicInt64OnGroupShared` | 23 | 64-bit atomics on groupshared |
| `DerivativesInMeshAndAmpShaders` | 24 | ddx/ddy in mesh shaders |
| `ResourceDescriptorHeapIndexing` | 25 | Bindless SRV/UAV heap (SM 6.6) |
| `SamplerDescriptorHeapIndexing` | 26 | Bindless sampler heap (SM 6.6) |
| `AtomicInt64OnHeapResource` | 28 | 64-bit atomics on heap (SM 6.6) |
| `AdvancedTextureOps` | 29 | SM 6.7 texture operations |
| `WriteableMSAATextures` | 30 | Writeable MSAA render targets |
| `SampleCmpWithGradientOrBias` | 31 | `SampleCmpGrad`/`SampleCmpBias` |

The GPU driver reads `SFI0` flags to verify hardware capability before dispatching the shader; a flag indicating an unsupported feature causes the PSO creation call to return an error.

### 51.6.3 The DXIL Bitcode Part

Within the `DXIL` part, a `ProgramHeader` precedes the raw LLVM bitcode:

```cpp
struct ProgramHeader {
  uint8_t Version;    // major in high nibble, minor in low nibble
  uint8_t Unused;
  uint16_t ShaderKind; // stage index (0 = PS, 1 = VS, 5 = CS, ...)
  uint32_t Size;       // size in uint32_t words
  BitcodeHeader Bitcode;
};
struct BitcodeHeader {
  uint8_t Magic[4];    // "DXIL"
  uint8_t MinorVersion;
  uint8_t MajorVersion;
  uint16_t Unused;
  uint32_t Offset;     // offset to LLVM bitcode from start of header
  uint32_t Size;       // size of LLVM bitcode
};
```

This is the raw LLVM bitcode encoding, not an additional container level. The LLVM bitcode includes the IR module with all `!dx.*` metadata attached.

### 51.6.4 Disassembly and Validation

`llvm-dxil-dis` (built from `llvm/tools/llvm-dxil-dis/`) disassembles a DX Container to LLVM IR textual form:

```bash
llvm-dxil-dis foo.dxil -o -
```

The offline DXIL validator (`dxil-val`, also available as `llvm-dxil-val`) verifies all DXIL constraints and produces the same errors the GPU driver would emit:

```bash
/usr/lib/llvm-22/bin/llvm-dxil-val foo.dxil
```

---

## 51.7 Resource Binding Model

### 51.7.1 Register Spaces and Classes

HLSL's binding model assigns resources to four register classes, each with its own namespace of register indices and zero-based spaces:

| Register class | Letter | HLSL types |
|---|---|---|
| Constant buffer (CBV) | `b` | `cbuffer`, `ConstantBuffer<T>` |
| Shader resource view (SRV) | `t` | `Texture2D`, `Buffer`, `StructuredBuffer`, `ByteAddressBuffer` |
| Unordered access view (UAV) | `u` | `RWBuffer`, `RWTexture2D`, `RWStructuredBuffer`, `AppendStructuredBuffer` |
| Sampler | `s` | `SamplerState`, `SamplerComparisonState` |

The `dxil::ResourceClass` enum in [`llvm/Support/DXILABI.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/DXILABI.h) encodes this:

```cpp
enum class ResourceClass : uint8_t {
  SRV = 0, UAV, CBuffer, Sampler,
};
```

The register space mechanism (` : register(t0, space1)`) enables bindless resource models where arrays of resources span thousands of slots in a logical space without conflicting with other spaces.

### 51.7.2 Resource Kind and Element Type

Within each register class, DXIL further categorizes resources by their **kind** (shape) and **element type**. The `dxil::ResourceKind` enum in `DXILABI.h` covers:

```cpp
enum class ResourceKind : uint32_t {
  Invalid = 0,
  Texture1D,           Texture2D,           Texture2DMS,
  Texture3D,           TextureCube,
  Texture1DArray,      Texture2DArray,      Texture2DMSArray,
  TextureCubeArray,
  TypedBuffer,         RawBuffer,           StructuredBuffer,
  CBuffer,             Sampler,             TBuffer,
  RTAccelerationStructure,
  FeedbackTexture2D,   FeedbackTexture2DArray,
  NumEntries,
};
```

The mapping from HLSL source types to `ResourceKind` is:

| HLSL type | ResourceClass | ResourceKind |
|---|---|---|
| `Texture1D<T>` | SRV | `Texture1D` |
| `Texture2D<T>` | SRV | `Texture2D` |
| `Texture2DMS<T>` | SRV | `Texture2DMS` |
| `Texture3D<T>` | SRV | `Texture3D` |
| `TextureCube<T>` | SRV | `TextureCube` |
| `Buffer<T>` | SRV | `TypedBuffer` |
| `ByteAddressBuffer` | SRV | `RawBuffer` |
| `StructuredBuffer<T>` | SRV | `StructuredBuffer` |
| `RWTexture2D<T>` | UAV | `Texture2D` |
| `RWBuffer<T>` | UAV | `TypedBuffer` |
| `RWByteAddressBuffer` | UAV | `RawBuffer` |
| `RWStructuredBuffer<T>` | UAV | `StructuredBuffer` |
| `AppendStructuredBuffer<T>` | UAV | `StructuredBuffer` |
| `FeedbackTexture2D<T>` | UAV | `FeedbackTexture2D` |
| `RaytracingAccelerationStructure` | SRV | `RTAccelerationStructure` |
| `cbuffer` | CBuffer | `CBuffer` |
| `SamplerState` | Sampler | `Sampler` |

For typed resources (TypedBuffer, Texture2D, etc.) the element type is recorded via `dxil::ElementType`:

```cpp
enum class ElementType : uint32_t {
  Invalid = 0,
  I1,    I16,  U16,  I32,  U32,  I64,  U64,
  F16,   F32,  F64,
  SNormF16, UNormF16, SNormF32, UNormF32, SNormF64, UNormF64,
  PackedS8x32, PackedU8x32,
};
```

The element type together with the resource kind is encoded in the `!dx.resources` metadata and drives `DXILTranslateMetadata`'s emission of the `dx.typeAnnotations` type description.

### 51.7.3 Implicit Binding Assignment

Resources declared without explicit `register()` bindings receive implicit assignments at the end of the translation unit. `SemaHLSL::ActOnEndOfTranslationUnit()` iterates all `DeclBindingInfo` records with `BindType::NotAssigned` and calls `BindingInfo::findAvailableBinding()` to locate a free slot. The order is deterministic: declarations are processed in their implicit order ID (assigned at declaration time via `getNextImplicitBindingOrderID()`).

The `BindingInfo` class maintains a `BindingSpaces` per `ResourceClass`, each holding a sorted list of free slot ranges. `RegisterSpace::findAvailableBinding(int32_t Size)` walks the free range list seeking a contiguous block of `Size` slots (or any slot if `Size == -1` for unbounded arrays):

```cpp
struct RegisterSpace {
  uint32_t Space;
  SmallVector<BindingRange> FreeRanges;  // sorted, non-overlapping
  // Size == -1 means unbounded array
  std::optional<uint32_t> findAvailableBinding(int32_t Size);
};
```

When a resource is assigned a slot, its range is removed from `FreeRanges`, splitting it if necessary. The `BindingInfoBuilder` class accumulates explicit bindings, detects overlaps via `findOverlapping()`, and produces a final `BindingInfo` via `calculateBindingInfo()`. The `BoundRegs` object returned by `takeBoundRegs()` provides the `findBoundReg()` lookup used during validation.

### 51.7.4 Root Signature Binding

The root signature constrains which resource bindings a shader can see at a given draw call. A mismatch between the shader's expected bindings and the root signature causes a D3D12 validation error or a GPU timeout. In Clang 22, root signature parsing produces `llvm::hlsl::rootsig::RootElement` variants that CodeGen serializes into the `RTS0` container part using `llvm::Frontend::HLSL::RootSignatureMetadata` serialization routines in [`llvm/Frontend/HLSL/RootSignatureMetadata.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Frontend/HLSL/RootSignatureMetadata.h).

The `DescriptorTableClause` struct captures individual ranges:

```cpp
struct DescriptorTableClause {
  dxil::ResourceClass Type;
  Register Reg;           // register type + number
  uint32_t NumDescriptors; // count (0xffffffff = unbounded)
  uint32_t Space;
  uint32_t Offset;
  dxbc::DescriptorRangeFlags Flags;
};
```

---

## 51.8 Shader Intrinsics and Wave Operations

### 51.8.1 Wave Intrinsics (SM 6.0+)

Wave (warp/wavefront) operations allow threads within a wave (32 threads on AMD, 32 or 64 on NVIDIA) to communicate without shared memory. HLSL exposes them as builtin functions that lower to DXIL wave opcodes:

| HLSL intrinsic | DXIL `@dx.op.*` suffix | Description |
|---|---|---|
| `WaveGetLaneCount()` | `waveGetLaneCount` | Threads per wave |
| `WaveGetLaneIndex()` | `waveGetLaneIndex` | This thread's lane index |
| `WaveIsFirstLane()` | `waveIsFirstLane` | True for lowest active lane |
| `WaveActiveAnyTrue(cond)` | `waveAnyTrue` | OR reduction |
| `WaveActiveAllTrue(cond)` | `waveAllTrue` | AND reduction |
| `WaveActiveBallot(cond)` | `waveActiveBallot` | 4×uint ballot bitmask |
| `WaveActiveSum(val)` | `waveActiveOp` (sum) | Arithmetic sum |
| `WaveActiveProduct(val)` | `waveActiveOp` (product) | Arithmetic product |
| `WaveActiveMax(val)` | `waveActiveOp` (max) | Maximum |
| `WavePrefixSum(val)` | `wavePrefixOp` (sum) | Exclusive prefix sum |
| `WaveReadLaneAt(val, lane)` | `waveReadLaneAt` | Broadcast from lane |
| `WaveReadLaneFirst(val)` | `waveReadLaneFirst` | Broadcast from first active lane |
| `WaveMatch(val)` | `waveMatch` | SM 6.5 ballot within equal values |
| `WaveMultiPrefixSum(val, mask)` | `waveMultiPrefixOp` (sum) | SM 6.5 multi-prefix |

The DXIL opcode integer values for each `@dx.op.*` call are defined in the DXIL specification and generated from `llvm/lib/Target/DirectX/DXIL.td`; they are not exposed in the installed header hierarchy. Wave operations require that the calling shader was compiled with the `WaveOps` feature flag, which is set automatically when the compiler detects wave intrinsic usage.

### 51.8.2 Quad Operations (PS, SM 6.0+)

Pixel shaders execute in 2×2 quads to enable screen-space derivative instructions. The `ddx()`/`ddy()` intrinsics request hardware-computed coarse derivatives, while `ddx_fine()`/`ddy_fine()` request per-pixel fine derivatives:

```hlsl
float dFdx  = ddx(uv.x);        // DerivCoarseX — DXIL dx.op.derivCoarseX
float dFdy  = ddy(uv.y);        // DerivCoarseY — DXIL dx.op.derivCoarseY
float dFdxf = ddx_fine(uv.x);   // DerivFineX   — DXIL dx.op.derivFineX
float dFdyf = ddy_fine(uv.y);   // DerivFineY   — DXIL dx.op.derivFineY
```

Coarse derivatives are computed once per 2×2 quad (cheap), fine derivatives are computed per pixel (accurate for varying rates). Quad-communication intrinsics (SM 6.0) enable explicit quad-lane reads:

```hlsl
float v_x  = QuadReadAcrossX(val);        // dx.op.quadReadAcrossX
float v_y  = QuadReadAcrossY(val);        // dx.op.quadReadAcrossY
float v_d  = QuadReadAcrossDiagonal(val); // dx.op.quadReadAcrossDiagonal
float v_at = QuadReadLaneAt(val, idx);    // dx.op.quadReadLaneAt
```

### 51.8.3 Atomic Operations

Atomic operations in HLSL target `RWBuffer` and `RWStructuredBuffer` members:

```hlsl
RWBuffer<int> counter : register(u0);
void increment(uint idx) {
    int original;
    InterlockedAdd(counter[idx], 1, original);
}
```

`InterlockedAdd()` lowers to `@llvm.dx.atomic.binop` during Clang CodeGen and then to `@dx.op.atomicBinOp.i32` during `DXILOpLowering`. SM 6.6 added 64-bit atomics on typed resources (`AtomicInt64OnTypedResource` feature flag).

### 51.8.4 Inline Raytracing (SM 6.5+)

SM 6.5 introduced `RayQuery` objects for inline ray-scene intersection without separate shader tables:

```hlsl
RaytracingAccelerationStructure scene : register(t0);

float4 PSMain(float4 pos : SV_Position) : SV_Target {
    RayQuery<RAY_FLAG_CULL_BACK_FACING_TRIANGLES> q;
    RayDesc ray;
    ray.Origin = float3(pos.xy, 0.0);
    ray.Direction = float3(0, 0, 1);
    ray.TMin = 0.001; ray.TMax = 1000.0;
    q.TraceRayInline(scene, RAY_FLAG_NONE, 0xFF, ray);
    q.Proceed();
    if (q.CommittedStatus() == COMMITTED_TRIANGLE_HIT)
        return float4(q.CommittedRayT(), 0, 0, 1);
    return float4(0, 0, 0, 1);
}
```

`RayQuery` methods lower to the `@dx.op.rayQuery_*` family of DXIL opcode calls; the exact opcode values are defined in the DXIL specification. The `Raytracing_Tier_1_1` feature flag is set automatically in `SFI0`.

---

## 51.9 HLSL and SPIR-V (Vulkan Target)

### 51.9.1 The HLSL → SPIR-V Path

Clang 22 supports a second HLSL output target for Vulkan:

```bash
clang -x hlsl --target=spirv-unknown-vulkan1.3-library \
    -fspv-target-env=vulkan1.3 foo.hlsl -o foo.spv
```

The `spirv-unknown-vulkan*` triple triggers the SPIR-V backend instead of the DirectX backend. This path is used by DXVK, vkd3d-proton, and game engines that compile a single HLSL shader to both DXIL (for D3D12) and SPIR-V (for Vulkan).

The SPIR-V path shares the HLSL front end (same `LangOptions::HLSL`, same `SemaHLSL`, same `HLSLExternalSemaSource`) but diverges at CodeGen. Resource handles lower to SPIR-V `OpTypeImage`/`OpTypeSampledImage` decorations rather than `target("dx.Texture", ...)` types. The `SFI0` and `RTS0` container parts are irrelevant for SPIR-V; instead the binding model uses Vulkan descriptor sets. The `vkd3d-proton` project (D3D12 → Vulkan translation layer used by Steam Proton) consumes DXIL bitcode and translates it to SPIR-V at runtime, relying on `dxil-spirv` for the conversion. DXVK, by contrast, handles D3D9/D3D10/D3D11 shaders compiled to DXBC, which is the older pre-DXIL format.

### 51.9.2 Vulkan-Specific Binding Annotations

Vulkan uses `(set, binding)` pairs rather than DirectX register slots. Clang 22 supports the `[[vk::binding(binding, set)]]` attribute (handled by `SemaHLSL::handleVkBindingAttr()`):

```hlsl
[[vk::binding(0, 0)]]
Texture2D<float4> tex;

[[vk::binding(1, 0)]]
SamplerState samp;
```

The `HLSLVkBindingAttr` in the AST is later used by the SPIR-V backend to emit correct `OpDecorate Binding` and `OpDecorate DescriptorSet` instructions. The `clang::hlsl::ResourceBindingAttrs` helper abstracts over both `HLSLResourceBindingAttr` (DXIL path) and `HLSLVkBindingAttr` (SPIR-V path).

### 51.9.3 Push Constants and Root Constants

Vulkan push constants map to HLSL root constants. `SemaHLSL::handleVkPushConstantAttr()` and `handleVkConstantIdAttr()` handle the `[[vk::push_constant]]` and `[[vk::constant_id(N)]]` annotations. These have no DXIL equivalents; they are guarded to only be valid when targeting SPIR-V.

---

## 51.10 HLSL in the DirectX 12 / Agility SDK Pipeline

### 51.10.1 Runtime Consumption of DXIL

The D3D12 runtime ingests DXIL blobs directly via pipeline state object (PSO) creation:

```cpp
D3D12_COMPUTE_PIPELINE_STATE_DESC pso = {};
pso.CS = { dxilBlob->GetBufferPointer(), dxilBlob->GetBufferSize() };
pso.pRootSignature = rootSig;
device->CreateComputePipelineState(&pso, IID_PPV_ARGS(&computePSO));
```

Before accepting the shader, `d3d12.dll` invokes the DXIL validator (`dxcompiler.dll`'s `IDxcValidator` interface) to verify all DXIL invariants. If validation fails, `CreateComputePipelineState` returns `E_INVALIDARG`. The failure message is available via `IDxcOperationResult::GetErrorBuffer()`.

The DirectX Agility SDK decouples the validator version from the OS-shipped `d3d12.dll`, allowing applications to ship a newer `d3d12core.dll` in their package and access SM 6.7/6.8 features without requiring OS updates.

### 51.10.2 Shader Reflection

`ID3D12ShaderReflection` parses the `DXIL` part's `!dx.typeAnnotations` metadata to expose cbuffer layout, resource bindings, and input/output signatures to the application:

```cpp
Microsoft::WRL::ComPtr<ID3D12ShaderReflection> reflect;
D3DReflect(blob->GetBufferPointer(), blob->GetBufferSize(),
           IID_PPV_ARGS(&reflect));

D3D12_SHADER_DESC shaderDesc;
reflect->GetDesc(&shaderDesc);

for (UINT i = 0; i < shaderDesc.ConstantBuffers; i++) {
    ID3D12ShaderReflectionConstantBuffer *cb = reflect->GetConstantBufferByIndex(i);
    D3D12_SHADER_BUFFER_DESC cbDesc;
    cb->GetDesc(&cbDesc);
    // cbDesc.Name, cbDesc.Size, cbDesc.Variables
}
```

The `!dx.typeAnnotations` metadata produced by `DXILTranslateMetadata` encodes each cbuffer member with its name, type, byte offset, and size — exactly the information needed here.

### 51.10.3 Debug Information

HLSL debug info flows through the standard DWARF/CodeView pipeline already integrated in Clang. When compiled with `-Zi` (DXC mode) or `-g` (GCC mode), Clang emits `DIFile`, `DISubprogram`, and `DILocalVariable` metadata. The DXIL backend preserves this in the bitcode, and PIX (Microsoft's GPU debugger) reads it to provide source-level debugging. The DirectX Shader Compiler flag `-Zs` (source embedding) additionally encodes the raw source text in the `HLSL` container part, enabling source-reconstructing disassembly.

---

## 51.11 Toolchain Integration and Testing

### 51.11.1 Compilation Workflow

A complete compute shader compilation with Clang 22 in GCC-compatible mode:

```bash
# Basic compilation
clang -x hlsl \
  --target=dxil-pc-shadermodel6.6-compute \
  -hlsl-entry CSMain \
  -O2 -o sum.dxil sum.hlsl

# DXC-compatible mode
clang-dxc -T cs_6_6 -E CSMain -O2 -Fo sum.dxil sum.hlsl

# With debug info and validation
clang-dxc -T cs_6_6 -E CSMain -Zi -Qembed_debug -Fo sum.dxil sum.hlsl
/usr/lib/llvm-22/bin/llvm-dxil-val sum.dxil

# Emit IR for inspection
clang-dxc -T cs_6_6 -E CSMain -Fc sum.ll sum.hlsl
/usr/lib/llvm-22/bin/llvm-dxil-dis sum.dxil
```

A minimal compute shader to test the pipeline:

```hlsl
// sum.hlsl
RWBuffer<float> output : register(u0);
cbuffer Params : register(b0) {
    uint count;
    float scale;
}

[numthreads(64, 1, 1)]
void CSMain(uint3 tid : SV_DispatchThreadID) {
    if (tid.x < count)
        output[tid.x] = (float)tid.x * scale;
}
```

The compiled DXIL (after `llvm-dxil-dis`) exposes the resource table, entry metadata, and `@dx.op.*` calls:

```llvm
; !dx.entryPoints
!0 = !{void ()* @CSMain, !"CSMain", !1, !2, !3}
!1 = !{!4, !5}  ; signature
!3 = !{i32 1, i32 64, i32 1, i32 1}  ; numthreads 64,1,1

; Resource binding
@output = external addrspace(1) global %"dx.RawBuffer<float, 1, 0>" 
```

### 51.11.2 Test Infrastructure

HLSL tests reside in two locations in the Clang tree:

- `clang/test/SemaHLSL/` — semantic analysis tests checking diagnostics for invalid HLSL.
- `clang/test/CodeGenHLSL/` — CodeGen tests checking emitted IR patterns.

Tests use `FileCheck` and the standard Clang lit infrastructure:

```llvm
; clang/test/CodeGenHLSL/cbuffer.hlsl
// RUN: %clang_cc1 -finclude-default-header -triple dxil-pc-shadermodel6.3-library \
// RUN:   -fnative-half-type -emit-llvm -o - %s | FileCheck %s

cbuffer CB : register(b0) {
  float a;
  int b;
}

// CHECK: !dx.resources = !{[[RSRC:![0-9]+]]}
// CHECK: [[RSRC]] = !{[[CBS:![0-9]+]]
```

The `%dxil_validation` substitution in `.lit_site.cfg.py` enables tests that invoke `llvm-dxil-val` only when the validator is available. The `%clang_cc1` invocation with `-finclude-default-header` injects the HLSL standard header (defining `float4`, `uint3`, `dot()`, etc.) that is built into the compiler.

Sema tests in `clang/test/SemaHLSL/` exercise diagnostics using `// expected-error` and `// expected-warning` annotations. The suite covers resource binding conflicts, stage-mismatch attribute diagnostics, entry-point parameter validation, and root signature string parse errors. Tests are parameterized over shader model versions using `%clang_cc1 -x hlsl -triple dxil-pc-shadermodel6.3-library` and higher; features introduced in later shader models are tested with both valid (no error) and invalid (stage mismatch) invocations.

### 51.11.3 HLSL 2021 Features

HLSL 2021 (enabled by `-HV 2021` or the `HLSL_2021` enum value) adds:
- **Templates**: `template<typename T> T square(T x) { return x * x; }` — lowered through the standard Clang template instantiation machinery.
- **Operator overloading**: Allows `struct` types to define `operator+`, `operator*`, etc.
- **`[WaveSize(N)]`** attribute for requiring a specific wave lane count (SM 6.8).
- **Bitfield qualifiers** in structs.
- **`select()`** ternary for branchless code.

These features require no special Sema treatment beyond what C++ already provides; the HLSL-specific work is in ensuring that the template instantiations and overload resolution follow HLSL's implicit type conversion rules (notably: HLSL allows integer-to-float narrowing without a cast, unlike C++).

---

## Chapter Summary

- **HLSL compiles to DXIL** — a strict subset of LLVM IR — via a DirectX target triple (`dxil-pc-shadermodel6.6-*`) or to SPIR-V for Vulkan (`spirv-unknown-vulkan1.3-*`).
- **`Driver::DXCMode`** (`IsDXCMode()`) activates DXC-compatible flag processing; the underlying pipeline is the same as the `-x hlsl` GCC-mode path.
- **`SemaHLSL`** owns buffer declarations (`ActOnStartBuffer`/`ActOnFinishBuffer`), entry-point validation (`CheckEntryPoint`), stage-mismatch diagnostics (`diagnoseAttrStageMismatch`), and resource binding analysis (`ResourceBindings`, `DeclBindingInfo`).
- **`HLSLExternalSemaSource`** injects HLSL builtin types (`vector<T,N>`, `matrix<T,R,C>`, texture/sampler types) into the `HLSLNamespace` using lazy on-demand completion.
- **Resource handles** lower to LLVM target extension types (`target("dx.RawBuffer", ...)`) and `@llvm.dx.handle.fromBinding` intrinsic calls, avoiding fragile address space conventions.
- **`CGHLSLRuntime`** builds `!dx.resources` and `!dx.entryPoints` metadata and wraps user entry functions in DXIL-compliant trampolines.
- **The DirectX backend** (`llvm/lib/Target/DirectX/`) applies `DXILPrepare`, `DXILTranslateMetadata`, `DXILOpLowering`, and `DXContainerObjectWriter` to produce a valid DX Container binary.
- **The DX Container** wraps LLVM bitcode in the `DXIL` part alongside `PSV0`, `ISG1`/`OSG1`, `RTS0`, and `SFI0` parts encoding pipeline validation data, root signatures, and feature flags.
- **Wave intrinsics** (`WaveActiveSum`, `WaveGetLaneIndex`, etc.) and quad operations lower to DXIL opcodes 111–130; inline raytracing (`RayQuery`) uses opcodes 178–230 (SM 6.5+).
- **Root signatures** are parsed by `clang::hlsl::RootSignatureParser` into `llvm::hlsl::rootsig::RootElement` variants and serialized to the `RTS0` binary part.
- **HLSL tests** live in `clang/test/SemaHLSL/` and `clang/test/CodeGenHLSL/`, driven by `lit` and `FileCheck`; offline validation uses `llvm-dxil-val`.

---

## Cross-References

- [Chapter 28 — The Clang Driver](../part-05-clang-frontend/ch28-the-clang-driver.md) — `Driver::DXCMode`, argument translation from DXC flags to internal compiler flags.
- [Chapter 39 — CodeGenModule and CodeGenFunction](../part-06-clang-codegen/ch39-codegenmodule-and-codegenfunction.md) — The `CodeGenModule::Release()` hook where `CGHLSLRuntime` finalizes metadata.
- [Chapter 40 — Lowering Statements and Expressions](../part-06-clang-codegen/ch40-lowering-statements-and-expressions.md) — Builtin call lowering infrastructure that `CGHLSLRuntime` extends for HLSL intrinsics.
- [Chapter 104 — The SPIR-V Backend](../part-15-targets/ch104-spirv-backend.md) — How the HLSL front end targets Vulkan via the SPIR-V code generator.
- [Chapter 105 — DXIL and DirectX Shader Compilation](../part-15-targets/ch105-dxil-and-directx-shader-compilation.md) — Deep dive into `DXILPrepare`, `DXILOpLowering`, and the full DirectX backend pass pipeline.

## Reference Links

- [`clang/include/clang/Sema/SemaHLSL.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/SemaHLSL.h) — `SemaHLSL` class, `ResourceBindings`, `DeclBindingInfo`
- [`clang/include/clang/Sema/HLSLExternalSemaSource.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Sema/HLSLExternalSemaSource.h) — Lazy HLSL builtin type completion
- [`clang/include/clang/AST/HLSLResource.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/AST/HLSLResource.h) — `ResourceBindingAttrs` abstraction
- [`clang/include/clang/Basic/LangOptions.def`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Basic/LangOptions.def) — `LANGOPT(HLSL, ...)` and `HLSLLangStd` enum
- [`clang/include/clang/Driver/Driver.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Driver/Driver.h) — `Driver::DXCMode`, `IsDXCMode()`
- [`llvm/include/llvm/Frontend/HLSL/HLSLBinding.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Frontend/HLSL/HLSLBinding.h) — `BindingInfo`, `BindingSpaces`, `findAvailableBinding()`
- [`llvm/include/llvm/Frontend/HLSL/HLSLRootSignature.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Frontend/HLSL/HLSLRootSignature.h) — `RootElement` variant, `DescriptorTableClause`
- [`llvm/include/llvm/Frontend/HLSL/CBuffer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Frontend/HLSL/CBuffer.h) — `CBufferMetadata`, `CBufferMapping`, `CBufferMember`
- [`llvm/include/llvm/Support/DXILABI.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/Support/DXILABI.h) — `dxil::ResourceClass`, `dxil::ResourceKind`, `dxil::ElementType`
- [`llvm/include/llvm/BinaryFormat/DXContainer.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/BinaryFormat/DXContainer.h) — Container layout, `dxbc::Header`, `ProgramHeader`, `BitcodeHeader`
- [`llvm/include/llvm/BinaryFormat/DXContainerConstants.def`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/BinaryFormat/DXContainerConstants.def) — Part type enumeration, feature flags
- [`clang/include/clang/Parse/ParseHLSLRootSignature.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/clang/include/clang/Parse/ParseHLSLRootSignature.h) — `RootSignatureParser` interface
- [DXIL Specification](https://github.com/Microsoft/DirectXShaderCompiler/blob/main/docs/DXIL.rst) — Canonical DXIL format specification
- [DirectX Shader Compiler source](https://github.com/microsoft/DirectXShaderCompiler) — DXC reference implementation
