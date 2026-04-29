# Appendix D — Migration Notes: LLVM 18 → 22

*Quick Reference | April 2026*

This appendix catalogs breaking API changes, behavioral changes, and deprecations across LLVM 18 through 22. Organized by release. For a complete list of changes, consult the release notes in `llvm/docs/ReleaseNotes.rst` for each version. Symbol names in `monospace` refer to the C++ API. IR-level changes affect `.ll`/`.bc` files.

---

## D.1 LLVM 18 → 19

### Opaque Pointers Now Default

**The single most impactful change in the 18→19 cycle.** Typed pointers (`i32*`, `%Foo*`) are no longer valid in LangRef; all pointers are now `ptr` or `ptr addrspace(N)`. The compatibility mode (`-opaque-pointers=0`) was removed.

**Impact on IR files:**
- Old: `%p = getelementptr i32, i32* %base, i64 %i`
- New: `%p = getelementptr i32, ptr %base, i64 %i`
- Old: `load i32, i32* %p`
- New: `load i32, ptr %p`

**C++ API changes:**
- `PointerType::getElementType()` removed — callers must track element type independently
- `GetElementPtrInst::getSourceElementType()` now requires the explicit type to be stored; this was previously inferrable from the pointer type
- `LoadInst::getPointerOperandType()` always returns `ptr`
- `Value::getType()` for pointer values always returns `ptr`; old code checking `->isPointerTy()` and casting to `PointerType` to get the element type must be rewritten

**Migration pattern:**
```cpp
// Before (typed pointers)
Type *EltTy = cast<PointerType>(Ptr->getType())->getElementType();

// After (opaque pointers)
// Track element type separately, e.g., from the alloca instruction type
Type *EltTy = cast<AllocaInst>(Ptr)->getAllocatedType();
// or from a GEP's source element type
Type *EltTy = GEP->getSourceElementType();
```

### New Pass Manager: Legacy PM Further Removed

The Legacy PM was retained only for codegen passes in LLVM 18. In LLVM 19 several more legacy analysis wrappers were removed. Code using `legacy::PassManager` for module-level or function-level IR optimization passes must migrate to `PassBuilder` + `ModulePassManager` / `FunctionPassManager`.

Key removed interfaces:
- `legacy::FunctionPassManager::add(Pass*)` for many standard passes — these passes now only register via `PassBuilder::registerPipelineParsingCallback`
- `createSROAPass()` free function removed; use `SROAPass{}` with new PM

### `scf.foreach_thread` → `scf.forall` (MLIR)

`scf.foreach_thread` was renamed `scf.forall` as part of the parallel-for unification. MLIR files using `scf.foreach_thread` must be updated:

```mlir
// Before
scf.foreach_thread (%i, %j) in (%n, %m) {
  ...
}

// After
scf.forall (%i, %j) in (%n, %m) {
  ...
}
```

Migration: run `mlir-opt --scf-foreach-thread-to-scf-for` (if available) or sed-replace.

### AArch64 SME (Scalable Matrix Extension)

SME dialect ops stabilized. The `arm_sme` dialect gained `arm_sme.tile_load`, `arm_sme.tile_store`, `arm_sme.outerproduct`, `arm_sme.zero`. The AArch64 backend added `SME` feature flag support for codegen.

### Other Changes

| Area | Change |
|---|---|
| `DIBuilder` | `createBasicType` now takes `DWARFSourceLanguage` for `DW_AT_language` |
| `IRMover` | Module linking now validates opaque pointer consistency more strictly |
| RISC-V | GlobalISel fully enabled at `-O0` |
| x86 | `x86_64-v4` microarchitecture level support (AVX-512 baseline) |
| Clang | `[[clang::musttail]]` attribute for enforced tail calls |
| MLIR `bufferization` | `BufferizationOptions::allowReturnAllocsFromLoops` added |
| MLIR ODS | `TypeConstraint` and `AttrConstraint` refactored; `Pred` mechanism expanded |

---

## D.2 LLVM 19 → 20

### `freeze` Instruction Canonicalization

