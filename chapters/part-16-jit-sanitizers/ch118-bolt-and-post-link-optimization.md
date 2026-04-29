# Chapter 118 — BOLT and Post-Link Optimization

*Part XVI — JIT, Sanitizers, and Diagnostic Tools*

Most compiler optimizations operate before linking: the compiler processes individual translation units, possibly with LTO merging all IR, and generates optimized object code. After linking, the binary is considered finished. BOLT (Binary Optimization and Layout Tool) challenges this assumption: it operates on the final linked binary, using actual runtime profile data to reorganize code in ways impossible at compile time. BOLT reconstructs the binary's control flow graph from machine code, maps a runtime profile (collected from `perf` hardware counters) to this graph, and applies profile-driven optimizations — primarily function and basic block reordering — that reduce instruction cache misses and improve branch prediction. This chapter covers BOLT's pipeline from profile collection through binary rewriting, the algorithms it applies, and practical deployment on production binaries including Clang itself.

---

## 118.1 BOLT Overview

### What Post-Link Optimization Is

Compile-time optimization must be conservative: the compiler cannot know which functions will be hot in production, which branches are almost always taken, or which code paths are exercised by real workloads rather than test suites. Profile-Guided Optimization (PGO, Chapter 67) addresses some of this by instrumenting a representative run, but PGO works at the compiler IR level and makes decisions that affect code generation.

BOLT operates at the fully-linked binary level. Its inputs are:

1. **The binary**: a final ELF (or Mach-O) executable or shared library
2. **A runtime profile**: either Linux `perf` data with LBR (Last Branch Record) samples, or data from BOLT's own instrumentation pass

Its output is a rewritten binary with the same semantics but reorganized code.

### What BOLT Optimizes

BOLT's primary optimization targets are **instruction-cache (i-cache) and iTLB (instruction TLB) efficiency**:

- Large programs (web servers, compilers, databases) have code footprints of tens of megabytes. The working set of frequently-executed code is much smaller, but it may be scattered across the binary because the linker places functions in link order (typically alphabetical or source-file order), not hotness order.
- Each i-cache or iTLB miss adds 50–300 cycles of latency. For a server with millions of requests per second, even a 1% reduction in i-cache misses translates to measurable throughput improvement.

BOLT can achieve 5–15% speedup on large C++ binaries — not by making individual operations faster, but by making the CPU's hardware more efficient at fetching and predicting code.

### Production Use

BOLT was developed at Meta (Facebook) and open-sourced. It is used in production at Meta for serving infrastructure binaries, achieving measured 2–10% throughput improvements on services like `mysqld`, `nginx`, and Facebook's TAO social graph cache server. Google uses a similar approach (Propeller) for Chromium.

BOLT was integrated into the LLVM monorepo as `llvm/tools/bolt/` in LLVM 14.

---

## 118.2 BOLT Pipeline

### Pipeline Stages

```
Input: ELF binary + profile data
         │
         ▼
┌─────────────────────────────────────────────────────┐
│ 1. BinaryContext construction                       │
│    - Parse ELF: sections, symbols, relocations      │
│    - Disassemble functions using LLVM MCDisassembler│
│    - Reconstruct Control Flow Graph (CFG)           │
│    - Identify call targets, jump tables             │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│ 2. Profile reading and annotation                   │
│    - Parse perf LBR data or BOLT .fdata             │
│    - Map branch counts to CFG edges                 │
│    - Infer missing counts (profile inference)       │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│ 3. Optimization passes                              │
│    - ReorderBlocks (BB reordering within functions) │
│    - ReorderFunctions (HFSort, CDSort)              │
│    - SplitFunctions (hot/cold splitting)            │
│    - Peephole, ICF, InlineSmallFunctions            │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│ 4. Binary emission                                  │
│    - Rewrite .text section                          │
│    - Update relocations, symbol table               │
│    - Write new ELF file                             │
└─────────────────────────────────────────────────────┘
         │
         ▼
Output: Optimized binary
```

### Source Organization

