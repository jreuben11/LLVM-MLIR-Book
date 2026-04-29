# Chapter 157 — PJRT: The Plugin Runtime Interface

*Part XXII — XLA and the OpenXLA Stack*

PJRT (Pretty Just a Runtime — the acronym is self-deprecating) is XLA's device abstraction layer and compilation interface. It provides a stable C API through which ML frameworks (JAX, TensorFlow, PyTorch via OpenXLA) interact with hardware backends without depending on the backends' internal implementation details. A PJRT plugin is a shared library that implements this C API; frameworks load plugins dynamically, enabling new hardware support without recompiling the framework. This chapter examines PJRT's design, its C API surface, the C++ wrapper classes, the IFRT abstraction that sits above PJRT for distributed computation, and how JAX uses PJRT as its sole interface to hardware.

---

## 157.1 PJRT Design Philosophy

Before PJRT, adding a new hardware backend to JAX required modifying JAX itself and linking against the backend's proprietary C++ libraries. PJRT inverts this dependency: the framework defines a stable C API, and backends implement it. The framework loads the backend as a shared library at runtime.

The design goals:

1. **Stability**: the C ABI does not change across XLA releases. Backends compiled against PJRT 1.0 still work with frameworks using PJRT 2.0 (through versioned capability queries).
2. **Pluggability**: any hardware vendor can implement PJRT without upstreaming code to JAX or TensorFlow.
3. **Portability**: PJRT supports local execution, multi-device execution, and remote execution (via IFRT).