`freeze` semantics were tightened. Previously `freeze(undef)` and `freeze(poison)` were treated identically. In LLVM 20, InstCombine performs more aggressive canonicalization:

- `freeze(freeze(x))` → `freeze(x)` (idempotent)
- `freeze(constant)` → `constant` (constants cannot be poison)
- Distributes over select: `freeze(select c, a, b)` may be distributed if c is not poison

**Pass behavior change:** Some canonicalizations that were safe under `undef` semantics may now fire under `poison` semantics. Code that generated IR with `undef` (deprecated) should switch to `poison` or `freeze`.

### `nsw`/`nuw` Propagation Through InstCombine

`nsw` (no signed wrap) and `nuw` (no unsigned wrap) flags are now more aggressively propagated through chains:
- `add nsw %a, add nsw %b, %c` can now propagate `nsw` on the outer add if the inner is provably nsw
- Some transformations that previously dropped these flags now preserve them — verify passes should no longer flag this as `nsw` elimination

### GlobalISel: x86-64 Stable

GlobalISel reached production status for x86-64 at `-O0`. New behavior:
- `clang -O0` on x86-64 defaults to GlobalISel instead of FastISel
- Pass `--fast-isel` to `llc` to force FastISel
- GlobalISel legalizer tables updated for all common patterns

### AArch64 PAC/BTI (Pointer Authentication / Branch Target Identification)

Landing pad MC support added in LLVM 19. The `llvm.aarch64.pacia`/`llvm.aarch64.autia` intrinsics are now generated by Clang for `-mbranch-protection=pac-ret`. LLVM MC emits `BTI c` landing pads for indirect call targets when `-mbranch-protection=bti` is set.

### MLIR: Properties System

`hasProperties` in ODS (`Op.td`) is now fully supported for ops with structured properties (not just attributes). The `Properties` struct allows typed, structured per-op data that differs from `DictionaryAttr`:

```tablegen
def MyOp : Op<...> {
  let arguments = (ins ...);
  let extraClassDefinition = [{
    // properties are accessed via getProperties() → MyOpProperties&
  }];
}
```

Migration: ops using `extraClassDeclaration` to store cached data should consider migrating to Properties.

### Other Changes

| Area | Change |
|---|---|
| `TargetTransformInfo` | `getUnrollingPreferences` signature changed; added `UCF` (unroll-and-jam) options |
| `MachineInstr` | `getNumMemOperands()` deprecated; use range-based `memoperands()` |
| MLIR `transform` | `transform.apply_patterns` op added as a simpler interface to PatternRewriter |
| MLIR `linalg` | `linalg.elemwise_unary/binary` removed; use `linalg.map` |
| Flang | HLFIR promoted to default IR; `-emit-fir` now emits HLFIR |
| NVPTX | PTX 8.0 support; `nvvm.mma.sync` intrinsics updated for Hopper |
| SPIR-V | SPIR-V 1.6 support added in `spirv` dialect |
| Clang | `__builtin_counted_by_ref` for bounds-safe flexible array access |

---

## D.3 LLVM 20 → 21

### `llvm::Function::getEntryCount()` Deprecated

Replaced by `llvm::ProfileCount`. This struct unifies instrumentation and sampling profile counts, and carries the `ProfileCountType` (Real vs Synthetic).

```cpp
// Before
Optional<uint64_t> Count = F.getEntryCount();

// After
Optional<ProfileCount> PC = F.getEntryCount();
if (PC) {
  uint64_t count = PC->getCount();
  bool isReal = PC->getType() == ProfileCountType::Real;
}
```

### `DIBuilder` DWARF 5 Name Index Changes

`DIBuilder::finalize()` now automatically generates `.debug_names` (the DWARF 5 name index) when `DW_AT_language` indicates C++14 or later and the DWARF version is 5. Previously this required explicit calls. Code that manually built name index entries must be updated; the automatic generation may conflict with manual construction.

Also: `DIBuilder::createNameSpace` now takes an optional `bool ExportSymbols` for DWARF 5 namespace export semantics.