```
llvm/tools/bolt/
├── include/bolt/
│   ├── Core/
│   │   ├── BinaryContext.h      # Root object
│   │   ├── BinaryFunction.h     # Reconstructed function
│   │   ├── BinaryBasicBlock.h   # Basic block with edge counts
│   │   └── MCPlusBuilder.h      # MC instruction manipulation
│   ├── Passes/                  # Optimization pass headers
│   └── Profile/                 # Profile reading
├── lib/bolt/
│   ├── Core/
│   ├── Passes/
│   │   ├── ReorderBasicBlocks.cpp
│   │   ├── ReorderFunctions.cpp
│   │   ├── SplitFunctions.cpp
│   │   ├── Peephole.cpp
│   │   ├── IdenticalCodeFolding.cpp
│   │   └── InlineSmallFunctions.cpp
│   └── Profile/
│       ├── DataReader.cpp       # BOLT .fdata format
│       └── PerfDataReader.cpp   # Linux perf output
└── tools/
    ├── llvm-bolt/               # Main BOLT tool
    └── perf2bolt/               # perf → BOLT converter
```

---

## 118.3 The BinaryContext

### BinaryContext

[`bolt/Core/BinaryContext.h`](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.0/bolt/include/bolt/Core/BinaryContext.h) is BOLT's root object, analogous to LLVM's `LLVMContext`. It owns:

- `std::map<uint64_t, BinaryFunction *>`: all functions, keyed by address
- `BinarySection` list: all ELF sections
- `MCContext`, `MCAsmInfo`, `MCInstrInfo`, `MCSubtargetInfo`: LLVM MC infrastructure
- `MCDisassembler`: used for initial disassembly
- `MCPlusBuilder`: extended instruction manipulation

### BinaryFunction

`BinaryFunction` represents one function in the binary after CFG reconstruction. It contains:

```cpp
class BinaryFunction {
    // Basic blocks, in original order:
    std::vector<std::unique_ptr<BinaryBasicBlock>> BasicBlocks;
    
    // Profile data:
    uint64_t ExecutionCount;          // how many times this function ran
    
    // Address range:
    uint64_t Address;                 // original entry point
    uint64_t Size;                    // original size
    
    // Layout (set during optimization):
    std::vector<BinaryBasicBlock *> Layout;  // reordered BBs
    
    // MC state:
    MCSymbol *Symbol;                 // function's MCSymbol
    
    bool isSimple() const;            // can BOLT optimize this function?
    bool hasProfile() const;          // has runtime profile data?
};
```

A function is "simple" if BOLT can fully reconstruct its CFG. Functions with non-standard control flow (indirect tail calls, exception handling edge cases, assembly with computed GOT offsets) may be marked non-simple and left unmodified.

### BinaryBasicBlock

```cpp
class BinaryBasicBlock {
    std::vector<MCInst> Instructions;   // the instructions in this BB
    
    // Successor edges (outgoing edges from this BB):
    std::vector<BinaryBasicBlock *> Successors;
    std::vector<uint64_t> BranchInfo;   // profile counts per successor
    
    uint64_t ExecutionCount;            // total executions of this BB
    
    bool isCold() const;                // is this BB rarely executed?
    bool isHot()  const;                // is this BB frequently executed?
};
```

Edge counts come from the profile. BOLT uses them to determine:
- Which basic blocks are hot (should be placed in the hot text region)
- Which branches are predicted as taken vs not-taken
- The optimal ordering of basic blocks within a function

### CFG Reconstruction

BOLT uses `MCDisassembler` to disassemble each function:

```cpp
// Simplified CFG reconstruction:
void BinaryFunction::buildCFG() {
    for (uint64_t offset = 0; offset < Size; ) {
        MCInst Inst;
        uint64_t InstSize;
        Disasm->getInstruction(Inst, InstSize, Code + offset, Address + offset);
        
        // Identify control flow:
        if (MCII->get(Inst.getOpcode()).isBranch()) {
            // Create edge to target
            uint64_t target = getJumpTarget(Inst);
            BB->addSuccessor(getOrCreateBBAt(target));
        }
        if (MCII->get(Inst.getOpcode()).isCall()) {
            // Record call site
            recordCallSite(Inst, Address + offset);
        }
        offset += InstSize;
    }
}
```

Jump tables are handled specially: BOLT reads the jump table data from the binary's read-only data sections to enumerate switch case targets.

---

## 118.4 Profile Collection and Processing

### Linux perf with LBR

**LBR (Last Branch Record)** is an Intel CPU feature that records the last 8–32 taken branch source/destination pairs in special MSRs. `perf record` can capture these along with instruction samples:

```bash
# Collect LBR profile during normal workload:
perf record \
  -e cycles:u \
  --branch-filter any,u \
  -o perf.data \
  -- ./binary --run-workload

# Or for an already-running process:
perf record \
  -e cycles:u \
  --branch-filter any,u \
  -o perf.data \
  -p <PID> \
  -- sleep 30
```

