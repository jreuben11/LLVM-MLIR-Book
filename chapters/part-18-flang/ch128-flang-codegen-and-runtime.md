# Chapter 128 — Flang Codegen and Runtime

*Part XVIII — Flang*

The final stage of Flang's compilation pipeline converts the FIR dialect representation to LLVM IR and then to machine code, while the Fortran runtime library — `flang-rt` — provides the services that generated code cannot live without: formatted I/O, intrinsic array operations, allocatable lifetime management, and IEEE floating-point control. This chapter covers the FIR-to-LLVM conversion machinery, the optimization options Flang exposes, the structure and key interfaces of `flang-rt`, IEEE exception handling, and the testing strategy that validates end-to-end correctness.

---

## 128.1 FIR to LLVM Conversion

### The ConvertFIRToLLVM Pass

The central codegen pass is `ConvertFIRToLLVM`, defined in [`flang/lib/Optimizer/CodeGen/CodeGen.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Optimizer/CodeGen/CodeGen.cpp). It is a standard MLIR dialect conversion pass that applies a set of **type converters** and **op conversion patterns** to transform every FIR type and op into the MLIR LLVM dialect.

### FIR Type Conversion

The `FIRToLLVMTypeConverter` ([`flang/lib/Optimizer/CodeGen/TypeConverter.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Optimizer/CodeGen/TypeConverter.h)) maps:

| FIR Type | LLVM Dialect Type |
|----------|-------------------|
| `fir.ref<T>` | `!llvm.ptr` |
| `fir.ptr<T>` | `!llvm.ptr` |
| `fir.heap<T>` | `!llvm.ptr` |
| `fir.array<NxT>` | `!llvm.array<N x T_llvm>` |
| `fir.box<T>` | `!llvm.struct<(ptr,i64,i32,i8,i8,i8,i8,array<R×array<3×i64>>)>` |
| `fir.char<1,N>` | `!llvm.array<N x i8>` |
| `fir.boxchar<1>` | `!llvm.struct<(ptr, i64)>` |
| `fir.complex<4>` | `!llvm.struct<(f32, f32)>` |

`fir.box` becomes an LLVM struct with the CFI descriptor layout:
```
struct {
  void *base_addr;     // i64 pointer-sized
  size_t elem_len;     // i64
  int version;         // i32 = CFI_VERSION
  signed char rank;    // i8
  signed char type;    // i8 (CFI_type_t)
  signed char attribute; // i8
  unsigned char extra; // i8
  struct { i64 lower_bound; i64 extent; i64 stride; } dim[rank];
}
```

### Op Conversion Patterns

Each FIR op has a corresponding conversion pattern. Notable examples:

**`fir.alloca` → `llvm.alloca`**:
```cpp
// FIR:  %x = fir.alloca i32
// LLVM: %x = llvm.alloca %c1 x i32
```

**`fir.load` / `fir.store` → `llvm.load` / `llvm.store`**:
```cpp
// FIR:  %v = fir.load %ref : (!fir.ref<f64>) -> f64
// LLVM: %v = llvm.load %ref : !llvm.ptr -> f64
```

**`fir.call` → `llvm.call`**:
```cpp
// Direct call preserved; indirect call through function pointer uses llvm.call with ptr
```

**`fir.do_loop` → LLVM CFG**:
The structured `fir.do_loop` becomes a three-block CFG: preheader (compute bounds), header (condition check), body, and latch (increment + back-edge):
```
   preheader: compute %lb, %ub, %step
   header:    %cond = icmp sle %i, %ub; condbr %cond, body, exit
   body:      ...
   latch:     %i_next = add %i, %step; br header
   exit:
```

**`fir.embox` → `llvm.insertvalue` sequence**:
Box construction fills each field of the descriptor struct via `llvm.insertvalue`:
```mlir
%desc0 = llvm.mlir.undef : !llvm.struct<(ptr,i64,i32,i8,i8,i8,i8,...)>
%desc1 = llvm.insertvalue %base_addr, %desc0[0] : ...
%desc2 = llvm.insertvalue %elem_len,  %desc1[1] : ...
// ... etc.
```

### TargetRewrite Pass