The PJRT C API is defined in [`xla/pjrt/c/pjrt_c_api.h`](https://github.com/openxla/xla/blob/main/xla/pjrt/c/pjrt_c_api.h). It uses a struct of function pointers:

```c
// Simplified from pjrt_c_api.h
struct PJRT_Api {
  size_t struct_size;  // versioning
  
  // Client operations
  PJRT_Error* (*PJRT_Client_Create)(PJRT_Client_Create_Args*);
  PJRT_Error* (*PJRT_Client_Destroy)(PJRT_Client_Destroy_Args*);
  PJRT_Error* (*PJRT_Client_PlatformName)(PJRT_Client_PlatformName_Args*);
  PJRT_Error* (*PJRT_Client_Devices)(PJRT_Client_Devices_Args*);
  PJRT_Error* (*PJRT_Client_Compile)(PJRT_Client_Compile_Args*);
  
  // Buffer operations
  PJRT_Error* (*PJRT_Buffer_Destroy)(PJRT_Buffer_Destroy_Args*);
  PJRT_Error* (*PJRT_Buffer_ToHostBuffer)(PJRT_Buffer_ToHostBuffer_Args*);
  PJRT_Error* (*PJRT_Buffer_CopyToDevice)(PJRT_Buffer_CopyToDevice_Args*);
  
  // Executable operations
  PJRT_Error* (*PJRT_LoadedExecutable_Execute)(PJRT_LoadedExecutable_Execute_Args*);
  PJRT_Error* (*PJRT_LoadedExecutable_Destroy)(PJRT_LoadedExecutable_Destroy_Args*);
  
  // ... ~80 more function pointers
};
```

Each function takes a single `Args` struct pointer containing all parameters and output fields. This design ensures the ABI remains stable as parameters are added (new fields go at the end of the struct, and `struct_size` enables the callee to check which fields are valid).

---

## 157.2 PJRT Core Concepts

### 157.2.1 PjRtClient

`PjRtClient` represents a connection to a set of devices. In C++ (the framework-facing wrapper):

```cpp
// From xla/pjrt/pjrt_client.h
class PjRtClient {
 public:
  // Platform info
  virtual absl::string_view platform_name() const = 0;
  virtual int addressable_device_count() const = 0;
  virtual absl::Span<PjRtDevice* const> addressable_devices() const = 0;

  // Compilation: StableHLO MLIR → LoadedExecutable
  virtual absl::StatusOr<std::unique_ptr<PjRtLoadedExecutable>> Compile(
      mlir::ModuleOp module,
      CompileOptions options) = 0;

  // Alternative: compile from serialized StableHLO bytecode
  virtual absl::StatusOr<std::unique_ptr<PjRtLoadedExecutable>> Compile(
      const XlaComputation& computation,
      CompileOptions options) = 0;

  // Buffer management
  virtual absl::StatusOr<std::unique_ptr<PjRtBuffer>> BufferFromHostBuffer(
      const void* data,
      PrimitiveType type,
      absl::Span<int64_t const> dims,
      std::optional<absl::Span<int64_t const>> byte_strides,
      HostBufferSemantics host_buffer_semantics,
      std::function<void()> on_done_with_host_buffer,
      PjRtDevice* device) = 0;
};
```

`CompileOptions` carries sharding specifications, optimization flags, and device assignment for multi-device programs.

### 157.2.2 PjRtDevice

`PjRtDevice` represents a single accelerator (one GPU, one TPU core, one CPU). It provides:

```cpp
class PjRtDevice {
  virtual int id() const = 0;                 // global device ID
  virtual int local_device_ordinal() const;   // index on this host
  virtual absl::string_view device_kind() const;  // "gpu", "cpu", "tpu"
  virtual PjRtClient* client() const;
  virtual absl::StatusOr<std::unique_ptr<PjRtBuffer>> CopyToDevice(
      PjRtBuffer* buffer) const;
};
```

### 157.2.3 PjRtBuffer

`PjRtBuffer` represents a device-resident tensor. Its key operations:

```cpp
class PjRtBuffer {
  // Type information
  virtual const Shape& on_device_shape() const = 0;

  // Transfer to host
  virtual absl::StatusOr<std::shared_ptr<Literal>> ToLiteralSync() const;
  virtual PjRtFuture<> ToLiteral(MutableLiteralBase* literal) const = 0;

  // Copy to another device
  virtual absl::StatusOr<std::unique_ptr<PjRtBuffer>> CopyToDevice(
      PjRtDevice* dst_device) const = 0;

  // Donate the buffer (mark as consumed; enables aliasing)
  virtual absl::StatusOr<std::unique_ptr<PjRtBuffer>> DonateBuffer();

  // Block until ready
  virtual absl::Status BlockHostUntilReady() const = 0;
};
```

The `PjRtFuture<>` return type from `ToLiteral` is a non-blocking future — the host can continue issuing more GPU work before the transfer completes.

### 157.2.4 PjRtLoadedExecutable

`PjRtLoadedExecutable` wraps a compiled program:

```cpp
class PjRtLoadedExecutable {
  // Execute on devices
  // Returns one output buffer per (output, replica, partition)
  virtual absl::StatusOr<std::vector<std::vector<std::unique_ptr<PjRtBuffer>>>>
  Execute(absl::Span<const std::vector<PjRtBuffer*>> argument_handles,
          const ExecuteOptions& options,
          std::optional<std::vector<PjRtFuture<>>>& returned_futures) = 0;

  // Metadata
  virtual absl::Span<const LogicalDeviceIds> addressable_device_logical_ids() const;
  virtual const DeviceAssignment& device_assignment() const;
};
```

The double-vector output `[replica][output]` reflects that a single executable can run on multiple replicas simultaneously.

---

## 157.3 PJRT Plugin Model

### 157.3.1 Plugin Loading

A PJRT plugin is a shared library that exports the function:

```c
// Entry point that every PJRT plugin must export:
const PJRT_Api* GetPjrtApi();
```

Frameworks discover and load plugins via:

```python
# JAX plugin loading
import jax
jax.config.update('jax_platforms', 'my_accelerator')

# Or via environment variable:
# PJRT_PLUGIN_LIBRARY_PATH=/path/to/pjrt_my_accel.so
```

The JAX PJRT plugin registry in `jax/_src/xla_bridge.py` handles plugin registration:

```python
# Register a plugin
jax._src.xla_bridge.register_backend_factory(
    'my_accelerator',
    lambda: jax._src.xla_bridge.make_pjrt_client(
        plugin_path='/path/to/pjrt_my_accel.so'))
```

### 157.3.2 Built-in Plugins

XLA ships several built-in plugins:

| Plugin | Platform | Implementation |
|--------|----------|----------------|
| `cpu` | Host CPU | `xla/pjrt/cpu/cpu_client.cc` |
| `cuda` | NVIDIA GPU | `xla/pjrt/gpu/gpu_client.cc` |
| `rocm` | AMD GPU | `xla/pjrt/gpu/gpu_client.cc` (ROCm path) |
| `tpu` | Google TPU | Proprietary; loadable shared library |
| `mock_tpu` | Testing | `xla/pjrt/mock_pjrt_client.h` |

The GPU plugin (`cuda`/`rocm`) uses `xla::GpuExecutable` internally; the CPU plugin uses `xla::CpuExecutable`.

### 157.3.3 Capability Versioning

PJRT versions capabilities via `PJRT_Api.struct_size` and per-extension structs. A plugin can advertise optional capabilities:

```c
// Check if the plugin supports the "custom call" extension
PJRT_CustomCallExtension* ext = nullptr;
PJRT_Plugin_Attributes_Args attrs_args;
GetPjrtApi()->PJRT_Plugin_Attributes(&attrs_args);
if (attrs_args.custom_call_extension != nullptr) {
  ext = attrs_args.custom_call_extension;
}
```

---

## 157.4 Compilation Flow

The compilation path from JAX to PJRT:

```python
import jax
import jax.numpy as jnp

@jax.jit
def matmul(a, b):
    return a @ b

# Compilation happens here (lazy, on first call):
result = matmul(jnp.ones((4, 4)), jnp.ones((4, 4)))
```

Under the hood:

```
jax.jit(matmul)(a, b)
    │ trace with abstract values
    ▼
jaxpr (JAX's functional IR)
    │ lower_jaxpr_to_stablehlo
    ▼
mlir.ModuleOp (StableHLO)
    │ serialize to bytecode
    ▼
pjrt_client.Compile(mlir_bytecode, CompileOptions)
    │ plugin-specific compilation
    ▼
PjRtLoadedExecutable
    │ cached by jax.jit
```

`CompileOptions` passed from JAX includes:
- `num_replicas`, `num_partitions`: for data/model parallelism
- `device_assignment`: which devices to use
- `argument_layouts`: layouts of input tensors
- `parameter_is_tupled_arguments`: whether inputs are wrapped in a tuple

### 157.4.1 Compilation from MLIR

The primary compilation path uses `mlir::ModuleOp`:

```cpp
// C++ usage
mlir::MLIRContext ctx;
auto module = ...; // StableHLO module

xla::CompileOptions options;
options.executable_build_options.set_num_replicas(1);
options.executable_build_options.set_num_partitions(4);

auto exec = pjrt_client->Compile(module, options).value();
```

### 157.4.2 Compilation from XlaComputation

For backward compatibility with non-MLIR paths:

```cpp
xla::XlaComputation computation = ...; // from XlaBuilder
auto exec = pjrt_client->Compile(computation, options).value();
```

---

## 157.5 Buffer Management

### 157.5.1 Host to Device Transfer

```cpp
// Transfer host data to device
std::vector<float> host_data(256, 1.0f);
auto buffer = pjrt_client->BufferFromHostBuffer(
    host_data.data(),
    xla::F32,
    {16, 16},  // shape
    std::nullopt,  // byte_strides (use default C order)
    PjRtClient::HostBufferSemantics::kImmutableZeroCopy,
    nullptr,  // on_done callback
    pjrt_client->addressable_devices()[0]).value();
```

`HostBufferSemantics` controls lifetime:
- `kImmutableZeroCopy`: the device may zero-copy from the host buffer; don't touch it until `on_done` fires
- `kImmutableOnlyDuringCall`: copy immediately; caller may reuse host buffer after function returns
- `kMutableZeroCopy`: device uses host memory directly (pinned memory path)

### 157.5.2 Device to Host Transfer

```cpp
// Non-blocking transfer
xla::Literal result_literal(xla::ShapeUtil::MakeShape(xla::F32, {16, 16}));
auto future = buffer->ToLiteral(&result_literal);

// ... do other work ...

// Block when we need the result
future.Await();
// result_literal now contains the data
```

### 157.5.3 Cross-Device Copies

```cpp
auto device1_buffer = ...; // buffer on device 1
auto device0 = pjrt_client->addressable_devices()[0];

// Non-blocking cross-device copy
auto device0_buffer = device1_buffer->CopyToDevice(device0).value();
```

---

## 157.6 Execution

### 157.6.1 Single-Device Execution

```cpp
std::vector<PjRtBuffer*> inputs = {a_buffer.get(), b_buffer.get()};
std::vector<std::optional<PjRtFuture<>>> futures;

xla::ExecuteOptions exec_opts;
exec_opts.untuple_result = true;

// Execute: inputs are per-replica, outputs are [replica][output]
auto outputs = exec->Execute(
    {inputs},  // outer vector: one per replica
    exec_opts,
    futures).value();

// Output buffer
auto& result_buf = outputs[0][0];  // replica 0, output 0
```

### 157.6.2 Multi-Device SPMD Execution

For a SPMD-partitioned executable:

```cpp
// 4-way partitioned GEMM
std::vector<std::vector<PjRtBuffer*>> sharded_inputs(4);
for (int i = 0; i < 4; ++i) {
  sharded_inputs[i] = {a_shards[i].get(), b_shards[i].get()};
}

auto outputs = exec->Execute(sharded_inputs, exec_opts, futures).value();
// outputs[0][0] through outputs[3][0] are the 4 output shards
```

---

## 157.7 IFRT: The Distributed Runtime Interface

IFRT (Interfaces for Runtime) is a higher-level abstraction built on top of PJRT, designed for distributed computation across multiple hosts.

### 157.7.1 IFRT Concepts

```
IFRT
├── Client ─────────────── represents all devices across all hosts
├── Device ─────────────── individual accelerator (wraps PjRtDevice)
├── Array ──────────────── sharded array across multiple devices
│   ├── SingleDeviceSharding
│   ├── OpaqueSharding
│   └── ConcreteSharding (explicit per-shard mapping)
└── Executable ─────────── compiled program (wraps PjRtLoadedExecutable)
```

The key addition IFRT makes over PJRT is the **Array** abstraction — a logical tensor whose shards live on different devices, possibly on different hosts:

```cpp
// IFRT Array: a 1024x1024 matrix sharded across 8 GPUs
auto sharding = xla::ifrt::HloSharding::Create(
    devices, xla::HloSharding::Tile(/*tile_assignment=*/...));

auto array = ifrt_client->MakeArrayFromHostBuffer(
    host_data,
    xla::ifrt::DType::F32,
    xla::ifrt::Shape({1024, 1024}),
    /*byte_strides=*/std::nullopt,
    sharding,
    xla::ifrt::Client::HostBufferSemantics::kImmutableOnlyDuringCall,
    on_done_callback).value();
```

### 157.7.2 JAX's Use of IFRT

JAX uses IFRT directly (not PJRT) for `jax.experimental.multihost_utils` and distributed array operations. The `jax.sharding.Sharding` objects in JAX map directly to IFRT shardings.

---

## 157.8 Custom Call Integration with PJRT

User-defined GPU kernels can be registered as custom calls through PJRT's FFI (Foreign Function Interface):

```cpp
// Register a custom CUDA kernel as a PJRT custom call
XLA_FFI_DEFINE_HANDLER(
    MyKernelHandler, MyKernelImpl,
    ffi::Ffi::Bind()
        .Arg<ffi::Buffer<ffi::F32>>()     // input
        .Ret<ffi::Buffer<ffi::F32>>()     // output
        .Attr<int64_t>("block_size")      // constant attribute
);

XLA_FFI_REGISTER_HANDLER(xla::ffi::GetXlaFfiApi(),
                          "my_custom_kernel",
                          "CUDA",
                          MyKernelHandler);
```

From JAX, custom calls are invoked via:

```python
from jax._src.lib.mlir.dialects import hlo

# In a traced function:
result = hlo.CustomCallOp(
    result_types=[output_type],
    operands=[input],
    call_target_name="my_custom_kernel",
    backend_config=b'{"block_size": 256}')
```

---

## 157.9 Performance Profiling via PJRT

PJRT exposes profiling hooks through the `PjRtClient::TracingInterface`:

```cpp
// Start a profiling session
pjrt_client->StartProfiling();

// Execute workload
exec->Execute(...);

// Stop and collect
auto profile = pjrt_client->StopProfiling();
// profile contains kernel timing, memory transfers, etc.
```

For JAX users, this is exposed through `jax.profiler`:

```python
import jax

with jax.profiler.trace("/tmp/profile"):
    result = my_model(inputs)

# Open the trace in TensorBoard or Perfetto
```

---

## Chapter Summary

- PJRT is a stable C API (struct of function pointers) that decouples ML frameworks from hardware backends; backends are loaded as shared libraries via `GetPjrtApi()`.
- Core abstractions: `PjRtClient` (device set + compiler), `PjRtDevice` (one accelerator), `PjRtBuffer` (device tensor), `PjRtLoadedExecutable` (compiled program).
- `PjRtClient::Compile(mlir::ModuleOp, CompileOptions)` is the primary compilation entry point; it accepts StableHLO MLIR directly.
- Buffer semantics (`kImmutableZeroCopy`, `kMutableZeroCopy`) control host/device memory lifetime; `PjRtFuture<>` enables non-blocking transfers.
- Execute returns `[replica][output]` buffers; multi-device SPMD execution passes per-partition inputs.
- IFRT adds a distributed Array abstraction on top of PJRT, enabling logical sharded arrays across hosts; JAX's sharding model maps directly to IFRT.
- Custom calls are registered through XLA's FFI API and invokable from traced JAX/MLIR code.
- `HostBufferSemantics`, capability versioning via `struct_size`, and the function pointer ABI together ensure PJRT plugins remain compatible across XLA versions.


---

@copyright jreuben11
