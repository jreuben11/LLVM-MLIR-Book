# Part XVII — Runtime Libraries — Part Summary

*This part examines the six runtime libraries that every LLVM-compiled binary depends on at execution time — compiler-rt builtins, libunwind, libc++, libc++abi, LLVM-libc, and the OpenMP/offload runtimes — showing how each layer is designed, deployed, and extended for targets ranging from GPU kernels to bare-metal microcontrollers.*

---

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 119 | compiler-rt Builtins | Low-level arithmetic, soft-float, and profile/sanitizer support infrastructure |
| 120 | libunwind | DWARF CFI and ARM EHABI stack unwinding for C++ exceptions and backtraces |
| 121 | libc++ | LLVM C++ standard library: ABI versioning, hardening, modules, PSTL |
| 122 | libc++abi | Itanium C++ ABI runtime: throw/catch, personality function, RTTI, guards |
| 123 | LLVM-libc | LLVM's C standard library with TableGen entrypoints, correct rounding, GPU support |
| 124 | OpenMP and Offload Runtimes | Host thread pool, GPU offload pipeline, and unified liboffload API |

---

## Part Overview

Part XVII addresses the execution substrate that sits between compiler-generated code and the operating system. Every binary produced by Clang silently links against at least one of these libraries, yet they are rarely examined together. Understanding them collectively reveals a coherent design philosophy: each library occupies a precise niche, they compose in a strict dependency order, and all six are built from the LLVM monorepo under the `LLVM_ENABLE_RUNTIMES` cmake variable, which means they can be bootstrapped together against a freshly compiled Clang.

The foundational layer is **compiler-rt**, specifically its builtins sub-library. Before any standard library code runs, the ABI demands that integer division, 128-bit arithmetic, soft-float operations, and bit-manipulation primitives be reachable via compiler-generated calls (e.g., `__udivdi3`, `__mulsf3`). compiler-rt provides these as pure C ABI functions with no C++ or libc dependency, making them safe for freestanding firmware. Alongside the builtins, compiler-rt hosts the profile instrumentation runtime (`libclang_rt.profile`) and the shared foundation (`sanitizer_common`) used by all sanitizer runtimes. Chapter 119 thus serves as the base on which everything else in the part rests.

Stack unwinding, covered in Chapter 120, is the next layer. **libunwind** implements the Itanium `_Unwind_*` ABI by interpreting DWARF CFI bytecode encoded in `.eh_frame` sections. It supports x86_64, AArch64, 32-bit ARM EHABI, macOS compact unwind, and Windows SEH. Every C++ exception and every sanitizer backtrace passes through libunwind. The `UnwindCursor<A,R>` template decouples the address-space accessor from the register-context implementation, allowing the same logic to work in bare-metal static builds (with manual `__register_frame` calls) as well as full OS environments using `dl_iterate_phdr`. Chapter 120 is a direct prerequisite for understanding Chapters 121 and 122.

The C++ standard library stack spans Chapters 121 and 122. **libc++** (Chapter 121) delivers `std::string`, `std::vector`, `<algorithm>`, `<format>`, `std::generator`, `std::mdspan`, and the PSTL parallel algorithms backend, all wrapped in an inline-namespace ABI versioning scheme (`std::__2::`) that produces deliberate link-time incompatibility rather than silent ODR corruption. LLVM 22 stabilizes four hardening levels and ships first-class C++23 named modules (`import std;`). **libc++abi** (Chapter 122) is the layer libc++ calls but does not implement itself: `__cxa_throw`, `__gxx_personality_v0`, `__cxa_guard_acquire`, `__cxa_demangle`, and `__dynamic_cast`. Together with libunwind, these three libraries must be built in a single `LLVM_ENABLE_RUNTIMES="libunwind;libcxxabi;libcxx"` invocation to guarantee ABI cohesion.