Before `ConvertFIRToLLVM`, the `TargetRewrite` pass ([`flang/lib/Optimizer/CodeGen/TargetRewrite.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Optimizer/CodeGen/TargetRewrite.cpp)) performs **ABI adjustments**:

- **Struct passing**: Fortran complex numbers and small derived types may be passed in integer registers on some ABIs (e.g., x86-64 System V). `TargetRewrite` splits them into integer pairs or restructures call signatures.
- **Box passing**: assumed-shape arrays passed by reference (pointer to descriptor) vs. by value (copy of descriptor) depending on the calling convention.
- **Character length arguments**: Fortran passes assumed-length character lengths as hidden trailing arguments. `TargetRewrite` adds these `i64` arguments to function signatures.

The target-specific logic uses `CodeGenSpecifics` subclasses:
```cpp
class X86_64TargetCodeGenInfo : public CodeGenSpecifics { ... };
class AArch64TargetCodeGenInfo : public CodeGenSpecifics { ... };
```

### BoxedProcedurePass

The `BoxedProcedurePass` ([`flang/lib/Optimizer/Transforms/BoxedProcedure.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/lib/Optimizer/Transforms/BoxedProcedure.cpp)) handles **procedure pointers** — Fortran's typed function pointers stored in `fir.box<fir.func<...>>`. These are converted to `!llvm.ptr` with appropriate casts.

---

## 128.2 Compiler Options and Optimization

### Optimization Levels

Flang's optimization levels map to LLVM's standard pipeline:

| Flang Flag | LLVM Pipeline Level | Effect |
|------------|--------------------|----|
| `-O0` | `OptimizationLevel::O0` | No optimization; fastest compilation |
| `-O1` | `OptimizationLevel::O1` | Basic optimizations (instcombine, simplifycfg) |
| `-O2` | `OptimizationLevel::O2` | Full middle-end pipeline |
| `-O3` | `OptimizationLevel::O3` | Aggressive inlining, loop optimizations |
| `-Os` | `OptimizationLevel::Os` | Optimize for size |
| `-Oz` | `OptimizationLevel::Oz` | Aggressively optimize for size |

The MLIR FIR pipeline also respects the optimization level:
- At `-O0`: HLFIR → FIR → LLVM with minimal passes
- At `-O2`/`-O3`: includes `InlineElementals`, `MemRefDataFlowOpt`, array copy elimination, loop invariant code motion

### Floating-Point Flags

```bash
# Flush-to-zero + reciprocal approximations (like Clang -ffast-math):
flang-new -ffast-math -O3 kernel.f90

# Fused multiply-add (when FP model allows):
flang-new -ffp-contract=fast -O2 kernel.f90

# IEEE strict semantics (no FP reordering):
flang-new -ffp-model=strict kernel.f90

# Flush denormals to zero only:
flang-new -fdenormal-fp-math=preserve-sign,preserve-sign kernel.f90
```

The `-ffp-model` flag accepts the same values as Clang: `precise` (default), `strict`, `fast`.

### FIR-Specific Optimization Flags

```bash
# Enable loop vectorization hints in FIR:
flang-new -mmlir --enable-licm-version-of-loop-opt=true kernel.f90

# Control inlining threshold:
flang-new -mmlir --mlir-inlining-threshold=100 kernel.f90

# Emit FIR after each pass (debugging):
flang-new -fc1 -emit-fir -mmlir --mlir-print-ir-after-all kernel.f90
```

### Vectorization

Flang does not have a Fortran-level auto-vectorizer separate from LLVM's. Array elemental operations lower to scalar loops in FIR, and LLVM's loop vectorizer (`LoopVectorizePass`) handles vectorization at the LLVM IR level. OpenMP `!$omp simd` directives add `llvm.loop.vectorize.enable` metadata to guide the vectorizer.

---

## 128.3 The Fortran Runtime Library (flang-rt)

`flang-rt` is the Fortran runtime library, located in `flang/runtime/`. It is separate from the C/C++ runtime (libc, libc++) but links against libc for low-level I/O and math functions. All entry points use `_Fortran` or `_FortranA` name prefixes to avoid conflicts.

### I/O Runtime

The I/O subsystem is the most complex part of `flang-rt`. Fortran has a rich I/O model: formatted, unformatted, list-directed, namelist, internal, sequential, direct-access, stream, and asynchronous I/O.

