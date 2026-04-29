# Chapter 143 — SPIR-V Dialect

*Part XX — In-Tree Dialects*

SPIR-V is the intermediate language standardized by Khronos for shader and compute programs targeting Vulkan, OpenCL, and OpenXR. MLIR's `spirv` dialect provides a faithful MLIR-level representation of SPIR-V, enabling the full MLIR infrastructure — pattern rewriting, dialect conversion, passes, and the type system — to be applied to shader and compute kernel compilation. This chapter covers the SPIR-V dialect's type system, operation set, capability model, and the conversion pipeline that transforms GPU-level MLIR into binary SPIR-V for Vulkan compute dispatch.

---

## 143.1 SPIR-V Dialect Overview

### 143.1.1 Design Philosophy

The `spirv` dialect ([`mlir/include/mlir/Dialect/SPIRV/IR/SPIRVOps.td`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/include/mlir/Dialect/SPIRV/IR/SPIRVOps.td)) maps nearly 1:1 to SPIR-V binary instructions. Every SPIR-V instruction in the specification becomes one MLIR op. This design choice means:

- The dialect can serialize/deserialize arbitrary SPIR-V binaries
- Optimization passes on the dialect directly correspond to SPIR-V-level transformations
- The type system mirrors SPIR-V's type system exactly
- Capabilities, extensions, and addressing models are first-class attributes

SPIR-V has two primary execution models in MLIR's context:
- **Vulkan Compute**: `GLCompute` execution model, used for general GPU compute
- **OpenCL Kernel**: `Kernel` execution model, used for OpenCL-compatible code

### 143.1.2 `spirv.module`

```mlir
spirv.module Logical GLSL450 {
  // Capability declarations
  spirv.capability Shader
  spirv.capability Float64

  // Extension declarations
  spirv.extension "SPV_EXT_shader_atomic_float_add"

  // Memory model
  // (Logical = no raw pointer arithmetic; GLSL450 = GLSL memory model)

  // Global variable declarations (interface variables)
  spirv.globalVariable @input_buffer : !spirv.ptr<!spirv.struct<(!spirv.rtarray<f32>)>, StorageBuffer>
      bind(0, 0) {decorations = [#spirv.decoration<NonWritable>]}

  spirv.globalVariable @output_buffer : !spirv.ptr<!spirv.struct<(!spirv.rtarray<f32>)>, StorageBuffer>
      bind(0, 1)

  spirv.globalVariable @builtin_thread_id : !spirv.ptr<vector<3xi32>, Input>
      {built_in = "GlobalInvocationId"}

  // Function definitions
  spirv.func @compute_main() -> () "None" {
    // ...
    spirv.Return
  }

  // Entry point
  spirv.EntryPoint "GLCompute" @compute_main, @builtin_thread_id, @input_buffer
  spirv.ExecutionMode @compute_main "LocalSize", 64, 1, 1
}
```

The addressing model (`Logical`) and memory model (`GLSL450`) are top-level attributes. Logical addressing means no raw pointer arithmetic — pointers only arise from `spirv.Variable` and `spirv.AccessChain`. Physical addressing (`Physical32`, `Physical64`) allows raw pointer manipulation.

---

## 143.2 SPIR-V Types

### 143.2.1 Scalar Types

SPIR-V scalar types are the same as MLIR built-in scalars in most cases, but some are SPIR-V-specific:

```mlir
i32      // integer (signless in MLIR, SPIR-V signs added per op)
f32      // 32-bit float
f16      // 16-bit float
bf16     // bfloat16 (requires SPV_INTEL_bfloat16)
i1       // boolean (SPIR-V OpTypeBool)
```

### 143.2.2 Pointer Types

```mlir
!spirv.ptr<f32, Function>        // pointer to f32 in Function (private) storage
!spirv.ptr<i32, StorageBuffer>   // pointer to i32 in StorageBuffer (SSBO)
!spirv.ptr<f32, Uniform>         // pointer to f32 in Uniform (UBO)
!spirv.ptr<f32, Workgroup>       // pointer to f32 in Workgroup (shared memory)
!spirv.ptr<f32, CrossWorkgroup>  // OpenCL global memory
!spirv.ptr<f32, Input>           // built-in input variable
!spirv.ptr<f32, Output>          // built-in output variable
!spirv.ptr<f32, Private>         // private per-invocation variable
!spirv.ptr<f32, PushConstant>    // Vulkan push constants
```

