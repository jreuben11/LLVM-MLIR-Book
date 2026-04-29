# Chapter 109 — JITLink

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

When ORC v2 needed a linker, it could not simply reuse `RuntimeDyld`. That code had accumulated years of single-threaded assumptions, ELF-centric data structures, and a fixup model that was incompatible with W^X memory policies on Apple Silicon. JITLink was written from scratch to be the in-process linker that production JIT compilers deserve: platform-native (ELF, MachO, COFF), fully concurrent, extensible through a clean plugin architecture, and built around an abstract graph representation that enables multi-pass optimization before code is placed in executable memory. Every object file that ORC's `ObjectLinkingLayer` links passes through JITLink's pipeline — from the `LinkGraph` representation through GOT/PLT synthesis, relocation fixup, and permission-respecting memory management. Understanding JITLink means understanding what happens between "IR compiled to object bytes" and "function pointer ready to call."

---

## 109.1 Motivation: Replacing RuntimeDyld

### 109.1.1 RuntimeDyld's Limitations

`RuntimeDyld` (`llvm/lib/ExecutionEngine/RuntimeDyld/`) was written alongside MCJIT and carried several architectural constraints:

**Monolithic fixup model**: Relocations were processed in a single pass over raw section bytes. There was no intermediate representation — once bytes were handed to RuntimeDyld, there was no point to insert passes, inspect the relocation graph, or apply transformations. GOT/PLT synthesis had to happen before parsing, which required ad-hoc format-specific logic scattered throughout the code.

**Write-then-execute assumption**: RuntimeDyld allocated memory via `sys::Memory::allocateMappedMemory`, wrote relocation data, then called `finalizeMemory`. On macOS with Apple Silicon, the hardware enforces W^X (write XOR execute) at the page level — a page that was ever writable cannot be made executable without a privilege transition via `MAP_JIT`. RuntimeDyld had no mechanism for this.

**ELF-centricity**: The ELF path was the most complete; MachO support was limited to basic `x86_64` relocations; COFF was incomplete. Adding AArch64 MachO required surgery throughout the code.

**No plugin architecture**: There was no clean way to register post-link passes for EH frame registration, debug info notification, or perf event emission. All these were hardcoded in the RuntimeDyld implementation.

**Thread-safety**: RuntimeDyld was protected by a single mutex. Concurrent linking of two independent object files was serialized — a critical performance bottleneck for multi-threaded ORC JITs.

### 109.1.2 JITLink's Design Goals

JITLink (`llvm/lib/ExecutionEngine/JITLink/`) addresses each limitation with a clean design:

- **`LinkGraph`**: a target-independent intermediate representation of a relocatable object file, amenable to multi-pass transformation and inspection
- **Plugin architecture**: `JITLinkPlugin` with well-defined hooks for pre-prune, post-prune, and post-fixup passes
- **W^X-safe**: the `JITLinkMemoryManager` interface supports a two-phase commit model — allocate writable, apply fixups, then transition to executable — with `MAP_JIT` support on macOS
- **Concurrent-safe**: each link graph is a private data structure; concurrent link operations are fully independent with no shared mutable state
- **Platform-native parsers**: ELF, MachO, and COFF parsers produce the same `LinkGraph` representation

```
llvm/lib/ExecutionEngine/JITLink/          # Core implementation
llvm/include/llvm/ExecutionEngine/JITLink/ # Public headers (LinkGraph.h, JITLink.h)
llvm/lib/ExecutionEngine/Orc/ObjectLinkingLayer.cpp  # ORC integration
compiler-rt/lib/orc/                        # EH frame registration runtime
llvm/tools/llvm-jitlink/                    # Standalone testing tool
```

---

## 109.2 The LinkGraph Model

### 109.2.1 Overview

