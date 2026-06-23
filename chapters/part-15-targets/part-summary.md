# Part XV — Targets — Part Summary

*This part dissects every major LLVM target backend — from the battle-tested x86 and AArch64 CPU backends through RISC-V's modular ISA, the diverse GPU pipelines of NVPTX, AMDGPU, SPIR-V, and DXIL, and finally to the virtual and embedded targets that define execution environments by verifier semantics rather than hardware circuits — equipping the reader to generate, debug, extend, and cross-compile for any platform the LLVM ecosystem supports.*

---

## Chapters in This Part

| Chapter | Title | Key Topic |
|---------|-------|-----------|
| 95 | The X86 Backend | Register hierarchy, AVX-512, APX, CET, DAG combining |
| 96 | The AArch64 Backend | NEON/SVE/SME, PAuth, BTI, MTE, GlobalISel pipeline |
| 97 | The 32-bit ARM Backend | A32/Thumb/Thumb-2 interworking, VFP/NEON, Cortex-M |
| 98 | The RISC-V Backend Architecture | Extension alphabet, relaxation, GlobalISel on RV64 |
| 99 | The RISC-V Vector Extension (RVV) | Scalable vectors, vtype/LMUL/SEW, loop vectorization |
| 100 | RISC-V Bit-Manipulation, Crypto, and Custom Extensions | Zb*, Zk*, CFI extensions, adding custom ISA extensions |
| 101 | PowerPC, SystemZ, MIPS, SPARC, and LoongArch | Server, mainframe, and legacy platform backends |
| 101b | The LoongArch Backend | LP64D ABI, LSX/LASX SIMD, TLSDESC, JITLink |
| 102 | NVPTX and the CUDA Path | PTX virtual ISA, address spaces, tensor core lowering |
| 103 | AMDGPU and the ROCm Path | SGPR/VGPR file, EXEC mask, HSA kernel ABI, wavefronts |
| 104 | The SPIR-V Backend | Logical IR, GlobalISel-based emission, capability tracking |
| 105 | DXIL and DirectX Shader Compilation | HLSL frontend, DXIL restrictions, Shader Model features |
| 106 | WebAssembly and BPF | Structured CFG, CO-RE relocations, verifier constraints |
| 107 | Embedded Targets | Cortex-M/RISC-V/AVR/MSP430, code size, bare-metal toolchain |

---

## Part Overview

Part XV is the implementation companion to Part XIV (Backend). Where Part XIV explains the architecture of LLVM's backend machinery — instruction selection pipelines, register allocation, scheduling, the MC layer — Part XV puts those mechanisms into concrete target-specific context, showing exactly how each target instantiates, specialises, and extends that infrastructure to generate correct and efficient code for radically different hardware realities.

The part opens with the three dominant CPU targets. Chapter 95 examines x86-64 in full depth: a 60-year ISA with a layered vector register hierarchy (XMM/YMM/ZMM), two distinct calling conventions (SysV AMD64 vs Win64), AVX-512's EVEX-encoded opmask predicates, Intel APX's 32-register GPR expansion and NDD 3-operand forms, and CET's dual defenses of shadow stack and indirect branch tracking. The X86 DAG combining infrastructure — load-op-store folding, LEA synthesis, hierarchical shuffle decomposition — is examined in detail because it represents decades of accumulated micro-optimization. Chapter 96 covers AArch64, now the dominant RISC ISA, from NEON's fixed 128-bit SIMD through SVE/SVE2's scalable-vector `<vscale x N x T>` types to the SME matrix tile register file; security features (PAuth, BTI, MTE, LSE atomics) are treated as first-class backend concerns because they are emitted at every function boundary by modern toolchains. Chapter 97 completes the ARM family with the 32-bit backend, covering the three-way ISA mix of A32, Thumb-1, and Thumb-2, the interworking veneer mechanism, VFP/NEON coprocessors, the radically different constraints of ARMv7-M Cortex-M microcontrollers, and the Armv8.1-M PACBTI extension for safety-critical firmware.

The next cluster addresses RISC-V's modular ISA ecosystem across three chapters. Chapter 98 explains the base ISA selection (RV32I/RV64I), the extension alphabet and its mapping to LLVM subtarget features, six ABI variants, pseudo-instruction expansion (LI/LA/CALL/TAIL), the linker relaxation pipeline that shrinks `AUIPC+JALR` to `JAL` after layout, and GlobalISel's production-ready status on RV64GC. Chapter 99 is dedicated to the V vector extension: the vtype/SEW/LMUL configuration model, LLVM's `<vscale x N x T>` scalable-vector IR type, the `RISCVInsertVSETVLI` pass that eliminates redundant configuration changes, EVL-based tail-folding in the Loop Vectorizer, and the complete RVV intrinsic families. Chapter 100 covers the extension composability that makes RISC-V uniquely suited to specialised workloads: Zba address-generation instructions, Zbb/Zbc/Zbs bit manipulation, the Zk cryptography suite (AES, SHA-2, SM3/SM4), Zfh/Zfa floating-point additions, Zicfilp/Zicfiss hardware CFI, Zc* compressed extensions, vendor extensions from T-Head and SiFive, and a full step-by-step guide to adding a custom extension.