Storage classes map to GPU memory hierarchy:
- `Function` ≈ registers/stack (per-thread)
- `Workgroup` ≈ shared memory (per-workgroup)
- `StorageBuffer` ≈ device global memory (SSBO, read/write)
- `Uniform` ≈ constant memory (UBO, read-only from shader)

### 143.2.3 Composite Types

```mlir
// Fixed array
!spirv.array<4 x f32>
!spirv.array<4 x f32, stride = 16>  // explicit byte stride for buffer layout

// Runtime array (used for SSBOs with variable length)
!spirv.rtarray<f32>
!spirv.rtarray<f32, stride = 4>

// Struct (for interface blocks)
!spirv.struct<(f32, i32, f64)>
!spirv.struct<(!spirv.rtarray<f32> [0])>   // [0] = member offset decoration

// Matrix and vector (SPIR-V uses built-in vector types)
vector<4xf32>            // vec4 in GLSL terms
vector<4xi32>            // ivec4

// Sampler/Image types
!spirv.sampler
!spirv.image<f32, 2D, NoDepth, NonArrayed, SingleSampled, SampledFormat, RGBA32f>
!spirv.sampled_image<!spirv.image<f32, 2D, ...>>
!spirv.acceleration_structure_nv  // for ray tracing
```

### 143.2.4 Function Type

```mlir
!spirv.func<(i32, f32) -> i32>
```

---

## 143.3 SPIR-V Operations

### 143.3.1 Arithmetic

```mlir
// Integer arithmetic
%add = spirv.IAdd %a, %b : i32
%sub = spirv.ISub %a, %b : i32
%mul = spirv.IMul %a, %b : i32
%div_s = spirv.SDiv %a, %b : i32
%div_u = spirv.UDiv %a, %b : i32
%rem_s = spirv.SRem %a, %b : i32
%mod   = spirv.SMod %a, %b : i32   // floor division remainder

// Float arithmetic
%fadd = spirv.FAdd %a, %b : f32
%fsub = spirv.FSub %a, %b : f32
%fmul = spirv.FMul %a, %b : f32
%fdiv = spirv.FDiv %a, %b : f32
%frem = spirv.FRem %a, %b : f32   // sign from dividend
%fmod = spirv.FMod %a, %b : f32   // sign from divisor
%fneg = spirv.FNegate %a : f32

// Vector dot product
%dot = spirv.Dot %va, %vb : vector<4xf32> -> f32

// Comparison
%lt = spirv.SLessThan %a, %b : i32 -> i1
%flt = spirv.FOrdLessThan %a, %b : f32 -> i1  // ordered (false if NaN)
%flt_u = spirv.FUnordLessThan %a, %b : f32 -> i1  // unordered
```

### 143.3.2 Memory Operations

```mlir
// Variable (allocate on stack / workgroup)
%var = spirv.Variable : !spirv.ptr<f32, Function>

// Load and store (through pointer)
%val = spirv.Load "Function" %ptr : f32
spirv.Store "Workgroup" %ptr, %val : f32

// Load/store with memory access semantics
%val = spirv.Load "StorageBuffer" %ptr {memory_access = #spirv.memory_access<Volatile|Aligned>, alignment = 16 : i32} : f32
spirv.Store "StorageBuffer" %ptr, %val {memory_access = #spirv.memory_access<NonTemporal>} : f32

// Access chain (GEP equivalent)
%elem_ptr = spirv.AccessChain %struct_ptr[%c0, %idx] : !spirv.ptr<!spirv.struct<(!spirv.rtarray<f32>)>, StorageBuffer>, i32, i32 -> !spirv.ptr<f32, StorageBuffer>

// In bounds access chain (asserts no OOB)
%safe_ptr = spirv.InBoundsPtrAccessChain %array_ptr[%idx] : !spirv.ptr<f32, StorageBuffer>, i32 -> !spirv.ptr<f32, StorageBuffer>
```

`spirv.AccessChain` is the SPIR-V equivalent of LLVM's `getelementptr` — it computes an element pointer from a base pointer and a sequence of indices navigating through struct fields and array elements.