Key header: [`flang/runtime/io-api.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/runtime/io-api.h)

```cpp
// Entry points for WRITE(*,*) scalar:
Cookie IONAME(BeginExternalListOutput)(ExternalUnit = defaultUnit,
                                       const char *sourceFile = nullptr,
                                       int sourceLine = 0);
bool IONAME(OutputInteger8)(Cookie, std::int8_t);
bool IONAME(OutputInteger32)(Cookie, std::int32_t);
bool IONAME(OutputInteger64)(Cookie, std::int64_t);
bool IONAME(OutputReal32)(Cookie, float);
bool IONAME(OutputReal64)(Cookie, double);
bool IONAME(OutputLogical)(Cookie, bool);
bool IONAME(OutputCharacter)(Cookie, const char *, std::size_t, int kind = 1);
enum Iostat IONAME(EndIoStatement)(Cookie);
```

The `Cookie` is an opaque handle to the current I/O statement context (`IoStatementState *`), allocated on the stack via a fixed-size buffer to avoid heap allocation in the common case.

Fortran format item processing (`FORMAT` strings) is implemented in [`flang/runtime/format-implementation.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/runtime/format-implementation.h) as a template-heavy state machine that handles `I`, `F`, `E`, `D`, `G`, `A`, `L`, `X`, `/`, `\`, `(...)` repeat groups, and all Fortran 2018 format items.

### Intrinsic Array Runtime

For intrinsics too complex to inline (large arrays, runtime-determined shapes), FIR emits runtime calls:

```cpp
// flang/runtime/matmul.h:
void RTNAME(Matmul)(Descriptor &result, const Descriptor &matrix1,
                    const Descriptor &matrix2, const char *sourceFile = nullptr,
                    int line = 0);

// flang/runtime/reduction.h:
double RTNAME(SumReal8)(const Descriptor &, const char *source, int line,
                        int dim = 0, const Descriptor *mask = nullptr);
std::int64_t RTNAME(Count)(const Descriptor &mask, const char *source,
                            int line, int dim = 0);

// flang/runtime/transformational.h:
void RTNAME(Reshape)(Descriptor &result, const Descriptor &source,
                     const Descriptor &shape, const Descriptor *pad = nullptr,
                     const Descriptor *order = nullptr, ...);
void RTNAME(Transpose)(Descriptor &result, const Descriptor &matrix, ...);
```

The `Descriptor` class ([`flang/runtime/descriptor.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/flang/runtime/descriptor.h)) is the C++ mirror of `CFI_cdesc_t`:

```cpp
class Descriptor {
public:
  void *raw() { return header_.base_addr; }
  std::size_t ElementBytes() const { return header_.elem_len; }
  int rank() const { return header_.rank; }
  const Dimension &GetDimension(int r) const { return dim_[r]; }
  // ... element addressing, extent queries, stride iteration
private:
  ISO::CFI_cdesc_t header_;
  Dimension dim_[maxRank];  // maxRank = 16
};
```

### Allocatable Operations

```cpp
// flang/runtime/allocatable.h:
int RTNAME(AllocatableAllocate)(Descriptor &, bool hasStat = false,
                                 const Descriptor *errMsg = nullptr, ...);
int RTNAME(AllocatableDeallocate)(Descriptor &, bool hasStat = false,
                                   const Descriptor *errMsg = nullptr, ...);
// For allocatable arrays: sets base_addr, elem_len, extents, strides
int RTNAME(AllocatableAllocateSource)(Descriptor &alloc,
                                       const Descriptor &source, ...);
```

### Type Information for Polymorphism

Fortran's `CLASS` polymorphism requires runtime type information. `flang-rt` provides the type info infrastructure:

```cpp
// flang/runtime/type-info.h:
namespace Fortran::runtime::typeInfo {
struct DerivedType {
  const char *name;               // type name
  std::uint64_t sizeInBytes;
  const DerivedType *parent;      // for EXTENDS
  const Binding *binding;         // virtual procedure table
  const Component *component;     // data components
  const ProcedurePointer *proc;   // procedure pointer components
  // ...
};
struct Binding {
  const char *name;
  void (*proc)(...);  // function pointer
};
} // namespace
```

The type descriptor for each derived type is emitted as a global constant (`__Fortran_type_info_<typename>`) by the code generator and used by `ALLOCATE` with `CLASS`, `SELECT TYPE`, and polymorphic procedure dispatch.

### Error Termination