The LBR data gives branch pairs `(from_addr, to_addr)` with counts. BOLT's `perf2bolt` converts this to function-level and BB-level edge counts:

```bash
# Convert perf data to BOLT profile:
perf2bolt ./binary \
  -p perf.data \
  -o bolt_profile.fdata

# Or using the aggregated format:
llvm-bolt ./binary \
  -p perf.data \
  --perf2bolt-mode \
  -o /dev/null
```

### BOLT Instrumentation

For environments where LBR is unavailable (VMs, non-Intel hardware), BOLT provides its own instrumentation:

```bash
# Step 1: Instrument the binary:
llvm-bolt ./binary \
  --instrument \
  -o ./binary.instrumented

# Step 2: Run the instrumented binary under the workload:
./binary.instrumented --run-workload
# Creates: /tmp/prof.fdata (default path)

# Step 3: Optimize with the collected profile:
llvm-bolt ./binary \
  -data /tmp/prof.fdata \
  -o ./binary.bolt \
  -reorder-blocks=ext-tsp \
  -reorder-functions=hfsort+ \
  -split-functions \
  -split-all-cold
```

BOLT instrumentation inserts counters on each basic block edge, similar to PGO instrumentation but at the binary level (post-link).

### Profile Inference

Real workloads rarely cover 100% of code paths. When the profile covers only 60–70% of functions, BOLT uses **profile inference** (also called stale profile handling) to:

1. Infer edge counts for unexecuted paths via the flow equations (total in-flow = total out-flow for non-entry/exit blocks)
2. Mark uncovered functions as cold without profile
3. Handle address layout changes (if the binary was rebuilt without matching the profile)

---

## 118.5 Function Reordering (HFSort)

### The Problem

The default function order in a linked binary is determined by link order (the order `.o` files appear on the linker command line). This order has no relationship to runtime call patterns. Hot functions are interleaved with cold functions, causing:

- **iTLB misses**: the instruction TLB is too small to simultaneously hold all frequently-used function pages
- **i-cache pollution**: cache lines containing cold code displace hot code cache lines

### HFSort Algorithm

HFSort (Hotness-First Sort) clusters hot functions based on call relationships:

1. Build a call graph with edge weights = call count × callee execution count
2. Sort functions by weight (descending)
3. Greedily merge caller/callee clusters when they are frequently called together

The result: hot functions that frequently call each other are placed adjacently in the binary, maximizing cache line sharing.

```
Example function layout before/after HFSort:

Before (link order):       After (HFSort):
a_cold()                   main()
b_hot()                    b_hot()
c_cold()                   b_helper_hot()
d_cold()                   c_medium()
main()                     a_cold()
b_helper_hot()             c_cold()
c_medium()                 d_cold()
```

### HFSort+ and CDSort

BOLT offers multiple ordering algorithms:

- `hfsort`: basic hotness-first
- `hfsort+`: improved clustering with density-based merging
- `cdsort`: call-density sort, optimizes for i-cache line utilization
- `ext-tsp` (Extended TSP): traveling salesman approximation weighted by branch probabilities; generally best for `ReorderBlocks`, often also used for functions

```bash
# Choose ordering algorithm:
llvm-bolt ./binary \
  -data profile.fdata \
  -o ./binary.bolt \
  -reorder-functions=hfsort+  # or cdsort or ext-tsp
```

### Hot/Cold Split

`-split-functions` moves rarely-executed basic blocks (cold BBs) to a separate section at the end of the binary. This concentrates hot code in the first few pages of the text segment, dramatically improving iTLB hit rates:

```
.text (hot code):
  [main hot path]
  [b_hot hot BBs]
  [c_medium hot BBs]
  ...

.text.cold (cold code, placed after all hot code):
  [error handling BBs]
  [initialization-only BBs]
  [rarely-taken exception paths]
```

---

## 118.6 Basic Block Reordering

### The Fallthrough Opportunity

On x86-64, conditional branches (`je`, `jne`, etc.) require a branch instruction only when the branch is *taken*; the fallthrough path is implicit and costs nothing. If the hot (frequently-taken) path is always the fallthrough, the CPU's branch predictor predicts it easily and the branch is practically free.

Without profile data, the compiler arranges basic blocks in source order: the "then" branch is typically the fallthrough, and the "else" is a taken jump. With profile data, BOLT can reverse this when the "else" path is hotter.