### MLIR Properties: Deprecation of Some `attr-dict` Patterns

Following the Properties stabilization in LLVM 20, the use of `attr-dict` for storing structured op configuration that could be expressed as Properties is deprecated. The ODS codegen emits warnings for ops with `DictionaryAttr`-based fields that have a clear Properties equivalent.

### NVPTX PTX 8.5 and WGMMA Intrinsics

WGMMA (warpgroup-level matrix multiply-accumulate) intrinsics for NVIDIA Hopper (H100) are fully supported:
- `llvm.nvvm.wgmma.mma.async.*` family added
- PTX 8.5 feature support: `tcgen05` (tensor core gen 5) intrinsics
- `nvgpu` dialect: `nvgpu.warpgroup.mma`, `nvgpu.warpgroup.mma.setup_accumulator`

### AMDGPU GFX12 (RDNA4)

Full codegen support for GFX12 (`-mcpu=gfx1200`). New features:
- `VOPD` (Dual Issue VALU) scheduling
- Updated flat/global atomics encoding
- New image intrinsics for GFX12 sample format
- `rocdl` dialect updates for GFX12 specifics

### Other Changes

| Area | Change |
|---|---|
| `MemorySSA` | `MemoryUseOrDef::getOptimized()` deprecated; use `getOptimizedAccessType()` |
| `SCEVExpander` | `expand()` now takes `Loop*` parameter for loop-aware expansion |
| MLIR `vector` | `vector.mask` op stabilized; replaces `vector.create_mask`+`select` pattern |
| MLIR `gpu` | `gpu.binary`/`gpu.object` ops added for serialized GPU binaries |
| Clang | `_BitInt(N)` extended precision arithmetic fully supported through LLVM IR |
| LLD | RISC-V linker relaxation for `TLSDESC` |
| `TableGen` | `defset` keyword added for defining sets of records |

---

## D.4 LLVM 21 → 22

### `SelectionDAGTargetInfo` API Changes

`SelectionDAGTargetInfo::EmitTargetCodeForMemcpy` and related functions changed signature to pass `MachineFunction&` explicitly:

```cpp
// Before
SDValue EmitTargetCodeForMemcpy(SelectionDAG &DAG, const SDLoc &dl,
    SDValue Chain, SDValue Dst, SDValue Src, SDValue Size, unsigned Align,
    bool isVolatile, bool AlwaysInline, MachinePointerInfo DstPtrInfo,
    MachinePointerInfo SrcPtrInfo) const;

// After (LLVM 22)
SDValue EmitTargetCodeForMemcpy(SelectionDAG &DAG, MachineFunction &MF,
    const SDLoc &dl, SDValue Chain, SDValue Dst, SDValue Src,
    SDValue Size, Align Alignment, bool isVolatile, bool AlwaysInline,
    MachinePointerInfo DstPtrInfo, MachinePointerInfo SrcPtrInfo) const;
```

All downstream targets (`X86`, `AArch64`, `ARM`, `PowerPC`, etc.) updated.

### `LLVMContext::enableOpaquePointers()` Removed

This function was deprecated since LLVM 17 (when opaque pointers became opt-in) and now removed entirely. Opaque pointers are always enabled. Calling this function causes a compile error — remove all calls.

### MLIR Bytecode Version 6

The MLIR bytecode format incremented to version 6. Files produced by LLVM 22 `mlir-opt --emit-bytecode` are not backward-compatible with LLVM < 21. Version 6 changes:
- `BytecodeDialectInterface::getVersion()` added — dialects must implement this to support versioned bytecode
- Forward-compatible encoding for properties (from LLVM 20+ system)
- Compressed attribute storage for repeated integer attributes

Migration: add `getVersion()` to any custom dialect's `BytecodeDialectInterface`:
```cpp
int64_t getVersion() const override { return 1; }
```

### Flang: HLFIR Fully Default; `-emit-fir` Deprecated