The final two chapters broaden scope beyond the C++ model. **LLVM-libc** (Chapter 123) reimagines the C standard library from scratch: a TableGen pipeline assigns every function an entrypoint in a versioned C++ namespace (`__llvm_libc_22`), the header generator omits functions absent from the platform entrypoint list, and the test suite demands per-function hermetic unit tests before any function ships. The math functions are correctly rounded (0.5 ULP error across all inputs), and the GPU build provides device-callable `sin/cos/memcpy/printf` via an RPC ring-buffer protocol for NVPTX and AMDGPU kernels. **OpenMP and offload** (Chapter 124) closes the part with the most complex runtime: `libomp` for host fork-join threading, the device runtime library (`DeviceRTL`) embedded as bitcode in host ELFs, and LLVM 22's stabilized `liboffload` API that abstracts CUDA, ROCm, and OpenCL behind a common `olInit/olLaunchKernel` surface exposed through the NextGen plugin architecture.

---

## Key Concepts Introduced

- **compiler-rt builtins ABI**: All builtins are pure C ABI (`__udivsi3`, `__mulsf3`, etc.) with no libc or C++ dependency; selected at link time via `--rtlib=compiler-rt` and named `libclang_rt.builtins-<arch>.a`.
- **Soft-float naming convention**: The `sf/df/tf` suffix family (`__addsf3`, `__muldf3`) and ARM EABI `__aeabi_*` wrappers cover targets from Cortex-M0 to ARMv9, with BFloat16/FP16 extensions in LLVM 22.
- **LLVM profile runtime (`libclang_rt.profile`)**: Counters stored in `__llvm_prf_cnts` ELF sections with three update modes (atomic, prefer-atomic, single); continuous mmap mode enables crash-safe PGO for long-running servers.
- **`_Unwind_*` Itanium ABI**: The two-phase unwind protocol (search then cleanup) embodied in `_Unwind_RaiseException`, `_Unwind_Resume`, and personality function callbacks; implemented by libunwind's `UnwindCursor<A,R>`.
- **DWARF CFI and `.eh_frame`**: CIE/FDE binary encoding of a stack-unwinding virtual machine; `.eh_frame_hdr` provides an O(log n) binary-search index keyed on program counter; ARM EHABI uses `.ARM.exidx`/`.ARM.extab` compact 32-bit encodings instead.
- **libc++ inline-namespace ABI versioning**: All symbols are emitted as `std::__2::` (version 2) rather than `std::`, so version mismatches fail at link time rather than producing silent binary corruption at runtime.
- **libc++ hardening framework**: Four levels (`none`, `fast`, `extensive`, `debug`) enabling progressively more expensive runtime bounds and invariant checks, with a replaceable `__libcpp_verbose_abort` handler; `fast` mode adds under 1% overhead.
- **`__cxa_exception` header and the personality function**: The `__cxa_exception` struct immediately precedes the user exception object in memory; `__gxx_personality_v0` parses the `.gcc_except_table` LSDA per frame to find matching catch types and execute cleanups.
- **RTTI type info hierarchy**: `__shim_type_info` → `__si_class_type_info` → `__vmi_class_type_info` virtual dispatch chain; `__dynamic_cast` traverses `offset_to_top` fields in vtables to handle multiple and virtual inheritance.
- **Thread-safe static initialization guards**: `__cxa_guard_acquire/release` use a 64-bit guard variable with an in-progress bit and a mutex/futex to serialize concurrent C++11 local-static initialization.
- **LLVM-libc TableGen entrypoint pipeline**: `.td` entrypoint declarations → `HdrGen` reads platform `entrypoints.txt` → generates stripped `<string.h>` headers; each implementation lives in `LIBC_NAMESPACE_DECL` (`__llvm_libc_22`) with an alias to the bare C name.
- **Correctly rounded math functions**: LLVM-libc's `sinf/cosf/expf/logf` and double variants use Sollya-generated polynomials with extended-precision intermediates to guarantee 0.5 ULP error, honoring the current `fesetround` rounding mode.
- **GPU RPC ring buffer**: LLVM-libc device code writes I/O requests to a shared device-visible ring buffer; a host polling thread reads requests and executes `printf`/`scanf` on behalf of GPU threads, enabling standard C I/O from NVPTX/AMDGPU kernels.
- **liboffload unified offload API**: LLVM 22's stable `Offload.h` provides a device-agnostic `olInit/olAllocDevice/olMemcpy/olLaunchKernel` surface; the NextGen plugin architecture loads backend-specific `.so` files (`liboffload_plugin_cuda.so`, `liboffload_plugin_amdgpu.so`) at runtime via `GenericPluginTy`.
- **OpenMP DeviceRTL and SPMD/Generic modes**: The bitcode device runtime (`DeviceRTL`) is embedded in host ELFs and provides `__kmpc_*` device implementations; SPMD mode maps all GPU threads to OpenMP threads directly for efficiency, while generic mode supports nested parallelism with a master-thread dispatch loop.