Chapter 101 surveys the remaining general-purpose CPU targets as a group — PowerPC (VSX, ELFv2, Power10 MMA), IBM SystemZ (z/Architecture vector ISA, HTM), MIPS (o32/n32/n64 ABIs, MSA SIMD), and SPARC (register windows) — plus a detailed treatment of LoongArch in both the survey chapter and the dedicated Chapter 101b, which covers the LA64 ISA, LP64D ABI, TLSDESC-based TLS, LSX/LASX SIMD, LLD integration, and JITLink support for the ORC JIT. Chapters 102 through 105 address the GPU and shader ecosystem. NVPTX (Chapter 102) explains PTX as a virtual ISA consumed by NVIDIA's driver, the CUDA thread hierarchy, address-space annotation, kernel metadata, tensor core intrinsics, and the full pass pipeline from IR to PTX text. AMDGPU (Chapter 103) is the most architecturally complex backend in the tree, requiring a dual SGPR/VGPR register file, a 64-entry EXEC mask that gates per-lane execution, wavefront-size-dependent code generation, HSA kernel descriptors, and GCN-specific correctness passes that cannot be reused from generic infrastructure. SPIR-V (Chapter 104) is the only backend that deliberately uses GlobalISel rather than SelectionDAG, because its logical-addressing type system has no register allocation and requires type reconstruction during emission; capability tracking, decoration, and the binary word-order module format are covered in detail. DXIL (Chapter 105) closes the GPU section with DirectX shader compilation: HLSL as a Clang language mode, the DXIL restriction set (no indirect calls, no recursion, typed resource handles via `dx.op`), Shader Model feature progression through SM 6.8 work graphs, and the relationship between the in-tree LLVM backend and Microsoft's standalone `dxc`.

The final pair of chapters addresses virtual and embedded targets. Chapter 106 covers WebAssembly — with its CFGStackify pass that converts arbitrary CFGs to structured control flow, RegStackify/ExplicitLocals stack discipline, SIMD128, WASI, threads/atomics, and the Wasm GC proposal — alongside eBPF, whose compiler cooperation with the kernel verifier requires CO-RE BTF relocations, the BPFAbstractMemberAccess pass, loop-depth constraints, and verifier-aware code generation. Chapter 107 closes the part with the embedded target landscape: Cortex-M Thumb-2 (vector table, linker scripts, TrustZone CMSE, code-size optimization), RISC-V RV32I/E bare-metal, AVR (16-bit with address-space complications), MSP430 (ultra-low-power with complex addressing modes), multilib selection, compiler-rt/picolibc integration, DWARF for constrained flash, and LLDB remote debugging.

---

## Key Concepts Introduced