### ReorderBlocks Pass

The `ReorderBlocks` pass (`bolt/lib/Passes/ReorderBasicBlocks.cpp`) reorders BBs within each function using profile-guided layout:

**ExtTSP algorithm** (Extended Traveling Salesman Problem for basic blocks):
- Models the benefit of fallthrough chains: BBs that flow into each other without branches are worth placing adjacently
- Uses a greedy approach: repeatedly merge the highest-benefit adjacent chain pair
- Objective: minimize the number of taken branches (each taken branch wastes 1 instruction slot and may cause a branch misprediction)

```
Before (branch taken for hot path):
BB0: cmp eax, 0
     jne BB2        ← hot path (taken 900/1000)
BB1: [cold path]   ← fallthrough (only 100/1000)
     jmp BB3
BB2: [hot path]    ← must jump here 900/1000
BB3: [continuation]

After (ReorderBlocks makes hot path the fallthrough):
BB0: cmp eax, 0
     je BB1         ← cold path (taken only 100/1000)
BB2: [hot path]    ← fallthrough (900/1000 free)
BB3: [continuation]
BB1: [cold path]
```

This reduces branch instructions on the hot path from one conditional-taken to one conditional-not-taken (or no branch if the conditional falls through).

### Split Functions Pass

`-split-functions` and `-split-all-cold` move cold BBs to a `.text.cold` section:

```bash
llvm-bolt ./binary \
  -data profile.fdata \
  -o ./binary.bolt \
  -reorder-blocks=ext-tsp \
  -split-functions \
  -split-all-cold \          # move ALL cold BBs (not just 50/50)
  -split-eh                  # also split exception handler BBs
```

After splitting, a function's code may appear in two places: hot BBs in `.text`, cold BBs in `.text.cold`. BOLT inserts a `jmp` from the last hot BB to the first cold BB when needed.

---

## 118.7 Code Transformations

### Peephole Optimizations

`bolt/lib/Passes/Peephole.cpp` applies architecture-specific local rewrites:

- **Short jump encoding**: replace `jmp near` (5 bytes) with `jmp short` (2 bytes) when the target is within ±127 bytes — common after reordering moves code closer together
- **NOP removal**: remove alignment NOPs that are no longer needed after reordering
- **Unconditional branch removal**: remove `jmp` to the immediately following instruction (a fallthrough)
- **Redundant move elimination**: pairs like `mov rax, rbx; mov rbx, rax` after reordering

### Identical Code Folding (ICF)

`bolt/lib/Passes/IdenticalCodeFolding.cpp` merges functions with identical machine code:

```cpp
// These two functions compile to identical machine code:
int add1(int x) { return x + 1; }
int inc(int x) { return x + 1; }

// After ICF: both symbols point to the same function body
// References to 'inc' are redirected to 'add1' (or vice versa)
```

This is conceptually similar to LLD's `--icf=all` but applied at the binary level, after LTO has already run. ICF can eliminate several percent of code in large C++ binaries with many template instantiations.

### Inline Small Functions

`bolt/lib/Passes/InlineSmallFunctions.cpp` inlines functions that:
- Are called frequently (hot call site)
- Have small code size (< threshold, default: 4 instructions)

This eliminates function call overhead (push/pop, branch, return) for hot getters/setters that a PGO-unaware compiler may not have inlined.

### SimplifyRODataLoads

`bolt/lib/Passes/SimplifyRODataLoads.cpp` replaces loads from read-only data sections with constant values:

```asm
; Before:
movq   .rodata_table+8(%rip), %rax   ; load from .rodata

; After (BOLT replaces with immediate if value is known constant):
movq   $42, %rax                      ; immediate
```

This is possible because BOLT operates on the final binary where all .rodata values are fixed.

---

## 118.8 Linker Considerations

### Relocation Requirements

BOLT's ability to rewrite a binary depends on whether it can update all references (addresses) that point into the rewritten code. For a complete rewrite, BOLT needs relocations:

```bash
# Link with relocation information preserved:
clang-22 -Wl,--emit-relocs -O2 source.cpp -o binary

# Verify relocations are present:
readelf -r binary | head -20
# Should show .rela.text entries

# BOLT with relocations (full capability):
llvm-bolt binary \
  -relocs \                       # use relocation data
  -data profile.fdata \
  -o binary.bolt \
  -reorder-blocks=ext-tsp \
  -reorder-functions=hfsort+
```