`-emit-fir` now silently emits HLFIR. Explicit `-emit-hlfir` is the canonical form. FIR (without high-level constructs) is only accessible through internal passes. Code that processes FIR directly (e.g., LLVM IR-generation pipelines) should ensure they run HLFIR lowering passes first.

### Xtensa Target Merged

`llvm/lib/Target/Xtensa` is now in-tree. Target triple: `xtensa-unknown-elf`. Build with `-DLLVM_TARGETS_TO_BUILD="...,Xtensa"`. Initial support covers:
- Xtensa ISA instruction encoding/decoding
- Basic register allocation and scheduling
- ABI for Xtensa Windowed Register Option

### `MachineInstr::getDebugVariable()` API Change

```cpp
// Before
const DILocalVariable *getDebugVariable() const;

// After
DILocalVariable *getDebugVariable() const;  // non-const; removed const-ptr
```

Also: `MachineInstr::getDebugLoc()` now returns `DebugLoc` (previously `const DebugLoc&`); callers that stored the reference must copy.

### Other Changes

| Area | Change |
|---|---|
| `RegAllocGreedy` | Priority heuristic refactored; custom `RegAllocEvictionAdvisor` interface stabilized for ML-guided regalloc |
| `MemoryLocation` | `getForLoad()`/`getForStore()` now handle scalable vectors; callers that assumed fixed size may need updates |
| MLIR `transform` | `transform.tile_using_for` and `transform.tile_using_forall` unified; old names kept as aliases |
| MLIR `arith` | `arith.addui_extended` and `arith.mului_extended` added for carry-based arithmetic |
| NVPTX | `cp.async.bulk` (TMA bulk copy) fully supported via `nvgpu.tma.async.load/store` |
| AMDGPU | GFX940 (MI300) improvements; `buffer_atomic_cmpswap_x2` support |
| Clang | `[[clang::lifetimebound]]` and pointer provenance attributes promoted |
| LLD | COFF ARM64EC (Emulation Compatible) support for Windows on ARM64 |
| `llvm-readobj` | `--decompress` flag to decompress compressed debug sections inline |
| `clangd` | Module map support improved for C++20 modules; `compile_commands.json` generation from CMake improved |

---

## D.5 Compatibility Matrix

| Feature | LLVM 18 | LLVM 19 | LLVM 20 | LLVM 21 | LLVM 22 |
|---|---|---|---|---|---|
| Typed pointers | compat mode | removed | removed | removed | removed |
| Legacy PM for IR opts | deprecated | further removed | removed | removed | removed |
| `scf.foreach_thread` | available | deprecated | removed | removed | removed |
| Opaque pointers default | opt-in | **default** | default | default | default |
| MLIR Properties | experimental | stable | stable | deprecated old patterns | full |
| MLIR Bytecode version | 4 | 4 | 5 | 5 | **6** |
| HLFIR default in Flang | opt-in | opt-in | opt-in | default | `-emit-fir` deprecated |
| GlobalISel x86-64 O0 | FastISel | **GlobalISel** | GlobalISel | GlobalISel | GlobalISel |
| Xtensa target | out-of-tree | out-of-tree | out-of-tree | out-of-tree | **in-tree** |

---

## D.6 Automated Migration Aids

| Tool | Purpose |
|---|---|
| `clang-tidy -checks=modernize-*` | Automated C++ modernizations; some LLVM API migrations covered |
| `llvm/utils/update_test_checks.py` | Regenerate FileCheck patterns in LLVM test files after IR changes |
| `llvm/utils/update_cc_test_checks.py` | Regenerate Clang codegen test checks |
| `llvm/utils/update_mir_test_checks.py` | Regenerate MIR test checks |
| `mlir/utils/update_filecheck_checks.py` | Regenerate MLIR FileCheck patterns |
| `sed -i 's/i32\*/ptr/g; s/i64\*/ptr/g'` | Quick sed to strip typed pointer syntax from IR (must verify GEP source types manually) |
| `opt --verify-uselistorder` | Verify use-list order stability (for bitcode reproducibility) |
| Custom `llvm-project/llvm/utils/migration/` scripts | Version-specific migration helpers when available |
