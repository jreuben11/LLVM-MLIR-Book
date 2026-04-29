# Chapter 166 — Mojo, Polygeist, Enzyme-MLIR, and Beyond

*Part XXIII — MLIR in Production*

MLIR has catalyzed a generation of specialized compilers and language systems that use it as a foundation. This chapter surveys four notable examples — Mojo (a Python superset for ML systems programming), Polygeist (a C/C++ polyhedral lifting tool), Enzyme-MLIR (automatic differentiation), and HEIR (homomorphic encryption) — plus briefly examines CIRCT (hardware design) and VAST (C/C++ analysis). Each represents a different use case for MLIR's extensibility: new programming languages, polyhedral analysis of existing code, program transformations with formal guarantees, and domain-specific analyses that benefit from a shared compiler infrastructure.

---

## 166.1 Mojo: Python Superset via MLIR

Mojo ([modular.com/mojo](https://www.modular.com/mojo)) is a Python-compatible programming language designed for high-performance ML systems work. It is compiled via MLIR, with extensions for GPU/SIMD programming that map directly to MLIR dialects.

### 166.1.1 Mojo's Design Philosophy

Python's dynamic nature makes it unsuitable for systems programming — you cannot write a CUDA kernel in Python. Mojo extends Python with:

- **Static type annotations** (optional but required for performance): `var x: Int32 = 42`
- **`fn` functions**: strictly typed, no dynamic dispatch, compiles to optimal machine code
- **`def` functions**: Python-compatible, dynamic semantics
- **`struct` types**: value semantics, like C structs, not Python dicts
- **`@parameter` blocks**: compile-time computation
- **SIMD types**: `SIMD[DType.float32, 8]` maps to `vector<8xf32>`
- **`@gpu.func` kernels**: GPU kernel annotation

### 166.1.2 Mojo's MLIR Compilation

Mojo's compiler (proprietary but based on open-source MLIR) emits MLIR for:

```mojo
fn matmul[M: Int, N: Int, K: Int](
    a: DTypePointer[DType.float32],
    b: DTypePointer[DType.float32],
    c: DTypePointer[DType.float32]
):
    alias TILE = 32
    
    @parameter
    for i in range(M):
        @parameter  
        for j in range(N):
            var acc = SIMD[DType.float32, TILE](0)
            for k in range(K // TILE):
                var a_vec = a.load[width=TILE](i * K + k * TILE)
                var b_vec = b.load[width=TILE](k * TILE + j)
                acc = fma(a_vec, b_vec, acc)
            c.store[width=TILE](i * N + j, acc)
```

The `@parameter` decorator marks the loop for unrolling/specialization at compile time. `SIMD[DType.float32, 32]` maps to `vector<32xf32>` in MLIR, and `fma` maps to `vector.fma`.

### 166.1.3 GPU Kernels in Mojo

```mojo
from gpu import block_idx, thread_idx, sync_threads

@gpu.func
fn vector_add_kernel[N: Int](
    x: DTypePointer[DType.float32],
    y: DTypePointer[DType.float32],
    out: DTypePointer[DType.float32]
):
    let tid = block_idx.x * 256 + thread_idx.x
    if tid < N:
        out[tid] = x[tid] + y[tid]
```

This lowers to `gpu.launch` in MLIR, then to NVVM or ROCDL as in Chapter 165.

### 166.1.4 Python Interoperability

Mojo's Python interop model is unique: both run in the same process via CPython embedding. Mojo can import Python modules:

```mojo
from python import Python

fn use_numpy():
    let np = Python.import_module("numpy")
    let arr = np.array([1.0, 2.0, 3.0])
    print(arr)  # Python array printed via CPython
```

Python objects are wrapped as `PythonObject` in Mojo — a reference to a CPython object with automatic refcount management. There is zero serialization overhead for function calls between Mojo and Python.

### 166.1.5 Mojo's MLIR Foundation

Mojo exposes MLIR types and ops directly for systems programmers who need maximum control:

```mojo
# Create MLIR types and ops from within Mojo
from sys.info import simd_width
from builtin.dtype import DType

# SIMD[DType.float32, 8] → !llvm.vec<8 x float> in MLIR
let v = SIMD[DType.float32, 8](1.0)
let w = SIMD[DType.float32, 8](2.0)
let result = v + w  # lowers to vector.add in MLIR
```

---

## 166.2 Polygeist: C/C++ Polyhedral Lifting

Polygeist ([github.com/llvm/Polygeist](https://github.com/llvm/Polygeist)) is a C/C++ → MLIR compiler that extracts affine loop structure from regular C programs, enabling polyhedral transformations (tiling, parallelization, loop interchange) on real-world code.

### 166.2.1 The Problem Polygeist Solves

MLIR's `affine` dialect requires loops to have affine bounds and induction variables as MLIR SSA values. C `for` loops with array indexing typically cannot be directly lowered to `affine.for` because:
1. The loop bounds may involve pointer arithmetic
2. Memory accesses are through raw pointers (not typed memrefs)
3. Side effects and aliasing prevent clean analysis

Polygeist performs **polyhedral lifting**: it analyzes C loops statically and converts them to `affine.for` / `affine.load` / `affine.store` when it can prove correctness.

### 166.2.2 Example: Matrix Multiply

```c
// Input C code
void matmul(float* A, float* B, float* C, int N) {
    for (int i = 0; i < N; i++)
        for (int j = 0; j < N; j++) {
            float acc = 0.0f;
            for (int k = 0; k < N; k++)
                acc += A[i*N + k] * B[k*N + j];
            C[i*N + j] = acc;
        }
}
```

Polygeist analyzes the access patterns `A[i*N+k]`, `B[k*N+j]`, `C[i*N+j]` and determines they are affine functions of `(i, j, k, N)`. It converts to:

```mlir
// Polygeist output: affine.for with affine memory accesses
func.func @matmul(%A: memref<?xf32>, %B: memref<?xf32>,
                  %C: memref<?xf32>, %N: index) {
  affine.for %i = 0 to %N {
    affine.for %j = 0 to %N {
      %acc = memref.alloca() : memref<f32>
      affine.store %cst_0, %acc[0] : memref<f32>
      affine.for %k = 0 to %N {
        // Affine access patterns: A[i*N + k] = A[(d0 * s0 + d1)]
        %a = affine.load %A[%i * symbol(%N) + %k] : memref<?xf32>
        %b = affine.load %B[%k * symbol(%N) + %j] : memref<?xf32>
        %acc_val = affine.load %acc[0] : memref<f32>
        %product = arith.mulf %a, %b : f32
        %new_acc = arith.addf %acc_val, %product : f32
        affine.store %new_acc, %acc[0] : memref<f32>
      }
      %final = affine.load %acc[0] : memref<f32>
      affine.store %final, %C[%i * symbol(%N) + %j] : memref<?xf32>
    }
  }
}
```

With the `affine.for` structure, standard MLIR passes can now apply polyhedral transformations:

```bash
polygeist-opt \
    --affine-loop-tile="tile-size=32" \
    --affine-parallelize \
    --convert-affine-to-scf \
    --convert-scf-to-openmp \
    matmul_lifted.mlir
```

### 166.2.3 Polygeist Pipeline

```bash
# Step 1: Parse C and emit MLIR
polygeist-opt --cppheaders=/usr/include \
    --function=matmul \
    matmul.c > matmul_lifted.mlir

# Step 2: Optimize with polyhedral passes
mlir-opt --affine-loop-tile="tile-size=64" \
         --affine-loop-interchange \
         --affine-loop-unroll="unroll-factor=4" \
         matmul_lifted.mlir > optimized.mlir

# Step 3: Lower to LLVM and compile
mlir-opt --lower-affine --convert-scf-to-cf --convert-cf-to-llvm \
         --convert-arith-to-llvm --convert-func-to-llvm \
         optimized.mlir | mlir-translate --mlir-to-llvmir | \
         llc -march=x86-64 -O3 -o matmul.s
```

### 166.2.4 Limitations and Scope

Polygeist handles:
- Static and parametric loop bounds
- Affine memory accesses (linear functions of loop variables and parameters)
- Simple conditional guards within loops

It does not handle:
- Pointer-based data structures (linked lists, trees)
- Function calls with unknown effects
- Irregular access patterns (sparse matrices, hash maps)

---

## 166.3 Enzyme-MLIR: Automatic Differentiation

Enzyme ([github.com/EnzymeAD/Enzyme](https://github.com/EnzymeAD/Enzyme)) is an automatic differentiation (AD) tool that works at the LLVM IR level. Enzyme-MLIR is its MLIR integration, enabling AD on arbitrary MLIR programs before lowering to LLVM.

### 166.3.1 The enzyme.autodiff Op

```mlir
// Forward mode: differentiate f(x) with respect to x
%result, %grad = enzyme.autodiff @f(%x) ({%dx})
    {
      activity = "active",
      returnPrimal = true
    }
    : (f64) -> (f64, f64)

// Reverse mode (adjoint): given output gradient, compute input gradient
%grad_x = enzyme.autodiff @f(%x) ({%grad_output})
    {
      mode = "reverse",
      returnPrimal = false
    }
    : (f64) -> f64
```

### 166.3.2 Forward Mode AD

Forward mode computes the Jacobian-vector product (JVP): given input `x` and tangent `dx`, compute `(f(x), df/dx * dx)`. It inserts a shadow variable for each active variable:

```mlir
// Original:
func.func @f(%x: f64) -> f64 {
  %y = arith.mulf %x, %x : f64  // y = x^2
  return %y : f64
}

// After forward-mode Enzyme-MLIR:
func.func @f_fwd(%x: f64, %dx: f64) -> (f64, f64) {
  %y = arith.mulf %x, %x : f64   // primal: y = x^2
  %dy1 = arith.mulf %dx, %x : f64
  %dy2 = arith.mulf %x, %dx : f64
  %dy = arith.addf %dy1, %dy2 : f64  // tangent: dy = 2*x*dx
  return %y, %dy : f64, f64
}
```

### 166.3.3 Reverse Mode AD

Reverse mode computes vector-Jacobian products (VJP): given output gradient `g`, compute `g^T * (df/dx)`. This is the operation used in backpropagation.

Enzyme implements reverse mode by:
1. Running the primal forward pass while recording an activity tape
2. Replaying the tape in reverse with adjoint (gradient) propagation

```mlir
// After reverse-mode Enzyme-MLIR:
func.func @f_rev(%x: f64, %g: f64) -> f64 {
  // Forward pass
  %y = arith.mulf %x, %x : f64
  
  // Reverse pass (adjoint propagation)
  // d(x^2)/dx = 2x, so grad_x = g * 2 * x
  %two = arith.constant 2.0 : f64
  %two_x = arith.mulf %two, %x : f64
  %grad_x = arith.mulf %g, %two_x : f64
  
  return %grad_x : f64
}
```

### 166.3.4 Working on Arbitrary MLIR Dialects

Unlike JAX's transformable AD (which requires the function to be written in JAX primitives) or PyTorch Autograd (which requires `torch.Tensor` ops), Enzyme-MLIR works on **any MLIR dialect** as long as the ops have proper memory effects and the dialect supports basic algebraic operations.

This means Enzyme can differentiate through:
- Custom hardware-specific dialect ops (if the op's mathematical semantics are known to Enzyme)
- Linalg ops (Enzyme has built-in rules for `linalg.matmul`, `linalg.conv2d`, etc.)
- Arbitrary `scf.for` loops
- Calls to external functions (with annotation)

### 166.3.5 JAX-Enzyme Integration

The Enzyme-JAX package allows using Enzyme's MLIR-level AD within JAX programs:

```python
import jax
import jax.numpy as jnp
import enzyme_jax

def complex_kernel(x, y):
    # Some function that JAX's AD struggles with
    return jnp.sum(jnp.linalg.solve(x, y))

# Use Enzyme-MLIR for AD instead of JAX's built-in AD
grad_fn = enzyme_jax.grad(complex_kernel, argnums=0)
grad_x = grad_fn(x, y)
```

---

## 166.4 HEIR: Homomorphic Encryption IR

HEIR ([github.com/google/heir](https://github.com/google/heir)) is an MLIR-based compiler for Fully Homomorphic Encryption (FHE) — a cryptographic primitive that allows computation on encrypted data.

### 166.4.1 FHE Circuits as MLIR

FHE schemes (BGV, BFV, CKKS) represent computation as arithmetic circuits over encrypted polynomials. HEIR provides MLIR dialects for these circuits:

```mlir
// HEIR's polynomial arithmetic dialect
%a_enc = poly.encrypt %plaintext using %key : !poly.ciphertext<bgv, degree=8192>
%b_enc = poly.encrypt %other using %key : !poly.ciphertext<bgv, degree=8192>

// Homomorphic addition (on encrypted data)
%sum_enc = poly.add %a_enc, %b_enc : !poly.ciphertext<bgv, degree=8192>

// Homomorphic multiplication (expensive; requires relinearization)
%prod_enc = poly.mul %a_enc, %b_enc : !poly.ciphertext<bgv, degree=8192>
%prod_relined = poly.relinearize %prod_enc using %relin_key
    : !poly.ciphertext<bgv, degree=8192>
```

### 166.4.2 Program Optimization for FHE

HEIR's key contribution is **noise budget optimization** — FHE ciphertexts accumulate noise with each operation; when noise exceeds a threshold, the plaintext becomes unrecoverable. HEIR passes:

- `noise-estimator`: tracks noise budget through the circuit
- `bootstrapping-inserter`: inserts expensive `bootstrap()` ops where noise is too high
- `depth-optimizer`: minimizes circuit depth (depth = max chain of multiplications)

### 166.4.3 Target Backend

HEIR lowers FHE circuits to calls to OpenFHE (an open-source FHE library):

```mlir
// HEIR output: calls to OpenFHE runtime
func.func @bgv_matmul(%ct_vec: !openfhe.ciphertext) -> !openfhe.ciphertext {
  %result = openfhe.eval_inner_product %ct_vec, %pt_matrix
      : (!openfhe.ciphertext, !openfhe.plaintext) -> !openfhe.ciphertext
  return %result
}
```

---

## 166.5 CIRCT: Circuit IR Compilers and Tools

CIRCT ([github.com/llvm/circt](https://github.com/llvm/circt)) is an MLIR-based infrastructure for hardware design — essentially applying MLIR's extensibility philosophy to the electronic design automation (EDA) domain.

### 166.5.1 CIRCT Dialects

```
Input HDL (System Verilog, FIRRTL)
    │
    ▼
FIRRTL dialect (circuits, modules, wires, regs, mems)
    │
    ▼
HW dialect (combinational hardware)
Comb dialect (combinational logic: And, Or, Xor, Add, Mux)
Seq dialect (sequential logic: FFs, memories)
    │
    ▼
SV dialect (SystemVerilog output)
    │
    ▼
SystemVerilog file → synthesis tools
```

### 166.5.2 Example: HW Dialect

```mlir
// A simple adder in CIRCT's HW dialect
hw.module @adder(%a: i8, %b: i8) -> (sum: i8, carry: i1) {
  %sum = comb.add %a, %b : i8
  
  // Detect overflow for carry
  %a_zext = comb.concat %c0, %a : i1, i8  // zero-extend to 9 bits
  %b_zext = comb.concat %c0, %b : i1, i8
  %sum_wide = comb.add %a_zext, %b_zext : i9
  %carry = comb.extract %sum_wide from 8 : (i9) -> i1
  
  hw.output %sum, %carry : i8, i1
}
```

CIRCT is used by Chipyard (RISC-V chip design framework) and is the reference implementation of the MLIR infrastructure for hardware.

---

## 166.6 VAST: C/C++ AST Analysis

VAST ([github.com/trailofbits/vast](https://github.com/trailofbits/vast)) (Verified Analysis and Static analysis Tools) provides an MLIR dialect for C/C++ AST-level analysis. Unlike ClangIR (Chapter 52–54), which focuses on codegen, VAST targets security analysis and formal verification.

### 166.6.1 VAST's HL Dialect

```mlir
// VAST's hl (High Level) dialect for C code analysis
hl.func @buffer_copy(%dst: !hl.ptr<!hl.char>, %src: !hl.ptr<!hl.char>,
                      %n: !hl.int) {
  hl.for {
    %i = hl.var "i" : !hl.int = 0
    hl.cond: (hl.lt %i, %n : !hl.int)
    hl.inc %i : !hl.int
  } do {
    // Array access: dst[i] = src[i]
    %dst_i = hl.subscript %dst[%i] : !hl.ptr<!hl.char>, !hl.int
    %src_i = hl.subscript %src[%i] : !hl.ptr<!hl.char>, !hl.int
    %val = hl.deref %src_i : !hl.ptr<!hl.char>
    hl.assign %val to %dst_i : !hl.char, !hl.ptr<!hl.char>
  }
}
```

The HL dialect preserves C semantics (including pointer arithmetic, casts, and UB-inducing operations) for precise security analysis. VAST passes can then:
- Detect buffer overflows by tracking pointer bounds
- Identify use-after-free by tracking object lifetimes
- Generate verification conditions for formal proof tools

---

## 166.7 Trends and Convergence

### 166.7.1 MLIR as Universal Compiler IR

The projects surveyed here share a common pattern: they use MLIR as a shared compiler infrastructure and contribute to or consume from the LLVM monorepo. This creates network effects:
- A new optimization pass in MLIR's affine dialect benefits Polygeist users
- A new vectorization pass benefits Mojo, IREE, and Triton users simultaneously
- FlashAttention improvements in Triton can be leveraged by XLA users

### 166.7.2 Domain Specialization vs. Reuse

| Project | Specialty | Shared MLIR Infrastructure Used |
|---------|-----------|--------------------------------|
| Mojo | New ML language | LLVM dialect, GPU dialect, vectorization |
| Polygeist | Polyhedral lifting from C | Affine dialect, SCF |
| Enzyme-MLIR | Automatic differentiation | Linalg, LLVM dialect |
| HEIR | Homomorphic encryption | Polynomial ops, custom types |
| CIRCT | Hardware design | Custom dialects, general MLIR infra |
| VAST | C/C++ security analysis | Custom AST-level dialect |

### 166.7.3 The Emerging Landscape (2026)

As of early 2026:
- **Mojo** is in production at Modular; growing usage for ML kernels and systems code
- **Polygeist** is a research tool with growing production adoption for HPC workloads
- **Enzyme-MLIR** is production-ready for common MLIR dialects; Enzyme-JAX is in active development
- **HEIR** is under active Google development; targets privacy-preserving ML inference
- **CIRCT** is used in RISC-V chip design; Chipyard/BOOM processor families use it
- **VAST** is Trail of Bits research infrastructure; used in CVE-finding campaigns

---

## Chapter Summary

- Mojo compiles Python-compatible syntax via MLIR; `SIMD[DType, N]` maps to `vector<Nxf32>`, `@gpu.func` maps to `gpu.launch`, `@parameter` blocks are compile-time specialization.
- Polygeist lifts C/C++ loop nests to MLIR's `affine.for` dialect when loop bounds and array accesses are affine functions; this enables standard polyhedral passes (tiling, interchange, parallelization).
- Enzyme-MLIR provides the `enzyme.autodiff` op for forward and reverse mode AD on arbitrary MLIR programs; it works on linalg, SCF, and custom dialects, not just LLVM IR.
- HEIR provides MLIR dialects for FHE arithmetic; its passes optimize circuit depth and insert bootstrapping to stay within noise budget; it targets OpenFHE as a backend.
- CIRCT applies MLIR's extensibility to hardware design: HW/Comb/Seq/SV dialects represent combinational and sequential logic, lowering to SystemVerilog for synthesis.
- VAST provides C/C++ AST-level MLIR dialects for security analysis and formal verification; it preserves C semantics including pointer arithmetic and UB.
- The common theme: each project contributes to a shared MLIR ecosystem, creating network effects where improvements to core dialects benefit multiple downstream projects simultaneously.