[`LinkGraph`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/JITLink/JITLink.h#L900) is the central data structure of JITLink. It is a directed graph where:

- **Sections** group **Blocks** by name and memory permissions
- **Blocks** contain bytes (content) and a list of **Edges** (outbound relocations)
- **Symbols** name offsets within Blocks, or represent external/absolute targets
- **Edges** represent relocations: from a Block at some offset, to a target Symbol, with an addend

This representation is analogous to ELF's (sections, symbols, relocations) but fully abstracted from binary encoding. After parsing, all format-specific knowledge is gone — only the graph remains. Transformation passes operate entirely on the graph without knowing whether the input was ELF or MachO.

### 109.2.2 Section and Block

A `Section` has a name, `MemProt` (Read/Write/Execute flags), a section ordinal for deterministic ordering, and a set of `Block*`s. Sections correspond to ELF sections, MachO sections, or COFF sections, but the graph also contains synthetic sections created during linking (`.got`, `.plt`, range-extension stubs).

A [`Block`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/JITLink/JITLink.h#L412) is a contiguous range of bytes within a section:

| Field | Type | Description |
|-------|------|-------------|
| `Content` | `ArrayRef<char>` | The block's bytes; may point into the input object or a synthetic buffer |
| `Address` | `ExecutorAddr` | Initially invalid; assigned during allocation phase |
| `Alignment` | `uint64_t` | Required alignment when placed in memory |
| `AlignmentOffset` | `uint64_t` | Offset of the first byte from the aligned boundary |
| `Edges` | `EdgeVector` | List of relocations from this block |
| `IsZeroFill` | `bool` | True for BSS-like blocks (zero-initialized, no stored content) |

Unlike ELF sections (which can be monolithic), JITLink can split a section into multiple blocks for finer-grained dead-stripping. Each `.text` function can become its own block, referenced only by symbols; unreferenced blocks are pruned.

### 109.2.3 Symbol

A [`Symbol`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/JITLink/JITLink.h#L618) is a named point in the graph:

| Symbol type | Definition | Address |
|-------------|------------|---------|
| Defined | Has offset within a `Block` | Block.Address + Offset |
| External | No block; must be resolved via JITDylib | Assigned during symbol lookup phase |
| Absolute | Fixed address not in any block | Constant value |

Symbol flags (`JITSymbolFlags`) are a subset of ORC's symbol flags: `Exported`, `Callable`, `Weak`, `Common`. Exported symbols are made available to the ORC JITDylib; non-exported symbols are private to this link.

The symbol's `getName()` returns a `StringRef` into the `LinkGraph`'s string slab — a pooled allocation for all symbol name strings, avoiding per-symbol heap allocations.

### 109.2.4 Edge

An [`Edge`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/JITLink/JITLink.h#L314) represents one relocation:

| Field | Type | Description |
|-------|------|-------------|
| `Offset` | `uint32_t` | Byte offset within the parent block where the fixup is applied |
| `Target` | `Symbol *` | The relocation target symbol |
| `Addend` | `int64_t` | Relocation addend (e.g., the `+4` in `[sym+4]`) |
| `Kind` | `Edge::Kind` (uint8_t) | Target-specific relocation type |

Edge kinds are defined per-target:

```cpp
// From llvm/include/llvm/ExecutionEngine/JITLink/x86_64.h:
namespace x86_64 {
  enum EdgeKind : Edge::Kind {
    PCRel32 = 0,                // 32-bit PC-relative
    PCRel64,                    // 64-bit PC-relative
    Pointer32,                  // 32-bit absolute
    Pointer64,                  // 64-bit absolute
    PCRel32GOTLoad,             // GOT-indirect load (movq sym@GOTPCREL(%rip))
    PCRel32GOTLoadRelaxable,    // GOT-indirect, may be relaxed to direct
    PCRel32TLVPLoadRelaxable,   // Thread-local via GOT
    PCRel32PLTCall,             // Call via PLT (can be relaxed)
    // ... more
  };
}

// From llvm/include/llvm/ExecutionEngine/JITLink/aarch64.h:
namespace aarch64 {
  enum EdgeKind : Edge::Kind {
    Branch26 = 0,           // B / BL instruction (±128MB)
    MoveWide16,             // MOVZ/MOVK immediate
    Page21,                 // ADRP (21-bit page offset)
    PageOffset12,           // ADD/LDR 12-bit page offset
    PageOffset12Anon,       // PAGE_OFFSET with anonymous symbol
    GOTPage21,              // ADRP to GOT entry
    GOTPageOffset12,        // LDR from GOT entry
    LDRLiteral32,           // LDR literal (PC-relative 32-bit)
    Delta32,                // 32-bit delta between two addresses
    Delta64,                // 64-bit delta
    NegDelta32,             // Negative 32-bit delta
    // ... more
  };
}
```

### 109.2.5 Graph Traversal

```cpp
// Full traversal of a LinkGraph:
void inspectGraph(const LinkGraph &G) {
  llvm::dbgs() << "Graph: " << G.getName()
               << " triple=" << G.getTargetTriple().str() << "\n";
  
  for (auto &Sec : G.sections()) {
    llvm::dbgs() << "  Section: " << Sec.getName()
                 << " prot=";
    if (Sec.getMemProt() & MemProt::Read)  llvm::dbgs() << "R";
    if (Sec.getMemProt() & MemProt::Write) llvm::dbgs() << "W";
    if (Sec.getMemProt() & MemProt::Exec)  llvm::dbgs() << "X";
    llvm::dbgs() << "\n";

    for (auto *B : Sec.blocks()) {
      llvm::dbgs() << "    Block @ " << B->getAddress()
                   << " size=" << B->getSize()
                   << " align=" << B->getAlignment() << "\n";
      for (auto &E : B->edges()) {
        llvm::dbgs() << "      Edge @+" << E.getOffset()
                     << " kind=" << (int)E.getKind()
                     << " -> " << E.getTarget().getName()
                     << " + " << E.getAddend() << "\n";
      }
    }
    
    for (auto *S : Sec.symbols()) {
      llvm::dbgs() << "    Symbol: " << S->getName()
                   << " @ " << S->getAddress()
                   << (S->isCallable() ? " [callable]" : "")
                   << (S->isWeak() ? " [weak]" : "") << "\n";
    }
  }
  
  llvm::dbgs() << "  External symbols:\n";
  for (auto *S : G.external_symbols())
    llvm::dbgs() << "    " << S->getName() << "\n";
    
  llvm::dbgs() << "  Absolute symbols:\n";
  for (auto *S : G.absolute_symbols())
    llvm::dbgs() << "    " << S->getName()
                 << " = " << S->getAddress() << "\n";
}
```

---

## 109.3 JITLink Pipeline

The pipeline is implemented in [`JITLink.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/JITLink/JITLink.cpp) and driven by `JITLinkerBase::linkPhase1/2/3`. It proceeds in numbered phases:

```
Object file bytes (MemoryBuffer)
      │
      ▼
Phase 1: PARSE
  Format-specific parser (ELF/MachO/COFF) → LinkGraph
      │
      ▼
Phase 2: PRE-PRUNE PLUGIN PASSES
  JITLinkPlugin::modifyPassConfig() (each plugin adds passes)
  Passes run: markAllSymbolsLive (optional), custom transforms
      │
      ▼
Phase 3: PRUNE (dead-stripping)
  Remove blocks/symbols not reachable from exported symbols
      │
      ▼
Phase 4: POST-PRUNE PLUGIN PASSES
  GOT/PLT synthesis, relaxation, thunk insertion
  Target-specific passes (ELF_x86_64_GOTAndStubsBuilder, etc.)
      │
      ▼
Phase 5: ALLOCATE
  JITLinkMemoryManager::allocate() → assign addresses to all blocks
  External symbols looked up via JITDylib (ORC session)
      │
      ▼
Phase 6: APPLY FIXUPS
  For each edge in each block: compute fixup value, write to block content
  Range check: verify offsets fit in relocation field widths
      │
      ▼
Phase 7: POST-FIXUP PLUGIN PASSES
  EHFrameRegistrationPlugin, DebugObjectManagerPlugin, PerfSupportPlugin
      │
      ▼
Phase 8: FINALIZE
  JITLinkMemoryManager::InFlightAlloc::finalize()
  Transitions memory from W (writable) to RX (readable+executable) or R
      │
      ▼
Phase 9: NOTIFY ORC
  MaterializationResponsibility::notifyResolved(symbols)
  MaterializationResponsibility::notifyEmitted()
  ORC lookup callers unblocked
```

### 109.3.1 Phase 1: Parsing

Format-specific builders read binary data and call `LinkGraph` factory methods:

- [`ELFLinkGraphBuilder_x86_64`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/JITLink/ELF_x86_64.cpp): reads ELF64LE, maps `SHT_RELA` entries to `Edge` objects
- [`MachOLinkGraphBuilder_arm64`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/JITLink/MachO_arm64.cpp): reads MachO arm64 Mach-O objects, handles `ARM64_RELOC_*` types
- [`COFFLinkGraphBuilder_x86_64`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/JITLink/COFF_x86_64.cpp): reads COFF64, maps `IMAGE_REL_AMD64_*` types

Each parser maps format-specific relocation types to `Edge::Kind` values and identifies special sections (`.eh_frame`, DWARF, ObjC metadata, TLS) for post-processing.

### 109.3.2 Phase 3: Dead-Stripping (Pruning)

Pruning removes blocks and symbols not reachable from "roots" — symbols that are exported or explicitly marked live. The algorithm:

1. Initialize the live set with all exported symbols and all blocks they reference
2. For each live block, add all blocks referenced by its edges to the live set
3. Repeat until fixpoint
4. Remove all blocks not in the live set; remove symbols whose defining blocks were removed

JITLink's pruning is more aggressive than static linkers because JIT compilation typically compiles one function at a time. A compiler frontend may emit helper stubs or cold error paths in the same object file; if the function under compilation doesn't reference them, pruning eliminates them without requiring LTO.

`markAllSymbolsLive(G)` is a utility that marks every defined symbol as live, effectively disabling pruning — useful for debugging.

### 109.3.3 Phase 4: Post-Prune Passes

After pruning, passes can add new blocks and edges. These passes run via the `PassConfiguration::PostPrunePasses` list:

```cpp
// PassConfiguration is built by JITLinkerBase and populated by plugins:
struct PassConfiguration {
  LinkGraphPassList PrePrunePasses;
  LinkGraphPassList PostPrunePasses;    // GOT/PLT synthesis goes here
  LinkGraphPassList PostAllocationPasses;
  LinkGraphPassList PreFixupPasses;
  LinkGraphPassList PostFixupPasses;    // EH frame registration goes here
};
```

---

## 109.4 GOT and PLT Synthesis

### 109.4.1 Why JIT Needs GOT/PLT

In a static link, GOT entries and PLT stubs are created once when the final binary is linked. In a JIT, each link session places code at freshly mmap'd addresses; external function addresses are resolved at JIT-link time, not at static link time. JITLink creates GOT entries and PLT stubs synthetically, on demand, in every link that needs them.

### 109.4.2 GOT Entry Creation

When the post-prune pass encounters an edge with a GOT-requesting kind (e.g., `x86_64::PCRel32GOTLoad`), it:

1. Checks whether the target already has a GOT entry in the synthetic `.got` section
2. If not: creates a new 8-byte `Block` in `.got` with a `Pointer64` edge pointing to the target
3. Creates a `Symbol` naming the GOT entry
4. Rewrites the original edge to reference the new GOT symbol with `PCRel32` kind

```
Before GOT synthesis (from parser output):
  .text block: Edge{Offset=7, Kind=PCRel32GOTLoad, Target=external_printf, Addend=-4}

After GOT synthesis (post-prune pass):
  .text block: Edge{Offset=7, Kind=PCRel32, Target=got_entry_printf, Addend=-4}
  .got block:  Edge{Offset=0, Kind=Pointer64, Target=external_printf, Addend=0}
```

The GOT block's `Pointer64` edge is resolved during the allocation phase when the external symbol `printf` is looked up via the JITDylib.

[`ELF_x86_64_GOTAndStubsBuilder`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/JITLink/ELF_x86_64.cpp) is the concrete implementation. It handles:
- `R_X86_64_GOTPCREL` → `PCRel32GOTLoad` edge → synthesize GOT entry
- `R_X86_64_REX_GOTPCRELX` → `PCRel32GOTLoadRelaxable` → GOT entry, eligible for relaxation
- `R_X86_64_PLT32` → `PCRel32PLTCall` → PLT stub if needed, or direct call if target is local

### 109.4.3 PLT Stub Creation

PLT stubs are synthetic executable blocks that implement indirect calls for external functions. The x86_64 PLT stub (6 bytes):

```asm
; .plt stub block for printf:
ff 25 xx xx xx xx   ; jmpq *[rip + got_entry_printf_offset]
```

The stub has:
- A `PCRel32` edge at offset 2 pointing to the GOT entry (to fill in `xx xx xx xx`)
- A symbol exported as `printf@plt` (for local calls to use)

On AArch64, the stub is larger (12 bytes) due to ADRP's limited range:

```asm
; .plt stub for printf (AArch64):
adrp x16, :got_page:printf     ; GOTPage21 edge
ldr  x16, [x16, :got_pageoff:printf]  ; GOTPageOffset12 edge
br   x16
```

### 109.4.4 GOT Relaxation

JITLink can eliminate unnecessary GOT indirections through relaxation. If a `PCRel32GOTLoadRelaxable` edge targets a defined symbol in the same link unit and the reference fits in a direct PC-relative encoding, JITLink rewrites the instruction:

```asm
; Before relaxation (with GOT):
48 8b 05 xx xx xx xx    ; movq symbol@GOTPCREL(%rip), %rax

; After relaxation (direct LEA):
48 8d 05 xx xx xx xx    ; leaq symbol(%rip), %rax
```

Or for call references:

```asm
; Before:
ff 15 xx xx xx xx       ; callq *symbol@GOTPCREL(%rip)

; After:
e8 xx xx xx xx          ; callq symbol@PLT (direct)
90                      ; nop
```

The `optimizeGOTAndStubsX86_64` pass ([`ELF_x86_64.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/JITLink/ELF_x86_64.cpp)) handles x86_64 relaxation; similar passes exist for AArch64.

---

## 109.5 Memory Management

### 109.5.1 JITLinkMemoryManager Interface

[`JITLinkMemoryManager`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/include/llvm/ExecutionEngine/JITLink/JITLinkMemoryManager.h) is the abstract interface for all JITLink memory allocation. It has two key methods:

```cpp
class JITLinkMemoryManager {
public:
  // Allocate memory for the LinkGraph's sections.
  // The callback receives an InFlightAlloc or an error.
  // After allocation, all Block addresses are set.
  virtual void allocate(const JITLinkDylib *JD, LinkGraph &G,
                        OnAllocatedFunction OnAllocated) = 0;
};

class InFlightAlloc {
public:
  // Finalize: apply permissions (W→RX for .text, W→R for .rodata, etc.)
  virtual void finalize(OnFinalizedFunction OnFinalized) = 0;
  
  // Abandon: release pages without finalizing (used on error path)
  virtual void abandon(OnAbandonedFunction OnAbandoned) = 0;
};
```

### 109.5.2 InProcessMemoryManager

[`InProcessMemoryManager`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/JITLink/InProcessMemoryManager.cpp) allocates memory in the current process. It maintains a slab pool to amortize `mmap` system call overhead:

```cpp
auto MemMgr = std::make_unique<InProcessMemoryManager>(
    /*PageSize=*/sys::Process::getPageSizeEstimate());
```

Internally, `InProcessMemoryManager` uses a `BumpPtrAllocator` over `mmap(MAP_ANONYMOUS)` slabs. Executable sections are mapped as `PROT_READ|PROT_WRITE` initially; after fixups, `mprotect(PROT_READ|PROT_EXEC)` is called. Data sections receive `PROT_READ` for `.rodata` or retain `PROT_READ|PROT_WRITE` for `.data`/`.bss`.

The allocator respects section alignment requirements from `Block::getAlignment()`, inserting padding between sections as necessary.

### 109.5.3 W^X Compliance on Apple Silicon

Apple Silicon (M1/M2/M3) enforces hardware W^X via PAC (Pointer Authentication Code) and memory-mapped I/O restrictions. The correct sequence for allocating JIT memory on macOS arm64:

```cpp
// 1. Map with MAP_JIT (required for JIT memory on macOS 10.14+):
void *p = mmap(nullptr, size,
               PROT_READ | PROT_WRITE,
               MAP_PRIVATE | MAP_ANONYMOUS | MAP_JIT,
               -1, 0);

// 2. Write phase — call pthread_jit_write_protect_np(0) to enable writes:
pthread_jit_write_protect_np(0);
// ... write object code and apply fixups to p ...

// 3. Execute phase — call pthread_jit_write_protect_np(1) to enable execution:
pthread_jit_write_protect_np(1);
// ... now [p] is readable + executable, not writable ...

// Note: this is a per-thread setting; each thread that writes JIT memory
// must call pthread_jit_write_protect_np(0) before writing.
```

[`InProcessMemoryManager`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/JITLink/InProcessMemoryManager.cpp) handles this via `sys::Memory::setExecutable()` which calls the appropriate platform-specific API.

### 109.5.4 MapperJITLinkMemoryManager

[`MapperJITLinkMemoryManager`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/MapperJITLinkMemoryManager.cpp) is used for out-of-process JIT. It communicates with the executor via the `ExecutorProcessControl` memory write API:

1. Compiler requests pages via EPC RPC → executor calls `mmap`, returns addresses
2. Compiler writes fixup'd bytes via EPC memory-write calls → bytes land in executor process
3. Compiler sends a finalize RPC → executor calls `mprotect` to set permissions
4. Compiler can send a deallocate RPC → executor calls `munmap`

The entire protocol is multiplexed over the `SimpleRemoteEPC` communication channel (see Chapter 108, Section 108.7).

---

## 109.6 Platform-Specific Support

### 109.6.1 ELF x86_64

The ELF x86_64 backend ([`ELF_x86_64.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/JITLink/ELF_x86_64.cpp)) handles the full range of x86_64 ELF relocations relevant to JIT:

| ELF Relocation | JITLink Edge Kind | Fixup Formula |
|---------------|-------------------|---------------|
| `R_X86_64_64` | `Pointer64` | `S + A` (absolute 64-bit) |
| `R_X86_64_PC32` | `PCRel32` | `S + A - P` (32-bit PC-relative) |
| `R_X86_64_PLT32` | `PCRel32PLTCall` | `S + A - P` (via PLT if external) |
| `R_X86_64_GOTPCREL` | `PCRel32GOTLoad` | `GOT[S] + A - P` |
| `R_X86_64_REX_GOTPCRELX` | `PCRel32GOTLoadRelaxable` | `GOT[S] + A - P` (or relaxed) |
| `R_X86_64_32` | `Pointer32` | `S + A` (zero-extended) |
| `R_X86_64_32S` | `Pointer32Signed` | `S + A` (sign-extended) |
| `R_X86_64_GOTOFF64` | `GOTOffset64` | `S + A - GOT` |
| `R_X86_64_TPOFF64` | `Delta64` (TLS) | TLS thread-pointer-relative |

Where `S` = symbol value, `A` = addend, `P` = fixup location address. Range checking: `PCRel32` must satisfy `|S + A - P - 4| <= 2^31 - 1`; JITLink reports `JITLinkError` if the range is exceeded (unlike static linkers that may silently truncate).

### 109.6.2 ELF AArch64

The AArch64 backend ([`ELF_aarch64.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/JITLink/ELF_aarch64.cpp)) handles AArch64's page-based addressing model.

**ADRP/ADD instruction pairs**: AArch64 code accesses data via a two-instruction sequence: `ADRP Xn, page(sym)` (loads the 4KB page address) followed by `ADD Xn, Xn, pageoff(sym)` or `LDR Xn, [Xn, pageoff(sym)]` (adds the within-page offset). JITLink represents this as two separate edges:

```asm
adrp x0, :pg_hi21:target    ; Page21 edge → write ADRP immediate
ldr  x0, [x0, :lo12:target] ; PageOffset12 edge → write LDR offset
```

The `Page21` edge writes the 21-bit page-relative value into bits [23:5] of the ADRP encoding. The `PageOffset12` edge writes bits [11:0] of the page offset into bits [21:10] of the ADD/LDR encoding.

**Range-extension thunks**: AArch64 `B`/`BL` instructions have a ±128MB range (26-bit signed offset in units of 4 bytes). JIT-compiled functions may be mapped far from runtime library functions. JITLink detects `Branch26` edges that overflow and inserts thunks:

```cpp
// ELF_aarch64.cpp: thunk insertion in postPrunePasses:
Error createThunks(LinkGraph &G) {
  for (auto &Sec : G.sections()) {
    for (auto *B : Sec.blocks()) {
      for (auto &E : B->edges()) {
        if (E.getKind() != aarch64::Branch26) continue;
        
        // Check if target is in range:
        int64_t Offset = E.getTarget().getAddress().getValue()
                        - (B->getAddress() + E.getOffset()).getValue();
        if (Offset >= -(1<<27) && Offset < (1<<27)) continue;
        
        // Insert thunk:
        auto *ThunkBlock = createThunkBlock(G, E.getTarget());
        auto *ThunkSym = G.addDefinedSymbol(*ThunkBlock, 0, ThunkName,
                                            ThunkBlock->getSize(),
                                            JITSymbolFlags::Callable, true);
        E.setTarget(*ThunkSym);
      }
    }
  }
  return Error::success();
}
```

The thunk is a three-instruction sequence using `ADRP`+`ADD`+`BR` that can reach any 64-bit address.

### 109.6.3 RISC-V

The RISC-V backend ([`ELF_riscv.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/JITLink/ELF_riscv.cpp)) handles RISC-V's complex relocation and relaxation rules. RISC-V static linkers aggressively relax instruction sequences. JITLink conservatively implements only the relaxations that are safe without full program knowledge:

| RISC-V Relocation | Action |
|-------------------|--------|
| `R_RISCV_CALL` | `AUIPC`+`JALR` pair — relax to `JAL` if within ±1MB |
| `R_RISCV_PCREL_HI20` | `AUIPC` instruction (page-offset high bits) |
| `R_RISCV_PCREL_LO12_I` | I-type instruction (low bits; must pair with HI20) |
| `R_RISCV_HI20` | `LUI` instruction |
| `R_RISCV_LO12_I` | I-type instruction (low bits with LUI pair) |
| `R_RISCV_GOT_HI20` | `AUIPC` for GOT entry |

The RISC-V backend uses a two-pass approach for `PCREL_HI20`/`PCREL_LO12` pairs: the HI20 relocation records the page-relative high bits, and the corresponding LO12 relocation refers back to the HI20 anchor symbol. JITLink resolves the LO12 by finding the corresponding HI20 edge and computing the offset relative to it.

### 109.6.4 COFF/PE (Windows)

The COFF backend ([`COFF.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/JITLink/COFF.cpp), [`COFF_x86_64.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/JITLink/COFF_x86_64.cpp)) supports JIT compilation on Windows:

| COFF Relocation | JITLink Edge Kind |
|----------------|-------------------|
| `IMAGE_REL_AMD64_REL32` | `PCRel32` |
| `IMAGE_REL_AMD64_REL32_1..5` | `PCRel32` with addend 1–5 |
| `IMAGE_REL_AMD64_ADDR64` | `Pointer64` |
| `IMAGE_REL_AMD64_ADDR32NB` | `Pointer32` (image-base-relative) |
| `IMAGE_REL_AMD64_SECTION` | Section index (for PDB/debug info) |
| `IMAGE_REL_AMD64_SECREL` | Section-relative offset |

The COFF backend also processes `.pdata` and `.xdata` sections for Windows Structured Exception Handling (SEH). `.pdata` contains `RUNTIME_FUNCTION` entries that describe the unwind info for each function; `.xdata` contains the `UNWIND_INFO` structure. JITLink links these sections and registers them with `RtlAddFunctionTable` for correct unwinding through JIT'd code.

### 109.6.5 MachO arm64

[`MachO_arm64.cpp`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/JITLink/MachO_arm64.cpp) handles Apple Silicon's specific requirements:

- **Compact unwind info**: MachO uses `__TEXT,__unwind_info` for stack unwinding. JITLink's `MachOPlatform` processes these sections and registers them with the libunwind personality function.
- **`__DATA_CONST` segment**: MachO on arm64 has a `__DATA_CONST` segment for read-only data that must be mapped read-write during linking (to apply relocations) then transitioned to read-only.
- **Objective-C metadata**: `__DATA,__objc_classlist`, `__DATA,__objc_catlist`, etc. JITLink's `MachOPlatform` plugin handles ObjC class registration at link time.
- **`B26` thunks**: same range-extension thunk mechanism as AArch64 ELF.

---

## 109.7 Plugin Architecture

### 109.7.1 JITLinkPlugin Interface

```cpp
class JITLinkPlugin {
public:
  virtual ~JITLinkPlugin() = default;

  // Called before any passes run; modify PassConfiguration to add passes:
  virtual void modifyPassConfig(MaterializationResponsibility &MR,
                                LinkGraph &G,
                                PassConfiguration &Config) {}

  // Called after the link graph is constructed (before pruning):
  virtual void notifyLoaded(MaterializationResponsibility &MR) {}

  // Called after all symbols are emitted to executor memory:
  virtual Error notifyEmitted(MaterializationResponsibility &MR) {
    return Error::success();
  }

  // Called on error (link abandoned):
  virtual Error notifyFailed(MaterializationResponsibility &MR) {
    return Error::success();
  }
};
```

Plugins are registered with `ObjectLinkingLayer::addPlugin`. Multiple plugins can be active simultaneously; their `modifyPassConfig` calls are invoked in registration order.

### 109.7.2 EHFrameRegistrationPlugin

[`EHFrameRegistrationPlugin`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/EHFrameRegistrationPlugin.cpp) registers `.eh_frame` sections with the C++ unwinder. Without this, `throw`/`catch` from JIT-compiled code fails (the unwinder cannot find the DWARF unwind tables), and `backtrace()` from a JIT frame may crash.

The plugin adds a post-fixup pass that:
1. Finds the `.eh_frame` section in the link graph
2. Records its executor address and size
3. In `notifyEmitted()`, calls `__register_frame(addr)` on Linux (via `libgcc_s` or `libunwind`) or the equivalent Mac OS X API

```cpp
// Setup (in ObjectLinkingLayer configuration):
ObjLayer.addPlugin(std::make_unique<EHFrameRegistrationPlugin>(
    *ES,
    std::make_unique<InProcessEHFrameRegistrar>()));
// InProcessEHFrameRegistrar calls __register_frame/__deregister_frame
// directly in the current process (for in-process JIT).
```

For out-of-process JIT, `EHFrameRegistrationPlugin` sends the frame data to the executor via EPC for registration there.

### 109.7.3 DebugObjectManagerPlugin

[`DebugObjectManagerPlugin`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/DebugObjectManagerPlugin.cpp) captures a copy of each linked object file and registers it with the GDB/LLDB JIT debug interface. The plugin:

1. In `notifyLoaded()`: copies the object bytes (with all relocations applied, to produce a valid ELF/MachO with live addresses)
2. In `notifyEmitted()`: writes the object to the GDB JIT linked list and calls `__jit_debug_register_code()`
3. In `notifyFailed()`: releases the object copy without registering

```cpp
ObjLayer.addPlugin(std::make_unique<DebugObjectManagerPlugin>(
    *ES,
    std::make_unique<InProcessDebugObjectRegistrar>(*ES)));
```

### 109.7.4 PerfSupportPlugin

[`PerfSupportPlugin`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/llvm/lib/ExecutionEngine/Orc/Debugging/PerfSupportPlugin.cpp) writes `jit_code_load` records to a per-process file readable by `perf inject --jit`. Records include the function name, load address, code size, and (optionally) the ELF object for source annotation.

### 109.7.5 Writing a Custom Plugin

```cpp
class AllocationLogger : public JITLinkPlugin {
public:
  void modifyPassConfig(MaterializationResponsibility &MR,
                        LinkGraph &G,
                        PassConfiguration &Config) override {
    // Add a post-allocation pass that logs all block addresses:
    Config.PostAllocationPasses.push_back([](LinkGraph &G) {
      for (auto &Sec : G.sections())
        for (auto *B : Sec.blocks())
          llvm::outs() << Sec.getName() << " @ " << B->getAddress()
                       << " size=" << B->getSize() << "\n";
      return Error::success();
    });
  }

  Error notifyEmitted(MaterializationResponsibility &MR) override {
    llvm::outs() << "Emitted " << MR.getSymbols().size()
                 << " symbols from " << MR.getTargetJITDylib().getName()
                 << "\n";
    return Error::success();
  }
};

// Register:
ObjLayer.addPlugin(std::make_unique<AllocationLogger>());
```

---

## 109.8 JITLink vs RuntimeDyld

### 109.8.1 Feature Comparison

| Feature | RuntimeDyld | JITLink |
|---------|-------------|---------|
| ELF x86_64 | Full | Full |
| ELF AArch64 | Partial (no thunks) | Full (thunks, relaxation) |
| ELF RISC-V | Partial | Full (relaxation, pair relocations) |
| MachO x86_64 | Full | Full |
| MachO arm64 | Partial | Full (W^X, MAP_JIT, compact unwind) |
| COFF x86_64 | Partial | Growing (SEH, `.pdata`/`.xdata`) |
| Concurrent linking | No (global mutex) | Yes (fully independent) |
| W^X compliance | No | Yes (`MAP_JIT`, two-phase commit) |
| Plugin architecture | No | Yes (`JITLinkPlugin`) |
| Dead-stripping | No | Yes (per-block granularity) |
| LinkGraph inspection | No | Yes (full graph traversal API) |
| GOT relaxation | No | Yes (x86_64, AArch64) |
| Range-extension thunks | No | Yes (AArch64, RISC-V) |
| Out-of-process linking | No | Yes (`MapperJITLinkMemoryManager`) |
| ObjC metadata registration | No | Yes (via `MachOPlatform` plugin) |

### 109.8.2 Migration from RuntimeDyld

```cpp
// Old: RTDyldObjectLinkingLayer with SectionMemoryManager:
RTDyldObjectLinkingLayer OldObjLayer(*ES, []() {
  return std::make_unique<SectionMemoryManager>();
});

// New: ObjectLinkingLayer with JITLink and plugins:
auto MemMgr = std::make_unique<InProcessMemoryManager>(
    sys::Process::getPageSizeEstimate());
ObjectLinkingLayer NewObjLayer(*ES, std::move(MemMgr));
NewObjLayer.addPlugin(std::make_unique<EHFrameRegistrationPlugin>(
    *ES, std::make_unique<InProcessEHFrameRegistrar>()));
NewObjLayer.addPlugin(std::make_unique<DebugObjectManagerPlugin>(
    *ES, std::make_unique<InProcessDebugObjectRegistrar>(*ES)));
```

### 109.8.3 Using llvm-jitlink

`llvm-jitlink` is the reference implementation and debugging tool for JITLink:

```bash
# Basic: link and execute ELF objects:
llvm-jitlink -entry=main foo.o bar.o

# Dump the LinkGraph before and after passes:
llvm-jitlink -show-graphs -noexec foo.o

# Verify all relocations without executing:
llvm-jitlink -verify -noexec foo.o

# Show all defined/external symbols:
llvm-jitlink -show-defined-symbols -show-external-symbols foo.o

# Out-of-process execution:
llvm-jitlink foo.o --oop-executor=./llvm-jitlink-executor

# Cross-architecture (AArch64 on x86 host):
llvm-jitlink -triple=aarch64-linux-gnu \
             --oop-executor=./llvm-jitlink-executor-aarch64 \
             foo_aarch64.o

# Benchmark: measure JIT link time for 100 iterations:
for i in $(seq 100); do llvm-jitlink -noexec foo.o; done | time cat
```

---

## Chapter 109 Summary

- JITLink replaces RuntimeDyld as ORC's in-process linker, addressing W^X compliance, concurrency, plugin extensibility, and incomplete platform support.
- `LinkGraph` is the central IR: `Section`→`Block`→`Edge` (relocations) + `Symbol`s; all format-specific knowledge is consumed during parsing; downstream passes operate on the format-independent graph.
- The pipeline has 9 phases: parse → pre-prune plugins → dead-strip → post-prune plugins (GOT/PLT synthesis, thunks) → allocate (assign addresses, resolve externals) → apply fixups → post-fixup plugins → finalize permissions → notify ORC.
- GOT entries and PLT stubs are created synthetically on demand in post-prune passes; x86_64 and AArch64 support GOTPCREL relaxation to eliminate redundant indirections; AArch64 and RISC-V support automatic range-extension thunk insertion.
- `JITLinkMemoryManager` provides W^X-safe two-phase commit: write phase (apply fixups to writable pages) then finalize phase (transition to read-only+executable); `MAP_JIT` + `pthread_jit_write_protect_np` on Apple Silicon.
- Platform-specific backends handle ELF x86_64 (full relocation table), ELF AArch64 (ADRP/ADD pairs, Branch26 thunks), RISC-V (call relaxation, paired HI20/LO12), COFF x86_64 (SEH), and MachO arm64 (compact unwind, ObjC metadata).
- The `JITLinkPlugin` architecture provides `modifyPassConfig` / `notifyEmitted` / `notifyFailed` hooks; standard plugins handle EH frame registration, GDB JIT debug info, and perf event emission.
- `ObjectLinkingLayer` replaces `RTDyldObjectLinkingLayer`; `llvm-jitlink` is the standalone tool for testing, verification, and benchmarking JITLink configurations.