- **Register hierarchy and sub-register aliasing**: How LLVM models overlapping register views (x86's XMM/YMM/ZMM, AArch64's Wn/Xn, ARM32's S/D/Q) through TableGen register class definitions and sub-register constraints.
- **Calling convention lowering**: Platform-specific argument/return-value classification (SysV AMD64, Win64, AAPCS64, AAPCS, LP64D RISC-V, ELFv2 PowerPC) implemented in `TargetLowering::LowerFormalArguments()` and Clang's `TargetInfo::classify()`.
- **SIMD instruction selection hierarchy**: The cascade of pattern-matching used to lower vector shuffles (x86 VPSHUFB/VPERM2F128/TBL/ZIP/UZP; NEON; RVV `vrgather`), and how type legalization widens, splits, or promotes vector types to match available ISA widths.
- **Scalable-vector IR type (`<vscale x N x T>`)**: LLVM's representation for hardware-length-agnostic vectors, used by SVE, RVV, and `vscale` intrinsic; drives tail-loop elimination in the Loop Vectorizer via EVL-based folding.
- **Linker relaxation**: RISC-V's post-layout optimization pipeline that shrinks conservative multi-instruction sequences (AUIPC+JALR → JAL; AUIPC+ADDI → ADDI via GP) after final symbol addresses are resolved, enabled by `R_RISCV_RELAX` marker relocations.
- **Hardware CFI (Control-Flow Integrity)**: Target-specific implementations — x86 CET (ENDBR64 IBT + shadow stack), AArch64 BTI + PAuth, Armv8.1-M PACBTI, RISC-V Zicfilp/Zicfiss — and how LLVM emits the required instrumentation in frame lowering and branch-target passes.
- **GPU address-space annotation**: How NVPTX (`.global`/`.shared`/`.local`/`.const`) and AMDGPU (flat/global/local/private/constant) address spaces are represented as LLVM IR address-space attributes and inferred by `NVPTXInferAddressSpaces` and `AMDGPUInferAddressSpaces`.
- **SGPR/VGPR dual register file and EXEC mask**: AMDGPU's unique two-file model where scalar (SGPR) registers hold wave-uniform values and vector (VGPR) registers hold per-lane values; the EXEC mask predicates all VGPR lanes, implementing SIMT divergence without branch hardware.
- **SPIR-V type system reconstruction**: The challenge of recovering explicit `OpType*` instructions from LLVM IR's opaque pointer model during emission; solved via `SPIRVGlobalRegistry` and type-annotation intrinsics injected during IRTranslation.
- **DXIL structural restrictions**: How DXIL constrains LLVM IR (no indirect calls, no recursion, no `alloca` outside entry, typed resource handles via `dx.op.*`) and why these constraints exist to guarantee driver-side JIT compilation with predictable bounds.
- **Wasm CFGStackify pass**: The algorithm that converts a general control-flow graph (with arbitrary goto targets) into WebAssembly's required structured control flow (nested `block`/`loop`/`if` constructs) while preserving semantics.
- **BPF CO-RE (Compile Once, Run Everywhere)**: The BTF-based relocation model that encodes struct field accesses symbolically at compile time, allowing libbpf to rewrite field offsets at load time for kernel version compatibility.
- **RISC-V extension composability and custom extensions**: The full pathway from `SubtargetFeature` definition in TableGen through instruction patterns, intrinsic declaration, Clang driver integration, and C header exposure, enabling per-workload ISA specialisation without forking the compiler.
- **Embedded bare-metal toolchain**: `-nostdlib` with custom linker scripts, `compiler-rt` AEABI builtins for soft-float and hardware-divide emulation, multilib selection, code-size optimization (`-Oz`, machine outliner, LTO), and DWARF generation for constrained flash targets.

---

## How This Part Fits the Book

Part XIV (Chapters 83–94) establishes the generic backend machinery — instruction selection (SelectionDAG and GlobalISel), register allocation, instruction scheduling, the MC layer, and the AsmPrinter — that every target in Part XV instantiates. Part XV is where those abstractions become concrete: each chapter shows exactly which `TargetLowering` hooks are overridden, which `TargetRegisterInfo.td` patterns are defined, and which target-specific machine passes are added to the pipeline. Part XVI (JIT, Sanitizers & Runtime Tooling), which follows immediately, builds on Part XV by examining how JIT compilation selects and configures these same target backends at runtime (ORC JIT's `TargetMachine` creation), how sanitizer instrumentation interacts with target ABI constraints (stack-frame layout for AddressSanitizer, EXEC-mask awareness for GPU sanitizers), and how runtime libraries like `compiler-rt` provide the builtins and support code that these backends depend on.

---

## Cross-Part Dependencies

The following chapters in other parts depend on or are enriched by material introduced in this part:

- Ch108 (Part XVI) — ORC JIT: uses `TargetMachine` for AArch64, x86-64, and RISC-V to compile IR to native code at runtime; JITLink's per-target ELF relocation handlers correspond directly to target-specific relocation types covered in Chs 95–101b.
- Ch111 (Part XVI) — Sanitizers: AddressSanitizer, HWAddressSanitizer, and MemTagSanitizer interact with AArch64 MTE (Ch96 §96.9) and GPU address-space semantics (Chs 102–103); shadow-stack sanitizers relate to x86 CET and AArch64 PAuth (Chs 95.5, 96.6).
- Ch113 (Part XVII) — compiler-rt: the AEABI soft-float builtins referenced in Ch97 and Ch107, the `__cpu_supports()` resolver used by x86 multiversioning (Ch95.7), and RISC-V libgcc-compatible builtins are all implemented in compiler-rt.
- Ch125–127 (Part XIX) — MLIR foundations: the SPIR-V dialect (Ch104 §104.10.5), NVVM dialect, ROCDL dialect, and WebAssembly targets all have corresponding MLIR dialects; Ch104–106 explain the LLVM target semantics that those dialects must accurately model.
- Ch130–131 (Part XX) — In-tree MLIR dialects: the MLIR `amdgpu` dialect maps directly to GCN MFMA instructions (Ch103 §103.5.3); the `arm_neon`, `arm_sve`, and `arm_sme` dialects correspond to Ch96 §96.3–96.5.
- Ch143 (Part XXI) — MLIR transformations: the MLIR loop vectorizer and tiling passes targeting RVV or SVE must respect the `<vscale x N x T>` type constraints and VSETVLI insertion requirements explained in Chs 98–99.
- Ch154 (Part XXII) — XLA/OpenXLA: code generation for TPU, NVPTX, and AMDGPU targets that XLA uses depends on the target-specific lowering paths covered in Chs 102–103.
- Ch171 (Part XXV) — Contribution guide: adding a new LLVM target follows the patterns established throughout Part XV (TableGen instruction definitions, `TargetLowering` subclass, MC layer, scheduling model, test infrastructure).
- Ch183 (Part XXVIII) — Rust language ecosystem: Rust's cross-compilation story for embedded Cortex-M (§97.5, Ch107) and RISC-V bare-metal (Ch107 §107.4) uses the same `-target thumbvXm-none-eabi` and `-target riscv32imac-none-elf` triples and multilib infrastructure.

---

## Navigation

- ← Part XIV — Backend
- → Part XVI — JIT, Sanitizers & Runtime Tooling

---

*@copyright jreuben11*