**Without relocations**, BOLT uses disassembly heuristics to find and patch all code references. This is less reliable and some transformations (function reordering across object boundaries) may not be available. Always prefer `--emit-relocs` for production BOLT usage.

### LLD Integration

LLD supports `--emit-relocs` directly:

```bash
# CMake:
target_link_options(my_target PRIVATE -Wl,--emit-relocs)

# Direct link:
clang-22 -fuse-ld=lld -Wl,--emit-relocs -O2 source.cpp -o binary
```

Note: `--emit-relocs` increases binary size by the size of the relocation table (typically 5–20% of code size). This overhead disappears in BOLT's output binary.

### BOLT + LTO

BOLT and LTO are complementary:

1. Compile with LTO (`-flto`): inter-procedural optimizations, devirtualization, inlining across TUs
2. Link: LTO generates the final binary
3. Apply BOLT: reorder and optimize the LTO-generated binary with actual runtime profile

The order matters — BOLT must run last, on the final binary. LTO's output is the input to BOLT.

```bash
# LTO + BOLT workflow:
clang-22 -flto -O2 -Wl,--emit-relocs source.cpp -o binary_lto
perf record -e cycles:u --branch-filter any,u -o perf.data -- ./binary_lto --workload
llvm-bolt binary_lto -p perf.data -o binary_lto_bolt \
  -reorder-blocks=ext-tsp -reorder-functions=hfsort+
```

### BOLT + PGO

PGO and BOLT are also complementary:

- **PGO** (Chapter 67): feedback to the compiler for IR-level decisions (inlining threshold, loop unrolling, branch hints, code placement hints)
- **BOLT**: post-link layout optimization using actual hardware branch data

Using both gives complementary benefits: PGO improves individual code generation; BOLT improves the whole binary's cache behavior.

```bash
# Full optimization pipeline:
# 1. PGO instrumented build
clang-22 -fprofile-instr-generate -O2 source.cpp -o binary_pgo_instr
./binary_pgo_instr --workload
llvm-profdata merge default.profraw -o pgo.profdata

# 2. PGO optimized build with relocs
clang-22 -fprofile-instr-use=pgo.profdata -O2 \
         -Wl,--emit-relocs source.cpp -o binary_pgo

# 3. BOLT profile collection
perf record -e cycles:u --branch-filter any,u \
  -o perf.data -- ./binary_pgo --workload

# 4. BOLT optimization
llvm-bolt binary_pgo \
  -p perf.data \
  -o binary_pgo_bolt \
  -reorder-blocks=ext-tsp \
  -reorder-functions=hfsort+
```

---

## 118.9 BOLT Results and Measurement

### Profile Coverage Statistics

```bash
# Check profile coverage:
llvm-bolt ./binary \
  -data profile.fdata \
  -o /dev/null \
  -print-profile-stats

# Output:
# BOLT-INFO: binary contains 8421 functions
# BOLT-INFO: 5234 functions with profile (62.15%)
# BOLT-INFO: profile covers 89.7% of execution (by samples)
# BOLT-INFO: top 100 hot functions cover 67.3% of execution
```

Profile coverage below 50% by execution weight suggests the workload used for profiling is not representative. More representative profiling (longer runs, more diverse inputs) improves BOLT's effectiveness.

### Before/After Benchmarking

```bash
# Baseline (before BOLT):
perf stat --repeat 5 ./binary --benchmark
# Performance counter stats for './binary --benchmark':
#   1,234,567 iTLB-load-misses
#   9,876,543 L1-icache-load-misses
#   3.142000000 seconds time elapsed (± 0.5%)

# After BOLT:
perf stat --repeat 5 ./binary.bolt --benchmark
# Performance counter stats for './binary.bolt --benchmark':
#     456,789 iTLB-load-misses        # -63%
#   4,321,098 L1-icache-load-misses   # -56%
#   2.891000000 seconds time elapsed  # -8%
```

The primary metrics for BOLT's effectiveness are i-cache and iTLB misses. Reduction in these correlates directly with runtime improvement for compute-bound workloads.

### Binary Size Impact

BOLT rewrites the `.text` section and adds metadata:

```bash
size binary binary.bolt
#   text    data     bss     dec     hex filename
# 8234567  456789  234567  8925923  883e83 binary
# 8891234  456789  234567  9582590  924ffe binary.bolt
# ^ slight increase from added jump trampolines and split function stubs
```

The size increase is typically 5–15%, mostly from:
- Jump trampolines for split cold functions
- Alignment padding between reordered functions
- BOLT metadata sections (`.bolt`, `.bolt.org`)