```cpp
// flang/runtime/terminator.h:
class Terminator {
public:
  explicit Terminator(const char *sourceFileName, int sourceLine)
      : sourceFileName_{sourceFileName}, sourceLine_{sourceLine} {}

  [[noreturn]] void Crash(const char *message, ...) const;
  [[noreturn]] void CheckFailed(const char *predicate, ...) const;
};
```

`Terminator::Crash()` writes to `stderr` and calls `abort()`. All runtime error paths (shape mismatch, out-of-bounds if `-fbounds-check`, I/O error without `IOSTAT=`) go through `Terminator`.

---

## 128.4 IEEE Floating-Point Exception Handling

### Fortran IEEE Intrinsic Module

Fortran 2003 defines the `IEEE_ARITHMETIC` and `IEEE_EXCEPTIONS` intrinsic modules, which provide portable control over floating-point exceptions, rounding modes, and special values.

Key procedures from `IEEE_ARITHMETIC`:
- `IEEE_SET_HALTING_MODE(flag, halting)`: enable/disable FP exception trapping
- `IEEE_GET_FLAG(flag, flag_value)`: query whether an exception occurred
- `IEEE_SET_FLAG(flag, flag_value)`: set/clear exception flags
- `IEEE_GET_ROUNDING_MODE(round_value)`: query current rounding mode
- `IEEE_SET_ROUNDING_MODE(round_value)`: set rounding mode
- `IEEE_IS_NAN(x)`, `IEEE_IS_FINITE(x)`, `IEEE_IS_NEGATIVE(x)`: special value queries

### Runtime Implementation

```cpp
// flang/runtime/ieee_arithmetic.h:
// Implemented as inline wrappers around hardware FP control registers

// Get/set FP exception flags (maps to fegetexceptflag / fesetexceptflag):
void RTNAME(IeeeFlagGet)(int flag, bool &value);
void RTNAME(IeeeFlagSet)(int flag, bool value);

// Halting mode (maps to feenableexcept / fedisableexcept on Linux):
void RTNAME(IeeeHaltingGet)(int flag, bool &halting);
void RTNAME(IeeeHaltingSet)(int flag, bool halting);

// Rounding mode (maps to fegetround / fesetround):
void RTNAME(IeeeRoundGet)(int &round);
void RTNAME(IeeeRoundSet)(int round);
```

### SIGFPE Handling

When `IEEE_SET_HALTING_MODE(IEEE_INVALID, .TRUE.)` is active, FP operations that produce `NaN` raise `SIGFPE`. `flang-rt` installs a `SIGFPE` handler at startup that translates the signal into a Fortran `IEEE_HALTING_MODE_CHANGED` condition, which can be caught with `IEEE_GET_FLAG`.

### Current Limitations

- `IEEE_SET_HALTING_MODE` works on Linux x86-64 and AArch64 via `feenableexcept()`; macOS does not provide a portable API for this and requires inline assembly
- `IEEE_NEXT_AFTER`, `IEEE_NEXT_DOWN`, `IEEE_NEXT_UP` (navigating the IEEE lattice) are implemented but `REAL(10)` (80-bit extended) has edge cases
- `IEEE_SELECTED_REAL_KIND` with min-exponent queries requires careful mapping to Fortran KINDs

---

## 128.5 Testing Flang

### Lit Test Infrastructure

Flang uses LLVM's **lit** test infrastructure with site-configured substitutions:

```
%flang              = flang-new (full driver)
%flang_fc1          = flang-new -fc1 (front end)
%bbc                = bbc (MLIR-level testing)
%flang-new          = flang-new (alias)
```

Test categories in `flang/test/`:

```
Driver/         -- driver option tests; CHECK driver stderr/stdout
Parser/         -- prescan + parse tests; --fdebug-unparse
Semantics/      -- semantic error checking; CHECK-ERROR: messages
Lower/          -- bbc --emit-hlfir/--emit-fir; FileCheck MLIR output
HLFIR/          -- individual HLFIR pass tests via mlir-opt
Fir/            -- FIR pass tests via mlir-opt  
Transforms/     -- optimizer pass tests
Codegen/        -- FIR→LLVM conversion; FileCheck LLVM dialect output
Runtime/        -- runtime library correctness tests (lit + gtest)
```

### HLFIR Test Example

```fortran
! RUN: bbc --emit-hlfir %s -o - | FileCheck %s

subroutine matmul_test(a, b, c)
  real(8) :: a(10,10), b(10,10), c(10,10)
  c = matmul(a, b)
end subroutine
! CHECK: hlfir.matmul
! CHECK-NOT: fir.call @_FortranAMatmul
! (SimplifyHLFIRIntrinsics should not inline this — too large)
```

