# Appendix B — MLIR Dialect Quick Reference

*Quick Reference | MLIR in-tree (LLVM 22.1.x)*

Compact reference for all major in-tree MLIR dialects. For full dialect documentation see the MLIR dialect docs at [mlir.llvm.org](https://mlir.llvm.org/docs/Dialects/) and [Chapter 137 — Core Dialects](../chapters/part-20-in-tree-dialects/ch137-core-dialects.md) through [Chapter 146](../chapters/part-20-in-tree-dialects/ch146-async-openmp-openacc-dlti-emitc.md). All ops follow the MLIR generic assembly syntax: `dialect.op_name(operands) attr-dict : (type-list)`.

---

## B.1 Dialect Index

| Dialect | Namespace | Primary Use | Key Ops |
|---|---|---|---|
| `arith` | `arith` | Integer/float arithmetic | `addi`, `muli`, `cmpi`, `cmpf`, `constant` |
| `affine` | `affine` | Polyhedral loop/memory | `for`, `if`, `apply`, `load`, `store` |
| `linalg` | `linalg` | Structured linear algebra | `generic`, `matmul`, `conv_2d_*`, `fill` |
| `scf` | `scf` | Structured control flow | `for`, `while`, `if`, `forall`, `parallel` |
| `vector` | `vector` | SIMD/vector ops | `transfer_read/write`, `contract`, `broadcast` |
| `memref` | `memref` | Memory reference management | `alloc`, `dealloc`, `subview`, `cast`, `copy` |
| `tensor` | `tensor` | Value-semantic tensors | `extract_slice`, `insert_slice`, `empty`, `pack` |
| `func` | `func` | Function definitions/calls | `func`, `call`, `return` |
| `cf` | `cf` | Low-level control flow | `br`, `cond_br`, `switch` |
| `llvm` | `llvm` | LLVM IR bridge | `mlir_zero`, `alloca`, `getelementptr`, `call` |
| `gpu` | `gpu` | GPU abstractions | `launch`, `barrier`, `block_id`, `thread_id` |
| `spirv` | `spirv` | SPIR-V targeting | `module`, `func`, `EntryPoint`, `variable` |
| `nvgpu` | `nvgpu` | NVIDIA GPU specifics | `warpgroup.mma`, `tma.async.load`, `mbarrier` |
| `rocdl` | `rocdl` | AMD GPU specifics | ROCDL intrinsic wrappers |
| `omp` | `omp` | OpenMP dialect | `parallel`, `wsloop`, `target`, `reduction` |
| `acc` | `acc` | OpenACC dialect | `parallel`, `kernels`, `data`, `loop` |
| `async` | `async` | Async execution | `execute`, `create_group`, `await` |
| `math` | `math` | Math functions | `sqrt`, `exp`, `log`, `sin`, `cos`, `fma` |
| `complex` | `complex` | Complex arithmetic | `add`, `mul`, `abs`, `re`, `im`, `create` |
| `index` | `index` | Index arithmetic | `add`, `mul`, `ceildivs`, `cmpui` |
| `ub` | `ub` | Undefined behavior | `poison` |
| `bufferization` | `bufferization` | One-shot bufferization | `alloc_tensor`, `to_buffer`, `to_tensor` |
| `transform` | `transform` | IR transformation | `sequence`, `apply_patterns`, `tile`, `fuse` |
| `pdl` | `pdl` | Pattern description | `pattern`, `operation`, `type`, `constraint` |
| `quant` | `quant` | Quantization | `uniform_quantize`, `dequantize` |
| `shape` | `shape` | Shape inference | `shape_of`, `broadcast`, `num_elements` |
| `sparse_tensor` | `sparse_tensor` | Sparse tensor compiler | `compress`, `convert`, `foreach`, `reduce` |
| `tosa` | `tosa` | Tensor operator set arch | `add`, `matmul`, `conv2d`, `clamp` |
| `mlprogram` | `ml_program` | ML model description | `func`, `global`, `global_load`, `global_store` |
| `emitc` | `emitc` | Emit C/C++ | `func`, `call`, `expr`, `if`, `for` |
| `dlti` | `dlti` | Data layout/target info | `dl_spec`, `target_system_spec` |
| `irdl` | `irdl` | IR definition language | `dialect`, `operation`, `type`, `attribute` |

---

## B.2 Builtin Types

### Integer and Float

| Type | C++ Constructor | Notes |
|---|---|---|
| `i1` / `iN` | `IntegerType::get(ctx, N)` | Arbitrary-width integer; no signedness in type |
| `si8` / `ui8` | `IntegerType::get(ctx, 8, Signed/Unsigned)` | Explicitly signed/unsigned variants |
| `f16` | `Float16Type::get(ctx)` | IEEE 754 half-precision |
| `bf16` | `BFloat16Type::get(ctx)` | Brain float |
| `f32` | `Float32Type::get(ctx)` | IEEE 754 single |
| `f64` | `Float64Type::get(ctx)` | IEEE 754 double |
| `f80` | `Float80Type::get(ctx)` | x86 extended precision |
| `f128` | `Float128Type::get(ctx)` | IEEE 754 quad |
| `tf32` | `FloatTF32Type::get(ctx)` | TensorFloat-32 (NVIDIA) |
| `f8E4M3FN` / `f8E5M2` | `Float8E4M3FNType::get(ctx)` etc. | 8-bit floats for ML |
| `index` | `IndexType::get(ctx)` | Platform-native integer (pointer-width) |
| `none` | `NoneType::get(ctx)` | Unit type |

### Composite Types

| Type | Syntax | C++ Constructor | Notes |
|---|---|---|---|
| Complex | `complex<f32>` | `ComplexType::get(f32Ty)` | Complex number |
| Tuple | `tuple<i32, f64>` | `TupleType::get(ctx, {i32, f64})` | Heterogeneous tuple |
| Function | `(i32, f32) -> f64` | `FunctionType::get(ctx, ins, outs)` | First-class function type |
| Vector (fixed) | `vector<4xf32>` | `VectorType::get({4}, f32)` | Fixed-size vector |
| Vector (scalable) | `vector<[4]xf32>` | `VectorType::get({4}, f32, {true})` | Scalable (SVE/RVV) |
| Ranked tensor | `tensor<4x8xf32>` | `RankedTensorType::get({4,8}, f32)` | Static shapes; `?` = dynamic |
| Unranked tensor | `tensor<*xf32>` | `UnrankedTensorType::get(f32)` | Unknown rank |
| Ranked memref | `memref<4x8xf32>` | `MemRefType::get({4,8}, f32)` | Buffer; optional layout/space |
| Ranked memref (strided) | `memref<4x8xf32, strided<[8,1], offset: 0>>` | With `StridedLayoutAttr` | Explicit strides |
| Unranked memref | `memref<*xf32>` | `UnrankedMemRefType::get(f32, 0)` | Unknown rank |
| Opaque | `!dialect<"opaque_type">` | `OpaqueType::get(ctx, "dialect", "data")` | Unregistered type |

---

## B.3 `arith` Dialect Operations

| Op | Signature | Semantics |
|---|---|---|
| `arith.constant` | `() → T` | Materialize compile-time constant |
| `arith.addi` | `(i, i) → i` | Integer add (wrapping) |
| `arith.subi` | `(i, i) → i` | Integer subtract |
| `arith.muli` | `(i, i) → i` | Integer multiply |
| `arith.divui` | `(i, i) → i` | Unsigned divide |
| `arith.divsi` | `(i, i) → i` | Signed divide |
| `arith.remui` | `(i, i) → i` | Unsigned remainder |
| `arith.remsi` | `(i, i) → i` | Signed remainder |
| `arith.andi` | `(i, i) → i` | Bitwise AND |
| `arith.ori` | `(i, i) → i` | Bitwise OR |
| `arith.xori` | `(i, i) → i` | Bitwise XOR |
| `arith.shli` | `(i, i) → i` | Shift left |
| `arith.shrui` | `(i, i) → i` | Logical shift right |
| `arith.shrsi` | `(i, i) → i` | Arithmetic shift right |
| `arith.maxui` / `arith.minui` | `(i, i) → i` | Unsigned max/min |
| `arith.maxsi` / `arith.minsi` | `(i, i) → i` | Signed max/min |
| `arith.cmpi` | `(pred, i, i) → i1` | Integer compare; predicates: `eq ne slt sle sgt sge ult ule ugt uge` |
| `arith.addf` | `(f, f) → f` | Float add |
| `arith.subf` | `(f, f) → f` | Float subtract |
| `arith.mulf` | `(f, f) → f` | Float multiply |
| `arith.divf` | `(f, f) → f` | Float divide |
| `arith.remf` | `(f, f) → f` | Float remainder |
| `arith.negf` | `(f) → f` | Float negate |
| `arith.maximumf` / `arith.minimumf` | `(f, f) → f` | Float max/min (NaN-propagating) |
| `arith.maxnumf` / `arith.minnumf` | `(f, f) → f` | Float max/min (NaN-suppressing) |
| `arith.cmpf` | `(pred, f, f) → i1` | Float compare; predicates: `oeq ogt oge olt ole one ord ueq ugt uge ult ule une uno true false` |
| `arith.extui` / `arith.extsi` | `(i<N>) → i<M>` | Zero/sign-extend |
| `arith.trunci` | `(i<N>) → i<M>` | Truncate integer |
| `arith.extf` | `(f<narrow>) → f<wide>` | Float extend |
| `arith.truncf` | `(f<wide>) → f<narrow>` | Float truncate |
| `arith.uitofp` / `arith.sitofp` | `(i) → f` | Integer to float |
| `arith.fptoui` / `arith.fptosi` | `(f) → i` | Float to integer |
| `arith.bitcast` | `(T) → U` | Bit-cast (same width) |
| `arith.index_cast` | `(index/i) → i/index` | Cast between index and integer |
| `arith.index_castui` | `(index) → i` | Unsigned index cast |
| `arith.addui_extended` | `(i, i) → (i, i1)` | Add with overflow flag |

---

## B.4 `affine` Dialect Operations

Affine ops operate over affine maps and sets. Integer bounds/indices must be affinely expressible.

| Op | Syntax | Notes |
|---|---|---|
| `affine.for` | `affine.for %i = lb to ub step S { ... }` | `lb`/`ub` can be affine expressions in surrounding `%dim` symbols |
| `affine.if` | `affine.if #set(%dims)[%syms] { ... } else { ... }` | Polyhedral condition |
| `affine.apply` | `%r = affine.apply #map(%dims)[%syms]` | Apply affine map; returns `index` values |
| `affine.load` | `%v = affine.load %mem[#map(%idx)]` | Affine-indexed memref load |
| `affine.store` | `affine.store %v, %mem[#map(%idx)]` | Affine-indexed memref store |
| `affine.min` | `%r = affine.min #map(%dims)[%syms]` | Min of affine map results |
| `affine.max` | `%r = affine.max #map(%dims)[%syms]` | Max of affine map results |
| `affine.parallel` | `affine.parallel (%i, %j) = (%lb0, %lb1) to (%ub0, %ub1) { }` | Parallel affine loop nest |
| `affine.vector_load` | `%v = affine.vector_load %mem[#map(%idx)]` | Vector load with affine index |
| `affine.vector_store` | `affine.vector_store %v, %mem[#map(%idx)]` | Vector store with affine index |
| `affine.dma_start` | `affine.dma_start %src[...], %dst[...], %tag[...], %len` | DMA initiation |
| `affine.dma_wait` | `affine.dma_wait %tag[...], %len` | DMA completion |
| `affine.yield` | `affine.yield %vals...` | Return values from affine region |
| `affine.prefetch` | `affine.prefetch %mem[#map(...)], (read\|write), ...` | Prefetch with affine index |

**Affine map syntax**: `#map = affine_map<(d0,d1)[s0] -> (d0+s0, d1*2)>`
**Integer set syntax**: `#set = affine_set<(d0)[s0] : (d0 >= 0, s0 - d0 >= 0)>`

---

## B.5 `linalg` Dialect Operations

### Named Ops

| Op | Semantics |
|---|---|
| `linalg.matmul` | `C += A * B` (2D matrix multiply) |
| `linalg.matmul_transpose_a` | `C += A^T * B` |
| `linalg.matmul_transpose_b` | `C += A * B^T` |
| `linalg.batch_matmul` | Batched matrix multiply |
| `linalg.matvec` | Matrix-vector product |
| `linalg.vecmat` | Vector-matrix product |
| `linalg.dot` | Inner product |
| `linalg.fill` | Fill tensor/memref with scalar |
| `linalg.copy` | Element-wise copy |
| `linalg.add` / `linalg.sub` / `linalg.mul` | Element-wise arithmetic |
| `linalg.max` / `linalg.min` | Element-wise max/min |
| `linalg.abs` / `linalg.negf` | Element-wise unary |
| `linalg.exp` / `linalg.log` | Element-wise math |
| `linalg.map` | User-defined element-wise (with region) |
| `linalg.reduce` | Reduction along dimensions (with region) |
| `linalg.transpose` | Permute dimensions |
| `linalg.broadcast` | Broadcast dimensions |
| `linalg.conv_2d_nhwc_hwcf` | 2D convolution (NHWC input, HWCF filter) |
| `linalg.conv_2d_nchw_fchw` | 2D convolution (NCHW input, FCHW filter) |
| `linalg.depthwise_conv_2d_nhwc_hwc` | Depthwise 2D convolution |
| `linalg.pooling_nhwc_max` | Max pooling |
| `linalg.pooling_nhwc_sum` | Sum pooling |
| `linalg.pooling_nhwc_min` | Min pooling |

### `linalg.generic`

```mlir
linalg.generic {
  indexing_maps = [
    affine_map<(m, n, k) -> (m, k)>,   // A
    affine_map<(m, n, k) -> (k, n)>,   // B
    affine_map<(m, n, k) -> (m, n)>    // C
  ],
  iterator_types = ["parallel", "parallel", "reduction"]
} ins(%A, %B : tensor<4x4xf32>, tensor<4x4xf32>)
  outs(%C : tensor<4x4xf32>) {
^bb0(%a: f32, %b: f32, %c: f32):
  %mul = arith.mulf %a, %b : f32
  %add = arith.addf %c, %mul : f32
  linalg.yield %add : f32
} -> tensor<4x4xf32>
```

Iterator type values: `"parallel"`, `"reduction"`, `"window"`.

---

## B.6 `scf` Dialect Operations

| Op | Syntax | Notes |
|---|---|---|
| `scf.for` | `scf.for %i = %lb to %ub step %s iter_args(%a = %init) -> T { ... }` | Loop with carried values; `scf.yield` returns results |
| `scf.while` | `scf.while (%a = %init) : (T) -> T { before_region } do { after_region }` | While loop with condition in before_region |
| `scf.if` | `%r = scf.if %cond -> T { ... } else { ... }` | Conditional; both branches yield same types |
| `scf.parallel` | `scf.parallel (%i) = (%lb) to (%ub) step (%s) init (%red) reducer(...)` | Parallel loop with optional reduction |
| `scf.forall` | `scf.forall (%i) in (%n) shared_outs(%out = %init) -> T { ... }` | Parallel-for with output tensors; replaces `foreach_thread` (LLVM 19+) |
| `scf.reduce` | `scf.reduce(%v) : T { ^bb0(%a: T, %b: T): ... scf.reduce.return %r }` | Reduction combiner for `scf.parallel` |
| `scf.yield` | `scf.yield %vals...` | Yield values from `scf` regions |
| `scf.condition` | `scf.condition(%cond) %vals...` | Condition value for `scf.while` before_region |
| `scf.index_switch` | `scf.index_switch %idx : index { case 0: { ... } default: { ... } }` | Multi-way index branch |
| `scf.execute_region` | `scf.execute_region -> T { ... }` | Region with escape (for canonicalization) |

---

## B.7 `vector` Dialect Operations

| Op | Signature | Semantics |
|---|---|---|
| `vector.transfer_read` | `(memref/tensor, indices, padding, [mask]) → vector` | Generalized n-D load; padding on out-of-bounds |
| `vector.transfer_write` | `(vector, memref/tensor, indices, [mask]) → [tensor]` | Generalized n-D store |
| `vector.load` | `(memref, indices) → vector` | Simple aligned load (no padding) |
| `vector.store` | `(vector, memref, indices)` | Simple aligned store |
| `vector.broadcast` | `(scalar/vector) → vector` | Broadcast to higher rank |
| `vector.extract` | `(vector, static_pos...) → element/subvector` | Extract element or subvector |
| `vector.insert` | `(element/subvector, dest_vector, pos...) → vector` | Insert element or subvector |
| `vector.extract_strided_slice` | `(vector) → vector` | Extract with offsets/sizes/strides |
| `vector.insert_strided_slice` | `(sub, dest) → vector` | Insert into strided slice |
| `vector.shuffle` | `(v1, v2, mask) → vector` | Permute elements from two vectors |
| `vector.splat` | `(scalar) → vector` | Fill vector with scalar |
| `vector.contract` | `(lhs, rhs, acc) → result` | Generalized matrix contraction with indexing maps |
| `vector.outerproduct` | `(v1, v2, [acc]) → matrix` | Outer product |
| `vector.multi_reduction` | `(vector, acc, kind) → result` | Reduction over multiple dims |
| `vector.reduction` | `(kind, vector, [acc]) → scalar` | 1-D reduction; kinds: `add mul min max and or xor minsi maxsi minui maxui minnumf maxnumf minimumf maximumf` |
| `vector.mask` | `(mask, passthru) { vector op } → result` | Apply predication mask to enclosed op |
| `vector.yield` | `(vals) → ()` | Yield from `vector.mask` region |
| `vector.fma` | `(a, b, c) → vector` | Element-wise FMA |
| `vector.scan` | `(kind, vector, initial, inclusive) → (vector, vector)` | Prefix scan |
| `vector.warp_execute_on_lane_0` | `(laneid) { ... }` | Execute region on lane 0; distribute via `vector.yield` |
| `vector.matrix_multiply` | `(lhs, rhs) → result` | Flat 2-D matrix multiply |

---

## B.8 `memref` Dialect Operations

| Op | Signature | Notes |
|---|---|---|
| `memref.alloc` | `([dyn_sizes], [align]) → memref` | Heap allocation |
| `memref.alloca` | `([dyn_sizes], [align]) → memref` | Stack allocation; freed at function exit |
| `memref.alloca_scope` | `() → T { ... }` | Scope for stack allocations |
| `memref.dealloc` | `(memref) → ()` | Heap deallocation |
| `memref.load` | `(memref, indices) → T` | Element load |
| `memref.store` | `(T, memref, indices) → ()` | Element store |
| `memref.subview` | `(memref, offsets, sizes, strides) → memref` | Create a view/slice |
| `memref.cast` | `(memref<static>) → memref<dynamic>` | Rank-preserving cast |
| `memref.reinterpret_cast` | `(memref, offsets, sizes, strides) → memref` | Arbitrary layout reinterpretation |
| `memref.reshape` | `(memref, shape_memref) → memref` | Reshape (dynamic) |
| `memref.expand_shape` | `(memref) → memref` | Expand dims via reassociation |
| `memref.collapse_shape` | `(memref) → memref` | Collapse dims via reassociation |
| `memref.copy` | `(src_memref, dst_memref) → ()` | Memory copy |
| `memref.transpose` | `(memref) → memref` | Permute dims (metadata only) |
| `memref.view` | `(i8_memref, byte_offset, [dyn_sizes]) → memref` | Raw byte view |
| `memref.atomic_rmw` | `(kind, val, memref, indices) → T` | Atomic read-modify-write |
| `memref.atomic_yield` | `(T) → ()` | Yield from atomic region |
| `memref.generic_atomic_rmw` | `(memref, indices) { region } → T` | Custom atomic via region |
| `memref.prefetch` | `(memref, indices, isWrite, locality, isData) → ()` | Prefetch hint |
| `memref.assume_alignment` | `(memref, alignment) → ()` | Assert alignment |

---

## B.9 `func` and `cf` Dialects

### `func` Dialect

| Op | Syntax | Notes |
|---|---|---|
| `func.func` | `func.func @name(%arg: T) -> T { ... }` | Function definition; `private`/`public`/`nested` visibility |
| `func.call` | `%r = func.call @name(%args) : (T) -> T` | Direct call |
| `func.call_indirect` | `%r = func.call_indirect %fn(%args) : (T) -> T` | Indirect call via function value |
| `func.return` | `func.return %vals...` | Return from function |
| `func.constant` | `%f = func.constant @name : (T) -> T` | Take function reference |

Attributes on `func.func`: `sym_visibility`, `arg_attrs`, `res_attrs`, `function_type`, `sym_name`.

### `cf` Dialect

| Op | Syntax | Notes |
|---|---|---|
| `cf.br` | `cf.br ^bb(%vals : T)` | Unconditional branch to block with args |
| `cf.cond_br` | `cf.cond_br %c, ^true(%v), ^false(%v)` | Conditional branch |
| `cf.switch` | `cf.switch %val : i32, [default: ^default, 0: ^bb0, 1: ^bb1]` | Multi-way branch |
| `cf.assert` | `cf.assert %cond, "msg"` | Runtime assertion (debug) |

---

## B.10 `gpu` Dialect Operations

| Op | Notes |
|---|---|
| `gpu.module` | Wraps GPU kernel functions |
| `gpu.func` | GPU kernel function; `gpu.kernel` attr marks entry point |
| `gpu.launch` | Launch GPU kernel from host: `gpu.launch blocks(%bx,%by,%bz) in (%gbx,%gby,%gbz) threads(...)` |
| `gpu.launch_func` | Launch compiled GPU function: `gpu.launch_func @module::@kernel blocks(...) threads(...) args(...)` |
| `gpu.block_id` | `gpu.block_id x/y/z → index` | Block ID in current launch |
| `gpu.thread_id` | `gpu.thread_id x/y/z → index` | Thread ID within block |
| `gpu.grid_dim` | Grid dimension |
| `gpu.block_dim` | Block dimension |
| `gpu.global_id` | Global thread ID |
| `gpu.barrier` | `gpu.barrier` | Synchronize all threads in block |
| `gpu.shuffle` | `(val, offset, width, mode) → (val, active)` | Warp shuffle; modes: `xor down up idx` |
| `gpu.alloc` | `([dyn_sizes], [host_shared]) → memref` | GPU memory allocation |
| `gpu.dealloc` | `(memref) → ()` | GPU memory free |
| `gpu.memcpy` | `(dst, src) → ()` | GPU memory copy |
| `gpu.memset` | `(memref, val) → ()` | GPU memory fill |
| `gpu.wait` | `([async_tokens]) → [async_token]` | Async operation sync |
| `gpu.subgroup_mma_load_matrix` | Load into MMA fragment |
| `gpu.subgroup_mma_store_matrix` | Store MMA fragment |
| `gpu.subgroup_mma_compute` | Subgroup matrix multiply-accumulate |
| `gpu.set_default_device` | Set active GPU device |
| `gpu.binary` | Container for GPU binary code (CUBIN/HSAco/SPIR-V) |
| `gpu.object` | `{props}` — compiled binary for a target |

---

## B.11 Common Lowering Paths

| Source Dialect | Lowering Target | Pass |
|---|---|---|
| `arith` | `llvm` | `convert-arith-to-llvm` |
| `affine` | `scf` + `memref` | `lower-affine` |
| `scf` | `cf` | `convert-scf-to-cf` |
| `cf` | `llvm` | `convert-cf-to-llvm` |
| `linalg` (named) | `linalg.generic` | `linalg-generalize-named-ops` |
| `linalg.generic` | `affine`/`scf`/`memref` | `convert-linalg-to-loops` / `convert-linalg-to-affine-loops` |
| `vector` | `llvm` / `spirv` | `convert-vector-to-llvm` / `convert-vector-to-spirv` |
| `memref` | `llvm` | `finalize-memref-to-llvm` |
| `tensor` | `memref` | one-shot bufferization: `one-shot-bufferize` |
| `func` | `llvm` | `convert-func-to-llvm` |
| `gpu` | `nvvm` / `rocdl` | `convert-gpu-to-nvvm` / `convert-gpu-to-rocdl` |
| `gpu` | CUBIN/HSAco | `gpu-to-cubin` / `gpu-to-hsaco` |
| `math` | `llvm` | `convert-math-to-llvm` |
| `index` | `llvm` | `convert-index-to-llvm` |
| `complex` | `llvm` | `convert-complex-to-llvm` |
| whole module | `llvm.mlir.module` | `convert-to-llvmir` (umbrella) |