---

## 118.10 BOLT for Clang/LLVM Itself

### Self-Optimization

BOLT can optimize the Clang binary itself — using Clang to compile a large codebase as the profiling workload:

```bash
# 1. Link Clang with relocs:
cmake -GNinja -DCMAKE_C_COMPILER=clang-21 \
      -DCMAKE_CXX_COMPILER=clang++-21 \
      -DCMAKE_EXE_LINKER_FLAGS="-Wl,--emit-relocs" \
      -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_ENABLE_PROJECTS="clang" \
      ../llvm
ninja clang-22

# 2. Collect a profile using Clang compiling a real project:
perf record \
  -e cycles:u \
  --branch-filter any,u \
  -o clang_perf.data \
  -- clang-22 -O2 -c /path/to/large_file.cpp

# 3. Process perf data:
perf2bolt build/bin/clang-22 \
  -p clang_perf.data \
  -o clang_profile.fdata

# 4. Apply BOLT:
llvm-bolt build/bin/clang-22 \
  -data clang_profile.fdata \
  -o build/bin/clang-22-bolt \
  -reorder-blocks=ext-tsp \
  -reorder-functions=hfsort+ \
  -split-functions \
  -split-all-cold \
  -dyno-stats

# 5. Verify:
clang-22-bolt --version   # should work identically
```

### Measured Results

LLVM's own CI has measured BOLT improvements for the Clang binary:

| Benchmark | Without BOLT | With BOLT | Improvement |
|-----------|-------------|-----------|-------------|
| Compile LLVM headers | 100% | 93.2% | -6.8% |
| Compile large C++ TU | 100% | 95.1% | -4.9% |
| clang-tidy on codebase | 100% | 92.7% | -7.3% |
| iTLB misses | 100% | 41.2% | -58.8% |
| L1-icache misses | 100% | 46.3% | -53.7% |

The i-cache improvements are dramatic; the wall-clock improvements are smaller (bounded by non-cached work like disk I/O and parsing), but real and consistent.

### Integration with the LLVM Build System

LLVM 18+ added CMake support for building BOLT-optimized Clang as part of the standard toolchain build:

```cmake
# CMake option:
set(CLANG_BOLT_INSTRUMENT OFF CACHE BOOL "Instrument Clang with BOLT")
set(CLANG_BOLT "INSTRUMENT" CACHE STRING "BOLT mode: INSTRUMENT, PERF, or OFF")
```

The multi-stage build process:
1. Stage 1: build Clang normally
2. Stage 2: use stage-1 Clang to build a release Clang with `--emit-relocs`
3. Profile stage: instrument or collect perf data
4. BOLT stage: run `llvm-bolt` on stage-2 Clang

This is the recommended approach for building a production Clang toolchain with maximum performance.

---

## Chapter Summary

- **BOLT** is a post-link binary optimizer that uses runtime profile data (Linux `perf` LBR or BOLT instrumentation) to reorder functions and basic blocks in the final binary, reducing i-cache and iTLB misses by 50–60% on typical large C++ programs.

- **The BOLT pipeline** reconstructs a control flow graph from machine code using `MCDisassembler`, annotates it with profile edge counts, applies optimization passes, and rewrites the binary's `.text` section.

- **Profile collection** uses `perf record --branch-filter any,u` to capture LBR data on modern Intel CPUs, or BOLT's own instrumentation for non-LBR platforms. `perf2bolt` converts `perf.data` to BOLT's `.fdata` profile format.

- **HFSort/HFSort+/CDSort** reorder functions by call graph clustering to maximize co-location of hot caller/callee pairs. `ext-tsp` (Extended Traveling Salesman approximation) reorders basic blocks within functions to maximize fallthrough opportunities.

- **Split functions** (`-split-functions`) move cold basic blocks to `.text.cold`, concentrating hot code in the first few pages of the text segment for maximum iTLB efficiency.

- **Code transformations**: ICF (identical code folding), peephole rewrites (short jumps, NOP removal), and small function inlining complement the layout optimizations.

- **Linker requirements**: `--emit-relocs` enables full BOLT capability. BOLT is compatible with both LTO and PGO — all three optimizations are complementary and can be combined.

- **Applied to Clang itself**, BOLT achieves 5–8% compile-time speedup with 50–60% reduction in i-cache misses, making it a practical addition to any production toolchain build pipeline.


---

@copyright jreuben11