### FIR Codegen Test Example

```
! RUN: bbc --emit-fir %s -o - | \
! RUN:   mlir-opt --fir-to-llvm-ir | FileCheck %s

subroutine store_elem(a, i, val)
  real :: a(:), val
  integer :: i
  a(i) = val
end subroutine
! CHECK: llvm.store
! CHECK: llvm.gep
```

### End-to-End Tests

Runtime correctness is validated by:

1. **SPEC CPU 2017**: standard industry benchmarks. All Fortran benchmarks (503, 507, 521, 527, 549, 554, 628, 654) must produce correct results.
2. **ECMWF IFS**: large operational weather forecast model.
3. **`flang/test/Runtime/`**: unit tests for individual runtime entry points using GTest.

```cpp
// Example runtime unit test:
TEST(Io, ListDirectedIntegerOutput) {
  std::string got{};
  auto *io = IONAME(BeginInternalListOutput)(got.data(), got.size());
  EXPECT_TRUE(IONAME(OutputInteger64)(io, 42));
  IONAME(EndIoStatement)(io);
  EXPECT_EQ(got, " 42");
}
```

---

## 128.6 Compiler Pipeline Summary

Pulling together all the pieces covered in Chapters 125–128, here is the complete Flang pipeline from source to object:

```
flang-new hello.f90 -O2 -o hello.o
    │
    ├─ Driver: parse flags, determine actions, invoke -fc1
    │
    └─ flang-new -fc1 hello.f90 -O2 -emit-obj -o hello.o
        │
        ├─ Prescanner: fixed/free-form normalization, INCLUDE expansion
        ├─ Parser: recursive-descent → ParseTree
        ├─ Semantics: ResolveNames, CheckDeclarations, Evaluate::Expr<T>
        │
        ├─ Lower::Bridge: ParseTree+Semantics → mlir::ModuleOp (HLFIR)
        │
        ├─ MLIR FIR/HLFIR Pass Pipeline:
        │   SimplifyHLFIRIntrinsics
        │   LowerHLFIROrderedAssignments
        │   LowerHLFIRIntrinsics
        │   BufferizeHLFIR
        │   ConvertHLFIRtoFIR
        │   [FIR canonicalization]
        │   ArrayValueCopy
        │   CharacterConversion
        │   MemRefDataFlowOpt
        │   TargetRewrite
        │   BoxedProcedurePass
        │   ConvertFIRToLLVM
        │
        ├─ ConvertMLIRToLLVMIR → llvm::Module
        │
        └─ LLVM Backend (O2 pipeline):
            SROAPass, InstCombine, GVN, LoopVectorize, ...
            → MachineFunction → MCInst → object file
```

---

## Chapter Summary

- `ConvertFIRToLLVM` is the primary codegen pass; it converts FIR types (especially `fir.box` → CFI descriptor struct) and ops (loops, memory ops, calls) to the MLIR LLVM dialect, followed by `ConvertMLIRToLLVMIR`
- `TargetRewrite` handles ABI-specific adjustments: complex number struct splitting, character length hidden arguments, and box passing conventions; `BoxedProcedurePass` resolves procedure pointer boxes
- Optimization levels `-O0` through `-O3` control both the MLIR FIR pass pipeline and the downstream LLVM optimization pipeline; `-ffast-math` and `-ffp-contract=fast` enable floating-point optimizations
- `flang-rt` (in `flang/runtime/`) provides I/O (`_FortranAio*`), array intrinsics (`_FortranAMatmul`, `_FortranASumReal8`), allocatable management, polymorphic type information (`DerivedType` structs), and error termination
- The `Descriptor` class in `flang-rt` mirrors `CFI_cdesc_t` and is the runtime representation of `fir.box<T>` passed to all runtime routines
- IEEE floating-point exception handling is implemented via `feenableexcept`/`fegetexceptflag` on Linux; `IEEE_SET_HALTING_MODE` is functional on x86-64 and AArch64 with known limitations on macOS and for extended-precision types
- The test suite uses lit with `%flang_fc1`/`%bbc` substitutions; SPEC CPU 2017 Fortran benchmarks serve as the primary end-to-end correctness validation


---

@copyright jreuben11