---

## How This Part Fits the Book

Part XVII is built upon the code-generation and LLVM IR foundations of Parts IV through VIII (especially the IR-level exception handling model in Chapter 26 and Clang's C++ ABI lowering in Chapter 42), and is enriched by the sanitizer and production-allocator coverage of the immediately preceding Part XVI (Chapters 110–115). The runtime libraries introduced here are in turn consumed by Part XVIII (Flang), which relies on `libomp` for OpenMP parallelism in Fortran code, by Part XXII (XLA/OpenXLA), which builds on the GPU offload infrastructure, and by Part XXIV (Verified Compilation), which references the formally verified compiler-rt and libunwind goals discussed in the roadmap sections of Chapters 119–120.

---

## Cross-Part Dependencies

- **Ch 26 (Part IV)** — Exception Handling in LLVM IR: the `invoke`/`landingpad`/`cleanupret` instructions generated there are consumed at runtime by libunwind (Ch 120) and libc++abi (Ch 122).
- **Ch 42 (Part VI)** — C++ ABI Lowering (Itanium): Clang emits the `__cxa_*` call sites and `.gcc_except_table` LSDA that libc++abi (Ch 122) interprets; the vtable layout described in Ch 42 is what `__dynamic_cast` traverses.
- **Ch 50 (Part VII)** — Clang as SYCL/OpenCL/OpenMP-Offload: the compilation side of the OpenMP offload pipeline that produces device bitcode embedded and loaded by the liboffload/DeviceRTL runtime (Ch 124).
- **Ch 102–103 (Part XV)** — NVPTX and AMDGPU targets: device-side code generation for the GPU kernels that DeviceRTL (Ch 124) and LLVM-libc GPU support (Ch 123) execute on.
- **Ch 110, Ch 112 (Part XVI)** — Sanitizers and Scudo: sanitizer runtimes live in compiler-rt and share `sanitizer_common` (Ch 119); Scudo is optionally integrated as LLVM-libc's `malloc`/`free` backend (Ch 123).
- **Ch 115 (Part XVI)** — Source-Based Coverage: the `.profraw` serialization workflow and `llvm-cov` coverage mapping sections described there are produced by the profile instrumentation runtime in compiler-rt (Ch 119).
- **Ch 148 (Part XXIV)** — Alive2 Translation Validation: the R&D roadmap in Ch 119 and Ch 120 references Alive2 as a potential vehicle for formally verifying soft-float and DWARF parsing correctness.
- **Ch 155 (Part XXV)** — Contributing to LLVM Runtimes: the build and test workflows documented in this part (CMake runtime flags, `ninja check-cxx`, `ninja check-cxxabi`) are the operational baseline for the contribution workflow chapter.

---

## Navigation

- ← Part XVI — JIT, Sanitizers & Runtime Tooling
- → Part XVIII — Flang

---

*@copyright jreuben11*