### 143.3.3 Control Flow

```mlir
// Unconditional branch
spirv.Branch ^target_block

// Conditional branch
spirv.BranchConditional %cond, ^true_block, ^false_block

// Switch
spirv.Switch %selector : i32 [
  0: ^zero_block,
  1: ^one_block,
  default: ^default_block
]

// Function call
%result = spirv.FunctionCall @helper(%arg0, %arg1) : (i32, f32) -> f32

// Return
spirv.Return          // void function
spirv.ReturnValue %v : f32  // value-returning function

// Kill / unreachable
spirv.Unreachable     // block must not be reached
spirv.Kill            // fragment shader kill (discard)
```

### 143.3.4 GLSL Extended Instruction Set

```mlir
// GLSL math functions via the GLSL.std.450 extended instruction set
%sqrt = spirv.GL.Sqrt %x : f32
%sin  = spirv.GL.Sin  %x : f32
%cos  = spirv.GL.Cos  %x : f32
%pow  = spirv.GL.Pow  %x, %y : f32
%exp  = spirv.GL.Exp  %x : f32
%log  = spirv.GL.Log  %x : f32
%abs  = spirv.GL.FAbs %x : f32
%fma  = spirv.GL.Fma  %a, %b, %c : f32
%mix  = spirv.GL.FMix %a, %b, %t : f32  // LERP
%clamp = spirv.GL.FClamp %x, %lo, %hi : f32
%round = spirv.GL.RoundEven %x : f32
%inv_sqrt = spirv.GL.InverseSqrt %x : f32
%ldexp = spirv.GL.Ldexp %x, %exp : f32
```

### 143.3.5 Atomic Operations

```mlir
// Atomic add to storage buffer
%old = spirv.AtomicIAdd "StorageBuffer" "AcquireRelease" %ptr, %val : !spirv.ptr<i32, StorageBuffer>, i32

// Atomic float add (requires SPV_EXT_shader_atomic_float_add)
%old_f = spirv.EXT.AtomicFAddEXT "StorageBuffer" "AcquireRelease" %ptr, %fval : !spirv.ptr<f32, StorageBuffer>, f32

// Compare and exchange
%old_val = spirv.AtomicCompareExchange "StorageBuffer" "AcquireRelease" "Monotonic"
    %ptr, %value, %comparator : !spirv.ptr<i32, StorageBuffer>, i32

// Atomic min/max
%old = spirv.AtomicSMin "StorageBuffer" "AcquireRelease" %ptr, %val : !spirv.ptr<i32, StorageBuffer>, i32

// Memory barrier
spirv.MemoryBarrier "WorkgroupMemory", "AcquireRelease"

// Control barrier (workgroup synchronization)
spirv.ControlBarrier "Workgroup", "Workgroup", "AcquireRelease|WorkgroupMemory"
```

### 143.3.6 Cooperative Matrix (NVIDIA / KHR)

```mlir
// SPV_NV_cooperative_matrix / SPV_KHR_cooperative_matrix
%A = spirv.NV.CooperativeMatrixLoad %ptr, %stride {layout = ColumnMajor}
    : !spirv.ptr<f16, Workgroup>, i32 -> !spirv.coopmatrix<8x8xf16, Subgroup, MatrixA>

%result = spirv.NV.CooperativeMatrixMulAdd %A, %B, %C
    : !spirv.coopmatrix<8x8xf16, Subgroup, MatrixA>,
      !spirv.coopmatrix<8x8xf16, Subgroup, MatrixB>,
      !spirv.coopmatrix<8x8xf32, Subgroup, MatrixAcc>
      -> !spirv.coopmatrix<8x8xf32, Subgroup, MatrixAcc>
```

---

## 143.4 The GPU-to-SPIR-V Lowering

### 143.4.1 `--convert-gpu-to-spirv`

This is the main pass converting GPU dialect + standard dialects to SPIR-V:

```bash
mlir-opt \
  --gpu-kernel-outlining \
  --convert-gpu-to-spirv \
  input.mlir
```

The conversion performs:
- `gpu.func` → `spirv.func` with `spirv.EntryPoint` + `spirv.ExecutionMode`
- `gpu.thread_id x` → `spirv.GlobalInvocationId` builtin variable access
- `gpu.block_id x` → `spirv.WorkgroupId` builtin
- `gpu.block_dim x` → `spirv.WorkgroupSize` builtin
- `gpu.barrier` → `spirv.ControlBarrier`
- `gpu.shuffle` → `spirv.GroupNonUniform*` instructions
- Function arguments → `spirv.globalVariable` with interface variable annotations

### 143.4.2 Additional Conversion Passes

```bash
# Convert arith dialect to SPIR-V
mlir-opt --convert-arith-to-spirv input.mlir

# Convert func dialect to SPIR-V
mlir-opt --convert-func-to-spirv input.mlir

# Convert memref to SPIR-V (for explicit memory operations)
mlir-opt --convert-memref-to-spirv input.mlir

# Convert vector to SPIR-V
mlir-opt --convert-vector-to-spirv input.mlir
```

These are separate passes because different parts of the program may be converted at different times or with different policies.

### 143.4.3 Interface Variables and Bindings

Vulkan compute shaders access memory through descriptor sets. SPIR-V interface variables connect shader inputs/outputs to Vulkan resource bindings:

```mlir
spirv.globalVariable @ssbo
    : !spirv.ptr<!spirv.struct<(!spirv.rtarray<f32> [0])>, StorageBuffer>
    bind(0, 0)  // descriptor set 0, binding 0
{decorations = []}
```

The `bind(set, binding)` attribute generates SPIR-V `Binding` and `DescriptorSet` decorations. On the MLIR side, the conversion from `memref` function arguments to SPIR-V global variables with bindings is driven by `SPIRVTypeConverter` and the `InterfaceVarABIAttr`:

```mlir
// Source (gpu.func argument annotated with binding info)
gpu.func @kernel(%buf: memref<4096xf32>)
    kernel attributes {
      spirv.interface_var_abi = [#spirv.interface_var_abi<(0, 0), StorageBuffer>]
    } { ... }
```

### 143.4.4 Entry Points and Execution Modes

```mlir
// GLCompute entry point with local size 64×1×1
spirv.EntryPoint "GLCompute" @main, @builtin_id, @input_ssbo, @output_ssbo
spirv.ExecutionMode @main "LocalSize", 64, 1, 1

// Alternative: LocalSizeId (dynamic local size via spec constants)
spirv.ExecutionMode @main "LocalSizeId", %spec_x, %spec_y, %spec_z : i32, i32, i32
```

The execution mode `LocalSize` specifies the workgroup dimensions at SPIR-V level. This must match the `gpu.ExecutionMode` attribute and the Vulkan dispatch parameters.

### 143.4.5 Specialization Constants

```mlir
// Specialization constant (can be overridden at pipeline creation time)
spirv.SpecConstant @wg_size = 64 : i32
%wg_sz = spirv.mlir.referenceof @wg_size : i32

// Composite specialization constant
spirv.SpecConstantComposite @wg_dims (@wg_x, @wg_y, @wg_z) : vector<3xi32>
```

Specialization constants are SPIR-V's equivalent of compile-time parameters that can be overridden when creating a Vulkan pipeline. They enable generating a single SPIR-V binary that is specialized for different tile sizes or data types at pipeline creation time.

---

## 143.5 SPIR-V Capabilities and Extensions

### 143.5.1 Capability System

SPIR-V has a hierarchical capability system where each instruction requires one or more capabilities. The `spirv.module` declares which capabilities it uses:

```mlir
spirv.module Logical GLSL450 {
  spirv.capability Shader          // basic compute/graphics
  spirv.capability Float16         // f16 operations
  spirv.capability Float64         // f64 operations
  spirv.capability Int8            // i8 operations
  spirv.capability Int16           // i16 operations
  spirv.capability StorageBuffer16BitAccess  // f16 in SSBOs
  spirv.capability CooperativeMatrix        // warp-level matmul
  ...
}
```

The MLIR SPIR-V verifier checks that all operations used in a module have their required capabilities declared. This catches compatibility issues before serialization.

### 143.5.2 Target Environment

The `TargetEnvAttr` describes what a target device supports:

```mlir
#spirv.target_env<#spirv.vce<v1.5, [Shader, Float16, CooperativeMatrix],
    [SPV_KHR_cooperative_matrix]>,
    #spirv.resource_limits<
        max_compute_workgroup_size = [1024, 1024, 64],
        subgroup_size = 32>>
```

The conversion passes use the target environment to select appropriate instructions and verify that required capabilities are available. If a conversion would require a capability not in the target env, the pass fails with a meaningful error.

---

## 143.6 Serialization and Deserialization

### 143.6.1 Serializing to Binary SPIR-V

```bash
# Serialize spirv.module to binary SPIR-V
mlir-translate --serialize-spirv input.mlir -o output.spv

# Or inline in pipeline:
mlir-opt --serialize-spirv input.mlir | \
    spirv-val --target-env vulkan1.2  # Khronos validation

# Deserialize binary SPIR-V back to MLIR
mlir-translate --deserialize-spirv output.spv
```

The serializer ([`mlir/lib/Target/SPIRV/Serialization.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/mlir/lib/Target/SPIRV/Serialization.cpp)) walks the `spirv.module` and emits binary SPIR-V words. The binary format is:
1. Magic number (`0x07230203`)
2. Version (e.g., `0x00010500` for SPIR-V 1.5)
3. Generator magic number
4. Bound (maximum ID + 1)
5. Schema (reserved, 0)
6. Instruction stream

Each instruction is encoded as: word count | opcode, followed by operand words.

### 143.6.2 Using with Vulkan

A complete Vulkan compute pipeline using MLIR-generated SPIR-V:

```bash
# Compile MLIR to SPIR-V
mlir-opt \
  --gpu-kernel-outlining \
  --convert-gpu-to-spirv \
  --convert-arith-to-spirv \
  --convert-memref-to-spirv \
  --reconcile-unrealized-casts \
  input.mlir | \
mlir-translate --serialize-spirv -o compute.spv

# Validate
spirv-val compute.spv

# Use in Vulkan application:
# VkShaderModuleCreateInfo.pCode = (uint32_t*)mmap(compute.spv)
# VkComputePipelineCreateInfo.stage.pName = "main"
```

### 143.6.3 `mlir-translate` Integration

The `mlir-translate` tool supports SPIR-V targets directly:

```bash
# MLIR SPIR-V → binary
mlir-translate --mlir-to-spirv input.mlir -o output.spv

# MLIR SPIR-V → SPIR-V text assembly
mlir-translate --mlir-to-spirvasm input.mlir
# Can be disassembled with: spirv-dis output.spv
```

---

## Chapter 143 Summary

- The `spirv` dialect maps 1:1 to SPIR-V binary instructions. Every SPIR-V instruction is one MLIR op. `spirv.module` carries addressing model, memory model, capabilities, and extensions.
- SPIR-V types include scalar types, pointer types with storage classes (`StorageBuffer`, `Workgroup`, `Function`, `Input`, `Output`, `Uniform`), composite types (`spirv.array`, `spirv.rtarray`, `spirv.struct`), and image/sampler types.
- Core operations: `spirv.IAdd`/`spirv.FMul` (arithmetic), `spirv.Load`/`spirv.Store`/`spirv.AccessChain` (memory), `spirv.Branch`/`spirv.BranchConditional`/`spirv.Switch` (control flow), `spirv.GL.*` (GLSL extended instructions), `spirv.AtomicIAdd` and related (atomics), `spirv.ControlBarrier` (workgroup sync).
- `--convert-gpu-to-spirv` is the main conversion: `gpu.func` → SPIR-V entry point, `gpu.thread_id`/`gpu.block_id` → builtin variables, `gpu.barrier` → `spirv.ControlBarrier`, memref arguments → `spirv.globalVariable` with binding annotations.
- The capability system requires each instruction's capabilities to be declared in `spirv.module`. `TargetEnvAttr` describes device support for verification.
- Serialization via `mlir-translate --serialize-spirv` produces binary SPIR-V suitable for Vulkan's `VkShaderModuleCreateInfo`. Deserialization allows round-tripping arbitrary SPIR-V through MLIR.
- Specialization constants (`spirv.SpecConstant`) enable parameterizing compiled SPIR-V at Vulkan pipeline creation time without recompilation.
